# Replit Handoff — M7 Delta: Admin Customer Management & Add Customer
**Version:** 1.1  
**Date:** 2026-05-06  
**Parent handoff:** `m7-customer-authentication.md` (v1.0 → v1.1)

---

## What This File Is

File ini **hanya memuat perubahan baru** yang ditambahkan di v1.1. Gunakan file ini jika M7 v1.0 sudah selesai diimplementasi di Replit dan kamu hanya perlu menerapkan tambahan berikut.

Jika M7 belum diimplementasi sama sekali, gunakan `m7-customer-authentication.md` (full handoff).

---

## Ringkasan Perubahan v1.1

| Area | Perubahan |
|------|-----------|
| DB: `customers` | Tambah kolom `registration_source VARCHAR(20) DEFAULT 'SELF'` |
| DB: `customer_documents` | Tambah kolom `doc_label VARCHAR(100)`, `is_custom BOOLEAN DEFAULT FALSE`, `uploaded_by VARCHAR(20) DEFAULT 'CUSTOMER'` |
| Go model | Field baru di `Customer` dan `CustomerDocument` |
| Repository | Tambah `FindAll(statusFilter)`, `CreateByAdmin` |
| Admin API | Tambah `GET /api/admin/customers`, `POST /api/admin/customers`; rename `activate` → `approve`; tambah email notif di approve dan reject |
| Frontend Admin | Refactor `PendingCustomersPage` → `CustomerManagementPage` (full list + filter + detail page); buat `AddCustomerPage` baru |
| Email | Tambah 3 email notification: approval, rejection, welcome (admin-add) |
| Env vars | Tambah `FRONTEND_URL`, `SMTP_*` |

---

## Step A: Database Migration (Alter Tables)

Buat 2 migration file baru. Jangan ubah migration lama.

### Migration A1 — `migrations/XXXXXX_alter_customers_add_registration_source.up.sql`

```sql
ALTER TABLE customers
    ADD COLUMN registration_source VARCHAR(20) NOT NULL DEFAULT 'SELF';
-- SELF = customer self-registered via portal
-- ADMIN = created directly by Admin from dashboard

CREATE INDEX idx_customers_registration_source ON customers(registration_source);
```

```sql
-- down
ALTER TABLE customers DROP COLUMN registration_source;
```

### Migration A2 — `migrations/XXXXXX_alter_customer_documents_add_custom_fields.up.sql`

```sql
ALTER TABLE customer_documents
    ADD COLUMN doc_label   VARCHAR(100),
    ADD COLUMN is_custom   BOOLEAN NOT NULL DEFAULT FALSE,
    ADD COLUMN uploaded_by VARCHAR(20) NOT NULL DEFAULT 'CUSTOMER';
-- doc_label: nama bebas untuk custom documents (NULL untuk dokumen standar)
-- is_custom: TRUE untuk dokumen yang ditambahkan Admin di luar 3 dokumen standar
-- uploaded_by: 'CUSTOMER' | 'ADMIN'

CREATE INDEX idx_customer_documents_is_custom ON customer_documents(is_custom);
```

```sql
-- down
ALTER TABLE customer_documents
    DROP COLUMN doc_label,
    DROP COLUMN is_custom,
    DROP COLUMN uploaded_by;
```

Run: `golang-migrate -path migrations -database $DATABASE_URL up`

---

## Step B: Update Go Models

File: `internal/customer/model.go`

Tambah field berikut ke struct yang sudah ada:

```go
// Di struct Customer — tambah setelah field Status:
RegistrationSource string `gorm:"not null;default:'SELF'"` // SELF | ADMIN

// Di struct CustomerDocument — tambah setelah field DocType:
DocLabel   *string // nil for standard docs; set by Admin for custom docs
IsCustom   bool    `gorm:"not null;default:false"`
UploadedBy string  `gorm:"not null;default:'CUSTOMER'"` // CUSTOMER | ADMIN
```

---

## Step C: Update Repository

File: `internal/customer/repository.go`

Tambah 2 method baru ke interface dan implementasi:

```go
// Tambah ke interface Repository:
FindAll(ctx context.Context, statusFilter string) ([]Customer, error)
CreateByAdmin(ctx context.Context, customer *Customer, standardDocs []CustomerDocument, customDocs []CustomerDocument) error
```

### `FindAll` implementation

```go
func (r *repo) FindAll(ctx context.Context, statusFilter string) ([]Customer, error) {
    var customers []Customer
    q := r.db.WithContext(ctx).Order("created_at DESC")
    if statusFilter != "" {
        q = q.Where("status = ?", statusFilter)
    }
    return customers, q.Find(&customers).Error
}
```

### `CreateByAdmin` implementation

