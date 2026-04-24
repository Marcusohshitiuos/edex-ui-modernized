# eDEX-UI Modernization Roadmap

Archived Oct 2021, v2.2.8. Electron 12, Node.js dependency ecosystem from mid-2021.
Goal: get this running on a Nobara (Fedora) desktop and keep it maintainable.

---

## Phase 0 — Get It Building and Running

**Scope**

Make the app launch on the current host without touching any source code. This means:
- Resolving native module compilation (`node-pty` requires rebuilding for the installed Electron ABI)
- Working around any immediate npm install failures caused by deprecated build scripts or missing peer deps
- Confirming the `electron src --nointro` dev-run path works end-to-end

The `install-linux` script in root `package.json` is the intended entry point:
```
npm install && cd src && npm install && ../node_modules/.bin/electron-rebuild -f -w node-pty
```

Likely friction points:
- `node-pty` 0.10.1 may fail to compile against Node 22 / Electron 12's expected ABI
- `electron-rebuild` 2.x has a dependency on `node-abi` 2.x which has an outdated ABI table — may not know about Node 22
- Python 2/3 and `node-gyp` must be available for the native build (`python3-devel`, `make`, `gcc` via `dnf`)
- `geolite2-redist` downloads a ~70 MB GeoIP database at install time; may time out or 404

**Difficulty:** S  
**What breaks if skipped:** Everything. Nothing can be validated until the app runs.

**Acceptance criteria**
- `electron src --nointro` launches without crashing
- A terminal tab opens and accepts input
- Left/right monitor columns render (even if system stats show zeros)
- No fatal errors in the Electron main process console

**Commit strategy**
```
fix: resolve node-pty native build for Node 22 / Electron 12 ABI
fix: work around <specific install-time failure>
chore: document build prerequisites for Fedora/Nobara
```

---

## Phase 1 — Dependency Upgrades (Electron-API-neutral)

**Scope**

Update dependencies whose upgrade path does not require changes to Electron IPC, `contextIsolation`, or `nodeIntegration`. These can be done while still on Electron 12 and validated against the running app.

| Package | From | To | Risk |
|---------|------|----|------|
| `xterm` | 4.14.1 | 5.x | Breaking API changes in v5: `Terminal.element` → `Terminal.element`, addon constructors, `onData` vs `onBinary` |
| `xterm-addon-attach` | 0.6.0 | 0.9.x (xterm 5-compatible) | Addon APIs changed |
| `xterm-addon-fit` | 0.5.0 | 0.8.x | Minor |
| `xterm-addon-ligatures` | 0.5.1 | 0.7.x | Minor |
| `xterm-addon-webgl` | 0.11.2 | 0.18.x | Constructor signature changed |
| `pdfjs-dist` | 2.11.338 | 4.x | `getPage()` return shape, `render()` API, worker loading changed significantly |
| `nanoid` | 3.x (CJS) | keep 3.x pinned OR update carefully | v4+ is ESM-only; the `nanoid/non-secure` subpath import used in `_renderer.js:179` breaks on v4+ |
| `electron-rebuild` | 2.3.5 | `@electron/rebuild` 3.x | Package renamed; adjust `install-linux` script |
| `node-abi` | 2.30.1 | 3.x | ABI lookup table; update in root package.json |
| `geolite2-redist` | ~2021 snapshot | latest | Drop-in; GeoIP accuracy improvement only |
| `smoothie` | 1.35.0 | 1.36.x | Minor; check `SmoothieChart` constructor API |

Changes required in `src/classes/terminal.class.js`:
- xterm 5 removed the `Terminal` default export in favour of named export; `require("xterm").Terminal` still works but check addon constructor changes
- `WebglAddon` in v5 requires explicit `onContextLoss` handling

Changes required in `src/classes/docReader.class.js`:
- pdfjs v3+ changed `getDocument()` options and the `render()` task API; the worker path resolution also changed

**Difficulty:** M  
Xterm 4→5 and pdfjs 2→4 each have their own mini-migration. The others are near drop-in.

