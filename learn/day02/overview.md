Dựa đúng theo **Day 2 trong `Infrastructure_Operations_14_Day_Syllabus.md`**, Day 2 là phần chuyển từ Linux foundation của Day 1 sang **Linux/RHEL operational administration**: quản trị package/service/process, network, security controls, storage, logs, Bash automation, scheduling, backup/restore và troubleshooting theo cách vận hành enterprise. Toàn bộ phạm vi Concept/Lecture và Assignment/Lab của Day 2 nằm tại syllabus. Infrastructure_Operations_14_Da…

# Day 2 — Tổng quan

Day 1 chủ yếu giúp bạn có nền móng về Linux architecture, filesystem, CLI, users/groups, permissions, `sudo`, SSH và secure administrative access. Sang Day 2, trọng tâm thay đổi từ **“biết sử dụng Linux”** sang **“biết vận hành một Linux/RHEL server”**.

Có thể chia toàn bộ Day 2 thành **2 module lớn**:

| Module                   | Nội dung chính                                                                                                                                                                        | Kết quả mong đợi                                                                            |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Module 1 — Lý thuyết** | Linux/RHEL system administration, packages, processes, `systemd`, networking, SELinux, `firewalld`, storage, logging, Bash, cron, backup/restore, troubleshooting                     | Hiểu server hoạt động như thế nào và biết phương pháp kiểm tra, chẩn đoán, vận hành an toàn |
| **Module 2 — Thực hành** | Package install, service management/recovery, network/DNS/firewall/SELinux tests, disk/mount, logs, Bash health-check/backup, cron, restore, failure injection, evidence & escalation | Thực hiện được các tác vụ L1/L2 thực tế, verify kết quả, troubleshoot lỗi và tạo evidence   |

Day 2 vẫn thuộc module **Orientation & Linux Administration Foundations**, trong đó Days 1–2 bao phủ Linux/RHEL, SSH, `systemd`, networking, storage, logs, Bash và backup. Infrastructure_Operations_14_Da…

---

# Module 1 — Lý thuyết: Linux/RHEL Operational Administration

Module này cần giúp bạn hiểu **cơ chế**, chứ không chỉ nhớ command. Mục tiêu cuối cùng là khi một Linux/RHEL server có vấn đề, bạn có thể xác định vấn đề thuộc package, process, service, security, network, storage, logging hay application layer rồi điều tra có hệ thống.

## 1. Linux/RHEL system administration foundation

Bạn cần hiểu cách một RHEL/Linux server được vận hành hàng ngày: kiểm tra trạng thái, thay đổi configuration có kiểm soát, xác nhận service, resource, network, storage và log.

Điểm quan trọng là phân biệt:

| Loại hoạt động           | Ví dụ                                                              |
| ------------------------ | ------------------------------------------------------------------ |
| Read-only verification   | `systemctl status`, `ip addr`, `ss`, `df`, `journalctl`            |
| Configuration change     | cài package, sửa firewall rule, sửa `/etc/fstab`, enable service   |
| Service-impacting change | restart service, network change, mount/unmount                     |
| Potentially destructive  | partition/disk manipulation, xóa dữ liệu, filesystem operation sai |

Tư duy operational xuyên suốt Day 2 sẽ là:

**Observe → Verify → Diagnose → Change only if authorized → Verify again → Collect evidence → Rollback/Escalate if necessary.**

---

## 2. Package management: `dnf`, `yum`, `rpm`

Bạn phải hiểu ba lớp quan trọng của package administration.

`rpm` làm việc trực tiếp với RPM package database và package file.

`yum` là package manager truyền thống trên RHEL/CentOS.

`dnf` là package manager hiện đại được sử dụng trên các RHEL generation mới hơn.

Nội dung cần nắm gồm:

| Chủ đề                  | Cần hiểu                                           |
| ----------------------- | -------------------------------------------------- |
| Package installation    | Cài software/package được approve                  |
| Package removal         | Dependency và service impact                       |
| Package query           | Package có cài chưa, version nào                   |
| Repository              | Package được lấy từ đâu                            |
| Dependency              | Package phụ thuộc library/package nào              |
| Update                  | Update package khác với update toàn server thế nào |
| Verification            | Xác nhận package/file/version                      |
| Rollback considerations | Downgrade/reinstall/version lock tùy environment   |

### Repository và subscription awareness

Đặc biệt với RHEL, bạn phải hiểu sự liên hệ giữa:

**RHEL system → subscription → enabled repository → package metadata → package installation/update**

Không phải lỗi `dnf install` nào cũng có nghĩa package không tồn tại; nguyên nhân có thể là repository chưa enable, DNS lỗi, proxy/network lỗi, subscription issue hoặc metadata stale.

---

## 3. Environment variables

Environment variables là những giá trị được process và shell sử dụng để xác định runtime environment.

Các concept quan trọng gồm:

`PATH`, `HOME`, `USER`, `SHELL`, temporary variables, exported variables, shell initialization và process inheritance.

Bạn cần hiểu tại sao một command:

```bash
mycommand
```

có thể chạy với user này nhưng lại báo:

```text
command not found
```

với user khác.

Ngoài ra phải biết phân biệt environment của interactive shell với environment của `systemd`, `cron` hoặc script.

Đây là nguyên nhân rất phổ biến khiến:

> “Script chạy manual thì thành công nhưng chạy bằng cron/service lại fail.”

---

## 4. Processes và process states

Mỗi chương trình đang chạy trên Linux tồn tại dưới dạng process.

Bạn cần học:

- PID và PPID.
- Parent/child processes.
- Foreground/background processes.
- Process owner.
- Process state.
- CPU/memory/resource usage.
- Process termination.
- Zombie process awareness.
- Process hierarchy.

Các tool cơ bản sẽ xoay quanh:

```bash
ps
top
pgrep
pidof
kill
```

Quan trọng hơn command là phải hiểu:

**Process đang tồn tại không có nghĩa application đang healthy.**

Một process có thể chạy nhưng socket không listen, dependency fail, memory exhausted hoặc application đang deadlock.

---

## 5. Signals

Linux dùng signals để process nhận các event/control request.

Các signal operationally quan trọng gồm:

`SIGTERM`, `SIGKILL`, `SIGHUP`, `SIGINT`.

Bạn phải hiểu khác biệt giữa:

```bash
kill PID
```

và:

```bash
kill -9 PID
```

Trong enterprise operation, `SIGKILL` không nên là lựa chọn đầu tiên vì process không có cơ hội cleanup state/resource.

Troubleshooting phải đi theo hướng:

**graceful operation trước → forceful operation sau**.

---

## 6. Resource usage

Bạn cần biết cách đánh giá server có đang thiếu resource không.

Các nhóm resource chính:

| Resource | Ví dụ cần kiểm tra          |
| -------- | --------------------------- |
| CPU      | load, CPU utilization       |
| Memory   | used/free/available memory  |
| Swap     | swap pressure               |
| Disk     | space usage                 |
| inode    | inode exhaustion            |
| Process  | process consuming resources |
| Network  | socket/listener/connections |

Điểm quan trọng là tránh kết luận chỉ dựa trên một metric.

Ví dụ: `free memory` thấp chưa chắc Linux thiếu memory vì Linux sử dụng RAM cho cache.

---

# 7. `systemd` và service lifecycle

Đây là một trong những phần quan trọng nhất của Day 2.

Bạn cần hiểu:

**systemd → unit → service lifecycle**

Các trạng thái operational thường gặp:

```text
active
inactive
failed
activating
deactivating
```

Các operation chính:

```text
start
stop
restart
reload
enable
disable
status
```

Đặc biệt cần phân biệt:

```bash
systemctl start service
```

với:

```bash
systemctl enable service
```

`start` liên quan đến trạng thái hiện tại.

`enable` liên quan đến startup behavior ở boot.

Bạn cũng cần hiểu unit dependency, service startup failure và cách liên hệ `systemctl status` với `journalctl`.

