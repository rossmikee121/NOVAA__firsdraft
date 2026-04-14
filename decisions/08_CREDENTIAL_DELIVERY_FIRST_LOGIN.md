# Decision 08 — Credential Delivery & First Login Flow

**Document Version**: 1.0
**Date**: 2026
**Status**: PENDING DECISION
**Module**: Authentication, Onboarding
**Prepared By**: Technical Team

---

## 1. Context

When a new college is onboarded onto the platform, a College Admin account is created with a temporary password. Currently, the system attempts to send credentials via email. However, email delivery is not always reliable — college IT policies may block external senders, SMTP configuration may fail, or the college may not have provided a working email address at onboarding time.

Additionally, the current system has no mechanism to force a password change on first login. A College Admin could continue using the temporary password indefinitely, which is a security risk.

The onboarding process also lacks a guided setup flow. A new College Admin who logs in for the first time sees an empty dashboard with no guidance on what to do next (add departments, courses, fee structures, etc.).

This decision covers three related concerns:
1. How credentials are delivered to new College Admins
2. How the system enforces a password change on first login
3. What the onboarding experience looks like after first login

---

## 2. Current Implementation

### 2.1 College Admin Creation

When SUPER_ADMIN creates a new college, a `User` account is created with:
- `role: "COLLEGE_ADMIN"`
- `password: tempPassword` (format: `"TempPass" + random 8 chars`)
- `isActive: true`

The temp password is hashed and stored. It is not displayed anywhere in the admin UI. The only delivery mechanism is email.

### 2.2 Student Credential Creation

When a student is approved (current flow — see Decision 01), a `User` account is created with the same temp password pattern. Credentials are sent via `sendAdmissionApprovalEmail`.

### 2.3 Gaps

- Temp password is never shown in the admin UI — if email fails, there is no fallback
- No `mustChangePassword` flag on the User model
- No first-login detection
- No onboarding wizard
- Temp passwords do not expire

---

## 3. Credential Delivery Options

### Option A — Email Only (Current)

Credentials are sent via email at account creation. No fallback.

| Aspect | Detail |
|--------|--------|
| Reliability | Low — email delivery not guaranteed |
| Security | Medium — credentials in email |
| Fallback | None |
| Recommended | No |

---

### Option B — Show Once in Admin UI + Email (Recommended)

After creating a college, the SUPER_ADMIN UI displays the temp credentials once (similar to AWS IAM key creation). The credentials are also sent via email as a backup. After the page is closed or refreshed, the temp password is no longer visible.

| Aspect | Detail |
|--------|--------|
| Reliability | High — admin always has credentials regardless of email |
| Security | High — shown once, then gone |
| Fallback | Admin can reset password via "Reset Credentials" button |
| Recommended | Yes |

---

### Option C — Manual Entry by Admin

SUPER_ADMIN sets the initial password manually when creating the college. No auto-generation.

| Aspect | Detail |
|--------|--------|
| Reliability | High |
| Security | Depends on admin choosing a strong password |
| Recommended | No — increases admin workload, risk of weak passwords |

---

## 4. First Login Password Change

### 4.1 Proposed User Model Addition

```
User {
  mustChangePassword: { type: Boolean, default: true }
}
```

Set to `true` on account creation. Set to `false` after the user successfully changes their password.

### 4.2 Enforcement Options

**Option A — Middleware Enforcement (Recommended)**

A middleware checks `req.user.mustChangePassword` on every authenticated request. If `true`, all routes except `/api/auth/change-password` return:

```json
{
  "status": "error",
  "code": "PASSWORD_CHANGE_REQUIRED",
  "message": "Please change your password to continue.",
  "redirectUrl": "/auth/change-password"
}
```

The frontend intercepts this response and redirects to the change password screen.

**Option B — Frontend Only**

After login, if `mustChangePassword` is `true` in the response, the frontend redirects to the change password screen. No backend enforcement.

| Aspect | Option A | Option B |
|--------|----------|----------|
| Security | Strong — cannot bypass via API | Weak — can bypass by calling API directly |
| Complexity | Medium | Low |
| Recommended | Yes | No |

---

## 5. Temp Password Expiry

Temp passwords should expire if not used within a defined period. This prevents orphaned accounts with known credentials.

### 5.1 Proposed User Model Addition

```
User {
  tempPasswordExpiresAt: Date   // Set to createdAt + 48 hours
}
```

### 5.2 Expiry Check

