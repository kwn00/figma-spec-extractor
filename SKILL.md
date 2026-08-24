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
| Inventory (Step 1) | no depth control — costly, see `access.md` | `depth=` does exactly this |
| Text inside the screens (Step 3) | `get_design_context` | `characters` on TEXT nodes |
| Screenshots | `get_screenshot` | `GET /v1/images` |

With both available, the good split is **REST for Step 1, MCP for Step 3**. With only one, you still need that route's caveats — each has a step it does badly, and neither fails loudly.

**Read [`references/access.md`](references/access.md) now, for the route or routes you have.** It carries the tool arguments, the endpoint table, the error codes, and the two constraints that shape the whole workflow. The URL rules below stay here because they apply whichever route you take.

If neither route is available, stop here and tell the user. Do not push ahead by guessing from a handful of screenshots.

**On the MCP route, confirm a subagent can actually see the Figma tools before you rely on one.** Step 1 hands the expensive call to a subagent, and a subagent that inherited no Figma tools fails the whole attempt rather than part of it. Ask one to report which Figma tools it has before giving it work. If none can see them, fall back to REST, or ask the user to name the frames — do not keep trying subagent types blindly.

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

## Workflow

### Step 1: Skim shallow, build an inventory

**Never request a page's entire node tree at once.** Real files run to tens of thousands of nodes and the context window blows instantly. Failing here is this skill's most common failure mode.

**First settle which page you are working on.** This comes before either route below. **A page list that comes back empty, or holding one entry like `📕 Cover` when the file plainly has more, is not the file's page list — it is a bad answer that looks like a good one.** When the URL carries a `node-id`, stop trusting the list and enter through that node directly; when it does not, tell the user what you got and ask which page they mean rather than processing the one page you were handed. The page list (`?depth=1`, or `get_metadata` with no `nodeId`) is cheap, so always look at it. A URL with a `node-id` resolves the question — that frame's page is the page. Without one, or when the list holds several plausible candidates for the same feature — `Mobile` / `Desktop`, `v1` / `v2`, `Design` / `Archive` — **ask which, and offer to do more than one.** A feature split across `Mobile` and `Desktop` pages is ordinary; processing whichever page happened to sort first, silently, is how the spec ends up half a feature. When you do cover several pages, keep the frame inventory per page — Step 2 needs to know that two similar screens came from different pages rather than being a duplicate.

On REST, cap the depth and take only the top-level frame list of **that one page** — `GET /v1/files/{key}/nodes?ids={pageId}&depth=2`.

**MCP has no depth cap, so isolate the call instead.** Hand `get_metadata` for the page to a subagent whose only job is to hand back the top-level frame list — ID, name, coordinates, size, nothing nested. The tens of thousands of tokens stay in the subagent and never reach you. If subagents are not available and there is no REST token either, say that plainly and get the user to name the specific frames they care about. Do not call `get_metadata` on a page and hope.

Record only this per frame: node ID, name, coordinates, size.

**Look for a change-log board.** Files often carry a `History`, `변경이력`, or `Change Log` board. Take the file version, the date, and the last three to five entries: the version and date go in the document header so a reader knows how fresh this is, and the recent entries tell you which parts of the file moved lately — which is exactly where the design and whatever exists in code are most likely to have drifted apart.

**Chapter covers are not frames to process.** A board carrying a big title and an author name and nothing else is a divider. Keep it out of the count — six covers in a sixty-six board file is six phantom screens with nothing in them — but keep the board itself, because its title is often the only plain-language name its group has.

**Watch the frame sizes for a storyboard file.** A page of 1920×1080 (or otherwise wide) frames holding what is plainly a mobile product is not a set of screens — it is a set of storyboard boards, each one a phone mockup beside a written specification table. Frame size is the only signal the inventory itself carries, and it is not enough to be sure. **Open one board as a sample and look**: layer names like `디스크립션`, `설명 텍스트`, `영역명`, `description`, `spec` settle it, and the sample also shows you how that file's description tables are built, which is what Step 3 needs. One board is cheap; guessing wrong changes what every later batch reads first. Note the answer on the inventory: it changes what Step 3 reads first, and it usually means the file answers far more than a screen-only file would.

Two things about the tree, both of which bite later:

- **Coordinates are relative to the parent.** Only the top-level frames carry absolute canvas coordinates — which is exactly what Step 2 needs, so read placement off *those* and not off anything nested
- **Hidden nodes are in the tree** (`hidden="true"`). Keep them out of the frame count here. They matter in Step 3 as conditional elements, not before

**Get names for the groups before you ask.** The example below reads well because `해지_01_회선선택` says what it is. Real files often name every board `main_01_2`, and a list of those tells the user nothing — they cannot choose, and neither can you. Read the chapter cover titles first, or open one board per group and read its heading. That one cheap round turns `main_01` into `홈 콘텐츠별 정의`, which is the difference between a question the user can answer and one they cannot.

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

Ask one more thing while you have their attention: **is this feature already built?** One word — shipped, in progress, or not started — changes who the output is for. Record it; Step 5 uses it. **No answer means treat it as built**, here as everywhere else in this step: the warning it triggers costs one line if you are wrong, and omitting it costs the credibility of the whole question list if you are.

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

**First decide what the board is.** A storyboard page is not a stack of screens — a good share of its boards are documents *about* the screens: flowcharts, exposure matrices, card-type definitions, button-logic tables, chapter covers. Sixty-six boards can hold six screens. Get this wrong and the rest of the extraction is filled in against the wrong shape.

Per frame, extract:

- What the board is (`kind`) and, if it carries one, its screen ID
- Screen name and purpose (one line)
- **Data fields displayed** — the most important item. This is what gets checked against the API spec later
- User actions (buttons, inputs, gestures) and the resulting screen for each, marking any that leave this feature entirely
- Real copy vs. dummy text (mark `Lorem ipsum`, `홍길동`, `Enter text here` as dummy) — **and watch for the file's own notation**: a value written `%8초%`, `명세서%1%건`, or `▲ %10,000%원` is using a convention this file invented. Find its legend; if there is none, do not decide whether it is a placeholder, a CMS value, or a hard-coded constant — raise the convention itself as a question. One of those turned out to be a front-end constant, not content, and the spec had no way to tell
- States the screen defines — including any drawn as a component variant, and separately any you suspect but could not confirm
- What kind of screen it is (`traits`), which is what selects the checklist sections in Step 4
- Who this screen is for and when it appears (`shown_when`) — a condition, not a route
- Domain words it uses (`terms`) and full sentences shown to the user (`copy`)
- Whatever the frame's description table says, if it has one — quoted, and read before the mockup
- Conditional elements (hidden layers, badges, tooltips)

**Every batch comes back in this shape** — subagent or sequential, same format. Step 4 merges these as they are, so a batch that invents its own layout has to be re-read.

```yaml
- node_id: "45:678"
  kind: screen                   # screen · definition · cover
  screen_id: "MW01"              # the ID printed on the board, if there is one
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
      leaves_flow: unknown       # true when the action ends this feature's flow
  states_defined: [default]      # fixed vocabulary, below
  states_other: ["카드 선택됨"]   # screen-specific states, named as in the file
  states_unconfirmed: ["..."]    # suspected but not verified — never printed as defined
  traits: [list, form]           # fixed vocabulary, below
  conditional: ["..."]           # hidden layers, badges, tooltips
  shown_when: "..."              # who sees this screen and when — "not defined" if the file never says
  terms: ["준회원", "결합"]        # domain words this screen uses, exactly as written
  copy: ["해지 시 남은 혜택이 사라집니다"]   # sentences shown to the user, quoted whole
  spec_notes: ["..."]            # quoted from the frame's description table, if it has one
  matrix: {...}                  # a grid the board defines — shape below
  repeats: "회선 카드 ×3, same structure"
  evidence: strong               # weak = layer names meaningless, read off the screenshot
  unread: "..."                  # why, if the frame could not be read at all
```

`states_defined` takes **only** these values:

```
default · loading · empty · error · unauthorized · maintenance · first_visit
```

That list is the one `references/gap-checklist.md` § 1 checks against, so Step 4 diffs it mechanically. Anything outside it — `카드 선택됨`, `[다음] 활성` — goes in `states_other` and is not diffed. Put a screen-specific state in `states_defined` and the gap analysis silently stops working.

**A user segment is never one of these values.** This is the most common way the vocabulary gets misused, and it is easy to do because the words nearly fit:

