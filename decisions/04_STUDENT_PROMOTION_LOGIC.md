# Decision 04 — Student Promotion Logic (Result-Based vs Fee-Based)

**Document Version**: 1.0
**Date**: 2026
**Status**: PENDING DECISION
**Module**: Student Promotion, Academic Results
**Prepared By**: Technical Team

---

## 1. Context

The current promotion system gates student promotion entirely on fee payment status. A student can only be promoted if their fees are fully paid (`FULLY_PAID`). Students with `PARTIALLY_PAID` or `PENDING` fee status are blocked from promotion.

This does not reflect how Indian colleges actually determine promotion eligibility. Fee payment is one criterion, but the primary criterion is academic performance — specifically, how many subjects the student has passed and how many backlogs (ATKTs) they carry.

A student who has paid all fees but failed too many subjects cannot be promoted. Conversely, a student who has passed all subjects but has a pending fee installment may be promoted with admin approval and a payment plan.

---

## 2. Current Implementation

### 2.1 Current Promotion Gate

```
Promotion allowed IF:
  student.status === "APPROVED"
  AND fee.status === "FULLY_PAID"
```

### 2.2 What Is Missing

- No `StudentResult` model exists
- No subject-wise marks or pass/fail tracking
- No ATKT (Allowed To Keep Terms) count
- No re-evaluation or re-attempt workflow
- No concept of "advance booking" for next year

### 2.3 Indian Academic Promotion Rules

In Maharashtra (and most Indian universities), a student is promoted based on:

1. **Subjects passed**: Student must pass a minimum number of subjects to be promoted
2. **ATKT limit**: Student can carry a limited number of backlogs (failed subjects) and still be promoted
3. **Fee status**: Fees must be paid (or a payment plan approved) — but this is secondary to academic performance
4. **Re-attempt**: Students who fail can appear for a supplementary exam (typically October) and be promoted if they pass
5. **Re-evaluation**: Students can apply for re-checking of answer sheets; if marks change and they pass, they are eligible for promotion
6. **Advance booking**: A student confident of passing their re-attempt can pay fees for the next year in advance

---

## 3. The Full Picture — What Needs to Be Built

### 3.1 StudentResult Model (Does Not Exist)

```
StudentResult {
  student_id, college_id, course_id,
  semester: Number,
  academicYear: String,
  subjects: [
    {
      subject_id,
      marksObtained: Number,
      maxMarks: Number,
      passingMarks: Number,
      status: PASS | FAIL | ATKT | ABSENT | WITHHELD,
      examType: REGULAR | SUPPLEMENTARY | REVALUATION
    }
  ],
  totalSubjects: Number,
  passedSubjects: Number,
  failedSubjects: Number,
  atktSubjects: Number,
  overallStatus: PASS | FAIL | ATKT | WITHHELD,
  resultDeclaredAt: Date,
  enteredBy: ObjectId (ref: User)
}
```

### 3.2 Promotion Eligibility Logic (Full Version)

```
Promotion allowed IF:
  (passedSubjects >= college.promotionThreshold)
  AND (atktSubjects <= college.maxAtktAllowed)
  AND (fee.status === "FULLY_PAID" OR admin override with reason)
```

### 3.3 Advance Booking Logic

```
Advance booking allowed IF:
  student has pending re-attempt OR re-evaluation
  AND student pays next year's fees
  → Student gets status: "ADVANCE_BOOKED"
  → On result update: if passed → auto-promote, if failed → refund or hold
```

---

## 4. Options Evaluated

### Option A — Build Full Result Module Now

Build `StudentResult` model, marks entry UI, ATKT tracking, re-evaluation workflow, and advance booking in the current sprint.

| Aspect | Detail |
|--------|--------|
| Completeness | Full solution |
| Timeline | 4-6 weeks additional development |
| Risk | Delays all other features |
| Dependency | Requires Exam Coordinator role (Decision 06) |
| Recommended for | V1.1 |

---

### Option B — Admin Override Only (Minimal Change)

Keep fee-based promotion. Add an "Override" button that allows admin to promote a student despite pending fees, with a mandatory reason field.

| Aspect | Detail |
|--------|--------|
| Completeness | Partial — no result tracking |
| Timeline | 2 days |
| Risk | Low |
| Limitation | No academic result data in system |
| Recommended for | Not recommended — does not address the core problem |

---

### Option C — Simple Pass/Fail Flag (Recommended for V1.0)

Add a `promotionEligibility` object to the Student model with a simple admin-set flag:

```
Student {
  promotionEligibility: {
    academicStatus: PASS | FAIL | ATKT | PENDING_RESULT,
    atktCount: Number,
    overriddenBy: ObjectId,
    overrideReason: String,
    setAt: Date
  }
}
```

Promotion gate becomes:
```
Promotion allowed IF:
  academicStatus === "PASS"
  AND (fee.status === "FULLY_PAID" OR overriddenBy is set)
```

