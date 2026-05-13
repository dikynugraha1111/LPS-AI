# M7 — Customer Authentication & Onboarding · UI Design

**Last updated:** 2026-05-13 (v1.1) · **Surface:** A (customer side) + B (admin side) · **Status:** ACTIVE

> Wajib dibaca bersama `lps-design-system.md` (foundation tokens, komponen, surface preset).

---

## 1. Ringkasan

M7 punya dua surface karena melayani dua pengguna berbeda:

- **Surface A — Customer Portal**: registrasi mandiri, login customer.
- **Surface B — Internal Operator**: Customer Management (list, detail, approve/reject), Add Customer manual oleh Admin.

Module-level boundary lihat `module/customer-authentication/README.md`. Functional requirement source: FR-CA-01..FR-CA-10.

---

## 2. Page Inventory

### Surface A — Customer

| Route | Halaman | Akses |
|---|---|---|
| `/customer/login` | Login customer | Public |
| `/customer/register` | Registrasi mandiri (2 section: Company Info + Required Documents) | Public |
| `/customer/register/success` | Konfirmasi submit registrasi | Public (post-submit) |

### Surface B — Admin

| Route | Halaman | Akses |
|---|---|---|
| `/admin/login` | Login admin | Public |
| `/admin/customers` | Customer Management — list semua customer dengan filter status & tab | Admin, SuperAdmin |
| `/admin/customers/:id` | Customer Detail — view registrasi + dokumen + tombol Approve/Reject | Admin, SuperAdmin |
| `/admin/customers/new` | Add Customer manual oleh Admin (dengan custom documents) | Admin, SuperAdmin |

---

## 3. Surface A — Customer Pages

### 3.1 Customer Login (`/customer/login`)

**Layout:** **Auth Split-Screen** (lihat design system §3.4). Form panel kiri 50%, image showcase carousel kanan 50%. Pada `< lg` (1024px) image panel hidden, form full-width.

```
┌────────────────────────────────────┬───────────────────────────────────┐
│ [⚓ LPS System]                    │                                   │
│                                    │     [Image showcase full-bleed]   │
│                                    │                                   │
│      Selamat Datang                │                                   │
│      Masuk ke Portal Pelanggan LPS │     Slide image kapal/operasi     │
│                                    │                                   │
│      Email                         │                                   │
│      [✉ email@perusahaan.co.id]    │                                   │
│                                    │                                   │
│      Password                      │                                   │
│      [🔒 ••••••••••       👁]      │                                   │
│                                    │                                   │
│      [       Masuk       ]         │     Keselamatan & Keandalan       │
│                                    │     Standar operasional terbaik   │
│      ───────── atau ─────────      │     untuk setiap pelayaran.       │
│                                    │                                   │
│      Belum punya akun?             │                          ─ • •    │
│      Daftar di sini                │                                   │
│                                    │                                   │
│  © 2026 LPS System. Seluruh        │                                   │
│  hak dilindungi.                   │                                   │
└────────────────────────────────────┴───────────────────────────────────┘
                50%                                  50%
```

**Komponen utama:**

| Element | Pattern dari design system |
|---|---|
| Layout shell | `Auth Split-Screen layout` (§3.4) |
| Brand mark top-left | Wordmark inline: anchor icon `h-5 w-5 text-[#0F2A4D]` + "LPS System" `text-base font-semibold` (§2.10) |
| Heading "Selamat Datang" | `text-4xl font-semibold text-slate-900` |
| Subtitle | `text-sm text-slate-500 mt-2` — copy: "Masuk ke Portal Pelanggan LPS" |
| Email field | Input with leading icon (§3.1) — icon `Mail` |
| Password field | Input with leading icon + eye toggle — icon `Lock` |
| Button "Masuk" | Primary, full-width, py-3 |
| Divider "atau" | Divider with text (§3.1) |
| Link "Daftar di sini" | Inline link: "Belum punya akun? **Daftar di sini**" — "Daftar di sini" bold navy hover underline |
| Footer | `text-xs text-slate-400 text-center` — "© 2026 LPS System. Seluruh hak dilindungi." |
| Image showcase | Image Showcase Carousel (§3.4) dengan default 3 slides Surface A (Keselamatan & Keandalan / Integrasi STS Real-time / Pantauan 24/7) |

