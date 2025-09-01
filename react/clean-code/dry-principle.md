# DRY Principle (Don't Repeat Yourself) in React

## Overview

The DRY principle states that every piece of knowledge should have a single,
unambiguous representation in the system. In React, this means centralizing
business logic, validation rules, and UI patterns to avoid duplication and
ensure consistency across your application.

## Core Concept

In React, DRY means:

* Extract common UI patterns into reusable components
* Use custom hooks for shared logic

The above together with the other clean code and best practices will get
you to a clean codebase. Using libraries for data loading, forms etc
will get you a far way already.

## Implementation Example

### Scaffolding for Examples

```typescript
// types.ts
interface FormField {
  value: string
  error?: string
  touched?: boolean
}

interface User {
  id: string
  email: string
  age: number
  role: 'admin' | 'user' | 'guest'
}

interface ValidationRule {
  test: (value: any) => boolean
  message: string
}
```

### BAD — Duplicated Logic (DRY violation)

> Business rules and validation logic scattered across components.

```typescript
// VIOLATION: Email validation duplicated in multiple places
function LoginForm() {
  const [email, setEmail] = useState('')
  const [emailError, setEmailError] = useState('')
  
  const validateEmail = (value: string) => {
    // VIOLATION: Duplicate validation logic
    if (!value) {
      setEmailError('Email is required')
      return false
    }
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      setEmailError('Invalid email format')
      return false
    }
    setEmailError('')
    return true
  }
  
  return (
    <form>
      <input value={email} onChange={e => setEmail(e.target.value)} />
      {emailError && <span>{emailError}</span>}
    </form>
  )
}

function SignupForm() {
  const [email, setEmail] = useState('')
  const [emailError, setEmailError] = useState('')
  
  // VIOLATION: Same validation logic repeated
  const handleEmailChange = (value: string) => {
    setEmail(value)
    if (!value) {
      setEmailError('Email is required')
    } else if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value)) {
      setEmailError('Invalid email format')
    } else {
      setEmailError('')
    }
  }
  
  return (
    <form>
      <input value={email} onChange={e => handleEmailChange(e.target.value)} />
      {emailError && <span>{emailError}</span>}
    </form>
  )
}

function UserProfile() {
  const [email, setEmail] = useState('')
  
  const saveProfile = () => {
    // VIOLATION: Same validation logic again
    if (!email) {
      alert('Email is required')
      return
    }
    if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
      alert('Invalid email format')
      return
    }
    // save logic
  }
  
  return <div>{/* form */}</div>
}

// VIOLATION: Permission checks duplicated
function AdminPanel() {
  const user = useUser()
  
  // VIOLATION: Role check logic duplicated
  if (user.role !== 'admin') {
    return <div>Access denied</div>
  }
  
  return <div>Admin content</div>
}

function SettingsPage() {
  const user = useUser()
  
  // VIOLATION: Same role check logic
  const canEditSettings = user.role === 'admin' || user.role === 'user'
  
  if (!canEditSettings) {
    return <div>Access denied</div>
  }
  
  return <div>Settings</div>
}

// VIOLATION: Date formatting duplicated
function MessageItem({ message }: { message: Message }) {
  const formatDate = (date: Date) => {
    // VIOLATION: Date formatting logic repeated
    const now = new Date()
    const diff = now.getTime() - date.getTime()
    const hours = Math.floor(diff / (1000 * 60 * 60))
    
    if (hours < 1) return 'Just now'
    if (hours < 24) return `${hours}h ago`
    return date.toLocaleDateString()
  }
  
  return <div>{formatDate(message.createdAt)}</div>
}

function CommentItem({ comment }: { comment: Comment }) {
  // VIOLATION: Same date formatting logic
  const getTimeAgo = (date: Date) => {
    const now = new Date()
    const diff = now.getTime() - date.getTime()
    const hours = Math.floor(diff / (1000 * 60 * 60))
    
    if (hours < 1) return 'Just now'
    if (hours < 24) return `${hours}h ago`
    return date.toLocaleDateString()
  }
  
  return <div>{getTimeAgo(comment.postedAt)}</div>
}
```

### GOOD — Centralized Logic (DRY compliant)

