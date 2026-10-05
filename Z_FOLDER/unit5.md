Ta tiếp tục **Phần 2 → Unit 5 – Database & Monitoring Foundations (20 phút)** và dùng nguyên trạng hệ thống cuối Unit 4, không dựng lại Java/PostgreSQL.

Theo guide, Unit 5 bắt buộc phải thể hiện ba nhóm kỹ năng: PostgreSQL least-privilege + app connectivity + backup/restore; chẩn đoán connection/slow-query bằng session/lock/`EXPLAIN` và quyết định L1.5 hay escalate; cuối cùng là Grafana dashboard + alert + failure injection + notification. :chatgpt-content-reference{index="0"} Đây chính xác là Day 15–17 của syllabus. :chatgpt-content-reference{index="1"}

Trạng thái đầu vào mà ta giữ từ Unit 4 là:

```text
EC2 Ubuntu
│
├── Tomcat 10 :8080
│    └── fresher-app 1.0.0
│         ├── /health
│         ├── /info
│         ├── /db-check
│         └── /metrics
│
├── PostgreSQL 16 :5432
│    └── fresherdb
│         └── app.demo_message
│
├── Prometheus :9090
├── node_exporter :9100
└── Grafana :3000
```

Ứng dụng hiện đã kết nối DB qua JDBC + TLS với role `fresher_app`.

---

# Syllabus flow của Unit 5

Ta sẽ xây Unit 5 thành một chuỗi liên tục:

```text
PostgreSQL least privilege
        ↓
pg_hba.conf awareness
        ↓
backup
        ↓
restore verification
        ↓
session / lock diagnostics
        ↓
EXPLAIN
        ↓
L1 vs escalation
        ↓
Prometheus scrape Java /metrics
        ↓
Grafana dashboard
        ↓
Grafana Webhook contact point
        ↓
Grafana alert
        ↓
STOP PostgreSQL
        ↓
Java stays UP
DB metric becomes 0
        ↓
Grafana alert fires
        ↓
Webhook receives notification
        ↓
START PostgreSQL
        ↓
full recovery
```

Điều này cũng chuẩn bị trực tiếp cho Unit 6.

---

# A. Baseline trước khi thay đổi

Tất cả command dưới đây chạy **sau khi SSH vào EC2**.

Xác nhận đúng host:

```bash
whoami
hostname
```

Application:

```bash
systemctl is-active tomcat10
```

Database:

```bash
systemctl is-active postgresql
```

Monitoring:

```bash
systemctl is-active prometheus
systemctl is-active prometheus-node-exporter
systemctl is-active grafana-server
```

PostgreSQL readiness:

```bash
pg_isready \
  -h 127.0.0.1 \
  -p 5432
```

App health:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/health
```

Expected:

```text
status=UP
```

DB:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/db-check
```

Expected invariant:

```text
db=UP
database=fresherdb
user=fresher_app
tls=true
```

Metrics:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/metrics
```

Expected:

```text
fresher_app_up 1
fresher_app_db_up 1
fresher_app_db_tls 1
```

Đây là **BEFORE state** của Unit 5.

---

# B. Kiểm tra PostgreSQL cluster

```bash
pg_lsclusters
```

Expected dạng:

```text
Ver Cluster Port Status Owner    Data directory
16  main    5432 online postgres ...
```

Socket:

```bash
sudo ss -lntp | grep ':5432'
```

Capacity:

```bash
df -h /var/lib/postgresql
```

Database size:

```bash
sudo -u postgres psql \
  -d fresherdb \
  -c "
SELECT
    current_database(),
    pg_size_pretty(pg_database_size(current_database()))
        AS database_size;
"
```

Điểm cần giải thích:

> Capacity diagnosis không chỉ nhìn query. Database có thể gặp vấn đề do filesystem full, session exhaustion, locks hoặc application connection.

---

# C. Kiểm tra `fresher_app` đã least privilege chưa

Unit 4 đã tạo application identity:

```text
fresher_app
```

Đừng tạo một role mới rồi bỏ qua role thật mà application đang dùng.

Kiểm tra:

```bash
sudo -u postgres psql \
  -d fresherdb \
  -c "\du fresher_app"
```

Kiểm tra quyền table:

```bash
sudo -u postgres psql \
  -d fresherdb \
  -P pager=off \
  -c "
SELECT
    grantee,
    table_schema,
    table_name,
    privilege_type
FROM information_schema.role_table_grants
WHERE grantee = 'fresher_app'
ORDER BY table_schema, table_name, privilege_type;
"
```

Ta muốn thấy `SELECT` trên:

```text
app.demo_message
```

Application runtime không cần:

```text
SUPERUSER
CREATEDB
CREATEROLE
DROP TABLE
CREATE DATABASE
```

---

# D. Tạo thêm một least-privilege role để demo

Để chứng minh hiểu PostgreSQL role model, ta tạo:

```text
fresher_readonly
```

là **NOLOGIN group role**, và:

```text
fresher_report
```

là LOGIN role kế thừa quyền đọc.

Đây là mô hình:

```text
fresher_readonly
     NOLOGIN
        ↑
       GRANT
        |
fresher_report
      LOGIN
```

Tạo password tạm trong memory:

```bash
REPORT_PASS="$(openssl rand -hex 24)"
```

Không chạy:

```bash
echo "$REPORT_PASS"
```

---

# E. Tạo roles

```bash
sudo -u postgres psql \
  -v ON_ERROR_STOP=1 <<SQL

CREATE ROLE fresher_readonly
    NOLOGIN;

CREATE ROLE fresher_report
    LOGIN
    PASSWORD '${REPORT_PASS}'
    INHERIT
    NOSUPERUSER
    NOCREATEDB
    NOCREATEROLE
    NOREPLICATION;

GRANT fresher_readonly
TO fresher_report;

