# PageMeta Pattern

## Context

Every LiveView needs a page title, a path for nav active state, and breadcrumb context. Without a standard these scatter across templates and layouts. The [`phoenix_page_meta`](https://github.com/exfoundry/phoenix_page_meta) library centralises this into a single struct with compile-time field validation — forgetting `:title`, `:path`, or `:parent` is a `mix compile` error, not a runtime crash.

## Decision

The struct lives in `PlatformWeb.PageMeta`, defined explicitly via `defstruct` (no macro hides the fields). `use PhoenixPageMeta` injects the wiring around it: behaviour, helpers, default `base_url/0` and `lang_path/2`, and `@after_compile` field validation.

```elixir
defmodule PlatformWeb.PageMeta do
  use PhoenixPageMeta

  @enforce_keys [:title, :path]
  defstruct [
    :title,
    :path,
    :breadcrumb_title,
    :parent,
    :description,
    :og_image,
    :json_ld,
    :canonical_path,
    :skip_breadcrumb,
    og_type: "website",
    noindex: false,
    supported_locales: [:en, :es]
  ]
end
```

No `config.exs` entry. No `Application` module. The components dispatch site-wide callbacks (`base_url/0`, `lang_path/2`) via `page_meta.__struct__` — the struct itself carries the dispatch target.

Every LiveView implements `page_meta/2` (`@behaviour PhoenixPageMeta.LiveView`). The `live_view` macro in `PlatformWeb` adds the behaviour and imports `assign_page_meta/1` automatically.

`page_meta.path` includes the locale prefix (`/en/locations/123`). The macro's default `lang_path/2` does locale-prefix swap, which works for boqueteya's routing without override. `base_url/0` auto-detects `PlatformWeb.Endpoint.url()` from the `PlatformWeb.PageMeta` namespace.

## Library module structure

```
PhoenixPageMeta                          # __using__ macro, active?/2,3 (lib-level)
PhoenixPageMeta.Breadcrumb               # struct + build/1
PhoenixPageMeta.Components.Breadcrumbs   # list/1 component (slot-based, a11y)
PhoenixPageMeta.Components.MetaTags      # default/1 component (SEO tags)
PhoenixPageMeta.Site                     # behaviour: base_url/0, lang_path/2
PhoenixPageMeta.LiveView                 # behaviour: page_meta/2 + assign_page_meta/1
```

## Do

- Implement `page_meta/2` on every LiveView — one clause per `live_action`
- Set all assigns referenced in `page_meta/2` before calling `assign_page_meta/1` in the mount pipeline — order matters
- Reference the parent LiveView's `page_meta/2` for the parent breadcrumb rather than duplicating the struct
- Use `:breadcrumb_title` when the full title is too long for a breadcrumb
- Use `:skip_breadcrumb: true` on modal or overlay routes — the breadcrumb then shows the underlying page as current
- Use `:canonical_path` when the same content lives at multiple paths (filtered listings, pagination)

## Don't

- Don't call `assign_page_meta/1` before the assigns it depends on are set
- Don't duplicate parent page meta structs inline — call the parent module's `page_meta/2`
- Don't render breadcrumbs by hand. Use `<Breadcrumb.default page_meta={@page_meta} />` (project wrapper) which delegates to `<PhoenixPageMeta.Components.Breadcrumbs.list>` with project styling and home-icon-on-first behaviour
- Don't render SEO meta tags by hand. Use `<MetaTags.default page_meta={@page_meta} />` (aliased to `PhoenixPageMeta.Components.MetaTags`) in `root.html.heex`
- Don't add `config :phoenix_page_meta, ...`. The library reads no config

## Active-state helpers

Two project helpers in `PlatformWeb.Layouts` bridge to `PhoenixPageMeta.active?/3`, handling locale-prefix stripping:

- `is_active?(page_meta, path, :exact | :prefix)` — used in admin sidebar
- `match_path?(page_meta, path)` — alias for `is_active?(_, _, :prefix)`, used in main navigation

Both accept paths *without* locale prefix; the helpers prepend it before calling `PhoenixPageMeta.active?/3`. Slash-boundary matching is inherited from the package (`/locations` matches `/locations/123` but not `/location-foo`).

For direct calls in templates: `PageMeta.active?(@page_meta, ~p"/...")` works thanks to the `defdelegate`-style wrappers injected by `use PhoenixPageMeta`.

## Examples

Simple index:

```elixir
@impl PhoenixPageMeta.LiveView
def page_meta(_socket, :index),
  do: %PageMeta{
    title: "Articles",
    path: ~p"/#{@locale}/articles",
    parent: PlatformWeb.HomeLive.page_meta(_socket, :index)
  }
```

Show page with dynamic title:

```elixir
@impl PhoenixPageMeta.LiveView
def page_meta(socket, :show),
  do: %PageMeta{
    title: socket.assigns.article.title,
    path: ~p"/#{socket.assigns.locale}/articles/#{socket.assigns.article.slug}",
    parent: page_meta(socket, :index)
  }
```

Modal route that should not push to the breadcrumb:

```elixir
@impl PhoenixPageMeta.LiveView
def page_meta(socket, :edit),
  do: %PageMeta{
    title: "Edit Location — #{socket.assigns.location.name}",
    path: ~p"/#{socket.assigns.locale}/locations/#{socket.assigns.location.slug}/edit",
    parent: page_meta(socket, :show),
    skip_breadcrumb: true
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