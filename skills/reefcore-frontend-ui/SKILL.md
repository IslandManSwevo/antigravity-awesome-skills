---
name: reefcore-frontend-ui
description: >
  Use this skill whenever building, modifying, or reviewing any frontend UI in
  ReefCore — including white-label marina sites, the Operator Dashboard, the
  ReefCore admin console, GovTech compliance dashboards, or any new React/Next.js
  component in the platform. Covers the MarinerTemplate design system, CSS variable
  token rules, GSAP animation patterns, tenant white-labeling, multi-page template
  structure, and GovTech dashboard conventions. Trigger on any mention of:
  component, page, dashboard, layout, GSAP, MarinerTemplate, white-label, marina
  site, KPI strip, slip board, operator dashboard, compliance dashboard, tenant
  branding, CSS tokens, or any frontend/UI/UX work in the ReefCore stack.
---

# ReefCore Frontend & UI/UX Skill

## Design Philosophy

ReefCore's UI has two distinct surfaces. Know which one you're working on before writing a single line of CSS.

| Surface | Audience | Tone | Primary Typeface |
| :--- | :--- | :--- | :--- |
| **MarinerTemplate** (marina white-label sites) | Boat owners, tourists, charter guests | Refined maritime luxury | Playfair Display (headers) + Outfit (body) |
| **Operator / Admin / GovTech Dashboards** | Marina operators, business owners, compliance staff | Utilitarian precision — data-dense, never cramped | Playfair Display (hero numbers) + Outfit (data) |

Both surfaces share the same token system and GSAP conventions. Neither surface should ever feel like a generic SaaS product template.

---

## The MarinerTemplate Token System

All color and spacing values **must** flow through CSS custom properties. Never hardcode hex values outside of a `:root` or `.plm-root` block.

### Core Palette Tokens

```css
:root {
  /* Depths — background layers from darkest to lightest */
  --ink:    #060810;   /* page background */
  --deep:   #090e1c;   /* panel/sidebar background */
  --navy:   #0f2044;   /* elevated surfaces */

  /* Foam — text and foreground */
  --foam:   #e8f4f8;   /* primary text */
  --muted:  rgba(232, 244, 248, 0.45); /* secondary/label text */
  --white:  #fafcff;   /* highest-contrast text */

  /* Gold — the tenant accent (overridable) */
  --gold:   var(--tenant-primary-color, #c9a84c);
  --goldlt: #e8c97a;   /* hover/active state of gold */

  /* Status */
  --positive: rgba(74, 222, 128, 0.85);
  --warn:     #f59e0b;
  --danger:   rgba(255, 99, 99, 0.85);

  /* Structure */
  --border: rgba(232, 244, 248, 0.08);
}
```

### Tenant Overrides

The `--tenant-primary-color` CSS variable is the **only** value a white-label client controls. It is injected by the tenant config at runtime.

```tsx
// ✅ Correct: gold is always derived from --tenant-primary-color
<div style={{ '--tenant-primary-color': tenant.brandColor } as React.CSSProperties}>
  {children}
</div>

// ❌ Never hardcode brand colour in component logic
<div style={{ color: '#c9a84c' }}>
```

If a client has no `brandColor`, `--gold` falls back to `#c9a84c` (Port Lucaya gold). Every accent in the UI must reference `var(--gold)` — never `--goldlt` for primary CTA states.

---

## Typography Rules

```text
Display headings:   Playfair Display, serif — weight 700 or 900
Section headings:   Playfair Display, serif — weight 400 or 700
Body / labels:      Outfit, sans-serif — weight 300–600
Numeric KPIs:       Playfair Display, serif — weight 700
Meta / timestamps:  Outfit, sans-serif — weight 300, tracked (letter-spacing: 0.04em)
Uppercase labels:   Outfit, sans-serif — weight 500, letter-spacing: 0.14–0.20em, font-size ≤ 11px
```

**Font import** — always load both families together from Google Fonts:

