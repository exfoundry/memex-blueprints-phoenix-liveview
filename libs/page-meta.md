# PageMeta Pattern

## Context

Every LiveView needs a page title, a path for nav active state, and breadcrumb context. Without a standard these scatter across templates and layouts. The [`phoenix_page_meta`](https://github.com/exfoundry/phoenix_page_meta) library centralises this into a single struct with compile-time behaviour binding — forgetting `page_meta/2` is a compile warning, not a runtime crash.

## Decision

The struct lives in `PlatformWeb.PageMeta`, defined explicitly via `defstruct` (no macro hides the fields). It implements `PhoenixPageMeta.Site` for site-wide concerns (`endpoint_url/0`, `lang_path/2`) and delegates `breadcrumbs/1` and `active?/2,3` to the package. The package binds to it at compile time:

```elixir
config :phoenix_page_meta, app_module: PlatformWeb.PageMeta
```

Every LiveView implements `page_meta/2` (`@behaviour PhoenixPageMeta.LiveView`). The `live_view` macro in `PlatformWeb` adds the behaviour and imports `assign_page_meta/1` automatically.

The struct (with boqueteya-specific defaults):

```elixir
@enforce_keys [:title, :path]
defstruct [
  :title,             # used in <title> tag and og:title
  :path,              # used for nav active state, canonical URL, hreflang
  :breadcrumb_title,  # optional shorter title for breadcrumb display
  :parent,            # parent PageMeta — builds breadcrumb trail recursively
  :description,
  :og_image,
  :json_ld,
  :canonical_path,
  :skip_breadcrumb,   # filter out of breadcrumb (modals, overlays)
  og_type: "website",
  noindex: false,
  supported_locales: [:en, :es]
]
```

`page_meta.path` includes the locale prefix (`/en/locations/123`). Site-wide helpers in `PlatformWeb.PageMeta` and `PlatformWeb.Layouts` strip it when needed.

## Library module structure

```
PhoenixPageMeta                          # active?/2,3, breadcrumbs/1 (delegate)
PhoenixPageMeta.Breadcrumb               # struct + build/1
PhoenixPageMeta.Components.Breadcrumbs   # list/1 component (slot-based, a11y)
PhoenixPageMeta.Components.MetaTags      # default/1 component (SEO tags)
PhoenixPageMeta.Site                     # behaviour: endpoint_url/0, lang_path/2
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
- Don't render breadcrumbs by hand. Use `<Breadcrumb.default page_meta={@page_meta} />` (project wrapper) which delegates to `<PhoenixPageMeta.Components.Breadcrumbs.list>` with project styling and home-icon-on-root behaviour
- Don't render SEO meta tags by hand. Use `<PhoenixPageMeta.Components.MetaTags.default page_meta={@page_meta} />` in `root.html.heex`

## Active-state helpers

Two project helpers in `PlatformWeb.Layouts` bridge to `PhoenixPageMeta.active?/3`, handling locale-prefix stripping:

- `is_active?(page_meta, path, :exact | :prefix)` — used in admin sidebar
- `match_path?(page_meta, path)` — alias for `is_active?(_, _, :prefix)`, used in main navigation

Both accept paths *without* locale prefix; the helpers prepend it before calling `PhoenixPageMeta.active?/3`. Slash-boundary matching is inherited from the package (`/locations` matches `/locations/123` but not `/location-foo`).

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