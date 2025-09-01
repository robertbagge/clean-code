# Naming

## Goals

* Consistent, descriptive names make intent obvious across React, Next.js,
  and React Native.
* Prefer clarity over brevity. Use the project’s domain language consistently.

## Components

* Components are PascalCase nouns: `UserCard`, `AccountScreen`, `SettingsPanel`.
* Files match component names: `UserCard.tsx`.
* Display vs container naming to make consuming code neat and clean
  * Display/UI-only:
    * `XDisplay` when there is a container/behavioural wrapper, e.g
      * `UserCardDisplay`
      * `AvatarDisplay`
    * `X` when there is no container/behaviour, e.g.
      * `UserCard`
      * `Avatar`.
  * Container/behavioral: `UserCard`, `UserList`.

## Hooks

* Hooks start with `use`: `useUser`, `useAuth`, `useDebounce`.
* Return a stable object with named fields; avoid positional tuples beyond
  common patterns.
* Eventful hooks use verb phrases: `useFetchUser`, `usePersistDraft`.

## "function" for components, "const" for the rest

### Component

```typescript
function Component(props: Props) {
  // implementation
}
```

### Other functions

```typescript
const useX = () => {
  // implementation
}

const doSomething = () => {
  // implementation
}
```

## Events and Handlers

* Props that represent events use `onX`: `onSave`, `onDismiss`.
* Internal handlers use `handleX`: `handleSave`, `handleDismiss`.
* Avoid passing synthetic events across boundaries; pass data.

```tsx
// GOOD
<ItemRow item={item} onSelect={(id) => open(id)} />

// Inside ItemRow
type Props = { item: Item; onSelect: (id: string) => void }
function ItemRow({ item, onSelect }: Props) {
  return <Pressable onPress={() => onSelect(item.id)}>{item.name}</Pressable>
}
```

## Booleans and Variants

* Booleans are `isX`/`hasX`/`canX`/`shouldX`: `isActive`, `hasBorder`.
* Prefer a single `variant` union over multiple booleans.

```tsx
type ButtonVariant = 'primary' | 'ghost' | 'destructive'
type ButtonProps = { variant?: ButtonVariant }
```

## Types and Interfaces

* Domain models are nouns: `User`, `Message`, `AuthToken`.
* Error types end with `Error`: `UserNotFoundError`.
* Discriminated unions use a common tag: `status | kind | type`.

```ts
type Loading = { status: 'loading' }
type Failure = { status: 'error'; error: Error }
type Success<T> = { status: 'success'; data: T }
type AsyncState<T> = Loading | Failure | Success<T>
```

## Functions and Variables

* Functions are verbs: `loadUsers`, `updateProfile`.
* Include units in names where relevant: `timeoutMs`, `widthPx`.
* Collections are plural: `users`, `itemsById`.
* IDs end with `Id`: `userId`, `messageId`.

## Generics

* Use short, meaningful names: `T`, `TItem`, `TResult`.
* When generic meaning is domain-specific, prefer explicit: `TUser`.

## Constants and Enums

* Immutable config-like constants are UPPER\_SNAKE: `DEFAULT_PAGE_SIZE`.
* String unions preferred over enums for props and variants in UI code.

## Files and Modules

* One component per file when practical.
* Re-export public APIs via `index.ts` within a feature folder.
* Avoid abbreviation-heavy names; consistency beats terseness.
