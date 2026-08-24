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

Work through every section a screen selects. A section nothing selects is skipped. If a screen's record has no `traits` key at all, that is a Step 3 omission, not a screen with no traits — infer the traits from its `data_fields` and `actions` before continuing, and note it in Extraction notes.

Within a selected section, still skip individual items that plainly cannot apply. The section-level choice is the lookup; the item-level one is yours.

## How to prioritize

- 🔴 **Blocker** — no answer, no code (branch conditions, where required data comes from)
- 🟡 **Edge case** — you will hit it during implementation (errors, empty states)
- 🟢 **Worth confirming** — fine to fix later (long text, accessibility)

Aim for roughly seven 🔴 per output file — but never demote a real blocker just to hit that. And **one question, one entry** — when the same question falls out of five screens, list the screens on a single line rather than repeating it five times. See Step 4.

---

## 1. Screen states  —  diff of `states_defined`

Applies to nearly every screen. The most common omission by far.

Diff the fixed vocabulary against `states_defined`, and **trust that field over your own reading of the screen.** A state drawn as a component variant is defined even though no separate frame exists for it; Step 3 is responsible for having caught that. Reporting a state as missing when the designer drew it is worse than missing one — it costs the reader's trust in every other item here.

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

## 6. Permissions and auth  —  `traits: permission`

- [ ] Access while logged out
- [ ] Session expiry
- [ ] Whether identity verification is required, and at what point
- [ ] Screen differences by role

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
