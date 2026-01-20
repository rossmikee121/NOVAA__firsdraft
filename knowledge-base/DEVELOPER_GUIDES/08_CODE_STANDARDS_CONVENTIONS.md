# CODE STANDARDS & CONVENTIONS

**For**: All Developers  
**Version**: 1.0  
**Date**: January 20, 2026

---

## 🎯 OVERVIEW

This guide ensures consistency across the NOVAA codebase so:
- All code looks familiar to all developers
- PRs are easier to review
- Bugs are easier to spot
- New developers integrate faster

---

## 📝 JAVASCRIPT/NODE.JS STANDARDS

### File Naming

```
✅ CORRECT:
- services/authService.js
- middleware/authentication.js
- utils/validators.js
- models/User.js
- controllers/admissionController.js

❌ WRONG:
- services/auth_service.js
- middleware/authenticateMiddleware.js
- utils/HelperFunctions.js
- models/user.js
- controllers/AdmissionController.js
```

**Rule**: camelCase for files, PascalCase only for class/model files

### Variable Naming

```javascript
// ✅ CORRECT
const studentId = "123";
const isActive = true;
const userId = req.user._id;
const maxRetries = 3;

// ❌ WRONG
const student_id = "123";
const is_active = true;
const userID = req.user._id;
const MAX_RETRIES = 3;  // Use const without ALL_CAPS
```

**Rule**: camelCase for all variables, UPPERCASE only for .env constants

### Function Naming

```javascript
// ✅ CORRECT
const getUserById = async (userId) => { };
const validateEmail = (email) => { };
const calculateGST = (amount) => { };
const isAuthenticated = (req, res, next) => { };

// ❌ WRONG
const get_user_by_id = async (userId) => { };
const validateEmailAddress = (email) => { };
const calc_gst = (amount) => { };
const checkIfAuthenticated = (req, res, next) => { };
```

**Rule**: camelCase, verb + noun, descriptive name

### Async/Await

```javascript
// ✅ CORRECT
const submitApplication = async (req, res) => {
  try {
    const admission = await admissionService.submit(req.body);
    res.status(201).json({ data: admission });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
};

// ❌ WRONG
const submitApplication = (req, res) => {
  admissionService.submit(req.body).then(admission => {
    res.status(201).json({ data: admission });
  }).catch(error => {
    res.status(500).json({ error: error.message });
  });
};
```

**Rule**: Use async/await instead of .then().catch()

### Comments

```javascript
// ✅ CORRECT
// Calculate total with GST (18% on taxable fees)
const taxAmount = (labFee + sportsFee) * 0.18;

// ❌ WRONG
// tax calc
const taxAmount = (labFee + sportsFee) * 0.18;

// TODO: Implement payment retry logic
const retryPayment = () => { };

// FIXME: This query is slow, needs index
const getStudents = async () => { };
```

**Rule**: Be specific, include TODO/FIXME for incomplete work

### Indentation & Formatting

```javascript
// ✅ CORRECT (2 spaces)
const user = {
  id: "123",
  email: "test@example.com",
  roles: ["ADMIN", "STAFF"]
};

// Use Prettier for consistent formatting
// .prettierrc configuration at project root
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5"
}
```

**Rule**: 2-space indentation, use Prettier, add .prettierrc to project

### Error Handling

```javascript
// ✅ CORRECT
try {
  const result = await database.query(sql);
  return result;
} catch (error) {
  console.error('Database query failed:', error);
  throw new Error('Failed to fetch data');
}

// ❌ WRONG
try {
  const result = await database.query(sql);
  return result;
} catch (error) {
  console.log(error);
  // Silently fail
}
```

**Rule**: Always handle errors, log appropriately, throw meaningful errors

---

## ⚛️ REACT CONVENTIONS

### Component File Structure

```
components/
├── Button.js                 (Functional component)
├── Button.module.css         (Scoped styles)
├── Modal.js
├── Modal.module.css
├── FormInput.js
└── FormInput.module.css
```

### Component Naming & Structure

```javascript
// ✅ CORRECT
import React, { useState } from 'react';
import styles from './ApplicationForm.module.css';

const ApplicationForm = ({ admissionId, onSubmit }) => {
  const [formData, setFormData] = useState({
    courseId: '',
    email: ''
  });
  const [loading, setLoading] = useState(false);

  // Handlers
  const handleChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({ ...prev, [name]: value }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    try {
      await onSubmit(formData);
    } catch (error) {
      console.error('Error:', error);
    } finally {
      setLoading(false);
    }
  };

  // Render
  return (
    <form onSubmit={handleSubmit} className={styles.form}>
      {/* JSX */}
    </form>
  );
};

export default ApplicationForm;

// ❌ WRONG
const applicationForm = () => {  // Should be PascalCase
  // All logic mixed together
  return <form>...</form>;
};
```

**Rule**: PascalCase for components, separate logic/handlers/render sections

### Props & PropTypes

