# PRD — ShopSphere: Multi-Database E-Commerce on AWS

| Field | Value |
|---|---|
| Assignment | Set-E — "Develop the E-Commerce use case on AWS and demonstrate it" (20 marks) |
| Team | 2 members — **Member A** (Relational & Data Lead), **Member B** (Application, NoSQL & Cloud Lead) |
| Deliverables | Working app on AWS · 10–20 min video (both faces + console) · RDS/Aurora/DynamoDB comparison · faculty proposal |
| Status | Draft v1.0 — see `MEMORY.md` for live state |

---

## 1. Problem statement

An online shop needs to let customers **register/log in, browse products, add to cart, place orders, pay and track orders**. Different parts of this data have very different shapes and traffic patterns:

- Customers → Orders → Order Items → Products is **relational** with integrity constraints and money → needs SQL, JOINs, ACID transactions.
- During a **festival sale** traffic jumps ~100× (1,000 → 100,000+ users), overwhelmingly **reads** (browsing) → needs read scaling and high availability.
- The **shopping cart, sessions and preferences** are per-user key-value/document data accessed by one key, at very high frequency → needs fast key lookups, flexible schema, serverless scale.

The assignment asks us to show *why* each of RDS, Aurora and DynamoDB fits a part of this problem, not just that each one can be created.

## 2. Product vision

**One working shop, three databases, every screen proves which database served it.**
Each API response carries a data-source tag (e.g. `Aurora · reader · instance-2 · 4 ms`) that the UI shows as a badge, so the evaluator can *see* read/write splitting and NoSQL access in the running app, then verify it in the AWS console.

## 3. Goals and success metrics

| ID | Goal | Metric / evidence |
|---|---|---|
| G1 | All six customer capabilities work end-to-end on AWS | Each flow demonstrated live in video, no mocked screens |
| G2 | RDS used as relational system of record | The two SQL queries from the brief run unchanged against RDS and return app-created data |
| G3 | Aurora shows *why* it beats RDS at festival scale | Writer/reader in different AZs; browse traffic served by reader; capacity scales during load test; manual failover recovers |
| G4 | DynamoDB used for cart/session/preferences | Cart item in console has exactly the brief's JSON shape; GetItem by `customer_id` |
| G5 | Clear, justified comparison | Completed comparison table (§9) with demo evidence per row |
| G6 | Near-zero cost | ≤ $10 of credits consumed, $0 billed; all resources deleted after recording |
| G7 | Balanced team contribution | Each member presents ~50 % of the video and owns ~50 % of commits |

## 4. Non-goals (explicitly out of scope)

Real payment gateway (we use a deterministic **mock gateway**) · real shipping/couriers · email/SMS notifications · product image uploads (static image URLs) · multi-region · CDN/WAF · native mobile app · recommendation engine · production-grade CI/CD.

## 5. Personas

- **Customer (Priya, customer_id 101)** — shops on a phone/laptop, expects a fast catalog and a cart that never loses items.
- **Store admin (team)** — advances order status, switches DB mode, runs the festival simulation.
- **Evaluator (faculty)** — wants to see each service configured correctly in the console and understand the trade-offs.

## 6. User stories & acceptance criteria

### Epic 1 — Register & log in (RDS + DynamoDB)
- **US-01** As a visitor I can register with name, email, phone, password.
  - AC: email unique (409 on duplicate); password stored only as a bcrypt hash in `customers`; new customers get ids from **101** upward.
- **US-02** As a customer I can log in and stay logged in.
  - AC: on login a session item is written to DynamoDB `ShopSphere_Sessions` with a TTL (`expires_at`); JWT in an httpOnly cookie carries only `session_id` + `customer_id`.
- **US-03** As a customer I can log out.
  - AC: session item deleted; requests with the old cookie get 401 (proves server-side revocation).

### Epic 2 — Browse products (RDS → Aurora reader)
- **US-04** Browse a paginated catalog, filter by category, search by name, sort by price.
  - AC: p95 < 300 ms at 50 concurrent users on AWS; in Aurora mode served by the **reader endpoint** (badge shows the reader instance id).
