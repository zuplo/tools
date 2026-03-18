# AGENTS.md

This file provides context for AI coding assistants (Cursor, GitHub Copilot, Claude Code, etc.) working with the Zudoku Agent Skills repository.

## Project Overview

The **Zudoku Agent Skills** repository provides official agent skills for coding agents working with the [Zudoku framework](https://zudoku.dev). These skills enable AI assistants to write accurate Zudoku code with production-ready patterns validated against the current codebase.

- **Repository**: https://github.com/zuplo/skills
- **Zudoku Framework**: https://github.com/zuplo/zudoku
- **Documentation**: https://zudoku.dev/docs
- **License**: Apache-2.0

## Repository Structure

| Directory         | Description                                                    |
| ----------------- | -------------------------------------------------------------- |
| `skills/`         | Agent skill definitions                                        |
| `skills/zudoku/`  | Single comprehensive Zudoku skill with progressive disclosure  |

## Specification

Fetch the up-to-date [Agent Skills Specification](https://agentskills.io/specification.md) for details on skill structure, frontmatter fields, and best practices.

## Development Guidelines

### Adding New Skills

1. Create directory in `skills/`
2. Add `SKILL.md` with proper frontmatter
3. Include production-ready code patterns
4. Add reference documentation in `references/` if needed
5. Test patterns against current Zudoku codebase
6. Update root `README.md`

### Updating Existing Skills

1. **Validate against current code**: Always verify patterns against the latest Zudoku source code
2. **Use embedded docs**: Check `node_modules/zudoku/` for current APIs and types
3. **Test patterns**: Ensure all code examples are runnable
4. **Update references**: Keep migration guides and troubleshooting current
5. **Version alignment**: Note which Zudoku versions the patterns support

### Code Pattern Requirements

- **Completeness**: Patterns should be copy-paste ready
- **Validation**: All APIs must be verified against current Zudoku codebase
- **TODO placeholders**: Use `// TODO:` for user customization points
- **Imports**: Include all necessary import statements
- **Comments**: Explain non-obvious concepts, not syntax
- **No marketing language**: Technical documentation only
