# Module 3 — Production Operations, Integrated Troubleshooting & Final Readiness

## Day 18–21: Batch/Change/Release → Integrated Stack → Failure Troubleshooting → Operational Acceptance

## 1. Syllabus Alignment

Module 3 bao phủ toàn bộ phần **Operations Controls & Integrated Practice**:

**Day 18** yêu cầu batch scheduling, dependency và exit-code checks, release/change ticket flow, approval gates, evidence, rollback và shift handover.

**Day 19** yêu cầu tích hợp Linux + AWS + Java + PostgreSQL + monitoring, configuration baseline, security checks và cost checks.

**Day 20** yêu cầu failure injection/troubleshooting các scenario về service, port, permission, disk, database connection, alert và deployment rollback; đồng thời phân biệt L1-safe remediation với escalation.

**Day 21** yêu cầu configuration review, operational acceptance, knowledge consolidation, mock assessment và final readiness. 00_MASTER_SYLLABUS_Java_Fresher…

Module này khác Module 1–2 ở chỗ chúng ta gần như **không học thêm một technology độc lập lớn nào**. Mục tiêu là biến kiến thức Day 1–17 thành khả năng vận hành:

```text
understand
    ↓
configure
    ↓
deploy
    ↓
validate
    ↓
monitor
    ↓
troubleshoot
    ↓
recover / rollback
    ↓
document
    ↓
escalate
    ↓
handover
```

Đây chính là phần gần nhất với công việc thực tế của L1 Java/Application Administrator.

---

# PHẦN I — DAY 18

# Batch Scheduling, Change/Release Control, Evidence & Handover

# 2. Batch processing là gì?

**Batch job** là một workload được thực thi mà không cần interaction liên tục từ user.

Ví dụ:

```text
Daily billing
Nightly reconciliation
Generate monthly report
Import CSV
Export customer data
Database cleanup
Log archive
Backup
Settlement
Send notification batch
```

Khác request-response web application:

```text
User
 ↓
HTTP request
 ↓
Application
 ↓
HTTP response
```

Batch thường có flow:

```text
Trigger
  ↓
Pre-check
  ↓
Execute job
  ↓
Output/result
  ↓
Validation
  ↓
Success / Failure
```

Một batch tốt phải trả lời được:

```text
Khi nào chạy?
Ai/điều gì trigger?
Input ở đâu?
Dependency là gì?
Success được định nghĩa thế nào?
Failure được phát hiện thế nào?
Có retry không?
Có chạy lại an toàn không?
Output ở đâu?
Ai chịu trách nhiệm nếu fail?
```

---

# 3. Batch scheduling

Job có thể được trigger theo thời gian:

```text
01:00 every day
Every Sunday
First day of month
Every 15 minutes
```

hoặc theo event/dependency:

```text
File A arrives
↓
Job A
↓
Job A succeeds
↓
Job B
```

Syllabus yêu cầu **batch scheduling**, nên bạn cần hiểu ít nhất Linux `cron` và logic dependency.

---

# 4. Cron refresher

Crontab syntax:

```text
MIN HOUR DOM MONTH DOW COMMAND
```

Ví dụ:

```cron
0 2 * * * /opt/batch/daily-report.sh
```

nghĩa là:

```text
minute = 0
hour   = 2
day of month = every
month = every
day of week = every

→ chạy 02:00 mỗi ngày
```

Ví dụ:

```cron
*/15 * * * * /opt/batch/health-check.sh
```

→ mỗi 15 phút.

Ví dụ:

```cron
30 1 * * 0 /opt/batch/weekly-job.sh
```

→ 01:30 Sunday.

---

# 5. Cron environment khác interactive shell

Đây là lỗi rất thường gặp.

Script:

```bash
/opt/batch/job.sh
```

chạy thủ công:

```bash
./job.sh
```

thành công.

Cron chạy:

```text
FAILED
```

Nguyên nhân có thể do:

```text
PATH khác
HOME khác
working directory khác
environment variables không có
credentials/config không được load
relative path sai
```

Ví dụ không nên dựa vào:

```bash
psql ...
```

nếu PATH chưa chắc có binary.

Safer:

```bash
/usr/bin/psql ...
```

hoặc set explicitly:

```bash
PATH=/usr/local/bin:/usr/bin:/bin
```

---

# 6. Relative path là anti-pattern trong batch

Bad:

```bash
cat input/orders.csv
```

Cron có thể execute từ directory khác.

Better:

```bash
cat /data/import/input/orders.csv
```

hoặc:

```bash
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
```

rồi dùng path được xác định rõ.

---

# 7. Batch scheduling không đồng nghĩa batch control

Có:

```cron
0 2 * * * job.sh
```

chỉ chứng minh job được schedule.

Nó chưa giải quyết:

```text
Job trước chưa xong thì sao?
Database down thì sao?
Input chưa tới thì sao?
Job bị chạy hai lần thì sao?
Disk full thì sao?
Exit code được capture chưa?
Ai nhận notification?
```

Đây mới là operations thinking.

---

# 8. Job dependency

Giả sử:

```text
Job A = import transaction
Job B = reconciliation
Job C = generate settlement report
```

Dependency:

```text
A
↓
B
↓
C
```

C không nên chạy nếu B fail.

B không nên chạy nếu A fail.

Bad:

```cron
00 01 * * * job-a.sh
30 01 * * * job-b.sh
00 02 * * * job-c.sh
```

với assumption:

> A chắc chạy xong trong 30 phút.

Nếu A hôm đó mất 70 phút:

```text
A still running
      +
B starts
```

→ corrupted/incomplete processing có thể xảy ra.

---

# 9. Dependency nên dựa vào state, không chỉ clock

Concept tốt hơn:

```text
Job A
 ↓
SUCCESS marker / exit status
 ↓
Job B
```

Ví dụ:

```bash
/opt/batch/job-a.sh

rc=$?

if [ "$rc" -ne 0 ]; then
    echo "Job A failed with rc=$rc"
    exit "$rc"
fi

/opt/batch/job-b.sh
```

Ở đây Job B chỉ chạy nếu Job A trả success.

---

# 10. Exit code

Linux process trả về exit status.

Convention:

```text
0
→ success

non-zero
→ error/failure/special outcome
```

Sau command:

```bash
some-command
echo $?
```

`$?` chứa exit status của command **gần nhất**.

Ví dụ:

```bash
ls /tmp
echo $?
```

có thể:

```text
0
```

Trong khi:

```bash
ls /does-not-exist
echo $?
```

có thể trả non-zero.

---

# 11. Phải capture `$?` ngay

Bad:

```bash
run_batch

echo "Batch finished"

rc=$?
```

`rc` lúc này là exit status của:

```bash
echo
```

không phải `run_batch`.

Correct:

```bash
run_batch
rc=$?

echo "Batch return code: $rc"
```

---

# 12. Exit code contract

Một application/script nên có documented contract.

Ví dụ:

```text
0 = completed successfully
1 = generic failure
2 = configuration invalid
3 = dependency unavailable
4 = input missing
```

Không được tự suy đoán:

> exit 1 chắc là database error.

Nếu application/runbook không định nghĩa như vậy.

---

# 13. Shell script example

```bash
#!/usr/bin/env bash

LOG_FILE="/var/log/mybatch/import.log"

echo "$(date -Is) starting import" >> "$LOG_FILE"

/opt/app/bin/import-orders

rc=$?

if [ "$rc" -ne 0 ]; then
    echo "$(date -Is) import failed rc=$rc" >> "$LOG_FILE"
    exit "$rc"
fi

echo "$(date -Is) import succeeded" >> "$LOG_FILE"
exit 0
```

Điểm quan trọng:

```text
command
↓
capture exit code
↓
record evidence
↓
propagate failure
```

---

# 14. Pipeline exit status

Đây là một shell trap quan trọng.

Ví dụ:

```bash
some_command | grep ERROR
```

Exit status mặc định của pipeline thường liên quan tới command cuối.

Do đó:

```bash
failing_command | tee output.log
echo $?
```

có thể vô tình thấy success của `tee`.

**Supplementary / Beyond explicit syllabus:** Bash có:

```bash
set -o pipefail
```

