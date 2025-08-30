# React Best Practices

This document collects general coding practices that cut across multiple
principles. It complements the [Clean Code-focused docs](./clean-code/) (e.g.
DIP, ISP, DRY) with conventions for TypeScript interfaces, error handling, testing, and
component management.

---

## TypeScript Interfaces

### Discriminated Unions vs Optional Fields

* **Discriminated unions** for variants with different shapes:

  ```typescript
  // GOOD - Type-safe, no optional fields
  type LoadingState = { status: 'loading' }
  type ErrorState = { status: 'error'; error: Error }
  type SuccessState<T> = { status: 'success'; data: T }
  
  type AsyncState<T> = LoadingState | ErrorState | SuccessState<T>
  
  function handleState<T>(state: AsyncState<T>) {
    switch (state.status) {
      case 'loading':
        return <Spinner />
      case 'error':
        return <ErrorMessage error={state.error} />
      case 'success':
        return <DataDisplay data={state.data} />
    }
  }
  ```

* **Avoid optional fields** when types are mutually exclusive:

  ```typescript
  // BAD - Unclear which fields are valid when
  type AsyncState<T> = {
    status: 'loading' | 'error' | 'success'
    data?: T
    error?: Error
  }
  ```

### Where to Define Interfaces

* **Define interfaces near consumers** (components/hooks that use them).
  This keeps them minimal and tailored to actual needs.

  ```typescript
  // UserList.tsx - Consumer defines what it needs
  interface UserService {
    getUsers(): Promise<User[]>
    deleteUser(id: string): Promise<void>
  }
  
  interface UserListProps {
    userService: UserService
  }
  ```

* **Services/adapters** should export concrete implementations only.
  They satisfy consumer-defined interfaces implicitly.
* **Exception:** Truly generic interfaces (e.g. `Observable`) can live in shared types.

### Interface Guidelines

* Keep interfaces **focused** on single capabilities
* Use **intersection types** to compose larger contracts:

  ```typescript
  type UserReader = {
    getUser(id: string): Promise<User>
  }
  
  type UserWriter = {
    saveUser(user: User): Promise<void>
  }
  
  type UserRepository = UserReader & UserWriter
  ```

---

## Error Handling

### Error Boundaries

* Use error boundaries to catch React rendering errors:

  ```typescript
  class ErrorBoundary extends Component<Props, State> {
    static getDerivedStateFromError(error: Error): State {
      return { hasError: true, error }
    }
    
    componentDidCatch(error: Error, info: ErrorInfo) {
      logErrorToService(error, info)
    }
    
    render() {
      if (this.state.hasError) {
        return <ErrorFallback error={this.state.error} />
      }
      return this.props.children
    }
  }
  ```

### Domain Errors

* Create typed domain errors for business logic:

  ```typescript
  class UserNotFoundError extends Error {
    constructor(public readonly userId: string) {
      super(`User ${userId} not found`)
      this.name = 'UserNotFoundError'
    }
  }
  
  class ValidationError extends Error {
    constructor(public readonly fields: Record<string, string>) {
      super('Validation failed')
      this.name = 'ValidationError'
    }
  }
  ```

### Map API Errors → Domain Errors

* Don't leak HTTP/GraphQL details to components:

  ```typescript
  async function fetchUser(id: string): Promise<User> {
    const response = await fetch(`/api/users/${id}`)
    
    if (response.status === 404) {
      throw new UserNotFoundError(id)
    }
    
    if (!response.ok) {
      throw new Error(`Failed to fetch user: ${response.statusText}`)
    }
    
    return response.json()
  }
  ```

---

## Testing Practices

### Component Testing

* Test behavior via user interactions, not implementation:

  ```typescript
  // UserProfile.test.tsx
  import { render, screen, waitFor } from '@testing-library/react'
  import userEvent from '@testing-library/user-event'
  
  test('displays user after loading', async () => {
    const mockService = {
      getUser: jest.fn().mockResolvedValue({
        id: '1',
        name: 'Alice',
        email: 'alice@example.com'
      })
    }
    
    render(<UserProfile userId="1" userService={mockService} />)
    
    expect(screen.getByText(/loading/i)).toBeInTheDocument()
    
    await waitFor(() => {
      expect(screen.getByText('Alice')).toBeInTheDocument()
    })
    
    expect(mockService.getUser).toHaveBeenCalledWith('1')
  })
  ```

### Hook Testing

