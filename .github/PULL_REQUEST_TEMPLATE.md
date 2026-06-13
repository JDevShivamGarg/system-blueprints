## Plan Submission Checklist

A maintainer will not begin review until every box is checked. Unchecked boxes = closed PR.

### File

- [ ] File is placed in the correct `plans/` subdirectory
- [ ] File name is `kebab-case` and ends in `.md`
- [ ] `PLAN_INDEX.md` has been updated with a new row (newest first)

### Template Compliance

- [ ] All 20 sections from `PLAN_TEMPLATE.md` are present
- [ ] No section contains placeholder text or `[TODO]`

### Architecture

- [ ] System architecture Mermaid diagram is present and renders correctly
- [ ] Data flow Mermaid sequence diagram is present and renders correctly
- [ ] Data model Mermaid ER diagram is present and renders correctly
- [ ] Each component in the architecture diagram has a documented responsibility

### Tech Stack

- [ ] Every row in the Tech Stack table has: Choice, Reason, Rejected Alternative, Why Rejected
- [ ] No row says "popular" or "widely used" as the primary reason for a choice

### Security

- [ ] Threat model table has at least 4 named threats with specific mitigations
- [ ] Auth/AuthZ decisions include the reason for each choice
- [ ] Blast radius is described for: database compromise, server instance compromise, user token compromise

### Scalability

- [ ] A numeric user/request ceiling is stated for the current design
- [ ] The first bottleneck is named (specific component, not "the backend")
- [ ] The scaling path table has at least 3 stages

### Decision Log

- [ ] The Decision Log has at least one row per major technology or architecture choice
- [ ] Every row has: Options Considered, Chosen, Reason, Trade-off Accepted

### Honesty Sections

- [ ] Limitations contains specific failure conditions, not vague disclaimers
- [ ] Rejected Alternatives contains at least one system-level alternative with a genuine reason for rejection
- [ ] Cost Architecture has estimates for all three user tiers (100 / 10K / 100K)

### Future Work

- [ ] Every phase has a trigger condition (measurable signal, not a date)

---

**Plan title:**  
**Category:**  
**One-line problem statement:**