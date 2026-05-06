# Business Requirements Document (BRD) — Local Port System Platform

**Disiapkan oleh:** PT Solutionlabs Group Indonesia  
**Disiapkan untuk:** GK Group & PT Tata Bumi Khatulistiwa (PT TBK)  
**Versi:** 3.0  
**Tanggal Update:** April 2026  
**Status:** [DRAFT]

---

## Reference Document

| Document | Version | Date |
|----------|---------|------|
| [BRD] Local Port System Platform V2 | 2.0 | Maret 2026 |
| [BRD] Ship to Ship Platform V2 | 2.0 | Maret 2026 |
| Mapping Module (Before & After) | — | April 2026 |

## Document Control

### Change Log (V2 → V3)

| No | Section | Item/Deskripsi Perubahan | Action | Sumber |
|----|---------|--------------------------|--------|--------|
| 1 | Section 2.1 | Modul Master Data & Configuration: dikeluarkan dari scope LPS. Pengelolaan data master (vessel, stakeholder, rate card) dipindahkan ke STS Platform. LPS hanya mempertahankan sub-modul System Configuration. | HAPUS / PINDAH | Mapping Module Revision |
| 2 | Section 2.1 | Modul Reporting & Analytics: dikeluarkan dari scope LPS. Digantikan oleh modul Monitoring & Visibility Dashboard yang berfokus pada monitoring real-time saja. | HAPUS / GANTI | Mapping Module Revision |
| 3 | Section 2.1 | Modul Billing & Service Charge Management: dikeluarkan dari scope LPS. Seluruh proses billing, invoice, dan PNBP dikelola oleh STS Platform. | HAPUS / PINDAH | Mapping Module Revision |
| 4 | Section 2.1 | Modul Nomination Request (Customer Portal): ditambahkan ke scope LPS. Customer mengajukan nominasi melalui LPS Portal, data diteruskan ke STS Platform untuk proses selanjutnya. | TAMBAH | Mapping Module Revision + BRD STS |
| 5 | Section 2.1 | Modul Monitoring & Visibility Dashboard: ditambahkan sebagai pengganti Reporting & Analytics, fokus pada real-time operational monitoring. | TAMBAH | Mapping Module Revision |
| 6 | Section 2.2 | Out of Scope: ditambahkan Master Data Management, Billing & Invoice, Reporting & Analytics sebagai tanggung jawab STS Platform. | UPDATE | Mapping Module Revision |
| 7 | Section 3.3 | Role & Access Control: role Finance dihapus dari LPS. Role Customer (Pemilik Kapal/Shipper) ditambahkan untuk Nomination. | UPDATE | Mapping Module Revision |
| 8 | Section 3.4 | Functional Requirements: hapus FR Billing (3.4.6), FR Reporting (3.4.7), FR Master Data (3.4.8). Tambah FR Nomination Request dan FR Monitoring Dashboard. | UPDATE | Mapping Module Revision |
| 9 | Section 3.5 | Non-Functional Requirements: tambah NFR integrasi API STS Platform. Hapus referensi billing accuracy. | UPDATE | Mapping Module Revision |
| 10 | Section 5 | Risk Area: hapus risk PNBP/billing formula. Tambah risk integrasi STS-LPS API dependency. | UPDATE | Mapping Module Revision |
| 11 | Section 6 | Acceptance Criteria: hapus SC billing. Tambah SC Nomination flow dan integrasi STS Platform. | UPDATE | Mapping Module Revision |

---

## 1. EXECUTIVE SUMMARY

### 1.1 Background

PT. TBK (Tata Bumi Khatulistiwa) sebagai Badan Usaha Pelabuhan (BUP) yang mengelola Wilayah Tertentu di Perairan (WTDP) untuk kegiatan Ship to Ship (STS) Transfer di Pelabuhan Bunati, Kabupaten Tanah Bumbu, Kalimantan Selatan. Dalam mendukung kegiatan STS tersebut, diperlukan sistem pengawasan, komunikasi, dan manajemen lalu lintas kapal yang andal dan terintegrasi.

Local Port Service (LPS) adalah sistem Vessel Traffic System (VTS) yang dirancang khusus untuk area perairan STS Bunati. LPS bukan merupakan bagian dari kegiatan STS itu sendiri, melainkan infrastruktur pendukung yang memastikan seluruh kegiatan STS berlangsung secara aman, efisien, dan sesuai regulasi. LPS berfungsi sebagai menara kontrol yang memantau posisi kapal, mengelola komunikasi, mendeteksi insiden, serta mengintegrasikan data dengan instansi terkait seperti Kemenhub (Inaportnet dan SIMoPEL) dan Bea Cukai.

Saat ini seluruh kegiatan pengawasan dan komunikasi di area perairan STS Bunati masih dilakukan secara manual dan belum terintegrasi secara digital. Kondisi ini menimbulkan beberapa tantangan operasional, antara lain:

- Tidak adanya visibilitas real-time terhadap posisi dan status kapal di area perairan STS.
- Risiko keterlambatan penanganan insiden akibat komunikasi yang tidak terstruktur.
- Kesulitan integrasi data dengan sistem pemerintah (Inaportnet, SIMoPEL, Bea Cukai).
- Tidak adanya mekanisme deteksi dan respons otomatis terhadap tumpahan minyak dan insiden pencemaran.
- Absennya rekam jejak digital (audit trail) untuk keperluan regulasi dan pelaporan MARPOL.
- Belum adanya portal terpadu bagi customer untuk mengajukan nominasi kapal sebelum memasuki area perairan.

Berdasarkan tantangan-tantangan di atas dan mengacu pada dokumen Studi Kelayakan Penetapan Lokasi WTOP PT TBK, diperlukan pengembangan sistem digital LPS yang terintegrasi untuk mendukung keselamatan, efisiensi, dan kepatuhan regulasi kegiatan STS.

> **Catatan Perubahan V3:** Fungsi billing/PNBP, master data entity, dan reporting periodik telah dikonsolidasikan ke STS Platform sebagai single system of record. LPS Platform berfokus pada core VTS operations: monitoring, komunikasi, keselamatan, integrasi pemerintah, dan penerimaan nominasi customer.

### 1.2 Tujuan

- Membangun sistem LPS digital yang mengintegrasikan seluruh fungsi pengawasan, komunikasi, dan manajemen lalu lintas kapal di area STS Bunati.
- Menyediakan visibilitas real-time posisi kapal melalui integrasi AIS (Automatic Identification System).
- Mengintegrasikan data dengan sistem pemerintah: Inaportnet dan SIMoPEL Kemenhub, serta Bea Cukai.
- Menyediakan sistem deteksi dan penanganan insiden serta tumpahan secara digital.
- Memastikan kepatuhan terhadap regulasi EDI-MARPOL dan standar safety management system.
- Menyediakan portal customer untuk pengajuan nominasi kapal yang terintegrasi dengan STS Platform.
- Menyediakan dashboard monitoring operasional real-time bagi seluruh stakeholder (KSOP, Bea Cukai, BUP, Pemilik Kapal).

### 1.3 Hubungan LPS dan STS Platform

Sistem LPS dan STS Platform beroperasi sebagai **dua sistem terpisah yang terhubung melalui API**. Pembagian tanggung jawab antar kedua sistem adalah sebagai berikut:

| Fungsi | LPS Platform | STS Platform |
|--------|-------------|--------------|
| Vessel Monitoring (AIS/RADAR) | ✅ Mengelola | Konsumsi data posisi kapal |
| Vessel Communication (VHF) | ✅ Mengelola | — |
| Government Integration | ✅ Mengelola | — |
| Incident & Emergency Mgmt | ✅ Mengelola | — |
| Weather Monitoring & Alert | ✅ Mengelola | Konsumsi data cuaca |
| Radar & Navigation Surveillance | ✅ Mengelola | — |
| Nomination Request (Customer) | ✅ Portal submit | Proses approval, scheduling, resource assignment |
| Monitoring & Visibility Dashboard | ✅ Real-time dashboard | — |
| System Configuration | ✅ Mengelola (API, weather threshold, alert, user/role) | — |
| Master Data (Vessel, Stakeholder, Rate Card) | Konsumsi via API | ✅ Mengelola (CRUD) |
| Billing & Service Charge | — | ✅ Mengelola |
| Reporting & Analytics | — | ✅ Mengelola |
| Voyage Lifecycle Mgmt | — | ✅ Mengelola |
| Field Operation Mgmt | — | ✅ Mengelola |

---

## 2. PROJECT SCOPE

### 2.1 In Scope

#### 1. Vessel Monitoring & AIS Integration

**Definisi**  
Modul inti yang mengelola pemantauan posisi dan status seluruh kapal yang beroperasi di area perairan STS Bunati secara real-time melalui integrasi dengan AIS Base Station.

**Tujuan**
- Menyediakan visibilitas real-time posisi kapal di area konsesi perairan.
- Mengintegrasikan data AIS dengan peta digital area STS Bunati.
- Mendeteksi keberadaan kapal yang tidak terdaftar di area perairan.
- Menjadi dasar data untuk seluruh modul operasional LPS.

**Cakupan**
- AIS Base Station Integration (2 arah: pencarian posisi kapal & komunikasi).
- Real-time vessel tracking dengan VHF Antenna & PPS Antenna.
- Peta digital area STS Bunati dengan overlay posisi kapal.
- Vessel status dashboard: kecepatan, arah, ETA, status operasional.
- Alert otomatis jika kapal memasuki/meninggalkan zona tertentu.
- History pergerakan kapal dengan timestamp.

#### 2. Vessel Communication Management

**Definisi**  
Modul yang mengelola seluruh komunikasi radio antara darat (LPS Station) dan kapal menggunakan VHF Coastal Radio terintegrasi dengan sistem AIS monitoring.

**Tujuan**
- Menyediakan sarana komunikasi 2 arah yang andal antara LPS Station dan kapal.
- Merekam seluruh komunikasi radio untuk keperluan audit dan investigasi.
- Mengintegrasikan komunikasi VHF dengan data posisi AIS.

**Cakupan**
- VHF Coastal Radio dengan integrasi software AIS monitoring.
- Perekaman otomatis seluruh komunikasi radio (voice recording).
- Log komunikasi dengan timestamp dan identifikasi kapal.
- Channel management: LPS Operator, VHF Operator, Security Operator.
- Notifikasi digital ke kapal terkait kondisi cuaca, insiden, dan informasi operasional.

#### 3. Government Integration (Inaportnet, SIMoPEL & Bea Cukai)

**Definisi**  
Modul yang mengelola interkoneksi sistem LPS dengan platform pemerintah: Inaportnet dan SIMoPEL Kemenhub serta sistem Bea Cukai, untuk memastikan kepatuhan regulasi dan kelancaran administrasi kepelabuhan.

**Tujuan**
- Memastikan seluruh data operasional kapal terlaporkan ke sistem Kemenhub secara otomatis.
- Memfasilitasi proses bea cukai untuk muatan yang dipindahkan dalam kegiatan STS.
- Mengurangi proses manual pelaporan ke instansi pemerintah.

**Cakupan**
- Integrasi dengan Inaportnet Kemenhub: pelaporan kedatangan/keberangkatan kapal.
- Integrasi dengan SIMoPEL Kemenhub: data pelayaran dan kepelabuhan.
- Integrasi dengan sistem Bea Cukai: pelaporan muatan dan dokumen manifes.
- EDI-MARPOL compliance reporting.
- Generate dan transmit dokumen elektronik kepelabuhan secara otomatis.
- Monitoring status pengiriman data ke seluruh sistem pemerintah.

#### 4. Incident & Emergency Management

**Definisi**  
Modul yang mengelola deteksi, pelaporan, dan penanganan insiden serta kondisi darurat di area perairan STS Bunati, termasuk insiden pencemaran dan tumpahan minyak.

**Tujuan**
- Menyediakan sistem peringatan dini dan respons cepat terhadap insiden.
- Mengelola prosedur tanggap darurat secara digital dan terstruktur.
- Memastikan seluruh insiden terdokumentasi untuk keperluan audit dan regulasi.

**Cakupan**
- Pelaporan insiden digital: form insiden, foto bukti, timestamp otomatis.
- Alert sistem untuk kondisi darurat: kebakaran, tumpahan, kecelakaan kapal.
- Oil Spill Response Management: aktivasi peralatan (Boom, Skimmer, Storage Tank).
- Eskalasi otomatis ke KSOP, Tim SAR, dan instansi terkait.
- Emergency response checklist digital.
- Post-incident report generation.

#### 5. Weather Monitoring & Alert System

**Definisi**  
Modul yang memantau kondisi cuaca dan gelombang di area perairan STS Bunati secara real-time, serta menghasilkan peringatan otomatis jika kondisi melebihi batas aman operasi.

**Tujuan**
- Memastikan keselamatan operasi STS dengan pemantauan cuaca real-time.
- Memberikan peringatan dini kepada seluruh kapal di area STS.
- Mendukung analisis produktivitas berdasarkan data cuaca historis.

**Cakupan**
- Integrasi Weather API untuk data cuaca real-time.
- Alert otomatis jika tinggi gelombang > 2.5m (warning) dan > 3m (operasi berhenti).
- Weather forecast H+24 dan H+48 untuk perencanaan operasi.
- Weather log otomatis terintegrasi dengan data operasional STS.
- Dashboard cuaca real-time untuk LPS Operator dan kapal.

#### 6. Radar & Navigation Surveillance

**Definisi**  
Modul yang mengelola sistem RADAR dan navigasi untuk pengawasan lalu lintas kapal di seluruh area cakupan LPS Bunati, terintegrasi dengan data AIS.

**Tujuan**
- Memastikan tidak ada blind spot dalam pengawasan lalu lintas kapal.
- Mengintegrasikan data RADAR dengan posisi AIS untuk akurasi tinggi.
- Mendeteksi kapal yang tidak memiliki transponder AIS aktif.

**Cakupan**
- Sistem RADAR dengan jangkauan sesuai wilayah konsesi perairan STS Bunati.
- Overlay RADAR dengan peta AIS untuk deteksi komprehensif.
- Identifikasi kapal tanpa AIS aktif.
- Integrasi data RADAR ke LPS Operator Station.
- Alert untuk kapal yang mendekati zona larangan.

