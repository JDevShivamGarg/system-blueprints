# system-blueprints

> A public archive of high-level design documents for software projects. No code. Only thinking.

---

## What This Is

A curated collection of complete, decision-driven design documents for software project ideas. Each document is a full architectural plan — written before any code exists — covering the problem, the system, every technology decision with its reasoning, security posture, failure modes, scalability ceiling, and honest limitations.

The artifact here is not the implementation. It is the reasoning behind it.

---

## Why This Exists

In a world where code generation is increasingly automated, the scarcest and most valuable thing is not syntax — it is design judgment. The ability to define a problem precisely, choose the right architecture, make explicit trade-offs, and document the reasoning behind every decision.

Most public repositories contain code with no explanation of why it was written that way. This repo inverts that. The document *is* the deliverable.

---

## What a Plan Contains

Every submitted plan must cover:

| Section | Description |
|---|---|
| **Genesis** | Problem observed, why existing solutions fail, core hypothesis |
| **Target Persona** | Behavioral description of who this is for |
| **Problem Statement** | Precise problem definition, no solution language |
| **Solution Overview** | What this does and what it explicitly does not do |
| **System Architecture** | Component breakdown with Mermaid diagram |
| **Data Flow** | Request/data lifecycle with Mermaid sequence diagram |
| **Data Models** | Entities, relationships, schema decisions with Mermaid ER diagram |
| **Tech Stack** | Every layer: choice, reason, rejected alternative, why rejected |
| **Project Structure** | Folder layout with reasoning for structure decisions |
| **API Design** | Endpoints, auth strategy, versioning — each with reason |
| **Security** | Threat model, auth/authz decisions, sensitive data handling, blast radius |
| **Scalability** | Design ceiling, first bottleneck, scaling path, caching strategy |
| **Build vs Buy** | Per component: build or buy, with reason |
| **Failure Modes** | Per critical component: failure scenario and degradation strategy |
| **Cost Architecture** | Rough infra cost at 100 / 10K / 100K users |
| **Limitations** | Specific, honest. Not vague disclaimers |
| **Rejected Alternatives** | System-level alternatives considered and discarded, with reasons |
| **Decision Log** | Table of every major decision: options → chosen → reason → trade-off accepted |
| **Future Work** | Phased roadmap. Each phase has a trigger condition, not a date |
| **References** | Papers, products, and patterns that influenced the design |

---

## Repository Structure

```
system-blueprints/
├── plans/
│   ├── web/
│   ├── mobile/
│   ├── ai-ml/
│   ├── devtools/
│   ├── infra/
│   └── saas/
├── template/
│   └── PLAN_TEMPLATE.md
├── PLAN_INDEX.md
├── CONTRIBUTING.md
└── README.md
```

---

## Plan Index

| Plan | Category | Author | Date | Problem |
|---|---|---|---|---|
| *(first plan coming soon)* | — | — | — | — |

---

## How to Contribute

1. Fork this repository.
2. Copy `template/PLAN_TEMPLATE.md` into the appropriate `plans/` subdirectory.
3. Name the file in `kebab-case`: `ai-powered-code-review-tool.md`.
4. Complete every section. Incomplete sections are grounds for rejection.
5. Open a pull request. The PR template will enforce a checklist before merge.

**Pull requests that skip sections, use vague language in the Decision Log, or omit the threat model will not be merged.**

---

## Quality Standard

This is not a brainstorming dump. Every plan submitted here must reflect the thinking of someone who has considered the problem seriously, evaluated alternatives, and made explicit, reasoned decisions. Vague statements like "it will scale horizontally" or "security will be handled with JWT" without further reasoning do not meet the bar.

The test: could an engineer read this document and build the system without asking a single clarifying question? If no, the plan is incomplete.

---

## What This Is Not

- A list of startup ideas
- A code repository
- A tutorial collection
- A wishlist

---

## License

All plans in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You may use, adapt, and build upon any plan with attribution.
