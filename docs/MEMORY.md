# MEMORY — Living Project State

> Agents: read this first, update it last. Humans: this is the single place to look for "where are we?".
> Append-only for the Decisions Log and Session Log (never rewrite history; supersede with a new entry).

## 1. Current status
| Field | Value |
|---|---|
| Current phase | P0 — Kick-off |
| Next task | P0.1 Create GitHub repo |
| Blockers | — |
| DB mode on AWS | n/a (not deployed) |
| Last updated | YYYY-MM-DD by A/B |

## 2. Fixed facts
| Key | Value |
|---|---|
| Project | ShopSphere (Set-E e-commerce) |
| AWS region | `ap-south-1` ← change once, at P0.4, then never |
| AWS plan | Free plan (credits) / Paid plan ← circle one |
| Account alias | ________ (never show account ID on video) |
| Repo | github.com/<org>/shopsphere |
| Team | A = ________ (Relational & Data Lead) · B = ________ (App, NoSQL & Cloud Lead) |
| Faculty / course | ________ |
| Video due | YYYY-MM-DD |

## 3. Decisions log (ADR-lite)
| ID | Date | Decision | Why | Alternatives |
|---|---|---|---|---|
| D-001 | init | PostgreSQL for RDS **and** Aurora | Aurora PostgreSQL is the Aurora flavour offered on the AWS Free plan (express configuration); one schema/SQL dialect everywhere | MySQL 8 |
| D-002 | init | Node 22 + Express 5 monolith on one EC2, React (Vite) built and served by Express | Simple for 2 people; one deploy; persistent pools | Lambda + API Gateway; separate S3 static site |
| D-003 | init | Mock payment gateway (decline on card ending 0002) | Deterministic demo; focus on transactions | Stripe/Razorpay test mode |
| D-004 | init | Own auth (bcrypt + JWT) with sessions in DynamoDB (TTL) | Customers live in relational schema as brief shows; extra genuine DynamoDB use case | Cognito |
| D-005 | init | DynamoDB on-demand, 3 tables (Carts, Sessions, UserPreferences) | Pay per request, no throttling in load test; clearer than single-table for evaluators | Single-table design; provisioned |
| D-006 | init | Runtime `DB_MODE` switch rds ⇄ aurora | Shows cut-over live without redeploy | Two deployments |
| D-007 | init | `product_id` is VARCHAR (`'P100'`) | Matches the brief's cart JSON and DynamoDB item exactly | INT ids |
| D-008 | init | `customer_id` identity starts at 101 | Brief's query `WHERE customer_id = 101` works on real data | — |
| D-009 | init | Local-first development (docker-compose) | Keeps AWS running hours to ~3 days | Develop on AWS |
| D-010 | init | No SSH; Session Manager for EC2 shell | No port 22 open; works in the browser for the video | SSH key pair |
| D-011 | init | Aurora: writer + 1 reader, serverless 0–2 ACU | Demonstrates HA + read scaling within Free-plan limits (≤ 4 ACU, ≤ 2 instances) | Provisioned instances |

## 4. AWS resource inventory (fill as you create — delete in reverse order)
| # | Resource | Name / ID | Endpoint / ARN / IP | Created by | Date | Deleted? |
|---|---|---|---|---|---|---|
| 1 | Budget | `shopsphere-zero-spend`, `shopsphere-monthly-5usd` | — | B | | ☐ (optional to keep) |
| 2 | DynamoDB table | `ShopSphere_Carts` | arn: | B | | ☐ |
| 3 | DynamoDB table | `ShopSphere_Sessions` (TTL `expires_at`) | arn: | B | | ☐ |
| 4 | DynamoDB table | `ShopSphere_UserPreferences` | arn: | B | | ☐ |
| 5 | Security group | `shopsphere-app-sg` | sg- | B | | ☐ |
| 6 | Security group | `shopsphere-rds-sg` | sg- | B | | ☐ |
| 7 | IAM role | `ShopSphereEC2Role` (+ instance profile) | arn: | B | | ☐ |
| 8 | RDS instance | `shopsphere-rds` (PostgreSQL __, db.t4g.micro) | host: | A | | ☐ |
| 9 | EC2 instance | `shopsphere-app` (t4g.____) | i- / public IP: | B | | ☐ |
| 10 | Aurora cluster | `shopsphere-aurora` (express) | writer: / reader: / resource id: cluster- | A | | ☐ |
| 11 | Aurora instance | writer `…-instance-1` (AZ __) | | A | | ☐ |
| 12 | Aurora instance | reader `…-reader-1` (AZ __) | | A | | ☐ |
| 13 | CloudWatch dashboard | `ShopSphere-Festival` | | B | | ☐ |

## 5. Non-secret runtime values
| Key | Value |
|---|---|
| RDS_HOST | |
| AURORA_WRITER_HOST | |
| AURORA_READER_HOST | |
| Aurora cluster resource id | `cluster-XXXXXXXX` (needed for IAM policy) |
| EC2 public IP (changes on stop/start!) | |
| App URL | `http://<ip>:3000` |
Secrets (RDS password, JWT secret) live **only** in `backend/.env` on EC2 and in the team password manager — never here.

## 6. Known gotchas (pre-filled — add your own)
1. Aurora express clusters accept **IAM auth only**; tokens last 15 min and are needed only to open a connection.
2. Aurora at 0 ACU is **paused**; the first connection waits while it resumes. Set min 0.5 ACU before recording; back to 0 after.
3. Open idle connections keep Aurora awake → keep `PG_IDLE_TIMEOUT_MS` short and stop pm2 when not testing.
4. A stopped RDS instance **restarts automatically after 7 days**; storage is billed even while stopped.
5. EC2 public IP changes after stop/start → update bookmarks / MEMORY.
6. PostgreSQL folds unquoted identifiers to lower case — the brief's `Customers`/`Orders` queries work; quoted `"Customers"` would not.
7. RDS PostgreSQL forces TLS; use `sslmode=verify-full sslrootcert=global-bundle.pem` with psql.
8. `pg_dump` client major version must be ≥ server version.
9. DynamoDB TTL deletion is not instant → always check `expires_at` in code.
10. On the Free plan, if a console option is greyed out/blocked, don't upgrade blindly — check the cost first and note it here.

## 7. Open questions
| # | Question | Owner | Status |
|---|---|---|---|
| Q1 | Does faculty want the source code submitted too? | A | open |
| Q2 | Video upload platform & size limit? | B | open |

## 8. Session log (append newest at bottom)
```
### YYYY-MM-DD HH:MM — <A|B|agent> — task <ID>
Done: …
Changed files: …
Decisions: (D-0xx if any)
Issues / follow-ups: …
Next: <task ID>
```
