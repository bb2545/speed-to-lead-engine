# Speed-to-Lead Engine — Debug Notes

Internal reference log of real issues encountered while building and testing
the Speed-to-Lead Engine (Portfolio 2) in Make.com. Kept for case study
writing and future troubleshooting — not intended for client-facing use as-is.

**Scenario:** `Speed-to-Lead Engine`
**Flow:** HubSpot Watch Deals → List Pipelines → List Associations → Get a
Contact → Router → (Slack / Google Sheets / Gmail), each with error handlers.

---

## Issue 1 — Missing Deal ↔ Contact Association

**Symptom:** `Get a Contact` module failed with `Missing value of required
parameter 'contactId'`. `Code: BundleValidationError`.

**Root cause:** The test deal used for the run had zero associated contacts
in HubSpot. `List Associations` correctly returned an empty result set, and
the downstream module had nothing to work with.

**Fix:** Manually associated a test contact with the deal in HubSpot, then
re-ran the scenario.

**Lesson for production:** A deal reaching this automation with no linked
contact is a realistic edge case (e.g. a rep creates a deal before adding
contact details). The current build assumes a contact always exists — this
is a known gap, documented for the v2 roadmap (add a branch/filter to handle
contact-less deals gracefully instead of erroring).

---

## Issue 2 — Leftover `map()` Formula Broke Google Sheets Branch

**Symptom:** After inserting two new modules (`List Associations`, `Get a
Contact`) into the existing flow, the Google Sheets `Add a Row` module
started failing with `Function 'map' finished with error! Invalid array`.

**Root cause:** A `map()` formula used earlier (Sprint 4c) to translate the
raw HubSpot `dealstage` ID into a human-readable label had been left in the
Sheets "Stage" field mapping. That attempt had been intentionally abandoned
in favor of the raw ID, but the formula itself was never removed — only
the visible behavior was reverted. Changing the upstream bundle structure
(by adding new modules) broke the array shape the old formula expected.

**Fix:** Removed the leftover `map()` formula and remapped the "Stage"
column directly to the raw `dealstage` property from `Watch Deals`.

**Lesson:** "Reverting a decision" and "removing the artifact of that
decision" are two different actions. A partially-reverted formula can sit
dormant and break later when the scenario structure changes elsewhere —
worth a full audit pass of field mappings after any structural edit.

---

## Issue 3 — Cross-Language Character Contamination in Email Field ⭐

**Symptom:** Gmail `Send an email` module failed with `Invalid email address
in parameter 'to'`, even after:
- Swapping to a completely different email address (ruled out a bad value)
- Applying `trim()` to the mapped field (ruled out standard whitespace)

**Root cause:** A Thai vowel sign ("ิ", a combining character that must
attach to a preceding consonant) had been entered at the very start of the
email field in HubSpot — almost certainly from an input-method language
switch that happened mid-keystroke. As a floating combining mark with no
consonant to attach to, most UIs render it as a dotted-circle placeholder,
which made it effectively invisible in the HubSpot UI until inspected
closely. `trim()` does not strip this class of character since it isn't
standard whitespace.

**Diagnosis method:** Rather than guessing further, the value was
temporarily surfaced in the email **Subject line** (`[chip]`) so it could be
visually inspected in the received email — confirming the dotted-circle
artifact was present in the raw data, not introduced by Make.

**Fix attempt that did NOT resolve it:** A whitelist regex —
`replace(field; "/[^a-zA-Z0-9@._%+-]/g"; "")` — was built to strip any
character outside the allowed email character set in Make. This is a solid
defensive pattern in general, but it did not fix this specific case: a
first attempt to simply retype the field directly in HubSpot (click into
the field, select-all, delete, type new value) *also* failed to resolve
it — the contamination reappeared even after retyping, most likely because
a normal click-and-drag or in-place selection didn't reliably capture/clear
a leading zero-width combining character sitting before the visible text.

**Fix that actually worked — "clean room" retype via Notepad:**
1. Opened a plain-text editor (Notepad) with the keyboard confirmed to be
   in EN input mode (not TH) — a neutral space guaranteed free of any
   lingering IME state.
2. Typed the email address fresh, character by character, directly in
   Notepad, visually confirming no stray marks were visible.
