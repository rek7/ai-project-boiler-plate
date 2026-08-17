# Engineering Standards

> Directives for every human and AI agent working in this repo.
> `CLAUDE.md` is a symlink to this file. Edit this one.

Each rule below states what has to be true and why. How you satisfy it is a decision for
the project, and once made, that decision goes in `docs/spec.md`.

This file is deliberately not a recipe. A recipe goes stale the first time a project needs
something slightly different, and then agents follow it anyway. If a directive here is
wrong for what you are building, write down what you did instead and why. Silently
ignoring it is the only wrong answer.

---

## 1. docs/spec.md

**The first file you create in a new project, and the one you keep true.**

`docs/spec.md` is the current answer to "what is this and how does it work". Not the plan,
not the history, not the pitch. What exists right now.

It holds:

- What the project does and who uses it
- The stack actually in use, and anything chosen against the defaults in section 3
- The shape of the system: services, packages, boundaries, what talks to what
- The data model and the invariants the schema cannot express on its own
- Public surfaces: routes, APIs, jobs, integrations
- Decisions in force, each with the reason and what it rules out
- What is deliberately not built, so nobody rediscovers a closed question

Rules:

- Create it in the first commit. A project without one starts accumulating undocumented
  decisions immediately, and they are much harder to recover than to record.
- Update it in the same change that changes the shape of the system. Not afterwards, not
  in a cleanup task.
- Write the current state, in present tense. Delete what stopped being true rather than
  appending a correction. This is a reference, not a changelog; Git holds the history.
- Keep it short enough that keeping it accurate is realistic. If it is growing past what
  someone will read, split the detail out and link to it.
- When it disagrees with the code, the code is right and the spec is a bug.

Every other document is optional and has to earn its place. Add `docs/runbook.md` the
first time someone performs an operational procedure, `docs/decisions/` when a choice
needs more room than a line in the spec. Six accurate pages beat forty stale ones.

---

## 2. The Loop

Every task, no exceptions.

1. Write or change the code.
2. Write or update tests for what you touched, in the same change.
3. Run the gates: lint, typecheck, unit tests, and any generated-artifact drift check.
4. If a gate fails, fix the root cause and run again. Do not skip, suppress, or defer.
5. Repeat until clean.
6. Update `docs/spec.md` if the shape changed.

Run the build when the change touches runtime behavior, package exports, env wiring, or
deployment. Run the browser tests when it touches a user flow.

A task with failing lint, type errors, or broken tests is unfinished. Reporting it as done
is the one unrecoverable mistake in this file.

---

## 3. Defaults

Starting points, not requirements. Swap any of them for a good reason, keep the role
filled, and record the swap in `docs/spec.md`.

| Role | Default |
| --- | --- |
| Language | TypeScript, strict |
| Runtime validation | Zod |
| Package manager and tasks | pnpm workspaces, Turborepo |
| Web framework | Next.js App Router |
| API contract | ts-rest, contracts built from the validation schemas |
| Auth | Better Auth for sessions, plus first-party hashed API keys |
| Database | PostgreSQL via Prisma |
| Component library | shadcn/ui in a shared package |
| Tests | Vitest for unit and integration, Playwright for browser |
| Lint and format | ESLint type-checked config, Prettier |
| Runtime and environments | Docker, one Compose file per environment |
| Logging | structured JSON |

The gates in section 2 and the directives below survive any swap. The table does not.

---

## 4. Structure and duplication

- Shared packages hold runtime-neutral code. Product concepts live in the app that owns
  them. A package named after a feature is a boundary violation waiting to spread.
- Code becomes shared after a second runtime actually needs it, not in anticipation.
- Dependencies point one direction. An app depends on a package; a package never depends
  on an app.
- Wrap external clients in one module each. Nothing else imports the vendor SDK directly,
  so replacing it is a contained change.
- One definition per concept. Field lists, enums, status values, error codes, and copy
  strings each exist once and are imported everywhere else.
- The second copy of a pattern is a warning; the third is a refactor. If you are about to
  paste more than a few lines, extract instead.
- Before writing anything new, look for what already exists: shared package first, then
  the app's common components, then the feature folder. Extend what you find.

---

## 5. TypeScript

- Strict mode everywhere, including `noUncheckedIndexedAccess`.
- No `any`. Use `unknown`, a generic, a type guard, or a schema-inferred type.
- No `@ts-ignore`. `@ts-expect-error` only with a comment saying what would remove it.
- Assertions are a last resort, after narrowing, type guards, and parsing. `as const` and
  `satisfies` are fine.