**What breaks if skipped**
- `nanoid` stays pinned at 3.x which is fine for now; skipping the rest means Phase 3 (Electron bump) will have to absorb these migrations simultaneously under a much larger diff
- pdfjs 2.x has known CVEs

**Acceptance criteria**
- `electron src --nointro` still launches cleanly after each package bump
- Terminal renders, accepts input, scrolls, and ligatures work
- PDF files open in the doc reader without console errors
- No `xterm`-related console warnings about deprecated APIs
- `electron-rebuild` (as `@electron/rebuild`) still compiles `node-pty` successfully

**Commit strategy**
```
chore: replace electron-rebuild with @electron/rebuild, update node-abi
chore: bump xterm and addons to v5-compatible versions
fix: update terminal.class.js for xterm 5 API changes
chore: bump pdfjs-dist to 4.x
fix: update docReader.class.js for pdfjs v4 API changes
chore: refresh geolite2-redist GeoIP database
chore: pin nanoid at 3.x with explanatory comment (v4+ ESM-only)
```

---

## Phase 2 — The Blocker: Preload / contextBridge Migration

**Scope**

This is the hardest phase and the gate for Phase 3. The entire renderer currently runs with:

```js
webPreferences: {
    nodeIntegration: true,
    contextIsolation: false,
    enableRemoteModule: true,
}
```

This security model was deprecated in Electron 12 and is gone in Electron 28+. Every `require()` call in the renderer and every use of `@electron/remote` must be replaced.

### What needs to change

**1. Create a preload script (`src/preload.js`)**

The preload script runs in a privileged context with Node.js access. It exposes a curated API to the renderer via `contextBridge.exposeInMainWorld('edex', { ... })`. Nothing else.

**2. Replace all `@electron/remote` usage in `_renderer.js`**

| Current call | Replacement |
|---|---|
| `remote.app.getVersion()` | expose via contextBridge |
| `remote.app.getPath('userData')` | expose via contextBridge |
| `remote.app.relaunch()` / `app.quit()` | IPC call to main |
| `remote.app.focus()` | IPC call to main |
| `remote.screen.getAllDisplays()` | IPC call to main |
| `remote.getCurrentWindow().webContents.toggleDevTools()` | IPC call to main |
| `remote.getCurrentWindow().isFullScreen()` / `setFullScreen()` | IPC call to main |
| `remote.getCurrentWindow().setSize()` | IPC call to main |
| `remote.globalShortcut.register()` / `unregisterAll()` | IPC call to main (register shortcuts in main, trigger actions via IPC) or use `ipcRenderer.invoke` |
| `remote.clipboard.readText()` | `navigator.clipboard.readText()` (Web API) |
| `electron.remote.process.argv` | expose `process.argv` slice via contextBridge at startup |
| `electron.shell.openPath()` | IPC call to main → `shell.openPath()` |

**3. Replace `require()` in renderer class files**

Every class file in `src/classes/` uses bare `require()` because `nodeIntegration: true` made it a Node.js module. With `nodeIntegration: false`, these become plain browser scripts. Options:

- **Minimal approach**: Keep loading via `<script>` tags but replace each `require()` with either a value injected by the preload or a dynamic import from the built bundle.
- **Cleaner approach** (overlaps with Phase 4): introduce a bundler (vite) so class files become proper ES modules.

The minimal approach is safer to do in isolation here. Each `require()` in the classes falls into one of these categories:

| `require(...)` call location | Replacement strategy |
|---|---|
| `require("electron").ipcRenderer` (multiple classes) | expose `ipc` via contextBridge |
| `require("electron").webFrame` | expose via contextBridge or drop (used only for zoom limits) |
| `require("@electron/remote")` | eliminate entirely (see table above) |
| `require("path")` | replace with `path` polyfill or pass resolved paths from preload |
| `require("fs")` | expose a scoped `fs` API via contextBridge (readFile, writeFile, readdir for known dirs only) |
| `require("os")` | expose specific values (platform, type) via contextBridge |
| `require("nanoid/non-secure")` | already a renderer-side npm dep; import via bundler or inline |
| `require("color")` | renderer-side npm dep; same |
| `require("xterm*")` | renderer-side npm dep; same |
| `require("howler")` | renderer-side npm dep; same |
| `require("pdfjs-dist")` | renderer-side npm dep; same |

