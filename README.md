# ShopSphere — Multi-Database E-Commerce on AWS (Set-E)

> One application, three AWS databases, each used where it genuinely fits:
> **Amazon RDS for PostgreSQL** (orders system of record) → **Amazon Aurora PostgreSQL Serverless** (festival-sale scale with writer/reader split) → **Amazon DynamoDB** (cart, sessions, preferences).

Team of 2 · Target video length 15–17 min · Target AWS spend: **$0 out of pocket** (Free plan credits), < $10 of credits consumed.

---

## 1. Documentation set (read in this order)

| # | File | Purpose | Primary reader | Updated when |
|---|------|---------|----------------|--------------|
| 1 | `docs/PRD.md` | *What* we build and *why*; user stories, acceptance criteria, rubric mapping | Both + agent | Scope changes only |
| 2 | `docs/ARCHITECTURE.md` | *How* it fits together: components, schema DDL, DynamoDB keys, read/write routing, security, flows | Both + agent | Any structural decision |
| 3 | `docs/DESIGN.md` | UI/UX spec, design tokens, page inventory, **API contract**, "data-source badge" spec | Both + agent | UI/API changes |
| 4 | `docs/RULES.md` | Non-negotiable coding, security, **AWS cost** and agent-workflow rules | Agent (every session) | Rarely |
| 5 | `docs/PHASES.md` | Phase-by-phase plan, task owners, exit criteria, timeline | Both | Daily (tick boxes) |
| 6 | `docs/MEMORY.md` | Living state: decisions log, AWS resource inventory, gotchas, session log | Agent (start & end of every session) | Every session |
| 7 | `docs/AWS_SETUP_GUIDE.md` | Zero-to-running AWS setup at the lowest possible cost, plus teardown | Both | When AWS steps change |
| 8 | `docs/DEMO_SCRIPT.md` | Minute-by-minute video run-of-show split between both members | Both | Before recording |
| — | `proposal/proposal.tex` | Formal solution proposal for faculty (compile with `pdflatex` twice) | Faculty | Once |

Files 1–6 are your standard hackathon set. Files 7–8 are added because this assignment is graded on **a live AWS demo video**, so the cloud setup and the recording are first-class deliverables, not afterthoughts.

---

## 2. Repository structure (target)

```
shopsphere/
├── README.md                       # this file
├── docker-compose.yml              # LOCAL ONLY: postgres:17 + amazon/dynamodb-local (free dev loop)
├── .gitignore                      # must include .env, node_modules, dist, *.pem
│
├── docs/                           # the 8 planning docs above
│   ├── PRD.md  ARCHITECTURE.md  DESIGN.md  RULES.md
│   ├── PHASES.md  MEMORY.md  AWS_SETUP_GUIDE.md  DEMO_SCRIPT.md
│   └── diagrams/                   # exported PNG/SVG of architecture + ERD for slides/report
│
├── proposal/
│   └── proposal.tex                # faculty proposal (LaTeX)
│
├── database/
│   ├── schema.sql                  # PostgreSQL DDL (same file for RDS, Aurora, local)
│   ├── seed.sql                    # customers from 101, products P100…, sample orders
│   ├── demo_queries.sql            # the exact SQL shown in the video (incl. the 2 queries from the brief)
│   ├── grants_app_user.sql         # least-privilege app role (+ rds_iam on Aurora)
│   └── dynamodb/
│       ├── create-tables.sh        # AWS CLI: Carts, UserPreferences, Sessions (+TTL)
│       └── sample-cart.json        # the exact cart item from the brief (customer 101)
│
├── backend/                        # Node.js 22 + Express 5 (ES modules)
│   ├── package.json
│   ├── .env.example                # every variable documented, no real values
│   └── src/
│       ├── server.js               # starts HTTP server
│       ├── app.js                  # express app, middleware, routes, serves frontend/dist
│       ├── config/env.js           # zod-validated env loader
│       ├── db/
│       │   ├── sql/pools.js        # writer/reader pools; DB_MODE = rds | aurora
│       │   ├── sql/auroraAuth.js   # IAM auth token via @aws-sdk/rds-signer
│       │   ├── sql/tx.js           # withTransaction(fn) helper
│       │   ├── sql/nodeInfo.js     # which instance served this query (for badges)
│       │   └── dynamo/client.js    # DynamoDBDocumentClient (local endpoint aware)
│       ├── modules/                # one folder per domain: routes.js + service.js (+ repo.js)
│       │   ├── auth/               # register, login, logout, me  (RDS + DynamoDB Sessions)
│       │   ├── products/           # list, search, detail          (reader)
│       │   ├── cart/               # get, add, update, remove      (DynamoDB Carts)
│       │   ├── orders/             # checkout txn, list, detail, track (writer)
│       │   ├── payments/           # mock gateway + payment txn    (writer)
│       │   ├── preferences/        # theme, categories, recently viewed (DynamoDB)
│       │   └── admin/              # db-mode, status advance, insights, reset-demo
│       ├── middleware/             # auth.js, dataSource.js (X-Data-Source headers), error.js, rateLimit.js
│       └── utils/                  # logger.js, money.js, ids.js
│   └── tests/                      # vitest + supertest against docker-compose stack
│
├── frontend/                       # React 18 + Vite (built to static files served by Express)
│   ├── index.html  vite.config.js  package.json
│   └── src/
│       ├── main.jsx  App.jsx  routes.jsx
│       ├── api/client.js           # fetch wrapper; reads X-Data-Source / X-Served-By headers
│       ├── components/             # Navbar, ProductCard, CartDrawer, DataSourceBadge, StatusTimeline …
│       ├── pages/                  # Catalog, Product, Cart, Checkout, Payment, Orders, OrderTrack,
│       │                           # Login, Register, Profile, Admin, Insights
│       └── styles/tokens.css       # design tokens from DESIGN.md
│
├── scripts/
│   ├── ec2-userdata.sh             # installs git, psql client, node 22 (nvm), pm2, swap
│   ├── deploy.sh                   # git pull → npm ci → build → pm2 restart
│   ├── migrate-rds-to-aurora.sh    # pg_dump (RDS) | psql (Aurora, IAM token)
│   ├── load-test.sh                # autocannon "festival sale" traffic
│   └── teardown-checklist.md       # what to delete, in order
│
└── infra/
    └── iam/
        ├── ec2-app-policy.json     # DynamoDB (ShopSphere_* only) + rds-db:connect (Aurora)
        └── trust-ec2.json
```

---

## 3. How to use these docs with a coding agent

Start every agent session with:

```
Read docs/RULES.md and docs/MEMORY.md fully, then the sections of docs/ARCHITECTURE.md,
docs/DESIGN.md and docs/PRD.md relevant to task <PHASE.TASK-ID> from docs/PHASES.md.
Implement only that task. When done: run tests, then append a session entry to
docs/MEMORY.md (what changed, decisions, open issues) and tick the task in docs/PHASES.md.
```

End every session by checking that `MEMORY.md → Current Status` names the next task.

---

## 4. Quick start (local, costs nothing)

```bash
docker compose up -d                       # postgres :5432, dynamodb-local :8000
psql postgresql://postgres:postgres@localhost:5432/shopsphere -f database/schema.sql
psql postgresql://postgres:postgres@localhost:5432/shopsphere -f database/seed.sql
bash database/dynamodb/create-tables.sh --local
cd backend && cp .env.example .env && npm i && npm run dev
cd ../frontend && npm i && npm run dev     # http://localhost:5173
```

Only move to AWS once the app works end-to-end locally (see `docs/PHASES.md`, Phase 3).
