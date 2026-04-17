# Controllers: Implementation Guide

## Context

Phoenix controllers in this project are **not** for rendering HTML. All
user-facing pages go through LiveView (see [[liveview]]). Controllers exist
for the cases LiveView cannot cover:

- JSON / text APIs for external clients (`api/v1/*`)
- XML endpoints (`sitemap`, RSS, etc.)
- Plain-text endpoints consumed by tooling (health, build info)
- Redirects (locale negotiation, canonical URL enforcement)
- Auth-session actions (login/logout form POSTs that need `put_session` /
  `configure_session`)

Injected for `lib/platform_web/controllers/*_controller.ex`.

## Use When

- The response is machine-readable (JSON, text, XML) — no HTML template
- The endpoint is a redirect with no render
- A POST must manipulate the session and then redirect (login/logout)
- External clients (LLM tooling, search engines, monitoring, cron) consume it
- File downloads / streamed binary responses

## Don't Use For

- Rendering HTML pages — always LiveView
- Multi-step user interactions — LiveView
- Anything with form validation feedback — LiveView forms

If you reach for `render(conn, :show, ...)` with a `.html.heex` template,
stop and make it a LiveView instead.

## Structure

```elixir
defmodule PlatformWeb.API.V1.LocationsController do
  use PlatformWeb, :controller

  alias Platform.Directory.Locations

  def index(conn, params) do
    scope = conn.assigns.current_scope
    locations = Locations.list(scope, order_by: :name, limit: parse_limit(params["limit"]))
    text(conn, LLMText.table(...))
  end

  def show(conn, %{"slug" => slug}) do
    scope = conn.assigns.current_scope

    case Locations.get_by(scope, slug: slug) do
      nil      -> send_error(conn, 404, "Location not found: #{slug}")
      location -> text(conn, location_document(location))
    end
  end

  # --- Private

  defp send_error(conn, status, message) do
    conn |> put_status(status) |> text(message)
  end
end
```

## Scope

- **Authenticated endpoints:** `scope = conn.assigns.current_scope` — set by
  the auth plug in the router pipeline.
- **Public endpoints** (sitemap, health, public API reads): build a visitor
  scope explicitly: `scope = Scope.for_user(nil)`. Never reach for
  `Scope.global_access/0` — public read is not the same as unscoped.
- All DB access goes through the context with the scope — never `Repo`
  directly, never bypass the context. Same rule as [[context]].

## Response Helpers

- `json(conn, map)` — JSON body, sets content-type
- `text(conn, string)` — plain text / pre-formatted content (LLM tables,
  documents, sitemap XML wrapped in `put_resp_content_type/2` first)
- `put_resp_content_type(conn, "text/xml")` before `text/2` for XML
- `redirect(conn, to: ~p"/...")` / `redirect(conn, external: url)`
- `send_resp(conn, status, body)` for raw responses
- `put_status(conn, code)` before the response helper for non-200 codes

## Actions

Mirror context function names where possible — `index`, `show`, `create`,
`update`, `delete`. Keep the controller thin: parse params, call the
context, render the result. No business logic in the controller.

```elixir
def create(conn, params) do
  scope = conn.assigns.current_scope
  attrs = Map.take(params, ["name", "slug", "category_id"])

  case Locations.create(scope, attrs) do
    {:ok, location} ->
      conn |> put_status(201) |> text(location_document(location))

    {:error, changeset} ->
      send_error(conn, 422, Formatter.format_errors(changeset))
  end
end
```

## Do

- Use controllers **only** for non-HTML responses (JSON, text, XML, redirect)
- Take scope from `conn.assigns.current_scope` (authed) or
  `Scope.for_user(nil)` (public); pass to the context
- Mirror context names in actions (`index`, `show`, `create`, ...)
- Extract private helpers (`location_document/1`, `send_error/3`,
  `parse_limit/1`) when the action body grows past ~15 lines
- Return proper HTTP status codes via `put_status/2` before the response
  helper — 201 on create, 404 on not-found, 422 on validation error
- Set `put_resp_content_type/2` before `text/2` for non-text-plain responses
  (XML, CSV, etc.)
- Keep the controller file under ~100 lines — if it grows past that, split
  by resource or move helpers into a sibling module

## Don't

- Don't render HTML — use LiveView instead
- Don't embed business logic — call the context and return its result
- Don't use `Scope.global_access/0` for public endpoints — build
  `Scope.for_user(nil)` explicitly
- Don't call `Repo` directly — everything goes through the context
- Don't reinvent pagination / filtering — the context's `list/2` opts
  (`limit:`, `order_by:`, `query:`) already cover it
- Don't swallow context errors — map them to the right HTTP status and
  return a useful body (via `Formatter.format_errors/1` for changesets)

## Testing

Controller tests use `PlatformWeb.ConnCase`. Assert the response body and
status; don't re-test the context — that's covered in [[context]] tests.

```elixir
defmodule PlatformWeb.API.V1.LocationsControllerTest do
  use PlatformWeb.ConnCase

  alias Platform.Factory

  describe "GET /api/v1/locations" do
    test "lists published locations as an LLM table", %{conn: conn} do
      Factory.insert(:location, name: "Cafe Sueños", published_at: DateTime.utc_now())

      conn = get(conn, ~p"/api/v1/locations")

      assert response = text_response(conn, 200)
      assert response =~ "Cafe Sueños"
    end
  end

  describe "POST /api/v1/locations" do
    test "returns 422 on validation error", %{conn: conn} do
      conn = post(conn, ~p"/api/v1/locations", %{"name" => ""})
      assert text_response(conn, 422) =~ "can't be blank"
    end
  end
end
```

- `text_response/2`, `json_response/2`, `redirected_to/2` — use the helper
  that matches the response type
- One `describe` per route (`GET /path`, `POST /path`)
- Don't test controller-internal helpers — they're private
- `async: false` for controllers that hit the DB (SQLite sandbox) — same
  rule as [[context]] tests
