# Day 1 — Module Lý thuyết

## Infrastructure Operations Foundation + Linux/RHEL Administration Foundation

Day 1 trong syllabus có hai mục tiêu song song. Mục tiêu thứ nhất là xây dựng **operational mindset** của một Infrastructure Operations engineer: hiểu authorization, support boundary, least privilege, change control, escalation, baseline, asset inventory và evidence. Mục tiêu thứ hai là xây dựng **Linux/RHEL administration foundation**: architecture, filesystem, CLI, users/groups, permissions, `sudo`, `SSH` và secure administrative access. Ngoài ra, Day 1 phải thiết lập ngay **evidence handling baseline** dùng xuyên suốt 14 ngày. Infrastructure_Operations_14_Da…

Điểm quan trọng là chương trình này hướng tới **operational readiness**, không mặc định rằng sau một ngày người học đã có advanced production mastery. Lab phải sử dụng authorized accounts, isolated/preconfigured environments và mentor supervision phù hợp. Infrastructure_Operations_14_Da…

## 1. Syllabus coverage checklist của Module Lý thuyết Day 1

Toàn bộ các topic phải nắm trong module này gồm: \*\*Infrastructure administration roles and responsibilities; L1/L2 support boundaries; operational safety; production/non-production environment separation; change authorization; least privilege; SOP; runbook; incident escalation; access management; configuration baseline; asset inventory; audit evidence; lab readiness; Linux architecture; kernel; shell; Linux distributions; RHEL; filesystem hierarchy; CLI navigation; absolute/relative paths; file/directory operations; command syntax; pipes; redirection; command history; help utilities; safe command execution; users/groups; ownership; file permissions; `chmod`; `chown`; `chgrp`; `umask`; `sudo`; SSH authentication; SSH key management; secure administrative access; evidence ID/source/host/environment/timestamp/query/collector/ticket/size/count/SHA-256/classification; raw evidence versus working/redacted copy; hash integrity và giới hạn của hash. Infrastructure_Operations_14_Da…

Phần dưới đây sẽ giải thích toàn bộ phạm vi đó. Những topic trên là **theo syllabus**; các explanation sâu hơn về workflow, failure modes, troubleshooting và security là **kiến thức bổ sung để vận hành**.

---

# PHẦN I — INFRASTRUCTURE OPERATIONS FOUNDATION

## 2. Infrastructure Administration là gì?

### 2.1 What — Infrastructure Administration

`Infrastructure Administration` là hoạt động vận hành, duy trì, kiểm tra, thay đổi có kiểm soát và troubleshoot các thành phần tạo thành nền tảng IT.

Trong enterprise, infrastructure có thể bao gồm:

```text
Physical infrastructure
        ↓
Virtualization
        ↓
Operating systems
        ↓
Network / Firewall / Load Balancer
        ↓
Cloud infrastructure
        ↓
Storage / Backup
        ↓
Monitoring / Logging
        ↓
Identity / Access
        ↓
Applications
```

Infrastructure Operations không nhất thiết sở hữu toàn bộ các layer trên. Điểm quan trọng là phải biết **ownership boundary**.

Ví dụ một web application không hoạt động:

```text
User
  ↓
DNS
  ↓
Firewall
  ↓
Load Balancer
  ↓
Web Server
  ↓
Application
  ↓
Database
```

Infrastructure engineer không nên lập tức kết luận:

> "Server có vấn đề."

Phải xác định layer nào đang thất bại.

### 2.2 Why — tại sao cần hiểu role

Trong enterprise, một thao tác kỹ thuật đúng nhưng được thực hiện bởi sai team, sai scope hoặc không có approval vẫn có thể là một operational failure.

Ví dụ bạn biết cách restart database:

```bash
systemctl restart database-service
```

Nhưng nếu database thuộc Application/DBA team và change chưa được authorized, việc restart đó vẫn không hợp lệ.

Operational competence vì vậy gồm cả:

```text
Technical capability
        +
Authorization
        +
Scope awareness
        +
Impact awareness
        +
Evidence
        =
Safe operation
```

### 2.3 Các trách nhiệm điển hình

Một Infrastructure Operations engineer thường phải:

- kiểm tra health/status;
- thực hiện standard maintenance;
- quản lý operating system configuration trong phạm vi được giao;
- kiểm tra access;
- troubleshoot incident;
- collect logs/evidence;
- thực hiện approved change;
- verify sau change;
- rollback khi cần;
- escalate khi vượt boundary;
- cập nhật ticket;
- thực hiện shift handover.

Không phải mọi organization đều chia nhiệm vụ giống nhau.

**Support boundary luôn phụ thuộc organization, contract, runbook và authorization.**

---

# 3. L1/L2 Support Boundaries

## 3.1 L1 là gì?

`L1` thường là tuyến đầu của support.

Mục tiêu chính của L1 thường là:

```text
Detect
→ Validate
→ Classify
→ Perform safe standard checks
→ Execute approved basic runbook
→ Collect evidence
→ Resolve or Escalate
```

Một L1 engineer giỏi không phải người "thử mọi thứ cho đến khi chạy".

Một L1 engineer tốt biết:

> "Tôi đã chứng minh được điều gì, chưa chứng minh được điều gì, và boundary tiếp theo ở đâu?"

Ví dụ Linux server unreachable.

L1 có thể kiểm tra:

```text
Correct hostname?
Correct IP?
Correct environment?
Host powered on?
Network reachable?
SSH port reachable?
Authentication failure or network failure?
Other users affected?
Recent approved change?
```

Nếu phát hiện:

```text
SSH reachable
+
account denied
+
multiple users unaffected
```

thì vấn đề khác hẳn:

```text
host unreachable for everyone
```

---

## 3.2 L2 là gì?

L2 thường có phạm vi technical troubleshooting sâu hơn.

Ví dụ:

```text
Configuration validation
Service troubleshooting
Permission troubleshooting
Log correlation
Controlled configuration correction
Rollback
Advanced diagnostic collection
```

L2 thường có quyền cao hơn L1 nhưng **không có nghĩa là unlimited privilege**.

L2 vẫn phải tuân thủ:

```text
authorization
change policy
least privilege
maintenance window
rollback
audit
```

---

## 3.3 L3/vendor boundary

Một số vấn đề cần escalation tới:

```text
L3 engineering
Vendor
Security team
Network engineering
Database team
Application owner
Cloud platform owner
Hardware vendor
```

Ví dụ:

- kernel crash không rõ nguyên nhân;
- filesystem corruption;
- recurring storage controller failure;
- security incident;
- unsupported configuration;
- product defect;
- complex performance degradation;
- destructive recovery.

---

## 3.4 Boundary không nên được xác định bằng cảm giác

Không nên nghĩ:

> "Command này trông đơn giản nên L1 chắc được phép."

Boundary phải đến từ:

```text
Job responsibility
Runbook
RBAC
Change policy
Ticket scope
Manager/team ownership
Contract/SLA
```

---

## 3.5 Escalation tốt phải mang theo evidence

Escalation kém:

```text
Linux server not working.
Please check.
```

Escalation tốt:

```text
Asset: rhel-app01
Environment: PROD
Incident: INCxxxx
Impact: 40 users unable to access application

Observed:
- host responds to ICMP
- TCP/22 reachable
- SSH connection reaches authentication
- affected account receives publickey authentication failure
- alternate authorized account succeeds

Last known good:
10:05 +07:00

First observed:
10:21 +07:00

Recent change:
CHGxxxx completed at 10:15 +07:00

Evidence:
EVID-001 ... EVID-005

Action taken:
Read-only verification only.

Reason for escalation:
SSH identity/access configuration is outside L1 modification scope.
```

Đó là operational thinking.

---

# 4. Operational Safety

Đây là một trong những chủ đề quan trọng nhất của toàn chương trình.

## 4.1 Operational safety nghĩa là gì?

`Operational safety` là khả năng thực hiện công việc mà giảm tối đa nguy cơ:

```text
wrong host
wrong environment
wrong command
wrong parameter
wrong file
wrong account
wrong timing
unexpected blast radius
data loss
service outage
security exposure
unrecoverable change
```

Không có command nào "an toàn tuyệt đối". Safety phụ thuộc context.

Ví dụ:

```bash
rm test.txt
```

trong disposable lab khác hoàn toàn với:

```bash
rm important_file
```

trên production server.

---

# 5. Read-only trước, write sau

Một principle rất hữu ích:

```text
Observe before changing.
```

Ví dụ cần sửa permission.

Không nên bắt đầu:

```bash
chmod 777 application_directory
```

Workflow tốt hơn:

```text
1. Identify symptom
2. Identify object
3. Inspect existing permission
4. Inspect ownership
5. Inspect user/group identity
6. Determine required access
7. Determine minimum change
8. Obtain authorization
9. Change
10. Verify
11. Record evidence
```

Read-only commands thường có blast radius thấp hơn configuration-changing commands.

Ví dụ:

```bash
ls -ld /path
stat /path
id username
getent group groupname
```

trước khi dùng:

```bash
chmod
chown
chgrp
```

---

# 6. Pre-check và Post-check

Mọi operational change tốt nên có:

```text
Pre-check
    ↓
Change
    ↓
Post-check
```

Ví dụ thay đổi file permission.

### Pre-check

```bash
stat /srv/team/report.txt
```