GRANT CONNECT
ON DATABASE fresherdb
TO fresher_readonly;

SQL
```

`NOLOGIN` nghĩa là role đó không phải account dùng để đăng nhập trực tiếp.

`fresher_report` là login identity.

`INHERIT` cho phép nó sử dụng privilege của role membership.

---

# F. Cấp đúng quyền đọc

Các object trong schema `app` nằm trong database `fresherdb`, nên:

```bash
sudo -u postgres psql \
  -d fresherdb \
  -v ON_ERROR_STOP=1 <<'SQL'

GRANT USAGE
ON SCHEMA app
TO fresher_readonly;

GRANT SELECT
ON ALL TABLES IN SCHEMA app
TO fresher_readonly;

ALTER DEFAULT PRIVILEGES
FOR ROLE postgres
IN SCHEMA app
GRANT SELECT ON TABLES
TO fresher_readonly;

SQL
```

`USAGE ON SCHEMA` cho phép role truy cập objects nằm trong namespace.

`SELECT` cho phép đọc table.

`ALTER DEFAULT PRIVILEGES` áp dụng cho **future tables được `postgres` tạo trong schema `app`**.

Nó không retroactively grant object cũ; vì vậy ta vẫn cần:

```sql
GRANT SELECT ON ALL TABLES ...
```

---

# G. Test allowed operation

```bash
PGPASSWORD="$REPORT_PASS" \
psql \
  "host=127.0.0.1 port=5432 dbname=fresherdb user=fresher_report sslmode=require" \
  -c \
  "SELECT * FROM app.demo_message;"
```

Expected success.

---

# H. Test denied operation

```bash
PGPASSWORD="$REPORT_PASS" \
psql \
  "host=127.0.0.1 port=5432 dbname=fresherdb user=fresher_report sslmode=require" \
  -c \
  "DELETE FROM app.demo_message WHERE id = 1;"
```

Expected:

```text
ERROR: permission denied for table demo_message
```

Đây **không phải incident**.

Đây là expected security behavior.

Bạn nên nói:

> "`fresher_report` chỉ cần đọc dữ liệu nên em cấp `SELECT`, không cấp `INSERT/UPDATE/DELETE`. Việc DELETE bị từ chối chứng minh least privilege đang hoạt động."

Sau test:

```bash
unset REPORT_PASS
```

---

# I. `pg_hba.conf` awareness

Tìm file thực tế, không hard-code path:

```bash
HBA_FILE="$(
  sudo -u postgres psql \
    -Atc 'SHOW hba_file;'
)"
```

```bash
echo "$HBA_FILE"
```

Ubuntu 24.04/PostgreSQL 16 thường sẽ ra dạng:

```text
/etc/postgresql/16/main/pg_hba.conf
```

nhưng command `SHOW hba_file` mới là source of truth.

---

# J. Inspect HBA

Không sửa ngay.

```bash
sudo grep \
  -nEv \
  '^[[:space:]]*(#|$)' \
  "$HBA_FILE"
```

Bạn có thể thấy những rule kiểu:

```text
local   all   postgres                  peer
local   all   all                       peer
host    all   all   127.0.0.1/32        scram-sha-256
host    all   all   ::1/128             scram-sha-256
```

Exact content có thể khác.

---

# K. Hiểu một HBA record

Ví dụ:

```text
host    all    all    127.0.0.1/32    scram-sha-256
```

Nghĩa là:

```text
host
```

TCP/IP connection.

```text
all
```

database.

```text
all
```

user.

```text
127.0.0.1/32
```

source address.

```text
scram-sha-256
```

authentication method.

Một nguyên tắc cực kỳ quan trọng:

> PostgreSQL đi qua `pg_hba.conf` **theo thứ tự** và dùng rule đầu tiên match connection; nó không thử các rule tiếp theo nếu authentication của rule match bị fail. `pg_hba_file_rules` cũng có thể được dùng để kiểm tra lỗi cấu hình HBA. :chatgpt-content-reference{index="2"}

---

# L. Dùng `pg_hba_file_rules`

Đây là cách tốt hơn việc chỉ đọc text:

```bash
sudo -u postgres psql \
  -P pager=off \
  -c "
SELECT
    line_number,
    type,
    database,
    user_name,
    address,
    auth_method,
    error
FROM pg_hba_file_rules
ORDER BY line_number;
"
```

Quan trọng:

```text
error = null
```

cho các rule hợp lệ.

Nếu `error` khác null, phải investigate trước reload.

---

# M. Vì sao ta chưa cần sửa HBA?

Application Unit 4 hiện đã:

```text
connect TCP → 127.0.0.1:5432
sslmode=require
SCRAM authentication
```

và:

```text
/db-check → tls=true
```

Cho nên không nên sửa `pg_hba.conf` chỉ để chứng minh biết edit file.

Đây là operations mindset:

> Không tạo change nếu environment hiện đã đáp ứng requirement.

Syllabus yêu cầu **pg_hba.conf awareness**, và ta đã chứng minh được cách tìm file, đọc rule, hiểu authentication flow và validate bằng `pg_hba_file_rules`. :chatgpt-content-reference{index="3"}

---

# N. Health check DB từ application

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/db-check
```

Application phải trả:

```text
HTTP/1.1 200
...
db=UP
database=fresherdb
user=fresher_app
tls=true
```

Đây là bằng chứng tốt hơn chỉ:

```bash
systemctl status postgresql
```

Vì nó chứng minh toàn flow:

```text
Java
→ JDBC
→ authentication
→ TLS
→ database
→ schema
→ SELECT
→ response
```

---

# O. Backup PostgreSQL

Tạo directory:

```bash
sudo install \
  -d \
  -o postgres \
  -g postgres \
  -m 750 \
  /opt/java-fresher/backups/postgresql
```

Tạo biến filename:

```bash
PG_BACKUP="/opt/java-fresher/backups/postgresql/fresherdb_$(date +%Y%m%d_%H%M%S).dump"
```

Backup:

```bash
sudo -u postgres pg_dump \
  --format=custom \
  --dbname=fresherdb \
  --file="$PG_BACKUP"
```

Capture exit code ngay:

```bash
RC=$?
echo "pg_dump_exit_code=$RC"
```

Expected:

```text
pg_dump_exit_code=0
```

`pg_dump` tạo logical backup nhất quán của một database và không block normal readers/writers trong lúc dump. Custom format `-Fc` phù hợp với `pg_restore` và cho phép restore chọn lọc linh hoạt. :chatgpt-content-reference{index="4"}

---

# P. Validate backup

File existence:

```bash
sudo ls -lh "$PG_BACKUP"
```

File type:

```bash
sudo file "$PG_BACKUP"
```

Hash:

```bash
sudo sha256sum "$PG_BACKUP"
```

Quan trọng hơn: archive có đọc được không?

```bash
sudo -u postgres pg_restore \
  --list \
  "$PG_BACKUP" \
  | head -40
```

Nếu `pg_restore --list` đọc được table of contents:

```text
backup archive structurally readable
```

Nhưng vẫn chưa đủ.

Ta phải restore thật.

---

# Q. Restore vào database riêng

**Không restore đè `fresherdb`.**

Trước hết:

```bash
sudo -u postgres dropdb \
  --if-exists \
  fresherdb_restore
```

Tạo database clean:

```bash
sudo -u postgres createdb \
  -T template0 \
  fresherdb_restore
```

Tại sao `template0`?

Đây là clean database template phù hợp khi restore logical dump.

PostgreSQL docs cũng lưu ý database destination phải tồn tại trước khi restore nếu dump không tự create database. :chatgpt-content-reference{index="5"}

---

# R. Restore

```bash
sudo -u postgres pg_restore \
  --exit-on-error \
  --no-owner \
  --dbname=fresherdb_restore \
  "$PG_BACKUP"
```

Capture:

```bash
RC=$?
echo "pg_restore_exit_code=$RC"
```

Expected:

```text
pg_restore_exit_code=0
```

`pg_restore` là utility dành cho archive format từ `pg_dump`, bao gồm custom format chúng ta đang dùng. :chatgpt-content-reference{index="6"}

---

# S. Verify recovered data

```bash
sudo -u postgres psql \
  -d fresherdb_restore \
  -c "
SELECT
    id,
    message
FROM app.demo_message;
"
```

Expected:

```text
1 | Java Fresher integrated lab database is healthy
```

Kiểm tra schema:

```bash
sudo -u postgres psql \
  -d fresherdb_restore \
  -c '\dt app.*'
```

Ta đã chứng minh:

```text
backup exists
+
archive readable
+
database restored
+
schema exists
+
actual application data recovered
```

Đây mới là **backup verification**.

---

# T. Cleanup restore database

Sau khi validate:

```bash
sudo -u postgres dropdb \
  fresherdb_restore
```

Không cần giữ một duplicate DB gây lãng phí/nhầm lẫn cho Unit 6.

Ta giữ:

```text
$PG_BACKUP
```

làm recovery artifact.

---

# U. Session / lock / slow-query diagnosis

Đây là phần quan trọng nhất của Day 16.

PostgreSQL cung cấp `pg_stat_activity` để nhìn sessions/current activity và `pg_locks` để inspect locks; `pg_blocking_pids(pid)` giúp xác định session nào đang block một backend. :chatgpt-content-reference{index="7"}

Ta tạo **controlled lock failure**.

Không làm việc này trên production.

---

# V. BEFORE lock injection

App:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/db-check
```

DB:

```bash
pg_isready
```

Check current sessions:

```bash
sudo -u postgres psql \
  -d fresherdb \
  -P pager=off \
  -c "
SELECT
    pid,
    usename,
    application_name,
    state,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE datname = 'fresherdb'
ORDER BY pid;
"
```

---

# W. Tạo blocker có kiểm soát

Chạy:

```bash
sudo -u postgres psql \
  -d fresherdb \
  -v ON_ERROR_STOP=1 \
  -c "
BEGIN;
LOCK TABLE app.demo_message
    IN ACCESS EXCLUSIVE MODE;
SELECT pg_sleep(120);
COMMIT;
" \
  >/tmp/fresher-lock-holder.log \
  2>&1 &
```

Lưu shell job PID:

```bash
LOCK_CLIENT_PID=$!
```

Không cần hiểu `pg_sleep` như root cause.

Ở đây nó chỉ giữ connection/transaction sống đủ lâu để ta inspect lock.

---

# X. Tạo blocked query

```bash
sudo -u postgres psql \
  -d fresherdb \
  -v ON_ERROR_STOP=1 \
  -c "
SELECT *
FROM app.demo_message
WHERE id = 1;
" \
  >/tmp/fresher-blocked-query.log \
  2>&1 &
```

```bash
BLOCKED_CLIENT_PID=$!
```

Bây giờ query thứ hai đang chờ lock.

---

# Y. Inspect sessions

```bash
sudo -u postgres psql \
  -d fresherdb \
  -P pager=off \
  -c "
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    age(clock_timestamp(), query_start)
        AS running_for,
    left(query, 90)
        AS query
FROM pg_stat_activity
WHERE datname = 'fresherdb'
  AND pid <> pg_backend_pid()
ORDER BY query_start;
"
```

Bạn nên thấy một session có:

```text
wait_event_type = Lock
```

Đây là bằng chứng rất quan trọng.

Query "slow" không nhất thiết do planner/index.

Nó có thể đơn giản là **đang bị block**.

---

# Z. Tìm blocking PID

```bash
sudo -u postgres psql \
  -d fresherdb \
  -P pager=off \
  -c "
