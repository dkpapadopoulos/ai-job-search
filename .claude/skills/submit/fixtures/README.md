# /submit fixtures — field-detection + mapping dry-run oracle

This directory de-risks `/submit` (Task 4) **before** it ever touches a live
employer ATS form. It contains two saved form fixtures and a hand-verified
"oracle" table of the expected field → answer-bank-intent → action mapping.
Task 4's skill logic must reproduce this table exactly against these fixtures
before it is trusted against a real Greenhouse/Lever posting.

## Provenance

`greenhouse-sample.html` and `lever-sample.html` are **hand-authored,
representative fixtures**, not scraped copies of a live posting. This
execution had no bound live-browser tool (`claude-in-chrome` requires a real
Chrome session, which was not available here), and `WebFetch` only returns an
AI-summarized markdown rendering of a page rather than raw form markup — not
usable for exact `<input name>`/`<select>` structure. Per the task brief's
explicit fallback ("If live fetch isn't possible, hand-author a representative
form..."), both fixtures were built to match the well-known public DOM
conventions of each ATS:

- **Greenhouse:** `#application_form`, `job_application[first_name]` /
  `job_application[email]` style field names, `job_application[answers_attributes][n][text_value]`
  for custom screening questions, `job_application[eeo][gender]` for
  demographics.
- **Lever:** `form.application-form`, `.application-question` /
  `.application-label` blocks, plain `name="email"` / `name="phone"` /
  `name="resume"` fields, `cards[custom-*][field0]` for custom questions,
  `eeo[veteran_status]` for demographics.

Both fixtures include, as required by the brief: a text input for name/email,
a file upload for resume + cover letter, a select dropdown for work
authorization (Yes/No), a textarea "why you" essay question, and one EEO
self-identification select. The salary field is deliberately implemented
differently in each fixture to exercise both branches of the compensation
mapping rule:
- **Greenhouse** salary field = `<input type="number">` (strict numeric ATS field).
- **Lever** salary field = `<input type="text">` (free-text ATS field).

**Live `claude-in-chrome` validation against a real employer ATS is
DEFERRED** to the first real `/submit` run with the user present. This
fixture set only proves the mapping *logic*, not live-DOM interaction
(dynamic React/Vue rendering, iframes, shadow DOM, etc. on the real sites).

## Mapping rules (carried into Task 4's skill text)

1. Text/email/tel input labeled name / email / phone / location → **Identity & Contact** intent → `fill` the literal value from the answer bank.
2. `type="file"` input whose label/name contains "resume" or "cv" → `upload:cv.pdf`. One containing "cover" or "additional" (document) → `upload:cover.pdf`.
3. Select/dropdown whose label asks about legal work authorization → **Work Authorization** intent → `fill:"Yes"`.
4. Salary/compensation field:
   - If the input is a **strict number field** (`type="number"`, or otherwise constrained to digits only) → **Compensation (number)** sub-case → `blank+flag` (never guess a number).
   - If the input is **free-text** (`type="text"`/textarea) → **Compensation (free-text)** sub-case → `fill` the negotiable phrase from the answer bank.
5. Textarea asking an open-ended "why this company / why you" essay question → **Free-text essays** intent → `blank+flag` (flag as "seed from cover.pdf", never auto-fill unseen).
6. Select/fieldset asking gender, race/ethnicity, veteran, or disability self-identification → **Voluntary/EEO** intent → `decline-self-id` (select the "decline to answer" option where offered).

## Expected-mapping oracle

| Fixture | Field label | Answer-bank intent | Expected action |
|---|---|---|---|
| greenhouse | First Name | Identity & Contact → name | fill:"Alexis" |
| greenhouse | Last Name | Identity & Contact → name | fill:"Example" |
| greenhouse | Email | Identity & Contact → email | fill:"alexis.example@example.com" |
| greenhouse | Phone | Identity & Contact → phone | fill:"+41 79 000 00 00" |
| greenhouse | Resume/CV | Document upload | upload:cv.pdf |
| greenhouse | Cover Letter | Document upload | upload:cover.pdf |
| greenhouse | Are you legally authorized to work in Switzerland? | Work Authorization | fill:"Yes" |
| greenhouse | What is your desired annual salary (CHF)? | Compensation (strict number field) | blank+flag |
| greenhouse | Why do you want to work with us? | Free-text essays | blank+flag (seed from cover.pdf) |
| greenhouse | Gender | Voluntary/EEO | decline-self-id |
| lever | Full Name | Identity & Contact → name | fill:"Alexis Example" |
| lever | Email | Identity & Contact → email | fill:"alexis.example@example.com" |
| lever | Phone | Identity & Contact → phone | fill:"+41 79 000 00 00" |
| lever | Current location | Identity & Contact → location | fill:"Zurich, Switzerland" |
| lever | Resume/CV | Document upload | upload:cv.pdf |
| lever | Additional Document (Cover Letter) | Document upload | upload:cover.pdf |
| lever | Are you authorized to work in the country in which this job is based? | Work Authorization | fill:"Yes" |
| lever | What are your salary expectations? | Compensation (free-text field) | fill:"Open / negotiable, aligned to market for the role." |
| lever | Why are you interested in this role? | Free-text essays | blank+flag (seed from cover.pdf) |
| lever | Veteran Status | Voluntary/EEO | decline-self-id |

## Dry-run result

**Method:** No live browser/`claude-in-chrome` session was available in this
execution, so this is a **paper dry-run**: each fixture's HTML was read
field-by-field, the mapping rules above were applied by hand (as Task 4's
skill logic is specified to do), and the resulting field→action list was
compared line-for-line against the oracle table above.

### Actual field → action list (paper dry-run)

**greenhouse-sample.html**
| Field | Detected input type | Action taken |
|---|---|---|
| First Name | text | fill:"Alexis" |
| Last Name | text | fill:"Example" |
| Email | email | fill:"alexis.example@example.com" |
| Phone | tel | fill:"+41 79 000 00 00" |
| Resume/CV | file | upload:cv.pdf |
| Cover Letter | file | upload:cover.pdf |
| Are you legally authorized to work in Switzerland? | select | fill:"Yes" |
| What is your desired annual salary (CHF)? | number | blank+flag |
| Why do you want to work with us? | textarea | blank+flag (seed from cover.pdf) |
| Gender | select (EEO) | decline-self-id |

**lever-sample.html**
| Field | Detected input type | Action taken |
|---|---|---|
| Full Name | text | fill:"Alexis Example" |
| Email | email | fill:"alexis.example@example.com" |
| Phone | text | fill:"+41 79 000 00 00" |
| Current location | text | fill:"Zurich, Switzerland" |
| Resume/CV | file | upload:cv.pdf |
| Additional Document (Cover Letter) | file | upload:cover.pdf |
| Are you authorized to work in the country in which this job is based? | select | fill:"Yes" |
| What are your salary expectations? | text | fill:"Open / negotiable, aligned to market for the role." |
| Why are you interested in this role? | textarea | blank+flag (seed from cover.pdf) |
| Veteran Status | select/fieldset (EEO) | decline-self-id |

### Comparison to oracle

| Check | Result |
|---|---|
| greenhouse: 10/10 fields match oracle | PASS |
| lever: 10/10 fields match oracle | PASS |
| Salary-number (greenhouse) → blank+flag | PASS |
| Salary-free-text (lever) → fill negotiable phrase | PASS |
| Both EEO selects → decline-self-id | PASS |
| Both why-you textareas → blank+flag | PASS |
| **Overall match** | **20/20 fields — 100% PASS** |

No discrepancies were found; no fixes to the mapping rules or fixtures were
required.

### What remains deferred

- Live `claude-in-chrome` execution against these (or real) fixture pages —
  deferred to the first real `/submit` run with the user present, per the
  execution constraint noted above.
- Live scrape of a real, currently-open Greenhouse/Lever posting to confirm
  the hand-authored DOM conventions still match production markup (ATS
  vendors do change field markup over time).
