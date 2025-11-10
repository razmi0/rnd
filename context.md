
# MFE Dashboard System - Technical Documentation

## Project Overview

This is a research and development project for building a **Dashboard Microfrontend (MFE)** as part of a larger microfrontend architecture. The Dashboard MFE acts as a centralized aggregation point that displays widgets, shortcuts, and bookmarks contributed by other independent MFEs in the system.

**Project Location:** `/Users/thomascuesta/Desktop/Code/MFS/rnd`

**Key Philosophy:** Complete decoupling through event-driven architecture—the Dashboard never imports from or directly depends on other MFEs.

---

## Architecture Overview

### Multi-Product Architecture

The orchestrator supports **multiple aircraft platforms** (A320, A380) with product-specific configurations:

**Per-Product Assets:**

- **Import Maps**: Product-specific MFE registrations (`products/{product}/importmaps/importmap.{env}.json`)
- **Layouts**: Custom UI arrangements (`products/{product}/layout.html`)
- **Build Commands**: Separate pipelines per product
  - A320: `npm run build:a320:prod`
  - A380: `npm run build:a380:prod`

**Dashboard Implication:** The Dashboard MFE must remain product-agnostic and adapt dynamically to different aircraft contexts.

---

### System Components

```text
┌─────────────────────────────────────────────────────┐
│                    ORCHESTRATOR                     │
│  (Port 9000, HTTPS with self-signed certs)          │
│  Multi-product: A320, A380                          │
└────────────────────┬────────────────────────────────┘
                     │
     ┌───────────────┼───────────────────┐
     │               │                   │
┌────▼────┐   ┌─────▼──────┐   ┌───────▼────────┐
│ EVENT   │   │ LOCAL      │   │ URL STATE      │
│ BUS     │   │ STORAGE    │   │ (Shareable)    │
└────┬────┘   └─────┬──────┘   └───────┬────────┘
     │              │                   │
     ├──────────────┴───────────────────┤
     │                                  │
┌────▼────────────────┐    ┌───────────▼──────────┐
│ Existing MFEs:      │    │   DASHBOARD MFE      │
│ • GraphicalContainer│    │   (This project)     │
│ • Notifications     │◄───┤   Aggregates:        │
│ • Permissions       │    │   • Widgets          │
│ • Sidebar/Topbar    │    │   • Shortcuts        │
│ • Module-ESD        │    │   • Bookmarks        │
└─────────────────────┘    └──────────────────────┘
     (Port 9018)                (Port 9018)
```

### Communication Patterns

**Event Bus (Primary):** All MFE-to-MFE communication happens via typed events
**Shared Storage:** Persistence via namespaced localStorage keys
**Runtime API:** MFEs expose functionalities via `window.MFEnamespace.{mfeName}`

---

## Core Concepts

### 1. **Widgets** - Data Display

Widgets are composable UI components that display data from other MFEs.

**Key Features:**

- Dynamic registration at runtime
- Lazy loading (code splitting)
- Sandboxed execution (Shadow DOM)
- User-configurable layout
- Persistent positioning

**Example Registration:**

```javascript
eventBus.emit('dashboard:registerWidget', {
  id: 'graphical-stats',
  title: 'Data Overview',
  category: 'monitoring',
  render: () => import('graphicalContainer/WidgetComponent'),
  metadata: {
    size: 'medium',
    refreshInterval: 30000
  }
});
```

**Storage:** `localStorage["mfe.dashboard.widgets"]`

📄 **Full Documentation:** [`dashboard/widgets.md`](dashboard/widgets.md)

---

### 2. **Shortcuts** - Quick Actions

Shortcuts are action triggers providing instant access to common tasks.

**Types:**

- **Navigation:** Jump to specific routes
- **Action:** Trigger operations (export, refresh)
- **Modal:** Open dialogs/overlays
- **External:** Link to external tools
- **Command:** Broadcast system-wide events

**Example Registration:**

```javascript
eventBus.emit('dashboard:registerShortcut', {
  id: 'export-data',
  label: 'Export Current Data',
  icon: 'download',
  category: 'actions',
  hotkey: 'cmd+e',
  action: {
    type: 'event',
    event: 'graphicalContainer:export',
    payload: { format: 'csv' }
  }
});
```

**Storage:** `localStorage["mfe.dashboard.shortcuts"]`

