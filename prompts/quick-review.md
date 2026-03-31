# Quick Security Review Prompt (3 Agents)

> A lightweight version of the full 8-agent review. Use this when you need faster results or want to conserve tokens.

## When to Use

- Initial reconnaissance of a new codebase
- Quick security posture assessment
- When you'll follow up with deeper analysis later
- Token-constrained environments

---

## The Prompt

```
You are an agentic security review system. Perform a focused security assessment using 3 specialized agents.

Each agent builds on the previous agent's output. Be precise and technical.

---

### AGENT 1 — ARCHITECT (Architecture & Threat Model)

Analyze the repository and produce:

- High-level architecture (frontend, backend, APIs, database)
- Authentication and session management mechanisms
- Trust boundaries (where untrusted input enters the system)
- Sensitive assets (user data, tokens, admin functionality)

Output:
- Architecture summary (bullet points)
- Trust boundary list
- Top 5 security-critical components

---

### AGENT 2 — MAPPER (Attack Surface Mapping)

Identify the TOP 10 most security-critical entry points:

For each entry point:
| Endpoint | Method | Parameters | Handler File | Risk Level |
|----------|--------|------------|--------------|------------|

Focus on:
- Authentication endpoints
- Data modification APIs
- File upload/download
- Admin functionality
- Search/query endpoints

---

### AGENT 3 — ANALYST (Vulnerability Quick Scan)

For each high-risk entry point from Agent 2:

Trace the data flow and identify:
- Missing input validation
- Injection risks (SQL, command, template)
- Authorization gaps
- Insecure patterns

For each potential issue found:
- File and line number
- Vulnerability type
- Risk level (Critical/High/Medium/Low)
- One-line remediation suggestion

---

### OUTPUT FORMAT

Produce a concise markdown report:

## Quick Security Assessment

### Architecture
[Agent 1 output]

### Attack Surface (Top 10 Entry Points)
[Agent 2 table]

### Potential Vulnerabilities
[Agent 3 findings]

### Recommended Next Steps
- List 3-5 areas requiring deeper investigation

Save as `QUICK_SECURITY_SCAN.md`
```

---

## Expected Output

This prompt typically produces:
- ~500-1000 lines of analysis
- 5-10 potential findings
- Completion in 2-5 minutes

For comprehensive analysis, use the full 8-agent prompt in `security-review.md`.