#### 7. Customer Authentication & Onboarding — **BARU**

**Definisi**  
Modul yang mengelola seluruh proses registrasi, verifikasi, dan autentikasi customer (Pemilik Kapal/Shipper) sebelum dapat mengakses fitur LPS Portal. Merupakan entry point untuk seluruh modul Customer Portal.

**Tujuan**
- Menyediakan mekanisme self-registration bagi customer baru.
- Memfasilitasi validasi dan aktivasi akun oleh Admin.
- Menjamin hanya customer terverifikasi yang dapat mengakses portal.

**Cakupan**
- Registrasi akun customer baru melalui LPS Portal (self-registration) dengan field: Customer Name, NPWP, PIC Name, Phone Number, Email, Address, Note. Tipe Pelanggan di-default ke "Cargo Owner" oleh sistem secara otomatis.
- Upload 3 dokumen wajib saat registrasi: NPWP, NIP, Company Profile (masing-masing dengan Description, Issue Date, Expiry Date opsional).
- Customer Code di-generate otomatis oleh sistem setelah Admin menyetujui registrasi.
- Validasi dan review dokumen oleh Admin sebelum akun aktif.
- Login dan akses dashboard customer setelah akun terverifikasi.
- Data registrasi customer disinkronisasi ke STS Platform secara otomatis.

#### 8. Nomination Request Submission — **BARU**

**Definisi**  
Modul yang mengelola proses pengajuan nominasi kapal oleh customer melalui LPS Portal, dari pengisian form hingga pengiriman data ke STS Platform.

**Tujuan**
- Menyediakan form pengajuan nominasi yang terstruktur.
- Mendukung penyimpanan Draft sebelum submit final.
- Meneruskan data nominasi ke STS Platform untuk proses approval dan operasional selanjutnya.

**Cakupan**
- Form pengajuan nominasi dengan field: Vessel Name, ETA, Cargo Type, Cargo Quantity, Charterer, Estimasi Jumlah Barge.
- Upload dokumen pendukung: Rencana Kerja, Shipping Instruction, Surat Penunjukan PBM, Nomor PKK (input manual).
- Opsi simpan sebagai Draft atau Submit langsung.
- Generate Nomor Nominasi otomatis setelah submit.
- Pengiriman data nominasi ke STS Platform via API setelah submit.

#### 9. Nomination Status & Payment — **BARU**

**Definisi**  
Modul yang mengelola seluruh proses setelah nominasi disubmit: pemantauan status, tampilan EPB, upload bukti pembayaran, dan penerimaan hasil verifikasi dari STS Platform. Berakhir ketika status menjadi "Payment Confirmed" dan voyage dapat dimulai.

**Tujuan**
- Menyediakan visibilitas status nominasi secara real-time kepada customer.
- Memfasilitasi proses upload bukti pembayaran sesuai EPB dari STS Platform.
- Menerima dan menampilkan hasil verifikasi pembayaran dari STS Platform.

**Cakupan**

*Status Tracking*
- Customer dapat melihat status nominasi dengan lifecycle lengkap: **Draft** → **Submitted** → **Pending** (Menunggu proses di STS Platform) → **Approved** / **Need Revision** → **Waiting Payment Verification** → **Payment Confirmed** / **Payment Rejected**.
- Customer menerima notifikasi saat status nominasi berubah.
- Customer dapat melakukan revisi data nominasi jika status = Need Revision, kemudian re-submit ke STS Platform.

*EPB & Jadwal*
- Customer dapat melihat EPB (Estimasi Perkiraan Biaya) yang digenerate oleh STS Platform beserta nominal yang harus dibayarkan (view-only).
- Customer dapat melihat detail jadwal (anchor point, ETB, estimasi durasi) setelah nominasi di-approve.

*Payment Proof Upload*
- Customer mengupload bukti pembayaran (Proof of Payment) sesuai EPB yang diterima. Format: PDF, JPG, PNG (max 5MB per file).
- Data bukti bayar dikirimkan ke STS Platform via API. Status berubah menjadi "Waiting Payment Verification".
- STS Platform melakukan verifikasi. Jika Confirmed: status → "Payment Confirmed" dan voyage dapat dimulai. Jika Rejected: customer menerima notifikasi beserta alasan dan dapat mengupload ulang.

> **Batasan Modul:** LPS hanya berfungsi sebagai portal upload dan monitoring. Proses approval, resource assignment, scheduling, generate EPB, dan verifikasi pembayaran seluruhnya dilakukan oleh STS Platform.

#### 10. Customer Dashboard & Monitoring — **BARU**

**Definisi**  
Modul yang menyediakan halaman dashboard utama customer beserta fitur monitoring: ringkasan nominasi, pemantauan voyage yang sedang berjalan, dan akses informasi cuaca dan alert area STS Bunati.

**Tujuan**
- Menyediakan single-view dashboard bagi customer untuk memantau seluruh aktivitas mereka di LPS Portal.
- Menyediakan akses cuaca dan alert untuk mendukung perencanaan operasional customer.

**Cakupan**
- Customer Portal Dashboard: ringkasan nominasi aktif beserta status terkini, daftar voyage yang sedang berjalan, dan widget cuaca real-time area STS Bunati.
- Nomination Status page: daftar seluruh nominasi customer beserta status terkini, termasuk Draft.
- Voyage tracking: pemantauan voyage aktif (view-only) termasuk posisi kapal dari data AIS LPS.
- Weather & Alert: data cuaca real-time dan history alert area perairan STS Bunati (view-only).
- **Document Master:** repositori dokumen pribadi customer yang bersifat append-only (upload saja; tidak bisa hapus/edit). Dokumen yang diupload di form nominasi otomatis tersimpan ke Document Master dan dapat dipilih kembali untuk nominasi berikutnya.

> **Catatan:** Data dashboard customer bersifat view-only. Sumber data: LPS (cuaca, AIS) dan STS Platform (status nominasi, voyage). Document Master bersifat private per customer.

#### 11. Monitoring & Visibility Dashboard — **BARU (Pengganti Reporting & Analytics)**

**Definisi**  
Modul yang menyediakan dashboard monitoring operasional real-time untuk seluruh stakeholder LPS. Modul ini menggantikan Reporting & Analytics yang telah dipindahkan ke STS Platform, dengan fokus khusus pada visualisasi data operasional secara real-time.

**Tujuan**
- Menyediakan visibilitas operasional real-time kepada seluruh stakeholder.
- Menampilkan ringkasan status kapal, cuaca, insiden, dan komunikasi dalam satu dashboard terpadu.
- Mendukung pengambilan keputusan operasional yang cepat oleh LPS Operator.

**Cakupan**
- Dashboard utama menampilkan: jumlah kapal aktif, status cuaca, insiden aktif, status anchor point.
- Peta STS area real-time dengan overlay posisi kapal, zona, dan anchor point.
- Panel recent activity: kapal masuk/keluar, alert terbaru, komunikasi terakhir.
- Status integrasi API pemerintah (Inaportnet, SIMoPEL, Bea Cukai) — last sync status.
- Weather widget real-time dengan indikator level bahaya.
- Akses dashboard sesuai role (KSOP, Bea Cukai, Pemilik Kapal mendapat view terbatas sesuai hak akses masing-masing).

