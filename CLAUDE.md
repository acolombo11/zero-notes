# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Zero Notes is a personal fork of [siyuan-note/siyuan](https://github.com/siyuan-note/siyuan) (AGPL-3.0), a privacy-first knowledge management desktop app. The fork removes the subscription paywall so that S3, WebDAV, local, and SiYuan-cloud sync providers all work without an account.

## Fork-specific files

These files diverge from upstream and must be handled carefully when merging upstream changes:

| File | What changed |
|------|-------------|
| `app/src/util/needSubscribe.ts` | `needSubscribe()` always returns `false`; `isPaidUser()` always returns `true` |
| `kernel/model/conf.go` | `IsSubscriber()` and `IsPaidUser()` always return `true` |
| `kernel/model/export.go` | Nil-guard on `Conf.GetUser()` (needed because `IsSubscriber()` is now always true) |
| `kernel/model/cloud_service.go` | Same nil-guard for subscription expiry reminder |
| `app/src/config/index.ts` | Account menu item hidden via `fn__none` (not removed) |
| `app/appearance/langs/*.json` | `siyuanNote` key changed to `"Zero Notes"` in all 16 locale files |
| `app/electron/init.html` | Title and heading renamed to Zero Notes |
| `app/electron-builder-darwin-arm64.yml` | `productName` set to `"Zero Notes"` |
| `app/package.json` | `name` set to `"Zero Notes"` |
| `.github/workflows/cd.yml` | Mac ARM64 DMG + GHCR Docker image only; no other platforms |
| `.github/workflows/sync-upstream.yml` | Weekly upstream sync workflow (new) |

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

## Keeping the fork up to date

A GitHub Actions workflow (`.github/workflows/sync-upstream.yml`) runs every Monday and opens a PR from `sync/upstream-YYYYMMDD` with upstream changes merged in. Review that PR for conflicts in the fork-specific files listed above before merging.

To sync manually:

```bash
git remote add upstream https://github.com/siyuan-note/siyuan.git  # first time only
git fetch upstream
git merge upstream/master
```
