# Route-Aware Shimmer Skeleton Loader Implementation Guide (HTML / CSS / React / Jinja)

This guide explains **in full detail** how to replace generic spinner/dots loading screens with **route-aware Shimmer Skeleton Loaders** that match each page's exact layout, container width, and spacing — eliminating layout jumps when real content loads.

---

## 1. Core Shimmer CSS (`static/css/hirfa-shimmer.css`)

Create one shared CSS file and include it in your base HTML templates (`base.html`, `spa_shell.html`, `admin/index.html`, `admin_panel/base.html`):

```css
/* Shimmer Loader (matches Flutter ShimmerWidget design tokens) */
:root {
  --shimmer-base: rgba(158, 158, 158, 0.14);
  --shimmer-highlight: #f6f1ec; /* Match your page background */
}

[data-theme="dark"],
.dark {
  --shimmer-base: rgba(255, 255, 255, 0.08);
  --shimmer-highlight: #141210; /* Match dark page background */
}

@keyframes hirfa-shimmer {
  0% {
    background-position: 200% 0;
  }
  100% {
    background-position: -200% 0;
  }
}

.hirfa-shimmer {
  display: block;
  background: linear-gradient(
    90deg,
    var(--shimmer-base) 25%,
    var(--shimmer-highlight) 50%,
    var(--shimmer-base) 75%
  );
  background-size: 200% 100%;
  animation: hirfa-shimmer 1.2s ease-in-out infinite;
  border-radius: 8px;
}

/* RTL support: sweep right-to-left */
[dir="rtl"] .hirfa-shimmer {
  animation-direction: reverse;
}
```

### Include in HTML `<head>`:
```html
<link rel="stylesheet" href="/static/css/hirfa-shimmer.css?v=1">
```

---

## 2. Critical Rules to Prevent Layout Jump & Header Overlap

When replacing a full-page spinner with a Shimmer skeleton, **never use a single generic skeleton for all pages**, and follow these 3 layout rules:

1. **Match the exact container class** used by your real pages (e.g. `container mx-auto px-4 md:px-6` — do NOT use `max-w-6xl` if the real page uses `container`, or the columns will jump horizontally when loaded).
2. **Match the exact vertical padding** of the real page (`py-8 md:py-20`, `py-8 md:py-12`, etc.).
3. **Respect fixed/floating headers** in embedded/SPA wrappers:
   Set `paddingTop: 'var(--hirfa-header-h, 88px)'` on the loading overlay container so the top of the shimmer is never clipped under the navbar.

---

## 3. Route-Aware React SPA Shimmer (`frontend/shop/src/ui.tsx`)

Instead of a single `LoadingBlock` with a spinner, create a `ShimmerPageSkeleton` that inspects `path` (`location.pathname`) and renders a skeleton matching that specific page's real DOM structure:

