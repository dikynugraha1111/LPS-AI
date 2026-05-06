# Replit Handoff — Customer Portal UI Prototype (M7–M10)

## Purpose
This is a **frontend-only UI prototype** of the LPS Customer Portal. There is no backend, no database, and no real API calls. All data is mocked. The goal is to demonstrate the complete customer journey to the client with a fully clickable, navigable interface.

## Tech Stack (Frontend Only)
- React 19, Vite, TailwindCSS v4, shadcn/ui, wouter (routing), TanStack React Query, Framer Motion

**Do NOT create any backend, Go files, database migrations, or API endpoints.**

---

## Step 1: Project Setup

If starting a fresh Vite project:
```bash
npm create vite@latest lps-customer-portal -- --template react-ts
cd lps-customer-portal
npm install
npm install tailwindcss @tailwindcss/vite
npm install wouter
npm install @tanstack/react-query
npm install framer-motion
npm install lucide-react
npm install react-leaflet leaflet
npm install -D @types/leaflet
```

Install shadcn/ui:
```bash
npx shadcn@latest init
npx shadcn@latest add button input label card badge table tabs dialog form select toast
```

---

## Step 2: Mock Data

Create file: `src/data/mockData.ts`

```typescript
export type NominationStatus =
  | 'DRAFT'
  | 'PENDING'
  | 'APPROVED'
  | 'NEED_REVISION'
  | 'WAITING_PAYMENT_VERIFICATION'
  | 'PAYMENT_CONFIRMED'
  | 'PAYMENT_REJECTED';

export const STATUS_LABELS: Record<NominationStatus, string> = {
  DRAFT: 'Draft',
  PENDING: 'Menunggu proses di STS Platform',
  APPROVED: 'Nominasi Disetujui',
  NEED_REVISION: 'Perlu Revisi',
  WAITING_PAYMENT_VERIFICATION: 'Menunggu Verifikasi Pembayaran',
  PAYMENT_CONFIRMED: 'Pembayaran Dikonfirmasi',
  PAYMENT_REJECTED: 'Pembayaran Ditolak',
};

export const STATUS_COLORS: Record<NominationStatus, string> = {
  DRAFT: 'bg-gray-100 text-gray-600',
  PENDING: 'bg-blue-100 text-blue-700',
  APPROVED: 'bg-green-100 text-green-700',
  NEED_REVISION: 'bg-orange-100 text-orange-700',
  WAITING_PAYMENT_VERIFICATION: 'bg-purple-100 text-purple-700',
  PAYMENT_CONFIRMED: 'bg-emerald-100 text-emerald-700',
  PAYMENT_REJECTED: 'bg-red-100 text-red-700',
};

export interface Nomination {
  id: string;
  nominationNumber: string | null;
  vesselName: string;
  eta: string;
  cargoType: string;
  cargoQuantity: number;
  charterer: string;
  estimatedBargeCount: number;
  nomorPkk: string;
  status: NominationStatus;
  createdAt: string;
  updatedAt: string;
  // APPROVED fields
  anchorPoint?: string;
  etb?: string;
  estimatedDurationHours?: number;
  epbNumber?: string;
  epbAmount?: number;
  epbDueDate?: string;
  // NEED_REVISION
  revisionNotes?: string;
  // PAYMENT_REJECTED
  paymentRejectionReason?: string;
}

export const mockNominations: Nomination[] = [
  {
    id: '1',
    nominationNumber: 'NOM-20260425-0012',
    vesselName: 'MV Kalimantan Jaya',
    eta: '2026-05-03T08:00:00+07:00',
    cargoType: 'Batubara',
    cargoQuantity: 45000,
    charterer: 'PT Sinar Bara Energi',
    estimatedBargeCount: 4,
    nomorPkk: 'PKK-2026-0089',
    status: 'APPROVED',
    createdAt: '2026-04-22T10:30:00+07:00',
    updatedAt: '2026-04-24T14:00:00+07:00',
    anchorPoint: 'AP-2',
    etb: '2026-05-03T10:00:00+07:00',
    estimatedDurationHours: 48,
    epbNumber: 'EPB-2026-0156',
    epbAmount: 18500000,
    epbDueDate: '2026-05-01T23:59:59+07:00',
  },
  {
    id: '2',
    nominationNumber: 'NOM-20260424-0008',
    vesselName: 'MV Borneo Star',
    eta: '2026-05-10T06:00:00+07:00',
    cargoType: 'Batubara',
    cargoQuantity: 52000,
    charterer: 'PT Mega Energi Nusantara',
    estimatedBargeCount: 5,
    nomorPkk: 'PKK-2026-0091',
    status: 'NEED_REVISION',
    createdAt: '2026-04-21T09:00:00+07:00',
    updatedAt: '2026-04-23T16:30:00+07:00',
    revisionNotes: 'Dokumen Rencana Kerja tidak lengkap. Harap lampirkan rencana operasional lengkap termasuk estimasi waktu bongkar muat.',
  },
  {
    id: '3',
    nominationNumber: 'NOM-20260423-0003',
    vesselName: 'MV Mahakam Glory',
    eta: '2026-04-28T07:00:00+07:00',
    cargoType: 'Batubara',
    cargoQuantity: 38000,
    charterer: 'PT Borneo Coal Resources',
    estimatedBargeCount: 3,
    nomorPkk: 'PKK-2026-0077',
    status: 'PAYMENT_CONFIRMED',
    createdAt: '2026-04-18T11:00:00+07:00',
    updatedAt: '2026-04-22T09:00:00+07:00',
    anchorPoint: 'AP-1',
    etb: '2026-04-28T09:00:00+07:00',
    estimatedDurationHours: 36,
    epbNumber: 'EPB-2026-0143',
    epbAmount: 14200000,
  },
  {
    id: '4',
    nominationNumber: 'NOM-20260425-0015',
    vesselName: 'MV Selat Makassar',
    eta: '2026-05-15T09:00:00+07:00',
    cargoType: 'Batubara',
    cargoQuantity: 60000,
    charterer: 'PT Anugrah Bara Kaltim',
    estimatedBargeCount: 6,
    nomorPkk: 'PKK-2026-0095',
    status: 'PENDING',
    createdAt: '2026-04-25T08:00:00+07:00',
    updatedAt: '2026-04-25T08:05:00+07:00',
  },
  {
    id: '5',
    nominationNumber: null,
    vesselName: 'MV Balikpapan Express',
    eta: '2026-05-20T10:00:00+07:00',
    cargoType: 'Batubara',
    cargoQuantity: 42000,
    charterer: 'PT Delta Karya Energi',
    estimatedBargeCount: 4,
    nomorPkk: '',
    status: 'DRAFT',
    createdAt: '2026-04-25T15:00:00+07:00',
    updatedAt: '2026-04-25T15:00:00+07:00',
  },
  {
    id: '6',
    nominationNumber: 'NOM-20260420-0002',
    vesselName: 'MV Kutai Mandiri',
    eta: '2026-04-26T08:00:00+07:00',
    cargoType: 'Batubara',
    cargoQuantity: 35000,
    charterer: 'PT Surya Energi Prima',
    estimatedBargeCount: 3,
    nomorPkk: 'PKK-2026-0068',
    status: 'WAITING_PAYMENT_VERIFICATION',
    createdAt: '2026-04-17T10:00:00+07:00',
    updatedAt: '2026-04-25T11:00:00+07:00',
    anchorPoint: 'AP-3',
    etb: '2026-04-26T10:00:00+07:00',
    estimatedDurationHours: 30,
    epbNumber: 'EPB-2026-0139',
    epbAmount: 12800000,
    epbDueDate: '2026-04-25T23:59:59+07:00',
  },
];

export const mockWeather = {
  waveHeightM: 1.6,
  windSpeedKmh: 18,
  visibilityKm: 12,
  condition: 'NORMAL' as 'NORMAL' | 'WARNING' | 'CRITICAL',
  updatedAt: '2026-04-25T22:45:00+07:00',
  alerts: [
    {
      id: 'a1',
      level: 'WARNING',
      message: 'Tinggi gelombang diprediksi meningkat hingga 2.8m pada pukul 14.00-18.00 WIB',
      createdAt: '2026-04-25T10:00:00+07:00',
      resolvedAt: '2026-04-25T18:30:00+07:00',
    },
    {
      id: 'a2',
      level: 'WARNING',
      message: 'Kecepatan angin meningkat, waspadai kondisi di area Anchor Point AP-4 dan AP-5',
      createdAt: '2026-04-25T19:00:00+07:00',
      resolvedAt: null,
    },
  ],
};

export interface DocumentMasterItem {
  id: string;
  fileName: string;
  mimeType: string;
  fileSizeBytes: number;
  uploadedAt: string;
}

export const mockDocuments: DocumentMasterItem[] = [
  { id: 'd1', fileName: 'NPWP_PT_Nusantara.pdf', mimeType: 'application/pdf', fileSizeBytes: 204800, uploadedAt: '2026-04-10T09:00:00+07:00' },
  { id: 'd2', fileName: 'Rencana_Kerja_MV_Kalimantan_Jaya.pdf', mimeType: 'application/pdf', fileSizeBytes: 512000, uploadedAt: '2026-04-20T14:30:00+07:00' },
  { id: 'd3', fileName: 'Shipping_Instruction_Apr2026.pdf', mimeType: 'application/pdf', fileSizeBytes: 153600, uploadedAt: '2026-04-21T08:00:00+07:00' },
  { id: 'd4', fileName: 'Surat_Penunjukan_PBM.pdf', mimeType: 'application/pdf', fileSizeBytes: 98304, uploadedAt: '2026-04-22T11:15:00+07:00' },
];

export const mockAISPosition = {
  lat: -3.7823,
  lng: 115.9512,
  speedKnots: 3.8,
  heading: 132,
  lastUpdated: '2026-04-25T22:50:00+07:00',
};

export const mockCustomer = {
  customerCode: 'CUST-202604-00007',
  customerName: 'PT Nusantara Bulk Shipping',
  type: 'Cargo Owner', // always defaulted by system
  picName: 'Budi Santoso',
  email: 'budi.santoso@nusantarabulk.co.id',
  phone: '+6281234567890',
  address: 'Jl. Industri Pelabuhan No. 12, Balikpapan, Kalimantan Timur',
};
```