- **US-05** View product detail with live stock.
  - AC: viewing a product appends it to `recently_viewed` (max 10) in DynamoDB preferences.

### Epic 3 — Cart (DynamoDB)
- **US-06** Add, change quantity, remove items; cart persists across devices/log-ins.
  - AC: one item per customer in `ShopSphere_Carts`, partition key `customer_id`, shape identical to the brief: `{customer_id, items:[{product_id, quantity}]}`; writes use optimistic locking (`version` attribute).
- **US-07** Cart shows live prices/stock from the relational catalog (cart stores ids + qty only).

### Epic 4 — Place order (RDS/Aurora writer, ACID)
- **US-08** Checkout converts the cart into an order.
  - AC: single transaction: lock product rows (`SELECT … FOR UPDATE`), verify stock, insert `orders` + `order_items`, decrement stock, insert status history `PLACED`; **all-or-nothing**. Insufficient stock → full rollback + clear error. Cart cleared only after commit.

### Epic 5 — Payment (mock gateway, writer)
- **US-09** Pay with Card/UPI/COD via the mock gateway.
  - AC: card ending `0002` → declined (order → `PAYMENT_FAILED`, can retry); success → `payments` row + order `PAID` in one transaction; a duplicate submit with the same `idempotency_key` never charges twice.

### Epic 6 — Track orders (writer, read-your-writes)
- **US-10** See my orders and a status timeline per order.
  - AC: timeline from `order_status_history`; statuses `PLACED → PAID → SHIPPED → OUT_FOR_DELIVERY → DELIVERED` (+ `CANCELLED`, `PAYMENT_FAILED`); always read from the writer so a just-placed order is visible immediately.
- **US-11** Admin advances an order's status; the customer page reflects it on refresh.

### Epic 7 — Demonstration features (what makes the video convincing)
- **US-12** Data-source badges on every page (store, role, instance id, latency).
- **US-13** Admin **DB mode switch** `RDS ⇄ Aurora` without redeploy.
- **US-14** **Festival simulation**: scripted load test while CloudWatch shows Aurora capacity/connection metrics.
- **US-15** **Insights page**: median latency of DynamoDB GetItem vs reader query vs writer query (20 samples each).
- **US-16** Admin "Reset demo data" restores seed state before recording.

## 7. Functional requirements summary

| FR | Requirement | Priority | Data store |
|---|---|---|---|
| FR-01 | Register / login / logout | P0 | `customers` (RDS/Aurora) + DynamoDB `Sessions` |
| FR-02 | Catalog list/search/filter/detail | P0 | RDS / Aurora **reader** |
| FR-03 | Cart CRUD | P0 | DynamoDB `Carts` |
| FR-04 | Checkout transaction | P0 | RDS / Aurora **writer** |
| FR-05 | Mock payment + idempotency | P0 | writer |
| FR-06 | Order list + tracking timeline | P0 | writer |
| FR-07 | Admin status advance | P0 | writer |
| FR-08 | Preferences + recently viewed | P1 | DynamoDB `UserPreferences` |
| FR-09 | Data-source badges | P0 (demo-critical) | all |
| FR-10 | DB mode switch | P0 (demo-critical) | — |
| FR-11 | Insights latency page | P1 | all |
| FR-12 | Reset demo data | P1 | all |
| FR-13 | "Log out all devices" via GSI on Sessions | P2 | DynamoDB |

## 8. Non-functional requirements

| Area | Requirement |
|---|---|
| Performance | Catalog p95 < 300 ms @ 50 concurrent; cart GetItem p95 < 50 ms server-side |
| Availability | Aurora writer + reader in **different AZs**; app recovers from a manual failover in < 60 s without restart |
| Consistency | Orders/payments ACID; user's own orders read from the writer (read-your-writes); catalog may be milliseconds stale on the reader |
| Security | No DB publicly reachable except Aurora's IAM-only internet access gateway; no AWS keys on the server (instance role); SQL fully parameterised; secrets never in git |
| Cost | Single-AZ RDS `db.t4g.micro`; Aurora serverless min 0 ACU (auto-pause), max ≤ 4 ACU; DynamoDB on-demand; no NAT gateway/ALB; full teardown |
| Observability | CloudWatch metrics shown live; structured JSON logs with request id + data source |
| Demo-ability | Every claim in the video is visible either as a UI badge or in the AWS console |

