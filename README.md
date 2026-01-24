# Agent Skills

A collection of skills for AI coding agents. Skills are packaged instructions that extend agent capabilities with specialized knowledge and workflows.

## Skills

### Conventional Commit

Conventional Commits v1.0.0 standards for git messages. Activates when:
- Creating git commits
- Writing or drafting commit messages
- Reviewing commit message format
- Explaining commit conventions
- Validating commit message compliance

Covers message structure, commit types (`feat`, `fix`, `docs`, `refactor`, etc.), SemVer correlation, body formatting, and breaking change notation.

### Enhance Prompt

Prompt engineering principles for writing efficient LLM instructions. Activates when:
- Writing or improving system prompts
- Creating Claude Code skills or CLAUDE.md files
- Reviewing agent instructions
- General prompt engineering questions

Covers token economics, imperative language, determinism, formatting, emphasis modifiers, and anti-patterns. Applies across LLM platforms (Claude, GPT, Gemini, Llama).

## Installation

```bash
npx add-skill jkappers/agent-skills
```

## Usage

Skills activate automatically based on context. Natural language prompts trigger relevant skills:

- "Commit these changes" → Conventional Commit
- "Review this system prompt" → Enhance Prompt
- "Help me write a CLAUDE.md file" → Enhance Prompt

## Skill Architecture

Each skill contains:

```
skills/
└── skill-name/
    └── SKILL.md         # Agent instructions with frontmatter
```

The `SKILL.md` file includes:
- **Frontmatter**: Name and description (used for activation matching)
- **Instructions**: Domain-specific guidance for the agent

## License

MIT
