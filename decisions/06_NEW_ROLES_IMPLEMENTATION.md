# Decision 06 — New Roles Implementation (ACCOUNTANT, ADMISSION_OFFICER, EXAM_COORDINATOR, PRINCIPAL)

**Document Version**: 1.0
**Date**: 2026
**Status**: PENDING DECISION
**Module**: Authentication, Role Management, Access Control
**Prepared By**: Technical Team

---

## 1. Context

The platform currently has 4 active roles: `SUPER_ADMIN`, `COLLEGE_ADMIN`, `TEACHER`, `STUDENT`. The `HOD` role exists in constants but is implemented as a permission overlay on `TEACHER`, not as a standalone role.

As the platform grows, college operations require more granular role separation. A College Admin currently has access to everything — fees, admissions, academic data, reports, teacher management. In a real college, these responsibilities are divided across multiple staff members:

- The **Accountant** handles fee collection and financial reporting but should not access academic records
- The **Admission Officer** handles student intake and document verification but should not see enrolled students' data
- The **Exam Coordinator** manages exam schedules, hall tickets, and results but is not involved in day-to-day administration
- The **Principal** has broad oversight but limited write access — they approve decisions rather than execute them

An Architecture Decision Record (ADR) has been prepared recommending a **Hybrid RBAC** approach — keeping roles as primary identity and adding a `Permission` model for fine-grained control. This document decides the implementation approach and sequencing.

---

## 2. Current Role Architecture

### 2.1 How Roles Work Today

```javascript
// Role check in routes (current pattern)
router.get("/students", role("COLLEGE_ADMIN", "TEACHER"), studentController.getAll);
router.post("/fees", role("COLLEGE_ADMIN"), feeController.record);
```

- Role is stored as a single string on the `User` model
- `role.middleware.js` does a simple allowlist check
- No permission model exists
- No secondary roles
- No department-scoped access (except HOD via separate middleware)

### 2.2 Known Issues with Current Architecture

| Issue | Impact |
|-------|--------|
| College Admin has access to everything | No separation of duties |
| No fine-grained permissions | Cannot give Accountant fee access without full admin access |
| Role explosion risk | Adding 4+ roles to a flat enum becomes unmanageable |
| Frontend-backend mismatch | Frontend has a detailed `rolePermissions.js` matrix; backend only checks role strings |
| PRINCIPAL defined in constants but never used | Dead code |

---

## 3. Options Evaluated

### Option A — Add Roles to Enum Only (Pure RBAC)

Add the 4 new roles to the `User.role` enum and add route guards for each.

```javascript
// Example
router.get("/payments", role("COLLEGE_ADMIN", "ACCOUNTANT"), paymentController.getAll);
router.post("/payments/record", role("COLLEGE_ADMIN", "ACCOUNTANT"), paymentController.record);
```

| Aspect | Detail |
|--------|--------|
| Implementation time | 2-3 weeks |
| Complexity | Low |
| Backward compatible | Yes |
| Flexibility | Low — cannot handle nuanced access (e.g., Accountant can view but not modify fee structures) |
| Future risk | Role explosion as more roles are added |
| Recommended for | Immediate need, small number of roles |

---

### Option B — Full Hybrid RBAC (Role + Permission Model)

Build a `Permission` model that stores resource-action-scope mappings per role. Add `secondaryRoles` to User model. Build permission middleware.

```javascript
// Example
router.get("/payments",
  auth,
  collegeIsolation,
  requirePermission("payments", "read", { scope: "own_college" }),
  paymentController.getAll
);
```

| Aspect | Detail |
|--------|--------|
| Implementation time | 8-10 weeks |
| Complexity | High |
| Backward compatible | Yes (roles remain primary) |
| Flexibility | Very high — any access pattern possible |
| Future risk | Low — scales to any number of roles |
| Recommended for | Long-term architecture |

---

### Option C — Phased: Enum First, Hybrid Later (Recommended)

Add the 4 new roles to the enum now with carefully defined route guards. Migrate to the hybrid model in a future sprint when the complexity justifies it.

| Phase | What Gets Built | Timeline |
|-------|----------------|----------|
| Phase 1 (Now) | Add 4 roles to enum. Define route guards per role. Build role-specific dashboards. | 2-3 weeks |
| Phase 2 (V1.1) | Build Permission model. Migrate route guards to permission checks. Add secondary roles. | 6-8 weeks |

| Aspect | Detail |
|--------|--------|
| Implementation time | 2-3 weeks for Phase 1 |
| Complexity | Low for Phase 1 |
| Backward compatible | Yes |
| Risk | Low — Phase 1 is additive, no breaking changes |
| Recommended | Yes |

---

## 4. Recommendation

**Option C** — Phased implementation.

Reasons:
- The 4 new roles are needed now. Waiting 8-10 weeks for the full hybrid system is not acceptable.
- The hybrid system is the right long-term architecture but requires careful design and testing.
- Phase 1 (enum + route guards) is safe, fast, and reversible.
- Phase 2 can be planned properly once Phase 1 is live and real usage patterns are observed.

---

## 5. Phase 1 — Role Definitions and Permissions

### 5.1 ACCOUNTANT

**Purpose**: Fee collection, payment tracking, financial reporting. No access to academic data.

