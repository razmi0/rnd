# Bookmarks System for Dashboard MFE

Bookmarks are saved application states that allow users to return to specific views, filters, or configurations instantly. They enable users to preserve and quickly access their preferred dashboard configurations, filtered views, and custom layouts.

## Overview

Bookmarks differ from shortcuts in that they:

- **Save state** rather than trigger actions
- **Preserve context** (filters, layout, route)
- Allow **quick return** to specific views
- Support **sharing and collaboration**

## Real-World Enterprise Examples

### Google Analytics

- Saved reports with specific date ranges and filters
- Custom dashboard configurations
- Shared report bookmarks across teams

### JIRA

- Saved filters/JQL queries
- Favorite boards and views
- Bookmarked sprint views

### Tableau

- Saved workbook views
- Custom parameter sets
- Shared dashboard bookmarks

### AWS Console

- Pinned services
- Saved CloudWatch metric views
- Bookmarked resource configurations

## Types of Bookmarks

### 1. View Bookmarks

Save entire page states including route, filters, columns, and sort order.

```javascript
{
  id: 'monthly-sales-view',
  name: 'Monthly Sales Report',
  type: 'view',
  state: {
    route: '/graphical-container',
    filters: {
      dateRange: 'last-30-days',
      region: 'EMEA',
      status: 'active'
    },
    columns: ['name', 'amount', 'date'],
    sort: { field: 'amount', order: 'desc' }
  },
  createdAt: '2024-10-15T10:30:00Z'
}
```

**Use Cases:**

- Save filtered table views
- Preserve dashboard layouts
- Remember specific page states

### 2. Filter Bookmarks

Save specific filter combinations for quick reapplication.

```javascript
{
  id: 'high-priority-bugs',
  name: 'High Priority Bugs',
  type: 'filter',
  filters: {
    type: 'bug',
    priority: ['high', 'critical'],
    status: 'open',
    assignee: 'current-user'
  }
}
```

**Use Cases:**

- Common filter combinations
- Saved search queries
- Reusable filter presets

### 3. Dashboard Layout Bookmarks

Save widget arrangements and dashboard configurations.

```javascript
{
  id: 'exec-dashboard',
  name: 'Executive Dashboard',
  type: 'layout',
  layout: {
    widgets: [
      { id: 'revenue-chart', position: { x: 0, y: 0, w: 6, h: 4 } },
      { id: 'kpi-metrics', position: { x: 6, y: 0, w: 6, h: 2 } },
      { id: 'recent-alerts', position: { x: 6, y: 2, w: 6, h: 2 } }
    ],
    filters: {
      dateRange: 'this-quarter',
      department: 'all'
    }
  }
}
```

**Use Cases:**

- Custom dashboard layouts
- Role-specific views (executive, manager, analyst)
- Project-specific configurations

### 4. Quick Access Bookmarks

Favorites/pins for frequently accessed queries or entities.

```javascript
{
  id: 'favorite-customers',
  name: 'Top 10 Customers',
  type: 'query',
  query: {
    entity: 'customers',
    sort: 'revenue',
    limit: 10
  }
}
```

**Use Cases:**

- Favorite entities
- Pinned queries
- Quick access lists

## Data Structure

```typescript
interface Bookmark {
  id: string;
  name: string;
  description?: string;
  type: 'view' | 'filter' | 'layout' | 'query';
  icon?: string;
  tags?: string[];
  folder?: string;
  visibility: 'private' | 'team' | 'public';
  state: BookmarkState;
  metadata: {
    createdAt: string;
    updatedAt?: string;
    createdBy: string;
    usageCount: number;
    lastAccessed?: string;
    favorite?: boolean;
    expiresAt?: string;
  };
}

interface BookmarkState {
  route?: string;
  filters?: Record<string, any>;
  widgets?: WidgetLayout[];
  query?: QueryConfig;
  params?: Record<string, any>;
}

interface WidgetLayout {
  id: string;
  position: {
    x: number;
    y: number;
    w: number;
    h: number;
  };
  config?: Record<string, any>;
}

interface QueryConfig {
  entity: string;
  filters?: Record<string, any>;
  sort?: { field: string; order: 'asc' | 'desc' };
  limit?: number;
  offset?: number;
}
```

## Key Features

### Sharing

Bookmarks can be shared across users and teams:

- **Generate Shareable URLs**: Create links that restore bookmark state
- **Export/Import**: Share bookmark collections as JSON
- **Team Bookmarks**: Shared bookmarks visible to team members
- **Public Bookmarks**: Organization-wide bookmark library

```javascript
// Generate shareable URL
const bookmarkUrl = `https://dashboard.company.com/view?bookmark=${bookmarkId}`;

// Or with full state encoded
const stateUrl = `https://dashboard.company.com/graphical?
  filters=${encodeURIComponent(JSON.stringify(filters))}
  &layout=${encodeURIComponent(JSON.stringify(layout))}`;
