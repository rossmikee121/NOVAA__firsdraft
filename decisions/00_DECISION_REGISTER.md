# NOVAA — Technical Decision Register

**Version**: 1.0
**Date**: 2026
**Status**: IN REVIEW
**Purpose**: Pre-implementation decisions required before development begins or PRD is updated

---

## Overview

This register contains all open architectural and product decisions for the NOVAA platform. Each decision document defines the problem, evaluates options, makes a recommendation, and lists open questions that require stakeholder input before implementation begins.

No development work should start on any of these areas until the corresponding decision document has been reviewed and signed off.

---

## Decision Status Summary

| # | Decision | Module | Status | Priority | Dependency |
|---|----------|--------|--------|----------|------------|
| 01 | [Student Admission Workflow & Status States](./01_STUDENT_ADMISSION_WORKFLOW.md) | Student Management | ⏳ Pending | 🔴 Critical | None |
| 02 | [Course Year Labels (FY / SY / TY)](./02_COURSE_YEAR_LABELS.md) | Course Management | ⏳ Pending | 🔴 High | None |
| 03 | [Teacher Classroom & Division Assignment](./03_TEACHER_CLASSROOM_ASSIGNMENT.md) | Teacher Management, Timetable | ⏳ Pending | 🔴 High | Decision 02 |
| 04 | [Student Promotion Logic](./04_STUDENT_PROMOTION_LOGIC.md) | Promotion, Academic Results | ⏳ Pending | 🔴 High | Decision 01 |
| 05 | [Subscription & Billing Model](./05_SUBSCRIPTION_BILLING_MODEL.md) | Platform Administration | ⏳ Pending | 🔴 Critical | None |
| 06 | [New Roles Implementation](./06_NEW_ROLES_IMPLEMENTATION.md) | Authentication, Access Control | ⏳ Pending | 🟡 High | Decision 01 |
| 07 | [HOD Timetable Self-Management](./07_HOD_TIMETABLE_SELF_MANAGEMENT.md) | Timetable, HOD Role | ⏳ Pending | 🟡 Medium | Decision 03 |
| 08 | [Credential Delivery & First Login Flow](./08_CREDENTIAL_DELIVERY_FIRST_LOGIN.md) | Authentication, Onboarding | ⏳ Pending | 🔴 High | None |
| 09 | [Student ID Card & Hall Ticket Generation](./09_ID_CARD_HALL_TICKET_GENERATION.md) | Student Services | ⏳ Pending | 🟡 Medium | Decision 02 |

---

## Decision Dependencies

```
Decision 01 (Admission Workflow)
  └── Decision 04 (Promotion Logic)
  └── Decision 06 (New Roles)

Decision 02 (Year Labels)
  └── Decision 03 (Classroom Assignment)
  └── Decision 07 (HOD Timetable)
  └── Decision 09 (ID Card)

Decision 03 (Classroom Assignment)
  └── Decision 07 (HOD Timetable)

Decision 05 (Billing) — Independent
Decision 08 (Credentials) — Independent
```

---

## Recommended Review Order

Review decisions in this order to avoid revisiting earlier decisions when later ones are discussed:

1. **Decision 05** — Billing model (independent, critical for commercial viability)
2. **Decision 08** — Credential delivery (independent, needed for onboarding)
3. **Decision 01** — Admission workflow (unblocks Decisions 04 and 06)
4. **Decision 02** — Year labels (unblocks Decisions 03, 07, 09)
5. **Decision 03** — Classroom assignment (unblocks Decision 07)
6. **Decision 04** — Promotion logic (depends on Decision 01)
7. **Decision 06** — New roles (depends on Decision 01)
8. **Decision 07** — HOD timetable (depends on Decisions 02 and 03)
9. **Decision 09** — ID card (depends on Decision 02)

---

## What Happens After Sign-Off

Once a decision is signed off:

1. Technical Lead creates implementation tasks in the project tracker
2. Developer assigned writes a brief implementation spec (model changes, API contract, UI requirements)
3. Implementation spec reviewed by Technical Lead before coding begins
4. PRD updated to reflect the decided approach
5. Developer implements, Technical Lead reviews

---

## Document Status Legend

| Status | Meaning |
|--------|---------|
| ⏳ Pending | Awaiting review and sign-off |
| 🔄 In Review | Currently being discussed |
| ✅ Approved | Decision made, ready for implementation |
| ❌ Rejected | Approach rejected, needs rework |
| 🔒 Implemented | Decision implemented in codebase |

---

## Open Questions Tracker

The following questions appear across multiple decision documents and require answers from the product team before any implementation begins.

| Question | Relevant Decisions | Priority |
|----------|--------------------|----------|
| What are the exact plan tiers and pricing for the billing model? | 05 | 🔴 Critical |
| Does LemmeCode have a GSTIN for invoice generation? | 05 | 🔴 Critical |
| Who enters student exam results — Admin, Exam Coordinator, or Teacher? | 04, 06 | 🔴 High |
| Should promotion be triggered by year (FY→SY) or by semester (Sem 2→Sem 3)? | 02, 04 | 🔴 High |
| Who can trigger "Confirm Admission" — College Admin only or also Admission Officer? | 01, 06 | 🔴 High |
| Should the sales representative have a login to view temp credentials? | 08 | 🟡 Medium |
| Should room conflict detection be a hard block or soft warning? | 03, 07 | 🟡 Medium |
| Should students be able to download their own ID card? | 09 | 🟡 Medium |
| What is the maximum ATKT count allowed for promotion — fixed or configurable per college? | 04 | 🟡 Medium |
| Should EXAM_COORDINATOR be built in Phase 1 (limited) or deferred to V1.1? | 06 | 🟡 Medium |

---

**Document Owner**: Technical Team
**Last Updated**: 2026
**Next Review**: Decision review meeting with stakeholders
