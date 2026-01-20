# AUTHENTICATION & SECURITY GUIDE

**For**: All Developers  
**Version**: 1.0  
**Date**: January 20, 2026  
**Critical**: Read before writing any authentication or database code

---

## 🔐 OVERVIEW

NOVAA is a **multi-tenant system** where data isolation is **critical**. Each college's data MUST NOT leak to other colleges. This guide explains how to implement secure, compliant authentication.

---

## 🎯 AUTHENTICATION FLOW

### Complete Login Workflow

```javascript
// 1. USER SUBMITS CREDENTIALS (Frontend)
// POST /api/auth/login
{
  email: "admin@college.edu",
  password: "SecurePassword123"
}

// 2. SERVER VALIDATES (Backend)
// - Find user by email
// - Verify password using bcrypt
// - Generate JWT token
// - Return token + user info

// 3. STORE TOKEN (Frontend)
// localStorage.setItem('token', responseToken)

// 4. SEND TOKEN IN REQUESTS (Frontend)
// Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

// 5. VERIFY TOKEN (Backend middleware)
// - Extract token from header
// - Verify with JWT_SECRET
// - Attach user to request
// - Check collegeId matches
```

### Login Endpoint (Backend)

```javascript
// routes/auth.js
router.post('/login', validate(loginSchema), authController.login);

// controllers/authController.js
const login = async (req, res) => {
  try {
    // 1. Validate email/password format
    const { email, password } = req.body;
    
    // 2. Find user
    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // 3. Compare password with bcrypt
    const isPasswordValid = await bcrypt.compare(password, user.password);
    if (!isPasswordValid) {
      return res.status(401).json({ error: 'Invalid credentials' });
    }

    // 4. Check if user is active
    if (!user.isActive) {
      return res.status(403).json({ error: 'Account disabled' });
    }

    // 5. Generate JWT token (valid for 2 hours)
    const token = jwt.sign(
      {
        userId: user._id,
        email: user.email,
        collegeId: user.collegeId,
        role: user.role
      },
      process.env.JWT_SECRET,
      { expiresIn: '2h' }
    );

    // 6. Remove password from response
    const userResponse = user.toObject();
    delete userResponse.password;

    // 7. Return token + user
    res.status(200).json({
      token,
      user: userResponse,
      expiresIn: '2h'
    });

  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({ error: 'Login failed' });
  }
};

// services/authService.js
class AuthService {
  static async loginUser(email, password) {
    // Validate inputs
    if (!email || !password) {
      throw new Error('Email and password required');
    }

    // Find user
    const user = await User.findOne({ email }).select('+password');
    if (!user) {
      throw new Error('Invalid credentials');
    }

    // Compare passwords
    const isValid = await bcrypt.compare(password, user.password);
    if (!isValid) {
      throw new Error('Invalid credentials');
    }

    // Generate token
    const token = jwt.sign(
      {
        userId: user._id,
        email: user.email,
        collegeId: user.collegeId,
        role: user.role
      },
      process.env.JWT_SECRET,
      { expiresIn: '2h' }
    );

    return { token, user };
  }
}
```

---

## 🔑 JWT TOKEN MANAGEMENT

### Token Structure

```javascript
const token = jwt.sign(
  // PAYLOAD (what's encoded in token)
  {
    userId: '507f1f77bcf86cd799439011',
    email: 'admin@college.edu',
    collegeId: '507f1f77bcf86cd799439012',  // CRITICAL: All multi-tenant apps need this
    role: 'ADMIN'
  },
  // SECRET (signs the token)
  process.env.JWT_SECRET,
  // OPTIONS
  { 
    expiresIn: '2h',
    issuer: 'NOVAA',
    subject: 'user-auth'
  }
);
```

### Token Verification Middleware

