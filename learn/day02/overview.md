Dưới đây là **bản đồ kiến thức Day 2** được chia thành đúng **2 module: Lý thuyết và Thực hành**. Tôi sẽ chưa đi sâu đến mức giải thích từng command hay từng lab step; mục tiêu của phần này là để bạn nhìn thấy **toàn bộ phạm vi Day 2**, dependency giữa các chủ đề, những năng lực vận hành cần đạt và các deliverable/evidence cần tạo. Nội dung bám theo **Day 2 trong `Infrastructure_Operations_14_Day_Syllabus.md`**, bao gồm cả phần **Additive skill coverage — RHEL maintenance and server log extraction**. Infrastructure_Operations_14_Da…

# Day 2 — Tổng quan

Day 2 tiếp tục nền tảng Linux/RHEL của Day 1 nhưng chuyển trọng tâm từ **“biết sử dụng Linux và quản lý access/files”** sang **“vận hành một Linux/RHEL server như một Infrastructure Operations engineer”**.

Nếu Day 1 tập trung vào CLI, filesystem, users/groups, permissions, `sudo`, SSH và operational safety, thì Day 2 yêu cầu bạn biết quản trị **software → processes → services → boot → security controls → network → storage → logs → scripting → scheduling → backup/restore → troubleshooting → maintenance evidence**.

Theo syllabus, Day 2 phải bao phủ đầy đủ: package management, `systemd`, processes/signals, SELinux, `firewalld`, networking, disks/filesystems/mounts, logging, text processing, Bash scripting, `cron`, compression/backup/restore, health checks và operational troubleshooting. Lab còn yêu cầu tạo lỗi có kiểm soát, khôi phục service/network/storage và xây dựng troubleshooting/escalation record. Infrastructure_Operations_14_Da…

Ngoài ra, Day 2 có phần mở rộng bắt buộc về **RHEL maintenance** và **audit-quality server log extraction**: pre/post-maintenance checks, dependency/reboot assessment, bounded UTC log extraction, rotated logs, exact commands/filters, hash/count evidence và nhận biết các khoảng audit không tồn tại hoặc không đầy đủ. Infrastructure_Operations_14_Da…

---

# MODULE 1 — LÝ THUYẾT: Linux/RHEL System Administration & Operational Troubleshooting

Module Lý thuyết có thể chia thành **12 khối kiến thức**. Các khối này liên kết với nhau thành workflow vận hành một RHEL server thực tế.

## 1. RHEL package management và software lifecycle

Bạn cần hiểu cách software được cài đặt và quản lý trên RHEL bằng:

- `dnf`
- `yum`
- `rpm`
- package metadata
- package dependencies
- repositories
- repository configuration
- package version
- install / update / remove / query
- package verification
- subscription/repository awareness

Không chỉ học cách chạy `dnf install`, mà cần hiểu dependency chain:

**RHEL host → repository → metadata → package → dependencies → installed files → service/configuration → verification.**

Bạn cũng cần phân biệt:

- package installed hay chưa;
- package available từ repository nào;
- installed version và candidate version;
- package được cài trực tiếp bằng `rpm` hay thông qua `dnf`;
- repository enabled/disabled;
- vấn đề package failure do network, DNS, subscription, repository hay dependency.

### Operational perspective

Trong enterprise, cài package thường phải đi qua:

**ticket → approval → baseline → package/repository check → change → verification → evidence → rollback/escalation.**

Phần additive của syllabus còn yêu cầu hiểu:

- release/subscription prerequisites;
- patch prerequisites;
- dependency assessment;
- reboot assessment;
- recovery limitations.

---

## 2. Environment variables

Bạn cần hiểu Linux process nhận environment như thế nào và các variable thường gặp:

- `PATH`
- `HOME`
- `USER`
- `SHELL`
- `LANG`
- `PWD`
- custom environment variables

Phải phân biệt:

- shell variable;
- exported environment variable;
- process environment;
- session-level configuration;
- persistent configuration.

