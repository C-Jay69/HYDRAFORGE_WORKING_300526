# Design Direction — M&A Contract Review Platform

## Vibe
Dark, professional, authoritative. Feels like a premium legal-tech SaaS — not a toy.
Think: Bloomberg Terminal meets modern SaaS. Dense information, zero clutter.

## Typography
- Font: Poppins (headings), Inter (body/mono data)
- Heading scale: 2.5rem / 2rem / 1.5rem / 1.25rem
- Body: 0.9rem, line-height 1.6
- Labels/meta: 0.75rem uppercase tracking-wide

## Color Palette (CSS Variables)
- `--bg-primary`: #0a0d14 (near-black navy)
- `--bg-secondary`: #111520 (card backgrounds)
- `--bg-tertiary`: #1a1f2e (elevated surfaces, borders)
- `--border`: #252b3b
- `--text-primary`: #e8ecf4
- `--text-secondary`: #8b93a8
- `--text-muted`: #545d72
- `--accent-gold`: #d4a843 (primary accent — risk scores, CTAs)
- `--accent-gold-dim`: #a07c2a
- `--risk-low`: #22c55e (green)
- `--risk-moderate`: #eab308 (yellow)
- `--risk-high`: #f97316 (orange)
- `--risk-critical`: #ef4444 (red)

## Layout
- Max content width: 1200px, centered
- Sidebar nav: 220px fixed left
- Main content: fluid right
- Cards: sharp corners (radius 6px), 1px border, no heavy shadows
- Spacing: 8px base unit

## Backgrounds
- App shell: solid `--bg-primary`
- Cards: `--bg-secondary` with `--border` border
- Active states: subtle `--bg-tertiary` highlight

## Motion
- Page transitions: 150ms fade
- Analysis progress: pulsing step indicators
- Report reveal: staggered section fade-in

## Score Badge Colors
- 90–100: `--risk-low` (green)
- 75–89: `--risk-moderate` (yellow)  
- 50–74: `--risk-high` (orange)
- 0–49: `--risk-critical` (red)
