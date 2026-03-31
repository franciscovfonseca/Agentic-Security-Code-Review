# Methodology: Agentic Security Code Review

## Why Single-Pass AI Reviews Fail

When you ask an AI to "review this code for security issues," you get:

- **Generic OWASP checklist** — "Check for SQL injection" without specific findings
- **No code references** — Vague statements without file paths or line numbers
- **Many false positives** — No validation of whether issues are actually exploitable
- **Surface-level analysis** — Missing the deeper architectural vulnerabilities

The AI has no guidance on:
- How deep to analyze
- What to prioritize
- How to structure the output
- How to validate findings

## The Agentic Approach

Instead of one prompt, we use **8 specialized agents** that mirror how professional AppSec teams work:

```
Human Security Team          →    AI Agent Pipeline
─────────────────────────────────────────────────────
Security Architect           →    ARCHITECT agent
Threat Modeler               →    ARCHITECT agent
Penetration Tester           →    MAPPER + HUNTER agents
Code Reviewer                →    ANALYST agent
Risk Analyst                 →    RISK ANALYST agent
Security Engineer            →    FIXER agent
QA / Validator               →    VALIDATOR agent
Red Team                     →    ATTACKER agent
```

## Agent Pipeline Detail

### Phase 1: Discovery (Agents 1-3)

**Goal:** Understand the system before looking for vulnerabilities.

| Agent | Input | Process | Output |
|-------|-------|---------|--------|
| ARCHITECT | Full codebase | Map components, auth, trust boundaries | Architecture diagram, threat model |
| MAPPER | Architecture | Enumerate all entry points | Attack surface inventory |
| ANALYST | Entry points | Trace data flows, identify patterns | Code path analysis |

**Why this matters:** You can't find broken access control if you don't understand what access control exists. You can't find injection if you don't know where input enters the system.

### Phase 2: Analysis (Agents 4-5)

**Goal:** Identify and score real vulnerabilities.

| Agent | Input | Process | Output |
|-------|-------|---------|--------|
| HUNTER | Data flows | Match patterns to vulnerability classes | Specific findings with code refs |
| RISK ANALYST | Findings | Apply CVSS scoring, assess business impact | Prioritized risk table |

**Why this matters:** Not all vulnerabilities are equal. A SQL injection in an admin-only endpoint is different from one in a public search.

### Phase 3: Resolution (Agents 6-7)

**Goal:** Provide actionable fixes and eliminate noise.

| Agent | Input | Process | Output |
|-------|-------|---------|--------|
| FIXER | Scored findings | Generate secure code alternatives | Code-level remediation |
| VALIDATOR | All findings | Re-check for false positives | High-confidence findings only |

**Why this matters:** Security reports with 200 findings are ignored. Validated, prioritized findings with fixes get implemented.

### Phase 4: Verification (Agent 8)

**Goal:** Prove exploitability.

| Agent | Input | Process | Output |
|-------|-------|---------|--------|
| ATTACKER | Top findings | Develop exploitation scenarios | Working attack demonstrations |

**Why this matters:** "This could be exploited" is theory. "Here's the exact payload that extracts the database" is proof.

## Comparison: Generic vs. Agentic

### Finding: SQL Injection

**Generic AI Output:**
> "The application may be vulnerable to SQL injection. Ensure all user input is sanitized and use parameterized queries."

**Agentic Output:**
```
FINDING: SQL Injection in Product Search
─────────────────────────────────────────
File: routes/search.js
Line: 12
Function: searchProducts()
CVSS: 9.8 (Critical)

VULNERABLE CODE:
  models.sequelize.query(
    "SELECT * FROM Products WHERE name LIKE '%" + q + "%'"
  )

ROOT CAUSE:
  User input from req.query.q concatenated directly into SQL string

EXPLOITATION:
  GET /rest/products/search?q=' UNION SELECT password FROM Users--
  → Returns all user password hashes

REMEDIATION:
  models.Product.findAll({
    where: { name: { [Op.like]: `%${q}%` } }
  })
```

## When to Use Each Prompt

| Scenario | Prompt | Time |
|----------|--------|------|
| Quick assessment of new codebase | `quick-review.md` (3 agents) | 2-5 min |
| Comprehensive security review | `security-review.md` (8 agents) | 10-20 min |
| Pre-release security gate | Full 8-agent | 10-20 min |
| PR review for security changes | Custom 2-agent (ANALYST + HUNTER) | 3-5 min |

## Limitations

This methodology works best when:
- ✅ Codebase is readable (not obfuscated)
- ✅ Standard frameworks and patterns are used
- ✅ Code is in a language Claude understands well

It may miss:
- ❌ Vulnerabilities requiring runtime analysis
- ❌ Complex race conditions
- ❌ Issues in compiled dependencies
- ❌ Infrastructure-level security issues

**Always complement AI analysis with:**
- Dynamic testing (DAST)
- Manual penetration testing
- Dependency scanning
- Infrastructure security review