```go
func (r *repo) CreateByAdmin(ctx context.Context, customer *Customer, standardDocs []CustomerDocument, customDocs []CustomerDocument) error {
    return r.db.WithContext(ctx).Transaction(func(tx *gorm.DB) error {
        if err := tx.Create(customer).Error; err != nil {
            return err
        }
        allDocs := append(standardDocs, customDocs...)
        for i := range allDocs {
            allDocs[i].CustomerID = customer.ID
        }
        if len(allDocs) > 0 {
            return tx.Create(&allDocs).Error
        }
        return nil
    })
}
```

---

## Step D: Add Email Notification Helper

File: `internal/notification/email.go` (buat baru jika belum ada)

```go
type EmailService interface {
    SendApprovalEmail(ctx context.Context, toEmail, customerName, customerCode, loginURL string) error
    SendRejectionEmail(ctx context.Context, toEmail, customerName, reason string) error
    SendWelcomeEmail(ctx context.Context, toEmail, customerName, customerCode, password, loginURL string) error
}
```

Implementasi menggunakan `net/smtp` atau library SMTP pilihan. Config dari env vars:

```
SMTP_HOST=
SMTP_PORT=587
SMTP_USER=
SMTP_PASSWORD=
SMTP_FROM=noreply@lps.example.com
FRONTEND_URL=https://lps.example.com
```

**Template — Approval email:**
- Subject: `Akun LPS Portal Anda Telah Diaktifkan`
- Body:
  ```
  Yth. [customerName],

  Akun LPS Portal Anda telah diaktifkan oleh Admin.
  Customer Code Anda: [customerCode]

  Silakan login di: [loginURL]

  Terima kasih,
  Tim LPS Platform
  ```

**Template — Rejection email:**
- Subject: `Registrasi LPS Portal Anda Ditolak`
- Body:
  ```
  Yth. [customerName],

  Mohon maaf, registrasi akun LPS Portal Anda ditolak.
  [Jika ada alasan: Alasan: [reason]]

  Untuk informasi lebih lanjut, hubungi Admin LPS.

  Terima kasih,
  Tim LPS Platform
  ```

**Template — Welcome email (Admin-created account):**
- Subject: `Akun LPS Portal Anda Telah Dibuat`
- Body:
  ```
  Yth. [customerName],

  Akun LPS Portal Anda telah dibuat oleh Admin.
  Customer Code: [customerCode]
  Email login: [email]
  Password awal: [password]

  Silakan login di: [loginURL]
  Disarankan untuk mengganti password setelah login pertama.

  Terima kasih,
  Tim LPS Platform
  ```

---

## Step E: Update Admin Handler

File: `internal/admin/customer_handler.go`

### E1 — Rename endpoint activate → approve

Ubah route registration dari:
```go
adminGroup.PUT("/customers/:id/activate", handler.ActivateCustomer)
```
menjadi:
```go
adminGroup.PUT("/customers/:id/approve", handler.ApproveCustomer)
```

> ⚠️ Jika endpoint lama (`/activate`) masih digunakan oleh UI yang sudah live, bisa buat alias sementara, lalu hapus setelah UI diupdate.

Ubah nama handler function dari `ActivateCustomer` → `ApproveCustomer`. Logic tetap sama, tambah:
- Setelah update status → `async: emailSvc.SendApprovalEmail(...)`.

### E2 — Update reject handler

Tambah ke `RejectCustomer` handler setelah update status:
```go
go func() {
    _ = emailSvc.SendRejectionEmail(ctx, customer.Email, customer.CustomerName, reason)
}()
```

### E3 — Tambah GET /api/admin/customers

```go
func (h *Handler) ListAllCustomers(c echo.Context) error {
    statusFilter := c.QueryParam("status") // optional
    customers, err := h.repo.FindAll(c.Request().Context(), statusFilter)
    if err != nil {
        return c.JSON(500, map[string]string{"error": "Internal server error"})
    }
    return c.JSON(200, customers)
}
```

Register route:
```go
adminGroup.GET("/customers", handler.ListAllCustomers)
```

### E4 — Tambah POST /api/admin/customers

```go
func (h *Handler) AddCustomerByAdmin(c echo.Context) error {
    // 1. Parse multipart form
    // 2. Validate required fields (same as register handler)
    // 3. Check unique email + NPWP
    // 4. Hash password
    // 5. Save standard doc files → []CustomerDocument (is_custom=false, uploaded_by="ADMIN")
    // 6. Parse custom_doc_label[], custom_doc_file[] arrays
    //    For each index i:
    //      - save file
    //      - build CustomerDocument{DocType: label, DocLabel: &label, IsCustom: true, UploadedBy: "ADMIN"}
    // 7. Generate customer_code immediately (same helper used in approve flow)
    // 8. Build Customer{Status: "ACTIVE", RegistrationSource: "ADMIN", ActivatedBy: &adminID, ActivatedAt: &now}
    // 9. repo.CreateByAdmin(ctx, &customer, standardDocs, customDocs)
    // 10. async: STS sync
    // 11. async: emailSvc.SendWelcomeEmail(...)
    // 12. Return 201
}
```

