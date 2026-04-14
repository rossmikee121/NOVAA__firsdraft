# Decision 05 — Subscription & Billing Model

**Document Version**: 1.0
**Date**: 2026
**Status**: PENDING DECISION
**Module**: Platform Administration, Super Admin
**Prepared By**: Technical Team

---

## 1. Context

The platform currently has no functional subscription or billing system. The `collegeIsolation` middleware checks `college.subscriptionExpiry` to block expired colleges, but the `College` model has no `subscriptionExpiry` field — meaning this check never triggers and all colleges have indefinite access regardless of payment status.

This is a critical gap for the platform's commercial viability. Without a billing system, there is no mechanism to:
- Track which colleges have paid and which have not
- Restrict access for non-paying colleges
- Generate invoices for accounting and GST compliance
- Manage renewals and plan upgrades

---

## 2. Why Standard SaaS Billing Does Not Apply

Most SaaS platforms use automated billing: credit card on file, auto-renewal, instant access on payment. This model does not work for Indian educational institutions for the following reasons:

1. **Procurement process**: Government-aided and autonomous colleges follow a formal procurement process. Purchases above a threshold require a tender, purchase order (PO), or committee approval. This can take weeks.

2. **Payment methods**: Institutional payments are made via NEFT/RTGS bank transfer, demand draft (DD), or cheque — not credit cards. UPI is used only for smaller amounts.

3. **GST compliance**: Invoices must be GST-compliant with GSTIN, HSN code, place of supply, and correct tax split (CGST/SGST for intra-state, IGST for inter-state).

4. **Approval hierarchy**: The person who uses the software (College Admin) is often not the person who authorises payment (Principal, Finance Officer, or Bursar). Payment proof must be uploaded and verified manually.

5. **Academic year alignment**: Billing cycles align with the academic year (June to May), not the calendar year.

---

## 3. Proposed Billing Workflow

```
Step 1: SUPER_ADMIN creates invoice for a college
        → Selects plan, billing period, amount
        → System generates GST-compliant invoice PDF
        → Invoice sent to college admin + billing contact email

Step 2: College Admin / Principal / Accountant receives invoice
        → Views invoice in portal
        → Downloads PDF for internal approval process
        → Makes payment via bank transfer / cheque / DD / UPI
        → Uploads payment proof (screenshot, cheque photo, UTR number)

Step 3: SUPER_ADMIN / Billing Manager verifies payment
        → Matches uploaded proof with bank statement
        → Marks invoice as PAID
        → System automatically extends college.subscriptionExpiry
        → Confirmation email sent to college

Step 4: Access management
        → 7 days before expiry: warning banner shown to college admin
        → At expiry: non-critical features soft-blocked
        → 15 days after expiry: read-only mode
        → 30 days after expiry: full suspension (configurable)
```

---

## 4. New Models Required

### 4.1 College Model Additions

```
College {
  // Existing fields remain unchanged

  subscription: {
    plan: String (enum: TRIAL | BASIC | PREMIUM | ENTERPRISE),
    subscriptionExpiry: Date,
    trialEndsAt: Date,
    gracePeriodDays: Number (default: 15),
    autoRenew: Boolean (default: false)
  },

  billingContact: {
    name: String,
    email: String,
    phone: String,
    designation: String   // e.g., "Finance Officer", "Bursar"
  },

  gstDetails: {
    gstin: String,        // 15-digit GSTIN of the college
    legalName: String,    // As per GST registration
    address: String,
    stateCode: String     // 2-digit state code
  }
}
```

### 4.2 New Invoice Model

```
Invoice {
  college_id: ObjectId (ref: College),
  invoiceNumber: String (unique),   // Format: INV-2026-000001
  invoiceDate: Date,
  dueDate: Date,

  billingPeriod: {
    from: Date,
    to: Date
  },

  plan: String (enum: BASIC | PREMIUM | ENTERPRISE),

  amount: {
    subtotal: Number,
    cgst: Number,         // 9% for intra-state
    sgst: Number,         // 9% for intra-state
    igst: Number,         // 18% for inter-state
    total: Number,
    currency: String (default: INR)
  },

  gstDetails: {
    hsnCode: String,      // e.g., "998314" for software services
    placeOfSupply: String,
    reverseCharge: Boolean (default: false)
  },

  tenderReference: {
    tenderNumber: String,
    poNumber: String,
    issuedBy: String,
    attachmentUrl: String
  },

  status: String (enum: DRAFT | SENT | PARTIAL | PAID | OVERDUE | CANCELLED),

  paymentProof: {
    uploadedBy: ObjectId (ref: User),
    uploadedAt: Date,
    files: [
      {
        url: String,
        filename: String,
        fileType: String,       // bank_transfer | cheque | dd | upi
        referenceNumber: String, // UTR / Cheque No / DD No
        remarks: String
      }
    ],
    verifiedBy: ObjectId (ref: User),
    verifiedAt: Date,
    verificationRemarks: String
  },

  remindersSent: [
    {
      sentAt: Date,
      type: String (enum: DUE_SOON | OVERDUE_7 | OVERDUE_15 | FINAL_NOTICE),
      channel: String (enum: EMAIL | IN_APP)
    }
  ],

  auditLog: [
    {
      action: String,
      by: ObjectId (ref: User),
      at: Date,
      ip: String,
      notes: String
    }
  ]
}
```

---

## 5. Plan Tiers

The following is a proposed structure. Exact pricing and feature inclusions require business decision.

