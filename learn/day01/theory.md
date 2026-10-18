# Day 1 — Module Lý thuyết: Infrastructure Operations & Linux Administration Foundations

Theo syllabus, phần `Concept/Lecture` của Day 1 gồm hai khối kiến thức lớn. Khối thứ nhất xây dựng **enterprise Infrastructure Operations mindset**: roles and responsibilities, L1/L2 boundaries, operational safety, production/non-production separation, change authorization, least privilege, SOP, runbook, incident escalation, access management, configuration baseline, asset inventory, audit evidence và lab readiness. Khối thứ hai xây dựng **Linux administration foundation**: Linux architecture, kernel, shell, distributions/RHEL, filesystem hierarchy, CLI, paths, file operations, command syntax, pipes/redirection/history/help, safe execution, users/groups, ownership/permissions, `chmod/chown/chgrp`, `umask`, `sudo`, SSH authentication, SSH keys và secure administrative access. Infrastructure_Operations_14_Da…

Phần dưới đây giữ nguyên phạm vi đó. Những phần tôi mở rộng để giúp bạn hiểu cơ chế sâu hơn sẽ được đánh dấu **Kiến thức bổ sung**; chúng hỗ trợ syllabus chứ không thay đổi phạm vi Day 1. Day 2 mới đi sâu vào `dnf/yum/rpm`, processes, `systemd`, SELinux, `firewalld`, networking, storage, logging, Bash scripting, backup và operational troubleshooting chuyên sâu. Infrastructure_Operations_14_Da…

---

# PHẦN I — INFRASTRUCTURE OPERATIONS FOUNDATION

## 1. Infrastructure administration roles and responsibilities

### 1.1. What — Infrastructure Administration là gì?

`Infrastructure Administration` là hoạt động vận hành, duy trì và kiểm soát các thành phần hạ tầng cung cấp nền tảng cho applications và business services.

Một enterprise infrastructure có thể bao gồm:

```text
Physical servers
Virtual machines
Operating systems
Storage
Network
Firewall
Load balancer
Cloud resources
Backup platforms
Monitoring systems
Logging platforms
Identity systems
ITSM systems
Automation
```

Infrastructure administrator không chỉ là người "biết cấu hình server". Công việc thường bao gồm nhiều loại hoạt động:

```text
Provision
Configure
Verify
Monitor
Maintain
Troubleshoot
Recover
Secure
Document
Escalate
Audit
Handover
```

Một kỹ sư giỏi không được đánh giá chỉ dựa trên số command nhớ được, mà còn ở khả năng thực hiện những command đó **đúng system, đúng thời điểm, đúng authorization, đúng scope, có verification và có evidence**.

---

## 1.2. Why — Tại sao Infrastructure Operations cần discipline?

Hãy tưởng tượng bạn nhận ticket:

```text
"Application không truy cập được thư mục /data/app."
```

Một người chỉ tập trung vào kỹ thuật có thể ngay lập tức chạy:

```bash
chmod -R 777 /data/app
```

Có thể application chạy lại.

Nhưng về operations, đây có thể là một thay đổi cực kỳ tệ.

Bạn chưa xác định:

- lỗi thực sự có phải permission hay không;
- user nào cần access;
- group nào được phép access;
- `777` có vi phạm security policy không;
- có sensitive data không;
- recursive change tác động bao nhiêu objects;
- cấu hình trước đó là gì;
- cách rollback;
- change có được authorization không.

Infrastructure Operations vì thế không chỉ hỏi:

> "Command nào sửa được?"

mà còn hỏi:

> "Root cause là gì, tôi có quyền sửa không, cách sửa ít rủi ro nhất là gì, làm sao chứng minh kết quả đúng và làm sao quay lại trạng thái cũ nếu thất bại?"

---

# 2. Operational workflow nền tảng

Đây là tư duy cần giữ xuyên suốt 14 Day.

```text
Request / Alert / Incident
        ↓
Confirm identity & authorization
        ↓
Confirm target & environment
        ↓
Understand current state
        ↓
Collect initial evidence
        ↓
Determine scope / blast radius
        ↓
Choose lowest-risk action
        ↓
Perform approved action
        ↓
Verify technical result
        ↓
Verify business/service result
        ↓
Collect post-change evidence
        ↓
Update ticket / documentation
        ↓
Close or Escalate
```

Không phải mọi situation đều có từng bước giống hệt nhau, nhưng mental model này cực kỳ quan trọng.

### Một distinction phải nhớ

`Technical success` và `operational success` không hoàn toàn giống nhau.

Ví dụ:

```text
Command exit successfully
```

không nhất thiết đồng nghĩa:

```text
Service problem has been resolved.
```

Một permission có thể được thay đổi thành công nhưng application vẫn không hoạt động.

Một SSH login có thể thành công nhưng lại dùng sai account.

Một configuration change có thể đúng nhưng được thực hiện trên sai server.

Vì vậy phải verify **kết quả thực tế**, không chỉ output của command.

---

# 3. L1/L2 support boundaries

## 3.1. What

`Support boundary` xác định những operation mà một support level được:

- inspect;
- diagnose;
- execute;
- modify;
- approve;
- escalate.

Không tồn tại một chuẩn toàn cầu nói rằng "L1 luôn làm X, L2 luôn làm Y". Boundary phụ thuộc organization, contract, access model và runbook.

Những mô tả dưới đây là operational model phổ biến, không phải luật cố định.

---

## 3.2. L1 thường làm gì?

L1 thường thiên về:

```text
Monitoring
Alert triage
Basic health checking
Standard diagnostics
Known-error handling
Runbook execution
Evidence collection
Ticket update
Initial incident classification
Escalation
```

Ví dụ một Linux server không SSH được.

L1 có thể được phép kiểm tra:

```bash
hostname
whoami
id
```

trên systems vẫn truy cập được, hoặc kiểm tra reachability từ workstation nếu runbook cho phép.

L1 có thể thu thập:

```text
Timestamp
Target hostname/IP
Username
Exact error message
Recent changes
Affected users
Scope
```

Nhưng L1 có thể không được phép sửa:

```text
/etc/ssh/sshd_config
firewall rules
PAM configuration
centralized identity settings
```

---

# 4. L2 support

L2 thường có scope kỹ thuật rộng hơn.

Ví dụ:

```text
Configuration diagnosis
Service dependency analysis
Permission troubleshooting
Advanced OS investigation
Approved configuration changes
Recovery actions
Root-cause isolation
```

Nhưng L2 cũng không có nghĩa là:

> "Có quyền root nên muốn làm gì cũng được."

`root privilege` là technical capability.

`authorization` là organizational permission.

Hai thứ này hoàn toàn khác nhau.

---

# 5. Escalation boundary

Một operator phải biết lúc nào **không tiếp tục**.

Các dấu hiệu thường phải escalate gồm:

- action vượt privilege/scope của bạn;
- cần production configuration change chưa được approve;
- root cause chưa rõ nhưng proposed change có blast radius lớn;
- security incident suspected;
- sensitive credentials có nguy cơ compromise;
- filesystem hoặc permissions liên quan nhiều applications;
- business-critical system;
- rollback không rõ;
- evidence không đủ để tiếp tục an toàn;
- runbook yêu cầu escalation;
- có dấu hiệu issue nằm ngoài technology domain của team.

Một escalation tốt không phải:

```text
"Server lỗi. Nhờ check."
```

Mà nên có:

```text
Affected asset:
Environment:
Start time:
Symptoms:
Business impact:
Checks completed:
Results:
Changes attempted:
Evidence:
Suspected cause:
Current state:
Reason for escalation:
```

---

# 6. Operational Safety

Đây là một trong những phần quan trọng nhất của Day 1.

## 6.1. Nguyên tắc inspect before modify

Nếu chưa hiểu trạng thái hiện tại, ưu tiên command read-only.

Ví dụ trước khi đổi ownership:

```bash
ls -ld /path
```

trước khi:

```bash
chown ...
```

Trước khi đổi permissions:

```bash
ls -l file
```

trước:

```bash
chmod ...
```

Trước khi cấp `sudo`:

```bash
sudo -l -U username
```

nếu bạn có quyền kiểm tra.

Infrastructure Operations nên ưu tiên:

```text
Observe
→ Understand
→ Change
```

thay vì:

```text
Change
→ See what happens
```

---

# 7. Confirm target

Một trong những lỗi operational nguy hiểm nhất:

> đúng command nhưng sai target.

Trước thao tác quan trọng, cần biết mình đang ở đâu.

Các command cơ bản:

```bash
hostname
whoami
id
pwd
```

Ý nghĩa:

```text
hostname  → system nào?
whoami    → account nào?
id        → UID/GID/groups nào?
pwd       → directory nào?
```

Nếu terminal prompt là:

```text
[user@server ~]$
```

không nên coi prompt là evidence tuyệt đối, vì prompt có thể được customize.

`hostname` hoặc các system query thích hợp đáng tin cậy hơn.

---

# 8. Blast radius

`Blast radius` là phạm vi tác động nếu operation sai.

Ví dụ:

```bash
chmod 640 onefile.txt
```

blast radius nhỏ.

Trong khi recursive operation trên một directory tree lớn có thể tác động:

```text
thousands of files
multiple users
multiple services
backup agents
applications
```

Do đó trước action cần biết:

```text
What object?
How many objects?
Who uses them?
Can the change be reversed?
```

---

# 9. Reversibility

Một operation an toàn thường nên có khả năng rollback.

Ví dụ trước khi thay permission:

```bash
stat file.txt
```

hoặc:

```bash
ls -l file.txt
```

để biết state trước đó.

Nếu chỉ biết:

```text
"Permission mới = 640"
```

nhưng không biết permission cũ là gì, bạn chưa có rollback plan tốt.

---

# 10. Read-only và configuration-changing operations

Đây là distinction rất quan trọng.

Ví dụ read-only:

```bash
pwd
ls -l
id
cat /etc/os-release
```

Những command này thường dùng để inspect state.

Ví dụ configuration-changing:

```bash
chmod
chown
useradd
usermod
```

Chúng thay đổi state.

Không nên suy luận rằng tất cả read-only command đều vô hại tuyệt đối; một số command có thể tiêu thụ nhiều resources hoặc làm lộ dữ liệu. Nhưng về operational classification, distinction này vẫn rất hữu ích.

---

# 11. Production và non-production environment separation

## 11.1. Các environment thường gặp

Có thể tồn tại:

```text
Development
Test
QA
UAT
Staging
Production
DR
Lab/Sandbox
```

Mỗi organization có naming convention riêng.

Không được giả định rằng:

```text
test01
```

chắc chắn là non-production nếu chưa kiểm tra inventory/documentation.

---

## 11.2. Production khác gì?

Production thường:

- phục vụ business thực;
- sử dụng live data;
- có SLA;
- có monitoring;
- yêu cầu formal change;
- có business owner;
- có security restrictions cao hơn;
- có backup/recovery obligations;
- có incident escalation requirements.

Một thao tác vô hại trong lab có thể không được chấp nhận trên production.

---

## 11.3. Vì sao phải tách credentials?

Lý tưởng:

```text
Lab credential ≠ Production credential
```

và:

```text
Normal user ≠ Privileged identity
```

Điều này giúp hạn chế accidental access và blast radius.

---

# 12. Change Authorization

## 12.1. Change là gì?

Một `change` là modification có kiểm soát đối với system, configuration hoặc service.

Ví dụ:

```text
Change file permission
Create account
Add sudo privilege
Change SSH configuration
Install package
Restart production service
Modify firewall policy
```

Không phải tất cả organization phân loại change giống nhau.

---

# 13. Change authorization không giống technical access

Giả sử:

```text
sudo -l
```

cho thấy bạn có quyền chạy một administrative command.

