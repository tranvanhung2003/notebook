# Day 16 — PostgreSQL Diagnostics, Sessions/Locks, Capacity, EXPLAIN & L1 Escalation

Day 16 trong MASTER_SYLLABUS quy định chính xác:

> **Concept/Lecture:** SQL/log diagnostics, sessions/locks, capacity checks, EXPLAIN basics, and L1 escalation boundaries.  
> **Assignment/Lab:** Diagnose a connection or slow-query scenario, collect evidence, and decide whether to remediate or escalate. 00_MASTER_SYLLABUS_Java_Fresher…

Đây là một ngày rất quan trọng về **troubleshooting mindset**. Day 15 giúp bạn provision và vận hành PostgreSQL; Day 16 chuyển trọng tâm sang câu hỏi thực tế của Operations:

**“Ứng dụng báo database lỗi hoặc chậm — tôi phải xác định nguyên nhân như thế nào mà không làm sự cố nghiêm trọng hơn?”**

Tài liệu dưới đây dùng PostgreSQL 14–18 làm phạm vi thực hành, ưu tiên syntax tương thích rộng. Tại thời điểm hiện tại, tài liệu PostgreSQL chính thức đánh dấu PostgreSQL 18 là bản **Current**, trong khi PostgreSQL 19 vẫn đang ở Beta 4. [PostgreSQL](https://www.postgresql.org/docs/?utm_source=chatgpt.com)

---

# 1. Syllabus Alignment

## Mandatory — bắt buộc Day 16

Bạn cần thực sự làm chủ sáu năng lực:

| Nội dung        | Bạn phải làm được                                                          |
| --------------- | -------------------------------------------------------------------------- |
| SQL diagnostics | Xác định query/session nào có vấn đề, duration, state, wait                |
| Log diagnostics | Tìm đúng PostgreSQL log, đọc severity, SQLSTATE và correlation information |
| Sessions        | Hiểu connection → backend/session → transaction → query                    |
| Locks           | Nhận diện blocked session và blocking session                              |
| Capacity checks | Connection capacity, database/table size, filesystem/disk                  |
| EXPLAIN basics  | Đọc được scan, cost, rows, actual time, buffers cơ bản                     |
| L1 boundary     | Biết hành động nào được phép, hành động nào phải escalation                |
| Assignment      | Diagnose → evidence → remediation/escalation decision                      |

## Supplementary / Beyond the explicit syllabus

Mình sẽ bổ sung những kiến thức rất hữu ích cho Fresher Operations nhưng không thay thế syllabus: `wait_event`, `SQLSTATE`, `pg_blocking_pids()`, `statement_timeout`, `lock_timeout`, `pg_cancel_backend()`, `pg_terminate_backend()`, correlation với Linux, application connection pool và cách xây dựng evidence pack.

---

# 2. Mục tiêu cuối Day 16

Sau Day 16, khi gặp ticket kiểu:

```text
Application API is slow.
Database connection timeout.
Request /orders takes 30 seconds.
PostgreSQL CPU looks normal.
```

bạn không được đoán ngay:

> “Database chậm vì thiếu index.”

Bạn phải suy luận theo chuỗi:

```text
Symptom
   ↓
Scope
   ↓
Connectivity / authentication?
   ↓
Connection capacity?
   ↓
Session executing or waiting?
   ↓
Lock / blocker?
   ↓
Database / filesystem capacity?
   ↓
Logs?
   ↓
Execution plan?
   ↓
Root cause / strongest hypothesis
   ↓
L1-safe remediation OR escalation
   ↓
Validation
   ↓
Evidence / handover
```

Đây chính là tư duy Operations mà Day 16 muốn xây dựng.

---

# 3. Mental Model quan trọng nhất

Hãy hình dung một Java application kết nối PostgreSQL:

```text
Java Application
      │
      │ JDBC / TCP
      ▼
DNS / IP
      │
      ▼
TCP port 5432
      │
      ▼
PostgreSQL listening socket
      │
      ▼
pg_hba.conf authentication rule
      │
      ▼
Authentication
      │
      ▼
PostgreSQL backend/session
      │
      ▼
Transaction
      │
      ▼
SQL statement
      │
      ├── waiting for lock?
      ├── waiting for I/O?
      ├── executing CPU work?
      │
      ▼
Planner → Execution Plan
      │
      ▼
Tables / Indexes / Buffers
      │
      ▼
Filesystem / Disk
```

Một request chậm ở Java chỉ nói cho bạn rằng:

> **elapsed time của request lớn.**

Nó chưa chứng minh database engine đang “xử lý SQL chậm”.

Query có thể:

```text
30 seconds total
│
├── 29.8 seconds WAITING for another transaction
└── 0.2 seconds actually executing
```

Nếu bạn chỉ chạy `EXPLAIN`, bạn có thể hoàn toàn điều tra sai hướng.

---

# 4. PostgreSQL session là gì?

Mỗi client connection tới PostgreSQL thường tương ứng với một **backend process/session**.

Bạn có thể xem activity bằng:

```sql
SELECT *
FROM pg_stat_activity;
```

`pg_stat_activity` có một row cho mỗi server process và cung cấp các thông tin quan trọng như `pid`, user, database, client, session start, transaction start, query start, `state`, `wait_event_type`, `wait_event`, query hiện tại. [PostgreSQL](https://www.postgresql.org/docs/current/monitoring-stats.html)

Trong Operations, không nên dùng ngay `SELECT *`. Hãy lấy những cột thực sự cần:

```sql
SELECT
    pid,
    datname,
    usename,
    application_name,
    client_addr,
    backend_start,
    xact_start,
    query_start,
    state,
    wait_event_type,
    wait_event,
    LEFT(query, 150) AS query
FROM pg_stat_activity
ORDER BY query_start;
```

---

# 5. Hiểu `state` chính xác

Các state Fresher phải nhớ:

| State                           | Ý nghĩa                                            |
| ------------------------------- | -------------------------------------------------- |
| `active`                        | Backend đang thực hiện query                       |
| `idle`                          | Connection tồn tại nhưng đang chờ command mới      |
| `idle in transaction`           | Transaction vẫn mở nhưng hiện không chạy statement |
| `idle in transaction (aborted)` | Transaction đã gặp error nhưng chưa rollback       |

PostgreSQL documentation xác định `active`, `idle`, `idle in transaction` theo đúng các ý nghĩa trên. [PostgreSQL](https://www.postgresql.org/docs/current/monitoring-stats.html)

Điểm đặc biệt cần nhớ:

```text
idle != lỗi
```

Connection pool của Java có thể giữ nhiều session `idle` hoàn toàn bình thường.

Ngược lại:

```text
idle in transaction lâu
```

lại rất đáng quan tâm.

Ví dụ:

```sql
BEGIN;

UPDATE orders
SET status = 'PROCESSING'
WHERE id = 1;

-- application không COMMIT/ROLLBACK
-- rồi connection bị bỏ idle
```

Transaction vẫn tồn tại và lock vẫn có thể được giữ.

Đây là một trong những tình huống kinh điển dẫn tới blocking.

---

# 6. Query age và transaction age

Hai timestamp cực kỳ quan trọng:

```text
query_start
xact_start
```

Query đang chạy bao lâu:

```sql
SELECT
    pid,
    usename,
    state,
    now() - query_start AS query_age,
    LEFT(query, 120) AS query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY query_start;
```

Transaction tồn tại bao lâu:

```sql
SELECT
    pid,
    usename,
    state,
    now() - xact_start AS transaction_age,
    LEFT(query, 120) AS query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

Đừng nhầm hai khái niệm:

```text
Transaction 20 minutes old
        │
        ├── Query A: 2 ms
        ├── Query B: 8 ms
        └── currently idle
```

Transaction age có thể rất lớn dù query hiện tại không chạy lâu.

---

# 7. `wait_event_type` và `wait_event`

Đây là bước phân biệt:

> Query đang **working** hay **waiting**?

Ví dụ:

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    now() - query_start AS query_age,
    LEFT(query, 150) AS query
FROM pg_stat_activity
WHERE state <> 'idle';
```

Nếu nhận:

```text
 pid  | state  | wait_event_type | wait_event
------+--------+-----------------+------------
 8123 | active | Lock            | transactionid
```

điều đó không có nghĩa backend “idle”.

Nó đang:

```text
state = active
```

nhưng query bị blocked trong khi execution.

PostgreSQL documentation nhấn mạnh `state` và `wait_event` độc lập; một backend có thể `active` nhưng `wait_event` khác NULL, nghĩa là query đang execution nhưng bị chặn ở đâu đó. [PostgreSQL](https://www.postgresql.org/docs/current/monitoring-stats.html)

Đây là kiến thức thi và troubleshooting rất quan trọng.

---

# 8. Locks — nền tảng lý thuyết

Database cần locks để bảo đảm concurrency correctness.

Ví dụ hai transaction:

### Transaction A

```sql
BEGIN;

UPDATE orders
SET status = 'PROCESSING'
WHERE id = 100;
```

Chưa commit.

### Transaction B

```sql
UPDATE orders
SET status = 'PAID'
WHERE id = 100;
```

Transaction B không thể tùy tiện overwrite row mà transaction A đang thay đổi.

Nó phải đợi.

Mental model:

```text
Transaction A
      │
      │ holds lock
      ▼
  order id=100
      ▲
      │ wants conflicting lock
      │
Transaction B
      │
      ▼
    WAIT
```

---

# 9. `pg_locks`

View chuẩn để quan sát locks là:

```sql
SELECT *
FROM pg_locks;
```

Các cột quan trọng:

```text
pid
locktype
mode
granted
waitstart
relation
```

`granted = true`

nghĩa là process đã có lock.

`granted = false`

nghĩa là process đang chờ lock.

PostgreSQL mô tả `pg_locks` là view chứa active locks; `granted=false` thể hiện process đang chờ conflicting lock. [PostgreSQL](https://www.postgresql.org/docs/current/view-pg-locks.html)

Query hữu dụng hơn:

```sql
SELECT
    a.pid,
    a.usename,
    a.state,
    l.locktype,
    l.mode,
    l.granted,
    l.waitstart,
    LEFT(a.query, 120) AS query
FROM pg_locks l
JOIN pg_stat_activity a
    ON a.pid = l.pid
WHERE a.datname = current_database()
ORDER BY l.granted, a.pid;
```

Tuy nhiên, bạn **không nên cố tự viết một self-join phức tạp trên `pg_locks` để xác định blocker**.

Có công cụ tốt hơn.

---

# 10. `pg_blocking_pids()` — cực kỳ quan trọng

Giả sử PID 8123 đang bị block:

```sql
SELECT pg_blocking_pids(8123);
```

Kết quả:

```text
 pg_blocking_pids
------------------
 {7984}
```

Nghĩa là PID:

```text
7984
```

đang block:

```text
8123
```

PostgreSQL documentation cũng khuyến nghị dùng `pg_blocking_pids()` thay vì cố tự tính quan hệ blocking bằng self-join phức tạp trên `pg_locks`. [PostgreSQL](https://www.postgresql.org/docs/current/view-pg-locks.html)

Một query cực kỳ hữu dụng cho L1:

```sql
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - xact_start AS xact_age,
    now() - query_start AS query_age,
    pg_blocking_pids(pid) AS blocking_pids,
    LEFT(query, 150) AS query
FROM pg_stat_activity
WHERE datname = current_database()
  AND pid <> pg_backend_pid()
ORDER BY query_start;
```

Ví dụ:

```text
 pid  | state  | wait_event_type | blocking_pids
------+--------+-----------------+--------------
 8123 | active | Lock            | {7984}
 7984 | idle in transaction      | {}
```

Đây gần như là smoking gun:

```text
PID 7984
   ↓
idle in transaction
   ↓
holds transaction locks
   ↓
blocks PID 8123
```

---

# 11. Blocking khác Deadlock

Đây là câu hỏi lý thuyết rất hay.

## Blocking

```text
A holds resource
B waits for A

A → B
```

B có thể đợi đến khi A commit/rollback.

## Deadlock

```text
Transaction A owns X
Transaction B owns Y

A waits for Y
B waits for X
```

Tức là:

```text
A → waits B
↑         ↓
└─────────┘
```

Không transaction nào có thể tiếp tục.

PostgreSQL sẽ phát hiện deadlock và abort một transaction.

SQLSTATE đáng chú ý:

```text
40P01 = deadlock_detected
```

PostgreSQL sử dụng SQLSTATE 5 ký tự; các application nên dựa vào error code thay vì chỉ parse human-readable message vì code ổn định hơn và không phụ thuộc localization. [PostgreSQL](https://www.postgresql.org/docs/current/errcodes-appendix.html?utm_source=chatgpt.com)

---

# 12. SQL diagnostics — đừng chỉ đọc message text

Một database error nên được phân tích theo:

```text
timestamp
severity
SQLSTATE
database/user/application
PID/session
message
detail
hint
context
statement/query
```

Một số SQLSTATE hữu ích:

| SQLSTATE / Class | Ý nghĩa                |
| ---------------- | ---------------------- |
| `08xxx`          | Connection exception   |
| `23505`          | unique violation       |
| `42501`          | insufficient privilege |
| `42P01`          | undefined table        |
| `40P01`          | deadlock detected      |
| `53300`          | too many connections   |

Không cần học thuộc toàn bộ bảng SQLSTATE.

Bạn cần hiểu:

> SQLSTATE tốt hơn việc search một chuỗi error text mơ hồ.

---

# 13. PostgreSQL logs nằm ở đâu?

Không có một path duy nhất áp dụng cho mọi hệ thống.

Ví dụ Debian/Ubuntu package, RHEL package, container và custom installation có thể khác nhau.

Đầu tiên hỏi PostgreSQL:

```sql
SHOW log_destination;
SHOW logging_collector;
SHOW log_directory;
SHOW log_filename;
```

Nếu logging collector đang dùng:

```sql
SELECT pg_current_logfile();
```

`pg_current_logfile()` trả path log hiện tại của logging collector và trả `NULL` nếu logging collector không hoạt động. [PostgreSQL](https://www.postgresql.org/docs/18/functions-info.html?utm_source=chatgpt.com)

Bạn cũng nên biết:

```sql
SHOW data_directory;
SHOW config_file;
SHOW hba_file;
```

Đây là cách tốt hơn việc đoán path.

---

# 14. systemd logs

Nếu PostgreSQL gửi output qua systemd journal:

Trước tiên tìm unit thật:

```bash
systemctl list-units --type=service | grep -i postgres
```

Sau đó:

```bash
sudo journalctl -u <actual-postgresql-unit>
```

Trong troubleshooting gần thời điểm incident:

```bash
sudo journalctl -u <actual-postgresql-unit> \
  --since "10 minutes ago"
```

Không nên blindly dùng:

```bash
journalctl -u postgresql
```

và mặc định cho rằng đó luôn là unit thật.

Ví dụ Debian có thể có cluster-specific unit khác với RHEL hoặc custom deployment.

---

# 15. Logging settings quan trọng

## `log_min_duration_statement`

Ví dụ:

```conf
log_min_duration_statement = 500ms
```

nghĩa là statement hoàn thành với duration ≥ 500 ms sẽ được logged.

PostgreSQL xác nhận parameter này được dùng để tìm những query mất nhiều thời gian; `-1` mặc định là disabled. [PostgreSQL](https://www.postgresql.org/docs/current/runtime-config-logging.html)

### Nhưng không được bật bừa trong Production

Ví dụ:

```conf
log_min_duration_statement = 0
```

có thể khiến mọi statement duration được log.

Hậu quả:

```text
huge log volume
disk growth
I/O overhead
sensitive SQL/parameter exposure
```

Vì vậy:

> Thay logging configuration là configuration change, không phải diagnostic command vô hại.

---

# 16. `log_lock_waits`

Một setting hữu ích khác:

```conf
log_lock_waits = on
```

PostgreSQL có thể log khi session phải chờ lock lâu hơn `deadlock_timeout`. [PostgreSQL](https://www.postgresql.org/docs/current/runtime-config-logging.html)

Rất hữu ích nếu incident chỉ xảy ra thỉnh thoảng.

Nhưng vẫn cần:

```text
approval
change record
baseline
rollback
disk/log-volume consideration
```

Không tự bật Production vì “muốn debug”.

---

# 17. Correlation bằng `log_line_prefix`

Một log line có PID giúp bạn liên hệ:

```text
application error
      ↓
PostgreSQL PID
      ↓
pg_stat_activity
      ↓
pg_locks
      ↓
database log
```

Ví dụ configurable prefix:

```conf
log_line_prefix = '%m [%p] %q%u@%d/%a '
```

Trong đó có thể chứa timestamp, PID, user, database, application.

PostgreSQL documentation cho phép các fields như `%p` PID, `%u` username, `%d` database, `%a` application name và `%e` SQLSTATE. [PostgreSQL](https://www.postgresql.org/docs/current/runtime-config-logging.html)

Đây chính là **correlation**, một kỹ năng Operations rất quan trọng.

---

# 18. Security khi đọc và thu thập log

Log có thể chứa:

```text
SQL statement
table names
business data
usernames
client IP
application parameters
error details
```

Vì vậy evidence không nên copy cả file log ra ticket.

Hãy lấy khoảng thời gian liên quan:

```bash
grep ...
```

hoặc:

```bash
journalctl --since ...
```

và redact dữ liệu nhạy cảm nếu policy yêu cầu.

Không thu thập:

```text
password
JDBC password
AWS access key
token
private key
full secret connection string
```

---

# 19. Connection diagnostics — tư duy theo layer

Giả sử Java báo:

```text
Connection to db-server:5432 refused
```

Đừng bắt đầu bằng việc sửa `pg_hba.conf`.

Điều tra theo layer:

```text
Application config
      ↓
DNS
      ↓
Network route
      ↓
TCP
      ↓
Server listening
      ↓
PostgreSQL available
      ↓
Authentication / pg_hba
      ↓
Database/user privilege
      ↓
Connection capacity
```

## Client-side

DNS:

```bash
getent hosts db-server.example.internal
```

Port:

```bash
nc -vz db-server.example.internal 5432
```

nếu `nc` có sẵn.

PostgreSQL-aware test:

```bash
pg_isready \
  -h db-server.example.internal \
  -p 5432
```

Ví dụ:

```text
db-server.example.internal:5432 - accepting connections
```

Điểm cần nhớ:

> `pg_isready` cho biết PostgreSQL có phản hồi ở mức connection availability; nó không chứng minh application credentials có quyền đăng nhập thành công.

---

# 20. Test thật bằng `psql`

Ví dụ:

```bash
psql \
  -h db-server.example.internal \
  -p 5432 \
  -U app_user \
  -d appdb
```

Để client prompt password.

Không viết:

```bash
psql postgresql://app_user:SuperSecretPassword@db-server/appdb
```

rồi copy command vào ticket.

Password có thể xuất hiện trong:

```text
shell history
process inspection
screenshots
logs
chat
ticket evidence
```

---

# 21. Server-side connectivity

Trên DB server:

```bash
ss -ltnp | grep ':5432'
```

Bạn muốn xác định có process listening hay không.

Sau đó trong PostgreSQL:

```sql
SHOW listen_addresses;
SHOW port;
```

Port mặc định PostgreSQL là 5432, nhưng production hoàn toàn có thể cấu hình khác. PostgreSQL cũng xác định `listen_addresses` điều khiển các interface nhận connection. [PostgreSQL](https://www.postgresql.org/docs/current/runtime-config-connection.html?utm_source=chatgpt.com)

Nếu TCP không hoạt động nhưng local Unix socket connection vẫn chạy:

```bash
sudo -u postgres psql
```

thì scope đã thu hẹp đáng kể:

```text
database engine likely running
but TCP/network/listen/auth path has problem
```

---

# 22. `pg_hba.conf`

Tìm chính xác file:

```sql
SHOW hba_file;
```

`pg_hba.conf` quyết định host/user/database nào được authenticate bằng phương thức nào.

Nhưng Day 16 L1 principle:

> **Đọc configuration là diagnostic. Thay authentication rule là change.**

Không được thấy:

```text
no pg_hba.conf entry
```

rồi lập tức thêm:

```conf
host all all 0.0.0.0/0 trust
```

Đây là security anti-pattern cực kỳ nghiêm trọng.

---

# 23. Connection capacity

Ứng dụng có thể không kết nối được dù:

```text
PostgreSQL running
port open
password correct
pg_hba correct
```

vì connection slots đã đầy.

Xem limit:

```sql
SHOW max_connections;
```

PostgreSQL 18 mặc định thường là 100, nhưng giá trị thực tế phụ thuộc configuration. Tăng `max_connections` làm PostgreSQL phải allocate thêm một số resources; đây không phải parameter nên tăng tùy tiện. [PostgreSQL](https://www.postgresql.org/docs/current/runtime-config-connection.html?utm_source=chatgpt.com)

Kiểm tra client connections:

```sql
SELECT count(*) AS client_connections
FROM pg_stat_activity
WHERE backend_type = 'client backend';
```

Chi tiết:

```sql
SELECT
    state,
    count(*)
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY state
ORDER BY count(*) DESC;
```

Một query tổng hợp:

```sql
SELECT
    current_setting('max_connections')::integer AS max_connections,

    count(*) FILTER (
        WHERE backend_type = 'client backend'
    ) AS client_connections,

    count(*) FILTER (
        WHERE backend_type = 'client backend'
          AND state = 'active'
    ) AS active,

    count(*) FILTER (
        WHERE backend_type = 'client backend'
          AND state = 'idle'
    ) AS idle,

    count(*) FILTER (
        WHERE backend_type = 'client backend'
          AND state LIKE 'idle in transaction%'
    ) AS idle_in_transaction
FROM pg_stat_activity;
```

---

# 24. Đừng kết luận “nhiều idle connections = vấn đề”

Giả sử:

```text
max_connections = 100

client connections = 70
active             = 5
idle               = 63
idle in transaction= 2
```

70 connections không tự động nghĩa là incident.

Có connection pool thì `idle` sessions là bình thường.

Bạn phải hỏi:

```text
Có gần max_connections không?
Trend tăng hay ổn định?
Connection pool configured thế nào?
Có connection leak?
Idle-in-transaction có lâu không?
Application có error too_many_connections không?
```

Đây là reasoning tốt hơn chỉ nhìn count.

---

# 25. Capacity không chỉ có connections

Capacity của database còn gồm:

```text
filesystem space
filesystem inode
database size
table/index size
I/O pressure
memory
CPU
connection slots
```

Day 16 bắt buộc capacity checks, nên ít nhất phải kiểm tra database-level và OS-level capacity.

---

# 26. Database size

Current database:

```sql
SELECT
    pg_size_pretty(
        pg_database_size(current_database())
    ) AS database_size;
```

Tất cả databases:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

PostgreSQL cung cấp `pg_database_size()` để tính database disk usage và `pg_total_relation_size()` để tính table bao gồm indexes và TOAST. [PostgreSQL](https://www.postgresql.org/docs/18/functions-admin.html?utm_source=chatgpt.com)

---

# 27. Table size

Ví dụ:

```sql
SELECT
    schemaname,
    relname,
    pg_size_pretty(
        pg_total_relation_size(relid)
    ) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC
LIMIT 10;
```

Hữu ích khi:

```text
filesystem suddenly growing
backup suddenly slow
query table huge
unexpected storage consumption
```

---

# 28. Linux filesystem capacity

Trên DB host:

```bash
df -hT
```

Kiểm tra inode:

```bash
df -ih
```

Tại sao inode?

Filesystem có thể:

```text
50% bytes used
100% inodes used
```

và vẫn không tạo được file mới.

Bạn đã học disk/filesystem ở Day 5; Day 16 là lúc tích hợp kiến thức đó với PostgreSQL.

---

# 29. Capacity troubleshooting phải cross-layer

Ví dụ:

```text
Java request slow
       ↓
PostgreSQL query slow
       ↓
wait_event indicates I/O-related wait
       ↓
filesystem nearly full
       ↓
host under I/O pressure
```

Đừng coi database như một hộp tách rời Linux.

PostgreSQL documentation cũng khuyến nghị sử dụng PostgreSQL statistics kết hợp với OS utilities để có bức tranh I/O đầy đủ hơn. [PostgreSQL](https://www.postgresql.org/docs/current/monitoring-stats.html)

---

# 30. EXPLAIN là gì?

Khi PostgreSQL nhận:

```sql
SELECT *
FROM orders
WHERE customer_id = 100;
```

nó không đơn giản “chạy SQL”.

Nó:

```text
Parse
  ↓
Rewrite
  ↓
Planner / Optimizer
  ↓
Select execution plan
  ↓
Executor
```

`EXPLAIN` cho bạn xem plan mà optimizer lựa chọn:

```sql
EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 100;
```

Ví dụ:

```text
Seq Scan on orders
  (cost=0.00..1800.00 rows=100 width=48)
  Filter: (customer_id = 100)
```

---

# 31. Đọc `cost`, `rows`, `width`

Ví dụ:

```text
(cost=0.00..1800.00 rows=100 width=48)
```

ý nghĩa:

```text
0.00       startup cost
1800.00    estimated total cost
100        estimated output rows
48         estimated average row width
```

Một lỗi fresher rất phổ biến:

> `cost=1800` nghĩa là query chạy 1800 ms.

**Sai.**

Cost là planner units, không phải milliseconds. PostgreSQL documentation mô tả cost là arbitrary units dựa trên planner cost parameters. [PostgreSQL](https://www.postgresql.org/docs/current/using-explain.html)

---

# 32. Các scan cơ bản

## Sequential Scan

```text
Seq Scan
```

PostgreSQL quét table pages để tìm row.

Seq Scan **không tự động là xấu**.

Nếu query cần 80% table:

```sql
SELECT *
FROM orders
WHERE status IS NOT NULL;
```

Seq Scan có thể là lựa chọn chính xác.

Nếu table chỉ vài page, index cũng có thể không mang lại lợi ích.

## Index Scan

```text
Index Scan using idx_orders_customer_id
```

PostgreSQL sử dụng index để tìm rows, sau đó lấy heap tuples.

## Index Only Scan

Nếu dữ liệu cần thiết có thể được phục vụ thông qua index và visibility conditions phù hợp:

```text
Index Only Scan
```

## Bitmap scan

Bạn cũng có thể thấy:

```text
Bitmap Index Scan
Bitmap Heap Scan
```

Thường hữu ích khi cần nhiều rows nhưng index vẫn có giá trị.

---

# 33. EXPLAIN không chạy query?

Plain:

```sql
EXPLAIN SELECT ...
```

chỉ tạo/show execution plan.

Nhưng:

```sql
EXPLAIN ANALYZE SELECT ...
```

**thực sự executes query.**

PostgreSQL documentation nói rõ `EXPLAIN ANALYZE` thực thi query để lấy actual row counts và timing. [PostgreSQL](https://www.postgresql.org/docs/current/using-explain.html)

Đây là điểm phải nhớ.

---

# 34. CẢNH BÁO: `EXPLAIN ANALYZE`

Với:

```sql
EXPLAIN ANALYZE
DELETE FROM customers;
```

PostgreSQL **thực sự chạy DELETE**.

Không được nghĩ:

> Có chữ EXPLAIN nên chỉ xem plan.

Sai.

PostgreSQL cảnh báo side effects của statement vẫn xảy ra khi dùng `EXPLAIN ANALYZE`. [PostgreSQL](https://www.postgresql.org/docs/current/using-explain.html)

Vì vậy trong Production, Fresher/L1 nên mặc định:

```text
SELECT → cân nhắc ANALYZE nếu safe/approved

INSERT
UPDATE
DELETE
MERGE
DDL
    → KHÔNG tùy tiện EXPLAIN ANALYZE
```

---

# 35. `EXPLAIN (ANALYZE, BUFFERS)`

Đây là công cụ rất mạnh:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 100;
```

Ví dụ:

```text
Index Scan using idx_orders_customer_id on orders
  (cost=0.42..8.50 rows=3 width=40)
  (actual time=0.030..0.042 rows=4 loops=1)
  Index Cond: (customer_id = 100)
  Buffers: shared hit=5

Planning Time: 0.150 ms
Execution Time: 0.065 ms
```

PostgreSQL 18 tự động bật BUFFERS với ANALYZE; buffer information cho thấy các buffer hits/reads/writes liên quan tới plan nodes. [PostgreSQL](https://www.postgresql.org/docs/current/using-explain.html)

Đối với Day 16, chỉ cần hiểu:

```text
shared hit
```

dữ liệu đã được tìm thấy trong shared buffer.

```text
shared read
```

PostgreSQL phải đưa block vào shared buffers qua read path.

Không nên từ một plan duy nhất suy ra toàn bộ hệ thống storage có vấn đề.

---

# 36. Estimated rows vs Actual rows

Ví dụ:

```text
rows=10
actual rows=50000
```

Planner nghĩ:

```text
10 rows
```

nhưng thực tế:

```text
50,000 rows
```

Đây là discrepancy đáng điều tra.

Có thể liên quan:

```text
statistics
data distribution
predicate selectivity
correlated columns
parameterized queries
```

Nhưng Fresher không nên lập tức:

```sql
CREATE INDEX ...
```

hay chỉnh planner settings.

Nhiệm vụ L1 trước tiên:

```text
collect plan
identify discrepancy
collect table/query context
escalate to DBA/app team if tuning required
```

---

# 37. `loops` — lỗi đọc EXPLAIN phổ biến

Ví dụ:

```text
actual time=0.010..0.020 rows=5 loops=1000
```

Đừng nghĩ node chỉ chạy một lần.

Nó chạy:

```text
1000 loops
```

Một operation nhỏ nhưng chạy hàng nghìn/hàng triệu lần vẫn có thể chiếm thời gian lớn.

Đặc biệt cần chú ý ở nested-loop plans.

---

# 38. Execution Time vs Application Response Time

Suppose:

```text
EXPLAIN ANALYZE Execution Time = 20 ms
```

nhưng API mất:

```text
5 seconds
```

Bạn không được kết luận:

> PostgreSQL báo sai.

API time có thể gồm:

```text
connection acquisition
network latency
application processing
serialization
thread queue
database lock before measurement
multiple SQL statements
downstream services
client-side processing
```

Đây là lý do Operations phải correlate nhiều layers.

---

# 39. Query chậm: quy trình chuẩn Day 16

Khi nhận slow-query incident, đi theo flow sau:

```text
1. Capture exact symptom + timestamp
          ↓
2. Identify affected DB/app/user
          ↓
3. pg_stat_activity
          ↓
4. Is query actually active?
          ↓
5. wait_event_type / wait_event
          ↓
6. pg_blocking_pids()
          ↓
7. transaction/session age
          ↓
8. capacity
          ↓
9. database logs
          ↓
10. EXPLAIN
          ↓
11. EXPLAIN ANALYZE only if safe
          ↓
12. root-cause hypothesis
          ↓
13. L1 action or escalation
```

Thứ tự này quan trọng.

---

# 40. L1 remediation boundaries

Day 16 không chỉ hỏi:

> “Bạn biết fix không?”

Nó hỏi quan trọng hơn:

> **“Bạn có được phép fix không?”**

### Mức rủi ro

| Action                                             | L1 view                                     |
| -------------------------------------------------- | ------------------------------------------- |
| `SELECT` từ `pg_stat_activity`                     | Thường safe/read-only                       |
| `SELECT` từ `pg_locks`                             | Thường safe/read-only                       |
| `pg_blocking_pids()`                               | Diagnostic                                  |
| `SHOW ...`                                         | Safe/read-only                              |
| `EXPLAIN SELECT ...`                               | Thường low risk                             |
| `df -hT`, `ss`, logs                               | Diagnostic                                  |
| Rollback transaction của chính lab/session của bạn | Safe trong lab                              |
| `pg_cancel_backend()`                              | Có operational impact; cần runbook/approval |
| `pg_terminate_backend()`                           | Rủi ro cao hơn                              |
| `CREATE INDEX`                                     | Schema/performance/locking impact           |
| `ALTER SYSTEM`                                     | Configuration change                        |
| tăng `max_connections`                             | Configuration/resource change               |
| thay `pg_hba.conf`                                 | Security/access change                      |
| restart PostgreSQL                                 | Availability impact                         |
| kill Linux `postgres` PID bằng `kill -9`           | Không phải L1 routine remediation           |

---

# 41. `pg_cancel_backend()` vs `pg_terminate_backend()`

## Cancel current query

```sql
SELECT pg_cancel_backend(<pid>);
```

Nó yêu cầu backend cancel **current query**.

PostgreSQL mô tả `pg_cancel_backend()` là hủy query hiện tại của session đó. [PostgreSQL](https://www.postgresql.org/docs/18/functions-admin.html?utm_source=chatgpt.com)

Nhưng chú ý trường hợp:

```text
Session:
BEGIN;
UPDATE ...;
-- UPDATE finished
-- session now idle in transaction
```

Nếu session hiện không chạy query nhưng vẫn giữ transaction open, cancel query không giải quyết root cause.

---

## Terminate session

```sql
SELECT pg_terminate_backend(<pid>);
```

Đây là hành động mạnh hơn: terminate session/backend. PostgreSQL xác định chức năng này terminate backend tương ứng. [PostgreSQL](https://www.postgresql.org/docs/18/functions-admin.html?utm_source=chatgpt.com)

Nó có thể:

```text
disconnect application
rollback uncommitted work
cause retries
cause user-visible error
affect dependent workflow
```

Vì vậy trong production:

> **“Tôi đã xác định blocker” không đồng nghĩa “tôi được phép kill blocker”.**

Đây chính là L1 escalation boundary.

---

# 42. LAB DAY 16 — Slow query caused by lock

Đây là lab chính, bám rất sát Assignment của syllabus:

> Diagnose a connection or slow-query scenario, collect evidence, and decide whether to remediate or escalate. 00_MASTER_SYLLABUS_Java_Fresher…

Chúng ta sẽ tạo:

```text
Session A → giữ transaction + lock
Session B → bị block và trông giống slow query
Session C → Operations diagnostic session
```

## Cảnh báo lab

Chỉ làm trong:

```text
day16_lab
local VM
lab PostgreSQL
non-production environment
```

Không inject blocking vào production.

---

# 43. Lab prerequisites

Bạn cần PostgreSQL từ Day 15 và có quyền tạo database/table trong lab.

Kiểm tra:

```bash
psql --version
```

Server:

```sql
SELECT version();
```

Health:

```bash
pg_isready
```

**Expected invariant:** PostgreSQL accepting connections.

---

# 44. Tạo lab database

Với native Linux installation, tùy setup:

```bash
sudo -u postgres psql
```

Sau đó:

```sql
CREATE DATABASE day16_lab;
```

Kết nối:

```sql
\c day16_lab
```

Nếu database đã tồn tại từ lần lab trước, không tạo lại.

---

# 45. Tạo test table

```sql
CREATE TABLE orders (
    id          bigint PRIMARY KEY,
    customer_id integer NOT NULL,
    status      text NOT NULL,
    amount      numeric(12,2) NOT NULL,
    created_at  timestamptz NOT NULL DEFAULT now()
);
```

Insert sample data:

```sql
INSERT INTO orders (
    id,
    customer_id,
    status,
    amount
)
SELECT
    g,
    (g % 10000) + 1,
    CASE
        WHEN g % 3 = 0 THEN 'PAID'
        WHEN g % 3 = 1 THEN 'NEW'
        ELSE 'PROCESSING'
    END,
    ((g % 50000) + 100) / 100.0
FROM generate_series(1, 200000) AS g;
```

Sau đó:

```sql
ANALYZE orders;
```

Verify:

```sql
SELECT count(*)
FROM orders;
```

**Expected invariant:**

```text
200000
```

---

# 46. Baseline capacity evidence

Trước khi inject failure:

```sql
SELECT
    pg_size_pretty(
        pg_database_size(current_database())
    ) AS db_size;
```

Connection state:

```sql
SELECT
    state,
    count(*)
FROM pg_stat_activity
WHERE datname = 'day16_lab'
GROUP BY state;
```

Linux:

```bash
df -hT
df -ih
```

Lưu evidence.

Mục đích: biết system trông như thế nào **trước failure**.

---

# 47. Mở Session A — blocker

Terminal 1:

```bash
psql day16_lab
```

Trong Session A:

```sql
SELECT pg_backend_pid();
```

Ghi PID lại.

Ví dụ output:

```text
 pg_backend_pid
----------------
 12001
```

**Đây chỉ là Example output, PID của bạn sẽ khác.**

Bây giờ:

```sql
BEGIN;

UPDATE orders
SET status = 'PROCESSING'
WHERE id = 1;
```

Verify trong chính transaction:

```sql
SELECT id, status
FROM orders
WHERE id = 1;
```

Không commit.

Không rollback.

Giữ terminal này mở.

---

# 48. Session A hiện đang làm gì?

Sau khi `UPDATE` kết thúc, terminal chờ command mới.

Tức là session có khả năng ở:

```text
idle in transaction
```

nhưng transaction vẫn mở.

Đây là key concept.

Query không còn “running”, nhưng lock từ transaction chưa nhất thiết được giải phóng.

---

# 49. Mở Session B — blocked query

Terminal 2:

```bash
psql day16_lab
```

Để lab không chờ vô hạn:

```sql
SET statement_timeout = '60s';
```

Bây giờ:

```sql
UPDATE orders
SET status = 'PAID'
WHERE id = 1;
```

Terminal này sẽ chờ.

**Đừng đóng nó.**

Đây là symptom:

```text
UPDATE appears slow/hung
```

---

# 50. Mở Session C — Operations

Terminal 3:

```bash
psql day16_lab
```

Đây là diagnostic session.

Bắt đầu bằng activity:

```sql
SELECT
    pid,
    usename,
    state,
    wait_event_type,
    wait_event,
    now() - xact_start AS xact_age,
    now() - query_start AS query_age,
    pg_blocking_pids(pid) AS blocking_pids,
    LEFT(query, 120) AS query
FROM pg_stat_activity
WHERE datname = 'day16_lab'
  AND pid <> pg_backend_pid()
ORDER BY pid;
```

Bạn nên thấy logic tương tự:

```text
PID 12001
state = idle in transaction
blocking_pids = {}

PID 12042
state = active
wait_event_type = Lock
blocking_pids = {12001}
```

**Expected invariant:**

```text
blocked PID có blocker PID của Session A.
```

PID cụ thể không phải invariant.

---

# 51. Quan sát locks

Session C:

```sql
SELECT
    a.pid,
    a.state,
    l.locktype,
    l.mode,
    l.granted,
    l.waitstart,
    LEFT(a.query, 100) AS query
FROM pg_locks l
JOIN pg_stat_activity a
    ON a.pid = l.pid
WHERE a.datname = 'day16_lab'
ORDER BY a.pid, l.granted;
```

Tìm:

```text
granted = false
```

ở waiting process.

Không cần cố hiểu mọi lock row ngay Day 16.

Mục tiêu:

```text
identify waiter
identify blocker
prove relationship
```

---

# 52. Evidence-driven conclusion

Đến thời điểm này bạn có thể viết:

```text
Symptom:
UPDATE on orders.id=1 did not complete.

Scope:
Database day16_lab, one session reproduced.

Evidence:
Blocked PID: 12042
State: active
Wait type: Lock
Blocker PID: 12001

Blocker:
State: idle in transaction
Transaction remains open.

Hypothesis:
PID 12042 waits for a lock held by the open
transaction of PID 12001.
```

Đây là conclusion có evidence.

Không phải:

```text
Probably DB slow.
```

---

# 53. Remediation an toàn trong lab

Quay lại Session A.

Chạy:

```sql
ROLLBACK;
```

Ngay sau đó Session B sẽ có thể tiếp tục.

Verify Session B hoàn tất.

---

# 54. Post-remediation validation

Session C:

```sql
SELECT
    pid,
    state,
    wait_event_type,
    wait_event,
    pg_blocking_pids(pid)
FROM pg_stat_activity
WHERE datname = 'day16_lab';
```

**Expected invariant:**

Không còn Session B đang bị Session A block.

Kiểm tra row:

```sql
SELECT id, status
FROM orders
WHERE id = 1;
```

Nếu Session B đã execute và autocommit:

```text
status = PAID
```

---

# 55. Root cause lab

Root cause không phải:

```text
UPDATE query inefficient
```

mà là:

```text
A transaction remained open while holding a
conflicting lock, causing another transaction
to wait.
```

Đây là distinction cực kỳ quan trọng.

Nếu bạn chạy:

```sql
EXPLAIN UPDATE ...
```

trước khi kiểm tra locks, bạn có thể đi sai hoàn toàn.

---

# 56. Production decision

Giả sử đây là production và blocker là:

```text
PID 12001
user = payment_service
client = production Java application
transaction age = 25 minutes
```

L1 **không nên tự ý**:

```sql
SELECT pg_terminate_backend(12001);
```

nếu chưa có authority.

Bạn cần xem:

```text
runbook
incident severity
application ownership
transaction business impact
change/incident approval
retry behavior
rollback implications
```

Nếu ngoài L1 authority:

```text
ESCALATE
```

nhưng escalation phải có evidence.

---

# 57. Evidence tốt cho escalation

Ví dụ:

```text
Incident:
Order update requests timing out.

Observed:
2026-10-12 17:xx ICT.

Database:
day16_lab / production equivalent.

Waiting PID:
12042

Waiting state:
active

Wait:
Lock / transactionid

Blocking PID:
12001

Blocking session state:
idle in transaction

Blocking transaction age:
25m13s

Application:
payment-service

Action taken:
Read-only diagnostics only.
No session termination performed.

Reason for escalation:
Terminating blocker can rollback an application
transaction and exceeds L1 authority.

Recommended next decision:
Application/DB owner to determine whether the
blocking transaction may safely be terminated.
```

Đây là professional operations evidence.

---

# 58. LAB phụ — EXPLAIN slow-query analysis

Giờ dùng table `orders` vừa tạo.

Check index:

```sql
\d orders
```

Bạn mới có primary key trên:

```text
id
```

Không có index trên:

```text
customer_id
```

Chạy:

```sql
EXPLAIN
SELECT *
FROM orders
WHERE customer_id = 500;
```

Bạn có thể thấy:

```text
Seq Scan on orders
```

Chi tiết chính xác phụ thuộc version/data/statistics.

**Expected invariant không phải “phải là Seq Scan” tuyệt đối**, vì planner được quyền chọn plan.

Với dataset này, Seq Scan là kết quả thường gặp khi chưa có index thích hợp.

---

# 59. EXPLAIN ANALYZE cho SELECT

Trong lab:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 500;
```

Quan sát:

```text
plan node
estimated rows
actual rows
rows removed
buffers
planning time
execution time
```

Ví dụ conceptually:

```text
Seq Scan on orders
  Filter: (customer_id = 500)
  Rows Removed by Filter: ...
  Buffers: ...
Planning Time: ...
Execution Time: ...
```

Không cần kết quả của bạn giống từng số.

---

# 60. Tạo index — LAB ONLY

**Đây là schema change. Không chuyển hành động này sang production nếu chưa có change/DBA approval.**

Trong lab:

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

Refresh planner statistics nếu cần:

```sql
ANALYZE orders;
```

Sau đó:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM orders
WHERE customer_id = 500;
```

Có khả năng bạn sẽ thấy:

```text
Bitmap Heap Scan
```

hoặc:

```text
Index Scan
```

tùy dữ liệu/planner/version.

Điều bạn cần hiểu không phải “index luôn thắng”.

Mà là:

```text
before:
planner considered full/large scan cheapest

after:
an additional access path exists

planner:
chooses plan according to estimated cost
```

---

# 61. Tại sao không được “thấy Seq Scan → tạo index”?

Ví dụ:

```sql
SELECT *
FROM orders
WHERE status IN ('NEW', 'PAID', 'PROCESSING');
```

Nếu gần như toàn table match:

```text
Index Scan
```

có thể đắt hơn:

```text
Seq Scan
```

vì phải đọc rất nhiều index entries rồi rất nhiều heap pages.

Do đó:

> **Seq Scan là một plan node, không phải error code.**

---

# 62. Cleanup lab index

Nếu muốn trả table về trạng thái trước phần tuning:

```sql
DROP INDEX idx_orders_customer_id;
```

Lab only.

Nếu bạn tiếp tục dùng table cho practice, có thể giữ lại và ghi vào evidence rằng schema đã thay đổi.

---

# 63. Connection failure lab — phương pháp chẩn đoán

Bạn cũng nên luyện scenario còn lại của Assignment.

Giả sử application báo:

```text
Connection refused
```

Không cần phá production.

Trong lab bạn có thể intentionally dùng port sai:

```bash
psql \
  -h 127.0.0.1 \
  -p 5433 \
  -U postgres \
  -d day16_lab
```

nếu PostgreSQL thật đang ở 5432.

Sau đó compare:

```bash
pg_isready -h 127.0.0.1 -p 5433
```

với:

```bash
pg_isready -h 127.0.0.1 -p 5432
```

Check:

```bash
ss -ltnp | grep postgres
```

hoặc:

```bash
ss -ltnp | grep ':5432'
```

Trong PostgreSQL:

```sql
SHOW port;
SHOW listen_addresses;
```

Conclusion phải là:

```text
Client attempted the wrong TCP port.
```

Không phải:

```text
PostgreSQL service is broken.
```

---

# 64. Phân loại lỗi connection

Mental model hữu ích:

| Symptom                        | Layer cần nghĩ tới                         |
| ------------------------------ | ------------------------------------------ |
| hostname cannot resolve        | DNS                                        |
| timeout                        | network/firewall/routing/listen            |
| connection refused             | no listener / wrong IP/port / service down |
| `no pg_hba.conf entry`         | authentication policy                      |
| password authentication failed | credential/auth                            |
| database does not exist        | DB configuration                           |
| role does not exist            | role/configuration                         |
| too many connections           | connection capacity                        |
| app timeout nhưng psql works   | application pool/config/network path khác  |

Không sửa tất cả layer cùng lúc.

Một troubleshooting tốt thay đổi **một hypothesis một lần**.

---

# 65. Anti-pattern: restart PostgreSQL

Một Fresher thường nghĩ:

```text
App cannot connect
→ restart database
```

Đây là anti-pattern.

Restart có thể:

```text
drop every connection
interrupt transactions
cause application errors
hide root cause
destroy transient diagnostic evidence
create larger outage
```

Trước restart cần ít nhất:

```text
scope
service state
logs
listener
capacity
approval
rollback/recovery plan
```

Nếu không được authority:

```text
escalate
```

---

# 66. Anti-pattern: kill blocker ngay

Bạn nhìn thấy:

```text
pg_blocking_pids() = {12001}
```

và chạy:

```sql
SELECT pg_terminate_backend(12001);
```

Đó không phải troubleshooting tốt.

Bạn mới biết:

```text
who blocks whom
```

Bạn chưa biết:

```text
business purpose
transaction impact
whether rollback is safe
whether application retries safely
whether session belongs to critical batch
```

Identification ≠ authorization.

---

# 67. Anti-pattern: query quá nhiều dữ liệu trong diagnostic

Ví dụ:

```sql
SELECT *
FROM gigantic_orders;
```

chỉ để “test database”.

Bạn có thể tự tạo thêm load.

Thay bằng:

```sql
SELECT 1;
```

nếu chỉ cần basic query execution.

Hoặc:

```sql
SELECT count(*)
...
```

cũng phải cân nhắc table size vì `count(*)` trên table rất lớn có thể không hề rẻ.

---

# 68. Anti-pattern: production `EXPLAIN ANALYZE` vô điều kiện

Nhớ:

```text
EXPLAIN
    planner only

EXPLAIN ANALYZE
    executes
```

Đặc biệt với:

```text
UPDATE
DELETE
INSERT
MERGE
```

rủi ro tăng mạnh.

Ngay cả expensive SELECT cũng có thể gây load.

---

# 69. Anti-pattern: tăng `max_connections`

Incident:

```text
too many connections
```

Fix nhanh:

```text
100 → 1000
```

có thể khiến hệ thống tệ hơn.

Bạn cần tìm:

```text
connection pool size
number of app instances
connection leak
idle transaction
long-running requests
reserved slots
resource capacity
```

`max_connections` là capacity boundary, không phải nút “make database faster”.

---

# 70. Anti-pattern: bật `log_statement = all`

Trong production high-throughput:

```conf
log_statement = 'all'
```

có thể tạo:

```text
enormous volume
performance overhead
disk pressure
sensitive data exposure
```

Logging thay đổi cũng cần operational discipline.

---

# 71. Safe query toolkit Day 16

Đây là nhóm command bạn nên thực sự thuộc.

```sql
-- Server/version
SELECT version();

-- Connection identity
SELECT
    current_database(),
    current_user,
    inet_server_addr(),
    inet_server_port();

-- Server configuration locations
SHOW config_file;
SHOW hba_file;
SHOW data_directory;

-- Port/listen
SHOW port;
SHOW listen_addresses;

-- Capacity
SHOW max_connections;

-- Session activity
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    now() - xact_start AS xact_age,
    now() - query_start AS query_age,
    LEFT(query, 150) AS query
FROM pg_stat_activity
WHERE backend_type = 'client backend';

-- Blockers
SELECT
    pid,
    pg_blocking_pids(pid)
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- Database size
SELECT
    pg_size_pretty(
        pg_database_size(current_database())
    );

-- Logs
SHOW log_destination;
SHOW logging_collector;
SHOW log_directory;

-- Plan without execution
EXPLAIN
SELECT ...;
```

Đây là nhóm command đáng luyện đến mức không cần tra cứu nhiều.

---

# 72. Permission considerations

Thông tin trong `pg_stat_activity` có security restriction.

Ordinary users nhìn đầy đủ session của mình nhưng một số information của session khác có thể bị restricted; các role như `pg_read_all_stats` có quyền quan sát rộng hơn. [PostgreSQL](https://www.postgresql.org/docs/current/monitoring-stats.htm?utm_source=chatgpt.com)

Operational principle:

```text
Do not make application account superuser
just to troubleshoot.
```

Có thể có một operations/monitoring role được cấp privilege thích hợp theo policy.

Least privilege vẫn áp dụng cho troubleshooting.

---

# 73. `pg_monitor` / monitoring privileges — Supplementary

**Supplementary / Beyond the explicit syllabus**

Trong môi trường thật, monitoring account thường không cần:

```text
SUPERUSER
```

PostgreSQL có predefined monitoring roles.

Thiết kế hợp lý có thể là:

```text
app_user
    → application DML only

monitor_user
    → monitoring/statistics privileges

dba/admin
    → elevated administrative operations
```

Tách trách nhiệm giúp hạn chế blast radius.

---

# 74. Failure Injection reasoning drill

Giả sử bạn thấy:

```text
PID 101
state: active
query age: 45 sec
wait_event_type: Lock
blocking_pids: {55}
```

PID 55:

```text
state: idle in transaction
xact_age: 15 min
```

Lập luận đúng:

```text
1. PID 101 appears slow.
2. But it is waiting on Lock.
3. PostgreSQL identifies PID 55 as blocker.
4. PID 55 has an old open transaction.
5. The strong hypothesis is lock blocking,
   not primarily expensive query execution.
6. Need ownership/business-impact evaluation
   before terminating PID 55.
```

Lập luận sai:

```text
Query > 45 seconds.
Therefore create an index.
```

---

# 75. Troubleshooting scenario: DB disk 96%

Giả sử:

```bash
df -hT
```

trả:

```text
/dev/xvdf1  100G  96G  4G  96%
```

Database vẫn đang chạy.

L1 không nên:

```bash
rm -rf /var/lib/postgresql/...
```

hoặc xóa `pg_wal` thủ công.

Đây là cực kỳ nguy hiểm.

Bạn cần:

```text
identify filesystem
identify largest consumers
check DB/table/log sizes
determine growth source
preserve database files
escalate if cleanup affects PostgreSQL internals
```

Không bao giờ xóa file trực tiếp bên trong PostgreSQL data directory như một cách giải phóng disk thông thường.

---

# 76. Operations rollback thinking

Mọi remediation nên có câu hỏi:

```text
Nếu thao tác này sai thì quay lại thế nào?
```

Ví dụ tạo index:

```text
CREATE INDEX ...
```

Rollback:

```text
DROP INDEX ...
```

Nhưng rollback itself cũng là change.

Change `postgresql.conf`:

```text
backup/record old value
change one parameter
validate syntax/behavior
reload/restart only as required
validate
restore old value if necessary
```

Terminate session:

```text
không có "undo terminate"
```

Transaction đã rollback thì không thể “restore session” chỉ bằng một command.

Do đó cần thận trọng hơn.

---

# 77. Một Incident Workflow hoàn chỉnh

Bạn có thể dùng template này cho Day 16:

```text
INCIDENT
=======

Reported symptom:
<exact error/slow behavior>

Expected behavior:
<what should normally happen>

Timestamp/timezone:
<time>

Scope:
<one user / one app / one database / all users>

Recent changes:
<deployment/config/restart/schema/network>

DATABASE HEALTH
===============
PostgreSQL service:
<state>

Listening port:
<evidence>

Connection test:
<evidence>

SESSION EVIDENCE
================
Active sessions:
<evidence>

Long transactions:
<evidence>

Wait events:
<evidence>

Blocking PIDs:
<evidence>

CAPACITY
========
Connections:
<n>/<max>

Filesystem:
<usage>

Database size:
<size>

LOG EVIDENCE
============
Relevant timestamps:
<evidence>

SQLSTATE/error:
<evidence>

QUERY PLAN
==========
EXPLAIN:
<evidence>

ANALYSIS
========
Strongest hypothesis:
<reasoning>

Evidence supporting it:
<evidence>

REMEDIATION DECISION
====================
L1-safe:
<action>

or

Escalate:
<team/reason>

VALIDATION
==========
Expected behavior restored:
<yes/no + evidence>
```

Nếu bạn làm Day 16 lab, mình khuyến nghị lưu evidence theo format này.

---

# 78. Phân biệt Evidence, Hypothesis và Root Cause

Đây là kỹ năng fresher thường thiếu.

### Evidence

```text
pg_stat_activity shows PID 8123
wait_event_type = Lock.
```

Fact.

### Hypothesis

```text
The request is slow because PID 8123
is waiting for a database lock.
```

Inference dựa trên evidence.

### Stronger evidence

```text
pg_blocking_pids(8123) = {7984}
```

### Root cause

Sau investigation:

```text
Application transaction PID 7984 remained
idle in transaction because an error path failed
to commit/rollback the transaction.
```

Đó mới là root cause sâu hơn.

L1 có thể tìm được:

```text
technical immediate cause
```

nhưng application-code root cause có thể cần escalation sang developer.

---

# 79. L1 vs DBA vs Developer

Một scenario:

```text
blocked request
    ↓
idle-in-transaction blocker
```

L1:

```text
detect
collect evidence
identify blocker
validate scope
execute approved remediation
escalate
```

DBA có thể:

```text
investigate transaction/database behavior
review DB configuration
query/schema/index tuning
advanced lock analysis
```

Developer có thể phải sửa:

```text
missing commit/rollback
transaction boundary
connection leak
bad JDBC pool config
SQL/query design
```

Operations troubleshooting không có nghĩa Operations phải tự sửa source code.

---

# 80. Liên hệ với Java/JDBC từ Day 12–14

Day 16 phải nối lại Java layer.

Ví dụ:

```text
HikariCP maximumPoolSize = 20
```

và có:

```text
10 Java instances
```

Potential pool capacity:

```text
20 × 10 = 200 connections
```

Nếu database:

```text
max_connections = 100
```

thì có architecture mismatch tiềm năng.

Đây chỉ là ví dụ; thực tế còn reserved slots và các client khác.

Điểm cần học:

```text
DB connection capacity
≠ chỉ là PostgreSQL setting.

Nó là interaction giữa:
application instances
× connection pool
× database capacity
```

---

# 81. Troubleshooting integrated stack

Từ nay khi Java app báo database issue, hãy nghĩ:

```text
Client request
     ↓
Java thread
     ↓
Connection pool
     ↓
DNS
     ↓
Network
     ↓
Security/firewall
     ↓
PostgreSQL listener
     ↓
pg_hba/authentication
     ↓
Connection/session
     ↓
Transaction
     ↓
Lock/wait
     ↓
Query plan
     ↓
Table/index
     ↓
Filesystem/I/O
```

Day 20 sẽ đòi hỏi chính kiểu integrated reasoning này.

---

# 82. Exam Focus — lý thuyết

Bạn phải trả lời chắc những vấn đề sau:

| Chủ đề                 | Phải giải thích được                            |
| ---------------------- | ----------------------------------------------- |
| `pg_stat_activity`     | Activity/session diagnostics                    |
| `active`               | Query executing                                 |
| `idle`                 | Session waiting for command                     |
| `idle in transaction`  | Open transaction currently idle                 |
| `wait_event_type`      | Backend đang chờ loại resource/event nào        |
| `pg_locks`             | Locks held/requested                            |
| `pg_blocking_pids()`   | Ai block session cụ thể                         |
| `max_connections`      | Connection capacity boundary                    |
| `pg_database_size()`   | Database disk usage                             |
| `EXPLAIN`              | Estimated execution plan                        |
| `EXPLAIN ANALYZE`      | Actually executes query                         |
| `cost`                 | Planner units, không phải ms                    |
| SQLSTATE               | Machine-readable error classification           |
| `pg_cancel_backend`    | Cancel current query                            |
| `pg_terminate_backend` | Terminate backend/session                       |
| L1 boundary            | Diagnostic authority khác remediation authority |

---

# 83. Exam Focus — practical

Nếu practical exam cho:

```text
Java application database request slow
```

một response tốt có thể là:

```bash
# Host
df -hT
df -ih
ss -ltnp
```

```sql
SELECT version();

SHOW max_connections;

SELECT
    pid,
    usename,
    application_name,
    state,
    wait_event_type,
    wait_event,
    now() - xact_start AS xact_age,
    now() - query_start AS query_age,
    pg_blocking_pids(pid),
    LEFT(query, 150)
FROM pg_stat_activity
WHERE backend_type = 'client backend';

SELECT
    pg_size_pretty(
        pg_database_size(current_database())
    );
```

Sau đó log evidence và, nếu query thật sự execution-bound:

```sql
EXPLAIN
SELECT ...;
```

Không spam random commands.

---

# 84. Knowledge Check

Hãy tự trả lời trước khi xem đáp án trong cột cuối:

| Câu hỏi                                                             | Đáp án                                           |
| ------------------------------------------------------------------- | ------------------------------------------------ |
| `active` có luôn nghĩa query đang dùng CPU không?                   | Không. Nó có thể `active` nhưng waiting          |
| `idle` có tự động là lỗi connection leak không?                     | Không                                            |
| `idle in transaction` tại sao đáng chú ý?                           | Transaction vẫn mở và có thể giữ locks/resources |
| Công cụ tốt để tìm blocker?                                         | `pg_blocking_pids(pid)`                          |
| `EXPLAIN ANALYZE` có execute query không?                           | Có                                               |
| `cost=500` có nghĩa 500 ms?                                         | Không                                            |
| Seq Scan có luôn xấu?                                               | Không                                            |
| Query 30 s có chứng minh execution plan xấu?                        | Không; có thể đang waiting                       |
| Thấy blocker có được tự kill không?                                 | Không, phụ thuộc authority/runbook/impact        |
| Tăng `max_connections` có phải fix chuẩn cho connection exhaustion? | Không; phải tìm nguyên nhân trước                |
| `pg_cancel_backend` và terminate giống nhau?                        | Không                                            |
| Có nên ghi password vào evidence không?                             | Không                                            |

Nếu bạn trả lời được những câu trên **và chứng minh bằng lab**, bạn đang đúng hướng Day 16.

---

# 85. Independent Practical Challenge

Sau khi hoàn thành guided lab, hãy tự làm bài này mà không nhìn lại phần hướng dẫn.

Scenario:

```text
Order Service reports:

"Database operation sometimes hangs for 20–30 seconds."

PostgreSQL service remains running.
CPU usage is not obviously high.
Other users can still connect.
```

Bạn phải cung cấp một evidence pack chứng minh được:

```text
service/database health
current connection utilization
relevant sessions
transaction age
wait event
blocked PID
blocking PID
filesystem capacity
database size
relevant PostgreSQL log evidence
safe EXPLAIN if appropriate
root-cause hypothesis
L1 remediation/escalation decision
post-action validation
```

Không được fix bằng:

```text
restart PostgreSQL
kill random PID
increase max_connections
create random index
change pg_hba.conf
```

trừ khi bạn chứng minh được tại sao cần và có authorization giả lập tương ứng.

---

# 86. Mastery Check Day 16

Để coi Day 16 **thực sự hoàn thành**, chỉ đọc bài này là chưa đủ.

Bạn nên đạt được trạng thái:

```text
[Theory]
Hiểu session/state/wait/lock/capacity/EXPLAIN.

[Hands-on]
Tự tạo blocking scenario.

[Diagnosis]
Tìm đúng waiting PID và blocking PID.

[Evidence]
Chứng minh bằng pg_stat_activity +
pg_blocking_pids + pg_locks.

[Remediation]
Giải phóng transaction an toàn trong lab.

[Validation]
Chứng minh blocked query tiếp tục.

[Performance]
Đọc được EXPLAIN cơ bản.

[Safety]
Biết EXPLAIN ANALYZE executes statement.

[Operations]
Phân biệt diagnostic action với
production-impacting remediation.

[Escalation]
Biết khi nào phải dừng L1 và hand over.
```

Nếu chỉ biết chạy:

```sql
SELECT * FROM pg_stat_activity;
```

nhưng chưa biết **state nào quan trọng, query nào đang wait, ai block ai, hành động nào an toàn và evidence nào cần giữ**, thì Day 16 chưa đạt mastery.

---

# 87. Tóm tắt mental model cần nhớ nhất

Toàn bộ Day 16 có thể được cô đọng thành chuỗi sau:

```text
"Database is slow"
        │
        ▼
Đừng đoán
        │
        ▼
Capture symptom + timestamp
        │
        ▼
Check connectivity / capacity
        │
        ▼
Inspect pg_stat_activity
        │
        ▼
WORKING or WAITING?
        │
        ├──────────── WAITING ────────────┐
        │                                 │
        │                          wait_event
        │                                 │
        │                       pg_blocking_pids()
        │                                 │
        │                           blocker/session
        │
        └──────────── WORKING ────────────┐
                                          │
                                    logs/capacity
                                          │
                                       EXPLAIN
                                          │
                                  safe ANALYZE?
                                          │
                                          ▼
                                Evidence-based cause
                                          │
                                ┌─────────┴─────────┐
                                ▼                   ▼
                          L1-safe fix          ESCALATE
                                │                   │
                                └─────────┬─────────┘
                                          ▼
                                       Validate
                                          │
                                          ▼
                                Preserve evidence
```

**Điểm quan trọng nhất của Day 16:** một administrator giỏi không phải người “kill process nhanh nhất” hay “restart nhanh nhất”, mà là người **xác định đúng failing layer, thu thập đủ evidence, chọn test an toàn nhất, chỉ remediation trong authority của mình, và biết dừng để escalation trước khi biến một incident nhỏ thành outage lớn**.
