# Day 2 — Module Lý thuyết

## Linux/RHEL System Administration & Operational Troubleshooting

Theo syllabus, Module Lý thuyết Day 2 phải cover đầy đủ: **Linux/RHEL system administration; `dnf`/`yum`/`rpm`; repository và subscription awareness; environment variables; processes, process states, signals và resource usage; `systemd` units/service lifecycle; boot targets và service recovery; SELinux; `firewalld`; IP, routing, DNS, TCP/UDP; `ping`, `traceroute`/`tracepath`, `curl`, `ss`, `ip`; disks, partitions, filesystems, mounts, `/etc/fstab`, disk/inode capacity; `journalctl`, syslog, log rotation; `grep`/`awk`/`sed`, regular expressions; Bash variables/control flow/exit codes/error handling; `cron`; compression; `tar`/`rsync`; restore; health checks và operational troubleshooting**. Infrastructure_Operations_14_Da…

Syllabus là **source of truth** cho phạm vi. Phần giải thích sâu hơn dưới đây là **kiến thức bổ sung nhằm giúp bạn vận hành thực tế**, không thay đổi scope của syllabus.

Một lưu ý về version: syllabus chỉ nói RHEL nói chung, không khóa vào một major release. Tài liệu Red Hat hiện tại đã có **RHEL 10**, vì vậy những kiến thức architecture bên dưới được trình bày theo cách áp dụng chung cho RHEL hiện đại; các hành vi phụ thuộc release phải kiểm tra lại trên chính server. Chẳng hạn, tài liệu RHEL 10 hiện dùng `dnf` làm công cụ software management chính; trong RHEL 9, `yum` được giữ như alias/compatibility interface của `dnf`. Vì vậy không nên mặc định mọi RHEL release xử lý `yum` giống nhau. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10?utm_source=chatgpt.com)

---

# 1. Bức tranh lớn: một Linux/RHEL server vận hành như thế nào?

Trước khi học từng command, cần hình dung Linux server dưới góc nhìn Infrastructure Operations.

Một application đang hoạt động thường phụ thuộc vào chuỗi:

```text
Hardware / Virtual Hardware
        ↓
Kernel
        ↓
Storage + Filesystems
        ↓
Network stack
        ↓
Security controls
(DAC permissions + SELinux + firewall)
        ↓
systemd
        ↓
Service / Process
        ↓
Application configuration
        ↓
Listening socket / Port
        ↓
Client connectivity
        ↓
Application response
```

Bên cạnh chuỗi này còn có hai hệ thống quan trọng chạy song song:

```text
Logging / Monitoring
        ↓
Evidence / Alert / Troubleshooting
```

và:

```text
Automation / Scheduling / Backup
        ↓
Repeatable Operations
```

Day 2 thực chất dạy bạn hiểu và kiểm tra gần như toàn bộ dependency chain trên.

Ví dụ có incident:

```text
"Web application không truy cập được."
```

Người chưa có troubleshooting framework thường làm:

```bash
sudo systemctl restart httpd
```

Nếu không được thì restart server.

Người làm Infrastructure Operations đúng cách sẽ hỏi:

```text
1. Server có reachable không?
2. Interface/IP có đúng không?
3. Routing có đúng không?
4. DNS có resolve đúng không?
5. Service có active không?
6. Process có tồn tại không?
7. Port có LISTEN không?
8. Local curl có thành công không?
9. firewalld có permit traffic không?
10. SELinux có deny application không?
11. Filesystem có full không?
12. Application log nói gì?
13. Có change nào xảy ra trước incident không?
```

Đó là tư duy xuyên suốt Day 2.

---

# 2. Linux/RHEL system administration

## 2.1. System administration nghĩa là gì?

Linux system administration không đơn giản là biết command.

Infrastructure administrator chịu trách nhiệm duy trì server ở trạng thái:

```text
Available
Predictable
Secure
Supportable
Recoverable
Auditable
```

Một thay đổi được xem là hoàn thành khi không chỉ “command chạy được”, mà còn phải có:

```text
Pre-check
→ Change
→ Verification
→ Functional validation
→ Evidence
→ Rollback readiness
→ Documentation
```

Ví dụ cài một package:

```text
Sai tư duy:
dnf install httpd
→ Done

Đúng tư duy:
Check OS/repository
→ Check change approval
→ Check package/version
→ Install
→ Verify RPM
→ Verify files
→ Verify service
→ Verify network behavior
→ Save evidence
```

Đây chính là khác biệt giữa “biết Linux” và “vận hành Linux”.

---

# 3. Package management: `dnf`, `yum`, `rpm`

## 3.1. Package là gì?

Một **package** là đơn vị software được đóng gói để operating system có thể:

```text
Install
Upgrade
Query
Verify
Remove
Track
```

Trong RHEL ecosystem, package phổ biến sử dụng định dạng **RPM**.

Ví dụ package:

```text
httpd
openssh-server
rsyslog
chrony
curl
bash
```

RPM package chứa không chỉ binary mà có thể bao gồm:

```text
Executable files
Libraries
Configuration files
Documentation
Metadata
Dependencies
Package scripts
```

---

# 4. `rpm` — low-level package management

`rpm` vừa là package format vừa là utility quản lý RPM database.

Ví dụ query:

```bash
rpm -q bash
```

Có thể trả về dạng:

```text
bash-5.x.x-...
```

Ý nghĩa:

```text
rpm
-q     query
bash   package name
```

Kiểm tra package chưa cài:

```bash
rpm -q example-package
```

Có thể nhận:

```text
package example-package is not installed
```

---

## 4.1. Package metadata

Thông thường package identity gồm các thành phần conceptual:

```text
Name
Version
Release
Architecture
```

Thường viết:

```text
NAME-VERSION-RELEASE.ARCH
```

Ví dụ conceptual:

```text
httpd-2.4.x-xx.elX.x86_64
```

Điều quan trọng trong Operations là không chỉ nói:

> "`httpd` đã được cài."

Mà phải có khả năng nói:

> "`httpd` version/release nào đang được cài, architecture gì, đến từ repository nào và có đúng baseline không."

---

## 4.2. Query package bằng `rpm`

Các kiểu kiểm tra quan trọng:

```bash
rpm -q package
rpm -qi package
rpm -ql package
rpm -qf /path/to/file
```

Concept:

```text
-q       query package
-qi      package information
-ql      files belonging to package
-qf      package owning a file
```

Ví dụ:

```bash
rpm -qf /usr/bin/ssh
```

Trả lời câu hỏi:

> File `/usr/bin/ssh` thuộc package nào?

Đây là kỹ thuật hữu ích khi troubleshoot missing/corrupt binary.

---

# 5. `dnf` — dependency-aware package manager

Khác với `rpm` ở mức thấp, `dnf` làm việc với repository và dependency graph.

Nếu package A cần package B và C:

```text
A
├── B
└── C
```

`dnf` có thể resolve và cài các dependencies cần thiết.

Red Hat hiện hướng dẫn RHEL 10 sử dụng `dnf`; khi cài package, `dnf` tự động resolve dependencies từ repository phù hợp. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/installing-rhel-content?utm_source=chatgpt.com)

Ví dụ:

```bash
sudo dnf install httpd
```

Các thao tác conceptual quan trọng:

```bash
dnf search <term>
dnf info <package>
dnf install <package>
dnf remove <package>
dnf update
dnf repolist
dnf list --installed
```

Red Hat RHEL 10 documentation xác nhận `dnf repolist`, `dnf list --installed`, `dnf repoquery` và các command tương tự là các interface tiêu chuẩn để kiểm tra content và repositories. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/dnf-commands-list?utm_source=chatgpt.com)

---

# 6. `yum` awareness

Syllabus yêu cầu hiểu cả:

```text
dnf
yum
rpm
```

`yum` rất quan trọng vì enterprise environment vẫn tồn tại nhiều thế hệ RHEL.

Với RHEL 9, Red Hat ghi rõ `yum` là alias của `dnf` nhằm compatibility với các RHEL version cũ. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_software_with_the_dnf_tool/con_software-management-tools-in-red-hat-enterprise-linux-9_managing-software-with-the-dnf-tool?utm_source=chatgpt.com)

Nhưng trong môi trường thực:

```text
RHEL 7          → yum historically central
RHEL 8/9        → dnf engine, yum compatibility
RHEL 10         → documentation focuses on dnf
```

Do đó runbook nên ưu tiên command được support trên version thực tế.

Đừng viết automation dựa trên giả định:

```text
"If Linux is RHEL, yum always behaves exactly the same."
```

---

# 7. Repository

Package manager cần biết **package lấy từ đâu**.

Repository là nguồn content có metadata và packages.

Concept:

```text
RHEL host
   |
   | dnf request
   v
Repository metadata
   |
   +---- package A
   +---- package B
   +---- dependencies
```

Trên RHEL hiện đại, repository configuration có thể nằm dưới:

```text
/etc/yum.repos.d/*.repo
```

và DNF global configuration:

```text
/etc/dnf/dnf.conf
```

Red Hat RHEL 10 vẫn sử dụng các location này và khuyến nghị custom repository được cấu hình thành `.repo` riêng. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/managing-custom-software-repositories?utm_source=chatgpt.com)

---

## 7.1. BaseOS và AppStream

Trong RHEL hiện đại, hai repository concepts rất quan trọng là:

```text
BaseOS
AppStream
```

**BaseOS** cung cấp nền tảng OS cốt lõi.

**AppStream** cung cấp nhiều user-space application và runtime content.