**Copy reference:**
- Heading: **"Selamat Datang"** (Title Case, 2 kata).
- Subtitle: "Masuk ke Portal Pelanggan LPS"
- Field placeholder email: "email@perusahaan.co.id"
- Field placeholder password: (kosong — pakai dots indicator native)
- Button: "Masuk"
- Divider text: "atau" (lowercase)
- Link prompt: "Belum punya akun? **Daftar di sini**"
- Footer: "© 2026 LPS System. Seluruh hak dilindungi."
- Error generic (toast atau inline card di atas form): "Email atau password salah. Silakan coba lagi."

**Error state:**
- Inline error card di atas form fields: `rounded-lg bg-rose-50 border border-rose-200 p-3` dengan icon `AlertCircle` kiri + text `text-sm text-rose-700`. Dismissable.
- Field-level error (e.g., format email): helper text `text-xs text-rose-600 mt-1` di bawah field + border field jadi `border-rose-300`.

**State akun:**
- Login sukses → redirect ke `/customer/dashboard`.
- Akun belum approved → toast warning "Akun Anda masih menunggu persetujuan admin." Tidak redirect.
- Akun rejected → toast error "Pendaftaran Anda ditolak. Silakan hubungi admin." Tidak redirect.
- Akun disabled → toast error "Akun dinonaktifkan. Hubungi admin." Tidak redirect.

**Carousel behavior:**
- 3 slides default (lihat design system §3.4 tabel default slides).
- Auto-advance 6 detik, pause saat hover.
- Dots indicator bottom-right clickable untuk navigasi manual.
- Image disimpan di `public/images/auth-showcase/` (slide-1.jpg, slide-2.jpg, slide-3.jpg) — placeholder di-replace dengan asset final saat marketing approve.
- Config slides di `src/config/authSlides.ts` (ID dan copy verbatim dari design system §3.4 default slides).

### 3.2 Customer Register (`/customer/register`)

**Layout:** **Auth Split-Screen** (sama dengan login, konsisten visual). Form panel kiri 50%, image showcase carousel kanan 50%. Pada `< lg` image panel hidden, form full-width.

**Catatan layout register:**
- Form register lebih panjang dari login (2 section: Company Info + Required Documents dengan 3 file upload). Form panel **scrollable** secara vertikal — brand mark tetap fixed top-left, footer tetap fixed bottom; section form di antara keduanya scroll.
- Image showcase kanan tetap fixed (tidak scroll) — full-height.
- Form max-width dinaikkan ke `max-w-xl` (~576px) untuk register, dari `max-w-md` (~448px) di login, agar field tidak terlalu sempit.

**Brand mark, heading, footer:** Sama pattern dengan login. Heading register: "Daftar Akun Pelanggan" (`text-4xl font-semibold`). Subtitle: "Lengkapi data perusahaan dan unggah dokumen wajib."

**Image showcase carousel:** Pakai 3 slides yang sama dengan login (lihat design system §3.4 default slides). Konsistensi brand.

**Existing form structure tetap berlaku:**

**Struktur:**

