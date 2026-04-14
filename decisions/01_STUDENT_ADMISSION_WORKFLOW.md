# Decision 01 — Student Admission Workflow & Status States

**Document Version**: 1.0
**Date**: 2026
**Status**: PENDING DECISION
**Module**: Student Management
**Prepared By**: Technical Team

---

## 1. Context

The current system handles student admission through a linear flow where an admin approves a student application and the system immediately creates a User account and sends login credentials via email.

This does not reflect the actual admission process in Indian colleges. A student typically applies to multiple colleges simultaneously. An approval from one college does not mean the student will join that college. The student visits the college, confirms their seat, and makes a downpayment before they are considered enrolled.

Sending login credentials at the point of approval creates orphaned accounts for students who never actually join, pollutes the user database, and creates confusion for both the college and the student.

---

## 2. Current Implementation

### 2.1 Current Status States

```
PENDING → APPROVED → REJECTED
                   → DEACTIVATED
                   → ALUMNI
                   → DELETED
```

### 2.2 Current Approval Flow

1. Student submits application → status: `PENDING`
2. Admin clicks Approve → system immediately:
   - Creates `User` account with temp password
   - Creates `StudentFee` record from fee structure
   - Sends admission approval email with login credentials
   - Sets status: `APPROVED`

### 2.3 Problem with Current Flow

- Credentials are sent before the student has confirmed they want to join
- Fee record is created for students who may never enroll
- No way to track how many students were offered admission vs. how many confirmed
- No intermediate state between "we approved them" and "they are actually enrolled"

---

## 3. Required Flow (Business Reality)

```
Student applies
      ↓
Admin reviews documents
      ↓
Admin approves internally (offer made — no credentials yet)
      ↓
College contacts student (call or email)
      ↓
Student visits college and confirms seat
      ↓
Admin confirms admission + records downpayment
      ↓
System generates credentials and sends to student
      ↓
Student logs in and is active
```

---

## 4. Options Evaluated

### Option A — Add Two New States

**New flow:**
```
PENDING → OFFER_MADE → SEAT_CONFIRMED → ENROLLED
        → REJECTED
        → DEACTIVATED
        → ALUMNI
```

| Aspect | Detail |
|--------|--------|
| `OFFER_MADE` | Admin approved internally. No credentials. College contacts student. |
| `SEAT_CONFIRMED` | Student visited and confirmed. Credentials generated. Fee record created. |
| `ENROLLED` | Downpayment received. Student is active. |
| Migration needed | Yes — all existing `APPROVED` students must be mapped to `ENROLLED` |
| Reporting benefit | Can track offer-to-confirmation conversion rate |
| Complexity | High — 3 new API endpoints, frontend changes across all student views |

---

### Option B — Add One New State (Recommended)

**New flow:**
```
PENDING → APPROVED → CONFIRMED → REJECTED
                              → DEACTIVATED
                              → ALUMNI
```

| Aspect | Detail |
|--------|--------|
| `APPROVED` | Admin approved internally. No credentials sent. College contacts student. |
| `CONFIRMED` | Student confirmed seat. Credentials generated. Fee record created. Downpayment recorded. |
| Migration needed | Yes — all existing `APPROVED` students must be mapped to `CONFIRMED` |
| Reporting benefit | Can track approval-to-confirmation rate |
| Complexity | Medium — 1 new status, 1 new API endpoint, targeted frontend changes |

---

### Option C — Keep Current States, Add Manual Trigger

**Flow remains:**
```
PENDING → APPROVED → REJECTED
```

| Aspect | Detail |
|--------|--------|
| Change | Remove auto-credential sending from approval. Add "Send Credentials" button in admin UI. |
| Migration needed | None |
| Reporting benefit | None — no way to distinguish offered vs. enrolled |
| Complexity | Low — 1 backend change, 1 frontend button |
| Downside | No audit trail of when student confirmed. No intermediate state. |

---

## 5. Recommendation

**Option B** — Add one new `CONFIRMED` state.

Reasons:
- Matches the actual two-step process (offer → confirmation)
- Minimal migration effort
- Enables conversion rate reporting (how many offers converted to enrollments)
- Clean separation: `APPROVED` = college's decision, `CONFIRMED` = student's decision

---

## 6. Impact on Existing Code

| File | Change Required |
|------|----------------|
| `backend/src/utils/constants.js` | Add `CONFIRMED` to `STUDENT_STATUS` |
| `backend/src/models/student.model.js` | Add `CONFIRMED` to status enum. Add `confirmedAt`, `confirmedBy` fields |
| `backend/src/controllers/studentApproval.controller.js` | Split into `approveStudent` (no credentials) and `confirmAdmission` (credentials + fee record) |
| `backend/src/controllers/studentApproval.controller.js` | Move User account creation and StudentFee creation from `approveStudent` to `confirmAdmission` |
| `backend/scripts/migrate.js` | Migration: set all existing `APPROVED` students to `CONFIRMED` |
| `frontend` — all student list views | Add `CONFIRMED` status display and filter |
| `frontend` — PendingApprovals.jsx | Add "Confirm Admission" button separate from "Approve" |

---

## 7. New API Endpoints Required

| Method | Endpoint | Action | Who Can Call |
|--------|----------|--------|-------------|
| `PATCH` | `/api/students/:id/approve` | Sets status to `APPROVED`. No credentials. Sends approval notification. | COLLEGE_ADMIN, ADMISSION_OFFICER |
| `PATCH` | `/api/students/:id/confirm` | Sets status to `CONFIRMED`. Creates User account. Creates StudentFee. Sends credentials. | COLLEGE_ADMIN, ADMISSION_OFFICER |
| `PATCH` | `/api/students/:id/reject` | Sets status to `REJECTED`. Sends rejection email. | COLLEGE_ADMIN, ADMISSION_OFFICER |

---

## 8. Open Questions — Requires Decision

| # | Question | Options | Impact |
|---|----------|---------|--------|
| Q1 | Should a student who was `APPROVED` but never `CONFIRMED` be able to reapply the following year? | Yes / No | Affects `canReapply` flag logic |
| Q2 | Should the approval notification to the student be an email, SMS, or both? | Email / SMS / Both | Affects notification service |
| Q3 | Who can trigger `confirmAdmission` — College Admin only, or also Admission Officer? | Admin only / Both | Affects role middleware |
| Q4 | Should the downpayment be recorded at confirmation time, or is confirmation separate from payment? | Combined / Separate | Affects StudentFee creation timing |
| Q5 | How long should an `APPROVED` (unconfirmed) application remain active before auto-expiry? | 7 days / 15 days / No expiry | Affects cron job logic |

---

## 9. Migration Plan

All existing students with status `APPROVED` must be moved to `CONFIRMED` since they already have User accounts and fee records — they are effectively enrolled.

```
Before migration: APPROVED (has User account + StudentFee)
After migration:  CONFIRMED (has User account + StudentFee)
```

This is a one-time, non-destructive migration. No data is deleted.

---

## 10. Sign-Off Required

| Role | Name | Decision | Date |
|------|------|----------|------|
| Product Owner | | ☐ Approved ☐ Changes Needed | |
| Technical Lead | | ☐ Approved ☐ Changes Needed | |
| Development Team | | ☐ Acknowledged | |

---

**Document Owner**: Technical Team
**Next Review**: Before sprint planning
