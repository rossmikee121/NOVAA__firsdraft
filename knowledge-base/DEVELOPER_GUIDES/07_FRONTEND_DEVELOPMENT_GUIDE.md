# FRONTEND DEVELOPMENT GUIDE

**For**: Frontend Developers, React Developers  
**Version**: 1.0  
**Date**: January 20, 2026

---

## 📱 OVERVIEW

This guide explains how to build React components and manage state in NOVAA frontend.

You'll learn:
- Component structure
- State management with Context API
- API communication with backend
- Form handling
- Error handling
- Routing

---

## 🏗️ FRONTEND STRUCTURE

```
frontend/src/
├── pages/                    # Full page components
│   ├── Auth/
│   │   ├── Login.js
│   │   ├── Register.js
│   │   └── ForgotPassword.js
│   ├── Student/
│   │   ├── Dashboard.js
│   │   ├── ApplicationForm.js
│   │   ├── ApplicationStatus.js
│   │   ├── PaymentPage.js
│   │   └── AttendanceDashboard.js
│   └── Admin/
│       ├── AdminDashboard.js
│       ├── VerifyDocuments.js
│       ├── FeeCollectionDashboard.js
│       └── AttendanceReports.js
├── components/               # Reusable components
│   ├── Header.js
│   ├── Sidebar.js
│   ├── Modal.js
│   ├── Button.js
│   ├── FormInput.js
│   └── LoadingSpinner.js
├── context/                  # State management
│   ├── AuthContext.js
│   ├── CollegeContext.js
│   └── UIContext.js
├── services/                 # API communication
│   ├── authService.js
│   ├── admissionService.js
│   ├── paymentService.js
│   └── attendanceService.js
├── utils/                    # Helper functions
│   ├── formatters.js
│   ├── validators.js
│   └── constants.js
├── App.js                    # Main component
└── index.js                  # Entry point
```

---

## 🎨 CREATING A REACT COMPONENT

### Component Pattern

