# Day 2 — Module Lý thuyết

## Linux/RHEL System Administration, Maintenance, Troubleshooting & Evidence

Phần này trình bày **toàn bộ kiến thức Lý thuyết Day 2** theo `Infrastructure_Operations_14_Day_Syllabus.md`. Phạm vi chính thức gồm package management bằng `dnf/yum/rpm`, repository/subscription, environment variables, processes/signals/resource usage, `systemd`, boot targets, SELinux, `firewalld`, networking, storage/filesystem/mount, logging, `grep/awk/sed` và Regex, Bash, exit codes/error handling, `cron`, compression, `tar/rsync`, restore, health checks và operational troubleshooting. Phần additive bắt buộc mở rộng sang **RHEL maintenance**, pre/post checks, dependency/reboot assessment, recovery limitations và **audit-quality server log extraction**. Infrastructure_Operations_14_Da…

Một lưu ý quan trọng về version: syllabus nói chung về **RHEL**, không cố định RHEL 8, 9 hay 10. Tôi sẽ dạy theo các concept ổn định giữa các thế hệ và chỉ ra chỗ nào phụ thuộc version. Tài liệu Red Hat hiện hành cho RHEL 10 dùng `DNF` làm package-management tool; RHEL 9 cũng dùng DNF và Red Hat ghi rõ `yum` là compatibility alias cho `dnf`. Vì vậy trong môi trường enterprise, bước đầu tiên luôn là xác định chính xác `RHEL major/minor release` trước khi áp dụng runbook. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/dnf-commands-list?utm_source=chatgpt.com)

---

# 1. Mental model tổng thể của Day 2

Day 2 không phải là học một danh sách command Linux rời rạc. Bạn cần xây dựng mental model của một Linux/RHEL server như sau:

```text
                         +----------------------+
                         |     Users / Apps     |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         |     Processes        |
                         | PID / signals / env  |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | systemd / Services   |
                         | startup / dependency |
                         +----------+-----------+
                                    |
              +---------------------+----------------------+
              |                     |                      |
              v                     v                      v
      +---------------+      +-------------+       +-------------+
      | SELinux       |      | Networking  |       | Storage     |
      | permissions   |      | IP/DNS/port |       | FS/mount    |
      +-------+-------+      +------+------+       +------+------+
              |                     |                     |
              +---------------------+---------------------+
                                    |
                                    v
                         +----------------------+
                         | Logs / Evidence      |
                         | journal/syslog/audit |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | Automation           |
                         | Bash / cron          |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | Backup / Restore     |
                         +----------+-----------+
                                    |
                                    v
                         +----------------------+
                         | Troubleshooting      |
                         | Maintenance          |
                         | Escalation           |
                         +----------------------+
```

Một service failure có thể xuất phát từ bất kỳ layer nào trong sơ đồ đó. Ví dụ `httpd.service` failed có thể không phải vì `httpd` bị lỗi mà vì filesystem chứa document root không mount, disk full, SELinux context sai, port conflict, configuration syntax sai hoặc dependency không tồn tại.

Đó là lý do Infrastructure Operations cần **layer-by-layer troubleshooting**, không phải restart ngẫu nhiên.

---

# 2. Linux/RHEL system administration

## 2.1 What

`System administration` là việc duy trì server ở trạng thái:

```text
available
+ secure
+ correctly configured
+ supportable
+ observable
+ recoverable
+ auditable
```

Một administrator không chỉ làm cho service “running”. Bạn phải biết trạng thái mong muốn là gì, trạng thái thực tế là gì, chênh lệch ở đâu và có bằng chứng nào chứng minh.

Một RHEL server operationally healthy thường liên quan đồng thời đến:

```text
OS release
packages
repositories
kernel
processes
services
networking
storage
security controls
logs
scheduled jobs
backup
capacity
time
configuration
```

## 2.2 Configuration state và runtime state

Đây là distinction cực kỳ quan trọng.

Ví dụ một service:

```text
Enabled at boot ≠ Running now
```

Một firewall rule:

```text
Configured permanently ≠ Active in runtime
```

Một filesystem:

```text
Present in /etc/fstab ≠ Currently mounted
```

Một package:

```text
Installed ≠ Service correctly configured
```

Một cron job:

```text
Present in crontab ≠ Successfully executing
```

Do đó verification luôn phải kiểm tra cả **configuration state** và **effective/runtime state**.

---

# 3. Package management: RPM, DNF và YUM

## 3.1 RPM package là gì?

RHEL sử dụng **RPM package format**.

Một RPM package chứa về cơ bản:

```text
package metadata
+
payload/files
+
dependency information
+
scripts
+
signature/integrity metadata
```

Red Hat mô tả RPM package gồm metadata/header, payload và signature; RPM được tích hợp với higher-level package managers như DNF. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/packaging_and_distributing_software/Red_Hat_Enterprise_Linux-10-Packaging_and_distributing_software-en-US.pdf?utm_source=chatgpt.com)

Package thường được nhận diện theo mô hình gần với:

```text
Name-Version-Release.Architecture
```

Ví dụ:

```text
httpd-2.4.x-xx.el9.x86_64
```

Trong thực tế bạn cũng sẽ gặp khái niệm **NEVRA**:

```text
Name
Epoch
Version
Release
Architecture
```

`Epoch` thường không xuất hiện rõ trong tên file nhưng được dùng khi package manager so sánh version.

---

# 4. `rpm` — low-level package database tool

`rpm` làm việc trực tiếp với installed RPM database và RPM file.

Ví dụ read-only:

```bash
rpm -q httpd
```

Ý nghĩa:

```text
-q
query
```

Nếu installed:

```text
httpd-2.4.x-...
```

Nếu không:

```text
package httpd is not installed
```

Xem detailed metadata:

```bash
rpm -qi httpd
```

Xem files thuộc package:

```bash
rpm -ql httpd
```

Tìm package nào sở hữu một file:

```bash
rpm -qf /usr/sbin/sshd
```

Đây là command cực kỳ hữu ích khi troubleshooting:

```text
Unknown file
    ↓
rpm -qf
    ↓
Identify owning package
    ↓
Package/version/config investigation
```

Có thể kiểm tra differences so với RPM metadata baseline bằng:

```bash
rpm -V package_name
```

Nhưng phải hiểu đúng:

> `rpm -V` phát hiện một số thuộc tính/file khác với RPM database baseline; nó **không tự động kết luận server bị compromise**.

Configuration file có thể được administrator thay đổi hợp lệ.

---

# 5. Vì sao không nên dùng `rpm` trực tiếp để install package như default workflow?

Bạn có thể gặp:

```bash
rpm -ivh package.rpm
```

Nhưng `rpm` ở layer thấp không cung cấp dependency-resolution workflow giống DNF.

Nếu package A cần:

```text
library B
package C
capability D
```

thì DNF có thể resolve dependency từ repository.

Red Hat hiện hướng dẫn cài local RPM bằng:

```bash
dnf install /path/to/package.rpm
```

và DNF sẽ cố resolve dependencies từ configured repositories. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/installing-rhel-content?utm_source=chatgpt.com)

Vì vậy mental model:

```text
rpm
=
package database / package-level inspection

dnf
=
repository-aware package lifecycle management
```

---

# 6. DNF

## 6.1 What

`dnf` là high-level package management tool.

Nó quản lý:

```text
repository metadata
package search
dependency resolution
package installation
updates
removal
transactions
package querying
```

Ví dụ:

```bash
dnf search nginx
dnf info nginx
dnf install nginx
dnf remove nginx
dnf list --installed
dnf repolist
```

Red Hat RHEL 10 documentation xác nhận các workflow như `dnf search`, `dnf repoquery`, `dnf list`, `dnf repolist`, `dnf info`, `dnf install` và automatic dependency resolution. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/dnf-commands-list?utm_source=chatgpt.com)

---

# 7. Repository

## 7.1 Repository là gì?

Repository là source cung cấp package và package metadata.

Conceptually:

```text
RHEL server
   |
   v
repository metadata
   |
   +-- Package A
   +-- Package B
   +-- Package C
   +-- dependency information
```

RHEL hiện đại phân phối content qua các repository families như `BaseOS` và `AppStream`; exact repository names phụ thuộc architecture, release và subscription. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/distribution-of-content-in-rhel-10?utm_source=chatgpt.com)

Custom `.repo` configuration thường nằm dưới:

```text
/etc/yum.repos.d/
```

DNF main configuration:

```text
/etc/dnf/dnf.conf
```

Red Hat RHEL 10 documentation vẫn sử dụng các location này. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/managing-custom-software-repositories?utm_source=chatgpt.com)

---

# 8. Repository awareness

Một package installation có thể fail với message gần như:

```text
No match for argument
```

Nhưng root cause có thể là:

```text
wrong package name
repository disabled
repository metadata stale
repository unreachable
DNS failure
proxy failure
subscription/access problem
unsupported repository
package unavailable for release
architecture mismatch
```

Do đó không được kết luận:

> “Package không tồn tại.”

chỉ từ một failure message.

Read-only investigation có thể bắt đầu bằng:

```bash
dnf repolist
dnf info package_name
dnf search package_name
```

---

# 9. Subscription awareness

RHEL commercial environment thường gắn với Red Hat subscription/content-access model hoặc organization repository infrastructure.

Điểm Day 2 cần hiểu không phải học toàn bộ Red Hat subscription architecture, mà là:

```text
Package availability
       ↓
configured repository
       ↓
content entitlement/access
       ↓
RHEL release / architecture
       ↓
organization support policy
```

Trong enterprise có thể không được phép server truy cập Red Hat CDN trực tiếp.

Thay vào đó:

```text
RHEL host
   ↓
Satellite / internal mirror
   ↓
approved repositories
```

Do đó **không tự thêm random Internet repository** chỉ để làm package install thành công.

Đó có thể gây:

```text
unsupported packages
dependency conflicts
security exposure
supply-chain risk
future patching problems
vendor-support problems
```

Red Hat cũng cảnh báo repository không được xác minh có thể gây security, stability và maintainability risks. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_software_with_the_dnf_tool/assembly_managing-custom-software-repositories_managing-software-with-the-dnf-tool?utm_source=chatgpt.com)

---

# 10. Package operational workflow

Một package change đúng operationally không bắt đầu bằng:

```bash
sudo dnf update
```

Mà là:

```text
1. Identify host/environment
2. Confirm ticket/change scope
3. Record OS release
4. Record package baseline
5. Verify repository
6. Assess dependency
7. Assess service impact
8. Assess reboot requirement
9. Perform approved transaction
10. Verify package
11. Verify service/application
12. Check logs
13. Record evidence
14. Rollback/escalate if validation fails
```

