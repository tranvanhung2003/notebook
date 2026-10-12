# Day 21 — Configuration Review, Operational Acceptance, Knowledge Consolidation, Mock Assessment & Final Readiness

Theo **MASTER_SYLLABUS**, Day 21 bắt buộc gồm:

> **Concept/Lecture:** Configuration review, operational acceptance, knowledge consolidation, mock assessment, and final readiness.  
> **Assignment/Lab:** Complete peer/mentor configuration review, close gaps, rehearse handover, and submit the final evidence package. 00_MASTER_SYLLABUS_Java_Fresher…

Day 21 nằm ngay sau Day 19 — integrated project build — và Day 20 — failure injection/troubleshooting. Vì vậy hôm nay không phải là “cài thêm một công nghệ”, mà là **kiểm tra xem toàn bộ hệ thống đã thực sự đủ điều kiện để vận hành hay chưa**. 00_MASTER_SYLLABUS_Java_Fresher…

Day 21 cũng là bước chuẩn bị trực tiếp cho Day 22, nơi syllabus yêu cầu cả **theory exam** và **practical exam** bao phủ Linux, AWS, Java, PostgreSQL, monitoring, change controls, escalation, đồng thời yêu cầu deploy → configure → validate → troubleshoot → document → hand over một integrated scenario. 00_MASTER_SYLLABUS_Java_Fresher…

---

## 1. Syllabus Alignment

### Mandatory — bắt buộc theo syllabus

| Thành phần              | Bạn phải đạt được hôm nay                                                    |
| ----------------------- | ---------------------------------------------------------------------------- |
| Configuration review    | Biết kiểm tra cấu hình thực tế so với configuration baseline/expected state  |
| Operational acceptance  | Biết quyết định hệ thống có đủ điều kiện vận hành hay chưa dựa trên evidence |
| Knowledge consolidation | Kết nối Linux → AWS → Java/Tomcat → PostgreSQL → monitoring → operations     |
| Mock assessment         | Tự xử lý câu hỏi lý thuyết và tình huống thực hành gần với Day 22            |
| Final readiness         | Biết mình còn yếu ở đâu trước final exam                                     |
| Peer/mentor review      | Có checklist để người khác review cấu hình của mình                          |
| Close gaps              | Mọi finding phải được fix, accepted risk hoặc escalated                      |
| Handover rehearsal      | Có thể bàn giao hệ thống cho ca/người vận hành tiếp theo                     |
| Final evidence package  | Có bộ evidence hoàn chỉnh chứng minh hệ thống đã được kiểm tra               |

### Supplementary / Beyond the explicit syllabus

Mình sẽ bổ sung một số kỹ thuật vận hành thực tế: **configuration drift**, acceptance gates, severity classification, evidence traceability, risk acceptance, configuration checksum, operational readiness checklist và readiness decision matrix.

