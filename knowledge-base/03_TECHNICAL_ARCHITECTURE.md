# TECHNICAL ARCHITECTURE DOCUMENT - NOVAA MVP

**Document Version**: 1.0  
**Date**: January 20, 2026  
**Status**: DRAFT - Ready for Technical Review  

---

## 1. ARCHITECTURE OVERVIEW

### 1.1 System Boundaries

```
┌─────────────────────────────────────────────────────────────┐
│                    NOVAA MVP SYSTEM (2026)                  │
│                                                             │
│  ┌──────────────┐    ┌──────────────────┐ ┌──────────────┐ │
│  │  Frontend    │    │  Backend API     │ │  Database    │ │
│  │  (React)     │◄──►│  (Node.js)       │◄┤  (MongoDB)   │ │
│  │  Vercel      │    │  Render.com      │ │  Atlas Free  │ │
│  └──────────────┘    └──────────────────┘ └──────────────┘ │
│         ▲                     ▲                              │
│         │                     │                              │
│         └─────────────────────┘                              │
│                     │                                        │
│         ┌───────────┴────────────┐                          │
│         ▼                        ▼                           │
│    ┌──────────────┐      ┌──────────────┐                  │
│    │  Razorpay    │      │  AWS S3      │                  │
│    │  (Payments)  │      │  (Documents) │                  │
│    └──────────────┘      └──────────────┘                  │
│                                                             │
│         COLLEGES (5 pilot)                                  │
│         ┌────────────────────────────────┐                 │
│         │ • St. Xavier's Mumbai           │                │
│         │ • Modern College Pune           │                │
│         │ • [3 more Maharashtra colleges] │                │
│         └────────────────────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Technology Stack

| Layer | Technology | Reason for Choice | Licenses |
|-------|-----------|-------------------|----------|
| **Frontend** | React 18 + TypeScript | Type-safe, component-based | MIT |
| **Backend** | Node.js 18 + Express | Async I/O for real-time features | MIT |
| **Database** | MongoDB 5.0+ | Flexible schema for multi-tenancy | Server-side Public License |
| **Payment** | Razorpay API | Market leader in India | Proprietary API |
| **Storage** | AWS S3 | Secure document storage | Pay-per-use |
| **Hosting** | Render.com (Backend) | Free tier for MVP, auto-scaling | Freemium |
| **Hosting** | Vercel (Frontend) | Optimized for React, CI/CD | Freemium |
| **Auth** | JWT + bcrypt | Industry-standard, no licensing | MIT |

### 1.3 MVP Architecture Principles

| Principle | Implementation | Why |
|-----------|-----------------|-----|
| **Single Tenant Per Context** | Every request includes collegeId | Prevents data leakage |
| **Fail-Safe Defaults** | Queries without collegeId rejected | Defense in depth |
| **Immutable Audit Trails** | All admin actions logged | Regulatory compliance |
| **Graceful Degradation** | System works if S3 is slow | Rural colleges have poor connectivity |
| **No State Assumptions** | Only Maharashtra rules in MVP | Avoids complex rule engine |

---

## 2. DATA MODEL & ISOLATION

### 2.1 Multi-Tenancy Architecture

**Choice: Shared Database, Tenant-Specific Schema**

```
MongoDB Collections Structure:
┌──────────────────────────────────────┐
│ colleges                              │
├──────────────────────────────────────┤
│ { _id, code, name, state, isActive } │
└──────────────────────────────────────┘
         │
         ├─ users (has collegeId)
         ├─ students (has collegeId)
         ├─ admissions (has collegeId)
         ├─ feeStructures (has collegeId)
         ├─ transactions (has collegeId)
         ├─ attendance (has collegeId)
         └─ auditLogs (has collegeId)

