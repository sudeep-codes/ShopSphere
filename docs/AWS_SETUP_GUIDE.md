# AWS SETUP GUIDE — From zero to a running ShopSphere at the lowest possible cost

> Written for October 2026 AWS console/pricing. AWS changes console labels often — if a button name differs, look for the closest equivalent and note it in `MEMORY.md → Known gotchas`.
> Prices below are **approximate** on-demand rates; always confirm in the console's cost estimate panel or the AWS Pricing Calculator.

---

## 0. Cost strategy in one minute

**Golden rules**
1. **Build locally first** (Docker, free). Open AWS only for the last ~3 days.
2. **Use a new AWS account on the Free plan.** New accounts get $100 in credits at sign-up and can earn up to $100 more; on the Free plan, usage is paid from those credits and you are not billed. The Free plan lasts up to 6 months or until credits run out.
3. **No NAT gateway, no load balancer, no Multi-AZ RDS, no Elastic IP.** These are the classic student-bill killers.
4. **Stop what you're not using** every evening; **delete everything** right after recording.
5. **One region** for everything.

**Expected consumption (worst case: everything left on for 3 days)**

| Resource | Rate (approx.) | 3-day cost |
|---|---|---|
| EC2 `t4g.micro` + public IPv4 + 10 GB gp3 | ~$0.0084/h + $0.005/h + tiny | ≈ $1.00 |
| RDS PostgreSQL `db.t4g.micro` Single-AZ + 20 GB | ~$0.016–0.02/h + storage | ≈ $1.40 |
| Aurora PostgreSQL serverless, writer + reader at 0.5 ACU when active (paused at 0 otherwise) | ~$0.12 per ACU-hour (us-east-1) | ≈ $1–2 for ~10 active hours |
| DynamoDB on-demand, 3 small tables | per request; 25 GB storage always free | < $0.05 |
| CloudWatch basic metrics + 1 dashboard | free tier | $0 |
| **Total** | | **≈ $3–6 of credits, $0 billed** |

Regional prices differ (e.g. Mumbai is a little higher than N. Virginia). Either is fine; pick the region closest to you and stick to it.

---

## 1. Create the AWS account (Member B, with A watching)

1. Go to `aws.amazon.com` → **Create an AWS account**. Use a **team email** (e.g. a new Gmail) that both can access — not a personal one you'll lose.
2. When asked to choose a plan, choose **Free plan**.
   - Free plan: no charges; account closes when credits run out or after 6 months (data kept for a grace period).
   - Paid plan: same credits, but anything beyond them is billed to your card. Use only if your faculty requires a service the Free plan blocks.
3. Complete card + phone verification (a small temporary authorisation may appear on the card).
4. Sign in as **root** once.

> **Using an older account (created before 15 July 2025) or AWS Academy Learner Lab?** See Appendix B/C — the steps are the same but limits differ.

## 2. Secure the account and create your two logins

1. **Root MFA:** top-right account menu → *Security credentials* → *Assign MFA device* → authenticator app.
2. **Account alias** (nicer sign-in URL): IAM → Dashboard → *Account alias* → `shopsphere-<team>`.
3. **Two IAM users** (one per member — this also gives an audit trail of who created what):
   - IAM → Users → *Create user* → `memberA-admin` → ✅ Provide console access → custom password → attach policy **AdministratorAccess** → create. Repeat for `memberB-admin`.
   - Each member signs in with their user and **adds their own MFA**.
4. From now on **never use root** except for billing/account settings.
5. Choose the **region** (top-right) — e.g. `Asia Pacific (Mumbai) ap-south-1` or `US East (N. Virginia) us-east-1`. Write it in `MEMORY.md`. Every step below happens in this region.

## 3. Cost guardrails (do this before creating anything)

1. **Billing and Cost Management → Budgets → Create budget**
   - Template **Zero spend budget** → name `shopsphere-zero-spend` → both emails → create.
   - Create a second one: **Monthly cost budget** → `shopsphere-monthly-5usd` → amount **$5** → alerts at 50 %, 80 %, 100 % (actual) and 100 % (forecasted) → both emails.
   - In the budget's advanced/filter options, if you can choose which charge types to include, **exclude Credits** so the budget tracks the real usage that is consuming your credits.
