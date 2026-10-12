# Day 20 — Failure Injection & Troubleshooting tích hợp

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 20 nằm ngay sau Day 19 — nơi bạn đã build môi trường tích hợp Linux + AWS + Java + PostgreSQL + monitoring — và yêu cầu chính xác:

**Concept/Lecture:** Failure injection và troubleshooting cho **service, port, permission, disk, database connection, alert, deployment rollback**.

**Assignment/Lab:** Điều tra các failure được giao, **restore service trong phạm vi L1**, thu thập logs/evidence, và **escalate những vấn đề vượt L1 scope**. 00_MASTER_SYLLABUS_Java_Fresher…

Đây là ngày cực kỳ quan trọng vì Day 20 chuyển bạn từ trạng thái:

> “Tôi biết cấu hình hệ thống”

sang:

> “Khi hệ thống hỏng, tôi có thể xác định layer lỗi, chứng minh root cause bằng evidence, phục hồi an toàn, validate và escalation đúng cách.”

---

# 2. Mandatory scope và Supplementary scope

| Phần                                                     | Trạng thái                                                                      |
| -------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Service failure                                          | **Mandatory**                                                                   |
| Port failure                                             | **Mandatory**                                                                   |
| Permission failure                                       | **Mandatory**                                                                   |
| Disk/capacity failure                                    | **Mandatory**                                                                   |
| PostgreSQL connection failure                            | **Mandatory**                                                                   |
| Monitoring/alert failure                                 | **Mandatory**                                                                   |
| Deployment failure + rollback                            | **Mandatory**                                                                   |
| Logs/evidence collection                                 | **Mandatory**                                                                   |
| L1 remediation vs escalation                             | **Mandatory**                                                                   |
| Troubleshooting methodology có hệ thống                  | **Supplementary / Beyond explicit syllabus**, nhưng rất quan trọng cho vận hành |
| Blast-radius control                                     | **Supplementary / Beyond explicit syllabus**                                    |
| Incident timeline / evidence pack                        | **Supplementary / Beyond explicit syllabus**                                    |
| Inode exhaustion, process ownership, configuration drift | **Supplementary / Beyond explicit syllabus**                                    |

---

# 3. Môi trường giả định cho Day 20

Day 20 **không phải ngày upgrade version**. Hãy tiếp tục dùng chính môi trường đã build ở Day 19.

Ví dụ xuyên suốt bài:

```text
Client
  |
  v
Linux / EC2
  |
  +--> Tomcat systemd service
  |       |
  |       +--> Java application
  |                |
  |                +--> PostgreSQL
  |
  +--> node_exporter
           |
           v
       Prometheus
           |
           v
      Alertmanager / Grafana
```

Các giá trị ví dụ:

```bash
APP_SERVICE=tomcat
APP_PORT=8080
APP_URL=http://127.0.0.1:8080/myapp/

PGHOST=127.0.0.1
PGPORT=5432
PGDATABASE=appdb
PGUSER=appuser

PROMETHEUS_URL=http://127.0.0.1:9090
```

**Không copy mù các giá trị trên.** Service của bạn có thể tên `tomcat.service`, `tomcat10.service`, `myapp.service`; context path có thể khác `/myapp`; PostgreSQL có thể nằm trên máy khác.

Trước khi bắt đầu:

```bash
systemctl list-units --type=service | grep -Ei 'tomcat|java|postgres|prometheus|node'
ss -lntp
java -version
psql --version
df -hT
```

