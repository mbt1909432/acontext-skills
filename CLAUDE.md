# CLAUDE.md

Guidelines for AI coding agents working on this repository.

## Commit Conventions

Use [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>: <description>
```

**Types:**
- `feat` - New feature
- `fix` - Bug fix
- `docs` - Documentation changes
- `refactor` - Code refactoring
- `test` - Adding or updating tests
- `chore` - Maintenance tasks

**Examples:**
```
docs: add quickstart section to README
feat: add learning spaces integration
fix: correct sandbox tool context handling
refactor: rename skill to acontext-agent-integration
```

**Commit body (optional):**
```
<type>: <description>

- Bullet point 1
- Bullet point 2

Co-Authored-By: Claude Opus 4.6 <noreply@anthropic.com>
```

## Documentation Standards

- All documentation must be written in **English**
- This includes: README, SKILL.md, reference docs, code comments
- Use clear, concise language
- Code examples should be self-contained and runnable

## File Naming

- Use lowercase with hyphens: `session-management.md`
- Skill directories use hyphens: `acontext-agent-integration/`