| Aspect | Detail |
|--------|--------|
| Completeness | Partial — admin manually sets pass/fail |
| Timeline | 3-4 days |
| Risk | Low |
| Limitation | No subject-wise marks, no auto-calculation |
| Benefit | Unblocks promotion workflow immediately |
| Path forward | Full StudentResult module replaces this in V1.1 |

---

### Option D — Phased: Simple Flag Now, Full Module in V1.1 (Recommended)

Implement Option C immediately. Build the full `StudentResult` module in V1.1 which replaces the simple flag with calculated eligibility.

| Phase | What Gets Built | Timeline |
|-------|----------------|----------|
| V1.0 | Simple `academicStatus` flag on Student model. Admin sets PASS/FAIL/ATKT manually. Promotion gate checks flag + fee. | 3-4 days |
| V1.1 | Full `StudentResult` model. Subject-wise marks entry. Auto-calculated eligibility. Re-attempt and re-evaluation workflow. Advance booking. | 4-6 weeks |

---

## 5. Recommendation

**Option D** — Phased approach.

Reasons:
- The full result module is a significant feature that requires its own planning, UI design, and role assignments (who enters results?)
- Blocking promotion on a feature that doesn't exist yet is not acceptable for pilot colleges
- The simple flag approach is honest — admin knows the result, they set the flag, promotion proceeds
- V1.1 replaces the flag with calculated data — no data loss, clean migration

---

## 6. V1.0 Implementation (Simple Flag)

### 6.1 Student Model Addition

```
promotionEligibility: {
  academicStatus: {
    type: String,
    enum: ["PASS", "FAIL", "ATKT", "PENDING_RESULT"],
    default: "PENDING_RESULT"
  },
  atktCount: { type: Number, default: 0 },
  setBy: { type: ObjectId, ref: "User" },
  setAt: Date,
  remarks: String
}
```

### 6.2 Updated Promotion Gate

```
Promotion allowed IF:
  student.promotionEligibility.academicStatus === "PASS"
  OR (academicStatus === "ATKT" AND atktCount <= college.maxAtktAllowed)
  AND (fee.status === "FULLY_PAID" OR feeOverrideApproved === true)
```

### 6.3 New Admin Action

Admin can set `academicStatus` for a student before promotion:
- `PASS` — student passed all required subjects
- `FAIL` — student failed, not eligible for promotion
- `ATKT` — student has backlogs but within allowed limit
- `PENDING_RESULT` — result not yet declared (default, blocks promotion)

---

## 7. V1.1 Implementation (Full Result Module)

### 7.1 New Model: StudentResult

Full subject-wise marks entry per semester per student.

### 7.2 Auto-Calculation

System calculates `academicStatus` from `StudentResult` data. Admin no longer sets it manually.

### 7.3 Re-attempt Workflow

```
Student fails → status: FAIL
Student registers for supplementary exam → status: REATTEMPT_PENDING
Result entered → if pass: status: PASS, trigger promotion
              → if fail: status: FAIL, no promotion
```

### 7.4 Re-evaluation Workflow

```
Student applies for re-evaluation → status: REVALUATION_PENDING
University updates marks → admin enters updated marks
System recalculates → if now pass: status: PASS, trigger promotion
```

### 7.5 Advance Booking

```
Student with REATTEMPT_PENDING or REVALUATION_PENDING
  → Can pay next year's fees
  → Status: ADVANCE_BOOKED
  → On result update: auto-promote if pass, refund/hold if fail
```

---

## 8. Impact on Existing Code (V1.0)

| File | Change Required |
|------|----------------|
| `backend/src/models/student.model.js` | Add `promotionEligibility` object |
| `backend/src/controllers/promotion.controller.js` | Update gate to check `academicStatus` + fee |
| `backend/src/controllers/student.controller.js` | Add `setAcademicStatus` endpoint |
| `frontend` — StudentPromotion.jsx | Show `academicStatus` per student, allow admin to set it |
| `frontend` — ViewStudent.jsx | Show academic status in student detail view |

---

## 9. Open Questions — Requires Decision

| # | Question | Options | Impact |
|---|----------|---------|--------|
| Q1 | Who enters student results in V1.1 — College Admin, Exam Coordinator, or Teacher? | Admin / Exam Coordinator / Teacher | Affects role permissions |
| Q2 | Does the system need to calculate SGPA/CGPA or just pass/fail per subject? | Pass/fail only / SGPA/CGPA | Affects result model complexity |
| Q3 | Are results entered manually or imported from university (Excel/CSV)? | Manual / Import | Affects UI design |
| Q4 | What is the maximum ATKT count allowed for promotion? Is it configurable per college? | Fixed / Configurable | Affects college settings model |
| Q5 | Should advance booking create a new StudentFee record for the next year? | Yes / No | Affects fee management |
| Q6 | If a student on advance booking fails their re-attempt, what happens to the fee paid? | Refund / Hold / Transfer | Affects payment workflow |
| Q7 | Should the system send an automatic notification to students when their result is entered? | Yes / No | Affects notification service |

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
