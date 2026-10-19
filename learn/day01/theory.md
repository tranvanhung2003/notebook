# Day 1 — Module Lý thuyết

## Infrastructure Operations Foundation + Linux Administration Foundation

Day 1 là nền móng cho gần như toàn bộ 13 ngày còn lại. Syllabus yêu cầu hai nhóm kiến thức lớn: **enterprise Infrastructure Operations fundamentals** và **Linux administration fundamentals**, từ operational safety, L1/L2 boundaries, least privilege, change control, evidence cho đến Linux architecture, filesystem, CLI, users/groups, permissions, `sudo` và `SSH`. Infrastructure_Operations_14_Da…

Một điểm cần phân biệt ngay từ đầu:

- Những mục được đánh dấu **Theo syllabus** là nội dung bắt buộc của Day 1.
- Những phần giải thích sâu hơn về workflow enterprise, troubleshooting methodology, security model, command behavior… là **Kiến thức bổ sung** nhằm giúp bạn vận hành thực tế. Syllabus liệt kê phạm vi nhưng không quy định chi tiết procedure cụ thể của từng công ty.
- Với L1/L2 boundary, approval flow, naming convention, `sudo` policy, SSH hardening… **organization runbook/policy luôn có quyền ưu tiên**. Không nên coi các ví dụ tổng quát dưới đây là authorization cho production.

---

# 1. Day 1 — Mục tiêu của Module Lý thuyết

Sau Module Lý thuyết, bạn phải có khả năng nhìn một Linux administrative task không chỉ theo kiểu:

```text
"Command nào để làm việc này?"
```

mà theo kiểu:

```text
1. Tôi đang làm trên system nào?
2. Environment là production hay non-production?
3. Tôi có authorization hay không?
4. Current state là gì?
5. Desired state là gì?
6. Operation có impact gì?
7. Quyền tối thiểu cần thiết là gì?
8. Command nào an toàn nhất?
9. Làm sao verify?
10. Nếu fail thì troubleshoot theo thứ tự nào?
11. Nếu gây lỗi thì rollback thế nào?
12. Evidence nào cần lưu?
13. Khi nào phải stop và escalate?
```

Đây mới là mental model của Infrastructure Operations.

---

# 2. Syllabus Coverage Checklist — Module Lý thuyết Day 1

Theo syllabus, toàn bộ Concept/Lecture của Day 1 gồm:

### Infrastructure Operations foundation

- Infrastructure administration roles and responsibilities
- L1/L2 support boundaries
- operational safety
- production/non-production environment separation
- change authorization
- least-privilege principles
- standard operating procedures
- runbooks
- incident escalation
- access management
- configuration baselines
- asset inventory
- audit evidence
- lab readiness

### Linux foundation

- Linux architecture
- kernel
- shell
- distributions
- Red Hat Enterprise Linux (`RHEL`) và các Linux distributions khác
- filesystem hierarchy
- CLI navigation
- absolute paths
- relative paths
- file and directory operations
- command syntax
- pipes
- redirection
- command history
- help utilities
- safe command execution

### Linux identity, permissions và secure administration

- users and groups
- ownership
- file permissions
- `chmod`
- `chown`
- `chgrp`
- `umask`
- `sudo`
- SSH authentication
- SSH key management
- secure administrative access

Đó là toàn bộ phạm vi bắt buộc cần cover trong phần lý thuyết. Infrastructure_Operations_14_Da…

---

# PHẦN I — INFRASTRUCTURE OPERATIONS FOUNDATION

---

# 3. Infrastructure Administration Roles and Responsibilities

## 3.1 Infrastructure Administration là gì?

Infrastructure Administration là hoạt động vận hành, duy trì, kiểm soát và hỗ trợ các thành phần nền tảng mà applications và business services phụ thuộc vào.

Có thể hình dung:

```text
Business Service
      │
      ▼
Application
      │
      ▼
Operating System
      │
      ├── Compute
      ├── Storage
      ├── Network
      ├── Identity
      ├── Security
      ├── Virtualization
      └── Cloud
```

Infrastructure Operations thường không sở hữu business logic của application nhưng chịu trách nhiệm cho rất nhiều dependency bên dưới nó.

Ví dụ một website không truy cập được.

Nguyên nhân có thể nằm ở:

```text
Application bug
DNS
Load balancer
Firewall
Routing
VM
Operating system
Filesystem full
Service stopped
Certificate
Authentication
Storage
Cloud resource
```

Do đó Infrastructure Operations engineer phải biết **xác định layer chịu trách nhiệm** thay vì đoán.

---

## 3.2 Những trách nhiệm cốt lõi

Một Infrastructure Operations engineer thường phải quan tâm tới ít nhất:

### Availability

System có hoạt động và phục vụ workload hay không?

Ví dụ:

```text
Server up?
Service running?
Storage mounted?
Network reachable?
```

### Reliability

System có hoạt động ổn định hay thường xuyên fail?

Một server `up` chưa chắc khỏe.

Ví dụ:

```text
CPU = 99%
Filesystem = 100%
Service restart mỗi 10 phút
Network có packet loss
```

### Security

System có được quản trị theo principle như:

```text
least privilege
controlled access
authorized changes
credential protection
auditability
```

hay không.

### Maintainability

System có dễ bảo trì và troubleshoot hay không?

Một environment không có baseline, inventory hoặc documentation sẽ rất khó vận hành dù hiện tại vẫn chạy.

### Recoverability

Nếu xảy ra lỗi:

```text
Có backup?
Có rollback?
Có recovery procedure?
Có biết current/previous configuration?
```

### Auditability

Có chứng minh được:

```text
Ai?
Đã làm gì?
Ở đâu?
Khi nào?
Tại sao?
Kết quả?
```

hay không.

---

# 4. Operational mindset: "working" không đồng nghĩa với "correct"

Một người mới thường đánh giá task bằng:

```text
Command chạy thành công.
→ Xong.
```

Infrastructure Operations phải đánh giá sâu hơn:

```text
Command chạy thành công
        │
        ▼
State có thay đổi đúng chưa?
        │
        ▼
Service có hoạt động đúng chưa?
        │
        ▼
Security có bị yếu đi không?
        │
        ▼
Dependency có bị ảnh hưởng không?
        │
        ▼
Monitoring có healthy không?
        │
        ▼
Evidence đã đủ chưa?
```

Ví dụ:

```bash
chmod 777 application.conf
```

Command có thể return:

```text
exit code 0
```

và application bắt đầu đọc được file.

Nhưng về security, đây có thể là một thay đổi rất tệ vì mọi user đều có khả năng `read`, `write`, hoặc tùy object còn có `execute`.

Do đó:

> `Technical success` không nhất thiết là `operational success`.

---

# 5. L1/L2 Support Boundaries

## 5.1 Theo syllabus

Day 1 yêu cầu hiểu `L1/L2 support boundaries`. Infrastructure_Operations_14_Da…

Syllabus không định nghĩa cụ thể L1 và L2 được làm command nào, vì boundary thực tế phụ thuộc organization.

Đây là điểm cực kỳ quan trọng.

Không tồn tại rule universal rằng:

```text
L1 luôn được restart service
```

hoặc:

```text
L2 luôn được sửa firewall
```

Mỗi organization có:

- RBAC riêng;
- SOP riêng;
- approval model riêng;
- support contract riêng;
- segregation of duties riêng.

---

# 6. L1 thường làm gì?

Một mô hình enterprise phổ biến là L1 thiên về:

```text
Receive
→ Validate
→ Triage
→ Collect evidence
→ Execute approved basic procedure
→ Verify
→ Escalate
```

Ví dụ Linux incident:

> User báo không truy cập được server qua SSH.

L1 có thể được phép kiểm tra:

```bash
ping <host>
```

nếu ICMP được cho phép.

Kiểm tra DNS:

```bash
getent hosts <hostname>
```

Kiểm tra current identity:

```bash
whoami
```

Kiểm tra thông tin được runbook cho phép.

Sau đó document:

```text
Target:
Timestamp:
Source:
Symptom:
Connectivity result:
Authentication symptom:
Evidence:
```

L1 thường không nên tự ý sửa security control chỉ để "cho chạy".

Ví dụ SSH fail không có nghĩa là được phép:

```text
disable security
change SSH policy
give user root
open firewall broadly
```

---

# 7. L2 thường làm gì?

L2 thường có technical scope lớn hơn:

```text
configuration analysis
permission troubleshooting
service administration
identity troubleshooting
log correlation
authorized remediation
```

Ví dụ:

SSH login fail vì:

```text
incorrect file ownership
incorrect SSH key permission
wrong group membership
sudo configuration problem
```

L2 có thể điều tra sâu và thực hiện remediation nếu procedure và authorization cho phép.

---

# 8. Escalation boundary

Một engineer giỏi không phải người cố giải quyết mọi thứ.

Một engineer giỏi biết:

```text
"I can investigate this."
```

khác với:

```text
"I am authorized to change this."
```

Các tình huống nên stop/escalate thường gồm:

- ngoài support scope;
- thiếu authorization;
- production impact không xác định;
- cần credential/quyền mà mình không được cấp;
- root cause thuộc team khác;
- procedure không tồn tại;
- evidence cho thấy cybersecurity incident;
- change có thể gây outage diện rộng;
- rollback chưa rõ;
- current state khác baseline nghiêm trọng;
- có dấu hiệu data loss.

---

# 9. Operational Safety

Operational safety là nguyên tắc vận hành nhằm giảm nguy cơ:

```text
outage
data loss
security exposure
configuration drift
unrecoverable change
audit violation
```

Một command có thể hoàn toàn hợp lệ về syntax nhưng vẫn nguy hiểm về operation.

Ví dụ:

```bash
rm -rf <directory>
```

Có thể syntactically valid.

Câu hỏi quan trọng hơn là:

```text
Target có đúng không?
Directory đó chứa gì?
Có backup?
Có dependency?
Có authorization?
```

---

# 10. Mô hình thao tác an toàn

Một workflow tốt:

```text
1. Identify
2. Observe
3. Understand
4. Authorize
5. Capture baseline
6. Execute smallest necessary change
7. Verify
8. Monitor
9. Document
10. Roll back or escalate if needed
```

### Identify

Bạn đang làm trên machine nào?

```bash
hostname
```

Ai đang login?

```bash
whoami
id
```

Current directory là gì?

```bash
pwd
```

Những câu hỏi này rất đơn giản nhưng tránh được rất nhiều lỗi.

---

# 11. Principle: verify before modify

Ví dụ cần thay permission.

Không nên bắt đầu bằng:

```bash
chmod 640 important.conf
```

Nên inspect trước:

```bash
ls -l important.conf
stat important.conf
```

Bạn cần biết state ban đầu.

Nếu change gây lỗi mà không biết original state, rollback sẽ trở thành đoán.

---

# 12. Principle: smallest possible change

Giả sử application không đọc được một file.

Không nên ngay lập tức:

```bash
chmod 777 file
```

Cần xác định:

```text
Which user runs the process?
Which group?
Which permission is actually missing?
Which directory permissions affect traversal?
```

Sau đó sửa đúng minimum required access.

Đây là intersection giữa:

```text
operational safety
+
least privilege
+
troubleshooting
```

