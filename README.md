<div align="center">

# KREAM Solution Pro

**A production-grade cross-platform desktop productivity tool for KREAM sellers**  
**KREAM 판매자를 위한 프로덕션급 크로스플랫폼 데스크톱 생산성 툴**

[![Electron](https://img.shields.io/badge/Electron-27-47848F?style=flat-square&logo=electron&logoColor=white)](https://www.electronjs.org/)
[![Next.js](https://img.shields.io/badge/Next.js-15-000000?style=flat-square&logo=next.js&logoColor=white)](https://nextjs.org/)
[![TypeScript](https://img.shields.io/badge/TypeScript-4.9-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://www.typescriptlang.org/)
[![React](https://img.shields.io/badge/React-18-61DAFB?style=flat-square&logo=react&logoColor=black)](https://react.dev/)
[![SQLite](https://img.shields.io/badge/SQLite-WAL_Mode-003B57?style=flat-square&logo=sqlite&logoColor=white)](https://www.sqlite.org/)
[![Ant Design](https://img.shields.io/badge/Ant_Design-5-0170FE?style=flat-square&logo=antdesign&logoColor=white)](https://ant.design/)

> ⚠️ **This is a portfolio repository.** Core business logic is proprietary and not included.  
> ⚠️ **포트폴리오 전용 레포지토리입니다.** 핵심 서비스 로직은 포함되어 있지 않습니다.

</div>

---

## 📌 Overview / 개요

**EN** — KREAM Solution Pro is a commercial desktop application built for sellers on the [KREAM](https://kream.co.kr) platform. It consolidates multi-account management, product data collection with margin analysis, settlement tracking, inventory monitoring, and bid management into a single native app running on macOS and Windows.

**KR** — KREAM Solution Pro는 [KREAM](https://kream.co.kr) 플랫폼 판매자를 위한 상업용 데스크톱 애플리케이션입니다. 멀티 계정 관리, 마진 분석을 포함한 상품 데이터 수집, 정산 추적, 재고 현황 모니터링, 입찰 관리까지 — macOS와 Windows에서 단일 네이티브 앱으로 처리합니다.

---

## ✨ Key Features / 주요 기능

| Feature | Description (EN) | 설명 (KR) |
|---|---|---|
| 🔍 **Product Data Collection** | Two-phase data collection pipeline with configurable pacing and automatic retry on throttle | 2단계 데이터 수집 파이프라인, 딜레이 설정 및 스로틀 자동 재시도 |
| 📊 **Margin Analysis** | Real-time margin evaluation with configurable filter conditions | 실시간 마진 평가 및 필터 조건 설정 |
| 💰 **Settlement (정산)** | Fee calculation across multiple trade types with Excel export | 거래 유형별 수수료 계산 및 Excel 내보내기 |
| 📦 **Inventory (현황)** | Live inventory tracking across multiple position types | 포지션 유형별 재고 현황 실시간 추적 |
| ⭐ **Favorites** | Watchlist management with bulk action support | 즐겨찾기 관리 및 대량 작업 지원 |
| 🎯 **Bid Management** | Bulk bid submission with configurable strategies | 입찰 전략 설정 및 대량 입찰 관리 |
| 🔄 **Auto Update** | Silent background update via electron-updater + GitHub Releases | electron-updater + GitHub Releases 자동 업데이트 |
| ☁️ **Cloud Backup** | Scheduled SQLite + store backup to iCloud/OneDrive with rotation | 스케줄 백업 및 세대 관리, iCloud/OneDrive 자동 감지 |
| 🌐 **Net Gate** | Offline detection with graceful degradation UI | 오프라인 감지 및 서비스 차단 UI |
| 🔐 **Secret Key** | Activation key validation via OS Keychain (keytar) | OS Keychain 기반 라이선스 키 검증 |

---

## 🏗 Architecture / 아키텍처

The app follows the standard **Electron dual-process architecture** with a strict IPC boundary between Main and Renderer.

앱은 Main 프로세스와 Renderer 프로세스 사이의 엄격한 IPC 경계를 갖는 **Electron 이중 프로세스 아키텍처**를 따릅니다.

```
┌─────────────────── Electron Shell ────────────────────────┐
│                                                            │
│  ┌──────────────────────┐     ┌────────────────────────┐  │
│  │   Main Process       │     │  Renderer Process      │  │
│  │   (Node.js)          │     │  (Next.js static)      │  │
│  │                      │     │                        │  │
│  │  • IPC Handlers      │     │  • Pages (5 routes)    │  │
│  │  • Data Collection   │◄────►  • Zustand Stores      │  │
│  │  • Task Loop Mgr     │ IPC │  • Custom Hooks        │  │
│  │  • Backup Runner     │     │  • Ant Design 5 UI     │  │
│  │  • Net Monitor       │     │  • Recharts / Motion   │  │
│  │  • Secret Key Mgr    │     │  • react-window virt.  │  │
│  │  • Config Service    │     │                        │  │
│  │  • Domain Services   │     │  contextBridge preload │  │
│  └──────────┬───────────┘     └───────────┬────────────┘  │
│             │                             │               │
│  ┌──────────▼─────────────────────────────▼────────────┐  │
│  │              Data Layer                             │  │
│  │  SQLite ×4 (WAL)  │  electron-store  │  Keytar      │  │
│  └─────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────┘
                    │
     ┌──────────────┼──────────────────┐
     ▼              ▼                  ▼
 Platform API  GitHub Releases    CF Config API
                                  iCloud/OneDrive
```

> 📐 See [`docs/architecture.md`](./docs/architecture.md) for detailed component breakdown.

---

## 🗂 Project Structure / 프로젝트 구조

```
kream-solution-pro/
├── electron-src/              # Main Process (Node.js / TypeScript)
│   ├── index.ts               # App entry — window lifecycle, IPC registration
│   ├── preload.ts             # contextBridge — secure IPC surface
│   ├── windows/               # BrowserWindow factories
│   │   ├── mainWindow.ts
│   │   ├── networkOfflineWindow.ts
│   │   └── secretKeyWindow.ts
│   ├── services/              # Domain service layer
│   │   ├── accounts/          # Multi-account management
│   │   ├── calc/              # Settlement engine + SQLite repo
│   │   ├── current/           # Inventory tracking
│   │   ├── favorite/          # Watchlist management
│   │   ├── bid/               # Bid management
│   │   ├── trades/            # Trade history
│   │   ├── products/          # Product data + caching
│   │   ├── kream/             # Platform data layer
│   │   ├── secret/            # License key flow
│   │   └── autoUpdater.ts     # electron-updater wrapper
│   ├── loop/                  # Async task managers (data collection, calc, etc.)
│   │   ├── prdScrapingManager.ts
│   │   ├── calcTaskManager.ts
│   │   ├── currentTaskManager.ts
│   │   └── storedTaskManager.ts
│   ├── backup/                # Scheduled cloud backup system
│   │   ├── backupRunner.ts    # Zip bundle builder
│   │   ├── backupScheduler.ts
│   │   ├── backupScanner.ts
│   │   ├── detectCloud.ts     # iCloud / OneDrive path detection
│   │   └── restoreRunner.ts
│   └── status/                # App-wide status broadcasting
│       ├── netMonitor.ts
│       ├── netGate.ts
│       └── statusBroadCaster.ts
│
├── renderer/                  # Renderer Process (Next.js / React)
│   ├── pages/                 # Route pages
│   │   ├── index.tsx          # Dashboard / entry
│   │   ├── calc.tsx           # Settlement management
│   │   ├── current.tsx        # Inventory status
│   │   ├── favorite.tsx       # Watchlist
│   │   ├── bidding.tsx        # Bid automation
│   │   └── offline.tsx        # Network error page
│   ├── components/            # UI component library
│   │   ├── Card/              # Feature cards (Calc, Current, Fav, Scraping…)
│   │   ├── Modal/             # Dialog components
│   │   ├── Table/             # Data table variants
│   │   ├── Chart/             # Recharts wrappers
│   │   ├── Layout/            # MainLayout, sidebar
│   │   └── …                  # Button, Input, Select, Pop, etc.
│   ├── stores/                # Zustand state slices
│   │   ├── calc/
│   │   ├── current/
│   │   ├── favorite/
│   │   ├── useScrapingStore.ts
│   │   ├── useAccountsStore.ts
│   │   └── …
│   ├── hooks/                 # IPC sync hooks per domain
│   │   ├── calc/
│   │   ├── current/
│   │   ├── favoirte/
│   │   ├── useScrapingSync.ts
│   │   └── useUpdater.ts
│   └── utils/                 # Pure business helpers
│       ├── calculate.ts       # Fee / margin calculations
│       ├── analyze.ts         # Product evaluation logic
│       └── …
│
├── shared/                    # Shared TypeScript types
├── next.config.js             # Static export config + antd transpile
└── package.json
```

---

## 🛠 Tech Stack / 기술 스택

### Core Framework

| Layer | Technology | Role |
|---|---|---|
| Shell | **Electron 27** | Cross-platform desktop runtime |
| Renderer | **Next.js (static export)** | React page routing & build |
| Language | **TypeScript 4.9** | End-to-end type safety |
| IPC | **contextBridge + ipcMain.handle** | Secure process communication |

### Frontend

| Technology | Purpose |
|---|---|
| **React 18** | UI rendering |
| **Ant Design 5** + antd-style | Component library + CSS-in-JS theming |
| **Zustand 5** | Lightweight global state management |
| **Framer Motion 12** | Animations & transitions |
| **Recharts 2** | Charts & data visualization |
| **react-window** | Virtualized list rendering for large datasets |
| **@tabler/icons-react** | Icon set |

### Backend (Main Process)

| Technology | Purpose |
|---|---|
| **better-sqlite3** (WAL mode) | Embedded database ×4 instances |
| **electron-store** | JSON config persistence |
| **Keytar** | OS Keychain credential storage |
| **electron-updater** | OTA auto-update via GitHub Releases |
| **axios** | HTTP client for external data fetching & config |
| **archiver / adm-zip** | Backup bundle creation |
| **ExcelJS** | Excel report generation |

### Build & Distribution

| Technology | Purpose |
|---|---|
| **electron-builder** | Packaging — NSIS installer (Win) + DMG (Mac) |
| **electron-rebuild** | Native module rebuild for each Electron version |
| **GitHub Actions** | CI/CD pipeline |
| **GitHub Releases** | Update delivery channel |

---

## 🔑 Design Decisions / 설계 결정

### 1. Dual-DB architecture
Settlement, inventory, products, and trades are stored in **separate SQLite files**, each with its own singleton + WAL mode. This avoids lock contention and makes per-domain backup/restore trivial.

정산, 재고, 상품, 거래 데이터를 **별도의 SQLite 파일**로 분리하여 lock contention을 방지하고 도메인별 백업/복원을 단순화했습니다.

### 2. Two-phase data collection pipeline
Product data collection runs in two phases — **primary** (page-level listing) then **detail** (per-item enrichment) — with configurable pacing and automatic retry on throttle. Progress state is streamed to the renderer in real time.

상품 데이터 수집은 **1차 목록 수집 → 2차 상세 데이터 보강** 2단계로 실행되며, 페이스 설정 및 스로틀 자동 재시도를 지원합니다.

### 3. Preload-scoped IPC surface
All renderer-to-main calls go through a **typed `contextBridge` interface** (`window.electron.ipcRenderer`). The renderer has zero access to Node.js — all platform calls are whitelisted through the preload script.

렌더러의 모든 Main 프로세스 호출은 **타입이 지정된 contextBridge 인터페이스**를 통해 이루어집니다. 렌더러는 Node.js에 직접 접근할 수 없으며, 모든 플랫폼 호출은 preload 스크립트를 통해 화이트리스트 처리됩니다.

### 4. OS Keychain for secrets
License keys and account credentials are stored in the **OS native Keychain** via `keytar` — never written to disk or transmitted in plaintext.

라이선스 키와 계정 자격증명은 `keytar`를 통해 **OS 네이티브 Keychain**에 저장됩니다. 디스크에 기록되거나 평문으로 전송되지 않습니다.

### 5. Cloud backup with provider detection
The backup system auto-detects iCloud Drive / OneDrive root paths from OS-specific locations and saves daily zip bundles with configurable retention count.

백업 시스템은 OS별 경로에서 iCloud Drive / OneDrive 루트를 자동 감지하고, 보존 세대 수를 설정할 수 있는 일별 zip 번들로 저장합니다.

---

## 🧩 Challenges & Learnings / 개발 과정에서 마주한 문제들

Real technical problems encountered during development and how they were resolved.  
개발 중 실제로 맞닥뜨린 문제들과 해결 과정입니다.

---

### 1. SQLite concurrent access crash in a multi-domain app

**Problem** — Early versions used a single SQLite database file for all domains (calc, current, trades, products). As task loop managers began running concurrently in the Main Process, write operations from different domains started colliding, causing `SQLITE_BUSY` errors and occasional data corruption under load.

**Solution** — Split into **four independent SQLite files**, each managed by its own singleton with `journal_mode = WAL` and `synchronous = NORMAL`. WAL mode allows concurrent reads alongside a single writer without locking, which matched the app's actual access pattern (one writer loop per domain, many reads from IPC handlers).

**Lesson** — Domain isolation at the DB layer isn't just an architectural preference — in a multi-loop Electron app it's a correctness requirement.

> **문제** — 초기에는 모든 도메인이 단일 SQLite 파일을 공유했습니다. 메인 프로세스에서 복수의 태스크 루프가 동시에 실행되면서 도메인 간 쓰기 충돌이 발생했고 `SQLITE_BUSY` 에러와 데이터 손상으로 이어졌습니다.  
> **해결** — 도메인별로 SQLite 파일을 분리하고 각각 WAL 모드 싱글톤으로 관리했습니다. WAL은 단일 라이터와 다중 리더를 동시에 허용하여 루프-핸들러 혼재 환경에 적합했습니다.

---

### 2. Backup consistency — snapshotting a live SQLite database

**Problem** — A naive `fs.copyFileSync` on a live WAL-mode database produces an inconsistent snapshot: the main DB file and WAL file can be mid-transaction at copy time, resulting in a corrupt backup that fails to open on restore.

**Solution** — Used SQLite's `VACUUM INTO` command, which atomically writes a fully consistent, WAL-flushed copy to a new file path without touching the live database. A `wal_checkpoint(TRUNCATE)` is issued first to fold any pending WAL frames into the snapshot.

```typescript
db.pragma("wal_checkpoint(TRUNCATE)");
db.exec(`VACUUM INTO '${destPath}';`);
```

**Lesson** — File-level copy of a live database is almost never safe. The database engine's own snapshot API should always be the first choice.

> **문제** — 실행 중인 WAL 모드 DB를 `fs.copyFileSync`로 복사하면 메인 파일과 WAL 파일의 일관성이 보장되지 않아 복원 시 열리지 않는 손상된 백업이 만들어졌습니다.  
> **해결** — `VACUUM INTO`로 원자적인 일관성 있는 스냅샷을 생성했습니다. 파일 복사 대신 SQLite 엔진이 직접 쓰기 때문에 진행 중인 트랜잭션도 안전하게 처리됩니다.

---

### 3. Renderer state desync from long-running Main Process loops

**Problem** — When a data collection task runs for several minutes and the user navigates between pages, Zustand stores reset their "in-progress" state on unmount. On return, the UI showed an idle state even though the Main Process loop was still running.

**Solution** — IPC sync hooks are mounted **globally in `_app.tsx`**, not inside individual page components, so they survive all page navigations. On re-mount, a `scraping:get-state` invoke call rehydrates the store from the Main Process's in-memory state, ensuring the UI always reflects ground truth.

**Lesson** — In an Electron app, the source of truth for long-running operations belongs in the Main Process. The Renderer is a view — it should pull state on mount, not own it.

> **문제** — 수집 작업이 수 분간 실행되는 중 페이지를 이동하면 Zustand 스토어가 언마운트와 함께 초기화되어 돌아왔을 때 UI가 완료 상태로 표시됐습니다.  
> **해결** — IPC 싱크 훅을 `_app.tsx`에 글로벌하게 등록해 내비게이션 전반에 걸쳐 유지되도록 했습니다. 재진입 시에는 `invoke`로 메인 프로세스의 인메모리 상태를 리하이드레이션합니다.

---

### 4. UI freeze during high-frequency streaming updates on large tables

**Problem** — The result table renders thousands of rows. When the Main Process streams per-item detail updates (~one per 5 seconds), each `webContents.send` caused a full React re-render of the table, producing visible frame drops even with `react-window` virtualization in place.

**Solution** — Wrapped state updates in `startTransition` + `requestAnimationFrame` via a `useDeferredReady` hook, deferring non-urgent renders to after the current paint. Zustand updates were also made surgical — only the specific item in `cachedItems` is mutated via `map` rather than replacing the entire array reference.

**Lesson** — React 18's `startTransition` isn't just for loading states — it's essential for streaming-data UIs where high-frequency updates must not block interaction.

> **문제** — 상세 데이터가 스트리밍될 때마다 수천 개 항목의 테이블 전체가 리렌더링되어 `react-window` 가상화에도 불구하고 프레임 드롭이 발생했습니다.  
> **해결** — `startTransition` + `requestAnimationFrame`을 조합한 `useDeferredReady` 훅으로 비긴급 렌더를 다음 페인트 이후로 미루고, Zustand 업데이트도 배열 전체 교체 대신 `map`으로 해당 아이템만 교체하도록 최적화했습니다.

---

### 5. Throttle detection and graceful retry without losing progress

**Problem** — The platform's data layer throttles requests after a certain volume. An unhandled throttle response would crash the collection loop mid-run, losing all accumulated primary-phase data and forcing a full restart.

**Solution** — The loop detects throttle responses and enters a **structured retry state** — recording `{ stage, target, attempt, waitMs, nextRetryAt }` and broadcasting it to the Renderer so the user sees a live countdown rather than a frozen UI. After the backoff window (30s), the loop resumes from the exact page or item that was blocked, with all previously collected data intact in the in-memory cache.

**Lesson** — In any data collection pipeline, partial-progress preservation and transparent retry UX are as important as the collection logic itself.

> **문제** — 플랫폼이 요청을 스로틀할 경우 루프가 중간에 중단되면서 1차 수집한 데이터가 전부 손실됐습니다.  
> **해결** — 스로틀 감지 시 `retryState` 객체를 구성해 렌더러에 브로드캐스트하고, 사용자에게는 카운트다운 UI를 보여주면서 30초 후 차단된 정확한 위치부터 재시도합니다. 기존 수집 데이터는 인메모리 캐시에 그대로 유지됩니다.

---

### 6. Credential security in a desktop app without a backend

**Problem** — Account passwords and tokens must persist across app restarts. Storing them in `electron-store` (a plain JSON file) would leave them on disk in plaintext, readable by any process with file-system access.

**Solution** — All sensitive credentials are stored exclusively in the **OS native Keychain** via `keytar` (Credential Manager on Windows, Keychain on macOS). `electron-store` holds only non-sensitive metadata (email, displayName). Even if the app's data directory is exfiltrated, credentials cannot be recovered without OS-level access.

**Lesson** — Desktop apps face a different threat model than web apps. The absence of a backend doesn't mean security can be deferred — it means OS security primitives must be used directly.

> **문제** — 계정 비밀번호와 토큰을 `electron-store`(평문 JSON)에 저장하면 파일 시스템 접근 권한만 있으면 누구든 읽을 수 있습니다.  
> **해결** — 민감한 자격증명은 전부 `keytar`를 통해 OS 네이티브 Keychain에만 저장합니다. 앱 데이터 디렉토리가 유출되더라도 자격증명은 복구할 수 없습니다.

---

## 📸 Screenshots / 스크린샷

> Screenshots will be added here.  
> 스크린샷은 여기에 추가될 예정입니다.

---

## 📄 Documentation / 문서

- [`docs/architecture.md`](./docs/architecture.md) — Detailed architecture breakdown
- [`docs/folder-structure.md`](./docs/folder-structure.md) — Full folder structure explanation
- [`docs/ipc-flow.md`](./docs/ipc-flow.md) — IPC communication patterns

---

## 👤 Author / 개발자

**YJ CODE** — Solo developer / 1인 개발

---
</div>
