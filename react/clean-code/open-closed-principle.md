# Open-Closed Principle (OCP) in React

## Overview

The Open-Closed Principle states that components should be open for extension
but closed for modification. In React, this means using composition, children
props, and render props to allow components to be extended with new behavior
without changing their source code.

## Core Concept

In React, OCP means:

* Use composition over modification
* Leverage children and render props for flexibility
* Create plugin-like systems with component maps
* Design components to accept new variants without editing
* Use discriminated unions for extensible behavior

## Implementation Example

### Scaffolding for Examples

```typescript
// types.ts
interface Notification {
  id: string
  type: 'info' | 'warning' | 'error' | 'success'
  title: string
  message: string
}

interface IconProps {
  className?: string
  size?: number
}
```

### BAD — Switch Statement Modification (OCP violation)

> Adding new notification types requires modifying the component.

```typescript
function NotificationItem({ notification }: { notification: Notification }) {
  // VIOLATION: Must edit this switch for every new type
  const getIcon = () => {
    switch (notification.type) {
      case 'info':
        return <InfoIcon />
      case 'warning':
        return <WarningIcon />
      case 'error':
        return <ErrorIcon />
      case 'success':
        return <CheckIcon />
      default:
        return null
    }
  }

  // VIOLATION: Must edit this switch for every new style
  const getClassName = () => {
    switch (notification.type) {
      case 'info':
        return 'notification-info'
      case 'warning':
        return 'notification-warning'
      case 'error':
        return 'notification-error'
      case 'success':
        return 'notification-success'
      default:
        return 'notification-default'
    }
  }

  // VIOLATION: Must edit component to add new behaviors
  const handleClick = () => {
    switch (notification.type) {
      case 'error':
        console.error('Error clicked:', notification.message)
        break
      case 'warning':
        console.warn('Warning clicked:', notification.message)
        break
      default:
        console.log('Notification clicked:', notification.message)
    }
  }

  return (
    <div className={getClassName()} onClick={handleClick}>
      <div className="notification-icon">{getIcon()}</div>
      <div className="notification-content">
        <h4>{notification.title}</h4>
        <p>{notification.message}</p>
      </div>
    </div>
  )
}

// VIOLATION: Adding a new type requires editing the component
// What if we want to add 'debug', 'critical', or custom types?
```

### GOOD — Composition-Based Extension (OCP compliant)

> New notification types can be added without modifying existing components.

