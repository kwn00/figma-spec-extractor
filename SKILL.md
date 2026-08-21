---
name: figma-spec-extractor
description: Reads a whole Figma page and extracts it into a feature-level product spec (markdown), then surfaces every state, exception, and data requirement the design never defined as a "Needs Answer" list. Use this whenever someone gives a Figma URL or file key and asks for a spec, PRD, requirements write-up, or documentation, or wants context assembled before turning a design into code. Especially useful when one feature is scattered across many frames rather than contained in a single one. If Figma is mentioned and the deliverable is closer to "a document, an understanding" than to "code", reach for this skill first. Also triggers on Korean requests such as "이 화면들 정리해줘", "피그마 보고 스펙 뽑아줘", "무슨 기능인지 파악해줘", "기획서로 뽑아줘".
---

# Figma Spec Extractor

Turn a Figma page into a spec a developer can actually start building from.

## What this skill really solves

Figma holds **only what was drawn**. What was not drawn — the empty list, the failed request, the user without permission — is nowhere in the file. Yet development cannot proceed without knowing it.

So the most important part of this skill's output is not the screen descriptions. It is the **`## Needs Answer`** section. What the user actually gets is *knowing today what they would otherwise discover three weeks in*. Nail the screen summaries but do the gap analysis carelessly, and this skill has failed.

**Never fill a blank with a guess.** Inventing a plausible state that Figma does not define is worse than doing nothing — the user will read it as settled and move on. When you do not know, send it to `Needs Answer`.

## Before you start

Confirm how you can reach the file. **Neither route covers the whole job well**, so check for both before starting rather than committing to one.

| | Figma MCP | REST API |
|---|---|---|
| Inventory (Step 1) | no depth control — costly, see below | `depth=` does exactly this |
| Text inside the screens (Step 3) | `get_design_context` | `characters` on TEXT nodes |
| Screenshots | `get_screenshot` | `GET /v1/images` |

With both available, the good split is **REST for Step 1, MCP for Step 3**. With only one, read that route's caveats below — each has a step it does badly, and neither fails loudly.

If neither is available, stop here and tell the user. Do not push ahead by guessing from a handful of screenshots.

### Parsing the URL

`figma.com/design/{fileKey}/{name}?node-id={nodeId}`

Older files use `/file/` in place of `/design/` — same shape, same fileKey position. `/proto/` links carry a fileKey too.

**Branch URLs**: `figma.com/design/{fileKey}/branch/{branchKey}/{name}`. Pass **`branchKey`** as the file key. The key in front of `/branch/` points at the main file, not at the branch the user linked.

`/board/` is FigJam and `/slides/` is Slides. Neither is a design file — `get_metadata` and the node endpoints return nothing useful. Say so and stop, unless screenshots alone are enough for what was asked.

**On REST, convert the node ID before you send it.** The URL writes it with a hyphen, the API takes a colon:

```
node-id=45-678   →   45:678
```

Skip this and the API answers `{"nodes": {}}`. That is not an error — it is an empty result that reads exactly like an empty page. It isn't one.

This is a REST rule only. MCP takes either form.

### Figma MCP

Tools are prefixed `mcp__plugin_figma_figma__`. Four of them matter here:

| Tool | Returns | Watch for |
|---|---|---|
| `get_metadata` | node tree as XML — `id · type · name · x · y · width · height` | **No text content.** No depth limit |
| `get_design_context` | reference code, screenshot, and metadata for one node | Requires loading its own `figma-design-to-code` guidance first — the tool says so and means it |
| `get_screenshot` | PNG render | Works on `/design/`, `/board/`, `/slides/` alike |
| `get_variable_defs` | variables and tokens | |

Arguments are `fileKey` (required) and `nodeId` (optional). Nothing depends on what is selected in the desktop app, so a URL is enough on its own. Responses do report the user's current selection at the top — useful for confirming you and the user are looking at the same screen, not something to depend on.

**Omit `nodeId` and `get_metadata` returns the file's top-level page list** instead of a tree. That is the entry point when you do not yet know which page you want. Never send an empty string or an ID you guessed.

`get_metadata` is `/design/` only.

Two constraints shape the whole workflow on this route:

**1. `get_metadata` has no depth parameter.** It returns the entire recursive subtree, every time. A single 1920×1080 frame measured 424 nodes and 56 KB at 13 levels deep; a page holding thirty of those is far past what the main context can take. See Step 1 for how to call it anyway.

**2. `get_metadata` does not return text.** It gives you the layer's *name*, never the string inside it:

```xml
<text id="5060:78819" name="영역명 텍스트" x="29" y="0" width="170" height="25" />
```

