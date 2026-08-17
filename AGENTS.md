# Engineering Standards

> Canonical rules for every human and AI agent working in this repo.
> `CLAUDE.md` is a symlink to this file. Edit this one.

Replace `<project>` and `@scope` throughout when you start a new project. Everything else
is the default. Deviating from a rule requires a line in `docs/decisions/` saying why.

---

## 0. Reference stack

The rules below are written against a concrete stack so they stay enforceable. Swap a
tool if a project needs it, but keep the role filled and keep the gates.

| Role | Default |
| --- | --- |
| Package manager | pnpm (workspaces), lockfile committed |
| Task runner | Turborepo |
| Language | TypeScript, `strict: true` |
| Runtime validation | Zod 4 |
| Web framework | Next.js App Router |
| API contract | ts-rest, contracts built from Zod schemas |
| API spec | `openapi.json` generated from the contract, committed |
| API docs | Scalar, rendered from `openapi.json` |
| Database | PostgreSQL via Prisma |
| Component library | shadcn/ui, vendored into `@scope/ui` |
| Styling | Tailwind |
| Unit / integration tests | Vitest |
| Browser tests | Playwright |
| Lint | ESLint, `@typescript-eslint/recommended-type-checked` |
| Format | Prettier |
| Logging | pino, structured JSON |

---

## 1. The Loop

Every task, no exceptions.

1. Write or change the code.
2. Write or update tests for everything you touched. A service function gets unit tests
   (happy path, each error branch, edge cases). A route gets handler tests. Middleware
   gets allow and deny tests. A component gets a render test.
3. Run the gates:
   ```bash
   pnpm lint          # zero errors, zero warnings
   pnpm typecheck     # zero errors across every package
   pnpm test:unit     # all passing
   pnpm openapi:check # spec matches the contract
   ```
4. If a gate fails, read the output, fix the root cause, run them again. Do not skip,
   do not suppress, do not mark the task done.
5. Repeat until all four pass clean.
6. Run `pnpm build` when the change touches runtime behavior, package exports, env
   wiring, or deployment.
7. Run `pnpm test:e2e` when the change touches a user-facing flow.

A task with failing lint, type errors, spec drift, or broken tests is an unfinished task.
Reporting it as done is the only unrecoverable mistake in this list.

---

## 2. Repo shape

```
src/
  packages/
    db/          -> @scope/db          Prisma schema, migrations, client singleton
    contracts/   -> @scope/contracts   Zod schemas + ts-rest contract + generated types
    ui/          -> @scope/ui          design system primitives, shadcn components
    env/         -> @scope/env         parsed, typed environment access
    logger/      -> @scope/logger      structured logger, request context
  apps/
    web/         -> Next.js app: pages, API route handlers, services
    worker/      -> background jobs (add when needed)
docs/             -> see section 18 for the full set
  architecture.md
  runbook.md
  decisions/      -> one file per non-obvious choice
.husky/           -> pre-commit, commit-msg, pre-push (section 16)
.github/
  workflows/      -> the same gates CI runs
  PULL_REQUEST_TEMPLATE.md
openapi.json      -> generated, committed, diffed in CI
```

Rules:

- Shared packages hold runtime-neutral code only. No product concepts, no feature flows.
- Never create a product-named folder under `packages/`. `packages/billing` is wrong;
  billing lives in `apps/web/src/billing`.
- Code moves into `packages/` only after two or more runtimes actually need it. Not
  before, not speculatively.
- Never import Prisma directly. Import from `@scope/db`.
- An app may depend on a package. A package must never depend on an app. Packages may
  depend on each other only in the order listed above.
- Each app owns its own `AGENTS.md` for app-specific rules. It extends this file, it does
  not contradict it.

---

## 3. TypeScript

- `strict: true` in every `tsconfig.json`.
- No `any`. Use `unknown`, a generic, a type guard, or a Zod-inferred type. The one
  allowed exception is a `details` field on an error schema validated at runtime.
- No `@ts-ignore`. `@ts-expect-error` is allowed only with a comment explaining why the
  error is expected and what would remove it.
