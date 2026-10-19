# Day 2 — Module Thực hành

## RHEL/Linux Operational Administration Lab

Phần này bám trực tiếp Assignment/Lab của Day 2 trong syllabus: **install và verify approved packages; quản lý `systemd` service; inspect processes và environment variables; recover deliberately failed service; kiểm tra network/DNS/firewall/SELinux; mount storage và kiểm tra capacity; extract/filter logs; xây Bash health-check và backup script có exit-code handling; schedule bằng cron; validate backup/restore; inject service/network/storage failures; collect diagnostic evidence; và tạo Linux troubleshooting/escalation record**. Infrastructure_Operations_14_Da…

Tôi sẽ tổ chức toàn bộ thành **một integrated lab**, giống cách bạn nhận một RHEL server trong enterprise thay vì 15 bài command rời rạc.

---

# 1. Mục tiêu tổng thể của Lab

Sau khi hoàn thành Module Thực hành Day 2, bạn phải có khả năng tự thực hiện workflow:

```text
Receive RHEL server
       │
       ▼
Identify system
       │
       ▼
Verify package/repositories
       │
       ▼
Inspect environment/process/resources
       │
       ▼
Deploy + manage service
       │
       ▼
Verify IP/routing/DNS
       │
       ▼
Validate port/firewall/SELinux
       │
       ▼
Provision + mount storage
       │
       ▼
Inspect capacity/inodes
       │
       ▼
Investigate system/service logs
       │
       ▼
Build health-check automation
       │
       ▼
Build backup automation
       │
       ▼
Schedule with cron
       │
       ▼
Restore + integrity check
       │
       ▼
Inject failures
       │
       ▼
Troubleshoot systematically
       │
       ▼
Collect evidence
       │
       ▼
Rollback / Escalate / Close
```

Mục tiêu quan trọng nhất không phải là “chạy hết command”.

Bạn phải chứng minh được:

```text
I know:
- what I changed
- why I changed it
- what state existed before
- how I verified the change
- how I detect failure
- how I roll back
- what evidence I keep
- when I must escalate
```

---

# 2. Lab Environment

## Recommended topology

Tốt nhất sử dụng:

```text
                    Lab Network
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
     RHEL Server VM          Linux Client VM
     rhel-day2-01            client-day2-01
     192.168.56.10           192.168.56.20
```

Linux Client không bắt buộc, nhưng rất hữu ích để kiểm tra firewall và remote connectivity.

Nếu chỉ có một VM, vẫn có thể làm phần lớn lab bằng:

```text
localhost
127.0.0.1
server's own IP
```

nhưng một số network/firewall tests sẽ không thực tế bằng client riêng.

---

# 3. Recommended server specification

Ví dụ:

```text
OS:        RHEL 9.x hoặc compatible authorized RHEL lab
CPU:       2 vCPU
RAM:       4 GB
Disk 1:    OS disk
Disk 2:    2 GB disposable lab disk
Network:   1 NIC
Account:   non-root administrative user with approved sudo
SELinux:   Enforcing
Firewall:  firewalld
```

Các command được thiết kế chủ yếu theo RHEL 8/9-style administration. Các chi tiết phụ thuộc release phải luôn được xác minh bằng:

```bash
cat /etc/redhat-release
cat /etc/os-release
```

DNF là package-management workflow chuẩn trên RHEL 9 và tự xử lý dependencies của package từ configured repositories. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_software_with_the_dnf_tool/assembly_installing-rhel-9-content_managing-software-with-the-dnf-tool?utm_source=chatgpt.com)

---

# 4. Safety Rules — bắt buộc đọc trước khi lab

## Rule 1 — Không làm trên production

Tất cả bước:

```text
firewall
SELinux
service failure
disk partitioning
filesystem formatting
/etc/fstab
storage filling
cron
```

chỉ thực hiện trên:

```text
Lab
Sandbox
Disposable VM
Authorized non-production server
```

---

## Rule 2 — Có VM snapshot trước lab

Nếu hypervisor hỗ trợ snapshot:

```text
Snapshot name:
Day2_PreLab
```

Đặc biệt trước storage và `/etc/fstab` lab.

---

## Rule 3 — Không sử dụng `/dev/sda` hoặc OS disk

Trước mọi thao tác destructive:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS
```

Ví dụ:

```text
NAME   SIZE TYPE FSTYPE MOUNTPOINTS
sda     40G disk
├─sda1   1G part xfs    /boot
└─sda2  39G part LVM
sdb      2G disk
```

Nếu `/dev/sda` đang chứa OS:

```text
DO NOT TOUCH /dev/sda
```

Chỉ dùng dedicated disposable disk, ví dụ:

```text
/dev/sdb
```

sau khi bạn đã xác minh chắc chắn.

---

## Rule 4 — Không disable SELinux

Lab SELinux sẽ sửa **label/root cause**, không dùng:

```bash
setenforce 0
```

như một “solution”.

---

## Rule 5 — Không remove SSH firewall rule

Nếu bạn SSH vào VM từ xa, tuyệt đối không thử:

```bash
firewall-cmd --remove-service=ssh
```

trong lab này.

---

## Rule 6 — Không fill root filesystem

Storage-capacity failure injection chỉ thực hiện trên dedicated lab filesystem.

Không tạo giant file ở:

```text
/
 /var
 /home
```

nếu các directory đó thuộc production/system filesystem.

---

# 5. Evidence Repository

Ngay đầu lab tạo nơi giữ evidence.

```bash
mkdir -p ~/day2-evidence/{baseline,packages,processes,service,network,security,storage,logs,scripts,backup,failures}
```

Verify:

```bash
find ~/day2-evidence -maxdepth 1 -type d | sort
```

Có thể tạo variable:

```bash
export EVIDENCE="$HOME/day2-evidence"
```

Verify:

```bash
echo "$EVIDENCE"
```

Expected:

```text
/home/<user>/day2-evidence
```

---

# LAB 1 — System Baseline & Inventory

## Objective

Xác định chính xác server nào đang được quản lý trước khi thay đổi bất cứ thứ gì.

Đây là habit rất quan trọng trong enterprise.

---

## Step 1 — Host identity

```bash
hostname
hostnamectl
```

Thu thập:

```text
hostname
OS
kernel
architecture
virtualization
```

---

## Step 2 — OS release

```bash
cat /etc/os-release
cat /etc/redhat-release
```

Evidence:

```bash
cat /etc/os-release | tee "$EVIDENCE/baseline/os-release.txt"
```

---

## Step 3 — Kernel

```bash
uname -r
uname -a
```

---

## Step 4 — Uptime

```bash
uptime
```

Hoặc:

```bash
uptime -p
```

---

## Step 5 — CPU và memory

```bash
lscpu
free -h
```

---

## Step 6 — Storage baseline

```bash
lsblk
df -hT
df -i
```

---

## Step 7 — Network baseline

```bash
ip -br addr
ip route
```

---

## Step 8 — SELinux và firewall baseline

```bash
getenforce
sudo firewall-cmd --state
```

Expected:

```text
Enforcing
running
```

tùy environment.

---

## Evidence

Bạn cần lưu:

```text
hostname
OS release
kernel
uptime
CPU
memory
disk
filesystem
IP
route
SELinux
firewall state
timestamp
```

Ví dụ:

```bash
date --iso-8601=seconds | tee "$EVIDENCE/baseline/timestamp.txt"
```

---

# LAB 2 — Package Management: `dnf`, `yum`, `rpm`

## Objective

Thực hiện package lifecycle đúng operational workflow:

```text
Identify requirement
→ inspect repository
→ verify current state
→ install approved packages
→ verify package
→ identify installed files
```

---

# 2.1 Inspect repositories

Read-only:

```bash
dnf repolist
```

Chi tiết:

```bash
dnf repolist -v
```

Bạn cần tìm:

```text
repo id
repo name
enabled status
repository source
```

---

## RHEL subscription awareness

Nếu system là registered RHEL:

```bash
sudo subscription-manager status
```

và:

```bash
sudo subscription-manager repos --list-enabled
```

Không phải mọi cloud image hoặc lab image đều có cùng subscription workflow.

Nếu command không có hoặc status khác dự kiến:

**không tự đăng ký bằng credential thật nếu lab không yêu cầu.**

Ghi lại:

```text
Subscription/repository management owned by <team/environment>
```

---

# 2.2 Kiểm tra package trước khi install

Ví dụ service chính của Day 2:

```text
httpd
```

Kiểm tra:

```bash
rpm -q httpd
```

Nếu chưa cài:

```text
package httpd is not installed
```

DNF:

```bash
dnf info httpd
```

---

# 2.3 Packages dùng trong lab

Có thể cần:

```text
httpd
curl
rsync
cronie
xfsprogs
parted
policycoreutils-python-utils
```

Kiểm tra trước:

```bash
rpm -q httpd curl rsync cronie xfsprogs parted policycoreutils-python-utils
```

---

# 2.4 Installation

### Configuration-changing command

```bash
sudo dnf install httpd curl rsync cronie xfsprogs parted policycoreutils-python-utils
```

Tôi cố tình không thêm `-y` trong lần học đầu tiên.

Hãy đọc transaction trước khi confirm.

Quan sát:

```text
Installing:
Dependencies:
Transaction Summary:
Download size:
Installed size:
```

Sau đó xác nhận.

---

# 2.5 Verify bằng RPM

```bash
rpm -q httpd
rpm -q curl
rpm -q rsync
```

Ví dụ:

```text
httpd-...
curl-...
rsync-...
```

---

# 2.6 Package information

```bash
rpm -qi httpd
```

Tìm:

```text
Name
Version
Release
Architecture
Install Date
Vendor
```

---

# 2.7 Package file list

```bash
rpm -ql httpd | less
```

Bạn sẽ thấy:

```text
configuration files
binary files
directories
documentation
systemd-related files
```

---

# 2.8 File ownership

Ví dụ:

```bash
rpm -qf /usr/sbin/httpd
```

Expected:

```text
httpd-...
```

Operationally, command này hữu ích khi bạn thấy một binary/configuration và muốn biết package owner.

---

# Verification

Bạn phải chứng minh:

```text
Repository available
Package installed
Correct architecture
Package database recognizes it
Executable exists
```

Ví dụ:

```bash
command -v httpd
rpm -q httpd
```

---

# Evidence

```bash
dnf repolist > "$EVIDENCE/packages/repositories.txt"
rpm -q httpd curl rsync cronie xfsprogs parted \
    > "$EVIDENCE/packages/installed-packages.txt"
