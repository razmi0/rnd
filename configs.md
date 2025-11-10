# Configuration Analysis

## Architecture Overview

This is a **hybrid micro-frontend architecture** combining:

- **Orchestrator**: Uses single-spa + SystemJS import maps to load MFEs
- **MFEs (Sidebar, ESD, etc.)**: Use Module Federation for internal dependency sharing
- **Shared Dependencies**: Mix of CDN-loaded (orchestrator) and bundled (MFEs)

### How It Works

```text
Orchestrator (single-spa + import maps)
    ↓ loads via SystemJS
[Sidebar MFE] ←--Module Federation--> [ESD MFE] ←--> [Other MFEs]
    ↓ shares                               ↓ shares
  React, MUI, Airbus libs            React, MUI, Airbus libs
```

**Key Pattern**: Orchestrator loads MFEs via import maps, but MFEs share dependencies with each other via Module Federation. This is intentional but requires careful alignment.

---

## Sidebar MFE - package.json

### Good Points

- Modern single-spa micro-frontend setup with TypeScript support
- Solid testing foundation (Jest + React Testing Library)
- Webpack 5 - current generation
- ESLint v9 with new flat config
- Up-to-date tooling overall

### Issues/Concerns

#### 1. Version Inconsistency

- React is pinned at `^18.2.0` but `@types/react` has a tight range `>=18.0.28 <=18.2.42`
- This creates fragility - any React update could break types
- **Fix**: Either pin both or loosen the types constraint

#### 2. Material-UI v7.3.1

- This is quite recent/cutting edge
- Verify stability for production use

#### 3. Babel + TypeScript

- You have both Babel and TypeScript compilers
- Common for single-spa but adds complexity
- Ensure you need both

#### 4. Testing Library Versions

- Using `^6.x` and `^14.x` notation is inconsistent with other dependencies
- **Fix**: Be explicit with versions

#### 5. Missing Tools

- No package manager lockfile enforcement tool (like `@npm/package-json-lint`) to prevent drift

#### 6. Airbus Components

- `@airbus/components-react` and `@airbus/icons` - are these internal?
- Ensure you have reliable access to these registries

#### 7. Minor Issue

- Extra space after `@babel/preset-react` version on line 11

---

## tsconfig.json

### Critical Problem

#### File vs Include Mismatch

- Line 70-71: `"files": ["src/CoreElec-aura-module-sidebar.tsx"]` - This explicitly includes ONE file
- Line 73-74: `"include": ["src/**/*"]` - This includes everything

The `files` array overrides typical behavior. Either remove that specific file entry or understand it's limiting what gets type-checked unless `include` supplements it (which it does, but it's confusing).

### Minor Issue

- `"declarationDir": "dist"` without `"declaration": true` - this does nothing

---

## babel.config.json

### Major Problem - Wrong Target

#### Browser vs Node Target

```json
"targets": "current node"
```

You're building for **browsers** (single-spa micro-frontend), not Node.js. This should be:

```json
"targets": "> 0.5%, last 2 versions, not dead"
```

or define specific browser targets. This is likely causing unnecessary polyfills or missing ones.

### Redundancy

- Test environment (lines 110-121) is redundant - it's identical to the main config
- **Fix**: Remove the redundant test env or make it actually different

### Architecture Note

- Both configs use automatic JSX runtime (good, consistent)

---

## Sidebar MFE - webpack.config.js

### Critical Issues

#### 1. React Loading Conflict (CRITICAL - Will Break Runtime)

**The Problem:**

Orchestrator import map loads React from CDN:

```javascript
// Orchestrator webpack.config.js
'react@18': 'https://unpkg.com/react@18/umd/react.production.min.js'
```

But Sidebar MFE expects different naming:

```javascript
// Sidebar webpack - Module Federation
shared: [
  { "react": { singleton: true, eager: true, requiredVersion: '^18' } }
]
```

**Result:** Import name mismatch (`react@18` vs `react`) causes **multiple React instances** → broken hooks, crashes, state loss.

**Fix Required:** Either:

- Orchestrator uses `"react": "https://..."` (remove @18 suffix)
- OR MFEs external config uses `externals: { react: 'react@18' }`

#### 2. Module Federation + single-spa Hybrid (Intentional but Misaligned)

The architecture uses **both**:

- Line 135: `webpack-config-single-spa-ts` - exports for SystemJS (orchestrator consumption)
- Lines 206-218: `ModuleFederationPlugin` - shares deps between MFEs

This is **intentional** for the hybrid architecture but creates dual export formats. The issue isn't the pattern itself but the **misaligned naming conventions** (see issue #1).