CRITICAL: Every collection EXCEPT colleges has collegeId field
```

**Why This Approach?**
- ✅ Simpler deployment (1 database, 1 app)
- ✅ Easier backup/restore (entire database is atomic)
- ✅ Row-level security on all queries (collegeId in WHERE clause)
- ❌ Harder to scale (database connection limits at ~1000 colleges)
- ❌ Noisy neighbor (one college's heavy query slows others)

**Scaling Strategy (Beyond MVP)**:
- V2.0: Database sharding by collegeCode
- V3.0: Kubernetes with per-college service replicas

---

### 2.2 Core Collections

#### COLLEGES
```javascript
{
  _id: ObjectId,
  code: String (unique), // "STX_MUMBAI_001"
  name: String,
  state: String (enum: "MAHARASHTRA"), // MVP only
  principalEmail: String,
  principalPhone: String,
  gstNumber: String,
  isActive: Boolean (default: true),
  createdAt: Date,
  updatedAt: Date
}

// Index: code (unique)
// Index: (state, isActive) for filtering
```

#### USERS
```javascript
{
  _id: ObjectId,
  collegeId: ObjectId (ref: colleges), // MANDATORY
  email: String (lowercase),
  password: String (bcrypt hashed),
  role: String (enum: ["ADMIN", "STAFF", "STUDENT"]),
  isActive: Boolean (default: true),
  lastLogin: Date,
  createdAt: Date
}

// CRITICAL INDEX: (collegeId, email) unique
// This prevents same email across colleges
// Queries ALWAYS include: { collegeId, email }
```

#### STUDENTS
```javascript
{
  _id: ObjectId,
  collegeId: ObjectId (ref: colleges),
  userId: ObjectId (ref: users),
  name: String,
  phone: String,
  aadhaarNumber: String (masked in logs),
  casteCategory: String (enum: ["GEN", "SC", "ST", "OBC", "EWS"]),
  class12Score: Number,
  class12Percentage: Number,
  admissionYear: Number,
  createdAt: Date
}

// Index: (collegeId, userId)
// Index: (collegeId, admissionYear) for batch operations
```

#### ADMISSIONS
```javascript
{
  _id: ObjectId,
  collegeId: ObjectId (ref: colleges),
  studentId: ObjectId (ref: students),
  courseId: String,
  status: String (enum: ["DRAFT", "SUBMITTED", "PENDING_VERIFICATION", 
                          "VERIFIED", "REJECTED", "APPROVED"]),
  documents: [
    {
      url: String, // S3 path
      category: String (enum: ["AADHAAR", "MARKSHEET", "CASTE_CERT"]),
      verificationStatus: String (enum: ["PENDING", "APPROVED", "REJECTED"]),
      rejectionReason: String,
      uploadedAt: Date
    }
  ],
  notes: String,
  createdAt: Date,
  updatedAt: Date
}

// Index: (collegeId, status)
// Index: (collegeId, studentId) for filtering by student
```

#### TRANSACTIONS (Payments)
```javascript
{
  _id: ObjectId,
  collegeId: ObjectId (ref: colleges),
  studentId: ObjectId (ref: students),
  admissionId: ObjectId (ref: admissions),
  amount: Number,
  currency: String (default: "INR"),
  razorpayOrderId: String,
  razorpayPaymentId: String,
  status: String (enum: ["PENDING", "SUCCESS", "FAILED"]),
  failureReason: String,
  receiptUrl: String, // S3 path to PDF
  idempotencyKey: String (unique per transaction), // Prevents duplicates
  createdAt: Date,
  updatedAt: Date
}

// CRITICAL INDEX: (collegeId, razorpayOrderId)
// CRITICAL INDEX: idempotencyKey (unique globally)
// INDEX: (collegeId, studentId) for payment history
```

#### ATTENDANCE
```javascript
{
  _id: ObjectId,
  collegeId: ObjectId (ref: colleges),
  studentId: ObjectId (ref: students),
  courseId: String,
  date: Date,
  status: String (enum: ["PRESENT", "ABSENT", "LEAVE"]),
  qrScannedAt: Date,
  markedBy: ObjectId (ref: users), // Staff who marked
  createdAt: Date
}

