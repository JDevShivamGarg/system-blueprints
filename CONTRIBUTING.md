# Contributing to system-blueprints

## What This Repo Accepts

A submitted plan is accepted if an engineer could read it and build the described system without asking a single clarifying question. That is the only quality bar.

Plans are rejected if they contain:
- Vague technology choices without documented reasoning
- A Decision Log with empty rows
- A threat model limited to "use HTTPS" or equivalent
- Scalability claims without a stated ceiling and identified bottleneck
- A Limitations section with disclaimers instead of specific failure conditions
- Engineering Principles that are slogans with no stated practical meaning
- A Testing Strategy table with empty tool or cadence columns
- An Implementation Roadmap with no week or order indicators
- A Risk Register that duplicates Failure Modes instead of covering wrong assumptions

---

## Before You Submit

**Core**
- [ ] You have used `template/PLAN_TEMPLATE.md` as the base
- [ ] Every section is filled — no placeholder text, no `[TODO]`
- [ ] Engineering Principles state practical meaning, not just a slogan
- [ ] Mermaid diagrams are present for: system architecture, data flow, and data models
- [ ] System Architecture includes non-negotiable constraints, not just components

**Technical depth**
- [ ] Every tech stack row has a rejected alternative and a reason for rejection
- [ ] Key Dependencies lists actual package names, not just layer-level choices
- [ ] Data Models includes runnable DDL, not only the ER diagram
- [ ] Configuration Reference covers every tunable parameter referenced elsewhere in the plan
- [ ] Internal Service Contracts are defined if the system has more than one internal service or worker
- [ ] Key Algorithms & Critical Logic includes a reference implementation for any non-obvious logic, or is explicitly justified as not applicable

**Rigor**
- [ ] The Decision Log has a row for every non-obvious choice in the plan
- [ ] The threat model names specific threats with specific mitigations
- [ ] The scalability section states a numeric ceiling and names the first bottleneck
- [ ] The Testing Strategy table names a tool and cadence for every test level
- [ ] The Risk Register covers wrong design assumptions, distinct from Failure Modes
- [ ] The Limitations section contains specific failure conditions, not caveats
- [ ] Cost estimates exist for all three user tiers

**Roadmap**
- [ ] Implementation Roadmap covers the pre-launch build order with phases or weeks
- [ ] Future Work phases have trigger conditions, not dates, and are distinct from the Implementation Roadmap

---

## File Naming

`kebab-case-project-name.md`

Examples:
- `ai-powered-code-review-tool.md`
- `realtime-collaborative-whiteboard.md`
- `multi-tenant-saas-billing-engine.md`

---

## Directory

Place the file in the correct subdirectory under `plans/`:

| Category | Use when |
|---|---|
| `web/` | Browser-first product; primary interface is a website |
| `mobile/` | iOS or Android native or cross-platform mobile app |
| `ai-ml/` | Core value proposition is an AI/ML model or pipeline |
| `devtools/` | Primary users are software engineers; tool aids development |
| `infra/` | Infrastructure, platform, or developer operations tooling |
| `saas/` | Multi-tenant, subscription-based, any category |

If a plan fits multiple categories, choose the one that describes the primary user interaction surface.

---

## Updating PLAN_INDEX.md

Add one row to the table in `PLAN_INDEX.md`:

```markdown
| [Project Name](plans/category/filename.md) | `category` | @github-handle | YYYY-MM-DD | One-sentence problem statement |
```

The row goes at the top of the table (newest first).

---

## Pull Request Process

1. Fork the repository.
2. Create a branch: `plan/your-project-name`.
3. Add your plan file and update `PLAN_INDEX.md`.
4. Open a pull request against `main`.
5. The PR template checklist must be fully checked before review begins.
6. A maintainer will review for completeness, not for agreement with your design choices. Design disagreements are not grounds for rejection. Incomplete sections are.

---

## What Maintainers Will Not Do

- Rewrite your plan for you
- Accept a plan with a comment saying "will fill in later"
- Merge a plan where the Decision Log has fewer entries than major technology choices
- Engage in debates about which tech stack is objectively correct

---

## Editing an Existing Plan

If you are the original author, open a PR with changes and update the `Date` field and set `Status: Superseded` on the old version if the changes are architectural in nature.

If you are not the original author, open an Issue first. Do not submit unsolicited rewrites of another contributor's plan.

---

## Licensing

By submitting a plan, you agree that your contribution is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). You retain authorship credit. Others may build upon your plan with attribution.