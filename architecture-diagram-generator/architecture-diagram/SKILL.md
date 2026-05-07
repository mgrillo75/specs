---
name: architecture-diagram
description: Create dark-themed architecture diagrams as self-contained HTML files with inline SVG.
---

# Architecture Diagram Generator

Output: Single HTML file with embedded CSS, inline SVG, no JS.

## Color System

| Type | Fill | Stroke |
|------|------|--------|
| Frontend | `rgba(8, 51, 68, 0.4)` | `#22d3ee` |
| Backend | `rgba(6, 78, 59, 0.4)` | `#34d399` |
| Database | `rgba(76, 29, 149, 0.4)` | `#a78bfa` |
| Cloud/AWS | `rgba(120, 53, 15, 0.3)` | `#fbbf24` |
| Security | `rgba(136, 19, 55, 0.4)` | `#fb7185` |
| External | `rgba(30, 41, 59, 0.5)` | `#94a3b8` |

## Visual Rules

- **Background:** `#020617` with 40px grid pattern
- **Font:** JetBrains Mono from Google Fonts
- **Component boxes:** `rx="6"`, 1.5px stroke, semi-transparent fill
- **Arrows first:** Draw all arrows before components (z-order)
- **Arrow masking:** Draw opaque `fill="#0f172a"` rect before styled rect
- **Legend:** Outside all boundaries, 20px below lowest element

## Component Pattern

```svg
<rect x="X" y="Y" width="W" height="H" rx="6" fill="#0f172a"/>
<rect x="X" y="Y" width="W" height="H" rx="6" fill="FILL" stroke="STROKE" stroke-width="1.5"/>
<text x="CX" y="Y+20" fill="white" font-size="11" font-weight="600" text-anchor="middle">Name</text>
<text x="CX" y="Y+36" fill="#94a3b8" font-size="9" text-anchor="middle">Detail</text>
```

## Arrow Pattern

```svg
<marker id="arrowhead" markerWidth="10" markerHeight="7" refX="9" refY="3.5" orient="auto">
  <polygon points="0 0, 10 3.5, 0 7" fill="#64748b" />
</marker>
<line x1="X1" y1="Y1" x2="X2" y2="Y2" stroke="#64748b" stroke-width="1.5" marker-end="url(#arrowhead)"/>
```

## Output Structure

1. Header with pulsing dot and title
2. SVG diagram (viewBox ~900x400-680)
3. 3 summary cards
4. Footer

## Template

See `assets/template.html` for base structure.
