# Day 2 — Module Lý thuyết

## Linux/RHEL System Administration & Operational Troubleshooting

Theo syllabus, toàn bộ phần **Concept/Lecture của Day 2** bao gồm: Linux/RHEL system administration; `dnf/yum/rpm`; repository và subscription; environment variables; processes, process states, signals, resource usage; `systemd`, service lifecycle, boot targets, startup/recovery; SELinux; `firewalld`; IP addressing, routing, DNS, TCP/UDP; `ping`, `traceroute/tracepath`, `curl`, `ss`, `ip`; disks, partitions, filesystems, mounts, `/etc/fstab`, disk/inode capacity; `journalctl`, syslog, log rotation; `grep/awk/sed`, regular expressions; Bash variables/control flow, exit codes/error handling; cron; compression; `tar`/`rsync`; restore; health checks và operational troubleshooting. Infrastructure_Operations_14_Da…

Đây là **module Lý thuyết** nên trọng tâm không phải “gõ command nào”, mà là hiểu **Linux/RHEL hoạt động như thế nào, mỗi layer phụ thuộc vào layer nào, dấu hiệu lỗi xuất hiện ở đâu, kiểm tra theo thứ tự nào và giới hạn xử lý của Infrastructure Operations**.

---

# I. Mental model quan trọng nhất của Day 2

Một Linux/RHEL server không nên được nhìn như một tập hợp command. Hãy hình dung nó như một stack:

```text
Business/Application Service
            │
            ▼
Application / Package
            │
            ▼
Process
            │
            ▼
systemd Service
            │
   ┌────────┼────────┐
   ▼        ▼        ▼
Security  Network   Storage
SELinux   IP/DNS    Filesystem
Firewall  Route     Mount
   │        │        │
   └────────┼────────┘
            ▼
       Linux Kernel
            │
            ▼
        Hardware/VM
```

Xung quanh toàn bộ stack là:

```text
Logging
Monitoring
Automation
Backup/Restore
Security Controls
Change Control
Evidence
```

Vì vậy một symptom đơn giản:

```text
https://server/app không truy cập được
```

có thể xuất phát từ rất nhiều root cause:

```text
Package chưa cài
Service không chạy
Process crash
Port không listen
Bind sai IP
firewalld block
SELinux block
DNS sai
Route sai
Filesystem full
inode full
Configuration syntax error
Certificate/application issue
Dependency unavailable
```

Đây là lý do Day 2 tập trung vào **operational troubleshooting**, không chỉ Linux commands.

---

# II. Linux/RHEL System Administration

## 1. Linux system administration thực chất là gì?

### What

Linux system administration là việc duy trì operating system và services sao cho hệ thống:

- available;
- correctly configured;
- secure;
- observable;
- recoverable;
- auditable.

Một Infrastructure Operations administrator thường không chỉ “sửa Linux”. Bạn quản lý trạng thái của system.

Ví dụ:

```text
Desired State
httpd installed
httpd enabled
httpd active
TCP/443 reachable
SELinux enforcing
Firewall only permits required ports
Filesystem < 80%
Backup succeeded
```

so với:

```text
Actual State
```

Nhiệm vụ vận hành là liên tục xác minh:

```text
Desired State == Actual State ?
```

---

## 2. Administrative action và operational risk

Các hành động Linux có thể chia theo mức tác động.

### Read-only

Ví dụ:

```bash
uname -a
cat /etc/os-release
systemctl status sshd
ip addr
ip route
ss -lntp
df -h
journalctl -u sshd
```

Mục tiêu là quan sát mà không thay đổi system state.

Đây thường là bước đầu của troubleshooting.

### State-changing

Ví dụ:

```bash
systemctl restart httpd
dnf install httpd
firewall-cmd --add-service=http
mount /dev/sdb1 /data
```

Các command này làm thay đổi trạng thái.

Trong enterprise environment, state-changing action thường cần:

```text
Authorization
Impact assessment
Rollback plan
Evidence
```

### Destructive hoặc high-risk

Ví dụ:

```bash
rm -rf ...
mkfs ...
fdisk ...
dnf remove ...
reboot
```

Nếu chạy sai target, hậu quả có thể nghiêm trọng.

Do đó nguyên tắc:

> **Read first, change second.**

Và:

> **Không sử dụng một command chỉ vì nó có khả năng “làm lỗi biến mất”; cần hiểu system state mà command đó thay đổi.**

---

# III. Package Management — `dnf`, `yum`, `rpm`

Package management là layer quản lý software của Linux.

---

# 1. RPM package là gì?

RHEL sử dụng RPM package format.

Một RPM package chứa:

```text
Application binaries
Libraries
Configuration templates
Documentation
Metadata
Version
Architecture
Dependencies
Package scripts
File ownership information
```

Ví dụ package:

```text
httpd-2.x.x-....x86_64.rpm
```

Tên package thường phản ánh:

```text
name
version
release
architecture
```

---

# 2. `rpm` là gì?

`rpm` là lower-level package management utility.

Nó làm việc với:

```text
RPM package file
RPM package database
```

Ví dụ query:

```bash
rpm -q httpd
```

Ý nghĩa:

```text
rpm
-q    query
httpd package name
```

Nếu package đã cài:

```text
httpd-<version>
```

Nếu chưa:

```text
package httpd is not installed
```

Các query concept quan trọng:

```bash
rpm -qa
rpm -qi <package>
rpm -ql <package>
rpm -qf <file>
```

Conceptually:

```text
rpm -qa
```

= query tất cả installed packages.

```text
rpm -qi
```

= package information.

```text
rpm -ql
```

= package cung cấp những file nào.

```text
rpm -qf /path/file
```

= file này thuộc package nào.

---

# 3. Vì sao không nên mặc định dùng `rpm -i` để cài package?

Vì `rpm` ở level thấp hơn không cung cấp dependency resolution đầy đủ giống `dnf`.

Ví dụ package A cần:

```text
library B
package C
```

`rpm` có thể báo missing dependency.

Trong khi:

```bash
dnf install packageA
```

có thể tìm dependency từ repository và xử lý transaction.

---

# 4. DNF là gì?

`dnf` là package manager ở level cao hơn.

Nó quản lý:

```text
Repositories
Metadata
Dependency resolution
Install
Remove
Update
Transaction
Package groups
```

Ví dụ:

```bash
dnf install httpd
```

flow conceptual:

```text
Administrator
      │
      ▼
     DNF
      │
      ├── Read repository metadata
      │
      ├── Find package
      │
      ├── Resolve dependencies
      │
      ├── Download RPMs
      │
      ├── Validate package
      │
      └── Perform RPM transaction
```

Red Hat hiện dùng `dnf` làm software management tool trên RHEL hiện đại; documentation RHEL 10 cũng mô tả package installation và dependency handling thông qua `dnf`. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/managing_software_with_the_dnf_tool/index?utm_source=chatgpt.com)

---

# 5. `yum` và `dnf`

Đây là nơi cần lưu ý version.

Trên các thế hệ RHEL cũ, `yum` là package manager quen thuộc.

Trên RHEL 8/9, DNF là implementation chính; Red Hat vẫn giữ `yum` để compatibility với workflow cũ. Với RHEL 9, Red Hat mô tả `yum` là compatibility alias cho DNF. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/pdf/managing_software_with_the_dnf_tool/Red_Hat_Enterprise_Linux-9-Managing_software_with_the_DNF_tool-en-US.pdf?utm_source=chatgpt.com)

Vì syllabus ghi:

```text
dnf/yum/rpm
```

bạn cần biết cả ba vì enterprise estate có thể bao gồm nhiều RHEL generation.

Operationally:

```text
RHEL 7 legacy estate → thường thấy yum
RHEL 8/9+           → DNF-based workflow
RPM                  → low-level package/database inspection
```

Không nên giả định tất cả server trong enterprise đều cùng major version.

---

# 6. Repository

Repository là nguồn chứa packages và metadata.

DNF không tự “biết Internet có package gì”.

Nó đọc các configured repositories.

Conceptual:

```text
dnf install package
      │
      ▼
Read enabled repositories
      │
      ▼
Read metadata
      │
      ▼
Find matching package
      │
      ▼
Resolve dependencies
      │
      ▼
Download/install
```

Repository configuration thường liên quan tới:

```text
/etc/yum.repos.d/*.repo
```

và DNF configuration:

```text
/etc/dnf/dnf.conf
```

Red Hat hiện khuyến nghị custom repository nên được định nghĩa bằng `.repo` riêng thay vì nhồi tất cả vào global DNF configuration. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/managing-custom-software-repositories?utm_source=chatgpt.com)

---

# 7. BaseOS và AppStream

Trong RHEL hiện đại, bạn sẽ thường gặp:

```text
BaseOS
AppStream
```

BaseOS chứa phần nền tảng của operating system.

AppStream chứa nhiều user-space applications, runtimes và các thành phần workload.

Trong RHEL 10 documentation, Red Hat xác định cả BaseOS và AppStream là các content set quan trọng của RHEL. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/managing_software_with_the_dnf_tool/configuring-dnf?utm_source=chatgpt.com)

Điều cần nhớ ở Operations:

```text
Package not found
```

không nhất thiết nghĩa là package “không tồn tại”.

Có thể là:

```text
repository disabled
subscription/content access issue
repository metadata stale
DNS issue
proxy issue
network issue
repository URL unavailable
wrong RHEL version
wrong architecture
```

---

# 8. Subscription awareness

RHEL không chỉ là “Linux có file RPM”.

Trong environment có Red Hat subscription/content infrastructure, package availability có thể phụ thuộc:

```text
System registration
Entitlement/content access model
Enabled repositories
Network path to content source
Internal Satellite/repository architecture
```

Infrastructure operator không nhất thiết là người quản lý subscription policy, nhưng phải biết nhận diện khi package failure có vẻ thuộc content/subscription layer để escalate đúng team.

---

# 9. Security implications của package management

Cài một package đồng nghĩa đưa executable code vào server.

Do đó không nên:

```text
download random RPM
rpm -i random-package.rpm
```

trên production.

Official RHEL documentation cũng cảnh báo rằng package từ unverified/untrusted repositories có thể gây security, stability, compatibility và maintainability risk. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/managing-custom-software-repositories?utm_source=chatgpt.com)

Operational requirements thường gồm:

```text
Approved repository
Approved package/version
Change ticket
Security validation
Compatibility confirmation
Rollback plan
```

---

# 10. Troubleshooting package management

Một flow tốt:

```text
dnf install fails
      │
      ▼
Package name correct?
      │
      ▼
Repository enabled?
      │
      ▼
Metadata accessible?
      │
      ▼
DNS works?
      │
      ▼
Route works?
      │
      ▼
Proxy/firewall path works?
      │
      ▼
Subscription/content access valid?
      │
      ▼
Dependency/version conflict?
```

Đừng bắt đầu bằng việc xoá cache, disable security hoặc thêm repository lạ.

---

# IV. Environment Variables

## 1. Environment variable là gì?

Environment variable là named value được truyền cho process.

Ví dụ:

```text
PATH
HOME
USER
SHELL
LANG
TZ
```

Có thể xem:

```bash
env
printenv
```

Ví dụ:

```bash
echo "$PATH"
```

---

# 2. `PATH`

`PATH` chứa danh sách directory mà shell tìm executable.

Ví dụ:

```text
/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin
```

Khi chạy:

```bash
curl
```

shell sẽ tìm executable theo `PATH`.

Nếu:

```text
/usr/local/myapp/bin
```

không có trong `PATH`, user có thể nhận:

```text
command not found
```

trong khi binary vẫn tồn tại.

Có thể chạy bằng absolute path:

```bash
/usr/local/myapp/bin/mycommand
```

---

# 3. Shell variable vs environment variable

Ví dụ:

```bash
NAME=test
```

đây là shell variable.

Nếu:

```bash
export NAME
```

thì variable được đưa vào environment để child processes kế thừa.

Concept:

```text
Parent Shell
   NAME=test
       │
       ├── not exported
       │      ↓
       │ child không nhận
       │
       └── exported
              ↓
          child nhận NAME
```

---

# 4. Environment inheritance

Một process không tự động đọc `.bashrc` của user theo cách bạn nghĩ.

Đây là lý do:

```text
Script chạy bằng interactive SSH shell → OK
cron → FAIL
systemd → FAIL
```

có thể xảy ra.

Vì:

```text
Interactive shell environment
≠
cron environment
≠
systemd service environment
```

Đây là một operational lesson cực kỳ quan trọng.

Script tốt nên sử dụng:

```text
explicit paths
explicit configuration
predictable environment
```

thay vì vô tình phụ thuộc vào shell profile.

---

# V. Processes

## 1. Process là gì?

Program là executable code trên disk.

Process là một running instance của program.

Ví dụ:

```text
/usr/sbin/sshd
```

là executable.

Một instance đang chạy sẽ có:

```text
PID
PPID
UID
GID
state
CPU usage
memory usage
open files
environment
```

---

# 2. PID và PPID

PID:

```text
Process ID
```

PPID:

```text
Parent Process ID
```

Linux tạo process hierarchy.

Ví dụ:

```text
systemd PID 1
   │
   ├── sshd
   │    └── ssh session
   │          └── bash
   │
   └── httpd
        ├── worker
        └── worker
```

Hiểu process tree giúp tìm:

- process nào spawn process lỗi;
- daemon chính là process nào;
- child process nào bị stuck;
- script được launch từ cron hay service.

---

# 3. `ps`

`ps` cho snapshot process state.

Ví dụ quen thuộc:

```bash
ps aux
```

hoặc:

```bash
ps -ef
```

Các thông tin thường cần đọc:

```text
USER
PID
PPID
CPU
MEM
START
COMMAND
```

---

# 4. Process state

Linux process có nhiều state. Ở foundation level, cần nhận diện:

```text
R = running/runnable
S = interruptible sleep
D = uninterruptible sleep
T = stopped/traced
Z = zombie
```

### `R`

Process đang chạy hoặc ready để scheduler chạy.

### `S`

Process đang ngủ/chờ event bình thường.

Đây không phải lỗi.

### `D`

Uninterruptible sleep, thường liên quan I/O wait.

Nếu nhiều process stuck lâu ở `D`, cần nghĩ đến:

```text
disk
storage
NFS
I/O subsystem
```

### `T`

Process stopped hoặc traced.

### `Z`

Zombie process.

Process đã kết thúc execution nhưng parent chưa thu nhận exit status.

Zombie không thực sự tiêu thụ CPU như process hoạt động, nhưng số lượng lớn có thể báo hiệu vấn đề trong parent application/process management.

---

# 5. `top`

`top` cung cấp realtime-ish view về:

```text
load
CPU
memory
swap
processes
```

Nhưng `top` chỉ là một observation tool.

Đừng thấy:

```text
CPU 90%
```

rồi lập tức kill process.

Bạn phải hỏi:

```text
CPU 90% trong bao lâu?
Process nào?
Expected workload?
Business impact?
Load average?
I/O wait?
Recent change?
```

---

# VI. Signals

Linux sử dụng signals để gửi asynchronous notification cho process.

---

# 1. `SIGTERM`

Thường là signal dùng để yêu cầu process terminate một cách có kiểm soát.

Ví dụ:

```bash
kill <PID>
```

mặc định thường gửi:

```text
SIGTERM
```

Application có cơ hội:

```text
close files
finish transaction
flush buffers
release locks
clean temporary data
```

---

# 2. `SIGKILL`

```text
SIGKILL
```

không thể bị process catch hoặc ignore.

Ví dụ:

```bash
kill -9 <PID>
```

Kernel terminate process ngay.

Điều này mạnh nhưng rủi ro hơn.

Có thể để lại:

```text
incomplete transaction
stale temporary state
unflushed application data
recovery operation
```

Do đó:

```text
SIGTERM first
SIGKILL only when justified
```

---

# 3. `SIGHUP`

Historically thường được daemon sử dụng để:

```text
reload configuration
reopen log files
```

Nhưng behavior phụ thuộc application.

Không được giả định service nào cũng xử lý `SIGHUP` giống nhau.

---

# 4. `SIGINT`

Thường tương ứng với:

```text
Ctrl+C
```

trong terminal.

Nó yêu cầu foreground process interrupt.

---

# VII. Resource Usage

Infrastructure Operations phải hiểu ít nhất bốn nhóm resource:

```text
CPU
Memory
Storage
Network
```

và process/resource relationship.

---

# 1. CPU

CPU utilization cao có thể là:

```text
normal workload
runaway process
compression
backup
batch processing
application loop
security scan
```

Một snapshot CPU cao không đủ kết luận incident.

---

# 2. Load average

Load average không giống đơn giản với `%CPU`.

Nó phản ánh số task đang:

```text
runnable
hoặc trong một số loại uninterruptible wait
```

Do đó:

```text
load cao
```

có thể liên quan CPU hoặc I/O pressure.

Cần đọc load tương đối với số CPU cores và workload pattern.

---

# 3. Memory

Linux chủ động dùng memory cho caching.

Do đó:

```text
free memory thấp
```

không đồng nghĩa chắc chắn:

```text
memory exhausted
```

Cần quan tâm:

```text
available memory
swap usage
paging behavior
OOM events
application RSS
```

---

# 4. Swap

Swap được dùng khi kernel quản lý memory pressure.

Swap usage > 0 tự nó chưa phải incident.

Nhưng:

```text
heavy sustained swap
performance degradation
memory pressure
```

cần investigate.

---

# 5. OOM

Nếu memory nghiêm trọng, Linux có thể gặp Out Of Memory situation.

OOM-related evidence có thể nằm trong:

```text
kernel/journal logs
```

Một application “tự nhiên biến mất” có thể thực tế đã bị OOM handling terminate.

---

# VIII. `systemd`

Đây là một trong những nội dung quan trọng nhất Day 2.

---

# 1. `systemd` là gì?

`systemd` là system/service manager trên RHEL hiện đại.

PID 1 thường là:

```text
systemd
```

Nó quản lý nhiều loại unit.

Red Hat documentation hiện liệt kê các unit type như:

````text
service
target
device
mount
timer
``` :chatgpt-content-reference{index="6"}


---

# 2. Unit

Unit là object mà `systemd` quản lý.

Ví dụ:

```text
sshd.service
multi-user.target
data.mount
dnf-automatic.timer
````

Không phải unit nào cũng là service.

---

# 3. Service unit

Service unit thường có extension:

```text
.service
```

Ví dụ:

```text
sshd.service
httpd.service
firewalld.service
```

Service unit mô tả:

```text
how to start
how to stop
dependencies
environment
restart behavior
install target
```

---

# 4. Unit file locations

Hiện nay RHEL documentation phân biệt ba location quan trọng:

```text
/usr/lib/systemd/system/
/run/systemd/system/
/etc/systemd/system/
```

Package-provided units thường ở `/usr/lib/systemd/system/`; runtime units có thể ở `/run/systemd/system/`; administrator customizations và overrides thường thuộc `/etc/systemd/system/`. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_systemd_unit_files_to_customize_and_optimize_your_system/using_systemd_unit_files_to_customize_and_optimize_your_system?utm_source=chatgpt.com)

Operational lesson:

> Không nên trực tiếp sửa vendor unit dưới `/usr/lib/systemd/system/` nếu mục đích là persistent customization, vì package update có thể thay đổi nó.

---

# 5. Cấu trúc service unit

Một service unit thường có:

```ini
[Unit]
...

[Service]
...

[Install]
...
```

### `[Unit]`

Metadata và dependency/order relationship.

Ví dụ:

```ini
Description=...
After=network.target
```

### `[Service]`

Cách process được chạy.

Ví dụ:

```ini
ExecStart=/path/program
```

### `[Install]`

Liên quan enable relationship.

Ví dụ:

```ini
WantedBy=multi-user.target
```

---

# 6. `daemon-reload`

Sau khi administrator tạo/sửa unit configuration, `systemd` cần reload unit definitions:

```bash
systemctl daemon-reload
```

Red Hat cũng nhấn mạnh cần `daemon-reload` sau khi tạo hoặc sửa unit file để systemd cập nhật dependency/configuration tree. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/working-with-systemd-unit-files?utm_source=chatgpt.com)

Lưu ý:

```text
daemon-reload
≠
restart application
```

Nó reload **systemd configuration**, không đồng nghĩa restart mọi service.

---

# IX. Service Lifecycle

Các operation cần phân biệt thật rõ.

---

## 1. `start`

```bash
systemctl start httpd
```

Ý nghĩa:

```text
Start service now
```

Không có nghĩa service sẽ tự start sau reboot.

---

## 2. `stop`

```bash
systemctl stop httpd
```

Dừng service hiện tại.

Không nhất thiết disable boot startup.

---

## 3. `restart`

```bash
systemctl restart httpd
```

Thông thường:

```text
stop + start
```

Có thể gây interruption.

Do đó production restart phải cân nhắc impact.

---

## 4. `reload`

```bash
systemctl reload httpd
```

Yêu cầu service reload configuration mà không full restart nếu service hỗ trợ.

Không phải service nào cũng support reload; Red Hat cũng lưu ý điểm này trong systemd documentation. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/managing-system-services-with-systemctl?utm_source=chatgpt.com)

---

## 5. `enable`

```bash
systemctl enable httpd
```

Cấu hình service tham gia boot dependency/startup relationship.

Nó không hoàn toàn đồng nghĩa:

```text
start now
```

---

## 6. `enable --now`

```bash
systemctl enable --now httpd
```

Kết hợp:

```text
enable boot startup
+
start immediately
```

---

## 7. `disable`

```bash
systemctl disable httpd
```

Gỡ enable relationship cho boot.

Nhưng service vẫn có thể được start manually hoặc do dependency tùy trường hợp.

---

## 8. `mask`

`mask` mạnh hơn `disable`.

```bash
systemctl mask <service>
```

ngăn unit được start theo cách thông thường cho đến khi unmask.

Red Hat cũng phân biệt: `disable` ngăn automatic boot start, trong khi `mask` ngăn unit được start cả manually cho tới khi được unmask. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/managing-system-services-with-systemctl?utm_source=chatgpt.com)

Đây là distinction rất hay được hỏi trong interview.

---

# X. Service State

Các state thường gặp:

```text
active
inactive
failed
activating
deactivating
```

Nhưng cần phân biệt:

```text
Loaded state
Active state
Sub-state
Enabled state
```

Ví dụ:

```text
service active nhưng disabled
```

hoàn toàn có thể xảy ra.

Nghĩa là:

```text
đang chạy hiện tại
nhưng không configured tự start theo enable relationship
```

Ngược lại:

```text
enabled nhưng failed
```

cũng có thể xảy ra.

`enabled` không chứng minh service healthy.

---

# XI. systemd Dependencies

Một service có thể phụ thuộc:

```text
network
filesystem
other service
socket
mount
```

Ví dụ:

```text
Application service
      │
      ├── database service
      ├── /data mount
      └── network
```

Do đó application failure có thể chỉ là downstream symptom.

Ví dụ:

```text
app.service failed
```

root cause:

```text
/data không mount
```

thay vì binary application có lỗi.

---

# XII. Boot Targets

`systemd target` là group của units và synchronization point.

Các target quan trọng trong syllabus context:

```text
multi-user.target
graphical.target
rescue.target
emergency.target
```

Red Hat hiện mô tả các target trên như common system targets. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/booting-into-a-target-system-state?utm_source=chatgpt.com)

---

## 1. `multi-user.target`

System ở trạng thái multi-user, thường phù hợp server operation không yêu cầu graphical login.

Conceptually gần với traditional runlevel 3.

---

## 2. `graphical.target`

Bao gồm graphical login environment.

Thường dựa trên `multi-user.target`.

---

## 3. `rescue.target`

Minimal recovery environment với nhiều thành phần cơ bản hơn emergency.

Dùng cho system recovery có kiểm soát.

---

## 4. `emergency.target`

Minimal hơn nữa.

Dùng khi system boot nghiêm trọng và cần low-level intervention.

---

# XIII. Service Startup and Recovery

Một trong các mục tiêu Day 2 là học cách phục hồi deliberately failed service.

Điều quan trọng nhất:

> Service failed không phải là root cause.

Một service có thể failed vì:

```text
configuration syntax
missing file
permissions
SELinux denial
port conflict
dependency unavailable
mount missing
certificate issue
invalid environment
binary/library issue
resource exhaustion
```

---

## Standard service troubleshooting flow

```text
1. Confirm symptom
        ↓
2. systemctl status
        ↓
3. Review journal
        ↓
4. Identify startup command/config
        ↓
5. Validate dependency
        ↓
6. Validate file permissions
        ↓
7. Validate SELinux
        ↓
8. Validate required port
        ↓
9. Validate filesystem/resource
        ↓
10. Correct authorized root cause
        ↓
11. Start/restart
        ↓
12. Verify process + port + function
```

Điểm quan trọng:

```text
systemctl status = một nguồn evidence
```

không phải toàn bộ diagnosis.

---

# XIV. SELinux

SELinux là phần rất quan trọng của RHEL operations.

---

# 1. SELinux là gì?

SELinux là Mandatory Access Control framework.

Traditional Linux permissions thuộc dạng Discretionary Access Control:

```text
owner
group
mode
ACL
```

SELinux bổ sung policy-based access controls.

Do đó access có thể cần đồng thời:

```text
Traditional permission allows
AND
SELinux policy allows
```

Nếu một trong hai deny:

```text
operation fails
```

---

# 2. SELinux Context

Objects như file/process có SELinux security context.

Bạn thường thấy dạng:

```text
user:role:type:level
```

Ví dụ conceptual:

```text
system_u:object_r:httpd_sys_content_t:s0
```

Đối với operational troubleshooting, `type` thường đặc biệt quan trọng.

Ví dụ một web server process chỉ được đọc những content type được policy cho phép.

---

# 3. SELinux modes

Syllabus yêu cầu:

```text
Enforcing
Permissive
Disabled
```

### Enforcing

Policy được áp dụng.

Unauthorized operation:

```text
deny
+
log
```

Đây là mode mặc định/recommended trong RHEL hiện đại. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_selinux/using_selinux?utm_source=chatgpt.com)

### Permissive

SELinux:

```text
logs denial
nhưng không enforce denial
```

Hữu ích để troubleshoot.

Nhưng không nên xem Permissive là “fix”.

### Disabled

SELinux infrastructure/policy enforcement bị disabled.

Red Hat strongly discourages việc disable SELinux để troubleshooting; permissive mode phù hợp hơn cho diagnostic scenarios. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_selinux/using_selinux?utm_source=chatgpt.com)

---

# 4. Vì sao `chmod 777` có thể không sửa lỗi?

Giả sử:

```text
-rwxrwxrwx
```

nhưng application vẫn:

```text
Permission denied
```

Có thể:

```text
SELinux context wrong
```

Điều này cho thấy:

```text
UNIX permission
≠
SELinux authorization
```

---

# 5. SELinux troubleshooting mindset

Không:

```text
Permission denied
→ setenforce 0
→ done
```

