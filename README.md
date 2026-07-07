# tctc-skills

Agent skill for **TCTC / ERC-7303** — on-chain permission control for AI
agents without a permission server. Roles are tokens: grant = mint,
revoke = burn.

## Install

```bash
npx skills add kofujimura/tctc-skills
```

This installs the `tctc` skill into your agent (Claude Code and other
SKILL.md-compatible agents). The skill teaches the agent to:

- check its own on-chain permissions before acting (via the
  [tctc-mcp](https://github.com/kofujimura/tctc-mcp) MCP server),
- guide the one-time setup of tctc-mcp (`npx -y tctc-mcp`),
- follow safe delegation rules (never self-grant, never put keys in configs),
- gate smart-contract functions with ERC-7303 control tokens.

## Contents

- [SKILL.md](SKILL.md) — the skill itself
- [templates/config.template.json](templates/config.template.json) —
  tctc-mcp config starting point

## See also

- 60-second demo: https://www.youtube.com/watch?v=o547bwYT32A
- ERC-7303: https://eips.ethereum.org/EIPS/eip-7303
- Concept & white paper: https://github.com/kofujimura/TCTC

## License

MIT
