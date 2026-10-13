# Module 2 — PostgreSQL Administration, Diagnostics & Monitoring

## Day 15–17: PostgreSQL → troubleshooting/performance → Prometheus/node_exporter/Grafana/Alerting

## 1. Syllabus Alignment

Module 2 bao phủ chính xác ba ngày:

- **Day 15:** PostgreSQL installation/container, users/roles, database/schema, connectivity, `pg_hba.conf` awareness, health checks, backup/restore.
- **Day 16:** SQL/log diagnostics, sessions/locks, capacity checks, `EXPLAIN` basics, L1 escalation boundaries.
- **Day 17:** Prometheus/node_exporter, Grafana dashboard, basic alert rules, service/host/application signals, alert validation. 00_MASTER_SYLLABUS_Java_Fresher…

Ta sẽ không làm lab, nhưng vẫn đi sâu vào SQL, CLI, configuration, PromQL và troubleshooting.

Về version: các ví dụ bên dưới cố ý viết **version-neutral**. Tài liệu PostgreSQL production hiện hành mà tôi đối chiếu là PostgreSQL 18; PostgreSQL 19 vẫn đang ở Beta 4 ngày 24/09/2026, vì vậy không nên lấy documentation của development version 19 rồi mặc định đó là behavior production hiện tại. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

---

# PHẦN I — DAY 15

# PostgreSQL Foundations & Administration

## 2. Mental model tổng thể

Từ Module 1:

```text
Java application
      |
      | JDBC
      v
PostgreSQL server
      |
      +-- roles / authentication
      +-- databases
      +-- schemas
      +-- tables / indexes / sequences
      +-- transactions / locks
      +-- logs
      +-- physical data files
```

Nhìn từ phía Linux:

```text
Linux
 │
 ├── PostgreSQL processes
 ├── TCP :5432 / Unix socket
 ├── PostgreSQL data directory
 │     ├── database files
 │     ├── WAL
 │     └── configuration
 └── systemd / container runtime
```

Administrator phải phân biệt được ít nhất 5 layer:

```text
Network
   ↓
PostgreSQL listener
   ↓
pg_hba.conf authentication
   ↓
Database role authentication
   ↓
SQL authorization/privileges
```

Đây là nền tảng để troubleshoot đúng.

---

# 3. PostgreSQL là gì?

PostgreSQL là một **relational database management system — RDBMS**.

Application gửi:

```sql
SELECT id, status
FROM orders
WHERE id = 1001;
```

PostgreSQL phải:

```text
receive SQL
   ↓
parse
   ↓
rewrite
   ↓
plan
   ↓
execute
   ↓
access buffer/disk
   ↓
return rows
```

Application không đọc trực tiếp database files.

Nó kết nối tới PostgreSQL server qua:

```text
TCP/IP
```

hoặc local:

```text
Unix-domain socket
```

---

# 4. PostgreSQL terminology cực kỳ quan trọng

Có một hierarchy rất dễ nhầm:

```text
PostgreSQL cluster / instance
│
├── Database A
│   ├── schema public
│   ├── schema app
│   └── schema audit
│
├── Database B
│   └── ...
│
└── cluster-wide roles
```

Lưu ý PostgreSQL dùng từ **database cluster** theo nghĩa:

> Một collection các databases được quản lý bởi một PostgreSQL server installation/data directory.

Nó **không đồng nghĩa** với HA cluster nhiều servers.

---

# 5. Server / instance

Một running PostgreSQL installation có:

```text
postgres server process
data directory
configuration
databases
roles
```

Ví dụ:

```bash
ps -ef | grep '[p]ostgres'
```

Có thể thấy nhiều processes:

```text
postgres
postgres: checkpointer
postgres: background writer
postgres: walwriter
postgres: autovacuum launcher
postgres: app_user appdb ...
```

PostgreSQL dùng process-based architecture: một client connection bình thường có một backend server process phục vụ nó, ngoài các background/parallel processes khác.

---

# 6. Installation model

PostgreSQL có thể được chạy bằng ít nhất hai mô hình phổ biến:

```text
OS package
```

hoặc:

```text
Container
```

Syllabus yêu cầu awareness cả hai.

---

# 7. OS package installation

Ví dụ Debian/Ubuntu thường bắt đầu bằng:

```bash
sudo apt install postgresql
```

RHEL-family có thể dùng:

```bash
sudo dnf install ...
```

nhưng package name, initialization procedure và service unit **phụ thuộc distro/repository/version**.

Không học thuộc:

```text
postgresql.service
```

là universal service name.

Có hệ thống dùng những unit kiểu:

```text
postgresql
postgresql-18
postgresql@18-main
```

Do đó inspection trước:

```bash
systemctl list-unit-files | grep -i postgres
```

và:

```bash
systemctl status <actual-unit>
```

---

# 8. `initdb`

Một PostgreSQL cluster mới cần được initialized.

Core tool:

```bash
initdb
```

Concept:

```text
empty directory
      |
    initdb
      |
      v
PostgreSQL data directory
├── database system catalogs
├── pg_hba.conf
├── postgresql.conf
├── WAL directories
└── internal structures
```

Không chạy `initdb` lên data directory đang chứa production database.

Đây là destructive-risk area.

---

# 9. Containerized PostgreSQL

Concept:

```text
Docker/Podman
    |
    v
PostgreSQL container
    |
    +---- container filesystem
    |
    +---- persistent volume
```

Demo-only example:

```bash
docker run \
  --name demo-postgres \
  -e POSTGRES_PASSWORD='example-only' \
  -p 127.0.0.1:5432:5432 \
  -d postgres:<approved-major>
```

**Không dùng cách truyền real production password như trên**, vì command line/shell history có thể làm lộ secret.

Quan trọng hơn:

> Database container nhưng data chỉ nằm trong ephemeral container filesystem thì container bị remove có thể kéo theo mất data.

Production-like design cần:

```text
Container
    |
    v
Persistent storage
```

Exact environment variables, `PGDATA` path và volume mount phải kiểm tra documentation của image/version đang dùng, không hard-code từ tutorial cũ.

---

# 10. Container không thay đổi bản chất PostgreSQL

Dù package:

```text
systemd → postgres
```

hay:

```text
container runtime → postgres
```

bên trong vẫn có:

```text
PostgreSQL server
roles
databases
schemas
TCP/socket
transactions
WAL
logs
backup requirements
```

Container chỉ thay đổi packaging/runtime boundary.

---

# 11. `psql`

`psql` là PostgreSQL interactive terminal/client.

Ví dụ:

```bash
psql -h db.example.internal \
     -p 5432 \
     -U app_user \
     -d appdb
```

Breakdown:

```text
-h = host
-p = port
-U = database role/user
-d = database
```

Sau connection:

```text
appdb=>
```

---

# 12. Useful `psql` meta-commands

Các command bắt đầu bằng `\` là `psql` commands, không phải SQL standard.

```text
\conninfo    current connection
\l           list databases
\du          list roles
\dn          list schemas
\dt          list tables
\d table     describe relation
\dp          access privileges
\q           quit
```

Ví dụ:

```text
\conninfo
```

giúp chứng minh:

```text
đang connect database nào?
user nào?
host nào?
port nào?
```

Đây là evidence rất hữu ích khi troubleshooting.

---

# 13. SQL vs `psql` command

Đừng nhầm:

```sql
SELECT current_user;
```

là SQL.

Trong khi:

```text
\du
```

là command do `psql` client xử lý.

Nếu dùng JDBC:

```text
\du
```

không phải valid SQL gửi tới server.

---

# 14. Database roles

PostgreSQL authorization xoay quanh **roles**.

PostgreSQL hiện không còn khái niệm internal riêng biệt kiểu "user object" và "group object": role có thể đóng vai trò user, group, hoặc cả hai. [PostgreSQL](https://www.postgresql.org/docs/current/user-manag.html?utm_source=chatgpt.com)

Ví dụ:

```sql
CREATE ROLE app_runtime;
```

Role này mặc định chưa nhất thiết login được.

---

# 15. LOGIN role

Để role dùng làm login identity:

```sql
CREATE ROLE app_runtime LOGIN;
```

Hoặc:

```sql
ALTER ROLE app_runtime LOGIN;
```

Conceptually:

```text
ROLE
 |
 +-- LOGIN?
       |
       +-- yes → can initiate DB session
```

`CREATE USER` trong PostgreSQL về bản chất là convenience form tương tự `CREATE ROLE ... LOGIN`.

Mental model nên là:

```text
PostgreSQL User
≈
Role with LOGIN
```

---

# 16. PostgreSQL role ≠ Linux user

Ví dụ Linux:

```text
postgres
```

có thể là OS service account.

PostgreSQL cũng có role:

```text
postgres
```

nhưng chúng là hai security namespaces khác nhau.

Official documentation cũng nhấn mạnh database roles hoàn toàn conceptually separate với operating-system users. [PostgreSQL](https://www.postgresql.org/docs/current/database-roles.html?utm_source=chatgpt.com)

---

# 17. Role attributes

Một role có thể có các attributes như:

```text
LOGIN
SUPERUSER
CREATEDB
CREATEROLE
REPLICATION
BYPASSRLS
CONNECTION LIMIT
```

Application runtime role thường **không nên** có:

```text
SUPERUSER
CREATEDB
CREATEROLE
BYPASSRLS
```

Ví dụ anti-pattern:

```sql
CREATE ROLE app_user
LOGIN
SUPERUSER;
```

chỉ vì application gặp permission error.

Đây là equivalent database version của:

```bash
chmod -R 777
```

---

# 18. Least privilege role design

Một pattern tốt hơn:

```text
migration/owner role
        |
        | owns schema/objects
        |
        +----------------+
                         |
                  runtime role
                  limited CRUD
