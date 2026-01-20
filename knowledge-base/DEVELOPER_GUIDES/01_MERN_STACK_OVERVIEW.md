# NOVAA DEVELOPER GUIDES - MERN DEVELOPMENT

**For**: Backend Developers, Frontend Developers, Full-Stack Developers  
**Version**: 1.0  
**Date**: January 20, 2026  

---

## 📁 DEVELOPER GUIDES FOLDER STRUCTURE

```
DEVELOPER_GUIDES/
├── 01_MERN_STACK_OVERVIEW.md (THIS FILE)
│   └─ Complete introduction to MERN architecture
│
├── 02_DEVELOPMENT_ENVIRONMENT_SETUP.md
│   └─ Step-by-step local development setup
│
├── 03_MODULES_ARCHITECTURE.md
│   └─ How NOVAA is organized into modules
│
├── 04_MODULE_INTERCONNECTIONS.md
│   └─ How modules communicate with each other
│
├── 05_DATABASE_DEVELOPER_GUIDE.md
│   └─ MongoDB schema, queries, indexing
│
├── 06_API_DEVELOPMENT_GUIDE.md
│   └─ Creating REST APIs with Express
│
├── 07_FRONTEND_DEVELOPMENT_GUIDE.md
│   └─ React component structure & best practices
│
├── 08_CODE_STANDARDS_CONVENTIONS.md
│   └─ Coding standards & project conventions
│
├── 09_AUTHENTICATION_SECURITY_GUIDE.md
│   └─ JWT, multi-tenancy, security patterns
│
├── 10_PAYMENT_PROCESSING_GUIDE.md
│   └─ Razorpay integration & payment flows
│
├── 11_TESTING_DEVELOPER_GUIDE.md
│   └─ Unit tests, integration tests, E2E tests
│
└── 12_DEBUGGING_TROUBLESHOOTING.md
    └─ Common issues & debugging techniques
```

---

# 📚 MERN STACK OVERVIEW FOR NOVAA

## 1. WHAT IS MERN?

MERN is a full-stack JavaScript framework combining:

```
┌─────────────────────────────────────────────┐
│              MERN STACK                     │
├─────────────────────────────────────────────┤
│                                             │
│  M → MongoDB       (Database Layer)         │
│  E → Express       (Backend Framework)      │
│  R → React         (Frontend Framework)     │
│  N → Node.js       (JavaScript Runtime)     │
│                                             │
└─────────────────────────────────────────────┘
```

### Why MERN for NOVAA?

✅ **Single Language**: JavaScript everywhere (easier team coordination)  
✅ **Flexible Database**: MongoDB adapts to changing requirements  
✅ **Component-Based UI**: React for building complex dashboards  
✅ **Scalable Backend**: Node.js handles concurrent connections  
✅ **Rich Ecosystem**: Mature libraries for every need  
✅ **Performance**: Async I/O perfect for real-time applications  

---

## 2. MERN ARCHITECTURE FLOW

### Request-Response Cycle

```
CLIENT SIDE (React - Browser)
     │
     ├─ User clicks button
     │
     ├─ Event handler triggered
     │
     ├─ API call made (fetch/axios)
     │          │
     │          │ HTTP Request
     │          ▼
     │
SERVER SIDE (Node.js + Express)
     │
     ├─ Request received at endpoint
     │
     ├─ Middleware processes (auth, logging, validation)
     │
     ├─ Route handler executes business logic
     │
     ├─ Database query (MongoDB)
     │          │
     │          │ Query
     │          ▼
     │   MongoDB Atlas
     │          │
     │          │ Result
     │          ▲
     │
     ├─ Format response
     │
     └─ Send back to client
              │
              │ HTTP Response
              ▼

CLIENT SIDE (React - Browser)
     │
     ├─ Response received
     │
     ├─ State updated (setState/Redux)
     │
     ├─ Component re-renders
     │
     └─ User sees updated UI
```

### Layer Breakdown

**Frontend Layer (React)**
- User Interface Components
- State Management
- API Communication
- Client-side Routing

