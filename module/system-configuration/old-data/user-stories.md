# M9 — User Stories

8 user stories untuk modul Master Data & Configuration (US-MD-001 s/d US-MD-008).

## US-MD-001: Vessel Data Management

**Sub-module:** Vessel Master
**Epic:** Vessel Data Management
**Related FR:** FR-MD-01

> Sebagai Admin Kepelabuhan, saya ingin dapat mengelola data master kapal (MV, Tug Boat, Barge) lengkap dengan spesifikasi teknisnya, sehingga semua modul operasional LPS memiliki referensi data kapal yang akurat dan konsisten.

**Acceptance Criteria:**

- AC-01: Admin dapat melakukan CRUD untuk data kapal: nama, IMO, MMSI, Call Sign, flag, tipe kapal, kapasitas, dan status aktif
- AC-02: Admin tidak dapat menghapus kapal yang masih memiliki history voyage aktif (hanya dapat dinonaktifkan)
- AC-03: Setiap perubahan data kapal tersimpan dalam audit trail dengan timestamp dan nama Admin
- AC-04: Data kapal dapat dicari dan difilter berdasarkan: nama, IMO, tipe, atau status
- AC-05: Duplikasi IMO atau MMSI tidak diizinkan oleh sistem dan akan ditolak dengan pesan error yang informatif

---

## US-MD-002: Stakeholder Data Management

**Sub-module:** Stakeholder Master
**Epic:** Stakeholder Data Management
**Related FR:** FR-MD-02

> Sebagai Admin Kepelabuhan, saya ingin mengelola data master seluruh stakeholder (KSOP, Bea Cukai, BUP, Pemilik Kapal, Badan Usaha Pemanduan) dalam satu tempat, sehingga informasi kontak dan akses setiap stakeholder selalu terkini dan terkelola baik.

**Acceptance Criteria:**

- AC-01: Admin dapat melakukan CRUD untuk data stakeholder: nama, tipe, kontak person, email, telepon, dan status aktif
- AC-02: Setiap stakeholder dapat dihubungkan dengan satu atau lebih akun user untuk akses portal
- AC-03: Admin dapat menonaktifkan stakeholder tanpa menghapus history yang berkaitan
- AC-04: Perubahan data stakeholder tersimpan dalam audit trail
- AC-05: Notifikasi dikirimkan kepada stakeholder ketika data akun mereka diperbarui oleh Admin

---

## US-MD-003: Navigation Zone Configuration

**Sub-module:** Waterway Zone Management
**Epic:** Navigation Zone Configuration
**Related FR:** FR-MD-03, FR-VM-04

> Sebagai Admin Kepelabuhan, saya ingin mendefinisikan dan mengelola zona-zona perairan di area konsesi STS Bunati pada peta digital, sehingga batas area konsesi, zona berlabuh, dan zona larangan terkonfigurasi dengan benar di seluruh sistem.

**Acceptance Criteria:**

- AC-01: Admin dapat membuat, mengubah, dan menonaktifkan zona perairan menggunakan drawing tool pada peta interaktif
- AC-02: Jenis zona yang dapat dikonfigurasi: Area Konsesi, Anchor Point, Zona Larangan, Jalur Masuk, dan Buffer Zone
- AC-03: Perubahan zona berlaku pada seluruh modul (monitoring, alert, RADAR) setelah disimpan
- AC-04: Admin dapat menetapkan warna dan label unik untuk setiap zona
- AC-05: History perubahan konfigurasi zona tersimpan dalam audit trail
- AC-06: Zona yang sedang aktif digunakan oleh kapal tidak dapat dihapus sampai penggunaan selesai

---

## US-MD-004: Service Tariff Administration

**Sub-module:** Rate Card Management
**Epic:** Service Tariff Administration
**Related FR:** FR-MD-04

> Sebagai Finance TBK, saya ingin mengelola rate card layanan kepelabuhan dengan effective date yang fleksibel, sehingga perubahan tarif dapat disiapkan lebih awal dan diaktifkan otomatis pada tanggal yang tepat.

**Acceptance Criteria:**

- AC-01: Finance dapat membuat rate card baru dengan effective date di masa depan
- AC-02: Sistem otomatis menggunakan rate card yang berlaku sesuai effective date pada saat layanan diberikan
- AC-03: Rate card lama tidak dapat dihapus dan tetap tersimpan untuk audit historis
- AC-04: Finance dapat membandingkan perbedaan rate card lama vs baru dalam tampilan side-by-side
- AC-05: Perubahan rate card memerlukan approval Supervisor/Manager sebelum aktif
- AC-06: Notifikasi dikirim ke Admin Kepelabuhan 7 hari sebelum rate card baru berlaku

---

## US-MD-005: Access Control (User & Role Management)

