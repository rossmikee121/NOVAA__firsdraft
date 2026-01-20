# MODULE ARCHITECTURE - How NOVAA is Organized

**For**: All Developers  
**Version**: 1.0  
**Date**: January 20, 2026  

---

## 🏗️ NOVAA MODULE STRUCTURE

NOVAA is organized into **6 core modules**, each responsible for a specific feature set:

```
┌────────────────────────────────────────────────────────┐
│                      NOVAA CORE MODULES                │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Module 1: AUTH (Authentication & Access Control)     │
│  ├─ User login/registration                           │
│  ├─ JWT token generation                              │
│  ├─ Role-based access control                         │
│  └─ College context switching                         │
│                                                        │
│  Module 2: COLLEGES (Tenant Management)               │
│  ├─ College registration                              │
│  ├─ College settings configuration                    │
│  ├─ Staff management                                  │
│  └─ Data isolation enforcement                        │
│                                                        │
│  Module 3: ADMISSIONS (Student Applications)          │
│  ├─ Application form submission                       │
│  ├─ Document upload & verification                    │
│  ├─ Application tracking                              │
│  └─ Approval workflows                                │
│                                                        │
│  Module 4: PAYMENTS (Fee Management)                  │
│  ├─ Fee structure configuration                       │
│  ├─ Razorpay payment processing                       │
│  ├─ GST calculation & receipt generation              │
│  └─ Payment reconciliation                            │
│                                                        │
│  Module 5: ATTENDANCE (Attendance Tracking)           │
│  ├─ QR code generation                                │
│  ├─ Attendance marking                                │
│  ├─ Attendance reports                                │
│  └─ At-risk student detection                         │
│                                                        │
│  Module 6: REPORTS (Analytics & Dashboards)           │
│  ├─ Admission funnel analytics                        │
│  ├─ Fee collection dashboard                          │
│  ├─ Attendance analytics                              │
│  └─ Custom report generation                          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## 📋 MODULE 1: AUTH (Authentication & Access Control)

### What Does It Do?

The AUTH module handles:
- User login and registration
- JWT token generation and verification
- Role-based access (Admin, Staff, Student)
- Session management
- Password security

### Module Structure

```
backend/src/modules/auth/
├── routes.js                 # API endpoints
├── controller.js             # Request handlers
├── service.js                # Business logic
├── model.js                  # User schema
├── middleware.js             # Custom middleware
└── validation.js             # Input validation

Example Endpoints:
POST   /api/auth/register     (Create new user)
POST   /api/auth/login        (Authenticate user)
POST   /api/auth/logout       (End session)
GET    /api/auth/profile      (Get logged-in user)
PUT    /api/auth/change-password (Security)
```

### Frontend Components

```
frontend/src/pages/Auth/
├── Login.js                  # Login form
├── Register.js               # Registration form
├── ForgotPassword.js         # Password recovery
└── AuthGuard.js              # Protected routes

Uses:
- AuthContext (stores token, user data)
- useAuth hook (easy access to auth state)
```

### Database Collections Used

```
users collection:
{
  _id: ObjectId,
  collegeId: ObjectId (CRITICAL for multi-tenancy),
  email: String (unique per college),
  password: String (bcrypt hashed),
  role: String (ADMIN, STAFF, STUDENT),
  name: String,
  lastLogin: Date,
  isActive: Boolean
}

Key Index: (collegeId, email) unique
```

### Key Security Features

```javascript
// Password Hashing
const bcrypt = require('bcrypt');
const hashedPassword = await bcrypt.hash(password, 10); // 10 salt rounds

// JWT Token
const token = jwt.sign(
  { userId, collegeId },
  process.env.JWT_SECRET,
  { expiresIn: '2h' }
);

// Rate Limiting (prevents brute force)
loginLimiter: 5 attempts per minute per email

// Multi-tenancy Enforcement
Every user tied to collegeId
Can't access other college's data
```

### Development Checklist

- [ ] User registration works with password validation
- [ ] Login returns JWT token with 2-hour expiry
- [ ] Password hashed with bcrypt (salt=10)
- [ ] Every user has collegeId attached
- [ ] Rate limiting prevents brute force (5/min)
- [ ] Token refresh endpoint available
- [ ] Logout invalidates token
- [ ] Test with different roles (Admin/Staff/Student)

---

## 📋 MODULE 2: COLLEGES (Tenant Management)

### What Does It Do?

The COLLEGES module handles:
- College registration and setup
- College-specific configurations
- Staff member management
- Data isolation enforcement
- College dashboard settings

### Module Structure

```
backend/src/modules/colleges/
├── routes.js                 # API endpoints
├── controller.js             # Request handlers
├── service.js                # Business logic
├── model.js                  # College schema
└── validation.js             # Input validation