```text
Playfair Display:ital,wght@0,400;0,700;0,900;1,400;1,700
Outfit:wght@300;400;500;600
```

Never substitute Inter, Roboto, or any system font family.

---

## Component Architecture

### `.plm-root` Scoping

All ReefCore components are scoped under a `.plm-root` class. This prevents token bleed when the component is embedded in a host page (Next.js App Router) that has its own styles.

```tsx
export default function MyComponent() {
  return (
    <div className="plm-root">
      <style>{STYLE}</style>
      {/* content */}
    </div>
  );
}
```

The `STYLE` constant is a tagged template string containing all component CSS. This is the preferred pattern for operator dashboard components. For multi-page marina sites, use Tailwind utility classes against the CSS variable tokens instead.

### Panel Pattern

The fundamental building block for operator-facing UI:

```css
.panel {
  background: var(--deep);
  border: 1px solid var(--border);
}
.panel-head {
  padding: 16px 24px;
  border-bottom: 1px solid var(--border);
  display: flex;
  align-items: center;
  justify-content: space-between;
}
.panel-head h3 {
  font-size: 13px;
  color: var(--foam);
  font-weight: 500;
  letter-spacing: 0.04em;
}
```

No `border-radius` on panels. ReefCore uses sharp corners throughout. **This is intentional and non-negotiable.** Rounded corners soften the nautical-precision aesthetic.

### KPI Strip Pattern

Used on every dashboard overview page. Always 5 columns on desktop, collapsing to 3 then 2:

```css
.kpi-strip {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 1px;
  border: 1px solid var(--border);
}
.kpi { padding: 20px 24px; background: var(--deep); }
.kpi-label { font-size: 9px; text-transform: uppercase; letter-spacing: 0.18em; color: var(--muted); }
.kpi-val { font-family: 'Playfair Display', serif; font-size: 28px; color: var(--white); font-weight: 700; line-height: 1; }
.kpi-delta { font-size: 11px; margin-top: 4px; }
.delta-up   { color: var(--positive); }
.delta-down { color: var(--danger); }
```

### Status Badges

```css
/* Generic status badge — colour via modifier class */
.status-badge { font-size: 10px; text-transform: uppercase; letter-spacing: 0.14em; padding: 3px 10px; border: 1px solid; }
.status-active     { background: rgba(74, 222, 128, 0.1);  color: var(--positive); border-color: rgba(74, 222, 128, 0.2); }
.status-pending    { background: rgba(245, 158, 11, 0.12); color: var(--warn);     border-color: rgba(245, 158, 11, 0.2); }
.status-failed     { background: rgba(255, 99, 99, 0.1);   color: var(--danger);   border-color: rgba(255, 99, 99, 0.2); }
.status-held       { background: rgba(201, 168, 76, 0.1);  color: var(--gold);     border-color: rgba(201, 168, 76, 0.2); }
```

Map payment/escrow states to status classes:

- `COMPLETED` → `status-active`
- `PENDING` / `PENDING_EXTERNAL` / `HELD` → `status-pending`
- `FAILED` / `FAILED_EXTERNAL` → `status-failed`
- `HELD` (escrow) → `status-held`

---

## GSAP Animation Conventions

GSAP is the **only** animation library permitted in ReefCore frontends. No Framer Motion, no CSS `transition` for entrance animations (CSS transitions are allowed for hover states only).

### Non-Negotiable GSAP Rules

**1. Always use `gsap.context` and call `ctx.revert()` on cleanup.**

```tsx
// ✅ Correct — no memory leaks, safe for React Strict Mode
useLayoutEffect(() => {
  const ctx = gsap.context(() => {
    gsap.from('.kpi', { y: 20, opacity: 0, stagger: 0.06, duration: 0.7, ease: 'power3.out', delay: 0.2 });
    gsap.from('.panel, .slip-board, .sweep-widget', { y: 30, opacity: 0, stagger: 0.1, duration: 0.9, ease: 'power3.out', delay: 0.4 });
  }, rootRef);
  return () => ctx.revert();
}, []);

// ❌ Never animate without gsap.context — leaks on unmount
useLayoutEffect(() => {
  gsap.from('.kpi', { y: 20, opacity: 0 });
}, []);
```

