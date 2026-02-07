# Skill Patterns

Common patterns for structuring skill content. Select based on task type.

## Contents

- [Feedback Loop Pattern](#feedback-loop-pattern)
- [Checklist Tracking Pattern](#checklist-tracking-pattern)
- [Progressive Disclosure Pattern](#progressive-disclosure-pattern)
- [Conditional Deep-Dive Pattern](#conditional-deep-dive-pattern)
- [Scoring/Assessment Pattern](#scoringassessment-pattern)
- [Code Generation Pattern](#code-generation-pattern)
- [Reference Skill Pattern](#reference-skill-pattern)
- [Complete Skill Example](#complete-skill-example)

## Feedback Loop Pattern

Use when output quality requires iterative validation.

```markdown
## Validation Loop
1. Generate output following the template
2. Run validation: `python scripts/validate.py output/`
3. If validation fails, fix errors and return to step 2
4. Proceed only when validation passes
```

Variant without scripts:

```markdown
## Review Loop
1. Draft content following STYLE_GUIDE.md
2. Check against requirements checklist
3. If issues found, revise and re-check
4. Finalize when all requirements met
```

## Checklist Tracking Pattern

Use for complex multi-step workflows where progress tracking aids reliability.

````markdown
## Workflow

Copy this checklist and update as you progress:

```
Progress:
- [ ] Step 1: Analyze input
- [ ] Step 2: Generate plan
- [ ] Step 3: Execute plan
- [ ] Step 4: Validate output
- [ ] Step 5: Report results
```
````

## Progressive Disclosure Pattern

Use when the skill covers a broad domain with specialized sub-topics.

**Directory structure:**
```
skill-name/
├── SKILL.md              # Overview, quick start, navigation
├── references/
│   ├── domain-a.md       # Deep-dive for domain A
│   ├── domain-b.md       # Deep-dive for domain B
│   └── examples.md       # Extended examples
└── scripts/
    └── helper.py         # Utility scripts
```

**SKILL.md navigation:**
```markdown
## Quick Start
[Core instructions for the most common use case]

## Advanced Features
- **Domain A**: See [references/domain-a.md](references/domain-a.md)
- **Domain B**: See [references/domain-b.md](references/domain-b.md)
- **Extended examples**: See [references/examples.md](references/examples.md)
```

## Conditional Deep-Dive Pattern

Use when different inputs require different detailed workflows.

```markdown
## Deep-Dive Triggers

| Condition | Action | Reference |
|-----------|--------|-----------|
| Score < 4 on dimension X | Apply framework A | [references/framework-a.md](references/framework-a.md) |
| Input type is Y | Follow specialized workflow | [references/workflow-y.md](references/workflow-y.md) |
| Risk level is high | Run extended validation | [references/validation.md](references/validation.md) |
```

## Scoring/Assessment Pattern

Use for evaluation skills that produce structured ratings.

```markdown
## Assessment

Score each dimension 1-5. Require evidence for each score.

| Dimension | Question | Evidence Sources |
|-----------|----------|------------------|
| Quality | Does it meet standards? | Tests, reviews, metrics |
| Impact | How significant is the change? | User count, revenue, risk |
| Effort | How much work is required? | Story points, dependencies |

### Scoring Scale

| Score | Criteria |
|-------|----------|
| 5 | Strong evidence, low risk |
| 4 | Solid evidence, minor concerns |
| 3 | Mixed signals, validation needed |
| 2 | Weak evidence, significant concerns |
| 1 | Red flags, likely blockers |
```

## Code Generation Pattern

Use for skills that produce code artifacts.

````markdown
## Generation Workflow

1. Detect project configuration from `package.json` / `tsconfig.json`
2. Select pattern from Pattern Guide table
3. Generate code using selected pattern
4. Add to `.gitignore` if new build artifacts created

## Pattern Guide

| Scenario | Pattern | Base Template |
|----------|---------|---------------|
| REST API | Express handler | See Standard Pattern |
| GraphQL | Resolver | See GraphQL Pattern |
| CLI tool | Commander setup | See CLI Pattern |

## Standard Pattern

```typescript
import express from 'express'

const router = express.Router()

router.post('/endpoint', async (req, res) => {
  const validated = schema.parse(req.body)
  const result = await service.process(validated)
  res.status(201).json(result)
})
```

## Verification Checklist
- [ ] Types exported from `types/` directory
- [ ] Error handling follows project conventions
- [ ] Tests added in `__tests__/` directory
````

## Reference Skill Pattern

Use for skills that provide domain knowledge without a procedural workflow.

```markdown
## API Conventions

### Naming
- Endpoints: plural nouns (`/users`, `/orders`)
- Actions: verb prefix (`/users/activate`)
- Query params: camelCase

### Response Format
All responses follow:

| Field | Type | Required |
|-------|------|----------|
| `data` | object/array | Yes |
| `error` | object | On error only |
| `meta` | object | For paginated responses |

### Error Codes
| Code | Meaning | HTTP Status |
|------|---------|-------------|
| `VALIDATION_ERROR` | Invalid input | 400 |
| `NOT_FOUND` | Resource missing | 404 |
| `CONFLICT` | Duplicate resource | 409 |
```

## Complete Skill Example

A fully realized skill using universal frontmatter fields only:

````yaml
---
name: api-test-generator
description: Generate API integration tests from endpoint definitions. Use when (1) creating tests for REST API endpoints, (2) adding test coverage for new routes, (3) user asks to test an API, or (4) reviewing API test completeness.
---

# API Test Generator

Generate integration tests for REST API endpoints. Detect test framework from project configuration. Default to Vitest.

## Workflow

1. Read the target endpoint file or path
2. Detect test framework from `package.json` (Vitest > Jest > Mocha)
3. Identify all route handlers and their HTTP methods
4. Generate test file with cases for each handler
5. Validate test file runs without syntax errors

## Test Structure

Each endpoint test includes:

| Category | Test Cases |
|----------|-----------|
| Happy path | Valid input returns expected status and body |
| Validation | Missing/invalid fields return 400 with error details |
| Auth | Unauthenticated returns 401, unauthorized returns 403 |
| Edge cases | Empty body, max length, special characters |

## Output Format

```typescript
import { describe, it, expect } from 'vitest'
import request from 'supertest'
import { app } from '../src/app'

describe('[METHOD] [PATH]', () => {
  it('returns [STATUS] for valid input', async () => {
    const res = await request(app)
      .[method]('[path]')
      .send([validBody])
    expect(res.status).toBe([expectedStatus])
    expect(res.body).toMatchObject([expectedShape])
  })

  it('returns 400 for invalid input', async () => {
    const res = await request(app)
      .[method]('[path]')
      .send([invalidBody])
    expect(res.status).toBe(400)
    expect(res.body.error).toBeDefined()
  })
})
```

## Verification Checklist
- [ ] All route handlers have corresponding test cases
- [ ] Happy path, validation, and auth categories covered
- [ ] Test file runs without errors
- [ ] No hardcoded test data that duplicates source constants
````

> **With Claude Code extensions**: Add `argument-hint: "[endpoint path or file]"` to frontmatter for slash command autocomplete, and use `$ARGUMENTS` in the workflow to reference the passed argument (e.g., `1. Read endpoint file or path: $ARGUMENTS`).
