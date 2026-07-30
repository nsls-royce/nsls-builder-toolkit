# 2026 NSLS Fellowship — Faculty Nomination System: HubSpot Build Spec

**Owner (build):** Royce Rowan (RevOps)
**Project direction:** Gary / Ashley · **Operational lead:** Jenna Fontanez · **Copy/QA:** Joanne Sandoval
**Status:** Spec — pending Log-object confirmation + pilot data
**Last updated:** 2026-07-30

---

## 1. What we're building

A HubSpot-native faculty outreach + nomination system for the 2026 Fellowship recruitment cycle:
professors at partner schools are contacted, asked to nominate high-potential students, and each
nomination is captured, logged, and connected back to the nominating professor — so we always know
who referred whom, and the nominated student gets followed up automatically.

**Core principle (decided on the 6/23 RevOps call):** build the whole thing in HubSpot by **cloning
the existing faculty-to-faculty referral process** (the custom **Log** object, `log type = Referral`).
Devin's dossier tool stays a **sourcing/export** tool only — it is **not** a HubSpot enrollment
integration. That descope is what makes an end-of-summer launch realistic.

### Decisions locked
| # | Decision |
|---|---|
| 1 | Nomination records reuse the existing custom **Log** object (`log type` = Referral / new nomination value — TBC). |
| 2 | Outreach **emails send from Joanne** (single sender); **call tasks are divvied** to the contractor/callers via workflow. |
| 3 | Student follow-up on consent = Yes → **both** an automated "apply" email **and** a rep task (backup). |
| 4 | **Call logic stays simple for the pilot** — calls are plain sequence tasks; **no** branching on call outcome. |
| 5 | Devin dossier-tool API enrollment button → **descoped**. |

---

## 2. Reference material

- Project scope / kick-off (Joanne): `2026 NSLS Fellowship — Faculty Nomination Outreach Sequence Project Kick Off`
- Sequence + form copy (Gary): `Professor Nomination Sequence.docx`
- Faculty FAQ (attach to Email 1): `2026 NSLS Fellowship — Faculty FAQ`
- Campaign tracker (Jenna): `Fellowship Recruitment Campaign - Tracker.docx`
- Existing model: `CS: Referral Program` form (HubSpot form id `1b176d2f-86a1-4aae-818d-92000d010b7b`)
- 6/23 call: RevOps Request Review — HubSpot Sequence Build (Fellowship Recruiting)

---

## 3. Data model

### 3.1 Custom Log object (the "nomination record")
Reuse the existing referral Log. **TO CONFIRM (Royce)** — I can't introspect the custom object via
the current tooling, so confirm before cloning:
- Does `log type` already have a **"Faculty Nomination"** option, or do we ride the existing **"Referral"** value?
- Which fields live **on the Log** vs. **on the Contact**?
- Anything on the current referral workflow to change vs. clone as-is?

The Log is the connector: `Professor (contact) ↔ Log ↔ Student (contact)`.

### 3.2 Contact properties — Professor (the submitter / nominator)
| Field | HubSpot property | Notes |
|---|---|---|
| First / last name | `firstname` / `lastname` | existing |
| Email | `email` | existing — dedupe key |
| Phone | `phone` | existing |
| Title / role | `jobtitle` | existing |
| Institution | `school` (+ company association) | existing |
| Department / program | `department` | **new custom field** |