```

---

# Troubleshooting checkpoint

Nếu:

```bash
sudo dnf install httpd
```

báo package unavailable, investigation:

```text
1. Package name correct?
2. OS/release correct?
3. Repository enabled?
4. DNS?
5. Default route?
6. Proxy?
7. Firewall?
8. Subscription/content access?
```

Không giải quyết bằng cách tải random RPM trên Internet.

---

# LAB 3 — Environment Variables

## Objective

Hiểu và chứng minh variable inheritance.

---

# 3.1 Inspect environment

```bash
env | sort
```

Hoặc:

```bash
printenv
```

---

# 3.2 Important variables

```bash
echo "$USER"
echo "$HOME"
echo "$SHELL"
echo "$PATH"
```

---

# 3.3 Create shell variable

```bash
DAY2_VAR="RHEL_OPERATION_LAB"
```

Verify:

```bash
echo "$DAY2_VAR"
```

---

# 3.4 Test child process before export

```bash
bash -c 'echo "$DAY2_VAR"'
```

Có thể không có output.

Vì shell variable chưa được exported.

---

# 3.5 Export

```bash
export DAY2_VAR
```

Retry:

```bash
bash -c 'echo "$DAY2_VAR"'
```

Expected:

```text
RHEL_OPERATION_LAB
```

Bạn vừa chứng minh:

```text
parent shell
   │
   ├─ shell-only variable
   │
   └─ exported environment variable
             ↓
         child process
```

---

# 3.6 Inspect `PATH`

```bash
echo "$PATH" | tr ':' '\n'
```

Nếu một script phụ thuộc:

```text
custom executable
```

nhưng directory không nằm trong cron `PATH`, script có thể fail.

Đây sẽ quay lại ở Cron Lab.

---

# 3.7 Cleanup

```bash
unset DAY2_VAR
```

---

# Evidence

```bash
env | sort > "$EVIDENCE/processes/environment.txt"
```

Không lưu environment nếu nó chứa:

```text
password
secret
token
cloud credential
```

Trong enterprise phải review/redact evidence trước khi attach ticket.

---

# LAB 4 — Process Inspection, States, Resource Usage & Signals

## Objective

Quan sát process thay vì chỉ dựa vào service status.

---

# 4.1 Process list

```bash
ps -ef
```

hoặc:

```bash
ps aux
```

---

# 4.2 Tìm process cụ thể

Sau khi `httpd` chạy ở lab sau, dùng:

```bash
pgrep -a httpd
```

Hoặc:

```bash
ps -ef | grep '[h]ttpd'
```

---

# 4.3 Process hierarchy

```bash
ps -ef --forest
```

Bạn sẽ thấy conceptual relationship:

```text
systemd
 └─ httpd
     ├─ httpd worker
     ├─ httpd worker
     └─ httpd worker
```

---

# 4.4 Custom process display

```bash
ps -eo user,pid,ppid,state,%cpu,%mem,etime,comm --sort=-%cpu | head
```

Fields:

```text
USER
PID
PPID
STATE
CPU
MEM
elapsed time
command
```

---

# 4.5 Resource inspection

```bash
uptime
free -h
top
```

Thoát `top`:

```text
q
```

Có thể dùng:

```bash
ps -eo pid,user,%cpu,%mem,comm --sort=-%mem | head
```

---

# 4.6 Safe signal experiment

Không kill critical service.

Tạo harmless process:

```bash
sleep 600 &
```

Shell sẽ trả PID, ví dụ:

```text
[1] 5421
```

Lưu PID:

```bash
TESTPID=$!
echo "$TESTPID"
```

Check:

```bash
ps -o pid,ppid,state,etime,cmd -p "$TESTPID"
```

---

## Send `SIGTERM`

```bash
kill -TERM "$TESTPID"
```

Verify:

```bash
ps -p "$TESTPID"
```

Expected: process không còn tồn tại.

---

## Vì sao không bắt đầu bằng `kill -9`?

Vì:

```text
SIGTERM
→ process có cơ hội graceful cleanup
```

trong khi:

```text
SIGKILL
→ kernel terminate immediately
```

Trong production:

```text
graceful first
force only when justified
```

---

# Evidence

```bash
ps -eo user,pid,ppid,state,%cpu,%mem,etime,comm \
   > "$EVIDENCE/processes/process-snapshot.txt"

free -h > "$EVIDENCE/processes/memory.txt"
uptime > "$EVIDENCE/processes/uptime.txt"
```

---

# LAB 5 — `systemd` Service Lifecycle

`systemctl` trên RHEL dùng để start/stop/restart service, inspect status và manage boot enablement. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/managing-systemd_configuring-basic-system-settings?utm_source=chatgpt.com)

## Objective

Quản lý `httpd` service và hiểu:

```text
installed
≠
running
≠
enabled
≠
healthy
```

---

# 5.1 Check initial state

```bash
systemctl status httpd
```

Đừng lo nếu thấy:

```text
inactive
```

sau package installation.

---

# 5.2 Check enabled state

```bash
systemctl is-enabled httpd
```

Possibilities:

```text
enabled
disabled
static
masked
```

---

# 5.3 Start service

### Configuration-changing

```bash
sudo systemctl start httpd
```

Verify:

```bash
systemctl status httpd --no-pager
```

Và:

```bash
systemctl is-active httpd
```

Expected:

```text
active
```

---

# 5.4 Verify process

```bash
pgrep -a httpd
```

Đây là verification layer thứ hai.

---

# 5.5 Verify listening port

```bash
sudo ss -lntp | grep ':80'
```

Có thể thấy:

```text
LISTEN ... :80 ... httpd
```

Đây là verification layer thứ ba.

---

# 5.6 Verify application

```bash
curl -I http://127.0.0.1/
```

Đây là functional verification.

Bạn đang chứng minh:

```text
systemd active
        +
process exists
        +
port listens
        +
HTTP responds
```

Tốt hơn nhiều so với chỉ:

```bash
systemctl status
```

---

# 5.7 Enable at boot

```bash
sudo systemctl enable httpd
```

Verify:

```bash
systemctl is-enabled httpd
```

Expected:

```text
enabled
```

---

# 5.8 Distinguish start vs enable

Hiện tại:

```text
start
→ runtime state

enable
→ startup configuration
```

Test intellectually:

```text
active + disabled
```

là trạng thái hợp lệ.

---

# 5.9 Restart

```bash
sudo systemctl restart httpd
```

Verify lại cả:

```bash
systemctl is-active httpd
sudo ss -lntp | grep ':80'
curl -I http://127.0.0.1/
```

---

# LAB 6 — Create Web Content

## Objective

Tạo controlled application data để dùng xuyên suốt network/firewall/SELinux/backup labs.

```bash
echo 'Day 2 RHEL Operations Lab' | \
sudo tee /var/www/html/index.html
```

Verify:

```bash
cat /var/www/html/index.html
```

HTTP verification:

```bash
curl http://127.0.0.1/
```

Expected:

```text
Day 2 RHEL Operations Lab
```

---

# LAB 7 — Deliberately Failed Service & Recovery

Đây là requirement quan trọng của syllabus. Infrastructure_Operations_14_Da…

## Objective

Không chỉ restart service mà phải:

```text
Create failure
→ observe
→ collect logs
→ identify root cause
→ repair
→ verify
```

---

# Safety

Chỉ lab.

Không tạo syntax error trên production application.

---

# 7.1 Baseline trước failure

```bash
systemctl is-active httpd
sudo httpd -t
curl http://127.0.0.1/
```

Expected:

```text
active
Syntax OK
Day 2 RHEL Operations Lab
```

Lưu evidence:

```bash
systemctl status httpd --no-pager \
  > "$EVIDENCE/service/httpd-before-failure.txt"
```

---

# 7.2 Inject invalid configuration

Tạo file riêng để dễ rollback:

```bash
echo 'Day2InvalidDirective enabled' | \
sudo tee /etc/httpd/conf.d/day2-broken.conf
```

Không chỉnh trực tiếp `/etc/httpd/conf/httpd.conf`.

Đây là operational principle tốt:

```text
isolated change
→ simple rollback
```

---

# 7.3 Validate configuration BEFORE restart

```bash
sudo httpd -t
```

Expected failure tương tự:

```text
Invalid command ...
```

Đây là một bài học cực quan trọng:

> Với service hỗ trợ config validation, validate trước restart.

Nếu đây là production, bước này sẽ ngăn outage.

---

# 7.4 Tạo deliberately failed service

Trong lab:

```bash
sudo systemctl stop httpd
sudo systemctl start httpd
```

Start sẽ fail vì configuration không hợp lệ.

---

# 7.5 Observe failure

```bash
systemctl status httpd --no-pager
```

Bạn phải nhìn:

```text
Active: failed
exit code
recent log messages
```

---

# 7.6 Check journal

```bash
sudo journalctl -u httpd --since "-10 min" --no-pager
```

RHEL hỗ trợ `journalctl` để query system journal và filter messages của services. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_basic_system_settings/assembly_troubleshooting-problems-using-log-files_configuring-basic-system-settings?utm_source=chatgpt.com)

---

# 7.7 Confirm root cause

```bash
sudo httpd -t
```

Bạn đã correlation:

```text
Symptom:
httpd failed

Evidence 1:
systemctl shows failed

Evidence 2:
journal reports configuration problem

Evidence 3:
httpd -t reproduces syntax/config error

Root cause:
day2-broken.conf contains invalid directive
```

---

# 7.8 Correct root cause

```bash
sudo rm /etc/httpd/conf.d/day2-broken.conf
```

Validate:

```bash
sudo httpd -t
```

Expected:

```text
Syntax OK
```

---

# 7.9 Recover

```bash
sudo systemctl start httpd
```

---

# 7.10 Multi-layer verification

```bash
systemctl is-active httpd
pgrep -a httpd
sudo ss -lntp | grep ':80'
curl http://127.0.0.1/
```

Expected:

```text
active
processes exist
TCP/80 listening
HTTP page returned
```

---

# 7.11 Evidence

```bash
systemctl status httpd --no-pager \
  > "$EVIDENCE/service/httpd-after-recovery.txt"

sudo journalctl -u httpd --since "-15 min" --no-pager \
  > "$EVIDENCE/service/httpd-recovery-journal.txt"
```

Your incident record should say:

```text
Failure:
HTTP service unavailable.

Root cause:
Invalid Apache configuration directive introduced in controlled lab.

Correction:
Removed invalid include configuration.

Verification:
systemd active
process present
TCP/80 listening
HTTP request successful