SELECT
    pid AS blocked_pid,
    pg_blocking_pids(pid)
        AS blocking_pids,
    wait_event_type,
    wait_event,
    left(query, 80)
        AS blocked_query
FROM pg_stat_activity
WHERE datname = 'fresherdb'
  AND cardinality(pg_blocking_pids(pid)) > 0;
"
```

Expected conceptually:

```text
blocked_pid | blocking_pids | wait_event_type
------------+---------------+----------------
xxxxx       | {yyyyy}       | Lock
```

`yyyyy` chính là blocking backend.

---

# AA. Inspect `pg_locks`

```bash
sudo -u postgres psql \
  -d fresherdb \
  -P pager=off \
  -c "
SELECT
    a.pid,
    a.usename,
    l.locktype,
    l.mode,
    l.granted,
    a.wait_event_type,
    a.wait_event
FROM pg_locks l
JOIN pg_stat_activity a
  ON a.pid = l.pid
WHERE l.relation =
      'app.demo_message'::regclass
ORDER BY a.pid, l.granted;
"
```

Bạn có thể thấy:

```text
AccessExclusiveLock    granted = true
AccessShareLock        granted = false
```

Ý nghĩa:

```text
holder
owns incompatible lock

blocked query
waits for lock
```

---

# AB. Xác định blocker chính xác

```bash
BLOCKER_PID="$(
  sudo -u postgres psql \
    -d fresherdb \
    -Atc "
SELECT blocking_pid
FROM (
    SELECT unnest(
        pg_blocking_pids(pid)
    ) AS blocking_pid
    FROM pg_stat_activity
    WHERE datname = 'fresherdb'
      AND cardinality(
          pg_blocking_pids(pid)
      ) > 0
) x
LIMIT 1;
"
)"
```

Verify:

```bash
echo "$BLOCKER_PID"
```

Không terminate nếu variable rỗng:

```bash
test -n "$BLOCKER_PID" \
  && echo "blocking backend identified"
```

---

# AC. L1.5 boundary

Đây là điểm bạn phải nói thật rõ.

Trong **lab**, blocker do chính chúng ta tạo, nên ta biết:

```text
owner
purpose
impact
```

Trong production, nếu thấy một blocking session của business application:

> L1.5 được phép collect evidence, xác định blocked/blocking PID, user, query, duration, wait event và recent changes. **Không nên tự terminate một session không rõ business transaction** nếu runbook/change approval không cho phép.

Terminate backend có thể rollback một transaction đang thực hiện.

Đây thường là điểm cần:

```text
DBA / L2
application owner
change approval
```

tùy runbook.

---

# AD. Remediation trong lab

Vì blocker này **chính chúng ta inject**, có thể terminate:

```bash
sudo -u postgres psql \
  -d fresherdb \
  -c "
SELECT pg_terminate_backend(
    $BLOCKER_PID
);
"
```

Expected:

```text
t
```

**Cảnh báo:** `pg_terminate_backend()` là action có impact. Không dùng ngẫu nhiên trên production.

---

# AE. Wait các lab client kết thúc

```bash
wait "$LOCK_CLIENT_PID" || true
```

```bash
wait "$BLOCKED_CLIENT_PID" || true
```

Kiểm tra query bị block đã hoàn thành:

```bash
cat /tmp/fresher-blocked-query.log
```

Expected trả row `demo_message`.

---

# AF. Confirm không còn blocker

```bash
sudo -u postgres psql \
  -d fresherdb \
  -P pager=off \
  -c "
SELECT
    pid,
    pg_blocking_pids(pid)
FROM pg_stat_activity
WHERE datname = 'fresherdb'
  AND cardinality(pg_blocking_pids(pid)) > 0;
"
```

Expected:

```text
0 rows
```

---

# AG. `EXPLAIN`

Bây giờ mới inspect query plan:

```bash
sudo -u postgres psql \
  -d fresherdb \
  -c "
EXPLAIN
SELECT *
FROM app.demo_message
WHERE id = 1;
"
```

`EXPLAIN` chỉ cho planner estimate; nó không execute SELECT result theo nghĩa `ANALYZE`.

PostgreSQL dùng `EXPLAIN` để show execution plan như sequential scan/index scan/join strategy và planner cost. :chatgpt-content-reference{index="8"}

---

# AH. `EXPLAIN ANALYZE`

Với SELECT nhỏ và an toàn của lab:

```bash
sudo -u postgres psql \
  -d fresherdb \
  -c "
EXPLAIN (
    ANALYZE,
    BUFFERS
)
SELECT *
FROM app.demo_message
WHERE id = 1;
"
```

Bạn muốn quan sát:

```text
actual time
rows
loops
planning time
execution time
buffers
```

Điểm quan trọng:

> `EXPLAIN ANALYZE` **thực sự chạy statement**.

Do đó không được chạy vô thức:

```sql
EXPLAIN ANALYZE DELETE ...
```

hoặc:

```sql
EXPLAIN ANALYZE UPDATE ...
```

trên production.

---

# AI. Kết luận slow-query scenario

Bạn nên trình bày:

> "Ban đầu query phản hồi chậm. Em không kết luận thiếu index ngay. `pg_stat_activity` cho thấy session đang chờ `Lock`, `pg_blocking_pids()` xác định blocking backend và `pg_locks` chứng minh lock conflict. Sau khi controlled blocker được release, query hoàn tất. `EXPLAIN ANALYZE` của query bản thân có execution time nhỏ, nên root cause ở đây là lock contention, không phải query plan."

Đây là troubleshooting reasoning rất tốt.

---

# AJ. L1.5 vs escalation matrix

| Tình huống                                    | L1.5                                             |
| --------------------------------------------- | ------------------------------------------------ |
| `pg_isready` fail                             | Thu thập service/socket/log evidence             |
| Application DB connection fail                | Kiểm tra app config, port, TLS, PostgreSQL state |
| Xem `pg_stat_activity`                        | Có                                               |
| Xem `pg_locks`                                | Có                                               |
| Xác định blocking PID                         | Có                                               |
| `EXPLAIN` SELECT                              | Có, nếu runbook cho phép                         |
| Restart PostgreSQL sau controlled lab failure | Có trong lab/runbook                             |
| Terminate unknown production backend          | **Escalate / approval**                          |
| Create/drop index production                  | **DBA/change approval**                          |
| Rewrite slow SQL                              | **App/DBA/L2**                                   |
| Restore production DB                         | **Escalate/change control**                      |
| Data corruption                               | **Escalate**                                     |
| Repeated PostgreSQL crash                     | **Escalate sau evidence collection**             |

Đây chính là requirement của guide: không chỉ tìm lỗi mà phải biết **phạm vi xử lý**. :chatgpt-content-reference{index="9"}

---

# AK. Đưa Java application vào Prometheus

Unit 4 đã có:

```text
http://127.0.0.1:8080/fresher-app/metrics
```

Kiểm tra:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/metrics
```

