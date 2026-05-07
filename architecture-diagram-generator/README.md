# Architecture Diagram Generator

Self-contained HTML architecture diagrams. Dark theme, SVG-based, no dependencies.

## Quick Start

1. Copy `assets/template.html`
2. Edit the SVG: boxes, arrows, labels
3. Update the 3 cards at the bottom
4. Open in browser

## Color Code

| Type | Fill | Stroke |
|------|------|--------|
| Frontend | `rgba(8, 51, 68, 0.4)` | `#22d3ee` |
| Backend | `rgba(6, 78, 59, 0.4)` | `#34d399` |
| Database | `rgba(76, 29, 149, 0.4)` | `#a78bfa` |
| Cloud | `rgba(120, 53, 15, 0.3)` | `#fbbf24` |
| Security | `rgba(136, 19, 55, 0.4)` | `#fb7185` |

## Key Pattern

```svg
<!-- Opaque bg masks arrows -->
<rect x="X" y="Y" width="W" height="H" rx="6" fill="#0f172a"/>
<!-- Styled rect on top -->
<rect x="X" y="Y" width="W" height="H" rx="6" fill="rgba(6, 78, 59, 0.4)" stroke="#34d399" stroke-width="1.5"/>
<text x="CX" y="Y+20" fill="white" font-size="11" font-weight="600" text-anchor="middle">Label</text>
```

## Structure

- `assets/template.html` — Starting point
- `examples/` — Sample diagrams
- `architecture-diagram/SKILL.md` — Claude skill definition (optional)

## License

MIT
