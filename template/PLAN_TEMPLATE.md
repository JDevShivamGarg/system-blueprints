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

## 2. Engineering Principles

Non-negotiable axioms that govern every decision in this plan. If a later decision conflicts with one of these, the principle wins unless this section is explicitly revised.

1. **[Principle]** — [What it means in practice, not just the slogan]
2. **[Principle]** — [What it means in practice]
3. **[Principle]** — [What it means in practice]

Examples of real principles, not placeholders: "Modular independence — every component is independently deployable and replaceable without touching others." "No vibe coding — every decision has a written rationale, every component has tests." "Anonymous by design — no field in the system is linkable to real-world identity."

---

## 3. Target Persona

Not demographics. Behavior.

**Who they are:**  
Describe what they do daily, what tools they use, and what they are trying to accomplish.

**How they currently solve this problem:**  
Name the exact workaround, tool, or manual process they use today.

**What they will switch from and why:**  
Name the incumbent. State the switching trigger — what specifically makes them leave it.

---

## 4. Problem Statement

State the problem in precise language. No solution language in this section.

- What is happening that should not be, or what is not happening that should be?
- What is the cost of this problem? (time, money, risk, quality)
- What assumptions underpin this problem statement?

---

## 5. Solution Overview

**What this system does:**  
State the core function in one paragraph.

**Explicit non-goals (what this does NOT do):**  
List what is out of scope and why. This section prevents scope creep and sets contributor expectations.

- Does not: [X] — Reason: [Y]
- Does not: [X] — Reason: [Y]

---

## 6. System Architecture

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

**Non-negotiable constraints:**

Hard rules that shape every component above. Violating one of these is a design defect, not a trade-off to revisit later.

