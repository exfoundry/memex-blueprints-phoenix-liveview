# Migrations Guide

## SQLite version: 3.51.2

Supports `DROP COLUMN` (3.35+) and `RENAME COLUMN` (3.25+). Column type changes and adding constraints to existing columns are **not supported** — avoid both.

## Hard rules

- **Never run `mix ecto.reset`** in dev — the dev database has irreplaceable test data. Forward-only. `MIX_ENV=test mix ecto.reset` is fine.
- **`ON DELETE CASCADE` is fine** — but not when the referenced table needs to be dropped in a migration. SQLite cannot drop a table that other tables reference with a foreign key constraint, even with cascade. Remove the FK constraints first, drop the table, then recreate if needed.
- **No `validate: false`** — this is a PostgreSQL feature. `Ecto.Migration.constraint/3` uses `ALTER TABLE ADD CONSTRAINT` which SQLite does not support at all.
- **Check constraints only at column creation** — add them inline on new columns only:
  ```elixir
  add :status, :string, check: %{name: "status_valid", expr: "status IN ('active', 'inactive')"}
  ```
  For existing columns, use app-level validation or a trigger instead.

## Adding a NOT NULL column to an existing table

SQLite has no deferred constraint validation. The safe pattern:

1. **Migration 1:** Add as nullable
   ```elixir
   alter table(:companies) do
     add :currency, :string
   end
   ```
2. **Data migration:** Backfill in the same migration or via a separate `Repo.update_all`
   ```elixir
   execute "UPDATE companies SET currency = 'USD' WHERE currency IS NULL"
   ```
3. **Leave as nullable** — enforce NOT NULL at the app level (changeset `validate_required`). Adding the constraint later requires a full table recreation — not worth it.

## Backwards-compatible deploys (Kamal)

The old/new container overlap is only seconds, so compatibility requirements are minimal. Still worth following:

- **Adding a column:** Always safe — old code ignores new columns.
- **Removing a column:** Drop the code reference first, deploy, then drop the column in a follow-up migration.
- **Renaming a column:** Add the new name, migrate data, update code, deploy, drop the old column — all as separate steps.

## Migration organisation

Always generate migration files with `mix ecto.gen.migration migration_name_using_underscores` — never create them manually. This ensures the correct timestamp prefix and file naming conventions are applied.

One migration file per change. The current state (one file per context) reflects squashing — not the default. New changes always get their own migration file. Squashing periodically folds them back into the context file.

## Migration squashing

**Only on explicit request from Sascha** — never initiate a squash autonomously. Whether migrations have run everywhere is not knowable from the codebase.

To squash: fold all `alter table` changes back into the original `create table` in the context's migration file. **Always reuse the existing timestamp — never create a new one.** A new timestamp causes Ecto to treat it as an unrun migration and attempt to run it again.

## Counter caches

Use triggers to maintain counter cache columns. Three triggers needed: after insert, before delete, before update (to forbid changing the FK):

```elixir
execute """
  CREATE TRIGGER event_schedules_after_insert_row_trigger AFTER INSERT ON event_schedules
  FOR EACH ROW
  BEGIN
    UPDATE events SET event_schedules_count = (event_schedules_count + 1) WHERE id = NEW.event_id;
  END
""",
"DROP TRIGGER IF EXISTS event_schedules_after_insert_row_trigger"

execute """
  CREATE TRIGGER event_schedules_before_delete_row_trigger BEFORE DELETE ON event_schedules
  FOR EACH ROW
  BEGIN
    UPDATE events SET event_schedules_count = (event_schedules_count - 1) WHERE id = OLD.event_id;
  END
""",
"DROP TRIGGER IF EXISTS event_schedules_before_delete_row_trigger"

execute """
  CREATE TRIGGER event_schedules_before_update_row_trigger BEFORE UPDATE ON event_schedules
  FOR EACH ROW
  BEGIN
    SELECT IIF(NEW.event_id <> OLD.event_id, RAISE(FAIL, 'Changes to event_schedules#event_id are forbidden'), true);
  END
""",
"DROP TRIGGER IF EXISTS event_schedules_before_update_row_trigger"
```

The update trigger guards against FK changes that would corrupt the count. Always provide `DROP TRIGGER IF EXISTS` as the `down` argument to `execute/2`.

When adding a counter cache to a table with existing data, backfill before creating the triggers:

```elixir
execute """
  UPDATE events SET event_schedules_count = (
    SELECT COUNT(*) FROM event_schedules WHERE event_schedules.event_id = events.id
  )
"""
```

## Triggers (for constraints on existing columns)

When a constraint on an existing column is necessary, use a trigger rather than a schema constraint:

```elixir
execute """
  CREATE TRIGGER enforce_status_valid
  BEFORE INSERT ON companies
  BEGIN
    SELECT RAISE(ABORT, 'invalid status')
    WHERE NEW.status NOT IN ('active', 'inactive');
  END
""",
"""
  DROP TRIGGER IF EXISTS enforce_status_valid
"""
```

Always provide the `down` SQL as the second argument to `execute/2`.
