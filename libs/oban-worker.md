# Oban Worker: Implementation & Testing Guide

Injected automatically for `*_oban_worker.ex` files. Follow these rules.

## Module Declaration

Workers live in a `workers/` subfolder under their context:

```
lib/platform/fundamentals/documents/workers/pdf_converter_oban_worker.ex
lib/platform/markets/serie_sources/workers/serie_source_oban_worker.ex
```

```elixir
defmodule Platform.Fundamentals.Documents.Workers.PdfConverterWorker do
  @moduledoc """
  One-line summary of what this worker does.

  Enqueued by: [where/when it gets enqueued].
  Sets status to :converting before starting, :converted on success.

  ## Duplicate prevention
  `unique: [period: :infinity, fields: [:args]]` prevents duplicate jobs
  while available/scheduled/executing. Status guard in perform/1 handles
  crash + retry scenarios.
  """

  use Oban.Worker,
    queue: :default,
    max_attempts: 3,
    unique: [period: :infinity, fields: [:args], states: [:available, :scheduled, :executing]]

  @impl Oban.Worker
  def perform(%Oban.Job{args: %{"document_id" => document_id}}) do
    ...
  end
end
```

## Queues

- `:default` — general background work
- `:data_fetcher` — external data fetches (rate-limited separately)

## perform/1

Pattern match directly on `%Oban.Job{args: ...}`. Args are JSON — always **string keys**, never atoms:

```elixir
@impl Oban.Worker
def perform(%Oban.Job{args: %{"document_id" => document_id}}) do
  scope = Scope.global_access()   # workers have no user — always global_access()
  document = Documents.get!(scope, document_id)
  ...
end
```

## Return Values

| Return | Effect |
|--------|--------|
| `:ok` | Job succeeded |
| `{:error, reason}` | Job failed — retried up to `max_attempts` |
| `{:snooze, seconds}` | Defer and retry after N seconds — not counted as attempt |
| `:discard` | Permanent failure — no retry |

## Idempotency: Status Guard

Retries are unavoidable. Guard against re-running with a status check at the top of `perform/1`:

```elixir
def perform(%Oban.Job{args: %{"document_id" => document_id}}) do
  document = Documents.get!(Scope.global_access(), document_id)

  case document.status do
    :converting -> :ok   # already in progress — skip
    :converted  -> :ok   # already done — skip
    _ ->
      {:ok, document} = Documents.update(scope, document, %{status: :converting})
      # ... do the work
  end
end
```

## Unique Jobs

Prevent duplicate enqueues with `unique`:

```elixir
use Oban.Worker,
  queue: :default,
  max_attempts: 3,
  unique: [period: :infinity, fields: [:args], states: [:available, :scheduled, :executing]]
```

`fields: [:args]` — uniqueness is keyed on the full args map. Use a single identifying key (e.g. `document_id`) per job type to keep args minimal.

## Enqueueing

String keys in args — they survive JSON serialization:

```elixir
# Enqueue and handle conflict gracefully
%{"document_id" => document.id}
|> PdfConverterWorker.new()
|> Oban.insert()

# Enqueue and raise on failure
Oban.insert!(PdfConverterWorker.new(%{"document_id" => document.id}))
```

Workers chain to the next step on success:

```elixir
case PdfConverter.convert(document, scope) do
  {:ok, _} ->
    Oban.insert!(ExtractionWorker.new(%{"document_id" => document_id}))
    :ok
  {:error, reason} ->
    {:error, reason}
end
```

## Re-enqueueing for Pagination

When a source returns `:more`, re-enqueue the same job to drain the backlog:

```elixir
with {:ok, series, continuation} <- fetch_fn.() do
  write_results = Enum.map(series, &write_fn.(&1))

  if continuation == :more do
    args |> __MODULE__.new() |> Oban.insert()
  end

  :ok
end
```

## Do

