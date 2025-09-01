# React Best Practices

This document collects general coding practices that cut across multiple
principles. It complements the [Clean Code-focused docs](./clean-code/) (e.g.
DIP, ISP, DRY) with conventions for TypeScript interfaces, error handling, testing, and
component management.

## Other recommendations

### Use TS Discriminated Unions over Optional Fields

#### BAD - Fat type with optional fields

  ```typescript
  // BAD - Unclear which fields are valid when
  type AsyncState<T> = {
    status: 'loading' | 'error' | 'success'
    data?: T
    error?: Error
  }
  ```

#### Good discriminated union with strongly defined types

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

### Loading and Error states

#### Use Error Boundaries and Suspense to handle loading states

```typescript
interface DisplayProps {
  user: User
}

function UserInfoDisplay({ user }: DisplayProps) {
  return <div>{user.name}</div>
}

interface Props {
  userID: string
}

function UserInfo({ userID }: Props) {
  const user = useUser(userID) // suspense-enabled hook
  return <UserInfoDisplay user={user} />
}

function UserInfoError() {
  return <div>Something went wrong.</div>
}

function UserInfoLoading() {
  return <div>Loading…</div>
}

function UserInfoSection({ userID }: Props) {
  return (
    <ErrorBoundary fallback={<UserInfoError />}>
      <Suspense fallback={<UserInfoLoading />}>
        <UserInfo userID={userID} />
      </Suspense>
    </ErrorBoundary>
  )
}
```

### Component Composition to allow for reuse and efficient renders

- Use composition over inheritance
- Leverage children and render props:

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

### Performance Optimization

#### Memoization

Use `useMemo` for expensive computations:

```typescript
function DataTable({ data, filters }: DataTableProps) {
  const filteredData = useMemo(
    () => applyFilters(data, filters),
    [data, filters]
  )
  
  return <Table data={filteredData} />
}
```

## Key Takeaways

- Use **discriminated unions** over types with optional fields
- **No jest.mock()** - use dependency injection with stubs
- Encapsulate **async data loading** and **complex render** logic in hooks with injected dependencies
- Keep **components focused** - separate display from logic
- Test **behavior, not implementation**
- Keep **state local** unless sharing is necessary

## Further Reading

- [React Documentation](https://react.dev)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [Testing Library Best Practices](https://testing-library.com/docs/guiding-principles)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app)
