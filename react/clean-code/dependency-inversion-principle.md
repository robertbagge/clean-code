# Dependency Inversion Principle (DIP) in React

## Overview

The Dependency Inversion Principle states that high-level components should not
depend on low-level implementations; both should depend on abstractions. In
React, this is achieved through dependency injection via props and context,
allowing components to depend on interfaces rather than concrete implementations.
This makes components more testable, portable, and maintainable.

## Core Concept

In React, DIP means:

* Components depend on interfaces, not implementations
* Inject services via props or context
* Hooks declare the contracts they need
* Keep API/database details out of components
* Use dependency injection for testability

## Implementation Example

### Scaffolding for Examples

```typescript
// types.ts
interface User {
  id: string
  name: string
  email: string
  createdAt: Date
}

interface AuthToken {
  token: string
  expiresAt: Date
}

class UserNotFoundError extends Error {
  constructor(public userId: string) {
    super(`User ${userId} not found`)
  }
}
```

### BAD — Direct Implementation Dependency (DIP violation)

> Components directly depend on concrete API implementations.

```typescript
import axios from 'axios'

// VIOLATION: Hook directly depends on axios and API details
function useUser(userId: string) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    // VIOLATION: Direct axios dependency
    axios.get(`https://api.example.com/users/${userId}`, {
      headers: {
        'Authorization': `Bearer ${localStorage.getItem('token')}`
      }
    })
    .then(response => {
      // VIOLATION: Knows about API response structure
      setUser({
        id: response.data.id,
        name: response.data.full_name,  // API-specific field name
        email: response.data.email_address,
        createdAt: new Date(response.data.created_timestamp)
      })
    })
    .catch(err => {
      // VIOLATION: Axios-specific error handling
      if (err.response?.status === 404) {
        setError(new Error('User not found'))
      } else {
        setError(err)
      }
    })
    .finally(() => setLoading(false))
  }, [userId])

  return { user, loading, error }
}

// VIOLATION: Component knows about localStorage implementation
function LoginForm() {
  const [credentials, setCredentials] = useState({ email: '', password: '' })
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    
    try {
      // VIOLATION: Direct API call in component
      const response = await axios.post('https://api.example.com/auth/login', {
        email: credentials.email,
        password: credentials.password
      })
      
      // VIOLATION: Direct localStorage manipulation
      localStorage.setItem('token', response.data.access_token)
      localStorage.setItem('userId', response.data.user_id)
      
      // VIOLATION: Direct window navigation
      window.location.href = '/dashboard'
    } catch (err) {
      console.error('Login failed:', err)
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      {/* form fields */}
    </form>
  )
}

// VIOLATION: Direct WebSocket dependency
function ChatRoom({ roomId }: { roomId: string }) {
  const [messages, setMessages] = useState<Message[]>([])
  
  useEffect(() => {
    // VIOLATION: Component knows WebSocket details
    const ws = new WebSocket(`wss://chat.example.com/rooms/${roomId}`)
    
    ws.onmessage = (event) => {
      const message = JSON.parse(event.data)
      setMessages(prev => [...prev, message])
    }
    
    return () => ws.close()
  }, [roomId])
  
  const sendMessage = (text: string) => {
    // VIOLATION: Direct WebSocket usage
    ws.send(JSON.stringify({ type: 'message', text }))
  }
  
  return <div>{/* chat UI */}</div>
}
```

### GOOD — Abstraction-Based (DIP compliant)

> Components depend on interfaces; implementations are injected.

```typescript
// Define interfaces that components need
interface UserService {
  getUser(id: string): Promise<User>
  updateUser(id: string, data: Partial<User>): Promise<User>
  deleteUser(id: string): Promise<void>
}

interface AuthService {
  login(email: string, password: string): Promise<AuthToken>
  logout(): Promise<void>
  getCurrentUser(): Promise<User | null>
  getToken(): string | null
}

interface MessageService {
  connect(roomId: string): void
  disconnect(): void
  sendMessage(text: string): void
  onMessage(handler: (message: Message) => void): () => void
}

// Hook depends on interface, not implementation
function useUser(userId: string, userService: UserService) {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    userService.getUser(userId)
      .then(setUser)
      .catch(setError)
      .finally(() => setLoading(false))
  }, [userId, userService])

  const updateUser = useCallback(async (data: Partial<User>) => {
    const updated = await userService.updateUser(userId, data)
    setUser(updated)
    return updated
  }, [userId, userService])

  return { user, loading, error, updateUser }
}

// Component depends on abstraction
function LoginForm({ 
  authService,
  onSuccess 
}: { 
  authService: AuthService
  onSuccess: () => void
}) {
  const [credentials, setCredentials] = useState({ email: '', password: '' })
  const [error, setError] = useState<string | null>(null)
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setError(null)
    
    try {
      await authService.login(credentials.email, credentials.password)
      onSuccess()
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Login failed')
    }
  }
  
  return (
    <form onSubmit={handleSubmit}>
      <input
        type="email"
        value={credentials.email}
        onChange={e => setCredentials(prev => ({ ...prev, email: e.target.value }))}
      />
      <input
        type="password"
        value={credentials.password}
        onChange={e => setCredentials(prev => ({ ...prev, password: e.target.value }))}
      />
      {error && <div className="error">{error}</div>}
      <button type="submit">Login</button>
    </form>
  )
}

// HTTP implementation (can be swapped)
class HttpUserService implements UserService {
  constructor(
    private apiClient: ApiClient,
    private authService: AuthService
  ) {}