để pipeline trả failure nếu một component thích hợp fail.

Example:

```bash
set -o pipefail

some_job | tee /var/log/job.log

rc=$?
```

Không dùng shell strict options mà chưa hiểu behavior; chúng có thể thay đổi control flow của script.

---

# 15. Dependency checks

Một batch nên thực hiện **pre-checks** trước khi thay đổi data.

Ví dụ cần kiểm tra:

```text
Input file exists?
Enough disk?
Database reachable?
Previous batch succeeded?
Another copy already running?
Required config exists?
Required filesystem mounted?
```

Ví dụ code:

```bash
INPUT="/data/input/orders.csv"

if [ ! -f "$INPUT" ]; then
    echo "ERROR: missing input $INPUT"
    exit 4
fi
```

Disk:

```bash
df -h /data
```

Database:

```bash
pg_isready -h db.internal -p 5432
```

Application endpoint:

```bash
curl -fsS http://127.0.0.1:8080/health
```

---

# 16. Dependency check phải chứng minh đúng dependency

Ví dụ:

```bash
ping db.internal
```

successful chỉ chứng minh một số network conditions.

Nó không chứng minh:

```text
PostgreSQL listening 5432
DB accepts connection
credentials valid
application schema usable
```

Better:

```bash
pg_isready -h db.internal -p 5432
```

Nhưng ngay cả vậy cũng chưa chứng minh table privilege.

Evidence có scope.

---

# 17. Duplicate batch execution

Một failure mode:

```text
Job instance A starts
         |
         | still running
         |
cron triggers again
         |
         v
Job instance B starts
```

Potential:

```text
duplicate records
double payment
double email
file corruption
locking
```

Một production batch cần strategy chống concurrent duplicate execution.

**Supplementary:** Linux thường có thể dùng `flock`.

Ví dụ:

```bash
flock -n /var/run/mybatch.lock \
    /opt/batch/job.sh
```

Nhưng lock design phải phù hợp environment, đặc biệt nếu nhiều hosts có thể chạy cùng batch.

---

# 18. Idempotency

**Idempotent operation** là operation có thể thực hiện lặp lại mà không làm trạng thái sai thêm sau lần đầu.

Ví dụ conceptual:

```text
Set status = CLOSED
```

thường dễ idempotent hơn:

```text
balance = balance - 100
```

Nếu batch fail halfway rồi rerun:

```text
row 1 processed
row 2 processed
crash
```

Rerun từ đầu có thể:

```text
process row 1 twice
```

Batch design phải biết:

```text
checkpoint?
transaction?
processed marker?
unique constraint?
idempotency key?
```

Đây là **Supplementary**, nhưng cực kỳ quan trọng cho operations.

---

# 19. Retry

Không phải failure nào cũng nên retry.

Transient:

```text
temporary network timeout
short DB connection issue
```

có thể retry.

Permanent:

```text
invalid input format
missing required column
bad configuration
permission denied
```

retry 100 lần không sửa được.

Mental model:

```text
Failure
↓
Transient?
 ├─ yes → controlled retry
 └─ no  → fail + escalate/remediate
```

---

# 20. Change vs release

**Change** là controlled modification đối với production/service/configuration.

Ví dụ:

```text
change JVM heap
modify security group
deploy WAR
change Tomcat config
rotate certificate
change PostgreSQL config
```

**Release** thường là đưa một approved software/application version vào environment.

Ví dụ:

```text
orders 2.4.1
→ production
```

Một release thường là một loại change, nhưng không phải mọi change đều là software release.

---

# 21. Change ticket

Ticket không phải bureaucracy thuần túy.

Nó cung cấp:

```text
traceability
authorization
shared understanding
rollback plan
evidence
accountability
handover
```

Một change ticket tốt phải cho một operator khác khả năng hiểu:

```text
What are we changing?
Why?
Where?
When?
How?
Risk?
Expected impact?
Validation?
Rollback?
Who approved?
Current result?
```

---

# 22. Standard change flow

Mental model quan trọng:

```text
Request
  ↓
Scope
  ↓
Impact / Risk Assessment
  ↓
Implementation Plan
  ↓
Validation Plan
  ↓
Rollback Plan
  ↓
Approval
  ↓
Pre-check
  ↓
Implementation
  ↓
Post-check
  ↓
Decision
 ├───────────────┐
 ↓               ↓
SUCCESS         FAILURE
 ↓               ↓
Evidence       Rollback
 ↓               ↓
Closure        Validate recovery
                 ↓
               Evidence
                 ↓
               Closure/escalation
```

Tên bước cụ thể phụ thuộc organization; syllabus không quy định một ITSM product/process cụ thể.

---

# 23. Scope

Bad scope:

> Update Java server.

Good scope:

```text
Host: app-prod-01
Service: orders-tomcat.service
Artifact: orders-2.4.1.war
Context: /orders
Expected downtime: <= approved window
No database schema change
```

Scope giảm blast radius.

---

# 24. Implementation plan

Bad:

```text
Deploy new version.
```

Better conceptual plan:

```text
verify current version
verify host/service health
preserve current artifact/config
place approved artifact
restart/redeploy via approved procedure
validate service
validate socket
validate application
validate database-backed function
validate monitoring
```

Plan nên deterministic.

---

# 25. Approval gate

Approval gate nghĩa là:

> Không thực hiện step có controlled production impact trước khi required authorization đã được đáp ứng.

Examples:

```text
change manager approval
application owner approval
security approval
DBA approval
business approval
```

tùy change type.

Một L1 operator không được suy nghĩ:

> Tôi biết sửa nên tôi sửa luôn.

Operational capability ≠ authorization.

---

# 26. Pre-check

Pre-check trả lời:

> System trước change có ở expected state không?

Ví dụ:

```bash
systemctl is-active tomcat
```

```bash
ss -lntp | grep ':8080'
```

```bash
curl -fsS http://127.0.0.1:8080/orders/health
```

```bash
df -h
```

Nếu service đã hỏng trước change nhưng không ghi lại, sau change khó xác định:

```text
pre-existing issue
```

hay:

```text
change caused issue
```

---

# 27. Evidence

Evidence nên chứng minh:

```text
before state
action performed
after state
result
```

Evidence tốt có:

```text
timestamp
host/environment
command/action
relevant output
version
ticket/change reference
```

---

# 28. Example evidence

Không tốt:

```text
Service OK.
```

Tốt hơn:

```text
2026-10-13T02:15:17+07:00
Host: app-prod-01
Command: systemctl is-active orders
Result: active
```

Application:

```text
2026-10-13T02:15:29+07:00
GET https://orders.example.com/health
HTTP: 200
Version: 2.4.1
```

Evidence phải đủ để người khác audit reasoning.

---

# 29. Không lưu secret vào evidence

Không paste:

```text
database password
API token
private key
AWS secret access key
session cookie
```

vào ticket/screenshots.

Sanitize:

```text
Authorization: Bearer [REDACTED]
```

Nếu evidence command tự in secret, cần dùng safer command hoặc redact theo policy.

---

# 30. Rollback plan phải tồn tại trước implementation

Một rollback plan không nên được nghĩ ra sau khi release fail.

Trước change cần biết:

```text
Known-good version?
Backup ở đâu?
Old config ở đâu?
DB migration reversible?
Rollback trigger?
Rollback duration?
Ai có authority?
Validation sau rollback?
```

Ví dụ:

```text
Current = orders-2.4.0.war
New     = orders-2.4.1.war
Rollback = restore 2.4.0 + previous config
```

---

# 31. Rollback trigger

Trigger phải objective.

Ví dụ:

```text
service fails to start
health endpoint != expected status
critical DB function fails
unexpected HTTP 5xx after release
certificate/TLS validation fails
approved validation step fails
```

Không nên:

> Tôi cảm thấy deployment không ổn.

---

# 32. Failed change không có nghĩa cứ debug mãi trong production

Nếu approved window nhỏ:

```text
new release
↓
validation fails
↓
root cause unknown
```

và rollback criteria đã đạt:

```text
rollback
↓
restore service
↓
preserve evidence
↓
investigate separately
```

thường an toàn hơn:

```text
keep modifying production
```

---

# 33. Shift handover

Operations thường chạy theo shift/team.

