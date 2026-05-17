# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Zero Notes is a personal fork of [siyuan-note/siyuan](https://github.com/siyuan-note/siyuan) (AGPL-3.0), a privacy-first knowledge management desktop app, rebranded with custom icons and name.

## Fork-specific files

These files diverge from upstream and must be handled carefully when merging upstream changes:

| File | What changed |
|------|-------------|
| `app/appearance/langs/*.json` | `siyuanNote` key changed to `"Zero Notes"` in all 16 locale files |
| `app/electron/init.html` | Title and heading renamed to Zero Notes |
| `app/electron-builder-darwin-arm64.yml` | `productName: "Zero Notes"`, code signing disabled (`identity: null`) |
| `app/package.json` | `name` set to `"zero-notes"` |
| `app/src/assets/`, `app/stage/`, `app/appearance/boot/`, `app/electron/` | Icons replaced with Zero Notes branding |
| `.github/workflows/cd.yml` | Mac ARM64 DMG built on every master push; rolling `latest` pre-release |
| `.github/workflows/dockerimage.yml` | Pushes to GHCR (`ghcr.io/acolombo11/zero-notes`) instead of Docker Hub |
| `.github/workflows/sync-upstream.yml` | Weekly upstream sync workflow (new) |
| `app/appearance/.gitignore` | Added `!themes/Zero` exception to allow the Zero theme submodule |
| `app/appearance/themes/Zero` | Git submodule pointing at `acolombo11/zero-notes-theme` |
| `kernel/conf/appearance.go` | Default theme changed from `daylight`/`midnight` to `Zero` for both modes |

## Architecture

The app has two independent processes that communicate via HTTP/WebSocket:

**Kernel** (`kernel/`) — Go binary, the backend:
- `main.go` boots the kernel: initialises config → starts HTTP server → initialises SQLite databases → starts background jobs
- `server/` — Gin HTTP router; routes are registered in `kernel/api/` (one file per domain: `sync.go`, `block.go`, `export.go`, etc.)
- `model/` — all business logic; `conf.go` owns global app config (`Conf`), `sync.go` drives data sync, `repository.go` handles the encrypted data repo (dejavu)
- `sql/` — SQLite FTS5 full-text search index over all blocks
- Data is stored in the workspace as `.sy` JSON files (one per document)

**Frontend** (`app/`) — Electron + TypeScript + webpack:
- `electron/main.js` — Electron main process; spawns the kernel binary, opens `BrowserWindow` with `titleBarStyle: "hidden"`
- `app/src/` — renderer TypeScript source, bundled by webpack into `app/stage/`
  - `index.ts` / `boot/` — application bootstrap, connects to kernel WebSocket
  - `protyle/` — the core block editor (the largest subsystem)
  - `layout/` — tab/pane/dock window management
  - `config/` — all Settings dialog panels; `repos.ts` is the sync settings panel
  - `sync/` — sync guide/cloud directory picker UI
  - `mobile/` — separate entry point for the mobile web UI
- `app/stage/` — static assets served by the kernel's HTTP server (the web UI when accessed from a browser)
- `app/appearance/langs/` — all UI strings; the `siyuanNote` key is used for `document.title`

The kernel serves the frontend at `http://localhost:6806`. When running in Electron the renderer loads this URL directly; in Docker/browser mode users navigate to it manually.

## Development commands

### Frontend

```bash
cd app
pnpm install
pnpm run dev              # watch-mode build (all entry points)
pnpm run dev:desktop      # watch-mode for desktop entry point only
pnpm run build            # production build (all entry points)
pnpm run lint             # ESLint --fix
```

### Kernel

Requires Go with CGO enabled (`CGO_ENABLED=1`).

```bash
cd kernel

# Build for local dev (output next to app/)
go build -tags fts5 -o "../app/kernel/SiYuan-Kernel"    # macOS/Linux
go build -tags fts5 -o "../app/kernel/SiYuan-Kernel.exe" # Windows

# Run kernel in dev mode (from the kernel output dir)
cd ../app/kernel
./SiYuan-Kernel --wd=.. --mode=dev
```

The `-tags fts5` flag is required — it enables SQLite FTS5 full-text search.

### Run the Electron app (dev)

With the kernel already running:

```bash
cd app
pnpm run start
```

### Production packaging (Mac ARM64)

```bash
cd app
pnpm run build
# Build kernel for darwin/arm64 first (see CI workflow for the exact flags)
pnpm run dist-darwin-arm64
```

### Docker

```bash
docker build -t zero-notes .
docker run -d -p 6806:6806 \
  -v /your/workspace:/siyuan/workspace \
  zero-notes \
  --workspace=/siyuan/workspace \
  --accessAuthCode=yourcode
```

Always set `--accessAuthCode` when exposing the container to a network.

## Zero theme

The Zero theme lives in a separate repo ([acolombo11/zero-notes-theme](https://github.com/acolombo11/zero-notes-theme)) and is included here as a git submodule at `app/appearance/themes/Zero`. It is set as the default theme for both light and dark modes in `kernel/conf/appearance.go`.

After a fresh clone, initialise it:

```bash
git submodule update --init
```

To pull in theme updates after pushing new commits to `zero-notes-theme`:

```bash
git submodule update --remote app/appearance/themes/Zero
git add app/appearance/themes/Zero
git commit -m "chore(theme): update Zero theme submodule"
```

> **Note:** the submodule URL in `.gitmodules` is a relative path (`../zero-notes-theme`), which git resolves to `https://github.com/acolombo11/zero-notes-theme`. Theme commits must be pushed to that repo before running `--remote` here.

## Keeping the fork up to date

A GitHub Actions workflow (`.github/workflows/sync-upstream.yml`) runs every Monday and opens a PR from `sync/upstream-YYYYMMDD` with upstream changes merged in. Review that PR for conflicts in the fork-specific files listed above before merging. The branding files (langs, init.html, electron-builder config, icons) are the most likely to conflict.

To sync manually:

```bash
git remote add upstream https://github.com/siyuan-note/siyuan.git  # first time only
git fetch upstream
git merge upstream/master
```