Ta muốn:

```text
fresher_app_up 1
fresher_app_db_up 1
fresher_app_db_tls 1
```

---

# AL. Backup Prometheus config

```bash
sudo cp -a \
  /etc/prometheus/prometheus.yml \
  "/etc/prometheus/prometheus.yml.bak.$(date +%Y%m%d_%H%M%S)"
```

Inspect:

```bash
sudo cat \
  /etc/prometheus/prometheus.yml
```

Không có secret trong file này.

---

# AM. Cấu hình scrape app

Baseline user data của chúng ta đang có jobs `prometheus` và `node`.

Ta chuyển thành:

```bash
sudo tee \
  /etc/prometheus/prometheus.yml \
  >/dev/null <<'EOF'

global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:

  - job_name: prometheus
    static_configs:
      - targets:
          - '127.0.0.1:9090'

  - job_name: node
    static_configs:
      - targets:
          - '127.0.0.1:9100'

  - job_name: fresher-app
    metrics_path: /fresher-app/metrics
    static_configs:
      - targets:
          - '127.0.0.1:8080'

EOF
```

Prometheus configuration chính thức sử dụng `scrape_configs` để định nghĩa jobs/targets và hỗ trợ reload configuration sau khi validation. :chatgpt-content-reference{index="10"}

---

# AN. Validate Prometheus trước restart

```bash
sudo promtool check config \
  /etc/prometheus/prometheus.yml
```

Expected:

```text
SUCCESS
```

Nếu fail:

**không restart**.

Fix config trước.

Flow đúng:

```text
backup
→ edit
→ promtool validate
→ restart
→ runtime validate
```

---

# AO. Apply

```bash
sudo systemctl restart prometheus
```

Capture:

```bash
RC=$?
echo "prometheus_restart_exit=$RC"
```

Expected:

```text
0
```

Check:

```bash
systemctl is-active prometheus
```

---

# AP. Verify targets

```bash
curl -s \
  http://127.0.0.1:9090/api/v1/targets \
  | jq '
      .data.activeTargets[]
      |
      {
        job: .labels.job,
        health: .health,
        scrapeUrl: .scrapeUrl
      }
    '
```

Ta muốn có:

```text
prometheus  up
node        up
fresher-app up
```

---

# AQ. Query application metric trong Prometheus

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=fresher_app_up' \
  | jq
```

Tương tự:

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=fresher_app_db_up' \
  | jq
```

Ta muốn value:

```text
1
```

TLS:

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=fresher_app_db_tls' \
  | jq
```

Expected:

```text
1
```

---

# AR. Vì sao không mở Security Group cho `/metrics`?

Prometheus chạy trên **chính EC2**.

Flow:

```text
Prometheus
→ 127.0.0.1:8080
→ Java /metrics
```

Không cần Internet.

Do đó không tạo thêm inbound rule.

Đây là least exposure.

---

# AS. Chuẩn bị notification receiver cho Grafana

Guide yêu cầu:

> giả lập lỗi để kiểm tra **notification evidence**. :chatgpt-content-reference{index="11"}

Để demo ổn định và không phụ thuộc Gmail/SMTP/Teams webhook ngoài, ta tạo **local webhook receiver**.

Đây là **Supplementary / Beyond explicit syllabus** nhưng rất phù hợp để chứng minh notification thật.

Grafana hiện hỗ trợ Webhook như một contact-point integration chính thức. :chatgpt-content-reference{index="12"}

---

# AT. Tạo webhook receiver Python

Unit 2 đã tạo user:

```text
demoops
```

và group:

```text
appops
```

Verify:

```bash
id demoops
```

Tạo script:

```bash
sudo tee \
  /opt/java-fresher/scripts/grafana_webhook_receiver.py \
  >/dev/null <<'PYTHON'

#!/usr/bin/env python3

from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)

class Handler(BaseHTTPRequestHandler):

    def do_POST(self):
        length = int(
            self.headers.get("Content-Length", "0")
        )

        body = self.rfile.read(length).decode(
            "utf-8",
            errors="replace"
        )

        logging.info(
            "Grafana notification path=%s body=%s",
            self.path,
            body
        )

        self.send_response(204)
        self.end_headers()

    def log_message(self, format, *args):
        return


server = ThreadingHTTPServer(
    ("127.0.0.1", 9088),
    Handler
)

logging.info(
    "Grafana webhook receiver listening on 127.0.0.1:9088"
)

server.serve_forever()

PYTHON
```

---

# AU. Permission

```bash
sudo chown \
  root:root \
  /opt/java-fresher/scripts/grafana_webhook_receiver.py
```

```bash
sudo chmod 755 \
  /opt/java-fresher/scripts/grafana_webhook_receiver.py
```

Syntax:

```bash
python3 -m py_compile \
  /opt/java-fresher/scripts/grafana_webhook_receiver.py
