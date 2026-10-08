# STRUCTURE — ShopSphere File Map

> One page to find anything in the repo. Paths are relative to the repo root and clickable on GitHub/VS Code.
> **Legend:** ✅ exists now · 🔨 to be created (phase/task ID from [`docs/PHASES.md`](docs/PHASES.md)) · 👤 A = Member A (Relational & Data) · 👤 B = Member B (App, NoSQL & Cloud)
> **Rule:** when you add, move or delete a file, update this page in the same commit (see [`docs/RULES.md`](docs/RULES.md) §1).

---

## 1. "I want to…" — quick navigation

| I want to… | Go to |
|---|---|
| Understand what we're building and why | [`docs/PRD.md`](docs/PRD.md) |
| See the architecture diagram, schema DDL, DynamoDB keys | [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) §2, §5, §6 |
| Check how reads/writes are routed (RDS vs Aurora reader/writer) | [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) §7 · code: [`backend/src/db/sql/pools.js`](backend/src/db/sql/pools.js) |
| Find an API endpoint's request/response format | [`docs/DESIGN.md`](docs/DESIGN.md) §5 |
| Find UI colours, fonts, page list | [`docs/DESIGN.md`](docs/DESIGN.md) §2–§4 · code: [`frontend/src/styles/tokens.css`](frontend/src/styles/tokens.css) |
| Know what I'm allowed / not allowed to do (code, security, AWS cost) | [`docs/RULES.md`](docs/RULES.md) |
| Know what to do next | [`docs/MEMORY.md`](docs/MEMORY.md) §1 → [`docs/PHASES.md`](docs/PHASES.md) |
| Find an AWS endpoint, ARN, IP or resource name | [`docs/MEMORY.md`](docs/MEMORY.md) §4–§5 |
| Set up / stop / delete AWS resources | [`docs/AWS_SETUP_GUIDE.md`](docs/AWS_SETUP_GUIDE.md) (§15 daily stop, §17 teardown) |
| Prepare or record the video | [`docs/DEMO_SCRIPT.md`](docs/DEMO_SCRIPT.md) |
| Run the SQL shown in the video | [`database/demo_queries.sql`](database/demo_queries.sql) |
| Edit the faculty proposal | [`proposal/proposal.tex`](proposal/proposal.tex) |
| Fix login / sessions | [`backend/src/modules/auth/`](backend/src/modules/auth/) |
| Fix the cart | [`backend/src/modules/cart/`](backend/src/modules/cart/) |
| Fix checkout / stock rollback | [`backend/src/modules/orders/service.js`](backend/src/modules/orders/service.js) |
| Fix payments | [`backend/src/modules/payments/`](backend/src/modules/payments/) |
| Fix the data-source badges | [`backend/src/middleware/dataSource.js`](backend/src/middleware/dataSource.js) + [`frontend/src/components/DataSourceBadge.jsx`](frontend/src/components/DataSourceBadge.jsx) |
| Switch RDS ⇄ Aurora | Admin page → [`backend/src/modules/admin/routes.js`](backend/src/modules/admin/routes.js) |
| Deploy to EC2 | [`scripts/deploy.sh`](scripts/deploy.sh) |
| Run the festival load test | [`scripts/load-test.sh`](scripts/load-test.sh) |

---

## 2. Full tree

