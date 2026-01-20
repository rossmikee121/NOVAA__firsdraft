# MODULE INTERCONNECTIONS - How Modules Communicate

**For**: All Developers  
**Version**: 1.0  
**Date**: January 20, 2026  

---

## 🔌 OVERVIEW

Modules in NOVAA don't work in isolation. They communicate via:

1. **Direct Database Queries** - One module reads data written by another
2. **API Calls** - Backend-to-backend synchronous communication
3. **Event-Driven Updates** - Status changes trigger cascading updates
4. **Shared Services** - Utility functions used by multiple modules

---

## 🔄 CRITICAL INTERCONNECTION FLOWS

### FLOW 1: Complete Admission Process (All 6 Modules)

**Timeline**: Student fills form → Admin verifies → Payment → Confirmed

```
Day 1: Student Submits Application

AUTH Module (Foundation)
├─ Student logs in
├─ JWT token attached to all requests
└─ Every request includes collegeId validation

        │ (authenticated request)
        ▼

ADMISSIONS Module
├─ Student fills application form
├─ Uploads documents to S3
├─ Creates admission record with status="SUBMITTED"
└─ Database stores:
   {
     _id: AdmissionID_001,
     collegeId: "ST_XAVIER_MUMBAI",
     studentId: StudentID_001,
     courseId: "BSC_CS",
     status: "SUBMITTED",
     documents: [S3 URLs],
     createdAt: 2026-01-20
   }

        │ (admission created)
        ▼

REPORTS Module
├─ Admin views "Admissions Dashboard"
├─ Queries: Admissions.find({ collegeId, status: "SUBMITTED" })
├─ Dashboard updates: "23 applications pending verification"
└─ Admin sees AdmissionID_001 in list


Day 2-3: Admin Verifies Documents

ADMISSIONS Module
├─ Admin logs in (role check by AUTH)
├─ Opens VerifyDocuments.js
├─ Reviews AdmissionID_001 documents
├─ Clicks "Approve"
├─ Updates admission record:
   status: "SUBMITTED" → "VERIFIED"
└─ Database updates:
   {
     _id: AdmissionID_001,
     status: "VERIFIED",  // Changed
     verifiedAt: 2026-01-21,
     verifiedBy: StaffID_002
   }

        │ (admission verified)
        ▼

PAYMENTS Module
├─ Student sees PaymentPage.js
├─ System queries: Admissions.findById(AdmissionID_001)
├─ Checks status == "VERIFIED"
├─ Retrieves fee structure for course "BSC_CS":
   {
     tuitionFee: 50000,     (GST exempt)
     labFee: 10000,         (18% GST = ₹1,800)
     sportsFee: 2000,       (18% GST = ₹360)
     totalGST: 2160,
     totalAmount: 64160
   }
├─ Displays fee details
└─ Student clicks "Pay Now"

        │ (payment initiated)
        ▼

PAYMENTS Module (Razorpay Integration)
├─ Creates Razorpay order
├─ Stores transaction record:
   {
     _id: TransactionID_001,
     admissionId: AdmissionID_001,
     studentId: StudentID_001,
     amount: 64160,
     status: "PENDING",
     razorpayOrderId: "order_xyz",
     idempotencyKey: "unique_key_123",  // Prevents duplicates
     createdAt: 2026-01-21
   }
├─ Redirects to Razorpay payment page
├─ Student pays
└─ Razorpay sends webhook


Day 4: Payment Webhook Received

PAYMENTS Module (Webhook Handler)
├─ Endpoint: POST /api/payments/webhook
├─ Verifies webhook signature (security)
├─ Extracts: razorpayOrderId, razorpayPaymentId, amount
├─ Updates transaction:
   {
     _id: TransactionID_001,
     status: "SUCCESS",  // Changed from PENDING
     razorpayPaymentId: "pay_abc123",
     confirmedAt: 2026-01-21 14:30:00
   }
├─ Triggers post-payment flow
└─ Sends success email to student

        │ (transaction successful)
        ▼

ADMISSIONS Module (Linked Update)
├─ Service listens for payment success event
├─ Updates admission:
   {
     _id: AdmissionID_001,
     status: "APPROVED"  // Changed from VERIFIED
   }
├─ Updates student confirmed status
└─ Sends "Admission Approved" email

        │ (admission approved)
        ▼

ATTENDANCE Module
├─ Student now eligible to mark attendance
├─ System enables attendance features
├─ Generates student QR code for attendance
├─ Stores: StudentID_001 is in course "BSC_CS"
└─ Ready for attendance marking

        │ (student confirmed)
        ▼

REPORTS Module
├─ Admin views "Admission Funnel"
├─ Queries:
   ├─ Total apps: 156
   ├─ Verified: 135
   ├─ Payments successful: 93
   ├─ Approved: 93
├─ Calculates: "59.6% conversion rate"
└─ Dashboard shows all metrics updated with AdmissionID_001
```