Đặc biệt trong production, chạy:

```bash
dnf update
```

một cách không scoped có thể update nhiều package ngoài intended change.

---

# 11. Environment variables

## 11.1 What

Environment variables là key-value data được process sử dụng làm execution context.

Ví dụ:

```bash
PATH=/usr/local/bin:/usr/bin:/bin
HOME=/home/alice
LANG=en_US.UTF-8
```

Xem environment:

```bash
env
```

Xem một variable:

```bash
echo "$PATH"
```

---

# 12. Shell variable khác environment variable

Ví dụ:

```bash
APP_ENV=lab
```

Đây trước tiên là shell variable.

Sau:

```bash
export APP_ENV
```

nó được đặt vào environment mà child processes có thể inherit.

Mental model:

```text
Shell
 |
 | export VAR=value
 |
 +----> child process
          receives VAR
```

Nhưng child process thay đổi environment của chính nó không làm environment của parent shell tự thay đổi.

---

# 13. `PATH`

`PATH` xác định directories mà shell search command.

Ví dụ:

```bash
echo "$PATH"
```

Nếu:

```text
/usr/local/bin:/usr/bin:/bin
```

và bạn nhập:

```bash
mytool
```

shell search:

```text
/usr/local/bin/mytool
/usr/bin/mytool
/bin/mytool
```

---

# 14. Vì sao environment rất quan trọng trong Operations?

Một common failure:

```text
Interactive shell:
backup-script works

cron:
backup-script fails
```

Có thể vì cron không có cùng:

```text
PATH
HOME
working directory
locale
environment variables
credentials/context
```

Tương tự với `systemd`.

Một service không nhất thiết inherit:

```text
~/.bashrc
~/.bash_profile
```

của user interactive shell.

Đây là lý do production automation nên:

```text
use explicit paths
define required environment
not rely on interactive shell assumptions
```

Ví dụ tốt hơn:

```bash
/usr/bin/curl
/usr/bin/rsync
/usr/bin/date
```

khi execution context cần determinism.

---

# 15. Linux processes

## 15.1 Process là gì?

Program là executable/file.

Process là **running instance** của program.

Ví dụ:

```text
/usr/sbin/sshd
```

là executable.

Khi chạy, nó trở thành một process có:

```text
PID
PPID
UID/GID
memory mappings
open files
environment
CPU state
scheduling state
```

---

# 16. PID và PPID

`PID`:

```text
Process ID
```

`PPID`:

```text
Parent Process ID
```

Ví dụ:

```bash
ps -ef
```

hoặc:

```bash
ps -eo pid,ppid,user,state,cmd
```

Process hierarchy có thể được xem conceptually:

```text
systemd PID 1
 |
 +-- sshd
 |    |
 |    +-- ssh session
 |           |
 |           +-- bash
 |
 +-- crond
 |
 +-- application service
```

---

# 17. Process states

Bạn sẽ thường thấy states như:

```text
R
S
D
T
Z
```

Ý nghĩa khái quát:

| State | Ý nghĩa operational                                     |
| ----- | ------------------------------------------------------- |
| `R`   | Running hoặc runnable                                   |
| `S`   | Interruptible sleep, thường đang chờ event              |
| `D`   | Uninterruptible sleep, thường liên quan I/O/kernel wait |
| `T`   | Stopped/traced                                          |
| `Z`   | Zombie                                                  |

## Zombie

Zombie không phải process đang “ăn CPU vô hạn”.

Một zombie là child process đã exit nhưng parent chưa collect termination status.

Nếu thấy:

```text
Z
```

việc `kill -9` zombie thường không giải quyết vấn đề vì process đã kết thúc; cần xem parent process và lý do parent không reap child.

---

# 18. Signals

Signal là mechanism để kernel/process giao tiếp asynchronous events.

Các signal quan trọng:

```text
SIGTERM
SIGKILL
SIGHUP
SIGINT
```

`SIGTERM` yêu cầu process terminate một cách graceful nếu process xử lý signal đó.

```bash
kill PID
```

mặc định thường gửi `SIGTERM`.

`SIGKILL`:

```bash
kill -9 PID
```

kernel chấm dứt process ngay; process không thể catch hay ignore `SIGKILL`.

Vì vậy:

```text
SIGTERM
   ↓
wait/verify
   ↓
investigate
   ↓
SIGKILL only when justified
```

tốt hơn:

```text
problem
↓
kill -9
```

---

# 19. SIGHUP

`SIGHUP` historically là hangup signal.

Nhiều daemon dùng nó theo convention để reload configuration, nhưng:

> Không phải service nào nhận `SIGHUP` cũng reload.

Phải kiểm tra service documentation.

Với service do `systemd` quản lý, thường ưu tiên:

```bash
systemctl reload service
```

nếu service/unit hỗ trợ reload.

---

# 20. Process resource usage

Một Infrastructure Operations engineer cần xem ít nhất:

```text
CPU
memory
load
I/O symptoms
process count
```

Các command thường gặp:

```bash
top
ps
uptime
free
```

---

# 21. Load average không phải CPU percentage

Ví dụ:

```text
load average: 2.40, 1.80, 1.20
```

không có nghĩa:

```text
CPU = 240%
```

Linux load average liên quan đến workload đang runnable và một số task đang trong uninterruptible wait.

Phải interpret cùng:

```text
CPU count
CPU utilization
I/O wait
process state
```

Ví dụ:

```text
load = 8
```

trên server 2 CPU khác rất nhiều với:

```text
load = 8
```

trên server 64 CPU.

---

# 22. Memory: `free` không đơn giản bằng unused RAM

Linux cố dùng RAM hiệu quả cho:

```text
page cache
buffers
kernel structures
applications
```

Do đó thấy ít `free` memory không tự động là problem.

Thông thường metric `available` hữu ích hơn khi đánh giá memory pressure.

Cần xem thêm:

```text
swap
process RSS
OOM events
application behavior
```

---

# 23. `systemd`

`systemd` là init/service-management system trên RHEL hiện đại.

PID 1 thường là:

```text
systemd
```

Nó quản lý không chỉ service mà nhiều loại **unit**.

Red Hat hiện mô tả các unit như `service`, `target`, `device`, `mount`, `timer` và các unit types khác. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/working-with-systemd-unit-files?utm_source=chatgpt.com)

---

# 24. Systemd unit

Một unit đại diện cho resource/action mà `systemd` quản lý.

Ví dụ:

```text
sshd.service
multi-user.target
data.mount
backup.timer
```

Tên thường có dạng:

```text
name.type
```

---

# 25. Unit-file locations

Các location quan trọng:

```text
/usr/lib/systemd/system/
/run/systemd/system/
/etc/systemd/system/
```

Khái niệm chung:

```text
/usr/lib/systemd/system/
vendor/package supplied units

/run/systemd/system/
runtime/transient data

/etc/systemd/system/
administrator configuration/overrides
```

RHEL 10 documentation hiện vẫn dùng separation này. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/working-with-systemd-unit-files?utm_source=chatgpt.com)

Operationally:

> Không nên sửa vendor unit dưới `/usr/lib/systemd/system/` trực tiếp nếu có thể dùng override, vì package update có thể thay thế vendor file.

---

# 26. Unit file structure

Một `.service` thường có sections:

```ini
[Unit]
Description=Example Service
After=network.target

[Service]
ExecStart=/usr/local/bin/app
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Ba khái niệm:

```text
[Unit]
dependencies/order/description

[Service]
how process runs

[Install]
how enablement is connected
```

---

# 27. `systemctl`

Read-only/status:

```bash
systemctl status sshd
systemctl is-active sshd
systemctl is-enabled sshd
systemctl list-units --type=service
systemctl list-unit-files --type=service
```

State-changing:

```bash
systemctl start sshd
systemctl stop sshd
systemctl restart sshd
systemctl reload sshd
systemctl enable sshd
systemctl disable sshd
```

---

# 28. `active` và `enabled` khác nhau

Đây là câu hỏi phỏng vấn rất phổ biến.

```text
active
=
running/current runtime state
```

Trong khi:

```text
enabled
=
configured to be pulled in at appropriate boot target
```

Có thể có:

```text
active + disabled
```

vì administrator manually start service.

Cũng có thể:

```text
inactive + enabled
```

nếu service được cấu hình start at boot nhưng hiện đã bị stop/fail.

---

# 29. `disable` và `mask`

`disable` thường ngăn service tự start theo enablement relationship.

Nhưng vẫn có thể:

```bash
systemctl start service
```

manually.

`mask` mạnh hơn: ngăn unit được start thông thường cho tới khi unmask.

Red Hat cũng phân biệt rõ disabling và masking theo cách này. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/managing-system-services-with-systemctl?utm_source=chatgpt.com)

Operationally, đừng `mask` một service chỉ để “chắc chắn nó không chạy” nếu chưa hiểu dependency impact.

---

# 30. `systemctl daemon-reload`

Nếu sửa/create unit file:

```bash
systemctl daemon-reload
```

để systemd reload unit definitions.

Nó **không tự động restart application service**.

Conceptually:

```text
Edit unit file
      ↓
daemon-reload
      ↓
systemd knows new definition
      ↓
restart/reload service if required
      ↓
verification
```

---

# 31. Dependency và ordering

Đây là concept nâng cao nhưng rất cần hiểu.

Ví dụ:

```ini
After=network.target
```

nói về **ordering**.

Nó không nhất thiết có nghĩa:

> service này bắt buộc network.target phải thành công.

Systemd có nhiều semantics khác nhau như:

```text
After / Before
Wants
Requires
```

Không nên xem tất cả là cùng một loại dependency.

---

# 32. Service lifecycle

Operational lifecycle:

```text
installed
  ↓
configured
  ↓
enabled
  ↓
started
  ↓
healthy
  ↓
monitored
  ↓
