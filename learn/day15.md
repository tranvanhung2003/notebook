# Day 15 — PostgreSQL Foundations: Provisioning, Access Control, Connectivity, Health Check, Backup & Restore

## 1. Syllabus Alignment

Theo **MASTER SYLLABUS**, Day 15 thuộc phần **Database & Monitoring Foundations** và yêu cầu chính xác:

> **Concept/Lecture:** PostgreSQL installation/container, users/roles, database/schema, connectivity, `pg_hba.conf` awareness, health checks, và backup/restore.  
> **Assignment/Lab:** Provision PostgreSQL, tạo least-privilege access, kết nối application, backup, restore và nộp evidence. 00_MASTER_SYLLABUS_Java_Fresher…

Do đó, Day 15 hôm nay có 7 năng lực bắt buộc:

| Năng lực             | Mức cần đạt                                                     |
| -------------------- | --------------------------------------------------------------- |
| Provision PostgreSQL | Hiểu native installation và container, tự dựng được DB          |
| Roles/users          | Tạo account đúng mục đích, không dùng superuser cho application |
| Database/schema      | Hiểu database → schema → table/object                           |
| Connectivity         | Phân tích được TCP/port/authentication/privilege path           |
| `pg_hba.conf`        | Đọc, giải thích, sửa an toàn và reload                          |
| Health checks        | Chứng minh DB đang chạy, reachable và usable                    |
| Backup/restore       | Tạo logical backup, restore sang DB sạch và xác minh dữ liệu    |

**Day 16 mới đi sâu SQL/log diagnostics, sessions/locks, capacity checks, `EXPLAIN` và L1 escalation boundary**, nên hôm nay tôi chỉ dùng một phần `pg_stat_activity` để health-check/troubleshooting, không thay thế nội dung Day 16. 00_MASTER_SYLLABUS_Java_Fresher…

### Phiên bản sử dụng

