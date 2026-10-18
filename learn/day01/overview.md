# Day 1 — Bản đồ nội dung học theo 2 Module

Theo syllabus, **Day 1 thuộc “Orientation & Linux Administration Foundations”** và có hai mục tiêu song song: xây dựng tư duy vận hành hạ tầng an toàn trong môi trường enterprise, đồng thời hình thành nền tảng Linux/RHEL đủ để thao tác bằng CLI, quản lý access, permissions, `sudo` và SSH một cách có kiểm soát. Infrastructure_Operations_14_Da…

Day 1 chưa đi sâu vào `systemd`, package management, networking, storage, logs hay Bash automation; các nội dung đó chủ yếu thuộc Day 2. Vì vậy Day 1 nên được xem là **foundation bắt buộc** trước khi bước sang Linux operations thực thụ. Infrastructure_Operations_14_Da…

Dưới đây là toàn bộ Day 1 được tổ chức lại thành đúng **2 Module: Lý thuyết và Thực hành**. Đây mới là bản đồ nội dung; khi bạn yêu cầu từng Module, tôi sẽ giảng sâu theo `What → Why → How → Verification → Troubleshooting → Evidence → Rollback → Escalation`.

---

# MODULE 1 — LÝ THUYẾT

Module Lý thuyết bao phủ toàn bộ phần `Concept/Lecture` của Day 1. Infrastructure_Operations_14_Da…

## Phần 1 — Infrastructure Operations Foundation

Đây là phần giúp bạn hiểu **một Infrastructure Operations engineer thực sự làm gì**, được phép làm gì và phải kiểm soát rủi ro như thế nào trước khi đụng tới Linux hay bất kỳ production system nào.

### 1. Infrastructure administration roles and responsibilities

Cần hiểu vai trò của Infrastructure Operations trong vòng đời hệ thống:

- provisioning;
- configuration;
- administration;
- monitoring;
- maintenance;
- troubleshooting;
- backup/recovery;
- security compliance;
- access control;
- patching;
- evidence collection;
- incident handling;
- change execution;
- escalation;
- operational handover.

Bạn cần phân biệt việc “biết command” với việc “được phép thực thi command”.

Trọng tâm tư duy:

**Need → Authorization → Change → Execute → Verify → Evidence → Close/Escalate.**

---

## 2. L1/L2 support boundaries

Hiểu ranh giới giữa các tầng support.

### L1

Thông thường tập trung vào:

- monitoring;
- health checks;
- standard verification;
- basic diagnosis;
- log collection;
- executing approved runbook;
- collecting evidence;
- ticket updates;
- escalation.

### L2

Có thể xử lý sâu hơn:

- configuration diagnosis;
- permission problems;
- service/application dependencies;
- approved configuration changes;
- advanced Linux troubleshooting;
- root-cause isolation;
- recovery actions trong phạm vi được phân quyền.

Điểm quan trọng không phải chỉ là “L1 biết ít hơn L2”, mà là:

> **support boundary là authorization boundary.**

Bạn phải biết khi nào nên dừng troubleshooting và escalate.

---

# Phần 2 — Operational Safety & Governance

## 3. Operational safety

Học cách tránh biến một lỗi nhỏ thành outage lớn.

Các nguyên tắc nền tảng:

- inspect trước khi modify;
- prefer read-only verification;
- xác nhận target host;
- xác nhận environment;
- xác nhận user/context;
- kiểm tra impact;
- không chạy command chưa hiểu;
- tránh wildcard/destructive operations không kiểm soát;
- backup/config snapshot khi cần;
- chuẩn bị rollback;
- log lại thay đổi;
- verify sau khi thao tác.

Một thói quen rất quan trọng:

```bash
whoami
hostname
pwd
```

trước những thao tác nhạy cảm.

---

## 4. Production và non-production environment separation

Bạn phải hiểu sự khác biệt giữa:

- Development;
- Test;
- UAT;
- Staging;
- Production;
- DR environment;
- Lab.

Các environment khác nhau về:

- dữ liệu;
- quyền access;
- change process;
- maintenance window;
- monitoring;
- security control;
- business impact;
- rollback expectation.

Day 1 yêu cầu hình thành thói quen:

> Không bao giờ giả định system đang thao tác là lab hoặc non-production.

---

## 5. Change authorization

Học nguyên tắc:

**Không phải mọi technical change hợp lý đều được phép thực hiện.**