---

# 8. Boot targets

`systemd` sử dụng targets để group các unit tương ứng với operating state.

Cần hiểu các target quan trọng như:

```text
multi-user.target
graphical.target
rescue.target
emergency.target
```

và khái niệm:

```text
default target
```

Phần này kết nối trực tiếp với service startup và recovery.

Bạn cần hiểu nếu server boot không bình thường thì L1/L2 có thể kiểm tra gì và khi nào phải escalate.

---

# 9. Service startup và recovery

Syllabus yêu cầu không chỉ quản lý service mà còn phải **recover deliberately failed service**. Infrastructure_Operations_14_Da…

Do đó lý thuyết phải chuẩn bị cho workflow:

```text
Service failed
    ↓
Check status
    ↓
Read logs
    ↓
Check configuration
    ↓
Check dependency
    ↓
Check permissions/SELinux
    ↓
Check port conflict
    ↓
Apply authorized correction
    ↓
Restart
    ↓
Verify
```

Bạn phải tránh tư duy:

> “Service lỗi → restart ngay.”

Restart có thể làm mất evidence quan trọng hoặc che khuất root cause.

---

# 10. SELinux

SELinux là Mandatory Access Control security mechanism rất quan trọng trên RHEL.

Bạn phải hiểu ba mode:

```text
Enforcing
Permissive
Disabled
```

và các concept:

```text
SELinux policy
context
type
label
denial
```

Một permission issue trên RHEL có thể xảy ra ngay cả khi traditional UNIX permission đã đúng:

```text
owner/group/mode OK
```

nhưng:

```text
SELinux context incorrect
```

Do đó troubleshooting phải phân biệt:

**DAC permissions** và **SELinux MAC controls**.

Một nguyên tắc quan trọng của Day 2 là không giải quyết vấn đề bằng cách mặc định disable SELinux.

---

# 11. `firewalld`

Bạn cần hiểu host-based firewall trên RHEL.

Các concept chính gồm:

```text
zones
services
ports
runtime configuration
permanent configuration
interfaces
rules
```

Phải hiểu mối liên hệ:

```text
Application listening
        ↓
Host firewall allows
        ↓
Network path reachable
        ↓
Client connects
```

Application listen đúng nhưng firewall block thì client vẫn không kết nối được.

Ngược lại, mở firewall nhưng application không listen thì cũng không giải quyết được vấn đề.

---

# 12. Linux networking

Day 2 bao phủ một lượng networking khá lớn, gồm:

**IP addressing → routing → DNS → TCP/UDP ports → connectivity diagnostics.** Infrastructure_Operations_14_Da…

### IP addressing

Bạn cần hiểu:

```text
IP address
prefix/subnet
interface
loopback
IPv4/IPv6 awareness
```

### Routing

Phải hiểu:

```text
destination
gateway
interface
default route
routing table
```

Linux dựa trên routing table để quyết định packet đi qua interface/gateway nào.

### DNS resolution

Phải phân biệt:

```text
Connectivity by IP
```

và:

```text
Connectivity by hostname
```

Nếu IP hoạt động nhưng hostname không hoạt động, vấn đề có khả năng nằm ở DNS/name resolution thay vì routing.

---

# 13. TCP/UDP ports

Bạn cần hiểu:

- Listening port.
- Local/remote address.
- TCP connection.
- UDP socket.
- Well-known/application ports.
- Bind address.
- Port conflicts.

Đây sẽ là nền tảng để dùng `ss`.

Một server có IP reachable chưa có nghĩa service reachable:

```text
Host reachable
≠
Port reachable
≠
Application healthy
```

---

# 14. Network diagnostic commands

Syllabus yêu cầu trực tiếp:

```text
ping
traceroute/tracepath
curl
ss
ip
```

Infrastructure_Operations_14_Da…

Vai trò sơ bộ:

| Command                    | Mục tiêu                                     |
| -------------------------- | -------------------------------------------- |
| `ip`                       | Kiểm tra interface, addresses, routes        |
| `ping`                     | Basic IP reachability                        |
| `tracepath` / `traceroute` | Quan sát network path                        |
| `ss`                       | Kiểm tra sockets/listening ports/connections |
| `curl`                     | Test application protocol như HTTP/HTTPS     |

Sau này trong phần chi tiết, chúng ta sẽ xây troubleshooting theo layer:

```text
Interface
→ IP
→ Route
→ DNS
→ Firewall
→ Listening port
→ Application protocol
```

---

# 15. Disks và partitions

Bạn cần hiểu block storage ở mức Linux administrator:

```text
disk
partition
filesystem
mount point
```

Đây là bốn object khác nhau.

Ví dụ conceptual flow:

```text
/dev/sdb
   ↓
partition
   ↓
/dev/sdb1
   ↓
filesystem
   ↓
mount
   ↓
/data
```

Không hiểu distinction này rất dễ thực hiện nhầm destructive operation.

---

# 16. Filesystems và mounts

Bạn cần hiểu filesystem được Linux sử dụng như thế nào và mount có ý nghĩa gì.

Các nội dung gồm:

- Filesystem awareness.
- Mount point.
- Temporary mount.
- Persistent mount.
- Mount verification.
- Device/filesystem identifiers.

Phần này kết nối với `/etc/fstab`.

---

# 17. `/etc/fstab`

`/etc/fstab` mô tả các filesystem/mount cần được hệ thống xử lý theo configuration.

Một lỗi trong `/etc/fstab` có thể làm:

- mount không thành công;
- application không thấy data;
- boot bị ảnh hưởng;
- filesystem mount vào sai location.

Vì vậy đây là configuration file phải sửa rất cẩn thận, backup trước và verify trước reboot.

---

# 18. Disk capacity và inode capacity

Day 2 yêu cầu cả:

**disk capacity** và **inode capacity**. Infrastructure_Operations_14_Da…

Hai tình trạng:

```text
Disk blocks exhausted
```

và:

```text
inodes exhausted
```

đều có thể dẫn đến lỗi kiểu:

```text
No space left on device
```

nhưng root cause khác nhau.

Đây là lỗi production rất thực tế, đặc biệt với application tạo hàng triệu small files.

---

# 19. `journalctl`

`journalctl` là công cụ đọc logs từ `systemd-journald`.

Bạn cần học cách lọc theo:

```text
service/unit
time
boot
priority
keyword
```

Nó đặc biệt quan trọng khi investigate service startup/failure.

---

# 20. syslog

Ngoài journal, Day 2 yêu cầu hiểu **syslog**.

Bạn cần nắm:

```text
system/application logging
facility/severity awareness
log files
log forwarding awareness
```

Mục tiêu là biết log nào nằm ở đâu và khi nào dùng journal so với traditional log files.

---

# 21. Log rotation

Logs không thể tăng vô hạn.

Bạn cần hiểu mục tiêu của log rotation:

```text
rotate
compress
retain
expire
```

Nếu log rotation lỗi, filesystem có thể full và kéo theo application/service failure.

Do đó log management có liên hệ trực tiếp với storage management.

---

# 22. `grep`, `awk`, `sed`

Đây là bộ công cụ text processing quan trọng cho administrator.

Bạn cần hiểu vai trò:

```text
grep → search/filter
awk  → field-based processing
sed  → stream transformation
```

Không chỉ để thao tác text file mà còn để xử lý:

```text
command output
logs
CSV-like data
configuration
script output
```

---

# 23. Regular expressions

Regular expressions giúp xác định pattern.

Ví dụ use case:

```text
lọc ERROR
lọc IP addresses
lọc timestamp
lọc failed login
lọc service message
```

Regex sẽ tiếp tục xuất hiện mạnh ở các Day automation sau này.

---

# 24. Bash variables

Bạn cần hiểu:

```bash
name=value
```

variable expansion:

```bash
"$name"
```

environment variables, positional parameters và command substitution.

Bash scripting ở Day 2 chưa hướng tới software engineering, mà hướng tới:

**operational automation nhỏ, dễ hiểu, dễ kiểm tra và dễ rollback.**

---

# 25. Bash control flow

Các construct cần hiểu ở mức operational:

```text
if/then/else
case
for
while
```

Ví dụ:

```text
Nếu filesystem usage > threshold
→ báo warning

Nếu service inactive
→ return failure
```

---

# 26. Exit codes

Đây là concept đặc biệt quan trọng.

Linux command thường trả về:

```text
0     = success
non-0 = failure/other condition
```

Shell cung cấp:

```bash
$?
```

để xem exit status.

Một script operational tốt phải có exit code rõ ràng để con người hoặc scheduler biết script thành công hay thất bại.

---

# 27. Error handling

Bash script không được giả định mọi command luôn thành công.

Bạn cần học tư duy:

```text
Execute
→ check status
→ record error
→ choose continue/stop
→ return meaningful exit code
```

Điều này đặc biệt quan trọng với backup.

Một script tạo file `.tar` chưa chắc backup thành công nếu intermediate command đã fail.

---

# 28. Cron scheduling

`cron` được dùng để schedule task định kỳ.

Bạn cần hiểu:

```text
minute
hour
day-of-month
month
day-of-week
command
```

Ngoài syntax còn phải hiểu environment khác interactive shell.

Một cron job phải được verify qua:

```text
execution
output/log
exit status
generated artifact
timestamp
```

Chỉ nhìn crontab không chứng minh job chạy thành công.

---

# 29. File compression

Phải hiểu compression khác với archive.

Ví dụ conceptually:

```text
tar = archive
gzip/xz = compression
```

Một archive có thể chứa nhiều file, sau đó được compress.

---

# 30. `tar` backup

`tar` là công cụ cơ bản dùng để tạo filesystem-level backup/archive.

Bạn cần hiểu:

```text
source
archive
compression
preservation
verification
extraction
```

Quan trọng nhất là **backup phải được restore-test**.

---

# 31. `rsync` backup

`rsync` phục vụ file synchronization/copy hiệu quả.

Bạn cần hiểu:

```text
source
destination
recursive copy
metadata
incremental behavior
remote/local awareness
```

Và đặc biệt phải cẩn thận với direction:

```text
source → destination
```

nhầm chiều có thể dẫn tới mất hoặc ghi đè dữ liệu.

---

# 32. Basic restore procedures

Backup không phải mục tiêu cuối cùng.

Mục tiêu thực sự là:

```text
Recover usable data
```

Do đó phải biết:

```text
identify backup
restore to safe location
verify file existence
verify content/integrity
compare source/restored data
document result
```

Syllabus nhấn mạnh phải **validate backup and restore results**, không chỉ tạo backup. Infrastructure_Operations_14_Da…

---

# 33. Health checks

Một Linux health check cơ bản cần xem tổng hợp nhiều layer:

```text
System uptime
CPU/load
Memory
Disk/inodes
Processes
Critical services
Network
Listening ports
Logs/errors
Backup status
```

Không có một command duy nhất chứng minh server healthy.

Health là trạng thái tổng hợp.

---

# 34. Operational troubleshooting

Đây là phần kết nối mọi kiến thức của Day 2.

Một workflow chuẩn sẽ gần như:

```text
1. Xác nhận symptom
2. Xác định scope
3. Kiểm tra recent change
4. Thu thập trạng thái hiện tại
5. Kiểm tra resource
6. Kiểm tra service/process
7. Kiểm tra logs
8. Kiểm tra network/security/storage dependency
9. Xác định probable root cause
10. Thực hiện authorized corrective action
11. Verify recovery
12. Thu thập evidence
13. Document hoặc escalate
```

Điểm quan trọng là phân biệt:

**symptom ≠ root cause.**

Ví dụ:

```text
Web page unavailable
```

chỉ là symptom.

Root cause có thể là:

```text
service down
port not listening
firewall blocked
SELinux denial
disk full
DNS failure
routing problem
configuration syntax error
```

---

# Module 2 — Thực hành: Linux/RHEL Operations Lab