On login, if `mustChangePassword === true` and `tempPasswordExpiresAt < now`:
- Block login
- Return error: "Temporary credentials have expired. Please contact your administrator to reset your password."
- Admin must use "Reset Credentials" to generate a new temp password

---

## 6. Onboarding Wizard

After a College Admin logs in and changes their password for the first time, they should be guided through the minimum setup required before the system is usable.

### 6.1 Trigger Condition

Show onboarding wizard if:
- `mustChangePassword` was just set to `false` (first password change), OR
- College has zero departments

### 6.2 Wizard Steps

```
Step 1: Add Your First Department
  → Department name, code, type (Academic/Administrative)
  → "Skip" option available

Step 2: Add Your First Course
  → Course name, program level, duration, year labels
  → Linked to department from Step 1
  → "Skip" option available

Step 3: Set Up Fee Structure
  → Select course, category, total fee, installments
  → "Skip" option available

Step 4: Done
  → Summary of what was set up
  → Links to: Add Teachers, Add Students, Configure Documents
```

### 6.3 Wizard Storage

Track wizard completion in the User model:

```
User {
  onboardingCompleted: { type: Boolean, default: false },
  onboardingStep: { type: Number, default: 0 }
}
```

---

## 7. "Reset Credentials" Feature

SUPER_ADMIN should be able to reset a College Admin's credentials at any time (e.g., if the admin is locked out or temp password expired).

### 7.1 Reset Flow

1. SUPER_ADMIN clicks "Reset Credentials" for a college
2. System generates new temp password
3. Sets `mustChangePassword: true`
4. Sets `tempPasswordExpiresAt: now + 48 hours`
5. Displays new temp password once in admin UI
6. Sends email notification (if email is configured)

---

## 8. Applies To All Account Types

This credential delivery and first-login flow applies to:

| Account Type | Created By | Delivery Method |
|-------------|-----------|----------------|
| College Admin | SUPER_ADMIN | Show once in admin UI + email |
| Teacher | College Admin | Email (teacher sets own password via link) |
| Student | System (on admission confirmation — see Decision 01) | Email |
| Accountant / Admission Officer / etc. | College Admin | Email |

---

## 9. Impact on Existing Code

| File | Change Required |
|------|----------------|
| `backend/src/models/user.model.js` | Add `mustChangePassword`, `tempPasswordExpiresAt`, `onboardingCompleted`, `onboardingStep` |
| `backend/src/middlewares/auth.middleware.js` | Add check for `mustChangePassword` — return `PASSWORD_CHANGE_REQUIRED` if true |
| `backend/src/controllers/auth.controller.js` | Add `changePassword` endpoint that sets `mustChangePassword: false` |
| `backend/src/controllers/master.controller.js` | Update `createCollege` to display temp credentials in response (shown once) |
| `backend/src/controllers/master.controller.js` | Add `resetCredentials` endpoint |
| `frontend` — CreateNewCollege.jsx | Show temp credentials in a modal after college creation |
| `frontend` — Login.jsx | Handle `PASSWORD_CHANGE_REQUIRED` response, redirect to change password |
| `frontend` — New: ChangePassword.jsx | Dedicated change password screen for first login |
| `frontend` — New: OnboardingWizard.jsx | 3-step wizard shown after first password change |

---

## 10. Open Questions — Requires Decision

| # | Question | Options | Impact |
|---|----------|---------|--------|
| Q1 | Should the sales representative have a login to view temp credentials, or does the LemmeCode ops team handle all college creation? | Sales login / Ops team only | Affects who can see credentials in admin UI |
| Q2 | What is the temp password expiry period — 24 hours or 48 hours? | 24h / 48h | Affects `tempPasswordExpiresAt` calculation |
| Q3 | Should the onboarding wizard be mandatory (cannot skip) or optional? | Mandatory / Optional | Affects wizard UX |
| Q4 | Should teacher accounts also have `mustChangePassword` enforced, or only College Admin accounts? | All accounts / Admin only | Affects scope of middleware |
| Q5 | Should the system send an SMS with credentials as a fallback when email fails? | Yes / No | Affects SMS integration priority |

---

## 11. Sign-Off Required

| Role | Name | Decision | Date |
|------|------|----------|------|
| Product Owner | | ☐ Approved ☐ Changes Needed | |
| Technical Lead | | ☐ Approved ☐ Changes Needed | |
| Development Team | | ☐ Acknowledged | |

---

**Document Owner**: Technical Team
**Next Review**: Before sprint planning
