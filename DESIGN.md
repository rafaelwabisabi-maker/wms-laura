# WMS Laura — Design Guidelines

> Source of truth for visual decisions. Claude reads this before any UI work.

## Brand Essence
Mini warehouse management system for pet shop supply. Operational tool — clarity and speed over aesthetics. Brazilian user (Laura, sobrinha). Portuguese UI.

## Colors (from globals.css :root, shadcn theme)
| Token | HSL | Usage |
|---|---|---|
| `--primary` | `142.1 76.2% 36.3%` | Green — main actions, success |
| `--primary-foreground` | `355.7 100% 97.3%` | Text on primary |
| `--background` | `0 0% 100%` | White page bg |
| `--foreground` | `0 0% 3.9%` | Near-black text |
| `--muted` | `0 0% 96.1%` | Gray backgrounds |
| `--muted-foreground` | `0 0% 45.1%` | Gray text |
| `--destructive` | `0 84.2% 60.2%` | Red — errors, delete |
| `--border` | `0 0% 89.8%` | Light gray borders |
| `--ring` | `142.1 76.2% 36.3%` | Focus rings (green) |

**Chart colors:** chart-1 (#E76E50 orange) | chart-2 (#3B9E8B teal) | chart-3 (#2E4A5A dark) | chart-4 (#D4A54A gold) | chart-5 (#E08A4A amber)

## Typography
System defaults via shadcn/Tailwind. No custom fonts — keep it fast.

## Stack
- Next.js 15 + Tailwind CSS + shadcn/ui
- SQLite (Prisma) — local-first
- Dark mode support via `.dark` class

## Component Patterns
- shadcn components everywhere — no custom UI primitives
- Sidebar navigation (collapsible)
- Data tables for inventory, orders, picking
- Toast notifications for actions

## UX Principles
- Laura is NOT technical — every action needs clear labels in PT-BR
- Operational speed: 1-2 clicks to complete common tasks (pick item, update stock)
- Mobile-friendly — warehouse workers use phones
- Scan-first: barcode/QR integration expected

## Anti-Patterns
- No English labels in the UI — everything PT-BR
- No complex multi-step wizards — keep flows flat
- No decorative elements — this is a work tool, not a marketing site
- No tiny fonts — warehouse lighting is bad, minimum 14px body
