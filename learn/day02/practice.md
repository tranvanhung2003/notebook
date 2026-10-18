# Day 2 — Module Thực hành

## Linux/RHEL Administration, Troubleshooting, Automation, Backup & Recovery Lab

Theo syllabus, **Assignment/Lab của Day 2** yêu cầu bạn phải thực hiện đủ các nhóm công việc sau: cài và verify approved packages trên RHEL; quản lý một `systemd service`; inspect processes và environment variables; recover một deliberately failed service; cấu hình/test network, DNS, firewall, SELinux-related access, disk mounts và storage capacity; extract/filter logs; xây Bash health-check và backup script có exit-code handling; schedule bằng `cron`; validate backup/restore; inject service/network/storage failures; collect diagnostic evidence; và cuối cùng tạo **Linux troubleshooting and escalation record**. Infrastructure_Operations_14_Da…

Phần dưới đây sẽ biến toàn bộ yêu cầu đó thành **một lab xuyên suốt**, chứ không phải hàng loạt command rời rạc.

Các command dùng syntax phù hợp với RHEL hiện đại, đặc biệt RHEL 9/10. Red Hat hiện dùng `dnf` cho software management; `systemctl` để quản lý `systemd`; `/etc/fstab` + generated mount units cho persistent mounts; `firewalld` vẫn phân biệt runtime/permanent configuration; và SELinux tiếp tục là security control mặc định quan trọng. Những điểm này được đối chiếu với tài liệu Red Hat hiện hành. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/installing-rhel-content?utm_source=chatgpt.com)

---

# 1. Mục tiêu cuối cùng của Module Thực hành Day 2

Sau module này, mục tiêu không phải chỉ là bạn nhớ command.

Bạn phải có khả năng nhận một RHEL server và thực hiện workflow:

```text
Receive task / incident
        ↓
Confirm scope + authorization
        ↓
Capture baseline
        ↓
Perform read-only checks
        ↓
Make controlled change
        ↓
Verify technical state
        ↓
Verify functional result
        ↓
Inject controlled failure
        ↓
Observe symptom
        ↓
Collect evidence
        ↓
Diagnose layer-by-layer
        ↓
Recover service
        ↓
Verify recovery
        ↓
Rollback / cleanup
        ↓
Document
        ↓
Escalate if outside boundary
```

Đây mới là operational competency mà Day 2 hướng tới.

---

# 2. Lab environment đề xuất

## 2.1. Topology

Lab tốt nhất có tối thiểu hai machines:

```text
                    LAB NETWORK
                 192.0.2.0/24
                       |
         +-------------+-------------+
         |                           |
         |                           |
+------------------+        +------------------+
| rhel-day2        |        | client-day2      |
| RHEL 9/10        |        | Linux client     |
| 192.0.2.10       |        | 192.0.2.20       |
+------------------+        +------------------+
         |
         +-- systemd
         +-- httpd
         +-- SELinux
         +-- firewalld
         +-- test storage
         +-- Bash scripts
         +-- cron
```

Các address `192.0.2.x` ở đây chỉ là **documentation examples**. Bạn phải dùng IP được cấp cho lab thực tế.

Một DNS record lab có thể là:

```text
rhel-day2.lab.example
```

trỏ đến:

```text
192.0.2.10
```

Nếu lab không có DNS server riêng, bạn vẫn có thể thực hành phần lớn module nhưng DNS verification nên sử dụng hostname được infrastructure lab cung cấp.

---

# 3. Prerequisites

Bạn cần:

| Requirement                                  | Mục đích                                     |
| -------------------------------------------- | -------------------------------------------- |
| Authorized RHEL VM                           | Không thao tác production                    |
| `sudo` least-privilege access                | Administrative tasks                         |
| SSH hoặc console access                      | Recovery nếu network config lỗi              |
| Snapshot/checkpoint VM nếu hypervisor hỗ trợ | Safety net                                   |
| Internet/internal RHEL repository access     | Package installation                         |
| Optional second Linux client                 | Remote connectivity test                     |
| Optional extra blank virtual disk            | Partition/mount lab                          |
| Evidence directory                           | Audit artifacts                              |
| Day 1 knowledge                              | users, groups, permissions, `sudo`, SSH, CLI |

Không sử dụng:

- production server;
- real credentials trong script;
- private key thật làm demo;
- production firewall;
- production disk;
- filesystem chứa dữ liệu quan trọng.

---

# 4. Safety model của lab

Trong suốt lab, hãy phân biệt hai nhóm command.

### READ-ONLY

Ký hiệu:

```text
[RO]
```

Ví dụ:

```bash
systemctl status
journalctl
ps
ss
ip address
ip route
df
findmnt
getenforce
rpm -q
dnf repolist
```

Các command này chủ yếu dùng để observe state.

### CONFIGURATION-CHANGING

Ký hiệu:

```text
[CHANGE]
```

Ví dụ:

```bash
dnf install
systemctl restart
firewall-cmd --add-service
mount
semanage fcontext
restorecon
editing /etc/fstab
```

Mọi `[CHANGE]` phải có ít nhất:

```text
Purpose
Expected impact
Verification
Rollback
Evidence
```

Đó là operational discipline.

---

# 5. Evidence repository

Trước khi làm lab, tạo evidence directory riêng.

```bash
mkdir -p "$HOME/day2-evidence"
chmod 700 "$HOME/day2-evidence"

EVID="$HOME/day2-evidence/$(date +%Y%m%d-%H%M%S)"
mkdir -p "$EVID"
chmod 700 "$EVID"

echo "$EVID"
```

Ví dụ:

```text
/home/student/day2-evidence/20261018-140505
```

Tạo metadata ban đầu:

```bash
date -Is | tee "$EVID/00-timestamp.txt"
hostnamectl | tee "$EVID/01-hostnamectl.txt"
cat /etc/os-release | tee "$EVID/02-os-release.txt"
uname -r | tee "$EVID/03-kernel.txt"
id | tee "$EVID/04-user-context.txt"
```

Mục đích là sau này evidence không chỉ có:

```text
httpd active
```

mà còn chứng minh:

```text
Host nào?
RHEL release nào?
Ai thực hiện?
Khi nào?
```

---

# 6. Phase 0 — Baseline trước khi thay đổi

Một nguyên tắc cực kỳ quan trọng:

> **Không thể chứng minh change thành công nếu không biết state trước change.**

Thu baseline.

## 6.1. Network baseline

```bash
ip -brief address | tee "$EVID/10-ip-address-before.txt"
ip route | tee "$EVID/11-route-before.txt"
```

Kiểm tra hostname resolution configuration:

```bash
cat /etc/resolv.conf | tee "$EVID/12-resolv-conf-before.txt"
```

Không sửa `/etc/resolv.conf` thủ công nếu file đang được NetworkManager hoặc cơ chế khác quản lý.

---

## 6.2. Firewall baseline

```bash
sudo firewall-cmd --state
```

Nếu chạy:

```bash
sudo firewall-cmd --get-active-zones \
  | tee "$EVID/13-firewall-zones-before.txt"
```

Sau đó:

```bash
sudo firewall-cmd --list-all \
  | tee "$EVID/14-firewall-default-zone-before.txt"
```

Nếu nhiều zones, xác định đúng zone của lab interface thay vì mặc định thao tác default zone.

---

## 6.3. SELinux baseline

```bash
getenforce | tee "$EVID/15-selinux-mode-before.txt"
sestatus | tee "$EVID/16-selinux-status-before.txt"
```

Expected trong một RHEL lab đúng chuẩn:

```text
Enforcing
```

Không chuyển sang `Disabled`.

---

## 6.4. Storage baseline

```bash
lsblk -f | tee "$EVID/17-lsblk-before.txt"
findmnt | tee "$EVID/18-findmnt-before.txt"
df -hT | tee "$EVID/19-df-before.txt"
df -i | tee "$EVID/20-inodes-before.txt"
```

Bốn command trả lời bốn câu khác nhau:

```text
lsblk
→ block devices nào tồn tại?

findmnt
→ filesystem nào đang mounted ở đâu?

df -hT
→ filesystem capacity/type thế nào?

df -i
→ inode capacity thế nào?
```

---

# 7. Lab Track 1 — Package Management

## Objective

Thực hiện đúng workflow:

```text
Repository
→ package discovery
→ approval
→ install
→ RPM verification
→ functional verification
→ evidence
```

RHEL 10 documentation xác nhận `dnf install` tự resolve package dependencies từ configured repositories. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/installing-rhel-content?utm_source=chatgpt.com)

---

# 8. Kiểm tra repositories trước khi install

### [RO]

```bash
sudo dnf repolist
```

Chi tiết hơn:

```bash
sudo dnf repolist --all
```

RHEL hiện hỗ trợ `dnf repolist --all` để xem cả enabled và disabled repositories. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/dnf-commands-list?utm_source=chatgpt.com)

Capture:

```bash
sudo dnf repolist --all \
  | tee "$EVID/30-dnf-repositories.txt"
```

Nếu đây là registered RHEL system, có thể kiểm tra registration/subscription bằng các công cụ tổ chức sử dụng.

Tuy nhiên cần nhớ:

```text
Direct Red Hat subscription
Satellite-managed system
Cloud RHUI image
Custom enterprise mirror
```

có thể khác nhau.

Không được thấy `subscription-manager` khác kỳ vọng rồi tự kết luận server lỗi.

---

# 9. Kiểm tra package trước installation

Ta sẽ dùng các packages phù hợp với Day 2 lab, ví dụ:

```text
httpd
traceroute
bind-utils
rsync
policycoreutils-python-utils
cronie
```

Package availability có thể phụ thuộc RHEL release/repository.

Kiểm tra:

```bash
dnf info httpd
dnf info traceroute
dnf info bind-utils
dnf info rsync
```

Kiểm tra đã cài chưa:

```bash
rpm -q httpd
rpm -q rsync
rpm -q cronie
```

Nếu chưa cài:

```text
package httpd is not installed
```

đây chưa phải error của lab; chỉ là pre-state.

---

# 10. Install approved packages

### [CHANGE]

Ví dụ:

```bash
sudo dnf install \
  httpd \
  traceroute \
  bind-utils \
  rsync \
  policycoreutils-python-utils \
  cronie
```

Không dùng `-y` lúc đang học lần đầu.

Hãy đọc transaction summary:

```text
Installing:
...
Installing dependencies:
...
Transaction Summary
...
```

Bạn cần nhìn:

- package nào được thêm;
- dependency nào được thêm;
- repository nào cung cấp;
- download size;
- installed size.

Sau khi hiểu transaction mới confirm.

---

# 11. Verify package installation

### [RO]

```bash
rpm -q httpd
rpm -q rsync
rpm -q cronie
```

Chi tiết:

```bash
rpm -qi httpd
```

Files package cung cấp:

```bash
rpm -ql httpd | less
```

File ownership query:

```bash
rpm -qf /usr/sbin/httpd
```

Expected:

```text
httpd-...
```

Capture:

```bash
rpm -q httpd rsync cronie \
  | tee "$EVID/31-package-verification.txt"
```

---

# 12. Operational lesson của package lab

Không dừng ở:

```text
dnf install returned success
```

Package task hoàn thành phải chứng minh:

```text
package available
      ↓
correct repository
      ↓
transaction completed
      ↓
RPM database contains package
      ↓
expected executable exists
      ↓
associated service/function works
```

---

# 13. Rollback package change

Không tự động remove toàn bộ packages vừa cài nếu chúng đã trở thành dependency cho lab khác.

Sau khi lab kết thúc, nếu environment yêu cầu cleanup:

```bash
sudo dnf remove <package>
```

Nhưng **đọc transaction summary trước khi confirm**, vì `dnf remove` có thể remove dependent packages hoặc unused dependencies. Red Hat documentation cũng lưu ý remove operation xử lý dependencies, vì vậy đây không phải command nên chạy mù quáng. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/managing_software_with_the_dnf_tool/configuring-dnf?utm_source=chatgpt.com)

---

# 14. Lab Track 2 — Tạo một controlled `systemd` service

Thay vì cố tình phá service hệ thống quan trọng, chúng ta tạo một service riêng:

```text
day2-heartbeat.service
```

