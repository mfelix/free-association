# Mobile UI Improvements Design Spec

## Overview

Three targeted improvements to the mobile experience of the Free Association Counter app. All changes are scoped to the single `index.html` file (CSS and minimal HTML).

## Problem Statement

1. **Landscape mode on phones is broken** — The app uses a single `max-width: 768px` breakpoint for mobile. When a phone rotates to landscape, two things can happen: (a) phones with landscape width >768px (most modern phones) actually get the desktop layout, which works but may have oversized elements for the short viewport, and (b) narrower phones like iPhone SE (667px landscape width) stay in the mobile stacked layout, which has almost no vertical room and is unusable.
2. **Drawer close button looks out of place** — The boxed `x` button with border and background feels widget-y against the minimal dark aesthetic.
3. **Bottom nav bar feels heavy** — The frosted-glass backdrop-filter background and emoji-style HTML entity icons give an iOS tab bar feel that clashes with the app's minimal design.

## Design Decisions

### 1. Landscape Phone Layout

**Approach:** Two complementary media queries.

**Query A — Narrow phones in landscape (nested inside 768px block):**
The landscape query is nested inside the existing `@media (max-width: 768px)` block. This targets narrow phones (like iPhone SE at 667px) where the mobile column layout fires but landscape makes it unusable.

```css
@media (max-width: 768px) {
  /* ...existing mobile styles... */

  @media (orientation: landscape) and (max-height: 500px) {
    /* Override mobile column layout back to side-by-side */
  }
}
```

**Query B — Wide phones in landscape (standalone):**
Most modern phones in landscape exceed 768px width and already get the desktop side-by-side layout by default. However, the short viewport (~350-430px) means elements sized for desktop are too large. A standalone query handles this:

```css
@media (orientation: landscape) and (max-height: 500px) and (min-width: 769px) {
  /* Compact overrides for desktop layout on short viewports */
}
```

**Layout changes (Query A — narrow phones):**
- `.main-area`: restore `flex-direction: row` so panels sit side-by-side
- `.main-area`: set `padding-bottom: 0` (remove nav bar reservation including safe-area-inset)
- `.panel-left`: remove `border-bottom`, restore `border-right: 1px solid var(--border)` divider
- `.ring-wrapper`: shrink to `min(30vh, 140px)` width and height
- `.number`: reduce to `clamp(2rem, 8vh, 4rem)`
- `.mobile-nav`: `display: none !important` — hide bottom nav
- `.panel-left::after` (touch hint): `display: none` — save vertical space
- `.go-countdown`: `font-size: clamp(4rem, 20vh, 8rem)` (use `vh` to scale with the constrained axis)
- `.ring-label`, `.label`: `margin-top: 0.2rem; font-size: 0.65rem`
- `.ring-below`: `margin-top: 0.2rem`
- `.info-panel.drawer-open, .sidebar.drawer-open`: reset `transform: translateY(100%)` — CSS fallback in case the JS orientation listener doesn't fire before paint (the JS listener is the primary mechanism)

**Layout changes (Query B — wide phones):**

Query B only needs size compaction — the desktop layout is already in effect at 769px+, so there is no mobile nav to hide, no touch hint, no padding-bottom reservation, and `.panel-left` already has its `border-right`.

- `.ring-wrapper`: shrink to `min(30vh, 140px)` width and height
- `.number`: reduce to `clamp(2rem, 8vh, 4rem)`
- `.go-countdown`: `font-size: clamp(4rem, 20vh, 8rem)`
- `.ring-label`, `.label`: `margin-top: 0.2rem; font-size: 0.65rem`
- `.ring-below`: `margin-top: 0.2rem`

**Small phone interaction (max-width: 480px):** The existing 480px breakpoint shrinks `.ring-wrapper` and `.number` for very small phones. A phone with landscape width under 480px is extremely rare, but to be safe, the landscape Query A values should use `!important` on ring-wrapper sizing, or a nested landscape override should be added inside the 480px block as well. The simplest approach: add the same landscape ring-wrapper override inside the 480px block.

**Why max-height 500px:** iPhone 14 Pro Max landscape is ~428px tall, iPad Mini portrait is ~1024px. The 500px threshold cleanly separates phones in landscape from tablets in any orientation.

**Drawer rotation handling:** Add a small JS listener for orientation changes that closes any open drawer when rotating to landscape. This prevents a drawer opened in portrait from remaining stuck open over the landscape layout.

```javascript
window.matchMedia('(orientation: landscape)').addEventListener('change', function(e) {
  if (e.matches) closeDrawer();
});
```

### 2. Drawer Close Button — Circular Pill

**Current:** 32px rounded square, `border: 1px solid var(--border-light)`, `background: var(--surface)`, `border-radius: 8px`.

**New:** 28px circle, no border, subtle translucent background.

```css
.drawer-close-btn {
  width: 28px;
  height: 28px;
  background: rgba(255, 255, 255, 0.08);
  border: none;
  border-radius: 50%;
  font-size: 0.9rem;
}

.drawer-close-btn:active {
  background: rgba(255, 255, 255, 0.15);
  color: var(--text);
}

.drawer-close-btn:focus-visible {
  outline: 2px solid var(--accent);
  outline-offset: 2px;
}
```

### 3. Bottom Nav — Transparent, No Emoji Icons

**Background:** Remove `backdrop-filter` and the opaque background. Use a very subtle semi-transparent background to maintain readability across all 7 themes (some themes have lighter surfaces where fully transparent text would be hard to read):

```css
.mobile-nav {
  background: rgba(0, 0, 0, 0.4);
  -webkit-backdrop-filter: none;
  backdrop-filter: none;
}
```

**Icons:** Remove HTML entity icons, keep text-only labels:

Before:
```html
<button class="mobile-nav-btn" id="mobile-info-btn">
  <span class="mobile-nav-icon">&#8505;</span>
  <span class="mobile-nav-label">Info</span>
</button>
```

After:
```html
<button class="mobile-nav-btn" id="mobile-info-btn">
  <span class="mobile-nav-label">Info</span>
</button>
```

Same for the Scores button (remove `&#9733;` icon span). Theme button keeps its `.mobile-theme-dot`.

**Button layout:** Switch from `flex-direction: column` (icon over label) to center-aligned single label. Set `min-height: 44px` on `.mobile-nav-btn` to maintain Apple's recommended minimum tap target. Set `font-size: 0.7rem` directly on `.mobile-nav-btn` (up from `0.6rem`), remove the separate `.mobile-nav-label` font-size rule since the label is now the only text child. Remove `.mobile-nav-icon` CSS rule at line 1242-1245 (dead code after HTML change).

## Files Modified

- `index.html` — CSS changes (landscape media queries, close button restyle, nav restyle), minor HTML changes (remove icon spans from nav buttons), small JS addition (orientation change listener)
- `public/index.html` — mirror all changes (deployed copy)

## Testing

- Chrome DevTools responsive mode: test at 667x375 (iPhone SE landscape), 844x390 (iPhone 14 landscape), 926x428 (iPhone 14 Pro Max landscape)
- Verify portrait mode is unaffected at all tested widths
- Verify tablet (768px+) portrait/landscape is unaffected
- Test drawer open/close in portrait, then rotate to landscape — drawer should auto-close
- Test that narrow-phone landscape (SE) shows side-by-side panels
- Test that wide-phone landscape (14 Pro Max) shows compact side-by-side
- Confirm close button pill style in drawer, verify focus-visible outline
- Confirm nav readability across all 7 themes (default, midnight, ember, forest, violet, rose, mono)
- Verify nav tap targets meet 44px minimum height
