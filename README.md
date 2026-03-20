# Civic Agent Skills

A collection of skills for AI coding agents (Claude Code, etc.) that provide guidance on integrating with [Civic](https://www.civic.com) products.

## Available Skills

| Skill | Description |
|-------|-------------|
| [developer-guide-nextjs-authjs-vercelai](./developer-guide-nextjs-authjs-vercelai/) | Build a Next.js app with Auth.js authentication, Civic MCP tool calling, and Vercel AI SDK |

## Installation

Install a specific skill using the [Skills CLI](https://skills.sh/):

```bash
npx skills add civicteam/agent-skills -s developer-guide-nextjs-authjs-vercelai
```

Or install all skills:

```bash
npx skills add civicteam/agent-skills --all
```

## What are Skills?

Skills are modular knowledge packages that give AI coding agents specialized context about tools, frameworks, and integration patterns. They contain best practices, gotchas, and reference links so the agent can build correct integrations without trial and error.

## Contributing

To add a new skill, create a directory with a `SKILL.md` file following the format of existing skills.
