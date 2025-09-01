# Liskov Substitution Principle (LSP) in React

## Overview

The Liskov Substitution Principle states that child components should be
substitutable for their parent components without breaking functionality. In
React, this means components accepting the same props should behave
consistently, and component variants should honor the same contracts.

## Core Concept

In React, LSP means:

* Child components fully implement parent interfaces
* Component variants behave consistently
* Props contracts are honored across implementations
* Error states are handled uniformly
* Callbacks have consistent signatures

## Implementation Example

### Scaffolding for Examples

```typescript
// types.ts
interface ButtonProps {
  onClick: () => void
  disabled?: boolean
  children: React.ReactNode
  className?: string
}

interface InputProps {
  value: string
  onChange: (value: string) => void
  disabled?: boolean
  placeholder?: string
}

interface FormData {
  name: string
  email: string
}
```

### BAD — Inconsistent Component Behavior (LSP violation)

> Component variants break expected behavior contracts.

```typescript
// Base button interface
interface ActionButtonProps {
  onClick: () => void
  disabled?: boolean
  children: React.ReactNode
}

// VIOLATION: AsyncButton changes onClick signature
function AsyncButton({ 
  onClick,  // Expects () => Promise<void> but typed as () => void
  disabled,
  children 
}: ActionButtonProps) {
  const [loading, setLoading] = useState(false)
  
  const handleClick = async () => {
    setLoading(true)
    // VIOLATION: Assumes onClick returns a Promise
    await (onClick as any)()  // Type assertion hides the violation
    setLoading(false)
  }
  
  return (
    <button onClick={handleClick} disabled={disabled || loading}>
      {loading ? 'Loading...' : children}
    </button>
  )
}

// VIOLATION: ConfirmButton adds unexpected behavior
function ConfirmButton({ 
  onClick,
  disabled,
  children 
}: ActionButtonProps) {
  // VIOLATION: onClick might not be called, breaking expectations
  const handleClick = () => {
    if (window.confirm('Are you sure?')) {
      onClick()
    }
    // Silent failure if user cancels - caller has no way to know
  }
  
  return (
    <button onClick={handleClick} disabled={disabled}>
      {children}
    </button>
  )
}

// VIOLATION: Input components with inconsistent onChange behavior
function TextInput({ value, onChange }: InputProps) {
  return (
    <input
      value={value}
      // Direct value pass-through
      onChange={(e) => onChange(e.target.value)}
    />
  )
}

function NumericInput({ value, onChange }: InputProps) {
  return (
    <input
      type="number"
      value={value}
      // VIOLATION: Silently coerces/filters input
      onChange={(e) => {
        const num = parseInt(e.target.value) || 0
        onChange(String(num))  // Loses decimal points!
      }}
    />
  )
}

function FormattedInput({ value, onChange }: InputProps) {
  // VIOLATION: Changes value format without caller knowing
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const formatted = e.target.value.toUpperCase().replace(/\s/g, '-')
    onChange(formatted)
  }
  
  return <input value={value} onChange={handleChange} />
}

// Usage shows the violations
function MyForm() {
  const [formData, setFormData] = useState({ text: '', number: '' })
  
  // These inputs behave differently despite same interface
  return (
    <div>
      <TextInput 
        value={formData.text}
        onChange={(v) => setFormData({ ...formData, text: v })}
      />
      <NumericInput 
        value={formData.number}
        // Expecting string "3.14" but will get "3"
        onChange={(v) => setFormData({ ...formData, number: v })}
      />
    </div>
  )
}
```

### GOOD — Consistent Component Contracts (LSP compliant)

> Component variants maintain consistent behavior contracts.