Rollback:
Removal of day2-broken.conf returned configuration to baseline.
```

---

# LAB 8 — Network Configuration Inspection

## Objective

Verify:

```text
interface
IP
route
gateway
DNS
transport
application
```

theo thứ tự.

---

# 8.1 Interface overview

```bash
ip -br link
```

Ví dụ:

```text
lo       UNKNOWN
ens192   UP
```

---

# 8.2 Addressing

```bash
ip -br addr
```

Ví dụ:

```text
ens192 UP 192.168.56.10/24
```

Evidence:

```bash
ip -br addr > "$EVIDENCE/network/ip-address.txt"
```

---

# 8.3 Detailed interface

```bash
ip addr show
```

---

# 8.4 Routing table

```bash
ip route
```

Tìm:

```text
connected route
default route
gateway
interface
```

Example:

```text
default via 192.168.56.1 dev ens192
192.168.56.0/24 dev ens192 proto kernel ...
```

---

# 8.5 Route decision

Nếu Internet/network policy cho phép:

```bash
ip route get 8.8.8.8
```

Command này giúp xem kernel dự kiến chọn route nào.

Nó không phải connectivity test hoàn chỉnh.

---

# 8.6 Determine default gateway

```bash
GW=$(ip route | awk '/^default/ {print $3; exit}')
echo "$GW"
```

---

# 8.7 Ping gateway

Nếu environment cho phép ICMP:

```bash
ping -c 3 "$GW"
```

Không nhận response không tự động chứng minh gateway down vì ICMP có thể bị filter.

---

# LAB 9 — DNS Resolution

## Objective

Tách:

```text
IP connectivity
```

khỏi:

```text
name resolution
```

---

# 9.1 Check resolver view

```bash
cat /etc/resolv.conf
```

Chỉ đọc.

Không sửa ngay.

---

# 9.2 Name Service Switch

```bash
grep '^hosts:' /etc/nsswitch.conf
```

Bạn có thể thấy:

```text
hosts: files dns ...
```

Điều này giải thích tại sao `/etc/hosts` có thể tham gia resolution trước DNS tùy configuration.

---

# 9.3 Local resolution

```bash
getent hosts localhost
```

---

# 9.4 Resolve external/enterprise hostname

Nếu Internet được phép:

```bash
getent hosts example.com
```

Nếu dùng corporate DNS:

```bash
getent hosts <approved-hostname>
```

---

## Optional `dig`

Nếu package phù hợp được approve:

```bash
dig example.com
```

Nhưng `getent` rất hữu ích vì nó kiểm tra theo host-resolution mechanism của system thay vì chỉ query DNS trực tiếp.

---

# Operational comparison

Case A:

```text
ping <IP> works
getent hosts <name> fails
```

Likely investigation direction:

```text
DNS/resolver
```

Case B:

```text
DNS resolves
but destination IP unreachable
```

Direction:

```text
route/network/firewall
```

Case C:

```text
DNS + network work
but curl fails
```

Direction:

```text
port/firewall/application
```

---

# LAB 10 — `ss`, TCP Port & HTTP Validation

## Objective

Chứng minh application service ở transport layer.

---

# 10.1 Listening TCP sockets

```bash
sudo ss -lntp
```

Options:

```text
-l listen
-n numeric
-t TCP
-p process
```

---

# 10.2 Filter HTTP

```bash
sudo ss -lntp | grep ':80'
```

---

# 10.3 Application test

```bash
curl -v http://127.0.0.1/
```

`-v` giúp quan sát:

```text
connect attempt
remote endpoint
request
response headers
```

Không cần dùng `-v` khi thu evidence có sensitive HTTP headers.

---

# 10.4 Test server's own IP

```bash
SERVER_IP=$(hostname -I | awk '{print $1}')
echo "$SERVER_IP"

curl "http://$SERVER_IP/"
```

---

# 10.5 `tracepath`

Nếu available:

```bash
tracepath <approved-remote-host>
```

hoặc:

```bash
traceroute <approved-remote-host>
```

Mục tiêu:

```text
identify path
not prove every silent hop is failed
```

---

# Network Evidence Pack

```bash
ip -br addr > "$EVIDENCE/network/address.txt"
ip route > "$EVIDENCE/network/routes.txt"
cat /etc/resolv.conf > "$EVIDENCE/network/resolver.txt"
sudo ss -lntp > "$EVIDENCE/network/listeners.txt"
```

Nếu `/etc/resolv.conf` có internal DNS addresses, evidence phải được xử lý theo policy tổ chức.

---

# LAB 11 — `firewalld`

RHEL `firewalld` tách **runtime** và **permanent** configurations; runtime-only change không tồn tại sau reload/restart. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld_firewall-packet-filters?utm_source=chatgpt.com)

## Objective

Cho phép HTTP một cách có kiểm soát và verify cả configuration lẫn traffic.

---

# 11.1 Check firewall service

```bash
systemctl status firewalld --no-pager
```

---

# 11.2 Identify active zone

```bash
sudo firewall-cmd --get-active-zones
```

Ví dụ:

```text
public
  interfaces: ens192
```

---

# 11.3 Inspect current rules

```bash
sudo firewall-cmd --list-all
```

Permanent:

```bash
sudo firewall-cmd --permanent --list-all
```

So sánh hai output.

---

# 11.4 Check HTTP permission

```bash
sudo firewall-cmd --query-service=http
```

Possible:

```text
yes
no
```

---

# 11.5 Safe temporary test

Nếu HTTP chưa được allow, có thể thêm **runtime** trước:

```bash
sudo firewall-cmd --add-service=http
```

Verify:

```bash
sudo firewall-cmd --query-service=http
```

---

# 11.6 Remote verification

Từ Client VM:

```bash
curl http://192.168.56.10/
```

Expected:

```text
Day 2 RHEL Operations Lab
```

Đây mới là firewall verification có ý nghĩa.

---

# 11.7 Make persistent — chỉ sau khi approved

Một cách:

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --query-service=http
sudo firewall-cmd --permanent --query-service=http
```

RHEL cũng hỗ trợ workflow runtime-to-permanent, nhưng trong enterprise phải hiểu rằng việc persist toàn bộ runtime state cần được kiểm soát; đừng vô thức persist các temporary changes khác. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld_firewall-packet-filters?utm_source=chatgpt.com)

---

# Firewall rollback

Nếu HTTP chỉ phục vụ lab:

```bash
sudo firewall-cmd --permanent --remove-service=http
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --query-service=http
```

---

# Firewall Evidence

```bash
sudo firewall-cmd --get-active-zones \
  > "$EVIDENCE/security/firewall-zones.txt"

sudo firewall-cmd --list-all \
  > "$EVIDENCE/security/firewall-runtime.txt"

sudo firewall-cmd --permanent --list-all \
  > "$EVIDENCE/security/firewall-permanent.txt"
```

---

# LAB 12 — SELinux Access Control

RHEL guidance cho SELinux troubleshooting tập trung vào xác định incorrect context và dùng `restorecon`/persistent context rules thay vì disable SELinux. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/troubleshooting-problems-related-to-selinux_using-selinux?utm_source=chatgpt.com)

## Objective

Tạo controlled SELinux denial rồi sửa đúng root cause.

---

# 12.1 Verify SELinux state

```bash
getenforce
```

Expected:

```text
Enforcing
```

Chi tiết:

```bash
sestatus
```

---

# 12.2 Inspect web content context

```bash
ls -lZ /var/www/html/
```

Bạn thường sẽ thấy type phù hợp với HTTP content.

Ví dụ conceptual:

```text
httpd_sys_content_t
```

---

# 12.3 Create test file

```bash
echo "SELinux test page" | \
sudo tee /var/www/html/day2-selinux.html
```

Verify:

```bash
curl http://127.0.0.1/day2-selinux.html
```

Expected:

```text
SELinux test page
```

---

# 12.4 Check expected context

```bash
matchpathcon -V /var/www/html/day2-selinux.html
```

---

# 12.5 Inject controlled wrong label

**Lab only:**

```bash
sudo chcon -t user_home_t /var/www/html/day2-selinux.html
```

Verify:

```bash
ls -lZ /var/www/html/day2-selinux.html
```

---

# 12.6 Attempt access

```bash
curl -i http://127.0.0.1/day2-selinux.html
```

Bạn có thể nhận HTTP denial như:

```text
403 Forbidden
```

tùy exact environment/configuration.

---

# 12.7 Traditional permissions check

```bash
ls -l /var/www/html/day2-selinux.html
```

Nếu UNIX mode vẫn readable nhưng HTTP access bị deny, bạn có evidence rằng phải điều tra layer khác.

---

# 12.8 Find AVC evidence

Nếu `auditd` available:

```bash
sudo ausearch -m AVC -ts recent
```

Hoặc:

```bash
sudo journalctl --since "-10 min" | grep -i 'avc'
```

Không giả định absence của grep output luôn chứng minh SELinux không liên quan; logging stack có thể khác.

---

# 12.9 Correct root cause

Không:

```bash
chmod 777
```

Không:

```bash
setenforce 0
```

Đúng:

```bash
sudo restorecon -v /var/www/html/day2-selinux.html
```

RHEL documentation dùng `restorecon` để áp lại expected SELinux file context. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/troubleshooting-problems-related-to-selinux_using-selinux?utm_source=chatgpt.com)

---

# 12.10 Verify

```bash
ls -lZ /var/www/html/day2-selinux.html

curl http://127.0.0.1/day2-selinux.html
```

Expected:

```text
SELinux test page
```

---

# Evidence

```bash
getenforce > "$EVIDENCE/security/selinux-mode.txt"

ls -lZ /var/www/html \
  > "$EVIDENCE/security/selinux-contexts-after.txt"

sudo ausearch -m AVC -ts recent \
  > "$EVIDENCE/security/selinux-avc.txt" 2>&1
```

---

# LAB 13 — Storage: Disk → Partition → Filesystem → Mount

Đây là phần có **destructive commands**.

## Safety gate

Bạn chỉ tiếp tục khi:

```text
1. VM is non-production.
2. Snapshot exists.
3. Dedicated disposable disk exists.
4. Disk contains no required data.
5. You verified the device multiple times.
```

Giả sử lab disk:

```text
/dev/sdb
```

**Không copy `/dev/sdb` nếu server của bạn dùng tên khác.**

---

# 13.1 Identify disks

