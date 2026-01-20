# DATABASE DEVELOPER GUIDE

**For**: Backend Developers, Database Designers  
**Version**: 1.0  
**Date**: January 20, 2026  

---

## 📊 OVERVIEW

This guide explains how to work with MongoDB in NOVAA. You'll learn:

- MongoDB schema design
- Writing efficient queries
- Creating proper indexes
- Handling relationships between collections
- Multi-tenancy data isolation
- Performance optimization

---

## 🏗️ MONGODB FUNDAMENTALS

### What is MongoDB?

MongoDB is a **NoSQL document database**:

- **Document-based**: Stores JSON-like documents (BSON)
- **Flexible schema**: Columns don't need to be predefined
- **Scalable**: Handles millions of documents easily
- **Query-rich**: Powerful filtering, sorting, aggregation

### Terminology Mapping

```
SQL         →   MongoDB
Table       →   Collection
Row         →   Document
Column      →   Field
Index       →   Index
Join        →   $lookup (aggregation)
```

---

## 🗄️ NOVAA DATABASE COLLECTIONS

NOVAA uses 7 main collections. Here's each one:

---

## 📋 COLLECTION 1: `users`

**Purpose**: Stores all user accounts (students, staff, admins)

### Schema

```javascript
// users collection
{
  _id: ObjectId("507f1f77bcf86cd799439011"),
  collegeId: ObjectId("507f1f77bcf86cd799439012"),  // CRITICAL
  email: "student@stxavier.edu",
  password: "$2b$10$...",  // bcrypt hashed
  role: "STUDENT",        // ADMIN | STAFF | STUDENT
  name: "Aditya Sharma",
  phone: "9876543210",
  isActive: true,
  lastLogin: ISODate("2026-01-20T10:30:00Z"),
  createdAt: ISODate("2026-01-15T14:20:00Z"),
  updatedAt: ISODate("2026-01-20T10:30:00Z")
}
```

### Indexes

```javascript
// Most important indexes
db.users.createIndex({ collegeId: 1, email: 1 }, { unique: true });
// This ensures emails are unique PER college (not globally)

db.users.createIndex({ collegeId: 1, role: 1 });
// For queries like: Find all ADMINS in college

db.users.createIndex({ lastLogin: -1 });
// For: Get recently active users
```

### Common Queries

```javascript
// Find user by email (for login)
db.users.findOne({
  collegeId: ObjectId("507f1f77bcf86cd799439012"),
  email: "student@stxavier.edu"
});

// Find all admin users in college
db.users.find({
  collegeId: ObjectId("507f1f77bcf86cd799439012"),
  role: "ADMIN"
});

// Count total students
db.users.countDocuments({
  collegeId: ObjectId("507f1f77bcf86cd799439012"),
  role: "STUDENT"
});

// Update last login
db.users.updateOne(
  { _id: ObjectId("507f1f77bcf86cd799439011") },
  { $set: { lastLogin: new Date() } }
);
```

### In Mongoose (Node.js)

```javascript
const userSchema = new mongoose.Schema({
  collegeId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'College',
    required: true
  },
  email: {
    type: String,
    required: true,
    lowercase: true
  },
  password: {
    type: String,
    required: true,
    minlength: 6
  },
  role: {
    type: String,
    enum: ['ADMIN', 'STAFF', 'STUDENT'],
    required: true
  },
  name: String,
  phone: String,
  isActive: {
    type: Boolean,
    default: true
  },
  lastLogin: Date
}, { timestamps: true });

// Create unique index per college
userSchema.index({ collegeId: 1, email: 1 }, { unique: true });
userSchema.index({ collegeId: 1, role: 1 });
userSchema.index({ lastLogin: -1 });

module.exports = mongoose.model('User', userSchema);
```

---

## 📋 COLLECTION 2: `colleges`

**Purpose**: Stores information about each college (tenant)

### Schema

