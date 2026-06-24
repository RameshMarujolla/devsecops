# Threat Modeling Guide

> **"The best time to fix a security flaw is before it's code."**

Threat modeling is the practice of **systematically identifying, analyzing, and mitigating threats** to a software system **during the design phase** — before a single line of code is written. It shifts security all the way to the left edge of the SDLC: the architecture and requirements stage.

This guide covers how to run threat modeling sessions, frameworks you can use, and a complete walkthrough using a real-world example.

---

## Table of Contents

- [Why Threat Modeling Matters](#why-threat-modeling-matters)
- [The 4 Core Questions](#the-4-core-questions)
- [When to Do Threat Modeling](#when-to-do-threat-modeling)
- [Threat Modeling Frameworks](#threat-modeling-frameworks)
- [Complete Walkthrough: E-Commerce Web App](#complete-walkthrough-e-commerce-web-app)
- [STRIDE in Practice](#stride-in-practice)
- [Threat Modeling Templates](#threat-modeling-templates)
- [Common Anti-Patterns](#common-anti-patterns)
- [Tools for Threat Modeling](#tools-for-threat-modeling)
- [Integrating Threat Modeling into DevSecOps](#integrating-threat-modeling-into-devsecops)
- [Summary](#summary)

---

## Why Threat Modeling Matters

### Cost at Each SDLC Phase

| Phase | Relative Cost to Fix | Threat Modeling Impact |
|---|---|---|
| Design | **1x** | ✅ Catches threats here |
| Code | 5x | 📋 Architecture already hardened |
| Testing | 10x | 🔒 Most high-risk paths addressed |
| Staging | 15x | 🔒 Minor residual risk only |
| Production | **100x+** | 💰 Crisis avoided |

### What Threat Modeling Prevents

- Security debt accumulating in the foundation
- Expensive redesigns later in the project
- Compliance gaps discovered at audit time
- Attack paths the team never considered
- "I didn't think of that" moments after a breach

---

## The 4 Core Questions

Every threat model answers these four questions:

### 1. What are we working on?
Identify the scope. Draw a **Data Flow Diagram (DFD)** or architecture diagram. Know your system's components, data stores, external entities, and **trust boundaries**.

### 2. What can go wrong?
Apply a threat framework (STRIDE, PASTA, etc.) to systematically identify threats at each trust boundary.

### 3. What are we going to do about it?
For each threat, define a **mitigation** — a control that reduces the threat to an acceptable level.

### 4. Did we do a good enough job?
Review, validate, and iterate. Check against frameworks like OWASP Top 10 and validate with penetration testing.

---

## When to Do Threat Modeling

Threat modeling is not a one-time event. It happens at **specific trigger points** throughout the SDLC. Here are the real-world moments when teams perform it.

### 1. New Project / Architecture (Design Phase)

**When:** Before the first user story is written.  
**Who:** Architects, security champions, product owner, lead developers.  
**Duration:** 2–4 hours for initial model.

**Example:**
Your team is building a new payment microservice. On day 1 of sprint 0, you gather around a whiteboard, draw the data flow (user → API → payment gateway → ledger DB), and run STRIDE. You discover that if the ledger DB is compromised, an attacker could reverse transactions. So you design **immutable append-only ledgers** and **event sourcing** from the start — not as a patch later.

> *This is the highest-ROI threat modeling session. A 3-hour session here prevents 3 weeks of rework later.*

### 2. New Feature with Data Flow Changes

**When:** Sprint planning, before implementation begins.  
**Who:** Feature team + security champion.  
**Duration:** 20–30 minutes (lightweight).

**Example:**
You're adding a "file upload" feature to your existing app. The team draws: `Browser → S3 → Lambda → Processing Queue`. The security champion asks: *"What if someone uploads a malicious file that exploits the Lambda processor?"* You add:
- File type validation (not just extension)
- Virus scanning via ClamAV
- Lambda timeout limits (prevent infinite loops)
- S3 pre-signed URLs with 15-min expiry

> *This is "continuous threat modeling" — small, frequent sessions rather than exhaustive, one-time audits.*

### 3. New Integration (APIs, OAuth, Payment Gateway)

**When:** Before any third-party SDK or API is added.  
**Who:** Integration lead + security team.

**Example:**
Your team wants to add **Stripe Connect** for marketplace payouts. You threat model the OAuth flow:
- Where does the Stripe secret key live? *(Answer: not in env vars → use AWS Secrets Manager with rotation)*
- What if Stripe's callback URL is tampered with? *(Answer: verify `state` parameter, whitelist redirect URIs)*
- How do we detect fraudulent connected accounts? *(Answer: webhook validation + anomaly alerting)*

> *Third-party integrations are where most breach stories start. The threat model forces you to understand trust boundaries you don't control.*

### 4. Technology Change (New Framework, Database, Cloud Service)

**When:** Before adopting a new stack component.

**Example:**
Team decides to migrate from PostgreSQL to MongoDB. In the threat model session:
- PostgreSQL had row-level security policies — does MongoDB have equivalent?
- MongoDB's default bind without auth — how is this mitigated?
- Backup strategy: MongoDB snapshots vs. logical backups — encryption at rest?

The model reveals MongoDB's schema-less nature introduces **NoSQL injection** risks the ORM doesn't prevent. The team adds a **validation layer** before queries.

### 5. Incident Post-Mortem

**When:** Within 48 hours of a security incident.

**Example:**
An attacker exploited an **IDOR** in `/api/orders/{id}` to view other customers' orders. The post-mortem includes updating the threat model:

```markdown
## Incident-Driven Threat Model Update
**Date:** 2026-06-20
**Incident:** IDOR-2026 (customer data exposure)

### Finding
Order API lacked authorization checks. Attacker iterated order IDs.

### Model Update
- **Trust Boundary:** API → Database
- **New Threat (Elevation):** User A → User B's orders via ID enumeration
- **Existing Control:** JWT authentication ✅
- **Missing Control:** Resource-level authorization (does user own this order?)
- **Mitigation Added:** `require_ownership(order_id, user_id)` middleware
- **Test Added:** Automated IDOR fuzzing in CI/CD
```

> *The threat model becomes a living document. Every incident teaches the model — and the model teaches future designs.*

### 6. Quarterly Review / Architecture Drift

**When:** Every quarter or when architecture significantly drifts.

**Example:**
Over 3 months, a "simple" monolith evolved into 6 microservices. The original threat model assumed a single trust boundary. Now there are:
- Service mesh (Istio)
- Inter-service mTLS
- New external API gateway
- Kafka event bus

The quarterly session redraws the DFD and discovers:
- Kafka topics have no ACLs (any service can read payment events)
- mTLS is enabled but certificate rotation is manual (expiration risk)

> *Architecture drift silently invalidates old threat models. Quarterly reviews catch the mismatch.*

---

### Quick Trigger Reference

| Trigger | Why | Priority |
|---|---|---|
| **New project / architecture** | Establish security foundation | 🔴 High |
| **New feature with data flow changes** | Catch new attack paths | 🔴 High |
| **New integration** (API, OAuth, payment) | Third-party risk assessment | 🔴 High |
| **Technology change** (new database, framework, cloud service) | New threat surface | 🟡 Medium |
| **Incident post-mortem** | Update model based on real attack | 🔴 High |
| **Quarterly review** | Validate assumptions remain valid | 🟡 Medium |

---

### Real-World Sprint Schedule

Here's how a mature team integrates threat modeling into their rhythm:

| Day | Activity | Threat Modeling Moment |
|-----|----------|------------------------|
| **Monday** | Sprint Planning | 30-min mini threat model for new features |
| **Wednesday** | Mid-sprint review | Security champion reviews high-risk changes |
| **Friday** | Sprint Demo | Update threat model with architecture changes |
| **End of Quarter** | Architecture review | Full model refresh, validate assumptions |
| **Ad-hoc** | Incident post-mortem | Update model within 48 hours of incident |

---

### The Golden Rule

> **Threat model when the cost of modeling is less than the cost of not modeling.**

| Scenario | Threat Model? | Why |
|---|---|---|
| Changing a button color | ❌ No | No security impact |
| Adding a new API endpoint | ✅ Yes | New attack surface |
| Updating a CSS library | ❌ No | No trust boundary change |
| Integrating a payment provider | ✅ Yes | High-value target, compliance |
| Renaming a database column | ❌ No | No architectural change |
| Switching from SQL to NoSQL | ✅ Yes | New threat classes (NoSQLi) |

---

## Threat Modeling Frameworks

### STRIDE (Most Common)
Developed by Microsoft. Categorizes threats into 6 types.

| Letter | Threat | Description | Example |
|---|---|---|---|
| **S** | **Spoofing** | Pretending to be someone/something else | Stolen session cookie, fake API client |
| **T** | **Tampering** | Modifying data/code in transit or at rest | Changing product price in HTTP request |
| **R** | **Repudiation** | Denying an action happened | No audit log for payment transaction |
| **I** | **Information Disclosure** | Leaking sensitive data to unauthorized parties | Error message reveals SQL structure |
| **D** | **Denial of Service** | Making a system unavailable | DDoS, resource exhaustion |
| **E** | **Elevation of Privilege** | Gaining unauthorized capabilities | Normal user gains admin access |

### PASTA (Risk-Focused)
7-stage process aligned with business risk. Best for high-assurance systems.

1. Define objectives
2. Define technical scope
3. Application decomposition
4. Threat analysis
5. Vulnerability and weakness analysis
6. Attack modeling
7. Risk and impact analysis

### OCTAVE (Organizational)
Carnegie Mellon framework. Focuses on organizational risk and asset identification.

### LINDDUN (Privacy-Focused)
For GDPR/CCPA/privacy-heavy systems. Categories: Linkability, Identifiability, Non-repudiation, Detectability, Disclosure of information, Unawareness, Non-compliance.

### Attack Trees
Visual decision-tree structure where the root is the attacker's goal and branches are sub-goals. Best for complex systems.

```
[Steal Customer Data]
    ├── [Gain Database Access]
    │       ├── [SQL Injection]
    │       ├── [Compromise Credentials]
    │       └── [Exploit Unpatched DB]
    └── [Intercept in Transit]
            ├── [Man-in-the-Middle]
            └── [Downgrade to HTTP]
```

---

## Complete Walkthrough: E-Commerce Web App

### System Overview

An online store with:
- Web frontend (React)
- API backend (Node.js / Express)
- PostgreSQL database (products, users, orders)
- Redis cache (session store)
- Stripe API (payments)
- AWS S3 (image storage)

### Step 1: Draw the Data Flow Diagram (DFD)

```
┌─────────────────┐     HTTPS      ┌───────────────┐
│   Customer      │ ─────────────> │  Web Server   │
│   (Browser)     │                │  (Node.js)    │
└─────────────────┘                └───────┬───────┘
       │                                   │
       │  Upload Image                     │  SQL / Redis
       │                                   │
       v                                   v
┌───────────────┐              ┌──────────────────────┐
│   AWS S3      │              │   PostgreSQL         │
│  (Images)     │              │   (Users/Orders)     │
└───────────────┘              └──────────────────────┘
                                        │
                                        │ Session Cache
                                        v
                               ┌──────────────────────┐
                               │     Redis            │
                               │   (Sessions)         │
                               └──────────────────────┘

┌───────────────┐     Payment Token    ┌───────────────┐
│  Admin Portal │                      │  Stripe API   │
│   (Internal)  │ <──────────────────> │  (External)   │
└───────────────┘                      └───────────────┘
```

**Trust Boundaries** (where control changes):
1. **External Boundary**: Browser ↔ Web Server (untrusted / internet)
2. **Service Boundary**: Web Server ↔ Database / Redis / S3 / Stripe (internal services)
3. **Admin Boundary**: Admin Portal ↔ Web Server (privileged access, high-value target)

---

### Step 2: Apply STRIDE at Every Trust Boundary

#### Boundary 1: Browser ↔ Web Server (Untrusted)

| STRIDE | Threat | Attack Vector | Mitigation |
|---|---|---|---|
| **S**poofing | Session hijacking | Steal JWT from browser | HttpOnly cookies, secure flag, short expiry |
| **T**ampering | Man-in-the-middle attack | Intercept unencrypted traffic | TLS 1.3 everywhere, HSTS header |
| **R**epudiation | User denies placing order | No audit trail | Immutable order logs with timestamps |
| **I**nformation Disclosure | Error pages leak stack traces | Trigger errors intentionally | Generic error messages in production |
| **D**enial of Service | Flood login endpoint | Automated credential stuffing | Rate limiting, CAPTCHA, WAF |
| **E**levation | XSS steals admin cookie | Inject `<script>` in product review | CSP headers, output encoding, input validation |

#### Boundary 2: Web Server ↔ Database

| STRIDE | Threat | Attack Vector | Mitigation |
|---|---|---|---|
| **S**poofing | Impersonate database | DNS spoofing | TLS for DB connections, certificate pinning |
| **T**ampering | Modify order data | SQL injection | Parameterized queries, ORM, least-privilege DB user |
| **I**nformation Disclosure | Dump customer table | Union-based SQLi | Input validation, WAF rules, DB encryption at rest |
| **E**levation | DB user elevated to `postgres` | Exploit DB misconfig | Separate DB users per app, no `SUPERUSER` |

#### Boundary 3: Web Server ↔ S3

| STRIDE | Threat | Attack Vector | Mitigation |
|---|---|---|---|
| **T**ampering | Public S3 bucket writeable | Bucket ACL misconfig | `block_public_acls = true`, IAM conditions |
| **I**nformation Disclosure | Public image access | Signed URLs with expiry | Pre-signed URLs (15 min expiry), bucket policy |
| **D**enial of Service | Delete all images | Compromised AWS keys | S3 versioning, MFA delete, backup buckets |

#### Boundary 4: Admin Portal ↔ Web Server

| STRIDE | Threat | Attack Vector | Mitigation |
|---|---|---|---|
| **S**poofing | Fake admin login | Credential stuffing | MFA (TOTP), IP allowlisting, session timeout |
| **T**ampering | Privilege escalation via request | Modify role parameter to `admin` | Server-side role validation, never trust client input |
| **E**levation | Normal user becomes admin | IDOR on `/admin/users/123/role` | RBAC middleware, audit all privilege changes |
| **R**epudiation | Admin denies deleting user | No admin action audit | Immutable admin audit log (separate table, append-only) |

---

### Step 3: Prioritize Threats by Risk

Use a **Risk Matrix** (Likelihood × Impact):

| # | Threat | Likelihood | Impact | Risk Score | Priority |
|---|---|---|---|---|---|
| 1 | SQL Injection via search box | High (common attack) | Critical (full DB dump) | **Critical** | **P0** |
| 2 | XSS in product reviews | Medium | High (session theft) | **High** | **P1** |
| 3 | Public S3 bucket | Low (if IaC scanned) | Critical (data breach) | **High** | **P1** |
| 4 | Credential stuffing on login | High | Medium (account takeover) | **High** | **P1** |
| 5 | Admin privilege escalation | Low | Critical (total compromise) | **Medium** | **P2** |
| 6 | DDoS on flash sale | Medium | Medium (revenue loss) | **Medium** | **P2** |

---

### Step 4: Document Mitigations & Tracking

| Threat ID | Mitigation | Owner | Status | Verification |
|---|---|---|---|---|
| TM-001 | Implement parameterized queries for all DB calls | Backend Team | ✅ Done | SAST check passes |
| TM-002 | Add CSP header + output encoding for product reviews | Frontend Team | ✅ Done | Security headers verified |
| TM-003 | Checkov S3 bucket scan in CI/CD | Platform Team | ✅ Done | Terraform plan passes |
| TM-004 | Rate limit 5 attempts per IP per minute | Backend Team | 🔄 In Progress | Load test scheduled |
| TM-005 | Server-side RBAC validation on all admin endpoints | Backend Team | 🔄 In Progress | Code review pending |
| TM-006 | CloudFront + WAF rate limiting for flash sales | Platform Team | 📋 Backlog | Q3 planning |

---

## STRIDE in Practice

### Quick Reference: STRIDE per Data Flow Element

| DFD Element | Relevant STRIDE |
|---|---|
| **External Entity** (User, Browser, API Client) | Spoofing, Repudiation, Denial of Service |
| **Process** (Web Server, Microservice) | All six |
| **Data Store** (Database, Cache, S3) | Tampering, Information Disclosure, Denial of Service, Elevation |
| **Data Flow** (HTTP Request, DB Query) | Tampering, Information Disclosure, Denial of Service |
| **Trust Boundary** | Elevation of Privilege (crossing boundaries) |

### STRIDE Mitigation Cheat Sheet

| Threat | Design Principle | Implementation |
|---|---|---|
| Spoofing | Authentication | OAuth 2.0 + PKCE, MFA, JWT with expiry |
| Tampering | Integrity | HMAC signatures, checksums, TLS 1.3 |
| Repudiation | Non-repudiation | Immutable audit logs, signed transactions |
| Information Disclosure | Confidentiality | Encryption (at rest + in transit), least privilege |
| Denial of Service | Availability | Rate limiting, caching, horizontal scaling, DDoS protection |
| Elevation of Privilege | Authorization | RBAC, ABAC, server-side validation, zero trust |

---

## Threat Modeling Templates

### Lightweight Template (30-Minute Session)

Use for sprint planning or feature reviews.

```markdown
## Threat Model: [Feature Name]
**Date:** YYYY-MM-DD
**Team:** [Names]
**Scope:** [What is included/excluded]

### Data Flow Diagram
[Insert simple diagram or ASCII art]

### Trust Boundaries
1. [Boundary name]: [Who controls each side]
2. [Boundary name]: [Who controls each side]

### STRIDE Analysis
| Element | S | T | R | I | D | E | Notes |
|---------|---|---|---|---|---|---|-------|
| User → API | | | | | | | |
| API → DB | | | | | | | |

### Top 3 Threats
1. [Threat] — [Mitigation] — [Owner] — [Status]
2. [Threat] — [Mitigation] — [Owner] — [Status]
3. [Threat] — [Mitigation] — [Owner] — [Status]

### Decisions / Accepted Risks
- [Risk accepted with justification]

### Next Review
[Date / Trigger]
```

### Full Template (Architecture Review)

```markdown
## Threat Model: [System Name]
**Version:** 1.0
**Date:** YYYY-MM-DD
**Author(s):** [Names]
**Reviewers:** [Security Champion, Architect]

### 1. System Overview
[Brief description of what the system does]

### 2. Assets
| Asset | Sensitivity | Owner |
|-------|-------------|-------|
| Customer PII | High | Data Team |
| Payment Data | Critical | Compliance |
| Source Code | Medium | Engineering |

### 3. Architecture Diagram
[Detailed DFD with all components, data flows, and trust boundaries]

### 4. Threat Analysis
#### STRIDE per Trust Boundary
[Full table as shown in walkthrough]

### 5. Attack Trees
[For highest-risk threats]

### 6. Risk Register
| Threat ID | Description | Likelihood | Impact | Risk | Mitigation | Owner | Status |
|-----------|-------------|------------|--------|------|------------|-------|--------|
| TM-001 | ... | High | High | Critical | ... | ... | ... |

### 7. Compliance Mapping
| Requirement | Control | Verification |
|-------------|---------|--------------|
| PCI-DSS 3.4 | PAN encrypted at rest | DB encryption audit |
| GDPR 32 | PII access logging | Audit log review |

### 8. Assumptions & Out-of-Scope
- [Assumption: "AWS IAM is properly configured at account level"]
- [Out of scope: "Physical security of AWS data centers"]

### 9. Review History
| Date | Change | Author |
|------|--------|--------|
| YYYY-MM-DD | Initial model | ... |
```

---

## Common Anti-Patterns

### ❌ Anti-Pattern 1: "We'll threat model after we build it"
The purpose of threat modeling is to influence design. Doing it after coding is just expensive documentation.

### ❌ Anti-Pattern 2: Security team does it alone
The best threat models are collaborative. Developers know the code. Architects know the design. Security knows the attack patterns. All three must be in the room.

### ❌ Anti-Pattern 3: Trying to find every possible threat
Threat modeling is about finding the **most important** threats, not every theoretical edge case. Prioritize by risk.

### ❌ Anti-Pattern 4: No follow-through
Creating a threat model but not tracking mitigations is worse than not doing it at all — it creates false confidence.

### ❌ Anti-Pattern 5: One-and-done
Threat models are living documents. They rot when the architecture changes underneath them.

### ❌ Anti-Pattern 6: Perfect diagrams
A messy whiteboard photo with the right trust boundaries is infinitely better than a perfect diagram nobody reviewed together.

---

## Tools for Threat Modeling

| Tool | Purpose | Best For |
|---|---|---|
| **Microsoft Threat Modeling Tool** | Draw DFDs, auto STRIDE analysis | Windows teams, Microsoft-heavy environments |
| **OWASP Threat Dragon** | Open-source STRIDE tool | Web apps, open-source preference |
| **PyTM** | Python API for threat models as code | CI/CD integration, automated models |
| **ThreatSpec** | Markdown-based threat modeling | Docs-as-code workflows |
| **Miro / Mural / Lucidchart** | Collaborative diagramming | Remote teams, real-time sessions |
| **draw.io / diagrams.net** | Free DFD creation | Quick diagrams, no license cost |
| **泡沫 (Pytm)** | Pythonic threat models | Automated threat modeling at scale |

---

## Integrating Threat Modeling into DevSecOps

### Threat Modeling + Shift Left

```
Requirements → Architecture → Threat Model → Implementation → CI/CD → Production
     ↑           ↑              ↑              ↑              ↑           ↑
     │           │              │              │              │           │
  Security   Security      Security       SAST/SCA       Container    Runtime
  reqs       review        design         scans          scans        detection
             checklist     pattern                                                
```

### In a Sprint Cycle

| Day | Activity | Output |
|-----|----------|--------|
| **Sprint Planning** | 30-min mini threat model for new features |威胁 register for sprint |
| **Mid-Sprint** | Security champion review of high-risk changes | Mid-course corrections |
| **Sprint Review** | Update threat model with architecture changes | Updated model + risk register |
| **Sprint Retro** | Did any new risks emerge? | Lessons learned |

### Automation Opportunities

```yaml
# .github/workflows/threat-model.yml
name: Threat Model Validation

on:
  pull_request:
    paths:
      - 'architecture/**'
      - 'docs/threat-models/**'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Check Threat Model Exists
        run: |
          if [ ! -f "docs/threat-models/${{ github.event.pull_request.head.ref }}.md" ]; then
            echo "❌ Threat model not found for this architecture change"
            exit 1
          fi
```

---

## Summary

Threat modeling is **the cheapest security control that exists** because:
- It requires only **time, a whiteboard, and the right people**
- It finds flaws **before code is written**
- It creates **shared understanding** across teams
- It produces **living documentation** for security posture

| Without Threat Modeling | With Threat Modeling |
|---|---|
| Security as afterthought | Security as design principle |
| Reactionary patching | Proactive risk reduction |
| Blame during incidents | Shared ownership of risk |
| Expensive redesigns | Informed trade-offs |

> **"The best threat model is the one your developers actually use. Make it collaborative, lightweight, and iterative."**

---

*Generated for DevSecOps reference, architecture reviews, and security onboarding.*
