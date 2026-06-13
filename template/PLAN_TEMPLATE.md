# [Project Name]

**Tagline:** One sentence. What it does and for whom.  
**Category:** `web` | `mobile` | `ai-ml` | `devtools` | `infra` | `saas`  
**Author:** [@github-handle](https://github.com/username)  
**Date:** YYYY-MM-DD  
**Status:** `Draft` | `Complete` | `Superseded`

---

## 1. Genesis

**Problem observed:**  
Describe the real-world situation that revealed this problem. Be specific — where were you, what were you doing, what broke or frustrated you.

**Why existing solutions fail:**  
Name actual existing tools or approaches and state precisely why they are insufficient. Do not write "there is no good solution." Name what exists and what it gets wrong.

**Core hypothesis:**  
One sentence: if [this system] exists, then [this specific outcome] happens for [this person].

---

## 2. Target Persona

Not demographics. Behavior.

**Who they are:**  
Describe what they do daily, what tools they use, and what they are trying to accomplish.

**How they currently solve this problem:**  
Name the exact workaround, tool, or manual process they use today.

**What they will switch from and why:**  
Name the incumbent. State the switching trigger — what specifically makes them leave it.

---

## 3. Problem Statement

State the problem in precise language. No solution language in this section.

- What is happening that should not be, or what is not happening that should be?
- What is the cost of this problem? (time, money, risk, quality)
- What assumptions underpin this problem statement?

---

## 4. Solution Overview

**What this system does:**  
State the core function in one paragraph.

**Explicit non-goals (what this does NOT do):**  
List what is out of scope and why. This section prevents scope creep and sets contributor expectations.

- Does not: [X] — Reason: [Y]
- Does not: [X] — Reason: [Y]

---

## 5. System Architecture

Describe each major component and its sole responsibility. One component = one responsibility.

```mermaid
graph TD
    A[Client] -->|HTTPS| B[API Gateway]
    B --> C[Auth Service]
    B --> D[Core Service]
    D --> E[Database]
    D --> F[Cache]
    D --> G[Queue]
    G --> H[Worker]
```

**Component breakdown:**

| Component | Responsibility | Communicates With |
|---|---|---|
| API Gateway | Route, rate-limit, terminate TLS | Auth Service, Core Service |
| Auth Service | Issue and validate tokens | Database |
| Core Service | Business logic | Database, Cache, Queue |
| Worker | Async job processing | Database, external APIs |

---

## 6. Data Flow

Trace a primary user action end-to-end through the system.

```mermaid
sequenceDiagram
    participant U as User
    participant G as API Gateway
    participant A as Auth Service
    participant C as Core Service
    participant D as Database

    U->>G: POST /api/resource (JWT)
    G->>A: Validate token
    A-->>G: Token valid, user_id: 123
    G->>C: Forward request + user context
    C->>D: Write operation
    D-->>C: Acknowledge
    C-->>G: 201 Created
    G-->>U: Response
```

---

## 7. Data Models

**Core entities and relationships:**

```mermaid
erDiagram
    USER {
        uuid id PK
        string email UK
        string password_hash
        timestamp created_at
    }
    PROJECT {
        uuid id PK
        uuid owner_id FK
        string name
        jsonb config
        timestamp created_at
    }
    USER ||--o{ PROJECT : owns
```

**Schema decisions:**

| Decision | Reason |
|---|---|
| UUID over auto-increment IDs | Prevents enumeration attacks, safe for distributed systems |
| `jsonb` for config | Schema for config evolves frequently; avoids migrations for optional fields |
| Soft delete pattern | Audit trail requirement; data recovery without backup restore |

---

## 8. Tech Stack

Every row is mandatory. Every rejection reason is mandatory.

| Layer | Choice | Reason | Rejected Alternative | Why Rejected |
|---|---|---|---|---|
| Language | | | | |
| Framework | | | | |
| Database (primary) | | | | |
| Cache | | | | |
| Queue | | | | |
| Auth | | | | |
| File storage | | | | |
| Hosting | | | | |
| CI/CD | | | | |
| Monitoring | | | | |

---

## 9. Project Structure

```
project-root/
├── src/
│   ├── api/            # Route definitions and request validation
│   ├── services/       # Business logic, no framework imports
│   ├── repositories/   # All database access, no logic
│   ├── workers/        # Async job handlers
│   └── lib/            # Shared utilities with no side effects
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── infra/              # IaC — Terraform, Docker, k8s manifests
├── docs/               # ADRs and supplementary design docs
├── .env.example        # All env vars documented with descriptions
└── docker-compose.yml  # Local development environment
```

**Structural decisions:**

| Decision | Reason |
|---|---|
| `services/` has zero framework imports | Services remain testable without spinning up the HTTP layer |
| `repositories/` isolates all DB access | Swap database engine without touching business logic |
| `lib/` functions are pure | No hidden state; every utility is unit testable in isolation |

---

## 10. API Design

**Authentication strategy:** [JWT / OAuth2 / API Key / Session] — Reason: [why this over alternatives]

**Versioning strategy:** [URI versioning `/v1/` / Header versioning] — Reason: [why]

**Core endpoints:**

| Method | Endpoint | Auth Required | Description | Rate Limit |
|---|---|---|---|---|
| POST | `/v1/auth/register` | No | Create account | 10/min per IP |
| POST | `/v1/auth/login` | No | Issue JWT | 20/min per IP |
| GET | `/v1/resource` | Yes | List resources (paginated) | 100/min per user |
| POST | `/v1/resource` | Yes | Create resource | 30/min per user |
| PATCH | `/v1/resource/:id` | Yes | Partial update | 30/min per user |
| DELETE | `/v1/resource/:id` | Yes | Soft delete | 10/min per user |

**Error response shape:**
```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Human-readable message",
    "field": "email"
  }
}
```

**Pagination shape:**
```json
{
  "data": [],
  "pagination": {
    "cursor": "opaque_string",
    "has_next": true,
    "limit": 20
  }
}
```

---

## 11. Security

**Threat model:**

| Threat | Vector | Mitigation |
|---|---|---|
| Credential stuffing | Brute force on `/auth/login` | Rate limiting + CAPTCHA after 5 failures |
| Broken object-level authorization | User A accessing User B's resource via ID manipulation | Ownership check on every resource query |
| SQL injection | User input in queries | Parameterized queries only; ORM with no raw string interpolation |
| Secrets in source | Env vars committed | `.env` in `.gitignore`; secrets manager in production |
| Data exfiltration | Bulk export via paginated API | Rate limiting; audit log on bulk access patterns |

**Auth/AuthZ decisions:**

- Token type: [JWT / Opaque] — Reason:
- Token expiry: [duration] — Reason:
- Refresh strategy: [sliding / absolute] — Reason:
- Permission model: [RBAC / ABAC / flat scopes] — Reason:

**Sensitive data handling:**

- Passwords: bcrypt with cost factor ≥ 12
- PII fields: [encrypted at rest / not stored / hashed]
- Logging: PII fields stripped before any log statement
- Data at rest: [encryption method]
- Data in transit: TLS 1.2 minimum, 1.3 preferred

**Blast radius of a breach:**

State what an attacker has if they compromise: (a) the database, (b) an API server instance, (c) a user's session token.

---

## 12. Scalability

**Current design ceiling:**  
This architecture handles approximately [N] concurrent users / [N] requests per second before [specific component] becomes the bottleneck.

**First bottleneck:**  
[Component] — [why it breaks first and at what load signal]

**Scaling path:**

| Stage | Users | Change Required |
|---|---|---|
| MVP | 0–1K | Single server, managed DB |
| Growth | 1K–50K | Read replicas, Redis cache, CDN |
| Scale | 50K–500K | Horizontal API scaling, DB sharding or managed OLAP |
| Beyond | 500K+ | Redesign [specific component]; this document does not cover that |

**Caching strategy:**

| What is cached | Where | TTL | Invalidation trigger |
|---|---|---|---|
| Session tokens | Redis | 24h | Logout, password change |
| [Resource] list | Redis | 5min | Write to [resource] |

---

## 13. Build vs Buy

| Component | Decision | Reason |
|---|---|---|
| Auth | Buy (Auth0 / Clerk / Supabase Auth) | Auth edge cases (MFA, OAuth flows, session management) take months to get right; not a differentiator |
| Email delivery | Buy (Resend / Postmark) | Deliverability is a specialized problem involving IP reputation, SPF/DKIM; build only if email is the product |
| File storage | Buy (S3 / Cloudflare R2) | Object storage is commodity infrastructure |
| [Core feature] | Build | This is the product's differentiating logic; no existing solution covers it |

---

## 14. Failure Modes

| Component | Failure Scenario | Degradation Strategy |
|---|---|---|
| Database | Primary unavailable | Read replica promotes; writes queue for [N] minutes then fail with 503 |
| Cache | Redis unavailable | Fall through to database; accept latency increase; alert fires |
| Queue | Broker unavailable | Jobs fail fast with retry; UI shows pending state; no data loss |
| Third-party API | [Service] returns 5xx | Circuit breaker opens after 5 failures; fallback to [X] or user-visible error |
| Auth service | Token validation fails | Fail closed — no access granted; return 401 |

---

## 15. Cost Architecture

All figures are rough order-of-magnitude estimates. Assumes cloud provider pricing as of plan date.

| Resource | 100 users/mo | 10K users/mo | 100K users/mo |
|---|---|---|---|
| Compute | $X | $X | $X |
| Database | $X | $X | $X |
| Cache | $X | $X | $X |
| Storage | $X | $X | $X |
| CDN / Bandwidth | $X | $X | $X |
| Third-party APIs | $X | $X | $X |
| **Total estimate** | **$X** | **$X** | **$X** |

**Cost ceiling:** At [N] users, [specific cost component] dominates. The unit economics [are / are not] viable at that scale because [reason].

---

## 16. Limitations

Specific and honest. No vague disclaimers.

- **[Limitation]:** [Precise description of what breaks, under what conditions, and why fixing it requires a redesign or was deprioritized]
- **[Limitation]:** 
- **[Limitation]:**

---

## 17. Rejected Alternatives

System-level alternatives that were seriously considered and discarded.

| Alternative | Why Considered | Why Rejected |
|---|---|---|
| [Architecture pattern / different stack] | [Genuine merit it had] | [Specific reason it lost] |

---

## 18. Decision Log

Every major decision made during the design of this plan.

| # | Decision | Options Considered | Chosen | Reason | Trade-off Accepted |
|---|---|---|---|---|---|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |

---

## 19. Future Work

Each phase has a trigger condition — a measurable signal that makes the phase relevant. No dates.

**Phase 1 — [Name]**  
Trigger: [condition, e.g., "when DAU exceeds 5,000"]  
- [ ] Feature A
- [ ] Feature B

**Phase 2 — [Name]**  
Trigger: [condition]  
- [ ] Feature C

**Phase 3 — [Name]**  
Trigger: [condition]  
- [ ] Major architectural change

---

## 20. References

Papers, products, systems, and patterns that directly influenced decisions in this plan.

- [Title / Product](URL) — Influenced: [which section or decision]
- [Title / Product](URL) — Influenced: [which section or decision]