---

## Step 3: Routing Setup

File: `src/App.tsx`

```tsx
import { Route, Switch, Redirect } from 'wouter';

export default function App() {
  return (
    <Switch>
      {/* Public */}
      <Route path="/customer/login" component={LoginPage} />
      <Route path="/register" component={RegisterPage} />
      <Route path="/register/success" component={RegisterSuccessPage} />

      {/* Customer Portal */}
      <Route path="/customer/dashboard" component={DashboardPage} />
      <Route path="/customer/nominations/new" component={NominationFormPage} />
      <Route path="/customer/nominations/:id/revise" component={NominationRevisionPage} />
      <Route path="/customer/nominations/:id" component={NominationStatusPage} />
      <Route path="/customer/nominations" component={NominationListPage} />
      <Route path="/customer/documents" component={DocumentMasterPage} />
      <Route path="/customer/voyages/:id" component={VoyageTrackingPage} />
      <Route path="/customer/weather" component={WeatherPage} />

      {/* Default redirect */}
      <Route path="/">
        <Redirect to="/customer/login" />
      </Route>
    </Switch>
  );
}
```

---

## Step 4: Customer Layout (Sidebar + Navigation)

File: `src/components/CustomerLayout.tsx`

```tsx
import { Link, useLocation } from 'wouter';
import { LayoutDashboard, FileText, FolderOpen, Cloud, LogOut } from 'lucide-react';
import { mockCustomer } from '../data/mockData';

const navItems = [
  { href: '/customer/dashboard', label: 'Dashboard', icon: LayoutDashboard },
  { href: '/customer/nominations', label: 'Nominasi', icon: FileText },
  { href: '/customer/documents', label: 'Document Master', icon: FolderOpen },
  { href: '/customer/weather', label: 'Cuaca & Alert', icon: Cloud },
];

export function CustomerLayout({ children }: { children: React.ReactNode }) {
  const [location, navigate] = useLocation();
  return (
    <div className="flex h-screen bg-gray-50">
      {/* Sidebar */}
      <aside className="w-64 bg-white border-r flex flex-col">
        <div className="p-6 border-b">
          <div className="font-bold text-lg text-blue-700">LPS Portal</div>
          <div className="text-sm text-gray-500 mt-1">{mockCustomer.customerName}</div>
        </div>
        <nav className="flex-1 p-4 space-y-1">
          {navItems.map(({ href, label, icon: Icon }) => (
            <Link key={href} href={href}>
              <a className={`flex items-center gap-3 px-3 py-2 rounded-lg text-sm font-medium transition-colors
                ${location.startsWith(href)
                  ? 'bg-blue-50 text-blue-700'
                  : 'text-gray-600 hover:bg-gray-100'}`}>
                <Icon className="w-4 h-4" />
                {label}
              </a>
            </Link>
          ))}
        </nav>
        <div className="p-4 border-t">
          <button
            onClick={() => { localStorage.removeItem('customer_token'); navigate('/customer/login'); }}
            className="flex items-center gap-3 px-3 py-2 w-full rounded-lg text-sm text-red-600 hover:bg-red-50">
            <LogOut className="w-4 h-4" />
            Keluar
          </button>
        </div>
      </aside>
      {/* Main Content */}
      <main className="flex-1 overflow-y-auto p-8">
        {children}
      </main>
    </div>
  );
}
```