---

# 13. Production và Non-Production Separation

## 13.1 Production là gì?

`Production` là environment đang trực tiếp phục vụ production workload/business users/customer workload theo định nghĩa của organization.

Production thường có yêu cầu cao hơn về:

- availability;
- security;
- monitoring;
- change control;
- backup;
- audit;
- access;
- recovery.

---

## 13.2 Non-production

Có thể bao gồm:

```text
Development
Test
QA
UAT
Staging
Sandbox
Lab
Training
```

Nhưng không nên giả định mọi non-production environment đều không quan trọng.

Một `UAT` system vẫn có thể chứa sensitive data hoặc phụ thuộc business testing quan trọng.

---

# 14. Environment separation không chỉ là đặt tên

Ví dụ:

```text
prod-web01
test-web01
```

Có tên khác nhau chưa có nghĩa đã separation đúng.

Separation thực sự có thể liên quan:

```text
Different accounts
Different VLAN/VPC
Different credentials
Different RBAC
Different secrets
Different databases
Different backup policy
Different change policy
Different monitoring
```

---

# 15. Vì sao phải phân biệt environment?

Imagine bạn muốn test:

```bash
sudo userdel testuser
```

Nếu accidentally đang SSH vào production host thay vì lab host, hậu quả có thể hoàn toàn khác.

Do đó trước state-changing command, habit tốt là verify:

```bash
hostname
whoami
pwd
```

và đối chiếu ticket/environment inventory.

---

# 16. Production safety principle

Trước một production change, nên trả lời được:

```text
What?
Why?
Where?
When?
Who approved?
What impact?
How to verify?
How to roll back?
```

Nếu không trả lời được các câu trên, change chưa sẵn sàng.

---

# 17. Change Authorization

## 17.1 Change là gì?

Trong Infrastructure Operations, `change` là modification làm thay đổi state/configuration của managed environment.

Ví dụ:

```text
create user
change permissions
change sudo rights
modify SSH configuration
restart critical service
change network route
upgrade package
```

Không phải mọi change đều có cùng mức risk.

---

# 18. Read-only vs configuration-changing

Đây là distinction cực kỳ hữu ích.

### Read-only / observational

Ví dụ:

```bash
id
ls -l
pwd
hostname
cat /etc/os-release
```

Mục đích chính là inspect.

### Configuration-changing

Ví dụ:

```bash
useradd
usermod
chmod
chown
chgrp
```

Các command này thay đổi system state.

### Destructive

Ví dụ:

```bash
rm
userdel
```

có khả năng remove object/data.

Do đó mức approval và care phải tăng theo impact.

---

# 19. Change authorization ≠ technical permission

Đây là một nguyên tắc cần nhớ suốt chương trình:

```text
sudo permission ≠ business authorization
```

Bạn có thể technically chạy:

```bash
sudo chmod ...
```

nhưng vẫn không được phép làm nếu:

```text
không có ticket
không có approval
ngoài maintenance window
ngoài support scope
```

Technical ability không thay thế governance.

---

# 20. Anatomy của một controlled change

Một change tốt thường có:

```text
Change ID
Reason
Scope
Affected systems
Pre-checks
Implementation steps
Verification
Risk
Rollback/backout
Owner
Approval
Maintenance window
Post-check
Evidence
```

Day 11 sẽ học ITSM/ServiceNow chi tiết hơn; Day 1 chỉ cần xây mental model này. Syllabus dành phần formal ITSM/change workflow sâu hơn cho Day 11. Infrastructure_Operations_14_Da…

---

# 21. Least Privilege

## 21.1 Định nghĩa

`Least privilege` nghĩa là:

> Một identity chỉ nên có đúng mức access cần thiết để hoàn thành nhiệm vụ được authorize, trong phạm vi và thời gian cần thiết.

Có ba câu hỏi:

```text
Who needs access?
Access to what?
For what operation?
```

Và thường thêm:

```text
For how long?
```

---

# 22. Ví dụ least privilege

Requirement:

> User cần đọc report trong directory `/srv/reports`.

Bad design:

```text
Give root access.
```

Better design:

```text
group-based read access
```

Requirement:

> Operator chỉ cần restart một approved service.

Bad design:

```text
Give unrestricted sudo.
```

Better design:

```text
sudo rule limited to required operation
```

---

# 23. Least privilege theo nhiều layer

Không nên nghĩ least privilege chỉ liên quan `sudo`.

Nó tồn tại ở:

```text
Identity
Group membership
Filesystem permissions
sudo rights
SSH access
Network access
Application permissions
Cloud IAM
Database permissions
```

Các Day sau sẽ tiếp tục lặp lại nguyên tắc này ở AWS, Nutanix, firewall và IAM audit.

---

# 24. Defense in Depth

**Kiến thức bổ sung**

Least privilege thường hoạt động cùng `defense in depth`.

Ví dụ administrative access có thể cần đồng thời:

```text
Approved source
   +
Network reachability
   +
SSH authentication
   +
Valid Linux user
   +
Filesystem permissions
   +
sudo authorization
```

Một control fail chưa nhất thiết làm toàn bộ environment mất security.

---

# 25. SOP — Standard Operating Procedure

## 25.1 SOP là gì?

`SOP` là procedure chuẩn của organization cho một loại operation.

Ví dụ:

```text
Linux User Provisioning SOP
Server Access Review SOP
Incident Escalation SOP
```

SOP có thể nói:

```text
Who may request?
Who approves?
Required fields?
Which naming standard?
Required verification?
Required evidence?
```

---

# 26. Tại sao SOP quan trọng?

Nếu 10 engineers cùng xử lý cùng một task bằng 10 cách khác nhau:

```text
inconsistency
configuration drift
audit difficulty
higher operational risk
```

SOP giúp standardize.

---

# 27. Runbook

`Runbook` thường thiên về execution cụ thể hơn.

Ví dụ:

```text
Runbook: SSH Authentication Failure
```

Có thể yêu cầu:

```text
1. Confirm correct target
2. Check network reachability
3. Capture exact SSH error
4. Confirm account
5. Check public-key deployment
6. Check ownership/permissions
7. Check relevant server logs
8. Remediate only within authorized scope
9. Verify
10. Escalate if unresolved
```

---

# 28. SOP vs Runbook

Một cách dễ nhớ:

| SOP                             | Runbook                                 |
| ------------------------------- | --------------------------------------- |
| quy định quy trình chuẩn        | hướng dẫn execution cho scenario cụ thể |
| governance-oriented             | operations-oriented                     |
| thường rộng hơn                 | thường chi tiết hơn                     |
| mô tả responsibilities/approval | mô tả checks/actions                    |

Trong thực tế hai thuật ngữ có thể được organization dùng hơi khác nhau.

Không nên tranh luận tên gọi; quan trọng là tuân thủ tài liệu chính thức.

---

# 29. Incident Escalation

## 29.1 Incident là gì ở mức nền tảng?

Có thể hiểu đơn giản là:

> Một interruption, degradation hoặc abnormal behavior cần investigation/restoration theo process của organization.

Day 11 mới đi sâu ITSM formal definitions.

---

# 30. Escalation có hai chiều

### Functional escalation

Chuyển cho team có specialization/quyền phù hợp.

Ví dụ:

```text
Linux L1
→ Linux L2
→ Linux engineering
```

hoặc:

```text
Linux Ops
→ Network team
```

### Hierarchical/management escalation

Dùng khi cần attention do:

```text
high business impact
SLA risk
major incident
approval
resource coordination
```

Chi tiết thực tế phụ thuộc organization.

---

# 31. Một escalation tốt cần gì?

Không gửi:

> "SSH không được. Nhờ check."

Một escalation tốt nên chứa:

```text
Incident/ticket ID
System
Environment
Timestamp/timezone
User/source
Exact symptom
Expected behavior
Observed behavior
Checks performed
Relevant outputs
Changes made
Current impact
Reason for escalation
```

---

# 32. Evidence before escalation

Ví dụ exact symptom:

```text
Permission denied (publickey)
```

tốt hơn:

```text
SSH error.
```

Bạn cần giữ raw error bởi vì:

```text
Connection timed out
Connection refused
No route to host
Permission denied
Host key verification failed
```

là các failure classes hoàn toàn khác nhau.

---

# 33. Access Management

Access management là lifecycle quản lý quyền truy cập.

Một model đơn giản:

```text
Request
  ↓
Validate identity
  ↓
Approval
  ↓
Provision
  ↓
Verify
  ↓
Use
  ↓
Review
  ↓
Modify / Revoke
```

---

# 34. Authentication và Authorization

Hai concept phải phân biệt.

### Authentication

> Bạn là ai?

Ví dụ:

```text
Password
SSH key
MFA
```

### Authorization

> Sau khi xác minh bạn là ai, bạn được phép làm gì?

Ví dụ:

```text
Can user read file?
Can user run sudo?
Can user access this server?
```

SSH login thành công chỉ chứng minh authentication + một phần account authorization.

Nó không có nghĩa user được phép làm mọi operation.

---

# 35. Authentication ≠ Authorization

Mental model:

```text
User
 │
 │ authentication
 ▼
Identity verified
 │
 │ authorization
 ▼
Allowed operations/resources
```

Ví dụ:

```text
SSH login successful
```

nhưng:

```bash
cat /root/secret
```

bị:

```text
Permission denied
```

Hoàn toàn bình thường.

---

# 36. Access lifecycle risks

Các access risks thường gặp:

```text
Dormant account
Orphaned account
Excessive privilege
Incorrect group membership
Old SSH key
Shared account
Permanent privilege
Unrevoked access
```

Day 13 sẽ mở rộng thành IAM review; Day 1 chỉ cần hiểu foundation.

---

# 37. Configuration Baseline

## 37.1 Baseline là gì?

`Configuration baseline` là documented reference state dùng để biết system dự kiến phải như thế nào hoặc đã ở trạng thái nào tại thời điểm kiểm soát.

Ví dụ Linux baseline:

```text
hostname
OS distribution
OS version
kernel
network identity
users/groups
filesystem permissions
SSH configuration
sudo policy
```

---

# 38. Baseline vs Current State

Troubleshooting rất thường là:

```text
Expected/Baseline State
          VS
Current State
```

Ví dụ:

Baseline:

```text
Owner = app
Group = appops
Mode = 640
```

Current:

```text
Owner = root
Group = root
Mode = 600
```

Bạn đã có hypothesis rất mạnh về permission issue.

---

# 39. Configuration Drift

**Kiến thức bổ sung**

`Configuration drift` là việc actual configuration dần khác approved/expected configuration.

Nguồn drift:

```text
manual changes
emergency fixes
automation errors
incomplete rollback
package updates
unauthorized changes
```

Không phải mọi difference đều là lỗi, nhưng difference phải được hiểu.

---

# 40. Asset Inventory

`Asset inventory` trả lời:

```text
Chúng ta đang quản lý những tài sản nào?
```

Đối với server:

```text
hostname
asset identifier
environment
owner
operating system
version
IP
hardware/VM/cloud identity
location
service role
```

---

# 41. Inventory khác baseline

Hai khái niệm liên quan nhưng không giống nhau.

### Asset inventory

```text
What exists?
```

### Configuration baseline

