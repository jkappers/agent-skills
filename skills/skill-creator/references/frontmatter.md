# Frontmatter Reference

Complete reference for YAML frontmatter fields in SKILL.md files. Universal fields follow the [Agent Skills open standard](https://agentskills.io). Platform extensions are documented separately below.

## Contents

- [Universal Fields (Agent Skills Standard)](#universal-fields-agent-skills-standard)
- [Platform Extensions: Claude Code](#platform-extensions-claude-code)

## Universal Fields (Agent Skills Standard)

| Field | Required | Constraints |
|-------|----------|-------------|
| `name` | Yes | Max 64 chars. Lowercase letters, numbers, hyphens. Must not start/end with hyphen. No consecutive hyphens (`--`). **Must match parent directory name.** |
| `description` | Yes | Non-empty. Max 1024 chars. Describes what the skill does and when to use it. |
| `license` | No | License name or reference to a bundled license file. |
| `compatibility` | No | Max 500 chars. Declare environment requirements (intended platform, system packages, network access). |
| `metadata` | No | Arbitrary key-value map (string keys, string values). Use reasonably unique key names to avoid conflicts. |
| `allowed-tools` | No | Space-delimited list of pre-approved tools. Experimental; support varies across platforms. |

### `name` Rules

- Lowercase letters, numbers, hyphens only
- 1-64 characters
- Must not start or end with `-`
- Must not contain consecutive hyphens (`--`)
- **Must match the parent directory name**
- Do not use reserved words specific to any platform (e.g., "anthropic", "claude")

### `description` Guidelines

- Write in third person
- Include what the skill does AND when to activate
- Enumerate 4-6 specific activation triggers based on user intent
- Template: `[Value proposition]. Use when (1) [trigger], (2) [trigger], ... (N) [trigger].`

### `license`

Specify the license applied to the skill. Keep it short: either the license name or a reference to a bundled license file.

```yaml
license: Apache-2.0
```

```yaml
license: Proprietary. LICENSE.txt has complete terms
```

### `compatibility`

Only include if the skill has specific environment requirements. Most skills do not need this field.

```yaml
compatibility: Requires git, docker, and access to the internet
```

```yaml
compatibility: Designed for Claude Code (or similar products)
```

### `metadata`

Store additional properties not defined by the standard. Use reasonably unique key names to avoid conflicts with other tools.

```yaml
metadata:
  author: example-org
  version: "1.0"
```

### `allowed-tools`

Space-delimited list of tools that are pre-approved to run. Experimental; support and syntax varies across platforms.

```yaml
allowed-tools: Bash(git:*) Bash(jq:*) Read
```

## Platform Extensions: Claude Code

The following fields are specific to Claude Code and extend the universal standard. Other platforms may ignore these fields.

### Invocation Control

| Field | Default | Effect |
|-------|---------|--------|
| `disable-model-invocation` | `false` | `true`: Only user can invoke via `/name`. Description excluded from system prompt. Use for workflows with side effects (deploy, commit). |
| `user-invocable` | `true` | `false`: Only Claude can invoke. Hidden from `/` menu. Use for background knowledge not actionable as a command. |
| `argument-hint` | none | Shown during autocomplete. Example: `[issue-number]`, `[filename] [format]`. |

#### Invocation Matrix

| Configuration | User invokes | Claude invokes | Description in context |
|---------------|-------------|----------------|----------------------|
| Default | Yes | Yes | Yes |
| `disable-model-invocation: true` | Yes | No | No |
| `user-invocable: false` | No | Yes | Yes |

### Execution Context

| Field | Default | Effect |
|-------|---------|--------|
| `context` | (inline) | `fork`: Run in isolated subagent. No conversation history access. Skill content becomes the subagent prompt. |
| `agent` | `general-purpose` | Agent type when `context: fork`. Options: `Explore`, `Plan`, `general-purpose`, or custom from `.claude/agents/`. |
| `model` | (inherited) | Override model for this skill. |

#### When to Use `context: fork`

Use forked context for:
- Read-only research tasks that should not modify conversation state
- Tasks producing large output that would bloat the main context
- Parallel execution of independent subtasks

Do not fork for:
- Tasks requiring conversation history
- Interactive workflows needing user feedback mid-execution
- Tasks that modify files the main conversation tracks

### Security (Claude Code)

| Field | Default | Effect |
|-------|---------|--------|
| `allowed-tools` | (all tools) | Comma-separated list of tools Claude can use without permission. Example: `Read, Grep, Glob` for read-only skills. |

> Note: Claude Code uses comma-separated `allowed-tools` syntax. The universal standard uses space-delimited syntax. Both formats are accepted by Claude Code.

#### Tool Restriction Examples

```yaml
# Read-only research skill
allowed-tools: Read, Grep, Glob, WebSearch, WebFetch

# Git-only skill
allowed-tools: Bash(git *)

# Full filesystem access, no network
allowed-tools: Read, Write, Edit, Glob, Grep, Bash
```

### String Substitutions

| Variable | Description |
|----------|-------------|
| `$ARGUMENTS` | All arguments passed when invoking the skill |
| `$ARGUMENTS[N]` / `$N` | Specific argument by 0-based index |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `` !`command` `` | Execute shell command, replace with output (runs before Claude sees content) |

#### Dynamic Context Example

```yaml
---
name: review-changes
description: Review uncommitted changes against project standards
context: fork
agent: Explore
---
## Current Changes
- Staged diff: !`git diff --cached`
- Unstaged diff: !`git diff`
- Status: !`git status --short`

Review these changes for:
1. Code quality issues
2. Missing tests
3. Breaking changes
```