```javascript
// middleware/authentication.js
const authenticate = (req, res, next) => {
  try {
    // 1. Extract token from header
    const authHeader = req.headers.authorization;
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({ error: 'Missing token' });
    }

    const token = authHeader.substring(7); // Remove "Bearer "

    // 2. Verify token
    const decoded = jwt.verify(token, process.env.JWT_SECRET);

    // 3. Attach to request
    req.user = {
      userId: decoded.userId,
      email: decoded.email,
      collegeId: decoded.collegeId,
      role: decoded.role
    };

    next();

  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({ error: 'Token expired' });
    }
    if (error.name === 'JsonWebTokenError') {
      return res.status(401).json({ error: 'Invalid token' });
    }
    res.status(500).json({ error: 'Authentication failed' });
  }
};

module.exports = authenticate;
```

### Token Refresh (Optional but Recommended)

```javascript
// For better UX, implement refresh tokens
const refreshToken = jwt.sign(
  { userId: user._id },
  process.env.JWT_REFRESH_SECRET,
  { expiresIn: '7d' }
);

// Store refresh token in database
await RefreshToken.create({
  token: refreshToken,
  userId: user._id,
  expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
});

// Return both tokens
res.json({
  accessToken: token,
  refreshToken: refreshToken
});

// Endpoint to refresh token
router.post('/refresh', refreshController.refresh);
```

---

## 🔐 PASSWORD HASHING WITH BCRYPT

### Hashing on Signup

```javascript
// ✅ CORRECT
const signup = async (req, res) => {
  try {
    const { email, password, collegeId, role } = req.body;

    // 1. Validate password strength
    if (password.length < 8) {
      return res.status(400).json({ error: 'Password must be 8+ characters' });
    }

    // 2. Check if user exists
    const existingUser = await User.findOne({ email, collegeId });
    if (existingUser) {
      return res.status(409).json({ error: 'Email already exists' });
    }

    // 3. Hash password (salt rounds = 10)
    const hashedPassword = await bcrypt.hash(password, 10);

    // 4. Create user
    const user = await User.create({
      email,
      password: hashedPassword,  // Store hashed, never plain text
      collegeId,
      role,
      isActive: true
    });

    // 5. Generate token
    const token = jwt.sign(
      { userId: user._id, email, collegeId, role },
      process.env.JWT_SECRET,
      { expiresIn: '2h' }
    );

    res.status(201).json({ token, user });

  } catch (error) {
    res.status(500).json({ error: 'Signup failed' });
  }
};

// ❌ WRONG
const signup_wrong = async (req, res) => {
  const { email, password } = req.body;
  
  // NEVER store plain password
  const user = await User.create({ email, password });
  res.json(user);
};
```

### Bcrypt Configuration

```javascript
// salt rounds = 10 means 2^10 = 1024 iterations
// Higher = more secure but slower
// 10 is standard for user authentication
// Don't increase beyond 12 (causes timeout)

const hashedPassword = await bcrypt.hash(plainPassword, 10);

// Verify during login
const isMatch = await bcrypt.compare(plainPassword, hashedPassword);
```

---

## 🚨 MULTI-TENANCY SECURITY

### ❌ CRITICAL VULNERABILITY: Missing CollegeId Check

```javascript
// ❌ WRONG - College A can see College B's students!
router.get('/students', authenticate, async (req, res) => {
  const students = await Student.find({});  // NO collegeId check!
  res.json(students);
});

// ✅ CORRECT - Only show current college's students
router.get('/students', authenticate, async (req, res) => {
  const students = await Student.find({
    collegeId: req.user.collegeId  // MUST include this
  });
  res.json(students);
});
```

### Enforcing CollegeId at Middleware Level

```javascript
// middleware/collegeContext.js
const enforceCollegeContext = (req, res, next) => {
  // Middleware runs AFTER authentication
  if (!req.user || !req.user.collegeId) {
    return res.status(401).json({ error: 'Invalid college context' });
  }
  
  // Attach collegeId to all queries
  req.collegeId = req.user.collegeId;
  
  next();
};

// app.js
app.use(authenticate);
app.use(enforceCollegeContext);

// Now every request has req.collegeId
// Use in queries:
await Student.find({ collegeId: req.collegeId });
await Admission.findById(id, { collegeId: req.collegeId });
```

### Database-Level Isolation

