# Dashboard MFE Architecture Design

## Project Context

A micro-frontend (MFE) project that unifies multiple applications into one. The system includes:

- **Orchestrator**: Entry point module that bootstraps MFEs
- **Event Bus**: Communication layer for MFE interactions
- **Communication Library**: Shared library for MFE communication
- **Logger Library**: Shared logging functionality
- **Existing MFEs**: NotificationManager, GraphicalContainer (table with filters/sorting), Permissions
- **Modules**: module-sidebar, module-topbar, module-esd
- **State Management**: States stored in localStorage, URL, or in-memory
- **Runtime API**: MFEs can attach singletons to `window.MFEnamespace` to expose functionalities

**Dashboard MFE Requirements:**

- Display shortcuts, bookmarks, filters, monitoring, and statistics
- Must be completely decoupled from other MFEs
- Can use events and localStorage for communication
- Should not be coupled with GraphicalContainer or other MFEs

---

## Core Design Principles

### Strict Isolation

- Dashboard runs as its own MFE — no shared imports except from stable, versioned shared packages
- No direct DOM or JS access to other MFEs

### Event-Driven Interactions

All communication via the event bus.

**Dashboard listens for:**

- `mfe:graphicalContainer:state:update`
- `mfe:bookmarks:changed`
- `mfe:monitoring:data`

**Dashboard emits:**

- `dashboard:shortcut:activated`
- `dashboard:filter:applied`

### Shared Storage Contracts

Use localStorage with namespaced key pattern:

- `mfe.dashboard.bookmarks`
- `mfe.dashboard.layout`
- `mfe.dashboard.metrics`

Optionally mirror state to URL (query params) for shareability.

### Dashboard Runtime Contract

Expose itself in `window.MFEnamespace.dashboard`:

```javascript
window.MFEnamespace.dashboard = {
  reload: () => eventBus.emit('dashboard:reload'),
  addWidget: (widget) => eventBus.emit('dashboard:widget:add', widget)
}
```

### Composable Widget System

Dashboard is essentially a grid of widgets. Widgets are declarative JSON configs published via events:

```javascript
eventBus.emit('dashboard:registerWidget', {
  id: 'graphical-stats',
  title: 'Data Overview',
  render: () => import('graphicalContainer/WidgetComponent')
});
```

Dashboard decides layout, persistence, visibility — not the other MFEs.

### Visual Composition

Dashboard shell structure:

- **Top**: filters/search/global actions
- **Main**: grid of widgets
- **Optional sidebar**: bookmarks or quick stats
- Each widget runs sandboxed (via dynamic import or shadow DOM)

---

## Example Interaction Flow

```javascript
// GraphicalContainer loads and emits metrics
eventBus.emit('graphicalContainer:metrics', { totalRows: 320, filtersActive: 2 });

// Dashboard listens
eventBus.on('graphicalContainer:metrics', (data) => {
  updateWidget('graphicalContainer', data);
});
```

Dashboard updates its metrics card — no direct dependency, no import.

---

## Implementation Steps

1. **Define shared contracts**
   - Create `@shared/eventContracts` package with typed event schemas
2. **Implement global event bus**
   - Support typed events and wildcard subscriptions
3. **Prototype Dashboard MFE**
   - Minimal React/Vue/Svelte app
   - Listen to `dashboard:widget:*` events
   - Maintain widget registry + grid layout
4. **Allow dynamic widget contribution**
   - Other MFEs emit widget configs on load
   - Dashboard persists them in localStorage
5. **Iterate on UI**
   - Build composable widget cards
   - Add filters/search later

---

## System Architecture