Read-only:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,UUID
```

Thêm:

```bash
sudo blkid
```

Hãy xác định:

```text
OS disk
Lab disk
Existing filesystem
Existing mountpoint
```

---

# 13.2 Verify target disk is unused

Ví dụ:

```bash
lsblk /dev/sdb
```

Expected lab disk:

```text
sdb  2G disk
```

không có mounted partition.

Nếu có:

```text
filesystem
LVM
mount
data
```

**DỪNG.**

---

# 13.3 Partition disk

### DESTRUCTIVE — LAB DISK ONLY

Ví dụ dùng GPT:

```bash
sudo parted /dev/sdb
```

Trong interactive prompt:

```text
mklabel gpt
mkpart primary xfs 1MiB 100%
print
quit
```

Hoặc scripted:

```bash
sudo parted -s /dev/sdb mklabel gpt
sudo parted -s /dev/sdb mkpart primary xfs 1MiB 100%
```

Sau đó:

```bash
sudo partprobe /dev/sdb
```

Verify:

```bash
lsblk /dev/sdb
```

Expected:

```text
sdb
└─sdb1
```

---

# 13.4 Create filesystem

### DESTRUCTIVE

```bash
sudo mkfs.xfs /dev/sdb1
```

Không chạy command này trên partition có dữ liệu cần giữ.

Verify:

```bash
lsblk -f /dev/sdb
```

Expected:

```text
sdb1 xfs ... UUID=...
```

---

# 13.5 Create mount point

```bash
sudo mkdir -p /mnt/day2data
```

---

# 13.6 Temporary mount

```bash
sudo mount /dev/sdb1 /mnt/day2data
```

Verify:

```bash
findmnt /mnt/day2data
```

Red Hat khuyến nghị `findmnt` để liệt kê và inspect mounted filesystems. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/mounting-file-systems_managing-file-systems?utm_source=chatgpt.com)

---

# 13.7 Functional write test

```bash
echo "Day2 storage test" | \
sudo tee /mnt/day2data/storage-test.txt
```

Read:

```bash
cat /mnt/day2data/storage-test.txt
```

---

# LAB 14 — Persistent Mount with `/etc/fstab`

RHEL sử dụng `/etc/fstab` cho persistent mount definitions, và Red Hat hướng dẫn sử dụng persistent identifiers như UUID; `systemd` sinh mount units từ các entries này. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_file_systems/assembly_persistently-mounting-file-systems_managing-file-systems?utm_source=chatgpt.com)

## 14.1 Retrieve UUID

```bash
sudo blkid /dev/sdb1
```

Hoặc:

```bash
UUID=$(sudo blkid -s UUID -o value /dev/sdb1)
echo "$UUID"
```

---

# 14.2 Back up `/etc/fstab`

Bắt buộc:

```bash
sudo cp -a /etc/fstab /etc/fstab.day2-backup
```

Verify:

```bash
sudo ls -l /etc/fstab*
```

---

# 14.3 Create entry

Ví dụ:

```text
UUID=<YOUR-UUID> /mnt/day2data xfs defaults,nofail 0 0
```

`nofail` có thể hữu ích trong lab để giảm boot impact nếu lab disk absent, nhưng production options phải theo design/standard của organization.

Không copy placeholder `<YOUR-UUID>`.

---

## Safe way to append

```bash
echo "UUID=$UUID /mnt/day2data xfs defaults,nofail 0 0" | \
sudo tee -a /etc/fstab
```

---

# 14.4 Reload systemd configuration

```bash
sudo systemctl daemon-reload
```

---

# 14.5 Critical verification BEFORE reboot

Đầu tiên:

```bash
findmnt --verify
```

Nếu environment/tool version hỗ trợ.

Sau đó test thực tế.

Unmount:

```bash
sudo umount /mnt/day2data
```

Verify:

```bash
findmnt /mnt/day2data
```

Không nên có active mount.

Mount bằng fstab:

```bash
sudo mount /mnt/day2data
```

Verify:

```bash
findmnt /mnt/day2data
```

---

# 14.6 Verify data survived unmount/remount

```bash
cat /mnt/day2data/storage-test.txt
```

Expected:

```text
Day2 storage test
```

Đây là important verification:

```text
filesystem
+
fstab
+
mount
+
data accessibility
```

---

# `/etc/fstab` rollback

Nếu entry sai:

```bash
sudo cp -a /etc/fstab.day2-backup /etc/fstab
sudo systemctl daemon-reload
```

Sau đó:

```bash
sudo mount -a
```

phải được dùng cẩn thận vì nó xử lý configured mounts.

---

# LAB 15 — Disk Capacity & Inodes

## Objective

Phân biệt:

```text
block capacity
```

và:

```text
inode capacity
```

---

# 15.1 Filesystem capacity

```bash
df -hT /mnt/day2data
```

Observe:

```text
Filesystem
Type
Size
Used
Avail
Use%
Mounted on
```

---

# 15.2 Inodes

```bash
df -i /mnt/day2data
```

Observe:

```text
Inodes
IUsed
IFree
IUse%
```

---

# 15.3 Directory consumption

```bash
sudo du -sh /mnt/day2data
```

Difference:

```text
df
→ filesystem-level allocation

du
→ files/directories visible through directory tree
```

Khi `df` và `du` chênh lớn, có thể cần investigate:

```text
deleted-but-open files
mounts
reserved blocks/filesystem behavior
```

tùy filesystem.

---

# LAB 16 — Safe Storage Capacity Failure Injection

## Objective

Tạo gần-full filesystem **chỉ trên lab disk**.

---

# Confirm correct filesystem first

```bash
findmnt /mnt/day2data
df -h /mnt/day2data
```

Nếu output không chỉ rõ dedicated lab filesystem:

**Không tiếp tục.**

---

# Create controlled large file

Ví dụ trên 2 GB lab disk:

```bash
sudo fallocate -l 1500M /mnt/day2data/day2-fill.bin
```

Verify:

```bash
df -h /mnt/day2data
```

Mục tiêu là tạo:

```text
high utilization
```

không nhất thiết 100%.

Không cố tình làm filesystem 100% nếu không cần.

---

# Symptom

Health-check sau này có thể báo:

```text
WARNING: filesystem usage above threshold
```

---

# Rollback immediately

```bash
sudo rm /mnt/day2data/day2-fill.bin
```

Verify:

```bash
df -h /mnt/day2data
```

---

# Evidence

```bash
df -hT /mnt/day2data \
  > "$EVIDENCE/storage/filesystem-capacity.txt"

df -i /mnt/day2data \
  > "$EVIDENCE/storage/inode-capacity.txt"

lsblk -f \
  > "$EVIDENCE/storage/lsblk.txt"

findmnt /mnt/day2data \
  > "$EVIDENCE/storage/mount.txt"
```

---

# LAB 17 — `journalctl`, syslog & Log Investigation

## Objective

Tìm đúng log trong đúng time window.

---

# 17.1 Current boot

```bash
sudo journalctl -b --no-pager | less
```

---

# 17.2 `httpd`

```bash
sudo journalctl -u httpd --no-pager
```

---

# 17.3 Recent events

```bash
sudo journalctl --since "-15 min" --no-pager
```

---

# 17.4 Warning and higher

```bash
sudo journalctl -p warning --since "-1 hour" --no-pager
```

Lưu ý: priority không hoàn toàn tương đương business impact.

---

# 17.5 Kernel logs

```bash
sudo journalctl -k --since "-1 hour" --no-pager
```

Useful cho:

```text
disk
I/O
NIC
kernel
OOM
```

---

# 17.6 Traditional log files

Inspect:

```bash
sudo ls -lh /var/log
```

Tùy configuration có thể gặp:

```text
/var/log/messages
/var/log/secure
/var/log/cron
/var/log/httpd/
```

Không giả định mọi environment có giống hệt nhau.

---

# LAB 18 — `grep`, `awk`, `sed`, Regex

## Objective

Lọc log thay vì đọc hàng nghìn dòng.

---

# 18.1 `grep`

```bash
sudo journalctl -u httpd --since "-1 hour" --no-pager |
grep -i 'error'
```

---

# 18.2 Multiple patterns

```bash
sudo journalctl -u httpd --since "-1 hour" --no-pager |
grep -Ei 'error|failed|denied'
```

`-E`:

```text
extended regex
```

`-i`:

```text
case-insensitive
```

---

# 18.3 Context

```bash
grep -B 2 -A 3 'ERROR' logfile
```

Meaning:

```text
-B 2
two lines before

-A 3
three lines after
```

Useful vì root cause có thể nằm trước error line.

---

# 18.4 `awk`

Example process report:

```bash
ps -eo user,pid,%cpu,%mem,comm |
awk 'NR==1 || $3 > 1 {print}'
```

Mục tiêu:

```text
show header
+
processes above selected CPU threshold
```

---

# 18.5 Filesystem processing

```bash
df -P |
awk 'NR==1 || $5+0 >= 80 {print}'
```

Ý tưởng:

```text
report filesystems >= 80%
```

Threshold thực tế phải theo monitoring standard của organization.

---

# 18.6 `sed`

Read-only transform:

```bash
printf '%s\n' 'status=FAILED' |
sed 's/FAILED/ERROR/'
```

Expected:

```text
status=ERROR
```

---

# 18.7 Regular expression

Ví dụ log:

```text
2026-10-19T10:15:00 ERROR httpd failed
```

Filter lines bắt đầu bằng date-like value:

```bash
grep -E '^[0-9]{4}-[0-9]{2}-[0-9]{2}' logfile
```

Nhớ:

```text
regex match
≠
semantic validation
```

---

# Log Evidence

```bash
sudo journalctl -u httpd --since "-1 hour" --no-pager \
  > "$EVIDENCE/logs/httpd-journal.txt"

sudo journalctl -p warning --since "-1 hour" --no-pager \
  > "$EVIDENCE/logs/warnings.txt"
```

---

# LAB 19 — Log Rotation Inspection

Assignment không bắt buộc bạn phải thay log-rotation policy, nhưng syllabus yêu cầu hiểu log rotation.

## Read-only inspection

```bash
cat /etc/logrotate.conf
```

List:

```bash
ls -l /etc/logrotate.d/
```

Find HTTP config if present:

```bash
ls /etc/logrotate.d/ | grep -i http
```

---

## Debug configuration

Nếu supported:

```bash
sudo logrotate --debug /etc/logrotate.conf
```

`--debug` phục vụ inspection và không nên được hiểu là actual rotation execution.

Không force rotation trên production chỉ để “test”.

---

# LAB 20 — Bash Health-Check Script

Đây là phần trọng tâm của Assignment.

## Objective

Tạo script kiểm tra:

```text
hostname
timestamp
uptime
memory
filesystem capacity
inode capacity
httpd service
HTTP response
SELinux
firewalld
default route
```

và trả exit code có ý nghĩa.

---

# 20.1 Script design

Chúng ta dùng:

```text
0 = healthy
1 = warning/one or more checks failed
```

Trong enterprise có thể định nghĩa nhiều exit code chi tiết hơn.

---

# 20.2 Create script

```bash
sudo vi /usr/local/bin/day2_health_check.sh
```

Nội dung:

```bash
#!/usr/bin/env bash