Ghi nhận:

```text
owner
group
mode
timestamp
```

### Change

```bash
chmod 640 /srv/team/report.txt
```

### Post-check

```bash
stat /srv/team/report.txt
```

Sau đó test effective access bằng đúng account cần xác minh.

Không nên kết luận thành công chỉ vì command trả về exit code `0`.

Ví dụ:

```bash
chmod 640 file
```

success chỉ chứng minh:

> `chmod` đã thực hiện thay đổi được yêu cầu.

Nó chưa chứng minh:

> Application hiện truy cập file đúng như business requirement.

---

# 7. Stop Conditions

`Stop condition` là điều kiện bắt buộc dừng thao tác.

Ví dụ:

```text
Expected:
Only /srv/lab/test.conf will be modified.

Observed:
Command indicates recursive change on /srv.
```

→ **STOP**.

Một số stop conditions phổ biến:

```text
Target identity mismatch
Unexpected production host
Unexpected number of affected objects
Loss of management access
Pre-check not matching change plan
Backup/config export unavailable when required
Error outside documented runbook
Unexpected service dependency
Security impact discovered
Rollback no longer viable
```

Stop không phải thất bại.

Trong Infrastructure Operations:

> Dừng đúng lúc thường tốt hơn "cố sửa tiếp".

---

# 8. Production và Non-Production Separation

## 8.1 Environment là gì?

Thông thường có thể gặp:

```text
LAB
DEV
TEST
QA
UAT
STAGING
PRE-PROD
PROD
DR
```

Tên gọi phụ thuộc organization.

### PROD

`Production` phục vụ workload thật.

Sai sót có thể ảnh hưởng:

- business;
- customer;
- revenue;
- compliance;
- data integrity;
- SLA.

### Non-production

Được dùng để:

- development;
- integration;
- testing;
- training;
- validation.

Nhưng **non-production không có nghĩa là không cần security**.

Staging có thể vẫn chứa:

```text
sensitive configuration
production-like credentials
customer-like data
internal certificate
network connectivity
```

---

## 8.2 Environment separation

Separation có thể được thực hiện bằng:

```text
different accounts
different networks
different credentials
different DNS domains
different bastion hosts
different cloud accounts
different RBAC
different naming
different change procedures
```

Một naming scheme tốt giúp giảm mistake:

```text
rhel-lab-01
rhel-dev-01
rhel-prod-01
```

nhưng tên host **không phải verification duy nhất**.

---

## 8.3 "Wrong host" là failure mode rất thực tế

Trước privileged action nên xác minh:

```bash
hostname
whoami
id
pwd
```

Khi relevant:

```bash
cat /etc/os-release
```

Tư duy cần hình thành:

```text
WHO am I?
WHERE am I?
WHAT am I about to change?
WHY?
UNDER WHICH ticket/approval?
HOW will I verify?
HOW will I back out?
```

---

# 9. Change Authorization

## 9.1 Change là gì?

`Change` là modification đối với controlled environment.

Ví dụ:

```text
changing permissions
creating privileged access
changing SSH configuration
installing software
restarting critical service
modifying firewall rule
changing route
replacing certificate
```

Day 1 mới học foundation, nhưng mindset này áp dụng từ ngày đầu tiên.

---

## 9.2 Authorization khác authentication

Đây là distinction cực kỳ quan trọng.

Bạn login được không có nghĩa là bạn được phép thực hiện mọi action.

```text
Authentication:
Who are you?

Authorization:
What are you allowed to do?
```

Ví dụ:

```text
SSH login success
```

chỉ chứng minh authentication thành công.

Nó không đồng nghĩa với:

```text
authorized to edit /etc/ssh/sshd_config
```

---

# 10. Một controlled change có những thành phần gì?

Một change tốt thường có logic:

```text
Reason
→ Scope
→ Risk
→ Approval
→ Pre-check
→ Implementation
→ Verification
→ Rollback
→ Post-check
→ Evidence
→ Closure
```

### Scope

Phải rõ:

```text
one host?
multiple hosts?
one user?
one directory?
all files recursively?
```

### Risk

Ví dụ permission change:

```text
Risk:
application may lose read access.
```

### Rollback

Ví dụ:

```text
Previous:
owner = appuser
group = appgroup
mode = 0640
```

Nếu change thất bại có thể restore baseline đó.

Không có pre-change evidence thì rollback thường khó đáng tin cậy hơn.

---

# 11. Least Privilege

Syllabus đặt `least-privilege principles` ngay Day 1. Infrastructure_Operations_14_Da…

## 11.1 Definition

`Least privilege` nghĩa là identity chỉ được cấp:

> minimum privilege cần thiết để thực hiện authorized function, trong minimum scope và duration phù hợp.

Không chỉ là "không cho root".

Privilege có nhiều dimension:

```text
What actions?
On which systems?
On which resources?
For how long?
From where?
Under which conditions?
```

---

## 11.2 Ví dụ least privilege

Không tốt:

```text
operator → full root access
```

Tốt hơn:

```text
operator
   ↓
sudo
   ↓
only approved administrative command
```

Ví dụ một engineer chỉ cần xem service state thì read-only command có thể đủ.

Không nên cấp toàn bộ root shell chỉ vì:

> "Đỡ phải chỉnh permission."

---

## 11.3 Privilege creep

Theo thời gian, account có thể tích tụ permission:

```text
Project A access
+
temporary incident access
+
old support role
+
new team role
```

Nếu không review:

```text
privilege creep
```

xảy ra.

Day 13 sẽ quay lại sâu hơn ở IAM review; Day 1 xây nền khái niệm.

---

## 11.4 Least privilege và operational efficiency

Least privilege không có nghĩa:

> cấm engineer làm việc.

Mục tiêu là:

```text
Enough permission to work
but
Not unnecessary permission
```

Thiết kế quá restrictive cũng gây:

- operational delay;
- workaround;
- shared credentials;
- unauthorized privilege escalation attempts.

Do đó permission phải dựa trên job function và runbook.

---

# 12. Standard Operating Procedure — SOP

## 12.1 SOP là gì?

`Standard Operating Procedure` là quy trình chuẩn mô tả cách thực hiện một recurring operational task.

Ví dụ:

```text
Daily Linux Health Check SOP
```

có thể mô tả:

```text
Purpose
Scope
Prerequisites
Roles
Procedure
Validation
Evidence
Escalation
```

---

## 12.2 Vì sao cần SOP?

Nếu mỗi engineer thao tác theo trí nhớ:

```text
Engineer A → checks A/B/C
Engineer B → checks B/D
Engineer C → immediately restarts service
```

thì:

```text
inconsistent operation
```

SOP giúp:

```text
repeatability
consistency
auditability
training
reduced error rate
```

---

# 13. Runbook

`Runbook` thường operational và action-oriented hơn.

Ví dụ:

```text
Runbook: SSH Login Failure
```

có thể có decision tree:

```text
Can host be resolved?
   ├─ No → DNS path
   └─ Yes
        ↓
TCP/22 reachable?
   ├─ No → network/firewall/service path
   └─ Yes
        ↓
Authentication prompt?
   ├─ No → SSH negotiation path
   └─ Yes
        ↓
Authentication succeeds?
```

Runbook nên có:

```text
approved commands
expected output
decision points
stop conditions
evidence
escalation boundary
```

---

# 14. SOP vs Runbook

Không có một universal definition tuyệt đối cho mọi organization, nhưng practical distinction thường là:

| SOP                           | Runbook                              |
| ----------------------------- | ------------------------------------ |
| Quy trình chuẩn ở mức process | Hướng dẫn execution cụ thể           |
| Có thể mô tả governance       | Thường rất action-oriented           |
| Nói ai làm, khi nào, vì sao   | Nói check gì, command gì, branch nào |
| Có thể bao quát lifecycle     | Có thể tập trung một task/incident   |

Trong một số organization, hai từ này được dùng gần như thay thế. Luôn theo internal terminology.

---

# 15. Incident Escalation

## 15.1 Incident

`Incident` là interruption hoặc degradation không mong muốn đối với service.

Ví dụ:

```text
User cannot SSH
Application unavailable
Permission denied
Disk full
Service down
```

Day 1 chỉ cần nắm escalation foundation; formal ITSM sẽ học sâu ở Day 11.

---

## 15.2 Symptom khác root cause

Ví dụ:

```text
Symptom:
Permission denied.

Possible root causes:
wrong username
wrong file permission
wrong directory permission
wrong group membership
wrong SSH key
server-side SSH restriction
sudo policy
```

Không được biến symptom thành root cause.

Sai:

> Root cause là "Permission denied".

Đúng hơn:

> `Permission denied` là symptom.

---

## 15.3 Evidence-driven escalation

Trước escalate nên biết:

```text
What happened?
When?
Where?
Who is affected?
What changed?
What has been tested?
What was not tested?
What evidence exists?
What boundary prevents further action?
```

---

# 16. Access Management

## 16.1 Identity, Authentication, Authorization

Ba concept phải phân biệt:

```text
Identity
   = account/entity

Authentication
   = proving identity

Authorization
   = deciding permitted actions
```

Có thể thêm fourth concept:

```text
Accounting/Auditing
   = recording activity
```

Gần với mô hình `AAA`:

```text
Authentication
Authorization
Accounting
```

---

