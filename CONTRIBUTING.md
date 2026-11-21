# Contributing

Thanks for looking. This is a small plugin with a lot of deliberate choices; the fastest way to get a change accepted is to work with them rather than around them.

- [The two rules](#the-two-rules)
- [Getting set up](#getting-set-up)
- [Changing the stylesheet](#changing-the-stylesheet)
- [Changing the client](#changing-the-client)
- [Adding a palette](#adding-a-palette)
- [Adding a lane](#adding-a-lane)
- [Working on the model ribbon](#working-on-the-model-ribbon)
- [What will be asked of your PR](#what-will-be-asked-of-your-pr)
- [Things that will be declined](#things-that-will-be-declined)

## The two rules

1. **`client.js` is generated.** Never edit it by hand. Edit `client.src.js` or `styles/statusbar.css`, then `node scripts/build.mjs`, and commit both the source and the regenerated bundle.
2. **Read [ARCHITECTURE.md](docs/ARCHITECTURE.md) before changing anything structural.** The layout rules listed there each fix a bug that took a round trip to find, and several look like details until you remove them.

## Getting set up

```
git clone https://github.com/kashiparsavan/DSH-statusline
cd DSH-statusline
```

There is nothing to install to work on it. Two conveniences:

```powershell
.\install.cmd        # link + activate in your profile
.\update.cmd         # pull and relink
```

Rebuild after editing source:

```powershell
node scripts\build.mjs
```

Node comes from the DSH bundle if it is not on your `PATH`:

```
%LOCALAPPDATA%\Programs\DeepSeek Harness\resources\runtime\primary-runtime\dependencies\node\bin\node.exe
```

There is no test runner. The build is the syntax check (`node --check client.src.js` also works), and the checks that matter are listed per area below.

## Changing the stylesheet

`styles/statusbar.css` is the portable sheet: framework-agnostic, attribute-driven, no preprocessor. It is spliced into the bundle at build time.

Rules for it:

- **Keep it portable.** It must work on any DOM host driven by `data-sb*` attributes alone. No class-name assumptions about DSH.
- **Accent stays a bare RGB triple.** `--sb-glow: 0, 229, 255`, never `rgb(0,229,255)`. It is spliced into `rgba(var(--sb-glow), a)` by every keyframe.
- **Respect the fixed boxes.** A 27ch status slot and a 1.2em glyph box exist so the strip cannot reflow mid-breath.
- **Do not add `container-type` to the shell.** Inline-size containment reports zero intrinsic width and collapses the strip.

Verify by rebuilding, reloading with Ctrl+Shift+R, and checking the strip at two window widths — one wide enough for the full line, one narrow enough to force the wrap.

## Changing the client

`client.src.js` is one factoried module. Keep these conventions, which the existing code follows throughout:

- **`apply`, not load-time side effects.** Stylesheets, listeners, timers, locale dictionaries and slot registrations all go inside `apply` so they can be disposed. A factory that registers at load time leaks on every module generation.
- **Guard every projection read.** Use the `projectionHook` closure pattern — the hook is always called, only the lookup is caught — so hook order stays stable and a mid-teardown throw costs one figure, not the strip.
- **Shape-check before rendering.** `readSelection`, `readSteps`, `readRun` validate and return `null`/`[]` rather than letting a malformed projection reach a prop access.
- **Do not poll.** Projections push. If you find yourself adding `setInterval` to fetch state, the answer is a projection.
- **Locale every visible string.** Add the key to both `en` and `zh`; a missing translation renders the key.

## Adding a palette

A palette is four custom properties. Add one row to `THEMES` in `client.src.js` and one rule to the palette block at the bottom of `styles/statusbar.css`:

```css
[data-sb][data-sb-theme="cyan"] {
  --sb-accent: #00e5ff;
  --sb-accent2: #5ad8ff;
  --sb-glow: 0, 229, 255;
  --sb-wash: #04161b;
}
```

Then rebuild. The swatch in the settings page and the card's palette row pick it up from `THEMES`, and the model ribbon's accent follows automatically. Check the new palette against a dark background *and* at 12px text — a raw accent is often unreadable as text even when it looks right as a glow.

## Adding a lane

The flow lane has five skins, each of which reads the same `tok/s` number so the lane's speed always means the same thing. A sixth must keep that promise.

Add it to `FLOW_ORDER`, add a builder next to the existing ones, add its copy to `FLOW_TEXT` (label plus busy/idle titles), and add its locale keys. The rule that matters: **the lane's motion encodes the token rate and nothing else** — no skin may move faster for decoration.

## Working on the model ribbon

The ribbon is in two pieces: the component (`ModelRibbon`) and the classifier (`modelVendor` / `routeMismatch` against `VENDOR_ROUTES`).

The classifier is the part with a correctness bar, so treat it as such:

- **It compares only what is checkable.** A model id names a vendor; a provider route belongs to a company. Those two can disagree, and that disagreement is honest. Do not extend it to claim anything about the weights served.
- **Silence is the default.** An id naming no known vendor, or a session with no recorded route, is never flagged. A false alarm here trains readers to ignore the warning.
- **Provider ids are named after companies**, not model families. Adding a vendor to `modelVendor` means adding its route aliases to `VENDOR_ROUTES` in the same change, or you will flag every first-party route (`anthropic` serves `claude`, `google` serves `gemini`, `xai` serves `grok`).

A quick check while iterating:

```js
modelVendor('deepseek-v4.1-flash')                                       // 'deepseek'
routeMismatch({ provider: 'r10',      model: 'deepseek-v4.1-flash' })    // true  → ≠
routeMismatch({ provider: 'deepseek', model: 'deepseek-v4.1-flash' })    // false → quiet
routeMismatch({ provider: 'r10',      model: 'nvidia/nemotron-…'    })    // false → quiet
```

Also re-verify the ribbon's resize ladder by resizing the window, since it keys on the header's inline size and not the viewport: full → provider/namespace dropped → effort dropped → hidden.

## What will be asked of your PR

- **What problem does it fix, in one sentence?** A feature without a stated problem tends to be decoration.
- **Did you rebuild `client.js`?** A PR touching `client.src.js` or the stylesheet without the regenerated bundle cannot be installed.
- **Does it survive a narrow window and a missing projection?** Those are the two ways this plugin breaks in practice.
- **Is the new text in both dictionaries?**

Small, focused diffs get merged quickly. A large diff that mixes a fix, a refactor and a new palette will be asked to split.

## Things that will be declined

- **Vendor logos.** The marks are original geometry on purpose.
- **Anything that phones home.** No telemetry, no analytics, no remote config.
- **A model "identity verifier".** The client cannot know which weights answered. A feature that claims to would be a lie in the UI, which is exactly what the ribbon exists to prevent.
- **Dropping the shipped figures by default.** Taking the cells over is supported and documented, but it is not the default because it hides information the harness chose to show.
- **A test-suite dependency.** The plugin ships with no dev dependency tree, and keeping it that way is why installing needs no build step.
