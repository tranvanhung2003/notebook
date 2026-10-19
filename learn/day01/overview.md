# Day 1 — Bản đồ kiến thức cần học

Theo syllabus, **Day 1** thuộc phần **Orientation & Linux Administration Foundations**. Đây là ngày xây nền cho toàn bộ chương trình: trước khi học sâu về `systemd`, networking, storage, logging, Bash, AWS, Windows Server, Nutanix, firewall hay automation, bạn phải hiểu **cách một Infrastructure Operations engineer làm việc an toàn**, đồng thời có nền tảng Linux đủ chắc để thao tác hệ thống bằng `CLI`, quản lý `users/groups`, `permissions`, `sudo` và `SSH`. Infrastructure_Operations_14_Da…

Tôi sẽ chia toàn bộ Day 1 thành đúng **2 module lớn** như bạn yêu cầu:

1. **Module Lý thuyết — Infrastructure Operations Foundation + Linux Administration Foundation**
2. **Module Thực hành — Linux/RHEL Administrative Lab + Access/Security/Evidence Lab**

Đây là bản **sơ đồ đầy đủ nội dung cần học**, chưa đi vào độ sâu tối đa của từng command hay từng lab scenario. Khi bạn yêu cầu từng module sau, chúng ta sẽ triển khai đến mức `concept → workflow → commands → verification → troubleshooting → evidence → rollback → escalation`.

---

# MODULE 1 — LÝ THUYẾT

## Infrastructure Operations Foundation + Linux Administration Foundation

Module này có thể hình dung thành hai nửa:

**Operations mindset** → học cách vận hành infrastructure trong enterprise.

**Linux administration foundation** → học cách thao tác Linux/RHEL một cách đúng, an toàn và có kiểm soát.

---

## Phần 1 — Infrastructure Administration Roles & Responsibilities

Bạn cần hiểu Infrastructure Operations không đơn giản là “biết chạy command”.

Một Infrastructure Administrator/Operations Engineer thường phải đảm bảo:

`Availability` → hệ thống hoạt động.

`Reliability` → hệ thống hoạt động ổn định.

`Security` → không tạo access hoặc configuration vượt mức cần thiết.

`Recoverability` → khi có sự cố phải biết cách recover.

`Auditability` → mọi thay đổi quan trọng phải có evidence.

`Change control` → không tự ý sửa production.

`Escalation` → biết giới hạn quyền và năng lực của mình.

Điểm quan trọng Day 1 là hình thành tư duy:

> Không phải cứ biết cách sửa là được phép sửa.

Bạn phải phân biệt:

- việc mình **có technical capability** để làm;
- việc mình **có authorization** để làm;
- việc mình **được SOP/runbook cho phép** làm;
- việc cần approval/change ticket;
- việc phải escalate.

---

## Phần 2 — L1/L2 Support Boundaries

Bạn cần hiểu ranh giới cơ bản giữa các support level.

### L1

Thường tập trung vào:

- initial triage;
- kiểm tra trạng thái;
- thu thập evidence;
- thực hiện documented procedure;
- xác minh symptom;
- thực hiện low-risk remediation đã được phê duyệt;
- mở/escalate incident.

Ví dụ Linux:

```bash
df -h
free -m
uptime
ip addr
systemctl status <service>
```

Đây chủ yếu là `read-only verification`.

### L2

Có thể thực hiện các hoạt động sâu hơn như:

- configuration troubleshooting;
- permission troubleshooting;
- service configuration;
- user/group administration;
- log analysis;
- controlled remediation;
- configuration change theo procedure.

Nhưng L2 vẫn không đồng nghĩa với quyền thay đổi mọi thứ.

Ví dụ:

```bash
sudo systemctl restart <service>
sudo usermod ...
sudo chmod ...
```

có thể là L2 operation nhưng vẫn cần đúng authorization.

Bạn cũng cần hiểu khi nào phải chuyển tiếp lên:

- L3;
- platform engineer;
- network team;
- security team;
- application team;
- vendor support.

---

# Phần 3 — Operational Safety

Đây là một trong những nội dung quan trọng nhất Day 1.

Bạn cần hình thành thói quen trước mọi administrative action:

**Identify → Assess → Authorize → Backup → Execute → Verify → Evidence → Close/Escalate**