## 16.2 Access lifecycle

Access không nên chỉ có bước "create account".

Một lifecycle đầy đủ:

```text
Request
↓
Identity validation
↓
Business justification
↓
Approval
↓
Provisioning
↓
Verification
↓
Use
↓
Review
↓
Modification
↓
Expiry/Revocation
↓
Audit
```

Day 1 chủ yếu xây mindset. Day 4, Day 11 và Day 13 sẽ mở rộng IAM/privileged access.

---

## 16.3 Shared account

Shared account gây khó khăn cho accountability.

Ví dụ:

```text
linuxadmin
```

được 10 người cùng biết password.

Nếu log ghi:

```text
linuxadmin changed permission
```

thì khó xác định individual actor.

Enterprise thường ưu tiên:

```text
individual identity
→ controlled privilege elevation
```

thay vì shared generic privileged credentials, trừ trường hợp được quản trị qua PAM hoặc mechanism có audit phù hợp.

---

# 17. Configuration Baseline

## 17.1 Baseline là gì?

`Configuration baseline` là trạng thái configuration/reference được biết và được chấp nhận tại một thời điểm.

Ví dụ Day 1:

```text
Hostname
OS distribution
Kernel version
Users
Groups
Ownership
Permissions
sudo policy
SSH-related state
```

Baseline trả lời:

> "Normal/expected state của hệ thống là gì?"

---

## 17.2 Baseline và change

Logic:

```text
Known baseline
     ↓
Approved change
     ↓
New verified state
     ↓
Updated baseline
```

Nếu không có baseline:

```text
Current state ≠ expected state
```

sẽ rất khó đánh giá.

---

## 17.3 Configuration drift

`Configuration drift` là việc actual configuration lệch dần khỏi expected baseline.

Ví dụ:

```text
Expected:
0640

Actual:
0666
```

hoặc:

```text
Expected sudo:
service status only

Actual sudo:
ALL
```

Drift có thể do:

- manual changes;
- automation lỗi;
- emergency fixes;
- package upgrade;
- incomplete rollback;
- configuration management mismatch.

---

# 18. Asset Inventory

## 18.1 Asset inventory là gì?

Asset inventory trả lời:

```text
What systems do we have?
Where are they?
Who owns them?
What are they used for?
What version/configuration do they have?
```

Một asset record tốt thường có:

```text
Asset ID
Hostname
Environment
Platform
OS
Version
IP/management identity
Owner
Support group
Business service
Criticality
Lifecycle state
```

Không phải tất cả fields trên đều bắt buộc trong Day 1, nhưng đây là operational context.

---

## 18.2 Tại sao inventory rất quan trọng?

Không thể:

```text
patch
monitor
secure
backup
review access
manage vulnerabilities
```

một asset mà organization không biết nó tồn tại.

Một câu rất quan trọng:

> Unknown asset thường trở thành unmanaged asset.

---

# 19. Audit Evidence

Syllabus yêu cầu `audit evidence` ngay Day 1 và còn bổ sung evidence handling chi tiết. Infrastructure_Operations_14_Da…

## 19.1 Evidence khác note như thế nào?

Note:

```text
I tested the account and it worked.
```

Evidence tốt hơn:

```text
Host
Account
Timestamp
Command/test
Relevant output
Ticket
Collector
```

Mục tiêu là để reviewer có khả năng:

```text
trace
reproduce where possible
validate
understand limitations
```

---

# 20. Lab Readiness

Syllabus đặt `lab readiness` trong Concept/Lecture Day 1. Infrastructure_Operations_14_Da…

Readiness không chỉ là VM bật lên.

Cần nghĩ về:

```text
Environment available?
Correct OS?
Authorized account?
Network access?
SSH client?
Evidence storage?
Ticket/reference?
Time synchronization?
Enough disk space?
Rollback mechanism?
No production dependency?
```

Chương trình yêu cầu instructor chuẩn bị trước lab environment, authorized product access và sanitized data; Day 1 cũng là điểm review prerequisite của learner. Infrastructure_Operations_14_Da…

---

# PHẦN II — LINUX FOUNDATION

# 21. Linux là gì?

Linux về mặt kỹ thuật ban đầu chỉ `Linux kernel`, nhưng trong thực tế khi nói "Linux server", người ta thường nói tới một operating system distribution kết hợp:

```text
Linux kernel
+
system libraries
+
shell
+
utilities
+
package ecosystem
+
services
+
distribution configuration
```

Do đó:

```text
Linux ≠ một sản phẩm duy nhất
```

---

# 22. Linux Architecture

Một simplified architecture:

```text
+-----------------------------------+
| Applications / Utilities         |
+-----------------------------------+
| Shell / System Services          |
+-----------------------------------+
| System Libraries                 |
+-----------------------------------+
| System Call Interface            |
+-----------------------------------+
| Linux Kernel                     |
| CPU | Memory | FS | Network | IO |
+-----------------------------------+
| Hardware                         |
+-----------------------------------+
```

Có thể chia thành:

```text
Kernel space
User space
```

---

# 23. Kernel

## 23.1 Kernel là gì?

`Kernel` là core của operating system.

Nó thực hiện các chức năng quan trọng như:

```text
process scheduling
memory management
filesystem interface
device management
networking
access to hardware resources
```

Application thông thường không trực tiếp tự ý truy cập hardware.

Nó đi qua controlled interfaces do kernel cung cấp.

---

## 23.2 Kernel space vs user space

`Kernel space` có privilege rất cao.

`User space` chứa:

- shells;
- commands;
- services;
- applications.

Việc phân tách giúp security và stability.

Một user-space program crash không nhất thiết làm kernel crash.

---

## 23.3 Kiểm tra kernel

Ví dụ:

```bash
uname -r
```

cho kernel release.

```bash
uname -a
```

cung cấp nhiều system/kernel information hơn.

Điểm operational:

Đừng nhầm:

```text
kernel version
```

với:

```text
Linux distribution release
```

Đó là hai information khác nhau.

---

# 24. Shell

## 24.1 Shell là gì?

`Shell` là interface giúp user gửi command tới hệ thống.

Ví dụ:

```text
bash
sh
zsh
```

Trong enterprise Linux, `bash` rất phổ biến, nhưng không nên mặc định mọi environment đều dùng cùng shell.

---

## 24.2 Command flow

Ví dụ user nhập:

```bash
ls -l /etc
```

Simplified flow:

```text
User
↓
Shell parses command
↓
Runs builtin or executable
↓
Program requests OS resources
↓
Kernel handles request
↓
Output returned
↓
Terminal displays result
```

---

# 25. Shell Built-in và External Command

Một số command là shell built-in.

Ví dụ:

```bash
cd
```

phải thay đổi working directory của chính shell hiện tại, nên thường là shell built-in.

Nhiều command khác là executable files.

Ví dụ có thể kiểm tra:

```bash
type cd
type ls
command -v ls
```

Đây là kiến thức bổ sung hữu ích khi troubleshooting.

---

# 26. Linux Distribution

Một `distribution` đóng gói Linux kernel với:

```text
user-space tools
package management
default configuration
security integration
support model
release lifecycle
documentation
```

Ví dụ:

```text
Red Hat Enterprise Linux
Fedora
Ubuntu
Debian
SUSE Linux Enterprise
Rocky Linux
AlmaLinux
```

---

# 27. Red Hat Enterprise Linux — RHEL

RHEL là enterprise Linux distribution của Red Hat.

Điểm Day 1 cần hiểu không phải học subscription/package management sâu — phần đó thuộc Day 2 — mà là nhận thức:

```text
RHEL is a Linux distribution
```

và trong enterprise environment cần quan tâm:

```text
release
support lifecycle
vendor support
package source
security update model
configuration standards
```

Các chi tiết về `dnf/yum/rpm`, repository và subscription awareness thuộc Day 2 theo syllabus.

---

# 28. Không nên giả định mọi Linux distribution giống nhau

Dù nhiều Linux command cơ bản giống nhau, differences có thể xuất hiện ở:

```text
package management
service naming
default filesystem layout details
network configuration
security defaults
repositories
support tools
release cadence
```

Ví dụ:

```text
RHEL family → dnf/rpm ecosystem
Debian family → apt/dpkg ecosystem
```

Day 1 chỉ cần hiểu distinction.

---

# PHẦN III — LINUX FILESYSTEM

# 29. Filesystem Hierarchy

Linux sử dụng một single hierarchical namespace bắt đầu từ:

```text
/
```

gọi là root directory.

Ví dụ:

```text
/
├── etc
├── home
├── root
├── var
├── tmp
├── usr
├── boot
├── dev
├── proc
└── sys
```

---

# 30. `/` — root directory

`/` là top-level của filesystem hierarchy.

Không được nhầm:

```text
/
```

với:

```text
/root
```

`/` = root of filesystem tree.

`/root` = home directory thường dùng cho `root` user.

---

# 31. `/etc`

`/etc` chứa rất nhiều system-wide configuration.

Ví dụ:

```text
/etc/passwd
/etc/group
/etc/ssh/
```

Operational principle:

> File nằm trong `/etc` thường có ảnh hưởng lớn hơn file cá nhân trong home directory.

Không có nghĩa mọi thứ trong `/etc` đều critical như nhau, nhưng thay đổi ở đây cần đặc biệt cẩn thận.

---