📄 **Full Documentation:** [`dashboard/shortcuts.md`](dashboard/shortcuts.md)

---

### 3. **Bookmarks** - Saved States

Bookmarks preserve application states (filters, layouts, configurations) for quick restoration.

**Types:**

- **View Bookmarks:** Complete page states
- **Filter Bookmarks:** Filter combinations
- **Layout Bookmarks:** Widget arrangements
- **Query Bookmarks:** Saved queries

**Example Usage:**

```javascript
// Save current state
eventBus.emit('dashboard:bookmark:create', {
  name: 'Q4 Performance View',
  type: 'view',
  visibility: 'team',
  state: {
    route: window.location.pathname,
    filters: getCurrentFilters(),
    widgets: getCurrentWidgetLayout()
  }
});

// Restore bookmark
eventBus.emit('dashboard:bookmark:activate', {
  bookmarkId: 'q4-performance-view'
});
```

**Storage:** `localStorage["mfe.dashboard.bookmarks"]`

📄 **Full Documentation:** [`dashboard/bookmarks.md`](dashboard/bookmarks.md)

---

## Event Replay / Snapshot System

### The Problem

Late-loading MFEs (like Dashboard) miss events emitted before they subscribe, causing race conditions and stale data.

### The Solution: Hybrid Approach

**1. Event Replay Buffer**

- Orchestrator maintains buffer of recent events (last 50, TTL 30s)
- Late subscribers receive replay of relevant events

**2. State Snapshots (`getState()`)**

- Each MFE exposes current state via `getState()` method
- Dashboard queries state on initialization

**3. Regular Subscriptions**

- Continue listening for future events

**Implementation:**

```javascript
// MFE exposes state
window.MFEnamespace.graphicalContainer = {
  getState: () => ({
    metrics: { totalRows: 320, filteredRows: 89 },
    filters: { dateRange: 'last-30-days' }
  })
};

// Dashboard subscribes with replay
eventBus.subscribe('graphicalContainer:metrics', callback, { 
  initializeFromState: 'graphicalContainer',
  replay: true 
});
```

📄 **Full Documentation:** [`comunication.md`](comunication.md)

---

## Hexagonal Architecture

### Folder Structure

```text
dashboard-mfe/
├── src/
│   ├── domain/              # Pure business logic
│   │   ├── entities/        # Widget, Bookmark, Shortcut
│   │   ├── services/        # Domain services
│   │   └── usecases/        # Use cases
│   │
│   ├── ports/               # Interfaces (abstractions)
│   │   ├── eventBus/        # EventBusPort + Adapter
│   │   ├── storage/         # StoragePort + Adapter
│   │   ├── permissions/     # PermissionsPort + Adapter
│   │   └── view/            # ViewPort
│   │
│   ├── infrastructure/      # Concrete implementations
│   │   ├── adapters/        # EventBus, Storage implementations
│   │   ├── config/          # Environment configs
│   │   └── utils/           # Utilities
│   │
│   ├── view/                # UI Layer (React/Vue)
│   │   ├── components/      # UI Components
│   │   ├── hooks/           # React hooks
│   │   ├── pages/           # Page components
│   │   └── styles/          # CSS
│   │
│   └── app/                 # Entry point & DI wiring
│       ├── main.tsx         # Bootstrap
│       └── index.ts         # mount/unmount API
```

**Benefits:**

- Framework agnostic core
- Easy to swap implementations (localStorage → IndexedDB)
- Testable in isolation
- Future-proof

📄 **Full Documentation:** [`dashboard/source.md`](dashboard/source.md)

---

## Event Contracts

### Naming Convention

```
{mfeName}:{module}:{action}
```

Examples:

- `dashboard:widget:add`
- `graphicalContainer:metrics`
- `notifications:unread`
- `permissions:changed`

### Key Events

| Event | Direction | Payload | Description |
|-------|-----------|---------|-------------|
| `dashboard:widget:add` | MFE → Dashboard | `WidgetConfig` | Register widget |
| `dashboard:shortcut:activated` | Dashboard → All | `{ shortcutId, action }` | Shortcut triggered |
| `dashboard:bookmark:activate` | Dashboard → All | `{ bookmarkId }` | Restore bookmark |
| `graphicalContainer:metrics` | GC → Dashboard | `{ totalRows, filtered }` | Data metrics |

---

## Technical Stack

### Current Setup

