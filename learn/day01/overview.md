Dựa đúng vào `Infrastructure_Operations_14_Day_Syllabus.md`, **Day 1 thuộc phần “Orientation & Linux Administration Foundations”** và có hai trục chính: **operational foundation của Infrastructure Operations** và **Linux/RHEL administration foundation**. Ngoài phần Linux cơ bản, Day 1 còn đặt nền móng về **operational safety, access control, least privilege, evidence handling và audit traceability** để dùng xuyên suốt cả 14 ngày. Infrastructure_Operations_14_Da…

Tôi sẽ chia toàn bộ Day 1 thành đúng **2 module lớn: Lý thuyết và Thực hành**. Đây là bản **scope map chi tiết**; khi bạn yêu cầu từng module sau, tôi sẽ đi sâu theo hướng `What → Why → How → Verification → Troubleshooting → Evidence → Rollback → Escalation`.

# Day 1 — Tổng quan

Sau Day 1, mục tiêu không chỉ là “biết vài Linux commands”. Bạn cần hình thành tư duy của một Infrastructure Operations engineer: biết **mình được phép làm gì, trên environment nào, theo ticket/change nào, với privilege nào, phải kiểm tra gì trước/sau thao tác và phải lưu evidence gì**. Song song với đó, bạn phải có đủ Linux foundation để navigation, thao tác file, quản lý user/group/permissions, sử dụng `sudo`, thiết lập `SSH key` và chứng minh cả **allowed access** lẫn **denied access**. Infrastructure_Operations_14_Da…

Syllabus còn bổ sung một yêu cầu rất quan trọng: ngay Day 1 phải thiết lập **evidence handling baseline** dùng xuyên suốt chương trình. Mỗi evidence phải có khả năng trace về source, host/asset, environment, timestamp, ticket và integrity information; original evidence phải được giữ nguyên, còn working copy/derived copy phải được phân biệt rõ. Infrastructure_Operations_14_Da…

---

# Module 1 — LÝ THUYẾT

## Infrastructure Operations Foundation + Linux/RHEL Administration Foundation

Module này gồm **5 khối kiến thức lớn**.

## 1. Infrastructure Operations — vai trò, trách nhiệm và operational boundaries

Đây là phần định hình cách làm việc trước khi học command.

Bạn cần học:

- **Infrastructure administration roles and responsibilities**
  - Infrastructure administrator làm gì.
  - Phạm vi server, OS, access, configuration, monitoring, evidence.
  - Phân biệt operation với engineering/project work.
  - Ownership giữa server, network, security, application, cloud, vendor.

- **L1/L2 support boundaries**
  - L1 được phép verify, collect evidence, execute runbook nào.
  - L2 xử lý configuration/troubleshooting tới mức nào.
  - Khi nào phải escalate tới L3/vendor/security/application owner.
  - Không “trial-and-error” vượt authorization.

- **Operational safety**
  - Read-only inspection trước configuration change.
  - Xác nhận target host/environment.
  - Xác nhận impact.
  - Pre-check → execute → verify → post-check.
  - Stop condition.
  - Rollback thinking.

- **Production và non-production environment separation**
  - Production.
  - UAT/staging.
  - Development.
  - Lab/sandbox.
  - Không dùng production credentials/data/configuration một cách tùy tiện trong lab.

- **Change authorization**
  - Không phải mọi technical capability đều đồng nghĩa với authorization.
  - Ticket/change/approval.
  - Maintenance window.
  - Implementation plan.
  - Verification.
  - Backout/rollback.

- **Standard Operating Procedures — SOP**
  - Standardized procedure.
  - Preconditions.
  - Execution.
  - Expected result.
  - Escalation path.

- **Runbook**
  - Khác SOP ở điểm nào.
  - Runbook dành cho operation cụ thể.
  - Command sequence.
  - Decision points.
  - Stop conditions.

- **Incident escalation**
  - Symptom.
  - Impact.
  - Severity.
  - Evidence.
  - Escalation owner.
  - Không escalate chỉ bằng câu “server lỗi”.

Đây là một phần chính thức của Day 1, không phải kiến thức phụ. Infrastructure_Operations_14_Da…

---

## 2. Access Management, Security và Configuration Governance

Đây là khối security/operations quan trọng nhất của Day 1.

Bạn cần hiểu:

### Least privilege

User/service/admin chỉ được cấp đúng permission cần thiết.