> **Catatan:** Laporan periodik (harian/bulanan/tahunan), laporan billing, dan export PDF/Excel dikelola oleh STS Platform melalui modul Operational & Financial Reporting.

#### 12. System Configuration (Sub-modul dari Master Data lama)

**Definisi**  
Sub-modul yang mengelola konfigurasi sistem LPS, pengelolaan user dan role, serta parameter operasional. Modul ini merupakan sisa dari Master Data & Configuration sebelumnya, setelah data master entity (vessel, stakeholder, rate card) dipindahkan ke STS Platform.

**Tujuan**
- Menyediakan pengelolaan user, role, dan permission untuk akses LPS.
- Menyediakan konfigurasi parameter sistem (weather threshold, alert config, API keys).
- Menyediakan audit trail untuk seluruh aktivitas sistem.

**Cakupan**

*User & Role Management*
- CRUD user dengan role assignment.
- Permission matrix per role dan per modul.
- Session management dan security policy.

*System Configuration*
- Weather threshold configuration (gelombang, angin, visibilitas).
- Alert & notifikasi configuration.
- Integrasi API configuration (Inaportnet, SIMoPEL, Weather API, AIS Provider, **STS Platform API**).
- AIS message template configuration.
- Data retention parameter configuration.

*Equipment Management*
- CRUD data equipment LPS (AIS Station, VHF Radio, RADAR, CCTV, Weather Station).
- Status operasional equipment: Operational, Maintenance, Out of Service, Standby.
- Jadwal next maintenance.

*Audit Log*
- Immutable log seluruh aktivitas sistem.
- Filter berdasarkan entity type, actor, dan rentang waktu.

> **Catatan:** Data master vessel, stakeholder, zona perairan, anchor point, dan rate card dikelola oleh STS Platform dan dikonsumsi oleh LPS melalui API sinkronisasi. LPS menyimpan local cache data master untuk operasional, dengan mekanisme sync berkala. Registrasi customer baru di LPS Portal akan disinkronisasi ke STS Platform secara otomatis.

### 2.2 Out of Scope

1. Pengadaan hardware fisik peralatan LPS (AIS Base Station, VHF Radio, RADAR, Tower).
2. Pengelolaan operasional STS Transfer (dikelola oleh STS Platform).
3. Sistem manajemen SDM internal PT TBK.
4. Data migration dari sistem manual/legacy yang ada.
5. Perubahan regulasi perpajakan di luar konfigurasi parameter.
6. **Master Data Management** — pengelolaan CRUD data vessel, stakeholder, zona perairan, anchor point, rate card (dikelola oleh STS Platform).
7. **Zones & Anchor Point Management** — pengelolaan CRUD zona perairan dan anchor point (dikelola oleh STS Platform). LPS mengkonsumsi data ini via API untuk keperluan monitoring.
8. **Billing & Service Charge Management** — seluruh proses kalkulasi biaya layanan, PNBP, invoice, dan payment reconciliation (dikelola oleh STS Platform).
9. **Reporting & Analytics** — laporan periodik operasional dan finansial, export PDF/Excel (dikelola oleh STS Platform).
10. **Nomination approval, scheduling, dan resource assignment** — proses approval nominasi, pengecekan ketersediaan, penjadwalan, dan assignment resource (dikelola oleh STS Platform).

### 2.3 Assumptions

Berikut adalah asumsi yang digunakan dalam penyusunan Business Requirement Document (BRD) ini.

1. Hardware LPS (AIS Base Station, VHF Coastal Radio, RADAR, Tower, Supporting System) telah tersedia atau dalam proses pengadaan terpisah.
2. AIS provider tersedia dengan API yang stabil dan mendukung integrasi 2 arah.
3. Koneksi internet antara LPS Station darat dan sistem cloud tersedia dengan bandwidth yang memadai.
4. Inaportnet dan SIMoPEL Kemenhub menyediakan API/EDI yang dapat diintegrasikan.
5. Sistem Bea Cukai menyediakan jalur integrasi data elektronik.
6. **STS Platform telah beroperasi dan menyediakan API yang stabil untuk sinkronisasi data master (vessel, stakeholder, zona perairan, rate card), penerimaan data nominasi, dan pengiriman status update ke LPS.**
7. LPS Operator tersedia 24/7 dengan akses ke Operator Station yang telah terpasang.
8. Tidak ada perubahan regulasi kepelabuhan yang signifikan selama pengembangan.
9. **Customer memiliki akses internet untuk mengakses LPS Portal (Nomination & Monitoring).**
10. **Format API integrasi LPS-STS telah disepakati oleh kedua tim development sebelum sprint dimulai.**

---

## 3. END-TO-END PROCESS

### 3.1 Modul LPS (12 modul)

**Sistem Eksternal yang terintegrasi dengan LPS:**
- AIS Provider → Data posisi kapal real-time
- Weather API → Data cuaca real-time
- Inaportnet / SIMoPEL Kemenhub → Pelaporan regulasi
- Bea Cukai → Laporan manifes
- **STS Platform → Sinkronisasi data master, pengiriman nominasi, penerimaan status update**

**Modul Inti LPS (12 modul):**

| No | Modul | Kategori |
|----|-------|----------|
| M1 | Vessel Monitoring & AIS Integration | Core Monitoring |
| M2 | Vessel Communication Management | Core Monitoring |
| M3 | Government Integration | Integrasi Pemerintah |
| M4 | Incident & Emergency Management | Keselamatan & Darurat |
| M5 | Weather Monitoring & Alert System | Keselamatan & Darurat |
| M6 | Radar & Navigation Surveillance | Navigasi |
| M7 | Customer Authentication & Onboarding | Customer Portal |
| M8 | Nomination Request Submission | Customer Portal |
| M9 | Nomination Status & Payment | Customer Portal |
| M10 | Customer Dashboard & Monitoring | Customer Portal |
| M11 | Monitoring & Visibility Dashboard | Monitoring |
| M12 | System Configuration | Konfigurasi |

### 3.2 End-to-End Flow — Skenario Operasional Utama

**PHASE 1 — Pre-Arrival (Nomination & Payment)**
1. Customer login ke LPS Portal.
2. Customer mengajukan nominasi: isi form, upload dokumen, submit.
3. Data nominasi dikirim ke STS Platform via API.
4. STS Platform melakukan pengecekan ketersediaan, approval, resource assignment.
5. Status update (Approved/Need Revision) diterima LPS dari STS Platform.
6. Setelah Approved, STS Platform mengirimkan EPB ke LPS → Customer melihat EPB di LPS Portal.
7. Customer mengupload bukti pembayaran (Proof of Payment) di LPS Portal sesuai EPB.
8. Bukti pembayaran dikirim ke STS Platform via API → Status menjadi "Waiting Payment Verification".
9. STS Platform memverifikasi pembayaran → Hasil (Confirmed/Rejected) dikirim ke LPS.
10. Jika Payment Confirmed → Voyage dapat dimulai.
11. Kapal mendekati area perairan → M1 (AIS) mendeteksi posisi.
12. M6 (RADAR) melakukan cross-check deteksi.
13. Alert otomatis jika kapal masuk zona → LPS Operator mendapat notifikasi.
14. M2 (VHF) membuka komunikasi dengan kapal.

