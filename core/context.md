# Context: Implementation & Testing Guide

Injected for files under `lib/platform/*/*/` and their `test/` counterparts.
Covers context modules, schemas, private implementation files, and tests.

For the `ecto_context` and `static_context` macro APIs (return values, opts,
callbacks), read the libraries' usage rules directly:

- `deps/ecto_context/usage-rules.md`
- `deps/static_context/usage-rules.md`

This file covers Platform-specific conventions on top of those. Read both
the library rules and this file when touching a context.

## File Structure

```
lib/platform/<namespace>/<context>/
  <context>.ex              ← public API — the ONLY entry point from outside
  <schema>.ex               ← Ecto schema
  <schema>_<role>.ex        ← private implementation (e.g. article_exporter.ex)

test/platform/<namespace>/<context>/
  <context>_test.exs        ← mirrors lib path; NOT flat in namespace folder
```

Private files are prefixed with the schema name (`article_exporter.ex`,
`article_parser.ex`). This groups related files in fuzzy search and makes
ownership clear at a glance.

Public functions defined in private files are exposed via `defdelegate` in
the context — never call private modules from outside.

## Ecto Contexts

All DB access goes through `ecto_context`. No hand-written
`list/get/create/update/delete`.

```elixir
defmodule Platform.Content.Articles do
  import Ecto.Query
  import EctoContext

  alias Platform.Content.Articles.Article
  alias Platform.Core.Scope

  ecto_context schema: Article, scope: &__MODULE__.scope/2 do
    list()
    list_for()
    get!()
    create()
    update()
    delete()
  end

  def scope(query, %Scope{user: user}), do: where(query, user_id: ^user.id)

  def permission(_action, record, %Scope{user: user}), do: record.user_id == user.id
  def permission(_action, _record, _scope), do: false
end
```

Declare only the functions actually needed. For opts, return values, and
the `scope/2` vs `permission/3` contract, see `ecto_context/usage-rules.md`.

## Authorization Contract

Every context implements `scope/2` and `permission/3`. No exceptions.
Public-read contexts use `def scope(q, _), do: q` and
`permission(:get, _, _), do: true` — explicit no-op, not omitted.

```elixir
# Public read, admin-only writes
def scope(query, _scope), do: query
def permission(:get, _record, _scope), do: true
def permission(_action, _record, %Scope{user: %{is_admin: true}}), do: true
def permission(_action, _record, _scope), do: false
```

Use `_action` as a catch-all when the same rule applies to all actions.
Split by action atom (`:create`, `:update`, `:delete`, `:get`) when rules
differ per operation.

`Scope.global_access/0` is a **last-resort escape hatch** — only at call
sites that have no user context and no way to acquire one:

- Oban workers (background jobs — no HTTP request, no session)
- Plugs that *establish* identity (before a user scope exists)
- Seeds

Every other call site — LiveViews, controllers, context-to-context calls,
higher-level business workflows — receives a scope via parameter. Context
functions **never** create a scope. Reaching for `global_access/0` inside
a context is a red flag; the caller should have passed a scope in.

> The name invites overuse. When in doubt, don't. If you can't point to
> an Oban worker / identity plug / seed, you're in the wrong place.

## Schemas (`use Platform, :schema`)

`use Platform, :schema` replaces `use Ecto.Schema` and provides:

- `use Ecto.Schema`
- `import Ecto.Changeset`
- `import StaticContext.Schema` — makes `static_belongs_to` available
- `@timestamps_opts [type: :utc_datetime]` — UTC timestamps by default, so
  `timestamps()` needs no arguments

Defined in `lib/platform.ex` alongside the `__using__` dispatch — same
pattern as `PlatformWeb`.

```elixir
defmodule Platform.Content.Articles.Article do
  use Platform, :schema

  schema "articles" do
    field :title, EctoTrim
    field :body,  EctoTrim, mode: :multi_line
    belongs_to :author, Platform.Accounts.Users.User
    timestamps()
  end

  def changeset(article, attrs, _scope \\ nil) do
    article
    |> cast(attrs, [:title, :body, :author_id])
    |> validate_required([:title, :author_id])
  end
end
```

**`EctoTrim`** — always use for user-facing text fields. Two modes, and
**picking the wrong one is a recurring bug**:

- **Default (`:single_line`)** — collapses ALL whitespace (including newlines)
  to single spaces. Use for names, titles, slugs, single-line inputs.
- **`mode: :multi_line`** — preserves newlines, collapses 3+ consecutive
  newlines to 2. **Required for any textarea / body / description field.**
  Using `:single_line` here destroys paragraph breaks silently.

Rule of thumb: if the form uses `<textarea>`, the field needs `mode: :multi_line`.

Changesets always take a third `_scope` argument — even if unused.

## Static Lookup Data

For lookup tables that live in code (mine types, document types, statuses)
— not in the database. For the `static_context` macro API (return values,
opts, available functions), see `deps/static_context/usage-rules.md`.