Điều đó chứng minh system technically cho phép.

Nó **không chứng minh** ticket/change process cho phép bạn chạy nó ngay lúc đó.

Có thể bạn có persistent privileged access để phục vụ support, nhưng vẫn phải có:

```text
Incident ticket
Approved request
Change record
Maintenance window
```

tùy operation.

---

# 14. Một change plan tốt

Một implementation plan nên trả lời được:

```text
What is being changed?
Why?
Where?
Who approved?
When?
What is the current state?
What commands/actions will be used?
How will success be verified?
What might fail?
How will rollback work?
What evidence will be captured?
```

---

# 15. Least Privilege

## 15.1. Definition

`Least privilege` nghĩa là một identity, process hoặc service chỉ được cấp những permissions cần thiết cho nhiệm vụ được phép và chỉ trong phạm vi cần thiết.

Concept này áp dụng cho:

```text
Users
Groups
sudo
SSH
Filesystem permissions
Service accounts
AWS IAM later in the course
Privileged access systems
```

---

# 16. Vì sao least privilege quan trọng?

Giả sử một account chỉ cần đọc reports.

Nếu account có quyền:

```text
read + write + delete
```

thì quyền vượt quá business need.

Nếu account bị compromise, attacker cũng kế thừa các quyền dư thừa đó.

Security risk có thể hình dung:

```text
Risk ≈ Compromise probability × Privilege × Reach
```

Không phải công thức định lượng chính thức, nhưng rất hữu ích để hiểu tư duy.

---

# 17. Least privilege không đồng nghĩa "càng ít quyền càng tốt"

Nếu giảm quyền tới mức user không hoàn thành nhiệm vụ hợp lệ, access model cũng sai.

Mục tiêu là:

```text
Minimum sufficient privilege
```

chứ không phải:

```text
Minimum possible privilege
```

---

# 18. Separation of duties — Kiến thức bổ sung

Một principle liên quan là `separation of duties`.

Ví dụ người request privileged access không nhất thiết cũng là người tự approve access cho mình.

Mô hình:

```text
Requester
    ↓
Approver
    ↓
Implementer
    ↓
Reviewer/Auditor
```

Trong team nhỏ có thể roles bị overlap, nhưng enterprise controls thường cố giảm self-approval.

---

# 19. SOP — Standard Operating Procedure

## 19.1. SOP là gì?

`SOP` là tài liệu chuẩn hóa một operational process.

Ví dụ:

```text
SOP — Linux User Access Provisioning
```

có thể định nghĩa:

```text
Required ticket
Required approval
Naming convention
Group mapping
Access expiration
Verification requirements
Evidence requirements
Revocation procedure
```

---

## 19.2. Tại sao SOP quan trọng?

Nếu mỗi administrator thực hiện theo kinh nghiệm riêng:

```text
Admin A → method A
Admin B → method B
Admin C → method C
```

thì operations khó audit, khó reproduce và dễ tạo inconsistency.

SOP hướng team về:

```text
Standard
Repeatable
Auditable
Controlled
```

---

# 20. Runbook

`Runbook` thường thiên về từng bước execution hoặc troubleshooting cụ thể.

Ví dụ:

```text
Runbook: SSH Authentication Failure
```

có thể có flow:

```text
Confirm target
   ↓
Confirm user
   ↓
Capture exact SSH error
   ↓
Confirm approved key
   ↓
Inspect ownership
   ↓
Inspect permissions
   ↓
Inspect authorized_keys
   ↓
Escalate if server-side policy change is required
```

SOP trả lời nhiều về **process**.

Runbook thường trả lời nhiều về **how to execute/diagnose a specific operational procedure**.

Trong thực tế tên gọi có thể overlap.

---

# 21. Runbook không thay thế reasoning

Không nên chạy từng command trong runbook một cách máy móc.

Trước mỗi step vẫn phải hiểu:

```text
What does it do?
Is this the correct target?
Is this still applicable?
Is it safe?
```

Một runbook cũ có thể không còn phù hợp sau architecture change.

---

# 22. Incident Escalation

## 22.1. Incident là gì?

Trong IT operations, `incident` thường là unplanned interruption hoặc degradation đối với service.

Ví dụ:

```text
SSH administration unavailable
Application cannot read required file
User cannot authenticate
Service unavailable
```

Day 11 sẽ đi sâu vào ITSM/ServiceNow; Day 1 chỉ cần foundation.

---

# 23. Symptom và root cause

Phân biệt:

```text
Symptom:
User cannot read file.

Possible root causes:
Wrong ownership
Wrong group
Missing execute permission on parent directory
Incorrect ACL
Wrong filesystem mount
Application identity changed
```

Thấy symptom không có nghĩa đã biết root cause.

---

# 24. Evidence before intervention

Khi troubleshooting, initial evidence rất quan trọng.

Nếu bạn thay config quá sớm, bạn có thể làm mất trạng thái giúp tìm root cause.

Ví dụ trước sửa:

```bash
id user1
ls -ld /path
ls -l /path/file
```

Output này có thể quan trọng hơn screenshot sau khi vấn đề đã hết.

---

# 25. Access Management

## 25.1. Authentication và Authorization

Hai concept phải phân biệt rõ.

`Authentication`:

> Bạn là ai?

Ví dụ:

```text
Password
SSH private key
MFA
Certificate
```

`Authorization`:

> Bạn được phép làm gì?

Ví dụ user login được nhưng:

```bash
cat restricted.txt
```

bị:

```text
Permission denied
```

Authentication thành công.

Authorization thất bại.

Đây không phải cùng một problem.

---

# 26. Accounting/Auditing — Kiến thức bổ sung

Trong security thường gặp bộ ba:

```text
Authentication
Authorization
Accounting
```

`Accounting` hoặc auditing ghi nhận:

```text
Who
Did what
When
Where
Result
```

Đây là lý do named accounts tốt hơn shared accounts.

Nếu năm người đều dùng:

```text
root
```

audit attribution kém hơn rất nhiều so với:

```text
alice → sudo
bob   → sudo
carol → sudo
```

---

# 27. Access lifecycle

Một access lifecycle tốt thường gồm:

```text
Request
   ↓
Validate business need
   ↓
Approval
   ↓
Provision
   ↓
Verify
   ↓
Review
   ↓
Modify if role changes
   ↓
Revoke when no longer required
```

Sai lầm lớn trong access management không chỉ là cấp sai quyền.

Còn có:

```text
Stale access
Orphaned accounts
Unexpired temporary access
Excessive privilege
Shared credentials
```

Các vấn đề này sẽ quay lại ở Day 11–13.

---

# 28. Configuration Baseline

## 28.1. Baseline là gì?

`Configuration baseline` là reference state đã được xác định để so sánh với current state.

Ví dụ Linux baseline đơn giản:

```text
Hostname
OS release
Kernel
Expected accounts
Expected groups
Expected SSH settings
Expected sudo rules
Expected permissions
```

Baseline không nhất thiết là một file duy nhất. Nó có thể tồn tại trong:

```text
CMDB
Configuration management tool
Documentation
Source control
Build standard
Hardening standard
```

---

# 29. Configuration drift

Khi current state khác expected baseline:

```text
Baseline
   ↓
approved changes
   ↓
Expected configuration
```

nhưng system thực tế trở thành:

```text
Unexpected configuration
```

thì có thể gọi là `configuration drift`.

Không phải mọi drift đều xấu; có thể có approved exceptions.

Điều quan trọng là drift phải được hiểu và kiểm soát.

---

# 30. Asset Inventory

## 30.1. Asset là gì?

Trong Infrastructure Operations, một `asset` có thể là:

```text
Physical server
Virtual machine
Network device
Storage appliance
Cloud resource
```

Day 1 chủ yếu quan tâm Linux/RHEL lab asset.

---

# 31. Minimum useful inventory

Một Linux inventory cơ bản có thể có:

```text
Hostname
Asset identifier
Environment
OS distribution
OS release
Kernel release
CPU
Memory
Storage
IP addresses
Owner/team
Business/service role
```

Không phải mọi field đều thuộc Day 1 lab, nhưng bạn cần hiểu inventory phải đủ để **identify đúng system**.

---

# 32. Asset inventory và configuration baseline khác nhau

Inventory trả lời:

> "Chúng ta có system nào?"

Baseline trả lời:

> "System đó nên được cấu hình như thế nào?"

Ví dụ:

```text
Inventory:
server01 = RHEL server

Baseline:
server01 SSH admin access must use approved identities,
expected group membership and approved configuration.
```

---

# 33. Audit Evidence

## 33.1. Evidence là gì?

`Audit evidence` là artifact chứng minh một operation hoặc state.

Có thể gồm:

```text
Command output
Logs
Screenshots
Configuration extract
Ticket reference
Approval record
Timestamp
Hash
Report
```

Day 1 yêu cầu hình thành discipline này ngay từ đầu.

---

# 34. Evidence phải trả lời được gì?

Một evidence tốt nên giúp reviewer biết:

```text
What system?
What time?
Which account?
What was checked?
What result?
What operation?
Was it successful?
```

Ví dụ output:

```bash
ls -l file.txt
```

mà không biết hostname có thể rất yếu nếu bạn quản trị nhiều systems.

Do đó context cực kỳ quan trọng.

---

# 35. Evidence quality

Evidence nên:

```text
Relevant
Complete enough
Readable
Timestamped
Attributable
Protected
Reproducible where possible
```

Không nên chứa:

```text
Passwords
Private keys
API tokens
Secrets
Sensitive credential material
```

---

# 36. Screenshots không phải evidence duy nhất

Screenshot hữu ích cho GUI hoặc visual state.

Nhưng CLI thường tốt hơn khi lưu:

```text
command + output + timestamp
```

vì text:

- searchable;
- diff-able;
- easier to review;
- easier to archive.

Trong enterprise operations, đôi khi cần cả hai.

---

# 37. Lab Readiness

Trước hands-on, phải chắc chắn environment phù hợp.

Về theory, bạn phải hiểu sáu loại readiness.

### Access readiness

Bạn có:

```text
Approved account
Correct privilege
Correct authentication method
```

### Platform readiness

Linux/RHEL VM hoạt động và bạn biết:

```text
Hostname
IP
OS
Environment
```

### Workstation readiness

Có:

```text
Terminal
SSH client
Evidence storage
Text editor
```

### Authorization readiness

Lab target là authorized system.

### Safety readiness

Biết:

```text
What can be changed
What cannot
How to rollback
```

### Evidence readiness

Có location lưu artifacts mà không chứa secrets.

---

# PHẦN II — LINUX ARCHITECTURE FOUNDATION

# 38. Linux là gì?

Cần tránh một nhầm lẫn phổ biến.

Về kỹ thuật, `Linux` chính xác nhất là kernel.

Nhưng trong giao tiếp thực tế, người ta thường gọi toàn bộ operating system distribution là Linux.

Ví dụ:

```text
Linux kernel
        +
GNU utilities
        +
Libraries
        +
Package management
        +
Init/service system
        +
Distribution tooling
        =
Linux distribution
```

RHEL là một Linux distribution.

---

# 39. Linux architecture

Có thể hình dung đơn giản:

```text
+--------------------------------------+
|              Applications            |
+--------------------------------------+
|        Shell / User Utilities         |
+--------------------------------------+
|        Libraries / User Space         |
+--------------------------------------+
|               Kernel                 |
+--------------------------------------+
|              Hardware                |
+--------------------------------------+
```

Mô hình này được đơn giản hóa nhưng phù hợp Day 1.

---

# 40. Kernel

`kernel` chạy với privilege cao nhất trong OS và quản lý các tài nguyên cốt lõi.

Các trách nhiệm quan trọng gồm:

```text
CPU scheduling
Memory management
Process management
Device interaction
Filesystem access
Networking
Security enforcement mechanisms
```

Applications thông thường không trực tiếp điều khiển hardware.

Chúng yêu cầu kernel thực hiện operations.

---

# 41. System calls — Kiến thức bổ sung

User-space program thường giao tiếp với kernel qua `system calls`.

Conceptual flow:

```text
Application
    ↓
Library/API
    ↓
System call
    ↓
Kernel
    ↓
Hardware / filesystem / network
```

Bạn chưa cần lập trình system call ở Day 1.

Mục đích là hiểu:

> shell command cuối cùng vẫn dựa vào OS/kernel để thực hiện công việc.

---

# 42. User space và kernel space

`Kernel space` dành cho kernel và privileged kernel operations.

`User space` là nơi phần lớn applications và user processes hoạt động.

Một user application crash thường không đồng nghĩa kernel crash.

Đây là isolation cơ bản của modern operating systems.

---

# 43. Kiểm tra kernel

Command:

```bash
uname
```

thường trả về kernel/system name.

```bash
uname -r
```

cho kernel release.

Ví dụ dạng:

```text
5.x.x-...
```

hoặc:

```text
6.x.x-...
```

tùy distribution/release.

Không nên suy luận OS distribution chỉ bằng kernel version.

Để biết distribution, cách thông dụng hơn là:

```bash
cat /etc/os-release
```

---

# 44. Shell

## 44.1. Shell là gì?

`Shell` là command interpreter.

Bạn nhập:

```bash
ls -l /etc
```

shell parse command rồi thực thi phù hợp.

Shell cũng hỗ trợ:

```text
Variables
Pipes
Redirection
Command substitution
Scripting
Control operators
```

Day 1 chỉ dùng phần foundation.

Day 2 mới đi sâu Bash control flow và scripting.

---

# 45. Terminal, shell và CLI khác gì?

Đây là ba concept dễ lẫn.

`Terminal` hoặc terminal emulator là interface hiển thị và nhận input.

Ví dụ terminal application mở một pseudo-terminal.

`Shell` là program chạy bên trong terminal, ví dụ:

```text
bash
zsh
```

`CLI` là phương thức interaction bằng textual command line.

Có thể hình dung:

```text
Terminal window
      ↓
Shell: bash
      ↓
Command
      ↓
Operating system
```

---

# 46. Xác định shell

Có thể thấy login shell từ account database bằng cách phù hợp, ví dụ:

```bash
getent passwd "$USER"
```

Hoặc xem shell process/context tùy trường hợp.

Biến:

```bash
$SHELL
```

thường biểu diễn configured login shell, nhưng không phải lúc nào cũng chứng minh shell process hiện tại.

Đây là distinction nhỏ nhưng hữu ích.

---

# 47. Linux distributions

Distribution kết hợp Linux kernel với packages, libraries, installers, repositories và management tools.

Các examples:

```text
Red Hat Enterprise Linux
Rocky Linux
AlmaLinux
Fedora
Ubuntu
Debian
SUSE Linux Enterprise Server
```

Không nên giả định commands/package management giống hoàn toàn giữa distributions.

Ví dụ package ecosystem có khác nhau.

Day 2 sẽ tập trung RHEL-style tools như:

```text
dnf
yum
rpm
```

---

# 48. Red Hat Enterprise Linux — RHEL

RHEL là enterprise Linux distribution của Red Hat.

Trong operational environment, một enterprise distribution thường quan trọng bởi:

```text
Vendor support
Lifecycle management
Security updates
Certified ecosystem
Documentation
Enterprise tooling
Predictable maintenance model
```

Bạn chưa cần nhớ version matrix ở Day 1.

Khi làm việc thực tế phải xác định chính xác release trước khi áp dụng procedure vì behavior và supported options có thể thay đổi theo version.

---

# 49. Xác định Linux distribution

File phổ biến:

```bash
cat /etc/os-release
```

Ví dụ output có thể chứa:

```text
NAME=
VERSION=
ID=
VERSION_ID=
```

Không nên parse thủ công dựa trên một line cố định nếu viết automation portable, nhưng Day 1 chỉ cần đọc và hiểu.

---

# PHẦN III — LINUX FILESYSTEM

# 50. Filesystem hierarchy

Linux không tổ chức storage bằng drive letters như:

```text
C:
D:
```

theo mô hình Windows phổ biến.

Linux xây dựng một unified filesystem tree bắt đầu từ:

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
├── usr
├── tmp
├── dev
├── proc
└── ...
```

Các disks/filesystems khác có thể được mount vào các directory trong tree này.

Mounting được học sâu hơn ở Day 2.

---

# 51. `/`

`/` là filesystem hierarchy root.

Không được nhầm:

```text
/
```

với:

```text
/root
```

`/` là root directory của entire filesystem hierarchy.

`/root` thường là home directory của root user.

---

# 52. `/home`

Thường chứa home directories của regular users.

Ví dụ:

```text
/home/alice
/home/bob
```

Nhưng enterprise environments có thể dùng home location khác hoặc centrally mounted home directories.

Không nên hard-code assumption nếu chưa verify.

---

# 53. `/root`

Thông thường là home của `root`.

Regular users thường không được truy cập nội dung của `/root`.

---

# 54. `/etc`

Chứa nhiều system-wide configuration files.

Ví dụ:

```text
/etc/passwd
/etc/group
/etc/ssh/
/etc/sudoers
```

Vì `/etc` chứa critical configuration, thay đổi ở đây cần đặc biệt cẩn trọng.

---

# 55. `/var`

`/var` chứa variable data — dữ liệu thay đổi trong quá trình vận hành.

Ví dụ:

```text
logs
spools
caches
state data
```

Một directory rất quan trọng sau này là:

```text
/var/log
```

Day 2 đi sâu hơn vào logs.

---

# 56. `/tmp`

Dùng cho temporary files.

Không nên coi `/tmp` là permanent storage.

Behavior về cleanup và retention phụ thuộc system configuration.

Security cũng quan trọng vì đây thường là shared writable location.

---

# 57. `/usr`

Chứa phần lớn user-space software, libraries và read-mostly resources.

Trên nhiều modern distributions:

```text
/bin
/sbin
/lib
```

có thể được liên kết với corresponding locations dưới `/usr`.

Do đó không nên dùng một mô hình filesystem cũ như chân lý tuyệt đối cho mọi distribution.

---

# 58. `/bin` và `/sbin`

Lịch sử:

```text
/bin   → essential user commands
/sbin  → system administration commands
```

Trên modern systems có thể là symbolic links đến:

```text
/usr/bin
/usr/sbin
```

Concept cần nhớ không phải physical separation tuyệt đối mà là command locations và compatibility.

---

# 59. `/dev`

Chứa device nodes.

Ví dụ storage devices có thể xuất hiện dưới dạng:

```text
/dev/...
```

Không thực hành destructive disk operations trong Day 1.

---

# 60. `/proc`

`/proc` là pseudo-filesystem cung cấp kernel/process information.

Nó không phải ordinary persistent disk directory.

Ví dụ:

```bash
cat /proc/cpuinfo
```

có thể cung cấp CPU information.

---

# 61. `/sys`

`/sys` cũng là virtual filesystem liên quan kernel/device model.

Day 1 chỉ cần biết vai trò.

---

# 62. `/boot`

Thường chứa boot-related files.

Không được tự ý chỉnh hoặc xóa nội dung `/boot`.

---

# 63. `/run`

Thường chứa runtime state từ lúc boot.

Ví dụ:

```text
PID-related runtime files
sockets
temporary runtime state
```

Dữ liệu thường không được coi là persistent qua reboot.

---

# 64. `/mnt`, `/media`, `/opt`

`/mnt` thường dùng cho temporary/manual mount points.

`/media` thường liên quan removable media trong một số environments.

`/opt` thường được dùng cho optional/add-on software.

Exact usage vẫn phụ thuộc distribution/vendor application.

---

# PHẦN IV — CLI NAVIGATION VÀ PATHS

# 65. Current Working Directory

Mỗi shell process có một `current working directory`.

Kiểm tra bằng:

```bash
pwd
```

Ví dụ:

```text
/home/student
```

Đây là context rất quan trọng vì relative paths được resolve từ đây.

---

# 66. `ls`

Basic:

```bash
ls
```

Detailed:

```bash
ls -l
```

Include hidden names:

```bash
ls -a
```

Combine:

```bash
ls -la
```

Cần hiểu hidden file trên Linux thường là filename bắt đầu bằng:

```text
.
```

Ví dụ:

```text
.bashrc
.ssh
```

Không có một hidden attribute tương tự theo cách hiểu cơ bản của Windows; naming convention với leading dot là quan trọng.

---

# 67. `cd`

Ví dụ:

```bash
cd /etc
```

đi tới absolute location.

```bash
cd ..
```

đi parent directory.

```bash
cd ~
```

đi home directory.

Thông thường:

```bash
cd
```

không argument cũng đi về home directory của shell/user trong Bash-compatible usage.

---

# 68. Absolute Path

Absolute path bắt đầu từ `/`.

Ví dụ:

```text
/etc/ssh/sshd_config
```

Interpretation:

```text
/
└── etc
    └── ssh
        └── sshd_config
```

Không phụ thuộc current directory.

---

# 69. Relative Path

Không bắt đầu từ `/`.

Ví dụ current directory:

```text
/home/student/lab
```

và bạn dùng:

```text
files/report.txt
```

thì path resolve thành:

```text
/home/student/lab/files/report.txt
```

---

# 70. `.` và `..`

`.`:

```text
current directory
```

`..`:

```text
parent directory
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

# 71. `~`

Shell expansion:

```text
~
```

thường trỏ home directory của current user.

Ví dụ:

```bash
cd ~/.ssh
```

---

# 72. Case sensitivity

Linux filenames thường case-sensitive.

Do đó:

```text
Report.txt
report.txt
REPORT.txt
```

có thể là ba files khác nhau.

Đây là nguồn lỗi phổ biến với users quen Windows environments.

---

# 73. Spaces trong filename

Một file:

```text
My Report.txt
```

không thể luôn xử lý như:

```bash
cat My Report.txt
```

vì shell có thể hiểu thành multiple arguments.

Phải quote hoặc escape:

```bash
cat "My Report.txt"
```

hoặc:

```bash
cat My\ Report.txt
```

Quoting là shell concept rất quan trọng.

---

# 74. File và Directory Operations

## `touch`

Ví dụ:

```bash
touch file.txt
```

Nếu file chưa tồn tại, thường tạo empty file.

Nếu đã tồn tại, `touch` thường cập nhật timestamps thay vì "tạo lại" file.

Điều này quan trọng vì command name dễ khiến beginner hiểu sai.

---

# 75. `mkdir`

Tạo directory:

```bash
mkdir reports
```

Tạo parent hierarchy khi cần:

```bash
mkdir -p lab/input
```

`-p` cần được hiểu trước khi dùng, đặc biệt trong automation.

---

# 76. `cp`

Copy file:

```bash
cp source.txt destination.txt
```

Copy directory thường cần recursive option:

```bash
cp -r source_dir destination_dir
```

Nhưng recursive copy có thể tác động nhiều data; trước production operation phải kiểm tra scope.

---

# 77. `mv`

`mv` có thể:

- move;
- rename.

Ví dụ rename:

```bash
mv old.txt new.txt
```

Move:

```bash
mv report.txt archive/
```

Cần lưu ý overwrite behavior và filesystem semantics tùy operation.

---

# 78. `rm`

Remove file:

```bash
rm file.txt
```

Recursive removal:

```bash
rm -r directory
```

Các combinations sử dụng `-f` hoặc recursive force có rủi ro cao vì giảm protection prompts/error handling.

Day 1 cần hiểu nguyên tắc:

> Không chạy destructive recursive command trên path mà bạn chưa verify.

Production safety nên bắt đầu bằng:

```bash
pwd
ls -ld target
```

và kiểm tra scope trước.

---

# 79. Linux deletion không có "Recycle Bin" mặc định ở shell

CLI:

```bash
rm file
```

thường không đưa file vào desktop recycle bin.

Recovery có thể rất khó hoặc không khả thi.

Do đó `rm` phải được xem là operation cần attention.

---

# PHẦN V — COMMAND SYNTAX

# 80. General structure

Mô hình:

```text
command [options] [arguments]
```

Ví dụ:

```bash
ls -l /etc
```

Trong đó:

```text
ls      command
-l      option
/etc    argument
```

---

# 81. Short và long options

Một số commands có:

```text
-h
```

và long option dạng:

```text
--help
```

Không được suy luận rằng cùng một option letter có cùng ý nghĩa ở mọi command.

Ví dụ `-r` ở command này có thể hoàn toàn khác `-r` ở command khác.

Khi không chắc:

```bash
command --help
```

hoặc:

```bash
man command
```

---

# 82. Argument order

Không nên giả định mọi commands đều parse arguments giống nhau.

GNU tools thường khá flexible nhưng scripts, vendor utilities hoặc POSIX tools có thể khác.

Operational habit:

> Dùng syntax được documentation xác nhận.

---

# PHẦN VI — STANDARD STREAMS, PIPES VÀ REDIRECTION

# 83. stdin, stdout, stderr

Unix/Linux process model thường có ba standard streams:

```text
0 = stdin
1 = stdout
2 = stderr
```

### `stdin`

Standard input.

### `stdout`

Normal output.

### `stderr`

Error/diagnostic output.

Understanding này rất quan trọng cho redirection và scripting sau này.

---

# 84. Pipe `|`

Pipe kết nối:

```text
stdout của process A
        ↓
stdin của process B
```

Ví dụ:

```bash
ls -l /etc | grep ssh
```

Flow:

```text
ls
 ↓ stdout
pipe
 ↓ stdin
grep
```

Shell không nhất thiết tạo temporary file giữa hai commands.

---

# 85. Pipe không truyền stderr mặc định

Ví dụ:

```bash
somecommand | grep text
```

thường pipe chỉ nhận stdout.

Errors trên stderr có thể vẫn hiển thị terminal.

Đây là lý do beginner đôi khi thắc mắc tại sao output "vẫn xuất hiện" dù đã pipe/filter.

---

# 86. `>` — overwrite stdout destination

Ví dụ:

```bash
command > output.txt
```

Thường stdout được redirect vào file.

Nếu file tồn tại, nó thường bị truncate trước khi ghi.

Đây là rủi ro quan trọng.

---

# 87. `>>` — append

```bash
command >> output.txt
```

Append stdout thay vì truncate existing file.

Không có nghĩa luôn an toàn; file vẫn có thể tăng kích thước không giới hạn nếu dùng sai.

---

# 88. `<` — input redirection

Ví dụ conceptual:

```bash
command < input.txt
```

Program lấy stdin từ file.

Không phải command nào cũng được thiết kế để xử lý stdin theo cùng cách.

---

# 89. `2>`

Redirect stderr:

```bash
command 2> errors.txt
```

Trong đó:

```text
2 = stderr
```

---

# 90. `2>&1` — Kiến thức bổ sung cần thiết

Pattern:

```bash
command > output.log 2>&1
```

nghĩa là:

1. stdout đi `output.log`;
2. stderr được redirect tới nơi stdout đang trỏ.

Order có ý nghĩa trong shell redirection.

Day 1 chưa cần làm shell scripting sâu nhưng nên hiểu cơ chế.

---

# 91. `tee` — Kiến thức bổ sung

Khi muốn:

```text
see output on screen
+
write output to file
```

có thể dùng:

```bash
command | tee output.txt
```

Đây là tool rất hữu ích cho evidence collection.

Nhưng cần đảm bảo output không chứa secret.

---

# PHẦN VII — COMMAND HISTORY

# 92. `history`

Bash thường cung cấp:

```bash
history
```

để xem command history.

History hữu ích cho:

```text
Recall
Reproduce
Troubleshoot
Review
```

---

# 93. Security implications của history

Không được nhập secrets trực tiếp vào command line nếu tool hỗ trợ phương thức an toàn khác.

Ví dụ conceptual command:

```text
tool --password MySecret123
```

có thể làm secret xuất hiện trong:

```text
shell history
process listing
logs
terminal recording
```

depending on environment.

Do đó operational principle:

> Credentials không được xem như ordinary command arguments nếu có secure alternative.

---

# 94. History không phải audit log hoàn chỉnh

Không được coi shell history là authoritative audit evidence.

Vì history có thể:

- bị disabled;
- không ghi mọi thứ;
- bị sửa;
- phụ thuộc shell/session;
- thiếu privileged context.

Enterprise audit thường cần central logging hoặc privileged session records, nhưng đây là kiến thức cho các Day sau.

---

# PHẦ VIII — HELP UTILITIES

# 95. `man`

Ví dụ:

```bash
man chmod
```

Manual page thường cung cấp:

```text
NAME
SYNOPSIS
DESCRIPTION
OPTIONS
EXAMPLES
FILES
SEE ALSO
```

Không phải mọi man page đều có tất cả sections.

---

# 96. Đọc SYNOPSIS

Ví dụ conceptual:

```text
command [OPTION]... FILE...
```

Convention thường:

```text
[]      optional
...     repeatable
```

Hiểu synopsis giúp giảm việc chạy command sai.

---

# 97. `--help`

Ví dụ:

```bash
chmod --help
```

thường nhanh hơn `man` khi chỉ cần syntax/options.

---

# 98. Shell builtin `help`

Một số commands là shell builtins.

Ví dụ trong Bash:

```bash
help cd
```

có thể phù hợp hơn:

```bash
man cd
```

vì `cd` được shell thực hiện.

---

# 99. `whatis` và `apropos` — Kiến thức bổ sung

Khi biết command:

```bash
whatis passwd
```

có thể xem short description.

Khi chưa biết command:

```bash
apropos keyword
```

có thể tìm manual database nếu system được cấu hình phù hợp.

---

# PHẦN IX — SAFE COMMAND EXECUTION

# 100. Mental checklist trước command

Trước configuration-changing command, phải tự hỏi:

```text
1. Tôi đang ở đúng host?
2. Tôi đang dùng đúng account?
3. Đây là đúng environment?
4. Current directory là gì?
5. Command tác động objects nào?
6. Tôi hiểu options không?
7. Có authorization không?
8. Blast radius là gì?
9. Có rollback không?
10. Tôi sẽ verify bằng cách nào?
11. Evidence nào cần lưu?
```

Nếu chưa trả lời được câu quan trọng, đừng vội chạy command.

---

# 101. Dry observation trước write operation

Ví dụ bạn muốn change permissions.

Thay vì chạy ngay:

```bash
chmod ...
```

hãy inspect:

```bash
ls -l target
```

và nếu directory:

```bash
ls -ld target
```

Hai command khác nhau.

```bash
ls -l directory
```

thường list contents.

```bash
ls -ld directory
```

inspect directory object itself.

Đây là một distinction rất hữu ích.

---

# 102. Preserve before-state evidence

Trước:

```text
Change A
```

hãy lưu:

```text
State before A
```

Sau:

```text
State after A
```

Nhờ vậy bạn vừa có:

```text
verification
+
rollback reference
+
audit evidence
```

---

# PHẦ X — LINUX USERS VÀ GROUPS

# 103. User identity

Linux kernel không thực sự nhìn username theo cách con người nhìn.

Nó chủ yếu xử lý numeric identity:

```text
UID
GID
```

Username/group name là human-readable mappings.

Ví dụ:

```bash
id alice
```

có thể cho:

```text
uid=1001(alice) gid=1001(alice) groups=1001(alice),...
```

Exact IDs thay đổi giữa systems.

---

# 104. Root user

`root` thường có:

```text
UID 0
```

UID 0 có đặc quyền cao nhất trong traditional Unix/Linux privilege model.

Điều quan trọng:

> Account name không phải yếu tố duy nhất; UID mới là core identity.

Một account khác có UID 0 cũng có root-equivalent privilege.

Do đó duplicated UID 0 là security concern nghiêm trọng.

---

# 105. Regular users và system accounts

Có thể tồn tại:

```text
Human interactive accounts
Service accounts
System accounts
```

Không nên dựa chỉ vào UID range để kết luận account type nếu chưa biết distribution policy.

System accounts thường phục vụ services và không nhất thiết cần interactive login.

---

# 106. `/etc/passwd`

Thông thường chứa account metadata.

Format conceptual:

```text
name:x:UID:GID:GECOS:home:shell
```

Ví dụ generic:

```text
alice:x:1001:1001:Alice:/home/alice:/bin/bash
```

Field `x` thường cho biết password hash không nằm trực tiếp ở đây.

`/etc/passwd` thường readable để system/tools map IDs, vì vậy nó không nên chứa password hash hiện đại.

---

# 107. `/etc/shadow`

Thường chứa password-related security fields và account aging data.

Access bị giới hạn.

Không nên:

```bash
cat /etc/shadow
```

chỉ vì tò mò hoặc để chụp evidence.

Nó chứa sensitive security information.

---

# 108. `/etc/group`

Chứa group information.

Conceptual format:

```text
group:x:GID:member1,member2
```

Nhưng membership phải được hiểu cùng primary group data từ user account.

---

# 109. Primary và supplementary groups

Một user có:

```text
one primary group
```

và có thể có:

```text
zero or more supplementary groups
```

Kiểm tra:

```bash
id username
```

hoặc:

```bash
groups username
```

---

# 110. Vì sao groups quan trọng?

Thay vì cấp access từng user:

```text
Alice → permission
Bob   → permission
Carol → permission
```

có thể dùng:

```text
appops group
     ↓
filesystem access
```

rồi quản lý membership.

Đây thường dễ maintain hơn và phù hợp role-based administration.

---

# 111. `id`

Một trong những command quan trọng nhất Day 1:

```bash
id
```

cho current user.

```bash
id username
```

cho user cụ thể.

Nó giúp verify:

```text
UID
primary GID
supplementary groups
```

---

# 112. `whoami`

```bash
whoami
```

cho effective username.

Nó rất hữu ích sau SSH hoặc `sudo` context changes.

---

# 113. `getent`

`getent` truy vấn Name Service Switch databases.

Ví dụ:

```bash
getent passwd alice
```

So với trực tiếp:

```bash
grep alice /etc/passwd
```

`getent` có advantage rằng environment có thể dùng centralized identity source chứ không chỉ local files.

Đây là **Kiến thức bổ sung rất hữu ích** cho enterprise operations.

---

# 114. User management commands

Thông thường gặp:

```text
useradd
usermod
userdel
passwd
```

Group:

```text
groupadd
groupmod
groupdel
```

Behavior về default home directory, shell hoặc skeleton files có thể phụ thuộc distribution/configuration.

Không nên memorise defaults rồi giả định giống mọi server.

---

# 115. `useradd`

Ví dụ conceptual:

```bash
sudo useradd username
```

Tạo user theo system defaults.

Trong enterprise, trước đó cần:

```text
Approved naming
UID policy
Group assignment
Home requirement
Shell requirement
Expiry requirement
Ticket approval
```

---

# 116. `usermod`

Có thể thay đổi user attributes.

Một common pitfall rất quan trọng:

Khi quản lý supplementary groups, một option có thể thay thế toàn bộ group list nếu dùng sai thay vì append.

Do đó không chạy `usermod` theo trí nhớ nếu không chắc syntax; hãy đọc:

```bash
man usermod
```