Phải phân biệt:

- normal user;
- privileged user;
- `root`;
- `sudo`;
- service account;
- temporary elevated access.

Và hiểu tại sao không nên:

```bash
chmod 777
```

hoặc cấp:

```text
ALL=(ALL) ALL
```

cho mọi user chỉ để “cho chạy được”.

### Access management

Bạn cần hiểu lifecycle:

```text
Request
→ Approval
→ Provision
→ Verify
→ Use
→ Review
→ Revoke/Expire
→ Audit
```

### Configuration baseline

Một **configuration baseline** là trạng thái cấu hình chuẩn/đã được chấp thuận dùng để:

- compare;
- identify drift;
- troubleshoot;
- audit;
- rollback planning.

Ví dụ Day 1 có thể baseline:

- hostname;
- OS release;
- kernel;
- users/groups;
- permission;
- SSH configuration;
- sudo configuration.

### Asset inventory

Phải biết hệ thống mình đang quản lý là gì:

```text
Asset ID
Hostname
IP
OS
Version
Environment
Owner
Purpose
Management method
```

### Audit evidence

Mọi administrative activity quan trọng phải có evidence thích hợp:

- command;
- output;
- timestamp;
- hostname;
- ticket reference;
- configuration before/after;
- log;
- checksum/hash khi cần.

Các nội dung `access management`, `configuration baselines`, `asset inventory` và `audit evidence` đều được syllabus đặt ngay trong Day 1. Infrastructure_Operations_14_Da…

---

# 3. Linux Architecture và RHEL Foundation

Sau operational foundation mới đi vào Linux.

Bạn cần hiểu Linux theo các lớp:

```text
Hardware
   ↓
Linux Kernel
   ↓
System libraries / system services
   ↓
Shell / utilities
   ↓
Applications
   ↓
Users
```

### Linux architecture

Cần nhận biết vai trò của:

- hardware;
- kernel;
- userspace;
- shell;
- filesystem;
- processes;
- services;
- utilities.

Day 1 chưa đi sâu `systemd`, process troubleshooting, networking, storage — các phần đó thuộc Day 2.

### Kernel

Hiểu kernel chịu trách nhiệm chính về:

- CPU scheduling;
- memory management;
- device access;
- filesystem interface;
- process management;
- networking stack;
- system calls.

Các command kiểm tra cơ bản sẽ bao gồm dạng:

```bash
uname -a
uname -r
```

### Shell

Hiểu shell là command interpreter.

Cần nhận biết:

```text
bash
sh
zsh
```

và hiểu quan hệ:

```text
User
→ Shell
→ Command/program
→ Kernel
```

### Linux distributions

Phải hiểu Linux không phải một sản phẩm duy nhất.

Ví dụ:

- Red Hat Enterprise Linux — RHEL;
- Rocky Linux;
- AlmaLinux;
- Fedora;
- Ubuntu;
- Debian;
- SUSE.

Day 1 đặc biệt yêu cầu nhận thức về **RHEL và các Linux distributions khác**. Infrastructure_Operations_14_Da…

---

# 4. Linux Filesystem và CLI Foundation

Đây là phần command-line foundation rất lớn của Day 1.

## Filesystem hierarchy

Phải hiểu cấu trúc filesystem dạng:

```text
/
├── bin
├── boot
├── dev
├── etc
├── home
├── root
├── tmp
├── usr
└── var
```

Không chỉ nhớ directory mà phải biết vai trò operational của chúng.

Ví dụ:

- `/etc` — configuration;
- `/home` — normal-user home directories;
- `/root` — root user's home;
- `/var` — variable application/system data;
- `/tmp` — temporary files;
- `/usr` — user-space binaries/libraries/data.

---

## CLI navigation

Phải thành thạo:

```bash
pwd
ls
cd
```

và hiểu:

```text
.
..
~
/
```

---

## Absolute path và relative path

Ví dụ:

```bash
/etc/ssh/sshd_config
```

là absolute path.

Trong khi:

```bash
../logs/system.log
```

là relative path.

Đây là concept rất nhỏ nhưng cực kỳ quan trọng vì path sai có thể khiến admin:

- sửa nhầm file;
- xóa nhầm target;
- copy sai configuration;
- chạy command ở sai directory.

---

## File và directory operations

Phải hiểu và thực hành:

```bash
touch
mkdir
cp
mv
rm
rmdir
```

và các concepts:

- source;
- destination;
- overwrite;
- recursive operation;
- force;
- wildcard;
- current working directory.

Đặc biệt phải hiểu risk của `rm`.

---

## Command syntax

General pattern:

```bash
command [options] [arguments]
```

Ví dụ:

```bash
ls -la /etc
```

Trong đó:

```text
ls      command
-la     options
/etc    argument
```

---

## Pipes

Cần hiểu:

```text
command A stdout
        |
        v
command B stdin
```

Ví dụ:

```bash
cat file.txt | grep error
```

Sau này pipe sẽ trở thành foundation cho Day 2, logging và automation.

---

## Redirection

Phải hiểu:

```bash
>
>>
<
2>
2>&1
```

và các stream:

```text
stdin  = 0
stdout = 1
stderr = 2
```

Điểm operational quan trọng:

```bash
>
```

có thể overwrite file.

Trong khi:

```bash
>>
```

append.

---

## Command history

Phải biết:

```bash
history
```

và hiểu rằng shell history:

- có ích cho troubleshooting;
- có thể hỗ trợ evidence;
- nhưng **không được coi là audit mechanism hoàn chỉnh**;
- có thể chứa sensitive data nếu sử dụng command sai cách.

---

## Help utilities

Phải biết cách tự tìm hiểu command thay vì đoán syntax:

```bash
man
command --help
info
```

Ví dụ:

```bash
man chmod
chmod --help
```

---

## Safe command execution

Phải xây habit:

```text
1. Verify target
2. Understand command
3. Understand flags
4. Estimate impact
5. Prefer read-only check
6. Execute
7. Verify result
8. Capture evidence
```

Syllabus yêu cầu đầy đủ CLI navigation, path, file/directory operation, command syntax, pipes, redirection, history, help utilities và safe command execution. Infrastructure_Operations_14_Da…

---

# 5. Linux Identity, Permissions, sudo và SSH

Đây là security core của Day 1.

## Users

Phải hiểu user identity:

- username;
- UID;
- primary group;
- supplementary groups;
- home directory;
- login shell.

Một số files quan trọng sẽ được giới thiệu:

```text
/etc/passwd
/etc/shadow
/etc/group
```

Không có nghĩa là sẽ tùy tiện chỉnh trực tiếp các file này.

---

## Groups

Phải hiểu:

- primary group;
- supplementary groups;
- group-based access.

Ví dụ mô hình:

```text
User: alice
Groups:
  alice
  linuxops
  backup-readers
```

---

## Ownership

Mỗi filesystem object có:

```text
user owner
group owner
```

Kiểm tra bằng:

```bash
ls -l
```

---

## Linux permissions

Phải hiểu ba permission:

```text
r = read
w = write
x = execute
```

và ba đối tượng:

```text
u = user
g = group
o = others
```

Ví dụ:

```text
-rwxr-x---
```

phải có khả năng phân tích thành:

```text
owner : rwx
group : r-x
others: ---
```

---

## chmod

Phải hiểu cả symbolic mode và numeric mode.

Ví dụ:

```bash
chmod u+x script.sh
```

và:

```bash
chmod 750 script.sh
```

Cần hiểu logic:

```text
r = 4
w = 2
x = 1
```

---

## chown

Thay đổi owner:

```bash
chown user file
```

hoặc:

```bash
chown user:group file
```

---

## chgrp

Thay đổi group:

```bash
chgrp linuxops file
```

---

## umask

Đây là concept thường bị bỏ qua nhưng có trong syllabus.

Phải hiểu `umask` kiểm soát default permission khi tạo object mới.

Không được hiểu đơn giản rằng:

```text
umask = permission
```

mà nó là mask được áp dụng lên base permissions.

---

## sudo

Cần hiểu:

```text
sudo ≠ root account
```

`sudo` cho phép delegated privilege theo policy.

Phải học:

- least-privilege sudo;
- command-specific privilege;
- verification;
- denied sudo access;
- sudo auditability.

Ví dụ:

```bash
sudo -l
```

là một verification quan trọng.

---

# 6. SSH Authentication và Secure Administrative Access

Day 1 phải xây nền móng remote Linux administration.

## SSH

Phải hiểu các thành phần:

```text
SSH client
SSH server
TCP connection
Authentication
Session
```

### Authentication methods

Nhận thức về:

- password authentication;
- public-key authentication.

Trọng tâm lab sẽ là **SSH key authentication**.

