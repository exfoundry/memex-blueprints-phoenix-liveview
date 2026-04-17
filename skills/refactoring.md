# Refactoring Guide: Renaming Modules and Functions

## Context

Module and function renaming is frequent as the codebase evolves. Manual renaming across many files is slow and error-prone. Two Mix tasks cover the most common operations as single commands.

## Decision

Two tasks: `mix refactor.rename_module` for module renames (AST + string literals + file move), `mix igniter.refactor.rename_function` for function renames (built into Igniter via `ex_refactor` path dep).

Always run `--dry-run` first, then `--yes`. Always run `mix test` after to catch any references the task missed.

## Do

- Run `--dry-run` first to preview the file move
- Run `mix test` immediately after applying — the compiler surfaces missed references
- Use `mix igniter.refactor.rename_function Old.mod/fun New.mod/fun` with arity pinning for precision

## Don't

- Don't rely on either task for dynamic dispatch via `apply/3` — fix those manually
- Don't skip `mix test` after renaming — the compiler is the final validator

## Examples

Rename a module:

```
mix refactor.rename_module Old.Module.Name New.Module.Name --dry-run
mix refactor.rename_module Old.Module.Name New.Module.Name --yes
mix test
```

Rename a function:

```
mix igniter.refactor.rename_function OldModule.old_fun/2 NewModule.new_fun/2
```

**Known false positive — short alias collision:**
When a file uses `alias Foo.Bar` and you rename an unrelated `Some.Other.Bar`, the short alias guard may fire and incorrectly rename `Bar.*` call sites. Symptom: compiler warning about `SomeNewName.function/1 is undefined`. One-line fix — the compiler catches it immediately.

**Igniter lessons for custom task authors:**
- Batch all AST transforms for a file into a single `update_elixir_file` callback — multiple calls are silently discarded after the first
- Don't combine `include_all_elixir_files` with `update_all_elixir_files` — they share the same internal flag
- String literals need a separate text pass via `Rewrite.Source.update`
- Grep for affected files first: `grep -rl OldModule lib/ test/ config/`
- Test module naming: build with `List.update_at(parts, -1, &(&1 <> "Test"))` — `Module.concat(Foo, Test)` produces `Foo.Test` (wrong)
