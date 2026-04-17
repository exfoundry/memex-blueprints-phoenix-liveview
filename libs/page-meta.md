# PageMeta Pattern

## Context

Every LiveView needs a page title, a path for nav active state, and breadcrumb context. Without a standard these are scattered across templates and layouts. `PageMeta` centralises this into a single struct with compiler-enforced implementation — forgetting `page_meta/2` is a compiler warning, not a runtime crash.

## Decision

Every LiveView must implement `page_meta/2`. The `live_view` macro in `PlatformWeb` adds `@behaviour PlatformWeb.PageMeta` automatically and imports `assign_page_meta/1`. No exceptions.

The struct:

```elixir
@enforce_keys [:title, :path]
defstruct [
  :title,            # used in <title> tag
  :path,             # used for nav active state
  :breadcrumb_title, # optional shorter title for breadcrumb display
  :parent,           # parent PageMeta — builds breadcrumb trail recursively
  :description,
  canonical_path: nil,
  noindex: false,
  modal: false
]
```

Breadcrumbs are built by walking the `parent` chain via `PageMeta.breadcrumbs/1` — no separate configuration, each LiveView declares its own place in the hierarchy.

## Do

- Implement `page_meta/2` on every LiveView — one clause per `live_action`
- Set all assigns referenced in `page_meta/2` before calling `assign_page_meta/1` in the mount pipeline — order matters
- Reference the parent LiveView's `page_meta/1` for the parent breadcrumb rather than duplicating the struct
- Use `breadcrumb_title` when the full title is too long for a breadcrumb

## Don't

- Don't call `assign_page_meta/1` before the assigns it depends on are set
- Don't duplicate parent page meta structs inline — call the parent module's `page_meta/1`

## Examples

Simple index:

```elixir
@impl true
def page_meta(_socket, :index),
  do: %PageMeta{
    title: "Articles",
    path: ~p"/articles",
    parent: PlatformWeb.HomeLive.page_meta(_socket, :index)
  }
```

Show page with dynamic title and breadcrumb:

```elixir
@impl true
def page_meta(socket, :show),
  do: %PageMeta{
    title: socket.assigns.article.title,
    path: ~p"/articles/#{socket.assigns.article.slug}",
    parent: page_meta(socket, :index)
  }
```

Nested page with shorter breadcrumb title:

```elixir
@impl true
def page_meta(socket, :blueprint_versions),
  do: %PageMeta{
    title: "Blueprint Versions — #{socket.assigns.blueprint.title}",
    breadcrumb_title: "Blueprint Versions",
    path: ~p"/blueprints/#{socket.assigns.blueprint.slug}/blueprint_versions",
    parent: page_meta(socket, :show)
  }
```

Assigning in mount — assigns must come before `assign_page_meta/1`:

```elixir
def mount(params, _session, socket) do
  {:ok,
   socket
   |> assign(:article, Articles.get!(scope, params["id"]))
   |> assign_page_meta()}
end
```

Passing to layout:

```heex
<Layouts.app flash={@flash} page_meta={@page_meta}>
```

See also: [[components]]