**PHASE 2 — Operasi Berlangsung (Monitoring)**
1. M5 (Weather) memantau cuaca real-time.
2. M1 + M6 memantau posisi kapal selama operasi STS.
3. M2 memfasilitasi komunikasi darat-kapal.
4. Jika insiden terjadi → M4 (Incident) menangani: laporan, alert, eskalasi.
5. M8 (Dashboard) menampilkan seluruh status operasional secara real-time.

**PHASE 3 — Post-Operation & Compliance**
1. M3 (Government Integration) mengirim laporan ke Inaportnet, SIMoPEL, Bea Cukai.
2. Jika oil spill → M4 auto-generate EDI-MARPOL report.
3. Customer memantau status voyage via LPS Portal (view-only, data dari STS).

**PHASE 4 — Stakeholder Monitoring (Ongoing)**
- KSOP: monitoring aktivitas kapal dan laporan operasional via dashboard (view-only).
- Bea Cukai: akses data manifes dan laporan bea cukai (view-only).
- Pemilik Kapal/Customer: status kapal, status nominasi, tracking voyage (view-only).
- Admin/LPS Operator: full operational dashboard.

### 3.3 Role & Access Control

Sistem menerapkan Role-Based Access Control (RBAC) untuk memastikan setiap pengguna hanya dapat mengakses modul dan melakukan tindakan sesuai tanggung jawabnya.

#### 3.3.1. Role Definition (High-Level)

| Role | Deskripsi |
|------|-----------|
| LPS Operator | Memantau posisi kapal, mengelola komunikasi VHF, mencatat insiden, mengoperasikan sistem RADAR |
| VHF Operator | Mengelola komunikasi radio VHF dengan kapal, mencatat log komunikasi |
| Security Operator | Memantau keamanan area perairan, mengaktifkan prosedur darurat |
| Admin Kepelabuhan | Mengelola konfigurasi sistem, approval registrasi customer, integrasi pemerintah, monitoring operasional |
| KSOP (View) | Monitoring real-time aktivitas kapal dan laporan operasional di area konsesi |
| Bea Cukai (View) | Akses data manifes muatan dan laporan bea cukai |
| Customer / Pemilik Kapal | Mengajukan nominasi, mengupload bukti pembayaran, memonitor status nominasi dan voyage, melihat EPB |
| SuperAdmin | Pengelolaan user, role, permission, dan konfigurasi sistem |

> **Perubahan V3:** Role **Finance** dihapus dari LPS karena seluruh fungsi billing dan keuangan dikelola oleh STS Platform. Role **Customer / Pemilik Kapal** ditambahkan untuk mendukung modul Nomination Request.

#### 3.3.2. Module Access Matrix

| Modul | LPS Operator | VHF Operator | Security | Admin | KSOP | Bea Cukai | Customer | SuperAdmin |
|-------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Vessel Monitoring | Full | View | View | Full | View | — | — | View |
| Vessel Communication | Full | Full | View | Full | — | — | — | View |
| Government Integration | View | — | — | Full | View | View | — | View |
| Incident & Emergency | Full | — | Full | Full | View | — | — | View |
| Weather Monitoring | Full | View | View | Full | View | — | View | View |
| Radar & Navigation | Full | — | Full | Full | View | — | — | View |
| Customer Authentication | — | — | — | Approve Reg. | — | — | Register & Login | View |
| Nomination Submission | — | — | — | View | — | — | Full | View |
| Nomination Status & Payment | — | — | — | View | — | — | Full | View |
| Customer Dashboard & Monitoring | — | — | — | View | — | — | Full | View |
| Monitoring Dashboard | Full | View | View | Full | View | View | View (limited) | Full |
| System Configuration | — | — | — | Limited | — | — | — | Full |

### 3.4 Functional Requirements

#### 3.4.1. Vessel Monitoring & AIS Integration

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-VM-01 | Sistem harus menampilkan posisi real-time seluruh kapal di area perairan STS Bunati melalui integrasi AIS Base Station | System |
| FR-VM-02 | Sistem harus menampilkan informasi detail kapal: nama, IMO, kecepatan, arah, status, ETA | System |
| FR-VM-03 | Sistem harus memperbarui data posisi kapal dengan delay maksimal 5 menit | System |
| FR-VM-04 | Sistem harus menampilkan peta digital area STS Bunati dengan overlay posisi kapal real-time | LPS Operator |
| FR-VM-05 | Sistem harus memberikan alert otomatis jika kapal memasuki atau meninggalkan zona yang telah dikonfigurasi | System |
| FR-VM-06 | Sistem harus mendeteksi dan memberi alert jika ada kapal tanpa transponder AIS aktif melalui data RADAR | System |
| FR-VM-07 | Sistem harus menyimpan history pergerakan kapal dengan timestamp minimal 5 tahun | System |
| FR-VM-08 | LPS Operator harus dapat melakukan komunikasi 2 arah dengan kapal melalui sistem AIS (mengirim/menerima pesan) | LPS Operator |

#### 3.4.2. Vessel Communication Management

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-VC-01 | Sistem harus merekam otomatis seluruh komunikasi VHF antara LPS Station dan kapal beserta timestamp | System |
| FR-VC-02 | Sistem harus menampilkan log komunikasi yang dapat dicari berdasarkan waktu, kapal, dan operator | LPS Operator |
| FR-VC-03 | VHF Operator harus dapat memilih dan berpindah antar channel operasi sesuai kebutuhan | VHF Operator |
| FR-VC-04 | Sistem harus mendukung pengiriman notifikasi digital ke kapal terkait cuaca, insiden, dan instruksi operasional | LPS Operator |
| FR-VC-05 | Rekaman komunikasi harus dapat diakses kembali untuk keperluan investigasi dan audit | Admin |

#### 3.4.3. Government Integration

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-GI-01 | Sistem harus mengirimkan data kedatangan dan keberangkatan kapal ke Inaportnet Kemenhub secara otomatis | System |
| FR-GI-02 | Sistem harus mengirimkan data operasional pelayaran ke SIMoPEL Kemenhub sesuai format yang ditetapkan | System |
| FR-GI-03 | Sistem harus menghasilkan dan mengirimkan dokumen manifes muatan ke sistem Bea Cukai secara elektronik | System |
| FR-GI-04 | Sistem harus menghasilkan laporan EDI-MARPOL sesuai standar regulasi yang berlaku | System |
| FR-GI-05 | Admin harus dapat memantau status pengiriman data ke seluruh sistem pemerintah beserta timestamp konfirmasi | Admin |
| FR-GI-06 | Sistem harus memiliki mekanisme retry otomatis jika pengiriman data ke sistem pemerintah gagal | System |
| FR-GI-07 | Sistem harus menyimpan bukti pengiriman (acknowledgement) dari setiap integrasi pemerintah | System |