```elixir
defmodule Platform.Fundamentals.MineTypes do
  import StaticContext
  alias Platform.Fundamentals.MineTypes.MineType

  static_context struct: MineType do
    list()
    list_by()
    get()
    get!()
    get_by()
    get_by!()
  end

  def entries do
    [
      %MineType{id: "open_pit",    name: "Open Pit"},
      %MineType{id: "underground", name: "Underground"}
    ]
  end
end
```

Struct lives in `<context>/<struct>.ex` as a plain `defstruct` with
`@enforce_keys`:

```elixir
defmodule Platform.Fundamentals.MineTypes.MineType do
  @enforce_keys [:id, :name]
  defstruct [:id, :name]
end
```

Use `static_belongs_to` in Ecto schemas (via `use Platform, :schema`):

```elixir
schema "mines" do
  field :name, EctoTrim
  static_belongs_to :mine_type, Platform.Fundamentals.MineTypes
  timestamps()
end
```

In changesets, cast and validate the `_id` field — never the virtual struct:

```elixir
|> cast(attrs, [:mine_type_id])
|> validate_inclusion(:mine_type_id, Enum.map(MineTypes.list(), & &1.id))
```

**`static_belongs_to` fields are NOT Ecto associations.** They cannot be
preloaded via `Repo.preload` or the `preload:` opt. To resolve, call the
static context directly: `CompanyTypes.get(company.company_type_id)`.

## Query Composition

Declare named query functions, pass via `query:` opt. Never create
`list_published`, `list_active` wrappers.

```elixir
def published(query, now \\ DateTime.utc_now()) do
  where(query, [a], a.published_at <= ^now)
end

# At the call site
Articles.list(scope, query: &Articles.published/1, order_by: [desc: :published_at])
```

## Testing

> **Tests are part of the change — not a follow-up.** Every context
> modification ships with its tests in the same commit:
>
> - **New public function** → add a `describe "<fn>/<arity>"` block covering
>   happy path + every `permission/3` rule that guards it
> - **New `scope/2` clause** → add a test asserting the new filtering for
>   the scope variant it handles
> - **New `permission/3` clause** → add a test per new rule (assert `true`
>   and `false` cases)
> - **New custom changeset** → add a test per validation rule
> - **Changed return shape / error tuple** → update the existing tests to
>   assert the new shape
>
> Before marking a context change done: open the `_test.exs`, count
> `describe` blocks against public functions in the context, and confirm
> every `permission/3` clause has a matching assertion. Untested functions
> drift silently and only surface in production.

Tests verify the two things unique per context: `scope/2` filtering and
`permission/3` authorization. The generated CRUD plumbing is covered by
`ecto_context` itself — don't re-test it.

**Never `async: true` on `DataCase` tests.** SQLite serializes writes, and
concurrent transactions hit `database is locked` / `database busy` errors
under the sandbox. Context tests always run synchronously.

```elixir
defmodule Platform.Content.ArticlesTest do
  use Platform.DataCase

  alias Platform.Content.Articles
  alias Platform.Core.Scope

  setup do
    admin = Factory.insert(:user, admin: true)
    user  = Factory.insert(:user, admin: false)

    %{
      admin_scope:   Scope.for_user(admin),
      user_scope:    Scope.for_user(user),
      visitor_scope: Scope.for_user(nil)
    }
  end

  describe "list/2" do
    test "filters by user_id", %{user_scope: scope} do
      Factory.insert(:article, user_id: scope.user.id)
      Factory.insert(:article)  # other user
      assert [_] = Articles.list(scope)
    end
  end

  describe "permission/3" do
    test "owner may update", %{user_scope: scope} do
      article = Factory.insert(:article, user_id: scope.user.id)
      assert Articles.permission(:update, article, scope)
    end

    test "non-owner may not update", %{user_scope: scope} do
      other = Factory.insert(:article)
      refute Articles.permission(:update, other, scope)
    end
  end
end
```

### Factories

Use `Factory.insert/2` and `Factory.build/2` — never `Repo.insert` in
tests, never fixtures. Alias explicitly: `alias Platform.Factory`.

Prefix sequence names with the factory name: `:blueprint_title`, not `:title`.
Globally shared sequence names collide across factories.

```elixir
def blueprint_factory do
  %Blueprint{
    slug: sequence(:blueprint_slug, &"blueprint-#{&1}"),
    title: sequence(:blueprint_title, &"Blueprint #{&1}")
  }
end
```

### Static Context Associations in Factories

Schemas with static context associations use a lazy resolver — the struct
is derived from the ID at build time:

```elixir
def mine_factory do
  %Mine{
    mine_type_id: "open_pit",
    mine_type: &Platform.Fundamentals.MineTypes.get!(&1.mine_type_id),
    mine_state_id: "production",
    mine_state: &Platform.Fundamentals.MineStates.get!(&1.mine_state_id),
  }
end
```