```javascript
// ✅ CORRECT
import PropTypes from 'prop-types';

const Button = ({ label, onClick, disabled, variant }) => {
  return (
    <button onClick={onClick} disabled={disabled} className={variant}>
      {label}
    </button>
  );
};

Button.propTypes = {
  label: PropTypes.string.isRequired,
  onClick: PropTypes.func.isRequired,
  disabled: PropTypes.bool,
  variant: PropTypes.oneOf(['primary', 'secondary', 'danger'])
};

Button.defaultProps = {
  disabled: false,
  variant: 'primary'
};

export default Button;

// ❌ WRONG
const Button = (props) => {
  // Use props.label, props.onClick everywhere
};
```

**Rule**: Destructure props, use PropTypes for validation

### Hooks Usage

```javascript
// ✅ CORRECT
const UserProfile = ({ userId }) => {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    const fetchUser = async () => {
      setLoading(true);
      try {
        const data = await api.getUser(userId);
        setUser(data);
      } catch (error) {
        console.error('Error:', error);
      } finally {
        setLoading(false);
      }
    };
    
    fetchUser();
  }, [userId]); // Dependency array

  if (loading) return <LoadingSpinner />;
  if (!user) return <div>No user found</div>;
  
  return <div>{user.name}</div>;
};

// ❌ WRONG
const UserProfile = ({ userId }) => {
  const [user, setUser] = useState(null);

  useEffect(() => {
    api.getUser(userId).then(setUser);
    // Missing dependency array
  });

  return <div>{user?.name}</div>;
};
```

**Rule**: Always include dependency array, handle loading/error states

### CSS Classes

```javascript
// ✅ CORRECT - Use CSS Modules
import styles from './Card.module.css';

const Card = ({ title }) => (
  <div className={styles.cardContainer}>
    <h2 className={styles.title}>{title}</h2>
  </div>
);

// Or use className library for conditional classes
import classNames from 'classnames';

const Button = ({ variant, disabled }) => (
  <button className={classNames(
    'btn',
    {
      'btn-primary': variant === 'primary',
      'btn-disabled': disabled
    }
  )}>
    Click me
  </button>
);
```

**Rule**: Use CSS Modules for component-scoped styles, avoid global class names

---

## 🗄️ FOLDER STRUCTURE

```
backend/src/
├── modules/                    # Feature modules
│   ├── auth/
│   │   ├── routes.js
│   │   ├── controller.js
│   │   ├── service.js
│   │   ├── model.js
│   │   ├── validation.js
│   │   └── middleware.js
│   ├── admissions/
│   ├── payments/
│   └── [other modules]
│
├── middleware/                 # Shared middleware
│   ├── authentication.js
│   ├── errorHandler.js
│   ├── validation.js
│   └── collegeContext.js
│
├── utils/                      # Helper functions
│   ├── validators.js
│   ├── formatters.js
│   ├── gst.js
│   ├── emailService.js
│   └── constants.js
│
├── config/                     # Configuration
│   ├── database.js
│   ├── env.js
│   └── logger.js
│
├── app.js                      # Express app
└── server.js                   # Entry point

frontend/src/
├── pages/                      # Full page components
│   ├── Auth/
│   ├── Student/
│   └── Admin/
│
├── components/                 # Reusable components
│   ├── Button.js
│   ├── FormInput.js
│   └── Modal.js
│
├── context/                    # State management
│   ├── AuthContext.js
│   └── UIContext.js
│
├── services/                   # API services
│   ├── authService.js
│   ├── apiClient.js
│   └── [module services]
│
├── utils/                      # Helpers
│   ├── validators.js
│   ├── formatters.js
│   └── constants.js
│
├── styles/                     # Global styles
│   ├── index.css
│   ├── variables.css
│   └── utilities.css
│
├── App.js
└── index.js
```

**Rule**: Keep similar files grouped, maintain consistent structure

---

## 📝 GIT CONVENTIONS

### Commit Messages

```
✅ CORRECT:
git commit -m "feat: add admission verification endpoint"
git commit -m "fix: prevent duplicate payment submissions"
git commit -m "docs: update README with setup steps"
git commit -m "test: add unit tests for GST calculation"
git commit -m "refactor: simplify authentication middleware"

❌ WRONG:
git commit -m "update"
git commit -m "fixed bug"
git commit -m "changes to payment module"
git commit -m "WIP"
```

**Format**: `<type>: <description>`
- **feat**: New feature
- **fix**: Bug fix
- **docs**: Documentation
- **test**: Tests
- **refactor**: Code restructuring
- **perf**: Performance improvement

### Branch Naming

```
✅ CORRECT:
git checkout -b feat/admission-verification
git checkout -b fix/duplicate-payment-bug
git checkout -b docs/setup-guide
git checkout -b test/add-gst-tests

❌ WRONG:
git checkout -b feature/fix-stuff
git checkout -b aditya-branch
git checkout -b test123
```

**Format**: `<type>/<description>` (lowercase, hyphens)

### Pull Request Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] New feature
- [ ] Bug fix
- [ ] Documentation
- [ ] Breaking change

## Changes Made
- Change 1
- Change 2