Khái niệm này rất phù hợp với cách AWS mô tả **Operational Readiness Review (ORR)**: một workload nên được đánh giá bằng checklist nhất quán về architecture, operational process, event management, release quality, security/governance; các finding không có mitigation thích hợp cần được xử lý trước launch. [AWS Documentation](https://docs.aws.amazon.com/wellarchitected/latest/operational-readiness-reviews/wa-operational-readiness-reviews.html?utm_source=chatgpt.com)

---

# 2. Objectives — Sau Day 21 bạn phải làm được gì?

Bạn chưa được coi là “ready” chỉ vì:

```text
systemctl status tomcat
Active: active (running)
```

Một process chạy không chứng minh application hoạt động đúng.

Tương tự:

```text
pg_isready
accepting connections
```

chỉ chứng minh PostgreSQL đang accept connection, không chứng minh application có đúng DB credentials, đúng schema hay query nghiệp vụ hoạt động.

Sau hôm nay, bạn cần có mental model:

```text
Requirement
    ↓
Expected configuration / baseline
    ↓
Actual configuration
    ↓
Validation
    ↓
Evidence
    ↓
Gap?
   /   \
 No     Yes
 |       |
PASS   Fix / Mitigate / Escalate
         ↓
      Revalidate
         ↓
       Evidence
         ↓
Operational Acceptance Decision
```

Và ở mức hệ thống:

```text
Client
   ↓
DNS / Network
   ↓
AWS VPC / Route / SG
   ↓
EC2 / Linux
   ↓
socket / port
   ↓
systemd
   ↓
Java / JVM / Tomcat
   ↓
application
   ↓
PostgreSQL
   ↓
logs + metrics
   ↓
Prometheus / Grafana / alert
   ↓
operations / rollback / handover
```

Đây chính là integrated thinking mà fresher administrator cần có.

---

# 3. Mental Model quan trọng nhất của Day 21

## 3.1 Configuration review không phải “đọc config”

Configuration review trả lời bốn câu hỏi:

| Câu hỏi                   | Ý nghĩa                                   |
| ------------------------- | ----------------------------------------- |
| What should it be?        | Baseline/requirement là gì?               |
| What is it now?           | Actual state là gì?                       |
| How do you know?          | Evidence nào chứng minh?                  |
| What happens if it fails? | Recovery/rollback/escalation như thế nào? |

Ví dụ baseline:

```text
Tomcat service account: tomcat
Application port: 8080
Context path: /demo
DB endpoint: db.internal.example
DB port: 5432
Prometheus scrape: enabled
node_exporter: enabled
Restart after reboot: enabled
```

Không được review theo kiểu:

> “Chắc là Tomcat chạy port 8080.”

Phải kiểm chứng:

```bash
ss -lntp
```

và:

```bash
systemctl status <tomcat-service>
```

và:

```bash
curl <approved-application-endpoint>
```

Ba lệnh trả lời ba câu hỏi khác nhau:

```text
systemctl → service/process state
ss        → socket/network listener
curl      → application-level behavior
```

Một trong ba PASS không suy ra hai cái còn lại PASS.

---

# 4. Configuration Baseline

**Configuration baseline** là bộ giá trị đã được approve và dùng làm expected state.

Ví dụ:

| Configuration item | Approved state                            |
| ------------------ | ----------------------------------------- |
| OS                 | Approved Linux distribution/version       |
| Hostname           | app01                                     |
| Java               | approved Java version                     |
| Tomcat service     | enabled + running                         |
| Service user       | non-root dedicated account                |
| App artifact       | approved release                          |
| App context        | `/demo`                                   |
| App port           | approved listening port                   |
| DB host            | approved PostgreSQL endpoint              |
| DB role            | least privilege                           |
| PostgreSQL port    | 5432 unless environment defines otherwise |
| Prometheus scrape  | healthy                                   |
| node_exporter      | running                                   |
| Alert rules        | loaded                                    |
| EC2 tags           | required tags present                     |
| SG ingress         | only required sources/ports               |
| Backup             | recent and recovery-tested                |
| Rollback           | documented and rehearsed                  |

Baseline có thể đến từ:

```text
build sheet
change ticket
deployment document
architecture document
CMDB
runbook
environment specification
approved configuration file
```

### Một lỗi rất thường gặp

Admin kiểm tra:

```bash
java -version
```

thấy Java chạy được rồi ghi:

```text
PASS
```

Sai.

Bạn phải so:

```text
Actual Java version
      VS
Approved Java version
```

Nếu baseline yêu cầu Java 17 mà máy chạy một version khác, “Java vẫn chạy” không đủ để PASS.

---

# 5. Configuration Drift

### Supplementary / Beyond the explicit syllabus

**Configuration drift** xảy ra khi actual system dần khác approved baseline.

Ví dụ:

```text
Baseline:
Tomcat port = 8080

Sau troubleshooting:
admin đổi sang 8081

Incident resolved

Nhưng:
ticket không update
documentation không update
monitoring vẫn scrape 8080
SG chỉ allow 8080
```

Bạn có:

```text
application có thể chạy
nhưng configuration state không còn trustworthy
```

Day 21 phải tìm những drift này.

Một kỹ thuật hữu ích:

```bash
sha256sum <approved-config-file>
```

Ví dụ:

```bash
sha256sum /path/to/server.xml
```

Nếu tổ chức giữ approved checksum:

```text
expected SHA256
vs
actual SHA256
```

Checksum chỉ cho biết file có thay đổi hay không.

Nó **không nói thay đổi đó đúng hay sai**.

---

# 6. Operational Acceptance là gì?

Operational acceptance khác functional acceptance.

### Functional acceptance

Câu hỏi:

> Application có thực hiện đúng chức năng nghiệp vụ không?

Ví dụ:

```text
login thành công
create order thành công
API trả response đúng
```

### Operational acceptance

Câu hỏi rộng hơn:

> Operations team có khả năng vận hành, monitor, troubleshoot, recover và hand over hệ thống an toàn không?

Ví dụ application hoạt động tốt nhưng:

```text
không có monitoring
backup chưa từng restore
không biết log ở đâu
service chạy bằng root
credentials nằm plaintext world-readable
không có rollback
không ai biết owner
```

Thì hệ thống **chưa operationally ready**.

AWS cũng mô tả operational readiness là đánh giá workload, process, procedure và personnel readiness, đồng thời yêu cầu runbook cho routine activities, playbook cho investigation và cơ chế quản lý change. [AWS Documentation](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/operational-readiness.html?utm_source=chatgpt.com)

---

# 7. Acceptance States

Trong Day 21, đừng chỉ dùng PASS/FAIL một cách quá đơn giản.

Một mô hình hữu ích là:

| State                | Ý nghĩa                                              |
| -------------------- | ---------------------------------------------------- |
| PASS                 | Requirement đáp ứng và có evidence                   |
| PASS WITH KNOWN RISK | Có gap nhưng được approve mitigation/risk acceptance |
| FAIL / BLOCKER       | Không đủ điều kiện acceptance                        |
| N/A                  | Requirement thực sự không áp dụng                    |
| NOT VERIFIED         | Chưa có evidence; không được giả định là PASS        |

Đặc biệt:

```text
NOT VERIFIED ≠ PASS
```

Nếu bạn không biết backup restore được hay không:

```text
Status = NOT VERIFIED
```

không được viết:

```text
Backup configured → PASS
```

---

# 8. Những domain phải review trong Day 21

Đây là checklist tổng hợp Day 1–20.

| Domain        | Review chính                                             |
| ------------- | -------------------------------------------------------- |
| Linux         | users, permissions, disk, memory, process, service, logs |
| Networking    | IP, DNS, ports, socket, firewall/network path            |
| AWS           | EC2, EBS, VPC, subnet, route, SG, IAM, tags, costs       |
| Java/JVM      | Java version, JVM options, process ownership             |
| Tomcat        | service, connectors, deployment, logs, application       |
| Configuration | environment-specific config, profiles, secrets           |
| PostgreSQL    | connectivity, role, database, health, capacity, backup   |
| Monitoring    | Prometheus, node_exporter, Grafana, alerts               |
| Operations    | ticket, approval, runbook, rollback                      |
| Recovery      | backup, restore, rollback                                |
| Evidence      | timestamped proof                                        |
| Handover      | current state, risk, pending work, escalation            |

---

# 9. Evidence-first Administration

Một fresher thường báo:

> “Em kiểm tra rồi, mọi thứ OK.”

Senior administrator sẽ hỏi:

> Evidence đâu?

Một evidence tốt phải có đủ:

```text
WHAT was checked
WHEN it was checked
WHERE it was checked
HOW it was checked
WHAT result was obtained
WHO/WHAT owns the finding
```

Ví dụ evidence yếu:

```text
Tomcat OK
```

Evidence tốt hơn:

```text
2026-10-12T17:00:00+07:00
Host: app01
Check: Tomcat service state
Command: systemctl is-active tomcat
Result: active
Status: PASS
```

Nhưng vẫn cần endpoint validation để chứng minh application layer.

---

# 10. Day 21 Practical Lab — Final Operational Readiness Review

Phần này là lab chính hôm nay.

## Lab objective

Bạn sẽ kiểm tra toàn bộ stack:

```text
AWS
 ↓
EC2/Linux
 ↓
Tomcat/Java
 ↓
Java application
 ↓
PostgreSQL
 ↓
Prometheus/node_exporter/Grafana
```

Sau đó:

```text
review
→ record findings
→ close gaps
→ revalidate
→ handover rehearsal
→ final evidence package
```

---

# 11. Lab Assumptions

Không giả định service của bạn tên `tomcat`.

Không giả định config nằm `/opt/tomcat`.

Không giả định application có `/health`.

Không giả định PostgreSQL local.

Trước tiên phải discovery actual environment.

Các placeholder:

```text
<TOMCAT_SERVICE>
<APP_CONTEXT>
<APP_URL>
<DB_HOST>
<DB_NAME>
<DB_USER>
<PROMETHEUS_CONFIG>
<ALERT_RULE_FILE>
```

phải được thay bằng giá trị của lab environment.

---

# 12. Safety Gate trước khi review

Day 21 nên ưu tiên **read-only validation**.

Không chạy các hành động sau chỉ để “test”:

```text
rm
chmod -R
chown -R
iptables flush
firewall disable
EC2 terminate
EBS delete
DROP DATABASE
DELETE FROM
TRUNCATE
kill -9
```

Trước bất kỳ remediation nào:

```text
1. Capture current state
2. Xác định baseline
3. Xác định blast radius
4. Có rollback plan
5. Có backup nếu liên quan data/config quan trọng
6. Apply minimum change
7. Validate
8. Record evidence
```

---

# 13. Bước 1 — Tạo evidence package

Ví dụ:

```bash
mkdir -p ~/day21-final-evidence/{00-scope,01-architecture,02-inventory,03-linux,04-aws,05-java-tomcat,06-postgresql,07-monitoring,08-security-cost,09-change-rollback,10-gaps,11-handover,12-mock}
```

Kiểm tra:

```bash
find ~/day21-final-evidence -maxdepth 1 -type d | sort
```

Ghi thời gian bắt đầu:

```bash
date -Is | tee ~/day21-final-evidence/00-scope/review-start.txt
```

Hostname:

```bash
hostname | tee ~/day21-final-evidence/00-scope/hostname.txt
```

### Expected invariant

Bạn phải biết chính xác:

```text
mình đang review máy nào
environment nào
thời điểm nào
scope nào
```

---

# 14. Bước 2 — Host/OS Inventory

Chạy:

```bash
hostnamectl
```

```bash
uname -r
```

```bash
cat /etc/os-release
```

```bash
uptime
```

Evidence:

```bash
{
    date -Is
    hostnamectl
    uname -r
    cat /etc/os-release
    uptime
} | tee ~/day21-final-evidence/02-inventory/os-inventory.txt
```

Bạn phải đọc được:

```text
OS distribution
OS version
kernel
hostname
uptime
```

Không chỉ chạy command.

---

# 15. Bước 3 — Capacity Review

## Filesystem capacity

```bash
df -hT
```

Đừng chỉ nhìn `/`.

Kiểm tra filesystem chứa:

```text
application
Tomcat
logs
PostgreSQL
Prometheus
backup
/tmp
```

### Inode

```bash
df -i
```

Một filesystem có thể:

```text
disk usage = 40%
inode usage = 100%
```

và vẫn không tạo được file.

## Memory

```bash
free -h
```

## CPU/load

```bash
uptime
```

Nếu có:

```bash
top
```

hoặc:

```bash
ps aux --sort=-%cpu | head
```

và:

```bash
ps aux --sort=-%mem | head
```

### Không kết luận sai

Load average cao không tự động có nghĩa CPU 100%.

Bạn cần correlate:

```text
load
CPU
I/O
processes
memory
```

---

# 16. Bước 4 — systemd Review

Tìm failed units:

```bash
systemctl --failed
```

Đối với từng critical service:

```bash
systemctl status <SERVICE> --no-pager
```

Kiểm tra boot enablement:

```bash
systemctl is-enabled <SERVICE>
```

Kiểm tra active:

```bash
systemctl is-active <SERVICE>
```

Ví dụ:

```bash
systemctl is-active <TOMCAT_SERVICE>
systemctl is-enabled <TOMCAT_SERVICE>
```

### Khác nhau

```text
enabled
```

nghĩa là configured để khởi động theo boot target thích hợp.

```text
active
```

nghĩa là hiện tại unit đang active.

Có thể:

```text
enabled nhưng failed
```

hoặc:

```text
disabled nhưng active
```

Do đó phải kiểm tra cả hai.

---

# 17. Review service account

```bash
systemctl show <TOMCAT_SERVICE> \
  -p User \
  -p Group \
  -p ExecStart
```

Mục tiêu:

```text
Tomcat không chạy bằng root nếu không có requirement đặc biệt
```

Tomcat documentation hiện tại cũng tiếp tục khuyến nghị chạy service bằng một user riêng với reduced permissions. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/setup.html?utm_source=chatgpt.com)