2. **Billing → Credits**: note the starting balance in `MEMORY.md`.
3. **Earn extra credits:** Console Home → **Explore AWS** widget. Activities such as *setting up a budget*, *launching an EC2 instance* and *creating an RDS database* each add credits — and you are doing them anyway.
4. **Cost Explorer**: enable it (first load takes up to 24 h).

## 4. Local development (free) — do Phases 1–2 first

See `README.md → Quick start`. Only continue to §5 once the full journey works locally.

---

## 5. DynamoDB tables (Member B) — ~5 min

**Console:** DynamoDB → Tables → *Create table*

| Table name | Partition key | Settings |
|---|---|---|
| `ShopSphere_Carts` | `customer_id` (String) | *Customize settings* → Capacity mode **On-demand** → tag `Project=ShopSphere` |
| `ShopSphere_Sessions` | `session_id` (String) | On-demand; after creation: *Additional settings* → **Time to Live** → Turn on → attribute `expires_at` |
| `ShopSphere_UserPreferences` | `customer_id` (String) | On-demand |

**Or CloudShell** (terminal icon in the console top bar — free):
```bash
for T in "ShopSphere_Carts customer_id" "ShopSphere_Sessions session_id" "ShopSphere_UserPreferences customer_id"; do
  set -- $T
  aws dynamodb create-table --table-name "$1" \
    --attribute-definitions AttributeName=$2,AttributeType=S \
    --key-schema AttributeName=$2,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST \
    --tags Key=Project,Value=ShopSphere Key=Owner,Value=B
done
aws dynamodb wait table-exists --table-name ShopSphere_Sessions
aws dynamodb update-time-to-live --table-name ShopSphere_Sessions \
  --time-to-live-specification "Enabled=true,AttributeName=expires_at"
```

**Insert the brief's exact cart (for the demo):**
```bash
aws dynamodb put-item --table-name ShopSphere_Carts --item '{
  "customer_id": {"S": "101"},
  "items": {"L": [
    {"M": {"product_id": {"S": "P100"}, "quantity": {"N": "2"}}},
    {"M": {"product_id": {"S": "P200"}, "quantity": {"N": "1"}}}
  ]},
  "version": {"N": "1"}
}'
aws dynamodb get-item --table-name ShopSphere_Carts --key '{"customer_id":{"S":"101"}}'
```
Console check: table → **Explore table items** (toggle *JSON view*), and **PartiQL editor**:
`SELECT * FROM "ShopSphere_Carts" WHERE customer_id = '101'`

## 6. Security groups (Member B) — ~5 min

EC2 → Network & Security → **Security Groups** → *Create security group* (VPC = **default VPC**)

| Name | Inbound rules | Outbound |
|---|---|---|
| `shopsphere-app-sg` | Custom TCP **3000**, source *My IP* while developing → change to *Anywhere-IPv4* only for the recording day | default (all) |
| `shopsphere-rds-sg` | PostgreSQL **5432**, source = **security group `shopsphere-app-sg`** | default |

No port 22 anywhere — we use Session Manager.

## 7. IAM role for the EC2 instance (Member B) — ~5 min