```

No output = syntax OK.

---

# AV. Tạo systemd service

```bash
sudo tee \
  /etc/systemd/system/grafana-webhook-demo.service \
  >/dev/null <<'EOF'

[Unit]
Description=Grafana Local Webhook Receiver for Java Fresher Lab
After=network.target

[Service]
Type=simple

User=demoops
Group=appops

ExecStart=/usr/bin/python3 /opt/java-fresher/scripts/grafana_webhook_receiver.py

Restart=on-failure
RestartSec=2

NoNewPrivileges=true
PrivateTmp=true

[Install]
WantedBy=multi-user.target

EOF
```

---

# AW. Validate + start webhook

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/grafana-webhook-demo.service
```

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable --now \
  grafana-webhook-demo.service
```

Verify:

```bash
systemctl is-active \
  grafana-webhook-demo.service
```

Expected:

```text
active
```

Socket:

```bash
sudo ss -lntp | grep ':9088'
```

Expected listener:

```text
127.0.0.1:9088
```

Không phải:

```text
0.0.0.0:9088
```

Không cần expose Internet.

---

# AX. Test webhook thủ công

```bash
curl -i \
  -X POST \
  -H 'Content-Type: application/json' \
  -d '{"source":"manual-test","status":"ok"}' \
  http://127.0.0.1:9088/alert
```

Expected:

```text
HTTP/1.0 204 No Content
```

Log:

```bash
sudo journalctl \
  -u grafana-webhook-demo.service \
  -n 20 \
  --no-pager
```

Bạn phải thấy JSON test.

---

# AY. Grafana datasource

Mở Windows browser:

```text
http://EC2_PUBLIC_IP:3000
```

Login bằng Grafana account bạn đã cấu hình.

Prometheus datasource đã được user data provision từ đầu:

```text
Prometheus
URL:
http://127.0.0.1:9090
```

Có thể kiểm tra trong:

```text
Connections
→ Data sources
→ Prometheus
```

Không tạo datasource duplicate.

---

# AZ. Tạo Grafana Dashboard

Grafana current docs dùng flow:

```text
Dashboards
→ New
→ New Dashboard
→ Add visualization
```

mỗi panel query datasource và biến result thành visualization. :chatgpt-content-reference{index="13"}

Tên dashboard:

```text
Java Fresher Integrated Health
```

---

# BA. Panel 1 — Java Application

Datasource:

```text
Prometheus
```

PromQL:

```promql
fresher_app_up
```

Visualization:

```text
Stat
```

Title:

```text
Java Application
```

Value mapping nên set:

```text
1 → UP
0 → DOWN
```

---

# BB. Panel 2 — PostgreSQL Connectivity

PromQL:

```promql
fresher_app_db_up
```

Title:

```text
PostgreSQL Connectivity
```

Visualization:

```text
Stat
```

Value:

```text
1 = UP
0 = DOWN
```

Panel này sẽ được dùng cho failure injection.

---

# BC. Panel 3 — PostgreSQL TLS

PromQL:

```promql
fresher_app_db_tls
```

Title:

```text
PostgreSQL TLS
```

Mapping:

```text
1 = ENABLED
0 = NOT ACTIVE
```

---

# BD. Panel 4 — EC2 CPU

Dùng node_exporter metrics:

```promql
100 -
(
  avg by (instance)
  (
    rate(
      node_cpu_seconds_total{
        job="node",
        mode="idle"
      }[5m]
    )
  )
  * 100
)
```

Visualization:

```text
Time series
```

Unit:

```text
Percent (0-100)
```

Title:

```text
Host CPU Usage
```

Như vậy dashboard có cả:

```text
host signal
application signal
database dependency signal
security/TLS signal
```

---

# BE. Save dashboard

Tên:

```text
Java Fresher Integrated Health
```

Grafana official docs hiện vẫn dùng dashboard + panel model này. :chatgpt-content-reference{index="14"}

---

# BF. Tạo Grafana Contact Point

Đi:

```text
Alerts & IRM
→ Alerting
→ Notification configuration
→ Contact points
→ New contact point
```

Tên:

```text
java-fresher-local-webhook
```

Integration:

```text
Webhook
```

URL:

```text
http://127.0.0.1:9088/alert
```

Không disable resolved notification.

Save.

Grafana hiện cho phép gắn contact point trực tiếp với alert rule hoặc thông qua notification policy. :chatgpt-content-reference{index="15"}

---

# BG. Test Contact Point

Trong contact point:

```text
Test
→ Send test notification
```

Grafana hiện hỗ trợ Test cho contact points dùng Grafana Alertmanager. :chatgpt-content-reference{index="16"}

Trên EC2:

```bash
sudo journalctl \
  -u grafana-webhook-demo.service \
  -n 30 \
  --no-pager