Example Endpoints:
POST   /api/colleges/register     (Register new college)
GET    /api/colleges/:id          (Get college details)
PUT    /api/colleges/:id          (Update settings)
POST   /api/colleges/:id/staff    (Add staff member)
GET    /api/colleges/:id/staff    (List staff)
DELETE /api/colleges/:id/staff/:staffId (Remove staff)
```

### Frontend Components

```
frontend/src/pages/Admin/
├── CollegeSettings.js        # Configure college
├── StaffManagement.js        # Add/remove staff
└── DashboardConfig.js        # Dashboard setup
```

### Database Collections Used

```
colleges collection:
{
  _id: ObjectId,
  code: String (unique, e.g., "ST_XAVIER_MUMBAI"),
  name: String,
  state: String,
  principalEmail: String,
  principalPhone: String,
  gstNumber: String,
  isActive: Boolean,
  createdAt: Date
}

Key Index: code (unique)
Key Index: (state, isActive)
```

### Critical: Multi-Tenancy Enforcement

```javascript
// EVERY module must enforce college context
// This is the bedrock of data isolation

// Middleware example:
const enforceCollegeContext = async (req, res, next) => {
  const collegeCode = req.headers['x-college-code'];
  
  if (!collegeCode) {
    return res.status(400).json({ error: 'MISSING_COLLEGE_CODE' });
  }
  
  const college = await College.findOne({ 
    code: collegeCode,
    isActive: true 
  });
  
  if (!college) {
    return res.status(403).json({ error: 'INVALID_COLLEGE' });
  }
  
  // Attach to request so all handlers can use it
  req.college = college;
  next();
};

// Used in EVERY route:
app.get('/api/admissions', enforceCollegeContext, authenticate, async (req, res) => {
  // req.college is guaranteed to exist here
  // All queries automatically filtered by collegeId
});
```

### Development Checklist

- [ ] College registration captures all required fields
- [ ] Code generation is unique (no duplicates)
- [ ] College context middleware enforced on all routes
- [ ] Staff members can be added with role selection
- [ ] Cannot query other college's data
- [ ] College settings saved and retrieved correctly
- [ ] Multi-tenancy tests pass (data isolation)

---

## 📋 MODULE 3: ADMISSIONS (Student Applications)

### What Does It Do?

The ADMISSIONS module handles:
- Student application submissions
- Document upload and storage
- Admin verification workflow
- Application status tracking
- Reservation management

### Module Structure

```
backend/src/modules/admissions/
├── routes.js                 # API endpoints
├── controller.js             # Request handlers
├── service.js                # Business logic
├── model.js                  # Application schema
├── validation.js             # Input validation
└── documentUpload.js         # File handling

Example Endpoints:
POST   /api/admissions/apply           (Submit application)
GET    /api/admissions/:id             (Get app details)
PUT    /api/admissions/:id/verify      (Admin verification)
GET    /api/admissions/:id/documents   (View documents)
PATCH  /api/admissions/:id/status      (Update status)
GET    /api/admissions?status=PENDING  (List pending apps)
```

### Frontend Components

```
frontend/src/pages/
├── Student/ApplicationForm.js    # Student fills form
├── Student/ApplicationStatus.js  # Track status
├── Admin/VerifyDocuments.js      # Admin verification
└── Admin/ApplicationsList.js     # Pending list
```

### Data Flow: Student Applies for Admission

```
STEP 1: Frontend (React)
└─ Student fills form on ApplicationForm.js
   • Enters: name, email, phone, caste category
   • Uploads: Aadhaar, Marksheet
   • Clicks: Submit

STEP 2: Frontend Validation
└─ Check before sending to backend
   • Email valid format?
   • File size < 5MB?
   • All required fields filled?