```
┌──────────────────────────────────────────────────────────────┐
│  ← Kembali ke Login (link kecil di bawah brand mark)         │
│                                                              │
│  Daftar Akun Pelanggan                                       │
│  Lengkapi data perusahaan dan unggah dokumen wajib.          │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 1. Informasi Perusahaan                                │ │
│  │ ─────────────────────────────────────────────────────  │ │
│  │ Nama Perusahaan *      [____________________]          │ │
│  │ Email *                [____________________]          │ │
│  │ Nomor Telepon *        [____________________]          │ │
│  │ Nama PIC *             [____________________]          │ │
│  │ Alamat *               [____________________]          │ │
│  │                        [____________________]          │ │
│  │ Catatan (opsional)     [____________________]          │ │
│  │ Password *             [______________] [👁]           │ │
│  │ Konfirmasi Password *  [______________] [👁]           │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ 2. Dokumen Wajib                                       │ │
│  │ ─────────────────────────────────────────────────────  │ │
│  │ Unggah NPWP, NIP, dan Company Profile.                 │ │
│  │                                                        │ │
│  │ ┌──────────────────────────────────────────────────┐  │ │
│  │ │ NPWP *                                           │  │ │
│  │ │ Deskripsi  [_______________________________]    │  │ │
│  │ │ Tgl Terbit [DD/MM/YYYY]                          │  │ │
│  │ │ Tgl Expired [DD/MM/YYYY]                         │  │ │
│  │ │ File       [Pilih file...]                       │  │ │
│  │ │            PDF/JPG/PNG · maks 10 MB              │  │ │
│  │ └──────────────────────────────────────────────────┘  │ │
│  │ (kartu sama untuk NIP & Company Profile)               │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                              │
│              [  Batal  ]   [  Daftar Sekarang  ]            │
└──────────────────────────────────────────────────────────────┘
```

**Komponen:**
- Each section pakai Section Card (lihat design system §3.2). Section title `text-lg font-semibold` + subtitle muted.
- Form field: pakai `<Label>` + `<Input>` shadcn. Required indicator: `*` text rose-500 di label.
- Document card: nested `rounded-xl border border-slate-200 p-5 bg-slate-50/40` mengandung 4 field (Deskripsi text, Issue Date date picker, Expiry Date date picker, File input dengan preview filename).
- File input pattern: button outlined "Pilih file..." → trigger native file picker → setelah pilih tampilkan filename + size + icon X untuk remove.
- Form action di bottom: button Batal (outlined ghost) + Daftar Sekarang (primary).
- Validation inline: helper text rose-600 di bawah field saat error.
- Field "Tipe Pelanggan" TIDAK ditampilkan (hardcode `Cargo Owner` di backend).
- Customer Code TIDAK ditampilkan (auto-generated saat approve oleh admin).

**State:**
- Loading saat submit: button primary disabled + spinner inline + label "Mendaftarkan...".
- Success: redirect ke `/customer/register/success`.
- Error: card error di atas form + scroll ke field invalid.

### 3.3 Register Success (`/customer/register/success`)

**Layout:** Centered, simple confirmation card.

**Konten:**
- Icon check circle 64px text-emerald-500
- Heading: "Pendaftaran Berhasil Dikirim"
- Body: "Tim kami akan meninjau data Anda dalam 1–2 hari kerja. Status akan dikirim ke email **{email}** setelah review selesai."
- Button outlined "Kembali ke Halaman Login" → `/customer/login`.

---

## 4. Surface B — Admin Pages

### 4.1 Admin Login (`/admin/login`)

**Layout:** **Auth Centered** (design system §3.4) — minimal, no showcase carousel. Beda dari customer login yang split-screen.

**Rationale:** Admin login adalah halaman fungsional internal, bukan touchpoint customer-facing. Marketing showcase tidak relevan. Pakai centered card pattern (Surface B tone: simple, operational).

**Struktur:**

```
┌──────────────────────────────────────────────┐
│                                              │
│                                              │
│            [⚓ LPS System]                    │
│                                              │
│              Sign In                         │
│       Operations console — Bunati Port       │
│                                              │
│   ┌────────────────────────────────────┐     │
│   │ Email                              │     │
│   │ [✉ admin@lps.id              ]    │     │
│   │                                    │     │
│   │ Password                           │     │
│   │ [🔒 ••••••••••           👁]      │     │
│   │                                    │     │
│   │ [        Sign In         ]         │     │
│   └────────────────────────────────────┘     │
│                                              │
│       © 2026 LPS System                      │
└──────────────────────────────────────────────┘
```

