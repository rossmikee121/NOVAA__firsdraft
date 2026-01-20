# PRODUCT REQUIREMENTS DOCUMENT (PRD) - NOVAA MVP

**Document Version**: 1.0  
**Date**: January 20, 2026  
**Status**: DRAFT - Ready for Product Team Review  

---

## 1. EXECUTIVE SUMMARY

NOVAA MVP is a focused, 12-week delivery of a college management system targeting 5 Maharashtra colleges. It solves three core pain points:

1. **Admissions**: Replace Excel-based admission tracking with instant status updates
2. **Payments**: Eliminate fee reconciliation headaches with GST-compliant, instant receipts
3. **Attendance**: Remove paper sign-up sheets with one-tap QR scanning

**Out of Scope**: State-specific rules, mobile app, offline mode, APAAR integration

---

## 2. PRODUCT VISION & PRINCIPLES

### 2.1 Vision
A secure, multi-tenant platform where colleges eliminate 70% of manual administrative work while maintaining complete data sovereignty.

### 2.2 Core Principles

| Principle | What It Means | Why It Matters |
|-----------|---------------|----------------|
| **Security First** | Data isolation by design, not retrofit | Prevents catastrophic data breaches |
| **Compliance by Default** | GST/DPDPA rules baked into workflows | Avoids ₹50+ lakh fines per violation |
| **Operational Simplicity** | No feature requires more than 3 clicks | Busy college staff will use it |
| **Tenant Independence** | Colleges control their own destiny | No shared configuration disasters |
| **Zero Downtime Deployment** | Updates without interrupting colleges | Admissions can't wait for patches |

---

## 3. USER PERSONAS

### 3.1 College Administrator (Dr. Sharma, 55)
**Context**: Manages 2,000 students, 50+ staff at St. Xavier's Mumbai
**Goals**:
- Reduce admission processing time from 30 days to 3 days
- Ensure every payment is tracked for audit compliance
- Get real-time dashboard of daily operations
- Handle 70 new admission applications on "last day" without system crashing

**Pain Points**:
- "I spend 3 hours every morning manually verifying documents"
- "Last year we lost ₹5 lakhs in fees due to payment reconciliation errors"
- "I have no visibility into which students are at risk of dropout"

**Success Metric**: 95% of applications approved/rejected within 48 hours

### 3.2 Teaching Staff (Prof. Desai, 35)
**Context**: Teaches 5 courses (120 students total) across multiple sections
**Goals**:
- Mark attendance in <2 minutes for entire class
- Get automatic alerts for students with poor attendance
- No manual spreadsheets by end of semester

**Pain Points**:
- "I spend 15 minutes each class just taking attendance"
- "One student has attended only 3 classes but I don't know until exam time"
- "Tracking attendance in Excel is error-prone and I lose data constantly"

**Success Metric**: QR scanning system marks 50 students in <90 seconds

### 3.3 Student (Aditya, 19)
**Context**: First-year student, parents anxious about admission confirmation
**Goals**:
- Know admission status in real-time (not waiting 2 weeks)
- Get instant fee receipt for parents' records
- Attend class and prove it with QR scan (no physical token)

**Pain Points**:
- "I paid fees 5 days ago but still don't have a receipt"
- "My parents keep asking if I got admitted – I don't know the status"
- "I got marked absent last week but I was definitely in class"

**Success Metric**: Status updates visible within 4 hours of action

---

## 4. FEATURE SPECIFICATIONS

### 4.1 USER MANAGEMENT & AUTHENTICATION

#### Feature: College Registration & Multi-Tenancy
```
ACCEPTANCE CRITERIA:
✅ New college can be added without code deployment
✅ Each college gets unique URL (e.g., app.novaa.in/st-xaviers)
✅ College data completely isolated from others
✅ College admin sees only their college's data
✅ System prevents querying other colleges even with modified URLs
```

**User Story 1.1**: As a NOVAA admin, I want to onboard a new college so that they can start using the system immediately.

**Acceptance Criteria**:
- [ ] College registration form collects: name, principal email, state (Maharashtra only for MVP)
- [ ] System generates unique college code (auto-generated, e.g., "STXM001")
- [ ] Admin credentials auto-created and emailed to principal
- [ ] College dashboard accessible within 1 hour of registration
- [ ] First login shows onboarding wizard (3 steps max)

