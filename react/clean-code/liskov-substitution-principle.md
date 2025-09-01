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

### BAD — Inconsistent Component Behavior (LSP violation)

> Component variants break expected behavior contracts.

```typescript
interface InputProps {
  value: string
  onChange: (value: string) => void
  disabled?: boolean
  placeholder?: string
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
      disabled={false} // VIOLATION: ignores provided disabled prop
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
  
  return (<input value={value} onChange={handleChange} />)
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
const TextInput: React.FC<Omit<TypedInputProps<string>, 'parse' | 'format'>> =
  (props) => (
  <TypedInput
    {...props}
    parse={(s) => s}
    format={(s) => s}
  />
)

const NumberInput: React.FC<Omit<TypedInputProps<number>, 'parse' | 'format'>> =
  (props) => (
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

const DateInput: React.FC<Omit<TypedInputProps<Date>, 'parse' | 'format'>> =
  (props) => (
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
5. **Ignoring controlled props**: Managing internal state that conflicts with
`value/checked/selected` or `disabled/readOnly`

## React-Specific LSP Techniques

1. **Explicit variant props** over implicit behavior changes
2. **Consistent callback signatures** across variants
3. **Clear error handling** contracts
4. **Generic components** with type parameters
5. **Stable base props + explicit extension**: keep core prop names/semantics
the same; avoid blind prop spreading that masks incompatibilities

## Key Takeaways

* Component variants must **honor parent contracts**
* Keep **behavior predictable** across implementations
* Make **format changes explicit** in the interface
* Use **generic components** for consistent behavior
* **Document deviations** when absolutely necessary

## Related Best Practices

For component interfaces and testing patterns, see
👉 [best-practices.md](../best-practices/best-practices.md)