Wrap all `/customer/*` pages with `<CustomerLayout>`.

---

## Step 5: Login Page

File: `src/pages/LoginPage.tsx`

- Show form: Email, Password fields (use shadcn Input)
- "Login" button → navigate to `/customer/dashboard` (no validation needed for prototype)
- "Belum punya akun? Daftar" link → navigate to `/register`
- Pre-fill email with `budi.santoso@nusantarabulk.co.id` for easy demo

---

## Step 6: Register Page

File: `src/pages/RegisterPage.tsx`

Header:
- "← Back to Login" link top-left → `/login`
- Ship icon + "Customer Registration" title + subtitle "Register your company to access the STS Platform"

**Section 1 — Company Information** (Card):
- Customer Code: Input disabled, placeholder "Auto-generated on approval", helper text "Auto-generated on approval"
- Customer Name *: Input, placeholder "Enter company name"
- NPWP *: Input, placeholder "XX.XXX.XXX.X-XXX.XXX"
- PIC Name *: Input, placeholder "Enter PIC name"
- Phone Number *: Input, placeholder "Enter phone number"
- Email *: Input, placeholder "Enter email address"
- Address: Textarea, placeholder "Enter company address"
- Note: Textarea, placeholder "Additional notes (optional)"

Layout: Customer Code + Customer Name (2-col). NPWP + PIC Name (2-col). Phone Number + Email (2-col). Address (full-width). Note (full-width).