**Effort Estimate**: 2 days (Backend Intern + Database Intern)

---

#### Feature: Role-Based Access Control (RBAC)
```
ROLES & PERMISSIONS (MVP):

ADMIN Role:
- Can approve/reject admissions
- Can verify documents
- Can create staff accounts
- Can view all reports
- CAN'T modify GST rates (hardcoded for MVP)
- CAN'T see student passwords

STAFF Role:
- Can mark attendance via QR
- Can view their own class's attendance
- CAN'T approve admissions
- CAN'T see fee details

STUDENT Role:
- Can apply for admission
- Can upload documents
- Can pay fees
- Can view own attendance
- CAN'T see other students' data
- CAN'T modify anything
```

**User Story 1.2**: As a college admin, I want to create staff accounts and assign them attendance-marking duties only so that they can't accidentally access sensitive financial data.

**Acceptance Criteria**:
- [ ] Admin can create new staff account with email
- [ ] Staff account creation sends reset password link
- [ ] Staff role automatically restricts to "Mark Attendance" only
- [ ] Audit log records who created staff account and when
- [ ] Attempt to access fee data shows "Unauthorized" error

**Effort Estimate**: 1.5 days (Backend Intern)

---

### 4.2 ADMISSIONS MODULE

#### Feature: Digital Application Form

**User Story 2.1**: As a student, I want to fill an online admission form so that I don't have to visit the college multiple times.

**Form Fields**:
```
Personal Information:
- Full Name (required, min 5 chars)
- Email (required, valid format)
- Phone (required, 10 digits)
- Aadhaar Number (required, validates format)
- Date of Birth (required)

Educational History:
- Class 12 School Name
- Class 12 Score (required, 0-100)
- Class 12 Percentage (required, 0-100)
- Marksheet File Upload (required, PDF/JPEG only)

Reservation Details:
- Caste Category (required, dropdown)
  Options: General, SC, ST, OBC, EWS
- If non-General: Certificate file (conditional upload)

Course Selection:
- Preferred Course (required, dropdown)
  Options: [Populate from college's course list]
```

**Acceptance Criteria**:
- [ ] Form validates all required fields before submission
- [ ] File uploads check file type (reject .exe, .zip, etc.)
- [ ] Student sees clear error messages (not technical jargon)
- [ ] Form auto-saves every 30 seconds
- [ ] Student can resume if session times out
- [ ] Submission creates unique Application ID displayed to student

**Effort Estimate**: 2.5 days (Frontend Intern + Backend Intern)

---

#### Feature: Document Verification Workflow

**User Story 2.2**: As an admin, I want to review uploaded documents so that I can approve genuine students and reject fakes.

**Verification States**:
```
SUBMITTED ──> PENDING_VERIFICATION ──> APPROVED or REJECTED
```

**Workflow**:
1. Admin views "Pending Verifications" dashboard (sorted by date)
2. Clicks on student to see:
   - Uploaded Aadhaar image
   - Uploaded Marksheet image
   - Student's entered data
3. For each document:
   - Checks if image is clear and legible
   - Compares data with entered information
   - Clicks "Approve" or "Reject with Reason"
4. If all documents approved → Admission moves to "VERIFIED"
5. If any document rejected → Admission moves to "REJECTED"
6. Student receives SMS: "Your [document] has been [APPROVED/REJECTED]. Reason: [specific reason]"

