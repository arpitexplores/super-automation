# Super Automation

<!-- super-series-intro -->
> The go-to AI vibe coding skill for automation: SaaS workflows, API integrations, triggers, permissions, retries, monitoring, and GitHub automation.

[View the full SUPER Skills catalogue](https://github.com/arpitexplores/skills-super).
<!-- /super-series-intro -->

Design reliable automations across SaaS tools, APIs, triggers, permissions, retries, and monitoring.

## Why SUPER Skills For AI Vibe Coding

Super Automation turns AI-assisted coding into reliable workflow design. Use it to map triggers, connect SaaS tools and APIs, define permissions, handle retries, validate data flow, and monitor automations instead of building brittle glue scripts.

These skills are designed for **AI vibe coding**: fast, agent-assisted building where the AI needs strong domain context, practical workflows, guardrails, and implementation-ready outputs.

**SEO and discovery keywords:** workflow automation, SaaS automation, API integrations, GitHub automation, business process automation, no-code automation, agentic automation, AI vibe coding, agent skills.

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

## SUPER Skills Series

This repository is part of the **SUPER Skills** series: standalone, installable agent skills that can be used independently or together.

### Published Series Repositories

| Repository | Purpose |
| --- | --- |
| [skills-super](https://github.com/arpitexplores/skills-super) | Master catalogue for the full SUPER Skills collection: AI vibe coding skills, agent workflows, and installable Markdown skills. |
| [super-seo-growth](https://github.com/arpitexplores/super-seo-growth) | AI SEO, GEO, LLM visibility, content optimisation, programmatic SEO, and citation readiness. |
| [super-seo-foundation](https://github.com/arpitexplores/super-seo-foundation) | Technical SEO, SEO audits, crawlability, indexing, schema, sitemaps, hreflang, Core Web Vitals, and Google tooling. |
| [super-marketing-execution](https://github.com/arpitexplores/super-marketing-execution) | Campaign orchestration, CRO, copywriting, analytics, email, social, paid ads, and growth execution. |
| [super-design-core](https://github.com/arpitexplores/super-design-core) | UI/UX, product design, design systems, frontend UI patterns, IA, flows, and visual systems. |
| [super-ai-ml-agents](https://github.com/arpitexplores/super-ai-ml-agents) | AI agents, agent architecture, tool use, memory, orchestration, multi-agent systems, and guardrails. |

Start with the skill that matches the task. Use the catalogue when you want to browse the full collection or install multiple skills.

### Additional SUPER Skills In The Catalogue

The full catalogue also includes AI/ML foundation, AI/ML ops, automation, cloud, data analytics, design quality, engineering/DevOps, gaming/3D/media, healthcare/wellness, industry ops, legal/HR/compliance, marketing strategy, office/docs/presentation, product/business/finance, security, and specialised platform SDK skills.


## Version

See `VERSION` and `CHANGELOG.md`.

## Licence

MIT. See the root repository `LICENSE`.
