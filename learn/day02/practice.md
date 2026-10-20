# Day 2 — Module Thực hành

## RHEL Operations Lab: Administration, Troubleshooting, Maintenance, Backup/Restore & Evidence

Module Thực hành Day 2 dưới đây được xây dựng trực tiếp từ Assignment/Lab trong syllabus: cài và verify approved packages; quản lý `systemd` service; inspect processes và environment variables; recover một deliberately failed service; kiểm tra network/DNS/firewall/SELinux; quản lý disk/filesystem/mount/capacity; extract/filter logs; viết Bash health-check và backup script với exit-code handling; schedule bằng `cron`; verify backup/restore; inject service/network/storage failures; thu thập diagnostic evidence và tạo troubleshooting/escalation record. Phần additive còn yêu cầu thực hiện một RHEL maintenance ticket, pre/post baseline, bounded UTC log extraction, rotated-log awareness, correlation giữa `sudo`/service event với ticket, counts và hashes. Infrastructure_Operations_14_Da…

Các command liên quan RHEL hiện hành được đối chiếu với tài liệu Red Hat hiện tại. DNF vẫn là package-management workflow trên RHEL 10; `systemctl` quản lý service lifecycle; `firewalld` phân biệt runtime/permanent state; Red Hat khuyến nghị sửa SELinux labeling bằng persistent file-context mapping + `restorecon` thay vì disable SELinux; và `/etc/fstab` persistent mount cần được register/verify cẩn thận. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/installing-rhel-content?utm_source=chatgpt.com)

---

# 1. Mục tiêu của toàn bộ Module Thực hành

Sau module này, bạn phải có khả năng nhận một RHEL server lab và thực hiện chuỗi công việc:

```text
Identify server
      ↓
Capture baseline
      ↓
Verify repository/packages
      ↓
Inspect processes/environment
      ↓
Manage systemd service
      ↓
Diagnose failed service
      ↓
Validate network/DNS/ports
      ↓
Validate firewalld
      ↓
Diagnose SELinux denial
      ↓
Inspect/create filesystem & mount
      ↓
Detect block/inode capacity issues
      ↓
Extract/correlate logs
      ↓
Write health-check automation
      ↓
Create backup
      ↓
Restore and verify
      ↓
Schedule with cron
      ↓
Inject controlled failures
      ↓
Execute maintenance ticket
      ↓
Create audit-quality evidence
      ↓
Rollback / Escalate
```

Mục tiêu không phải chỉ làm command chạy thành công. Mỗi lab phải chứng minh bốn thứ:

```text
Configuration
+
Runtime state
+
Functional verification
+
Evidence
```

---

# 2. Quy ước an toàn

Trong toàn bộ phần dưới, tôi dùng ba ký hiệu:

| Ký hiệu    | Ý nghĩa                                                         |
| ---------- | --------------------------------------------------------------- |
| **[RO]**   | Read-only hoặc diagnostic, không chủ ý thay đổi system state    |
| **[CHG]**  | Thay đổi configuration/runtime state                            |
| **[RISK]** | Có thể gây outage, mất data hoặc boot issue nếu chạy sai target |

Mọi `[CHG]` và `[RISK]` mặc định chỉ áp dụng cho:

> **authorized RHEL lab / non-production environment**.

Đặc biệt không chạy trên production nếu chưa có ticket/change approval:

```text
dnf install/update/remove
systemctl restart/stop
firewall-cmd add/remove
SELinux configuration changes
mount/umount
mkfs.*
partitioning
/etc/fstab modification
cron installation
reboot
```

Không dùng credential thật trong script hoặc evidence.

---

# 3. Lab topology đề xuất

## 3.1 RHEL server

Ví dụ:

```text
Hostname:      rhel-day2
Environment:   LAB
OS:            RHEL 9.x hoặc RHEL 10.x
CPU:           2 vCPU
Memory:        4 GB+
Root disk:     20 GB+
Network:       isolated lab network
SELinux:       Enforcing
firewalld:     active
Access:        SSH + approved sudo
```

Không bắt buộc chính xác cấu hình trên; quan trọng là VM có snapshot hoặc khả năng rebuild.

## 3.2 Client VM

Khuyến nghị thêm:

```text
client-day2
```

trong cùng lab network.

Client được dùng để verify:

```text
server inbound connectivity
firewall allow/deny
HTTP access
DNS resolution
```

Nếu chỉ có một VM, bạn vẫn làm hầu hết lab nhưng firewall validation sẽ yếu hơn vì test localhost không đại diện đầy đủ inbound traffic.

## 3.3 Không cần secondary physical disk

Để tránh chạm nhầm disk, phần storage chính sẽ sử dụng một **filesystem image** an toàn.

Như vậy:

```text
/var/tmp/day2-storage.img
```

sẽ đóng vai trò virtual storage fixture.

Phần partitioning trên disk thật được để thành **optional advanced lab**, chỉ làm khi instructor cấp dedicated secondary disk.

---

# 4. Evidence repository

Tạo một workspace riêng.

**[CHG]**

```bash
mkdir -p ~/day2-evidence/{baseline,packages,processes,systemd,network,selinux,firewall,storage,logs,scripts,backup,restore,maintenance,troubleshooting}
```

Verify:

```bash
find ~/day2-evidence -maxdepth 1 -type d | sort
```

Expected:

```text
.../baseline
.../packages
.../processes
.../systemd
...
```

Không lưu:

```text
password
private key
token
sudo credential
```

vào evidence repository.

---

# 5. Lab 0 — Host identity và baseline

## Objective

Trước bất kỳ change nào, phải trả lời được:

```text
Tôi đang thao tác server nào?
Server thuộc environment nào?
OS release nào?
Kernel nào?
Time/timezone nào?
System hiện healthy tới mức nào?
```

## 5.1 Host identity

**[RO]**

```bash
hostname
hostnamectl
cat /etc/os-release
cat /etc/redhat-release
uname -r
```

Lưu:

```bash
{
  echo "=== hostname ==="
  hostname
  echo
  echo "=== hostnamectl ==="
  hostnamectl
  echo
  echo "=== os-release ==="
  cat /etc/os-release
  echo
  echo "=== kernel ==="
  uname -r
} > ~/day2-evidence/baseline/host_identity.txt
```

### Verification

```bash
cat ~/day2-evidence/baseline/host_identity.txt
```

Bạn phải xác định được ít nhất:

```text
hostname
RHEL major/minor release
architecture
running kernel
```

---

# 6. Time và timezone baseline

Log correlation thất bại rất thường xuyên vì timezone.

**[RO]**

```bash
date
date -u
timedatectl
```

Evidence:

```bash
{
  date
  date -u
  timedatectl
} > ~/day2-evidence/baseline/time_baseline.txt
```

Bạn cần biết rõ:

```text
local timezone
UTC time
NTP synchronization state nếu available
```

Không được ghi evidence chỉ bằng:

```text
10:35
```

Nên có timezone hoặc UTC.

---

# 7. Basic health baseline

**[RO]**

```bash
uptime
free -h
df -h
df -i
findmnt
ip addr
ip route
systemctl --failed
```

Capture:

```bash
{
  echo "=== UPTIME ==="
  uptime

  echo "=== MEMORY ==="
  free -h

  echo "=== FILESYSTEM ==="
  df -h

  echo "=== INODES ==="
  df -i

  echo "=== FAILED UNITS ==="
  systemctl --failed

  echo "=== ROUTES ==="
  ip route
} > ~/day2-evidence/baseline/precheck.txt
```

Điểm quan trọng:

> Nếu pre-check đã có lỗi, không được sau này khẳng định lỗi đó do maintenance change.

---

# 8. Lab 1 — Repository, subscription và package management

## Objective

Thực hành đúng flow:

```text
Repository
→ package discovery
→ approval
→ install
→ RPM verification
→ transaction evidence
```

---

# 9. Kiểm tra DNF repositories

**[RO]**

```bash
dnf repolist
```

Chi tiết hơn:

```bash
dnf repolist --all
```

Không bật repository lạ chỉ để package installation hoạt động.

Red Hat cảnh báo repository không trusted có thể gây security, compatibility và maintainability problems. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/managing_software_with_the_dnf_tool/installing-rhel-content?utm_source=chatgpt.com)

Capture:

```bash
dnf repolist --all > ~/day2-evidence/packages/repositories.txt
```

---

# 10. Subscription awareness

Nếu environment dùng Red Hat subscription trực tiếp và utility có sẵn:

**[RO]**

```bash
command -v subscription-manager
```

Nếu có:

```bash
sudo subscription-manager status
sudo subscription-manager identity
```

Nhưng enterprise có thể dùng:

```text
Satellite
internal mirror
golden image
cloud entitlement
```

nên trạng thái `subscription-manager` không tự động quyết định server “hợp lệ” hay không.

Evidence cần ghi actual model.

---

# 11. Package discovery trước khi install

Lab package đề xuất:

```text
httpd
rsync
```

Không cài ngay.

**[RO]**

```bash
dnf info httpd
dnf info rsync
```

Kiểm tra đã installed:

```bash
rpm -q httpd
rpm -q rsync
```

Possible:

```text
package httpd is not installed
```

Điều này là bình thường.

---

# 12. Approved package installation

**[CHG]**

Giả sử ticket cho phép:

```bash
sudo dnf install httpd rsync
```

Không cần dùng `-y` trong training vì bạn nên đọc transaction summary trước khi accept.

DNF sẽ resolve dependencies từ repositories configured trên host. Đây là behavior được Red Hat document cho RHEL hiện tại. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_software_with_the_dnf_tool/installing-rhel-content?utm_source=chatgpt.com)

Trước khi confirm, đọc:

```text
Installing:
Upgrading:
Dependencies:
Total download size:
Installed size:
```

---

# 13. Package verification

Sau install:

**[RO]**

```bash
rpm -q httpd
rpm -q rsync
```

Chi tiết:

```bash
rpm -qi httpd
```

Files của package:

```bash
rpm -ql httpd | head
```

Tìm owner của executable:

```bash
rpm -qf /usr/sbin/httpd
```

Expected:

```text
httpd-<version>
```

---

# 14. DNF transaction evidence

```bash
dnf history
```

Chi tiết transaction gần nhất:

```bash
sudo dnf history info last
```

Capture:

```bash
rpm -q httpd rsync \
  > ~/day2-evidence/packages/package_versions.txt

sudo dnf history info last \
  > ~/day2-evidence/packages/dnf_transaction.txt
```

---

# 15. Troubleshooting checkpoint — package install fail

Nếu:

```text
Unable to find a match
```

không vội add random repository.

Check:

```bash
dnf repolist
dnf info <package>
```

Sau đó:

```text
DNS?
repository URL reachable?
subscription/content access?
correct RHEL release?
correct architecture?
approved repository enabled?
```

Nếu source repository chưa approved:

> **Escalate**, không tự cấu hình Internet repository.

---

# 16. Lab 2 — Environment variables

## Objective

Thấy trực tiếp khác nhau giữa:

```text
shell variable
exported environment variable
child process environment
systemd environment
cron environment
```

---

# 17. Shell variable

```bash
DAY2_SCOPE="lab"
```

Verify:

```bash
echo "$DAY2_SCOPE"
```

Expected:

```text
lab
```

Nhưng child shell:

```bash
bash -c 'echo "${DAY2_SCOPE-unset}"'
```

Expected:

```text
unset
```

Vì variable chưa export.

---

# 18. Export variable

```bash
export DAY2_SCOPE
```

Test lại:

```bash
bash -c 'echo "${DAY2_SCOPE-unset}"'
```

Expected:

```text
lab
```

Capture:

```bash
env | sort > ~/day2-evidence/processes/environment_exported.txt
```

---

# 19. PATH

```bash
echo "$PATH"
command -v systemctl
command -v curl
command -v rsync
```

Điểm cần rút ra:

```text
interactive PATH
```

không được assume giống:

```text
systemd PATH
cron PATH
```

Vì vậy trong automation quan trọng nên dùng explicit path khi runbook yêu cầu.

---

# 20. Lab 3 — Processes, process states và signals

## Objective

Thực hành:

```text
PID
PPID
process state
process owner
signals
```

---

# 21. Tạo process fixture an toàn

```bash
sleep 600 &
```

Shell trả:

```text
[1] <PID>
```

Lưu PID:

```bash
LAB_PID=$!
echo "$LAB_PID"
```

---

# 22. Inspect process

```bash
ps -o pid,ppid,user,stat,etime,cmd -p "$LAB_PID"
```

Expected gần:

```text
PID  PPID USER STAT ELAPSED CMD
...  ...  user S   ...     sleep 600
```

`S` thường là sleeping.

Process tree:

```bash
ps -ef --forest | grep -A3 -B3 "[s]leep 600"
```

---

# 23. Resource inspection

```bash
top -b -n 1 | head -20
```

Memory:

```bash
free -h
```

Load:

```bash
uptime
```

Không kết luận performance issue từ duy nhất một number.

---

# 24. Graceful signal

Gửi `SIGTERM`:

```bash
kill -TERM "$LAB_PID"
```

Verify:

```bash
ps -p "$LAB_PID"
```

Expected:

```text
no process
```

Không cần:

```bash
kill -9
```

vì process đã graceful terminate.

---

# 25. Evidence

```bash
ps -eo pid,ppid,user,stat,%cpu,%mem,cmd \
  > ~/day2-evidence/processes/process_snapshot.txt
```

---

# 26. Lab 4 — Tạo và quản lý custom systemd service

Lab này đặc biệt quan trọng vì nó kết hợp:

```text
service account
systemd
environment
process
signals
journal
failed-service recovery
```

---

# 27. Tạo dedicated service account

Không chạy service custom dưới root nếu không cần.

Kiểm tra:

```bash
id day2svc
```

Nếu chưa tồn tại:

**[CHG]**

```bash
NOLOGIN=$(command -v nologin)

sudo useradd \
  --system \
  --no-create-home \
  --shell "$NOLOGIN" \
  day2svc
```

Verify:

```bash
id day2svc
```

---

# 28. Tạo service executable

**[CHG]**

```bash
sudo mkdir -p /usr/local/libexec
```

Tạo:

```bash
sudo tee /usr/local/libexec/day2-demo.sh >/dev/null <<'EOF'
#!/usr/bin/env bash

set -u

trap 'echo "$(date -u +%FT%TZ) received termination signal"; exit 0' TERM INT

while true; do
    echo "$(date -u +%FT%TZ) PID=$$ DAY2_MODE=${DAY2_MODE:-unset}"
    sleep 20
done
EOF
```

Permissions:

```bash
sudo chown root:root /usr/local/libexec/day2-demo.sh
sudo chmod 0755 /usr/local/libexec/day2-demo.sh
```

Verify:

```bash
ls -l /usr/local/libexec/day2-demo.sh
```

---

# 29. Tạo systemd unit

**[CHG]**

```bash
sudo tee /etc/systemd/system/day2-demo.service >/dev/null <<'EOF'
[Unit]
Description=Day 2 Operations Demo Service
After=network.target

[Service]
Type=simple
User=day2svc
Group=day2svc
Environment=DAY2_MODE=systemd
ExecStart=/usr/local/libexec/day2-demo.sh
Restart=on-failure
RestartSec=5s
NoNewPrivileges=yes
PrivateTmp=yes

[Install]
WantedBy=multi-user.target
EOF
```

`User`/`Group` cho least privilege.

`NoNewPrivileges=yes` giảm khả năng service giành thêm privilege.

---

# 30. Load unit

**[CHG]**

```bash
sudo systemctl daemon-reload
```

Đây chỉ reload unit definitions; chưa start service.

Verify unit:

```bash
systemctl cat day2-demo.service
```

---

# 31. Start service

**[CHG]**

```bash
sudo systemctl start day2-demo.service
```

Status:

```bash
systemctl status day2-demo.service
```

Expected:

```text
Active: active (running)
```

---

# 32. Verify process

```bash
pgrep -a -u day2svc
```

hoặc:

```bash
ps -u day2svc -o pid,ppid,user,stat,cmd
```

---

# 33. Verify environment thông qua journal

```bash
journalctl -u day2-demo.service -n 10 --no-pager
```

Expected line:

```text
... DAY2_MODE=systemd
```

Như vậy bạn đã chứng minh variable được inject từ `systemd`, không phải từ interactive shell.

---

# 34. Enable at boot

**[CHG]**

```bash
sudo systemctl enable day2-demo.service
```

Verify:

```bash
systemctl is-enabled day2-demo.service
```

Expected:

```text
enabled
```

Red Hat phân biệt `start` và `enable`: `start` ảnh hưởng current runtime; `enable` tạo boot-time relationship. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/managing-system-services-with-systemctl?utm_source=chatgpt.com)

---

# 35. Verify active vs enabled

```bash
systemctl is-active day2-demo.service
systemctl is-enabled day2-demo.service
```

Expected:

```text
active
enabled
```

Đây là hai verification khác nhau.

---

# 36. Service restart

**[CHG]**

```bash
sudo systemctl restart day2-demo.service
```

Sau đó:

```bash
systemctl status day2-demo.service
journalctl -u day2-demo.service -n 20 --no-pager
```

Quan sát:

```text
old PID terminated
new PID started
```

---

# 37. Lab 5 — Controlled service failure

Bây giờ deliberately làm service fail.

Không phá executable.

Ta dùng reversible systemd drop-in.

---

# 38. Capture pre-failure evidence

```bash
systemctl status day2-demo.service --no-pager \
  > ~/day2-evidence/systemd/pre_failure_status.txt

journalctl -u day2-demo.service -n 30 --no-pager \
  > ~/day2-evidence/systemd/pre_failure_journal.txt
```

---

# 39. Inject failure

**[CHG — LAB ONLY]**

```bash
sudo mkdir -p /etc/systemd/system/day2-demo.service.d
```

```bash
sudo tee /etc/systemd/system/day2-demo.service.d/99-failure.conf >/dev/null <<'EOF'
[Service]
ExecStart=
ExecStart=/usr/local/libexec/day2-does-not-exist
Restart=no
EOF
```

Reload:

```bash
sudo systemctl daemon-reload
```

Restart:

```bash
sudo systemctl restart day2-demo.service
```

Expected failure.

---

# 40. Diagnose — không fix ngay

## Step 1 — status

```bash
systemctl status day2-demo.service --no-pager
```

Expected:

```text
failed
```

## Step 2 — journal

```bash
journalctl -u day2-demo.service -n 50 --no-pager
```

Bạn sẽ thấy symptom liên quan missing executable/failed spawn.

## Step 3 — inspect merged unit

```bash
systemctl cat day2-demo.service
```

Ở đây bạn phải thấy drop-in override.

---

# 41. Root-cause statement

Không ghi:

> Service failed.

Đó chỉ là symptom.

Correct:

> `day2-demo.service` failed because the `99-failure.conf` override replaced `ExecStart` with a non-existent executable.

---

# 42. Recovery

Remove faulty override:

```bash
sudo rm /etc/systemd/system/day2-demo.service.d/99-failure.conf
```

Reload:

```bash
sudo systemctl daemon-reload
```

Clear failed state:

```bash
sudo systemctl reset-failed day2-demo.service
```

Start:

```bash
sudo systemctl start day2-demo.service
```

---

# 43. Post-recovery verification

```bash
systemctl is-active day2-demo.service
systemctl status day2-demo.service --no-pager
journalctl -u day2-demo.service -n 20 --no-pager
```

Expected:

```text
active
```

Evidence:

```bash
systemctl status day2-demo.service --no-pager \
  > ~/day2-evidence/systemd/post_recovery_status.txt
```

---

# 44. Lab 6 — HTTP service foundation

Ta dùng Apache `httpd` cho:

```text
systemd
ports
curl
firewalld
SELinux
network
logs
```

---

# 45. Verify package

```bash
rpm -q httpd
```

Nếu chưa installed và approved:

```bash
sudo dnf install httpd
```

---

# 46. Inspect service trước khi start

```bash
systemctl status httpd --no-pager
systemctl is-enabled httpd
```

---

# 47. Start Apache

**[CHG]**

```bash
sudo systemctl start httpd
```

Verify:

```bash
systemctl is-active httpd
```

---

# 48. Verify listening socket

```bash
sudo ss -lntp | grep ':80'
```

Expected gần:

```text
LISTEN ... 0.0.0.0:80 ...
```

Nếu service active nhưng không có port 80 listener:

> chưa thể coi application ready.

---

# 49. Local HTTP verification

```bash
curl -I http://127.0.0.1/
```

Expected thường:

```text
HTTP/1.1 403 Forbidden
```

hoặc:

```text
HTTP/1.1 200 OK
```

tùy document root/configuration.

Quan trọng là TCP + HTTP response tồn tại.

---

# 50. Lab 7 — SELinux controlled access failure

Đây là một lab cực kỳ quan trọng.

Mục tiêu:

```text
Unix permissions correct
+
Apache configuration correct
+
SELinux blocks access
```

Sau đó bạn chẩn đoán và fix đúng.

Red Hat dùng chính scenario custom web content dưới `/srv` để minh họa SELinux labeling; persistent solution là `semanage fcontext` + `restorecon`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/troubleshooting-problems-related-to-selinux?utm_source=chatgpt.com)

---

# 51. Verify SELinux baseline

```bash
getenforce
sestatus
```

Expected:

```text
Enforcing
```

Nếu lab đang `Permissive`, failure này có thể không reproduce đúng.

Không disable SELinux.

---

# 52. Create custom content

**[CHG]**

```bash
sudo mkdir -p /srv/day2web
```

```bash
echo "Day 2 SELinux Lab - $(hostname)" \
  | sudo tee /srv/day2web/index.html
```

Unix permissions:

```bash
sudo chmod 0755 /srv/day2web
sudo chmod 0644 /srv/day2web/index.html
```

Verify:

```bash
ls -ld /srv/day2web
ls -l /srv/day2web/index.html
```

---

# 53. Inspect SELinux context

```bash
ls -Zd /srv/day2web
ls -Z /srv/day2web/index.html
```

Thông thường dưới `/srv`, context không phải `httpd_sys_content_t`.

---

# 54. Configure Apache Alias

**[CHG]**

```bash
sudo tee /etc/httpd/conf.d/day2-lab.conf >/dev/null <<'EOF'
Alias /day2/ "/srv/day2web/"

<Directory "/srv/day2web/">
    Require all granted
</Directory>
EOF
```

---

# 55. Validate Apache config trước restart

Đừng restart trước syntax check.

```bash
sudo apachectl configtest
```

Expected:

```text
Syntax OK
```

Nếu không OK:

> stop tại đây, sửa configuration; không restart.

---

# 56. Reload/restart

```bash
sudo systemctl restart httpd
```

Verify process:

```bash
systemctl is-active httpd
```

---

# 57. Functional test

```bash
curl -i http://127.0.0.1/day2/
```

Với SELinux `Enforcing` và custom path chưa label đúng, expected là access failure, thường HTTP `403`.

---

# 58. Troubleshoot đúng thứ tự

### Unix mode

```bash
namei -l /srv/day2web/index.html
```

hoặc:

```bash
ls -ld /srv /srv/day2web
ls -l /srv/day2web/index.html
```

### SELinux context

```bash
ls -Zd /srv/day2web
ls -Z /srv/day2web/index.html
```

### Apache log

```bash
sudo tail -n 30 /var/log/httpd/error_log
```

### SELinux AVC

Nếu Audit available:

```bash
sudo ausearch -m AVC -ts recent
```

Có thể thấy denial liên quan `httpd`.

---

# 59. Sai cách — không làm

Không chạy:

```bash
sudo chmod -R 777 /srv/day2web
```

và không:

```bash
sudo setenforce 0
```

rồi coi problem solved.

Red Hat hiện cũng khuyến cáo không disable SELinux cho troubleshooting thông thường. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/changing-selinux-states-and-modes?utm_source=chatgpt.com)

---

# 60. Correct SELinux fix

Check `semanage`:

```bash
command -v semanage
```

Nếu unavailable, xác định package approved cung cấp command:

```bash
dnf provides '*/semanage'
```

Sau khi package được approved/install:

**[CHG]**

```bash
sudo semanage fcontext \
  -a \
  -t httpd_sys_content_t \
  "/srv/day2web(/.*)?"
```

Điều này tạo persistent labeling rule.

Apply:

```bash
sudo restorecon -Rv /srv/day2web
```

Red Hat mô tả đúng sequence này: `semanage fcontext` tạo mapping, `restorecon` thực sự apply label. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_selinux/troubleshooting-problems-related-to-selinux?utm_source=chatgpt.com)

---

# 61. Verify context

```bash
ls -Zd /srv/day2web
ls -Z /srv/day2web/index.html
```

Expected type:

```text
httpd_sys_content_t
```

---

# 62. Functional verification

```bash
curl -i http://127.0.0.1/day2/
```

Expected:

```text
HTTP/1.1 200 OK
...
Day 2 SELinux Lab ...
```

Evidence:

```bash
{
  getenforce
  ls -Zd /srv/day2web
  ls -Z /srv/day2web/index.html
} > ~/day2-evidence/selinux/post_fix.txt
```

---

# 63. Lab 8 — Network baseline

## Objective

Thực hành:

```text
interface
IP
route
DNS
TCP port
application HTTP
```

---

# 64. Interface state

```bash
ip link
ip addr
```

Evidence:

```bash
ip addr > ~/day2-evidence/network/ip_addr.txt
```

Xác định interface thực sự của host.

Không assume:

```text
eth0
```

Có thể là:

```text
ens192
ens3
enp1s0
```

---

# 65. Routing

```bash
ip route
```

Identify default route:

```bash
ip route show default
```

Test route selection:

```bash
ip route get <CLIENT_IP>
```

hoặc một approved test IP.

Output phải cho biết:

```text
destination
via/gateway nếu có
dev
src
```

---

# 66. DNS

Dùng organization lab DNS target:

```bash
getent hosts <TEST_HOSTNAME>
```

Ví dụ nếu server hostname resolve:

```bash
getent hosts "$(hostname -f)"
```

Nếu có approved external DNS:

```bash
getent hosts www.redhat.com
```

nhưng external Internet không phải requirement.

---

# 67. DNS failure fixture

Reserved `.invalid` domain có thể dùng để tạo expected DNS failure:

```bash
getent hosts day2-does-not-exist.invalid
```

Expected:

```text
no result / non-zero exit
```

Check:

```bash
echo $?
```

Điểm cần nhận biết:

> Name-resolution failure không phải bằng chứng rằng IP connectivity cũng fail.

---

# 68. `ping`

```bash
ping -c 3 <CLIENT_OR_GATEWAY_IP>
```

Nhưng nếu ping fail:

```text
ICMP blocked
```

vẫn là possibility.

Không được kết luận host unreachable chỉ bằng ping.

---

# 69. `tracepath`

Nếu available:

```bash
tracepath <TEST_IP>
```

hoặc:

```bash
traceroute <TEST_IP>
```

Không phải mọi intermediate hop sẽ reply.

---

# 70. Socket verification

Server:

```bash
sudo ss -lntp
```

Specific:

```bash
sudo ss -lntp | grep ':80'
```

Đây là proof:

```text
application/process
→ listening socket
```

---

# 71. HTTP/TCP verification

Local:

```bash
curl --connect-timeout 5 \
     --max-time 10 \
     -v \
     http://127.0.0.1/day2/
```

`--connect-timeout` và `--max-time` tránh command treo vô hạn trong automation.

---

# 72. Lab 9 — `firewalld`

## Safety

Phải biết SSH đang đi qua network nào trước khi thay firewall.

Trong lab dedicated server, capture:

```bash
sudo firewall-cmd --state
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
sudo firewall-cmd --list-all --permanent
```

Red Hat hiện phân biệt runtime và permanent configuration; predefined `http` service có thể được allow bằng `firewall-cmd`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld?utm_source=chatgpt.com)

---

# 73. Determine active zone

```bash
sudo firewall-cmd --get-active-zones
```

Ví dụ:

```text
public
  interfaces: ens192
```

Đặt:

```bash
ZONE=public
```

**Bạn phải thay `public` bằng actual lab zone.**

---

# 74. Baseline rule

```bash
sudo firewall-cmd --zone="$ZONE" --query-service=http
```

Possible:

```text
no
```

Permanent:

```bash
sudo firewall-cmd \
  --permanent \
  --zone="$ZONE" \
  --query-service=http
```

---

# 75. Test trước firewall change

Từ `client-day2`:

```bash
curl --connect-timeout 3 \
     --max-time 5 \
     http://<RHEL_IP>/day2/
```

Nếu HTTP chưa allowed:

Expected:

```text
timeout / connection failure
```

Trong khi trên server:

```bash
curl http://127.0.0.1/day2/
```

vẫn thành công.

Đây là strong evidence cho:

```text
application healthy locally
+
network/firewall blocks remote path
```

---

# 76. Add runtime HTTP permission

**[CHG]**

```bash
sudo firewall-cmd \
  --zone="$ZONE" \
  --add-service=http
```

Verify:

```bash
sudo firewall-cmd \
  --zone="$ZONE" \
  --query-service=http
```

Expected:

```text
yes
```

---

# 77. Retest from client

```bash
curl http://<RHEL_IP>/day2/
```

Expected:

```text
Day 2 SELinux Lab ...
```

Bạn vừa chứng minh:

```text
httpd active
+ listener
+ SELinux correct
+ firewall allows
+ IP path works
+ HTTP works
```

---

# 78. Persistent firewall rule

Nếu lab requirement là persistent:

**[CHG]**

```bash
sudo firewall-cmd \
  --permanent \
  --zone="$ZONE" \
  --add-service=http
```

Verify both:

```bash
sudo firewall-cmd \
  --zone="$ZONE" \
  --query-service=http

sudo firewall-cmd \
  --permanent \
  --zone="$ZONE" \
  --query-service=http
```

Cả hai:

```text
yes
```

---

# 79. Vì sao không luôn dùng `--runtime-to-permanent`?

Red Hat có document `--runtime-to-permanent`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/pdf/configuring_firewalls_and_packet_filters/using-and-configuring-firewalld?utm_source=chatgpt.com)

Nhưng operationally nếu runtime hiện chứa nhiều temporary changes ngoài ticket, copy toàn runtime sang permanent có thể persist cả những thứ không mong muốn.

Trong lab này ta dùng explicit:

```bash
--permanent --add-service=http
```

để scope thay đổi rõ hơn.

---

# 80. Firewall rollback

Cuối lab nếu baseline ban đầu không có HTTP rule:

```bash
sudo firewall-cmd \
  --zone="$ZONE" \
  --remove-service=http
```

và:

```bash
sudo firewall-cmd \
  --permanent \
  --zone="$ZONE" \
  --remove-service=http
```

Verify:

```bash
sudo firewall-cmd --zone="$ZONE" --query-service=http
```

Expected:

```text
no
```

Chỉ rollback cuối module nếu HTTP không còn cần cho các lab sau.

---

# 81. Network troubleshooting matrix

Khi remote HTTP fail:

| Test                             | Nếu fail                  | Layer nghi ngờ              |
| -------------------------------- | ------------------------- | --------------------------- |
| `ip addr`                        | IP missing                | local network config        |
| `ip route get`                   | no route                  | routing                     |
| `getent hosts`                   | name fail                 | DNS                         |
| `ping`                           | no response               | ICMP/path, chưa kết luận    |
| `ss -lnt`                        | port missing              | service/listener            |
| local `curl`                     | HTTP fail                 | service/application/SELinux |
| remote `curl`                    | local works, remote fails | firewall/path               |
| remote HTTP connects but 4xx/5xx | transport works           | application layer           |

---

# 82. Lab 10 — Storage fundamentals bằng filesystem image

Mục tiêu là thực hành:

```text
filesystem
mount
fstab
capacity
```

mà không chạm disk thật.

---

# 83. Create storage image

**[CHG — SAFE LAB FIXTURE]**

```bash
sudo truncate -s 256M /var/tmp/day2-storage.img
```

Inspect:

```bash
ls -lh /var/tmp/day2-storage.img
```

Expected:

```text
256M
```

---

# 84. Create filesystem

Ta dùng `ext4` cho image lab vì tạo filesystem trực tiếp trong regular file thuận tiện và an toàn.

**[CHG]**

```bash
sudo mkfs.ext4 -F /var/tmp/day2-storage.img
```

`-F` force tạo filesystem trên regular file fixture.

Không áp dụng cùng command này cho random `/dev/sdX`.

---

# 85. Create mount point

```bash
sudo mkdir -p /mnt/day2data
```

Check chưa mounted:

```bash
findmnt /mnt/day2data
```

Expected no result.

---

# 86. Mount bằng loop

**[CHG]**

```bash
sudo mount \
  -o loop \
  /var/tmp/day2-storage.img \
  /mnt/day2data
```

Verify:

```bash
findmnt /mnt/day2data
df -h /mnt/day2data
```

Expected filesystem khoảng 256 MB.

---

# 87. Test read/write

```bash
echo "Day 2 storage test" \
  | sudo tee /mnt/day2data/test.txt
```

Verify:

```bash
cat /mnt/day2data/test.txt
```

---

# 88. Persistent mount bằng `/etc/fstab`

Đây là `[RISK]` category trên real servers.

Trước thay đổi:

```bash
sudo cp -a \
  /etc/fstab \
  /etc/fstab.day2-before
```

Hash:

```bash
sudo sha256sum \
  /etc/fstab \
  /etc/fstab.day2-before
```

---

# 89. Add controlled fstab entry

Đầu tiên unmount:

```bash
sudo umount /mnt/day2data
```

Thêm:

```bash
echo '/var/tmp/day2-storage.img /mnt/day2data ext4 loop,nofail 0 0' \
  | sudo tee -a /etc/fstab
```

`nofail` làm lab fixture ít gây boot-risk hơn nếu file missing.

---

# 90. Register fstab changes

Red Hat hiện hướng dẫn `systemctl daemon-reload` sau khi thêm persistent filesystem entry vì systemd tạo mount units từ `fstab`. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/managing_file_systems/persistently-mounting-file-systems?utm_source=chatgpt.com)

```bash
sudo systemctl daemon-reload
```

---

# 91. Validate trước mount

```bash
sudo findmnt --verify --verbose
```

Nếu output có error liên quan Day 2 entry:

> Dừng lại và rollback.

Không tiếp tục reboot.

---

# 92. Mount exact target

Thay vì:

```bash
mount -a
```

có thể tác động mọi unmounted `fstab` entry trên host, ta scope:

```bash
sudo mount /mnt/day2data
```

Verify:

```bash
findmnt /mnt/day2data
df -h /mnt/day2data
cat /mnt/day2data/test.txt
```

Dữ liệu cũ phải còn.

---

# 93. `fstab` rollback

Nếu entry lỗi:

```bash
sudo cp -a \
  /etc/fstab.day2-before \
  /etc/fstab
```

Sau:

```bash
sudo systemctl daemon-reload
```

Và verify:

```bash
sudo findmnt --verify --verbose
```

---

# 94. Lab 10B — Optional real partition lab

**Không làm phần này nếu không có dedicated secondary lab disk.**

Giả sử instructor cấp:

```text
<LAB_DISK>
```

Bạn phải identify bằng:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL,SERIAL
```

và:

```bash
sudo wipefs -n <LAB_DISK>
```

`-n` chỉ inspect signatures.

Trước partitioning cần xác nhận:

```text
not root disk
not boot disk
not mounted
not LVM PV in use
no valuable data
dedicated lab device
```

Sau đó mới có thể dùng approved partitioning workflow.

Không bao giờ copy literal:

```text
/dev/sdb
```

từ tài liệu và assume đó là lab disk.

---

# 95. Lab 11 — Disk capacity failure

Dùng chính filesystem image để failure không lan sang root filesystem.

Check:

```bash
df -h /mnt/day2data
```

---

# 96. Create large test data

Ví dụ:

```bash
sudo dd \
  if=/dev/zero \
  of=/mnt/day2data/fill.bin \
  bs=1M \
  count=200 \
  status=progress
```

Sau:

```bash
df -h /mnt/day2data
```

Bạn sẽ thấy usage tăng đáng kể.

---

# 97. Simulate near-full filesystem

Nếu muốn thêm:

```bash
sudo dd \
  if=/dev/zero \
  of=/mnt/day2data/fill2.bin \
  bs=1M \
  count=80 \
  status=progress
