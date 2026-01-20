# DEVELOPMENT ENVIRONMENT SETUP

**For**: All Developers  
**Version**: 1.0  
**Date**: January 20, 2026  
**OS Support**: macOS, Linux (Ubuntu), Windows (with WSL2)

---

## 📋 PRE-REQUISITES

Before starting, ensure you have:

- [ ] GitHub account created
- [ ] Git installed on your machine
- [ ] Terminal/Command prompt access
- [ ] 5GB free disk space
- [ ] Administrator access to install software
- [ ] MongoDB Atlas account (free tier)
- [ ] AWS account (for S3 testing)
- [ ] Razorpay account (for payment testing)

---

## 🔧 STEP 1: INSTALL NODE.JS

Node.js is the runtime for our backend. We need Node.js 18 or higher.

### On macOS (Using Homebrew)

```bash
# Install Homebrew if you don't have it
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Node.js
brew install node

# Verify installation
node --version        # Should show v18.x or higher
npm --version         # Should show 9.x or higher
```

### On Ubuntu/Linux

```bash
# Update package lists
sudo apt update

# Install Node.js 18
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Verify installation
node --version
npm --version
```

### On Windows (Using WSL2 + Ubuntu)

```bash
# Inside WSL2 terminal
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs

# Verify
node --version
npm --version
```

### On Windows (Using Chocolatey)

```powershell
# Run PowerShell as Administrator
choco install nodejs

# Verify
node --version
npm --version
```

---

## 🔧 STEP 2: INSTALL GIT

Git is for version control and collaborating with team members.

### On macOS

```bash
brew install git

git --version
```

### On Ubuntu/Linux

```bash
sudo apt install -y git

git --version
```

### On Windows (Using Chocolatey)

```powershell
choco install git

git --version
```

### Configure Git

After installation, configure your identity:

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
git config --global core.editor "code"  # Use VS Code as editor

# Verify configuration
git config --global --list
```

---

## 🔧 STEP 3: INSTALL VS CODE

VS Code is our code editor.

### macOS / Linux / Windows

Visit: https://code.visualstudio.com/

Download and install for your OS.

### Essential VS Code Extensions

After installing VS Code, open the Extensions marketplace (Ctrl+Shift+X or Cmd+Shift+X) and install:

1. **ES7+ React/Redux/React-Native snippets** by dsznajder.es7-react-js-snippets
2. **Prettier - Code formatter** by esbenp.prettier-vscode
3. **ESLint** by dbaeumer.vscode-eslint
4. **Thunder Client** (or Postman) by rangav.vscode-thunder-client
5. **MongoDB for VS Code** by MongoDB
6. **REST Client** by humao.rest-client

---

## 🔧 STEP 4: INSTALL MONGODB LOCALLY (Optional)

You can either install MongoDB locally OR use MongoDB Atlas (cloud). For development, we recommend **MongoDB Atlas** (cloud) to match production.

### Option A: MongoDB Atlas (Recommended)

Visit: https://www.mongodb.com/cloud/atlas

1. Create free account
2. Create new project (e.g., "NOVAA-DEV")
3. Create cluster (M0 free tier)
4. Select region: **India (Mumbai)**
5. Create database user:
   - Username: `novaa_dev`
   - Password: `[Generate secure password]`
6. Add IP address: `0.0.0.0/0` (allows local + online connections)
7. Get connection string:
   - Click "Connect"
   - Choose "Connect your application"
   - Copy URL: `mongodb+srv://novaa_dev:<password>@cluster0.xxxxx.mongodb.net/novaa_dev?retryWrites=true&w=majority`

**Save this URL - you'll need it in Step 8.**

### Option B: Local MongoDB Installation

#### On macOS

```bash
brew tap mongodb/brew
brew install mongodb-community

# Start MongoDB
brew services start mongodb-community

# Verify
mongosh
# Should show connected to MongoDB
exit
```

#### On Ubuntu/Linux

```bash
sudo apt update
sudo apt install -y mongodb

# Start MongoDB
sudo systemctl start mongodb
sudo systemctl enable mongodb

# Verify
mongosh
exit
```

---

## 🔧 STEP 5: CLONE GITHUB REPOSITORY

Clone the NOVAA project from GitHub to your local machine.

### Get Repository URL

Ask your tech lead for the GitHub repository URL. It will look like:
```
https://github.com/[org]/novaa.git
```

### Clone

```bash
# Navigate to where you want the project
cd ~/projects    # or any directory

# Clone the repository
git clone https://github.com/[org]/novaa.git

# Navigate into project
cd novaa

# Verify structure
ls -la
```

You should see:
```
frontend/
backend/
docs/
knowledge-base/
.gitignore
README.md
```

---

## 🔧 STEP 6: INSTALL BACKEND DEPENDENCIES

The backend needs npm packages to run.

### Navigate to backend folder

```bash
cd backend

# Check Node modules don't exist yet
ls -la
# You shouldn't see "node_modules" folder
```

### Install dependencies