---

### FLOW 2: Attendance Marking → At-Risk Alert

**Timeline**: Teacher marks attendance → System detects at-risk student

```
Class Time: Teacher marks attendance

ATTENDANCE Module
├─ Staff opens AttendanceMarking.js
├─ Scans QR for StudentID_001
├─ Creates attendance record:
   {
     _id: AttendanceID_123,
     collegeId: "ST_XAVIER_MUMBAI",
     studentId: StudentID_001,
     courseId: "BSC_CS",
     date: 2026-01-20,
     status: "PRESENT",
     qrScannedAt: 2026-01-20 09:02:00
   }
└─ Attendance marked

        │ (attendance marked)
        ▼

REPORTS Module (At-Risk Detection)
├─ Runs calculation:
   
   Total classes for StudentID_001 in "BSC_CS": 22
   Classes present: 12
   Attendance percentage: 12/22 = 54.5%
   
   If percentage < 75%:
   ├─ Add to "At-Risk Students" list
   └─ Generate alert

├─ Database check (last 5 attendances):
   [PRESENT, PRESENT, ABSENT, PRESENT, ABSENT]
   
   Trend: Getting worse (more absences recently)

└─ Alert sent:
   "Your attendance is 54.5%. You need 5 more classes to reach 75%."

        │ (alert generated)
        ▼

ADMISSIONS Module (Historical Context)
├─ Query: When was StudentID_001 admitted?
├─ Course: BSC_CS
├─ Join with ATTENDANCE data
└─ Report context: Shows admission date and current status
```

---

### FLOW 3: Failed Payment Recovery

**Timeline**: Student payment fails → System retries → Manual override if needed

```
Initial Payment Attempt

PAYMENTS Module
├─ Student initiates payment
├─ Razorpay processes → FAILED (insufficient funds)
├─ Stores transaction:
   {
     _id: TransactionID_002,
     admissionId: AdmissionID_001,
     studentId: StudentID_001,
     amount: 64160,
     status: "FAILED",
     failureReason: "INSUFFICIENT_FUNDS",
     razorpayErrorCode: "BAD_REQUEST_ERROR",
     retryCount: 0,
     nextRetryAt: 2026-01-21 + 15 minutes
   }
├─ Shows error to student
└─ Email: "Payment failed. We'll retry in 15 minutes."


15 Minutes Later: Automatic Retry

PAYMENTS Module (Scheduled Job - runs every 15 min)
├─ Query failed transactions with retryCount < 3
├─ For TransactionID_002:
   ├─ Recreates Razorpay order
   ├─ Retries payment
   ├─ If success:
   │  └─ Update status to "SUCCESS"
   └─ If fails again:
      ├─ Increment retryCount: 0 → 1
      ├─ Update nextRetryAt: +15 minutes
      └─ Try again next cycle


If Payment Fails 3 Times: Manual Override

PAYMENTS Module (Admin Dashboard)
├─ Admin views "Failed Payments"
├─ Sees TransactionID_002 (3 retry failures)
├─ Click "Manual Override → Mark as Paid"
├─ Requires password verification (security)
├─ Update:
   {
     _id: TransactionID_002,
     status: "MANUAL_PAID",  // Special status
     overrideBy: AdminID_001,
     overrideAt: 2026-01-21 15:00:00,
     reason: "Bank delay - student will pay offline"
   }

        │ (manual override)
        ▼

ADMISSIONS Module
├─ Detects payment status changed
├─ Updates admission:
   status: "VERIFIED" → "APPROVED"
└─ Sends confirmation email


REPORTS Module
├─ "Fee Collection Dashboard" updated
├─ Shows: Override marked payment
└─ Manual payments tracked separately for audit
```

---

## 📡 API COMMUNICATION BETWEEN MODULES

### Direct API Calls

Some modules make synchronous API calls to other modules:

```javascript
// Example: ADMISSIONS → PAYMENTS (Check if fees collected)

// admissions/service.js
const checkIfPaidAndApprove = async (admissionId, collegeId) => {
  try {
    // Call PAYMENTS module API
    const response = await fetch(
      'http://localhost:5000/api/payments/check-paid',
      {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'x-internal-token': process.env.INTERNAL_API_TOKEN
        },
        body: JSON.stringify({
          admissionId,
          collegeId
        })
      }
    );
    
    const { isPaid } = await response.json();
    
    if (isPaid) {
      // Update admission to APPROVED
      await Admission.updateOne(
        { _id: admissionId },
        { status: 'APPROVED' }
      );
    }
  } catch (error) {
    console.error('Inter-module API call failed:', error);
    // Handle gracefully
  }
};
```

### Database-Level Communication

Most module communication happens through shared database collections:

```javascript
// PAYMENTS writes:
await Transaction.create({
  admissionId: AdmissionID,
  status: 'SUCCESS'
});

// Later, ADMISSIONS reads:
const transaction = await Transaction.findOne({ admissionId });
if (transaction.status === 'SUCCESS') {
  // Update admission
}
```

---

## 🔐 MULTI-TENANCY ENFORCEMENT AT INTERCONNECTIONS

**Critical Rule**: Every cross-module communication must include `collegeId`

```javascript
// ✅ CORRECT - Includes collegeId
const admissions = await Admission.find({
  collegeId: req.college.id,
  status: 'VERIFIED'
});

// ❌ WRONG - Missing collegeId
const admissions = await Admission.find({
  status: 'VERIFIED'
});
// This could return data from OTHER colleges!

// ✅ CORRECT - When joining collections
const admissionsWithPayments = await Admission.aggregate([
  {
    $match: {
      collegeId: req.college.id,  // Filter here
      status: 'VERIFIED'
    }
  },
  {
    $lookup: {
      from: 'transactions',
      let: { admissionId: '$_id', college: '$collegeId' },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ['$admissionId', '$$admissionId'] },
                { $eq: ['$collegeId', '$$college'] }  // Also filter here!
              ]
            }
          }
        }
      ],
      as: 'payment'
    }
  }
]);
```

---

## 📊 DATA FLOW DIAGRAMS

### Diagram 1: Student Admission Workflow (All Modules)

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  AUTH: Student Logs In                                          │
│  └─ Token issued, collegeId attached                            │
│                                                                 │
│  ↓                                                              │
│                                                                 │
│  ADMISSIONS: Submit Application                                │
│  ├─ Form data validated                                        │
│  ├─ Documents uploaded to S3                                   │
│  ├─ Admission record created (status=SUBMITTED)                │
│  └─ collegeId stored in every record                           │
│                                                                 │
│  ↓ (Admin verifies documents)                                  │
│                                                                 │
│  ADMISSIONS: Admin Verifies                                    │
│  ├─ Updates status → VERIFIED                                  │
│  └─ Locks further document changes                             │
│                                                                 │
│  ↓                                                              │
│                                                                 │
│  PAYMENTS: Fee Collection                                      │
│  ├─ Query: admission.status == VERIFIED                        │
│  ├─ Fetch fee structure for course                             │
│  ├─ Calculate GST                                              │
│  ├─ Create Razorpay order                                      │
│  ├─ Payment successful → status=SUCCESS                        │
│  └─ Generate receipt                                           │
│                                                                 │
│  ↓                                                              │
│                                                                 │
│  ADMISSIONS: Mark Approved                                     │
│  ├─ Query: transaction.status == SUCCESS                       │
│  ├─ Update admission → APPROVED                                │
│  └─ Send approval email                                        │
│                                                                 │
│  ↓                                                              │
│                                                                 │
│  ATTENDANCE: Enable QR                                         │
│  ├─ Student now eligible for attendance                        │
│  ├─ Generate QR code for student                               │
│  └─ Add to class roster                                        │
│                                                                 │
│  ↓ (Throughout semester)                                       │
│                                                                 │
│  ATTENDANCE: Mark Attendance                                   │
│  ├─ Daily QR scans create attendance records                   │
│  └─ Each record stores: collegeId, studentId, date, status    │
│                                                                 │
│  ↓ (Periodic - daily/weekly)                                  │
│                                                                 │
│  REPORTS: Analytics                                            │
│  ├─ Query ADMISSIONS: total students = 150                     │
│  ├─ Query PAYMENTS: fees collected from 120 students           │
│  ├─ Query ATTENDANCE: average attendance = 78%                 │
│  ├─ Generate dashboard metrics                                 │
│  └─ Export reports                                             │
│                                                                 │
│  ↓ (Final - end of semester)                                  │
│                                                                 │
│  REPORTS: Semester Summary                                     │
│  ├─ All 6 modules contribute data                              │
│  ├─ Generate comprehensive report                              │
│  └─ Archive for compliance                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

