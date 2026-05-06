# M8 — Nomination Request Submission: User Stories

## US-NS-01: Fill and Save Nomination as Draft
**As a** Customer,  
**I want to** fill in a nomination form and save it as Draft,  
**So that** I can complete and submit it later without losing my progress.

**Acceptance Criteria:**
- [ ] Nomination form shows fields: Vessel Name, ETA, Cargo Type, Cargo Quantity (MT), Charterer, Estimasi Jumlah Barge
- [ ] Form shows document upload areas for: Rencana Kerja, Shipping Instruction, Surat Penunjukan PBM, and text input for Nomor PKK
- [ ] Customer can click "Simpan Draft" with any subset of fields filled (no required field validation on Draft save)
- [ ] Draft is saved with status DRAFT and appears in Nomination Status list
- [ ] Customer can re-open and edit draft from the Nomination Status page
- [ ] ETA field rejects past dates

## US-NS-02: Upload Supporting Documents
**As a** Customer,  
**I want to** upload supporting documents to my nomination,  
**So that** the STS Platform has all required information to process my nomination.

**Acceptance Criteria:**
- [ ] Document upload accepts PDF, JPG, PNG formats only
- [ ] Maximum file size per document is 10MB; larger files show error: "Ukuran file maksimal 10MB"
- [ ] Invalid file type shows error: "Format file tidak didukung. Gunakan PDF, JPG, atau PNG."
- [ ] Uploaded document shows file name and upload status (success/failed)
- [ ] Customer can replace an uploaded document by uploading again

## US-NS-03: Submit Nomination to STS Platform
**As a** Customer,  
**I want to** submit my completed nomination,  
**So that** it is sent to STS Platform for approval and scheduling.

**Acceptance Criteria:**
- [ ] "Submit" button triggers full form validation; missing required fields show inline errors
- [ ] All four documents (Rencana Kerja, Shipping Instruction, Surat Penunjukan PBM) must be uploaded and Nomor PKK must be filled before submit succeeds
- [ ] On successful submit: system generates Nomination Number (format: NOM-YYYYMMDD-NNNN)
- [ ] On successful submit: page redirects to Nomination Status page showing the new nomination with status "Submitted"
- [ ] On STS API failure after retries: customer sees error: "Gagal mengirim nominasi. Silakan coba lagi." with a retry button
- [ ] Submitted nomination can no longer be edited
