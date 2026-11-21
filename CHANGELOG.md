# Changelog

Notable changes, newest first. This project uses [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) structure and does not yet follow semantic versioning — the package version tracks the bundle, and a user-visible change bumps the minor.

## [Unreleased]

### Added — the model ribbon, and the relabel warning

A read-only tag in the session header naming the model the session is actually running, registered into `conversation.session.header.utilities` as a fresh id so nothing shipped is shadowed.

It exists for one reason beyond convenience: **a model id is a label the route chose.** The harness records which provider endpoint it called and what that endpoint names the model; it never learns which weights answered. On a reseller route — an OpenAI-compatible gateway, a free-tier aggregator, a self-hosted proxy — an id like `deepseek-v4.1-flash` is a claim by the reseller, not a fact the client can verify.

So when the id names a vendor the route does not belong to, the ribbon shows both halves of the seam instead of repeating one as though it were the other:

- `data-sb-rb="relabel"` on the tag, a **dashed** seam and a palette-accent tint, and `≠` printed between route and id — `r10/≠deepseek-v4.1-flash`;
- the tooltip is replaced with a sentence naming the route, the vendor and the id.

The seam is carried by shape and glyph rather than hue: the tint follows the same palette accent as the rest of the tag, so the warning never fights the strip beneath it, and a reader who cannot separate two hues still sees the dashed border and the `≠`. (An earlier cut used a fixed amber literal here; it looked like a different component inside a themed header.)

The comparison is deliberately conservative: vendor detection looks at the **model id only** (never the provider), provider ids resolve through an alias table (`anthropic`→`claude`, `google`→`gemini`, `xai`→`grok`, `meta`→`llama`, `alibaba`→`qwen`), and an id naming no known vendor is never flagged. Vendors recognised: `deepseek`, `claude`, `openai`, `gemini`/`gemma`, `llama`, `mistral`, `qwen`, `moonshot`, `grok`, `nvidia`, `cohere`, `baidu`, `zhipu`/`glm`. Silence is the default, because a false alarm teaches readers to ignore the warning. The tint follows the palette rather than a fixed literal — a warning that fights the theme reads as a different component — and the meaning is carried by the dashed seam and the `≠` so it survives any palette and any colour vision.

The tag degrades on the **header's own inline size**, not the viewport: >720px full, ≤720px provider/namespace dropped, ≤560px effort dropped, ≤420px hidden. Its namespace drops rather than ellipsizing, because a namespaced id is mostly path and cutting from the end would keep the path while discarding the one word a reader recognises.

Switchable in **Settings → Plugins → model ribbon**; `localStorage["dsh.statusbar.ribbon"]`.

### Added — documentation

`docs/INSTALL.md`, `docs/CONFIGURATION.md`, `docs/TROUBLESHOOTING.md`, `docs/ARCHITECTURE.md`, `CONTRIBUTING.md`, `CHANGELOG.md` and `NOTICE.md`, plus a table of contents and a route-vs-name section in the README. The installer's one-liner URLs moved to this fork so an install no longer silently fetches the upstream build.

### Fixed

- **The ribbon's tooltip overclaimed.** It said "model this session *runs*" while preferring `next`, which is a statement about the future. `readSelection` now reports `settled`, and a value that came from `lastUsed` is described as *last ran*.
- **The ribbon's accessible name ran the id and effort together** — adjacent spans carry no whitespace, so a screen reader read `deepseek-v4.1-flashhigh`. A real `·` element now separates them.
- **The 561–720px step spent its width on a truncated namespace** and still cut the id. The namespace now drops with the provider.
- **An id tail could be sliced mid-glyph** by the parent's clip; it now ellipsizes.
- **The effort caption was a 9.5px `opacity:.55` whisper** for the one dimension that changes behaviour. Now 10.5px in `--dsw-alias-label-tertiary`.
- **The ribbon's surface bypassed the host token.** It now uses `--dsw-alias-fill-tsp-secondary` with the accent tint applied over it via `color-mix()` where supported.
- **`anthropic` was flagged as a relabel.** Provider ids are named after companies, not model families; the alias table fixes it and the classifier is now checked against 30 route/model pairs, including every vendor-named model in a real aggregator config.

## [2026-11-14]

### Fixed

- The `.cmd` wrappers paused only when double-clicked, so a scripted invocation could stop on a keypress.
- The **harness bar** switch now controls the current context ring and defaults to on, matching the composer's current layout.

### Added

- One-command installer and updater (`install.ps1`, `install.cmd`, `update.cmd`), including a `-DryRun` plan, BOM repair and bundle-list repair.
- A **harness bar** switch, independent of the strip presentation.

## [2026-11-12]

### Added

- One presentation at a time, a real settings preview, and an opt-in context gauge.
- A **tok/s tachometer**: a 30×16 gauge beside the live decode rate, filling toward 200 tok/s with an easing arc and needle.
- A docked status line and an independent floating card, each with its own switch.
- Editable phrases, and a floating card that homes above the line.
- A **board** panel that folds to a one-line summary.
- A settings page under **Settings → Plugins**.
- Terminal and highway flow lanes; the highway's car count and road speed both scale with the token rate.
- The running model's name, the agent's checklist mirrored as step boxes, and `w`/`h` size steppers.

### Fixed

- The card is portalled onto `<body>` so the compose box cannot paint over it.
- The strip's width is pinned to the layout instead of to its own text.
- The hide rule no longer hides this plugin's own strip.
- Tool buttons read as the state they are in, and the FX latch shows in the accent.

## [2026-11-04]

### Added

- The first living status line for the DSH composer: breathing accent glow, rotating status phrase, token-flow beads, real session figures, click wave.
- Claude Code's 187 spinner verbs and its 8 turn-end words, verbatim.
- Low-frame **pixel** mode for RDP and software rendering.
- A demo toggle that rehearses the working state without spending a prompt.
- Optional flat ECG lane; corner-anchored and draggable cards; an installer.
- A colour hook for DSH's own context meter.

### Changed

- `ctx` and `tok` dropped from the strip — the harness already renders context occupancy, and both remain in the hover card.
- The shipped pills are stacked under the strip rather than replaced.

### Removed

- The diving-whale lane, superseded by the highway.
