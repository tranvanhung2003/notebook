Được. Từ **Unit 2 trở đi**, tôi sẽ tuân theo hai quy ước của bạn:

- Mọi command Linux đều chạy **trên EC2 sau khi SSH vào**, không thao tác shell trên WSL nữa.
- Những phần liên quan AWS như EC2 state, Security Group, route table… sẽ thao tác bằng **AWS Console**, không dùng AWS CLI.
- Riêng mục 6 của Unit 2 “trình chiếu evidence Day 2–6” chúng ta **bỏ khỏi kịch bản** theo yêu cầu của bạn.

Theo Exam Demo Guide, Unit 2 có 5 nhóm thao tác bắt buộc còn lại: CLI/file/text processing; user/group/permission/SSH key; package + systemd + recovery; connectivity/storage diagnosis; health-check + backup + cron + exit code/journalctl. :chatgpt-content-reference{index="0"} Đây chính là phần tổng hợp Day 2–6 trong syllabus. :chatgpt-content-reference{index="1"}

# Unit 2 — Linux Fundamentals — 25 phút

Baseline hiện tại của chúng ta là:

```text
EC2 Ubuntu 24.04
├── Java 17
├── Tomcat 10
├── PostgreSQL
├── Prometheus
├── node_exporter
├── Grafana
└── SSH đang hoạt động
```

Tất cả command bên dưới được hiểu là chạy tại prompt kiểu:

```text
ubuntu@java-fresher-lab:~$
```

Không chạy chúng trên WSL.

## 1. Phân bổ thời gian đề xuất

| Thời gian  | Nội dung                                                       |
| ---------- | -------------------------------------------------------------- |
| 0–4 phút   | CLI navigation + file/directory + `grep`/`awk`/`sed`           |
| 4–9 phút   | User/group + ownership/permissions + SSH key                   |
| 9–15 phút  | Package + custom `systemd` service + failure recovery          |
| 15–19 phút | Connectivity/storage diagnosis với OSI Mindset                 |
| 19–25 phút | Health-check + `tar`/`rsync` + cron + exit code + `journalctl` |

Mục tiêu là **thao tác và giải thích**, không đọc lý thuyết dài dòng.

---

# 2. Chuẩn bị vùng lab riêng cho Unit 2

Trước tiên xác nhận:

```bash
whoami
hostname
pwd
```

Expected invariant:

```text
user     = ubuntu
hostname = java-fresher-lab
```

Kiểm tra timestamp history từ Unit 1:

```bash
echo "$HISTTIMEFORMAT"
```

Expected:

```text
%Y-%m-%d %H:%M:%S
```

Tạo vùng lab:

```bash
sudo mkdir -p /srv/java-fresher/unit2
sudo chown ubuntu:ubuntu /srv/java-fresher/unit2
cd /srv/java-fresher/unit2
```

Giải thích:

```text
/srv
```

thường phù hợp với dữ liệu phục vụ bởi service.

```bash
mkdir -p
```

`-p` tạo cả parent directories và không báo lỗi nếu directory đã tồn tại.

```bash
chown ubuntu:ubuntu
```

đổi owner và group để user hiện tại thao tác lab không cần `sudo` cho mọi file.

Verify:

```bash
pwd
ls -ld /srv/java-fresher/unit2
```

---

# 3. Phần 1 — CLI navigation, files, directories

Tạo cấu trúc:

```bash
mkdir -p demo/{config,logs,data,backup}
```

Kiểm tra:

```bash
find demo -maxdepth 2 -type d | sort
```

Expected:

```text
demo
demo/backup
demo/config
demo/data
demo/logs
```

Di chuyển:

```bash
cd demo
pwd
```

Xem file:

```bash
ls -lah
```

Các option:

```text
-l = long format
-a = include hidden files
-h = human-readable sizes
```

Tạo file:

```bash
touch data/users.txt
```

Tạo nội dung:

```bash
cat > data/users.txt <<'EOF'
alice,developer,active
bob,administrator,active
charlie,developer,disabled
david,operations,active
emma,administrator,disabled
EOF
```

Verify:

```bash
cat data/users.txt
```

Copy:

```bash
cp data/users.txt backup/users.txt.bak
```

Verify:

```bash
ls -l data/users.txt backup/users.txt.bak
```

So sánh:

```bash
diff data/users.txt backup/users.txt.bak
```

Expected: không có output.

Đây là một điểm bạn nên giải thích:

> "`diff` không output nghĩa là hai file không có khác biệt. Em dùng nó để validate backup trước khi thay đổi."

---

# 4. `grep`

Tìm administrator:

```bash
grep 'administrator' data/users.txt
```

Expected:

```text
bob,administrator,active
emma,administrator,disabled
```

Tìm không phân biệt hoa thường:

```bash
grep -i 'developer' data/users.txt
```

Hiển thị số dòng:

```bash
grep -n 'disabled' data/users.txt
```

Bạn cần nhớ:

```text
-i = ignore case
-n = line number
-v = invert match
-r = recursive
```

