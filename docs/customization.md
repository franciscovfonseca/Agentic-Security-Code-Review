# Customizing the Security Review Prompt

## Framework-Specific Adaptations

### Django / Python

Add this context at the beginning of the prompt:

```
CONTEXT: This is a Django application using:
- Django REST Framework for APIs
- Django ORM (PostgreSQL)
- Celery for async tasks
- Redis for caching/sessions

Focus on:
- Django-specific injection patterns (ORM bypass, raw queries)
- CSRF token handling in DRF
- Permission classes and authentication backends
- Celery task input validation
- Settings.py security (DEBUG, ALLOWED_HOSTS, SECRET_KEY)
```

### Node.js / Express

```
CONTEXT: This is an Express.js application using:
- Express middleware stack
- Sequelize/Mongoose ORM
- JWT authentication
- Multer for file uploads

Focus on:
- Prototype pollution
- NoSQL injection (if MongoDB)
- Express middleware ordering issues
- JWT implementation flaws
- File upload path traversal
```

### React / Next.js

```
CONTEXT: This is a Next.js application with:
- App Router / API Routes
- Server Components and Server Actions
- NextAuth.js for authentication

Focus on:
- Server Action input validation
- API route authorization
- Client-side data exposure (hydration)
- SSRF in server components
- Environment variable exposure
```

### Spring Boot / Java

```
CONTEXT: This is a Spring Boot application using:
- Spring Security
- Spring Data JPA
- Thymeleaf templates

Focus on:
- SpEL injection
- Mass assignment via @ModelAttribute
- Thymeleaf SSTI
- Spring Security configuration gaps
- Actuator endpoint exposure
```

## Vulnerability Focus Areas

### Authentication-Focused Review

Add to HUNTER agent:

```
Focus specifically on authentication vulnerabilities:
- Password reset flow weaknesses
- Session fixation
- JWT algorithm confusion (alg:none, RS256→HS256)
- OAuth/OIDC misconfigurations
- MFA bypass possibilities
- Account enumeration
- Brute force protection gaps
```

### API Security Review

```
Focus specifically on API security:
- Broken Object Level Authorization (BOLA/IDOR)
- Broken Function Level Authorization
- Mass assignment
- Excessive data exposure in responses
- Rate limiting gaps
- API versioning security
- GraphQL-specific issues (introspection, batching, depth)
```

### File Handling Review

```
Focus specifically on file handling:
- Unrestricted file upload
- Path traversal (../../../etc/passwd)
- ZIP slip attacks
- XML External Entity (XXE)
- Server-side request forgery via file URLs
- Insecure file storage locations
- Missing content-type validation
```

## Output Format Variations

### Executive Summary Only

Add to final output section:

```
OUTPUT FORMAT:
Produce a 1-page executive summary with:
- Risk score (Critical/High/Medium/Low)
- Top 5 findings with one-sentence descriptions
- Immediate action items
- Recommended timeline for remediation

Do not include code snippets or technical details.
Save as EXECUTIVE_SUMMARY.md
```

### Developer-Focused Report

```
OUTPUT FORMAT:
Produce a developer-focused report with:
- Each finding as a separate section
- Full code context (10 lines before/after vulnerable code)
- Copy-paste ready secure code replacements
- Links to OWASP/CWE references
- Test cases to verify the fix

Skip executive summary and risk scoring.
Save as DEVELOPER_REMEDIATION_GUIDE.md
```

### Compliance-Focused Report

```
OUTPUT FORMAT:
Map all findings to compliance frameworks:
- PCI-DSS requirements
- OWASP ASVS levels
- NIST 800-53 controls
- SOC 2 criteria

For each finding, include:
- Compliance requirement violated
- Evidence of non-compliance
- Required remediation for compliance
- Verification steps

Save as COMPLIANCE_FINDINGS.md
```

## Reducing Token Usage

### 3-Agent Quick Scan

Use only: ARCHITECT → MAPPER → ANALYST

Remove agents 4-8 and adjust output format.

### Single-File Focus

```
SCOPE LIMITATION:
Only analyze the following files:
- src/auth/*.js
- src/api/users.js
- src/middleware/auth.js

Do not analyze other files. Focus depth over breadth.
```

### Specific Vulnerability Hunt

```
SINGLE FOCUS:
Search only for SQL Injection vulnerabilities.

For each potential SQL injection:
1. Trace input source
2. Identify query construction
3. Check for parameterization
4. Provide exploitation proof

Ignore all other vulnerability types.
```

## Multi-Language Repositories

```
CONTEXT: This is a monorepo with:
- Frontend: React (TypeScript) in /frontend
- Backend: Go in /backend
- Mobile: React Native in /mobile
- Infrastructure: Terraform in /infra

Analysis order:
1. Backend API (highest risk)
2. Frontend (XSS, sensitive data)
3. Infrastructure (misconfigurations)
4. Mobile (data storage, cert pinning)

Produce separate findings sections for each component.
```
