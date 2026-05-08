# M10 — Customer Dashboard & Monitoring

## Overview
Provides the customer-facing landing dashboard and all monitoring views: nomination summary, active voyage tracking with AIS vessel position, and weather & alert access. This is a read-only aggregation module — it does not create or modify any business data.

## Boundaries
**In scope:**
- Customer Portal landing dashboard (post-login)
- Nomination Status list page (all nominations including Drafts)
- Active voyage tracking (view-only, AIS vessel position)
- Weather real-time data and alert history (view-only)
- Document Master: repositori dokumen pribadi customer, append-only (upload saja; tidak bisa hapus/edit); terintegrasi dengan form nominasi sebagai sumber dokumen alternatif

**Out of scope:**
- Any write operations (handled by M8, M9)
- Internal LPS operational dashboard (M11)
- Weather alert broadcasting to vessels (M5 operator function)

## Dependencies
| Dependency | Source | Data Provided |
|-----------|--------|--------------|
| M7 Authentication | LPS internal | Customer session and identity |
| M8 / M9 Nomination data | LPS DB | Nomination list, statuses, EPB |
| LPS AIS module (M1) | LPS internal API | Vessel position for active voyages |
| Weather API / M5 | LPS internal API | Real-time weather, alert history |
| STS Platform | Via LPS cache | Voyage status (view-only) |

## Downstream
None — this is a terminal read-only module.

## Key Decisions
- Dashboard data is assembled server-side and served via a single `/api/customer/dashboard` endpoint to minimize frontend round-trips.
- AIS vessel position is only shown for voyages with status `PAYMENT_CONFIRMED` (voyage started).
- Weather data shown to customers is the same data as the operator view but limited to current conditions and the last 24h of alerts.
- Customer sees only their own nominations and voyages — no cross-customer data.
- **Status scope (BRD v3.1):** M10 hanya membaca status di tabel `nominations`. Status payment (UNPAID, PENDING_REVIEW, PAYMENT_REJECT, PAID) adalah milik tabel `epb_payments` (M9b) — tidak tampil di daftar nominasi M10. Nominasi berstatus `EPB_CONFIRMATION_SUBMITTED` ditampilkan dengan label "Konfirmasi EPB Terkirim" dan shortcut link ke halaman EPB & Invoice M9b.
