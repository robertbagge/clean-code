# Single Responsibility Principle (SRP) in React

## Overview

The Single Responsibility Principle states that a component or hook should have
one reason to change. In React, this means each component should handle one
specific aspect of the UI, and each hook should encapsulate one specific piece
of business logic. This makes components more reusable, testable, and maintainable.

## Core Concept

In React, SRP means:

* Components handle either presentation OR logic, not both
* Hooks encapsulate a single piece of business logic
* Separate data fetching from data display
* Extract complex state management into custom hooks
* Keep validation, formatting, and API calls in dedicated modules

## Implementation Example

### Scaffolding for Examples

```typescript
// types.ts
interface User {
  id: string
  name: string
  email: string
  avatar?: string
  lastActive: Date
}

interface UserService {
  getUser(id: string): Promise<User>
  updateUser(id: string, data: Partial<User>): Promise<User>
}

// utils.ts
function formatDate(date: Date): string {
  return new Intl.DateTimeFormat('en-US').format(date)
}
```

### BAD — Multiple Responsibilities (SRP violation)

> Component handles data fetching, state management, validation, and presentation.

```typescript
function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)
  const [editing, setEditing] = useState(false)
  const [formData, setFormData] = useState({ name: '', email: '' })
  const [formErrors, setFormErrors] = useState<Record<string, string>>({})

  // VIOLATION: Data fetching mixed with component
  useEffect(() => {
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data)
        setFormData({ name: data.name, email: data.email })
      })
      .catch(err => setError(err.message))
      .finally(() => setLoading(false))
  }, [userId])

  // VIOLATION: Validation logic inside component
  const validateForm = () => {
    const errors: Record<string, string> = {}
    if (!formData.name) errors.name = 'Name is required'
    if (!formData.email) errors.email = 'Email is required'
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(formData.email)) {
      errors.email = 'Invalid email format'
    }
    setFormErrors(errors)
    return Object.keys(errors).length === 0
  }

  // VIOLATION: Save logic mixed with component
  const handleSave = async () => {
    if (!validateForm()) return
    
    try {
      const res = await fetch(`/api/users/${userId}`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(formData)
      })
      const updated = await res.json()
      setUser(updated)
      setEditing(false)
    } catch (err) {
      setError('Failed to save')
    }
  }

  // VIOLATION: Date formatting logic in render
  if (loading) return <div>Loading...</div>
  if (error) return <div>Error: {error}</div>
  if (!user) return null

  return (
    <div>
      {editing ? (
        <div>
          <input
            value={formData.name}
            onChange={e => setFormData({ ...formData, name: e.target.value })}
          />
          {formErrors.name && <span>{formErrors.name}</span>}
          <input
            value={formData.email}
            onChange={e => setFormData({ ...formData, email: e.target.value })}
          />
          {formErrors.email && <span>{formErrors.email}</span>}
          <button onClick={handleSave}>Save</button>
        </div>
      ) : (
        <div>
          <h1>{user.name}</h1>
          <p>{user.email}</p>
          <p>Last active: {new Date(user.lastActive).toLocaleDateString()}</p>
          <button onClick={() => setEditing(true)}>Edit</button>
        </div>
      )}
    </div>
  )
}
```

### GOOD — Single Responsibility (SRP compliant)

> Each component/hook has one clear responsibility.

