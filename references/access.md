# Reaching the file

Read this once, at the start, for whichever route you are on. The URL rules that apply to both — file key, branch key, the node-ID hyphen — stay in SKILL.md because they bite on every route.

## Figma MCP

Tools are prefixed `mcp__plugin_figma_figma__`. Four of them matter here:

| Tool | Returns | Watch for |
|---|---|---|
| `get_metadata` | node tree as XML — `id · type · name · x · y · width · height` | **No text content.** No depth limit |
| `get_design_context` | reference code, screenshot, and metadata for one node | Requires loading its own `figma-design-to-code` guidance first — the tool says so and means it |
| `get_screenshot` | PNG render | Works on `/design/`, `/board/`, `/slides/` alike |
| `get_variable_defs` | variables and tokens | |

Arguments are `fileKey` (required) and `nodeId` (optional). Nothing depends on what is selected in the desktop app, so a URL is enough on its own. Responses do report the user's current selection at the top — useful for confirming you and the user are looking at the same screen, not something to depend on.

**Omit `nodeId` and `get_metadata` returns the file's top-level page list** instead of a tree. That is the entry point when you do not yet know which page you want. Never send an empty string or an ID you guessed — and do not assume the list is complete; Step 1 says what to do when it comes back empty or holding a single entry.

`get_metadata` is `/design/` only.

Two constraints shape the whole workflow on this route:

**1. `get_metadata` has no depth parameter.** It returns the entire recursive subtree, every time. A single 1920×1080 frame measured 424 nodes and 56 KB at 13 levels deep; a page holding thirty of those is far past what the main context can take. See Step 1 for how to call it anyway.

**2. `get_metadata` does not return text.** It gives you the layer's *name*, never the string inside it:

```xml
<text id="5060:78819" name="영역명 텍스트" x="29" y="0" width="170" height="25" />
```

That says a text node exists and nothing whatsoever about what it says. Since **Data fields displayed** is the single most important thing this skill extracts, `get_metadata` alone can never complete Step 3. See Step 3.

## REST API

**Token**: check the environment for `FIGMA_TOKEN` first. Ask the user only if it is not there — and when you ask, say they can set the variable instead of pasting the token into the conversation. Header is `X-Figma-Token: {token}`. Never echo it back, print it inside a command you show the user, or write it into a file.

Getting to the right page takes two calls. The node ID in the URL usually points at a frame, not at the page holding it.

| What you need | Call |
|---|---|
| Page list | `GET /v1/files/{key}?depth=1` |
| Frames on one page (Step 1) | `GET /v1/files/{key}/nodes?ids={pageId}&depth=2` |
| One frame's subtree (Step 3) | `GET /v1/files/{key}/nodes?ids=45:678&depth=2` |
| Screenshot | `GET /v1/images/{key}?ids=45:678&format=png` |

**Do not reach for `GET /v1/files/{key}?depth=2` to build the inventory.** It returns the top-level frames of *every* page in the file. On a file with ten pages that is the context explosion Step 1 exists to prevent.

`ids` is required on `/nodes` — without it the call 400s rather than returning the whole file. Batch IDs comma-separated, and raise `depth` only for the frames you are actually reading.

When a call fails, these three account for nearly all of it:

- `403` — token missing, wrong, or without access to this file. Not worth a retry
- `404` — bad file key, or the token's account cannot see the file
- `200` with `{"nodes": {}}` — the node ID did not resolve. Almost always the hyphen, above
