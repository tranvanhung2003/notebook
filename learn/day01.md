# Day 1 — Orientation & Administration Foundations

## 1. Syllabus Alignment — phạm vi bắt buộc hôm nay

Theo **MASTER_SYLLABUS**, Day 1 thuộc phần **Orientation & Administration Foundations** và có đúng hai nhóm yêu cầu:

| Thành phần          | Nội dung bắt buộc                                                                            |
| ------------------- | -------------------------------------------------------------------------------------------- |
| **Concept/Lecture** | Administration role, scope, safety, access boundaries, runbooks, evidence, lab readiness     |
| **Assignment/Lab**  | Prepare the workstation, access checklist, evidence folder, and a personal learning baseline |

Đây là phạm vi bắt buộc của Day 1. 00_MASTER_SYLLABUS_Java_Fresher… Syllabus cũng quy định đây là nguồn chương trình chính thức; kiến thức bổ sung được phép thêm để tăng khả năng thực hành/vận hành nhưng phải phân biệt với nội dung bắt buộc. 00_MASTER_SYLLABUS_Java_Fresher…

Vì vậy, hôm nay **chúng ta chưa học sâu Linux commands, AWS IAM/VPC/EC2, Tomcat, PostgreSQL hay Prometheus**. Những phần đó nằm ở Day 2 trở đi. Tuy nhiên, ta sẽ chuẩn bị workstation và tư duy vận hành để khi tới các Day đó bạn không thao tác theo kiểu “thử lệnh đến khi chạy”.

---

# 2. Mục tiêu Day 1

Sau Day 1, bạn cần hiểu và làm được bốn tầng năng lực:

| Tầng               | Bạn phải đạt được                                                                             |
| ------------------ | --------------------------------------------------------------------------------------------- |
| **Understand**     | Hiểu Administrator làm gì, trách nhiệm đến đâu, vì sao production operation phải có kiểm soát |
| **Operate safely** | Biết inspection → change → validation → evidence → rollback/escalation                        |
| **Prepare**        | Workstation, terminal, SSH/client tools và thư mục học tập đã sẵn sàng                        |
| **Prove**          | Có evidence chứng minh trạng thái máy, access readiness và mức năng lực ban đầu               |

Day 1 không đánh giá bạn bằng việc “đã đọc hết bài”. Một Administrator phải chứng minh được:

> **Tôi biết trạng thái trước khi thay đổi → tôi biết mình được phép làm gì → tôi biết chính xác mình thay đổi gì → tôi biết cách kiểm tra → tôi có bằng chứng → và tôi biết làm gì nếu thất bại.**

Đó là mindset xuyên suốt toàn bộ 22 ngày.

---

# 3. Mental Model: Administrator thực sự làm gì?

Một Fresher đôi khi hình dung Administrator là người:

> “SSH vào server rồi chạy command.”

Định nghĩa đó quá hẹp.

Administrator/Operations engineer chịu trách nhiệm duy trì một hệ thống ở trạng thái **đúng cấu hình, có thể sử dụng, an toàn, quan sát được và có thể phục hồi**.

Một vòng đời operational điển hình là:

```text
Request / Incident / Change
          |
          v
Understand scope
          |
          v
Check authorization
          |
          v
Inspect current state
          |
          v
Assess risk / blast radius
          |
          v
Prepare rollback
          |
          v
Execute approved action
          |
          v
Validate
          |
     +----+----+
     |         |
   PASS       FAIL
     |         |
 Evidence   Troubleshoot
     |         |
 Handover   Rollback / Escalate
```

Điểm cực kỳ quan trọng:

```text
Command executed ≠ task completed
```

Ví dụ:

```bash
sudo systemctl restart tomcat
```

Command trả về không có lỗi **không có nghĩa application đã hoạt động**.

Bạn còn phải kiểm tra những tầng khác, ví dụ:

```text
systemd service
      ↓
Java process
      ↓
TCP listening port
      ↓
Tomcat
      ↓
Application context
      ↓
Database connectivity
      ↓
HTTP endpoint
      ↓
Monitoring
```

Tư duy này sẽ được dùng liên tục từ Day 2 đến Day 22.

---

# 4. Administration Role — vai trò của Administrator

## 4.1 Developer và Administrator khác nhau ở đâu?

Ranh giới thực tế tùy công ty, nhưng mental model cơ bản:

| Developer                   | Administrator / Operations                           |
| --------------------------- | ---------------------------------------------------- |
| Viết application logic      | Chuẩn bị và vận hành runtime environment             |
| Build artifact              | Deploy artifact theo quy trình                       |
| Fix code bug                | Diagnose environment/runtime/availability problems   |
| Application configuration   | Quản lý configuration theo operational control       |
| Unit/integration test       | Deployment/health/operational validation             |
| Commit code                 | Quản lý service/config/change/evidence               |
| Application-level debugging | OS/network/runtime/database/monitoring investigation |

Administrator không chỉ “giữ server chạy”.

Bạn còn phải quan tâm:

```text
Availability
Security
Configuration consistency
Access control
Capacity
Logging
Monitoring
Backup
Recovery
Change management
Evidence
Escalation
Handover
Cost awareness
```

---

# 5. Integrated system mà khóa học hướng tới

**Supplementary / Beyond the explicit Day 1 syllabus**

Đây chỉ là orientation, chưa phải nội dung cấu hình của các Day sau.

Cuối chương trình, bạn sẽ vận hành một flow gần như:

```text
                    User / Client
                         |
                         v
                   DNS / Network
                         |
                         v
              AWS networking/security
                         |
                         v
                      EC2
                         |
                +--------+--------+
                |                 |
              Linux           Storage
                |
                v
             systemd
                |
                v
           Java / JVM
                |
                v
             Tomcat
                |
                v
          Java Application
                |
                v
            PostgreSQL
                |
                v
         Persistent Data


Observability:
Linux --------+
Java ---------+
Tomcat -------+----> Prometheus ----> Grafana / Alerts
Application --+
Database -----+
```

Khi xảy ra:

```text
"Website không truy cập được."
```