Đây chính là ví dụ của safe command execution.

---

# 117. `userdel`

Xóa account là potentially destructive operation.

Tùy options có thể liên quan home directory/data.

Trong production:

> Account deprovisioning phải đi cùng ownership/data/access review, không chỉ chạy một command.

---

# PHẦ XI — FILE OWNERSHIP VÀ PERMISSIONS

# 118. Linux DAC model

Traditional Linux filesystem permission model thuộc loại `Discretionary Access Control` — DAC.

Mỗi filesystem object thường có:

```text
Owner UID
Group GID
Permission mode
```

Kernel dùng các thông tin này để quyết định access cùng các security mechanisms khác.

Day 1 tập trung basic DAC.

Day 2 sẽ giới thiệu SELinux, một layer khác.

---

# 119. Đọc `ls -l`

Ví dụ:

```text
-rwxr-x--- 1 alice appops 1024 Oct 18 10:30 script.sh
```

Các fields quan trọng:

```text
-rwxr-x---
alice
appops
```

Mode:

```text
- rwx r-x ---
```

Tách ra:

```text
-     object type indicator
rwx   owner permissions
r-x   group permissions
---   other permissions
```

---

# 120. First character

Một số common examples:

```text
-   regular file
d   directory
l   symbolic link
```

Có thêm các file types khác nhưng Day 1 chưa cần đi sâu.

---

# 121. `r`, `w`, `x` trên regular file

Đối với regular file:

### `r` — read

Được đọc content.

### `w` — write

Được modify content.

### `x` — execute

Có thể execute như program/script nếu các điều kiện khác phù hợp.

---

# 122. `r`, `w`, `x` trên directory

Đây là kiến thức cực kỳ quan trọng vì semantics khác file.

### `r` trên directory

Cho phép đọc directory entries/list names.

### `w` trên directory

Cho phép thay đổi directory entries, ví dụ create/delete/rename entries, khi các điều kiện khác cho phép.

### `x` trên directory

Cho phép traverse/search directory.

Nếu thiếu `x`, user có thể gặp:

```text
Permission denied
```

ngay cả khi target file có read permission.

---

# 123. Parent directory permissions

Giả sử:

```text
/data/projects/report.txt
```

Để access `report.txt`, process phải traverse:

```text
/
→ /data
→ /data/projects
→ report.txt
```

Nếu thiếu execute/search permission ở parent directory phù hợp, access có thể fail.

Do đó troubleshooting không chỉ nhìn:

```bash
ls -l report.txt
```

mà còn phải xem path components.

---

# 124. Deleting a file

Một common misconception:

> "File không có write permission nên không thể xóa."

Trong Unix filesystem semantics, deleting một filename chủ yếu là operation trên **parent directory entry**.

Do đó directory permissions rất quan trọng.

Đây là lý do hiểu semantics tốt hơn học thuộc `rwx`.

---

# 125. Permission classes

Ba classes:

```text
u = user/owner
g = group
o = other
```

và:

```text
a = all
```

trong symbolic `chmod` context.

---

# 126. `chmod` symbolic mode

Ví dụ:

```bash
chmod u+x script.sh
```

nghĩa là:

```text
u → owner
+ → add
x → execute
```

Ví dụ:

```bash
chmod g-w file.txt
```

remove group write.

Ví dụ:

```bash
chmod o-r file.txt
```

remove other read.

Symbolic mode có lợi khi muốn thay đúng một permission mà không rewrite toàn mode.

---

# 127. Numeric/octal mode

Values:

```text
r = 4
w = 2
x = 1
```

Tổng theo từng class.

Ví dụ:

```text
7 = 4 + 2 + 1 = rwx
6 = 4 + 2     = rw-
5 = 4 + 1     = r-x
4 = 4         = r--
0             = ---
```

---

# 128. Ví dụ `chmod 750`

```text
7 = rwx
5 = r-x
0 = ---
```

Nên:

```text
750
```

tương ứng:

```text
rwxr-x---
```

---

# 129. Ví dụ `640`

```text
6 = rw-
4 = r--
0 = ---
```

tương ứng:

```text
rw-r-----
```

Không nên học:

```text
640 = secure
755 = normal
777 = bad
```

một cách máy móc.

Correct permission luôn phụ thuộc business requirement.

---

# 130. Tại sao `777` thường là red flag?

```text
777
```

cho:

```text
owner rwx
group rwx
other rwx
```

Nếu áp dụng tùy tiện, mọi local user có thể có quyền quá lớn.

Nhưng câu đúng hơn không phải:

> "777 luôn luôn sai."

mà là:

> `777` hiếm khi phù hợp cho enterprise application data và phải có business/security justification nếu sử dụng.

---

# 131. `chown`

Change ownership.

Generic syntax:

```bash
chown owner file
```

hoặc:

```bash
chown owner:group file
```

Ví dụ conceptual:

```bash
chown appuser:appgroup config.dat
```

Đây là state-changing privileged operation trong nhiều situations.

---

# 132. `chgrp`

Change group ownership:

```bash
chgrp appgroup file
```

Functionally group owner cũng có thể được thay đổi bằng một số forms của `chown`, nhưng `chgrp` thể hiện intent rõ.

---

# 133. Ownership vs permissions

Sai ownership và sai permission tạo symptom gần giống nhau.

Ví dụ:

```text
Permission denied
```

có thể do:

```text
Wrong owner
Wrong group
Missing group membership
Wrong mode
Missing parent x permission
Other security control
```

Do đó không nên giải quyết `Permission denied` bằng `chmod 777`.

---

# 134. Verification

Read-only verification:

```bash
ls -l file
```

Detailed metadata:

```bash
stat file
```

User identity:

```bash
id username
```

Khi troubleshoot, cần xem cả:

```text
file state
+
user identity/group membership
+
parent directory state
```

---

# 135. Escalation boundary cho permissions

L1/L2 boundary phụ thuộc organization, nhưng nên escalate khi:

- permission tree phức tạp;
- owner/group change có thể ảnh hưởng nhiều services;
- shared storage/NFS involved;
- ACLs hoặc SELinux có liên quan ngoài scope;
- proposed recursive `chmod/chown`;
- sensitive production data;
- baseline không rõ;
- root cause chưa xác định.

---

# PHẦ XII — `umask`

# 136. `umask` là gì?

`umask` là mask loại bỏ permission bits khi process tạo filesystem objects.

Điểm rất quan trọng:

> `umask` không phải "default permission" trực tiếp.

Nó mask bits từ base creation mode.

---

# 137. Base modes

Thông thường conceptual model:

Regular file:

```text
666
```

Directory:

```text
777
```

Lý do regular files không mặc định executable.

---

# 138. Ví dụ `umask 022`

File:

```text
666
masked by 022
→ 644
```

Directory:

```text
777
masked by 022
→ 755
```

Kết quả:

File:

```text
rw-r--r--
```

Directory:

```text
rwxr-xr-x
```

---

# 139. `umask 027`

File:

```text
666
→ 640
```

Directory:

```text
777
→ 750
```

Tương ứng:

```text
file      rw-r-----
directory rwxr-x---
```

---

# 140. Vì sao nói "666 - 022 = 644" chỉ là shortcut?

Về bản chất `umask` là bit mask operation, không phải arithmetic subtraction theo nghĩa chung.

Với các examples phổ biến phép trừ dễ cho đúng result, nhưng nên hiểu actual logic:

```text
requested_mode AND NOT umask
```

Hiểu bản chất giúp tránh sai ở các combination phức tạp.

---

# 141. `umask` chỉ ảnh hưởng creation

Thay đổi `umask` không tự đổi permissions của existing files.

Nó ảnh hưởng objects được tạo sau đó bởi process/context liên quan.

---

# 142. `umask` và child processes

Process thường inherit `umask` từ parent.

Do đó shell/session configuration có thể ảnh hưởng applications được launch từ đó.

Services cũng có thể có configuration riêng.

Không nên giả định interactive shell `umask` luôn giống service `umask`.

---

# PHẦ XIII — `sudo`

# 143. `sudo` giải quyết vấn đề gì?

Nếu mọi administrator đều login trực tiếp root:

```text
Who did what?
```

khó trả lời hơn.

Mô hình tốt hơn:

```text
Named account
   ↓
sudo
   ↓
Specific approved privileged operation
```

Benefits:

```text
Individual accountability
Least privilege
Auditability
Controlled elevation
```

---

# 144. `sudo` không có nghĩa "become root"

Mặc dù use case phổ biến là execute command với root privileges, `sudo` có thể được cấu hình để run as other identities.

Concept quan trọng là:

```text
controlled privilege delegation
```

---

# 145. Basic usage

```bash
sudo command
```

Ví dụ read-only operation được authorized:

```bash
sudo some-admin-query
```

Exact command phụ thuộc environment.

Không cần chạy `sudo` nếu operation không yêu cầu privilege.

Nguyên tắc:

> Không elevate chỉ vì "cho chắc".

---

# 146. `sudo -l`

Một command rất hữu ích:

```bash
sudo -l
```

Hiển thị privileges mà current user được phép sử dụng theo configuration/applicable policy.

Đây thường là verification command quan trọng.

---

# 147. `sudoers`

Main policy thường liên quan:

```text
/etc/sudoers
```

và có thể include fragments dưới:

```text
/etc/sudoers.d/
```

Exact configuration phụ thuộc system.

---

# 148. Vì sao dùng `visudo`?

Editing sudo policy sai syntax có thể làm mất privileged access.

`visudo` hỗ trợ syntax validation và locking phù hợp.

Do đó không nên tùy tiện:

```bash
vi /etc/sudoers
```

trên production chỉ vì bạn biết `vi`.

Use approved procedure.

---

# 149. Sudo rule conceptual structure

Một rule về mặt concept xác định:

```text
WHO
ON WHICH HOST
RUN AS WHOM
WHICH COMMANDS
WITH WHAT OPTIONS
```

Mục tiêu least privilege là thu hẹp các dimensions này.

---

# 150. Broad sudo rule

Một rule quá rộng kiểu:

```text
user may run almost anything as root
```

gần tương đương full administrative capability.

Nếu business requirement chỉ là một controlled operation, privilege nên giới hạn hơn.

---

# 151. Command path quan trọng

Sudo policies thường reference command path.

Do đó cần biết actual executable:

```bash
command -v commandname
```

Không giả định executable luôn nằm cùng path trên mọi distribution.

---

# 152. Wildcards trong sudoers

Wildcards có thể mở rộng privilege theo những cách không mong muốn.

Một rule tưởng "chỉ cho một folder" có thể dẫn tới unintended access nếu arguments/tool behavior không được hiểu kỹ.

Vì vậy production sudo policy design cần security review khi phức tạp.

---

# 153. Shell escape risk

Một số programs tưởng chỉ "view file" nhưng có khả năng:

```text
launch shell
execute external commands
write arbitrary files
```

Nếu cho program đó chạy qua `sudo`, bạn có thể vô tình cấp quyền lớn hơn dự kiến.

Least-privilege `sudo` vì vậy không chỉ là chọn command name.

Phải hiểu tool capabilities.

---

# 154. `NOPASSWD`

`NOPASSWD` có thể phù hợp trong một số controlled automation contexts, nhưng giảm authentication friction và tăng risk nếu account/session compromise.

Không được coi:

```text
NOPASSWD = convenient = good
```

Mỗi use case cần security justification.

---

# 155. `sudo -k`

Có thể invalidate cached sudo credential state trong common configurations:

```bash
sudo -k
```

Hữu ích khi test privilege/authentication behavior trong lab.

Exact behavior vẫn phụ thuộc policy/version.

---

# 156. Evidence cho sudo

Evidence nên chứng minh:

```text
Approved command → allowed
Unapproved command → denied
```

