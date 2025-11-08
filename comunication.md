# Event Replay / Snapshot System

The **Event Replay / Snapshot System** is one of the most critical patterns for MFE architectures. It solves the fundamental problem of ensuring that late-loading MFEs (like the Dashboard) can synchronize with MFEs that have already loaded and emitted their state.

## The Problem: Race Conditions in Async MFE Loading

### Timeline of the Issue

- Time 0ms:  Orchestrator starts
- Time 50ms: GraphicalContainer MFE loads
- Time 75ms: GraphicalContainer emits: 'graphicalContainer:metrics' → { totalRows: 320 }
- Time 100ms: Notifications MFE loads
- Time 125ms: Notifications emits: 'notifications:unread' → { count: 5 }
- Time 200ms: Dashboard MFE loads
- Time 210ms: Dashboard subscribes to events
- Time 220ms: ⚠️ Dashboard is now listening, but missed all previous events!

**Result**: Dashboard shows empty/stale data even though other MFEs already emitted their state.

This is a classic **race condition** in distributed systems. The Dashboard MFE needs to:

1. Know what state other MFEs are in
2. Display widgets that depend on data from other MFEs
3. Stay synchronized even when loaded asynchronously

## Solution 1: Event Replay Buffer

### Concept

The Orchestrator or EventBus maintains a **short-term buffer** of recent events, then "replays" them to late subscribers.

### Implementation

```typescript
// EventBus with replay buffer
class EventBusWithReplay {
  private listeners = new Map<string, Set<Function>>();
  private replayBuffer: Array<{ event: string; payload: any; timestamp: number }> = [];
  private readonly BUFFER_SIZE = 50; // Keep last 50 events
  private readonly BUFFER_TTL = 30000; // 30 seconds

  emit(event: string, payload: any) {
    // Add to replay buffer
    this.replayBuffer.push({
      event,
      payload: structuredClone(payload), // Deep clone to prevent mutations
      timestamp: Date.now()
    });

    // Trim buffer
    if (this.replayBuffer.length > this.BUFFER_SIZE) {
      this.replayBuffer.shift();
    }

    // Clean expired events
    const now = Date.now();
    this.replayBuffer = this.replayBuffer.filter(
      e => (now - e.timestamp) < this.BUFFER_TTL
    );

    // Emit to current listeners
    const callbacks = this.listeners.get(event) || new Set();
    callbacks.forEach(cb => cb(payload));
  }

  on(event: string, callback: Function, options?: { replay?: boolean }) {
    // Add listener
    if (!this.listeners.has(event)) {
      this.listeners.set(event, new Set());
    }
    this.listeners.get(event)!.add(callback);

    // Replay past events if requested
    if (options?.replay) {
      this.replayBuffer
        .filter(e => this.matchesPattern(e.event, event))
        .forEach(e => callback(e.payload));
    }
  }

  private matchesPattern(emittedEvent: string, pattern: string): boolean {
    // Support wildcards: 'graphicalContainer:*'
    if (pattern.endsWith('*')) {
      const prefix = pattern.slice(0, -1);
      return emittedEvent.startsWith(prefix);
    }
    return emittedEvent === pattern;
  }
}
```

### Usage in Dashboard

```javascript
// Dashboard MFE loads late
export function bootstrap() {
  // Subscribe with replay option
  eventBus.on('graphicalContainer:metrics', (data) => {
    updateMetricsWidget(data);
  }, { replay: true }); // ← Gets past events!

  eventBus.on('notifications:unread', (data) => {
    updateNotificationBadge(data);
  }, { replay: true });
}
```

### Pros & Cons

**Pros:**
✅ Simple to implement  
✅ Works automatically for all events  
✅ No changes needed in individual MFEs  
✅ Good for high-frequency events (metrics, updates)

**Cons:**
❌ Memory overhead (storing event history)  
❌ Potential stale data if buffer expires  
❌ May replay events that are no longer relevant  
❌ Buffer size/TTL configuration needed

---

## Solution 2: State Snapshot (`getState()` Methods)

### Concept

Each MFE exposes a **`getState()`** method that returns its current state. Late-loading MFEs can explicitly request the current state instead of relying on past events.

### Implementation

```javascript
// GraphicalContainer MFE exposes its state
window.MFEnamespace.graphicalContainer = {
  getState: () => ({
    metrics: {
      totalRows: 320,
      filteredRows: 89,
      selectedRows: 5,
      lastUpdate: Date.now()
    },
    filters: {
      dateRange: 'last-30-days',
      status: 'active'
    }
  }),

  // Also continue emitting events
  emitMetrics: (data) => {
    eventBus.emit('graphicalContainer:metrics', data);
  }
};

// Notifications MFE exposes its state
window.MFEnamespace.notifications = {
  getState: () => ({
    unread: 5,
    recent: [
      { id: 1, message: 'System alert', level: 'warning' },
      { id: 2, message: 'Update available', level: 'info' }
    ]
  })
};
```

