# BRD Modul M1 — Vessel Monitoring & AIS Integration

**Kategori:** Core Monitoring  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.1](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul inti yang mengelola pemantauan posisi dan status seluruh kapal yang beroperasi di area perairan STS Bunati secara real-time melalui integrasi dengan AIS Base Station.

## Tujuan

- Menyediakan visibilitas real-time posisi kapal di area konsesi perairan.
- Mengintegrasikan data AIS dengan peta digital area STS Bunati.
- Mendeteksi keberadaan kapal yang tidak terdaftar di area perairan.
- Menjadi dasar data untuk seluruh modul operasional LPS.

## Cakupan (In Scope)

- AIS Base Station Integration (2 arah: pencarian posisi kapal & komunikasi).
- Real-time vessel tracking dengan VHF Antenna & PPS Antenna.
- Peta digital area STS Bunati dengan overlay posisi kapal.
- Vessel status dashboard: kecepatan, arah, ETA, status operasional.
- Alert otomatis jika kapal memasuki/meninggalkan zona tertentu.
- History pergerakan kapal dengan timestamp.

---

## Functional Requirements

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

---

## Role & Access

| Role | Akses |
|------|-------|
| LPS Operator | Full |
| VHF Operator | View |
| Security Operator | View |
| Admin Kepelabuhan | Full |
| KSOP | View |
| Bea Cukai | — |
| Customer | — |
| SuperAdmin | View |

---

## Referensi

- BRD Utama: [Section 2.1 M1](../BRD-LPS-V3-Analysis.md) & [Section 3.4.1](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/`](../../module/) *(belum ada subfolder M1)*
- Modul terkait: [M6 — Radar & Navigation](m6-radar-navigation.md), [M2 — Vessel Communication](m2-vessel-communication.md)