Nhưng không được để lộ secrets.

Một access test chỉ chứng minh allowed case chưa đủ để chứng minh least privilege.

Negative test rất quan trọng.

---

# PHẦ XIV — SSH FUNDAMENTALS

# 157. SSH là gì?

`SSH` — Secure Shell — là protocol dùng cho secure remote communication và administration.

Common use cases:

```text
Remote shell
Remote command execution
Secure file transfer
Tunneling
Authentication
```

Day 1 tập trung remote administrative access và authentication.

---

# 158. Client/server model

Conceptual flow:

```text
Administrative workstation
        |
        | SSH
        v
Linux server
    sshd / SSH server
```

Client có thể dùng:

```bash
ssh username@server
```

Server thường chạy SSH daemon.

Trên many Linux systems daemon/service thường liên quan `sshd`, nhưng service management học sâu ở Day 2.

---

# 159. TCP port

SSH mặc định sử dụng:

```text
TCP/22
```

Nhưng administrator có thể cấu hình port khác.

Vì vậy:

> "SSH = always port 22" là không chính xác.

Đúng hơn:

> TCP/22 là default well-known SSH port.

---

# 160. SSH connection phases

Simplified:

```text
1. TCP connection
2. Protocol negotiation
3. Key exchange
4. Server host authentication
5. Encrypted channel established
6. User authentication
7. Authorized session/command
```

Đây là model rất hữu ích để troubleshoot.

---

# 161. Host authentication khác user authentication

Đây là distinction cực kỳ quan trọng.

### Host authentication

Client hỏi:

> "Đây có thực sự là server mà tôi muốn kết nối không?"

Sử dụng server `host key`.

### User authentication

Server hỏi:

> "Client/user này có được phép đăng nhập không?"

Có thể dùng:

```text
Password
Public-key authentication
Other approved methods
```

Hai authentication directions khác nhau.

---

# 162. `known_hosts`

Client thường lưu server host keys hoặc fingerprints trong:

```text
~/.ssh/known_hosts
```

Khi host key thay đổi, client có thể cảnh báo.

Không nên reflexively xóa warning chỉ để login được.

Host key mismatch có thể do:

```text
Legitimate server rebuild
IP/hostname reassignment
SSH host key rotation
Wrong target
Man-in-the-middle risk
```

Cần verify theo approved process.

---

# 163. SSH public-key authentication

Public-key authentication dùng asymmetric key pair:

```text
Private key
Public key
```

Simplified:

```text
Client holds private key
Server trusts corresponding public key
```

Private key không được gửi sang server như password.

Protocol sử dụng cryptographic proof để chứng minh client possesses private key.

---

# 164. Private key

Private key phải được coi là secret.

Không:

```text
Email private key
Paste private key into ticket
Store in shared evidence folder
Screenshot private key contents
Commit private key to source control
```

Nếu private key bị exposure, phải xử lý như credential compromise theo organization policy.

---

# 165. Public key

Public key không phải secret giống private key.

Public key có thể được đặt ở authorized location của server.

Nhưng vẫn cần quản lý vì nó đại diện cho trust relationship.

Nếu một public key không còn cần thiết, phải revoke/remove theo access process.

---

# 166. `authorized_keys`

Common default path:

```text
~/.ssh/authorized_keys
```

Nó chứa public keys được authorize cho account đó, trừ khi server configuration chỉ định khác.

Không được hiểu:

> Có public key trong file = chắc chắn login sẽ được.

Server còn xem xét:

```text
Account state
SSH server policy
File ownership
Permissions
Allowed authentication methods
Key restrictions
Security policies
```

---

# 167. `.ssh` permissions

Common secure conventions thường là:

```text
~/.ssh                  700
~/.ssh/authorized_keys  600
```

và ownership thuộc đúng user.

Nhưng quan trọng hơn việc học thuộc `700/600` là hiểu:

> SSH server không muốn trust authentication material có thể bị unauthorized users chỉnh sửa.

Exact acceptance rules phụ thuộc OpenSSH configuration/platform.

---

# 168. Private key permissions

Client private key cũng phải được bảo vệ.

Nếu permissions quá permissive, OpenSSH client có thể từ chối sử dụng key.

Đây là deliberate security behavior.

---

# 169. `ssh-keygen`

SSH key pair được tạo bằng:

```bash
ssh-keygen
```

Algorithm phải theo organization policy và platform compatibility.

Không nên nói rằng một algorithm duy nhất luôn là lựa chọn đúng cho mọi enterprise environment, vì compliance/FIPS/platform policies có thể khác.

---

# 170. Passphrase

Private key có thể được protected bằng passphrase.

Passphrase giúp giảm risk nếu key file bị copy.

Không nên nhầm:

```text
SSH account password
```

với:

```text
Private key passphrase
```

Đó là hai credentials khác nhau.

---

# 171. `ssh-agent` — Kiến thức bổ sung

`ssh-agent` có thể giữ private key material/unlocked identities trong user session để không phải nhập passphrase cho mỗi connection.

Nó tăng usability nhưng cũng tạo security considerations:

```text
Agent session security
Forwarding risk
Workstation compromise
```

Day 1 chỉ cần awareness.

---

# 172. SSH key fingerprint

Fingerprint là compact identifier của key.

Nó hữu ích để:

```text
Verify correct key
Compare approvals
Identify authorized key
```

thay vì so entire key text bằng mắt.

---

# 173. `ssh-copy-id`

Một số environments có:

```bash
ssh-copy-id user@host
```

để install public key.

Nhưng enterprise environment có thể yêu cầu controlled provisioning thay vì tự động append.

Vì vậy:

> Tool availability không đồng nghĩa procedure được authorization.

---

# 174. SSH connection command

Generic:

```bash
ssh user@hostname
```

Nếu port không default:

```bash
ssh -p PORT user@hostname
```

Nếu cần chọn identity:

```bash
ssh -i /path/to/private_key user@hostname
```

Không nên share actual private key paths/evidence nếu chúng tiết lộ sensitive workstation structure không cần thiết.

---

# 175. Debugging SSH

Client verbosity có thể tăng bằng:

```bash
ssh -v user@hostname
```

và có các mức verbose cao hơn.

Output rất hữu ích để phân biệt:

```text
Connection failure
Host-key stage
Authentication method
Key offered
Key rejected
Session establishment
```

Nhưng debug output có thể chứa environment details, usernames, hostnames hoặc metadata nhạy cảm. Phải sanitize trước khi gửi ticket bên ngoài authorized audience.

---

# PHẦ XV — SSH TROUBLESHOOTING MODEL

Một error:

```text
Permission denied
```

không đủ để kết luận root cause.

Phải xác định failure stage.

---

# 176. Stage 1 — Target correctness

Hỏi:

```text
Đúng hostname?
Đúng IP?
Đúng environment?
```

Một login failure đôi khi chỉ là connecting sai server.

Evidence:

```bash
hostname
```

sau khi có một access path hợp lệ, hoặc asset inventory/DNS data theo scope.

---

# 177. Stage 2 — Network connection

Nếu error kiểu:

```text
Connection timed out
Connection refused
No route to host
```

đây chưa chắc là authentication issue.

Có thể liên quan:

```text
Network
Firewall
Port
Service
Routing
```

Networking troubleshooting sâu thuộc Day 2/8/9.

Day 1 chỉ cần biết **phân loại failure stage**.

---

# 178. Stage 3 — Host identity

Nếu host key warning:

```text
REMOTE HOST IDENTIFICATION HAS CHANGED
```

không nên xóa entry ngay.

Trước tiên verify:

```text
Was host rebuilt?
Was IP reused?
Was key legitimately rotated?
Is this the correct asset?
```

Nếu không xác nhận được, escalate.

---

# 179. Stage 4 — Username

Một public key có thể đúng nhưng authorized cho:

```text
alice
```

không phải:

```text
bob
```

Do đó username là part of authentication context.

---

# 180. Stage 5 — Key availability

Kiểm tra:

```text
Đúng private key?
File accessible?
Agent đang giữ đúng identity?
```

Không dùng trial-and-error với random keys trên sensitive systems vì có thể tạo noise/audit alerts.

---

# 181. Stage 6 — Server authorization

Server phải có corresponding trust.

Common causes:

```text
Public key missing
Wrong authorized_keys
Wrong owner
Wrong permissions
Account disabled
Server policy rejects method
Key restriction
```

---

# 182. Stage 7 — Post-authentication access

Có thể authentication thành công nhưng session vẫn fail vì:

```text
Invalid shell
Account restriction
Forced command
Policy
Resource problem
```

Do đó cần distinguish:

```text
Authentication success
```

và:

```text
Usable administrative session
```

---

# PHẦ XVI — SECURE ADMINISTRATIVE ACCESS

# 183. Named accounts

Shared account:

```text
opsadmin
```

được 20 người dùng chung làm giảm attribution.

Tốt hơn:

```text
alice
bob
carol
```

mỗi người có named identity và elevation riêng.

---

# 184. Root login

Nhiều organizations hạn chế direct remote root login, thay vào đó dùng:

```text
Named account
→ authenticated SSH
→ sudo
```

Tuy nhiên đây là security policy decision và server configuration phụ thuộc environment; không nên khẳng định mọi system đều phải giống nhau.

---

# 185. Administrative chain

Ideal conceptual chain:

```text
Human administrator
        ↓
Named identity
        ↓
Approved SSH authentication
        ↓
Linux account
        ↓
Least privilege
        ↓
sudo only when required
        ↓
Auditable administrative action
```

Đây là một trong những mental models quan trọng nhất Day 1.

---

# 186. Secure workstation

Secure access không chỉ phụ thuộc server.

Nếu administrative workstation bị compromise, private key và privileged session cũng có thể bị compromise.

Do đó workstation cần:

```text
Endpoint protection
OS patching
Screen lock
Credential protection
Approved software
Controlled storage
```

Các chi tiết endpoint security ngoài scope Day 1, nhưng concept phải hiểu.

---

# PHẦ XVII — VERIFICATION THINKING

Một Infrastructure Operations engineer không chỉ "configure". Phải verify.

Có ba lớp verification nên phân biệt.

---

# 187. Configuration verification

Xác nhận state được cấu hình như mong muốn.

Ví dụ:

```bash
ls -l file
```

cho permission.

```bash
id user
```

cho group membership.

```bash
sudo -l
```

cho privilege.

---

# 188. Functional verification

Xác nhận operation thật sự hoạt động.

Ví dụ:

```text
User expected to read file → can read
User expected to SSH → can authenticate
Approved sudo operation → works
```

---

# 189. Negative verification

Xác nhận operation không được phép thực sự bị deny.

Ví dụ:

```text
Unauthorized user → cannot read
Unauthorized sudo command → denied
Unapproved SSH key → rejected
```

Negative testing rất quan trọng cho security controls.

Một security control chưa được test negative path thì chưa được chứng minh đầy đủ.

---

# 190. Why configuration-only verification is insufficient

Giả sử:

```text
ls -l
```

trông đúng.

Nhưng user vẫn không access được vì:

```text
missing parent x
wrong group membership
SELinux
ACL
application running as different identity
```

Configuration inspection không thay thế functional test.

---

# PHẦ XVIII — TROUBLESHOOTING PHILOSOPHY DAY 1

Troubleshooting ở Day 1 nên theo nguyên tắc từ ít rủi ro đến nhiều rủi ro.

```text
Observe
↓
Confirm context
↓
Reproduce safely
↓
Gather evidence
↓
Inspect dependencies
↓
Form hypothesis
↓
Test hypothesis read-only if possible
↓
Apply approved smallest change
↓
Verify
↓
Document
```

---

# 191. Không "shotgun troubleshoot"

`Shotgun troubleshooting` là đổi nhiều thứ cùng lúc:

```text
chmod
chown
restart
change config
reboot
```

rồi thấy system hoạt động.

Kết quả:

- không biết root cause;
- khó rollback;
- khó audit;
- có thể tạo latent security issue.

Một change tốt nên controlled và attributable.

---

# 192. One-variable-at-a-time principle

Trong controlled troubleshooting:

```text
Observe A
Change one relevant variable
Observe B
```

giúp xác định causal relationship tốt hơn.

Không phải mọi emergency cho phép process hoàn hảo, nhưng đây là default tốt.

---

# 193. Permission troubleshooting example

Symptom:

```text
alice cannot read /srv/report/data.txt
```

Không sửa ngay.

Investigate:

```text
1. Confirm alice identity/groups
2. Inspect target permissions
3. Inspect owner/group
4. Inspect parent directory permissions
5. Determine expected access model
6. Compare to baseline/request
7. Only then propose change
```

Read-only commands có thể là:

```bash
id alice
ls -ld /srv
ls -ld /srv/report
ls -l /srv/report/data.txt
```

Sau này có thể cần ACL/SELinux investigation, nhưng đó vượt core Day 1.

---

# PHẦ XIX — EVIDENCE COLLECTION

# 194. Command evidence

Một command/output artifact tốt nên có context.

Ví dụ conceptual evidence:

```text
Timestamp: 2026-10-18 10:00 +07
Host: rhel-lab01
User: student
Purpose: Verify ownership and permissions

$ ls -l /lab/report.txt
-rw-r----- 1 appuser appops ...
```

Reviewer ngay lập tức biết:

```text
When
Where
Who
Why
Result
```

---

# 195. Before/after evidence

Ví dụ:

```text
BEFORE
owner = alice
group = alice
mode  = 600

APPROVED CHANGE
...

AFTER
owner = appuser
group = appops
mode  = 640
```

Đây là evidence có giá trị hơn chỉ lưu after state.

---

# 196. Negative evidence

Ví dụ:

```text
Expected:
guest must not read file.

Actual:
Permission denied.

Result:
PASS
```

Security evidence phải chứng minh denied case một cách controlled.

---

# 197. Evidence sanitization

Trước submit:

Không để:

```text
Private key
Password
Passphrase
Token
Credential
Sensitive data
```

Nếu command output chứa sensitive fields, cần redact theo organization procedure nhưng vẫn giữ evidence integrity đủ dùng.

---

# PHẦ XX — COMMON MISTAKES DAY 1

Đây là nhóm sai lầm bạn cần chủ động tránh.

### Sai target

Chạy đúng command trên sai host.

### Sai environment

Tưởng lab nhưng thực ra production.

### Dùng root cho mọi việc

Làm mất least privilege và audit attribution.

### `chmod 777` để "fix nhanh"

Che symptom và tạo security exposure.

### Recursive operation thiếu scope

Ví dụ recursive `chmod/chown/rm` trên directory tree không được review.

### Không lưu before state

Không có rollback reference.

### Chỉ test positive path

Access được nhưng chưa chứng minh unauthorized access bị deny.

### Nhầm authentication với authorization

Login thành công nhưng file access fail → không phải SSH authentication problem.

### Nhầm file permission và directory permission

Chỉ nhìn file mode mà quên parent directories.

### Nhầm `/` và `/root`

Rất cơ bản nhưng nguy hiểm.

### Đưa password/private key vào evidence

Security breach.

### Xóa SSH host-key warning mà không verify

Có thể bỏ qua attack hoặc target mismatch.

### Chỉnh `/etc/sudoers` trực tiếp

Risk lockout do syntax error.

### Dùng shared admin account

Audit attribution kém.

### Học thuộc command mà không hiểu effect

Đây là anti-pattern lớn nhất.

---

# PHẦ XXI — OPERATIONAL BOUNDARIES

Day 1 phải hình thành tư duy: "biết làm" không đồng nghĩa "nên làm".

Có thể chia actions thành ba vùng.

```text
LOWER RISK
Read-only inspection
Basic verification
Evidence collection

CONTROLLED CHANGE
Create lab user
Change lab permission
Configure approved sudo
Install approved public key

HIGHER RISK / ESCALATE OR REQUIRE CHANGE
Production-wide recursive permissions
Direct root-access policy change
SSH daemon security change
Critical account deletion
Unknown host-key mismatch
Privileged configuration without rollback
```

Classification cụ thể phải theo organization runbook.

---

# PHẦ XXII — TỔNG HỢP CÁC OBJECTS PHẢI HIỂU

Sau Module Lý thuyết Day 1, khi nhìn các objects sau bạn phải biết chúng là gì và liên quan với nhau như thế nào:

```text
Infrastructure asset
Environment
Account
User
UID
Group
GID
Primary group
Supplementary group
Owner
Group owner
Permissions
r / w / x
umask
sudo policy
Private key
Public key
Host key
known_hosts
authorized_keys
Configuration baseline
Runbook
SOP
Incident
Change
Evidence
Escalation
```

---

# PHẦ XXIII — DATA FLOW TỔNG HỢP

## Linux local access

```text
User
 ↓
Shell
 ↓
Command
 ↓
Kernel
 ↓
Filesystem
 ↓
UID/GID + ownership + permissions
 ↓
Allow / Deny
```

---

## SSH remote access

```text
Administrator
     ↓
Administrative workstation
     ↓
SSH client
     ↓
Host verification
     ↓
Encrypted channel
     ↓
User authentication
     ↓
Linux account
     ↓
Authorization
     ↓
Shell
     ↓
sudo when explicitly required
```

---

## Operational control flow

```text
Ticket / Request
      ↓
Authorization
      ↓
Target confirmation
      ↓
Baseline/current-state check
      ↓
Safe implementation
      ↓
Verification
      ↓
Evidence
      ↓
Close / Rollback / Escalate
```

Ba flow này khi kết hợp chính là core tư duy Day 1.

---

# PHẦ XXIV — COMMAND REFERENCE CỦA MODULE LÝ THUYẾT

Đây không phải lab procedure mà là command map bạn cần hiểu.

| Mục đích                              | Command               |
| ------------------------------------- | --------------------- |
| Current identity                      | `whoami`              |
| UID/GID/groups                        | `id`                  |
| Host                                  | `hostname`            |
| Current directory                     | `pwd`                 |
| List files                            | `ls`                  |
| Detailed listing                      | `ls -l`               |
| Hidden files                          | `ls -la`              |
| Change directory                      | `cd`                  |
| Kernel release                        | `uname -r`            |
| Kernel/system detail                  | `uname -a`            |
| OS distribution                       | `cat /etc/os-release` |
| Query account                         | `getent passwd USER`  |
| Query group                           | `getent group GROUP`  |
| Create empty file/update timestamp    | `touch`               |
| Create directory                      | `mkdir`               |
| Copy                                  | `cp`                  |
| Move/rename                           | `mv`                  |
| Remove                                | `rm`                  |
| Change mode                           | `chmod`               |
| Change owner                          | `chown`               |
| Change group owner                    | `chgrp`               |
| Inspect metadata                      | `stat`                |
| Show current `umask`                  | `umask`               |
| Check sudo privileges                 | `sudo -l`             |
| Sudo configuration validation/editing | `visudo`              |
| Generate SSH key                      | `ssh-keygen`          |
| SSH connection                        | `ssh`                 |
| SSH verbose diagnostics               | `ssh -v`              |
| Manual page                           | `man`                 |
| Command help                          | `--help`              |
| Bash builtin help                     | `help`                |
| Command history                       | `history`             |

Điều quan trọng nhất không phải thuộc bảng này mà phải biết **command nào read-only, command nào change state và command nào có destructive potential**.

---

# PHẦ XXV — READ-ONLY VÀ CHANGE COMMAND CLASSIFICATION

### Mostly read-only / verification

```bash
whoami
id
hostname
pwd
ls
uname
cat /etc/os-release
getent
stat
sudo -l
```

Lưu ý `cat` chỉ read-only khi target là ordinary readable file; không có nghĩa mọi cách đọc virtual device/file đều vô hại.

### Configuration-changing

```bash
useradd
usermod
userdel
groupadd
chmod
chown
chgrp
visudo
```

### Context-dependent

```bash
ssh-keygen
```

thay đổi local key files.

```bash
ssh
```

tự nó mở session; những gì làm sau login có thể thay đổi remote system.

```bash
cp
mv
rm
```

đều có khả năng làm mất/overwrite data tùy arguments.

---

# PHẦ XXVI — SECURITY MODEL DAY 1

Day 1 thực chất xây nền cho một security model rất quan trọng:

```text
Identity
  ↓
Authentication
  ↓
Authorization
  ↓
Least Privilege
  ↓
Accountability
  ↓
Audit Evidence
```

Ví dụ SSH:

```text
Identity:
alice

Authentication:
SSH private key

Authorization:
alice account + groups

Privilege:
sudo rule

Accountability:
named account + sudo records

Evidence:
command/log/ticket
```

Nếu hiểu flow này, Day 4 AWS IAM và Day 11–13 privileged access/IAM audit sẽ dễ hơn rất nhiều.

---

# PHẦ XXVII — CONFIGURATION VS EFFECTIVE ACCESS

Một concept nâng cao nhưng cần hiểu ngay từ Day 1:

> Configured permission không nhất thiết bằng effective permission.

Ví dụ configuration:

```text
file group = appops
group permission = read
```

Nhưng Alice không đọc được vì:

```text
Alice is not actually a member of appops
```

hoặc session được tạo trước khi group membership update.

Hoặc parent directory không cho traverse.

Hoặc security control khác deny access.

Do đó luôn distinguish:

```text
Configured state
```

với:

```text
Effective behavior
```

Đây là lý do functional verification bắt buộc.

---

# PHẦ XXVIII — POSITIVE VS NEGATIVE CONTROL

Hãy hình dung một requirement:

```text
opsuser được đọc report.
guest không được đọc report.
```

Một test:

```text
opsuser reads successfully
```

chỉ chứng minh half requirement.

Phải thêm:

```text
guest receives access denied
```

mới chứng minh security boundary.

Pattern này sẽ lặp lại sau này:

```text
Firewall:
allowed flow works
denied flow fails

IAM:
approved API works
unapproved API denied

BIG-IP:
healthy backend gets traffic
failed backend does not

Backup:
backup succeeds
restore succeeds
```

Day 1 là nơi bắt đầu tư duy verification hai chiều này.

---

# PHẦ XXIX — BASELINE → CHANGE → DRIFT

Một configuration management mental model:

```text
Approved Baseline
       |
       | approved change
       v
New Approved Baseline
```

Nếu có change ngoài control:

```text
Approved Baseline
       |
       | undocumented modification
       v
Configuration Drift
```

Khi troubleshoot, một câu hỏi rất có giá trị là:

> "What changed?"

Nhưng câu thứ hai phải là:

> "Was that change authorized?"

---

# PHẦ XXX — INCIDENT VS CHANGE

Hai concepts có thể xuất hiện cùng lúc nhưng khác nhau.

Incident:

```text
Something is broken/degraded.
```

Change:

```text
We intentionally modify something.
```

Ví dụ:

```text
Incident:
User cannot SSH.

Diagnosis:
authorized_keys ownership incorrect.

Change:
Correct ownership.

Verification:
Authorized SSH works.
Unauthorized key still denied.
```

Một incident có thể cần một approved change để resolve.

---

# PHẦ XXXI — WHY EVIDENCE BEFORE AND AFTER MATTERS

Giả sử bạn phát hiện:

```text
authorized_keys owner = root
```

thay vì intended user.

Bạn sửa ownership.

Nếu chỉ lưu evidence sau change:

```text
owner = alice
```

reviewer không biết:

