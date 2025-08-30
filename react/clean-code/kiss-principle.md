# KISS Principle (Keep It Simple, Stupid) in React

## Overview

The KISS principle advocates for simplicity in design and implementation. In
React, this means choosing the simplest solution that works, avoiding premature
optimization, and not over-engineering components. Simple code is easier to
understand, maintain, and debug.

## Core Concept

In React, KISS means:

* Start with the simplest implementation
* Avoid premature abstraction
* Use built-in React features before external libraries
* Keep component hierarchies flat when possible
* Choose clarity over cleverness

## Implementation Example

### Scaffolding for Examples

```typescript
// types.ts
interface Todo {
  id: string
  text: string
  completed: boolean
}

interface FilterOption {
  value: 'all' | 'active' | 'completed'
  label: string
}
```

### BAD — Over-Engineered (KISS violation)

> Unnecessary complexity for simple requirements.

```typescript
// VIOLATION: Over-engineered state management for simple counter
interface CounterState {
  value: number
  history: number[]
  future: number[]
  min?: number
  max?: number
}

type CounterAction = 
  | { type: 'INCREMENT'; payload?: number }
  | { type: 'DECREMENT'; payload?: number }
  | { type: 'SET'; payload: number }
  | { type: 'UNDO' }
  | { type: 'REDO' }
  | { type: 'RESET' }

function counterReducer(state: CounterState, action: CounterAction): CounterState {
  switch (action.type) {
    case 'INCREMENT': {
      const newValue = state.value + (action.payload ?? 1)
      if (state.max !== undefined && newValue > state.max) {
        return state
      }
      return {
        ...state,
        value: newValue,
        history: [...state.history, state.value],
        future: []
      }
    }
    case 'DECREMENT': {
      const newValue = state.value - (action.payload ?? 1)
      if (state.min !== undefined && newValue < state.min) {
        return state
      }
      return {
        ...state,
        value: newValue,
        history: [...state.history, state.value],
        future: []
      }
    }
    case 'UNDO': {
      if (state.history.length === 0) return state
      const previous = state.history[state.history.length - 1]
      return {
        ...state,
        value: previous,
        history: state.history.slice(0, -1),
        future: [state.value, ...state.future]
      }
    }
    case 'REDO': {
      if (state.future.length === 0) return state
      const next = state.future[0]
      return {
        ...state,
        value: next,
        history: [...state.history, state.value],
        future: state.future.slice(1)
      }
    }
    case 'SET':
      return {
        ...state,
        value: action.payload,
        history: [...state.history, state.value],
        future: []
      }
    case 'RESET':
      return {
        value: 0,
        history: [],
        future: [],
        min: state.min,
        max: state.max
      }
    default:
      return state
  }
}

// VIOLATION: Over-abstracted for a simple counter
function Counter() {
  const [state, dispatch] = useReducer(counterReducer, {
    value: 0,
    history: [],
    future: [],
    min: 0,
    max: 100
  })
  
  return (
    <div>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>-</button>
      <span>{state.value}</span>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>+</button>
      <button onClick={() => dispatch({ type: 'UNDO' })} disabled={state.history.length === 0}>
        Undo
      </button>
      <button onClick={() => dispatch({ type: 'REDO' })} disabled={state.future.length === 0}>
        Redo
      </button>
    </div>
  )
}

// VIOLATION: Over-engineered todo list with unnecessary abstractions
interface TodoStore {
  todos: Map<string, Todo>
  filters: Set<FilterOption['value']>
  sortOrder: 'asc' | 'desc'
  searchQuery: string
}

class TodoManager {
  private store: TodoStore
  private listeners: Set<() => void> = new Set()
  
  constructor() {
    this.store = {
      todos: new Map(),
      filters: new Set(['all']),
      sortOrder: 'asc',
      searchQuery: ''
    }
  }
  
  subscribe(listener: () => void) {
    this.listeners.add(listener)
    return () => this.listeners.delete(listener)
  }
  
  private notify() {
    this.listeners.forEach(listener => listener())
  }
  
  addTodo(text: string) {
    const id = crypto.randomUUID()
    this.store.todos.set(id, { id, text, completed: false })
    this.notify()
  }
  
  updateTodo(id: string, updates: Partial<Todo>) {
    const todo = this.store.todos.get(id)
    if (todo) {
      this.store.todos.set(id, { ...todo, ...updates })
      this.notify()
    }
  }
  
  getFilteredTodos(): Todo[] {
    const todos = Array.from(this.store.todos.values())
    
    // VIOLATION: Complex filtering for simple use case
    return todos
      .filter(todo => {
        if (this.store.searchQuery) {
          if (!todo.text.toLowerCase().includes(this.store.searchQuery.toLowerCase())) {
            return false
          }
        }
        
        if (this.store.filters.has('active') && todo.completed) {
          return false
        }
        
        if (this.store.filters.has('completed') && !todo.completed) {
          return false
        }
        
        return true
      })
      .sort((a, b) => {
        const order = this.store.sortOrder === 'asc' ? 1 : -1
        return a.text.localeCompare(b.text) * order
      })
  }
}

// VIOLATION: Factory pattern for simple config
interface ThemeConfig {
  primary: string
  secondary: string
  background: string
}

abstract class ThemeFactory {
  abstract createTheme(): ThemeConfig
}

class LightThemeFactory extends ThemeFactory {
  createTheme(): ThemeConfig {
    return {
      primary: '#007bff',
      secondary: '#6c757d',
      background: '#ffffff'
    }
  }
}

class DarkThemeFactory extends ThemeFactory {
  createTheme(): ThemeConfig {
    return {
      primary: '#0d6efd',
      secondary: '#6c757d',
      background: '#212529'
    }
  }
}

function useTheme(factory: ThemeFactory) {
  const [theme, setTheme] = useState(() => factory.createTheme())
  // Over-complicated for simple theme switching
  return theme
}
```