---

## SSH key pair

Một pair bao gồm:

```text
private key
public key
```

Nguyên tắc rất quan trọng:

```text
Private key → phải bảo vệ
Public key  → có thể phân phối tới authorized server
```

Files thường gặp:

```text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
~/.ssh/authorized_keys
~/.ssh/known_hosts
```

---

## SSH key management

Cần hiểu:

- generate;
- distribute;
- permissions;
- identify correct key;
- revoke;
- replace;
- avoid exposing private keys.

---

## Secure administrative access

Mục tiêu không phải chỉ “SSH thành công”.

Bạn phải kiểm chứng:

```text
Who am I?
Which host?
Which authentication mechanism?
Which permissions?
What sudo capability?
Which operation is denied?
```

Vì lab yêu cầu chứng minh cả **authorized access và denied access**. Infrastructure_Operations_14_Da…

---

# 7. Evidence Handling và Operational Baseline

Đây là phần syllabus bổ sung nhưng **bắt buộc của Day 1**.

Bạn cần hiểu một evidence item nên có metadata như:

```text
Evidence ID
Source owner
Hostname / Asset ID
Environment
Collection time
Timezone
Query/window
Collector
Ticket reference
File size
Record count
SHA-256
Storage/access classification
```

Syllabus quy định rõ những trường này. Infrastructure_Operations_14_Da…

Phải phân biệt:

```text
Original / raw evidence
        ↓
Working copy
        ↓
Processed / redacted deliverable
```

### Nguyên tắc quan trọng

Original evidence:

```text
do not modify
```

Derived evidence:

```text
may be transformed
but must remain traceable to original
```

### SHA-256

Phải hiểu hash dùng để phát hiện thay đổi so với baseline đã ghi nhận.

Ví dụ:

```bash
sha256sum evidence.txt
```

Nhưng syllabus nhấn mạnh:

> Hash chỉ có thể giúp phát hiện thay đổi so với trusted recorded baseline; hash **không tự chứng minh source authenticity hoặc completeness**. Infrastructure_Operations_14_Da…

Đây là distinction rất quan trọng cho các Day sau về audit, ServiceNow, IAM, centralized logging và automation.

---

# Module 2 — THỰC HÀNH

## Day 1 Administrative & Linux Foundation Lab

Module thực hành sẽ gom toàn bộ Assignment/Lab của Day 1 thành một workflow thống nhất.

Syllabus yêu cầu bạn chuẩn bị environment, thực hiện Linux CLI/user/permission/sudo/SSH tasks và nộp system inventory, access verification, configuration evidence. Infrastructure_Operations_14_Da…

Module này sẽ gồm khoảng **10 lab phases**.

---

## Lab Phase 1 — Administrative Workstation Preparation

Chuẩn bị workstation được dùng để quản trị lab.

Cần xác nhận:

- terminal/SSH client;
- text editor;
- working directories;
- evidence directory;
- timestamp/timezone;
- access method;
- naming convention.

Một structure mẫu về sau có thể là:

```text
Day01/
├── evidence/
│   ├── raw/
│   ├── working/
│   └── deliverable/
├── commands/
├── screenshots/
├── notes/
└── manifest/
```

---

# Lab Phase 2 — Approved Access Checklist

Trước khi login hệ thống, lập checklist:

```text
Target hostname
Environment
Purpose
Approved account
Access level
Ticket/reference
Permitted actions
Forbidden actions
Escalation contact
```

Mục tiêu là xây thói quen:

```text
authorization before administration
```

---

# Lab Phase 3 — Linux/RHEL Environment Baseline

Thu thập system inventory.

Ví dụ nhóm thông tin:

```text
Hostname
OS
RHEL/Linux distribution
Kernel
Architecture
Current user
Groups
Home directory
Shell
IP/basic identity information
```

Typical commands có thể gồm:

```bash
hostname
uname -a
cat /etc/os-release
whoami
id
pwd
```

Sau này khi triển khai module chi tiết, từng command sẽ có:

- mục đích;
- expected output;
- evidence;
- failure interpretation.

---

# Lab Phase 4 — CLI Navigation

Thực hành:

```bash
pwd
ls
cd
```

với:

```text
absolute path
relative path
.
..
~
```

Mục tiêu không chỉ là navigation mà còn phải chứng minh bạn hiểu **current working directory** trước khi thao tác.

---

