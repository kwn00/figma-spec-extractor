# Gap checklist

Read this at step 4 only. Check the extracted screens against the items below, and send anything Figma never defined to `Needs Answer`.

Do not mechanically list every item. **Pick only what actually applies to the screen in front of you.** Attaching a pagination question to a static information page is noise.

How to prioritize:
- 🔴 **Blocker** — no answer, no code (branch conditions, where required data comes from)
- 🟡 **Edge case** — you will hit it during implementation (errors, empty states)
- 🟢 **Worth confirming** — fine to fix later (long text, accessibility)

---

## 1. Screen states

Applies to nearly every screen. The most common omission by far.

- [ ] Loading — skeleton or spinner, whole screen or partial
- [ ] Empty — when the list has zero items
- [ ] Error — server failure, network loss
- [ ] Unauthorized — not logged in, not the account owner, insufficient tier
- [ ] Under maintenance / outside service hours
- [ ] First visit vs. return visit

## 2. Lists and tables

- [ ] Pagination style (load more / infinite scroll / page numbers)
- [ ] How many per fetch
- [ ] Sort options and the default
- [ ] Behavior when filters combine, and when a filter returns zero results
- [ ] Upper bound on list length
- [ ] Pull to refresh

## 3. Forms and input

- [ ] Required vs. optional
- [ ] Validation rules (length, format, range)
- [ ] Error message wording and where it appears
- [ ] When validation fires (as you type / on blur / on submit)
- [ ] Button state while submitting, and double-submit prevention
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

## 6. Permissions and auth

- [ ] Access while logged out
- [ ] Session expiry
- [ ] Whether identity verification is required, and at what point
- [ ] Screen differences by role

## 7. Business rules

The area most often missing wholesale from a spec. You can never learn it from the screens alone.

- [ ] Conditions that block this action (unpaid balance, already applied, outside the eligible window)
- [ ] When blocked, is the button hidden or disabled, and is a reason shown
- [ ] Whether processing takes time (immediate / next day / under review)
- [ ] Whether it can be cancelled or withdrawn, and by when
- [ ] How duplicate submissions are handled
- [ ] How amounts are calculated, and the rounding rule

## 8. Platform

- [ ] Responsive breakpoints (only mobile screens exist — what about desktop?)
- [ ] iOS / Android differences
- [ ] Dark mode
- [ ] Accessibility — screen reader labels, anything conveyed by color alone

## 9. External integrations

- [ ] Returning from an external app
- [ ] Payment or auth abandoned midway
- [ ] Timeout thresholds and what shows when one is hit
- [ ] Whether a retry is possible
