# workbuddy-switch

A desktop app (Tauri 2 + Rust + React) for switching between multiple WorkBuddy / CodeBuddy CLI / CodeBuddy CN IDE accounts, with built-in credit-expiry monitoring, auto check-in, and automatic CodeBuddy CLI account rotation.

## Introduction

workbuddy-switch is built for heavy WorkBuddy users who run multiple accounts on a single machine. It solves three everyday pain points:

- The WorkBuddy desktop client only holds one logged-in account at a time; switching manually means replacing auth files and restarting the app — tedious and error-prone.
- Each account's credit resources expire independently, and untracked credits quietly go to waste.
- CodeBuddy CLI and CodeBuddy CN IDE maintain account systems separate from the WorkBuddy desktop app, so each has to be managed on its own.

This app unifies all three into a single interface: multiple accounts share one login state (`workbuddy-desktop.info`), one click switches the active WorkBuddy account (optionally copying the current account's sessions to the target), expiring credits are highlighted and sorted by urgency, check-in and token keep-alive run automatically, and the account with the most urgent expiring credits can be set automatically as the default for future CodeBuddy CLI sessions.

Who it's for: macOS / Windows / Linux users who hold multiple WorkBuddy / CodeBuddy accounts and want to squeeze the most out of every account's credit quota.

Two flavors are available:

- **Desktop app**: a native Tauri 2 application (recommended for daily use) — grab an installer from GitHub Releases
- **npm / webui**: `npm i -g workbuddy-switch`, then use the same account-management features in your browser

## Features

| Module | Description |
| --- | --- |
| Account management | OAuth QR-code login (device flow), import from local machine, add token manually, delete accounts |
| Account switching | Backs up the auth file → quits WorkBuddy → writes the target account → restarts, with live progress feedback |
| Session copy | Copies selected sessions from the current account to the target under new IDs (jsonl bodies + `workbuddy.db` index + edge-sync registration, so the cloud attributes them to the target) |
| Auto check-in | On by default; checks immediately at launch and re-checks every 30 minutes while running; one-click check-in for all accounts; 30-day check-in log |
| Token keep-alive | Lazy refresh (refreshes before an operation when the remainder falls below the threshold) + daily keep-alive (unconditionally refreshes once a day by default; with a threshold > 0, only refreshes accounts whose remainder is under that many days) — keeps refresh tokens from expiring |
| Credit expiry query | Automatically queries each account's credit resources, remaining balance, and expiry; resources expiring within 7 days are highlighted and sorted by urgency, with the most urgent flagged "recommended to use first" |
| Credit statistics | Aggregates official WorkBuddy request usage: daily trends, model breakdown, per-account consumption, and request details; falls back explicitly to local balance snapshots when official data is unavailable |
| Token statistics | Separate overviews for WorkBuddy, CodeBuddy CLI, and CodeBuddy IDE; input/output/cache read/write shown in K/M/B units; trend charts merge daily token composition with call counts, plus composition share, activity heatmap, project/model Top 10, and session rankings |
| CodeBuddy CLI integration | Shares the account library with WorkBuddy but keeps its own default account; sets the account for future sessions via `apiKeyHelper` on macOS/Linux and `env.CODEBUDDY_AUTH_TOKEN` in `settings.json` on Windows; never touches the currently running session on any platform |
| CodeBuddy CN IDE integration | Injects Safe Storage credentials (`state.vscdb` / `planning-genie.new.accessTokencn`) into the CodeBuddy CN desktop client (www.codebuddy.cn) and restarts the IDE; fully independent from CodeBuddy CLI and the international CodeBuddy |
| Auto rotation | Periodically sets the default account for future CodeBuddy CLI sessions to the one with the most urgent expiring credits, so credits never quietly lapse |
| Auto update | Checks GitHub Releases for new versions; full-package updates are signature-verified by tauri-updater |
| Permission detection | macOS authorization guidance (App Management / Full Disk Access drag-to-authorize + automatic detection) |

## How It Works / Tech Stack

### Tech stack

- **Backend**: a Rust workspace (Cargo) on Tauri 2 (with tray-icon)
  - `crates/wb-switch-core`: all business logic, split into 20 modules — account, auth_file, checkin, codebuddy_cli, codebuddy_cn_ide, config, credit_usage, credits, export_import, oauth, official_usage, process, refresh, rotate, session, switch, token_stats, travel, update, vscode_cn_inject
  - `crates/wb-switch-server`: an axum HTTP server with rust-embed static assets, powering the npm/webui flavor with the same capabilities as the desktop app
  - `src-tauri`: a thin command layer exposing core modules as Tauri commands; wires up tauri-plugin-opener / updater / notification / dialog / autostart
- **Frontend**: React 19 + TypeScript + Vite 7, Tailwind CSS 4 + Radix UI, zustand for state, recharts for charts; `src/lib/api.ts` is a dual-channel wrapper — Tauri `invoke` on desktop, HTTP `fetch` in webui — so every page is shared verbatim
- **npm distribution**: the main `workbuddy-switch` package pulls in four per-platform binary packages (`workbuddy-switch-darwin-arm64` / `-darwin-x64` / `-win32-x64` / `-linux-x64`) via `optionalDependencies`; a postinstall script picks the right executable
- **Updates & signing**: the Tauri updater verifies packages against a bundled public key (the `TAURI_SIGNING_PRIVATE_KEY` private key is injected at release time); update manifests are generated by `scripts/gen-update-json.sh` and `scripts/merge-update-manifests.py`

### Key mechanisms

- **Shared login state**: all accounts live in a local account library; switching = back up the current auth file → quit the WorkBuddy process → write the target account's credentials → relaunch the app
- **Auto-rotation policy**: every check runs through 7 decision rules in order (see "Auto-Rotation Policy" below); it only updates the default account for future sessions and never hot-swaps a running one
- **CodeBuddy CLI integration**: on macOS/Linux it writes an `apiKeyHelper` (invoked by the CLI each time a new session starts to fetch a token); on Windows it writes `env.CODEBUDDY_AUTH_TOKEN` in `~/.codebuddy/settings.json` (preserving all other settings, with no `.cmd` shim involved)

## Installation & Usage

### Option 1: Desktop app (recommended)

Download the installer for your platform from [GitHub Releases](https://github.com/sheen945/workbuddy-switch/releases/latest):

| Platform | Package | How to install |
| --- | --- | --- |
| macOS Apple Silicon (arm64) | `workbuddy-switch_<version>_aarch64.dmg` | Open the DMG and drag `workbuddy-switch.app` into Applications |
| macOS Intel (x86_64) | `workbuddy-switch_<version>_x86_64.dmg` | Same as above |
| Windows x64 | `workbuddy-switch_<version>_x64-setup.exe` | Run the installer and follow the prompts |
| Linux x64 | `workbuddy-switch_<version>_amd64.deb` / `_amd64.AppImage` | Install the `.deb` on Debian/Ubuntu; elsewhere, `chmod +x` the AppImage and run it |

If macOS says the developer cannot be verified on first launch: Control-click the app in Finder and choose "Open", or go to "System Settings → Privacy & Security" and choose "Open Anyway". Only if the package genuinely came from the official Releases and macOS still reports it as "damaged", run:

```bash
xattr -rd com.apple.quarantine "/Applications/workbuddy-switch.app"
```

### Option 2: npm / webui

```bash
npm i -g workbuddy-switch
workbuddy-switch              # start the local service and open the browser
workbuddy-switch status       # show the current account in the terminal
```

The webui matches the desktop app: account switching, credit-expiry monitoring, auto check-in, session copy, token statistics, and token keep-alive are all available.

### Daily workflow

1. **Add accounts**: Accounts page → "Scan QR to log in" (OAuth device flow), "Import from this machine", or "Add manually"
2. **Switch accounts**: account card → "Switch", optionally checking "copy current sessions"
3. **Auto check-in**: toggle directly on the Accounts page; the Settings page exposes keep-alive parameters, one-click check-in, and logs
4. **Credit expiry**: the Accounts page queries each account's credit resources automatically; "Refresh credits" updates manually; soon-to-expire resources are highlighted and sorted by nearest expiry
5. **Credit statistics**: sidebar → "Credit Statistics" for overview, 30-day trends, model breakdown, per-account consumption, and request details; filtering never re-hits the official API — only "Refresh statistics" re-collects
6. **Token statistics**: sidebar → "Token Statistics", pick a source (WorkBuddy / CodeBuddy CLI / CodeBuddy IDE)
7. **CodeBuddy CN IDE**: one click on the account card switches the mainland-China desktop client; switching closes and restarts CodeBuddy CN and writes the selected account into the login state under `~/Library/Application Support/CodeBuddy CN`; before first use, open and log in to CodeBuddy CN manually once so Keychain Safe Storage is created
8. **CodeBuddy CLI**: one click on the Accounts page connects/updates auth. "Switch CodeBuddy" only changes the default account used by subsequently loaded sessions; reload the session via ACP or restart the CLI for it to take effect. Running `/resume` inside the same plain CLI process is not guaranteed to re-read the auth configuration
9. **Auto rotation**: Settings → CodeBuddy CLI auto-rotation; once enabled, a background timer checks on an interval
10. **Updates**: the app checks GitHub Releases automatically; when a new version is found, upgrade directly from the bottom-left corner, or open the Release page from Settings to download manually

### Auto-rotation policy

The goal is to keep credits from expiring unused: a background timer periodically queries every account's credit expiry and points future CodeBuddy CLI sessions at the "most urgent" account (earliest expiry with credits remaining). To avoid flapping the default account, every check runs these rules in order:

1. **Eligible accounts**: only accounts that queried successfully, are unexpired, and have credits left can be targets
2. **Urgency check**: if even the most urgent account has more than `min_urgency_hours` left (default 72) → don't switch
3. **Already the target**: if the CLI default account is already the most urgent → don't switch
4. **Cooldown**: don't switch again within `cooldown_minutes` of the last switch (default 120)
5. **Active guard**: if a CLI session has written within the last `active_guard_minutes` (default 30, i.e. a conversation is in progress) → don't switch
6. **Value filter**: if the target's remaining credits are below `min_remaining_credits` → not worth switching (default 0 = disabled; every check logs each account's remainder so you can tune this)
7. **Hysteresis**: if the target expires earlier than the current one by less than `min_gap_hours` (default 24) → don't switch

> **Scope of effect**: rotation only updates the default account used by future restore/load operations — it never hot-swaps a running session. On macOS/Linux the next helper invocation picks up the new account; on Windows the latest token is written to settings. A running session keeps the account it captured at startup.

Configuration keys: `check_interval_minutes` (default 5), `cooldown_minutes`, `min_urgency_hours`, `active_guard_minutes`, `min_remaining_credits`, `min_gap_hours`. Tune them on the Settings page or edit `~/.wb-switch/auto_rotate_config.json` directly.

## Building from Source

Prerequisites: Node.js ≥ 20 and a stable Rust toolchain; macOS, Windows, and Linux are all supported.

```bash
git clone https://github.com/sheen945/workbuddy-switch.git
cd workbuddy-switch
npm install

# Development mode (hot-reloading frontend + Tauri window)
npm run tauri dev

# Build only the frontend bundle
npm run build

# Package a macOS .app (debug; the signing key defaults to ~/.wb-switch/wb-switch-updater.key)
npm run build:app

# Release build
npm run build:app:release
```

For the release process, update-manifest generation (`scripts/gen-update-json.sh`, `scripts/merge-update-manifests.py`), and publishing the npm webui (`cd npm && npm publish`), see [docs/DEVELOPMENT.md](./docs/DEVELOPMENT.md).

> Note: `src-tauri` enables the `custom-protocol` feature by default. A bare `cargo build --release` without that feature produces a dev-mode executable that loads `http://localhost:1420` — not a standalone app. Use the npm scripts above instead.

## Project Structure

```
workbuddy-switch/
├── src-tauri/            # Tauri shell: command wrappers, plugins, app config (tauri.conf.json)
├── crates/
│   ├── wb-switch-core/   # Core business library: 20 modules (accounts/switching/check-in/rotation/stats...)
│   └── wb-switch-server/ # axum HTTP server + rust-embed; backend for the npm/webui flavor
├── src/                  # React frontend
│   ├── pages/            # Accounts, Credit Stats, Token Stats, Settings
│   ├── components/       # account card, OAuth login, switch/import/export/update dialogs, ui kit
│   ├── lib/              # api.ts (Tauri/HTTP dual channel), demo-mode, theme, types, update
│   └── stores/           # zustand stores (accounts, etc.)
├── npm/                  # npm package: bin entry, postinstall, per-platform binary optionalDependencies
├── scripts/              # Build & release scripts (make-dmg, gen-update-json, fix-app, version bump...)
├── docs/                 # DEVELOPMENT.md and UI screenshots
├── public/               # Icons and other static assets
├── Cargo.toml            # Rust workspace definition
└── LICENSE               # MIT
```

## Notes & Caveats

- **macOS permissions**: switching accounts writes to WorkBuddy's auth files, so macOS requires "App Management" (or "Full Disk Access") authorization. When the first switch fails with a permission error, follow the in-app prompt to open System Settings and grant access, then restart the app; "Permission check" on the Settings page verifies at any time. In webui mode, permissions come from the terminal process that launched the service.
- **Session-switch boundary**: CodeBuddy CLI integration and auto-rotation only affect future sessions; a currently running session is never switched — reload the session or restart the CLI.
- **CodeBuddy CN IDE prerequisite**: before the first injection, open and log in to CodeBuddy CN manually once so Keychain Safe Storage is generated.
- **Privacy**: the account library, keys, and token files all stay on your machine (`~/.wb-switch/` etc.); the repo gitignores them and runs a token scan before releases. Never commit `accounts.json` or key files anywhere public.
- **Online demo**: the [GitHub Pages demo](https://sheen945.github.io/workbuddy-switch/) is read-only — all accounts, credits, and request records are fictional, and every business operation is disabled.

## Acknowledgements

Thanks to the [Linux.do](https://linux.do) community.

## License

[MIT](./LICENSE)

## Author

sheen945
