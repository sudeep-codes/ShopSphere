# PHASES — Delivery Plan (9 working days, 2 people)

**Strategy:** build and test everything **locally for free** (Phases 1–2), then open a short **AWS window** (Phases 3–7, ~3 days) utilizing the $100 free AWS credit. The goal is to strictly stay within the credit limits and incur $0 of out-of-pocket spending, recording and tearing down the same week.

Owners: **A** = Member A (Relational & Data Lead) · **B** = Member B (Application, NoSQL & Cloud Lead) · **AB** = pair.
Estimated effort ≈ 40 h each.

| Part | Phase | Days | Theme | AWS cost running? |
|---|---|---|---|---|
| **Prerequisite** | P0 | D1 | Kick-off, repo, AWS account guardrails | No (budgets only) |
| **Part 1: Development** | P1 | D1–D2 | Local foundation | No |
| | P2 | D2–D5 | Core features (local) | No |
| | P3 | D6 | AWS baseline: DynamoDB, RDS, EC2 | Yes |
| **Part 2: AWS Config** | P4 | D6–D7 | Aurora festival mode | Yes |
| | P5 | D7 | Insights, comparison, dashboard | Yes |
| | P6 | D8 | Hardening & rehearsal | Yes (stop overnight) |
| | P7 | D8–D9 | Recording + proposal | Yes |
| **Part 3: Finalization**| P8 | D9 | Bringing everything together & Teardown | → $0 |

---

## P0 — Kick-off & guardrails (D1)
| ID | Task | Owner | Est |
|---|---|---|---|
| P0.1 | Create GitHub repo, add these docs, `.gitignore`, branch rules | B | 0.5 h |
| P0.2 | Create AWS account (w/ $100 free credit), root MFA, two IAM admin users with MFA (`AWS_SETUP_GUIDE` §1–2) | B | 1 h |
| P0.3 | Budgets: set strict $0 out-of-pocket limit (track against $100 credit), email alerts (`AWS_SETUP_GUIDE` §3) | B | 0.5 h |
| P0.4 | Pick region, record it in MEMORY | AB | 0.1 h |
| P0.5 | Read brief together; agree comparison table wording (PRD §9) | AB | 0.5 h |
**Exit:** both can log in to AWS with MFA; budget alert emails confirmed; repo has docs.

# Part 1: Development