Đúng hơn:

```text
Permission denied
      │
      ▼
Traditional ownership/mode?
      │
      ▼
SELinux enforcing?
      │
      ▼
AVC denial?
      │
      ▼
Correct expected context?
      │
      ▼
Was file moved/copied incorrectly?
      │
      ▼
Is custom policy actually needed?
```

---

# 6. SELinux evidence

Useful evidence:

```bash
getenforce
sestatus
ls -Z
ps -eZ
```

và audit/AVC logs.

Current Red Hat guidance cũng sử dụng AVC search, ví dụ thông qua `ausearch`, khi troubleshoot denials. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/changing-selinux-states-and-modes?utm_source=chatgpt.com)

---

# XV. `firewalld`

`firewalld` quản lý host firewall policy trên RHEL.

---

# 1. Firewall để làm gì?

Firewall quyết định traffic nào được:

```text
allow
deny/drop/reject
```

dựa vào policy.

Một application có thể chạy bình thường nhưng remote users vẫn không truy cập được nếu firewall không permit.

---

# 2. `firewalld` architecture

Các concept quan trọng:

```text
zone
service
port
protocol
interface/source
runtime configuration
permanent configuration
```

Current RHEL documentation mô tả `firewalld` là dynamic firewall service và sử dụng zone, policy, service abstractions để quản lý traffic. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld?utm_source=chatgpt.com)

---

# 3. Zone

Zone đại diện trust level/context cho network traffic.

Ví dụ common names có thể gồm:

```text
public
internal
trusted
dmz
```

Không nên suy luận security chỉ dựa tên zone; phải kiểm tra actual configuration.

---

# 4. Service abstraction

`firewalld` service là predefined set liên quan ports/protocols.

Ví dụ:

```text
ssh
http
https
```

Thay vì nhớ:

```text
TCP/22
TCP/80
TCP/443
```

có thể permit named firewall service.

Named service thường dễ audit hơn raw port nếu đúng use case.

---

# 5. Runtime vs Permanent

Đây là distinction quan trọng nhất.

### Runtime

Current firewall state.

Có thể mất sau:

```text
reload/restart/reboot
```

### Permanent

Persistent configuration dùng để xây runtime state khi reload/start.

RHEL documentation xác nhận `firewalld` duy trì runtime và permanent configuration riêng biệt. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld?utm_source=chatgpt.com)

Operational mistake kinh điển:

```text
firewall rule works now
```

nhưng sau reboot:

```text
rule gone
```

vì chỉ sửa runtime.

---

# 6. Firewall troubleshooting

Application connectivity nên troubleshoot:

```text
Is service running?
        ↓
Is process listening?
        ↓
Which address?
        ↓
Which port/protocol?
        ↓
Does firewalld allow it?
        ↓
Is upstream network allowing it?
```

Đừng mở firewall trước khi biết application có thực sự listen không.

---

# XVI. Networking Fundamentals

Day 2 networking tập trung vào server-side operational troubleshooting.

Mental model:

```text
NIC
 ↓
Link
 ↓
IP address
 ↓
Subnet
 ↓
Routing
 ↓
DNS
 ↓
Firewall
 ↓
TCP/UDP port
 ↓
Application
```

---

# XVII. IP Addressing

IP address định danh network interface/logical interface ở IP layer.

Ví dụ:

```text
192.168.10.25/24
```

Trong đó:

```text
192.168.10.25 = host IP
/24           = prefix length
```

Với `/24`:

```text
network: 192.168.10.0/24
```

---

# 1. Interface

Một server có thể có:

```text
one NIC
multiple NICs
loopback
virtual interfaces
bond/team
VLAN interfaces
```

Do đó câu:

> “Server IP là gì?”

có thể không có duy nhất một câu trả lời.

---

# 2. Loopback

Thường:

```text
127.0.0.1
```

đại diện local host.

Application bind:

```text
127.0.0.1:8080
```

thì chỉ local system có thể truy cập trực tiếp.

Bind:

```text
0.0.0.0:8080
```

thường có nghĩa listen trên all IPv4 addresses phù hợp.

Đây là root cause rất phổ biến:

```text
curl localhost:8080 works
remote client fails
```

vì application bind loopback.

---

# XVIII. Routing

Routing quyết định packet đi đâu.

Routing table chứa conceptual fields như:

```text
destination
prefix
next hop/gateway
interface
metric
```

---

# 1. Connected route

Nếu destination nằm cùng subnet, host có thể gửi trực tiếp qua local interface.

---

# 2. Default route

Nếu không có more-specific route:

```text
default route
```

được sử dụng.

Ví dụ concept:

```text
default via 192.168.1.1 dev eth0
```

Nghĩa:

```text
Unknown remote destinations
→ send to 192.168.1.1
through eth0
```

---

# 3. Longest-prefix principle

Khi nhiều route match, routing lựa chọn route phù hợp nhất theo prefix specificity trước những tiêu chí khác.

Bạn chưa cần routing engineering sâu trong Day 2, nhưng cần hiểu:

```text
default route không phải luôn được dùng
```

nếu tồn tại more-specific route.

---

# 4. Routing troubleshooting

Nếu:

```text
ping 127.0.0.1 OK
local IP OK
remote subnet FAIL
```

cần xem:

```text
interface
address
route
gateway
upstream network
firewall
```

---

# XIX. DNS Resolution

DNS chuyển:

```text
hostname → IP
```

Ví dụ:

```text
app01.example.com
        ↓
192.0.2.10
```

---

# 1. DNS và connectivity là hai vấn đề khác nhau

Case:

```text
ping 8.8.8.8 works
ping example.com fails
```

đây là dấu hiệu strong candidate cho:

```text
DNS/name resolution issue
```

không nhất thiết network hoàn toàn down.

---

# 2. Name-resolution path

Linux có thể liên quan:

```text
/etc/nsswitch.conf
/etc/hosts
DNS resolver configuration
NetworkManager-generated settings
```

Không phải mọi hostname lookup đều chỉ hỏi DNS server.

---

# 3. Operational DNS troubleshooting

Tư duy:

```text
Can reach IP?
     │
     ├── no → network/routing/firewall
     │
     └── yes
          ↓
Can resolve hostname?
          │
          ├── no → resolver/DNS
          └── yes → move upward to application
```

---

# XX. TCP and UDP

## 1. TCP

TCP là connection-oriented transport protocol.

Concepts:

```text
connection
sequence
acknowledgement
retransmission
state
```

Application examples thường dùng TCP:

```text
SSH
HTTP
HTTPS
```

---

# 2. UDP

UDP là connectionless transport protocol.

Không có TCP-style connection establishment/reliability mechanism.

Examples có thể bao gồm:

```text
DNS queries
NTP
syslog in some deployments
```

tùy configuration.

---

# 3. Port

Port xác định logical endpoint ở transport layer.

Một connection TCP có thể hình dung bằng tuple:

```text
source IP
source port
destination IP
destination port
protocol
```

---

# 4. Listening socket

Server application thường tạo listening socket.

Ví dụ:

```text
0.0.0.0:22
```

nghĩa SSH service đang listen TCP 22 trên relevant IPv4 interfaces.

Nếu không có listener:

```text
firewall allow
```

cũng không tạo application service.

---

# XXI. `ip`

`ip` thuộc nhóm core networking commands.

Các nhóm phổ biến:

```bash
ip addr
ip link
ip route
```

---

## `ip addr`

Xem addresses.

Dùng để trả lời:

```text
Interface có IP chưa?
IP đúng subnet không?
```

---

## `ip link`

Xem link/interface state.

Ví dụ:

```text
UP
DOWN
```

---

## `ip route`

Xem routing table.

Dùng để trả lời:

```text
Default gateway là gì?
Destination này sẽ dùng route nào?
```

---

# XXII. `ping`

`ping` thường dùng ICMP Echo để kiểm tra một dạng IP reachability.

Nhưng:

```text
ping success
```

chỉ chứng minh một số điều về connectivity.

Không chứng minh:

```text
TCP/443 works
application works
DNS works
authentication works
```

Và:

```text
ping failure
```

không luôn đồng nghĩa host down.

ICMP có thể bị filtered.

---

# XXIII. `traceroute` / `tracepath`

Hai tool này hỗ trợ quan sát network path/hops.

Dùng khi:

```text
local network works
destination remote unreachable
```

và bạn muốn biết path dừng ở đâu.

Nhưng cần nhớ:

```text
hop không trả lời
≠
hop chắc chắn down
```

Network device có thể filter/deprioritize diagnostic traffic.

Tool này hỗ trợ **localize path problem**, không tự động chứng minh root cause.

---

# XXIV. `ss`

`ss` dùng để inspect sockets.

Rất quan trọng trong application troubleshooting.

Ví dụ conceptual:

```bash
ss -lntp
```

Các option thường hiểu:

```text
-l listen
-n numeric
-t TCP
-p process
```

Có thể trả lời:

```text
Port có listen không?
TCP hay UDP?
Bind address nào?
Process nào sở hữu socket?
```

---

# XXV. `curl`

`curl` là application-level diagnostic tool rất mạnh.

Nếu:

```bash
curl http://localhost:8080/
```

