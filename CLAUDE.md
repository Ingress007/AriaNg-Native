# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

AriaNg Native is a desktop wrapper around [AriaNg](https://github.com/mayswind/AriaNg) (an AngularJS 1.6 web frontend for the aria2 download daemon), packaged with Electron and extended with native desktop capabilities (file associations, magnet protocol handling, tray, custom titlebar, drag-and-drop tasks, a floating "infobox" stats window, startup command execution).

The codebase is split into two cooperating Electron processes that never share memory and talk only over IPC:

- **`main/`** — the Electron **main process** (Node.js). Entry point `main/main.js` (`"main"` in package.json).
- **`app/`** — the **renderer process**: the AngularJS single-page app. Entry `app/index.html` (`"entry"` in package.json).

`app/` is largely vendored/synced from upstream AriaNg (see the `AriaNg` header banner in `app/scripts/core/__core.js`). When touching files under `app/scripts/services/ariaNg*` or `app/scripts/controllers/`, assume they originate upstream — the native-specific glue lives in `ariaNgNativeElectronService.js` and the `main/` tree. Recent commits show the merge workflow pulls upstream AriaNg commits in periodically.

## Tech stack

| Layer | Stack |
|-------|-------|
| Shell | **Electron 22** (Chromium + Node ≥14), packaged with **electron-builder 24** |
| Renderer (`app/`) | **AngularJS 1.6** SPA, Bootstrap 3.4 + AdminLTE 2.4, jQuery 3.4, ECharts 3.8, angular-translate, no bundler (plain `<script>` tags) |
| Main (`main/`) | Node CommonJS. **axios** (HTTP RPC), **ws** (WebSocket RPC), **electron-store** (settings), **bencode** (torrent parsing), **yargs** (CLI args), **electron-localshortcut** |
| Build tooling | `fs-extra`, `git-rev-sync`, `jsonfile`, `rimraf`, custom `node` copy scripts |

There is no TypeScript, no transpilation, and no module bundler anywhere — both processes run plain JS as authored.

## Commands

There is **no test suite and no linter** configured — do not invent `npm test`/`npm run lint`. The npm scripts in `package.json` are the source of truth:

```bash
npm install            # also runs `install-app-deps` (electron-builder native rebuild) via postinstall
npm start              # launch the app locally via `electron .`
npm run clean          # rimraf dist

# Full packaged builds (run clean → generate-build-json → copy deps → electron-builder):
npm run publish:win    # builds Windows x86 + x64 (electron-builder-windows-x86.json / -x64.json)
npm run publish:osx    # builds macOS (electron-builder-mac.json)
```

Build helper scripts (invoked by the publish targets, runnable standalone with `node`):
- `generate-build-json.js` → writes `dist/build.json` (git commit hash, read by `main/core.js` as `buildCommit`).
- `copy-main-modules.js` → copies the Node modules listed in `package.json` `"mainDependencies"` into `dist` for the main process.
- `copy-app-modules.js` → parses `app/index.html` for `node_modules/*.{css,js}` and font `url(...)` references and copies just those assets into `dist`. **Consequence:** a renderer dependency only ships if it is referenced by a `<link>`/`<script>` tag in `index.html` with the literal `../node_modules/...` path pattern.

Runtime flags (see README / `main/cmd.js`): `--development`/`-d` (dev mode, enables F12 / Cmd+Alt+I devtools), `--classic`/`-c` (native titlebar on Windows), `--minimal`/`-m` (start hidden).

## Architecture: the IPC bridge

The renderer has `nodeIntegration: true` and `contextIsolation: false` (see the `BrowserWindow` config in `main/main.js`), so renderer code can `nodeRequire('electron')` directly. All native functionality is funneled through one Angular service:

- **`app/scripts/services/ariaNgNativeElectronService.js`** — the *only* renderer-side gateway to the main process. Every native call (HTTP, WebSocket, config, dialogs, tray, notifications, bittorrent parsing, infobox) goes through its `invokeMainProcessMethod` (`ipcRenderer.send`), `invokeMainProcessMethodSync` (`sendSync`), or `invokeMainProcessMethodAsync` (`invoke`) helpers.
- **`main/ipc/main-process.js`** — registers every `ipcMain` handler. Channels follow the `render-*` naming convention (renderer → main).
- **`main/ipc/render-proecss.js`** — main → renderer direction: `notifyRenderProcess*` functions that `webContents.send(...)` on `on-main-*` channels, plus `onRenderProcess*` listeners for `on-render-*` channels. **Note the filename is misspelled (`render-proecss`); this is the real path — do not "correct" it.**

IPC channel naming conventions:
- `render-*` — renderer invokes main (send / sendSync / invoke).
- `on-main-*` — main pushes events to renderer (logs, navigation, new-task-from-file/text, websocket open/close/message).
- `on-render-*` — renderer notifies main (view loaded, drop file/text, electron-service-inited).

### Native networking (key divergence from browser AriaNg)

Unlike the browser build, RPC traffic to aria2 does **not** use browser fetch/WebSocket — it is proxied through the main process to avoid CORS and use native networking:
- HTTP RPC: `app/scripts/services/aria2HttpRpcService.js` → `ariaNgNativeElectronService.requestHttp` → IPC `render-request-http` → `main/lib/http.js` (axios).
- WebSocket RPC: `ariaNgNativeElectronService.createWebSocketClient` builds a proxy object whose `send`/`reconnect`/`readyState` are all IPC calls → `main/lib/websocket.js` (the `ws` package). Incoming frames arrive as `on-main-websocket-message` events, filtered by `rpcUrl`.

## Main process layout (`main/`)

- `main.js` — app lifecycle, single-instance lock, `BrowserWindow` creation, window position/size persistence, magnet protocol registration, macOS `open-file`/`open-url` handlers, second-instance handling (opening files/URLs into a running app).
- `core.js` — shared mutable singleton: `mainWindow`, `isDevMode`, version/build info, `startupCommandOutput`, `isConfirmExit`.
- `cmd.js` — yargs command-line parsing (`file/url` positional + flags).
- `config/config.js` — persisted user settings via **electron-store**. The schema is declared but note the inline comment: schema validation is *not* actually wired up, so values are clamped manually (e.g. infobox opacity). Call `config.save('<key>')` to persist a single field.
- `config/constants.js` — `supportedFileExtensions` (.torrent/.meta4/.metalink), AriaNg route locations.
- `components/` — `menu.js`, `tray.js`, `titlebar.js` (Windows custom titlebar overlay color), `notification.js`, `infobox.js`.
- `lib/` — `http.js`, `websocket.js`, `bittorrent.js` (parse torrent metadata via `bencode`), `localfs.js` (file existence/read for local file ops), `file.js`, `page.js` (URL/route parsing), `process.js` (startup command exec), `utils.js`.

The **infobox** (`main/components/infobox.js` + `app/views/infobox.html`) is a native-only feature: a frameless, transparent, always-on-top floating `BrowserWindow` showing live download/upload speed. It has its own IPC channels (`render-set-infobox-*`, `render-send-infobox-stat`, `infobox-drag-*`) and persists its position/opacity. It is the focus of the most recent native development.

## Renderer layout (`app/scripts/`)

Standard AngularJS module (`ariaNg`, bootstrapped in `core/app.js`) with subfolders: `core/`, `config/`, `controllers/`, `directives/`, `filters/`, `services/`. Scripts are loaded via explicit `<script>` tags in `app/index.html` (no bundler) — **adding a new renderer file requires adding its `<script>` tag to `index.html`**, and if it pulls a new node_modules asset that asset must also be referenced there to be packaged (see `copy-app-modules.js` above).

- `services/ariaNg*` — upstream AriaNg services (settings, monitor, localization, logging, notifications, version, etc.). `ariaNgNativeElectronService.js` is the native bridge.
- `services/aria2*RpcService.js` — RPC client layer (HTTP / WebSocket / shared `aria2RpcService`).

## Localization

Translations live in `app/langs/<langcode>.txt`. To add a language: register it in `app/scripts/config/languages.js`, copy `i18n/en.sample.txt` into `app/langs/` named with the language code, and translate. `i18n/en.sample.txt` is the canonical key list.

## Coding style

The two processes follow different (consistent) conventions — match the surrounding file:

**Main process (`main/`)** — Node CommonJS:
- Every file starts with `'use strict';`.
- Declarations use `const` for imports/constants, `let` for everything else (no `var`).
- Functions are written as assigned expressions: `let doThing = function (args) { ... };`, not `function doThing() {}`.
- Modules export an object literal at the bottom via `module.exports = { ... }`.
- IPC handlers are registered top-level with `ipcMain.on(...)` / `ipcMain.handle(...)`.
- 4-space indentation.

**Renderer process (`app/`)** — AngularJS 1.6:
- Each file is an IIFE wrapper: `(function () { 'use strict'; ... }());`.
- Components register on the shared module: `angular.module('ariaNg').controller/factory/directive/filter(...)`.
- **Always use array-style dependency injection** (`['$scope', 'ariaNgSettingService', function ($scope, ariaNgSettingService) { ... }]`) — there is no minification-safe annotation pass, so inline DI arrays are mandatory.
- `var` (not `let`/`const`) — this is upstream AngularJS-era code.
- 4-space indentation.