Kiểm tra process:

```bash
ps -ef | grep '[j]ava'
```

hoặc:

```bash
pgrep -a java
```

So sánh:

```text
systemd unit
vs
running process
```

---

# 18. Logs Review

```bash
journalctl -u <TOMCAT_SERVICE> -n 100 --no-pager
```

Nếu cần xem boot hiện tại:

```bash
journalctl -u <TOMCAT_SERVICE> -b --no-pager
```

Tìm lỗi:

```bash
journalctl -u <TOMCAT_SERVICE> -b --no-pager |
grep -Ei 'error|exception|failed|fatal'
```

Nhưng đừng phạm lỗi:

```text
grep thấy chữ "error" → hệ thống fail
```

Ví dụ:

```text
Previous error count: 0
```

vẫn match `error`.

Bạn phải đọc context.

---

# 19. Port / Socket Review

```bash
ss -lnt
```

Nếu có quyền xem process:

```bash
sudo ss -lntp
```

`sudo` chỉ dùng khi được phép.

Ví dụ bạn cần xác minh:

```text
Tomcat → expected port
PostgreSQL → expected binding
Prometheus → expected port
node_exporter → expected port
Grafana → expected port
```

Đừng mặc định mọi port phải bind:

```text
0.0.0.0
```

Nhiều service nên chỉ bind:

```text
127.0.0.1
```

hoặc private interface.

---

# 20. Java Runtime Review

```bash
java -version
```

Tìm Java binary:

```bash
command -v java
```

Nếu muốn xem actual path:

```bash
readlink -f "$(command -v java)"
```

So sánh với baseline:

```text
approved Java version
JAVA_HOME if applicable
actual runtime used by service
```

Một lỗi kinh điển:

```text
shell:
java -version → Java A

systemd:
ExecStart uses → Java B
```

Do đó `java -version` trong shell chưa chắc là JVM Tomcat đang sử dụng.

Bạn cần kiểm tra:

```bash
systemctl show <TOMCAT_SERVICE> -p ExecStart
```

và process:

```bash
ps -ef | grep '[j]ava'
```

---

# 21. JVM Configuration Review

Bạn cần xác minh các setting đã học Day 13:

```text
-Xms
-Xmx
other approved JVM options
```

Không được tự ý “tune RAM” trong Day 21.

Review:

```text
approved:
-Xms512m
-Xmx1024m

actual:
-Xms512m
-Xmx1024m

→ PASS
```

Hoặc:

```text
approved:
-Xmx1024m

actual:
-Xmx4096m

→ GAP
```

Sau đó tìm:

```text
why was it changed?
change ticket?
memory available?
rollback?
```

---

# 22. Không được dump secrets vào evidence

Một anti-pattern nguy hiểm:

```bash
env > evidence.txt
```

hoặc:

```bash
cat application.properties > evidence.txt
```

File đó có thể chứa:

```text
DB_PASSWORD
API_TOKEN
SECRET_KEY
AWS credentials
```

Evidence package **không được trở thành secret leakage package**.

Thay vì capture value, capture:

```text
configuration file exists
owner
permissions
checksum
approved key presence where safe
```

Ví dụ:

```bash
stat <CONFIG_FILE>
```

hoặc Linux GNU `stat`:

```bash
stat -c '%a %U:%G %n' <CONFIG_FILE>
```

Checksum:

```bash
sha256sum <CONFIG_FILE>
```

Không hiển thị contents nếu có khả năng chứa credentials.

---

# 23. Tomcat Configuration Review

Tomcat terminology rất quan trọng:

```text
CATALINA_HOME
```

là Tomcat installation.

```text
CATALINA_BASE
```

là instance-specific runtime/configuration base.

Tomcat documentation hiện tại vẫn mô tả `CATALINA_BASE` là nơi chứa instance-specific configuration và yêu cầu các configuration quan trọng như `conf/server.xml`. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/introduction.html?utm_source=chatgpt.com)

Bạn cần biết trong environment của mình:

```text
CATALINA_HOME = ?
CATALINA_BASE = ?
```

Không đoán.

---

# 24. Application Deployment Review

Kiểm tra deployed artifact.

Ví dụ:

```bash
ls -lah <CATALINA_BASE>/webapps/
```

Nhưng đường dẫn thực tế phụ thuộc deployment model.

Bạn phải xác minh:

```text
approved artifact
version/build
context
ownership
deployment status
```

Tomcat hiện tại cũng cung cấp Manager functionality có thể list application context, running/stopped status và active sessions khi Manager được cấu hình và người dùng có quyền phù hợp. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/index.html?utm_source=chatgpt.com)

Không bật Tomcat Manager chỉ để Day 21 review nếu lab chưa dùng nó.

---

# 25. Application Endpoint Validation

Nếu application **thực sự cung cấp health endpoint**, dùng endpoint đó.

Ví dụ:

```bash
curl -fsS http://127.0.0.1:<PORT>/<APP_CONTEXT>/<HEALTH_ENDPOINT>
```

Nếu không có health endpoint, test một known safe endpoint:

```bash
curl -i http://127.0.0.1:<PORT>/<APP_CONTEXT>/
```

Không được tự giả định:

```text
/health
/actuator/health
/status
```

tồn tại.

### Validate status code

```bash
curl -o /dev/null \
     -s \
     -w '%{http_code}\n' \
     http://127.0.0.1:<PORT>/<APP_CONTEXT>/
```

Expected status phụ thuộc application.

Không phải cứ `200` mới đúng; một endpoint có authentication có thể trả response khác theo design.

---

# 26. Network Validation — Test theo từng layer

Nếu client không truy cập app:

```text
DNS?
 ↓
IP routing?
 ↓
AWS SG?
 ↓
Linux firewall?
 ↓
socket listening?
 ↓
Tomcat?
 ↓
application?
```

Không bắt đầu bằng restart.

Ví dụ:

```bash
getent hosts <APP_HOSTNAME>
```

```bash
ping <HOST>
```

Ping chỉ sử dụng nếu ICMP được phép.

```bash
curl -v http://<HOST>:<PORT>/<CONTEXT>
```

```bash
ss -lnt
```

Evidence-driven logic:

```text
localhost curl PASS
remote curl FAIL
```

gợi ý investigate:

```text
network path
firewall
SG
bind address
```

thay vì application logic trước tiên.

---

# 27. AWS Configuration Review

Day 21 không yêu cầu tạo thêm resource.

Ưu tiên read-only.

## Verify current identity

```bash
aws sts get-caller-identity
```

Điều này giúp tránh sai account.

## Verify CLI configuration

```bash
aws configure list
```

Không paste secret access keys vào evidence.

## EC2 inventory

Ví dụ:

```bash
aws ec2 describe-instances \
  --instance-ids <INSTANCE_ID> \
  --query 'Reservations[].Instances[].{State:State.Name,Type:InstanceType,Vpc:VpcId,Subnet:SubnetId,PrivateIP:PrivateIpAddress,SG:SecurityGroups[].GroupId,Tags:Tags}' \
  --output json
```

Bạn cần xác minh:

```text
correct account
correct region
correct EC2 instance
correct VPC
correct subnet
correct Security Groups
required tags
instance state
```

---

# 28. Security Group Review

```bash
aws ec2 describe-security-groups \
  --group-ids <SG_ID>
```

Đặt câu hỏi:

```text
Port nào exposed?
Source nào allowed?
Có 0.0.0.0/0 không?
Nếu có, business requirement là gì?
Database có public exposure không?
SSH exposure có đúng approved source không?
```

Không kết luận:

```text
0.0.0.0/0 = luôn luôn sai
```

Ví dụ public HTTP/HTTPS service có thể cần public ingress.

Nhưng:

```text
PostgreSQL 5432 from 0.0.0.0/0
```

thường là một security finding nghiêm trọng trong kiến trúc thông thường.

---

# 29. AWS Cost Review

Bạn đã học Day 11 về cost controls; Day 21 phải kiểm tra lại.

Review:

```text
unused EC2
unattached EBS
snapshots
public IPv4
unused resources
required tags
budget/alarm
```

Ví dụ read-only:

```bash
aws ec2 describe-volumes \
  --filters Name=status,Values=available
```

`available` volume nghĩa là volume hiện không attached.

Nhưng:

```text
available → delete
```

là suy luận nguy hiểm.

Có thể đó là:

```text
recovery volume
retained data
planned attachment
```

Do đó chỉ record finding.

---

# 30. PostgreSQL Operational Review