```javascript
// Model: Student
const studentSchema = new Schema({
  name: String,
  email: String,
  collegeId: {
    type: Schema.Types.ObjectId,
    ref: 'College',
    required: true,
    index: true  // CRITICAL: Index on collegeId for fast queries
  }
});

// Create unique index on email per college
studentSchema.index({ email: 1, collegeId: 1 }, { unique: true });

// Query with index
const students = await Student.find({
  collegeId: collegeId,
  email: searchEmail
});
```

### Role-Based Access Control (RBAC)

```javascript
// middleware/authorization.js
const authorize = (...allowedRoles) => {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Not authenticated' });
    }

    if (!allowedRoles.includes(req.user.role)) {
      return res.status(403).json({ 
        error: 'Insufficient permissions' 
      });
    }

    next();
  };
};

// Usage
router.post(
  '/admissions/verify',
  authenticate,
  authorize('ADMIN', 'ADMISSIONS_OFFICER'),
  verifyAdmission
);

router.delete(
  '/students/:id',
  authenticate,
  authorize('ADMIN'),
  deleteStudent
);
```

---

## 🔒 SECURE PRACTICES

### Input Validation

```javascript
// validation/schemas.js
const loginSchema = {
  email: {
    type: 'string',
    format: 'email',
    required: true
  },
  password: {
    type: 'string',
    minLength: 8,
    required: true
  }
};

// Validation middleware
const validate = (schema) => {
  return (req, res, next) => {
    const ajv = new Ajv();
    const valid = ajv.validate(schema, req.body);
    
    if (!valid) {
      return res.status(400).json({
        error: 'Validation failed',
        details: ajv.errorsText()
      });
    }

    next();
  };
};

// Apply to routes
router.post('/login', validate(loginSchema), authController.login);
```

### Rate Limiting

```javascript
// middleware/rateLimit.js
const rateLimit = require('express-rate-limit');

const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 5,                     // Max 5 attempts
  message: 'Too many login attempts, try again later',
  standardHeaders: true,
  legacyHeaders: false
});

const paymentLimiter = rateLimit({
  windowMs: 60 * 60 * 1000,   // 1 hour
  max: 10,                    // Max 10 payments/hour
  keyGenerator: (req) => `${req.user.userId}-payment`
});

// app.js
app.post('/auth/login', loginLimiter, authController.login);
app.post('/payments', paymentLimiter, paymentController.process);
```

### CORS Configuration

```javascript
// config/cors.js
const corsOptions = {
  origin: process.env.FRONTEND_URL,  // Only allow frontend
  credentials: true,                 // Allow cookies/auth headers
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
};

// app.js
app.use(cors(corsOptions));
```

### HTTPS/SSL

```javascript
// Always use HTTPS in production
// environment variable check
if (process.env.NODE_ENV === 'production') {
  const https = require('https');
  const fs = require('fs');
  
  const options = {
    key: fs.readFileSync('/path/to/private-key.pem'),
    cert: fs.readFileSync('/path/to/certificate.pem')
  };
  
  https.createServer(options, app).listen(443);
} else {
  app.listen(5000);
}
```

---

## 🛡️ SECURITY HEADERS

### Helmet Middleware

```javascript
// app.js
const helmet = require('helmet');
app.use(helmet());

// This sets:
// X-Content-Type-Options: nosniff
// X-Frame-Options: DENY
// X-XSS-Protection: 1; mode=block
// Strict-Transport-Security: max-age=31536000
```

### Custom Security Headers

```javascript
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  next();
});
```

---

## 🔍 COMMON SECURITY MISTAKES

### 1. Token in URL

```javascript
// ❌ WRONG
GET /api/students?token=eyJhbGciOi...
// Token visible in browser history, server logs

// ✅ CORRECT
GET /api/students
Authorization: Bearer eyJhbGciOi...
// Token in header, not logged
```

### 2. Storing Sensitive Data in Token

```javascript
// ❌ WRONG
jwt.sign({
  userId: user._id,
  password: user.password,  // NEVER
  creditCard: user.card,    // NEVER
  ssn: user.ssn             // NEVER
}, secret);

// ✅ CORRECT
jwt.sign({
  userId: user._id,
  email: user.email,
  role: user.role
}, secret);
```