```bash
# Install all packages from package.json
npm install

# This will create "node_modules" folder and install:
# - express (web server)
# - mongoose (MongoDB driver)
# - dotenv (environment variables)
# - bcrypt (password hashing)
# - jsonwebtoken (JWT auth)
# - And many others...

# Verify
npm list    # Shows installed packages
```

---

## 🔧 STEP 7: INSTALL FRONTEND DEPENDENCIES

The frontend (React) also needs npm packages.

### Navigate to frontend folder

```bash
# From project root
cd frontend

# Install dependencies
npm install

# This will create node_modules and install:
# - react
# - react-router-dom
# - axios (HTTP client)
# - And many others...
```

---

## 🔧 STEP 8: CREATE ENVIRONMENT FILES

Environment files (.env) store sensitive configuration like API keys, database URLs, etc.

### Backend .env file

Create a file named `.env` in the `backend/` folder:

```bash
# backend/.env

# Server
PORT=5000
NODE_ENV=development

# Database
MONGODB_URI=mongodb+srv://novaa_dev:<password>@cluster0.xxxxx.mongodb.net/novaa_dev?retryWrites=true&w=majority
# Replace <password> with actual password from MongoDB Atlas

# Authentication
JWT_SECRET=your_super_secret_jwt_key_change_this_in_production
JWT_EXPIRY=2h

# Email (if using)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your-email@gmail.com
SMTP_PASS=your-app-password

# AWS S3 (Document storage)
AWS_ACCESS_KEY_ID=your_aws_access_key
AWS_SECRET_ACCESS_KEY=your_aws_secret_key
AWS_S3_BUCKET=novaa-documents
AWS_REGION=ap-south-1

# Razorpay (Payments)
RAZORPAY_KEY_ID=your_razorpay_key_id
RAZORPAY_KEY_SECRET=your_razorpay_secret

# Security
BCRYPT_SALT_ROUNDS=10
RATE_LIMIT_MAX_REQUESTS=5
RATE_LIMIT_WINDOW_MS=60000
```

**⚠️ SECURITY**: Never commit `.env` to Git! It's already in `.gitignore`.

### Frontend .env file

Create a file named `.env` in the `frontend/` folder:

```bash
# frontend/.env

# API
REACT_APP_API_URL=http://localhost:5000/api
REACT_APP_API_TIMEOUT=30000

# Environment
REACT_APP_ENV=development
```

---

## ▶️ STEP 9: START THE DEVELOPMENT SERVERS

Now you're ready to run the application!

### Terminal Setup

Open two terminals side by side in VS Code:

1. Terminal 1: Backend server
2. Terminal 2: Frontend server

### Start Backend Server

In Terminal 1:

```bash
# Make sure you're in backend folder
cd backend

# Start the server
npm start

# You should see:
# ✅ Server running on port 5000
# ✅ Connected to MongoDB
# ✅ Ready to accept requests
```

### Start Frontend Server

In Terminal 2:

```bash
# Make sure you're in frontend folder
cd frontend

# Start React development server
npm start

# Browser will open automatically at http://localhost:3000
# You should see the login page
```

---

## ✅ STEP 10: VERIFY SETUP

### Check Backend is Running

Open your browser and go to:
```
http://localhost:5000/api/health
```

You should see:
```json
{
  "status": "ok",
  "message": "NOVAA API Server is running",
  "timestamp": "2026-01-20T10:30:00Z"
}
```

### Check Frontend is Running

Open your browser and go to:
```
http://localhost:3000
```

You should see the NOVAA login page.

### Check Database Connection

In backend Terminal 1, you should see:
```
✅ MongoDB connected successfully
```

---

## 🗂️ PROJECT FOLDER STRUCTURE

After setup, your project looks like:

```
novaa/
├── backend/
│   ├── src/
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   ├── admissions/
│   │   │   ├── payments/
│   │   │   ├── attendance/
│   │   │   ├── colleges/
│   │   │   └── reports/
│   │   ├── middleware/
│   │   ├── utils/
│   │   ├── app.js          # Express app
│   │   └── server.js       # Entry point
│   ├── .env                # Environment (you created)
│   ├── .gitignore
│   ├── package.json
│   └── node_modules/       # Installed packages
│
├── frontend/
│   ├── src/
│   │   ├── pages/
│   │   ├── components/
│   │   ├── context/
│   │   ├── services/
│   │   ├── App.js
│   │   └── index.js
│   ├── .env                # Environment (you created)
│   ├── .gitignore
│   ├── package.json
│   └── node_modules/
│
├── docs/                   # Documentation
├── knowledge-base/         # Developer guides
└── README.md
```

---

## 🧪 FIRST TEST: LOGIN

### Create Test User (Backend)

```bash
# From backend Terminal 1, use MongoDB shell
# Connect to MongoDB
mongosh

# List databases
show dbs

# Use NOVAA database
use novaa_dev

# Create test user
db.users.insertOne({
  _id: ObjectId(),
  collegeId: ObjectId("000000000000000000000001"),
  email: "admin@stxavier.edu",
  password: "$2b$10$...",  // bcrypt hashed - ask backend team
  role: "ADMIN",
  name: "Admin User",
  isActive: true,
  createdAt: new Date()
})

# Exit MongoDB shell
exit
```