**Backend Layer (Node.js + Express)**
- API Endpoints
- Business Logic
- Authentication
- Data Validation
- Error Handling

**Database Layer (MongoDB)**
- Data Storage
- Indexing
- Aggregations
- Transactions

---

## 3. PROJECT DIRECTORY STRUCTURE

```
novaa/
│
├── backend/                          # Node.js + Express
│   ├── src/
│   │   ├── config/                   # Configuration files
│   │   │   ├── database.js           # MongoDB connection
│   │   │   ├── env.js                # Environment variables
│   │   │   └── constants.js          # App constants
│   │   │
│   │   ├── middlewares/              # Express middlewares
│   │   │   ├── auth.js               # JWT verification
│   │   │   ├── multiTenancy.js       # College context enforcement
│   │   │   ├── errorHandler.js       # Global error handler
│   │   │   └── logging.js            # Request/response logging
│   │   │
│   │   ├── modules/                  # Feature modules
│   │   │   ├── admissions/           # Admissions feature
│   │   │   │   ├── routes.js
│   │   │   │   ├── controller.js
│   │   │   │   ├── service.js
│   │   │   │   ├── model.js
│   │   │   │   └── validation.js
│   │   │   │
│   │   │   ├── payments/             # Payments feature
│   │   │   │   ├── routes.js
│   │   │   │   ├── controller.js
│   │   │   │   ├── service.js
│   │   │   │   ├── model.js
│   │   │   │   └── validation.js
│   │   │   │
│   │   │   ├── attendance/           # Attendance feature
│   │   │   ├── auth/                 # Authentication
│   │   │   ├── colleges/             # College management
│   │   │   └── reports/              # Analytics & reporting
│   │   │
│   │   ├── utils/                    # Utility functions
│   │   │   ├── validators.js         # Input validation
│   │   │   ├── formatters.js         # Response formatting
│   │   │   ├── errors.js             # Custom error classes
│   │   │   └── razorpay.js           # Payment gateway helper
│   │   │
│   │   └── server.js                 # Express app entry point
│   │
│   ├── tests/                        # Test files
│   ├── .env.example                  # Environment template
│   ├── package.json
│   └── package-lock.json
│
├── frontend/                         # React
│   ├── src/
│   │   ├── components/               # Reusable React components
│   │   │   ├── Navbar.js
│   │   │   ├── Sidebar.js
│   │   │   ├── Forms/
│   │   │   ├── Tables/
│   │   │   ├── Modals/
│   │   │   └── Common/
│   │   │
│   │   ├── pages/                    # Page components
│   │   │   ├── Admin/
│   │   │   │   ├── Dashboard.js
│   │   │   │   ├── Admissions.js
│   │   │   │   ├── Payments.js
│   │   │   │   └── Reports.js
│   │   │   │
│   │   │   ├── Staff/
│   │   │   │   └── AttendanceMarking.js
│   │   │   │
│   │   │   ├── Student/
│   │   │   │   ├── ApplicationForm.js
│   │   │   │   ├── Dashboard.js
│   │   │   │   └── PaymentPage.js
│   │   │   │
│   │   │   └── Auth/
│   │   │       ├── Login.js
│   │   │       └── Register.js
│   │   │
│   │   ├── services/                 # API communication
│   │   │   ├── api.js                # Axios instance with interceptors
│   │   │   ├── authService.js        # Auth API calls
│   │   │   ├── admissionsService.js  # Admissions API calls
│   │   │   ├── paymentsService.js    # Payments API calls
│   │   │   └── attendanceService.js  # Attendance API calls
│   │   │
│   │   ├── context/                  # React Context (state management)
│   │   │   ├── AuthContext.js        # Auth state
│   │   │   ├── CollegeContext.js     # College context
│   │   │   └── NotificationContext.js# Toast notifications
│   │   │
│   │   ├── hooks/                    # Custom React hooks
│   │   │   ├── useAuth.js
│   │   │   ├── useCollegeContext.js
│   │   │   └── useFetch.js
│   │   │
│   │   ├── utils/                    # Frontend utilities
│   │   │   ├── formatters.js         # Date/currency formatting
│   │   │   ├── validators.js         # Client-side validation
│   │   │   └── constants.js
│   │   │
│   │   ├── styles/                   # CSS/SCSS
│   │   │   ├── globals.css
│   │   │   ├── variables.css
│   │   │   └── responsive.css
│   │   │
│   │   ├── App.js                    # Main React component
│   │   ├── index.js                  # React entry point
│   │   └── index.css
│   │
│   ├── public/                       # Static assets
│   ├── package.json
│   └── .env.example
│
├── docs/                             # This folder - documentation
│   └── (all your documentation)
│
└── .gitignore
```

