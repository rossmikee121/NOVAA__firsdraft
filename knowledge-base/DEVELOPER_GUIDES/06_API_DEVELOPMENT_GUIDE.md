# API DEVELOPMENT GUIDE

**For**: Backend Developers  
**Version**: 1.0  
**Date**: January 20, 2026

---

## 📡 OVERVIEW

This guide explains how to build REST APIs in NOVAA backend using Express.js.

You'll learn:
- Creating API endpoints
- Request/response patterns
- Error handling
- Validation
- Authentication
- Status codes

---

## 🏗️ API STRUCTURE

Each module has this structure:

```
backend/src/modules/[MODULE_NAME]/
├── routes.js           # URL endpoints
├── controller.js       # Request handlers
├── service.js          # Business logic
├── model.js            # Mongoose schema
├── validation.js       # Input validation
└── middleware.js       # Custom middleware
```

### Data Flow

```
Browser Request
    ↓
Express Middleware (auth, validation, etc.)
    ↓
Router (routes.js) - Routes to correct handler
    ↓
Controller (controller.js) - Receives request
    ↓
Service (service.js) - Business logic
    ↓
Database (model.js)
    ↓
Response sent back to browser
```

---

## 🔌 SETTING UP A NEW API ENDPOINT

### Step 1: Create Mongoose Schema (model.js)

```javascript
// backend/src/modules/admissions/model.js

const mongoose = require('mongoose');

const admissionSchema = new mongoose.Schema(
  {
    collegeId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'College',
      required: true,
      index: true
    },
    studentId: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true
    },
    courseId: {
      type: String,
      required: true
    },
    status: {
      type: String,
      enum: ['DRAFT', 'SUBMITTED', 'VERIFIED', 'APPROVED', 'REJECTED'],
      default: 'DRAFT'
    },
    documents: [{
      category: String,
      url: String,
      uploadedAt: Date
    }],
    createdAt: {
      type: Date,
      default: Date.now
    }
  },
  { timestamps: true }
);

// Create unique index per college
admissionSchema.index({ collegeId: 1, status: 1 });

module.exports = mongoose.model('Admission', admissionSchema);
```

### Step 2: Create Validation (validation.js)

```javascript
// backend/src/modules/admissions/validation.js

const validateApplicationSubmission = (data) => {
  const errors = [];

  // Check required fields
  if (!data.courseId) {
    errors.push('Course ID is required');
  }

  if (!data.studentEmail) {
    errors.push('Email is required');
  } else if (!isValidEmail(data.studentEmail)) {
    errors.push('Email format is invalid');
  }

  if (!data.documents || data.documents.length === 0) {
    errors.push('At least one document must be uploaded');
  }

  // Validate document categories
  const validCategories = ['AADHAAR', 'MARKSHEET', 'CASTE'];
  data.documents?.forEach((doc, index) => {
    if (!validCategories.includes(doc.category)) {
      errors.push(`Document ${index + 1} has invalid category`);
    }
  });

  return {
    isValid: errors.length === 0,
    errors
  };
};

const isValidEmail = (email) => {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
};

module.exports = {
  validateApplicationSubmission,
  isValidEmail
};
```

### Step 3: Create Service (service.js)

```javascript
// backend/src/modules/admissions/service.js

const Admission = require('./model');
const { validateApplicationSubmission } = require('./validation');
const { uploadToS3 } = require('../../utils/s3');

const submitApplication = async (collegeId, studentId, formData) => {
  // Validate input
  const validation = validateApplicationSubmission(formData);
  if (!validation.isValid) {
    throw new Error(validation.errors.join(', '));
  }

  // Check for existing application
  const existing = await Admission.findOne({
    collegeId,
    studentId,
    status: { $ne: 'REJECTED' }
  });

  if (existing) {
    throw new Error('You already have an active application');
  }

  // Upload documents to S3
  const uploadedDocuments = [];
  for (const doc of formData.documents) {
    const s3Url = await uploadToS3(
      doc.file,
      `admissions/${collegeId}/${studentId}`
    );
    
    uploadedDocuments.push({
      category: doc.category,
      url: s3Url,
      uploadedAt: new Date()
    });
  }

  // Create admission record
  const admission = new Admission({
    collegeId,
    studentId,
    courseId: formData.courseId,
    documents: uploadedDocuments,
    status: 'SUBMITTED'
  });

  await admission.save();

  // Send confirmation email
  await sendConfirmationEmail(formData.studentEmail, admission._id);

  return admission;
};

const getApplicationStatus = async (collegeId, admissionId) => {
  const admission = await Admission.findOne({
    _id: admissionId,
    collegeId  // MULTI-TENANCY
  });

  if (!admission) {
    throw new Error('Application not found');
  }

  return admission;
};

module.exports = {
  submitApplication,
  getApplicationStatus
};
```