# 32. `/home`

Normal users thường có home directory dạng:

```text
/home/alice
/home/bob
```

Home có thể chứa:

```text
personal files
shell configuration
SSH user configuration
```

Ví dụ:

```text
~/.ssh/
```

---

# 33. `/root`

Home directory của `root` thường là:

```text
/root
```

Không giống:

```text
/
```

---

# 34. `/var`

`/var` chứa variable data.

Có thể gồm:

```text
logs
cache
spool
application variable data
```

Ví dụ log management chi tiết sẽ học Day 2.

---

# 35. `/tmp`

Temporary storage.

Không nên giả định:

> dữ liệu trong `/tmp` được bảo tồn lâu dài.

Security concern:

Temporary files phải tránh:

- predictable filenames;
- unsafe permissions;
- sensitive plaintext data.

---

# 36. `/usr`

Chứa phần lớn user-space programs, libraries và read-only/shareable data theo filesystem organization hiện đại.

Trên nhiều Linux hiện đại, một số path lịch sử như `/bin`, `/sbin` có thể liên quan hoặc được merged vào `/usr`. Cấu trúc cụ thể phải xác nhận trên distribution/version thực tế.

---

# 37. `/boot`

Chứa boot-related files.

Day 1 không thực hành thay đổi khu vực này.

Operational boundary:

> Không sửa boot files chỉ để thử nghiệm.

---

# 38. `/dev`

Device representations.

Ví dụ block device có thể được biểu diễn bằng filesystem path.

Storage sâu sẽ học Day 2.

---

# 39. `/proc` và `/sys`

Đây không phải ordinary persistent directories theo nghĩa đơn giản.

Chúng expose kernel/system information dưới dạng filesystem-like interfaces.

Ví dụ:

```bash
cat /proc/cpuinfo
```

Nhưng Day 1 chỉ cần awareness.

---

# PHẦN IV — CLI NAVIGATION

# 40. Current Working Directory

Shell luôn có current working directory.

Kiểm tra:

```bash
pwd
```

Ví dụ:

```text
/home/student
```

Điều này cực quan trọng trước command sử dụng relative path.

---

# 41. `ls`

Dùng để list directory contents.

```bash
ls
```

Một số options phổ biến:

```bash
ls -l
ls -a
ls -la
```

General meaning:

```text
-l → long listing
-a → include hidden names
```

Không nên học command chỉ bằng thuộc lòng option; phải biết đọc `man`/`--help`.

---

# 42. `cd`

Dùng thay đổi current directory.

```bash
cd /etc
```

Home directory:

```bash
cd ~
```

Parent:

```bash
cd ..
```

Current:

```bash
cd .
```

---

# 43. Absolute Path

Absolute path bắt đầu từ `/`.

Ví dụ:

```text
/etc/ssh/sshd_config
```

Nó không phụ thuộc current working directory.

---

# 44. Relative Path

Relative path được resolve từ current working directory.

Nếu đang ở:

```text
/home/student
```

và chạy:

```bash
cd documents
```

thì shell sẽ resolve:

```text
/home/student/documents
```

---

# 45. `.` và `..`

```text
.  = current directory
.. = parent directory
```

Ví dụ:

```bash
./script.sh
```

nghĩa là script trong current directory.

```bash
../file.txt
```

nghĩa là file trong parent directory.

---

# 46. `~`

`~` thường expand thành home directory của current user.

Ví dụ:

```bash
cd ~
```

Không nên nhầm với `/`.

---

# 47. Path safety

Một common mistake:

```bash
rm -r logs
```

trong sai current directory.

Do đó trước risky relative-path operations:

```bash
pwd
ls
```

có thể là một safety check rất giá trị.

---

# PHẦN V — FILE VÀ DIRECTORY OPERATIONS

# 48. Tạo file — `touch`

```bash
touch file.txt
```

Nếu file chưa tồn tại, thường tạo empty file.

Nếu đã tồn tại, `touch` có thể cập nhật timestamps thay vì "tạo lại".

Đây là nuance cần biết.

---

# 49. Tạo directory — `mkdir`

```bash
mkdir reports
```

Nested directories có thể cần:

```bash
mkdir -p day1/evidence/raw
```

`-p` có thể tạo parent directories cần thiết.

---

# 50. Copy — `cp`

Basic:

```bash
cp source.txt destination.txt
```

Copy directory thường cần recursive behavior, ví dụ:

```bash
cp -r source_dir destination_dir
```

Operational concerns:

```text
Does destination exist?
Will something be overwritten?
Are permissions/metadata important?
Is recursive copying intended?
```

---

# 51. Move/Rename — `mv`

```bash
mv old.txt new.txt
```

có thể rename.

Hoặc:

```bash
mv file.txt /another/path/
```

có thể move.

Risk:

Nếu destination tồn tại, behavior có thể dẫn tới overwrite tùy command/options/context.

Không chạy theo thói quen mà không kiểm tra destination.

---

# 52. Delete — `rm`

```bash
rm file.txt
```

Đây là destructive operation.

Recursive deletion:

```bash
rm -r directory
```

có blast radius lớn hơn.

Một principle Day 1:

> Với destructive command, xác minh target quan trọng hơn tốc độ.

Không nên hình thành thói quen dùng `-f` cho mọi command.

---

# 53. Directory deletion

Empty directory có thể xóa bằng:

```bash
rmdir directory
```

Nếu directory không empty, `rmdir` sẽ không xử lý như recursive deletion.

Đây có thể là một safety property hữu ích.

---

# 54. Wildcards và blast radius

Ví dụ:

```bash
rm *.log
```

Shell có thể expand wildcard thành nhiều files.

Trước khi sử dụng destructive wildcard, có thể inspect:

```bash
printf '%s\n' *.log
```

hoặc list candidates trước.

Concept quan trọng:

> Shell expansion xảy ra trước khi command nhận arguments.

---

# PHẦN VI — COMMAND SYNTAX

# 55. General command anatomy

Một command thường có dạng:

```text
command [options] [arguments]
```

Ví dụ:

```bash
ls -l /etc
```

Phân tích:

```text
ls      → command
-l      → option
/etc    → argument
```

---

# 56. Short và long options

Một số command hỗ trợ:

```text
-l
-a
```

hoặc long form:

```text
--help
```

Không có universal guarantee rằng mọi command dùng cùng option syntax.

Phải đọc help của chính command.

---

# 57. Quoting

Đây là kiến thức bổ sung cần thiết cho safe CLI.

Spaces và shell expansion có thể làm arguments thay đổi.

Ví dụ filename:

```text
Daily Report.txt
```

nên dùng:

```bash
cat "Daily Report.txt"
```

Không quote:

```bash
cat Daily Report.txt
```

shell sẽ hiểu thành hai arguments.

---

# 58. Single quote vs double quote

Trong shell như Bash:

```bash
echo "$HOME"
```

variable có thể expand.

Trong:

```bash
echo '$HOME'
```

thường in literal `$HOME`.

Đây là foundation quan trọng cho scripting sau này.

---

# PHẦN VII — STANDARD STREAMS, PIPES VÀ REDIRECTION

# 59. Standard Streams

Một process CLI thường có ba standard streams:

```text
0 = stdin
1 = stdout
2 = stderr
```

### stdin

Input.

### stdout

Normal output.

### stderr

Error/diagnostic output.

---

# 60. Pipe `|`

Pipe nối:

```text
stdout of command A
        ↓
stdin of command B
```

Ví dụ:

```bash
cat file.txt | grep ERROR
```

Nhưng nếu command hỗ trợ filename trực tiếp:

```bash
grep ERROR file.txt
```

thường đơn giản hơn.

---

# 61. Pipeline thinking

Ví dụ:

```bash
grep ERROR app.log | wc -l
```

Flow:

```text
app.log
↓
grep keeps matching lines
↓
wc counts lines
```

Pipeline cho phép compose nhiều tools nhỏ.

Đây là một triết lý mạnh của Unix/Linux CLI.

---

# 62. Output redirection `>`

```bash
command > output.txt
```

Redirect stdout sang file.

Nếu file đã tồn tại, shell thường truncate/overwrite file trước khi command chạy.

Vì vậy:

```bash
>
```

là một operator có risk.

---

# 63. Append `>>`

```bash
command >> output.txt
```

append stdout.

Nhưng append cũng có thể tạo duplicate evidence hoặc unbounded log growth nếu dùng sai.

---

# 64. Error redirection

```bash
command 2> error.txt
```

redirect stderr.

Combined example:

```bash
command > output.txt 2> error.txt
```

---

# 65. Merge stderr vào stdout

Ví dụ:

```bash
command > all.txt 2>&1
```

Conceptually:

```text
stdout → all.txt
stderr → same destination as stdout
```

Ordering matters.

```bash
command > all.txt 2>&1
```

không nhất thiết tương đương:

```bash
command 2>&1 > all.txt
```

vì redirections được processed theo thứ tự.

Day 1 chỉ cần biết để tránh mất diagnostic output.

---

# 66. Input redirection

```bash
command < input.txt
```

redirect file làm stdin.

---

# PHẦN VIII — COMMAND HISTORY

# 67. `history`

Trong Bash:

```bash
history
```

cho phép xem shell history hiện có.

History hữu ích để:

```text
recall commands
review recent activity
support troubleshooting
```

---