### Diagram 2: Fee Collection Waterfall

```
Student Completes Admission

        ↓
        
Admission Status: VERIFIED
(Ready for payment)

        ↓ [PAYMENTS module]
        
Create Transaction (status=PENDING)
└─ idempotencyKey = unique_id (prevents double payment)

        ↓ [Student action]
        
Razorpay Payment Page

        ├─ SUCCESS → Razorpay webhook
        │           └─ Verify signature
        │           └─ Update Transaction (status=SUCCESS)
        │           └─ Generate Receipt PDF
        │           └─ Update Admission (status=APPROVED)
        │           └─ Emit "AdmissionApproved" event
        │
        ├─ FAILED → Razorpay webhook
        │          └─ Update Transaction (status=FAILED)
        │          └─ Schedule retry (15 min later)
        │          └─ Show error to student
        │
        └─ TIMEOUT → No webhook
                    └─ Scheduled job queries pending
                    └─ Checks Razorpay API for status
                    └─ Verifies with DB (idempotency)
                    └─ Updates if complete
                    └─ Retries if failed


Result: Database state is consistent
- Transaction updated
- Admission updated
- Receipt generated
- All with collegeId context
```

---

### Diagram 3: Attendance to Report

```
QR Scanned

        ↓ [ATTENDANCE Module]
        
Create Attendance Record
└─ collegeId, studentId, courseId, date, status, timestamp

        ↓ [REPORTS Module - Periodic Aggregation]
        
Run Daily At-Risk Calculation

        ├─ For each student in college:
        │  ├─ Count: Total classes attended
        │  ├─ Condition: attendance < 75%?
        │  └─ Add to alert list
        │
        ├─ Students at risk:
        │  ├─ StudentID_001: 54% (12 out of 22) → EMAIL ALERT
        │  ├─ StudentID_003: 68% (15 out of 22) → EMAIL ALERT
        │  └─ StudentID_007: 72% (18 out of 25) → EMAIL ALERT
        │
        └─ Dashboard Update:
           ├─ "23 students at risk"
           ├─ Shows trending data
           └─ Ready for admin action

        ↓ [Admin Response]
        
Admin views dashboard
└─ Contacts at-risk students
```

---

## ⚙️ SHARED UTILITIES (Used by Multiple Modules)

Some code is shared between modules:

```javascript
// utils/validators.js (used by all modules)
const validateEmail = (email) => {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
};

const validatePhone = (phone) => {
  return /^[0-9]{10}$/.test(phone);
};

// In admissions/validation.js
const { validateEmail } = require('../../utils/validators');

// In colleges/validation.js
const { validateEmail, validatePhone } = require('../../utils/validators');


// utils/gst.js (used by PAYMENTS)
const calculateGST = (feeComponents) => {
  return {
    tuitionFee: feeComponents.tuitionFee,
    taxableAmount: feeComponents.labFee + feeComponents.sportsFee,
    gstRate: 18,
    gstAmount: (feeComponents.labFee + feeComponents.sportsFee) * 0.18,
    total: feeComponents.tuitionFee + 
           feeComponents.labFee + 
           feeComponents.sportsFee +
           ((feeComponents.labFee + feeComponents.sportsFee) * 0.18)
  };
};


// utils/emailService.js (used by all modules)
const sendEmail = async (to, subject, template, data) => {
  // Send via SMTP
  // Used by:
  // - AUTH (login confirmation, password reset)
  // - ADMISSIONS (application received, approval)
  // - PAYMENTS (receipt, payment reminder)
  // - ATTENDANCE (low attendance alert)
};


// middleware/collegeContext.js (used by ALL modules)
const enforceCollegeContext = async (req, res, next) => {
  const collegeCode = req.headers['x-college-code'];
  
  if (!collegeCode) {
    return res.status(400).json({ error: 'MISSING_COLLEGE_CODE' });
  }
  
  const college = await College.findOne({ code: collegeCode, isActive: true });
  
  if (!college) {
    return res.status(403).json({ error: 'INVALID_COLLEGE' });
  }
  
  req.college = college;
  next();
};

// Applied to EVERY route:
app.use(enforceCollegeContext);
```