```tsx
import React from 'react'

function Box({
  className = '',
  radius = 8,
  style = {},
}: {
  className?: string
  radius?: number | string
  style?: React.CSSProperties
}) {
  return (
    <span
      className={`hirfa-shimmer ${className}`}
      style={{ borderRadius: radius, ...style }}
    />
  )
}

function ProductGridShimmer({ count = 8 }: { count?: number }) {
  return (
    <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-4">
      {Array.from({ length: count }).map((_, i) => (
        <div key={i} className="rounded-2xl border border-border/40 bg-card p-3 space-y-2.5">
          <Box className="w-full aspect-square" radius={12} />
          <Box className="h-3.5 w-4/5" />
          <Box className="h-3 w-1/2" />
        </div>
      ))}
    </div>
  )
}

function HomeShimmer() {
  return (
    <div className="container mx-auto px-4 md:px-6 py-6 md:py-10 space-y-8">
      <Box className="h-44 md:h-72 w-full" radius={24} />
      <div className="flex items-center justify-between gap-4 overflow-hidden py-2">
        {Array.from({ length: 7 }).map((_, i) => (
          <div key={i} className="flex flex-col items-center gap-2 shrink-0">
            <Box className="w-16 h-16 md:w-20 md:h-20" radius={9999} />
            <Box className="h-3 w-14" />
          </div>
        ))}
      </div>
      <Box className="h-6 w-44" />
      <ProductGridShimmer count={4} />
    </div>
  )
}

function ProductDetailShimmer() {
  return (
    <div className="container mx-auto px-4 md:px-6 py-8 md:py-12 space-y-6">
      {/* Breadcrumb */}
      <Box className="h-4 w-64" radius={8} />
      <div className="grid md:grid-cols-2 gap-8 md:gap-12 items-start">
        <div className="space-y-4">
          <Box className="w-full aspect-square" radius={24} />
          <div className="grid grid-cols-4 gap-3">
            {[0, 1, 2, 3].map((i) => (
              <Box key={i} className="w-full aspect-square" radius={12} />
            ))}
          </div>
        </div>
        <div className="space-y-5 pt-1">
          <Box className="h-6 w-28" radius={9999} />
          <Box className="h-8 w-4/5" radius={10} />
          <Box className="h-4 w-36" radius={8} />
          <Box className="h-8 w-32" radius={10} />
          <Box className="h-24 w-full" radius={16} />
          <Box className="h-12 w-36" radius={14} />
          <div className="flex gap-3 pt-1">
            <Box className="h-14 flex-1" radius={9999} />
            <Box className="h-14 w-14 shrink-0" radius={16} />
          </div>
        </div>
      </div>
    </div>
  )
}

function CartShimmer() {
  return (
    <div className="container mx-auto px-4 md:px-6 py-8 md:py-20 space-y-6 md:space-y-12">
      <Box className="h-9 md:h-12 w-48" radius={12} />
      <div className="grid lg:grid-cols-3 gap-6 md:gap-10 lg:gap-16">
        <div className="lg:col-span-2 space-y-5">
          {[0, 1].map((g) => (
            <div key={g} className="rounded-2xl md:rounded-3xl border border-border/40 bg-card overflow-hidden">
              <div className="px-5 py-3.5 bg-muted/30 border-b border-border/40 flex justify-between items-center">
                <div className="flex items-center gap-2">
                  <Box className="w-7 h-7" />
                  <Box className="h-4 w-28" />
                </div>
                <Box className="h-3.5 w-20" />
              </div>
              <div className="p-4 sm:p-5 flex items-center gap-4">
                <Box className="w-20 h-20 sm:w-24 sm:h-24 shrink-0" radius={14} />
                <div className="flex-1 space-y-2.5">
                  <Box className="h-4 w-3/4" />
                  <Box className="h-3.5 w-24" />
                  <Box className="h-8 w-28" radius={12} />
                </div>
                <Box className="h-5 w-16 hidden md:block" />
              </div>
            </div>
          ))}
        </div>
        <div className="rounded-2xl md:rounded-[2.5rem] border border-border/40 bg-card p-6 md:p-10 space-y-5 h-fit">
          <Box className="h-7 w-36" />
          <div className="flex justify-between"><Box className="h-4 w-20" /><Box className="h-4 w-16" /></div>
          <div className="flex justify-between"><Box className="h-4 w-16" /><Box className="h-4 w-24" /></div>
          <Box className="h-12 w-full" radius={14} />
          <div className="h-px bg-border/40" />
          <div className="flex justify-between"><Box className="h-6 w-24" /><Box className="h-7 w-24" /></div>
        </div>
      </div>
    </div>
  )
}

function CheckoutShimmer() {
  return (
    <div className="container mx-auto px-4 md:px-6 py-8 md:py-20 space-y-6 md:space-y-12">
      <Box className="h-9 md:h-12 w-40 hidden md:block" radius={12} />
      <div className="grid md:grid-cols-3 gap-6 md:gap-10">
        <div className="md:col-span-2 space-y-6">
          <div className="rounded-2xl md:rounded-[2.5rem] border border-border/40 bg-card p-6 md:p-10 space-y-4">
            <Box className="h-6 w-44" />
            <div className="grid sm:grid-cols-2 gap-4">
              <Box className="h-20 w-full" radius={14} />
              <Box className="h-20 w-full" radius={14} />
            </div>
          </div>
          <div className="rounded-2xl md:rounded-3xl border border-border/40 bg-card p-6 md:p-8 space-y-4">
            <Box className="h-5 w-36" />
            <Box className="h-16 w-full" radius={16} />
          </div>
        </div>
        <div className="rounded-2xl md:rounded-[2.5rem] border border-border/40 bg-card p-6 md:p-10 space-y-4">
          <Box className="h-7 w-36" />
          {[0, 1, 2].map((i) => (
            <div key={i} className="flex items-center gap-3">
              <Box className="w-12 h-12 shrink-0" radius={10} />
              <div className="flex-1 space-y-1.5">
                <Box className="h-3.5 w-3/4" />
                <Box className="h-3 w-12" />
              </div>
              <Box className="h-4 w-14" />
            </div>
          ))}
          <div className="h-px bg-border/40 my-2" />
          <div className="flex justify-between"><Box className="h-5 w-24" /><Box className="h-6 w-20" /></div>
        </div>
      </div>
      <div className="rounded-2xl md:rounded-3xl border border-border/40 bg-card p-6 md:p-8 space-y-4">
        <Box className="h-6 w-48" />
        <div className="grid grid-cols-2 md:grid-cols-4 gap-3">
          {[0, 1, 2, 3].map((i) => (
            <Box key={i} className="h-20 w-full" radius={14} />
          ))}
        </div>
      </div>
    </div>
  )
}

function ShopsShimmer() {
  return (
    <div className="container mx-auto px-4 md:px-6 py-8 md:py-16 space-y-6">
      <Box className="h-9 w-44" radius={12} />
      <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-5">
        {Array.from({ length: 6 }).map((_, i) => (
          <div key={i} className="rounded-3xl border border-border/40 bg-card overflow-hidden">
            <Box className="h-32 w-full" radius={0} />
            <div className="p-4 flex items-center gap-3">
              <Box className="w-14 h-14 shrink-0" radius={9999} />
              <div className="flex-1 space-y-2">
                <Box className="h-4 w-3/4" />
                <Box className="h-3 w-1/2" />
              </div>
            </div>
          </div>
        ))}
      </div>
    </div>
  )
}

function CatalogShimmer() {
  return (
    <div className="container mx-auto px-4 md:px-6 py-8 md:py-16 space-y-6">
      <div className="flex flex-wrap items-center justify-between gap-4">
        <Box className="h-9 w-44" radius={12} />
        <div className="flex gap-2">
          <Box className="h-10 w-24" radius={9999} />
          <Box className="h-10 w-24" radius={9999} />
          <Box className="h-10 w-24" radius={9999} />
        </div>
      </div>
      <ProductGridShimmer count={8} />
    </div>
  )
}

function DashboardShimmer() {
  return (
    <div className="container mx-auto px-4 md:px-6 py-8 md:py-12 space-y-6">
      <Box className="h-36 w-full" radius={24} />
      <div className="grid grid-cols-1 sm:grid-cols-3 gap-4">
        <Box className="h-28 w-full" radius={20} />
        <Box className="h-28 w-full" radius={20} />
        <Box className="h-28 w-full" radius={20} />
      </div>
      <Box className="h-56 w-full" radius={20} />
    </div>
  )
}

export function ShimmerPageSkeleton({
  path = '/',
  label = 'Loading…',
}: {
  path?: string
  label?: string
}) {
  const p = (path || '/').toLowerCase()
  let content: React.ReactNode
  if (p === '/' || p === '') content = <HomeShimmer />
  else if (p === '/cart' || p.startsWith('/cart/')) content = <CartShimmer />
  else if (p === '/checkout' || p.startsWith('/checkout/')) content = <CheckoutShimmer />
  else if (p.startsWith('/product/')) content = <ProductDetailShimmer />
  else if (p === '/shops' || p.startsWith('/shops/')) content = <ShopsShimmer />
  else if (p.startsWith('/dashboard')) content = <DashboardShimmer />
  else content = <CatalogShimmer />

  return (
    <div role="status" aria-label={label}>
      {content}
      <span className="visually-hidden">{label}</span>
    </div>
  )
}
```