**Acceptance Criteria**:
- [ ] Admin sees pending verifications sorted by newest first
- [ ] Document images display clearly (zoomable)
- [ ] Admin can add rejection reason (dropdown + free text)
- [ ] Student notified via SMS within 1 minute of admin action
- [ ] Rejected applications show specific reason (e.g., "Aadhaar number mismatch")
- [ ] Approval action creates audit log entry
- [ ] System prevents duplicate rejections (already rejected can't be re-rejected)

**Effort Estimate**: 2 days (Frontend Intern + Backend Intern)

---

#### Feature: Real-Time Application Status Tracking

**User Story 2.3**: As a student, I want to see my application status update in real-time so I'm not wondering if I got admitted.

**Status Display**:
```
Step 1: ✅ Application Submitted (Jan 15)
Step 2: ⏳ Documents Under Review (Jan 16-17)
Step 3: ⏳ Payment Pending (Jan 18)
Step 4: ⏳ Final Approval
```

**Student Dashboard Shows**:
- Current step highlighted
- Timestamp of each completed step
- If rejected: Red banner with specific reason
- If approved: Green confirmation with next steps

**Acceptance Criteria**:
- [ ] Status updates within 30 seconds of admin action
- [ ] Student receives SMS when status changes
- [ ] Application history visible (who did what and when)
- [ ] Status persists even if student closes browser
- [ ] Mobile-friendly status view (readable on 5-inch screen)

**Effort Estimate**: 1.5 days (Frontend Intern)

---

### 4.3 FEE MANAGEMENT & PAYMENTS

#### Feature: Fee Structure Configuration

**User Story 3.1**: As an admin, I want to define fee structure for each course so that all students pay the correct amount.

**Admin Interface**:
```
College: St. Xavier's Mumbai
Course: BSc Computer Science (2026-27)

Fee Components:
┌─────────────────────────────────────────┐
│ Component     │ Amount   │ GST Eligible? │
├─────────────────────────────────────────┤
│ Tuition Fee   │ ₹50,000  │ No (UGC)     │
│ Lab Fee       │ ₹10,000  │ Yes (18%)    │
│ Sports Fee    │ ₹2,000   │ Yes (18%)    │
│ Library Fee   │ ₹1,000   │ Yes (18%)    │
└─────────────────────────────────────────┘

Total: ₹63,000 + GST = ₹66,160
```

**Acceptance Criteria**:
- [ ] Admin can add/edit fee components per course
- [ ] GST eligibility toggle per component
- [ ] System auto-calculates total with GST
- [ ] Fee structure shows on student payment page
- [ ] Historical fee structures stored (for audit)
- [ ] Can mark structure as "Active" or "Archived"

**Effort Estimate**: 1 day (Frontend Intern + Backend Intern)

---

#### Feature: GST-Compliant Payment Processing

**User Story 3.2**: As a student, I want to pay fees and immediately get a GST-compliant receipt so my parents can file it for tax purposes.

**Payment Flow**:
```
1. Student clicks "Pay Now"
2. System shows fee breakdown:
   Tuition: ₹50,000 (Exempt)
   Lab Fee: ₹10,000 + ₹1,800 GST (18%)
   Total: ₹61,800

3. Redirect to Razorpay
4. Student completes payment (UPI/Card/Net Banking)
5. Payment success: System creates receipt
6. Receipt sent via SMS + email within 30 seconds
7. Student dashboard shows ✅ PAID
```

**GST Receipt Contents**:
```
ST XAVIER'S MUMBAI
GSTIN: 27AABCR2346H1Z5
FEE RECEIPT

Student: Aditya Sharma
Date: 2026-01-20
Amount: ₹61,800

Tax Breakdown:
Taxable Amount: ₹20,000 (Lab + Sports)
GST (18%): ₹3,600
Total Tax: ₹3,600

Payment ID: RZP_XXXXX
Status: SUCCESS
```

**Acceptance Criteria**:
- [ ] Razorpay integration with proper error handling
- [ ] GST split exactly matches CA guidelines
- [ ] Receipt PDF generated within 10 seconds
- [ ] SMS sent with receipt link within 30 seconds
- [ ] Email sent with PDF attachment within 30 seconds
- [ ] Failed payment doesn't create receipt
- [ ] Duplicate payments blocked with idempotency key
- [ ] Receipt downloadable from student dashboard anytime

**Effort Estimate**: 3 days (Backend Intern + Database Intern)

---

#### Feature: Failed Payment Recovery

**User Story 3.3**: As an admin, I want visibility into failed payments so I can manually help students and prevent revenue leakage.

**Admin Dashboard – Failed Payments**:
```
┌─ Failed Payments ───────────────────────┐
│ Status: FAILED (Last 48 hours)          │
├─────────────────────────────────────────┤
│ Student: Priya Gupta                    │
│ Amount: ₹61,800                         │
│ Attempt: 3 (01:15 PM, 01:45 PM, 02:30) │
│ Reason: Gateway timeout                 │
│ Action: [RETRY] [MANUAL_OVERRIDE]       │
│                                         │
│ Student: Vikram Singh                   │
│ Amount: ₹61,800                         │
│ Attempt: 1 (11:20 AM)                   │
│ Reason: Insufficient funds              │
│ Action: [SEND_REMINDER] [WAIVE]         │
└─────────────────────────────────────────┘
```

**Manual Override Process**:
1. Admin clicks [MANUAL_OVERRIDE]
2. System shows confirmation: "Mark as paid without actual payment?"
3. Admin provides reason (dropdown: Institutional Scholarship, Technical Issue, etc.)
4. Admin clicks confirm
5. System:
   - Creates transaction record with status "PAID_MANUAL"
   - Generates receipt identical to normal payment receipt
   - Sends SMS to student: "Your payment has been processed"
   - Logs admin action in audit trail

**Acceptance Criteria**:
- [ ] Failed payment dashboard shows last 48 hours
- [ ] Displays reason for failure (gateway response)
- [ ] Attempt count visible (3 attempts = likely stuck)
- [ ] Retry button re-triggers Razorpay payment
- [ ] Manual override requires admin confirmation
- [ ] Manual override logged with admin name + timestamp
- [ ] Can't manually override same payment twice

**Effort Estimate**: 2 days (Backend Intern)

---

### 4.4 ATTENDANCE MODULE

#### Feature: QR Code Generation & Management

**User Story 4.1**: As a staff member, I want QR codes for my classes so I don't have to maintain paper attendance sheets.

**Implementation**:
```
System generates unique QR code for each student:
QR Content: {
  "student_id": "STX_2026_12345",
  "student_name": "Aditya Sharma",
  "college": "st_xavier_mumbai",
  "unique_code": "abc123xyz"  // Changes each day for security
}

QR Code Display Options:
1. Print on student ID card (static QR)
2. Email to student as PDF (can show on phone)
3. Display on student dashboard
4. Generate on-demand if card lost
```

**Acceptance Criteria**:
- [ ] Unique QR per student per day
- [ ] QR changes daily (security measure)
- [ ] Can print in bulk for all students
- [ ] Can email individual QR to student
- [ ] QR code scannable from 1-2 feet distance
- [ ] QR valid for 24 hours (prev + current day only)

**Effort Estimate**: 1 day (Backend Intern)

---

#### Feature: QR Scanning Interface

**User Story 4.2**: As a staff member, I want to scan QR codes to mark attendance so I don't manually enter each student.

**Staff Interface**:
```
┌─ ATTENDANCE MARKING ────────────────────────┐
│ Course: BSc CS - Semester 1, Section A     │
│ Date: 2026-01-20 (Tue, 09:00-10:00 AM)    │
│                                             │
│ [📱 SCAN QR CODE]                          │
│                                             │
│ Present: 42 students                        │
│ Absent: 3 students                          │
│ ────────────────────────────────────────   │
│ Roll Call:                                  │
│ ✅ Aditya Sharma (09:02)                    │
│ ✅ Priya Gupta (09:03)                      │
│ ✅ Raj Kumar (09:01)                        │
│ ❌ Vikram Singh (ABSENT)                    │
│                                             │
│ [SUBMIT ATTENDANCE]                         │
└─────────────────────────────────────────────┘
```

**Scanning Workflow**:
1. Staff selects course/section
2. System shows today's schedule
3. Staff clicks [📱 SCAN QR CODE]
4. Camera opens
5. As QR is scanned → Student appears on list with ✅ timestamp
6. If duplicate scan → Shows "Already marked at 09:02"
7. Staff can scroll to manually mark absent students
8. After class: Click [SUBMIT ATTENDANCE]

**Acceptance Criteria**:
- [ ] Camera launches within 1 second of button click
- [ ] Scanned student appears on list within 100ms
- [ ] Prevents duplicate scans with error message
- [ ] Manual absent marking available
- [ ] Submit button disabled until ≥80% of students present/absent
- [ ] Attendance submitted creates permanent record
- [ ] Can't modify submitted attendance
- [ ] Works on tablet (landscape orientation)

**Effort Estimate**: 2.5 days (Frontend Intern + Backend Intern)

---

#### Feature: Student Attendance Dashboard

**User Story 4.3**: As a student, I want to see my attendance percentage so I know if I'm at risk of failing due to poor attendance.

**Student Dashboard**:
```
┌─ MY ATTENDANCE ─────────────────────────────┐
│ Overall: 78% (Below 75% threshold ⚠️)      │
│ Attendance deadline: May 31, 2026            │
│                                              │
│ By Course:                                   │
│ ┌────────────────────────────────────────┐  │
│ │ BSc Computer Science (Sem 1)           │  │
│ │ Present: 32/42 classes = 76%  ✅      │  │
│ │                                        │  │
│ │ Physics Lab (Sem 1)                    │  │
│ │ Present: 8/12 classes = 67%   ⚠️      │  │
│ │ Action: Attend next 3 classes         │  │
│ │                                        │  │
│ │ Calculus (Sem 1)                       │  │
│ │ Present: 18/22 classes = 82%  ✅      │  │
│ └────────────────────────────────────────┘  │
│                                              │
│ Last 7 Days:                                 │
│ Mon: ✅ Tue: ✅ Wed: ✅ Thu: ❌ Fri: ✅ ...  │
│                                              │
│ [GENERATE REPORT]                            │
└─────────────────────────────────────────────┘
```

**Alerts**:
- If attendance falls below 75% → Red warning banner
- At-risk students receive SMS: "Your attendance is 67%. You need 5 more classes."

**Acceptance Criteria**:
- [ ] Attendance percentage calculated correctly
- [ ] By-course breakdown shows detailed percentages
- [ ] Last 7 days calendar view visible
- [ ] Below-75% shows red warning
- [ ] SMS alert sent when attendance drops below 75%
- [ ] Report can be downloaded as PDF
- [ ] Mobile-friendly layout

**Effort Estimate**: 1.5 days (Frontend Intern)

---

### 4.5 REPORTING & ANALYTICS

#### Feature: Admission Funnel Dashboard

**User Story 5.1**: As an admin, I want to see the admission funnel so I understand where bottlenecks are.

**Dashboard Visualization**:
```
ADMISSION FUNNEL (Jan 1-20, 2026)

Applications Started: 156
        ↓ 94% filled form
Applications Submitted: 147
        ↓ 92% uploaded documents
Documents Uploaded: 135
        ↓ 78% admin verified
Documents Verified: 105
        ↓ 89% payments received
Payments Complete: 93
        ↓ 100% finalized
Confirmed Admissions: 93

Conversion Rate: 59.6% (93/156)
```

**Metrics Shown**:
- Total applications started
- Completed applications
- Documents pending verification
- Payments pending
- Confirmed admissions
- Drop-off at each stage

**Acceptance Criteria**:
- [ ] Funnel updated in real-time
- [ ] Filters available: Date range, Course, Caste category
- [ ] Clicking on each stage shows list of students
- [ ] Exportable to PDF/Excel
- [ ] Shows percentage drop at each stage

**Effort Estimate**: 2 days (Frontend Intern)

---

#### Feature: Fee Collection Report

**User Story 5.2**: As an admin, I want a fee collection report so I can verify all payments are accounted for.

**Report Contents**:
```
FEE COLLECTION REPORT
Period: Jan 1-20, 2026

SUMMARY:
Total Students: 1,247
Fees Collected: ₹74,82,000 (59.5%)
Fees Pending: ₹51,00,000 (40.5%)
Failed Payments: ₹2,80,000 (8 transactions)

BY COURSE:
BSc CS: 150 students, 92% paid (₹92.5L)
BSc Bio: 95 students, 45% paid (₹28.5L)
BA Economics: 180 students, 51% paid (₹34.2L)

PAYMENT METHODS:
UPI: 65% of payments
Card: 25% of payments
Net Banking: 10% of payments

OUTSTANDING PAYMENTS:
Raj Kumar: ₹61,800 (20 days overdue)
Priya Singh: ₹61,800 (15 days overdue)
[... list of all pending]
```

**Acceptance Criteria**:
- [ ] Summary shows total vs. pending
- [ ] Breakdown by course
- [ ] Payment method breakdown
- [ ] List of outstanding payments (sortable)
- [ ] Export to PDF/Excel with formatting
- [ ] Filters: Date range, Course, Payment status

**Effort Estimate**: 2 days (Frontend Intern)

---

#### Feature: Attendance Report

**User Story 5.3**: As staff/admin, I want attendance reports so I can identify at-risk students for intervention.

**Report Options**:
```
Option 1: Student Attendance Sheet (per course)
┌────────────┬───┬───┬───┬───┬───┬─────┐
│ Student    │ M │ T │ W │ T │ F │ % P │
├────────────┼───┼───┼───┼───┼───┼─────┤
│ Aditya     │ ✅│ ✅│ ✅│ ✅│ ✅│100% │
│ Priya      │ ✅│ ✅│ ❌│ ✅│ ✅│ 80% │
│ Raj        │ ✅│ ❌│ ❌│ ❌│ ✅│ 40% │
└────────────┴───┴───┴───┴───┴───┴─────┘

Option 2: Monthly Summary (department level)
BSc CS (Sem 1): Avg 82% attendance
BSc Bio (Sem 2): Avg 71% attendance (⚠️ Below 75%)
BA Econ (Sem 3): Avg 85% attendance
```

**Acceptance Criteria**:
- [ ] Per-student attendance sheet exportable
- [ ] Monthly/semester summaries available
- [ ] At-risk students (below 75%) highlighted
- [ ] Can drill-down to see individual dates
- [ ] Export to Excel with formatting

**Effort Estimate**: 1.5 days (Frontend Intern)

---

## 5. DATA & PRIVACY

### 5.1 DPDPA 2026 Compliance (MVP Minimum)

**Privacy Notice** (Shown at first login):
```
YOUR DATA IS PROTECTED
We collect the minimum data needed for college operations:
- Personal info (name, email, phone)
- Academic records (marks, attendance)
- Payment history (fee receipts only)

You can contact admin@college.novaa.in to:
✓ Download your data
✓ Request data deletion
✓ Update your information
✓ Withdraw consent

This system is audited annually for compliance.
[I AGREE]
```

**User Data Rights**:
- [ ] Users can request data export (admin provides within 7 days)
- [ ] Users can request deletion (admin deletes within 30 days)
- [ ] Audit logs show who accessed what data
- [ ] Sensitive fields (Aadhaar) never shown in non-essential screens

### 5.2 Data Retention Policy

| Data Type | Retention Period | Reason |
|-----------|------------------|--------|
| Student Records | 7 years | Academic regulation |
| Fee Receipts | 10 years | Tax/GST compliance |
| Attendance | 5 years | Accreditation requirements |
| Audit Logs | 1 year | Security investigation |

---

## 6. TECHNICAL REQUIREMENTS (FOR ARCHITECTS)

### 6.1 Platform Requirements
- Backend: Node.js 18+
- Frontend: React 18+
- Database: MongoDB 5.0+
- Payment Gateway: Razorpay
- Storage: AWS S3 for documents

### 6.2 Performance Requirements
- Page load time: <2 seconds
- QR scanning: <100ms detection
- Receipt generation: <10 seconds
- API response: <200ms for 99th percentile

### 6.3 Security Requirements
- All passwords hashed with bcrypt (salt rounds=10)
- API tokens expire in 2 hours
- Rate limiting: 5 login attempts/minute
- HTTPS for all endpoints
- Automated SQL injection scanning in CI/CD

---

## 7. SUCCESS METRICS & KPIs

| Metric | Target | Measurement |
|--------|--------|-------------|
| **Admission Processing Time** | 95% within 48 hours | Tracked in system |
| **GST Compliance** | 100% correct tax split | CA audit + auto-test |
| **Payment Success Rate** | 98% | Razorpay webhook data |
| **Attendance Accuracy** | 99% (no duplicate scans) | System validation |
| **System Uptime** | 99.5% | CloudWatch monitoring |
| **User Satisfaction** | 4.5/5 stars | Post-launch survey |

---

## 8. APPROVAL

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Product Owner | [Name] | _____ | _____ |
| Technical Lead | [Name] | _____ | _____ |
| Compliance Officer | [Name] | _____ | _____ |

---

**Document Owner**: Product Manager  
**Last Updated**: 2026-01-20  
**Next Review**: Weekly with product team