**Current State:** Partially working but fragile
**Recommendation:** Keep both but fix import/export naming alignment

#### 3. Missing Critical Shared Dependencies

Orchestrator provides these as dependencies:

```javascript
"@airbus/1t21-authentication-manager": "0.2.3"
"@airbus/1t21-aura-library-permission-manager-core": "0.1.19"
"@airbus/1t21-aura-library-notification-manager-core": "^0.0.0"
```

But Sidebar's Module Federation **doesn't share them** (lines 242-248). Result:

- Each MFE bundles its own copy
- **Authentication state isn't shared** between MFEs
- Permissions cache duplicated
- Notification queues broken

**Fix Required:**

```javascript
shared: [
  { "react": { singleton: true, eager: true, requiredVersion: '^18' } },
  { "react-dom": { singleton: true, eager: true, requiredVersion: '^18' } },
  { "@airbus/1t21-authentication-manager": { singleton: true, eager: true } },
  { "@airbus/1t21-aura-library-permission-manager-core": { singleton: true, eager: true } },
  { "@airbus/1t21-aura-library-notification-manager-core": { singleton: true, eager: true } },
  { "@airbus/components-react": { singleton: true, eager: true, requiredVersion: '^3' } },
  { "@airbus/icons": { singleton: true, eager: true, requiredVersion: '^3' } },
  { "@fontsource/inter": { singleton: true, eager: true, requiredVersion: '^5' } },
  // Add these missing ones:
  { "@emotion/react": { singleton: true, requiredVersion: '^11' } },
  { "@emotion/styled": { singleton: true, requiredVersion: '^11' } },
  { "@mui/material": { singleton: true, requiredVersion: '^7' } },
  { "@mui/icons-material": { singleton: true, requiredVersion: '^7' } }
]
```

#### 4. Always Minimizing in Dev

```javascript
optimization: {
  minimize: true,
  minimizer: [new TerserPlugin()],
}
```

This runs even in development mode (no env check). **Your dev builds are slow and harder to debug.** Should be:

```javascript
minimize: argv.mode === 'production' || webpackConfigEnv.goal === 'production'
```

#### 5. Missing Error Handling

- No `default` fallback after the switch - if `configEnv` is undefined/null, `currentPath` is empty string and builds break silently
- Lines 149-151: Empty string initialization is pointless

### Major Concerns

#### 6. Shared Dependencies Strategy

- `eager: true` on everything means all shared deps load immediately, defeating code-splitting benefits
- Missing `@emotion/react`, `@emotion/styled`, `@mui/material` - each MFE bundles ~500KB+ of MUI separately
- `requiredVersion` is too loose (`^18`, `^3`) - version mismatches between micro-frontends will cause runtime errors

#### 7. Environment Replacement Fragility

```javascript
new webpack.NormalModuleReplacementPlugin(
  /environments\/environment\.ts/,
  envPath
)
```

This regex replace is brittle. If import paths vary slightly, it fails silently. Use webpack aliases instead.

### Minor Issues

- Line 128: ESLint disabled for entire file just for indent? Fix the indent.
- Line 192: Returning JSON string `"running"` instead of object - inconsistent with JSON response type

---

## Orchestrator - webpack.config.js Analysis

### What It Does Right

✅ **Product-based multi-tenancy** - Clean `PRODUCT=a320/a380` env var strategy
✅ **Dynamic import maps** - Loads product-specific layouts and import maps from `products/{PRODUCT}/`
✅ **Environment-specific builds** - Clear separation between local/dev/val/prod
✅ **Static asset organization** - `output.filename: 'static/root-config.js'` keeps assets organized
✅ **CDK deployment scripts** - Consistent infrastructure setup

### Critical Orchestrator Issues

#### 1. React Import Naming Mismatch

```javascript
// Lines 394-396
'react@18': 'https://unpkg.com/react@18/umd/react.production.min.js',
'react-dom@18': 'https://unpkg.com/react-dom@18/umd/react-dom.production.min.js',
```

**Problem:** Orchestrator uses `react@18` but MFEs import `react`. This mismatch breaks Module Federation sharing.

**Fix:** Change to:

```javascript
'react': 'https://unpkg.com/react@18/umd/react.production.min.js',
'react-dom': 'https://unpkg.com/react-dom@18/umd/react-dom.production.min.js',
```

#### 2. Externals Mismatch

```javascript
// Lines 454-457
externals: {
  react: 'react@18',
  'react-dom': 'react-dom@18',
}
```

This tells orchestrator to expect `react@18` but it should be `react` to match MFEs.

