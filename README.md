# Krosyn agent skills

[Krosyn](https://krosyn.com) automates the repetitive work between your apps. Easy for anyone
and fairly priced.

This repo holds an [agent skill](https://skills.sh) that teaches your AI assistant to build,
test and fix Krosyn connectors for you through the Krosyn MCP server: Claude Code, Cursor,
Codex, and any other agent that supports skills.

## Install

```bash
npx skills add Chmura-Software/krosyn-skills
```

Then connect your assistant to Krosyn's MCP server at `https://krosyn.com/api/v1/mcp`.
Setup for each client: [krosyn.com/docs/mcp-integration](https://krosyn.com/docs/mcp-integration).

## What it does

With the skill installed, ask in plain words:

- "When a Shopify order comes in, post it to our #orders Slack channel."
- "Why did my invoice connector fail last night?"
- "Add the customer's company name from our CRM to the HubSpot request."

The assistant plans the connector, builds it as a draft, tests it with a realistic example,
and publishes it when you are happy with the result.

## Contents

| Skill | Path |
|---|---|
| `krosyn` | [skills/krosyn/SKILL.md](skills/krosyn/SKILL.md) |

The expression reference lives in [skills/krosyn/references/expressions.md](skills/krosyn/references/expressions.md).

## License

MIT. Krosyn is a product of [Chmura Software](https://chmurasoftware.com).
