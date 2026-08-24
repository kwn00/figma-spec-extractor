# Gap checklist

Read this at step 4 only. Check the extracted screens against the items below, and send anything Figma never defined to `Needs Answer`.

## Which sections apply

Not every section applies to every screen — attaching a pagination question to a static information page is noise. But do not judge that from the prose. Each screen's Step 3 record already says which sections it selects:

| Section | Selected by |
|---|---|
| 1. Screen states | `states_defined` — diff against the fixed vocabulary |
| 2. Lists and tables | `traits: list` |
| 3. Forms and input | `traits: form` |
| 4. Data | every screen |
| 5. Flow and navigation | every screen |
| 6. Permissions and auth | `traits: permission` |
| 7. Business rules | `traits: submit` or `traits: money` |
| 8. Platform and app shell | the feature as a whole, once — not per screen |
| 9. External integrations | `traits: external` |
| 10. Device capabilities | `traits: native` |
| 11. Composed screens | `traits: composed` |

Work through every section a screen selects. A section nothing selects is skipped. If a screen's record has no `traits` key at all, that is a Step 3 omission, not a screen with no traits — infer the traits from its `data_fields` and `actions` before continuing, and note it in Extraction notes.

Within a selected section, still skip individual items that plainly cannot apply. The section-level choice is the lookup; the item-level one is yours.

**Terms are collected, never defined.** If the extraction carries a domain word with no definition attached, that is a gap and it belongs in `Needs Answer` — a blocker when the word decides a branch. Writing a plausible definition yourself is the same failure as inventing a state.

**Check `spec_notes` before asking anything.** A storyboard frame's description table often answers checklist items outright. An item it answers is not a gap — record the answer in the spec body and move on. Asking the file's author what the file already says is the fastest way to lose them.

**Anything added to this file later hangs off a trait too.** § 1, 4 and 5 fire on every screen, so an item put there is an item every screen pays for; a rate-limit question on a confirmation screen is the noise this whole mechanism exists to prevent. A new concern gets a new trait and its own section, not a few more lines in § 1.

## How to prioritize

- 🔴 **Blocker** — no answer, no code (branch conditions, where required data comes from)
- 🟡 **Edge case** — you will hit it during implementation (errors, empty states)
- 🟢 **Worth confirming** — fine to fix later (long text, accessibility)

Aim for roughly seven 🔴 per output file — but never demote a real blocker just to hit that. And **one question, one entry** — when the same question falls out of five screens, list the screens on a single line rather than repeating it five times. See Step 4.

---

## 1. Screen states  —  diff of `states_defined`

Applies to nearly every screen. The most common omission by far.

Diff the fixed vocabulary against `states_defined`, and **trust that field over your own reading of the screen.** A state drawn as a component variant is defined even though no separate frame exists for it; Step 3 is responsible for having caught that. Reporting a state as missing when the designer drew it is worse than missing one — it costs the reader's trust in every other item here.

**A user segment is not a state.** Membership grade, contract status, product type, tier — `준회원`, `법인회원`, `일시정지`, `알뜰폰` — are *who the viewer is*, and a screen that shows them different content is not a screen in an error state. `unauthorized` is for a screen that refuses access, never for one that shows something else instead. Segments belong to `shown_when` and to **Who sees what**; mapping them onto this vocabulary is how a spec ends up claiming the designer defined an error state nobody drew.

These are whole-screen states. If the screen is `composed`, its blocks fail one at a time and § 11 asks about that — do not stretch the items below to cover it.

Anything in `states_unconfirmed` is neither defined nor missing. Raise it here as a 🟡 question — "the list component has an `Empty` variant; is it used on this screen?" — and keep it out of the **States defined** line.

- [ ] Loading — skeleton or spinner, whole screen or partial
- [ ] Empty — when the list has zero items
- [ ] Error — server failure, network loss
- [ ] Unauthorized — access is refused: not logged in, not the owner of this account
- [ ] Under maintenance / outside service hours
- [ ] First visit vs. return visit

## 2. Lists and tables  —  `traits: list`

- [ ] Pagination style (load more / infinite scroll / page numbers)
- [ ] How many per fetch
- [ ] Sort options and the default
- [ ] Behavior when filters combine, and when a filter returns zero results
- [ ] Upper bound on list length
- [ ] Pull to refresh

## 3. Forms and input  —  `traits: form`

