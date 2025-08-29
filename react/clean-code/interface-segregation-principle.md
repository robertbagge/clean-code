# Interface Segregation Principle (ISP) in React

## Overview

The Interface Segregation Principle states that components should not be forced
to depend on props they don't use. In React, this means creating focused prop
interfaces where components only receive the data and callbacks they actually
need, improving reusability and testability.

## Core Concept

In React, ISP means:

* Components receive only the props they use
* Split large prop interfaces into focused ones
* Use composition to combine capabilities
* Avoid "prop drilling" unnecessary data
* Create lean, specific interfaces for each use case

## Implementation Example

### Scaffolding for Examples

```typescript
// types.ts
interface User {
  id: string
  name: string
  email: string
  role: 'admin' | 'user' | 'guest'
  preferences: {
    theme: 'light' | 'dark'
    notifications: boolean
  }
  profile: {
    avatar?: string
    bio?: string
    location?: string
  }
}

interface Product {
  id: string
  name: string
  price: number
  stock: number
}
```

### BAD — Fat Interface (ISP violation)

> Components receive entire objects and props they don't use.

```typescript
// VIOLATION: Component receives entire user object but uses only 2 fields
function UserAvatar({ user }: { user: User }) {
  return (
    <div className="avatar">
      <img src={user.profile.avatar || '/default-avatar.png'} />
      <span>{user.name}</span>
    </div>
  )
}

// VIOLATION: Fat props interface with many unused fields
interface DashboardProps {
  user: User
  products: Product[]
  onUserUpdate: (user: User) => void
  onProductUpdate: (product: Product) => void
  onLogout: () => void
  onNavigate: (path: string) => void
  theme: 'light' | 'dark'
  language: string
  permissions: string[]
  analyticsEnabled: boolean
}

// VIOLATION: Header only uses 3 of 10 props
function DashboardHeader({ 
  user,
  onLogout,
  theme,
  // These are passed but never used
  products,
  onUserUpdate,
  onProductUpdate,
  onNavigate,
  language,
  permissions,
  analyticsEnabled
}: DashboardProps) {
  return (
    <header className={`header-${theme}`}>
      <UserAvatar user={user} />
      <button onClick={onLogout}>Logout</button>
    </header>
  )
}

// VIOLATION: Sidebar uses different subset of props
function DashboardSidebar({
  user,
  onNavigate,
  permissions,
  // These are passed but never used
  products,
  onUserUpdate,
  onProductUpdate,
  onLogout,
  theme,
  language,
  analyticsEnabled
}: DashboardProps) {
  return (
    <nav>
      {permissions.includes('admin') && (
        <button onClick={() => onNavigate('/admin')}>Admin</button>
      )}
      <button onClick={() => onNavigate('/profile')}>Profile</button>
    </nav>
  )
}

// VIOLATION: Form component with too many responsibilities
interface UserFormProps {
  user: User
  products: Product[]  // Why does a user form need products?
  onSave: (user: User) => void
  onCancel: () => void
  onDelete: (id: string) => void
  onImageUpload: (file: File) => Promise<string>
  validationRules: Record<string, any>
  availableRoles: string[]
  availableThemes: string[]
  maxBioLength: number
}

function UserForm({
  user,
  products,  // Never used
  onSave,
  onCancel,
  onDelete,  // Only used if editing
  onImageUpload,  // Only used in profile section
  validationRules,
  availableRoles,
  availableThemes,
  maxBioLength
}: UserFormProps) {
  // Component is forced to handle props it doesn't always need
  return <form>{/* ... */}</form>
}
```

### GOOD — Segregated Interfaces (ISP compliant)

> Components receive only the props they actually use.