---

## 🚨 COMMON INTERCONNECTION ISSUES

### Issue 1: Missing collegeId in Cross-Module Query

```javascript
// ❌ Problem:
const admissions = await Admission.find({ studentId });
// This could return admissions from OTHER colleges!

// ✅ Solution:
const admissions = await Admission.find({
  collegeId: req.college.id,
  studentId
});
```

### Issue 2: Using Stale Data from Another Module

```javascript
// ❌ Problem:
// admissions/controller.js
const feeStructure = await FeeStructure.findOne({ courseId });
// What if fee structure was just changed?

// ✅ Solution:
// Always query latest data at point of use
const feeStructure = await FeeStructure.findOne({
  collegeId: req.college.id,
  courseId,
  effectiveDate: { $lte: new Date() }
}).sort({ effectiveDate: -1 });
```

### Issue 3: Idempotency (Duplicate Payments)

```javascript
// ❌ Problem:
// If Razorpay sends webhook twice (network retry), payment processed twice

// ✅ Solution:
const Transaction = require('./transaction');

const processPayment = async (razorpayOrderId, razorpayPaymentId, amount) => {
  // Use idempotency key
  const existing = await Transaction.findOne({
    razorpayPaymentId
  });
  
  if (existing) {
    return existing; // Already processed
  }
  
  // First time, create
  const transaction = await Transaction.create({
    razorpayOrderId,
    razorpayPaymentId,
    amount,
    status: 'SUCCESS'
  });
  
  return transaction;
};
```

### Issue 4: Cascading Updates on Interdependent Data

```javascript
// ❌ Problem:
// If admission status changed, but payment not updated

// ✅ Solution:
// Emit events for important state changes

// admissions/service.js
const approveAdmission = async (admissionId) => {
  // Update admission
  const admission = await Admission.updateOne(
    { _id: admissionId },
    { status: 'APPROVED' }
  );
  
  // Emit event for other modules
  eventBus.emit('admission:approved', {
    admissionId,
    studentId: admission.studentId,
    courseId: admission.courseId
  });
};

// attendance/service.js
eventBus.on('admission:approved', async (event) => {
  // Enable attendance for this student
  await Student.updateOne(
    { _id: event.studentId },
    { attendanceEnabled: true }
  );
});
```

---

## 📋 MODULE INTERCONNECTION CHECKLIST

### When Creating New Endpoints

- [ ] All queries include `collegeId` filter
- [ ] Response includes only data for current college
- [ ] Cross-module calls include `x-college-code` header
- [ ] Idempotency key for payment-related operations
- [ ] Event emitted for state changes
- [ ] Tests include multi-college scenarios

### When Querying Another Module's Data

- [ ] Verify collegeId consistency
- [ ] Check data freshness (not cached stale data)
- [ ] Handle missing data gracefully (soft error)
- [ ] Log cross-module queries for debugging
- [ ] Consider data sync delays (eventual consistency)

---

## 🎯 NEXT DOCUMENTS

1. ✅ **01_MERN_STACK_OVERVIEW.md**
2. ✅ **02_DEVELOPMENT_ENVIRONMENT_SETUP.md**
3. ✅ **03_MODULES_ARCHITECTURE.md**
4. ✅ **04_MODULE_INTERCONNECTIONS.md** (You are here)
5. **05_DATABASE_DEVELOPER_GUIDE.md**
6. **06_API_DEVELOPMENT_GUIDE.md**
7. **07_FRONTEND_DEVELOPMENT_GUIDE.md**
8. **08_CODE_STANDARDS_CONVENTIONS.md**
9. **09_AUTHENTICATION_SECURITY_GUIDE.md**
10. **10_PAYMENT_PROCESSING_GUIDE.md**
11. **11_TESTING_DEVELOPER_GUIDE.md**
12. **12_DEBUGGING_TROUBLESHOOTING.md**

---

**Previous**: [03_MODULES_ARCHITECTURE.md](03_MODULES_ARCHITECTURE.md)  
**Next Document**: [05_DATABASE_DEVELOPER_GUIDE.md](05_DATABASE_DEVELOPER_GUIDE.md)