That says a text node exists and nothing whatsoever about what it says. Since **Data fields displayed** is the single most important thing this skill extracts, `get_metadata` alone can never complete Step 3. See Step 3.

### REST API

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

## Workflow

### Step 1: Skim shallow, build an inventory

**Never request a page's entire node tree at once.** Real files run to tens of thousands of nodes and the context window blows instantly. Failing here is this skill's most common failure mode.

On REST, cap the depth and take only the top-level frame list of **the one page you are working on** — `GET /v1/files/{key}/nodes?ids={pageId}&depth=2`.

**MCP has no depth cap, so isolate the call instead.** Hand `get_metadata` for the page to a subagent whose only job is to hand back the top-level frame list — ID, name, coordinates, size, nothing nested. The tens of thousands of tokens stay in the subagent and never reach you. If subagents are not available and there is no REST token either, say that plainly and get the user to name the specific frames they care about. Do not call `get_metadata` on a page and hope.

Record only this per frame: node ID, name, coordinates, size.

Two things about the tree, both of which bite later:

- **Coordinates are relative to the parent.** Only the top-level frames carry absolute canvas coordinates — which is exactly what Step 2 needs, so read placement off *those* and not off anything nested
- **Hidden nodes are in the tree** (`hidden="true"`). Keep them out of the frame count here. They matter in Step 3 as conditional elements, not before

Then show the user the inventory and confirm scope:

```
Found 34 frames on this page.

Likely groups:
A. 회선 해지 (7) — "해지_01_회선선택", "해지_02_약관", ...
B. 명의 변경 (5) — ...
C. Unclassifiable (9) — "Frame 12", "Rectangle 47", ...

Process all of them, or just one group?
```