Ví dụ tìm user không bị disabled:

```bash
grep -v 'disabled' data/users.txt
```

---

# 5. `awk`

File đang dùng comma làm delimiter:

```text
alice,developer,active
```

Cho nên:

```bash
awk -F',' '{print $1}' data/users.txt
```

Expected:

```text
alice
bob
charlie
david
emma
```

`-F','` nghĩa là delimiter là comma.

```text
$1 = field 1
$2 = field 2
$3 = field 3
```

Lọc active users:

```bash
awk -F',' '$3=="active" {print $1, $2}' data/users.txt
```

Expected:

```text
alice developer
bob administrator
david operations
```

Đếm:

```bash
awk -F',' '$3=="active" {count++} END {print "Active users:", count}' data/users.txt
```

---

# 6. `sed`

Trước khi sửa:

```bash
cp data/users.txt backup/users.before-sed.txt
```

Dùng `sed` chỉ để preview:

```bash
sed 's/disabled/inactive/g' data/users.txt
```

Điểm cực kỳ quan trọng:

Command trên **chưa sửa file**.

Nó chỉ output kết quả transformation.

Sau khi kiểm tra đúng mới:

```bash
sed -i 's/disabled/inactive/g' data/users.txt
```

Verify:

```bash
grep -n 'inactive' data/users.txt
```

Và:

```bash
diff -u backup/users.before-sed.txt data/users.txt
```

Flow bạn nên nói thành tiếng:

> "Trước khi chỉnh file em backup. Em preview transformation trước, sau đó mới dùng `sed -i` để sửa thật, rồi dùng `grep` và `diff` để validate."

Đây là đúng mindset mà Demo Guide yêu cầu đối với config/file modification. :chatgpt-content-reference{index="2"}

---

# 7. Phần 2 — User/group và least privilege

Ta dùng:

```text
Group: appops
User:  demoops
```

Đây chỉ là lab identity.

Trước tiên inspect, không create bừa:

```bash
getent group appops
```

```bash
getent passwd demoops
```

Nếu không có output thì chưa tồn tại.

Tạo group:

```bash
sudo groupadd appops
```

Verify:

```bash
getent group appops
```

Tạo user:

```bash
sudo useradd \
  -m \
  -s /bin/bash \
  -g appops \
  demoops
```

Ý nghĩa:

```text
-m
```

tạo home directory.

```text
-s /bin/bash
```

đặt login shell.

```text
-g appops
```

đặt primary group.

Verify:

```bash
id demoops
```

```bash
getent passwd demoops
```

Expected dạng:

```text
demoops:x:1001:...:/home/demoops:/bin/bash
```

---

# 8. Không cấp sudo cho user demo

Kiểm tra:

```bash
sudo -l -U demoops
```

Chúng ta **không** chạy:

```bash
sudo usermod -aG sudo demoops
```

Lý do:

> User chỉ cần quyền thực hiện nhiệm vụ được giao thì không nên được cấp full sudo. Đây là least privilege.

---

# 9. Disable password login cho `demoops`

Chạy:

```bash
sudo passwd -l demoops
```

Verify:

```bash
sudo passwd -S demoops
```

Bạn có thể giải thích:

> "Em không muốn dùng reusable password để SSH. Account này sẽ dùng public-key authentication."

---

# 10. Tạo controlled directory

Tạo:

```bash
sudo mkdir -p /srv/appops
sudo chown root:appops /srv/appops
sudo chmod 2770 /srv/appops
```

Kiểm tra:

```bash
ls -ld /srv/appops
```

Bạn có thể thấy dạng:

```text
drwxrws--- root appops ...
```

Permission:

```text
2 770
```

`770`:

```text
owner = rwx
group = rwx
others = ---
```

Leading `2`:

```text
setgid
```

trên directory.

Điều này giúp file/directory mới tạo bên trong inherit group `appops`.

Test:

```bash
sudo -u demoops touch /srv/appops/demoops-test.txt
```

```bash
ls -l /srv/appops/demoops-test.txt
```

Kiểm tra quyền:

```bash
sudo -u demoops cat /srv/appops/demoops-test.txt
```

Không cần `sudo` bên trong command của user đó; `sudo -u` ở đây chỉ để chúng ta mô phỏng hoạt động dưới identity `demoops`.

---

# 11. SSH key access — server-side

Vì bạn đã SSH thành công vào EC2 bằng public-key authentication nên:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Không chạy:

```bash
cat ~/.ssh/authorized_keys
```

trước camera nếu không cần thiết.

Public key không phải secret như private key, nhưng cũng không có lợi ích gì khi phơi toàn bộ nội dung trong recording.

Ta dùng public key authorization hiện tại để cấu hình lab user.

Tạo `.ssh`:

```bash
sudo install \
  -d \
  -m 700 \
  -o demoops \
  -g appops \
  /home/demoops/.ssh
```

Copy authorized public key:

```bash
sudo cp \
  /home/ubuntu/.ssh/authorized_keys \
  /home/demoops/.ssh/authorized_keys
```