### Dashboard Initialization Pattern

```javascript
// Dashboard MFE loads late
export function bootstrap() {
  // Step 1: Request current state from all MFEs
  initializeDashboard();
  
  // Step 2: Subscribe to future events
  subscribeToEvents();
}

async function initializeDashboard() {
  const mfeNamespace = window.MFEnamespace;

  // Request state from GraphicalContainer
  if (mfeNamespace.graphicalContainer?.getState) {
    const gcState = mfeNamespace.graphicalContainer.getState();
    updateMetricsWidget(gcState.metrics);
  }

  // Request state from Notifications
  if (mfeNamespace.notifications?.getState) {
    const notifState = mfeNamespace.notifications.getState();
    updateNotificationBadge({ count: notifState.unread });
  }

  // Request state from Permissions
  if (mfeNamespace.permissions?.getState) {
    const permState = mfeNamespace.permissions.getState();
    updateUserAccessWidget(permState);
  }
}

function subscribeToEvents() {
  // Now listen for future changes
  eventBus.on('graphicalContainer:metrics', updateMetricsWidget);
  eventBus.on('notifications:unread', updateNotificationBadge);
  eventBus.on('permissions:changed', updateUserAccessWidget);
}
```

### Event-Based State Request (Alternative)

```javascript
// Dashboard broadcasts a "ready" event requesting state
eventBus.emit('dashboard:ready', {
  requestState: ['graphicalContainer', 'notifications', 'permissions']
});

// MFEs respond by emitting their current state
eventBus.on('dashboard:ready', (request) => {
  if (request.requestState.includes('graphicalContainer')) {
    eventBus.emit('graphicalContainer:state', getCurrentState());
  }
});
```

### Pros & Cons

**Pros:**
✅ Always fresh data (no stale snapshots)  
✅ No memory overhead from buffering  
✅ Explicit control over what state is fetched  
✅ Works well with lazy-loaded MFEs