Ví dụ trước khi thay đổi permission:

```bash
chmod 640 file.conf
```

không nên chỉ chạy command.

Cần biết:

- file hiện tại có permission gì?
- owner/group là ai?
- application nào đang sử dụng file?
- permission mới có thể làm service fail không?
- có approval không?
- rollback là gì?
- command nào verify?
- evidence nào cần lưu?

Ví dụ baseline:

```bash
ls -l file.conf
stat file.conf
```

Sau change:

```bash
chmod 640 file.conf
```

Verification:

```bash
ls -l file.conf
stat file.conf
```

Rollback phải dựa trên state ban đầu.

---

# Phần 4 — Production vs Non-Production Environment

Bạn phải hiểu rõ sự khác nhau giữa:

`Production`

và:

`Development / Test / QA / UAT / Staging / Lab`

Day 1 phải hình thành nguyên tắc:

> Lab command không tự động trở thành production command.

Trong `production`, thường phải quan tâm thêm:

- impact;
- maintenance window;
- approval;
- dependency;
- backup;
- monitoring;
- rollback;
- communication;
- audit trail.

Một command vô hại trong lab có thể gây outage trong production.

Ví dụ:

```bash
rm file
```

trong lab có thể chỉ mất test file.

Trong production, nếu nhầm file configuration hoặc data file thì hậu quả hoàn toàn khác.

---

# Phần 5 — Change Authorization

Bạn cần hiểu không nên thực hiện infrastructure change tùy ý.

Một controlled change thường phải xác định:

- What will change?
- Why?
- Which systems are affected?
- Who approved?
- When will it happen?
- What is the implementation plan?
- How will success be verified?
- What is the rollback plan?
- What evidence must be collected?

Day 1 chưa đi sâu ServiceNow change workflow — phần đó sẽ xuất hiện ở Day 11 — nhưng ngay từ Day 1 bạn phải xây tư duy `authorized change`.

---

# Phần 6 — Least Privilege

`Least privilege` nghĩa là account chỉ được cấp đúng quyền cần thiết để thực hiện nhiệm vụ.

Không nên:

```text
User cần restart một service
→ cấp toàn bộ root access.
```

Tốt hơn là:

```text
User cần restart một service
→ chỉ cho phép sudo đối với operation cụ thể nếu policy cho phép.
```

Bạn sẽ áp dụng principle này trong chính Day 1 khi cấu hình:

- users;
- groups;
- file permissions;
- `sudo`;
- `SSH`.

Đây là một chủ đề nối trực tiếp giữa Linux administration và enterprise security.

---

# Phần 7 — SOP và Runbook

Bạn cần phân biệt khái niệm:

### SOP — Standard Operating Procedure

Mô tả procedure chuẩn của tổ chức.

Ví dụ:

`Linux User Provisioning SOP`

có thể quy định:

- request phải đến từ đâu;
- ai được approve;
- naming convention;
- group nào được phép;
- account expiry;
- verification;
- evidence.

### Runbook

Thường thiên về operational execution cụ thể.

Ví dụ:

`Linux SSH Access Troubleshooting Runbook`

có thể gồm:

```text
1. Verify server reachable
2. Verify TCP/22
3. Verify SSH service
4. Verify user
5. Verify key
6. Inspect logs
7. Escalate if authentication backend issue
```

Bạn phải học cách **làm theo runbook trước khi improvisation**.

---

# Phần 8 — Incident Escalation

Day 1 cần hiểu escalation không phải là “không biết thì đẩy ticket”.

Một escalation tốt phải chứa đủ context.

Ví dụ:

```text
Host: rhel-lab01
Time: 10:35 ICT
Issue: SSH authentication denied
Source: admin-workstation
User: ops01
Network connectivity: OK
TCP/22: reachable
SSH service: running
Password login: disabled as expected
Public key: present
Relevant log: ...
Actions performed: ...
Configuration changes: none
```

Như vậy người nhận escalation có thể tiếp tục investigation mà không phải bắt đầu lại từ đầu.

---

# Phần 9 — Access Management

Bạn sẽ học foundational lifecycle:

```text
Request
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
Modify/Revoke
```

Access không chỉ là username/password.

