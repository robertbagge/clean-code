# Styling

A tiny guide to picking and using a styling approach consistently
across React, Next.js, and React Native. This repo is
**styling-system agnostic**: pick one primary approach and use it
consistently. Define tokens (colors, spacing, typography, radii)
once and wire them into your chosen system.

## Common approaches

* Tailwind CSS (web) / NativeWind (React Native)
* StyleX
* Tamagui
* CSS-in-JS (styled-components, Emotion)
* CSS Modules / PostCSS

## Recommendations

* Pick **one** primary styling system per codebase and stick to it
* Centralize **design tokens** and consume them via your system’s theme mechanism
* Standardize a **variant pattern**
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
