# M9 — Nomination Status & Payment: User Stories

## US-NP-01: Track Nomination Status
**As a** Customer,  
**I want to** see the current status of my submitted nomination,  
**So that** I know what action (if any) is required from me.

**Acceptance Criteria:**
- [ ] Nomination detail page shows current status with a clear label
- [ ] Status `PENDING` displays as "Menunggu proses di STS Platform"
- [ ] Status `APPROVED` displays as "Nominasi Disetujui" with schedule and EPB details
- [ ] Status `NEED_REVISION` displays as "Perlu Revisi" with revision notes from STS
- [ ] Status `WAITING_PAYMENT_VERIFICATION` displays as "Menunggu Verifikasi Pembayaran"
- [ ] Status `PAYMENT_CONFIRMED` displays as "Pembayaran Dikonfirmasi — Voyage dapat dimulai"
- [ ] Status `PAYMENT_REJECTED` displays as "Pembayaran Ditolak" with rejection reason
- [ ] Status updates within 1 minute of STS sending the webhook

## US-NP-02: View EPB and Schedule After Approval
**As a** Customer,  
**I want to** see the EPB amount and voyage schedule after my nomination is approved,  
**So that** I know how much to pay and when my vessel will be serviced.

**Acceptance Criteria:**
- [ ] EPB section is only visible when status is APPROVED or later
- [ ] EPB shows: EPB Number, Total Amount (formatted as IDR), Due Date
- [ ] Schedule shows: Anchor Point, ETB (formatted date-time), Estimated Duration
- [ ] EPB and schedule data are read-only (view-only, sourced from STS Platform)

## US-NP-03: Revise Nomination on Need Revision
**As a** Customer,  
**I want to** revise my nomination when requested by STS,  
**So that** my nomination can be re-evaluated and approved.

**Acceptance Criteria:**
- [ ] When status is NEED_REVISION, a banner shows the revision notes from STS
- [ ] Customer can click "Revisi Nominasi" to open the edit form
- [ ] Edit form pre-fills with existing nomination data
- [ ] Customer can update any form field and re-upload documents
- [ ] On re-submit, status returns to PENDING and a new submission is sent to STS
- [ ] Re-submitted nomination cannot be edited again until next STS response

## US-NP-04: Upload Proof of Payment
**As a** Customer,  
**I want to** upload my payment proof after receiving the EPB,  
**So that** STS Platform can verify my payment and authorize voyage start.

**Acceptance Criteria:**
- [ ] Payment proof upload section is visible only when status is APPROVED
- [ ] Upload accepts PDF, JPG, PNG; rejects other formats with clear error
- [ ] File larger than 5MB shows error: "Ukuran file maksimal 5MB"
- [ ] After successful upload, status immediately changes to WAITING_PAYMENT_VERIFICATION
- [ ] Customer can re-upload a new proof if previous was rejected
- [ ] Uploaded file name is displayed with upload timestamp

## US-NP-05: Receive Payment Verification Result
**As a** Customer,  
**I want to** be notified of the payment verification result from STS,  
**So that** I know if my payment was accepted or if I need to upload a corrected proof.

**Acceptance Criteria:**
- [ ] On PAYMENT_CONFIRMED: status banner changes to "Pembayaran Dikonfirmasi — Voyage dapat dimulai"; customer receives in-app notification
- [ ] On PAYMENT_REJECTED: status shows "Pembayaran Ditolak" with the rejection reason from STS; customer receives in-app notification with reason
- [ ] On PAYMENT_REJECTED: "Upload Ulang Bukti Bayar" button is visible
- [ ] Customer receives email notification for both PAYMENT_CONFIRMED and PAYMENT_REJECTED