- [Constraint] — [why it's non-negotiable, what breaks if violated]
- [Constraint] — [why it's non-negotiable]

Examples of real constraints: "All async jobs must be resumable from DB state after a process restart — no in-memory-only job state." "Service-to-service calls never block on a third-party LLM API — always queued."

---

## 7. Data Flow

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

## 8. Data Models

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

**Schema definition (runnable DDL):**

The ER diagram is the overview. This is the contract. Include actual `CREATE TABLE` statements for every core table — not a subset, not an approximation.

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## 9. Tech Stack

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

**Key dependencies (concrete package list):**

The table above is layer-level. This is package-level — the exact library a contributor will install. One line per package.

| Package | Purpose |
|---|---|
| `package-name` | One-line reason this exact package was chosen over the generic alternative |

---

## 10. Project Structure

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

## 11. Configuration Reference

Every parameter that controls runtime behavior lives here, not hardcoded in source. If a value can reasonably change between dev, staging, and prod — or might need tuning after launch — it is an environment variable documented in this table.

```bash
# === Service URLs ===
DATABASE_URL=postgresql://user:pass@host:5432/db
REDIS_URL=redis://redis:6379/0

# === Auth ===
JWT_SECRET=replace-in-prod
JWT_EXPIRY_SECONDS=900

# === Rate Limiting ===
RATE_LIMIT_PER_MINUTE=60
```

| Variable | Default | Controls | Why This Default |
|---|---|---|---|
| `JWT_EXPIRY_SECONDS` | 900 | Access token lifetime | Short enough to limit stolen-token blast radius, long enough to avoid excessive refresh traffic |

---

## 12. API Design

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

**Internal service contracts (required only if this system has more than one internal service or background worker):**

Public API above is for external clients. This is the contract between your own components — what every other module, and every other contributor, codes against. Define explicit input and output shapes per internal service.

```python
# auth-service — Input
class AuthRequest(BaseModel):
    token: str

# auth-service — Output
class AuthContext(BaseModel):
    user_id: str
    roles: list[str]
    permissions: list[str]
```

---

## 13. Security

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

## 14. Scalability

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

## 15. Testing Strategy

| Level | Tool | What It Tests | When It Runs |
|---|---|---|---|
| Unit | | Individual functions in isolation | Every commit |
| Integration | | Service-to-storage interactions | Every PR |
| Contract | | Inter-service API contracts match | Every PR |
| End-to-end | | Full user flow start to finish | Nightly |
| Security | | Injection attacks, authz bypass, PII leakage | Every PR |
| Load | | Concurrent users, target p95 latency | Weekly / pre-release |

---

## 16. Build vs Buy

| Component | Decision | Reason |
|---|---|---|
| Auth | Buy (Auth0 / Clerk / Supabase Auth) | Auth edge cases (MFA, OAuth flows, session management) take months to get right; not a differentiator |
| Email delivery | Buy (Resend / Postmark) | Deliverability is a specialized problem involving IP reputation, SPF/DKIM; build only if email is the product |
| File storage | Buy (S3 / Cloudflare R2) | Object storage is commodity infrastructure |
| [Core feature] | Build | This is the product's differentiating logic; no existing solution covers it |

---

## 17. Key Algorithms & Critical Logic

Reference implementation — pseudocode or real code — for any logic where a careless implementation silently breaks the system. Skip this section only if the system genuinely has no non-obvious algorithmic logic; do not skip it to save time.

### [Algorithm Name]
**Why this needs a reference implementation:** [what goes wrong if implemented carelessly, e.g. "naive rate limiting drops legitimate requests at window boundaries"]

```python
def algorithm():
    """Reference implementation."""
    pass
```

---

## 18. Failure Modes

| Component | Failure Scenario | Degradation Strategy |
|---|---|---|
| Database | Primary unavailable | Read replica promotes; writes queue for [N] minutes then fail with 503 |
| Cache | Redis unavailable | Fall through to database; accept latency increase; alert fires |
| Queue | Broker unavailable | Jobs fail fast with retry; UI shows pending state; no data loss |
| Third-party API | [Service] returns 5xx | Circuit breaker opens after 5 failures; fallback to [X] or user-visible error |
| Auth service | Token validation fails | Fail closed — no access granted; return 401 |

---

## 19. Risk Register

Distinct from Failure Modes above. Failure Modes covers what happens when a component goes down. This covers what happens when a design assumption turns out to be wrong.

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| [Design assumption that might not hold] | Low/Medium/High | Low/Medium/High | [What you'll do if it turns out wrong] |

---

## 20. Cost Architecture

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

## 21. Limitations

Specific and honest. No vague disclaimers.

- **[Limitation]:** [Precise description of what breaks, under what conditions, and why fixing it requires a redesign or was deprioritized]
- **[Limitation]:** 
- **[Limitation]:**

---

## 22. Rejected Alternatives

System-level alternatives that were seriously considered and discarded.

| Alternative | Why Considered | Why Rejected |
|---|---|---|
| [Architecture pattern / different stack] | [Genuine merit it had] | [Specific reason it lost] |

---

## 23. Decision Log

Every major decision made during the design of this plan.

| # | Decision | Options Considered | Chosen | Reason | Trade-off Accepted |
|---|---|---|---|---|---|
| 1 | | | | | |
| 2 | | | | | |
| 3 | | | | | |

---

## 24. Implementation Roadmap

The build order to reach a working v1. Distinct from Future Work below — this section ends at launch, not after it.

**Phase 1 — [Name] (Week N–N)**
- [ ] Task
- [ ] Task

**Phase 2 — [Name] (Week N–N)**
- [ ] Task
- [ ] Task

---

## 25. Future Work

Work that follows launch. Each phase has a trigger condition — a measurable signal that makes the phase relevant. No dates.

**Phase 1 — [Name]**  
Trigger: [condition, e.g., "when DAU exceeds 5,000"]  
- [ ] Feature A
- [ ] Feature B

**Phase 2 — [Name]**  
Trigger: [condition]  
- [ ] Feature C

---

## 26. References

Papers, products, systems, and patterns that directly influenced decisions in this plan.

- [Title / Product](URL) — Influenced: [which section or decision]
- [Title / Product](URL) — Influenced: [which section or decision]