- problem ban đầu thực sự là gì;
- bạn đã thay đổi gì;
- change có giải quyết root cause không.

Before/after evidence:

```text
Before:
root:root

Action:
approved ownership correction

After:
alice:alice

Functional test:
key authentication successful
```

tạo một operational narrative hoàn chỉnh.

---

# PHẦ XXXII — ESCALATION QUALITY

Một junior operator giỏi không phải người không bao giờ escalate.

Ngược lại, họ biết escalate **đúng lúc và có chất lượng**.

Poor escalation:

```text
"SSH not working."
```

Good escalation:

```text
Asset: rhel-app01
Environment: Production
Impact: Two administrators unable to establish SSH sessions
Started: 14:10 +07
Network connection: TCP connection established
Host key: Verified against inventory
Affected accounts: alice, bob
Authentication: Approved keys offered but rejected
File ownership: Read-only inspection shows unexpected state
No configuration changes performed
Recent change: CHGxxxx completed at 13:45
Evidence: attached sanitized SSH verbose output
Request: L2 review of server-side SSH authorization configuration
```

Người nhận escalation có thể tiếp tục ngay thay vì làm lại từ đầu.

---

# PHẦ XXXIII — NHỮNG THỨ CHƯA HỌC SÂU TRONG DAY 1

Để tránh scope creep, bạn nên biết rõ Day 1 **chưa yêu cầu mastery** các nội dung sau:

```text
Package management
systemd administration
Process signals
SELinux troubleshooting
firewalld configuration
Routing/DNS
Storage partitioning
/etc/fstab
journalctl deep analysis
syslog
logrotate
awk/sed deep usage
Bash control flow
cron
tar/rsync backup
Linux recovery
```

Chúng thuộc Day 2 theo syllabus. Infrastructure_Operations_14_Da…

Điều này rất quan trọng vì Day 1 phải tập trung làm nền thật chắc thay vì vội học quá rộng.

---

# PHẦ XXXIV — MỐI LIÊN HỆ DAY 1 VỚI CÁC DAY SAU

Day 1 không phải một bài Linux isolated. Nó tạo concepts được tái sử dụng toàn khóa.

`Least privilege` của Day 1 trở thành:

```text
AWS IAM — Day 4
Privileged access — Day 11
IAM review — Day 13
```

`Configuration baseline` trở thành:

```text
Nutanix baseline
Network configuration baseline
Firewall policies
Asset reconciliation
```

`Evidence` trở thành:

```text
CloudTrail
Event logs
Prism events
FortiGate logs
ServiceNow history
Audit packages
```

`Allowed/denied testing` trở thành:

```text
Security Groups
Firewall policies
BIG-IP
IAM permissions
```

`Runbook/escalation` xuất hiện xuyên suốt toàn bộ Infrastructure Operations.

---

# PHẦ XXXV — DAY 1 THEORY TROUBLESHOOTING FRAMEWORK

Khi gặp một Linux access problem, dùng framework:

```text
CONTEXT
│
├── Correct host?
├── Correct environment?
├── Correct account?
└── Authorized operation?
        ↓
IDENTITY
│
├── UID?
├── GID?
└── Group membership?
        ↓
OBJECT
│
├── Owner?
├── Group owner?
└── Permission mode?
        ↓
PATH
│
├── Parent permissions?
└── Can user traverse?
        ↓
PRIVILEGE
│
├── sudo required?
└── sudo authorized?
        ↓
REMOTE ACCESS
│
├── SSH connection?
├── Host verified?
├── Correct key?
└── Key authorized?
        ↓
VERIFY
│
├── Positive test
└── Negative test
        ↓
EVIDENCE
        ↓
RESOLVE / ROLLBACK / ESCALATE
```

Nếu bạn thực sự hiểu flow này, bạn đã đạt phần lớn learning objective của Module Lý thuyết Day 1.

---

# PHẦ XXXVI — MỨC ĐỘ HIỂU CẦN ĐẠT CHO TỪNG TOPIC

| Topic syllabus         | Sau khi học bạn phải làm được về mặt kiến thức                  |
| ---------------------- | --------------------------------------------------------------- |
| Infrastructure roles   | Giải thích nhiệm vụ Infrastructure Operations và accountability |
| L1/L2 boundaries       | Phân biệt authorized scope và escalation                        |
| Operational safety     | Áp dụng inspect-first, smallest change, rollback, evidence      |
| Prod/non-prod          | Giải thích vì sao cùng một action có risk khác nhau             |
| Change authorization   | Phân biệt technical privilege và organizational approval        |
| Least privilege        | Thiết kế permission đủ dùng, không quá rộng                     |
| SOP                    | Hiểu standard process                                           |
| Runbook                | Hiểu execution/troubleshooting instructions                     |
| Incident escalation    | Thu thập context và escalate có chất lượng                      |
| Access management      | Phân biệt authentication/authorization/access lifecycle         |
| Configuration baseline | Biết expected state và drift                                    |
| Asset inventory        | Xác định system chính xác                                       |
| Audit evidence         | Biết evidence nào có giá trị và cách bảo vệ secrets             |
| Lab readiness          | Biết prerequisites trước administration                         |
| Linux architecture     | Phân biệt kernel/user space/shell                               |
| Distribution/RHEL      | Hiểu distribution concept và enterprise context                 |
| Filesystem hierarchy   | Hiểu `/`, `/etc`, `/home`, `/var`, `/usr`...                    |
| CLI navigation         | Hiểu `pwd`, `ls`, `cd`                                          |
| Paths                  | Phân biệt absolute/relative, `.`, `..`, `~`                     |
| File operations        | Hiểu create/copy/move/delete và risk                            |
| Command syntax         | Hiểu command/options/arguments                                  |
| Pipes                  | Hiểu stdout → stdin chaining                                    |
| Redirection            | Hiểu stdout/stderr/input và overwrite/append                    |
| History                | Dùng và nhận biết credential risk                               |
| Help utilities         | Tự tra syntax đúng                                              |
| Safe execution         | Xác nhận target/scope/rollback/verification                     |
| Users/groups           | Hiểu UID/GID/primary/supplementary groups                       |
| Ownership              | Hiểu owner/group owner                                          |
| Permissions            | Hiểu `rwx` trên file và directory                               |
| `chmod`                | Đọc symbolic và octal mode                                      |
| `chown/chgrp`          | Hiểu ownership changes                                          |
| `umask`                | Tính effective creation permissions                             |
| `sudo`                 | Hiểu controlled privilege elevation                             |
| SSH                    | Hiểu secure remote access flow                                  |
| SSH authentication     | Phân biệt host/user authentication                              |
| SSH keys               | Phân biệt private/public key                                    |
| Secure admin access    | Kết hợp named identity + SSH + least privilege + audit          |

---

# PHẦ XXXVII — CHEAT SHEET LÝ THUYẾT DAY 1

## Identity

```bash
whoami
id
groups
getent passwd USER
getent group GROUP
```

Core concepts:

```text
UID
GID
Primary group
Supplementary groups
```

---

## System identity

```bash
hostname
uname -r
uname -a
cat /etc/os-release
```

---

## Navigation

```bash
pwd
ls
ls -l
ls -la
cd
cd ..
cd ~
```

Concepts:

```text
/
.
..
~
absolute path
relative path
```

---

## Files

```bash
touch
mkdir
cp
mv
rm
stat
```

Safety:

```text
Confirm target
Confirm current directory
Avoid uncontrolled recursive actions
Preserve before state
```

---

## Permissions

```bash
ls -l
chmod
chown
chgrp
umask
```

Mode:

```text
r = 4
w = 2
x = 1
```

Classes:

```text
u
g
o
```

---

## `umask`

Typical conceptual calculation:

```text
Files:       666 & ~umask
Directories: 777 & ~umask
```

Example:

```text
umask 022
file → 644
dir  → 755
```

---

## Privilege

```bash
sudo -l
visudo
```

Mental model:

```text
Named account
→ least privilege
→ sudo only when required
→ evidence
```

---

## SSH

```bash
ssh user@host
ssh -v user@host
ssh-keygen
```

Objects:

```text
Private key
Public key
Host key
known_hosts
authorized_keys
```

Typical trust flow:

```text
Verify server
→ Authenticate user
→ Establish session
→ Elevate only if authorized
```

---

## Evidence

Capture:

```text
Timestamp
Hostname
Username
Command
Output
Purpose
Result
Ticket/change reference
```

Never capture:

```text
Password
Private key
Token
Secret
```

---

# MODULE LÝ THUYẾT DAY 1 — COMPLETION CHECKLIST

Bạn có thể xem Module Lý thuyết Day 1 là hoàn thành khi bạn tự giải thích được toàn bộ chuỗi sau mà không cần học thuộc máy móc:

- Infrastructure Operations khác với việc đơn thuần biết technical commands như thế nào.
- Tại sao `L1/L2 boundary` là cả technical boundary lẫn authorization boundary.
- Tại sao phải xác nhận `hostname`, user, environment và scope trước một change.
- Khác biệt giữa production, non-production và lab.
- Khác biệt giữa technical privilege và change authorization.
- Least privilege được áp dụng vào user, group, permissions, `sudo` và SSH như thế nào.
- SOP và runbook khác nhau về mục tiêu ra sao.
- Khi nào nên troubleshoot, rollback hoặc escalate.
- Authentication và authorization khác nhau như thế nào.
- Configuration baseline và asset inventory khác nhau như thế nào.
- Một audit evidence tốt phải chứa context gì và không được chứa secrets gì.
- Linux kernel, shell, user space và distribution liên hệ với nhau ra sao.
- RHEL là gì trong Linux ecosystem.
- Linux filesystem hierarchy bắt đầu từ `/` và các directory quan trọng có vai trò gì.
- Absolute path và relative path được resolve như thế nào.
- `stdin`, `stdout`, `stderr`, pipe và redirection hoạt động ra sao.
- Vì sao `>` khác `>>`.
- Command history có giá trị và security risk gì.
- Cách tự sử dụng `man`, `--help` và `help`.
- UID/GID, primary group và supplementary groups hoạt động thế nào.
- `rwx` có ý nghĩa khác nhau trên regular file và directory như thế nào.
- Tại sao parent directory permissions có thể gây `Permission denied`.
- Cách đọc `chmod 750`, `chmod 640` và symbolic modes.
- `chown` và `chgrp` thay đổi metadata gì.
- `umask` thực chất là permission mask chứ không đơn giản là một "default permission".
- Tại sao named account + `sudo` tốt hơn shared root access về least privilege và auditability.
- Tại sao `sudoers` cần được kiểm soát và `visudo` quan trọng.
- SSH client/server flow hoạt động qua các phase nào.
- Host authentication và user authentication khác nhau như thế nào.
- Private key, public key, host key, `known_hosts` và `authorized_keys` hoàn toàn khác chức năng ra sao.
- Tại sao SSH host-key warning không được xử lý bằng cách xóa entry một cách máy móc.
- Tại sao phải test cả **allowed access** và **denied access**.
- Tại sao mọi configuration change phải đi cùng verification, evidence và rollback/escalation thinking.

Đó là toàn bộ **Module Lý thuyết Day 1** theo phạm vi `Concept/Lecture` của syllabus, được mở rộng tới mức operational foundation cần thiết nhưng chưa lấn sang nội dung chuyên sâu của Day 2. Infrastructure_Operations_14_Da…

Bước tiếp theo của Day 1 là **Module Thực hành**, nơi toàn bộ kiến thức trên sẽ được chuyển thành một lab Linux/RHEL có kiểm soát: chuẩn bị workstation và evidence repository → system inventory → CLI/filesystem → users/groups → permissions/`umask` → least-privilege `sudo` → SSH key authentication → positive/negative access tests → evidence package → cleanup/rollback.