STEP 3: Backend (Node.js)
└─ POST /api/admissions/apply

   Middleware Chain:
   ├─ enforceCollegeContext (attach req.college)
   ├─ authenticate (verify JWT)
   └─ validateApplicationInput (check data)

   Controller (controller.js):
   ├─ Extract form data
   ├─ Call service.submitApplication()
   └─ Return response

   Service (service.js):
   ├─ Validate business rules
   ├─ Upload documents to AWS S3
   ├─ Create application document in MongoDB
   ├─ Generate confirmation email
   └─ Return result

STEP 4: Database Update
└─ MongoDB admissions collection
   {
     _id: ObjectId,
     collegeId: req.college.id, // CRITICAL
     studentId: ObjectId,
     status: "SUBMITTED",
     documents: [
       { url: "s3://...", category: "AADHAAR", status: "PENDING" },
       { url: "s3://...", category: "MARKSHEET", status: "PENDING" }
     ],
     createdAt: Date.now()
   }

STEP 5: Frontend Response
└─ Response received
   ├─ Update component state
   ├─ Show success message with Application ID
   └─ Redirect to status tracking page

STEP 6: Admin Verification (Later)
└─ Admin logs in
   ├─ Sees "Pending Verifications" dashboard
   ├─ Reviews uploaded documents
   ├─ Clicks "Approve" or "Reject"
   └─ System updates database + notifies student
```

### Database Collections Used

```
admissions collection:
{
  _id: ObjectId,
  collegeId: ObjectId,
  studentId: ObjectId,
  courseId: String,
  status: String (DRAFT, SUBMITTED, VERIFIED, APPROVED, REJECTED),
  documents: [
    {
      url: String (S3 path),
      category: String (AADHAAR, MARKSHEET),
      verificationStatus: String (PENDING, APPROVED, REJECTED),
      rejectionReason: String
    }
  ],
  createdAt: Date
}

Index: (collegeId, status)
Index: (collegeId, studentId)

students collection:
{
  _id: ObjectId,
  collegeId: ObjectId,
  userId: ObjectId,
  name: String,
  email: String,
  phone: String,
  casteCategory: String
}

Index: (collegeId, userId)
```

### Document Upload Flow

```javascript
// Backend (Node.js + Multer + AWS S3)

const multer = require('multer');
const AWS = require('aws-sdk');
const s3 = new AWS.S3();

// 1. Multer middleware (validates file on server before upload)
const upload = multer({
  limits: { fileSize: 5 * 1024 * 1024 }, // 5MB
  fileFilter: (req, file, cb) => {
    if (!['image/jpeg', 'image/png', 'application/pdf'].includes(file.mimetype)) {
      return cb(new Error('Invalid file type'));
    }
    cb(null, true);
  }
});

// 2. Upload to S3
app.post('/api/admissions/upload-document', upload.single('document'), async (req, res) => {
  const params = {
    Bucket: 'novaa-documents',
    Key: `admissions/${req.college.id}/${req.file.filename}`,
    Body: req.file.buffer,
    ContentType: req.file.mimetype,
    ServerSideEncryption: 'AES256' // Security
  };
  
  const result = await s3.upload(params).promise();
  res.json({ url: result.Location });
});

// 3. Frontend receives S3 URL and includes in form submission
```

### Development Checklist

- [ ] Application form validates all fields
- [ ] Documents uploaded to S3 with correct path structure
- [ ] Application status transitions work correctly
- [ ] Admin verification updates database + notifies student
- [ ] Rejection reason captured and visible to student
- [ ] GST data collected (for future payments)
- [ ] Can't submit duplicate applications
- [ ] All queries include collegeId filter

---

## 📋 MODULE 4: PAYMENTS (Fee Management)

### What Does It Do?

The PAYMENTS module handles:
- Fee structure configuration
- Razorpay payment gateway integration
- GST calculation and invoice generation
- Payment reconciliation
- Failed payment recovery

### Module Structure

```
backend/src/modules/payments/
├── routes.js                 # API endpoints
├── controller.js             # Request handlers
├── service.js                # Business logic
├── model.js                  # Payment schema
├── validation.js             # Input validation
├── razorpay.js               # Payment gateway integration
└── gst.js                    # GST calculations

