# LPS Platform — Design System

**Version:** 1.4 · **Last updated:** 2026-05-15 · **Status:** ACTIVE — single source of truth untuk semua UI work di LPS Platform.

> **Cara pakai:** Setiap kerja UI (baru, edit, refactor) **wajib** baca dokumen ini dulu, lalu invoke skill `ui-ux-pro-max` untuk validasi/generate komponen. Per-modul design doc (`m<N>-<name>-ui.md`) menjabarkan penerapan untuk modul masing-masing dan tetap merujuk ke dokumen ini.

---

## 1. Ringkasan

LPS Platform punya **dua surface UI**:

| Surface | Pengguna | Brand | Bahasa | Karakter |
|---|---|---|---|---|
| **A — Customer Portal** | Pemilik kapal / cargo owner (PIC) | "PORTAL PELANGGAN" (nama PT customer) | Bahasa Indonesia formal | Spacious, card-heavy, navigasi sederhana |
| **B — Internal Operator** | Operator VTS, Admin, SuperAdmin | "LPS SYSTEM — BUNATI PORT" | English | Data-dense, nested nav, top bar persistent, KPI + map + chart |

Style line keseluruhan: **Professional Maritime** — minimalism + flat design dengan palet navy gelap sebagai jangkar identitas, aksen sesuai status, dan surface terang untuk readability di siang hari.

---

## 2. Foundation (shared antara Surface A & B)

### 2.1 Color tokens

Semua nilai sudah disesuaikan dengan production. Token dipresentasikan sebagai Tailwind class + raw hex agar bisa dipakai di shadcn theme (`globals.css` / `tailwind.config`).

#### Brand / Navy (primary)

| Token | Hex | Tailwind | Penggunaan |
|---|---|---|---|
| `navy-950` | `#0B1F3A` | `bg-[#0B1F3A]` | Sidebar background, primary button hover |
| `navy-900` | `#0F2A4D` | `bg-[#0F2A4D]` | Sidebar active item, primary button |
| `navy-700` | `#1E3A66` | — | Hover state sidebar, button border emphasis |
| `navy-500` | `#3B5B89` | — | Icon di sidebar inactive |
| `navy-100` | `#DCE5F1` | — | Background informasi netral |

#### Neutral (canvas, text, border)

| Token | Hex | Tailwind | Penggunaan |
|---|---|---|---|
| `bg-canvas` | `#F5F7FA` | `bg-slate-50` | Main canvas (di kanan sidebar) |
| `bg-card` | `#FFFFFF` | `bg-white` | Card background |
| `border-subtle` | `#E5E9F0` | `border-slate-200` | Card border, divider |
| `border-input` | `#CBD5E1` | `border-slate-300` | Input border |
| `text-primary` | `#0F172A` | `text-slate-900` | Heading, value penting |
| `text-secondary` | `#475569` | `text-slate-600` | Body text |
| `text-muted` | `#94A3B8` | `text-slate-400` | Label, subtitle, placeholder |
| `text-on-navy` | `#FFFFFF` / `#CBD5E1` | `text-white` / `text-slate-300` | Text di atas sidebar navy |

#### Status palette (untuk badge — soft pill pattern)

Setiap status pakai tiga shade: `50` (background), `200` (border), `700` (text).

| Status semantik | Background | Border | Text | Token Tailwind |
|---|---|---|---|---|
| **Info / Review** | `#EFF6FF` | `#BFDBFE` | `#1D4ED8` | `bg-blue-50 border-blue-200 text-blue-700` |
| **Success / Approved** | `#ECFDF5` | `#A7F3D0` | `#047857` | `bg-emerald-50 border-emerald-200 text-emerald-700` |
| **Warning / Need Revision** | `#FFF7ED` | `#FED7AA` | `#C2410C` | `bg-orange-50 border-orange-200 text-orange-700` |
| **Pending verify** | `#F0FDFA` | `#99F6E4` | `#0F766E` | `bg-teal-50 border-teal-200 text-teal-700` |
| **Confirmed / Paid** | `#ECFDF5` | `#A7F3D0` | `#047857` | `bg-emerald-50 border-emerald-200 text-emerald-700` |
| **Neutral (Belum Dibayar, Draft)** | `#F1F5F9` | `#E2E8F0` | `#475569` | `bg-slate-100 border-slate-200 text-slate-600` |
| **Action Required (Perlu Tindakan)** | `#FEF2F2` | `#FECACA` | `#B91C1C` | `bg-rose-50 border-rose-200 text-rose-700` |
| **Overdue indicator** | `#FEE2E2` | `#FCA5A5` | `#991B1B` | `bg-rose-100 border-rose-300 text-rose-800` (dengan icon ⚠ `AlertTriangle`) |
| **Severity Warning (uppercase)** | `#FEFCE8` | `#FDE68A` | `#92400E` | `bg-yellow-50 border-yellow-300 text-yellow-800 uppercase` |
| **Error / Urgent** | `#FEF2F2` | `#FECACA` | `#B91C1C` | `bg-rose-50 border-rose-200 text-rose-700` |
| **Loading / In Progress** | `#EFF6FF` | `#BFDBFE` | `#1D4ED8` | `bg-blue-50 border-blue-200 text-blue-700` |

#### Status mapping LPS (per business status)

| Business status | Label ID (Surface A) | Label EN (Surface B) | Variant |
|---|---|---|---|
| `DRAFT` | Draft | Draft | Neutral (italic) |
| `SUBMITTED` / `PENDING` | Menunggu Review | Pending Review | Info / Review |
| `APPROVED` | Disetujui | Approved | Success |
| `NEED_REVISION` | Perlu Revisi | Need Revision | Warning |
| `UNPAID` (EPB/Invoice) | Belum Dibayar | Unpaid | Neutral |
| `WAITING_PAYMENT_VERIFICATION` | Menunggu Verifikasi | Waiting Verification | Pending verify |
| `PAYMENT_REJECT` | Pembayaran Ditolak | Payment Rejected | Error |
| `PAID` / `PAYMENT_CONFIRMED` | Lunas | Paid | Confirmed |
| **Kategori filter "Perlu Tindakan"** (UNPAID + PAYMENT_REJECT + Overdue) | Perlu Tindakan | Action Required | Action Required |
| **Overdue indicator** (due_date < now & status != Lunas) | Lewat jatuh tempo | Overdue | Overdue indicator |
| `WARNING` (cuaca) | WARNING | WARNING | Severity Warning |
| `ACTIVE` (vessel) | Aktif | ACTIVE | Success |
| `LOADING` (vessel) | Loading | LOADING | Info |
| `URGENT` (alert) | Mendesak | URGENT | Error |
| `ACTIVE` (master data / MV) | Active | Active | Confirmed |
| `INACTIVE` (master data / MV) | Inactive | Inactive | Neutral |
| `PENDING_APPROVAL` (approval request) | Pending Approval | Pending Approval | Pending verify |
| `REJECTED` (approval request) | Rejected | Rejected | Error |

### 2.2 Typography

- **Font family:** Inter (atau system sans-serif fallback). Loaded via `<link>` Google Fonts atau `@fontsource/inter`.
- **Weight scale:** 400 (regular), 500 (medium), 600 (semibold). Bold (700) hanya untuk emphasis spesifik (status uppercase, value besar).
- **Tracking:** default normal. `tracking-wider` + `uppercase` untuk label kategori (subtitle nav, KPI label).

| Role | Class | Size / Line-height | Weight |
|---|---|---|---|
| Page title (h1) | `text-3xl font-semibold` | 30 / 36 | 600 |
| Section title | `text-xl font-semibold` | 20 / 28 | 600 |
| Card title | `text-lg font-semibold` | 18 / 28 | 600 |
| Body | `text-sm text-slate-600` | 14 / 20 | 400 |
| Body emphasis | `text-sm font-medium text-slate-900` | 14 / 20 | 500 |
| Label muted | `text-xs uppercase tracking-wider text-slate-400` | 12 / 16 | 500 |
| Value besar (KPI) | `text-3xl font-bold` | 30 / 36 | 700 |
| Caption | `text-xs text-slate-500` | 12 / 16 | 400 |

### 2.3 Spacing & Layout

- **Spacing scale:** Tailwind default (1 = 4px). Pakai kelipatan 4: `space-2/4/6/8/10/12`.
- **Page padding:** `p-8` (32px) untuk main canvas; `p-10` atau `p-12` untuk halaman luas.
- **Card padding:** `p-8` (32px) standar; `p-6` untuk card compact (KPI tile).
- **Gap antar card stacked:** `space-y-8` (32px).
- **Sidebar width:** Surface A `w-64` (256px); Surface B `w-60` (240px), collapsed `w-16` (64px).
- **Top bar height (Surface B):** `h-16` (64px).
- **Max content width:** tidak ada constraint kaku — main canvas mengisi sisa width setelah sidebar. Untuk halaman form, batasi inner content `max-w-3xl` atau `max-w-4xl`.

### 2.4 Radius