Mục tiêu operational là chẩn đoán các tình huống kiểu:

> Command hoạt động khi chạy interactive shell nhưng service chạy bằng `systemd` lại báo `command not found`.

Hoặc:

> Script chạy bằng tay thành công nhưng chạy bằng `cron` thất bại vì `PATH` khác.

---

# 3. Linux processes, process states và signals

Bạn cần nắm:

- process;
- PID;
- PPID;
- foreground/background;
- process ownership;
- process tree;
- process states;
- CPU/memory usage;
- parent-child relationship;
- zombie process awareness.

Các signal quan trọng cần hiểu về mặt cơ chế:

- `SIGTERM`
- `SIGKILL`
- `SIGHUP`
- `SIGINT`

Điểm operational quan trọng là:

**Không dùng `kill -9` như phản xạ đầu tiên.**

Troubleshooting đúng thường là:

**observe → identify process → understand ownership/service dependency → graceful stop → verify → stronger intervention nếu được phép.**

---

# 4. `systemd` và service lifecycle

Đây là một trong các chủ đề trọng tâm nhất của Day 2.

Bạn cần hiểu:

- `systemd`;
- unit;
- service unit;
- unit file;
- active/inactive;
- enabled/disabled;
- failed state;
- dependency;
- startup order;
- restart behavior;
- service lifecycle.

Workflow cơ bản:

**start → stop → restart → reload → status → enable → disable → failure investigation → recovery.**

Bạn cần hiểu các command quan trọng như:

- `systemctl status`
- `systemctl start`
- `systemctl stop`
- `systemctl restart`
- `systemctl reload`
- `systemctl enable`
- `systemctl disable`
- `systemctl is-active`
- `systemctl is-enabled`
- `systemctl list-units`
- `systemctl list-unit-files`

Quan trọng hơn command là phải hiểu mối quan hệ:

**service failure → systemd state → exit status → journal → process → configuration → port → dependency → filesystem/permission → SELinux/firewall/network.**

---

# 5. Boot process, boot targets và service recovery

Bạn cần hiểu RHEL khởi động service bằng `systemd` như thế nào.

Các concept:

- boot target;
- `multi-user.target`;
- `graphical.target`;
- default target;
- service startup;
- enabled service;
- failed boot service;
- recovery awareness.

Mục đích không phải trở thành Linux boot specialist mà là đủ để trả lời:

> Service này có được thiết kế để tự chạy sau reboot không?

> Server reboot xong application không lên thì kiểm tra từ đâu?

> `systemctl start` thành công nhưng reboot xong service lại không chạy vì sao?

---

# 6. SELinux

Day 2 yêu cầu hiểu cả:

- SELinux modes;
- SELinux contexts;
- SELinux-related access problems.

Các mode phải biết:

- `Enforcing`
- `Permissive`
- `Disabled`

Nhưng điều quan trọng hơn là tư duy operational:

**SELinux không phải thứ cần disable mỗi khi application bị Permission denied.**

Bạn cần hiểu:

- security context;
- labels;
- policy;
- enforcement;
- denial;
- file/context mismatch.

Phải phân biệt được:

**Linux file permissions problem**

với

**SELinux policy/context problem**.

Một troubleshooting flow tốt sẽ là:

**permissions → ownership → process identity → filesystem/mount → SELinux → application logs**, thay vì ngay lập tức disable SELinux.

---

# 7. `firewalld` và host firewall

Bạn sẽ học:

- firewall concept;
- zones;
- services;
- ports;
- runtime configuration;
- persistent configuration;
- allowed/blocked traffic.

Quan trọng nhất là hiểu flow:

**application process → listening socket → host firewall → network route → remote client.**

Ví dụ application không truy cập được qua TCP/443 có thể do:

- service chưa chạy;
- service chỉ listen localhost;
- wrong port;
- `firewalld` block;
- route problem;
- remote firewall;
- DNS problem.