```

Bạn phải thấy notification JSON của Grafana.

Đây đã là **notification evidence**.

---

# BH. Tạo alert rule

Grafana:

```text
Alerts & IRM
→ Alerting
→ Alert rules
→ New alert rule
```

Name:

```text
FresherAppDatabaseDown
```

Grafana-managed alert rules hiện có thể query Prometheus datasource trực tiếp rồi áp dụng expressions/thresholds và route notification tới contact point. :chatgpt-content-reference{index="17"}

---

# BI. Query A

Datasource:

```text
Prometheus
```

PromQL:

```promql
fresher_app_db_up
```

Đặt:

```text
A
```

---

# BJ. Reduce expression

Add expression:

```text
Reduce
```

Input:

```text
A
```

Function:

```text
Last
```

Expression:

```text
B
```

Mục đích:

```text
time series
→ current scalar value
```

---

# BK. Threshold

Thêm:

```text
Threshold
```

Input:

```text
B
```

Condition:

```text
IS BELOW 1
```

Expression:

```text
C
```

Set:

```text
C = Alert condition
```

Tức:

```text
DB metric 1 → healthy
DB metric 0 → alert
```

---

# BL. Evaluation

Tạo evaluation group ví dụ:

```text
JavaFresherLab
```

Evaluation interval:

```text
10s
```

Pending period:

```text
0s / None
```

nếu version UI cho phép.

Mục tiêu lab là alert phản ánh ngay controlled failure.

Production thường cần pending period để tránh transient false positives.

---

# BM. Alert metadata

Folder:

```text
JavaFresherLab
```

Labels:

```text
severity=warning
service=fresher-app
component=postgresql
environment=demo
```

Summary:

```text
Java Fresher application cannot reach PostgreSQL
```

---

# BN. Notification

Chọn:

```text
Select contact point
```

Contact point:

```text
java-fresher-local-webhook
```

Save rule.

Grafana docs hiện cho phép trực tiếp chọn contact point ngay trong Grafana-managed alert rule. :chatgpt-content-reference{index="18"}

---

# BO. BEFORE failure injection

Đây là trạng thái phải quay trước khi phá DB.

Tomcat:

```bash
systemctl is-active tomcat10
```

PostgreSQL:

```bash
systemctl is-active postgresql
```

App:

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/health
```

Expected:

```text
HTTP 200
status=UP
```

DB:

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/db-check
```

Expected:

```text
HTTP 200
db=UP
tls=true
```

Metrics:

```bash
curl -s \
  http://127.0.0.1:8080/fresher-app/metrics
```

Expected:

```text
fresher_app_up 1
fresher_app_db_up 1
fresher_app_db_tls 1
```

Grafana:

```text
Java Application = UP
PostgreSQL Connectivity = UP
```

Alert:

```text
Normal
```

---

# BP. Inject PostgreSQL failure

**LAB ONLY.**

```bash
sudo systemctl stop postgresql
```

Capture:

```bash
RC=$?
echo "stop_postgresql_exit=$RC"
```

Expected:

```text
0
```

---

# BQ. Không sửa ngay — troubleshoot

Check app first:

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/health
```

Expected:

```text
HTTP 200
status=UP
```

Điều này nói:

```text
Tomcat/Java application layer
still alive
```

---

# BR. DB endpoint

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/db-check
```

Expected:

```text
HTTP 503
db=DOWN
```

Bây giờ ta đã khoanh vùng:

```text
app alive
dependency failed
```

---

# BS. Transport check

```bash
sudo ss -lntp | grep ':5432'
```

Expected:

```text
no output
```

Không có listener 5432.

Port 8080:

```bash
sudo ss -lntp | grep ':8080'
```

vẫn tồn tại.

---

# BT. Service check

```bash
systemctl status postgresql \
  --no-pager
```

Expected:

```text
inactive
```

`pg_isready`:

```bash
pg_isready \
  -h 127.0.0.1 \
  -p 5432
```

Expected failure.

---

# BU. Log evidence

```bash
sudo journalctl \
  -u postgresql \
  -n 50 \
  --no-pager
```

Do ta chủ động stop nên log phải phù hợp với controlled shutdown.

Không cần inspect Security Group.

Connection Java → PostgreSQL xảy ra:

```text
127.0.0.1
```

trên cùng EC2.

AWS SG không phải root cause.

---

# BV. Metric sau failure

```bash
curl -s \
  http://127.0.0.1:8080/fresher-app/metrics
```

Expected:

```text
fresher_app_up 1
fresher_app_db_up 0
fresher_app_db_tls 0
```

Đây là thiết kế monitoring rất tốt:

```text
Application availability = 1
Database dependency      = 0
```

---

# BW. Prometheus vẫn scrape được

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=fresher_app_db_up' \
  | jq
```

Sau khi Prometheus nhận sample mới:

```text
value = 0
```

Target vẫn:

```text
UP
```

vì `/metrics` vẫn trả HTTP 200.

Điều này rất quan trọng.

`target UP` không có nghĩa:

```text
all application dependencies healthy
```

Nó chỉ có nghĩa Prometheus scrape target thành công.

---

# BX. Grafana

Dashboard sẽ thể hiện:

```text
Java Application:
UP

PostgreSQL Connectivity:
DOWN

PostgreSQL TLS:
NOT ACTIVE
```

Alert:

```text
FresherAppDatabaseDown
→ Firing
```

---

# BY. Verify notification

Trên EC2:

```bash
sudo journalctl \
  -u grafana-webhook-demo.service \
  --since '-5 minutes' \
  --no-pager
```

Bạn muốn thấy Grafana notification JSON chứa những thông tin kiểu:

```text
FresherAppDatabaseDown
firing
severity=warning
component=postgresql
```

Không cần thuộc chính xác JSON structure.

Invariant:

```text
Grafana alert fired
→ Webhook POST sent
→ receiver got notification
```

Đó là requirement notification evidence.

---

# BZ. OSI Mindset cho failure này

Bạn nên diễn giải:

> "Em chọn top-down vì symptom xuất phát từ application DB dependency. `/health` trả 200 nên Java/Tomcat vẫn hoạt động. `/db-check` trả 503, do đó em đi xuống dependency. `ss` cho thấy 8080 vẫn listen nhưng 5432 không còn listener. `systemctl status postgresql` xác nhận PostgreSQL inactive. Recent change cũng là controlled stop PostgreSQL. Vì vậy root cause là database service down, không phải Tomcat, WAR hay AWS Security Group."

Đừng nói:

> "DB error nên em restart PostgreSQL."

Bạn phải chứng minh root cause trước.

---

# CA. Recovery

Vì đây là controlled failure:

```bash
sudo systemctl start postgresql
```

Capture:

```bash
RC=$?
echo "postgresql_start_exit=$RC"
```

Expected:

```text
0
```

---

# CB. Validate DB recovery

