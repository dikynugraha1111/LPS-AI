# M7 — Customer Authentication & Onboarding: User Stories

## US-CA-01: Customer Self-Registration
**As a** Pemilik Kapal / Shipper / Cargo Owner / PBM / Agen / Surveyor / Vendor Dozer,  
**I want to** register an account on the LPS Portal,  
**So that** I can submit vessel nominations.

**Acceptance Criteria:**
- [ ] Registration page menampilkan section "Company Information" dengan field: Customer Name, Type (checkboxes), NPWP, PIC Name, Phone Number, Email, Address, Note
- [ ] Field Customer Code ditampilkan sebagai read-only dengan label "Auto-generated on approval"
- [ ] Type menggunakan multi-select checkbox: Cargo Owner, Shipper, PBM, Agen, Surveyor, Vendor Dozer; minimal 1 harus dipilih
- [ ] Registration page menampilkan section "Required Documents" dengan 3 dokumen wajib: NPWP, NIP, Company Profile
- [ ] Setiap dokumen memiliki sub-field: File (required), Description (optional), Issue Date (optional), Expiry Date (optional)
- [ ] Tombol submit berlabel "Submit Registration"; tombol batal berlabel "Cancel"
- [ ] Field wajib yang kosong saat submit menampilkan error inline per field
- [ ] NPWP field menolak format selain XX.XXX.XXX.X-XXX.XXX
- [ ] Email duplikat menampilkan error: "Email sudah terdaftar"
- [ ] NPWP duplikat menampilkan error: "NPWP sudah terdaftar"
- [ ] Jika salah satu dari 3 dokumen wajib tidak diupload, form tidak dapat di-submit
- [ ] Submit berhasil → redirect ke halaman konfirmasi: "Akun Anda sedang menunggu validasi Admin. Kami akan menghubungi Anda melalui email setelah akun diaktifkan."
- [ ] Customer tidak dapat login sebelum akun diaktifkan

## US-CA-02: Admin Validates Customer Account
**As an** Admin Kepelabuhan,  
**I want to** review and activate or reject pending customer registrations,  
**So that** only verified businesses can use the LPS Portal.

**Acceptance Criteria:**
- [ ] Admin melihat daftar customer dengan status PENDING_VALIDATION di admin panel
- [ ] Daftar menampilkan: Customer Name, Type, NPWP, PIC Name, Email, Phone Number, tanggal registrasi
- [ ] Admin dapat membuka detail registrasi dan mengunduh/mereview dokumen yang diupload (NPWP, NIP, Company Profile)
- [ ] Tombol "Activate" mengubah status ke ACTIVE, meng-generate Customer Code, dan mentrigger STS Platform sync
- [ ] Tombol "Reject" membuka modal untuk mengisi alasan penolakan (opsional), lalu mengubah status ke REJECTED
- [ ] Customer yang diaktifkan menerima email: "Akun Anda telah diaktifkan. Customer Code Anda: [KODE]. Silakan login di [URL]."
- [ ] Customer yang ditolak menerima email dengan alasan penolakan (jika diisi)
- [ ] Aksi Admin tercatat di audit log dengan timestamp dan identitas Admin

## US-CA-03: Customer Login
**As an** active Customer,  
**I want to** log in to the LPS Portal,  
**So that** I can access the customer dashboard and submit nominations.

**Acceptance Criteria:**
- [ ] Halaman login menampilkan field Email dan Password
- [ ] Kredensial valid untuk akun ACTIVE → redirect ke /customer/dashboard
- [ ] Kredensial valid untuk akun PENDING_VALIDATION → error: "Akun Anda sedang menunggu validasi Admin"
- [ ] Kredensial valid untuk akun REJECTED → error: "Akun Anda ditolak. Hubungi Admin untuk informasi lebih lanjut."
- [ ] Kredensial tidak valid → error: "Email atau password salah"
- [ ] Session autentikasi persist saat refresh halaman
- [ ] Logout membersihkan session dan redirect ke /login
- [ ] Akses tanpa autentikasi ke /customer/* redirect ke /login