```javascript
// frontend/src/pages/Student/ApplicationForm.js

import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '../../context/AuthContext';
import { submitApplication } from '../../services/admissionService';
import FormInput from '../../components/FormInput';
import Button from '../../components/Button';
import LoadingSpinner from '../../components/LoadingSpinner';

const ApplicationForm = () => {
  // ========== STATE ==========
  const navigate = useNavigate();
  const { user } = useAuth();
  
  const [formData, setFormData] = useState({
    courseId: '',
    email: '',
    phone: '',
    documents: []
  });
  
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  const [success, setSuccess] = useState(null);

  // ========== HANDLERS ==========
  const handleInputChange = (e) => {
    const { name, value } = e.target;
    setFormData(prev => ({
      ...prev,
      [name]: value
    }));
    
    // Clear error when user starts typing
    if (error) setError(null);
  };

  const handleFileUpload = (e) => {
    const files = Array.from(e.target.files);
    
    // Validate file size (max 5MB)
    const validFiles = files.filter(file => {
      if (file.size > 5 * 1024 * 1024) {
        setError(`${file.name} is too large (max 5MB)`);
        return false;
      }
      return true;
    });

    setFormData(prev => ({
      ...prev,
      documents: [...prev.documents, ...validFiles]
    }));
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    // Validation
    if (!formData.courseId || !formData.email || formData.documents.length === 0) {
      setError('Please fill all required fields');
      return;
    }

    setLoading(true);
    
    try {
      const response = await submitApplication(formData);
      
      setSuccess('Application submitted successfully! Your ID is: ' + response.data.admissionId);
      
      // Clear form
      setFormData({
        courseId: '',
        email: '',
        phone: '',
        documents: []
      });
      
      // Redirect to status page after 2 seconds
      setTimeout(() => {
        navigate(`/student/application/${response.data.admissionId}`);
      }, 2000);
      
    } catch (err) {
      setError(err.message || 'Failed to submit application');
    } finally {
      setLoading(false);
    }
  };

  // ========== RENDER ==========
  if (!user || user.role !== 'STUDENT') {
    return <div>Access denied</div>;
  }

  return (
    <div className="application-form-container">
      <h1>Submit Application</h1>

      {/* Error Message */}
      {error && (
        <div className="alert alert-error" role="alert">
          {error}
          <button onClick={() => setError(null)}>✕</button>
        </div>
      )}

      {/* Success Message */}
      {success && (
        <div className="alert alert-success" role="alert">
          {success}
        </div>
      )}

      {/* Form */}
      <form onSubmit={handleSubmit}>
        {/* Course Selection */}
        <FormInput
          label="Select Course"
          type="select"
          name="courseId"
          value={formData.courseId}
          onChange={handleInputChange}
          required
        >
          <option value="">Choose a course</option>
          <option value="BSC_CS">B.Sc Computer Science</option>
          <option value="BSC_CHEM">B.Sc Chemistry</option>
          <option value="BA_ECO">B.A Economics</option>
        </FormInput>

        {/* Email */}
        <FormInput
          label="Email"
          type="email"
          name="email"
          value={formData.email}
          onChange={handleInputChange}
          placeholder="your.email@example.com"
          required
        />

        {/* Phone */}
        <FormInput
          label="Phone"
          type="tel"
          name="phone"
          value={formData.phone}
          onChange={handleInputChange}
          placeholder="9876543210"
        />

        {/* Document Upload */}
        <div className="form-group">
          <label>Upload Documents (Aadhaar, Marksheet)</label>
          <input
            type="file"
            multiple
            onChange={handleFileUpload}
            accept=".pdf,.jpg,.jpeg,.png"
          />
          <small>Max 5MB per file. Accepted: PDF, JPG, PNG</small>
          
          {/* Show uploaded files */}
          {formData.documents.length > 0 && (
            <ul className="uploaded-files">
              {formData.documents.map((file, index) => (
                <li key={index}>
                  {file.name} ({(file.size / 1024).toFixed(2)} KB)
                </li>
              ))}
            </ul>
          )}
        </div>

        {/* Submit Button */}
        <Button
          type="submit"
          disabled={loading}
          className="btn-primary"
        >
          {loading ? <LoadingSpinner /> : 'Submit Application'}
        </Button>
      </form>
    </div>
  );
};

export default ApplicationForm;
```

---

## 🔄 STATE MANAGEMENT WITH CONTEXT API

### Creating a Context

```javascript
// frontend/src/context/AuthContext.js

import React, { createContext, useState, useContext, useEffect } from 'react';
import { loginUser } from '../services/authService';

// Create context
const AuthContext = createContext();

// Provider component
export const AuthProvider = ({ children }) => {
  const [user, setUser] = useState(null);
  const [token, setToken] = useState(localStorage.getItem('token'));
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  // Check if user is still logged in (on app load)
  useEffect(() => {
    if (token) {
      // Verify token with backend
      const verifyToken = async () => {
        try {
          const response = await fetch('http://localhost:5000/api/auth/profile', {
            headers: { Authorization: `Bearer ${token}` }
          });
          const data = await response.json();
          setUser(data.user);
        } catch (err) {
          // Token invalid, logout
          logout();
        }
      };
      
      verifyToken();
    }
  }, [token]);

  // Login function
  const login = async (email, password) => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await loginUser(email, password);
      
      setUser(response.user);
      setToken(response.token);
      
      // Store token in localStorage
      localStorage.setItem('token', response.token);
      localStorage.setItem('user', JSON.stringify(response.user));
      
      return response;
    } catch (err) {
      setError(err.message);
      throw err;
    } finally {
      setLoading(false);
    }
  };

  // Logout function
  const logout = () => {
    setUser(null);
    setToken(null);
    localStorage.removeItem('token');
    localStorage.removeItem('user');
  };

  return (
    <AuthContext.Provider value={{
      user,
      token,
      loading,
      error,
      login,
      logout,
      isAuthenticated: !!token
    }}>
      {children}
    </AuthContext.Provider>
  );
};

// Hook to use auth context
export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

### Using Context in Components

```javascript
// frontend/src/components/Header.js