* Test hooks in isolation with renderHook:

  ```typescript
  import { renderHook, act } from '@testing-library/react'
  
  test('useCounter increments value', () => {
    const { result } = renderHook(() => useCounter())
    
    expect(result.current.count).toBe(0)
    
    act(() => {
      result.current.increment()
    })
    
    expect(result.current.count).toBe(1)
  })
  ```

### Dependency Injection for Testing

* **NO jest.mock()** - Use dependency injection instead:

  ```typescript
  // BAD - Using jest.mock
  jest.mock('../services/api')
  
  // GOOD - Dependency injection
  function useUsers(userService: UserService) {
    const [users, setUsers] = useState<User[]>([])
    
    useEffect(() => {
      userService.getUsers().then(setUsers)
    }, [userService])
    
    return users
  }
  
  // In tests, pass a stub
  const stubService: UserService = {
    getUsers: jest.fn().mockResolvedValue(mockUsers)
  }
  ```

### Stub Implementations

* Create simple stubs for testing:

  ```typescript
  class StubUserService implements UserService {
    private users = new Map<string, User>()
    
    async getUser(id: string): Promise<User> {
      const user = this.users.get(id)
      if (!user) throw new UserNotFoundError(id)
      return user
    }
    
    async saveUser(user: User): Promise<void> {
      this.users.set(user.id, user)
    }
    
    // Test helper
    addUser(user: User): void {
      this.users.set(user.id, user)
    }
  }
  ```

### Testing Re-renders

* Verify components don't re-render unnecessarily:

  ```typescript
  test('memoizes expensive computation', () => {
    const renderSpy = jest.fn()
    
    function ExpensiveComponent({ value }: { value: number }) {
      renderSpy()
      const result = useMemo(() => expensiveCalculation(value), [value])
      return <div>{result}</div>
    }
    
    const { rerender } = render(<ExpensiveComponent value={1} />)
    expect(renderSpy).toHaveBeenCalledTimes(1)
    
    // Same prop - should not re-render
    rerender(<ExpensiveComponent value={1} />)
    expect(renderSpy).toHaveBeenCalledTimes(1)
    
    // Different prop - should re-render
    rerender(<ExpensiveComponent value={2} />)
    expect(renderSpy).toHaveBeenCalledTimes(2)
  })
  ```

---

## Component Structure

A clean component structure helps enforce separation of concerns and maintainability.

### Suggested Organization

```txt
/components
  /common           # Shared UI components
    Button.tsx
    Input.tsx
    Modal.tsx
  /features
    /users          # Feature-specific components
      UserList.tsx
      UserProfile.tsx
      UserForm.tsx
/hooks              # Custom hooks
  useAuth.ts
  useApi.ts
  useDebounce.ts
/services           # Business logic & API calls
  userService.ts
  authService.ts
/types              # Shared TypeScript types
  user.ts
  api.ts
```

### Guidelines

1. **Presentation Components**

   * Pure functions that render UI based on props
   * No business logic or side effects
   * Example:

     ```typescript
     interface UserCardProps {
       user: User
       onEdit: (user: User) => void
       onDelete: (id: string) => void
     }
     
     function UserCard({ user, onEdit, onDelete }: UserCardProps) {
       return (
         <div className="user-card">
           <h3>{user.name}</h3>
           <p>{user.email}</p>
           <button onClick={() => onEdit(user)}>Edit</button>
           <button onClick={() => onDelete(user.id)}>Delete</button>
         </div>
       )
     }
     ```

2. **Container Components**

   * Handle data fetching and business logic via hooks
   * Pass data and callbacks to presentation components
   * Example:

     ```typescript
     function UserListContainer({ userService }: { userService: UserService }) {
       const users = useUsers(userService)
       const { deleteUser } = useUserActions(userService)
       
       return <UserList users={users} onDelete={deleteUser} />
     }
     ```

3. **Custom Hooks**

   * Encapsulate all business logic and side effects
   * Return data and functions for components to use
   * Accept dependencies for testability:

     ```typescript
     function useUsers(service: UserService) {
       const [users, setUsers] = useState<User[]>([])
       const [loading, setLoading] = useState(true)
       const [error, setError] = useState<Error | null>(null)
       
       useEffect(() => {
         service.getUsers()
           .then(setUsers)
           .catch(setError)
           .finally(() => setLoading(false))
       }, [service])
       
       return { users, loading, error }
     }
     ```