```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439012"),
  code: "ST_XAVIER_MUMBAI",              // Unique identifier
  name: "St. Xavier's College",
  state: "Maharashtra",
  city: "Mumbai",
  principalName: "Dr. Principal",
  principalEmail: "principal@stxavier.edu",
  principalPhone: "9876543210",
  gstNumber: "27AAACR5055K1Z0",           // For GST calculations
  gstin: "27AAACR5055K1Z0",
  academicYear: "2025-2026",
  isActive: true,
  subscription: {
    plan: "ENTERPRISE",                   // FREE | STARTER | ENTERPRISE
    maxStudents: 5000,
    expiryDate: ISODate("2027-01-15")
  },
  settings: {
    currency: "INR",
    timezone: "Asia/Kolkata",
    language: "en",
    admissionDeadline: ISODate("2026-06-30"),
    startDate: ISODate("2026-07-01")
  },
  createdAt: ISODate("2026-01-01T08:00:00Z"),
  updatedAt: ISODate("2026-01-20T10:00:00Z")
}
```

### Indexes

```javascript
// Code is unique globally
db.colleges.createIndex({ code: 1 }, { unique: true });

// For active college lookup
db.colleges.createIndex({ isActive: 1 });

// For filtering by state
db.colleges.createIndex({ state: 1, isActive: 1 });
```

### Common Queries

```javascript
// Get college by code (used everywhere)
db.colleges.findOne({ code: "ST_XAVIER_MUMBAI" });

// Get all active colleges
db.colleges.find({ isActive: true });

// Get colleges in Maharashtra
db.colleges.find({ state: "Maharashtra", isActive: true });

// Check subscription status
db.colleges.findOne({
  code: "ST_XAVIER_MUMBAI",
  "subscription.expiryDate": { $gt: new Date() }
});
```

### In Mongoose

```javascript
const collegeSchema = new mongoose.Schema({
  code: {
    type: String,
    required: true,
    unique: true,
    uppercase: true
  },
  name: {
    type: String,
    required: true
  },
  state: String,
  city: String,
  principalName: String,
  principalEmail: String,
  gstNumber: String,
  isActive: {
    type: Boolean,
    default: true
  },
  subscription: {
    plan: {
      type: String,
      enum: ['FREE', 'STARTER', 'ENTERPRISE'],
      default: 'FREE'
    },
    maxStudents: Number,
    expiryDate: Date
  },
  settings: {
    currency: String,
    timezone: String,
    admissionDeadline: Date,
    startDate: Date
  }
}, { timestamps: true });

collegeSchema.index({ code: 1 }, { unique: true });
collegeSchema.index({ isActive: 1 });

module.exports = mongoose.model('College', collegeSchema);
```

---

## 📋 COLLECTION 3: `admissions`

**Purpose**: Stores student application data

### Schema

```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439013"),
  collegeId: ObjectId("507f1f77bcf86cd799439012"),  // CRITICAL
  studentId: ObjectId("507f1f77bcf86cd799439011"),
  courseId: "BSC_CS",
  branchId: "BRANCH_A",
  applicationNumber: "APP-2026-0001",              // Unique per college
  status: "APPROVED",     // DRAFT | SUBMITTED | VERIFIED | APPROVED | REJECTED
  
  // Personal Information
  personalInfo: {
    firstName: "Aditya",
    lastName: "Sharma",
    email: "aditya@example.com",
    phone: "9876543210",
    dateOfBirth: ISODate("2006-05-15"),
    gender: "M",
    aadhar: "XXXX-XXXX-1234"
  },
  
  // Address Information
  address: {
    street: "123 Main Road",
    city: "Mumbai",
    state: "Maharashtra",
    pincode: "400001",
    country: "India"
  },
  
  // Academic Information
  academics: {
    previousPercentage: 85.5,
    previousBoard: "ISC",
    casteCategory: "GENERAL"
  },
  
  // Documents
  documents: [
    {
      _id: ObjectId(),
      category: "AADHAAR",           // AADHAAR | MARKSHEET | CASTE
      url: "s3://novaa/admissions/...",
      uploadedAt: ISODate("2026-01-15T10:00:00Z"),
      verificationStatus: "APPROVED", // PENDING | APPROVED | REJECTED
      verifiedBy: ObjectId(),
      rejectionReason: null
    }
  ],
  
  // Verification
  verification: {
    verifiedAt: ISODate("2026-01-16T14:00:00Z"),
    verifiedBy: ObjectId("507f1f77bcf86cd799439020"),
    notes: "All documents verified"
  },
  
  // Payment
  paymentStatus: "COMPLETE",   // PENDING | COMPLETE | FAILED
  paymentId: ObjectId("507f1f77bcf86cd799439030"),
  
  createdAt: ISODate("2026-01-15T09:30:00Z"),
  updatedAt: ISODate("2026-01-20T10:30:00Z")
}
```