**Komponen:**
- Layout shell: Auth Centered layout (§3.4).
- Brand mark stacked: anchor `h-6 w-6` + "LPS System" `text-lg font-semibold` (size lebih besar dari customer login karena perlu visual anchor di tengah).
- Heading: "Sign In" `text-2xl font-semibold`.
- Subtitle: "Operations console — Bunati Port" muted.
- Card: Section Card pattern (`rounded-2xl border bg-white p-8 shadow-sm`).
- Input fields: pakai Input with leading icon (§3.1).
- Button "Sign In": Primary, full-width.
- **TIDAK ada** link "Daftar di sini" (admin tidak self-register).
- **TIDAK ada** divider "atau".

**Token storage:** HTTP-only cookie (bukan localStorage seperti customer). Set oleh backend response.

**Copy English (Surface B tone):**
- Heading: "Sign In"
- Subtitle: "Operations console — Bunati Port"
- Email label: "Email"
- Password label: "Password"
- Button: "Sign In"
- Error generic: "Invalid email or password. Please try again."
- Footer: "© 2026 LPS System"

### 4.2 Customer Management List (`/admin/customers`)

**Layout:** Surface B (sidebar nested + top bar). Sidebar item active: `CUSTOMERS`.

**Struktur:**

```
Top bar: [Search]                                  [datetime · waves · 🌙 · 🔔 · Andi]
─────────────────────────────────────────────────────────────────────────────────────
Page header:
  Customers
  Manage customer accounts, approvals, and onboarding.                  [+ Add Customer]

Tabs (underline):
  All (47)  |  Pending Review (5)  |  Active  |  Rejected

Filter bar:
  [Search by name/email/code...]   [Source: All ▾]   [Sort: Newest first ▾]

Table:
┌─────────────────────────────────────────────────────────────────────────────┐
│ Customer Code  Company Name        PIC          Source   Status   Created  │
├─────────────────────────────────────────────────────────────────────────────┤
│ —              PT Samudra Biru     Budi S.      SELF     Pending  09 May   │ → View
│ CUST-202604-...PT Tata Bumi K.     Andi W.      SELF     Active   03 May   │ → View
│ CUST-202604-...PT Khatulistiwa     Sari L.      ADMIN    Active   02 May   │ → View
│ —              PT Borneo Jaya      Hari P.      SELF     Rejected 28 Apr   │ → View
└─────────────────────────────────────────────────────────────────────────────┘
                                                  Showing 1–10 of 47  ← →
```

**Komponen:**
- Page header dengan tombol primary "+ Add Customer" → `/admin/customers/new`.
- Tabs underline style (lihat design system §3.2 Tabs). Tab counter italic muted.
- Filter bar: search input rounded-full kiri, select compact (shadcn `<Select>`) kanan. Stacked dalam card `rounded-xl border bg-white p-4`.
- Status mapping:
  - Pending Review → Info variant.
  - Active → Success variant.
  - Rejected → Error variant.
- Customer Code "—" italic muted untuk customer yang belum di-approve (belum punya code).
- Source badge: "SELF" neutral, "ADMIN" info variant (small uppercase).
- Action kolom: link-action "View" + arrow.

### 4.3 Customer Detail (`/admin/customers/:id`)

**Layout:** Surface B. Breadcrumb `Customers > {Company Name}` di atas page header.

**Struktur 2-column:**

```
Left column (2/3 width):
  [Section Card] Company Information
    Company Name, Email, Phone, PIC Name, Address, Note
    Customer Code (jika sudah approved) atau "Belum diterbitkan" italic
    Source: SELF / ADMIN badge
    Registered at: timestamp
  
  [Section Card] Documents
    List 3 dokumen wajib (NPWP, NIP, Company Profile):
      Per dokumen: nama, deskripsi, issue date, expiry date, button "View" (open file in new tab)
    Custom documents (jika ada, dari ADMIN add) — di section terpisah dengan label "Custom Documents (5)"

Right column (1/3 width):
  [Section Card] Status
    Big status badge
    Timeline:
      - Registered: 09 May 2026, 10:24 (otomatis terisi)
      - Approved: — (atau timestamp + admin name)
      - Activated: —
    
  [Section Card] Actions (hanya tampil saat status = Pending Review)
    Button primary "Approve" full-width
    Button destructive "Reject" full-width
    Reject membuka modal: textarea alasan + button Confirm.
```