#### 3.4.4. Incident & Emergency Management

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-IE-01 | LPS Operator harus dapat membuat laporan insiden digital dengan field: jenis insiden, lokasi (koordinat GPS), deskripsi, foto bukti, kapal terlibat | LPS Operator |
| FR-IE-02 | Sistem harus mengirimkan alert insiden secara otomatis ke seluruh pihak terkait (KSOP, Tim SAR, Admin) sesuai level insiden | System |
| FR-IE-03 | Sistem harus menyediakan checklist respons darurat digital yang harus diisi oleh LPS Operator | LPS Operator |
| FR-IE-04 | Sistem harus mengelola status aktivasi peralatan Oil Spill Response (Boom, Skimmer, Storage Tank) saat terjadi tumpahan | Security Operator |
| FR-IE-05 | Sistem harus mencatat seluruh timeline penanganan insiden dari deteksi hingga selesai beserta petugas yang menangani | System |
| FR-IE-06 | Sistem harus menghasilkan Post-Incident Report secara otomatis berdasarkan data penanganan insiden | System |

#### 3.4.5. Weather Monitoring & Alert

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-WM-01 | Sistem harus menampilkan data cuaca real-time area perairan STS Bunati melalui integrasi Weather API | System |
| FR-WM-02 | Sistem harus memberikan warning alert jika tinggi gelombang melebihi 2.5 meter | System |
| FR-WM-03 | Sistem harus memberikan critical alert dan rekomendasi penghentian operasi jika tinggi gelombang melebihi 3 meter | System |
| FR-WM-04 | Sistem harus menampilkan weather forecast H+24 dan H+48 untuk perencanaan operasi | LPS Operator |
| FR-WM-05 | Sistem harus mencatat log cuaca otomatis setiap interval waktu yang dikonfigurasi | System |
| FR-WM-06 | LPS Operator harus dapat mengirimkan notifikasi cuaca ke seluruh kapal di area perairan secara massal | LPS Operator |

#### 3.4.6. Radar & Navigation Surveillance

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-RN-01 | Sistem harus menampilkan data RADAR dengan overlay peta AIS untuk deteksi komprehensif | System |
| FR-RN-02 | Sistem harus mengidentifikasi kapal tanpa AIS aktif melalui data RADAR | System |
| FR-RN-03 | Sistem harus memberikan alert untuk kapal yang mendekati zona larangan | System |
| FR-RN-04 | Sistem harus mengintegrasikan data RADAR ke LPS Operator Station | System |

#### 3.4.7. Customer Authentication & Onboarding — **BARU**

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-CA-01 | Sistem harus menyediakan halaman registrasi bagi customer baru dengan field: Customer Name, NPWP, PIC Name, Phone Number, Email, Address, Note; Tipe Pelanggan di-default ke "Cargo Owner" oleh sistem dan tidak ditampilkan sebagai field input di form registrasi | Customer |
| FR-CA-02 | Customer wajib mengupload 3 dokumen saat registrasi: NPWP (file), NIP (file), Company Profile (file), masing-masing dengan field tambahan Description (opsional), Issue Date (opsional), dan Expiry Date (opsional) | Customer |
| FR-CA-03 | Customer Code harus di-generate otomatis oleh sistem setelah Admin menyetujui registrasi; tidak diisi oleh customer | System |
| FR-CA-04 | Admin harus dapat memvalidasi, mengunduh dokumen, dan mengaktifkan atau menolak akun customer yang baru mendaftar | Admin |
| FR-CA-05 | Customer yang sudah terverifikasi harus dapat login dan mengakses dashboard customer | Customer |

#### 3.4.8. Nomination Request Submission — **BARU**

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-NS-01 | Sistem harus menyediakan form pengajuan nominasi dengan field: Vessel Name, ETA, Cargo Type, Cargo Quantity, Charterer, Estimasi Jumlah Barge | Customer |
| FR-NS-02 | Customer harus dapat mengupload dokumen pendukung nominasi: Rencana Kerja, Shipping Instruction, Surat Penunjukan PBM, Nomor PKK (input manual) | Customer |
| FR-NS-03 | Customer harus dapat menyimpan nominasi sebagai Draft sebelum melakukan Submit | Customer |
| FR-NS-04 | Sistem harus mengenerate Nomor Nominasi otomatis setelah customer melakukan Submit | System |
| FR-NS-05 | Sistem harus mengirimkan data nominasi ke STS Platform via API setelah customer melakukan Submit | System |

#### 3.4.9. Nomination Status & Payment — **BARU**

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-NP-01 | Sistem harus menerima status update nominasi dari STS Platform (Pending, Approved, Need Revision) dan menampilkannya ke customer. Status Pending ditampilkan dengan label "Menunggu proses di STS Platform". | System |
| FR-NP-02 | Customer harus dapat melihat EPB (Estimasi Perkiraan Biaya) yang digenerate oleh STS Platform beserta detail nominal yang harus dibayarkan | Customer |
| FR-NP-03 | Customer harus dapat melihat detail jadwal (anchor point, ETB, estimasi durasi) setelah nominasi di-approve oleh STS Platform | Customer |
| FR-NP-04 | Sistem harus mengirim notifikasi ke customer saat status nominasi berubah | System |
| FR-NP-05 | Customer harus dapat melakukan revisi data nominasi jika status = Need Revision, kemudian re-submit | Customer |
| FR-NP-06 | Customer harus dapat mengupload bukti pembayaran (Proof of Payment) melalui LPS Portal sesuai dengan EPB yang diterima dari STS Platform. Format yang didukung: PDF, JPG, PNG (max 5MB) | Customer |
| FR-NP-07 | Sistem harus mengirimkan data bukti pembayaran yang diupload customer ke STS Platform via API untuk proses verifikasi | System |
| FR-NP-08 | Setelah customer mengupload bukti pembayaran, status nominasi harus berubah menjadi "Waiting Payment Verification" | System |
| FR-NP-09 | Sistem harus menerima hasil verifikasi pembayaran dari STS Platform (Confirmed/Rejected) dan menampilkannya ke customer | System |
| FR-NP-10 | Jika pembayaran ditolak oleh STS Platform, customer harus menerima notifikasi beserta alasan penolakan dan dapat mengupload ulang bukti pembayaran yang benar | Customer |
| FR-NP-11 | Setelah pembayaran dikonfirmasi oleh STS Platform, status nominasi berubah menjadi "Payment Confirmed" dan voyage dapat dimulai | System |

#### 3.4.10. Customer Dashboard & Monitoring — **BARU**

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-CD-01 | Customer harus dapat memantau status voyage yang sedang berjalan (view-only) termasuk posisi kapal dari data AIS | Customer |
| FR-CD-02 | Sistem harus menampilkan halaman Nomination Status dengan daftar seluruh nominasi customer beserta status terkini, termasuk nominasi berstatus Draft | Customer |
| FR-CD-03 | Sistem harus menampilkan Customer Portal Dashboard setelah customer login, berisi: ringkasan nominasi aktif beserta status terkini, daftar voyage yang sedang berjalan (view-only, data dari STS Platform), dan widget cuaca real-time area STS Bunati | Customer |
| FR-CD-04 | Customer harus dapat melihat data cuaca real-time dan history alert area perairan STS Bunati melalui Customer Portal (view-only) | Customer |
| FR-CD-05 | Sistem harus menyediakan halaman Document Master sebagai repositori dokumen pribadi customer; customer hanya dapat mengupload dokumen baru — tidak dapat menghapus atau mengedit dokumen yang sudah diupload (append-only) | Customer |
| FR-CD-06 | Dokumen yang diupload di halaman Document Master harus menampilkan metadata: nama file, tanggal upload, ukuran file, tipe file; customer dapat mengunduh atau preview dokumen (read-only) | Customer |
| FR-CD-07 | Pada form pengajuan nominasi baru, customer harus dapat memilih antara dua metode upload dokumen: (1) Upload dari komputer lokal — file otomatis tersimpan ke Document Master; (2) Pilih dari Document Master yang sudah ada | Customer |
| FR-CD-08 | Document Master harus bersifat private per customer; dokumen satu customer tidak boleh dapat diakses oleh customer lain | System |