# 68. History không phải complete audit log

Điều này rất quan trọng.

Không được nói:

> "Có trong history nên đây là complete audit evidence."

History có limitations:

```text
shell-dependent
user-dependent
may not capture every activity
may be modified
may not be immediately written
may omit actions performed through other tools
may contain secrets if user enters them
```

Do đó:

```text
command history
≠
authoritative audit trail
```

---

# 69. Không nhập secret vào command line khi không cần thiết

Một số tools có thể để credential xuất hiện trong:

```text
shell history
process arguments
transcript
logs
```

Day 1 phải bắt đầu thói quen:

```text
No real password/private key/token in submitted evidence.
```

---

# PHẦN IX — HELP UTILITIES

# 70. `man`

```bash
man chmod
```

Manual thường cung cấp:

```text
name
synopsis
description
options
examples/references
```

Engineer giỏi không phải người nhớ mọi option.

Engineer giỏi biết:

> cách xác minh syntax từ trusted local documentation.

---

# 71. `--help`

```bash
chmod --help
```

thường cho quick reference.

Không phải mọi command đều implement chính xác giống nhau, nhưng đây là pattern rất phổ biến.

---

# 72. `info`

Một số GNU utilities có `info` documentation.

```bash
info command
```

Mức độ sử dụng tùy environment.

---

# 73. Help trước risky command

Một habit tốt:

```bash
man command
```

hoặc:

```bash
command --help
```

khi bạn không chắc một option.

Không nên "đoán flag".

---

# PHẦN X — SAFE COMMAND EXECUTION

# 74. Mental checklist trước command

Trước command quan trọng:

```text
Identity?
Host?
Environment?
Working directory?
Target?
Permission?
Scope?
Command effect?
Expected output?
Rollback?
Evidence?
```

Ví dụ:

```bash
chmod -R ...
```

`-R` có thể mở rộng blast radius rất lớn.

Phải hỏi:

> Recursive có thực sự cần thiết không?

---

# 75. Idempotent vs destructive thinking

Một số operations dễ repeat hơn.

Ví dụ check:

```bash
ls -l
```

không thay đổi state.

Trong khi:

```bash
rm
```

là destructive.

Không cần Day 1 học formal automation idempotency, nhưng cần bắt đầu phân loại commands:

```text
read-only
state-changing
destructive
privileged
```

---

# 76. Exit status

Exit code sẽ được học sâu hơn trong scripting Day 2/12, nhưng ngay Day 1 cần hiểu:

```text
command completed
```

không đồng nghĩa:

```text
business outcome verified
```

Một command có exit status success nhưng configuration vẫn không đáp ứng requirement.

---

# PHẦN XI — USERS VÀ GROUPS

# 77. User là gì?

Linux dùng user identity để kiểm soát ownership và access.

Một user thường có:

```text
username
UID
primary GID
home directory
login shell
```

Xem current identity:

```bash
whoami
```

Xem chi tiết:

```bash
id
```

---

# 78. Root user

`root` có UID:

```text
0
```

và có administrative power rất lớn.

Principle:

> Không sử dụng root chỉ vì tiện.

Nên ưu tiên:

```text
normal account
+
controlled sudo
```

khi policy cho phép.

---

# 79. System/service accounts

Một server cũng có nhiều non-human accounts phục vụ service.

Không nên tùy tiện:

```text
change shell
change home
delete account
reset password
```

chỉ vì account "trông lạ".

Phải xác định ownership và purpose.

---

# 80. `/etc/passwd`

Chứa account metadata cơ bản.

Ví dụ format conceptual:

```text
name:x:UID:GID:comment:home:shell
```

Password hash hiện đại không nên nằm trực tiếp tại đây; shadow information thường được bảo vệ riêng.

---

# 81. `/etc/shadow`

Chứa sensitive password-aging/hash-related information.

Không phải normal user nào cũng được đọc.

Security principle:

> Không đưa `/etc/shadow` vào lab evidence hoặc chat chỉ để chứng minh account tồn tại.

Dùng safer lookup.

---

# 82. `/etc/group`

Chứa group definitions/membership information theo traditional local account model.

Có thể inspect thông qua:

```bash
getent group groupname
```

Thường tốt hơn việc tự parse file khi system có thể dùng nhiều identity sources.

---

# 83. `getent`

`getent` có thể query system databases thông qua configured name-service mechanisms.

Ví dụ:

```bash
getent passwd alice
getent group linuxops
```

Đây là operationally useful vì environment enterprise có thể không chỉ dựa vào local `/etc/passwd`.

---

# 84. Primary và supplementary groups

Một user có:

```text
primary group
+
zero or more supplementary groups
```

Kiểm tra:

```bash
id alice
```

Group membership là nền tảng cho shared filesystem access.

---

# PHẦN XII — OWNERSHIP

# 85. File ownership

Một filesystem object thường có:

```text
user owner
group owner
```

Ví dụ:

```bash
ls -l report.txt
```

Có thể thấy:

```text
-rw-r----- 1 alice linuxops ... report.txt
```

Trong đó:

```text
alice     → user owner
linuxops  → group owner
```

---

# 86. Ownership không đồng nghĩa permission

File owner không tự động nói chính xác access được phép.

Phải kết hợp:

```text
ownership
+
mode bits
+
identity/group membership
+
directory permissions
+
possible additional security controls
```

Day 1 tập trung classic Unix permission model. SELinux sẽ học Day 2.

---

# PHẦN XIII — FILE PERMISSIONS

# 87. Permission classes

Classic mode bits chia thành:

```text
user
group
others
```

viết tắt:

```text
u
g
o
```

---

# 88. Permission bits

```text
r = read
w = write
x = execute
```

Numeric:

```text
r = 4
w = 2
x = 1
```

---

# 89. File permission semantics

Với regular file:

### `r`

Đọc nội dung.

### `w`

Modify/truncate nội dung nếu các điều kiện filesystem khác cho phép.

### `x`

Execute file như executable/script khi relevant.

---

# 90. Directory permission semantics

Đây là chỗ rất nhiều người mới học sai.

Với directory:

### `r`

Cho phép đọc/list directory entries.

### `w`

Cho phép modify directory entries — tạo/xóa/rename entries khi kết hợp với các permissions cần thiết.

### `x`

Cho phép traverse/search directory.

Do đó:

> `rwx` trên directory không có cùng ý nghĩa với `rwx` trên regular file.

---

# 91. Ví dụ permission string

```text
-rwxr-x---
```

Tách:

```text
-    regular file

rwx  user
r-x  group
---  others
```

Numeric:

```text
7 5 0
```

vì:

```text
rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
--- = 0
```

→ `0750`.

---

# 92. Ví dụ khác

```text
-rw-r-----
```

Numeric:

```text
6 4 0
```

vì:

```text
rw- = 6
r-- = 4
--- = 0
```

---

# 93. `chmod`

Thay đổi permission mode.

Symbolic:

```bash
chmod u+x script.sh
```

Nghĩa:

```text
add execute to user/owner class
```

Numeric:

```bash
chmod 750 script.sh
```

Nghĩa:

```text
u = rwx
g = r-x
o = ---
```

---

# 94. Không reflexively dùng `chmod 777`

`777` nghĩa:

```text
user   rwx
group  rwx
others rwx
```

Nó thường rộng hơn nhiều so với requirement.

Nếu gặp:

```text
Permission denied
```

và ngay lập tức dùng:

```bash
chmod 777 ...
```

thì bạn đã bỏ qua root-cause analysis và least privilege.

---

# 95. Permission troubleshooting

Nếu user không đọc được:

```text
/srv/app/config.txt
```

workflow an toàn:

```text
Identify user
↓
id user
↓
Inspect file
↓
ls -l /srv/app/config.txt
↓
Inspect path directories
↓
namei -l /srv/app/config.txt   [nếu available]
↓
Check owner/group
↓
Check group membership
↓
Determine expected policy
↓
Change only if authorized
```

Đôi khi file permission đúng nhưng parent directory thiếu `x`.

---

# 96. `chown`

Đổi user owner.

```bash
chown alice report.txt
```

Đổi owner và group:

```bash
chown alice:linuxops report.txt
```

Chỉ nên thay đổi khi biết application/access dependency.

---

# 97. `chgrp`

Đổi group owner.

```bash
chgrp linuxops report.txt
```

Use case:

Shared team files.

---

# 98. Recursive ownership change

Ví dụ:

```bash
chown -R ...
```

có blast radius rất lớn.

Trước khi recursive change cần:

```text
verify target
inventory contents
understand symlink behavior/options
capture baseline
confirm application expectations
```

Không sử dụng recursive mode theo thói quen.

---

# PHẦN XIV — UMASK

# 99. `umask` là gì?

`umask` là mask ảnh hưởng tới default mode khi process tạo filesystem objects.

Xem current value:

```bash
umask
```

Ví dụ:

```text
0022
```

---

# 100. Không hiểu `umask` là "permission"

`umask` không phải permission mode trực tiếp.

Conceptually:

```text
requested/base mode
AND
NOT umask
=
resulting mode
```

Thực tế application có thể request mode riêng, nên không nên dùng subtraction như một universal rule.

---

# 101. Typical base modes

Thường để giải thích basic concept:

```text
regular file base ≈ 0666
directory base    ≈ 0777
```