RHEL 10 tiếp tục phân phối content qua BaseOS, AppStream cùng các repository bổ sung. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/managing_software_with_the_dnf_tool/index?utm_source=chatgpt.com)

---

## 7.2. Repository trust

Không được hiểu repository chỉ như “URL để download RPM”.

Repository là **software supply-chain trust boundary**.

Nếu thêm repository không kiểm soát:

```text
Untrusted repository
        ↓
Untrusted package
        ↓
Code executes on server
        ↓
Potential system compromise
```

Red Hat cảnh báo custom/unverified repositories có thể tạo security, compatibility, stability và maintainability risk. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/managing-custom-software-repositories?utm_source=chatgpt.com)

Trong enterprise, L1/L2 không nên tự tiện:

```bash
dnf config-manager --add-repo <random-url>
```

chỉ để “cài package cho nhanh”.

---

# 8. Subscription awareness

RHEL là commercial enterprise distribution.

Subscription liên quan đến:

```text
Access to supported content
Software updates
Security updates
Vendor support
Entitlements
Lifecycle/supportability
```

Ở Operations level, bạn không nhất thiết phải là subscription architect, nhưng phải nhận ra symptom:

```text
Package không tìm thấy
Repository không available
Metadata download thất bại
System chưa register
Entitlement/content access không đúng
```

Không nên vội kết luận:

> Internet lỗi.

Có thể issue nằm ở:

```text
Subscription
Repository
Proxy
DNS
Certificate
Repository enablement
```

---

# 9. Package management troubleshooting framework

Một package install failure nên được tách layer:

```text
Package name đúng?
        ↓
Package available?
        ↓
Repository enabled?
        ↓
Subscription/content access?
        ↓
DNS?
        ↓
Network route?
        ↓
Proxy?
        ↓
TLS/certificate?
        ↓
Dependency conflict?
        ↓
Disk space?
        ↓
RPM database?
```

Evidence nên giữ:

```text
OS release
Timestamp
dnf output
Repository list
Package version
Relevant error
Change/ticket reference
```

L1/L2 thường có thể query/install approved packages. Việc thêm unsigned/unapproved repository, force downgrade critical packages, bypass package signature validation hoặc thay đổi enterprise repository architecture thường phải escalate/change-control.

---

# 10. Environment variables

## 10.1. Environment variable là gì?

Environment variable là cặp:

```text
NAME=value
```

được process sử dụng như contextual configuration.

Ví dụ:

```bash
PATH=/usr/local/bin:/usr/bin:/bin
HOME=/home/operator
LANG=en_US.UTF-8
```

Process thường inherit environment từ parent process.

Concept:

```text
Shell
  |
  +-- environment
  |
  +---- child process
         inherits exported environment
```

---

# 11. Shell variable và environment variable

Một shell variable:

```bash
APP_ENV=test
```

chưa nhất thiết được child process inherit.

Khi:

```bash
export APP_ENV=test
```

nó trở thành environment variable đối với child processes được launch sau đó.

Có thể kết hợp:

```bash
export APP_ENV=test
```

---

## 11.1. `$PATH`

`PATH` đặc biệt quan trọng.

Khi bạn nhập:

```bash
curl
```

shell không mặc định tìm toàn bộ disk.

Nó search directories trong:

```bash
echo "$PATH"
```

theo thứ tự.

Ví dụ:

```text
/usr/local/bin:/usr/bin:/bin
```

Nếu executable nằm trong:

```text
/opt/myapp/bin
```

nhưng `/opt/myapp/bin` không nằm trong `PATH`, command:

```bash
mytool
```

có thể báo:

```text
command not found
```

trong khi:

```bash
/opt/myapp/bin/mytool
```

lại chạy được.

---

# 12. Environment và automation failure

Một issue kinh điển:

```text
./backup.sh
```

chạy được interactively.

Nhưng cron chạy script lại fail.

Tại sao?

Interactive shell có thể có:

```text
PATH
HOME
LANG
custom variables
credentials/config paths
shell initialization
```

khác environment của `cron`.

Vì vậy production-grade scripts thường không nên phụ thuộc mù quáng vào interactive environment.

---

## Verification

Read-only:

```bash
env
printenv
echo "$PATH"
printenv HOME
```

Khi troubleshooting cần hỏi:

```text
User nào chạy?
Shell nào?
Interactive hay non-interactive?
Environment nào được inherit?
Variable có exported không?
```

Evidence cần lưu variable liên quan, nhưng **không bao giờ dump secret/token/password vào ticket hoặc log**.

---

# 13. Linux process

Một **process** là một running instance của program.

Ví dụ executable:

```text
/usr/sbin/httpd
```

có thể tạo một hoặc nhiều processes.

Process có:

```text
PID
PPID
User/UID
State
CPU usage
Memory usage
Open files
Environment
Working directory
Command line
```

---

# 14. PID và PPID

**PID — Process ID** nhận diện process.

**PPID — Parent Process ID** cho biết parent.

Concept:

```text
systemd PID 1
    |
    +--- sshd
    |
    +--- chronyd
    |
    +--- application service
```

Bạn không nên coi process hoàn toàn độc lập; parent-child relationship đôi khi cực kỳ quan trọng.

---

# 15. Process states

Ở mức foundation, các states thường gặp gồm:

```text
Running / runnable
Sleeping
Uninterruptible sleep
Stopped
Zombie
```

Tên hiển thị bởi tools có thể dùng single-letter codes.

### Running/runnable

Process đang chạy trên CPU hoặc sẵn sàng được scheduler cho CPU.

### Sleeping

Process đang chờ một event.

Sleeping không có nghĩa process lỗi.

Server có hàng trăm sleeping processes là bình thường.

### Uninterruptible sleep

Thường liên quan đến chờ kernel/I/O.

Nếu nhiều processes mắc prolonged uninterruptible wait, có thể phải điều tra:

```text
Storage latency
NFS
Device issue
Kernel I/O
```

### Stopped

Process bị stop/suspend.

### Zombie

Process đã kết thúc nhưng parent chưa thu hồi exit status.

Zombie không giống một process đang tiêu tốn nhiều CPU; root cause thường liên quan parent process behavior.

---

# 16. Process inspection

Các command commonly used:

```bash
ps
ps -ef
ps aux
top
```

Ví dụ:

```bash
ps -ef | grep httpd
```

Nhưng cần cẩn thận vì `grep` cũng có thể xuất hiện trong result.

`top` giúp quan sát:

```text
CPU
Memory
Load
Processes
```

Kiến thức bổ sung hữu ích là không đánh đồng:

```text
High CPU = always bad
```

Một batch job được thiết kế để tận dụng CPU có thể dùng 100% một core hoàn toàn hợp lệ.

Câu hỏi đúng phải là:

```text
Expected workload hay unexpected?
Duration bao lâu?
System latency thế nào?
SLA bị ảnh hưởng không?
Process có progress không?
```

---

# 17. Signals

Signal là cơ chế kernel/process dùng để truyền event/control.

Các signal quan trọng ở Operations level:

```text
SIGTERM
SIGKILL
SIGHUP
SIGINT
```

### SIGTERM

Yêu cầu process terminate theo cách có cơ hội cleanup.

Đây thường là lựa chọn trước.

Concept:

```bash
kill <PID>
```

mặc định thường gửi `SIGTERM`.

### SIGKILL

Force kernel terminate process.

```bash
kill -9 <PID>
```

Process không thể handle hoặc ignore SIGKILL.

Điều này đồng nghĩa nó không có cơ hội cleanup gracefully.

Vì thế:

```text
kill -9
```

không phải “kill mạnh hơn nên tốt hơn”.

Nó là escalation trong process termination.

### SIGHUP

Historically liên quan terminal disconnect và thường được daemons dùng để reload configuration, nhưng behavior phụ thuộc application.

Không được giả định mọi process đều reload với `SIGHUP`.

---

# 18. Service và process không phải một thứ

Đây là distinction cực kỳ quan trọng.

**Process** là running program instance.

**Service** là một managed workload/lifecycle abstraction, trên RHEL hiện đại thường do `systemd` quản lý.

Một service có thể:

```text
spawn one process
spawn multiple workers
restart processes
manage dependencies
set environment
set privileges
```

Do đó:

```bash
kill <PID>
```

không nhất thiết tương đương:

```bash
systemctl stop service
```

Thậm chí `systemd` có thể tự restart process.

---

# 19. Resource usage

Một server phải được đánh giá ít nhất qua:

```text
CPU
Memory
Disk
I/O
Network
Process count
```

Day 2 tập trung foundation, nhưng bạn cần hiểu symptom mapping.

| Symptom                 | Có thể liên quan                                    |
| ----------------------- | --------------------------------------------------- |
| CPU rất cao             | busy process, loop, workload spike                  |
| Memory pressure         | application growth, cache, leak, oversized workload |
| Disk full               | logs, backup, temp data                             |
| Inode full              | quá nhiều small files                               |
| I/O wait                | slow storage, heavy disk access                     |
| Port unavailable        | process/service/socket conflict                     |
| Process repeatedly dies | config/dependency/resource/security problem         |

Không được chẩn đoán root cause chỉ từ một metric.

---

# 20. `systemd`

`systemd` là system/service manager trung tâm trên RHEL hiện đại. Red Hat mô tả **unit** là object cơ bản mà `systemd` quản lý; nó hỗ trợ service startup song song, on-demand activation và dependency-based control. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/managing-systemd_configuring-basic-system-settings?utm_source=chatgpt.com)