1. IAM → Roles → *Create role* → Trusted entity **AWS service** → Use case **EC2** → Next.
2. Attach managed policy **AmazonSSMManagedInstanceCore** (for Session Manager) → name `ShopSphereEC2Role` → create.
3. Open the role → *Add permissions* → *Create inline policy* → JSON → paste (replace `REGION` and `ACCOUNT_ID`) → name `ShopSphereAppAccess`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBAppTables",
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem","dynamodb:PutItem","dynamodb:UpdateItem",
                 "dynamodb:DeleteItem","dynamodb:Query","dynamodb:DescribeTable",
                 "dynamodb:BatchGetItem","dynamodb:BatchWriteItem"],
      "Resource": "arn:aws:dynamodb:REGION:ACCOUNT_ID:table/ShopSphere_*"
    }
  ]
}
```
(The Aurora `rds-db:connect` statement is added in §12 once the cluster exists.)

## 8. RDS for PostgreSQL (Member A) — ~10 min + ~15 min wait

RDS → Databases → **Create database** → choose the **full/standard** configuration flow (not "easy create", which can pick bigger defaults).

| Setting | Value |
|---|---|
| Engine | **PostgreSQL**, latest **17.x** (avoid old majors that move to paid Extended Support) |
| Template | **Free tier** (if shown) else **Dev/Test** |
| Availability | **Single-AZ DB instance** |
| DB instance identifier | `shopsphere-rds` |
| Master username | `shopadmin` |
| Credentials management | **Self managed** password (Secrets Manager costs extra per secret) — store it in your password manager |
| Instance class | Burstable → **db.t4g.micro** |
| Storage | General Purpose SSD, **20 GiB**, **uncheck storage autoscaling** |
| Compute resource | *Don't connect to an EC2 compute resource* |
| VPC / subnet group | Default VPC / default |
| Public access | **No** |
| VPC security group | Choose existing → **`shopsphere-rds-sg`** (remove `default`) |
| Database authentication | Password authentication |
| Monitoring | Leave free/basic defaults; **Performance Insights paid retention off; Enhanced Monitoring off** |
| Additional configuration → Initial database name | **`shopsphere`** |
| Backup retention | **1 day** |
| Deletion protection | Off |
| RDS Extended Support (if shown) | Unchecked |
| Tags | `Project=ShopSphere`, `Owner=A` |

Check the **Estimated monthly costs** panel before clicking *Create*. Copy the endpoint into `MEMORY.md` when status = *Available*.

## 9. Launch the EC2 app server (Member B) — ~10 min

EC2 → Instances → **Launch instances**

| Setting | Value |
|---|---|
| Name | `shopsphere-app` (+ tag `Project=ShopSphere`) |
| AMI | **Amazon Linux 2023**, architecture **64-bit (Arm)** |
| Instance type | **t4g.micro** (pick **t4g.small** only if it's labelled free-tier/credit eligible — 2 GB RAM builds faster) |
| Key pair | **Proceed without a key pair** (Session Manager instead) |
| Network | Default VPC, any subnet, **Auto-assign public IP: Enable** |
| Security group | Select existing → **`shopsphere-app-sg`** |
| Storage | 10 GiB gp3 |
| Advanced → IAM instance profile | **`ShopSphereEC2Role`** |
| Advanced → User data | paste `scripts/ec2-userdata.sh` (below) |

`scripts/ec2-userdata.sh`
```bash
#!/bin/bash
set -eux
dnf update -y
dnf install -y git jq
# PostgreSQL client: take the newest available major (must be >= server major for pg_dump)
dnf install -y postgresql17 || dnf install -y postgresql16 || dnf install -y postgresql15
# 2 GB swap so npm/vite builds don't run out of memory on 1 GB instances
fallocate -l 2G /swapfile && chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
echo '/swapfile swap swap defaults 0 0' >> /etc/fstab
# Node.js 22 + pm2 for ec2-user
sudo -u ec2-user bash -lc 'curl -fsSL https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash \
  && source ~/.nvm/nvm.sh && nvm install 22 && npm i -g pm2'
# RDS CA bundle for TLS verification
sudo -u ec2-user bash -lc 'mkdir -p ~/certs && curl -fsSL -o ~/certs/global-bundle.pem \
  https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem'