---

## 4. DATA FLOW BETWEEN LAYERS

### Example: Student Submits Admission Application

```
STEP 1: Frontend (React Component)
┌─────────────────────────────────────────┐
│ User fills form:                        │
│ - Name, Email, Phone                    │
│ - Caste Category                        │
│ - Upload documents                      │
│                                         │
│ Validation: Client-side checks          │
│ - Email format valid?                   │
│ - File size < 5MB?                      │
│                                         │
│ Form submitted → Call API               │
└─────────────────────────────────────────┘
         │
         │ POST /api/admissions/apply
         │ Body: { name, email, phone, casteCategory, documents }
         ▼

STEP 2: Backend (Node.js + Express)
┌─────────────────────────────────────────┐
│ Middleware Chain:                       │
│ 1. requireCollegeContext()               │
│    └─ Attach req.college from headers   │
│                                         │
│ 2. authenticate()                       │
│    └─ Verify JWT token                  │
│                                         │
│ 3. validateApplicationInput()            │
│    └─ Check required fields              │
│                                         │
│ Route Handler (controller.js)            │
│ • Extract data from request              │
│ • Call service layer                     │
│                                         │
│ Service Layer (service.js)               │
│ • Validate data (business rules)         │
│ • Upload documents to S3                 │
│ • Create database record                 │
│ • Generate confirmation email            │
│                                         │
│ Response:                                │
│ {                                        │
│   status: "success",                     │
│   applicationId: "APP_12345",            │
│   message: "Application submitted"       │
│ }                                        │
└─────────────────────────────────────────┘
         │
         │ Response back to client
         ▼

STEP 3: Frontend (React)
┌─────────────────────────────────────────┐
│ Response received                        │
│ ├─ Parse JSON                           │
│ ├─ Update component state                │
│ ├─ Show success message                  │
│ └─ Redirect to status page               │
│                                         │
│ User sees: "Application submitted!"      │
│ with Application ID                      │
└─────────────────────────────────────────┘

STEP 4: Database (MongoDB)
┌─────────────────────────────────────────┐
│ Document inserted in admissions          │
│ collection:                              │
│                                         │
│ {                                        │
│   _id: ObjectId,                        │
│   collegeId: ObjectId,                  │
│   studentId: ObjectId,                  │
│   status: "SUBMITTED",                  │
│   documents: [ ... ],                   │
│   createdAt: Date.now(),                │
│   ...                                    │
│ }                                        │
└─────────────────────────────────────────┘
```

---

## 5. KEY MERN TECHNOLOGIES

### Backend Technologies

```
Node.js
├─ Express (Web framework)
├─ Mongoose (MongoDB ODM)
├─ JWT (Authentication)
├─ Bcrypt (Password hashing)
├─ Razorpay (Payment gateway)
├─ Multer (File uploads)
├─ Nodemailer (Email sending)
└─ Winston (Logging)

Middleware Concept:
app.use(middleware1) → app.use(middleware2) → Route Handler
```

### Frontend Technologies

```
React
├─ Context API (State management - MVP level)
├─ Axios (HTTP client)
├─ React Router (Navigation)
├─ React Icons (Icons)
├─ Qrcode.react (QR generation)
├─ Moment.js (Date formatting)
└─ React Toastify (Notifications)
```

