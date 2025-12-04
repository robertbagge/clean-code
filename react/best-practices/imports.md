# Imports

## Goals

* Explicit, predictable import paths across the codebase
* Clean refactoring - moving files doesn't break imports elsewhere
* Enforce module boundaries via barrel exports

## Path Aliases

Use `@this/*` to reference any file under `src/`. Never use parent-relative
imports (`../`).

```typescript
// GOOD
import { Button } from '@this/components/ui'
import { useAuth } from '@this/features/auth'
import { formatTime } from '@this/lib/time'

// BAD - parent-relative imports
import { Button } from '../../components/ui'
import { useAuth } from '../../../features/auth'
```

### Why?

* **Refactoring**: Moving a file doesn't require updating its imports
* **Readability**: Import paths show where code lives in the architecture
* **Consistency**: Same path works from any file depth

### Setup (Babel)

```javascript
// babel.config.js
plugins: [
  ['module-resolver', {
    alias: {
      '@this': './src',
    },
  }],
]
```

## Barrel Exports

Each module exposes its public API via `index.ts`. Import from the barrel,
not internal files.

```typescript
// GOOD - import from barrel
import { UserCard, UserAvatar } from '@this/features/user'

// BAD - reaching into module internals
import { UserCard } from '@this/features/user/components/user-card'
```

### Why?

* **Encapsulation**: Internal structure can change without breaking consumers
* **Discoverability**: Barrel shows what's public
* **Refactoring**: Rename/move internals freely

### Barrel Structure

```typescript
// features/user/index.ts
export { UserCard } from './components/user-card'
export { UserAvatar } from './components/user-avatar'
export { useUser } from './hooks/use-user'
export type { User } from './types'
```

## Quick Reference

| Pattern | Allowed | Reason |
|---------|---------|--------|
| `@this/features/user` | Yes | Barrel import |
| `@this/features/user/hooks/use-user` | No | Internal file |
| `../components/button` | No | Parent-relative |
| `./child-component` | Yes | Same-directory sibling |
| `react-native` | Yes | External package |