// CRITICAL INDEX: (collegeId, studentId, date) unique
// INDEX: (collegeId, date) for daily reports
```

#### AUDIT_LOGS
```javascript
{
  _id: ObjectId,
  collegeId: ObjectId (ref: colleges),
  userId: ObjectId (ref: users),
  action: String (enum: ["DOCUMENT_VERIFIED", "ADMISSION_REJECTED", 
                          "PAYMENT_PROCESSED", "ATTENDANCE_MARKED"]),
  resourceType: String (enum: ["ADMISSION", "TRANSACTION", "ATTENDANCE"]),
  resourceId: ObjectId,
  oldValues: Object, // What changed
  newValues: Object,
  ipAddress: String,
  userAgent: String,
  createdAt: Date
}

// Index: (collegeId, userId, createdAt) for audit trails
```

---

## 3. API ARCHITECTURE

### 3.1 Request/Response Pattern

**EVERY API endpoint follows this pattern:**

```javascript
// Middleware: Enforce college context
app.use(requireCollegeContext); // Attaches req.college from headers

// API: Get student admissions
app.get("/api/admissions", authenticate, async (req, res) => {
  
  // MANDATORY: Use req.college.id in query
  const admissions = await Admission.find({
    collegeId: req.college.id, // AUTOMATIC FILTERING
    status: req.query.status || undefined
  });
  
  res.json({ data: admissions });
});

// ❌ FORBIDDEN pattern (will be caught in code review):
// app.get("/api/admissions", (req, res) => {
//   const admissions = await Admission.find({}); // NO COLLEGE FILTER!
// })
```

### 3.2 Error Handling

```javascript
// Standardized error responses
class APIError extends Error {
  constructor(message, statusCode, collegeContext) {
    this.message = message;
    this.statusCode = statusCode;
    this.collegeContext = collegeContext;
  }
}

// Example: Missing college context
if (!req.headers["x-college-code"]) {
  throw new APIError(
    "Missing college context",
    400,
    "MISSING_COLLEGE_CODE"
  );
}

// Response format:
{
  "status": "error",
  "message": "Document verification failed",
  "errorCode": "VERIFICATION_FAILED",
  "details": {
    "documentId": "65a3b2c1d4e5f6g7h8i9j0k1",
    "reason": "Image too blurry"
  }
}
```

### 3.3 Authentication Flow

```
1. College Admin logs in
   POST /api/auth/login
   {
     email: "admin@college.edu",
     password: "hashed_password"
   }

2. System returns JWT token (2-hour expiry)
   {
     token: "eyJhbGc...",
     user: { id, email, role },
     college: { id, code, name }
   }

3. All subsequent requests include token
   Authorization: Bearer eyJhbGc...
   X-College-Code: STX_MUMBAI_001

4. Middleware validates:
   - Token signature ✅
   - Token not expired ✅
   - College code matches token ✅
   - User role has permission ✅

5. Request proceeds with req.user + req.college attached
```

---

## 4. CRITICAL SECURITY SAFEGUARDS

### 4.1 Data Isolation Enforcement

**MIDDLEWARE: Every API protected**

```javascript
// middlewares/multiTenancy.js
const enforceMultiTenancy = async (req, res, next) => {
  const collegeCode = req.headers["x-college-code"];
  
  if (!collegeCode) {
    return res.status(400).json({ 
      error: "MISSING_COLLEGE_CONTEXT" 
    });
  }
  
  const college = await College.findOne({ 
    code: collegeCode,
    isActive: true 
  });
  
  if (!college) {
    return res.status(403).json({ 
      error: "INVALID_COLLEGE" 
    });
  }
  
  // Attach to request
  req.college = college;
  
  // MONKEY-PATCH: Force collegeId in all DB queries
  const originalFind = mongoose.Query.prototype.find;
  mongoose.Query.prototype.find = function(query = {}) {
    if (!query.collegeId && !query._id) {
      throw new Error(
        `SECURITY_VIOLATION: Query missing collegeId filter. ` +
        `Model: ${this.model.modelName}. ` +
        `Query: ${JSON.stringify(query)}`
      );
    }
    return originalFind.call(this, { ...query, collegeId: req.college._id });
  };
  
  next();
};

