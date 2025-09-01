# State Management

## Goals

* Keep state as close to where it is used as possible

## Local State

Local state close to the component while separating logic with display

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

## Lifting State

Lift state only when multiple components need it:

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

## Global State

* Use Context for truly global state.
* Shared UI state belongs in a container component/hook

```typescript
type ApiClients = {
  messageApi: MessageApi
  userApi: UserApi
}

const ApiContext = createContext<ApiClients | null>(null)

function ApiProvider({ children, apiClients }: {
  children: React.ReactNode; apiClients: ApiClients
}) {
  return <ApiContext.Provider value={apiClients}>{children}</ApiContext.Provider>
}

function useMessageApi(): MessageApi {
  const apiClients = useContext(ApiContext)
  const messageApi = apiClients?.messageApi
  if (!messageApi) {
    throw new Error('useMessageApi must be used within ApiProvider')
  }
  return messageApi
}

function useUserApi(): UserApi {
  const apiClients = useContext(ApiContext)
  const userApi = apiClients?.userApi
  if (!userApi) {
    throw new Error('useUserApi must be used within ApiProvider')
  }
  return userApi
}
```