### Indexes

```javascript
// Find applications by college
db.admissions.createIndex({ collegeId: 1, status: 1 });

// Find student's applications
db.admissions.createIndex({ collegeId: 1, studentId: 1 });

// Find by application number (unique per college)
db.admissions.createIndex({ collegeId: 1, applicationNumber: 1 }, { unique: true });

// Find pending verifications
db.admissions.createIndex({ collegeId: 1, status: 1, createdAt: -1 });
```

### Common Queries

```javascript
// Find pending applications
db.admissions.find({
  collegeId: ObjectId("..."),
  status: "SUBMITTED"
}).sort({ createdAt: 1 });

// Get student's application
db.admissions.findOne({
  collegeId: ObjectId("..."),
  studentId: ObjectId("...")
});

// Count applications by status
db.admissions.aggregate([
  {
    $match: { collegeId: ObjectId("...") }
  },
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  }
]);

// Find applications needing payment
db.admissions.find({
  collegeId: ObjectId("..."),
  status: "VERIFIED",
  paymentStatus: "PENDING"
});
```

### In Mongoose

```javascript
const admissionSchema = new mongoose.Schema({
  collegeId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'College',
    required: true
  },
  studentId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User',
    required: true
  },
  courseId: String,
  status: {
    type: String,
    enum: ['DRAFT', 'SUBMITTED', 'VERIFIED', 'APPROVED', 'REJECTED'],
    default: 'DRAFT'
  },
  personalInfo: {
    firstName: String,
    lastName: String,
    email: String,
    phone: String,
    dateOfBirth: Date,
    gender: String,
    aadhar: String
  },
  documents: [{
    category: String,
    url: String,
    uploadedAt: Date,
    verificationStatus: String,
    verifiedBy: mongoose.Schema.Types.ObjectId,
    rejectionReason: String
  }],
  paymentStatus: String,
  paymentId: mongoose.Schema.Types.ObjectId
}, { timestamps: true });

admissionSchema.index({ collegeId: 1, status: 1 });
admissionSchema.index({ collegeId: 1, studentId: 1 });
admissionSchema.index({ collegeId: 1, applicationNumber: 1 }, { unique: true });

module.exports = mongoose.model('Admission', admissionSchema);
```

---

## 📋 COLLECTION 4: `transactions`

**Purpose**: Stores payment records (Razorpay integration)

### Schema

```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439030"),
  collegeId: ObjectId("507f1f77bcf86cd799439012"),  // CRITICAL
  admissionId: ObjectId("507f1f77bcf86cd799439013"),
  studentId: ObjectId("507f1f77bcf86cd799439011"),
  
  // Amount details
  amount: 64160,                    // Total in paise (₹641.60 = 64160 paise)
  currency: "INR",
  
  // GST breakdown
  gst: {
    tuitionFee: 50000,              // Exempt
    taxableAmount: 12000,           // Lab + Sports fee
    gstRate: 18,
    gstAmount: 2160,
    totalGST: 2160
  },
  
  // Razorpay details
  razorpayOrderId: "order_xyz123",
  razorpayPaymentId: "pay_abc456",
  razorpaySignature: "sig_def789",
  
  // Idempotency (prevents duplicate payments)
  idempotencyKey: "unique_key_2026_01_15_001",
  
  // Status
  status: "SUCCESS",                // PENDING | SUCCESS | FAILED | CANCELLED
  failureReason: null,
  failureCode: null,
  
  // Retry information
  retryCount: 0,
  nextRetryAt: null,
  lastRetryAt: null,
  
  // Receipt
  receiptUrl: "s3://novaa/receipts/REC-2026-001.pdf",
  receiptNumber: "REC-2026-001",
  
  // Metadata
  paymentMethod: "UPI",             // UPI | CARD | NETBANKING
  bankName: null,
  cardLast4: null,
  
  createdAt: ISODate("2026-01-20T14:30:00Z"),
  updatedAt: ISODate("2026-01-20T14:35:00Z"),
  completedAt: ISODate("2026-01-20T14:35:00Z")
}
```

### Indexes