Set ownership:

```bash
sudo chown \
  demoops:appops \
  /home/demoops/.ssh/authorized_keys
```

Set permission:

```bash
sudo chmod 600 \
  /home/demoops/.ssh/authorized_keys
```

Validate:

```bash
sudo stat \
  -c '%a %U %G %n' \
  /home/demoops/.ssh \
  /home/demoops/.ssh/authorized_keys
```

Expected:

```text
700 demoops appops /home/demoops/.ssh
600 demoops appops /home/demoops/.ssh/authorized_keys
```

Kiểm tra toàn bộ path:

```bash
namei -l /home/demoops/.ssh/authorized_keys
```

---

# 12. Validate SSH daemon configuration

Không restart SSH bừa bãi.

Trước hết:

```bash
sudo sshd -t
```

Expected invariant:

```text
no output
exit code 0
```

Kiểm tra:

```bash
echo $?
```

Expected:

```text
0
```

Sau đó xem effective config:

```bash
sudo sshd -T | grep -Ei \
'pubkeyauthentication|passwordauthentication|authorizedkeysfile'
```

Bạn cần hiểu ba setting này.

`PubkeyAuthentication` quyết định public-key authentication.

`PasswordAuthentication` quyết định password-based SSH authentication.

`AuthorizedKeysFile` xác định nơi SSH server tìm public key được phép.

**Chúng ta không sửa `sshd_config` trong demo nếu baseline hiện tại đã đúng.** Việc sửa SSH configuration không cần thiết có thể làm bạn tự khóa mình khỏi EC2.

Đây là một boundary rất quan trọng của administrator.

---

# 13. Phần 3 — Demo cài package

Kiểm tra package trước:

```bash
dpkg -s tree 2>/dev/null | grep '^Status:'
```

Nếu chưa cài:

```bash
sudo apt-get update
sudo apt-get install -y tree
```

Verify:

```bash
dpkg -s tree | grep -E '^(Package|Status|Version):'
```

```bash
command -v tree
```

Sau đó:

```bash
tree /srv/java-fresher/unit2
```

Không chạy:

```bash
sudo apt upgrade
```

chỉ để demo package install.

`apt upgrade` làm thay đổi phạm vi lớn hơn nhiều và không cần thiết.

---

# 14. Tạo custom systemd service

Ta tạo một HTTP service đơn giản trên port:

```text
8088
```

Port này **không cần mở trên Security Group**.

Tạo web root:

```bash
sudo mkdir -p /srv/appops/www
```

Tạo nội dung:

```bash
sudo tee /srv/appops/www/index.html >/dev/null <<'EOF'
Java Fresher Unit 2 systemd demo service
EOF
```

Ownership:

```bash
sudo chown -R root:appops /srv/appops/www
sudo chmod 2770 /srv/appops/www
sudo chmod 660 /srv/appops/www/index.html
```

---

# 15. Tạo unit file

```bash
sudo tee /etc/systemd/system/fresher-demo.service >/dev/null <<'EOF'
[Unit]
Description=Java Fresher Unit 2 Demo HTTP Service
After=network.target

[Service]
Type=simple
User=demoops
Group=appops
WorkingDirectory=/srv/appops/www
ExecStart=/usr/bin/python3 -m http.server 8088 --bind 0.0.0.0 --directory /srv/appops/www
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
EOF
```

Giải thích phần quan trọng.

```ini
[Unit]
```

metadata và dependencies.

```ini
After=network.target
```

service này được order sau network target.

```ini
Type=simple
```

process trong `ExecStart` chính là main process.

```ini
User=demoops
Group=appops
```

service **không chạy root**.

Đây là least privilege.

```ini
WorkingDirectory=/srv/appops/www
```

working directory của process.

```ini
ExecStart=...
```

command systemd chạy.

```text
8088
```

listening port.

```text
0.0.0.0
```

listen trên tất cả IPv4 interfaces của instance.

Security Group vẫn không mở TCP/8088 ra Internet, nên đây không đồng nghĩa port trở thành public.

```ini
Restart=on-failure
```

systemd tự restart nếu process unexpected failure.

```ini
WantedBy=multi-user.target
```

cho phép enable service trong normal multi-user boot.

---

# 16. Validate unit trước khi start