Assignment/Lab chính thức của Day 2 yêu cầu bạn thực sự vận hành một RHEL system và tạo troubleshooting evidence, chứ không chỉ chạy command minh họa. Infrastructure_Operations_14_Da…

Module thực hành sẽ được chia thành các workstream sau.

| Lab workstream                    | Bạn sẽ thực hiện                                                                            | Kỹ năng được kiểm chứng                            |
| --------------------------------- | ------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| **1. Package Management Lab**     | Install approved packages, query package/version, verify repositories                       | `dnf/yum/rpm`, repository awareness                |
| **2. Process & Environment Lab**  | Inspect processes/resource usage, verify environment variables                              | Process, environment, resource troubleshooting     |
| **3. systemd Service Lab**        | Configure/manage service, start/stop/restart/enable, intentionally cause failure và recover | Service lifecycle, `systemctl`, `journalctl`       |
| **4. Network Lab**                | Verify IP, route, DNS, reachability, ports và HTTP connectivity                             | `ip`, `ping`, `tracepath/traceroute`, `ss`, `curl` |
| **5. Firewall Lab**               | Configure/test permitted traffic                                                            | `firewalld`, service/port exposure                 |
| **6. SELinux Lab**                | Investigate controlled SELinux-related access failure                                       | Modes, contexts, denial diagnosis                  |
| **7. Storage Lab**                | Inspect disk/partition/filesystem, mount storage, configure persistent mount                | disks, filesystems, mounts, `/etc/fstab`           |
| **8. Capacity Lab**               | Inspect disk space và inode usage                                                           | Storage capacity troubleshooting                   |
| **9. Logging Lab**                | Extract service/system logs và filter relevant events                                       | `journalctl`, syslog, `grep/awk/sed`, regex        |
| **10. Bash Health-Check Lab**     | Viết script kiểm tra resource/service/system state                                          | Bash variables, control flow, exit codes           |
| **11. Backup Script Lab**         | Tạo controlled backup bằng `tar`/`rsync`, xử lý lỗi                                         | Backup automation và error handling                |
| **12. Cron Lab**                  | Schedule health check/backup                                                                | Cron scheduling và execution verification          |
| **13. Restore Lab**               | Restore backup vào safe location và compare                                                 | Restore verification                               |
| **14. Failure Injection Lab**     | Simulate service, network và storage failures                                               | Structured troubleshooting                         |
| **15. Evidence & Escalation Lab** | Thu thập outputs/logs/timestamps và tạo troubleshooting record                              | Enterprise operational discipline                  |

---

# Chuỗi lab tổng thể của Day 2

Thay vì thực hiện 15 phần hoàn toàn rời rạc, module thực hành chi tiết sau này sẽ xây thành một scenario thống nhất:

```text
RHEL Lab Server
     │
     ├── Approved packages
     │
     ├── systemd service
     │      └── deliberate service failure
     │
     ├── Network
     │      ├── IP
     │      ├── Route
     │      ├── DNS
     │      ├── firewalld
     │      └── SELinux
     │
     ├── Storage
     │      ├── Disk
     │      ├── Filesystem
     │      ├── Mount
     │      ├── /etc/fstab
     │      └── Capacity/inodes
     │
     ├── Logging
     │      ├── journalctl
     │      ├── syslog
     │      └── filtering
     │
     ├── Bash automation
     │      ├── Health check
     │      ├── Exit-code handling
     │      └── Backup
     │
     ├── cron
     │
     ├── Restore verification
     │
     └── Troubleshooting evidence
```

---

# Failure Injection bắt buộc trong phần Thực hành

Syllabus yêu cầu rõ việc inject các lỗi về **service, network và storage**. Infrastructure_Operations_14_Da…

Khi học chi tiết Module Thực hành, chúng ta sẽ tạo controlled failures kiểu:

**Service failure:** service không start do configuration/dependency/permission issue.

**Network failure:** DNS, route, firewall hoặc port accessibility bị cấu hình sai có kiểm soát.

