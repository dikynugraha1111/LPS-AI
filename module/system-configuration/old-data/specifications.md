# M9 — Specifications

Spesifikasi fungsional detail untuk modul Master Data & Configuration, diambil dari FSD section 4.1.

## 4.1 Master Data & Configuration

Modul pondasi yang menyediakan data referensi untuk seluruh sistem LPS. Semua data master harus diisi sebelum modul operasional dapat digunakan.

### 4.1.1 Vessel Master

CRUD data kapal yang beroperasi di area perairan STS Bunati, digunakan sebagai referensi di semua modul.

**Field Specification:**

| Field | Tipe | Wajib | Validasi | Keterangan |
|---|---|---|---|---|
| Nama Kapal | Text | Ya | Min 3, Max 100 karakter | Nama resmi kapal |
| IMO Number | Text | Ya | 7 digit angka, unik di sistem | Nomor identifikasi IMO |
| MMSI | Text | Ya | 9 digit angka, unik di sistem | Maritime Mobile Service Identity |
| Call Sign | Text | Ya | Max 10 karakter, alfanumerik | Tanda panggil radio |
| Tipe Kapal | Dropdown | Ya | MV / Barge / Tanker / Lainnya | Jenis kapal |
| Bendera | Dropdown | Ya | Dari lookup negara | Negara asal kapal |
| GRT | Number | Tidak | Angka positif | Gross Register Tonnage (dasar tarif) |
| LOA (meter) | Number | Tidak | Angka positif | Length Overall |
| Pemilik/Operator | Lookup | Tidak | Dari Stakeholder Master | Relasi ke stakeholder |
| Status | Toggle | Ya | Active / Inactive | Inactive = tidak muncul di dropdown operasional |

**Business Rules:**

| Rule ID | Business Rule |
|---|---|
| BR-MD-01 | IMO Number harus unik di seluruh sistem. Duplikat ditolak sebelum save. |
| BR-MD-02 | MMSI harus 9 digit angka. Sistem otomatis memvalidasi format. |
| BR-MD-03 | Kapal Inactive tidak muncul di dropdown modul operasional lainnya. |
| BR-MD-04 | Jika MMSI kapal match dengan data AIS, sistem otomatis menandai kapal sebagai 'Terdeteksi AIS'. |
| BR-MD-05 | Import bulk via CSV/Excel; baris dengan data tidak valid diskip dan dilaporkan di summary. |

### 4.1.2 Stakeholder Master

Data perusahaan dan pihak yang terlibat dalam operasional (pemilik kapal, agen, perusahaan bongkar muat).

**Stakeholder Types:**

- KSOP
- Bea Cukai
- BUP
- Pemilik Kapal
- Badan Usaha Pemanduan

**Business Rules:**

| Rule ID | Business Rule |
|---|---|
| BR-MD-06 | Nama perusahaan dan NPWP harus unik di sistem. |
| BR-MD-07 | Satu stakeholder dapat memiliki lebih dari satu kapal (relasi 1-to-many ke Vessel Master). |
| BR-MD-08 | Email PIC digunakan untuk pengiriman invoice dan notifikasi sistem. |

### 4.1.3 Zone & Anchor Point Configuration

Konfigurasi zona perairan dan titik jangkar sebagai referensi monitoring AIS, RADAR, dan alert intrusi.

**Zone Types:**

- Restricted Area
- Warning Area
- Normal Operations Area
- Concession Area
- Buffer Zone
- Entry Lane

**Business Rules:**

| Rule ID | Business Rule |
|---|---|
| BR-MD-09 | Sistem mendukung minimum 5 anchor point yang dikonfigurasi dengan koordinat latitude/longitude. |
| BR-MD-10 | Setiap zone didefinisikan dengan polygon minimum 3 titik koordinat. |
| BR-MD-11 | Tipe zone: Restricted Area, Warning Area, atau Normal Operations Area. |
| BR-MD-12 | Perubahan konfigurasi zone diperbarui di seluruh modul dalam waktu maksimal 5 menit. |

**Anchor Point Configuration:**

- Setiap anchor point memiliki nama, koordinat (latitude/longitude), dan status aktif
- Minimum 5 anchor point sesuai operasional STS Bunati
- Status: Active / Inactive

### 4.1.4 Rate Card Management

Tarif layanan kepelabuhan sebagai dasar kalkulasi billing. Riwayat rate card tersimpan permanen untuk audit.

**Service Types:**

| Tipe Layanan | Satuan Kalkulasi |
|---|---|
| Labuh | Per GT |
| Tambat | Per GT/jam |
| Pandu | Per call |
| Tunda | Per call |
| Layanan tambahan | Sesuai konfigurasi |

**Business Rules:**

