# Faculty Nomination — HubSpot Build Runbook

**For:** a Claude session operating the **live NSLS HubSpot in an authenticated browser** (account `5345251`).
**Companion doc:** `docs/faculty-nomination-build-spec.md` — read it first for context.
**Goal:** stand up faculty-nomination capture by cloning the existing referral machinery, **without**
contaminating the live chapter-referral flow.

---

## Operating rules (read before touching anything)

1. **Non-destructive by default.** The new workflow stays **OFF** until QA passes. Never enroll real
   faculty during the build.
2. **Test with a dummy record**, never a real professor/student.
3. **Do not invent copy or URLs.** Where something is missing (student apply-email copy, application
   URL), insert a clearly-labeled `PLACEHOLDER` and stop — do not fabricate a live-sending email.
4. **One edit that must happen first** (Phase B) before *any* nomination Log exists, or nominations
   will auto-run through the chapter-referral flow.
5. Check off each `[ ]`. Where a step says **VERIFY**, confirm before continuing.
6. If reality differs from this runbook (extra steps, different field names), **document it and pause**
   rather than guessing — update this file with what you actually found.

Key IDs:
- Log object type: `2-1484012` · property group: `cs:_chapter_referrals`
- Existing referral workflow: `493077772` ("Referral Program - Log, Create Contact, Email, Task")
- Existing referral form: `1b176d2f-86a1-4aae-818d-92000d010b7b` ("CS: Referral Program")
- Classifier values: `log_type` = **Referral**, `referral_type` = **Student Nomination**

---

## Phase A — Recon (map what actually exists before cloning)

- [ ] **A1. Fully expand workflow `493077772`.** Open it and expand *every* branch below step 10
  (the `referee_blurb_options` fan-out). Write down each action in order: contact-creation step(s),
  email step(s), task step(s), delays, and the exact **association labels** used
  (e.g. "Contact: First created"). This is the ground truth you're replicating.