Override `mine_type_id` in a test and `mine_type` resolves automatically.
Never override the struct field directly.

## Do

- Use `ecto_context` for all DB access — no hand-written CRUD
- Implement `scope/2` and `permission/3` on every context
- Use fully-qualified module names in `belongs_to`/`has_many`:
  ```elixir
  belongs_to :user, Platform.Accounts.Users.User
  ```
- Prefix private implementation files with the schema name
  (`article_exporter.ex`)
- Expose private module functions via `defdelegate` in the context
- Use `EctoTrim` for all user-facing text fields — never plain `:string`
- Use `static_belongs_to` for static lookup associations
- Define changesets as `def changeset(struct, attrs, _scope \\ nil)`
- Mirror lib paths in test paths:
  `test/platform/<ns>/<ctx>/<ctx>_test.exs`
- Ship tests in the same commit as the context change — new function,
  new `scope/2` clause, new `permission/3` rule, new changeset → each
  needs a matching test, not "later"
- Run DataCase tests synchronously — SQLite writes serialize; `async: true`
  causes `database is locked` errors
- Build scopes in a `setup` block, pass via test context — not `defp`
- One `describe` per function: `describe "list/2" do`, `describe "create/3" do`
- Test `permission/3` directly: one `describe "permission/3"` with a test
  per meaningful rule — assert `true`/`false` by action and scope
- Use `Factory.insert/2` / `Factory.build/2` with factory-prefixed sequences

## Don't

- Don't hand-write `list/get/create/update/delete` outside the macro
- Don't create `list_published` / `list_active` wrapper functions — use
  `query:` opt with a named query function
- Don't skip `scope/2` and `permission/3` — `def scope(q, _), do: q` is
  valid and explicit
- Don't call private implementation files from outside their context
- Don't use plain `:string` for user-facing text fields
- Don't preload `static_belongs_to` fields via `Repo.preload` or
  `preload:` — fetch via the static context instead
- Don't use `Scope.global_access/0` inside tests — cover admin, user,
  and visitor scopes explicitly
- Don't test generated CRUD plumbing (Repo.insert called, broadcast
  fired, etc.) — that's `ecto_context`'s job
- Don't use fixtures — factories produce valid structs with sensible defaults
- Don't use `defp` helpers for building scopes — use `setup`
- Don't place context tests flat in the namespace folder — mirror the lib path
- Don't ship a context change without its tests — "I'll add tests later"
  becomes "there are no tests"

## Direct `Repo` Usage Is a Bug

Never call `Platform.Repo` directly inside a context. `ecto_context`
handles scoping, authorization, changeset dispatch, and error normalization.
Bypassing it with `Repo.insert` / `Repo.update` / `Repo.delete` silently
skips all of that.

```elixir
# BUG — never
def generate(scope, name) do
  %McpApiKey{user_id: scope.user.id, name: name}
  |> Repo.insert()
end

# Correct — go through the macro-generated function
def generate(scope, name) do
  create(scope, %{name: name})
end
```

Legitimate `Repo` calls in a context:
- `Repo.one/1` / `Repo.all/1` for custom reads not covered by the macro
- `Repo.update_all/2` for bulk updates (e.g. `last_used_at` timestamps)
  where `update/3` would be wasteful
- Explicit transaction boundaries via `Repo.transact/2` (see below)

If you reach for `Repo.insert` or `Repo.update` on a single record, you
are using the wrong abstraction. Use `create/2` or `update/3`.

## Transactions: `Repo.transact/2`

For multi-step writes that must succeed or roll back together, use
**`Repo.transact/2`** — never `Repo.transaction/1`, never `Ecto.Multi`,
and never a hand-rolled wrapper.

```elixir
# Correct — Repo.transact with a zero-arity function returning {:ok, _} | {:error, _}
def publish_and_notify(scope, article_attrs) do
  Repo.transact(fn ->
    with {:ok, article} <- Articles.create(scope, article_attrs),
         {:ok, _}       <- Notifications.enqueue(scope, :published, article) do
      {:ok, article}
    end
  end)
end
```

Rules:

- Each step must return `{:ok, value}` or `{:error, reason}`. `transact`
  commits on `{:ok, _}`, rolls back on `{:error, _}`, and returns the
  lambda's value either way.
- Use `with` inside the lambda for sequential steps. The first `{:error, _}`
  short-circuits and rolls back the transaction.
- Don't use `Repo.transaction/1` — it requires `Repo.rollback/1` gymnastics
  and tuples the error value into `{:error, reason}` awkwardly.
- Don't use `Ecto.Multi` — `transact` + `with` is clearer, easier to read,
  and composes with our `{:ok, _} | {:error, _}` return shapes natively.
- Don't nest `transact` inside another `transact` — Ecto already handles
  savepoints, but nesting makes error flow hard to follow. Flatten it.