**4. `_boot.js` changes**

- Remove `enableRemoteModule: true`
- Set `contextIsolation: true`
- Add `preload: path.join(__dirname, 'preload.js')`
- Replace deprecated `contents.on('new-window', …)` with `contents.setWindowOpenHandler()`
- Replace `require('@electron/remote/main').initialize()` with nothing (remote is gone)

**5. `_multithread.js`**

- Replace `cluster.setupMaster()` with `cluster.setupPrimary()` (one-line fix)

**6. `_renderer.js` miscellaneous**

- Replace `document.execCommand('copy')` in clipboard handler with `navigator.clipboard.writeText(selection)`
- Replace `window.performance.navigation.type === 1` with `PerformanceNavigationTiming` or simply remove the hot-reload guard (it's a cosmetic optimization)

**Difficulty:** XL  
This phase touches `_boot.js`, `_renderer.js`, `preload.js` (new), and all 18 class files. The risk is regression in any of the ~15 subsystems. Do it in small, runnable commits.

**What breaks if skipped**
Phase 3 is impossible. The app will refuse to start on any Electron version > 27 with `contextIsolation: true` enforced by default.

**Acceptance criteria**
- `BrowserWindow` created with `contextIsolation: true`, `nodeIntegration: false`, `enableRemoteModule` absent
- No `@electron/remote` imports anywhere in renderer or class files
- No bare `require()` calls in renderer context (all Node APIs go through contextBridge)
- `cluster.setupPrimary()` used in `_multithread.js`
- All existing functionality works: terminal, filesystem viewer, system monitors, keyboard, theming, audio, settings editor, tab switching
- `electron src --nointro` launches on Electron 12 (Phase 3 is the version bump — this phase validates the migration on the known-working Electron version first)

**Commit strategy**
```
feat: add preload.js with contextBridge skeleton
refactor: expose app metadata and paths via contextBridge, remove remote.app usage
refactor: replace remote.getCurrentWindow / screen / globalShortcut with IPC handlers
refactor: expose scoped fs API via contextBridge, remove require('fs') from renderer
refactor: expose ipcRenderer and path utilities via contextBridge
refactor: remove require() from terminal.class.js
refactor: remove require() from filesystem.class.js
refactor: remove require() from keyboard.class.js
refactor: remove require() from system monitor classes (clock, cpuinfo, netstat, …)
refactor: remove require() from audiofx, modal, updateChecker
fix: replace document.execCommand('copy') with navigator.clipboard
fix: replace contents.on('new-window') with setWindowOpenHandler in _boot.js
fix: replace cluster.setupMaster with cluster.setupPrimary in _multithread.js
chore: set contextIsolation=true, nodeIntegration=false in BrowserWindow
```

---

## Phase 3 — Electron Version Bump

**Scope**

With Phase 2 complete, the app runs on Electron 12 with a clean security model. Now bump to Electron 29+ (or whatever the current stable LTS is at the time).

Steps:
1. Update `"electron"` in root `package.json`
2. Rebuild `node-pty` for the new Electron ABI via `@electron/rebuild`
3. Work through any remaining deprecated API warnings that only appear in the new version
4. Update `electron-builder` to a version that supports the target Electron
5. Verify AppImage packaging still produces a working binary

Expected breakage to triage:
- `ipcRenderer` / `ipcMain` API surface changes between 12 and 29 are minimal but verify `ipc.once` vs `ipc.handle` usage
- WebGL renderer in xterm may behave differently under the newer Chromium; watch for `WebglAddon` context-loss issues
- `globalShortcut` behavior differences (now registered in main after Phase 2 refactor, so this should be cleaner)
- `webFrame.setVisualZoomLevelLimits` may have moved or changed

**Difficulty:** M  
The hard work was Phase 2. This phase is mostly: bump version, rebuild native, fix whatever yells.

**What breaks if skipped**
The app stays frozen on a 2021 Electron with known security vulnerabilities and no vendor support. Eventually the OS or system Node.js will drift far enough that even the Phase 0 build fails.

**Acceptance criteria**
- `electron --version` in the dev run shows 29.x or later
- `node-pty` compiles cleanly against the new ABI
- All Phase 0 acceptance criteria still pass
- No `[Deprecation]` or `[Warning]` lines in the Electron main process log
- AppImage builds and launches on Fedora

**Commit strategy**
```
chore: bump electron to 29.x, update electron-builder
fix: rebuild node-pty for Electron 29 ABI
fix: address Electron 29 deprecation warnings
chore: update CI workflow Node.js and Electron versions
```

---

## Phase 4 — Optional QoL Improvements

**Scope**

Nothing here is required for a working, maintainable app. These are quality-of-life improvements worth doing if the codebase sees continued development.

### 4a — Introduce a module bundler (Vite)

Replace the raw `<script>` tag loading chain in `ui.html` with Vite + `vite-plugin-electron`. Benefits:
- Class files become proper ES modules (eliminate global namespace pollution)
- Tree-shaking removes dead code
- Hot module replacement speeds up development iteration
- Opens the door to TypeScript

Difficulty: M — requires restructuring all class files as ES modules with explicit exports.

### 4b — TypeScript

Add `tsconfig.json`, rename class files to `.ts`, add type annotations incrementally. The xterm and systeminformation packages ship their own `.d.ts` files, so the terminal and monitor subsystems get the most immediate benefit.

Difficulty: M (with bundler already in place) / L (without bundler)

### 4c — ESLint + Prettier

Add `.eslintrc` targeting ES2022 browser + Node. Add `prettier` config. Hook into pre-commit via `lint-staged`. Mostly mechanical but catches the remaining deprecated patterns (`var` declarations, `==` vs `===`, etc.).

Difficulty: S

### 4d — Automated tests

The class files have no test coverage. Add:
- Unit tests for the `Terminal` server-role PTY management logic (mockable because it's pure Node)
- Integration smoke test that launches Electron and asserts the window opens

Framework: `vitest` for unit, `@playwright/test` with Electron support for integration.

Difficulty: L — writing the first tests in an untested codebase always is.

### 4e — GeoIP update automation

The `geolite2-redist` database becomes stale. Add a `package.json` script that re-downloads the latest snapshot and commit-locks the version hash. Or replace with an online IP geolocation API (adds a network dependency, but removes the 70 MB binary).

Difficulty: S

**Acceptance criteria (per sub-phase)**
- 4a: `npm run dev` uses Vite HMR; `npm run build` produces the same app bundle
- 4b: `tsc --noEmit` passes with zero errors across all class files
- 4c: `eslint src/` passes with zero errors; `prettier --check` passes
- 4d: `vitest run` passes; Playwright smoke test passes in CI
- 4e: `npm run update-geoip` fetches fresh database without manual steps

---

## Dependency Version Pinning Notes

A few packages need special handling regardless of phase:

- **`nanoid`**: Keep pinned at `^3.x` (CJS). Version 4+ is ESM-only and the `nanoid/non-secure` subpath import used in `_renderer.js` does not exist in v4. If Phase 4a introduces a bundler, revisit — a bundler can handle ESM-only packages.
- **`node-pty`**: Native module. Every time the Electron version changes, `@electron/rebuild` must run. Document this in the install script.
- **`smoothie`**: Loaded from `node_modules` via a `<script>` tag in `ui.html` (implicit path). If a bundler is introduced, this needs to become an import.
- **`encom-globe.js`**: Vendored into `src/vendor/`. Not on npm. Any updates require manual replacement.