### Database

```
MongoDB
├─ Document-based NoSQL
├─ Flexible schema (adapts to changes)
├─ Indexing (performance optimization)
├─ Aggregation pipeline (complex queries)
└─ Atlas (Cloud hosting)
```

---

## 6. THE THREE-DEVELOPER TEAM STRUCTURE

### For NOVAA MVP, we assume:

```
Developer 1: FULL-STACK (Primary)
├─ Setup project structure
├─ Create core modules (auth, admissions, payments)
├─ Implement payment processing
├─ Database schema design
└─ Deployment & DevOps

Developer 2: FRONTEND FOCUS
├─ Build React components
├─ Create admin dashboard
├─ Attendance marking UI
├─ Student portal
└─ Responsive design

Developer 3: BACKEND FOCUS
├─ Implement APIs
├─ Database optimization
├─ Security & middleware
├─ Testing & validation
└─ Razorpay integration

Collaboration:
• Daily standup (15 min)
• Feature branches (Git workflow)
• Code reviews before merge
• Shared documentation
• Regular integration testing
```

### Work Division by Module

```
Admissions Module
├─ Backend API (Dev 1 + Dev 3)
├─ Frontend UI (Dev 2)
└─ Database schema (Dev 1)

Payments Module
├─ Razorpay integration (Dev 1)
├─ Payment UI (Dev 2)
├─ Webhook handling (Dev 3)
└─ Transaction schema (Dev 1)

Attendance Module
├─ QR generation (Dev 1)
├─ QR scanner UI (Dev 2)
├─ Attendance API (Dev 3)
└─ Real-time updates (Dev 1)

Auth Module
├─ JWT implementation (Dev 1 + Dev 3)
├─ Login/Register forms (Dev 2)
└─ Multi-tenancy enforcement (Dev 3)
```

---

## 7. DEVELOPMENT WORKFLOW

### Git Workflow

```
main (production code - protected)
├─ development (integration branch)
│  ├─ feature/admissions (Dev 1)
│  ├─ feature/payments (Dev 2)
│  ├─ feature/attendance (Dev 3)
│  └─ bugfix/auth (Dev 1)
│
└─ (Never code directly on main/development)
```

### Weekly Workflow

```
MONDAY 9 AM: Sprint Planning
└─ Divide tasks for the week
└─ Create feature branches
└─ Assign modules to developers

MONDAY-FRIDAY: Development
├─ Each dev works on their feature branch
├─ Daily standup (4 PM): Status updates
├─ Code review before merge (min 1 approval)
└─ Merge to development when done

FRIDAY 4 PM: Integration Testing
├─ All features merged to development
├─ Test complete workflows
├─ Fix integration issues
└─ Friday evening: Working system ready

NEXT MONDAY: Rinse and Repeat
```

---

## 8. CODING STANDARDS (TL;DR)

### JavaScript/Node.js

```javascript
// ✅ DO:
const calculateFee = (amount) => amount * 1.18; // Clear naming
const users = await User.find({ collegeId }); // Multi-tenancy enforced
await Transaction.create({ idempotencyKey, ...data }); // Idempotency

// ❌ DON'T:
const x = y * z; // Unclear variable names
const all_users = await User.find({}); // Missing collegeId filter!
async function(err) { console.log(err); } // Swallow errors
```

### React Components

```javascript
// ✅ DO:
function AdmissionForm() {
  const [formData, setFormData] = useState({});
  
  const handleSubmit = async () => {
    const validation = validateForm(formData);
    if (!validation.valid) return setError(validation.errors);
    
    const response = await submitApplication(formData);
    if (response.success) setStatus("submitted");
  };
  
  return <form onSubmit={handleSubmit}>...</form>;
}

// ❌ DON'T:
function Form() {
  const [a, setA] = useState(); // Unclear variable names
  
  const handleClick = () => {
    // Too much logic in component
    // Database calls mixed with UI
  };
}
```

---

## 9. COMMON PATTERNS & CONVENTIONS