Example Endpoints:
GET    /api/payments/fee-structure/:courseId  (Get fee details)
POST   /api/payments/create-order             (Initiate payment)
POST   /api/payments/webhook                   (Razorpay callback)
GET    /api/payments/receipt/:transactionId   (Download receipt)
GET    /api/payments?status=PENDING            (Payment dashboard)
PUT    /api/payments/:id/mark-paid            (Manual override)
```

### Frontend Components

```
frontend/src/pages/
├── Student/PaymentPage.js         # Pay fees
├── Student/ReceiptDownload.js     # Get receipt
├── Admin/FeeStructure.js          # Configure fees
└── Admin/PaymentDashboard.js      # Track payments
```

### GST Calculation Logic

```javascript
// CRITICAL: GST split must be correct for legal compliance

const calculateGST = (feeComponents) => {
  return {
    tuitionFee: 50000,           // GST EXEMPT (UGC rule)
    labFee: 10000,               // 18% GST applicable
    sportsFee: 2000,             // 18% GST applicable
    
    // Tax breakdown
    taxableAmount: 12000,        // labFee + sportsFee
    gstRate: 18,
    gstAmount: 2160,             // 12000 * 0.18
    
    // Total
    totalAmount: 50000 + 12000 + 2160 = 64160
  };
};

// Invoice receipt shows:
/*
TUITION FEE        ₹50,000  (Exempt)
LAB FEE            ₹10,000
  + GST (18%)      ₹1,800
SPORTS FEE         ₹2,000
  + GST (18%)      ₹360

TOTAL TAX          ₹2,160
TOTAL AMOUNT       ₹64,160
*/

// Store this in database for audit
```

### Payment Flow with Razorpay

```
CLIENT                          SERVER                    RAZORPAY
  │                              │                          │
  ├─ Click Pay ──────────────────→│                          │
  │                              │                          │
  │                        Create Order                      │
  │                         (pending)                        │
  │                              │                          │
  │                              ├──────────────────────────→│
  │                              │   Create Order Response   │
  │                              │←──────────────────────────┤
  │                              │                          │
  │  Redirect to Razorpay ←──────┤                          │
  │──────────────────────────────────────────────────────────→│
  │                              │                          │
  │  User enters card/UPI details                           │
  │  Razorpay processes payment                             │
  │                              │                          │
  │  Success/Failure response                               │
  │←──────────────────────────────────────────────────────────┤
  │                              │                          │
  │  Webhook: Payment succeeded                             │
  │                              │←─────────────────────────┤
  │                              │  (Signature verified)    │
  │                              │                          │
  │                        Update Transaction               │
  │                         (success)                        │
  │                              │                          │
  │  Generate Receipt ←──────────┤                          │
  │  (PDF + email)               │                          │
  │                              │                          │
  │  User sees ✅ Payment successful                         │
  │                              │                          │
```

### Database Collections Used

```
feeStructures collection:
{
  _id: ObjectId,
  collegeId: ObjectId,
  courseId: String,
  tuitionFee: Number (GST exempt),
  labFee: Number (18% GST),
  sportsFee: Number (18% GST),
  effectiveDate: Date
}

transactions collection:
{
  _id: ObjectId,
  collegeId: ObjectId,
  admissionId: ObjectId,
  studentId: ObjectId,
  amount: Number,
  razorpayOrderId: String,
  razorpayPaymentId: String,
  status: String (PENDING, SUCCESS, FAILED),
  idempotencyKey: String (CRITICAL - prevents duplicates),
  receiptUrl: String (S3 path to PDF),
  gstSplit: {
    taxableAmount: Number,
    gstRate: Number,
    gstAmount: Number
  },
  createdAt: Date
}

Key Index: (collegeId, razorpayOrderId)
Key Index: idempotencyKey (unique)
Key Index: (collegeId, studentId, status)
```

### Development Checklist

- [ ] Fee structure editable per course
- [ ] GST calculation matches CA guidelines (verified)
- [ ] Razorpay webhook signature verified
- [ ] Payment idempotency prevents duplicates
- [ ] Receipt PDF generated within 10 seconds
- [ ] Receipt emailed to student + admin
- [ ] Failed payments visible on admin dashboard
- [ ] Manual override requires admin password
- [ ] All transactions logged with collegeId

---

## 📋 MODULE 5: ATTENDANCE (Attendance Tracking)

### What Does It Do?

The ATTENDANCE module handles:
- QR code generation for students
- Attendance marking via QR scan
- Real-time attendance dashboard
- At-risk student detection
- Monthly/semester reports

### Module Structure

```
backend/src/modules/attendance/
├── routes.js                 # API endpoints
├── controller.js             # Request handlers
├── service.js                # Business logic
├── model.js                  # Attendance schema
├── qr.js                     # QR code generation
└── validation.js             # Input validation