| The file shows | Not this | It is a |
|---|---|---|
| `준회원` | `empty` | segment → `shown_when` |
| `법인회원` | `unauthorized` | segment → `shown_when` |
| `일시정지` | `error` | contract status → `shown_when` |
| `알뜰폰` | `unauthorized` | product type → `shown_when` |

Membership grade, contract status, product type, tier — these say *who is looking*, not *what went wrong*. `unauthorized` is a screen that refuses access, not a screen that shows a different thing. A segment goes to `shown_when` and surfaces in **Who sees what**; calling it a state tells the reader the designer defined an error case nobody drew.

The pull toward this mistake is real: the variant rule above pushes you to promote anything state-shaped into `states_defined`, and there is no matching pressure in the other direction. This table is that pressure. When a candidate is a kind of person rather than a kind of moment, it is not a state.

The split exists for the diff, not for the reader. In the final document `states_defined` and `states_other` collapse back into one **States defined** line.

**A state drawn as a component variant is still a defined state.** Modern files define `error`, `empty`, and `loading` inside variant sets far more often than as separate frames — a `State=Error` variant, an `Empty` variant of a list component. Map those onto the fixed vocabulary and put them in `states_defined`, keeping the original name in `states_other`:

```yaml
states_defined: [default, empty]
states_other: ["List / State=Empty"]     # where the empty state was found
```

File them under `conditional` instead and Step 4 reports "error not defined" for a screen where the designer drew the error. That is the worst thing this skill can do — it sends the user to ask a question the file already answered, and one such entry costs the credibility of the whole `Needs Answer` list. `conditional` is for elements that appear or vanish *within* a state (badges, tooltips, hidden helper text), not for the state itself.

Variant sets do not show up as variants in a node tree — an instance node carries the selected variant's name, and the other variants live on the main component elsewhere in the file. So a frame using an `Error` variant may only reveal it through the component's name, and you often cannot tell whether this screen uses it.

That case goes in `states_unconfirmed`, not in either of the other two. Confirmed on the screen → `states_defined` (+ `states_other` for the name). Suspected from a component name → `states_unconfirmed`. Absent → leave it out and let Step 4's diff catch it.

**`states_defined` describes the whole screen, and only the whole screen.** It cannot say "card 3 has an error state, the screen does not" — the vocabulary has no room for it. On a screen assembled from independently loaded blocks that limitation matters, because failure there is per block: one card hides itself, another shows a retry, a third is replaced by a notice. Mark the screen `composed` and let § 11 of the checklist ask about the blocks; do not try to encode per-block states in `states_defined`.

**`states_unconfirmed` never reaches the States defined line.** The other two both assert something — "this state is defined", "defined, under this name". A state you merely suspect asserts neither, and filing it with them prints a guess as a fact, which is the one thing this skill must never do. It routes to `Needs Answer` 🟡 as a question — "the 회선 카드 component has a `Disabled` variant; is it used on this screen?" — and to Extraction notes.

`leaves_flow` takes `true`, `false`, or `unknown`. It marks an action that ends this feature — the user lands somewhere this spec does not cover. It changes what back means, where they resume, and whether they come back at all, so it is a product question and it belongs here. **How** the app gets there — an in-app route, a webview URL, a bridge call — is not: that is decided by the codebase, not by the design, and a spec that guesses at it is guessing. Leave it `unknown` unless the file says; an element that changes state in place takes neither `next` nor `leaves_flow`. `next` still names a frame in this file, so a leaving action usually carries `next: "not defined"` alongside `leaves_flow: true` — "we know it goes out, we do not know where" is a real and common state, and a useful thing to have said.

`traits` takes **only** these values, and marks what *kind* of screen this is:

```
list · form · submit · money · permission · external · native · composed
```

- `list` — repeated rows or cards backed by a collection
- `form` — any user input beyond a single tap
- `submit` — the screen commits something (an order, an application, a cancellation). Independent of `form`: a confirm-and-go screen with one button is `submit` and not `form`
- `money` — an amount, a fee, a balance, or a calculation is shown
- `permission` — visibility or content depends on login, role, or ownership
- `external` — an external app, payment, or auth provider is involved
- `native` — the screen reaches for the device: camera, biometrics, push permission, share sheet, file picker, shake, clipboard, location. **Not** for the header and tab bar every mobile frame draws — that ownership question is § 8, asked once for the feature, and marking every screen `native` for it buys seven questions per screen and answers none
- `composed` — the screen is an assembly of independently loaded blocks (cards, modules, widgets) rather than one thing that succeeds or fails as a unit