- No `as` assertions until narrowing, type guards, and `.parse()` are exhausted.
  `as const` and `satisfies` are fine and encouraged.
- Explicit return types on every exported function, service, handler, middleware, hook.
- No implicit `any` parameters.
- No non-null `!` on values that come from IO, user input, or the database.
- `noUncheckedIndexedAccess: true`. Array and record access returns `T | undefined` and
  you handle it.
- Prefer discriminated unions over optional-field soup. Prefer `Result`-style returns
  over throwing for expected failures; reserve throws for programmer error.

---

## 4. Zod is the source of truth

One schema per concept, defined once, reused everywhere. Types are derived from schemas,
never hand-written alongside them.

```ts
export const CreateUserInput = z.object({ email: z.email(), name: z.string().min(1) })
export type CreateUserInput = z.infer<typeof CreateUserInput>
```

Parse at every boundary where data enters the process:

- HTTP request body, query, params, and headers you read.
- Environment variables, at boot, in `@scope/env`. The process exits on a bad env with a
  readable message listing every missing or invalid key. No `process.env` access anywhere
  else in the codebase; lint enforces it.
- Third-party API responses. Their contract is not your contract.
- Webhook payloads, after signature verification, before use.
- Anything read from a queue, cache, or file.
- Data crossing the client and server boundary in either direction.

Also:

- Shape the response. Never return a raw database row. Map it through an output schema so
  a new column cannot leak a secret next quarter.
- Parse database JSON columns on read. Prisma types them as `JsonValue`, which is a lie
  about the contents.
- Keep schemas in `@scope/contracts` when both the client and server need them. Keep them
  next to the feature when only the server does.
- Use `.strict()` on input objects so unexpected keys are rejected rather than ignored.
- Validation error responses list every failing field at once, not the first one.

---

## 5. API contract and OpenAPI

The ts-rest contract is the single source of truth for the API. Types, runtime
validation, client, server handlers, and the spec all come from it.

- Every route lives in the contract. A route handler with no contract entry is a bug and
  a test fails the build for it.
- Every endpoint declares: method, path, path params, query, body, every response status
  it can return, a `summary`, and a `description`. Undocumented endpoints do not ship.
- Errors use one schema everywhere: `{ error: string, code: ErrorCode, details?: unknown }`
  with `ErrorCode` a shared union. No route invents its own error shape.
- `pnpm openapi:generate` writes `openapi.json` from the contract.
- `pnpm openapi:check` regenerates into a temp file and diffs. Drift fails the build.
  This is what keeps the spec honest, so it runs in The Loop, not just in CI.
- `openapi.json` is committed. It is a reviewable artifact: a diff on it in a pull request
  is how a reviewer sees the API changed.
- The app serves the spec at `/openapi.json` and human docs at `/docs/api`, rendered from
  that same file by Scalar. Docs are never written by hand.
- Version the API in the path (`/api/v1/...`) from day one. Breaking a v1 endpoint means
  adding v2, not editing v1.
- Publish a typed client from the contract so other services and internal tools never
  hand-roll fetch calls.

If a project is not using ts-rest, the equivalents are `@hono/zod-openapi` for Hono or
`@asteasolutions/zod-to-openapi` for anything else. The rule that survives the swap:
the spec is generated from the same Zod schemas that validate at runtime, and drift
fails the build.

---

## 6. Testing

Five layers. Each has a job. Do not use one to fake another.

**Unit** (Vitest, `__tests__/unit/`)
Services, pure functions, schemas, utilities. Mock IO. Cover the happy path, every error
branch, and the boring edges: empty, one, many, null, unicode, very long, negative,
zero, boundary values. Fast enough to run on every save.

**Handler** (Vitest, `__tests__/unit/handlers/`)
Call each route handler directly with mocked auth and mocked services. Every route needs,
at minimum: 401 without credentials, 403 with the wrong role, 200 or 201 on the happy
path, 400 on invalid input, and each domain error status the contract declares. This
layer is why authorization bugs do not reach production.