```typescript
// Clear interface with explicit async support
interface ButtonAction {
  execute: () => void | Promise<void>
  canExecute?: () => boolean
}

interface SmartButtonProps {
  action: ButtonAction
  disabled?: boolean
  children: React.ReactNode
  variant?: 'default' | 'async' | 'confirm'
}

// Base button that all variants properly extend
function SmartButton({ 
  action,
  disabled,
  children,
  variant = 'default'
}: SmartButtonProps) {
  const [loading, setLoading] = useState(false)
  
  const handleClick = async () => {
    if (action.canExecute && !action.canExecute()) {
      return
    }
    
    if (variant === 'confirm') {
      if (!window.confirm('Are you sure?')) {
        return
      }
    }
    
    if (variant === 'async') {
      setLoading(true)
    }
    
    try {
      await action.execute()
    } finally {
      setLoading(false)
    }
  }
  
  const isDisabled = disabled || loading || 
    (action.canExecute && !action.canExecute())
  
  return (
    <button onClick={handleClick} disabled={isDisabled}>
      {loading ? 'Loading...' : children}
    </button>
  )
}

// Consistent input interface with explicit formatting
interface TypedInputProps<T> {
  value: T
  onChange: (value: T) => void
  parse: (raw: string) => T
  format: (value: T) => string
  validate?: (value: T) => string | null
  disabled?: boolean
  placeholder?: string
}

function TypedInput<T>({ 
  value,
  onChange,
  parse,
  format,
  validate,
  disabled,
  placeholder
}: TypedInputProps<T>) {
  const [rawValue, setRawValue] = useState(() => format(value))
  const [error, setError] = useState<string | null>(null)
  
  useEffect(() => {
    setRawValue(format(value))
  }, [value, format])
  
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const raw = e.target.value
    setRawValue(raw)
    
    try {
      const parsed = parse(raw)
      const validationError = validate?.(parsed)
      
      if (validationError) {
        setError(validationError)
      } else {
        setError(null)
        onChange(parsed)
      }
    } catch {
      setError('Invalid input')
    }
  }
  
  return (
    <div className="typed-input">
      <input
        value={rawValue}
        onChange={handleChange}
        disabled={disabled}
        placeholder={placeholder}
      />
      {error && <span className="error">{error}</span>}
    </div>
  )
}

// Concrete implementations that honor the contract
const TextInput: React.FC<Omit<TypedInputProps<string>, 'parse' | 'format'>> = (props) => (
  <TypedInput
    {...props}
    parse={(s) => s}
    format={(s) => s}
  />
)

const NumberInput: React.FC<Omit<TypedInputProps<number>, 'parse' | 'format'>> = (props) => (
  <TypedInput
    {...props}
    parse={(s) => {
      const n = parseFloat(s)
      if (isNaN(n)) throw new Error('Not a number')
      return n
    }}
    format={(n) => String(n)}
  />
)

const DateInput: React.FC<Omit<TypedInputProps<Date>, 'parse' | 'format'>> = (props) => (
  <TypedInput
    {...props}
    parse={(s) => {
      const d = new Date(s)
      if (isNaN(d.getTime())) throw new Error('Invalid date')
      return d
    }}
    format={(d) => d.toISOString().split('T')[0]}
  />
)

// Component family with consistent behavior
interface ListItemProps<T> {
  item: T
  onSelect: (item: T) => void
  selected: boolean
}

// All list items honor the same selection contract
function UserListItem({ item, onSelect, selected }: ListItemProps<User>) {
  return (
    <div 
      className={`list-item ${selected ? 'selected' : ''}`}
      onClick={() => onSelect(item)}
    >
      <Avatar user={item} />
      <span>{item.name}</span>
    </div>
  )
}

function ProductListItem({ item, onSelect, selected }: ListItemProps<Product>) {
  return (
    <div 
      className={`list-item ${selected ? 'selected' : ''}`}
      onClick={() => onSelect(item)}
    >
      <ProductImage product={item} />
      <span>{item.title}</span>
      <span>{item.price}</span>
    </div>
  )
}

// Generic list that works with any compliant item component
function SelectableList<T>({ 
  items,
  ItemComponent,
  onSelectionChange
}: {
  items: T[]
  ItemComponent: React.FC<ListItemProps<T>>
  onSelectionChange: (selected: T[]) => void
}) {
  const [selected, setSelected] = useState<Set<T>>(new Set())
  
  const handleSelect = (item: T) => {
    const newSelected = new Set(selected)
    if (selected.has(item)) {
      newSelected.delete(item)
    } else {
      newSelected.add(item)
    }
    setSelected(newSelected)
    onSelectionChange(Array.from(newSelected))
  }
  
  return (
    <div className="selectable-list">
      {items.map((item, index) => (
        <ItemComponent
          key={index}
          item={item}
          onSelect={handleSelect}
          selected={selected.has(item)}
        />
      ))}
    </div>
  )
}
```

## Pragmatic Scope

In practice, focus LSP compliance on:

* Public component APIs that others depend on
* Component libraries and design systems
* Generic components meant for reuse
* Components with multiple implementations

## When to Apply LSP in React

### Apply LSP for

* Component libraries with variants
* Form input components
* List item components
* Modal/dialog variants
* Button families

### Relaxed Requirements for

* One-off internal components
* Prototypes and experiments
* Components with single implementation
* Private implementation details

## Anti-patterns to Avoid

1. **Silent behavior changes**: Modifying data without clear indication
2. **Inconsistent callbacks**: Different signatures for same prop name
3. **Hidden side effects**: Components doing more than interface suggests
4. **Type lies**: Using type assertions to hide incompatibilities

## React-Specific LSP Techniques

1. **Explicit variant props** over implicit behavior changes
2. **Consistent callback signatures** across variants
3. **Clear error handling** contracts
4. **Generic components** with type parameters
5. **Prop spreading** to maintain interface compatibility

## Key Takeaways

* Component variants must **honor parent contracts**
* Keep **behavior predictable** across implementations
* Make **format changes explicit** in the interface
* Use **generic components** for consistent behavior
* **Document deviations** when absolutely necessary

## Related Best Practices

For component interfaces and testing patterns, see
👉 [best-practices.md](../best-practices/best-practices.md)