3. Selected the full line using `Home` → `Shift+End` (not click-drag) to
   guarantee the selection started exactly at the true beginning of the
   line, then copied it.
4. In HubSpot, clicked into the email field and pressed `Home` first
   (forcing the cursor to the actual start of the field, ahead of any
   invisible character) → `Shift+End` to select everything through to the
   end → `Delete` → pasted the clean value from Notepad → saved.
5. Re-ran the Make scenario — email sent successfully with a clean subject
   line, confirming the contamination was gone.

**Why the first in-HubSpot retype attempt failed but this worked:** mouse-
based click-and-drag selection in the HubSpot field can start just *after*
a leading invisible character rather than before it, leaving it behind even
after "select all and delete." Explicitly using `Home` to jump to the true
start of the field — before making any selection — was the step that
actually guaranteed full removal. Composing the clean text in an external
plain-text editor first also removed any risk of the OS input method
reintroducing a stray character mid-retype.

**Lesson for client-facing framing:** This is a real, recurring risk for any
CRM populated by multilingual teams or copy-pasted from mixed-language
sources — not a one-off typo. Worth naming explicitly in the case study as
"defensive handling of multi-language data entry environments."

---

## Issue 4 — Regex Syntax: Missing Delimiter Slashes

**Symptom:** The whitelist `replace()` formula from Issue 3 ran without
error but had no visible effect — the contaminating character was still
present in the output.

**Root cause:** Make's `replace()` function requires the regex pattern to
be wrapped in `/pattern/flags` delimiters (e.g. `/[^a-z]/g`). The first
attempt used a bare pattern in quotes (`"[^a-zA-Z0-9@._%+-]"`) with no
slashes — Make interpreted this as a literal string to search for, not a
regex, so it silently matched nothing and passed the original text through
unchanged.

**Fix:** Rewrote the pattern with delimiters: `"/[^a-zA-Z0-9@._%+-]/g"`.

**Secondary snag:** Typing `/` directly into a Make formula field triggers
an autocomplete/search popup (Make uses `/` as a field-picker shortcut),
which corrupted manual attempts to type the pattern. Worked around by
copying the `/` character from elsewhere and pasting it in rather than
typing it live.

---

## Issue 5 — Free-Tier Polling Interval Floor (Not a Bug — a Constraint)

**Observation:** Attempted to set the `Watch Deals` polling interval to 5
minutes to match the "5-minute response time" benchmark cited in the
Business Impact section. Make enforced a minimum interval of **15 minutes**
on the account tier in use.

**Resolution:** Left at 15 minutes for this build. Documented as an
environment constraint, not a design flaw — production deployments on paid
HubSpot/Make tiers, or using a true webhook trigger instead of polling,
would close this gap to near-instant.

**Client-facing framing:** Be upfront about this rather than hiding it —
it demonstrates awareness of the difference between a demo environment and
a production deployment.

---

## Issue 6 — Orphaned Router Branch (Slack Fallback) — Deferred

**Context:** Attempted to add a Slack notification as a fallback for the
Gmail branch (triggered only if the Gmail retry sequence exhausts all 3
attempts) — i.e. `Gmail → Retry → Slack (fallback)`.

**Symptom:** The new Slack module repeatedly appeared disconnected from the
scenario flow ("This module is not connected to the flow" warning),
regardless of drag-to-connect attempts.

**Resolution:** Deprioritized as an MVP-plus feature. The primary error
handling (Retry, 3 attempts / 2-minute interval) on the Gmail branch is
functional and unaffected. The fallback notification is deferred to v2
rather than spending further time on a UI connection issue with a lower-
priority enhancement — consistent with the "done > perfect" principle
applied throughout this project.

---

## Summary Table

| # | Issue | Type | Status |
|---|---|---|---|
| 1 | Deal with no associated contact | Data edge case | Documented, v2 roadmap |
| 2 | Leftover `map()` formula broke Sheets | Incomplete revert | Fixed |
| 3 | Thai combining character in email field | Data contamination | Fixed at source |
| 4 | Regex missing `/pattern/flags` delimiters | Syntax error | Fixed |
| 5 | 15-minute polling floor on free tier | Platform constraint | Documented |
| 6 | Orphaned Slack fallback branch | UI/connection issue | Deferred to v2 |