Nó có thể phụ thuộc vào:

- account;
- group;
- file permissions;
- `sudo`;
- SSH key;
- authorized source;
- authentication policy.

Day 1 tập trung vào local Linux access và secure administrative access.

---

# Phần 10 — Configuration Baseline

`Configuration baseline` là trạng thái chuẩn hoặc trạng thái đã được ghi nhận của hệ thống.

Ví dụ baseline Linux có thể gồm:

```text
hostname
OS
kernel
IP address
users/groups
permissions
sudo configuration
SSH configuration
disk information
```

Nếu ngày mai xảy ra vấn đề, bạn có thể so sánh:

```text
Known good state
vs
Current state
```

Đây là nền tảng của configuration troubleshooting.

---

# Phần 11 — Asset Inventory

Bạn cần hiểu một infrastructure engineer phải biết mình đang quản lý cái gì.

Linux inventory cơ bản có thể gồm:

```text
Hostname
Operating system
Distribution
Version
Kernel
Architecture
CPU
Memory
Disk
IP addresses
Network interfaces
Administrative access
```

Ví dụ một số command:

```bash
hostnamectl
cat /etc/os-release
uname -r
uname -m
lscpu
free -h
lsblk
ip addr
```

Day 1 lab yêu cầu bạn submit **system inventory evidence**. Infrastructure_Operations_14_Da…

---

# Phần 12 — Audit Evidence

Một thao tác đã thực hiện nhưng không có evidence có thể rất khó chứng minh sau này.

Day 1 cần hiểu các dạng evidence như:

```text
Command output
Terminal log
Screenshot
Timestamp
Configuration file
File metadata
User/group listing
Permission listing
SSH verification
sudo verification
System inventory
Ticket/change reference
```

Evidence tốt phải trả lời được:

```text
Who?
What?
Where?
When?
Result?
```

Sau này chủ đề này sẽ được mở rộng rất mạnh ở logging, ServiceNow, IAM audit và Day 14.

---

# Phần 13 — Lab Readiness

Trước khi chạy lab, cần chuẩn bị:

```text
Administrative workstation
Linux/RHEL system
Approved account
Network connectivity
Terminal/SSH client
Evidence directory
Naming convention
Timestamp
Rollback awareness
```

Không sử dụng:

- production credentials;
- real private keys;
- production systems;
- sensitive business data.

---

# Phần 14 — Linux Architecture

Sau phần operations foundation, Day 1 chuyển sang Linux.

Bạn cần xây mental model cơ bản:

```text
Hardware
   ↓
Linux Kernel
   ↓
System libraries / services
   ↓
Shell
   ↓
Commands / Applications
   ↓
User
```

Không cần Day 1 hiểu toàn bộ internals của Linux kernel, nhưng phải hiểu vai trò từng tầng.

---

# Phần 15 — Kernel

`kernel` là thành phần lõi của operating system.

Nó chịu trách nhiệm các lĩnh vực như:

- CPU/process management;
- memory management;
- device interaction;
- filesystem;
- networking;
- system calls.

Ví dụ:

```bash
uname -r
```

để xem kernel version.

Bạn cũng cần hiểu:

`Linux` về mặt kỹ thuật là kernel, còn một hệ điều hành hoàn chỉnh như `RHEL`, `Ubuntu`, `Debian` bao gồm kernel + userspace tools + package ecosystem + distribution-specific components.

---

# Phần 16 — Shell

`Shell` là command interpreter.

Ví dụ phổ biến:

```text
bash
sh
zsh
```

Trong chương trình này bạn sẽ chủ yếu làm việc với `Bash`.

Workflow cơ bản:

```text
User
 ↓
Shell
 ↓
Command
 ↓
Kernel / system resources
 ↓
Output
```

Cần phân biệt:

`Shell`

với:

`Terminal`

với:

`Kernel`.

---

# Phần 17 — Linux Distributions

Bạn cần hiểu `distribution` là một operating system distribution xây dựng quanh Linux kernel.

Ví dụ:

```text
Red Hat Enterprise Linux
Rocky Linux
AlmaLinux
Ubuntu
Debian
SUSE Linux Enterprise Server
```

Day 1 đặc biệt yêu cầu nhận biết **RHEL và các Linux distributions khác**. Infrastructure_Operations_14_Da…

