# JavaScript Testing Setup

Step-by-step setup for Vitest + TypeScript in a Phoenix project.

## 1. Install dependencies

```bash
cd assets
npm install --save-dev vitest typescript
```

## 2. `assets/vitest.config.ts`

```ts
import { defineConfig } from "vitest/config"

export default defineConfig({
  test: {
    include: ["js/**/*.test.ts"],
  },
})
```

## 3. `assets/tsconfig.json`

```json
{
  "compilerOptions": {
    "allowJs": false,
    "baseUrl": ".",
    "lib": ["ES2022", "DOM"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "noEmit": true,
    "strict": true,
    "target": "ES2022"
  },
  "include": ["js/**/*"],
  "exclude": ["node_modules", "js/**/*.test.ts"]
}
```

Note: test files are excluded from `tsc` — Vitest handles them separately. The `baseUrl: "."`
combined with esbuild's `--alias:@=.` means the `@` import alias works consistently in both
the build and the type checker.

## 4. `assets/package.json` scripts

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "typecheck": "tsc --noEmit"
  }
}
```

## 5. `mix.exs` — aliases and preferred env

Add to `aliases/0`:

```elixir
"test.js":  ["cmd npm test --prefix assets"],
"test.all": ["test.js", "test"],
```

Add to `preferred_cli_env` in `project/0` so `MIX_ENV=test` is set when running `mix test.js`:

```elixir
preferred_cli_env: [
  "test.js": :test,
]
```

`mix test.all` is the recommended way to run everything — it runs JS tests, TypeScript type
checking, and Elixir tests in sequence. This catches type errors that esbuild silently ignores
at build time.

See also: [[javascript]]