- Explicit return types on everything exported.
- Derive types from schemas rather than declaring them alongside. Two definitions of the
  same shape will disagree eventually, and the compiler will not tell you which is right.
- Model impossible states out of existence. A discriminated union beats five optional
  fields and a comment explaining which combinations are real.

---

## 6. Validation

Parse at every boundary where data enters the process, and let the parsed type flow
inward. Inside the boundary, trust your types; at the boundary, trust nothing.

Boundaries include HTTP requests, environment variables, third-party API responses,
webhook payloads after signature checks, queue messages, files, and database JSON columns.

- Environment config is parsed once at startup and the process refuses to boot if it is
  invalid, listing every problem at once. Nothing else reads raw environment variables.
- Shape what you return. Never hand back a raw database row, or a column added next
  quarter becomes a leak nobody reviewed.
- Reject unexpected input rather than ignoring it.
- Validation errors report every failing field, not the first one.

---

## 7. API contract, spec, and docs

The contract is the source of truth. Types, runtime validation, the client, the server
handlers, and the published spec all derive from it.

- Every route is in the contract. A handler outside it is invisible to consumers and to
  the spec, so a test should fail on one.
- Every endpoint declares its inputs, every response status it can return, and a
  description worth reading.
- Errors use one shape across the whole API. A route that invents its own makes every
  client write a special case.
- `openapi.json` is generated, never hand-written, and committed so an API change shows
  up as a reviewable diff. If it can drift from the contract, a check for drift belongs
  in the gates.
- Human docs render from that spec. Documentation maintained separately from the contract
  is documentation that lies.
- Version the API in the path from the first release. Changing a released endpoint is
  adding a new version, not editing the old one.

---

## 8. Testing

Cover each layer for what only that layer can catch, and do not use one to fake another.

- **Unit**: logic, in isolation. Happy path, every error branch, and the boring edges:
  empty, one, many, null, very long, boundary values.
- **Handler**: every route called directly with mocked dependencies. At minimum, rejected
  without credentials, rejected with the wrong role, the happy path, invalid input, and
  each error status the contract declares. This layer is why authorization bugs do not
  reach production.
- **Integration**: against a real database from Compose (section 14) with migrations
  applied. Transactions, constraints, cascades, and the queries unit tests mock away.
  Never against a shared or remote database.
- **End to end**: critical user journeys in a browser, against a real build. Sign up,
  sign in, the thing the product exists to do, payment, one admin flow.
- **Smoke**: against the deployed URL after every release, fast, read-only, and safe to
  run against production. Health, the spec endpoint, key pages, one authenticated action.
  A failure rolls the deploy back.

Across all of them:

- Tests ship with the code, not as a follow-up task.
- A bug fix starts with a failing test that reproduces it.
- Test behavior through the public interface. Asserting on internals turns every refactor
  into a test rewrite.
- Deterministic or deleted. Control time, seed randomness, wait on conditions rather than
  sleeping. A flaky test teaches the team to ignore red, which costs more than the test
  is worth.
- Real assertions. Checking that something is defined is not a test.
- Coverage is a smoke alarm, not a target. The review question is what breaks silently if
  a line is wrong.

---

## 9. Interface

- One component library per project, used for everything. Mixing two produces two of
  every primitive, two focus styles, and a design that never converges.
- Customize primitives in the shared package, in place. Do not wrap a wrapper.
- Design tokens live in one place. No hardcoded colors or spacing in an app.
- One approach to forms and one to data fetching, used everywhere. A hand-rolled form in
  one corner of the app is a bug report waiting to happen.
- Every interactive surface handles idle, loading, empty, error, and success. An empty
  state that says nothing and an error that swallows its cause are both defects.
- Works at phone width. Not as a later pass: a component that can exceed the viewport
  owns its own wrapping or scrolling, and never relies on a parent to hide overflow.
- Test with hostile content, because real content is hostile: long unbroken strings,
  missing values, the longest realistic value for every field. Truncate only when the
  full value stays reachable.
- Keyboard reachable, labeled, visible focus, sufficient contrast, semantic HTML before
  ARIA. Automate the check in the browser tests so it is a gate and not an intention.
- Respect reduced-motion and color-scheme preferences.

---

## 10. Copy

Every user-visible string is product surface: pages, forms, helper text, empty states,
errors, emails, docs, metadata, and machine-readable endpoints.