> Tipe Pelanggan tidak ditampilkan — sistem default ke "Cargo Owner".

**Section 2 — Required Documents** (Card, title "Required Documents", subtitle "All documents below are required for registration"):

Render 3 document cards: **1. NPWP**, **2. NIP**, **3. Company Profile**. Each card:
- Document title with red asterisk
- File * → "Choose file" button with upload icon. On click: trigger file input (no actual upload). Show mock filename on "select".
- Description → Input, placeholder "Optional description"
- Issue Date + Expiry Date → side-by-side date inputs (placeholder "dd/mm/yyyy")

Footer buttons:
- "Cancel" (outline) → navigate to `/login`
- "Submit Registration" (primary navy blue) → navigate to `/register/success`

File: `src/pages/RegisterSuccessPage.tsx`
- Show success icon + "Registrasi Berhasil" title
- Message: "Akun Anda sedang menunggu validasi Admin. Kami akan menghubungi Anda melalui email setelah akun diaktifkan."
- "Kembali ke Login" button → `/login`

---

## Step 7: Dashboard Page

File: `src/pages/customer/DashboardPage.tsx`

Use `mockNominations` and `mockWeather` from `mockData.ts`.

**Active Nominations** (status NOT in DRAFT, PAYMENT_CONFIRMED):
Filter `mockNominations` accordingly. Show in a table with Status badges.
Each row is clickable → navigate to `/customer/nominations/:id`.

