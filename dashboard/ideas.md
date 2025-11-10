# Ideas for the Dashboard MFE

## Architecture & Design

- Domain
  - ?
Ports
  - ?
Adapters
  - ?
View
  - ?

## Dashboard Elements

- for the moment
  - bookmarks
- in the future
  - shortcuts, monitoring, statistics, widgets, search, notifs

## Technical discussion

### Current Problem

- Dashboard features (bookmarks, shortcuts, etc.) need to be accessible from other MFEs
- com-lib event system is immature and fragile
- Dashboard MFE won't always be mounted when other MFEs need these features
- No shared storage-lib exists for data persistence

### ❌ Bad Solution: Dashboard as Persistence Layer

Making dashboard expose singleton methods for persistence is an anti-pattern:

- **Wrong responsibility** - Dashboard is UI/view, not infrastructure
- **Tight coupling** - Creates critical dependency across all MFEs (defeats MFE independence)
- **Mounting paradox** - If dashboard isn't mounted, singleton methods won't work anyway
- **Single point of failure** - Dashboard failure breaks persistence everywhere

### ✅ Better Solutions

Option 1: Create storage-lib (Recommended for client-side data)

- Build a lightweight shared library wrapping localStorage/sessionStorage/IndexedDB
- Each MFE imports it directly - no mounting dependencies
- Simple, independent, testable
- Good for: bookmarks, UI preferences, local shortcuts

Option 2: Backend Service (Recommended for shared/important data)

- Persist server-side via REST/GraphQL API
- Solves persistence, sync, and cross-MFE sharing properly
- No dependency on any MFE being mounted
- Good for: user data, sync across devices, shared state

Option 3: Fix com-lib (Only if events are architecturally correct)

- Invest in maturing com-lib if cross-MFE events are the right pattern
- But events aren't great for data persistence anyway

Recommendation: Use storage-lib for client-side data + backend API for anything that needs to survive beyond the session or be shared across devices.