```javascript
// Find by college and status
db.transactions.createIndex({ collegeId: 1, status: 1 });

// Find failed payments for retry
db.transactions.createIndex({ 
  status: 1, 
  nextRetryAt: 1 
});

// Prevent duplicate payments
db.transactions.createIndex({ idempotencyKey: 1 }, { unique: true });

// Find by Razorpay IDs
db.transactions.createIndex({ razorpayPaymentId: 1 }, { unique: true });

// For revenue reports
db.transactions.createIndex({ collegeId: 1, status: 1, createdAt: -1 });
```

### Common Queries

```javascript
// Get payment status for admission
db.transactions.findOne({
  collegeId: ObjectId("..."),
  admissionId: ObjectId("..."),
  status: "SUCCESS"
});

// Find failed payments (for retry)
db.transactions.find({
  collegeId: ObjectId("..."),
  status: "FAILED",
  nextRetryAt: { $lte: new Date() },
  retryCount: { $lt: 3 }
});

// Calculate total revenue
db.transactions.aggregate([
  {
    $match: {
      collegeId: ObjectId("..."),
      status: "SUCCESS"
    }
  },
  {
    $group: {
      _id: null,
      totalAmount: { $sum: "$amount" },
      totalGST: { $sum: "$gst.gstAmount" },
      count: { $sum: 1 }
    }
  }
]);

// Check for duplicate payment (idempotency)
db.transactions.findOne({ idempotencyKey: "unique_key_2026_01_15_001" });
```

### In Mongoose

```javascript
const transactionSchema = new mongoose.Schema({
  collegeId: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'College',
    required: true
  },
  admissionId: mongoose.Schema.Types.ObjectId,
  studentId: mongoose.Schema.Types.ObjectId,
  amount: Number,
  currency: String,
  gst: {
    tuitionFee: Number,
    taxableAmount: Number,
    gstRate: Number,
    gstAmount: Number
  },
  razorpayOrderId: String,
  razorpayPaymentId: String,
  razorpaySignature: String,
  idempotencyKey: {
    type: String,
    unique: true
  },
  status: {
    type: String,
    enum: ['PENDING', 'SUCCESS', 'FAILED', 'CANCELLED'],
    default: 'PENDING'
  },
  failureReason: String,
  retryCount: {
    type: Number,
    default: 0
  },
  receiptUrl: String,
  receiptNumber: String
}, { timestamps: true });

transactionSchema.index({ collegeId: 1, status: 1 });
transactionSchema.index({ status: 1, nextRetryAt: 1 });
transactionSchema.index({ idempotencyKey: 1 }, { unique: true });
transactionSchema.index({ razorpayPaymentId: 1 }, { unique: true });

module.exports = mongoose.model('Transaction', transactionSchema);
```

---

## 📋 COLLECTION 5: `attendance`

**Purpose**: Stores attendance records

### Schema

```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439040"),
  collegeId: ObjectId("507f1f77bcf86cd799439012"),  // CRITICAL
  studentId: ObjectId("507f1f77bcf86cd799439011"),
  courseId: "BSC_CS",
  sectionId: "SECTION_A",
  
  // Date and time
  date: ISODate("2026-01-20"),      // Date of attendance
  scannedAt: ISODate("2026-01-20T09:02:30Z"),
  
  // Status
  status: "PRESENT",                // PRESENT | ABSENT | LEAVE | SICK_LEAVE
  
  // Who marked
  markedBy: ObjectId("507f1f77bcf86cd799439020"),  // Staff who marked
  
  // QR details
  qrCode: "STX_2026_12345",
  
  createdAt: ISODate("2026-01-20T09:02:30Z")
}
```

### Indexes

```javascript
// Prevent duplicate scan (same student, same date, same course)
db.attendance.createIndex(
  { collegeId: 1, studentId: 1, courseId: 1, date: 1 },
  { unique: true }
);

// Find attendance for student
db.attendance.createIndex({ collegeId: 1, studentId: 1, date: 1 });

// Find all attendance for a date
db.attendance.createIndex({ collegeId: 1, courseId: 1, date: 1 });

// Calculate attendance percentage
db.attendance.createIndex({ collegeId: 1, studentId: 1, status: 1 });
```

### Common Queries