Administrator không được ngay lập tức kết luận:

> Tomcat hỏng.

Bạn phải xác định **failing layer**:

```text
Client?
DNS?
Network?
Security Group?
Linux?
Firewall?
Port?
systemd?
Java?
Tomcat?
Application?
Database?
```

Đây là troubleshooting mindset quan trọng nhất bạn sẽ xây dựng trong khóa học.

---

# 6. Scope — phạm vi trách nhiệm

Một Administrator cần phân biệt ba khái niệm:

### Scope of responsibility

Những hệ thống bạn chịu trách nhiệm.

Ví dụ:

```text
Server A        → in scope
Tomcat service  → in scope
Application DB  → DBA team owns it
Network router  → Network team owns it
Source code     → Development team owns it
```

### Scope of change

Bạn được phép thay đổi đến đâu.

Ví dụ ticket chỉ cho phép:

```text
Restart Tomcat on TEST server.
```

Điều đó **không đồng nghĩa** bạn được phép:

```text
edit server.xml
change JVM memory
restart PostgreSQL
reboot EC2
modify firewall
deploy new WAR
```

### Scope of access

Account của bạn có quyền kỹ thuật làm một hành động **không đồng nghĩa** với việc bạn được organizationally authorized để làm hành động đó.

Đây là khác biệt rất quan trọng:

```text
CAN do
≠
MAY do
```

---

# 7. Access Boundaries

## 7.1 Authentication và Authorization

Hai khái niệm không được nhầm.

```text
Authentication
"Bạn là ai?"

Authorization
"Bạn được phép làm gì?"
```

Ví dụ:

```text
SSH key
   |
   v
Authentication successful
   |
   v
Linux account: operator01
   |
   v
Authorization
   |
   +--> read /var/log/...     YES
   +--> restart tomcat        MAYBE
   +--> edit /etc/shadow      NO
```

---

# 8. Least Privilege

Nguyên tắc cốt lõi:

> Account/process chỉ nên được cấp lượng quyền tối thiểu cần thiết để thực hiện nhiệm vụ được giao.