PID 1 thường là:

```text
systemd
```

Nó tham gia quản lý:

```text
Boot
Services
Targets
Sockets
Timers
Mounts
Dependencies
```

---

# 21. `systemd unit`

Unit là object được `systemd` quản lý.

Các loại unit có thể gồm:

```text
.service
.target
.socket
.timer
.mount
.path
```

Day 2 tập trung chủ yếu:

```text
service
target
```

nhưng hiểu các loại khác giúp bạn thấy architecture thống nhất.

---

# 22. Service unit

Ví dụ:

```text
sshd.service
firewalld.service
chronyd.service
```

Service unit mô tả:

```text
How to start
How to stop
Dependencies
Environment
Restart behavior
Execution user
Startup ordering
```

---

# 23. Unit file locations và precedence

Không cần thuộc lòng mọi precedence ngay Day 2, nhưng cần biết unit definitions có thể đến từ vendor package hoặc local administrator configuration.

Điều quan trọng về Operations:

```text
Không chỉnh vendor unit một cách tùy tiện.
```

Thường local customization nên sử dụng supported override/drop-in mechanism.

Tại sao?

Package upgrade có thể thay vendor-managed files.

---

# 24. Service lifecycle

Lifecycle cơ bản:

```text
inactive
  ↓ start
active
  ↓ stop
inactive
```

Nếu startup fail:

```text
inactive
  ↓ start
activating
  ↓ failure
failed
```

Một service cũng có thể:

```text
active (running)
active (exited)
failed
```

tùy service type.

Không nên hard-code assumption rằng mọi healthy service phải hiện exact string:

```text
active (running)
```

---

# 25. `systemctl`

Common operations:

```bash
systemctl status <unit>
systemctl start <unit>
systemctl stop <unit>
systemctl restart <unit>
systemctl reload <unit>
systemctl enable <unit>
systemctl disable <unit>
systemctl is-active <unit>
systemctl is-enabled <unit>
```

Cần hiểu khác biệt.

### `start`

Thay đổi runtime state hiện tại.

### `enable`

Thay đổi startup relationship để unit được kéo vào khi boot qua target/dependency phù hợp.

Vì vậy:

```text
active ≠ enabled
```

và:

```text
enabled ≠ currently running
```

Ví dụ hoàn toàn có thể:

```text
active + disabled
inactive + enabled
active + enabled
inactive + disabled
```

---

# 26. `restart` và `reload`

Không giống nhau.

`restart` thường:

```text
stop
→ start
```

Có khả năng tạo interruption.

`reload` yêu cầu service nạp lại configuration mà không full restart, nhưng chỉ có ý nghĩa nếu service hỗ trợ.

Không được giả định:

```bash
systemctl reload something
```

luôn được support.

---

# 27. Tại sao không nên restart quá sớm?

Suppose:

```text
Application unavailable
```

Bạn lập tức:

```bash
systemctl restart app
```

Có thể service phục hồi.

Nhưng bạn vừa:

```text
Destroyed transient failure state
Changed timestamps/state
Potentially lost diagnostic context
Made root cause harder to identify
```

Operationally, trước restart nên cân nhắc thu:

```text
systemctl status
journal
process state
port state
resource state
application log
```

trừ trường hợp service restoration SLA buộc immediate recovery.

---

# 28. Service dependency

Service A có thể phụ thuộc:

```text
Network
Filesystem
Socket
Database
Another service
Credential/config file
```

Ví dụ:

```text
Application
   ↓
Database
   ↓
Mounted filesystem
```

Restart application không sửa root cause nếu filesystem chưa mount.

Vì vậy troubleshooting service phải nhìn dependency, không chỉ nhìn unit.

---

# 29. Boot target

`systemd target` là nhóm units đại diện cho một system state/synchronization point. Red Hat mô tả các target thông dụng như `multi-user.target`, `graphical.target`, `rescue.target` và `emergency.target`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/managing-systemd_configuring-basic-system-settings?utm_source=chatgpt.com)

Conceptual mapping:

```text
multi-user.target
```

thường là non-GUI multi-user operating state.

```text
graphical.target
```

bao gồm graphical environment.

```text
rescue.target
```

dành cho recovery mode với environment hạn chế.

```text
emergency.target
```

tối giản hơn nữa để emergency recovery.

---

# 30. Default target

Kiểm tra:

```bash
systemctl get-default
```

Concept:

```text
default.target
       ↓
selected boot target
```

Changing default target là configuration-changing operation và có thể ảnh hưởng boot behavior, nên không làm tùy tiện trên production.

---

# 31. Service startup recovery

Service startup failure thường thuộc một trong các nhóm:

```text
Configuration syntax
Missing file/directory
Wrong permissions
SELinux denial
Port conflict
Dependency failure
Filesystem unavailable
Environment issue
Resource exhaustion
Package/library problem
Credential/certificate issue
```

Flow tốt:

```text
systemctl status service
        ↓
journalctl for service
        ↓
configuration validation
        ↓
dependencies
        ↓
permissions / SELinux
        ↓
port/socket
        ↓
resource/storage
        ↓
correct root cause
        ↓
start
        ↓
functional test
```

---

# 32. SELinux

SELinux là một trong những topics Day 2 quan trọng nhất.

SELinux triển khai **Mandatory Access Control (MAC)**, bổ sung một security layer ngoài traditional UNIX **Discretionary Access Control (DAC)**. Tài liệu RHEL 10 hiện mô tả SELinux là MAC mechanism nhằm giới hạn interactions giữa processes, files và network resources. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/getting-started-with-selinux?utm_source=chatgpt.com)

Traditional permissions:

```text
owner
group
other
rwx
```

là DAC.

SELinux đưa thêm policy decision.

Concept:

```text
Application request
      |
      v
DAC permissions
      |
  allowed?
      |
      v
SELinux policy
      |
  allowed?
      |
      v
Operation proceeds
```

Nếu DAC đã deny, SELinux không “override để allow”.

---

# 33. SELinux contexts

Objects như files và processes có SELinux security context.

Typical conceptual form:

```text
user:role:type:level
```

Ở foundation level, **type** thường là phần bạn gặp nhiều nhất khi troubleshooting.

Ví dụ conceptual:

```text
httpd_t
httpd_sys_content_t
```

Một web process chạy trong domain phù hợp được policy cho phép access các file types nhất định.

Do đó file có:

```text
-rw-r--r--
```

đúng UNIX permissions vẫn có thể bị deny nếu SELinux labeling/policy không phù hợp.

---

# 34. SELinux modes

Các modes bạn cần biết:

```text
Enforcing
Permissive
Disabled
```

Red Hat khuyến nghị `Enforcing`; `Permissive` vẫn thực hiện labeling và log denials nhưng không block operation; `Disabled` bị Red Hat strongly discouraged vì không chỉ ngừng enforcement mà còn ảnh hưởng labeling của persistent objects. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/getting-started-with-selinux_using-selinux?utm_source=chatgpt.com)

### Enforcing

Policy được apply và deny thực sự xảy ra.

### Permissive

Policy violations được log nhưng operation không bị SELinux block.

Useful cho controlled diagnostics.

### Disabled

SELinux security mechanism không hoạt động theo cách bình thường.

Đây không phải phương án troubleshooting đầu tiên.

---

# 35. SELinux troubleshooting anti-pattern

Anti-pattern:

```text
Application không chạy
→ set SELinux permissive
→ chạy được
→ disable SELinux permanently
```

Điều này biến symptom thành security degradation.

Approach tốt:

```text
Confirm denial
→ identify source/target/context
→ determine expected access
→ fix labeling/configuration/policy correctly
→ retain enforcing mode
```

---

# 36. SELinux verification

Read-only:

```bash
getenforce
sestatus
ls -Z /path
ps -eZ
```

Denial investigation thường liên quan audit records. Red Hat chỉ ra AVC denial records có thể xuất hiện trong `/var/log/audit/audit.log`; `ausearch` cũng có thể được dùng để tra SELinux denials. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/getting-started-with-selinux_using-selinux?utm_source=chatgpt.com)

Một troubleshooting clue rất mạnh:

```text
DAC permissions look correct
service config looks correct
application reports Permission denied
AVC denial exists
```

→ SELinux phải được xem xét.

---

# 37. `firewalld`

`firewalld` là host firewall service.

Mục tiêu:

```text
Control traffic
Reduce exposed attack surface
Permit required communication
Deny unnecessary access
```

Trong RHEL 10, Red Hat tiếp tục mô tả `firewalld` là dynamic firewall daemon dựa trên concepts như **zones, policies và services**, với separate **runtime** và **permanent** configuration. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld?utm_source=chatgpt.com)

---

# 38. Firewall zones

Zone biểu diễn mức trust/context của network connection/interface/source.

Concept:

```text
NIC / connection
      ↓
Zone
      ↓
Allowed services/ports/rules
```

Ví dụ administrator có thể có:

```text
public
internal
trusted
```

Nhưng phải inspect environment thực tế; không đoán zone dựa trên name.

---

# 39. Services và ports trong `firewalld`

Bạn có thể permit bằng predefined service hoặc port/rule phù hợp.

Ví dụ conceptual:

```text
HTTP
→ TCP/80

HTTPS
→ TCP/443

SSH
→ TCP/22
```

Nhưng “service name” không phải chỉ là label đẹp; predefined firewalld service có thể encapsulate các settings cần thiết.

---

# 40. Runtime vs permanent firewall configuration

Đây là distinction cực kỳ quan trọng.

### Runtime