Like `states_defined`, this exists so Step 4 can look sections up instead of judging them. `references/gap-checklist.md` § 2, 3, 6, 7, 9, 10 and 11 each hang off one of these, so a missing trait means a whole checklist section is silently skipped for that screen. Mark a trait when it plausibly applies — a false `list` costs one question the user skips, a missing one costs a section nobody notices was never asked.

`kind` takes **only** these values:

```
screen · definition · cover
```

- `screen` — a board drawing something the user looks at. Everything else in this schema assumes this
- `definition` — a board *about* the product rather than a picture of it: a flowchart, an exposure matrix, a card-type table, button logic, a term list. It has no states, no `shown_when`, no `data_fields`, and **their absence is not a gap** — see Step 4
- `cover` — a chapter title board, usually a heading and an author name and nothing else. Keep it out of the frame count. Its title is often the only plain-language name a group of boards has, so read it

A `definition` board takes a reduced shape, plus whatever it defines:

```yaml
- node_id: "45:200"
  kind: definition
  name: "main_01_2"
  defines: "회선·요금제별 카드 노출 여부"   # one line: what this board settles
  matrix: {...}
  spec_notes: ["..."]
```

**Reduced does not mean lesser.** On an enterprise storyboard the definition boards usually carry more of the real specification than the screens do — the screens show one arrangement, the definition boards say which arrangement each customer gets. Give them the same care, and read their tables the same way you read a description table.

`screen_id` is whatever the board prints as its own identifier — `MW01`, `HOME-03`, `A-2-1`. It is the key that ties this document to the code, the QA cases, and the tracker, and it is the single most useful thing you can carry out of the file for someone checking work that already exists. Quote it exactly; never construct one.

`shown_when` is the screen's entry *condition*, not its entry *path*. § 5 asks which screens lead here; this asks who is allowed to see it at all — logged out, an 준회원 account, a suspended line, a first visit. A file that draws five near-identical screens named `main_03_1` … `main_03_5` is drawing one screen with five conditions, and a spec that lists them as five screens has thrown the condition away — the one thing a developer needs. Read the condition off the frame name, the description table, or the differences between the frames, and write `not defined` when none of those say.

`terms` collects the domain words the screen uses — `준회원`, `일시정지`, `결합`, `대표회선`, `당겨쓰기`. Take the word exactly as written and do not translate or gloss it. You are collecting the vocabulary, not defining it: a definition that is not in the file is not yours to write.

**A word can be both a term and a field label, and that is fine — the two ask different questions.** `회선번호` is a field and nothing else: you need to know where the value comes from. `위약금` is a field *and* a term: you need to know where the value comes from **and** what the word means, and the second is not answered by the first. Put a label in `terms` when a developer who did not know the word would build it wrong. Leave out labels that are self-evident from the value beside them.

`matrix` is a grid the board defines — the O/X table of which card shows for which plan, a state-by-role table, a button-logic table. **On a composed product this table is the specification**, and paraphrasing it into prose destroys it.

```yaml
matrix:
  title: "회선·요금제별 카드 노출"     # exactly as written on the board
  rows: ["개인 5G 무제한", "..."]      # exactly as written
  columns: ["데이터", "이번달 요금"]
  cells: [["O", "X"], ["O", "O"]]     # row-major, marks quoted as they appear
  complete: true
```

**Only fill `cells` from text you actually retrieved.** A 20 × 11 grid is 220 cells, and reading them off a rendered image produces a table that looks authoritative and is wrong — worse than no table, because nobody re-checks a table. If the cells did not come back as text, set `complete: false`, keep the rows and columns you do have, and say so; a named grid you could not read is a finding, not a failure to hide. Never infer a cell from the ones around it.

`copy` is the sentences the product says to the user, quoted whole and unedited — guidance text, warnings, empty-state lines, error messages, confirmations. Not labels and not data values; those are already in `data_fields` and `actions`. Mark dummy copy the same way you mark dummy data.

Drop a key only when it has nothing in it. Do not fill one to look complete — a missing `states_defined` entry is exactly what Step 4 is hunting for. Every `evidence: weak` and every `unread` has to reach Extraction notes.

