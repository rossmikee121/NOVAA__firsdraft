# Decision 09 — Student ID Card & Hall Ticket Generation

**Document Version**: 1.0
**Date**: 2026
**Status**: PENDING DECISION
**Module**: Student Services, Document Generation
**Prepared By**: Technical Team

---

## 1. Context

Two physical documents are required at different points in the student lifecycle:

1. **Student ID Card** — Issued at the time of enrollment. Used for daily identification, library access, attendance (QR code), and campus entry. Required from Day 1 of the academic year.

2. **Hall Ticket** — Issued before each examination. Contains student details, exam schedule, seat number, and examination rules. Required before every exam cycle.

The platform already has `pdfkit` installed in the backend, which is used for payment receipt generation. The infrastructure for PDF generation exists and can be extended.

However, the two documents have different dependencies:
- ID cards can be generated immediately — all required data (student name, photo, course, year, enrollment number) already exists in the system
- Hall tickets require exam schedules and seat number assignments, which do not exist yet (no exam module)

---

## 2. Student ID Card

### 2.1 Required Data (All Available)

| Field | Source |
|-------|--------|
| Student full name | `Student.fullName` |
| Enrollment number | `Student.enrollmentNumber` |
| Course name | `Course.name` |
| Year label (FY/SY/TY) | Decision 02 — `Course.yearLabels` |
| Academic year | `Student.currentAcademicYear` |
| Department | `Department.name` |
| College name | `College.name` |
| College logo | `College.logo` |
| Student photo | `Student.passportPhoto` (uploaded during registration) |
| QR code | Generated from `Student._id` + `Student.enrollmentNumber` |
| Valid until | End of current academic year |

### 2.2 ID Card Layout

```
┌─────────────────────────────────────────┐
│  [COLLEGE LOGO]    [COLLEGE NAME]        │
│                                          │
│  [STUDENT PHOTO]   Name: Aditya Sharma  │
│                    Enroll: STX2026001   │
│                    Course: BCS          │
│                    Year: TY             │
│                    Dept: Computer Sci   │
│                    AY: 2025-2026        │
│                                          │
│  [QR CODE]         Valid Until: May 2026│
└─────────────────────────────────────────┘
```

### 2.3 Generation Options

**Option A — On-Demand Single Card**

Admin or student can generate their ID card at any time from the student profile page.

| Aspect | Detail |
|--------|--------|
| Trigger | Button click in student profile or admin student view |
| Output | Single PDF downloaded immediately |
| Complexity | Low |

**Option B — Bulk Generation**

Admin can generate ID cards for all students in a course-year group at once (e.g., all FY BCS students).

| Aspect | Detail |
|--------|--------|
| Trigger | Admin selects course + year, clicks "Generate All ID Cards" |
| Output | ZIP file containing individual PDFs, or a single multi-page PDF |
| Complexity | Medium — requires batch processing |

**Option C — Both (Recommended)**

Support both on-demand single card and bulk generation.

---

### 2.4 Who Can Generate

| Role | Can Generate |
|------|-------------|
| COLLEGE_ADMIN | Yes — single and bulk |
| ADMISSION_OFFICER | Yes — single only (for enrolled students) |
| STUDENT | Yes — own card only |
| TEACHER | No |

---

## 3. Hall Ticket

### 3.1 Required Data

| Field | Available Now | Dependency |
|-------|--------------|------------|
| Student name, enrollment, course, year | ✅ Yes | None |
| Student photo | ✅ Yes | None |
| Exam schedule (subject, date, time, venue) | ❌ No | Exam module (V1.1) |
| Seat number | ❌ No | Exam Coordinator assigns seats |
| Examination centre | ❌ No | Exam module |
| Attendance eligibility (75% rule) | ✅ Partial | Attendance data exists, eligibility check needed |
| Fee clearance status | ✅ Yes | StudentFee model |
| University exam registration number | ❌ No | External — university provides |

### 3.2 Conclusion on Hall Tickets

Hall tickets cannot be generated in V1.0 because the exam schedule, seat number, and examination centre data does not exist in the system. These are created by the Exam Coordinator role (Decision 06) using the exam module (V1.1).

**Hall ticket generation is deferred to V1.1.**

---

## 4. Recommendation

| Document | Version | Approach |
|----------|---------|----------|
| Student ID Card | V1.0 | Build now — all data available |
| Hall Ticket | V1.1 | Defer — requires exam module |

---

## 5. ID Card Technical Implementation

### 5.1 Backend

New endpoint:
```
GET /api/students/:id/id-card
  → Generates and returns PDF for single student

POST /api/students/id-cards/bulk
  Body: { course_id, year, academicYear }
  → Generates ZIP of PDFs for all matching students
```

### 5.2 PDF Generation

Using existing `pdfkit` library. ID card dimensions: standard credit card size (85.6mm × 54mm) or A4 with 4 cards per page for printing.

### 5.3 QR Code on ID Card

The QR code on the ID card encodes:
```json
{
  "studentId": "...",
  "enrollmentNumber": "STX2026001",
  "collegeCode": "stx_mumbai"
}
```

This QR code is static (does not change daily). It is used for library access and campus entry — not for attendance marking. Attendance uses a separate daily-rotating QR (already implemented).

### 5.4 Photo Handling

Student passport photo is uploaded during registration and stored in the uploads directory. The ID card generator fetches this file by path. If no photo is uploaded, a placeholder image is used.

---

## 6. Impact on Existing Code

| File | Change Required |
|------|----------------|
| `backend/src/controllers/student.controller.js` | Add `generateIdCard` and `bulkGenerateIdCards` functions |
| `backend/src/routes/student.routes.js` | Add ID card generation routes |
| `backend/src/utils/` | Create `idCardGenerator.js` utility using pdfkit |
| `frontend` — ViewStudent.jsx | Add "Download ID Card" button |
| `frontend` — StudentProfile.jsx | Add "Download My ID Card" button |
| `frontend` — College Admin student list | Add "Generate ID Cards" bulk action |

---

## 7. Open Questions — Requires Decision

| # | Question | Options | Impact |
|---|----------|---------|--------|
| Q1 | What is the ID card size — credit card (85.6×54mm) or A4 with multiple cards? | Credit card / A4 | Affects PDF layout |
| Q2 | Should the ID card include the student's QR code for attendance? | Yes / No | Affects QR code type (static vs daily-rotating) |
| Q3 | Should bulk ID card generation produce one PDF per student (ZIP) or all on one multi-page PDF? | ZIP / Multi-page PDF | Affects bulk generation implementation |
| Q4 | Should the college logo be printed on the ID card? | Yes / No | Affects layout and logo upload requirement |
| Q5 | Should the ID card have an expiry date (end of academic year)? | Yes / No | Affects card content |
| Q6 | Should students be able to download their own ID card, or only admin can generate it? | Both / Admin only | Affects student dashboard |
| Q7 | Is a physical card printing workflow needed (e.g., print queue, batch print), or just PDF download? | PDF only / Print queue | Affects complexity significantly |

---

## 8. Hall Ticket — V1.1 Requirements (For Planning)

When the exam module is built in V1.1, hall tickets will require:

1. **Exam Schedule Model** — subject, date, time, venue, duration
2. **Seat Assignment** — Exam Coordinator assigns seat numbers to students
3. **Eligibility Check** — student must have ≥75% attendance and fee clearance
4. **Hall Ticket Model** — links student to exam schedule + seat number
5. **Generation** — PDF with all exam details, student photo, rules

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