#### 3.4.11. Monitoring & Visibility Dashboard — **BARU**

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-MD-01 | Dashboard harus menampilkan jumlah dan posisi kapal aktif real-time di area STS beserta status (Verified/Unknown) | Admin |
| FR-MD-02 | Dashboard harus menampilkan status cuaca real-time dengan indikator level bahaya (Normal/Warning/Critical) | LPS Operator |
| FR-MD-03 | Dashboard harus menampilkan status insiden aktif dan progres penanganan | LPS Operator |
| FR-MD-04 | Dashboard harus menampilkan status anchor point (Available/Occupied) secara real-time | Admin |
| FR-MD-05 | Dashboard harus menampilkan recent activity: kapal masuk/keluar area, alert terbaru, komunikasi VHF terakhir | LPS Operator |
| FR-MD-06 | Dashboard harus menampilkan status koneksi API eksternal (AIS, Weather, Inaportnet, SIMoPEL, Bea Cukai, STS Platform) | Admin |
| FR-MD-07 | KSOP, Bea Cukai, dan Customer harus dapat mengakses dashboard sesuai hak akses masing-masing (view-only) | System |

#### 3.4.12. System Configuration

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-SC-01 | SuperAdmin harus dapat mengelola user, role, dan permission | SuperAdmin |
| FR-SC-02 | SuperAdmin harus dapat mengkonfigurasi parameter sistem (weather threshold, alert config, API keys, AIS message templates, data retention) | SuperAdmin |
| FR-SC-03 | Admin harus dapat mengelola data equipment LPS (AIS Station, VHF Radio, RADAR) beserta status operasionalnya | Admin |
| FR-SC-04 | Sistem harus menyimpan seluruh aktivitas dalam immutable audit log yang dapat difilter berdasarkan entity type, actor, dan rentang waktu | System |
| FR-SC-05 | Sistem harus melakukan sinkronisasi data master (vessel, stakeholder, zona, anchor point) dari STS Platform secara berkala melalui API | System |
| FR-SC-06 | Sistem harus menyimpan local cache data master dari STS Platform untuk memastikan operasional berjalan meskipun koneksi ke STS terputus sementara | System |

### 3.5 Non-Functional Requirements

1. **Availability (Mandatory)**  
   Sistem harus beroperasi 24/7 dengan uptime minimal 99.5%. LPS adalah sistem keselamatan kritis yang tidak boleh down saat kapal beroperasi.

2. **Real-time Performance**  
   Data posisi kapal (AIS) harus diperbarui dengan delay maksimal 5 menit. Alert cuaca dan insiden harus terkirim dalam waktu kurang dari 1 menit setelah trigger.

3. **Auditability**  
   Semua event harus ter-timestamp. Tidak boleh ada penghapusan permanen. Semua perubahan data tercatat dalam immutable audit log.

4. **Data Retention**  
   Data operasional kapal, komunikasi, dan insiden harus disimpan minimal 5 tahun sesuai regulasi kepelabuhan.

5. **Security**  
   Semua akses menggunakan autentikasi (username/password). RBAC diterapkan ketat. Data sensitif dienkripsi. Komunikasi menggunakan protokol aman (HTTPS/TLS).

6. **Fallback Manual**  
   Sistem harus memiliki prosedur fallback manual jika integrasi API eksternal (AIS, Weather, Inaportnet) mengalami gangguan.

7. **Scalability**  
   Sistem harus dapat menangani minimal 50 kapal simultan di area perairan dan 30 user concurrent.

8. **Compatibility**  
   Web aplikasi kompatibel dengan browser modern. Dapat diakses dari berbagai ukuran layar (desktop, tablet).

9. **STS Platform API Integration (BARU)**  
   - LPS harus mampu mengirim data nominasi ke STS Platform via REST API dengan latency maksimal 3 detik.
   - LPS harus mampu menerima webhook/callback dari STS Platform untuk status update nominasi.
   - LPS harus memiliki mekanisme retry (max 3x) jika pengiriman data ke STS Platform gagal, dengan interval exponential backoff.
   - LPS harus menyimpan local cache data master dari STS Platform dengan mekanisme sinkronisasi berkala (interval dikonfigurasi, default setiap 15 menit).
   - Jika koneksi ke STS Platform terputus, LPS harus tetap beroperasi menggunakan data cache terakhir. Sistem harus menampilkan indikator status koneksi STS Platform di dashboard.

---

## 4. PROJECT CONSTRAINT

### 4.1. Operational Area Constraints

| Constraint | Implikasi terhadap Sistem LPS |
|-----------|-------------------------------|
| Jarak 9 Nautical Mile dari pantai | Infrastruktur komunikasi data antara LPS Station darat dan kapal harus menggunakan teknologi yang mampu menjangkau jarak tersebut (VHF, AIS, Satellite backup) |
| 5 Anchor Points aktif | Sistem monitoring harus menampilkan status real-time seluruh 5 titik berlabuh secara simultan |
| 68 hari cuaca buruk per tahun | Weather log dan alert system menjadi komponen kritis. Sistem harus dapat memproses dan menyimpan data cuaca dengan volume tinggi |
| Max gelombang aman: 2.5m | Threshold 2.5m sebagai warning dan 3m sebagai trigger penghentian operasi harus dikonfigurasi di sistem alert |
| Operasi 24/7 | Sistem LPS harus beroperasi tanpa henti. Diperlukan redundansi sistem dan SOP penanganan downtime |

### 4.2. Technical Constraints

- **Ketergantungan pada Hardware:** Sistem digital LPS bergantung pada ketersediaan dan fungsi hardware fisik (AIS Station, VHF Radio, RADAR). Diperlukan monitoring status hardware secara berkala.
- **Integrasi API Pemerintah:** Kualitas dan ketersediaan integrasi bergantung pada stabilitas API Inaportnet, SIMoPEL, dan Bea Cukai. Sistem harus memiliki fallback dan retry mechanism.
- **Koneksi Internet:** Koneksi internet di lokasi operasional harus memadai untuk transmisi data AIS dan komunikasi sistem. Diperlukan backup koneksi (misalnya VSAT).
- **Ketergantungan STS Platform API (BARU):** LPS bergantung pada ketersediaan API STS Platform untuk sinkronisasi data master dan pengiriman nominasi. Sistem harus memiliki mekanisme caching dan graceful degradation jika STS Platform tidak tersedia.

### 4.3. Regulatory Constraints