Đừng tạo file xong chạy ngay.

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/fresher-demo.service
```

Nếu không có error nghiêm trọng:

```bash
sudo systemctl daemon-reload
```

`daemon-reload` yêu cầu systemd đọc lại unit definitions.

Start + enable:

```bash
sudo systemctl enable --now fresher-demo.service
```

`enable` bản thân nó thiết lập service cho boot sau nhưng không nhất thiết start service ngay; `--now` yêu cầu thực hiện start cùng lúc. Đây vẫn là behavior được systemd mô tả trên Ubuntu hiện tại. :chatgpt-content-reference{index="3"}

---

# 17. Validate service

```bash
systemctl status fresher-demo.service --no-pager
```

Expected invariant:

```text
Active: active (running)
```

Check process:

```bash
ps -ef | grep '[p]ython3.*8088'
```

Check socket:

```bash
sudo ss -lntp | grep ':8088'
```

Expected:

```text
0.0.0.0:8088
```

Application validation:

```bash
curl -v http://127.0.0.1:8088/
```

Expected body:

```text
Java Fresher Unit 2 systemd demo service
```

Bạn nên nói:

> "`systemctl status` chứng minh service manager thấy process active. `ss` chứng minh socket đang listen. `curl` chứng minh application thực sự trả response. Em không dùng một command duy nhất để kết luận toàn bộ stack hoạt động."

Đây là tư duy vận hành rất quan trọng.

---

# 18. Failure injection — systemd service fail

Đây là phần rất quan trọng của Unit 2.

Guide yêu cầu không restart ngẫu nhiên mà phải dùng OSI Mindset để khoanh vùng. :chatgpt-content-reference{index="4"}

Trước khi inject failure, backup:

```bash
sudo cp -a \
  /etc/systemd/system/fresher-demo.service \
  /etc/systemd/system/fresher-demo.service.before-failure
```

Verify:

```bash
sudo ls -l \
  /etc/systemd/system/fresher-demo.service*
```

Inject lỗi:

```bash
sudo sed -i \
  's#^WorkingDirectory=.*#WorkingDirectory=/srv/appops/missing#' \
  /etc/systemd/system/fresher-demo.service
```

Validate thay đổi:

```bash
grep '^WorkingDirectory' \
  /etc/systemd/system/fresher-demo.service
```

Expected:

```text
WorkingDirectory=/srv/appops/missing
```

Apply unit definition:

```bash
sudo systemctl daemon-reload
```

Restart:

```bash
sudo systemctl restart fresher-demo.service
```

Command này dự kiến **fail**.

Đừng sửa ngay.

---

# 19. OSI Mindset khi service fail

Triệu chứng:

```bash
systemctl status fresher-demo.service --no-pager
```

Bạn có thể thấy:

```text
failed
```

Bây giờ bạn nên nói trước camera:

> "Service vừa hoạt động trước thay đổi và hiện systemd báo fail ngay khi start. Đây không có dấu hiệu ban đầu của lỗi VPC hay Security Group, nên em chọn **top-down** và kiểm tra application/service layer trước."

Đọc log:

```bash
sudo journalctl \
  -u fresher-demo.service \
  -n 30 \
  --no-pager
```

Tìm evidence kiểu:

```text
Failed at step CHDIR
```

hoặc lỗi liên quan WorkingDirectory.

Kiểm tra effective unit:

```bash
sudo systemctl cat fresher-demo.service
```

Kiểm tra path:

```bash
ls -ld /srv/appops/missing
```

Expected:

```text
No such file or directory
```

Bây giờ mới kết luận:

```text
Symptom:
service failed to start

Evidence:
journalctl reports working-directory/CHDIR failure

Root cause:
WorkingDirectory points to nonexistent directory

Layer:
Application/service configuration

Not root cause:
Security Group
VPC route
Internet Gateway
Tomcat
PostgreSQL
```

Đây chính xác là mindset:

```text
symptom
→ evidence
→ hypothesis
→ safe test
→ root cause
→ remediation
→ validation
```

---

# 20. Recovery

Khôi phục backup:

```bash
sudo cp -a \
  /etc/systemd/system/fresher-demo.service.before-failure \
  /etc/systemd/system/fresher-demo.service
```

Validate config:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/fresher-demo.service
```

Reload:

```bash
sudo systemctl daemon-reload
```

Start:

```bash
sudo systemctl restart fresher-demo.service
```

Validate:

```bash
systemctl is-active fresher-demo.service
```

Expected:

```text
active
```

Port:

```bash
sudo ss -lntp | grep ':8088'
```

Endpoint:

```bash
curl -fsS http://127.0.0.1:8088/
```

Expected:

```text
Java Fresher Unit 2 systemd demo service
```

Đây là **rollback/recovery hoàn chỉnh**, không chỉ restart.

---

# 21. Phần 4 — Connectivity diagnosis

Ta sử dụng chính custom service để mô phỏng lỗi network/socket một cách an toàn.

Trước khi thay đổi:

```bash
sudo cp -a \
  /etc/systemd/system/fresher-demo.service \
  /etc/systemd/system/fresher-demo.service.before-bind-test
```

Inject:

```bash
sudo sed -i \
  's/--bind 0.0.0.0/--bind 127.0.0.1/' \
  /etc/systemd/system/fresher-demo.service
```

Validate:

```bash
grep '^ExecStart' \
  /etc/systemd/system/fresher-demo.service
```

