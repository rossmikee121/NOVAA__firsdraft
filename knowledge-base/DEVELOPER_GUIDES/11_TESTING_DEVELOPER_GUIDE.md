# TESTING DEVELOPER GUIDE

**For**: All developers  
**Version**: 1.0  
**Date**: January 20, 2026  
**Quality Target**: 80% code coverage, all critical paths tested

---

## 🧪 OVERVIEW

Testing strategy for NOVAA:

| Level | Tool | Coverage | When |
|-------|------|----------|------|
| **Unit** | Jest | Individual functions | Every commit |
| **Integration** | Jest + Supertest | API endpoints | Every commit |
| **E2E** | Playwright | Full workflows | Before PR merge |
| **Manual** | Browser | UI/UX | Before release |

---

## 📦 SETUP

### Install Dependencies

```bash
# Backend testing
npm install --save-dev jest supertest mongodb-memory-server

# Frontend testing
npm install --save-dev @testing-library/react @testing-library/jest-dom jest

# E2E testing
npm install --save-dev @playwright/test
```

### Jest Configuration

#### Backend (jest.config.js)

```javascript
module.exports = {
  testEnvironment: 'node',
  coverageDirectory: 'coverage',
  collectCoverageFrom: ['src/**/*.js'],
  testMatch: ['**/__tests__/**/*.test.js', '**/*.test.js'],
  setupFilesAfterEnv: ['<rootDir>/src/__tests__/setup.js'],
  testTimeout: 10000
};
```

#### Frontend (jest.config.js)

```javascript
module.exports = {
  testEnvironment: 'jsdom',
  setupFilesAfterEnv: ['<rootDir>/src/setupTests.js'],
  moduleNameMapper: {
    '\\.(css|less)$': 'identity-obj-proxy'
  },
  transform: {
    '^.+\\.(js|jsx)$': 'babel-jest'
  }
};
```

### Setup File (Backend)

```javascript
// src/__tests__/setup.js
beforeAll(async () => {
  // Any global setup
});

afterAll(async () => {
  // Any global cleanup
});

afterEach(async () => {
  // Clear database after each test
});
```

### Setup File (Frontend)

```javascript
// src/setupTests.js
import '@testing-library/jest-dom';

// Mock window.matchMedia
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: jest.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: jest.fn(),
    removeListener: jest.fn(),
    addEventListener: jest.fn(),
    removeEventListener: jest.fn(),
    dispatchEvent: jest.fn(),
  })),
});
```

---

## ✅ UNIT TESTING

### Backend: Utility Functions

```javascript
// utils/gst.js
const GSTCalculator = {
  calculateGST: (feeType, amount) => {
    const rates = { TUITION: 0, LAB: 0.18, SPORTS: 0.18 };
    return Math.round(amount * (rates[feeType] || 0));
  }
};

// src/__tests__/gst.test.js
describe('GSTCalculator', () => {
  describe('calculateGST', () => {
    test('should return 0 for tuition (no tax)', () => {
      const result = GSTCalculator.calculateGST('TUITION', 50000);
      expect(result).toBe(0);
    });

    test('should calculate 18% for lab fees', () => {
      const result = GSTCalculator.calculateGST('LAB', 5000);
      expect(result).toBe(900);
    });

    test('should round to nearest rupee', () => {
      const result = GSTCalculator.calculateGST('LAB', 333);
      expect(result).toBe(60); // 333 * 0.18 = 59.94 → 60
    });

    test('should throw for unknown fee type', () => {
      expect(() => {
        GSTCalculator.calculateGST('UNKNOWN', 1000);
      }).toThrow();
    });
  });
});
```

### Backend: Service Functions

```javascript
// services/authService.js
class AuthService {
  static async loginUser(email, password) {
    if (!email || !password) {
      throw new Error('Email and password required');
    }
    // ... rest of logic
  }
}

// src/__tests__/authService.test.js
describe('AuthService', () => {
  describe('loginUser', () => {
    test('should throw error if email is missing', async () => {
      await expect(
        AuthService.loginUser(undefined, 'password')
      ).rejects.toThrow('Email and password required');
    });

    test('should throw error if password is missing', async () => {
      await expect(
        AuthService.loginUser('test@example.com', undefined)
      ).rejects.toThrow('Email and password required');
    });

    test('should return user object on successful login', async () => {
      // Mock database
      User.findOne = jest.fn().mockResolvedValue({
        _id: '123',
        email: 'test@example.com',
        password: await bcrypt.hash('password', 10)
      });

      const result = await AuthService.loginUser('test@example.com', 'password');
      expect(result).toHaveProperty('token');
      expect(result.user).toHaveProperty('_id');
    });
  });
});
```