thành công, bạn đã test nhiều layer hơn `ping`.

Có thể xác minh:

```text
TCP connectivity
HTTP interaction
status codes
headers
TLS behavior nếu HTTPS
```

Ví dụ distinction:

```text
ping works
```

nhưng:

```text
curl fails
```

cho thấy vấn đề có thể nằm từ transport/application trở lên.

---

# XXVI. Network troubleshooting theo layer

Một workflow tốt:

```text
1. Interface exists?
2. Link up?
3. Correct IP/prefix?
4. Correct route?
5. Gateway reachable?
6. Destination reachable by IP?
7. DNS resolves?
8. Port listening locally?
9. Host firewall permits?
10. Remote port reachable?
11. Application protocol responds?
```

Không nên troubleshoot ngược bằng cách random restart network services.

---

# XXVII. Storage: Disk, Partition, Filesystem, Mount

Đây là bốn khái niệm bắt buộc phải phân biệt.

```text
Disk
 ↓
Partition / logical device
 ↓
Filesystem
 ↓
Mount
 ↓
Directory tree
```

---

# 1. Disk

Disk/block device là storage device mà OS nhìn thấy.

Ví dụ:

```text
/dev/sda
/dev/sdb
/dev/nvme0n1
```

Tên phụ thuộc loại device/platform.

---

# 2. Partition

Partition chia block device thành logical regions.

Ví dụ:

```text
/dev/sdb
 ├── /dev/sdb1
 └── /dev/sdb2
```

Không phải mọi storage đều bắt buộc partition; enterprise systems còn có LVM, multipath, SAN, virtual disks, v.v.

Nhưng Day 2 cần hiểu partition foundation.

---

# 3. Filesystem

Raw block device chưa tự động có directory/file structure.

Filesystem cung cấp abstraction:

```text
files
directories
metadata
ownership
permissions
timestamps
inodes
```

RHEL thường sử dụng filesystem như XFS trong nhiều deployments; ext4 cũng phổ biến trong Linux ecosystems.

---

# 4. Mount

Mount gắn filesystem vào Linux directory tree.

Ví dụ:

```text
filesystem /dev/sdb1
        ↓
mount at /data
```

Sau đó user thấy content qua:

```text
/data
```

---

# XXVIII. Mount Point

Mount point chỉ là directory nơi filesystem được attach.

Operational pitfall rất quan trọng:

Giả sử `/data` ban đầu chứa file trên root filesystem.

Sau đó một filesystem khác được mount lên `/data`.

Các file cũ không nhất thiết bị delete; chúng bị **hidden bởi mount**.

Nếu mount mất, “data khác” có thể xuất hiện.

Điều này đôi khi khiến operator tưởng:

```text
Files disappeared/reappeared mysteriously
```

trong khi thực tế mount state thay đổi.

---

# XXIX. Temporary vs Persistent Mount

Command:

```bash
mount ...
```

có thể tạo current runtime mount.

Nhưng sau reboot, nếu không có persistent configuration thích hợp:

```text
mount disappears
```

Đây là nơi `/etc/fstab` quan trọng.

---

# XXX. `/etc/fstab`

`/etc/fstab` mô tả persistent filesystem mounts.

Current RHEL documentation mô tả mỗi dòng `/etc/fstab` với các field:

```text
device
mount point
filesystem type
mount options
dump
fsck order
```

và `systemd-fstab-generator` chuyển entries thành systemd mount units. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/managing_file_systems/Red_Hat_Enterprise_Linux-10-Managing_file_systems-en-US.pdf?utm_source=chatgpt.com)

Ví dụ conceptual:

```text
UUID=<uuid> /data xfs defaults 0 0
```

---

# 1. Vì sao thường ưu tiên UUID?

Device naming có thể thay đổi tùy hardware/storage discovery.

Persistent identifier như UUID thường ổn định hơn raw path trong nhiều scenarios.

---

# 2. Rủi ro của `/etc/fstab`

Một typo có thể dẫn đến:

```text
mount failure
application failure
boot delay
recovery situation
```

Do đó enterprise workflow:

```text
backup file
edit carefully
validate
test mount
only then consider reboot
```

Không nên:

```text
edit fstab
reboot immediately
hope it works
```

---

# XXXI. Disk Capacity

Command concept phổ biến:

```bash
df -h
```

cho biết filesystem block capacity.

Các metric:

```text
Size
Used
Available
Use%
Mount point
```

---

# 1. Disk full symptoms

Khi filesystem đầy:

```text
application cannot write
logs stop
database errors
package install fails
temporary file creation fails
service crashes
```

Disk full có thể trở thành cascading incident.

---

# XXXII. Inodes

Filesystem không chỉ cần data blocks mà còn metadata objects/inodes.

Một inode đại diện metadata structure cho filesystem object.

Nếu workload tạo hàng triệu small files:

```text
data capacity còn
```

nhưng:

```text
inodes exhausted
```

thì vẫn có thể không tạo file mới.

Check:

```bash
df -i
```

Đây là lý do phải kiểm tra cả:

```text
df -h
df -i
```

---

# XXXIII. Storage troubleshooting

Một storage incident có thể đi theo flow:

```text
Application write failed
        ↓
Filesystem mounted?
        ↓
Correct mount?
        ↓
Read-only?
        ↓
Disk blocks full?
        ↓
Inodes full?
        ↓
Underlying device present?
        ↓
Kernel/I/O errors?
        ↓
Storage infrastructure dependency?
```

L1/L2 không nên tùy tiện dùng destructive filesystem repair nếu chưa xác định scope và authorization.

---

# XXXIV. Logging Architecture

Logging giúp trả lời:

```text
What happened?
When?
Which process?
On which host?
Under which user?
What failed?
What changed?
```

Không có log/evidence, troubleshooting thường trở thành phỏng đoán.

---

# XXXV. `systemd-journald`

`systemd-journald` thu thập messages từ nhiều sources như:

```text
kernel
boot process
services
stdout/stderr của daemons
syslog
```

RHEL 10 documentation tiếp tục mô tả `journald` và `rsyslog` là hai thành phần quan trọng trong system logging. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/risk_reduction_and_recovery_operations/Red_Hat_Enterprise_Linux-10-Risk_reduction_and_recovery_operations-en-US.pdf?utm_source=chatgpt.com)

---

# XXXVI. `journalctl`

`journalctl` là command để query journal.

Basic:

```bash
journalctl
```

Nhưng operationally không nên dump toàn bộ journal rồi đọc từ đầu.

Cần filter.

---

## 1. Theo unit

```bash
journalctl -u httpd
```

Dùng khi troubleshoot service.

---

## 2. Theo boot

```bash
journalctl -b
```

Current boot.

Rất hữu ích với startup issue.

---

## 3. Theo time

Concept:

```text
since
until
```

Nếu incident xảy ra 14:32, bạn nên tập trung time window đó thay vì log cả tháng.

---

## 4. Follow

```bash
journalctl -f
```

Quan sát new events khi reproduce issue.

Current Red Hat troubleshooting docs cũng sử dụng `journalctl -f` cho ongoing troubleshooting. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/risk_reduction_and_recovery_operations/index?utm_source=chatgpt.com)

---

# XXXVII. syslog và Rsyslog

`syslog` là logging concept/protocol truyền thống.

RHEL thường sử dụng `rsyslog` để process và route syslog messages.

Flow simplified:

```text
Application/kernel
      ↓
systemd-journald
      ↓
rsyslog
      ↓
/var/log/... or remote destination
```

RHEL documentation mô tả `rsyslog` có thể filter events, write local files hoặc forward tới remote systems. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/risk_reduction_and_recovery_operations/index?utm_source=chatgpt.com)

---

# 1. Các log files quen thuộc

Tùy version/configuration:

```text
/var/log/messages
/var/log/secure
/var/log/cron
/var/log/boot.log
```

Một số application có riêng:

```text
/var/log/httpd/
/var/log/<application>/
```

RHEL documentation hiện cũng liệt kê những locations này cho syslog-related logging. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/risk_reduction_and_recovery_operations/Red_Hat_Enterprise_Linux-10-Risk_reduction_and_recovery_operations-en-US.pdf?utm_source=chatgpt.com)

---

# XXXVIII. Log Severity

Syslog có severity levels từ rất nghiêm trọng đến debug.

Conceptually:

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

Bạn không cần chỉ nhớ tên; phải hiểu:

```text
severity giúp prioritize
```

nhưng severity do producer assign nên không phải luôn bằng business impact.

Một `warning` đôi khi ảnh hưởng business nhiều hơn một `error` khác không liên quan.

---

# XXXIX. Timestamp

Timestamp là cực kỳ quan trọng.

Trong incident:

```text
Server A: 10:00 UTC
Server B: 17:00 +07
```

nếu operator không normalize timezone có thể correlation sai hoàn toàn.

Day 10 sẽ đào sâu centralized logging, nhưng Day 2 phải hình thành habit:

```text
hostname
timestamp
timezone
command
result
```

khi lưu evidence.

---

# XL. Log Rotation

Logs tăng liên tục.

Nếu không rotate:

```text
log grows
   ↓
filesystem fills
   ↓
applications cannot write
   ↓
services fail
```

Log rotation thường giải quyết:

```text
rotate old file
compress
retain N versions/time
remove expired logs
```

---

# 1. `logrotate`

`logrotate` là utility quản lý log file rotation.

Một lưu ý version-sensitive: trên RHEL 9, Red Hat chuyển scheduling của `logrotate` từ traditional cron job sang `systemd` timer. Vì vậy không nên mặc định rằng mọi RHEL release đều chạy `logrotate` từ `/etc/cron.daily`. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/considerations_in_adopting_rhel_9/index?utm_source=chatgpt.com)

Đây là ví dụ tốt cho mindset:

> Học concept ổn định, nhưng verify implementation theo version.

---

# XLI. `grep`

`grep` dùng để search/filter text dựa trên pattern.

Ví dụ conceptual:

```bash
grep ERROR application.log
```

Dùng để giảm:

```text
100,000 log lines
```

xuống:

```text
relevant subset
```

---

## Useful concepts

```text
case-sensitive
case-insensitive
recursive
line numbers
inverse match
regex
```

Nhưng operational mistake là grep chỉ một từ và bỏ mất context.

Ví dụ:

```text
ERROR
```

có thể xuất hiện sau root-cause line 20 dòng trước.

Do đó cần đôi khi xem surrounding context.

---

# XLII. `awk`

`awk` đặc biệt mạnh cho field-oriented processing.

Ví dụ input:

```text
user1 10.1.1.1 SUCCESS
user2 10.1.1.2 FAILED
```

`awk` có thể lấy:

```text
field 1
field 2
conditional field 3
```

Mental model:

```text
line
 ↓
split into fields
 ↓
select/process fields
```

Rất hữu ích với:

```text
logs
command output
space-delimited reports
CSV-like simple data
```

---

# XLIII. `sed`

`sed` là stream editor.

Có thể:

```text
substitute
filter
transform
```

Ví dụ use cases:

```text
replace value in generated text
remove unwanted pattern
normalize strings
```

Trong operations, phải rất cẩn thận với in-place editing.

Read-only transform:

```bash
sed ...
```

an toàn hơn ngay lập tức sử dụng destructive/in-place edit.

---

# XLIV. Regular Expressions

Regular expression mô tả text pattern.

Ví dụ pattern categories:

```text
start/end of line
character classes
repetition
groups
alternation
```

Use cases:

```text
find IPv4-like strings
filter timestamps
identify ERROR/WARN
extract user IDs
identify filenames
```

---

# 1. Regex không phải magic truth

Regex chỉ tìm pattern.

Một regex match IP-like string không chứng minh string là valid routable IP.

Ví dụ:

```text
999.999.999.999
```

có thể match simplistic numeric-dot regex nhưng không phải valid IPv4.

Vì vậy:

```text
pattern matching
≠
semantic validation
```

---

# XLV. Bash Variables

Bash scripting của Day 2 tập trung operations automation.

---

# 1. Assignment

```bash
name=value
```

Không có spaces quanh `=` trong basic assignment.

---

# 2. Expansion

```bash
echo "$name"
```

Trong scripting, quoting variable expansion rất quan trọng.

Nên hình thành habit:

```bash
"$variable"
```

thay vì:

```bash
$variable
```

khi muốn tránh unwanted word splitting/globbing trong nhiều scenarios.

---

# 3. Command substitution

```bash
NOW=$(date)
```

Command output được gán vào variable.

Useful cho:

```text
timestamps
hostnames
disk usage
health data
```

---

# XLVI. Bash Control Flow

## `if`

Concept:

```bash
if condition
then
    action
fi
```

Operational use:

```text
if filesystem > threshold
→ warning
```

---

## `if/else`

```text
If service active
    report OK
else
    report FAIL
```

---

## `for`

Dùng khi lặp một tập object:

```text
services
filesystems
hosts
files
```

---

## `while`

Dùng khi loop phụ thuộc condition.

Cần cẩn thận infinite loop trong operational scripts.

---

## `case`

Hữu ích cho nhiều branch:

```text
start
stop
status
backup
restore
```

---

# XLVII. Exit Codes

Đây là một trong những concepts quan trọng nhất scripting.

Unix/Linux convention phổ biến:

```text
0     success
non-0 non-success/error/special result
```

Kiểm tra previous command:

```bash
echo $?
```

---

# 1. Vì sao exit code quan trọng?

Giả sử script:

```text
backup.sh
```

in:

```text
Backup finished
```

nhưng underlying `tar` failed.

Nếu script vẫn return:

```text
0
```

monitoring/cron/orchestrator có thể tưởng backup thành công.

Đây là dangerous false positive.

---

# 2. Application-specific exit codes

Không được giả định mọi non-zero có cùng ý nghĩa.

Mỗi command có thể định nghĩa exit code riêng.

Do đó:

```text
man page
documentation
```

là authoritative reference.

---

# XLVIII. Error Handling

Operational scripts cần thiết kế thất bại có kiểm soát.

Mental model:

```text
Run operation
    ↓
Check success?
 ┌──┴──┐
no    yes
│       │
log     continue
│
cleanup?
│
exit non-zero
```

---

# 1. Fail fast vs continue

Không phải lỗi nào cũng phải stop toàn script.

Ví dụ health checker:

```text
disk check failed
```

có thể vẫn tiếp tục:

```text
memory check
service check
network check
```

để tạo full report.

Nhưng backup pipeline:

```text
source inaccessible
```

thì có thể phải abort.

Design phụ thuộc purpose.

---

# 2. Error messages tốt

Không chỉ:

```text
ERROR
```

mà nên có:

```text
timestamp
operation
target
error condition
exit code
```

Ví dụ conceptual:

```text
2026-10-19T09:30:00+07:00 ERROR backup source=/data exit_code=2
```

---

# XLIX. Cron Scheduling

`cron` dùng để schedule recurring commands.

Basic crontab fields:

```text
minute
hour
day-of-month
month
day-of-week
command
```

Concept:

```text
* * * * *
│ │ │ │ │
│ │ │ │ └─ day of week
│ │ │ └─── month
│ │ └───── day of month
│ └─────── hour
└───────── minute
```

---

# 1. Cron environment trap

Một script:

```bash
./backup.sh
```

chạy manual được nhưng cron fail vì:

```text
different PATH
different working directory
different environment
no interactive shell
permissions
missing variables
```

Do đó production-style jobs nên:

```text
use absolute paths
set required variables explicitly
log stdout/stderr
```

---

# 2. Cron job configured ≠ job successful

Có entry:

```text
0 2 * * * /opt/scripts/backup.sh
```

chỉ chứng minh:

```text
schedule exists
```

không chứng minh:

```text
job executed
job succeeded
backup valid
```

Verification phải gồm:

```text
cron log
script log
output artifact
timestamp
exit condition
backup verification
```

---

# L. Compression

Compression giảm storage size.

Common conceptual tools:

```text
gzip
bzip2
xz
```

Nhưng:

```text
compression
```

khác với:

```text
archive
```

---

# 1. Archive

Archive gom nhiều files/directories vào một container.

`tar` historically mang nghĩa Tape Archive.

Ví dụ:

```text
file1
file2
directory/
   ↓
tar archive
```

---

# 2. Compression

Sau đó archive có thể compress:

```text
archive.tar
   ↓
gzip
   ↓
archive.tar.gz
```

Do đó:

```text
.tar
```

không tự động đồng nghĩa compressed.

---

# LI. `tar`

`tar` thường dùng cho simple filesystem backup/archive.

Concepts cần hiểu:

```text
create
extract
list
compression integration
file ownership
permissions
path behavior
```

---

# 1. Backup không phải chỉ “tạo tar thành công”

Bạn cần biết:

```text
What was included?
Were files readable?
Was archive complete?
Can it be listed?
Can it be extracted?
Does restored data match expectation?
```

---

# LII. `rsync`

`rsync` dùng để synchronize/copy files/directories hiệu quả.

Mental model:

```text
SOURCE
   │
   ▼
compare
   │
   ▼
DESTINATION
```

Nó có thể copy local hoặc remote tùy use case.

---

# 1. `rsync` direction rất quan trọng

Sai:

```text
destination → source
```

thay vì:

```text
source → destination
```

có thể gây hậu quả nghiêm trọng nếu dùng option destructive.

---

# 2. Trailing slash awareness

Một subtlety thường gặp:

```text
/source
```

và:

```text
/source/
```

có thể tạo destination layout khác nhau.

Do đó trước operations quan trọng, cần hiểu exact semantics.

---

# 3. `rsync` không tự động đồng nghĩa backup

Nếu bạn synchronize source deletion sang destination, destination có thể mất historical copy.

Backup thường cần:

```text
retention
version/history
integrity
recovery procedure
```

Trong khi simple synchronization có thể chỉ tạo mirror.

---

# LIII. Backup Fundamentals

Backup có bốn câu hỏi:

```text
What?
Where?
When?
Can restore?
```

Enterprise backup còn thêm:

```text
Who owns it?
How retained?
How protected?
How monitored?
```

---

# 1. Backup scope

