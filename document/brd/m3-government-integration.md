# BRD Modul M3 — Government Integration

**Kategori:** Integrasi Pemerintah  
**Versi BRD:** 3.2  
**Tanggal Update:** Mei 2026  
**Sumber:** [BRD Utama — Section 2.1 & 3.4.3](../BRD-LPS-V3-Analysis.md)

---

## Definisi

Modul yang mengelola interkoneksi sistem LPS dengan platform pemerintah: Inaportnet dan SIMoPEL Kemenhub serta sistem Bea Cukai, untuk memastikan kepatuhan regulasi dan kelancaran administrasi kepelabuhan.

## Tujuan

- Memastikan seluruh data operasional kapal terlaporkan ke sistem Kemenhub secara otomatis.
- Memfasilitasi proses bea cukai untuk muatan yang dipindahkan dalam kegiatan STS.
- Mengurangi proses manual pelaporan ke instansi pemerintah.

## Cakupan (In Scope)

- Integrasi dengan Inaportnet Kemenhub: pelaporan kedatangan/keberangkatan kapal.
- Integrasi dengan SIMoPEL Kemenhub: data pelayaran dan kepelabuhan.
- Integrasi dengan sistem Bea Cukai: pelaporan muatan dan dokumen manifes.
- EDI-MARPOL compliance reporting.
- Generate dan transmit dokumen elektronik kepelabuhan secara otomatis.
- Monitoring status pengiriman data ke seluruh sistem pemerintah.

---

## Functional Requirements

| FR ID | Requirements | Actor |
|-------|-------------|-------|
| FR-GI-01 | Sistem harus mengirimkan data kedatangan dan keberangkatan kapal ke Inaportnet Kemenhub secara otomatis | System |
| FR-GI-02 | Sistem harus mengirimkan data operasional pelayaran ke SIMoPEL Kemenhub sesuai format yang ditetapkan | System |
| FR-GI-03 | Sistem harus menghasilkan dan mengirimkan dokumen manifes muatan ke sistem Bea Cukai secara elektronik | System |
| FR-GI-04 | Sistem harus menghasilkan laporan EDI-MARPOL sesuai standar regulasi yang berlaku | System |
| FR-GI-05 | Admin harus dapat memantau status pengiriman data ke seluruh sistem pemerintah beserta timestamp konfirmasi | Admin |
| FR-GI-06 | Sistem harus memiliki mekanisme retry otomatis jika pengiriman data ke sistem pemerintah gagal | System |
| FR-GI-07 | Sistem harus menyimpan bukti pengiriman (acknowledgement) dari setiap integrasi pemerintah | System |

---

## Role & Access

| Role | Akses |
|------|-------|
| LPS Operator | View |
| VHF Operator | — |
| Security Operator | — |
| Admin Kepelabuhan | Full |
| KSOP | View |
| Bea Cukai | View |
| Customer | — |
| SuperAdmin | View |

---

## Referensi

- BRD Utama: [Section 2.1 M3](../BRD-LPS-V3-Analysis.md) & [Section 3.4.3](../BRD-LPS-V3-Analysis.md)
- Module folder: [`module/`](../../module/) *(belum ada subfolder M3)*
- Sistem eksternal: Inaportnet, SIMoPEL Kemenhub, Bea Cukai