State đang hoạt động hiện tại.

### Permanent

Configuration được lưu để dùng sau reload/restart.

Có thể xảy ra:

```text
Runtime allows TCP/8080
Permanent does not
```

Sau reload:

```text
TCP/8080 disappears
```

Ngược lại, permanent config được chỉnh nhưng runtime chưa apply cũng tạo confusion.

Red Hat xác nhận runtime config của `firewalld` không persist qua service reload/restart; startup state được xây từ permanent configuration. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld?utm_source=chatgpt.com)

---

# 41. Firewall troubleshooting logic

Một service không reachable:

```text
Client
  ↓
Network
  ↓
Server interface
  ↓
firewalld
  ↓
listening socket
  ↓
application
```

Nếu:

```bash
curl http://127.0.0.1:8080
```

works,

và:

```bash
ss -lnt
```

thấy port 8080,

nhưng remote client không connect được,

thì candidates gồm:

```text
Host firewall
Network firewall
Routing
Security policy
Application bind address
```

Không thể chỉ từ symptom kết luận `firewalld`.

---

# 42. IP addressing

Network interface cần Layer 3 identity.

IPv4 address thường gồm:

```text
IP address
Prefix length
```

Ví dụ:

```text
192.0.2.10/24
```

Prefix `/24` quyết định network boundary.

Host dùng subnet/prefix để xác định destination:

```text
local subnet?
```

hay:

```text
need routing?
```

---

# 43. Interface

Linux có network interfaces:

```text
physical NIC
virtual NIC
loopback
bridge
bond
VLAN interface
```

Foundation Day 2 chủ yếu cần biết:

```text
interface state
IP address
link state
```

Command quan trọng:

```bash
ip addr
ip link
```

---

# 44. Loopback

Thông thường:

```text
127.0.0.1
```

liên quan local host loopback.

Một application bound chỉ vào:

```text
127.0.0.1:8080
```

có thể truy cập local nhưng không reachable qua external interface.

Đây là classic troubleshooting case.

Ví dụ `ss` có thể cho:

```text
127.0.0.1:8080
```

thay vì:

```text
0.0.0.0:8080
```

hoặc specific server IP.

---

# 45. Routing

Routing trả lời:

> Packet muốn đi đến destination X thì đi qua interface/next hop nào?

Routing table conceptual:

```text
Destination
Prefix
Next hop
Interface
Metric
```

Default route thường biểu diễn:

```text
0.0.0.0/0
```

tức những destinations không có specific route khác.

---

# 46. Longest-prefix concept

Kernel ưu tiên route cụ thể phù hợp nhất.

Ví dụ:

```text
10.20.0.0/16 via A
10.20.30.0/24 via B
0.0.0.0/0 via C
```

Destination:

```text
10.20.30.10
```

sẽ match `/24` cụ thể hơn `/16`.

Đây là networking principle rất quan trọng khi troubleshooting.

---

# 47. `ip route`

Read-only:

```bash
ip route
```

Bạn có thể thấy conceptual output:

```text
default via 192.0.2.1 dev eth0
192.0.2.0/24 dev eth0 ...
```

Từ đó hỏi:

```text
Có default route?
Correct gateway?
Destination có more-specific route?
Đúng interface?
```

---

# 48. DNS resolution

DNS chuyển:

```text
hostname
↓
IP address
```

Ví dụ:

```text
app.example.com
↓
192.0.2.50
```

Điểm cực kỳ quan trọng:

```text
DNS failure ≠ connectivity failure
```

Ví dụ:

```bash
ping 192.0.2.50
```

works,

nhưng:

```bash
curl https://app.example.com
```

fails vì DNS.

---

# 49. DNS troubleshooting layers

Tách:

```text
Can reach DNS server?
        ↓
Can resolve name?
        ↓
Resolved IP correct?
        ↓
Can reach resolved IP?
        ↓
Application works on destination port?
```

Đừng chỉ test hostname rồi nói:

> Network down.

Name resolution là một dependency riêng.

---

# 50. TCP và UDP

## TCP

Connection-oriented transport.

Conceptual flow:

```text
Client
→ connection establishment
→ data exchange
→ connection termination
```

Có reliability/order mechanisms.

Common services:

```text
SSH
HTTP
HTTPS
```

thường dùng TCP.

## UDP

Connectionless transport ở transport layer.

Không có cùng connection setup/reliability semantics như TCP.

Nhiều services có thể sử dụng UDP, TCP hoặc cả hai tùy protocol.

---

# 51. Ports

IP address xác định host/interface endpoint.

Port giúp xác định service endpoint trên host.

Concept:

```text
192.0.2.10:22
192.0.2.10:443
192.0.2.10:8080
```

Port + protocol matter:

```text
TCP/53
UDP/53
```

là hai transport endpoints khác nhau dù cùng number.

---

# 52. Listening socket

Một server application thường phải bind/listen trên socket.

Flow:

```text
Process
   ↓ bind()
IP:port
   ↓ listen
Client can attempt connection
```

Nếu process running nhưng không listening expected port:

```text
service active
```

không chứng minh functional availability.

---

# 53. `ss`

`ss` là tool quan trọng để inspect sockets.

Common read-only pattern:

```bash
ss -lnt
```

Concept:

```text
-l   listening
-n   numeric addresses/ports
-t   TCP
```

UDP:

```bash
ss -lnu
```

Process information có thể yêu cầu privilege:

```bash
ss -lntp
```

Bạn có thể xác định:

```text
Port có listening?
TCP hay UDP?
Bind address nào?
Process nào?
```

---

# 54. `ping`

`ping` thường dùng ICMP Echo để kiểm tra IP reachability/path behavior.

Nhưng rất quan trọng:

```text
ping fails
```

không luôn có nghĩa:

```text
host down
```

ICMP có thể bị firewall/filter.

Và:

```text
ping succeeds
```

không chứng minh application port hoạt động.

Do đó:

```text
Ping success ≠ application success
```

---

# 55. `traceroute` và `tracepath`

Hai tools này giúp quan sát path/hops và network behavior.

Concept:

```text
Host
 ↓
Gateway 1
 ↓
Router 2
 ↓
Router 3
 ↓
Destination
```

Nhưng incomplete traceroute không luôn chứng minh fault ở hop cuối hiển thị, vì intermediate devices có thể không respond probe.

Dùng như diagnostic evidence, không phải absolute truth.

---

# 56. `curl`

`curl` cực kỳ hữu ích vì có thể test application-layer behavior.

Ví dụ:

```bash
curl http://127.0.0.1:8080
```

giúp test local application.

```bash
curl http://server-ip:8080
```

test bằng IP.

```bash
curl https://service.example.com
```

thêm DNS/TLS/application dependencies.

Một good troubleshooting progression:

```text
localhost
→ local interface IP
→ remote client
→ DNS name
```

giúp isolate layer.

---

# 57. Network troubleshooting matrix

Nếu:

```text
ip addr            correct
ip route           correct
ping gateway       works
DNS resolution     works
ss shows port      listening
local curl         works
remote curl        fails
```

thì likely investigation chuyển sang:

```text
bind address
host firewall
network ACL/firewall
routing return path
load balancer
```

Nếu:

```text
IP curl works
hostname curl fails
```

ưu tiên DNS/name-based behavior.

Nếu:

```text
port not listening
```

đừng sửa firewall trước; trước hết investigate service/application.

---

# 58. Storage conceptual model

Storage là topic rất dễ nhầm nếu học command trước architecture.

Flow đơn giản:

```text
Disk / Block Device
        ↓
Partition
        ↓
Filesystem
        ↓
Mount
        ↓
Mount point
        ↓
Files/directories
```

Có environments phức tạp hơn:

```text
Disk
 ↓
Partition
 ↓
LVM Physical Volume
 ↓
Volume Group
 ↓
Logical Volume
 ↓
Filesystem
```

LVM không được syllabus Day 2 liệt kê rõ, nên ở đây chỉ cần awareness, không biến nó thành trọng tâm.

---

# 59. Block device

Linux đại diện storage devices trong `/dev`.

Ví dụ conceptual:

```text
/dev/sda
/dev/sdb
/dev/nvme0n1
```

Partitions có thể là:

```text
/dev/sda1
/dev/sda2
```

hoặc NVMe naming khác.

Không được hard-code assumption về device name trong enterprise automation nếu chưa kiểm tra environment.

---

# 60. Partition

Partition chia block device thành regions.

Ví dụ:

```text
Disk
├── partition 1
├── partition 2
└── partition 3
```

Partition chưa đồng nghĩa dữ liệu đã usable qua normal filesystem path.

Nó thường cần filesystem hoặc được dùng làm layer khác như LVM.

---

# 61. Filesystem

Filesystem tổ chức files/directories và metadata trên storage.

RHEL hiện hỗ trợ các local filesystems như **XFS** và **ext4**, bên cạnh network filesystems và các storage abstractions khác. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/managing_file_systems/managing_file_systems?utm_source=chatgpt.com)

Các filesystem có different characteristics.

Ở Day 2, trọng tâm không phải filesystem engineering mà là:

```text
Identify filesystem
Mount
Capacity
Inodes
Persistent mount
Troubleshoot
```

---

# 62. Mount

Mount gắn filesystem vào Linux directory tree.

Ví dụ:

```text
Filesystem on block device
       ↓
mounted at
       ↓
/data
```

Sau đó application truy cập:

```text
/data/file1
```