```

Nó có thể fail:

```text
No space left on device
```

Điều này **chỉ xảy ra trên 256 MB lab image**.

Không làm trên `/`.

---

# 98. Diagnose

```bash
df -h /mnt/day2data
df -i /mnt/day2data
du -sh /mnt/day2data/*
```

Ở scenario này:

```text
block capacity high
inodes likely fine
```

Root cause:

> Filesystem data blocks exhausted by lab fill files.

---

# 99. Recovery

```bash
sudo rm -f \
  /mnt/day2data/fill.bin \
  /mnt/day2data/fill2.bin
```

Verify:

```bash
df -h /mnt/day2data
```

---

# 100. Lab 12 — Inode exhaustion simulation

Đây là failure rất quan trọng.

Tạo filesystem riêng với số inode nhỏ.

---

# 101. Create inode fixture

```bash
sudo truncate -s 64M /var/tmp/day2-inode.img
```

Create ext4 với restricted inode count:

```bash
sudo mkfs.ext4 \
  -F \
  -N 1024 \
  /var/tmp/day2-inode.img
```

Mount:

```bash
sudo mkdir -p /mnt/day2inode
```

```bash
sudo mount \
  -o loop \
  /var/tmp/day2-inode.img \
  /mnt/day2inode
```

---

# 102. Baseline

```bash
df -h /mnt/day2inode
df -i /mnt/day2inode
```

---

# 103. Exhaust inodes

**[CHG — isolated filesystem only]**

```bash
for i in $(seq 1 1500); do
    sudo touch "/mnt/day2inode/file-$i" || break
done
```

Eventually:

```text
No space left on device
```

---

# 104. Compare block vs inode

```bash
df -h /mnt/day2inode
```

Có thể vẫn còn nhiều MB.

Nhưng:

```bash
df -i /mnt/day2inode
```

gần:

```text
IUse% 100%
```

Đây chính là:

```text
free storage blocks
+
no free inode
=
cannot create another file
```

---

# 105. Recovery

```bash
sudo rm -f /mnt/day2inode/file-*
```

Verify:

```bash
df -i /mnt/day2inode
```

Cleanup:

```bash
sudo umount /mnt/day2inode
sudo rm -f /var/tmp/day2-inode.img
sudo rmdir /mnt/day2inode
```

---

# 106. Lab 13 — `journalctl`

## Objective

Thực hành:

```text
unit filtering
boot filtering
time-window filtering
severity
structured extraction
```

---

# 107. Service journal

```bash
journalctl \
  -u day2-demo.service \
  --no-pager
```

Latest:

```bash
journalctl \
  -u day2-demo.service \
  -n 30 \
  --no-pager
```

---

# 108. Current boot

```bash
journalctl \
  -b \
  -u day2-demo.service \
  --no-pager
```

---

# 109. Errors

```bash
journalctl \
  -p err \
  -b \
  --no-pager
```

Không được assume root cause nhất thiết có priority `err`; warning/info có thể quan trọng.

---

# 110. Lab 14 — Bounded UTC log extraction

Đây là additive requirement chính thức.

Ta sẽ tạo một controlled event.

---

# 111. Capture UTC start

```bash
START_UTC=$(date -u '+%Y-%m-%d %H:%M:%S')
echo "$START_UTC"
```

---

# 112. Generate approved event

```bash
sudo systemctl restart day2-demo.service
```

Verify:

```bash
systemctl is-active day2-demo.service
```

Capture end:

```bash
END_UTC=$(date -u '+%Y-%m-%d %H:%M:%S')
echo "$END_UTC"
```

---

# 113. Human-readable bounded extraction

```bash
sudo journalctl \
  --utc \
  -u day2-demo.service \
  --since "$START_UTC UTC" \
  --until "$END_UTC UTC" \
  -o short-iso-precise \
  --no-pager \
  > ~/day2-evidence/logs/day2-demo-readable.log
```

---

# 114. Structured extraction

Có thể tạo Journal Export Format:

```bash
sudo journalctl \
  --utc \
  -u day2-demo.service \
  --since "$START_UTC UTC" \
  --until "$END_UTC UTC" \
  -o export \
  --no-pager \
  > ~/day2-evidence/logs/day2-demo.export
```

Đồng thời có thể tạo JSON để analysis dễ hơn:

```bash
sudo journalctl \
  --utc \
  -u day2-demo.service \
  --since "$START_UTC UTC" \
  --until "$END_UTC UTC" \
  -o json \
  --no-pager \
  > ~/day2-evidence/logs/day2-demo.json
```

---

# 115. Record count

Vì JSON output thường one-record-per-line:

```bash
wc -l ~/day2-evidence/logs/day2-demo.json
```

Lưu:

```bash
wc -l ~/day2-evidence/logs/day2-demo.json \
  > ~/day2-evidence/logs/day2-demo.count.txt
```

---

# 116. Hash

```bash
sha256sum \
  ~/day2-evidence/logs/day2-demo.export \
  ~/day2-evidence/logs/day2-demo-readable.log \
  ~/day2-evidence/logs/day2-demo.json \
  > ~/day2-evidence/logs/SHA256SUMS
```

---

# 117. Extraction transcript

Tạo file:

```bash
cat > ~/day2-evidence/logs/extraction_metadata.txt <<EOF
Host: $(hostname)
Collector: $(whoami)
Collection time UTC: $(date -u +%FT%TZ)
Window start UTC: ${START_UTC} UTC
Window end UTC: ${END_UTC} UTC
Unit: day2-demo.service
EOF
```

Phải ghi actual commands vào ticket/evidence notes nữa.

---

# 118. Lab 15 — Syslog/authentication logs

Check:

```bash
ls -lh /var/log/
```

RHEL environment có thể có:

```text
/var/log/messages
/var/log/secure
/var/log/cron
```

nhưng existence/usage phụ thuộc configuration.

Không assume tất cả đều tồn tại.

---

# 119. Sudo evidence

Sau một approved:

```bash
sudo systemctl restart day2-demo.service
```

Search recent authentication records.

Nếu `/var/log/secure` có:

```bash
sudo grep 'sudo' /var/log/secure | tail -20
```

Journal:

```bash
sudo journalctl \
  --since "-10 minutes" \
  --no-pager \
  | grep -E 'sudo|day2-demo'
```

Đây là investigation output, không phải lúc nào cũng là complete audit trail.

---

# 120. Audit subsystem

Check:

```bash
systemctl status auditd --no-pager
```

Nếu active:

```bash
sudo ausearch -m AVC -ts recent
```

Có thể explore recent audit activity theo approved scope.

Nếu Audit không configured hoặc collector không có permission:

```text
Audit coverage unavailable
```

phải được ghi như một evidence gap.

Syllabus đặc biệt yêu cầu không giả định mọi action đều logged. Infrastructure_Operations_14_Da…

---

# 121. Rotated logs

Inspect:

```bash
sudo ls -lh /var/log/secure*
sudo ls -lh /var/log/messages*
sudo ls -lh /var/log/cron*
```

Có thể gặp:

```text
secure
secure-20261018
secure.1
secure.2.gz
```

Nếu `.gz`:

```bash
sudo zgrep 'pattern' /var/log/secure*.gz
```

Nếu plain rotated file:

```bash
sudo grep 'pattern' /var/log/secure*
```

Không bắt buộc phải force rotation để hoàn thành lab.

---

# 122. Inspect logrotate configuration

```bash
cat /etc/logrotate.conf
```

```bash
ls -l /etc/logrotate.d/
```

Debug/dry-run:

```bash
sudo logrotate -d /etc/logrotate.conf
```

`-d` dùng cho debug; không nên force production log rotation chỉ để thử.

---

# 123. Lab 16 — `grep`

Ví dụ:

```bash
grep 'failed' \
  ~/day2-evidence/logs/day2-demo-readable.log
```

Case-insensitive:

```bash
grep -i 'fail' \
  ~/day2-evidence/logs/day2-demo-readable.log
```

Multiple patterns:

```bash
grep -Ei 'fail|error|denied|timeout' \
  ~/day2-evidence/logs/day2-demo-readable.log
```

---

# 124. Regex

Beginning:

```bash
grep -E '^2026-' file.log
```

IPv4-like pattern demonstration:

```bash
grep -E \
  '[0-9]{1,3}(\.[0-9]{1,3}){3}' \
  file.log
```

Lưu ý đây chỉ detect textual pattern, không validate IPv4 range đầy đủ.

---

# 125. `awk`

Ví dụ process report:

```bash
ps -eo user,pid,%cpu,%mem,cmd \
  | awk 'NR==1 || $3 > 1.0'
```

Filesystem usage:

```bash
df -P \
  | awk 'NR==1 || $5+0 >= 80'
```

---

# 126. `sed`

Readable redaction example trên **working copy**, không original:

```bash
cp \
  ~/day2-evidence/logs/day2-demo-readable.log \
  ~/day2-evidence/logs/day2-demo-readable.redacted.log
```

Ví dụ transform:

```bash
sed 's/day2svc/<SERVICE_USER>/g' \
  ~/day2-evidence/logs/day2-demo-readable.redacted.log
```

Không overwrite original evidence.

---

# 127. Evidence principle

Source-of-truth syllabus yêu cầu giữ untouched raw export và process trên working copy; evidence manifest phải chứa source, asset/environment, ticket, collector, timestamp/timezone, exact query/filter, format, file size, count, SHA-256 và limitations/errors. Infrastructure_Operations_14_Da…

---

# 128. Lab 17 — Bash health-check script

## Objective

Script phải kiểm tra:

```text
service
disk
inode
mount
route
DNS
HTTP
```

và trả meaningful exit code.

---

# 129. Tạo script

```bash
sudo tee /usr/local/sbin/day2-health-check.sh >/dev/null <<'EOF'
#!/usr/bin/env bash

set -u
set -o pipefail

EXIT_STATUS=0
FS_WARN="${FS_WARN:-85}"
INODE_WARN="${INODE_WARN:-85}"
TEST_URL="${TEST_URL:-http://127.0.0.1/day2/}"
TEST_HOST="${TEST_HOST:-localhost}"
TEST_MOUNT="${TEST_MOUNT:-/mnt/day2data}"

timestamp() {
    date -u +%FT%TZ
}

ok() {
    printf '%s OK   %s\n' "$(timestamp)" "$*"
}

fail() {
    printf '%s FAIL %s\n' "$(timestamp)" "$*" >&2
    EXIT_STATUS=1
}

fatal() {
    printf '%s ERROR %s\n' "$(timestamp)" "$*" >&2
    exit 2
}

for cmd in systemctl df awk findmnt ip getent curl; do
    command -v "$cmd" >/dev/null 2>&1 \
        || fatal "Required command missing: $cmd"
done

printf '%s INFO host=%s\n' "$(timestamp)" "$(hostname)"

if systemctl is-active --quiet day2-demo.service; then
    ok "day2-demo.service active"
else
    fail "day2-demo.service inactive/failed"
fi

if systemctl is-active --quiet httpd.service; then
    ok "httpd.service active"
else
    fail "httpd.service inactive/failed"
fi

ROOT_USAGE=$(
    df -P / \
    | awk 'NR==2 {gsub(/%/,"",$5); print $5}'
)

if [[ "$ROOT_USAGE" =~ ^[0-9]+$ ]] && (( ROOT_USAGE < FS_WARN )); then
    ok "root filesystem usage=${ROOT_USAGE}%"
else
    fail "root filesystem usage=${ROOT_USAGE}% threshold=${FS_WARN}%"
fi

ROOT_INODES=$(
    df -Pi / \
    | awk 'NR==2 {gsub(/%/,"",$5); print $5}'
)

if [[ "$ROOT_INODES" =~ ^[0-9]+$ ]] && (( ROOT_INODES < INODE_WARN )); then
    ok "root inode usage=${ROOT_INODES}%"
else
    fail "root inode usage=${ROOT_INODES}% threshold=${INODE_WARN}%"
fi

if findmnt -rn "$TEST_MOUNT" >/dev/null 2>&1; then
    ok "mount present: $TEST_MOUNT"
else
    fail "mount missing: $TEST_MOUNT"
fi

if ip route show default | grep -q .; then
    ok "default route present"
else
    fail "default route missing"
fi

if getent hosts "$TEST_HOST" >/dev/null 2>&1; then
    ok "name resolution works for $TEST_HOST"
else
    fail "name resolution failed for $TEST_HOST"
fi

if curl \
    --fail \
    --silent \
    --show-error \
    --connect-timeout 3 \
    --max-time 5 \
    "$TEST_URL" \
    >/dev/null; then
    ok "HTTP check passed: $TEST_URL"
else
    fail "HTTP check failed: $TEST_URL"
fi

printf '%s INFO exit_status=%d\n' "$(timestamp)" "$EXIT_STATUS"
exit "$EXIT_STATUS"
EOF
```

---

# 130. Permissions

```bash
sudo chown root:root /usr/local/sbin/day2-health-check.sh
sudo chmod 0755 /usr/local/sbin/day2-health-check.sh
```

---

# 131. Syntax validation

```bash
bash -n /usr/local/sbin/day2-health-check.sh
```

Expected:

```text
no output
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

# 132. Execute

```bash
sudo /usr/local/sbin/day2-health-check.sh
```

Possible:

```text
... OK day2-demo.service active
... OK httpd.service active
... OK root filesystem usage=...
...
... INFO exit_status=0
```

---

# 133. Verify exit code

Immediately:

```bash
echo $?
```

Healthy:

```text
0
```

---

# 134. Negative test

Stop demo service:

```bash
sudo systemctl stop day2-demo.service
```

Run:

```bash
sudo /usr/local/sbin/day2-health-check.sh
```

Expected:

```text
FAIL day2-demo.service inactive/failed
...
exit_status=1
```

Check:

```bash
echo $?
```

Expected:

```text
1
```

Restore:

```bash
sudo systemctl start day2-demo.service
```

---

# 135. Meaning của exit codes trong lab

Ta thiết kế:

```text
0 = health checks passed
1 = check completed, one or more unhealthy conditions
2 = script could not perform required checks
```

Đây là operationally tốt hơn:

```text
all failures = 1
```

vì phân biệt system unhealthy với script itself broken.

---

# 136. Evidence

```bash
sudo /usr/local/sbin/day2-health-check.sh \
  > ~/day2-evidence/scripts/health_check.out \
  2>&1

echo $? \
  > ~/day2-evidence/scripts/health_check.exitcode
```

---

# 137. Lab 18 — Backup source dataset

Tạo controlled data.

```bash
sudo mkdir -p /srv/day2backup-src/{config,data}
```

```bash
echo 'application=day2' \
  | sudo tee /srv/day2backup-src/config/app.conf
```

```bash
echo 'record-001' \
  | sudo tee /srv/day2backup-src/data/file1.txt
```

```bash
echo 'record-002' \
  | sudo tee /srv/day2backup-src/data/file2.txt
```

---

# 138. Source manifest

```bash
sudo bash -c '
cd /srv/day2backup-src &&
find . -type f -print0 |
sort -z |
xargs -0r sha256sum
' | tee ~/day2-evidence/backup/source_manifest.sha256
```

Đây là expected content baseline.

---

# 139. Lab 19 — Backup script

Tạo:

```bash
sudo tee /usr/local/sbin/day2-backup.sh >/dev/null <<'EOF'
#!/usr/bin/env bash

set -u
set -o pipefail

SRC="${SRC:-/srv/day2backup-src}"
DEST="${DEST:-/var/backups/day2}"

timestamp() {
    date -u +%FT%TZ
}

fail() {
    printf '%s ERROR %s\n' "$(timestamp)" "$*" >&2
    exit 1
}

[[ -d "$SRC" ]] \
    || fail "Source directory missing: $SRC"

mkdir -p "$DEST" \
    || fail "Cannot create destination: $DEST"

STAMP=$(date -u +%Y%m%dT%H%M%SZ)

ARCHIVE="${DEST}/day2-${STAMP}.tar.gz"
MANIFEST="${DEST}/day2-${STAMP}.manifest.sha256"

printf '%s INFO creating manifest\n' "$(timestamp)"

(
    cd "$SRC" &&
    find . -type f -print0 |
    sort -z |
    xargs -0r sha256sum
) > "$MANIFEST" \
    || fail "Manifest creation failed"

printf '%s INFO creating archive=%s\n' "$(timestamp)" "$ARCHIVE"

tar \
    -C "$SRC" \
    -czf "$ARCHIVE" \
    . \
    || fail "tar archive creation failed"

gzip -t "$ARCHIVE" \
    || fail "gzip integrity test failed"

tar -tzf "$ARCHIVE" >/dev/null \
    || fail "archive listing failed"

sha256sum "$ARCHIVE" \
    > "${ARCHIVE}.sha256" \
    || fail "archive hash failed"

printf '%s OK backup=%s\n' "$(timestamp)" "$ARCHIVE"
printf '%s OK manifest=%s\n' "$(timestamp)" "$MANIFEST"

exit 0
EOF
```

---

# 140. Permissions

```bash
sudo chown root:root /usr/local/sbin/day2-backup.sh
sudo chmod 0755 /usr/local/sbin/day2-backup.sh
```

Syntax:

```bash
bash -n /usr/local/sbin/day2-backup.sh
```

---

# 141. Run backup

```bash
sudo /usr/local/sbin/day2-backup.sh
```

Expected:

```text
OK backup=/var/backups/day2/day2-....tar.gz
```

Check exit:

```bash
echo $?
```

Expected:

```text
0
```

---

# 142. Inspect artifacts

```bash
sudo ls -lh /var/backups/day2/
```

List archive:

```bash
sudo tar -tzf /var/backups/day2/<ARCHIVE>
```

Verify archive hash:

```bash
sudo sha256sum -c /var/backups/day2/<ARCHIVE>.sha256
```

Expected:

```text
OK
```

---

# 143. Lab 20 — Restore validation

**Backup is not complete until restore is tested.**

Create isolated restore destination:

```bash
sudo mkdir -p /var/tmp/day2-restore
```

Do not restore over source.

---

# 144. Select recovery point

List:

```bash
sudo ls -1t /var/backups/day2/day2-*.tar.gz
```

Choose one:

```bash
ARCHIVE=/var/backups/day2/<SELECTED_ARCHIVE>
```

Corresponding manifest:

```bash
MANIFEST=/var/backups/day2/<SELECTED_MANIFEST>
```

---

# 145. Restore

Create isolated directory:

```bash
RESTORE_DIR=/var/tmp/day2-restore/test-restore
sudo rm -rf "$RESTORE_DIR"
sudo mkdir -p "$RESTORE_DIR"
```

Extract:

```bash
sudo tar \
  -xzf "$ARCHIVE" \
  -C "$RESTORE_DIR"
```

---

# 146. Content verification

```bash
sudo bash -c "
cd '$RESTORE_DIR' &&
sha256sum -c '$MANIFEST'
"
```

Expected:

```text
./config/app.conf: OK
./data/file1.txt: OK
./data/file2.txt: OK
```

---

# 147. File-count verification

Source:

```bash
sudo find /srv/day2backup-src -type f | wc -l
```

Restored:

```bash
sudo find "$RESTORE_DIR" -type f | wc -l
```

Counts phải reconcile.

---

# 148. Functional read

```bash
sudo cat "$RESTORE_DIR/config/app.conf"
sudo cat "$RESTORE_DIR/data/file1.txt"
```

Bây giờ bạn mới có thể nói:

> Backup recovery was successfully validated for the tested dataset and recovery point.

Không nói:

> Enterprise DR is verified.

Lab `tar` không chứng minh full application DR.

---

# 149. Negative backup test

Tạm thời trỏ source tới missing directory:

```bash
sudo SRC=/does/not/exist \
  /usr/local/sbin/day2-backup.sh
```

Expected:

```text
ERROR Source directory missing
```

Exit:

```bash
echo $?
```

Expected:

```text
non-zero
```

Đây là error-handling test.

---

# 150. Lab 21 — Cron scheduling

## Objective

Chứng minh:

```text
schedule configured
→ crond running
→ job executes
→ output produced
→ result verifiable
```

Không chỉ:

```text
cron entry exists
```

---

# 151. Verify `crond`

```bash
systemctl status crond --no-pager
```

Nếu package/service chưa available, xác định approved RHEL package theo environment trước.

Không disable `crond`; Red Hat lưu ý crontab jobs sẽ không chạy khi `crond.service` bị disabled. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/using_systemd_unit_files_to_customize_and_optimize_your_system/optimizing-systemd-to-shorten-the-boot-time?utm_source=chatgpt.com)

---

# 152. Create temporary lab schedule

Trong training có thể dùng every minute để tạo observable event nhanh; **không giữ schedule này sau lab**.

```bash
sudo tee /etc/cron.d/day2-backup >/dev/null <<'EOF'
* * * * * root /usr/local/sbin/day2-backup.sh >> /var/log/day2-backup-cron.log 2>&1
EOF
```

Permissions:

```bash
sudo chmod 0644 /etc/cron.d/day2-backup
```

---

# 153. Verify cron configuration

```bash
sudo cat /etc/cron.d/day2-backup
```

Service:

```bash
systemctl is-active crond
```

---

# 154. Evidence of actual execution

Sau khi scheduler có một lần thực thi, verify artifacts thay vì chỉ nhìn entry:

```bash
sudo tail -n 50 /var/log/day2-backup-cron.log
```

Check:

```bash
sudo ls -lt /var/backups/day2/ | head
```

Cron logs nếu configured:

```bash
sudo tail -n 50 /var/log/cron
```

hoặc:

```bash
sudo journalctl -u crond -n 50 --no-pager
```

---

# 155. Cron failure scenario — PATH

Một common case là script chạy manually nhưng cron fail vì environment khác.

Inspect environment assumptions trong script:

```bash
grep -nE 'PATH|curl|tar|sha256sum|find|sort|xargs' \
  /usr/local/sbin/day2-backup.sh
```

Operational solution thường là:

```text
explicit known paths
or
controlled PATH declaration
```

chứ không assume interactive `.bashrc`.

---

# 156. Cron cleanup

Sau lab:

```bash
sudo rm -f /etc/cron.d/day2-backup
```

Verify:

```bash
test ! -e /etc/cron.d/day2-backup && echo "cron fixture removed"
```

Không để mỗi phút tạo backup vô hạn.

---

# 157. Failure Injection Suite

Syllabus yêu cầu service/network/storage failures. Ta sẽ có 5 controlled scenarios.

---

# Failure 1 — Broken systemd `ExecStart`

Đây là lab đã làm.

### Symptom

```text
day2-demo.service failed
```

### Evidence

```bash
systemctl status day2-demo
journalctl -u day2-demo
systemctl cat day2-demo
```

### Root cause

Broken systemd override.

### Recovery

Remove drop-in → `daemon-reload` → `reset-failed` → start.

---

# Failure 2 — Firewall blocks HTTP

Precondition:

```text
httpd healthy locally
```

Verify:

```bash
curl http://127.0.0.1/day2/
```

Sau đó trên dedicated lab server remove runtime HTTP service:

```bash
sudo firewall-cmd \
  --zone="$ZONE" \
  --remove-service=http
```

Client test:

```bash
curl \
  --connect-timeout 3 \
  --max-time 5 \
  http://<RHEL_IP>/day2/
```

Failure expected.

Server checks:

```bash
systemctl is-active httpd
ss -lnt | grep ':80'
curl http://127.0.0.1/day2/
```

Nếu cả ba healthy:

```text
application OK locally
remote blocked
```

Firewall becomes strong hypothesis.

Recover:

```bash
sudo firewall-cmd \
  --zone="$ZONE" \
  --add-service=http
```

Retest client.

---

# Failure 3 — SELinux denial

Có thể reproduce bằng temporarily removing lab-specific file-context rule.

**Chỉ với Day 2 path.**

First record existing:

```bash
sudo semanage fcontext -l \
  | grep '/srv/day2web'
```

Remove lab rule:

```bash
sudo semanage fcontext \
  -d \
  "/srv/day2web(/.*)?"
```

Restore default context:

```bash
sudo restorecon -Rv /srv/day2web
```

Test:

```bash
curl http://127.0.0.1/day2/
```

Diagnose:

```bash
getenforce
ls -Zd /srv/day2web
sudo ausearch -m AVC -ts recent
```

Recover:

```bash
sudo semanage fcontext \
  -a \
  -t httpd_sys_content_t \
  "/srv/day2web(/.*)?"

sudo restorecon -Rv /srv/day2web
```

Retest.

---

# Failure 4 — Filesystem capacity

Trên isolated `/mnt/day2data`:

```bash
sudo dd \
  if=/dev/zero \
  of=/mnt/day2data/capacity-test.bin \
  bs=1M \
  count=300 \
  status=progress
```

Nó phải fail trước khi ghi đủ 300 MB.

Diagnose:

```bash
df -h /mnt/day2data
df -i /mnt/day2data
du -sh /mnt/day2data/*
```

Recover:

```bash
sudo rm -f /mnt/day2data/capacity-test.bin
```

Verify `df`.

---

# Failure 5 — Missing mount

Unmount isolated storage:

```bash
sudo umount /mnt/day2data
```

Run health-check:

```bash
sudo /usr/local/sbin/day2-health-check.sh
```

Expected:

```text
FAIL mount missing: /mnt/day2data
```

Verify `/etc/fstab`:

```bash
grep '/mnt/day2data' /etc/fstab
```

Mount:

```bash
sudo mount /mnt/day2data
```

Verify:

```bash
findmnt /mnt/day2data
```

Health-check lại.

---

# 158. Integrated troubleshooting discipline

Mỗi failure phải ghi theo template:

```text
Symptom:
Observed impact:

Time detected:
Host:
Environment:

Initial evidence:
1.
2.
3.

Hypothesis 1:
Test:
Result:

Hypothesis 2:
Test:
Result:

Confirmed/probable root cause:

Corrective action:
Authorization:

Post-check:

Rollback performed:
Residual risk:

Escalation required:
```

---

# 159. Lab 22 — RHEL maintenance-ticket simulation

Đây là additive lab quan trọng.

Giả sử ticket:

```text
Ticket: CHG-DAY2-001
Host: rhel-day2
Environment: LAB
Change: Install or update one approved package
Window: Authorized lab window
Rollback: Defined per package/runbook
Validation: OS/service/application checks
```

---

# 160. Không hard-code package

Dùng:

```bash
MAINT_PACKAGE="<APPROVED_PACKAGE>"
```

Instructor có thể chọn:

```text
rsync
httpd
một approved lab package
```

Nếu package đã latest:

> Không tự invent change chỉ để tạo transaction. Dùng package khác được instructor/ticket approve.

---

# 161. Maintenance pre-check

Capture:

```bash
{
  echo "=== TIME ==="
  date -u

  echo "=== HOST ==="
  hostname

  echo "=== OS ==="
  cat /etc/redhat-release

  echo "=== KERNEL ==="
  uname -r

  echo "=== FAILED UNITS ==="
  systemctl --failed

  echo "=== FILESYSTEM ==="
  df -h

  echo "=== INODES ==="
  df -i

  echo "=== NETWORK ==="
  ip route

} | tee ~/day2-evidence/maintenance/precheck.txt
```

---

# 162. Package baseline

```bash
rpm -q "$MAINT_PACKAGE" \
  | tee ~/day2-evidence/maintenance/package_before.txt
```

Nếu absent, exit code non-zero là valid baseline.

Repository:

```bash
dnf info "$MAINT_PACKAGE" \
  > ~/day2-evidence/maintenance/package_candidate.txt
```

---

# 163. Service dependency assessment

Nếu package cung cấp service:

```bash
systemctl status <SERVICE> --no-pager
```

Capture:

```text
running?
enabled?
clients/users?
dependent service?
maintenance impact?
```

---

# 164. Reboot assessment

Phải trả lời:

```text
Does the change touch kernel?
core library?
running daemon?
service restart?
full reboot?
```

Exact reboot-detection tool/policy có thể khác theo RHEL release và organization.

Không assume:

```text
package installed
=
no reboot/restart needed
```

và cũng không reboot mặc định.

---

# 165. Stop conditions

Dừng maintenance nếu:

```text
wrong host
wrong environment
unexpected repository
pre-existing critical failure
filesystem nearly full
package scope differs from ticket
dependency not understood
recovery plan missing
```

---

# 166. Apply approved change

Nếu package absent:

```bash
sudo dnf install "$MAINT_PACKAGE"
```

Nếu approved update và update exists:

```bash
sudo dnf upgrade "$MAINT_PACKAGE"
```

Không dùng:

```bash
sudo dnf upgrade
```

nếu ticket chỉ approve một package.

Scope phải khớp ticket.

---

# 167. Transaction evidence

```bash
sudo dnf history info last \
  > ~/day2-evidence/maintenance/dnf_history.txt
```

Package after:

```bash
rpm -q "$MAINT_PACKAGE" \
  > ~/day2-evidence/maintenance/package_after.txt
```

---

# 168. Service post-check

Nếu affected service:

```bash
systemctl status <SERVICE> --no-pager
systemctl is-active <SERVICE>
```

Listener:

```bash
ss -lntp
```

Functional test phù hợp:

```bash
curl ...
```

---

# 169. Full post-check

```bash
{
  echo "=== TIME ==="
  date -u

  echo "=== FAILED UNITS ==="
  systemctl --failed

  echo "=== FILESYSTEM ==="
  df -h

  echo "=== INODES ==="
  df -i

} > ~/day2-evidence/maintenance/postcheck.txt
```

---

# 170. Compare

```bash
diff -u \
  ~/day2-evidence/maintenance/precheck.txt \
  ~/day2-evidence/maintenance/postcheck.txt
```

Không yêu cầu identical; timestamps/load có thể thay đổi.

Bạn đang tìm:

```text
unexpected deterioration
new failed unit
storage issue
```

---

# 171. Maintenance success criteria

Change chỉ thành công nếu:

```text
intended package state achieved
+
dependent services healthy
+
functional validation passed
+
no unacceptable new failures
+
evidence captured
```

`dnf exit 0` một mình không đủ.

---

# 172. Rollback limitation record

Ticket phải ghi:

```text
Package rollback available?
Configuration compatibility?
Data/schema change involved?
Service restart needed?
Snapshot/backup available?
```

Không viết chung chung:

```text
Rollback: dnf downgrade
```

nếu chưa chứng minh downgrade đủ để khôi phục system state.

---

# 173. Lab 23 — Correlate ticket → sudo → service event

Mục tiêu là xây timeline.

---

# 174. Record maintenance action window

```bash
ACTION_START=$(date -u +'%Y-%m-%d %H:%M:%S')
```

Execute approved action:

```bash
sudo systemctl restart day2-demo.service
```

Record:

```bash
ACTION_END=$(date -u +'%Y-%m-%d %H:%M:%S')
```

---

# 175. Service evidence

```bash
sudo journalctl \
  --utc \
  -u day2-demo.service \
  --since "$ACTION_START UTC" \
  --until "$ACTION_END UTC" \
  -o short-iso-precise \
  --no-pager
```

---

# 176. Sudo/auth evidence

If relevant source available:

```bash
sudo journalctl \
  --utc \
  --since "$ACTION_START UTC" \
  --until "$ACTION_END UTC" \
  --no-pager \
  | grep -E 'sudo|day2-demo'
```

If `/var/log/secure` exists:

```bash
sudo grep \
  -E 'sudo|systemctl' \
  /var/log/secure \
  | tail -30
```

---

# 177. Timeline

Tạo:

```text
Ticket: CHG-DAY2-001

T0  Change approved
T1  sudo operation initiated
T2  systemd stop event
T3  systemd start event
T4  process started
T5  service active
T6  application check passed
```

Nếu một step không có log:

```text
No record available in source X
```

Không invent timestamp.

---

# 178. Lab 24 — Troubleshooting & escalation record

Đây là official deliverable của Day 2.

Ví dụ:

```text
Incident ID:
INC-DAY2-003

Asset:
rhel-day2

Environment:
LAB

Reported symptom:
Remote HTTP access failed.

Time:
2026-10-20Txx:xx:xxZ

Initial checks:
- httpd active
- TCP/80 listening
- localhost curl returns HTTP 200
- remote curl times out

Hypothesis:
Host firewall blocks TCP/80.

Evidence:
firewall-cmd --zone=... --query-service=http -> no

Root cause:
HTTP service was absent from active firewalld zone.

Corrective action:
Added HTTP service to the approved lab zone.

Verification:
remote curl HTTP 200
local service unchanged
firewalld runtime query -> yes

Rollback:
remove HTTP service from lab zone

Escalation:
Not required
```

---

# 179. Escalation example

Nếu bạn thấy:

```text
filesystem I/O errors
kernel stack trace
suspected disk corruption
```

không tiếp tục:

```text
mkfs
fsck on mounted production filesystem
random mount changes
```

Escalation record:

```text
Observed:
kernel/storage errors

Actions performed:
read-only log/storage checks

Not performed:
filesystem repair because data-integrity risk exceeds L1/L2 authorization

Escalated to:
Linux/storage team

Evidence:
journal excerpt
lsblk
findmnt
df
timestamps
ticket
```

Đây là correct Operations behavior.

---

# 180. Evidence manifest

Syllabus chung yêu cầu mỗi evidence item phải traceable theo source, version/environment/asset, ticket, collector, timestamp/timezone, query/window, command/filter, file path/size/count/hash và limitations. Infrastructure_Operations_14_Da…

Một CSV header có thể là:

```text
evidence_id,
source_system,
product_version,
environment,
asset_id,
ticket,
collector,
collection_time_utc,
requested_window_utc,
command_or_filter,
format,
encoding,
file_path,
size_bytes,
record_count,
sha256,
parent_evidence_id,
classification,
limitations
```

---

# 181. Hash toàn bộ evidence files

Từ evidence root:

```bash
cd ~/day2-evidence
```

```bash
find . \
  -type f \
  ! -name SHA256SUMS.txt \
  -print0 \
  | sort -z \
  | xargs -0 sha256sum \
  > SHA256SUMS.txt
```

Verify:

```bash
sha256sum -c SHA256SUMS.txt
```

Expected:

```text
... OK
```

Nhưng nhớ:

> Matching hash chứng minh file hiện tại giống file tại lúc hash được tạo; nó không chứng minh collection query đầy đủ hoặc source log truthful.

---

# 182. Evidence tree cuối Day 2

Một bộ evidence tốt có thể trông như:

```text
day2-evidence/
├── baseline/
│   ├── host_identity.txt
│   ├── time_baseline.txt
│   └── precheck.txt
├── packages/
│   ├── repositories.txt
│   ├── package_versions.txt
│   └── dnf_transaction.txt
├── processes/
│   ├── environment_exported.txt
│   └── process_snapshot.txt
├── systemd/
│   ├── pre_failure_status.txt
│   ├── pre_failure_journal.txt
│   └── post_recovery_status.txt
├── network/
│   └── ...
├── selinux/
│   └── post_fix.txt
├── firewall/
│   └── ...
├── storage/
│   └── ...
├── logs/
│   ├── day2-demo.export
│   ├── day2-demo.json
│   ├── day2-demo-readable.log
│   ├── extraction_metadata.txt
│   ├── day2-demo.count.txt
│   └── SHA256SUMS
├── scripts/
│   ├── health_check.out
│   └── health_check.exitcode
├── backup/
│   └── source_manifest.sha256
├── restore/
│   └── ...
├── maintenance/
│   ├── precheck.txt
│   ├── package_before.txt
│   ├── package_after.txt
│   ├── dnf_history.txt
│   └── postcheck.txt
├── troubleshooting/
│   └── incident_record.txt
└── SHA256SUMS.txt
```

---

# 183. Cleanup / rollback cuối module

Cleanup là một phần của lab, không phải optional.

## Cron

```bash
sudo rm -f /etc/cron.d/day2-backup
```

## Storage image

Trước:

```bash
findmnt /mnt/day2data
```

Unmount:

```bash
sudo umount /mnt/day2data
```

Remove Day 2 fstab entry **bằng cách restore verified backup hoặc edit đúng entry**.

Safer lab rollback:

```bash
sudo cp -a /etc/fstab.day2-before /etc/fstab
sudo systemctl daemon-reload
```

Sau đó:

```bash
sudo findmnt --verify --verbose
```

Delete:

```bash
sudo rm -f /var/tmp/day2-storage.img
sudo rmdir /mnt/day2data
```

---

# 184. SELinux cleanup

Nếu custom web path không cần:

```bash
sudo semanage fcontext \
  -d \
  "/srv/day2web(/.*)?"
```

Sau đó remove content:

```bash
sudo rm -rf /srv/day2web
```

Không cần `restorecon` nếu directory đã remove.

---

# 185. Apache cleanup

```bash
sudo rm -f /etc/httpd/conf.d/day2-lab.conf
```

Validate:

```bash
sudo apachectl configtest
```

Only if:

```text
Syntax OK
```

thì:

```bash
sudo systemctl restart httpd
```

---

# 186. Firewall cleanup

Chỉ remove rule nếu Day 2 đã add và baseline trước đó không có:

```bash
sudo firewall-cmd \
  --zone="$ZONE" \
  --remove-service=http
```

```bash
sudo firewall-cmd \
  --permanent \
  --zone="$ZONE" \
  --remove-service=http
```

Verify against baseline.

---

# 187. Custom systemd service cleanup

Disable/stop:

```bash
sudo systemctl disable --now day2-demo.service
```

Remove:

```bash
sudo rm -f /etc/systemd/system/day2-demo.service
sudo rm -rf /etc/systemd/system/day2-demo.service.d
sudo rm -f /usr/local/libexec/day2-demo.sh
```

Reload:

```bash
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

Optionally remove service account:

```bash
sudo userdel day2svc
```

chỉ nếu account được tạo riêng cho lab và không được reuse.

---

# 188. Backup fixture cleanup

Sau khi evidence đã được retained theo requirement:

```bash
sudo rm -rf /srv/day2backup-src
sudo rm -rf /var/backups/day2
sudo rm -rf /var/tmp/day2-restore
sudo rm -f /usr/local/sbin/day2-backup.sh
sudo rm -f /usr/local/sbin/day2-health-check.sh
```

Không delete evidence package trước review.

---

# 189. Final post-cleanup health check

Capture:

```bash
{
  date -u
  hostname
  systemctl --failed
  df -h
  df -i
  findmnt
  ip route
} > ~/day2-evidence/baseline/final_post_cleanup.txt
```

So sánh:

```bash
diff -u \
  ~/day2-evidence/baseline/precheck.txt \
  ~/day2-evidence/baseline/final_post_cleanup.txt
```

Expected differences phải explain được.

---

# 190. Day 2 Practical — Acceptance Checklist

Bạn chỉ nên coi Module Thực hành Day 2 hoàn tất khi có thể tự mình chứng minh:

### Package & maintenance

- Xác định đúng RHEL release.
- Inspect approved repositories.
- Hiểu subscription/content model của lab.
- Install approved package bằng `dnf`.
- Verify bằng `rpm`.
- Thu được DNF transaction evidence.
- Làm pre/post maintenance checks.
- Đánh giá dependency và reboot/restart requirement.
- Biết recovery limitation.

### Process & environment

- Tạo và inspect process.
- Xác định PID/PPID/state.
- Gửi graceful signal.
- Chứng minh khác biệt giữa shell variable và exported environment.
- Hiểu systemd/cron environment không giống interactive shell.

### `systemd`

- Tạo custom service.
- Chạy dưới dedicated unprivileged account.
- `start`, `stop`, `restart`, `enable`.
- Phân biệt active/enabled.
- Inject failure.
- Dùng `status`, `journalctl`, `systemctl cat` để tìm root cause.
- Recover đúng service.

### Networking

- Xác định interface/IP.
- Đọc routing table.
- Dùng `ip route get`.
- Test DNS bằng `getent`.
- Hiểu limitation của `ping`.
- Dùng `tracepath`/`traceroute`.
- Dùng `ss`.
- Dùng `curl` để phân biệt transport/application.

### `firewalld`

- Identify active zone.
- Inspect runtime rules.
- Inspect permanent rules.
- Test denied path.
- Add approved HTTP permission.
- Test allowed path.
- Rollback về baseline.

### SELinux

- Xác nhận `Enforcing`.
- Inspect file context.
- Reproduce HTTP denial.
- Collect AVC evidence nếu available.
- Không `chmod 777`.
- Không disable SELinux.
- Sửa persistent labeling bằng `semanage fcontext`.
- Apply bằng `restorecon`.
- Verify HTTP recovery.

### Storage

- Hiểu block device/filesystem/mount.
- Tạo isolated filesystem fixture.
- Mount/unmount.
- Configure controlled `/etc/fstab` entry.
- Backup `fstab`.
- Run `daemon-reload`.
- Validate/mount exact target.
- Detect block-capacity problem.
- Detect inode exhaustion.
- Cleanup filesystem fixture.

### Logs

- Dùng `journalctl -u`.
- Dùng `journalctl -b`.
- Filter theo UTC time window.
- Tạo structured/raw export.
- Tạo readable output.
- Inspect auth/syslog sources.
- Hiểu rotated logs.
- Hiểu Audit source có thể unavailable.
- Dùng `grep`, `awk`, `sed` và Regex.
- Không overwrite raw evidence.

### Bash automation

- Health-check có meaningful output.
- Có non-zero failure status.
- Phân biệt unhealthy check và script execution error.
- Test positive case.
- Test negative case.

### Backup & restore

- Tạo controlled source.
- Hash source data.
- Tạo `tar` backup.
- Verify archive integrity.
- Hash archive.
- Restore vào isolated directory.
- Verify file count.
- Verify checksums.
- Test backup failure path.

### Cron

- Verify `crond`.
- Install controlled schedule.
- Chứng minh job thực sự chạy.
- Capture output/logs.
- Hiểu environment/PATH issues.
- Remove temporary schedule sau lab.

### Troubleshooting & evidence

- Hoàn thành ít nhất service/network/storage Failure Injection.
- Viết symptom riêng với root cause.
- Thực hiện least-risk-first.
- Ghi exact commands.
- Ghi timestamp/timezone.
- Ghi ticket.
- Ghi rollback.
- Ghi escalation boundary.
- Tạo SHA-256 manifest.
- Ghi rõ missing evidence/audit gaps.

Nếu hoàn thành toàn bộ checklist này, bạn đã đáp ứng đúng mục tiêu Day 2 của syllabus: **RHEL administration + controlled troubleshooting + maintenance + automation + backup/recovery + audit-quality evidence**, thay vì chỉ biết chạy Linux commands. Infrastructure_Operations_14_Da…