Đây cũng là định nghĩa mà NIST sử dụng cho **least privilege**. [csrc.nist.gov](https://csrc.nist.gov/glossary/term/least_privilege?utm_source=chatgpt.com)

Không nên suy nghĩ:

```text
Give sudo/root because it is easier.
```

Mà phải suy nghĩ:

```text
What exact operation is required?
           ↓
What minimum permission enables it?
```

Ví dụ một operator chỉ cần:

```text
read Tomcat logs
check systemctl status
restart approved service
```

thì không có lý do mặc định để account đó được:

```text
useradd
rm -rf /
modify SSH daemon
read database passwords
modify firewall
```

---

# 9. Các loại account

Bạn cần phân biệt:

| Account                 | Ý nghĩa                                                                |
| ----------------------- | ---------------------------------------------------------------------- |
| **Named user account**  | Gắn với một cá nhân cụ thể                                             |
| **Service account**     | Dùng để chạy một service/application                                   |
| **Shared account**      | Nhiều người dùng cùng một identity; nên hạn chế vì giảm accountability |
| **Privileged account**  | Có quyền quản trị cao                                                  |
| **Break-glass account** | Account khẩn cấp, thường được kiểm soát/audit nghiêm ngặt              |

Ví dụ sau này Tomcat có thể chạy dưới:

```text
tomcat
```

Đây là **service account**.

Không nên chạy application bằng:

```text
root
```

chỉ vì “như vậy không gặp permission denied”.

Đó là anti-pattern.

---

# 10. Credential và Secret boundary

Không bao giờ đưa các dữ liệu sau vào screenshot/evidence/chat/ticket nếu không có quy trình bảo vệ phù hợp:

```text
Password
Private SSH key
AWS Secret Access Key
API token
Database password
Session token
TLS private key
```

Đặc biệt:

```text
id_rsa
*.pem private key
AWS credentials
.env containing passwords
```

không phải “evidence để chứng minh access”.

Evidence đúng là:

```text
Authentication succeeded.
```

không phải:

```text
Đây là private key tôi dùng.
```

---

# 11. Safety — tư duy thay đổi an toàn

Một Administrator giỏi không được đánh giá bằng số lượng commands nhớ được.

Một Administrator tốt thường tuân theo sequence:

```text
1. Inspect
2. Understand
3. Backup / prepare rollback where necessary
4. Change
5. Validate
6. Capture evidence
```

Không phải:

```text
1. Change
2. See what happens
```

---

## 11.1 Inspect before change

Ví dụ sau này bạn muốn sửa:

```text
/etc/tomcat/server.xml
```

Tư duy đúng:

```text
current state?
owner?
permissions?
running configuration?
backup available?
syntax validation available?
which environment?
who approved change?
rollback?
```

rồi mới sửa.

---

# 12. Blast Radius

**Blast radius** = phạm vi hậu quả nếu thao tác thất bại.

Ví dụ:

```text
Edit one test file
→ very small blast radius

Restart test Tomcat
→ application unavailable temporarily

Restart production database
→ potentially many applications impacted

Delete AWS VPC
→ potentially catastrophic
```

Trước bất kỳ change nào, hỏi:

```text
Nếu thao tác này sai thì cái gì sẽ bị ảnh hưởng?
```

---

# 13. Reversibility

Ưu tiên thay đổi có thể đảo ngược.

Ví dụ tốt:

```text
config
↓
copy backup
↓
edit config
↓
syntax check
↓
restart
↓
validation
```

Rollback:

```text
restore backup
↓
restart
↓
validation
```

Một operation mà bạn không biết rollback thế nào cần được xem là có rủi ro cao hơn.

---

# 14. One change at a time

Giả sử application hỏng và bạn đồng thời:

```text
change firewall
change JVM parameters
change DB configuration
restart Tomcat
change application.properties
```

Sau đó hệ thống chạy lại.

Bạn không biết:

```text
Change nào sửa được lỗi?
Change nào không cần thiết?
Change nào tạo technical debt?
```

Do đó:

```text
Hypothesis
    ↓
Minimal safe change
    ↓
Validate
```

rồi mới tiếp tục.

---

# 15. Expected State vs Observed State

Troubleshooting luôn bắt đầu từ sự khác biệt:

```text
Expected
vs
Observed
```

Ví dụ:

```text
Expected:
Tomcat listens on TCP 8080.

Observed:
Nothing listens on TCP 8080.
```

Đây là evidence tốt.

Còn:

```text
Tomcat looks broken.
```

không phải evidence tốt.

---

# 16. Runbook là gì?

**Runbook** là tài liệu operational hướng dẫn cách thực hiện một procedure cụ thể theo cách có thể lặp lại.

Ví dụ:

```text
Tomcat Restart Runbook
PostgreSQL Backup Runbook
Application Deployment Runbook
EC2 Provisioning Runbook
Disk Capacity Incident Runbook
```

Một runbook tốt giảm sự phụ thuộc vào trí nhớ.

---

# 17. Runbook không phải một danh sách command

Runbook kém:

```text
sudo systemctl restart tomcat
```

Runbook tốt phải cho biết ít nhất:

| Nội dung      | Câu hỏi phải trả lời                    |
| ------------- | --------------------------------------- |
| Purpose       | Procedure này nhằm đạt mục tiêu gì?     |
| Scope         | Server/service/environment nào?         |
| Preconditions | Điều kiện nào phải đúng trước khi chạy? |
| Authorization | Có ticket/approval cần thiết không?     |
| Risk          | Có thể ảnh hưởng gì?                    |
| Pre-check     | Current state là gì?                    |
| Procedure     | Các bước cụ thể                         |
| Validation    | Chứng minh thành công bằng cách nào?    |
| Rollback      | Nếu fail thì phục hồi ra sao?           |
| Evidence      | Cần lưu những gì?                       |
| Escalation    | Khi nào dừng và chuyển cấp?             |

---

# 18. Ví dụ một runbook mini

Giả sử procedure hoàn toàn vô hại:

```text
Create a Day-1 validation file
```

### Purpose

Xác nhận workstation có khả năng tạo và đọc file trong workspace cá nhân.

### Scope

Chỉ:

```text
~/java-admin-mastery/day01
```

### Pre-check

```bash
pwd
ls
```

### Change

```bash
echo "Day 1 validation" > validation.txt
```

### Validation

```bash
cat validation.txt
```

Expected invariant:

```text
File exists.
File contains the expected text.
```

### Rollback

```bash
rm validation.txt
```

### Post-validation

```bash
test ! -e validation.txt
```

Notice:

`rm` ở đây được dùng với một file do **chính bạn vừa tạo trong lab directory**. Không áp dụng thói quen `rm` tùy tiện vào system directories.

---

# 19. Evidence là gì?

Evidence là thông tin có thể dùng để chứng minh:

```text
What state existed?
What action was performed?
What result occurred?
When?
Where?
Under which context?
```

Ví dụ:

```text
command output
configuration file excerpt
log excerpt
system status
HTTP response
ticket
timestamp
before/after comparison
screenshot
monitoring graph
checksum
```

Logs là một dạng evidence rất quan trọng. NIST mô tả log management là việc tạo, truyền, lưu trữ, truy cập và xử lý log data; logs có thể phục vụ cả điều tra security incident và phát hiện operational issues. [csrc.nist.gov](https://csrc.nist.gov/pubs/sp/800/92/r1/ipd?utm_source=chatgpt.com)

---

# 20. Evidence tốt và evidence kém

Ví dụ ticket:

> “Server OK.”

Rất kém.

Một evidence tốt hơn:

```text
Host: app-test-01
Timestamp: 2026-10-12 16:30 +07
Check:
systemctl is-active tomcat

Observed:
active

HTTP validation:
curl http://localhost:8080/health

Observed:
HTTP 200
```

Điểm quan trọng:

```text
Conclusion
```

phải dựa trên:

```text
Evidence
```

chứ không phải cảm giác.

---

# 21. Example output vs Expected invariant

Từ hôm nay hãy phân biệt hai thứ này.

### Example output

Output có thể khác giữa các hệ thống.

Ví dụ:

```text
Filesystem      Size  Used Avail Use%
/dev/sda2       100G   20G   80G  20%
```

Đừng học thuộc:

```text
Disk phải là /dev/sda2
```

### Expected invariant

Điều bắt buộc phải đúng.

Ví dụ:

```text
Workspace filesystem must have sufficient free space.
```

Trong các Day tiếp theo tôi sẽ tiếp tục phân biệt hai khái niệm này.

---

# 22. Evidence phải có context

Output:

```text
active
```

một mình gần như vô dụng.

Ta cần context:

```text
Host: app-test-01
Service: tomcat.service
Command: systemctl is-active tomcat
Timestamp: ...
Result: active
```

---

# 23. Evidence không nên chứa secret

Sai:

```text
cat ~/.aws/credentials
```

rồi screenshot.

Sai:

```text
cat ~/.ssh/id_rsa
```

rồi lưu evidence.

Đúng là kiểm tra metadata hoặc access result mà không reveal secret.

Ví dụ:

```bash
ls -l ~/.ssh
```

thay vì:

```bash
cat ~/.ssh/id_rsa
```

---

# 24. L1 boundary và Escalation

Trong khóa học này bạn sẽ được đào tạo theo tư duy một fresher/L1 operator.

L1 không đồng nghĩa:

> “Không biết thì escalate.”

Và cũng không đồng nghĩa:

> “Có sudo nên tự sửa.”

Quy trình tốt:

```text
Detect symptom
      ↓
Define expected behavior
      ↓
Determine scope
      ↓
Collect evidence
      ↓
Identify likely layer
      ↓
Build hypotheses
      ↓
Safest test
      ↓
+----------------------------+
| Approved + understood fix? |
+----------------------------+
       |             |
      YES           NO
       |             |
 L1 remediation   Escalate
       |
       v
 Validation
```

---

# 25. Khi nào nên dừng và escalate?

Những tình huống điển hình:

| Tình huống                                      | Hành động                     |
| ----------------------------------------------- | ----------------------------- |
| Procedure nằm trong approved runbook            | Có thể thực hiện nếu đủ quyền |
| Root cause rõ và fix được phê duyệt             | L1 remediation                |
| Không biết blast radius                         | Stop / escalate               |
| Phải xóa production data                        | Escalate                      |
| Cần quyền ngoài authorization                   | Escalate                      |
| Suspected security compromise                   | Preserve evidence + escalate  |
| Procedure yêu cầu credential bạn không được cấp | Không bypass                  |
| Runbook không còn khớp environment              | Stop và xác minh              |
| Rollback thất bại                               | Escalate                      |
| Có nguy cơ mở rộng outage                       | Stop / escalate               |

Một kỹ năng quan trọng của Administrator là biết **không nên tiếp tục khi nào**.

---

# 26. Lab Readiness

Bây giờ bắt đầu phần thực hành bắt buộc của Day 1.

Mục tiêu cuối lab:

```text
java-admin-mastery/
└── day01/
    ├── 01-workstation/
    ├── 02-access/
    ├── 03-evidence/
    ├── 04-baseline/
    └── 05-runbook/
```

Bốn deliverable syllabus yêu cầu tương ứng:

```text
Workstation ready
Access checklist
Evidence folder
Personal learning baseline
```

Thư mục `05-runbook` là **Supplementary**, dùng để luyện lý thuyết runbook.

---

# 27. LAB 1 — Xác định workstation hiện tại

## Safety check trước lab

Hôm nay chúng ta chỉ thực hiện:

```text
read-only inspection
create files under your own learning directory
delete only temporary lab files explicitly created by you
```

Không:

```text
change firewall
change OS users
modify system services
change sudo configuration
change SSH server configuration
touch production/cloud resources
```

---

# 28. Nếu bạn dùng Windows

Mở **PowerShell** bình thường trước, chưa cần Run as Administrator.

Kiểm tra:

```powershell
$PSVersionTable.PSVersion
whoami
hostname
Get-Date
Get-Location
Get-PSDrive -PSProvider FileSystem
```

Ý nghĩa:

| Command                     | Kiểm tra           |
| --------------------------- | ------------------ |
| `$PSVersionTable.PSVersion` | PowerShell version |
| `whoami`                    | Identity hiện tại  |
| `hostname`                  | Machine name       |
| `Get-Date`                  | System time        |
| `Get-Location`              | Current directory  |
| `Get-PSDrive`               | Filesystem/storage |

---

# 29. Nếu bạn dùng Linux

Chạy:

```bash
uname -a
cat /etc/os-release
whoami
id
hostname
pwd
date
df -h .
```

Ý nghĩa:

| Command           | Kiểm tra                  |
| ----------------- | ------------------------- |
| `uname -a`        | Kernel/system information |
| `/etc/os-release` | Distribution              |
| `whoami`          | Effective user            |
| `id`              | UID/groups                |
| `hostname`        | Host identity             |
| `pwd`             | Working directory         |
| `date`            | System date/time          |
| `df -h .`         | Filesystem capacity       |

Ở Day 2 chúng ta mới học sâu các commands này.

Hôm nay mục tiêu chỉ là biết:

```text
Who am I?
Where am I?
Which machine?
Which OS?
Do I have workspace/storage?
```

---

# 30. Nếu dùng macOS

Có thể chạy:

```bash
sw_vers
uname -a
whoami
id
hostname
pwd
date
df -h .
```

Không dùng `/etc/os-release` như Linux vì macOS không phải Linux distribution.

---

# 31. LAB 2 — Kiểm tra các client tools

Không cần mọi tool phải tồn tại ngay trong Day 1.

Chúng ta đang làm **inventory**, không phải cài bằng mọi giá.

Trên Linux/macOS:

```bash
ssh -V
curl --version
git --version
java -version
aws --version
psql --version
```

Trên PowerShell:

```powershell
ssh -V
curl.exe --version
git --version
java -version
aws --version
psql --version
```

Nếu xuất hiện:

```text
command not found
```

hoặc PowerShell báo command không được recognized, **đừng coi đó là lab failure ngay lập tức**.

Ví dụ:

```text
AWS CLI
```

chỉ được syllabus yêu cầu rõ khi tới AWS training.

```text
PostgreSQL client
```

sẽ trở nên quan trọng ở database section.

Bạn chỉ cần record:

```text
Installed
Missing
Not required yet
```

---

# 32. Recommended readiness matrix

**Supplementary / Beyond explicit syllabus**

| Component             | Day 1 expectation                                             |
| --------------------- | ------------------------------------------------------------- |
| Web browser           | Ready                                                         |
| Terminal / PowerShell | Ready                                                         |
| Text editor           | Ready                                                         |
| SSH client            | Prefer Ready                                                  |
| curl                  | Prefer Ready                                                  |
| Git                   | Useful                                                        |
| WSL2                  | Recommended for Windows students, not mandatory from syllabus |
| Java                  | Inventory only                                                |
| AWS CLI               | Inventory only                                                |
| PostgreSQL client     | Inventory only                                                |
| Docker                | Optional                                                      |
| IDE                   | Optional                                                      |

Nếu máy công ty bị hạn chế installation:

> Không bypass corporate policy để cài tool.

Ghi trạng thái:

```text
Installation requires administrator approval.
```

đó mới là behavior đúng của Administrator.

---

# 33. LAB 3 — Tạo Evidence Workspace

## Linux/macOS

```bash
ROOT="$HOME/java-admin-mastery/day01"

mkdir -p \
  "$ROOT/01-workstation" \
  "$ROOT/02-access" \
  "$ROOT/03-evidence" \
  "$ROOT/04-baseline" \
  "$ROOT/05-runbook"

cd "$ROOT"

pwd
```

Kiểm tra:

```bash
find . -maxdepth 1 -type d -print
```

Nếu `find` implementation khác nhau giữa OS thì chỉ cần:

```bash
ls -la
```

---

## Windows PowerShell

```powershell
$root = "$HOME\java-admin-mastery\day01"

New-Item -ItemType Directory -Force -Path `
  "$root\01-workstation", `
  "$root\02-access", `
  "$root\03-evidence", `
  "$root\04-baseline", `
  "$root\05-runbook" | Out-Null

Set-Location $root

Get-Location
Get-ChildItem
```

---

# 34. LAB 4 — Capture terminal evidence

## Linux/macOS

Thông thường có thể dùng:

```bash
script "$HOME/java-admin-mastery/day01/03-evidence/terminal-session.txt"
```

Thực hiện các inspection commands.

Khi hoàn thành:

```bash
exit
```

`script` availability/behavior có thể khác giữa Linux distribution và macOS. Nếu không có thì ghi lại output thủ công thay vì cài package một cách mù quáng.

---

## Windows PowerShell

Bắt đầu transcript:

```powershell
Start-Transcript -Path "$HOME\java-admin-mastery\day01\03-evidence\terminal-session.txt"
```

Thực hiện checks.

Kết thúc:

```powershell
Stop-Transcript
```

---

# 35. Workstation inventory

Tạo:

```text
01-workstation/workstation-inventory.md
```

Nội dung nên theo dạng:

```text
# Day 1 Workstation Inventory

Date:
Machine:
OS:
OS version:
Architecture:
User:
Terminal:
Shell version:

Storage status:

Tools:
SSH:
curl:
Git:
Java:
AWS CLI:
psql:

Notes:
```

Không cần ghi:

```text
Password
SSH private key
AWS access key
```

---

# 36. LAB 5 — Access Checklist

Tạo:

```text
02-access/access-checklist.md
```

Checklist nên kiểm tra **khả năng và authorization**, không lưu credentials.

Bạn có thể dùng bảng:

| Access item                         | Status                       | Evidence/Note                        |
| ----------------------------------- | ---------------------------- | ------------------------------------ |
| Local workstation account           | Ready / Missing              | username only                        |
| Terminal access                     | Ready / Missing              | shell/version                        |
| Internet/docs access                | Ready / Restricted           | no password                          |
| SSH client                          | Ready / Missing              | `ssh -V`                             |
| Local admin access                  | Yes / No / Approval required | do not elevate unnecessarily         |
| Linux lab access                    | Ready / Pending              | hostname only                        |
| AWS access                          | Ready / Pending / Day 7      | no access keys                       |
| Git                                 | Ready / Missing              | version                              |
| Java                                | Ready / Pending              | version if installed                 |
| PostgreSQL client                   | Ready / Pending              | version                              |
| Sensitive credentials stored safely | Yes/No                       | never copy credentials into evidence |

Đặc biệt:

```text
Access not provisioned
```

không phải lúc nào cũng là problem.

Ví dụ trước Day 7:

```text
AWS access: Pending
```

hoàn toàn hợp lý.

---

# 37. Quy tắc credential trong lab

Một quy tắc từ hôm nay cho toàn khóa học:

**Không gửi cho tôi hoặc paste vào bài làm bất kỳ private credential nào.**

Bao gồm:

- AWS Access Key / Secret Access Key
- passwords
- private SSH keys
- session tokens
- database passwords
- TLS private keys
- API tokens

Đây là danh sách duy nhất tôi cố ý trình bày dạng bullet để bạn có thể dùng như security rule cố định.

Khi cần troubleshooting authentication, chúng ta dùng:

```text
metadata
permission
error messages
fingerprints
identity
authorization result
```

chứ không reveal secret.

---

# 38. LAB 6 — Evidence naming convention

**Supplementary / Beyond explicit syllabus**

Thay vì:

```text
screenshot1.png
new.txt
abc.log
```

dùng tên có context:

```text
day01_workstation_os.txt
day01_ssh_version.txt
day01_access_checklist.md
day01_terminal_session.txt
day01_baseline.md
```

Trong production environments có thể dùng format như:

```text
<ticket>_<host>_<check>_<timestamp>
```

Ví dụ:

```text
CHG1234_app01_tomcat-status_20261012T163000.txt
```

Không bắt buộc Day 1 phải dùng chính xác format này; mục đích là hình thành discipline.

---

# 39. LAB 7 — Personal Learning Baseline

Đây là **một trong bốn deliverable bắt buộc của syllabus**.

Tạo:

```text
04-baseline/personal-baseline.md
```

Dùng thang điểm sau:

| Score | Meaning                                                   |
| ----: | --------------------------------------------------------- |
|     0 | Chưa biết                                                 |
|     1 | Biết khái niệm                                            |
|     2 | Làm được khi có hướng dẫn                                 |
|     3 | Làm độc lập và validate được                              |
|     4 | Có thể troubleshoot, recover và giải thích cho người khác |

Đánh giá bản thân **trước khi học sâu**:

| Domain                         | Score 0–4 | Evidence / Comment |
| ------------------------------ | --------: | ------------------ |
| Linux CLI/navigation           |           |                    |
| Linux users/permissions/sudo   |           |                    |
| Linux process/systemd          |           |                    |
| Linux networking               |           |                    |
| Logs/Bash/cron                 |           |                    |
| AWS fundamentals               |           |                    |
| IAM                            |           |                    |
| VPC/networking                 |           |                    |
| EC2/EBS                        |           |                    |
| CloudWatch/cost                |           |                    |
| Java/JVM                       |           |                    |
| Tomcat                         |           |                    |
| Java application configuration |           |                    |
| PostgreSQL                     |           |                    |
| SQL troubleshooting            |           |                    |
| Prometheus                     |           |                    |
| Grafana                        |           |                    |
| Change/release procedures      |           |                    |
| Backup/recovery/rollback       |           |                    |
| Troubleshooting methodology    |           |                    |
| Evidence/handover/escalation   |           |                    |

Đừng tự nâng điểm.

Ví dụ bạn biết:

```bash
systemctl restart nginx
```

chưa đủ để tự chấm:

```text
systemd = 3
```

Score 3 nghĩa là bạn có thể:

```text
inspect
change
validate
troubleshoot
collect evidence
```

một cách độc lập.

---

# 40. LAB 8 — Runbook Practice

**Supplementary nhưng rất nên làm vì runbook nằm trong Lecture bắt buộc.**

Tạo:

```text
05-runbook/workstation-validation-runbook.md
```

Nội dung:

```text
# Workstation Validation Runbook

## Purpose
Validate that the Day 1 workstation is ready for future administration labs.

## Scope
Local personal lab workstation only.

## Preconditions
- Authorized to use machine
- Terminal available
- No production systems involved

## Risk
Low. Read-only inspection except files created under Day 1 workspace.

## Pre-check
Confirm current user, hostname and workspace.

## Procedure
1. Verify OS.
2. Verify user.
3. Verify hostname.
4. Verify disk capacity.
5. Verify terminal.
6. Inventory SSH/curl/Git/Java/AWS CLI/psql.
7. Store evidence.

## Validation
- Evidence directory exists.
- Workstation inventory exists.
- Access checklist exists.
- Baseline exists.
- No secrets are stored.

## Rollback
Delete only the Day 1 learning workspace if intentionally resetting the lab.

## Escalation
If workstation policy blocks required software, request approved installation rather than bypassing controls.
```

---

# 41. LAB 9 — Safe Failure Injection

Một Administrator phải học cả failure path.

Ta sẽ tạo một failure hoàn toàn an toàn.

Linux/macOS:

```bash
cd "$HOME/java-admin-mastery/day01/03-evidence"

echo "original state" > evidence-test.txt

cp evidence-test.txt evidence-test.txt.bak
```

Kiểm tra:

```bash
cat evidence-test.txt
cat evidence-test.txt.bak
```

Sau đó cố tình thay đổi:

```bash
echo "changed state" >> evidence-test.txt
```

Kiểm tra:

```bash
cat evidence-test.txt
```

Observed state bây giờ không còn giống original.

Rollback:

```bash
cp evidence-test.txt.bak evidence-test.txt
```

Validation:

```bash
cat evidence-test.txt
```

Expected invariant:

```text
Content must be:
original state
```

---

# 42. Bonus: chứng minh bằng checksum

**Supplementary**

Trên Linux:

```bash
sha256sum evidence-test.txt
```

Trên macOS:

```bash
shasum -a 256 evidence-test.txt
```

PowerShell:

```powershell
Get-FileHash .\evidence-test.txt -Algorithm SHA256
```

Checksum giúp chứng minh byte content đã thay đổi hay chưa.

Mental model:

```text
Original file
    |
    v
Hash A

Modify file
    |
    v
Hash B

A != B
→ content changed
```

Restore original:

```text
Hash becomes original hash again
```

nếu file được phục hồi chính xác.

Đây là một ví dụ nhỏ về **validation bằng evidence thay vì nhìn bằng mắt rồi đoán**.

---

# 43. Troubleshooting lab theo chuẩn Operations

Giả sử:

```bash
aws --version
```

trả về:

```text
command not found
```

Một fresher có thể lập tức tìm:

```text
sudo ... install aws ...
```

Một operator tốt hơn phải suy luận:

```text
Symptom
aws command unavailable

Expected?
Is AWS CLI actually required on Day 1?

Scope
Local workstation only

Evidence
command not found

Hypotheses
1. AWS CLI not installed
2. Installed but missing from PATH
3. Shell environment issue

Safest test
Check command availability / PATH

Operational decision
AWS CLI is not yet a mandatory Day 1 requirement.

Action
Record gap.
Do not make unnecessary system changes.
```

Đó chính là troubleshooting discipline.

---

# 44. Failure scenario: Permission denied

Ví dụ:

```text
Permission denied
```

Phản xạ sai:

```bash
sudo ...
```

Phản xạ đúng:

```text
What resource?
Who owns it?
What permissions?
Which user am I?
Should I have access?
Is sudo authorized?
```

Tư duy:

```text
Permission denied
      |
      v
Identity?
      |
      v
Ownership?
      |
      v
Permission?
      |
      v
Authorization?
```

`sudo` không phải universal solution.

---

# 45. Failure scenario: command not found

Đừng lập tức install.

Điều tra theo:

```text
Is command expected?
      ↓
Installed?
      ↓
PATH?
      ↓
Correct command/name/version?
      ↓
Installation authorized?
```

Đây sẽ cực kỳ quan trọng khi làm Java, AWS CLI, PostgreSQL và monitoring.

---

# 46. Failure scenario: Access denied

Cần phân biệt:

```text
Authentication failure
```

với:

```text
Authorization failure
```

Ví dụ:

```text
SSH login rejected
```

có thể là authentication.

Trong khi:

```text
sudo: user is not allowed...
```

thường liên quan authorization.

Không được gom tất cả thành:

> “Access lỗi.”

---

# 47. Security & Least Privilege Checklist

Từ Day 1 hãy hình thành các thói quen:

```text
Use individual identities
         ↓
Minimum privileges
         ↓
Do not expose credentials
         ↓
Separate service accounts
         ↓
Inspect before elevating privileges
         ↓
Maintain audit/evidence
```

NIST cũng nhấn mạnh rằng least privilege giới hạn user/process ở các authorization và resource tối thiểu để thực hiện chức năng cần thiết. [csrc.nist.gov](https://csrc.nist.gov/glossary/term/least_privilege?utm_source=chatgpt.com)

---

# 48. Operations: Before / During / After

Một framework rất hữu ích.

## Before change

Bạn phải biết:

```text
What?
Why?
Where?
Who approved?
Current state?
Risk?
Blast radius?
Backup?
Rollback?
Validation?
```

## During change

Bạn cần:

```text
follow approved runbook
avoid unnecessary deviations
record significant observations
stop if assumptions become invalid
```

## After change

Bạn cần:

```text
validate service
compare expected vs observed
capture evidence
update ticket/runbook
handover if necessary
```

---

# 49. Troubleshooting framework của toàn khóa học

Hãy ghi nhớ framework này từ Day 1:

```text
SYMPTOM
   |
   v
EXPECTED BEHAVIOR
   |
   v
RECENT CHANGES
   |
   v
SCOPE
   |
   v
FAILING LAYER
   |
   v
EVIDENCE
   |
   v
HYPOTHESES
   |
   v
SAFEST TEST
   |
   v
ROOT CAUSE
   |
   +--------------------------+
   |                          |
L1-safe                    Out of scope
remediation                  |
   |                         v
   |                      ESCALATE
   v
VALIDATION
   |
   v
EVIDENCE
   |
   v
PREVENTION / HANDOVER
```

Đừng đảo nó thành:

```text
symptom → random fixes
```

---

# 50. Ví dụ integrated troubleshooting mindset

**Supplementary orientation — chưa phải bài Day 2–17.**

Symptom:

```text
User cannot open Java application.
```

Các layer có thể kiểm tra sau này:

```text
Client
 ↓
DNS
 ↓
Network route
 ↓
AWS Security Group / NACL
 ↓
EC2
 ↓
Linux firewall
 ↓
Listening socket
 ↓
systemd service
 ↓
JVM
 ↓
Tomcat
 ↓
Application
 ↓
Application configuration
 ↓
PostgreSQL
 ↓
Logs / metrics / alerts
```

Một trong những mục tiêu cuối khóa là bạn có thể thu hẹp:

```text
Whole system
↓
Affected component
↓
Specific failing layer
↓
Evidence-backed root cause
```

---

# 51. Common Mistakes / Anti-patterns Day 1

| Anti-pattern                           | Vì sao sai                   | Tư duy đúng                     |
| -------------------------------------- | ---------------------------- | ------------------------------- |
| “Có sudo thì cứ dùng”                  | Vi phạm least privilege      | Dùng minimum required privilege |
| Chạy command lấy từ Internet không đọc | Unknown effects              | Hiểu command trước              |
| Sửa rồi mới backup                     | Rollback khó                 | Prepare rollback first          |
| Thay đổi nhiều thứ cùng lúc            | Mất root cause               | One controlled change           |
| “Không error = success”                | Không có validation          | Verify outcome                  |
| Screenshot chứa password/key           | Credential exposure          | Sanitize evidence               |
| `chmod 777` để sửa permission          | Mở quyền quá mức             | Diagnose ownership/mode         |
| Run everything as root                 | Security risk                | Service/user separation         |
| Không ghi timestamp/hostname           | Evidence thiếu context       | Contextual evidence             |
| Không biết scope vẫn tiếp tục          | Blast radius không kiểm soát | Stop/escalate                   |
| Xóa logs sau incident                  | Evidence loss                | Preserve evidence               |
| Guess root cause                       | Dẫn tới random fixes         | Evidence-driven hypotheses      |

`chmod 777` và các Linux command chi tiết sẽ học sau; hôm nay chỉ cần nhận diện mindset sai.

---

# 52. Day 1 Evidence Pack hoàn chỉnh

Sau lab, cây thư mục lý tưởng:

```text
java-admin-mastery/
└── day01/
    ├── 01-workstation/
    │   └── workstation-inventory.md
    │
    ├── 02-access/
    │   └── access-checklist.md
    │
    ├── 03-evidence/
    │   ├── terminal-session.txt
    │   └── evidence-test.txt
    │
    ├── 04-baseline/
    │   └── personal-baseline.md
    │
    └── 05-runbook/
        └── workstation-validation-runbook.md
```

Mandatory theo syllabus:

```text
✓ Workstation prepared
✓ Access checklist
✓ Evidence folder
✓ Personal learning baseline
```

Runbook file là phần thực hành bổ sung giúp chứng minh bạn hiểu Lecture.

---

# 53. Validation — làm sao biết Day 1 lab thành công?

Không dựa vào câu:

> “Em đã làm rồi.”

Dựa vào evidence:

| Requirement           | Validation                                      |
| --------------------- | ----------------------------------------------- |
| Workstation           | OS/user/hostname/storage/tool inventory exists  |
| Terminal              | Commands execute normally                       |
| SSH client            | Version check hoặc documented gap               |
| Access                | Checklist completed without secrets             |
| Evidence              | Directory exists and contains records           |
| Baseline              | Self-assessment completed                       |
| Runbook understanding | Có thể nói pre-check/change/validation/rollback |
| Security              | Evidence không chứa credentials                 |
| Operations            | Bạn biết khi nào dừng và escalate               |

---

# 54. Definition of Done cho Day 1

Bạn chỉ nên coi phần thực hành Day 1 **hoàn tất** khi có thể chứng minh:

```text
I know my workstation state.
I know which tools are ready.
I know which access is provisioned or missing.
I have a clean evidence directory.
I have recorded my learning baseline.
I understand scope and authorization.
I understand least privilege.
I understand runbooks.
I understand evidence.
I can explain pre-check → change → validation → rollback.
I know when to escalate.
```

---

# 55. Exam Focus — những gì rất dễ được hỏi

### Câu hỏi lý thuyết

**Q: Authentication và Authorization khác gì nhau?**

Authentication:

```text
Who are you?
```

Authorization:

```text
What may you do?
```

---

**Q: Least privilege là gì?**

Cấp đúng lượng quyền tối thiểu cần thiết để user/process thực hiện nhiệm vụ được giao.

---

**Q: Tại sao có root/sudo access không có nghĩa được phép thực hiện mọi thao tác?**

Vì:

```text
technical capability ≠ organizational authorization
```

---

**Q: Runbook cần gì ngoài command?**

Ít nhất phải hiểu:

```text
scope
preconditions
procedure
validation
rollback
evidence
escalation
```

---

**Q: Evidence tốt là gì?**

Evidence phải:

```text
objective
contextual
reproducible/interpretable
timestamped where relevant
free of exposed secrets
```

---

**Q: Khi procedure thất bại thì làm gì?**

Không random-fix.

Đi theo:

```text
collect evidence
determine scope
safe troubleshooting
rollback if appropriate
escalate if beyond boundary
validate recovery
```

---

# 56. Practical Exam Thinking

Giả sử examiner nói:

> Restart application service.

Câu trả lời non-operational:

```bash
sudo systemctl restart app
```

Câu trả lời có tư duy Administrator:

```text
1. Confirm correct host/environment/service.
2. Confirm authorization/change scope.
3. Inspect current status.
4. Capture pre-change evidence.
5. Confirm expected impact.
6. Confirm rollback/recovery procedure.
7. Execute approved restart.
8. Check service status.
9. Check listening port.
10. Check application endpoint.
11. Check relevant logs.
12. Capture post-change evidence.
```

Các commands chính xác để thực hiện những bước này sẽ được học lần lượt ở Linux và Java sections.

---

# 57. Knowledge Check

Hãy tự trả lời trước khi nhìn phần đáp án.

|   # | Câu hỏi                                                      |
| --: | ------------------------------------------------------------ |
|   1 | Administrator khác Developer ở điểm operational nào?         |
|   2 | Authentication khác Authorization thế nào?                   |
|   3 | Least privilege nghĩa là gì?                                 |
|   4 | Tại sao quyền `root` không đồng nghĩa với authorization?     |
|   5 | Blast radius nghĩa là gì?                                    |
|   6 | Tại sao phải inspect trước khi change?                       |
|   7 | Runbook khác một danh sách command thế nào?                  |
|   8 | Evidence tốt cần những context nào?                          |
|   9 | Tại sao không được đưa private SSH key vào evidence?         |
|  10 | Sau khi command chạy không lỗi, tại sao vẫn phải validate?   |
|  11 | Khi không hiểu blast radius, L1 nên làm gì?                  |
|  12 | Tại sao thay đổi năm configuration cùng lúc là anti-pattern? |

### Đáp án ngắn

**1.** Admin chịu trách nhiệm operational state, deployment/configuration, availability, troubleshooting, recovery và evidence; Developer chủ yếu xây application/code.

**2.** Authentication xác minh identity; Authorization quyết định permissions.

**3.** Chỉ cấp quyền tối thiểu cần thiết.

**4.** Khả năng kỹ thuật không đồng nghĩa với phê duyệt tổ chức.

**5.** Phạm vi ảnh hưởng nếu thao tác/change thất bại.

**6.** Cần baseline/current state để hiểu impact và rollback.

**7.** Runbook chứa context, scope, precondition, validation, rollback và escalation ngoài procedure.

**8.** Host/resource, command/check, timestamp khi cần, observed result và expected result.

**9.** Đó là secret; exposure có thể dẫn tới compromise.

**10.** Command success không chứng minh system outcome đã đạt.

**11.** Dừng change và xác minh/escalate.

**12.** Không biết change nào gây ra hoặc sửa vấn đề, làm troubleshooting và rollback khó.

---

# 58. Independent Practical Challenge

Sau khi làm Guided Lab, tự làm thử scenario này mà không nhìn lại hướng dẫn:

> **“Bạn vừa nhận một workstation mới để tham gia Java Administration team. Hãy chứng minh workstation sẵn sàng cho training mà không thay đổi system configuration ngoài phạm vi cần thiết.”**

Deliverable của bạn phải chứng minh được:

```text
machine identity
OS
user identity
disk/workspace
terminal
client tools
access status
missing prerequisites
evidence storage
personal baseline
security hygiene
```

Không được:

```text
paste passwords
paste private keys
arbitrarily use sudo/admin
install software merely because command is missing
```

Đây là bài thực hành rất gần tư duy operations thật.

---

# 59. Mastery Check Day 1

Bạn nên có thể giải thích trôi chảy tình huống sau:

> “Bạn được cấp sudo trên một production server. Application đang lỗi. Một đồng nghiệp bảo bạn chạy vài command để thử.”

Cách suy nghĩ đúng:

```text
Do I have an approved task?
        ↓
Which system/environment?
        ↓
What exactly is my scope?
        ↓
What is currently failing?
        ↓
What evidence exists?
        ↓
What will those commands change?
        ↓
What is the blast radius?
        ↓
Can I roll back?
        ↓
Are these actions authorized?
        ↓
Execute only if safe/approved
        ↓
Validate
        ↓
Preserve evidence
```

Nếu bạn hiểu flow này, bạn đã nắm được mục tiêu cốt lõi của Day 1.

---

# 60. Điều quan trọng nhất cần nhớ hôm nay

Day 1 không phải một ngày “giới thiệu nhẹ nhàng”. Nó xây nền tảng cho cách bạn hành xử trong **tất cả các Day sau**.

Mental model cần mang theo là:

```text
                 ADMINISTRATOR
                      |
      +---------------+---------------+
      |               |               |
     SCOPE           SAFETY         ACCESS
      |               |               |
      +---------------+---------------+
                      |
                   RUNBOOK
                      |
                 PRE-CHECK
                      |
                    CHANGE
                      |
                  VALIDATE
                      |
                   EVIDENCE
                      |
               +------+------+
               |             |
            SUCCESS        FAILURE
               |             |
            HANDOVER    TROUBLESHOOT
                             |
                      +------+------+
                      |             |
                   RECOVER       ESCALATE
```

Và công thức vận hành chúng ta sẽ sử dụng xuyên suốt khóa học:

> **Understand → Inspect → Authorize → Change safely → Validate → Evidence → Recover/Rollback → Escalate/Handover**

Đó chính là nền móng để sau này bạn có thể độc lập **configure → deploy → validate → monitor → troubleshoot → recover → document → escalate → hand over** hệ thống Linux + AWS + Java + PostgreSQL + monitoring.

### Trạng thái sau module Day 1 này

Phần **lý thuyết bắt buộc Day 1** đã được trình bày đầy đủ theo syllabus. Phần **thực hành chưa được xem là mastered cho đến khi bạn thực sự chạy lab và có evidence**. Bước tiếp theo tốt nhất là bạn thực hiện lần lượt LAB 1–9 trên máy của mình; khi gặp command/output nào khác với dự kiến, hãy gửi **command + output đã loại bỏ secrets**, và chúng ta sẽ troubleshoot theo đúng quy trình Operations thay vì chỉ tìm cách “cho nó chạy”.