**Integration** (Vitest, `__tests__/integration/`)
Real Postgres from docker compose or Testcontainers, migrations applied, no mocks below
the service layer. Covers transactions, constraints, cascades, race conditions, and the
queries that unit tests mock away. Each test runs in a transaction that rolls back, or
against a truncated schema. Never against a shared or remote database.

**E2E** (Playwright, `e2e/`)
Critical user journeys against a real running app: sign up, sign in, the core action the
product exists for, payment if there is one, and one admin flow. Test what a user does,
not what a component renders. Run against a preview deployment or a local build, never
production. Select by role and accessible name, not CSS class.

**Smoke** (Playwright project `smoke`, or `scripts/smoke.ts`)
Runs against a deployed URL after every deploy, in under sixty seconds, and never writes
data a human would have to clean up. It checks: `/api/health` returns 200 with the
expected build SHA, `/openapi.json` parses and matches the deployed version, the landing
page and one authenticated page return 200, sign-in works with a dedicated smoke account,
and one read-only core query returns results. Failure rolls the deploy back.

Rules across all layers:

- Write tests in the same change as the code. Not as a follow-up, not in a later task.
- A bug fix starts with a test that reproduces the bug and fails.
- Test behavior through the public interface. Do not reach into private functions or
  assert on implementation details, or every refactor becomes a test rewrite.
- No `.skip` and no `.only` on `main`. Lint catches both.
- Deterministic or deleted. Freeze time, seed randomness, never sleep, wait on conditions.
  A flaky test is worse than no test because it teaches the team to ignore red.
- Real assertions. `expect(result).toBeDefined()` is not a test.
- Factories over fixtures. `makeUser({ role: "ADMIN" })` with sensible defaults beats a
  giant JSON blob nobody dares change.
- Coverage floor of 80% lines on services, route handlers, and middleware. Coverage is a
  smoke alarm, not a goal; the review question is always "what breaks silently if this
  line is wrong", not "is the number green".

---

## 7. One component library, zero duplication

Pick one component library per project and use it for everything. This project uses
shadcn/ui, vendored into `@scope/ui`. Mixing libraries produces two of every button, two
focus styles, two dark mode implementations, and a design that never converges.

Before you write any component, look in this order:

1. `@scope/ui` (primitives: button, input, dialog, table, card, toast)
2. `apps/<app>/src/components/common` (composed patterns shared across features)
3. The feature folder you are in

Extend what you find. Only build new when nothing fits.

- Customize shadcn components in place inside `@scope/ui`. Do not wrap a wrapper.
- Design tokens live in one place: colors, spacing, radius, typography, shadows, and
  motion as CSS variables in `@scope/ui`. No hardcoded hex values in an app.
- Rule of three: the second copy of a pattern is a warning, the third is a refactor.
  Table shells, pagination, empty states, loading skeletons, error states, copy buttons,
  confirm dialogs, and form field wrappers get centralized on sight.
- If you are about to paste more than ten lines, extract instead.
- Forms use one stack everywhere: react-hook-form plus the Zod resolver plus the shared
  field components. Not a hand-rolled `useState` form in one corner of the app.
- Data fetching uses one stack everywhere: the generated ts-rest client plus TanStack
  Query. No bare `fetch` in a component.
- Every interactive component handles all five states: idle, loading, empty, error, and
  success. An empty state that says nothing and an error state that swallows the cause
  are both bugs.

Duplication also gets checked at the data layer: field lists, enums, status values, and
copy strings each have exactly one definition that everything else imports.

---

## 8. Responsive and accessible, as a gate

Mobile is not a phase. It is part of done.

- Every surface works at 320px and 390px width: landing, blog, docs, auth, dashboard,
  admin, settings, errors.
- `document.documentElement.scrollWidth > window.innerWidth` is a failure. A Playwright
  test asserts it on every key route at 320px.
- A component that can exceed the viewport owns its own wrapping or horizontal scroll.
  Never rely on a parent to hide overflow.
- Use `min-w-0`, `max-w-full`, and `break-words` by default in flex and grid children.
- Test with hostile content: 80-character emails, unbroken URLs, hashes, long names,
  empty strings, and the longest realistic value for every field.