**Framework:** Single-spa + React 18.2.0  
**Build Tool:** Webpack 5 + Module Federation  
**Language:** TypeScript  
**Testing:** Jest + React Testing Library  
**UI Components:** Material-UI v7 + Airbus Design System  
**State:** Event-driven + localStorage  
**Infrastructure:** AWS CDK (Cloud Development Kit)  
**Deployment:** Multi-stage pipeline (dev → val → prod)

### Local Development Infrastructure

**Orchestrator:**

- **Port:** 9000
- **Protocol:** HTTPS with self-signed certificates
- **Certificates:** Stored in `secret/local.key` and `secret/local.pem`
- **WebSocket URL:** `wss://localhost:9000`

**MFE Modules:**

- **Port:** 9018 (HTTP, webpack-dev-server)
- **Health Check:** Each MFE exposes `/aura/status` endpoint
- **Standalone Mode:** Can run independently without orchestrator

**Shared Libraries (Injected by Orchestrator):**

- `@airbus/1t21-aura-library-notification-manager-core`
- `@airbus/1t21-aura-library-permission-manager-core` (v0.1.19)
- `@airbus/1t21-authentication-manager` (v0.2.3)
- `toastify-js` (toast notifications)

These are available at runtime via `window.MFEnamespace.*`

### Critical Issues Identified

⚠️ **Must Fix:**

**1. Babel Browser Targeting Bug**

- **Evidence:** `babel.config.json` line 121: `"targets": "current node"`
- **Impact:** Generates Node-specific code (CommonJS, Node APIs) instead of browser-compatible ES modules
- **Symptom:** Bundle includes unnecessary polyfills, potential runtime errors
- **Fix:** Change to `"targets": "> 0.25%, not dead"` or similar browserslist query

**2. Webpack Minimization in Dev Mode**

- **Evidence:** `webpack.config.js` line 210-213: `optimization: { minimize: true }`
- **Impact:** Slow build times, difficult debugging (minified code)
- **Fix:** Move minimization to production-only config

**3. Module Federation + single-spa Hybrid Approach**

- **Issue:** Using both Module Federation AND single-spa lifecycle
- **Risk:** Double-loading dependencies, version conflicts
- **Decision Needed:** Choose primary pattern (recommendation: Module Federation)

**4. Port Management Inconsistency**

- **Orchestrator:** Port 9000 (correct)
- **MFE standard:** Port 9018 (correct)
- **npm start --env:** Port 3000 (⚠️ conflicts risk)
- **Fix:** Standardize all dev scripts to use 9018

**5. Deployment Security Risk**

- **Evidence:** All CDK commands use `--require-approval never`
- **Risk:** Infrastructure changes deploy to production without manual review
- **Impact:** Accidental resource deletion, security misconfigurations
- **Fix:** Remove auto-approval for val/prod environments

⚠️ **Should Fix:**

**1. React Version Constraint Too Tight**

- **Current:** `@types/react: >=18.0.28 <=18.2.42`
- **Risk:** Blocks patch updates, incompatible with React 18.3+
- **Fix:** Use caret range: `@types/react: ^18.2.0`

**2. Missing Shared Dependencies in Module Federation**

- **Currently Shared:** react, react-dom, @airbus/components-react, @airbus/icons, @fontsource/inter
- **Missing (causes duplicate bundles):**
  - `@emotion/react` ^11.14.0
  - `@emotion/styled` ^11.14.1
  - `@mui/material` ^7.3.1
  - `@mui/icons-material` ^7.3.1
- **Impact:** Bundle bloat (~400KB duplicated), potential style conflicts
- **Fix:** Add to webpack shared array with `singleton: true, eager: true`

**3. Environment Variable Handling Fragility**

- **Pattern:** `--env goal=$npm_config_env` in package.json
- **Issues:**
  - Only works with npm (breaks with yarn/pnpm)
  - `webpackConfigEnv.goal` pattern is non-standard
  - Easy to misconfigure or forget the `--env=` flag
- **Fix:** Use standard `NODE_ENV` or webpack mode parameter

**4. Environment File Replacement Regex**

- **Pattern:** `NormalModuleReplacementPlugin(/environments\/environment\.ts/)`
- **Risk:** Fragile string matching, could break with path changes
- **Better:** Use webpack aliases or proper environment injection

📄 **Full Analysis:** [`sidebar/analysed.md`](sidebar/analysed.md)