### Step 4: Create Controller (controller.js)

```javascript
// backend/src/modules/admissions/controller.js

const admissionService = require('./service');

// POST /api/admissions/apply
const submitApplication = async (req, res) => {
  try {
    // req.college is attached by middleware
    const admission = await admissionService.submitApplication(
      req.college._id,
      req.user._id,
      req.body
    );

    res.status(201).json({
      success: true,
      message: 'Application submitted successfully',
      data: {
        admissionId: admission._id,
        status: admission.status
      }
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: error.message
    });
  }
};

// GET /api/admissions/:id
const getApplication = async (req, res) => {
  try {
    const admission = await admissionService.getApplicationStatus(
      req.college._id,
      req.params.id
    );

    res.status(200).json({
      success: true,
      data: admission
    });
  } catch (error) {
    res.status(404).json({
      success: false,
      message: error.message
    });
  }
};

module.exports = {
  submitApplication,
  getApplication
};
```

### Step 5: Create Routes (routes.js)

```javascript
// backend/src/modules/admissions/routes.js

const express = require('express');
const controller = require('./controller');
const { authenticate } = require('../../middleware/auth');
const { enforceCollegeContext } = require('../../middleware/collegeContext');

const router = express.Router();

// Apply middleware to all routes in this module
router.use(enforceCollegeContext);
router.use(authenticate);

// POST /api/admissions/apply
router.post('/apply', controller.submitApplication);

// GET /api/admissions/:id
router.get('/:id', controller.getApplication);

module.exports = router;
```

### Step 6: Register Routes in app.js

```javascript
// backend/src/app.js

const express = require('express');
const admissionsRoutes = require('./modules/admissions/routes');

const app = express();

// Middleware
app.use(express.json());
app.use(enforceCollegeContext);
app.use(authenticate);

// Routes
app.use('/api/admissions', admissionsRoutes);
app.use('/api/payments', paymentsRoutes);
app.use('/api/attendance', attendanceRoutes);
// ... other modules

module.exports = app;
```

---

## ✅ HTTP STATUS CODES

Always use correct HTTP status codes:

```
200 OK              - Request succeeded, returning data
201 CREATED         - Resource created successfully (POST)
204 NO CONTENT      - Successful, no data to return
400 BAD REQUEST     - Client error (invalid data)
401 UNAUTHORIZED    - Authentication required/failed
403 FORBIDDEN       - Authenticated but not authorized
404 NOT FOUND       - Resource doesn't exist
409 CONFLICT        - Data conflict (duplicate email, etc.)
500 SERVER ERROR    - Server error
```

### Examples

```javascript
// 200 OK - Getting data
res.status(200).json({ data: user });

// 201 CREATED - Creating resource
res.status(201).json({ data: newAdmission });

// 400 BAD REQUEST - Invalid data
res.status(400).json({ error: 'Invalid email format' });

// 401 UNAUTHORIZED - No token
res.status(401).json({ error: 'No authentication token provided' });

// 403 FORBIDDEN - No permission
res.status(403).json({ error: 'Only admins can approve applications' });

// 404 NOT FOUND - Resource missing
res.status(404).json({ error: 'Admission not found' });

// 409 CONFLICT - Duplicate
res.status(409).json({ error: 'Email already registered' });
```

---

## 📋 COMMON API PATTERNS

### Pattern 1: Create Resource

