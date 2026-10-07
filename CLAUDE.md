# CLAUDE.md

## Overview
<!-- One line: what is being built and for whom. Fill in from the spec before the first prompt. -->

## Stack (defaults, change only if the spec says otherwise)
- **Language:** TypeScript everywhere, strict mode on.
- **Backend:** Node.js with Express, REST API only. Swap the framework (Fastify, NestJS) only if the spec asks for it, and keep the same layering.
- **Database:** PostgreSQL. Parameterized queries only, no string-built SQL.
- **Frontend:** React with Vite, TanStack Query for server state, axios for HTTP.
- **Validation:** Zod on every backend endpoint. Share inferred types where practical.
- **Auth:** JWT with role-based access control. Authorize every protected route.
- **Testing:** Vitest. Backend endpoints get Supertest. Frontend gets React Testing Library for key flows.
- **Tooling:** ESLint, Prettier, and a build check before every commit.

## Conventions

### Architecture (MVC, split into two apps)
- **Backend = Model + Controller (MC), exposed as a REST API only.** Models hold data and business rules. Controllers handle requests and return JSON over REST endpoints. The backend renders no views or HTML.
- **View = a separate, standalone frontend application.** It lives in its own folder with its own dependencies, build, and version. It consumes the backend's REST API and nothing else.
- Never put view or UI logic in the backend, and never put business rules or direct database access in the frontend.
- **Third-party APIs are called from the backend only.** The frontend never calls a third-party service directly. The backend makes the call, maps the result into our own response shape, and returns that to the frontend. Never expose the vendor to the client: no vendor names in endpoints, field names, error messages, or logs sent to the browser, no vendor URLs, and no raw vendor responses or error payloads passed through. Keys and credentials stay in backend environment variables. Name endpoints and fields by what they do (`/payments`, `/sms/send`), not by who provides them.
- The API is the contract between the two apps. Define or update it (endpoints, request and response shapes, error format) in the plan before changing either side.

### Backend structure
- Folders: `routes/`, `controllers/`, `models/` (or `services/` for business rules), `middleware/`, `validation/`, `utils/`.
- **Thin controllers.** Parse input, call a model or service, return the response. No business rules inside route handlers.
- **One response shape.** Success: `{ data, meta? }`. Error: `{ error: { code, message, requestId } }`. Use correct HTTP status codes: 400 invalid input, 401 unauthenticated, 403 forbidden, 404 not found, 409 conflict, 500 unexpected.
- **One error handler.** A central error middleware turns thrown errors into the error shape above. Never leak stack traces or vendor details to the client.
- **Pagination by default** on any list endpoint.

### Frontend structure (View app)
- **TypeScript only.** The frontend is written in TypeScript with strict mode on. No `any` unless justified in a comment.
- **One HTTP client.** A single axios instance in `src/api/client.ts`, with the base URL from an env variable. A request interceptor attaches the token. A response interceptor handles `401` (refresh or log out and redirect) and converts every failure into one consistent error shape. The client knows nothing about specific resources.
- **One API module per resource.** Example: `src/api/users.api.ts`, `src/api/orders.api.ts`. Each exports typed functions (`list`, `getById`, `create`, `update`, `remove`) that call the shared client and return unwrapped, typed data. Nothing above this layer sees axios.
- **Global typed DTOs.** Types live in `src/types/`, outside `api/`, one file per resource (`user.types.ts`, `order.types.ts`) plus `common.types.ts` for shared shapes (`ApiError`, `PaginatedResponse<T>`). `types/` imports nothing from the app. Name request payloads by direction (`CreateUserPayload`) and responses by entity (`User`). If the backend format differs from what the UI needs, map it inside the API module.
- **Components never call axios or API modules directly.** Pages and components use hooks that own loading, error, and caching state, built on TanStack Query.
- **Every data view handles four states:** loading, error, empty, and success.
- Dependency direction: `components` → `hooks` → `api` → `client`, with `types` importable from anywhere.

### System logging (backend)
- **One logger module.** All backend code logs through a single logger (`info`, `warn`, `error`, `debug`). Never use raw `console.log` or write to files directly from feature code.
- **Swappable destination.** The logger decides where logs go, driven by config or env. Right now logs are written to a `.txt` file (for example `logs/system.txt`). Later this can change to any tool by changing the logger's sink only, with no changes to feature code.
- **One line per entry, structured.** Each entry has timestamp, level, request ID, message, and context, as one JSON object per line so it stays easy to search and to parse with AI-assisted debugging.
- **Request ID.** Generate one per incoming request, include it in every log line, and return it in API error responses so a frontend error can be matched to a backend log entry.
- **What to log:** unhandled exceptions, failed requests, auth events (login, failure, logout), and every third-party call (service name, endpoint, duration, status). Vendor details belong in server logs only, never in client responses.
- **Never log:** passwords, tokens, API keys, full third-party payloads containing personal data, or other sensitive personal data. Redact before logging.
- **File hygiene:** rotate by day or size, keep `logs/` in `.gitignore`, and never commit log files.
- **Quick mode:** use a minimal logger (level, message, request ID, to stdout and file). Add rotation and redaction rules when the project moves past a prototype.

