# Anti-Patterns

Patterns that degrade skill quality. Check generated skills against this list.

## Contents

- [Language Anti-Patterns](#language-anti-patterns)
- [Structural Anti-Patterns](#structural-anti-patterns)
- [Content Anti-Patterns](#content-anti-patterns)
- [Frontmatter Anti-Patterns](#frontmatter-anti-patterns)
- [Portability Anti-Patterns](#portability-anti-patterns)

## Language Anti-Patterns

### Suggestion Language

Suggestions create ambiguity. The model may or may not follow them.

| Anti-Pattern | Fix |
|-------------|-----|
| "Consider using async/await" | "Use async/await for I/O operations" |
| "You might want to validate input" | "Validate input at API boundaries" |
| "It would be good to add tests" | "Add tests for each public method" |

### Vague Quantifiers

| Anti-Pattern | Fix |
|-------------|-----|
| "Usually validate input" | "Validate input at API boundaries" |
| "Add logging where appropriate" | "Log errors with stack traces. Omit logging for expected control flow." |
| "Keep functions reasonably small" | "Limit API handlers to routing only. Move logic to services/" |

### Ambiguous Conditionals

| Anti-Pattern | Fix |
|-------------|-----|
| "Add error handling when needed" | "Wrap external API calls in try/catch. Let internal errors propagate." |
| "Use caching if it makes sense" | "Cache responses from `/api/products` for 5 minutes." |

### Multiple Options Without Default

| Anti-Pattern | Fix |
|-------------|-----|
| "Use Jest, Vitest, or Mocha" | "Use Vitest for tests. Jest acceptable for legacy files." |
| "Deploy to AWS, GCP, or Azure" | "Deploy to AWS ECS. Override with deployment target argument." |

## Structural Anti-Patterns

### Buried Critical Constraints

Place hard constraints at the beginning of their section, not hidden in the middle of prose.

| Anti-Pattern | Fix |
|-------------|-----|
| Long preamble, then critical rule in paragraph 3 | Critical constraint as first item, context after |

### Over-Emphasis

If everything is bold or MUST, nothing stands out. Reserve emphasis for genuine hard constraints.

| Anti-Pattern | Fix |
|-------------|-----|
| **Every** **other** **word** **bold** | Bold only for failure-causing violations |
| "MUST use lowercase. MUST add tests. MUST use tabs." | "Use lowercase. Add tests. **MUST NOT commit secrets.**" |

### Lists as Default

| Anti-Pattern | Fix |
|-------------|-----|
| Every section is a bulleted list | Tables for structured data, prose for relationships, lists for discrete items |

### Deeply Nested References

| Anti-Pattern | Fix |
|-------------|-----|
| `SKILL.md -> advanced.md -> details.md -> info` | `SKILL.md -> advanced.md` (one level deep) |

## Content Anti-Patterns

### Repeating Model Knowledge

The model already knows framework documentation. Only add project-specific knowledge.

| Anti-Pattern | Fix |
|-------------|-----|
| "React hooks let you use state in function components..." | "Store form state in URL params, not local state" |
| "TypeScript is a typed superset of JavaScript..." | "Enable strict mode in tsconfig.json" |
| "Docker containers provide isolated environments..." | Use the specific Dockerfile pattern directly |

### Generic Best Practices

| Anti-Pattern | Fix |
|-------------|-----|
| "Functions should be small and focused" | "Limit API handlers to routing only. Move logic to services/" |
| "Write clean, maintainable code" | (Delete. Adds no information.) |
| "Follow SOLID principles" | Specify which principle applies and how |

### Time-Sensitive Information

| Anti-Pattern | Fix |
|-------------|-----|
| "Before August 2025, use legacy API" | "Use v2 API at `api.example.com/v2/`" |
| "This is a new feature as of v3.2" | State the current behavior only |

### Decorative Content

| Remove | Reason |
|--------|--------|
| "Welcome to the X skill!" | Consumes tokens, no behavioral effect |
| "Remember to always..." | "Remember" is decorative filler |
| "Please make sure to..." | "Please" and "make sure" are filler |
| "It's important to note that..." | State the fact directly |

### Hypothetical Scenarios

| Anti-Pattern | Fix |
|-------------|-----|
| "If we ever migrate to Postgres..." | Address when actual, not hypothetical |
| "In case someone wants to use GraphQL..." | Cover current technology only |

## Frontmatter Anti-Patterns

### Vague Descriptions

| Anti-Pattern | Fix |
|-------------|-----|
| "A helpful tool for code" | "Generate unit tests for TypeScript modules. Use when (1) adding test coverage, (2) creating tests for new functions, or (3) user requests tests." |
| "Helps with Docker" | "Create optimized multi-stage Dockerfiles for Node.js applications. Use when (1) creating a Dockerfile, (2) containerizing a Node.js app, or (3) user mentions Docker." |

### Missing Activation Triggers

| Anti-Pattern | Fix |
|-------------|-----|
| "Generates Dockerfiles" | "Generates Dockerfiles. Use when (1) creating a new Dockerfile, (2) containerizing an application, (3) optimizing an existing Dockerfile, or (4) user mentions Docker/container." |

### First/Second Person Descriptions

| Anti-Pattern | Fix |
|-------------|-----|
| "I can help you process PDFs" | "Processes PDF files and extracts text" |
| "You can use this to analyze code" | "Analyzes code for quality and standards" |

### Vague Names

| Anti-Pattern | Fix |
|-------------|-----|
| `helper`, `utils`, `tools` | `api-test-generator`, `dockerfile-builder` |
| `documents`, `data` | `pdf-processor`, `csv-analyzer` |

### Windows-Style Paths in Scripts

| Anti-Pattern | Fix |
|-------------|-----|
| `scripts\helper.py` | `scripts/helper.py` |

## Portability Anti-Patterns

### Platform-Specific Fields When Universal Fields Suffice

| Anti-Pattern | Fix |
|-------------|-----|
| Using `argument-hint` as the only way to describe inputs | Describe expected inputs in the `description` field and skill body. Add `argument-hint` only as a platform enhancement. |
| Relying on `disable-model-invocation` for safety | Document invocation constraints in the skill body so all platforms respect them. |

### Platform-Specific Tool Names

| Anti-Pattern | Fix |
|-------------|-----|
| "Use the `Bash` tool to run commands" | "Run the following command:" (let the agent choose its execution method) |
| "Read the file using `Read`" | "Read the file:" (tool names differ across platforms) |
| "Use `WebSearch` to find..." | "Search for..." (describe the action, not the tool) |

### Hardcoded Platform Paths

| Anti-Pattern | Fix |
|-------------|-----|
| `~/.claude/settings.json` | Use relative paths from skill root, or describe the target generically |
| Absolute paths to platform directories | Relative paths from the skill directory: `scripts/validate.py`, `references/guide.md` |
