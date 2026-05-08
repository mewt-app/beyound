# beyound.in — Coming Soon Page Design Spec

## Overview

Single-page "coming soon" landing for beyound.in, a consumer-facing UPI + wallet payments fintech. Brand awareness only — no signups, no social links, no interactivity.

## Brand Identity

- **Name:** beyound
- **Domain:** beyound.in
- **Tagline:** "Payments, simplified"
- **Personality:** Clean, trustworthy, premium

## Visual Design

### Colors

| Token | Hex | Usage |
|-------|-----|-------|
| emerald-500 | #059669 | Primary accent, tagline text, gradient start |
| emerald-600 | #047857 | Logo gradient end, divider gradient end |
| emerald-700 | #065F46 | Hover states (future use) |
| slate-900 | #0F172A | Headline text |
| slate-700 | #334155 | "Coming Soon" text |
| slate-400 | #94A3B8 | Body text, footer |
| slate-300 | #CBD5E1 | Copyright text |
| white | #FFFFFF | Background |

Background has subtle radial gradients of emerald at ~4% opacity for depth.

### Typography

| Element | Font | Weight | Size | Tracking |
|---------|------|--------|------|----------|
| Brand name | Sora | 800 | 68px | -2px |
| Tagline | Sora | 500 | 13px | 4px, uppercase |
| "Coming Soon" | Sora | 600 | 24px | normal |
| Body text | Sora | 400 | 15px | normal |
| Copyright | Sora | 400 | 12px | normal |

Font loaded via Google Fonts: `Sora:wght@400;500;600;800`

### Logo Mark

- Circle, 80×80px
- `border-radius: 50%`
- Background: `linear-gradient(135deg, #059669, #047857)`
- Box shadow: `0 4px 24px rgba(5, 150, 105, 0.2)`
- Contains lowercase "b" in white, Sora 700, 36px

### Layout

Vertically and horizontally centered, single column. Order top-to-bottom:

1. Logo circle (80px)
2. Gap: 32px
3. Brand name "beyound" (68px)
4. Gap: 16px
5. Tagline "PAYMENTS, SIMPLIFIED" (13px uppercase, emerald)
6. Gap: 48px
7. Divider line (48px wide, 2px, emerald gradient)
8. Gap: 48px
9. "Coming Soon" (24px)
10. Gap: 12px
11. Subtext "We're building something new for the way you pay." (15px, max-width 400px)
12. Footer pinned to bottom: "© 2026 beyound. All rights reserved."

### Animations

- Fade-in + slight upward rise on page load (CSS only)
- Subtle pulse or glow on logo mark (optional, CSS only)

### Responsive

- Naturally responsive — centered column scales down
- Font sizes scale slightly on mobile (clamp or media query)
- Min padding: 24px horizontal on small screens

## Technical Spec

### Files

- `index.html` — single self-contained file with inline CSS
- No JavaScript
- No build tools
- No dependencies beyond Google Fonts CDN

### Meta Tags

- `<title>beyound — Payments, simplified</title>`
- `<meta name="description" content="beyound — a new way to pay. Coming soon.">`
- `<meta name="viewport" content="width=device-width, initial-scale=1">`
- `<meta charset="UTF-8">`
- Open Graph tags for social sharing (og:title, og:description, og:type)
- Theme color: `#059669`
- Favicon: inline SVG data URI (emerald circle with "b")

### Performance

- No JS = no render blocking
- Single HTTP request (plus font)
- Target: perfect Lighthouse score

### Deployment

Host anywhere — Netlify, Vercel, Cloudflare Pages, or basic static hosting. Single file upload.

## Out of Scope

- Email/waitlist collection
- Social media links
- Analytics/tracking
- Multiple pages
- Backend/API
