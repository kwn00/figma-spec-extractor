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
| 8. Platform | the feature as a whole, once — not per screen |
| 9. External integrations | `traits: external` |
| 10. Native and webview boundary | `traits: native` |
| 11. Composed screens | `traits: composed` |

Work through every section a screen selects. A section nothing selects is skipped. If a screen's record has no `traits` key at all, that is a Step 3 omission, not a screen with no traits — infer the traits from its `data_fields` and `actions` before continuing, and note it in Extraction notes.

Within a selected section, still skip individual items that plainly cannot apply. The section-level choice is the lookup; the item-level one is yours.

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

These are whole-screen states. If the screen is `composed`, its blocks fail one at a time and § 11 asks about that — do not stretch the items below to cover it.

Anything in `states_unconfirmed` is neither defined nor missing. Raise it here as a 🟡 question — "the list component has an `Empty` variant; is it used on this screen?" — and keep it out of the **States defined** line.

- [ ] Loading — skeleton or spinner, whole screen or partial
- [ ] Empty — when the list has zero items
- [ ] Error — server failure, network loss
- [ ] Unauthorized — not logged in, not the account owner, insufficient tier
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

- [ ] What shows when a value is null or empty
- [ ] Number formatting — thousands separators, decimals, currency symbol placement
- [ ] Date formatting, and the threshold for relative time ("3 minutes ago")
- [ ] Time zone handling
- [ ] Text overflow — truncate or wrap, and after how many lines
- [ ] Value ceilings (999+ and the like)
- [ ] Fallback when an image fails to load

## 5. Flow and navigation

- [ ] Every path that leads into this screen
- [ ] Back behavior — especially from a completion screen
- [ ] Where the user resumes after leaving mid-flow
- [ ] Whether deep-link entry is allowed
- [ ] Where the user lands after success
- [ ] Cancel and close behavior (does it need a confirmation modal)
- [ ] For each action, which kind of navigation it is — in-app route, webview URL, or native bridge call

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

## 8. Platform  —  once per feature

- [ ] Responsive breakpoints (only mobile screens exist — what about desktop?)
- [ ] iOS / Android differences
- [ ] Dark mode
- [ ] Accessibility — screen reader labels, anything conveyed by color alone

## 9. External integrations  —  `traits: external`

- [ ] Returning from an external app
- [ ] Payment or auth abandoned midway
- [ ] Timeout thresholds and what shows when one is hit
- [ ] Whether a retry is possible

## 10. Native and webview boundary  —  `traits: native`

Figma draws one canvas. A hybrid app does not render one canvas — a native shell owns some of those pixels and a webview owns the rest, and the file cannot show you where the seam is. Nothing here is inferable from the design; all of it has to be asked.

§ 9 covers leaving for another app and coming back. This section is about the seam inside your own app.

- [ ] Which parts of this frame the webview renders, and which the native shell owns (status bar, header, tab bar / GNB, floating buttons)
- [ ] Whether a device capability drawn here is implemented in the webview or delegated over a bridge — camera, biometrics, push, share sheet, file picker, shake, clipboard, location
- [ ] When delegated, whether the destination is a native screen or a separate full-screen webview route
- [ ] How the result comes back, and what the calling screen must refresh when it does
- [ ] Who owns the back gesture on this screen, and where back goes from a delegated screen
- [ ] Whether this screen can be entered directly by deep link, bypassing the shell that normally sets it up
- [ ] Minimum app version for any bridge this screen needs, and the behavior below it

## 11. Composed screens  —  `traits: composed`

A screen assembled from independently loaded blocks does not have one loading state or one error state, and § 1's vocabulary cannot express that. Ask per block.

- [ ] Whether blocks load independently with their own skeletons, or the screen waits for all of them
- [ ] What a single block's failed request does — hide the block, show an inline retry, replace it with a notice, or block the whole screen
- [ ] Whether different failures are treated differently (a scheduled-maintenance notice is not a network error)
- [ ] Whether a quota or rate limit exists on any block, and what it shows when hit
- [ ] Where the block list and its order come from — fixed in the design, or served
- [ ] When served: the default composition to use if that response is missing, partial, or names a block the client does not know
- [ ] Whether an empty block is hidden or shown empty
- [ ] Which blocks are mandatory, and whether the user can reorder or hide the rest — and where that is stored