Red Hat mô tả mount point là nơi filesystem được attach vào directory tree; khi filesystem được mount lên một directory, nội dung directory vốn có bên dưới tạm thời không accessible qua path đó. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/mounting-file-systems_managing-file-systems?utm_source=chatgpt.com)

Đây là detail rất quan trọng.

---

# 63. Hidden underlying data problem

Suppose:

```text
/data
```

có 10 GB data trên root filesystem.

Sau đó bạn mount another filesystem lên:

```text
/data
```

Old files bên dưới không biến mất, nhưng bị mount che.

Có thể gây:

```text
Disk usage confusion
Hidden data
Unexpected capacity problem
```

Đây là practical Operations pitfall.

---

# 64. `findmnt`, `lsblk`, `df`

Useful inspection concepts:

```bash
lsblk
findmnt
df -h
```

`lsblk` giúp nhìn block-device topology.

`findmnt` giúp nhìn mounts.

Red Hat hiện đề xuất `findmnt` để list mounted filesystems. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/mounting-file-systems_managing-file-systems?utm_source=chatgpt.com)

`df` giúp đánh giá filesystem capacity.

---

# 65. `/etc/fstab`

`/etc/fstab` dùng để define persistent mount behavior.

Red Hat mô tả mỗi entry với sáu fields:

```text
device
mount point
filesystem
options
dump field
fsck order
```

và `systemd-fstab-generator` chuyển entries thành systemd mount units. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/assembly_persistently-mounting-file-systems_managing-file-systems?utm_source=chatgpt.com)

Example:

```text
UUID=<uuid> /data xfs defaults 0 0
```

---

# 66. Tại sao dùng UUID?

Device paths có thể thay đổi trong một số environments.

Persistent identifiers như UUID giúp mapping ổn định hơn.

Example:

```bash
lsblk -f
```

có thể cho:

```text
NAME  FSTYPE UUID
...
```

RHEL documentation cũng sử dụng UUID trong examples cho persistent mount. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/assembly_persistently-mounting-file-systems_managing-file-systems?utm_source=chatgpt.com)

---

# 67. `/etc/fstab` là high-risk configuration

Sai entry có thể gây:

```text
mount failure
application filesystem unavailable
boot delays
boot/recovery issues
wrong filesystem mounted
```

Vì vậy workflow không phải:

```text
vi /etc/fstab
→ reboot
→ hope
```

Mà phải là:

```text
Record current state
→ Backup file
→ Edit carefully
→ Reload relevant systemd configuration
→ Validate
→ Test mount
→ Verify
→ Only then rely on reboot persistence
```

RHEL docs hiện hướng dẫn sau khi thêm `fstab` entry thì thực hiện `systemctl daemon-reload` và test mount để xác nhận configuration. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/assembly_persistently-mounting-file-systems_managing-file-systems?utm_source=chatgpt.com)

---

# 68. Disk capacity

`df -h` thường dùng để xem filesystem usage.

Concept:

```text
Size
Used
Available
Use%
Mount point
```

Ví dụ:

```text
Filesystem      Size Used Avail Use% Mounted on
/dev/...         20G  19G    1G  95% /
```

95% không tự động là incident; threshold phụ thuộc workload/runbook.

Nhưng capacity trend và free space phải được đánh giá.

---

# 69. Inodes

Filesystem không chỉ cần data blocks.

Nó còn cần metadata structures để represent files/directories; concept thường được biểu diễn qua inode usage.

Có thể xảy ra:

```text
Disk space available: 20 GB
Free inodes: 0
```

Kết quả:

```text
Cannot create new file
```

dù GB vẫn còn.

Đây là lý do syllabus explicitly yêu cầu **disk and inode capacity**.

---

# 70. `df -i`

Check inode utilization:

```bash
df -i
```

Nếu inode exhaustion:

```text
IUse% ≈ 100%
```

thì root cause thường là quá nhiều files, không nhất thiết files lớn.

Examples:

```text
millions of temp files
small cache files
mail queue files
application-created tiny files
```

Không được giải quyết bằng cách xóa ngẫu nhiên.

Phải xác định owner/workload và retention mechanism.

---

# 71. Storage troubleshooting framework

Symptom:

```text
Application cannot write
```

Possible chain:

```text
Filesystem mounted?
        ↓
Read-write or read-only?
        ↓
Capacity free?
        ↓
Inodes free?
        ↓
DAC permissions?
        ↓
SELinux?
        ↓
Underlying device healthy?
        ↓
Application-specific limit?
```

Nếu chỉ chạy:

```bash
df -h
```

rồi kết luận storage healthy là chưa đủ.

---

# 72. Logging — tại sao logs quan trọng?

Logs giúp tái dựng:

```text
What happened?
When?
Where?
Which component?
Which user/process?
Success or failure?
What error?
```

Trong enterprise:

```text
Log = troubleshooting evidence + security evidence + audit evidence
```

Restart trước khi xem log đôi khi làm mất contextual state quý giá.

---

# 73. `journald` và `journalctl`

`systemd-journald` thu thập journal events.

`journalctl` là query interface.

Common examples:

```bash
journalctl
journalctl -u sshd
journalctl -u httpd
journalctl -b
```

Concept:

```text
journalctl -u <unit>
```

lọc logs theo `systemd unit`.

```text
-b
```

liên quan boot hiện tại/boot selection.

---

# 74. Timeline thinking

Troubleshooting logs nên bắt đầu bằng **time window**.

Incident:

```text
09:42 service failed
```

Bạn không nên dump hàng triệu log lines.

Bạn muốn:

```text
09:35
→ preceding change/events

09:42
→ failure

09:45
→ recovery attempt
```

Correlation by timestamp là kỹ năng Operations cực kỳ quan trọng.

---

# 75. syslog và `rsyslog`

Syslog là logging convention/protocol architecture lâu đời trong UNIX/Linux.

Trên RHEL, `rsyslog` thường xử lý syslog-style messages và ghi chúng tới files/configured destinations.

Red Hat documentation cho biết `rsyslog` phân loại syslog messages và các log files phổ biến có thể nằm dưới `/var/log`; ví dụ `/var/log/messages`, `/var/log/secure`, `/var/log/cron`, nhưng actual files phụ thuộc services và configuration. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/configuring_basic_system_settings/index?utm_source=chatgpt.com)

Không được giả định mọi Linux distribution đều có exact same log path.

---

# 76. Journald và rsyslog không nên bị coi là đối thủ

Một server có thể sử dụng:

```text
systemd-journald
          ↓
     journal data
```

và đồng thời:

```text
systemd-journald / applications
          ↓
        rsyslog
          ↓
/var/log/... or remote log server
```

Architecture phụ thuộc configuration.

Ở Day 10 bạn sẽ học centralized logging sâu hơn.

---

# 77. Log severity và source

Khi đọc log, đừng chỉ search từ:

```text
error
```

Bạn cần xác định:

```text
Timestamp
Hostname
Service/process
PID
Severity
Message
Correlation context
```

Một log error không nhất thiết là root cause.

Ví dụ:

```text
Application error connecting database
```

có thể chỉ là downstream symptom của:

```text
network failure
database failure
DNS failure
```

---

# 78. Log rotation

Logs tăng liên tục có thể làm disk đầy.

**log rotation** giải quyết vòng đời log:

```text
Current log
    ↓ rotate
Old log
    ↓ compress
Older log
    ↓ retention
Expired
    ↓ remove
```

Typical Linux utility:

```text
logrotate
```

Concepts:

```text
frequency
size
retention count
compression
post-rotation handling
```

---

# 79. Log rotation failure

Có thể gặp:

```text
logrotate config incorrect
application keeps writing old inode/file
permissions incorrect
filesystem full
rotation never scheduled
high log generation rate
```

Symptom:

```text
/var full
```

Root cause có thể là logging subsystem/app, không phải storage infrastructure.

---

# 80. `grep`

`grep` lọc lines matching pattern.

Ví dụ:

```bash
grep "error" /var/log/example.log
```

Case insensitive:

```bash
grep -i "error" /var/log/example.log
```

Line number:

```bash
grep -n "error" file
```

---

# 81. Pipes trong log analysis

Từ Day 1 bạn đã học pipe. Day 2 đưa nó vào troubleshooting.

Ví dụ:

```bash
journalctl -u app.service | grep -i error
```

Data flow:

```text
journalctl output
      |
      | pipe
      v
grep filter
      |
      v
relevant lines
```

Cần nhớ pipeline có implications với exit status, sẽ nói ở Bash phần sau.

---

# 82. `awk`

`awk` mạnh ở field-based processing.

Suppose data:

```text
alice SUCCESS
bob FAILURE
charlie SUCCESS
```

Command:

```bash
awk '$2 == "FAILURE" {print $1}' file
```

Concept:

```text
$1 first field
$2 second field
```

Output:

```text
bob
```

Operational uses:

```text
extract columns
summarize log data
filter structured text
build reports
```

---

# 83. `sed`

`sed` là stream editor.

Common conceptual operations:

```text
substitute
delete
select
transform
```

Example:

```bash
sed 's/ERROR/WARNING/g' file
```

Nhưng lưu ý:

Nếu command chỉ đọc file và output stdout thì source file không thay đổi.

Các options như in-place editing có thể thay đổi file và phải cẩn thận trong production.

---

# 84. Regular expressions

Regex mô tả patterns.

Basic concepts:

```text
^       beginning of line
$       end of line
.       any character in many regex dialects
*       repetition
[...]   character class
```

Ví dụ:

```text
^ERROR
```

match line bắt đầu bằng `ERROR`.

```text
failed$
```

