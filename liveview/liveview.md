# LiveView: Implementation & Testing Guide

Injected automatically for `*_live.ex` files. Follow these rules.

## Module Declaration

```elixir
defmodule PlatformWeb.Articles.ArticleLive do
  use PlatformWeb, :live_view
  # Adds @behaviour PlatformWeb.PageMeta and imports assign_page_meta/1
end
```

## Function Order

Always follow this order — never rearrange:

1. `page_meta/2`
2. `render/1`
3. `mount/3`
4. `handle_params/3`
5. `handle_event/3`
6. `handle_info/2`
7. Private functions (`defp`)

## PageMeta (mandatory)

Every LiveView implements `page_meta/2` — one clause per `live_action`. Missing it is a compiler warning.

```elixir
@enforce_keys [:title, :path]
defstruct [
  :title,            # <title> tag
  :path,             # nav active state
  :breadcrumb_title, # shorter label for breadcrumb display
  :parent,           # parent PageMeta — builds breadcrumb trail recursively
  :description,
  canonical_path: nil,
  noindex: false,
  modal: false
]
```

Breadcrumbs are built by walking the `parent` chain — each LiveView declares its own place in the hierarchy:

```elixir
@impl true
def page_meta(_socket, :index),
  do: %PageMeta{title: "Articles", path: ~p"/articles"}

@impl true
def page_meta(socket, :show),
  do: %PageMeta{
    title: socket.assigns.article.title,
    path: ~p"/articles/#{socket.assigns.article.slug}",
    breadcrumb_title: "Article",
    parent: page_meta(socket, :index)
  }
```

## render/1

Always wrap content in a layout. Layouts are never applied automatically — always declare explicitly:

```heex
def render(assigns) do
  ~H"""
  <Layouts.app flash={@flash} page_meta={@page_meta}>
    <PageHeader.default page_meta={@page_meta} />
    ...
  </Layouts.app>
  """
end
```

Available layouts: `Layouts.app` (public), `Layouts.admin` (sidebar).

## HEEx Interpolation

Use `{...}` for tag attributes and inline values within tag bodies. Use `<%= ... %>` for block constructs (`if`, `case`, `cond`, `for`) in tag bodies. Mixing these up causes a compile-time crash.

```heex
<%!-- Correct --%>
<div id={@id}>
  {@my_assign}
  <%= if @condition do %>
    {@nested_assign}
  <% end %>
</div>

<%!-- Wrong — both crash at compile time --%>
<div id="<%= @id %>">
  {if @condition do}
  {end}
</div>
```

When rendering literal `{` or `}` characters (code snippets, JSON), annotate the parent tag with `phx-no-curly-interpolation`. Dynamic `<%= ... %>` expressions still work inside it:

```heex
<code phx-no-curly-interpolation>
  let obj = {key: "val"}
</code>
```

## mount/3

Set all assigns before calling `assign_page_meta/1` — it reads from assigns:

```elixir
def mount(params, _session, socket) do
  {:ok,
   socket
   |> MountScope.call(session)
   |> assign(:articles, Articles.list(scope))
   |> assign_page_meta()}
end
```

Reusable mount logic lives in `mounts/` — one file per concern (`mount_scope.ex`, `mount_subscriptions.ex`).

## Data Access and Scope

All data access goes through the context. The `scope` assign (set by `MountScope`) carries the current user and access level. Pass it to every context call:

```elixir
articles = Articles.list(scope)
article  = Articles.get!(scope, id)
```

A direct `Repo` call in a LiveView is always a bug. Enqueueing an Oban worker directly in a LiveView is also a bug — the context owns that decision:

```elixir
# Bad — LiveView knows about workers
alias Platform.Fundamentals.Documents.Workers.PdfConverterObanWorker
Oban.insert!(PdfConverterObanWorker.new(%{document_id: document.id}))

# Good — context owns the implementation detail
Documents.enqueue_conversion(document)
```

## Skinny LiveView, Fat Context

Business logic belongs in the context, not the LiveView. `handle_event` should call a context function and assign the result — nothing more.

```elixir
# Bad — logic in the LiveView
def handle_event("publish", _, socket) do
  now = DateTime.utc_now()
  article = Repo.get!(Article, socket.assigns.article.id)
  Repo.update!(Ecto.Changeset.change(article, published_at: now))
  {:noreply, assign(socket, :article, %{article | published_at: now})}
end

# Good — logic in the context
def handle_event("publish", _, socket) do
  {:ok, article} = Articles.publish(socket.assigns.scope, socket.assigns.article)
  {:noreply, assign(socket, :article, article)}
end
```

If a `handle_event` is growing logic, that logic belongs in a context function.

## Form LiveView (new + edit)

