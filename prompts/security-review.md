# Agentic Security Code Review Prompt

> Copy this prompt into Claude Code while inside the repository you want to review.

## Usage

```bash
# 1. Navigate to your target repository
cd /path/to/your/repo

# 2. Launch Claude Code
claude

# 3. Paste this entire prompt
```

---

## The Prompt

```
You are an agentic security review system composed of multiple specialized sub-agents.

Your task is to perform a complete, high-confidence security review of this repository.

Each sub-agent must perform its role and then pass its output to the next agent.
Be precise, technical, and avoid generic statements.

---

### AGENT 1 — ARCHITECT (Architecture & Threat Model)

Analyze the repository and produce:

- High-level architecture (frontend, backend, APIs, database)
- Authentication and session management mechanisms
- External integrations
- Trust boundaries (where untrusted input crosses into the system)
- Sensitive assets (user data, tokens, admin functionality)

Output:
- Architecture summary
- Trust boundary diagram (text-based)
- Key security assumptions

---

### AGENT 2 — MAPPER (Attack Surface Mapping)

Using the architecture from Agent 1:

Identify ALL entry points where untrusted input enters the system.

For each entry point, provide:
- Endpoint (URL/route)
- HTTP method
- Parameters (query, body, headers, cookies)
- File and function handling the request
- Initial data flow

Also include:
- Hidden inputs (headers, JWTs, file uploads, etc.)

Output as a structured table.

---

### AGENT 3 — ANALYST (Data Flow & Code Path Analysis)

Select the most critical entry points from Agent 2 and trace full execution paths:

- Input → processing → database → response

For each step:
- Identify validation and sanitization (or lack of it)
- Identify authorization checks
- Highlight dangerous functions or insecure patterns
- Call out trust assumptions

Output:
- Step-by-step data flow trace per endpoint
- Security observations at each step

---

### AGENT 4 — HUNTER (Vulnerability Identification)

Using the data flow analysis from Agent 3:

Identify REAL, exploitable vulnerabilities.

For each finding:
- Vulnerability type (OWASP Top 10 category)
- Exact file + function + line number
- Step-by-step exploit scenario
- Root cause
- Preconditions for exploitation

Focus on:
- Injection (SQL, NoSQL, command, template)
- Broken access control (IDOR, privilege escalation)
- Security misconfiguration
- Insecure deserialization
- Business logic flaws

DO NOT include generic or unverified issues.

---

### AGENT 5 — RISK ANALYST (Severity & Impact)

For each vulnerability from Agent 4:

Assign:
- CVSS v3.1 score (with breakdown)
- Attack vector
- Attack complexity
- Privileges required
- User interaction
- Impact (Confidentiality/Integrity/Availability)

Also include:
- Business impact (e.g., account takeover, data exposure, financial loss)
- Likelihood of exploitation

Output as a severity table.

---

### AGENT 6 — FIXER (Remediation)

For each vulnerability:

Provide:
1. Vulnerable code snippet (with file path and line numbers)
2. Explanation of why it is insecure
3. Secure fixed version of the code
4. Additional defensive controls to implement

Ensure fixes align with framework best practices and security standards.

---

### AGENT 7 — VALIDATOR (False Positive & Confidence Check)

Re-evaluate all findings from Agents 4-6:

- Are they truly exploitable given the application context?
- Are there false positives based on incomplete analysis?
- Are there existing mitigations that weren't initially identified?
- What is the confidence level for each finding?

Refine the list to only HIGH-CONFIDENCE vulnerabilities.

Output:
- Validated findings with confidence ratings
- Removed findings with explanations

---

### AGENT 8 — ATTACKER (Adversarial Simulation)

Act as a real attacker targeting this application.

For the top 2-3 highest-severity vulnerabilities:
- Show step-by-step exploitation
- Provide working payload examples
- Describe expected application responses
- Demonstrate potential attack chains (combining vulnerabilities)
- Show post-exploitation possibilities

---

### FINAL OUTPUT FORMAT

Produce a structured markdown report with these sections:

1. **Executive Summary** — Key findings overview for stakeholders
2. **Architecture Overview** — System components and trust boundaries
3. **Attack Surface** — Complete entry point inventory
4. **Data Flow Analysis** — Critical path traces
5. **Confirmed Vulnerabilities** — Validated findings with full details
6. **Risk Ratings** — CVSS scores and business impact
7. **Remediation Plan** — Prioritized fixes with code examples
8. **Exploitation Scenarios** — Proof-of-concept demonstrations

Save the report as `SECURITY_REVIEW_REPORT.md` in the repository root.

---

### IMPORTANT RULES

- Be specific: Include file paths, function names, and line numbers
- Do not hallucinate: Only report vulnerabilities supported by actual code analysis
- Validate everything: Each finding must trace back to specific code paths
- Prioritize depth over breadth: Better to thoroughly analyze 5 critical issues than superficially scan 50
- Think like a senior AppSec engineer: Not a vulnerability scanner, but an experienced security analyst
```

---

## Customization

### For faster reviews (3 agents only)

If you want a quicker analysis, use only the first 3 agents:
- Agent 1: ARCHITECT
- Agent 2: MAPPER  
- Agent 3: ANALYST

Add this line at the end of Agent 3's instructions:
```
Stop after Agent 3 and produce a summary report with architecture, attack surface, and data flow analysis.
```

### For specific vulnerability types

Add focus areas to the HUNTER agent:
```
Focus specifically on:
- Authentication bypass vulnerabilities
- JWT implementation flaws
- API rate limiting issues
```

### For specific frameworks

Add context to help the AI understand the codebase:
```
This is a Django application using:
- Django REST Framework for APIs
- Celery for async tasks
- PostgreSQL database
- Redis for caching
```