> Single source of truth for business rules and validation.

```typescript
// Centralized validation rules
class ValidationRules {
  static email: ValidationRule[] = [
    {
      test: (value: string) => !!value,
      message: 'Email is required'
    },
    {
      test: (value: string) => /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value),
      message: 'Invalid email format'
    }
  ]
  
  static age: ValidationRule[] = [
    {
      test: (value: number) => value >= 0,
      message: 'Age must be positive'
    },
    {
      test: (value: number) => value <= 120,
      message: 'Age must be realistic'
    }
  ]
  
  static password: ValidationRule[] = [
    {
      test: (value: string) => value.length >= 8,
      message: 'Password must be at least 8 characters'
    },
    {
      test: (value: string) => /[A-Z]/.test(value),
      message: 'Password must contain uppercase letter'
    },
    {
      test: (value: string) => /[0-9]/.test(value),
      message: 'Password must contain number'
    }
  ]
}

// Centralized validation logic
function validate(value: any, rules: ValidationRule[]): string | null {
  for (const rule of rules) {
    if (!rule.test(value)) {
      return rule.message
    }
  }
  return null
}

// Reusable validation hook
function useValidation<T extends Record<string, any>>(
  initialValues: T,
  validationSchema: Record<keyof T, ValidationRule[]>
) {
  const [values, setValues] = useState(initialValues)
  const [errors, setErrors] = useState<Record<keyof T, string>>({} as any)
  const [touched, setTouched] = useState<Record<keyof T, boolean>>({} as any)
  
  const setValue = (field: keyof T, value: any) => {
    setValues(prev => ({ ...prev, [field]: value }))
    
    // Validate on change
    const error = validate(value, validationSchema[field])
    setErrors(prev => ({ ...prev, [field]: error }))
    setTouched(prev => ({ ...prev, [field]: true }))
  }
  
  const validateAll = (): boolean => {
    const newErrors = {} as Record<keyof T, string>
    let isValid = true
    
    for (const field in validationSchema) {
      const error = validate(values[field], validationSchema[field])
      if (error) {
        newErrors[field] = error
        isValid = false
      }
    }
    
    setErrors(newErrors)
    setTouched(Object.keys(validationSchema).reduce((acc, key) => ({
      ...acc,
      [key]: true
    }), {} as Record<keyof T, boolean>))
    
    return isValid
  }
  
  return { values, errors, touched, setValue, validateAll }
}

// Use centralized validation
function LoginForm() {
  const form = useValidation(
    { email: '', password: '' },
    {
      email: ValidationRules.email,
      password: ValidationRules.password
    }
  )
  
  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault()
    if (form.validateAll()) {
      // Submit logic
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={form.values.email}
        onChange={e => form.setValue('email', e.target.value)}
      />
      {form.touched.email && form.errors.email && (
        <span>{form.errors.email}</span>
      )}
      <input
        type="password"
        value={form.values.password}
        onChange={e => form.setValue('password', e.target.value)}
      />
      {form.touched.password && form.errors.password && (
        <span>{form.errors.password}</span>
      )}
      <button type="submit">Login</button>
    </form>
  )
}

// Centralized permission logic
class Permissions {
  static canAccessAdmin(user: User): boolean {
    return user.role === 'admin'
  }
  
  static canEditSettings(user: User): boolean {
    return user.role === 'admin' || user.role === 'user'
  }
  
  static canViewContent(user: User): boolean {
    return ['admin', 'user', 'guest'].includes(user.role)
  }
  
  static canModerate(user: User): boolean {
    return user.role === 'admin'
  }
}

// Reusable permission hook
function usePermission(check: (user: User) => boolean) {
  const user = useUser()
  return check(user)
}

// Use centralized permissions
function AdminPanel() {
  const hasAccess = usePermission(Permissions.canAccessAdmin)
  
  if (!hasAccess) {
    return <AccessDenied />
  }
  
  return <div>Admin content</div>
}

function SettingsPage() {
  const canEdit = usePermission(Permissions.canEditSettings)
  
  if (!canEdit) {
    return <AccessDenied />
  }
  
  return <div>Settings</div>
}

// Centralized date formatting
class DateFormatter {
  static timeAgo(date: Date): string {
    const now = new Date()
    const diff = now.getTime() - date.getTime()
    const seconds = Math.floor(diff / 1000)
    const minutes = Math.floor(seconds / 60)
    const hours = Math.floor(minutes / 60)
    const days = Math.floor(hours / 24)
    
    if (seconds < 60) return 'Just now'
    if (minutes < 60) return `${minutes}m ago`
    if (hours < 24) return `${hours}h ago`
    if (days < 7) return `${days}d ago`
    return date.toLocaleDateString()
  }
  
  static formatDate(date: Date, format: 'short' | 'long' | 'iso'): string {
    switch (format) {
      case 'short':
        return date.toLocaleDateString()
      case 'long':
        return date.toLocaleDateString('en-US', {
          weekday: 'long',
          year: 'numeric',
          month: 'long',
          day: 'numeric'
        })
      case 'iso':
        return date.toISOString()
    }
  }
}

// Reusable time display component
function TimeAgo({ date }: { date: Date }) {
  const [timeAgo, setTimeAgo] = useState(() => DateFormatter.timeAgo(date))
  
  useEffect(() => {
    const interval = setInterval(() => {
      setTimeAgo(DateFormatter.timeAgo(date))
    }, 60000) // Update every minute
    
    return () => clearInterval(interval)
  }, [date])
  
  return <span>{timeAgo}</span>
}

// Use centralized formatting
function MessageItem({ message }: { message: Message }) {
  return (
    <div>
      <p>{message.content}</p>
      <TimeAgo date={message.createdAt} />
    </div>
  )
}

function CommentItem({ comment }: { comment: Comment }) {
  return (
    <div>
      <p>{comment.text}</p>
      <TimeAgo date={comment.postedAt} />
    </div>
  )
}

// Centralized API error handling
class ApiErrorHandler {
  static handle(error: any): string {
    if (error.response) {
      switch (error.response.status) {
        case 400:
          return 'Invalid request. Please check your input.'
        case 401:
          return 'Please log in to continue.'
        case 403:
          return 'You do not have permission to perform this action.'
        case 404:
          return 'The requested resource was not found.'
        case 500:
          return 'Server error. Please try again later.'
        default:
          return 'An unexpected error occurred.'
      }
    }
    
    if (error.request) {
      return 'Network error. Please check your connection.'
    }
    
    return error.message || 'An unknown error occurred.'
  }
}

// Reusable error handling hook
function useApiCall<T>(
  apiCall: () => Promise<T>
): {
  data: T | null
  loading: boolean
  error: string | null
  execute: () => Promise<void>
} {
  const [data, setData] = useState<T | null>(null)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)
  
  const execute = async () => {
    setLoading(true)
    setError(null)
    
    try {
      const result = await apiCall()
      setData(result)
    } catch (err) {
      setError(ApiErrorHandler.handle(err))
    } finally {
      setLoading(false)
    }
  }
  
  return { data, loading, error, execute }
}
```

## When to Apply DRY in React

### Apply DRY for

* Validation logic used in multiple forms
* Permission/authorization checks
* Data formatting and transformations
* API error handling
* Business rules and calculations
* Common UI patterns

### Balance with Clarity

* Some duplication for readability is acceptable
* Don't over-abstract simple patterns
* Consider maintenance cost vs. duplication cost

## Anti-patterns to Avoid

1. **Scattered validation**: Same rules in multiple components
2. **Duplicated formatting**: Date/number formatting everywhere
3. **Repeated conditionals**: Same business logic checks
4. **Copy-paste components**: Nearly identical components

## React-Specific DRY Techniques

1. **Custom hooks** for shared logic
2. **Utility classes** for business rules
3. **Component composition** for UI patterns
4. **Context providers** for cross-cutting concerns
5. **Higher-order components** for common behaviors

## Key Takeaways

* Create **single sources of truth** for business logic
* **Centralize validation** and formatting rules
* Use **custom hooks** for reusable stateful logic
* Extract **common patterns** into utilities
* Balance DRY with **code clarity**

## Related Best Practices

For reusable patterns and utilities, see
👉 [best-practices.md](../best-practices.md)