- Truncation is allowed only when the full value stays reachable through a title
  attribute, a copy control, or a detail view.
- Fixed-width things need explicit narrow-screen behavior: code blocks, tables, embeds,
  captcha widgets, QR codes, dialogs, tab bars, pagination, and button groups. If a
  third-party embed is unusable at phone width, give it a fallback or open it in a tab.
- Treat CMS and user-generated HTML as hostile to layout. Sanitize inline widths and
  guard article containers.
- Accessibility: every interactive element reachable and operable by keyboard, visible
  focus rings, labels tied to inputs, `alt` on meaningful images, ARIA only when semantic
  HTML cannot express it, and 4.5:1 contrast. `@axe-core/playwright` runs on key routes
  in E2E and fails on serious or critical violations.
- Respect `prefers-reduced-motion` and `prefers-color-scheme`.

---

## 9. Copy quality

Every user-visible string is product surface: landing pages, dashboards, admin screens,
forms, helper text, empty states, errors, emails, docs, API descriptions, metadata, and
machine-readable endpoints.

- Run every new or materially edited string through the `no-ai-slop` skill before the
  change is complete (<https://github.com/petergyang/no-ai-slop>). If it is not
  installed, install it globally and read `SKILL.md` and `eval.md` first.
- Concrete facts, direct verbs, shortest version that still helps the reader act.
- Cut: throat-clearing, portable filler that would fit any product, repeated claims,
  fake contrasts ("it's not just X, it's Y"), inflated importance, and decorative
  formatting.
- No em dashes in application copy.
- Preserve factual, legal, security, and operational meaning. Never invent a claim, and
  never compress away a limit, condition, or required instruction.
- Error messages say what happened and what the reader can do next. "Something went
  wrong" is not an error message.

Mechanize it. A style rule nobody can check is a style rule that decays, so
`__tests__/unit/copy-style.test.ts` parses every `.ts` and `.tsx` file in the app with the
TypeScript compiler API, visits every string literal, template part, and JSX text node,
and fails on:

- em dashes and their HTML entities (`—`, `&mdash;`, `&#8212;`, `&#x2014;`)
- a banned word list: delve, foster, leverage, utilize, facilitate, empower, streamline,
  robust, cutting-edge, paradigm shift, game changer, tapestry, realm, beacon,
  multifaceted, meticulous, intricate, paramount, transformative, elevate, embark,
  supercharge, harness, ever-evolving, seamless, unlock, unleash, dive into, navigate the

Walk the AST rather than grepping the file, so a banned word in a variable name or a code
comment does not fail the build while one in user-facing copy does. Extend the list when
review catches a new tic. Keep copy regression tests current when wording changes.

---

## 10. Database

- Schema changes go through migrations. Always. No manual edits to a running database,
  no `db push` outside a scratch branch.
- Migrations are forward-only and reviewed like code. A migration that cannot run against
  production data is not ready.
- Destructive migrations split into two deploys: stop writing the column, ship, then drop
  it. Never drop and deploy in one step.
- Every foreign key, every uniqueness rule, and every non-null constraint is expressed in
  the database, not only in application code.
- Index every column you filter, sort, or join on. Check the query plan for anything on a
  hot path.
- Money is integer minor units. Timestamps are `timestamptz`, stored UTC, formatted at
  the edge. Enums are database enums or a checked constraint, never a free string.
- Soft delete with `deletedAt` when the record has audit or support value. Every query
  path filters it, and a partial unique index handles the released-identifier case.
- Multi-step writes run in one transaction. Anything that must not be lost if the process
  dies goes in the same transaction as the state it describes, then to an outbox worker.
  Never fire a webhook or an email mid-transaction and call it delivered.
- Admin mutations of anything a human reviews write an audit row in the same transaction:
  actor, action, before state, after state, timestamp. Typed fields and operational
  metadata only, never free-text notes or PII.
- Seed scripts are idempotent and safe to run twice.

---

## 11. Security

- Every route makes an explicit authorization decision. There is no default-allow path,
  and handler tests prove it for each route.
- Authorization is checked against the resource, not only the role. "Is this user an
  admin" and "does this user own record 41" are different questions and both get asked.
- Parse and validate all input at the boundary (section 4). Shape all output.
- Secrets live in the environment and nowhere else. Only `.env.example` is committed, with
  every key present and every value blank or fake. Secret scanning runs in CI.
- Rate limit authentication, password reset, signup, search, and anything expensive.
  By IP and by account.
- Set security headers and a Content Security Policy. No `unsafe-inline` on scripts.
- Cookies: `httpOnly`, `secure`, `sameSite`. CSRF protection on cookie-authenticated
  state-changing requests.
- Hash passwords with a memory-hard function via a maintained library. Never invent
  crypto, never reuse a nonce, and pin the exact version of anything doing auth so a
  patch release cannot silently change your hash parameters.
- API tokens are stored hashed. They are shown once at creation and never again.
- Never log secrets, tokens, passwords, full card numbers, or PII. Redaction lives in the
  logger config so it cannot be forgotten at a call site.
- File uploads: validate type by content and not by extension, cap size, store outside
  the web root, and serve from a separate origin.
- Dependency audit runs in CI. Direct dependencies pinned exactly; a lockfile is not a
  substitute for pinning anything security-critical.
- Errors returned to users never leak stack traces, queries, or internal paths. The
  detail goes to the log with a correlation ID the user can quote.

---

## 12. Observability and operations

- Structured JSON logging through `@scope/logger`. No `console.log` in application code;
  lint enforces it.
- Every request gets an ID, logged on entry and exit with method, path, status, and
  duration, and propagated to downstream calls.
- Log levels mean something: `error` is someone gets paged, `warn` is someone looks
  tomorrow, `info` is a state change worth a timeline, `debug` is off in production.
- Error tracking (Sentry or equivalent) with PII scrubbing and release tagging.
- `/api/health` returns 200 with the build SHA, the version, and the status of each hard
  dependency. It is what the smoke test and the load balancer both read.
- Every background job is idempotent, retries with backoff, has a dead-letter path, and
  alerts when the queue stops draining.
- Alerts are actionable. An alert nobody acts on gets deleted, not muted.

**Backups.** An untested backup is a guess.

- Automated, scheduled, and off the machine that holds the primary. A snapshot on the
  same host survives a bad migration and nothing else.
- Checksummed on write and verified on read. A silently truncated dump is the normal
  failure, not a dramatic one.
- Copied to a second provider. One vendor account is one billing dispute away from zero.
- Restore drill on a schedule, into a scratch environment, from the real artifact, with
  the elapsed time written down. That number is your actual recovery time objective, and
  the first drill is always slower than anyone guessed.
- The backup job alerts on failure and on silence. A job that stops running quietly is
  the common case, so alert on a missing success, not only on a reported error.
- The restore procedure lives in `docs/runbook.md`, written by whoever ran the last drill.

---

## 13. Discovery surfaces

Every machine-readable index is generated from the same content the pages render. None of
them is a hand-maintained list. A static `public/sitemap.xml` is a file someone will
forget, and the first thing it does is advertise a deleted page to a crawler.

One module, `src/lib/discovery.ts`, exports the canonical route inventory: static routes
declared in code, dynamic routes read from the database at request time. Every surface
below reads from it, so adding a page updates all of them at once.

| Surface | Route | Contents |
| --- | --- | --- |
| Sitemap | `/sitemap.xml` (`app/sitemap.ts`) | Every indexable public URL with `lastModified`, `changeFrequency`, `priority`. Split into a sitemap index above 50,000 URLs or 50MB. |
| Robots | `/robots.txt` (`app/robots.ts`) | Allow and disallow rules plus the absolute sitemap URL. Blocks `/api`, `/admin`, `/dashboard`, auth callbacks, and any query-param trap. |
| LLM index | `/llms.txt` | Short markdown map of the site: one H1 project name, a blockquote summary, then link sections with a one-line description each. Points at the good stuff, not everything. |
| LLM full text | `/llms-full.txt` | Full markdown body of the public content, concatenated, for models that want the corpus in one request. |
| Feed | `/blog/feed.xml` (also `feed.json`) | Published posts, newest first, absolute URLs, real publish dates. |
| Manifest | `/manifest.webmanifest` (`app/manifest.ts`) | Name, icons, theme color, start URL. |
| Metadata | `app/layout.tsx` plus per-page `generateMetadata` | Title template, description, canonical URL, Open Graph, Twitter card, and a generated OG image per content page. |
| Structured data | JSON-LD in the page | `Organization` and `WebSite` at the root, `Article` on posts, `BreadcrumbList` on nested pages, `FAQPage` where it applies. |

Rules:

- All of these are dynamic route handlers reading live content. If a post is unpublished,
  it disappears from the sitemap, the feed, and both LLM files on the next request.
- Only public, indexable, canonical URLs appear. Never leak a draft, a private record, an
  admin route, a soft-deleted row, or a paginated duplicate into an index.
- Absolute URLs everywhere, built from one `SITE_URL` env value. Never a relative link and
  never a hardcoded domain.
- Set caching deliberately: revalidate the sitemap and feeds on a schedule rather than
  rebuilding them per request, and bust the cache when content publishes.
- Every one of these routes has a unit test asserting content type, structure, that a
  known published item appears, and that a known draft or private item does not.
  The private-item assertion is the one that matters.
- Next.js prerenders these at build time even with `force-dynamic`, so a Docker build
  needs a dummy `DATABASE_URL` in the builder stage or the build fails on a route that
  queries content.
- When you add, remove, rename, or materially change a public page or a machine-readable
  endpoint, update the inventory module and its tests in the same change. This is a
  checklist item in section 20, not an optional cleanup pass.

---

## 14. Environment and configuration

- One schema in `@scope/env`, parsed once at boot, split into server-only and
  client-exposed. The process refuses to start on invalid config.
- `.env.example` stays in sync with the schema. A test asserts every schema key appears
  in `.env.example`.
- Configuration is environment variables. Feature flags are a runtime store you can flip
  without a deploy. Do not confuse the two.
- No environment branching in business logic. `if (env.NODE_ENV === "production")` inside
  a service is a design problem; inject the difference instead.

---

## 15. Dependencies

- Every dependency is a liability with a maintenance cost. Prefer the standard library,
  then a small focused package, then a framework.
- Before adding one, check what is already in the lockfile. Three date libraries is a
  failure of review.
- Pin exact versions for auth, crypto, payments, and anything that generates a schema.
  Caret ranges are fine elsewhere.
- Lockfile committed. Renovate or Dependabot opens the upgrade PRs; CI gates the merge.
- Removing a dependency is a valid, valuable pull request.

---

## 16. Git hooks

The gates run locally before code leaves the machine. Finding a lint error twenty minutes
into a CI run is a waste of the run and of your attention. Hooks are managed by Husky and
committed to the repo, so a fresh clone plus `pnpm install` installs them automatically
via the `prepare` script.

**pre-commit** (fast, staged files only, target under 10 seconds)

```
lint-staged:
  *.{ts,tsx}       -> eslint --fix --max-warnings 0, prettier --write
  *.{json,md,css}  -> prettier --write
  *                -> secret scan (gitleaks protect --staged)
```

Keep it fast. A slow pre-commit hook is a hook people bypass, and a bypassed hook is
worse than no hook because you stop trusting it.

**commit-msg**

Validate the conventional commit format with commitlint. Reject anything that is not
`type(scope): subject`.

**pre-push** (the real gate, target under 3 minutes)

```bash
pnpm typecheck
pnpm lint
pnpm test:unit
pnpm openapi:check
```

Run these against the whole repo, not just staged files, because a change in one package
breaks types in another. If the push touches migrations, also run `pnpm test:integration`.

**Rules**

- The hooks run exactly the commands from The Loop (section 1). One definition, called
  from `package.json` scripts, invoked by both the hooks and CI. Never a second copy of
  the command list that drifts from the first.
- Hooks are a fast feedback loop, not the authority. CI re-runs everything on a clean
  checkout, because hooks can be skipped and local machines lie.
- `--no-verify` is for a genuine emergency. Using it means you say so in the pull request.
- Cache aggressively. Turborepo skips unchanged packages, which is what keeps pre-push
  inside its budget as the repo grows.
- If a hook gets slow enough that people want to bypass it, move work to CI rather than
  letting the team learn to ignore red.

---

## 17. Git, CI, and deployment

- `main` is protected. No direct pushes. Every change is a pull request.
- Conventional commits: `feat:`, `fix:`, `chore:`, `docs:`, `test:`, `refactor:`,
  `perf:`. The subject says what changed and why, not which files moved.
- One logical change per pull request. A refactor and a feature in one diff is a review
  you cannot actually do.
- Never commit `.env`, credentials, keys, dumps, or generated build output.
- CI runs, on every pull request: lint, typecheck, unit, integration, `openapi:check`,
  build, secret scan, dependency audit, and E2E. All required to merge.
- CI calls the same `package.json` scripts the hooks call. If a command exists in a
  workflow file and nowhere else, it will break and nobody will notice until it matters.
- Pin every GitHub Action to a full commit SHA with the version in a trailing comment
  (`uses: actions/checkout@11d5960a... # v4`). A tag is mutable and a compromised action
  runs with your credentials.
- Set `permissions:` explicitly at the workflow level, starting from `contents: read`.
  The default token is far broader than any job needs.
- Use a `concurrency` group for production jobs so two deploys cannot interleave, with
  `cancel-in-progress: false` for anything that touches a database.
- Untrusted pull request code never runs on a self-hosted runner that holds production
  credentials. Fork PRs run on ephemeral hosted runners or not at all.
- Deploys run migrations first, then release, then smoke tests, then keep the previous
  release warm for rollback. A failing smoke test rolls back automatically.
- Every deploy is traceable to a commit SHA, and the SHA is served by `/api/health`.

**Infrastructure lives in the repo.** Reproducing an environment from memory is how
environments diverge.

```
deploy/<target>/
  docker-compose.yml      the production topology
  Caddyfile               or nginx.conf, the edge config
  release.sh              build, transfer, migrate, restart, health check
  backup.service          plus backup.timer, the scheduled job units
  stack.env.example       every key present, every value blank
  README.md              first-time setup and verification steps
docker-compose.dev.yml    local dependencies
docker-compose.test.yml   integration test dependencies
```

Secrets stay on the host in a `0600` file and never enter Git or CI configuration. Local,
test, and production run the same images from the same compose definitions with different
env files, so "works on my machine" stops being a category of bug.

**One-off scripts are code.** Backfills, migrations, and seeds go in `scripts/`, get
committed, get reviewed, and get a dry-run flag. The one you ran by hand from a terminal
is the one nobody can audit six months later when the data looks wrong. Seeds are
idempotent; backfills are resumable and log what they changed.

---

## 18. Documentation

Documentation you have to maintain by hand rots. Generate what can be generated, and keep
the hand-written set small enough that keeping it true is realistic.

```
README.md              What this is, run it in five commands, test it, deploy it
AGENTS.md              This file. CLAUDE.md is a symlink to it
CHANGELOG.md           Generated from conventional commits
docs/
  architecture.md      Current shape of the system, one diagram, updated when it changes
  data-model.md        Entities, relationships, the invariants the schema cannot express
  api.md               Pointer to /docs/api and the auth model. Endpoints are generated
  runbook.md           Deploy, roll back, rotate a secret, restore a backup, common alerts
  onboarding.md        Zero to running locally, plus the five things that will confuse you
  security.md          Threat model, auth model, what is trusted, what is not
  testing.md           What each layer covers, how to run it, how to add a fixture
  decisions/
    0001-<title>.md    One per non-obvious choice
    template.md
```

- Every document has an owner and a reason to exist. Delete one that has neither. Six
  accurate pages beat forty stale ones.
- `docs/decisions/NNNN-title.md`: context, the decision, the alternatives rejected, and
  what this rules out later. Written when the decision is made, not reconstructed a year
  later from a Slack thread. Never edited after the fact; superseded by a later record
  that links back to it.
- `docs/runbook.md` is the one that pays for itself at 3am. Every operational procedure
  gets an entry the first time someone performs it, written by that person.
- Comments explain why, not what. If a comment restates the code, delete the comment. If
  the code needs a comment to be readable, fix the code first.
- API documentation is generated (section 5). Never hand-written, never duplicated.
- A pull request that changes the shape of the system, an operational procedure, or a
  tradeoff updates the corresponding document in the same diff.

---

## 19. Agent configuration lives in the repo

The rules only work if every agent reads them, and the tooling around them is part of the
project, not part of one person's laptop.

```
AGENTS.md                     canonical standards, this file
CLAUDE.md                     symlink to AGENTS.md
apps/<app>/AGENTS.md          app-specific rules that extend the root file
.claude/settings.json         committed: permission allowlist, hooks, env
.claude/settings.local.json   gitignored: personal overrides
.claude/commands/             repeatable prompts as slash commands
.agents/skills/<name>/        repo-local skills, committed and reviewed
```

- Commit `.claude/settings.json` with the permission allowlist for the safe read-only
  commands this repo uses. Everyone gets fewer prompts, and the list is reviewable. Keep
  personal overrides in `settings.local.json` and gitignore it.
- A procedure an agent performs more than twice becomes a committed skill or slash
  command. A prompt pasted from a notes app is not a process.
- Skills are reviewed like code. One with a destructive step needs a dry-run mode and an
  explicit confirmation, the same as any script in `scripts/`.
- Instruction files are load-bearing, so keep them honest: when a rule stops matching
  reality, fix the rule in the same change. A stale `AGENTS.md` produces confidently
  wrong work at scale, which is worse than no instructions.
- One canonical file, one symlink, no third copy. Duplicated instruction files drift, and
  agents then follow whichever one is wrong.

---

## 20. Definition of Done

A change is done when all of these are true:

- [ ] `pnpm lint` clean, zero warnings
- [ ] `pnpm typecheck` clean across every package
- [ ] `pnpm test:unit` passing, new code covered
- [ ] `pnpm openapi:check` clean, `openapi.json` committed if the API changed
- [ ] Integration tests passing if the change touches the database
- [ ] E2E passing if the change touches a user flow
- [ ] Every new route has handler tests for auth, happy path, and each error status
- [ ] Every user-visible string run through `no-ai-slop`, no em dashes
- [ ] Checked at 320px and 390px, no horizontal overflow, all actions reachable
- [ ] Keyboard reachable, labeled, sufficient contrast
- [ ] No new duplication: checked `@scope/ui` and `components/common` first
- [ ] No secrets in the diff, `.env.example` updated if config changed
- [ ] Migration written, reviewed, and safe against production data
- [ ] Sitemap, robots, `llms.txt`, `llms-full.txt`, feed, manifest, and metadata still
      correct if a public page or endpoint changed, with a test proving private content
      stays out
- [ ] Pre-push hook passed without `--no-verify`
- [ ] Docs and decision record updated if the shape or a tradeoff changed

---

## 21. Starting a new project from this

1. Copy `AGENTS.md` and `README.md` into the new repo. Recreate `CLAUDE.md` as a symlink
   to `AGENTS.md` (`ln -s AGENTS.md CLAUDE.md`).
2. Replace `<project>` and `@scope`.
3. Delete the sections the project genuinely does not have. A CLI tool has no responsive
   gate. A private internal tool has no discovery surfaces.
4. Keep sections 1, 3, 4, 6, 16, and 20 regardless of what you are building. Those are
   the ones that stop the project from rotting.
5. Add app-specific rules in `apps/<app>/AGENTS.md`, not here.

Instruction files drift when they are duplicated. `AGENTS.md` is canonical, `CLAUDE.md`
is a symlink to it, and no third copy exists.