PostgreSQL hiện tại có `pg_isready` để kiểm tra server connection status. Exit codes phân biệt accepting, rejecting, no response và invalid/no attempt. [PostgreSQL](https://www.postgresql.org/docs/16/app-pg-isready.html?utm_source=chatgpt.com)

Ví dụ:

```bash
pg_isready -h <DB_HOST> -p 5432
```

Sau đó:

```bash
echo $?
```

Theo official behavior:

```text
0 → accepting connections
1 → rejecting connections
2 → no response
3 → no attempt / invalid parameters
```

Điều này đặc biệt hữu ích trong script.

---

# 31. PostgreSQL Application-level Connectivity

`pg_isready` chưa đủ.

Dùng approved DB role:

```bash
psql \
  -h <DB_HOST> \
  -p 5432 \
  -U <DB_USER> \
  -d <DB_NAME>
```

Không viết password trực tiếp:

```bash
psql ... password=MySecret123
```

vào command line hoặc evidence.

Trong `psql`:

```sql
\conninfo
```

Sau đó:

```sql
SELECT current_database(), current_user;
```

Và:

```sql
SELECT now();
```

Đây là evidence tốt hơn:

```text
network works
authentication works
target database works
session works
```

---

# 32. PostgreSQL Capacity Review

Trong `psql`:

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
```

Session check:

```sql
SELECT count(*)
FROM pg_stat_activity;
```

Nếu bạn đang troubleshoot thì có thể xem thêm:

```sql
SELECT pid,
       usename,
       datname,
       state,
       wait_event_type,
       wait_event
FROM pg_stat_activity;
```

Quyền truy cập chi tiết có thể khác tùy role.

Không kill sessions chỉ vì thấy session lâu.

---

# 33. Backup / Recovery Review

Day 21 không hỏi:

> Có file backup không?

Mà hỏi:

> Bạn có evidence rằng backup có thể được dùng để recover không?

Mức độ assurance:

```text
backup configured
    ↓
backup file exists
    ↓
backup file non-empty
    ↓
backup can be inspected
    ↓
restore tested
    ↓
recovered data validated
```

Càng xuống thấp càng mạnh.

Với custom-format `pg_dump` archive:

```bash
pg_restore -l <BACKUP_FILE>
```

là một validation không destructive hữu ích.

Nhưng:

```text
pg_restore -l succeeds
```

không mạnh bằng:

```text
restore vào controlled test database
+
validate data
```

### Cảnh báo

Không thực hiện restore vào production database chỉ để chứng minh Day 21.

---

# 34. Monitoring Review — Prometheus

Current Prometheus `promtool` hỗ trợ:

```text
check config
check rules
check healthy
check ready
```

để validate configuration và server health/readiness. [prometheus.io](https://prometheus.io/docs/prometheus/latest/command-line/promtool/?utm_source=chatgpt.com)

## Check configuration

Ví dụ:

```bash
promtool check config <PROMETHEUS_CONFIG>
```

Official tool còn support:

```bash
promtool check rules <RULE_FILE>
```

Rule syntax validation được Prometheus chính thức khuyến nghị trước khi reload. [prometheus.io](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/?utm_source=chatgpt.com)

### Expected invariant

```text
SUCCESS / exit code 0
```

Không sửa config production trước khi syntax validation.

---

# 35. Prometheus Runtime Readiness

Nếu version hỗ trợ:

```bash
promtool check healthy --url=http://localhost:9090
```

và:

```bash
promtool check ready --url=http://localhost:9090
```

Official Prometheus documentation hiện tại hỗ trợ cả hai command. [prometheus.io](https://prometheus.io/docs/prometheus/latest/command-line/promtool/?utm_source=chatgpt.com)

Nhưng vẫn cần kiểm tra scrape state.

Query quan trọng nhất:

```promql
up
```

Ví dụ:

```promql
up{job="node_exporter"}
```

Mental model:

```text
Prometheus service running
        ≠
targets successfully scraped
```

---

# 36. node_exporter Review

Nếu node_exporter của lab expose local endpoint:

```bash
curl -fsS http://127.0.0.1:9100/metrics | head
```

Điều này xác minh:

```text
HTTP exporter endpoint responds
metrics text is exposed
```

Sau đó Prometheus phải scrape được.

Hai lớp khác nhau:

```text
curl localhost:9100 PASS
Prometheus up == 0
```

thì exporter có thể tốt nhưng scrape path/network/config có vấn đề.

---

# 37. Grafana Review

Bạn cần xác minh:

```text
Grafana running
datasource works
dashboard loads
key panels contain recent data
time range correct
```

Một dashboard mở được nhưng toàn:

```text
No data
```

không phải monitoring PASS.

Bạn cần trace:

```text
dashboard
 ↓
Grafana datasource
 ↓
Prometheus
 ↓
target
 ↓
exporter/application
```

---

# 38. Alert Review

Một alert tồn tại trong YAML chưa có nghĩa alert hoạt động.

Acceptance cần:

```text
rule syntax valid
 ↓
rule loaded
 ↓
expression evaluates
 ↓
state changes correctly
 ↓
notification path works if configured
 ↓
evidence recorded
```

Nếu Day 17 đã có failure injection evidence thì Day 21 nên đưa evidence đó vào final package hoặc revalidate theo lab requirement.

---

# 39. Security Review

Security review Day 21 ít nhất phải kiểm tra:

| Area             | Question                                         |
| ---------------- | ------------------------------------------------ |
| Linux            | Service có chạy root không?                      |
| File permissions | Config/secrets có readable quá rộng không?       |
| SSH              | Access có đúng key/approved user không?          |
| IAM              | Quyền có vượt nhu cầu không?                     |
| SG               | Có exposure không cần thiết không?               |
| Application      | Secrets có hardcoded không?                      |
| PostgreSQL       | App role có excessive privileges không?          |
| Monitoring       | Monitoring endpoints có exposure quá rộng không? |
| Evidence         | Evidence có chứa secrets không?                  |

Tomcat current security documentation cũng nhấn mạnh chỉ nên deploy trusted web applications và các capability có thể sửa application phải được hạn chế cho trusted users/scope. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/security-howto.html?utm_source=chatgpt.com)

---

# 40. Configuration Gap Management

Khi review phát hiện vấn đề, đừng sửa ngay mọi thứ.

Workflow:

```text
Finding
  ↓
Confirm expected state
  ↓
Assess impact
  ↓
Classify
  ↓
Can L1 safely remediate?
 / \
yes  no
 |    |
fix  escalate
 |
validate
 |
update evidence
```

### Supplementary severity model

| Severity | Ví dụ                                                                        |
| -------- | ---------------------------------------------------------------------------- |
| Blocker  | App inaccessible, data recovery impossible, critical security exposure       |
| Major    | Monitoring missing, backup unverified, config significantly differs baseline |
| Minor    | Documentation inconsistency, non-critical evidence missing                   |

Severity thực tế phải tuân theo organization policy nếu có.

---

# 41. L1 Remediation Boundary

Một junior/L1 operator không nên “creative fix”.

Ví dụ:

```text
App cannot connect PostgreSQL
```

Evidence:

```text
DNS correct
TCP 5432 reachable
pg_isready accepting
app log = password authentication failed
```

Bạn không nên tự:

```sql
ALTER ROLE appuser PASSWORD 'newpassword';
```

nếu credential management/change authorization không thuộc quyền.

Correct action có thể là:

```text
preserve evidence
identify authentication failure
check recent change
escalate to DB/application owner
```

Đó là một câu trả lời tốt hơn trong practical exam so với làm hệ thống hỏng thêm.

---

# 42. Peer / Mentor Configuration Review

Assignment Day 21 yêu cầu peer/mentor configuration review.

Bạn cần đưa reviewer:

| Item                    | Reviewer cần thấy                             |
| ----------------------- | --------------------------------------------- |
| Architecture            | Components và network flow                    |
| Configuration inventory | Host/service/ports/version/config locations   |
| Security                | identities, permissions, SG, secrets handling |
| Availability            | services and endpoints                        |
| Database                | connectivity + backup/recovery                |
| Monitoring              | targets/dashboard/alerts                      |
| Change                  | deployment/change evidence                    |
| Recovery                | rollback procedure                            |
| Open gaps               | finding + owner + status                      |
| Handover                | enough information for next operator          |

Reviewer không nên chỉ ký:

```text
Looks good
```

Mà phải trace:

```text
requirement → evidence → decision
```

---

# 43. Configuration Review Record

Một format tốt:

| ID     | Requirement         | Expected          | Actual      | Evidence           | Status       | Action       |
| ------ | ------------------- | ----------------- | ----------- | ------------------ | ------------ | ------------ |
| CFG-01 | Tomcat service user | dedicated account | tomcat      | systemctl evidence | PASS         | —            |
| CFG-02 | App endpoint        | reachable         | reachable   | curl evidence      | PASS         | —            |
| CFG-03 | DB connectivity     | works             | works       | psql evidence      | PASS         | —            |
| CFG-04 | Prometheus target   | UP                | DOWN        | screenshot/query   | FAIL         | investigate  |
| CFG-05 | Backup restore      | tested            | no evidence | —                  | NOT VERIFIED | test restore |

Điểm quan trọng nhất:

> Status phải xuất phát từ evidence, không từ cảm giác.

---

# 44. Close Gaps — Ví dụ hoàn chỉnh

Giả sử finding:

```text
Prometheus target node_exporter = DOWN
```

Đừng restart tất cả.

### Symptom

```text
up{job="node_exporter"} = 0
```

### Expected

```text
up = 1
```

### Scope

Kiểm tra:

```text
one target?
all targets?
```

### Host-side evidence

```bash
systemctl status <NODE_EXPORTER_SERVICE>
```

```bash
ss -lnt | grep ':9100'
```

```bash
curl -fsS http://127.0.0.1:9100/metrics | head
```

Nếu local PASS:

```text
exporter process good
local socket good
endpoint good
```

Tiếp theo investigate:

```text
Prometheus scrape address
routing
SG/firewall
Prometheus configuration
```

### Config validation

```bash
promtool check config <PROMETHEUS_CONFIG>
```

### Root cause example

```text
Prometheus configured old target IP.
```

### Remediation

Chỉ sửa nếu:

```text
correct new IP confirmed
change authorized
rollback known
```

### Revalidate

```promql
up{job="node_exporter"}
```

phải trở về:

```text
1
```

### Evidence

Record:

```text
before
change
after
root cause
validation
```

Đây là “close gap”, không phải đơn giản “service giờ chạy rồi”.

---

# 45. Rollback Readiness

Trước final acceptance, bạn phải trả lời được:

```text
Nếu deployment vừa làm hỏng application, rollback thế nào?
```

Một rollback tốt phải biết:

```text
previous artifact
previous config
database compatibility
commands/process
owner
verification
```

Ví dụ conceptual:

```text
release N
   ↓ failure
stop/safely transition
   ↓
restore artifact N-1
   ↓
restore compatible configuration
   ↓
start
   ↓
validate service
   ↓
validate endpoint
   ↓
validate DB connectivity
   ↓
validate monitoring
```

### Database warning

Application rollback không đồng nghĩa database rollback luôn an toàn.

Nếu release mới đã chạy destructive/non-backward-compatible migration:

```text
old application
+
new database schema
```

có thể không tương thích.

Đó có thể là escalation point.

---

# 46. Operational Handover

Một handover tốt phải cho người nhận biết:

```text
What is running?
What changed?
What was verified?
What is abnormal?
What remains pending?
What should be monitored?
What should be done if it fails?
Who owns escalation?
```

Không bàn giao:

> “Everything fine, check logs if issue.”

---

# 47. Handover rehearsal

Bạn hãy tập nói như một operator:

> Environment hiện đang operational. Application service đang active và endpoint validation thành công. PostgreSQL connectivity đã được kiểm tra bằng approved application role. Prometheus đang scrape các required targets và alert configuration đã validate. Không có failed critical systemd unit. Có một known gap là backup recovery evidence chưa được refresh cho release hiện tại; ticket GAP-03 đã được mở và đang chờ owner xác nhận. Nếu application availability giảm, runbook yêu cầu kiểm tra client/network → EC2/Linux → socket → Tomcat → application → PostgreSQL trước khi restart. Rollback artifact trước đã được xác định và không được thực hiện nếu chưa có approved change.

Đây mới là handover có giá trị.

---

# 48. Final Evidence Package

Một package hợp lý:

```text
day21-final-evidence/
├── 00-scope/
│   ├── review-start.txt
│   └── scope.txt
├── 01-architecture/
│   └── architecture.txt
├── 02-inventory/
│   ├── os-inventory.txt
│   └── component-inventory.txt
├── 03-linux/
│   ├── capacity.txt
│   ├── service-status.txt
│   └── ports.txt
├── 04-aws/
│   ├── instance.json
│   ├── security-groups.json
│   └── resource-review.txt
├── 05-java-tomcat/
│   ├── java-version.txt
│   ├── tomcat-status.txt
│   └── endpoint-validation.txt
├── 06-postgresql/
│   ├── readiness.txt
│   ├── connectivity.txt
│   └── backup-validation.txt
├── 07-monitoring/
│   ├── prometheus-config-check.txt
│   ├── target-validation.txt
│   └── alert-validation.txt
├── 08-security-cost/
│   └── review.txt
├── 09-change-rollback/
│   ├── release-record.txt
│   └── rollback-plan.txt
├── 10-gaps/
│   └── gap-register.txt
├── 11-handover/
│   └── handover.txt
└── 12-mock/
    └── assessment.txt
```

Đây là **Supplementary structure**; syllabus yêu cầu final evidence package nhưng không quy định tên directory cụ thể.

---

# 49. Evidence Integrity

Sau khi package hoàn tất:

```bash
find ~/day21-final-evidence \
  -type f \
  ! -name manifest.sha256 \
  -exec sha256sum {} \; \
  > ~/day21-final-evidence/manifest.sha256
```

Sau đó có thể verify:

```bash
sha256sum -c ~/day21-final-evidence/manifest.sha256
```

Điều này giúp phát hiện evidence đã thay đổi sau khi package hoàn tất.

Không phải security-grade signing; nó là integrity checking cơ bản.

---

# 50. Operational Acceptance Matrix cuối cùng

Trước khi ghi “READY”, review:

| Gate            | PASS condition                                               |
| --------------- | ------------------------------------------------------------ |
| Infrastructure  | Correct AWS/Linux resources and baseline                     |
| Access          | Approved identities and least privilege                      |
| Network         | Required paths work, unnecessary exposure controlled         |
| Compute         | Capacity acceptable, no unexplained failed critical services |
| Java/Tomcat     | Correct runtime/config/service                               |
| Application     | Approved endpoint validation succeeds                        |
| Database        | Connectivity, permissions and health validated               |
| Backup          | Backup evidence exists and recovery confidence acceptable    |
| Monitoring      | Required targets/dashboards/alerts operational               |
| Security        | No unresolved unacceptable exposure                          |
| Cost            | Unused/cost-risk resources reviewed                          |
| Change          | Deployment/change documented                                 |
| Rollback        | Procedure understood and feasible                            |
| Runbook         | Routine operation documented                                 |
| Troubleshooting | Escalation path known                                        |
| Handover        | Next operator can safely continue                            |
| Evidence        | Required proof collected                                     |
| Gaps            | Closed, mitigated or formally escalated                      |

Một blocker quan trọng có thể khiến toàn bộ system:

```text
NOT READY
```

dù 14/15 mục còn lại PASS.

---

# 51. Common Anti-Patterns

Đây là những lỗi Day 21 rất dễ bị đánh giá thấp trong practical exam:

| Anti-pattern                                | Vì sao sai                                  |
| ------------------------------------------- | ------------------------------------------- |
| Service active → application OK             | Chưa test endpoint                          |
| Port open → application OK                  | Listener không chứng minh business behavior |
| `pg_isready` PASS → DB fully OK             | Chưa chứng minh app authentication/query    |
| Dashboard opens → monitoring OK             | Panels có thể No data                       |
| Backup file exists → recovery OK            | Chưa chứng minh restore                     |
| Restart first                               | Destroy evidence, có thể tăng outage        |
| chmod 777 để fix permission                 | Security violation, không hiểu root cause   |
| Run as root để fix                          | Violates least privilege                    |
| Open SG to `0.0.0.0/0` để debug             | Blast radius/security                       |
| Dump environment into evidence              | Có thể leak secrets                         |
| Fix without ticket                          | Breaks change control                       |
| “No evidence” → PASS                        | Sai logic acceptance                        |
| Unknown issue → improvise production change | Vượt L1 boundary                            |

---

# 52. Troubleshooting Framework bạn phải thuộc

Trong practical exam hoặc work thực tế:

```text
1. Symptom
2. Expected behavior
3. Recent changes
4. Scope
5. Failing layer
6. Evidence
7. Hypotheses
8. Safest test
9. Root cause
10. L1-safe remediation OR escalation
11. Validation
12. Preserve evidence
13. Prevention / rollback / handover
```

Ví dụ không tốt:

> Web không vào → restart Tomcat.

Ví dụ tốt:

```text
Symptom:
external client receives connection timeout.

Expected:
HTTPS endpoint should respond.

Scope:
all external clients affected.

Evidence:
DNS resolves correctly.
EC2 running.
Tomcat local endpoint works.
Remote connection fails.

Likely failing layer:
network/security path.

Next safe check:
SG ingress / route / firewall.

No restart yet.
```

Đó là administrator thinking.

---

# 53. Mock Theory Assessment

Đây là **bài mock do mình xây dựng để luyện Day 22**, không phải format chính thức của final exam.

Bạn nên tự trả lời trước khi xem phần đáp án bên dưới.

| #   | Câu hỏi                                                                |
| --- | ---------------------------------------------------------------------- |
| 1   | `systemctl is-active` và `systemctl is-enabled` khác nhau thế nào?     |
| 2   | Vì sao `ss` PASS chưa chứng minh application healthy?                  |
| 3   | Configuration baseline là gì?                                          |
| 4   | Configuration drift là gì?                                             |
| 5   | `pg_isready` chứng minh được và không chứng minh được gì?              |
| 6   | Vì sao backup existence không bằng recovery readiness?                 |
| 7   | `up == 0` trong Prometheus gợi ý điều gì?                              |
| 8   | Vì sao không nên chạy `env` vào evidence file?                         |
| 9   | Khi nào L1 phải escalate thay vì sửa?                                  |
| 10  | Operational acceptance khác functional acceptance thế nào?             |
| 11  | Nếu localhost curl PASS nhưng remote curl FAIL, nên ưu tiên layer nào? |
| 12  | Vì sao restart-first là anti-pattern?                                  |
| 13  | Tại sao rollback application có thể nguy hiểm sau DB migration?        |
| 14  | Handover phải chứa thông tin gì?                                       |
| 15  | “Not verified” có được coi là PASS không?                              |

### Answer key

**1.** `active` mô tả runtime state hiện tại; `enabled` mô tả boot/startup configuration.

**2.** `ss` chỉ chứng minh socket đang listening; process có thể trả 500, deploy sai app hoặc backend DB lỗi.

**3.** Approved/expected state dùng để so sánh với actual state.

**4.** Actual configuration thay đổi khỏi approved baseline.

**5.** Nó kiểm tra PostgreSQL có accept connection hay không; không chứng minh application credentials/schema/business queries hoạt động.

**6.** Backup chưa restore thử có thể corrupt, incomplete hoặc unusable.

**7.** Prometheus không scrape target thành công tại lần evaluation tương ứng; cần kiểm tra exporter, network, scrape config.

**8.** Environment có thể chứa passwords/tokens/credentials.

**9.** Khi remediation vượt quyền, cần risky/destructive change, không đủ evidence, liên quan data/security, hoặc root cause thuộc owner khác.

**10.** Functional acceptance tập trung chức năng app; operational acceptance bao gồm vận hành, monitoring, troubleshooting, recovery, security, documentation và handover.

**11.** Network path/bind/firewall/SG trước.

**12.** Restart có thể destroy diagnostic evidence và không xử lý root cause.

**13.** Old application có thể incompatible với migrated schema.

**14.** Current state, changes, validation, issues, risks, pending work, monitoring, rollback và escalation.

**15.** Không.

---

# 54. Mock Practical Assessment

Giả sử mentor cho bạn scenario:

```text
Users cannot access Java application.

Environment:
EC2 Linux
Tomcat
Java application
PostgreSQL
Prometheus/node_exporter/Grafana

You have SSH access but cannot make destructive changes without approval.
```

Bạn không được lao vào restart.

Quy trình kỳ vọng:

```text
identify symptom
        ↓
collect timestamp / hostname
        ↓
check EC2/Linux health
        ↓
check capacity
        ↓
check Tomcat service
        ↓
check listening port
        ↓
local curl
        ↓
logs
        ↓
DB connectivity if app indicates DB issue
        ↓
network / SG if local succeeds but remote fails
        ↓
monitoring correlation
        ↓
identify root cause
        ↓
safe remediation or escalation
        ↓
revalidate
        ↓
document
        ↓
handover
```

---

# 55. Practical Mini-Scenario

Mentor cho evidence:

```text
$ systemctl is-active tomcat
active

$ ss -lnt
LISTEN 0 100 *:8080

$ curl http://localhost:8080/myapp/
HTTP/1.1 500

$ journalctl -u tomcat
...
org.postgresql.util.PSQLException:
Connection refused
```

Bạn phải kết luận:

```text
Tomcat process = working
socket = working
HTTP connector = reachable
application = failing
evidence points toward PostgreSQL connectivity
```

Không được nói:

```text
Tomcat down
```

Bước tiếp:

```bash
pg_isready -h <DB_HOST> -p 5432
```

và inspect application configuration/network path.

---

# 56. Một scenario khó hơn

```text
Tomcat local curl = 200
PostgreSQL connectivity = PASS
Prometheus reports app host up
Remote browser = timeout
```

Layer có khả năng cao hơn:

```text
client
DNS
route
AWS SG/NACL
Linux firewall
bind address
```

Application/DB đang thấp hơn trong hypothesis priority vì local evidence đã PASS.

Đây chính là **scope reduction**.

---

# 57. Final Readiness — Bạn phải tự đánh giá năng lực

Trước Day 22, bạn nên có khả năng tự thực hiện chuỗi sau mà không cần người hướng dẫn từng command:

```text
Linux
  ↓
identify host
check users/permissions
process/service
network
capacity
logs

AWS
  ↓
identity
EC2
VPC/subnet
routes
SG
EBS
cost/tag checks

Java
  ↓
runtime
JVM
systemd
Tomcat
deployment
configuration
logs
endpoint

PostgreSQL
  ↓
readiness
authentication
database
sessions
capacity
backup/recovery

Monitoring
  ↓
exporter
Prometheus
targets
PromQL
Grafana
alert

Operations
  ↓
ticket
approval
evidence
rollback
handover
escalation
```

---

# 58. Exam Focus

Day 22 syllabus nói rõ final theory sẽ cover các domain đã học và practical yêu cầu deploy/configure/validate/troubleshoot/document/handover. 00_MASTER_SYLLABUS_Java_Fresher…

Vì vậy với Day 21, ưu tiên cao nhất không phải học thuộc command mà là hiểu:

```text
Command này chứng minh layer nào?
Command này KHÔNG chứng minh điều gì?
Expected result là gì?
Nếu result khác thì hypothesis tiếp theo là gì?
Có được phép sửa không?
Sau sửa validate thế nào?
Evidence nào phải giữ?
```

Đó là kiểu tư duy rất dễ phân biệt người “biết gõ lệnh” và người “biết operate hệ thống”.

---

# 59. Independent Practical Challenge

Hãy coi đây là **bài thực hành bắt buộc Day 21** của bạn.

Không thực hiện destructive changes.

Mục tiêu cuối cùng là tạo:

```text
FINAL OPERATIONAL ACCEPTANCE REPORT
```

Report phải chứng minh ít nhất:

```text
Linux health verified
AWS resource configuration reviewed
Java runtime verified
Tomcat service verified
Application endpoint verified
PostgreSQL connectivity verified
Backup/recovery evidence reviewed
Prometheus configuration validated
Monitoring targets verified
Alert evidence reviewed
Security review completed
Cost/resource review completed
Rollback readiness reviewed
Open gaps documented
Handover completed
Final readiness decision made
```

Cuối cùng bạn phải đưa ra một trong ba decision:

```text
READY
READY WITH ACCEPTED RISKS
NOT READY
```

và **không được đưa ra decision nếu không có evidence**.

---

# 60. Mastery Check cho Day 21

Day 21 chỉ nên được xem là hoàn thành khi bạn thực sự có thể chứng minh:

```text
I can review
      ↓
I can detect gaps
      ↓
I can investigate
      ↓
I know what I may safely fix
      ↓
I know when to escalate
      ↓
I can revalidate
      ↓
I can preserve evidence
      ↓
I can explain rollback
      ↓
I can hand over
```

Không phải:

```text
I read the commands
```

---

## Tóm tắt Day 21

Day 21 là **final operational readiness gate** trước Day 22.

Bạn đang chuyển từ tư duy:

```text
“Service có chạy không?”
```

sang:

```text
“System có đúng approved configuration không?
Có evidence không?
Có monitor được không?
Có troubleshoot được không?
Có recover được không?
Có rollback được không?
Có security/cost risk nào chưa xử lý không?
Một operator khác có tiếp quản được không?”
```

Và đó chính là mục tiêu tổng hợp của toàn bộ chương trình: **understand → configure → deploy → validate → monitor → troubleshoot → recover/rollback → document → escalate → hand over**.

**Bài thực hành quan trọng nhất hôm nay:** hãy thực hiện từ **Bước 1 đến Bước 49** trên lab environment của bạn và giữ nguyên evidence. Khi có output, đặc biệt là `systemctl --failed`, service status, `ss`, application `curl`, `pg_isready`, `promtool check config`, Prometheus `up`, AWS inventory và gap register, chúng ta có thể dùng chính evidence đó để thực hiện **peer/mentor review + chấm mock practical Day 21 theo kiểu exam**, thay vì chỉ học lý thuyết.
