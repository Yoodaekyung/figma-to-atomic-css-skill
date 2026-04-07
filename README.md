# Figma → Atomic CSS Skill

Claude Code skill that converts Figma designs to HTML using **only Atomic CSS classes**. No inline styles, no CSS files — class names are the styles.

## How it works

```
/figma https://www.figma.com/design/xxxxx?node-id=123-456
```

1. Reads Figma design via Figma MCP
2. Maps every visual property to Atomic CSS classes
3. Outputs semantic HTML with responsive variants
4. Self-validates all classes via Atomic CSS MCP

## Requirements

- **Figma MCP** — Claude Desktop built-in or manually configured
- **Atomic CSS MCP** (recommended) — for real-time class validation

```json
{
    "mcpServers": {
        "atomic-css": {
            "type": "sse",
            "url": "https://mcp.atomiccss.dev/sse"
        }
    }
}
```

## Installation

```
/plugin marketplace add Yoodaekyung/figma-skill
/plugin install figma@Yoodaekyung-figma-skill
```

Or clone directly:

```bash
git clone https://github.com/Yoodaekyung/figma-skill.git
```

## Key Features

- **Color accuracy** — Figma HEX values used exactly, never rounded
- **Spacing snap** — Non-4x values snapped to nearest 4px multiple
- **Grid-first layout** — Figma Auto Layout → CSS Grid
- **9-step post-validation** — every class verified before output
- **Responsive** — auto-adds mobile/tablet breakpoints

## Links

- [Atomic CSS](https://atomiccss.dev)
- [Atomic CSS Skill](https://github.com/Yoodaekyung/atomic-css-skill)
- [VS Code Extension](https://marketplace.visualstudio.com/items?itemName=Drangon-Knight.atomicss)

## License

MIT