| Rule ID | Business Rule |
|---|---|
| BR-MD-13 | Rate card memiliki effective date dan expiry date. Kalkulasi menggunakan rate yang aktif pada saat layanan diberikan. |
| BR-MD-14 | Rate card tidak dapat dihapus, hanya dapat di-expire. Riwayat tersimpan untuk audit. |
| BR-MD-15 | Rate baru otomatis meng-expire rate sebelumnya untuk tipe layanan yang sama. |
| BR-MD-16 | Rate card approval workflow: status pending -> active (setelah approval). |

### 4.1.5 User & Role Management

Pengelolaan user account dan role-based access control (RBAC) dengan 9 role sistem.

**System Roles:**

| Role | Platform | Tipe Akses | Deskripsi |
|---|---|---|---|
| LPS Operator | Web | Internal TBK | Monitoring peta AIS, komunikasi kapal, laporan insiden, weather alert, 24/7 |
| VHF Operator | Web | Internal TBK | Channel management, log komunikasi VHF, broadcast notifikasi |
| Security Operator | Web | Internal TBK | Deteksi kapal asing, checklist darurat, oil spill management |
| Admin Kepelabuhan | Web | Internal TBK | Semua modul operasional, approval, monitoring integrasi pemerintah |
| Finance | Web | Internal TBK | Kalkulasi biaya, approval invoice, payment reconciliation, rate card |
| KSOP | Web Portal | Eksternal (Regulator) | View-only: monitoring kapal, laporan, insiden, status PKK |
| Bea Cukai | Web Portal | Eksternal (Regulator) | View-only: manifes muatan, dokumen kepabeanan |
| Pemilik Kapal | Web Portal | Eksternal (Customer) | View kapal milik, invoice, upload bukti bayar, download BPJK |
| SuperAdmin | Web | Internal TBK | CRUD user, role, konfigurasi sistem, API config, audit trail penuh |

**Business Rules:**

| Rule ID | Business Rule |
|---|---|
| BR-MD-17 | Username dan email harus unik di seluruh sistem. |
| BR-MD-18 | Password: minimal 8 karakter, kombinasi huruf besar, huruf kecil, dan angka. |
| BR-MD-19 | User yang di-nonaktifkan tidak dapat login; session aktif langsung diinvalidasi. |
| BR-MD-20 | Satu user hanya memiliki satu role aktif. Perubahan role efektif pada login berikutnya. |
| BR-MD-21 | SuperAdmin tidak dapat menonaktifkan akunnya sendiri. |
| BR-MD-22 | Semua perubahan user tersimpan di audit trail beserta aktor dan timestamp. |

### 4.1.6 Equipment Master & API Configuration

Data perangkat keras LPS (AIS, VHF, RADAR) untuk health monitoring, dan konfigurasi koneksi API eksternal.

**Equipment Types:**

- AIS Base Station
- VHF Coastal Radio
- RADAR System
- Oil Spill Response Equipment

**Equipment Status:**

| Status | Deskripsi |
|---|---|
| Operational | Perangkat berfungsi normal |
| Maintenance | Perangkat sedang dalam perawatan |
| Standby | Perangkat siap digunakan, tidak aktif |
| Out of Service | Perangkat tidak dapat digunakan |

**Business Rules:**

| Rule ID | Business Rule |
|---|---|
| BR-MD-23 | Setiap perangkat memiliki status: Operational / Maintenance / Offline. |
| BR-MD-24 | API Key dan Secret tersimpan terenkripsi. Tidak ditampilkan di UI setelah disimpan (masked). |
| BR-MD-25 | Tombol 'Test Connection' tersedia untuk verifikasi koneksi API sebelum disimpan. |

**API Integrations:**

| Integration Name | Deskripsi |
|---|---|
| AIS Provider | Posisi kapal real-time |
| Weather API | Data cuaca area Bunati |
| Inaportnet | Pelaporan ATA/ATD ke Kemenhub |
| SIMoPEL | Data voyage ke Kemenhub |
| Bea Cukai | Manifes muatan dan kepabeanan |
| Odoo (ERP) | Invoice dan jurnal akuntansi |

## System Rules (Cross-cutting)

Aturan sistem yang berlaku di seluruh modul, dikonfigurasi melalui M9:

| Rule | Deskripsi |
|---|---|
| No Hard Delete | Tidak ada hard delete pada sistem (soft delete only). |
| Session Timeout | Session timeout setelah 30 menit tidak ada aktivitas. |
| Audit Log | Semua override dan perubahan data kritis wajib tercatat dalam immutable audit log. |
| Data Retention | Data retention minimum 5 tahun untuk semua data operasional sesuai regulasi Kemenhub. |
| Invoice Immutability | Invoice yang sudah dikirim tidak dapat diubah; koreksi melalui dokumen amandemen terpisah. |
| Credential Security | API Key dan credential eksternal tersimpan terenkripsi, tidak pernah ditampilkan di UI. |