A single `FormLive` handles both `:new` and `:edit` via `apply_action`. Folder mirrors the URL path:

```
live/admin/companies/form_live.ex  →  /admin/companies/new
                                       /admin/companies/:id/edit
```

```elixir
defmodule PlatformWeb.Admin.Companies.FormLive do
  use PlatformWeb, :live_view

  embed_templates "_form_partials/*"  # large forms split into sibling partials

  @impl true
  def page_meta(_socket, :new),
    do: %PageMeta{title: "New Company", path: ~p"/admin/companies/new",
                  parent: PlatformWeb.Admin.Companies.IndexLive.page_meta(socket, :index)}

  @impl true
  def page_meta(socket, :edit),
    do: %PageMeta{title: "Edit #{socket.assigns.company.name}",   # reads assign — set before assign_page_meta
                  path: ~p"/admin/companies/#{socket.assigns.company}/edit",
                  parent: PlatformWeb.Admin.Companies.ShowLive.page_meta(socket, :show)}

  @impl true
  def mount(params, _session, socket) do
    {:ok,
     socket
     |> apply_action(socket.assigns.live_action, params)  # load data first
     |> assign_page_meta()}                               # page_meta reads assigns — must come last
  end

  defp apply_action(socket, :new, _params) do
    company = %Company{}
    socket
    |> assign(:company, company)
    |> assign(:form, to_form(Companies.change(scope, company)))
  end

  defp apply_action(socket, :edit, %{"id" => id}) do
    company = Companies.get!(scope, id)
    socket
    |> assign(:company, company)
    |> assign(:form, to_form(Companies.change(scope, company)))
  end

  @impl true
  def handle_event("save", %{"company" => params}, socket),
    do: save_company(socket, socket.assigns.live_action, params)

  defp save_company(socket, :new, params) do
    case Companies.create(scope, params) do
      {:ok, company}              -> {:noreply, socket |> put_flash(:info, "Created.") |> push_navigate(to: ~p"/admin/companies/#{company}")}
      {:error, %Changeset{} = cs} -> {:noreply, assign(socket, form: to_form(cs))}
    end
  end

  defp save_company(socket, :edit, params) do
    case Companies.update(scope, socket.assigns.company, params) do
      {:ok, company}              -> {:noreply, socket |> put_flash(:info, "Updated.") |> push_navigate(to: ~p"/admin/companies/#{company}")}
      {:error, %Changeset{} = cs} -> {:noreply, assign(socket, form: to_form(cs))}
    end
  end
end
```

Cancel navigates back via the breadcrumb parent — no hardcoded paths:

```heex
<.link navigate={@page_meta.parent.path} class="btn btn-ghost btn-sm font-mono">Cancel</.link>
```

## Components

No `CoreComponents` — it doesn't exist. Use `PlatformWeb.Components.*`. **Never hand-roll a card, table, or detail list** — the components already exist.

**Card** — always wrap page content sections:

```heex
<Card.default>
  <:header count={@count} add_href={~p"/admin/articles/new"}>Articles</:header>
  <:body>
    ...
  </:body>
</Card.default>
```

**Table** — never build a raw `<table>`:

```heex
<Table.default id="articles" rows={@articles} row_click={&JS.navigate(~p"/admin/articles/#{&1}")}>
  <:col label="Title"><%= @article.title %></:col>
  <:col label="State"><%= @article.state %></:col>
  <:action>
    <.link navigate={~p"/admin/articles/#{@article}/edit"}>Edit</.link>
  </:action>
</Table.default>
```

**Detail.list** — for show pages (label/value pairs), never bare `<dl>` or `<div>`:

```heex
<Detail.list>
  <:item title="Name"><%= @company.name %></:item>
  <:item title="Country"><%= @company.country %></:item>
</Detail.list>
```

**Other components:**

```heex
<Form.input field={@form[:title]} type="text" label="Title" />
<Button.default variant="primary" phx-disable-with="Saving...">Save<Button.default>
<Icon.hero name="hero-x-mark" class="size-4" />
<Tooltip.info tip="Calculated from records" />
<PageHeader.default page_meta={@page_meta} />
```

**Tailwind classes must be full literal strings** — the compiler can't detect dynamic ones:

```elixir
# Bad
"bg-#{color}/10"

# Good
defp row_class("active"),   do: "bg-success/10"
defp row_class("inactive"), do: "bg-error/10"
```

## Do

- Follow the function order: `page_meta` → `render` → `mount` → `handle_params` → `handle_event` → `handle_info` → private
- Implement `page_meta/2` with one clause per `live_action`
- Always wrap `render/1` in `Layouts.app` or `Layouts.admin`
- Set all assigns before calling `assign_page_meta/1`
- Pass `scope` to every context call
- Keep `handle_event` thin — delegate logic to the context