stopped/reloaded/restarted
```

`running` cũng chưa đồng nghĩa với:

```text
application healthy
```

Ví dụ:

```bash
systemctl status nginx
```

cho `active (running)` nhưng upstream database dead.

User vẫn nhận:

```text
HTTP 500
```

Do đó service verification phải có nhiều layer.

---

# 33. Failed-service troubleshooting

Flow tốt:

```text
1. Read service status
2. Read journal
3. Determine exit code
4. Check configuration
5. Check required files
6. Check permissions/ownership
7. Check SELinux
8. Check dependencies
9. Check ports
10. Check storage/capacity
11. Correct smallest confirmed cause
12. Restart
13. Re-test
```

Không phải:

```text
restart
restart
reboot
```

---

# 34. Boot targets

Systemd sử dụng **targets** để group units và biểu diễn system state/synchronization points.

Common targets:

```text
multi-user.target
graphical.target
rescue.target
emergency.target
```

RHEL 10 documentation mô tả `multi-user.target`, `graphical.target`, `rescue.target` và `emergency.target` theo mô hình này. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/booting-into-a-target-system-state?utm_source=chatgpt.com)

Xem target mặc định:

```bash
systemctl get-default
```

Ví dụ:

```text
multi-user.target
```

---

# 35. `rescue.target` và `emergency.target`

Hai mode này phục vụ recovery, nhưng có mức environment khác nhau.

Bạn không nên chuyển target trên production host chỉ để thử.

Lệnh như:

```bash
systemctl isolate rescue.target
```

là **disruptive** vì systemd có thể stop units không cần bởi target mới.

Đây là thao tác cần:

```text
maintenance approval
console/out-of-band access
rollback/recovery plan
```

---

# 36. Boot-time service startup

Khi user báo:

> “Hôm qua service chạy bình thường, sáng nay server reboot xong application down.”

Phải hỏi:

```text
Was service enabled?
Did it fail during boot?
Was dependency available?
Was storage mounted?
Did network become ready?
Did config change?
Was credential available?
```

Kiểm tra:

```bash
systemctl is-enabled service
systemctl status service
journalctl -b
journalctl -u service -b
```

`-b` tập trung journal vào current boot.

---

# 37. SELinux

SELinux = **Security-Enhanced Linux**.

Nó cung cấp **Mandatory Access Control (MAC)** bổ sung cho Unix discretionary permissions.

Traditional Unix:

```text
owner/group/mode
```

SELinux thêm:

```text
security policy/context
```

Do đó access được phép khi **tất cả relevant control layers cho phép**.

---

# 38. DAC và MAC

Ví dụ:

```text
File mode:
-rw-r--r--
```

có thể nói application user được đọc.

Nhưng SELinux vẫn có thể deny nếu policy không cho process domain access target type.

Mental model:

```text
Unix permissions OK
        +
SELinux policy OK
        =
access may proceed
```

Không phải:

```text
chmod 777
=
everything fixed
```

---

# 39. SELinux modes

Ba state/mode bạn phải hiểu:

```text
Enforcing
Permissive
Disabled
```

### Enforcing

Policy được áp dụng và violation bị deny.

### Permissive

Policy vẫn hoạt động và denial được log, nhưng action không bị block theo denial đó. Nó chủ yếu dùng troubleshooting/policy development.

### Disabled

SELinux enforcement bị tắt; Red Hat hiện khuyến cáo mạnh không dùng disabled như default troubleshooting solution. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/getting-started-with-selinux?utm_source=chatgpt.com)

Kiểm tra:

```bash
getenforce
sestatus
```

---

# 40. SELinux context

Context thường có format:

```text
user:role:type:level
```

Ví dụ conceptual:

```text
system_u:object_r:httpd_sys_content_t:s0
```

Trong normal server troubleshooting, **type** thường là phần bạn quan tâm nhất.

Xem file context:

```bash
ls -Z /path
```

Xem process context:

```bash
ps -eZ
```

---

# 41. SELinux Type Enforcement

Ví dụ process:

```text
httpd_t
```

và file:

```text
httpd_sys_content_t
```

SELinux policy quyết định:

```text
Can httpd_t read this type?
Can it write?
Can it network connect?
```

Nếu file được copy/move sai cách từ location khác, context có thể không phù hợp.

---

# 42. `restorecon` và `chcon`

Một important distinction:

```text
chcon
```

thay context trực tiếp nhưng không nhất thiết tạo persistent policy mapping.

`restorecon` đưa object về expected label theo SELinux file-context configuration.

Enterprise workflow thích:

```text
define correct persistent context rule
        ↓
restorecon
```

hơn là random `chcon`.

---

# 43. SELinux denial

SELinux denial thường tạo `AVC` records.

Nếu Audit subsystem được cấu hình, common location:

```text
/var/log/audit/audit.log
```

và có thể query:

```bash
ausearch -m AVC
```

Red Hat RHEL 10 documentation mô tả SELinux AVC records trong Audit logs và `ausearch` để filter chúng. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_selinux/using_selinux?utm_source=chatgpt.com)

---

# 44. Sai lầm SELinux phổ biến

Anti-pattern:

```text
Permission denied
     ↓
chmod 777
     ↓
still fails
     ↓
setenforce 0
     ↓
works
     ↓
disable SELinux forever
```

Cách đúng:

```text
1. Unix owner/group/mode
2. Process identity
3. SELinux mode
4. File/process context
5. AVC evidence
6. Confirm intended application behavior
7. Correct label/boolean/policy configuration
8. Re-test under Enforcing
```

`Permissive` có thể là diagnostic step được phê duyệt, không phải permanent fix.

---

# 45. `firewalld`

`firewalld` là firewall-management framework thường dùng trên RHEL.

Nó abstract underlying packet-filtering implementation và tổ chức policy theo concepts như:

```text
zones
services
ports
rules
policies
```

Red Hat hiện mô tả `firewalld` rules có thể match source/destination IP, protocol, ports và interfaces; zones nhóm rule sets theo mức trust/use case. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld?utm_source=chatgpt.com)

---

# 46. Firewall zone

Zone đại diện cho trust/security profile.

Ví dụ names thường thấy:

```text
public
internal
trusted
drop
```

Nhưng đừng suy diễn chỉ từ tên.

Điều cần biết là:

```text
Which interface/source is attached?
Which services/ports/rules are allowed?
```

---

# 47. Firewall service và port

Ví dụ cho phép predefined service:

```bash
firewall-cmd --add-service=http
```

hoặc explicit port:

```bash
firewall-cmd --add-port=8080/tcp
```

Nhưng thay đổi firewall là configuration-changing action.

Trước đó read-only:

```bash
firewall-cmd --state
firewall-cmd --get-active-zones
firewall-cmd --list-all
```

---

# 48. Runtime vs permanent `firewalld`

Một concept rất quan trọng:

```text
runtime configuration
```

và:

```text
permanent configuration
```

có thể khác.

Ví dụ một rule runtime có thể biến mất sau reload/reboot nếu không được persisted.

Ngược lại, permanent config có thể chưa được applied vào current runtime nếu chưa reload hoặc chưa được triển khai phù hợp.

Do đó verification cần kiểm tra cả hai state nếu change yêu cầu persistence.

---

# 49. Firewall troubleshooting

Nếu client không connect được port 443:

```text
Client
  |
  | TCP/443
  v
Network
  |
  v
Host firewall
  |
  v
Listening socket
  |
  v
Application
```

Phải phân biệt:

```text
service not running
service running but no listener
listener on wrong IP
host firewall
upstream firewall
routing
TLS failure
application failure
```

Không nên add firewall rule trước khi xác nhận application có listener.

---

# 50. Networking foundation

Day 2 yêu cầu:

```text
IP addressing
routing
DNS
TCP/UDP
ports
ping
traceroute/tracepath
curl
ss
ip
```

Mục tiêu không phải trở thành Network Engineer ở Day 2, mà phải troubleshoot server network path có phương pháp.

---

# 51. IP address

IPv4 example:

```text
192.0.2.10/24
```

`/24` nghĩa là prefix length.

Conceptually:

```text
IP address = host address
prefix     = network boundary
```

Server cần biết destination nào local subnet và destination nào cần router.

---

# 52. Interface

Network interface có:

```text
interface name
MAC address
IP addresses
state
MTU
```

Read-only:

```bash
ip link
ip addr
```

Example:

```bash
ip addr show
```

Bạn cần phân biệt:

```text
administratively UP
```

và actual link state/operability.

---

# 53. Routing table

Linux kernel dùng routing table để chọn đường đi.

Read:

```bash
ip route
```

Example conceptual:

```text
default via 192.0.2.1 dev ens192
192.0.2.0/24 dev ens192 src 192.0.2.10
```

Nghĩa:

```text
192.0.2.0/24
→ directly connected

everything else
→ default gateway 192.0.2.1
```

---

# 54. Route selection

Kernel không đơn giản chọn default route trước.

Nó thường ưu tiên:

```text
most specific matching route
```

Ví dụ có:

```text
10.0.0.0/8
10.1.0.0/16
10.1.2.0/24
```

destination:

```text
10.1.2.50
```

sẽ match `/24` cụ thể nhất.

Các yếu tố khác như metric, policy routing và multiple tables có thể tham gia ở môi trường phức tạp.

---

# 55. `ip route get`

Command rất hữu ích:

```bash
ip route get 203.0.113.10
```

Nó giúp trả lời:

```text
Which route?
Which interface?
Which source IP?
```

Red Hat cũng dùng `ip route` và `ip route get` trong current RHEL networking documentation để verify routing decisions. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/configuring_and_managing_networking/configuring-a-network-bond-by-using-nmstatectl?utm_source=chatgpt.com)

---

# 56. DNS resolution

DNS chuyển:

```text
hostname
→
IP address
```

Ví dụ:

```text
app.example.com
→
192.0.2.50
```

Một request:

```bash
curl https://app.example.com
```

có nhiều stage:

```text
DNS
 ↓
routing
 ↓
TCP
 ↓
TLS
 ↓
HTTP
 ↓
application
```

Nếu DNS fail, chưa có nghĩa network path đến server IP fail.

---

# 57. DNS troubleshooting mindset

Test hostname:

```bash
getent hosts app.example.com
```

và so sánh direct IP test nếu phù hợp.

Các source gây DNS issue:

```text
wrong DNS server
network path to DNS server
search domain issue
bad record
stale cache
resolver configuration
NetworkManager profile
split DNS
```

Trên RHEL hiện đại, NetworkManager thường quản lý connection profile và DNS settings; không nên assume manual edit `/etc/resolv.conf` là permanent solution. Current RHEL documentation đặt NetworkManager ở trung tâm network configuration. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/configuring_and_managing_networking/configuring-a-network-bond-by-using-nmstatectl?utm_source=chatgpt.com)

---

# 58. TCP

TCP là connection-oriented transport.

Simplified connection establishment:

```text
Client              Server
  | ---- SYN ------> |
  | <--- SYN/ACK --- |
  | ---- ACK ------> |