### 3.3 Contact properties — Student (the nominee / "referee")
| Field | HubSpot property | Notes |
|---|---|---|
| First / last name | `referee_first_name` / `referee_last_name` | existing |
| Email | `referee_email` | existing — dedupe key |
| Phone | `referee_phone` | existing |
| School | `referee_school` | existing |
| Major / field of study | `major` | existing |
| How prof knows student | `referee_relationship` | existing |
| Why nominated (context) | `referee_blurb` | existing |
| Academic standing | `academic_standing` | **new custom field** (dropdown) |
| OK for NSLS to contact student? | `contact_consent` | **new custom field** (Yes / No–I'll forward) |

### 3.4 New properties to create (Tier 1)
- `department` (contact, single-line text)
- `academic_standing` (contact, dropdown: First-year / Sophomore / Junior / Senior / Graduate / Unsure)
- `contact_consent` (contact, dropdown: Yes / No – I'll forward)
- `outreach_angle` (contact, multi-line text) — imported per-professor personalization snippet (see §6)
- Confirm/extend the Log `log type` option for nominations.

---

## 4. Faculty nomination form

Clone `CS: Referral Program`, then add student + consent fields. Sections mirror Gary's template
(`Professor Nomination Sequence.docx`):

1. **Faculty info** — name, title, department, institution, email (pre-filled for known contacts).
2. **Student info** — name, email (optional), major, academic standing, consent (Yes / No–I'll forward).
3. **Recommendation context** — relationship, how long known, short blurb.

Host: HubSpot form embedded on a landing page (faculty-facing). **TO CONFIRM** — embed vs. standalone.

---

## 5. Workflow logic (on form submission)

Clone the referral workflow's record-creation + association logic, then:

1. **Create Log record** (`log type` = nomination), stamp submission data.
2. **Create/associate student contact** from `referee_*` fields (a form submit only touches the
   submitter, so the student contact is spun up by the workflow — this already works in the referral flow).
3. **Associate** Professor ↔ Log ↔ Student.
4. **Confirmation email** to the professor (thank-you).
5. **Unenroll professor from the outreach sequence** — *flag:* sequences don't natively unenroll on
   form submit; needs the workflow "unenroll from sequence" action. **TO CONFIRM tier support.**
6. **Student follow-up (consent = Yes)** → **both**:
   - Auto-send student the Fellowship **apply email** — *dependency: Marketing copy + application URL.*
   - Create a **rep task** to reach out (backup).
7. **Consent = No ("I'll forward")** → no student outreach; optional internal note on the Log.

### 5.1 Enrollment & task ownership
- **Emails send from Joanne** (she enrolls, or enrollment workflow sends as her).
- **Call tasks** created by the sequence are **rotated/divvied** to the contractor/callers via a
  workflow owner-rotation action (e.g., contractor takes the bulk, split configurable). Test the split.

---

## 6. Outreach sequence

Linear Sales sequence, marketing-approved copy from `Professor Nomination Sequence.docx`:

1. Email 1 (Faculty Nomination Request) + FAQ attachment
2. Call task (voicemail script) — **plain task, no outcome branching (pilot)**
3. Email 2 (short follow-up)
4. Call task (2nd voicemail) — plain task

- Unenroll on reply / meeting booked (native) + on nomination-form submit (via workflow, §5.5).
- Enroll ~50 at a time from the pilot lists.

---

## 7. Data import & segmentation

1. Run dossier tool → **export to Excel** (Joanne — **pending**).
2. Clean data; rewrite each professor's **"outreach angle"** into a usable `outreach_angle` property
   (via Claude) so it can drop into email personalization.
3. Import ~800 contacts across the **10 pilot schools**; associate to companies.
4. Build **static lists** per school; enroll in the sequence in batches.

---

## 8. Effort tiers (triage)

**Tier 1 — do now, easy:** new properties · `log type` option · clone the form · lists · CSV import ·
confirmation email · student task.

**Tier 2 — moderate (config + test):** Log creation + associations (clone referral) · student-contact
creation from submission (clone referral) · sequence build · call-task divvying · student auto-email
(once contact exists + copy).

**Tier 3 — heavier / decisions:** call-outcome logic (**deferred** — pilot uses plain tasks) ·
auto-unenroll on form submit (confirm tier) · contractor recorded-calling setup (IT/Gary).

**Descoped:** dossier-tool → HubSpot API enrollment button.

---

## 9. Dependencies & gates (owners)

| Gate | Owner | Blocks |
|---|---|---|
| Pilot export (10 schools / ~800 contacts) | Joanne | Import + launch |
| Student apply-email copy + application URL | Marketing | Student auto-email |
| Log object schema + `log type` confirmation | Royce | Cloning the workflow |
| Sequence copy final sign-off | Joanne / Gary | Sequence build |
| Contractor HubSpot seat + recorded calling number | IT / Gary | Contractor making calls |

---

## 10. Timeline

- **Build effort is small** — most of it is cloning the referral assets you already run.
  Tier 1 + Tier 2 core are buildable in days once the Log schema is confirmed and copy is locked.
- **Launch is gated on the pilot data export (Joanne) and Marketing copy**, not the build.
- Target: live before faculty ramp back up in **August**.

---

## 11. Open items to confirm
1. Log object: `log type` value for nominations + Log-vs-Contact field split (Royce).
2. Form host: landing-page embed vs. standalone.
3. Auto-unenroll-on-submit: confirm Sales Hub tier supports the workflow action.
4. Call-task divvy split (what % to contractor vs. Joanne).
5. Current build status — anything already stood up from the June "workflow thing."