Trước change có thể cần:

- ticket/change record;
- scope;
- owner;
- impact assessment;
- risk;
- approval;
- implementation plan;
- verification plan;
- rollback/backout plan;
- maintenance window.

Bạn cũng phải phân biệt:

- diagnostic action;
- operational action;
- configuration-changing action.

---

## 6. Least-privilege principles

Một trong những concept quan trọng nhất của cả khóa.

Một user/process chỉ nên có:

> **quyền tối thiểu cần thiết để hoàn thành nhiệm vụ được phê duyệt.**

Áp dụng vào:

- Linux users;
- Linux groups;
- filesystem permissions;
- `sudo`;
- SSH;
- service accounts;
- administrator/root access.

Ví dụ:

Không cấp:

```text
ALL=(ALL) ALL
```

nếu người vận hành chỉ cần restart một service cụ thể.

---

# Phần 3 — SOP, Runbook, Incident và Evidence

## 7. Standard Operating Procedures — SOP

Hiểu SOP dùng để chuẩn hóa công việc định kỳ.

Một SOP thường mô tả:

- objective;
- scope;
- prerequisites;
- responsible role;
- approved steps;
- validation;
- evidence;
- rollback;
- escalation.

---

## 8. Runbooks

Runbook thường cụ thể hơn SOP và thiên về execution/troubleshooting.

Ví dụ:

```text
Runbook: Linux SSH Login Failure
```

có thể hướng dẫn:

```text
1. Verify host reachability
2. Verify target hostname
3. Verify username
4. Verify SSH client
5. Verify authentication method
6. Collect error message
7. Check authorized_keys
8. Verify file ownership/permissions
9. Escalate if sshd configuration must be changed
```

Bạn sẽ được học cách sử dụng runbook thay vì troubleshooting tùy hứng.

---

## 9. Incident escalation

Bạn phải hiểu:

- incident là gì;
- symptom khác root cause;
- evidence nào cần thu thập;
- escalation phải chứa thông tin gì.

Một escalation tốt phải trả lời được:

```text
What happened?
When?
Which system?
Who/what is affected?
What was checked?
What evidence exists?
What actions were taken?
What changed?
What remains unresolved?
```

---

## 10. Access management

Bao gồm tư duy quản lý:

- account request;
- account creation;
- entitlement;
- authentication;
- authorization;
- privileged access;
- access verification;
- access expiration/revocation;
- audit trail.

Đây là nền tảng trực tiếp cho:

- Linux user/group;
- `sudo`;
- SSH;
- AWS IAM sau này;
- privileged-access governance ở các Day sau.

---

## 11. Configuration baselines

Bạn cần hiểu **baseline** là trạng thái cấu hình chuẩn dùng làm điểm so sánh.

Ví dụ Linux baseline có thể gồm:

```text
Hostname
OS
Kernel version
Users
Groups
Sudo configuration
SSH configuration
Filesystem permissions
Network configuration
Installed packages
Running services
```

Day 1 mới tập trung baseline ở mức foundation.

---

## 12. Asset inventory

Bạn phải biết một infrastructure asset nên được nhận diện bằng dữ liệu như:

```text
Hostname
IP address
OS
Distribution
Version
Kernel
CPU
Memory
Disk
Environment
Owner
Role
Asset ID
```

Điều này liên quan trực tiếp tới asset management và CMDB ở các Day sau.

---

## 13. Audit evidence

Học cách chứng minh:

> “Tôi đã thực hiện đúng operation và kết quả đúng như yêu cầu.”

Evidence có thể gồm:

- command;
- command output;
- log;
- screenshot;
- timestamp;
- hostname;
- username;
- ticket/change ID;
- config file;
- before/after state.

Một screenshot không có hostname/timestamp/context thường là evidence yếu.

---

## 14. Lab readiness

Trước lab phải kiểm tra:

- administrative workstation;
- terminal/SSH client;
- approved account;
- Linux/RHEL VM;
- network access;
- privilege level;
- evidence folder;
- safe test data;
- rollback capability.

Đây chính là bridge từ governance sang hands-on Linux.

---

# Phần 4 — Linux Architecture Foundation

Sau Infrastructure Operations foundation, Day 1 chuyển sang Linux.

## 15. Linux architecture

Bạn cần hình dung hệ thống theo các layer cơ bản:

```text
User
 ↓
Applications / Utilities
 ↓
Shell
 ↓
System libraries
 ↓
Kernel
 ↓
Hardware
```

Cần hiểu mối liên hệ giữa:

- hardware;
- kernel;
- shell;
- userspace;
- processes;
- filesystem.

---

## 16. Kernel

`kernel` là core của operating system.

Bạn cần nắm ở mức foundation:

- process scheduling;
- memory management;
- device management;
- filesystem interaction;
- networking;
- security/access control.

Một số command kiểm tra:

```bash
uname
uname -r
uname -a
```

Day 2 mới đi sâu hơn vào process/service/resource operations.

---

## 17. Shell

Phân biệt:

**Terminal ≠ Shell ≠ Kernel.**

Shell là command interpreter giữa user và operating system.

Ví dụ:

```text
bash
sh
zsh
```

Day 1 chủ yếu sử dụng Bash-compatible CLI.

---

## 18. Linux distributions

Hiểu Linux kernel được đóng gói thành các distributions khác nhau.

Ví dụ:

- Red Hat Enterprise Linux;
- Rocky Linux;
- AlmaLinux;
- Ubuntu;
- Debian;
- SUSE Linux Enterprise Server.

Cần phân biệt:

```text
Linux
Linux kernel
Linux distribution
RHEL
```

---

## 19. Red Hat Enterprise Linux — RHEL

RHEL được nhấn mạnh vì syllabus hướng tới enterprise operations.

Cần hiểu ở mức tổng quan:

- enterprise Linux distribution;
- lifecycle/support;
- package ecosystem;
- enterprise tooling;
- security focus;
- compatibility/certification.

Day 2 mới đi sâu `dnf`, `yum`, `rpm`, repository và subscription awareness.

---

# Phần 5 — Linux Filesystem & CLI

## 20. Filesystem hierarchy

Bạn phải hiểu Linux filesystem bắt đầu từ:

```text
/
```

Các directory quan trọng ban đầu gồm:

```text
/
/home
/root
/etc
/var
/tmp
/usr
/bin
/sbin
/dev
/proc
```

Không cần thuộc máy móc; cần hiểu mục đích operational của từng vùng.

---

## 21. CLI navigation

Các command nền tảng:

```bash
pwd
ls
cd
```

Sau đó kết hợp:

```bash
ls -l
ls -a
ls -la
cd ..
cd ~
```

Bạn phải luôn biết:

> mình đang đứng ở đâu và command sẽ tác động vào đâu.

---

## 22. Absolute và relative paths

### Absolute path

Bắt đầu từ `/`.

```bash
/etc/ssh/sshd_config
```

### Relative path

Phụ thuộc `current working directory`.

```bash
./file.txt
../backup/
```

Sai path là nguyên nhân phổ biến của operational mistakes.

---

## 23. File và directory operations

Cần hiểu:

```bash
touch
mkdir
cp
mv
rm
rmdir
```

và hành vi của chúng với:

- file;
- directory;
- recursive operation;
- overwrite;
- deletion.

Đặc biệt phải hiểu rủi ro của:

```bash
rm
rm -r
rm -f
rm -rf
```

chứ không chỉ học syntax.

---

# Phần 6 — Linux Command Processing

## 24. Command syntax

Mẫu tổng quát:

```text
command [options] [arguments]
```

Ví dụ:

```bash
ls -l /etc
```

Trong đó:

```text
ls     command
-l     option
/etc   argument
```

---

## 25. Pipes

Pipe:

```bash
|
```

chuyển `stdout` của command trước sang `stdin` của command sau.

Ví dụ:

```bash
ls -l /etc | grep ssh
```

Đây là concept rất quan trọng vì Linux administration phụ thuộc mạnh vào command composition.

---

## 26. Redirection

Cần phân biệt:

```bash
>
>>
<
2>
```

Ví dụ:

```bash
command > output.txt
```

và:

```bash
command >> output.txt
```

khác nhau rất lớn vì `>` có thể overwrite file.

---

## 27. Command history

Các khái niệm:

```bash
history
```

và command recall.

History giúp:

- tăng productivity;
- reproduce troubleshooting;
- tìm command vừa chạy.

Nhưng cũng có security implications nếu command chứa sensitive information.

---

## 28. Help utilities

Bạn không cần nhớ mọi option.

Bạn cần biết cách tự tìm:

```bash
man
--help
help
```

Ví dụ:

```bash
man chmod
chmod --help
```