Về tài liệu hiện hành, PostgreSQL documentation hiện tại là PostgreSQL 18, còn Tomcat documentation hiện hành có Tomcat 11.x; nhưng chúng ta dùng các docs này để xác nhận behavior, **không coi đó là yêu cầu phải nâng môi trường Day 19 lên version mới**. Ví dụ PostgreSQL hiện vẫn mặc định TCP port `5432`, và Tomcat hiện vẫn có `autoDeploy`/`deployOnStartup`. [PostgreSQL](https://www.postgresql.org/docs/current/runtime-config-connection.html?utm_source=chatgpt.com)

---

# 4. Mental Model quan trọng nhất của Day 20

Khi một người dùng báo:

> “Application không vào được.”

Đừng lập tức chạy:

```bash
sudo systemctl restart tomcat
```

Đó là troubleshooting kiểu đoán.

Mental model đúng là:

```text
SYMPTOM
   ↓
EXPECTED BEHAVIOR
   ↓
SCOPE
   ↓
RECENT CHANGES
   ↓
FAILING LAYER
   ↓
EVIDENCE
   ↓
HYPOTHESES
   ↓
SAFEST TEST
   ↓
ROOT CAUSE
   ↓
REMEDIATION
   ↓
VALIDATION
   ↓
EVIDENCE + PREVENTION / ESCALATION
```

Đây sẽ là framework bạn dùng cho **tất cả 7 failure scenario** hôm nay.

---

# 5. Tư duy troubleshooting theo layer

Một request tới Java application thường đi qua:

```text
User / client
     ↓
DNS
     ↓
Network route
     ↓
AWS Security Group / NACL
     ↓
EC2 network interface
     ↓
Linux firewall
     ↓
TCP port
     ↓
Tomcat process
     ↓
Java web application
     ↓
Application configuration
     ↓
PostgreSQL TCP port
     ↓
PostgreSQL authentication
     ↓
Database/schema/query
```

Monitoring lại tạo một flow khác:

```text
node_exporter/app metrics
       ↓
HTTP metrics endpoint
       ↓
Prometheus scrape
       ↓
Time series
       ↓
PromQL expression
       ↓
Alert rule
       ↓
Pending
       ↓
Firing
       ↓
Alertmanager
       ↓
Notification destination
```

Nếu bạn hiểu hai flow này, phần lớn troubleshooting L1 trở thành việc tìm:

> **“Flow bị gãy tại bước nào?”**

---

# 6. Golden rule: Evidence before change

Trước khi restart hoặc sửa config, hãy chụp trạng thái hiện tại.

Một basic evidence snapshot:

```bash
date -Is
hostname
hostname -I
uptime

systemctl status "$APP_SERVICE" --no-pager

ss -lntp

curl -v "$APP_URL"

df -hT
df -i

journalctl -u "$APP_SERVICE" --since "-15 min" --no-pager
```

`journalctl` đọc log từ systemd journal và có thể filter theo systemd unit, vì vậy `journalctl -u <service>` là một nguồn evidence quan trọng cho service troubleshooting. [Freedesktop](https://www.freedesktop.org/software/systemd/man/255/journalctl.html?utm_source=chatgpt.com)

Một habit rất quan trọng:

```bash
mkdir -p ~/day20-evidence
```

Sau đó:

```bash
date -Is > ~/day20-evidence/timestamp.txt

systemctl status "$APP_SERVICE" --no-pager \
  > ~/day20-evidence/app-status.txt 2>&1

journalctl -u "$APP_SERVICE" --since "-30 min" --no-pager \
  > ~/day20-evidence/app-journal.txt

ss -lntp \
  > ~/day20-evidence/listeners.txt

df -hT \
  > ~/day20-evidence/disk.txt

df -i \
  > ~/day20-evidence/inodes.txt
```

Đây chính là tư duy **evidence preservation**.

---

# 7. Failure Scenario 1 — Service Failure

## 7.1 Expected behavior

Giả sử:

```bash
systemctl is-active "$APP_SERVICE"
```

Expected invariant:

```text
active
```

và:

```bash
curl "$APP_URL"
```

phải trả về HTTP response hợp lệ.

---

## 7.2 Failure injection

> ⚠️ Chỉ thực hiện trong lab/non-production. Lệnh dưới đây làm application unavailable.

```bash
sudo systemctl stop "$APP_SERVICE"
```

Không start lại ngay.

Hãy đóng vai L1 operator nhận ticket:

```text
User cannot access application.
```

---

# 8. Điều tra service failure

Bắt đầu từ symptom:

```bash
curl -v "$APP_URL"
```

Có thể thấy:

```text
Connection refused
```

Điều này chưa đủ để kết luận Tomcat chết.

Kiểm tra service:

```bash
systemctl status "$APP_SERVICE" --no-pager
```

Sau đó:

```bash
systemctl is-active "$APP_SERVICE"
systemctl is-failed "$APP_SERVICE"
```

Tiếp tục kiểm tra process:

```bash
ps -ef | grep -i '[t]omcat'
ps -ef | grep '[j]ava'
```

Kiểm tra socket:

```bash
sudo ss -lntp
```

Hoặc cụ thể:

```bash
sudo ss -lntp '( sport = :8080 )'
```

`ss` được thiết kế để inspect socket; `-l` chỉ listener, `-n` giữ địa chỉ/port dạng số, `-t` giới hạn TCP, `-p` cho biết process liên quan nếu đủ privilege. [man7](https://www.man7.org/linux/man-pages/man8/ss.8.html?utm_source=chatgpt.com)

Nếu:

```text
service inactive
process absent
port absent
```

thì chain evidence phù hợp với **service stopped**.

---

# 9. Nếu service ở trạng thái failed

Ví dụ:

```text
Active: failed (Result: exit-code)
```

Không restart ngay.

Đầu tiên:

```bash
systemctl status "$APP_SERVICE" -l --no-pager
```

Sau đó:

```bash
journalctl -u "$APP_SERVICE" -b --no-pager
```

Giới hạn thời gian:

```bash
journalctl -u "$APP_SERVICE" \
  --since "10 minutes ago" \
  --no-pager
```

Kiểm tra definition:

```bash
systemctl cat "$APP_SERVICE"
```

Nếu unit dùng environment file:

```text
EnvironmentFile=/etc/myapp/myapp.env
```

thì kiểm tra file đó tồn tại và permission:

```bash
sudo ls -l /etc/myapp/myapp.env
sudo stat /etc/myapp/myapp.env
```

Không `cat` file chứa password vào ticket hoặc screenshot.

---

# 10. L1 remediation cho service failure

Nếu evidence xác nhận service đơn giản bị stopped và ticket/change cho phép start:

```bash
sudo systemctl start "$APP_SERVICE"
```

Sau đó:

```bash
systemctl is-active "$APP_SERVICE"
systemctl status "$APP_SERVICE" --no-pager

sudo ss -lntp '( sport = :8080 )'

curl -v "$APP_URL"
```

Quan trọng:

```text
systemctl active
```

**không đồng nghĩa application healthy**.

Có thể process chạy nhưng:

```text
HTTP 500
```

vì database chết.

Do đó validation phải vượt qua nhiều layer.

---

# 11. Failure Scenario 2 — Port Failure

Port problem có ít nhất ba nhóm:

```text
No listener
Wrong listener
Listener exists but network path blocked
```

Ba trường hợp này hoàn toàn khác nhau.

---

# 12. Hiểu `Connection refused` và timeout

Nếu:

```bash
curl http://127.0.0.1:8080/
```

trả:

```text
Connection refused
```

thường có nghĩa kernel local biết rằng **không có listener phù hợp**.

Nếu request treo rồi timeout:

```text
Connection timed out
```

hãy nghĩ đến firewall/network/route/SG hoặc packet bị drop.

Đừng tuyệt đối hóa; nhưng đây là heuristic troubleshooting rất có giá trị.

---

# 13. Port investigation workflow

Kiểm tra expected port:

```bash
sudo ss -lntp | grep ':8080'
```

Nếu không thấy gì:

```bash
sudo ss -lntp
```

Có thể phát hiện app đang listen:

```text
*:8081
```

thay vì:

```text
*:8080
```

Tiếp tục:

```bash
curl -v http://127.0.0.1:8081/
```

Nếu `8081` hoạt động nhưng `8080` không:

```text
application/service = running
network stack       = running
actual listener     = 8081
expected listener   = 8080
```

Root cause nhiều khả năng nằm ở **configuration drift / wrong port**.

---

# 14. Tomcat port

Trong standalone Tomcat, HTTP Connector thường nằm trong:

```text
$CATALINA_BASE/conf/server.xml
```

Ví dụ:

```xml
<Connector port="8080"
           protocol="HTTP/1.1"
           connectionTimeout="20000" />
```

Không được giả định `$CATALINA_BASE=/opt/tomcat`.

Tìm trước:

```bash
systemctl cat "$APP_SERVICE"
```

hoặc:

```bash
ps -ef | grep '[o]rg.apache.catalina.startup.Bootstrap'
```

---

# 15. Safe port configuration change

Trước khi sửa:

```bash
sudo cp -a \
  "$CATALINA_BASE/conf/server.xml" \
  "$CATALINA_BASE/conf/server.xml.day20.bak"
```

Kiểm tra hash:

```bash
sha256sum \
  "$CATALINA_BASE/conf/server.xml" \
  "$CATALINA_BASE/conf/server.xml.day20.bak"
```

Sau khi sửa, Tomcat có `configtest` để thử parse configuration trước khi startup. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/api/org/apache/catalina/startup/Catalina.html?utm_source=chatgpt.com)

Ví dụ:

```bash
"$CATALINA_HOME/bin/catalina.sh" configtest
echo $?
```

Chỉ restart khi validation thành công:

```bash
sudo systemctl restart "$APP_SERVICE"
```

Sau đó:

```bash
sudo ss -lntp | grep ':8080'
curl -v "$APP_URL"
```

---

# 16. Port collision

Một dạng khác:

```text
Address already in use
BindException
```

Đừng kill process bừa.

Tìm owner:

```bash
sudo ss -lntp | grep ':8080'
```

Ví dụ:

```text
LISTEN ... users:(("java",pid=2815,fd=52))
```

Sau đó:

```bash
ps -fp 2815
```

Bạn cần trả lời:

```text
PID 2815 là process nào?
Nó thuộc service nào?
Nó có phải process production hợp lệ không?
Ai start nó?
Có recent deployment/change không?
```

`kill -9 2815` không phải troubleshooting.

---

# 17. Failure Scenario 3 — Permission Failure

Linux access không chỉ phụ thuộc vào file cuối cùng.

Ví dụ:

```text
/opt/myapp/config/application.properties
```

Để service user đọc được file, nó cần đủ quyền traverse các parent directory.

Kiểm tra:

```bash
namei -l /opt/myapp/config/application.properties
```

Ví dụ:

```text
f: /opt/myapp/config/application.properties
drwxr-xr-x root   root   /
drwxr-xr-x root   root   opt
drwxr-x--- tomcat tomcat myapp
drwxr-x--- tomcat tomcat config
-rw-r----- tomcat tomcat application.properties
```

`namei -l` cực hữu ích vì nó nhìn cả path.

---

# 18. Các command permission quan trọng

```bash
ls -ld /opt/myapp
ls -ld /opt/myapp/config
ls -l /opt/myapp/config/application.properties

stat /opt/myapp/config/application.properties

id tomcat
```

Quan trọng nhất: test dưới **identity thật của service**:

```bash
sudo -u tomcat test -r /opt/myapp/config/application.properties
echo $?
```

Exit code:

```text
0 = readable
non-zero = test failed
```

Hoặc:

```bash
sudo -u tomcat head \
  /opt/myapp/config/application.properties
```

Chỉ dùng nếu file không chứa secret mà bạn không nên hiển thị.

---

# 19. Owner ≠ permission

Ví dụ:

```text
-rw------- root root application.properties
```

Tomcat chạy user:

```text
tomcat
```

thì rất có thể:

```text
Permission denied
```

Nhưng fix:

```bash
chmod 777 application.properties
```

là **anti-pattern nghiêm trọng**.

`chmod` thay đổi access permission; `chown` thay đổi owner/group. Việc dùng recursive ownership hoặc permission rộng có thể gây security issue và blast radius lớn, đặc biệt với symlink traversal. [GNU](https://www.gnu.org/software/coreutils/manual/coreutils.html?utm_source=chatgpt.com)

---

# 20. Least-privilege remediation

Giả sử baseline đã xác nhận phải là:

```text
owner = root
group = tomcat
mode  = 0640
```

thì:

```bash
sudo chown root:tomcat \
  /opt/myapp/config/application.properties

sudo chmod 0640 \
  /opt/myapp/config/application.properties
```

Validation:

```bash
stat /opt/myapp/config/application.properties

sudo -u tomcat test -r \
  /opt/myapp/config/application.properties

echo $?
```

Quan trọng: bạn chỉ được làm vậy khi **baseline/runbook xác nhận đó là permission đúng**.

Nếu bạn không biết permission chuẩn:

> **Không tự đoán. Escalate.**

---

# 21. Safe permission failure injection

Để không phá actual application, ta có thể tạo isolated lab:

```bash
sudo mkdir -p /srv/day20-perm
sudo touch /srv/day20-perm/app.conf
```

Giả sử service account của bạn là `tomcat`.

Baseline:

```bash
sudo chown root:tomcat /srv/day20-perm/app.conf
sudo chmod 0640 /srv/day20-perm/app.conf

sudo -u tomcat test -r /srv/day20-perm/app.conf
echo $?
```

Expected:

```text
0
```

Inject:

```bash
sudo chmod 0600 /srv/day20-perm/app.conf
```

Test:

```bash
sudo -u tomcat test -r /srv/day20-perm/app.conf
echo $?
```

Expected:

```text
1
```

Evidence:

```bash
stat /srv/day20-perm/app.conf
id tomcat
```

Recover:

```bash
sudo chmod 0640 /srv/day20-perm/app.conf
```

Validate:

```bash
sudo -u tomcat test -r /srv/day20-perm/app.conf
echo $?
```

---

# 22. Failure Scenario 4 — Disk Failure

“Disk full” thực tế có thể là:

```text
filesystem blocks exhausted
inode exhausted
wrong filesystem full
log explosion
deleted-but-open file
mount missing
filesystem read-only
quota
```

Vì vậy:

```bash
df -h
```

không đủ.

GNU Coreutils cung cấp `df` để report filesystem space usage và `du` để estimate usage của files/directories. [GNU](https://www.gnu.org/software/coreutils/manual/html_node/File-space-usage.html?utm_source=chatgpt.com)

---

# 23. Disk investigation

Đầu tiên:

```bash
df -hT
```

Ví dụ:

```text
Filesystem     Type  Size Used Avail Use% Mounted on
/dev/xvda1     xfs    20G  20G  100M  100% /
```

Sau đó inode:

```bash
df -i
```

Có thể:

```text
IUse%=100%
```

dù:

```text
Use%=40%
```

Khi inode hết, application vẫn có thể gặp:

```text
No space left on device
```

mặc dù GB còn trống.

---

# 24. Tìm directory tiêu thụ disk

Nếu `/var` đầy:

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
```

Sau đó drill-down:

```bash
sudo du -xhd1 /var/log 2>/dev/null | sort -h
```

`-x` rất hữu ích để tránh đi sang filesystem khác.

Tìm large file:

```bash
sudo find /var/log -xdev \
  -type f \
  -size +100M \
  -printf '%s %p\n' 2>/dev/null \
  | sort -n
```

---

# 25. Đừng “rm log” theo phản xạ

Ví dụ bạn thấy:

```text
/var/log/myapp/app.log = 15 GB
```

Không được ngay lập tức:

```bash
sudo rm -f /var/log/myapp/app.log
```

Bạn phải hỏi:

```text
Log có cần retention không?
Có active writer không?
Logrotate có lỗi không?
Evidence có cần giữ không?
Đây có phải filesystem đúng không?
Có approval để delete không?
```

Ngoài ra nếu process vẫn giữ file descriptor mở, delete pathname chưa chắc giải phóng space ngay.

Có thể kiểm tra:

```bash
sudo lsof +L1
```

nếu `lsof` được cài.

---

# 26. Safe disk-full failure injection

> ⚠️ Không được cố ý fill `/`, `/var`, EBS production hoặc database filesystem.

Ta tạo filesystem riêng bằng `tmpfs`.

```bash
sudo mkdir -p /mnt/day20-full
sudo mount -t tmpfs \
  -o size=50M \
  tmpfs \
  /mnt/day20-full
```

Validate:

```bash
df -hT /mnt/day20-full
```

Inject:

```bash
dd if=/dev/zero \
   of=/mnt/day20-full/fill.bin \
   bs=1M \
   count=60 \
   status=progress
```

Nó sẽ dừng khi filesystem đầy.

Kiểm tra:

```bash
df -hT /mnt/day20-full
```

Thử write:

```bash
echo "test" >> /mnt/day20-full/app.log
echo $?
```

Expected symptom:

```text
No space left on device
```

---

# 27. Recovery disk lab

Evidence trước:

```bash
df -hT /mnt/day20-full
du -h /mnt/day20-full/*
```

Sau đó cleanup file **được biết chắc là failure-injection file**:

```bash
sudo rm /mnt/day20-full/fill.bin
```

Validate:

```bash
df -hT /mnt/day20-full

echo "recovered" \
  | sudo tee /mnt/day20-full/app.log
```

Cleanup hoàn toàn:

```bash
sudo umount /mnt/day20-full
sudo rmdir /mnt/day20-full
```

Đây là một ví dụ rất tốt về:

```text
known object
known purpose
known blast radius
known rollback
```

---

# 28. Failure Scenario 5 — Database Connection Failure

Đây là scenario thường khó nhất đối với fresher vì:

```text
"Database connection failed"
```

có thể nghĩa rất nhiều thứ.

Troubleshooting flow:

```text
Application
    ↓
DNS
    ↓
TCP connection
    ↓
PostgreSQL listener
    ↓
pg_hba.conf
    ↓
authentication
    ↓
database exists
    ↓
role privilege
    ↓
schema/object
```

---

# 29. Tách transport khỏi authentication

Đừng bắt đầu bằng password.

Trước hết:

```bash
pg_isready \
  -h "$PGHOST" \
  -p "$PGPORT"
```

Nếu:

```text
127.0.0.1:5432 - accepting connections
```

thì PostgreSQL endpoint đang nhận connection.

Current PostgreSQL documentation xác định exit code của `pg_isready` là `0` khi accepting connections, `1` khi rejecting, `2` khi không có response và `3` khi không thực hiện được attempt hợp lệ. Ngoài ra, `pg_isready` không cần credential đúng chỉ để kiểm tra server connection status. [PostgreSQL](https://www.postgresql.org/docs/16/app-pg-isready.html?utm_source=chatgpt.com)

Kiểm tra:

```bash
echo $?
```

---

# 30. Interpret `pg_isready`

```text
accepting connections
```

không chứng minh:

```text
appuser password đúng
appdb tồn tại
appuser có CONNECT
schema đúng
app có SELECT/INSERT permission
```

Nó chỉ giúp loại bỏ một số hypothesis.

Đây là nguyên tắc:

> Một test chỉ được kết luận điều mà test đó thực sự kiểm tra.

---

# 31. PostgreSQL listener

Trên database host:

```bash
sudo ss -lntp | grep ':5432'
```

Ví dụ:

```text
127.0.0.1:5432
```

Nếu app chạy trên host khác thì PostgreSQL chỉ listen loopback sẽ không nhận remote TCP connection.

PostgreSQL `listen_addresses` xác định interface/IP nào nhận connection; default là `localhost`, còn `port` default là `5432`. [PostgreSQL](https://www.postgresql.org/docs/current/runtime-config-connection.html?utm_source=chatgpt.com)

Kiểm tra từ SQL:

```sql
SHOW listen_addresses;
SHOW port;
```

---

# 32. `pg_hba.conf`

Sau khi network path và listener đúng, authentication control có thể nằm trong:

```text
pg_hba.conf
```

Current PostgreSQL documentation nhấn mạnh rằng records trong `pg_hba.conf` được xét **theo thứ tự**, nên rule order là significant. [PostgreSQL](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html?utm_source=chatgpt.com)

Ví dụ:

```text
host    appdb    appuser    10.0.1.25/32    scram-sha-256
```

Nhưng không được thêm:

```text
host    all    all    0.0.0.0/0    trust
```

chỉ vì muốn “test cho nhanh”.

Đó là phá security boundary.

---

# 33. Kiểm tra real authentication

Từ application host:

```bash
psql \
  -h "$PGHOST" \
  -p "$PGPORT" \
  -U "$PGUSER" \
  -d "$PGDATABASE"
```

Không đưa password trực tiếp vào command line hoặc screenshot.

Không viết:

```bash
psql postgresql://appuser:SuperSecretPassword@db/appdb
```

vào evidence.

Current libpq supports connection parameters như `host`, `port`, `dbname`, `user`, `password` và password/passfile mechanisms; từ góc độ vận hành, secret phải được giữ ngoài command/evidence khi có thể. [PostgreSQL](https://www.postgresql.org/docs/current/libpq-connect.html?utm_source=chatgpt.com)

---

# 34. Error interpretation

Ví dụ:

```text
connection refused
```

Tập trung:

```text
service
listener
host
port
```

Ví dụ:

```text
no route to host
timeout
```

Tập trung:

```text
route
SG/NACL
firewall
address
```

Ví dụ:

```text
password authentication failed
```

TCP connection đã đi xa hơn rất nhiều.

Tập trung:

```text
user
password
authentication rule
secret drift
```

Ví dụ:

```text
no pg_hba.conf entry
```

Tập trung:

```text
client source IP
database
user
SSL/non-SSL
rule ordering
```

Ví dụ:

```text
database "appdb" does not exist
```

Không phải network issue.

---

# 35. Safe database failure injection

Phương án tốt nhất nếu PostgreSQL là **dedicated lab database**:

```bash
sudo systemctl stop postgresql
```

> Không thực hiện nếu PostgreSQL phục vụ hệ thống khác.

Từ app host:

```bash
pg_isready -h "$PGHOST" -p "$PGPORT"
```

Capture:

```bash
journalctl -u "$APP_SERVICE" \
  --since "-5 min" \
  --no-pager
```

Bạn rất có thể thấy:

```text
connection refused
connection timeout
HikariPool...
PSQLException...
```

tùy application stack.

---

# 36. Database recovery

```bash
sudo systemctl start postgresql
```

Validate DB:

```bash
pg_isready \
  -h "$PGHOST" \
  -p "$PGPORT"
```

Sau đó application:

```bash
curl -v "$APP_URL"
```

Sau đó logs:

```bash
journalctl -u "$APP_SERVICE" \
  --since "-5 min" \
  --no-pager
```

Một số app tự reconnect; một số cần restart tùy connection pool/application behavior.

Không được restart app chỉ vì DB vừa trở lại nếu chưa xác định app thực sự cần restart.

---

# 37. Failure Scenario 6 — Monitoring / Alert Failure

Alert pipeline:

```text
Exporter
   ↓
Scrape
   ↓
Metric
   ↓
PromQL
   ↓
Rule evaluation
   ↓
Pending
   ↓
Firing
   ↓
Alertmanager
   ↓
Receiver
```

Nếu notification không đến thì lỗi có thể ở bất kỳ đoạn nào.

Prometheus chính thức tách alerting thành hai phần: Prometheus evaluate alert rules, còn Alertmanager xử lý aggregation/silencing/inhibition và notification delivery. [prometheus.io](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/?utm_source=chatgpt.com)

---

# 38. Example InstanceDown

Ví dụ rule:

```yaml
groups:
  - name: host-alerts
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Instance {{ $labels.instance }} is down"
```

`for: 2m` nghĩa là condition đúng chưa chắc lập tức `Firing`.

Alert có thể ở:

```text
Inactive
Pending
Firing
```

Prometheus docs xác nhận `for` giữ alert ở pending cho đến khi condition tiếp tục đúng đủ duration. [prometheus.io](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/?utm_source=chatgpt.com)

---

# 39. Validate rule trước khi reload

Đừng sửa YAML rồi restart Prometheus ngay.

Dùng:

```bash
promtool check rules /etc/prometheus/rules/host.yml
```

Expected:

```text
SUCCESS
```

`promtool check rules` là official validation mechanism cho Prometheus rule files. [prometheus.io](https://prometheus.io/docs/prometheus/latest/command-line/promtool/?utm_source=chatgpt.com)

Validate entire config:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

---

# 40. Failure injection — node_exporter down

> Chỉ trên lab host.

Kiểm tra baseline:

```bash
systemctl is-active node_exporter
curl -s http://127.0.0.1:9100/metrics | head
```

Inject:

```bash
sudo systemctl stop node_exporter
```

Sau một vài scrape intervals:

```text
up{job="node",instance="..."} = 0
```

Bạn có thể query Prometheus API hoặc UI.

Nếu có `promtool` mới:

```bash
promtool query instant \
  http://127.0.0.1:9090 \
  'up == 0'
```

---

# 41. Nếu alert không firing

Kiểm tra theo thứ tự:

```text
Target có Down không?
       ↓
Metric up có bằng 0 không?
       ↓
PromQL có match series không?
       ↓
Rule có được load không?
       ↓
Alert Pending hay Inactive?
       ↓
for: duration đã hết chưa?
       ↓
Alert có Firing không?
       ↓
Alertmanager có nhận alert không?
       ↓
Silence/inhibition có áp dụng không?
       ↓
Receiver có lỗi không?
```

Đây là troubleshooting theo pipeline thay vì:

> “Email không tới nên Prometheus hỏng.”

---

# 42. Recovery alert lab

```bash
sudo systemctl start node_exporter
```

Validate:

```bash
systemctl is-active node_exporter

curl -s \
  http://127.0.0.1:9100/metrics \
  | head
```

Prometheus query:

```promql
up
```

Expected eventually:

```text
1
```

Alert:

```text
Firing → resolved/inactive
```

Evidence phải capture cả:

```text
down state
firing state
recovery state
```

---

# 43. Failure Scenario 7 — Failed Deployment & Rollback

Đây là scenario gần nhất với operations thực tế.

Release flow:

```text
Known-good version
      ↓
Change approval
      ↓
Backup/current version recorded
      ↓
Deploy new artifact
      ↓
Technical validation
      ↓
Functional validation
      ↓
Monitoring validation
```

Nếu fail:

```text
STOP further change
      ↓
Collect evidence
      ↓
Rollback decision
      ↓
Restore known-good artifact/config
      ↓
Restart/redeploy safely
      ↓
Validate
      ↓
Document
```

---

# 44. Trước deployment phải biết “good state”

Ví dụ:

```bash
sha256sum myapp.war
```

Record:

```text
version
artifact
SHA256
deployment time
ticket/change ID
configuration version
```

Nếu không biết artifact nào là version trước:

> rollback của bạn chưa được chuẩn bị đúng.

---

# 45. Tomcat deployment behavior

Tomcat có `appBase`; với standard Host configuration, `deployOnStartup=true` và `autoDeploy=true` thường cho phép application trong appBase được deploy/redeploy tự động khi startup hoặc thay đổi. Automatic redeployment cũng có thể ảnh hưởng session state, nên việc thay WAR trực tiếp không phải là thao tác vô hại. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/config/host.html?utm_source=chatgpt.com)

Do đó trong production, hãy theo **approved deployment method của hệ thống**, không copy WAR tùy tiện chỉ vì Tomcat hỗ trợ auto-deployment.

---

# 46. Safe rollback lab setup

> ⚠️ Scenario này làm application unavailable. Chỉ làm trên isolated Day 20 environment.

Xác định WAR thật:

```bash
find "$CATALINA_BASE/webapps" \
  -maxdepth 1 \
  -type f \
  -name '*.war' \
  -ls
```

Giả sử:

```bash
WAR="$CATALINA_BASE/webapps/myapp.war"
```

Kiểm tra:

```bash
realpath "$WAR"
ls -lh "$WAR"
sha256sum "$WAR"
```

Backup:

```bash
sudo cp -a \
  "$WAR" \
  "${WAR}.day20.good"
```

Kiểm tra backup:

```bash
sha256sum \
  "$WAR" \
  "${WAR}.day20.good"
```

Hai checksum phải giống nhau.

---

# 47. Inject broken deployment

Có thể dùng artifact intentionally broken được chuẩn bị cho lab.

Ví dụ:

```bash
printf 'THIS IS NOT A VALID WAR\n' \
  > /tmp/myapp-bad.war
```

Check:

```bash
file /tmp/myapp-bad.war
```

Sau đó theo chính deployment procedure của lab để thay artifact.

Nếu dùng direct WAR deployment trong lab:

```bash
sudo cp /tmp/myapp-bad.war "$WAR"
```

Theo dõi:

```bash
journalctl -u "$APP_SERVICE" -f
```

Có thể thấy deployment errors.

Tomcat Manager/Deployer semantics cũng phân biệt deployment success/failure; modern Tomcat continues to support deployment management operations. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/api/org/apache/catalina/ant/package-summary.html?utm_source=chatgpt.com)

---

# 48. Điều tra failed deployment

Evidence:

```bash
systemctl status "$APP_SERVICE" --no-pager

sudo ss -lntp | grep ':8080'

curl -v "$APP_URL"

journalctl -u "$APP_SERVICE" \
  --since "-10 min" \
  --no-pager

sha256sum "$WAR"
```

Quan sát quan trọng:

Tomcat có thể:

```text
service active
port 8080 listening
```

nhưng:

```text
/myapp returns 404
```

Đây là ví dụ hoàn hảo chứng minh:

> **Service health ≠ application health.**

---

# 49. Rollback

Đầu tiên preserve failed artifact nếu evidence yêu cầu:

```bash
sudo cp -a \
  "$WAR" \
  "${WAR}.failed.$(date +%Y%m%d%H%M%S)"
```

Sau đó restore known-good:

```bash
sudo cp -a \
  "${WAR}.day20.good" \
  "$WAR"
```

Nếu runbook yêu cầu controlled restart:

```bash
sudo systemctl restart "$APP_SERVICE"
```

Không tự động `rm -rf` exploded webapp directory trong production.

Nếu lab setup của bạn dùng unpacked WAR và cần cleanup stale exploded directory, việc đó phải nằm trong known deployment procedure và chỉ làm sau khi đã xác minh **đúng path**.

---

# 50. Validation sau rollback

System:

```bash
systemctl is-active "$APP_SERVICE"
```

Process:

```bash
ps -ef | grep '[j]ava'
```

Port:

```bash
sudo ss -lntp | grep ':8080'
```

HTTP:

```bash
curl -v "$APP_URL"
```

Database:

```bash
pg_isready \
  -h "$PGHOST" \
  -p "$PGPORT"
```

Logs:

```bash
journalctl -u "$APP_SERVICE" \
  --since "-5 min" \
  --no-pager
```

Monitoring:

```promql
up
```

Cuối cùng:

```bash
sha256sum "$WAR"
```

và compare với checksum known-good.

Rollback chưa hoàn thành cho tới khi **service đã được chứng minh healthy**.

---

# 51. Công cụ Day 20 bạn phải thành thạo

| Công cụ                 | Câu hỏi nó trả lời                                        |
| ----------------------- | --------------------------------------------------------- |
| `systemctl status`      | Service đang ở state nào?                                 |
| `systemctl is-active`   | Service có active không?                                  |
| `systemctl cat`         | Unit thực tế chạy config nào?                             |
| `journalctl -u`         | Service đã log gì?                                        |
| `ps`                    | Process có tồn tại không?                                 |
| `ss -lntp`              | Port nào đang listen? Process nào sở hữu?                 |
| `curl -v`               | HTTP/TCP interaction trông thế nào?                       |
| `stat`                  | Owner/mode/timestamp là gì?                               |
| `namei -l`              | Permission trên toàn path ra sao?                         |
| `id`                    | User/group membership là gì?                              |
| `sudo -u`               | Service account có thực sự access được không?             |
| `df -hT`                | Filesystem nào đầy? Type gì?                              |
| `df -i`                 | Có hết inode không?                                       |
| `du`                    | Directory nào đang chiếm space?                           |
| `find`                  | File nào lớn/bất thường?                                  |
| `pg_isready`            | PostgreSQL endpoint có phản hồi không?                    |
| `psql`                  | Authentication/database/query thực tế có hoạt động không? |
| `promtool check rules`  | Alert rule hợp lệ không?                                  |
| `promtool check config` | Prometheus config hợp lệ không?                           |
| `sha256sum`             | Artifact có đúng binary/version không?                    |

Bạn không cần chỉ nhớ syntax.

Bạn phải nhớ:

> **Mỗi command trả lời câu hỏi troubleshooting nào?**

---

# 52. L1 Scope — Bạn được sửa tới đâu?

L1 không có nghĩa là “không được sửa gì”.

L1 nghĩa là:

> Bạn chỉ thực hiện những remediation **đã được phê duyệt, có runbook, có blast radius hiểu được và có rollback**.

Ví dụ L1-safe tùy organization:

```text
Start/restart một approved service
Restore approved known-good configuration
Correct permission về documented baseline
Clean một known temporary lab file
Restore known-good artifact
Run validation
Collect logs
Update ticket
```

Nhưng một fresher không nên tự ý:

```text
change AWS VPC routing
open SG to 0.0.0.0/0
change production database schema
reset unknown database passwords
delete large production data
fsck mounted production filesystem
kill unknown processes
chmod -R 777
chown -R blindly
disable TLS
disable authentication
turn pg_hba.conf into trust
terminate EC2 to "fix" problem
```

---

# 53. Khi nào phải escalate

Ví dụ bạn gặp:

```text
Filesystem corruption
Database corruption
Unknown repeated OOM
Kernel problem
Production EBS failure
Security compromise suspicion
Unknown permission baseline
Credential mismatch requiring secret rotation
Application bug / stack trace requiring code change
Schema incompatibility
No known-good rollback artifact
Rollback itself fails
Alertmanager infrastructure failure outside your ownership
Cross-account/cross-team AWS network problem
```

Bạn không cần chứng minh toàn bộ root cause trước escalation.

Nhưng phải cung cấp enough evidence để L2/L3 tiếp tục mà không bắt đầu lại từ zero.

---

# 54. Escalation tốt trông như thế nào?

Không viết:

```text
App broken. Please check.
```

Viết kiểu:

```text
Symptom:
HTTP /myapp returns 500 since 14:07.

Scope:
Only application myapp affected. Host reachable.
Tomcat is active.

Evidence:
- systemctl: active
- port 8080 listening by Java PID 3184
- local curl reaches Tomcat but returns HTTP 500
- pg_isready db01:5432 = accepting connections
- application log shows authentication failure for appuser
- issue began immediately after change CHG-1042

L1 actions:
No credential changed.
No DB configuration modified.
Restart not performed because service is already healthy at container layer.

Current hypothesis:
Application DB credential differs from current DB secret.

Request:
Database/application owner to validate approved secret version.

Evidence attached:
app-status.txt
app-journal-redacted.txt
pg-isready.txt
timeline.txt
```

Đó là operational maturity.

---

# 55. Common troubleshooting anti-patterns

| Anti-pattern                            | Vì sao nguy hiểm                           |
| --------------------------------------- | ------------------------------------------ |
| Restart trước khi lấy evidence          | Xóa/mất transient state                    |
| Thấy port lỗi rồi mở firewall           | Có thể chẳng có listener                   |
| `chmod 777`                             | Phá least privilege                        |
| `chown -R`                              | Blast radius cực lớn                       |
| Xóa log lớn ngay                        | Mất evidence/retention                     |
| Kill PID lạ                             | Có thể hạ system khác                      |
| Thấy DB error → reset password          | Có thể làm credential drift tệ hơn         |
| `pg_hba.conf trust`                     | Bypass authentication                      |
| Deploy lại version mới nhiều lần        | Làm incident phức tạp hơn                  |
| Rollback nhưng không validate           | Không biết hệ thống thực sự recover chưa   |
| Chỉ xem Grafana                         | Dashboard không thay thế raw evidence      |
| Chỉ thấy service active rồi kết luận OK | Container có thể active trong khi app fail |

---

# 56. Security during troubleshooting

Day 20 rất dễ khiến operator phá security vì áp lực phục hồi nhanh.

Bạn phải giữ nguyên nguyên tắc:

```text
Least privilege
Minimum change
Minimum blast radius
No secret leakage
Known rollback
Evidence preservation
Approved access
```

Đặc biệt đừng đưa vào ticket:

```text
DB password
AWS access key
secret environment value
private SSH key
JWT
API token
TLS private key
```

Có thể ghi:

```text
DATABASE_PASSWORD present: yes
secret source: /approved/secret/path
value: [REDACTED]
```

---

# 57. Guided Integrated Lab — Day 20

Đây là phần mình khuyên bạn **thực sự chạy**.

## Phase A — Baseline

Tạo evidence folder:

```bash
mkdir -p ~/day20-evidence/baseline
```

Capture:

```bash
date -Is \
  | tee ~/day20-evidence/baseline/time.txt

hostname \
  | tee ~/day20-evidence/baseline/hostname.txt

systemctl status "$APP_SERVICE" --no-pager \
  > ~/day20-evidence/baseline/app-service.txt 2>&1

sudo ss -lntp \
  > ~/day20-evidence/baseline/listeners.txt

curl -v "$APP_URL" \
  > ~/day20-evidence/baseline/curl.txt 2>&1

df -hT \
  > ~/day20-evidence/baseline/df.txt

df -i \
  > ~/day20-evidence/baseline/df-inode.txt

pg_isready \
  -h "$PGHOST" \
  -p "$PGPORT" \
  > ~/day20-evidence/baseline/pg-isready.txt 2>&1
```

Expected invariant:

```text
Application works
Tomcat active
Expected port listening
PostgreSQL reachable
Disk has safe free capacity
Monitoring targets up
No active alert for this environment
```

---

# 58. Lab Case A — Service stopped

Inject:

```bash
sudo systemctl stop "$APP_SERVICE"
```

Bạn phải tự chứng minh:

```text
application unavailable
service inactive
port disappears
```

Recover:

```bash
sudo systemctl start "$APP_SERVICE"
```

Validate toàn stack.

---

# 59. Lab Case B — Wrong application port

Backup:

```bash
sudo cp -a \
  "$CATALINA_BASE/conf/server.xml" \
  "$CATALINA_BASE/conf/server.xml.day20"
```

Change lab connector:

```text
8080 → 18080
```

Validate:

```bash
"$CATALINA_HOME/bin/catalina.sh" configtest
```

Restart:

```bash
sudo systemctl restart "$APP_SERVICE"
```

Bạn phải chứng minh:

```text
8080 has no expected listener
18080 has Tomcat listener
curl 8080 fails
curl 18080 succeeds
```

Restore:

```bash
sudo cp -a \
  "$CATALINA_BASE/conf/server.xml.day20" \
  "$CATALINA_BASE/conf/server.xml"

sudo systemctl restart "$APP_SERVICE"
```

Validate `8080` trở lại.

---

# 60. Lab Case C — Permission

Dùng isolated test file như phần trước.

Mục tiêu không phải “chạy chmod”.

Mục tiêu là bạn phải phân biệt:

```text
wrong owner
wrong group
wrong mode
missing directory execute/search permission
service user not in expected group
```

và prove bằng:

```bash
stat
namei -l
id
sudo -u <service-user> test -r ...
```

---

# 61. Lab Case D — Disk full

Dùng isolated:

```text
/mnt/day20-full
```

Không fill `/`.

Bạn phải capture:

```bash
df -hT
df -i
du
write failure
```

sau đó remove chỉ failure-injection artifact và prove write hoạt động lại.

---

# 62. Lab Case E — PostgreSQL unavailable

Nếu dedicated local lab DB:

```bash
sudo systemctl stop postgresql
```

Investigate:

```bash
pg_isready -h "$PGHOST" -p "$PGPORT"
curl -v "$APP_URL"
journalctl -u "$APP_SERVICE" --since "-5 min"
```

Recover:

```bash
sudo systemctl start postgresql
```

Validate:

```bash
pg_isready -h "$PGHOST" -p "$PGPORT"
curl -v "$APP_URL"
```

Nếu DB dùng chung:

**không stop database**.

---

# 63. Lab Case F — Alert

Baseline:

```bash
systemctl is-active node_exporter
```

Inject:

```bash
sudo systemctl stop node_exporter
```

Observe:

```text
target → down
up → 0
alert → pending
alert → firing
notification → generated
```

Recover:

```bash
sudo systemctl start node_exporter
```

Observe:

```text
up → 1
alert → resolved
```

Capture evidence.

---

# 64. Lab Case G — Broken deployment & rollback

Baseline:

```bash
sha256sum "$WAR"
curl -v "$APP_URL"
```

Backup known-good artifact.

Deploy intentionally invalid lab artifact.

Capture:

```bash
systemctl status
ss
curl
journalctl
sha256sum
```

Sau đó rollback known-good artifact.

Cuối cùng prove:

```text
artifact checksum = expected
Tomcat active
port listening
application endpoint healthy
DB connectivity healthy
no new fatal error
monitoring healthy
```

---

# 65. Một incident record hoàn chỉnh

Sau mỗi scenario, record của bạn nên có các field:

| Field         | Nội dung                          |
| ------------- | --------------------------------- |
| Incident ID   | LAB-D20-01                        |
| Start time    | ISO timestamp                     |
| Symptom       | User-visible problem              |
| Expected      | Baseline behavior                 |
| Scope         | Host/app/component affected       |
| Recent change | Nếu có                            |
| Evidence      | Commands/logs/screenshots         |
| Hypotheses    | Những nguyên nhân đã xét          |
| Tests         | Các safe tests                    |
| Root cause    | Nguyên nhân đã chứng minh         |
| Remediation   | Thao tác đã thực hiện             |
| Validation    | Evidence recovery                 |
| Rollback      | Nếu remediation fail thì làm gì   |
| L1/L2         | Ownership decision                |
| Prevention    | Monitoring/runbook/config control |
| End time      | Recovery timestamp                |

---

# 66. Example troubleshooting reasoning

Giả sử:

```bash
curl http://localhost:8080/myapp
```

trả:

```text
HTTP 500
```

Bạn kiểm tra:

```bash
systemctl is-active tomcat
```

```text
active
```

Tiếp:

```bash
ss -lntp | grep 8080
```

listener có.

Như vậy:

```text
systemd layer = likely OK
process layer = likely OK
socket layer  = OK
HTTP transport = OK
```

Không nên restart Tomcat.

Tiếp tục logs:

```text
PSQLException:
Connection to db01:5432 refused
```

Test:

```bash
pg_isready -h db01 -p 5432
```

```text
no response
```

Bây giờ hypothesis mạnh nhất chuyển sang:

```text
DB service/network/listener
```

Đây chính là **layer isolation**.

---

# 67. Một ví dụ khác: đừng bị log đánh lừa

User:

```text
Cannot access app.
```

Bạn thấy app log hôm qua:

```text
database connection failed
```

Không kết luận DB lỗi.

Kiểm tra current state:

```bash
date -Is
systemctl status tomcat
ss -lntp
curl -v
pg_isready
```

Log phải correlate với **timestamp của incident**.

Một lỗi hôm qua không nhất thiết liên quan lỗi hôm nay.

---

# 68. Rollback mindset

Rollback không có nghĩa:

```text
copy old WAR and hope
```

Rollback đúng nghĩa:

```text
Identify last known good
        ↓
Verify artifact/config
        ↓
Assess rollback impact
        ↓
Approved execution
        ↓
Restore
        ↓
Restart/redeploy if required
        ↓
Technical validation
        ↓
Functional validation
        ↓
Monitoring validation
        ↓
Evidence
```

Nếu không có known-good version:

> **Đừng tự sáng tạo rollback. Escalate.**

---

# 69. Recovery hierarchy

Khi incident xảy ra, luôn ưu tiên thao tác reversible hơn irreversible.

Ví dụ:

```text
inspect
   ↓
test
   ↓
reload
   ↓
restart
   ↓
restore known-good config/artifact
   ↓
larger infrastructure change
   ↓
destructive recovery
```

Không phải incident nào cũng theo đúng thứ tự này, nhưng nó thể hiện nguyên tắc:

> **Safest useful action first.**

---

# 70. Day 20 Exam Focus — Theory

Bạn cần giải thích được sự khác nhau giữa:

| Cặp khái niệm                    | Phải hiểu                                          |
| -------------------------------- | -------------------------------------------------- |
| Service active vs app healthy    | Process sống chưa chắc business endpoint hoạt động |
| Process vs port                  | Process tồn tại chưa chắc bind đúng port           |
| Listener vs reachability         | Listener local chưa chắc client remote reach được  |
| Permission vs ownership          | Mode bits khác owner/group                         |
| Disk capacity vs inode           | Có thể còn GB nhưng hết inode                      |
| DB reachable vs DB authenticated | `pg_isready` khác login thành công                 |
| Alert pending vs firing          | `for:` duration                                    |
| Prometheus vs Alertmanager       | Evaluate alert khác notification delivery          |
| Restart vs root-cause fix        | Restart có thể chỉ che symptom                     |
| Redeploy vs rollback             | Deploy lại không nhất thiết restore known-good     |
| L1 remediation vs escalation     | Scope/authority/runbook                            |
| Evidence vs assumption           | Evidence phải support conclusion                   |

---

# 71. Day 20 Practical Exam Focus

Nếu practical exam đưa symptom:

```text
Application unavailable
```

đừng cố đoán đề muốn “restart Tomcat”.

Examiner có thể cố tình tạo:

```text
service stopped
wrong port
wrong ownership
disk full
DB unavailable
alert broken
bad deployment
```

Điểm mạnh nằm ở process:

```text
inspect → isolate → prove → remediate → validate
```

không phải ai chạy lệnh nhanh nhất.

---

# 72. Knowledge Check

Bạn nên tự trả lời được các câu này trước khi coi phần lý thuyết Day 20 đã ổn:

1. `systemctl status` báo `active` nhưng `curl` trả HTTP 500 — tại sao chưa được kết luận service healthy?

2. `curl localhost:8080` trả `Connection refused`; command nào giúp xác định có listener không?

3. Java process tồn tại nhưng `ss` không thấy port 8080 — bạn nghi ngờ những gì?

4. Tại sao `chmod 777` không phải cách sửa permission đúng?

5. Tại sao phải kiểm tra permission ở parent directories?

6. `df -h` còn 40% nhưng app báo `No space left on device`; bạn kiểm tra tiếp gì?

7. `pg_isready` trả success có chứng minh DB password đúng không?

8. PostgreSQL listen `127.0.0.1:5432`; application ở EC2 khác có connect được không?

9. Alert expression true nhưng alert mới `Pending`; vì sao?

10. Alert đang `Firing` trên Prometheus nhưng không có notification; component nào cần điều tra tiếp?

11. Vì sao phải lấy SHA256 trước deployment?

12. Vì sao rollback xong vẫn phải `curl`, check DB và monitoring?

13. Khi nào L1 nên escalation thay vì tiếp tục thử?

14. Tại sao restart trước khi thu log có thể làm investigation khó hơn?

15. Khi port đã bị process khác chiếm, tại sao không được `kill -9` PID ngay?

---

# 73. Independent Practical Challenge

Khi bạn đã hoàn thành guided labs, hãy thử bài này mà **không nhìn solution**:

```text
Incident:

Users report that the Java application stopped responding
after a deployment.

You are L1.

Observed:
- EC2 is reachable through SSH.
- Tomcat service appears active.
- Grafana shows application availability down.
- A deployment was completed 7 minutes ago.

Your job:

Determine the actual failure.
Do not assume deployment is the root cause.

You must inspect:
service
process
port
HTTP endpoint
permissions
disk
database connectivity
application logs
monitoring state
current artifact

Restore service within L1 scope.

If recovery requires an unauthorized change,
stop and prepare escalation evidence.

Deliverables:
root cause
commands used
evidence
remediation
validation
rollback status
escalation decision
handover note
```

Đây gần như là bài tổng hợp hoàn hảo của Day 20.

---

# 74. Master troubleshooting cheat sheet

Khi có incident, bạn có thể bắt đầu bằng bộ này:

```bash
# Time / identity
date -Is
hostname
uptime

# Service
systemctl status "$APP_SERVICE" --no-pager
systemctl is-active "$APP_SERVICE"
systemctl cat "$APP_SERVICE"

# Process
ps -ef | grep '[j]ava'

# Port/socket
sudo ss -lntp

# Application
curl -v "$APP_URL"

# Service logs
journalctl -u "$APP_SERVICE" \
  --since "-15 min" \
  --no-pager

# Permissions
stat /path/to/file
namei -l /path/to/file
id <service-user>

# Disk
df -hT
df -i
sudo du -xhd1 /var 2>/dev/null | sort -h

# PostgreSQL transport
pg_isready -h "$PGHOST" -p "$PGPORT"

# PostgreSQL real connection
psql \
  -h "$PGHOST" \
  -p "$PGPORT" \
  -U "$PGUSER" \
  -d "$PGDATABASE"

# Prometheus config/rules
promtool check config /etc/prometheus/prometheus.yml
promtool check rules /etc/prometheus/rules/*.yml

# Deployment evidence
sha256sum "$WAR"
```

Đừng chạy chúng như robot. Với mỗi command hãy tự hỏi:

> **“Nếu output A thì hypothesis nào mạnh lên? Nếu output B thì layer nào được loại bỏ?”**

Đó mới là troubleshooting skill.

---

# 75. Mastery Check của Day 20

Sau Day 20, mình **chưa coi bạn mastery chỉ vì đã đọc hết bài này**.

Bạn đạt mức lý thuyết tốt khi có thể nhìn symptom và giải thích được:

```text
symptom
→ expected behavior
→ affected layer
→ evidence required
→ hypotheses
→ safest test
→ root cause
→ remediation
→ validation
→ rollback/escalation
```

Bạn đạt mức **practical mastery** khi thực sự có thể:

```text
inject service failure
→ diagnose
→ recover

inject port failure
→ diagnose
→ recover

inject permission failure
→ diagnose
→ recover

inject isolated disk-full
→ diagnose
→ recover

inject DB unavailability
→ diagnose
→ recover

inject monitoring failure
→ verify alert lifecycle
→ recover

inject bad deployment
→ collect evidence
→ rollback
→ validate
```

và quan trọng nhất:

> **không phá security, không mở rộng blast radius, không làm mất evidence và không sửa vượt L1 authority.**

Đó chính xác là năng lực Day 20 mà syllabus yêu cầu: **investigate failure → restore trong phạm vi L1 → collect logs/evidence → escalate phần vượt scope**. 00_MASTER_SYLLABUS_Java_Fresher…

Day 20 cũng là điểm nối trực tiếp sang **Day 21 — configuration review, operational acceptance, consolidation, mock assessment và final readiness**. Vì vậy, output tốt nhất của hôm nay không chỉ là “app chạy lại”, mà là một **evidence pack đầy đủ chứng minh bạn đã phát hiện, xử lý, validate, rollback/escalate và hand over incident một cách có kiểm soát**.
