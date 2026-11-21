# Configuration reference

Every setting this plugin reads and writes. Two surfaces edit the same keys: **Settings → Plugins → status line**, and the strip's own hover card. Changing either fires a `dsh-statusbar:settings` event, so the strip behind the settings page and the page behind the strip update live — no reload.

Settings live in the **browser's** `localStorage`. They do not travel with the profile, are not committed, and are not shared between machines or browsers. Reinstalling or updating the plugin does not touch them; clearing site data resets them.

- [Keys](#keys)
- [Themes](#themes)
- [Phrases](#phrases)
- [Size steps](#size-steps)
- [The model ribbon](#the-model-ribbon)
- [Resetting](#resetting)
- [Recolouring DSH's own context meter](#recolouring-dshs-own-context-meter)

## Keys

All keys are prefixed `dsh.statusbar.`. Every value is a string; `readStored` rejects anything outside the allowed set and falls back, so a corrupt entry behaves like an unset one.

| Key | Values | Default | What it does |
|---|---|---|---|
| `theme` | `deepseek`, `cyan`, `green`, `matrix`, `magenta`, `violet`, `amber`, `ice`, `blood`, `cyberpunk`, `cyberwar`, `superblack` | `deepseek` | palette; four custom properties carry the whole theme |
| `fx` | `on`, `off` | `off` | `on` suppresses the full-surface motion (breathing, splash, halos) while keeping every reading identical |
| `pixel` | `on`, `off` | `prefers-reduced-motion` | low-frame motion for RDP or software rendering |
| `flow` | `beads`, `ecg`, `wave`, `terminal`, `traffic` | `beads` | which flow lane is drawn next to the status phrase |
| `strip` | `show`, `hide` | `show` | the docked line under the composer |
| `float` | `show`, `hide` | `hide` | the draggable floating card; mutually exclusive with `strip` |
| `anchor` | `above`, `corner` | `above` | where the hover panel hangs off the strip |
| `view` | `card`, `board` | `card` | panel layout: one column, or the three-column board |
| `collapse` | `on`, `off` | `off` | whether the board stays folded to its summary line |
| `context` | `show`, `hide` | `hide` | the opt-in context gauge on the strip |
| `harness` | `show`, `hide` | `show` | whether DSH's own context ring is kept beside the strip |
| `ribbon` | `show`, `hide` | `show` | the model ribbon in the session header |
| `phrases` | free text | empty | one phrase per line; replaces the built-in vocabulary |
| `width` | step index `0`–`5` | `1` (980) | strip width, stored as an index so the preset list can grow |
| `height` | step index `0`–`6` | `2` (6px) | strip vertical padding, also an index |
| `floatpos` | JSON `{left, top}` | `null` | floating card position in viewport pixels |

Mutual exclusion is enforced on write, not read: setting `strip` to `show` writes `float: hide` and vice versa, so "both on" is unreachable through the UI. A hand-edited pair renders both.

## Themes

A theme is four custom properties, so adding one is a five-line patch. The accent must stay a bare RGB triple — it is spliced into `rgba(var(--sb-glow), a)` by every keyframe, so an `rgb()` wrapper breaks every glow at once.

```css
[data-sb][data-sb-theme="cyan"] {
  --sb-accent: #00e5ff;  --sb-accent2: #5ad8ff;
  --sb-glow: 0, 229, 255;  --sb-wash: #04161b;
}
```

| Palette | Accent | Reads as |
|---|---|---|
| `deepseek` | `#4d6bfe` | the product's own blue |
| `cyan` | `#00e5ff` | amoled cyan |
| `green` | `#00e676` | matrix |
| `matrix` | `#00ff6a` | hacker |
| `magenta` | `#ff2d95` | neon pink |
| `violet` | `#b026ff` | ultraviolet |
| `amber` | `#ffb300` | solar |
| `ice` | `#8ad8ff` | ice |
| `blood` | `#ff1744` | blood |
| `cyberpunk` | `#fcee0a` | cyberpunk |
| `cyberwar` | `#ff2b2b` | cyberwar |
| `superblack` | `#ffffff` | super black |

The palette also tints the model ribbon's surface and its family mark, which is what keeps the header tag reading as the same instrument as the bar below. It deliberately does **not** tint the relabel warning — see below.

## Phrases

`phrases` is a textarea, one phrase per line. Spaces are trimmed and blank lines ignored. A non-empty list replaces both the working verbs and the idle phrases with exactly what you typed; the turn-end word (`Crunched for 4s`) and the glyph stay as they are. Empty the box, or reset, to get the built-in lists back.

The status slot is fixed at 27ch, so a long line truncates rather than resizing the strip. That is deliberate: a strip that reflows once per breath is unreadable.

## Size steps

| Step | `width` | Step | `height` |
|---|---|---|---|
| 0 | 900px | 0 | 2px |
| 1 | 980px (default) | 1 | 4px |
| 2 | 1120px | 2 | 6px (default) |
| 3 | 1280px | 3 | 9px |
| 4 | 1500px | 4 | 12px |
| 5 | `full` | 5 | 16px |
| | | 6 | 20px |

Width stops at 900px rather than shrinking further: the row is `nowrap` and the status slot is a fixed 27ch, so a narrower strip would spill its own contents. Widths above the window are capped by the container, not by a second set of numbers.

## The model ribbon

`ribbon` controls the header tag. On by default.

When the model id names a vendor the provider route does not belong to, the ribbon marks itself `data-sb-rb="relabel"` and shows `≠` between route and id. This is not configurable, because making it dismissible would defeat the point; turn the whole ribbon off if you do not want it.

```js
// The classifier, for reference — see README for the full reasoning.
modelVendor('cmc-b/deepseek/deepseek-v4.1-flash')  // 'deepseek'
routeMismatch({ provider: 'r10', model: 'cmc-b/deepseek/deepseek-v4.1-flash' })  // true
routeMismatch({ provider: 'deepseek', model: 'deepseek-chat' })                  // false
```

`VENDOR_ROUTES` in `client.src.js` is the table of provider-name aliases (`anthropic` for `claude`, `google` for `gemini`, `xai` for `grok`, `meta` for `llama`, `alibaba` for `qwen`, …). A vendor missing from the table is never flagged — silence is the correct answer when there is nothing to compare.

Recognised vendors today: `deepseek`, `claude`, `openai`, `gemini`/`gemma`, `llama`, `mistral`, `qwen`, `moonshot`, `grok`, `nvidia`, `cohere`, `baidu`, `zhipu`/`glm`.

If your provider is an aggregator, nearly every model will carry the warning. That is accurate: those models are served by the aggregator's endpoint, not the vendor's.

## Resetting

The settings page's **reset to defaults** writes every key back to its default and clears the floating card position, in one commit. To do it by hand:

```js
Object.keys(localStorage)
  .filter(k => k.startsWith('dsh.statusbar.'))
  .forEach(k => localStorage.removeItem(k))
```

then reload. Removing one key resets exactly that setting — there is no derived state to get out of step.

## Recolouring DSH's own context meter

Not this plugin's setting, but reachable from it. The ring and percentage the composer draws beside the strip are composer chrome, not a slot, and they use DSH's label tokens — so they are recolourable without touching the harness:

```css
:root { --dsh-sb-ctx: #00e5ff }         /* ring + percentage */
:root { --dsh-sb-ctx-hover: #7af0ff }   /* optional; defaults to --dsh-sb-ctx */
```

The plugin ships the rules, inert until the variable is set, and scoped structurally (`[data-composer-dock]` plus the meter's 14px `viewBox` ring) rather than by a hashed class name that changes between DSH builds.
