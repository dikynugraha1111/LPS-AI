# M7 — Customer Authentication & Onboarding

## Overview
Entry point for all Customer Portal functionality. Handles self-registration, admin validation, login, and initial sync of customer data to STS Platform. No other Customer Portal module (M8–M10) is accessible without a verified, authenticated customer session from this module.

## Boundaries
**In scope:**
- Customer self-registration form
- Admin review and activation/rejection of pending accounts
- Customer login and session management
- Automatic sync of customer data to STS Platform upon Admin activation

**Out of scope:**
- Nomination form or tracking (M8, M9)
- Dashboard and monitoring (M10)
- Internal LPS user/role management (M12)

## Dependencies
- **Upstream:** None — this is the entry point for all Customer Portal modules.
- **STS Platform API:** Customer account data is pushed to STS Platform after Admin activation.

## Downstream
| Module | Dependency |
|--------|-----------|
| M8 Nomination Request Submission | Requires authenticated customer session |
| M9 Nomination Status & Payment | Requires authenticated customer session |
| M10 Customer Dashboard & Monitoring | Requires authenticated customer session |

## Key Decisions
- Admin must manually approve each registration before the account becomes active (FR-CA-02).
- STS Platform sync is triggered only after Admin activation, not at registration time.
- Customer role (`customer`) is distinct from all internal LPS operator roles.
- Failed STS sync does not block account activation — failure is logged and retried (max 3×, exponential backoff).

---

## UI/UX Design

M7 punya **dua surface**:
- **Surface A — Customer Portal** (Bahasa Indonesia): customer registration, login, register success.
- **Surface B — Internal Operator** (English): admin login, Customer Management list, customer detail (approve/reject), Add Customer manual.

**Reference wajib sebelum kerja UI:**
- Foundation & komponen: [`implementation/design/lps-design-system.md`](../../implementation/design/lps-design-system.md)
- Per-modul UI design: [`implementation/design/m7-customer-authentication-ui.md`](../../implementation/design/m7-customer-authentication-ui.md)

Setiap kerja UI di modul ini wajib invoke skill `ui-ux-pro-max` untuk validasi/generate komponen.