**Cons:**
❌ Requires each MFE to implement `getState()`  
❌ More boilerplate code  
❌ Doesn't capture event history (only current state)  
❌ Coordination needed (what if MFE isn't loaded yet?)

---

## Hybrid Approach: Best of Both Worlds

Combine both patterns for maximum reliability:

```typescript
class SmartEventBus {
  private replayBuffer: Event[] = [];
  private stateProviders = new Map<string, () => any>();

  // MFEs register state providers
  registerStateProvider(namespace: string, provider: () => any) {
    this.stateProviders.set(namespace, provider);
  }

  // Dashboard subscribes with smart initialization
  subscribe(event: string, callback: Function, options?: {
    initializeFromState?: string; // MFE namespace
    replay?: boolean;
  }) {
    // Option 1: Initialize from state provider
    if (options?.initializeFromState) {
      const provider = this.stateProviders.get(options.initializeFromState);
      if (provider) {
        try {
          const state = provider();
          callback(state);
        } catch (err) {
          console.error('Failed to initialize from state:', err);
        }
      }
    }

    // Option 2: Replay past events
    if (options?.replay) {
      this.replayBuffer
        .filter(e => this.matchesPattern(e.event, event))
        .forEach(e => callback(e.payload));
    }

    // Subscribe for future events
    this.addListener(event, callback);
  }
}

// Usage
eventBus.subscribe(
  'graphicalContainer:metrics',
  updateMetricsWidget,
  { 
    initializeFromState: 'graphicalContainer', // Try state first
    replay: true // Fallback to replay if state unavailable
  }
);
```

---

## Real-World Patterns

### Pattern 1: Handshake Protocol

```javascript
// Dashboard announces it's ready
eventBus.emit('dashboard:initialized', {
  mfeId: 'dashboard',
  timestamp: Date.now()
});

// Other MFEs respond with their state
eventBus.on('dashboard:initialized', () => {
  eventBus.emit('graphicalContainer:state:snapshot', {
    metrics: getCurrentMetrics(),
    filters: getCurrentFilters()
  });
});
```

### Pattern 2: Explicit State Request

```javascript
// Dashboard requests specific data
eventBus.emit('dashboard:request:state', {
  from: ['graphicalContainer', 'notifications'],
  dataTypes: ['metrics', 'unread-count']
});

// MFEs respond
eventBus.on('dashboard:request:state', (request) => {
  if (request.from.includes('graphicalContainer')) {
    eventBus.emit('graphicalContainer:response:state', getState());
  }
});
```

### Pattern 3: Persistent State Cache

```javascript
// Orchestrator maintains state cache
const stateCache = {
  'graphicalContainer:metrics': { totalRows: 320, timestamp: 1699456789 },
  'notifications:unread': { count: 5, timestamp: 1699456790 }
};

// Any MFE can query cache
function getCachedState(event: string) {
  const cached = stateCache[event];
  if (cached && (Date.now() - cached.timestamp) < 60000) {
    return cached;
  }
  return null;
}
```

---

## Recommendation for Your System

Based on your architecture, I'd suggest:

### **Use the Hybrid Approach**

```typescript
// shared/eventBus/EventBusAdapter.ts
export class EventBusAdapter implements EventBusPort {
  // Combine both solutions:
  // 1. getState() for initial load
  // 2. Replay buffer for safety net
  // 3. Regular events for ongoing updates

  subscribe(event: string, callback: Function) {
    // 1. Try to get current state first
    const mfeName = event.split(':')[0];
    const stateProvider = window.MFEnamespace[mfeName]?.getState;
    
    if (stateProvider) {
      try {
        const state = stateProvider();
        callback(state);
      } catch (err) {
        console.warn(`State provider failed for ${mfeName}:`, err);
      }
    }

    // 2. Replay recent events as backup
    this.replayRecentEvents(event, callback);

    // 3. Subscribe for future events
    this.addListener(event, callback);
  }
}
```

### Why This Works Best

1. **`getState()` first** → Always get fresh, current state
2. **Replay buffer as backup** → Catch events if `getState()` unavailable
3. **Regular subscriptions** → Stay updated going forward
4. **Graceful degradation** → Works even if some MFEs don't implement `getState()`

---

## Implementation Checklist

### For EventBus/Orchestrator

- [ ] Implement replay buffer with configurable size and TTL
- [ ] Support wildcard event patterns (`graphicalContainer:*`)
- [ ] Deep clone payloads to prevent mutations
- [ ] Clean expired events automatically
- [ ] Provide configuration options (buffer size, TTL)

### For Individual MFEs

- [ ] Expose `getState()` method on `window.MFEnamespace.{mfeName}`
- [ ] Return current state synchronously (or provide async version)
- [ ] Include timestamp/metadata in state
- [ ] Handle errors gracefully
- [ ] Continue emitting events for real-time updates

### For Dashboard MFE

- [ ] Call `getState()` during initialization
- [ ] Subscribe with replay option as fallback
- [ ] Handle missing MFEs gracefully
- [ ] Show loading states while initializing
- [ ] Cache initial state for offline scenarios

---

## Edge Cases to Consider

### 1. MFE Not Loaded Yet

```javascript
// Poll for MFE availability
async function waitForMFE(mfeName: string, timeout = 5000) {
  const start = Date.now();
  while (Date.now() - start < timeout) {
    if (window.MFEnamespace[mfeName]?.getState) {
      return true;
    }
    await new Promise(resolve => setTimeout(resolve, 100));
  }
  return false;
}
```

### 2. State Provider Throws Error

```javascript
try {
  const state = mfeNamespace.graphicalContainer.getState();
  updateWidget(state);
} catch (err) {
  console.error('State fetch failed:', err);
  // Fallback to replay buffer or show placeholder
  showPlaceholderWidget();
}
```

### 3. Stale State Detection

```javascript
const state = mfeNamespace.graphicalContainer.getState();
if (Date.now() - state.lastUpdate > 60000) {
  console.warn('State is stale, requesting refresh');
  eventBus.emit('graphicalContainer:refresh');
}
```

### 4. Circular Dependencies

```javascript
// Prevent infinite loops
const initializing = new Set();
function getState(mfeName: string) {
  if (initializing.has(mfeName)) {
    return null; // Circular dependency detected
  }
  initializing.add(mfeName);
  try {
    return window.MFEnamespace[mfeName].getState();
  } finally {
    initializing.delete(mfeName);
  }
}
```

---

## Summary

The Event Replay / Snapshot System is essential for ensuring that the Dashboard MFE (or any late-loading MFE) can synchronize with the rest of the application, even when loaded asynchronously.

**Key Takeaways:**

1. **Event Replay Buffer** provides automatic synchronization but may have stale data
2. **State Snapshots (`getState()`)** provide fresh data but require implementation in each MFE
3. **Hybrid Approach** combines both for maximum reliability
4. **Graceful Degradation** ensures the system works even when some MFEs don't support state snapshots

**Result**: Dashboard always in sync, even when loaded asynchronously. ✅
