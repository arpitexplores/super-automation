# Super Automation

Design reliable automations across SaaS tools, APIs, triggers, permissions, retries, and monitoring.

## Install

Copy this folder into your agent's skills directory, then restart or reload the agent.

```bash
cp -R super-automation ~/.your-agent/skills/
```

Use it by name:

```text
Use $super-automation to help with this request.
```

## Best For

- workflow automation
- SaaS/API integrations
- GitHub automation
- trigger and data-flow design
- error handling and retries

## Outputs

- workflow diagram
- integration checklist
- permissions plan
- retry and monitoring strategy
- validation steps

## Modules

| Module | Purpose |
| --- | --- |
| `github-automation.md` | GitHub workflow automation, issue/PR flows, repository operations, and CI-adjacent tasks |
| `workflow-automation.md` | General SaaS/API workflow design, triggers, data mapping, retries, and monitoring |

## Example Prompts

- `Use $super-automation to design an approval workflow between these tools.`
- `Use $super-automation to automate this GitHub issue triage flow.`
- `Use $super-automation to review this workflow for failure modes.`

## Package Contents

- `SKILL.md` is the installable skill entry point.
- `references/modules/` contains detailed workflows loaded only when needed.
- `agents/` contains optional agent metadata where supported.
- `scripts/` and `assets/` are optional helpers when bundled.

## Compatibility

This skill is plain Markdown and is intended to be agent-agnostic. If a bundled helper mentions a specific tool path, translate that instruction to the equivalent path for your environment.

## Version

See `VERSION` and `CHANGELOG.md`.

## Licence

MIT. See the root repository `LICENSE`.