# Lab Phase 5 — File and Directory Operations

Thực hành có kiểm soát:

```bash
mkdir
touch
cp
mv
rm
```

Bao gồm:

```text
Create
Copy
Rename
Move
Delete
```

Phải verify trước và sau.

Ví dụ tư duy:

```text
Create
→ ls
→ copy
→ verify destination
→ move
→ verify
→ delete test object
→ verify absence
```

---

# Lab Phase 6 — Searching và Text Processing

Syllabus yêu cầu có:

> searching and text-processing exercises. Infrastructure_Operations_14_Da…

Do đó thực hành sẽ sử dụng những công cụ foundation phù hợp như:

```bash
find
grep
cat
head
tail
less
wc
sort
```

kết hợp:

```text
pipe
redirection
```

Ví dụ conceptual workflow:

```bash
command
  |
  v
filter
  |
  v
output file
```

Phần nâng cao `awk`, `sed`, regex được syllabus đặt trọng tâm ở Day 2, nên Day 1 sẽ chỉ xây foundation.

---

# Lab Phase 7 — User and Group Administration

Tạo các controlled lab accounts/groups.

Ví dụ:

```text
linuxops group

alice
bob
```

Sau đó verify:

```bash
id alice
getent passwd alice
getent group linuxops
```

Các thao tác phải trong **authorized lab**.

Mục tiêu:

- user creation;
- group membership;
- user/group lookup;
- separation of privileges.

---

# Lab Phase 8 — Ownership, Permissions và umask

Tạo test directory/file rồi thiết lập:

```text
owner
group
permissions
```

Thực hành:

```bash
chmod
chown
chgrp
```

Sau đó kiểm tra:

```bash
ls -l
stat
```

Lab không chỉ yêu cầu positive test mà cả negative test.

Ví dụ:

```text
alice → allowed
bob   → denied
```

Mục tiêu là chứng minh policy thực sự hoạt động.

---

# Lab Phase 9 — Least-Privilege sudo

Đây là một lab trọng tâm.

Thay vì:

```text
full administrative rights
```

ta thiết kế:

```text
User
  ↓
Specific sudo permission
  ↓
Allowed operation
```

và:

```text
Other privileged operation
  ↓
Denied
```

Verification sẽ gồm kiểu:

```bash
sudo -l
```

và thử:

```text
allowed command
denied command
```

Không cần phá hệ thống để test denied access.

---

# Lab Phase 10 — SSH Key Authentication

Workflow:

```text
Generate key pair
      ↓
Protect private key
      ↓
Install public key
      ↓
Configure permissions
      ↓
Connect
      ↓
Verify identity
      ↓
Test denied case
```

Command cơ bản sẽ xoay quanh:

```bash
ssh-keygen
ssh
```

cùng việc kiểm tra:

```text
~/.ssh/
authorized_keys
permissions
```

Khi vào module chi tiết, chúng ta sẽ troubleshooting các lỗi phổ biến như:

```text
Permission denied (publickey)
wrong username
wrong key
incorrect ~/.ssh permissions
incorrect authorized_keys permissions
wrong host
```

---

# Lab Phase 11 — Evidence Repository và Evidence Manifest

Day 1 yêu cầu tạo repository và shared evidence manifest dùng xuyên suốt 14 ngày. Infrastructure_Operations_14_Da…

Một evidence manifest sẽ track tối thiểu:

```text
Evidence ID
Host
Environment
Source
Timestamp
Timezone
Command/query
Ticket
Filename
Size
Record count
SHA-256
Classification
Notes
```

Ví dụ conceptual record:

```text
EVID-D01-001
Host: rhel-lab01
Environment: LAB
Source: /etc/os-release
Collected: 2026-10-20T...
Collector: <learner>
Ticket: LAB-D01
SHA-256: ...
```

---

# Lab Phase 12 — Evidence Integrity Exercise

Đây là bài thực hành đặc biệt được syllabus yêu cầu.

Workflow:

```text
Original evidence
      ↓
SHA-256 hash
      ↓
Copy → working copy
      ↓
Modify working copy
      ↓
Hash again
      ↓
Detect mismatch
      ↓
Record transformation
```

Mục đích là chứng minh:

```text
original unchanged
modified copy != original hash
```

Acceptance criteria của syllabus là reviewer phải trace được evidence về source và ticket; original phải còn nguyên và modified copy phải fail comparison với original hash. Infrastructure_Operations_14_Da…