```bash
systemctl is-active postgresql
```

Expected:

```text
active
```

Socket:

```bash
sudo ss -lntp | grep ':5432'
```

Readiness:

```bash
pg_isready \
  -h 127.0.0.1 \
  -p 5432
```

Expected:

```text
accepting connections
```

---

# CC. Validate application end-to-end

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/health
```

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/db-check
```

Expected:

```text
db=UP
tls=true
```

Metrics:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/metrics
```

Expected:

```text
fresher_app_up 1
fresher_app_db_up 1
fresher_app_db_tls 1
```

---

# CD. Alert recovery

Prometheus:

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=fresher_app_db_up' \
  | jq
```

Metric trở lại:

```text
1
```

Grafana alert:

```text
Firing
→ Normal / Resolved
```

Vì ta không disable resolved notifications, webhook receiver cũng có thể nhận resolved notification.

Inspect:

```bash
sudo journalctl \
  -u grafana-webhook-demo.service \
  --since '-5 minutes' \
  --no-pager
```

Grafana contact points mặc định có thể gửi resolved messages trừ khi tùy chọn đó bị disable. :chatgpt-content-reference{index="19"}

---

# CE. Full integrated validation cuối Unit 5

Chạy:

```bash
echo '=== SERVICES ==='

for svc in \
  tomcat10 \
  postgresql \
  prometheus \
  prometheus-node-exporter \
  grafana-server \
  grafana-webhook-demo.service
do
    printf '%-35s ' "$svc"
    systemctl is-active "$svc"
done
```

Expected tất cả:

```text
active
```

Endpoints:

```bash
echo '=== APPLICATION ==='

curl -fsS \
  http://127.0.0.1:8080/fresher-app/health

curl -fsS \
  http://127.0.0.1:8080/fresher-app/db-check

curl -fsS \
  http://127.0.0.1:8080/fresher-app/metrics
```

DB:

```bash
echo '=== DATABASE ==='

pg_isready
```

Failed units:

```bash
echo '=== FAILED UNITS ==='

systemctl --failed --no-pager
```

Disk:

```bash
echo '=== CAPACITY ==='

df -h /
df -h /var/lib/postgresql
```

---

# CF. History cuối Unit 5

Theo convention từ Unit 1:

```bash
history -a
```

Master:

```bash
history > \
  ~/java-fresher-demo/history/history.log
```

Snapshot:

```bash
history > \
  ~/java-fresher-demo/history/"$(date +%F)_unit05_db_monitoring.log"
```

Verify:

```bash
tail -30 \
  ~/java-fresher-demo/history/history.log
```

Các command quan trọng như:

```text
pg_dump
pg_restore
pg_stat_activity
pg_blocking_pids
pg_locks
EXPLAIN
promtool
systemctl stop postgresql
systemctl start postgresql
journalctl
```

phải giải thích được trong Self Q&A.

---

# CG. Kịch bản Unit 5 trong đúng 20 phút

Các bước trên là **preparation/rehearsal đầy đủ**. Trong video chính thức không nên tạo tất cả từ đầu.

Flow 20 phút nên là:

| Thời gian | Demo                                                                                               |
| --------: | -------------------------------------------------------------------------------------------------- |
|      0–3' | PostgreSQL role/least privilege + `pg_hba_file_rules`                                              |
|      3–6' | `pg_dump` + show existing restore verification                                                     |
|     6–11' | Controlled lock → `pg_stat_activity` → `pg_locks` → `pg_blocking_pids` → `EXPLAIN` → L1/escalation |
|    11–14' | Prometheus app target + Grafana dashboard                                                          |
|    14–18' | Stop PostgreSQL → `/health UP`, `/db-check DOWN`, metric `0`, Grafana alert + webhook notification |
|    18–20' | Start PostgreSQL → validate full recovery + alert resolved                                         |

Trong video, backup/restore thật có thể đã được rehearsal trước, nhưng ít nhất bạn nên **chạy `pg_dump` trực tiếp và chứng minh một restored database đã từng được verify**, vì guide yêu cầu thực hành trực tiếp chứ không chỉ nói lý thuyết.

---

# CH. Cách nói ngắn gọn cho phần L1 vs escalation

Một đoạn rất nên tập:

> "L1.5 của em tập trung vào evidence và safe remediation: kiểm tra `pg_isready`, systemd, socket, logs, sessions, locks, blocking PID và execution plan. Em không tự terminate một business transaction không rõ owner, tạo index production, sửa query, thay tuning parameter hay restore production database nếu chưa có runbook/approval. Những trường hợp đó em giữ evidence và escalate DBA/L2 hoặc application owner."

Đây chính là điều guide muốn đánh giá chứ không phải chỉ xem bạn biết câu lệnh.

---

# CI. Trạng thái cần giữ cho Unit 6

**Không cleanup** các thành phần sau:

```text
PostgreSQL
├── fresherdb
├── fresher_app
├── fresher_readonly
├── fresher_report
└── app.demo_message

Backup
└── fresherdb_<timestamp>.dump

Java
└── fresher-app 1.0.0

Prometheus
├── prometheus job
├── node job
└── fresher-app job

Grafana
├── Java Fresher Integrated Health dashboard
├── java-fresher-local-webhook contact point
└── FresherAppDatabaseDown alert

systemd
└── grafana-webhook-demo.service
```

Unit 6 sẽ **dùng nguyên trạng này**, không dựng lại gì. Ta sẽ lấy `fresher-app 1.0.0` làm **current production-like baseline**, tạo release `1.1.0`, xây change ticket + dependency check + pre-deployment evidence + deployment + post-validation + một failure/rollback scenario + handover. Nhờ vậy Unit 4 → Unit 5 → Unit 6 trở thành đúng một hệ thống tích hợp Linux + AWS + Java + PostgreSQL + Monitoring + Operations, thay vì ba bài lab rời rạc.