### Backend API Pattern

```javascript
// GET request with multi-tenancy
app.get('/api/admissions', authenticate, async (req, res) => {
  try {
    const admissions = await Admission.find({
      collegeId: req.college.id,
      status: req.query.status
    });
    
    res.json({
      status: 'success',
      data: admissions,
      count: admissions.length
    });
  } catch (error) {
    res.status(500).json({ status: 'error', message: error.message });
  }
});

// POST request with idempotency
app.post('/api/payments', authenticate, async (req, res) => {
  const { idempotencyKey, amount } = req.body;
  
  // Check if already processed
  const existing = await Transaction.findOne({ idempotencyKey });
  if (existing) return res.json(existing);
  
  // Process new payment
  const transaction = await createTransaction({
    collegeId: req.college.id,
    idempotencyKey,
    amount
  });
  
  res.json({ status: 'success', data: transaction });
});
```

### Frontend Pattern

```javascript
// API Service (services/admissionsService.js)
export const submitApplication = async (data) => {
  try {
    const response = await api.post('/admissions/apply', data);
    return { success: true, data: response.data };
  } catch (error) {
    return { success: false, error: error.message };
  }
};

// React Component using the service
function ApplicationForm() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  const handleSubmit = async (formData) => {
    setLoading(true);
    const result = await submitApplication(formData);
    
    if (result.success) {
      setStatus('submitted');
    } else {
      setError(result.error);
    }
    setLoading(false);
  };
  
  return (
    <form onSubmit={handleSubmit}>
      {error && <div className="error">{error}</div>}
      {loading && <Spinner />}
      {/* Form fields */}
    </form>
  );
}
```

---

## 10. USEFUL COMMANDS

### Backend

```bash
# Start development server (watches for changes)
npm run dev

# Run tests
npm test

# Database connection test
npm run test:db

# See all users across all colleges (debug only)
npm run debug:query "db.users.find({})"
```

### Frontend

```bash
# Start development server
npm start

# Build for production
npm run build

# Run tests
npm test

# Check code quality
npm run lint
```

### Git Workflow

```bash
# Create new feature branch
git checkout -b feature/admissions

# Push to GitHub
git push origin feature/admissions

# Create pull request on GitHub (ask for code review)

# After approval, merge to development
git checkout development
git merge feature/admissions
git push origin development

# Delete feature branch when done
git branch -d feature/admissions
```

---

## 11. COMMUNICATION BETWEEN DEVELOPERS

### Daily Standup Topics

```
"What did I accomplish yesterday?"
→ Implemented admission form validation
→ Created API endpoint for document upload

"What am I working on today?"
→ Adding GST calculation logic
→ Testing payment flow

"Any blockers?"
→ Need DB schema finalized by Dev 1
→ Waiting for API spec from Dev 3
```

### Code Review Checklist

Before merging, reviewer checks:
- ✅ Every query includes collegeId
- ✅ No console.logs left in code
- ✅ Error handling present
- ✅ Follows naming conventions
- ✅ Tests included
- ✅ No hardcoded values

---

## 12. NEXT STEPS FOR DEVELOPERS

1. **Read Next**: 02_DEVELOPMENT_ENVIRONMENT_SETUP.md
2. **Then Read**: 03_MODULES_ARCHITECTURE.md
3. **Understand**: 04_MODULE_INTERCONNECTIONS.md
4. **Reference**: Code standards in 08_CODE_STANDARDS_CONVENTIONS.md

---

## 💡 KEY TAKEAWAY

NOVAA uses MERN because it allows three developers to:
- **Collaborate easily** (same language everywhere)
- **Move fast** (mature frameworks, less boilerplate)
- **Build securely** (patterns built into architecture)
- **Scale later** (databases can be sharded, services can be split)

**Each developer focuses on their strength, and the architecture ensures they fit together perfectly.**

---

**Next Document**: [02_DEVELOPMENT_ENVIRONMENT_SETUP.md](02_DEVELOPMENT_ENVIRONMENT_SETUP.md)

