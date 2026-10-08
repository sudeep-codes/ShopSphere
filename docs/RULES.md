# RULES — Non-negotiables for humans and coding agents

> Read this file at the start of **every** agent session. "MUST" = never violate. "SHOULD" = deviate only with a note in `MEMORY.md`.

## 0. Document precedence
`PRD.md` (what) > `ARCHITECTURE.md` (how) > `DESIGN.md` (UI/API) > `RULES.md` > `PHASES.md` (when). `MEMORY.md` records the current state and decisions; if MEMORY contradicts another doc, the newer decision in MEMORY wins **and** the other doc must be updated in the same session.

## 1. Agent workflow
1. MUST read `RULES.md` + `MEMORY.md` first, then only the doc sections relevant to the task.
2. MUST work on **one task ID from `PHASES.md` at a time**; no "while I'm here" refactors.
3. MUST NOT change the DB schema, API contract or env variable names without updating `ARCHITECTURE.md`/`DESIGN.md` and logging a decision (`D-0xx`) in `MEMORY.md`.
4. MUST NOT add a dependency that isn't in §2 without logging why.
5. MUST end each session by: running tests/lint, appending a Session Log entry to `MEMORY.md`, ticking the task in `PHASES.md`, and writing the next task in `MEMORY.md → Current Status`.
6. If a requirement is ambiguous **and** affects schema, API or AWS cost → stop and ask. Otherwise choose the simplest option and log it.
7. MUST NOT invent AWS resource names, endpoints or ARNs — read them from `MEMORY.md → Resource Inventory` or ask.

## 2. Stack lock
| Layer | Choice |
|---|---|
| Runtime | Node.js 22 LTS, ES modules (`"type": "module"`) |
| API | Express 5, `zod` (validation), `helmet`, `cors`, `express-rate-limit`, `cookie-parser`, `pino` (logs) |
| SQL | `pg` (node-postgres), plain SQL — **no ORM** (SQL is what we're demonstrating) |
| Aurora auth | `@aws-sdk/rds-signer` |
| DynamoDB | `@aws-sdk/client-dynamodb` + `@aws-sdk/lib-dynamodb` |
| Auth | `bcryptjs`, `jsonwebtoken`, `uuid` (or `crypto.randomUUID`) |
| Frontend | React 18 + Vite + React Router; plain CSS with `tokens.css`; `recharts` only on Insights page |
| Tests | `vitest` + `supertest` against docker-compose services |
| Process mgr | `pm2` on EC2 |

## 3. Code conventions
- Folder per domain: `modules/<domain>/{routes.js, service.js, repo.js}`. Routes = HTTP only; service = business rules; repo = SQL/DynamoDB calls.
- Async/await only; no callbacks. Express 5 handles rejected promises — still wrap domain errors in `AppError(code, httpStatus, message)`.
- Names: `camelCase` JS, `snake_case` SQL columns and JSON fields returned by the API (match the brief: `customer_id`, `total_amount`).
- Max ~200 lines per file; split otherwise.
- Every endpoint validates input with a zod schema.
- Log as JSON: `{reqId, route, status, ms, dataSource, servedBy}`. Never log passwords, tokens, cookies, full card numbers.

## 4. SQL rules (MUST)
- Parameterised queries only (`$1, $2…`). String-concatenated SQL is a bug.
- Every SQL call goes through `db.read()`, `db.write()` or `db.tx()` — never `pool.query` directly.
- Use `db.read()` **only** for catalog/reporting reads that tolerate replica lag. Anything the same user just wrote → `db.write()`.
- Multi-statement writes run inside `db.tx(async (client) => …)`; the helper does `BEGIN/COMMIT/ROLLBACK` and always releases the client.
- Stock changes only inside a transaction after `SELECT … FOR UPDATE` with rows locked in `ORDER BY product_id` order.
- Money: `NUMERIC` in SQL, strings in JSON, integer paise in JS arithmetic (`utils/money.js`). Never JS floats for money.
- Schema changes = new numbered file in `database/migrations/` (if needed after first deploy) **and** update ARCHITECTURE §5.

## 5. DynamoDB rules (MUST)
- Table names from env only.
- **No `Scan`** in request paths. GetItem/PutItem/UpdateItem/DeleteItem/Query only.
- Cart writes are conditional on `version` (optimistic locking) with ≤ 3 retries.
- Session validation checks `expires_at > now` in code (TTL deletes lazily).
- Keys are strings (`"101"`), even though SQL ids are integers — convert at the boundary in `cart/repo.js`.

## 6. Security rules (MUST)
- No secrets in git: `.env`, `*.pem`, `*.key` in `.gitignore`. Commit `.env.example` only.
- No AWS access keys on EC2 or in code; use the instance role. Local dev uses DynamoDB Local (dummy creds).
- RDS and Aurora connections use TLS (`ssl` option set; RDS with CA bundle).
- Passwords: bcrypt cost 10; never returned by any endpoint.
- JWT in httpOnly, `sameSite=lax` cookie; payload only `{sid, cid}`; 12 h expiry.
- Admin routes check `ADMIN_EMAILS`.
- Card numbers: keep only last 4 digits; mock gateway never stores full numbers.
- Don't record the AWS account ID, access keys, or `.env` contents on video (blur if visible).

## 7. AWS cost rules (MUST — these protect your credits)
| Never | Instead |
|---|---|
| Create a NAT gateway | Default VPC public subnet for EC2; Aurora express needs no VPC |
| Create a load balancer (ALB/NLB) | Single EC2 public IP, port 3000 |
| Enable RDS Multi-AZ or create RDS read replicas | Aurora is where we demonstrate HA |
| Choose RDS instance larger than `db.t4g.micro` | — |
| Create more than 2 Aurora instances (writer + 1 reader) | Free plan limit is 2 instances per account anyway |
| Set Aurora max capacity > 4 ACU | 2 ACU is plenty for the demo |
| Leave Aurora min capacity > 0 overnight | min 0 (auto-pause) except during recording |
| Allocate an Elastic IP | Use auto-assigned public IP (note it changes on stop/start) |
| Turn on Performance Insights paid retention / Enhanced Monitoring | Default free monitoring |
| Use DynamoDB provisioned capacity with auto scaling | On-demand |
| Leave EC2/RDS running when not working | Stop both at end of each day (RDS auto-restarts after 7 days — stop again) |
| Create resources in a second region | One region only, recorded in MEMORY |
- Every resource MUST carry tags `Project=ShopSphere`, `Owner=<A|B>` so teardown can find them (Resource Groups & Tag Editor).
- Every resource created MUST be added to `MEMORY.md → Resource Inventory` the same day.

## 8. Git rules
- Branches: `main` (deployable), `feat/<task-id>-short-name`. PR (or at least a self-review) before merge.
- Conventional commits: `feat(cart): optimistic locking on cart writes`.
- **Both members commit their own work from their own accounts** — commit history is evidence of a two-person contribution.
- Never commit `frontend/dist` to `main`.

## 9. Definition of Done (per task)
- [ ] Meets the acceptance criteria in PRD for the story.
- [ ] Works against docker-compose locally; tests added/updated and passing.
- [ ] Response carries correct data-source headers (if it touches a store).
- [ ] No new lint errors; no secrets in diff.
- [ ] Docs updated if schema/API/env changed; MEMORY session entry written.
- [ ] If AWS-facing: verified on EC2 and noted in MEMORY.