---

# Phần 18 — RHEL

`Red Hat Enterprise Linux (RHEL)` sẽ là trọng tâm Linux enterprise của chương trình.

Day 1 mới tập trung vào foundation.

Day 2 sẽ tiếp tục sâu hơn với:

- `dnf/yum/rpm`;
- `systemd`;
- processes;
- networking;
- `SELinux`;
- `firewalld`;
- disks;
- logs;
- Bash;
- cron;
- backup.

Vì vậy Day 1 phải làm cho CLI, files, permissions, users và SSH trở nên chắc chắn trước.

---

# Phần 19 — Filesystem Hierarchy

Bạn cần hiểu filesystem Linux được tổ chức dưới root:

```text
/
```

và các directory quan trọng như:

```text
/etc
/home
/root
/var
/tmp
/usr
/opt
/boot
/dev
/proc
/sys
```

Day 1 không cần thuộc từng file, nhưng phải biết loại dữ liệu thường nằm ở đâu.

Ví dụ:

`/etc`

→ system configuration.

`/home`

→ normal users' home directories.

`/var`

→ variable data, logs và application state.

`/tmp`

→ temporary files.

---

# Phần 20 — CLI Navigation

Bạn phải sử dụng thành thạo:

```bash
pwd
ls
cd
```

và hiểu:

```text
current working directory
parent directory
home directory
root directory
```

Ví dụ:

```bash
pwd
ls -la
cd /etc
cd ..
cd ~
```

---

# Phần 21 — Absolute Path và Relative Path

Đây là concept cực kỳ quan trọng.

### Absolute path

Bắt đầu từ `/`.

```bash
/etc/ssh/sshd_config
```

### Relative path

Được tính từ current working directory.

```bash
../logs
./script.sh
```

Nếu hiểu sai path, một command như:

```bash
rm
cp
mv
chmod
chown
```

có thể tác động nhầm object.

---

# Phần 22 — File & Directory Operations

Day 1 yêu cầu bạn làm được các operation cơ bản:

```bash
touch
mkdir
cp
mv
rm
rmdir
```

và biết cách kiểm tra trước/sau.

Ví dụ:

```bash
cp source.txt backup.txt
```

Verification:

```bash
ls -l source.txt backup.txt
```

Bạn cũng phải phát triển thói quen rất quan trọng:

> Với destructive commands, xác minh target trước khi execute.

Đặc biệt với:

```bash
rm
rm -r
```

---

# Phần 23 — Command Syntax

Bạn cần hiểu cấu trúc tổng quát:

```text
command [options] [arguments]
```

Ví dụ:

```bash
ls -la /etc
```

Trong đó:

```text
ls        command
-la       options
/etc      argument
```

Đây là nền tảng để học tất cả Linux commands về sau.

---

# Phần 24 — Pipes

`pipe` ký hiệu:

```bash
|
```

cho phép output của command trước trở thành input của command sau.

Ví dụ:

```bash
cat /etc/passwd | grep ops
```

Mental model:

```text
command A
   |
stdout
   |
command B
```

Đây là một trong những khả năng mạnh nhất của Unix/Linux CLI.

---

# Phần 25 — Redirection

Bạn phải hiểu ít nhất:

```bash
>
>>
<
2>
```

Ví dụ:

```bash
echo "test" > file.txt
```

ghi đè.

```bash
echo "test2" >> file.txt
```

append.

Operational risk quan trọng:

```bash
>
```

có thể overwrite file.

Do đó redirection cũng phải được xem như một configuration/data-changing action.

---

# Phần 26 — Command History

Bạn cần học cách xem và sử dụng command history:

```bash
history
```

và hiểu lợi ích operational:

- nhớ command vừa chạy;
- reproduce troubleshooting;
- thu thập evidence;
- tránh gõ lại command dài.

Nhưng cũng phải biết security concern:

**không nhập password, token hoặc secret trực tiếp trong command line nếu có khả năng bị lưu vào history.**

---

# Phần 27 — Help Utilities

Một administrator không cần thuộc lòng mọi option.

Quan trọng hơn là biết tự tìm documentation.

Bạn sẽ dùng:

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

Đây là một kỹ năng vận hành thực tế rất quan trọng.