```javascript
// Mark attendance (prevent duplicates)
const existingRecord = db.attendance.findOne({
  collegeId: ObjectId("..."),
  studentId: ObjectId("..."),
  courseId: "BSC_CS",
  date: ISODate("2026-01-20")
});

if (!existingRecord) {
  db.attendance.insertOne({
    collegeId: ObjectId("..."),
    studentId: ObjectId("..."),
    courseId: "BSC_CS",
    date: ISODate("2026-01-20"),
    status: "PRESENT"
  });
}

// Calculate attendance percentage
db.attendance.aggregate([
  {
    $match: {
      collegeId: ObjectId("..."),
      studentId: ObjectId("..."),
      courseId: "BSC_CS"
    }
  },
  {
    $group: {
      _id: "$status",
      count: { $sum: 1 }
    }
  }
]);

// Find at-risk students (< 75% attendance)
db.attendance.aggregate([
  {
    $match: {
      collegeId: ObjectId("..."),
      courseId: "BSC_CS",
      date: {
        $gte: ISODate("2025-12-01"),
        $lte: ISODate("2026-01-20")
      }
    }
  },
  {
    $group: {
      _id: "$studentId",
      total: { $sum: 1 },
      present: {
        $sum: { $cond: [{ $eq: ["$status", "PRESENT"] }, 1, 0] }
      }
    }
  },
  {
    $addFields: {
      percentage: { $multiply: [{ $divide: ["$present", "$total"] }, 100] }
    }
  },
  {
    $match: {
      percentage: { $lt: 75 }
    }
  }
]);
```

---

## 📋 COLLECTION 6: `feesStructures`

**Purpose**: Stores fee configuration per course

### Schema

```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439050"),
  collegeId: ObjectId("507f1f77bcf86cd799439012"),  // CRITICAL
  courseId: "BSC_CS",
  
  // Fees
  fees: {
    tuitionFee: 50000,              // GST EXEMPT
    labFee: 10000,                  // 18% GST
    sportsFee: 2000,                // 18% GST
    libraryFee: 1000,               // 18% GST
    developmentFee: 5000            // 18% GST
  },
  
  // Total calculations
  totalFees: 68000,
  taxableAmount: 18000,             // Non-exempt fees
  gstAmount: 3240,                  // 18% of taxable
  totalWithGST: 71240,
  
  // Validity
  effectiveDate: ISODate("2026-01-01"),
  expiryDate: ISODate("2026-12-31"),
  
  // Status
  isActive: true,
  
  createdAt: ISODate("2025-12-15T10:00:00Z"),
  updatedAt: ISODate("2025-12-15T10:00:00Z")
}
```

### Indexes

```javascript
// Get active fee structure for course
db.feesStructures.createIndex({ collegeId: 1, courseId: 1, isActive: 1 });

// Get fee structure for date range
db.feesStructures.createIndex({ 
  collegeId: 1, 
  courseId: 1, 
  effectiveDate: 1, 
  expiryDate: 1 
});
```

### Common Queries

```javascript
// Get current fee structure
db.feesStructures.findOne({
  collegeId: ObjectId("..."),
  courseId: "BSC_CS",
  effectiveDate: { $lte: new Date() },
  expiryDate: { $gte: new Date() },
  isActive: true
});

// Get all fee structures for college
db.feesStructures.find({
  collegeId: ObjectId("..."),
  isActive: true
});
```

---

## 📋 COLLECTION 7: `reports`

**Purpose**: Stores pre-calculated report snapshots

### Schema

```javascript
{
  _id: ObjectId("507f1f77bcf86cd799439060"),
  collegeId: ObjectId("507f1f77bcf86cd799439012"),  // CRITICAL
  reportType: "ADMISSION_FUNNEL",
  
  // Data
  data: {
    totalApplications: 156,
    submitted: 147,
    verified: 135,
    approved: 93,
    conversionRate: 59.6
  },
  
  // Date range
  generatedFor: "2026-01-20",
  generatedAt: ISODate("2026-01-20T23:59:59Z")
}
```

### Common Queries

```javascript
// Get latest report
db.reports.findOne({
  collegeId: ObjectId("..."),
  reportType: "ADMISSION_FUNNEL"
}, { sort: { generatedAt: -1 } });
```

---

## 🔐 CRITICAL: MULTI-TENANCY & DATA ISOLATION

**Every single query must include collegeId filter. This is non-negotiable.**