Đây là kỹ năng quan trọng hơn việc học thuộc command.

---

## 29. Safe command execution

Trước command thay đổi hệ thống:

```text
Understand command
↓
Confirm target
↓
Check current state
↓
Assess impact
↓
Check privilege
↓
Execute
↓
Verify
↓
Record evidence
```

Đây sẽ là operational pattern xuyên suốt 14 Day.

---

# Phần 7 — Linux Identity & Permissions

## 30. Users và groups

Bạn cần hiểu Linux identity model:

```text
user
UID
primary group
supplementary groups
GID
```

Các command cần làm quen:

```bash
id
whoami
groups
```

và sau này trong lab:

```bash
useradd
usermod
userdel
groupadd
groupdel
```

---

## 31. Ownership

Mỗi file thường có:

```text
user owner
group owner
```

Kiểm tra bằng:

```bash
ls -l
```

---

## 32. File permissions

Ba permission cơ bản:

```text
r = read
w = write
x = execute
```

áp dụng cho:

```text
user
group
other
```

Ví dụ:

```text
-rwxr-x---
```

Bạn phải đọc được permission string này.

---

## 33. `chmod`

Dùng thay đổi file permissions.

Hai kiểu chính:

### Symbolic

```bash
chmod u+x file
```

### Numeric/octal

```bash
chmod 750 file
```

Bạn phải hiểu ý nghĩa từng bit, không học thuộc số permission.

---

## 34. `chown`

Dùng thay đổi owner.

```bash
chown user file
```

hoặc:

```bash
chown user:group file
```

---

## 35. `chgrp`

Thay đổi group ownership:

```bash
chgrp group file
```

---

## 36. `umask`

`umask` quyết định permission mặc định bị loại bỏ khi tạo:

- files;
- directories.

Cần hiểu quan hệ giữa:

```text
default permission
-
umask
=
effective creation permission
```

Đây là một topic thường gây nhầm lẫn và sẽ cần học kỹ.

---

# Phần 8 — Privileged Administration

## 37. `sudo`

`sudo` cho phép thực hiện command với elevated privilege trong phạm vi được cấu hình.

Cần hiểu khác biệt giữa:

```text
normal user
sudo
root
```

và tại sao enterprise operations ưu tiên:

```text
named account + sudo
```

thay vì dùng chung `root`.

---

## 38. Least-privilege sudo

Không chỉ học:

```bash
sudo command
```

mà phải hiểu cách thiết kế privilege giới hạn.

Ví dụ conceptual:

```text
operator chỉ được phép restart một service cụ thể
```

thay vì được toàn quyền root.

Cũng phải hiểu:

```text
/etc/sudoers
/etc/sudoers.d/
visudo
```

và tại sao không chỉnh `sudoers` tùy tiện bằng text editor.

---

# Phần 9 — Secure Administrative Access với SSH

## 39. SSH authentication

Hiểu flow cơ bản:

```text
SSH client
     |
     | TCP connection
     v
SSH server
     |
Authentication
     |
Authorization
     |
Shell/session
```

Các authentication method có thể gồm:

- password;
- public-key authentication.

Day 1 tập trung vào secure administrative access.

---

## 40. SSH key management

Bạn phải hiểu:

```text
private key
public key
authorized_keys
```

Quan hệ:

```text
Client private key
        ↓
authentication
        ↓
Server authorized public key
```

Private key phải được bảo vệ tuyệt đối.

---

## 41. SSH key generation

Command điển hình:

```bash
ssh-keygen
```

Cần hiểu:

- algorithm;
- key pair;
- passphrase;
- output files;
- permissions.

Không được chia sẻ private key thật trong lab evidence.

---

## 42. `authorized_keys`

Public key hợp lệ được đặt tại:

```bash
~/.ssh/authorized_keys
```

Cần hiểu importance của:

```text
ownership
directory permission
file permission
```

Sai permission có thể dẫn đến SSH authentication failure.

---

## 43. Secure administrative access

Không chỉ là “SSH vào được”.

Phải đảm bảo:

- đúng account;
- approved source;
- approved authentication;
- least privilege;
- protected key;
- permission đúng;
- access được audit;
- unauthorized access bị deny.

Đây là kết thúc logic của Module Lý thuyết Day 1.

---

# MODULE 2 — THỰC HÀNH

