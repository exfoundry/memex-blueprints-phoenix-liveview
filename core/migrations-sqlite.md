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

## PRAGMA in Migrations: Connection-Pinning

Ecto's Migration Runner holt sich für jeden `execute`-Call eine neue Connection aus dem Pool. PRAGMA-State ist in SQLite connection-scoped — `PRAGMA foreign_keys = OFF` oder `PRAGMA writable_schema = ON` in einem `execute` haben daher keinen Effekt auf den nächsten `execute`-Call.

Lösung: `@disable_ddl_transaction true` + `repo().checkout/1` um alle Statements auf dieselbe Connection zu pinnen:

```elixir
defmodule MyApp.Repo.Migrations.SomeSchemaFix do
  use Ecto.Migration

  @disable_ddl_transaction true

  def up do
    repo().checkout(fn ->
      repo().query!("PRAGMA foreign_keys = OFF")
      repo().query!("DROP TABLE ...")
      repo().query!("PRAGMA foreign_keys = ON")
    end)
  end
end
```

`@disable_ddl_transaction true` ist nötig, weil SQLite PRAGMA-Änderungen nicht innerhalb einer aktiven Transaktion erlaubt. `repo().checkout/1` garantiert, dass alle Queries innerhalb des Blocks auf derselben Connection landen.

## Spalten-Definitionen ändern mit writable_schema

SQLite unterstützt kein `ALTER COLUMN`. Für rein kosmetische Schemaänderungen — DEFAULT entfernen, NOT NULL Constraint hinzufügen oder entfernen, Constraint-Name anpassen — also alles **ohne Datenverschiebung** — kann `PRAGMA writable_schema` die CREATE TABLE-Definition in `sqlite_master` direkt anpassen:

```elixir
defmodule MyApp.Repo.Migrations.DropDefaultOnSomeColumn do
  use Ecto.Migration

  @disable_ddl_transaction true

  def up do
    repo().checkout(fn ->
      repo().query!("PRAGMA writable_schema = ON")

      repo().query!("""
      UPDATE sqlite_master
      SET sql = REPLACE(sql,
        '"some_column" TEXT DEFAULT ''fallback'' NOT NULL',
        '"some_column" TEXT NOT NULL'
      )
      WHERE type = 'table' AND name = 'my_table'
      """)

      repo().query!("PRAGMA writable_schema = OFF")

      # Schema-Cache invalidieren
      %{rows: [[v]]} = repo().query!("PRAGMA schema_version")
      repo().query!("PRAGMA schema_version = #{v + 1}")

      repo().query!("PRAGMA integrity_check")
    end)
  end
end
```

**String-Escaping in SQLite:** Einzelne Anführungszeichen innerhalb von SQL-String-Literalen werden mit `''` (doppelt) escaped, nicht mit `\'`. Im Elixir-Heredoc ist `''` zwei aufeinanderfolgende Quotes — SQLite interpretiert das als einen `'`-Charakter.

**Wann verwenden:** Nur für Änderungen die das physische Daten-Layout nicht berühren. Spalten-Reihenfolge kann damit **nicht** geändert werden — die Daten liegen positional auf Disk, der DDL-String ist nur eine Namens-Positions-Zuordnung. Falsches Reordering im DDL-String führt zu silent data corruption.

**Vor dem Einsatz prüfen:** Den exakten DDL-String mit `sqlite3 priv/db/boqueteya_dev.db "SELECT sql FROM sqlite_master WHERE name='my_table';"` verifizieren — der REPLACE muss exakt matchen.

**Nach dem Migrate:** `PRAGMA integrity_check` immer ausführen um Konsistenz zu bestätigen.