import React from 'react';
import { useAuth } from '../context/AuthContext';

const Header = () => {
  const { user, logout } = useAuth();

  return (
    <header className="header">
      <h1>NOVAA</h1>
      
      <div className="user-info">
        <span>Welcome, {user?.name}</span>
        <span className="role">{user?.role}</span>
        <button onClick={logout}>Logout</button>
      </div>
    </header>
  );
};

export default Header;
```

---

## 🌐 API COMMUNICATION

### Creating API Service

```javascript
// frontend/src/services/admissionService.js

import { useAuth } from '../context/AuthContext';

const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:5000/api';

// Submit application
export const submitApplication = async (formData) => {
  const token = localStorage.getItem('token');
  const collegeCode = localStorage.getItem('collegeCode');

  const formDataWithFiles = new FormData();
  formDataWithFiles.append('courseId', formData.courseId);
  formDataWithFiles.append('email', formData.email);
  
  // Add files
  formData.documents.forEach((file, index) => {
    formDataWithFiles.append(`documents`, file);
  });

  try {
    const response = await fetch(`${API_URL}/admissions/apply`, {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${token}`,
        'x-college-code': collegeCode
      },
      body: formDataWithFiles
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.message || 'Failed to submit application');
    }

    return await response.json();
  } catch (error) {
    console.error('Error:', error);
    throw error;
  }
};

// Get application status
export const getApplicationStatus = async (admissionId) => {
  const token = localStorage.getItem('token');
  const collegeCode = localStorage.getItem('collegeCode');

  try {
    const response = await fetch(`${API_URL}/admissions/${admissionId}`, {
      headers: {
        'Authorization': `Bearer ${token}`,
        'x-college-code': collegeCode
      }
    });

    if (!response.ok) {
      throw new Error('Failed to fetch application status');
    }

    return await response.json();
  } catch (error) {
    console.error('Error:', error);
    throw error;
  }
};

// Get list of applications (for admin)
export const listApplications = async (filters = {}) => {
  const token = localStorage.getItem('token');
  const collegeCode = localStorage.getItem('collegeCode');

  const params = new URLSearchParams(filters);
  const url = `${API_URL}/admissions?${params}`;

  try {
    const response = await fetch(url, {
      headers: {
        'Authorization': `Bearer ${token}`,
        'x-college-code': collegeCode
      }
    });

    if (!response.ok) {
      throw new Error('Failed to fetch applications');
    }

    return await response.json();
  } catch (error) {
    console.error('Error:', error);
    throw error;
  }
};
```

### Creating Reusable HTTP Client

```javascript
// frontend/src/utils/httpClient.js

class HttpClient {
  constructor(baseURL) {
    this.baseURL = baseURL;
  }

  // GET request
  async get(endpoint, headers = {}) {
    return this.request('GET', endpoint, null, headers);
  }

  // POST request
  async post(endpoint, data = {}, headers = {}) {
    return this.request('POST', endpoint, data, headers);
  }

  // PUT request
  async put(endpoint, data = {}, headers = {}) {
    return this.request('PUT', endpoint, data, headers);
  }

  // DELETE request
  async delete(endpoint, headers = {}) {
    return this.request('DELETE', endpoint, null, headers);
  }

  // Main request method
  async request(method, endpoint, data, customHeaders) {
    const token = localStorage.getItem('token');
    const collegeCode = localStorage.getItem('collegeCode');

    const headers = {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
      'x-college-code': collegeCode,
      ...customHeaders
    };

    const config = {
      method,
      headers
    };

    if (data && (method === 'POST' || method === 'PUT')) {
      config.body = JSON.stringify(data);
    }

    try {
      const response = await fetch(`${this.baseURL}${endpoint}`, config);

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.message || `HTTP ${response.status}`);
      }

      return await response.json();
    } catch (error) {
      throw error;
    }
  }
}

export default new HttpClient(
  process.env.REACT_APP_API_URL || 'http://localhost:5000/api'
);
```

---

## 🚀 ROUTING

### Main App Component

```javascript
// frontend/src/App.js

import React from 'react';
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from './context/AuthContext';

// Pages
import Login from './pages/Auth/Login';
import Register from './pages/Auth/Register';
import Dashboard from './pages/Student/Dashboard';
import ApplicationForm from './pages/Student/ApplicationForm';
import ApplicationStatus from './pages/Student/ApplicationStatus';

// Layout
import Header from './components/Header';
import Sidebar from './components/Sidebar';

// Protected Route Component
const ProtectedRoute = ({ children, requiredRole }) => {
  const { user, isAuthenticated } = useAuth();
  
  if (!isAuthenticated) {
    return <Navigate to="/login" />;
  }
  
  if (requiredRole && user?.role !== requiredRole) {
    return <Navigate to="/unauthorized" />;
  }
  
  return children;
};

function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <Routes>
          {/* Public Routes */}
          <Route path="/login" element={<Login />} />
          <Route path="/register" element={<Register />} />

          {/* Student Routes */}
          <Route
            path="/student/*"
            element={
              <ProtectedRoute requiredRole="STUDENT">
                <div className="app-layout">
                  <Header />
                  <div className="main-content">
                    <Sidebar />
                    <Routes>
                      <Route path="/dashboard" element={<Dashboard />} />
                      <Route path="/apply" element={<ApplicationForm />} />
                      <Route path="/application/:id" element={<ApplicationStatus />} />
                    </Routes>
                  </div>
                </div>
              </ProtectedRoute>
            }
          />

          {/* Redirect root */}
          <Route path="/" element={<Navigate to="/login" />} />
        </Routes>
      </AuthProvider>
    </BrowserRouter>
  );
}

export default App;
```

---

## 🎯 FORM HANDLING

### Form Component with Validation

```javascript
// frontend/src/components/FormInput.js

import React from 'react';

const FormInput = ({
  label,
  type = 'text',
  name,
  value,
  onChange,
  error,
  required = false,
  placeholder,
  children,
  pattern,
  minLength,
  maxLength
}) => {
  return (
    <div className="form-group">
      {label && (
        <label htmlFor={name}>
          {label}
          {required && <span className="required">*</span>}
        </label>
      )}

      {type === 'select' ? (
        <select
          id={name}
          name={name}
          value={value}
          onChange={onChange}
          required={required}
          className={error ? 'input-error' : ''}
        >
          {children}
        </select>
      ) : type === 'textarea' ? (
        <textarea
          id={name}
          name={name}
          value={value}
          onChange={onChange}
          placeholder={placeholder}
          required={required}
          minLength={minLength}
          maxLength={maxLength}
          className={error ? 'input-error' : ''}
        />
      ) : (
        <input
          id={name}
          type={type}
          name={name}
          value={value}
          onChange={onChange}
          placeholder={placeholder}
          required={required}
          pattern={pattern}
          minLength={minLength}
          maxLength={maxLength}
          className={error ? 'input-error' : ''}
        />
      )}

      {error && <span className="error-message">{error}</span>}
    </div>
  );
};

export default FormInput;
```

### Form Validation

```javascript
// frontend/src/utils/validators.js

export const validateEmail = (email) => {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
};

export const validatePhone = (phone) => {
  return /^[0-9]{10}$/.test(phone);
};

export const validatePassword = (password) => {
  return password.length >= 6;
};

export const validateForm = (formData, requiredFields) => {
  const errors = {};

  requiredFields.forEach(field => {
    if (!formData[field] || formData[field].trim() === '') {
      errors[field] = `${field} is required`;
    }
  });

  return {
    isValid: Object.keys(errors).length === 0,
    errors
  };
};
```

---

## 🎨 STYLING

### CSS Structure

```css
/* frontend/src/styles/index.css */

:root {
  --primary-color: #3b82f6;
  --danger-color: #ef4444;
  --success-color: #10b981;
  --warning-color: #f59e0b;
  --light-gray: #f3f4f6;
  --border-color: #e5e7eb;
}

/* Components */
.btn-primary {
  background-color: var(--primary-color);
  color: white;
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-size: 16px;
}

.btn-primary:hover {
  opacity: 0.9;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  margin-bottom: 8px;
  font-weight: 500;
}

.form-group input,
.form-group textarea,
.form-group select {
  width: 100%;
  padding: 10px;
  border: 1px solid var(--border-color);
  border-radius: 4px;
  font-size: 16px;
}

.input-error {
  border-color: var(--danger-color) !important;
}

.error-message {
  color: var(--danger-color);
  font-size: 14px;
  margin-top: 4px;
  display: block;
}

.alert {
  padding: 12px 16px;
  border-radius: 4px;
  margin-bottom: 20px;
}

.alert-error {
  background-color: #fee;
  color: var(--danger-color);
  border: 1px solid var(--danger-color);
}

.alert-success {
  background-color: #efe;
  color: var(--success-color);
  border: 1px solid var(--success-color);
}
```

---

## 🧪 TESTING COMPONENTS

### Testing with Jest + React Testing Library

```javascript
// frontend/src/__tests__/ApplicationForm.test.js

import React from 'react';
import { render, screen, fireEvent } from '@testing-library/react';
import ApplicationForm from '../pages/Student/ApplicationForm';
import * as admissionService from '../services/admissionService';

// Mock the service
jest.mock('../services/admissionService');

describe('ApplicationForm', () => {
  test('renders form with all fields', () => {
    render(<ApplicationForm />);
    
    expect(screen.getByText('Submit Application')).toBeInTheDocument();
    expect(screen.getByLabelText('Select Course')).toBeInTheDocument();
    expect(screen.getByLabelText('Email')).toBeInTheDocument();
  });

  test('shows error when required fields are empty', async () => {
    render(<ApplicationForm />);
    
    const submitButton = screen.getByText('Submit Application');
    fireEvent.click(submitButton);
    
    expect(screen.getByText(/Please fill all required fields/)).toBeInTheDocument();
  });

  test('submits form with valid data', async () => {
    admissionService.submitApplication.mockResolvedValue({
      data: { admissionId: '123' }
    });

    render(<ApplicationForm />);
    
    // Fill form
    fireEvent.change(screen.getByLabelText('Select Course'), { target: { value: 'BSC_CS' } });
    fireEvent.change(screen.getByLabelText('Email'), { target: { value: 'test@example.com' } });
    
    // Submit
    fireEvent.click(screen.getByText('Submit Application'));
    
    // Wait for success message
    expect(await screen.findByText(/Application submitted successfully/)).toBeInTheDocument();
  });
});
```

---

## 📊 NEXT DOCUMENTS

1. ✅ **01_MERN_STACK_OVERVIEW.md**
2. ✅ **02_DEVELOPMENT_ENVIRONMENT_SETUP.md**
3. ✅ **03_MODULES_ARCHITECTURE.md**
4. ✅ **04_MODULE_INTERCONNECTIONS.md**
5. ✅ **05_DATABASE_DEVELOPER_GUIDE.md**
6. ✅ **06_API_DEVELOPMENT_GUIDE.md**
7. ✅ **07_FRONTEND_DEVELOPMENT_GUIDE.md** (You are here)
8. **08_CODE_STANDARDS_CONVENTIONS.md**
9. **09_AUTHENTICATION_SECURITY_GUIDE.md**
10. **10_PAYMENT_PROCESSING_GUIDE.md**
11. **11_TESTING_DEVELOPER_GUIDE.md**
12. **12_DEBUGGING_TROUBLESHOOTING.md**

---

**Previous**: [06_API_DEVELOPMENT_GUIDE.md](06_API_DEVELOPMENT_GUIDE.md)  
**Next Document**: [08_CODE_STANDARDS_CONVENTIONS.md](08_CODE_STANDARDS_CONVENTIONS.md)

