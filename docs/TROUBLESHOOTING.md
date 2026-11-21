# Troubleshooting

Symptom first. Each entry says what to check, then what to run.

- [Nothing appears](#nothing-appears)
- [The plugin is installed but the strip is missing](#the-plugin-is-installed-but-the-strip-is-missing)
- [The plugin's settings page is missing too](#the-plugins-settings-page-is-missing-too)
- [The old version is still on screen](#the-old-version-is-still-on-screen)
- [The header ribbon is missing](#the-header-ribbon-is-missing)
- [The ribbon shows a relabel warning](#the-ribbon-shows-a-relabel-warning-)
- [The strip jumps or reflows](#the-strip-jumps-or-reflows)
- [The strip is absent on the New Session screen](#the-strip-is-absent-on-the-new-session-screen)
- [Motion is choppy](#motion-is-choppy)
- [The installer refuses the profile](#the-installer-refuses-the-profile)
- [Execution policy blocks the scripts](#execution-policy-blocks-the-scripts)
- [The profile manifest will not parse](#the-profile-manifest-will-not-parse)
- [Diagnostics to include in a report](#diagnostics-to-include-in-a-report)

## Nothing appears

Work down this list; it is ordered by how often each is the cause.

1. **Did you reload with Ctrl+Shift+R?** A plain refresh can serve the cached module generation. This is the single most common cause.
2. **Did you restart DeepSeek Harness?** The client bundle is fetched once per module generation. If the hard reload did not help, quit and relaunch DSH.
3. **Is the bundle in the activation list?**

   ```powershell
   Get-Content "$env:USERPROFILE\.dsh\profiles\desktop\package.json"
   ```

   Look for `"@local/dsh-statusbar"` in `dsh.profile.bundles`. A dependency without a bundles entry is installed and completely inert. That is the design, and it is the trap most manual installs fall into.

4. **Does the link resolve?**

   ```powershell
   Test-Path "$env:USERPROFILE\.dsh\profiles\desktop\node_modules\@local\dsh-statusbar\client.js"
   ```

   `False` means the checkout moved or was deleted. The install is a link, not a copy — re-run the installer from where the checkout lives now.

5. **Is the strip switched off?** Settings → Plugins → status line → `status line` / `status card`. Both off means nothing is drawn; that is a valid configuration, not a failure.

## The plugin is installed but the strip is missing

Check the settings page first. If **Settings → Plugins → status line** exists, the client half loaded and the problem is a slot, not the install:

- the composer dock may have been restructured by a DSH upgrade, so `conversation.composer.dock` no longer exists under that name;
- another plugin may have taken the same `id` at the same priority, which throws and abdicates the entry.

Either way the plugin's own error boundary renders a fallback box reading `status bar unavailable` instead of a blank cell, so a silent absence points at the install rather than the render.

## The plugin's settings page is missing too

Then the module never loaded. In order of likelihood: not in `dsh.profile.bundles`; the link is broken; `client.js` is missing from the package directory; or the `dsh.client` manifest section was damaged. Re-run the installer — it repairs the first three and leaves the fourth to `git checkout package.json`.

## The old version is still on screen

The module generation is cached. In sequence:

1. **Ctrl+Shift+R** in the Web UI.
2. Quit and relaunch DeepSeek Harness.
3. Confirm the profile copy matches the checkout — they are linked, so this should be automatic:

   ```powershell
   (Get-FileHash .\client.js).Hash
   (Get-FileHash "$env:USERPROFILE\.dsh\profiles\desktop\node_modules\@local\dsh-statusbar\client.js").Hash
   ```

   Identical hashes plus a stale screen means the renderer, not the files.

4. On a developer checkout, rebuild before reinstalling:

   ```powershell
   node scripts\build.mjs
   ```

   `client.js` is committed, so a normal install never needs this — but an edited `client.src.js` or `styles/statusbar.css` is not live until spliced.

## The header ribbon is missing

The ribbon lives in `conversation.session.header.utilities`. It is absent when:

- **The switch is off** — Settings → Plugins → model ribbon.
- **The session is blank.** A New Session screen has no session header chrome at all, so there is nowhere for it to go. The strip under the composer still names the model there.
- **The header is too narrow.** Below roughly 420px of *header* width the tag hides itself deliberately. Widen the window, close the right panel, or collapse the sidebar. Note this is header width, not window width — the sidebar and panel both eat into it.
- **No model projection is published.** The ribbon renders nothing rather than an empty chip, so a host that does not publish `modelSelection` gets no tag. Every current DSH release does.

## The ribbon shows a relabel warning (≠)

That is the relabel warning doing its job: the model id names a vendor the provider route does not belong to. **It is not an error.**

Example: provider `r10`, model `cmc-b/deepseek/deepseek-v4.1-flash`. The id says DeepSeek; the route is R10, a reseller. The harness records which endpoint it called and what that endpoint named the model — it never learns which weights answered.

What it means: *the name is the route's claim, not proof of origin.* Check the provider in **Settings → Models** if you care which endpoint you are paying.

What it does **not** mean: that the weights are substituted. The client cannot know that. A relabel is a prompt to check, not proof.

To confirm the classifier agrees with your config, look at the provider id and the model id:

```js
modelVendor('deepseek-v4.1-flash')                                    // 'deepseek'
routeMismatch({ provider: 'r10',          model: 'deepseek-v4.1-flash' })  // true  → ≠ shown
routeMismatch({ provider: 'deepseek',     model: 'deepseek-v4.1-flash' })  // false → quiet
routeMismatch({ provider: 'openrouter',   model: 'nvidia/nemotron-…'   })  // false → no known vendor, quiet
```

If you see a relabel you believe is wrong, the alias table `VENDOR_ROUTES` in `client.src.js` is the one place to fix it — a provider named after a company rather than a model family needs its alias added (`google` for `gemini`, `xai` for `grok`, …).

## The strip jumps or reflows

The strip is built not to. If it does, something is overriding the load-bearing layout rules:

- a custom stylesheet setting `width` or `flex` on `.dsh-sb-shell`;
- `container-type` on the shell — inline-size containment reports zero intrinsic width and collapses the strip;
- a DSH upgrade that restructured the composer dock, which stops the `:has()` rules from matching. The failure mode is the shipped pills sitting *beside* the strip rather than below it, not a broken layout elsewhere.

## The strip is absent on the New Session screen

Expected. There is no session, so there are no session figures — and no session header for the ribbon. The strip appears once a conversation starts.

## Motion is choppy

Turn on **low frame rate** (`pixel`) in the card or in settings. It is the same strip with stepped animations, chosen automatically when `prefers-reduced-motion` is set. On a remote desktop this is usually the difference between usable and not.

## The installer refuses the profile

It fails loudly when it cannot pick a profile unambiguously — more than one exists and none is named `desktop` or `web`. Name it:

```powershell
.\install.ps1 -Profile <name>
```

An unknown name lists the profiles it found.

## Execution policy blocks the scripts

Use the `.cmd` wrapper, which sets `-ExecutionPolicy Bypass` for that one invocation:

```powershell
.\install.cmd
.\update.cmd
.\uninstall.cmd
```

## The profile manifest will not parse

Almost always a **UTF-8 BOM**, which Windows editors and PowerShell's `Set-Content` add readily and DSH cannot parse. The installer strips it. To check by hand:

```powershell
$bytes = [System.IO.File]::ReadAllBytes("$env:USERPROFILE\.dsh\profiles\desktop\package.json")
"starts with BOM: $($bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB -and $bytes[2] -eq 0xBF)"
```

To strip it:

```powershell
$p = "$env:USERPROFILE\.dsh\profiles\desktop\package.json"
$text = [System.IO.File]::ReadAllText($p).TrimStart([char]0xFEFF)
[System.IO.File]::WriteAllText($p, $text, (New-Object System.Text.UTF8Encoding $false))
```

## Diagnostics to include in a report

```powershell
$profileDir = "$env:USERPROFILE\.dsh\profiles\desktop"
"dsh on PATH : $([bool](Get-Command dsh -ErrorAction SilentlyContinue))"
"profile     : $profileDir ($(Test-Path $profileDir))"
"manifest    : $((Get-Content "$profileDir\package.json" -Raw | ConvertFrom-Json).dsh.profile.bundles -join ', ')"
"linked      : $(Test-Path "$profileDir\node_modules\@local\dsh-statusbar\client.js")"
"bundle hash : $((Get-FileHash "$profileDir\node_modules\@local\dsh-statusbar\client.js" -ErrorAction SilentlyContinue).Hash)"
"plugin keys : $((Get-ChildItem 'HKCU:\Software' -ErrorAction SilentlyContinue | Measure-Object).Count) (unused; listed for completeness)"
```

Also state: DSH version, Windows version, whether the settings page appears, and whether a hard reload or a restart was tried. "It does not work" is unactionable; "the settings page appears but the strip does not, after a restart" localises it to a slot in one step.