**On the MCP route this takes two calls per frame, minimum.** `get_metadata` gives you the skeleton — which text nodes exist, where, nested how. It does not give you a single character of what they say, so **Data fields displayed** and copy-vs-dummy cannot be filled from it. Call `get_design_context` on the frame for the actual strings.

Use the skeleton to aim: `get_metadata` tells you which handful of text nodes carry content, and `get_design_context` fills only those, which is far cheaper than pulling context for the whole frame and reading around it.

**On a storyboard frame, read the description table before the mockup.** Not for cost — because that table is the spec, and the mockup is an illustration of it. Enterprise files routinely put the real product thinking there: when a block is hidden, which API feeds it, what happens on failure, which segments see what. Skim the phone mockup and you will send the user off to ask a question the file answered two columns to the right. That is the same failure as reporting a variant-drawn state as missing, and it is the more common of the two in files like this.

The order is skeleton, then table, then mockup — and the middle step is still aimed, not a whole-frame pull. `get_metadata` (or `depth=2`) shows you which subtree holds the table; fetch the strings for *those* nodes with `get_design_context` or `characters`, and let what they say drive what you then look for in the mockup. Pulling context for the entire board to find the table is the context explosion Step 1 exists to prevent — on these frames it is the single most expensive call in the workflow.

Carry what the table says into `spec_notes`, quoted rather than paraphrased. Anything it answers is **defined**: it goes in the spec body and must not reappear in `Needs Answer`.

**Boards contradict each other, not just themselves.** One board declares a list — "적용 콘텐츠 5종" — and another enumerates the items and shows two. Both statements get transcribed, both look authoritative, and the spec ships with a number that is wrong. This is a merge-time check and Step 4 does it; note the declaration here so it is available to compare against.

Where the table and the mockup disagree, neither wins silently. Record both and raise it — a table saying "결합 회선은 미노출" against a mockup drawing the block is a real question for the author, and often the most valuable thing in the extraction.

**Pull a screenshot only when you need to see it.** That means layer names are meaningless or the structure is ambiguous. Screenshotting every frame wastes tokens.

Describe repeated components (list items and the like) once, and note that they repeat.

### Step 3.5: Ask each batch three questions

Send every batch back once with three questions and nothing else. This is the only place in the workflow that spends a turn to buy accuracy, and it is spent here because these three mistakes were made by every extraction pass that was measured — not occasionally, but by all of them, working from the same instructions the batch already had. More instruction does not fix what instruction failed at; a narrow question that is not competing with twenty others does.

1. **For each state you listed in `states_defined` — is it actually drawn, as a frame or as a variant?** Point at where.
2. **Is any of them really a user segment?** `준회원`, `법인회원`, `일시정지`, `알뜰폰` are not `empty`, `unauthorized`, or `error`. Move any that are to `shown_when`.
3. **On a `composed` screen — is per-block failure defined anywhere, or did you infer it?**

Take back only corrections. A batch that answers "no changes" costs one short turn; the one that does not has just saved a wrong claim from reaching the document, and a wrong claim in `Needs Answer` costs the credibility of every other line in it.

### Step 4: Merge and find the gaps

After merging the batch results, read `references/gap-checklist.md`. That file only needs to be read at this step.

**Cross-check the boards against each other before diffing any of them.** Merging is the only moment anything sees all the boards at once, so it is the only moment a contradiction between two of them can be caught. Where one board declares a count or a list and another enumerates it, compare them. Where two boards describe the same card, screen, or rule, compare what they say. A disagreement is not a thing to average out or to quietly pick a side on: record both, quote both boards, and raise it 🔴 — the file itself does not know which is right, and neither do you.

**Diff only `kind: screen` boards.** A flowchart has no loading state and a chapter cover has no data fields, so running the screen diffs over them manufactures questions that cannot be answered and should never have been asked — "the flowchart does not define an error state" is the same failure as reporting a variant-drawn state as missing, and it is easier to produce by accident. On a `definition` board the absent fields are absent by nature, not by omission. What a definition board *can* be short of is its own content: a matrix with `complete: false`, or a `defines` line you could not fill.

**A `matrix` answers questions, so ask them only if it did not.** A grid that says which blocks appear for which segment has settled § 11's "where the block list and its order come from" and § 6's segment item, exactly the way `spec_notes` settles what it covers. Reproduce the grid and move on. Publishing the table and then asking what the table says is the same wasted question in a more embarrassing form.

