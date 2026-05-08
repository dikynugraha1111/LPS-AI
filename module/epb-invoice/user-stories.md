# M9b — EPB & Invoice (Customer Portal): User Stories

## US-EI-01: View EPB & Invoice List
**As a** Customer,
**I want to** see a list of all my EPBs with their current payment status,
**So that** I can quickly identify which EPBs need action and which are settled.

**Acceptance Criteria:**
- [ ] EPB & Invoice menu shows a list of all EPBs belonging to the logged-in customer
- [ ] Each list item displays: EPB Number, Amount (formatted IDR), Due Date, and current status badge
- [ ] Status badges are color-coded: Unpaid (red), Waiting Payment Verification (yellow), Payment Reject (orange), Paid (green)
- [ ] List is sorted by most recently updated first
- [ ] Clicking a list item opens the EPB detail page
- [ ] EPB dengan status `UNPAID` menampilkan tombol "Pay" langsung di card list

## US-EI-02: View EPB Detail
**As a** Customer,
**I want to** see the full detail of a specific EPB,
**So that** I can review the invoice information and my payment history for that EPB.

**Acceptance Criteria:**
- [ ] Detail page shows: EPB Number, Amount, Currency, Due Date, related Nomination Number
- [ ] Detail page shows the current status prominently
- [ ] Detail page shows the history of all proof uploads with file name and upload timestamp
- [ ] If status is Payment Reject, the rejection reason from STS is displayed clearly
- [ ] All data is view-only (no editing of EPB data)

## US-EI-03: Pay an Unpaid EPB
**As a** Customer,
**I want to** upload my payment proof for an Unpaid EPB and submit it for verification,
**So that** STS Platform can confirm my payment and authorize my voyage.

**Acceptance Criteria:**
- [ ] "Pay" button is only visible when status is `UNPAID`
- [ ] Clicking "Pay" opens a payment proof upload form
- [ ] Upload accepts PDF, JPG, PNG; rejects other formats with a clear error message
- [ ] File larger than 5MB shows error: "Ukuran file maksimal 5MB"
- [ ] Optional fields available: Bank Name, Reference Number, Payment Date
- [ ] After clicking Submit: `epb_payments.status` dan `nominations.status` sama-sama berubah ke `WAITING_PAYMENT_VERIFICATION`
- [ ] "Pay" button is hidden after submission; status badge berubah ke "Menunggu Verifikasi"
- [ ] Customer receives in-app and email notification confirming submission

## US-EI-04: Monitor Waiting Payment Verification Status
**As a** Customer,
**I want to** see that my payment is being reviewed after I submit proof,
**So that** I know my submission was received and I am waiting for a decision.

**Acceptance Criteria:**
- [ ] Status badge shows "Menunggu Verifikasi Pembayaran" with yellow color
- [ ] No action buttons are visible while status is `WAITING_PAYMENT_VERIFICATION`
- [ ] The latest uploaded proof (file name + timestamp) is visible
- [ ] Page/list auto-refreshes or polls to reflect STS decision within 1 minute

## US-EI-05: Re-upload After Payment Rejection
**As a** Customer,
**I want to** upload a corrected payment proof after my previous one was rejected,
**So that** my payment can be re-verified and approved.

**Acceptance Criteria:**
- [ ] Status badge shows "Pembayaran Ditolak" with orange color
- [ ] Rejection reason from STS is displayed clearly below the status
- [ ] "Revision Data" button is only visible when status is `PAYMENT_REJECT`
- [ ] Clicking "Revision Data" opens the proof upload form
- [ ] Upload accepts PDF, JPG, PNG; rejects other formats with a clear error message
- [ ] File larger than 5MB shows error: "Ukuran file maksimal 5MB"
- [ ] After Submit: `epb_payments.status` dan `nominations.status` sama-sama berubah ke `WAITING_PAYMENT_VERIFICATION`
- [ ] Customer receives in-app and email notification confirming re-submission
- [ ] Proof upload history shows both rejected and new uploads

## US-EI-06: View Paid EPB
**As a** Customer,
**I want to** see that my EPB is fully paid and confirmed,
**So that** I have a record that my payment was accepted and my voyage is authorized.

**Acceptance Criteria:**
- [ ] Status badge shows "Lunas" with green color
- [ ] Confirmation timestamp (confirmed_at) is displayed
- [ ] No action buttons are present
- [ ] All proof upload history is visible for reference
- [ ] Page is fully view-only