```text
How should/configured is it?
```

Ví dụ:

Inventory:

```text
rhel-app01 exists
```

Baseline:

```text
RHEL version X
Owner team = AppOps
SSH policy = approved profile
```

---

# 42. Audit Evidence

## 42.1 Evidence là gì?

`Audit evidence` là material có thể hỗ trợ chứng minh:

```text
configuration
action
state
approval
result
```

Ví dụ:

```text
Command output
Log record
Screenshot
Config extract
Ticket ID
Timestamp
Change record
Inventory report
```

---

# 43. Evidence phải có context

Output:

```text
uid=1001(ops1) gid=1001(ops1)
```

một mình chưa chắc đủ.

Evidence tốt hơn có context:

```text
Host: rhel-lab01
Timestamp: 2026-...
Command: id ops1
Result: ...
Purpose: Verify account/group state
Ticket: LAB-D1-...
```

---

# 44. Evidence integrity

Bạn cần tránh:

- sửa output thủ công;
- cắt mất error quan trọng;
- trộn output từ nhiều hosts không label;
- mất timestamp;
- chứa secret;
- screenshot không có system context.

Evidence phải hữu ích cho người khác review.

---

# 45. Secret không phải audit evidence để chia sẻ

Không bao giờ đưa vào evidence package:

```text
Password
Private SSH key
Secret key
Token
Session secret
Recovery secret
```

Chụp screenshot chứa private key rồi gửi auditor không phải security.

---

# 46. Lab Readiness

Syllabus bao gồm `lab readiness`, bởi vì lab tốt phải được chuẩn bị trước khi execute. Infrastructure_Operations_14_Da…

Mental checklist:

```text
Authorized lab target?
Correct account?
Correct network?
Administrative workstation ready?
SSH client available?
Evidence location ready?
No real secrets?
Rollback possible?
```

Đây là bridge từ theory sang Module Thực hành.

---

# PHẦN II — LINUX ARCHITECTURE FOUNDATION

---

# 47. Linux là gì?

Ở mức chính xác về kiến trúc:

`Linux` ban đầu và về mặt technical core là **kernel**.

Một system mà người dùng gọi chung là "Linux" thường bao gồm:

```text
Linux kernel
+
system libraries
+
user-space utilities
+
shells
+
package management
+
services
+
distribution-specific components
```

Ví dụ:

```text
RHEL
Ubuntu
Debian
Rocky Linux
AlmaLinux
```

đều sử dụng Linux kernel nhưng organization, lifecycle, tooling và package ecosystem có thể khác nhau.

---

# 48. High-level Linux architecture

Mental model:

```text
+-----------------------------------+
| User / Administrator              |
+-----------------------------------+
                │
                ▼
+-----------------------------------+
| Applications / CLI utilities      |
+-----------------------------------+
                │
                ▼
+-----------------------------------+
| Shell / User-space libraries      |
+-----------------------------------+
                │
                ▼
+-----------------------------------+
| System Call Interface             |
+-----------------------------------+
                │
                ▼
+-----------------------------------+
| Linux Kernel                      |
| process / memory / FS / network   |
+-----------------------------------+
                │
                ▼
+-----------------------------------+
| Hardware                          |
| CPU / RAM / disk / NIC / devices  |
+-----------------------------------+
```

Không phải mọi command đi "qua shell" theo cùng một cách về implementation, nhưng model này đủ tốt để xây mental model ban đầu.

---

# 49. User Space và Kernel Space

**Kiến thức bổ sung cần thiết**

Linux phân tách logic giữa:

### Kernel space

Nơi kernel thực hiện privileged operations.

### User space

Nơi phần lớn applications/processes chạy.

Applications không tự ý trực tiếp điều khiển hardware tùy ý.

Chúng sử dụng interfaces do kernel cung cấp.

Điều này giúp:

```text
isolation
security
stability
resource control
```

---

# 50. Kernel

## 50.1 Kernel làm gì?

Các responsibility quan trọng gồm:

### Process management

Kernel schedule các processes/threads sử dụng CPU.

### Memory management

Quản lý:

```text
virtual memory
physical memory
page allocation
```

### Filesystem interaction

Cung cấp abstraction để processes đọc/ghi filesystems.

### Device management

Thông qua device drivers.

### Networking

TCP/IP stack và network interfaces liên quan nằm ở kernel layer.

### Security primitives

Kernel enforce nhiều access decisions liên quan user IDs, permissions và capabilities.

---

# 51. Kiểm tra kernel

Ví dụ:

```bash
uname -r
```

Có thể trả:

```text
5.14.0-...
```

Ý nghĩa:

```text
-r
```

yêu cầu kernel release.

Có thể xem thêm:

```bash
uname -a
```

Nhưng trong evidence, tránh thu quá nhiều output không cần thiết nếu mục tiêu chỉ cần kernel release.

---

# 52. Kernel version và OS version không giống nhau

Ví dụ:

```bash
uname -r
```

cho kernel.

Trong khi:

```bash
cat /etc/os-release
```

cho distribution/release metadata.

Không nên nói:

> "`uname -r` cho biết RHEL version."

Không chính xác.

---

# 53. Shell

## 53.1 Shell là gì?

`Shell` là command interpreter/environment cho phép user tương tác với operating system thông qua commands.

Ví dụ:

```text
Bash
sh
zsh
```

Day 1 tập trung nền tảng CLI và thường sử dụng `Bash`.

---

# 54. Terminal và Shell khác nhau

Đây là confusion rất phổ biến.

### Terminal / terminal emulator

Là interface/window cung cấp môi trường nhập và hiển thị text.

### Shell

Là chương trình interpret command.

Một terminal có thể chạy:

```text
bash
zsh
sh
```

Do đó:

> Terminal không phải shell.

---

# 55. Shell prompt

Prompt thường có dạng như:

```text
[user@host ~]$
```

hoặc privileged shell có thể thấy:

```text
[root@host ~]#
```

Nhưng không nên chỉ dựa vào `$`/`#` để xác định privilege.

Verify bằng:

```bash
whoami
id
```

---

# 56. Shell thực hiện command như thế nào?

Simplified flow:

```text
User enters command
       ↓
Shell parses command
       ↓
Expansion/redirection/pipes
       ↓
Find builtin or executable
       ↓
Execute
       ↓
Receive exit status
       ↓
Display/process output
```

Ví dụ:

```bash
ls -l /etc
```

Shell phải hiểu:

```text
Command = ls
Option = -l
Argument = /etc
```

---

# 57. Shell builtin vs external command

**Kiến thức bổ sung**

Một số commands là shell builtins.

Ví dụ trong Bash:

```bash
cd
history
```

Một số là executable bên ngoài.

Ví dụ thường gặp:

```bash
/usr/bin/ls
```

Bạn có thể tìm loại command bằng:

```bash
type cd
type ls
```

Điều này giải thích vì sao `cd` cần là builtin trong shell: nó phải thay đổi working directory của chính shell hiện tại.

---

# 58. Linux Distributions

## 58.1 Distribution là gì?

Một Linux distribution package hóa:

```text
Linux kernel
userspace utilities
libraries
package manager
repositories
installer
defaults
support lifecycle
security updates
```

thành một operating system usable.

---

# 59. Các distributions phổ biến

Ví dụ:

```text
Red Hat Enterprise Linux
Ubuntu
Debian
SUSE Linux Enterprise Server
Rocky Linux
AlmaLinux
```

Không nên giả định command/package naming giống hoàn toàn trên mọi distro.

---

# 60. RHEL

`Red Hat Enterprise Linux` là enterprise Linux distribution của Red Hat.

Trong syllabus, RHEL là platform Linux quan trọng xuyên suốt Day 1–2. Day 2 đi sâu vào `dnf/yum/rpm`, repositories, `systemd`, SELinux, networking, storage và logs. Infrastructure_Operations_14_Da…

Day 1 cần nhận diện được RHEL system và hiểu Linux foundation trước khi vào system administration sâu hơn.

---

# 61. Distribution identification

Common method:

```bash
cat /etc/os-release
```

Output thường chứa fields như:

```text
NAME=
VERSION=
ID=
VERSION_ID=
```

Tùy distro giá trị khác nhau.

Đừng hard-code assumption dựa chỉ vào hostname.

---

# PHẦN III — LINUX FILESYSTEM HIERARCHY

---

# 62. Một filesystem tree duy nhất