```text
                        ┌─────────────────────────────┐
                        │         ORCHESTRATOR        │
                        │  (Entry Point / Shell App)  │
                        └────────────┬────────────────┘
                                     │
                ┌────────────────────┼────────────────────┐
                │                    │                    │
        ┌───────▼───────┐    ┌───────▼────────┐   ┌───────▼────────┐
        │ EVENT BUS      │    │ LOCALSTORAGE   │   │ URL STATE       │
        │ (Pub/Sub API)  │    │ (Persistence)  │   │ (Shareable)     │
        └───────┬────────┘    └───────┬────────┘   └───────┬────────┘
                │                    │                    │
 ┌──────────────┼────────────────────┼────────────────────┼──────────────┐
 │              │                    │                    │              │
 │     ┌────────▼────────┐    ┌───────▼────────┐   ┌───────▼────────┐   │
 │     │ Graphical       │    │ Notification   │   │ Permissions     │   │
 │     │ Container MFE   │    │ Manager MFE    │   │ MFE             │   │
 │     │ (Table, Filters)│    │ (Toast, Alerts)│   │ (AuthZ Rules)   │   │
 │     └────────┬────────┘    └───────┬────────┘   └───────┬────────┘   │
 │              │                    │                    │              │
 │              └────────────┬───────┴────────────┬────────┘              │
 │                           │                    │                       │
 │                    ┌───────▼────────┐           │                       │
 │                    │  Dashboard MFE │◄──────────┘                       │
 │                    │  (Decoupled)   │                                   │
 │                    │  Displays:     │                                   │
 │                    │   - Shortcuts  │                                   │
 │                    │   - Bookmarks  │                                   │
 │                    │   - Filters    │                                   │
 │                    │   - Stats      │                                   │
 │                    └────────────────┘                                   │
 └────────────────────────────────────────────────────────────────────────┘
```

### Communication Flow

1. **Event Bus (Global Pub/Sub)**
   - Single source of truth for MFE communication
2. **Shared Storage**
   - Persistence & synchronization layer
   - `localStorage["mfe.dashboard.widgets"]` → stores user layout & bookmarks
   - `localStorage["mfe.graphical.filters"]` → stores current filters
3. **Runtime Exports**
   - Each MFE attaches to common namespace: `window.MFEnamespace`

### Event Contract Examples

| Event Name | Direction | Payload | Description |
|------------|-----------|--------|-------------|
| `dashboard:widget:add` | other → dashboard | `{ id, title, render }` | Register a new widget |
| `dashboard:widget:remove` | other → dashboard | `{ id }` | Remove widget |
| `graphicalContainer:metrics` | GC → dashboard | `{ totalRows, filtersActive }` | Update stats widget |
| `dashboard:shortcut:activated` | dashboard → all | `{ shortcutId }` | Notify shortcut activation |
| `notifications:new` | any → NotificationMFE | `{ message, level }` | Show toast |

### Responsibilities per Layer

| Component | Responsibility |
|-----------|---------------|
| Orchestrator | Mounts MFEs, provides event bus and storage API |
| Event Bus | Decoupled communication between MFEs |
| GraphicalContainer MFE | Displays tabular data, filters, emits metrics |
| Notification MFE | Displays toasts/alerts triggered by events |
| Permissions MFE | Manages access rights & roles |
| Dashboard MFE | Aggregates widgets, displays bookmarks/stats, persists layout |

### Dashboard Internal Structure

```text
Dashboard MFE
│
├── Event Listener Layer
│    ├─ Subscribes to MFE events
│    └─ Emits dashboard events
│
├── State Manager
│    ├─ Persists to localStorage
│    ├─ Hydrates on load
│    └─ Syncs with event bus
│
├── Widget Registry
│    ├─ Receives widget configs
│    ├─ Manages dynamic imports
│    └─ Renders widgets in grid
│
└── UI Layer (React/Vue)
     ├─ Grid Layout
     ├─ Bookmarks Sidebar
     └─ Filters/Search
```

---

## Hexagonal Architecture Folder Structure

```text
dashboard-mfe/
├── src/
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── Widget.ts
│   │   │   ├── DashboardLayout.ts
│   │   │   └── Bookmark.ts
│   │   ├── services/
│   │   │   ├── DashboardService.ts
│   │   │   ├── WidgetRegistryService.ts
│   │   │   └── MetricsAggregator.ts
│   │   └── usecases/
│   │       ├── RegisterWidget.ts
│   │       ├── RemoveWidget.ts
│   │       ├── UpdateLayout.ts
│   │       └── LoadDashboard.ts
│   │
│   ├── ports/
│   │   ├── eventBus/
│   │   │   ├── EventBusPort.ts        // interface
│   │   │   └── EventBusAdapter.ts     // concrete adapter
│   │   ├── storage/
│   │   │   ├── StoragePort.ts
│   │   │   └── LocalStorageAdapter.ts
│   │   ├── permissions/
│   │   │   ├── PermissionsPort.ts
│   │   │   └── PermissionsAdapter.ts
│   │   └── view/
│   │       └── ViewPort.ts            // interface exposing rendering contract
│   │
│   ├── infrastructure/
│   │   ├── adapters/
│   │   │   ├── EventBusImpl.ts        // concrete event bus impl
│   │   │   ├── LocalStorageImpl.ts
│   │   │   ├── UrlStorageImpl.ts
│   │   │   └── PermissionsImpl.ts
│   │   ├── config/
│   │   │   ├── environment.ts
│   │   │   └── mfe-registration.ts
│   │   └── utils/
│   │       ├── debounce.ts
│   │       ├── logger.ts
│   │       └── deepMerge.ts
│   │
│   ├── view/
│   │   ├── components/
│   │   │   ├── DashboardGrid.tsx
│   │   │   ├── WidgetCard.tsx
│   │   │   ├── FiltersPanel.tsx
│   │   │   └── BookmarkList.tsx
│   │   ├── hooks/
│   │   │   ├── useDashboard.ts
│   │   │   └── useEventBus.ts
│   │   ├── pages/
│   │   │   └── DashboardPage.tsx
│   │   └── styles/
│   │       └── dashboard.css
│   │
│   ├── app/
│   │   ├── main.tsx                    // MFE bootstrap
│   │   └── index.ts                    // Exports mount/unmount API
│   │
│   └── shared/
│       ├── types/
│       │   └── index.ts
│       └── constants/
│           └── events.ts
│
├── public/
│   └── index.html
│
├── package.json
├── tsconfig.json
└── vite.config.ts / webpack.config.js
```