### Frontend: Component Unit Tests

```javascript
// src/__tests__/Button.test.js
import { render, screen, fireEvent } from '@testing-library/react';
import Button from '../components/Button';

describe('Button Component', () => {
  test('should render button with label', () => {
    render(<Button label="Click me" onClick={() => {}} />);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  test('should call onClick handler when clicked', () => {
    const mockClick = jest.fn();
    render(<Button label="Click" onClick={mockClick} />);
    
    fireEvent.click(screen.getByText('Click'));
    expect(mockClick).toHaveBeenCalledTimes(1);
  });

  test('should be disabled when disabled prop is true', () => {
    render(<Button label="Click" onClick={() => {}} disabled={true} />);
    expect(screen.getByText('Click')).toBeDisabled();
  });

  test('should have correct variant class', () => {
    render(<Button label="Click" onClick={() => {}} variant="primary" />);
    expect(screen.getByText('Click')).toHaveClass('primary');
  });
});
```

### Frontend: Hook Testing

```javascript
// src/__tests__/useAuth.test.js
import { renderHook, act } from '@testing-library/react';
import { useAuth } from '../hooks/useAuth';
import { AuthProvider } from '../context/AuthContext';

describe('useAuth Hook', () => {
  test('should provide auth context', () => {
    const wrapper = ({ children }) => <AuthProvider>{children}</AuthProvider>;
    const { result } = renderHook(() => useAuth(), { wrapper });

    expect(result.current).toHaveProperty('user');
    expect(result.current).toHaveProperty('login');
    expect(result.current).toHaveProperty('logout');
  });

  test('should login user', async () => {
    const wrapper = ({ children }) => <AuthProvider>{children}</AuthProvider>;
    const { result } = renderHook(() => useAuth(), { wrapper });

    act(() => {
      result.current.login({
        email: 'test@example.com',
        role: 'ADMIN'
      });
    });

    expect(result.current.user).toBeDefined();
    expect(result.current.user.email).toBe('test@example.com');
  });
});
```

---

## 🔗 INTEGRATION TESTING

### Backend: API Endpoint Tests

```javascript
// src/__tests__/auth.integration.test.js
const request = require('supertest');
const app = require('../app');
const User = require('../models/User');

describe('Auth API Endpoints', () => {
  beforeEach(async () => {
    await User.deleteMany({});
  });

  describe('POST /api/auth/signup', () => {
    test('should create new user', async () => {
      const response = await request(app)
        .post('/api/auth/signup')
        .send({
          email: 'newuser@example.com',
          password: 'SecurePassword123',
          collegeId: '507f1f77bcf86cd799439012',
          role: 'STUDENT'
        });

      expect(response.status).toBe(201);
      expect(response.body).toHaveProperty('token');
      expect(response.body.user.email).toBe('newuser@example.com');
    });

    test('should return 400 if email already exists', async () => {
      // Create first user
      await User.create({
        email: 'existing@example.com',
        password: await bcrypt.hash('password', 10),
        collegeId: '507f1f77bcf86cd799439012',
        role: 'STUDENT'
      });

      // Try to create duplicate
      const response = await request(app)
        .post('/api/auth/signup')
        .send({
          email: 'existing@example.com',
          password: 'Password123',
          collegeId: '507f1f77bcf86cd799439012',
          role: 'STUDENT'
        });

      expect(response.status).toBe(409);
      expect(response.body).toHaveProperty('error');
    });

    test('should validate email format', async () => {
      const response = await request(app)
        .post('/api/auth/signup')
        .send({
          email: 'not-an-email',
          password: 'Password123',
          collegeId: '507f1f77bcf86cd799439012',
          role: 'STUDENT'
        });

      expect(response.status).toBe(400);
    });

    test('should validate password length', async () => {
      const response = await request(app)
        .post('/api/auth/signup')
        .send({
          email: 'test@example.com',
          password: 'short',
          collegeId: '507f1f77bcf86cd799439012',
          role: 'STUDENT'
        });

      expect(response.status).toBe(400);
    });
  });

  describe('POST /api/auth/login', () => {
    beforeEach(async () => {
      await User.create({
        email: 'test@example.com',
        password: await bcrypt.hash('SecurePassword123', 10),
        collegeId: '507f1f77bcf86cd799439012',
        role: 'STUDENT'
      });
    });

    test('should login successfully with correct credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'SecurePassword123'
        });

      expect(response.status).toBe(200);
      expect(response.body).toHaveProperty('token');
      expect(response.body.user.email).toBe('test@example.com');
    });

    test('should fail with incorrect password', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'WrongPassword'
        });

      expect(response.status).toBe(401);
    });

    test('should fail with non-existent email', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'nonexistent@example.com',
          password: 'SomePassword'
        });

      expect(response.status).toBe(401);
    });
  });

  describe('Protected Routes', () => {
    let token;

    beforeEach(async () => {
      const user = await User.create({
        email: 'test@example.com',
        password: await bcrypt.hash('SecurePassword123', 10),
        collegeId: '507f1f77bcf86cd799439012',
        role: 'ADMIN'
      });

      token = jwt.sign(
        {
          userId: user._id,
          email: user.email,
          collegeId: user.collegeId,
          role: user.role
        },
        process.env.JWT_SECRET,
        { expiresIn: '2h' }
      );
    });

    test('should access protected route with valid token', async () => {
      const response = await request(app)
        .get('/api/students')
        .set('Authorization', `Bearer ${token}`);

      expect(response.status).toBe(200);
    });

    test('should reject request without token', async () => {
      const response = await request(app)
        .get('/api/students');

      expect(response.status).toBe(401);
    });

    test('should reject request with invalid token', async () => {
      const response = await request(app)
        .get('/api/students')
        .set('Authorization', 'Bearer invalid.token.here');

      expect(response.status).toBe(401);
    });
  });
});
```