**Apply `spec_notes` before you diff anything.** A frame that came back with a description table has already answered some of what the checklist is about to ask. Fold those answers into the screen's record first — a note saying "결합 회선 미노출" defines a conditional, "조회 실패 시 카드 숨김" defines an error behavior — and only then look for what is left. Skip this and the file's own answers get asked back to its author.

Do not decide from the prose which sections apply. **Look them up.** Each screen already carries the two fields that select its sections:

- `states_defined` → § 1, diffed against the fixed vocabulary
- `traits` → § 2, 3, 6, 7, 9, 10, 11, per the table at the top of the checklist
- `leaves_flow: true` or `unknown` → § 5's last item. Every action that ends the feature needs a destination and a return path, and `unknown` on a button that plainly goes somewhere is itself the question
- § 4 and § 5 apply to every screen; § 8 applies to the feature as a whole, once

Three more fields are diffed the same mechanical way, and each produces questions on its own:

- **`shown_when: not defined`** → ask it. A screen nobody can say who sees is a screen nobody can build the guard for. Two frames that differ only by condition and neither says which — 🔴
- **A `terms` entry with no definition** anywhere in the extraction → ask it. 🔴 when the term decides a branch (`준회원`, `일시정지`), 🟢 when it is only a label
- **A `data_fields` entry the screen computes with** — a gauge, a percentage, a threshold, an increase — where only the rendered string was extracted → ask for the raw value's type and unit. 🔴, because the API contract is designed off this document

Deduplicate these like everything else: one 준회원 question, not one per screen.

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

**A `composed` feature routinely runs eight to ten 🔴, and that is expected.** Composition policy alone — case selection, independent loading, per-block failure, order, user editing — produces five or six before anything else is counted. Do not squeeze those into seven; the count below is a target for an ordinary feature, not a ceiling to force.

**Aim for about seven 🔴 per spec file.** Blockers are what the user takes into the meeting, and a list of twenty has no blockers in it. The count is per output file, so a file covering three features gets its own seven — split by feature before you start demoting.

If more than seven still survive after deduplication, re-read them: usually two or three are edge cases wearing a blocker's clothes, and moving those is the fix. **But do not demote a real blocker to hit the number.** A file that genuinely leaves twelve things unbuildable has twelve blockers; keep them, and say in Extraction notes that the count is unusually high and why. Losing one true blocker costs more than a list two items too long. 🟡 and 🟢 have no cap, but the same dedupe rule.

### Step 5: Output

Save as `{feature-name}-spec.md`. Split into multiple files if there are multiple features.

**If you narrowed the scope, say what you left out.** Step 1 pushes hard toward narrowing and this is where that gets paid for: the boards you skipped are often the ones that answer the questions you are about to ask. Name the groups you excluded, and say plainly that answers to some of these questions are probably in them. A reader who knows that checks there first; a reader who does not takes the whole list to the designer.

**If the feature is already built, say so at the top of `Needs Answer`.** The section is addressed to the product owner and the designer, and that address is wrong the moment an implementation exists: developers will have answered a good share of these questions in code already, and questions with answers are exactly what destroys the list's credibility. You are not reading the code — this skill reads Figma — so do not try to say which ones. Say that some of them will be, and where to look first:

```markdown
## Needs Answer

⚠️ This feature is already implemented. Figma does not define the items below, but
the codebase may — check there before taking any of them to the product owner.
```

Keep the questions themselves unchanged. The warning is the whole fix: it costs one line and it stops the meeting where four of seven blockers turn out to have shipped months ago.

Like every other line of the output, it is written in the request's language — the English above is this document's language, not the spec's. See [Language](#language).

## Output format

Follow this template. Do not delete a section that has no content — write "Not defined" instead. An empty section is itself information.

The ⚠️ line below is written for a file of screens. When the file is a storyboard whose description tables carried real intent, say that instead — "Extracted from Figma storyboard boards; the description tables are quoted, the rest is read off the mockups" — because the default line understates what you had to work with, and readers use it to decide how much to trust the spec.