Điều này an toàn và audit-friendly hơn.

---

# 15. Tạo service account

### [CHANGE]

Day 1 đã học user/group management.

Tạo service user không có interactive login:

```bash
sudo useradd \
  --system \
  --shell /sbin/nologin \
  day2svc
```

Verify:

```bash
getent passwd day2svc
```

Không dùng `root` nếu service không cần privilege.

Đây là least privilege.

---

# 16. Tạo service executable

### [CHANGE]

```bash
sudo tee /usr/local/sbin/day2-heartbeat.sh >/dev/null <<'EOF'
#!/usr/bin/env bash

set -u

while true; do
    printf '%s host=%s service=day2-heartbeat mode=%s status=alive\n' \
        "$(date -Is)" \
        "$(hostname)" \
        "${DAY2_MODE:-undefined}"
    sleep 10
done
EOF
```

Set ownership:

```bash
sudo chown root:root /usr/local/sbin/day2-heartbeat.sh
```

Permission:

```bash
sudo chmod 0755 /usr/local/sbin/day2-heartbeat.sh
```

Verify:

```bash
ls -l /usr/local/sbin/day2-heartbeat.sh
```

---

# 17. Giải thích script

```bash
#!/usr/bin/env bash
```

chọn Bash interpreter.

```bash
set -u
```

khi script reference undefined variable không được xử lý theo default behavior.

Trong lab, ta vẫn sử dụng:

```bash
${DAY2_MODE:-undefined}
```

nên có fallback.

Loop:

```bash
while true
```

giúp process chạy liên tục.

Output được ghi ra `stdout`.

Khi chạy dưới `systemd`, `stdout` có thể được journal thu thập.

---

# 18. Tạo systemd unit

Red Hat hiện quy định administrator-created/customized unit files đặt trong `/etc/systemd/system/`; vendor unit files thường nằm trong `/usr/lib/systemd/system/`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_systemd_unit_files_to_customize_and_optimize_your_system/using_systemd_unit_files_to_customize_and_optimize_your_system?utm_source=chatgpt.com)

### [CHANGE]

```bash
sudo tee /etc/systemd/system/day2-heartbeat.service >/dev/null <<'EOF'
[Unit]
Description=Day 2 Infrastructure Operations Heartbeat Lab
After=network.target

[Service]
Type=simple
User=day2svc
Group=day2svc
Environment=DAY2_MODE=lab
ExecStart=/usr/local/sbin/day2-heartbeat.sh
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF
```

---

# 19. Hiểu unit file

### `[Unit]`

```ini
Description=...
```

mô tả workload.

```ini
After=network.target
```

liên quan startup ordering.

Không được hiểu `After=` là guarantee rằng Internet/DNS/application network dependency đã sẵn sàng.

### `[Service]`

```ini
Type=simple
```

process được khởi chạy được xem là main process theo simple service semantics.

```ini
User=day2svc
Group=day2svc
```

không chạy bằng root.

```ini
Environment=DAY2_MODE=lab
```

demonstrate service environment variable.

```ini
ExecStart=...
```

program được start.

```ini
Restart=on-failure
```

có thể restart khi process failure.

### `[Install]`

```ini
WantedBy=multi-user.target
```

cho `systemctl enable` biết relationship cần tạo.

---

# 20. Reload systemd configuration

Sau khi tạo hoặc modify unit file:

### [CHANGE]

```bash
sudo systemctl daemon-reload
```

Red Hat hiện yêu cầu `daemon-reload` sau khi tạo/chỉnh unit file để `systemd` nhận state mới. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/working-with-systemd-unit-files?utm_source=chatgpt.com)

---

# 21. Start service

### [CHANGE]

Trước tiên:

```bash
sudo systemctl start day2-heartbeat.service
```

Verify:

```bash
systemctl status day2-heartbeat.service
```

Tách riêng machine-readable check:

```bash
systemctl is-active day2-heartbeat.service
```

Expected:

```text
active
```

---

# 22. `active` và `enabled`

Kiểm tra:

```bash
systemctl is-enabled day2-heartbeat.service
```

Ngay lúc này có thể nhận:

```text
disabled
```

Đó không mâu thuẫn với:

```text
active
```

Bởi vì:

```text
active
→ runtime state

enabled
→ startup configuration
```

Red Hat cũng tách riêng `systemctl is-active` và `systemctl is-enabled`, và cả hai sử dụng exit status để phản ánh state. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/managing-system-services-with-systemctl?utm_source=chatgpt.com)

---

# 23. Enable service

### [CHANGE]

```bash
sudo systemctl enable day2-heartbeat.service
```

Verify:

```bash
systemctl is-enabled day2-heartbeat.service
```

Expected:

```text
enabled
```

Hoặc ngay từ đầu có thể dùng:

```bash
sudo systemctl enable --now day2-heartbeat.service
```

nhưng trong lab ta tách `start` và `enable` để hiểu khác nhau.

---

# 24. Process inspection

Lấy main PID:

```bash
systemctl show \
  --property=MainPID \
  --value \
  day2-heartbeat.service
```

Ví dụ:

```text
12784
```

Store:

```bash
PID=$(systemctl show \
  --property=MainPID \
  --value \
  day2-heartbeat.service)
```

Inspect:

```bash
ps -o pid,ppid,user,group,stat,%cpu,%mem,etime,cmd -p "$PID"
```

Bạn phải identify:

```text
PID
PPID
USER
GROUP
STAT
CPU
MEM
ELAPSED TIME
COMMAND
```

Capture:

```bash
ps -o pid,ppid,user,group,stat,%cpu,%mem,etime,cmd -p "$PID" \
  | tee "$EVID/40-day2-service-process.txt"
```

---

# 25. Process state interpretation

Ví dụ:

```text
STAT
S
```

có thể cho thấy sleeping process.

Đó là bình thường vì heartbeat process phần lớn thời gian đang:

```bash
sleep 10
```

Đừng thấy process `sleeping` rồi kết luận service hung.

Bạn phải correlate state với workload behavior.

---

# 26. Verify environment variable trong service

Xem unit definition:

```bash
systemctl cat day2-heartbeat.service
```

Tìm:

```text
Environment=DAY2_MODE=lab
```

Xem journal:

```bash
journalctl \
  -u day2-heartbeat.service \
  -n 10 \
  --no-pager
```

Expected messages chứa:

```text
mode=lab
status=alive
```

Đây chứng minh:

```text
systemd Environment
        ↓
child process
        ↓
Bash variable
```

---

# 27. Shell environment vs service environment

Trong shell hiện tại:

```bash
echo "${DAY2_MODE:-not-set}"
```

có thể nhận:

```text
not-set
```

Trong service:

```text
mode=lab
```

Đây là bài học cực kỳ quan trọng:

> Service environment không nhất thiết giống interactive shell environment.

Điều tương tự sẽ xuất hiện với `cron`.

---

# 28. Service lifecycle practice

Thực hiện lần lượt.

### Stop

```bash
sudo systemctl stop day2-heartbeat.service
```

Verify:

```bash
systemctl is-active day2-heartbeat.service
```

Expected:

```text
inactive
```

### Start

```bash
sudo systemctl start day2-heartbeat.service
```

### Restart

```bash
sudo systemctl restart day2-heartbeat.service
```

Sau restart, kiểm tra PID:

```bash
systemctl show -p MainPID day2-heartbeat.service
```

PID có thể thay đổi.

---

# 29. Evidence service lifecycle

```bash
systemctl status day2-heartbeat.service \
  --no-pager \
  | tee "$EVID/41-service-status-good.txt"

systemctl is-enabled day2-heartbeat.service \
  | tee "$EVID/42-service-enabled.txt"

journalctl \
  -u day2-heartbeat.service \
  -n 20 \
  --no-pager \
  | tee "$EVID/43-service-journal-good.txt"
```

---

# 30. Lab Track 3 — Deliberately Failed Service

Syllabus yêu cầu rõ:

> recover a deliberately failed service. Infrastructure_Operations_14_Da…

Không phá `sshd`.

Không phá production application.

Ta chỉ phá custom service vừa tạo.

---

# 31. Pre-failure evidence

```bash
sudo cp -a \
  /etc/systemd/system/day2-heartbeat.service \
  /etc/systemd/system/day2-heartbeat.service.pre-failure
```

Capture current state:

```bash
systemctl status day2-heartbeat.service \
  --no-pager \
  | tee "$EVID/50-before-service-failure.txt"
```

---

# 32. Inject service failure

### [CHANGE — CONTROLLED FAILURE]

Edit:

```text
ExecStart=/usr/local/sbin/day2-heartbeat.sh
```

thành deliberate invalid path:

```text
ExecStart=/usr/local/sbin/day2-heartbeat-MISSING.sh
```

Ví dụ:

```bash
sudo sed -i \
  's|/usr/local/sbin/day2-heartbeat.sh|/usr/local/sbin/day2-heartbeat-MISSING.sh|' \
  /etc/systemd/system/day2-heartbeat.service
```

Reload:

```bash
sudo systemctl daemon-reload
```

Restart:

```bash
sudo systemctl restart day2-heartbeat.service
```

Expected:

```text
restart failure
```

---

# 33. Không sửa ngay

Đây chính là bước học troubleshooting.

Đừng restore file ngay.

Trước tiên thu evidence.

```bash
systemctl status day2-heartbeat.service \
  --no-pager \
  | tee "$EVID/51-failed-service-status.txt"
```

Sau đó:

```bash
journalctl \
  -u day2-heartbeat.service \
  --since "-5 minutes" \
  --no-pager \
  | tee "$EVID/52-failed-service-journal.txt"
```

Inspect unit:

```bash
systemctl cat day2-heartbeat.service \
  | tee "$EVID/53-failed-service-unit.txt"
```

Check executable:

```bash
ls -l /usr/local/sbin/day2-heartbeat-MISSING.sh
```

Expected:

```text
No such file or directory
```

---

# 34. Root cause reasoning

Evidence:

```text
Service used to work
        +
Unit now references missing executable
        +
Journal shows execution failure
        +
File does not exist
```

Root cause:

```text
Incorrect ExecStart path introduced into systemd unit.
```

Không phải:

```text
systemd is broken.
```

Không phải:

```text
Linux needs reboot.
```

---

# 35. Recover service

### [ROLLBACK]

Restore:

```bash
sudo cp -a \
  /etc/systemd/system/day2-heartbeat.service.pre-failure \
  /etc/systemd/system/day2-heartbeat.service
```

Reload:

```bash
sudo systemctl daemon-reload
```

Reset failed state:

```bash
sudo systemctl reset-failed day2-heartbeat.service
```

Start:

```bash
sudo systemctl start day2-heartbeat.service
```

Verify:

```bash
systemctl is-active day2-heartbeat.service
```

Expected:

```text
active
```

Functional evidence:

```bash
journalctl \
  -u day2-heartbeat.service \
  -n 5 \
  --no-pager
```

Expected:

```text
status=alive
```

---

# 36. Recovery evidence

```bash
systemctl status day2-heartbeat.service \
  --no-pager \
  | tee "$EVID/54-service-recovered.txt"
```

Bạn cần có ba states:

```text
before failure
failed state
recovered state
```

Đó chính là before/after evidence.

---

# 37. Lab Track 4 — Network Configuration & Verification

Networking lab nên tuân theo nguyên tắc:

> Không chỉnh management interface đang dùng để SSH nếu không có out-of-band console.

Ta ưu tiên inspect primary interface và dùng một **temporary loopback IP** cho configuration exercise.

---

# 38. Identify interfaces

### [RO]

```bash
ip -brief link
```

```bash
ip -brief address
```

Full:

```bash
ip address show
```

Xác định:

```text
interface name
state
MAC
IPv4
prefix
```

Modern RHEL interface names có thể là:

```text
ens192
ens160
enp1s0
eth0
```

Không hard-code `eth0`.

---

# 39. Routing table

```bash
ip route
```

Typical:

```text
default via 192.0.2.1 dev ens192
192.0.2.0/24 dev ens192 proto kernel ...
```

Hãy xác định:

```text
default gateway
connected network
interface
```

Test kernel route decision:

```bash
ip route get <CLIENT_IP>
```