// Usage
app.use(enforceMultiTenancy);
```

### 4.2 Password Security

```javascript
// User registration
const bcrypt = require("bcrypt");

async function registerUser(email, password, collegeId) {
  // Validate password
  if (password.length < 8) {
    throw new Error("Password must be 8+ characters");
  }
  if (!/[A-Z]/.test(password)) {
    throw new Error("Password must include uppercase letter");
  }
  if (!/[0-9]/.test(password)) {
    throw new Error("Password must include number");
  }
  
  // Hash with salt rounds = 10 (standard)
  const hashedPassword = await bcrypt.hash(password, 10);
  
  // Store
  const user = new User({
    collegeId,
    email,
    password: hashedPassword
  });
  
  await user.save();
}

// Login
async function login(email, password) {
  const user = await User.findOne({ email, collegeId });
  const isValid = await bcrypt.compare(password, user.password);
  
  if (!isValid) {
    // Log failed attempt for rate limiting
    await LoginAttempt.create({ email, ip: req.ip });
    throw new Error("Invalid credentials");
  }
  
  // Generate JWT
  const token = jwt.sign(
    { userId: user._id, collegeId: user.collegeId },
    process.env.JWT_SECRET,
    { expiresIn: "2h" }
  );
  
  return token;
}
```

### 4.3 Rate Limiting

```javascript
const rateLimit = require("express-rate-limit");

// Prevent brute force attacks
const loginLimiter = rateLimit({
  windowMs: 60000, // 1 minute
  max: 5, // 5 attempts per minute
  message: "Too many login attempts. Try again in 1 minute.",
  keyGenerator: (req) => req.body.email
});

app.post("/api/auth/login", loginLimiter, (req, res) => {
  // Handle login
});
```

---

## 5. PAYMENT PROCESSING ARCHITECTURE

### 5.1 Payment Flow (Idempotency)

```
CRITICAL: Must handle duplicate requests (network timeouts)

Step 1: Client generates IDEMPOTENCY_KEY (UUIDv7)
        POST /api/fees/create-order
        {
          admissionId: "...",
          amount: 61800,
          idempotencyKey: "7e7d3ecc-abe9-4f6c-a906-ca0d7c3bc7d2"
        }

Step 2: Backend checks if idempotencyKey exists
        const existing = await Transaction.findOne({
          idempotencyKey: "7e7d3ecc-abe9-4f6c-a906-ca0d7c3bc7d2"
        });
        
        if (existing) return existing; // Prevent duplicate order

Step 3: Create transaction with PENDING status
        const transaction = await Transaction.create({
          collegeId, studentId, amount,
          idempotencyKey,
          status: "PENDING"
        });

Step 4: Create Razorpay order
        const razorpayOrder = await razorpay.orders.create({
          amount: 61800 * 100, // in paise
          currency: "INR",
          receipt: transaction._id.toString()
        });

Step 5: Store razorpayOrderId
        transaction.razorpayOrderId = razorpayOrder.id;
        await transaction.save();

Step 6: Return to client
        {
          orderId: razorpayOrder.id,
          transactionId: transaction._id
        }

Step 7: Client submits to Razorpay
        Razorpay handles payment

Step 8: Razorpay sends webhook to backend
        POST /webhooks/razorpay
        {
          event: "payment.authorized",
          payload: {
            order: { id: "order_XXXXX" },
            payment: { id: "pay_YYYYY" }
          },
          signature: "webhook_signature"
        }

Step 9: Backend verifies signature
        const isValid = verifyRazorpaySignature(
          req.body.payload,
          req.headers["x-razorpay-signature"],
          process.env.RAZORPAY_SECRET
        );