---

# Phần 28 — Safe Command Execution

Bạn sẽ học cách phân loại command trước khi chạy.

### Read-only

Ví dụ:

```bash
pwd
ls
id
cat
stat
```

### State-changing

Ví dụ:

```bash
mkdir
cp
mv
chmod
chown
useradd
```

### Potentially destructive

Ví dụ:

```bash
rm
```

hoặc bất kỳ command nào modify/remove data.

Quy trình an toàn:

```text
Understand command
→ identify target
→ check current state
→ check authorization
→ execute
→ verify
→ retain evidence
```

---

# Phần 29 — Users & Groups

Bạn cần hiểu Linux identity model cơ bản.

Một user có:

```text
username
UID
primary group
supplementary groups
home directory
login shell
```

Một group có:

```text
group name
GID
members
```

Bạn cần biết information thường liên quan đến:

```text
/etc/passwd
/etc/group
```

và các command inspection như:

```bash
id
whoami
groups
```

---

# Phần 30 — Ownership

Mỗi filesystem object thường có:

```text
user owner
group owner
```

Ví dụ:

```bash
ls -l file.txt
```

có thể cho thấy:

```text
-rw-r----- 1 alice ops ...
```

`alice`

→ user owner.

`ops`

→ group owner.

---

# Phần 31 — File Permissions

Bạn phải hiểu ba nhóm permission:

```text
user
group
other
```

và ba loại quyền:

```text
r = read
w = write
x = execute
```

Ví dụ:

```text
-rwxr-x---
```

phải đọc được thành:

```text
Owner: rwx
Group: r-x
Other: ---
```

Đây là một trong những phần quan trọng nhất của Day 1.

---

# Phần 32 — chmod / chown / chgrp

Ba command phải hiểu rõ.

### `chmod`

Thay đổi permission.

```bash
chmod 640 file.txt
```

### `chown`

Thay đổi owner.

```bash
chown user1 file.txt
```

hoặc owner + group:

```bash
chown user1:ops file.txt
```

### `chgrp`

Thay đổi group ownership.

```bash
chgrp ops file.txt
```

Không chỉ cần biết syntax mà còn phải hiểu security consequence của mỗi thay đổi.

---

# Phần 33 — umask

`umask` kiểm soát default permissions khi tạo file/directory mới.

Bạn cần hiểu:

```bash
umask
```

và conceptual relationship giữa:

```text
default maximum permissions
-
umask
=
resulting permissions
```

Phần tính permission cụ thể sẽ được phân tích kỹ khi học Module Lý thuyết chi tiết.

---

# Phần 34 — sudo

`sudo` cho phép user được thực thi authorized command với privilege cao hơn.

Bạn cần hiểu:

```text
normal user
      ↓
sudo policy
      ↓
authorized privileged command
```

Mục tiêu không phải là:

> Làm sao để user thành root.

Mục tiêu là:

> Làm sao cấp đúng administrative capability cần thiết với least privilege.

---

# Phần 35 — SSH Authentication

`SSH` là nền tảng của remote Linux administration.

Mental model:

```text
Administrative Workstation
          |
          | SSH
          v
      Linux Server
```

Bạn cần hiểu:

- SSH client;
- SSH server;
- username;
- authentication;
- server identity;
- authorization.

Day 1 tập trung vào secure administrative access.

---

# Phần 36 — SSH Key Management

Bạn sẽ học key-based authentication gồm:

```text
Private Key
     |
     | cryptographic proof
     v
SSH authentication
     ^
     |
Public Key
```

Thông thường:

Private key:

```text
giữ trên administrative workstation
```

Public key:

```text
được đặt trên authorized server/account
```

Bạn phải hiểu:

> Public key có thể được phân phối; private key phải được bảo vệ.

Không gửi private key lên ticket, chat, screenshot hay evidence package.

---

# Phần 37 — Secure Administrative Access

Cuối phần lý thuyết, tất cả các concept phía trên hội tụ vào một workflow:

```text
Approved admin
      ↓
Approved workstation
      ↓
Secure authentication
      ↓
Restricted user
      ↓
Least-privilege sudo
      ↓
Authorized operation
      ↓
Verification
      ↓
Audit evidence
```