# Day 2 Linux/RHEL health-check lab.
# Lab/non-production educational script.

DISK_WARN=80
MOUNT_POINT="/mnt/day2data"
SERVICE="httpd"
URL="http://127.0.0.1/"

overall_rc=0

timestamp() {
    date --iso-8601=seconds
}

ok() {
    printf '%s OK: %s\n' "$(timestamp)" "$*"
}

warn() {
    printf '%s WARNING: %s\n' "$(timestamp)" "$*" >&2
    overall_rc=1
}

echo "===== Day 2 Health Check ====="
echo "Timestamp : $(timestamp)"
echo "Hostname  : $(hostname)"
echo

echo "--- Uptime ---"
uptime

echo
echo "--- Memory ---"
free -h

echo
echo "--- Root Filesystem ---"
df -h /

root_used=$(df -P / | awk 'NR==2 {gsub("%","",$5); print $5}')

if [[ "$root_used" -ge "$DISK_WARN" ]]; then
    warn "Root filesystem usage is ${root_used}%"
else
    ok "Root filesystem usage is ${root_used}%"
fi

echo
echo "--- Inodes ---"
df -i /

echo
echo "--- Lab Filesystem ---"

if mountpoint -q "$MOUNT_POINT"; then
    df -h "$MOUNT_POINT"
    df -i "$MOUNT_POINT"

    lab_used=$(df -P "$MOUNT_POINT" |
        awk 'NR==2 {gsub("%","",$5); print $5}')

    if [[ "$lab_used" -ge "$DISK_WARN" ]]; then
        warn "$MOUNT_POINT usage is ${lab_used}%"
    else
        ok "$MOUNT_POINT usage is ${lab_used}%"
    fi
else
    warn "$MOUNT_POINT is not mounted"
fi

echo
echo "--- Service ---"

if systemctl is-active --quiet "$SERVICE"; then
    ok "$SERVICE is active"
else
    warn "$SERVICE is not active"
fi

echo
echo "--- HTTP Functional Test ---"

if curl --fail --silent --show-error --max-time 5 "$URL" >/dev/null; then
    ok "$URL responded successfully"
else
    warn "$URL failed"
fi

echo
echo "--- SELinux ---"

selinux_mode=$(getenforce 2>/dev/null || true)

if [[ "$selinux_mode" == "Enforcing" ]]; then
    ok "SELinux is Enforcing"
else
    warn "SELinux state is $selinux_mode"
fi

echo
echo "--- Firewall ---"

if systemctl is-active --quiet firewalld; then
    ok "firewalld is active"
else
    warn "firewalld is not active"
fi

echo
echo "--- Default Route ---"

if ip route show default | grep -q '^default'; then
    ip route show default
    ok "Default route exists"
else
    warn "Default route is missing"
fi

echo
echo "===== Final Result ====="

if [[ "$overall_rc" -eq 0 ]]; then
    echo "HEALTHY"
else
    echo "WARNING"
fi

exit "$overall_rc"
```

---

# 20.3 Permissions

```bash
sudo chown root:root /usr/local/bin/day2_health_check.sh
sudo chmod 0755 /usr/local/bin/day2_health_check.sh
```

Verify:

```bash
ls -l /usr/local/bin/day2_health_check.sh
```

---

# 20.4 Syntax validation

```bash
bash -n /usr/local/bin/day2_health_check.sh
```

No output normally means syntax parse succeeded.

---

# 20.5 Execute

```bash
/usr/local/bin/day2_health_check.sh
```

---

# 20.6 Exit code

Immediately:

```bash
echo $?
```

Expected healthy:

```text
0
```

If warning:

```text
1
```

---

# 20.7 Capture report

```bash
/usr/local/bin/day2_health_check.sh \
  > "$EVIDENCE/scripts/health-check-output.txt" 2>&1

echo $? | tee "$EVIDENCE/scripts/health-check-exit-code.txt"
```

---

# Why this script is better than a simple command list

Nó có:

```text
explicit configuration
functions
threshold
conditional logic
command verification
functional HTTP check
exit-code handling
timestamps
stderr warnings
```

Đây chính là operational scripting mindset.

---

# LAB 21 — Health Script Failure Validation

Một script chưa được test negative cases thì chưa đáng tin.

## Failure A — stop `httpd`

Lab only:

```bash
sudo systemctl stop httpd
```

Run:

```bash
/usr/local/bin/day2_health_check.sh
echo $?
```

Expected:

```text
WARNING: httpd is not active
WARNING: HTTP test failed
exit code = 1
```

Recover:

```bash
sudo systemctl start httpd
```

---

## Failure B — unmount lab filesystem

Trước hết đảm bảo không process sử dụng mount:

```bash
sudo umount /mnt/day2data
```

Run health check.

Expected:

```text
WARNING: /mnt/day2data is not mounted
```

Recover:

```bash
sudo mount /mnt/day2data
```

Verify:

```bash
findmnt /mnt/day2data
```

---

# LAB 22 — Bash Backup Script

## Objective

Backup controlled data với:

```text
validation
timestamp
tar
checksum
exit codes
logging
```

---

# 22.1 Create source data

```bash
mkdir -p ~/day2-source
```

```bash
printf '%s\n' \
  'Day 2 backup test file 1' \
  > ~/day2-source/file1.txt

printf '%s\n' \
  'Day 2 backup test file 2' \
  > ~/day2-source/file2.txt
```

Verify:

```bash
find ~/day2-source -type f -maxdepth 1 -print
```

---

# 22.2 Prepare backup destination

```bash
sudo mkdir -p /mnt/day2data/backups
sudo chown "$USER":"$USER" /mnt/day2data/backups
```

---

# 22.3 Backup script

Create:

```bash
sudo vi /usr/local/bin/day2_backup.sh
```

Content:

```bash
#!/usr/bin/env bash

SOURCE="${HOME}/day2-source"
BACKUP_MOUNT="/mnt/day2data"
BACKUP_DIR="${BACKUP_MOUNT}/backups"
TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
ARCHIVE="${BACKUP_DIR}/day2_backup_${TIMESTAMP}.tar.gz"
CHECKSUM="${ARCHIVE}.sha256"

log() {
    printf '%s %s\n' "$(date --iso-8601=seconds)" "$*"
}

fail() {
    local rc="$1"
    shift
    log "ERROR: $*" >&2
    exit "$rc"
}

log "Starting backup"

if [[ ! -d "$SOURCE" ]]; then
    fail 10 "Source directory does not exist: $SOURCE"
fi

if ! mountpoint -q "$BACKUP_MOUNT"; then
    fail 20 "Backup filesystem is not mounted: $BACKUP_MOUNT"
fi

if [[ ! -d "$BACKUP_DIR" ]]; then
    fail 21 "Backup directory does not exist: $BACKUP_DIR"
fi

if ! tar -czf "$ARCHIVE" -C "$(dirname "$SOURCE")" "$(basename "$SOURCE")"; then
    fail 30 "tar backup failed"
fi

if [[ ! -s "$ARCHIVE" ]]; then
    fail 31 "Backup archive is missing or empty"
fi

if ! sha256sum "$ARCHIVE" > "$CHECKSUM"; then
    fail 40 "Checksum generation failed"
fi

log "Backup created: $ARCHIVE"
log "Checksum created: $CHECKSUM"
log "Backup completed successfully"

exit 0
```

---

# 22.4 Why explicit exit codes?

Ở lab này:

```text
10 = source missing
20 = backup filesystem not mounted
21 = backup directory missing
30 = tar failure
31 = archive invalid/empty
40 = checksum failure
0  = success
```

Monitoring hoặc operator có thể biết loại failure nhanh hơn.

---

# 22.5 Permissions

```bash
sudo chown root:root /usr/local/bin/day2_backup.sh
sudo chmod 0755 /usr/local/bin/day2_backup.sh
```

---

# 22.6 Syntax check

```bash
bash -n /usr/local/bin/day2_backup.sh
```

---

# 22.7 Run

```bash
/usr/local/bin/day2_backup.sh
```

Check:

```bash
echo $?
```

Expected:

```text
0
```

---

# 22.8 Verify artifacts

```bash
ls -lh /mnt/day2data/backups/
```

Expected:

```text
day2_backup_YYYYMMDD_HHMMSS.tar.gz
day2_backup_YYYYMMDD_HHMMSS.tar.gz.sha256
```

---

# 22.9 Inspect archive without extracting

```bash
tar -tzf /mnt/day2data/backups/day2_backup_*.tar.gz
```

Bạn cần confirm:

```text
day2-source/
day2-source/file1.txt
day2-source/file2.txt
```

---

# 22.10 Check checksum

Chọn archive cụ thể:

```bash
LATEST=$(ls -1t /mnt/day2data/backups/day2_backup_*.tar.gz | head -1)
echo "$LATEST"
```

Checksum file:

```bash
sha256sum -c "${LATEST}.sha256"
```

Expected:

```text
OK
```

RHEL documentation vẫn liệt kê `tar` và `rsync` trong các filesystem backup workflows. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/pdf/managing_file_systems/red_hat_enterprise_linux-9-managing_file_systems-en-us.pdf?utm_source=chatgpt.com)

---

# LAB 23 — `rsync`

## Objective

Thực hành file synchronization riêng biệt với archive backup.

---

# Create destination

```bash
mkdir -p ~/day2-rsync-copy
```

---

# Dry run first

```bash
rsync -av --dry-run ~/day2-source/ ~/day2-rsync-copy/
```

Hãy xem file nào sẽ được transferred.

Đây là habit enterprise rất tốt:

```text
preview
→ review
→ execute
```

---

# Execute

```bash
rsync -av ~/day2-source/ ~/day2-rsync-copy/
```

Verify:

```bash
find ~/day2-rsync-copy -type f -print
```

---

# Compare

```bash
diff -r ~/day2-source ~/day2-rsync-copy
```

No output:

```text
no difference detected
```

theo scope của `diff`.

---

# Important trailing slash lesson

```bash
rsync source/ destination/
```

khác về resulting directory structure so với một số forms dùng:

```bash
rsync source destination/
```

Do đó:

```text
Always know what directory itself vs its contents mean.
```

---

# LAB 24 — Restore Procedure

Syllabus yêu cầu phải validate backup **và restore**, không chỉ tạo file backup. Infrastructure_Operations_14_Da…

## Objective

Thực hiện restore vào isolated directory rồi kiểm tra integrity.

---

# 24.1 Identify correct backup

```bash
LATEST=$(ls -1t /mnt/day2data/backups/day2_backup_*.tar.gz | head -1)
echo "$LATEST"
```

Không được restore chỉ dựa vào:

```text
looks like newest
```

trong production.

Bạn cần:

```text
timestamp
backup ID
change/ticket
expected backup set
```

---

# 24.2 Verify checksum before restore

```bash
sha256sum -c "${LATEST}.sha256"
```

Expected:

```text
OK
```

Nếu FAIL:

```text
STOP
do not trust archive
investigate/escalate
```

---

# 24.3 Create isolated restore area

```bash
rm -rf ~/day2-restore-test
mkdir -p ~/day2-restore-test
```

Đây chỉ là lab-owned directory.

---

# 24.4 Extract

```bash
tar -xzf "$LATEST" -C ~/day2-restore-test
```

---

# 24.5 Inspect

```bash
find ~/day2-restore-test -type f -print
```

---

# 24.6 Compare

```bash
diff -r ~/day2-source ~/day2-restore-test/day2-source
```

Expected:

```text
no output
```

---

# 24.7 Hash-level comparison

Source:

```bash
find ~/day2-source -type f -exec sha256sum {} \; | sort
```

Restore:

```bash
find ~/day2-restore-test/day2-source -type f -exec sha256sum {} \; | sort
```

Paths differ, nên nếu cần exact automated comparison bạn có thể compare content individually hoặc normalize output.

---

# Evidence

Lưu:

```text
archive name
archive timestamp
checksum result
restore location
file count
diff result
restore start/end timestamp
```

Ví dụ:

```bash
diff -r ~/day2-source ~/day2-restore-test/day2-source \
  > "$EVIDENCE/backup/restore-diff.txt"

