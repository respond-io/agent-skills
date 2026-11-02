# respond.io Agent Skills

[Agent Skills](https://agentskills.io) that teach AI agents to work with [respond.io](https://respond.io) through the respond.io MCP server.

| Skill | What it does |
| ----- | ------------ |
| [`workflow-authoring`](skills/workflow-authoring/SKILL.md) | Build, edit, test, publish, stop and run respond.io Workflows safely. |

Skills are instructions only. The agent also needs the respond.io MCP server connected to call the tools they describe.

## 1. Connect the respond.io MCP server

Add this server URL as a remote (HTTP) MCP server in your agent:

```
https://mcp.respond.io/mcp
```

On first use you sign in to respond.io and pick the workspace the agent may access.

## 2. Install a skill

### Any agent that reads local skills (Claude Code, Cursor, Codex and others)

```bash
npx skills add respond-io/agent-skills --skill workflow-authoring
```

The CLI asks which agents to install into. Add `-g` to install for your user instead of the current project.

### Claude Code plugin (skills and MCP server in one install)

```
/plugin marketplace add respond-io/agent-skills
/plugin install respond-io@respond-io
```

### Claude.ai, Claude Desktop and other apps with skill upload

Download the skill folder (for example `skills/workflow-authoring/`), zip it with the folder at the top level, and upload the zip where your app accepts custom skills.

### Clients that support the MCP Skills extension

Nothing to install: the respond.io MCP server serves its skills directly to the client.

## Contributing

This repository is published automatically from respond.io's MCP server source, where the skills are tested against the tools they describe. Changes made here directly to `skills/` are overwritten on the next release. To report a problem or suggest a change, open an issue.