Reload/restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart fresher-demo.service
```

---

# 22. Tạo symptom

Lấy private IPv4 của EC2:

```bash
hostname -I
```

Có thể ra:

```text
10.10.10.25
```

Gán tự động:

```bash
PRIVATE_IP=$(hostname -I | awk '{print $1}')
echo "$PRIVATE_IP"
```

Local loopback:

```bash
curl -fsS http://127.0.0.1:8088/
```

thành công.

Nhưng:

```bash
curl -v "http://${PRIVATE_IP}:8088/"
```

dự kiến fail vì service chỉ bind `127.0.0.1`.

Bây giờ **không sửa ngay**.

---

# 23. Diagnose theo layer

## Network identity

```bash
ip addr
```

Ta xác nhận EC2 vẫn có private IP.

Route:

```bash
ip route
```

Interface/network không biến mất.

Ping chính private IP:

```bash
ping -c 2 "$PRIVATE_IP"
```

Nếu thành công, IP stack hoạt động.

---

## Transport

Đây là command quan trọng nhất:

```bash
sudo ss -lntp | grep ':8088'
```

Bạn sẽ thấy:

```text
127.0.0.1:8088
```

chứ không phải:

```text
0.0.0.0:8088
```

Root cause đã rõ.

Service chỉ listen trên loopback.

Không cần đi sang AWS Console kiểm tra Security Group, bởi connection từ chính EC2 tới private IP đã fail và `ss` đã chứng minh listener chỉ bind loopback.

Đây là đúng nguyên tắc của guide:

> khi đã tìm thấy tầng lỗi thì không tiếp tục kiểm tra lung tung các tầng khác. :chatgpt-content-reference{index="5"}

---

# 24. Recovery connectivity

Restore:

```bash
sudo cp -a \
  /etc/systemd/system/fresher-demo.service.before-bind-test \
  /etc/systemd/system/fresher-demo.service
```

Validate:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/fresher-demo.service
```

Apply:

```bash
sudo systemctl daemon-reload
sudo systemctl restart fresher-demo.service
```

Check:

```bash
sudo ss -lntp | grep ':8088'
```

Expected:

```text
0.0.0.0:8088
```

Test:

```bash
curl -fsS "http://${PRIVATE_IP}:8088/"
```

Expected success.

---

# 25. Kiểm tra storage/capacity

Unit 2 cần thể hiện `df -h`, nhưng chúng ta không cần fill root filesystem cho đầy trong demo chính.

Chạy:

```bash
df -h
```

Đặc biệt nhìn root filesystem:

```bash
df -h /
```

Ví dụ:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        29G  7.5G   21G  27% /
```

Bạn phải giải thích:

```text
Size  = filesystem capacity
Used  = đã sử dụng
Avail = còn khả dụng
Use%  = tỷ lệ sử dụng
Mounted on = mount point
```

Kiểm tra inode:

```bash
df -ih /
```

Đây là **Supplementary / Beyond explicit syllabus** nhưng rất hữu ích:

Filesystem có thể không hết GB nhưng hết inode do có quá nhiều file nhỏ.

---

# 26. `du` để tìm consumer

Nếu `/` cao:

```bash
sudo du -xhd1 /var 2>/dev/null | sort -h
```

Sau đó drill-down chẳng hạn:

```bash
sudo du -xhd1 /var/log 2>/dev/null | sort -h
```

Đừng thấy disk cao rồi chạy ngay:

```bash
sudo rm -rf ...
```

Quy trình phải là:

```text
df
→ identify filesystem
→ du
→ identify consumer
→ determine owner/service
→ cleanup according to policy
```

---

# 27. Phần 5 — Health-check + backup script

Ta sẽ viết một script có bốn trách nhiệm:

```text
service health
HTTP endpoint
disk capacity
backup bằng tar + rsync
```

Tạo backup directories:

```bash
sudo mkdir -p \
  /opt/java-fresher/backups/unit2 \
  /opt/java-fresher/backup-mirror/unit2
```

---

# 28. Tạo script

```bash
sudo tee /usr/local/sbin/fresher-health-backup.sh >/dev/null <<'EOF'
#!/usr/bin/env bash

set -uo pipefail

BACKUP_DIR="/opt/java-fresher/backups/unit2"
MIRROR_DIR="/opt/java-fresher/backup-mirror/unit2"
TIMESTAMP="$(date +%Y%m%d_%H%M%S)"
HEALTH_RC=0

log() {
    local message="$*"
    echo "$(date -Is) $message"
    logger -t fresher-health -- "$message"
}

check_service() {
    local service="$1"

    if systemctl is-active --quiet "$service"; then
        log "OK service=$service state=active"
    else
        log "ERROR service=$service state=inactive-or-failed"
        HEALTH_RC=1
    fi
}

log "START health-check"

for service in \
    tomcat10 \
    postgresql \
    prometheus \
    prometheus-node-exporter \
    grafana-server \
    fresher-demo.service
do
    check_service "$service"
done

if curl -fsS http://127.0.0.1:8088/ >/dev/null; then
    log "OK endpoint=http://127.0.0.1:8088/"
else
    log "ERROR endpoint=http://127.0.0.1:8088/"
    HEALTH_RC=1
fi

ROOT_USE="$(
    df -P / |
    awk 'NR==2 {gsub("%","",$5); print $5}'
)"

log "INFO root_filesystem_use=${ROOT_USE}%"