```

### Organization

Users can organize bookmarks for easy access:

- **Folders**: Group related bookmarks
- **Tags**: Multi-label categorization
- **Search**: Find bookmarks by name, tags, or content
- **Favorites**: Pin frequently used bookmarks

```javascript
{
  "folders": [
    { 
      id: "reports", 
      name: "Reports", 
      bookmarks: ["monthly-sales", "quarterly-review"],
      icon: "📊"
    },
    { 
      id: "monitoring", 
      name: "Monitoring", 
      bookmarks: ["server-health", "api-metrics"],
      icon: "📈"
    }
  ],
  "tags": ["sales", "reports", "monthly", "executive"],
  "favorites": ["monthly-sales", "exec-dashboard"]
}
```

### Smart Features

Advanced bookmark capabilities:

- **Auto-Update Data**: Bookmarks stay "live" - data refreshes when accessed
- **Expiration Dates**: Time-sensitive bookmarks expire automatically
- **Notifications**: Alert when bookmarked data changes significantly
- **Usage Tracking**: Track how often bookmarks are accessed

## Bookmark Registration & Activation

### Creating a Bookmark

```javascript
// Save current state as bookmark
eventBus.emit('dashboard:bookmark:create', {
  name: 'Q4 Performance View',
  description: 'Quarterly performance metrics',
  type: 'view',
  tags: ['performance', 'quarterly', 'executive'],
  folder: 'reports',
  visibility: 'team',
  state: {
    route: window.location.pathname,
    filters: getCurrentFilters(),
    widgets: getCurrentWidgetLayout(),
    params: getCurrentParams()
  }
});

// Dashboard responds by:
// 1. Validating bookmark data
// 2. Saving to localStorage
// 3. Emitting confirmation event
// 4. Updating UI to show new bookmark
```

### Activating a Bookmark

```javascript
// Apply a bookmark
eventBus.emit('dashboard:bookmark:activate', {
  bookmarkId: 'q4-performance-view'
});

// Dashboard responds by:
// 1. Loading bookmark state from storage
// 2. Restoring filters via 'dashboard:filter:changed'
// 3. Rearranging widgets via 'dashboard:widget:layout:update'
// 4. Navigating to saved route
// 5. Updating URL if needed
```

### Updating a Bookmark

```javascript
// Update existing bookmark
eventBus.emit('dashboard:bookmark:update', {
  bookmarkId: 'q4-performance-view',
  updates: {
    name: 'Q4 2024 Performance View',
    state: {
      // Updated state
    }
  }
});
```

### Deleting a Bookmark

```javascript
// Remove bookmark
eventBus.emit('dashboard:bookmark:delete', {
  bookmarkId: 'q4-performance-view'
});
```

## Storage Pattern

```javascript
{
  "mfe.dashboard.bookmarks": {
    "items": [
      {
        id: "monthly-sales",
        name: "Monthly Sales Report",
        description: "Sales metrics for the current month",
        type: "view",
        icon: "📊",
        tags: ["sales", "monthly", "reports"],
        folder: "reports",
        visibility: "team",
        state: {
          route: "/graphical-container",
          filters: {
            dateRange: "this-month",
            region: "all"
          },
          columns: ["name", "amount", "date"],
          sort: { field: "amount", order: "desc" }
        },
        metadata: {
          createdAt: "2024-10-15T10:30:00Z",
          updatedAt: "2024-10-20T14:22:00Z",
          createdBy: "user-123",
          usageCount: 47,
          lastAccessed: "2024-10-20T14:22:00Z",
          favorite: true,
          expiresAt: null
        }
      }
    ],
    "folders": [
      { 
        id: "reports", 
        name: "Reports", 
        icon: "📊",
        bookmarks: ["monthly-sales", "quarterly-review"],
        order: 1
      },
      { 
        id: "monitoring", 
        name: "Monitoring", 
        icon: "📈",
        bookmarks: ["server-health"],
        order: 2
      }
    ],
    "favorites": ["monthly-sales", "exec-dashboard"],
    "recent": [
      { id: "monthly-sales", accessedAt: "2024-10-20T14:22:00Z" },
      { id: "high-priority-bugs", accessedAt: "2024-10-20T10:15:00Z" }
    ],
    "tags": {
      "sales": ["monthly-sales", "quarterly-review"],
      "reports": ["monthly-sales", "exec-dashboard"],
      "monitoring": ["server-health", "api-metrics"]
    }
  }
}
```

## URL Encoding Pattern

Bookmarks can be encoded in URLs for sharing:

### Simple Bookmark Reference

```
https://dashboard.company.com/view?bookmark=monthly-sales
```

### Full State Encoding

```
https://dashboard.company.com/graphical?
  filters=eyJkYXRlUmFuZ2UiOiJsYXN0LTMwLWRheXMifQ==
  &layout=eyJ3aWRnZXRzIjpbXX0=
  &columns=name,amount,date
  &sort=amount:desc