### Usage in SPA Page Wrapper (`ClassicEmbedPage.tsx`):
```tsx
{busy ? (
  <div
    className="spa-embed-loading w-full h-full overflow-hidden"
    style={{ paddingTop: 'var(--hirfa-header-h, 88px)' }}
    aria-live="polite"
  >
    <ShimmerPageSkeleton path={location.pathname} label="Loading" />
  </div>
) : null}
```

---

## 4. Inline HTML / JS Shimmer (for Async API Sections)

For server-rendered HTML pages that fetch data via `fetch()` (e.g., Cart items, Recommendations, Checkout summary, Infinite Scroll, Map):

### A. Initial HTML Placeholder (zero spinner flash before JS runs):
```html
<div id="cart-items" class="space-y-4 md:space-y-6">
  <!-- Shimmer Skeleton Loader -->
  <div class="bg-card rounded-2xl md:rounded-3xl border border-border/40 shadow-sm overflow-hidden">
    <div class="px-4 sm:px-6 py-3.5 bg-muted/30 border-b border-border/40 flex items-center justify-between gap-3">
      <div class="flex items-center gap-2">
        <span class="hirfa-shimmer w-7 h-7" style="border-radius: 8px;"></span>
        <span class="hirfa-shimmer h-4 w-28"></span>
      </div>
      <span class="hirfa-shimmer h-3.5 w-20"></span>
    </div>
    <div class="p-3 sm:p-5 flex items-center gap-3 sm:gap-6">
      <span class="hirfa-shimmer w-16 h-16 sm:w-24 sm:h-24 shrink-0" style="border-radius: 14px;"></span>
      <div class="flex-1 space-y-2.5">
        <span class="hirfa-shimmer h-4 w-3/4"></span>
        <span class="hirfa-shimmer h-3.5 w-24"></span>
        <span class="hirfa-shimmer h-8 w-28" style="border-radius: 12px;"></span>
      </div>
    </div>
  </div>
</div>
```

