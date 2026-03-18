# Architecture Deep Dive

## Process Model

KREAM Solution Pro follows Electron's **dual-process model** with a hard security boundary between Main and Renderer.

```
┌──────────────────────────────────────────────────────────────┐
│                     Electron Shell                           │
│                                                              │
│  ┌─────────────────────────┐    ┌──────────────────────────┐ │
│  │     Main Process        │    │    Renderer Process       │ │
│  │  (full Node.js access)  │    │  (browser sandbox)        │ │
│  │                         │    │                           │ │
│  │  index.ts               │    │  Next.js static export    │ │
│  │  ├─ window factories    │    │  ├─ /pages (5 routes)     │ │
│  │  ├─ IPC handlers        │◄──►│  ├─ /stores (Zustand)     │ │
│  │  ├─ domain services     │IPC │  ├─ /hooks (IPC sync)     │ │
│  │  ├─ task loop managers  │    │  └─ /components (UI)      │ │
│  │  ├─ backup scheduler    │    │                           │ │
│  │  ├─ net monitor         │    │  window.electron (bridge) │ │
│  │  └─ auto updater        │    └──────────────────────────┘ │
│  │                         │                                 │
│  │  ┌─────────────────────┐│                                 │
│  │  │   Data Layer        ││                                 │
│  │  │  • SQLite ×4 (WAL)  ││                                 │
│  │  │  • electron-store   ││                                 │
│  │  │  • Keytar (Keychain)││                                 │
│  │  └─────────────────────┘│                                 │
│  └─────────────────────────┘                                 │
└──────────────────────────────────────────────────────────────┘
```

---

## IPC Communication Pattern

All renderer ↔ main communication goes through a **contextBridge whitelist** defined in `preload.ts`. The renderer never has direct access to Node.js APIs.

```
Renderer (React hook)
  │  window.electron.ipcRenderer.invoke('channel', payload)
  ▼
preload.ts (contextBridge)
  │  ipcRenderer.invoke(channel, ...args)
  ▼
Main Process (ipcMain.handle)
  │  executes service logic
  ▼
Response returned as Promise
  │
  ▼
Renderer receives result, updates Zustand store
```

**Example — Settlement calculation flow:**

```
useCalcIpc (hook)
  └─ ipcRenderer.invoke('calc:run', options)
       └─ calcHandler.ts (ipcMain.handle)
            └─ calcManager.ts (business logic)
                 └─ calcRepo.ts (SQLite queries)
                      └─ calcDb.ts (better-sqlite3 singleton)
```

**Push events** (main → renderer) use `ipcRenderer.on` listeners registered in custom hooks:

```
prdData collectionManager.ts
  └─ mainWindow.webContents.send('data collection:progress', data)
       └─ useData collectionSync.ts (hook)
            └─ useData collectionStore.ts (Zustand update)
                 └─ UI re-renders
```

---

## Domain Services

Each domain is fully isolated with its own handler, manager, repository, DB, and types.

```
services/
├── calc/          Settlement: fee calculation, purchase records, Excel export
├── current/       Inventory: resell + re-resell positions
├── favorite/      Watchlist: options, bulk operations
├── bid/           Bidding: bulk bid automation
├── trades/        Trade history: KREAM platform sync
├── products/      Product metadata + caching layer
├── accounts/      Multi-account + token management
├── kream/         Platform HTTP client (auth, data endpoints)
└── secret/        License key validation (Keychain-backed)
```

Each domain follows the same layered pattern:

```
*Handler.ts      ← IPC registration (ipcMain.handle)
    └─ *Manager.ts   ← Business logic orchestration
         └─ *Repo.ts     ← SQLite queries
              └─ *Db.ts      ← better-sqlite3 singleton (WAL mode)
```

---

## Task Loop Architecture

Long-running async operations are managed by **task managers** in `electron-src/loop/`. They maintain internal state (queue, progress counters, retry state) and push updates to the renderer via `webContents.send`.

```
prdData collectionManager.ts   — product data collection queue (2-phase: primary + detail)
calcTaskManager.ts      — settlement calculation loop
currentTaskManager.ts   — inventory sync loop
storedTaskManager.ts    — stored/history task runner
```

**Two-phase data collection pattern:**

```
Phase 1 (Primary)                    Phase 2 (Detail)
─────────────────                    ────────────────
For each page in [startPage..endPage]:   For each item in cachedPrimaryData:
  GET listing page                         GET detail + option data
  push items to cache                      enrich item in cache
  send 'data collection:primary-progress'         send 'data collection:detail-progress'
  await primaryDelay (1800ms)              await detailDelay (5200ms)
  
  On throttle block:
    set retryState {active, nextRetryAt}
    send 'data collection:retry-state'
    await BLOCK_RETRY_DELAY_MS (30s)
    retry same page
```

---

## State Management (Renderer)

Zustand stores are organized **per domain** with co-located actions.

```
stores/
├── calc/
│   ├── useCalcStore.ts         — calculation results & status
│   └── useCalcViewStore.ts     — UI view state (resell / re-resell tab)
├── current/
│   ├── useCurrentStore.ts      — inventory data
│   └── useCurViewStore.ts      — view toggle
├── favorite/
│   └── useFavStore.ts
├── useData collectionStore.ts         — data collection queue, cache, filter, progress
├── useAccountsStore.ts         — multi-account selection
├── useBackupStore.ts           — backup status & bundle list
├── useStatusStore.ts           — net / kream status flags
└── useUpdateStore.ts           — OTA update state
```

Stores are synced to Main Process via **IPC sync hooks** (`/hooks/**/*Sync.ts`). Hooks mount `ipcRenderer.on` listeners and dispatch to Zustand on each push event.

---

## Backup System

```
backupScheduler.ts  — cron-style timer, triggers runBackupOnce()
backupRunner.ts
  ├─ snapshotDbToFile()    SQLite backup API → temp file
  ├─ favStore copy         electron-store JSON
  ├─ buildZipBundle()      archiver → {runId}_bundle.zip
  ├─ write _meta.json      manifest with included files
  └─ prune old zips        keep N most recent (configurable)

detectCloud.ts  — resolves iCloud / OneDrive root path per OS
backupScanner.ts — lists existing bundles for restore UI
restoreRunner.ts — unzip → replace active DB files
```

All backups go to: `{cloudRoot}/YJCODE/{appId}/backups/{YYYY-MM-DD}/{runId}_bundle.zip`

Note: Keytar-stored credentials (account tokens) are **intentionally excluded** from backup bundles — they live in the OS Keychain and cannot be exported.

---

## Security Model

| Concern | Approach |
|---|---|
| License key storage | OS Keychain via `keytar` — never written to disk |
| Account credentials | OS Keychain via `keytar` — never in electron-store |
| Renderer isolation | `nodeIntegration: false`, `contextIsolation: true` |
| IPC surface | Whitelisted through `contextBridge` in preload.ts |
| Update integrity | `autoUpdater.verifyUpdateCodeSignature` + GitHub Releases |

---

## Build Pipeline

```
npm run dist
  ├─ next build renderer     → renderer/out/ (static HTML/JS)
  ├─ tsc -p electron-src     → main/ (compiled JS)
  └─ electron-builder
       ├─ macOS: DMG + .icns icon
       └─ Windows: NSIS installer + ZIP
            artifact: kream-solution-pro-Setup-{version}.exe

Release delivery:
  GitHub Releases (public repo for update binary)
  └─ autoUpdater polls every 10 min via electron-updater
  └─ GitHub API polled every 1 hour for release notes
```
