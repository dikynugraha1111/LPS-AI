# M10 — Customer Dashboard & Monitoring: Functional Requirements

Derived from BRD Section 3.4.10 and Section 2.1 Module 10.

| FR ID | Requirement | Actor | Priority |
|-------|-------------|-------|----------|
| FR-CD-01 | Customer harus dapat memantau status voyage yang sedang berjalan (view-only) termasuk posisi kapal dari data AIS | Customer | Must Have |
| FR-CD-02 | Sistem harus menampilkan halaman Nomination Status dengan daftar seluruh nominasi customer beserta status terkini, termasuk nominasi berstatus Draft | Customer | Must Have |
| FR-CD-03 | Sistem harus menampilkan Customer Portal Dashboard setelah customer login, berisi: ringkasan nominasi aktif beserta status terkini, daftar voyage yang sedang berjalan (view-only, data dari STS Platform), dan widget cuaca real-time area STS Bunati | Customer | Must Have |
| FR-CD-04 | Customer harus dapat melihat data cuaca real-time dan history alert area perairan STS Bunati melalui Customer Portal (view-only) | Customer | Must Have |
| FR-CD-05 | Sistem harus menyediakan halaman Document Master sebagai repositori dokumen pribadi customer; customer hanya dapat mengupload dokumen baru — tidak dapat menghapus atau mengedit dokumen yang sudah diupload (append-only) | Customer | Must Have |
| FR-CD-06 | Dokumen di Document Master harus menampilkan metadata: nama file, tanggal upload, ukuran file, tipe file; customer dapat mengunduh atau preview dokumen (read-only) | Customer | Must Have |
| FR-CD-07 | Pada form nominasi baru, customer harus dapat memilih antara: (1) Upload dari komputer lokal — file otomatis tersimpan ke Document Master; (2) Pilih dari Document Master yang sudah ada | Customer | Must Have |
| FR-CD-08 | Document Master harus bersifat private per customer; dokumen satu customer tidak boleh dapat diakses oleh customer lain | System | Must Have |

## Cross-Cutting Requirements

| NFR | Source | Description |
|-----|--------|-------------|
| Real-time | BRD §3.5 #2 | Data posisi kapal diperbarui max delay 5 menit. |
| Access Control | BRD §3.3.2 | Customer hanya melihat data milik diri sendiri. |
| Availability | BRD §3.5 #1 | Dashboard harus tersedia 24/7 (menggunakan cache jika AIS/Weather API sedang down). |