Linux filesystem nhìn từ user space được tổ chức bắt đầu tại:

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
├── usr
├── var
├── tmp
├── dev
├── proc
├── sys
└── ...
```

Đừng nhầm:

```text
/
```

root directory

với:

```text
/root
```

home directory thường dùng cho user `root`.

---

# 63. `/etc`

Thông thường chứa system-wide configuration.

Ví dụ liên quan Day 1:

```text
/etc/passwd
/etc/group
/etc/ssh/
```

Một principle:

> `/etc` thường chứa configuration, không phải nơi tùy tiện đặt personal data.

---

# 64. `/home`

Thông thường chứa home directories của normal users.

Ví dụ:

```text
/home/alice
/home/ops1
```

Home directory có thể chứa:

```text
shell config
user data
.ssh
```

---

# 65. `/root`

Home directory của `root`.

Không phải synonym của `/`.

```text
/
```

và:

```text
/root
```

là hai paths khác nhau.

---

# 66. `/var`

`/var` thường chứa variable data thay đổi trong quá trình system hoạt động.

Ví dụ:

```text
logs
spool data
cache/state depending on application
```

Day 2 sẽ quan tâm `/var/log` nhiều hơn.

---

# 67. `/tmp`

Thường dành cho temporary files.

Không nên giả định data trong `/tmp` là permanent.

Security cũng cần lưu ý vì `/tmp` là shared area trên nhiều systems.

---

# 68. `/usr`

Chứa phần lớn user-space programs, libraries và shared data theo filesystem layout của Linux system.

Bạn sẽ thường gặp:

```text
/usr/bin
/usr/sbin
/usr/lib
```

---

# 69. `/opt`

Thường được dùng cho optional/add-on software, tùy application/vendor.

Không phải mọi software đều nằm ở đây.

---

# 70. `/boot`

Liên quan boot files.

Day 1 chỉ cần nhận biết.

Không nên thay đổi file trong `/boot` như một exercise thông thường.

---

# 71. `/dev`

Chứa device nodes.

Ví dụ system có thể expose block devices dưới:

```text
/dev/...
```

Day 2 storage sẽ đi sâu hơn.

Không nên treat `/dev` như directory chứa ordinary files.

---

# 72. `/proc`

Là pseudo-filesystem expose kernel/process information.

Ví dụ:

```bash
cat /proc/cpuinfo
```

Thông tin được kernel expose, không phải ordinary file persist trên disk như document thông thường.

---

# 73. `/sys`

Pseudo-filesystem cung cấp view về kernel/device model.

Day 1 chỉ cần nhận biết.

---

# PHẦN IV — CLI NAVIGATION VÀ PATHS

---

# 74. Current Working Directory

Mỗi shell process có current working directory.

Xem bằng:

```bash
pwd
```

Ví dụ:

```text
/home/ops1
```

Điều này cực kỳ quan trọng vì relative path được giải quyết từ đây.

---

# 75. `ls`

`ls` dùng để list directory contents.

Ví dụ:

```bash
ls
```

Detailed:

```bash
ls -l
```

Bao gồm hidden entries:

```bash
ls -a
```

Kết hợp:

```bash
ls -la
```

---

# 76. Hidden files

Tên bắt đầu bằng `.` thường không hiện trong `ls` mặc định.

Ví dụ:

```text
.bashrc
.ssh
```

Xem bằng:

```bash
ls -a
```

Hidden không có nghĩa là encrypted hoặc protected.

Nó chỉ liên quan naming/display convention.

---

# 77. `cd`

Change current working directory:

```bash
cd /etc
```

Về home:

```bash
cd ~
```

hoặc nhiều shell khi không có argument:

```bash
cd
```

Parent:

```bash
cd ..
```

---

# 78. `.` và `..`

```text
.
```

current directory.

```text
..
```

parent directory.

Ví dụ:

```bash
./script.sh
```

nghĩa là file `script.sh` nằm trong current directory.

```bash
../file
```

nghĩa là file trong parent directory.

---

# 79. Absolute Path

Absolute path bắt đầu tại root `/`.

Ví dụ:

```text
/etc/ssh/sshd_config
```

Nó không phụ thuộc current working directory.

Nếu bạn đang ở:

```text
/home/alice
```

hay:

```text
/tmp
```

path vẫn chỉ một object đó.

---

# 80. Relative Path

Relative path được resolve từ current working directory.

Nếu:

```text
PWD=/home/alice
```

thì:

```text
documents/report.txt
```

tương ứng:

```text
/home/alice/documents/report.txt
```

---

# 81. Vì sao paths là vấn đề an toàn?

Ví dụ:

```bash
rm file.txt
```

Bạn phải biết current directory.

Nếu nhầm `pwd`, bạn có thể remove file không mong muốn.

Do đó trước destructive command:

```bash
pwd
ls
```

có giá trị rất lớn.

---

# 82. Spaces trong filenames

Shell phân tách arguments theo whitespace trừ khi quoting/escaping.

Ví dụ file:

```text
monthly report.txt
```

Command này sai ý:

```bash
cat monthly report.txt
```

Shell hiểu thành hai arguments.

Phải quote:

```bash
cat "monthly report.txt"
```

Đây là nền tảng shell syntax rất quan trọng.

---

# PHẦN V — FILE AND DIRECTORY OPERATIONS

---

# 83. Tạo directory — `mkdir`

Ví dụ:

```bash
mkdir reports
```

Tạo nested hierarchy:

```bash
mkdir -p reports/2026/october
```

`-p` cho phép tạo parent directories cần thiết và có behavior cụ thể khi directory tồn tại; luôn đọc `man mkdir` trên target system khi cần exact behavior.

---

# 84. Tạo empty file — `touch`

Ví dụ:

```bash
touch notes.txt
```

Cần hiểu `touch` chủ yếu update file timestamps; nếu file chưa tồn tại, thông thường nó tạo empty file.

Không nên mô tả `touch` đơn giản là "command tạo file" mà bỏ qua chức năng chính của nó.

---

# 85. Copy — `cp`

```bash
cp source.txt destination.txt
```

Copy directory thường dùng recursive option:

```bash
cp -r sourcedir targetdir
```

Nhưng production operation cần cân nhắc metadata preservation và workload-specific requirements; Day 1 chỉ học basic file operation.

---

# 86. Move/Rename — `mv`

Rename:

```bash
mv old.txt new.txt
```

Move:

```bash
mv report.txt archive/
```

`mv` có thể overwrite tùy circumstances/options, vì vậy inspect destination trước.

---

# 87. Remove — `rm`

Remove file:

```bash
rm file.txt
```

Recursive remove:

```bash
rm -r directory
```

Đây là destructive operation.

Không có universal "Recycle Bin" ở CLI shell operation này để bạn dựa vào.

Vì vậy:

```text
verify path
verify target
verify backup/recovery
verify authorization
```

trước khi remove.

---

# 88. `rmdir`

```bash
rmdir directory
```

thường chỉ remove empty directory.

Điều này làm nó an toàn hơn `rm -r` trong một số simple scenarios.

---

# 89. Safety misconception: `rm -rf`

Bạn cần nhận thức rằng:

```bash
rm -rf ...
```

là combination có khả năng remove recursively và suppress một số interaction/error behavior tùy trường hợp.

Day 1 không nên hình thành thói quen thêm `-rf` vào mọi `rm`.

Trong enterprise operations:

> Dùng option mạnh chỉ khi có lý do rõ ràng.

---

# PHẦN VI — COMMAND SYNTAX

---

# 90. General syntax

Mental model:

```text
command [options] [arguments]
```

Ví dụ:

```bash
ls -l /etc
```

- `ls`: command
- `-l`: option
- `/etc`: argument

---

# 91. Short options và long options

Nhiều GNU utilities hỗ trợ:

```text
-h
```

và:

```text
--help
```

Nhưng không phải mọi command đều có cùng option conventions.

Không đoán option.

Check:

```bash
command --help
```

hoặc:

```bash
man command
```

---

# 92. Case sensitivity

Linux filenames thường case-sensitive trên common Linux filesystems.

Ví dụ:

```text
Report.txt
report.txt
```

có thể là hai files khác nhau.

Commands cũng case-sensitive:

```text
LS
```

không phải `ls`.

---

# 93. Exit status

**Kiến thức bổ sung nhưng cực quan trọng**

Commands trả exit status.

Xem exit status gần nhất:

```bash
echo $?
```

Convention phổ biến:

```text
0 = success
non-zero = some form of failure/condition
```

Nhưng meaning cụ thể của non-zero status tùy command.

Day 2 sẽ học exit-code handling sâu hơn. Infrastructure_Operations_14_Da…

---

# PHẦN VII — STANDARD INPUT, OUTPUT, PIPES VÀ REDIRECTION

---

# 94. Ba standard streams

Một process CLI thường có:

```text
stdin  = standard input
stdout = standard output
stderr = standard error
```

Mental model:

```text
        stdin
          │
          ▼
     [ command ]
       │      │
       │      └── stderr
       ▼
     stdout
```

Hiểu ba streams này là nền tảng của Linux automation.

---

# 95. Pipe `|`

Pipe nối `stdout` của command bên trái vào `stdin` của command bên phải.

Ví dụ:

```bash
cat /etc/passwd | grep ops
```

Conceptually:

```text
cat
 │ stdout
 ▼
pipe
 │ stdin
 ▼