Tại thời điểm hiện tại, **PostgreSQL 18 là nhánh stable hiện hành**; PostgreSQL 19 vẫn đang ở Beta. Bản stable update được PostgreSQL Project công bố gần nhất là **18.6 ngày 13/08/2026**. Vì vậy lab sẽ pin `postgres:18.6`, không dùng `latest` hoặc beta. [PostgreSQL](https://www.postgresql.org/about/news/postgresql-186-1711-1615-1519-1424-and-19-beta-3-released-3365/?utm_source=chatgpt.com)

---

# 2. Objectives và prerequisites

Sau Day 15, bạn cần tự làm được chuỗi sau mà không phụ thuộc hoàn toàn vào hướng dẫn:

**Provision → inspect → create role → create database/schema → grant least privilege → test connectivity → configure application → validate → backup → destroy/corrupt lab data → restore → prove recovery → document evidence.**

Kiến thức Day 2–14 sẽ được sử dụng trực tiếp:

| Kiến thức cũ             | Day 15 sử dụng vào đâu            |
| ------------------------ | --------------------------------- |
| Linux files/permissions  | PostgreSQL config, backup files   |
| users/groups/sudo        | PostgreSQL OS service account     |
| systemd                  | Native PostgreSQL service         |
| networking/ports         | TCP 5432                          |
| `ss`, `curl`, DNS        | Connectivity diagnosis            |
| logs                     | PostgreSQL/container logs         |
| AWS SG                   | Nếu DB/application nằm khác EC2   |
| Java externalized config | JDBC URL/user/password            |
| secrets                  | DB password                       |
| TLS                      | Có thể bảo vệ DB connection       |
| deployment/rollback      | Application + database validation |

---

# 3. Mental Model quan trọng nhất của Day 15

Đừng xem PostgreSQL chỉ là:

> “Một chương trình nghe ở port 5432.”

Mental model chính xác hơn là:

```text
Linux / Container Host
│
├── PostgreSQL server process
│     │
│     ├── PostgreSQL cluster / instance
│     │      │
│     │      ├── Roles        ← cluster-wide
│     │      │
│     │      ├── Database A
│     │      │    ├── Schema public
│     │      │    └── Schema app
│     │      │          ├── tables
│     │      │          ├── sequences
│     │      │          ├── views
│     │      │          └── functions
│     │      │
│     │      └── Database B
│     │
│     ├── postgresql.conf
│     └── pg_hba.conf
│
└── persistent storage / PGDATA
```

Một Java application muốn query được database phải đi qua nhiều lớp:

```text
Java application
      │
      │ JDBC
      ▼
hostname / DNS
      │
      ▼
TCP/IP
      │
      ▼
PostgreSQL host:5432
      │
      ├── Is server running?
      ├── Is it listening on this interface?
      ├── OS/AWS firewall allows traffic?
      │
      ▼
pg_hba.conf
      │
      ├── Is client IP allowed?
      ├── Is database allowed?
      ├── Is PostgreSQL role allowed?
      ├── Which authentication method?
      │
      ▼
Authentication
      │
      ▼
Database CONNECT privilege
      │
      ▼
Schema USAGE privilege
      │
      ▼
Object privilege
SELECT / INSERT / UPDATE / DELETE / ...
      │
      ▼
SQL succeeds
```

Đây cũng chính là troubleshooting model bạn cần nhớ.

Một lỗi:

```text
Connection refused
```

khác hoàn toàn:

```text
no pg_hba.conf entry
```

và cũng khác:

```text
password authentication failed
```

hay:

```text
permission denied for table customer
```

Bốn lỗi đó xảy ra ở **bốn layer khác nhau**.

---

# 4. PostgreSQL terminology

## 4.1 PostgreSQL server / instance / cluster

Trong PostgreSQL, một **database cluster** không có nghĩa là HA cluster nhiều server.

Nó là một collection các database được quản lý bởi **một PostgreSQL server instance**, sử dụng một data directory.

Ví dụ:

```text
PostgreSQL instance
├── postgres
├── template0
├── template1
├── fresherdb
└── monitoringdb
```

`postgres` là database mặc định thường dùng để thực hiện administrative operations. Khi `initdb` tạo cluster mới, PostgreSQL cũng tạo `template0` và `template1`; database mới thường được clone từ `template1`. [PostgreSQL](https://www.postgresql.org/docs/18/manage-ag-createdb.html?utm_source=chatgpt.com)

---

# 5. PostgreSQL installation: native vs container

## 5.1 Native installation

Mô hình:

```text
Linux
  └── PostgreSQL package
        ├── binary
        ├── service
        ├── config
        ├── PGDATA
        └── logs
```

Ưu điểm là tích hợp tốt với OS:

```bash
systemctl status postgresql
journalctl ...
package updates
filesystem permissions
native backups
```

PostgreSQL Project khuyến nghị sử dụng package manager của Linux khi package phù hợp vì nó tích hợp với patching và OS management. [PostgreSQL](https://www.postgresql.org/download/linux/?utm_source=chatgpt.com)

Ví dụ Debian/Ubuntu:

```bash
sudo apt update
sudo apt install postgresql
```

Sau đó:

```bash
systemctl status postgresql
```

Nhưng distro thường snapshot một major version cụ thể. PostgreSQL Project cũng cung cấp PGDG repository nếu bạn cần một major version khác. [PostgreSQL](https://www.postgresql.org/download/linux/debian/?utm_source=chatgpt.com)

### Không giả định config path

Một fresher rất hay ghi nhớ:

```text
/etc/postgresql/18/main/postgresql.conf
```

rồi cho rằng hệ nào cũng vậy.

Không đúng.

Tốt hơn hãy hỏi PostgreSQL:

```sql
SHOW config_file;
SHOW hba_file;
SHOW data_directory;
```

Đây là cách vận hành tốt hơn việc đoán path.

---

# 6. Containerized PostgreSQL

Kiến trúc:

```text
Docker host
│
├── PostgreSQL container
│      └── postgres process
│
└── Docker volume
       └── database persistent data
```

Điểm sống còn:

> **Container không phải nơi lưu dữ liệu lâu dài. Persistent volume mới là nơi dữ liệu phải sống.**

Nếu container mất nhưng volume còn:

```text
container deleted
        │
        ▼
new container
        │
        ▼
same volume
        │
        ▼
data survives
```

Nếu không hiểu volume, bạn có thể recreate container và tưởng rằng PostgreSQL “làm mất database”.

Docker Official PostgreSQL image sử dụng `POSTGRES_PASSWORD`, `POSTGRES_USER`, `POSTGRES_DB` trong initialization. Những giá trị initialization này chỉ có tác dụng khi data directory còn **empty**; thay chúng sau khi database đã initialize không tự đổi user/database/password đang tồn tại. [Docker Hub](https://hub.docker.com/_/postgres?utm_source=chatgpt.com)

### PostgreSQL 18 container nuance — Supplementary / Beyond explicit syllabus

Docker Official Image có thay đổi layout volume với PostgreSQL 18+.

Đối với PostgreSQL 18+, `PGDATA` mặc định mang major version trong path, và official image khuyến nghị mount volume ở:

```text
/var/lib/postgresql
```

Trong khi PostgreSQL 17 trở xuống thường mount:

```text
/var/lib/postgresql/data
```

Đây là chi tiết rất quan trọng nếu sau này bạn làm container upgrade. [Docker Hub](https://hub.docker.com/_/postgres?utm_source=chatgpt.com)

---

# 7. Role, user và least privilege

Đây là một phần dễ nhầm nhất.

## 7.1 PostgreSQL không thật sự có hai object type riêng biệt “user” và “role”

PostgreSQL quản lý authentication và authorization bằng **roles**.

Role có thể:

```text
LOGIN             → giống user
NOLOGIN           → giống group/ownership role
```

Một role cũng có thể sở hữu database/table/function và có thể được grant cho role khác. [PostgreSQL](https://www.postgresql.org/docs/18/user-manag.html?utm_source=chatgpt.com)

Ví dụ:

```sql
CREATE ROLE app_runtime LOGIN;
```

Role này có thể login.

```sql
CREATE ROLE app_owner NOLOGIN;
```

Role này không login nhưng có thể sở hữu objects.

`CREATE USER` thực chất là alias của `CREATE ROLE`; khác biệt chính là `CREATE USER` mặc định có `LOGIN`. [PostgreSQL](https://www.postgresql.org/docs/current/sql-createuser.html?utm_source=chatgpt.com)

---

# 8. Vì sao application không được dùng `postgres`

Sai:

```text
Application
    │
    └── login as postgres
```

`postgres` thường là superuser.

Nếu application bị SQL injection hoặc credential bị lộ, attacker có thể có quyền cực lớn.

Mô hình tốt hơn:

```text
postgres/admin
     │
     │ administrative only
     ▼
app_owner
NOLOGIN
     │
     ├── owns DB/schema/tables
     │
     ▼
app_runtime
LOGIN
     │
     ├── CONNECT
     ├── USAGE schema
     ├── SELECT
     ├── INSERT
     ├── UPDATE
     └── DELETE
```

Runtime account không cần:

```text
SUPERUSER
CREATEDB
CREATEROLE
REPLICATION
BYPASSRLS
```

Least privilege nghĩa là:

> Chỉ cấp những quyền application cần để chạy, không cấp quyền “phòng trường hợp sau này cần”.

PostgreSQL object privileges bao gồm các quyền như `SELECT`, `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `CONNECT`, `USAGE`, `EXECUTE`... và quyền phù hợp tùy loại object. [PostgreSQL](https://www.postgresql.org/docs/current/ddl-priv.html?utm_source=chatgpt.com)

---

# 9. Database và Schema khác nhau thế nào?

Giả sử:

```text
PostgreSQL instance
    │
    └── database: fresherdb
            │
            ├── schema: public
            │
            └── schema: app
                    │
                    ├── app.customer
                    ├── app.orders
                    └── app.audit_log
```

Database là boundary cấp cao.

Schema là **namespace bên trong một database**.

Hai schema khác nhau có thể có table cùng tên:

```text
app.orders
archive.orders
```

Đây là hai object khác nhau.

---

# 10. `search_path`

Nếu SQL viết:

```sql
SELECT * FROM customer;
```

PostgreSQL phải quyết định:

```text
customer = schema nào?
```

Nó sử dụng:

```sql
SHOW search_path;
```

Một vấn đề security rất quan trọng là: nếu một schema nằm trong `search_path` và untrusted user có quyền `CREATE` object trong schema đó, object lookup có thể bị lợi dụng.

PostgreSQL documentation nhấn mạnh rằng quyền `CREATE` trên schema trong `search_path` tạo trust relationship và khuyến nghị thiết kế schema privilege cẩn thận. PostgreSQL 15+ đã thay đổi default theo hướng an toàn hơn đối với `public`. [PostgreSQL](https://www.postgresql.org/docs/18/ddl-schemas.html?utm_source=chatgpt.com)

Lab của chúng ta sẽ không để application tạo object trong `public`.

---

# 11. PostgreSQL connectivity

Default PostgreSQL TCP port:

```text
5432
```

Nhưng nhớ:

> Port không chứng minh database hoạt động đầy đủ.

Một TCP socket mở chỉ chứng minh một process đang listen.

PostgreSQL mặc định có `listen_addresses = localhost`, tức TCP connection từ remote machine sẽ không tự động được nhận. `listen_addresses` xác định interface nào PostgreSQL bind vào; port mặc định là 5432. [PostgreSQL](https://www.postgresql.org/docs/current/runtime-config-connection.html)

Kiểm tra:

```sql
SHOW listen_addresses;
SHOW port;
```

Linux:

```bash
ss -lntp
```

Ví dụ:

```text
LISTEN 0 244 127.0.0.1:5432 ...
```

Điều này nghĩa là:

```text
localhost → possible
remote IP → impossible
```

ngay cả khi `pg_hba.conf` cho phép remote client.

---

# 12. `pg_hba.conf` — Host-Based Authentication

Đây là file quan trọng nhất Day 15.

HBA =:

```text
Host-Based Authentication
```

Nó quyết định:

> Client nào → database nào → PostgreSQL role nào → từ network nào → authentication method nào.

Một record thường có dạng:

```text
TYPE DATABASE USER ADDRESS METHOD
```

Ví dụ:

```text
host fresherdb app_runtime 10.0.1.25/32 scram-sha-256
```

Nghĩa là:

```text
connection type : TCP/IP
database        : fresherdb
role            : app_runtime
source          : 10.0.1.25 only
authentication  : SCRAM-SHA-256
```

PostgreSQL xét rule **từ trên xuống dưới**.

Điểm đặc biệt cực kỳ quan trọng:

> **Rule đầu tiên match được sử dụng. Không có fallback sang rule sau nếu authentication thất bại.**

Nếu không có rule nào match → connection denied. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

Ví dụ:

```text
host fresherdb app_runtime 10.0.1.25/32 reject
host fresherdb app_runtime 10.0.0.0/8  scram-sha-256
```

Client:

```text
10.0.1.25
```

sẽ bị `reject`.

PostgreSQL sẽ **không thử rule thứ hai**.

---

# 13. Authentication methods cần biết

### `scram-sha-256`

Password authentication hiện đại:

```text
scram-sha-256
```

Nên ưu tiên cho password-based authentication.

### `trust`

```text
trust
```

nghĩa là:

> Nếu connection match rule, PostgreSQL không yêu cầu password.

Điều này rất nguy hiểm nếu dùng sai phạm vi.

PostgreSQL documentation nói rõ `trust` cho phép bất kỳ ai có thể kết nối theo rule đó đăng nhập dưới PostgreSQL username tương ứng mà không cần password. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

### `peer`

Thường dùng local Unix socket:

```text
Linux account name
        ↓
PostgreSQL role
```

### `md5`

MD5 password support đã **deprecated** và PostgreSQL documentation nói sẽ bị loại bỏ trong future release. Không nên thiết kế hệ thống mới dựa trên MD5. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

---

# 14. Reload `pg_hba.conf`

Đây là điểm thi/thực hành rất hay hỏi.

Sửa file:

```text
pg_hba.conf
```

không nhất thiết restart PostgreSQL.

Có thể reload:

```sql
SELECT pg_reload_conf();
```

hoặc các công cụ service/`pg_ctl`.

PostgreSQL đọc `pg_hba.conf` khi startup và khi nhận SIGHUP/reload. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

Quan trọng hơn, có thể kiểm tra parser bằng:

```sql
SELECT *
FROM pg_hba_file_rules
WHERE error IS NOT NULL;
```

Nếu query không trả row:

```text
Expected invariant:
không có syntax error được PostgreSQL phát hiện trong HBA rules.
```

`pg_hba_file_rules` được PostgreSQL cung cấp chính để kiểm tra/diagnose HBA configuration. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)

---

# 15. Quy trình an toàn khi chỉnh `pg_hba.conf`

Đây là operational habit rất quan trọng.

Không làm kiểu:

```text
edit → restart → đóng SSH/session → cầu mong
```

Thay vào đó:

```text
inspect existing state
        ↓
identify exact HBA path
        ↓
backup file
        ↓
keep current admin session open
        ↓
make smallest possible change
        ↓
validate parser
        ↓
reload
        ↓
test using NEW connection
        ↓
verify logs
        ↓
only then close old session
```

### Cảnh báo

Sai HBA có thể làm application mất DB access hoặc khóa chính administrator ra khỏi TCP access.

Trong production, cần:

```text
approved change
backup
blast-radius control
rollback
validation
evidence
```

---

# 16. Health check đúng nghĩa

Một health check database nên kiểm tra nhiều layer.

| Layer                 | Câu hỏi                                | Công cụ                  |
| --------------------- | -------------------------------------- | ------------------------ |
| Process               | PostgreSQL có chạy không?              | `systemctl`, `docker ps` |
| TCP socket            | Có listen không?                       | `ss -lntp`               |
| Protocol readiness    | Có accept PostgreSQL connection không? | `pg_isready`             |
| Authentication        | User login được không?                 | `psql`                   |
| Correct target        | Đúng DB/server/user không?             | SQL                      |
| SQL functionality     | Query thực thi được không?             | `SELECT 1`               |
| Application privilege | Runtime role đọc/ghi đúng quyền không? | DML                      |
| Application           | Java app thật sự dùng DB được không?   | endpoint/log             |
| Storage/capacity      | Có đủ disk/capacity không?             | Linux + SQL              |

---

# 17. `pg_isready`

Command:

```bash
pg_isready -h HOST -p PORT
```

Ví dụ:

```bash
pg_isready -h 127.0.0.1 -p 5432
```

Return code:

```text
0 = accepting connections
1 = rejecting connections
2 = no response
3 = no attempt made
```

PostgreSQL định nghĩa chính thức các exit status này, nên `pg_isready` rất phù hợp cho script/monitoring. [PostgreSQL](https://www.postgresql.org/docs/18/app-pg-isready.html?utm_source=chatgpt.com)

Ví dụ:

```bash
pg_isready -h 127.0.0.1 -p 5432
echo $?
```

**Example output:**

```text
127.0.0.1:5432 - accepting connections
0
```

**Expected invariant:**

```text
exit code = 0
```

Không được học thuộc nguyên văn output vì path/socket/host có thể khác.

---

# 18. SQL health check

Sau khi TCP và authentication thành công:

```sql
SELECT 1;
```

Nhưng tốt hơn:

```sql
SELECT
    current_database(),
    current_user,
    inet_server_addr(),
    inet_server_port();
```

Vì nó chứng minh:

```text
đúng server
đúng database
đúng identity
đúng port
```

Ngoài ra:

```sql
SHOW server_version;
SHOW transaction_read_only;
```

`transaction_read_only = off` thường là expected state nếu đây là primary writable database của application.

---

# 19. Sessions — Day 15 awareness

Một query cơ bản:

```sql
SELECT
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    state
FROM pg_stat_activity;
```

`pg_stat_activity` cung cấp một row cho mỗi server process/backend cùng thông tin như database, user, client address, application name và session state. [PostgreSQL](https://www.postgresql.org/docs/current/monitoring-stats.html)

Day 16 chúng ta sẽ dùng view này sâu hơn cho:

```text
sessions
locks
long-running query
idle in transaction
slow query troubleshooting
```

---

# 20. Backup không đồng nghĩa với copy directory

Day 15 tập trung **logical backup**:

```text
PostgreSQL DB
     │
     ▼
pg_dump
     │
     ▼
logical backup file
```

`pg_dump` tạo consistent export của **một database** và không chặn readers/writers thông thường trong thời gian dump. [PostgreSQL](https://www.postgresql.org/docs/current/backup-dump.html)

Format quan trọng:

```text
plain   → SQL script
custom  → pg_restore
directory → pg_restore, parallel support
```

Lab hôm nay dùng:

```bash
pg_dump -Fc
```

`-F c`:

```text
custom format
```

Vì:

```text
compressed
selective restore
pg_restore support
flexible
```

Official docs mô tả custom/directory là các format linh hoạt nhất; custom format được compressed mặc định. [PostgreSQL](https://www.postgresql.org/docs/current/app-pgdump.html)

---

# 21. `pg_dump` không backup roles

Điểm thi rất quan trọng.

```text
pg_dump fresherdb
```

backup:

```text
schema
tables
data
indexes
constraints
privileges within DB
...
```

nhưng không backup cluster-global objects như:

```text
roles
tablespaces
```

Để backup global objects:

```bash
pg_dumpall --globals-only
```

PostgreSQL documentation nói rõ `pg_dump` chỉ dump một database; `pg_dumpall` hoặc `--globals-only` dùng cho roles/tablespaces và các global objects. [PostgreSQL](https://www.postgresql.org/docs/current/backup-dump.html)

Do đó một bộ logical backup đầy đủ hơn có thể là:

```text
fresherdb.dump
globals.sql
```

---

# 22. Backup chỉ có giá trị nếu restore được

Đây là mindset Operations:

```text
Backup completed successfully
```

chưa đủ để nói:

```text
Recovery works.
```

Phải thực hiện:

```text
backup
  ↓
restore into isolated/clean target
  ↓
verify objects
  ↓
verify row counts/data
  ↓
verify application
```

Mục tiêu cuối cùng không phải:

> “Tôi có file `.dump`.”

Mà là:

> “Tôi đã chứng minh file đó có thể khôi phục service/data đúng expected state.”

---

# 23. GUIDED LAB — Day 15

Lab chính sẽ dùng:

```text
Linux host
Docker
PostgreSQL 18.6
Java app từ Day 12–14 nếu còn environment
```

Nếu chưa có Java app đang chạy, bạn vẫn hoàn tất toàn bộ PostgreSQL lab bằng `psql`, nhưng **phần application connection chỉ được coi là hoàn chỉnh khi application thực tế kết nối và chạy DB operation thành công**.

---

## Phase A — Pre-change inspection

Trước khi tạo resource:

```bash
docker --version
docker ps
ss -lntp | grep ':5432' || true
df -h
```

Mục tiêu:

```text
Docker usable
port 5432 không bị conflict
host còn disk
```

Không được bỏ qua disk check khi provision database.

---

# 24. Provision PostgreSQL container

Tạo workspace:

```bash
mkdir -p ~/day15-postgres/{backup,evidence}
cd ~/day15-postgres
```

Tạo network:

```bash
docker network create day15-net
```

Tạo volume:

```bash
docker volume create day15-pgdata
```

Kiểm tra:

```bash
docker volume inspect day15-pgdata
```

### Password

Không dùng password ví dụ của tôi cho account thật.

Tạo password **trên máy của bạn**, không gửi password đó cho tôi:

```bash
openssl rand -base64 24
```

Sau đó chạy PostgreSQL:

```bash
docker run -d \
  --name day15-postgres \
  --network day15-net \
  -p 127.0.0.1:5432:5432 \
  -e POSTGRES_PASSWORD='YOUR_LOCAL_LAB_PASSWORD' \
  -v day15-pgdata:/var/lib/postgresql \
  postgres:18.6
```

Điểm quan trọng:

```text
-p 127.0.0.1:5432:5432
```

khác:

```text
-p 5432:5432
```

Bind `127.0.0.1` chỉ expose PostgreSQL trên loopback host.

Cho lab application cùng host, blast radius nhỏ hơn.

Kiểm tra:

```bash
docker ps
```

Logs:

```bash
docker logs day15-postgres
```

Không cần học thuộc log output.

**Expected invariant:**

```text
container = running
PostgreSQL ready to accept connections
```

---

# 25. Verify process, port và readiness

```bash
docker ps --filter name=day15-postgres
```

```bash
ss -lnt | grep ':5432'
```

Expected:

```text
127.0.0.1:5432
```

Readiness từ trong container:

```bash
docker exec day15-postgres pg_isready -U postgres
```

Kiểm tra exit code:

```bash
echo $?
```

Expected:

```text
0
```

---

# 26. Inspect PostgreSQL runtime configuration

Login:

```bash
docker exec -it day15-postgres psql -U postgres
```

Trong `psql`:

```sql
SELECT version();

SHOW data_directory;
SHOW config_file;
SHOW hba_file;
SHOW listen_addresses;
SHOW port;
SHOW password_encryption;
```

Sau đó:

```sql
\l
```

Roles:

```sql
\du
```

Quit:

```text
\q
```

### Evidence

Lưu output không chứa secret:

```bash
docker exec day15-postgres psql -U postgres \
  -c "SELECT version();"
```

```bash
docker exec day15-postgres psql -U postgres \
  -c "SHOW hba_file;"
```

```bash
docker exec day15-postgres psql -U postgres \
  -c "SHOW listen_addresses;"
```

---

# 27. Tạo least-privilege role model

Login admin:

```bash
docker exec -it day15-postgres psql -U postgres
```

Tạo ownership role:

```sql
CREATE ROLE app_owner
    NOLOGIN
    NOSUPERUSER
    NOCREATEDB
    NOCREATEROLE;
```

Tạo runtime role:

```sql
CREATE ROLE app_runtime
    LOGIN
    NOSUPERUSER
    NOCREATEDB
    NOCREATEROLE
    NOREPLICATION;
```

Đặt password an toàn bằng interactive prompt:

```text
\password app_runtime
```

Không dùng:

```sql
CREATE ROLE app_runtime PASSWORD 'Secret123';
```

trong shell/history hoặc ticket thật nếu có thể tránh.

Verify:

```sql
\du
```

Bạn muốn thấy conceptually:

```text
app_owner   → Cannot login
app_runtime → Can login
```

và không có:

```text
Superuser
Create role
Create DB
Replication
```

---

# 28. Tạo database với ownership separation

```sql
CREATE DATABASE fresherdb OWNER app_owner;
```

Verify:

```sql
\l fresherdb
```

Reconnect:

```text
\c fresherdb
```

Tạo schema:

```sql
CREATE SCHEMA app AUTHORIZATION app_owner;
```

Kiểm tra:

```sql
\dn+
```

---

# 29. Tạo table bằng owner

Administrator có thể chuyển execution role:

```sql
SET ROLE app_owner;
```

Tạo table:

```sql
CREATE TABLE app.health_probe (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    message     text NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT current_timestamp
);
```

Insert seed:

```sql
INSERT INTO app.health_probe(message)
VALUES
    ('day15-seed-001'),
    ('day15-seed-002');
```

Kiểm tra:

```sql
SELECT * FROM app.health_probe;
```

Trở lại admin:

```sql
RESET ROLE;
```

---

# 30. Grant quyền cho application

Cho phép connect database:

```sql
GRANT CONNECT ON DATABASE fresherdb TO app_runtime;
```

Cho phép thấy objects trong schema:

```sql
GRANT USAGE ON SCHEMA app TO app_runtime;
```

Cho phép CRUD:

```sql
GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA app
TO app_runtime;
```

Identity/sequence:

```sql
GRANT USAGE, SELECT
ON ALL SEQUENCES IN SCHEMA app
TO app_runtime;
```

Đây mới chỉ giải quyết existing objects.

Nếu sau này owner tạo tables mới:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE app_owner
IN SCHEMA app
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES
TO app_runtime;
```

Và sequences:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE app_owner
IN SCHEMA app
GRANT USAGE, SELECT
ON SEQUENCES
TO app_runtime;
```

Đây là khác biệt giữa:

```text
GRANT ON ALL TABLES
```

và:

```text
ALTER DEFAULT PRIVILEGES
```

Cái đầu áp dụng objects **đã có**.

Cái sau kiểm soát privileges cho objects **được tạo trong tương lai bởi owner tương ứng**.

---

# 31. Configure schema lookup

Vì application của ta dùng schema `app`:

```sql
ALTER ROLE app_runtime
IN DATABASE fresherdb
SET search_path = app, pg_catalog;
```

Verify:

```sql
SELECT rolname
FROM pg_roles
WHERE rolname IN ('app_owner', 'app_runtime');
```

Sau đó exit:

```text
\q
```

---

# 32. Test bằng chính `app_runtime`

Đây là bước không được bỏ qua.

Không test bằng:

```text
postgres
```

rồi kết luận runtime user hoạt động.

Phải test bằng identity thật mà application sẽ sử dụng.

```bash
psql \
  -h 127.0.0.1 \
  -p 5432 \
  -U app_runtime \
  -d fresherdb
```

Nhập password ở prompt.

Sau khi vào:

```sql
SELECT current_user;
```

Expected:

```text
app_runtime
```

```sql
SELECT current_database();
```

Expected:

```text
fresherdb
```

```sql
SHOW search_path;
```

Sau đó:

```sql
SELECT * FROM health_probe;
```

Test INSERT:

```sql
INSERT INTO health_probe(message)
VALUES ('runtime-write-test');
```

Verify:

```sql
SELECT *
FROM health_probe
ORDER BY id;
```

---

# 33. Negative privilege testing

Least privilege chỉ được chứng minh khi ta test cả việc **allowed** lẫn **denied**.

Application runtime phải không thể:

```sql
CREATE DATABASE should_fail;
```

Expected:

```text
permission denied to create database
```

Test:

```sql
CREATE ROLE should_also_fail;
```

Expected:

```text
permission denied to create role
```

Test schema DDL:

```sql
CREATE TABLE app.should_fail (
    id integer
);
```

Expected:

```text
permission denied for schema app
```

Nhưng:

```sql
SELECT ...
INSERT ...
UPDATE ...
DELETE ...
```

phải hoạt động theo permission đã cấp.

Đây mới là bằng chứng của least privilege.

---

# 34. Inspect `pg_hba.conf`

Lấy exact path:

```bash
docker exec day15-postgres \
  psql -U postgres -Atc "SHOW hba_file;"
```

Không cần hardcode.

Có thể xem effective rule parsing:

```bash
docker exec day15-postgres \
  psql -U postgres \
  -c "SELECT line_number,type,database,user_name,address,auth_method,error FROM pg_hba_file_rules;"
```

Quan sát:

```text
rule ordering
local vs host
address
auth_method
error
```

Check lỗi:

```bash
docker exec day15-postgres \
  psql -U postgres \
  -c "SELECT line_number,error FROM pg_hba_file_rules WHERE error IS NOT NULL;"
```

**Expected invariant:**

```text
0 rows
```

---

# 35. Điều cần nhớ về HBA và network

Một rule:

```text
host fresherdb app_runtime 10.0.2.15/32 scram-sha-256
```

không có nghĩa remote app chắc chắn kết nối được.

Bạn vẫn phải có:

```text
PostgreSQL process running
        +
listen_addresses includes required interface
        +
Linux firewall permits
        +
AWS Security Group permits
        +
routing works
        +
pg_hba permits
        +
username/password correct
        +
database CONNECT
        +
schema/object privileges
```

Trong AWS:

```text
App EC2
    │
    │ TCP/5432
    ▼
DB EC2
```

Security Group nên giới hạn source tối đa có thể, chẳng hạn application security group hoặc private application source, **không mở `0.0.0.0/0:5432` chỉ để “test xem có chạy không”**.

---

# 36. Connect Java application

PostgreSQL JDBC URL cơ bản:

```text
jdbc:postgresql://127.0.0.1:5432/fresherdb
```

Credential:

```text
user     = app_runtime
password = secret
```

Không hardcode password vào source code.

Áp dụng cơ chế **externalized configuration của Day 13/14**.

Concept:

```text
Application config
│
├── DB URL
├── DB username
└── DB password
     │
     ▼
PostgreSQL JDBC Driver
     │
     ▼
PostgreSQL
```

Tôi không tự đặt tên biến cấu hình bắt buộc cho application của bạn vì syllabus không xác định framework/config keys cụ thể. Hãy dùng **đúng config keys mà application được cung cấp**.

### Supplementary — nếu application là Spring Boot

Ví dụ thường gặp:

```bash
SPRING_DATASOURCE_URL=jdbc:postgresql://127.0.0.1:5432/fresherdb
SPRING_DATASOURCE_USERNAME=app_runtime
SPRING_DATASOURCE_PASSWORD=...
```

Nhưng đây là ví dụ Spring Boot, **không phải requirement của syllabus**.

---

# 37. Application validation

Sau khi cấu hình:

```text
configuration updated
        ↓
syntax / secret permissions checked
        ↓
safe restart
        ↓
application startup logs
        ↓
DB connection succeeds
        ↓
application endpoint tested
        ↓
database side confirms session
```

Database-side evidence:

```sql
SELECT
    pid,
    usename,
    datname,
    application_name,
    client_addr,
    state
FROM pg_stat_activity
WHERE usename = 'app_runtime';
```

Bạn muốn chứng minh:

```text
Application → app_runtime → fresherdb
```

không chỉ:

```text
psql → database works
```

---

# 38. Backup lab

Tạo backup directory nếu chưa có:

```bash
cd ~/day15-postgres
mkdir -p backup
```

## Step 1 — logical database backup

```bash
docker exec day15-postgres \
  pg_dump \
    -U postgres \
    -d fresherdb \
    -Fc \
  > backup/fresherdb.dump
```

Check command:

```bash
echo $?
```

Expected:

```text
0
```

Check file:

```bash
ls -lh backup/fresherdb.dump
```

Không dùng:

```text
file exists
```

làm bằng chứng duy nhất.

Inspect archive:

```bash
docker exec -i day15-postgres \
  pg_restore -l \
  < backup/fresherdb.dump \
  | head
```

Nếu command đọc được archive listing, ta đã có thêm evidence rằng archive format hợp lệ ở mức cơ bản.

---

# 39. Backup global objects

```bash
docker exec day15-postgres \
  pg_dumpall \
    -U postgres \
    --globals-only \
  > backup/globals.sql
```

Check:

```bash
ls -lh backup/globals.sql
```

Cẩn thận:

`globals.sql` có thể chứa role/password metadata và privilege information.

Không upload công khai.

Không commit Git.

Không gửi vào ticket không được phép.

---

# 40. Record pre-restore evidence

Lấy invariant trước restore:

```bash
docker exec day15-postgres \
  psql -U postgres -d fresherdb \
  -c "SELECT count(*) FROM app.health_probe;"
```

Lấy actual records:

```bash
docker exec day15-postgres \
  psql -U postgres -d fresherdb \
  -c "TABLE app.health_probe;"
```

Có thể lưu:

```bash
docker exec day15-postgres \
  psql -U postgres -d fresherdb \
  -c "SELECT id,message FROM app.health_probe ORDER BY id;" \
  > evidence/source-data.txt
```

---

# 41. Restore vào database mới

Đây là cách an toàn hơn việc restore đè source database.

Không dùng ngay:

```text
--clean
```

trên DB đang phục vụ application.

Tạo target:

```bash
docker exec day15-postgres \
  createdb \
    -U postgres \
    -O app_owner \
    fresherdb_restore
```

Grant runtime access:

```bash
docker exec day15-postgres \
  psql -U postgres -d postgres \
  -c "GRANT CONNECT ON DATABASE fresherdb_restore TO app_runtime;"
```

Restore:

```bash
docker exec -i day15-postgres \
  pg_restore \
    -U postgres \
    --role=app_owner \
    --no-owner \
    -d fresherdb_restore \
  < backup/fresherdb.dump
```

Check:

```bash
echo $?
```

Expected:

```text
0
```

Custom-format dumps phải dùng `pg_restore`; plain-text dumps thường restore bằng `psql`. [PostgreSQL](https://www.postgresql.org/docs/current/backup-dump.html)

---

# 42. Validate restored database

Kiểm tra objects:

```bash
docker exec day15-postgres \
  psql -U postgres -d fresherdb_restore \
  -c "\dt app.*"
```

Check records:

```bash
docker exec day15-postgres \
  psql -U postgres -d fresherdb_restore \
  -c "SELECT id,message FROM app.health_probe ORDER BY id;"
```

Compare:

```text
source records
vs
restored records
```

Count:

```bash
docker exec day15-postgres \
  psql -U postgres -d fresherdb_restore \
  -c "SELECT count(*) FROM app.health_probe;"
```

Sau restore:

```bash
docker exec day15-postgres \
  psql -U postgres -d fresherdb_restore \
  -c "ANALYZE;"
```

PostgreSQL documentation khuyến nghị chạy `ANALYZE` sau restore để optimizer có statistics hữu ích. [PostgreSQL](https://www.postgresql.org/docs/current/backup-dump.html)

---

# 43. Test restored database như application

Đừng chỉ query bằng superuser.

```bash
psql \
  -h 127.0.0.1 \
  -U app_runtime \
  -d fresherdb_restore
```

Sau đó:

```sql
SELECT current_user;
SELECT * FROM app.health_probe;
```

Nếu cần schema search path cho restore database:

```sql
ALTER ROLE app_runtime
IN DATABASE fresherdb_restore
SET search_path = app, pg_catalog;
```

Application-like operation:

```sql
INSERT INTO app.health_probe(message)
VALUES ('restore-validation');
```

Nếu thành công:

```text
authentication          ✓
database access         ✓
schema access           ✓
table privilege         ✓
sequence privilege      ✓
restored structure      ✓
restored data           ✓
write operation         ✓
```

---

# 44. Backup theory quan trọng cần nhớ

Plain backup:

```bash
pg_dump dbname > dump.sql
```

Restore:

```bash
psql dbname < dump.sql
```

Custom backup:

```bash
pg_dump -Fc dbname > db.dump
```

Restore:

```bash
pg_restore -d dbname db.dump
```

Database-level:

```text
pg_dump
```

Cluster/global:

```text
pg_dumpall
```

Roles-only/global objects:

```bash
pg_dumpall --globals-only
```

`pg_dump` tạo internally consistent snapshot tại thời điểm dump bắt đầu, nhưng logical `pg_dump` không phải toàn bộ chiến lược disaster recovery cho mọi production workload. PostgreSQL docs thậm chí lưu ý rằng trong nhiều production use case phức tạp, chỉ dùng `pg_dump` cho regular backups không phải lựa chọn đầy đủ; các phương pháp như physical backup/WAL/PITR sẽ xuất hiện trong thiết kế production nâng cao hơn. [PostgreSQL](https://www.postgresql.org/docs/current/app-pgdump.html)

Đó là **Supplementary / Beyond explicit Day 15 syllabus**.

---

# 45. Failure Injection & Troubleshooting

Đây là phần bạn nên học theo layer, không theo kiểu thử ngẫu nhiên.

## Scenario A — `connection refused`

Ví dụ:

```text
psql: connection to server at "10.0.1.20", port 5432 failed:
Connection refused
```

Không sửa password.

Password chưa có cơ hội được kiểm tra.

Đi theo:

```text
Is hostname/IP correct?
      ↓
Can route reach host?
      ↓
Is PostgreSQL running?
      ↓
Is 5432 listening?
      ↓
Is listen_addresses correct?
      ↓
Firewall/SG?
```

Evidence:

```bash
getent hosts db-host
ping ...
ss -lntp
docker ps
docker logs ...
pg_isready ...
```

---

# 46. Scenario B — timeout

```text
connection timed out
```

Khác `refused`.

`refused` thường nghĩa target reachable nhưng không có service accept/bị reject trực tiếp.

Timeout thường khiến ta điều tra:

```text
routing
Security Group
NACL
firewall
wrong IP
network ACL
packet drop
```

Đừng lập tức restart PostgreSQL.

---

# 47. Scenario C — `no pg_hba.conf entry`

Ví dụ:

```text
FATAL: no pg_hba.conf entry for host ...
```

Điều này là evidence tốt:

```text
network reached PostgreSQL
      ↓
PostgreSQL received connection
      ↓
HBA did not permit it
```

Scope thu hẹp rất nhiều.

Check:

```sql
SHOW hba_file;
```

```sql
SELECT *
FROM pg_hba_file_rules;
```

Kiểm tra:

```text
database
user
source CIDR
SSL/non-SSL
rule order
```

---

# 48. Scenario D — password authentication failed

```text
FATAL: password authentication failed for user "app_runtime"
```

Ta đã vượt qua:

```text
DNS/network
TCP
listen
matching HBA rule
```

Khả năng:

```text
wrong username
wrong secret
wrong secret source
password rotated but application not updated
connecting to wrong DB server
```

Không “khắc phục” bằng cách đổi HBA sang:

```text
trust
```

Đó là anti-pattern nguy hiểm.

---

# 49. Scenario E — `permission denied for schema`

Ví dụ:

```text
ERROR: permission denied for schema app
```

Connectivity và authentication đều đã thành công.

Check:

```sql
\dn+
```

và:

```sql
SELECT has_schema_privilege(
    'app_runtime',
    'app',
    'USAGE'
);
```

Expected:

```text
true
```

---

# 50. Scenario F — `permission denied for table`

```text
ERROR: permission denied for table health_probe
```

Check:

```sql
SELECT has_table_privilege(
    'app_runtime',
    'app.health_probe',
    'SELECT'
);
```

Nếu application cần write:

```sql
SELECT has_table_privilege(
    'app_runtime',
    'app.health_probe',
    'INSERT'
);
```

---

# 51. Scenario G — INSERT fails on sequence

Một lỗi rất hay gặp:

```text
permission denied for sequence ...
```

Bạn đã grant:

```sql
INSERT ON TABLE
```

nhưng table sử dụng identity/sequence.

Runtime role vẫn cần appropriate sequence privilege.

Đó là lý do lab có:

```sql
GRANT USAGE, SELECT
ON ALL SEQUENCES IN SCHEMA app
TO app_runtime;
```

Troubleshooting phải dựa trên exact object trong error, không “GRANT ALL” vì lười phân tích.

---

# 52. Scenario H — application cannot see table

Application query:

```sql
SELECT * FROM health_probe;
```

nhưng nhận:

```text
relation "health_probe" does not exist
```

Trong khi:

```sql
SELECT * FROM app.health_probe;
```

hoạt động.

Check:

```sql
SHOW search_path;
```

Có thể đây là namespace resolution problem, không phải table mất.

---

# 53. Scenario I — backup succeeds nhưng restore fails

Ví dụ:

```text
role "some_owner" does not exist
```

Nguyên nhân thường là:

```text
database dump contains ownership/ACL references
but global role not present
```

Đây là lý do phải hiểu:

```text
pg_dump != roles backup
```

Có thể cần:

```bash
pg_dumpall --globals-only
```

hoặc restore policy:

```text
--no-owner
--role=...
```

tùy yêu cầu.

Official docs lưu ý các roles sở hữu objects hoặc nhận grants cần tồn tại nếu restore muốn giữ ownership/permissions tương ứng. [PostgreSQL](https://www.postgresql.org/docs/current/backup-dump.html)

---

# 54. Scenario J — container recreated và database “biến mất”

Điều tra:

```bash
docker inspect day15-postgres
docker volume ls
docker volume inspect day15-pgdata
```

Check:

```text
Did we attach expected volume?
Was wrong path mounted?
Was a new anonymous volume created?
Was PGDATA changed?
```

Không khởi tạo thêm nhiều container lung tung trước khi hiểu storage mapping.

---

# 55. Safe HBA failure injection

Chỉ làm trong **lab**, không production.

Conceptual exercise:

```text
backup pg_hba.conf
        ↓
add reject rule for test user
        ↓
validate
        ↓
reload
        ↓
new connection must fail
        ↓
restore config
        ↓
reload
        ↓
connection succeeds again
```

Điều quan trọng ở đây không phải thuộc command editing.

Điều quan trọng là operational procedure:

```text
backup
validation
reload
new-session test
rollback
revalidation
evidence
```

Không nên phá HBA trên machine bạn đang phụ thuộc để giữ access nếu chưa có recovery path.

---

# 56. Security & Least Privilege Checklist

| Control                                      | Expected |
| -------------------------------------------- | -------- |
| Application uses `postgres`                  | ❌       |
| Runtime role has `SUPERUSER`                 | ❌       |
| Runtime role has `CREATEDB`                  | ❌       |
| Runtime role has `CREATEROLE`                | ❌       |
| DB password committed to Git                 | ❌       |
| `trust` for remote network                   | ❌       |
| PostgreSQL exposed `0.0.0.0/0` unnecessarily | ❌       |
| Runtime has only required CRUD               | ✅       |
| Dedicated DB/schema                          | ✅       |
| SCRAM authentication                         | ✅       |
| Backup protected as sensitive data           | ✅       |
| Restore tested                               | ✅       |
| HBA source CIDR narrow                       | ✅       |
| Config changes validated                     | ✅       |

---

# 57. Common anti-patterns

### Anti-pattern 1

```text
"App không kết nối được → restart PostgreSQL."
```

Không evidence-driven.

### Anti-pattern 2

```sql
GRANT ALL PRIVILEGES ...
```

chỉ vì permission error.

Phải tìm exact missing privilege.

### Anti-pattern 3

Application sử dụng:

```text
postgres
```

### Anti-pattern 4

Expose:

```text
0.0.0.0:5432
```

ra Internet.

### Anti-pattern 5

HBA:

```text
host all all 0.0.0.0/0 trust
```

Rất nguy hiểm.

### Anti-pattern 6

Có backup nhưng chưa bao giờ restore.

### Anti-pattern 7

Chỉ chạy:

```bash
pg_isready
```

rồi kết luận application healthy.

`pg_isready` chứng minh readiness, không chứng minh application authentication/object permissions.

### Anti-pattern 8

Hardcode PostgreSQL config path.

Dùng:

```sql
SHOW hba_file;
SHOW config_file;
SHOW data_directory;
```

### Anti-pattern 9

Backup file đặt permission quá rộng.

Check:

```bash
ls -l backup/
```

---

# 58. Rollback / Recovery

Giả sử HBA modification làm application fail.

Rollback:

```text
preserve evidence
      ↓
restore known-good pg_hba.conf
      ↓
validate pg_hba_file_rules
      ↓
reload
      ↓
test new connection
      ↓
validate application
```

Nếu role permissions bị thay đổi:

```text
identify exact GRANT/REVOKE change
      ↓
restore previous privilege state
      ↓
test with app_runtime
```

Nếu restore thử nghiệm lỗi:

```text
Do NOT damage source DB
      ↓
drop/recreate isolated restore target
      ↓
retry restore with corrected procedure
```

---

# 59. Cảnh báo destructive operations

Những command như:

```sql
DROP DATABASE
DROP ROLE
DROP SCHEMA ... CASCADE
DROP TABLE
TRUNCATE
```

hay restore:

```text
--clean
```

có khả năng phá dữ liệu.

Trước production operation phải xác minh ít nhất:

```text
correct environment
correct server
correct database
recent backup
recovery path
change approval
blast radius
rollback procedure
```

Một command technically correct nhưng chạy nhầm production vẫn là operational failure.

---

# 60. Evidence pack bắt buộc cho Day 15

Nếu tôi là mentor chấm bài Day 15 của bạn, tôi muốn thấy evidence kiểu sau:

| Evidence                           | Chứng minh                  |
| ---------------------------------- | --------------------------- |
| `SELECT version()`                 | PostgreSQL provisioned      |
| `docker ps` / `systemctl status`   | Service running             |
| `ss ...5432`                       | Listener                    |
| `pg_isready` + rc 0                | Protocol readiness          |
| `\du`                              | Roles                       |
| `\l`                               | Database ownership          |
| `\dn+`                             | Schema ownership/privileges |
| Negative privilege tests           | Least privilege thật        |
| `SHOW hba_file`                    | HBA awareness               |
| `pg_hba_file_rules` no errors      | Valid HBA                   |
| Runtime `SELECT/INSERT`            | App role usable             |
| Application logs/endpoint          | Java → PostgreSQL works     |
| `pg_dump` exit 0 + archive         | Backup                      |
| Restore exit 0                     | Recovery executed           |
| Source vs restore row verification | Data recovered              |
| Cleanup record                     | Environment controlled      |

Không lưu password/token/private key vào evidence.

---

# 61. Cleanup

Sau khi **đã hoàn thành evidence và nếu lab không còn cần cho Day 16**, có thể cleanup.

Trước tiên:

```bash
docker ps
docker volume ls
ls -lh backup/
```

Stop:

```bash
docker stop day15-postgres
```

Remove container:

```bash
docker rm day15-postgres
```

Nếu **chắc chắn không cần data nữa**:

```bash
docker volume rm day15-pgdata
```

Remove network:

```bash
docker network rm day15-net
```

### CẢNH BÁO

```bash
docker volume rm day15-pgdata
```

xóa persistent PostgreSQL lab data.

Không thực hiện nếu Day 16 sẽ tiếp tục dùng cùng database.

Tôi khuyến nghị **giữ database này cho Day 16**, vì Day 16 yêu cầu diagnostic connection/slow query/session/lock/capacity. Syllabus chuyển trực tiếp từ PostgreSQL foundations Day 15 sang troubleshooting PostgreSQL Day 16. 00_MASTER_SYLLABUS_Java_Fresher…

---

# 62. L1 troubleshooting model cần thuộc

Khi application báo:

```text
Cannot connect to database
```

đừng bắt đầu bằng Google error.

Đi theo:

```text
1. Symptom
   ↓
2. Expected behavior
   ↓
3. Recent changes
   ↓
4. Scope
   one app?
   all apps?
   one DB?
   whole PostgreSQL?
   ↓
5. Network path
   DNS → route → SG/firewall → TCP
   ↓
6. PostgreSQL process/listener
   ↓
7. pg_hba
   ↓
8. authentication
   ↓
9. database CONNECT
   ↓
10. schema/object privileges
   ↓
11. SQL/app behavior
   ↓
12. logs/evidence
   ↓
13. safest remediation
   ↓
14. validate
   ↓
15. preserve evidence / escalate
```

Đây là tư duy mà fresher Administration phải hình thành.

---

# 63. Exam Focus — Theory

Bạn phải giải thích rõ được các cặp khái niệm sau:

| Câu hỏi                                      | Ý chính phải trả lời                                                         |
| -------------------------------------------- | ---------------------------------------------------------------------------- |
| PostgreSQL role vs user?                     | PostgreSQL unified role model; `LOGIN` biến role thành login identity        |
| Database vs schema?                          | Database boundary lớn; schema là namespace trong DB                          |
| `postgres` nên dùng cho app?                 | Không, vi phạm least privilege                                               |
| `listen_addresses` vs HBA?                   | Listen = interface PostgreSQL accept TCP; HBA = auth/access rules            |
| `pg_hba.conf` matching?                      | First matching rule wins, không fallback                                     |
| `pg_isready` chứng minh gì?                  | Server acceptance/readiness, không chứng minh app privilege                  |
| `pg_dump` backup gì?                         | Một database                                                                 |
| `pg_dump` có roles không?                    | Không                                                                        |
| Roles backup bằng gì?                        | `pg_dumpall --globals-only`                                                  |
| Custom dump restore bằng gì?                 | `pg_restore`                                                                 |
| Backup có đồng nghĩa recovery ready?         | Không, phải restore test                                                     |
| Schema `USAGE` là gì?                        | Cho phép truy cập objects qua namespace nhưng không tự cấp object privileges |
| `GRANT ON ALL TABLES` vs default privileges? | Existing objects vs future objects                                           |
| `trust` có an toàn remote không?             | Không phù hợp cho untrusted remote access                                    |
| Tại sao SCRAM?                               | Strong password authentication; MD5 đang deprecated                          |

---

# 64. Exam Focus — Practical

Một practical exam hợp với syllabus có thể giao tình huống:

```text
PostgreSQL available.
Create:

database: fresherdb
schema: app
runtime user: app_runtime

Application needs:
SELECT
INSERT
UPDATE
DELETE

It must not:
create database
create role
create table

Connect Java application.
Create a backup.
Restore to fresherdb_restore.
Submit evidence.
```

Bạn cần tự làm được workflow:

```text
inspect
→ role
→ database
→ schema
→ grants
→ connectivity
→ application config
→ application validation
→ dump
→ restore
→ verify
→ document
```

Nếu chỉ thuộc SQL syntax nhưng không biết cách validate thì chưa đạt mục tiêu Administration.

---

# 65. Knowledge Check

Hãy tự trả lời trước khi nhìn đáp án.

**Q1.** `CREATE USER foo` khác `CREATE ROLE foo` thế nào?  
**Đáp án:** Cả hai tạo role; `CREATE USER` mặc định mang `LOGIN`, trong khi `CREATE ROLE` mặc định `NOLOGIN`.

**Q2.** App kết nối tới IP/5432 bị `connection refused`. Có nên sửa `pg_hba.conf` trước không?  
**Đáp án:** Không. Connection chưa tới authentication layer. Kiểm tra process, listener, host/port và network trước.

**Q3.** `pg_isready` return code `2` có nghĩa gì?  
**Đáp án:** Không nhận được response từ server.

**Q4.** Application login được nhưng `SELECT * FROM app.customer` báo permission denied. HBA có phải nguyên nhân chính không?  
**Đáp án:** Không. Authentication đã qua; điều tra schema/table privileges.

**Q5.** `pg_dump fresherdb` có đủ để recreate tất cả roles không?  
**Đáp án:** Không.

**Q6.** Vì sao nên dùng `app_owner NOLOGIN`?  
**Đáp án:** Tách ownership khỏi runtime authentication; giảm quyền của credential mà application sử dụng.

**Q7.** Một HBA rule đầu tiên match nhưng password sai. PostgreSQL có thử rule match thứ hai không?  
**Đáp án:** Không.

**Q8.** Có `.dump` file nghĩa là backup strategy đã được xác nhận chưa?  
**Đáp án:** Chưa. Cần restore và validate.

**Q9.** Port 5432 đang LISTEN chứng minh gì?  
**Đáp án:** Có process listen trên socket đó; chưa chứng minh PostgreSQL authentication, DB availability, application privileges hoặc application functionality.

**Q10.** Khi application cần auto-generated identity nhưng INSERT báo sequence permission denied, tại sao?  
**Đáp án:** Table INSERT privilege và sequence privilege là hai permission khác nhau.

---

# 66. Independent Practical Challenge

Sau khi hoàn thành Guided Lab, hãy tự làm challenge này **không nhìn lại command phía trên**:

```text
Environment:
PostgreSQL 18.x.

Requirements:
- DB: opsdb
- Schema: inventory
- Ownership role: inventory_owner, NOLOGIN
- Runtime role: inventory_app, LOGIN
- Table:
    inventory.assets
    id
    hostname
    status
    created_at

Runtime role may:
- connect
- select
- insert
- update

Runtime role may NOT:
- delete
- create tables
- create DB
- create roles

Tasks:
1. Provision.
2. Configure privileges.
3. Prove allowed operations.
4. Prove denied operations.
5. Inspect HBA.
6. Run health checks.
7. Create 5 records.
8. Backup database.
9. Restore into opsdb_restore.
10. Prove all 5 original records exist.
11. Produce evidence.
```

**Mastery condition không phải chỉ là “command chạy được”.**

Bạn phải giải thích được:

```text
why each role exists
why each privilege was granted
why denied tests matter
why HBA wasn't enough
why backup required restore validation
```

---

# 67. Day 15 Mastery Check

Bạn chỉ nên xem Day 15 là **practically mastered** khi có thể tự chứng minh tất cả các trạng thái sau:

```text
PostgreSQL provisioned                         ✓
persistent storage understood                 ✓
service/container healthy                     ✓
TCP listener understood                       ✓
role model understood                         ✓
app does not use superuser                     ✓
database/schema ownership correct             ✓
least-privilege grants correct                ✓
negative privilege tests pass                 ✓
pg_hba rule matching understood               ✓
HBA parser validation understood              ✓
runtime authentication works                  ✓
Java application actually connects            ✓
backup created                                ✓
global objects distinction understood         ✓
clean restore performed                       ✓
restored data verified                        ✓
evidence preserved                            ✓
rollback/recovery thinking demonstrated       ✓
```

Điểm quan trọng nhất của Day 15 có thể cô đọng thành mental model này:

```text
                     PostgreSQL
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
   Availability      Security           Recovery
       │                 │                 │
   process           roles             pg_dump
   listener          HBA               pg_dumpall
   pg_isready        privileges        pg_restore
   SELECT 1          secrets           validation
       │                 │                 │
       └─────────────────┼─────────────────┘
                         │
                  Java Application
```

Một Java Administration fresher không chỉ cần biết **“cách tạo database”**. Bạn phải có khả năng chứng minh toàn chuỗi:

**provision → configure → restrict → connect → validate → observe → diagnose → backup → restore → verify → document → hand over.**

Đó chính là mục tiêu đầy đủ của Day 15 theo syllabus hôm nay. 00_MASTER_SYLLABUS_Java_Fresher…
