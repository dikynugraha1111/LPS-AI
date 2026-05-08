# M9 — Nomination Status & EPB Confirmation: User Stories

> **Scope note (BRD v3.1):** M9 user stories cover status tracking and the first payment proof submission only. The payment verification cycle (US-EI-*) belongs to M9b.

## US-NP-01: Track Nomination Status
**As a** Customer,
**I want to** see the current status of my submitted nomination,
**So that** I know what action (if any) is required from me.

**Acceptance Criteria:**
- [ ] Nomination detail page shows current status with a clear label
- [ ] Status `PENDING` displays as "Menunggu proses di STS Platform"
- [ ] Status `APPROVED` displays as "Nominasi Disetujui" dengan detail EPB dan tombol "Bayar EPB" yang mengarah ke M9b
- [ ] Status `WAITING_PAYMENT_VERIFICATION` displays as "Menunggu Verifikasi Pembayaran" dengan info banner dan link ke EPB & Invoice
- [ ] Status `NEED_REVISION` displays as "Perlu Revisi" with revision notes from STS
- [ ] Status updates within 1 minute of STS sending the webhook

## US-NP-02: View EPB and Schedule After Approval
**As a** Customer,
**I want to** see the EPB amount and voyage schedule after my nomination is approved,
**So that** I know how much to pay and when my vessel will be serviced.

**Acceptance Criteria:**
- [ ] EPB section is visible when status is `APPROVED` or `WAITING_PAYMENT_VERIFICATION`
- [ ] EPB shows: EPB Number, Total Amount (formatted as IDR), Due Date
- [ ] Schedule shows: Anchor Point, ETB (formatted date-time), Estimated Duration
- [ ] All EPB and schedule data are read-only (sourced from STS Platform)
- [ ] Tombol "Bayar EPB" muncul saat status `APPROVED`, mengarah ke halaman M9b EPB & Invoice

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

## US-NP-04: Navigate to EPB Payment Page
**As a** Customer,
**I want to** melihat EPB saya dan diarahkan ke halaman pembayaran setelah nominasi disetujui,
**So that** saya dapat dengan mudah melakukan pembayaran EPB melalui menu EPB & Invoice.

**Acceptance Criteria:**
- [ ] Saat status `APPROVED`, tombol "Bayar EPB" muncul di bawah detail EPB
- [ ] Klik "Bayar EPB" → navigasi ke halaman M9b EPB & Invoice (`/customer/epb-invoice`)
- [ ] Saat status `WAITING_PAYMENT_VERIFICATION`, tombol "Bayar EPB" diganti dengan info banner: "Menunggu Verifikasi Pembayaran" + link ke EPB & Invoice
- [ ] Customer tidak dapat melakukan upload proof dari halaman M9 — upload dilakukan di M9b
