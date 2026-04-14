# Decision 03 — Teacher Classroom & Division Assignment

**Document Version**: 1.0
**Date**: 2026
**Status**: PENDING DECISION
**Module**: Teacher Management, Timetable
**Prepared By**: Technical Team

---

## 1. Context

When a teacher is assigned to teach a subject, they teach it to a specific year group (FY, SY, TY) in a specific classroom or division. For example:

- Mrs. Shradhha Kapoor teaches PHP to FY students in Room 105 and to SY students in Room 201
- Mr. Prakash Bhujbal (HOD, Electronics) teaches Electronics to FY in Room 105 and SY in Room 201

This classroom-per-year assignment is the foundation of timetable generation. When a HOD or admin creates a timetable slot, the system should be able to auto-fill the room number based on which teacher is teaching which year group — rather than requiring manual entry every time.

Currently, the Teacher model stores only a flat array of course IDs and a flat array of subject IDs. There is no concept of which year group a teacher teaches, which classroom they use, or which division they are assigned to.

---

## 2. Current Implementation

### 2.1 Teacher Model (Relevant Fields)

```
Teacher {
  courses: [ObjectId]    // flat array — no year, no room, no division
  subjects: [ObjectId]   // flat array — no semester, no year context
}
```

### 2.2 TimetableSlot Model (Relevant Fields)

```
TimetableSlot {
  teacher_id: ObjectId
  subject_id: ObjectId
  course_id: ObjectId
  day: String
  startTime: String
  endTime: String
  room: String           // free text — manually entered every time
  slotType: LECTURE | LAB
}
```

### 2.3 Current Problem

- Room number is free text on each timetable slot — no validation, no auto-fill
- No way to detect if a room is double-booked at the same time
- No concept of "Division A" vs "Division B" within the same year group
- HOD creating a timetable must manually type the room number for every slot
- No way to filter "which rooms are available at 10am on Monday"

---

## 3. Indian College Context

In Indian colleges, classrooms are typically assigned to a course-year combination for the entire academic year:

```
BCS FY  → Room 105, Division A (capacity: 60 students)
BCS SY  → Room 201, Division B (capacity: 60 students)
BCS TY  → Room 302, Division A (capacity: 60 students)
```

A teacher who teaches multiple year groups moves between these rooms. The room assignment is stable for the year — it does not change slot by slot.

Some colleges also have divisions within a year (Division A and Division B of FY BCS), each with their own room.

---

## 4. Options Evaluated

### Option A — Structured Course Assignments on Teacher Model (Recommended)

Replace the flat `courses[]` array on the Teacher model with a structured `courseAssignments` array:

```
Teacher {
  courseAssignments: [
    {
      course_id: ObjectId,
      year: Number,           // 1, 2, 3
      yearLabel: String,      // "FY", "SY", "TY"
      classroom: String,      // "105"
      division: String        // "A", "B" (optional)
    }
  ]
}
```

| Aspect | Detail |
|--------|--------|
| Auto-fill | When HOD selects teacher + course + year in timetable builder, room auto-fills |
| Conflict detection | System can check if a room is already booked at a given time |
| Migration needed | Yes — existing `courses[]` data must be migrated to `courseAssignments[]` |
| Complexity | Medium |
| Benefit | Single source of truth for teacher-room-year mapping |

---

### Option B — Room on TimetableSlot Only (No Change to Teacher Model)

Keep the Teacher model as-is. Room remains free text on each TimetableSlot.

| Aspect | Detail |
|--------|--------|
| Auto-fill | Not possible |
| Conflict detection | Possible but requires scanning all slots — no indexed room data |
| Migration needed | None |
| Complexity | Low |
| Downside | HOD must type room number for every slot. No validation. Errors likely. |

---

### Option C — Separate ClassroomAllocation Model

Create a new model that maps course + year + time range to a classroom:

```
ClassroomAllocation {
  college_id: ObjectId,
  course_id: ObjectId,
  year: Number,
  yearLabel: String,
  classroom: String,
  division: String,
  academicYear: String,   // "2025-2026"
  isActive: Boolean
}
```

| Aspect | Detail |
|--------|--------|
| Auto-fill | Possible — look up by course + year |
| Conflict detection | Possible |
| Migration needed | None (new model) |
| Complexity | Medium — new model, new API, new admin UI |
| Downside | Classroom assignment is separate from teacher assignment — two places to manage |

---

## 5. Recommendation

**Option A** — Structured course assignments on the Teacher model.

