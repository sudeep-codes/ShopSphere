# DEMO SCRIPT — 16-minute video run-of-show

Rubric reminders: **faces visible**, **audio clear**, **both members present**, **10–20 min**, real **AWS Management Console**.
Target: **16:00** (buffer of ±3 min). Each member speaks ~8 min.

## Recording setup
- **OBS Studio** (free): Scene = full-screen browser + two webcam overlays (bottom-right A, bottom-left B). If recording remotely, one person shares screen in Google Meet/Teams and you record the call in gallery+screen layout.
- 1920×1080, 30 fps, mic close to mouth, test 20 s and listen back before the real take.
- Browser: 125 % zoom, tabs pre-opened in this order: App · App/Admin · App/Insights · RDS Databases · EC2 instance (Session Manager terminal already logged in) · DynamoDB tables · Aurora cluster (CloudShell ready) · CloudWatch dashboard · Budgets.
- Hide bookmarks bar; do-not-disturb on; no account ID visible.

## Pre-recording checklist (T-30 min)
- [ ] EC2 + RDS started; app `pm2 status` online; note new public IP
- [ ] Aurora min capacity **0.5 ACU**, both instances *Available*
- [ ] Admin → **Reset demo data**, DB mode = **RDS**
- [ ] Logged out in app (to show login live); cart for 101 = brief's JSON
- [ ] `database/demo_queries.sql` open in a second terminal pane
- [ ] Load test command typed but not run
- [ ] Phone timer visible to the speaker

---

## Segment table

| Seg | Time | Speaker | Screen | Content |
|---|---|---|---|---|
| 1 | 0:00–1:00 | A + B | Faces + title slide | Introductions (name, role). A: the problem and the 6 customer capabilities. B: "one app, three databases, each where it fits". |
| 2 | 1:00–2:30 | B | Architecture diagram → Budgets page | Walk the diagram left→right; why EC2 + default VPC; cost guardrails (budgets, no NAT/ALB), target ≈ $5 of credits. |
| 3 | 2:30–5:00 | B | App + DynamoDB console | Register → login → show **Sessions** item with `expires_at` TTL. Browse catalog (badge `RDS`). Add 2×P100 + 1×P200 → DynamoDB **Explore items** shows the brief's exact JSON for customer 101 → **PartiQL** `SELECT … WHERE customer_id='101'`. Explain partition key = direct lookup, no JOIN. |
| 4 | 5:00–8:00 | A | RDS console → EC2 Session Manager (psql) → App | RDS config: PostgreSQL, db.t4g.micro, Single-AZ, **Public access: No**, SG only from app. psql: `\dt`, then the **two queries from the brief verbatim**. Place an order in the app → re-run query, new row appears. **Rollback demo**: add 3× low-stock P107 → checkout fails → query shows stock unchanged and no partial order. |
| 5 | 8:00–9:15 | B | App (payment, tracking) + psql | Pay with card …0002 → declined, status PAYMENT_FAILED; retry with good card → PAID (`txn_ref`). Admin advances SHIPPED → OUT_FOR_DELIVERY; customer timeline updates. A runs the status-history query. |
| 6 | 9:15–12:45 | A | Aurora console → CloudShell → App → CloudWatch | Festival story (1,000 → 100,000 users). Cluster: writer + reader in **different AZs**, serverless 0–2 ACU, shared storage (6 copies / 3 AZs). CloudShell on **reader**: `pg_is_in_recovery()` = true, INSERT rejected. Run migration script (pre-run; show output). Admin → switch to **Aurora**: catalog badge = **reader-1**, checkout = **writer**. B starts load test; dashboard shows reader capacity/connections rising, writer flat. |
| 7 | 12:45–13:45 | A | Aurora console → App | **Failover**: Actions → Failover; roles swap; refresh app → writer badge shows the other instance; orders still work. |
| 8 | 13:45–15:15 | B | Insights page → comparison slide | Live median latencies (DynamoDB GetItem vs reader vs writer). Walk the comparison table row by row, pointing back to what was shown. |
| 9 | 15:15–16:00 | A + B | Faces + Billing/Credits page | Credits used so far; teardown plan; what we'd add in production (Multi-AZ, CloudFront, Cognito, real gateway). Each states their contribution in one sentence. Thank you. |

## Talking points (keep it natural, don't read)
- **RDS:** "Orders, items and payments must stay consistent — a payment without an order is a bug. That's ACID and foreign keys, so a relational engine. RDS gives us managed PostgreSQL for a few cents an hour."
- **Aurora:** "Same schema, same SQL — we just moved the data. What changes is the storage layer and the topology: readers share the writer's storage, so they don't copy data and lag is tiny. Browsing goes to readers, buying goes to the writer."
- **DynamoDB:** "A cart is read and written on almost every click, always by one key — the customer id. No joins, flexible items, and it scales without us choosing a server size."
- **Why not one database?** "We could put the cart in Postgres, but every click would hit the same instance that's processing payments during the sale."

## `database/demo_queries.sql`
```sql
-- Q1 (from the brief)
SELECT * FROM Orders WHERE customer_id = 101;

-- Q2 (from the brief)
SELECT Customers.name, Orders.order_id, Orders.total_amount
FROM Customers JOIN Orders ON Customers.customer_id = Orders.customer_id;

-- Q3 order with its items and products (Customers → Orders → Order Items → Products)
SELECT o.order_id, c.name, p.product_name, oi.quantity, oi.unit_price,
       oi.quantity * oi.unit_price AS line_total, o.status
FROM orders o
JOIN customers c   ON c.customer_id = o.customer_id
JOIN order_items oi ON oi.order_id = o.order_id
JOIN products p    ON p.product_id = oi.product_id
WHERE o.customer_id = 101
ORDER BY o.order_date DESC;

-- Q4 stock check for the rollback demo
SELECT product_id, product_name, stock FROM products WHERE product_id = 'P107';

-- Q5 tracking timeline
SELECT status, note, changed_at FROM order_status_history
WHERE order_id = (SELECT max(order_id) FROM orders WHERE customer_id = 101)
ORDER BY changed_at;

-- Q6 payments for the latest order (shows FAILED then SUCCESS)
SELECT payment_id, method, status, txn_ref, amount, created_at FROM payments
WHERE order_id = (SELECT max(order_id) FROM orders WHERE customer_id = 101);

-- Q7 revenue by category (reporting query — runs on the reader in Aurora mode)
SELECT p.category, sum(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi JOIN products p ON p.product_id = oi.product_id
JOIN orders o ON o.order_id = oi.order_id
WHERE o.status IN ('PAID','SHIPPED','OUT_FOR_DELIVERY','DELIVERED')
GROUP BY p.category ORDER BY revenue DESC;

-- Q8 Aurora only: which instance am I on?
SELECT aurora_db_instance_identifier() AS instance, pg_is_in_recovery() AS is_reader;
```

## If something breaks on camera
| Problem | Recovery line + action |
|---|---|
| Aurora slow first query | "That's Aurora resuming from zero capacity — the cost-saving feature." Wait; continue. |
| Failover takes long | Cut the wait in editing; keep the before/after. |
| CloudWatch graph flat | Metrics lag 1–3 min; switch to the badges, return to the graph at the end of the segment. |
| App error | Show `pm2 logs`, restart, re-take only that segment (record segments separately and join). |

## Cut list if over 20 min
1. Shorten Seg 2 to the diagram only (−45 s). 2. Skip Q3/Q7 (−40 s). 3. Skip preferences/recently viewed mention (−20 s). Never cut: brief's two queries, DynamoDB cart JSON, reader vs writer proof, comparison table, both faces.