Ví dụ:

```bash
ip route get 192.0.2.20
```

Output có thể cho:

```text
192.0.2.20 dev ens192 src 192.0.2.10
```

---

# 40. Temporary IP configuration exercise

Dùng documentation subnet trên loopback để không ảnh hưởng primary NIC.

### [CHANGE — TEMPORARY]

```bash
sudo ip address add 198.51.100.10/32 dev lo
```

Verify:

```bash
ip address show dev lo
```

Test:

```bash
ping -c 3 198.51.100.10
```

Expected success.

Cleanup:

```bash
sudo ip address del 198.51.100.10/32 dev lo
```

Verify removal:

```bash
ip address show dev lo
```

Điểm cần học:

```text
ip command modification này chỉ là runtime state.
```

Persistent network configuration trên RHEL hiện đại thường liên quan NetworkManager/profile configuration và phải theo runbook của environment. Ta không chỉnh primary NIC trong Day 2 lab nếu không có console recovery.

---

# 41. Gateway connectivity

Xác định default gateway:

```bash
ip route | grep '^default'
```

Ví dụ:

```text
default via 192.0.2.1 dev ens192
```

Test nếu ICMP được permit:

```bash
ping -c 3 192.0.2.1
```

Nếu fail, không kết luận ngay gateway down.

ICMP có thể bị filtering.

---

# 42. DNS verification

Kiểm tra resolver:

```bash
cat /etc/resolv.conf
```

Quan trọng hơn, test qua libc/NSS:

```bash
getent hosts rhel-day2.lab.example
```

Expected:

```text
192.0.2.10 rhel-day2.lab.example
```

Nếu `bind-utils` đã cài, có thể dùng:

```bash
dig rhel-day2.lab.example
```

Hoặc:

```bash
host rhel-day2.lab.example
```

---

# 43. DNS troubleshooting

Nếu:

```bash
ping -c 2 192.0.2.10
```

works,

nhưng:

```bash
getent hosts rhel-day2.lab.example
```

không trả về gì,

thì suspicion:

```text
DNS/name resolution
```

không phải basic IP connectivity.

Đây là distinction phải hình thành trong Day 2.

---

# 44. `tracepath` / `traceroute`

Nếu available:

```bash
tracepath <LAB_CLIENT_OR_ROUTED_HOST>
```

hoặc:

```bash
traceroute <LAB_CLIENT_OR_ROUTED_HOST>
```

Bạn đang quan sát:

```text
hop 1
hop 2
hop 3
...
destination
```

Không được hiểu timeout ở một hop là chắc chắn router đó hỏng.

Một router có thể forward packet nhưng không reply probe.

---

# 45. Lab Track 5 — HTTP service + TCP port + `firewalld`

Ta dùng `httpd` làm application service để tích hợp:

```text
systemd
process
TCP
socket
firewall
SELinux
logs
curl
```

---

# 46. Check httpd before start

```bash
systemctl status httpd --no-pager
```

Có thể:

```text
inactive
```

hoặc:

```text
disabled
```

Kiểm tra enable state:

```bash
systemctl is-enabled httpd
```

---

# 47. Start HTTP server

### [CHANGE]

```bash
sudo systemctl start httpd
```

Verify:

```bash
systemctl is-active httpd
```

Inspect process:

```bash
ps -ef | grep '[h]ttpd'
```

---

# 48. Socket verification

```bash
sudo ss -lntp
```

Tìm TCP port `80`.

Hoặc:

```bash
sudo ss -lntp | grep ':80 '
```

Expected conceptual:

```text
LISTEN ... 0.0.0.0:80 ...
```

hoặc IPv6 equivalent/configuration-specific result.

Điều cần chứng minh:

```text
Process active
        ↓
Socket LISTEN
```

---

# 49. Local functional test

```bash
curl \
  --connect-timeout 3 \
  http://127.0.0.1/
```

Hoặc chỉ headers:

```bash
curl -I http://127.0.0.1/
```

Một HTTP error response vẫn chứng minh khác với TCP connection failure.

Ví dụ:

```text
HTTP/1.1 403 Forbidden
```

nghĩa là bạn đã reach HTTP server.

Nó khác:

```text
Connection refused
```

---

# 50. Firewall inspection

Red Hat hiện mô tả `firewalld` với zones, services và separate runtime/permanent states. Runtime changes không survive reload/restart trừ khi được đưa vào permanent configuration. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld?utm_source=chatgpt.com)

Xác định active zone:

```bash
sudo firewall-cmd --get-active-zones
```

Giả sử interface thuộc:

```text
public
```

Inspect:

```bash
sudo firewall-cmd \
  --zone=public \
  --list-all
```

---

# 51. Check HTTP permission

```bash
sudo firewall-cmd \
  --zone=public \
  --query-service=http
```

Nếu:

```text
no
```

remote HTTP traffic chưa được permit theo service definition đó.

---

# 52. Add runtime firewall permission

### [CHANGE]

```bash
sudo firewall-cmd \
  --zone=public \
  --add-service=http
```

Verify:

```bash
sudo firewall-cmd \
  --zone=public \
  --query-service=http
```

Expected:

```text
yes
```

---

# 53. Remote verification

Từ `client-day2`:

```bash
curl \
  --connect-timeout 3 \
  http://192.0.2.10/
```

Hoặc:

```bash
curl \
  --connect-timeout 3 \
  http://rhel-day2.lab.example/
```

Nếu successful:

```text
client
 ↓
network
 ↓
RHEL NIC
 ↓
firewalld
 ↓
TCP/80
 ↓
httpd
```

đã hoạt động.

---

# 54. Make firewall rule persistent

### [CHANGE]

Sau khi xác định rule chính xác:

```bash
sudo firewall-cmd \
  --zone=public \
  --permanent \
  --add-service=http
```

Reload:

```bash
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd \
  --zone=public \
  --query-service=http
```

Và:

```bash
sudo firewall-cmd \
  --zone=public \
  --permanent \
  --query-service=http
```

Cả hai nên phản ánh desired state.

---

# 55. Evidence firewall

```bash
sudo firewall-cmd \
  --zone=public \
  --list-all \
  | tee "$EVID/60-firewall-http-enabled.txt"
```

Client:

```bash
curl -I \
  --connect-timeout 3 \
  http://192.0.2.10/
```

Lưu output client-side nếu evidence package hỗ trợ.

---

# 56. Network Failure Injection — Firewall denial

Syllabus yêu cầu network failure injection.

Ta sẽ không ngắt NIC hay route management.

Thay vào đó chỉ remove HTTP từ **runtime firewall**, để SSH vẫn hoạt động.

### [CONTROLLED FAILURE]

```bash
sudo firewall-cmd \
  --zone=public \
  --remove-service=http
```

Permanent config vẫn giữ HTTP.

Verify:

```bash
sudo firewall-cmd \
  --zone=public \
  --query-service=http
```

Expected:

```text
no
```

Từ client:

```bash
curl \
  --connect-timeout 3 \
  http://192.0.2.10/
```

Expected remote failure.

---

# 57. Troubleshoot injected network failure

Đừng biết trước root cause khi luyện.

Đi theo layer.

### Step 1 — Host reachable?

```bash
ping ...
```

nếu policy cho phép.

### Step 2 — Service?

```bash
systemctl is-active httpd
```

Expected:

```text
active
```

### Step 3 — Port?

```bash
ss -lnt | grep ':80 '
```

Expected:

```text
LISTEN
```

### Step 4 — Local application?

```bash
curl http://127.0.0.1/
```

Works.

### Step 5 — Remote?

Fails.

### Step 6 — Firewall?

```bash
sudo firewall-cmd \
  --zone=public \
  --query-service=http
```

Returns:

```text
no
```

Root cause localized:

```text
Host firewall runtime state.
```

---

# 58. Recover firewall failure

Because permanent state still has HTTP, simplest rollback:

```bash
sudo firewall-cmd --reload
```

Runtime config is rebuilt from permanent state.

Verify:

```bash
sudo firewall-cmd \
  --zone=public \
  --query-service=http
```

Expected:

```text
yes
```

Remote:

```bash
curl http://192.0.2.10/
```

Works again.

Đây là excellent demonstration của:

```text
runtime
vs
permanent
```

---

# 59. Lab Track 6 — SELinux-related access failure

Đây là một trong những phần giá trị nhất của Day 2.

Ta sẽ cố tình tạo một directory:

```text
/srv/day2web
```

và để `httpd` access nó.

Red Hat documentation hiện dùng gần như chính scenario này: `/srv` thường không có label phù hợp cho Apache; administrator cần map path thành `httpd_sys_content_t` bằng `semanage fcontext` rồi áp dụng bằng `restorecon`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_selinux/using_selinux?utm_source=chatgpt.com)

---

# 60. Create web content outside default DocumentRoot

### [CHANGE]

```bash
sudo mkdir -p /srv/day2web
```

```bash
echo 'Day 2 SELinux Lab OK' \
  | sudo tee /srv/day2web/index.html
```

Permissions:

```bash
sudo chmod 0755 /srv/day2web
sudo chmod 0644 /srv/day2web/index.html
```

Check DAC:

```bash
ls -ld /srv/day2web
ls -l /srv/day2web/index.html
```

---

# 61. Inspect initial SELinux context

```bash
ls -Zd /srv/day2web
ls -Z /srv/day2web/index.html
```

Bạn có thể thấy type không phải:

```text
httpd_sys_content_t
```

Đó là clue.

---

# 62. Configure Apache alias

### [CHANGE]

Tạo lab configuration riêng:

```bash
sudo tee /etc/httpd/conf.d/day2-lab.conf >/dev/null <<'EOF'
Alias /day2/ "/srv/day2web/"

<Directory "/srv/day2web">
    AllowOverride None
    Require all granted
</Directory>
EOF
```

Trước restart, validate configuration:

```bash
sudo apachectl configtest
```

Expected:

```text
Syntax OK
```

Đây là best practice quan trọng:

```text
validate configuration
before
restart
```

---

# 63. Reload/restart Apache

Trong lab:

```bash
sudo systemctl restart httpd
```

Verify:

```bash
systemctl is-active httpd
```

---

# 64. Test path

```bash
curl -i http://127.0.0.1/day2/
```

Nếu SELinux đang `Enforcing` và context không phù hợp, bạn có thể nhận:

```text
403 Forbidden
```

hoặc access failure tương tự.

Điều quan trọng:

```text
httpd service active
TCP/80 listening
default content reachable
/day2/ not readable
```

---

# 65. Đừng disable SELinux

Sai:

```bash
sudo setenforce 0
```

rồi tuyên bố solved.

Trong lab chúng ta muốn sửa root cause đúng cách.

---

# 66. Check audit evidence

Mode:

```bash
getenforce
```

Context:

```bash
ls -Zd /srv/day2web
```

Nếu audit subsystem available:

```bash
sudo ausearch \
  -m AVC \
  -ts recent
```

Bạn đang tìm SELinux denial liên quan:

```text
httpd
/srv/day2web
```

Không phải mọi AVC gần nhất đều liên quan incident; correlate timestamp/path/process.

---

# 67. Correct SELinux context persistently

### [CHANGE]

Add file-context mapping:

```bash
sudo semanage fcontext \
  -a \
  -t httpd_sys_content_t \
  '/srv/day2web(/.*)?'
```

Điều quan trọng:

```text
semanage fcontext
```

không trực tiếp relabel files.

Nó tạo persistent mapping.

Sau đó:

```bash
sudo restorecon \
  -Rv \
  /srv/day2web
```

Red Hat documentation cũng dùng chính sequence này cho Apache content trong `/srv`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html-single/using_selinux/using_selinux?utm_source=chatgpt.com)

---

# 68. Verify SELinux correction

```bash
ls -Zd /srv/day2web
ls -Z /srv/day2web/index.html
```

Expected type:

```text
httpd_sys_content_t
```

Functional check:

```bash
curl -i http://127.0.0.1/day2/
```

Expected:

```text
HTTP/1.1 200 OK
...
Day 2 SELinux Lab OK
```

Remote:

```bash
curl http://192.0.2.10/day2/
```

---

# 69. Evidence SELinux

Capture:

```bash
getenforce \
  | tee "$EVID/70-selinux-mode.txt"

ls -Zd /srv/day2web \
  | tee "$EVID/71-selinux-context-directory.txt"

ls -Z /srv/day2web/index.html \
  | tee "$EVID/72-selinux-context-file.txt"

curl -i http://127.0.0.1/day2/ \
  | tee "$EVID/73-selinux-functional-test.txt"
```

Một evidence package tốt nên có:

```text
failure
AVC/context clue
corrective action
correct label
successful HTTP response
```

---

# 70. Important SELinux lesson

Ba mechanisms riêng biệt:

```text
DAC permissions
        ↓
SELinux
        ↓
Firewall/network
```

Ví dụ:

```text
-rw-r--r--
```

không chứng minh Apache được SELinux permit.

Và:

```text
SELinux allowed
```

không chứng minh remote firewall path open.

Phải troubleshoot từng layer.

---

# 71. Lab Track 7 — Disk, Partition, Filesystem và Mount

Đây là phần có khả năng gây data loss nếu chọn sai disk.

Vì vậy:

> Chỉ sử dụng một **blank virtual disk dành riêng cho lab**.

Ví dụ hypervisor cấp thêm disk 2 GiB.

Không sử dụng `/dev/sda`, `/dev/nvme0n1` hay bất kỳ device nào chỉ vì ví dụ ở đây ghi như vậy.

---

# 72. Identify candidate lab disk

### [RO]

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

Giả sử mentor xác nhận:

```text
/dev/vdb
```

là blank lab disk.

Set variable sau khi **xác nhận thủ công**:

```bash
LAB_DISK=/dev/vdb
```

Check:

```bash
lsblk "$LAB_DISK"
```

Phải verify:

```text
no production filesystem
no important mount
correct expected size
```

---

# 73. Safety gate trước partition

Trước command destructive, phải có:

```text
☐ Correct VM
☐ Correct lab disk
☐ Correct device path
☐ No useful data
☐ No filesystem mounted
☐ Snapshot/rollback available if required
☐ Authorized for lab
```

Nếu không chắc bất kỳ mục nào:

```text
STOP
```

Không đoán.

---

# 74. Create partition

Một ví dụ dùng `parted`.

### [CHANGE — DESTRUCTIVE TO LAB DISK]

```bash
sudo parted "$LAB_DISK" --script \
  mklabel gpt
```

Create partition:

```bash
sudo parted "$LAB_DISK" --script \
  mkpart primary xfs 1MiB 100%
```

Sau đó:

```bash
lsblk "$LAB_DISK"
```

Có thể thấy:

```text
vdb
└─vdb1
```

Tên partition phụ thuộc device naming.

---

# 75. Identify partition safely

Không assume blindly.

Inspect:

```bash
lsblk -f "$LAB_DISK"
```

Giả sử partition là:

```text
/dev/vdb1
```

Set:

```bash
LAB_PART=/dev/vdb1
```

---

# 76. Create filesystem

### [CHANGE — DESTRUCTIVE]

```bash
sudo mkfs.xfs "$LAB_PART"
```

**Không chạy `mkfs` trên device chứa data.**

`mkfs` là một trong những command nguy hiểm nhất của lab nếu chọn sai target.

Verify:

```bash
lsblk -f "$LAB_DISK"
```

Expected:

```text
FSTYPE xfs
UUID ...
```

---

# 77. Create mount point

```bash
sudo mkdir -p /mnt/day2data
```

Check:

```bash
findmnt /mnt/day2data
```

Trước mount không nên có filesystem khác ở đó.

---

# 78. Temporary mount

### [CHANGE]

```bash
sudo mount "$LAB_PART" /mnt/day2data
```

Verify:

```bash
findmnt /mnt/day2data
```

```bash
df -hT /mnt/day2data
```

Test write:

```bash
sudo touch /mnt/day2data/day2-test.txt
```

Verify:

```bash
ls -l /mnt/day2data/day2-test.txt
```

---

# 79. Persistent mount preparation

Persistent configuration sẽ dùng UUID.

Red Hat hiện khuyến nghị lấy UUID, add entry vào `/etc/fstab`, chạy `systemctl daemon-reload`, rồi test mounting trước khi tin rằng configuration đúng. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/persistently-mounting-file-systems?utm_source=chatgpt.com)

Get UUID:

```bash
sudo blkid "$LAB_PART"
```

Hoặc:

```bash
lsblk -f "$LAB_PART"
```

Store:

```bash
LAB_UUID=$(blkid -s UUID -o value "$LAB_PART")
```

Check:

```bash
echo "$LAB_UUID"
```

---

# 80. Backup `/etc/fstab`

### [CHANGE PREPARATION]

```bash
sudo cp -a \
  /etc/fstab \
  "/etc/fstab.day2-backup.$(date +%Y%m%d-%H%M%S)"
```

Capture original:

```bash
sudo cat /etc/fstab \
  | tee "$EVID/80-fstab-before.txt"
```

---

# 81. Add persistent entry

### [CHANGE]

Example:

```bash
echo "UUID=$LAB_UUID /mnt/day2data xfs defaults 0 0" \
  | sudo tee -a /etc/fstab
```

Review manually:

```bash
tail -n 5 /etc/fstab
```

Expected entry:

```text
UUID=<uuid> /mnt/day2data xfs defaults 0 0
```

RHEL 10 documentation describes exactly this UUID-based `/etc/fstab` approach. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/persistently-mounting-file-systems?utm_source=chatgpt.com)

---

# 82. Reload generated mount units

```bash
sudo systemctl daemon-reload
```

Because `systemd` generates mount units from `/etc/fstab`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/persistently-mounting-file-systems?utm_source=chatgpt.com)

---

# 83. Test without reboot

Critical rule:

> **Không reboot chỉ để discover rằng `/etc/fstab` có typo.**

Unmount:

```bash
sudo umount /mnt/day2data
```

Check:

```bash
findmnt /mnt/day2data
```

Then mount by mount point:

```bash
sudo mount /mnt/day2data
```

Nếu thành công, `mount` đã sử dụng `/etc/fstab`.

Verify:

```bash
findmnt /mnt/day2data
df -hT /mnt/day2data
```

Data verification:

```bash
ls -l /mnt/day2data/day2-test.txt
```

File vẫn phải tồn tại.

---

# 84. Persistent mount evidence

```bash
lsblk -f "$LAB_DISK" \
  | tee "$EVID/81-lsblk-after.txt"

findmnt /mnt/day2data \
  | tee "$EVID/82-findmnt-day2data.txt"

df -hT /mnt/day2data \
  | tee "$EVID/83-df-day2data.txt"

grep '/mnt/day2data' /etc/fstab \
  | tee "$EVID/84-fstab-day2-entry.txt"
```

---

# 85. `/etc/fstab` failure scenario awareness

Typical mistakes:

```text
Wrong UUID
Wrong filesystem type
Typo mount point
Invalid option
Missing directory
Wrong dependency/network filesystem assumptions
```

Nếu test mount fails:

```bash
sudo mount /mnt/day2data
```

thu error.

Sau đó:

```bash
journalctl -b --no-pager | tail -n 100
```

và inspect `/etc/fstab`.

Không reboot cho tới khi configuration được validate.

---

# 86. Lab Track 8 — Disk Capacity và Inode Capacity

Không nên làm đầy `/` hoặc `/var` của lab VM.

Thay vào đó ta tạo **isolated loopback filesystem image**.

Đây là một kiến thức bổ sung phục vụ safe failure injection.

---

# 87. Safe inode-exhaustion filesystem

Tạo image nhỏ:

### [CHANGE — ISOLATED]

```bash
sudo truncate \
  -s 64M \
  /var/tmp/day2-inode.img
```

Create ext4 với số inode nhỏ:

```bash
sudo mkfs.ext4 \
  -F \
  -N 128 \
  /var/tmp/day2-inode.img
```

Create mount point:

```bash
sudo mkdir -p /mnt/day2-inode
```

Mount as loop device:

```bash
sudo mount \
  -o loop \
  /var/tmp/day2-inode.img \
  /mnt/day2-inode
```

---

# 88. Baseline capacity

```bash
df -h /mnt/day2-inode
```

```bash
df -i /mnt/day2-inode
```

Observe:

```text
block capacity
inode capacity
```

separately.

---

# 89. Exhaust inodes safely

### [CONTROLLED FAILURE]

Create small empty files:

```bash
for i in $(seq 1 300); do
    sudo touch "/mnt/day2-inode/file-$i" || break
done
```

Eventually:

```text
No space left on device
```

Check:

```bash
df -h /mnt/day2-inode
```

Bạn có thể thấy vẫn còn block capacity.

Nhưng:

```bash
df -i /mnt/day2-inode
```

có thể cho gần:

```text
IUse% 100%
```

---

# 90. Root cause reasoning

Symptom:

```text
touch: No space left on device
```

Naive check:

```bash
df -h
```

cho thấy còn MB.

Nếu dừng ở đây, bạn sẽ confused.

Correct check:

```bash
df -i
```

cho thấy inode exhaustion.

Root cause:

```text
Too many filesystem objects relative to available inodes.
```

---

# 91. Recover inode failure

Delete controlled files:

```bash
sudo rm -f /mnt/day2-inode/file-*
```

Verify:

```bash
df -i /mnt/day2-inode
```

Test:

```bash
sudo touch /mnt/day2-inode/recovery-test
```

Works.

Cleanup later:

```bash
sudo umount /mnt/day2-inode
sudo rm -f /var/tmp/day2-inode.img
sudo rmdir /mnt/day2-inode
```

---

# 92. Optional safe disk-space exhaustion

Có thể dùng isolated image riêng.

```bash
sudo truncate -s 128M /var/tmp/day2-space.img
sudo mkfs.ext4 -F /var/tmp/day2-space.img
sudo mkdir -p /mnt/day2-space
sudo mount -o loop /var/tmp/day2-space.img /mnt/day2-space
```

Baseline:

```bash
df -h /mnt/day2-space
```

Fill **chỉ filesystem isolated này**:

```bash
sudo dd \
  if=/dev/zero \
  of=/mnt/day2-space/fill.dat \
  bs=1M \
  status=progress
```

Command sẽ eventually stop vì filesystem full.

Không chạy kiểu này trên:

```text
/
 /var
 /home
```

hoặc production filesystem.

---

# 93. Verify disk-full condition

```bash
df -h /mnt/day2-space
```

Expected high/full usage.

Try:

```bash
echo test \
  | sudo tee /mnt/day2-space/new-file.txt
```

có thể fail.

---

# 94. Recover disk-full condition

```bash
sudo rm -f /mnt/day2-space/fill.dat
```

Verify:

```bash
df -h /mnt/day2-space
```

Cleanup:

```bash
sudo umount /mnt/day2-space
sudo rm -f /var/tmp/day2-space.img
sudo rmdir /mnt/day2-space
```

---

# 95. Storage diagnostic decision tree

Khi application nói:

```text
Cannot write
```

đừng chỉ chạy `df -h`.

Đi theo:

```text
Is filesystem mounted?
        |
       yes
        ↓
Is filesystem writable?
        |
       yes
        ↓
Block capacity available?
        |
       yes
        ↓
Inodes available?
        |
       yes
        ↓
Correct DAC ownership/permissions?
        |
       yes
        ↓
SELinux permits?
        |
       yes
        ↓
Application-specific problem?
```

---

# 96. Lab Track 9 — Logs và `journalctl`

RHEL logging sử dụng `systemd-journald`; RHEL environments cũng commonly use `rsyslog`, trong đó `rsyslogd` có thể process journal/syslog events và write files dưới `/var/log`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/configuring_basic_system_settings/index?utm_source=chatgpt.com)

---

# 97. Journal inspection

All current boot:

```bash
journalctl -b
```

Service-specific:

```bash
journalctl \
  -u day2-heartbeat.service
```

Apache:

```bash
journalctl \
  -u httpd.service
```

Last 20 entries:

```bash
journalctl \
  -u httpd.service \
  -n 20 \
  --no-pager
```

Since relative time:

```bash
journalctl \
  -u httpd.service \
  --since "-10 minutes" \
  --no-pager
```

---

# 98. Why time filtering matters