## P1 — Local foundation (D1–D2)
| ID | Task | Owner | Est |
|---|---|---|---|
| P1.1 | `docker-compose.yml`: `postgres:17`, `amazon/dynamodb-local` | B | 0.5 h |
| P1.2 | `database/schema.sql` exactly per ARCHITECTURE §5 | A | 1 h |
| P1.3 | `database/seed.sql` (customers 101–103, products P100–P111 + P200–P211, 3 orders for 101, low-stock P107) | A | 1.5 h |
| P1.4 | `database/demo_queries.sql` (brief's 2 queries + 6 more, see DEMO_SCRIPT) | A | 1 h |
| P1.5 | `database/dynamodb/create-tables.sh` (`--local` flag) + `sample-cart.json` | B | 1 h |
| P1.6 | Backend skeleton: env loader (zod), logger, error handler, `/api/health` | B | 2 h |
| P1.7 | `db/sql/pools.js`, `tx.js`, `nodeInfo.js` with `db.read/write/tx` (rds mode only for now) | A | 2.5 h |
| P1.8 | `db/dynamo/client.js` + `dataSource.js` middleware (headers) | B | 1.5 h |
| P1.9 | Frontend skeleton: Vite, router, tokens.css, Navbar, `DataSourceBadge`, api client reading headers | B | 3 h |
**Exit:** `/api/health` returns `{ok:true, mode:"rds"}` with headers; badge renders from a dummy call.

## P2 — Core features, local (D2–D5)
| ID | Task | Owner | Est |
|---|---|---|---|
| P2.1 | Auth: register/login/logout/me, bcrypt, JWT cookie, DynamoDB sessions + TTL attr | B | 4 h |
| P2.2 | Products: list/filter/search/sort/paginate/detail/categories via `db.read` | A | 3 h |
| P2.3 | Cart: DynamoDB CRUD with optimistic locking, enrich with live prices | B | 4 h |
| P2.4 | Checkout transaction (FOR UPDATE, rollback on stock), clear cart after commit | A | 4 h |
| P2.5 | Mock gateway + payment transaction + idempotency; decline on `…0002` | A | 3 h |
| P2.6 | Orders list, detail, status timeline; cancel with stock restore | A | 3 h |
| P2.7 | Preferences + recently viewed | B | 2 h |
| P2.8 | Pages: Catalog, Product, Cart drawer, Checkout, Payment, Orders, Tracking, Login/Register, Profile | B | 8 h |
| P2.9 | Admin page: status advance, demo reset | A (API) / B (UI) | 3 h |
| P2.10 | Tests: auth, checkout rollback, payment idempotency, cart conflict | AB | 4 h |
**Exit:** full customer journey works locally; tests green; demo reset returns DB + DynamoDB to seed state.

## P3 — AWS baseline (D6)  ⏱ AWS meter starts
| ID | Task | Owner | Est |
|---|---|---|---|
| P3.1 | DynamoDB tables + TTL on Sessions (`AWS_SETUP_GUIDE` §5) | B | 0.5 h |
| P3.2 | Security groups `shopsphere-app-sg`, `shopsphere-rds-sg` (§6) | B | 0.3 h |
| P3.3 | IAM role `ShopSphereEC2Role` + inline policy (§7) | B | 0.5 h |
| P3.4 | RDS PostgreSQL `db.t4g.micro` Single-AZ, private (§8) | A | 0.5 h (+15 min wait) |
| P3.5 | EC2 launch with user-data, Session Manager access (§9) | B | 0.5 h |
| P3.6 | On EC2: psql to RDS, run schema + seed + grants, run demo queries (§10) | A | 1 h |
| P3.7 | Deploy app with pm2, open `http://<ip>:3000`, smoke test all flows in RDS mode (§11) | B | 1.5 h |
| P3.8 | Fill MEMORY Resource Inventory | AB | 0.2 h |
**Exit:** whole journey works on AWS in RDS mode; brief's SQL queries return app-created orders.

# Part 2: AWS Configuration

## P4 — Aurora festival mode (D6–D7)
| ID | Task | Owner | Est |
|---|---|---|---|
| P4.1 | Create Aurora PostgreSQL (express), capacity 0–2 ACU; add reader instance (§12) | A | 0.5 h |
| P4.2 | Update IAM policy with cluster resource id (`rds-db:connect`) | B | 0.3 h |
| P4.3 | CloudShell: create DB `shopsphere`, `app_user` + `rds_iam` + grants | A | 0.5 h |
| P4.4 | `scripts/migrate-rds-to-aurora.sh` (pg_dump RDS → psql Aurora with IAM token) + run | A | 1.5 h |
| P4.5 | `auroraAuth.js` + aurora pools in `pools.js`; runtime mode switch endpoint | A | 2.5 h |
| P4.6 | Verify routing: catalog badge = reader instance, orders = writer; reader rejects INSERT in CloudShell | A | 0.5 h |
| P4.7 | `scripts/load-test.sh` (autocannon) + CloudWatch dashboard `ShopSphere-Festival` | B | 1.5 h |
| P4.8 | Failover drill: Actions → Failover; app recovers < 60 s; note timings in MEMORY | A | 0.5 h |
**Exit:** mode switch works both ways; load test shows reader capacity rising; failover recovered.

## P5 — Insights & comparison (D7)
| ID | Task | Owner | Est |
|---|---|---|---|
| P5.1 | `/api/admin/insights` (20 samples each store, median) | A | 1.5 h |
| P5.2 | Insights page (recharts bar + recent requests + mini comparison table) | B | 2 h |
| P5.3 | Final comparison table text + "evidence" column linking to demo moments | AB | 1 h |
**Exit:** insights page renders real numbers from AWS.

## P6 — Hardening & rehearsal (D8)
| ID | Task | Owner | Est |
|---|---|---|---|
| P6.1 | Edge cases: expired session, cart conflict, decline then retry, cancel restore stock | AB | 2 h |
| P6.2 | Pre-recording checklist from DEMO_SCRIPT; demo reset; Aurora min 0.5 ACU | AB | 0.5 h |
| P6.3 | Full timed rehearsal ×2 (target 15–17 min), trim | AB | 2 h |
| P6.4 | Stop EC2 + RDS overnight; Aurora back to min 0 | B | 0.1 h |

## P7 — Recording & proposal (D8–D9)
| ID | Task | Owner | Est |
|---|---|---|---|
| P7.1 | Record (OBS: screen + both webcams), follow DEMO_SCRIPT | AB | 2 h |
| P7.2 | Light edit (cuts only during long waits), export 1080p, check audio | B | 1.5 h |
| P7.3 | Fill names/roll numbers in `proposal/proposal.tex`, add screenshots, compile PDF | A | 1.5 h |

# Part 3: Bringing Everything Together

## P8 — Bringing everything together / Teardown (D9)  ⏱ AWS meter stops
| ID | Task | Owner | Est |
|---|---|---|---|
| P8.1 | Run teardown checklist (`AWS_SETUP_GUIDE` §17) in order | AB | 0.5 h |
| P8.2 | Tag Editor search `Project=ShopSphere` → nothing left; check snapshots, EBS, EIPs, log groups | B | 0.2 h |
| P8.3 | Next day: Billing → Bills, confirm no running charges; record final credit usage in MEMORY | B | 0.1 h |

---

## Contribution balance (for the proposal & viva)

| Area | Member A | Member B |
|---|---|---|
| AWS services owned | RDS, Aurora, migration, failover | DynamoDB, EC2, IAM, Budgets, CloudWatch |
| Backend | pools/routing, products, orders, payments, insights API | auth/sessions, cart, preferences, middleware |
| Frontend | admin API hooks | all pages & components |
| Docs | ARCHITECTURE §5–9, proposal | PRD, DESIGN, AWS guide, demo script |
| Video | Segments 4, 6, 7 (~8 min) | Segments 2, 3, 5, 8 (~8 min) |
