# Notice — provenance and licensing

Read this before redistributing or reusing this repository.

## Where this code came from

This repository is a **fork** of [`devopshelper/DSH-statusline`](https://github.com/devopshelper/DSH-statusline). The fork relationship is preserved on GitHub, so the full upstream history and authorship remain visible in `git log` and in the fork banner.

| | |
|---|---|
| Upstream | `devopshelper/DSH-statusline` |
| Fork | `kashiparsavan/DSH-statusline` |
| Upstream authors | `devopshelper` (all commits before the fork) |
| Upstream license | **none declared** |

## The licensing situation, stated plainly

**The upstream repository ships no `LICENSE` file and declares no license in its `package.json`.** Under default copyright, that means all rights are reserved by the upstream author: the absence of a license is not permission, it is the absence of permission.

That has three consequences worth being explicit about, because they are easy to get wrong:

1. **This fork does not grant you a license to the upstream code.** Nothing here can. Only the copyright holder can.
2. **A public fork is not a public license.** GitHub's terms permit forking public repositories on GitHub, which is why this fork exists and why it is legitimately hosted here. That is a platform permission, not a copyright grant, and it does not extend to copying the code elsewhere, bundling it into a product, or relicensing it.
3. **The additions in this fork are not separately licensed either**, because they are modifications of the same work and cannot be cleanly severed from it.

If you need to reuse this code in a way that requires a license — commercial use, redistribution outside GitHub, inclusion in a product — **ask the upstream author**. The correct sequence is upstream first: a license there would cover the whole tree, including this fork.

## What the fork changes

Every change is a commit on `main`; `git log --format='%h %an %s'` shows exactly which author made which. Broadly, the fork adds:

- `docs/` — install, configuration, troubleshooting and architecture references;
- `CONTRIBUTING.md`, `CHANGELOG.md`, this file;
- **the model ribbon** in the session header, including the route-vs-vendor relabel warning;
- documentation corrections and the installer's URL repointing.

## Third-party material

| Material | Origin | Status |
|---|---|---|
| The spinner verb list (187 words) and the 8 turn-end words | Claude Code, extracted from its shipped binary | Not this project's work. Reproduced as a vocabulary table. Reuse should be considered under that product's terms, not this repository's. |
| Provider family marks (`icon.svg` and the SVG geometry in `client.src.js`) | Original to this project | Hand-drawn geometry, not vendor logos. No vendor artwork is bundled. |
| The stylesheet and bundle | Original to this project, built on the upstream base | See the situation above. |

## Marks and names

"DeepSeek", "DeepSeek Harness", and any other product names appearing in the UI or documentation are the property of their respective owners. They are used here descriptively, to say what the plugin is for and what it talks to. No endorsement, affiliation or sponsorship is implied by this repository.

The provider marks are deliberately **not** vendor logos. That is a licensing decision, not an aesthetic one: shipping someone's trademark in a status bar is a question a decoration should not try to answer.

## No warranty

Provided as-is, with no warranty of any kind, express or implied, to the extent permitted by applicable law. This plugin renders information the DeepSeek Harness Host publishes; it does not verify the accuracy of that information, and in particular it does not and cannot verify which model weights answered a request. The relabel warning reports a disagreement between two recorded strings; it is not evidence of anything beyond that disagreement.
