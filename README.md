# Civic Agent Skills

Skills for AI coding agents to integrate with [Civic](https://www.civic.com) products.

## Available Skills

| Skill | Description |
|-------|-------------|
| `developer-guide-nextjs-authjs-vercelai` | Next.js + Auth.js + Civic MCP + Vercel AI SDK |

## How to Use

### Using npx skills

```bash
npx skills add civicteam/agent-skills -s <skill-name>
```

### Manual install

```bash
git clone https://github.com/civicteam/agent-skills.git /tmp/civic-skills
cp -r /tmp/civic-skills/<skill-name> ~/.claude/skills/
```

### Claude Code plugins

```
/plugin marketplace add civicteam/agent-skills
/plugin install <skill-name>
```