- [ ] **A2. Find the Log-creation mechanism.** `493077772` enrolls on Log *creation* — something else
  creates the Log when the referral form is submitted. Find it: search Automations for a workflow that
  creates a Log on `CS: Referral Program` submission (or check the form's own actions). Document how the
  Log gets `log_type=Referral`, how the referrer contact is associated, and how referee data lands.
- [ ] **A3. Check for deal / reward side-effects.** Search Automations for any workflow keyed off
  `create_deal_for_referral_applicant`, `referral_rewards_program`, or `log_type=Referral` that creates
  a **deal** or assigns a **reward**. Document them — nominations must NOT trigger these.

**Output of Phase A:** a short note appended to this file listing (a) the full step list of `493077772`,
(b) the Log-creation mechanism, (c) any deal/reward workflows.

---

## Phase B — Protect the live referral flow (DO THIS FIRST)

- [ ] **B1.** In `493077772`, open the **enrollment trigger** and add an AND condition:
  `referral_type` **is not equal to** `Student Nomination`.
- [ ] **B2. VERIFY** the trigger now reads: `Log Type = Referral AND referral_type ≠ Student Nomination`.
  Save. (Leave re-enrollment as it is.)
- [ ] **B3.** If Phase A found deal/reward workflows that trigger on `log_type=Referral`, add the same
  `referral_type ≠ Student Nomination` exclusion to each.

> After Phase B, it is safe for nomination Logs (`referral_type = Student Nomination`) to exist — they
> will no longer be swept into chapter-referral automation.

---

## Phase C — Add the gap properties (Log object `2-1484012`)

Settings → Properties → object **Log** → group `cs:_chapter_referrals`. Create (skip any that already exist):

- [ ] `referee_major` — single-line text
- [ ] `referee_academic_standing` — dropdown: `First-year` / `Sophomore` / `Junior` / `Senior` / `Graduate` / `Unsure`
- [ ] `referee_contact_consent` — dropdown: `Yes` / `No - I'll forward`
- [ ] `referrer_department` — single-line text
- [ ] `referrer_phone` — single-line text (optional)
- [ ] **VERIFY** all appear on a Log record's edit view.

---

## Phase D — Nomination form

- [ ] **D1.** Clone `CS: Referral Program` → name it **"Fellowship Faculty Nomination"**.
- [ ] **D2.** Map fields (faculty = referrer, student = referee):
  - Faculty: `referrer_first_name`, `referrer_last_name`, `referrer_email`, `referrer_job_title`,
    `referrer_company_name` (Institution), `referrer_department`
  - Student: `referee_first_name`, `referee_last_name`, `referee_email`, `referee_phone`,
    `referee_school`, `referee_major`, `referee_academic_standing`, `referee_contact_consent`,
    `referee_relationship`, `referee_blurb`
- [ ] **D3.** Replicate the Log-creation mechanism from **A2** so a submission creates a Log with
  `log_type = Referral` **and** `referral_type = Student Nomination`, associated to the professor.
  This is the critical linkage — get it from A2, don't improvise.
- [ ] **D4.** Host: landing-page embed (faculty-facing). **CONFIRM with Royce** embed vs. standalone
  before publishing.

---

## Phase E — Clone the nomination workflow (leave OFF)

- [ ] **E1.** Clone `493077772` → **"Faculty Nomination - Log, Create Contact, Email, Task"**.
- [ ] **E2. Trigger:** `Log Type = Referral AND referral_type = Student Nomination`.
- [ ] **E3.** Delete the **hardcoded one-off branch** (step 1, the "Stacey Malaret <> Josh Austin"
  → Fix Skipped patch).
- [ ] **E4.** Replace the `Referee Email = Log Name` step — the referee email should come from the
  form's `referee_email` field directly, not parsed from the Log name.
- [ ] **E5.** Keep: referrer enrichment from the associated professor contact (the old steps 3–7),
  the `Pending` status set, and the Log-name templating.
- [ ] **E6.** Keep the referee/student **contact-creation + association** step (from A1). Confirm it
  creates a real Student contact from `referee_*` and associates Student ↔ Log ↔ Professor.
- [ ] **E7.** Replace the `referee_blurb_options` email branch with a **consent branch** on
  `referee_contact_consent`:
  - `Yes` → (a) send the student **apply email** [`PLACEHOLDER` — needs Marketing copy + application
    URL; do not send live until provided], **and** (b) create a **task** for a rep to follow up with the
    student (include student name, student email, nominating professor).
  - `No - I'll forward` → no student outreach; optional internal note on the Log.
- [ ] **E8.** Add: **unenroll the professor from the outreach sequence** when this Log is created
  (via the "Unenroll from sequence" action). **CONFIRM** the Sales Hub tier supports this action; if
  not, flag it as a manual step and note it here.
- [ ] **E9.** Add a **confirmation email to the professor** (thank-you) on submission.
- [ ] **E10.** Leave the workflow **OFF**.

---

## Phase F — QA (before turning anything on)

- [ ] **F1.** Manually create a **test Log**: `log_type = Referral`, `referral_type = Student Nomination`,
  associate a **test professor** contact, fill the `referee_*` fields with test data.
- [ ] **F2.** Run the new workflow against the test record (use the workflow's test/preview if available).
- [ ] **F3. VERIFY isolation:** the test Log enrolls in the **new** workflow and does **NOT** appear in
  `493077772`'s enrollment history.
- [ ] **F4. VERIFY behavior:** referrer fields populated · Student contact created + associated · status
  `Pending` · consent branch routes correctly (`Yes` vs `No - I'll forward`) · rep task created with the
  right details · professor confirmation email queued · sequence unenroll fired (or flagged manual).
- [ ] **F5.** Delete the test records (Log, test contacts, any test tasks/emails).

---

## Phase G — Go live (only after QA + copy)

- [ ] **G1.** Confirm Marketing's student apply-email copy + application URL are in (replaces the E7
  `PLACEHOLDER`).
- [ ] **G2.** Turn the nomination workflow **ON**.
- [ ] **G3.** Notify Royce/Joanne it's live; import of Joanne's pilot list + sequence enrollment can proceed.

---

## Must-confirm / do-not-invent list
- Student apply-email copy + application URL (Marketing) — placeholder until provided.
- Sales Hub tier support for "unenroll from sequence" (E8).
- Exact Log-creation + contact-creation mechanism (A1/A2) — replicate, don't improvise.
- Form host: embed vs. standalone (D4).
- Whether any deal/reward workflow needs the exclusion filter (A3/B3).