```javascript
// POST /api/admissions/apply
router.post('/apply', async (req, res) => {
  try {
    // 1. Validate input
    if (!req.body.courseId) {
      return res.status(400).json({ error: 'Course ID required' });
    }

    // 2. Create document
    const admission = new Admission({
      collegeId: req.college._id,
      studentId: req.user._id,
      courseId: req.body.courseId
    });

    // 3. Save to database
    await admission.save();

    // 4. Return 201 with data
    res.status(201).json({
      message: 'Application submitted',
      data: admission
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Pattern 2: Read Resource

```javascript
// GET /api/admissions/:id
router.get('/:id', async (req, res) => {
  try {
    // 1. Find document with collegeId (MULTI-TENANCY)
    const admission = await Admission.findOne({
      _id: req.params.id,
      collegeId: req.college._id
    });

    // 2. Check if exists
    if (!admission) {
      return res.status(404).json({ error: 'Not found' });
    }

    // 3. Return 200
    res.status(200).json({ data: admission });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Pattern 3: Update Resource

```javascript
// PUT /api/admissions/:id/approve
router.put('/:id/approve', async (req, res) => {
  try {
    // 1. Verify authorization (admin only)
    if (req.user.role !== 'ADMIN') {
      return res.status(403).json({ error: 'Not authorized' });
    }

    // 2. Update with conditions
    const admission = await Admission.findOneAndUpdate(
      {
        _id: req.params.id,
        collegeId: req.college._id,
        status: 'VERIFIED'  // Can only approve from VERIFIED status
      },
      { $set: { status: 'APPROVED' } },
      { new: true }  // Return updated document
    );

    // 3. Check if update happened
    if (!admission) {
      return res.status(400).json({
        error: 'Cannot approve application in current status'
      });
    }

    // 4. Return 200
    res.status(200).json({
      message: 'Application approved',
      data: admission
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Pattern 4: Delete Resource

```javascript
// DELETE /api/admissions/:id
router.delete('/:id', async (req, res) => {
  try {
    // 1. Verify authorization
    if (req.user.role !== 'ADMIN') {
      return res.status(403).json({ error: 'Not authorized' });
    }

    // 2. Delete
    const result = await Admission.deleteOne({
      _id: req.params.id,
      collegeId: req.college._id
    });

    // 3. Check if deleted
    if (result.deletedCount === 0) {
      return res.status(404).json({ error: 'Not found' });
    }

    // 4. Return 204 (no content)
    res.status(204).send();
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Pattern 5: List with Filtering

```javascript
// GET /api/admissions?status=PENDING&page=1&limit=20
router.get('/', async (req, res) => {
  try {
    const { status, page = 1, limit = 20 } = req.query;

    // 1. Build query
    const query = { collegeId: req.college._id };
    if (status) {
      query.status = status;
    }

    // 2. Pagination
    const skip = (page - 1) * limit;

    // 3. Execute query
    const admissions = await Admission.find(query)
      .skip(skip)
      .limit(Number(limit))
      .sort({ createdAt: -1 });

    const total = await Admission.countDocuments(query);

    // 4. Return with pagination info
    res.status(200).json({
      data: admissions,
      pagination: {
        page: Number(page),
        limit: Number(limit),
        total,
        pages: Math.ceil(total / limit)
      }
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

### Pattern 6: Aggregation (Complex Query)

```javascript
// GET /api/reports/admission-funnel
router.get('/admission-funnel', async (req, res) => {
  try {
    const funnel = await Admission.aggregate([
      // 1. Filter by college
      {
        $match: { collegeId: req.college._id }
      },
      
      // 2. Group by status
      {
        $group: {
          _id: '$status',
          count: { $sum: 1 }
        }
      },
      
      // 3. Sort
      {
        $sort: { _id: 1 }
      }
    ]);

    // 4. Calculate percentages
    const total = funnel.reduce((sum, item) => sum + item.count, 0);
    const result = funnel.map(item => ({
      status: item._id,
      count: item.count,
      percentage: ((item.count / total) * 100).toFixed(2)
    }));

    res.status(200).json({ data: result });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```

---

## 🔐 MULTI-TENANCY IN APIs

**Critical**: Every API must filter by collegeId

```javascript
// ✅ CORRECT - Filters by college
router.get('/:id', async (req, res) => {
  const admission = await Admission.findOne({
    _id: req.params.id,
    collegeId: req.college._id  // ESSENTIAL
  });
  
  if (!admission) {
    return res.status(404).json({ error: 'Not found' });
  }
  
  res.json({ data: admission });
});

// ❌ WRONG - Missing collegeId filter
router.get('/:id', async (req, res) => {
  const admission = await Admission.findById(req.params.id);
  // Could return data from ANOTHER college!
  
  res.json({ data: admission });
});
```

---

## 🛡️ ERROR HANDLING

Always handle errors gracefully:

```javascript
router.post('/apply', async (req, res) => {
  try {
    // Validation
    if (!req.body.courseId) {
      return res.status(400).json({
        success: false,
        message: 'Course ID is required'
      });
    }

    // Try operation
    const admission = await admissionService.submitApplication(
      req.college._id,
      req.user._id,
      req.body
    );

    res.status(201).json({
      success: true,
      data: admission
    });

  } catch (error) {
    // Handle different error types
    if (error.message.includes('duplicate')) {
      return res.status(409).json({
        success: false,
        message: 'Application already exists'
      });
    }

    if (error.message.includes('permission')) {
      return res.status(403).json({
        success: false,
        message: 'You do not have permission'
      });
    }

    // Generic error
    res.status(500).json({
      success: false,
      message: 'Internal server error',
      error: process.env.NODE_ENV === 'development' ? error.message : undefined
    });
  }
});
```

---

## 📝 RESPONSE FORMAT

Always follow consistent response format:

```javascript
// Success response
{
  success: true,
  message: "Operation successful",
  data: {
    // Actual data
  }
}

// Error response
{
  success: false,
  message: "Error description",
  error: "ERROR_CODE" // Optional
}

// List response
{
  success: true,
  data: [...],
  pagination: {
    page: 1,
    limit: 20,
    total: 150,
    pages: 8
  }
}
```

---

## 🧪 TESTING APIS

### Using Thunder Client (VS Code Extension)

1. Open Thunder Client
2. Click "New Request"
3. Set method (GET, POST, etc.)
4. Enter URL: `http://localhost:5000/api/admissions/123`
5. Add headers: 
   - `Content-Type: application/json`
   - `x-college-code: ST_XAVIER_MUMBAI`
6. Add body (for POST/PUT)
7. Click "Send"

### Using Postman

1. Open Postman
2. Create new request
3. Set method and URL
4. Add Authorization tab
5. Set request/response format
6. Send request

### Using curl (Terminal)

```bash
# GET request
curl -H "x-college-code: ST_XAVIER_MUMBAI" \
  http://localhost:5000/api/admissions/123

# POST request
curl -X POST \
  -H "Content-Type: application/json" \
  -H "x-college-code: ST_XAVIER_MUMBAI" \
  -d '{"courseId":"BSC_CS"}' \
  http://localhost:5000/api/admissions/apply
```

---

## 📖 COMPLETE API ENDPOINT CHECKLIST

When creating new endpoint:

- [ ] Endpoint has clear HTTP method (GET/POST/PUT/DELETE)
- [ ] Path is logical (e.g., `/api/admissions/:id`)
- [ ] Authentication middleware applied
- [ ] College context enforced (collegeId filter)
- [ ] Input validation included
- [ ] Error handling for all cases
- [ ] Correct HTTP status code returned
- [ ] Response format consistent
- [ ] Documentation included
- [ ] Tests written

---

## 🚨 SECURITY CHECKLIST

- [ ] No passwords in responses
- [ ] No sensitive data exposed
- [ ] SQL injection prevented (using Mongoose)
- [ ] XSS prevented (JSON responses only)
- [ ] Rate limiting applied
- [ ] CORS configured correctly
- [ ] JWT tokens validate collegeId
- [ ] Multi-tenancy enforced
- [ ] Input sanitized
- [ ] Logs don't contain secrets

---

## 📊 NEXT DOCUMENTS

1. ✅ **01_MERN_STACK_OVERVIEW.md**
2. ✅ **02_DEVELOPMENT_ENVIRONMENT_SETUP.md**
3. ✅ **03_MODULES_ARCHITECTURE.md**
4. ✅ **04_MODULE_INTERCONNECTIONS.md**
5. ✅ **05_DATABASE_DEVELOPER_GUIDE.md**
6. ✅ **06_API_DEVELOPMENT_GUIDE.md** (You are here)
7. **07_FRONTEND_DEVELOPMENT_GUIDE.md**
8. **08_CODE_STANDARDS_CONVENTIONS.md**
9. **09_AUTHENTICATION_SECURITY_GUIDE.md**
10. **10_PAYMENT_PROCESSING_GUIDE.md**
11. **11_TESTING_DEVELOPER_GUIDE.md**
12. **12_DEBUGGING_TROUBLESHOOTING.md**

---

**Previous**: [05_DATABASE_DEVELOPER_GUIDE.md](05_DATABASE_DEVELOPER_GUIDE.md)  
**Next Document**: [07_FRONTEND_DEVELOPMENT_GUIDE.md](07_FRONTEND_DEVELOPMENT_GUIDE.md)

