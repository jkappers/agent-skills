# Specification Guide

Extended guidance for feature specifications. See SKILL.md for core rules.

---

## Document Boundary

A spec captures WHAT and WHY — never HOW. Do not prescribe implementation approach, technology choices, or technical strategy.

**Must not appear in a spec:** Implementation approach, architecture diagrams, code examples, database schemas, API signatures, technology choices, performance specifications.

---

## File Naming

**Rules:**
- Always use hyphens (never underscores)
- Always use lowercase
- Use descriptive names (not abbreviations)

**Spec directory pattern:** `specs/feature-name/`

Examples:
- `specs/user-notifications/`
- `specs/payment-integration/`

---

## When to Create an ADR

If a specification establishes a pattern with project-wide implications, create an ADR to document the reusable pattern separately.

| Content | Location | Reason |
|---------|----------|--------|
| "Use message queues for async processing" | ADR | Project-wide pattern |
| "Notification delivery uses message queue" | Spec | Feature-specific application |
| "Use PostgreSQL for persistence" | ADR | Project-wide technology |
| "Notification preferences stored in users table" | Spec | Feature-specific storage |

---

## Archival

Implemented specifications move to `specs/historical/` for reference.