**Storage failure:** mount/configuration/capacity-related problem trong lab.

Mục đích không phải chỉ sửa lỗi, mà phải chứng minh được:

```text
Symptom
→ Evidence
→ Diagnosis
→ Root cause
→ Corrective action
→ Verification
→ Rollback status
→ Escalation decision
```

---

# Evidence phải tạo trong Day 2

Day 2 hướng tới operational readiness nên mỗi lab sau này sẽ yêu cầu evidence tương ứng.

Bạn sẽ phải quen với việc giữ các artifact như:

```text
Command output
systemctl status
journalctl extracts
Network configuration
Routing table
DNS test results
Listening socket information
Firewall configuration
SELinux evidence
Filesystem/mount information
Disk and inode usage
Bash script source
Script execution output
Exit code
cron configuration
Backup file
Backup timestamp
Restore test
Integrity comparison
Failure-injection evidence
Troubleshooting record
Escalation record
```

Không nên chỉ chụp screenshot “đã chạy thành công”. Evidence tốt phải giúp một người khác trả lời:

> **Bạn đã làm gì, khi nào, trên system nào, kết quả ra sao và dựa vào đâu để kết luận?**

---

# Quan hệ giữa Module Lý thuyết và Module Thực hành

Mỗi kiến thức lý thuyết đều có một counterpart thực hành:

```text
dnf/yum/rpm
        ↓
Install + verify package

processes/signals
        ↓
Inspect running processes

systemd
        ↓
Manage + recover service

IP/routes/DNS
        ↓
Connectivity troubleshooting

firewalld
        ↓
Allowed/blocked connectivity

SELinux
        ↓
Access denial investigation

disk/filesystem/fstab
        ↓
Mount + persistent configuration

journalctl/syslog
        ↓
Incident evidence extraction

grep/awk/sed/regex
        ↓
Filter diagnostics

Bash
        ↓
Health-check/backup script

exit codes
        ↓
Detect script failures

cron
        ↓
Scheduled execution

tar/rsync
        ↓
Backup

restore
        ↓
Recovery verification

troubleshooting
        ↓
Failure injection + escalation record
```

---

# Kết quả bạn phải đạt sau Day 2

Sau khi hoàn thành **cả Module Lý thuyết và Module Thực hành**, bạn nên có khả năng nhìn một RHEL server theo nhiều layer thay vì xử lý từng command rời rạc:

```text
Linux/RHEL Server
│
├── Software
│   └── package/repository
│
├── Runtime
│   ├── process
│   ├── resource
│   └── systemd service
│
├── Security
│   ├── SELinux
│   └── firewalld
│
├── Network
│   ├── interface/IP
│   ├── route
│   ├── DNS
│   └── TCP/UDP
│
├── Storage
│   ├── disk
│   ├── partition
│   ├── filesystem
│   ├── mount
│   └── capacity/inodes
│
├── Observability
│   ├── journal
│   ├── syslog
│   └── text filtering
│
├── Automation
│   ├── Bash
│   ├── exit codes
│   └── cron
│
└── Resilience
    ├── compression/archive
    ├── backup
    ├── restore
    └── troubleshooting
```

Đây chính là phạm vi Day 2 mà syllabus yêu cầu: **Linux/RHEL system administration + networking + security + storage + logging + Bash automation + backup/restore + health checks + operational troubleshooting**, sau đó chứng minh năng lực bằng hands-on lab và troubleshooting evidence. Infrastructure_Operations_14_Da…

Khi bạn nói **“Module Lý thuyết”**, tôi sẽ bắt đầu giảng toàn bộ phần lý thuyết Day 2 theo chiều **foundation → operational workflow → verification → troubleshooting → evidence → escalation**, không bỏ sót topic nào trong syllabus. Khi bạn nói **“Module Thực hành”**, tôi sẽ xây trọn bộ lab Day 2 với **Objective, Environment, Safety Notes, Steps, Commands, Expected Result, Verification, Evidence, Failure Injection, Troubleshooting checkpoints và Cleanup/Rollback**.