## 9. Database comparison deliverable (presented in video & proposal)

| Feature | Amazon RDS (PostgreSQL) | Amazon Aurora (PostgreSQL) | Amazon DynamoDB |
|---|---|---|---|
| Database type | Relational (managed open-source engine) | Relational, cloud-native distributed storage | NoSQL key-value & document |
| SQL | Full SQL | Full SQL (PostgreSQL/MySQL compatible) | No SQL engine; key-based API + PartiQL (SQL-like, no joins) |
| JOINs | Yes | Yes | No — model data by access pattern (denormalise) |
| Flexible schema | No — fixed schema, `ALTER TABLE` (JSONB helps) | No — same as RDS | Yes — only key attributes are fixed |
| Serverless options | No — you choose an instance size | Yes — Aurora Serverless (scales in ACUs, can pause at 0) | Yes — fully serverless, on-demand capacity |
| Very large scale | Moderate — vertical scaling + async read replicas | High — shared storage kept as 6 copies across 3 AZs, up to 15 low-lag readers | Very high — virtually unlimited throughput/storage at consistent ms latency |
| Transactions | Full ACID, multi-table | Full ACID, multi-table | ACID `TransactWriteItems` / `TransactGetItems` (up to 100 items) |
| Failover | Optional Multi-AZ standby (extra cost) | Fast promotion of a reader to writer | Built in, multi-AZ by default |
| Cost model | Instance-hours + storage (cheapest at small steady load) | ACU-hours (serverless) or instance-hours + storage + I/O | Per request (on-demand) or provisioned capacity + storage |
| **Best for (in ShopSphere)** | Customer & order management at normal traffic | Same relational data at festival scale; read-heavy spikes; HA | Cart, sessions, preferences — single-key, high-frequency access |

## 10. Assignment rubric mapping

| Brief requirement | How ShopSphere satisfies it | Shown in (DEMO_SCRIPT) |
|---|---|---|
| Register & log in | US-01..03 | Seg 3 |
| Browse products | US-04/05, reader endpoint | Seg 3, 6 |
| Add to cart | US-06, DynamoDB console | Seg 3 |
| Place orders | US-08 transaction | Seg 4 |
| Make payments | US-09 mock gateway | Seg 5 |
| Track orders | US-10/11 timeline | Seg 5 |
| RDS tables + example SQL | `customers/orders/products` + both queries from the brief run verbatim | Seg 4 |
| Aurora writer/reader, festival sale | DB mode switch, reader badge, load test, failover | Seg 6, 7 |
| DynamoDB cart keyed by customer id | Exact JSON from brief, GetItem, PartiQL | Seg 3 |
| Comparison table | §9 + Insights page | Seg 8 |
| Faces + audio, all members, 10–20 min | 16-min script, ~8 min each | whole video |

## 11. Assumptions & constraints

- A new AWS account on the **Free plan** (credits) is preferred. On the Free plan, Aurora PostgreSQL must be created with **express configuration** (not inside a VPC, IAM authentication only, ≤ 4 ACU, ≤ 1 GiB storage, ≤ 2 instances). The Paid-plan path is in `AWS_SETUP_GUIDE.md` Appendix B.
- PostgreSQL is used for *both* RDS and Aurora so the same schema and SQL run everywhere.
- One region for everything.
- Development happens locally (Docker) to minimise AWS running hours.

## 12. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Credits drained by forgotten resources | Bill / account closure | Budgets + daily stop routine + teardown checklist |
| Aurora auto-pause causes a slow first query on camera | Awkward demo | Raise min capacity to 0.5 ACU 15 min before recording |
| IAM token auth misconfigured | Aurora mode fails | Test from CloudShell first; unit-test `auroraAuth.js`; fallback: demonstrate Aurora via CloudShell SQL |
| t4g.micro runs out of memory while building | Deploy fails | 2 GB swap in user-data, or build frontend locally |
| Video over 20 min | Penalty | Rehearse with a timer; cut list in DEMO_SCRIPT |