- Concrete facts, direct verbs, the shortest version that still helps the reader act.
- Cut throat-clearing, filler that would fit any product, repeated claims, fake contrasts,
  inflated importance, and decorative formatting.
- No em dashes.
- Preserve factual, legal, security, and operational meaning. Never invent a claim, never
  compress away a limit or a required instruction.
- Errors say what happened and what to do next.
- Run new or materially edited copy through the `no-ai-slop` skill
  (<https://github.com/petergyang/no-ai-slop>) before the change is done.

Mechanize it, because a style rule nobody can check decays. A test that walks the source
AST and fails the build on em dashes and a banned word list in string literals and JSX
text turns taste into a gate. Walk the syntax tree rather than grepping, so a banned word
in a variable name or a comment does not fail the build while one in user copy does.
Extend the list when review catches a new tic.

---

## 11. Data

- Schema changes go through reviewed migrations. No manual edits to a running database.
- Destructive changes split across two deploys: stop using it, ship, then remove it.
- Express constraints in the database, not only in application code. Application-only
  invariants hold until the first script that bypasses the application.
- Index what you filter, sort, and join on. Read the query plan for anything hot.
- Store money as integer minor units and timestamps as UTC with a timezone-aware type.
  Format at the edge.
- Multi-step writes are one transaction. Anything that must not be lost if the process
  dies is written in the same transaction as the state it describes, then delivered by a
  worker. A webhook fired mid-transaction is not delivery.
- Human actions on anything reviewable write an audit record in the same transaction:
  who, what, before, after, when. Typed fields, no free text, no personal data.
- One-off backfills and seeds are committed, reviewed code with a dry-run mode. The script
  someone ran from a terminal is the one nobody can audit when the data looks wrong later.

---

## 12. Security

- Every route makes an explicit authorization decision, and a test proves it. There is no
  default-allow path.
- Authorize against the resource, not only the role. "Is this an admin" and "does this
  user own this record" are different questions and both get asked.
- Validate input and shape output at the boundary (section 6).
- Secrets live in the environment and nowhere else. Only an example file is committed,
  with every key present and every value blank. Scan for leaked secrets automatically.
- Rate limit authentication, password reset, signup, and anything expensive, by both
  address and account.
- Set security headers and a content security policy. Cookies get the full set of flags,
  and cookie-authenticated state changes get CSRF protection.
- Never invent cryptography. Use maintained libraries and pin their exact versions
  (section 13).
- Never log secrets or personal data. Redact in the logger configuration, not at each
  call site, because a call site will be forgotten.
- User-facing errors never leak stack traces, queries, or internal paths. The detail goes
  to the log under an ID the user can quote.

---

## 13. Authentication and access

Authentication is the last thing to hand-roll. Use a maintained library, pin it exactly,
and spend your effort on authorization instead, which is the part no library can decide
for you.

- Adopt an auth library rather than assembling sessions, password hashing, verification,
  and reset flows yourself. Every one of those has a well-known way to get subtly wrong,
  and a library that thousands of projects audit is a better bet than your afternoon.
- When a library bootstraps schema, generate it once, then own it. Never re-run a
  generator against a live schema; it will overwrite the relations, indexes, and models
  you added. On upgrade, generate to a scratch file, diff, and apply changes deliberately.
- Pin the exact version. A patch release that changes hashing parameters silently
  invalidates every stored credential.

**Two credential types, from the first release.** Interactive sessions for browsers, and
API keys for everything programmatic: MCP servers, CI jobs, scripts, integrations, and
anyone building against your API. Adding the second type later means revisiting every
authorization check in the codebase, so build both in from the start even if only one has
a consumer today.

- API keys are stored hashed, shown once at creation, scoped to the narrowest set of
  permissions that works, revocable immediately, and attributable to a principal.
- Support expiry and rotation. A key that never expires is a key that outlives the
  contractor who created it.
- Rate limit and audit keys independently of sessions. Their traffic pattern is different
  and so is the blast radius.
- Both credential types resolve to the same caller identity and flow through the same
  authorization decision. One place decides what a caller may do, so a permission added
  for the UI cannot silently open the API.
- The differences that remain are transport-level: CSRF protection applies to cookies,
  not to bearer credentials.

**MCP servers are ordinary API clients.** An MCP server authenticates with a scoped key
over the documented API and gets no privileged path, no shared secret, and no direct
database access. If a tool it exposes needs a capability the API does not have, add the
endpoint and the permission rather than a side door. Treat tool inputs as untrusted, and
scope the key so a prompt injection reaches only what that server legitimately needs.

---

## 14. Containers

Everything runs in a container, from the first commit: local development, tests, CI, and
production. The gap between a laptop and production is a whole category of bug, and
containers delete it by construction rather than by discipline. Nobody installs a database
on their machine, and nobody debugs a failure that only happens in one environment.

- One Dockerfile per deployable. Multi-stage, so the runtime image carries the built
  artifact and its runtime dependencies and nothing else. Run as a non-root user.
- Compose describes the topology, one file per environment: development with source
  mounted and hot reload, test with the dependencies integration tests need and nothing
  persistent, production with the real thing. Same service names and shape across all
  three, so what you debug locally is what runs in production.
- Every dependency the app talks to comes from Compose: database, cache, search, queue,
  object storage. If a test needs a remote or shared service to pass, it is not a test you
  can trust.
- Build the image once in CI, tag it with the commit, and promote that exact artifact
  through environments. Only the env file changes. An image rebuilt at deploy time is not
  the image you tested.
- Pin base images by digest rather than a floating tag, and rebuild on a schedule so
  patches land deliberately instead of by surprise.
- Never bake secrets into an image. Build arguments persist in the layer history. Secrets
  arrive at runtime from files the host owns.
- The build must not need production credentials or network access to succeed. If a
  framework evaluates code at build time and demands a connection string, give the build
  stage a placeholder.
- Declare health checks and startup dependencies in Compose so ordering is deterministic
  rather than a race that usually wins.
- Persistent data lives in named volumes. A container is disposable and should be treated
  as such; if deleting one loses data, the topology is wrong.
- Keep the build context small and order layers so dependency installation caches. A slow
  image build is a gate people learn to skip.
- Getting a working environment is one command against a fresh clone. Onboarding that
  needs a page of manual steps is onboarding that rots, and the first person to hit a
  stale step usually fixes it locally and tells nobody.

---

## 15. Operations

- Structured logging with levels that mean something, and no stray print statements in
  application code.
- Every request carries an ID, logged on entry and exit and propagated downstream.
- A health endpoint reporting the running build and the state of each hard dependency.
  It is what the smoke tests and the load balancer both read.
- Every deploy traces to a commit, and the commit is visible at runtime.
- Background jobs are idempotent, retry with backoff, have a dead-letter path, and alert
  when the queue stops draining.
- Alerts are actionable. One nobody acts on gets deleted, not muted.
- Backups run automatically, off the machine holding the primary, checksummed, and copied
  somewhere a single compromised account cannot reach.
- Restore drills happen on a schedule, from the real artifact, with the elapsed time
  written down. Until a restore has actually been performed, the recovery time is a guess
  and the backup is a hope.
- Alert on a missing success, not only on a reported failure. A job that quietly stops
  running is the common case.
- Environments are reproducible from the repo (section 14). Topology, edge configuration,
  release steps, and scheduled jobs are committed; only the secret values live on the host.

---

## 16. Discovery surfaces

For anything with public pages, the machine-readable indexes are part of the app and
change with it. Generate them from the same content the pages render. A hand-maintained
list is a list someone forgets, and the first thing it does is advertise a deleted page.

Keep one module that knows the public route inventory, static routes declared in code and
dynamic ones read from live content, and have every surface below read from it:

- `sitemap.xml` with real modification dates, split into an index when it outgrows the
  size limits
- `robots.txt` with the rules and an absolute sitemap URL, blocking API, admin, and
  authenticated routes
- `llms.txt`, a short markdown map pointing at what matters, and `llms-full.txt`, the
  full public text in one response
- feeds for anything chronological
- web app manifest, page metadata, social cards, and structured data on content pages

Rules:

- Generated live. Unpublishing content removes it from every index on the next request.
- Only public, indexable, canonical URLs. Never a draft, a private record, a soft-deleted
  row, or a paginated duplicate.
- Absolute URLs from a single configured site URL. Never a hardcoded domain.
- Cache deliberately and invalidate on publish.
- Each surface has a test asserting a published item appears and a private one does not.
  The second assertion is the one that matters.
- Adding, removing, or renaming a public page updates the inventory and its tests in the
  same change.

---

## 17. Dependencies and configuration

**Adopt before you build.** Reach for a maintained library or a framework feature first,
and write custom code only for what is actually specific to this product. Auth, crypto,
payments, date handling, validation, rate limiting, migrations, retries, parsing, and
scheduling are all solved problems with known edge cases, and the version you write in an
afternoon has the same edge cases with none of the fixes. Use the framework's answer
before adding a library, and a library before writing your own.

The counterweight, which is real: every dependency is also a liability you now maintain.
Resolve the tension by choosing well rather than by choosing less.

- Prefer widely used, actively maintained, well-typed packages with a clear owner.
- Check what is already installed before adding anything. Three date libraries is a
  review failure.
- Wrap each external client in one module so replacing it is a contained change
  (section 4).
- Pin exactly anything touching auth, crypto, payments, or code generation.
- Automate upgrade pull requests and let the gates decide. Removing a dependency is a
  valuable change.
- Write it yourself when the requirement is genuinely yours, when the library is
  unmaintained or heavier than the problem, or when you would spend more time fighting its
  model than solving the task. Record that call in `docs/spec.md`.

Configuration:

- Environment variables, validated at startup. Feature flags are a runtime store you can
  change without a deploy. They are not the same thing.
- No environment branching inside business logic. Inject the difference instead.
- The example env file stays in sync with the schema, ideally enforced by a test.

---

## 18. Automation

Run the gates before code leaves the machine, and again on a clean checkout. Both, because
local hooks can be skipped and local machines lie.

- A fast pre-commit pass over staged files: format, lint, and a secret scan. Keep it fast
  enough that nobody wants to bypass it. A slow hook is a hook people learn to skip, which
  is worse than no hook because you stop trusting it.
- A pre-push pass running the full gates across the repo, since a change in one package
  breaks types in another.
- Commit messages validated to a convention, so history and release notes are derivable.
- Hooks and CI call the same script definitions. A command that exists only in a workflow
  file will break, and nobody notices until it matters.
- Pin actions to immutable references and grant the minimum token permissions. A mutable
  tag runs with your credentials.
- Serialize jobs that touch production so two releases cannot interleave.
- Untrusted code from forks never runs where production credentials live.
- Deploys migrate, release, smoke test, and keep the previous release ready to roll back.

If a gate gets slow enough that people want to bypass it, move work to CI rather than
letting the team learn to ignore red.

---

## 19. Agent configuration

The rules only work if every agent reads them, and the tooling belongs to the project
rather than to one laptop.

- `AGENTS.md` is canonical and `CLAUDE.md` is a symlink to it. No third copy: duplicated
  instruction files drift, and agents then follow whichever one is wrong.
- App-specific rules live with the app and extend this file rather than contradicting it.
- Commit the agent permission allowlist so everyone gets the same reviewed defaults, and
  keep personal overrides in an ignored local file.
- A procedure performed more than twice becomes a committed skill or command. A prompt
  pasted from a notes app is not a process.
- Review agent skills like code. One with a destructive step needs a dry run and an
  explicit confirmation.
- When a rule stops matching reality, fix the rule in the same change. Stale instructions
  produce confidently wrong work at scale, which is worse than no instructions.

---

## 20. Definition of Done

- [ ] Lint, typecheck, and unit tests clean
- [ ] Generated artifacts regenerated and committed
- [ ] New code tested, including auth and error paths for new routes
- [ ] New routes reachable and correctly authorized under both sessions and API keys
- [ ] Integration tests pass if the data layer changed; browser tests if a flow changed
- [ ] Copy reviewed for slop, no em dashes
- [ ] Works at phone width, keyboard reachable, adequate contrast
- [ ] No new duplication: checked what already exists first
- [ ] No secrets in the diff, example config updated
- [ ] Migration reviewed and safe against production data
- [ ] Still builds and comes up from a clean clone with one Compose command
- [ ] Public indexes still correct, with a test proving private content stays out
- [ ] `docs/spec.md` updated if the shape, the data model, or a decision changed

---

## 21. Adapting this

1. Copy `AGENTS.md` into the new repo and symlink `CLAUDE.md` to it.
2. Write `docs/spec.md` before writing code.
3. Drop the sections that genuinely do not apply. A CLI has no responsive gate; an
   internal tool has no discovery surfaces.
4. Keep sections 1, 2, 5, 6, 8, 17, and 20 regardless of what you are building. Those are
   the ones that stop a project from rotting.
5. Add project-specific rules to `docs/spec.md`, not here. This file is what is true
   across your projects; the spec is what is true about this one.
