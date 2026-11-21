# Installing DSH Statusline

Everything on this page is Windows-first, because that is what the installers cover. Each section states what it changes and how to undo it.

- [Before you start](#before-you-start)
- [One command](#one-command)
- [From a checkout](#from-a-checkout)
- [From an agent session](#from-an-agent-session)
- [What the installer actually does](#what-the-installer-actually-does)
- [Verifying the install](#verifying-the-install)
- [Updating](#updating)
- [Uninstalling](#uninstalling)
- [Installing by hand](#installing-by-hand)

## Before you start

| Requirement | Notes |
|---|---|
| DeepSeek Harness | Desktop or the `dsh web` CLI. The plugin is a **client** bundle, so it needs the Web UI, not just the CLI. |
| A profile | Start DSH at least once before installing. The installer targets `~/.dsh/profiles/<name>` and needs it to exist. |
| Windows | PowerShell 5.1 or later. The `.cmd` wrappers exist because the default execution policy blocks unsigned `.ps1` files. |
| Unix | No installer ships. Use [install by hand](#installing-by-hand). |

The plugin does not need Node, Git or pnpm on your `PATH`: the installer locates the runtime bundled with DeepSeek Harness. A normal install needs **no build step**, because `client.js` is committed.

## One command

From PowerShell:

```powershell
irm https://raw.githubusercontent.com/kashiparsavan/DSH-statusline/main/install.ps1 | iex
```

From `cmd.exe` or a double-click:

```bat
powershell -NoProfile -ExecutionPolicy Bypass -Command "irm https://raw.githubusercontent.com/kashiparsavan/DSH-statusline/main/install.ps1 | iex"
```

The same command installs **and** updates. Re-running is safe.

## From a checkout

```powershell
git clone https://github.com/kashiparsavan/DSH-statusline
cd DSH-statusline

.\install.cmd              # install or repair
.\update.cmd               # pull the latest, then relink
.\install.ps1 -DryRun      # print the plan, change nothing
```

| Parameter | Default | Meaning |
|---|---|---|
| `-Profile` | `$env:DSH_PROFILE`, then `desktop`, then `web`, then the only profile present | profile under `%USERPROFILE%\.dsh\profiles` |
| `-Source` | the linked checkout, then the script's own directory, then `-InstallDir` | package directory to install |
| `-InstallDir` | `%LOCALAPPDATA%\DSH\dsh-statusline` | where a checkout is cloned or unpacked when no source is found |
| `-RepoUrl` | this repository | where that checkout comes from |
| `-Branch` | `main` | branch to clone or pull |
| `-DshPath` | `$env:DSH_CLI`, then `PATH`, then the desktop runtime | explicit CLI |
| `-Update` | off | pull (or re-download) before linking |
| `-Force` | off | relink without pulling |
| `-DryRun` | off | print the plan, change nothing |

## From an agent session

Inside DeepSeek Harness, the two calls are install and activate:

```
plugin_manager install_bundle  target: <absolute path to this directory>
plugin_manager set_bundle      target: @local/dsh-statusbar  enabled: true
```

`install_bundle` links the package into the active profile; `set_bundle` only repairs the activation entry if the bundle list lost it. Both are idempotent.

## What the installer actually does

Four things, in this order:

1. **Finds the target profile.** `-Profile`, then `$env:DSH_PROFILE`, then `desktop`, then `web`, then the only profile that exists. An ambiguous choice fails loudly rather than guessing.
2. **Finds the CLI and runtime.** `-DshPath`, then `$env:DSH_CLI`, then `PATH`, then the paths bundled with the desktop app. `dsh` does not have to be on `PATH`, and neither does Node.
3. **Obtains a source tree.** An already-linked checkout wins. Otherwise the script's own directory is used, and failing that the repository is cloned into `-InstallDir` — or downloaded as a zip when Git is unavailable.
4. **Links and activates.** The package is added to the profile's dependencies as a `link:` entry (a link, never a copy), then registered in `dsh.profile.bundles` in the same `package.json`. That list entry is what actually turns the plugin on; linking alone does nothing visible.

It also repairs two specific breakages it has seen:

- a profile whose bundle list lost the package — re-registered;
- a **UTF-8 BOM** on the profile manifest, which DSH cannot parse — stripped, because Windows editors and PowerShell's `Set-Content` add them readily.

The install is a link, not a copy. **The checkout is the installed code**, so do not move or delete it afterwards. If you do, re-run the one-liner from the new location.

## Verifying the install

1. Reload the Web UI with **Ctrl+Shift+R**. The client bundle is fetched once per module generation, so a plain refresh may serve the old one.
2. Look under the composer for the strip, and in the session header's right-hand corner for the model ribbon.
3. Both should appear. If neither does, the activation entry is the thing to check — see [Troubleshooting](TROUBLESHOOTING.md#nothing-appears).
4. The plugin's own settings page is **Settings → Plugins → status line**, which is proof the client half loaded even if a slot is not rendering.

## Updating

```powershell
irm https://raw.githubusercontent.com/kashiparsavan/DSH-statusline/main/install.ps1 | iex
```

From a checkout, `.\update.cmd` or `.\install.ps1 -Update`: `git pull --ff-only`, then `dsh plugin remove` + `add` so DSH builds a new module generation, then re-registers the bundle. Reload with Ctrl+Shift+R; restart DSH if the old strip is still on screen.

Settings survive updates. Theme, palette, phrases, lane, size and the ribbon switch all live in the browser's `localStorage`, not in the repository. See [CONFIGURATION.md](CONFIGURATION.md).

A developer checkout is rebuilt only when `client.src.js` or `styles/statusbar.css` is newer than `client.js`; a normal update needs no compile step.

## Uninstalling

```powershell
.\uninstall.cmd     # or: .\uninstall.ps1 -Profile desktop
```

This removes the package from the profile's dependencies *and* from its bundle list, restores the `package.json` backups the installer kept, and leaves your `localStorage` settings alone. Reload afterwards.

The shipped pills come back — this plugin adds cells rather than shadowing the harness's own, so removing it restores the default line rather than leaving a gap.

## Installing by hand

For an unsupported platform or a manual setup. No step needs a build.

1. **Expose the package.** Either link it:
   ```
   cd <profile>
   pnpm add link:<absolute path to this checkout>
   ```
   or copy the directory into `<profile>/node_modules/@local/dsh-statusbar`.

2. **Register it** in `<profile>/package.json`:
   ```json
   {
     "dependencies": { "@local/dsh-statusbar": "link:C:/path/to/DSH-statusline" },
     "dsh": { "profile": { "bundles": [
       "@deepseek-ai/dsh-base",
       "@deepseek-ai/dsh-web-app",
       "@local/dsh-statusbar"
     ] } }
   }
   ```

   The `bundles` entry is required. Without it the package is installed and completely inert.

3. **Restart DeepSeek Harness**, then reload the Web UI.

`scripts/register.mjs` does step 2 for you, which is useful if you are scripting this:

```
node scripts/register.mjs <profileDir> @local/dsh-statusbar add
node scripts/register.mjs <profileDir> @local/dsh-statusbar remove
```

## Where the files go

| Path | What |
|---|---|
| `%USERPROFILE%\.dsh\profiles\<name>\package.json` | the profile manifest: dependency + `dsh.profile.bundles` |
| `%USERPROFILE%\.dsh\profiles\<name>\node_modules\@local\dsh-statusbar` | a link back to the checkout |
| `%LOCALAPPDATA%\DSH\dsh-statusline` | the default clone target for the one-liner |

Nothing is written outside those paths, and nothing is written to the registry or to a startup location.