echo $? > "$EVIDENCE/backup/restore-diff-exit-code.txt"
```

`diff`:

```text
0 = same
1 = differences
>1 = error
```

Đây là ví dụ vì sao không được mặc định:

```text
non-zero = cùng một meaning
```

cho mọi command.

---

# LAB 25 — Backup Failure Injection

## Failure scenario

Unmount backup filesystem.

```bash
sudo umount /mnt/day2data
```

Run:

```bash
/usr/local/bin/day2_backup.sh
```

Expected:

```text
ERROR: Backup filesystem is not mounted
```

Check:

```bash
echo $?
```

Expected theo script:

```text
20
```

Điểm rất quan trọng:

Script **không được vô tình tạo backup vào directory `/mnt/day2data` trên root filesystem** khi intended disk bị unmount.

Đó chính là lý do chúng ta dùng:

```bash
mountpoint -q "$BACKUP_MOUNT"
```

---

# Recover

```bash
sudo mount /mnt/day2data
```

Verify:

```bash
findmnt /mnt/day2data
```

Retry:

```bash
/usr/local/bin/day2_backup.sh
echo $?
```

Expected:

```text
0
```

---

# LAB 26 — Cron Scheduling

## Objective

Schedule health-check và prove job actually executed.

---

# 26.1 Verify cron package

```bash
rpm -q cronie
```

---

# 26.2 Verify daemon

```bash
systemctl status crond --no-pager
```

Nếu lab VM chưa chạy `crond`:

```bash
sudo systemctl enable --now crond
```

Verify:

```bash
systemctl is-active crond
systemctl is-enabled crond
```

---

# 26.3 Create log directory

```bash
mkdir -p ~/day2-cron
```

---

# 26.4 Edit user crontab

```bash
crontab -e
```

Thêm:

```text
*/5 * * * * /usr/local/bin/day2_health_check.sh >> /home/<YOUR_USER>/day2-cron/health.log 2>&1
```

**Thay `<YOUR_USER>` bằng actual username.**

Dùng absolute path.

Không dùng:

```text
~/day2-cron
```

trong production-style cron khi có thể dùng explicit path.

---

# 26.5 Understand expression

```text
*/5 * * * *
 │  │ │ │ │
 │  │ │ │ └── every day of week
 │  │ │ └──── every month
 │  │ └────── every day of month
 │  └──────── every hour
 └─────────── every 5 minutes
```

---

# 26.6 Verify schedule

```bash
crontab -l
```

Điều này chỉ chứng minh:

```text
entry exists
```

không chứng minh execution.

---

# 26.7 Verify actual execution

Sau execution window, kiểm tra:

```bash
tail -50 ~/day2-cron/health.log
```

Bạn cần thấy:

```text
Timestamp
health result
```

---

# 26.8 Check daemon logs

Tùy configuration:

```bash
sudo journalctl -u crond --since "-15 min" --no-pager
```

---

# Cron evidence

```bash
crontab -l > "$EVIDENCE/scripts/crontab.txt"

cp ~/day2-cron/health.log \
   "$EVIDENCE/scripts/cron-health.log"
```

---

# Cron troubleshooting

Nếu manual script:

```bash
/usr/local/bin/day2_health_check.sh
```

works nhưng cron không chạy:

investigate:

```text
1. Is crond running?
2. Is crontab syntax correct?
3. Is command path absolute?
4. Can user execute script?
5. Can user write output path?
6. Environment/PATH differences?
7. SELinux?
8. cron log/journal?
```

Không bắt đầu bằng:

```text
chmod 777 everything
```

---

# LAB 27 — Failure Injection 1: Service Failure

Đã thực hiện nhưng giờ ghi theo incident methodology.

## Inject

```text
Invalid httpd config
```

## Symptom

```text
HTTP unavailable
httpd failed
```

## Evidence

```text
systemctl status
journalctl
httpd -t
ss
curl
```

## Root cause

```text
invalid directive
```

## Recovery

```text
remove invalid include
config test
start service
```

## Verify

```text
active
port 80
HTTP response
```

## Escalate if

```text
configuration source unknown
application config owner unclear
recurring failure
security-impacting config
unknown dependency
```

---

# LAB 28 — Failure Injection 2: Firewall / Network Access

## Objective

Application remains healthy locally but remote client cannot reach it.

Đây là rất useful vì nó dạy:

```text
service healthy
≠
network-accessible
```

---

# Baseline

Server:

```bash
systemctl is-active httpd
sudo ss -lntp | grep ':80'
curl http://127.0.0.1/
```

Client:

```bash
curl http://192.168.56.10/
```

Tất cả phải work.

---

# Inject firewall failure

**Không remove SSH.**

```bash
sudo firewall-cmd --remove-service=http
```

Chỉ runtime nếu không dùng `--permanent`.

---

# Verify server locally

```bash
curl http://127.0.0.1/
```

Có thể vẫn work.

---

# Client test

```bash
curl --max-time 5 http://192.168.56.10/
```

Expected remote failure tùy topology.

---

# Troubleshooting reasoning

Server:

```bash
systemctl is-active httpd
```

→ active.

```bash
sudo ss -lntp | grep ':80'
```

→ listening.

```bash
curl http://127.0.0.1/
```

→ application works.

```bash
sudo firewall-cmd --query-service=http
```

→ no.

Conclusion:

```text
Application healthy
+
listener exists
+
local functional test passes
+
remote test fails
+
firewall rule absent
=
host firewall probable root cause
```

---

# Recover

```bash
sudo firewall-cmd --add-service=http
```

Client retest.

---

# Evidence

Before:

```bash
sudo firewall-cmd --list-all \
  > "$EVIDENCE/failures/firewall-failure.txt"
```

After recovery:

```bash
sudo firewall-cmd --list-all \
  > "$EVIDENCE/failures/firewall-recovered.txt"
```

---

# LAB 29 — Failure Injection 3: SELinux

Already performed:

```text
wrong file context
```

Troubleshooting pattern:

```text
HTTP service active
        ↓
port listening
        ↓
file UNIX permission appears usable
        ↓
specific resource gets denied
        ↓
AVC evidence
        ↓
incorrect SELinux context
        ↓
restorecon
        ↓
functional success
```

This failure teaches:

```text
chmod is not the universal answer to Permission denied.
```

---

# LAB 30 — Failure Injection 4: DNS / Name Resolution

Ta sẽ tạo lỗi local và reversible.

## Safety

Không sửa corporate DNS.

Chỉ dùng `/etc/hosts` lab entry.

---

# 30.1 Baseline

Add correct test name:

```bash
SERVER_IP=$(hostname -I | awk '{print $1}')

echo "$SERVER_IP day2app.lab" | \
sudo tee -a /etc/hosts
```

Verify:

```bash
getent hosts day2app.lab
```

HTTP:

```bash
curl http://day2app.lab/
```

---

# 30.2 Inject wrong mapping

Backup first:

```bash
sudo cp -a /etc/hosts /etc/hosts.day2-backup
```

Thay lab line sao cho:

```text
192.0.2.123 day2app.lab
```

`192.0.2.0/24` là documentation network range, nhưng vẫn chỉ dùng trong isolated lab scenario.

Verify:

```bash
getent hosts day2app.lab
```

Expected:

```text
192.0.2.123 ...
```

---

# Symptom

```bash
curl --max-time 5 http://day2app.lab/
```

fails.

Nhưng:

```bash
curl "http://$SERVER_IP/"
```

works.

Reasoning:

```text
HTTP by IP works
HTTP by name fails
name resolves to wrong IP
→ name resolution problem
```

---

# Recover

```bash
sudo cp -a /etc/hosts.day2-backup /etc/hosts
```

Verify:

```bash
getent hosts day2app.lab
```

Nếu backup được tạo trước correct lab entry, bạn có thể re-add correct mapping hoặc cleanup hoàn toàn.

---

# LAB 31 — Failure Injection 5: Storage

Có hai safe variants.

## Variant A — Missing mount

Unmount:

```bash
sudo umount /mnt/day2data
```

Run backup:

```bash
/usr/local/bin/day2_backup.sh
```

Expected:

```text
exit code 20
```

Recovery:

```bash
sudo mount /mnt/day2data
```

---

## Variant B — High utilization

Chỉ dedicated filesystem:

```bash
sudo fallocate -l 1500M /mnt/day2data/day2-fill.bin
```

Run:

```bash
/usr/local/bin/day2_health_check.sh
```

Expected nếu threshold exceeded:

```text
WARNING: /mnt/day2data usage is ...
```

Cleanup:

```bash
sudo rm /mnt/day2data/day2-fill.bin
```

Re-run:

```bash
/usr/local/bin/day2_health_check.sh
echo $?
```

---

# 32. Integrated Troubleshooting Drill

Bây giờ giả sử mentor không nói lỗi nào đã được inject.

Bạn chỉ nhận:

```text
Incident INC-DAY2-001

User reports:
"Day2 web service cannot be accessed."

