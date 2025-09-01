# Accessibility

## Using UI libraries? Quick gotcha

* UI kits (shadcn/Radix, Material UI, Tamagui, etc.) provide accessible
  primitives, but customizations can break them.
* Common pitfalls:

  * Replacing primitives with generic elements or misusing `asChild`, losing roles/labels.
  * Removing or obscuring focus outlines when theming.
  * Icon-only controls without accessible names (`aria-label`, `accessibilityLabel`).
  * Breaking `aria-labelledby`/`aria-describedby` links by altering
    structure or IDs.
  * Nesting interactive elements (button inside link, pressable inside pressable).
  * React Native/Tamagui require explicit semantics and focus handling.
* Quick checklist:

  * Dialogs and sheets are labeled and trap + restore focus.
  * Menus and listboxes keep their structure and keyboard behavior.
  * Theme tokens meet contrast guidelines, including focus rings and disabled states.
  * Async results and errors are announced to assistive tech.
  * Keyboard-only and screen-reader paths are tested after customizations.

## Overview

* Goal: make UIs usable with screen readers, keyboards, and assistive tech
  across Web, Next.js, and React Native/Expo.
* Principles: semantic first, keyboard-first, minimal ARIA, visible focus,
  sufficient contrast, motion sensitivity, robust announcements.

## Semantics First

* Prefer native elements and roles over ARIA.
* Web: use `<button>`, `<nav>`, `<main>`, `<ul>`, `<img alt="">`.
* React Native: set `accessibilityRole`, `accessibilityLabel`, `accessibilityHint`.

```tsx
// BAD — clickable <div>
<div onClick={onSave}>Save</div>

// GOOD — native button with accessible name
<button type="button" onClick={onSave}>Save</button>
```

```tsx
// React Native
import { Pressable, Text } from 'react-native'

<Pressable
  accessibilityRole="button"
  accessibilityLabel="Save changes"
  accessibilityHint="Saves your edits and returns to the list"
  onPress={onSave}
>
  <Text>Save</Text>
</Pressable>
```

## Keyboard and Focus

* All interactive elements must be reachable with keyboard or remote.
* Do not remove focus outlines; customize them if needed.
* Avoid `tabIndex > 0`. Use DOM order and `tabIndex={0}` only for truly custom widgets.

```css
/* Web: make focus visible and consistent */
button:focus-visible,
a:focus-visible,
[role="button"]:focus-visible {
  outline: 2px solid currentColor;
  outline-offset: 2px;
}
```

```tsx
// Icon-only button needs a name
<button type="button" aria-label="Close" onClick={onClose}>
  <XIcon aria-hidden="true" />
</button>
```

## Focus Management for Overlays

* When opening a modal, move focus into it and restore focus on close.
* Use `role="dialog"` with `aria-modal="true"` and label it.

```tsx
// Web modal (simplified)
function Modal({ open, onClose, titleId, children }: {
  open: boolean
  onClose: () => void
  titleId: string
  children: React.ReactNode
}) {
  const ref = React.useRef<HTMLDivElement>(null)
  const triggerRef = React.useRef<HTMLElement | null>(null)

  React.useEffect(() => {
    if (open) {
      triggerRef.current = document.activeElement as HTMLElement
      ref.current?.focus()
    } else {
      triggerRef.current?.focus()
    }
  }, [open])

  React.useEffect(() => {
    if (!open) return
    const onKey = (e: KeyboardEvent) => { if (e.key === 'Escape') onClose() }
    document.addEventListener('keydown', onKey)
    return () => document.removeEventListener('keydown', onKey)
  }, [open, onClose])

  if (!open) return null

  return (
    <div role="dialog" aria-modal="true" aria-labelledby={titleId}>
      <div tabIndex={-1} ref={ref}>
        {children}
        <button type="button" onClick={onClose}>Close</button>
      </div>
    </div>
  )
}
```

## Images and Media

* Provide meaningful `alt` text. Use `alt=""` for decorative images.
* Prefer captions and transcripts for audio and video.
* Avoid auto-playing media. If unavoidable, provide a pause or stop control.

```tsx
// Decorative image
<img src="/wave.svg" alt="" aria-hidden="true" />
```

## Announcing Dynamic Changes

* Use live regions for async success and error messages so screen readers hear them.

```tsx
function ToastRegion({ message }: { message: string | null }) {
  return (
    <div
      aria-live="polite"
      aria-atomic="true"
      style={{
        position: 'absolute',
        width: 1,
        height: 1,
        overflow: 'hidden',
        clip: 'rect(1px,1px,1px,1px)'
      }}
    >
      {message}
    </div>
  )
}
```

## Color and Contrast

* Do not rely on color alone to convey meaning.
* Ensure sufficient contrast (aim for at least 4.5:1 for normal text).
* Preserve visible focus styles even in custom themes.

```tsx
// Add a non-color cue as well
<span aria-hidden="true">●</span>
<span>Success</span>
```

## Motion and Reduced Motion

* Respect user preference to reduce motion.

```css
@media (prefers-reduced-motion: reduce) {
  * {
    animation-duration: 0.001ms !important;
    animation-iteration-count: 1 !important;
    transition: none !important;
  }
}
```

```tsx
const prefersReducedMotion =
  typeof window !== 'undefined' &&
  window.matchMedia?.('(prefers-reduced-motion: reduce)').matches

return <AnimatedThing disabled={prefersReducedMotion} />
```

## Lists, Tables, and Regions

* Use semantic lists for collections and headings for structure.
* For page layout, include one `<main>` region and use headings in order
  (`h1` → `h2`).
* Tables should use `<th scope="col|row">` and captions when data is tabular.

```tsx
<main>
  <h1>Users</h1>
  <ul>
    {users.map((u) => <li key={u.id}>{u.name}</li>)}
  </ul>
</main>
```

## Internationalization Hints

* Provide language metadata at the document or root level where possible.
* Avoid concatenating translated strings that break reading order.
* Support RTL by using logical CSS properties instead of hardcoded left or right.

```css
/* Example logical properties */
.card {
  padding-inline: 16px;
  margin-inline: auto;
}
```

## Anti-patterns to Avoid

* Clickable `<div>` or `<span>` instead of interactive elements.
* Removing focus outlines without providing a visible replacement.
* Custom widgets without keyboard support.
* Using ARIA to replace what a semantic element already does.
* Relying on color alone to communicate state.
* Trapping focus in overlays without an Escape key or a close button.

## Quick Checklist

* Semantic elements or explicit roles are used appropriately.
* Every interactive control is reachable and operable via keyboard.
* Focus is visible and managed on route and overlay changes.
* Icon-only controls have accessible names.
* Dynamic updates are announced when necessary.
* Contrast meets guidelines and motion respects user preference.
* Works with at least one screen reader and the platform’s accessibility service.

## Minimal Tests

```tsx
// Web
import { render, screen } from '@testing-library/react'
import userEvent from '@testing-library/user-event'

test('close button is accessible by role and name', async () => {
  render(<button aria-label="Close">×</button>)
  const btn = screen.getByRole('button', { name: /close/i })
  await userEvent.click(btn)
})
```

```tsx
// React Native (illustrative)
import { Pressable, Text } from 'react-native'

<Pressable accessibilityRole="button" accessibilityLabel="Close">
  <Text>×</Text>
</Pressable>
```

## Cross-Platform Notes

* Web has richer native semantics. Favor native elements first.
* React Native relies on `accessibilityRole`, `accessibilityLabel`, and
  focus management via screen readers and platform APIs.
* Keep abstractions thin so web and native can supply platform-appropriate
  accessibility without leaking DOM specifics into shared code.
