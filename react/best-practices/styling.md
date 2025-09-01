# Styling

A tiny guide to picking and using a styling approach consistently
across React, Next.js, and React Native. This repo is
**styling-system agnostic**: pick one primary approach and use it
consistently. Define tokens (colors, spacing, typography, radii)
once and wire them into your chosen system.

## Common approaches

* Tailwind CSS (web) / NativeWind (React Native)

  * Utility-first classes with a central theme config
  * Fast to ship, good constraints, excellent DX
  * Works well with class composition helpers (e.g., `clsx`, `cva`) for variants

* StyleX

  * Compile-time, atomic CSS with type-safe style objects
  * Great runtime performance and dead-code elimination
  * Encourages co-located, component-scoped styles

* Tamagui

  * Cross-platform **style props + variants** with a compiler for static extraction
  * Shared tokens/themes across web and native, built-in primitives
  (`Stack`, `XStack`, etc.)
  * Strong option when you want one styling system for both platforms

* CSS-in-JS (styled-components, Emotion)

  * Component-scoped styles with theming and variants
  * Powerful dynamic styling; mind runtime cost and configure SSR correctly

* CSS Modules / PostCSS

  * Familiar, zero-runtime CSS with per-file scoping
  * Pair with a variant pattern (class composition) for reusable components

## Recommendations

* Pick **one** primary styling system per codebase and stick to it
* Centralize **design tokens** and consume them via your system’s theme mechanism

  * Tailwind/NativeWind: `theme` config
  * StyleX: shared token maps
  * Tamagui: tokens and themes
* Standardize a **variant pattern**

  * Tailwind/NativeWind: `cva` or a small variant helper
  * StyleX: combine style objects per prop
  * Tamagui: component `variants`
* Ensure **dark mode** and theming pass through a single source of truth
* Keep **accessibility** visible: do not remove focus outlines; ensure contrast
* Prefer **compile-time** or static extraction where possible for performance
* For React Native:

  * Avoid recreating inline style objects every render
  * Prefer the styling system’s primitives or memoized style objects

## Scope of this guide

* This guide does not prescribe a single styling system
* Consistency matters more than the specific choice
* Use whatever aligns with your team’s stack and performance needs
