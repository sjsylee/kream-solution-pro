# Folder Structure

Annotated overview of every directory in the project.

```
kream-solution-pro/
│
├── electron-src/                    # ── Main Process ──────────────────────────
│   │                                #    Compiled to main/ by tsc
│   ├── index.ts                     # App entry point
│   │                                #   • Registers all IPC handlers
│   │                                #   • Manages app/window lifecycle
│   │                                #   • Single-instance lock
│   │                                #   • Boots secret key flow → main window
│   │                                #   • Schedules: update check, config refresh, backup
│   │
│   ├── preload.ts                   # contextBridge definition
│   │                                #   • Whitelists ipcRenderer methods
│   │                                #   • Only bridge between processes
│   │
│   ├── windows/                     # BrowserWindow factory functions
│   │   ├── mainWindow.ts            #   Main app window
│   │   ├── networkOfflineWindow.ts  #   Offline error overlay window
│   │   └── secretKeyWindow.ts       #   License activation window
│   │
│   ├── services/                    # ── Domain Service Layer ──────────────────
│   │   ├── accounts/
│   │   │   └── accountsManager.ts   # Account list, token storage, selection
│   │   │
│   │   ├── calc/                    # Settlement domain
│   │   │   ├── calcHandler.ts       #   IPC entry (handle registrations)
│   │   │   ├── calcManager.ts       #   Orchestration logic
│   │   │   ├── calcRepo.ts          #   SQLite queries
│   │   │   ├── calcDb.ts            #   DB singleton (WAL mode)
│   │   │   ├── calcSingleton.ts     #   Manager singleton
│   │   │   ├── genKey.ts            #   Record key generation
│   │   │   ├── types.ts             #   Domain types
│   │   │   └── purchaseStore/       #   Purchase record sub-domain
│   │   │
│   │   ├── current/                 # Inventory tracking domain
│   │   │   └── …                    #   (same layered pattern as calc)
│   │   │
│   │   ├── favorite/                # Watchlist domain
│   │   │   ├── favOptionsHandler.ts #   Single-item options IPC
│   │   │   ├── favBulkHandler.ts    #   Bulk operations IPC
│   │   │   ├── favOptionsManager.ts #   Business logic
│   │   │   └── …
│   │   │
│   │   ├── bid/                     # Bidding automation domain
│   │   │   └── bidBulkHandler.ts
│   │   │
│   │   ├── trades/                  # Trade history domain
│   │   │   ├── tradesHandler.ts
│   │   │   ├── tradesDb.ts          #   Separate SQLite file
│   │   │   └── …
│   │   │
│   │   ├── products/
│   │   │   └── kream/               # KREAM product metadata cache
│   │   │       ├── kreamProductsHandler.ts
│   │   │       ├── kreamProductsDb.ts  # Separate SQLite file
│   │   │       └── kreamProductsSingleton.ts
│   │   │
│   │   ├── kream/                   # Platform HTTP client
│   │   │   ├── kream_auth.ts        #   Token-based auth
│   │   │   ├── kream_data.ts        #   Data fetch endpoints
│   │   │   └── config/
│   │   │       └── kream_config.ts  #   Remote config polling (Cloudflare Worker)
│   │   │
│   │   ├── secret/                  # License key management
│   │   │   └── secretKeyManager.ts  #   Keytar read/write + validation flow
│   │   │
│   │   ├── apiHandlers.ts           # Top-level IPC handler aggregator
│   │   ├── autoUpdater.ts           # electron-updater wrapper + GitHub release check
│   │   └── utils.ts                 # Shared main-process utilities
│   │
│   ├── loop/                        # ── Async Task Managers ───────────────────
│   │   ├── prdData collectionManager.ts    # Product data collection queue (2-phase)
│   │   │                            #   • Primary: page-level listing collection
│   │   │                            #   • Detail: per-item data enrichment
│   │   │                            #   • Rate-limit retry (30s backoff)
│   │   │                            #   • Real-time progress push to renderer
│   │   ├── calcTaskManager.ts       # Settlement calculation runner
│   │   ├── currentTaskManager.ts    # Inventory sync runner
│   │   └── storedTaskManager.ts     # Stored/historical task runner
│   │
│   ├── backup/                      # ── Backup System ─────────────────────────
│   │   ├── backupScheduler.ts       # Cron-style timer, triggers runBackupOnce()
│   │   ├── backupRunner.ts          # Core: snapshot DBs → zip bundle → cloud path
│   │   ├── backupScanner.ts         # Lists existing bundles for restore UI
│   │   ├── backupSettings.ts        # Persisted backup config (path, keepCount)
│   │   ├── buildBundle.ts           # archiver wrapper
│   │   ├── detectCloud.ts           # iCloud / OneDrive path detection per OS
│   │   ├── restoreRunner.ts         # Unzip + replace active DB files
│   │   ├── sqliteSnapshot.ts        # SQLite backup API wrapper
│   │   └── backupIpc.ts             # IPC handlers for backup/restore UI actions
│   │
│   ├── status/                      # ── Status Broadcasting ───────────────────
│   │   ├── appStatusStore.ts        # In-memory status state (net, kream)
│   │   ├── netChecker.ts            # Single HTTP probe for connectivity
│   │   ├── netMonitor.ts            # Interval-based net check → publish
│   │   ├── netGate.ts               # Boot-time gate: wait for net or show offline window
│   │   ├── statusBroadCaster.ts     # webContents.send wrapper
│   │   ├── statusIpc.ts             # IPC handlers for status queries
│   │   └── kream/                   # Kream-specific status (blocked/throttleed)
│   │
│   ├── constants/                   # App constants (keys, IDs, ports)
│   ├── assets/                      # Bundled assets (e.g. calc template XLSX)
│   └── tsconfig.json                # Electron-side TypeScript config
│
├── renderer/                        # ── Renderer Process ──────────────────────
│   │                                #    Next.js static export (output: 'export')
│   │
│   ├── pages/                       # Route pages (file-based routing)
│   │   ├── _app.tsx                 #   Global provider, layout, IPC hook mount
│   │   ├── _document.tsx            #   HTML shell
│   │   ├── index.tsx                #   Dashboard / data collection entry
│   │   ├── calc.tsx                 #   Settlement management
│   │   ├── current.tsx              #   Inventory status
│   │   ├── favorite.tsx             #   Watchlist
│   │   ├── bidding.tsx              #   Bid automation
│   │   ├── offline.tsx              #   Network error page (separate window)
│   │   └── secret.tsx               #   License activation page
│   │
│   ├── components/                  # UI component library
│   │   ├── Card/                    #   Feature cards (largest component group)
│   │   │   ├── Calc/                #     Settlement cards
│   │   │   ├── Current/             #     Inventory cards
│   │   │   ├── Favorite/            #     Watchlist cards
│   │   │   └── Data collection/            #     Data collection control cards
│   │   ├── Modal/                   #   Dialog / drawer components
│   │   ├── Table/                   #   Data table variants
│   │   ├── Chart/                   #   Recharts wrappers
│   │   ├── Layout/                  #   MainLayout, sidebar, navigation
│   │   ├── Pop/                     #   Popover / tooltip variants
│   │   ├── Popconfirm/              #   Confirmation dialogs
│   │   ├── Button/                  #   Styled button variants
│   │   ├── Input/                   #   Input field variants
│   │   ├── Select/                  #   Select component variants
│   │   ├── Alert/                   #   Alert / notification components
│   │   ├── List/                    #   List components
│   │   ├── Empty/                   #   Empty state components
│   │   ├── Effect/                  #   Decorative effects (snow overlay, etc.)
│   │   └── Icon/                    #   Icon wrappers
│   │
│   ├── stores/                      # Zustand state management
│   │   ├── calc/                    #   Settlement state slices
│   │   ├── current/                 #   Inventory state slices
│   │   ├── favorite/                #   Watchlist state slices
│   │   ├── effect/                  #   UI effect state (snow toggle, etc.)
│   │   ├── useData collectionStore.ts      #   Data collection queue, cache, filter, progress
│   │   ├── useAccountsStore.ts      #   Account list + selection
│   │   ├── useBackupStore.ts        #   Backup status + bundle list
│   │   ├── useSecretKeyStore.ts     #   License key status
│   │   ├── useSellerLevelStore.ts   #   Seller tier data
│   │   ├── useStatusStore.ts        #   Net / Kream platform status flags
│   │   ├── useStoredPayStore.ts     #   Stored payment data
│   │   ├── useStoredTaskStore.ts    #   Stored task queue state
│   │   └── useUpdateStore.ts        #   OTA update state
│   │
│   ├── hooks/                       # IPC sync + business hooks
│   │   ├── calc/                    #   Settlement IPC hooks
│   │   ├── current/                 #   Inventory IPC hooks
│   │   ├── favoirte/                #   Watchlist IPC hooks
│   │   ├── useData collectionSync.ts       #   Data collection progress listener
│   │   ├── useCalcSync.ts           #   Calc result listener
│   │   ├── useStoredLoopSync.ts     #   Stored task listener
│   │   ├── useBackupSync.ts         #   Backup status listener
│   │   ├── useStatusListener.ts     #   Net / Kream status listener
│   │   ├── useUpdater.ts            #   OTA update event listener
│   │   ├── useStatsInitialLoad.ts   #   Initial data load on app start
│   │   └── useDeferredReady.ts      #   Deferred mount helper
│   │
│   ├── utils/                       # Pure utility functions
│   │   ├── calculate.ts             #   Fee / margin calculation formulas
│   │   ├── analyze.ts               #   Product opportunity evaluation
│   │   ├── filter.ts                #   Data collection result filter logic
│   │   ├── trades.ts                #   Trade data transformations
│   │   ├── favRows.ts               #   Favorite list row helpers
│   │   ├── marginBand.ts            #   Margin band classification
│   │   ├── margin.ts                #   Margin calculation helpers
│   │   ├── categoryTree.ts          #   Category hierarchy builder
│   │   ├── renderers/               #   Table cell renderer functions
│   │   ├── frameBox.ts              #   Layout box helpers
│   │   ├── resellStorageType.ts     #   Resell storage type enum
│   │   ├── singleFlight.ts          #   Dedup concurrent async calls
│   │   ├── stylesConstant.ts        #   Shared style constants
│   │   ├── day.ts                   #   Date formatting
│   │   ├── etc.ts                   #   Miscellaneous helpers
│   │   └── uid.ts                   #   UUID generation
│   │
│   ├── interfaces/                  # Shared TypeScript interfaces
│   ├── constants/                   # App-wide constants (categories, couriers, fees)
│   ├── theme/                       # Ant Design theme config
│   ├── styles/                      # Global CSS + color tokens
│   ├── public/                      # Static assets (fonts, images)
│   └── tsconfig.json                # Renderer TypeScript config
│
├── shared/                          # Shared types (Main ↔ Renderer)
│   └── manualBuysTypes.ts
│
├── build/                           # electron-builder resources (icons)
├── next.config.js                   # Next.js config (static export, antd transpile)
├── babel.config.js                  # Babel config
├── package.json                     # Dependencies, scripts, electron-builder config
├── upload-release.sh                # Release upload helper (macOS)
└── upload-release.ps1               # Release upload helper (Windows)
```