**Sub-module:** User & Role Management
**Epic:** Access Control
**Related FR:** FR-MD-05

> Sebagai SuperAdmin, saya ingin mengelola user, role, dan hak akses seluruh pengguna sistem LPS, sehingga setiap pengguna hanya dapat mengakses fitur dan data yang sesuai dengan tanggung jawabnya sesuai prinsip least privilege.

**Acceptance Criteria:**

- AC-01: SuperAdmin dapat melakukan CRUD untuk user account dengan field: nama, email, username, role, dan status aktif
- AC-02: SuperAdmin dapat menetapkan satu atau lebih role kepada setiap user
- AC-03: Role tersedia minimal: LPS Operator, VHF Operator, Security Operator, Admin Kepelabuhan, Finance, KSOP, Bea Cukai, Pemilik Kapal, SuperAdmin
- AC-04: Setiap role memiliki konfigurasi permission: modul yang diakses dan aksi diizinkan (view/create/edit/delete)
- AC-05: SuperAdmin dapat menonaktifkan user tanpa menghapus history aktivitasnya
- AC-06: Log seluruh aktivitas user tersimpan dalam audit trail system

---

## US-MD-006: System Parameter Configuration

**Sub-module:** System Parameter Configuration
**Epic:** System Settings
**Related FR:** FR-MD-06

> Sebagai SuperAdmin, saya ingin mengkonfigurasi parameter sistem seperti weather threshold, zona perairan, dan konfigurasi notifikasi dari dashboard admin, sehingga sistem dapat disesuaikan dengan kebutuhan operasional tanpa harus mengubah kode program.

**Acceptance Criteria:**

- AC-01: SuperAdmin dapat mengkonfigurasi: weather threshold (warning dan critical level), interval update cuaca, dan retention period data
- AC-02: Konfigurasi notifikasi dapat diatur per event: insiden, cuaca, kedatangan kapal, payment reminder, dan system alert
- AC-03: Perubahan konfigurasi memerlukan konfirmasi sebelum disimpan dan berlaku
- AC-04: Log perubahan konfigurasi sistem tersimpan dengan timestamp dan nama SuperAdmin
- AC-05: Konfigurasi yang disimpan langsung berlaku tanpa perlu restart sistem

---

## US-MD-007: Equipment Inventory

**Sub-module:** Equipment Master
**Epic:** Equipment Inventory
**Related FR:** FR-MD-05, FR-RD-04

> Sebagai Admin Kepelabuhan, saya ingin mengelola data master semua peralatan LPS (AIS Station, VHF Radio, RADAR, peralatan Oil Spill Response) beserta status operasionalnya, sehingga kondisi setiap peralatan selalu terpantau dan jadwal maintenance dapat direncanakan.

**Acceptance Criteria:**

- AC-01: Admin dapat melakukan CRUD untuk data peralatan: nama, tipe, serial number, tanggal pembelian, garansi, dan lokasi pemasangan
- AC-02: Status peralatan dapat diperbarui: Operational, Maintenance, Standby, atau Out of Service
- AC-03: Jadwal maintenance dapat diinput dan sistem mengirimkan reminder 7 hari sebelum jadwal maintenance
- AC-04: History status dan maintenance setiap peralatan tersimpan secara lengkap
- AC-05: Alert dikirim ke Admin jika ada peralatan kritis (AIS, RADAR, VHF) yang berstatus non-Operational
- AC-06: Laporan inventarisasi peralatan dapat digenerate dan diekspor

---

## US-MD-008: External System Configuration

**Sub-module:** Integration API Configuration
**Epic:** External System Configuration
**Related FR:** FR-MD-06, FR-GI-06

> Sebagai SuperAdmin, saya ingin mengelola konfigurasi koneksi API ke seluruh sistem eksternal (AIS provider, Weather API, Inaportnet, SIMoPEL, Bea Cukai, Odoo) dari satu tempat, sehingga pengelolaan integrasi menjadi terpusat dan perubahan konfigurasi dapat dilakukan dengan aman.

**Acceptance Criteria:**

- AC-01: Dashboard konfigurasi API menampilkan daftar semua integrasi eksternal beserta status koneksi real-time
- AC-02: SuperAdmin dapat mengupdate endpoint URL, API key, dan parameter koneksi setiap integrasi
- AC-03: Fungsi test connection tersedia untuk memverifikasi konfigurasi sebelum disimpan
- AC-04: Nilai API key dan credential disimpan secara terenkripsi dan tidak ditampilkan dalam bentuk plain text
- AC-05: Perubahan konfigurasi API dicatat dalam audit trail dengan timestamp
- AC-06: SuperAdmin dapat mengaktifkan/menonaktifkan integrasi tertentu tanpa menghapus konfigurasi
