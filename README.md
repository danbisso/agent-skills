# Full-Stack Skills

A library of [Claude Code](https://code.claude.com) **skills** for full-stack product work — design, UX, frontend engineering, AWS architecture, and code quality. Drop it into any project (or your user-level config) so the same expertise travels across all of them.

These skills encode a specific, opinionated stack and craft so you don't have to re-explain it every session.

## The stack these skills assume

- **Frontend:** React Router 7 (framework mode), Tailwind CSS, TypeScript
- **Data:** Drizzle ORM
- **Edge runtime:** Cloudflare Workers + D1
- **Cloud / backend:** AWS (CDK in TypeScript, serverless-first)
- **Discipline:** behavior-driven TDD, functional TypeScript, DRY / single source of truth

## Skills

| Skill | What it's for |
|---|---|
| [`frontend-design`](skills/frontend-design/SKILL.md) | Translate designs into production frontend — design tokens, Tailwind theming, responsive/fluid layout, pixel fidelity, component architecture. |
| [`designer`](skills/designer/SKILL.md) | Visual & brand design craft (type, color, space, hierarchy) and design review of live sites — yours or references on the web. |
| [`ux`](skills/ux/SKILL.md) | Information architecture, interaction design, usability heuristics, accessibility, and live UX audits. |
| [`frontend-dev`](skills/frontend-dev/SKILL.md) | Build React Router 7 apps in TypeScript — routing, loaders/actions, SSR, and how the framework wires into the data layer and edge runtime below. |
| [`drizzle`](skills/drizzle/SKILL.md) | Data layer — Drizzle ORM schema, relations, queries, type inference, and drizzle-kit migrations (Cloudflare D1 / SQLite focused). |
| [`cloudflare-workers`](skills/cloudflare-workers/SKILL.md) | Edge runtime & platform — Workers execution model, bindings (D1/KV/R2/Queues), `wrangler` config, secrets, local dev, and deploy. |
| [`aws-architect`](skills/aws-architect/SKILL.md) | Choose the right compute and orchestration — Lambda vs Fargate vs containers, Step Functions, Lambda durable, EventBridge, Scheduler. |
| [`aws-cdk`](skills/aws-cdk/SKILL.md) | Author AWS CDK in TypeScript — L1/L2/L3 constructs, patterns, community constructs, testing, and guardrails. |
| [`dynamodb-modeler`](skills/dynamodb-modeler/SKILL.md) | Design and review DynamoDB data models — access-pattern-first design, single/multi-table, key & index design, the building-block patterns, and a gap-finding audit workflow. |
| [`clean-code`](skills/clean-code/SKILL.md) | Refactor toward clean, modular, DRY TypeScript — single source of truth, functional composition, redundancy removal. |
| [`tdd`](skills/tdd/SKILL.md) | Drive code with behavior-first tests and a disciplined red-green-refactor loop — no testing of implementation details. |
| [`performance`](skills/performance/SKILL.md) | Review and fix performance using browser devtools, Core Web Vitals, profiling, and bundle analysis. |
| [`daniel-code-review`](skills/daniel-code-review/SKILL.md) | Review a diff, PR, or plan with Daniel Bissinger's engineering judgment — production truth, loud failure, customer/data impact, and simplicity — with calibrated severity and conversational feedback. |

## How skills relate

```
designer ──┐                                    ┌──► drizzle
ux ────────┼──► frontend-design ──► frontend-dev ┤
           │           │                  │      └──► cloudflare-workers
           └─ review   └─ clean-code ◄─────┴──► tdd ◄──► performance

backend / heavier workloads:  aws-architect ──► aws-cdk
                                    └──► dynamodb-modeler (data model) ──► aws-cdk (provision)

review:  daniel-code-review ◄── clean-code · tdd · performance (the standards it enforces)
```

Each `SKILL.md` cross-links the siblings it most often hands off to.

## Install

Skills live in a `skills/` directory that Claude Code discovers. Pick one:

**Per project** (shared with your team via the repo):
```bash
git clone https://github.com/<you>/fullstack-skills
cp -r fullstack-skills/skills/* your-project/.claude/skills/
```

**User-level** (available in every project on your machine):
```bash
cp -r fullstack-skills/skills/* ~/.claude/skills/
```

**As a submodule / symlink** (stay in sync with upstream):
```bash
git submodule add https://github.com/<you>/fullstack-skills .claude/vendor/fullstack-skills
ln -s ../vendor/fullstack-skills/skills/* .claude/skills/
```

Claude loads a skill's body only when its `description` matches what you're doing, so a large library has near-zero ambient cost.

## Authoring conventions

Every skill is a directory under `skills/` with a `SKILL.md`:

```markdown
---
name: skill-name
description: One line that says what it does AND when to use it. This is the only text Claude sees until the skill fires, so make the triggers concrete.
---

# Body: instructions written TO Claude, in the imperative.
```

- **Description = discovery.** Lead with the capability, then "Use when …" triggers. Concrete verbs and nouns beat adjectives.
- **Body talks to Claude,** not to a human reader — checklists, decision rules, and worked patterns, not marketing.
- **Stay in lane.** A skill should hand off to a sibling rather than duplicate it.
- **Optional supporting files** (checklists, reference tables) sit beside `SKILL.md` and are linked from it.

## License

MIT. Use, fork, and adapt freely.