Module này ánh xạ trực tiếp toàn bộ `Assignment/Lab` của Day 1: chuẩn bị workstation và Linux/RHEL lab, thực hành CLI và text processing, quản lý users/groups/permissions, cấu hình least-privilege `sudo`, thiết lập SSH key authentication, kiểm tra allow/deny access, ghi lại command và nộp inventory/access/configuration evidence. Infrastructure_Operations_14_Da…

Module Thực hành nên được triển khai theo chuỗi dưới đây.

---

# Lab 1 — Administrative Workstation Preparation

## Mục tiêu

Chuẩn bị workstation dùng để quản trị.

Cần xác nhận:

```text
Terminal available
SSH client available
Evidence directory available
Lab credentials ready
Linux VM reachable
No production target involved
```

Ví dụ evidence repository:

```text
Day01/
├── 01_inventory/
├── 02_cli/
├── 03_users_groups/
├── 04_permissions/
├── 05_sudo/
├── 06_ssh/
└── 07_final_evidence/
```

---

# Lab 2 — Approved Access Checklist

Bạn sẽ tạo checklist xác minh trước khi thao tác:

```text
[ ] Target hostname confirmed
[ ] Environment confirmed
[ ] Authorized account confirmed
[ ] Access approved
[ ] Required privilege identified
[ ] Evidence location prepared
[ ] Rollback understood
[ ] No real credential recorded
```

Mục đích là luyện operational discipline ngay từ Day 1.

---

# Lab 3 — Linux/RHEL Lab Environment Readiness

Bạn sẽ kiểm tra:

```bash
whoami
id
hostname
uname -a
cat /etc/os-release
pwd
```

Từ đó xác định:

```text
Current user
UID/GID
Hostname
Kernel
Linux distribution
RHEL/version nếu có
Current working directory
```

Đây cũng là bước đầu tiên của system inventory.

---

# Lab 4 — CLI Navigation

Thực hành:

```bash
pwd
ls
ls -l
ls -la
cd
cd ..
cd ~
```

Bạn phải chứng minh được:

- current directory;
- parent directory;
- home directory;
- hidden files;
- long listing.

---

# Lab 5 — File và Directory Operations

Thực hành tạo controlled workspace.

Ví dụ:

```text
day1-lab/
├── input/
├── working/
├── output/
└── backup/
```

Sau đó dùng:

```bash
mkdir
touch
cp
mv
rm
rmdir
```

Mỗi command phải có:

```text
Before state
Command
Expected result
Verification
Evidence
```

---

# Lab 6 — Absolute/Relative Path Exercises

Bạn sẽ thực hiện cùng một operation bằng:

```text
absolute path
```

và:

```text
relative path
```

để hiểu rõ path resolution.

Ví dụ concept:

```bash
/home/student/day1-lab/input/file.txt
```

so với:

```bash
../input/file.txt
```

---

# Lab 7 — Searching Exercises

Syllabus yêu cầu thực hiện searching.

Bạn sẽ tìm:

- file;
- directory;
- filename patterns;
- nội dung trong file.

Các công cụ phù hợp có thể gồm:

```bash
find
grep
```

Day 1 dùng ở foundation level; Day 2 mới đi sâu hơn vào `grep/awk/sed` và regular expressions. Infrastructure_Operations_14_Da…

---

# Lab 8 — Text-Processing Exercises

Bạn sẽ thực hành processing output bằng những công cụ CLI cơ bản, đặc biệt:

```text
pipes
redirection
search/filter
```

Ví dụ pattern:

```bash
command | grep pattern
```

hoặc:

```bash
command > evidence.txt
```

Mục tiêu là hiểu data flow qua CLI.

---

# Lab 9 — Users Management

Trong controlled lab, bạn sẽ:

```text
Create user
Inspect user
Modify user
Verify user
Optionally remove lab user during cleanup
```

Các command sẽ xoay quanh:

```bash
useradd
id
usermod
userdel
```

Tất cả chỉ chạy trong authorized lab.

---

# Lab 10 — Groups Management

Thực hành:

```bash
groupadd
usermod
groups
id
```

Mục tiêu:

- tạo group;
- thêm user vào group;
- verify membership;
- hiểu primary/supplementary group.

---

# Lab 11 — Ownership Configuration

Bạn sẽ tạo file với owner/group khác nhau và sử dụng:

```bash
chown
chgrp
ls -l
```

để quan sát ownership change.

Phải lưu evidence trước và sau thay đổi.

---