```typescript
// Validation logic extracted
class UserValidator {
  validateEmail(email: string): string | null {
    if (!email) return 'Email is required'
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      return 'Invalid email format'
    }
    return null
  }

  validateName(name: string): string | null {
    if (!name) return 'Name is required'
    if (name.length < 2) return 'Name must be at least 2 characters'
    return null
  }
}

// Data fetching hook (single responsibility: user data management)
function useUser(userId: string, service: UserService) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    service.getUser(userId)
      .then(setUser)
      .catch(setError)
      .finally(() => setLoading(false))
  }, [userId, service])

  const updateUser = async (data: Partial<User>) => {
    const updated = await service.updateUser(userId, data)
    setUser(updated)
    return updated
  }

  return { user, loading, error, updateUser }
}

// Form management hook (single responsibility: form state)
function useUserForm(initialValues: { name: string; email: string }) {
  const [values, setValues] = useState(initialValues)
  const [errors, setErrors] = useState<Record<string, string>>({})
  const validator = new UserValidator()

  const setValue = (field: string, value: string) => {
    setValues(prev => ({ ...prev, [field]: value }))
    // Clear error when user starts typing
    setErrors(prev => ({ ...prev, [field]: '' }))
  }

  const validate = () => {
    const newErrors: Record<string, string> = {}
    const nameError = validator.validateName(values.name)
    const emailError = validator.validateEmail(values.email)
    
    if (nameError) newErrors.name = nameError
    if (emailError) newErrors.email = emailError
    
    setErrors(newErrors)
    return Object.keys(newErrors).length === 0
  }

  return { values, errors, setValue, validate }
}

// Display component (single responsibility: showing user info)
function UserDisplay({ user, onEdit }: { 
  user: User
  onEdit: () => void 
}) {
  return (
    <div className="user-display">
      <h1>{user.name}</h1>
      <p>{user.email}</p>
      <p>Last active: {formatDate(user.lastActive)}</p>
      <button onClick={onEdit}>Edit</button>
    </div>
  )
}

// Edit form component (single responsibility: editing UI)
function UserEditForm({ 
  user,
  onSave,
  onCancel 
}: {
  user: User
  onSave: (data: Partial<User>) => void
  onCancel: () => void
}) {
  const form = useUserForm({ name: user.name, email: user.email })

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()
    if (form.validate()) {
      onSave(form.values)
    }
  }

  return (
    <form onSubmit={handleSubmit} className="user-form">
      <div>
        <input
          value={form.values.name}
          onChange={e => form.setValue('name', e.target.value)}
          placeholder="Name"
        />
        {form.errors.name && <span className="error">{form.errors.name}</span>}
      </div>
      <div>
        <input
          value={form.values.email}
          onChange={e => form.setValue('email', e.target.value)}
          placeholder="Email"
        />
        {form.errors.email && <span className="error">{form.errors.email}</span>}
      </div>
      <button type="submit">Save</button>
      <button type="button" onClick={onCancel}>Cancel</button>
    </form>
  )
}

// Container component (single responsibility: orchestration)
function UserProfile({ userId, userService }: { 
  userId: string
  userService: UserService 
}) {
  const { user, loading, error, updateUser } = useUser(userId, userService)
  const [editing, setEditing] = useState(false)

  if (loading) return <LoadingSpinner />
  if (error) return <ErrorMessage error={error} />
  if (!user) return <EmptyState message="User not found" />

  const handleSave = async (data: Partial<User>) => {
    await updateUser(data)
    setEditing(false)
  }

  return editing ? (
    <UserEditForm 
      user={user} 
      onSave={handleSave}
      onCancel={() => setEditing(false)}
    />
  ) : (
    <UserDisplay 
      user={user} 
      onEdit={() => setEditing(true)}
    />
  )
}
```

## When to Apply SRP in React

### Apply SRP for

* Components mixing data fetching with presentation
* Hooks handling multiple unrelated state pieces
* Components with both form logic and display logic
* Business logic mixed with UI components
* Components that are hard to test in isolation

### Acceptable Coupling

* Simple controlled inputs can handle their own state
* Small components can combine related presentation logic
* Container components can orchestrate multiple concerns

## Anti-patterns to Avoid

1. **God components**: Components doing everything
2. **Business logic in components**: Calculations, validations in render
3. **Mixed concerns in hooks**: One hook managing unrelated state
4. **Inline complex logic**: Complex operations directly in JSX

## React-Specific SRP Techniques

1. **Custom hooks** for business logic
2. **Presentation/Container** component split
3. **Utility functions** for formatting/validation
4. **Service classes** for API interactions
5. **Separate error boundary** components

## Key Takeaways

* Each component should have **one reason to change**
* Extract **business logic into hooks**
* Keep **presentation components pure**
* Use **composition** to combine single-purpose components
* **Test each piece** in isolation

## Related Best Practices

For component structure, testing patterns, and state management, see
👉 [best-practices.md](../best-practices/best-practices.md)