Đây chính là cách Day 1 liên kết `Infrastructure Operations` với Linux security.

---

# MODULE 2 — THỰC HÀNH

## Linux/RHEL Administrative Lab + Access/Security/Evidence Lab

Assignment chính thức của Day 1 yêu cầu bạn chuẩn bị workstation và Linux/RHEL lab, thực hiện filesystem/CLI exercises, quản lý users/groups, permissions, `sudo`, SSH keys, kiểm thử allowed/denied access và nộp inventory/access/configuration evidence. Infrastructure_Operations_14_Da…

Module Thực hành vì vậy sẽ được tổ chức thành chuỗi lab có dependency như sau:

```text
Lab readiness
      ↓
System inventory
      ↓
CLI/filesystem
      ↓
Search/text processing
      ↓
Users/groups
      ↓
Ownership/permissions
      ↓
sudo
      ↓
SSH keys
      ↓
Allowed/denied testing
      ↓
Evidence package
```

---

## Lab 1 — Administrative Workstation Preparation

Bạn sẽ chuẩn bị administrative workstation với:

```text
Terminal
SSH client
Working directory
Evidence repository
Naming convention
Approved account information
Target Linux/RHEL information
```

Ví dụ evidence structure có thể theo dạng:

```text
day1/
├── inventory/
├── cli/
├── users-groups/
├── permissions/
├── sudo/
└── ssh/
```

Mục tiêu là ngay từ đầu lab đã có chỗ lưu evidence có tổ chức.

---

# Lab 2 — Approved Access Checklist

Trước khi thao tác Linux server, bạn phải xác minh:

```text
Target system đúng chưa?
Environment là lab/non-production?
Account có được authorize?
Scope của account là gì?
Có sudo hay không?
SSH source có được phép?
Lab có rollback path?
```

Mục tiêu không chỉ là kết nối được mà còn phải chứng minh:

> Tôi đang truy cập đúng hệ thống, bằng đúng account, với đúng phạm vi.

---

# Lab 3 — Linux/RHEL Lab Readiness

Bạn sẽ xác minh machine bằng các command dạng:

```bash
hostnamectl
cat /etc/os-release
uname -r
whoami
id
pwd
```

Sau đó ghi nhận:

```text
hostname
distribution
version
kernel
current user
UID/GID
```

---

# Lab 4 — System Inventory

Bạn phải thu thập system inventory baseline.

Có thể gồm:

```bash
hostnamectl
cat /etc/os-release
uname -a
lscpu
free -h
lsblk
ip addr
```

Mục tiêu là tạo evidence mô tả lab machine trước khi thực hiện thay đổi.

---

# Lab 5 — CLI Navigation

Bạn sẽ luyện:

```bash
pwd
ls
ls -l
ls -la
cd
```

và phải thành thạo:

```text
/
.
..
~
absolute path
relative path
```

Bạn sẽ cần chứng minh mình hiểu path bằng cách di chuyển giữa nhiều directory mà không phụ thuộc GUI.

---

# Lab 6 — File & Directory Creation

Bạn sẽ thực hành:

```bash
mkdir
touch
```

Ví dụ tạo cấu trúc:

```text
day1_lab/
├── input/
├── output/
└── archive/
```

Sau mỗi operation phải verify:

```bash
ls -l
```

hoặc:

```bash
find
```

---

# Lab 7 — Copy / Move / Delete

Bạn sẽ thực hiện:

```bash
cp
mv
rm
rmdir
```

nhưng theo safe workflow:

```text
Check source
→ Check destination
→ Execute
→ Verify
```

Mục tiêu không phải chỉ biết command mà phải tránh sai target.

---

# Lab 8 — Searching

Assignment yêu cầu có `searching exercises`.

Bạn sẽ tìm:

- files;
- directories;
- text content.

Ví dụ các utility cơ bản có thể gồm:

```bash
find
grep
```

Sau này Day 2 mới mở rộng mạnh hơn sang:

```text
grep
awk
sed
regular expressions
```

---

# Lab 9 — Text Processing Foundation

Bạn sẽ thực hành những workflow dạng:

```bash
cat file
grep pattern file
command | grep pattern
```

Ví dụ:

```bash
cat /etc/passwd | grep ops
```