Bad evidence:

```text
500,000 lines from entire journal
```

Good evidence:

```text
Incident started 14:32
Evidence from 14:25–14:40
```

Điều này tạo usable timeline.

---

# 99. Generate safe test log entries

Utility `logger` có thể phát syslog/journal messages.

```bash
logger \
  -t day2lab \
  "event=healthcheck status=SUCCESS host=$(hostname)"
```

Generate failure sample:

```bash
logger \
  -t day2lab \
  "event=backup status=FAILURE code=12 host=$(hostname)"
```

Another:

```bash
logger \
  -t day2lab \
  "event=service status=RECOVERED service=day2-heartbeat"
```

---

# 100. Query by identifier

```bash
journalctl \
  -t day2lab \
  --since "-10 minutes" \
  --no-pager
```

Expected:

```text
event=healthcheck status=SUCCESS ...
event=backup status=FAILURE ...
event=service status=RECOVERED ...
```

---

# 101. `grep` filtering

```bash
journalctl \
  -t day2lab \
  -o cat \
  --no-pager \
  | grep 'status=FAILURE'
```

Expected:

```text
event=backup status=FAILURE ...
```

Case-insensitive example:

```bash
... | grep -i 'failure'
```

---

# 102. Regex filtering

Example:

```bash
journalctl \
  -t day2lab \
  -o cat \
  --no-pager \
  | grep -E 'status=(FAILURE|RECOVERED)'
```

Matches two states.

---

# 103. `awk` extraction

Given:

```text
event=backup status=FAILURE code=12 host=rhel-day2
```

Extract selected fields:

```bash
journalctl \
  -t day2lab \
  -o cat \
  --no-pager \
  | awk '/status=FAILURE/ {print $1, $2, $3}'
```

Possible:

```text
event=backup status=FAILURE code=12
```

---

# 104. `sed` transformation

Ví dụ readable report:

```bash
journalctl \
  -t day2lab \
  -o cat \
  --no-pager \
  | sed 's/status=FAILURE/STATUS=FAILURE/'
```

Đây chỉ transform output.

Không modify journal.

---

# 105. Combined pipeline

```bash
journalctl \
  -t day2lab \
  -o cat \
  --no-pager \
  | grep -E 'status=(FAILURE|RECOVERED)' \
  | awk '{print $1, $2, $3}' \
  | sed 's/status=/status: /'
```

Concept:

```text
journal
  ↓
grep: filter
  ↓
awk: extract
  ↓
sed: transform
```

---

# 106. Log evidence

```bash
journalctl \
  -u day2-heartbeat.service \
  --since "-30 minutes" \
  --no-pager \
  > "$EVID/90-heartbeat-journal.txt"

journalctl \
  -u httpd.service \
  --since "-30 minutes" \
  --no-pager \
  > "$EVID/91-httpd-journal.txt"

journalctl \
  -t day2lab \
  --since "-30 minutes" \
  --no-pager \
  > "$EVID/92-day2lab-events.txt"
```

---

# 107. Traditional log files awareness

Nếu `rsyslog` được cấu hình theo typical RHEL setup, bạn có thể gặp:

```text
/var/log/messages
/var/log/secure
/var/log/cron
```

Red Hat lưu ý actual list phụ thuộc services/configuration. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/configuring_basic_system_settings/index?utm_source=chatgpt.com)

Không viết automation assume tất cả RHEL đều có exact same logs.

---

# 108. Log rotation awareness trong lab

Check:

```bash
rpm -q logrotate
```

Inspect:

```bash
ls -l /etc/logrotate.conf
ls -l /etc/logrotate.d/
```

Trên RHEL 9, `logrotate` được start/manage qua a `systemd timer`, thay vì old `/etc/cron.daily/logrotate` mechanism. Đây là ví dụ rõ tại sao phải kiểm tra version/environment thay vì thuộc một behavior cũ. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/considerations_in_adopting_rhel_9/index?utm_source=chatgpt.com)

Check:

```bash
systemctl status logrotate.timer
```

nếu applicable.

---

# 109. Lab Track 10 — Bash Health-Check Script

Syllabus yêu cầu:

> develop a Bash health-check ... with exit-code handling. Infrastructure_Operations_14_Da…

Script phải:

```text
collect useful state
classify health
produce readable output
return meaningful exit code
```

Không chỉ:

```bash
echo "server healthy"
```

---

# 110. Health-check design

Ta sẽ kiểm tra:

```text
1. Required service: httpd
2. Required service: day2-heartbeat
3. TCP/80 listening
4. Root filesystem block usage
5. Root filesystem inode usage
6. DNS resolution
7. Local HTTP function
```

Exit-code contract do **script lab tự định nghĩa**:

```text
0 = OK
1 = WARNING
2 = CRITICAL
```

Đây không phải universal Linux rule ngoài việc `0` conventionally biểu thị success; 1/2 là contract riêng của script này.

---

# 111. Create health-check script

### [CHANGE]

```bash
sudo tee /usr/local/sbin/day2-healthcheck.sh >/dev/null <<'EOF'
#!/usr/bin/env bash

set -u
set -o pipefail

WARNINGS=0
CRITICALS=0

DISK_WARN=80
DISK_CRIT=90

INODE_WARN=80
INODE_CRIT=90

DNS_NAME="${DNS_NAME:-localhost}"

ok() {
    printf 'OK: %s\n' "$*"
}

warn() {
    printf 'WARNING: %s\n' "$*"
    WARNINGS=$((WARNINGS + 1))
}

crit() {
    printf 'CRITICAL: %s\n' "$*"
    CRITICALS=$((CRITICALS + 1))
}

check_service() {
    local service="$1"

    if systemctl is-active --quiet "$service"; then
        ok "service $service is active"
    else
        crit "service $service is not active"
    fi
}

check_service httpd.service
check_service day2-heartbeat.service

if ss -lnt | awk '{print $4}' | grep -Eq '(^|:|\])80$'; then
    ok "TCP/80 is listening"
else
    crit "TCP/80 is not listening"
fi

DISK_USED=$(
    df -P / |
    awk 'NR==2 {gsub(/%/,"",$5); print $5}'
)

if (( DISK_USED >= DISK_CRIT )); then
    crit "root filesystem usage is ${DISK_USED}%"
elif (( DISK_USED >= DISK_WARN )); then
    warn "root filesystem usage is ${DISK_USED}%"
else
    ok "root filesystem usage is ${DISK_USED}%"
fi

INODE_USED=$(
    df -Pi / |
    awk 'NR==2 {gsub(/%/,"",$5); print $5}'
)

if (( INODE_USED >= INODE_CRIT )); then
    crit "root inode usage is ${INODE_USED}%"
elif (( INODE_USED >= INODE_WARN )); then
    warn "root inode usage is ${INODE_USED}%"
else
    ok "root inode usage is ${INODE_USED}%"
fi

if getent hosts "$DNS_NAME" >/dev/null 2>&1; then
    ok "DNS/NSS resolution works for $DNS_NAME"
else
    warn "cannot resolve $DNS_NAME"
fi

if curl \
    --silent \
    --show-error \
    --fail \
    --max-time 5 \
    http://127.0.0.1/day2/ \
    >/dev/null 2>&1
then
    ok "local HTTP functional test succeeded"
else
    crit "local HTTP functional test failed"
fi

printf '%s\n' '---'
printf 'warnings=%d criticals=%d\n' "$WARNINGS" "$CRITICALS"

if (( CRITICALS > 0 )); then
    exit 2
elif (( WARNINGS > 0 )); then
    exit 1
else
    exit 0
fi
EOF
```

Set ownership/mode:

```bash
sudo chown root:root /usr/local/sbin/day2-healthcheck.sh
sudo chmod 0755 /usr/local/sbin/day2-healthcheck.sh
```

---

# 112. Validate health-check syntax

Before execution:

```bash
bash -n /usr/local/sbin/day2-healthcheck.sh
```

No output usually means syntax parsing succeeded.

Đây là example của:

```text
validate before execute
```

---

# 113. Run health check

```bash
sudo /usr/local/sbin/day2-healthcheck.sh
```

Example:

```text
OK: service httpd.service is active
OK: service day2-heartbeat.service is active
OK: TCP/80 is listening
OK: root filesystem usage is 35%
OK: root inode usage is 8%
OK: DNS/NSS resolution works for localhost
OK: local HTTP functional test succeeded
---
warnings=0 criticals=0
```

---

# 114. Check exit code correctly

Immediately after:

```bash
echo $?
```

Expected:

```text
0
```

Be careful:

```bash
sudo /usr/local/sbin/day2-healthcheck.sh
date
echo $?
```

will show `date`'s exit code, not script's.

Correct pattern:

```bash
sudo /usr/local/sbin/day2-healthcheck.sh
RC=$?
echo "healthcheck_rc=$RC"
```

---

# 115. Test negative case

Stop heartbeat service:

```bash
sudo systemctl stop day2-heartbeat.service
```

Run:

```bash
sudo /usr/local/sbin/day2-healthcheck.sh
RC=$?
echo "$RC"
```

Expected:

```text
CRITICAL: service day2-heartbeat.service is not active
```

Final exit code:

```text
2
```

Recover:

```bash
sudo systemctl start day2-heartbeat.service
```

---

# 116. Why script does not use `set -e`

Ta deliberately không dùng:

```bash
set -e
```

vì health-check cần cho phép individual checks fail rồi **report aggregate health**.

Nếu shell exit ngay ở first failure, script không collect được full health picture.

Đây là ví dụ cho thấy:

> Không copy Bash “strict mode” mechanically mà phải hiểu desired behavior.

---

# 117. Health-check evidence

```bash
sudo /usr/local/sbin/day2-healthcheck.sh \
  | tee "$EVID/100-healthcheck-output.txt"

echo "${PIPESTATUS[0]}" \
  | tee "$EVID/101-healthcheck-exit-code.txt"
```

Tại sao dùng:

```bash
PIPESTATUS[0]
```

?

Vì command đang nằm trong pipeline:

```text
healthcheck | tee
```

`$?` đơn giản sau pipeline có thể phản ánh command cuối là `tee`.

Đây là một subtle but important Bash lesson.

---

# 118. Lab Track 11 — Backup Data Preparation

Tạo controlled source dataset.

```bash
sudo mkdir -p /srv/day2data
```

```bash
echo 'configuration-A' \
  | sudo tee /srv/day2data/config-a.txt

echo 'configuration-B' \
  | sudo tee /srv/day2data/config-b.txt
```

Create nested directory:

```bash
sudo mkdir -p /srv/day2data/subdir
```

```bash
date -Is \
  | sudo tee /srv/day2data/subdir/timestamp.txt
```

List:

```bash
find /srv/day2data \
  -type f \
  -printf '%P\n'
```

---

# 119. Source checksum baseline

Compute:

```bash
cd /srv/day2data
sudo find . \
  -type f \
  -print0 \
  | sort -z \
  | sudo xargs -0 sha256sum
```

Save:

```bash
cd /srv/day2data

sudo find . \
  -type f \
  -print0 \
  | sort -z \
  | sudo xargs -0 sha256sum \
  | sudo tee /var/tmp/day2-source-sha256.txt
```

This lets restore verification compare content.

---

# 120. Backup destination

Create:

```bash
sudo mkdir -p /var/backups/day2
```

Permissions:

```bash
sudo chmod 0700 /var/backups/day2
```

Lab simplification:

```text
source and backup destination are on same VM
```

This is **not sufficient enterprise disaster protection**.

Nếu VM/disk mất toàn bộ thì cả source và backup có thể mất.

Mục tiêu Day 2 ở đây là học mechanism và verification.

---

# 121. Manual `tar` backup first

Trước khi automate, hiểu manual workflow.

```bash
sudo tar \
  -C /srv/day2data \
  -czf /var/backups/day2/manual-test.tar.gz \
  .
```

Ý nghĩa:

```text
-C /srv/day2data
→ change directory before archiving

-c
→ create archive

-z
→ gzip compression

-f
→ archive filename follows
```

---

# 122. Verify archive contents

```bash
sudo tar \
  -tzf \
  /var/backups/day2/manual-test.tar.gz
```

Bạn phải thấy:

```text
./
./config-a.txt
./config-b.txt
./subdir/
./subdir/timestamp.txt
```

Nếu archive command success nhưng expected files absent:

```text
backup is not correct
```

---

# 123. Hash backup artifact

```bash
sudo sha256sum \
  /var/backups/day2/manual-test.tar.gz
```

Checksum có thể dùng để:

```text
verify later file integrity
detect artifact alteration/corruption
document evidence
```

Nó không tự chứng minh semantic restore correctness.

---

# 124. `rsync` backup exercise

Create mirror target:

```bash
sudo mkdir -p /var/backups/day2/rsync-copy
```

Trước real run, optional dry-run:

```bash
sudo rsync \
  -a \
  --dry-run \
  /srv/day2data/ \
  /var/backups/day2/rsync-copy/
```

Observe changes.

Run:

```bash
sudo rsync \
  -a \
  /srv/day2data/ \
  /var/backups/day2/rsync-copy/
```

Verify:

```bash
find /var/backups/day2/rsync-copy \
  -type f \
  -printf '%P\n'
```

---

# 125. Trailing slash lesson

These can differ semantically:

```bash
rsync -a /source/ /destination/
```

versus:

```bash
rsync -a /source /destination/
```

Bạn phải test and understand desired directory structure.

Không copy an Internet `rsync` command containing destructive options blindly.

Đặc biệt các options xóa destination content cần change review.

---

# 126. Bash Backup Script

Bây giờ automate `tar` workflow.

### [CHANGE]

```bash
sudo tee /usr/local/sbin/day2-backup.sh >/dev/null <<'EOF'
#!/usr/bin/env bash

set -u
set -o pipefail

SOURCE="/srv/day2data"
BACKUP_DIR="/var/backups/day2"
TIMESTAMP="$(date +%Y%m%d-%H%M%S)"
ARCHIVE="$BACKUP_DIR/day2-$TIMESTAMP.tar.gz"
TEMP_ARCHIVE="$ARCHIVE.tmp"
CHECKSUM="$ARCHIVE.sha256"

log() {
    printf '%s %s\n' "$(date -Is)" "$*"
}

fail() {
    log "ERROR: $*"
    exit 1
}

if [[ ! -d "$SOURCE" ]]; then
    fail "source directory does not exist: $SOURCE"
fi

if [[ ! -d "$BACKUP_DIR" ]]; then
    fail "backup directory does not exist: $BACKUP_DIR"
fi

if [[ ! -w "$BACKUP_DIR" ]]; then
    fail "backup directory is not writable: $BACKUP_DIR"
fi

log "starting backup source=$SOURCE archive=$ARCHIVE"

if ! tar \
    -C "$SOURCE" \
    -czf "$TEMP_ARCHIVE" \
    .
then
    rm -f "$TEMP_ARCHIVE"
    fail "tar archive creation failed"
fi

if ! tar -tzf "$TEMP_ARCHIVE" >/dev/null; then
    rm -f "$TEMP_ARCHIVE"
    fail "archive verification failed"
fi

if ! mv "$TEMP_ARCHIVE" "$ARCHIVE"; then
    rm -f "$TEMP_ARCHIVE"
    fail "cannot finalize archive"
fi

if ! sha256sum "$ARCHIVE" > "$CHECKSUM"; then
    fail "cannot create checksum"
fi

log "backup completed archive=$ARCHIVE checksum=$CHECKSUM"
exit 0
EOF
```

Permissions:

```bash
sudo chown root:root /usr/local/sbin/day2-backup.sh
sudo chmod 0750 /usr/local/sbin/day2-backup.sh
```

---

# 127. Why temporary archive first?

Script creates:

```text
day2-....tar.gz.tmp
```

first.

Only after verification does it:

```bash
mv
```

to final archive.

Điều này giảm khả năng người/operator/tool nhìn thấy một partially created archive với final production-style filename.

Concept:

```text
create temporary
     ↓
verify
     ↓
promote to final
```

---

# 128. Script validation

Syntax:

```bash
sudo bash -n /usr/local/sbin/day2-backup.sh
```

Run:

```bash
sudo /usr/local/sbin/day2-backup.sh
```

Example:

```text
2026-10-18T... starting backup ...
2026-10-18T... backup completed ...
```

Exit:

```bash
echo $?
```

Expected:

```text
0
```

---

# 129. Verify generated backup

```bash
sudo ls -lh /var/backups/day2/
```

Find latest:

```bash
sudo ls -1t /var/backups/day2/day2-*.tar.gz \
  | head -n 1
```

Set:

```bash
LATEST=$(
    sudo ls -1t /var/backups/day2/day2-*.tar.gz |
    head -n 1
)
```

List:

```bash
sudo tar -tzf "$LATEST"
```

---

# 130. Verify checksum

Checksum file:

```bash
sudo cat "${LATEST}.sha256"
```

Because stored checksum may contain full filename, verify from appropriate path/environment:

```bash
sudo sha256sum "$LATEST"
```

Compare output with recorded checksum.

---

# 131. Negative backup test

Deliberately make source path invalid **by using a temporary copy of the script**, not modifying production-like lab script without backup.

```bash
sudo cp -a \
  /usr/local/sbin/day2-backup.sh \
  /usr/local/sbin/day2-backup-failure-test.sh
```

Change:

```text
SOURCE="/srv/day2data"
```

to:

```text
SOURCE="/srv/DOES-NOT-EXIST"
```

Run:

```bash
sudo /usr/local/sbin/day2-backup-failure-test.sh
```

Expected:

```text
ERROR: source directory does not exist
```

Exit:

```bash
echo $?
```

Expected nonzero.

Remove failure-test script afterward.

---

# 132. Why negative testing matters

Một automation chưa được test failure path chưa thực sự được validate.

Bạn phải test:

```text
happy path
missing source
unwritable destination
corrupt/invalid archive scenario where practical
```

Day 12–13 sẽ mở rộng automation testing sâu hơn.

---

# 133. Lab Track 12 — Restore

Syllabus yêu cầu:

> validate backup and restore results. Infrastructure_Operations_14_Da…

Đây là mandatory.

Backup không kết thúc ở:

```text
archive exists
```

---

# 134. Isolated restore directory

Không restore đè source trước.

Create:

```bash
sudo mkdir -p /srv/day2-restore
```

Clean controlled directory:

```bash
sudo rm -rf /srv/day2-restore/*
```

Ở đây path đã hard-coded và isolated trong lab.

Trong automation thực tế, `rm -rf` phải được kiểm soát chặt hơn.

---

# 135. Restore archive

```bash
sudo tar \
  -C /srv/day2-restore \
  -xzf "$LATEST"
```

List:

```bash
find /srv/day2-restore \
  -type f \
  -printf '%P\n'
```

Expected same files as source.

---

# 136. Content comparison

Simple recursive comparison:

```bash
sudo diff \
  -r \
  /srv/day2data \
  /srv/day2-restore
```

No output generally indicates no detected differences in compared file content/structure according to `diff`.

---

# 137. Hash comparison

Source:

```bash
cd /srv/day2data

sudo find . \
  -type f \
  -print0 \
  | sort -z \
  | sudo xargs -0 sha256sum \
  > /var/tmp/day2-source-after.sha256
```

Restore:

```bash
cd /srv/day2-restore

sudo find . \
  -type f \
  -print0 \
  | sort -z \
  | sudo xargs -0 sha256sum \
  > /var/tmp/day2-restore.sha256
```

Compare:

```bash
diff \
  /var/tmp/day2-source-after.sha256 \
  /var/tmp/day2-restore.sha256
```

No difference gives stronger file-content evidence.

---

# 138. What restore verification actually proves

You have demonstrated:

```text
backup artifact exists
        ↓
archive readable
        ↓
expected files present
        ↓
files can be extracted
        ↓
restored file set matches expected data
        ↓
checksums match
```

Đó là far stronger than:

```text
tar exited 0
```

---

# 139. Restore is not yet full Disaster Recovery

Cần hiểu scope.

Day 2 restore validates:

```text
file-level basic recovery
```

Nó không chứng minh:

```text
full operating system DR
database consistency
application-consistent recovery
cross-site disaster recovery
RPO/RTO compliance
```

Các Day sau mới mở rộng sang AWS backup/DR và HYCU.

---

# 140. Lab Track 13 — `cron` Scheduling

Syllabus yêu cầu schedule execution bằng `cron`.

Ta schedule:

```text
health check
backup
```

Không schedule quá nhiều production-like backup jobs.

---

# 141. Verify `crond`

```bash
systemctl status crond --no-pager
```

Nếu package installed nhưng service inactive:

### [CHANGE]

```bash
sudo systemctl enable --now crond
```

Verify:

```bash
systemctl is-active crond
systemctl is-enabled crond
```

---

# 142. User crontab vs `/etc/cron.d`

Hai formats quan trọng.

User crontab:

```text
minute hour day month weekday command
```

System file such as `/etc/cron.d/...`:

```text
minute hour day month weekday USER command
```

Đừng copy line từ `/etc/cron.d` vào:

```bash
crontab -e
```

mà giữ username field.

---

# 143. Create cron log files

```bash
sudo touch \
  /var/log/day2-healthcheck.log \
  /var/log/day2-backup.log
```

Permissions:

```bash
sudo chmod 0640 \
  /var/log/day2-healthcheck.log \
  /var/log/day2-backup.log
```

---

# 144. Create lab cron schedule

### [CHANGE]

```bash
sudo tee /etc/cron.d/day2-lab >/dev/null <<'EOF'
*/2 * * * * root /usr/local/sbin/day2-healthcheck.sh >> /var/log/day2-healthcheck.log 2>&1
*/5 * * * * root /usr/local/sbin/day2-backup.sh >> /var/log/day2-backup.log 2>&1
EOF
```

Permissions:

```bash
sudo chown root:root /etc/cron.d/day2-lab
sudo chmod 0644 /etc/cron.d/day2-lab
```

Đây là **lab cadence** để dễ quan sát, không phải recommendation production.

Production schedule phải theo:

```text
backup window
resource load
RPO
retention
maintenance schedule
```

---

# 145. Cron field interpretation

Example:

```text
*/2 * * * *
```

means every two minutes.

```text
*/5 * * * *
```

means every five minutes.

Không nhầm:

```text
minute
hour
day-of-month
month
day-of-week
```

---

# 146. Verify cron configuration

Inspect:

```bash
sudo cat /etc/cron.d/day2-lab
```

Check service:

```bash
systemctl is-active crond
```

Check later-generated output:

```bash
sudo tail -n 30 /var/log/day2-healthcheck.log
```

and:

```bash
sudo tail -n 30 /var/log/day2-backup.log
```

---

# 147. Cron execution evidence

Depending RHEL logging configuration:

```bash
sudo grep day2 /var/log/cron
```

may work.

Or:

```bash
journalctl \
  -u crond \
  --since "-20 minutes" \
  --no-pager
```

Do not assume `/var/log/cron` exists in all configurations; RHEL docs explicitly note actual `/var/log` files depend on system/services configuration. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html-single/configuring_basic_system_settings/index?utm_source=chatgpt.com)

---

# 148. Verify result, not just trigger

A `cron` entry being executed does **not** prove job succeeded.

For backup verify:

```bash
sudo ls -lt /var/backups/day2/ | head
```

Check archive:

```bash
sudo tar -tzf <latest-archive>
```

Health check:

```bash
sudo tail /var/log/day2-healthcheck.log
```

---

# 149. Cron environment failure lesson

Suppose interactive shell has:

```text
PATH=/custom/bin:/usr/bin:/bin
```

Cron may not.

Therefore script should not rely on mystery interactive setup.

A deliberate optional failure test:

Create script:

```bash
#!/usr/bin/env bash
echo "PATH=$PATH"
```

run manually and through cron, compare outputs.

Do **not** inject secrets into environment output.

---

# 150. `cron` troubleshooting flow

When scheduled job “didn't run”:

```text
Is crond active?
      ↓
Is schedule file valid/present?
      ↓
Correct ownership/mode?
      ↓
Correct user?
      ↓
Correct time/timezone?
      ↓
Was job triggered?
      ↓
Did command path exist?
      ↓
Environment adequate?
      ↓
Permission?
      ↓
SELinux?
      ↓
Script exit code?
      ↓
Expected artifact generated?
```