**Komponen:**
- Section Card pattern reused.
- Document item: nested card dengan icon FileText kiri + metadata + button outlined "View" kanan.
- Timeline: list vertical dengan dot bullet + connector line (`border-l-2 border-slate-200 pl-4 space-y-3`).
- Reject modal: shadcn `<Dialog>` dengan title "Reject Customer Registration", textarea required min 20 char, button destructive "Send Rejection".
- Approve & Reject keduanya trigger email ke customer (catatan visible di modal "Customer will receive email notification.").

**State transitions:**
- Approve → status badge animate ke Success, action card disappear, customer code muncul.
- Reject → status badge animate ke Error, action card disappear, alasan reject visible di timeline.

### 4.4 Add Customer (`/admin/customers/new`)

**Layout:** Surface B. Similar dengan customer register tapi:
- English copy.
- Tambah section ke-3: "Custom Documents (Optional)" dengan tombol `+ Add Document` (dynamic field array via `useFieldArray`).
- Per custom document: label (text input bebas), file upload, optional issue/expiry date.
- Tidak ada password field — admin tidak set password. Sistem generate random + kirim welcome email berisi setup password link.
- Customer Code auto-generated saat save (tampilkan preview "Will be generated upon save").
- Status default ACTIVE (no approval step).

---

## 5. Component Usage Summary

| Component (dari design system) | Dipakai di |
|---|---|
| **Auth Split-Screen layout** (§3.4) | Customer login, Customer register |
| **Auth Centered layout** (§3.4) | Admin login, Register Success |
| **Image Showcase Carousel** (§3.4) | Customer login, Customer register (3 default slides) |
| **Input with leading icon** (§3.1) | Email & password fields di semua auth pages |
| **Divider with text** (§3.1) | Customer login (di atas link "Daftar di sini") |
| **Brand mark wordmark** (§2.10) | Top-left auth pages |
| Section Card | Form sections di register, Customer detail sections, Admin login card |
| Status Badge | Customer list, Customer detail status, Source badge |
| Tabs underline | Customer Management list filter |
| File upload dropzone | Register (3 docs), Add Customer (3 docs + custom docs) |
| Document list item | Customer Detail documents section |
| Empty state | List dengan filter tidak match |
| Dialog (shadcn) | Reject modal |
| Top bar B | Semua admin pages (kecuali admin login) |
| Sidebar B | Semua admin pages (kecuali admin login) |

---

## 6. State Transitions & Edge Cases

| Trigger | UI behavior |
|---|---|
| Registration submit (success) | Redirect ke `/customer/register/success`. Email konfirmasi dikirim. |
| Registration submit (email duplikat) | Inline error di field email "Email sudah terdaftar." |
| File upload > 10 MB | Toast error "File terlalu besar (maks 10 MB)." File tidak ter-attach. |
| File MIME tidak match | Toast error "Format file tidak didukung. Gunakan PDF, JPG, atau PNG." |
| Admin approve | Optimistic update status badge, toast "Customer approved. Email confirmation sent." |
| Admin reject (alasan < 20 char) | Dialog tetap terbuka, textarea show inline error. |
| Customer login (account belum approved) | Toast warning "Akun Anda masih menunggu persetujuan admin." Tidak redirect. |
| Customer login (account rejected) | Toast error "Pendaftaran Anda ditolak. Silakan hubungi admin." |

---

## 7. Cross-references

- Foundation: `implementation/design/lps-design-system.md`
- Module scope: `module/customer-authentication/`
- Replit handoff: `implementation/replit-handoff/m7-customer-authentication.md` (v1.1) + `m7-customer-authentication-v1.1-delta.md`
- BRD: `document/brd/m7-customer-authentication.md`
