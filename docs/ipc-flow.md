# IPC Communication Patterns

This document covers how Main and Renderer processes communicate in KREAM Solution Pro.

---

## Channels

All IPC channels follow a `domain:action` naming convention.

### Request-Response (invoke / handle)

Used for operations that return data — queries, calculations, CRUD operations.

```
Renderer                         Main Process
   │                                 │
   │ ipcRenderer.invoke(ch, args)    │
   │────────────────────────────────►│
   │                                 │ ipcMain.handle(ch, handler)
   │                                 │ executes async logic
   │◄────────────────────────────────│
   │ returns Promise<result>         │
```

**Registered handlers (examples):**

| Channel | Handler | Description |
|---|---|---|
| `calc:run` | calcHandler | Run settlement calculation |
| `calc:export-excel` | calcHandler | Export to XLSX |
| `current:sync` | currentHandler | Sync inventory from platform |
| `fav:get-options` | favOptionsHandler | Fetch watchlist options |
| `fav:bulk-action` | favBulkHandler | Bulk watchlist operations |
| `bid:bulk` | bidBulkHandler | Submit bulk bids |
| `trades:get` | tradesHandler | Fetch trade history |
| `products:search` | kreamProductsHandler | Search product DB |
| `backup:run` | backupIpc | Trigger manual backup |
| `backup:list` | backupIpc | List backup bundles |
| `backup:restore` | backupIpc | Restore from bundle |
| `update-start-download` | autoUpdater | Begin update download |
| `update-install` | autoUpdater | Quit and install update |
| `get-latest-release-info` | autoUpdater | Fetch GitHub release notes |
| `key:status` | secretKeyManager | Check license key validity |
| `key:submit` | secretKeyManager | Submit + validate key |

### Push Events (send / on)

Used for streaming progress, status changes, and real-time updates from Main to Renderer.

```
Main Process                     Renderer (hook)
   │                                 │
   │ mainWindow.webContents.send(ch) │
   │────────────────────────────────►│
   │                                 │ ipcRenderer.on(ch, callback)
   │                                 │ → dispatches to Zustand store
   │                                 │ → UI re-renders
```

**Push channels (examples):**

| Channel | Sender | Receiver hook | Payload |
|---|---|---|---|
| `data collection:primary-progress` | prdData collectionManager | useData collectionSync | `{ items, page }` |
| `data collection:detail-progress` | prdData collectionManager | useData collectionSync | `{ item, doneCount, total }` |
| `data collection:retry-state` | prdData collectionManager | useData collectionSync | `RetryState` |
| `data collection:done` | prdData collectionManager | useData collectionSync | `{ cachedItems }` |
| `calc:progress` | calcTaskManager | useCalcSync | `{ progress, current }` |
| `stored:progress` | storedTaskManager | useStoredLoopSync | `{ progress }` |
| `status:net` | statusBroadCaster | useStatusListener | `{ ok, checkedAt }` |
| `status:kream` | statusBroadCaster | useStatusListener | `{ blocked, until }` |
| `update-available` | autoUpdater | useUpdater | `UpdateInfo` |
| `update-downloaded` | autoUpdater | useUpdater | — |
| `update-download-progress` | autoUpdater | useUpdater | `percent` |
| `github-latest-release` | autoUpdater | useUpdater | `{ tagName, body }` |

---

## Hook Pattern

Each domain has a dedicated **IPC sync hook** that registers listeners on mount and cleans up on unmount.

```typescript
// Example: useStatusListener.ts
export const useStatusListener = () => {
  const setNet = useStatusStore((s) => s.setNet);
  const setKream = useStatusStore((s) => s.setKream);

  useEffect(() => {
    const handleNet = (_: any, payload: any) => setNet(payload);
    const handleKream = (_: any, payload: any) => setKream(payload);

    window.electron.ipcRenderer.on('status:net', handleNet);
    window.electron.ipcRenderer.on('status:kream', handleKream);

    return () => {
      window.electron.ipcRenderer.removeListener('status:net', handleNet);
      window.electron.ipcRenderer.removeListener('status:kream', handleKream);
    };
  }, [setNet, setKream]);
};
```

All hooks are registered globally in `_app.tsx`, so they are active across all page navigations.

---

## Preload Bridge Surface

`preload.ts` exposes a minimal, typed interface to the renderer via `contextBridge`:

```typescript
contextBridge.exposeInMainWorld('electron', {
  ipcRenderer: {
    send:           (channel, data) => ipcRenderer.send(channel, data),
    on:             (channel, callback) => ipcRenderer.on(channel, ...),
    removeListener: (channel, callback) => ipcRenderer.removeListener(...),
    invoke:         (channel, ...args) => ipcRenderer.invoke(channel, ...args),
  },
});
```

The renderer accesses this as `window.electron.ipcRenderer.*`. This is the only bridge between the two processes — the renderer has no direct Node.js access.