Không được mặc định rằng “network lỗi” chỉ vì `curl` không connect được.

---

# 8. Linux networking fundamentals

Day 2 yêu cầu một nền tảng network operational khá rộng:

### IP addressing

- IPv4 addressing;
- interface;
- subnet;
- default gateway.

### Routing

- routing table;
- default route;
- destination route;
- interface selection.

### DNS

- name resolution;
- resolver configuration;
- hostname → IP mapping;
- phân biệt DNS failure với connectivity failure.

### TCP/UDP

- TCP;
- UDP;
- ports;
- listening ports;
- established connections.

### Diagnostic tools

Bạn cần hiểu công dụng và giới hạn của:

- `ping`
- `traceroute`
- `tracepath`
- `curl`
- `ss`
- `ip`

Một flow troubleshooting quan trọng sẽ là:

**interface → IP → route → gateway → DNS → destination IP → destination port → application protocol.**

Đây là tư duy mà bạn sẽ dùng lại nhiều ở AWS, Cisco, FortiGate và BIG-IP trong các Day sau.

---

# 9. Linux storage: disk, partition, filesystem và mount

Đây cũng là phần lớn của Day 2.

Bạn cần hiểu hierarchy:

**physical/virtual disk → partition → filesystem → mount point → application data.**

Các concept:

- disks;
- partitions;
- filesystems;
- mount;
- mount point;
- persistent mount;
- `/etc/fstab`;
- disk capacity;
- inode capacity.

Một điểm rất quan trọng:

> “Disk full” không phải lúc nào cũng là hết GB.

Có ít nhất hai nhóm capacity problem:

- hết block/data capacity;
- hết inode.

Do đó bạn phải biết đọc và phân biệt:

- filesystem usage;
- free space;
- inode usage;
- mount state.

### `/etc/fstab`

Bạn cần hiểu đây là một phần có risk vì cấu hình sai có thể ảnh hưởng boot.

Operational workflow phải gồm:

**backup → edit carefully → validate → test mount → verify → reboot-awareness/rollback.**

---

# 10. Linux logging và log extraction

Bạn sẽ học hai hệ thống chính:

### `journalctl`

Dùng với `systemd-journald`.

Bạn cần biết concept:

- journal;
- unit filtering;
- time filtering;
- boot filtering;
- priority;
- bounded extraction.

### syslog/log files

Hiểu:

- syslog architecture awareness;
- log files;
- authentication logs;
- service logs;
- rotated logs.

### Log rotation

Bạn cần biết:

- logs không thể tăng mãi;
- active log;
- rotated log;
- compressed historical log;
- retention limitations.

### Text-processing tools

Day 2 yêu cầu:

- `grep`
- `awk`
- `sed`
- regular expressions.

Không phải chỉ học syntax mà phải dùng để:

**search → filter → extract fields → correlate events → produce evidence.**

---

# 11. Bash scripting và automation foundation

Bạn sẽ bắt đầu scripting phục vụ Operations.

Các concept bắt buộc:

- Bash variables;
- quoting;
- command substitution awareness;
- conditional control flow;
- loops;
- command exit codes;
- error handling.

Một script Operations không được chỉ “chạy”.

Nó phải trả lời được:

- thành công hay thất bại;
- lỗi ở bước nào;
- exit code gì;
- output nào được lưu;
- có gây misleading success không.

Ví dụ logic:

```text
check service
      ↓
service healthy?
 ┌────┴────┐
yes       no
 ↓         ↓
check      log error
disk       set non-zero exit code
 ↓
...
```

---

# 12. `cron`, compression, backup và restore

## Scheduling

Bạn cần hiểu `cron` dùng để chạy task định kỳ.

Các điểm cần học:

- crontab syntax;
- execution schedule;
- cron environment;
- absolute paths;
- stdout/stderr;
- permission/user context;
- verification rằng job thực sự đã chạy.

Không được coi:

> “Tôi thêm entry vào crontab”