Example Endpoints:
POST   /api/attendance/mark              (Mark presence)
GET    /api/attendance/qr/:studentId     (Get QR)
GET    /api/attendance/dashboard         (Student view)
GET    /api/attendance/reports           (Admin reports)
GET    /api/attendance/at-risk           (At-risk students)
```

### Frontend Components

```
frontend/src/pages/
├── Staff/AttendanceMarking.js    # QR Scanner for staff
├── Student/AttendanceDashboard.js # Student attendance view
└── Admin/AttendanceReports.js     # Reports & analytics
```

### QR Code Generation

```javascript
// Each student gets a unique QR code

// QR Content (encoded in image):
{
  studentId: "STX_2026_12345",
  collegCode: "ST_XAVIER_MUMBAI",
  timestamp: Date.now(),
  validUntil: Date.now() + 24*60*60*1000 // 24 hours
}

// Generate on backend:
const QRCode = require('qrcode');
const qrData = JSON.stringify({
  studentId: "STX_2026_12345",
  collegeCode: "ST_XAVIER_MUMBAI"
});
const qrImage = await QRCode.toDataURL(qrData);

// Display on frontend:
<img src={qrImage} alt="Student QR" />

// Or print:
Print QR and put on student ID card
```

### Attendance Marking Flow

```
STAFF MARKS ATTENDANCE (React Component)

1. Open AttendanceMarking.js
   ├─ Select Course: "BSc CS - Semester 1"
   ├─ Select Section: "Section A"
   ├─ Show date/time

2. Click "Start QR Scanner"
   ├─ Camera permission requested
   ├─ Live camera preview
   ├─ "Scanning..." indicator

3. Student scans QR
   ├─ QR decoded
   ├─ studentId extracted
   ├─ API call: POST /api/attendance/mark
   │
   │ Backend:
   │ ├─ Verify student exists
   │ ├─ Check for duplicate scan today
   │ ├─ Create attendance record
   │ ├─ Add timestamp
   │ └─ Return success
   │
   ├─ ✅ "Aditya Sharma marked present at 09:02"
   └─ Student added to list

4. After all students scanned
   ├─ Manual entry for absent students
   ├─ Click [SUBMIT ATTENDANCE]
   └─ Entire class saved to database

STUDENT VIEWS ATTENDANCE (React Component)

1. Student logs in
2. Goes to Dashboard
3. Sees attendance by course
4. "Physics: 18/22 classes = 82% ✅"
5. "Chemistry: 12/20 classes = 60% ⚠️"
6. Can view day-by-day calendar
```

### Database Collections Used

```
attendance collection:
{
  _id: ObjectId,
  collegeId: ObjectId,
  studentId: ObjectId,
  courseId: String,
  date: Date,
  status: String (PRESENT, ABSENT, LEAVE),
  qrScannedAt: Date,
  markedBy: ObjectId (staff who marked),
  createdAt: Date
}

Key Index: (collegeId, studentId, date) unique
Key Index: (collegeId, date)
Key Index: (collegeId, studentId, status)
```

### At-Risk Student Detection

```javascript
// Identify students below 75% attendance

const getAtRiskStudents = async (collegeId, courseId) => {
  const students = await Student.find({ collegeId });
  
  const atRiskList = await Promise.all(
    students.map(async (student) => {
      const totalClasses = await Attendance.countDocuments({
        courseId,
        studentId: student._id,
        status: { $in: ['PRESENT', 'ABSENT', 'LEAVE'] }
      });
      
      const attendedClasses = await Attendance.countDocuments({
        courseId,
        studentId: student._id,
        status: 'PRESENT'
      });
      
      const percentage = (attendedClasses / totalClasses) * 100;
      
      if (percentage < 75) {
        return {
          student,
          attendance: percentage,
          neededClasses: Math.ceil((75 * totalClasses - attendedClasses * 100) / 25)
        };
      }
    })
  );
  
  return atRiskList.filter(x => x); // Remove null values
};