### The Multi-Tenancy Rule

```javascript
// ✅ ALWAYS DO THIS
db.users.find({
  collegeId: ObjectId("507f1f77bcf86cd799439012"),
  role: "STUDENT"
});

// ❌ NEVER DO THIS
db.users.find({ role: "STUDENT" });
// This returns students from ALL colleges!

// ❌ NEVER DO THIS
db.admissions.find({ status: "APPROVED" });
// This returns admissions from ALL colleges!
```

### Implementation in Code

```javascript
// middleware/collegeContext.js (applied to EVERY route)
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
  
  req.college = college;  // Attach to request
  next();
};

// All routes use this
app.use(enforceCollegeContext);

// Then in your service:
const getAdmissions = async (collegeId, status) => {
  // collegeId is guaranteed to exist
  return await Admission.find({
    collegeId: collegeId,    // CRITICAL
    status
  });
};
```

---

## 🔍 QUERY PATTERNS

### Pattern 1: Simple Find

```javascript
// Find a specific user
const user = await User.findOne({
  collegeId: req.college._id,
  email: "student@college.edu"
});
```

### Pattern 2: Find with Filter

```javascript
// Find all pending applications
const pending = await Admission.find({
  collegeId: req.college._id,
  status: "SUBMITTED"
}).sort({ createdAt: -1 });
```

### Pattern 3: Count

```javascript
// Count approved admissions
const count = await Admission.countDocuments({
  collegeId: req.college._id,
  status: "APPROVED"
});
```

### Pattern 4: Aggregation (Complex Queries)

```javascript
// Calculate revenue with GST breakdown
const revenue = await Transaction.aggregate([
  {
    $match: {
      collegeId: req.college._id,
      status: "SUCCESS"
    }
  },
  {
    $group: {
      _id: null,
      totalAmount: { $sum: "$amount" },
      totalGST: { $sum: "$gst.gstAmount" },
      totalWithoutGST: { $sum: { $subtract: ["$amount", "$gst.gstAmount"] } },
      count: { $sum: 1 }
    }
  }
]);
```

### Pattern 5: Join Collections ($lookup)

```javascript
// Get admissions with payment details
const admissionsWithPayments = await Admission.aggregate([
  {
    $match: { collegeId: req.college._id }
  },
  {
    $lookup: {
      from: 'transactions',
      let: { admissionId: '$_id' },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ['$admissionId', '$$admissionId'] },
                { $eq: ['$collegeId', req.college._id] }  // MULTI-TENANCY!
              ]
            }
          }
        }
      ],
      as: 'payments'
    }
  }
]);
```

### Pattern 6: Update with Conditions

```javascript
// Update admission status only if current status is VERIFIED
const result = await Admission.updateOne(
  {
    _id: admissionId,
    collegeId: req.college._id,
    status: "VERIFIED"
  },
  {
    $set: {
      status: "APPROVED",
      updatedAt: new Date()
    }
  }
);

if (result.modifiedCount === 0) {
  // Record didn't exist or wasn't in VERIFIED status
  throw new Error('Cannot approve application in current status');
}
```

---

## ⚡ PERFORMANCE OPTIMIZATION

### 1. Create Appropriate Indexes

**Good**:
```javascript
// This index supports finding users by college and email
db.users.createIndex({ collegeId: 1, email: 1 });
```

**Bad**:
```javascript
// Creating index on every field slows down writes
db.users.createIndex({ createdAt: 1 });
db.users.createIndex({ name: 1 });
db.users.createIndex({ phone: 1 });
// Too many indexes!
```

### 2. Use Projection to Limit Fields

**Before**:
```javascript
// Returns ALL fields
const users = await User.find({
  collegeId: req.college._id,
  role: "STUDENT"
});
// Each user document might be large (including password hash, etc.)
```

**After**:
```javascript
// Returns only needed fields
const users = await User.find(
  { collegeId: req.college._id, role: "STUDENT" },
  { name: 1, email: 1, phone: 1 }  // Only these fields
);
// Faster transmission + less memory
```

### 3. Use Pagination for Large Results

**Bad**:
```javascript
// Loading ALL admissions at once
const admissions = await Admission.find({
  collegeId: req.college._id
});
// If 100,000 admissions exist, this is slow!
```