là bằng chứng rằng task hoạt động.

Phải có:

**scheduled → executed → output produced → result verified.**

---

## Compression

Các chủ đề:

- archive;
- compression;
- file packaging;
- common compression workflows.

---

## `tar` và `rsync` backup

Bạn sẽ học backup cơ bản bằng:

- `tar`
- `rsync`

Phải hiểu:

- source;
- destination;
- archive;
- synchronization;
- preservation of metadata;
- overwrite risk;
- backup validation.

---

## Restore

Điểm syllabus nhấn mạnh là **backup phải có restore verification**.

Workflow:

**backup → inspect → restore to controlled destination → compare → verify content → record evidence.**

Không được kết luận backup tốt chỉ vì command trả về exit code `0`.

---

# 13. Health checks và operational troubleshooting

Đây là phần tích hợp tất cả các topic phía trên.

Một Linux health check phải có khả năng kiểm tra ít nhất:

- uptime;
- CPU/load;
- memory;
- disk;
- inode;
- processes;
- services;
- network;
- ports;
- filesystem/mount;
- logs;
- backup status.

Bạn sẽ học troubleshooting theo hướng **least-risk-first**:

**Observe → Verify → Isolate → Correlate → Correct within scope → Re-verify → Document → Escalate if necessary.**

Không nên:

**thấy lỗi → restart ngay → mất evidence → không biết root cause.**

---

# 14. RHEL maintenance workflow — phần additive bắt buộc

Day 2 không dừng ở administration thường ngày mà còn yêu cầu tư duy maintenance.

Bạn cần hiểu một maintenance workflow:

```text
Approved maintenance ticket
        ↓
Scope / risk / dependency review
        ↓
Release + subscription check
        ↓
Package/service baseline
        ↓
Reboot assessment
        ↓
Pre-checks
        ↓
Approved change
        ↓
Post-checks
        ↓
Service/application validation
        ↓
Evidence
        ↓
Close or Escalate
```

Bạn cũng phải hiểu **recovery limitations**.

Ví dụ:

> Package upgrade không phải lúc nào cũng có thể rollback hoàn hảo chỉ bằng `dnf downgrade`.

Do đó trước maintenance phải biết:

- dependency;
- configuration impact;
- reboot impact;
- service dependency;
- available recovery mechanism;
- stop conditions;
- escalation owner.

---

# 15. Audit-quality server log extraction

Đây là phần rất quan trọng đối với Infrastructure Operations enterprise.

Bạn phải phân biệt các nguồn như:

- system journal;
- service journal;
- authentication logs;
- `auditd` records nếu auditing được cấu hình;
- rotated logs.

Điểm cực kỳ quan trọng:

> Không được giả định mọi hành động đều có log.

Phải biết ghi nhận:

- log retention;
- rotation;
- missing intervals;
- permission limitations;
- auditing enabled/disabled;
- extraction errors.

Syllabus yêu cầu log extraction có **bounded UTC window**.

Ví dụ về mặt concept:

```text
Host: rhel-lab01
Window: 2026-10-20 10:00:00 UTC
        →
        2026-10-20 10:15:00 UTC
Service: sshd
Ticket: CHGxxxx
```

Chứ không chỉ:

```bash
journalctl | grep error
```

rồi coi đó là audit evidence.

Bạn phải lưu:

- exact command;
- exact filter;
- hostname;
- timestamp/timezone;
- record count;
- collection errors;
- file hash;
- readable filtered copy;
- ideally native/lossless export khi phù hợp.

---

# MODULE 2 — THỰC HÀNH: RHEL Operations Lab, Failure Recovery & Evidence

Module Thực hành sẽ ghép toàn bộ kiến thức trên vào một chuỗi lab có kiểm soát.

Theo syllabus, bạn phải thực hiện đầy đủ các nhóm lab dưới đây. Infrastructure_Operations_14_Da…

---

## Lab 1 — Package management

Bạn sẽ:

- kiểm tra RHEL release;
- kiểm tra repository/subscription awareness;
- tìm approved package;
- cài package bằng `dnf`;
- verify package bằng `dnf`/`rpm`;
- kiểm tra version;
- xác định file/package relationship;
- ghi lại before/after state.

### Evidence

Ví dụ:

- hostname;
- OS version;
- repository state;
- package version before/after;
- exact commands;
- timestamps.

---

# Lab 2 — Process inspection

Bạn sẽ:

- tìm running processes;
- xác định PID/PPID;
- xác định process owner;
- quan sát CPU/memory;
- xem process state;
- gửi signal an toàn;
- verify process termination/recovery.

Mục tiêu là hiểu process ở mức operational, không chỉ biết chạy `ps`.

---

# Lab 3 — Environment-variable verification

Bạn sẽ:

- kiểm tra environment;
- tạo variable;
- export variable;
- verify variable ở shell/process;
- quan sát khác biệt giữa interactive execution và automation context.

Phần này sẽ chuẩn bị cho `systemd`, Bash và `cron`.

---

# Lab 4 — `systemd` service management

Bạn sẽ:

- inspect service;
- start/stop/restart;
- check service status;
- kiểm tra enabled state;
- inspect unit information;
- inspect logs bằng `journalctl`.

Sau đó deliberately tạo một service failure an toàn.

---

# Lab 5 — Failed-service recovery

Bạn sẽ được inject lỗi kiểu:

```text
Service failed
```

và phải điều tra theo workflow:

```text
systemctl status
        ↓
journalctl
        ↓
process/configuration
        ↓
permissions/files
        ↓
port/dependencies
        ↓
SELinux
        ↓
fix
        ↓
restart
        ↓
verify
```

Điểm đánh giá không chỉ là “service chạy lại”, mà là bạn có chỉ ra được:

- symptom;
- root cause;
- evidence;
- corrective action;
- verification;
- rollback;
- escalation boundary.

---

# Lab 6 — Network configuration và connectivity troubleshooting

Bạn sẽ kiểm tra:

- network interfaces;
- assigned IP;
- routes;
- default gateway;
- DNS resolution;
- network reachability;
- service port;
- local listening sockets.

Các tool chính:

```bash
ip
ping
tracepath
traceroute
ss
curl
```

Bạn phải học cách phân biệt:

```text
DNS failure
vs
routing failure
vs
firewall failure
vs
service/port failure
vs
application-layer failure
```

---

# Lab 7 — `firewalld`

Bạn sẽ:

- inspect firewall state;
- inspect zones;
- inspect allowed services/ports;
- thực hiện một approved lab rule;
- test allowed traffic;
- test denied traffic;
- verify runtime/permanent behavior;
- rollback rule.

Mọi thay đổi đều trong **lab/non-production environment**.

---

# Lab 8 — SELinux troubleshooting

Bạn sẽ:

- inspect SELinux mode;
- inspect security context;
- tạo một controlled access failure;
- nhận diện symptom;
- phân biệt Unix permissions với SELinux denial;
- sửa theo cách phù hợp;
- verify.

Mục tiêu quan trọng:

> Không sử dụng `setenforce 0` hoặc disable SELinux như “solution mặc định”.

---

# Lab 9 — Disk, filesystem và mount

Bạn sẽ:

- identify disk;
- identify partition;
- identify filesystem;
- inspect mount points;
- mount filesystem;
- verify mounted storage;
- configure persistent mount bằng `/etc/fstab`;
- validate configuration.

Mọi thao tác destructive nếu có đều chỉ được dùng trên **dedicated lab disk**.

---

# Lab 10 — Disk và inode capacity

Bạn sẽ kiểm tra:

- disk consumption;
- filesystem capacity;
- inode usage.

Sau đó mô phỏng một capacity issue an toàn.

Bạn phải biết trả lời:

> Filesystem báo “No space left on device” nhưng `df` vẫn còn dung lượng — vì sao?