```

Sau đó application data.

Nếu TCP handshake không hoàn thành, hãy nghĩ:

```text
routing
firewall
no listener
network loss
ACL/security control
```

---

# 59. UDP

UDP không có TCP-style handshake.

Vì vậy:

```text
UDP test semantics ≠ TCP test semantics
```

Không thể suy luận UDP service health chỉ từ absence/presence của TCP-like connection state.

---

# 60. Ports

Port giúp identify transport endpoint.

Common examples:

```text
22/tcp   SSH
53/tcp/udp DNS
80/tcp   HTTP
443/tcp  HTTPS
```

Nhưng application có thể được configured dùng port khác.

Một port không tồn tại đơn độc; relevant tuple gần với:

```text
protocol
local address
local port
remote address
remote port
state
process
```

---

# 61. `ss`

`ss` dùng inspect sockets.

Ví dụ:

```bash
ss -lnt
```

Có thể hiểu options:

```text
-l  listening
-n  numeric
-t  TCP
```

Common troubleshooting:

```bash
ss -lntp
```

với đủ quyền sẽ giúp map listener ↔ process.

Ví dụ:

```text
LISTEN 0 128 0.0.0.0:22 ...
```

khác:

```text
127.0.0.1:8080
```

Nếu service chỉ listen:

```text
127.0.0.1
```

remote client không thể truy cập qua server network IP dù firewall mở port.

---

# 62. `ping`

`ping` chủ yếu test ICMP Echo reachability.

Một successful ping chứng minh một số layer hoạt động nhưng **không chứng minh application port hoạt động**.

Ngược lại ping fail không chứng minh server down vì ICMP có thể bị block.

Therefore:

```text
ping success
≠
HTTPS success

ping failure
≠
host definitely dead
```

---

# 63. `traceroute` và `tracepath`

Hai tool giúp quan sát network path/hops dựa trên TTL/hop-limit behavior.

Dùng để phát hiện:

```text
path changes
where responses stop
possible routing issue
MTU/path observations
```

Nhưng:

> Một hop không trả lời không nhất thiết là hop bị lỗi.

Network devices có thể không trả control-plane response nhưng vẫn forward data.

Vì vậy traceroute output phải correlate với end-to-end test.

---

# 64. `curl`

`curl` rất mạnh vì test application layer.

Ví dụ:

```bash
curl -v https://app.example.com/
```

có thể giúp phân biệt:

```text
DNS resolved?
TCP connected?
TLS handshake?
HTTP status?
response?
```

Bạn phải học đọc từng phase, không chỉ cuối cùng thấy:

```text
curl: (7) Failed to connect
```

---

# 65. Network troubleshooting flow

Một flow an toàn:

```text
1. Interface state
2. IP address
3. Local route
4. Route to destination
5. Gateway/path
6. DNS
7. Remote IP reachability where relevant
8. TCP/UDP port
9. TLS/application protocol
10. Application logs
```

Ví dụ:

```bash
ip addr
ip route
ip route get 203.0.113.10
getent hosts app.example.com
ping ...
tracepath ...
ss ...
curl ...
```

---

# 66. Storage architecture

Một simplified storage stack:

```text
Disk / Block device
        ↓
Partition
        ↓
Filesystem
        ↓
Mount
        ↓
Directory tree
        ↓
Application data
```

Có thể thêm LVM ở giữa trong enterprise:

```text
disk
↓
partition/PV
↓
VG
↓
LV
↓
filesystem
↓
mount
```

**LVM không được syllabus Day 2 gọi tên trực tiếp**, vì vậy đây là kiến thức bổ sung để bạn hiểu architecture; phần Lab Day 2 không bắt buộc phải đào sâu LVM trừ khi environment dùng nó.

---

# 67. Disk và block device

Inspect:

```bash
lsblk
```

Ví dụ:

```text
sda
├─sda1
└─sda2
```

Ở đây:

```text
sda
=
disk

sda1/sda2
=
partitions
```

Virtual environments có thể dùng names khác:

```text
vda
nvme0n1
```

Không assume luôn là `/dev/sda`.

---

# 68. Partition

Partition là logical region trên block device.

Partition table thường theo model như:

```text
GPT
MBR
```

Tạo/sửa partition là **configuration-changing và potentially destructive**.

Trong production phải có:

```text
device identification
data backup
dependency check
change approval
rollback/recovery plan
```

Day 2 theory cần hiểu concept, không khuyến khích thử partitioning trên disk chứa OS/data.

---

# 69. Filesystem

Filesystem tổ chức files/directories/metadata trên storage.

RHEL commonly uses:

```text
XFS
ext4
```

Red Hat RHEL 10 hiện dùng XFS là default local filesystem; XFS và ext4 có khác biệt operational, ví dụ XFS không có utility để shrink filesystem trong place theo workflow thông thường. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/managing_file_systems/Red_Hat_Enterprise_Linux-10-Managing_file_systems-en-US.pdf?utm_source=chatgpt.com)

Đây là ví dụ điển hình tại sao phải biết filesystem type trước khi làm capacity change.

---

# 70. Mount

Linux không dùng drive letters như Windows.

Filesystem được attach vào unified directory tree.

Ví dụ:

```text
/dev/sdb1
   ↓