### Frontend: Integration Testing

```javascript
// src/__tests__/LoginFlow.integration.test.js
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import LoginPage from '../pages/Auth/LoginPage';
import { AuthProvider } from '../context/AuthContext';

describe('Login Flow Integration', () => {
  test('complete login flow', async () => {
    const user = userEvent.setup();

    render(
      <AuthProvider>
        <LoginPage />
      </AuthProvider>
    );

    // 1. Enter email
    const emailInput = screen.getByPlaceholderText('Email');
    await user.type(emailInput, 'test@example.com');

    // 2. Enter password
    const passwordInput = screen.getByPlaceholderText('Password');
    await user.type(passwordInput, 'SecurePassword123');

    // 3. Click login
    const loginButton = screen.getByRole('button', { name: /login/i });
    fireEvent.click(loginButton);

    // 4. Wait for navigation
    await waitFor(() => {
      expect(screen.queryByText('Login')).not.toBeInTheDocument();
    });
  });

  test('shows error on invalid credentials', async () => {
    const user = userEvent.setup();

    render(
      <AuthProvider>
        <LoginPage />
      </AuthProvider>
    );

    await user.type(screen.getByPlaceholderText('Email'), 'wrong@example.com');
    await user.type(screen.getByPlaceholderText('Password'), 'WrongPassword');
    
    fireEvent.click(screen.getByRole('button', { name: /login/i }));

    await waitFor(() => {
      expect(screen.getByText('Invalid credentials')).toBeInTheDocument();
    });
  });
});
```

---

## 🎭 E2E TESTING

### Setup Playwright

```javascript
// playwright.config.js
module.exports = {
  testDir: './e2e',
  webServer: {
    command: 'npm start',
    port: 3000,
    reuseExistingServer: !process.env.CI
  },
  use: {
    baseURL: 'http://localhost:3000',
    screenshot: 'only-on-failure'
  }
};
```

### E2E Test Example