if (( ROOT_USE >= 85 )); then
    log "ERROR root filesystem usage is at or above 85%"
    HEALTH_RC=1
fi

mkdir -p "$BACKUP_DIR" "$MIRROR_DIR"

ARCHIVE="${BACKUP_DIR}/unit2-${TIMESTAMP}.tar.gz"

if tar \
    -C / \
    -czf "$ARCHIVE" \
    srv/appops \
    etc/systemd/system/fresher-demo.service
then
    log "OK tar_backup=$ARCHIVE"
else
    log "ERROR tar backup failed"
    exit 10
fi

if rsync \
    -a \
    "$BACKUP_DIR/" \
    "$MIRROR_DIR/"
then
    log "OK rsync_mirror=$MIRROR_DIR"
else
    log "ERROR rsync failed"
    exit 11
fi

log "RESULT health_exit_code=$HEALTH_RC"
exit "$HEALTH_RC"
EOF
```

---

# 29. Giải thích script

```bash
set -uo pipefail
```

`-u`: dùng undefined variable sẽ lỗi.

`pipefail`: pipeline không che giấu failure ở command phía trước.

Ở đây tôi **không dùng `-e`** có chủ đích.

Lý do script health-check cần tiếp tục kiểm tra các service khác ngay cả khi một check fail, để thu thập đầy đủ tình trạng.

Đây là một điểm hay để giải thích trong demo.

---

`HEALTH_RC=0`

Ban đầu:

```text
0 = healthy
```

Nếu bất kỳ health check nào fail:

```text
1 = unhealthy
```

Còn:

```text
10 = tar failure
11 = rsync failure
```

giúp phân biệt loại lỗi.

---

# 30. Hàm `log`

```bash
logger -t fresher-health -- "$message"
```

đẩy message vào system logging/journal với tag:

```text
fresher-health
```

Sau đó ta đọc bằng:

```bash
journalctl -t fresher-health
```

Đây là cách kết nối Bash script với operational logging.

---

# 31. Service loop

```bash
for service in ...
do
    check_service "$service"
done
```

Kiểm tra:

```text
Tomcat
PostgreSQL
Prometheus
node_exporter
Grafana
custom service
```

Mỗi service dùng:

```bash
systemctl is-active --quiet
```

Không parse human-readable `systemctl status`.

---

# 32. HTTP check

```bash
curl -fsS http://127.0.0.1:8088/
```

Đây là application-level validation.

Không chỉ kiểm tra process.

---

# 33. Disk threshold

```bash
df -P /
```

`-P` dùng output format ổn định hơn để parse.

```bash
awk 'NR==2 {gsub("%","",$5); print $5}'
```

Lấy field `% Use`.

Ví dụ:

```text
27%
```

thành:

```text
27
```

Sau đó:

```bash
(( ROOT_USE >= 85 ))
```

threshold lab là:

```text
85%
```

---

# 34. Backup bằng `tar`

```bash
tar \
    -C / \
    -czf "$ARCHIVE" \
    srv/appops \
    etc/systemd/system/fresher-demo.service
```

Options:

```text
-c = create
-z = gzip
-f = archive filename
-C / = đổi base directory trước khi archive
```

Ta backup cả:

```text
/srv/appops
/etc/systemd/system/fresher-demo.service
```

Tức là cả data lẫn configuration.

---

# 35. Backup bằng `rsync`

```bash
rsync -a \
    "$BACKUP_DIR/" \
    "$MIRROR_DIR/"
```

`-a` = archive mode.

Lưu ý dấu `/`:

```text
"$BACKUP_DIR/"
```

nghĩa là sync **contents bên trong directory**.

---

# 36. Permission cho script

```bash
sudo chown root:root \
  /usr/local/sbin/fresher-health-backup.sh
```

```bash
sudo chmod 750 \
  /usr/local/sbin/fresher-health-backup.sh
```

Verify:

```bash
ls -l /usr/local/sbin/fresher-health-backup.sh
```

---

# 37. Syntax validation

Trước khi chạy:

```bash
sudo bash -n \
  /usr/local/sbin/fresher-health-backup.sh
```

Expected:

```text
no output
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

# 38. Manual execution

```bash
sudo /usr/local/sbin/fresher-health-backup.sh
```

Ngay lập tức capture exit code:

```bash
echo "exit_code=$?"
```

Expected healthy case:

```text
exit_code=0
```

Đây là điểm quan trọng.

Nếu bạn chạy thêm một command trước:

```bash
sudo /usr/local/sbin/fresher-health-backup.sh
ls
echo $?
```

thì `$?` lúc này là exit code của `ls`, **không còn của script**.

Vì vậy phải capture ngay.

---

# 39. Validate tar backup

```bash
sudo ls -lh \
  /opt/java-fresher/backups/unit2
```

Xem archive:

```bash
LATEST="$(
    sudo find /opt/java-fresher/backups/unit2 \
      -type f \
      -name '*.tar.gz' \
      -printf '%T@ %p\n' |
    sort -n |
    tail -1 |
    cut -d' ' -f2-
)"
```