match line kết thúc bằng `failed`.

Regex syntax có differences giữa tools/dialects, vì vậy không nên giả định một pattern chạy identical ở mọi utility.

---

# 85. Regex trong Operations

Use cases:

```text
IP-like strings
IDs
timestamps
error codes
failed authentication records
service names
```

Nhưng tránh tạo regex quá clever đến mức operator khác không maintain được.

Infrastructure automation ưu tiên:

```text
correct
readable
testable
```

hơn “ngắn nhất có thể”.

---

# 86. Bash scripting

Bash script biến chuỗi manual commands thành repeatable procedure.

Một good Operations script cần có:

```text
Clear inputs
Predictable output
Validation
Error handling
Exit status
Logging
Safe defaults
Idempotency awareness
```

Day 2 chỉ yêu cầu foundation nhưng mindset phải đúng từ đầu.

---

# 87. Shebang

Typical:

```bash
#!/usr/bin/env bash
```

hoặc environment/runbook có thể yêu cầu explicit interpreter path.

Shebang cho system biết interpreter dùng để execute script khi file được chạy trực tiếp.

Không nên assume interpreter path nếu portability requirement khác nhau.

---

# 88. Bash variables

Example:

```bash
SERVICE_NAME="sshd"
BACKUP_DIR="/backup"
```

Reference:

```bash
echo "$SERVICE_NAME"
```

Best-practice foundation:

```bash
"$VARIABLE"
```

thường an toàn hơn unquoted expansion khi value có whitespace/globbing-sensitive characters.

---

# 89. Command substitution

Capture output:

```bash
HOSTNAME_VALUE=$(hostname)
```

Sau đó:

```bash
echo "$HOSTNAME_VALUE"
```

Concept:

```text
Run command
→ capture stdout
→ assign to variable
```

Nhưng cần nhớ stdout không luôn là clean structured data.

---

# 90. Quoting

Bash quoting là nguồn bug phổ biến.

### Double quotes

Variables được expand:

```bash
echo "$HOME"
```

### Single quotes

Content literal hơn:

```bash
echo '$HOME'
```

Output:

```text
$HOME
```

thay vì home path.

Quoting đặc biệt quan trọng khi xử lý:

```text
spaces
wildcards
user input
filenames
```

---

# 91. Bash control flow — `if`

Concept:

```bash
if condition; then
    action
else
    alternate_action
fi
```

Operations example:

```bash
if systemctl is-active --quiet sshd; then
    echo "OK"
else
    echo "CRITICAL"
fi
```

Đây chính là foundation của health-check script.

---

# 92. Loops

Bạn có thể repeat action cho multiple objects:

```bash
for service in sshd chronyd; do
    systemctl is-active "$service"
done
```

Loop giúp tránh duplicated code.

Nhưng automation trên multiple systems/resources làm blast radius tăng.

Do đó:

```text
Loop + privileged destructive action
```

phải được kiểm soát đặc biệt.

---

# 93. Exit codes

Unix/Linux commands trả về exit status.

Convention phổ biến:

```text
0       success
non-zero failure/special condition
```

Nhưng exact non-zero values là command-specific.

Sau command:

```bash
some_command
echo $?
```

`$?` chứa exit status gần nhất.

---

# 94. Tại sao exit code rất quan trọng?

Suppose backup script:

```bash
tar ...
echo "Backup completed"
```

Nếu `tar` fail nhưng script vẫn print:

```text
Backup completed
```

human có thể hiểu sai.

Monitoring system cũng có thể coi task successful.

Script tốt phải:

```text
Run operation
→ inspect return status
→ verify artifact
→ report correct state
→ exit appropriately
```

---

# 95. Exit codes và pipelines

Đây là kiến thức bổ sung nhưng rất quan trọng.

Suppose:

```bash
command1 | command2
```

Pipeline exit behavior cần hiểu trước khi dựa vào `$?`.

Bash có features như:

```bash
set -o pipefail
```

để giúp detect failures bên trong pipeline.

Nhưng options kiểu:

```bash
set -e
```

có nuanced behavior và không nên dùng mù quáng nếu chưa hiểu semantics.

Automation safety quan trọng hơn việc copy “strict mode” từ Internet.

---

# 96. Error handling

Error handling nghĩa là dự đoán:

```text
What can fail?
How do I detect it?
What evidence do I keep?
Can I retry?
Should I stop?
Can I rollback?
```

Ví dụ health-check:

```text
Unable to execute check
```

khác:

```text
Check executed and service is unhealthy
```

Một script tốt phải phân biệt hai tình trạng này.

---

# 97. Safe scripting

Một script có privilege nên kiểm tra assumptions trước khi modify.

Concept:

```text
Validate input
→ Validate source exists
→ Validate destination
→ Confirm permissions
→ Execute
→ Verify
→ Exit
```

Đặc biệt tránh:

```bash
rm -rf "$VARIABLE"
```

nếu variable có khả năng empty/incorrect và chưa validate.

Day 2 lab phải nằm trong lab/non-production environment theo syllabus safety philosophy. High-risk operations trong chương trình yêu cầu authorization, rollback và supervision thích hợp. Infrastructure_Operations_14_Da…

---

# 98. `cron`

`cron` dùng để schedule recurring jobs.

Concept:

```text
Scheduler
  ↓
Time match
  ↓
Command/script executes
  ↓
Output/error/result
```

Cron entry thường có time fields tương ứng:

```text
minute
hour
day-of-month
month
day-of-week
command
```

---

# 99. Cron mindset

Không chỉ hỏi:

> Cron entry có tồn tại không?

Phải hỏi:

```text
Was it triggered?
Under which user?
What environment?
Did script run?
What exit result?
Where stdout/stderr went?
Did expected artifact appear?
```

---

# 100. Cron và timezone

Schedules phụ thuộc system time/timezone.

Nếu ticket nói:

```text
Backup failed at 02:00
```

cần biết:

```text
02:00 local timezone nào?
Server timezone?
Log timezone?
Monitoring timezone?
```

Đây sẽ trở nên cực kỳ quan trọng khi Day 10 học centralized log correlation.

---

# 101. Cron environment

Cron thường có environment khác interactive shell.

Do đó script nên:

```text
use explicit paths where appropriate
set required variables
not assume shell profile was loaded
record useful logs
```

Classic failure:

```text
manual:
backup-tool works

cron:
backup-tool: command not found
```

vì PATH khác.

---

# 102. Compression vs archiving

Hai concepts không giống nhau.

### Archiving

Gộp nhiều files/directories thành một logical archive.

### Compression

Giảm size của byte stream/data.

Ví dụ `tar` chủ yếu là archive tool, có thể kết hợp compression.

Concept:

```text
Files
  ↓ tar
Archive
  ↓ compression
Compressed archive
```

---

# 103. `tar`

Typical conceptual backup:

```bash
tar -cf backup.tar /path
```

Common flags:

```text
-c create
-x extract
-t list
-f archive filename
```

Compression flags phụ thuộc algorithm/tool usage.

Điều quan trọng hơn syntax là:

```text
What is source?
What is included?
What is excluded?
Where is archive stored?
Can it be read?
Can it restore?
```

---

# 104. Archive is not automatically a valid backup

Nếu command:

```bash
tar -cf backup.tar /data
```

return success, ta vẫn chưa chứng minh:

```text
all intended data included
archive stored safely
archive integrity acceptable
restore works
```

Backup validity cần verification.

---

# 105. `rsync`

`rsync` dùng để synchronize/copy files efficiently.

Concept:

```text
SOURCE
   ↓
compare
   ↓
DESTINATION
```

Repeated runs thường chuyển differences thay vì blindly copy toàn bộ theo cùng cách simple copy.

Nó rất hữu ích cho backup-like workflows, nhưng `rsync` không tự động biến một destination thành full enterprise backup solution.

---

# 106. `rsync` safety

Bạn phải rất rõ:

```text
Source
Destination
Trailing slash semantics
Delete behavior
Permissions/ownership
Remote endpoint
```

Đặc biệt destructive options có thể khiến destination data bị remove.

Do đó trước privileged `rsync`:

```text
verify source
verify destination
use dry-run if applicable
record command
understand delete semantics
```

---

# 107. Backup fundamentals

Backup cần trả lời:

```text
What are we protecting?
From what failure?
How often?
Where is backup?
Who can access it?
How long retained?
How do we restore?
How do we verify?
```

Day 2 chỉ giới thiệu basic backup/restore, nhưng những câu hỏi này là foundation cho AWS backup/DR Day 5 và HYCU Day 7.

---

# 108. Backup versus copy

Một copy có thể hữu ích nhưng không nhất thiết đáp ứng backup requirements.

Nếu:

```text
source corrupted
```

và automated sync lập tức overwrite destination với corrupted state:

```text
copy exists
```

nhưng recoverability có thể đã mất.

Backup architecture sâu hơn cần:

```text
versioning
retention
isolation
integrity
restore testing
```

Những topics nâng cao sẽ xuất hiện sau.

---

# 109. Basic restore procedure

Restore không phải chỉ:

```text
extract archive
```

Restore workflow:

```text
Identify recovery point
→ Ensure target location safe
→ Restore isolated/test target if possible
→ Verify files
→ Verify ownership/permissions
→ Verify application usability
→ Compare expected data
→ Document observed result
```

---

# 110. Restore verification

Good verification có thể gồm:

```text
File exists
Expected size/count
Ownership/permissions
Readable content
Checksum if applicable
Application can consume it
```