| Plan | Target | Features | Price (Proposed) |
|------|--------|----------|-----------------|
| **TRIAL** | New colleges | Full access, limited to 30 days | Free |
| **BASIC** | Small colleges (<500 students) | Core modules: Admissions, Fees, Attendance | TBD |
| **PREMIUM** | Medium colleges (500-2000 students) | All modules including Timetable, Reports, Notifications | TBD |
| **ENTERPRISE** | Large colleges (2000+ students) | All modules + priority support + custom integrations | TBD |

---

## 6. Role-Based Access for Billing

| Role | View Invoices | Upload Payment Proof | Verify Payment | Create Invoice |
|------|--------------|---------------------|----------------|----------------|
| SUPER_ADMIN | All colleges | No | Yes | Yes |
| BILLING_MANAGER | All colleges | No | Yes | No |
| COLLEGE_ADMIN | Own college | Yes | No | No |
| PRINCIPAL | Own college | Yes | No | No |
| ACCOUNTANT | Own college | Yes | No | No |

### 6.1 New Role: BILLING_MANAGER

A platform-level role (not college-level) for the LemmeCode finance team. Can verify payments and manage invoices across all colleges but cannot access academic data.

---

## 7. Email Notifications

| Trigger | Recipients | Content |
|---------|-----------|---------|
| Invoice created and sent | College Admin + Billing Contact | Invoice PDF, payment instructions, due date |
| Payment proof uploaded | LemmeCode billing team | Proof files, college name, invoice number |
| Payment verified | College Admin + Billing Contact | Confirmation, new expiry date, receipt PDF |
| 7 days before expiry | College Admin + Billing Contact | Renewal reminder, invoice link |
| Invoice overdue | College Admin + Billing Contact | Overdue notice, suspension warning |
| Final notice (15 days overdue) | College Admin + Principal | Final warning before suspension |

---

## 8. Access Restriction Logic

```
College access states based on subscription:

ACTIVE (subscriptionExpiry > today):
  → Full access to all features

WARNING (subscriptionExpiry within 7 days):
  → Full access
  → Warning banner shown on every page

GRACE_PERIOD (subscriptionExpiry passed, within gracePeriodDays):
  → Full access
  → Prominent warning banner
  → Daily reminder emails

SOFT_BLOCKED (grace period expired):
  → Read-only access
  → Cannot add new students, process payments, mark attendance
  → Can view existing data

SUSPENDED (30 days after expiry):
  → No access
  → Data retained for 90 days
  → Reactivation possible on payment
```

---

## 9. Cron Jobs Required

| Job | Frequency | Action |
|-----|-----------|--------|
| Check expiring subscriptions | Daily at 9am | Send DUE_SOON reminder for colleges expiring in 7 days |
| Check overdue invoices | Daily at 9am | Update invoice status to OVERDUE, send reminder |
| Check suspended colleges | Daily at 9am | Apply SOFT_BLOCKED or SUSPENDED status |
| Weekly billing summary | Monday 10am | Generate summary report for SUPER_ADMIN |

---

## 10. GST Compliance Requirements

All invoices must include:
- Seller GSTIN (LemmeCode's GSTIN)
- Buyer GSTIN (college's GSTIN, if registered)
- HSN/SAC code for software services
- Place of supply (determines CGST+SGST vs IGST)
- Invoice number in sequential format
- Tax breakdown (subtotal, tax rate, tax amount, total)

---

## 11. Impact on Existing Code

| File | Change Required |
|------|----------------|
| `backend/src/models/college.model.js` | Add `subscription`, `billingContact`, `gstDetails` fields |
| `backend/src/middlewares/collegeIsolation.middleware.js` | Update to check actual `subscription.subscriptionExpiry` field |
| `backend/src/models/` | Create new `invoice.model.js` |
| `backend/src/controllers/` | Create `invoice.controller.js` |
| `backend/src/routes/` | Create `invoice.routes.js` |
| `backend/src/cron/` | Create `billing.cron.js` |
| `backend/src/services/email.service.js` | Add invoice email templates |
| `backend/src/utils/constants.js` | Add `BILLING_MANAGER` to ROLE enum, add plan and invoice status enums |
| `frontend` — Super Admin | Build invoice creation UI, payment verification UI |
| `frontend` — College Admin | Build invoice view, payment proof upload UI |

---

## 12. Open Questions — Requires Decision

| # | Question | Options | Impact |
|---|----------|---------|--------|
| Q1 | What are the exact plan tiers and pricing? | TBD | Affects invoice creation UI |
| Q2 | Does LemmeCode have a GSTIN? | Yes / No / In progress | Affects invoice GST fields |
| Q3 | Is BILLING_MANAGER a separate person or does SUPER_ADMIN handle billing? | Separate / Same | Affects role creation priority |
| Q4 | What happens to college data after 30-day suspension — retained or deleted? | Retained 90 days / Deleted | Affects data retention policy |
| Q5 | Should trial colleges be created with a default 30-day expiry or no expiry? | 30 days / No expiry | Affects college creation flow |
| Q6 | Should the system support partial payments (e.g., 50% now, 50% in 3 months)? | Yes / No | Affects invoice status and StudentFee model |
| Q7 | What is the billing cycle — academic year (Jun-May) or calendar year (Jan-Dec)? | Academic / Calendar | Affects invoice date logic |
| Q8 | Should colleges be able to download their own GST-compliant receipts? | Yes / No | Affects receipt generation |

---

## 13. Sign-Off Required

| Role | Name | Decision | Date |
|------|------|----------|------|
| Product Owner | | ☐ Approved ☐ Changes Needed | |
| Technical Lead | | ☐ Approved ☐ Changes Needed | |
| Finance / Legal | | ☐ Approved ☐ Changes Needed | |
| Development Team | | ☐ Acknowledged | |

---

**Document Owner**: Technical Team
**Next Review**: Before sprint planning