- [ ] Required vs. optional
- [ ] Validation rules (length, format, range)
- [ ] Error message wording and where it appears
- [ ] When validation fires (as you type / on blur / on submit)
- [ ] Warning on leaving mid-edit, or draft saving
- [ ] Keyboard type, autocomplete, input masks

## 4. Data

- [ ] The raw value behind any number the screen computes with — its type, unit, and precision. Figma only ever shows the rendered string
- [ ] What shows when a value is null or empty
- [ ] Number formatting — thousands separators, decimals, currency symbol placement
- [ ] Date formatting, and the threshold for relative time ("3 minutes ago")
- [ ] Time zone handling
- [ ] Text overflow — truncate or wrap, and after how many lines
- [ ] Value ceilings (999+ and the like)
- [ ] Fallback when an image fails to load

## 5. Flow and navigation

- [ ] Every path that leads into this screen
- [ ] Who is allowed to see it once they get there, and what a user who is not sees instead
- [ ] Back behavior — especially from a completion screen
- [ ] Where the user resumes after leaving mid-flow
- [ ] Whether deep-link entry is allowed
- [ ] Where the user lands after success
- [ ] Cancel and close behavior (does it need a confirmation modal)
- [ ] Which actions leave this feature entirely, and whether the user is expected back

## 6. Permissions and auth  —  `traits: permission`

- [ ] Access while logged out
- [ ] Session expiry
- [ ] Whether identity verification is required, and at what point
- [ ] Screen differences by role
- [ ] Whether the screen's content differs by segment and not just by role — plan, tier, contract type, account status (composition itself: § 11)

## 7. Business rules  —  `traits: submit` · `money`

The area most often missing wholesale from a spec. You can never learn it from the screens alone.

- [ ] Conditions that block this action (unpaid balance, already applied, outside the eligible window)
- [ ] When blocked, is the button hidden or disabled, and is a reason shown
- [ ] Whether processing takes time (immediate / next day / under review)
- [ ] Whether it can be cancelled or withdrawn, and by when
- [ ] How duplicate submissions are handled, and the button's state while one is in flight
- [ ] How amounts are calculated, and the rounding rule

## 8. Platform and app shell  —  once per feature

- [ ] Responsive breakpoints (only mobile screens exist — what about desktop?)
- [ ] iOS / Android differences
- [ ] Dark mode
- [ ] Accessibility — screen reader labels, anything conveyed by color alone

Every mobile frame in Figma draws a status bar, a header, and a tab bar, and in a hybrid app most of that is not the web layer's to build. The file cannot show the seam, so ask once for the whole feature — not per screen, since the answer does not change between them:

- [ ] Which parts of these frames the webview renders and which the native shell owns (status bar, header, tab bar / GNB, floating buttons)
- [ ] Whether what the shell shows changes from screen to screen — the title, a badge, whether a back arrow is there at all
- [ ] Who owns the back gesture, and what back means on the first screen of the flow

## 9. External integrations  —  `traits: external`

- [ ] Returning from an external app
- [ ] Payment or auth abandoned midway
- [ ] Timeout thresholds and what shows when one is hit
- [ ] Whether a retry is possible

## 10. Device capabilities  —  `traits: native`

A screen that asks for the camera, the fingerprint reader, or the push permission has edges the design almost never draws: the user says no, the user cancels, the device cannot do it. Whoever builds it decides how — a web API, a native bridge, a separate screen — and that is their call, not a question for the designer. What the user *sees* at each of those edges is the question, and it is undrawn.

- [ ] What the screen shows when the permission is refused, and whether there is a way back from that
- [ ] Where the user lands if they cancel or abandon the capability mid-use
- [ ] What the calling screen refreshes when the capability returns a result
- [ ] Whether the capability is optional — can the user complete this flow without it

## 11. Composed screens  —  `traits: composed`

A screen assembled from independently loaded blocks does not have one loading state or one error state, and § 1's vocabulary cannot express that. Ask per block.

- [ ] Whether blocks load independently with their own skeletons, or the screen waits for all of them
- [ ] What a single block's failed request does — hide the block, show an inline retry, replace it with a notice, or block the whole screen
- [ ] Whether different failures are treated differently (a scheduled-maintenance notice is not a network error)
- [ ] Whether a quota or rate limit exists on any block, and what it shows when hit
- [ ] Where the block list and its order come from — fixed in the design, or served
- [ ] When served: what the user sees if that response is missing or partial — a default arrangement, or nothing
- [ ] Which blocks are mandatory, and whether the user can reorder or hide the rest — and where that is stored
