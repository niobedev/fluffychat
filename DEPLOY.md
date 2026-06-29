# Toral Chat deployment

This fork hosts an always-up-to-date, customized build of
[FluffyChat](https://github.com/krille-chan/fluffychat) at
**https://chat.housetoral.uk** via GitHub Pages.

## How it works

A single workflow — [`.github/workflows/track-upstream.yaml`](.github/workflows/track-upstream.yaml) —
runs daily (and can be triggered manually). On each run it:

1. Resolves upstream's **latest stable release** (`krille-chan/fluffychat`, via the
   `/releases/latest` API, which excludes pre-releases/drafts).
2. Skips if a release with that tag already exists in this repo (idempotent).
3. Clones upstream **at that tag** into a fresh `upstream/` directory.
4. Overlays our [`deploy/config.json`](deploy/config.json) onto `upstream/web/config.json`.
5. Builds the web app with `--base-href "/"` (root hosting).
6. Deploys `build/web` straight to GitHub Pages via the native Actions deployment
   (`actions/upload-pages-artifact` + `actions/deploy-pages`) — **no `gh-pages` branch**. The
   [`chat.housetoral.uk`](deploy/CNAME) custom domain is included in the artifact.
7. In parallel, builds a native **macOS (Apple Silicon / arm64)** `.app` on a `macos-14`
   runner and zips it.
8. Creates a matching GitHub **release** here with both the web bundle and the macOS `.zip`
   attached.

## macOS app

Each release attaches `toralchat-macos-arm64-<tag>.zip` — a native Apple Silicon build of stock
FluffyChat (not pre-configured; enter `matrix.housetoral.uk` on first login).

It is **ad-hoc signed, not notarized** (no Apple Developer account), so Gatekeeper quarantines it on
download. To run it the first time:

```sh
xattr -dr com.apple.quarantine /path/to/FluffyChat.app   # or right-click the app → Open
```

then move it to `/Applications`. The build is patched at CI time only to blank upstream's hardcoded
`DEVELOPMENT_TEAM` so the unsigned/ad-hoc build succeeds — no app behavior is changed.

### Why an overlay (and not a merge)?

All customizations are **additive** — a runtime `config.json`, a build flag, and a `CNAME`. We never
edit upstream Dart source, so there are **no merge conflicts**. The app source committed on `main` is
a fork snapshot and is *not* used by the pipeline; every build comes from a fresh clone of the pinned
upstream release tag.

## Customizations

[`deploy/config.json`](deploy/config.json) — applied at runtime by FluffyChat's web config loader
(`lib/config/setting_keys.dart`). Kept minimal so we inherit upstream's evolving defaults for
everything else:

| Key | Value | Effect |
| --- | --- | --- |
| `applicationName` | `Toral Chat` | Displayed app name |
| `presetHomeserver` | `matrix.housetoral.uk` | Locks the app to this homeserver (hides the picker) |
| `defaultHomeserver` | `matrix.housetoral.uk` | Pre-fills the homeserver field |

## One-time manual setup

These live outside the codebase and must be done once:

- **DNS**: add a `CNAME` record `chat.housetoral.uk → niobedev.github.io` at the housetoral.uk DNS
  provider.
- **Repo → Settings → Pages**: set **Source = GitHub Actions** (not a branch). After the first
  deploy, set the custom domain to `chat.housetoral.uk` and enable **Enforce HTTPS**.
- **Repo → Settings → Actions → General → Workflow permissions**: **Read and write permissions**.
- **Matrix server** (server-side): `matrix.housetoral.uk` must be reachable and serve a correct
  `/.well-known/matrix/client` so the locked homeserver resolves.

## Manual / forced builds

Run the **Track Upstream & Deploy** workflow from the Actions tab. Leave `force_tag` empty to build
the latest stable release, or set it to a specific tag (e.g. `v2.7.2`) to rebuild/redeploy that
version even if it was already released here.