```
Note the **public IPv4** in `MEMORY.md` (it changes after every stop/start).

## 10. Connect to EC2 and load the RDS schema (Member A) — ~20 min

1. EC2 → select instance → **Connect** → **Session Manager** tab → *Connect* (wait 2–5 min after launch for it to become available). If it never appears: check the role is attached, then reboot once.
2. In the browser terminal:
```bash
sudo su - ec2-user
node -v && psql --version           # verify user-data finished (see /var/log/cloud-init-output.log if not)
git clone https://github.com/<org>/shopsphere.git && cd shopsphere
export RDS_HOST=<shopsphere-rds endpoint>
export RDS_ADMIN="host=$RDS_HOST port=5432 dbname=shopsphere user=shopadmin sslmode=verify-full sslrootcert=$HOME/certs/global-bundle.pem"
psql "$RDS_ADMIN" -v ON_ERROR_STOP=1 -f database/schema.sql
psql "$RDS_ADMIN" -v ON_ERROR_STOP=1 -f database/seed.sql
```
3. Least-privilege application user (run in `psql "$RDS_ADMIN"`):
```sql
CREATE ROLE app_user LOGIN PASSWORD '<generate-a-strong-one>';
GRANT CONNECT ON DATABASE shopsphere TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;
```
4. Smoke test with the brief's queries:
```sql
SELECT * FROM Orders WHERE customer_id = 101;
SELECT Customers.name, Orders.order_id, Orders.total_amount
FROM Customers JOIN Orders ON Customers.customer_id = Orders.customer_id;
```

## 11. Deploy the app in RDS mode (Member B) — ~20 min

```bash
cd ~/shopsphere/backend
cp .env.example .env && chmod 600 .env
nano .env        # set AWS_REGION, DB_MODE=rds, RDS_HOST, RDS_PASSWORD (app_user), JWT_SECRET (openssl rand -hex 32), ADMIN_EMAILS
npm ci
cd ../frontend && npm ci && npm run build
cd ../backend && pm2 start src/server.js --name shopsphere && pm2 save
pm2 logs shopsphere --lines 50
curl -s localhost:3000/api/health
```
Open `http://<public-ip>:3000` → register, browse, add to cart, checkout, pay, track. Check the DynamoDB console shows your cart/session items.
Redeploys later: `scripts/deploy.sh` (`git pull && npm ci && npm run build && pm2 restart shopsphere --update-env`).

---

## 12. Aurora PostgreSQL — festival mode (Member A) — ~30 min

### 12.1 Create the cluster (express configuration)
On the Free plan, Aurora PostgreSQL serverless is available only through **express configuration**, limited to 4 ACUs and 1 GiB storage per cluster, 2 clusters and 2 instances per account. Express clusters are not placed in your VPC; they're reached through an **internet access gateway** that accepts **IAM authentication only**.

1. RDS → Databases → **Create with express configuration** → *Create*.
2. Cluster identifier `shopsphere-aurora`; **capacity range: min 0 (or the lowest allowed), max 2 ACU** → *Create database*. Available in about a minute.
3. **Add the reader:** select the cluster → **Actions → Add reader** → identifier `shopsphere-aurora-reader-1`, class *Serverless* → add. Wait until both instances are *Available*. Confirm in the cluster view that writer and reader are in **different AZs** (screenshot this for the report).
4. **Configuration** tab → copy the cluster **Resource ID** (`cluster-XXXX…`). **Connectivity & security** tab → copy the **writer (cluster) endpoint** and **reader endpoint**. Put all three in `MEMORY.md`.

### 12.2 Allow the EC2 role to connect (Member B)
Add this statement to the `ShopSphereAppAccess` inline policy:
```json
{
  "Sid": "AuroraIamAuth",
  "Effect": "Allow",
  "Action": "rds-db:connect",
  "Resource": [
    "arn:aws:rds-db:REGION:ACCOUNT_ID:dbuser:cluster-XXXX/app_user",
    "arn:aws:rds-db:REGION:ACCOUNT_ID:dbuser:cluster-XXXX/postgres"
  ]
}
```
(Remove the `/postgres` line after migration — least privilege.)

### 12.3 Create database and app user (CloudShell, from the console)
Cluster → **Connectivity & security → CloudShell → Launch → Run** (pre-filled psql command, already authenticated). Then:
```sql
CREATE DATABASE shopsphere;
\c shopsphere
CREATE ROLE app_user LOGIN;
GRANT rds_iam TO app_user;
```