Start time:
10:25 local time
```

Bạn phải điều tra theo flow sau.

---

## Phase 1 — Confirm

```bash
date --iso-8601=seconds
hostname
```

Test:

```bash
curl --max-time 5 http://day2app.lab/
```

Record exact error.

Ví dụ:

```text
Could not resolve host
Connection refused
Connection timed out
403
500
```

Mỗi error dẫn investigation khác nhau.

---

# Phase 2 — Resolve name

```bash
getent hosts day2app.lab
```

Question:

```text
Does name resolve?
Does it resolve to expected IP?
```

---

# Phase 3 — IP state

```bash
ip -br addr
ip route
```

---

# Phase 4 — Service

```bash
systemctl status httpd --no-pager
```

---

# Phase 5 — Process

```bash
pgrep -a httpd
```

---

# Phase 6 — Listener

```bash
sudo ss -lntp | grep ':80'
```

---

# Phase 7 — Local functional test

```bash
curl -v http://127.0.0.1/
```

---

# Phase 8 — Firewall

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

---

# Phase 9 — SELinux

```bash
getenforce
ls -lZ /var/www/html/
sudo ausearch -m AVC -ts recent
```

---

# Phase 10 — Storage

```bash
findmnt /mnt/day2data
df -hT
df -i
```

---

# Phase 11 — Logs

```bash
sudo journalctl -u httpd --since "-30 min" --no-pager
```

---

# Phase 12 — Determine root cause

Không viết:

```text
"server problem"
```

Root cause phải specific.

Ví dụ:

```text
HTTP traffic was blocked because the http service
was absent from the active firewalld zone.
```

Hoặc:

```text
httpd failed to start because an invalid Apache
configuration directive was present in
/etc/httpd/conf.d/day2-broken.conf.
```

---

# 33. Evidence Collection Standard

Syllabus yêu cầu collect diagnostic evidence và tạo troubleshooting/escalation record. Infrastructure_Operations_14_Da…

Một evidence package tốt nên có structure:

```text
day2-evidence/
├── baseline/
├── packages/
├── processes/
├── service/
├── network/
├── security/
├── storage/
├── logs/
├── scripts/
├── backup/
└── failures/
```

---

# 34. Evidence Metadata

Mỗi evidence set nên gắn ít nhất:

```text
Hostname
Environment
Timestamp
Timezone
Operator
Ticket/reference
Command
Purpose
Result
```

Ví dụ:

```text
Host: rhel-day2-01
Environment: LAB
Timestamp: 2026-10-19T11:30:00+07:00
Reference: INC-DAY2-001
Check: HTTP listener verification
Command: ss -lntp
Result: TCP/80 LISTEN by httpd
```

---

# 35. Evidence Integrity

Sau khi hoàn thành:

```bash
find ~/day2-evidence -type f -exec sha256sum {} \; \
  > ~/day2-evidence/SHA256SUMS.txt
```

Lưu ý:

Không chạy lệnh lại sao cho `SHA256SUMS.txt` tự hash chính nó theo cách gây confusion.

Có thể archive:

```bash
tar -czf ~/day2-evidence.tar.gz \
    -C "$HOME" day2-evidence
```

Then:

```bash
sha256sum ~/day2-evidence.tar.gz \
  > ~/day2-evidence.tar.gz.sha256
```

---

# 36. Linux Troubleshooting Record

Đây là artifact cuối cùng syllabus yêu cầu.

Bạn nên ghi theo format:

| Field             | Nội dung                       |
| ----------------- | ------------------------------ |
| Ticket ID         | `INC-DAY2-001`                 |
| Environment       | LAB                            |
| Hostname          | `rhel-day2-01`                 |
| Start time        | timestamp + timezone           |
| Reported symptom  | User cannot access web service |
| Scope             | One lab server                 |
| Recent change     | If known                       |
| Initial checks    | IP, route, DNS, service, port  |
| Evidence          | Paths/commands/logs            |
| Root cause        | Exact technical root cause     |
| Corrective action | Exact approved action          |
| Verification      | Service, process, port, HTTP   |
| Rollback          | Performed/not required         |
| Residual risk     | Any unresolved issue           |
| Escalation        | Team/person if required        |
| End time          | timestamp                      |
| Status            | Resolved / Escalated           |

---

# 37. Example completed troubleshooting record

```text
Ticket:
INC-DAY2-001

Environment:
LAB

Affected host:
rhel-day2-01

Reported symptom:
Remote client unable to reach HTTP service.

Initial observations:
- Host IP reachable.
- httpd active.
- httpd processes running.
- TCP/80 listening.
- Local curl to 127.0.0.1 successful.
- Remote curl timed out.

Investigation:
Reviewed active firewalld zone and found HTTP service
was not permitted.

Root cause:
HTTP service had been removed from the runtime
firewalld configuration during controlled failure injection.

Corrective action:
Restored HTTP permission in the approved active zone.

Verification:
- firewalld query-service=http returned yes.
- TCP/80 remained listening.
- Remote client successfully retrieved test page.
- httpd journal showed no service failure.

Rollback:
Not required; correction restored documented baseline.

Residual risk:
None in lab environment.

Escalation:
Not required.

Status:
Resolved.
```

---

# 38. Escalation Boundary trong Lab

Bạn cần luyện cả việc **không sửa** khi vượt scope.

## Có thể xử lý trong Day 2 lab

```text
Known package missing
Known service stopped
Known syntax error
Known firewall service missing
Known SELinux label mismatch
Known fstab entry error
Lab filesystem high usage
Cron permission/path problem
Backup source/destination problem
```

---

## Dừng và escalate nếu gặp

```text
Kernel panic
Repeated filesystem corruption
Disk I/O errors
Unknown storage hardware failure
Unexplained data loss
Security compromise suspicion
Unknown SELinux policy requirement
Production firewall change outside authorization
Network route owned by Network team
Application-specific deep failure
Repeated OOM killing critical services
Unknown package/source repository requirement
Production filesystem repair
Privileged operation outside runbook
```

Một Infrastructure Operator giỏi không phải người “sửa được mọi thứ”.

Họ biết:

```text
what is safe
what is authorized
what evidence is needed
when ownership changes
```

---

# 39. Full Verification Matrix

Sau lab, trạng thái chuẩn nên được chứng minh bằng nhiều layer:

| Layer            | Verification                     |
| ---------------- | -------------------------------- |
| OS               | `cat /etc/os-release`            |
| Packages         | `rpm -q ...`                     |
| Repository       | `dnf repolist`                   |
| Environment      | `env`, `$PATH`                   |
| Process          | `ps`, `pgrep`                    |
| Resources        | `uptime`, `free`, `df`           |
| Service          | `systemctl is-active httpd`      |
| Startup          | `systemctl is-enabled httpd`     |
| Listener         | `ss -lntp`                       |
| IP               | `ip -br addr`                    |
| Routing          | `ip route`                       |
| DNS              | `getent hosts`                   |
| HTTP             | `curl`                           |
| Firewall         | `firewall-cmd`                   |
| SELinux          | `getenforce`, `ls -Z`            |
| Storage          | `lsblk`, `findmnt`               |
| Persistent mount | `/etc/fstab` + remount test      |
| Capacity         | `df -h`                          |
| Inodes           | `df -i`                          |
| Logs             | `journalctl`                     |
| Script           | execution + exit code            |
| Cron             | crontab + actual output          |
| Backup           | archive + checksum               |
| Restore          | isolated extraction + comparison |
| Evidence         | timestamps + hashes              |
| Troubleshooting  | root cause + verification        |

---

# 40. Cleanup / Rollback

Sau khi lab hoàn thành, bạn phải biết cách đưa environment về trạng thái mong muốn.

## Cron

```bash
crontab -e
```

Remove Day 2 lab entry.

Verify:

```bash
crontab -l
```

---

## Test data

```bash
rm -rf ~/day2-source
rm -rf ~/day2-rsync-copy
rm -rf ~/day2-restore-test
rm -rf ~/day2-cron
```

Chỉ remove directories bạn tạo cho lab.

---

## Scripts

Nếu không cần giữ:

```bash
sudo rm /usr/local/bin/day2_health_check.sh
sudo rm /usr/local/bin/day2_backup.sh
```

---

## Web test files

```bash
sudo rm -f /var/www/html/index.html
sudo rm -f /var/www/html/day2-selinux.html
```

Nếu files pre-existed before lab, **không xóa**; restore từ pre-lab copy.

---

# Firewall rollback

Nếu bạn đã thêm HTTP chỉ vì lab:

```bash
sudo firewall-cmd --permanent --remove-service=http
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --query-service=http
```

---

# Service rollback

Nếu `httpd` không nên tồn tại sau lab:

```bash
sudo systemctl disable --now httpd
```

Nếu package cũng chỉ phục vụ lab và policy cho phép remove:

```bash
sudo dnf remove httpd
```

Không remove dependencies/package tùy tiện trên shared server.

---

# Storage cleanup

Đây là high-risk section.

Đầu tiên:

```bash
findmnt /mnt/day2data
```

Unmount:

```bash
sudo umount /mnt/day2data
```

Restore fstab:

```bash
sudo cp -a /etc/fstab.day2-backup /etc/fstab
sudo systemctl daemon-reload
```

Verify:

```bash
grep day2data /etc/fstab
```

không còn lab entry.

Không cần wipe disk nếu lab VM sẽ reuse; snapshot revert thường an toàn hơn.

---

# 41. Final Failure-Injection Matrix

| Failure                | Symptom              | Key Evidence                          | Root Cause Layer    | Recovery                |
| ---------------------- | -------------------- | ------------------------------------- | ------------------- | ----------------------- |
| Invalid `httpd` config | Service failed       | `systemctl`, `journalctl`, `httpd -t` | Service/config      | Remove bad config       |
| HTTP firewall removed  | Remote access fails  | `ss`, local `curl`, `firewall-cmd`    | Firewall            | Restore allowed service |
| Wrong SELinux context  | 403/access denied    | `ls -Z`, AVC                          | SELinux             | `restorecon`            |
| Wrong name resolution  | Name fails, IP works | `getent`, `curl`                      | DNS/name resolution | Correct mapping         |
| Backup mount absent    | Backup script aborts | `findmnt`, exit code 20               | Storage             | Remount                 |
| High filesystem usage  | Health warning       | `df -h`                               | Capacity            | Remove lab filler       |

---

# 42. Day 2 Practical Troubleshooting Logic

Sau toàn bộ lab, bạn nên tự động suy luận như sau:

```text
Application unavailable
        │
        ▼
Can name resolve?
        │
        ├── no → DNS/name resolution
        │
        └── yes
             ↓