- Always use string keys in args — atoms don't survive JSON
- Always use `Scope.global_access()` — workers have no user context
- Always guard against retries with a status check at the top of `perform/1`
- Document in `@moduledoc`: what it does, when enqueued, and how duplicate prevention works
- Return `:ok` from early-exit status guards — not an error

## Don't

- Don't use atom keys in args (`%{document_id: id}`) — they become strings after JSON round-trip
- Don't put workers outside a `workers/` subfolder
- Don't call `Repo` directly — use context functions with `Scope.global_access()`
- Don't re-run work that already completed — check status first

## Testing

### Setup

```elixir
defmodule Platform.Fundamentals.Documents.Workers.PdfConverterWorkerTest do
  use Platform.DataCase, async: false   # never async — Oban sandbox conflicts
  use Oban.Testing, repo: Platform.Repo

  alias Platform.Factory
  alias Platform.Fundamentals.Documents
  alias Platform.Fundamentals.Documents.Workers.PdfConverterWorker
  alias Platform.Accounts.Users.Scope

  setup do
    admin = Factory.insert(:user, admin: true)
    %{scope: Scope.for_user(admin)}
  end
```

`async: false` is mandatory — `Oban.Testing` uses the sandbox in shared mode, which is incompatible with concurrent connection ownership.

### perform_job/2

`Oban.Testing` provides `perform_job/2` — call the worker directly without going through the queue:

```elixir
assert :ok = perform_job(PdfConverterWorker, %{"document_id" => document.id})
```

Args can use string or atom keys — `perform_job` normalizes them. The worker will receive string keys as it does in production.

### What to Test

One `describe "perform/1"` block covering:

1. **Happy path** — insert a factory record in the right starting state, call `perform_job`, assert the DB state changed
2. **Each idempotency guard** — one test per status that causes early `:ok` exit
3. **Error path** — missing file, bad data, or upstream failure

```elixir
describe "perform/1" do
  test "converts document and sets status to :converted", %{scope: scope} do
    document = Factory.insert(:document, status: :uploaded)
    write_pdf(document)   # set up fixture file if needed

    assert :ok = perform_job(PdfConverterWorker, %{"document_id" => document.id})

    updated = Documents.get!(scope, document.id)
    assert updated.status == :converted
  end

  test "returns :ok without re-converting when already :converting", %{scope: scope} do
    document = Factory.insert(:document, status: :converting)

    assert :ok = perform_job(PdfConverterWorker, %{"document_id" => document.id})

    updated = Documents.get!(scope, document.id)
    assert updated.status == :converting   # unchanged
  end

  test "returns :ok without re-converting when already :converted", %{scope: scope} do
    document = Factory.insert(:document, status: :converted)

    assert :ok = perform_job(PdfConverterWorker, %{"document_id" => document.id})

    updated = Documents.get!(scope, document.id)
    assert updated.status == :converted   # unchanged
  end

  test "returns error when required file is missing" do
    document = Factory.insert(:document, status: :uploaded)

    assert {:error, :pdf_not_found} =
             perform_job(PdfConverterWorker, %{"document_id" => document.id})
  end
end
```

### Asserting Enqueued Jobs

When a worker chains to the next step, assert the downstream job was enqueued:

```elixir
test "enqueues ExtractionWorker on success" do
  document = Factory.insert(:document, status: :uploaded)
  write_pdf(document)

  assert :ok = perform_job(PdfConverterWorker, %{"document_id" => document.id})

  assert_enqueued worker: ExtractionWorker, args: %{"document_id" => document.id}
end
```

### Testing — Do

- Use `async: false` — mandatory for Oban.Testing
- Use `perform_job/2` — never call `worker.perform/1` directly
- Test each idempotency guard (one test per early-exit status)
- After `perform_job`, assert DB state via the context — not raw Repo

### Testing — Don't

- Don't use `async: true` — breaks Oban sandbox
- Don't assert on internal implementation (Repo called, broadcast fired)
- Don't skip idempotency tests — they catch the most common retry bugs

See also: [[context]]