### GOOD — Simple and Clear (KISS compliant)

> Straightforward implementation that meets requirements.

```typescript
// Simple counter with just what's needed
function Counter() {
  const [count, setCount] = useState(0)
  
  return (
    <div>
      <button onClick={() => setCount(count - 1)}>-</button>
      <span>{count}</span>
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  )
}

// Simple todo list without over-engineering
function TodoList() {
  const [todos, setTodos] = useState<Todo[]>([])
  const [filter, setFilter] = useState<'all' | 'active' | 'completed'>('all')
  const [inputText, setInputText] = useState('')
  
  const addTodo = (e: React.FormEvent) => {
    e.preventDefault()
    if (!inputText.trim()) return
    
    setTodos([
      ...todos,
      {
        id: Date.now().toString(),
        text: inputText,
        completed: false
      }
    ])
    setInputText('')
  }
  
  const toggleTodo = (id: string) => {
    setTodos(todos.map(todo =>
      todo.id === id
        ? { ...todo, completed: !todo.completed }
        : todo
    ))
  }
  
  const deleteTodo = (id: string) => {
    setTodos(todos.filter(todo => todo.id !== id))
  }
  
  const filteredTodos = todos.filter(todo => {
    if (filter === 'active') return !todo.completed
    if (filter === 'completed') return todo.completed
    return true
  })
  
  return (
    <div>
      <form onSubmit={addTodo}>
        <input
          value={inputText}
          onChange={e => setInputText(e.target.value)}
          placeholder="Add todo..."
        />
        <button type="submit">Add</button>
      </form>
      
      <div>
        <button onClick={() => setFilter('all')}>All</button>
        <button onClick={() => setFilter('active')}>Active</button>
        <button onClick={() => setFilter('completed')}>Completed</button>
      </div>
      
      <ul>
        {filteredTodos.map(todo => (
          <li key={todo.id}>
            <input
              type="checkbox"
              checked={todo.completed}
              onChange={() => toggleTodo(todo.id)}
            />
            <span style={{ textDecoration: todo.completed ? 'line-through' : 'none' }}>
              {todo.text}
            </span>
            <button onClick={() => deleteTodo(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  )
}

// Simple theme without factory pattern
const themes = {
  light: {
    primary: '#007bff',
    secondary: '#6c757d',
    background: '#ffffff'
  },
  dark: {
    primary: '#0d6efd',
    secondary: '#6c757d',
    background: '#212529'
  }
}

function useTheme() {
  const [theme, setTheme] = useState<'light' | 'dark'>('light')
  
  const toggleTheme = () => {
    setTheme(theme === 'light' ? 'dark' : 'light')
  }
  
  return {
    colors: themes[theme],
    theme,
    toggleTheme
  }
}

// Simple form without over-abstraction
function ContactForm() {
  const [name, setName] = useState('')
  const [email, setEmail] = useState('')
  const [message, setMessage] = useState('')
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    
    // Simple validation
    if (!name || !email || !message) {
      alert('Please fill all fields')
      return
    }
    
    // Simple email check
    if (!email.includes('@')) {
      alert('Please enter a valid email')
      return
    }
    
    // Submit
    try {
      await submitForm({ name, email, message })
      alert('Form submitted!')
      setName('')
      setEmail('')
      setMessage('')
    } catch (error) {
      alert('Error submitting form')
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        value={name}
        onChange={e => setName(e.target.value)}
        placeholder="Name"
      />
      <input
        value={email}
        onChange={e => setEmail(e.target.value)}
        placeholder="Email"
      />
      <textarea
        value={message}
        onChange={e => setMessage(e.target.value)}
        placeholder="Message"
      />
      <button type="submit">Send</button>
    </form>
  )
}

// Simple data fetching without over-engineering
function UserList() {
  const [users, setUsers] = useState<User[]>([])
  const [loading, setLoading] = useState(true)
  
  useEffect(() => {
    fetch('/api/users')
      .then(res => res.json())
      .then(setUsers)
      .catch(console.error)
      .finally(() => setLoading(false))
  }, [])
  
  if (loading) return <div>Loading...</div>
  
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  )
}

// Simple modal without complex state management
function Modal({ isOpen, onClose, children }: {
  isOpen: boolean
  onClose: () => void
  children: React.ReactNode
}) {
  if (!isOpen) return null
  
  return (
    <div className="modal-backdrop" onClick={onClose}>
      <div className="modal-content" onClick={e => e.stopPropagation()}>
        <button onClick={onClose}>×</button>
        {children}
      </div>
    </div>
  )
}

// Simple pagination without over-abstraction
function Pagination({ currentPage, totalPages, onPageChange }: {
  currentPage: number
  totalPages: number
  onPageChange: (page: number) => void
}) {
  return (
    <div>
      <button
        onClick={() => onPageChange(currentPage - 1)}
        disabled={currentPage === 1}
      >
        Previous
      </button>
      <span>{currentPage} / {totalPages}</span>
      <button
        onClick={() => onPageChange(currentPage + 1)}
        disabled={currentPage === totalPages}
      >
        Next
      </button>
    </div>
  )
}
```

## When to Apply KISS in React

### Keep Simple for

* Prototypes and MVPs
* Internal tools
* Components with single responsibility
* Straightforward CRUD operations
* Simple state management

### Consider Complexity When

* Building reusable libraries
* Performance is critical
* Complex business requirements
* Multiple team collaboration
* Long-term maintenance is crucial

## Anti-patterns to Avoid

1. **Premature optimization**: Complex solutions for simple problems
2. **Over-abstraction**: Too many layers of indirection
3. **Pattern overuse**: Applying patterns where not needed
4. **Library addiction**: Using libraries for trivial tasks

## React-Specific KISS Techniques

1. Start with **useState** before reaching for reducers
2. Use **built-in browser APIs** before external libraries
3. Keep **component trees shallow**
4. Use **inline styles** or CSS classes before CSS-in-JS
5. Write **inline handlers** before extracting them

## Key Takeaways

* **Start simple**, refactor when needed
* Avoid **premature abstraction**
* Use **built-in features** first
* Choose **clarity over cleverness**
* **Simple code** is maintainable code

## Related Best Practices

For pragmatic approaches and patterns, see
👉 [best-practices.md](../best-practices.md)