Một khả năng quan trọng chính là **inode exhaustion**.

---

# Lab 11 — Log extraction và text processing

Bạn sẽ dùng:

```bash
journalctl
grep
awk
sed
```

và regular expressions để:

- tìm service errors;
- lọc một time window;
- extract relevant records;
- tìm authentication/sudo events;
- correlate event sequence.

Không chỉ xem log trên terminal, mà phải tạo **reproducible extraction**.

---

# Lab 12 — Bash health-check script

Bạn sẽ phát triển một Bash script kiểm tra các hạng mục như:

```text
hostname
uptime
CPU/load
memory
disk capacity
inode capacity
service status
network
listening ports
mounts
```

Script cần có:

- variables;
- conditions;
- command execution;
- exit-code handling;
- meaningful output;
- error handling.

Ví dụ expectation:

```text
Exit 0 → checks passed
Non-zero → one or more checks failed
```

Không được để script báo “SUCCESS” trong khi command bên trong đã fail.

---

# Lab 13 — Backup script

Bạn sẽ tạo backup workflow bằng:

- `tar`;
- và/hoặc `rsync`.

Script cần biết:

```text
source
destination
timestamp
success/failure
exit code
logging
```

Backup phải dùng controlled sample data, không phải production data.

---

# Lab 14 — Restore verification

Sau khi backup, bạn phải:

- restore vào isolated directory;
- verify file count;
- verify content;
- kiểm tra permissions/metadata khi cần;
- so sánh backup với expected data.

Tư duy phải là:

> **Backup success ≠ Recovery success.**

---

# Lab 15 — `cron` scheduling

Bạn sẽ schedule:

- health-check script;
- hoặc backup script.

Sau đó verify:

```text
cron configured
      ↓
job triggered
      ↓
script actually executed
      ↓
output/log created
      ↓
exit status/result acceptable
```

Đây là nơi bạn sẽ thấy vấn đề environment/`PATH` rất thường gặp.

---

# Lab 16 — Failure Injection

Day 2 yêu cầu deliberately inject các lỗi an toàn thuộc ít nhất ba nhóm:

### Service failure

Ví dụ:

- bad configuration fixture;
- service stopped;
- dependency unavailable.

### Network failure

Ví dụ:

- wrong DNS fixture;
- blocked lab port;
- incorrect route fixture.

### Storage failure

Ví dụ:

- filesystem near capacity;
- mount missing;
- wrong lab `fstab` entry được xử lý với recovery plan.

Mục đích không phải “phá server”, mà để tập:

**symptom → evidence → hypothesis → safe test → root cause → fix → verification.**

---

# Lab 17 — RHEL maintenance ticket simulation

Bạn sẽ được cung cấp một prepared maintenance ticket và thực hiện:

### Pre-maintenance

- identify host;
- identify OS/release;
- identify affected package;
- identify affected service;
- capture package baseline;
- capture service baseline;
- check capacity;
- check dependencies;
- assess reboot requirement;
- record risk.

### Maintenance

- thực hiện approved lab change.

### Post-maintenance

- verify package;
- verify service;
- verify application/network;
- inspect logs;
- confirm health.

Nếu post-check fail, bạn phải xác định:

**rollback / recovery / escalation.**

---

# Lab 18 — Bounded UTC server-log extraction

Bạn sẽ nhận một event hoặc ticket và phải extract đúng time window.

Ví dụ:

```text
Ticket event:
2026-10-20T09:41:00Z

Requested evidence:
09:35:00Z – 09:50:00Z
```

Bạn phải lấy:

- relevant journal;
- relevant authentication/service logs;
- rotated logs nếu window đi qua rotation boundary;
- `auditd` nếu có.

Và phải ghi:

- hostname;
- exact UTC window;
- exact command;
- filters;
- collector;
- extraction time;
- errors/limitations.

---

# Lab 19 — Correlate `sudo` / service event với ticket