## Don't

- Don't reorder `page_meta`, `render`, `mount`
- Don't call `Repo` directly — it's always a bug in a LiveView
- Don't use `CoreComponents` — it doesn't exist
- Don't put business logic in `handle_event` — it belongs in the context
- Don't call `assign_page_meta/1` before the assigns it reads are set
- Don't build Tailwind classes with string interpolation
- Don't use `Scope.global_access()` in a LiveView — this is a bug, it bypasses user context entirely
- Don't use `Scope.global_access()` inside a context function — context functions never create a scope, callers always provide one. Oban workers and plugs that need global access pass `Scope.global_access()` at the call site.

## Do (for public access)

When a public action needs to be allowed for non-admin scopes, add a matching `permission/3` clause in the context (e.g. `def permission(:create, _record, _scope), do: true`) and use `current_scope` in the LiveView as normal.

## Testing

Every LiveView must have a test.

### Context

LiveView smoke tests are the highest-value tests per line of code. They catch nil preloads, missing assigns, broken component calls, and routing errors in a single pass. Every LiveView must have at least one smoke test per action — a LiveView without a test is considered incomplete.

### Decision

Use `Phoenix.LiveViewTest` with `ConnCase`. One `describe` block per live action, using the action atom as the label. Never `async: true` in ConnCase — SQLite sandbox does not support concurrent connection ownership.

Form tests cover all states in one test: navigate to the form, trigger validation errors via `render_change`, submit valid attrs, follow the redirect, assert on the result.

No authorization tests by default. [[context]] scopes all queries — unauthorized access returns empty results, not errors. Admin areas are protected by a shared mount. Only write authorization tests when there is custom logic beyond these defaults.

### Scopes in Tests

Never use `Scope.global_access()` in LiveView tests. Log in as the appropriate actor — visitor (not logged in), regular user, or admin — using the conn helpers:

```elixir
setup %{conn: conn} do
  admin = Factory.insert(:user, admin: true)
  user  = Factory.insert(:user, admin: false)

  %{
    conn:       conn,
    admin_conn: log_in_user(conn, admin),
    user_conn:  log_in_user(conn, user)
  }
end
```

Use whichever actors the LiveView's access rules actually distinguish.

### Testing — Do

- Write at least one smoke test per live action
- Use the action atom as the describe label: `describe ":new" do`
- Set `async: false` on all ConnCase tests
- Cover form validation, submit, and redirect in a single test — not three separate tests
- Define `@create_attrs`, `@update_attrs`, `@invalid_attrs` as module attributes for clarity
- Assert on meaningful content — a name, a slug, a flash message — not just that the page renders

### Testing — Don't

- Don't use `async: true` in ConnCase — SQLite sandbox crashes with `:already_shared` under concurrency
- Don't split form states across multiple tests — one test per action covers the full happy path and the validation error
- Don't write authorization tests unless there is custom logic beyond EctoContext scopes or shared mount protection

### Form test example

Covering validation, submit, and redirect in one test:

```elixir
defmodule PlatformWeb.Blueprints.FormLiveTest do
  use PlatformWeb.ConnCase, async: false
  import Phoenix.LiveViewTest

  @create_attrs %{slug: "my-bp", title: "My Blueprint", summary: "A summary.", body: "Body."}
  @update_attrs %{title: "Updated Blueprint"}
  @invalid_attrs %{slug: nil, title: nil}

  describe ":new" do
    test "saves new blueprint", %{conn: conn} do
      {:ok, form_live, _html} = live(conn, ~p"/blueprints/new")

      assert form_live
             |> form("form", blueprint: @invalid_attrs)
             |> render_change() =~ "can&#39;t be blank"

      assert {:ok, _lv, html} =
               form_live
               |> form("form", blueprint: @create_attrs)
               |> render_submit()
               |> follow_redirect(conn, ~p"/blueprints/my-bp")

      assert html =~ "My Blueprint"
    end
  end

  describe ":edit" do
    test "updates blueprint", %{conn: conn} do
      Factory.insert(:blueprint, slug: "my-bp", title: "Old Title", summary: ".")

      {:ok, form_live, _html} = live(conn, ~p"/blueprints/my-bp/edit")

      assert form_live
             |> form("form", blueprint: @invalid_attrs)
             |> render_change() =~ "can&#39;t be blank"

      assert {:ok, _lv, html} =
               form_live
               |> form("form", blueprint: @update_attrs)
               |> render_submit()
               |> follow_redirect(conn, ~p"/blueprints/my-bp")

      assert html =~ "Updated Blueprint"
    end
  end
end
```

See also: [[page-meta]], [[components]], [[context]]
