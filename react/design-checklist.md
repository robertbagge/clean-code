# React Design Checklist

Quick checklist before shipping React components and features.

## Component Design

- [ ] **Single Responsibility** - Component does one thing well
- [ ] **Props Interface** - Only accepts props it actually uses
- [ ] **Composition Ready** - Uses children/render props for extensibility
- [ ] **Error Boundary** - Wrapped in error boundary if it can fail
- [ ] **Accessibility** - Proper ARIA labels, keyboard navigation

## TypeScript

- [ ] **Discriminated Unions** - No types with many optional fields
- [ ] **Strict Types** - No `any` types without justification
- [ ] **Consumer Interfaces** - Interfaces defined where they're used
- [ ] **Type Guards** - Proper narrowing for union types

## State Management

- [ ] **Local First** - State kept as close to usage as possible
- [ ] **Controlled Inputs** - Form inputs properly controlled
- [ ] **Side Effects** - All side effects in useEffect or event handlers
- [ ] **Dependencies** - useEffect dependencies complete and minimal

## Performance

- [ ] **Memoization** - Expensive computations wrapped in useMemo
- [ ] **Callbacks** - Event handlers wrapped in useCallback when passed down
- [ ] **Lists** - Large lists virtualized or paginated
- [ ] **Code Splitting** - Large features lazy loaded

## Testing

- [ ] **Behavior Tests** - Tests user interactions, not implementation
- [ ] **No jest.mock** - Dependencies injected, not mocked
- [ ] **Error Cases** - Both success and failure paths tested
- [ ] **Accessibility** - Screen reader experience tested

## Business Logic

- [ ] **In Hooks** - All business logic encapsulated in custom hooks
- [ ] **Dependency Injection** - Services injected via props/context
- [ ] **Error Mapping** - API errors mapped to domain errors
- [ ] **Validation** - Input validation centralized and reusable

## Code Quality

- [ ] **No Magic Numbers** - All constants named and documented
- [ ] **Error Messages** - User-friendly error messages
- [ ] **Loading States** - Proper loading indicators
- [ ] **Empty States** - Meaningful empty state displays