---

## Deployment & Infrastructure

### AWS CDK Pipeline

The project uses **AWS Cloud Development Kit** for infrastructure as code, enabling automated, multi-stage deployments:

**Environment Stages:**

| Stage | Domain | CDK Command |
|-------|--------|-------------|
| **Development** | `https://dev.coreelec.1t21-coreelec.aws.cloud.airbus-v.corp/aura` | `npm run cdk:deploy:dev` |
| **Validation** | `https://val.coreelec.1t21-coreelec.aws.cloud.airbus-v.corp/aura` | `npm run cdk:deploy:val` |
| **Production** | `https://coreelec.1t21-coreelec.aws.cloud.airbus.corp/aura` | `npm run cdk:deploy:prod` |

**Key Scripts:**

```bash
# Synthesize CloudFormation templates
npm run cdk:synth:dev
npm run cdk:diff:dev      # Preview changes

# Deploy (automated, no approval required)
npm run cdk:deploy:dev
npm run cdk:deploy:val
npm run cdk:deploy:prod   # ⚠️ Auto-approval risky!
```

### Module Federation Hosting

Each MFE is bundled and served from environment-specific CDN paths:

```javascript
// Webpack publicPath configuration (from webpack.config.js)
dev:  "https://dev.coreelec.../aura/CoreElec-aura-module-sidebar.js"
val:  "https://val.coreelec.../aura/CoreElec-aura-module-sidebar.js"
prod: "https://coreelec.../aura/CoreElec-aura-module-sidebar.js"
```

The orchestrator dynamically loads MFEs from these URLs at runtime via Webpack Module Federation.

**Environment File Swapping:**

Webpack uses `NormalModuleReplacementPlugin` to swap environment configs based on build target:

- `environment.ts` → Default (local dev)
- `environment.dev.ts` → Development deployment
- `environment.val.ts` → Validation deployment  
- `environment.prod.ts` → Production deployment

### Development Modes

**1. Standalone Mode**

```bash
npm run start:standalone  # Port 9018
# Command: webpack serve --env standalone --port 9018
```

- MFE runs **independently** without orchestrator
- Useful for isolated component development
- No dependencies on other MFEs
- `webpack --env standalone` flag activates this mode
- **Best for:** UI component development, unit testing

**2. Integrated Development Mode**

```bash
npm run start:dev  # Port 9018
# Command: webpack serve --port 9018
```

- Connects to local or remote orchestrator (running on port 9000)
- Full MFE ecosystem available
- Event bus communication active
- Mimics production environment
- **Best for:** Integration testing, full-stack development

**3. Dynamic Environment Mode (⚠️ Not Recommended)**

```bash
npm start --env=development  # Port 3000
# Command: webpack serve --port 3000 --env goal=$npm_config_env
```

- Dynamically sets build environment via `npm_config_env`
- **⚠️ Port conflict risk** - Uses 3000 instead of standard 9018
- **⚠️ npm-only** - Breaks with yarn/pnpm
- **⚠️ Fragile pattern** - Easy to misconfigure or forget `--env=` flag

**Port Allocation Summary:**

| Service | Port | Protocol | Purpose |
|---------|------|----------|---------|
| Orchestrator | 9000 | HTTPS | Main application shell |
| MFE Modules | 9018 | HTTP | Development servers |
| Dynamic mode | 3000 | HTTP | ⚠️ Non-standard, avoid |

**Recommendation:** Standardize on port 9018 for all MFE dev modes. Remove or fix port 3000 usage.

---

## Development Recommendations

### EventBus Best Practices

1. **Standardize Event Contracts** - Use typed schemas in shared package
2. **Namespace Everything** - Avoid collisions with consistent naming
3. **Event Replay** - Implement buffer for late subscribers
4. **Event Introspection** - Build dev tool for debugging
5. **Debounce/Coalesce** - Prevent event storms
6. **Typed Middleware** - Add validation/logging/auth layers
7. **Graceful Degradation** - Handle missing MFEs

### Storage Best Practices

**Namespacing:**

```javascript
localStorage["mfe.dashboard.widgets"]
localStorage["mfe.dashboard.shortcuts"]
localStorage["mfe.dashboard.bookmarks"]
localStorage["mfe.dashboard.layout"]
```

**State Synchronization:**

