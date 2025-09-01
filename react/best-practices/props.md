# Props

## Goals

* Make components easy to compose, test, and optimize.
* Separate data (nouns) from behavior (verbs).

## Data vs Behavior

* Data props are nouns: `user`, `items`, `title`.
* Behavior props are callbacks: `onSave`, `onClose`.
* Invert side effects to the parent via callbacks.

```tsx
// GOOD: behavior injected
type ExportButtonProps = {
  onExport: () => Promise<void>
  onError?: (error: Error) => void
}
function ExportButton({ onExport, onError }: ExportButtonProps) {
  const [busy, setBusy] = useState(false)
  const click = async () => {
    setBusy(true)
    try { await onExport() } catch (e) { onError?.(e as Error) } finally { setBusy(false) }
  }
  return <Button disabled={busy} onPress={click}>{busy ? 'Exporting…' : 'Export'}</Button>
}
```

## Event Props

* Use `onX` naming; first parameter is the meaningful payload.
* Prefer specific payloads over raw events.

```tsx
type Props = { onSelect: (id: string) => void }
```

## Booleans vs Variants

* Avoid boolean explosions. Use a `variant` prop for mutually exclusive styles.
* Keep independent toggles as booleans (`isLoading`, `disabled`).

```tsx
type ChipVariant = 'default' | 'info' | 'success' | 'warning'
type ChipProps = { variant?: ChipVariant; isSelected?: boolean }
```

## Required vs Optional

* Make required props truly required in the type.
* Provide defaults via parameter defaults, not `defaultProps`.

```tsx
type BadgeProps = { label: string; max?: number }
function Badge({ label, max = 99 }: BadgeProps) { /* ... */ }
```

## Stable References

* Avoid inline object/array/function props that change each render.
* Memoize where necessary at the call site.

```tsx
const actions = useMemo(() => [{ key: 'refresh', label: 'Refresh' }], [])
<Toolbar actions={actions} />
```

## Children and Slots

* Use `children` for the main slot.
* For multiple regions, use named slots/props: `header`, `footer`, `icon`.

```tsx
type CardProps = { header?: ReactNode; footer?: ReactNode; children?: ReactNode }
```

## Shaping Props for Performance

* Prefer a small number of well-shaped props over many fine-grained ones.
* Keep prop identity stable to leverage `React.memo`.

```tsx
const UserRow = React.memo(function UserRow({ user }: { user: User }) { /* ... */ })
```

## Prop Drilling and Alternatives

* If a prop crosses more than 2–3 levels and is truly global, use context.
* For localized cross-cutting behavior, consider render props or a small provider near the subtree.

## TypeScript Tips

* Use discriminated unions for visual states instead of multiple booleans.
* Mark read-only inputs as `readonly` when appropriate.

```ts
type AvatarProps = { readonly url: string; readonly size: number }
```

## Cross-Platform Notes (Web/Native)

* Prefer platform-agnostic names (`onPress` in RN, `onClick` on web). If sharing code, consider adapter props or wrappers.
* Avoid web-only DOM types in public prop signatures; use `ReactNode`, strings, numbers, objects.