**2. Use `useLayoutEffect` not `useEffect` for GSAP.** `useEffect` can cause a visible flash before the animation fires.

**3. Animation timing hierarchy** — stagger order: KPI strip first (`delay: 0.2`), panels second (`delay: 0.4`). Never animate both at the same time.

**4. Ease vocabulary** — always use `'power3.out'` for entrances. Use `'power2.inOut'` for transitions between states (e.g., tab changes). Never use the default linear ease.

**5. Scroll-triggered reveals** — use `ScrollTrigger.create` for marina landing pages (hero sections, testimonials, booking CTAs). Operator dashboards do not use scroll-triggered animations.

---

## Layout Patterns

### Operator Dashboard Layout (2-column)

```css
.ops-layout {
  display: grid;
  grid-template-columns: 240px 1fr;
  min-height: 100vh;
}
@media (max-width: 900px) {
  .ops-layout { grid-template-columns: 1fr; }
}
```

Sidebar: `position: sticky; top: 0; height: 100vh; overflow-y: auto;`
Main content: `padding: 32px 40px; overflow-y: auto;`

### Marina Site Layout (full-width with section rhythm)

Full-bleed sections. No sidebar. Consistent section padding: `padding: 100px 0` desktop, `padding: 60px 0` mobile.

Max content width: `1280px`, centred. Navigation is sticky with `backdrop-filter: blur(12px)` and a `background: rgba(6,8,16,0.85)` — not opaque.

### GovTech Compliance Dashboard Layout (3-column on wide, 2-column on medium)

```css
.govtech-layout {
  display: grid;
  grid-template-columns: 200px 1fr 320px; /* nav | content | filing-sidebar */
}
@media (max-width: 1200px) {
  .govtech-layout { grid-template-columns: 200px 1fr; }
  /* filing sidebar becomes a bottom sheet on tablet */
}
```

---

## Multi-Page Marina Site Template

Every white-label marina deployment uses the same 12-page template. Clients customize content and `--tenant-primary-color`. They **never** modify template structure.

| # | Page | Layout Pattern |
| :--- | :--- | :--- |
| 1 | Homepage / Landing | Full-bleed hero + feature sections |
| 2 | About | Alternating image/text split |
| 3 | Marina & Facilities | Grid cards + slip map |
| 4 | Experiences / Charters | Card grid with booking CTA |
| 5 | Individual Experience | Detail page + sticky booking panel |
| 6 | Dining & Amenities | Photo-heavy editorial layout |
| 7 | Events | Calendar list + feature event hero |
| 8 | Gallery | Masonry grid |
| 9 | Contact & Directions | Map embed + contact form |
| 10 | Book Online | Multi-step booking flow |
| 11 | Booking Confirmation | Receipt layout |
| 12 | Guest Portal | Waivers, documents, trip details |

The booking flow (pages 10–12) is the only section connected to the ReefCore payment gateway. All other pages are static or near-static.

---

## GovTech Dashboard Conventions

GovTech has its own component language — same tokens, different density.

### Filing Status Cards

```tsx
// Filing deadlines use a countdown + urgency colour
type FilingUrgency = 'ok' | 'due-soon' | 'overdue';

// Colour mapping
const urgencyColor: Record<FilingUrgency, string> = {
  'ok':        'var(--positive)',
  'due-soon':  'var(--warn)',
  'overdue':   'var(--danger)',
};
```

Filing card layout: header shows **filing type** (NIB, VAT, Business Licence, HCA) + **due date** + **status badge**. Body shows last-submitted period, next-due period, and a single action CTA.

### NIB / VAT Data Tables