```

### Base64 Encoded State

```javascript
// Encode bookmark state
const encodedState = btoa(JSON.stringify({
  filters: { dateRange: 'last-30-days' },
  layout: { widgets: [] }
}));

// Decode on load
const decodedState = JSON.parse(atob(encodedState));
```

## Event **Contracts

### Create Bookmark

```javascript
// Event: dashboard:bookmark:create
// Direction: Dashboard → Event Bus (internal)
// Payload: BookmarkRegistration
eventBus.emit('dashboard:bookmark:create', {
  name: 'My Bookmark',
  type: 'view',
  state: { /* ... */ }
});
```

### Activate Bookmark

```javascript
// Event: dashboard:bookmark:activate
// Direction: Dashboard → Event Bus → Affected MFEs
// Payload: { bookmarkId: string }
eventBus.emit('dashboard:bookmark:activate', {
  bookmarkId: 'monthly-sales'
});
```

### Update Bookmark

```javascript
// Event: dashboard:bookmark:update
// Direction: Dashboard → Event Bus (internal)
// Payload: { bookmarkId: string, updates: Partial<Bookmark> }
eventBus.emit('dashboard:bookmark:update', {
  bookmarkId: 'monthly-sales',
  updates: { name: 'Updated Name' }
});
```

### Delete Bookmark

```javascript
// Event: dashboard:bookmark:delete
// Direction: Dashboard → Event Bus (internal)
// Payload: { bookmarkId: string }
eventBus.emit('dashboard:bookmark:delete', {
  bookmarkId: 'monthly-sales'
});
```

### Bookmark Changed (from other MFEs)

```javascript
// Event: mfe:bookmarks:changed
// Direction: Other MFEs → Dashboard
// Payload: { bookmarks: Bookmark[] }
eventBus.emit('mfe:bookmarks:changed', {
  bookmarks: [
    { id: 'gc-filter-1', name: 'Active Items', filters: {...} }
  ]
});
```

## UI Layout Examples

### Sidebar Bookmarks Panel

```
┌─────────────────────┐
│ 🔖 Bookmarks        │
├─────────────────────┤
│ 📊 Reports          │
│   • Monthly Sales   │
│   • Quarterly       │
│                     │
│ 📈 Monitoring       │
│   • Server Health   │
│                     │
│ ⭐ Favorites        │
│   • Monthly Sales   │
│   • Exec Dashboard  │
└─────────────────────┘
```

### Dropdown Menu

```
┌──────────────────────────┐
│ Bookmarks ▼              │
├──────────────────────────┤
│ ⭐ Monthly Sales         │
│ ⭐ Executive Dashboard   │
│ ──────────────────────── │
│ 📊 Reports               │
│ 📈 Monitoring            │
│ ──────────────────────── │
│ + Create Bookmark        │
└──────────────────────────┘
```

### Grid View

```
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Monthly Sales│ │ Exec Dashboard││ Server Health│
│    📊        │ │     📈       │ │     🔍       │
│ Last used:   │ │ Last used:   │ │ Last used:   │
│ 2 hours ago  │ │ 1 day ago    │ │ 5 min ago    │
└──────────────┘ └──────────────┘ └──────────────┘
```

## Integration with Other Dashboard Elements

### Bookmarks + Shortcuts

- Shortcuts can save current state as a bookmark
- Bookmarks can be accessed via shortcuts
- "Save as Bookmark" shortcut action

### Bookmarks + Filters

- Bookmarks preserve filter states
- Filter presets can be saved as bookmarks
- Applying bookmark restores filters

### Bookmarks + Widgets

- Bookmarks save widget layouts
- Widget configurations preserved in bookmarks
- Applying bookmark restores widget arrangement

## Best Practices

### 1. Meaningful Names

- Use descriptive, user-friendly names
- Include context (e.g., "Q4 2024 Sales" vs "Sales")
- Support search-friendly naming

### 2. State Validation

- Validate bookmark state on load
- Handle missing or invalid data gracefully
- Provide fallbacks for deprecated states

### 3. Performance

- Lazy load bookmark metadata
- Cache frequently accessed bookmarks
- Limit bookmark count per user

### 4. Versioning

- Handle state schema changes
- Migrate old bookmarks to new formats
- Deprecate incompatible bookmarks

### 5. Privacy & Permissions

- Respect visibility settings (private/team/public)
- Check permissions before sharing
- Audit bookmark access

## Summary

Bookmarks provide a powerful way for users to save and quickly return to their preferred dashboard configurations. They complement widgets and shortcuts by focusing on **state preservation** rather than **data display** or **action execution**, creating a complete dashboard experience where users can customize, save, and share their views.

The event-based system ensures that bookmarks can be created, activated, and managed without tight coupling between MFEs, maintaining the architectural principles of the Dashboard MFE system.
