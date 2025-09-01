# Performance Guardrails

## Goals

* Provide sensible default guardrails for good performance

## Techniques

### State Colocation and Minimal State

* Keep state as close as possible to where it’s used.
* Store the minimal source of truth; derive the rest.

```typescript
// BAD: storing derived state (double source of truth)
const [filtered, setFiltered] = useState(applyFilters(data, filters))

// GOOD: derive in render/memo
const filtered = useMemo(() => applyFilters(data, filters), [data, filters])
```

### Stable References

* Memoize expensive computations and stable objects you pass as props.
* Memoize context values to avoid provider-wide re-renders.

```typescript
const value = useMemo(() => ({ theme, setTheme }), [theme])
<ThemeContext.Provider value={value}>{children}</ThemeContext.Provider>
```

### Rendering Lists

* Always provide stable `key`s (`id` over index).
* Virtualize large lists (web/RN platform-appropriate virtual list).

```typescript
items.map((item) => <Row key={item.id} item={item} />)
```

### Avoid Unnecessary Re-renders

* Prefer splitting components so prop changes are localized.
* Use `React.memo` for pure display components receiving stable props.

```typescript
const UserRow = React.memo(function UserRow({ user }: { user: User }) {
  return <div>{user.name}</div>
})
```

### Effects Hygiene

* Keep `useEffect` small and focused.
* Include all dependencies; move unstable function creation outside or into
  `useCallback` when passing them down.

```typescript
const handleSelect = useCallback((id: string) => onSelect(id), [onSelect])
```

### Profiler and Budget

* Establish render budgets for hot paths (e.g., list row < 1ms).
* Use React Profiler/Flipper/Native dev tools to spot regressions.

## Notes

* Large object props cause shallow-compare misses; pass only what you use.
See [ISP](../clean-code/interface-segregation-principle.md)