Bạn sẽ lấy một event ví dụ:

```text
Engineer executed approved sudo operation
        ↓
service restarted
        ↓
application recovered
```

và correlate:

```text
Ticket
   │
   ├── approval
   │
   ├── maintenance time
   │
   └── expected operation
            │
            ↓
       sudo/auth log
            │
            ↓
       systemd event
            │
            ↓
       application/service result
```

Mục tiêu là tạo một timeline mà reviewer có thể follow được.

---

# Lab 20 — Evidence package

Đây là deliverable cuối cùng của Day 2.

Bạn phải tạo một package có thể gồm:

```text
Day2/
├── precheck/
├── postcheck/
├── package/
├── service/
├── networking/
├── storage/
├── logs/
├── backup/
├── restore/
├── scripts/
├── troubleshooting/
└── evidence_manifest.csv
```

Không nhất thiết filename phải đúng như trên, nhưng logic phải bảo đảm traceability.

Theo syllabus, evidence cuối Day 2 phải đủ để reviewer xác minh được:

- pre/post checks;
- package/service baseline;
- native hoặc lossless structured exports khi phù hợp;
- readable filtered log output;
- extraction transcript;
- record counts;
- hashes;
- ticket correlation;
- audit/logging gaps nếu có. Infrastructure_Operations_14_Da…

---

# Mối quan hệ giữa hai Module

Toàn bộ Day 2 thực chất hình thành một chuỗi vận hành:

```text
Package
   ↓
Process
   ↓
systemd Service
   ↓
SELinux / Permissions
   ↓
Network / Ports / Firewall
   ↓
Storage / Mount
   ↓
Logs
   ↓
Bash Automation
   ↓
cron
   ↓
Backup / Restore
   ↓
Health Check
   ↓
Failure Troubleshooting
   ↓
Maintenance
   ↓
Evidence / Escalation
```

Điều này rất quan trọng: **các topic Day 2 không phải những command Linux rời rạc**. Chúng tạo thành một troubleshooting model thống nhất.

Ví dụ khi một web service không hoạt động, bạn có thể phải kiểm tra liên tiếp:

```text
Package installed?
        ↓
Process running?
        ↓
systemd service healthy?
        ↓
Configuration valid?
        ↓
Permissions correct?
        ↓
SELinux allowing access?
        ↓
Port listening?
        ↓
firewalld allowing traffic?
        ↓
IP/route/DNS correct?
        ↓
Filesystem mounted?
        ↓
Disk/inodes available?
        ↓
What do logs say?
```

Đó chính là mindset quan trọng nhất của Day 2.

---

# Kết quả cần đạt sau Day 2

Sau khi hoàn thành cả hai module, bạn phải có khả năng thực hiện một workflow kiểu:

> **Nhận một RHEL server + ticket → xác định baseline → kiểm tra package/process/service/network/storage/security/logs → thực hiện approved change → xử lý failure trong phạm vi L1/L2 → verify service recovery → backup/restore → automate health check → collect audit-quality evidence → rollback hoặc escalate khi vượt boundary.**

Về chương trình tổng thể, Day 1–2 chính là module **Orientation & Linux Administration Foundations**, trong đó RHEL evidence phải thể hiện được users/permissions từ Day 1 và packages/services, SELinux, network, storage, maintenance, log extraction và backup/recovery của Day 2. Infrastructure_Operations_14_Da… Infrastructure_Operations_14_Da…

Khi bạn bảo **“Trình bày Module Lý thuyết Day 2”**, tôi sẽ đi lần lượt toàn bộ phần Lý thuyết trên theo hướng **foundation → mechanism → operational workflow → security → verification → troubleshooting → evidence → escalation**, kèm command và scenario thực tế. Khi bạn bảo **“Trình bày Module Thực hành Day 2”**, tôi sẽ chuyển toàn bộ phần Lab thành một **guided hands-on lab hoàn chỉnh từ setup đến Failure Injection, rollback và evidence package**.