**Active Voyages** (status = PAYMENT_CONFIRMED):
Show as cards with vessel name, anchor point, ETB. "Lacak Posisi" button → `/customer/voyages/:id`.

**Weather Widget**:
Show wave height, wind speed, condition badge (color-coded). "Lihat Detail Cuaca" → `/customer/weather`.

**"Buat Nominasi Baru" button** → `/customer/nominations/new`.

Use Framer Motion `motion.div` with `initial={{ opacity: 0, y: 20 }} animate={{ opacity: 1, y: 0 }}` for each section card.

---

## Step 8: Nomination List Page

File: `src/pages/customer/NominationListPage.tsx`

Show all `mockNominations`. Use shadcn `Tabs` for: Semua | Aktif | Draft | Selesai.

Filter logic (in-component, no API):
- **Aktif**: status NOT IN DRAFT, PAYMENT_CONFIRMED, SUBMIT_FAILED
- **Draft**: status = DRAFT
- **Selesai**: status = PAYMENT_CONFIRMED

Table columns: Nomor Nominasi, Nama Kapal, ETA, Status (badge), Tanggal Dibuat, Aksi (Lihat Detail button).
Clicking row or "Lihat Detail" → `/customer/nominations/:id`.

"Buat Nominasi Baru" button → `/customer/nominations/new`.

---

## Step 9: Nomination Status Page

File: `src/pages/customer/NominationStatusPage.tsx`

Read `:id` param from wouter. Find nomination in `mockNominations` by id.

**Status Banner** (full width, color-coded per STATUS_COLORS):
Display `STATUS_LABELS[nomination.status]`.

**Nomination Details section** (always visible):
Show: Nomor Nominasi, Nama Kapal, ETA, Jenis Muatan, Jumlah Muatan, Charterer, Estimasi Jumlah Barge, Nomor PKK.
Show uploaded document names as static text (mock: "rencana_kerja.pdf ✓", "shipping_instruction.pdf ✓", "surat_penunjukan_pbm.pdf ✓").

**Schedule & EPB section** (show only if status is APPROVED or later, and epbNumber exists):
Show: Anchor Point, ETB, Estimasi Durasi, Nomor EPB, Jumlah (formatted `Rp XX.XXX.XXX`), Batas Pembayaran.

**Payment Proof Upload section** (show only if status = APPROVED):
File input (accept PDF/JPG/PNG). "Upload Bukti Pembayaran" button.
On click: update the nomination status in local React state to `WAITING_PAYMENT_VERIFICATION` and show success toast.
(No actual file upload — just state change for demo.)

**Payment Proof Re-upload** (show only if status = PAYMENT_REJECTED):
Show rejection reason. File input. "Upload Ulang" button → same state change to WAITING_PAYMENT_VERIFICATION.

**Revision section** (show only if status = NEED_REVISION):
Show revision notes in an orange alert box.
"Revisi Nominasi" button → navigate to `/customer/nominations/:id/revise`.

> Note: Since this is a prototype, use React `useState` to track status changes within the page session. Changes do NOT persist between page navigations — that is acceptable for a client demo.

---

## Step 10: Nomination Form Page (New Nomination)

File: `src/pages/customer/NominationFormPage.tsx`
Route: `/customer/nominations/new`

Show all form fields: Vessel Name, ETA (date-time input), Cargo Type, Cargo Quantity (MT), Charterer, Estimasi Jumlah Barge, Nomor PKK.

Show document upload cards (3 cards: Rencana Kerja, Shipping Instruction, Surat Penunjukan PBM). Each card has a file input — on file select, show the file name with a green check. (No actual upload.)

Two buttons:
- "Simpan Draft" → navigate to `/customer/nominations` with a toast "Draft berhasil disimpan."
- "Submit Nominasi" → navigate to `/customer/nominations` with a toast "Nominasi berhasil disubmit! Nomor: NOM-20260425-0016."

