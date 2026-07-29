# figma-spec-extractor

An agent skill that reads a Figma page and turns it into a spec a developer can start building from.

Figma MCP answers "how do I render this frame." This skill handles the step before that —
**"what is this feature, and what has nobody defined yet?"**

## Why

Design handoff doesn't break at the pixels. It breaks because the product thinking is scattered
across screens, and because nobody ever drew the empty list, the error, or the permission denial.
The developer finds out three weeks later.

The most important part of this skill's output is not the screen summaries. It's the **`Needs Answer`** section.

## Install

```bash
npx skills add kwn00/figma-spec-extractor
```

## Requirements

A Figma MCP server **or** a Figma REST API token.

## Usage

```
Pull a spec out of this page
https://figma.com/design/abc123/service?node-id=45-678
```

The spec comes back in whatever language you asked in. Text quoted from the design —
screen names, button labels, data values — stays exactly as it appears in the file,
so you can still search for it in Figma.

## Output

```
{feature-name}-spec.md
├── Overview
├── Flow
├── Screens (data displayed / actions / states defined / states not defined)
├── Data requirements   ← the list to check against your backend
├── Needs Answer        ← 🔴 blockers / 🟡 edge cases / 🟢 worth confirming
└── Extraction notes
```

## Limits

- It doesn't invent what isn't in Figma. Instead of guessing, it sends the question to `Needs Answer`
- Accuracy drops on files where every layer is named `Frame 12`. In that case it says the evidence is weak
- It doesn't touch pixels, colors, or fonts. Figma MCP hands those over directly at code-generation time

## Roadmap

- [ ] OpenAPI spec cross-check — find fields on screen that no API provides
- [ ] Better automatic frame grouping
- [ ] Claude Code plugin manifest — once there are several skills, or hooks/MCP to bundle

## License

MIT
