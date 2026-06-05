# AI Search Component — Generation Prompt

## What this builds

An animated AI-powered search bar for a site header. Expands inline on click, shows filtered results with categories, and renders a typewriter AI Answer block at the bottom.

---

## Prompt

```
Build an animated AI search component for a Next.js (App Router) header using Framer Motion and Tailwind CSS.

### Behavior
- In the navbar: a small search icon button (36×36px, rounded-xl)
- On click: the button smoothly expands to a full input field (280px wide) using `motion.div` with `animate={{ width: open ? 280 : 36 }}`
- Expansion easing: `[0.16, 1, 0.3, 1]` (spring-like overshoot) — match this curve everywhere
- On click again (or ESC): collapses back to icon
- Clicking outside the component closes it
- The icon morphs: search → close with `AnimatePresence mode="wait"` and rotate transition

### Input field
- Placeholder: "Ask AI or search…"
- `opacity` animates 0→1 on open with a 0.1s delay
- `pointerEvents: none` when collapsed

### Dropdown panel (appears below the expanded field)
- `initial={{ opacity: 0, y: 8, scale: 0.97 }}` → `animate={{ opacity: 1, y: 0, scale: 1 }}`
- Dark glass background: `rgba(11, 18, 32, 0.97)` + `backdropFilter: blur(24px)`
- Border: themed accent color variable (`--accent-border`)
- Box shadow: `0 24px 64px rgba(0,0,0,0.5)` + accent border glow
- Width: 380px, `right-0` aligned

### Results list
- Show 5 default suggestions when query is empty (label: "Suggested")
- Filter items by label, description, and section as user types
- Each result: themed icon (SVG, 16×16 in a 28×28 rounded-lg container), label, description, section badge
- Section badge: small pill with accent bg at 8% opacity and accent text color
- Items animate in with stagger: `delay: i * 0.04`
- Hover: `bg-white/[0.05]` transition
- "No results" empty state

### AI Answer block
- Sits below the results list, above the footer
- Header row: star/spark icon in an accent gradient badge + "AI ANSWER" label (uppercase, tracking-widest, accent color)
- While typing: three pulsing dots (`animate={{ opacity: [0.3, 1, 0.3], scale: [0.8, 1, 0.8] }}`, staggered 0.15s each)
- Typewriter effect: reveal text character by character using a `useTypewriter` hook with a `setTimeout` loop at ~18ms/char
- Blinking cursor: `motion.span` `w-0.5 h-3` with `animate={{ opacity: [1, 0] }}` repeating, shown while not done
- When query is empty: show default AI intro text; when query matches a known item (after 600ms debounce): show item-specific answer
- Background: `rgba(var(--accent-rgb), 0.05)` with accent border

### Footer
- `Press [ESC] to close` on the left
- `Powered by AI` (accent color) on the right
- Border-top with accent border color

### Theme integration
- Use CSS variables throughout: `--accent`, `--accent-rgb`, `--accent-dim`, `--accent-border`, `--accent-glow`
- The glowing border on the input uses `--accent-border` and `0 0 20px var(--accent-glow)`
- A shimmer overlay on the open input: `background: linear-gradient(90deg, transparent, rgba(var(--accent-rgb),0.06), transparent)` + `backgroundSize: 200% 100%` + `animation: shimmer 2s infinite`
- All colors adapt automatically when the user switches themes

### Data shape
Each search item:
```ts
{ label: string, section: string, href: string, icon: string, desc: string }
```
AI answers: a `Record<iconKey, string>` map + a default fallback string.

### Hooks
- `useTypewriter(text, running, speed?)` — returns `{ displayed, done }`
  - Resets and reruns whenever `text` or `running` changes
  - 300ms initial delay before first character
- Debounce query changes by 600ms before triggering AI answer swap

### Integration
Place `<SearchBar />` in the navbar's right-side flex row, before the theme switcher and CTA buttons. No SSR issues — it's fully `'use client'`.

### Tech stack
- React 18+ with hooks
- Framer Motion (`motion`, `AnimatePresence`)
- Tailwind CSS utility classes
- CSS custom properties for theming (no hardcoded colors)
- TypeScript
```

---

## Key design decisions

| Decision | Reason |
|---|---|
| Expand inline (not modal) | Keeps context, feels native to the navbar |
| `[0.16, 1, 0.3, 1]` easing everywhere | Matches the rest of the theme's motion language |
| 600ms debounce for AI trigger | Avoids firing on every keystroke |
| Typewriter at 18ms/char | Fast enough to feel alive, slow enough to be readable |
| CSS variables for all colors | Works with any theme switcher automatically |
| `right-0` aligned dropdown | Prevents overflow on narrower viewports |

---

## Files produced

- `components/SearchBar.tsx` — standalone component, drop-in
- Add `import SearchBar from './SearchBar'` and `<SearchBar />` to navbar's right-side row