---

## Step 11: Nomination Revision Page

File: `src/pages/customer/NominationRevisionPage.tsx`
Route: `/customer/nominations/:id/revise`

Pre-fill form with existing nomination data from `mockNominations`.
Show orange banner at top with `revisionNotes`.

All fields editable. "Re-Submit Nominasi" button → navigate to `/customer/nominations/:id` with toast "Nominasi berhasil di-resubmit." and update local status to PENDING.

---

## Step 12: Voyage Tracking Page

File: `src/pages/customer/VoyageTrackingPage.tsx`
Route: `/customer/voyages/:id`

Find nomination in `mockNominations` by id (must have status PAYMENT_CONFIRMED).

**Left panel**: Vessel Name, Anchor Point (AP-1), ETB, Estimasi Durasi, last AIS update timestamp.

**Right panel**: Leaflet map.

Leaflet map setup:
```tsx
import { MapContainer, TileLayer, Marker, Popup, Circle } from 'react-leaflet';
import 'leaflet/dist/leaflet.css';

// Fix default marker icon for Vite
import L from 'leaflet';
import markerIcon from 'leaflet/dist/images/marker-icon.png';
import markerShadow from 'leaflet/dist/images/marker-shadow.png';
L.Marker.prototype.options.icon = L.icon({ iconUrl: markerIcon, shadowUrl: markerShadow, iconAnchor: [12, 41] });

// In component:
<MapContainer center={[-3.7823, 115.9512]} zoom={12} style={{ height: '400px', width: '100%' }}>
  <TileLayer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" />
  <Marker position={[mockAISPosition.lat, mockAISPosition.lng]}>
    <Popup>MV Mahakam Glory<br/>Speed: 3.8 knots</Popup>
  </Marker>
  <Circle center={[-3.7700, 115.9400]} radius={500} color="blue" fillOpacity={0.1}>
    {/* Anchor Point AP-1 zone */}
  </Circle>
</MapContainer>
```

Show "Last updated: [timestamp]" below map.

---

## Step 13: Weather Page

File: `src/pages/customer/WeatherPage.tsx`
Route: `/customer/weather`

Use `mockWeather` data.

**Current Conditions card**:
- Large wave height display (e.g. "1.6 m")
- Wind speed, Visibility
- Condition badge: green=NORMAL, yellow=WARNING, red=CRITICAL
- "Last updated: [timestamp]"

**Alert History table** (use `mockWeather.alerts`):
Columns: Level (badge), Pesan, Waktu, Status (Berlangsung / Selesai [timestamp])
Ongoing alerts: highlight row with yellow background.

---

## Step 14: Status Badge Component

File: `src/components/StatusBadge.tsx`

```tsx
import { NominationStatus, STATUS_LABELS, STATUS_COLORS } from '../data/mockData';

export function StatusBadge({ status }: { status: NominationStatus }) {
  return (
    <span className={`inline-flex items-center px-2.5 py-0.5 rounded-full text-xs font-medium ${STATUS_COLORS[status]}`}>
      {STATUS_LABELS[status]}
    </span>
  );
}
```

Use this component consistently across Dashboard, NominationListPage, and NominationStatusPage.

---

## Step 15: Format Helpers

File: `src/lib/format.ts`

```typescript
export function formatRupiah(amount: number): string {
  return new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', minimumFractionDigits: 0 }).format(amount);
}

export function formatDate(iso: string): string {
  return new Date(iso).toLocaleDateString('id-ID', { day: 'numeric', month: 'long', year: 'numeric' });
}

export function formatDateTime(iso: string): string {
  return new Date(iso).toLocaleString('id-ID', { day: 'numeric', month: 'short', year: 'numeric', hour: '2-digit', minute: '2-digit', timeZone: 'Asia/Makassar' }) + ' WITA';
}
```

---

## Step 14 (Added): Document Master Page

File: `src/pages/customer/DocumentMasterPage.tsx`
Route: `/customer/documents`

Use `mockDocuments` from `mockData.ts`.

**Header:** "Document Master" + "Upload Dokumen" button (primary)