Reasons:
- The classroom assignment is inherently tied to the teacher-course-year relationship, not to the course alone
- A single teacher may teach the same course in different rooms depending on the year group
- Auto-fill in timetable builder directly reduces HOD workload
- Conflict detection becomes straightforward with indexed room data

---

## 6. Proposed Data Model Change

### 6.1 Teacher Model

```
// Replace:
courses: [ObjectId]

// With:
courseAssignments: [
  {
    course_id: ObjectId (ref: Course),
    year: Number,
    yearLabel: String,
    classroom: String,
    division: String        // optional
  }
]
```

The existing `subjects[]` array remains unchanged — it stores which subjects the teacher is qualified to teach, independent of year/room.

### 6.2 TimetableSlot Model

The `room` field remains on TimetableSlot but is now auto-populated from `courseAssignments` when a slot is created. It can still be manually overridden for exceptions (e.g., a lab session in a different room).

---

## 7. Timetable Builder UI Changes

When HOD creates a timetable slot:

```
Step 1: Select Teacher
        → Dropdown shows only teachers in HOD's department

Step 2: Select Subject
        → Dropdown shows only subjects assigned to selected teacher

Step 3: Select Course + Year
        → Dropdown shows course-year combinations from teacher's courseAssignments
        → Room field auto-fills from courseAssignments

Step 4: Select Day + Time
        → System checks: is this room already booked at this time?
        → System checks: is this teacher already booked at this time?

Step 5: Confirm
        → Slot created with room pre-filled
        → Admin can override room if needed
```

---

## 8. Conflict Detection Rules

| Conflict Type | Rule | Error Message |
|---------------|------|---------------|
| Room double-booking | Same room, same day, overlapping time | "Room 105 is already booked for FY BCS at 10:00–11:00" |
| Teacher double-booking | Same teacher, same day, overlapping time | "Mrs. Kapoor is already scheduled at 10:00–11:00" |
| Course-year double-booking | Same course+year, same day, overlapping time | "FY BCS already has a class at 10:00–11:00" |

---

## 9. Impact on Existing Code

| File | Change Required |
|------|----------------|
| `backend/src/models/teacher.model.js` | Replace `courses[]` with `courseAssignments[]` |
| `backend/src/controllers/teacher.controller.js` | Accept `courseAssignments` in create/update |
| `backend/src/controllers/timetableSlot.controller.js` | Auto-fill room from teacher's courseAssignments |
| `backend/src/controllers/timetableSlot.controller.js` | Add conflict detection on slot creation |
| `frontend` — AddTeacher.jsx | Replace course multi-select with courseAssignments form |
| `frontend` — EditTeacher.jsx | Update to courseAssignments form |
| `frontend` — Timetable builder | Auto-fill room, add conflict warning |
| `backend/scripts/migrate.js` | Migrate existing `courses[]` to `courseAssignments[]` (room/division will be blank — admin fills in) |

---

## 10. Open Questions — Requires Decision

| # | Question | Options | Impact |
|---|----------|---------|--------|
| Q1 | Who assigns classrooms to teachers — College Admin at teacher creation, or HOD at timetable creation? | Admin / HOD / Both | Affects which UI gets the courseAssignments form |
| Q2 | Can one classroom be shared by two different courses at different times? | Yes / No | Affects conflict detection scope |
| Q3 | Is "Division" (A/B/C) a required field or optional? | Required / Optional | Affects model validation |
| Q4 | Can a teacher be assigned to the same course-year in two different classrooms (e.g., theory in 105, lab in Lab 2)? | Yes / No | Affects courseAssignments structure |
| Q5 | Should room conflict detection be a hard block (cannot save) or a soft warning (can override)? | Hard block / Soft warning | Affects UX and admin flexibility |
| Q6 | Should classroom assignments reset every academic year or carry forward? | Reset / Carry forward | Affects academic year management |

---

## 11. Migration Plan

Existing teachers have `courses[]` populated. Migration steps:

1. For each teacher, create a `courseAssignments` entry for each course in `courses[]`
2. Set `year` to null, `classroom` to null, `division` to null (admin fills in after migration)
3. Keep `courses[]` as a deprecated field for 1 sprint, then remove
4. Admin UI shows a "Complete Setup" prompt for teachers with incomplete courseAssignments

---

## 12. Sign-Off Required

| Role | Name | Decision | Date |
|------|------|----------|------|
| Product Owner | | ☐ Approved ☐ Changes Needed | |
| Technical Lead | | ☐ Approved ☐ Changes Needed | |
| Development Team | | ☐ Acknowledged | |

---

**Document Owner**: Technical Team
**Next Review**: Before sprint planning