- Use `font-variant-numeric: tabular-nums` on all monetary columns so BSD amounts align.
- Currency column format: always `BSD $` prefix, 2 decimal places.
- Never show raw centavo/cent integers in the UI — always format before display.

```ts
// ✅ Always format BSD amounts before display
const formatBSD = (amount: number): string =>
  `BSD $${amount.toFixed(2).replace(/\B(?=(\d{3})+(?!\d))/g, ',')}`;

// ❌ Never display raw numbers
<td>{transaction.amount}</td>
```

### HCA Exemption Indicator

Transactions that were GBPA-exempt must display a distinct indicator — not just a status badge. Use a small `⬡` (hexagon) glyph in `var(--gold)` with tooltip text "GBPA Hawksbill Creek Exemption" on hover. This visually signals the HCA connection without needing a full label.

---

## Sidebar Navigation Conventions

Both the Operator Dashboard and GovTech dashboards use the same sidebar pattern:

- **Brand lockup** at top: `TENANT NAME` in Playfair Display caps, accent dot in `var(--gold)`, subtitle in uppercase Outfit 10px tracked.
- **Nav sections** with uppercase 9px labels (opacity 0.25) dividing groups.
- **Active state**: `border-left: 2px solid var(--gold)` + `color: var(--gold)` + `background: rgba(201,168,76,0.05)`.
- **Badge** on nav items with pending counts: `background: rgba(201,168,76,0.2); color: var(--gold)`.

```tsx
// ✅ Active nav item class pattern
className={`ops-nav-item${isActive ? ' active' : ''}`}
```

Never use a filled pill or boxed highlight for the active state. The left border is the only affordance.

---

## Grain Overlay

All full-bleed hero sections use a subtle noise/grain overlay for depth. This is a visual signature of the MarinerTemplate.

```css
.hero::after {
  content: '';
  position: absolute;
  inset: 0;
  background-image: url("data:image/svg+xml,..."); /* SVG noise filter */
  opacity: 0.03;
  pointer-events: none;
}
```

For React components where an inline SVG data URI is too verbose, use the `grain-overlay` utility class (defined in the global CSS layer):

```css
.grain-overlay { isolation: isolate; }
.grain-overlay::after { /* noise pattern */ opacity: 0.03; pointer-events: none; position: absolute; inset: 0; content: ''; }
```

---

## Responsive Breakpoints

```text
≥ 1280px  — Desktop full layout
≥ 1100px  — Operator dashboard KPI strip 5→3 columns
≥  900px  — Dashboard sidebar collapses
≥  700px  — Slip board grid 10→5 columns
<  640px  — Mobile: single column, reduced padding (24px horizontal)
```

**Always write mobile-first for marina pages, desktop-first for dashboards.** Marina sites are browsed on mobile by tourists. Dashboards are used on desktop by operators.

---

## Accessibility & Performance

- All interactive elements (buttons, nav items, CTAs) must have `:focus-visible` outlines using `outline: 2px solid var(--gold)`.
- Booking flow must be keyboard-navigable end-to-end.
- Marina site images: always use Next.js `<Image>` with explicit `width`/`height` and a `blurhash`-based placeholder.
- No layout shift on font load — use `font-display: swap` for Google Fonts and pre-connect to `fonts.gstatic.com`.

---

## What NOT to Do

```
❌ border-radius on panels or cards — sharp corners only
❌ Inter, Roboto, or system fonts anywhere
❌ Purple gradients, teal-on-white, or any generic SaaS palette
❌ Framer Motion or CSS keyframe entrances (use GSAP)
❌ gsap.from() without gsap.context()
❌ Hardcoded hex brand colours — always var(--gold) / var(--tenant-primary-color)
❌ Rounded pill badges for nav active states
❌ Displaying raw BSD amounts without formatting (toFixed(2) + comma separator)
❌ Adding new page types outside the 12-page template without architectural review
❌ Any animation on the Operator Dashboard triggered by scroll
```