grep
```

---

# 96. Không cần `cat` mọi lúc

Ví dụ:

```bash
grep ops /etc/passwd
```

đơn giản hơn:

```bash
cat /etc/passwd | grep ops
```

Cả hai giúp minh họa khác nhau:

- dạng `cat | grep` dễ hiểu pipeline;
- dạng `grep file` gọn hơn cho use case này.

Operationally, nên chọn cách rõ ràng.

---

# 97. Output redirection `>`

```bash
command > output.txt
```

Redirect `stdout` vào file.

Quan trọng:

```text
>
```

thường truncate/overwrite destination file trước khi ghi.

Do đó nó là operation có khả năng làm mất content cũ.

---

# 98. Append `>>`

```bash
command >> output.txt
```

Append vào cuối file thay vì truncate.

Nhưng append không đồng nghĩa với "an toàn hoàn toàn":

- có thể làm file tăng rất lớn;
- có thể làm duplicate data;
- có thể ghi nhầm file.

---

# 99. Redirect stderr

Ví dụ:

```bash
command 2> errors.txt
```

File descriptor convention:

```text
0 = stdin
1 = stdout
2 = stderr
```

---

# 100. Redirect stdout và stderr khác nhau

Ví dụ:

```bash
command > output.txt 2> errors.txt
```

Giúp phân tách normal output và error.

Đây là pattern rất hữu ích cho evidence/troubleshooting.

---

# 101. `/dev/null`

**Kiến thức bổ sung**

Có thể discard output bằng:

```bash
command > /dev/null
```

Nhưng khi troubleshoot, suppress output/error quá sớm có thể làm mất evidence.

Principle:

> Khi investigation, giữ information trước; suppress sau khi hiểu behavior.

---

# 102. Input redirection `<`

Ví dụ:

```bash
command < input.txt
```

Nội dung file được đưa vào `stdin` của command.

Không phải command nào cũng cần hoặc sử dụng stdin theo cách giống nhau.

---

# PHẦN VIII — COMMAND HISTORY

---

# 103. `history`

Trong Bash:

```bash
history
```

có thể hiển thị command history tùy shell configuration.

History giúp:

```text
reproduce commands
review work
avoid retyping
troubleshoot
```

---

# 104. Security risk của history

Không nên đặt secret trên command line nếu có alternative an toàn hơn.

Ví dụ anti-pattern:

```text
somecommand --password SuperSecret123
```

vì command có thể xuất hiện ở:

```text
shell history
process listings
logs
audit systems
```

tùy environment/tool.

---

# 105. History không phải authoritative audit trail

Command history có thể:

```text
không đầy đủ
bị disabled
bị thay đổi
không có context
không phản ánh output
```

Do đó:

> `history` hữu ích cho troubleshooting nhưng không nên mặc định coi nó là audit system chính thức.

---

# PHẦN IX — HELP UTILITIES VÀ SELF-SERVICE DOCUMENTATION

---

# 106. `man`

Manual pages:

```bash
man chmod
```

Typical sections có thể chứa:

```text
NAME
SYNOPSIS
DESCRIPTION
OPTIONS
EXAMPLES
SEE ALSO
```

Kỹ năng đọc `SYNOPSIS` rất quan trọng.

---

# 107. `--help`

Ví dụ:

```bash
chmod --help
```

Thường nhanh hơn `man` để kiểm tra syntax/options.

---

# 108. Shell `help`

Đối với Bash builtins:

```bash
help cd
```

có thể phù hợp hơn:

```bash
man cd
```

tùy environment.

---

# 109. `type`

Hữu ích để biết shell resolve command như thế nào:

```bash
type cd
type ls
```

Có thể cho biết:

```text
builtin
alias
function
executable
```

tùy command/environment.

---

# 110. Rule quan trọng: không đoán syntax

Khi không chắc:

```text
Don't guess
→ Read local help/manual
→ Confirm target version/environment
```

Điều này đặc biệt quan trọng về sau với:

```text
RHEL
AWS CLI
FortiGate
BIG-IP
Nutanix
PowerShell
```

---

# PHẦN X — SAFE COMMAND EXECUTION

---

# 111. Phân loại command theo impact

Một useful mental model:

| Class                   | Ví dụ                    | Risk            |
| ----------------------- | ------------------------ | --------------- |
| Inspection              | `id`, `pwd`, `ls`        | thấp            |
| Data creation           | `mkdir`, `touch`, `cp`   | thấp–trung bình |
| State modification      | `mv`, `chmod`, `chown`   | trung bình      |
| Identity/privilege      | `useradd`, `sudo` config | cao hơn         |
| Destructive             | `rm`, `userdel`          | cao             |
| Security control change | SSH/sudo/security policy | cao             |

Đây không phải formal universal classification, nhưng rất hữu ích.

---

# 112. Pre-flight check

Trước state-changing command:

```text
Target?
Environment?
Identity?
Current state?
Authorization?
Impact?
Rollback?
```

Ví dụ:

```bash
hostname
whoami
pwd
ls -ld <target>
```

không phải ritual vô nghĩa. Đó là risk reduction.

---

# 113. Post-change verification

Không dừng ở:

```text
Command returned no error.
```

Bạn phải verify desired state.

Ví dụ:

```bash
chmod 640 report.txt
```

Sau đó:

```bash
stat report.txt
```

hoặc:

```bash
ls -l report.txt
```

và quan trọng hơn có thể cần kiểm tra bằng identity thực tế.

---

# PHẦN XI — USERS AND GROUPS

---

# 114. Linux User

User identity được kernel và userspace biểu diễn chủ yếu qua numeric `UID`.

Username là human-readable mapping.

Ví dụ:

```text
alice
```

có thể tương ứng:

```text
UID 1001
```

System về nhiều mặt dựa trên UID, không chỉ string username.

---

# 115. User attributes

Một user record thường liên quan:

```text
username
UID
primary GID
home directory
login shell
```

Xem account database qua:

```bash
getent passwd <username>
```

Hoặc local file:

```bash
cat /etc/passwd
```

Nhưng `getent` thường tốt hơn khi muốn account resolution vì nó có thể sử dụng configured name service sources, không chỉ local file.

---

# 116. `/etc/passwd`

Một line điển hình có colon-separated fields dạng:

```text
username:x:UID:GID:comment:home:shell
```

Quan trọng:

Password hash hiện đại thường **không nằm trực tiếp trong `/etc/passwd`**; field `x` thường chỉ reference việc password information được quản lý nơi khác như `/etc/shadow`.

Không cần mở `/etc/shadow` trong lab nếu không có lý do/authorization.

---

# 117. Root user

`root` có:

```text
UID 0
```

Root có extremely broad privilege.

Do đó principle:

```text
Do not work as root by default when not required.
```

Nên dùng controlled privilege elevation theo policy, thường thông qua `sudo`.

---

# 118. System/service accounts

Không phải mọi account đều là human user.

Linux có thể có accounts dùng cho:

```text
services
daemons
applications
```

Không nên disable/delete account chỉ vì bạn không nhận ra tên.

Đầu tiên xác định owner/purpose/dependency.

---

# 119. Group

Groups giúp quản lý access theo tập hợp identities.

Một user có:

```text
primary group
+
zero or more supplementary groups
```

Xem:

```bash
id alice
```

hoặc:

```bash
groups alice
```

---

# 120. Tại sao group quan trọng?

Thay vì:

```text
grant file access individually to 50 users
```

có thể:

```text
create/use operations group
grant group access
manage membership
```

Đây là scalable access management.

---

# 121. `/etc/group`

Local group definitions thường ở:

```text
/etc/group
```

Nhưng tương tự users, enterprise system có thể sử dụng identity source khác.

Do đó:

```bash
getent group <groupname>
```

thường là useful verification method.

---

# 122. Effective identity

Khi troubleshoot access, không chỉ hỏi:

> User tên gì?

Cần hỏi:

```text
UID?
Primary group?
Supplementary groups?
Current effective context?
```

`id` thường là command rất hữu ích.

---

# PHẦN XII — OWNERSHIP

---

# 123. File ownership model

Một filesystem object thường có:

```text
user owner
group owner
```

Ví dụ:

```text
-rw-r----- 1 alice appops 2048 Oct 19 report.txt
```

Relevant fields:

```text
alice  = owner
appops = group
```

---

# 124. Ownership khác permissions

Ownership trả lời:

```text
Who owns the object?
Which group owns it?
```

Permissions trả lời:

```text
What access does owner/group/other receive?
```

Hai khái niệm liên quan nhưng không giống nhau.

---

# 125. Inspect ownership

```bash
ls -l file
```

hoặc chi tiết:

```bash
stat file
```

`stat` hữu ích khi cần:

```text
numeric mode
ownership
timestamps
inode-related information
```

---

# PHẦN XIII — FILE PERMISSIONS

---

# 126. Three permission classes

Traditional UNIX mode chia thành:

```text
user
group
other
```

thường viết:

```text
u
g
o
```

---

# 127. Three basic permissions

```text
r = read
w = write
x = execute
```

Nhưng semantics khác giữa regular file và directory.

Đây là chỗ người mới rất hay nhầm.

---

# 128. Permission trên regular file

### `r`

Cho phép đọc file content.

### `w`

Cho phép modify file content.

### `x`

Cho phép execute file như program/script nếu các điều kiện khác phù hợp.

---

# 129. Permission trên directory

Đây là concept rất quan trọng.

### `r` trên directory

Cho phép đọc/list directory entries theo filesystem semantics.

### `w` trên directory

Cho phép modify directory entries, ví dụ create/remove/rename entries, kết hợp với các conditions khác.

### `x` trên directory

Cho phép traverse/search directory.

Do đó:

> `x` trên directory không có nghĩa "execute directory".

---

# 130. Ví dụ permission string

```text
-rwxr-x---
```

Tách:

```text
-    object type indicator
rwx  user
r-x  group
---  other
```

User:

```text
read write execute
```

Group:

```text
read execute
```

Other:

```text
no permission
```

---

# 131. Numeric permissions

Values:

```text
r = 4
w = 2
x = 1
```

Cộng lại.

Ví dụ:

```text
rw- = 4 + 2 = 6
r-- = 4
--- = 0
```

Nên:

```text
640
```

tương ứng:

```text
rw-r-----
```

---

# 132. Common modes

Một số examples để đọc, không phải recommendation universal:

```text
600 = rw-------
640 = rw-r-----
644 = rw-r--r--
700 = rwx------
750 = rwxr-x---
755 = rwxr-xr-x
```

Đừng học thuộc:

```text
file = 644
directory = 755
```

rồi áp dụng mọi nơi.

Desired permissions phụ thuộc security requirement.

---

# 133. Permission evaluation high-level

Với traditional mode, system xác định class applicable dựa trên identity/ownership và checks permission tương ứng.

Điều quan trọng:

> Permission classes không đơn giản là "cộng tất cả quyền user + group + other".

Nếu process identity match owner, owner bits có vai trò tương ứng; behavior chính xác còn có thể bị ảnh hưởng bởi ACLs, SELinux và các mechanisms khác.

Day 2 sẽ học SELinux. Infrastructure_Operations_14_Da…

---

# 134. Permission denied không nhất thiết do `chmod`

Một error:

```text
Permission denied
```

có thể liên quan:

```text
file mode
directory traversal
ownership
group membership
ACL
SELinux
mount option
application behavior
```

Day 1 tập trung traditional permissions; Day 2 bổ sung SELinux/filesystem context.

Do đó không nên reflexively:

```bash
chmod 777 ...
```

---

# PHẦN XIV — `chmod`

---

# 135. `chmod` là gì?

`chmod` thay đổi file mode bits.

Hai style phổ biến:

### Symbolic

```bash
chmod u+x script.sh
```

### Numeric

```bash
chmod 750 script.sh
```

---

# 136. Symbolic mode

Components:

```text
u = user
g = group
o = other
a = all
```

Operations:

```text
+ add
- remove
= set
```

Ví dụ:

```bash
chmod g+r report.txt
```

Thêm group read.

```bash
chmod o-r report.txt
```

Bỏ other read.

---

# 137. Numeric mode

```bash
chmod 640 report.txt
```

Set:

```text
user  = rw-
group = r--
other = ---
```

Numeric mode rất rõ nếu bạn biết desired full mode.

Symbolic mode hữu ích khi muốn modify một phần mà không thay các bits khác ngoài scope chỉ định.

---

# 138. Operational risk của `chmod`

Bad troubleshooting:

```text
Permission denied
→ chmod 777
```

Tại sao sai?

Vì:

- chưa xác định root cause;
- có thể expose sensitive data;
- cho phép unexpected modification;
- tạo configuration drift;
- che giấu incorrect ownership/group design.

---

# PHẦN XV — `chown` VÀ `chgrp`

---

# 139. `chown`

Thay owner:

```bash
chown alice file.txt
```

Owner và group:

```bash
chown alice:appops file.txt
```

Các operation này thường cần appropriate privilege tùy context.

---

# 140. `chgrp`

Thay group ownership:

```bash
chgrp appops file.txt
```

Useful khi owner không cần thay.

---

# 141. Recursive ownership changes

Commands có thể hỗ trợ recursive changes, ví dụ:

```bash
chown -R ...
```

Nhưng đây là high-risk pattern trên large/important directory trees.

Trước recursive change cần:

```text
confirm path
inspect scope
understand symlink behavior/options
capture baseline
know rollback
```

Day 1 lab nên giữ scope nhỏ và controlled.

---

# PHẦN XVI — `umask`

---

# 142. `umask` là gì?

`umask` ảnh hưởng permission bits bị loại khỏi requested/default creation mode khi programs tạo new filesystem objects.

Xem:

```bash
umask
```

Ví dụ có thể:

```text
0022
```

hoặc:

```text
0002
```

tùy policy/environment.

---

# 143. Mental model

Thường dạy foundation:

```text
Files:
base mode 666

Directories:
base mode 777

Apply umask
→ resulting permissions
```

Ví dụ umask:

```text
022
```

Typical result:

```text
file      644
directory 755
```

Nhưng cần hiểu chính xác hơn:

> `umask` removes permission bits from mode requested by application; application có thể request mode restrictive hơn ngay từ đầu.

Vì vậy không nên tin rằng mọi file với umask `022` chắc chắn sẽ là `644`.

---

# 144. Vì sao regular file base thường không có execute?

Vì arbitrary new data file không nên tự động executable.

Do đó basic model sử dụng:

```text
666
```

không phải:

```text
777
```

cho regular files.

---

# 145. Security implication của umask

Umask quá permissive có thể khiến newly created files accessible rộng hơn yêu cầu.

Umask quá restrictive có thể khiến shared workflow fail.

Do đó desired umask phải theo system/application policy.

---

# PHẦN XVII — `sudo`

---

# 146. `sudo` giải quyết vấn đề gì?

Bạn không muốn mọi administrator:

```text
login trực tiếp root
```

cho mọi việc.

`sudo` hỗ trợ controlled execution theo authorization policy.

Mental model:

```text
User
 │
 ▼
sudo request
 │
 ▼
policy evaluation
 │
 ├── denied
 │
 └── authorized
       │
       ▼
 privileged command
