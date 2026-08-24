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

**First settle which page you are working on.** This comes before either route below. The page list (`?depth=1`, or `get_metadata` with no `nodeId`) is cheap, so always look at it. A URL with a `node-id` resolves the question — that frame's page is the page. Without one, or when the list holds several plausible candidates for the same feature — `Mobile` / `Desktop`, `v1` / `v2`, `Design` / `Archive` — **ask which, and offer to do more than one.** A feature split across `Mobile` and `Desktop` pages is ordinary; processing whichever page happened to sort first, silently, is how the spec ends up half a feature. When you do cover several pages, keep the frame inventory per page — Step 2 needs to know that two similar screens came from different pages rather than being a duplicate.

On REST, cap the depth and take only the top-level frame list of **that one page** — `GET /v1/files/{key}/nodes?ids={pageId}&depth=2`.

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

**This question must not become a dead end.** Running as a subagent, in a scheduled job, or anywhere else with nobody to answer, do not stop here — take the largest coherent group, process it, and put the choice in Extraction notes: which group you took, which you skipped, and that nobody confirmed it. A spec for one of three features plus a note saying so is useful. A halt waiting on an answer that will never come is not.

### Step 2: Group into features

Use these signals, in order:

1. **Naming conventions** — prefixes and separators like `해지_02_약관`, `Signup / Step 2`
2. **Canvas placement** — people lay related screens out in a row or a column. Proximity is often more reliable than names
3. **Prototype links** — decisive when you can get them, but they are not in what Step 1 fetched. Read below before relying on this
4. **Text inside the screens** — last resort, when the three above all fail

**On prototype links.** Nothing collected so far contains them. `get_metadata` returns geometry only, and `depth=2` from a page reaches the frames, not the buttons — and `reactions` / `transitionNodeID` sit on the *interactive node*, usually a button several levels down. So a link never appears by accident; you have to go get it.

Two ways, in this order:

- **`flowStartingPoints` on the page node**, from `GET /v1/files/{key}/nodes?ids={pageId}&depth=1`. Cheap, and it names the flows the designer marked — often the grouping answer on its own
- **Per-frame `reactions`**, which means raising `depth` on frames you already suspect belong together. Do this for a handful to confirm a grouping, never across all 34 frames

**Both of those are REST.** On the MCP route `get_metadata` returns geometry and nothing else, so signal 3 is not expensive there — it is *unavailable*. With no REST token, drop it and say so; do not spend calls hunting for links this route cannot return.

When you drop signal 3 for either reason — unavailable, or not worth its cost — fall back to signals 1, 2 and 4 and say in Extraction notes that the flow order was inferred rather than read off prototype links. Do not present an inferred order as a linked one.

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
- States the screen defines — including any drawn as a component variant, and separately any you suspect but could not confirm
- What kind of screen it is (`traits`), which is what selects the checklist sections in Step 4
- Conditional elements (hidden layers, badges, tooltips)

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
      next: "45:912"             # or "not defined" — see Step 2 on prototype links
  states_defined: [default]      # fixed vocabulary, below
  states_other: ["카드 선택됨"]   # screen-specific states, named as in the file
  states_unconfirmed: ["..."]    # suspected but not verified — never printed as defined
  traits: [list, form]           # fixed vocabulary, below
  conditional: ["..."]           # hidden layers, badges, tooltips
  repeats: "회선 카드 ×3, same structure"
  evidence: strong               # weak = layer names meaningless, read off the screenshot
  unread: "..."                  # why, if the frame could not be read at all