- **Card besar:** `rounded-2xl` (16px) — section card, KPI card.
- **Card kecil / nested:** `rounded-xl` (12px) — voyage card, alert item.
- **Button, input, select:** `rounded-lg` (8px).
- **Badge:** `rounded-full`.
- **Avatar:** `rounded-full`.
- **Search field:** `rounded-full`.

### 2.5 Shadow & Elevation

Hanya tiga level. Hindari shadow berat.

- `shadow-none` — default sidebar, top bar (pakai border-bottom saja).
- `shadow-sm` — card (very subtle: `0 1px 2px rgba(15, 23, 42, 0.04)`).
- `shadow-md` — modal, dropdown, popover.

### 2.6 Border

Default tipis: `border border-slate-200`. Untuk KPI card dengan accent (warning/error state) tambahkan `border-l-4` dengan warna sesuai severity:

- Default: `border-slate-200`
- KPI accent warning: `border-l-4 border-l-yellow-400`
- KPI accent navy (primary metric): `border-l-4 border-l-[#0F2A4D]`
- KPI accent error: `border-l-4 border-l-rose-400`

### 2.7 Iconography

- **Library:** [Lucide React](https://lucide.dev) (`lucide-react`). Konsisten dengan shadcn/ui.
- **Size default:** `h-4 w-4` (16px) untuk inline, `h-5 w-5` (20px) untuk button/nav, `h-6 w-6` (24px) untuk page header icon, `h-8 w-8` (32px) untuk illustrasi kosong.
- **Stroke:** default 2px; jangan ganti.
- **Color:** `text-slate-400` untuk inactive/muted, `text-slate-600` untuk default, `text-white` di sidebar, status color untuk emphasis (e.g., `text-rose-600` untuk error icon).
- **Common icons:** `Ship` (vessel), `LayoutDashboard`, `FileText` (document), `Receipt` (invoice), `FolderOpen`, `CloudSun` (weather), `MapPin`, `Bell`, `Search`, `LogOut`, `Plus`, `ArrowRight`, `Download`, `Upload`, `Anchor`, `AlertTriangle`, `Eye` (visibility), `Wind`, `Waves`.

### 2.8 Motion (Framer Motion)

Preset durasi: **fast 150ms · normal 250ms · slow 400ms**. Easing default `easeOut`.

| Use | Preset |
|---|---|
| Page transition | `initial={{ opacity: 0, y: 8 }} animate={{ opacity: 1, y: 0 }} transition={{ duration: 0.25 }}` |
| Modal enter | `initial={{ opacity: 0, scale: 0.96 }} animate={{ opacity: 1, scale: 1 }} transition={{ duration: 0.15 }}` |
| Toast | slide-in dari kanan, duration 0.25, auto-dismiss 4s |
| Hover lift (card actionable) | `whileHover={{ y: -2 }}` |
| Sidebar collapse | width transition 0.25s |

**Tidak pakai:** efek heavy parallax, scroll-jacking, atau animasi auto-loop yang mengganggu.

### 2.9 Accessibility (mandatory)

- **Contrast:** minimal WCAG AA. Semua text diatas navy harus pakai `text-white` atau `text-slate-300` (sudah lulus AA).
- **Focus ring:** semua interactive element wajib visible focus. Pakai `focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2 focus-visible:ring-[#0F2A4D]`.
- **Keyboard nav:** tab order logis, escape close modal, enter submit form.
- **Hit target:** minimal 40×40px untuk button/icon button.
- **Label form:** wajib `<label htmlFor>` atau `aria-label`. Tidak boleh placeholder-only.
- **Status badge:** warna saja TIDAK cukup; selalu ada label text.

### 2.10 Brand mark

Brand mark LPS punya dua varian:

| Varian | Pakai di | Pattern |
|---|---|---|
| **Wordmark inline** | Header public/auth pages, footer | Anchor icon Lucide (`h-5 w-5 text-[#0F2A4D]`) + "LPS System" `text-base font-semibold text-slate-900`. Compact, casual. |
| **Stacked brand** | Sidebar (A & B) | Ship icon `h-6 w-6 text-white` + brand text 2-line: title bold + subtitle uppercase tracked muted. Lihat Sidebar A & B di §3.3. |

**Canonical naming:**
- Sistem keseluruhan: **"LPS System"** (bukan "LPS Platform" untuk display; "LPS Platform" hanya untuk dokumentasi internal/BRD).
- Customer Portal context: subtitle "Portal Pelanggan LPS" boleh muncul di brand stacked saat customer sudah login.
- Operator context: "LPS System — Bunati Port" stacked di sidebar.

### 2.11 Copywriting tone

- **Surface A (ID):** formal, sopan, dewasa. Contoh: "Selamat datang kembali", "Daftar seluruh nominasi yang pernah Anda ajukan", "Lihat Detail", "Buat Nominasi Baru", "Tidak ada riwayat peringatan cuaca."
- **Surface B (EN):** professional, concise, operational. Contoh: "Overview of port operations and live tracking", "Active Vessels", "Recent Activity", "View All".
- **Hindari:** singkatan tidak formal ("Yuk", "Sob"), emoji di copy, jargon teknis tanpa konteks.
- **Currency:** "Rp 1.250.000" (titik sebagai ribuan).
- **Tanggal:** Surface A `09 Mei 2026`; Surface B `Wed, May 13, 06:20 PM`.
- **Waktu:** Surface A WIB/WITA explicit ("03.04 WIB"); Surface B `06:20 PM`.

---

## 3. Component Library

Semua komponen pakai **shadcn/ui** sebagai basis. Custom variant didokumentasikan di sini.

### 3.1 Atoms

#### Button

| Variant | Tailwind class |
|---|---|
| Primary | `bg-[#0F2A4D] text-white hover:bg-[#0B1F3A] rounded-lg px-4 py-2.5 font-medium` |
| Outlined | `bg-white border border-slate-300 text-slate-900 hover:bg-slate-50 rounded-lg px-4 py-2.5 font-medium` |
| Ghost | `text-slate-600 hover:bg-slate-100 rounded-lg px-3 py-2` |
| Link-action (table row) | `text-slate-900 font-medium hover:underline inline-flex items-center gap-1` (+`<ArrowRight className="h-4 w-4">`) |
| Destructive | `bg-rose-600 text-white hover:bg-rose-700 rounded-lg px-4 py-2.5 font-medium` |
| Icon only | `inline-flex items-center justify-center h-9 w-9 rounded-lg hover:bg-slate-100` |

Primary button dengan icon: `<Plus className="h-4 w-4" />` di kiri text, gap `gap-2`.

#### Status Badge

```jsx
<span className="inline-flex items-center rounded-full border px-2.5 py-0.5 text-xs font-medium {bg} {border} {text}">
  {label}
</span>
```

Mapping `{bg} {border} {text}` ambil dari Status palette §2.1.

#### KPI Filter Pill (kategori billing/EPB)

Pattern untuk menampilkan ringkasan jumlah item per kategori filter di atas tabs. Dipakai di list page EPB & Invoice. **Clickable** untuk shortcut filter (klik pill → aktifkan tab terkait).

```jsx
<div className="flex flex-wrap gap-3 mb-6">
  <button className="rounded-xl border border-rose-200 bg-rose-50 px-4 py-3 text-left transition hover:border-rose-300">
    <div className="text-2xl font-bold text-rose-700">1</div>
    <div className="text-xs font-medium text-rose-700 mt-0.5">Perlu Tindakan</div>
  </button>
  <button className="rounded-xl border border-amber-200 bg-amber-50 px-4 py-3 text-left transition hover:border-amber-300">
    <div className="text-2xl font-bold text-amber-700">2</div>
    <div className="text-xs font-medium text-amber-700 mt-0.5">Menunggu Verifikasi</div>
  </button>
  <button className="rounded-xl border border-emerald-200 bg-emerald-50 px-4 py-3 text-left transition hover:border-emerald-300">
    <div className="text-2xl font-bold text-emerald-700">1</div>
    <div className="text-xs font-medium text-emerald-700 mt-0.5">Lunas</div>
  </button>
</div>
```

**Aturan:**
- Hanya 3 pill default: Perlu Tindakan (rose), Menunggu Verifikasi (amber), Lunas (emerald).
- Count = jumlah item dalam kategori. Tampil "0" jika kosong (jangan hide — biar customer paham kategori-nya ada).
- Hover: border-color naik 1 step. Active (kategori sedang dipilih sebagai filter): tambah `ring-2 ring-offset-1 ring-{color}-400`.
- Mobile: pill jadi stack 2-up (`grid grid-cols-2 gap-2`).

#### Overdue Date Display

Pattern untuk menampilkan tanggal jatuh tempo yang sudah lewat. Dipakai di list/detail EPB/Invoice.

```jsx
{/* Normal due date */}
<div className="inline-flex items-center gap-1.5 text-xs text-slate-500">
  <Calendar className="h-3.5 w-3.5" />
  <span>Jatuh tempo: 20 Mei 2026</span>
</div>

{/* Overdue */}
<div className="inline-flex items-center gap-1.5 text-xs font-medium text-rose-700">
  <Calendar className="h-3.5 w-3.5" />
  <span>Lewat jatuh tempo: 27 April 2026</span>
  <AlertTriangle className="h-3.5 w-3.5" />
</div>
```

**Aturan deteksi:**
- Overdue = `due_date < now()` AND `status != Lunas`.
- Color: text rose-700, icon AlertTriangle kanan untuk emphasis.
- Untuk Lunas yang overdue saat dibayar (paid_at > due_date): tampilkan "Lewat jatuh tempo: {date}" rose tapi **tanpa** AlertTriangle (sudah resolved, hanya informational).

#### Inline Info Banner (di dalam card list item)

Pattern banner kecil yang muncul di dalam card list untuk memberi konteks status. Dipakai di EPB/Invoice list item.

```jsx
{/* Action required */}
<div className="rounded-lg border border-rose-200 bg-rose-50/60 px-3 py-2 text-sm text-rose-700 flex items-start gap-2">
  <CreditCard className="h-4 w-4 mt-0.5 flex-shrink-0" />
  <span>Silakan lakukan pembayaran dan unggah bukti transfer untuk melanjutkan proses.</span>
</div>

{/* Waiting verification */}
<div className="rounded-lg border border-amber-200 bg-amber-50/60 px-3 py-2 text-sm text-amber-800 flex items-start gap-2">
  <Clock className="h-4 w-4 mt-0.5 flex-shrink-0" />
  <span>Bukti pembayaran sedang dalam proses verifikasi oleh tim kami. Harap tunggu konfirmasi.</span>
</div>

{/* Paid */}
<div className="rounded-lg border border-emerald-200 bg-emerald-50/60 px-3 py-2 text-sm text-emerald-700 flex items-start gap-2">
  <CheckCircle className="h-4 w-4 mt-0.5 flex-shrink-0" />
  <span>Pembayaran telah dikonfirmasi pada 29 April 2026.</span>
</div>

{/* Rejected */}
<div className="rounded-lg border border-rose-200 bg-rose-50/60 px-3 py-2 text-sm text-rose-700 flex items-start gap-2">
  <XCircle className="h-4 w-4 mt-0.5 flex-shrink-0" />
  <span>Bukti pembayaran ditolak. Alasan: {reason}. Silakan upload ulang.</span>
</div>
```

**Aturan:**
- Wajib pakai di setiap row list EPB/Invoice (selalu ada konteks per status).
- Icon kiri sesuai semantik: `CreditCard` untuk Belum Dibayar, `Clock` untuk Menunggu Verifikasi, `CheckCircle` untuk Lunas, `XCircle` untuk Pembayaran Ditolak.
- Background `/60` (lebih halus dari banner detail page yang `/40`) karena berada dalam list density.

#### Severity Badge (uppercase)

```jsx
<span className="inline-flex items-center rounded-md border px-2 py-1 text-xs font-bold uppercase tracking-wide bg-yellow-50 border-yellow-300 text-yellow-800">
  WARNING
</span>
```

#### Input / Select / Textarea

- Base: `w-full rounded-lg border border-slate-300 bg-white px-3 py-2.5 text-sm text-slate-900 placeholder:text-slate-400 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-[#0F2A4D]`.
- Error state: ganti border `border-rose-300` + helper text `text-rose-600 text-xs mt-1`.
- Disabled: `bg-slate-100 text-slate-500 cursor-not-allowed`.

#### Input with leading icon (auth & specialized forms)

Pattern field dengan ikon di kiri (dan optional eye toggle di kanan untuk password). Dipakai di halaman auth.

```jsx
{/* Email field */}
<div className="space-y-1.5">
  <label className="text-sm font-medium text-slate-700">Email</label>
  <div className="relative">
    <Mail className="absolute left-3.5 top-1/2 -translate-y-1/2 h-4 w-4 text-slate-400" />
    <input
      type="email"
      className="w-full rounded-lg border border-slate-200 bg-slate-50/60 py-3 pl-11 pr-3 text-sm text-slate-900 placeholder:text-slate-400 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-[#0F2A4D] focus-visible:bg-white"
      placeholder="email@perusahaan.co.id"
    />
  </div>
</div>

{/* Password field with eye toggle */}
<div className="space-y-1.5">
  <label className="text-sm font-medium text-slate-700">Password</label>
  <div className="relative">
    <Lock className="absolute left-3.5 top-1/2 -translate-y-1/2 h-4 w-4 text-slate-400" />
    <input
      type={showPassword ? "text" : "password"}
      className="w-full rounded-lg border border-slate-200 bg-slate-50/60 py-3 pl-11 pr-11 text-sm text-slate-900 placeholder:text-slate-400 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-[#0F2A4D] focus-visible:bg-white"
    />
    <button
      type="button"
      onClick={() => setShowPassword(!showPassword)}
      className="absolute right-3 top-1/2 -translate-y-1/2 text-slate-400 hover:text-slate-600"
      aria-label={showPassword ? "Hide password" : "Show password"}
    >
      {showPassword ? <EyeOff className="h-4 w-4" /> : <Eye className="h-4 w-4" />}
    </button>
  </div>
</div>
```

**Differences vs Input base:**
- Background `bg-slate-50/60` (subtle tinted) di idle state, putih saat focus.
- Padding kiri `pl-11` untuk accommodate icon 16px + spacing.
- Padding vertikal `py-3` (12px) sedikit lebih besar dari base untuk auth-friendly hit target.
- Icon container absolute kiri-tengah dengan `text-slate-400`.

#### Divider with text (auth pages)

```jsx
<div className="relative flex items-center py-2">
  <div className="flex-grow border-t border-slate-200" />
  <span className="px-3 text-xs text-slate-400">atau</span>
  <div className="flex-grow border-t border-slate-200" />
</div>
```

#### Search field (rounded-full, Surface B top bar)

```jsx
<div className="relative w-80">
  <Search className="absolute left-3 top-1/2 -translate-y-1/2 h-4 w-4 text-slate-400" />
  <input className="w-full rounded-full border border-slate-200 bg-white py-2 pl-9 pr-4 text-sm placeholder:text-slate-400 focus-visible:ring-2 focus-visible:ring-[#0F2A4D]" placeholder="Search vessels, incidents..." />
</div>
```

### 3.2 Molecules

#### Section Card

```jsx
<section className="rounded-2xl border border-slate-200 bg-white p-8 shadow-sm">
  <header className="mb-6">
    <h2 className="text-lg font-semibold text-slate-900">Nominasi Aktif</h2>
    <p className="text-sm text-slate-500 mt-1">Nominasi yang sedang dalam proses review atau menunggu pembayaran.</p>
  </header>
  {/* content */}
</section>
```

Variant: header bisa punya action di kanan (`flex items-start justify-between`). Severity card tambahkan `border-l-4 border-l-{color}` di outer.

#### KPI Card (Surface B)

```jsx
<div className="rounded-xl border border-slate-200 bg-white p-6">
  <div className="flex items-start justify-between mb-3">
    <span className="text-xs uppercase tracking-wider text-slate-500 font-medium">Active Vessels</span>
    <Ship className="h-4 w-4 text-slate-400" />
  </div>
  <div className="text-3xl font-bold text-slate-900">24</div>
  <div className="mt-3 flex flex-wrap gap-1.5">
    <StatusBadge variant="success">18 VERIFIED</StatusBadge>
    <StatusBadge variant="neutral">6 UNKNOWN</StatusBadge>
  </div>
</div>
```

Accent variant (warning/error/primary metric): tambah `border-l-4 border-l-{color}`.

#### Metric Tile (Surface A weather widget)

```jsx
<div className="flex items-start gap-3">
  <Waves className="h-5 w-5 text-slate-400 mt-1" />
  <div>
    <div className="text-xs uppercase tracking-wider text-slate-400">Tinggi Gelombang</div>
    <div className="text-2xl font-bold text-slate-900 mt-1">3.0 m</div>
  </div>
</div>
```

3-column grid: `grid grid-cols-1 md:grid-cols-3 gap-6`.

#### Data Table (di dalam Section Card)

- Header row: `text-xs uppercase tracking-wider text-slate-500 font-medium` (atau `text-sm text-slate-500`).
- Row: `border-b border-slate-100 last:border-0`, padding `py-4`, hover `hover:bg-slate-50`.
- Tidak ada vertical divider antar kolom.
- Action di kolom paling kanan pakai link-action style (lihat Button § Link-action).
- Empty state: row tunggal `colSpan` dengan text muted center "Tidak ada data."

#### Tabs (underline style)

```jsx
<div className="border-b border-slate-200 flex items-center gap-6 mb-6">
  <button className="border-b-2 border-[#0F2A4D] text-slate-900 font-medium pb-3 text-sm">
    Semua <span className="text-slate-400">(7)</span>
  </button>
  <button className="text-slate-500 hover:text-slate-900 pb-3 text-sm">Aktif</button>
  <button className="text-slate-500 hover:text-slate-900 pb-3 text-sm">Draft</button>
  <button className="text-slate-500 hover:text-slate-900 pb-3 text-sm">Selesai</button>
</div>
```

#### File Upload Dropzone

```jsx
<div className="rounded-xl border-2 border-dashed border-slate-300 bg-slate-50/40 p-12 text-center">
  <UploadCloud className="h-10 w-10 text-slate-400 mx-auto mb-4" />
  <p className="text-sm text-slate-700 font-medium">Seret & lepas file di sini, atau klik untuk memilih</p>
  <p className="text-xs text-slate-400 mt-1">PDF, JPG, PNG — maks. 10 MB per file</p>
  <button className="mt-5 inline-flex items-center gap-2 rounded-lg border border-slate-300 bg-white px-4 py-2 text-sm font-medium">
    <Upload className="h-4 w-4" /> Pilih File
  </button>
</div>
```

#### Document List Item

```jsx
<div className="flex items-center justify-between py-3 border-b border-slate-100 last:border-0">
  <div className="flex items-center gap-3">
    <div className="h-9 w-9 rounded-lg bg-slate-100 flex items-center justify-center">
      <FileText className="h-4 w-4 text-slate-500" />
    </div>
    <div>
      <div className="text-sm font-medium text-slate-900">Cargo Manifest</div>
      <div className="text-xs text-slate-400">PDF · 03 Mei 2026, 22.04</div>
    </div>
  </div>
  <button className="h-9 w-9 rounded-lg hover:bg-slate-100 inline-flex items-center justify-center">
    <Download className="h-4 w-4 text-slate-500" />
  </button>
</div>
```

#### Voyage Card (nested di dalam Section Card)

```jsx
<div className="rounded-xl border border-slate-200 bg-white p-6">
  <div className="flex items-start justify-between mb-4">
    <div className="flex items-start gap-3">
      <Ship className="h-5 w-5 text-slate-400 mt-0.5" />
      <div>
        <div className="text-base font-semibold text-slate-900">MV Sumatra Trader</div>
        <div className="text-xs text-slate-400">NOM-20260325-00005</div>
      </div>
    </div>
    <StatusBadge variant="success">Lunas</StatusBadge>
  </div>
  <dl className="space-y-1.5 text-sm">
    <div className="flex items-center gap-2 text-slate-600"><MapPin className="h-4 w-4 text-slate-400" /> AP-01</div>
    <div className="text-slate-600">ETB: 05 Mei 2026 pukul 03.04 WIB</div>
    <div className="text-slate-600">Cargo: Batubara</div>
  </dl>
  <button className="mt-5 w-full inline-flex items-center justify-center gap-2 rounded-lg border border-slate-300 bg-white py-2.5 text-sm font-medium hover:bg-slate-50">
    Lacak Posisi <ArrowRight className="h-4 w-4" />
  </button>
</div>
```

#### Invoice Detail Card (Surface A, v1.3)

Pattern detail tagihan yang menyerupai invoice fisik. Dipakai di Detail EPB (M9b), preview EPB di M9, dan dapat direuse untuk Invoice (M9c) saat detail butuh dirinci.

Struktur tiga blok berurutan (header status banner di luar card):
1. **Vessel Ops Grid** — info operasional voyage (2 kolom × N baris).
2. **Line Items Table** — Item Layanan / Volume / Rate / Jumlah + Subtotal / PPn / Total.
3. **Payment Instruction Box** — bank info + kode bayar + batas pembayaran dengan countdown.

```jsx
<section className="rounded-2xl border border-slate-200 bg-white shadow-sm overflow-hidden">
  {/* Block 1 — Vessel Ops Grid */}
  <div className="p-6 sm:p-8 border-b border-slate-100">
    <dl className="grid grid-cols-1 sm:grid-cols-2 gap-x-12 gap-y-4 text-sm">
      <div className="flex items-center justify-between">
        <dt className="text-slate-500">Vessel</dt>
        <dd className="font-semibold text-slate-900">MV Pacific Star</dd>
      </div>
      <div className="flex items-center justify-between">
        <dt className="text-slate-500">Crane</dt>
        <dd className="font-semibold text-slate-900">Crane 2</dd>
      </div>
      {/* … STS Slot, Mooring Team, ETA, Surveyor, Anchor, Est. Duration */}
    </dl>
  </div>

  {/* Block 2 — Line Items Table */}
  <div className="p-6 sm:p-8 border-b border-slate-100">
    <table className="w-full text-sm">
      <thead>
        <tr className="text-slate-500 border-b border-slate-200">
          <th className="text-left font-medium py-3">Item Layanan</th>
          <th className="text-right font-medium py-3">Volume</th>
          <th className="text-right font-medium py-3">Rate</th>
          <th className="text-right font-medium py-3">Jumlah</th>
        </tr>
      </thead>
      <tbody>
        <tr className="border-b border-slate-100">
          <td className="py-3.5 text-slate-900">STS Fee</td>
          <td className="py-3.5 text-right tabular-nums text-slate-700">50,000 MT</td>
          <td className="py-3.5 text-right tabular-nums text-slate-700">$2.50/ton</td>
          <td className="py-3.5 text-right tabular-nums font-semibold text-slate-900">$125,000</td>
        </tr>
        <tr className="border-b border-slate-100">
          <td className="py-3.5 text-slate-900">Biaya Jasa Tambahan</td>
          <td className="py-3.5 text-right text-slate-400">—</td>
          <td className="py-3.5 text-right text-slate-400">—</td>
          <td className="py-3.5 text-right text-slate-400">—</td>
        </tr>
      </tbody>
      <tfoot>
        <tr>
          <td colSpan={3} className="text-right py-2 text-slate-500">Subtotal</td>
          <td className="text-right py-2 tabular-nums font-semibold text-slate-900">$125,000</td>
        </tr>
        <tr>
          <td colSpan={3} className="text-right py-2 text-slate-500">PPn (11%)</td>
          <td className="text-right py-2 tabular-nums font-semibold text-slate-900">$13,750</td>
        </tr>
        <tr className="bg-slate-50/60">
          <td colSpan={3} className="text-right py-3.5 text-slate-700 font-semibold">Total</td>
          <td className="text-right py-3.5 tabular-nums text-lg font-bold text-[#0F2A4D]">$138,750</td>
        </tr>
      </tfoot>
    </table>
  </div>

  {/* Block 3 — Payment Instruction Box */}
  <div className="m-6 sm:m-8 rounded-xl border border-amber-200 bg-amber-50/50 p-5 sm:p-6">
    <div className="text-sm font-semibold text-amber-900 mb-4">Instruksi Pembayaran</div>
    <dl className="grid grid-cols-1 sm:grid-cols-2 gap-x-12 gap-y-3 text-sm">
      <div>
        <dt className="text-xs text-amber-800/70">Bank</dt>
        <dd className="font-semibold text-slate-900 mt-0.5">BNI</dd>
      </div>
      <div>
        <dt className="text-xs text-amber-800/70">Kode Bayar</dt>
        <dd className="font-semibold text-slate-900 mt-0.5 font-mono">TBK-202603-001</dd>
      </div>
      <div>
        <dt className="text-xs text-amber-800/70">No Rekening</dt>
        <dd className="font-semibold text-slate-900 mt-0.5 font-mono">1234567890</dd>
      </div>
      <div>
        <dt className="text-xs text-amber-800/70">Batas Pembayaran</dt>
        <dd className="font-semibold text-rose-700 mt-0.5">5 Mar 2026 <span className="text-rose-600">(3 hari)</span></dd>
      </div>
      <div>
        <dt className="text-xs text-amber-800/70">Atas Nama</dt>
        <dd className="font-semibold text-slate-900 mt-0.5">PT Tata Bumi Khatulistiwa</dd>
      </div>
      <div>
        <dt className="text-xs text-amber-800/70">Total</dt>
        <dd className="font-bold text-[#0F2A4D] mt-0.5 tabular-nums">$138,750</dd>
      </div>
    </dl>
  </div>

  {/* Footer actions (slot) */}
  <div className="flex flex-col-reverse sm:flex-row items-stretch sm:items-center justify-end gap-3 px-6 sm:px-8 pb-6 sm:pb-8">
    <button className="inline-flex items-center justify-center gap-2 rounded-lg border border-slate-300 bg-white px-4 py-2.5 text-sm font-medium hover:bg-slate-50">
      <Download className="h-4 w-4" /> Download EPB PDF
    </button>
    <button className="inline-flex items-center justify-center gap-2 rounded-lg bg-[#0F2A4D] text-white px-4 py-2.5 text-sm font-medium hover:bg-[#0F2A4D]/90">
      <CheckCircle2 className="h-4 w-4" /> Konfirmasi Pembayaran
    </button>
  </div>
</section>
```

**Aturan pakai:**
- Currency formatter: gunakan `Intl.NumberFormat(locale, { style: 'currency', currency })` di runtime. `currency` field dari payload menentukan render (IDR → "Rp 138.750.000", USD → "$138,750"). Angka di `tabular-nums` agar kolom rapi.
- Volume & Rate adalah string display dari STS (bukan dihitung di LPS) — jangan reformat.
- **Compact variant (untuk preview di M9):** sembunyikan block 3 (Payment Instruction Box) dan footer actions; tetap tampilkan block 1 + block 2. Tombol "Bayar EPB" + "Download EPB PDF" dipindah ke action card kontextual di luar.
- **Fallback minimal (data legacy):** jika `line_items` kosong dan `vessel_ops` null, render fallback ke pattern "Detail Tagihan" lama (Nomor EPB / Total / Currency / Due Date) — tidak boleh kosong/broken.

#### Payment Instruction Box (standalone, v1.3)

Sub-component dari Invoice Detail Card. Bisa juga dipakai standalone (misal di info card sederhana). Pattern fixed amber tone untuk "ada aksi pembayaran yang harus dilakukan". Lihat block 3 di atas. Wajib include 5 field minimum: Bank, No Rekening, Atas Nama, Kode Bayar, Batas Pembayaran. Field `Kode Bayar` dan `No Rekening` pakai `font-mono` agar mudah disalin.

**Countdown indicator (`Batas Pembayaran`):**
- ≤ 3 hari atau overdue: `text-rose-700` + (X hari) atau "Lewat X hari".
- 4–7 hari: `text-amber-800`.
- > 7 hari: `text-slate-900` (no emphasis).

#### Line Items Table (standalone, v1.3)

Sub-component dari Invoice Detail Card. Pattern: header row muted, body row dengan border-bottom slate-100, tfoot rows dengan label di kanan dan amount `tabular-nums`. Baris **Total** pakai bg-slate-50/60, font-bold, color `#0F2A4D`, size `text-lg`. Lihat block 2 di Invoice Detail Card.

#### Status Banner (di detail page)

```jsx
<div className="rounded-xl border-l-4 border-l-blue-500 border border-slate-200 bg-blue-50/40 p-5">
  <div className="flex items-start gap-3">
    <Info className="h-5 w-5 text-blue-600 mt-0.5" />
    <div>
      <div className="font-semibold text-slate-900">Menunggu Review STS Platform</div>
      <p className="text-sm text-slate-600 mt-1">Nominasi Anda sedang diverifikasi oleh tim STS. Estimasi waktu review 1–2 hari kerja.</p>
    </div>
  </div>
</div>
```

#### Activity Feed Item (Surface B)

```jsx
<div className="flex items-center justify-between py-3">
  <div>
    <div className="text-sm font-medium text-slate-900">MV. Sinar Bangun</div>
    <div className="text-xs text-slate-400">Bulk Carrier · 10:45 AM</div>
  </div>
  <StatusBadge variant="error">ARRIVED</StatusBadge>
</div>
```

#### Alert List Item

```jsx
<div className="rounded-lg border border-rose-200 bg-rose-50/40 p-4">
  <div className="flex items-start justify-between gap-3">
    <div className="flex items-start gap-2.5">
      <AlertTriangle className="h-4 w-4 text-rose-600 mt-0.5" />
      <div>
        <div className="text-sm font-bold text-rose-700 uppercase tracking-wide">Oil Spill Detected</div>
        <p className="text-sm text-slate-700 mt-1">INC-20260311-001 reported near AP-1.</p>
        <p className="text-xs text-slate-400 mt-1">10:42 AM</p>
      </div>
    </div>
  </div>
</div>
```

#### Empty State

```jsx
<div className="py-12 text-center">
  <p className="text-sm text-slate-400">Tidak ada riwayat peringatan cuaca.</p>
</div>
```

Untuk empty state yang lebih impactful (halaman kosong): tambah icon `h-12 w-12 text-slate-300 mx-auto mb-4` + CTA button di bawah copy.

#### Modal Dialog (form / detail) — v1.4

Pattern modal terpusat untuk form (Add/Edit) atau detail view. Basis shadcn/ui `Dialog`. Dipakai di master-data CRUD (M13) dan form modal lain.

```jsx
{/* Overlay */}
<div className="fixed inset-0 z-50 bg-slate-900/50 backdrop-blur-[1px]" />

{/* Dialog panel — centered, scrollable body */}
<div className="fixed left-1/2 top-1/2 z-50 w-full max-w-2xl -translate-x-1/2 -translate-y-1/2
                rounded-2xl border border-slate-200 bg-white shadow-md
                max-h-[90vh] flex flex-col">
  {/* Header (sticky) */}
  <header className="flex items-start justify-between gap-4 border-b border-slate-200 px-6 py-5">
    <div>
      <h2 className="text-lg font-semibold text-slate-900">Add New MV</h2>
      <p className="text-sm text-slate-500 mt-1">Lengkapi detail di bawah. Aksi ini membuat permintaan persetujuan.</p>
    </div>
    <button aria-label="Tutup" className="h-9 w-9 rounded-lg hover:bg-slate-100 inline-flex items-center justify-center">
      <X className="h-4 w-4 text-slate-500" />
    </button>
  </header>

  {/* Body — scrollable */}
  <div className="flex-1 overflow-y-auto px-6 py-5 space-y-5">
    {/* form / detail content */}
  </div>

  {/* Footer (sticky) — actions kanan */}
  <footer className="flex items-center justify-end gap-3 border-t border-slate-200 px-6 py-4">
    <button className="bg-white border border-slate-300 text-slate-900 hover:bg-slate-50 rounded-lg px-4 py-2.5 font-medium">Batal</button>
    <button className="bg-[#0F2A4D] text-white hover:bg-[#0B1F3A] rounded-lg px-4 py-2.5 font-medium">Submit for Approval</button>
  </footer>
</div>
```

**Aturan:**
- Width: `max-w-md` (confirm kecil), `max-w-2xl` (form sedang ~2 kolom), `max-w-3xl` (form besar / detail).
- Body scroll internal (`overflow-y-auto`), header & footer sticky (tidak ikut scroll). Modal tidak boleh > `90vh`.
- Overlay `bg-slate-900/50` (memenuhi standar scrim 40–60%). Klik overlay = close (kecuali form dengan unsaved changes → konfirmasi).
- Esc close. Focus trap aktif; focus pindah ke field pertama (form) atau heading (detail) saat open. Saat close, focus balik ke trigger.
- Motion: `initial={{ opacity: 0, scale: 0.96 }} animate={{ opacity: 1, scale: 1 }} transition={{ duration: 0.15 }}` (§2.8 Modal enter).
- Hanya **satu primary** button di footer (kanan), secondary di kiri-nya (outlined).

#### Confirmation Dialog (destructive / state change) — v1.4

Varian Modal Dialog ukuran kecil untuk konfirmasi aksi berisiko (Deactivate, Hapus draft, dll).

```jsx
<div className="fixed left-1/2 top-1/2 z-50 w-full max-w-md -translate-x-1/2 -translate-y-1/2
                rounded-2xl border border-slate-200 bg-white shadow-md p-6">
  <div className="flex items-start gap-4">
    <div className="h-10 w-10 rounded-full bg-amber-50 flex items-center justify-center flex-shrink-0">
      <AlertTriangle className="h-5 w-5 text-amber-600" />
    </div>
    <div className="flex-1">
      <h2 className="text-base font-semibold text-slate-900">Nonaktifkan MV Pacific Glory?</h2>
      <p className="text-sm text-slate-600 mt-1.5">
        Status akan diubah menjadi <strong>Inactive</strong> setelah disetujui admin STS. MV tidak akan
        muncul di pilihan nominasi.
      </p>
    </div>
  </div>
  <footer className="flex items-center justify-end gap-3 mt-6">
    <button className="bg-white border border-slate-300 text-slate-900 hover:bg-slate-50 rounded-lg px-4 py-2.5 font-medium">Batal</button>
    <button className="bg-[#0F2A4D] text-white hover:bg-[#0B1F3A] rounded-lg px-4 py-2.5 font-medium">Submit for Approval</button>
  </footer>
</div>
```

**Aturan:**
- Icon bulat kiri: `bg-amber-50 + AlertTriangle text-amber-600` untuk **state change non-destruktif yang butuh approval** (Deactivate via approval). Untuk **delete permanen / destruktif sungguhan** pakai `bg-rose-50 + Trash2 text-rose-600` dan button konfirmasi `Destructive` (rose).
- Copy harus sebut konsekuensi konkret ("Status akan diubah menjadi Inactive", "tidak muncul di pilihan nominasi") — bukan generik "Apakah Anda yakin?".
- Tombol konfirmasi label aksi spesifik ("Submit for Approval", "Hapus Permanen"), bukan "OK"/"Ya".
- Default focus ke tombol **Batal** (mencegah konfirmasi tak sengaja via Enter).

#### Form Field Grid — v1.4

Pattern grid form 2-kolom untuk modal form dengan banyak field (master-data). Mendukung field disabled (auto-generated/auto-set) dan inline validation.

```jsx
<form className="space-y-6">
  {/* Disclaimer banner — selalu di atas form yang membuat approval request */}
  <div className="rounded-lg border border-yellow-300 bg-yellow-50 px-4 py-3 flex items-start gap-2.5">
    <AlertTriangle className="h-4 w-4 text-yellow-700 mt-0.5 flex-shrink-0" />
    <p className="text-sm text-yellow-800">
      Semua perubahan dikirim sebagai permintaan persetujuan dan memerlukan review admin sebelum diterapkan.
    </p>
  </div>

  {/* 2-column grid */}
  <div className="grid grid-cols-1 sm:grid-cols-2 gap-x-5 gap-y-4">
    {/* Field normal + required */}
    <div className="space-y-1.5">
      <label htmlFor="vesselName" className="text-sm font-medium text-slate-700">
        Vessel Name <span className="text-rose-600">*</span>
      </label>
      <input id="vesselName" className="w-full rounded-lg border border-slate-300 bg-white px-3 py-2.5 text-sm
        text-slate-900 placeholder:text-slate-400 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-[#0F2A4D]" />
      {/* error state (muncul on blur) */}
      {/* <p className="text-rose-600 text-xs mt-1">Vessel Name wajib diisi.</p> */}
    </div>

    {/* Field disabled — auto-generated */}
    <div className="space-y-1.5">
      <label htmlFor="assetCode" className="text-sm font-medium text-slate-700">Asset Code</label>
      <input id="assetCode" disabled placeholder="MV-XXXX"
        className="w-full rounded-lg border border-slate-300 bg-slate-100 px-3 py-2.5 text-sm
        text-slate-500 cursor-not-allowed" />
      <p className="text-xs text-slate-400">Dibuat otomatis saat disimpan.</p>
    </div>

    {/* Field full-width (span 2): contoh untuk catatan panjang */}
    <div className="space-y-1.5 sm:col-span-2">
      {/* ... */}
    </div>
  </div>
</form>
```

**Aturan:**
- Grid `grid-cols-1 sm:grid-cols-2 gap-x-5 gap-y-4`. Field panjang/penuh boleh `sm:col-span-2`.
- **Required indicator:** asterisk `<span className="text-rose-600">*</span>` setelah label. Optional field tanpa asterisk (tidak pakai teks "(opsional)" agar ringkas, kecuali butuh disambiguasi).
- **Label wajib `<label htmlFor>`** — tidak boleh placeholder-only (a11y).
- **Disabled vs read-only:** field auto-generated/auto-set pakai `disabled` + `bg-slate-100 text-slate-500 cursor-not-allowed` + helper text menjelaskan kenapa ("Dibuat otomatis saat disimpan.", "Otomatis diisi sesuai organisasi Anda."). Bedakan jelas dari field aktif.
- **Inline validation:** validasi **on blur** (bukan on keystroke, bukan submit-only). Error tampil di bawah field: `text-rose-600 text-xs mt-1` + border field jadi `border-rose-300`. Setelah submit gagal, auto-focus field invalid pertama.
- **Field grouping:** untuk form >8 field, kelompokkan secara visual dengan subheading (`text-xs uppercase tracking-wider text-slate-400 font-medium sm:col-span-2 pt-2`) per grup logis (mis. "Identitas", "Dimensi & Kapasitas").
- **Submit feedback:** tombol submit `disabled` + spinner saat loading; sukses → toast + close modal; error → toast + tetap buka, preserve input.

#### Audit Timeline (Riwayat Perubahan) — v1.4

Pattern vertical timeline untuk menampilkan riwayat approval request / audit log per entity.

```jsx
<ol className="relative space-y-6">
  {history.map((item, i) => (
    <li key={item.id} className="relative pl-8">
      {/* Connector line (kecuali item terakhir) */}
      {i !== history.length - 1 && (
        <span className="absolute left-[11px] top-6 bottom-[-24px] w-px bg-slate-200" aria-hidden />
      )}
      {/* Dot status */}
      <span className={`absolute left-0 top-1 h-[22px] w-[22px] rounded-full border-2 flex items-center justify-center
        ${item.status === 'APPROVED' ? 'border-emerald-200 bg-emerald-50'
        : item.status === 'REJECTED' ? 'border-rose-200 bg-rose-50'
        : 'border-teal-200 bg-teal-50'}`}>
        {item.status === 'APPROVED' ? <Check className="h-3 w-3 text-emerald-600" />
        : item.status === 'REJECTED' ? <X className="h-3 w-3 text-rose-600" />
        : <Clock className="h-3 w-3 text-teal-600" />}
      </span>

      <div className="flex items-start justify-between gap-3">
        <div>
          <div className="text-sm font-medium text-slate-900">
            {item.actionLabel} {/* "Pengajuan MV Baru" / "Perubahan Data" / "Nonaktifkan" */}
          </div>
          <div className="text-xs text-slate-500 mt-0.5">
            oleh {item.submitterName} · {item.submittedAt}
          </div>
        </div>
        <StatusBadge variant={mapVariant(item.status)}>{item.statusLabel}</StatusBadge>
      </div>

      {/* Resolution detail */}
      {item.status !== 'PENDING_APPROVAL' && (
        <div className="text-xs text-slate-500 mt-1.5">
          {item.status === 'APPROVED' ? 'Disetujui' : 'Ditolak'} oleh {item.reviewerName} · {item.resolvedAt}
        </div>
      )}
      {/* Rejection reason — emphasized */}
      {item.status === 'REJECTED' && item.rejectionReason && (
        <div className="mt-2 rounded-lg border border-rose-200 bg-rose-50/60 px-3 py-2 text-sm text-rose-700">
          Alasan: {item.rejectionReason}
        </div>
      )}
    </li>
  ))}
</ol>
```

**Aturan:**
- Urutan **descending** (terbaru di atas).
- Dot status: Approved emerald + `Check`, Rejected rose + `X`, Pending teal + `Clock`. Warna ambil dari Status palette §2.1.
- Connector line `w-px bg-slate-200` antar item; item terakhir tanpa connector.
- Rejection reason wajib ditonjolkan dalam box rose (`bg-rose-50/60`) — bukan teks biasa, karena ini info paling actionable bagi user.
- Empty state: "Belum ada riwayat perubahan." (jarang; entry pertama selalu ada saat record dibuat).

#### Disclaimer Banner — v1.4

Banner peringatan ringan (non-error) untuk set ekspektasi sebelum aksi. Berbeda dari Inline Info Banner (yang per-status di list item) — ini untuk form/page context.

```jsx
<div className="rounded-lg border border-yellow-300 bg-yellow-50 px-4 py-3 flex items-start gap-2.5">
  <AlertTriangle className="h-4 w-4 text-yellow-700 mt-0.5 flex-shrink-0" />
  <p className="text-sm text-yellow-800">{message}</p>
</div>
```

Pakai variant **Severity Warning** palette (§2.1). Untuk konteks informatif netral (bukan peringatan) pakai border/ bg `blue-200/blue-50` + icon `Info text-blue-600`.

#### Locked Row Action (pending approval) — v1.4

Affordance untuk icon aksi yang dikunci sementara (mis. Edit/Deactivate saat record `PENDING_APPROVAL`).

```jsx
{/* Enabled icon action */}
<button className="h-9 w-9 rounded-lg hover:bg-slate-100 inline-flex items-center justify-center"
        aria-label="Edit MV">
  <Pencil className="h-4 w-4 text-slate-500" />
</button>

{/* Locked icon action — disabled + tooltip */}
<Tooltip content="Edit tidak tersedia — sedang menunggu approval admin.">
  <button disabled aria-label="Edit terkunci (menunggu approval)"
    className="h-9 w-9 rounded-lg inline-flex items-center justify-center opacity-40 cursor-not-allowed">
    <Pencil className="h-4 w-4 text-slate-400" />
  </button>
</Tooltip>
```

**Aturan:**
- Locked = `disabled` + `opacity-40 cursor-not-allowed` + tooltip menjelaskan **kenapa** terkunci dan **kapan** akan terbuka.
- `aria-label` harus menyatakan status terkunci untuk screen reader (jangan hanya andalkan opacity/tooltip).
- Icon View **tetap enabled** saat row lain dikunci (read tidak pernah dikunci).
- Jangan sembunyikan tombol — disable + jelaskan (prinsip `empty-nav-state`: jelaskan kenapa unavailable, jangan diam-diam hilang).

### 3.3 Organisms

#### Sidebar A (Customer Portal)

Lebar `w-64`, full height, bg `#0F2A4D`. Struktur:

```
┌─────────────────────────────┐
│ [Ship icon] PT Tata Bumi... │  ← Brand area, p-5
│            PORTAL PELANGGAN │     subtitle uppercase tracked
├─────────────────────────────┤
│ [Icon] Dashboard            │  ← Nav items, py-2 px-4
│ [Icon] Nominasi             │     active: bg navy lighter
│ [Icon] EPB & Invoice        │
│ [Icon] Document Master      │
│ [Icon] Cuaca & Alert        │
│                             │
│                             │
│           ...               │
├─────────────────────────────┤
│ [Icon] Logout               │  ← Sticky bottom
└─────────────────────────────┘
```

Active nav: `bg-[#1E3A66] text-white`. Inactive: `text-slate-300 hover:bg-[#13315A] hover:text-white`. Tiap item `rounded-lg mx-3 px-3 py-2.5 flex items-center gap-3 text-sm font-medium`.

#### Sidebar B (Operator) — collapsible + nested

Lebar `w-60`, collapsible ke `w-16`. Bg `#0F2A4D`. Tombol collapse di top-right brand area (chevron).

```
┌─────────────────────────────┐
│ [Ship] LPS SYSTEM       <   │  ← Brand + collapse toggle
│        BUNATI PORT          │
├─────────────────────────────┤
│ DASHBOARD              [✓]  │  ← Level 1: UPPERCASE
│ VESSEL MONITORING       ^   │     dengan chevron expand
│   LIVE TRACKING        [✓]  │  ← Level 2: indented
│   VESSEL LIST               │
│   ANCHOR POINTS             │
│   ALERT LOG                 │
│   AIS MESSAGES              │
│   AIS HEALTH                │
│ COMMUNICATION               │
│ WEATHER                 ^   │
│   DASHBOARD                 │
│   WEATHER LOGS              │
│ INCIDENTS                   │
│ CUSTOMERS                   │
│ SETTINGS                    │
├─────────────────────────────┤
│ ● System Online             │  ← Status indicator
└─────────────────────────────┘
```

Level 1 (group/leaf top): `text-xs uppercase tracking-wider font-semibold px-4 py-3 text-slate-300`. Active leaf level 1: `bg-[#1E3A66] text-white`. Sub-item level 2: `pl-10 py-2 text-xs uppercase tracking-wider`. Chevron rotate on expand. Status indicator bottom: `<span className="h-2 w-2 rounded-full bg-emerald-400" />` + label.

#### Top Bar B (Operator)

```
┌─────────────────────────────────────────────────────────────────────┐
│ [Search...]              Wed, May 13, 06:20 PM  2.1m●  ☾  🔔  [Andi]│
└─────────────────────────────────────────────────────────────────────┘
```

`h-16 border-b border-slate-200 bg-white px-6 flex items-center justify-between sticky top-0 z-10`.
- Kiri: search field rounded-full.
- Kanan cluster: datetime (text muted), waves indicator (mini stat dengan dot indicator hijau/kuning/merah berdasar threshold), theme toggle, notification bell (`relative`, dot indicator merah di kanan-atas saat ada unread), user pill (`h-9 px-3 rounded-lg hover:bg-slate-100 flex items-center gap-2`: avatar + name + role muted), logout icon button.

#### Data Table (full pattern)

Lihat Molecules §3.2 Data Table. Untuk volume besar tambahkan: filter bar di atas table (search + select), pagination di bawah (`Showing 1–10 of 47` + prev/next buttons).

#### Map Widget (Leaflet + OpenStreetMap)

- **Library:** `react-leaflet` + `leaflet` CSS.
- **Tile:** OpenStreetMap standard (`https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png`).
- **Container:** `rounded-xl border border-slate-200 overflow-hidden h-[480px]` (atau `h-[640px]` untuk full page).
- **Vessel marker:** custom divIcon — segitiga arah (rotated) + label tooltip di bawah.
  - Registered: emerald (`#10B981`).
  - Unknown: rose (`#F43F5E`).
  - Anchor point: anchor icon (Lucide) emerald/rose berdasar status.
- **Legend overlay:** kiri-bawah, white card dengan shadow, list kecil warna + label.
- **Zoom control:** kiri-atas (default Leaflet, style override agar match button outlined kita).
- **Status chips overlay:** kanan-atas atau di toolbar atas (Active/Loading/Urgent counts).
- **Alerts panel side-by-side:** `lg:grid-cols-[1fr_360px]` — map kiri, alerts kanan dengan scroll.

#### Chart Widget (Recharts)

- **Library:** `recharts`.
- **Bar chart (threshold coloring):** lihat Weather Dashboard forecast.
  - Green `#10B981` for safe (< warning threshold).
  - Yellow `#F59E0B` for warning (≥ warning, < danger threshold).
  - Red `#EF4444` for danger (≥ danger threshold).
  - Dashed horizontal threshold lines dengan `<ReferenceLine strokeDasharray="4 4">`.
- **Axis style:** `tick={{ fill: '#94A3B8', fontSize: 11 }}`, axis line muted `stroke: '#E5E9F0'`.
- **Grid:** `<CartesianGrid strokeDasharray="3 3" vertical={false} stroke="#E5E9F0" />`.
- **Tooltip:** custom dengan card style (`rounded-lg border bg-white shadow-md p-3 text-xs`).
- **Empty state:** "No trend data available" center, text muted.

### 3.4 Templates (page layouts)

#### Auth Split-Screen layout (Customer-facing auth pages)

Dipakai untuk customer login & customer register. Bukan untuk admin login (admin pakai centered card minimal — lihat di bawah).

```
┌─────────────────────────────────────────────────────────────┐
│ [Brand mark wordmark]      │                                │
│                            │                                │
│                            │                                │
│      ┌─────────────────┐   │     [Image showcase            │
│      │ Heading large   │   │      full-bleed]               │
│      │ Subtitle muted  │   │                                │
│      │                 │   │     Optional carousel of       │
│      │ [Form fields    │   │     value propositions with    │
│      │  dengan         │   │     text overlay bottom-left.  │
│      │  leading icon]  │   │                                │
│      │                 │   │     Dots indicator             │
│      │ [Primary CTA]   │   │     bottom-right.              │
│      │                 │   │                                │
│      │ Divider "atau"  │   │                                │
│      │                 │   │                                │
│      │ Secondary link  │   │                                │
│      └─────────────────┘   │                                │
│                            │                                │
│  Footer text muted         │                                │
└─────────────────────────────────────────────────────────────┘
        50%                              50%
```

**Implementation:**

```jsx
<div className="min-h-screen flex">
  {/* Left: form panel */}
  <div className="w-full lg:w-1/2 bg-slate-50/40 flex flex-col">
    {/* Brand mark top-left */}
    <div className="px-12 pt-10">
      <div className="inline-flex items-center gap-2">
        <Anchor className="h-5 w-5 text-[#0F2A4D]" />
        <span className="text-base font-semibold text-slate-900">LPS System</span>
      </div>
    </div>

    {/* Form centered vertically */}
    <div className="flex-1 flex items-center justify-center px-12">
      <div className="w-full max-w-md space-y-6">
        <header>
          <h1 className="text-4xl font-semibold text-slate-900">Selamat Datang</h1>
          <p className="text-sm text-slate-500 mt-2">Masuk ke Portal Pelanggan LPS</p>
        </header>
        {/* form fields */}
      </div>
    </div>

    {/* Footer */}
    <div className="px-12 pb-8 text-center">
      <p className="text-xs text-slate-400">© 2026 LPS System. Seluruh hak dilindungi.</p>
    </div>
  </div>

  {/* Right: image showcase — hidden di mobile */}
  <div className="hidden lg:block lg:w-1/2 relative">
    <ImageShowcase slides={authSlides} />
  </div>
</div>
```

**Breakpoint behavior:**
- `< lg` (1024px): image panel hidden, form full-width, brand mark + footer tetap visible, form max-w-md centered.
- `≥ lg`: split 50/50.

**Spacing:**
- Form panel padding: `px-12 py-10` (48px horizontal, 40px vertical).
- Form max-width: `max-w-md` (~448px).
- Brand mark margin-top: `pt-10` (40px).
- Footer margin-bottom: `pb-8` (32px).

#### Image Showcase Carousel (Auth right panel)

Pattern: full-bleed image dengan caption overlay bottom-left, dots indicator bottom-right, auto-advance optional.

```jsx
<div className="relative h-full w-full overflow-hidden">
  {/* Active slide */}
  <img
    src={slide.image}
    alt={slide.alt}
    className="h-full w-full object-cover"
  />

  {/* Subtle gradient untuk readability text di bottom */}
  <div className="absolute inset-x-0 bottom-0 h-1/3 bg-gradient-to-t from-black/40 to-transparent" />

  {/* Caption overlay */}
  <div className="absolute bottom-12 left-12 max-w-md text-white">
    <h2 className="text-3xl font-semibold drop-shadow-md">
      {slide.heading}
    </h2>
    <p className="mt-2 text-sm text-white/90 drop-shadow-md">
      {slide.subtitle}
    </p>
  </div>

  {/* Dots indicator */}
  <div className="absolute bottom-12 right-12 flex items-center gap-2">
    {slides.map((s, i) => (
      <button
        key={s.id}
        onClick={() => setActive(i)}
        aria-label={`Slide ${i + 1}`}
        className={
          i === active
            ? "h-1.5 w-6 rounded-full bg-white"
            : "h-1.5 w-1.5 rounded-full bg-white/50 hover:bg-white/80"
        }
      />
    ))}
  </div>
</div>
```

**Slide content rules:**
- Image: high-quality maritime photo, landscape orientation, min 1920×1080, focus subject sebelah kanan/center (caption di kiri tidak menutupi subject).
- Heading: 2–4 kata, kapital di awal kata utama, **bahasa sesuai surface** (Surface A → ID).
- Subtitle: 1 kalimat, max ~80 karakter, deskripsi value/benefit.
- Active dot: panjang `w-6 h-1.5`, putih solid. Inactive: bulat kecil `w-1.5 h-1.5`, putih semi-transparent.
- Auto-advance: 6 detik per slide (optional). Pause saat hover.

**Default slides untuk Customer Portal auth (3 slides):**

| # | Image theme | Heading (ID) | Subtitle (ID) |
|---|---|---|---|
| 1 | Kapal bulk carrier sandar / loading di STS | "Keselamatan & Keandalan" | "Standar operasional terbaik untuk setiap pelayaran." |
| 2 | Operator monitoring AIS map / dashboard | "Integrasi STS Real-time" | "Status nominasi, EPB, dan voyage Anda selalu sinkron dengan STS Platform." |
| 3 | Kondisi cuaca/laut atau radar VTS | "Pantauan 24/7" | "Monitoring cuaca, voyage, dan keselamatan area STS Bunati tanpa jeda." |

Disimpan sebagai konstanta config di frontend (e.g., `src/config/authSlides.ts`).

#### Auth Centered layout (Admin & operator auth)

Dipakai untuk admin login dan halaman auth fungsional internal. Minimal, tanpa showcase.

```jsx
<div className="min-h-screen flex items-center justify-center bg-slate-50 px-4">
  <div className="w-full max-w-md">
    <div className="mb-8 text-center">
      <div className="inline-flex items-center gap-2 mb-6">
        <Anchor className="h-6 w-6 text-[#0F2A4D]" />
        <span className="text-lg font-semibold text-slate-900">LPS System</span>
      </div>
      <h1 className="text-2xl font-semibold text-slate-900">Sign In</h1>
      <p className="text-sm text-slate-500 mt-1">Operations console — Bunati Port</p>
    </div>
    <div className="rounded-2xl border border-slate-200 bg-white p-8 shadow-sm space-y-5">
      {/* form fields */}
    </div>
  </div>
</div>
```

Brand mark stacked (icon + wordmark) di atas card. Card pakai pattern Section Card standard. English copy.

#### Customer Portal page layout (Surface A)

```jsx
<div className="min-h-screen flex bg-slate-50">
  <SidebarA />
  <main className="flex-1 p-10 space-y-8">
    <PageHeader title="Dashboard" subtitle="..." action={<PrimaryButton />} />
    {/* sections */}
  </main>
</div>
```

`PageHeader` pattern: `<h1 className="text-3xl font-semibold text-slate-900">...</h1>` + subtitle `<p className="text-sm text-slate-500 mt-1.5">...</p>` + action button di kanan via flex.

#### Operator page layout (Surface B)

```jsx
<div className="min-h-screen flex bg-slate-50">
  <SidebarB />
  <div className="flex-1 flex flex-col">
    <TopBarB />
    <main className="flex-1 p-8 space-y-6">
      <PageHeader title="Dashboard" subtitle="Overview of port operations and live tracking" />
      {/* KPI grid */}
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
        <KpiCard ... />
        <KpiCard ... />
        <KpiCard ... />
        <KpiCard ... />
      </div>
      {/* main content grid */}
      <div className="grid grid-cols-1 lg:grid-cols-[1fr_360px] gap-6">
        {/* map / chart */}
        {/* side panel */}
      </div>
    </main>
  </div>
</div>
```

---

## 4. Responsive Strategy

LPS Platform **desktop-first** (operator selalu desktop, customer mayoritas desktop). Tetap mobile-aware tapi bukan mobile-first.

| Breakpoint | Tailwind | Strategi |
|---|---|---|
| `< 768px` (mobile) | default | Sidebar drawer (hamburger toggle), single column, stack semua KPI vertikal, table → card list. |
| `768–1023px` (tablet) | `md:` | Sidebar tetap di kiri tapi compact, KPI 2-up, content stay table. |
| `≥ 1024px` (desktop) | `lg:` | Layout penuh, sidebar expanded, KPI 4-up, side-by-side panels. |
| `≥ 1280px` (wide) | `xl:` | Tidak ada perubahan struktural, hanya max-width container yang melonggar. |

Map widget: di mobile jadi full-width sebelum alerts panel (alerts pindah di bawah).

---

## 5. Definition of Done (UI work)

Sebelum klaim selesai untuk pekerjaan UI:

1. ✅ Komponen pakai token dari design system ini (warna, spacing, radius). Tidak ada hardcoded hex random.
2. ✅ Status badge pakai variant yang ada di §2.1 Status palette. Label match dengan §2.1 Status mapping LPS.
3. ✅ Layout pakai template yang sesuai (Surface A atau B).
4. ✅ Copy match dengan tone (ID/EN) sesuai surface.
5. ✅ Empty/loading/error state ada (jangan biarkan blank).
6. ✅ Focus ring visible di semua interactive element.
7. ✅ Contrast lulus AA (cek dengan extension Stark/Axe atau visual inspect untuk text di navy).
8. ✅ Sudah invoke skill `ui-ux-pro-max` untuk validasi atau generate komponen kompleks.
9. ✅ Per-modul design doc terkait sudah diupdate jika ada pattern baru.

---

## 6. Cross-references

- Foundational scope per modul: `document/BRD-LPS-V3-Analysis.md` + `document/brd/<modul>.md`.
- Penerapan UI per modul: `implementation/design/m<N>-<name>-ui.md`.
- Module scoping (4-file rule): `module/<submodule>/`.
- Execution handoff: `implementation/replit-handoff/m<N>-<name>.md` (harus reference dokumen ini di Prerequisites).
- Policy enforcement: `CLAUDE.md` section "UI/UX Standard".

---

## 7. Changelog

| Tanggal | Versi | Perubahan |
|---|---|---|
| 2026-05-13 | 1.0 | Initial release — foundation, dua surface preset, component library, mapping status LPS. Dibuat dari analisis screenshot production (Customer Portal + LPS System Bunati Port). |
| 2026-05-13 | 1.1 | Tambah pattern auth split-screen layout (customer login/register), Input with leading icon + eye toggle, Divider with text, Image Showcase Carousel dengan 3 default slides Surface A, Auth Centered layout (admin login), brand mark wordmark variant (anchor + "LPS System"), canonical naming. Dipicu oleh screenshot login customer baru. |
| 2026-05-14 | 1.3 | **Invoice-style Detail Card** baru di §3.2: tiga blok (Vessel Ops Grid + Line Items Table + Payment Instruction Box) untuk Detail EPB invoice-like. Sub-komponen baru: **Line Items Table** (standalone, table dengan tfoot Subtotal/PPn/Total dan baris Total emphasized) dan **Payment Instruction Box** (amber tone, font-mono untuk Kode Bayar & No Rek, countdown indicator pada Batas Pembayaran). Currency support multi-currency (IDR/USD) via `Intl.NumberFormat`. Compact variant untuk preview di M9 (Block 1+2 only, no payment instruction). Dipicu oleh permintaan improve UI Detail Tagihan menyerupai invoice fisik. |
| 2026-05-13 | 1.2 | **Status label sync dengan production:** UNPAID→"Belum Dibayar", Pembayaran Dikonfirmasi→"Lunas", Menunggu Verifikasi Pembayaran→"Menunggu Verifikasi" (lebih ringkas). Tambah kategori filter "Perlu Tindakan" (UNPAID+PAYMENT_REJECT+Overdue) sebagai meta-status untuk KPI pill & tab. Tambah komponen baru: KPI Filter Pill (3 default kategori billing), Overdue Date Display (deteksi due_date < now), Inline Info Banner (varian per status untuk list items). Tambah Overdue indicator palette token. Dipicu oleh pemisahan M9b/M9c dan screenshot production EPB & Invoice. |
| 2026-05-15 | 1.4 | **Master-data CRUD patterns** untuk M13 (Mother Vessel Master). Tambah status mapping `ACTIVE`/`INACTIVE` (master data) + `PENDING_APPROVAL`/`REJECTED` (approval request) di §2.1. Komponen molecule baru di §3.2: **Modal Dialog** (form/detail, header+body scroll+footer sticky), **Confirmation Dialog** (state change/destructive, icon bulat + copy konsekuensi konkret), **Form Field Grid** (2-kolom, required asterisk, disabled vs read-only, inline validation on blur, field grouping), **Audit Timeline** (Riwayat Perubahan — vertical timeline approval history dengan rejection reason emphasized), **Disclaimer Banner** (severity warning, set ekspektasi pre-aksi), **Locked Row Action** (disabled icon + tooltip untuk pending-approval lock). Divalidasi via skill ui-ux-pro-max (forms/UX domain). Dipicu oleh modul baru M13. |
