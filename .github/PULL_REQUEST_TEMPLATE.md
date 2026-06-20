## Plan Submission Checklist

A maintainer will not begin review until every box is checked. Unchecked boxes = closed PR.

### File

- [ ] File is placed in the correct `plans/` subdirectory
- [ ] File name is `kebab-case` and ends in `.md`
- [ ] `PLAN_INDEX.md` has been updated with a new row (newest first)

### Template Compliance

- [ ] All 26 sections from `PLAN_TEMPLATE.md` are present
- [ ] No section contains placeholder text or `[TODO]`

### Foundation

- [ ] Engineering Principles state practical meaning, not slogans
- [ ] Target Persona is behavioral, not demographic
- [ ] Problem Statement contains no solution language

### Architecture

- [ ] System architecture Mermaid diagram is present and renders correctly
- [ ] Non-negotiable constraints are listed alongside the architecture, not just components
- [ ] Data flow Mermaid sequence diagram is present and renders correctly
- [ ] Data model Mermaid ER diagram is present and renders correctly
- [ ] Runnable DDL is present for every core table, not only the ER diagram
- [ ] Each component in the architecture diagram has a documented responsibility

### Tech Stack

- [ ] Every row in the Tech Stack table has: Choice, Reason, Rejected Alternative, Why Rejected
- [ ] No row says "popular" or "widely used" as the primary reason for a choice
- [ ] Key Dependencies lists actual package names with one-line purpose

### Configuration & Contracts

- [ ] Configuration Reference covers every env var referenced anywhere else in the plan
- [ ] Internal Service Contracts are defined if the system has more than one internal service or worker, or this section is explicitly marked not applicable

### Security

- [ ] Threat model table has at least 4 named threats with specific mitigations
- [ ] Auth/AuthZ decisions include the reason for each choice
- [ ] Blast radius is described for: database compromise, server instance compromise, user token compromise

### Scalability

- [ ] A numeric user/request ceiling is stated for the current design
- [ ] The first bottleneck is named (specific component, not "the backend")
- [ ] The scaling path table has at least 3 stages

### Testing & Build Integrity

- [ ] Testing Strategy table names a tool and cadence for every test level
- [ ] Build vs Buy table covers every major component, build or buy, with reason
- [ ] Key Algorithms & Critical Logic includes a reference implementation for non-obvious logic, or is explicitly marked not applicable

### Honesty Sections

- [ ] Limitations contains specific failure conditions, not vague disclaimers
- [ ] Rejected Alternatives contains at least one system-level alternative with a genuine reason for rejection
- [ ] Risk Register covers wrong design assumptions, distinct from the Failure Modes table
- [ ] Cost Architecture has estimates for all three user tiers (100 / 10K / 100K)

### Decision Log

- [ ] The Decision Log has at least one row per major technology or architecture choice
- [ ] Every row has: Options Considered, Chosen, Reason, Trade-off Accepted

### Roadmap

- [ ] Implementation Roadmap covers the pre-launch build order with phases or weeks
- [ ] Future Work phases have trigger conditions (measurable signal, not a date)
- [ ] Implementation Roadmap and Future Work do not duplicate each other

---

**Plan title:**  
**Category:**  
**One-line problem statement:**