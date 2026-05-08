# M9b — EPB & Invoice (Customer Portal)

## Overview
A dedicated menu within the LPS Customer Portal that serves as the payment status monitoring interface after the customer has submitted their first proof of payment via M9 (EPB Confirmation). This module manages the full payment verification cycle: Unpaid → Pending Review → Payment Reject / Paid. All billing calculations and verification decisions are made by STS Platform; LPS only displays statuses and facilitates upload/re-upload of payment proofs.

## Boundaries
**In scope:**
- Displaying the list of all EPBs belonging to a customer with their current payment status
- Viewing EPB & Invoice detail (view-only, data sourced from STS Platform via M9)
- Unpaid flow: customer clicks "Pay" → uploads payment proof → submits
- Pending Review flow: view-only, waiting for STS Platform decision
- Payment Reject flow: customer clicks "Revision Data" → uploads corrected proof → re-submits
- Paid flow: view-only confirmation that payment is complete
- Receiving payment status update webhooks from STS Platform
- Forwarding uploaded payment proofs to STS Platform via API

**Out of scope:**
- First payment proof upload / EPB Confirmation (handled by M9)
- Billing calculations, invoice generation, or payment reconciliation (STS Platform)
- Nomination approval logic (STS Platform)
- Dashboard summary (M10)

## Dependencies
| Dependency | Reason |
|-----------|--------|
| M9 Nomination Status & EPB Confirmation | Creates the initial `epb_payments` row (status UNPAID) after EPB Confirmation submit |
| M7 Customer Authentication | Customer must be authenticated |
| STS Platform Webhook (inbound) | Receives payment status updates: PENDING_REVIEW, PAYMENT_REJECT, PAID |
| STS Platform API (outbound) | Sends uploaded payment proofs for verification |

## Downstream
| Module | Dependency |
|--------|-----------|
| M10 Customer Dashboard & Monitoring | Displays EPB payment summary on dashboard |

## Key Decisions
- M9b is intentionally a **separate module** from M9. The split reflects the Swimlane V3 architecture: M9 is the EPB Confirmation entry point; M9b is the ongoing payment management menu.
- LPS does **not** verify payments. All verification decisions come from STS Platform via webhook.
- The `epb_payments` table is owned by M9b but the first row is inserted by M9 (Step 6 of M9 handoff).
- Payment proof uploads from both Unpaid (FR-EI-03) and Payment Reject (FR-EI-05) flows use the same upload endpoint with a mode flag.
- STS Platform webhook events handled here: `EPB_PENDING_REVIEW`, `EPB_PAYMENT_REJECT`, `EPB_PAID`.