```

---

# 147. `sudo` không có nghĩa "become root"

Common use:

```bash
sudo command
```

thường execute với target user theo policy, thường là root trong common configuration.

Nhưng `sudo` có thể được cấu hình linh hoạt hơn.

Do đó fundamental concept là:

> Controlled privilege delegation.

---

# 148. Check sudo access

Common command:

```bash
sudo -l
```

có thể list permitted sudo commands cho current user, tùy policy/authentication state.

Đây là verification tốt hơn việc cứ thử arbitrary privileged commands.

---

# 149. Least-privilege sudo

Bad design:

```text
ALL commands as root
```

khi requirement chỉ là:

```text
restart one specific service
```

Better principle:

```text
Allow the minimum command set required
```

Tuy nhiên designing secure `sudoers` rules có subtle security details; production rules phải theo organization-approved pattern.

---

# 150. `sudoers`

Sudo configuration thường liên quan:

```text
/etc/sudoers
/etc/sudoers.d/
```

Không nên edit `/etc/sudoers` tùy tiện bằng generic editor.

Common safe mechanism là:

```bash
visudo
```

vì nó hỗ trợ validation/locking behavior phù hợp hơn cho sudo configuration.

Trong lab thực tế chúng ta sẽ chỉ thay đổi theo controlled instructions.

---

# 151. Why sudo matters for audit

Sudo usage có thể giúp:

```text
identify invoking user
limit command scope
create audit/log records
reduce shared root usage
```

nhưng mức audit chi tiết phụ thuộc system configuration.

---

# 152. Dangerous misconception

```text
sudo command succeeded
```

không có nghĩa:

```text
change authorized
```

Nhắc lại:

```text
Technical privilege
≠
Operational authorization
```

---

# PHẦN XVIII — SSH FUNDAMENTALS

---

# 153. SSH là gì?

`SSH` — Secure Shell — là protocol/tool family được dùng phổ biến để secure remote administration.

Typical flow:

```text
Admin workstation
      │
      │ SSH
      ▼
Linux server
```

SSH cung cấp protected channel qua insecure network.

---

# 154. SSH client và server

### Client

Initiates connection.

Ví dụ command:

```bash
ssh user@host
```

### Server

Remote system chạy SSH server implementation/service.

On many Linux systems this is associated with OpenSSH server and `sshd`.

---

# 155. TCP port

SSH mặc định thường sử dụng:

```text
TCP/22
```

Nhưng administrator có thể configure service trên port khác.

Vì vậy:

> `SSH = TCP/22` là default association, không phải universal guarantee.

---

# 156. SSH workflow

High-level:

```text
1. Establish network connection
2. Negotiate SSH protocol/cryptography
3. Verify server identity
4. Authenticate client/user
5. Establish session/channel
6. Apply OS authorization
```

Điểm cần nhớ:

> Authentication không phải bước duy nhất.

---

# 157. Server identity / Host key

Khi kết nối SSH, client cần xác định server mà nó nói chuyện có expected identity hay không.

Host key giúp nhận diện server.

Bạn có thể gặp:

```text
The authenticity of host ... can't be established
```

lần đầu kết nối.

Hoặc:

```text
REMOTE HOST IDENTIFICATION HAS CHANGED
```

khi recorded host identity khác.

---

# 158. Không bypass host-key warning vô điều kiện

Host key mismatch có thể xuất hiện vì:

```text
server rebuilt
host key rotated
DNS/IP reused
legitimate infrastructure change
```

nhưng cũng có thể là security concern.

Do đó không nên chỉ xóa entry rồi tiếp tục mà không xác minh.

Workflow:

```text
Stop
→ Verify target identity
→ Check approved change/rebuild
→ Confirm trusted fingerprint/source
→ Update known host information appropriately
```

---

# 159. SSH user authentication

Common methods có thể gồm:

```text
password
public-key
keyboard-interactive
certificate-based mechanisms
```

tùy server configuration.

Day 1 tập trung mạnh vào SSH key management.

---

# PHẦN XIX — SSH PUBLIC-KEY AUTHENTICATION

---

# 160. Key pair

Asymmetric key pair gồm:

```text
Private key
Public key
```

Private key:

```text
must remain secret
```

Public key:

```text
can be provisioned to authorized target
```

---

# 161. Mental model

```text
Admin Workstation
──────────────────────
Private key
     │
     │ proves possession
     ▼

SSH authentication

     ▲
     │
Authorized public key
──────────────────────
Linux Server
```

Private key không được gửi lên server chỉ để login thông thường.

---

# 162. SSH private key không phải password file

Private key là cryptographic private material.

Nếu attacker lấy được unprotected private key và các controls khác không ngăn được, attacker có thể impersonate key holder trong relevant contexts.

Do đó bảo vệ private key rất quan trọng.

---

# 163. Passphrase

SSH private key có thể được encrypt/protected bằng passphrase tùy key setup.

Passphrase giúp giảm risk khi key file bị đánh cắp.

Nhưng enterprise implementation có thể còn dùng:

```text
agent
hardware-backed key
certificate
centralized access solution
```

Day 1 tập trung basic key handling.

---

# 164. Public key placement

Trên common OpenSSH user configuration, authorized keys thường liên quan:

```text
~/.ssh/authorized_keys
```

nhưng exact server behavior có thể thay đổi theo `sshd` configuration.

Không nên tuyệt đối hóa path này cho mọi organization.

---

# 165. File ownership và mode matters

SSH servers thường enforce security checks đối với key-related files/directories để tránh unauthorized users sửa authentication material.

Typical concern:

```text
Who owns ~/.ssh?
Who can write authorized_keys?
```

Exact accepted modes/settings phải được verify theo server configuration/OpenSSH policy.

Điểm conceptual:

> Authentication material không được writable bởi unauthorized identities.

---

# 166. SSH key generation

Common modern OpenSSH command có thể là:

```bash
ssh-keygen
```

Algorithm selection, key size, hardware-backed key hoặc policy phải theo organizational security standard.

Day 1 không nên tự chọn thuật toán "vì thấy trên Internet" nếu enterprise policy đã quy định.

---

# 167. Key fingerprint

Fingerprint là compact identifier derived from public key.

Nó hữu ích để:

```text
compare
verify
inventory
identify expected key
```

thay vì nhìn nguyên public key blob.

---

# 168. Key lifecycle

Secure key management không kết thúc sau `ssh-keygen`.

Lifecycle:

```text
Generate
  ↓
Protect
  ↓
Register/Deploy
  ↓
Verify
  ↓
Use
  ↓
Review
  ↓
Rotate
  ↓
Revoke/remove
```

Nếu employee rời team nhưng public key vẫn tồn tại trên servers, access governance có vấn đề.

---

# PHẦN XX — SECURE ADMINISTRATIVE ACCESS

---

# 169. Secure administrative access là một chain

Một secure Linux administrative workflow thường không dựa vào một control duy nhất.

Ví dụ:

```text
Approved administrator
        │
        ▼
Approved workstation
        │
        ▼
Approved network path
        │
        ▼
SSH server identity verified
        │
        ▼
Strong authentication
        │
        ▼
Named Linux account
        │
        ▼
Least privilege
        │
        ▼
sudo only when required
        │
        ▼
