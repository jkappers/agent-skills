# Architecture Decision Record (ADR) Guide

This guide documents how to structure Architecture Decision Records.

---

## Purpose

ADRs document project-wide architectural decisions. Create an ADR for decisions that:

- **Cross-cut multiple systems** (affect architecture, not just one feature)
- **Set project-wide precedent** (patterns to follow across the codebase)
- **Are difficult to reverse** (significant investment to change)
- **Require explicit rationale** (non-obvious trade-offs)

**Examples requiring ADRs:**
- Core architectural patterns (e.g., extension mechanisms, plugin architecture)
- Data storage technology choices (e.g., database selection, caching strategy)
- Deployment architecture (e.g., containerization, infrastructure decisions)
- Cross-system patterns (e.g., API conventions, error handling standards)

**Examples NOT requiring ADRs:**
- Feature-specific design choices (document in the feature spec instead)
- Implementation details within a single system
- Temporary workarounds or experiments

---

## Directory Structure

```
docs/adr/
├── 001-database-selection.md
├── 002-api-versioning-strategy.md
├── 003-caching-architecture.md
├── 004-authentication-approach.md
└── 005-deployment-architecture.md
```

---

## File Naming Conventions

Pattern: `docs/adr/NNN-descriptive-title.md`

- Use 3-digit sequential numbers: `001`, `002`, `003`...
- Never reuse numbers (even for rejected ADRs)
- Always use hyphens (never underscores)
- Always use lowercase

Examples:
- `docs/adr/001-database-selection.md`
- `docs/adr/007-api-versioning-strategy.md`

---

## ADR Template

Use [`assets/adr-template.md`](../assets/adr-template.md) as the canonical template for all ADRs.

---

## ADR Status

| Status | Meaning |
|--------|---------|
| **Proposed** | Draft under review |
| **Accepted** | Decision finalized and in effect |
| **Rejected** | Considered but not adopted (retained for historical record) |
| **Superseded** | Replaced by a newer ADR (link to replacement) |

ADRs are immutable once accepted. To change a decision, create a new ADR that supersedes the previous one.

---

## Content Guidelines

### Prohibited Content

**Never include in any ADR:**

**Development Effort Estimates:**
- "Estimated Effort: 40-60 hours"
- "Implementation time: 2-3 weeks"

**Timeline References:**
- "Target delivery: December 2025"
- "Sprint 3 completion"
