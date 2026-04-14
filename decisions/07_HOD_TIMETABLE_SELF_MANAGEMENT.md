# Decision 07 — HOD Timetable Self-Management & Department Filtering

**Document Version**: 1.0
**Date**: 2026
**Status**: PENDING DECISION
**Module**: Timetable, HOD Role
**Prepared By**: Technical Team

---

## 1. Context

The HOD (Head of Department) is a teacher with elevated timetable management permissions for their department. The current `hod.middleware.js` correctly restricts timetable creation and management to the HOD of the department that owns the timetable.

However, two gaps exist:

1. **HOD's own timetable**: The College Admin does not create timetables for the HOD. The HOD must create their own teaching schedule. Currently, the HOD can add themselves as a teacher in a slot, but there is no explicit self-scheduling workflow.

2. **Frontend filtering**: When a HOD creates a timetable slot and selects a teacher, the dropdown shows all teachers in the college — not just teachers in the HOD's department. Similarly, room selection is free text with no auto-fill or validation.

These gaps mean HODs must manually filter and type information that the system already knows, increasing the chance of errors and reducing usability.

---

## 2. Current Implementation

### 2.1 HOD Middleware

```
hod.middleware.js checks:
1. req.user.role === "TEACHER"
2. Teacher._id matches Department.hod_id for the timetable's department
3. Attaches req.teacher, req.department, req.timetable to request
```

### 2.2 What Works

- HOD can create, edit, and publish timetables for their department ✅
- HOD can add any teacher (including themselves) to a slot ✅
- HOD can manage timetable exceptions ✅

### 2.3 What Does Not Work

- Teacher dropdown in slot creation shows all college teachers, not just department teachers ❌
- Room field is free text — no auto-fill from teacher's classroom assignment ❌
- No explicit "My Schedule" view for HOD's own teaching slots ❌
- No conflict detection when HOD creates overlapping slots ❌

---

## 3. Required Behaviour

### 3.1 HOD Creating a Slot for Another Teacher

```
HOD opens timetable builder for their department
  ↓
Selects "Add Slot"
  ↓
Teacher dropdown: shows ONLY teachers in HOD's department
  ↓
Subject dropdown: shows ONLY subjects assigned to selected teacher
  ↓
Course + Year dropdown: shows ONLY course-year combinations from teacher's courseAssignments
  ↓
Room field: auto-fills from teacher's courseAssignments for selected course-year
  ↓
Day + Time: HOD selects
  ↓
System validates: no room conflict, no teacher conflict
  ↓
Slot saved
```

### 3.2 HOD Creating Their Own Slot

```
HOD opens "My Schedule" or timetable builder
  ↓
Selects "Add My Slot"
  ↓
Teacher field: pre-filled with HOD's own teacher profile (not a dropdown)
  ↓
Subject, Course, Year, Room: same flow as above
  ↓
System validates conflicts
  ↓
Slot saved
```

---

## 4. Options Evaluated

### Option A — Backend Filtering (API-Level)

Add a query parameter to the teacher list API: `?department_id=xxx`. The frontend passes the HOD's department ID when fetching teachers for the slot creation form.

| Aspect | Detail |
|--------|--------|
| Implementation | Backend: add `department_id` filter to teacher list endpoint. Frontend: pass HOD's department_id when fetching teachers for timetable builder. |
| Complexity | Low |
| Security | Correct — filtering happens on the server |
| Dependency | Requires Decision 03 (courseAssignments) for room auto-fill |

---

### Option B — Frontend Filtering Only

Fetch all teachers, filter in the frontend based on `department_id`.

| Aspect | Detail |
|--------|--------|
| Implementation | Frontend only — filter teacher list by department |
| Complexity | Very low |
| Security | Weak — all teacher data is sent to client, just not displayed |
| Dependency | None |
| Recommended | No — security concern for large colleges |

---

### Option C — Dedicated HOD Timetable Builder Endpoint (Recommended)

Create a dedicated API endpoint for HOD timetable slot creation that automatically scopes all data to the HOD's department.

```
GET /api/timetable/hod/teachers
  → Returns only teachers in HOD's department

GET /api/timetable/hod/subjects?teacher_id=xxx
  → Returns only subjects assigned to that teacher

GET /api/timetable/hod/rooms?teacher_id=xxx&course_id=xxx&year=1
  → Returns room from teacher's courseAssignments
```