```typescript
// Define notification renderer interface
interface NotificationRenderer {
  icon: React.ComponentType<IconProps>
  className: string
  onClick?: (notification: Notification) => void
}

// Registry pattern for extensibility
class NotificationRegistry {
  private renderers = new Map<string, NotificationRenderer>()

  register(type: string, renderer: NotificationRenderer) {
    this.renderers.set(type, renderer)
    return this
  }

  get(type: string): NotificationRenderer | undefined {
    return this.renderers.get(type)
  }
}

// Base notification component - closed for modification
function NotificationItem({ 
  notification,
  renderer 
}: { 
  notification: Notification
  renderer: NotificationRenderer
}) {
  const Icon = renderer.icon
  
  return (
    <div 
      className={`notification ${renderer.className}`}
      onClick={() => renderer.onClick?.(notification)}
    >
      <div className="notification-icon">
        <Icon size={20} />
      </div>
      <div className="notification-content">
        <h4>{notification.title}</h4>
        <p>{notification.message}</p>
      </div>
    </div>
  )
}

// Composition-based notification system
function NotificationSystem({ 
  notifications,
  registry 
}: {
  notifications: Notification[]
  registry: NotificationRegistry
}) {
  return (
    <div className="notification-container">
      {notifications.map(notification => {
        const renderer = registry.get(notification.type)
        if (!renderer) return null
        
        return (
          <NotificationItem
            key={notification.id}
            notification={notification}
            renderer={renderer}
          />
        )
      })}
    </div>
  )
}

// Configure renderers - open for extension
const createNotificationRegistry = () => {
  return new NotificationRegistry()
    .register('info', {
      icon: InfoIcon,
      className: 'notification-info',
      onClick: (n) => console.log('Info:', n.message)
    })
    .register('warning', {
      icon: WarningIcon,
      className: 'notification-warning',
      onClick: (n) => console.warn('Warning:', n.message)
    })
    .register('error', {
      icon: ErrorIcon,
      className: 'notification-error',
      onClick: (n) => console.error('Error:', n.message)
    })
    .register('success', {
      icon: CheckIcon,
      className: 'notification-success'
    })
}

// Adding new types without modification
const extendedRegistry = createNotificationRegistry()
  .register('critical', {
    icon: AlertIcon,
    className: 'notification-critical',
    onClick: (n) => {
      alertService.send(n)
      console.error('CRITICAL:', n.message)
    }
  })
  .register('debug', {
    icon: BugIcon,
    className: 'notification-debug',
    onClick: (n) => debugger
  })

// Alternative: Using children for extension
function Card({ 
  header,
  footer,
  children 
}: {
  header?: React.ReactNode
  footer?: React.ReactNode
  children: React.ReactNode
}) {
  return (
    <div className="card">
      {header && <div className="card-header">{header}</div>}
      <div className="card-body">{children}</div>
      {footer && <div className="card-footer">{footer}</div>}
    </div>
  )
}

// Extend via composition, not modification
function UserCard({ user }: { user: User }) {
  return (
    <Card
      header={<UserAvatar user={user} />}
      footer={<UserActions user={user} />}
    >
      <UserDetails user={user} />
    </Card>
  )
}

// Alternative: Render props for behavior extension
interface ListProps<T> {
  items: T[]
  renderItem: (item: T, index: number) => React.ReactNode
  renderEmpty?: () => React.ReactNode
  renderLoading?: () => React.ReactNode
  loading?: boolean
}

function List<T>({ 
  items,
  renderItem,
  renderEmpty = () => <div>No items</div>,
  renderLoading = () => <div>Loading...</div>,
  loading = false
}: ListProps<T>) {
  if (loading) return <>{renderLoading()}</>
  if (items.length === 0) return <>{renderEmpty()}</>
  
  return (
    <div className="list">
      {items.map((item, index) => (
        <div key={index} className="list-item">
          {renderItem(item, index)}
        </div>
      ))}
    </div>
  )
}

// Extend list behavior without modification
function UserList({ users }: { users: User[] }) {
  return (
    <List
      items={users}
      renderItem={(user) => <UserCard user={user} />}
      renderEmpty={() => <EmptyState message="No users found" />}
      renderLoading={() => <Skeleton count={5} />}
    />
  )
}
```

## When to Apply OCP in React

### Use OCP for

* Components that need multiple visual variants
* Lists that render different item types
* Forms with dynamic field types
* Notification/alert systems
* Theme-able components

### Keep Simple When

* Component has truly fixed behavior
* Only 2-3 well-defined variants exist
* Changes are extremely unlikely
* Performance is critical

## Anti-patterns to Avoid

1. **Giant switch statements**: Checking type to determine behavior
2. **Flag proliferation**: Many boolean props for variants
3. **Inheritance chains**: Using class inheritance for variants
4. **Over-abstraction**: Making everything pluggable when not needed

## React-Specific OCP Techniques

1. **Children props** for content extension
2. **Render props** for behavior customization
3. **Component maps** for type-based rendering
4. **Higher-order components** for behavior decoration
5. **Context providers** for cross-cutting extensions

## Key Takeaways

* Use **composition** to add new features
* **Children and render props** enable extension
* Create **registries** for pluggable behavior
* Design components to be **extended, not modified**
* Keep core components **stable and closed**

## Related Best Practices

For component composition and patterns, see
👉 [best-practices.md](../best-practices.md)
