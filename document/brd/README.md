# BRD Per Modul — Index

Folder ini berisi pecahan BRD utama (`document/BRD-LPS-V3-Analysis.md`) menjadi **satu file per modul**.  
Tujuannya adalah kemudahan membaca — setiap file hanya berisi konten yang relevan untuk modul tersebut.

## Aturan Penting

- **Sumber kebenaran tetap di BRD utama:** `document/BRD-LPS-V3-Analysis.md`
- **File di folder ini adalah turunan (derived).** Jika ada perubahan pada BRD utama, file terkait di sini **harus ikut diupdate dalam sesi yang sama**.
- Jangan mengedit file di folder ini secara langsung tanpa mengedit BRD utama terlebih dahulu.

## Daftar File Per Modul

| File | Modul | Kategori |
|------|-------|----------|
| [m1-vessel-monitoring.md](m1-vessel-monitoring.md) | M1 — Vessel Monitoring & AIS Integration | Core Monitoring |
| [m2-vessel-communication.md](m2-vessel-communication.md) | M2 — Vessel Communication Management | Core Monitoring |
| [m3-government-integration.md](m3-government-integration.md) | M3 — Government Integration | Integrasi Pemerintah |
| [m4-incident-emergency.md](m4-incident-emergency.md) | M4 — Incident & Emergency Management | Keselamatan & Darurat |
| [m5-weather-monitoring.md](m5-weather-monitoring.md) | M5 — Weather Monitoring & Alert System | Keselamatan & Darurat |
| [m6-radar-navigation.md](m6-radar-navigation.md) | M6 — Radar & Navigation Surveillance | Navigasi |
| [m7-customer-authentication.md](m7-customer-authentication.md) | M7 — Customer Authentication & Onboarding | Customer Portal |
| [m8-nomination-submission.md](m8-nomination-submission.md) | M8 — Nomination Request Submission | Customer Portal |
| [m9-nomination-status.md](m9-nomination-status.md) | M9 — Nomination Status & EPB Confirmation | Customer Portal |
| [m9b-epb-invoice.md](m9b-epb-invoice.md) | M9b — EPB (Customer Portal) | Customer Portal |
| [m9c-invoice.md](m9c-invoice.md) | M9c — Invoice (Customer Portal) | Customer Portal |
| [m10-customer-dashboard.md](m10-customer-dashboard.md) | M10 — Customer Dashboard & Monitoring | Customer Portal |
| [m11-monitoring-dashboard.md](m11-monitoring-dashboard.md) | M11 — Monitoring & Visibility Dashboard | Monitoring |
| [m12-system-configuration.md](m12-system-configuration.md) | M12 — System Configuration | Konfigurasi |

## Struktur Setiap File

Setiap file modul berisi:
1. **Header** — nomor modul, nama, kategori, versi BRD, tanggal update
2. **Definisi & Tujuan** — dari Section 2.1 BRD utama
3. **Cakupan (In Scope)** — dari Section 2.1 BRD utama
4. **Functional Requirements** — dari Section 3.4 BRD utama
5. **Role & Access** — ringkasan akses per role dari Section 3.3
6. **Referensi** — link ke BRD utama dan modul terkait lainnya