- Mirror critical state to URL for shareability
- Use structured cloning to prevent mutations
- Version your storage schemas

### Widget Best Practices

1. **Limit Registration** - Max 10-15 widgets to avoid clutter
2. **Lazy Loading** - Always use dynamic imports
3. **Error Boundaries** - Isolate widget failures
4. **Size Guidelines** - Define standard sizes (small/medium/large)
5. **Refresh Strategies** - Implement polling, pauseWhenHidden

### Quality Assurance Pipeline

**Testing:**

```bash
npm test              # Run Jest test suite
npm run watch-tests   # Watch mode for TDD
npm run coverage      # Generate coverage reports
```

**Linting:**

```bash
npm run lint          # Check code quality
npm run lint:fix      # Auto-fix issues
```

**Build Analysis:**

```bash
npm run analyze       # Webpack bundle analysis (identifies bloat)
npm run build:types   # TypeScript type checking
```

**Git Hooks (Husky):**

- Pre-commit: Runs linting automatically
- Enforced on `npm install` via `prepare` script
- Ensures code quality before commits reach repository

**Recommended Workflow:**

1. Develop with `npm run start:standalone`
2. Run `npm run lint:fix` before commits
3. Verify tests pass with `npm test`
4. Check bundle size with `npm run analyze` periodically
5. Build for target environment: `npm run build:dev|val|prod`

---

## Project Files

### Dashboard Documentation

- **[ideas.md](dashboard/ideas.md)** - Dashboard elements overview
- **[source.md](dashboard/source.md)** - Core architecture & design principles
- **[widgets.md](dashboard/widgets.md)** - Composable widget system
- **[shortcuts.md](dashboard/shortcuts.md)** - Quick actions system
- **[bookmarks.md](dashboard/bookmarks.md)** - State preservation system

### Sidebar Documentation

- **[studies.md](sidebar/studies.md)** - Development dependencies & config
- **[analysed.md](sidebar/analysed.md)** - Critical configuration analysis

### Communication

- **[comunication.md](comunication.md)** - Event replay & snapshot patterns

---

## Configuration Deep Dive (from studies.md)

### Module Federation Shared Dependencies

**Current Configuration:**

```javascript
shared: [
  { "react": { singleton: true, eager: true, requiredVersion: '^18' } },
  { "react-dom": { singleton: true, eager: true, requiredVersion: '^18' } },
  { "@airbus/components-react": { singleton: true, eager: true, requiredVersion: '^3' } },
  { "@airbus/icons": { singleton: true, eager: true, requiredVersion: '^3' } },
  { "@fontsource/inter": { singleton: true, eager: true, requiredVersion: '^5' } }
]
```

**Missing Dependencies (Present in package.json but not shared):**

- `@emotion/react` ^11.14.0
- `@emotion/styled` ^11.14.1
- `@mui/material` ^7.3.1
- `@mui/icons-material` ^7.3.1

**Impact:** Each MFE bundles its own copy → 400KB+ duplication per module.

---

### Health Check Middleware Pattern

All MFEs implement a health check endpoint for orchestrator verification:

```javascript
// From webpack.config.js devServer configuration
middlewares.unshift({
  name: 'first-in-array',
  path: `/${PREFIX}/status`,
  middleware: (req, res) => {
    res.json("running");
  }
});
```

**Endpoint:** `http://localhost:9018/aura/status`  
**Response:** `"running"`  
**Usage:** Orchestrator polls this before attempting to load the MFE

---

### TypeScript Configuration (Sidebar MFE)

```json
{
  "extends": "ts-config-single-spa",
  "compilerOptions": {
    "jsx": "react-jsx",
    "declarationDir": "dist"
  },
  "files": ["src/CoreElec-aura-module-sidebar.tsx"]
}
```

**Note:** This is the **Sidebar MFE** configuration, not a global config. Each MFE has its own `tsconfig.json` extending `ts-config-single-spa`.

---

### Orchestrator External Dependencies

The orchestrator injects these libraries into the global scope, making them available to all MFEs:

```javascript
externals: {
  react: 'react@18',
  'react-dom': 'react-dom@18'
}
```

**Available at Runtime:**

- `window.react` → React 18
- `window['react-dom']` → ReactDOM 18
- `window.MFEnamespace.notificationManager` → Toast notifications
- `window.MFEnamespace.permissionManager` → Authorization
- `window.MFEnamespace.authManager` → Authentication