```javascript
// e2e/auth.spec.js
import { test, expect } from '@playwright/test';

test.describe('Authentication E2E', () => {
  test('complete signup and login flow', async ({ page }) => {
    // 1. Navigate to signup
    await page.goto('/signup');
    
    // 2. Fill signup form
    await page.fill('input[name="email"]', 'newuser@example.com');
    await page.fill('input[name="password"]', 'SecurePassword123');
    await page.fill('input[name="confirmPassword"]', 'SecurePassword123');
    await page.selectOption('select[name="role"]', 'STUDENT');
    
    // 3. Submit
    await page.click('button:has-text("Sign Up")');
    
    // 4. Should redirect to dashboard
    await page.waitForURL('/student/dashboard');
    expect(page.url()).toContain('/student/dashboard');
    
    // 5. Logout
    await page.click('button:has-text("Logout")');
    
    // 6. Should redirect to login
    await page.waitForURL('/login');
    
    // 7. Login
    await page.fill('input[name="email"]', 'newuser@example.com');
    await page.fill('input[name="password"]', 'SecurePassword123');
    await page.click('button:has-text("Login")');
    
    // 8. Should be in dashboard
    await page.waitForURL('/student/dashboard');
    expect(page.url()).toContain('/student/dashboard');
  });

  test('complete payment flow', async ({ page }) => {
    // Login first
    await page.goto('/login');
    await page.fill('input[name="email"]', 'student@example.com');
    await page.fill('input[name="password"]', 'SecurePassword123');
    await page.click('button:has-text("Login")');
    
    // Navigate to fees
    await page.goto('/student/fees');
    
    // Click pay button
    await page.click('button:has-text("Pay Now")');
    
    // Check payment modal appears
    const modal = await page.locator('text=₹50000').isVisible();
    expect(modal).toBeTruthy();
  });
});
```

---

## 📊 MOCKING & FIXTURES

### Mock Database

```javascript
// src/__tests__/mocks/database.js
jest.mock('../models/User');
jest.mock('../models/Student');

const mockUsers = [
  {
    _id: 'user1',
    email: 'admin@college.edu',
    role: 'ADMIN',
    collegeId: 'college1'
  },
  {
    _id: 'user2',
    email: 'student@college.edu',
    role: 'STUDENT',
    collegeId: 'college1'
  }
];

User.findOne = jest.fn((query) => {
  return Promise.resolve(
    mockUsers.find(u => u.email === query.email)
  );
});

User.findById = jest.fn((id) => {
  return Promise.resolve(
    mockUsers.find(u => u._id === id)
  );
});
```

### Mock API Responses

```javascript
// src/__tests__/mocks/api.js
export const mockAuthResponse = {
  token: 'eyJhbGciOiJIUzI1NiIs...',
  user: {
    _id: 'user1',
    email: 'test@example.com',
    role: 'STUDENT',
    collegeId: 'college1'
  }
};

export const mockStudentResponse = {
  data: [
    { _id: 'student1', name: 'John Doe', email: 'john@example.com' },
    { _id: 'student2', name: 'Jane Smith', email: 'jane@example.com' }
  ]
};

// Usage in tests
global.fetch = jest.fn(() =>
  Promise.resolve({
    ok: true,
    json: () => Promise.resolve(mockAuthResponse)
  })
);
```

---

## 📈 COVERAGE REPORTING

### Run Tests with Coverage

```bash
# Backend
npm run test:coverage

# Frontend
npm run test:coverage

# Both
npm run test:coverage:all
```

### Coverage Goals

```
Statements: 80%+
Branches: 75%+
Functions: 80%+
Lines: 80%+
```

### Coverage Script (package.json)

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:integration": "jest --testPathPattern=integration",
    "test:e2e": "playwright test"
  }
}
```

---

## ✅ TESTING CHECKLIST

**For All PRs**:
- [ ] Unit tests added for new functions
- [ ] Integration tests for new endpoints
- [ ] 80%+ code coverage maintained
- [ ] All tests passing locally
- [ ] No console.log() statements
- [ ] Mock data realistic
- [ ] Error cases tested
- [ ] Edge cases considered
- [ ] Async/await errors handled
- [ ] Multi-tenancy tested (collegeId isolation)

**Before Release**:
- [ ] E2E tests passing
- [ ] Manual testing in staging
- [ ] Performance tests run
- [ ] Security tests passed
- [ ] Regression tests executed

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
9. ✅ **09_AUTHENTICATION_SECURITY_GUIDE.md**
10. ✅ **10_PAYMENT_PROCESSING_GUIDE.md**
11. ✅ **11_TESTING_DEVELOPER_GUIDE.md** (You are here)
12. **12_DEBUGGING_TROUBLESHOOTING.md**

---

**Previous**: [10_PAYMENT_PROCESSING_GUIDE.md](10_PAYMENT_PROCESSING_GUIDE.md)  
**Next Document**: [12_DEBUGGING_TROUBLESHOOTING.md](12_DEBUGGING_TROUBLESHOOTING.md)