4. **Component Composition**

   * Use composition over inheritance
   * Leverage children and render props:

     ```typescript
     function Layout({ header, sidebar, children }: LayoutProps) {
       return (
         <div className="layout">
           <header>{header}</header>
           <aside>{sidebar}</aside>
           <main>{children}</main>
         </div>
       )
     }
     ```

---

## Dependency Injection

### Context for Cross-cutting Concerns

* Use React Context for app-wide dependencies:

  ```typescript
  const ServiceContext = createContext<Services | null>(null)
  
  function ServiceProvider({ children, services }: ServiceProviderProps) {
    return (
      <ServiceContext.Provider value={services}>
        {children}
      </ServiceContext.Provider>
    )
  }
  
  function useServices(): Services {
    const services = useContext(ServiceContext)
    if (!services) {
      throw new Error('useServices must be used within ServiceProvider')
    }
    return services
  }
  ```

### Props for Component-specific Dependencies

* Pass dependencies directly when scope is limited:

  ```typescript
  interface UserFormProps {
    userService: UserService
    validator: UserValidator
    onSuccess: (user: User) => void
  }
  
  function UserForm({ userService, validator, onSuccess }: UserFormProps) {
    const handleSubmit = async (data: UserData) => {
      const errors = validator.validate(data)
      if (errors) return
      
      const user = await userService.createUser(data)
      onSuccess(user)
    }
    
    // ...
  }
  ```

### Factory Functions for Complex Dependencies

* Create factories for configurable dependencies:

  ```typescript
  function createApiService(config: ApiConfig): ApiService {
    return {
      async get<T>(path: string): Promise<T> {
        const response = await fetch(`${config.baseUrl}${path}`, {
          headers: config.headers
        })
        return response.json()
      },
      // ...
    }
  }
  ```

---

## Performance Optimization

### Memoization

* Use `useMemo` for expensive computations:

  ```typescript
  function DataTable({ data, filters }: DataTableProps) {
    const filteredData = useMemo(
      () => applyFilters(data, filters),
      [data, filters]
    )
    
    return <Table data={filteredData} />
  }
  ```

* Use `React.memo` for pure components:

  ```typescript
  const UserCard = React.memo(function UserCard({ user }: UserCardProps) {
    return <div>{user.name}</div>
  })
  ```

### Code Splitting

* Split large features into separate bundles:

  ```typescript
  const UserManagement = lazy(() => import('./features/users/UserManagement'))
  
  function App() {
    return (
      <Suspense fallback={<Loading />}>
        <Routes>
          <Route path="/users/*" element={<UserManagement />} />
        </Routes>
      </Suspense>
    )
  }
  ```

### Virtual Lists

* Use virtualization for long lists:

  ```typescript
  import { FixedSizeList } from 'react-window'
  
  function UserList({ users }: { users: User[] }) {
    const Row = ({ index, style }: RowProps) => (
      <div style={style}>
        <UserCard user={users[index]} />
      </div>
    )
    
    return (
      <FixedSizeList
        height={600}
        itemCount={users.length}
        itemSize={100}
        width="100%"
      >
        {Row}
      </FixedSizeList>
    )
  }
  ```

---

## State Management

### Local State

* Keep state as close to where it's used as possible:

  ```typescript
  function SearchBar() {
    const [query, setQuery] = useState('')
    
    return (
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search..."
      />
    )
  }
  ```

### Lifting State

* Lift state only when multiple components need it:

  ```typescript
  function Parent() {
    const [selectedUser, setSelectedUser] = useState<User | null>(null)
    
    return (
      <>
        <UserList onSelect={setSelectedUser} />
        <UserDetails user={selectedUser} />
      </>
    )
  }
  ```

### Global State

* Use Context or state management libraries sparingly:

  ```typescript
  // Only for truly global state
  const AuthContext = createContext<AuthState | null>(null)
  
  function useAuth() {
    const auth = useContext(AuthContext)
    if (!auth) throw new Error('Auth not provided')
    return auth
  }
  ```

---

## Key Takeaways

* Use **discriminated unions** over types with optional fields
* Define **interfaces in consumers**; services expose implementations
* **No jest.mock()** - use dependency injection with stubs
* Encapsulate **business logic in hooks** with injected dependencies
* Keep **components focused** - separate presentation from logic
* Test **behavior, not implementation**
* **Memoize** expensive operations; virtualize long lists
* Keep **state local** unless sharing is necessary

---

## Further Reading

* [React Documentation](https://react.dev)
* [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
* [Testing Library Best Practices](https://testing-library.com/docs/guiding-principles)
* [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app)