---

### Environment-Specific Build Configuration

**Webpack Environment Replacement Pattern:**

```javascript
// In webpack.config.js
const configEnv = webpackConfigEnv.goal; // 'development', 'validation', 'production'

new webpack.NormalModuleReplacementPlugin(
  /environments\/environment\.ts/,
  envPath // e.g., 'environment.prod.ts'
)
```

**Files:**

- `environment.ts` → Local dev (default)
- `environment.dev.ts` → Dev deployment (<https://dev.coreelec>...)
- `environment.val.ts` → Val deployment (<https://val.coreelec>...)
- `environment.prod.ts` → Prod deployment (<https://coreelec>...)

**Triggered By:**

```bash
npm run build:dev   # Uses environment.dev.ts
npm run build:val   # Uses environment.val.ts
npm run build:prod  # Uses environment.prod.ts
```

---

## Key Takeaways

### Design Principles

✅ **Strict Isolation** - No direct MFE-to-MFE imports  
✅ **Event-Driven** - All communication via EventBus  
✅ **Shared Storage** - Namespaced localStorage contracts  
✅ **Composable Widgets** - Dynamic registration at runtime  
✅ **Hexagonal Architecture** - Framework-agnostic core  
✅ **Graceful Degradation** - Works even with missing MFEs

### What Makes This Work

1. **Runtime API** - `window.MFEnamespace.{mfeName}.getState()`
2. **Event Replay** - Late subscribers don't miss critical events
3. **State Snapshots** - Always fresh data on initialization
4. **Widget Registry** - Dynamic discovery without coupling
5. **localStorage Contracts** - Persistence without dependencies
6. **Infrastructure as Code** - AWS CDK enables repeatable, multi-stage deployments
7. **Standalone Development** - MFEs can be developed independently without orchestrator
8. **Health Check Pattern** - `/aura/status` endpoints enable orchestrator verification
9. **Multi-Product Support** - Product-specific import maps and layouts (A320, A380)
10. **Shared Library Injection** - Orchestrator provides common services (auth, notifications, permissions)

---

## Next Steps

### Priority 1: Fix Critical Configuration Issues

1. **Fix Babel Browser Targeting**
   - Update `babel.config.json` line 121: `"targets": "> 0.25%, not dead"`
   - Test bundle with browser-compatible output
   - Verify polyfills are appropriate

2. **Secure Production Deployment**
   - Remove `--require-approval never` from prod/val CDK commands
   - Add manual approval gates for infrastructure changes
   - Document approval process

3. **Standardize Port Management**
   - Change port 3000 → 9018 in dynamic mode
   - Update all documentation to reflect 9000 (orchestrator) / 9018 (MFE) split
   - Remove fragile `npm_config_env` pattern

4. **Add Missing Module Federation Shared Dependencies**
   - Add @emotion/react, @emotion/styled, @mui/material, @mui/icons-material
   - Test for duplicate bundle elimination
   - Measure bundle size improvement

### Priority 2: Dashboard MFE Implementation

5. **Implement Event Replay** - Add buffer to EventBus adapter
6. **Build Dashboard Shell** - Create minimal React shell with widget grid
7. **Prototype Widget System** - Implement widget registration & rendering
8. **Add State Management** - Implement bookmarks & shortcuts
9. **Product-Agnostic Design** - Ensure dashboard works across A320/A380

### Priority 3: Quality & Testing

10. **Testing Strategy** - Unit tests for domain logic, integration tests for events
11. **Health Check Implementation** - Add `/aura/status` endpoint to dashboard
12. **Bundle Analysis** - Run `npm run analyze` and optimize

---

**Project Status:** Production-grade infrastructure with R&D phase for Dashboard MFE  
**Architecture:** Microfrontend (Single-spa + Module Federation)  
**Pattern:** Event-Driven, Hexagonal Architecture  
**Infrastructure:** AWS CDK with multi-stage deployment pipeline  
**Products:** Multi-tenant support for A320 and A380 platforms  
**Goal:** Decoupled, composable dashboard system in enterprise environment

---

**Documentation Sources:**

- Core architecture and concepts from project exploration
- Configuration analysis from `studies.md` (package.json, webpack, babel, tsconfig)
- Critical issues identified from concrete implementation evidence
- Best practices derived from existing MFE implementations (Sidebar, Module-ESD)