### Hexagonal Architecture Layer Responsibilities

| Layer | Role |
|-------|------|
| `domain/` | Pure business logic — no framework, no DOM. Defines entities, services, and use cases |
| `ports/` | Define interfaces for the outside world — event bus, storage, permissions, view — without knowing implementation details |
| `infrastructure/` | Provide adapters implementing the ports. Handles I/O, side effects, integration |
| `view/` | UI rendering (React, Vue, etc.). Uses the domain and ports indirectly. Should never contain business logic |
| `app/` | Entry point, DI wiring (inject adapters into use cases), and MFE registration (expose mount/unmount) |
| `shared/` | Constants, types, or small utilities that are not business logic but used across layers |

### Benefits

- Swap storage mechanisms (localStorage → IndexedDB) or event bus implementation with zero impact on domain
- Tests can mock ports to verify use cases in isolation
- Dashboard remains framework-agnostic, testable, and evolution-proof

---

## EventBus Improvements & Best Practices

### 1. Standardize Event Contracts

Define typed, versioned contracts shared by all MFEs:

```typescript
export interface DashboardEvents {
  'dashboard:widget:add': WidgetConfig;
  'dashboard:widget:update': Partial<WidgetConfig>;
  'dashboard:widget:remove': { id: string };
}
```

Add validation layer to reject malformed payloads.

**Result**: Predictable cross-MFE communication and safer evolution.

### 2. Namespace Everything

Use consistent naming convention:

- `dashboard:*`
- `graphicalContainer:*`
- `permissions:*`
- `notifications:*`

Optionally scope events per app instance: `dashboard@main:widget:add`

**Result**: Avoids conflicts when multiple MFEs emit similar events.

### 3. Event Replay / Snapshot System

If Dashboard loads after other MFEs, it may miss events.

**Solutions:**

- Maintain short-term replay buffer in orchestrator or shared singleton
- Expose `getState()` methods in MFEs (Dashboard can request current data after subscribing)

**Result**: Dashboard always in sync, even when loaded asynchronously.

### 4. Event Introspection

Build lightweight developer tool showing:

```log
[mfe:eventbus] emitted → graphicalContainer:metrics { totalRows: 150 }
```

**Result**: Observability and traceability across MFEs.

### 5. Event Debouncing and Coalescing

Prevent flooding from rapid UI changes.

**Solutions:**

- Add debouncer or throttle on EventBus adapter
- Aggregate bursts into single summarized event: `graphicalContainer:metrics:batch`

**Result**: Prevents UI lag and unnecessary re-renders.

### 6. Scoped Listeners

Allow Dashboard to subscribe only to relevant event namespaces (e.g., `graphicalContainer:*`).

**Result**: Reduces noise and prevents unrelated MFEs from interfering.

### 7. Typed Event Middleware Layer

Middleware can:

- Log, transform, or veto certain events
- Validate permissions before acting
- Enforce rate limits or deduplication

```javascript
eventBus.use((event, payload, next) => {
  if (event.startsWith('admin:') && !user.isAdmin) return;
  next();
});
```

**Result**: Centralized control, easier to evolve behavior globally.

### 8. Graceful Degradation

When dependent MFE isn't loaded or emits invalid data:

- Dashboard should fallback to cached or placeholder widgets
- Display non-blocking "data unavailable" card

**Result**: Resilience and better UX.