Audit/evidence
```

---

# 170. Named account vs shared admin account

Named identity có lợi thế audit:

```text
Alice performed X
Bob performed Y
```

Shared account:

```text
admin performed X
```

làm attribution khó hơn.

Organization có thể có exception/break-glass patterns, nhưng normal administration nên ưu tiên traceable identity theo policy.

---

# 171. Root login

Direct root login qua SSH thường bị hạn chế trong security-conscious environments, nhưng exact policy phụ thuộc organization.

Mental model được ưu tiên:

```text
Authenticate as named user
→ elevate with sudo when authorized
```

thay vì:

```text
everyone uses root
```

---

# 172. Secure access does not mean “SSH works”

Bạn cần verify:

```text
Correct server?
Correct account?
Correct key?
Correct privilege?
Denied actions actually denied?
Audit trail generated?
```

Ví dụ sau login:

```bash
hostname
whoami
id
```

---

# PHẦN XXI — TROUBLESHOOTING METHODOLOGY CHO DAY 1

Đây là **kiến thức bổ sung**, nhưng nó kết nối tất cả syllabus topics.

---

# 173. Không troubleshoot bằng random changes

Bad flow:

```text
SSH failed
→ chmod 777
→ restart service
→ change firewall
→ recreate user
→ regenerate key
```

Sau vài phút:

- bạn không biết root cause;
- nhiều state đã thay đổi;
- rollback khó;
- audit evidence tệ;
- có thể tạo thêm failure.

---

# 174. Structured troubleshooting

Dùng flow:

```text
1. Define symptom precisely
2. Determine scope
3. Gather current state
4. Compare expected state
5. Form hypothesis
6. Run low-risk test
7. Narrow layer
8. Change only when justified
9. Verify
10. Document
```

---

# 175. Ví dụ: User nói "SSH không được"

Đừng vội kiểm tra keys.

Đầu tiên phân loại symptom.

### Case A

```text
No route to host
```

Khả năng tập trung network/routing.

### Case B

```text
Connection timed out
```

Có thể network/firewall/service path.

### Case C

```text
Connection refused
```

Target reachable nhưng port/service path có vấn đề.

### Case D

```text
Permission denied (publickey)
```

Bạn đã đi xa hơn trong connection process; investigation nên tập trung authentication/account/key/policy.

### Case E

```text
Host key verification failed
```

Client-side trust/server identity issue.

Điểm chính:

> Exact error quyết định next troubleshooting branch.

---

# 176. Troubleshooting permission issue

Symptom:

```text
Permission denied
```

Investigation order có thể:

```text
1. Confirm user identity
2. Confirm target path
3. Inspect object owner/group
4. Inspect object mode
5. Inspect parent directory traversal permissions
6. Check group memberships
7. Consider ACL/SELinux if traditional mode looks correct
8. Escalate if outside Day 1/scope
```

Day 1 dừng chủ yếu ở traditional user/group/mode analysis.

Day 2 sẽ bổ sung SELinux. Infrastructure_Operations_14_Da…

---

# PHẦN XXII — VERIFICATION MINDSET

---

# 177. Verification phải độc lập với action

Nếu chạy:

```bash
chmod 640 file.txt
```

verification nên là:

```bash
stat file.txt
```

hoặc:

```bash
ls -l file.txt
```

Tốt hơn nữa nếu access requirement quan trọng:

```text
test as intended user/group
```

---

# 178. Positive và negative verification

Security control không chỉ cần:

```text
Authorized user succeeds
```

mà còn:

```text
Unauthorized user fails
```

Ví dụ:

```text
ops1 can read report
guest cannot read report
```

Đây là lý do Assignment của Day 1 yêu cầu verify cả authorized và denied access. Infrastructure_Operations_14_Da…

---

# 179. State verification vs functional verification

Ví dụ permission:

### State

```bash
ls -l file
```

cho thấy:

```text
-rw-r-----
```

### Functional

Test user thực sự có/không có access.

Cả hai hữu ích.

State đúng chưa chắc functionality đúng nếu dependency khác ảnh hưởng.

---

# PHẦN XXIII — EVIDENCE MODEL CHO DAY 1

---

# 180. Một evidence record tốt

Có thể theo mẫu:

```text
Ticket/Lab ID:
Timestamp:
Timezone:
Hostname:
Environment:
Operator:
Objective:
Pre-state:
Command:
Output:
Post-state:
Verification:
Result:
Notes:
```

Không nhất thiết mọi command đều cần full form này, nhưng critical administrative change nên có traceability.

---

# 181. Evidence cho users/groups

Useful examples:

```bash
id <user>
getent passwd <user>
getent group <group>
```

Nhưng redact/scope appropriately nếu output có unnecessary sensitive information.

---

# 182. Evidence cho permissions

```bash
ls -ld <path>
stat <path>
```

Nếu path nằm sâu:

```text
/opt/app/config/file
```

có thể cần evidence của parent directory permissions nếu troubleshooting traversal.

---

# 183. Evidence cho sudo

```bash
sudo -l
```

và controlled positive/negative test theo lab design.

Không cần expose password trong screenshot/recording.

---

# 184. Evidence cho SSH

Có thể gồm:

```text
source workstation identity
target hostname
user
public-key fingerprint
successful login verification
denied test
timestamp
```

Không bao gồm private key.

---

# PHẦN XXIV — COMMON MISTAKES CỦA NGƯỜI MỚI

---

# 185. Mistake: cứ lỗi permission là `chmod 777`

Tại sao nguy hiểm:

```text
over-permission
data modification risk
security exposure
configuration drift
root cause masked
```

Better:

```text
identify required identity
inspect owner/group/mode
grant minimum needed
verify
```

---

# 186. Mistake: dùng root cho mọi việc

Hậu quả:

```text
large blast radius
weak accountability
accidental destructive changes
least-privilege violation
```

---

# 187. Mistake: nhầm `/` và `/root`

```text
/
```

là filesystem root.

```text
/root
```

là directory.

Đây là distinction cơ bản nhưng cực quan trọng.

---

# 188. Mistake: nhầm relative path

Ví dụ:

```bash
rm logs/*
```

Nếu current directory khác dự kiến, target khác.

Do đó:

```bash
pwd
ls
```

trước operation quan trọng.

---

# 189. Mistake: nghĩ hidden file được bảo mật

File:

```text
.secret
```

không secure chỉ vì có dấu `.`.

Access control đến từ permissions/ownership/security mechanisms, không phải hidden naming.

---

# 190. Mistake: nghĩ `sudo` = authorization

Không.

```text
sudo capability
```

là technical privilege.

Ticket/change approval mới là operational authorization theo process.

---

# 191. Mistake: copy private key lên server

Private key phải được bảo vệ phía owner/client theo intended authentication model.

Server nhận authorized public key, không cần private key của user.

---

# 192. Mistake: gặp host-key warning thì bypass

Có thể bỏ qua security signal rất quan trọng.

Phải verify identity/change context.

---

# 193. Mistake: chỉ test success

Security configuration cần cả:

```text
allowed path succeeds
denied path fails
```

---

# 194. Mistake: change rồi mới nghĩ rollback

Rollback phải được nghĩ **trước** change.

Đặc biệt production.

---

# PHẦN XXV — ESCALATION BOUNDARY CHO DAY 1

---

# 195. L1 có thể xử lý đến đâu?

Không có universal boundary.

Nhưng trong training model, L1 nên có khả năng:

```text
identify host/user
inspect basic files
navigate CLI
collect permissions/ownership evidence
capture SSH error
verify known runbook items
perform pre-approved basic actions
```

Nếu required change vượt scope → escalate.

---

# 196. L2 nên làm gì?

L2 có thể được expected:

```text
analyze permission chains
analyze group membership
troubleshoot SSH key access
implement approved ownership/mode corrections
manage controlled users/groups
implement approved sudo configuration
```

nhưng vẫn không vượt organization authorization.

---

# 197. Escalate ngay nếu

Ví dụ:

```text
Suspected security breach
Unknown privileged account
Unexpected SSH key
Production access outside ticket
Potential data loss
Configuration appears tampered
Need to weaken security controls
Need direct root credential not provisioned
Change requires security/network/platform owner
```

---

# PHẦN XXVI — CÁCH CÁC CONCEPT DAY 1 KẾT NỐI VỚI NHAU

Đây là phần cực kỳ quan trọng để tránh học rời rạc.

---

# 198. Scenario: cấp access cho operations user

Requirement:

> Một engineer cần đọc application reports và thực hiện một approved administrative operation.

Bạn không bắt đầu với `chmod`.

Bạn bắt đầu từ governance:

```text
Access request
      ↓
Approval
      ↓
Identify target server
      ↓
Verify environment
      ↓
Create/identify named user
      ↓
Assign correct group
      ↓
Set filesystem ownership/mode
      ↓
Configure minimum sudo if required
      ↓
Deploy SSH public key
      ↓
Test authorized access
      ↓
Test denied access
      ↓
Collect evidence
```

Toàn bộ Day 1 nằm trong scenario này.

---

# 199. Dependency chain

Một user không đọc được file có thể do:

```text
SSH login
     │
     ▼
Linux account
     │
     ▼
Group membership
     │
     ▼
Directory traversal
     │
     ▼
Object ownership
     │
     ▼
Mode permissions
     │
     ▼
Additional security controls
```

Bạn cần troubleshoot theo chain.

---

# 200. Operational control chain

Tương tự, một successful change cần:

```text
Authorization
     ↓
Correct system
     ↓
Correct identity
     ↓
Correct command
     ↓
Minimal scope
     ↓
Successful execution
     ↓
Verification
     ↓
Evidence
```

Nếu một mắt xích thiếu, operation chưa complete về mặt enterprise.

---

# PHẦN XXVII — DAY 1 THEORY COMMAND REFERENCE

Các command dưới đây là **reference cho concepts**, chưa phải lab procedure hoàn chỉnh.

| Mục đích                     | Command điển hình         |
| ---------------------------- | ------------------------- |
| Current user                 | `whoami`                  |
| Identity/groups              | `id`                      |
| Host information             | `hostname`, `hostnamectl` |
| OS information               | `cat /etc/os-release`     |
| Kernel                       | `uname -r`                |
| Working directory            | `pwd`                     |
| List files                   | `ls`, `ls -l`, `ls -la`   |
| Change directory             | `cd`                      |
| Create directory             | `mkdir`                   |
| Create/update file timestamp | `touch`                   |
| Copy                         | `cp`                      |
| Move/rename                  | `mv`                      |
| Remove                       | `rm`, `rmdir`             |
| View content                 | `cat`                     |
| Search text                  | `grep`                    |
| Search filesystem            | `find`                    |
| File metadata                | `stat`                    |
| User lookup                  | `getent passwd <user>`    |
| Group lookup                 | `getent group <group>`    |
| Permission change            | `chmod`                   |
| Owner change                 | `chown`                   |
| Group owner change           | `chgrp`                   |
| Current umask                | `umask`                   |
| Check sudo privileges        | `sudo -l`                 |
| Manual                       | `man <command>`           |
| Bash builtin help            | `help <builtin>`          |
| Basic command help           | `<command> --help`        |
| Command type                 | `type <command>`          |
| History                      | `history`                 |
| SSH client                   | `ssh user@host`           |
| SSH key utility              | `ssh-keygen`              |

---

# PHẦN XXVIII — READ-ONLY VS CHANGE COMMAND REFERENCE

Đây là classification operational, không phải absolute security rating.

### Chủ yếu inspection/read-only

```bash
whoami
id
hostname
hostnamectl
uname -r
pwd
ls
stat
cat
getent
umask
sudo -l
man
```

Lưu ý: một số command có options/modes khác hoặc access side effects tùy system; bảng này nói về common usage Day 1.

### Thay đổi state

```bash
mkdir
touch
cp
mv
chmod
chown
chgrp
useradd
usermod
groupadd
ssh-keygen
```

### Destructive hoặc cần extra care

```bash
rm
userdel
groupdel
```

Và recursive variants cần đặc biệt thận trọng.

---

# PHẦN XXIX — MENTAL MODEL VỀ SECURITY DAY 1

Bạn nên nhớ bốn tầng:

```text
IDENTITY
Who are you?
    │
    ▼
AUTHENTICATION
Can you prove it?
    │
    ▼
AUTHORIZATION
What are you allowed to do?
    │
    ▼
AUDIT
Can we prove what happened?
```

Linux mapping:

```text
Identity
→ user / UID / group

Authentication
→ SSH key / other approved method

Authorization
→ permissions / groups / sudo

Audit/Evidence
→ logs / command output / ticket / evidence
```

Đây là một trong những mental models quan trọng nhất của Day 1.

---

# PHẦN XXX — MENTAL MODEL VỀ FILE ACCESS

Khi user hỏi:

> "Tại sao tôi không mở được file?"

Hãy nghĩ:

```text
Who am I?
    ↓
What UID/GIDs do I have?
    ↓
What path am I accessing?
    ↓
Can I traverse parent directories?
    ↓
Who owns the target?
    ↓
What are its mode bits?
    ↓
Are there additional controls?
    ↓
What exact operation is attempted?
```

Không nhảy ngay vào:

```text
chmod
```

---

# PHẦN XXXI — MENTAL MODEL VỀ SSH

Khi SSH fail:

```text
Can I resolve/reach target?
        ↓
Can I establish TCP connection?
        ↓
Is SSH server responding?
        ↓
Does server identity match?
        ↓
Is user valid?
        ↓
Does authentication succeed?
        ↓
Does account/session authorization allow login?
```

Ở Day 1 bạn tập trung identity/authentication/access.

Day 2 sẽ bổ sung sâu hơn network/firewall/service/log troubleshooting. Infrastructure_Operations_14_Da…

---

# PHẦN XXXII — MENTAL MODEL VỀ SUDO

Khi:

```text
sudo command
```

fail, đừng mặc định password sai.

Potential questions:

```text
Is current user correct?
Does sudo policy authorize operation?
Is exact command/path allowed?
Is target user correct?
Is policy syntax valid?
Is authentication requirement satisfied?
```

Và nhớ:

```text
Denied by policy
```

có thể là **correct security behavior**, không phải system malfunction.

---

# PHẦN XXXIII — DAY 1 THEORY: 15 CORE RULES CẦN NHỚ

1. **Biết command không đồng nghĩa được phép chạy command.**

2. **Technical privilege không đồng nghĩa operational authorization.**

3. **Verify target trước khi modify.**

4. **Capture current state trước khi change.**

5. **Ưu tiên read-only investigation trước state-changing actions.**

6. **Chỉ thay đổi minimum cần thiết.**

7. **Least privilege áp dụng cho user, group, permissions, `sudo` và SSH.**

8. **Authentication và authorization là hai thứ khác nhau.**

9. **Successful command không tự động có nghĩa successful operation.**

10. **Permission denied không tự động có nghĩa phải `chmod`.**

11. **Private SSH key phải được bảo vệ và không đưa vào evidence.**

12. **Host-key warning cần được investigate, không bypass tùy tiện.**

13. **Security control phải test cả allowed và denied behavior.**

14. **Rollback phải được nghĩ trước change, không phải sau incident.**

15. **Một escalation tốt phải chuyển cả context và evidence, không chỉ chuyển symptom.**

---

# PHẦN XXXIV — KNOWLEDGE CHECK CHO MODULE LÝ THUYẾT

Tôi chưa đưa đáp án để bạn có thể tự kiểm tra sau.

### Foundation

**Câu 1.** Tại sao một administrator có quyền `sudo` vẫn có thể không được phép thực hiện một change?

**Câu 2.** Sự khác nhau giữa technical capability và operational authorization là gì?

**Câu 3.** Vì sao phải phân biệt production và non-production?

**Câu 4.** SOP và runbook khác nhau thế nào?

**Câu 5.** Một escalation tốt phải chứa những thông tin nào?

### Linux architecture

**Câu 6.** Linux kernel khác Linux distribution như thế nào?

**Câu 7.** Shell và terminal khác nhau ra sao?

**Câu 8.** `/` khác `/root` như thế nào?

**Câu 9.** `uname -r` và `/etc/os-release` cung cấp hai loại thông tin gì?

### CLI

**Câu 10.** Absolute path và relative path khác nhau như thế nào?

**Câu 11.** Vì sao `pwd` có giá trị trước một destructive operation?

**Câu 12.** `>` và `>>` khác nhau thế nào?

**Câu 13.** Pipe `|` làm gì?

**Câu 14.** `stdin`, `stdout`, `stderr` là gì?

### Permissions

**Câu 15.** `640` tương ứng permission string nào?

**Câu 16.** `r`, `w`, `x` trên directory khác regular file như thế nào?

**Câu 17.** Tại sao `chmod 777` không phải giải pháp tốt cho permission issue?

**Câu 18.** `chown`, `chgrp`, `chmod` khác nhau thế nào?

**Câu 19.** `umask` có vai trò gì?

### Security

**Câu 20.** Authentication và authorization khác nhau thế nào?

**Câu 21.** Tại sao SSH private key không được đặt vào evidence package?

**Câu 22.** Tại sao phải test cả allowed access và denied access?

**Câu 23.** `sudo -l` giúp xác minh điều gì?

**Câu 24.** Tại sao SSH host-key mismatch không nên bị bỏ qua tự động?

### Scenario

**Câu 25.** User báo `Permission denied` khi đọc file. Bạn sẽ điều tra theo thứ tự nào?

**Câu 26.** SSH báo `Permission denied (publickey)`. Bạn ưu tiên điều tra networking hay authentication? Tại sao?

**Câu 27.** Bạn phát hiện file application đang có `777`. Application vẫn hoạt động. Đây có phải trạng thái tốt không? Vì sao?

**Câu 28.** Ticket chỉ approve quyền đọc report nhưng user hiện có unrestricted `sudo`. Bạn đánh giá vấn đề gì?

**Câu 29.** Bạn vừa chạy `chmod`, command không báo lỗi. Bạn đã có thể close ticket chưa?

**Câu 30.** Bạn phát hiện system state khác baseline nhưng không rõ ai thay đổi. Bạn nên tự sửa ngay về baseline hay trước tiên làm gì?

---

# PHẦN XXXV — INTERVIEW / OPERATIONS SCENARIOS

## Scenario 1 — Permission issue

User báo:

```text
"I cannot read /srv/reports/report.csv"
```

Một câu trả lời operations tốt sẽ không bắt đầu bằng:

```bash
chmod 777 /srv/reports/report.csv
```

Mà bắt đầu bằng:

```text
Identity?
Group memberships?
Parent directory permissions?
File ownership?
File mode?
Expected access?
Approved access?
Additional controls?
```

---

## Scenario 2 — SSH failure

User:

```text
ssh ops1@server
Permission denied (publickey)
```

Bạn phải hiểu đây khác:

```text
Connection timed out
```

vì failure xảy ra ở layer khác.

---

## Scenario 3 — Emergency request

Application owner gọi trực tiếp:

> "Cho tôi root access nhanh để fix production."

Bạn technically có thể cấp.

Nhưng operationally phải kiểm tra:

```text
authorization
emergency access process
ticket
approval
scope
expiry
audit
```

Không dùng urgency để bypass controls tùy ý.

---

## Scenario 4 — Unknown SSH key

Bạn thấy một key lạ trong:

```text
authorized_keys
```

Không nên:

```text
delete immediately
```

nếu chưa biết impact, trừ khi approved incident procedure yêu cầu.

Bạn cần:

```text
capture evidence
identify owner
check expected baseline
review authorized change
escalate security concern
```

---

## Scenario 5 — Recursive ownership

Someone proposes:

```bash
chown -R appuser:appgroup /opt/app
```

Bạn phải hỏi:

```text
How many objects?
Expected ownership?
Symlinks?
Application dependencies?
Current baseline?
Rollback?
Maintenance impact?
Authorization?
```

---

# PHẦN XXXVI — CHEAT SHEET DAY 1 THEORY

## Infrastructure Operations

```text
Observe before modify
Verify target
Know environment
Use least privilege
Follow SOP/runbook
Have authorization
Have rollback
Verify after change
Collect evidence
Escalate when outside scope
```

## Linux identity

```bash
whoami
id
groups
getent passwd <user>
getent group <group>
```

## System identity

```bash
hostname
hostnamectl
cat /etc/os-release
uname -r
```

## Navigation

```bash
pwd
ls
ls -la
cd
```

Key path concepts:

```text
/    root
.    current directory
..   parent
~    home
```

## Files

```bash
touch
mkdir
cp
mv
rm
rmdir
stat
```

## Text/search foundation

```bash
cat
grep
find
```

## Streams

```text
stdin  = 0
stdout = 1
stderr = 2
```

```bash
|
>
>>
<
2>
```

## Permissions

```text
r = 4
w = 2
x = 1
```

```bash
chmod
chown
chgrp
umask
```

Example:

```text
640 = rw-r-----
750 = rwxr-x---
```

## Privilege

```bash
sudo -l
sudo <authorized-command>
```

Principle:

```text
sudo capability ≠ change authorization
```

## SSH

Default association:

```text
TCP/22
```

Basic client:

```bash
ssh user@host
```

Key utility:

```bash
ssh-keygen
```

Typical authorized-key location:

```text
~/.ssh/authorized_keys
```

Security rule:

```text
Private key → protect
Public key  → deploy where authorized
```

---

# PHẦN XXXVII — DAY 1 LÝ THUYẾT COMPLETION CHECKLIST

Sau khi học kỹ module này, bạn nên tự tin tick được toàn bộ:

### Infrastructure Operations

- [ ] Tôi giải thích được trách nhiệm cơ bản của Infrastructure Operations.
- [ ] Tôi hiểu L1/L2 boundary phụ thuộc organization.
- [ ] Tôi phân biệt technical capability và operational authorization.
- [ ] Tôi hiểu operational safety.
- [ ] Tôi phân biệt production và non-production.
- [ ] Tôi hiểu change authorization.
- [ ] Tôi hiểu least privilege.
- [ ] Tôi phân biệt SOP và runbook.
- [ ] Tôi hiểu escalation tốt cần context và evidence.
- [ ] Tôi hiểu access-management lifecycle.
- [ ] Tôi hiểu authentication khác authorization.
- [ ] Tôi hiểu configuration baseline.
- [ ] Tôi hiểu asset inventory.
- [ ] Tôi hiểu audit evidence.
- [ ] Tôi biết những thông tin không được đưa vào evidence.
- [ ] Tôi hiểu lab readiness.

### Linux foundation

- [ ] Tôi giải thích được Linux architecture ở high level.
- [ ] Tôi giải thích được kernel.
- [ ] Tôi phân biệt kernel và Linux distribution.
- [ ] Tôi phân biệt shell và terminal.
- [ ] Tôi hiểu RHEL nằm ở đâu trong Linux ecosystem.
- [ ] Tôi biết các directory cơ bản `/etc`, `/home`, `/root`, `/var`, `/tmp`, `/usr`, `/opt`, `/boot`, `/dev`, `/proc`, `/sys`.
- [ ] Tôi phân biệt `/` và `/root`.
- [ ] Tôi hiểu current working directory.
- [ ] Tôi phân biệt absolute và relative path.
- [ ] Tôi hiểu `.`, `..`, `~`.
- [ ] Tôi hiểu basic file/directory operations.
- [ ] Tôi hiểu command/options/arguments.
- [ ] Tôi hiểu quoting cơ bản.
- [ ] Tôi hiểu `stdin`, `stdout`, `stderr`.
- [ ] Tôi hiểu pipe.
- [ ] Tôi hiểu redirection.
- [ ] Tôi hiểu risk của `>`.
- [ ] Tôi hiểu command history và security implications.
- [ ] Tôi biết sử dụng `man`, `--help`, `help`.
- [ ] Tôi hiểu safe command execution.

### Identity, permissions và secure access

- [ ] Tôi hiểu Linux user/UID.
- [ ] Tôi hiểu primary/supplementary groups.
- [ ] Tôi hiểu user owner và group owner.
- [ ] Tôi đọc được permission string.
- [ ] Tôi chuyển đổi cơ bản giữa symbolic và numeric permissions.
- [ ] Tôi hiểu `r/w/x` trên file và directory.
- [ ] Tôi hiểu chức năng của `chmod`.
- [ ] Tôi hiểu chức năng của `chown`.
- [ ] Tôi hiểu chức năng của `chgrp`.
- [ ] Tôi hiểu `umask`.
- [ ] Tôi hiểu `sudo` là controlled privilege delegation.
- [ ] Tôi hiểu least-privilege `sudo`.
- [ ] Tôi hiểu SSH client/server.
- [ ] Tôi hiểu SSH host identity.
- [ ] Tôi hiểu SSH public-key authentication.
- [ ] Tôi phân biệt private và public key.
- [ ] Tôi hiểu SSH key lifecycle.
- [ ] Tôi hiểu secure administrative access.
- [ ] Tôi hiểu positive và negative access verification.

---

# Kết luận Module Lý thuyết Day 1

Nếu phải cô đọng toàn bộ Module Lý thuyết Day 1 thành một operational model duy nhất, hãy nhớ chuỗi này:

```text
                 GOVERNANCE
                     │
        ┌────────────┴────────────┐
        │                         │
 Authorization              Support Scope
        │                         │
        └────────────┬────────────┘
                     ▼
              IDENTIFY TARGET
                     │
                     ▼
               OBSERVE STATE
                     │
                     ▼
              UNDERSTAND ACCESS
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        User       Group    Permissions
          │          │          │
          └──────────┴──────────┘
                     │
                     ▼
             AUTHENTICATION
                  SSH
                     │
                     ▼
              AUTHORIZATION
           permissions + sudo
                     │
                     ▼
           MINIMUM SAFE CHANGE
                     │
                     ▼
                VERIFY
            ┌────────┴─────────┐
            ▼                  ▼
         Allowed             Denied
         succeeds            fails
            │                  │
            └────────┬─────────┘
                     ▼
                 EVIDENCE
                     │
                     ▼
          CLOSE / ROLLBACK / ESCALATE
```

Đó chính xác là nền tảng mà syllabus Day 1 muốn xây dựng trước khi Day 2 chuyển sang administration sâu hơn như `dnf/yum/rpm`, processes, `systemd`, SELinux, `firewalld`, networking, storage, logs, Bash, `cron` và backup/restore. Infrastructure_Operations_14_Da…

Quan trọng nhất: **Day 1 không phải bài học “Linux commands cơ bản” đơn thuần.** Nó là bài học về cách một Infrastructure Operations engineer sử dụng Linux **đúng system, đúng identity, đúng quyền, đúng authorization, với minimum necessary change, có verification, evidence, rollback và escalation**. Đây cũng phù hợp với định hướng chung của syllabus là xây `operational readiness` thông qua hands-on, verification evidence và controlled troubleshooting chứ không tuyên bố advanced product mastery chỉ sau khóa học ngắn. Infrastructure_Operations_14_Da…
