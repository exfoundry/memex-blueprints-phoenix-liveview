# Migration Squashing

## Context

Migrations accumulate over time — each feature adds its own `alter table` file. Left unchecked, the migration history becomes a long chain of incremental changes that's hard to read and slow to replay. Squashing folds addendum migrations back into their context files, keeping the history clean.

## Decision

Migrations are organised into two categories: context files and addendum files. Squashing folds addendums back into context files. Always use the `/squash-migrations` skill — never squash manually or autonomously.

## File categories

**Context files** — one per domain context, named after what they create:
```
20260201000000_create_markets_tables.exs
20260201000001_create_accounts_tables.exs
```
These are the squash targets. They contain the canonical `create table` definitions for their context.

**Addendum files** — subsequent migrations that modify a context's tables:
```
20260315200000_add_serie_source_id_to_series.exs
20260320000000_remove_deprecated_status_from_companies.exs
```
These get folded back into their context file during squashing.

## Transformation rules

When folding an addendum into a context file:

| Addendum operation | Action in context file |
|---|---|
| `add :col` | Add column to the `create table` block |
| `remove :col` | Remove column from `create table` |
| `rename column :old, :new` | Update the column name in `create table` |
| `create index` | Add after the table definition |
| `drop index` | Remove the index from the context file |

**Data transformations cannot be folded** — any migration containing `execute`, `Repo.update_all`, or other data operations must stay as a separate file. Flag these and skip them.

## Constraints

- **Never create a new timestamp** — always reuse the context file's existing timestamp. A new timestamp causes Ecto to treat it as an unrun migration and attempt to run it again.
- **Never delete an addendum file** before the context file has been successfully written and verified.
- **Never squash without explicit confirmation** at each step — whether migrations have run everywhere is not knowable from the codebase.
- **Never squash autonomously** — only on explicit request.

## Verification

After squashing, always run:
```bash
MIX_ENV=test mix ecto.reset
MIX_ENV=test mix test
```

A clean test run confirms the squashed migrations reproduce the correct schema.

See also: [[migrations-sqlite]]