# Lab 12 — File Permission Configuration

Thực hành cả hai kiểu:

```text
symbolic permission
octal permission
```

Ví dụ:

```bash
chmod u+x script.sh
chmod 640 report.txt
chmod 750 directory
```

Sau đó verify:

```bash
ls -l
```

Nhưng quan trọng hơn là thực hiện functional test:

```text
User A được read?
User B được write?
Unauthorized user bị deny?
```

---

# Lab 13 — `umask`

Bạn sẽ:

1. kiểm tra current `umask`;
2. tạo file/directory;
3. quan sát default permissions;
4. thay đổi `umask` trong lab;
5. tạo objects mới;
6. so sánh kết quả.

Mục tiêu là chứng minh bạn hiểu default permission creation.

---

# Lab 14 — Least-Privilege `sudo`

Đây là một trong các lab quan trọng nhất.

Scenario điển hình:

```text
opsuser cần thực hiện một administrative task cụ thể
nhưng không được cấp toàn quyền root.
```

Bạn sẽ:

```text
Create controlled user
Identify required privileged operation
Configure limited sudo
Verify permitted command
Verify prohibited command
Collect evidence
```

Cần kiểm tra config bằng phương thức an toàn như `visudo`.

---

# Lab 15 — SSH Key Generation

Bạn sẽ tạo key pair trong lab bằng:

```bash
ssh-keygen
```

Sau đó xác định:

```text
private key
public key
fingerprint
```

và bảo đảm private key không bị đưa vào evidence submission.

---

# Lab 16 — SSH Public-Key Authentication

Bạn sẽ cấu hình public key cho controlled account thông qua:

```text
~/.ssh/
~/.ssh/authorized_keys
```

Sau đó verify login.

Data flow cần hiểu:

```text
Administrative workstation
        |
        | SSH private key
        v
Linux/RHEL server
        |
        | public key validation
        v
Authorized account session
```

---

# Lab 17 — Authorized Access Verification

Bạn phải chứng minh legitimate user:

```text
can authenticate
```

và sau login:

```bash
whoami
hostname
id
```

để xác nhận identity và target.

---

# Lab 18 — Denied Access Verification

Một điểm rất quan trọng trong syllabus là không chỉ chứng minh “allowed works” mà còn phải chứng minh:

> **unauthorized access is denied.**

Có thể kiểm thử controlled scenarios như:

```text
Wrong user
User không nằm đúng group
User thiếu filesystem permission
User không có sudo permission
SSH key không được authorize
```

Mỗi case phải có expected denial.

---

# Lab 19 — Command Documentation

Syllabus yêu cầu document các command đã thực hiện.

Một command log tốt nên chứa:

```text
Timestamp
Hostname
Current user
Command
Purpose
Expected result
Actual result
Verification
Status
```

Không được đưa:

```text
password
private key
token
secret
```

vào evidence.

---

# Lab 20 — System Inventory Submission

Bạn phải tạo một inventory tối thiểu.

Ví dụ:

```text
Hostname
Linux distribution
OS version
Kernel
Current user
UID/GID
CPU
Memory
Filesystem overview
Lab IP
SSH access status
```

Đây là một trong các deliverable chính của Day 1.

---

# Lab 21 — Access Verification Evidence

Evidence package phải chứng minh ít nhất:

```text
Approved account exists
Correct group membership
Correct ownership
Correct permissions
Approved sudo works
Unapproved sudo is denied
SSH key authentication works
Unauthorized access is denied
```

---

# Lab 22 — Configuration Evidence

Cuối lab, bạn phải có bằng chứng cho các cấu hình đã thực hiện:

```text
Users/groups
Ownership
Permissions
umask
sudo
SSH
authorized_keys
System inventory
```

Kèm:

```text
timestamp
hostname
command/output
```

đủ để người khác review.

---

# Lab 23 — Cleanup / Rollback

Một lab Infrastructure Operations không hoàn thành chỉ vì “test thành công”.

Bạn còn phải:

- remove temporary test data nếu được yêu cầu;
- revoke temporary privileges;
- remove temporary lab accounts nếu scope yêu cầu;
- remove unused SSH authorization;
- restore modified configuration;
- verify final state;
- preserve evidence.

Tư duy cần hình thành:

```text
Change
→ Verify
→ Test
→ Evidence
→ Cleanup / Rollback
→ Final Verification
```

---

# Lab 24 — Personal Learning Baseline