- **Data Retention:** Data operasional harus disimpan minimal 5 tahun sesuai regulasi kepelabuhan Indonesia.
- **EDI-MARPOL Compliance:** Sistem harus menghasilkan laporan sesuai standar MARPOL untuk setiap kegiatan yang berpotensi pencemaran laut.
- **Standar Kemenhub:** Sistem AIS dan VHF harus memenuhi standar Direktorat Kenavigasian Kemenhub yang berlaku.
- **Audit Trail:** Semua perubahan data harus tercatat dalam immutable event log untuk mendukung audit oleh instansi berwenang.

---

## 5. RISK AREA

| No | Risk | Level | Mitigasi |
|----|------|-------|----------|
| 1 | API Inaportnet/SIMoPEL belum tersedia atau tidak stabil | High | Konfirmasi ketersediaan API sejak awal proyek. Siapkan fallback manual. |
| 2 | AIS provider belum dipastikan dan kualitas data belum tervalidasi | High | Lakukan evaluasi dan seleksi AIS provider sebelum development dimulai. |
| 3 | Hardware LPS belum tersedia saat software siap | Medium | Koordinasi jadwal pengadaan hardware dengan jadwal pengembangan software. |
| 4 | Regulasi EDI-MARPOL dan format pelaporan belum terdefinisi lengkap | Medium | Konfirmasi format pelaporan dengan Kemenhub sebelum modul integrasi dikembangkan. |
| 5 | Koneksi internet di lokasi operasional tidak memadai | High | Lakukan survei konektivitas di lokasi. Siapkan solusi backup (VSAT). |
| 6 | **STS Platform API belum tersedia atau tidak stabil saat LPS development dimulai (BARU)** | **High** | **Definisikan API contract (OpenAPI spec) bersama tim STS sebelum development. Gunakan mock API selama STS belum ready. Implementasikan local cache dan fallback mode.** |
| 7 | **Format dan struktur data integrasi LPS-STS belum disepakati (BARU)** | **Medium** | **Lakukan workshop teknis bersama tim STS untuk menyepakati data mapping, payload structure, dan error handling sebelum sprint dimulai.** |

---

## 6. ACCEPTANCE CRITERIA

### 6.1. Success Criteria

| ID | Success Criteria | Metric |
|----|-----------------|--------|
| SC-01 | Sistem berhasil go-live dengan minimum critical issue | 0 critical bugs pada go-live |
| SC-02 | 100% modul dalam scope selesai dan berfungsi sesuai BRD | UAT sign-off per modul |
| SC-03 | Integrasi AIS menampilkan posisi kapal real-time dengan delay < 5 menit | Integration Test |
| SC-04 | Integrasi Weather API berfungsi dengan akurasi > 95% | Integration Test |
| SC-05 | Alert insiden dan cuaca terkirim ke stakeholder dalam waktu < 1 menit setelah trigger | Performance Test |
| SC-06 | Integrasi Inaportnet dan SIMoPEL berfungsi dan data terkonfirmasi diterima oleh sistem Kemenhub | Integration Test |
| SC-07 | **Customer dapat mengajukan nominasi melalui LPS Portal dan data berhasil diterima oleh STS Platform (BARU)** | **End-to-end Integration Test** |
| SC-08 | **Status update nominasi dari STS Platform berhasil ditampilkan ke customer di LPS Portal (BARU)** | **Integration Test** |
| SC-09 | **Sinkronisasi data master dari STS Platform berjalan dengan benar dan LPS dapat beroperasi menggunakan cache saat koneksi terputus (BARU)** | **Failover Test** |
| SC-10 | User Acceptance Test (UAT) mencapai tingkat kepuasan minimal 4 dari 5 | Feedback Survey |

### 6.2. Deliverables and Milestones (On Hold)

| Milestone | Deliverables | Release Target |
|-----------|-------------|----------------|
| Project Initiation & BRD Finalization | BRD V3 disetujui stakeholder, Project timeline | 24 April 2026 |
| UI/UX Design & Prototype Completion | Clickable prototype, Design system | 24 April 2026 |
| Core Monitoring & Communication | Modul Vessel Monitoring, Modul Vessel Communication, System Configuration | 8 Mei 2026 |
| Government Integration & Safety | Modul Government Integration, Modul Incident & Emergency Mgmt, Weather Monitoring | 22 Mei 2026 |
| Customer Portal & Dashboard | Modul Nomination Request (Customer Portal), Modul Monitoring & Visibility Dashboard, Modul Radar & Navigation Surveillance | 5 Juni 2026 |
| Integration Validation & SIT | Weather API, AIS Integration, Berthing Map + AIS, Government Integration, **STS Platform API Integration** | 19 Juni 2026 |
| User Acceptance Test (UAT) Completion | UAT Report, Bug fixes, User manual, Training materials | 19 Juni 2026 |
| GO-LIVE | Production deployment, Data migration, Go-live report | 19 Juni 2026 |
| User Training | | 30 Juni 2026 |

### 6.3. Definition of Done (DoD)

1. Semua mockup disetujui stakeholder sebelum masuk sprint selanjutnya, prototype dapat diakses secara online.
2. Semua modul dalam sprint telah di-demo dan mendapatkan UAT sign-off.
3. Integrasi AIS menampilkan data posisi kapal real-time yang tervalidasi.
4. Integrasi Inaportnet dan SIMoPEL terkonfirmasi berfungsi dengan acknowledgement dari sistem Kemenhub.
5. **Nominasi yang disubmit customer di LPS Portal berhasil diterima dan diproses oleh STS Platform (end-to-end verified).**
6. **Data master dari STS Platform berhasil disinkronisasi dan tersimpan di local cache LPS.**
7. Sistem dapat diakses di production environment oleh seluruh user.
8. Dokumentasi teknis dan user manual lengkap diserahkan.
9. Tidak ada critical issues selama 30 hari setelah go-live.

---

## APPENDIX A: Modul yang Dipindahkan ke STS Platform

Berikut adalah modul yang sebelumnya ada di BRD LPS V2 dan telah dipindahkan ke STS Platform di V3:

### A.1 Master Data & Configuration (Dipindahkan)

Data master entity berikut kini dikelola oleh STS Platform:
- Vessel Master: MV, Tug Boat, Barge, data identitas kapal (IMO, Call Sign, Flag)
- Stakeholder Master: KSOP, Bea Cukai, BUP, Pemilik Kapal, Badan Usaha Pemanduan
- Zona Perairan: batas area konsesi, zona berlabuh, zona larangan
- Anchor Point Master
- Rate Card layanan kepelabuhan

LPS mengkonsumsi data ini melalui API sinkronisasi dari STS Platform.

### A.2 Billing & Service Charge Management (Dipindahkan)

Seluruh fungsi berikut kini dikelola oleh STS Platform:
- Kalkulasi biaya layanan kepelabuhan
- Kalkulasi PNBP
- Generate tagihan otomatis
- Approval workflow billing
- Invoice delivery
- Integrasi ERP

### A.3 Reporting & Analytics (Dipindahkan)

Seluruh fungsi berikut kini dikelola oleh STS Platform:
- Laporan lalu lintas kapal periodik
- Laporan insiden periodik
- Laporan cuaca periodik
- Laporan billing dan pendapatan
- Export laporan ke PDF dan Excel

LPS hanya menyediakan Monitoring & Visibility Dashboard untuk real-time operational monitoring.

---

*— End of Document —*