**Fix:** Either remove these externals (orchestrator doesn't need them) or change to:

```javascript
externals: {
  react: 'react',
  'react-dom': 'react-dom',
}
```

#### 3. Missing Shared Libraries in Import Map

Orchestrator loads these as dependencies but doesn't expose them in import map:

- `@airbus/1t21-authentication-manager`
- `@airbus/1t21-aura-library-permission-manager-core`
- `@airbus/1t21-aura-library-notification-manager-core`

MFEs need access to these as singletons. They should be in the import map OR MFEs must share them via Module Federation (see Sidebar issue #3).

### What's Missing

- No `@mui/material`, `@emotion/*` in import map - each MFE bundles separately
- No error handling for missing `PRODUCT` or `ENV` values
- Local proxy setup only proxies to dev environment (hardcoded URLs)

---

## Orchestrator - package.json Analysis

### Good Points

✅ Well-organized product-specific scripts (`start:a320:local`, `build:a380:prod`, etc.)
✅ Consistent CDK deployment scripts
✅ Uses `cross-env` for cross-platform compatibility

### Concerns

#### Version Dependencies

- `single-spa: ^6.0.3` and `single-spa-layout: ^3.0.0` - check compatibility (layout v3 requires spa v5.9+, you have v6, good)
- Airbus libraries have inconsistent versioning:
  - `notification-manager-core: ^0.0.0` - unstable version
  - `permission-manager-core: 0.1.19` - more stable
  - `authentication-manager: 0.2.3` - most stable

**Recommendation:** Pin Airbus library versions explicitly, don't use `^0.0.0`.

---

## Unified Architecture Summary

### The Hybrid Pattern Explained

```text
┌─────────────────────────────────────────────┐
│  Orchestrator (single-spa root config)     │
│  - Loads MFEs via SystemJS import maps     │
│  - Manages routing (single-spa-layout)     │
│  - Handles auth/permissions at root        │
└──────────────┬──────────────────────────────┘
               │ (SystemJS loads)
    ┌──────────┴──────────┬──────────────┐
    │                     │              │
┌───▼────────┐     ┌──────▼──────┐  ┌───▼────────┐
│  Sidebar   │────▶│ Module ESD  │  │  Other     │
│    MFE     │     │    MFE      │  │   MFEs     │
└────────────┘     └─────────────┘  └────────────┘
     ↑                    ↑                ↑
     └────────────────────┴────────────────┘
       (Module Federation shares deps)
```

**Why This Pattern?**

- Orchestrator uses single-spa for **routing and MFE lifecycle**
- MFEs use Module Federation for **dependency sharing between themselves**
- Both systems work together BUT require aligned naming conventions

**Current State:** Architecture is sound, but implementation has naming mismatches and missing shared dependencies.

---

## Action Items by Priority

### 🔴 CRITICAL (Will Break Production)

1. **Fix React import naming** across orchestrator and all MFEs
   - Orchestrator: Change `react@18` → `react` in import map
   - OR add `externals: { react: 'react@18' }` to all MFE webpack configs
   - **Impact:** Multiple React instances = broken hooks, crashes

2. **Add Airbus libraries to MFE Module Federation shared**
   - Authentication, permissions, notifications must be singletons
   - **Impact:** Broken auth state, permission checks fail randomly

3. **Fix Babel targets** - Change from `"current node"` to browser targets
   - **Impact:** Wrong polyfills, potential runtime errors in older browsers

### 🟡 MAJOR (Performance & Stability)

1. **Add MUI/Emotion to Module Federation shared**
   - **Impact:** Each MFE bundles 500KB+ separately (slow load, memory waste)

2. **Disable minimize in dev mode**
   - **Impact:** Slow dev builds, debugging harder

3. **Tighten version constraints**
   - React type definitions
   - Airbus library versions (especially `^0.0.0`)
   - **Impact:** Runtime version conflicts

4. **Environment replacement fragility**
   - Replace `NormalModuleReplacementPlugin` with webpack aliases
   - **Impact:** Silent failures when import paths vary

### 🟢 MINOR (Quality of Life)

1. Remove redundant Babel test env
2. Fix TypeScript config clarity (files vs include)
3. Fix ESLint disable comment
4. Add error handling for missing env vars
5. Use explicit testing library versions

---

## Recommended Next Steps

1. **Document the architecture** - Create diagram showing orchestrator → MFEs → shared deps
2. **Create a dependency matrix** - Which libs are shared where and how
3. **Standardize naming conventions** - Document whether to use `react` or `react@18` everywhere
4. **Set up shared config files** - babel.config.json and tsconfig.json as actual shared files
5. **Add integration tests** - Test that shared dependencies actually share (one React instance, one auth manager, etc.)