### 12.4 Migrate RDS → Aurora (Member A, on EC2)
`scripts/migrate-rds-to-aurora.sh`
```bash
#!/bin/bash
set -euo pipefail
: "${AWS_REGION:?}" "${RDS_HOST:?}" "${AURORA_WRITER_HOST:?}"
RDS_ADMIN="host=$RDS_HOST port=5432 dbname=shopsphere user=shopadmin sslmode=verify-full sslrootcert=$HOME/certs/global-bundle.pem"
echo "1/3 dumping RDS (enter shopadmin password)…"
pg_dump "$RDS_ADMIN" --no-owner --no-privileges -f /tmp/shopsphere.sql
echo "2/3 restoring into Aurora writer with an IAM token…"
export PGPASSWORD="$(aws rds generate-db-auth-token --hostname "$AURORA_WRITER_HOST" \
  --port 5432 --region "$AWS_REGION" --username postgres)"
AURORA_ADMIN="host=$AURORA_WRITER_HOST port=5432 dbname=shopsphere user=postgres sslmode=require"
psql "$AURORA_ADMIN" -v ON_ERROR_STOP=1 -f /tmp/shopsphere.sql
echo "3/3 granting app_user…"
psql "$AURORA_ADMIN" -v ON_ERROR_STOP=1 <<'SQL'
GRANT CONNECT ON DATABASE shopsphere TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_user;
SQL
psql "$AURORA_ADMIN" -c "SELECT count(*) AS orders FROM orders;"
```
Run it **right before** switching modes, so orders placed on RDS are carried over.
If the console's CloudShell/psql snippet shows a different token command for your cluster, use the console's version — it is the source of truth.

### 12.5 Prove reader vs writer (great on camera)
In CloudShell, connect to the **reader endpoint** (edit the host in the pre-filled command), then:
```sql
SELECT aurora_db_instance_identifier() AS instance, pg_is_in_recovery() AS is_reader;  -- reader-1, true
INSERT INTO products(product_id, product_name, category, price, stock) VALUES ('PX', 'x', 'x', 1, 1);
-- ERROR: cannot execute INSERT in a read-only transaction
```

### 12.6 Switch the app
```bash
nano ~/shopsphere/backend/.env   # AURORA_WRITER_HOST=…, AURORA_READER_HOST=…
pm2 restart shopsphere --update-env
```
Admin page → **DB mode → Aurora**. Catalog badge should read `AURORA · reader · …reader-1`; checkout/orders `AURORA · writer · …instance-1`.

## 13. Festival-sale simulation + CloudWatch (Member B)

1. CloudWatch → Dashboards → *Create* `ShopSphere-Festival` → add **Line** widgets:
   - RDS → *Per-Database Metrics* → `ServerlessDatabaseCapacity` and `DatabaseConnections` for **both** Aurora instances.
   - RDS `CPUUtilization` for `shopsphere-rds` (for contrast).
   - DynamoDB → `ConsumedReadCapacityUnits` for `ShopSphere_Carts`.
   - Period **1 minute**, time range 30 min, auto-refresh 10 s.
2. Load test from EC2 (`scripts/load-test.sh`):
```bash
npx autocannon -c 50 -d 180 "http://localhost:3000/api/products?page=1&limit=12"
```
3. Metrics lag 1–3 min — **start the load test before you begin narrating** this segment. Expect reader capacity/connections to rise while the writer stays flat.

## 14. Failover drill (Member A)

Cluster → **Actions → Failover** → confirm. Watch the *Role* column swap writer/reader (typically completes within tens of seconds). Refresh the app: the writer badge now shows the other instance id. Note the observed time in `MEMORY.md`.

## 15. Daily "lights-off" routine (2 minutes, every evening)

| What | How |
|---|---|
| App | `pm2 stop shopsphere` (lets Aurora auto-pause) |
| EC2 | Instance state → **Stop** |
| RDS | Actions → **Stop temporarily** (auto-restarts after 7 days!) |
| Aurora | Min capacity 0 → pauses itself after inactivity; set min back to 0 if you raised it |
| Check | Billing → Credits / Cost Explorer once a day |

## 16. Recording-day prep

- Start EC2 + RDS 30 min early; Aurora **min capacity 0.5 ACU** 15 min before (no resume pause on camera).
- App security group: port 3000 from *Anywhere* (only today).
- Admin → *Reset demo data*; run migration again if you'll show Aurora with fresh data.
- Close other console tabs that show the account ID; set browser zoom to 125 %.

