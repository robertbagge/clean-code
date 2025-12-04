# Expo App Folder Structure

## Goals

* A folder structure that is easy to navigate for other engineers and AI agents
* Clear separation between app-specific wiring and reusable code
* Explicit dependency direction between layers

## Mobile App Structure

```
expo-app/
├── src/
│   ├── app/           # Routes only (Expo Router)
│   ├── components/    # UI library - reusable & extractable
│   ├── core/          # App-specific wiring
│   │   ├── config/    # App configuration
│   │   ├── shell/     # Provider composition + app setup
│   │   └── theme/     # Theme setup, colors, sizes and other constants
│   ├── features/      # Domain modules - reusable & extractable
│   └── lib/           # Third-party and utility wrappers - reusable & extractable
└── test-utils/        # Test infrastructure
```

## Layer Definitions

### `app/` — Routes Only

Contains Expo Router file-based routes. No business logic, no components
beyond route wiring.

```typescript
// app/(tabs)/trips.tsx
import { TripsScreen } from '@this/features/trips'

export default function TripsRoute() {
  return <TripsScreen />
}
```

### `core/` — App-Specific Wiring

The glue that makes THIS app work. Not reusable across apps.

* `config/` — Environment variables, feature flags
* `shell/` — Provider composition (auth, theme, data layers)
* `theme/` — Tamagui config, color tokens, theme providers

### `components/` — UI Library

Reusable & extractable UI components. Could be published as a separate
package.

* **Theme-agnostic**: No hardcoded colors, padding, sizes. Use theme tokens.
* Expose public API via barrel (`index.ts`)

```typescript
// GOOD - uses theme tokens
<Button backgroundColor="$primary" padding="$md" />

// BAD - hardcoded values
<Button backgroundColor="#007AFF" padding={16} />
```

### `features/` — Domain Modules

Business logic organized by domain. Reusable & extractable across apps.

* **Theme-agnostic**: No hardcoded colors, padding, sizes. Use theme tokens.
* Each feature has its own components, hooks, types
* Expose public API via barrel (`index.ts`)
* A feature can use components from another feature

```
features/
├── auth/
│   ├── components/
│   ├── hooks/
│   ├── types.ts
│   └── index.ts       # Public API
├── trips/
└── user/
```

### `lib/` — Third-Party Wrappers

Wrappers around external libraries. Reusable & extractable as individual
packages.

* `relay/` — GraphQL client setup
* `supabase/` — Database client
* `time/` — Date/time utilities

### `test-utils/` — Test Infrastructure

Lives outside `src/`. Not production code.

## Dependency Direction

Layers should only import from layers below them:

```
app/
  ↓
core/  →  features/  →  components/
  ↓         ↓              ↓
         lib/
```

* `app/` can import from `core/`, `features/`, `components/`, `lib/`
* `core/` can import from `features/`, `components/`, `lib/`
* `features/` can import from other `features/`, `components/`, `lib/`
* `components/` can import from other `components/`, `lib/`
* `lib/` can import from other `lib/`, but not from layers above

## Theme-Agnostic Principle

Code in `components/` and `features/` must not hardcode visual values:

| Hardcoded (Bad) | Theme Token (Good) |
|-----------------|-------------------|
| `color="#333"` | `color="$text"` |
| `padding={16}` | `padding="$md"` |
| `fontSize={14}` | `fontSize="$body"` |
| `borderRadius={8}` | `borderRadius="$sm"` |

This ensures components adapt to different themes and can be extracted to
shared packages.

## Reusability & Extractability

The structure supports extracting layers to separate packages:

| Layer | Extractable To |
|-------|---------------|
| `components/` | `@org/ui` package |
| `features/` | `@org/features` package |
| `lib/` | Individual packages per wrapper |
| `core/` | Not extractable - app-specific |

## See Also

* [imports.md](./imports.md) — Import aliases, barrel exports, ESLint rules
