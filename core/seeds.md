# Seeds Guide

## Guard requirement

Every seeds file **must** wrap all seed logic in a test-env guard:

```elixir
if Application.get_env(:platform, :env) != :test do
  # ... seed data here
end
```

**Why:** Seeds run during `mix test` setup in some configurations. Without this guard, seed data leaks into the test database and causes non-deterministic failures — tests that pass alone but fail when the full suite runs.

## Structure

```elixir
# priv/repo/seeds.exs

if Application.get_env(:platform, :env) != :test do
  alias Platform.Repo
  alias Platform.SomeContext.SomeSchema

  Repo.insert!(%SomeSchema{...})
end
```

## Rules

- **No bare inserts at the top level** — all inserts live inside the guard.
- **Idempotent inserts preferred** — use `Repo.insert!` with `on_conflict: :nothing` or check-before-insert to make seeds re-runnable.
- **No test data in seeds** — seeds are for dev/prod baseline data only. Test fixtures belong in `test/support/`.