Syllabus Day 2 yêu cầu basic restore procedures; Assignment sau đó yêu cầu validate backup và restore results. Infrastructure_Operations_14_Da…

Điều đó cho thấy tư duy của course:

```text
Backup job success ≠ recovery proven
```

---

# 111. Health checks

Health check là structured set of observations xác định server/service có đang trong acceptable state không.

Không nên chỉ có:

```bash
systemctl is-active app
```

Một service có thể active nhưng unusable.

Health model tốt hơn:

```text
Host reachable?
        ↓
Required filesystem mounted?
        ↓
Capacity acceptable?
        ↓
Service state healthy?
        ↓
Process exists?
        ↓
Port listening?
        ↓
Local functional request works?
        ↓
Dependencies reachable?
```

---

# 112. Liveness vs functional health

Conceptual distinction:

### Liveness

```text
Process tồn tại?
Service active?
```

### Functional health

```text
Service thực sự đáp ứng expected function?
```

Ví dụ:

```text
httpd active
TCP/443 listening
```

chưa chắc website healthy.

Backend/database dependency có thể fail.

---

# 113. Baseline

Bạn chỉ biết “abnormal” nếu có hình dung “normal”.

Baseline có thể gồm:

```text
Expected services
Expected ports
Expected mounts
Normal disk utilization
Normal memory range
Normal routing
Expected repository configuration
SELinux mode
Firewall zone
```

Không có baseline, troubleshooting dễ thành guesswork.

---

# 114. Operational troubleshooting — triết lý cốt lõi của Day 2

Troubleshooting không phải:

```text
try commands until it works
```

Mà là scientific workflow:

```text
Symptom
  ↓
Scope
  ↓
Evidence
  ↓
Hypothesis
  ↓
Safe test
  ↓
Root cause
  ↓
Authorized corrective action
  ↓
Verification
  ↓
Monitoring
  ↓
Documentation
```

---

# 115. Bước 1 — Define symptom

Bad description:

```text
Linux broken.
```

Good description:

```text
Client 192.0.2.100 cannot establish TCP connection
to server 192.0.2.10:8443 since approximately 10:15.
SSH to server remains available.
```

Good symptom definition đã giảm search space đáng kể.

---

# 116. Bước 2 — Determine scope

Questions:

```text
One user or all?
One server or cluster?
One service or whole OS?
Local access works?
Remote access fails?
IP works but hostname fails?
Only after recent change?
```

Scope giúp xác định domain.

---

# 117. Bước 3 — Capture state before changing

Trước configuration-changing command, ưu tiên read-only evidence:

```bash
date
hostname
systemctl status <service>
journalctl -u <service>
ss -lntp
ip addr
ip route
df -h
df -i
getenforce
```

Chỉ chạy commands phù hợp authority và privacy requirements.

---

# 118. Bước 4 — Layer-by-layer isolation

Một general Linux service flow:

```text
Host alive?
    ↓
OS healthy?
    ↓
Storage available?
    ↓
Network configured?
    ↓
Service active?
    ↓
Process alive?
    ↓
Port listening?
    ↓
Local request works?
    ↓
Firewall permits?
    ↓
Remote request works?
    ↓
DNS works?
    ↓
Application dependency works?
```

Không nhất thiết lúc nào cũng bắt đầu đúng thứ tự này; nhưng phải có logic.

---

# 119. Bước 5 — Form hypothesis

Ví dụ:

Observed:

```text
systemctl: active
ss: port 8080 LISTEN on 127.0.0.1
local curl localhost:8080 works
remote curl fails
```

Hypothesis mạnh:

```text
Application only bound to loopback.
```

Đây tốt hơn:

```text
Maybe firewall.
```

vì evidence directly supports hypothesis.

---

# 120. Bước 6 — Least-risk test

Giả sử nghi firewall.

Risky test:

```text
disable entire firewall
```

Safer test:

```text
inspect active zone
inspect allowed services/ports
confirm interface-to-zone mapping
compare expected rule
```

Luôn ưu tiên read-only verification.

---

# 121. Bước 7 — Correct root cause, không chỉ symptom

Example:

```text
Filesystem full because one application log grows without rotation.
```

Temporary symptom fix:

```text
delete random logs
```

Root-cause fix có thể cần:

```text
correct log retention/rotation
fix excessive application logging
increase capacity if justified
```

Tùy ownership, L1/L2 có thể chỉ cleanup approved files và escalate application behavior.

---

# 122. Bước 8 — Verification sau corrective action

Không nói:

> "Đã restart nên xong."

Phải kiểm tra:

```text
Service state
Process
Port
Functional request
Logs
Resource state
Client connectivity
Monitoring state
```

Verification phải tương ứng với original symptom.

Nếu complaint là remote HTTPS failure thì chỉ:

```bash
systemctl status
```

không đủ.

---

# 123. Bước 9 — Observe stability

Một service có thể:

```text
start successfully
```

rồi 30 giây sau crash.

Vì vậy recovery verification phải đủ để xác nhận:

```text
stable
```

không chỉ “momentarily active”.

---

# 124. Evidence collection

Một troubleshooting evidence package tốt thường gồm:

| Evidence                | Mục đích                     |
| ----------------------- | ---------------------------- |
| Timestamp               | Xây incident timeline        |
| Host identity           | Chứng minh đúng system       |
| Service status          | Current runtime state        |
| Relevant logs           | Failure/root-cause clues     |
| `ss` output             | Port/socket state            |
| Network state           | IP/routing                   |
| Firewall state          | Access control               |
| SELinux state/log       | MAC decision                 |
| Storage usage           | Capacity/mount state         |
| Before/after comparison | Chứng minh corrective action |
| Ticket/change ID        | Audit linkage                |

Không cần dump mọi thứ. Evidence tốt là:

```text
Relevant
Timestamped
Readable
Minimal but sufficient
Secure
```

---

# 125. Security implications và least privilege

Day 2 có rất nhiều commands có thể dùng bằng root.

Điều đó không có nghĩa:

```text
always sudo everything
```

Read-only queries thường nên dùng minimum privilege có thể.

Privileged operations gồm:

```text
package installation
service modification
firewall changes
mount configuration
SELinux policy/context changes
system configuration
```

cần:

```text
authorization
least privilege
change awareness
auditability
```

Không bao giờ đưa secrets vào:

```text
shell history
command line
logs
screenshots
tickets
```

nếu có thể tránh.

---

# 126. Common operational mistakes

Các lỗi tư duy cần loại bỏ trong Day 2 có thể gom thành một danh sách:

1. Service lỗi là restart ngay trước khi thu evidence.
2. Không chạy được application thì disable SELinux.
3. Remote không connect được thì disable `firewalld`.
4. Ping fail thì kết luận host down.
5. Ping success thì kết luận application healthy.
6. `systemctl active` thì kết luận service functional.
7. `df -h` còn dung lượng thì kết luận storage không có vấn đề, bỏ qua inode.
8. Backup command exit `0` thì kết luận restore chắc chắn được.
9. Script chạy manual thì mặc định cron cũng chạy.
10. Package tồn tại trên Internet thì tự ý thêm external repository.
11. Dùng `kill -9` làm lựa chọn đầu tiên.
12. Chỉnh `/etc/fstab` rồi reboot để “test”.
13. Chỉ xem final error line mà không xây timeline.
14. Xóa logs để giải phóng disk mà không xác định retention/root cause.
15. Thay đổi nhiều variables cùng lúc, khiến không biết action nào giải quyết issue.
16. Không lưu before/after evidence.
17. Fix ngoài authorized scope thay vì escalate.

---

# 127. Failure-domain thinking

Đây là kiến thức bổ sung cực kỳ quan trọng.

Một symptom nên được map vào possible domains:

```text
Software/package
Process
systemd/service
Configuration
Authentication/authorization
SELinux
Network
Firewall
Storage
Resource capacity
Logging
Automation/scheduling
Backup
External dependency
```

Sau đó eliminate từng domain bằng evidence.

Ví dụ:

```text
Web app unavailable
```

Không phải một root cause.

Nó chỉ là symptom có thể xuất phát từ mọi domain kể trên.

---

# 128. Read-only trước, change sau

Operational troubleshooting nên chia commands thành hai classes.

### Read-only / observation

Examples:

```bash
systemctl status
journalctl
ps
ss
ip addr
ip route
df
findmnt
getenforce
rpm -q
dnf repolist
```

### Configuration-changing

Examples:

```bash
dnf install
systemctl restart
firewall configuration changes
setenforce
mount
editing /etc/fstab
package remove/update
```

Read-only actions thường low-risk hơn và nên đi trước.

---

# 129. L1/L2 escalation boundary

Boundary thực tế phụ thuộc organization, nhưng conceptual model của Day 2 là:

### Có thể nằm trong L1

```text
Collect evidence
Check service state
Check logs
Check capacity
Check network state
Run approved health checks
Restart approved low-risk service theo runbook
Perform documented verification
```

### Thường thuộc L2 hoặc L1 với explicit runbook

```text
Package maintenance
Firewall rule adjustment
Filesystem mount correction
SELinux context correction
Application config changes
Cron/script correction
Backup/restore procedure
```

### Thường cần higher-level/escalation/change approval

```text
Kernel-level issue
Unknown filesystem corruption
Production partition manipulation
Unapproved repository
Security policy redesign
Major SELinux custom policy
Complex network routing
Repeated unexplained crash
Critical package downgrade
Potential compromise
Destructive recovery
```

Không phải skill yếu khi escalate.

