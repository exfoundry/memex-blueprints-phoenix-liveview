# Component Architecture: Namespaced Modules over CoreComponents

## Context

Phoenix generators produce a single `CoreComponents` module imported globally. This breaks down as the library grows — a 2000-line mixed file with no natural home for variants, hiding what a LiveView actually depends on. As the component library grows significantly (sortable tables, chart variants, JS-backed inputs), a scalable structure is needed from the start.

## Decision

Each component family lives in its own module under `PlatformWeb.Components.*`. Modules are aliased without renaming in `html_helpers`, making them available as `Table.default`, `Form.input`, `Card.default` etc. across all LiveViews. `CoreComponents` is deleted — there is no fallback import.

To get the current component inventory with all attrs and slots, run:

    mix memex.list_components

## Do

- Create one module per component family under `PlatformWeb.Components.*`
- Alias each module in `html_helpers` — no renaming
- Use `default` for the primary variant, named functions for semantic variants (`Table.grouped`, `Chart.tradingview`)
- Import `SharedHelpers` (not alias) — `show/hide` need no prefix noise
- Alias `Icon` explicitly in component modules that need it
- Embed client-side hooks using `<script :type={Phoenix.LiveView.ColocatedHook} name=".HookName">` — no manual `app.js` entry needed
- Document every public component with a `@doc`, attr/slot explanations, and at least one `## Examples` block

## Don't

- Don't use `CoreComponents` — it is deleted, there is no fallback
- Don't call context modules from component files — data flows in via assigns only
- Don't add a dynamic `variant` attr where a named function would be more explicit
- Don't construct Tailwind classes dynamically with string interpolation — the Tailwind compiler scans for complete class strings and won't find `"bg-#{color}"`. Every class must appear as a full literal string somewhere in the codebase
- Don't use `<%= ... %>` for tag attributes or inline values — use `{...}`. Don't use `{...}` for block constructs (`if`, `case`, `for`) — use `<%= ... %>`. Mixing these crashes at compile time
- Don't write literal `{` or `}` in template content without annotating the parent tag with `phx-no-curly-interpolation` (e.g. code snippets, JSON examples)

## Examples of Tailwind class handling

Wrong — Tailwind compiler can't detect these:

```elixir
# Bad
"bg-#{color}/10"
"text-#{status}"
```

Right — define all variants explicitly as complete strings:

```elixir
# Good — in the component or a helper function
defp color_class("success"), do: "bg-success/10 text-success"
defp color_class("error"),   do: "bg-error/10 text-error"
defp color_class("warning"), do: "bg-warning/10 text-warning"
```

Or use a map:

```elixir
@colors %{
  "success" => "bg-success/10 text-success",
  "error"   => "bg-error/10 text-error"
}
```


## Examples

```heex
<Card.default>
  <:header count={5}>Ownerships</:header>
  <:body class="overflow-hidden"><Table.default .../></:body>
</Card.default>

<PageHeader.default page_meta={@page_meta}>
  <:actions><Button.default phx-click="save">Save</Button.default></:actions>
</PageHeader.default>

<Table.grouped groups={@grouped} group_row_class={fn g -> g.color end}>
  <:group_header :let={g}>{g.name}</:group_header>
  <:col :let={row} label="Name">{row.name}</:col>
</Table.grouped>

<Icon.hero name="hero-x-mark" class="size-4" />
<Icon.flag country={@company.country} />
<Tooltip.info tip="Calculated from cost records" />
<Util.relative_time timestamp={@entry.updated_at} />
```

Adding a new component family:

1. Create `lib/platform_web/components/<n>.ex` as `PlatformWeb.Components.<n>`
2. `use Phoenix.Component` at the top
3. Add `alias PlatformWeb.Components.<n>` to `html_helpers` in `platform_web.ex`

## MapLibre components and phx-update="ignore"

Components that embed a MapLibre map (or any third-party library that writes its own children into the hook element) **must** add `phx-update="ignore"` to the container div.

**Why:** morphdom compares the server-rendered element (an empty `<div>`) with the client DOM (which has MapLibre's canvas, controls, etc. as children). Without `phx-update="ignore"`, morphdom removes those children on every server re-render, making the map disappear.

**Key behaviour of `phx-update="ignore"`:**
- Children: never patched by morphdom — third-party DOM is preserved ✓
- Data attributes: still merged from the server — `data-*` updates flow through normally ✓
- `updated()`: still fires when data attributes change ✓

This means you can still pass reactive data via `data-*` attrs and respond to changes in `updated()`.

**Hook lifecycle for map components:**

```javascript
export default {
  async mounted() {
    await this.initMap();
    // Register DOM/document listeners here — they persist through reconnect
  },

  // Data attrs changed (e.g. new locations after navigation).
  // Canvas is preserved by phx-update="ignore" so this.map is always valid here.
  updated() {
    this.clearMarkers();
    this.addMarkers(JSON.parse(this.el.dataset.locations));
    this.map.resize();
  },

  // Socket reconnected after loss (wake from sleep etc.).
  // mounted() does NOT re-run — rebuild the dead WebGL context manually.
  // DOM listeners registered in mounted() persist, so don't re-add them.
  async reconnected() {
    if (this.map) { this.map.remove(); this.map = null; }
    await this.initMap();
  },

  destroyed() {
    if (this.map) { this.map.remove(); this.map = null; }
    // Remove any document-level listeners registered in mounted()
  }
}
```

**Do not** use `beforeUpdate` to detach and re-attach children — temporarily removing the canvas causes WebGL context loss and resets the map viewport.

See also: [[page-meta]]