## How to Work in This Repo (AI Workflow)
1. **Read first.** Read this file and any relevant doc in `docs/` (plans, ADRs, migration notes) before writing code. Open the nearest similar file and follow its pattern. Do not invent new conventions.
2. **Plan before code.** For anything beyond a small fix, write a short plan first: goal, files to touch, data model changes, API contract, edge cases, test steps. Save it to `docs/plans/<feature-name>.md` and wait for approval before implementing.
3. **Work in small steps.** One logical change at a time (schema, model, endpoint, UI). Verify each step before moving on.
4. **Ask, don't guess.** If a requirement or business rule is unclear, ask. Never fill in domain logic from assumption.
5. **Stay in scope.** Do not refactor, rename, or reformat unrelated code in the same change.
6. **Stop on loops.** If the same error persists after 2 attempts, stop. Re-read the error, narrow the scope, revert to the last good state, and restate the problem with more context instead of retrying blindly.

### Quick mode (small, time-boxed tasks)
Use when a task is small, for example a single CRUD endpoint with a simple UI, or any time-boxed exercise.
- The plan is five lines or fewer, written in chat. No file needed.
- Order: types and validation schema, then endpoint, then a minimal UI, then one happy-path test and one edge-case test.
- Skip ADRs and migration docs, but write one line in the plan for any non-obvious decision.
- The Critical Rules and the Review checklist still apply in full. Quick mode shortens the paperwork, never the review.

## Model Selection
| Task | Use |
|------|-----|
| Planning, research, requirement breakdown, ADR drafting | Sonnet |
| Complex or risky coding (auth, payments, data migrations, concurrency) | Opus |
| Quick iterations (copy tweaks, small UI fixes, renames, boilerplate) | Sonnet or Haiku |
| Reviewing a diff or a failing log | Opus for logic and security, Sonnet for routine |

Switch model when the task type changes. Do not use one model the same way for everything.

## Git Identity
- Name: Syed Zohaib Hussain
- Email: roxas.zohbi786@gmail.com

## Git
- Never push to main directly
- New feature: fresh branch from the latest main, `feature/<feature-name>`
- Bug fix: `fix/<short-issue-description>`
- Small, focused commits with a prefix: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`

## Critical Rules
1. **New migration per schema change.** Never edit existing migration files.
2. **Validate on the server.** Every endpoint validates its own input with Zod. Frontend checks are for UX only.
3. **Authorize every route.** Authentication and permission checks on every protected endpoint.
4. **No secrets in code or commits.** Use environment variables. Never log tokens, credentials, or sensitive personal data.
5. **Build, lint, and test before every commit.** Fix all errors first. Never push a broken build.
6. **Reuse before creating.** Use existing components and utilities. UI prototyped with AI (Claude artifacts or similar) must be converted into shared components before merging.
7. **Idempotency on money and retries.** Any endpoint that creates a payment, order, or other duplicate-sensitive record accepts an idempotency key.

## Testing
- Every new endpoint ships with at least: one happy-path test, one validation-failure test, and one authorization test (wrong role or no token).
- Test edge cases from the review list below, not only the happy path.
- Tests describe behavior in plain names (`returns 403 when user lacks role`), not implementation details.
- A failing test is fixed or explained. Never delete or skip a test to get green.

## Review and Verification (before every commit)
AI output is a draft. Every line shipped must be understood by the developer.
1. **Read the full diff.** Check it matches the plan and touches only the intended files.
2. **Logic review.** Walk through the happy path, then edge cases: empty data, duplicates, null values, wrong role, expired token, double submit, timezone boundaries.
3. **Run it.** Build, run tests, and do a manual pass in the browser and against the endpoints.
4. **Check permissions.** A user without access gets a 403, and the UI hides what they cannot use.
5. **Check data safety.** Migrations must be reversible and safe on existing rows.

### Layered debugging
Work from the outside in, one layer at a time:
1. Browser console and network tab (status codes, request and response payloads)
2. API and server logs, matched by request ID
3. AI-assisted parsing: paste the relevant log lines or stack trace with the code path and ask for the root cause. Verify the explanation against the code before applying any fix.

## Where AI Output Needs Extra Scrutiny
Do not accept AI output on these areas without a line-by-line review and a manual test:
- **Security and auth:** verify against the existing implementation. Never weaken existing checks.
- **Money and domain logic:** business rules come from the requirements, not the model. Check rounding, totals, and who is allowed to do what.
- **Schema and migrations:** review each column, index, default, and rollback by hand.
- **Time logic:** timezones, midnight boundaries, double submissions.
- **Edge cases and concurrency:** AI tends to cover the happy path only. Write the edge cases down and test them explicitly.

## Decision Records and Docs
- **ADRs** live in `docs/adr/NNNN-short-title.md`. Write one for any significant decision: schema design, permission model, new dependency, auth approach, architecture change.
  - Format: Status, Context, Decision, Alternatives considered, Consequences.
  - An ADR is written before or with the change that implements it, and is never silently edited afterward. Supersede it with a new ADR instead.
- **Migration notes** go in `docs/migrations.md`: what changed, why, and rollback notes for any non-trivial schema change.
- **Plans** go in `docs/plans/<feature-name>.md`.
- Update this file when a new convention is agreed.

## Definition of Done
- Matches the plan and the API contract.
- Input validated, routes authorized, errors handled with the standard error shape.
- Tests added and passing, build and lint clean.
- Diff read in full and understood.
- Docs updated if a convention, schema, or major decision changed.