Một deliverable đặc biệt trong Day 1 là **personal learning baseline**. Infrastructure_Operations_14_Da…

Mục đích là xác định trước khi bước vào chương trình bạn đang ở mức nào đối với:

```text
Linux CLI
Filesystem
Users/groups
Permissions
sudo
SSH
Operational safety
Troubleshooting
Evidence collection
```

Từ đó cuối chương trình có thể so sánh mức tiến bộ.

---

# Kết quả cần đạt sau Day 1

Nếu hoàn thành đúng Day 1, bạn chưa cần trở thành Linux administrator nâng cao. Syllabus xác định toàn bộ chương trình là **accelerated foundation và operational readiness**, không phải advanced product mastery. Infrastructure_Operations_14_Da…

Bạn cần đạt được 5 năng lực nền tảng:

**1. Operational mindset**

Biết rằng mọi thao tác phải nằm trong chuỗi:

```text
Authorization
→ Inspection
→ Controlled action
→ Verification
→ Evidence
→ Rollback/Escalation
```

**2. Linux CLI foundation**

Tự tin với:

```text
filesystem
paths
files/directories
pipes
redirection
history
help
searching
basic text processing
```

**3. Linux identity & access**

Hiểu và vận hành được:

```text
users
groups
ownership
permissions
chmod
chown
chgrp
umask
sudo
```

**4. Secure remote administration**

Hiểu và thực hiện được:

```text
SSH
key pair
authorized_keys
secure administrative access
allowed/denied verification
```

**5. Enterprise evidence discipline**

Có thể tạo:

```text
system inventory
access evidence
configuration evidence
command history/log
verification record
```

mà không làm lộ credential.

---

# Cấu trúc chúng ta sẽ học chi tiết tiếp theo

Tôi đề xuất giữ nguyên Day 1 thành đúng hai Module lớn như sau:

```text
DAY 1
│
├── MODULE 1 — LÝ THUYẾT
│   ├── Infrastructure Operations roles
│   ├── L1/L2 boundaries
│   ├── Operational safety
│   ├── Production vs non-production
│   ├── Change authorization
│   ├── Least privilege
│   ├── SOP / Runbook / Escalation
│   ├── Access management
│   ├── Configuration baseline
│   ├── Asset inventory
│   ├── Audit evidence
│   ├── Lab readiness
│   ├── Linux architecture
│   ├── Kernel / Shell / Distribution / RHEL
│   ├── Filesystem hierarchy
│   ├── CLI / Paths / File operations
│   ├── Command syntax
│   ├── Pipes / Redirection / History / Help
│   ├── Safe command execution
│   ├── Users / Groups
│   ├── Ownership / Permissions
│   ├── chmod / chown / chgrp / umask
│   ├── sudo
│   └── SSH / SSH keys / Secure access
│
└── MODULE 2 — THỰC HÀNH
    ├── Workstation preparation
    ├── Access checklist
    ├── Linux/RHEL lab readiness
    ├── Evidence repository
    ├── Personal learning baseline
    ├── CLI navigation
    ├── Files/directories
    ├── Copy / Move / Delete
    ├── Searching
    ├── Text processing
    ├── Users/groups
    ├── Ownership/permissions
    ├── umask
    ├── Least-privilege sudo
    ├── SSH key generation
    ├── authorized_keys
    ├── Successful access verification
    ├── Denied access verification
    ├── Command documentation
    ├── System inventory
    ├── Access evidence
    ├── Configuration evidence
    └── Cleanup / Rollback
```

Toàn bộ các topic trong `Concept/Lecture` và `Assignment/Lab` của **Day 1 đều đã được ánh xạ vào hai Module trên**, không bỏ sót nội dung syllabus. Infrastructure_Operations_14_Da…

Khi bạn nói **“Trình bày Module Lý thuyết”**, tôi sẽ bắt đầu Module 1 từ foundation và đi cực chi tiết từng topic, bao gồm **What, Why, How, workflow, security, common mistakes, verification, troubleshooting, evidence và escalation boundary**, kèm command/example nhưng chưa chuyển sang bài lab. Khi bạn nói **“Trình bày Module Thực hành”**, chúng ta sẽ triển khai toàn bộ Day 1 thành lab có **Objective → Environment → Safety → Steps → Commands → Expected Result → Verification → Failure Injection → Evidence → Troubleshooting → Cleanup/Rollback**.