## 17. Teardown (immediately after the final video is exported)

Delete in this order (each step: confirm in console):
1. **EC2**: Terminate `shopsphere-app` (its root EBS volume deletes with it — verify under *Volumes*).
2. **Aurora**: delete `reader-1` instance → delete writer instance → delete cluster (uncheck *create final snapshot*, acknowledge).
3. **RDS**: delete `shopsphere-rds` → uncheck final snapshot → ✅ *delete automated backups*.
4. **Snapshots**: RDS → Snapshots (manual + system) → none left.
5. **DynamoDB**: delete the three tables.
6. **CloudWatch**: delete dashboard; Logs → delete any `/aws/rds/...` log groups.
7. **IAM**: delete role `ShopSphereEC2Role`; (keep or delete the two admin users).
8. **EC2 → Security Groups**: delete the two `shopsphere-*` groups; **Elastic IPs**: none allocated.
9. **Resource Groups & Tag Editor** → search all resource types, tag `Project=ShopSphere`, **all regions** → should be empty.
10. Next day: Billing → **Bills** (current month, by service) — nothing still accruing. Record final credits used in `MEMORY.md`.

## 18. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Session Manager "Connect" disabled | role missing / agent not registered yet | attach `ShopSphereEC2Role`, wait 5 min, reboot |
| `psql: timeout` to RDS | SG rule wrong | `shopsphere-rds-sg` inbound 5432 from **SG** `shopsphere-app-sg` (not an IP) |
| `no pg_hba.conf entry … no encryption` | TLS not used | add `sslmode=verify-full sslrootcert=…global-bundle.pem` |
| `PAM authentication failed` (Aurora) | token for wrong user/host/region, or policy lacks `rds-db:connect` for that user/resource id | regenerate token with exact endpoint; check policy ARN uses the **cluster resource id** |
| `pg_dump: server version mismatch` | client older than server | install newer `postgresqlNN` client |
| App first request slow in Aurora mode | cluster resuming from 0 ACU | raise min capacity before demo |
| Browser can't reach `:3000` | SG source "My IP" changed / app crashed | update SG; `pm2 logs` |
| `npm ci` killed | out of memory | confirm swap: `swapon --show` |
| Console option blocked on Free plan | service/feature not in Free plan | find an alternative; only upgrade to Paid plan after checking cost with the team |

---

## Appendix A — Useful CLI one-liners (CloudShell)

```bash
aws rds describe-db-clusters --db-cluster-identifier shopsphere-aurora \
  --query 'DBClusters[0].{writer:Endpoint,reader:ReaderEndpoint,resId:DbClusterResourceId,members:DBClusterMembers}'
aws rds describe-db-instances --db-instance-identifier shopsphere-rds --query 'DBInstances[0].Endpoint.Address'
aws ec2 describe-instances --filters Name=tag:Project,Values=ShopSphere \
  --query 'Reservations[].Instances[].{id:InstanceId,state:State.Name,ip:PublicIpAddress}'
aws rds stop-db-instance --db-instance-identifier shopsphere-rds
```

## Appendix B — Paid plan / older account (full Aurora configuration)

If you're on the Paid plan (or an account created before 15 July 2025), you may instead create Aurora with **full configuration** inside the default VPC: Aurora PostgreSQL → *Serverless v2* → min **0**, max **2** ACU → default VPC, **Public access No**, SG `shopsphere-rds-sg` → after creation *Add reader*. You can then use password auth (or keep IAM auth), and the RDS Query Editor/Data API become options. Everything else in this guide is unchanged. Budgets matter more here: anything beyond credits is billed.

## Appendix C — AWS Academy Learner Lab

If your college provides a Learner Lab, you usually **cannot create IAM roles** — attach the pre-made `LabInstanceProfile`/`LabRole` to EC2 instead of `ShopSphereEC2Role`, and check which RDS/Aurora options the lab allows before planning the Aurora segment. The lab session stops resources when it ends, but deletion is still your job.