### Test Login in Browser

1. Go to http://localhost:3000
2. Enter credentials:
   - Email: `admin@stxavier.edu`
   - Password: `admin123` (or whatever was set)
3. Click "Sign In"
4. Should redirect to dashboard ✅

---

## 🐛 TROUBLESHOOTING

### Issue 1: "Port 5000 Already in Use"

```bash
# Find process using port 5000
lsof -i :5000

# Kill the process (replace PID with actual process ID)
kill -9 <PID>

# Or change port in backend/.env
PORT=5001
```

### Issue 2: "MongoDB Connection Failed"

```bash
# Check if MongoDB is running
# For Atlas: Ensure IP whitelist includes your machine
# For local: mongosh (test connection manually)

# Verify connection string in .env
# Should be: mongodb+srv://user:pass@cluster.mongodb.net/dbname

# Check credentials:
# Username and password are correct in MongoDB Atlas
# User has database access privileges
```

### Issue 3: "npm install fails"

```bash
# Clear npm cache
npm cache clean --force

# Delete node_modules
rm -rf node_modules
rm package-lock.json

# Reinstall
npm install
```

### Issue 4: "Cannot find module 'express'"

```bash
# Make sure you're in correct folder
pwd  # Should end with /backend or /frontend

# Reinstall that folder
npm install
```

### Issue 5: "CORS Error when calling backend API"

In backend `.env`, check:
```
# Should allow frontend URL
CORS_ORIGIN=http://localhost:3000
```

---

## 📚 USEFUL COMMANDS

### Backend Commands

```bash
cd backend

# Start server
npm start

# Start with nodemon (auto-restart on changes)
npm run dev

# Run tests
npm test

# Check code quality
npm run lint

# Format code
npm run format
```

### Frontend Commands

```bash
cd frontend

# Start React app
npm start

# Build for production
npm run build

# Run tests
npm test

# Check code quality
npm run lint
```

### Git Commands

```bash
# Create new branch for your feature
git checkout -b feature/my-feature-name

# Check status
git status

# Add files to staging
git add .

# Commit changes
git commit -m "feat: add login functionality"

# Push to GitHub
git push origin feature/my-feature-name

# Pull latest changes
git pull origin main

# View git log
git log --oneline --all --graph
```

### MongoDB Commands

```bash
# Connect to MongoDB (local)
mongosh

# Or MongoDB Atlas (use connection string from .env)

# List databases
show dbs

# Use database
use novaa_dev

# Show collections
show collections

# Query documents
db.users.find()

# Find specific document
db.users.findOne({ email: "admin@stxavier.edu" })

# Exit
exit
```

---

## 🔐 SECURITY CHECKLIST

After setup, verify security:

- [ ] `.env` file is in `.gitignore` (not committed)
- [ ] `.env` contains placeholder values, not real secrets
- [ ] Real secrets are in team's secret manager (1Password, AWS Secrets Manager)
- [ ] MongoDB IP whitelist configured (not too permissive)
- [ ] JWT_SECRET is long and random (min 32 characters)
- [ ] Password hashing uses bcrypt with salt ≥ 10
- [ ] Rate limiting configured (5 attempts per minute)
- [ ] CORS configured correctly (not `*` in production)

---

## 🎯 NEXT STEPS

After setup is complete:

1. ✅ Verify all 3 developers can run the app
2. ✅ All can access the same MongoDB database
3. ✅ Communicate via Slack/email if setup issues
4. ✅ Create your first feature branch
5. ✅ Read [03_MODULES_ARCHITECTURE.md](03_MODULES_ARCHITECTURE.md) to understand module structure

---

## 📞 GETTING HELP

If you're stuck:

1. Check the **TROUBLESHOOTING** section above
2. Ask on team Slack channel
3. Check GitHub issues
4. Read error messages carefully (they often tell you exactly what's wrong)

---

## 🎯 NEXT DOCUMENTS

1. ✅ **01_MERN_STACK_OVERVIEW.md**
2. ✅ **02_DEVELOPMENT_ENVIRONMENT_SETUP.md** (You are here)
3. ✅ **03_MODULES_ARCHITECTURE.md**
4. ✅ **04_MODULE_INTERCONNECTIONS.md**
5. **05_DATABASE_DEVELOPER_GUIDE.md**
6. **06_API_DEVELOPMENT_GUIDE.md**
7. **07_FRONTEND_DEVELOPMENT_GUIDE.md**
8. **08_CODE_STANDARDS_CONVENTIONS.md**
9. **09_AUTHENTICATION_SECURITY_GUIDE.md**
10. **10_PAYMENT_PROCESSING_GUIDE.md**
11. **11_TESTING_DEVELOPER_GUIDE.md**
12. **12_DEBUGGING_TROUBLESHOOTING.md**

---

**Previous**: [01_MERN_STACK_OVERVIEW.md](01_MERN_STACK_OVERVIEW.md)  
**Next Document**: [03_MODULES_ARCHITECTURE.md](03_MODULES_ARCHITECTURE.md)

