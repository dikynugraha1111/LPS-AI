# M7 — Customer Authentication & Onboarding: User Stories

## US-CA-01: Customer Self-Registration
**As a** Pemilik Kapal / Shipper / Cargo Owner / PBM / Agen / Surveyor / Vendor Dozer,  
**I want to** register an account on the LPS Portal,  
**So that** I can submit vessel nominations.

**Acceptance Criteria:**
- [ ] Registration page menampilkan section "Company Information" dengan field: Customer Name, NPWP, PIC Name, Phone Number, Email, Address, Note
- [ ] Field Customer Code ditampilkan sebagai read-only dengan label "Auto-generated on approval"
- [ ] Field Tipe Pelanggan tidak ditampilkan di form; sistem otomatis menetapkan "Cargo Owner" saat akun dibuat
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

## US-CA-02: Admin Views Customer Management
**As an** Admin Kepelabuhan,  
**I want to** see a list of all customers and their status,  
**So that** I can monitor and manage customer accounts from one place.

**Acceptance Criteria:**
- [ ] Halaman Customer Management menampilkan daftar seluruh customer dengan kolom: Customer Name, NPWP, PIC Name, Email, Phone Number, Status, Registration Source (Self / Admin), Created At
- [ ] Daftar dapat difilter berdasarkan status: All, PENDING_VALIDATION, ACTIVE, REJECTED
- [ ] Setiap baris memiliki tombol/link untuk membuka halaman detail customer
- [ ] Halaman Customer Management menampilkan tombol "Tambah Customer" untuk Admin add customer manual

## US-CA-03: Admin Approves Customer Registration
**As an** Admin Kepelabuhan,  
**I want to** review and approve pending customer registrations,  
**So that** only verified businesses can access the LPS Portal.

**Acceptance Criteria:**
- [ ] Halaman detail customer menampilkan seluruh data Company Information dan daftar dokumen yang diupload
- [ ] Admin dapat preview dan download setiap dokumen (NPWP, NIP, Company Profile)
- [ ] Tombol "Approve" hanya tampil jika status = PENDING_VALIDATION
- [ ] Klik Approve → sistem generate Customer Code, ubah status ke ACTIVE, kirim email aktivasi ke customer, trigger STS Platform sync
- [ ] Email aktivasi berisi: Customer Code, URL login, informasi akun
- [ ] Aksi Approve tercatat di audit log dengan timestamp dan identitas Admin

## US-CA-04: Admin Rejects Customer Registration
**As an** Admin Kepelabuhan,  
**I want to** reject a customer registration that does not meet requirements,  
**So that** unqualified entities cannot access the portal.

**Acceptance Criteria:**
- [ ] Tombol "Reject" hanya tampil jika status = PENDING_VALIDATION
- [ ] Klik Reject → modal muncul dengan field alasan penolakan (opsional, textarea)
- [ ] Konfirmasi Reject → status berubah ke REJECTED, email penolakan dikirim ke customer
- [ ] Email penolakan menyertakan alasan penolakan jika diisi Admin
- [ ] Aksi Reject tercatat di audit log dengan timestamp, identitas Admin, dan alasan (jika ada)

## US-CA-05: Admin Adds Customer Manually
**As an** Admin Kepelabuhan,  
**I want to** add a new customer directly from the Admin Dashboard,  
**So that** I can onboard customers who were registered outside the self-registration flow.

**Acceptance Criteria:**
- [ ] Form "Tambah Customer" menampilkan field yang sama dengan form registrasi customer (Company Information + 3 dokumen standar)
- [ ] Admin wajib mengisi semua field wajib (Customer Name, NPWP, PIC Name, Phone Number, Email) dan mengupload 3 dokumen standar
- [ ] Admin mengisi password awal untuk akun customer
- [ ] Akun yang disimpan langsung berstatus ACTIVE; Customer Code langsung di-generate
- [ ] Sistem trigger STS Platform sync setelah akun berhasil disimpan
- [ ] Notifikasi email dikirim ke customer berisi informasi akun (email + password awal)
- [ ] Form menampilkan section "Custom Documents" (opsional): Admin dapat menambahkan satu atau lebih dokumen tambahan dengan field Document Name/Label (wajib), File (wajib), Description, Issue Date, Expiry Date
- [ ] Setiap custom document yang ditambahkan ditandai dengan `is_custom = true` di database
- [ ] Aksi pembuatan customer oleh Admin tercatat di audit log

## US-CA-06: Customer Login
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
