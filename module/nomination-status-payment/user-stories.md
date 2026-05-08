# M9 — Nomination Status & EPB Confirmation: User Stories

> **Scope note (BRD v3.1):** M9 user stories cover status tracking and the first payment proof submission only. The payment verification cycle (US-EI-*) belongs to M9b.

## US-NP-01: Track Nomination Status
**As a** Customer,
**I want to** see the current status of my submitted nomination,
**So that** I know what action (if any) is required from me.

**Acceptance Criteria:**
- [ ] Nomination detail page shows current status with a clear label
- [ ] Status `PENDING` displays as "Menunggu proses di STS Platform"
- [ ] Status `APPROVED` displays as "Nominasi Disetujui" with schedule and EPB details visible
- [ ] Status `NEED_REVISION` displays as "Perlu Revisi" with revision notes from STS
- [ ] Status updates within 1 minute of STS sending the webhook

## US-NP-02: View EPB and Schedule After Approval
**As a** Customer,
**I want to** see the EPB amount and voyage schedule after my nomination is approved,
**So that** I know how much to pay and when my vessel will be serviced.

**Acceptance Criteria:**
- [ ] EPB section is only visible when status is `APPROVED`
- [ ] EPB shows: EPB Number, Total Amount (formatted as IDR), Due Date
- [ ] Schedule shows: Anchor Point, ETB (formatted date-time), Estimated Duration
- [ ] All EPB and schedule data are read-only (sourced from STS Platform)

## US-NP-03: Revise Nomination on Need Revision
**As a** Customer,
**I want to** revise my nomination when requested by STS,
**So that** my nomination can be re-evaluated and approved.

**Acceptance Criteria:**
- [ ] When status is `NEED_REVISION`, a banner shows the revision notes from STS
- [ ] Customer can click "Revisi Nominasi" to open the edit form
- [ ] Edit form pre-fills with existing nomination data
- [ ] Customer can update any form field and re-upload documents
- [ ] On re-submit, status returns to `PENDING` and a new submission is sent to STS
- [ ] Re-submitted nomination cannot be edited again until next STS response

## US-NP-04: Submit EPB Confirmation (First Payment Proof)
**As a** Customer,
**I want to** upload my payment proof and submit EPB Confirmation after receiving the EPB,
**So that** my payment data is recorded and forwarded for verification in the EPB & Invoice menu.

**Acceptance Criteria:**
- [ ] EPB Confirmation upload section is only visible when status is `APPROVED`
- [ ] Upload accepts PDF, JPG, PNG; rejects other formats with a clear error message
- [ ] File larger than 5MB shows error: "Ukuran file maksimal 5MB"
- [ ] After successful Submit, customer is redirected to the EPB & Invoice page (M9b)
- [ ] EPB Confirmation Submit button is disabled after first submission to prevent duplicate
- [ ] Uploaded file name and upload timestamp are displayed before Submit