Register route:
```go
adminGroup.POST("/customers", handler.AddCustomerByAdmin)
```

---

## Step F: Frontend — Refactor Admin Pages

### F1 — Refactor PendingCustomersPage → CustomerManagementPage

**Rename/replace** `src/pages/admin/PendingCustomersPage.tsx` dengan `CustomerManagementPage.tsx`.

**Route update:** Ubah route dari `/admin/customers/pending` → `/admin/customers` di router.

Jika route lama masih ada di navigasi/sidebar, update ke `/admin/customers`.

**Komponen:**

```tsx
// State
const [statusFilter, setStatusFilter] = useState<string>("")
// "" = all, "PENDING_VALIDATION", "ACTIVE", "REJECTED"

// Query
const { data: customers } = useQuery({
    queryKey: ["admin-customers", statusFilter],
    queryFn: () => fetch(`/api/admin/customers?status=${statusFilter}`).then(r => r.json())
})
```

**Tab/Filter bar:**
```tsx
const tabs = [
  { label: "All", value: "" },
  { label: "Pending", value: "PENDING_VALIDATION" },
  { label: "Active", value: "ACTIVE" },
  { label: "Rejected", value: "REJECTED" },
]
```

**Tambah tombol di header:**
```tsx
<Button onClick={() => navigate("/admin/customers/new")}>+ Tambah Customer</Button>
```

**Table columns:**
```
No | Customer Name | NPWP | PIC Name | Email | Phone | Status | Source | Registered At | Actions
```

- Status: `<Badge>` dengan warna sesuai status
- Source: tampilkan "Self" jika `registration_source === "SELF"`, "Admin" jika `"ADMIN"`
- Actions: tombol "Detail" → navigate to `/admin/customers/:id`

### F2 — Buat CustomerDetailPage

File: `src/pages/admin/CustomerDetailPage.tsx`  
Route: `/admin/customers/:id`

```tsx
// Fetch
const { data: customer } = useQuery({
    queryKey: ["customer-detail", id],
    queryFn: () => fetch(`/api/admin/customers/${id}`).then(r => r.json())
})

// Mutations
const approveMutation = useMutation({
    mutationFn: () => fetch(`/api/admin/customers/${id}/approve`, { method: "PUT" }),
    onSuccess: () => { navigate("/admin/customers"); toast.success(`Akun diaktifkan. Customer Code: ${data.customer_code}`) }
})

const rejectMutation = useMutation({
    mutationFn: (reason: string) => fetch(`/api/admin/customers/${id}/reject`, {
        method: "PUT",
        body: JSON.stringify({ reason })
    }),
    onSuccess: () => { navigate("/admin/customers"); toast.success("Akun ditolak") }
})
```

**Layout:**
- Breadcrumb: Customer Management > Detail
- Card "Company Information": tampilkan semua field (read-only)
- Card "Documents": list semua dokumen
  - Setiap dokumen: nama file, badge "Standard" (gray) atau "Custom" (blue), label (untuk custom), tanggal upload, tombol "Download"
- Card "Actions" (hanya tampil jika `status === "PENDING_VALIDATION"`):
  - Tombol "Approve" (green) → confirm dialog sederhana → call `approveMutation`
  - Tombol "Reject" (red) → modal dengan textarea alasan → call `rejectMutation`

### F3 — Buat AddCustomerPage

File: `src/pages/admin/AddCustomerPage.tsx`  
Route: `/admin/customers/new`

Gunakan `react-hook-form` + `zod` + `useFieldArray` untuk custom documents.

```tsx
const { fields, append, remove } = useFieldArray({ control, name: "customDocs" })

// Schema zod untuk custom doc item:
// { label: string (min 1), file: FileList, description: string?, issueDate: string?, expiryDate: string? }
```

**On submit — build FormData:**
```tsx
const buildFormData = (values: FormValues): FormData => {
    const fd = new FormData()
    // Standard fields
    fd.append("customer_name", values.customerName)
    // ... append semua field standar ...

    // Standard docs
    fd.append("doc_npwp_file", values.docNpwp.file[0])
    // ... dst untuk NIP dan Company Profile ...

    // Custom docs
    values.customDocs.forEach((doc, i) => {
        fd.append(`custom_doc_label[${i}]`, doc.label)
        fd.append(`custom_doc_file[${i}]`, doc.file[0])
        if (doc.description) fd.append(`custom_doc_description[${i}]`, doc.description)
        if (doc.issueDate) fd.append(`custom_doc_issue_date[${i}]`, doc.issueDate)
        if (doc.expiryDate) fd.append(`custom_doc_expiry_date[${i}]`, doc.expiryDate)
    })
    return fd
}
```