---

## Edge Cases & Limitations

### 1. Race Conditions

**Problem**: MFEs can emit events before Dashboard subscribes → missed data.

**Solution**: Use replays or initialization handshakes (`dashboard:ready` / `graphicalContainer:state:request`).

### 2. Event Storms

**Problem**: Rapid UI changes cause hundreds of emits → performance issues.

**Solution**: Debounce, batch, or summarize events.

### 3. Tight Coupling via Implicit Expectations

**Problem**: MFEs rely too much on specific event sequences or payload shapes.

**Solution**: Keep events semantic, not procedural (`user:loggedIn` not `step3:done`).

### 4. No Backpressure or Delivery Guarantees

**Problem**: EventBus (usually synchronous, in-memory) provides no ordering, retries, or persistence.

**Solution**: Accept this tradeoff, or wrap with small queue abstraction if needed.

### 5. Cross-MFE Version Drift

**Problem**: Over time, MFEs may emit/expect slightly different event payloads.

**Solution**: Add schema version fields in payloads or event names (`dashboard:widget:add:v2`).

### 6. Visibility / Debug Difficulty

**Problem**: When something breaks, there's no call stack linking MFEs.

**Solution**: Add shared EventBusInspector (dev-only overlay).

### 7. Security / Permissions Edge Cases

**Problem**: Malicious or buggy MFEs could emit privileged events.

**Solution**: Permissions MFE or orchestrator should filter or sandbox emitters.

### 8. Global Namespace Pollution

**Problem**: If everything attaches to `window.MFEnamespace`, naming collisions or accidental overwrites can occur.

**Solution**: Use unique IDs per MFE and freeze the root namespace.

---

## Summary Table

| Improvement | Solves | Strategy |
|-------------|--------|----------|
| Standardized event contracts | Schema drift | Shared types package |
| Namespaced events | Collisions | Convention (`mfe:module:event`) |
| Replay buffer | Late subscribers | Event snapshot |
| Event introspection | Debug difficulty | Devtools overlay |
| Debounce / coalesce | Performance | Event middleware |
| Scoped listeners | Noise reduction | Subscription filters |
| Middleware layer | Global governance | Auth/logging/validation |
| Graceful degradation | Missing MFEs | Cached fallbacks |

---

## Theoretical Architecture with Shared Libraries

```text
                ┌──────────────────────────────────────────────────────────────┐
                │                       ORCHESTRATOR                          │
                │──────────────────────────────────────────────────────────────│
                │ - Bootstraps MFEs                                            │
                │ - Provides shared runtime services                           │
                │ - Loads libraries (event, storage, permissions, registry)    │
                │ - Defines system contracts and versioning policy             │
                └──────────────┬─────────────────────────────┬──────────────────┘
                               │                             │
       ┌───────────────────────┘                             └────────────────────────┐
       ▼                                                                            ▼
┌──────────────────────┐                                               ┌──────────────────────┐
│  SHARED LIBRARIES    │                                               │  DOMAIN MFEs         │
│──────────────────────│                                               │──────────────────────│
│  lib-eventbus        │   <-- core bus abstraction, typed, replayable │  GraphicalContainer  │
│  lib-storage         │   <-- local/url/state adapter contracts       │  Notifications       │
│  lib-permissions     │   <-- access rules, propagation policies      │  Permissions         │
│  lib-contracts       │   <-- event schemas + types                   │  Others...           │
│  lib-registry        │   <-- runtime widget/feature registry         │                      │
└─────────┬────────────┘                                               └─────────┬────────────┘
          │                                                                      │
          │                                                                      │
          ▼                                                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  DASHBOARD MFE (Consumer)                                   │
│─────────────────────────────────────────────────────────────────────────────────────────────│
│ - Subscribes to bus via lib-eventbus                                                         │
│ - Uses registry to discover widgets                                                          │
│ - Uses storage for layout/bookmarks                                                          │
│ - Aggregates domain metrics                                                                  │
│ - Emits only semantic events                                                                 │
│ - No direct imports from other MFEs                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

### Summary

- **EventBus** → central spine; abstracted, versioned, minimal API
- **Storage/Permissions/Registry** → stable libs managed by orchestrator; never directly coded in MFEs
- **Orchestrator** → governance + DI container; defines truth, not behavior
- **MFEs** → pure feature modules; emit semantic, typed events only
- **Dashboard** → high-level consumer; no orchestration, only listening and aggregation

**Rule**: MFEs depend only on shared libs, never on each other or the orchestrator internals.