Frame names are quoted **exactly as they appear in the file** — never translated. See [Language](#language).

Past 40 frames, **push** the user to narrow the scope. Narrow and accurate beats broad and shallow.

### Step 2: Group into features

Use these signals, in order:

1. **Naming conventions** — prefixes and separators like `해지_02_약관`, `Signup / Step 2`
2. **Canvas placement** — people lay related screens out in a row or a column. Proximity is often more reliable than names
3. **Prototype links** — the strongest signal when present. They hand you the flow directly
4. **Text inside the screens** — last resort, when the three above all fail

Files where every layer is named `Frame 12` or `Rectangle 47` are common. That is closer to the default than to an exception. In that case lean on coordinates and screenshots, and **state plainly in the output that the grouping rests on weak evidence.** Do not fake confidence.

### Step 3: Extract frame by frame (in batches)

This is the loop. Split frames into **batches of 5–8**.

- **If subagents are available**, run batches in parallel. Hand each subagent a list of frame IDs, the extraction items below, and the return format below. Take back only that. The point is to keep raw node trees out of the main context.
- **If not**, go sequentially, appending intermediate results to a file after each batch and discarding the raw data.

Per frame, extract:

- Screen name and purpose (one line)
- **Data fields displayed** — the most important item. This is what gets checked against the API spec later
- User actions (buttons, inputs, gestures) and the resulting screen for each
- Real copy vs. dummy text (mark `Lorem ipsum`, `홍길동`, `Enter text here` as dummy)
- Conditional elements (hidden layers, variants, badges, tooltips)

**Every batch comes back in this shape** — subagent or sequential, same format. Step 4 merges these as they are, so a batch that invents its own layout has to be re-read.

```yaml
- node_id: "45:678"
  name: "해지_01_회선선택"        # exactly as in the file, never translated
  purpose: "..."
  data_fields:
    - label: "회선번호"           # exactly as in the file
      sample: "010-1234-5678"
      dummy: true                # omit entirely when the value looks real
  actions:
    - element: "[다음]"           # exactly as in the file
      behavior: "..."
      next: "45:912"             # or "not defined"
  states_defined: [default]      # fixed vocabulary, below
  states_other: ["카드 선택됨"]   # screen-specific states, named as in the file
  conditional: ["..."]           # hidden layers, variants, badges, tooltips
  repeats: "회선 카드 ×3, same structure"
  evidence: strong               # weak = layer names meaningless, read off the screenshot
  unread: "..."                  # why, if the frame could not be read at all
```

`states_defined` takes **only** these values:

```
default · loading · empty · error · unauthorized · maintenance · first_visit
```

That list is the one `references/gap-checklist.md` § 1 checks against, so Step 4 diffs it mechanically. Anything outside it — `카드 선택됨`, `[다음] 활성` — goes in `states_other` and is not diffed. Put a screen-specific state in `states_defined` and the gap analysis silently stops working.

The split exists for the diff, not for the reader. In the final document the two collapse back into one **States defined** line.

Drop a key only when it has nothing in it. Do not fill one to look complete — a missing `states_defined` entry is exactly what Step 4 is hunting for. Every `evidence: weak` and every `unread` has to reach Extraction notes.

**On the MCP route this takes two calls per frame, minimum.** `get_metadata` gives you the skeleton — which text nodes exist, where, nested how. It does not give you a single character of what they say, so **Data fields displayed** and copy-vs-dummy cannot be filled from it. Call `get_design_context` on the frame for the actual strings.

Use the skeleton to aim. In files whose layer names are themselves a schema — `디스크립션 > 스크린 영역, 요소 정의 > 영역 정보 > {영역명, 영역 설명}` — `get_metadata` tells you which handful of text nodes carry the content, and `get_design_context` fills only those. That is much cheaper than pulling context for the whole frame and reading around it.

**Pull a screenshot only when you need to see it.** That means layer names are meaningless or the structure is ambiguous. Screenshotting every frame wastes tokens.

Describe repeated components (list items and the like) once, and note that they repeat.

### Step 4: Merge and find the gaps

After merging the batch results, read `references/gap-checklist.md` and work through it item by item. That file only needs to be read at this step.

When writing a gap, be specific about **which screen is missing what**. "Needs error handling" is useless. Write "Line list screen — nothing defined for a failed lookup."

### Step 5: Output

Save as `{feature-name}-spec.md`. Split into multiple files if there are multiple features.

## Output format

Follow this template. Do not delete a section that has no content — write "Not defined" instead. An empty section is itself information.

```markdown
# {Feature name}

> Source: {Figma link} · Extracted {date} · {N} frames
> ⚠️ Auto-extracted from Figma. Contains what was drawn, not what was intended.

## Overview
{2–3 sentences on what this feature does}

## Flow
{Entry point → screen order → exit. Mark branches}

## Screens

### 1. {Screen name} `{node-id}`
**Purpose**: {one line}

**Data displayed**
| Field | Sample value | Note |
|---|---|---|
| 회선번호 | 010-1234-5678 | |
| 요금제명 | 5G 시그니처 | |
| 위약금 | 42,000원 | likely dummy |

**Actions**
| Element | Behavior | Next screen |
|---|---|---|
| [다음] | Go to reason selection | Screen 2 |
| [취소] | Not defined | ? |

**States defined**: default, loading
**States not defined**: error, empty

## Data requirements
Every data field across all screens, deduplicated. The list to check against the backend.

| Field | Screen | Note |
|---|---|---|
| 회선번호 | 1 | |
| 위약금 | 2 | calculation unconfirmed |

## Needs Answer

Questions for the product owner and designer. Ordered by what must be answered before development can start.

### 🔴 Blockers — cannot build without this
- [ ] {Screen}: {question}

### 🟡 Edge cases — you will hit these during implementation
- [ ] {Screen}: {question}

### 🟢 Worth confirming
- [ ] {Screen}: {question}

## Extraction notes
{Where grouping rested on weak evidence, frames that could not be read, values assumed to be dummy}
```

Note the table above: the prose is English while `회선번호`, `5G 시그니처`, and `[다음]` stay exactly as they appear in the design. That is the rule, not an inconsistency — see below.

A filled-in spec is in [`references/example-spec.md`](references/example-spec.md), written in Korean for a Korean request. Read it only if this template leaves the shape unclear — the two together are the same document in two languages, which is the point.

## Language

**This document being written in English says nothing about what language to write the output in.** Do not let it pull you toward English. Decide as follows:

**Default: write the spec in the language the user made the request in.** Korean request → Korean spec, English request → English spec. Match the section headings too, including the template above (`Needs Answer` → `확인 필요`, `Blockers` → `블로커`, and so on).

**If the user names a language, that wins.** Otherwise do not ask — just go with the default.

**Never translate text quoted from the design.** Screen names, layer names, button labels, copy, and data values are identifiers the developer will use to search the Figma file. Translate them and that link breaks.

```
Design: [다음] button, 요금제명 field, "5G 시그니처"

✅  Tapping [다음] moves to reason selection. The 요금제명 field shows "5G 시그니처".
❌  Tapping [Next] moves to reason selection. The plan name field shows "5G Signature".
```

The sentence around it follows the request language. What sits inside the quotes does not.

## Do not

- Request an entire node tree at once (context explosion)
- Guess at states that were never drawn
- Describe dummy text as if it were real copy
- Include pixel values, colors, or fonts — Figma MCP hands those over directly at code-generation time. This document covers **what and why**, not **how it looks**
- Write with confidence about a grouping you are not sure of