## Testing
How to test these changes

## Screenshots (if applicable)
Add screenshots

## Checklist
- [ ] Code follows style guidelines
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] No new warnings generated
- [ ] All tests passing
```

---

## 🔍 CODE REVIEW CHECKLIST

Before submitting PR, verify:

- [ ] **Naming**: All variables, functions follow conventions
- [ ] **Structure**: Code organized logically
- [ ] **Comments**: Meaningful comments for complex logic
- [ ] **Error Handling**: All errors caught and handled
- [ ] **Testing**: Unit tests included
- [ ] **Security**: No hardcoded secrets, SQL injection safe
- [ ] **Multi-Tenancy**: All queries include collegeId
- [ ] **Performance**: No N+1 queries, proper indexes
- [ ] **Style**: Prettier applied, ESLint passes
- [ ] **Documentation**: Comments/docs updated
- [ ] **No Console Logs**: Remove debug logs
- [ ] **Dependencies**: No unnecessary packages

---

## 📋 LINTING CONFIGURATION

### .eslintrc.js (Backend)

```javascript
module.exports = {
  env: {
    node: true,
    es2021: true
  },
  extends: ['eslint:recommended'],
  parserOptions: {
    ecmaVersion: 12,
    sourceType: 'module'
  },
  rules: {
    'no-unused-vars': 'warn',
    'no-console': 'warn',
    'quotes': ['error', 'single'],
    'semi': ['error', 'always'],
    'indent': ['error', 2],
    'no-var': 'error',
    'prefer-const': 'error'
  }
};
```

### .eslintrc.js (Frontend)

```javascript
module.exports = {
  extends: ['react-app'],
  rules: {
    'no-unused-vars': 'warn',
    'no-console': 'warn',
    'react/prop-types': 'warn',
    'react-hooks/rules-of-hooks': 'error'
  }
};
```

### .prettierrc

```json
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "bracketSpacing": true,
  "arrowParens": "always"
}
```

### package.json Scripts

```json
{
  "scripts": {
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "format": "prettier --write 'src/**/*.{js,css}'",
    "format:check": "prettier --check 'src/**/*.{js,css}'"
  }
}
```

---

## 🧪 TESTING CONVENTIONS

### Test File Naming

```
✅ CORRECT:
src/__tests__/authService.test.js
src/__tests__/ApplicationForm.test.js
src/modules/auth/__tests__/auth.test.js

❌ WRONG:
src/tests/auth.js
src/authServiceTest.js
src/test_auth.js
```

**Rule**: `*.test.js` or `*.spec.js` in `__tests__` folder

### Test Structure

```javascript
describe('AuthService', () => {
  describe('loginUser', () => {
    test('should return token on successful login', async () => {
      // Arrange
      const email = 'test@example.com';
      const password = 'password123';

      // Act
      const result = await authService.loginUser(email, password);

      // Assert
      expect(result.token).toBeDefined();
      expect(result.user.email).toBe(email);
    });

    test('should throw error on invalid credentials', async () => {
      const email = 'test@example.com';
      const password = 'wrong';

      await expect(
        authService.loginUser(email, password)
      ).rejects.toThrow('Invalid credentials');
    });
  });
});
```

**Rule**: Arrange-Act-Assert pattern, describe nested tests

---

## 🎯 ENVIRONMENT VARIABLES

### .env.example (commit this)

```
# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb+srv://user:pass@cluster.mongodb.net/dbname

# Authentication
JWT_SECRET=your_secret_key_here
JWT_EXPIRY=2h

# AWS S3
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret

# Razorpay
RAZORPAY_KEY_ID=your_key
RAZORPAY_KEY_SECRET=your_secret
```

**Rule**: .env in .gitignore, commit .env.example only

---

## 📊 NEXT DOCUMENTS

1. ✅ **01_MERN_STACK_OVERVIEW.md**
2. ✅ **02_DEVELOPMENT_ENVIRONMENT_SETUP.md**
3. ✅ **03_MODULES_ARCHITECTURE.md**
4. ✅ **04_MODULE_INTERCONNECTIONS.md**
5. ✅ **05_DATABASE_DEVELOPER_GUIDE.md**
6. ✅ **06_API_DEVELOPMENT_GUIDE.md**
7. ✅ **07_FRONTEND_DEVELOPMENT_GUIDE.md**
8. ✅ **08_CODE_STANDARDS_CONVENTIONS.md** (You are here)
9. **09_AUTHENTICATION_SECURITY_GUIDE.md**
10. **10_PAYMENT_PROCESSING_GUIDE.md**
11. **11_TESTING_DEVELOPER_GUIDE.md**
12. **12_DEBUGGING_TROUBLESHOOTING.md**

---

**Previous**: [07_FRONTEND_DEVELOPMENT_GUIDE.md](07_FRONTEND_DEVELOPMENT_GUIDE.md)  
**Next Document**: [09_AUTHENTICATION_SECURITY_GUIDE.md](09_AUTHENTICATION_SECURITY_GUIDE.md)

