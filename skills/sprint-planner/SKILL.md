---
name: sprint-planner
description: >
  Convert complex implementation plans into focused, autonomous agent sprints with measurable acceptance criteria.
  Use when (1) decomposing a large plan into agent-executable chunks,
  (2) breaking down multi-endpoint migrations or multi-step projects,
  (3) creating sprint definitions with acceptance criteria for autonomous agents,
  (4) planning dependency chains across implementation sprints,
  or (5) structuring work for parallel agent execution.
argument-hint: "[path/to/plan or description]"
---

# Sprint Planner

Decompose complex implementation plans into focused, autonomous agent sprints. Each sprint is self-contained, has measurable acceptance criteria, and can execute without human intervention.

When invoked with arguments, decompose: $ARGUMENTS

## Workflow

1. **Load the plan** — Read the plan from the argument path or user input. If the input is a description rather than a file, ask for the full plan or spec location.
2. **Inventory work items** — List every discrete deliverable (endpoints, migrations, features, integrations). Categorize each by complexity: setup, simple, stub, complex, integration.
3. **Identify dependencies** — Map which items depend on others. Group independent items for parallel execution.
4. **Batch into sprints** — Apply the Batching Strategy table below. Each sprint gets 2-6 items maximum.
5. **Write sprint definitions** — Define each sprint using the Sprint Structure fields. Write acceptance criteria using the Acceptance Criteria Format.
6. **Build dependency chain** — Express the full chain as: `1 → 2, 3 → 4 → 5`
7. **Output the plan** — Write the sprint plan using [`assets/sprint-template.yaml`](assets/sprint-template.yaml). Append the Execution Protocol and Rollback sections from [`references/execution-protocol.md`](references/execution-protocol.md).

## Sprint Requirements

Every sprint **MUST** satisfy:

| Requirement | Constraint |
|-------------|------------|
| Duration | 2-4 hours |
| Scope | 2-6 discrete items maximum |
| Acceptance criteria | 3-5 verifiable checks with measurable outcomes |
| Dependencies | Explicitly listed by sprint ID |
| Autonomy | Executable without human intervention |

## Sprint Structure

Every sprint defines these fields:

| Field | Requirement |
|-------|-------------|
| `sprint_id` | Sequential integer |
| `title` | Format: "Category: Description" (e.g., "Setup: Create Feature Structure") |
| `duration_hours` | 1-4 hours |
| `dependencies` | List of sprint IDs that must complete first. Empty list if none |
| `input_state` | Exact filesystem/database state before sprint |
| `output_state` | Exact filesystem/database state after sprint |
| `scope` | Bulleted list of discrete work items |
| `acceptance_criteria` | 3-5 verifiable checks (see format below) |
| `constraints` | Explicit prohibitions — what NOT to do |
| `reference_material` | File path + line range or section name. Never paste full source |

## Acceptance Criteria Format

Use measurable, deterministic checks. **Never** use "code looks good", "follows conventions", or "works correctly".

| Criterion Type | Format | Example |
|----------------|--------|---------|
| Artifact exists | "Files exist: [paths]" | "Files exist: FundEndpoints.cs, AccountEndpoints.cs" |
| Code pattern present | "[File] contains [pattern]" | "InvestorApiEndpoints.cs registers 5 handler methods" |
| Build/test success | "[command] succeeds with [result]" | "dotnet build --configuration Release succeeds with 0 errors" |
| Verification output | "[endpoint/command] returns [expected]" | "GET /api/v1/investors returns 200 with JSON array" |
| Integration checkpoint | "Manual review: [action]" | "Manual review: Run smoke test and verify UI loads" |

## Batching Strategy

| Sprint Type | Duration | Items | Use For |
|-------------|----------|-------|---------|
| Setup | 1h | 1 complex setup | Folder structure, service registration, scaffolding |
| Simple CRUD bulk | 2-3h | 5-10 endpoints | Straightforward GET/POST handler migrations |
| Stub migration | 2h | 8-12 stubs | Replace hardcoded data with database queries |
| Complex work | 3-4h | 1-2 items | PDF generation, calculations, email, file processing |
| Integration | 2h | 1 task | Route registration, config updates, OpenAPI verification |

**Batching rules:**
- Separate infrastructure setup from implementation
- Group by feature or category to reduce context switching
- Isolate complex work into dedicated sprints
- Never mix setup and implementation in the same sprint

## Context Minimization

Provide only what the sprint needs. Reference the full plan by file path — do not repeat content.

```yaml
# Correct
reference_material: "docs/plan.md lines 45-67"

# Wrong
reference_material: "[full source code pasted here]"
```

## Dependency Ordering

Mark dependencies explicitly. Independent sprints can execute in parallel.

```yaml
sprint_id: 4
dependencies: [1, 3]  # Must complete sprints 1 and 3 first
```

Express the full chain in the plan header:
```
dependency_chain: "1 → 2, 3, 4 → 5 → 7"
```

Where commas indicate parallel sprints and arrows indicate sequential dependencies.

## Output

Write the sprint plan as YAML using [`assets/sprint-template.yaml`](assets/sprint-template.yaml).

After the sprint definitions, append:
- **Execution Protocol** from [`references/execution-protocol.md`](references/execution-protocol.md)
- **Rollback Procedures** from [`references/execution-protocol.md`](references/execution-protocol.md)

See [`references/example.md`](references/example.md) for a complete 43-endpoint migration decomposed into 7 sprints.

## Validation Checklist

- [ ] Every sprint has 3-5 measurable acceptance criteria
- [ ] No criterion uses subjective language
- [ ] All dependencies are explicitly declared
- [ ] No sprint exceeds 4 hours or 6 items
- [ ] Reference materials use file paths, not inline content
- [ ] Dependency chain accounts for all sprints
- [ ] Independent sprints are marked for parallel execution
- [ ] Constraints specify what NOT to do, not just what to do
- [ ] Input and output states are concrete and verifiable