**Section 3 — Custom Documents UI:**
```tsx
<div className="space-y-4">
    <div className="flex justify-between items-center">
        <h3>Custom Documents</h3>
        <span className="text-sm text-muted-foreground">Optional</span>
    </div>
    {fields.map((field, i) => (
        <Card key={field.id} className="p-4">
            <div className="flex justify-between">
                <span className="font-medium">Custom Document {i + 1}</span>
                <Button variant="ghost" size="sm" onClick={() => remove(i)}>×</Button>
            </div>
            <Input placeholder="Document Name / Label (e.g. SIUP, Akta Pendirian)" {...register(`customDocs.${i}.label`)} />
            {/* File upload, Description, Issue Date, Expiry Date */}
        </Card>
    ))}
    <Button variant="outline" type="button" onClick={() => append({ label: "", file: undefined })}>
        + Tambah Dokumen
    </Button>
</div>
```

---

## Step G: Update Router

File: `src/App.tsx` atau file router utama

```tsx
// Ubah/tambah route berikut:
<Route path="/admin/customers" component={CustomerManagementPage} />
<Route path="/admin/customers/new" component={AddCustomerPage} />
<Route path="/admin/customers/:id" component={CustomerDetailPage} />

// Route lama ini bisa dihapus jika sudah tidak dibutuhkan:
// <Route path="/admin/customers/pending" component={PendingCustomersPage} />
```

---

## Step H: Update Environment Variables

Tambah ke `.env` / Replit Secrets:

```
FRONTEND_URL=https://your-replit-url.replit.app
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASSWORD=your-app-password
SMTP_FROM=noreply@lps.example.com
```

---

## Acceptance Checklist v1.1

### Admin Customer Management
- [ ] Route `/admin/customers` menampilkan halaman Customer Management (bukan hanya pending)
- [ ] Tab/filter All / Pending / Active / Rejected berfungsi dan memfilter tabel
- [ ] Tabel menampilkan kolom Status (badge berwarna) dan Source (Self / Admin)
- [ ] Tombol "Tambah Customer" di header mengarah ke `/admin/customers/new`
- [ ] Baris tabel memiliki tombol "Detail" → navigate ke `/admin/customers/:id`

### Admin Customer Approval
- [ ] Halaman detail customer menampilkan semua data Company Information (read-only)
- [ ] Dokumen ditampilkan dengan badge "Standard" atau "Custom", dan doc_label untuk custom doc
- [ ] Tombol "Download" berfungsi untuk setiap dokumen
- [ ] Tombol Approve dan Reject **hanya tampil** jika status = PENDING_VALIDATION
- [ ] Klik Approve → Customer Code di-generate → status ACTIVE → toast sukses dengan Customer Code
- [ ] Email aktivasi terkirim ke email customer (berisi Customer Code dan URL login)
- [ ] Klik Reject → modal reason muncul → submit → status REJECTED → toast "Akun ditolak"
- [ ] Email penolakan terkirim ke email customer (dengan alasan jika diisi)
- [ ] Jika status bukan PENDING_VALIDATION, tombol Approve/Reject tidak muncul

### Admin Add Customer
- [ ] Form `/admin/customers/new` menampilkan semua field standard + 3 dokumen standar wajib
- [ ] Form memiliki section "Custom Documents" dengan tombol "+ Tambah Dokumen"
- [ ] Setiap custom doc card memiliki field: Label (wajib), File (wajib), Description, Issue Date, Expiry Date
- [ ] Tombol "×" pada custom doc card menghapus card tersebut
- [ ] Bisa menambah lebih dari 1 custom document
- [ ] Submit tanpa custom doc → berhasil (custom docs opsional)
- [ ] Customer yang berhasil dibuat langsung berstatus ACTIVE
- [ ] `customer_code` langsung di-generate saat save (format CUST-YYYYMM-XXXXX)
- [ ] `registration_source = "ADMIN"` tersimpan di DB
- [ ] Custom docs tersimpan dengan `is_custom = true` dan `doc_label` yang benar
- [ ] Welcome email terkirim ke customer dengan email, password awal, Customer Code, URL login
- [ ] STS Platform sync dipanggil setelah add customer berhasil

### General
- [ ] Semua aksi Admin (approve, reject, add customer) tercatat di audit log
- [ ] DB migration berjalan tanpa error (2 alter table migration baru)
- [ ] Tidak ada breaking change pada fitur v1.0 (self-registration, login, JWT guard)