// Send alerts to at-risk students
// "You attended only 12 out of 20 classes (60%). You need 3 more classes to meet 75%."
```

### Development Checklist

- [ ] QR generation works per student
- [ ] QR scanner handles valid codes
- [ ] Prevents duplicate scans (same student in same class)
- [ ] Manual absent entry works
- [ ] Attendance percentage calculated correctly
- [ ] Alerts sent when attendance <75%
- [ ] Reports generate monthly summaries
- [ ] All scans logged in database
- [ ] Queries include collegeId filter

---

## 📋 MODULE 6: REPORTS (Analytics & Dashboards)

### What Does It Do?

The REPORTS module handles:
- Admission funnel analytics
- Fee collection dashboards
- Attendance trends
- Custom report generation
- Exporting to PDF/Excel

### Module Structure

```
backend/src/modules/reports/
├── routes.js                 # API endpoints
├── controller.js             # Request handlers
├── service.js                # Business logic & aggregations
└── formatters.js             # PDF/Excel generation

Example Endpoints:
GET    /api/reports/admission-funnel     (Conversion metrics)
GET    /api/reports/fee-collection       (Revenue dashboard)
GET    /api/reports/attendance-trends    (Attendance analytics)
GET    /api/reports/export?format=pdf    (Export data)
GET    /api/reports/export?format=excel  (Export data)
```

### Frontend Components

```
frontend/src/pages/Admin/
├── AdmissionFunnel.js         # Visualization
├── FeeCollectionDashboard.js  # Revenue metrics
├── AttendanceTrends.js        # Charts
└── ExportReports.js           # Download reports
```

### Report Examples

**Admission Funnel Report**:
```
Total Applications: 156
└─ Completed Applications: 147 (94%)
   └─ Verified Documents: 135 (92%)
      └─ Payment Complete: 93 (69%)
         └─ Confirmed: 93 (100%)

Conversion Rate: 59.6% (93/156)
Drop-off Stages:
- Form submission: 6% (9 students)
- Document verification: 8% (12 students)
- Payment: 31% (42 students)
```

**Fee Collection Report**:
```
Total Students: 1,247
Fees Collected: ₹74,82,000 (59.5%)
Fees Pending: ₹51,00,000 (40.5%)
Failed Payments: ₹2,80,000 (8 transactions)

By Course:
- BSc CS: 150 students, 92% paid
- BSc Bio: 95 students, 45% paid
- BA Economics: 180 students, 51% paid

Payment Methods:
- UPI: 65%
- Card: 25%
- Net Banking: 10%

Outstanding (>30 days overdue): 23 students
```

### Database Aggregation Example

```javascript
// MongoDB aggregation pipeline to get admission funnel

const admissionFunnel = await Admission.aggregate([
  {
    $match: { collegeId: req.college._id }
  },
  {
    $group: {
      _id: '$status',
      count: { $sum: 1 }
    }
  },
  {
    $sort: { _id: 1 }
  }
]);

// Result:
[
  { _id: 'APPROVED', count: 93 },
  { _id: 'DRAFT', count: 20 },
  { _id: 'PENDING_VERIFICATION', count: 15 },
  { _id: 'SUBMITTED', count: 28 }
]

// Calculate percentages on backend
const totalApps = admissionFunnel.reduce((sum, x) => sum + x.count, 0);
const funnel = admissionFunnel.map(item => ({
  status: item._id,
  count: item.count,
  percentage: ((item.count / totalApps) * 100).toFixed(2)
}));
```

### Development Checklist

- [ ] Admission funnel calculated correctly
- [ ] Fee collection aggregates by course
- [ ] Attendance trends show monthly breakdown
- [ ] PDF export includes college header/logo
- [ ] Excel export includes all relevant columns
- [ ] Queries use MongoDB aggregation (efficient)
- [ ] All reports filtered by collegeId
- [ ] Reports generated on-demand (not pre-cached)

---

## 🔗 HOW MODULES INTERCONNECT

### Module Dependencies

```
                 ┌─────────────┐
                 │   AUTH      │  (Foundation - all modules depend on this)
                 └────┬────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
   ┌─────────┐  ┌──────────┐  ┌──────────┐
   │ COLLEGES │  │ADMISSIONS│  │ PAYMENTS │
   └────┬────┘  └────┬─────┘  └────┬─────┘
        │            │             │
        │            │             │
        └────────────┼─────────────┘
                     │
        ┌────────────┴────────────┐
        ▼                         ▼
   ┌──────────────┐          ┌────────────┐
   │ ATTENDANCE   │          │  REPORTS   │
   └──────────────┘          └────────────┘