Mục tiêu là bắt đầu xây tư duy:

```text
input
→ processing/filtering
→ output
```

Đây là nền cho log investigation và automation sau này.

---

# Lab 10 — Controlled User Creation

Bạn sẽ tạo test users trong lab.

Ví dụ conceptual workflow:

```text
Create user
→ inspect user
→ verify UID/GID
→ verify home
→ test login/access
```

Command thực tế sẽ được giải thích chi tiết khi học Module Thực hành.

Quan trọng: chỉ dùng **controlled lab users**, không dùng production account.

---

# Lab 11 — Controlled Group Management

Bạn sẽ:

- tạo group;
- thêm user vào group;
- xác minh membership;
- kiểm tra access dựa trên group.

Mental model:

```text
User
  ↓
Group membership
  ↓
Group permission
  ↓
Resource access
```

---

# Lab 12 — Ownership Configuration

Bạn sẽ tạo file/directory rồi thay đổi:

```text
user ownership
group ownership
```

bằng:

```bash
chown
chgrp
```

Sau đó verify bằng:

```bash
ls -l
stat
```

---

# Lab 13 — Permission Configuration

Bạn sẽ thực hành permission scenarios như:

```text
Owner được read/write
Group chỉ read
Other không có access
```

và biểu diễn thành:

```text
rw-r-----
```

sau đó áp dụng bằng `chmod`.

Quan trọng hơn là kiểm thử bằng các account khác nhau để xác minh permission thật sự hoạt động.

---

# Lab 14 — umask Verification

Bạn sẽ:

```text
Check current umask
→ create new file
→ inspect permission
→ create directory
→ inspect permission
```

để quan sát tác động của `umask`.

Không chỉ học công thức trên lý thuyết.

---

# Lab 15 — Least-Privilege sudo

Đây là lab security quan trọng.

Bạn sẽ thiết kế scenario:

```text
User A
   |
   | sudo?
   v
Specified administrative operation
```

Mục tiêu:

- authorized operation → được phép;
- unauthorized privileged operation → bị từ chối.

Bạn phải thu evidence của **cả thành công và thất bại**.

Đó là cách chứng minh least privilege thực sự có hiệu lực.

---

# Lab 16 — SSH Key Generation

Bạn sẽ tạo SSH key pair trong lab.

Conceptual result:

```text
Private key
Public key
```

Private key phải được giữ trên administrative workstation và không đưa vào evidence package.

Bạn sẽ kiểm tra:

- file location;
- ownership;
- permission;
- public key content;
- key fingerprint khi phù hợp.

---

# Lab 17 — SSH Public Key Configuration

Bạn sẽ cấu hình public key cho controlled Linux user.

Thông thường liên quan đến:

```text
~/.ssh/
authorized_keys
```

Bạn phải quan tâm đến:

- correct user;
- correct ownership;
- correct permissions;
- correct public key.

Sai một trong các yếu tố trên có thể làm SSH authentication fail.

---

# Lab 18 — SSH Access Verification

Bạn phải chứng minh:

```text
Authorized identity
→ Access succeeds
```

và xác minh sau login:

```bash
whoami
id
hostname
```

Điểm này rất quan trọng vì:

> “SSH connection succeeded” chưa đủ.

Bạn còn phải chứng minh login vào đúng host bằng đúng identity.

---

# Lab 19 — Denied Access Verification

Syllabus yêu cầu không chỉ test authorized access mà còn phải verify **denied access**. Infrastructure_Operations_14_Da…

Ví dụ conceptual scenarios:

```text
Unauthorized user
→ denied

Wrong SSH key
→ denied

Insufficient file permission
→ denied

Unauthorized sudo command
→ denied
```

Security control chỉ được chứng minh tốt khi kiểm thử cả:

```text
positive test
+
negative test
```

---

# Lab 20 — Command Documentation

Trong quá trình lab, bạn sẽ phải lưu lại các command đã thực hiện.

Nhưng command log phải đủ context.

Ví dụ không nên chỉ lưu:

```bash
chmod 640 test.txt
```

Mà evidence tốt hơn phải bao gồm:

```text
Timestamp
Host
User
Original state
Command
Result
Verification
```

---

# Lab 21 — Access Verification Evidence

