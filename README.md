# Zudoku Agent Skills

Official Zudoku skills for agents working with the [Zudoku framework](https://zudoku.dev). Zudoku is a framework for building beautiful API documentation sites and developer portals.

## Installation

```bash
npx skills add zuplo/skills
```

Zudoku also supports the [`.well-known` skills discovery standard](https://github.com/cloudflare/agent-skills-discovery-rfc):

```bash
npx skills add https://zudoku.dev/
```

## Included skills

### zudoku

Single comprehensive skill for all Zudoku development. Uses progressive disclosure with reference files covering:

- **Setup & Installation** (`references/create-zudoku.md`): CLI and manual project setup
- **Embedded Docs Lookup** (`references/embedded-docs.md`): Find APIs in `node_modules/zudoku/`
- **Remote Docs Lookup** (`references/remote-docs.md`): Fetch from `https://zudoku.dev/llms.txt`
- **Troubleshooting** (`references/common-errors.md`): Common errors and solutions
- **Migrations** (`references/migration-guide.md`): Version upgrade workflows

Main skill file teaches core concepts and routes to appropriate reference files based on user questions.

## Manual installation

```bash
git clone https://github.com/zuplo/skills.git
```

Then configure your agent to load skills from the cloned directory.

## `.well-known` skills discovery

This repository is served via the [RFC 8615 Well-Known URI](https://github.com/cloudflare/agent-skills-discovery-rfc) at `https://zudoku.dev/.well-known/skills/`.

Agents can discover available skills by fetching:

- **Index**: `https://zudoku.dev/.well-known/skills/index.json`
- **Skills**: `https://zudoku.dev/.well-known/skills/zudoku/SKILL.md`

This enables automatic skill discovery without manual configuration.

## Contributing

Contributions welcome!

1. Fork the repository
2. Make improvements to `SKILL.md` files
3. Test with actual development workflows
4. Submit a pull request

## Resources

- [Zudoku Docs](https://zudoku.dev/docs)
- [Zudoku GitHub](https://github.com/zuplo/zudoku)
- [Agent Skills Spec](https://agentskills.io)
- [`.well-known` Skills RFC](https://github.com/cloudflare/agent-skills-discovery-rfc)
- [Discord](https://discord.zudoku.dev)

## License

Apache-2.0 - See [LICENSE](LICENSE) for details