Step 10: Update transaction
         await Transaction.findOneAndUpdate(
           { razorpayOrderId: "order_XXXXX" },
           {
             status: "SUCCESS",
             razorpayPaymentId: "pay_YYYYY"
           }
         );

Step 11: Generate receipt PDF
         const receipt = generateReceiptPDF({
           studentName, amount, gstSplit, paymentId
         });
         
         const receiptUrl = await uploadToS3(
           receipt,
           `receipts/${transactionId}.pdf`
         );

Step 12: Send SMS + Email
         await sendSMS(student.phone, 
           "Payment successful. Receipt: " + receiptUrl);
         await sendEmail(student.email, receiptUrl);
```

### 5.2 Failed Payment Scenario

```
If Razorpay webhook fails/timeout:

1. Transaction remains PENDING
2. Cron job every 15 mins checks for old PENDING
3. For 2+ hour old PENDING transactions:
   - Call Razorpay API to check status
   - If payment exists but we missed webhook: Mark SUCCESS
   - If payment failed: Mark FAILED with reason
   - If payment still processing: Leave PENDING

4. Alert admin via dashboard:
   "Failed Payment: Raj Kumar, ₹61,800, pending 2+ hours"

5. Admin manually reviews:
   Option A: Mark paid (if gateway confirmed)
   Option B: Send retry link to student
   Option C: Issue refund credit
```

---

## 6. DEPLOYMENT & INFRASTRUCTURE

### 6.1 Development Environment Setup

```bash
# Backend setup
cd backend
npm install
cp .env.example .env
# Edit .env with:
#   MONGODB_URI=mongodb://localhost:27017/novaa_dev
#   RAZORPAY_KEY_ID=xxx
#   JWT_SECRET=dev_secret_123

npm run dev # Starts on http://localhost:5000

# Frontend setup
cd ../frontend
npm install
npm start # Starts on http://localhost:3000
```

### 6.2 Production Deployment

**Backend (Render.com)**:
```yaml
Environment: Node.js 18
Build: npm install && npm run build
Start: node dist/server.js
Environment Variables:
  - MONGODB_URI: (MongoDB Atlas connection)
  - RAZORPAY_KEY_ID: (Production key)
  - JWT_SECRET: (Secure 32-char random)
  - NODE_ENV: production
```

**Frontend (Vercel)**:
```yaml
Framework: Next.js (React)
Build: npm run build
Output: .next
Environment Variables:
  - REACT_APP_API_URL: https://api.novaa.in
  - RAZORPAY_KEY_ID: (Public key)
```

**Database (MongoDB Atlas)**:
```
Cluster: M0 free tier
Region: ap-south-1 (Mumbai)
Backup: Daily (7-day retention)
```

### 6.3 Monitoring & Logging

```javascript
// Centralized error logging
const logger = require("winston");

logger.error("Payment processing failed", {
  transactionId: "...",
  error: error.message,
  collegeId: req.college.id,
  timestamp: new Date()
});

// Alert threshold:
// - 3+ failed payments in 1 hour → Alert admin
// - >5% error rate → Page on-call engineer
// - Database connection lost → Immediate alert
```

---

## 7. ACCEPTANCE CRITERIA (ARCHITECTS)

- [ ] Every database query includes collegeId
- [ ] No hardcoded values (GST rates, dates, etc.)
- [ ] Payment idempotency keys prevent duplicates
- [ ] Razorpay webhook signature verified
- [ ] Passwords hashed with bcrypt (10 salt rounds)
- [ ] JWT tokens expire in 2 hours
- [ ] Rate limiting on auth endpoints
- [ ] Audit logs for all sensitive actions
- [ ] S3 upload validates file types (no .exe)
- [ ] API errors don't leak sensitive info

---

**Document Owner**: Technical Lead  
**Last Updated**: 2026-01-20  
**Next Review**: During architecture review with team
