# Shortcuts System for Dashboard MFE

Shortcuts are quick-access buttons or links that trigger actions or navigate to specific features. They're the "speed dial" of your dashboard, providing users with instant access to common tasks and frequently used features.

## Overview

Shortcuts differ from widgets in that they:

- **Trigger actions** rather than display data
- Are **action-oriented** (navigate, execute, open)
- Provide **quick access** to features across MFEs
- Can be **customized and personalized** by users

## Real-World Enterprise Examples

### Salesforce Lightning

```
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ + New Lead  │ │ Create Case │ │ View Reports│
└─────────────┘ └─────────────┘ └─────────────┘
```

### Microsoft 365 Dashboard

```
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│📧 Email  │ │📅 Calendar││📁 Files  ││👥 Teams  │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

### AWS Console

```
Recently visited services:
- EC2
- S3
- CloudWatch
- Lambda
```

## Types of Shortcuts

### 1. Navigation Shortcuts

Jump to specific MFEs or routes within the application.

```javascript
{
  id: 'goto-notifications',
  label: 'View Notifications',
  icon: 'bell',
  action: {
    type: 'navigate',
    target: '/notifications'
  }
}
```

**Use Cases:**

- Navigate to specific MFE routes
- Jump to feature pages
- Open sub-applications

### 2. Action Shortcuts

Trigger specific operations or events.

```javascript
{
  id: 'export-data',
  label: 'Export Current Data',
  icon: 'download',
  action: {
    type: 'event',
    event: 'graphicalContainer:export',
    payload: { format: 'csv' }
  }
}
```

**Use Cases:**

- Export data in various formats
- Trigger batch operations
- Execute system commands
- Refresh data sources

### 3. Modal/Overlay Shortcuts

Open dialogs, panels, or overlay interfaces.

```javascript
{
  id: 'quick-search',
  label: 'Search',
  icon: 'search',
  hotkey: 'cmd+k',
  action: {
    type: 'modal',
    component: 'QuickSearchModal'
  }
}
```

**Use Cases:**

- Quick search dialogs
- Create/edit forms
- Settings panels
- Help/documentation overlays

### 4. External Shortcuts

Link to external tools or resources.

```javascript
{
  id: 'open-grafana',
  label: 'Monitoring Dashboard',
  icon: 'external-link',
  action: {
    type: 'external',
    url: 'https://grafana.company.com',
    openInNewTab: true
  }
}
```

**Use Cases:**

- External monitoring tools
- Documentation sites
- Third-party integrations
- Legacy systems

### 5. Command Shortcuts

Execute system-wide commands or broadcast events.

```javascript
{
  id: 'refresh-all',
  label: 'Refresh All Data',
  icon: 'refresh',
  hotkey: 'cmd+r',
  action: {
    type: 'broadcast',
    event: 'dashboard:refresh:all'
  }
}
```

**Use Cases:**

- Global refresh operations
- System-wide notifications
- Multi-MFE coordination
- Bulk actions

## Shortcut Registration Pattern

Any MFE can register shortcuts via the event bus:

```javascript
// Any MFE can register shortcuts
eventBus.emit('dashboard:registerShortcut', {
  id: 'create-notification',
  label: 'New Alert',
  icon: 'bell-plus',
  category: 'notifications',
  permissions: ['notifications:create'],
  action: {
    type: 'event',
    event: 'notifications:create:dialog'
  },
  metadata: {
    color: 'blue',
    priority: 5,
    tooltip: 'Create a new notification'
  }
});
```

### Registration Event Contract

```typescript
interface ShortcutRegistration {
  id: string;
  label: string;
  icon: string;
  category?: string;
  hotkey?: string;
  badge?: number | string;
  permissions?: string[];
  action: ShortcutAction;
  metadata?: {
    color?: string;
    priority?: number;
    tooltip?: string;
    visible?: boolean;
  };
}
```

## Data Structure

```typescript
interface Shortcut {
  id: string;
  label: string;
  icon: string;
  category?: string;
  hotkey?: string;
  badge?: number | string;
  permissions?: string[];
  action: ShortcutAction;
  metadata?: {
    color?: string;
    priority?: number;
    tooltip?: string;
    visible?: boolean;
  };
}

type ShortcutAction =
  | { type: 'navigate'; target: string }
  | { type: 'event'; event: string; payload?: any }
  | { type: 'modal'; component: string }
  | { type: 'external'; url: string; openInNewTab?: boolean }
  | { type: 'broadcast'; event: string };
