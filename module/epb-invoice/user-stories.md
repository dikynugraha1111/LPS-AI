# M9b — EPB (Customer Portal): User Stories

> **v3.3:** Status labels disinkronkan ke Bahasa Indonesia natural: Belum Dibayar / Menunggu Verifikasi / Pembayaran Ditolak / Lunas. Added US-EI-07 untuk partial payment flow.

## US-EI-01: View EPB List
**As a** Customer,
**I want to** see a list of all my EPBs with their current payment status,
**So that** I can quickly identify which EPBs need action and which are settled.

**Acceptance Criteria:**
- [ ] Tab "EPB" di menu "EPB & Invoice" menampilkan list seluruh EPB customer.
- [ ] KPI pills di atas tabs menampilkan count per kategori: **Perlu Tindakan** (rose), **Menunggu Verifikasi** (amber), **Lunas** (emerald). Klik pill → aktifkan tab filter terkait.
- [ ] Tabs filter: **Semua**, **Perlu Tindakan**, **Menunggu Verifikasi**, **Lunas** dengan count per tab.
- [ ] Setiap row table: No. EPB, Ref Nominasi, Total, Jatuh Tempo, Status badge, Aksi.
- [ ] Status badge label sesuai mapping v3.3: "Belum Dibayar" / "Menunggu Verifikasi" / "Pembayaran Ditolak" / "Lunas".
- [ ] Untuk EPB overdue (`due_date < now()` AND `status != Lunas`): tanggal jatuh tempo tampil dengan warna rose-700 + icon `AlertTriangle`, label "Lewat jatuh tempo".
- [ ] List sorted by most recently updated first.
- [ ] Klik row → buka EPB detail page.
- [ ] EPB dengan status `Belum Dibayar` di kolom Aksi menampilkan tombol primary "Bayar" + link "Lihat Detail →".

## US-EI-02: View EPB Detail
**As a** Customer,
**I want to** see the full detail of a specific EPB,
**So that** I can review the invoice information and my payment history for that EPB.

**Acceptance Criteria:**
- [ ] Detail page menampilkan: EPB Number, Total Amount, Paid Amount (jika ada), Currency, Due Date, Ref Nominasi.
- [ ] Status banner besar di atas page sesuai variant status saat ini.
- [ ] Section Riwayat Pembayaran menampilkan semua attempt: timestamp, nominal yang dibayar, file proof, status outcome (jika reject: tampilkan alasan reject).
- [ ] Jika status = `Lunas` dan `paid_amount < total_amount`: tampilkan info banner "Sisa pembayaran Rp {shortfall} telah ditagihkan via Invoice. [Lihat Invoice →]" yang link ke detail Invoice di M9c.
- [ ] All data is view-only kecuali action card kontextual.

## US-EI-03: Pay an Unpaid EPB (with Partial Payment Option)
**As a** Customer,
**I want to** input my payment amount and upload my payment proof for an Unpaid EPB,
**So that** STS Platform can confirm my payment and authorize my voyage. **I should be able to pay partially (min 1 USD equivalent IDR) if I cannot pay the full amount immediately.**

**Acceptance Criteria:**
- [ ] Tombol "Bayar" hanya visible saat status = `Belum Dibayar`.
- [ ] Klik "Bayar" membuka modal/section upload form dengan field:
  - **Nominal Pembayaran** (required, numeric input dengan format IDR, default value = total_amount tapi editable)
  - File upload (PDF/JPG/PNG, max 5MB)
  - Bank Name (optional)
  - Reference Number (optional)
  - Payment Date (optional)
