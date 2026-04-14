# Decision 02 — Course Year Labels (FY / SY / TY / Final Year)

**Document Version**: 1.0
**Date**: 2026
**Status**: PENDING DECISION
**Module**: Course Management, Student Management, Timetable, Promotion
**Prepared By**: Technical Team

---

## 1. Context

Indian colleges do not refer to academic progress by semester number in day-to-day operations. A student enrolled in a 3-year Bachelor of Computer Science program is referred to as being in "TY BCS" (Third Year Bachelor of Computer Science), not "Semester 5 BCS."

This terminology is used across:
- Student identity (ID cards, hall tickets, certificates)
- Timetable display ("FY BCS — Room 105")
- Promotion ("Promote from SY to TY")
- Attendance reports ("TY students below 75%")
- Fee structures (some colleges charge different fees per year, not per semester)

The current system stores `durationSemesters` (e.g., 6) and auto-calculates `durationYears` (3) but has no concept of year labels. All UI references use semester numbers.

---

## 2. Current Implementation

### 2.1 Course Model (Relevant Fields)

```
Course {
  durationSemesters: Number  // e.g., 6
  durationYears: Number      // auto-calculated: ceil(semesters / 2) = 3
  programLevel: UG | PG | DIPLOMA | PHD
}
```

### 2.2 Student Model (Relevant Fields)

```
Student {
  currentSemester: Number    // e.g., 5
  currentYear: Number        // auto-calculated: ceil(semester / 2) = 3
  currentAcademicYear: String // e.g., "2025-2026"
}
```

### 2.3 Promotion Logic

Promotion increments `currentSemester` by 1. `currentYear` is recalculated automatically. No year label is stored or displayed.

---

## 3. The Problem

| What System Shows | What College Staff Expects |
|-------------------|---------------------------|
| "Semester 5" | "TY" or "Third Year" |
| "Year 3" | "TY" |
| "Promote to Semester 6" | "Promote to Final Year" |
| Timetable: "Course: BCS, Sem 1" | "FY BCS — Room 105" |

Additionally, the fixed formula `year = ceil(semester / 2)` breaks for non-standard programs:
- A 2-year PG program has FY and SY only
- A 4-year engineering program has FY, SY, TY, Final Year
- Some diploma programs have 3 semesters in 1.5 years

---

## 4. Options Evaluated

### Option A — Fixed Auto-Generated Labels

Auto-generate labels based on year number:
- Year 1 → "FY"
- Year 2 → "SY"
- Year 3 → "TY"
- Year 4 → "Final Year"

| Aspect | Detail |
|--------|--------|
| Implementation effort | Very low — just a display utility function |
| Flexibility | Low — breaks for programs that use different terminology |
| Migration needed | None |
| Works for | Standard 3-year UG programs |
| Fails for | PG programs, diploma, engineering (4 years), programs that say "First Year" instead of "FY" |

---

### Option B — Admin-Defined Labels at Course Creation (Recommended)

Admin defines year labels when creating a course:

```
Course: Bachelor of Computer Science
Total Semesters: 6
Year Mapping:
  Year 1 (Sem 1-2) → Label: "FY"
  Year 2 (Sem 3-4) → Label: "SY"
  Year 3 (Sem 5-6) → Label: "TY"
```

| Aspect | Detail |
|--------|--------|
| Implementation effort | Medium — new field on Course model, updated course creation UI |
| Flexibility | High — works for any program structure |
| Migration needed | Yes — existing courses need labels added (can default to FY/SY/TY/Final Year) |
| Works for | All program types |
| Fails for | Nothing — fully configurable |

---

### Option C — System Defaults with Admin Override

Auto-generate FY/SY/TY/Final Year as defaults. Allow admin to rename if needed.

| Aspect | Detail |
|--------|--------|
| Implementation effort | Medium |
| Flexibility | High |
| Migration needed | None — defaults apply to existing courses |
| Works for | All program types |
| Best of both | Reduces admin effort while allowing customisation |

---

## 5. Recommendation

**Option B** — Admin-defined labels at course creation.

Reasons:
- Indian colleges have varied terminology. Some say "FY", others say "First Year", some engineering colleges say "SE" (Second Engineering) instead of "SY"
- Labels appear on official documents (ID cards, hall tickets, certificates) — they must be exactly what the college uses
- One-time setup effort at course creation is acceptable

---

## 6. Proposed Data Model Change

### 6.1 Course Model Addition

```
yearLabels: [
  {
    year: Number,          // 1, 2, 3, 4
    label: String,         // "FY", "SY", "TY", "Final Year"
    semesters: [Number]    // [1, 2], [3, 4], [5, 6]
  }
]
```

### 6.2 Helper Function

A utility function `getYearLabel(course, semester)` returns the label for a given semester. Used across timetable, attendance, promotion, and reports.

### 6.3 Where Labels Are Displayed

| Location | Current Display | New Display |
|----------|----------------|-------------|
| Student profile | "Semester 5" | "TY — Semester 5" |
| Timetable slot | "Sem 1, BCS" | "FY BCS" |
| Attendance report | "Year 3 students" | "TY students" |
| Promotion screen | "Promote to Semester 6" | "Promote to TY (Sem 6)" |
| Student ID card | "Year 3" | "TY" |
| Hall ticket | "Semester 5" | "TY — Semester 5" |

---

## 7. Impact on Existing Code

| File | Change Required |
|------|----------------|
| `backend/src/models/course.model.js` | Add `yearLabels` array field |
| `backend/src/controllers/course.controller.js` | Accept `yearLabels` in create/update |
| `backend/src/utils/` | Add `getYearLabel(course, semester)` utility |
| `backend/src/controllers/promotion.controller.js` | Use year label in promotion response and history |
| `frontend` — AddCourse.jsx | Add year label input fields |
| `frontend` — EditCourse.jsx | Add year label edit fields |
| `frontend` — all student views | Display year label alongside semester |
| `frontend` — timetable views | Display year label in slot headers |
| `backend/scripts/migrate.js` | Add default FY/SY/TY/Final Year labels to existing courses |

---

## 8. Migration Plan

For all existing courses, auto-generate default labels based on `durationYears`:

| durationYears | Default Labels Generated |
|---------------|------------------------|
| 1 | Year 1 → "FY" |
| 2 | Year 1 → "FY", Year 2 → "SY" |
| 3 | Year 1 → "FY", Year 2 → "SY", Year 3 → "TY" |
| 4 | Year 1 → "FY", Year 2 → "SY", Year 3 → "TY", Year 4 → "Final Year" |

Colleges can update labels after migration if their terminology differs.

---

## 9. Open Questions — Requires Decision

| # | Question | Options | Impact |
|---|----------|---------|--------|
| Q1 | Should year labels appear on student ID cards? | Yes / No | Affects ID card generator spec |
| Q2 | Should year labels appear on hall tickets? | Yes / No | Affects hall ticket generator spec |
| Q3 | Should fee structures be defined per year-label or per semester? | Per year / Per semester | Affects FeeStructure model |
| Q4 | Should promotion be triggered by year (FY → SY) or by semester (Sem 2 → Sem 3)? | By year / By semester | Affects promotion controller logic |
| Q5 | Can a college have a course with an odd number of semesters (e.g., 3 semesters for a diploma)? | Yes / No | Affects year-label mapping logic |

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
