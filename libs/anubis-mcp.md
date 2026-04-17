# Anubis MCP: Implementation & Testing Guide

Library for building MCP (Model Context Protocol) servers inside Phoenix applications. Tools are
implemented as plain Elixir modules and registered with a central server process. The server
handles JSON-RPC, SSE/StreamableHTTP transport, and session management.

## Server Entry Point

One file at `lib/platform_mcp.ex`, alongside `lib/platform_web.ex`:

```elixir
defmodule PlatformMcp do
  use Anubis.Server,
    name: "miningcap",
    version: "1.0.0",
    capabilities: [:tools]

  component(Platform.Mcp.Tools.ListCompanies)
  component(Platform.Mcp.Tools.GetCompany)
end
```

Add each new tool with `component/1`, always passing an explicit `name:`. Anubis
derives names from the last module segment only (`Companies.Create` → `"create"`),
so without explicit names all tools in the same verb group collide. Convention:
`{domain}_{verb}` (e.g. `companies_create`, `persons_list`).

## Supervision

Add to `lib/platform/application.ex` children list, before the endpoint:

```elixir
{PlatformMcp, transport: :streamable_http},
PlatformWeb.Endpoint
```

## Router

```elixir
pipeline :mcp do
  plug :accepts, ["json"]
  plug PlatformWeb.Mcp.Plugs.AuthenticateMcpKey
end

scope "/mcp" do
  pipe_through :mcp
  forward "/", Anubis.Server.Transport.StreamableHTTP.Plug, server: PlatformMcp
end
```

## Tool Structure

Tools live in `lib/platform/mcp/tools/` in subdirectories matching the context namespace. One file per tool, namespace `Platform.Mcp.Tools.*`:

```
lib/platform/mcp/tools/
  companies/
    list_companies.ex
    get_company.ex
  mines/
    list_mines.ex
  documents/
    ...
```

```elixir
defmodule Platform.Mcp.Tools.ListCompanies do
  use Anubis.Server.Component, type: :tool

  alias Anubis.Server.Response
  alias Platform.Fundamentals.Companies

  schema do
    # declare input fields here; empty block = no params
  end

  def execute(%{} = _params, frame) do
    scope = frame.assigns.current_scope
    companies = Companies.list(scope, order_by: :name)
    text = format(companies)
    {:reply, Response.tool() |> Response.text(text), frame}
  end
end
```

For tools with parameters:

```elixir
schema do
  field :slug, :string,
    required: true,
    description: "The company slug to look up"
end

def execute(%{slug: slug}, frame) do
  scope = frame.assigns.current_scope
  company = Companies.get_by!(scope, slug: slug)
  {:reply, Response.tool() |> Response.text(format(company)), frame}
end
```

## Frame and Scope

`frame.assigns` inherits from `Plug.Conn.assigns` for HTTP transports. The auth plug assigns
`current_scope` to the conn, so every tool has it automatically:

```elixir
scope = frame.assigns.current_scope
```

Never hardcode `Scope.global_access()` in a tool — the scope comes from the authenticated request.
If a tool legitimately needs global access (e.g. read-only public data), the context's `scope/2`
clause should handle that, not the tool.

## Response Format

Plain text is the default for tabular or list output:

```elixir
{:reply, Response.tool() |> Response.text(text), frame}
```

For structured responses, build the text yourself — the tool decides the format.

## Auth

`PlatformWeb.Mcp.Plugs.AuthenticateMcpKey` handles Bearer token auth. It looks up the
hashed token in `mcp_api_keys`, resolves the user, and assigns `current_scope`. Returns 401
JSON on failure.

## Do

- One tool per file, one file per concern
- Keep `execute/2` thin — delegate to context functions
- Use `frame.assigns.current_scope` for all data access
- Return plain text — LLMs parse prose better than JSON blobs
- Add a `@moduledoc` describing what the tool returns and when to use it

## Don't

- Don't call `Repo` in tools — go through the context
- Don't hardcode `Scope.global_access()` in tools
- Don't put business logic in `execute/2` — it belongs in the context

## Testing

### Pattern

Test tools by calling `execute/2` directly — no HTTP, no server process, no LiveView. Pass a
`%Frame{}` with the assigns your tool reads, assert on the text content of the response.

```elixir
defmodule Platform.Mcp.Tools.ListCompaniesTest do
  use Platform.DataCase

  alias Anubis.Server.Frame
  alias Anubis.Server.Response
  alias Platform.Accounts.Users.Scope
  alias Platform.Factory
  alias Platform.Mcp.Tools.ListCompanies

  defp text_content(%Response{content: [%{"text" => text}]}), do: text

  setup do
    admin = Factory.insert(:user, admin: true)
    scope = Scope.for_user(admin)
    %{frame: %Frame{assigns: %{current_scope: scope}}, scope: scope}
  end

  describe "execute/2" do
    test "returns empty message when no data exists", %{frame: frame} do
      {:reply, response, _frame} = ListCompanies.execute(%{}, frame)
      assert text_content(response) =~ "No companies found"
    end

    test "lists companies with expected fields", %{frame: frame} do
      Factory.insert(:company, name: "Agnico Eagle", slug: "agnico-eagle", country: "CA")

      {:reply, response, _frame} = ListCompanies.execute(%{}, frame)
      text = text_content(response)

      assert text =~ "agnico-eagle"
      assert text =~ "Agnico Eagle"
      assert text =~ "CA"
    end
  end
end
```

### Key Points

**`async: false` required** — MCP tool tests use `DataCase` with SQLite's shared sandbox.
SQLite only supports one write transaction at a time; `async: true` causes concurrent
`start_owner!` calls to race and return `:already_shared`. Always use `use Platform.DataCase`
(no `async: true`).

**Frame construction** — build a frame with the assigns your tool reads. Always use a real user scope — never `Scope.global_access()`:
```elixir
admin = Factory.insert(:user, admin: true)
scope = Scope.for_user(admin)
frame = %Frame{assigns: %{current_scope: scope}}
```

**`text_content/1` helper** — always define this in the test module. Anubis wraps text in
a content list:
```elixir
defp text_content(%Response{content: [%{"text" => text}]}), do: text
```

**Positional assertions** — `String.index/2` doesn't exist in Elixir. Use `:binary.match/2`:
```elixir
{alpha_pos, _} = :binary.match(text, "Alpha Gold")
{zeta_pos, _}  = :binary.match(text, "Zeta Mining")
assert alpha_pos < zeta_pos
```

**Params** — pass params as a plain map matching the tool's `schema` fields:
```elixir
ListCompanies.execute(%{slug: "agnico-eagle"}, frame())
```

### File Location

Mirrors the lib path — subdirectory matches the context namespace:

```
lib/platform/mcp/tools/companies/list_companies.ex
→ test/platform/mcp/tools/companies/list_companies_test.exs
```

### What to Test

- Empty state ("No X found" message)
- Happy path with factory data — assert key fields appear in output
- Missing/nil optional fields render as expected (e.g. `—`)
- Sort order when ordering is part of the contract
- Scope filtering when the tool is user-scoped (create data across two users, assert only the right one appears)

### Testing — Don't

- Don't start the server process or make HTTP requests — call `execute/2` directly
- Don't test Anubis routing or transport — that's the library's job
- Don't use `ConnCase` for tool tests — `DataCase` is sufficient and faster