```

`states_defined` takes **only** these values:

```
default · loading · empty · error · unauthorized · maintenance · first_visit
```

That list is the one `references/gap-checklist.md` § 1 checks against, so Step 4 diffs it mechanically. Anything outside it — `카드 선택됨`, `[다음] 활성` — goes in `states_other` and is not diffed. Put a screen-specific state in `states_defined` and the gap analysis silently stops working.

The split exists for the diff, not for the reader. In the final document `states_defined` and `states_other` collapse back into one **States defined** line.

**A state drawn as a component variant is still a defined state.** Modern files define `error`, `empty`, and `loading` inside variant sets far more often than as separate frames — a `State=Error` variant, an `Empty` variant of a list component. Map those onto the fixed vocabulary and put them in `states_defined`, keeping the original name in `states_other`:

```yaml
states_defined: [default, empty]
states_other: ["List / State=Empty"]     # where the empty state was found
```

File them under `conditional` instead and Step 4 reports "error not defined" for a screen where the designer drew the error. That is the worst thing this skill can do — it sends the user to ask a question the file already answered, and one such entry costs the credibility of the whole `Needs Answer` list. `conditional` is for elements that appear or vanish *within* a state (badges, tooltips, hidden helper text), not for the state itself.

Variant sets do not show up as variants in a node tree — an instance node carries the selected variant's name, and the other variants live on the main component elsewhere in the file. So a frame using an `Error` variant may only reveal it through the component's name, and you often cannot tell whether this screen uses it.

That case goes in `states_unconfirmed`, not in either of the other two. Confirmed on the screen → `states_defined` (+ `states_other` for the name). Suspected from a component name → `states_unconfirmed`. Absent → leave it out and let Step 4's diff catch it.

**`states_unconfirmed` never reaches the States defined line.** The other two both assert something — "this state is defined", "defined, under this name". A state you merely suspect asserts neither, and filing it with them prints a guess as a fact, which is the one thing this skill must never do. It routes to `Needs Answer` 🟡 as a question — "the 회선 카드 component has a `Disabled` variant; is it used on this screen?" — and to Extraction notes.

`traits` takes **only** these values, and marks what *kind* of screen this is:

```
list · form · submit · money · permission · external
```

- `list` — repeated rows or cards backed by a collection
- `form` — any user input beyond a single tap
- `submit` — the screen commits something (an order, an application, a cancellation). Independent of `form`: a confirm-and-go screen with one button is `submit` and not `form`
- `money` — an amount, a fee, a balance, or a calculation is shown
- `permission` — visibility or content depends on login, role, or ownership
- `external` — an external app, payment, or auth provider is involved

Like `states_defined`, this exists so Step 4 can look sections up instead of judging them. `references/gap-checklist.md` § 2, 3, 6, 7 and 9 each hang off one of these, so a missing trait means a whole checklist section is silently skipped for that screen. Mark a trait when it plausibly applies — a false `list` costs one question the user skips, a missing one costs a section nobody notices was never asked.

Drop a key only when it has nothing in it. Do not fill one to look complete — a missing `states_defined` entry is exactly what Step 4 is hunting for. Every `evidence: weak` and every `unread` has to reach Extraction notes.

**On the MCP route this takes two calls per frame, minimum.** `get_metadata` gives you the skeleton — which text nodes exist, where, nested how. It does not give you a single character of what they say, so **Data fields displayed** and copy-vs-dummy cannot be filled from it. Call `get_design_context` on the frame for the actual strings.

Use the skeleton to aim. In files whose layer names are themselves a schema — `디스크립션 > 스크린 영역, 요소 정의 > 영역 정보 > {영역명, 영역 설명}` — `get_metadata` tells you which handful of text nodes carry the content, and `get_design_context` fills only those. That is much cheaper than pulling context for the whole frame and reading around it.

**Pull a screenshot only when you need to see it.** That means layer names are meaningless or the structure is ambiguous. Screenshotting every frame wastes tokens.

Describe repeated components (list items and the like) once, and note that they repeat.

### Step 4: Merge and find the gaps

After merging the batch results, read `references/gap-checklist.md`. That file only needs to be read at this step.

Do not decide from the prose which sections apply. **Look them up.** Each screen already carries the two fields that select its sections:

- `states_defined` → § 1, diffed against the fixed vocabulary
- `traits` → § 2, 3, 6, 7, 9, per the table at the top of the checklist
- § 4 and § 5 apply to every screen; § 8 applies to the feature as a whole, once

A section reached this way gets worked through. A section no screen selects gets skipped, and that is the whole judgment call — there is no third option where a section looked irrelevant.

When writing a gap, be specific about **which screen is missing what**. "Needs error handling" is useless. Write "Line list screen — nothing defined for a failed lookup."

**Then deduplicate the questions, the same way data fields are deduplicated.** Eight screens against nine sections generates the same question eight times, and a 60-item checklist gets read as carefully as an empty one — which is to say, not at all. One question, one entry, screens listed:

```
❌  - [ ] Screen 1: nothing defined for an error
    - [ ] Screen 2: nothing defined for an error
    - [ ] Screen 3: nothing defined for an error

✅  - [ ] Screens 1, 2, 3: nothing defined for a failed request. Is this one shared
          error treatment or three different ones?
```

Merge only questions with the same answer. Screen 2's "what happens when the fee lookup fails" is not Screen 1's generic error state — collapsing those loses the specific one, which was the more valuable of the two.

**Aim for about seven 🔴 per spec file.** Blockers are what the user takes into the meeting, and a list of twenty has no blockers in it. The count is per output file, so a file covering three features gets its own seven — split by feature before you start demoting.

If more than seven still survive after deduplication, re-read them: usually two or three are edge cases wearing a blocker's clothes, and moving those is the fix. **But do not demote a real blocker to hit the number.** A file that genuinely leaves twelve things unbuildable has twelve blockers; keep them, and say in Extraction notes that the count is unusually high and why. Losing one true blocker costs more than a list two items too long. 🟡 and 🟢 have no cap, but the same dedupe rule.

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

**States not defined** lists only what the diff found missing. A state you suspected but could not confirm belongs in neither line — it is a question, so it goes to `Needs Answer` and to Extraction notes. Putting it under "not defined" asserts it is absent, which is the same guess in the other direction.

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
- Call a state undefined without checking whether it exists as a component variant
- Ask the same question once per screen instead of listing the screens on one line
- Print a state you only suspect on the **States defined** line
- Describe dummy text as if it were real copy
- Include pixel values, colors, or fonts — Figma MCP hands those over directly at code-generation time. This document covers **what and why**, not **how it looks**
- Write with confidence about a grouping you are not sure of