### B. Dynamic JS Shimmer Before `fetch()`:
```javascript
grid.innerHTML = Array.from({ length: 6 }).map(() => `
  <div class="bg-card rounded-2xl border border-border/50 shadow-sm overflow-hidden flex flex-col">
    <span class="hirfa-shimmer w-full aspect-square" style="border-radius: 0;"></span>
    <div class="p-3 sm:p-3.5 space-y-2.5">
      <span class="hirfa-shimmer h-3.5 w-4/5"></span>
      <span className="hirfa-shimmer h-3 w-1/2"></span>
      <div class="flex items-center justify-between pt-1">
        <span class="hirfa-shimmer h-4 w-14"></span>
        <span class="hirfa-shimmer h-7 w-14" style="border-radius: 9999px;"></span>
      </div>
    </div>
  </div>
`).join('');
```

---

## 5. Admin / Dashboard Shimmer (`Bootstrap` / `frontend/admin/src/ui.tsx`)

For Bootstrap-based admin or seller dashboards, replace `DotsLoader` inside `LoadingBlock` with KPI cards + table row shimmers:

```tsx
export function ShimmerAdminSkeleton({ label = 'Loading…' }: { label?: string }) {
  return (
    <div className="container-fluid py-3" role="status" aria-label={label}>
      <div className="row g-3 mb-4">
        {[0, 1, 2, 3].map((i) => (
          <div key={i} className="col-6 col-md-3">
            <div className="card border-0 shadow-sm rounded-4 p-3">
              <span className="hirfa-shimmer mb-2" style={{ height: 14, width: '55%', borderRadius: 6 }} />
              <span className="hirfa-shimmer" style={{ height: 28, width: '75%', borderRadius: 8 }} />
            </div>
          </div>
        ))}
      </div>
      <div className="card border-0 shadow-sm rounded-4 p-4">
        <span className="hirfa-shimmer mb-3" style={{ height: 20, width: '32%', borderRadius: 8 }} />
        {[0, 1, 2, 3, 4].map((r) => (
          <div key={r} className="d-flex align-items-center gap-3 py-2">
            <span className="hirfa-shimmer" style={{ height: 40, width: 40, borderRadius: 10, flexShrink: 0 }} />
            <div className="flex-grow-1">
              <span className="hirfa-shimmer mb-1" style={{ height: 14, width: '65%', borderRadius: 6 }} />
              <span className="hirfa-shimmer" style={{ height: 12, width: '40%', borderRadius: 6 }} />
            </div>
            <span className="hirfa-shimmer" style={{ height: 24, width: 72, borderRadius: 999 }} />
          </div>
        ))}
      </div>
      <span className="visually-hidden">{label}</span>
    </div>
  )
}

export function LoadingBlock({ label = 'Loading…' }: { label?: string }) {
  return <ShimmerAdminSkeleton label={label} />
}
```

> **Note on Action Buttons:** Keep small inline spinners (`size="sm"`) **only** on submit buttons (e.g., "Saving...", "Processing order...") — use Shimmer skeletons for **all** page loads, cards, grids, tables, and maps.