| Resource | Can Do | Cannot Do |
|----------|--------|-----------|
| Fee structures | View | Create, Edit, Delete |
| Student fees | View, Record offline payment, Mark paid | Modify fee structure |
| Payment history | View, Export | Delete |
| Receipts | Generate, Download | — |
| Students | View name, contact, fee status | View academic records, attendance |
| Reports | View and export financial reports | View academic reports |
| Admissions | No access | — |
| Timetable | No access | — |
| Attendance | No access | — |

---

### 5.2 ADMISSION_OFFICER

**Purpose**: Student intake, document verification, enrollment processing. Access limited to pending applications only.

| Resource | Can Do | Cannot Do |
|----------|--------|-----------|
| Student applications | View pending, Approve, Reject | View enrolled students |
| Documents | View, Verify | — |
| Fee structures | View (for counseling) | Modify |
| Departments / Courses | View (for counseling) | Modify |
| Reports | View admission funnel | View financial or academic reports |
| Enrolled students | No access | — |
| Payments | No access | — |
| Timetable | No access | — |

---

### 5.3 EXAM_COORDINATOR

**Purpose**: Exam scheduling, hall tickets, result entry, invigilation management.

> **Note**: This role is partially blocked by the absence of the `StudentResult` model (see Decision 04). Full functionality requires V1.1. Phase 1 implementation covers what is possible without the result module.

| Resource | Can Do | Cannot Do |
|----------|--------|-----------|
| Students | View (for exam lists) | Modify |
| Teachers | View (for invigilation) | Modify |
| Attendance | View (for eligibility check) | Modify |
| Hall tickets | Generate, Download | — |
| Results (V1.1) | Enter, Update | Publish without Principal approval |
| Reports | View exam-related reports | View financial reports |
| Fee structures | No access | — |
| Admissions | No access | — |

---

### 5.4 PRINCIPAL

**Purpose**: Academic oversight, strategic reporting, approval of escalated decisions.

| Resource | Can Do | Cannot Do |
|----------|--------|-----------|
| Students | View all, Override academic holds | Delete |
| Teachers | View all | Delete |
| Departments | View, Update | Delete |
| Fees | View summary | Record payments, Modify structures |
| Attendance | View reports | Modify records |
| Reports | View all reports, Export | — |
| Notifications | Create college-wide announcements | — |
| Audit logs | View | — |
| Results (V1.1) | Approve result publication | Enter results |

---

## 6. Impact on Existing Code (Phase 1)

| File | Change Required |
|------|----------------|
| `backend/src/utils/constants.js` | Add `ACCOUNTANT`, `ADMISSION_OFFICER`, `EXAM_COORDINATOR`, `PRINCIPAL` to ROLE enum |
| `backend/src/models/user.model.js` | Add new roles to role enum |
| `backend/src/routes/*.js` | Add new roles to route guards where applicable |
| `backend/src/controllers/` | Create role-specific controllers or extend existing ones |
| `frontend/src/components/Sidebar/config/rolePermissions.js` | Add permission matrix for 4 new roles |
| `frontend/src/components/Sidebar/config/navigation.config.js` | Add navigation items for 4 new roles |
| `frontend/src/pages/dashboard/` | Create dashboard pages for each new role |
| `backend/src/utils/seedSuperAdmin.js` | No change — new roles are college-level, not platform-level |

---

## 7. New Role Dashboards Required

| Role | Dashboard Sections |
|------|--------------------|
| ACCOUNTANT | Fee collection summary, Pending payments, Payment history, Offline payment entry, Financial reports |
| ADMISSION_OFFICER | Pending applications, Document verification queue, Admission funnel, Approved/Rejected list |
| EXAM_COORDINATOR | Student lists by semester, Hall ticket generation, Invigilation schedule, Results entry (V1.1) |
| PRINCIPAL | College overview, Attendance summary, Fee collection summary, Academic reports, Audit logs |

---

## 8. Open Questions — Requires Decision

| # | Question | Options | Impact |
|---|----------|---------|--------|
| Q1 | Can ACCOUNTANT create fee structures or only record payments? | View only / Create too | Affects route guards |
| Q2 | Can ADMISSION_OFFICER reject students or only approve? | Approve only / Both | Affects approval controller |
| Q3 | Should PRINCIPAL be able to override College Admin decisions (e.g., approve a rejected student)? | Yes / No | Affects approval workflow |
| Q4 | Can a user have multiple roles (e.g., a teacher who is also Exam Coordinator)? | Yes (Phase 2) / No | Affects User model |
| Q5 | Who creates ACCOUNTANT and ADMISSION_OFFICER accounts — College Admin or Super Admin? | College Admin / Super Admin | Affects user creation flow |
| Q6 | Should EXAM_COORDINATOR be built in Phase 1 (limited) or deferred to V1.1 with the result module? | Phase 1 limited / V1.1 full | Affects sprint scope |
| Q7 | Does PRINCIPAL replace College Admin or work alongside them? | Replace / Alongside | Affects permission overlap |

---

## 9. Sign-Off Required

| Role | Name | Decision | Date |
|------|------|----------|------|
| Product Owner | | ☐ Approved ☐ Changes Needed | |
| Technical Lead | | ☐ Approved ☐ Changes Needed | |
| Development Team | | ☐ Acknowledged | |

---

**Document Owner**: Technical Team
**Next Review**: Before sprint planning