/data
```

Sau:

```bash
mount /dev/sdb1 /data
```

files trong filesystem xuất hiện dưới `/data`.

Red Hat mô tả Linux mounting chính xác theo model filesystem attached vào directory tree tại mount point. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/mounting-file-systems?utm_source=chatgpt.com)

---

# 71. Mount-point trap

Nếu `/data` đã có files và sau đó mount filesystem lên `/data`, những files cũ dưới directory đó bị **hidden by mount** cho đến khi unmount.

Không có nghĩa chúng đã bị delete.

Đây là một source của confusion:

> “Files biến mất sau khi mount.”

Có thể chỉ bị hidden.

---

# 72. `findmnt`

Một công cụ rất hữu ích:

```bash
findmnt
```

hoặc:

```bash
findmnt /data
```

Red Hat current docs cũng khuyến nghị `findmnt` để xem current mounts. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/mounting-file-systems?utm_source=chatgpt.com)

---

# 73. UUID

Device names như:

```text
/dev/sdb1
```

có thể không phải stable identifier lý tưởng trong mọi environment.

Persistent mounts thường dùng:

```text
UUID=...
```

Xem:

```bash
blkid
lsblk -f
```

---

# 74. `/etc/fstab`

`/etc/fstab` định nghĩa persistent filesystem mount configuration.

Format phổ biến gồm sáu fields:

```text
device  mount-point  fs-type  options  dump  fsck-order
```

Ví dụ:

```text
UUID=abcd... /data xfs defaults 0 0
```

Red Hat documentation cũng mô tả sáu fields và việc `systemd-fstab-generator` chuyển entries thành systemd mount units. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/persistently-mounting-file-systems?utm_source=chatgpt.com)

---

# 75. Vì sao `/etc/fstab` là high-risk file?

Bad entry có thể gây:

```text
mount failure
application failure
boot delay
boot/recovery issue
```

Vì vậy enterprise workflow:

```text
backup current fstab
↓
identify UUID
↓
confirm mount point
↓
edit
↓
reload relevant configuration
↓
test
↓
verify
↓
only then consider reboot validation
```

Current RHEL 10 documentation hướng dẫn `systemctl daemon-reload` sau khi thêm `fstab` entry và test mount trước khi coi cấu hình hợp lệ. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/persistently-mounting-file-systems?utm_source=chatgpt.com)

---

# 76. Disk capacity

`df` xem filesystem usage:

```bash
df -h
```

Bạn cần hiểu các columns:

```text
Size
Used
Avail
Use%
Mounted on
```

Nhưng:

```text
df
```

không cho bạn biết directory nào chiếm space.

Để investigate:

```bash
du
```

Ví dụ:

```bash
du -sh /var/*
```

phải cẩn thận trên filesystem lớn.

---

# 77. `df` và `du` có thể khác nhau

Ví dụ:

```text
df says filesystem almost full
du cannot find enough files
```

Một possible cause:

```text
deleted file still open by process
```

Process vẫn giữ file descriptor nên blocks chưa được release.

Điều này có thể investigate bằng tools như `lsof` nếu available.

Không nên reboot chỉ để reclaim space nếu chưa hiểu process impact.

---

# 78. Inodes

Filesystem không chỉ cần data blocks.

Mỗi file/directory cần inode/metadata structure.

Kiểm tra:

```bash
df -i
```

Có thể gặp:

```text
Disk blocks: 40% used
Inodes:      100% used
```

và application nhận:

```text
No space left on device
```

Dù còn GB free.

Common cause:

```text
millions of tiny files
```

Ví dụ:

```text
bad cache
mail spool
temporary-file leak
logging design issue
```

---

# 79. Storage troubleshooting flow

Khi application báo:

```text
No space left on device
```

đừng chỉ chạy:

```bash
df -h
```

Workflow:

```text
Filesystem mounted?
↓
df -h
↓
df -i
↓
largest paths / file counts
↓
deleted-open files
↓
application logs
↓
quota/LVM/storage layer where relevant
```

---

# 80. Logging architecture

Một RHEL server có thể có nhiều logging layers:

```text
kernel messages
systemd journal
rsyslog/syslog files
application-specific logs
authentication logs
Audit subsystem
```

Đây không phải các nguồn hoàn toàn interchangeable.

---

# 81. `systemd-journald`

`systemd-journald` thu thập journal records từ nhiều source như:

```text
kernel
boot
service stdout/stderr
syslog input
systemd metadata
```

Current Red Hat documentation mô tả `systemd-journald` và `rsyslog` cùng tham gia RHEL logging stack; `rsyslog` có thể xử lý và persist messages vào `/var/log`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/risk_reduction_and_recovery_operations/index?utm_source=chatgpt.com)

---

# 82. `journalctl`

Core command:

```bash
journalctl
```

Current boot:

```bash
journalctl -b
```

Specific unit:

```bash
journalctl -u sshd.service
```

Current boot + unit:

```bash
journalctl -u sshd.service -b
```

Time window:

```bash
journalctl \
  --since "2026-10-20 10:00:00" \
  --until "2026-10-20 10:15:00"
```

Red Hat current recovery/troubleshooting documentation explicitly uses `journalctl -b`, time range, priority và unit filtering. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/risk_reduction_and_recovery_operations/index?utm_source=chatgpt.com)

---

# 83. Priority

Journal/syslog events có severity/priority.

Classic syslog levels từ nghiêm trọng đến thấp hơn gồm:

```text
emerg
alert
crit
err
warning
notice
info
debug
```

Ví dụ:

```bash
journalctl -p err
```

lọc error và higher priorities theo semantics của tool.

Nhưng một operational incident không nhất thiết tạo `err`.

Application có thể log root cause ở:

```text
warning
info
```

nên priority filter chỉ là một lens.

---

# 84. Journal persistence

Không được assume journal luôn giữ log qua reboot.

Persistence phụ thuộc configuration.

Red Hat current documentation lưu ý journal có thể sử dụng volatile storage dưới `/run/log/journal`, và persistence cần được cấu hình phù hợp. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/automating_system_administration_by_using_rhel_system_roles/configuring-the-systemd-journal-by-using-rhel-system-roles?utm_source=chatgpt.com)

Do đó khi làm incident:

```text
No event found
```

không được ngay lập tức kết luận:

```text
Event never happened
```

Phải hỏi:

```text
Was journal persistent?
What is retention?
Was host rebooted?
Was collection healthy?
```

---

# 85. Rsyslog / syslog

`rsyslog` xử lý syslog messages dựa trên:

```text
facility
priority
filters
actions
```

Main config thường:

```text
/etc/rsyslog.conf
```

và configuration snippets thường dưới:

```text
/etc/rsyslog.d/
```

Log files thường nằm:

```text
/var/log/
```

Current RHEL docs nêu các common files như:

```text
/var/log/messages
/var/log/secure
/var/log/cron
/var/log/boot.log
```

tùy actual rsyslog/service configuration. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/risk_reduction_and_recovery_operations/index?utm_source=chatgpt.com)

---

# 86. Authentication logs

Authentication/sudo events có thể xuất hiện trong:

```text
journal
/var/log/secure
Audit logs
```

tùy configuration.

Điều quan trọng:

> Không được assume `/var/log/secure` chắc chắn chứa mọi privileged action.

Bạn phải xác định actual logging configuration.

---

# 87. Linux Audit / `auditd`

Audit subsystem được thiết kế cho security/audit records chi tiết hơn generic application logging.

Common log location khi configured:

```text
/var/log/audit/audit.log
```

Tools:

```bash
ausearch
aureport
```

Red Hat documentation lưu ý Audit phải được configured phù hợp và `ausearch` có thể filter audit trail theo conditions. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/risk_reduction_and_recovery_operations/auditing-the-system?utm_source=chatgpt.com)

Syllabus cố tình nói:

> auditd records **where auditing is configured**.

Nghĩa là nếu Audit không được enabled/configured cho event đó, không được bịa ra rằng evidence tồn tại. Infrastructure_Operations_14_Da…

---

# 88. Log rotation

Không thể để:

```text
/var/log/messages
```

tăng vô hạn.

`logrotate` thường quản lý rotation theo:

```text
time
size
retention count
compression
postrotate actions
```

Configuration thường:

```text
/etc/logrotate.conf
/etc/logrotate.d/
```

Một sequence có thể trông như:

```text
app.log
app.log.1
app.log.2.gz
app.log.3.gz
```

---

# 89. Rotation và incident investigation

Incident occurred:

```text
03:00
```

Bạn investigate lúc:

```text
15:00
```

Trong thời gian đó log rotated.

Nếu chỉ search:

```text
app.log
```

bạn có thể miss event.

Phải kiểm tra:

```text
current log
rotated log
compressed rotated logs
journal
central logging
```

nếu relevant.

---

# 90. Retention

Retention trả lời:

```text
How long can evidence reasonably remain available?
```

Retention không chỉ là một number.

Có thể phụ thuộc:

```text
disk capacity
rotation policy
journal limits
central forwarding
application settings
legal/security policy
```

Audit-quality report phải ghi limitation:

```text
Logs available from X to Y.
Earlier interval unavailable due to retention.
```

thay vì:

```text
No previous events occurred.
```

---

# 91. `grep`

`grep` dùng search matching lines.

Examples:

```bash
grep "error" app.log
```

Case insensitive:

```bash
grep -i "error" app.log
```

Line numbers:

```bash
grep -n "error" app.log
```

Extended regex:

```bash
grep -E "error|failed|timeout" app.log
```

---

# 92. Regular expressions

Regex mô tả patterns.

Examples:

```text
^
start of line

$
end of line

.
one character

*
zero or more of previous expression

+
one or more in extended regex

[0-9]
digit

[^ ]
not a space
```

Ví dụ:

```bash
grep -E '^ERROR'
```

match lines starting `ERROR`.

---

# 93. Quoting Regex

Shell cũng interpret metacharacters.

Do đó thường dùng:

```bash
grep -E 'pattern'
```

thay vì để shell expand pattern ngoài ý muốn.

Ví dụ:

```bash
grep '*.log' file
```

và unquoted `*` có thể có semantics hoàn toàn khác ở shell.

Quoting là một phần của correct text processing, không phải cosmetic style.

---

# 94. `awk`

`awk` rất mạnh cho field-oriented text.

Ví dụ:

```bash
awk '{print $1, $3}' file
```

`$1`, `$3` ở đây là fields của awk, không phải Bash positional arguments.

Custom delimiter:

```bash
awk -F: '{print $1}' /etc/passwd
```

Workflow common:

```text
input
↓
split fields
↓
condition
↓
print selected fields
```

---

# 95. `sed`

`sed` là stream editor.

Simple substitution:

```bash
sed 's/old/new/' file
```

Global per line:

```bash
sed 's/old/new/g' file
```

Một safety principle:

> Đừng bắt đầu bằng in-place edit trên production config khi chưa preview output.

Safer:

```bash
sed '...' file
```

review first.

Sau đó mới cân nhắc update có backup/change control.

---

# 96. Text-processing pipeline

Ví dụ:

```bash
journalctl -u sshd \
  | grep 'Failed' \
  | awk '{print $1,$2,$3,$NF}'
```

Pipeline rất mạnh nhưng có risks:

```text
information loss
incorrect field assumptions
locale differences
format changes
regex overmatch
regex undermatch
```

Audit evidence nên giữ:

```text
raw/native export
+
derived filtered output
```

Không chỉ giữ output cuối pipeline.

---

# 97. Bash variables

Assignment:

```bash
HOST="server01"
```

Reference:

```bash
echo "$HOST"
```

Strong habit:

```bash
"$variable"
```

thay vì:

```bash
$variable
```

khi variable có thể chứa:

```text
spaces
wildcards
empty values
```

---

# 98. Command substitution

Example:

```bash
NOW=$(date)
```

Command output trở thành value.

Trong operational scripts:

```bash
HOST=$(hostname)
```

hoặc safer specific hostname command depending runbook.

---

# 99. Exit codes

Unix convention:

```text
0
=
success

non-zero
=
some form of failure
```

Nhưng exact non-zero meaning phụ thuộc command.

Check last command:

```bash
echo $?
```

Ví dụ:

```bash
grep pattern file
echo $?
```

`grep` có distinctions giữa:

```text
match found
no match
actual error
```

nên **non-zero không phải lúc nào cũng có cùng semantic meaning**.

Script cần hiểu contract của command nó gọi.

---

# 100. Why exit codes matter

Bad script:

```bash
backup-command
echo "Backup successful"
exit 0
```

Nếu `backup-command` fail, script vẫn báo success.

Đây là operationally dangerous.

Correct conceptual logic:

```text
run command
↓
capture status
↓
status == expected success?
     /     \
   yes      no
   |         |
verify     log error
   |         |
success    non-zero exit
```

---

# 101. Bash conditionals

Example:

```bash
if systemctl is-active --quiet sshd; then
    echo "sshd is active"
else
    echo "sshd is not active"
fi
```

Đây là control flow.

Các constructs Day 2 cần biết:

```text
if / elif / else
for
while
case
```

Functions là kiến thức bổ sung hữu ích cho reusable scripts.

---

# 102. `&&` và `||`

```bash
command1 && command2
```

nghĩa:

```text
run command2 only if command1 succeeds
```

```bash
command1 || command2
```

nghĩa:

```text
run command2 if command1 fails
```

Nhưng đừng viết chain dài khó audit nếu behavior quan trọng.

Clear `if` blocks thường dễ maintain hơn.

---

# 103. Pipeline exit status

Một subtle issue:

```bash
command1 | command2
```

mặc định Bash pipeline status thường liên quan status của command cuối.

Nếu:

```text
command1 failed
command2 succeeded
```

script có thể bỏ qua failure nếu không thiết kế phù hợp.

`pipefail` có thể giúp:

```bash
set -o pipefail
```

nhưng phải hiểu semantics trước khi bật.

---

# 104. `set -e`

Bạn sẽ thường thấy:

```bash
set -e
```

và lời khuyên:

> “Bật nó là script tự xử lý lỗi.”

Đây là oversimplification.

`set -e` có context-sensitive behavior liên quan:

```text
conditions
pipelines
subshells
command lists
```

Không được coi nó thay cho explicit error handling.

Day 2 nên ưu tiên:

```text
clear validation
capture exit code
explicit error message
explicit exit
```

---

# 105. Error handling

Một health-check/backup script nên phân biệt:

```text
expected unhealthy condition
```

và:

```text
script itself failed to perform check
```

Ví dụ:

```text
service inactive
```

khác với:

```text
systemctl command unavailable/permission denied
```

Cả hai không nên bị report giống nhau.

---

# 106. Sanitized logging

Script operational nên log:

```text
timestamp
hostname
check/action
result
exit status
error
```

nhưng không log:

```text
password
token
private key
secret
credential
```

Evidence không được làm tăng security exposure.

---

# 107. Cron

`cron` là time-based scheduler.

Daemon thường:

```text
crond
```

User crontab:

```bash
crontab -l
```

Editing:

```bash
crontab -e
```

---

# 108. Crontab syntax

Typical user crontab:

```text
minute hour day-of-month month day-of-week command
```

Example:

```cron
0 2 * * * /usr/local/bin/backup.sh
```

Nghĩa:

```text
02:00 mỗi ngày
```

---

# 109. System crontab khác user crontab

Files như:

```text
/etc/crontab
/etc/cron.d/*
```

có thể chứa thêm **user field**.

Conceptually:

```text
user crontab:
schedule + command

system cron:
schedule + run-as-user + command
```

Đây là common source of syntax errors.

---

# 110. Cron environment

Cron thường có environment tối giản hơn interactive shell.

Vì thế script này:

```bash
backup_tool ...
```

có thể chạy manually nhưng fail under cron.

Better:

```bash
/usr/local/bin/backup_tool ...
```

hoặc define required environment explicitly.

Common cron failure categories:

```text
PATH
permissions
working directory
relative paths
missing HOME
missing credentials/context
SELinux
output not captured
service not running
```

---

# 111. Cron verification

Không đủ:

```text
crontab -l
```

Bạn phải verify:

```text
entry exists
crond healthy
scheduled time occurred
script executed
output/log produced
expected data changed
exit/failure observable
```

Syllabus nhấn mạnh operational verification, không chỉ configuration presence.

---

# 112. File compression

Compression giảm storage size.

Common tools/formats:

```text
gzip
bzip2
xz
```

Conceptual distinction:

```text
archiving
≠
compression
```

`tar` chủ yếu tạo archive.

Sau đó có thể kết hợp compression:

```text
tar + gzip
```

---

# 113. `tar`

Create archive:

```bash
tar -cf archive.tar directory/
```

List:

```bash
tar -tf archive.tar
```

Extract:

```bash
tar -xf archive.tar
```

Compressed archive ví dụ gzip:

```bash
tar -czf archive.tar.gz directory/
```

Extract:

```bash
tar -xzf archive.tar.gz
```

---

# 114. Archive success không đồng nghĩa backup success

Một `tar` file tồn tại chưa đủ.

Phải xác minh:

```text
archive readable?
expected paths present?
file counts reasonable?
restore works?
restored content correct?
required metadata preserved?
```

---

# 115. `rsync`

`rsync` copy/synchronize data efficiently.

Concept:

```text
source
 ↓
compare
 ↓
transfer required differences
 ↓
destination
```

Basic:

```bash
rsync -a source/ destination/
```

`-a` = archive mode, preserve một tập metadata phổ biến.

Nhưng enterprise backup requirements có thể đòi thêm handling cho:

```text
ACLs
extended attributes
SELinux contexts
hard links
sparse files
```

nên không được assume một generic command đáp ứng mọi backup requirement.

---

# 116. `rsync` trailing slash

Đây là pitfall nổi tiếng.

```bash
rsync -a /source /backup/
```

và:

```bash
rsync -a /source/ /backup/
```

không nhất thiết tạo cùng directory structure.

Bạn phải test trên lab trước khi production use.

---

# 117. `rsync --delete`

Một option nguy hiểm:

```bash
--delete
```

Nó có thể remove destination files không còn source.

Đừng dùng theo kiểu:

```bash
rsync -a --delete ...
```

trên production backup target mà chưa có:

```text
scope confirmation
dry run
versioning/restore consideration
approval
```

Một synchronization mirror có thể propagate accidental deletion.

---

# 118. Sync không phải backup

Đây là principle rất quan trọng.

Nếu:

```text
source file deleted accidentally
```

và sync job mirror deletion sang destination:

```text
source deleted
+
backup deleted
```

thì mirror không cung cấp historical recovery.

Một proper backup strategy thường quan tâm:

```text
versions
retention
immutability
separation
recovery points
restore testing
```

Day 2 chỉ học basic `tar/rsync`, nhưng không được nhầm chúng với full enterprise backup platform.

---

# 119. Basic restore

Correct mental model:

```text
Backup
  ↓
Select restore point
  ↓
Restore into isolated/test location
  ↓
Verify
  ↓
Only then consider production replacement
```

Không nên restore trực tiếp đè lên production data để “test”.

---

# 120. Restore verification

Verify:

```text
file exists
file count
size
content
checksum if available
permissions
ownership
required metadata
application readability
```

Example content hash:

```bash
sha256sum file
```

Compare:

```text
original expected hash
vs
restored hash
```

Matching hash chứng minh hai byte streams match cho item đó, **không tự động chứng minh backup set hoàn chỉnh hoặc source originally trustworthy**.

---

# 121. Health checks

Health check là structured assessment của server.

Không phải:

```bash
uptime
```

rồi kết luận healthy.

Một server health model:

```text
System
├── identity
├── uptime/boot
├── CPU/load
├── memory/swap
├── filesystem capacity
├── inode capacity
├── mounts
├── service status
├── process state
├── network interfaces
├── routes
├── DNS
├── ports
├── security controls
├── recent critical logs
├── scheduled jobs
└── backup/recovery status
```

---

# 122. Availability vs health

Service:

```text
active (running)
```

nhưng client requests fail.

Do đó:

```text
service state
≠
service health
```

A stronger check:

```text
systemd process state
+
port listener
+
local application request
+
dependency state
+
external request if authorized
```

---

# 123. Baseline

Health check chỉ hữu ích hơn khi có baseline.

Ví dụ:

```text
CPU load 4
```

không tự nó nói abnormal.

Bạn cần context:

```text
normal baseline
CPU count
workload schedule
historical trend
```

Tương tự:

```text
Disk = 75%
```

có thể healthy hoặc critical tùy growth rate và threshold.

---

# 124. Operational troubleshooting

Đây là skill tổng hợp quan trọng nhất của Day 2.

Một Infrastructure Operations engineer cần tránh:

```text
guess → change → guess → reboot
```

Thay vào đó:

```text
Observe
↓
Define symptom
↓
Establish scope
↓
Collect low-risk evidence
↓
Build hypotheses
↓
Test hypotheses
↓
Identify root cause or probable cause
↓
Apply minimum approved correction
↓
Verify
↓
Monitor
↓
Document
```

---

# 125. Symptom khác root cause

Example:

```text
Symptom:
HTTP service unavailable

Immediate observation:
httpd.service failed

Root cause:
filesystem containing certificates failed to mount
```

Nếu chỉ:

```bash
systemctl restart httpd
```

thì bạn xử lý symptom, không xử lý root cause.

---

# 126. Troubleshooting by layers

Một server application failure có thể kiểm tra:

```text
Layer 1  Host alive?
Layer 2  OS healthy?
Layer 3  Filesystem mounted?
Layer 4  Disk/inodes available?
Layer 5  Package present?
Layer 6  Configuration valid?
Layer 7  Process running?
Layer 8  systemd state?
Layer 9  Permissions?
Layer 10 SELinux?
Layer 11 Port listening?
Layer 12 firewalld?
Layer 13 routing/DNS?
Layer 14 dependency?
Layer 15 application response?
```

Bạn không nhất thiết luôn kiểm tra đúng thứ tự này. Nhưng phải có logic.

---

# 127. Least-risk-first

Read-only actions trước:

```bash
systemctl status
journalctl
ps
ss
ip
df
findmnt
```

rồi mới configuration-changing actions:

```text
restart
modify config
change firewall
change SELinux configuration
mount/unmount
package update
reboot
```

Nguyên tắc:

```text
Evidence before intervention
```

nếu incident cho phép.

---

# 128. Restart không phải diagnosis

Restart có thể:

```text
temporarily recover service
```

nhưng đồng thời:

```text
destroy transient state
alter logs
clear symptoms
change timestamps
hide race conditions
impact users
```

Do đó trước restart production service, thường cần:

```text
status
logs
process/socket evidence
ticket/update
approval if required
```

---

# 129. Reboot là high-impact troubleshooting action

“Have you tried rebooting?” không phải Infrastructure Operations methodology.

Reboot có thể:

```text
temporarily clear resource leak
recover stale state
```

nhưng cũng có thể:

```text
turn recoverable server into non-booting server
trigger bad fstab
expose failed disk
lose volatile logs
impact workloads
```

Phải biết:

```text
Why reboot?
What do we expect it to fix?
What evidence do we need first?
What if it does not boot?
```

---

# 130. Escalation boundary

L1/L2 không nên tiếp tục change vô hạn.

Escalate khi:

```text
root cause outside authorized scope
privileged access unavailable
production change approval required
data integrity risk
filesystem corruption suspected
hardware/storage issue suspected
security incident suspected
repeated service crash with unknown cause
vendor-specific defect
recovery requires destructive operation
business impact exceeds authority
```

Escalation tốt phải cung cấp:

```text
symptom
scope
timeline
checks completed
commands/results
changes made
current state
suspected cause
what is needed next
```

Không chỉ:

```text
Server broken, please check.
```

---

# 131. RHEL maintenance — additive syllabus

Đây là phần bắt buộc bổ sung của Day 2. Syllabus yêu cầu review:

```text
release/subscription prerequisites
patch prerequisites
dependencies
reboot assessment
pre-maintenance checks
post-maintenance checks
recovery limitations
```

Infrastructure_Operations_14_Da…

Maintenance khác daily troubleshooting vì nó thường là **planned change**.

---

# 132. Maintenance lifecycle

Một enterprise maintenance workflow nên được hình dung:

```text
                  Approved ticket
                       |
                       v
               Scope confirmation
                       |
                       v
              Dependency analysis
                       |
                       v
                 Risk analysis
                       |
                       v
              Recovery/backout plan
                       |
                       v
                  Pre-check
                       |
                       v
                Implementation
                       |
                       v
                  Post-check
                       |
             +---------+---------+
             |                   |
             v                   v
          Success              Failure
             |                   |
             v                   v
      document/close     rollback/recover/
                           escalate
```

---

# 133. Release check

Trước patch/package operation cần xác định OS.

Examples:

```bash
cat /etc/redhat-release
cat /etc/os-release
```

Có thể thêm:

```bash
uname -r
```

cho running kernel.

Nhưng đừng nhầm:

```text
OS release
```

với:

```text
kernel version
```

Chúng liên quan nhưng không phải cùng một field.

---

# 134. Package baseline

Trước change cần biết:

```text
current package version
repository source if relevant
service status
enabled state
application status
```

Ví dụ evidence:

```bash
rpm -q httpd
systemctl status httpd
systemctl is-enabled httpd
```

Sau change chạy tương đương để có before/after comparison.

---

# 135. Dependency assessment

Dependency không chỉ là RPM package dependency.

Có ít nhất:

```text
Package dependency
Service dependency
Application dependency
Business dependency
```

Ví dụ update database library:

```text
RPM dependency:
packages

Runtime dependency:
application service

Business dependency:
customer portal
```

Infrastructure change cần hiểu nhiều layer.

---

# 136. Reboot assessment

Package update không phải lúc nào cũng cần reboot.

Nhưng cũng không được assume:

```text
dnf completed
=
system already using all updated code
```

Ví dụ updated library có thể chỉ được process dùng sau restart.

Updated kernel thường liên quan boot vào new kernel để sử dụng nó.

Exact reboot-detection tooling và policy phụ thuộc RHEL release/environment, nên trong enterprise phải theo approved maintenance runbook thay vì đoán.

---

# 137. Pre-maintenance checks

Mục đích pre-check là trả lời:

> “Trước change, hệ thống đang ở trạng thái nào?”

Nếu không có baseline, sau change bạn có thể đổ lỗi cho change về một problem đã có từ trước.

Pre-check thường liên quan:

```text
hostname
time
OS release
package baseline
service state
process state
network
ports
filesystem
capacity
recent errors
application health
dependencies
backup/recovery readiness
```

---

# 138. Stop conditions

Một professional change có **stop conditions**.

Ví dụ:

```text
backup not available
filesystem already critical
cluster/dependency unhealthy
package repository unexpected
pre-check already failing
scope differs from ticket
business owner cannot validate
```

Không phải:

> “Ticket đã approved nên cứ làm.”

Approval dựa trên assumptions. Nếu assumptions sai, stop và reassess.

---

# 139. Post-maintenance checks

Post-check phải mirror relevant pre-checks.

Ví dụ:

```text
Before       After
------       -----
package A    package A'
service OK   service OK
port open    port open
disk OK      disk OK
HTTP 200     HTTP 200
```

Một package command exit `0` không phải end of maintenance.

---

# 140. Recovery limitations

Đây là một điểm syllabus đặc biệt nhấn mạnh.

“Rollback” không phải magic undo.

Ví dụ package downgrade có thể không hoàn tác:

```text
configuration schema change
database migration
application state
data conversion
dependency update
kernel state
external API compatibility
```

Do đó rollback plan phải realistic.

Có lúc recovery phù hợp là:

```text
restore config
restore snapshot
restore backup
redeploy known-good build
fail over
vendor-assisted recovery
```

tùy system design.

---

# 141. DNF history không phải universal rollback solution

DNF transaction history rất hữu ích để audit package transactions.

Nhưng đừng nghĩ:

```text
dnf history
=
full server snapshot
```

Package-level reverse operation không đảm bảo external state/data/configuration quay lại chính xác trước change.

Đây chính là **recovery limitation** mà syllabus muốn bạn nhận thức.

---

# 142. Server log extraction — additive syllabus

Syllabus yêu cầu:

> Distinguish system journals, authentication logs and auditd records where auditing is configured; document retention, rotation, missing intervals and permissions rather than assuming all actions were logged. Infrastructure_Operations_14_Da…

Đây là bước chuyển từ:

```text
“xem log”
```

sang:

```text
“collect defensible evidence”
```

---

# 143. Log extraction phải có bounded time window

Bad:

```bash
journalctl > logs.txt
```

và nộp file hàng triệu lines.

Better:

```text
Incident:
2026-10-20 09:41 UTC

Collection:
09:35 UTC → 09:50 UTC
```

Then:

```bash
journalctl \
  --since "2026-10-20 09:35:00" \
  --until "2026-10-20 09:50:00"
```

Current Red Hat docs cũng hỗ trợ time-bounded filtering theo `--since`/`--until`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/risk_reduction_and_recovery_operations/index?utm_source=chatgpt.com)

---

# 144. Vì sao dùng UTC?

Infrastructure enterprise có thể trải nhiều timezone:

```text
Vietnam     UTC+7
Japan       UTC+9
AWS logs    often UTC
application custom timezone
```

Nếu timeline chỉ ghi:

```text
10:30
```

thì không đủ.

Nên ghi:

```text
2026-10-20T10:30:00+07:00
```

hoặc normalized UTC:

```text
2026-10-20T03:30:00Z
```

Day 2 additive yêu cầu bounded **UTC** extraction.

---

# 145. Host identity

Evidence phải cho biết log thuộc host nào.

Ví dụ:

```text
hostname:
rhel-app-01

environment:
LAB

asset:
APP-001
```

Một `error.log` không có identity/context rất yếu về audit traceability.

---

# 146. Exact command/filter

Phải lưu command:

```bash
journalctl ...
```

không chỉ output.

Vì reviewer cần biết:

```text
What did you query?
Which unit?
Which time?
Which priority?
What was excluded?
```

Một output có thể đúng nhưng extraction logic sai.

---

# 147. Record counts

Có thể lưu:

```text
Raw export:       5,284 records
Filtered output:     73 records
```

Điều này giúp:

```text
reconciliation
detect truncation
detect accidental filtering
repeatability
```

Count không chứng minh completeness nhưng là useful control.

---

# 148. Hash

Evidence file có thể hash:

```bash
sha256sum evidence-file
```

Ví dụ:

```text
SHA256:
abc...
```

Hash giúp phát hiện file đã thay đổi so với recorded baseline.

Nhưng phải hiểu:

```text
hash match
=
same bytes as hashed baseline
```

không có nghĩa:

```text
source was truthful
query was correct
time window complete
all relevant records were logged
```

Đây là evidence-integrity mindset đã bắt đầu từ Day 1 và được sử dụng tiếp ở Day 2.

---

# 149. Raw evidence và working copy

Strong practice:

```text
source export
     |
     +----> untouched original
     |
     +----> working copy
                |
                +-- grep
                +-- awk
                +-- sed
                +-- redaction
```

Không overwrite original bằng filtered version.

---

# 150. Native/lossless export vs readable output

Syllabus yêu cầu:

```text
native or lossless structured exports
+
readable filtered output
```

Infrastructure_Operations_14_Da…

Hai mục tiêu khác nhau.

### Native/lossless

Tối ưu cho:

```text
preservation
reprocessing
forensic/audit review
```

### Readable filtered

Tối ưu cho:

```text
human reviewer
incident timeline
ticket attachment
```

Không nên chọn một và bỏ một khi requirement yêu cầu cả hai.

---

# 151. Missing intervals

Ví dụ cần log:

```text
00:00–01:00 UTC
```

nhưng host log chỉ có:

```text
00:00–00:27
00:41–01:00
```

Bạn phải ghi:

```text
Collection gap:
00:27–00:41 UTC
Reason: unknown / service unavailable / retention issue
```

Không silently concatenate hai đoạn và giả vờ full window.

---

# 152. Permission limitations

Có thể user không được access:

```text
/var/log/audit/audit.log
```

Do đó report:

```text
Audit source not collected:
collector lacks authorized access.
Escalated to security/platform owner.
```

tốt hơn việc dùng `sudo` ngoài approval.

---

# 153. Correlating sudo/service events

Ví dụ ticket:

```text
CHG00123

Approved action:
Restart httpd

Window:
03:00–03:15 UTC
```

Bạn muốn reconstruct:

```text
03:04:22 sudo authorization
       ↓
03:04:24 systemctl restart request
       ↓
03:04:25 old process stopped
       ↓
03:04:26 new process started
       ↓
03:04:27 listening port available
       ↓
03:05:03 application health restored
```

Đây là evidence correlation.

Không nhất thiết một log chứa toàn bộ story.

Bạn combine:

```text
authentication evidence
systemd journal
application logs
network/service verification
ticket timeline
```

---

# 154. Không được giả định tất cả actions đều logged

Đây là một trong những lessons quan trọng nhất của Day 2.

Một action có thể thiếu audit trail vì:

```text
audit rule absent
log source disabled
retention expired
rotation removed file
service crashed
collector failed
permissions denied
clock incorrect
application never logged event
```

Correct report:

```text
No supporting record was available in source X for the queried window.
```

Không phải:

```text
The action did not occur.
```

Hai statement hoàn toàn khác nhau.

---

# 155. Security và least privilege xuyên suốt Day 2

Day 2 có nhiều command cần privilege:

```text
package installation
service changes
firewall changes
SELinux changes
mount operations
system configuration
```

Nguyên tắc:

```text
Read as normal user where possible.
Use sudo only for required operation.
Do not stay root unnecessarily.
Do not broaden access to solve a diagnostic problem.
```

Ví dụ bad workaround:

```bash
chmod -R 777 /app
```

hoặc:

```text
disable SELinux
disable firewall
```

chỉ để “xem có chạy không”.

Những workaround này phá security model và có thể che root cause.

---

# 156. Operational read-only vs state-changing commands

Một distinction bạn nên hình thành ngay:

| Read-only / diagnostic    | State-changing / cần thận trọng |
| ------------------------- | ------------------------------- |
| `rpm -q`                  | `dnf install/remove/update`     |
| `systemctl status`        | `systemctl restart/stop`        |
| `journalctl`              | thay logging config             |
| `getenforce`              | `setenforce`                    |
| `firewall-cmd --list-all` | add/remove firewall rule        |
| `ip addr show`            | thay IP/network config          |
| `ip route`                | add/delete route                |
| `findmnt`                 | `mount`/`umount`                |
| `df`                      | storage modification            |
| `crontab -l`              | `crontab -e`                    |

Khi incident đang active, ưu tiên cột trái trước.

---

# 157. Một integrated troubleshooting scenario

Giả sử user báo:

```text
https://app01.example.com unavailable
```

Một novice có thể làm:

```bash
systemctl restart httpd
```

Một Infrastructure Operations engineer sẽ reason:

```text
Is host reachable?
       |
       v
Does DNS resolve?
       |
       v
Does route exist?
       |
       v
Is TCP/443 reachable?
       |
       v
Is anything listening on 443?
       |
       v
Is service active?
       |
       v
Did service fail?
       |
       v
What does journal say?
       |
       +---- config error?
       |
       +---- permission?
       |
       +---- SELinux?
       |
       +---- certificate?
       |
       +---- filesystem?
       |
       +---- disk/inode?
       |
       +---- dependency?
```

Giả sử:

```bash
systemctl status httpd
```

cho:

```text
failed
```

Journal cho:

```text
cannot open certificate file
```

Check file:

```text
exists
permissions correct
```

Check filesystem:

```bash
findmnt /etc/pki/custom
```

không mounted.

Root cause:

```text
certificate filesystem failed to mount
```

Không phải:

```text
httpd bug
```

Đây chính xác là kiểu dependency reasoning Day 2 hướng tới.

---

# 158. Một integrated maintenance scenario

Ticket:

```text
Upgrade approved package X on RHEL app server
```

Correct reasoning:

```text
Identify server
↓
Verify correct environment
↓
Check RHEL release
↓
Verify package baseline
↓
Verify repository/subscription
↓
Check package dependency
↓
Check application dependency
↓
Determine possible restart/reboot
↓
Verify backup/recovery
↓
Pre-check
↓
Apply scoped change
↓
Verify package transaction
↓
Restart only required services
↓
Verify process/port/application
↓
Review journal
↓
Post-check
↓
Capture evidence
```

Nếu package installs nhưng application validation fails:

```text
change not successful
```

dù:

```text
dnf exit code = 0
```

---

# 159. Mental model: “command success” vs “operation success”

Đây là principle lớn nhất Day 2.

Ví dụ:

```text
systemctl restart app
exit 0
```

chỉ chứng minh command request được xử lý thành công ở mức nhất định.

Operation success còn cần:

```text
service active
port listening
application responsive
dependency healthy
no new critical log
business check passed
```

Tương tự:

```text
tar exit 0
```

không chứng minh restore.

```text
dnf exit 0
```

không chứng minh application.

```text
cron entry exists
```

không chứng minh scheduled job.

```text
mount entry exists
```

không chứng minh mount.

Day 2 dạy bạn chuyển từ **command operator** thành **operations engineer**.

---

# 160. Troubleshooting evidence record

Một Linux troubleshooting record tốt có thể chứa:

```text
Ticket:
INC001234

Host:
rhel-app-01

Environment:
LAB

Incident start:
2026-10-20T03:41:00Z

Symptom:
HTTPS unavailable

Initial state:
httpd.service failed

Checks:
systemctl status httpd
journalctl -u httpd ...
ss -lntp
findmnt ...
df -h
getenforce
...

Root cause:
...

Approved corrective action:
...

Post-check:
...

Residual risk:
...

Escalation:
...
```

Đây là một phần deliverable chính thức của Day 2 assignment. Infrastructure_Operations_14_Da…

---

# 161. L1/L2 reasoning boundary trong Day 2

Một L1/L2 Infrastructure Operations engineer thường được kỳ vọng có thể:

```text
collect status
read logs
identify common failures
validate packages/services
check filesystem/network/security state
perform approved service operations
execute documented rollback
collect evidence
```

Nhưng thường cần escalate nếu cần:

```text
kernel debugging
filesystem repair with data risk
storage-array intervention
custom SELinux policy engineering
unknown application-code defect
unsupported package conflicts
security compromise analysis
complex routing/firewall architecture changes
unapproved production changes
```

Boundary cụ thể phụ thuộc organization/runbook.

---

# 162. Common Day 2 mistakes cần loại bỏ

Đây là nhóm lỗi tư duy cần tránh:

1. Thấy service down → restart ngay.
2. Thấy `Permission denied` → `chmod 777`.
3. Nghi SELinux → disable SELinux.
4. Nghi firewall → stop firewall.
5. DNS fail → kết luận host down.
6. Ping fail → kết luận network dead.
7. `df -h` còn space → bỏ qua inode.
8. Có `/etc/fstab` entry → assume mounted.
9. Có cron entry → assume job chạy.
10. Có backup file → assume restore được.
11. Package install exit `0` → close ticket.
12. Không thấy log → kết luận event không xảy ra.
13. `kill -9` là lựa chọn đầu tiên.
14. Reboot để tránh root-cause investigation.
15. Filter log rồi overwrite raw evidence.
16. Không ghi timezone.
17. Không capture pre-change baseline.
18. “Rollback” nhưng không biết rollback có thể hoàn tác data/config hay không.

---

# 163. Day 2 troubleshooting decision tree rút gọn

```text
                Service/Application problem
                         |
                         v
               Is host/OS reachable?
                  /             \
                no               yes
                |                 |
          network/host            v
           escalation      Service state?
                              |
                  +-----------+----------+
                  |                      |
                failed                 active
                  |                      |
                  v                      v
               journal               listener?
                  |                 /          \
                  v               no            yes
            config/deps             |             |
          permission/storage       config/       v
             SELinux              bind/port    network
                  |                            firewall
                  v                              DNS
             corrective                           |
               action                             v
                  |                           application
                  v                           dependency
               verify
```

Đây chỉ là guide; real incidents không luôn tuyến tính.

---

# 164. Evidence decision tree

```text
Need evidence
    |
    v
Define ticket/event
    |
    v
Define hostname/source
    |
    v
Define UTC window
    |
    v
Identify available sources
    |
    +-- journal
    |
    +-- auth/syslog
    |
    +-- auditd?
    |
    +-- application log
    |
    +-- rotated files
    |
    v
Export original
    |
    v
Record command/filter
    |
    v
Count/hash
    |
    v
Create working copy
    |
    v
Filter/correlate
    |
    v
Document gaps
```

---

# 165. Day 2 Theory — command map cần nhớ

Bạn chưa cần thuộc lòng mọi option, nhưng cần biết **tool nào trả lời câu hỏi nào**:

| Câu hỏi                        | Tool/command family                      |
| ------------------------------ | ---------------------------------------- |
| OS version?                    | `/etc/os-release`, `/etc/redhat-release` |
| Package installed/version?     | `rpm -q`, `dnf info`                     |
| Repository available?          | `dnf repolist`                           |
| Environment?                   | `env`, `printenv`, `echo`                |
| Process?                       | `ps`, `top`                              |
| Service healthy?               | `systemctl`                              |
| Service logs?                  | `journalctl -u`                          |
| Boot logs?                     | `journalctl -b`                          |
| SELinux mode?                  | `getenforce`, `sestatus`                 |
| SELinux context?               | `ls -Z`, `ps -eZ`                        |
| Firewall configuration?        | `firewall-cmd`                           |
| IP/interface?                  | `ip addr`, `ip link`                     |
| Route?                         | `ip route`, `ip route get`               |
| Socket/listener?               | `ss`                                     |
| Application HTTP/TLS?          | `curl`                                   |
| Basic reachability?            | `ping`                                   |
| Network path?                  | `tracepath`, `traceroute`                |
| Block devices?                 | `lsblk`, `blkid`                         |
| Mounted filesystem?            | `findmnt`                                |
| Disk capacity?                 | `df -h`                                  |
| Inode capacity?                | `df -i`                                  |
| Directory usage?               | `du`                                     |
| Search text?                   | `grep`                                   |
| Fields/reporting?              | `awk`                                    |
| Text transformation?           | `sed`                                    |
| Schedule?                      | `cron`/`crontab`                         |
| Archive?                       | `tar`                                    |
| Synchronize?                   | `rsync`                                  |
| Integrity comparison?          | `sha256sum`                              |
| Audit search where configured? | `ausearch`                               |

---

# 166. Kiến thức bổ sung — những dependency quan trọng giữa các topic

Syllabus liệt kê từng topic riêng, nhưng trong thực tế chúng liên kết rất chặt:

```text
DNF installs service
        ↓
systemd starts process
        ↓
process gets environment
        ↓
process reads files
        ↓
filesystem must be mounted
        ↓
Unix permissions must allow
        ↓
SELinux must allow
        ↓
process opens TCP socket
        ↓
firewalld must allow client
        ↓
routing/DNS must work
        ↓
logs record behavior
        ↓
Bash/cron automate checks
        ↓
backup protects data/config
        ↓
restore proves recoverability
```

Nếu bạn hiểu dependency chain này, phần thực hành Day 2 sẽ dễ hơn rất nhiều.

---

# 167. Kiến thức bổ sung — “five states” cần luôn suy nghĩ

Khi quản trị một object như service, filesystem hoặc firewall, hãy tự hỏi năm trạng thái:

```text
1. Intended state
2. Configured state
3. Runtime state
4. Observed/verified state
5. Recoverable state
```

Ví dụ service:

```text
Intended:
should start automatically

Configured:
enabled

Runtime:
active

Observed:
HTTP check successful

Recoverable:
known config backup + restart/backout procedure
```

Điểm này giúp tránh việc coi `systemctl status` là toàn bộ truth.

---

# 168. Kiến thức bổ sung — “three proofs” của Operations

Một change chỉ đáng tin khi có ba proof:

### Proof 1 — Configuration proof

Bạn đã cấu hình đúng chưa?

Ví dụ:

```text
service enabled
firewall rule present
fstab entry present
```

### Proof 2 — Runtime proof

Hệ thống có thực sự thực thi config đó không?

Ví dụ:

```text
process running
listener present
filesystem mounted
```

### Proof 3 — Functional proof

Business/application có thực sự hoạt động không?

Ví dụ:

```text
curl returns expected status
file readable
backup restored correctly
```

Nếu chỉ có Proof 1, chưa đủ.

---

# 169. Kiến thức bổ sung — “evidence before action”

Một mature operations workflow ưu tiên:

```text
Capture
→ Act
→ Capture again
```

Hay:

```text
Before
→ Change
→ After
```

Ví dụ:

```bash
rpm -q package
systemctl status service
```

sau đó change.

Rồi chạy lại:

```bash
rpm -q package
systemctl status service
```

Nhờ vậy reviewer có thể thấy actual delta.

---

# 170. Module Lý thuyết Day 2 — checklist hoàn tất

Sau khi học xong phần trên, bạn phải giải thích được ít nhất các nội dung sau:

- `RPM`, `DNF`, `YUM`, package, dependency và repository khác nhau thế nào; vì sao repository/subscription phải được kiểm tra trước package maintenance.
- Environment variable hoạt động ra sao; vì sao interactive shell, `systemd` và `cron` có thể có environment khác nhau.
- Process, PID, PPID, process states và `SIGTERM`/`SIGKILL`/`SIGHUP` khác nhau thế nào.
- CPU, memory và load phải được interpret theo context thay vì nhìn một number đơn lẻ.
- `systemd` unit, `.service`, target, dependency, `active`, `enabled`, `disabled`, `masked` và `daemon-reload`.
- Boot targets, startup dependency và cách reasoning khi service không lên sau reboot.
- SELinux `Enforcing`, `Permissive`, `Disabled`, security context, AVC và vì sao không disable SELinux như default fix.
- `firewalld` zone, service, port, runtime/permanent configuration.
- IP, prefix, gateway, routing table, DNS, TCP, UDP và ports.
- Khi nào dùng `ip`, `ss`, `ping`, `tracepath`/`traceroute` và `curl`.
- Disk, partition, filesystem, mount point, UUID và `/etc/fstab`.
- Sự khác nhau giữa disk-block exhaustion và inode exhaustion.
- `journald`, `journalctl`, `rsyslog`, authentication logs, Audit records và log rotation.
- Vì sao “không thấy log” không đồng nghĩa với “event không xảy ra”.
- `grep`, `awk`, `sed` và Regex được dùng thế nào trong Operations.
- Bash variables, quoting, conditionals, loops, exit code và error handling.
- Vì sao pipeline/script có thể tạo misleading success.
- `cron` scheduling và cron environment.
- Archive/compression, `tar`, `rsync`, backup và restore khác nhau thế nào.
- Vì sao **backup success không bằng recovery success**.
- Cách xây health-check từ nhiều layer.
- Cách troubleshoot theo least-risk-first.
- Khi nào restart/reboot có thể làm mất evidence hoặc tăng impact.
- Cách xác định L1/L2 boundary và escalation.
- Cách chuẩn bị RHEL maintenance: release/repository/dependency/reboot/precheck/postcheck/recovery.
- Cách extract bounded UTC log window, preserve raw evidence, record filter/count/hash và document evidence gaps.
- Cách correlate ticket → privileged action → `systemd` event → service recovery.

Toàn bộ những mục trên chính là coverage của **Module Lý thuyết Day 2**, bao gồm cả phần additive về RHEL maintenance và server-log extraction theo syllabus. Infrastructure_Operations_14_Da…

**Module Lý thuyết Day 2 đã hoàn tất về phạm vi kiến thức.** Phần tiếp theo khi bạn yêu cầu **“Module Thực hành Day 2”** sẽ biến toàn bộ kiến thức này thành một lab RHEL có trình tự: environment preparation → package/service → process/environment → SELinux/firewall/network → storage → logging → Bash/cron → backup/restore → Failure Injection → maintenance ticket → bounded UTC evidence extraction → troubleshooting record → cleanup/rollback.