```

Ví dụ:

```text
app_owner
  → owns schema/tables

app_runtime
  → CONNECT
  → USAGE schema
  → SELECT/INSERT/UPDATE/DELETE required tables
```

Application thường **không cần own database objects**.

---

# 19. Password setting

Có thể:

```sql
CREATE ROLE app_runtime
LOGIN
PASSWORD '<secret>';
```

nhưng đưa cleartext password vào terminal/script/log có risk.

Trong interactive `psql`, cách an toàn hơn thường là:

```text
\password app_runtime
```

để tránh password xuất hiện trực tiếp trong command history/query text.

Current PostgreSQL documentation khuyến nghị SCRAM-SHA-256; MD5 password authentication đã deprecated trong PostgreSQL 18 và dự kiến bị remove trong future release. [PostgreSQL](https://www.postgresql.org/docs/18/auth-password.html?utm_source=chatgpt.com)

---

# 20. Database

Database là isolation boundary tương đối lớn.

Ví dụ:

```sql
CREATE DATABASE appdb;
```

List:

```text
\l
```

Check:

```sql
SELECT current_database();
```

Một connection cụ thể connect tới **một database**.

Bạn không viết đơn giản:

```sql
SELECT * FROM another_database.schema.table;
```

theo kiểu cross-database giống một số RDBMS khác.

---

# 21. Schema

Schema là namespace **bên trong database**.

Ví dụ:

```text
appdb
├── public.orders
└── audit.orders
```

Hai table cùng tên:

```text
orders
```

có thể coexist vì khác schema.

Fully qualified names:

```sql
SELECT *
FROM app.orders;
```

versus:

```sql
SELECT *
FROM audit.orders;
```

---

# 22. Database vs schema

Đây là câu exam thường dễ nhầm:

```text
Cluster
  ↓
Database
  ↓
Schema
  ↓
Table
```

Không phải:

```text
schema → database
```

---

# 23. Create schema

```sql
CREATE SCHEMA app;
```

Hoặc owner rõ ràng:

```sql
CREATE SCHEMA app
AUTHORIZATION app_owner;
```

---

# 24. `search_path`

Nếu query:

```sql
SELECT *
FROM orders;
```

PostgreSQL phải tìm `orders` trong schemas thuộc:

```text
search_path
```

Check:

```sql
SHOW search_path;
```

Safer operational reasoning:

```sql
SELECT *
FROM app.orders;
```

thường rõ ràng hơn khi troubleshoot.

---

# 25. Least-privilege privileges

Giả sử application cần connect tới `appdb`.

```sql
GRANT CONNECT
ON DATABASE appdb
TO app_runtime;
```

Trong `appdb`:

```sql
GRANT USAGE
ON SCHEMA app
TO app_runtime;
```

Sau đó table permissions:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO app_runtime;
```

Nếu application dùng sequences:

```sql
GRANT USAGE, SELECT
ON ALL SEQUENCES IN SCHEMA app
TO app_runtime;
```

Important:

```text
CONNECT database
```

không tự động cấp:

```text
SELECT table
```

---

# 26. Authentication vs authorization

Phải phân biệt:

### Authentication

> Bạn là ai?

Ví dụ:

```text
password authentication succeeded
```

### Authorization

> Bạn được phép làm gì?

Ví dụ:

```text
permission denied for table orders
```

Flow:

```text
TCP connection
↓
authentication
↓
session established
↓
authorization checked per operation
```

---

# 27. PostgreSQL connectivity mental model

Application connection:

```text
Java
 |
 | JDBC URL:
 | jdbc:postgresql://db.internal:5432/appdb
 |
 v
DNS
 |
 v
Network
 |
 v
PostgreSQL :5432
 |
 v
pg_hba.conf
 |
 v
Role authentication
 |
 v
Database CONNECT privilege
 |
 v
Schema/table privileges
```

Lỗi ở mỗi layer có symptom khác nhau.

---

# 28. Listen address

`postgresql.conf` controls server networking.

Check:

```sql
SHOW listen_addresses;
```

```sql
SHOW port;
```

Default TCP port thường:

```text
5432
```

Một server chỉ listening localhost sẽ không tự nhiên nhận remote connection dù `pg_hba.conf` cho phép.

Official PostgreSQL documentation lưu ý remote TCP/IP không thể hoạt động nếu `listen_addresses` không phù hợp. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

---

# 29. Verify socket from Linux

```bash
ss -lntp | grep ':5432'
```

Có thể thấy:

```text
127.0.0.1:5432
```

hoặc:

```text
0.0.0.0:5432
```

hoặc một private IP cụ thể.

Như Module 1:

```text
listen
≠
reachable remotely
```

vì còn:

```text
firewall
AWS Security Group
routing
NACL
```

---

# 30. Local socket connection

Nếu chạy:

```bash
psql -U postgres
```

mà không `-h`, trên Unix-like system client thường thử Unix-domain socket theo configuration/defaults.

Trong khi:

```bash
psql -h 127.0.0.1 ...
```

ép TCP/IP.

Đây là distinction rất quan trọng.

Có trường hợp:

```text
local socket works
TCP fails
```

không hề mâu thuẫn.

---

# 31. `pg_hba.conf`

HBA:

> Host-Based Authentication.

File này kiểm soát **client authentication policy**.

Typical record:

```text
TYPE     DATABASE   USER          ADDRESS          METHOD
```

Ví dụ:

```conf
host     appdb      app_runtime   10.0.10.0/24    scram-sha-256
```

Meaning:

```text
TCP connection
database appdb
role app_runtime
client from 10.0.10.0/24
authenticate with SCRAM-SHA-256
```

PostgreSQL đọc `pg_hba.conf` khi startup và khi server được reload/SIGHUP; không nhất thiết phải restart DB chỉ để apply HBA change. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html?utm_source=chatgpt.com)

---

# 32. `local`, `host`, `hostssl`

Ví dụ:

```conf
local   all      all                         peer
```

Unix socket.

```conf
host    appdb    app_runtime   10.0.10.0/24  scram-sha-256
```

TCP.

```conf
hostssl appdb    app_runtime   10.0.10.0/24  scram-sha-256
```

TCP but requires SSL connection matching.

`hostssl` chỉ useful nếu SSL được enabled/configured appropriately. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

---

# 33. Rule ordering cực kỳ quan trọng

`pg_hba.conf` không phải firewall với "thử rule tiếp theo nếu auth fail".

PostgreSQL dùng **first matching record**.

Nếu rule match:

```text
connection type
address
database
user
```

thì PostgreSQL dùng rule đó.

Nếu authentication theo rule đó fail:

```text
STOP
```

không fall through xuống rule tiếp theo. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

Ví dụ:

```conf
host appdb app_runtime 10.0.0.0/8    reject
host appdb app_runtime 10.0.10.0/24  scram-sha-256
```

Client:

```text
10.0.10.25
```

đã match rule đầu tiên.

Rule thứ hai không cứu nó.

---

# 34. Dangerous HBA example

```conf
host all all 0.0.0.0/0 trust
```

Meaning gần như:

```text
mọi IPv4 source
mọi database
mọi role
không password authentication
```

Đây là cực kỳ nguy hiểm.

Không dùng:

```text
trust everything
```

để "test nhanh" trên environment reachable bởi người khác.

---

# 35. `scram-sha-256`

Current secure default direction:

```conf
host appdb app_runtime 10.0.10.0/24 scram-sha-256
```

SCRAM-SHA-256 dùng challenge-response và password hashes thay vì gửi password plaintext qua network; PostgreSQL hiện xem nó là password method an toàn nhất trong các built-in password authentication methods phổ biến. [PostgreSQL](https://www.postgresql.org/docs/18/auth-password.html?utm_source=chatgpt.com)

---

# 36. MD5 awareness

Tutorial cũ thường có:

```conf
host all all 10.0.0.0/8 md5
```

Nhưng **không nên học đây là current recommendation**.

PostgreSQL 18 đã deprecate support cho MD5-encrypted passwords và khuyến nghị migration sang SCRAM. [PostgreSQL](https://www.postgresql.org/docs/18/auth-password.html?utm_source=chatgpt.com)

---

# 37. Validate `pg_hba.conf`

Useful system view:

```sql
SELECT *
FROM pg_hba_file_rules;
```

Điều này đặc biệt useful để phát hiện malformed rules.

Official docs ghi rõ rows có non-null `error` giúp phát hiện problems trong file. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html?utm_source=chatgpt.com)

Sau controlled change:

```sql
SELECT pg_reload_conf();
```

là một cách request reload.

Hoặc platform tooling:

```bash
pg_ctl reload ...
```

Tùy environment.

---

# 38. `pg_hba.conf` không cấp SQL privileges

Ví dụ HBA:

```conf
host appdb app_runtime 10.0.10.0/24 scram-sha-256
```

chỉ có nghĩa:

> connection này được phép tiến tới authentication theo SCRAM.

Nó **không có nghĩa**:

```text
app_runtime may SELECT every table
```

Sau authentication, SQL privilege system vẫn áp dụng.

Official PostgreSQL cũng nói client vừa phải pass HBA vừa cần `CONNECT` privilege vào database. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html?utm_source=chatgpt.com)

---

# 39. Connection troubleshooting

## Case A — DNS fails

```bash
getent hosts db.internal.example
```

Không resolve:

```text
DNS/config problem
```

chưa cần nhìn password.

---

## Case B — timeout

```text
connection timed out
```

Suspect:

```text
routing
Security Group/firewall
wrong IP
network ACL
silent packet drop
```

---

## Case C — refused

```text
connection refused
```

Suspect:

```text
no listener
wrong port
service stopped
wrong bind address
active reject
```

---

## Case D — HBA rejects

Typical error concept:

```text
no pg_hba.conf entry ...
```

Evidence:

```text
reached PostgreSQL
PostgreSQL evaluated authentication configuration
no acceptable matching rule
```

Không quay lại troubleshoot DNS đầu tiên.

---

## Case E — bad password

```text
password authentication failed
```

Đã chứng minh rất nhiều layers hoạt động:

```text
DNS
network
listener
HBA matching
authentication exchange
```

Failure nằm ở auth identity/credential area.

---

## Case F — permission denied

```text
permission denied for table orders
```

Connection thành công.

Problem nằm ở:

```text
authorization
```

---

# 40. Health check không chỉ là "process running"

PostgreSQL health có nhiều layers:

```text
OS service
↓
process
↓
port/socket
↓
PostgreSQL accepting connections
↓
authentication
↓
SQL execution
↓
application DB operation
```

---

# 41. `pg_isready`

Tool:

```bash
pg_isready
```

Remote:

```bash
pg_isready \
  -h db.internal.example \
  -p 5432
```

Nó kiểm tra PostgreSQL server connection status/readiness.

Nhưng:

```text
pg_isready success
```

không chứng minh:

```text
app credentials valid
table permissions correct
business query works
```

---

# 42. SQL health check

Một simple query:

```sql
SELECT 1;
```

Hoặc:

```sql
SELECT
    current_database(),
    current_user,
    now();
```

Expected:

```text
connection usable
query execution works
identity/database correct
```

---

# 43. Check read-only state

```sql
SHOW transaction_read_only;
```

Useful trong environments có primary/replica hoặc recovery states.

Nếu application cần writes nhưng session/database state effectively read-only:

```text
SELECT works
INSERT fails
```

Health test chỉ `SELECT 1` có thể bỏ sót vấn đề này.

---

# 44. Backup fundamentals

Backup answer câu hỏi:

> Nếu database/data hiện tại bị mất/corrupt/changed incorrectly, ta có recover được không?

Backup strategy không chỉ là chạy command.

Nó gồm:

```text
what data?
how often?
where stored?
retention?
encryption?
access?
restore procedure?
restore time?
restore verified?
```

PostgreSQL documentation chia backup thành ba nhóm lớn:

````text
SQL dump
filesystem-level backup
continuous archiving / PITR
``` :chatgpt-content-reference{index="13"}


Trong syllabus fresher này, trọng tâm phù hợp nhất là **logical backup/restore**.

---

# 45. `pg_dump`

`pg_dump` export **một database**.

Plain SQL:

```bash
pg_dump \
  -h db.internal.example \
  -U backup_user \
  -d appdb \
  > appdb.sql
````

Or:

```bash
pg_dump \
  -h db.internal.example \
  -U backup_user \
  -d appdb \
  -f appdb.sql
```

Plain format chứa SQL cần thiết để reconstruct objects/data.