Handover trả lời:

```text
What happened?
What is current state?
What remains?
What must next operator watch?
What actions are prohibited/pending approval?
What ticket/incident/change IDs matter?
```

Một handover tệ:

> Please monitor server.

Một handover tốt:

```text
Orders release 2.4.1 completed 02:15.
HTTP/DB smoke tests passed.
CPU and memory normal after 20 minutes.
Change CHG-xxxx remains open pending business validation at 08:00.
Do not restart app unless incident is raised; rollback artifact 2.4.0 retained.
```

---

# 34. Golden Day 18 model

```text
Schedule
↓
Dependency
↓
Pre-check
↓
Approval
↓
Execution
↓
Exit code
↓
Validation
↓
Evidence
↓
Success / rollback
↓
Ticket update
↓
Handover
```

Đây là **operations control loop**.

---

# PHẦN II — DAY 19

# Integrated Linux + AWS + Java + PostgreSQL + Monitoring

# 35. Integrated architecture

Một architecture phù hợp với toàn syllabus:

```text
                         USERS
                           |
                        HTTPS 443
                           |
                           v
                 +-------------------+
                 | Reverse Proxy /   |
                 | Java endpoint     |
                 +-------------------+
                           |
                         :8080
                           |
                           v
                 +-------------------+
                 | EC2 Linux Host    |
                 |                   |
                 | systemd           |
                 |   ↓               |
                 | Tomcat / JVM      |
                 |   ↓               |
                 | Java Application  |
                 |                   |
                 | node_exporter     |
                 +-------------------+
                           |
                      JDBC/TLS
                        :5432
                           |
                           v
                 +-------------------+
                 | PostgreSQL        |
                 +-------------------+

node_exporter
      |
      v
Prometheus
      |
      +---- Alert rules
      |
      v
Grafana
```

Bên ngoài còn:

```text
AWS IAM
VPC/subnets
route tables
Security Groups
EBS
snapshots
CloudWatch
CloudTrail
cost controls
```

---

# 36. Layer model

To troubleshoot integrated environment, nghĩ theo layers:

```text
Layer 1  Client / DNS
Layer 2  Network path
Layer 3  AWS VPC / route / SG / NACL
Layer 4  EC2/Linux
Layer 5  Filesystem / permissions
Layer 6  systemd/process
Layer 7  socket/port
Layer 8  Tomcat/JVM
Layer 9  Java application/config
Layer 10 PostgreSQL connectivity/auth/SQL
Layer 11 Logs/metrics/alerts
Layer 12 Operational control/change state
```

Một symptom ở Layer 9 không được sửa bằng random change Layer 3 nếu evidence không dẫn tới đó.

---

# 37. Configuration baseline

**Configuration baseline** là approved/known reference state mà hệ thống nên conform.

Nó trả lời:

```text
What should this system look like?
```

Nếu không có baseline, operator chỉ thấy:

```text
current state
```

nhưng không biết:

```text
correct state
```

---

# 38. Baseline categories

Một baseline integrated có thể được mô hình hóa như sau:

| Area        | Baseline cần biết                              |
| ----------- | ---------------------------------------------- |
| AWS         | account/region, VPC, subnet, route, SG, tags   |
| EC2         | instance type, AMI/OS, IP model                |
| Storage     | EBS devices, filesystem, mount point, capacity |
| Linux       | users/groups, permissions, packages            |
| Java        | Java major version, `JAVA_HOME`                |
| Tomcat      | version, `CATALINA_BASE`, port, context        |
| Application | artifact/version, profile/config               |
| systemd     | unit name, runtime user, restart policy        |
| PostgreSQL  | host, port, DB, schema, role                   |
| Monitoring  | node_exporter, Prometheus target/job, alerts   |
| Backup      | snapshot/dump strategy                         |
| Security    | access paths, secrets, least privilege         |
| Cost        | active resources, public IPs, storage/logs     |

---

# 39. Baseline vs inventory

**Inventory**:

```text
what exists?
```

**Baseline**:

```text
what should exist/configuration should be?
```

Example:

```text
Inventory:
EC2 i-123 has port 8080 listening.

Baseline:
Tomcat must listen 127.0.0.1:8080 only.
```

Nếu hiện tại:

```text
0.0.0.0:8080
```

inventory đúng nhưng baseline violated.

---

# 40. Configuration drift

Drift:

```text
Approved baseline
      |
      | someone changes production
      v
Current state differs
```

Ví dụ baseline:

```text
-Xmx1024m
```

Current:

```text
-Xmx4096m
```

nhưng không có change record.

Hoặc:

```text
SG SSH source = corporate CIDR
```

Current:

```text
0.0.0.0/0
```

Drift là security/operations risk.

---

# 41. Build sheet

Syllabus Day 19 nhắc tới approved build sheet ở lab.

Conceptually build sheet là document quy định:

```text
resource
expected value
configuration
owner
validation
```

Ví dụ:

| Parameter        | Expected                           |
| ---------------- | ---------------------------------- |
| OS               | approved Linux release             |
| Java             | approved Java 17/25 runtime        |
| App service user | `tomcat`                           |
| Tomcat HTTP      | `127.0.0.1:8080`                   |
| Public HTTPS     | `443`                              |
| PostgreSQL       | `db.internal:5432`                 |
| App database     | `appdb`                            |
| Monitoring       | node_exporter `9100` internal only |

Không được tự thay build sheet vì "setting khác có vẻ tốt hơn".

---

# 42. Security review begins with attack surface

Hỏi:

```text
What is reachable?
By whom?
With what credential?
What privilege?
What data?
```

Typical desired shape:

```text
Internet
   |
   | 443
   v
Public application/reverse proxy

Administration source
   |
   | 22 restricted
   v
EC2

Application
   |
   | 5432 restricted
   v
PostgreSQL
```

Không cần:

```text
Internet → 5432
Internet → 9100
Internet → 9090
Internet → Tomcat 8080
```

trừ architecture explicitly yêu cầu và đã được secured.

---

# 43. AWS Security Groups

Security Groups là stateful filtering mechanism gắn với EC2/network interfaces.

AWS hiện vẫn xác nhận security groups là **stateful**: response traffic của allowed connection được xử lý dựa trên connection state, không phải như một inbound request hoàn toàn mới. [Tài liệu AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-connection-tracking.html?utm_source=chatgpt.com)

Ví dụ:

```text
Inbound:
TCP 22   ← admin CIDR only
TCP 443  ← approved clients / internet if public service
TCP 8080 ← proxy SG only if architecture requires
TCP 5432 ← application SG only
```

Không dùng:

```text
0.0.0.0/0 → TCP 5432
```

cho PostgreSQL production.

