# M9c — Invoice (Customer Portal): User Stories

## US-IN-01: View Invoice List
**As a** Customer,
**I want to** see a list of all my Invoices with their current payment status and source,
**So that** I can identify which Invoices need action and understand whether they came from EPB shortfall or Additional Service.

**Acceptance Criteria:**
- [ ] Tab "Invoice" di menu "EPB & Invoice" menampilkan list seluruh Invoice customer.
- [ ] KPI pills di atas tabs menampilkan count per kategori: **Perlu Tindakan** (rose), **Menunggu Verifikasi** (amber), **Lunas** (emerald). Klik pill → aktifkan tab filter terkait.
- [ ] Tabs filter: **Semua**, **Perlu Tindakan**, **Menunggu Verifikasi**, **Lunas** dengan count per tab.
- [ ] Setiap row table: No. Invoice, Source badge (Shortfall EPB / Additional Service), Ref Nominasi, Amount, Jatuh Tempo, Status badge, Aksi.
- [ ] Source badge "Shortfall EPB" = neutral pill; "Additional Service" = info pill.
- [ ] Status badge label: "Belum Dibayar" / "Menunggu Verifikasi" / "Pembayaran Ditolak" / "Lunas".
- [ ] Untuk Invoice overdue: tanggal jatuh tempo dengan warna rose-700 + icon AlertTriangle.
- [ ] List sorted by most recently updated first.
- [ ] Klik row → buka Invoice detail page.
- [ ] Invoice dengan status `Belum Dibayar` menampilkan tombol primary "Bayar" di kolom Aksi.

## US-IN-02: View Invoice Detail with Source Info
**As a** Customer,
**I want to** see the full detail of a specific Invoice including its origin (EPB shortfall or Additional Service),
**So that** I understand why I'm being charged and can verify the amount.

**Acceptance Criteria:**
- [ ] Detail page menampilkan: Invoice Number, Amount, Currency, Due Date, Ref Nominasi.
- [ ] Source-specific section:
  - **Jika source = `EPB_SHORTFALL`:** tampilkan "Asal: Shortfall EPB {parent_epb_number}" dengan link ke detail EPB di M9b. Tampilkan info "Tagihan ini muncul karena pembayaran EPB sebelumnya tidak penuh."
  - **Jika source = `ADDITIONAL_SERVICE`:** tampilkan "Asal: Additional Service" dengan list service yang ditagih (dengan label ID dari mapping di specifications.md §9). Contoh: "Tank Cleaning, Pengisian Bahan Bakar atau Air Bersih".
- [ ] Status banner besar di atas page sesuai variant status saat ini.
- [ ] Section Riwayat Pembayaran: semua attempt dengan timestamp, file proof, status outcome.
- [ ] All data is view-only kecuali action card kontextual.

## US-IN-03: Pay an Unpaid Invoice
**As a** Customer,
**I want to** upload my payment proof for an Unpaid Invoice and submit it for verification,
**So that** STS Platform can confirm my settlement payment.

**Acceptance Criteria:**
- [ ] Tombol "Bayar" hanya visible saat status = `Belum Dibayar`.
- [ ] Klik "Bayar" membuka modal/section upload form dengan field:
  - File upload (PDF/JPG/PNG, max 5MB)
  - Bank Name (optional)
  - Reference Number (optional)
  - Payment Date (optional)
- [ ] **Tidak ada field nominal pembayaran** — Invoice harus dibayar penuh sesuai `invoices.amount` (display amount sebagai read-only info).
- [ ] Upload format selain PDF/JPG/PNG → error message: "Format file tidak didukung."
- [ ] File > 5MB → error: "Ukuran file maksimal 5 MB."
- [ ] Setelah Submit: `invoices.status` berubah ke `WAITING_PAYMENT_VERIFICATION`.
- [ ] `nominations.status` **TIDAK** berubah (Invoice tidak block nominasi).
- [ ] Tombol "Bayar" hilang setelah submit; status badge berubah ke "Menunggu Verifikasi".
- [ ] Customer menerima in-app notification konfirmasi.

## US-IN-04: Monitor Waiting Verification Status
**As a** Customer,
**I want to** see that my Invoice payment is being reviewed,
**So that** I know my submission was received and I am waiting for a decision.

**Acceptance Criteria:**
- [ ] Status badge: "Menunggu Verifikasi" (amber/teal soft pill).
- [ ] Tidak ada tombol aksi saat status `WAITING_PAYMENT_VERIFICATION`.
- [ ] Latest uploaded proof (file name + timestamp) visible di section Riwayat Pembayaran.
- [ ] Inline info banner: "Bukti pembayaran sedang dalam proses verifikasi oleh tim kami. Harap tunggu konfirmasi."
- [ ] Page/list auto-refresh atau polling untuk reflect STS decision dalam < 1 menit.

## US-IN-05: Re-upload After Invoice Payment Rejection
**As a** Customer,
**I want to** upload a corrected payment proof after my previous Invoice payment was rejected,
**So that** my Invoice can be re-verified and approved.

**Acceptance Criteria:**
- [ ] Status badge: "Pembayaran Ditolak" (rose soft pill).
- [ ] Rejection reason dari STS displayed di status banner dan di history attempt yang ter-reject.
- [ ] Tombol "Revisi Data" hanya visible saat status = `PAYMENT_REJECT`.
- [ ] Klik "Revisi Data" membuka form upload yang sama dengan US-IN-03.
- [ ] Setelah Submit: status berubah ke `WAITING_PAYMENT_VERIFICATION`.
- [ ] Customer menerima notification konfirmasi re-submission.
- [ ] Riwayat Pembayaran menampilkan semua attempt: yang ditolak dan yang baru.

## US-IN-06: View Paid Invoice
**As a** Customer,
**I want to** see that my Invoice is fully paid and confirmed,
**So that** I have a record that my settlement was accepted.

**Acceptance Criteria:**
- [ ] Status badge: "Lunas" (emerald soft pill).
- [ ] Confirmation timestamp (`confirmed_at`) ditampilkan.
- [ ] Tidak ada action button.
- [ ] Riwayat Pembayaran tetap visible untuk reference.
- [ ] Page fully view-only.

## US-IN-07: Automatic Invoice Creation from STS Webhook
**As a** System,
**I want to** automatically create Invoice records when STS Platform sends webhook events,
**So that** customers always see up-to-date settlement obligations without manual intervention.

**Acceptance Criteria:**
- [ ] Webhook endpoint `POST /api/webhooks/sts/invoice` menerima 4 event: `EPB_SHORTFALL_DETECTED`, `ADDITIONAL_SERVICE_INVOICE`, `INVOICE_PAYMENT_REJECT`, `INVOICE_PAID`.
- [ ] HMAC signature verification berhasil sebelum process.
- [ ] Idempotency check: webhook dengan `idempotency_key` yang sudah pernah diproses → no-op + return 200.
- [ ] Create event: insert row baru ke `invoices` dengan status `UNPAID`, populate source-specific fields (`parent_epb_id` untuk shortfall; `service_keys[]` untuk additional service).
- [ ] Customer menerima notification baru: "Tagihan Invoice baru: Rp {amount} ({source-specific message})."
- [ ] Tab Invoice di UI customer menampilkan row baru dalam < 1 menit (via polling/SSE).