- [ ] Validasi nominal client-side: `≥ min_idr` (dihitung dari kurs USD→IDR yang di-cache) dan `≤ total_amount`. Inline error rose-600 jika invalid.
- [ ] Helper text di bawah field nominal: "Minimum pembayaran: Rp {min_idr} (ekuivalen 1 USD). Anda boleh membayar parsial; sisa akan ditagihkan via Invoice."
- [ ] Upload format selain PDF/JPG/PNG → error message: "Format file tidak didukung. Gunakan PDF, JPG, atau PNG."
- [ ] File > 5MB → error: "Ukuran file maksimal 5 MB."
- [ ] Setelah Submit: `epb_payments.status` dan `nominations.status` keduanya berubah ke `WAITING_PAYMENT_VERIFICATION` (Menunggu Verifikasi).
- [ ] Tombol "Bayar" hilang setelah submit; status badge berubah ke "Menunggu Verifikasi".
- [ ] Customer menerima in-app notification konfirmasi submission.

## US-EI-04: Monitor Waiting Verification Status
**As a** Customer,
**I want to** see that my payment is being reviewed after I submit proof,
**So that** I know my submission was received and I am waiting for a decision.

**Acceptance Criteria:**
- [ ] Status badge: "Menunggu Verifikasi" (amber/teal soft pill).
- [ ] Tidak ada tombol aksi saat status `WAITING_PAYMENT_VERIFICATION`.
- [ ] Latest uploaded proof (file name + timestamp + nominal) visible di section Riwayat Pembayaran.
- [ ] Inline info banner di card (list) atau status banner (detail): "Bukti pembayaran sedang dalam proses verifikasi oleh tim kami. Harap tunggu konfirmasi."
- [ ] Page/list auto-refresh atau polling untuk reflect STS decision dalam < 1 menit.

## US-EI-05: Re-upload After Payment Rejection
**As a** Customer,
**I want to** upload a corrected payment proof after my previous one was rejected,
**So that** my payment can be re-verified and approved.

**Acceptance Criteria:**
- [ ] Status badge: "Pembayaran Ditolak" (rose soft pill).
- [ ] Rejection reason dari STS displayed di status banner dan di history attempt yang ter-reject.
- [ ] Tombol "Revisi Data" hanya visible saat status = `PAYMENT_REJECT`.
- [ ] Klik "Revisi Data" membuka form upload yang sama dengan US-EI-03 (termasuk nominal input — customer bisa adjust nominal jika rejection terkait jumlah).
- [ ] Validasi sama dengan US-EI-03.
- [ ] Setelah Submit: status keduanya berubah ke `WAITING_PAYMENT_VERIFICATION`.
- [ ] Customer menerima notification konfirmasi re-submission.
- [ ] Riwayat Pembayaran menampilkan semua attempt: yang ditolak dan yang baru.

## US-EI-06: View Paid EPB
**As a** Customer,
**I want to** see that my EPB is fully paid and confirmed,
**So that** I have a record that my payment was accepted and my voyage is authorized.

**Acceptance Criteria:**
- [ ] Status badge: "Lunas" (emerald soft pill).
- [ ] Confirmation timestamp (`confirmed_at`) ditampilkan di banner.
- [ ] Tidak ada action button.
- [ ] Riwayat Pembayaran tetap visible untuk reference.
- [ ] Page fully view-only.

## US-EI-07: Partial Payment Triggers Invoice
**As a** Customer,
**I want to** see that my partial EPB payment automatically generates an Invoice for the shortfall,
**So that** I understand my remaining obligation and can settle it separately via the Invoice tab.

**Acceptance Criteria:**
- [ ] Saat EPB ter-mark `Lunas` dengan `paid_amount < total_amount`: detail EPB menampilkan baris baru "Sisa pembayaran Rp {shortfall} telah ditagihkan via Invoice".
- [ ] Link "[Lihat Invoice →]" mengarah ke detail Invoice di M9c (`/customer/billing/invoice/:id`).
- [ ] Customer menerima notification: "Pembayaran EPB Anda telah dikonfirmasi. Sisa pembayaran sebesar Rp {shortfall} telah ditagihkan via Invoice. Silakan cek tab Invoice."
- [ ] Tab "Invoice" di menu EPB & Invoice menampilkan row baru dengan source "Shortfall EPB" + reference ke parent EPB.
- [ ] Status nominasi tetap `PAYMENT_CONFIRMED` (parsial EPB dianggap settle untuk voyage — Invoice tidak block voyage).
