# Architecture

How the plugin is put together, and why each decision was made. Read this before changing anything structural; most of it is load-bearing in a way that is not obvious from the source.

- [Shape of the package](#shape-of-the-package)
- [The module format](#the-module-format)
- [The build step, and why there is so little of one](#the-build-step-and-why-there-is-so-little-of-one)
- [Two seats](#two-seats)
- [Data](#data)
- [Error containment](#error-containment)
- [The floating layer](#the-floating-layer)
- [Layout rules that are load-bearing](#layout-rules-that-are-load-bearing)
- [Why the ribbon is separate from the strip](#why-the-ribbon-is-separate-from-the-strip)
- [What the plugin deliberately does not do](#what-the-plugin-deliberately-does-not-do)

## Shape of the package

```
index.js              host half — apply() does nothing, by design
client.js             generated bundle, loaded directly by the Web UI
client.src.js         editable source for that bundle
styles/statusbar.css  portable, attribute-driven stylesheet (12 palettes)
scripts/build.mjs     splices the stylesheet into client.src.js
scripts/register.mjs  adds/removes the bundle in a profile manifest
cordis.patch.yml      the loader row that inserts this bundle
locale/{en,zh}.json   Plugin Manager display metadata
icon.svg              original artwork
```

There is a host half and a client half because the DSH bundle format has both, not because both are used. The host `apply()` is empty on purpose: everything this plugin does is rendering, and rendering belongs to the client. An empty host half is cheaper and safer than a host half that forwards.

## The module format

`client.js` is not an ES module. It is a script that registers a factory with the host's module loader:

```js
window.__ModuleLoader__.load({
  id: '@local/dsh-statusbar',       // must equal package.json "name"
  factory(require) {
    const React = require('react');
    const h = React.createElement;
    // ...
    return { inject, apply };
  },
});
```

Consequences worth knowing:

- **React comes from the host's module table** via `require('react')`. There is no second React in the page and no bundler resolving one.
- **The `id` must match the package name.** The loader keys factories by it; a mismatch is a module that loads and is never used.
- **`apply`, not side effects at load.** Registration, stylesheets, timers and listeners all happen inside `apply` so they can be disposed. A factory that registered anything at load time would leak on every module generation.
- `react-dom` is optional. It is part of the web boot's static table, but a host that omits it must still get a working card, so the `require` is wrapped and a missing module degrades to in-place rendering rather than failing at load.

## The build step, and why there is so little of one

The runtime has **no build step**: `client.js` is committed and loaded as-is. The only thing built is the splice of `styles/statusbar.css` into the bundle at the `__SB_CSS__` placeholder:

```js
source.replace('__SB_CSS__', JSON.stringify(css))
```

`JSON.stringify` is the whole trick. It produces a valid double-quoted JS string literal. Serializing the CSS any other way — PowerShell's `ConvertTo-Json` among them — can wrap it as `{"value": "..."}`, which makes the bundle concatenate `"[object Object]"` and silently drops every rule. That failure is invisible until the strip renders unstyled.

Why splice at all, rather than an injected `<style>` file fetched at runtime? Because the sheet is vendored and portable: keeping it as a separate editable file means a palette change is a CSS diff, not a search through 200 kB of generated JavaScript. The commit of the *generated* bundle is what makes installing a non-developer operation.

## Two seats

| Seat | Kind | What renders there |
|---|---|---|
| `conversation.composer.dock` | `list`, id `statusline`, `order: -1` | the strip |
| `conversation.session.header.utilities` | `list`, id `statusline-model`, `order: -10` | the model ribbon |
| `settings.plugins.tab` | `list`, id `statusline`, `order: 20` | the settings page |

Both visible seats are `list` slots with **fresh ids**, which means nothing shipped is shadowed: the harness's own entries keep their cells and these simply join them. The original build shadowed the `activity` and `usage` cells at `priority: -1`; that still works and is documented in the README, but it hides the shipped figures, which is not what this build wants.

`settings.plugins.tab` is why the plugin is editable from Settings rather than only from its own hover card. Its `label` is a function, so the tab name resolves in the reader's language when the tab is drawn instead of being frozen at registration.

The header registration is wrapped in a `try`/`catch` and happens **after** the strip. It is an addition; a host without that seat must cost the ribbon alone. Losing the strip to a ribbon would be a worse trade than never showing it.

## Data

Everything comes from **session projections** — no polling, no client-side folding, no arithmetic over accumulating events.

| Projection | Used for |
|---|---|
| `sessionStats` | turns, steps, `decodeTokens / decodeMs` → tok/s, `llmMs + toolMs` → wall time |
| `tokenUsage` | cache-hit %, the `↑in / ↓out` split, billed total |
| `contextPressure` | context occupancy, clamped at 100 |
| `contextBreakdown` | surface composition, in the panel |
| `modelSelection` | `{ next, lastUsed }` — the model chip and the header ribbon |
| `todos` | the step boxes |

Busy state comes from `useSessionStatus` — the session status stream, never token activity — so the breath cannot flicker idle between tool calls.

`useProjection` hands back either a published value or a face store with `getSnapshot`/`subscribe`; `useFaceValue` accepts both and keys its subscription on the **projection name**, not the face object, so a face rebuilt per render cannot churn the subscription.

`modelSelection` carries `{ next, lastUsed }`, which covers a blank session and a settled one. The strip treats them identically — it is naming the model either way. `readSelection` reports which one answered as `settled`, and only the ribbon acts on it, because *"this is what runs next"* is a claim about the future and must not be made from `lastUsed`.

## Error containment

Three nested boundaries, each chosen for what it protects:

1. **`Guard`** (class component, `getDerivedStateFromError`) wraps each seat. A render throw becomes a fallback box reading `status bar unavailable`, not a blank cell — silence is indistinguishable from "not installed".
2. **A guarded projection door.** `useProjection` is called through a closure that catches, so a projection that throws while the session is mid-teardown costs that one figure, never the whole strip. The hook itself is always called, so hook order stays stable.
3. **Shape checks before every render.** `readSelection` and `readSteps` validate before returning; a malformed projection yields `null` and the caller degrades. A host that does not publish a key gets the plain brand chip and no boxes, never a crash.

The ribbon's fallback returns `null` rather than a box: an empty chip in the header would read as a model that failed to load, which is worse than nothing.

## The floating layer

The hover panel is portalled onto `<body>`, not rendered in the dock. This is not decoration. Inside the composer dock the panel shares a stacking context with the compose box, which paints later and therefore over it — the panel ended up behind the textarea, half-visible, flickering as the composer repainted. Clipping compounds it: any ancestor of the dock is free to have `overflow: hidden`.

On `<body>` the panel has a context of its own and nothing in the dock can reach it. Two consequences:

- The layer is `position: fixed` at `z-index: 950` — above the dock (the frame's highest non-modal layer is 101) and below anything modal (dialogs start at 1000, toasts at 1100). A panel floating over a dialog would be worse than one hidden behind it.
- The layer stands in for the strip's **rectangle**, re-measured on `resize` and `scroll` with a sub-pixel guard, so the panel's `left: 12px; bottom: calc(100% + 8px)` still resolves against the strip and there is no second set of numbers to keep in sync.

If `react-dom` is unavailable or the strip could not be measured, the panel falls back to in-place rendering rather than disappearing.

## Layout rules that are load-bearing

Each of these looks like a detail and is not. Changing one re-introduces a bug that was already fixed once.

| Rule | Fails as |
|---|---|
| `--sb-glow` is a bare RGB triple | wrapped in `rgb()`, every `rgba(var(--sb-glow), a)` keyframe breaks at once |
| breath stops at `0% / 38% / 46% / 100%` | a sine reads as a mechanical pulse, not a breath |
| status slot is exactly **27ch**, glyph box **1.2em** | the strip reflows once per breath |
| the dock is forced to `width: 100%` | the owner renders it shrink-to-fit, so the strip grew and shrank with its own text |
| the splash restarts by class flip + reflow on a persistent node | a remount loses the animation and flashes |
| **no `container-type`** on the shell | inline-size containment reports zero intrinsic width and collapses the strip |
| the card's layer is `position: fixed` on `<body>` | it paints behind the compose box |

The `:has()` selectors that stack the shipped pills are written against **this plugin's own element**, never the owner's class names. That is deliberate: if DSH restructures the dock, the selector stops matching and the layout degrades to "pills beside the strip" rather than restyling somebody else's element.

## Why the ribbon is separate from the strip

They read the same projection and could have been one component. They are two because they have different lifetimes, different failure modes and different geometry:

- **Different seats and lifetimes.** The strip is a composer cell; the ribbon is a header utility. Either can be off without the other, and the ribbon must survive the strip's `Guard` failing.
- **Different clipping.** The strip is inside the dock, where it controls its own width. The ribbon is inside `.headerUtilities`, which is `flex:none` and sits outside the title cluster's fade mask — an overlong id there would push the breadcrumb out of its own clip instead of being cut itself.
- **Different resize basis.** The strip keys on the viewport-ish dock width. The ribbon keys on the **header's** inline size, because the sidebar and right panel decide how much room the title row actually has and the window width says nothing about it. The title row is already an anonymous `container-type: inline-size` — the shipped preset label uses it the same way — so the ribbon's steps need no second source of truth.
- **One is not scoped to the shell.** The strip's CSS lives under `.dsh-sb-shell`. The ribbon cannot: it renders in the title row, nowhere near the composer, so it inherits the header's own type and tint instead of painting a black strip into it.

## What the plugin deliberately does not do

- **No host behaviour.** No tools, no commands, no services, no files. The host half is empty.
- **No model self-report.** The ribbon shows the recorded selection. What a model *says* it is has no evidential value and is not read — which is exactly why the relabel warning exists as a separate, checkable signal.
- **No selection control.** The composer's model seat owns the click. A second control in the header would promise a switch the header cannot make.
- **No vendor logos.** The family marks are original geometry. Shipping someone's logo in a status bar is a licensing question a strip should not answer.
- **No telemetry.** Nothing is collected, sent or logged.
- **No polling.** Projections push; the strip renders what it is handed.