```
shopsphere/
├── README.md                          ✅  project intro, doc index, quick start
├── STRUCTURE.md                       ✅  this file
├── docker-compose.yml                 🔨 P1.1  local postgres:17 + dynamodb-local
├── .gitignore                         🔨 P0.1  .env, node_modules, dist, *.pem
│
├── docs/                              ── planning & reference ──────────────────────
│   ├── PRD.md                         ✅  requirements, user stories, rubric mapping
│   ├── ARCHITECTURE.md                ✅  system, schema, DynamoDB, routing, security
│   ├── DESIGN.md                      ✅  UI tokens, pages, API contract
│   ├── RULES.md                       ✅  coding / security / cost / agent rules
│   ├── PHASES.md                      ✅  plan, task IDs, owners
│   ├── MEMORY.md                      ✅  live state, decisions, AWS inventory
│   ├── AWS_SETUP_GUIDE.md             ✅  low-cost AWS setup + teardown
│   ├── DEMO_SCRIPT.md                 ✅  16-min video run-of-show
│   └── diagrams/                      🔨 P7.3  architecture.png, erd.png, screenshots/
│
├── proposal/                          ── faculty submission ────────────────────────
│   ├── proposal.tex                   ✅  LaTeX source
│   └── proposal.pdf                   ✅  compiled preview
│
├── database/                          ── data layer (A owns SQL, B owns DynamoDB) ──
│   ├── schema.sql                     🔨 P1.2  PostgreSQL DDL (RDS, Aurora, local)
│   ├── seed.sql                       🔨 P1.3  customers 101–103, P100–P111, P200–P211
│   ├── demo_queries.sql               🔨 P1.4  Q1–Q8 used in the video
│   ├── grants_app_user.sql            🔨 P3.6  least-privilege role (RDS & Aurora)
│   ├── migrations/                    🔨 later only if schema changes after deploy
│   └── dynamodb/
│       ├── create-tables.sh           🔨 P1.5  3 tables + TTL (--local flag)
│       └── sample-cart.json           🔨 P1.5  the brief's cart for customer 101
│
├── backend/                           ── Node 22 + Express 5 API ───────────────────
│   ├── package.json                   🔨 P1.6
│   ├── .env.example                   🔨 P1.6  every env var, no real values
│   ├── src/
│   │   ├── server.js                  🔨 P1.6  starts HTTP server
│   │   ├── app.js                     🔨 P1.6  middleware, routes, serves frontend/dist
│   │   ├── config/
│   │   │   └── env.js                 🔨 P1.6  zod-validated env loader
│   │   ├── db/
│   │   │   ├── sql/
│   │   │   │   ├── pools.js           🔨 P1.7 / P4.5  db.read / db.write / db.tx, DB_MODE
│   │   │   │   ├── auroraAuth.js      🔨 P4.5  IAM token (rds-signer)
│   │   │   │   ├── tx.js              🔨 P1.7  withTransaction helper
│   │   │   │   └── nodeInfo.js        🔨 P1.7  which instance served the query
│   │   │   └── dynamo/
│   │   │       └── client.js          🔨 P1.8  DynamoDB DocumentClient
│   │   ├── modules/                   (each: routes.js → service.js → repo.js)
│   │   │   ├── auth/                  🔨 P2.1  register, login, logout, me
│   │   │   ├── products/              🔨 P2.2  catalog (reader)
│   │   │   ├── cart/                  🔨 P2.3  DynamoDB cart, optimistic locking
│   │   │   ├── orders/                🔨 P2.4 / P2.6  checkout txn, list, track, cancel
│   │   │   ├── payments/              🔨 P2.5  mockGateway.js + payment txn
│   │   │   ├── preferences/           🔨 P2.7  theme, categories, recently viewed
│   │   │   └── admin/                 🔨 P2.9 / P4.5 / P5.1  db-mode, advance, insights, reset
│   │   ├── middleware/
│   │   │   ├── auth.js                🔨 P2.1  JWT + DynamoDB session check
│   │   │   ├── dataSource.js          🔨 P1.8  X-Data-Source / X-Served-By headers
│   │   │   ├── error.js               🔨 P1.6  AppError → JSON envelope
│   │   │   └── rateLimit.js           🔨 P2.1  login rate limit
│   │   └── utils/
│   │       ├── logger.js              🔨 P1.6  pino JSON logs
│   │       ├── money.js               🔨 P2.4  paise arithmetic
│   │       └── ids.js                 🔨 P2.1  uuid / txn refs
│   └── tests/                         🔨 P2.10  auth, checkout rollback, idempotency, cart conflict
│
├── frontend/                          ── React 18 + Vite ───────────────────────────
│   ├── package.json  vite.config.js  index.html        🔨 P1.9
│   └── src/
│       ├── main.jsx  App.jsx  routes.jsx               🔨 P1.9
│       ├── api/client.js              🔨 P1.9  fetch wrapper, reads badge headers
│       ├── styles/tokens.css          🔨 P1.9  design tokens (DESIGN §2)
│       ├── components/                🔨 P1.9 / P2.8
│       │   ├── Navbar.jsx  Footer.jsx (DB mode + region)
│       │   ├── DataSourceBadge.jsx  ProductCard.jsx  CartDrawer.jsx
│       │   └── StatusTimeline.jsx  Toast.jsx  Skeleton.jsx
│       └── pages/                     🔨 P2.8 / P2.9 / P5.2
│           ├── Catalog.jsx  Product.jsx  Checkout.jsx  Payment.jsx
│           ├── Orders.jsx  OrderTrack.jsx  Login.jsx  Register.jsx  Profile.jsx
│           └── Admin.jsx  Insights.jsx
│
├── scripts/                           ── operations ────────────────────────────────
│   ├── ec2-userdata.sh                🔨 P3.5  git, psql, node 22, pm2, swap, RDS CA
│   ├── deploy.sh                      🔨 P3.7  pull → build → pm2 restart
│   ├── migrate-rds-to-aurora.sh       🔨 P4.4  pg_dump | psql with IAM token
│   ├── load-test.sh                   🔨 P4.7  autocannon festival traffic
│   └── teardown-checklist.md          🔨 P8.1  copy of AWS guide §17 to tick off
│
└── infra/
    └── iam/
        ├── ec2-app-policy.json        🔨 P3.3 / P4.2  DynamoDB + rds-db:connect
        └── trust-ec2.json             🔨 P3.3  EC2 trust policy
```

---

## 3. Folder guide