This is much better than continuously editing schedule.

---

# 151. Integrated Failure Injection Exercises

Syllabus explicitly yêu cầu:

```text
service failure
network failure
storage failure
```

Ta đã tạo từng loại, nhưng đây là cách formalize chúng như incident drills.

---

## Failure Injection 1 — Service startup failure

### Injection

Break:

```text
ExecStart path
```

của `day2-heartbeat.service`.

### Symptom

```text
systemctl status
→ failed
```

### Evidence

```text
systemctl status
journalctl -u
systemctl cat
ls -l executable
```

### Root cause

Invalid `ExecStart`.

### Recovery

Restore unit → `daemon-reload` → `reset-failed` → start.

### Verification

```text
service active
process exists
heartbeat entries return
```

### Escalation boundary

Nếu custom service vẫn crash dù configuration chính xác và log cho thấy application defect/core dump:

```text
escalate application/L2/L3
```

---

## Failure Injection 2 — Network access failure

### Injection

Remove only HTTP service from runtime `firewalld`.

### Symptom

```text
local curl succeeds
remote curl fails
```

### Checks

```text
IP
route
service
socket
local application
firewall
```

### Root cause

Runtime firewall does not permit HTTP.

### Recovery

Reload permanent desired state hoặc authorized add rule.

### Verification

Remote curl works.

### Escalation

Nếu host firewall permits but traffic vẫn không đến:

```text
network firewall / routing / ACL / upstream team
```

---

## Failure Injection 3 — Storage inode exhaustion

### Injection

Create hundreds of files inside tiny isolated filesystem.

### Symptom

```text
No space left on device
```

while `df -h` still shows space.

### Evidence

```bash
df -h
df -i
```

### Root cause

Inode exhaustion.

### Recovery

Remove approved controlled files.

### Verification

```text
inode free
new file creation succeeds
```

### Escalation

Nếu actual production filesystem accumulates millions of application files:

```text
application owner
capacity/storage team
retention owner
```

depending environment.

---

## Failure Injection 4 — SELinux access denial

### Injection

Apache content in `/srv/day2web` before proper SELinux labeling.

### Symptom

```text
service active
TCP port listening
DAC permissions reasonable
HTTP path access denied
```

### Evidence

```text
getenforce
ls -Z
AVC/audit
HTTP error
```

### Root cause

Incorrect SELinux file context.

### Recovery

```text
semanage fcontext
restorecon
```

### Verification

SELinux remains:

```text
Enforcing
```

and HTTP access succeeds.

### Critical lesson

Correct fix is **not** disabling SELinux.

---

## Failure Injection 5 — Optional scheduled-job failure

Temporarily use invalid backup source in a test copy.

### Symptom

Cron/job log:

```text
source directory does not exist
```

### Root cause

Invalid configured source path.

### Verification

Nonzero exit.

### Recovery

Correct source.

### Reverify

Archive appears and validates.

---

# 152. Integrated Incident Drill

Đây là bài cuối nên thực hiện mà không nhìn đáp án từng bước.

Scenario:

```text
Ticket:
After maintenance, users cannot access
http://rhel-day2.lab.example/day2/

Monitoring reports application unavailable.

SSH remains accessible.
```

Nhiệm vụ:

```text
restore service without exceeding authorization
collect evidence
identify root cause
record corrective action
```

---

# 153. Step 1 — Record ticket context

Record:

```text
Ticket ID: INC-DAY2-001
Environment: LAB
Impact: client cannot access HTTP application
Start time: <timestamp>
Operator: <lab-user>
Change reference: CHG-DAY2-001
```

---

# 154. Step 2 — Identity

```bash
date -Is
hostnamectl
```

Make sure:

```text
correct host
correct timeline
```

---

# 155. Step 3 — OS health

```bash
uptime
```

```bash
df -h
```

```bash
df -i
```

Look for obvious resource/capacity issue.

---

# 156. Step 4 — Service

```bash
systemctl status httpd --no-pager
```

Question:

```text
active?
failed?
inactive?
```

---

# 157. Step 5 — Process

```bash
ps -ef | grep '[h]ttpd'
```

Do actual process instances exist?

---

# 158. Step 6 — Port

```bash
sudo ss -lntp | grep ':80 '
```

Is expected port listening?

---

# 159. Step 7 — Local functional test

```bash
curl \
  --connect-timeout 3 \
  -i \
  http://127.0.0.1/day2/
```

This isolates remote network.

---

# 160. Step 8 — Server IP test

Get address:

```bash
ip -brief address
```

Test:

```bash
curl \
  --connect-timeout 3 \
  http://<SERVER_IP>/day2/
```

---

# 161. Step 9 — DNS

```bash
getent hosts rhel-day2.lab.example
```

Does it return expected server IP?

---

# 162. Step 10 — Route

```bash
ip route
```

Client path if relevant:

```bash
ip route get <CLIENT_IP>
```

---

# 163. Step 11 — Firewall

```bash
sudo firewall-cmd --get-active-zones
```

Then correct zone:

```bash
sudo firewall-cmd \
  --zone=<ZONE> \
  --list-all
```

Is `http` permitted?

---

# 164. Step 12 — SELinux

If local `/day2/` returns access error:

```bash
getenforce
ls -Zd /srv/day2web
```

Then inspect relevant audit denial.

---

# 165. Step 13 — Logs

```bash
journalctl \
  -u httpd \
  --since "-15 minutes" \
  --no-pager
```

Look at timeline.

---

# 166. Root-cause hierarchy

Based on findings:

```text
service inactive
→ systemd/application branch

service active, port absent
→ process/socket/config branch

port listening, local curl fails
→ application/config/security/storage branch

local works, remote IP fails
→ firewall/network branch

IP works, hostname fails
→ DNS branch

403 + AVC
→ SELinux/context branch
```

This is the core reasoning skill of Day 2.

---

# 167. Troubleshooting record format

Final record should look approximately:

| Field                 | Example                                                      |
| --------------------- | ------------------------------------------------------------ |
| Incident              | INC-DAY2-001                                                 |
| Host                  | rhel-day2                                                    |
| Environment           | LAB                                                          |
| Symptom               | HTTP `/day2/` inaccessible remotely                          |
| Start                 | 2026-...                                                     |
| Initial service state | `active`                                                     |
| Socket                | TCP/80 `LISTEN`                                              |
| Local test            | Successful                                                   |
| Remote test           | Failed                                                       |
| Firewall              | HTTP missing from runtime zone                               |
| Root cause            | Runtime firewall state differed from desired permanent state |
| Correction            | Reloaded approved permanent configuration                    |
| Verification          | Remote HTTP 200                                              |
| Evidence              | firewall before/after, curl before/after, service state      |
| Rollback              | Reapply prior runtime state if required                      |
| Escalation            | Not required / or destination team                           |

---

# 168. Evidence quality standards

Evidence phải tránh hai extremes.

### Quá ít

```text
"Fixed."
```

Không đủ.

### Quá nhiều

```text
Full 2 GB /var/log dump
```

cũng không tốt.

Good evidence:

```text
relevant
timestamped
before/after
minimal
reproducible
secure
```

---

# 169. Evidence package structure đề xuất

Đến cuối lab có thể có:

```text
day2-evidence/
├── 00-timestamp.txt
├── 01-hostnamectl.txt
├── 02-os-release.txt
├── 10-ip-address-before.txt
├── 11-route-before.txt
├── 13-firewall-zones-before.txt
├── 15-selinux-mode-before.txt
├── 17-lsblk-before.txt
├── 30-dnf-repositories.txt
├── 31-package-verification.txt
├── 40-day2-service-process.txt
├── 41-service-status-good.txt
├── 51-failed-service-status.txt
├── 52-failed-service-journal.txt
├── 54-service-recovered.txt
├── 60-firewall-http-enabled.txt
├── 70-selinux-mode.txt
├── 71-selinux-context-directory.txt
├── 73-selinux-functional-test.txt
├── 81-lsblk-after.txt
├── 82-findmnt-day2data.txt
├── 84-fstab-day2-entry.txt
├── 90-heartbeat-journal.txt
├── 91-httpd-journal.txt
├── 100-healthcheck-output.txt
├── 101-healthcheck-exit-code.txt
└── troubleshooting-record.txt
```

---

# 170. Evidence integrity

Sau khi hoàn thành:

```bash
cd "$EVID"
```

Generate checksums:

```bash
sha256sum * > SHA256SUMS.txt
```

Đây là knowledge bổ sung vượt nhẹ Day 2 nhưng chuẩn bị cho Day 13 audit evidence integrity.

Lưu ý nếu file names có spaces hoặc subdirectories, production-grade hashing workflow cần robust hơn; lab hiện giữ filename đơn giản.

---

# 171. Package evidence bundle

Optional archive:

```bash
cd "$(dirname "$EVID")"

tar \
  -czf "$(basename "$EVID").tar.gz" \
  "$(basename "$EVID")"
```

Đây là demonstration thêm về compression/archive.

Không coi archive này là long-term secure evidence repository nếu policy tổ chức yêu cầu centralized storage.

---

# 172. Cleanup / Rollback Plan

Một lab enterprise-style không kết thúc với environment đầy test objects.

---

## 172.1. Remove cron lab schedule

```bash
sudo rm -f /etc/cron.d/day2-lab
```

Không cần disable `crond` nếu nó là legitimate system service.

---

## 172.2. Remove custom service

Stop:

```bash
sudo systemctl stop day2-heartbeat.service
```

Disable:

```bash
sudo systemctl disable day2-heartbeat.service
```

Remove unit:

```bash
sudo rm -f \
  /etc/systemd/system/day2-heartbeat.service \
  /etc/systemd/system/day2-heartbeat.service.pre-failure
```

Reload:

```bash
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

Remove script:

```bash
sudo rm -f /usr/local/sbin/day2-heartbeat.sh
```

Optional user cleanup:

```bash
sudo userdel day2svc
```

chỉ nếu user tạo riêng cho lab và không còn dependency.

---

# 173. SELinux cleanup

Remove Apache lab configuration:

```bash
sudo rm -f /etc/httpd/conf.d/day2-lab.conf
```

Validate:

```bash
sudo apachectl configtest
```

Remove custom SELinux mapping:

```bash
sudo semanage fcontext \
  -d \
  '/srv/day2web(/.*)?'
```

Then if directory still exists and you want default labeling:

```bash
sudo restorecon -Rv /srv/day2web
```

Sau đó remove controlled content nếu required:

```bash
sudo rm -rf /srv/day2web
```

Vì path ở đây là lab-specific, fixed path. Trong production không chạy recursive deletion mà không re-verify target.

---

# 174. Firewall cleanup

Nếu `http` chỉ được added vì lab:

```bash
sudo firewall-cmd \
  --zone=public \
  --permanent \
  --remove-service=http
```

Then:

```bash
sudo firewall-cmd --reload
```

**Nhưng chỉ làm nếu bạn xác nhận rule không tồn tại trước lab.**

Đây là lý do ta capture baseline.

Rollback không có nghĩa:

```text
remove everything we changed
```

mà là:

```text
restore original desired state
```

---

# 175. Storage cleanup

Trước tiên preserve evidence/data nếu cần.

Unmount:

```bash
sudo umount /mnt/day2data
```

Remove corresponding `/etc/fstab` entry **carefully**.

Không dùng broad `sed` pattern có khả năng xóa unrelated entries.

Compare với backup.

Sau edit:

```bash
sudo systemctl daemon-reload
```

Verify:

```bash
grep '/mnt/day2data' /etc/fstab
```

should no longer return lab entry.

Disk partition itself chỉ wipe/remove nếu lab cleanup explicitly requires it.

Không cần destructive cleanup chỉ để “đẹp”.

---

# 176. Backup data cleanup

Nếu authorized:

```bash
sudo rm -rf /var/backups/day2
```

và lab directories:

```bash
sudo rm -rf /srv/day2data
sudo rm -rf /srv/day2-restore
```

Trước mỗi recursive deletion:

```bash
ls -ld <target>
```

và verify exact path.

---

# 177. Post-cleanup verification

Run:

```bash
systemctl status day2-heartbeat.service
```

should indicate unit no longer exists if removed.

Firewall:

```bash
sudo firewall-cmd --list-all
```

Mount:

```bash
findmnt /mnt/day2data
```

SELinux custom mapping can be reviewed if appropriate.

Ensure no test loop mounts:

```bash
findmnt | grep day2
```

---

# 178. Troubleshooting Checkpoint — Package install fails

Suppose:

```text
No match for argument
```

Check sequentially:

```text
Correct package name?
        ↓