Bạn phải biết backup gì:

```text
configuration
application data
scripts
logs
database
```

Không phải mọi file đều có giá trị như nhau.

---

# 2. Consistency

Nếu application đang ghi data, copying files không phải lúc nào cũng tạo application-consistent backup.

Day 2 chỉ yêu cầu basic restore procedure, nhưng cần biết limitation này.

Database/application-aware backup sẽ yêu cầu specialized method.

---

# 3. Security

Backup thường chứa production data.

Do đó backup cần:

```text
least privilege
appropriate filesystem permissions
secure transport
retention control
possibly encryption
```

Backup copy không được trở thành cách bypass access control.

---

# LIV. Basic Restore

Một backup chỉ đáng tin khi restore được.

Đây là principle quan trọng:

> **Backup success ≠ Recovery success.**

Restore workflow:

```text
Identify correct backup
        ↓
Confirm restore target
        ↓
Avoid overwriting production unnecessarily
        ↓
Restore to isolated/safe path
        ↓
Verify files
        ↓
Verify permissions/ownership
        ↓
Verify integrity/content
        ↓
Application-level validation if applicable
```

---

# 1. Restore to alternate location

Trong lab và nhiều validation scenario:

```text
restore to /restore-test
```

an toàn hơn overwrite source ngay.

Bạn có thể compare:

```text
source
vs
restored
```

trước khi final recovery.

---

# 2. Data integrity

Có thể verify bằng:

```text
file count
file size
timestamps where relevant
hash/checksum
application validation
```

Không chỉ:

```text
file exists
```

---

# LV. Health Checks

Health check là assessment tổng hợp.

Không có một Linux command duy nhất cho biết:

```text
Server is healthy = true
```

---

# 1. Layered health model

### Layer 1 — Host

```text
uptime
kernel/system state
```

### Layer 2 — CPU/Memory

```text
load
CPU
available memory
swap
```

### Layer 3 — Storage

```text
filesystems mounted
disk utilization
inode utilization
I/O errors
```

### Layer 4 — Network

```text
interfaces
IP
routes
DNS
```

### Layer 5 — Services

```text
critical systemd units
processes
listening ports
```

### Layer 6 — Security

```text
SELinux
firewalld
```

### Layer 7 — Logs

```text
recent errors
service failures
kernel warnings
```

### Layer 8 — Backup/Operational

```text
last backup
scheduled task state
```

---

# 2. Healthy infrastructure vs healthy application

Ví dụ:

```text
CPU OK
RAM OK
disk OK
service active
port listening
```

nhưng database dependency unavailable.

Application vẫn unhealthy.

Do đó server health và application/business health là different layers.

---

# LVI. Operational Troubleshooting

Đây là phần kết tinh toàn bộ Day 2.

---

# 1. Symptom, Cause, Root Cause

Ví dụ:

```text
User cannot access website
```

đây là symptom.

Bạn kiểm tra:

```text
HTTP connection refused
```

đây là more specific symptom.

Sau đó:

```text
httpd failed
```

là technical cause.

Log cho biết:

```text
cannot write because filesystem full
```

Root cause có thể là:

```text
log rotation failure → filesystem full → httpd failure
```

Nếu chỉ restart `httpd`, bạn chưa sửa root cause.

---

# 2. Troubleshooting phải hypothesis-driven

Sai:

```text
Restart service
Flush DNS
Disable SELinux
Open firewall
Reboot
```

tất cả cùng lúc.

Sau đó service hoạt động nhưng:

```text
không biết action nào sửa
không biết root cause
security posture bị thay đổi
evidence bị mất
```

Đúng:

```text
Symptom
   ↓
Hypothesis
   ↓
Test
   ↓
Evidence
   ↓
Accept/reject hypothesis
   ↓
Next test
```

---

# 3. Least invasive first

Thứ tự ưu tiên:

```text
Read-only inspection
        ↓
Low-risk diagnostic
        ↓
Configuration validation
        ↓
Controlled change
        ↓
Service restart
        ↓
Higher-impact recovery
```

Không nên:

```text
reboot first
```

trừ khi known procedure và authorized.

---

# 4. Scope determination

Hỏi:

```text
One user?
One application?
One server?
One subnet?
All servers?
All environments?
```

Scope giúp localize problem.

Ví dụ:

```text
Only one host fails DNS
```

khác hoàn toàn:

```text
All RHEL hosts fail DNS
```

---

# 5. Recent Changes

Một trong những câu quan trọng nhất:

```text
What changed?
```

Có thể là:

```text
package update
firewall rule
DNS record
certificate
filesystem mount
application config
SELinux change
network route
deployment
reboot
```

Nhưng correlation không luôn là causation; cần evidence.

---

# LVII. Integrated Troubleshooting Scenarios

## Scenario 1 — HTTP application unreachable

User báo:

```text
http://app01 không truy cập được
```

Tư duy đúng:

```text
1. DNS resolves?
2. IP reachable?
3. HTTP port reachable?
4. Process listening?
5. service active?
6. journal errors?
7. firewall allows?
8. SELinux denial?
9. filesystem full?
10. application dependency?
```

Dependency chain:

```text
DNS
 ↓
Routing
 ↓
Firewall
 ↓
Socket
 ↓
systemd
 ↓
Process
 ↓
Filesystem
 ↓
Application
```

---

## Scenario 2 — Service starts manually but fails after reboot

Candidates:

```text
service not enabled
dependency startup order
required mount unavailable
environment variable absent
network not available at expected time
configuration race
```

Đây không chỉ là “systemctl problem”.

---

## Scenario 3 — Script works manually, fails in cron

Candidates:

```text
PATH difference
working directory
permissions
environment variables
relative paths
credential availability
SELinux context
```

---

## Scenario 4 — Application says “No space left on device”

Đừng kết luận ngay:

```text
disk 100%
```

Phải kiểm tra:

```text
df -h
df -i
mount state
quota if relevant
```

Có thể:

```text
inodes exhausted
```

---

## Scenario 5 — Permission denied dù `chmod 777`

Candidates:

```text
SELinux
parent directory
ACL
filesystem mount options
application policy
```

Đây là lý do `chmod 777` không phải troubleshooting strategy.

---

# LVIII. Security Implications xuyên suốt Day 2

Security không phải riêng một section.

Nó tồn tại trong mọi operation.

Package:

```text
trusted repository
```

Process:

```text
run as least-privileged identity
```

systemd:

```text
service account
environment secrets
unit permissions
```

SELinux:

```text
MAC enforcement
```

Firewall:

```text
minimum required exposure
```

Storage:

```text
ownership/mount controls
```

Logs:

```text
sensitive data
tamper protection
```

Scripts:

```text
permissions
input validation
avoid secrets
```

Backup:

```text
data confidentiality
access control
```

Cron:

```text
who owns scheduled execution
```

---

# LIX. L1/L2 Escalation Boundary

Exact responsibility phụ thuộc organization, nhưng Day 2 cần hình thành boundary mindset.

## Thường phù hợp L1/read-only

```text
Collect system information
Check service status
Check logs
Check CPU/memory
Check disk/inodes
Check IP/routes
Check DNS
Check sockets
Check existing firewall configuration
Check SELinux status
Verify backup artifact
```

---

## Có thể thuộc L2 nếu runbook/authorization cho phép

```text
Restart approved service
Install approved package
Correct known service configuration
Add approved firewall rule
Correct known mount configuration
Restore controlled backup
Modify approved cron job
Resolve standard SELinux labeling issue
```

---

## Nên escalate khi

```text
Unknown production impact
Kernel panic
Filesystem corruption
Repeated I/O errors
Hardware/storage failure
Security compromise suspicion
Unknown SELinux policy requirement
Production network/routing ownership outside server team
Critical service repeated crash
Potential data loss
Unapproved software/package requirement
Destructive disk operation required
Recovery exceeds documented runbook
```

Điểm cốt lõi:

> **Biết cách thực hiện một command không đồng nghĩa bạn có authority để thực hiện nó.**

---

# LX. Evidence Collection

Mọi investigation tốt cần evidence.

Một evidence package tốt nên giúp người khác reproduce reasoning của bạn.

Thông tin nên có:

```text
Hostname
Environment
Timestamp + timezone
Incident/change/ticket reference
Observed symptom
Commands executed
Relevant output
Relevant logs
Configuration state
Action performed
Before/after state
Verification result
Outstanding risk
Escalation owner
```

---

# LXI. Before/After Evidence

Ví dụ:

```text
BEFORE
systemctl status app → failed
journal → configuration error

ACTION
approved config correction

AFTER
systemctl status app → active
ss → port listening
curl → HTTP 200
journal → no startup error
```

Đây có giá trị hơn screenshot:

```text
Service active
```

vì nó chứng minh full recovery chain.

---

# LXII. Day 2 Dependency Map

Toàn bộ lý thuyết Day 2 có thể nối với nhau như sau:

```text
                    ┌───────────────┐
                    │   Repository  │
                    └───────┬───────┘
                            │
                            ▼
                      Package/RPM
                            │
                            ▼
                       Executable
                            │
                            ▼
                    Environment
                            │
                            ▼
                        Process
                            │
                            ▼
                        systemd
                            │
              ┌─────────────┼──────────────┐
              │             │              │
              ▼             ▼              ▼
           SELinux       Network         Storage
              │             │              │
              │      ┌──────┼──────┐       │
              │      ▼      ▼      ▼       │
              │      IP   Route   DNS      │
              │             │              │
              ▼             ▼              ▼
          Permissions    Firewall       Filesystem
                            │              │
                            ▼              ▼
                          Ports          Mount
                            │              │
                            └──────┬───────┘
                                   ▼
                              Application
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
            Logs               Health Check         Backup
              │                    │                    │
       grep/awk/sed              Bash               tar/rsync
              │                    │                    │
            Regex              Exit code             Restore
                                   │
                                  cron
```

Nếu bạn hiểu diagram này, bạn đã bắt đầu nhìn server theo cách của Infrastructure Operations thay vì chỉ là Linux learner.

---

# LXIII. Operational Troubleshooting Master Flow

Đây là flow nên thuộc về tư duy, không phải thuộc lòng command:

```text
ALERT / USER REPORT
        │
        ▼
1. Confirm exact symptom
        │
        ▼
2. Determine scope
        │
        ▼
3. Establish timestamp
        │
        ▼
4. Check recent changes
        │
        ▼
5. Collect read-only baseline
        │
        ├── resource
        ├── service
        ├── process
        ├── network
        ├── storage
        ├── security
        └── logs
        │
        ▼
6. Form hypothesis
        │
        ▼
7. Run least-invasive test
        │
        ▼
8. Identify probable root cause
        │
        ▼
9. Check authorization
        │
        ├── not authorized → escalate
        │
        └── authorized
                │
                ▼
10. Apply controlled correction
                │
                ▼
11. Functional verification
                │
                ▼
12. Security/config verification
                │
                ▼
13. Collect before/after evidence
                │
                ▼
14. Document ticket/change
                │
                ▼
15. Close or escalate
```

---

# LXIV. Những sai lầm Day 2 cần loại bỏ

| Sai lầm                                | Vì sao sai                                | Tư duy đúng                                |
| -------------------------------------- | ----------------------------------------- | ------------------------------------------ |
| Service lỗi → restart ngay             | Mất evidence, chưa biết root cause        | Status → logs → dependency → correction    |
| Permission denied → `chmod 777`        | Có thể là SELinux/ACL và làm yếu security | Xác định đúng control đang deny            |
| Network lỗi → ping là đủ               | Ping không test application port          | IP → route → DNS → socket → firewall → app |
| Package missing → download RPM bất kỳ  | Supply-chain/security risk                | Approved repository                        |
| Disk còn space → storage OK            | Có thể hết inode                          | Check blocks + inodes                      |
| Cron entry tồn tại → job OK            | Job có thể fail                           | Verify execution + output                  |
| Backup file tồn tại → backup OK        | Restore có thể fail                       | Test restore                               |
| Service active → app healthy           | App có thể không respond                  | Service + port + protocol/function         |
| `firewall-cmd` rule works → persistent | Có thể runtime-only                       | Verify runtime/permanent                   |
| Disable SELinux để app chạy            | Bypass security control                   | Find context/policy root cause             |
| Reboot để “fix”                        | Mất state/evidence, impact rộng           | Least-invasive diagnosis                   |
| Non-zero exit code = cùng một lỗi      | Exit code semantics khác từng command     | Check documentation                        |
| CPU cao = incident                     | Có thể normal workload                    | Correlate duration/workload/impact         |
| Free RAM thấp = memory problem         | Linux uses cache                          | Look at available + pressure               |
| `rsync` = backup                       | Mirror có thể replicate deletion          | Design retention/recovery                  |
| Có log error = root cause              | Error có thể downstream                   | Correlate timeline/dependency              |

---

# LXV. Cheat Sheet Lý thuyết Day 2

### Software

```text
RPM           = package format/database layer
DNF           = dependency/repository-aware package manager
Repository    = package + metadata source
Subscription  = RHEL content/access awareness
```

### Runtime

```text
Program       = executable on disk
Process       = running instance
PID           = process ID
PPID          = parent PID
Signal        = asynchronous process control/event
```

### systemd

```text
.service      service unit
.target       grouped system state
start         run now
stop          stop now
restart       stop/start
reload        reload config if supported
enable        configure startup relationship
disable       remove enable relationship
mask          prevent normal starts
```

### Security

```text
DAC       ownership/mode/ACL
SELinux   MAC/policy/context
firewalld host firewall
```

### Networking

```text
ip addr     addresses
ip link     interfaces
ip route    routing
ping        ICMP reachability
tracepath   network path
ss          sockets
curl        application/protocol test
```

### Storage

```text
disk
 → partition/logical device
 → filesystem
 → mount
 → directory tree

/etc/fstab    persistent mounts
df -h         block capacity
df -i         inode capacity
```

### Logging

```text
journald
journalctl
syslog
rsyslog
/var/log
logrotate
```

### Text processing

```text
grep    pattern filtering
awk     field processing
sed     stream transformation
regex   pattern language
```

### Bash

```text
variables
if
case
for
while
exit codes
error handling
```

### Scheduling

```text
cron
minute hour DOM month DOW command
```

### Backup

```text
compression ≠ archive
tar       archive/basic backup
rsync     synchronization/copy
backup success ≠ restore success
```

---

# LXVI. Module Lý thuyết Day 2 — Completion Checklist

Sau khi học xong module này, bạn cần tự giải thích được toàn bộ các câu sau mà không chỉ trả lời bằng command:

- `dnf`, `yum` và `rpm` khác nhau về layer và responsibility như thế nào?
- Repository và RHEL subscription/content access liên quan package installation ra sao?
- Vì sao script chạy qua SSH shell được nhưng chạy qua `cron` hoặc `systemd` lại fail?
- PID, PPID và process state giúp troubleshoot điều gì?
- `SIGTERM` khác `SIGKILL` về operational safety như thế nào?
- Vì sao high CPU hoặc low free memory chưa đủ kết luận incident?
- `systemd` unit là gì?
- `start`, `enable`, `disable`, `mask`, `restart`, `reload` khác nhau ra sao?
- `multi-user.target`, `graphical.target`, `rescue.target`, `emergency.target` dùng để làm gì?
- Vì sao service `failed` chỉ là symptom?
- UNIX permissions và SELinux khác nhau như thế nào?
- Vì sao không nên disable SELinux để “fix” application?
- `firewalld` zone/service/port/runtime/permanent khác nhau thế nào?
- Làm thế nào phân biệt IP, routing, DNS, port và application failure?
- `ping`, `tracepath`, `ss`, `curl`, `ip` test những layer khác nhau như thế nào?
- Disk, partition, filesystem và mount là bốn object khác nhau ra sao?
- `/etc/fstab` có vai trò gì và vì sao sửa sai có thể ảnh hưởng boot?
- Disk block exhaustion và inode exhaustion khác nhau thế nào?
- `journald`, `journalctl`, syslog và `rsyslog` liên hệ ra sao?
- Vì sao log rotation có thể ảnh hưởng availability?
- `grep`, `awk`, `sed` và regex có vai trò gì trong log investigation?
- Bash variable, control flow và exit code giúp automation đáng tin cậy ra sao?
- Vì sao cron job configured không chứng minh job successful?
- Archive và compression khác nhau như thế nào?
- `tar` và `rsync` phục vụ hai nhu cầu khác nhau ra sao?
- Vì sao backup không được xem là thành công cho đến khi khả năng restore được xác minh?
- Một Linux health check đầy đủ cần kiểm tra những layer nào?
- Vì sao troubleshooting phải đi theo **symptom → evidence → hypothesis → root cause → authorized correction → verification**?
- Evidence nào cần lưu trước và sau một administrative change?
- Khi nào L1/L2 nên dừng lại và escalate?

Nếu trả lời chắc được những câu trên, bạn đã có nền tảng lý thuyết cần thiết để chuyển sang **Module Thực hành Day 2**, nơi toàn bộ các layer này sẽ được nối lại thành một RHEL operational scenario có package installation, `systemd` service recovery, networking, DNS, `firewalld`, SELinux, storage/mount, logging, Bash health-check, backup, cron, restore, failure injection, evidence và escalation record — đúng yêu cầu Assignment/Lab của syllabus. Infrastructure_Operations_14_Da…

Các chi tiết RHEL version-sensitive ở trên đã được đối chiếu với tài liệu Red Hat hiện hành về DNF, `systemd`, SELinux, `firewalld`, `/etc/fstab` và logging; syllabus không khóa vào một major version cụ thể, vì vậy trong môi trường thực tế luôn cần xác nhận chính xác RHEL release trước khi áp dụng configuration-changing procedure. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/managing_software_with_the_dnf_tool/index?utm_source=chatgpt.com)

Nếu bạn muốn, tôi có thể tiếp tục tra cứu **official Red Hat documentation** để lập một bộ “RHEL Day 2 Reference Map” theo từng topic, chỉ ra tài liệu chính thức nào nên dùng cho `DNF`, `systemd`, SELinux, `firewalld`, networking, storage và logging.
