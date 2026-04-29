---
name: super-automation
description: "Workflow and app automations across SaaS tools, APIs, and integrations."
---

# Super Automation

## Overview
Design reliable automations with clear triggers, data flow, and monitoring.


## User Intent Examples
- "Need help with GitHub Automation for my product/site."
- "Create a plan for Workflow Automation."
- "Not sure where to start, need a quick assessment."

## Workflow
1. Define the goal, trigger, and success criteria.
2. Map required systems and permissions.
3. Design workflow steps and error handling.
4. Implement integrations and test with sample data.
5. Add monitoring, alerts, and audit logs.
6. Document ownership and maintenance steps.

## Minimal Intake Questions
- Primary goal or outcome
- Scope (pages, systems, teams, or timeframe)
- Constraints (tools, budget, timeline)

## Output Format
- Workflow diagram and trigger spec
- Integration and permissions checklist
- Error handling and retry strategy
- Test results and validation steps
- Monitoring and maintenance plan

## Routing Map (Modules)
- **GitHub Automation** -> `references/modules/github-automation.md`
- **Workflow Automation** -> `references/modules/workflow-automation.md`

## Bundled References
- `references/modules/`
- `references/toolkit/`
- `scripts/`
- `assets/`
- `agents/`

## Compatibility Notes
- If any module references slash commands or tool-specific legacy paths, translate them into plain-language steps.
- Keep outputs platform-agnostic unless the user specifies a specific tool, stack, or agent.

## Guardrails
- Do not store secrets in plain text.
- Use idempotent actions where possible.
- Log failures with enough context to debug.