Tại sao file không mặc định `0777`?

Vì new regular files thường không tự động được cấp execute bit chỉ vì `umask`.

---

# 102. Ví dụ `umask 0022`

Regular file:

```text
0666
masked by 0022
→ 0644
```

Directory:

```text
0777
masked by 0022
→ 0755
```

Nghĩa là thường:

```text
file:
owner rw
group r
others r

directory:
owner rwx
group r-x
others r-x
```

---

# 103. Ví dụ `umask 0027`

Typical result:

```text
file → 0640
directory → 0750
```

Phù hợp hơn trong một số controlled multi-user environment.

Nhưng organization phải có baseline riêng.

---

# PHẦN XV — SUDO VÀ PRIVILEGE ELEVATION

# 104. `sudo` là gì?

`sudo` cho phép authorized user thực thi command với elevated identity theo configured policy.

Không nên hiểu:

```text
sudo = root password tool
```

`sudo` là một authorization mechanism.

---

# 105. Vì sao `sudo` tốt hơn login root trực tiếp trong nhiều use case?

Nó hỗ trợ:

```text
named individual accounts
policy-based privilege
restricted commands
better accountability
temporary elevation workflow
```

Tuy nhiên logging/audit detail cụ thể còn phụ thuộc configuration.

---

# 106. `sudo -l`

Một verification rất hữu ích:

```bash
sudo -l
```

cho biết sudo privileges mà policy cho user hiện tại.

Nên dùng trước khi thử random privileged commands.

---

# 107. Least-privilege sudo

Không tốt:

```text
operator ALL=(ALL) ALL
```

nếu requirement chỉ là một operational action cụ thể.

Tốt hơn về principle:

```text
Allow only required command(s)
for required identity
on required host(s)
```

Actual sudoers syntax phải được thiết kế cẩn thận và test trong lab.

---

# 108. Sudoers safety

Không nên chỉnh sudoers một cách tùy tiện bằng generic editor nếu organization yêu cầu validation.

Tool truyền thống:

```bash
visudo
```

giúp syntax validation và safe editing behavior phù hợp hơn.

Có thể có files trong:

```text
/etc/sudoers.d/
```

tùy configuration.

---

# 109. Allowed và denied test đều quan trọng

Least privilege phải chứng minh hai chiều:

```text
Required operation → allowed
Unauthorized operation → denied
```

Nếu chỉ test:

> allowed command succeeds

thì chưa chứng minh policy không quá rộng.

Đây cũng chính là reason syllabus yêu cầu Day 1 verify **authorized and denied access**. Infrastructure_Operations_14_Da…

---

# PHẦN XVI — SSH FOUNDATION

# 110. SSH là gì?

`SSH` — Secure Shell — là protocol phổ biến để secure remote administration.

Simplified flow:

```text
SSH client
     ↓
Network
     ↓
SSH server
     ↓
Server identity validation
     ↓
User authentication
     ↓
Authorized session
```

---

# 111. SSH có nhiều security dimensions

SSH security không chỉ là encrypt network connection.

Cần quan tâm:

```text
Which server?
Which user?
Which authentication method?
Which key?
Which permissions?
Which privilege after login?
```

---

# 112. SSH server identity

Khi client lần đầu connect tới server, client cần xác minh server host identity.

`known_hosts` có vai trò lưu host-key information.

Nếu host key bất ngờ thay đổi, không nên reflexively xóa warning để "cho connect được".

Possible causes:

```text
server rebuilt
IP reused
host key legitimately changed
wrong target
DNS issue
man-in-the-middle risk
```

Phải verify.

---

# 113. User authentication

Common SSH authentication có thể gồm:

```text
password
public key
```

Enterprise configuration có thể còn các mechanisms khác tùy environment.

Day 1 tập trung SSH key authentication.

---

# PHẦN XVII — SSH KEY PAIRS

# 114. Public key cryptography concept

Một SSH key pair có:

```text
private key
public key
```

Private key:

```text
must remain secret
```

Public key:

```text
can be distributed to authorized server
```

---

# 115. Authentication simplified

Public-key SSH simplified:

```text
Client has private key
        ↓
Server has corresponding public key authorization
        ↓
Cryptographic challenge/proof
        ↓
Server verifies client possesses private key
        ↓
Login allowed if policy permits
```

Private key **không cần được copy lên server**.

Đó là điểm rất quan trọng.

---

# 116. Common key locations

User SSH directory:

```text
~/.ssh/
```

Private key ví dụ:

```text
~/.ssh/id_ed25519
```

Public key:

```text
~/.ssh/id_ed25519.pub
```

Server-side authorized keys thường:

```text
~/.ssh/authorized_keys
```

Known server host identities phía client:

```text
~/.ssh/known_hosts
```

Exact behavior có thể bị thay đổi bởi SSH configuration; đây là common defaults/conventions chứ không nên xem như universal truth trong mọi environment.

---

# 117. Generate key

Typical modern example:

```bash
ssh-keygen -t ed25519
```

Trong enterprise, allowed algorithm có thể phụ thuộc:

```text
security policy
FIPS requirements
server/client versions
organization standards
```

Do đó lab phải theo approved environment.

Không nên tự khẳng định một algorithm là universally required.

---

# 118. Private key protection

Private key phải:

```text
not be emailed casually
not be committed to Git
not be included in screenshots
not be pasted into tickets
not be stored in shared evidence
not be uploaded into chat
```

Nếu private key bị lộ:

> coi nó là potentially compromised credential.

---

# 119. Passphrase

Private key có thể được bảo vệ bằng passphrase.

Passphrase giúp giảm risk nếu file private key bị đánh cắp.

Enterprise may additionally use:

```text
ssh-agent
hardware-backed key
certificate-based SSH
PAM/MFA
bastion/PAM gateway
```

Những thứ này vượt Day 1 core nhưng hữu ích để biết.

---

# 120. `authorized_keys`

User public key có thể được authorized trên server thông qua:

```text
~/.ssh/authorized_keys
```

Conceptually:

```text
one authorized public key
=
one credential allowed to attempt login as that account,
subject to server policy
```

Không nên nghĩ:

> có public key trong file là chắc chắn login được.

Other controls vẫn có thể deny.

---

# 121. SSH file permissions

OpenSSH thường yêu cầu secure ownership/permissions trên user SSH files/directories.

Nếu quá permissive, server có thể refuse public-key authentication.

Do exact checks phụ thuộc environment/version/configuration, troubleshooting nên inspect:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

thay vì blindly `chmod 777`.

---

# 122. SSH connection command

Basic:

```bash
ssh username@hostname
```

Explicit key:

```bash
ssh -i /path/to/private_key username@hostname
```

Không đưa private key content vào command.

---

# 123. Debugging SSH

Useful client verbosity:

```bash
ssh -v username@hostname
```

Có các higher verbosity levels tùy client, nhưng nên bắt đầu với mức vừa đủ.

Diagnostic output có thể chứa:

```text
hostname
username
key path
key fingerprint
network details
```

nên sanitize evidence nếu cần.

---

# PHẦN XVIII — SSH TROUBLESHOOTING MODEL

# 124. Không xem mọi SSH failure là một loại lỗi

SSH path:

```text
DNS/name resolution
↓
IP routing
↓
TCP connection
↓
SSH protocol negotiation
↓
Server identity
↓
User authentication
↓
Authorization/session
↓
Post-login privilege
```

Mỗi layer có failure khác nhau.

---

# 125. `Connection timed out`

Có thể gợi ý:

```text
routing problem
firewall drop
wrong IP
host unavailable
network path issue
```

Không trực tiếp chứng minh sshd bị lỗi.

---

# 126. `Connection refused`

Có thể gợi ý host reachable nhưng TCP destination không accept connection.

Possible:

```text
service not listening
wrong port
local firewall reject
service failed
```

Cần evidence khác.

---

# 127. `Permission denied`

Nếu thấy:

```text
Permission denied (publickey)
```

network/TCP/SSH handshake đã đi xa hơn "server unreachable".

Possible root causes:

```text
wrong username
wrong private key
public key not authorized
bad file permissions
server-side policy
account restrictions
```

---

# 128. Secure Administrative Access

Syllabus không chỉ yêu cầu "SSH login" mà yêu cầu `secure administrative access`. Infrastructure_Operations_14_Da…

Security model nên là:

```text
Named identity
↓
Secure authentication
↓
Correct target
↓
Minimum base privilege
↓
Controlled sudo
↓
Logged/auditable administrative action
```

Không nên:

```text
shared root password
↓
direct root login everywhere
↓
untracked configuration changes
```

---

# PHẦN XIX — EVIDENCE HANDLING & OPERATIONAL BASELINE

Đây là phần additive coverage của syllabus và có giá trị xuyên suốt Day 1–14.

Syllabus yêu cầu mỗi evidence item có metadata về source, environment, collector, timestamp, command/query, ticket, file characteristics, hash và classification; raw exports phải được giữ nguyên, derived copies phải trace về source. Infrastructure_Operations_14_Da…

Common evidence contract của syllabus còn mở rộng yêu cầu thành manifest có source system/product/version, asset/account identifier, stable ticket/reference key, exact filters/commands, export format/encoding, parent evidence ID cho derived files và limitations/errors. Infrastructure_Operations_14_Da…

---

# 129. Evidence ID