### `docs/` — read before coding
| File | Read it when… | Owner | Update frequency |
|---|---|---|---|
| [`PRD.md`](docs/PRD.md) | starting any feature; checking acceptance criteria | B | only on scope change |
| [`ARCHITECTURE.md`](docs/ARCHITECTURE.md) | touching DB, routing, security, env vars | A | on structural decisions |
| [`DESIGN.md`](docs/DESIGN.md) | building a page or endpoint | B | on UI/API change |
| [`RULES.md`](docs/RULES.md) | every agent session, every PR review | A + B | rarely |
| [`PHASES.md`](docs/PHASES.md) | picking the next task | A + B | daily |
| [`MEMORY.md`](docs/MEMORY.md) | start + end of every session | A + B | every session |
| [`AWS_SETUP_GUIDE.md`](docs/AWS_SETUP_GUIDE.md) | creating/stopping/deleting AWS resources | B | when steps change |
| [`DEMO_SCRIPT.md`](docs/DEMO_SCRIPT.md) | rehearsal and recording | A + B | before recording |

### `database/` — one schema, three targets
- `schema.sql` and `seed.sql` run **unchanged** on local Postgres, RDS and Aurora.
- SQL files are owned by 👤 A; `dynamodb/` is owned by 👤 B.
- Never edit `schema.sql` after first AWS deploy without also adding a file in `migrations/` and updating ARCHITECTURE §5.

### `backend/src/modules/` — one folder per domain
Every module has the same three files, so you always know where to look:

| File | Contains | Must not contain |
|---|---|---|
| `routes.js` | URL paths, zod validation, HTTP status codes | SQL or DynamoDB calls |
| `service.js` | business rules (stock checks, status transitions, idempotency) | `req` / `res` objects |
| `repo.js` | SQL via `db.read/write/tx`, DynamoDB calls | business decisions |

| Module | Data store | Owner | Endpoints (DESIGN §5) |
|---|---|---|---|
| `auth/` | writer + DynamoDB Sessions | B | `/api/auth/*` |
| `products/` | reader | A | `/api/products*` |
| `cart/` | DynamoDB Carts (+ reader for prices) | B | `/api/cart*` |
| `orders/` | writer (transactions) | A | `/api/orders*` |
| `payments/` | writer (transactions) | A | `/api/orders/:id/pay` |
| `preferences/` | DynamoDB UserPreferences | B | `/api/preferences` |
| `admin/` | all | A + B | `/api/admin/*` |

### `frontend/src/` — pages map 1:1 to routes
| Route | Page file | Main API calls |
|---|---|---|
| `/` | `pages/Catalog.jsx` | `GET /products` |
| `/product/:id` | `pages/Product.jsx` | `GET /products/:id`, `POST /cart/items` |
| (drawer) | `components/CartDrawer.jsx` | `GET/PATCH/DELETE /cart*` |
| `/checkout` | `pages/Checkout.jsx` | `POST /orders/checkout` |
| `/pay/:orderId` | `pages/Payment.jsx` | `POST /orders/:id/pay` |
| `/orders` | `pages/Orders.jsx` | `GET /orders` |
| `/orders/:id` | `pages/OrderTrack.jsx` | `GET /orders/:id` |
| `/login`, `/register` | `pages/Login.jsx`, `pages/Register.jsx` | `POST /auth/*` |
| `/profile` | `pages/Profile.jsx` | `GET/PUT /preferences` |
| `/admin` | `pages/Admin.jsx` | `/admin/db-mode`, `/admin/orders/:id/advance`, `/admin/reset-demo` |
| `/insights` | `pages/Insights.jsx` | `GET /admin/insights` |

### `scripts/` and `infra/` — run on EC2 or CloudShell, never commit secrets
| Script | Run where | When |
|---|---|---|
| `ec2-userdata.sh` | pasted into EC2 launch wizard | once, P3.5 |
| `deploy.sh` | EC2 (Session Manager) | after every `git push` you want live |
| `migrate-rds-to-aurora.sh` | EC2 | right before switching to Aurora |
| `load-test.sh` | EC2 | festival segment of the video |
| `infra/iam/*.json` | IAM console (paste) | P3.3, P4.2 |

---

## 4. Request path cheat-sheet (where a request travels in code)

```
Browser click
  → frontend/src/pages/<Page>.jsx
  → frontend/src/api/client.js                       (fetch, reads X-Data-Source headers)
  → backend/src/app.js                               (helmet, cors, cookies, logger)
  → backend/src/middleware/auth.js                   (JWT → DynamoDB session)
  → backend/src/modules/<domain>/routes.js           (validate)
  → backend/src/modules/<domain>/service.js          (rules)
  → backend/src/modules/<domain>/repo.js             (data access)
      ├→ backend/src/db/sql/pools.js → RDS | Aurora writer | Aurora reader
      └→ backend/src/db/dynamo/client.js → DynamoDB
  → backend/src/middleware/dataSource.js             (sets badge headers)
  → frontend/src/components/DataSourceBadge.jsx      (shows them)
```

---

## 5. Never commit

`backend/.env` · `*.pem` (RDS CA bundle lives in `~/certs/` on EC2) · `node_modules/` · `frontend/dist/` · DB dumps (`/tmp/shopsphere.sql`) · screenshots showing the AWS account ID.
