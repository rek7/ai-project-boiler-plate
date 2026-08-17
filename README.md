# ai-project-boiler-plate

Engineering directives for TypeScript projects built with AI agents.

This repo holds the rules, not the code. Copy them into a new project so the humans and
the agents working on it start from the same definition of done, instead of rediscovering
it in review three months in.

## Files

| File | Purpose |
| --- | --- |
| [`AGENTS.md`](AGENTS.md) | The directives. Canonical, and the only copy. |
| `CLAUDE.md` | Symlink to `AGENTS.md`. Claude Code reads this name. |
| `README.md` | This file. |

Two instruction files with the same content drift within a month, and then agents follow
whichever one is stale. One file, one symlink.

## Use it

```bash
cd your-new-project
curl -O https://raw.githubusercontent.com/rek7/ai-project-boiler-plate/main/AGENTS.md
ln -s AGENTS.md CLAUDE.md
```

Then write `docs/spec.md` before writing code.

## Two files, two jobs

`AGENTS.md` is what is true across every project: the gates, the boundaries, the
directives. It is stable and it is not a recipe. Where it names a tool, that is a default
you can swap.

`docs/spec.md` is what is true about this one project: what it does, the stack actually in
use, the shape of the system, the data model, the public surfaces, the decisions in force,
and what was deliberately left unbuilt. It is the first file created in a new project and
it gets updated in the same change that changes the system. Written in present tense,
describing what exists now. Git holds the history; the spec holds the current answer.

Project-specific rules go in the spec, never in `AGENTS.md`.

## What the directives cover

- **The Loop.** Lint, typecheck, tests, drift checks after every change. Fix and re-run
  until clean. The rule the others exist to support.
- **Structure and duplication.** What belongs in a shared package, when code earns the
  move, one definition per concept, rule of three for extraction.
- **TypeScript.** Strict, no `any`, no ignores, types derived from schemas.
- **Validation.** Parse at every boundary: requests, env, third-party responses, webhooks,
  queue messages, database JSON. Shape what you return.
- **API contract.** One source of truth generating types, validation, client, handlers,
  and a committed `openapi.json`. Docs render from the spec, never hand-written.
- **Five test layers.** Unit, handler, integration, end-to-end, and post-deploy smoke,
  with the job each one does and the rules that keep them from going flaky.
- **Interface.** One component library, one forms approach, one fetching approach, all
  five states handled, phone width and accessibility as gates rather than intentions.
- **Copy.** No slop, no em dashes, and an AST-based test that makes it a build gate.
- **Data.** Reviewed migrations, two-step destructive changes, constraints in the
  database, transactional audit records, backfills as committed code.
- **Security.** Explicit authorization per route proven by tests, secret handling, rate
  limits, headers, errors that do not leak internals.
- **Auth and access.** Never hand-rolled. Sessions and scoped API keys from the first
  release, both resolving to one authorization decision, so MCP servers, CI, and
  integrations are ordinary clients rather than side doors.
- **Adopt before you build.** Reach for the framework's answer, then a maintained
  library, then your own code, and write down which one you picked and why.
- **Operations.** Structured logs, request IDs, a health endpoint, and backups that are
  off-host, checksummed, replicated, and actually restore-drilled.
- **Discovery surfaces.** `sitemap.xml`, `robots.txt`, `llms.txt`, `llms-full.txt`, feeds,
  manifest, metadata, and structured data, all generated from live content and tested to
  prove private records never reach an index.
- **Automation.** Fast pre-commit, full gates on pre-push, the same scripts in CI, pinned
  actions, least privilege, deploys that can roll back.
- **Agent configuration.** Canonical instructions, a committed permission allowlist, and
  repeated procedures turned into reviewed skills instead of pasted prompts.
- **A Definition of Done** you can paste into a pull request template.

## Why

Standards that live in someone's head do not survive a context window. An agent follows a
written rule and cannot follow an unwritten preference, so anything that matters gets
written down once, in the file the agent already reads.