Mỗi evidence artifact nên có unique identifier.

Ví dụ:

```text
D01-EVID-001
```

Purpose:

```text
unambiguous reference
```

Thay vì:

> "Xem screenshot thứ hai."

Có thể nói:

> "See D01-EVID-004."

---

# 130. Source Owner / Source System

Evidence phải trả lời:

> Nó đến từ đâu?

Ví dụ:

```text
Source system: rhel-lab01
Source: uname -a
```

Hoặc:

```text
Source file: /etc/os-release
```

Source owner có thể chỉ operational/business ownership phù hợp với evidence contract của organization.

---

# 131. Hostname / Asset ID

Hostname một mình có thể thay đổi hoặc duplicate ở environment khác.

Asset ID giúp stable reference hơn nếu inventory có.

Ví dụ:

```text
Hostname: rhel-lab01
Asset ID: LAB-LNX-001
```

---

# 132. Environment

Evidence phải nói rõ:

```text
LAB
DEV
UAT
PROD
```

Nếu screenshot không có environment context, reviewer có thể không biết output thuộc hệ thống nào.

---

# 133. Collection Timestamp và Timezone

Timestamp không đủ nếu thiếu timezone.

Ví dụ tốt:

```text
2026-10-20 14:35:22 +07:00
```

hoặc standardized format phù hợp.

Lý do:

Hệ thống ở:

```text
Vietnam
Singapore
AWS UTC logs
US datacenter
```

có thể dùng timezone khác nhau.

Không normalize time sẽ khó correlate logs.

---

# 134. Query/Window

Nếu evidence là log extraction, phải biết:

```text
What query?
Which time window?
Which filters?
```

Ví dụ:

```text
2026-10-20T10:00:00Z
to
2026-10-20T10:15:00Z
```

Nếu không record time window:

> reviewer không biết bạn đã search đúng period hay chưa.

---

# 135. Collector

Ai đã collect?

Ví dụ:

```text
Collector: learner01
```

Không nhất thiết phải chứa personally sensitive data trong training package; dùng approved identifier.

---

# 136. Ticket Reference

Evidence phải liên kết operational context.

Ví dụ:

```text
INC0123456
CHG0123456
LAB-D01-001
```

Mục tiêu:

```text
Evidence
↔
Action
↔
Authorization
↔
Business/incident context
```

---

# 137. File Size

Useful integrity/handling metadata:

```text
bytes
KB
MB
```

File size thay đổi bất ngờ có thể gợi ý evidence bị thay đổi, nhưng **file size không phải cryptographic integrity mechanism**.

---

# 138. Record Count

Ví dụ log export:

```text
Record count: 1,245
```

Count giúp kiểm tra completeness.

Ví dụ:

Raw export:

```text
1,245
```

Processed:

```text
320 matching records
```

Hai con số khác nhau có thể hợp lệ nếu transformation được document.

---

# 139. SHA-256

Hash example:

```bash
sha256sum evidence.log
```

Output kiểu:

```text
<hash>  evidence.log
```

Hash là deterministic digest của bytes.

Nếu file thay đổi, hash gần như chắc chắn sẽ thay đổi.

---

# 140. Hash chứng minh điều gì?

Nếu bạn đã lưu trusted baseline:

```text
Original file
SHA-256 = ABC
```

sau đó file hiện tại:

```text
SHA-256 = XYZ
```

thì chứng minh:

> bytes hiện tại không giống bytes của baseline đã hash.

---

# 141. Hash KHÔNG chứng minh điều gì?

Syllabus nhấn mạnh rõ:

> Hash không tự chứng minh source authenticity hoặc completeness. Infrastructure_Operations_14_Da…

Ví dụ bạn export nhầm time window:

```text
09:00–09:10
```

thay vì:

```text
09:00–10:00
```

Hash vẫn hợp lệ.

Nó chỉ chứng minh file không đổi kể từ baseline, không chứng minh query đúng.

Tương tự, nếu source data vốn đã thiếu events, hashing không tạo lại events bị thiếu.

---

# 142. Raw Evidence

`Raw evidence` hoặc untouched source export là bản được giữ nguyên sau collection.

Principle:

```text
Collect
↓
Hash
↓
Preserve
```

Không:

```text
Collect
↓
Open original
↓
Delete lines
↓
Save
↓
Call it raw
```

---

# 143. Working Copy

Tạo copy:

```text
raw/evidence.log
        ↓
working/evidence.log
```

Analysis, filtering, formatting nên làm trên working copy.

Raw remains unchanged.

---

# 144. Redacted Deliverable

Nếu evidence chứa sensitive data:

```text
username
internal IP
email
identifier
secret-like data
```

có thể cần redaction theo policy.

Nhưng redacted file là **new artifact**.

Nó cần:

```text
its own hash
parent evidence ID
redaction note
```

Common evidence contract của syllabus yêu cầu redacted copies có own hashes và redaction note; screenshots chỉ bổ trợ exportable records chứ không tự động thay thế raw exports. Infrastructure_Operations_14_Da…

---

# 145. Parent Evidence ID

Ví dụ:

```text
D01-EVID-001
Raw export
```

Derived:

```text
D01-EVID-002
Parent: D01-EVID-001
Transformation:
filtered non-sensitive lines
```

Nhờ đó reviewer trace:

```text
deliverable
→ working copy
→ raw source
```

---

# 146. Evidence Classification

Có thể là:

```text
Internal
Restricted
Confidential
Security-sensitive
```

Exact classification phải theo organization.

Purpose:

> Không phải evidence nào cũng được phép gửi qua cùng một channel.

---

# 147. Evidence Gap

Một concept rất quan trọng trong syllabus.

Nếu không collect được vì:

```text
permission denied
log retention expired
source unavailable
unsupported field
collection failed
```

không nên giả vờ evidence đầy đủ.

Phải ghi:

```text
Evidence gap
Cause
Impact
Attempted collection
Escalation/reviewer decision
```

Syllabus yêu cầu limitations/errors phải được ghi lại thay vì biến unknown thành known. Infrastructure_Operations_14_Da…

---

# 148. Screenshots có vai trò gì?

Screenshot hữu ích để chứng minh:

```text
visible UI state
context
error message
configuration screen
```

Nhưng screenshot có limitations:

```text
may omit rows
hard to search
hard to parse
can crop context
often lacks complete metadata
```

Vì vậy syllabus quy định screenshot nên **supplement** exportable records, không mặc định thay thế chúng. Infrastructure_Operations_14_Da…

---

# PHẦN XX — KẾT NỐI CÁC KHÁI NIỆM DAY 1

Đây là phần quan trọng nhất để biến kiến thức rời rạc thành operational reasoning.

Giả sử có ticket:

```text
User alice needs read access to:
 /srv/reports/monthly.txt
```

Bạn không lập tức:

```bash
chmod 777 /srv/reports/monthly.txt
```

Workflow reasoning phải là:

```text
1. Confirm ticket and authorization
        ↓
2. Confirm environment/host
        ↓
3. Confirm current identity
        ↓
4. Inspect target
        ↓
5. Inspect alice identity/groups
        ↓
6. Inspect directory + file ownership/mode
        ↓
7. Determine least-privilege design
        ↓
8. Capture baseline
        ↓
9. Perform minimum approved change
        ↓
10. Verify alice = allowed
        ↓
11. Verify unauthorized user = denied
        ↓
12. Capture post-change evidence
        ↓
13. Hash/store evidence
        ↓
14. Update ticket
```

Đó chính là reason syllabus kết hợp:

```text
operations governance
+
Linux permissions
+
least privilege
+
evidence
```

trong cùng Day 1.

---

# 149. Common Mistakes của Day 1

Các lỗi tư duy cần tránh từ ngày đầu:

| Mistake                              | Vấn đề                                      |
| ------------------------------------ | ------------------------------------------- |
| Login được nên nghĩ có authorization | Authentication ≠ authorization              |
| Lỗi permission thì dùng `777`        | Phá least privilege, che root cause         |
| Luôn làm bằng `root`                 | Tăng blast radius và giảm accountability    |
| Không check `hostname`               | Có thể thao tác nhầm host                   |
| Không check `pwd`                    | Relative path có thể tác động sai directory |
| Dùng recursive option theo thói quen | Blast radius lớn                            |
| Sửa trước, inspect sau               | Mất baseline/root-cause evidence            |
| Chỉ test allowed case                | Không biết permission có quá rộng không     |
| Copy private SSH key lên server      | Sai mô hình key authentication              |
| Bỏ qua SSH host-key warning          | Có thể bỏ qua wrong host/security issue     |
| Dùng shell history như audit truth   | History không đầy đủ/authoritative          |
| Chỉnh raw evidence trực tiếp         | Phá evidence integrity                      |
| Hash xong nghĩ evidence complete     | Hash ≠ authenticity/completeness            |
| Screenshot mọi thứ, không export     | Thiếu machine-readable/raw evidence         |
| Escalate không có evidence           | Tăng thời gian resolution                   |
| Cố xử lý ngoài L1/L2 boundary        | Operational/security risk                   |

---

# 150. Verification Mindset của Day 1

Mọi topic Day 1 nên được verify theo pattern:

```text
Expected state
        ↓
Observed state
        ↓
Compare
        ↓
Evidence
```

Ví dụ permission:

```text
Expected:
alice can read
bob cannot read

Observed:
alice → success
bob → denied
```

Đây mạnh hơn:

```text
chmod command succeeded
```

---

# 151. Troubleshooting Mindset

Thứ tự ưu tiên:

```text
1. Confirm symptom
2. Confirm target
3. Confirm scope
4. Inspect current state
5. Compare baseline
6. Test least-risk hypothesis
7. Narrow layer
8. Change only if authorized
9. Verify
10. Escalate with evidence if boundary reached
```

Không:

```text
Guess
→ change
→ guess again
→ restart
→ chmod 777
→ reboot
```

---

# 152. Security mindset

Day 1 cần ghi nhớ mô hình:

```text
Identity
↓
Authentication
↓
Authorization
↓
Privilege
↓
Action
↓
Audit/Evidence
```

Nếu một link yếu:

```text
shared identity
weak authentication
excessive authorization
uncontrolled root privilege
no evidence
```

thì operation sẽ khó secure/audit.

---

# 153. Evidence mindset

Một action không nên chỉ có:

```text
"I did it."
```

Mà cần có:

```text
What?
Where?
Who?
When?
Why?
Under which approval?
Before state?
After state?
Expected result?
Actual result?
Evidence ID?
```

---

# 154. Escalation boundary tổng hợp cho Day 1

Một learner/L1/L2 trong Day 1 nên **dừng và escalate** nếu gặp các trường hợp như:

```text
production target unexpectedly involved
required approval missing
privilege higher than approved scope
root access required but not authorized
unknown account ownership
unknown service account
recursive change blast radius unclear
unexpected SSH host-key change
suspected credential/private-key compromise
sudo policy change beyond runbook
filesystem/permission issue may impact production application
evidence source incomplete or unavailable
suspected security incident
system behavior inconsistent with approved lab/runbook
```

Không được biến uncertainty thành assumption.

---

# 155. Ranh giới Day 1 và Day 2

Một điều cần giữ rất rõ: syllabus Day 1 chỉ đặt **Linux administration foundation**. Các chủ đề sau là Day 2 chứ chưa phải trọng tâm lý thuyết Day 1:

```text
dnf/yum/rpm
repository/subscription
process states
signals
resource usage
systemd lifecycle
boot targets
SELinux
firewalld
IP/routing/DNS
ping/traceroute/curl/ss/ip
disk/partition/filesystem/mount
/etc/fstab
journalctl/syslog
log rotation
advanced grep/awk/sed
regular expressions
Bash control flow
exit-code handling
cron
tar/rsync
backup/restore
operational failure injection
```

Việc giữ boundary này quan trọng để không học lẫn Day 2 vào Day 1. Syllabus đặt nhóm trên ở Day 2. Infrastructure_Operations_14_Da…

---

# 156. Concept Map hoàn chỉnh Day 1

```text
                    DAY 1
                      │
        ┌─────────────┴─────────────┐
        │                           │
 Infrastructure Ops             Linux Foundation
        │                           │
        │                   ┌───────┴────────┐
        │                   │                │
 Governance              Architecture     CLI
        │                   │                │
 ┌──────┼───────┐       Kernel            Paths
 │      │       │       Shell             Files
L1/L2  Safety  Change     Distros          Pipes
 │      │       │         RHEL             Redirect
 │      │       │                          History
 │      │       │                          Help
 │      │       │
 │   Prod/non-prod
 │
 ├── Least privilege
 ├── SOP / Runbook
 ├── Escalation
 ├── Access management
 ├── Configuration baseline
 ├── Asset inventory
 └── Audit evidence
                                      │
                                      ↓
                              Identity & Access
                                      │
                       ┌──────────────┼──────────────┐
                       │              │              │
                    Users/Groups   Permissions     SSH
                       │              │              │
                     UID/GID       ownership       keys
                                    chmod           auth
                                    chown           secure
                                    chgrp           admin
                                    umask           access
                                    sudo
                                      │
                                      ↓
                           Least-privilege access
                                      │
                                      ↓
                            Verification / Denial
                                      │
                                      ↓
                                Evidence
                                      │
                    ┌─────────────────┼────────────────┐
                    │                 │                │
                 Raw copy          Manifest        SHA-256
                    │                 │                │
                 Preserve          Trace            Integrity
                    │                                  │
                    └───────── Working/Redacted ──────┘
```

---

# 157. Mental Models cần thuộc sau Module Lý thuyết Day 1

Thay vì chỉ thuộc command, hãy nhớ các chuỗi reasoning sau.

### Operational change

```text
Authorize
→ Baseline
→ Pre-check
→ Change
→ Verify
→ Rollback if needed
→ Evidence
```

### Incident

```text
Symptom
→ Scope
→ Layer
→ Evidence
→ Hypothesis
→ Safe test
→ Resolve or Escalate
```

### Linux permission

```text
Identity
→ Group membership
→ Ownership
→ Directory permissions
→ File permissions
→ Effective access
```

### SSH

```text
Target
→ Network
→ SSH service
→ Server identity
→ User authentication
→ Authorization
→ sudo privilege
```

### Evidence

```text
Source
→ Collect
→ Metadata
→ Hash
→ Preserve raw
→ Work on copy
→ Record transformation
→ Deliver
```

### Least privilege

```text
Correct person
+
Correct resource
+
Correct operation
+
Correct duration
=
Required access only
```

---

# 158. Day 1 — Cheat Sheet Lý thuyết

| Area            | Nội dung cần nhớ                                          |
| --------------- | --------------------------------------------------------- |
| Support         | L1/L2 boundary, ownership, escalation                     |
| Safety          | Verify identity/host/environment/target trước change      |
| Change          | Approval → pre-check → implementation → verify → rollback |
| Security        | Authentication ≠ Authorization                            |
| Principle       | Least privilege                                           |
| Process         | SOP, runbook, ticket                                      |
| Baseline        | Expected known configuration                              |
| Inventory       | Biết asset nào tồn tại, ở đâu, owner là ai                |
| Linux           | Kernel + userspace + distribution                         |
| RHEL            | Enterprise Linux distribution                             |
| Root filesystem | `/`                                                       |
| Config          | `/etc`                                                    |
| User homes      | `/home`                                                   |
| Root home       | `/root`                                                   |
| Variable data   | `/var`                                                    |
| Navigation      | `pwd`, `cd`, `ls`                                         |
| File ops        | `touch`, `mkdir`, `cp`, `mv`, `rm`                        |
| Help            | `man`, `--help`, `info`                                   |
| Streams         | stdin=0, stdout=1, stderr=2                               |
| Pipe            | `command1 \| command2`                                    |
| Redirect        | `>`, `>>`, `2>`                                           |
| Identity        | `whoami`, `id`, `getent`                                  |
| Permissions     | `r=4`, `w=2`, `x=1`                                       |
| Ownership       | user + group                                              |
| Mode            | `chmod`                                                   |
| Owner           | `chown`                                                   |
| Group owner     | `chgrp`                                                   |
| Default mask    | `umask`                                                   |
| Privilege       | `sudo`, `sudo -l`                                         |
| SSH             | client/server, host identity, user authentication         |
| SSH keys        | private secret, public distributable                      |
| User auth file  | `~/.ssh/authorized_keys` commonly                         |
| Evidence        | raw original + working/derived copy                       |
| Integrity       | SHA-256 detects byte changes against recorded baseline    |
| Hash limitation | Không chứng minh authenticity/completeness                |
| Escalation      | Stop khi vượt scope, authority hoặc safety boundary       |

---

## Kết luận Module Lý thuyết Day 1

Nếu hiểu đúng Day 1, bạn chưa cần trở thành Linux expert. Nhưng bạn phải bắt đầu suy nghĩ như một Infrastructure Operations engineer.

Khi được yêu cầu "cấp quyền cho user", bạn không chỉ nghĩ tới `chmod`.

Bạn phải nghĩ:

```text
Which user?
Which host?
Which environment?
Who approved?
What exact access is required?
Can a group solve it?
What is the current state?
What is the minimum permission?
How will I prove unauthorized access is still denied?
How will I verify?
What will I save as evidence?
How will I rollback?
When must I stop and escalate?
```

Khi được yêu cầu "SSH vào server", bạn không chỉ nghĩ:

```bash
ssh user@server
```

Bạn phải nghĩ:

```text
Am I targeting the correct server?
Is this authorized?
How do I verify server identity?
Which credential should be used?
How is the private key protected?
What privilege does the account actually have?
Do I need sudo?
What actions are allowed?
What evidence should I retain?
```

Và khi được yêu cầu "gửi evidence", bạn không chỉ chụp screenshot. Syllabus yêu cầu một evidence model có **source, host/asset, environment, timestamp/timezone, command/query, collector, ticket reference, size/count, SHA-256, classification và traceability**, đồng thời original phải được giữ nguyên và transformations phải được ghi nhận. Infrastructure_Operations_14_Da… Common evidence contract của chương trình tiếp tục yêu cầu exact filters/commands, format/encoding, parent evidence ID và explicit limitations/errors. Infrastructure_Operations_14_Da…

Đó chính là nền móng mà Day 2–14 sẽ dựa lên: **không chỉ biết làm, mà phải biết làm đúng scope, đúng privilege, an toàn, verify được, troubleshoot được, rollback được, audit được và biết lúc nào phải escalate.**