---

# Deliverables cuối Day 1

Nếu hoàn thành đúng syllabus, cuối Day 1 bạn phải có một bộ artifacts gần như sau:

1. **Administrative workstation checklist**
2. **Approved access checklist**
3. **Linux/RHEL system inventory**
4. **CLI command transcript**
5. **File/directory exercise evidence**
6. **Search/text-processing evidence**
7. **User/group configuration evidence**
8. **Ownership/permission/umask evidence**
9. **Allowed/denied permission tests**
10. **Least-privilege sudo configuration + verification**
11. **SSH key configuration**
12. **SSH successful-access evidence**
13. **SSH/access denied evidence**
14. **Evidence repository**
15. **Evidence manifest**
16. **Original + working-copy hash integrity test**
17. **Personal learning baseline**

Tất cả các nhóm deliverable này đều xuất phát từ Assignment/Lab và additive evidence requirements của Day 1. Infrastructure_Operations_14_Da…

---

# Bản đồ toàn bộ Day 1

Có thể hình dung Day 1 theo dependency sau:

```text
DAY 1
│
├── MODULE 1 — LÝ THUYẾT
│
│   ├── Infrastructure Operations
│   │   ├── Roles & responsibilities
│   │   ├── L1/L2 boundaries
│   │   ├── Operational safety
│   │   ├── Prod / Non-prod separation
│   │   ├── Change authorization
│   │   ├── SOP / Runbook
│   │   └── Incident escalation
│   │
│   ├── Governance & Security
│   │   ├── Access management
│   │   ├── Least privilege
│   │   ├── Configuration baseline
│   │   ├── Asset inventory
│   │   └── Audit evidence
│   │
│   ├── Linux Foundation
│   │   ├── Architecture
│   │   ├── Kernel
│   │   ├── Shell
│   │   ├── Distribution
│   │   └── RHEL
│   │
│   ├── CLI & Filesystem
│   │   ├── Filesystem hierarchy
│   │   ├── Navigation
│   │   ├── Absolute/relative paths
│   │   ├── File/directory operations
│   │   ├── Command syntax
│   │   ├── Pipes
│   │   ├── Redirection
│   │   ├── History
│   │   ├── Help
│   │   └── Safe execution
│   │
│   ├── Identity & Permissions
│   │   ├── Users
│   │   ├── Groups
│   │   ├── Ownership
│   │   ├── Permissions
│   │   ├── chmod
│   │   ├── chown
│   │   ├── chgrp
│   │   ├── umask
│   │   └── sudo
│   │
│   ├── SSH
│   │   ├── Authentication
│   │   ├── Key pair
│   │   ├── authorized_keys
│   │   └── Secure admin access
│   │
│   └── Evidence Handling
│       ├── Evidence metadata
│       ├── Raw vs working copy
│       ├── SHA-256
│       ├── Traceability
│       └── Evidence limitations
│
└── MODULE 2 — THỰC HÀNH
    │
    ├── Workstation preparation
    ├── Access checklist
    ├── RHEL/Linux environment setup
    ├── System inventory
    ├── CLI navigation
    ├── File operations
    ├── Searching/text processing
    ├── User/group management
    ├── Ownership/permissions
    ├── umask
    ├── Least-privilege sudo
    ├── SSH key authentication
    ├── Allowed/denied tests
    ├── Evidence repository
    ├── Evidence manifest
    ├── SHA-256 integrity exercise
    └── Final Day-1 evidence package
```

Có một boundary cần nhớ: **Day 1 dừng ở Linux administration foundation**. Các chủ đề như `dnf/yum/rpm`, `systemd`, processes/signals, `SELinux`, `firewalld`, IP/routing/DNS, disk/mount, `journalctl`, advanced `grep/awk/sed`, Bash scripting, `cron`, backup/restore và operational troubleshooting sâu được syllabus chuyển sang **Day 2**, vì vậy chúng ta chưa trộn chúng vào nội dung chính của Day 1. Infrastructure_Operations_14_Da…

Khi bạn nói **“Trình bày Module Lý thuyết Day 1”**, tôi sẽ bắt đầu triển khai toàn bộ **Module 1** ở mức rất sâu, lần lượt từng topic với `What / Why / How / operational workflow / security / common failures / verification / troubleshooting / evidence / escalation / commands / scenarios`, không bỏ sót bất kỳ mục nào trong syllabus.
