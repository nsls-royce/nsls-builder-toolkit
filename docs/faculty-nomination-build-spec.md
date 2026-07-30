# 2026 NSLS Fellowship — Faculty Nomination System: HubSpot Build Spec

**Owner (build):** Royce Rowan (RevOps)
**Project direction:** Gary / Ashley · **Operational lead:** Jenna Fontanez · **Copy/QA:** Joanne Sandoval
**Status:** Spec — Log schema confirmed; pending pilot data + copy
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
| 1 | Nomination records reuse the existing custom **Log** object: `log_type` = Referral, `referral_type` = **Student Nomination** (both already exist). |
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

**Confirmed from the Log + Chapter property exports (2026-07-30).** The nomination record is the
existing custom **Log** object (type `2-1484012`), which already models referrals — no new object,
no new field namespace needed.

### 3.1 The Log object (the "nomination record")
- **Classification (already exists):** `log_type` = **Referral**; `referral_type` = **Student Nomination**
  (options: NSLS Employee / Student Nomination / CS Referral). Faculty nominations ride this exact value.
- **The Log is the connector:** `Professor (contact) ↔ Log ↔ Student (contact)`, and associates to the
  **Chapter** object (`2-1363395`, `chapter_name` + `hs_object_id`) for the school.
- **Referrer/referee fields are on the Log** (group `cs:_chapter_referrals`). A matching `referee_*` set
  also exists on the Contact (form capture); the workflow copies to the Log record.

### 3.2 Field map — Faculty (the referrer) → Log
| Nomination field | Log property | Status |
|---|---|---|
| Faculty first / last name | `referrer_first_name` / `referrer_last_name` | existing |
| Email | `referrer_email` | existing — dedupe key |
| Title / role | `referrer_job_title` | existing |
| Institution | `referrer_company_name` | existing |
| Department / program | — | **gap → add field or fold into note** |
| Phone | — | **gap** (no `referrer_phone` today) |

### 3.3 Field map — Student (the referee / nominee) → Log
| Nomination field | Log property | Status |
|---|---|---|
| Student first / last name | `referee_first_name` / `referee_last_name` | existing |
| Email | `referee_email` | existing — dedupe key |
| Phone | `referee_phone` | existing |
| School | `referee_school` | existing |
| How prof knows student | `referee_relationship` | existing |
| Why nominated (context) | `referee_blurb` (+ `referee_blurb_options`) | existing |
| Major / field of study | — | **gap → add field or fold into `referee_blurb`** |
| Academic standing | — | **gap → add field or fold into `referee_blurb`** |
| OK for NSLS to contact student? | — | **gap → add consent field** |

### 3.4 Process fields (already on the Log)
- `log_type` = Referral · `referral_type` = Student Nomination
- `referral_approval_status` (Pending → Approved) — nomination review state
- `referral_rewards_program` — N/A for faculty nominations (leave blank)
- `associated_contacts` (rollup) — the professor↔student connector count

### 3.5 New properties to add (small — the only true gaps)
Add to the Log (group `cs:_chapter_referrals`) so nothing is lost on import:
- `referee_major` (text)
- `referee_academic_standing` (dropdown: First-year / Sophomore / Junior / Senior / Graduate / Unsure)
- `referee_contact_consent` (dropdown: Yes / No – I'll forward)
- `referrer_department` (text)  ·  `referrer_phone` (text) — optional
- `outreach_angle` (contact, multi-line) — imported per-professor personalization snippet (see §6)

For the manual pilot, these four can also just live in `referee_blurb` free text until the fields exist.

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

### 5.0 Reference: the existing referral workflow (`493077772`) — mapped 2026-07-30
"Referral Program - Log, Create Contact, Email, Task" — **ON**, object-based, enrolls **Log** records.
- **Trigger:** `Log created < 1 day AND Log Type = Referral` — **does NOT filter on `referral_type`**, re-enrollment ON.
- Steps: hardcoded one-off branch (cruft) → set `referee_email` from Log Name (referral-form hack) →
  enrich `referrer_*` from the associated "first created" contact → template Log Name → set status
  `Pending` → branch on `referee_blurb_options` → delays → email/task. No deal creation, no reward assignment.

> **⚠️ Collision risk (do this first):** because the trigger fires on *any* `Log Type = Referral`,
> a nomination Log created with that type will auto-enroll here and run chapter-referral automation.
> **Before creating any nomination Logs:** add a `referral_type ≠ Student Nomination` exclusion to
> `493077772`, and build the nomination workflow as a separate clone scoped to
> `Log Type = Referral AND referral_type = Student Nomination`.

**Reuses cleanly:** trigger pattern · referrer enrichment from associated contact · `Pending` status ·
Log-name templating · the referee/student contact-creation step.
**Must rebuild:** the `referee_email = Log Name` hack (form populates `referee_email` directly) ·
the `referee_blurb_options` email/task branch (replace with student apply-email + task + consent gate) ·
drop the hardcoded one-off branch.

### 5.1 Nomination workflow (cloned + scoped)

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

### 5.2 Enrollment & task ownership
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

- **Build effort is small** — the data model already exists (`referral_type` = Student Nomination on
  the Log). The work is: add 3–4 gap fields, clone/point the form + workflow at that type, build the
  sequence. Buildable in days once copy is locked.
- **Launch is gated on the pilot data export (Joanne) and Marketing copy**, not the build.
- Target: live before faculty ramp back up in **August**.

---

## 11. Open items to confirm
1. ~~Log object schema~~ — **resolved** (Log `2-1484012`, `referral_type` = Student Nomination, full
   `referrer_*`/`referee_*` set). Confirm whether any nomination Logs already exist from prior use.
2. Add the gap fields (§3.5) or fold into `referee_blurb` for the pilot.
3. Form host: landing-page embed vs. standalone.
4. Auto-unenroll-on-submit: confirm Sales Hub tier supports the workflow action.
5. Call-task divvy split (what % to contractor vs. Joanne).
6. Current build status — anything already stood up from the June "workflow thing."