**Good**:
```javascript
// Load in pages
const page = req.query.page || 1;
const pageSize = 20;
const skip = (page - 1) * pageSize;

const admissions = await Admission.find({
  collegeId: req.college._id
})
  .skip(skip)
  .limit(pageSize)
  .sort({ createdAt: -1 });

const total = await Admission.countDocuments({
  collegeId: req.college._id
});

const totalPages = Math.ceil(total / pageSize);
```

### 4. Batch Operations

**Bad**:
```javascript
// Calling database 1000 times
for (let i = 0; i < 1000; i++) {
  await User.updateOne(
    { _id: userIds[i] },
    { $set: { isActive: false } }
  );
}
```

**Good**:
```javascript
// Single database call
await User.updateMany(
  { _id: { $in: userIds } },
  { $set: { isActive: false } }
);
```

---

## 🧪 TESTING YOUR QUERIES

### Using MongoDB Shell (mongosh)

```bash
# Connect to MongoDB
mongosh

# Connect to database
use novaa_dev

# Test query
db.users.find({ collegeId: ObjectId("...") })

# Count results
db.users.countDocuments({ collegeId: ObjectId("...") })

# Explain query performance
db.users.find({ collegeId: ObjectId("...") }).explain("executionStats")
```

### Using MongoDB Compass (GUI)

Download: https://www.mongodb.com/products/compass

1. Paste connection string
2. Browse collections
3. Write queries visually
4. See results instantly

---

## ⚠️ COMMON MISTAKES

### Mistake 1: Forgetting collegeId

```javascript
// ❌ WRONG
const users = await User.find({ role: "ADMIN" });
// Returns admins from ALL colleges!

// ✅ CORRECT
const users = await User.find({
  collegeId: req.college._id,
  role: "ADMIN"
});
```

### Mistake 2: Not Using Indexes

```javascript
// ❌ Without index (slow)
db.admissions.find({ studentId: ObjectId("...") });
// Scans entire collection

// ✅ With index (fast)
db.admissions.createIndex({ collegeId: 1, studentId: 1 });
db.admissions.find({
  collegeId: ObjectId("..."),
  studentId: ObjectId("...")
});
```

### Mistake 3: Modifying Documents Incorrectly

```javascript
// ❌ WRONG - overwrites entire document
await User.updateOne(
  { _id: userId },
  { name: "New Name", email: "new@email.com" }
  // Other fields deleted!
);

// ✅ CORRECT - only updates specified fields
await User.updateOne(
  { _id: userId },
  { $set: { name: "New Name", email: "new@email.com" } }
);
```

### Mistake 4: Race Conditions

```javascript
// ❌ Vulnerable to race condition
const user = await User.findOne({ _id: userId });
if (user.balance > 100) {
  // What if another process modified balance meanwhile?
  await User.updateOne(
    { _id: userId },
    { $set: { balance: user.balance - 100 } }
  );
}

// ✅ Atomic operation
const result = await User.updateOne(
  { _id: userId, balance: { $gte: 100 } },
  { $inc: { balance: -100 } }
);
if (result.modifiedCount === 0) {
  throw new Error('Insufficient balance');
}
```

---

## 📊 NEXT DOCUMENTS

1. ✅ **01_MERN_STACK_OVERVIEW.md**
2. ✅ **02_DEVELOPMENT_ENVIRONMENT_SETUP.md**
3. ✅ **03_MODULES_ARCHITECTURE.md**
4. ✅ **04_MODULE_INTERCONNECTIONS.md**
5. ✅ **05_DATABASE_DEVELOPER_GUIDE.md** (You are here)
6. **06_API_DEVELOPMENT_GUIDE.md**
7. **07_FRONTEND_DEVELOPMENT_GUIDE.md**
8. **08_CODE_STANDARDS_CONVENTIONS.md**
9. **09_AUTHENTICATION_SECURITY_GUIDE.md**
10. **10_PAYMENT_PROCESSING_GUIDE.md**
11. **11_TESTING_DEVELOPER_GUIDE.md**
12. **12_DEBUGGING_TROUBLESHOOTING.md**

---

**Previous**: [04_MODULE_INTERCONNECTIONS.md](04_MODULE_INTERCONNECTIONS.md)  
**Next Document**: [06_API_DEVELOPMENT_GUIDE.md](06_API_DEVELOPMENT_GUIDE.md)