| Aspect | Detail |
|--------|--------|
| Implementation | Medium — 3 new API endpoints |
| Security | Strong — all scoping on server |
| Dependency | Requires Decision 03 (courseAssignments) for room endpoint |
| Benefit | Clean separation of HOD timetable operations from general timetable operations |

---

## 5. Recommendation

**Option C** — Dedicated HOD timetable builder endpoints.

Reasons:
- Security: data scoping must happen on the server, not the client
- Clean API design: HOD operations are distinct from admin operations
- Enables room auto-fill once Decision 03 (courseAssignments) is implemented
- Easier to test and audit

---

## 6. Dependency on Decision 03

This decision is partially dependent on Decision 03 (Teacher Classroom Assignment). Specifically:

| Feature | Dependency |
|---------|-----------|
| Teacher dropdown filtering by department | No dependency — uses existing `department_id` on Teacher model |
| Subject dropdown filtering by teacher | No dependency — uses existing `subjects[]` on Teacher model |
| Room auto-fill | Requires Decision 03 — needs `courseAssignments` on Teacher model |
| Conflict detection | No dependency — uses existing TimetableSlot data |

**Recommendation**: Implement teacher and subject filtering now. Implement room auto-fill after Decision 03 is resolved and `courseAssignments` is built.

---

## 7. HOD Self-Scheduling

The HOD is a teacher. Their own teaching slots are created the same way as any other teacher's slots — the HOD selects themselves from the teacher dropdown (or uses the "Add My Slot" shortcut).

No special model changes are needed. The only UI change required is:

- Add an "Add My Slot" button in the HOD's timetable view that pre-fills the teacher field with the HOD's own profile
- Add a "My Teaching Schedule" tab in the HOD dashboard that shows only slots where `teacher_id === HOD's teacher_id`

---

## 8. Conflict Detection

Conflict detection should be enforced at the backend when a slot is created or updated:

| Conflict Type | Check |
|---------------|-------|
| Teacher double-booking | Same `teacher_id`, same `day`, overlapping `startTime`-`endTime` |
| Room double-booking | Same `room`, same `day`, overlapping time (requires Decision 03) |
| Course-year double-booking | Same `course_id` + `year`, same `day`, overlapping time |

Conflict response should be a **soft warning** (not a hard block) to allow admin override for exceptional cases (e.g., a teacher covering two sections simultaneously with a co-teacher).

---

## 9. Impact on Existing Code

| File | Change Required |
|------|----------------|
| `backend/src/routes/timetable.routes.js` | Add HOD-specific endpoints |
| `backend/src/controllers/timetable.controller.js` | Add `getHodTeachers`, `getHodSubjects`, `getHodRooms` functions |
| `backend/src/controllers/timetableSlot.controller.js` | Add conflict detection on slot create/update |
| `frontend` — Timetable builder | Filter teacher dropdown via new HOD endpoint |
| `frontend` — Timetable builder | Auto-fill room from HOD rooms endpoint |
| `frontend` — HOD dashboard | Add "My Teaching Schedule" tab |
| `frontend` — HOD dashboard | Add "Add My Slot" shortcut button |

---

## 10. Open Questions — Requires Decision

| # | Question | Options | Impact |
|---|----------|---------|--------|
| Q1 | Should room conflict detection be a hard block or soft warning? | Hard block / Soft warning | Affects UX and admin flexibility |
| Q2 | Can HOD assign a teacher from another department to a slot in their timetable (e.g., a visiting lecturer)? | Yes / No | Affects teacher dropdown scope |
| Q3 | Should HOD be able to see other departments' timetables (read-only)? | Yes / No | Affects timetable list view |
| Q4 | Should the "Add My Slot" shortcut be available or should HOD always use the standard slot creation form? | Shortcut / Standard form | Affects frontend complexity |

---

## 11. Sign-Off Required

| Role | Name | Decision | Date |
|------|------|----------|------|
| Product Owner | | ☐ Approved ☐ Changes Needed | |
| Technical Lead | | ☐ Approved ☐ Changes Needed | |
| Development Team | | ☐ Acknowledged | |

---

**Document Owner**: Technical Team
**Next Review**: After Decision 03 is resolved