```typescript
// Focused interface - only what's needed
interface AvatarProps {
  src?: string
  alt: string
}

function UserAvatar({ src, alt }: AvatarProps) {
  return (
    <div className="avatar">
      <img src={src || '/default-avatar.png'} alt={alt} />
      <span>{alt}</span>
    </div>
  )
}

// Segregated interfaces for different concerns
interface HeaderProps {
  userName: string
  userAvatar?: string
  theme: 'light' | 'dark'
  onLogout: () => void
}

function DashboardHeader({ userName, userAvatar, theme, onLogout }: HeaderProps) {
  return (
    <header className={`header-${theme}`}>
      <UserAvatar src={userAvatar} alt={userName} />
      <button onClick={onLogout}>Logout</button>
    </header>
  )
}

interface NavigationProps {
  isAdmin: boolean
  onNavigate: (path: string) => void
}

function DashboardSidebar({ isAdmin, onNavigate }: NavigationProps) {
  return (
    <nav>
      {isAdmin && (
        <button onClick={() => onNavigate('/admin')}>Admin</button>
      )}
      <button onClick={() => onNavigate('/profile')}>Profile</button>
    </nav>
  )
}

// Compose focused components instead of one fat component
function Dashboard({ user }: { user: User }) {
  const navigate = useNavigate()
  const { logout } = useAuth()
  const { theme } = useTheme()
  
  return (
    <div className="dashboard">
      <DashboardHeader
        userName={user.name}
        userAvatar={user.profile.avatar}
        theme={theme}
        onLogout={logout}
      />
      <DashboardSidebar
        isAdmin={user.role === 'admin'}
        onNavigate={navigate}
      />
    </div>
  )
}

// Split form into focused sections with specific interfaces
interface BasicInfoProps {
  name: string
  email: string
  onChange: (field: string, value: string) => void
}

function BasicInfoSection({ name, email, onChange }: BasicInfoProps) {
  return (
    <div>
      <input
        value={name}
        onChange={(e) => onChange('name', e.target.value)}
        placeholder="Name"
      />
      <input
        value={email}
        onChange={(e) => onChange('email', e.target.value)}
        placeholder="Email"
      />
    </div>
  )
}

interface ProfileProps {
  bio?: string
  location?: string
  maxBioLength: number
  onChange: (field: string, value: string) => void
}

function ProfileSection({ bio, location, maxBioLength, onChange }: ProfileProps) {
  return (
    <div>
      <textarea
        value={bio || ''}
        onChange={(e) => onChange('bio', e.target.value)}
        maxLength={maxBioLength}
        placeholder="Bio"
      />
      <input
        value={location || ''}
        onChange={(e) => onChange('location', e.target.value)}
        placeholder="Location"
      />
    </div>
  )
}

interface PreferencesProps {
  theme: 'light' | 'dark'
  notifications: boolean
  availableThemes: string[]
  onChange: (field: string, value: any) => void
}

function PreferencesSection({ 
  theme,
  notifications,
  availableThemes,
  onChange
}: PreferencesProps) {
  return (
    <div>
      <select 
        value={theme}
        onChange={(e) => onChange('theme', e.target.value)}
      >
        {availableThemes.map(t => (
          <option key={t} value={t}>{t}</option>
        ))}
      </select>
      <label>
        <input
          type="checkbox"
          checked={notifications}
          onChange={(e) => onChange('notifications', e.target.checked)}
        />
        Enable notifications
      </label>
    </div>
  )
}

// Compose form from focused sections
function UserForm({ userId }: { userId: string }) {
  const { user, updateUser } = useUser(userId)
  const [formData, setFormData] = useState(user)
  
  const handleFieldChange = (field: string, value: any) => {
    setFormData(prev => ({
      ...prev,
      [field]: value
    }))
  }
  
  const handleSubmit = () => {
    updateUser(formData)
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <BasicInfoSection
        name={formData.name}
        email={formData.email}
        onChange={handleFieldChange}
      />
      <ProfileSection
        bio={formData.profile.bio}
        location={formData.profile.location}
        maxBioLength={500}
        onChange={handleFieldChange}
      />
      <PreferencesSection
        theme={formData.preferences.theme}
        notifications={formData.preferences.notifications}
        availableThemes={['light', 'dark', 'auto']}
        onChange={handleFieldChange}
      />
      <button type="submit">Save</button>
    </form>
  )
}

// Use custom hooks to provide focused interfaces
function useUserDisplay(userId: string) {
  const { user } = useUser(userId)
  
  // Return only display-relevant data
  return {
    name: user?.name,
    avatar: user?.profile.avatar,
    role: user?.role
  }
}

function useUserActions(userId: string) {
  const { updateUser, deleteUser } = useUser(userId)
  
  // Return only action-relevant functions
  return {
    updateProfile: (profile: Partial<User['profile']>) => 
      updateUser({ profile }),
    updatePreferences: (preferences: Partial<User['preferences']>) =>
      updateUser({ preferences }),
    deleteAccount: () => deleteUser()
  }
}
```

## When to Apply ISP in React

### Apply ISP for

* Components receiving large objects but using few fields
* Reusable components in different contexts
* Components with many optional props
* Deep component trees with prop drilling
* Component libraries and design systems

### Balance with Practicality

* Small, simple components can receive full objects
* Tightly coupled components can share interfaces
* Performance-critical paths may bundle props

## Anti-patterns to Avoid

1. **God objects as props**: Passing entire state objects everywhere
2. **Prop drilling**: Passing props through components that don't use them
3. **Kitchen sink interfaces**: Components with 10+ props
4. **Unclear dependencies**: Can't tell what data component actually needs

## React-Specific ISP Techniques

1. **Destructure at call site** to show what's used
2. **Custom hooks** to provide focused data subsets
3. **Component composition** over prop passing
4. **Context splitting** for different concerns
5. **Prop spreading** with pick/omit utilities

## Key Takeaways

* Components should **only receive props they use**
* **Split fat interfaces** into focused ones
* Use **composition** to combine capabilities
* Create **custom hooks** for data selection
* Keep prop interfaces **lean and specific**

## Related Best Practices

For component composition and prop patterns, see
👉 [best-practices.md](../best-practices.md)