Official docs xác nhận `pg_dump` chỉ dump một database và tạo consistent export ngay cả khi database đang được dùng concurrently. [PostgreSQL](https://www.postgresql.org/docs/current/app-pgdump.html?trk=article-ssr-frontend-pulse_little-text-block&utm_source=chatgpt.com)

---

# 46. Custom-format dump

Very useful:

```bash
pg_dump \
  -Fc \
  -d appdb \
  -f appdb.dump
```

`-F c` / `-Fc`:

```text
custom archive format
```

Benefits:

````text
pg_restore support
selective restore
reordering
compression by default
``` :chatgpt-content-reference{index="15"}


---

# 47. Plain restore

Nếu backup là:

```text
appdb.sql
````

restore bằng `psql`:

```bash
psql \
  -d appdb_restore \
  -f appdb.sql
```

Concept:

```text
SQL script
   |
   v
psql
   |
   v
PostgreSQL executes statements
```

---

# 48. `pg_restore`

Custom archive:

```bash
pg_restore \
  -d appdb_restore \
  appdb.dump
```

Official docs:

> `pg_restore` restores archives created by `pg_dump` non-plain formats. [PostgreSQL](https://www.postgresql.org/docs/18/app-pgrestore.html?utm_source=chatgpt.com)

Do not use:

```bash
psql -f appdb.dump
```

for a custom archive.

---

# 49. Restore into clean test database

Safer validation idea:

```text
production backup
      |
      v
new isolated restore DB
      |
      v
validate objects + data
```

Instead of immediately running destructive:

```text
DROP production database
```

---

# 50. `pg_dumpall`

Difference:

```text
pg_dump
→ one database
```

```text
pg_dumpall
→ cluster-wide logical dump capability
→ can include globals such as roles/tablespaces
```

Official documentation explicitly notes `pg_dump` alone does not capture cluster-wide global objects such as roles. [PostgreSQL](https://www.postgresql.org/docs/current/app-pgdump.html?trk=article-ssr-frontend-pulse_little-text-block&utm_source=chatgpt.com)

---

# 51. Backup file permissions

A database dump may contain:

```text
customer data
personal data
credentials/config values stored in DB
business secrets
```

Therefore:

```bash
ls -l appdb.dump
```

matters.

Don't create world-readable backup:

```text
-rw-r--r--
```

without considering sensitivity.

Backup security is part of database security.

---

# 52. Backup ≠ verified backup

This:

```text
pg_dump exit code 0
```

is evidence that dump command completed.

It is **not complete evidence** that disaster recovery will work.

Better model:

```text
backup created
↓
integrity/metadata checked
↓
restore attempted in isolated environment
↓
schema/data validated
↓
application/business validation
```

A backup that has never been restore-tested is only partially proven.

---

# PHẦN II — DAY 16

# SQL / Log Diagnostics, Sessions, Locks, Capacity & EXPLAIN

## 53. Mental model của database troubleshooting

Một slow application request:

```text
Client
 ↓
Java application
 ↓
connection pool
 ↓
PostgreSQL session
 ↓
SQL
 ↓
planner
 ↓
locks/waits
 ↓
CPU/memory/disk
```

"Database slow" có thể thực chất là:

```text
no free DB connection
lock wait
bad query plan
disk saturation
network
too many sessions
large query
application transaction left open
```

---

# 54. PostgreSQL sessions

Một successful client connection tạo database session/backend.

Relevant dimensions:

```text
PID
database
user
application
client IP
state
query
query start time
wait event
```

Primary view:

```text
pg_stat_activity
```

Current PostgreSQL docs định nghĩa view này có một row cho mỗi relevant server process/backend và cung cấp database, PID, user, application, client address, activity information, v.v. [PostgreSQL](https://www.postgresql.org/docs/current/monitoring-stats.htm?utm_source=chatgpt.com)

---

# 55. Basic session query

```sql
SELECT
    pid,
    datname,
    usename,
    application_name,
    client_addr,
    state,
    query_start,
    query
FROM pg_stat_activity
ORDER BY query_start;
```

Operationally:

```text
Who?
From where?
Connected where?
Doing what?
Since when?
```

---

# 56. Useful session states

Common states include:

```text
active
idle
idle in transaction
```

### `active`

Backend đang execute query.

### `idle`

Session connected nhưng currently không execute query.

Không tự động là problem.

### `idle in transaction`

Client đã mở transaction nhưng không làm gì hiện tại.

Potentially dangerous vì transaction có thể giữ:

```text
locks
old transaction snapshot
resources
```

và ảnh hưởng vacuum/database behavior.

---

# 57. Find long-running active queries

Example:

```sql
SELECT
    pid,
    usename,
    now() - query_start AS duration,
    wait_event_type,
    wait_event,
    query
FROM pg_stat_activity
WHERE state = 'active'
  AND pid <> pg_backend_pid()
ORDER BY duration DESC;
```

Không nhìn duration rồi kill ngay.

Một 20-minute query có thể là:

```text
expected report
backup-related workload
maintenance
approved batch
```

Context matters.

---

# 58. Wait events

Useful columns:

```text
wait_event_type
wait_event
```

Concept:

```text
query active
```

không nhất thiết nghĩa CPU đang execute SQL continuously.

Nó có thể đang chờ:

```text
Lock
IO
client
IPC
...
```

Wait information giúp phân biệt:

```text
slow because computing
```

vs:

```text
slow because waiting
```

---

# 59. Query text security

`pg_stat_activity.query` có thể chứa operational/business information.

Query logs có thể chứa:

```text
user input
identifiers
potential sensitive values
```

Do đó access tới monitoring views/logs cũng cần least privilege.

Không paste raw production queries chứa secrets/customer data vào random chat/ticket.

---

# 60. Locks

Database concurrency cần locking.

Ví dụ:

Transaction A:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100
WHERE id = 1;
```

Chưa commit.

Transaction B:

```sql
UPDATE accounts
SET balance = balance + 50
WHERE id = 1;
```

B có thể phải wait.

Lock không phải bug.

Nó là mechanism để giữ consistency.

---

# 61. Blocking

Mental model:

```text
Session A
holds lock
    |
    v
resource
    ^
    |
Session B
waits
```

Question:

```text
Who is blocked?
Who is blocking?
How long?
What transaction/query?
Is blocker expected?
```

---

# 62. `pg_locks`

PostgreSQL exposes:

```text
pg_locks
```

để inspect lock information.

Basic:

```sql
SELECT *
FROM pg_locks;
```

nhưng output khá low-level.

Một fresher không nên chỉ dump 1000 rows rồi không biết đọc.

---

# 63. `pg_blocking_pids()`

Cách practical hơn để tìm blocker:

```sql
SELECT
    pid,
    usename,
    state,
    pg_blocking_pids(pid) AS blocking_pids,
    query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Interpretation:

```text
pid = blocked backend
blocking_pids = sessions blocking it
```

Sau đó inspect blockers.

---

# 64. Blocking chain

Có thể:

```text
PID 100
holds lock
   ↓
PID 200 waits

PID 200
also holds another lock
   ↓
PID 300 waits
```

Tạo chain:

```text
100 → 200 → 300
```

Khi troubleshoot phải tìm **root blocker**, không chỉ session cuối cùng bị chậm.

---

# 65. Deadlock

Khác blocking thông thường:

```text
Transaction A
holds X
waits Y

Transaction B
holds Y
waits X
```

Cycle:

```text
A → B
↑   ↓
└───┘
```

PostgreSQL có deadlock detection và sẽ abort một transaction để break cycle.

Application có thể thấy error kiểu:

```text
deadlock detected
```

Solution không phải "increase timeout forever".

Root cause thường cần xem:

```text
transaction ordering
application design
concurrent access
```

---

# 66. `pg_cancel_backend`

Nếu có authorization/runbook:

```sql
SELECT pg_cancel_backend(<pid>);
```

Purpose:

```text
cancel current query
```

Session có thể vẫn tồn tại.

---

# 67. `pg_terminate_backend`

Stronger:

```sql
SELECT pg_terminate_backend(<pid>);
```

Meaning approximately:

```text
terminate session/backend
```

Có thể rollback transaction và disrupt application.

Không dùng như casual L1 command.

---

# 68. Không `kill -9` PostgreSQL backend tùy tiện

Bad first response:

```bash
sudo kill -9 <postgres-backend-pid>
```

PostgreSQL server treats abnormal backend termination seriously; có thể kéo theo wider recovery/restart behavior tùy situation.

Preferred:

```text
understand session
↓
approved PostgreSQL cancellation/termination
↓
validate
```

L1 không tự ý OS-kill database processes.

---

# 69. SQL diagnostics

Khi query chậm:

```text
What SQL?
How long?
How often?
Blocked or running?
Expected row count?
Recent change?
Plan?
Data volume?
Resource capacity?
```

Đừng bắt đầu bằng:

```text
Add index everywhere
```

---

# 70. PostgreSQL logs

Logs có thể ghi:

```text
startup/shutdown
authentication failure
errors
deadlocks
statements depending config
slow queries depending config
checkpoints
recovery
```

Location phụ thuộc packaging/configuration:

```sql
SHOW log_destination;
SHOW logging_collector;
SHOW log_directory;
```

Không giả định log luôn nằm:

```text
/var/log/postgresql/postgresql.log
```

---

# 71. Useful logging settings awareness

Các setting cần nhận diện:

```text
log_min_duration_statement
log_statement
log_connections
log_disconnections
log_lock_waits
log_line_prefix
```

Không cần thuộc mọi default.

---

# 72. `log_min_duration_statement`

Concept:

```text
log SQL taking >= threshold
```

Useful để tìm slow statements.

Nhưng threshold quá thấp có thể:

```text
generate huge logs
increase I/O
expose sensitive query data
```

Do đó production change cần approval/rollback.

---

# 73. `log_statement`

Có thể configure statement logging levels.

Bật overly verbose SQL logging trên busy production DB có thể tạo:

```text
performance overhead
disk growth
sensitive data leakage
```

Do đó:

> More logging is not always safer.

---

# 74. Correlating logs

Good troubleshooting:

```text
application error timestamp
       |
       v
PostgreSQL logs same timestamp
       |
       v
pg_stat_activity / locks
       |
       v
host metrics
```

Không đọc database log tách rời application timeline.

---

# 75. Capacity checks

Capacity không chỉ là:

```bash
df -h
```

PostgreSQL capacity includes:

```text
filesystem space
database size
table/index growth
connection capacity
memory/CPU/IO pressure
WAL/temp growth
```

Ở fresher level, focus:

```text
disk
database/table size
connection count
host resources
```

---

# 76. Linux filesystem capacity

```bash
df -h
```

Inodes:

```bash
df -i
```

Một database filesystem gần 100% là high-risk.

PostgreSQL cần free space cho:

```text
data growth
WAL
temporary files
maintenance
indexes
```

---

# 77. Database size

```sql
SELECT pg_size_pretty(
    pg_database_size('appdb')
);
```

Official PostgreSQL exposes `pg_database_size()` để tính total disk space của database. [PostgreSQL](https://www.postgresql.org/docs/18/functions-admin.html?utm_source=chatgpt.com)

---

# 78. Table total size

```sql
SELECT pg_size_pretty(
    pg_total_relation_size('app.orders')
);
```

`pg_total_relation_size` gồm table cùng indexes và TOAST-related storage. [PostgreSQL](https://www.postgresql.org/docs/18/functions-admin.html?utm_source=chatgpt.com)

Useful:

```sql
SELECT
    relname,
    pg_size_pretty(
        pg_total_relation_size(relid)
    ) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

---

# 79. Connection capacity

Check:

```sql
SHOW max_connections;
```

Current sessions:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

By database:

```sql
SELECT
    datname,
    count(*)
FROM pg_stat_activity
GROUP BY datname
ORDER BY count(*) DESC;
```

By role:

```sql
SELECT
    usename,
    count(*)
FROM pg_stat_activity
GROUP BY usename
ORDER BY count(*) DESC;
```

---

# 80. Why connection count matters

If application creates unbounded DB connections:

```text
traffic rises
↓
application opens connections
↓
PostgreSQL processes increase
↓
memory/context switching rise
↓
max_connections reached
↓
new connections fail
```

Solution không nhất thiết là:

```text
increase max_connections
```

Có thể root cause là:

```text
bad connection pool settings
connection leak
traffic spike
downstream latency
```

---

# 81. Supplementary — connection pooling

Application side:

```text
Tomcat/Java
   |
   v
Connection pool
├── conn 1
├── conn 2
├── conn 3
└── ...
   |
   v
PostgreSQL
```

Important relationship:

```text
10 app servers
×
100 max pool connections
=
potential 1000 DB sessions
```

Nếu PostgreSQL:

```text
max_connections = 300
```

configuration rõ ràng có vấn đề.

Always think **fleet-wide capacity**, không chỉ một JVM.

---

# 82. `EXPLAIN`

`EXPLAIN` hỏi PostgreSQL:

> Planner dự định execute query này như thế nào?

Example:

```sql
EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 123;
```

Có thể:

```text
Seq Scan on orders
  Filter: (customer_id = 123)
```

hoặc:

```text
Index Scan using idx_orders_customer_id on orders
  Index Cond: (customer_id = 123)
```

Official docs xác định `EXPLAIN` hiển thị execution plan mà planner tạo, bao gồm scan methods, joins, estimated costs và row estimates. [PostgreSQL](https://www.postgresql.org/docs/current/using-explain.html?utm_source=chatgpt.com)

---

# 83. Planner

PostgreSQL evaluates possible strategies:

```text
Seq Scan?
Index Scan?
Bitmap Scan?
Nested Loop?
Hash Join?
Merge Join?
Sort?
Aggregate?
```

và chọn estimated lowest-cost reasonable plan.

Cost không phải milliseconds trực tiếp.

```text
cost=...
```

là internal planner cost unit.

---

# 84. Example EXPLAIN

```text
Seq Scan on orders
(cost=0.00..1850.00 rows=10 width=64)
```

Key concepts:

```text
startup cost
total estimated cost
estimated rows
estimated row width
```

Fresher không cần trở thành DBA query optimizer expert.

Nhưng phải hiểu:

> `EXPLAIN` cho estimated plan.

---

# 85. Sequential Scan

```text
Seq Scan
```

means scan table sequentially.

Không phải luôn bad.

Nếu table:

```text
100 rows
```

Seq Scan có thể rẻ hơn index.

Nếu query cần:

```text
90% rows
```

Seq Scan cũng có thể hợp lý.

Anti-pattern:

> Thấy Seq Scan = lập tức tạo index.

---

# 86. Index Scan

```text
Index Scan
```

can be useful when predicate selective.

Example:

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

Then:

```sql
EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 123;
```

may choose that index.

Nhưng index có cost:

```text
disk space
INSERT/UPDATE/DELETE overhead
maintenance
```

Không tạo index tùy tiện ở L1.

---

# 87. Statistics

Planner cần statistics để estimate data distribution.

Command:

```sql
ANALYZE orders;
```

Official docs:

> `ANALYZE` collects statistics which query planner uses to choose efficient plans. [PostgreSQL](https://www.postgresql.org/docs/current/sql-analyze.html?utm_source=chatgpt.com)

Nếu statistics stale/bad:

```text
actual data
   ≠
planner's model
```

→ poor estimates → possible poor plan.

---

# 88. `EXPLAIN ANALYZE`

Difference cực quan trọng:

```sql
EXPLAIN SELECT ...
```

Planner only.

```sql
EXPLAIN ANALYZE SELECT ...
```

**Actually executes the query** and reports actual timing/rows in addition to estimates. [PostgreSQL](https://www.postgresql.org/docs/current/using-explain.html?utm_source=chatgpt.com)

---

# 89. Danger of `EXPLAIN ANALYZE`

Never casually execute:

```sql
EXPLAIN ANALYZE
DELETE FROM orders;
```

because `ANALYZE` here means:

> execute statement and measure it.

It is **not** "analyze without making changes".

The `DELETE` is real.

Likewise:

```sql
EXPLAIN ANALYZE UPDATE ...
```

executes UPDATE.

Production rule:

> Treat `EXPLAIN ANALYZE` as execution, not inspection.

---

# 90. Even SELECT can be risky

```sql
EXPLAIN ANALYZE
SELECT ...
```

doesn't modify rows normally, but query could:

```text
consume lots of CPU
read huge amounts of disk
hold locks
run minutes/hours
```

So use runbook/non-production when query impact is unknown.

---

# 91. Estimated vs actual rows

Example conceptual:

```text
estimated rows = 10
actual rows    = 1,000,000
```

Huge mismatch can point toward:

```text
statistics issue
data distribution issue
complex predicate/correlation
```

This is useful DBA evidence.

An L1 operator collects it only when safe/approved, then escalates.

---

# 92. `BUFFERS`

Supplementary:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

Adds buffer usage information useful for deeper performance analysis.

This already approaches DBA/performance engineering territory.

Know what it is; don't blindly run heavy query analysis in production.

---

# 93. L1 escalation boundary

L1 should be able to:

```text
identify affected DB/application
check connectivity
check DB service
check capacity
inspect sessions
identify blockers
collect query/log evidence
run approved health queries
confirm recent changes
perform runbook remediation
```

---

# 94. Things L1 should not casually do

```text
DROP database/table
ALTER production schema
create random indexes
VACUUM FULL
REINDEX large production objects
change max_connections blindly
change memory configuration blindly
terminate unknown sessions
kill postgres process
delete WAL
edit DB files manually
disable authentication
```

---

# 95. Blocked application scenario

Symptom:

```text
API request hangs
```

Evidence:

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    pg_blocking_pids(pid),
    query
FROM pg_stat_activity;
```

Find:

```text
API PID 200
wait_event_type = Lock
blocking_pids = {100}
```

Now investigate PID 100.

Do **not**:

```text
restart PostgreSQL
```

as first response.

---

# 96. Disk nearly full scenario

Symptom:

```text
DB errors + host disk 98%
```

Collect:

```bash
df -h
df -i
```

SQL:

```sql
SELECT pg_size_pretty(
    pg_database_size(current_database())
);
```

Then identify:

```text
database?
WAL?
logs?
other OS files?
temp?
```

Do not randomly:

```bash
rm -rf <PostgreSQL data files>
```

or:

```bash
rm pg_wal/*
```

That can destroy recoverability/database integrity.

---

# 97. Slow query scenario

Correct flow:

```text
Confirm actual symptom
↓
Find query/session
↓
Blocked?
↓
Host capacity?
↓
Duration/frequency?
↓
Relevant log evidence?
↓
Safe EXPLAIN?
↓
Collect plan
↓
Escalate if query/index/schema tuning needed
```

Not:

```text
slow
↓
restart DB
```

---

# PHẦN III — DAY 17

# Prometheus, node_exporter, Grafana & Alerting

## 98. Why monitoring?

Without monitoring:

```text
User reports problem
↓
admin begins investigation
```

With monitoring:

```text
metric deviates
↓
alert fires
↓
operator investigates
↓
maybe before users notice
```

Monitoring answers:

```text
Is it healthy?
How has it changed?
When did problem start?
How severe?
What else changed?
```

---

# 99. Observability signals

Broader industry model often includes:

```text
Metrics
Logs
Traces
```

Syllabus Day 17 focuses mainly:

```text
Metrics
↓
Prometheus
↓
Grafana
↓
Alerts
```

Logs remain important evidence from previous Days.

---

# 100. Metric

Metric example:

```text
node_memory_MemAvailable_bytes
```

might produce:

```text
4123456789
```

at a timestamp.

Over time:

```text
timestamp    value
10:00        4.2 GB
10:01        4.1 GB
10:02        3.9 GB
...
```

This forms a time series.

---

# 101. Metric + labels

Prometheus data model concept:

```text
metric_name{label="value", ...}
```

Example:

```promql
up{job="node", instance="app01:9100"}
```

Different label sets:

```text
different time series
```

Example:

```promql
node_network_receive_bytes_total{device="eth0"}
```

versus:

```promql
node_network_receive_bytes_total{device="lo"}
```

---

# 102. Prometheus architecture

Core mental model:

```text
Linux host
   |
node_exporter :9100
   |
   | GET /metrics
   |
   v
Prometheus
   |
   +--> TSDB
   |
   +--> PromQL
   |
   +--> Alert rules
   |
   +--> Alertmanager
   |
   v
Grafana
```

Actually Grafana normally **queries Prometheus**:

```text
Grafana
  |
  | PromQL/query API
  v
Prometheus
```

Grafana is not where node_exporter normally pushes metrics.

---

# 103. Pull model

Prometheus typically performs:

```text
Prometheus
    |
    | HTTP GET /metrics
    v
target
```

This is **pull/scrape model**.

Target doesn't usually push every host metric into Prometheus itself.

---

# 104. Exporter

An exporter converts system/application information into Prometheus-compatible metrics.

Examples:

```text
node_exporter → OS/hardware
postgres_exporter → PostgreSQL  [supplementary]
application metrics endpoint
```

Syllabus explicitly requires `node_exporter`.

---

# 105. node_exporter

Official Prometheus node_exporter:

> exports hardware and operating-system metrics from Unix-like hosts. [GitHub](https://github.com/prometheus/node_exporter/blob/master/README.md?plain=1&utm_source=chatgpt.com)

Common metrics cover:

```text
CPU
memory
filesystem
network
load
kernel
```

Default HTTP port:

```text
9100
```

per official node_exporter documentation. [GitHub](https://github.com/prometheus/node_exporter/blob/master/README.md?plain=1&utm_source=chatgpt.com)

---

# 106. node_exporter endpoint

Typical:

```bash
curl http://127.0.0.1:9100/metrics
```

Output conceptually:

```text
# HELP node_memory_MemTotal_bytes ...
# TYPE node_memory_MemTotal_bytes gauge
node_memory_MemTotal_bytes 8.24e+09
...
```

This proves:

```text
node_exporter itself answering
```

not necessarily:

```text
Prometheus successfully scraping it
```

---

# 107. Exporter collectors

node_exporter uses collectors such as:

```text
cpu
filesystem
meminfo
netdev
loadavg
...
```

Some are enabled by default; others require flags.

Official documentation explicitly warns that some disabled collectors may add significant resource usage, scrape duration or cardinality, so enabling everything is not good practice. [GitHub](https://github.com/prometheus/node_exporter/blob/master/README.md?plain=1&utm_source=chatgpt.com)

---

# 108. Prometheus scrape configuration

Simplified configuration:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "linux-nodes"

    static_configs:
      - targets:
          - "app01.example:9100"
          - "db01.example:9100"
```

Official Prometheus configuration defines each `scrape_config` as a target set/job; static targets can be configured with `static_configs`, while larger environments can use service discovery. [Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/configuration/?target=_blank&utm_source=chatgpt.com)

---

# 109. `job` and `instance`

Conceptually:

```text
job
=
logical group of similar scrape targets
```

Example:

```text
job="linux-nodes"
```

`instance` usually identifies scraped target:

```text
instance="app01.example:9100"
```

Useful query:

```promql
up{job="linux-nodes"}
```

---

# 110. The `up` metric

Prometheus automatically generates scrape health metric:

```promql
up
```

Typical meaning:

```text
1 = scrape succeeded
0 = scrape failed
```

Important:

```text
up == 1
```

does **not** mean business application is healthy.

It means:

> Prometheus successfully scraped that target.

If target is node_exporter:

```text
up{job="linux-nodes"} == 1
```

means node_exporter reachable/scrapable.

Tomcat may still be dead.

PostgreSQL may still be dead.

---

# 111. PromQL

**PromQL — Prometheus Query Language.**

Official docs define it as the functional query language for selecting and aggregating time-series data. [Prometheus](https://prometheus.io/docs/prometheus/latest/querying/basics/?utm_source=chatgpt.com)

Simple:

```promql
up
```

Filter labels:

```promql
up{job="linux-nodes"}
```

Specific target:

```promql
up{
  job="linux-nodes",
  instance="app01.example:9100"
}
```

---

# 112. Gauge vs counter

Critical Prometheus concept.

## Gauge

Value can:

```text
increase
decrease
```

Example:

```text
available memory
temperature
current connections
```

## Counter

Normally monotonically increases until process/reset.

Example:

```text
total HTTP requests
CPU seconds accumulated
network bytes received
```

For counters, raw value often less useful than **rate of change**.

---

# 113. `rate()`

Example counter:

```promql
node_cpu_seconds_total
```

Rather than displaying lifetime value:

```promql
node_cpu_seconds_total
```

we commonly calculate:

```promql
rate(node_cpu_seconds_total[5m])
```

Meaning roughly:

```text
per-second average rate
over recent 5-minute range
```

Prometheus handles counter resets appropriately when using functions such as `rate()`.

---

# 114. CPU utilization example

Typical Linux CPU utilization approximation:

```promql
100 * (
  1 -
  avg by (instance) (
    rate(
      node_cpu_seconds_total{
        mode="idle"
      }[5m]
    )
  )
)
```

Mental model:

```text
CPU utilization
≈
100% - idle percentage
```

---

# 115. CPU utilization vs load average

Don't confuse:

```text
CPU usage %
```

with:

```text
load average
```

Metrics like:

```promql
node_load1
```

represent load average concept, not direct CPU percent.

High load could involve:

```text
CPU runnable work
uninterruptible waits
```

depending Linux state.

---

# 116. Memory metrics

Examples:

```promql
node_memory_MemTotal_bytes
```

```promql
node_memory_MemAvailable_bytes
```

Available percentage:

```promql
100 *
node_memory_MemAvailable_bytes
/
node_memory_MemTotal_bytes
```

Used percentage approximation:

```promql
100 *
(
  1 -
  node_memory_MemAvailable_bytes
  /
  node_memory_MemTotal_bytes
)
```

---

# 117. Why not use `MemFree` only?

Linux uses RAM for:

```text
page cache
buffers
```

So:

```text
MemFree small
```

không tự động là memory emergency.

`MemAvailable` thường hữu ích hơn để reason về memory actually available for new workload.

---

# 118. Filesystem metrics

Examples:

```promql
node_filesystem_size_bytes
```

```promql
node_filesystem_avail_bytes
```

Root filesystem free percentage:

```promql
100 *
node_filesystem_avail_bytes{mountpoint="/"}
/
node_filesystem_size_bytes{mountpoint="/"}
```

Real dashboards thường filter irrelevant pseudo filesystems.

---

# 119. Disk alert nuance

Bad alert:

```text
disk used > 80%
```

cho mọi filesystem.

Why?

```text
small filesystem
large filesystem
read-only filesystem
temporary mount
pseudo filesystem
```

có different risk.

Monitoring requires context, not just universal thresholds.

---

# 120. Network metrics

Examples:

```promql
node_network_receive_bytes_total
```

```promql
node_network_transmit_bytes_total
```

Rate:

```promql
rate(
  node_network_receive_bytes_total{
    device="eth0"
  }[5m]
)
```

---

# 121. Host signals

Syllabus says host/service/application signals.

Typical **host** signals:

```text
CPU
load
memory
swap
disk usage
disk I/O
network
filesystem
host availability
```

Node exporter is focused here.

---

# 122. Service signals

Service health might include:

```text
process running?
port open?
service responding?
restart count?
dependency available?
```

Important:

> node_exporter metrics alone do not magically prove every service is healthy.

Depending on setup, you may need:

```text
node_exporter collector
application exporter
blackbox probing
service-specific exporter
```

The latter details are **Supplementary / Beyond explicit syllabus**.

---

# 123. Application signals

For a Java web app:

```text
request rate
error rate
latency
active requests
JVM memory
GC
thread count
DB pool usage
business success/failure
health endpoint
```

The application must usually expose/instrument these metrics.

Node exporter cannot infer:

```text
HTTP 500 rate
order checkout latency
DB pool exhaustion
```

just from Linux CPU/memory.

---

# 124. Three monitoring layers

Excellent mental model:

```text
HOST
CPU/memory/disk/network
        |
        v
SERVICE
process/port/availability
        |
        v
APPLICATION
request/latency/error/business
```

One healthy layer does not guarantee next layer healthy.

---

# 125. Example failure

Metrics:

```text
CPU = 15%
memory = healthy
disk = healthy
node_exporter up = 1
```

But:

```text
Tomcat crashed
```

Therefore:

```text
Host healthy
≠
Application healthy
```

---

# 126. Another failure

Tomcat:

```text
HTTP 200 /health
```

But:

```text
checkout API HTTP 500 due DB permissions
```

Therefore:

```text
basic service health
≠
business function health
```

Monitoring should reflect important user-facing behavior.

---

# 127. Prometheus storage

Prometheus stores scraped time-series data in its local TSDB:

```text
timestamp
metric value
labels
```

Grafana then queries that data.

Do not think:

```text
Grafana stores all Prometheus metrics
```

in normal Prometheus data-source architecture.

---

# 128. Grafana

Grafana is mainly:

```text
query
visualization
dashboarding
alerting capabilities
```

Grafana data source is a configured connection to an external data system; a dashboard consists of panels which query and visualize data. [Grafana Labs](https://grafana.com/docs/grafana/latest/datasources/concepts/?utm_source=chatgpt.com)

---

# 129. Grafana data source

Typical:

```text
Grafana
   |
   | Prometheus data source
   |
   v
http://prometheus:9090
```

Prometheus data-source support is built into Grafana. [Grafana Labs](https://grafana.com/docs/grafana/latest/datasources/prometheus/?pg=blog&plcmt=body-txt&utm_source=chatgpt.com)

A panel:

```text
Panel
 |
 +-- Data source: Prometheus
 |
 +-- Query: PromQL
 |
 +-- Visualization
```

---

# 130. Dashboard

Grafana dashboard:

```text
Dashboard
├── CPU panel
├── Memory panel
├── Disk panel
├── Availability panel
└── Application latency panel
```

Official Grafana docs define a dashboard as a set of panels providing related information; each panel transforms/query results into visualization. [Grafana Labs](https://grafana.com/docs/grafana/latest/visualizations/?utm_source=chatgpt.com)

---

# 131. Panel

A panel combines:

```text
query
+
visualization
```

For Prometheus:

```promql
100 * (
  1 -
  avg by(instance)(
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

Visualization:

```text
Time series
```

or:

```text
Gauge
```

or:

```text
Stat
```

Grafana documentation similarly defines panels as data query + visualization, optionally with transformations. [Grafana Labs](https://grafana.com/docs/grafana/latest/visualizations/panels-visualizations/panel-overview/?utm_source=chatgpt.com)

---

# 132. Good dashboard principle

Dashboard should answer operational questions:

```text
Is system up?
Is performance degrading?
Where?
Since when?
Which host/service?
```

Bad dashboard:

```text
40 graphs
no units
no labels
no thresholds
no context
```

---

# 133. Basic dashboard example

For application host:

```text
Availability
  up

CPU
  CPU utilization %

Memory
  used %

Disk
  root free %

Network
  RX/TX rate

Application
  HTTP request/error/latency if available
```

---

# 134. Time range matters

A graph showing:

```text
Last 5 minutes
```

may look very different from:

```text
Last 24 hours
```

Troubleshooting needs correlation around:

```text
incident start
deployment time
traffic spike
```

Always check dashboard time range.

---

# 135. Alerting mental model

Metrics alone don't notify anyone.

```text
Metric
  ↓
Query
  ↓
Condition
  ↓
Evaluation
  ↓
Alert state
  ↓
Notification routing
  ↓
Operator
```

---

# 136. Prometheus alert rule

Basic rule:

```yaml
groups:
  - name: host-alerts

    rules:
      - alert: NodeExporterDown

        expr: up{job="linux-nodes"} == 0

        for: 5m

        labels:
          severity: critical

        annotations:
          summary: "Node exporter is unreachable"
```

Prometheus alert rules use PromQL expressions; optional `for` requires a condition to remain active for a duration before transitioning to firing. [Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/?utm_source=chatgpt.com)

---

# 137. Why `for: 5m`?

Without it:

```text
one missed scrape
↓
alert
```

could create noise.

With:

```yaml
for: 5m
```

Conceptually:

```text
condition first true
     ↓
Pending
     ↓
still true for 5m
     ↓
Firing
```

If recovers before 5m:

```text
Pending → inactive
```

This reduces transient noise.

---

# 138. Alert states

Simplified Prometheus states:

```text
Inactive
Pending
Firing
```

### Inactive

Expression not active.

### Pending

Expression true, but `for` duration not yet completed.

### Firing

Condition persisted long enough.

---

# 139. Host disk alert example

Example:

```yaml
- alert: RootFilesystemLowSpace

  expr: |
    (
      node_filesystem_avail_bytes{mountpoint="/"}
      /
      node_filesystem_size_bytes{mountpoint="/"}
    ) < 0.10

  for: 10m

  labels:
    severity: warning

  annotations:
    summary: "Root filesystem below 10% available"
```

This is a **learning example**, not universal production threshold.

Real threshold should reflect:

```text
growth rate
filesystem size
service behavior
runbook
business criticality
```

---

# 140. CPU alert example

```yaml
- alert: HighCPU

  expr: |
    100 * (
      1 -
      avg by(instance)(
        rate(
          node_cpu_seconds_total{
            mode="idle"
          }[5m]
        )
      )
    ) > 90

  for: 10m
```

Again:

```text
90%
10 minutes
```

are example values, not universal truth.

---

# 141. Prometheus alerting vs notification

Prometheus rule determines:

```text
WHEN alert condition exists
```

Notification delivery commonly involves:

```text
Alertmanager
```

Mental model:

```text
Prometheus
 evaluates rule
     |
     v
Alert firing
     |
     v
Alertmanager
     |
     +--> email
     +--> chat/on-call system
     +--> webhook
```

Alertmanager provides concepts such as routing/grouping/silencing/deduplication.

Detailed Alertmanager administration is supplementary to the explicit syllabus, but understanding where notification fits is necessary for "alert validation".

---

# 142. Grafana alerting distinction

Modern Grafana also has **Grafana-managed alert rules**.

So there are at least two relevant models:

```text
A) Prometheus-managed rule
   stored/evaluated by Prometheus

B) Grafana-managed alert
   stored/evaluated by Grafana
   queries Prometheus
```

Do not call both simply "Grafana alert" without knowing ownership.

Current Grafana documentation recommends Grafana-managed alerts for its richer Grafana workflow, while Prometheus rules from a Prometheus data source can be viewed in Grafana but remain data-source-managed/read-only there. [Grafana Labs](https://grafana.com/docs/grafana/latest/alerting/alerting-rules/?utm_source=chatgpt.com)

For syllabus/exam:

> Understand generic rule → condition → pending/firing → notification flow first.

---

# 143. Grafana-managed alert concept

Example:

```text
Query A:
PromQL CPU utilization

↓
Reduce:
last value

↓
Threshold:
> 90

↓
Pending:
10 minutes

↓
Firing

↓
Contact point / notification policy
```

Grafana documentation describes an alert rule as queries/expressions + trigger condition + evaluation behavior, while notification routing is handled separately through contact points/policies. [Grafana Labs](https://grafana.com/docs/grafana/latest/alerting/alerting-rules/?utm_source=chatgpt.com)

---

# 144. Alert labels

Example:

```yaml
labels:
  severity: critical
  service: payments
  environment: production
```

Labels help:

```text
classification
routing
ownership
silencing
grouping
```

Bad label:

```text
error_message="<random unique stack trace>"
```

can create high-cardinality alert instances.

---

# 145. Alert annotations

Human-readable context:

```yaml
annotations:
  summary: "Payments node unavailable"
  description: "node_exporter has been unreachable for 5 minutes"
```

Useful alert should help operator answer:

```text
What failed?
Where?
How severe?
What should I inspect?
Where is runbook?
```

---

# 146. Alert validation

Creating rule ≠ alert system works.

Complete validation chain:

```text
Metric exists
↓
Prometheus scrapes it
↓
PromQL returns expected value
↓
Rule evaluates
↓
condition becomes true
↓
Pending
↓
Firing
↓
notification generated
↓
receiver receives it
↓
condition clears
↓
alert resolves
```

This is central Day 17 thinking.

---

# 147. What evidence proves alert works?

Potential evidence:

```text
Prometheus target healthy
PromQL result
rule status
pending/firing timestamp
Alertmanager event
Grafana alert state
notification timestamp
resolved state
```

Screenshot alone is often weaker than timestamps + state + configuration evidence.

---

# 148. Common alert validation mistake

Operator creates:

```yaml
expr: up == 0
```

Then sees rule listed.

Concludes:

```text
Alert works.
```

No.

Only proven:

```text
configuration loaded
```

Not proven:

```text
fires
routes
notifies
resolves
```

---

# 149. Monitoring troubleshooting chain

If Grafana shows:

```text
No data
```

do not start by reinstalling Grafana.

Use:

```text
Exporter
  ↓
network
  ↓
Prometheus target
  ↓
Prometheus data
  ↓
PromQL
  ↓
Grafana data source
  ↓
Grafana panel/time range
```

---

# 150. Step 1 — exporter

```bash
curl http://app01:9100/metrics
```

If fails:

```text
node_exporter
network
firewall
port
```

---

# 151. Step 2 — Prometheus scrape

Query:

```promql
up{
  instance="app01:9100"
}
```

If:

```text
0
```

Prometheus cannot successfully scrape.

If:

```text
1
```

scrape succeeds.

---

# 152. Endpoint works locally, Prometheus says down

This scopes problem:

```text
node_exporter local health OK
```

but path:

```text
Prometheus → app01:9100
```

fails.

Suspect:

```text
DNS
routing
firewall
Security Group
wrong target address
Prometheus config
```

---

# 153. Prometheus has data, Grafana doesn't

If Prometheus UI/query returns:

```text
expected time series
```

but Grafana panel says no data:

suspect:

```text
wrong Grafana datasource
wrong PromQL
time range
dashboard variable
panel transform
permissions/network Grafana→Prometheus
```

Not node_exporter first.

---

# 154. Alert not firing

Investigate:

```text
Does metric exist?
Does expression return vector?
Is threshold actually breached?
How long?
What is `for`?
Rule loaded?
Evaluation errors?
```

Example:

```promql
up{job="linux-node"}
```

when actual job label:

```text
linux-nodes
```

may return nothing.

No data is not always equivalent to condition false/true—depends on alerting system semantics.

---

# 155. Firing but no notification

Then rule evaluation is probably not primary problem.

Scope toward:

```text
Alertmanager/contact point
notification policy
routing labels
receiver credentials/config
network
silence
inhibition
```

This illustrates troubleshooting by layer.

---

# 156. Metrics cardinality — Supplementary

Each unique label combination creates distinct time series.

Bad metric design:

```text
request{
  user_id="12345678",
  request_id="a1b2c3..."
}
```

Potentially millions of series.

This causes:

```text
memory/storage/query cost
```

Cardinality awareness is important operationally even though syllabus doesn't state it explicitly.

---

# 157. Monitoring security

Metrics endpoints can reveal:

```text
hostname
filesystem names
mount paths
network interfaces
kernel details
application names
internal topology
```

Therefore:

```text
node_exporter :9100
Prometheus :9090
Grafana
```

should not simply be exposed publicly because "there is no password in metrics".

Apply:

```text
network restriction
least privilege
authentication where appropriate
TLS where required
security groups/firewalls
```

---

# 158. Grafana credentials

Grafana should not use unnecessarily privileged data-source credentials.

Principle:

```text
Grafana needs read/query access
≠
Grafana needs database superuser
```

If connecting Grafana directly to PostgreSQL:

```text
read-only dedicated monitoring role
```

is preferable where feasible.

For Prometheus datasource, Grafana only needs the access required to query Prometheus endpoint.

---

# 159. Alert fatigue

Bad monitoring:

```text
100 alerts/day
95 irrelevant
```

Operators learn:

```text
ignore alerts
```

Good alert should generally correspond to:

```text
actionable situation
owner
severity
clear context
```

Avoid:

```text
Everything above arbitrary threshold = critical
```

---

# 160. Golden operational relationship

Dashboards and alerts serve different purposes:

```text
Dashboard
→ investigate / observe

Alert
→ draw attention / initiate response
```

Not every dashboard metric deserves an alert.

Example:

```text
CPU utilization
```

useful dashboard metric.

But transient 95% CPU for 5 seconds may not deserve paging.

---

# PHẦN IV — INTEGRATED DATABASE + MONITORING THINKING

## 161. End-to-end architecture

By the end of Module 2, mental model becomes:

```text
                         USER
                          |
                          v
                    Java / Tomcat
                          |
                         JDBC
                          |
                          v
                    PostgreSQL
                      :5432
                   /     |      \
              roles   schemas   data
                 |
        sessions / locks / SQL
                 |
                 v
            DB diagnostics


Linux Host
   |
   +--> CPU
   +--> Memory
   +--> Disk
   +--> Network
   |
node_exporter :9100
   |
   v
Prometheus
   |
   +--> PromQL
   +--> Alert rules
   |
   +--> Alertmanager
   |
   v
Grafana
```

---

# 162. Scenario — Java reports DB timeout

Don't immediately restart PostgreSQL.

Reasoning:

```text
Application DB timeout
↓
Can DB hostname resolve?
↓
Can host reach :5432?
↓
Is PostgreSQL listening?
↓
Is server accepting connections?
↓
Authentication?
↓
Connection capacity?
↓
Sessions blocked?
↓
Host/db resource pressure?
```

Commands/evidence:

```bash
getent hosts db.internal
```

```bash
pg_isready -h db.internal -p 5432
```

```bash
ss -lntp | grep ':5432'
```

SQL:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

plus logs/monitoring.

---

# 163. Scenario — application slow but CPU low

Naive assumption:

```text
CPU low
→ application should be fast
```

Wrong.

Could be:

```text
DB lock wait
network timeout
disk IO
downstream service
connection pool starvation
```

Check:

```sql
SELECT
    pid,
    wait_event_type,
    wait_event,
    pg_blocking_pids(pid)
FROM pg_stat_activity
WHERE state = 'active';
```

Host CPU alone doesn't tell full story.

---

# 164. Scenario — PostgreSQL connection limit

Symptom:

```text
remaining connection slots ...
```

Reason:

```text
session count reached capacity
```

Collect:

```sql
SHOW max_connections;
```

```sql
SELECT count(*)
FROM pg_stat_activity;
```

```sql
SELECT
    application_name,
    usename,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY application_name, usename, state
ORDER BY count(*) DESC;
```

Possible root causes:

```text
pool too large
connection leak
unexpected workload
many idle sessions
DB configuration mismatch
```

Do not automatically:

```text
double max_connections
```

---

# 165. Scenario — disk alert

Prometheus alert:

```text
filesystem < 10% available
```

Validate Linux:

```bash
df -h
```

Then:

```text
which filesystem?
what growing?
database data?
WAL?
logs?
backup files?
application logs?
```

DB:

```sql
SELECT
    pg_size_pretty(
        pg_database_size(current_database())
    );
```

Potential escalation if database/WAL growth requires DBA handling.

---

# 166. Scenario — DB query slow after deployment

Recent change:

```text
new application version
```

Possible:

```text
new SQL
different query parameters
missing index assumption
new transaction behavior
higher connection pool
schema migration
```

Correlate:

```text
release timestamp
↓
application logs
↓
PostgreSQL activity/log
↓
Grafana CPU/disk/connections
↓
EXPLAIN evidence
```

This is exactly the integrated reasoning Day 20 will expect.

---

# 167. Scenario — monitoring says host down but application works

Prometheus:

```promql
up{job="linux-nodes",instance="app01:9100"} == 0
```

But users can use application.

Likely:

```text
node_exporter/monitoring path failed
```

not necessarily host down.

This is why alert naming matters.

Better:

```text
NodeExporterUnreachable
```

than:

```text
ServerDown
```

unless signal truly proves entire server unavailable.

---

# 168. Scenario — node exporter up but disk already full

Possible if exporter still runs and reports:

```promql
node_filesystem_avail_bytes == near 0
```

`up=1` only says scrape works.

Again:

```text
availability of exporter
≠
healthy host capacity
```

---

# 169. L1 evidence package

When escalating DB problem, useful evidence:

```text
Incident timestamp
Affected service/users
DB host/database
Relevant error message
Connectivity test
pg_isready result
session counts
blocking evidence
capacity evidence
relevant DB log range
relevant app log range
recent changes
actions already taken
current impact
```

Do not send:

```text
"DB broken pls check"
```

---

# 170. L1-safe vs escalation

## Usually L1-safe, if authorized

```text
check service status
check port
run health query
inspect pg_stat_activity
inspect lock/blocking evidence
inspect disk
inspect logs
run known read-only diagnostics
validate metrics/alerts
follow approved restart/runbook
```

## Escalation territory

```text
schema changes
index design
deep query tuning
VACUUM FULL
database corruption
WAL manipulation
replication recovery
production role redesign
mass session termination
capacity architecture changes
major PostgreSQL parameter tuning
PITR
```

---

# 171. Anti-patterns Module 2

Bạn cần nhận diện ngay:

1. Cho application `SUPERUSER` để fix permission.
2. Dùng `trust` cho toàn mạng để fix authentication.
3. Nhầm `pg_hba.conf` với SQL table privileges.
4. Mở `0.0.0.0/0` unnecessarily.
5. Tăng `max_connections` thay vì điều tra connection leak/pool.
6. Kill PostgreSQL backend bằng `kill -9` như first action.
7. Xóa `pg_wal` khi disk full.
8. Chạy `EXPLAIN ANALYZE DELETE/UPDATE` production vì nghĩ chỉ "explain".
9. Thấy `Seq Scan` rồi tạo index ngay.
10. Có dump nhưng chưa từng restore test.
11. Để backup world-readable.
12. Coi `up == 1` là application business healthy.
13. Bật mọi node_exporter collector.
14. Alert mọi metric bằng arbitrary threshold.
15. Có alert rule nhưng chưa test notification path.
16. Dashboard "No data" rồi reinstall Grafana trước khi kiểm tra Prometheus.
17. Expose ports `5432`, `9090`, `9100` publicly mà không có need/security controls.

---

# PHẦN V — NHỮNG COMMAND / SQL / PROMQL PHẢI NHẬN DIỆN

## PostgreSQL connectivity

```bash
psql -h <host> -p 5432 -U <role> -d <database>
```

```bash
pg_isready -h <host> -p 5432
```

---

## Identity

```sql
SELECT current_user;
```

```sql
SELECT current_database();
```

```text
\conninfo
\du
\l
\dn
\dt
```

---

## Configuration

```sql
SHOW listen_addresses;
SHOW port;
SHOW max_connections;
SHOW transaction_read_only;
```

---

## Role

```sql
CREATE ROLE app_runtime LOGIN;
```

```sql
GRANT CONNECT
ON DATABASE appdb
TO app_runtime;
```

```sql
GRANT USAGE
ON SCHEMA app
TO app_runtime;
```

---

## Sessions

```sql
SELECT *
FROM pg_stat_activity;
```

---

## Blocking

```sql
SELECT
    pid,
    pg_blocking_pids(pid)
FROM pg_stat_activity;
```

---

## Capacity

```bash
df -h
df -i
```

```sql
SELECT pg_size_pretty(
    pg_database_size('appdb')
);
```

```sql
SELECT pg_size_pretty(
    pg_total_relation_size('app.orders')
);
```

---

## Planner

```sql
EXPLAIN
SELECT ...;
```

Carefully/approved:

```sql
EXPLAIN ANALYZE
SELECT ...;
```

---

## Backup

```bash
pg_dump -d appdb -f appdb.sql
```

```bash
pg_dump -Fc -d appdb -f appdb.dump
```

```bash
psql -d appdb_restore -f appdb.sql
```

```bash
pg_restore -d appdb_restore appdb.dump
```

---

## node_exporter

```bash
curl http://127.0.0.1:9100/metrics
```

---

## PromQL

```promql
up
```

```promql
up{job="linux-nodes"}
```

```promql
rate(node_cpu_seconds_total[5m])
```

```promql
100 *
node_memory_MemAvailable_bytes
/
node_memory_MemTotal_bytes
```

---

# PHẦN VI — EXAM FOCUS

Sau Module 2, bạn phải trả lời được không nhìn tài liệu:

### PostgreSQL architecture

1. PostgreSQL cluster, database và schema khác nhau thế nào?
2. PostgreSQL role khác Linux user thế nào?
3. PostgreSQL "user" liên quan role như thế nào?
4. Vì sao runtime application không nên là superuser?
5. Package deployment khác containerized PostgreSQL ở đâu?
6. Vì sao container database cần persistent storage?

### Connectivity/security

7. `listen_addresses` làm gì?
8. `pg_hba.conf` làm gì?
9. Tại sao `pg_hba.conf` không thay thế `GRANT`?
10. Vì sao rule ordering trong `pg_hba.conf` quan trọng?
11. `local`, `host`, `hostssl` khác nhau thế nào?
12. `scram-sha-256` dùng để làm gì?
13. Vì sao MD5 không còn là lựa chọn nên học làm recommendation?
14. Authentication khác authorization như thế nào?
15. Timeout, connection refused, HBA rejection, password failure và permission denied nằm ở layer nào?

### Backup/recovery

16. `pg_dump` làm gì?
17. Plain dump khác custom format thế nào?
18. Khi nào dùng `psql` restore và khi nào dùng `pg_restore`?
19. `pg_dump` khác `pg_dumpall` thế nào?
20. Tại sao backup chưa restore-test chưa đủ mạnh để gọi là proven recovery?

### Diagnostics

21. `pg_stat_activity` dùng để làm gì?
22. `active`, `idle`, `idle in transaction` nghĩa gì?
23. Lock khác blocking và deadlock thế nào?
24. `pg_blocking_pids()` hữu ích ra sao?
25. `pg_cancel_backend` khác `pg_terminate_backend` conceptually thế nào?
26. Vì sao không `kill -9` PostgreSQL backend tùy tiện?
27. Capacity check nên kiểm tra những gì?

### EXPLAIN

28. `EXPLAIN` cho estimated hay actual execution?
29. `EXPLAIN ANALYZE` khác gì?
30. Vì sao `EXPLAIN ANALYZE UPDATE/DELETE` nguy hiểm?
31. Seq Scan có phải luôn xấu không?
32. Statistics có vai trò gì đối với planner?

### Monitoring

33. Metric và time series là gì?
34. Label có vai trò gì?
35. Gauge khác counter thế nào?
36. Vì sao counter thường dùng `rate()`?
37. node_exporter làm gì?
38. Prometheus sử dụng pull/scrape model như thế nào?
39. `scrape_config`, `job`, `instance` là gì?
40. `up == 1` thực sự chứng minh điều gì?
41. Host/service/application signals khác nhau ra sao?
42. Grafana datasource, dashboard và panel là gì?
43. Grafana có lưu Prometheus metrics thay Prometheus không?

### Alerting

44. Alert rule gồm metric/query/condition/evaluation như thế nào?
45. `for:` có tác dụng gì?
46. Pending khác Firing thế nào?
47. Prometheus rule và Grafana-managed alert khác nhau thế nào?
48. Alertmanager nằm ở đâu trong notification flow?
49. Làm sao chứng minh alert thực sự hoạt động?
50. Nếu alert Firing nhưng không nhận notification, scope troubleshooting chuyển sang layer nào?

---

# 172. Master troubleshooting map

Đây là diagram quan trọng nhất của Module 2:

```text
APPLICATION PROBLEM
        |
        v
Can app resolve DB hostname?
        |
        v
Can network reach :5432?
        |
        v
Is PostgreSQL listening?
        |
        v
Does pg_hba.conf match?
        |
        v
Does authentication work?
        |
        v
Does role have permission?
        |
        v
Enough DB connections?
        |
        v
Blocked session / lock?
        |
        v
Slow SQL / bad plan?
        |
        v
Disk / CPU / memory / IO?
        |
        v
Logs + metrics evidence
        |
        v
L1 remediation
or
escalation
```

Monitoring side:

```text
HOST
 |
node_exporter
 |
:9100 /metrics
 |
 v
Prometheus scrape
 |
 +---- up
 |
 +---- node_* metrics
 |
 +---- PromQL
 |
 +---- alert rules
 |        |
 |        v
 |    Alertmanager
 |
 v
Grafana
 |
dashboard/panels
```

Và toàn bộ chain Day 12–17 bây giờ trở thành:

```text
Client
  ↓
Reverse Proxy / TLS
  ↓
Tomcat
  ↓
Java Application
  ↓
JDBC
  ↓
PostgreSQL
  ↓
Role / HBA / privilege
  ↓
SQL / sessions / locks / planner
  ↓
Linux CPU / memory / disk / network
  ↓
node_exporter
  ↓
Prometheus
  ↓
PromQL / Alerts
  ↓
Grafana
```

Đó chính là mental model bạn cần mang sang **Module 3 / Day 18–21**. Module 3 sẽ không còn chủ yếu dạy thêm một technology riêng lẻ nữa; nó sẽ dùng toàn bộ chain này để học **batch/release/change control → integrated architecture → failure injection/troubleshooting → rollback/escalation/handover → final readiness**.