```markdown
# {Feature name}

> Source: {Figma link} · Extracted {date} · {N} frames
> ⚠️ Auto-extracted from Figma. Contains what was drawn, not what was intended.

## Overview
{2–3 sentences on what this feature does}

## Flow
{Entry point → screen order → exit. Mark branches}

## Screens

### 1. {Screen name} `{screen-id, if the board prints one}` · `{node-id}`
**Purpose**: {one line}
**Shown when**: {condition — omit the line when no screen in the feature states one}

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

## Who sees what
One row per condition, not per screen. Collapses the near-identical frames a file uses to draw one screen in several states.

| Condition | Screens | What differs |
|---|---|---|
| Logged out | 1 | Login prompt instead of the line list |
| 준회원 | 1, 2 | 해지명세서 shown, 위약금 hidden |
| Not defined | 3, 4 | — |

## Glossary
Domain words the design uses. **Definitions are not extracted** — a word with no definition in the file goes to `Needs Answer`, because a developer who guesses at it names the code wrong.

| Term | Where it appears | Definition |
|---|---|---|
| 준회원 | Screens 1, 2 | Not defined — see Needs Answer |
| 결합 | Screen 2 | "2회선 이상 묶음 할인" (from the description table) |

## Definition tables
Every grid the file defines, reproduced — not summarised. On a composed product these carry more of the specification than the screen descriptions do.

**회선·요금제별 카드 노출** (`main_01_2`)

| | 데이터 | 이번달 요금 |
|---|---|---|
| 개인 5G 무제한 | O | O |
| 개인 LTE | O | X |

A grid whose cells could not be retrieved as text says so — rows, columns, and "cells not retrieved" — and goes to `Needs Answer`. It never gets filled in by eye.

## Data requirements
Every data field across all screens, deduplicated. The list to check against the backend.

**Figma shows the rendered string, never the value behind it.** `25GB` and `4,930원` are display, and a spec that carries only display gets an API that returns strings and a client that parses them back with a regex. Where the screen does arithmetic on a value — a gauge, a percentage, a threshold, an increase — the raw value has to exist, so ask for it. **Never invent a field name**: the design does not contain one, and a developer will search for whatever you write.

| Field | Displayed as | Raw value needed | Screen | Note |
|---|---|---|---|---|
| 회선번호 | 010-1234-5678 | — | 1 | |
| 잔여 데이터 | 25GB | Yes — 80% gauge | 2 | Type and unit not defined |
| 위약금 | 42,000원 | Yes — summed | 2 | Calculation unconfirmed |

## Copy
Every sentence the product says to the user, deduplicated. The list to hand to whoever reviews wording.

| Text | Screen | Note |
|---|---|---|
| 해지 시 남은 혜택이 사라집니다 | 2 | |
| 내용을 입력해 주세요 | 3 | placeholder, likely dummy |

## Destinations
Every place an action sends the user, deduplicated — the counterpart of Data requirements for navigation. Scattered destinations are the ones that get missed.

| Destination | From | Leaves this feature |
|---|---|---|
| Screen 2 | [다음] on 1 | No |
| Not defined | `←` on 1, 2, 3 | ? |

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

An action that ends the feature says so in **Next screen** — `Leaves this feature — 약관 전문 (external)`, or `Leaves this feature — ?` when the file does not say where. No extra column: what the reader needs is that the flow stops here, and that fits where the destination already goes.

**`screen_id`** goes in the heading whenever the board printed one. It is the key a reader uses to find this screen in the code, the test cases, and the tracker, and carrying it costs nothing. Where a `Needs Answer` item is about one screen or one card, put that identifier in the question too — the reader can then check it in seconds instead of taking it to a meeting.

**Shown when** appears per screen only where it says something. When no screen in the feature states a condition, drop the line from every screen and say it once in **Who sees what** — four screens each repeating "Not defined" is the repetition the dedupe rule exists to stop, and it buries the one screen that *does* carry a condition when there is one.

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
- Run the screen diffs over a flowchart or a cover and report what it "lacks"
- Fill a matrix cell you could not read as text
- Write a definition for a term the file never defines
- Invent an API field name — the design has none, and the reader will search for whatever you write
- Read a storyboard's phone mockup and skip the description table beside it
- Call a state undefined without checking whether it exists as a component variant
- Ask the same question once per screen instead of listing the screens on one line
- Print a state you only suspect on the **States defined** line
- Describe dummy text as if it were real copy
- Include pixel values, colors, or fonts — Figma MCP hands those over directly at code-generation time. This document covers **what and why**, not **how it looks**
- Write with confidence about a grouping you are not sure of