```bash
echo "$LATEST"
```

Listing mà không extract:

```bash
sudo tar -tzf "$LATEST"
```

Điều này rất quan trọng:

> Backup file tồn tại chưa đủ. Ít nhất phải chứng minh archive có thể được đọc/list.

---

# 40. Validate rsync mirror

```bash
sudo find \
  /opt/java-fresher/backup-mirror/unit2 \
  -maxdepth 1 \
  -type f \
  -ls
```

Bạn cần phân biệt:

```text
tar    = tạo archive
rsync  = synchronize/copy directory tree
```

Không xem chúng là hai command giống nhau.

---

# 41. Đọc log bằng journalctl

```bash
sudo journalctl \
  -t fresher-health \
  -n 30 \
  --no-pager
```

Bạn nên thấy dạng:

```text
OK service=tomcat10 state=active
OK service=postgresql state=active
...
OK endpoint=http://127.0.0.1:8088/
INFO root_filesystem_use=...
OK tar_backup=...
OK rsync_mirror=...
RESULT health_exit_code=0
```

---

# 42. Lập lịch bằng cron

Kiểm tra cron:

```bash
systemctl status cron --no-pager
```

Nếu đang active thì không cần restart.

Tạo cron file:

```bash
sudo tee /etc/cron.d/fresher-health-backup >/dev/null <<'EOF'
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

*/5 * * * * root /usr/local/sbin/fresher-health-backup.sh >> /var/log/fresher-health-backup-cron.log 2>&1
EOF
```

Permissions:

```bash
sudo chown root:root \
  /etc/cron.d/fresher-health-backup
```

```bash
sudo chmod 644 \
  /etc/cron.d/fresher-health-backup
```

Verify:

```bash
sudo cat /etc/cron.d/fresher-health-backup
```

Format cron hiện tại vẫn gồm các trường minute/hour/day/month/day-of-week; cron kiểm tra entries định kỳ theo minute. :chatgpt-content-reference{index="6"}

Dòng:

```text
*/5 * * * *
```

nghĩa là:

```text
every 5 minutes
```

Vì đây là `/etc/cron.d`, có thêm field:

```text
root
```

để xác định user chạy job.

---

# 43. Kiểm tra cron log

```bash
sudo journalctl \
  -u cron \
  -n 30 \
  --no-pager
```

Ubuntu cron quản lý scheduled commands và per-user/system crontabs; current Ubuntu documentation vẫn mô tả `/etc/crontab`, `/etc/cron.d` và user crontabs là các vị trí scheduling tiêu chuẩn. :chatgpt-content-reference{index="7"}

---

# 44. Failure injection cho health-check

Ta inject một failure rất an toàn:

```bash
sudo systemctl stop fresher-demo.service
```

Trước khi chạy health check, nói:

> "Em đã cố ý stop custom service để chứng minh script phát hiện failure. Em không đụng vào Tomcat/PostgreSQL production-like services."

Run:

```bash
sudo /usr/local/sbin/fresher-health-backup.sh
```

Capture ngay:

```bash
RC=$?
echo "exit_code=$RC"
```

Expected:

```text
exit_code=1
```

---

# 45. Xem journal

```bash
sudo journalctl \
  -t fresher-health \
  -n 30 \
  --no-pager
```

Bạn phải thấy:

```text
ERROR service=fresher-demo.service state=inactive-or-failed
ERROR endpoint=http://127.0.0.1:8088/
RESULT health_exit_code=1
```

Đây chính là:

```text
script
→ detect failure
→ non-zero exit code
→ journal evidence
```

---

# 46. Diagnose chứ không restart ngay

Kiểm tra:

```bash
systemctl status fresher-demo.service --no-pager
```

Trong case này:

```text
inactive
```

bởi vì chúng ta stop có chủ đích.

Xem recent log:

```bash
sudo journalctl \
  -u fresher-demo.service \
  -n 20 \
  --no-pager
```

Sau khi đã biết nguyên nhân:

```bash
sudo systemctl start fresher-demo.service
```

Validate:

```bash
systemctl is-active fresher-demo.service
```

```bash
curl -fsS http://127.0.0.1:8088/
```

Sau đó health script:

```bash
sudo /usr/local/sbin/fresher-health-backup.sh
RC=$?
echo "exit_code=$RC"
```

Expected:

```text
exit_code=0
```

Đây là full cycle:

```text
healthy
→ failure injection
→ detection
→ evidence
→ diagnosis
→ remediation
→ validation
```

---

# 47. Cách trình bày OSI Mindset trong Unit 2

Bạn nên dùng **systemd failure scenario** làm tình huống chính.

Không nên nói:

> "Service fail nên em restart."

Mà nên nói:

> "Em thấy `fresher-demo.service` fail sau một configuration change. Vì triệu chứng nằm ngay ở service/application, em chọn top-down. Em kiểm tra `systemctl status`, sau đó `journalctl -u fresher-demo`, log chỉ ra lỗi WorkingDirectory. Em kiểm tra path và xác nhận directory không tồn tại. Như vậy lỗi nằm ở service configuration/Application layer, không phải network hay AWS. Sau khi xác định root cause em restore bản backup, validate unit, daemon-reload, restart và kiểm tra lại bằng `systemctl`, `ss` và `curl`."

Đó là câu trả lời rất mạnh cho practical.

---

# 48. Trường hợp nào mới mở AWS Console?

Ví dụ SSH hoặc application thật sự inaccessible từ bên ngoài.

Khi đó theo quy ước của chúng ta, bạn mở:

```text
AWS Console
→ EC2
→ Instances
```

kiểm tra:

```text
Instance state
Status checks
Public IPv4
Subnet
Security Group
```

Sau đó:

```text
VPC Console
→ Route Tables
→ Security Groups
```

Nếu nghi Layer 3/4.

**Không dùng:**

```bash
aws ec2 ...
```

Từ đây về sau tôi sẽ giữ nguyên quy tắc này.

---

# 49. Update `history.log` cuối Unit 2

Vì Unit 1 chúng ta đã cấu hình timestamp, cuối Unit 2 trên **EC2** chạy:

```bash
history -a
```

Sau đó:

```bash
mkdir -p ~/java-fresher-demo/history
```

Master history:

```bash
history > \
  ~/java-fresher-demo/history/history.log
```

Snapshot riêng Unit 2:

```bash
history > \
  ~/java-fresher-demo/history/"$(date +%F)_unit02_linux.log"
```

Verify:

```bash
tail -20 \
  ~/java-fresher-demo/history/history.log
```

Bạn muốn format kiểu:

```text
812  2026-10-06 03:10:23 sudo systemctl stop fresher-demo.service
813  2026-10-06 03:10:30 sudo /usr/local/sbin/fresher-health-backup.sh
814  2026-10-06 03:10:36 echo $?
815  2026-10-06 03:10:45 sudo journalctl -t fresher-health -n 30 --no-pager
```

Bạn vẫn phải giải thích được những command này khi Self Q&A.

---

# 50. Những command bạn bắt buộc phải hiểu sau Unit 2

Đây là nhóm duy nhất tôi muốn bạn đặc biệt ôn trước khi quay practical:

```text
pwd / cd / ls / mkdir / cp / mv / rm / touch
cat / less
grep
awk
sed
diff

whoami
id
getent
groupadd
useradd
passwd
chown
chmod
stat
namei

apt-get
dpkg

ps
systemctl
systemd-analyze
journalctl

ip addr
ip route
ping
ss
curl
df
du

tar
rsync

bash
$?
cron
logger
history
```

Không cần thuộc hàng trăm option, nhưng những option **chính bạn dùng trong recording** phải giải thích được.

---

# 51. Rollback trạng thái failure injection

Trước khi kết thúc Unit 2, xác nhận custom service đang bình thường:

```bash
systemctl is-active fresher-demo.service
```

Expected:

```text
active
```

```bash
sudo ss -lntp | grep ':8088'
```

```bash
curl -fsS http://127.0.0.1:8088/
```

Kiểm tra toàn bộ main stack:

```bash
for svc in \
  tomcat10 \
  postgresql \
  prometheus \
  prometheus-node-exporter \
  grafana-server \
  fresher-demo.service
do
    printf '%-30s ' "$svc"
    systemctl is-active "$svc"
done
```

Expected:

```text
tomcat10                       active
postgresql                     active
prometheus                     active
prometheus-node-exporter       active
grafana-server                 active
fresher-demo.service           active
```

Disk:

```bash
df -h /
```

Failed systemd units:

```bash
systemctl --failed --no-pager
```

Sau đó:

```bash
sudo /usr/local/sbin/fresher-health-backup.sh
echo "exit_code=$?"
```

Final expected invariant:

```text
exit_code=0
```

---

## Unit 2 hoàn thành khi nào?

Về practical, tôi sẽ coi Unit 2 của bạn đạt khi bạn tự thực hiện và giải thích được chuỗi:

```text
CLI/files
    ↓
grep / awk / sed
    ↓
user/group
    ↓
ownership + permissions
    ↓
SSH public-key access
    ↓
package installation
    ↓
systemd service
    ↓
failure injection
    ↓
systemctl + journalctl diagnosis
    ↓
rollback/recovery
    ↓
ip / route / ping / ss / curl
    ↓
df / du
    ↓
health-check Bash script
    ↓
tar + rsync
    ↓
cron
    ↓
non-zero exit code
    ↓
journal evidence
    ↓
recovery + validation
```

Đây bao phủ toàn bộ **mục 1–5 của Practical Unit 2**, đồng thời đã **loại mục 6 evidence Day 2–6** đúng theo yêu cầu của bạn. Phần custom `fresher-demo.service` và failure injection được thiết kế riêng để bạn có một tình huống **an toàn, có thể lặp lại và rollback rõ ràng**, thay vì cố tình làm hỏng Tomcat/PostgreSQL thật trong buổi demo.