Ngược lại, **biết escalation boundary là một Operations skill**.

---

# 130. Một scenario tích hợp toàn bộ Day 2

Giả sử ticket:

```text
Incident:
Internal application https://app01.example.com:8443
is unavailable after maintenance.
```

Bạn SSH vào server được.

## Stage 1 — Identity và timestamp

```bash
date
hostname
```

Xác nhận đúng host và timeline.

## Stage 2 — Service

```bash
systemctl status app.service
```

Giả sử:

```text
failed
```

## Stage 3 — Logs

```bash
journalctl -u app.service
```

Log:

```text
Failed to write /data/app/runtime.pid:
No space left on device
```

## Stage 4 — Storage

```bash
df -h /data
```

Output:

```text
60% used
```

Nếu dừng đây bạn có thể nghĩ log sai.

Nhưng:

```bash
df -i /data
```

Output:

```text
100% inode use
```

Root cause direction rõ hơn.

## Stage 5 — Investigate

Bạn xác định một temp directory chứa số lượng file cực lớn.

Không xóa ngay nếu chưa biết ownership.

Kiểm tra:

```text
application ownership
retention policy
change history
runbook
```

## Stage 6 — Authorized correction

Sau approved cleanup/correction:

```bash
df -i /data
```

inode availability trở lại.

## Stage 7 — Service recovery

```bash
sudo systemctl start app.service
```

## Stage 8 — Verify process/socket

```bash
systemctl status app.service
ss -lnt
```

Giả sử port 8443 listening.

## Stage 9 — Local application verification

```bash
curl ...
```

Local response healthy.

## Stage 10 — Remote still fails

Bây giờ inspect:

```text
firewalld
routing
bind address
```

Suppose app listens correctly, nhưng firewall rule mất sau maintenance.

Investigate runtime/permanent state trước khi change.

## Stage 11 — Security

Nếu application log sau đó xuất hiện `Permission denied`, kiểm tra:

```bash
getenforce
```

và relevant AVC evidence trước khi kết luận SELinux.

## Stage 12 — Evidence

Record:

```text
Initial failure
journal error
df -h
df -i
root cause
approved cleanup
service recovery
socket verification
functional check
firewall verification
final status
```

Scenario này cho thấy topics Day 2 không độc lập. Chúng nối thành một troubleshooting workflow duy nhất.

---

# 131. Mental model cần thuộc sau Module Lý thuyết

Khi application có vấn đề, bạn nên tự động hình dung:

```text
                    APPLICATION
                         |
          +--------------+--------------+
          |              |              |
       SERVICE         NETWORK        STORAGE
          |              |              |
       systemd           IP          filesystem
          |            routing          mount
       process            DNS           fstab
          |            TCP/UDP         space
       resource          port           inode
          |              |
          +------ SECURITY ------+
                 |           |
              SELinux    firewalld

                         |
                       LOGS
                journal / syslog
                         |
                     ANALYSIS
             grep / awk / sed / regex
                         |
                    AUTOMATION
                 Bash / cron
                         |
                  BACKUP/RESTORE
                 tar / rsync
```

Nếu bạn nhớ được diagram này, gần như toàn bộ syllabus Day 2 đã có chỗ trong mental model.

---

# 132. Relationship giữa topics

Không học:

```text
systemctl
ss
firewall-cmd
getenforce
df
journalctl
```

như sáu command riêng biệt.

Hãy học dependency:

```text
systemctl
   ↓
"Service active?"

ps
   ↓
"Process exists?"

ss
   ↓
"Port listening?"

curl
   ↓
"Application responding?"

firewalld
   ↓
"Traffic allowed?"

SELinux
   ↓
"Application access permitted?"

df/findmnt
   ↓
"Required storage healthy?"

journalctl
   ↓
"What happened and why?"
```

Đây mới là Infrastructure Operations.

---

# 133. Day 2 Theory — Verification framework tổng hợp

Khi đánh giá một RHEL host, một baseline read-only assessment có thể được conceptualize như:

```text
IDENTITY
hostname / OS / timestamp

PACKAGE
installed version / repository

SERVICE
systemd state / enablement

PROCESS
PID / owner / resource

SECURITY
SELinux / firewall

NETWORK
interface / IP / route / DNS / sockets

STORAGE
devices / mounts / space / inode

LOGS
journal / syslog / relevant timeline

AUTOMATION
script / exit status / schedule

BACKUP
artifact / integrity / restore viability
```

Không nhất thiết chạy mọi check cho mọi incident. Chọn checks dựa trên symptom và hypothesis.

---

# 134. Day 2 Theory — Evidence framework

Mỗi investigation nên có thể trả lời sáu câu:

```text
WHEN?
Timestamp.

WHERE?
Hostname/system/environment.

WHAT?
Exact symptom.

WHY?
Root cause or current hypothesis.

WHAT DID YOU DO?
Authorized action.

DID IT WORK?
Objective verification.
```

Nếu escalation mà không trả lời được các câu này, receiving team thường phải troubleshoot lại từ đầu.

---

# 135. Day 2 Theory — Escalation format

Ví dụ weak escalation:

```text
App is down. Please check.
```

Strong escalation:

```text
Host: app01
Impact: HTTPS/8443 unavailable from internal clients
Start time: 10:15

Verified:
- Host reachable over SSH
- IP and default route present
- app.service failed
- journal reports write failure on /data
- /data has free block capacity but inode usage is 100%
- No configuration changes performed yet

Suspected domain:
Application-generated temp-file exhaustion

Requested action:
Application owner review retention behavior and approve cleanup.
```

Đây là kiểu evidence-driven escalation mà course muốn bạn hình thành.

---

# 136. Những gì syllabus chưa yêu cầu học sâu ở Day 2

Để tránh scope creep, Day 2 **không yêu cầu mastery sâu** về:

```text
Kernel internals
Advanced performance engineering
LVM architecture
RAID administration
NetworkManager deep configuration
Advanced nftables
Custom SELinux policy development
Advanced systemd dependency engineering
Enterprise backup architecture
Advanced Bash engineering
High Availability clustering
```

Ta có thể nhắc chúng khi cần giải thích dependency, nhưng không biến Day 2 thành khóa Linux expert-level.

Chương trình tự mô tả đây là accelerated foundation và operational-readiness program, không phải claim advanced product mastery. Infrastructure_Operations_14_Da…

---

# 137. Day 2 Module Lý thuyết — Checklist hoàn thành

Sau khi học xong phần này, về kiến thức bạn cần tự giải thích được toàn bộ chuỗi sau:

| Domain                    | Bạn phải hiểu                                        |
| ------------------------- | ---------------------------------------------------- |
| Linux/RHEL administration | Operational lifecycle và safe administration         |
| Package management        | `dnf`, `yum`, `rpm`, dependency                      |
| Repository                | Source, trust, enabled/disabled repositories         |
| Subscription              | Content/update access awareness                      |
| Environment               | Variables, export, `PATH`, inheritance               |
| Processes                 | PID, PPID, states, ownership                         |
| Signals                   | `SIGTERM`, `SIGKILL`, graceful vs forced             |
| Resources                 | CPU, memory, I/O awareness                           |
| `systemd`                 | Units, services, lifecycle                           |
| Boot                      | Targets, default target, recovery awareness          |
| SELinux                   | MAC, modes, contexts, AVC troubleshooting            |
| `firewalld`               | Zones, services, runtime/permanent config            |
| Networking                | IP, subnet, route, gateway                           |
| DNS                       | Name resolution separated from IP reachability       |
| TCP/UDP                   | Transport concepts và ports                          |
| Tools                     | `ip`, `ping`, `tracepath`/`traceroute`, `ss`, `curl` |
| Storage                   | Disk, partition, filesystem, mount                   |
| `/etc/fstab`              | Persistent mounting và risk                          |
| Capacity                  | Disk space và inode exhaustion                       |
| Logging                   | `journalctl`, syslog/rsyslog                         |
| Log lifecycle             | Rotation/retention                                   |
| Text processing           | `grep`, `awk`, `sed`                                 |
| Regex                     | Pattern matching foundation                          |
| Bash                      | Variables, quoting, control flow                     |
| Exit codes                | Success/failure signaling                            |
| Error handling            | Detect, report, stop/recover correctly               |
| `cron`                    | Scheduling và environment differences                |
| Compression/archive       | Distinction và purpose                               |
| `tar`                     | Archive/backup foundation                            |
| `rsync`                   | Synchronization/copy semantics                       |
| Restore                   | Recovery + verification                              |
| Health checks             | Liveness vs functional health                        |
| Troubleshooting           | Evidence-driven, least-risk workflow                 |
| Evidence                  | Before/after, logs, timestamps, ticket linkage       |
| Escalation                | Scope boundary và actionable handoff                 |

Nếu tất cả các dòng trên đều hiểu được **“nó là gì → tại sao cần → hoạt động thế nào → lỗi biểu hiện ra sao → verify bằng gì → troubleshoot thế nào → evidence gì cần lưu → khi nào escalate”**, thì phần **Module Lý thuyết Day 2 đã được cover đầy đủ theo syllabus**. Infrastructure_Operations_14_Da…

Bước tiếp theo hợp lý là **Module Thực hành Day 2**: chúng ta sẽ dựng một lab RHEL xuyên suốt, lần lượt thực hiện package → `systemd` → process → network/DNS → `firewalld` → SELinux → storage/`fstab` → logs → Bash health check → backup → `cron` → restore → failure injection → troubleshooting/evidence/escalation, thay vì làm các command rời rạc.