Một evidence package về access nên chứng minh được:

```text
Who logged in?
Which server?
Which authentication method?
What privilege?
What was allowed?
What was denied?
```

Các command như:

```bash
whoami
id
groups
sudo ...
```

có thể đóng vai trò verification tùy lab scenario.

---

# Lab 22 — Configuration Evidence

Bạn sẽ thu thập evidence cho:

```text
Users
Groups
Ownership
Permissions
sudo
SSH
```

Ví dụ:

```bash
id <user>
getent group <group>
ls -l <file>
stat <file>
```

Với configuration chứa security-sensitive information, evidence phải tránh làm lộ:

- password;
- private key;
- secret;
- token.

---

# Lab 23 — Final Day 1 Submission Package

Theo syllabus, cuối Day 1 phải có ít nhất ba nhóm deliverable quan trọng:

```text
System Inventory
+
Access Verification
+
Configuration Evidence
```

cùng documentation các command đã thực hiện. Infrastructure_Operations_14_Da…

Có thể hình dung final evidence package:

```text
Day1_Evidence/
│
├── 01_System_Inventory
│
├── 02_CLI_Filesystem
│
├── 03_User_Group
│
├── 04_Ownership_Permissions
│
├── 05_Sudo
│
├── 06_SSH
│
├── 07_Access_Tests
│
└── 08_Command_Log
```

---

# Failure Injection dự kiến trong Module Thực hành

Khi học chi tiết, tôi sẽ đưa vào một số lỗi mô phỏng an toàn để bạn tự troubleshoot, ví dụ:

```text
1. User không thuộc đúng group
2. File permission sai
3. File owner/group sai
4. sudo operation bị denied
5. SSH key authentication bị fail
```

Bạn sẽ không chỉ “sửa cho chạy”, mà phải thực hiện:

```text
Observe symptom
→ collect evidence
→ form hypothesis
→ test
→ identify root cause
→ remediate safely
→ verify
→ record evidence
```

---

# Boundary Day 1 → Day 2

Một điểm rất quan trọng là **không nhầm phạm vi Day 1 và Day 2**.

Day 1 kết thúc khi bạn đã có nền tảng:

```text
Infrastructure Operations mindset
Operational safety
Access management
Linux architecture
CLI
Filesystem
Users/groups
Permissions
sudo
SSH
Evidence collection
```

Day 2 mới chuyển sang Linux/RHEL system administration sâu hơn:

```text
Package management
Processes
systemd
SELinux
firewalld
Networking
Storage
Logs
Bash
cron
Backup/restore
Operational troubleshooting
```

Syllabus xác định Day 1–2 cùng thuộc module **Orientation & Linux Administration Foundations**, trong đó Day 1 tạo nền để Day 2 bắt đầu quản trị system services, networking, storage và troubleshooting. Infrastructure_Operations_14_Da…

---

# Mục tiêu hoàn thành Day 1

Sau khi hoàn thành đầy đủ **Module Lý thuyết + Module Thực hành**, bạn phải đạt được mức có thể tự giải thích và thực hiện được workflow:

```text
Understand operational scope
        ↓
Verify authorization
        ↓
Identify Linux system
        ↓
Navigate filesystem safely
        ↓
Manipulate files/directories
        ↓
Manage controlled users/groups
        ↓
Configure ownership/permissions
        ↓
Apply least privilege
        ↓
Use sudo appropriately
        ↓
Configure SSH key access
        ↓
Test allowed + denied access
        ↓
Verify configuration
        ↓
Collect audit evidence
        ↓
Rollback / Escalate when required
```

Đây cũng chính là triết lý của toàn bộ 14-day program: chương trình không chỉ yêu cầu biết technical commands mà hướng tới **operational readiness**, gồm hands-on lab, verification evidence và controlled troubleshooting. Infrastructure_Operations_14_Da…

Khi bạn muốn đi tiếp, chỉ cần nói **“Trình bày chi tiết Module Lý thuyết Day 1”** hoặc **“Trình bày chi tiết Module Thực hành Day 1”**; tôi sẽ triển khai module đó từ foundation đến operational level, đầy đủ `What → Why → How → Workflow → Security → Verification → Troubleshooting → Evidence → Rollback → Escalation`.