  async getUser(id: string): Promise<User> {
    try {
      const response = await this.apiClient.get(`/users/${id}`, {
        headers: this.getAuthHeaders()
      })
      return this.mapApiUserToUser(response.data)
    } catch (error) {
      if (error.status === 404) {
        throw new UserNotFoundError(id)
      }
      throw error
    }
  }

  async updateUser(id: string, data: Partial<User>): Promise<User> {
    const response = await this.apiClient.patch(`/users/${id}`, 
      this.mapUserToApiUser(data),
      { headers: this.getAuthHeaders() }
    )
    return this.mapApiUserToUser(response.data)
  }

  async deleteUser(id: string): Promise<void> {
    await this.apiClient.delete(`/users/${id}`, {
      headers: this.getAuthHeaders()
    })
  }

  private getAuthHeaders() {
    const token = this.authService.getToken()
    return token ? { Authorization: `Bearer ${token}` } : {}
  }

  private mapApiUserToUser(apiUser: any): User {
    return {
      id: apiUser.id,
      name: apiUser.full_name,
      email: apiUser.email_address,
      createdAt: new Date(apiUser.created_timestamp)
    }
  }

  private mapUserToApiUser(user: Partial<User>): any {
    return {
      full_name: user.name,
      email_address: user.email
    }
  }
}

// Test implementation (for testing)
class MockUserService implements UserService {
  private users = new Map<string, User>()

  async getUser(id: string): Promise<User> {
    const user = this.users.get(id)
    if (!user) throw new UserNotFoundError(id)
    return user
  }

  async updateUser(id: string, data: Partial<User>): Promise<User> {
    const user = await this.getUser(id)
    const updated = { ...user, ...data }
    this.users.set(id, updated)
    return updated
  }

  async deleteUser(id: string): Promise<void> {
    this.users.delete(id)
  }

  // Test helper
  addUser(user: User): void {
    this.users.set(user.id, user)
  }
}

// WebSocket implementation
class WebSocketMessageService implements MessageService {
  private ws: WebSocket | null = null
  private handlers: Set<(message: Message) => void> = new Set()

  connect(roomId: string): void {
    this.ws = new WebSocket(`wss://chat.example.com/rooms/${roomId}`)
    
    this.ws.onmessage = (event) => {
      const message = JSON.parse(event.data)
      this.handlers.forEach(handler => handler(message))
    }
  }

  disconnect(): void {
    this.ws?.close()
    this.ws = null
  }

  sendMessage(text: string): void {
    if (!this.ws) throw new Error('Not connected')
    this.ws.send(JSON.stringify({ type: 'message', text }))
  }

  onMessage(handler: (message: Message) => void): () => void {
    this.handlers.add(handler)
    return () => this.handlers.delete(handler)
  }
}

// Component uses injected service
function ChatRoom({ 
  roomId,
  messageService 
}: { 
  roomId: string
  messageService: MessageService
}) {
  const [messages, setMessages] = useState<Message[]>([])
  
  useEffect(() => {
    messageService.connect(roomId)
    
    const unsubscribe = messageService.onMessage((message) => {
      setMessages(prev => [...prev, message])
    })
    
    return () => {
      unsubscribe()
      messageService.disconnect()
    }
  }, [roomId, messageService])
  
  const sendMessage = (text: string) => {
    messageService.sendMessage(text)
  }
  
  return (
    <div className="chat-room">
      <MessageList messages={messages} />
      <MessageInput onSend={sendMessage} />
    </div>
  )
}

// Dependency injection via Context
const ServiceContext = createContext<{
  userService: UserService
  authService: AuthService
  messageService: MessageService
} | null>(null)

function ServiceProvider({ children }: { children: React.ReactNode }) {
  const services = useMemo(() => {
    const authService = new HttpAuthService()
    const apiClient = new ApiClient('https://api.example.com')
    
    return {
      authService,
      userService: new HttpUserService(apiClient, authService),
      messageService: new WebSocketMessageService()
    }
  }, [])
  
  return (
    <ServiceContext.Provider value={services}>
      {children}
    </ServiceContext.Provider>
  )
}

function useServices() {
  const services = useContext(ServiceContext)
  if (!services) {
    throw new Error('useServices must be used within ServiceProvider')
  }
  return services
}

// Usage in component
function UserProfile({ userId }: { userId: string }) {
  const { userService } = useServices()
  const { user, loading, error } = useUser(userId, userService)
  
  if (loading) return <Spinner />
  if (error) return <ErrorMessage error={error} />
  if (!user) return <EmptyState />
  
  return <UserCard user={user} />
}
```

## When to Apply DIP in React

### Use DIP for

* API/backend service calls
* Authentication and authorization
* Data persistence (localStorage, IndexedDB)
* Third-party service integrations
* Real-time connections (WebSocket, SSE)
* Complex business logic

### Keep Concrete for

* Simple utility functions
* Pure transformations
* UI-only logic
* Component-specific helpers

## Anti-patterns to Avoid

1. **Direct API calls in components**: fetch/axios in components
2. **Hard-coded URLs**: API endpoints in component code
3. **Direct storage access**: localStorage/sessionStorage in components
4. **Tight coupling to libraries**: Direct use of specific libraries throughout

## React-Specific DIP Techniques

1. **Props injection** for component-level dependencies
2. **Context providers** for app-wide services
3. **Custom hooks** as service facades
4. **Factory functions** for creating configured services
5. **Higher-order components** for service injection

## Key Takeaways

* Components should **depend on interfaces**, not implementations
* **Inject dependencies** via props or context
* Keep **infrastructure details** out of components
* Use **abstraction layers** for external services
* **Testing becomes trivial** with mock implementations

## Related Best Practices

For dependency injection patterns and testing, see
👉 [best-practices.md](../best-practices.md)