Is IP/network path valid?
             │
             ├── no → interface/route/network
             │
             └── yes
                  ↓
Is process/service active?
                  │
                  ├── no
                  │    ↓
                  │ status → logs → config → dependency
                  │
                  └── yes
                       ↓
Is port listening?
                       │
                       ├── no → bind/application
                       │
                       └── yes
                            ↓
Does local curl work?
                            │
                  ┌─────────┴─────────┐
                  │                   │
                 no                  yes
                  │                   │
         application/SELinux     firewall/network
         storage/dependency
```

Storage branch:

```text
Application cannot write
        │
        ▼
Correct filesystem mounted?
        │
        ├── no → mount/fstab/device
        │
        └── yes
             ↓
Blocks available?
        ┌────┴─────┐
       no          yes
       │            │
capacity issue   inodes available?
                     │
              ┌──────┴──────┐
             no             yes
             │               │
        inode issue      permissions/
                         SELinux/
                         application
```

Backup branch:

```text
Backup reported failure
        │
        ▼
Source exists?
        │
        ▼
Destination mounted?
        │
        ▼
Space available?
        │
        ▼
tar/rsync succeeded?
        │
        ▼
Artifact exists?
        │
        ▼
Checksum valid?
        │
        ▼
Can restore?
        │
        ▼
Restored content matches?
```

Chỉ khi đến cuối chain này mới có căn cứ nói:

```text
Backup and basic recovery validated.
```

---

# 43. Day 2 Practical Evidence Package — Deliverable

Sau Day 2, một submission tốt nên có:

```text
Day2_Submission/
│
├── 01_System_Baseline/
│   ├── os-release.txt
│   ├── kernel.txt
│   ├── cpu.txt
│   ├── memory.txt
│   └── timestamp.txt
│
├── 02_Package/
│   ├── repositories.txt
│   └── installed-packages.txt
│
├── 03_Process_Service/
│   ├── process-snapshot.txt
│   ├── httpd-before-failure.txt
│   ├── httpd-failure.txt
│   ├── httpd-recovery-journal.txt
│   └── httpd-after-recovery.txt
│
├── 04_Network/
│   ├── addresses.txt
│   ├── routes.txt
│   ├── dns.txt
│   └── listeners.txt
│
├── 05_Security/
│   ├── firewall-runtime.txt
│   ├── firewall-permanent.txt
│   ├── selinux-mode.txt
│   └── selinux-avc.txt
│
├── 06_Storage/
│   ├── lsblk.txt
│   ├── mount.txt
│   ├── capacity.txt
│   └── inode-capacity.txt
│
├── 07_Logs/
│   ├── httpd-journal.txt
│   └── warnings.txt
│
├── 08_Automation/
│   ├── day2_health_check.sh
│   ├── health-check-output.txt
│   ├── day2_backup.sh
│   ├── crontab.txt
│   └── cron-health.log
│
├── 09_Backup_Restore/
│   ├── backup-list.txt
│   ├── checksum-result.txt
│   └── restore-diff.txt
│
├── 10_Failure_Injection/
│   ├── service-failure.txt
│   ├── firewall-failure.txt
│   ├── selinux-failure.txt
│   ├── dns-failure.txt
│   └── storage-failure.txt
│
├── 11_Troubleshooting_Record/
│   └── INC-DAY2-001.txt
│
└── SHA256SUMS.txt
```

Đây mới là một **operational evidence package**, thay vì chỉ một loạt screenshots.

---

# 44. Day 2 Practical Completion Checklist

Bạn chỉ nên xem Module Thực hành Day 2 hoàn thành khi có thể đánh dấu toàn bộ:

- [ ] Xác định đúng hostname, RHEL release, kernel và baseline.
- [ ] Kiểm tra configured repositories.
- [ ] Cài approved packages bằng `dnf`.
- [ ] Verify package bằng `rpm`.
- [ ] Hiểu package → files relationship.
- [ ] Inspect environment variables và `PATH`.
- [ ] Chứng minh exported variable inheritance.
- [ ] Inspect process, PID, PPID, state, CPU và memory.
- [ ] Thực hiện safe `SIGTERM` experiment.
- [ ] Start/stop/restart/enable `systemd` service.
- [ ] Verify service bằng `systemctl`.
- [ ] Verify process bằng `pgrep`/`ps`.
- [ ] Verify listener bằng `ss`.
- [ ] Verify application bằng `curl`.
- [ ] Deliberately fail service.
- [ ] Diagnose bằng `systemctl`, `journalctl` và config validation.
- [ ] Recover service và collect before/after evidence.
- [ ] Verify interface/IP bằng `ip`.
- [ ] Verify routing table/default route.
- [ ] Verify DNS/name resolution.
- [ ] Hiểu `ping` limitation.
- [ ] Sử dụng `tracepath`/`traceroute` phù hợp.
- [ ] Inspect `firewalld` zone.
- [ ] Phân biệt runtime và permanent firewall configuration.
- [ ] Allow/test HTTP traffic.
- [ ] Inject và troubleshoot firewall failure.
- [ ] Verify SELinux mode.
- [ ] Inspect SELinux context.
- [ ] Inject controlled context error.
- [ ] Find AVC evidence.
- [ ] Correct bằng `restorecon`.
- [ ] Không dùng disable SELinux như solution.
- [ ] Identify dedicated lab disk.
- [ ] Tạo controlled partition.
- [ ] Tạo filesystem.
- [ ] Mount filesystem.
- [ ] Verify bằng `findmnt`.
- [ ] Tạo persistent `/etc/fstab` configuration.
- [ ] Backup `/etc/fstab` trước change.
- [ ] Test mount trước reboot.
- [ ] Kiểm tra disk capacity bằng `df -h`.
- [ ] Kiểm tra inode capacity bằng `df -i`.
- [ ] Inject high-storage-utilization safely.
- [ ] Query system/service logs bằng `journalctl`.
- [ ] Filter logs bằng `grep`.
- [ ] Process structured output bằng `awk`.
- [ ] Hiểu `sed`/regex use cases.
- [ ] Inspect log rotation configuration.
- [ ] Viết Bash health-check script.
- [ ] Có meaningful exit codes.
- [ ] Test positive case.
- [ ] Test negative case.
- [ ] Viết Bash backup script.
- [ ] Validate backup filesystem trước khi ghi.
- [ ] Tạo `tar` archive.
- [ ] Tạo/check SHA-256 checksum.
- [ ] Thực hành `rsync`.
- [ ] Restore vào isolated location.
- [ ] Compare restored content.
- [ ] Schedule health check bằng cron.
- [ ] Verify cron execution thay vì chỉ verify crontab.
- [ ] Inject service failure.
- [ ] Inject firewall/network failure.
- [ ] Inject SELinux failure.
- [ ] Inject DNS failure.
- [ ] Inject storage failure.
- [ ] Thu thập evidence cho từng failure.
- [ ] Viết root cause cụ thể.
- [ ] Ghi corrective action.
- [ ] Ghi rollback.
- [ ] Xác định escalation boundary.
- [ ] Tạo final evidence package với timestamp/checksum.

---

# 45. Practical Cheat Sheet Day 2

```bash
# ---- SYSTEM ----
cat /etc/os-release
hostnamectl
uname -r
uptime
free -h


# ---- PACKAGE ----
dnf repolist
dnf info <package>
sudo dnf install <package>
rpm -q <package>
rpm -qi <package>
rpm -ql <package>
rpm -qf <file>


# ---- PROCESS ----
ps -ef
ps aux
pgrep -a <name>
top
kill -TERM <PID>


# ---- SYSTEMD ----
systemctl status <service>
systemctl is-active <service>
systemctl is-enabled <service>

sudo systemctl start <service>
sudo systemctl stop <service>
sudo systemctl restart <service>
sudo systemctl enable <service>

sudo systemctl daemon-reload


# ---- NETWORK ----
ip -br link
ip -br addr
ip route
ip route get <IP>

ping -c 3 <IP>
tracepath <host>

getent hosts <hostname>

ss -lntp

curl -v http://<host>/


# ---- FIREWALL ----
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all

sudo firewall-cmd --query-service=http
sudo firewall-cmd --add-service=http

sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload


# ---- SELINUX ----
getenforce
sestatus

ls -lZ <path>
matchpathcon -V <path>

sudo ausearch -m AVC -ts recent
sudo restorecon -v <path>


# ---- STORAGE ----
lsblk
lsblk -f
blkid

findmnt
findmnt <mountpoint>

df -hT
df -i
du -sh <directory>

sudo mount <device> <mountpoint>
sudo umount <mountpoint>


# ---- LOGS ----
journalctl
journalctl -b
journalctl -u <service>
journalctl --since "-15 min"
journalctl -p warning

grep
awk
sed


# ---- BASH ----
bash -n script.sh
./script.sh
echo $?


# ---- CRON ----
crontab -e
crontab -l

systemctl status crond
journalctl -u crond


# ---- BACKUP ----
tar -czf backup.tar.gz <source>
tar -tzf backup.tar.gz
tar -xzf backup.tar.gz -C <restore-path>

rsync -av --dry-run source/ destination/
rsync -av source/ destination/

sha256sum file
sha256sum -c file.sha256
```

---

# 46. Mental Checklist cuối cùng trước mọi Linux operational change

Trước khi gõ một configuration-changing command, hãy tự hỏi:

```text
1. Tôi đang ở đúng host chưa?
2. Production hay non-production?
3. Tôi có authorization không?
4. Current state đã được ghi lại chưa?
5. Command này thay đổi cái gì?
6. Business/service impact là gì?
7. Có dependency nào không?
8. Có backup/config copy không?
9. Rollback là gì?
10. Tôi sẽ verify bằng command nào?
11. Functional verification là gì?
12. Evidence nào phải giữ?
13. Khi failure xảy ra, điểm nào tôi phải escalate?
```

Nếu một Infrastructure Operations engineer hình thành được thói quen này, các command như `dnf`, `systemctl`, `firewall-cmd`, `mount`, `journalctl`, `tar` hay `rsync` chỉ còn là **tools**. Năng lực thật sự nằm ở việc sử dụng chúng **có kiểm soát, reversible, verifiable và auditable**.

Với toàn bộ Module Thực hành này, chúng ta đã cover đầy đủ các yêu cầu Assignment/Lab chính thức của Day 2: package installation/verification, process/environment inspection, `systemd` management và service recovery; network, DNS, firewall, SELinux và storage; log extraction/filtering; Bash health-check/backup scripting với exit-code handling; cron; backup/restore validation; service/network/storage failure injection; diagnostic evidence; troubleshooting và escalation record. Infrastructure_Operations_14_Da…