**Document table:**
| Column | Notes |
|--------|-------|
| File Name | Show with file icon (FileText from lucide-react) |
| Type | Badge: "PDF" / "JPG" / "PNG" derived from mimeType |
| Size | Format: "200 KB", "1.2 MB" |
| Uploaded At | formatDateTime() |
| Actions | "Download" button (outline, small) — no Edit, no Delete |

- No Delete button anywhere on this page.
- "Upload Dokumen" button → opens Dialog with file input, "Upload" confirms and shows success toast + appends mock document to local state.
- Empty state if no documents: "Belum ada dokumen."

**In Nomination Form (NominationFormPage.tsx):**

Update each document card (Rencana Kerja, Shipping Instruction, Surat Penunjukan PBM) to show two tabs:
- **"Upload Baru"** (default): standard file input, on select shows filename. 
- **"Pilih dari Document Master"**: button opens Dialog modal showing `mockDocuments` table with search input and "Pilih" button per row. On "Pilih": close modal, show selected filename in the card.

---

## Demo Flow for Client Presentation

Walk the client through this sequence:

1. `/register` → tampilkan form registrasi lengkap (Company Info + 3 dokumen wajib) → "Submit Registration" → halaman sukses
2. `/login` → click Login → arrives at Dashboard
3. Dashboard → see active nominations (Kalimantan Jaya: APPROVED, Borneo Star: NEED_REVISION) + weather widget
3. Click "MV Kalimantan Jaya" → Nomination Status page showing APPROVED with EPB and schedule → click "Upload Bukti Pembayaran" → status changes to WAITING_PAYMENT_VERIFICATION
4. Back to list → click "MV Borneo Star" → see NEED_REVISION with revision notes → click "Revisi Nominasi" → fill form → Re-Submit
5. Click "Buat Nominasi Baru" → fill nomination form → demo dual upload: Tab "Upload Baru" for one doc, Tab "Pilih dari Document Master" (opens modal with mockDocuments) for another → "Simpan Draft" → back to list
6. Click "Document Master" in sidebar → Document Master page → "Upload Dokumen" → show new doc in list (no delete button visible)
7. Click voyage for "MV Mahakam Glory" (PAYMENT_CONFIRMED) → see map with vessel position
8. Click "Cuaca & Alert" in sidebar → weather page with alert history

---

## Acceptance Checklist
- [ ] Login page → clicking Login navigates to /customer/dashboard
- [ ] Register page shows 2 sections: Company Information + Required Documents
- [ ] Company Information has Customer Code (disabled/read-only), Customer Name, Type (6 checkboxes), NPWP, PIC Name, Phone Number, Email, Address, Note
- [ ] Required Documents shows 3 document cards (NPWP, NIP, Company Profile) each with File upload, Description, Issue Date, Expiry Date
- [ ] Submitting register navigates to success page
- [ ] Dashboard shows active nominations table with correct status badges
- [ ] Dashboard shows active voyages and weather widget
- [ ] "Buat Nominasi Baru" navigates to nomination form
- [ ] Nomination list shows all nominations with filter tabs working
- [ ] Nomination status page shows correct content based on nomination status
- [ ] Upload Bukti Pembayaran button changes status to WAITING_PAYMENT_VERIFICATION (in-page state)
- [ ] NEED_REVISION page shows revision notes and "Revisi Nominasi" button
- [ ] New nomination form is fillable; Simpan Draft and Submit both work with toast
- [ ] Voyage tracking page shows Leaflet map with vessel marker
- [ ] Weather page shows conditions and alert history
- [ ] "Document Master" menu item visible in sidebar
- [ ] Document Master page shows mockDocuments table with Download button only (no Delete/Edit)
- [ ] "Upload Dokumen" button opens dialog; uploading adds document to local state list
- [ ] Nomination form document cards show two tabs: "Upload Baru" and "Pilih dari Document Master"
- [ ] "Pilih dari Document Master" opens modal with mockDocuments list and search
- [ ] Selecting a document from modal shows filename in the card
- [ ] Sidebar navigation works for all pages
- [ ] Logout navigates back to /login
- [ ] All text is in Bahasa Indonesia