AWS control guidance cũng explicitly xem unrestricted ingress tới các port như `22`, `5432`, `8080`, `3000` là common-risk configuration. [Tài liệu AWS](https://docs.aws.amazon.com/controltower/latest/controlreference/ec2-rules.html?utm_source=chatgpt.com)

---

# 44. Security Group vs NACL refresher

```text
Security Group
→ resource/ENI level
→ stateful

Network ACL
→ subnet boundary
→ stateless
```

AWS NACL rules được evaluate theo rule number và NACL không tự động allow response traffic như Security Groups. [Tài liệu AWS](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html?utm_source=chatgpt.com)

Troubleshooting network phải nhớ distinction này.

---

# 45. Security is layered

Ví dụ PostgreSQL access:

```text
Network SG
   ↓
PostgreSQL listen_addresses
   ↓
pg_hba.conf
   ↓
role authentication
   ↓
SQL privileges
```

Một layer allow không nghĩa tất cả allow.

Ví dụ:

```text
SG permits 5432
```

nhưng:

```text
pg_hba.conf rejects
```

→ connection still fails.

Ngược lại đừng sửa HBA nếu packet chưa tới server.

---

# 46. IAM security

Application/EC2 nếu cần AWS API access nên ưu tiên:

```text
IAM role
↓
temporary credentials
```

thay vì:

```text
hard-coded access key
```

trong:

```text
source code
environment file
shell script
```

Runtime identity phải có **least privilege**.

Không dùng:

```text
AdministratorAccess
```

cho application chỉ cần đọc một object/bucket/resource cụ thể.

---

# 47. Service account + IAM role

Có hai identity layers khác nhau:

```text
Linux identity:
tomcat

AWS identity:
EC2 IAM role
```

`tomcat` xác định:

```text
local filesystem/process permissions
```

IAM role xác định:

```text
AWS API authorization
```

Không nhầm hai cái.

---

# 48. Secrets review

Check:

```text
Are DB passwords in Git?
Are credentials in systemd unit?
Can all Linux users read env file?
Does log print password?
Does process command line expose secret?
```

Potential safer design:

```text
application
   |
   v
approved secret source
```

và minimal filesystem/runtime access.

---

# 49. Cost checks

Day 19 explicitly yêu cầu cost awareness.

Cost review không chỉ là:

> EC2 có chạy không?

AWS stack có thể phát sinh cost từ:

```text
EC2 runtime
EBS
snapshots
public IPv4
NAT gateway
network/data transfer
CloudWatch logs/metrics
unused Elastic IP/public IP
retained resources
```

Current AWS documentation vẫn áp dụng charge đối với public IPv4 ngoài applicable Free Tier, và AWS cost examples sử dụng **$0.005 per public IPv4 address-hour**. [Tài liệu AWS](https://docs.aws.amazon.com/vpc/latest/userguide/?utm_source=chatgpt.com)

Vì pricing thay đổi được, production cost estimate phải kiểm tra current regional pricing thay vì học thuộc một con số.

---

# 50. Stop vs terminate

Đây là critical operational distinction.

Conceptually:

```text
STOP
EC2 compute stops
instance can later start
EBS may remain
other resources may still cost
```

versus:

```text
TERMINATE
instance lifecycle ends
recovery depends on retained volumes/snapshots/config
```

Không dùng `terminate` như cách "restart mạnh".

---

# 51. Stopped does not mean zero cost

Stopped EC2 có thể vẫn giữ:

```text
EBS volumes
snapshots
certain IP/resources
other attached services
```

Do đó cost cleanup cần inventory resource, không chỉ nhìn instance state.

---

# 52. Monitoring baseline

Integrated system cần minimum monitoring:

```text
Host:
CPU
memory
disk
network
availability

Service:
systemd/process
port
HTTP health

Application:
error rate
latency
functional health if available

Database:
connectivity
connections
locks/capacity
disk

Monitoring system:
Prometheus scrape health
alert health
```

---

# 53. CloudWatch vs CloudTrail reminder

Trong AWS:

**CloudWatch** chủ yếu liên quan monitoring/metrics/logs/alarms.

**CloudTrail** ghi AWS API/account activity để phục vụ audit: ai/role/service thực hiện API request, lúc nào, nguồn IP, request/response details phù hợp. AWS current documentation vẫn định nghĩa CloudTrail theo hướng này. [Tài liệu AWS](https://docs.aws.amazon.com/awscloudtrail/latest/APIReference/?utm_source=chatgpt.com)

Ví dụ:

```text
Who changed Security Group?
```

CloudTrail có thể là evidence phù hợp.

Trong khi:

```text
EC2 CPU > threshold?
```

CloudWatch phù hợp hơn.

---

# 54. Integrated validation sequence

Khi system vừa build/deploy:

```text
AWS resource exists
↓
network path correct
↓
EC2 reachable appropriately
↓
disk mounted
↓
Linux permissions correct
↓
Java version correct
↓
systemd active
↓
port listening
↓
Tomcat context deployed
↓
HTTP endpoint works
↓
DB connection works
↓
business smoke check works
↓
metrics scraped
↓
dashboard data present
↓
alert pipeline functional
```

Đây là operational acceptance chain.

---

# PHẦN III — DAY 20

# Failure Injection & Evidence-Driven Troubleshooting

# 55. Troubleshooting is not guessing

Bad troubleshooting:

```text
site down
↓
restart everything
```

Professional troubleshooting:

```text
Symptom
↓
Expected behavior
↓
Recent changes
↓
Scope
↓
Failing layer
↓
Evidence
↓
Hypotheses
↓
Safest discriminating test
↓
Root cause
↓
Remediation
↓
Validation
↓
Evidence + prevention
```

Đây là framework quan trọng nhất toàn curriculum.

---

# 56. Symptom vs root cause

Example:

```text
Symptom:
HTTP 500
```

Potential root cause:

```text
DB password wrong
table permission lost
disk full
application bug
missing config
certificate expired
```

`HTTP 500` không phải root cause.

---

# 57. Expected behavior

Trước khi biết "hỏng", phải biết "đúng" là gì.

Example:

```text
Expected:
GET /orders/health → 200
Tomcat listens 127.0.0.1:8080
PostgreSQL at db.internal:5432
```

Nếu không có expected baseline:

```text
0.0.0.0:8080
```

có thể bị hiểu nhầm là đúng.

---

# 58. Recent changes

Câu hỏi high-value:

```text
What changed?
```

Examples:

```text
WAR deployed?
JVM upgraded?
certificate renewed?
password rotated?
Security Group changed?
disk added?
permission modified?
DB role changed?
```

Temporal correlation không tự chứng minh causation, nhưng là hypothesis mạnh.

---

# 59. Scope

Xác định:

```text
one user?
all users?
one endpoint?
all application?
one host?
all hosts?
one database?
one environment?
```

Ví dụ:

```text
One API endpoint fails
but /health works
```

khác hoàn toàn:

```text
Nothing can TCP-connect to host.
```

---

# 60. Safest test

Một test tốt:

```text
low risk
read-only where possible
specific
distinguishes hypotheses
```

Ví dụ service unavailable:

```bash
systemctl is-active orders
```

safer hơn:

```bash
systemctl restart orders
```

Inspection trước modification.

---

# 61. Troubleshooting command ladder

Typical:

```bash
systemctl status <service>
```

```bash
journalctl -u <service> --since "10 minutes ago"
```

```bash
pgrep -a java
```

```bash
ss -lntp
```

```bash
curl -v http://127.0.0.1:8080/...
```

```bash
df -h
```

```bash
df -i
```

```bash
getent hosts <host>
```

Database:

```bash
pg_isready ...
```

SQL:

```sql
SELECT ...
FROM pg_stat_activity;
```

Monitoring:

```promql
up
```

Tool choice follows hypothesis.

---

# 62. Failure scenario 1 — Service failure

Symptom:

```text
Application unavailable
```

First:

```bash
systemctl status orders
```

Possible:

```text
failed
inactive
activating
active
```

If failed:

```bash
journalctl -u orders -n 100
```

Potential causes:

```text
bad ExecStart
wrong Java path
permission denied
bad JVM option
bad application config
port conflict
missing file
dependency failure
```

---

# 63. Service active does not prove application healthy

Possible:

```text
systemctl:
active
```

but:

```text
Tomcat startup incomplete
app context failed
DB connection failure
HTTP 500
```

Continue:

```bash
ss -lntp | grep ':8080'
```

then:

```bash
curl -i http://127.0.0.1:8080/orders/
```

---

# 64. Failure scenario 2 — Port failure

Symptom:

```text
curl: connection refused
```

Check:

```bash
ss -lntp | grep ':8080'
```

Case:

```text
no result
```

Hypotheses:

```text
service down
wrong configured port
startup failure
bind failure
```

Case:

```text
another process owns :8080
```

Potential log:

```text
Address already in use
```

---

# 65. Do not kill unknown process

Seeing:

```text
PID 200 owns 8080
```

does not authorize:

```bash
kill -9 200
```

First:

```bash
ps -fp 200
```

Determine:

```text
What process?
Expected?
Owner?
Change in progress?
```

Then runbook/escalation.

---

# 66. Bind address issue

Suppose:

```bash
ss -lntp
```

shows:

```text
127.0.0.1:8080
```

Remote request directly to:

```text
10.0.1.20:8080
```

fails.

That may be **intentional secure architecture** if Nginx on same host proxies to localhost.

Do not "fix":

```text
127.0.0.1
→
0.0.0.0
```

without architecture validation.

---

# 67. Failure scenario 3 — Permission failure

Example log:

```text
Permission denied
```

Could affect:

```text
WAR
config
log directory
certificate
private key
temp directory
mount
```

Investigation:

```bash
id tomcat
```

```bash
ls -l /path/to/file
```

```bash
namei -l /path/to/file
```

`namei -l` rất useful vì access cần permissions qua parent directories.

---

# 68. Permission reasoning

To read:

```text
/opt/orders/config/app.properties
```

user cần appropriate permissions trên:

```text
/
 /opt
 /opt/orders
 /opt/orders/config
 file itself
```

File:

```text
-rw-r----- tomcat app ...
```

vẫn có thể inaccessible nếu parent:

```text
drwx------ root root /opt/orders
```

---

# 69. Never fix permission with 777

Bad:

```bash
chmod -R 777 /opt/orders
```

Consequences:

```text
unnecessary write access
tampering risk
secret exposure
loss of baseline
```

Correct workflow:

```text
identify required access
↓
inspect owner/group/mode
↓
make minimal targeted change
↓
validate
```

---

# 70. Failure scenario 4 — Disk full

Symptoms:

```text
No space left on device
cannot write log
DB error
application startup failure
```

Check:

```bash
df -h
```

Then:

```bash
df -i
```

Why `df -i`?

Filesystem can have:

```text
free bytes
but zero free inodes
```

→ cannot create new files.

---

# 71. `df` vs `du`

`df` tells:

```text
filesystem allocation/free space
```

`du` tells:

```text
space reachable through directory tree/files
```

Example:

```bash
du -xh /var/log | sort -h | tail
```

Be careful on production: full-tree scans can be expensive.

---

# 72. Deleted-open-file problem

**Supplementary:** possible case:

```text
log file deleted
but process still has descriptor open
```

`df` still shows used disk while `du` doesn't account for deleted pathname.

Check:

```bash
lsof +L1
```

if available/authorized.

Do not immediately kill process; decide safe remediation.

---

# 73. PostgreSQL disk full

Extremely important:

Do not do:

```bash
rm -rf "$PGDATA/pg_wal/"*
```

This can destroy database recoverability/integrity.

Instead identify:

```text
filesystem
DB data
WAL
logs
backup files
other workload
```

If PostgreSQL internal storage/WAL needs intervention beyond runbook:

```text
escalate DBA
```

---

# 74. Failure scenario 5 — Database connection

Use layer sequence:

```text
Hostname
↓
Network
↓
TCP :5432
↓
PostgreSQL listener
↓
HBA
↓
Authentication
↓
Database
↓
Authorization
↓
SQL
```

---

# 75. DNS test

```bash
getent hosts db.internal.example
```

If no result:

```text
DNS / resolver / hostname
```

Don't rotate password.

---

# 76. Network/listener test

```bash
pg_isready \
  -h db.internal.example \
  -p 5432
```

or approved TCP test.

If timeout:

```text
network / SG / route / firewall
```

If refused:

```text
listener/service/port
```

---

# 77. Authentication failure

Error:

```text
password authentication failed
```

Evidence proves:

```text
DNS works
network path works
PostgreSQL responding
authentication attempt reached DB
```

Scope toward:

```text
role
password
credential rotation
credential source
```

Not:

```text
EC2 route table
```

---

# 78. HBA failure

Error:

```text
no pg_hba.conf entry
```

Then examine:

```text
source IP
database
role
SSL/non-SSL
rule order
```

Do not respond:

```conf
host all all 0.0.0.0/0 trust
```

---

# 79. Permission failure

Error:

```text
permission denied for table orders
```

This proves:

```text
connection successful
authentication successful
session established
```

Now problem is:

```text
SQL authorization
```

Potential approved fix involves appropriate `GRANT`, but production privilege change typically requires owner/DBA/change control.

---

# 80. Connection pool exhaustion

Application error may resemble DB connectivity failure but DB itself is healthy.

Check PostgreSQL sessions:

```sql
SELECT
    usename,
    application_name,
    state,
    count(*)
FROM pg_stat_activity
GROUP BY usename, application_name, state
ORDER BY count(*) DESC;
```

Possible:

```text
application pool leak
pool too large
slow queries
lock waits
DB connection cap
```

Don't assume network.

---

# 81. Failure scenario 6 — Alert failure

There are multiple meanings:

```text
Metric missing
Prometheus cannot scrape
Rule not loaded
Rule expression wrong
Alert never reaches firing
Notification not sent
Notification sent but receiver failed
```

Need locate layer.

---

# 82. Exporter layer

```bash
curl -fsS http://127.0.0.1:9100/metrics
```

If fails locally:

```text
node_exporter/service/listener issue
```

If works locally but Prometheus says:

```promql
up{instance="host:9100"} == 0
```

scope:

```text
network / Prometheus target / firewall
```

---

# 83. PromQL layer

If metric exists but alert doesn't fire, manually evaluate expression.

Example:

```promql
up{job="linux-nodes"} == 0
```

Maybe actual label:

```text
job="node"
```

Then expression returns empty.

The problem is rule query, not exporter.

---

# 84. Pending vs firing

Rule:

```yaml
for: 10m
```

Condition has been true for:

```text
4 minutes
```

→ alert may correctly remain:

```text
Pending
```

Not a bug.

---

# 85. Notification layer

If rule state:

```text
Firing
```

but no message arrives:

focus on:

```text
Alertmanager
routing
labels
silence
contact point
receiver
network
credentials
```

Not node_exporter.

---

# 86. Failure scenario 7 — Bad deployment

Timeline:

```text
10:00 app healthy
10:05 v2.4.1 deployed
10:07 HTTP 500
```

Recent change strongly focuses investigation.

Check:

```text
artifact
config delta
JVM/runtime compatibility
startup logs
DB migration
permissions
certificate
dependency
```

---

# 87. Rollback vs forward-fix

**Rollback**:

```text
restore previous known-good state
```

**Forward fix**:

```text
modify new release/config to repair it
```

L1 should not invent forward fix unless runbook explicitly allows it.

When time/impact matter and rollback has lower risk:

```text
rollback
```

is often preferred.

---

# 88. Rollback must restore compatible state

Bad rollback:

```text
old WAR
+
new incompatible config
```

Still broken.

True rollback might require:

```text
artifact
config
JVM setting
proxy config
certificate
permissions
database compatibility
```

---

# 89. Deployment rollback evidence

Before:

```text
v2.4.0 healthy
```

Change:

```text
v2.4.1 deployed
```

Failure:

```text
HTTP 500
DB initialization exception
```

Rollback:

```text
v2.4.0 restored
```

Validation:

```text
service active
8080 listening
HTTP 200
DB-backed smoke test passed
monitoring normal
```

This is useful incident evidence.

---

# 90. Failure scenario 8 — AWS network

Application host cannot connect PostgreSQL on another EC2 instance.

Chain:

```text
App config host/IP?
↓
DNS?
↓
route?
↓
source SG outbound?
↓
DB SG inbound?
↓
NACL both directions?
↓
DB listener?
↓
pg_hba?
```

Security Groups are stateful while NACLs are stateless, so the reasoning differs. [Tài liệu AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-connection-tracking.html?utm_source=chatgpt.com)

---

# 91. L1 safe remediation

"L1" does not mean a universal industry permission set. Actual authority comes from organization/runbook.

Conceptually L1-safe actions are:

```text
read-only inspection
approved health checks
approved restart
approved rollback
known cleanup procedure
known ticket workflow
evidence collection
```

High-risk/unapproved modifications escalate.

---

# 92. Escalation criteria

Escalate when:

```text
outside authorization
uncertain root cause and high blast radius
DB corruption suspected
security incident suspected
data loss risk
schema change required
AWS architecture change required
unapproved firewall/SG modification
unknown process must be killed
rollback fails
backup/recovery required beyond runbook
```

---

# 93. A good escalation is not "please help"

Good escalation contains:

```text
symptom
impact
start time
scope
recent change
evidence
tests
results
current state
actions already taken
what assistance is needed
```

Example:

```text
Orders API returns 500 for all requests since 10:07.
Deployment 2.4.1 occurred at 10:05.
Tomcat active and 8080 listening.
Application log shows permission denied on app.orders.
DB connectivity/authentication succeeds.
No privilege changes performed by L1.
Requesting DBA/application owner review of table grants.
```

This significantly reduces MTTR.

---

# PHẦN IV — DAY 21

# Configuration Review & Operational Acceptance

# 94. Configuration review

Configuration review asks:

> Does actual environment conform to approved requirements?

Not merely:

> Does application work right now?

System can "work" while dangerously misconfigured:

```text
Tomcat running as root
DB open 0.0.0.0/0
backup absent
disk 95%
monitoring disabled
password hard-coded
```

Functional ≠ operationally acceptable.

---

# 95. Review categories

Think in this order:

```text
Architecture
↓
Identity/access
↓
Network exposure
↓
OS/service config
↓
Application/JVM config
↓
Database
↓
TLS/secrets
↓
Monitoring
↓
Backup/recovery
↓
Cost
↓
Evidence/documentation
```

---

# 96. Linux review

Questions:

```text
Correct OS/version?
Unexpected users?
Service account least privileged?
Permissions appropriate?
Required filesystem mounted?
Disk capacity healthy?
systemd unit correct?
Service enabled as expected?
Logs available?
```

Commands:

```bash
uname -a
```

```bash
cat /etc/os-release
```

```bash
id tomcat
```

```bash
systemctl status orders
```

```bash
df -h
```

```bash
findmnt
```

---

# 97. Java review

```bash
java -version
```

Then verify actual service JVM, not just administrator shell:

```bash
pgrep -a java
```

Potential mismatch:

```text
shell java = Java 25
service JAVA_HOME = Java 17
```

Both can coexist.

Review:

```text
runtime version
JVM options
heap
service user
artifact
context
port
config source
logs
```

---

# 98. Tomcat review

Check:

```text
CATALINA_HOME
CATALINA_BASE
deployed WAR
context
connector/bind
runtime account
logs
permissions
```

Question:

```text
Is 8080 supposed to be private?
```

If yes:

```text
0.0.0.0:8080
```

might be a configuration gap even though service works.

---

# 99. Database review

Check:

```text
role least privilege
DB/schema expected
pg_hba scope
SCRAM authentication
listener scope
backup exists
restore strategy known
capacity healthy
sessions normal
```

Examples:

```sql
SELECT current_database(), current_user;
```

```text
\du
```

```sql
SHOW listen_addresses;
```

```sql
SELECT *
FROM pg_hba_file_rules;
```

---

# 100. Monitoring review

Verify entire path:

```text
Exporter
↓
Prometheus target
↓
Metric
↓
Dashboard
↓
Alert rule
↓
Notification
```

Not enough:

```text
Grafana page loads.
```

---

# 101. Security review

Operational security should search for:

```text
overprivileged accounts
public exposure
weak file permissions
hard-coded secrets
unrestricted SG
service running root
TLS certificate expiry
obsolete/unsupported runtime
uncontrolled changes
```

Security review is not separate from operations; it is part of acceptance.

---

# 102. Cost review

Review whether environment retains unnecessary:

```text
running EC2
public IPv4
large EBS
snapshots
NAT
CloudWatch logs
test resources
```

Important nuance:

> "Make it private" is not automatically "make it cheaper."

For example NAT gateway architectures themselves have hourly/data processing cost. AWS recommends actual cost comparison rather than assuming NAT is always cheaper than public IPv4. [Tài liệu AWS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html?utm_source=chatgpt.com)

Security architecture comes first; then cost-optimize safely.

---

# 103. Operational acceptance

Operational acceptance means operations team has enough confidence/information to own service.

Typical acceptance concepts:

```text
build matches baseline
service validated
monitoring exists
alerts tested
backup/recovery understood
access established
documentation exists
rollback known
ownership defined
known issues recorded
security risks accepted/resolved
cost controls known
```

---

# 104. Technical success vs operational acceptance

Example:

```text
Application HTTP 200
```

Technical function exists.

But:

```text
no monitoring
no backup
unknown password owner
no rollback artifact
root service account
```

→ not operationally ready.

---

# 105. Evidence package

Final evidence should allow another engineer to reconstruct what you did and why.

Conceptually:

```text
Architecture
Configuration inventory
Versions
Network/access state
Application validation
DB validation
Monitoring validation
Backup/recovery evidence
Change record
Rollback information
Known issues
Handover
```

Do not confuse evidence volume with quality.

100 screenshots without context can be worse than 10 timestamped, targeted records.

---

# 106. Evidence quality hierarchy

Weak:

```text
"It works."
```

Better:

```text
screenshot of dashboard
```

Stronger:

```text
timestamp + environment + command/query + expected result + actual result
```

Best operational evidence ties result to acceptance criterion:

```text
Criterion:
Public HTTPS endpoint responds successfully.

Evidence:
2026-10-13 02:15 +07
curl https://orders.example.com/health
HTTP 200
Version 2.4.1
```

---

# 107. Handover completeness

End-of-project handover should leave no hidden dependency in your head.

Another operator should know:

```text
architecture
access method
service names
ports
paths
versions
dependencies
monitoring
alerts
backup
known issues
restart procedure
rollback procedure
escalation contacts/ownership
```

A service that only one person knows how to recover is an operations risk.

---

# 108. Knowledge consolidation

By Day 21, you should stop thinking:

```text
Linux chapter
AWS chapter
Java chapter
DB chapter
Monitoring chapter
```

Instead think:

```text
One production service
```

with interacting layers.

---

# PHẦN V — THE COMPLETE TROUBLESHOOTING MODEL

# 109. User cannot access application

Use this exact mental journey:

```text
Client
  |
  v
DNS resolves?
  |
  v
Network path?
  |
  v
AWS route / SG / NACL?
  |
  v
EC2 reachable?
  |
  v
Proxy listening :443?
  |
  v
TLS certificate valid?
  |
  v
Backend Tomcat :8080?
  |
  v
systemd active?
  |
  v
Java process alive?
  |
  v
Context deployed?
  |
  v
Application config correct?
  |
  v
DB reachable?
  |
  v
DB authentication?
  |
  v
DB authorization?
  |
  v
Business SQL/function works?
```

At every arrow:

```text
evidence
```

not assumption.

---

# 110. Example complete investigation

Symptom:

```text
https://orders.example.com/orders
returns 502
```

502 often suggests proxy could not get acceptable backend response.

Check public layer:

```bash
curl -vk https://orders.example.com/orders
```

Then backend locally:

```bash
curl -v http://127.0.0.1:8080/orders
```

If local connection refused:

```bash
ss -lntp | grep ':8080'
```

No listener:

```bash
systemctl status orders
```

Failed:

```bash
journalctl -u orders --since "15 minutes ago"
```

Log:

```text
Address already in use: 8080
```

Then:

```bash
ss -lntp | grep ':8080'
```

find another PID.

Now root cause direction:

```text
unexpected process occupied required port
```

not:

```text
Nginx broken
```

even though user symptom was Nginx 502.

---

# 111. Example complete DB investigation

Symptom:

```text
POST /orders → HTTP 500
```

Application log:

```text
password authentication failed for user app_runtime
```

Evidence already proves:

```text
request reaches app
app tries DB
DNS/network/listener likely work
PostgreSQL responds
auth fails
```

So do not:

```text
open port 5432 wider
```

Investigate:

```text
credential rotated?
application secret stale?
wrong DB role?
wrong environment config?
```

This is evidence-driven scoping.

---

# 112. Example permission investigation

Symptom:

```text
Tomcat startup failed:
Permission denied reading /etc/orders/app.properties
```

Check:

```bash
systemctl show orders -p User -p Group
```

Suppose:

```text
User=tomcat
Group=tomcat
```

Then:

```bash
namei -l /etc/orders/app.properties
```

Suppose file:

```text
-rw-r----- root orders
```

but `tomcat` is not in `orders` group.

Root cause:

```text
runtime account lacks required group/file access
```

Remediation should be minimal and baseline-aligned, not `chmod 777`.

---

# 113. Example monitoring investigation

Symptom:

```text
Grafana CPU panel: No data
```

First inspect Prometheus query:

```promql
node_cpu_seconds_total{instance="app01:9100"}
```

No data.

Check:

```promql
up{instance="app01:9100"}
```

returns:

```text
0
```

Test Prometheus→node target network; local exporter:

```bash
curl http://127.0.0.1:9100/metrics
```

works.

Scope becomes:

```text
node_exporter healthy locally
Prometheus cannot reach target
```

Check:

```text
target address
SG/firewall
routing
Prometheus config
```

Grafana itself was only where symptom appeared.

---

# PHẦN VI — RELEASE / INCIDENT / PROBLEM DISTINCTIONS

# 114. Change vs incident

A **change** is planned controlled modification.

An **incident** is unplanned interruption/degradation.

Example:

```text
Deploy v2.4.1
= change
```

Then service crashes:

```text
production unavailable
= incident
```

Same event can connect both processes.

---

# 115. Root cause vs remediation

Example:

```text
Symptom:
HTTP 500

Immediate remediation:
rollback

Root cause:
application v2.4.1 sends incompatible SQL
```

Restoring service does not necessarily identify root cause.

Day 20 expects you to distinguish:

```text
restore service
```

from:

```text
permanently prevent recurrence
```

---

# 116. Recovery vs rollback

**Rollback:**

```text
undo/revert a particular change
```

**Recovery:**

```text
restore service/data to usable state
```

Recovery could involve rollback, but not necessarily.

Example disk failure:

```text
restore database from backup
```

is recovery, not simply application release rollback.

---

# PHẦN VII — OPERATIONAL SAFETY RULES

# 117. Before risky action

Use:

```text
Inspect
↓
Understand
↓
Backup/preserve where appropriate
↓
Validate syntax/config
↓
Control blast radius
↓
Change
↓
Verify
↓
Rollback if needed
```

Examples of risky actions:

```text
permissions
firewall/SG
systemd
database roles
database schema
disk/filesystem
certificate
application deployment
AWS termination
```

---

# 118. Restart is a change

A restart can:

```text
interrupt traffic
lose in-memory sessions
trigger startup failure
reconnect DB pools
expose bad config
```

Therefore:

> "Have you tried restarting?" is not a professional first step when evidence can be collected safely first.

Restart may be valid remediation, but it should have reason and validation.

---

# 119. Preserve evidence before destructive recovery

Example service crash.

Before overwriting everything, capture:

```bash
systemctl status orders
journalctl -u orders --since "..."
```

If you instantly:

```bash
redeploy
restart
delete logs
```

you may destroy evidence of root cause.

---

# 120. Never trade security for temporary availability silently

Bad fixes:

```text
chmod 777
run as root
disable TLS verification
open 0.0.0.0/0
use DB superuser
disable authentication
```

They may convert:

```text
availability incident
```

into:

```text
security incident
```

---

# PHẦN VIII — FINAL CONFIGURATION BASELINE EXAMPLE

A representative system might have:

```text
AWS
Region:
  approved region

VPC:
  public/private layout according to design

Security:
  SSH restricted
  HTTPS permitted appropriately
  PostgreSQL private/restricted
  monitoring ports internal

EC2:
  tagged
  approved instance size
  EBS mounted
  cost owner known

Linux:
  application user = tomcat
  no application root execution
  controlled sudo/access

Java:
  approved Java 17/25
  expected JAVA_HOME

Tomcat:
  expected release
  port 8080 internal
  context /orders

Application:
  artifact orders-2.4.1.war
  profile production
  config externalized

JVM:
  approved Xms/Xmx

Database:
  appdb
  schema app
  runtime role limited
  SCRAM/HBA appropriate

Monitoring:
  node_exporter
  Prometheus scrape
  Grafana dashboard
  alert rule

Backup:
  PostgreSQL dump/snapshot strategy
  EBS snapshot where required

Operations:
  restart procedure
  deployment procedure
  rollback procedure
  escalation route
```

Notice đây không phải chỉ là technology list.

Đó là **operating contract**.

---

# PHẦN IX — MOCK ASSESSMENT REASONING

# 121. Scenario A

> Tomcat shows `active`, but application URL returns `Connection refused`.

Sai response:

```text
Restart Tomcat.
```

Better reasoning:

```text
active
↓
check expected listener
↓
ss -lntp
↓
is correct port bound?
```

Possible conclusions:

```text
service active but app not listening
wrong configured port
bind/startup partially failed
request is using wrong endpoint
```

---

# 122. Scenario B

> `curl localhost:8080/orders` returns 200 but public HTTPS returns 502.

This strongly scopes away from core application availability.

Focus:

```text
reverse proxy
proxy configuration
backend URL/path
proxy→backend connection
TLS/public routing
```

If same host localhost backend works, don't immediately tune JVM heap.

---

# 123. Scenario C

> Application log says `permission denied for table orders`.

Network testing is low value now.

Evidence already proves:

```text
database reached
session authenticated
SQL attempted
```

Scope:

```text
PostgreSQL authorization
```

---

# 124. Scenario D

> Prometheus `up{job="linux-nodes"}` returns `1`, but application returns 500.

`up=1` only proves exporter target scrape.

It does not prove Java app health.

Investigate application/service/database signals.

---

# 125. Scenario E

> Deployment finished successfully because `cp orders.war webapps/` returned exit code 0.

Incorrect.

It proves:

```text
copy operation succeeded
```

Need:

```text
deployment/load
service
port
HTTP
DB
log
business
monitoring
```

validation.

---

# 126. Scenario F

> Disk is 100%; PostgreSQL data directory is large. Operator proposes deleting files from `pg_wal`.

Reject.

Risk:

```text
database corruption/recovery failure
```

Collect evidence and escalate according to DB/runbook.

---

# 127. Scenario G

> Production database does not accept app connection, so operator adds `0.0.0.0/0 trust`.

Reject.

It destroys authentication/access controls.

Troubleshoot exact layer:

```text
network?
HBA?
role?
credential?
```

and make smallest approved change.

---

# 128. Scenario H

> New application release fails; operator edits source code directly on production host.

Reject.

Production should use controlled artifact/release process.

Preferred:

```text
rollback known-good
preserve evidence
escalate defect
build/test approved corrected release
```

---

# PHẦN X — EXAM FOCUS

For Day 18, bạn phải giải thích được batch scheduling khác dependency control như thế nào; vì sao exit code phải được capture ngay; vì sao `0` thường biểu thị success; cron environment khác interactive shell ra sao; dependency checks cần kiểm tra gì; why duplicate batch execution matters; release khác generic change thế nào; approval gate để làm gì; evidence tốt cần chứng minh gì; rollback plan phải tồn tại lúc nào; shift handover phải chứa current state và pending work như thế nào.

For Day 19, bạn phải vẽ được toàn flow:

```text
Client
→ AWS network
→ EC2/Linux
→ systemd
→ Java/Tomcat
→ application
→ PostgreSQL
→ node_exporter
→ Prometheus
→ Grafana/alerts
```

và giải thích configuration baseline, inventory, drift, security review, least privilege, Security Group vs NACL, public vs private exposure, AWS cost drivers, backup, monitoring baseline và integrated validation.

For Day 20, bạn phải lấy bất kỳ symptom nào và đi qua:

```text
symptom
→ expected
→ recent changes
→ scope
→ layer
→ evidence
→ hypotheses
→ safe test
→ root cause
→ remediation/escalation
→ validation
```

Bạn phải troubleshoot được ít nhất:

```text
service
port
permission
disk
DB connection
alert
deployment/rollback
```

mà không random restart hay random permission/network modifications.

For Day 21, bạn phải phân biệt:

```text
system works now
```

với:

```text
system operationally acceptable
```

và review được:

```text
configuration
security
monitoring
backup/recovery
cost
evidence
handover
```

---

# PHẦN XI — COMMANDS PHẢI NHẬN DIỆN NGAY

## Process/service

```bash
systemctl status <service>
systemctl is-active <service>
systemctl restart <service>

journalctl -u <service>
journalctl -u <service> --since "10 minutes ago"

ps -ef
pgrep -a java
```

## Network/port

```bash
ss -lntp
getent hosts <hostname>

curl -v http://...
curl -I https://...
```

## Files/permissions/storage

```bash
id
ls -l
namei -l /path/to/file

df -h
df -i
du -sh
findmnt
```

## Java

```bash
java -version
jcmd <PID> VM.command_line
```

## PostgreSQL

```bash
pg_isready -h <host> -p 5432

psql -h <host> -U <role> -d <db>
```

```sql
SELECT current_database(), current_user;

SELECT *
FROM pg_stat_activity;

SELECT
    pid,
    pg_blocking_pids(pid)
FROM pg_stat_activity;
```

## Monitoring

```bash
curl http://127.0.0.1:9100/metrics
```

```promql
up
```

```promql
rate(node_cpu_seconds_total[5m])
```

## Batch

```bash
command
rc=$?

if [ "$rc" -ne 0 ]; then
    exit "$rc"
fi
```

---

# PHẦN XII — MASTER ANTI-PATTERNS

Đây là những hành vi bạn nên nhận diện ngay là nguy hiểm hoặc thiếu operational discipline:

| Anti-pattern                           | Vấn đề                             |
| -------------------------------------- | ---------------------------------- |
| Restart trước khi collect evidence     | mất clue/root-cause evidence       |
| `chmod -R 777`                         | phá least privilege                |
| Run Tomcat root                        | tăng blast radius                  |
| Open SG `0.0.0.0/0`                    | tăng attack surface                |
| Give DB `SUPERUSER`                    | overprivilege                      |
| Delete `pg_wal`                        | nguy cơ phá DB                     |
| Kill unknown process                   | nguy cơ outage khác                |
| Declare deploy success after file copy | chưa validate runtime              |
| Increase Xmx blindly                   | che root cause/có thể gây OS OOM   |
| Increase `max_connections` blindly     | có thể worsen resource pressure    |
| Disable TLS verification               | phá identity protection            |
| Store secret in Git/ticket             | credential exposure                |
| Run batch twice manually               | duplicate side effects             |
| Run change without approval            | unauthorized production change     |
| Have rollback but never verify it      | false safety                       |
| Alert configured but never tested      | notification path unproven         |
| Close ticket before validation         | change state không được chứng minh |
| Handover “everything okay”             | không actionable/auditable         |

---

# PHẦN XIII — THE MASTER INCIDENT MAP

Đây là sơ đồ quan trọng nhất Day 12–21:

```text
USER SAYS:
"APPLICATION IS DOWN"
        |
        v
+------------------+
| DNS              |
+------------------+
        |
        v
+------------------+
| Network / Route  |
+------------------+
        |
        v
+------------------+
| AWS SG / NACL    |
+------------------+
        |
        v
+------------------+
| EC2 / Linux      |
+------------------+
        |
        v
+------------------+
| Disk/Permission  |
+------------------+
        |
        v
+------------------+
| systemd          |
+------------------+
        |
        v
+------------------+
| Java process     |
+------------------+
        |
        v
+------------------+
| Port/socket      |
+------------------+
        |
        v
+------------------+
| Tomcat           |
+------------------+
        |
        v
+------------------+
| WAR / Context    |
+------------------+
        |
        v
+------------------+
| App Config       |
+------------------+
        |
        v
+------------------+
| JDBC             |
+------------------+
        |
        v
+------------------+
| PostgreSQL       |
+------------------+
        |
        v
+------------------+
| Role/HBA/SQL     |
+------------------+
        |
        v
+------------------+
| Business Func    |
+------------------+
        |
        v
+------------------+
| Metrics / Alert  |
+------------------+
```

Tại mỗi layer:

```text
Expected?
Actual?
Evidence?
```

Nếu actual khác expected:

```text
hypothesis
↓
safe test
```

---

# 130. Một nguyên tắc cực kỳ quan trọng: troubleshoot từ symptom inward

Nếu user không connect TCP được:

```text
don't start with SQL query plan
```

Nếu HTTP 500 và log nói DB permission denied:

```text
don't start with DNS
```

Nếu Grafana no data nhưng Prometheus has metric:

```text
don't restart node_exporter
```

Nếu service failed với:

```text
Permission denied reading config
```

```text
don't change Security Group
```

Troubleshooting giỏi là **giảm search space bằng evidence**.

---

# PHẦN XIV — FINAL READINESS MODEL

Đến Day 21, mục tiêu không phải thuộc nhiều command nhất.

Một candidate tốt phải nhìn scenario:

> Application unavailable after production release.

và nghĩ:

```text
What was the expected state?

What changed?

What is the impact/scope?

Was the change approved?

What evidence do I have?

Which layer is failing?

What is the safest test?

Can I fix within L1/runbook?

If yes:
  remediate

If no:
  preserve evidence and escalate

After fix:
  validate service
  validate dependency
  validate business function
  validate monitoring

Then:
  update evidence
  update ticket
  document root cause/current state
  hand over
```

Đây mới là **Administration & Operations**.

---

# 131. Toàn bộ Day 1–21 trong một mental model

Kiến thức trước đây của bạn cuối cùng hội tụ thành:

```text
                           USER
                            |
                            v
                       DNS / HTTPS
                            |
                            v
                  AWS VPC / Routing
                            |
                  Security Group/NACL
                            |
                            v
                           EC2
                            |
                    +-------+-------+
                    |               |
                 Linux           Storage
                    |               |
              user/permission      EBS
                    |
                 systemd
                    |
                    v
                 JVM/Java
                    |
                  Tomcat
                    |
                Application
                    |
                   JDBC
                    |
                    v
                PostgreSQL
          roles / HBA / SQL / locks
                    |
                    |
      +-------------+-------------+
      |                           |
      v                           v
    Logs                       Metrics
                                  |
                           node_exporter
                                  |
                                  v
                             Prometheus
                                  |
                         +--------+--------+
                         |                 |
                      Grafana            Alerts
```

Bao quanh toàn bộ architecture này là:

```text
IAM
Least privilege
Secrets
TLS
Backup
Cost control
Change control
Approval
Evidence
Rollback
Monitoring
Handover
Escalation
```

Nếu bạn chỉ hiểu các boxes nhưng không hiểu vòng ngoài, bạn biết technology nhưng **chưa phải operator**.

---

# 132. Mastery checkpoint cho Module 3

Bạn đã thực sự nắm Module 3 khi có thể tự xử lý bằng reasoning một scenario như sau:

> 02:00 batch release chạy. 02:05 application deployment hoàn tất. 02:07 Grafana báo application unavailable. systemd vẫn `active`; Tomcat port 8080 vẫn listening; `/health` trả 500; application log báo PostgreSQL authentication failure. Một DB password rotation vừa xảy ra lúc 01:55.

Bạn phải suy luận được:

```text
Host alive
↓
systemd alive
↓
Tomcat listening
↓
request reaches application
↓
application fails dependency
↓
PostgreSQL responds with authentication error
↓
likely credential/config mismatch after password rotation
```

Vì vậy **không**:

```text
reboot EC2
open SG 5432
restart PostgreSQL
chmod 777
```

mà:

```text
confirm approved credential rotation/change
↓
confirm app is using expected secret source/version
↓
follow approved credential update/redeploy/restart procedure
or rollback relevant change
↓
validate DB-backed function
↓
validate logs
↓
validate alert resolution
↓
capture evidence
↓
update incident/change
↓
handover
```

Đó chính xác là cách Day 18–21 kết nối lại với toàn bộ Day 1–17.

---

# 133. Ba Module Day 12–21 bây giờ kết nối thế nào?

Bạn có thể nén toàn bộ mười ngày thành ba câu:

```text
MODULE 1 — Day 12–14

How does the Java application run,
get configured, connect,
deploy and rollback?

            ↓

MODULE 2 — Day 15–17

How does its database work,
how do we diagnose it,
and how do we observe the system?

            ↓

MODULE 3 — Day 18–21

How do we safely operate,
change, troubleshoot, recover,
validate, document and hand over
the complete system?
```

Nếu mục tiêu của bạn là ôn gấp cho Day 22, **Module 3 là module quan trọng nhất về cách suy luận**, vì practical exam theo syllabus yêu cầu deploy, configure, validate, troubleshoot, document và hand over một integrated scenario; theory exam cũng bao phủ Linux, AWS, Java application configuration, PostgreSQL, monitoring, change controls và escalation. Module 3 chính là nơi tất cả các domain đó trở thành một workflow duy nhất. 00_MASTER_SYLLABUS_Java_Fresher…