```

## Key Features

### Customization

Users can personalize their shortcuts:

- **Pin/Unpin**: Add or remove shortcuts from the main bar
- **Reorder**: Drag to rearrange shortcut positions
- **Organize**: Group by categories (e.g., "Actions", "Navigation", "Tools")
- **Hide**: Remove shortcuts they don't use

### Smart Shortcuts

Advanced features for better UX:

- **Recently Used**: Show shortcuts based on usage frequency
- **Context-Aware**: Display shortcuts relevant to current view
- **Badge Counts**: Show notification counts (e.g., "3 pending approvals")
- **Dynamic Visibility**: Show/hide based on permissions or state

### Keyboard Support

Power user features:

- **Hotkeys**: Assign keyboard shortcuts (e.g., Cmd+K for search)
- **Number Shortcuts**: Quick access via Alt+1, Alt+2, etc.
- **Command Palette**: Searchable shortcut interface

## Storage Pattern

```javascript
// localStorage structure
{
  "mfe.dashboard.shortcuts": {
    "pinned": ["goto-notifications", "export-data", "quick-search"],
    "order": ["quick-search", "goto-notifications", "export-data"],
    "hidden": ["old-feature-shortcut"],
    "recentlyUsed": [
      { id: "export-data", lastUsed: 1699456789000 },
      { id: "quick-search", lastUsed: 1699456123000 }
    ],
    "categories": {
      "navigation": ["goto-notifications", "goto-permissions"],
      "actions": ["export-data", "refresh-all"],
      "tools": ["quick-search", "open-grafana"]
    }
  }
}
```

## Event Contracts

### Registration

```javascript
// Event: dashboard:registerShortcut
// Direction: any MFE → Dashboard
// Payload: ShortcutRegistration
eventBus.emit('dashboard:registerShortcut', {
  id: 'my-shortcut',
  label: 'My Action',
  icon: 'star',
  action: { type: 'navigate', target: '/my-feature' }
});
```

### Activation

```javascript
// Event: dashboard:shortcut:activated
// Direction: Dashboard → all MFEs
// Payload: { shortcutId: string, action: ShortcutAction }
eventBus.emit('dashboard:shortcut:activated', {
  shortcutId: 'export-data',
  action: { type: 'event', event: 'graphicalContainer:export' }
});
```

### Update/Remove

```javascript
// Update shortcut
eventBus.emit('dashboard:shortcut:update', {
  id: 'my-shortcut',
  updates: { badge: 5 }
});

// Remove shortcut
eventBus.emit('dashboard:shortcut:remove', {
  id: 'my-shortcut'
});
```

## UI Layout Examples

### Horizontal Bar Layout

```
┌─────────────────────────────────────────────────────────┐
│  ⚡ Export  │  ⚡ Refresh  │  ⚡ Search  │  ⚡ Settings  │
└─────────────────────────────────────────────────────────┘
```

### Grid Layout

```
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│  Export  │ │  Refresh │ │  Search  │ │ Settings │
│   📥     │ │   🔄     │ │   🔍     │ │   ⚙️     │
└──────────┘ └──────────┘ └──────────┘ └──────────┘
```

### Dropdown Menu

```
┌─────────────────┐
│ Quick Actions ▼ │
├─────────────────┤
│ ⚡ Export Data  │
│ ⚡ Refresh All   │
│ ⚡ New Item      │
│ ─────────────── │
│ ⚡ Settings      │
└─────────────────┘
```

## Integration with Other Dashboard Elements

### Shortcuts + Bookmarks

- Shortcuts can save current state as a bookmark
- Bookmarks can be accessed via shortcuts

### Shortcuts + Filters

- Shortcuts can apply filter presets
- Filter actions can be triggered via shortcuts

### Shortcuts + Widgets

- Shortcuts can add/remove widgets
- Widget actions can be exposed as shortcuts

## Best Practices

### 1. Limit Shortcut Count

- Keep main shortcuts to 5-8 items
- Use categories to organize more shortcuts
- Provide "More" dropdown for additional options

### 2. Clear Labeling

- Use action verbs: "Export", "Create", "View"
- Keep labels concise (1-3 words)
- Provide tooltips for additional context

### 3. Icon Consistency

- Use consistent icon library (e.g., Material Icons, Font Awesome)
- Choose icons that clearly represent the action
- Consider color coding by category

### 4. Permission-Based Visibility

- Hide shortcuts users don't have access to
- Show disabled state with explanation if needed
- Respect MFE-level permissions

### 5. Keyboard Accessibility

- Support keyboard navigation
- Provide visual focus indicators
- Include ARIA labels for screen readers

## Summary

Shortcuts provide a powerful way for MFEs to expose quick actions and navigation options to users. They complement widgets by focusing on **actions** rather than **data display**, creating a complete dashboard experience where users can both view information and take action efficiently.

The decoupled event-based registration pattern ensures that any MFE can contribute shortcuts without tight coupling, maintaining the architectural principles of the Dashboard MFE system.