RHEL release?
        ↓
Repository enabled?
        ↓
Package available?
        ↓
Subscription/RHUI/Satellite state?
```

Commands:

```bash
dnf repolist
dnf info <package>
```

Do not immediately add random Internet repository.

---

# 179. Troubleshooting Checkpoint — Service won't start

Flow:

```text
systemctl status
        ↓
journalctl -u service
        ↓
systemctl cat service
        ↓
ExecStart exists/executable?
        ↓
configuration syntax
        ↓
permissions
        ↓
SELinux
        ↓
dependencies
        ↓
resource/storage
```

Do not reboot before understanding failure unless runbook/incident recovery specifically requires it.

---

# 180. Troubleshooting Checkpoint — Remote service unreachable

Flow:

```text
systemctl
↓
process
↓
ss
↓
local curl
↓
server IP
↓
firewalld
↓
route
↓
remote client
↓
DNS
↓
external firewall/load balancer if applicable
```

This order gradually expands failure domain.

---

# 181. Troubleshooting Checkpoint — Permission denied

Potential layers:

```text
owner/group
mode
ACL if applicable
SELinux context
application user
filesystem mount options
```

Never assume:

```text
Permission denied = chmod 777
```

`chmod 777` is a classic unsafe shortcut.

---

# 182. Troubleshooting Checkpoint — Filesystem full

Always inspect both:

```bash
df -h
```

and:

```bash
df -i
```

Then identify consumption.

Do not start deleting files indiscriminately.

Ask:

```text
Which directory?
Which application owns data?
Retention policy?
Can files safely be removed?
Is this temporary or recurrent?
```

---

# 183. Troubleshooting Checkpoint — Cron script works manually but not scheduled

Check:

```text
User
PATH
working directory
environment variables
permissions
SELinux
script interpreter
absolute paths
cron logs
exit code
destination permissions
```

Classic error:

```bash
backup-tool
```

exists in interactive `PATH` but not cron environment.

---

# 184. Troubleshooting Checkpoint — Backup exists but restore fails

Potential causes:

```text
Archive incomplete
Archive corrupt
Wrong format/options
Permissions
Insufficient restore space
Wrong encryption/compression requirements
Source assumptions
Wrong recovery point
```

This is why restore validation is mandatory.

---

# 185. Escalation Boundaries trong lab

## L1-style responsibility

Bạn nên có thể:

```text
inspect system status
collect logs
check storage/network/service
run approved health check
verify package state
execute documented restart
collect evidence
```

---

## L2-style responsibility

Theo organizational policy, có thể gồm:

```text
controlled package install
systemd unit correction
firewall change
SELinux context correction
fstab correction
script correction
backup/restore
```

nhưng phải có authorization.

---

## Escalate

Escalate khi gặp:

```text
suspected filesystem corruption
kernel panic
unknown recurring kernel error
critical production disk modification
unauthorized repository requirement
custom SELinux policy redesign
complex routing problem
suspected security compromise
repeated unexplained service crash
data integrity uncertainty
restore affecting production data
insufficient authorization
```

---

# 186. Good escalation record

Không viết:

```text
Linux server broken. Please check.
```

Viết:

```text
Incident: INC-DAY2-004
Host: rhel-day2
Environment: LAB

Symptom:
httpd is active and TCP/80 is LISTEN.
Local /day2/ request returns HTTP 403.

Verified:
- Host networking healthy
- firewalld permits HTTP
- DAC permissions on /srv/day2web appear readable
- SELinux is Enforcing
- /srv/day2web has var_t context
- AVC denial corresponds to httpd access attempt

Root-cause hypothesis:
Non-standard Apache content path has incorrect SELinux context.

Action:
No security control disabled.

Requested:
Approval/L2 action to apply persistent supported
httpd_sys_content_t mapping if this were outside operator scope.
```

Đó là high-quality escalation.

---

# 187. Lab deliverables bắt buộc

Theo syllabus, cuối Module Thực hành Day 2 bạn phải có evidence chứng minh được ít nhất các capability sau. Infrastructure_Operations_14_Da…

| Capability              | Evidence nên có                 |
| ----------------------- | ------------------------------- |
| Package installation    | `dnf`/`rpm` verification        |
| Repository awareness    | `dnf repolist`                  |
| systemd management      | status/start/enable             |
| Process inspection      | PID/user/state/resource output  |
| Environment variables   | service environment/output      |
| Failed-service recovery | before/failure/recovery         |
| IP/network              | `ip address`, `ip route`        |
| DNS                     | `getent`/`dig` result           |
| TCP/UDP awareness       | `ss`                            |
| Firewall                | before/after rule + client test |
| SELinux                 | mode/context/AVC/correction     |
| Disk/partition          | `lsblk`                         |
| Filesystem              | filesystem creation/mount       |
| `/etc/fstab`            | persistent entry/test           |
| Capacity                | `df -h`                         |
| Inodes                  | `df -i`                         |
| Logs                    | `journalctl` extracts           |
| Text processing         | `grep`/`awk`/`sed` output       |
| Bash                    | health check                    |
| Exit codes              | positive/negative cases         |
| Backup                  | archive + checksum              |
| `rsync`                 | synchronized dataset            |
| Cron                    | schedule + execution log        |
| Restore                 | isolated restore + comparison   |
| Service failure         | diagnosis/recovery              |
| Network failure         | diagnosis/recovery              |
| Storage failure         | diagnosis/recovery              |
| Escalation              | structured record               |

---

# 188. Day 2 Practical Operational Flow phải thuộc

Cuối lab, hãy hình dung một server issue theo flow:

```text
                USER REPORT
                     |
                     v
              Identify symptom
                     |
                     v
             Capture timestamp
                     |
                     v
            Confirm correct host
                     |
                     v
              Check resources
            /        |        \
          CPU      Storage   Network
                    |          |
                 df -h/i    ip/route
                    |          |
                    +----+-----+
                         |
                         v
                    systemd
                         |
                         v
                      process
                         |
                         v
                       socket
                         |
                         v
                 local function
                         |
                +--------+--------+
                |                 |
             SELinux           firewall
                |                 |
                +--------+--------+
                         |
                         v
                   remote test
                         |
                         v
                       DNS
                         |
                         v
                       logs
                         |
                         v
                   root cause
                         |
                         v
              controlled correction
                         |
                         v
                   verification
                         |
                         v
                 evidence package
                         |
                         v
               closure / escalation
```

---

# 189. Day 2 Practical Cheat Sheet

Một số command cốt lõi bạn phải quen tay sau lab:

```bash
# Package
dnf repolist
dnf info <package>
rpm -q <package>
rpm -qi <package>
rpm -ql <package>
rpm -qf <file>

# systemd
systemctl status <service>
systemctl is-active <service>
systemctl is-enabled <service>
systemctl start <service>
systemctl stop <service>
systemctl restart <service>
systemctl enable <service>
systemctl daemon-reload
systemctl reset-failed <service>

# Process
ps -ef
ps -o pid,ppid,user,stat,%cpu,%mem,etime,cmd -p <PID>

# Logs
journalctl -u <service>
journalctl -u <service> --since "-10 minutes"
journalctl -b
journalctl -t <identifier>

# Networking
ip -brief address
ip route
ip route get <destination>
ping
tracepath
traceroute
ss -lntp
curl

# DNS
getent hosts <hostname>
dig <hostname>

# Firewall
firewall-cmd --get-active-zones
firewall-cmd --zone=<zone> --list-all
firewall-cmd --zone=<zone> --query-service=http

# SELinux
getenforce
sestatus
ls -Z
ausearch -m AVC -ts recent
semanage fcontext
restorecon

# Storage
lsblk -f
findmnt
df -hT
df -i
blkid
mount
umount

# Bash
bash -n <script>
echo $?

# Backup
tar -czf
tar -tzf
tar -xzf
rsync -a
sha256sum

# Cron
systemctl status crond
```

Không cần thuộc mọi option ngay lập tức. Điều quan trọng hơn là biết:

> **Command nào trả lời câu hỏi troubleshooting nào.**

---

# 190. Module Thực hành Day 2 — Completion Checklist

Trước khi coi Day 2 Practical hoàn thành, bạn nên có thể tự đánh dấu toàn bộ:

```text
[ ] Tôi biết xác nhận RHEL release và host identity.
[ ] Tôi biết inspect repository trước khi install.
[ ] Tôi cài được approved package và verify bằng RPM database.
[ ] Tôi hiểu package install không đồng nghĩa service đang chạy.
[ ] Tôi tạo/quản lý được systemd service.
[ ] Tôi phân biệt active và enabled.
[ ] Tôi inspect được PID, PPID, owner và process state.
[ ] Tôi chứng minh được service environment khác shell environment.
[ ] Tôi recover được deliberately failed service bằng evidence.
[ ] Tôi dùng được ip address/ip route.
[ ] Tôi phân biệt IP connectivity và DNS resolution.
[ ] Tôi dùng được ping, tracepath/traceroute, curl và ss.
[ ] Tôi xác định được listening socket.
[ ] Tôi cấu hình/test được controlled firewalld permission.
[ ] Tôi phân biệt runtime và permanent firewall state.
[ ] Tôi troubleshoot được remote failure trong khi local service vẫn healthy.
[ ] Tôi hiểu SELinux Enforcing và không disable SELinux để workaround.
[ ] Tôi inspect và correct được SELinux context trong lab.
[ ] Tôi identify đúng disk/partition/filesystem/mount.
[ ] Tôi tạo filesystem trên dedicated LAB disk mà không chọn nhầm device.
[ ] Tôi configure persistent mount bằng UUID.
[ ] Tôi test /etc/fstab trước reboot.
[ ] Tôi phân biệt block-capacity exhaustion với inode exhaustion.
[ ] Tôi dùng journalctl để xây incident timeline.
[ ] Tôi filter log bằng grep.
[ ] Tôi extract fields bằng awk.
[ ] Tôi transform text bằng sed.
[ ] Tôi viết được Bash health-check.
[ ] Tôi sử dụng exit code đúng.
[ ] Tôi test cả successful và failed execution.
[ ] Tôi tạo được tar backup.
[ ] Tôi verify archive content.
[ ] Tôi dùng rsync trong controlled workflow.
[ ] Tôi tạo checksum cho backup artifact.
[ ] Tôi schedule job bằng cron.
[ ] Tôi verify job thực sự chạy chứ không chỉ tồn tại trong crontab.
[ ] Tôi restore backup vào isolated directory.
[ ] Tôi compare restored data với source.
[ ] Tôi troubleshoot được service failure.
[ ] Tôi troubleshoot được network/firewall failure.
[ ] Tôi troubleshoot được storage/inode failure.
[ ] Tôi collect được before/failure/after evidence.
[ ] Tôi biết cleanup/rollback.
[ ] Tôi biết khi nào phải escalate.
[ ] Tôi tạo được Linux troubleshooting and escalation record.
```

Nếu bạn thực sự làm được **tất cả** các mục này mà không cần đoán command ngẫu nhiên, thì bạn đã hoàn thành đúng phần **Assignment/Lab Day 2** mà syllabus đặt ra. Infrastructure_Operations_14_Da…

Điểm quan trọng nhất của Day 2 là chuyển từ tư duy **“biết dùng Linux commands”** sang tư duy **“vận hành RHEL có kiểm soát”**:

```text
Observe
→ Verify
→ Diagnose
→ Change safely
→ Verify function
→ Preserve evidence
→ Roll back when needed
→ Escalate with useful information
```

Đó cũng là nền móng trực tiếp cho các Day sau: Windows Server operations, AWS, Nutanix, network/firewall, centralized logging, ITSM, backup/DR và automation.