```

### Example: Student Pays Fee (Cross-Module Flow)

```
1. ADMISSIONS module
   └─ Student completes admission
   └─ Admission status = "VERIFIED"

2. PAYMENTS module
   └─ Student clicks "Pay Fees"
   └─ Retrieves fee structure for student's course
   └─ Calculates GST
   └─ Presents to Razorpay
   └─ Payment successful
   └─ Creates transaction record
   └─ Updates admission status to "APPROVED"
   └─ Generates receipt PDF

3. REPORTS module
   └─ Admin views fee dashboard
   └─ Fee collection data comes from PAYMENTS
   └─ Shows "93 out of 147 students paid" (from ADMISSIONS)
   └─ Displays "Collection rate: 63.3%"

4. ATTENDANCE module
   └─ Once admission approved, student gets attendance rights
   └─ Can mark attendance via QR
   └─ Records added to attendance collection

5. Back to REPORTS module
   └─ Admin checks "At-Risk Students"
   └─ Pulls data from ATTENDANCE module
   └─ Shows which students have low attendance
```

### Data Passing Between Modules

```javascript
// Example 1: From ADMISSIONS to PAYMENTS

// admissions/service.js
const admission = await Admission.findById(admissionId);
const student = await Student.findById(admission.studentId);

// Pass to payments module
const feeInfo = {
  studentId: student._id,
  admissionId: admission._id,
  courseId: admission.courseId
};

// payments/controller.js
const feeStructure = await FeeStructure.findOne({
  collegeId: req.college.id,
  courseId: feeInfo.courseId
});

// Example 2: From PAYMENTS to REPORTS

// payments/model.js saves transaction
await Transaction.create({
  collegeId: req.college.id,
  admissionId,
  studentId,
  amount,
  status: 'SUCCESS'
});

// reports/service.js queries transactions
const totalPaid = await Transaction.countDocuments({
  collegeId: req.college.id,
  status: 'SUCCESS'
});

const totalStudents = await Admission.countDocuments({
  collegeId: req.college.id,
  status: 'APPROVED'
});

const collectionRate = (totalPaid / totalStudents) * 100;
```

---

## 📊 MODULE INTERACTION MATRIX

| From ▼ To → | AUTH | COLLEGES | ADMISSIONS | PAYMENTS | ATTENDANCE | REPORTS |
|-----------|------|----------|-----------|----------|-----------|---------|
| **AUTH** | - | ✅ (user role) | ✅ (user role) | ✅ (user role) | ✅ (user role) | ✅ (user role) |
| **COLLEGES** | - | - | ✅ (college context) | ✅ (college context) | ✅ (college context) | ✅ (college context) |
| **ADMISSIONS** | - | - | - | ✅ (admission ID) | ✅ (student ID) | ✅ (admission count) |
| **PAYMENTS** | - | - | ✅ (update status) | - | - | ✅ (payment data) |
| **ATTENDANCE** | - | - | ✅ (student list) | - | - | ✅ (attendance data) |
| **REPORTS** | - | - | - | - | - | - |

---

## 🎯 NEXT DOCUMENTS IN DEVELOPER GUIDES

1. ✅ **01_MERN_STACK_OVERVIEW.md** (You are here)
2. ✅ **02_DEVELOPMENT_ENVIRONMENT_SETUP.md** (Coming next)
3. ✅ **03_MODULES_ARCHITECTURE.md** (This document)
4. ✅ **04_MODULE_INTERCONNECTIONS.md** (Detailed cross-module flows)
5. **05_DATABASE_DEVELOPER_GUIDE.md**
6. **06_API_DEVELOPMENT_GUIDE.md**
7. **07_FRONTEND_DEVELOPMENT_GUIDE.md**
8. **08_CODE_STANDARDS_CONVENTIONS.md**
9. **09_AUTHENTICATION_SECURITY_GUIDE.md**
10. **10_PAYMENT_PROCESSING_GUIDE.md**
11. **11_TESTING_DEVELOPER_GUIDE.md**
12. **12_DEBUGGING_TROUBLESHOOTING.md**

---

**Next Document**: [04_MODULE_INTERCONNECTIONS.md](04_MODULE_INTERCONNECTIONS.md)

