# ai-project-boiler-plate

Engineering standards for TypeScript projects built with AI agents.

This repo holds the rules, not the code. Copy them into a new project so both the humans
and the agents working on it start from the same definition of done, instead of
rediscovering it in review three months in.

## Files

| File | Purpose |
| --- | --- |
| [`AGENTS.md`](AGENTS.md) | The standards. Canonical, and the only copy. |
| `CLAUDE.md` | Symlink to `AGENTS.md`. Claude Code reads this name. |
| `README.md` | This file. |

Two instruction files with the same content drift within a month and then agents follow
whichever one is stale. One file, one symlink.

## Use it

```bash
cd your-new-project
curl -O https://raw.githubusercontent.com/rek7/ai-project-boiler-plate/main/AGENTS.md
ln -s AGENTS.md CLAUDE.md
```

Then replace `<project>` and `@scope`, drop the sections that do not apply, and add
app-specific rules in `apps/<app>/AGENTS.md` rather than editing the root file.

## What it covers

- **The Loop.** Lint, typecheck, unit tests, and OpenAPI drift check after every change.
  Fix and re-run until clean. This is the rule the other twenty exist to support.
- **Repo shape.** Monorepo boundaries, what belongs in a shared package and what does
  not, and the rule that keeps product concepts out of `packages/`.
- **TypeScript.** Strict everywhere, no `any`, no `@ts-ignore`, explicit return types.
- **Zod as the source of truth.** One schema per concept. Parse at every boundary:
  requests, env, third-party responses, webhooks, database JSON.
- **API contract and OpenAPI.** ts-rest contracts built from Zod generate `openapi.json`,
  which is committed and diffed. Docs render from that file, never hand-written.
- **Five test layers.** Unit, handler, integration, E2E, and post-deploy smoke, with the
  job each one does and the rules that keep them from going flaky.
- **One component library.** shadcn in a shared `@scope/ui`, the rule of three for
  extraction, and one stack each for forms and data fetching.
- **Responsive and accessible as a gate.** 320px, hostile content, axe in E2E.
- **Copy quality.** Every user-visible string through `no-ai-slop`. No em dashes.
- **Database.** Migrations only, two-step destructive changes, transactional audit rows.
- **Security.** Explicit authorization per route proven by tests, output shaping, secret
  handling, rate limits, headers.
- **Observability.** Structured logs, request IDs, a health endpoint the smoke test reads.
- **Discovery surfaces.** `sitemap.xml`, `robots.txt`, `llms.txt`, `llms-full.txt`,
  feeds, manifest, metadata, and JSON-LD, all generated from live content, all tested to
  prove drafts and private records never leak into an index.
- **Git hooks.** Fast pre-commit, commitlint, and a pre-push that runs the same gates CI
  runs, from the same script definitions.
- **Docs that stay true.** A small fixed set: architecture, data model, runbook,
  onboarding, security, testing, and decision records.
- **A Definition of Done checklist** you can paste into a pull request template.

## Why

Standards that live in someone's head do not survive a context window. An agent will
follow a written rule and will not follow an unwritten preference, so anything that
matters gets written down once, in the file the agent already reads.