### 3. Not Validating Input

```javascript
// ❌ WRONG
const user = await User.findOne({ email: req.body.email });
// What if email is { $ne: null }? (NoSQL injection!)

// ✅ CORRECT
const email = String(req.body.email).toLowerCase().trim();
const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
if (!emailRegex.test(email)) {
  throw new Error('Invalid email');
}
const user = await User.findOne({ email });
```

### 4. Returning Too Much Data

```javascript
// ❌ WRONG
const user = await User.findById(id);
res.json(user);  // Includes password hash, sensitive fields

// ✅ CORRECT
const user = await User.findById(id).select('-password');
res.json(user);  // Password field removed
```

### 5. No Token Expiry

```javascript
// ❌ WRONG
jwt.sign(payload, secret);
// No expiration, token valid forever!

// ✅ CORRECT
jwt.sign(payload, secret, { expiresIn: '2h' });
// Token expires in 2 hours
```

---

## 🔐 FRONTEND SECURITY

### Storing Token Securely

```javascript
// ✅ CORRECT - Use localStorage (for production consider secure storage)
localStorage.setItem('token', response.token);

// Getting token
const token = localStorage.getItem('token');

// Removing token (logout)
localStorage.removeItem('token');
```

### Sending Token in Requests

```javascript
// authService.js
class AuthService {
  static getAuthHeader() {
    const token = localStorage.getItem('token');
    if (!token) return {};
    
    return {
      Authorization: `Bearer ${token}`
    };
  }
}

// Usage in API calls
const response = await fetch('/api/students', {
  method: 'GET',
  headers: {
    'Content-Type': 'application/json',
    ...AuthService.getAuthHeader()
  }
});
```

### Using Axios Interceptor

```javascript
// services/apiClient.js
import axios from 'axios';

const apiClient = axios.create({
  baseURL: process.env.REACT_APP_API_URL
});

// Add token to all requests
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Handle token expiry
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/login';
    }
    return Promise.reject(error);
  }
);

export default apiClient;
```

---

## 📋 SECURITY CHECKLIST

- [ ] JWT expiry set to 2 hours
- [ ] Passwords hashed with bcrypt (salt 10)
- [ ] Token never logged or exposed in URLs
- [ ] All sensitive data removed from responses
- [ ] Multi-tenancy: collegeId checked in all queries
- [ ] Rate limiting on login/payment endpoints
- [ ] CORS configured to allow only frontend domain
- [ ] HTTPS enforced in production
- [ ] Security headers set (Helmet)
- [ ] Input validation on all endpoints
- [ ] RBAC implemented for sensitive operations
- [ ] No hardcoded secrets (use .env)
- [ ] Audit log for sensitive operations
- [ ] Token refresh implemented
- [ ] Logout clears all user session data

---

## 📊 NEXT DOCUMENTS

1. ✅ **01_MERN_STACK_OVERVIEW.md**
2. ✅ **02_DEVELOPMENT_ENVIRONMENT_SETUP.md**
3. ✅ **03_MODULES_ARCHITECTURE.md**
4. ✅ **04_MODULE_INTERCONNECTIONS.md**
5. ✅ **05_DATABASE_DEVELOPER_GUIDE.md**
6. ✅ **06_API_DEVELOPMENT_GUIDE.md**
7. ✅ **07_FRONTEND_DEVELOPMENT_GUIDE.md**
8. ✅ **08_CODE_STANDARDS_CONVENTIONS.md**
9. ✅ **09_AUTHENTICATION_SECURITY_GUIDE.md** (You are here)
10. **10_PAYMENT_PROCESSING_GUIDE.md**
11. **11_TESTING_DEVELOPER_GUIDE.md**
12. **12_DEBUGGING_TROUBLESHOOTING.md**

---

**Previous**: [08_CODE_STANDARDS_CONVENTIONS.md](08_CODE_STANDARDS_CONVENTIONS.md)  
**Next Document**: [10_PAYMENT_PROCESSING_GUIDE.md](10_PAYMENT_PROCESSING_GUIDE.md)

