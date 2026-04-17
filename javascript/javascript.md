# TypeScript in Phoenix: Implementation & Testing Guide

## Context

All client-side code is TypeScript. esbuild handles the bundle and strips types at build
time — no `tsc` compilation step in the pipeline. TypeScript exists purely for editor
tooling, autocomplete, and catching errors early. `tsc --noEmit` validates types separately.

## Setup

- **esbuild** — bundles and strips types (no emit, no transpile step needed)
- **tsc** — type-checks only via `npm run typecheck` (`noEmit: true`)
- Entry point: `assets/js/app.js` (plain JS; imports `.ts` files freely)
- `@` alias maps to `assets/` — use `@/js/lib/...` for cross-directory imports
- `tsconfig.json`: `strict: true`, `moduleResolution: bundler`, targets ES2022 + DOM

## Do

- Write all new files as `.ts`
- Use interfaces and types freely — esbuild strips them at zero cost
- Use the `@` alias for imports that cross directory boundaries
- Keep shared, reusable logic in `assets/js/lib/` — importable from anywhere esbuild processes
- Use `export type` for type-only exports to make intent explicit

## Don't

- Don't write TypeScript syntax inside colocated hook `<script>` tags — those are written to
  `.js` files by `phoenix-colocated` and esbuild processes them as JavaScript. See [[components]]
  for the colocated hook pattern and the recommended extract-to-lib workaround
- Don't rely on `tsc` for build correctness — it only type-checks, never emits

## Directory structure

```
assets/js/
  app.js          # entry point (plain JS — imports .ts freely)
  hooks/          # external hooks: complex TS with imports, kept external when needed
  lib/
    lwc/          # Lightweight Charts domain logic: typed, tested, importable from hooks
```

## Testing

### Stack

- **Vitest** — test runner
- Test files: `js/**/*.test.ts`, co-located next to source
- Environment: Node (no DOM — `jsdom` not configured)
- Run via Mix: `mix test.js` (preferred) or `npm test` inside `assets/`
- `mix test.all` runs JS tests, TypeScript type checking, and Elixir tests — use this to
  ensure colocated hooks haven't broken any component behaviour and types are sound

If testing is not yet set up in the project, see [[testing-setup]] for the full setup:
Vitest, tsconfig, package.json scripts, and mix.exs aliases.

### Testing — Do

- Test pure functions and logic modules directly — no mocks needed for most `lib/` code
- Co-locate test files: `series-chart.ts` → `series-chart.test.ts`
- Use `describe` per function, `it` per case
- Build minimal mock objects with `vi.fn()` — only stub what the function under test actually calls
- Use factory functions over `beforeEach` shared state

### Testing — Don't

- Don't test DOM manipulation or hook lifecycle — no jsdom configured
- Don't test factory functions that create real chart/DOM instances — test the extracted
  pure functions they delegate to instead

### Mock pattern for object APIs

```ts
function makeChartMock() {
  const setVisibleRange = vi.fn()
  const timeScaleObj = { setVisibleRange, fitContent: vi.fn() }
  const timeScale = vi.fn(() => timeScaleObj)
  return { chart: { timeScale } as unknown as IChartApi, setVisibleRange }
}

it("sets range for '3m'", () => {
  const { chart, setVisibleRange } = makeChartMock()
  applyRange(chart, "3m")
  expect(setVisibleRange).toHaveBeenCalledOnce()
  const { from, to } = setVisibleRange.mock.calls[0]![0]
  expect(from < to).toBe(true)
})
```

### Imports

```ts
import { describe, it, expect, vi } from "vitest"
```
