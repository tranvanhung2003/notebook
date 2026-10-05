# UNIT 1 — ORIENTATION & ADMINISTRATION FOUNDATIONS

## Thời lượng mục tiêu: khoảng 3 phút

Unit 1 yêu cầu trả lời ba nội dung: scope/access boundaries, runbook/evidence và lab readiness checklist.

---

## Câu 1. Phạm vi trách nhiệm và access boundaries của L1.5 Administrator là gì? Việc gì được làm và việc gì phải xin duyệt?

Một L1.5 Administrator là người thực hiện các công việc vận hành kỹ thuật ở mức đã được phân quyền và đã có procedure hoặc runbook rõ ràng. Nguyên tắc quan trọng nhất là **không đồng nghĩa có quyền kỹ thuật với có quyền nghiệp vụ để thực hiện thay đổi**.

Ví dụ, nếu account của em có `sudo`, điều đó không có nghĩa em được phép tự ý restart service production, sửa firewall hoặc thay đổi database. Quyền kỹ thuật chỉ cho phép thao tác; còn việc có được phép thao tác hay không phụ thuộc vào **scope, approval và change process**.

### Những việc L1.5 thường có thể thực hiện trong phạm vi được giao

Ví dụ:

- kiểm tra trạng thái server và service;
- đọc log;
- kiểm tra CPU, memory, disk;
- kiểm tra socket và port;
- chạy health check;
- kiểm tra endpoint bằng `curl`;
- kiểm tra connectivity;
- kiểm tra process;
- restart một service nếu runbook và ticket cho phép;
- thực hiện deployment hoặc rollback đã được phê duyệt;
- thực hiện backup/restore trong lab hoặc theo approved procedure;
- thu thập evidence;
- update ticket và handover.

Ví dụ khi Tomcat có vấn đề, em có thể bắt đầu bằng:

```bash
systemctl status tomcat
journalctl -u tomcat
ss -lntp
curl -v http://localhost:8080/
```

Đây là các thao tác quan sát hoặc kiểm tra, blast radius thấp.

### Những việc không nên tự ý thực hiện

Ví dụ:

- xóa dữ liệu production;
- thay đổi firewall hoặc Security Group ngoài ticket;
- mở port public tùy ý;
- thay đổi IAM policy;
- cấp administrator permission;
- thay đổi schema/database production;
- kill database session quan trọng khi chưa hiểu ảnh hưởng;
- reboot hoặc terminate production EC2;
- thay đổi JVM memory một cách tùy ý;
- disable security control;
- sửa config production không có backup;
- rollback ngoài approved procedure;
- dùng credential của người khác.

Nếu cần thực hiện ngoài scope thì phải **escalate hoặc xin approval**.

### Câu chốt nên nói

> L1.5 không được đánh giá bằng việc sửa được càng nhiều càng tốt. Một administrator tốt phải biết cái gì mình được phép làm, cái gì phải dừng lại, thu thập evidence và escalate.

---

## Câu 2. Runbook và evidence có vai trò gì? Tại sao mọi thao tác phải có bằng chứng?

### Runbook

**Runbook** là tài liệu mô tả procedure chuẩn để thực hiện một operational task.

Một runbook tốt thường cho biết:

- mục tiêu;
- prerequisites;
- quyền cần thiết;
- bước kiểm tra trước khi thay đổi;
- command/config cần sử dụng;
- expected result;
- validation;
- rollback;
- escalation condition.

Runbook giúp các administrator thực hiện cùng một công việc theo cách **repeatable, controlled và predictable**, thay vì mỗi người làm theo kinh nghiệm riêng.

Ví dụ trước khi restart Tomcat, runbook có thể yêu cầu:

```bash
systemctl status tomcat
ss -lntp | grep 8080
df -h
free -h
```

sau đó mới restart nếu đủ điều kiện.

### Evidence

Evidence là bằng chứng chứng minh:

1. trạng thái trước thay đổi;
2. thao tác đã thực hiện;
3. trạng thái sau thay đổi;
4. kết quả cuối cùng.

Có thể gọi theo mô hình:

**BEFORE → ACTION → AFTER → RESULT**

Ví dụ deployment:

**Before**

```bash
curl http://localhost:8080/app/health
sha256sum app.war
systemctl status tomcat
```

**Action**

Deploy WAR mới.

**After**

```bash
systemctl status tomcat
journalctl -u tomcat
curl http://localhost:8080/app/health
```

**Result**

Endpoint trả về thành công, log không xuất hiện exception nghiêm trọng.

Evidence quan trọng vì:

- phục vụ audit;
- chứng minh task đã hoàn thành;
- hỗ trợ troubleshooting;
- xác định recent change;
- hỗ trợ rollback;
- giúp shift tiếp theo hiểu chuyện gì đã xảy ra;
- tránh tranh luận dựa trên trí nhớ.

---

## Câu 3. Lab readiness checklist gồm những gì?

Trước một ngày lab, em sẽ kiểm tra ít nhất các nhóm sau.

### 1. Access

- SSH access hoạt động;
- AWS account/profile đúng;
- không dùng credential của người khác;
- quyền đủ cho bài lab nhưng không vượt quá cần thiết.

### 2. Environment

Kiểm tra:

```bash
hostname
whoami
pwd
ip addr
df -h
free -h
```

Đảm bảo đang thao tác đúng host và tài nguyên đủ.

### 3. Tools

Ví dụ:

```bash
java -version
aws --version
psql --version
curl --version
```

### 4. Evidence folder

Chuẩn bị thư mục lưu:

- command history;
- logs;
- screenshots;
- configuration backup;
- result;
- ticket hoặc notes.

### 5. Backup và rollback

Nếu sửa configuration:

```bash
sudo cp application.conf application.conf.bak
```

Phải biết trước:

- nếu config sai thì restore file nào;
- nếu service không lên thì rollback như thế nào.

### 6. Cost và safety

Trong AWS phải biết:

- resource nào đang chạy;
- resource nào phát sinh cost;
- sau lab phải stop/delete cái gì.

### Câu chốt Unit 1

> Trước khi thao tác em phải biết mình đang ở đâu, được phép làm gì, expected result là gì, evidence lưu ở đâu và nếu thao tác thất bại thì rollback thế nào.

---

# UNIT 2 — LINUX FUNDAMENTALS

## Thời lượng mục tiêu: khoảng 12 phút

Đây là một trong hai Unit trọng tâm nhất. Guide yêu cầu permissions/sudo, user/group/SSH, systemd, networking/storage, log/text processing/Bash/backup/cron và exit code.

---

# Câu 1. Linux file permissions hoạt động như thế nào? Dùng sudo an toàn theo least privilege ra sao?

Linux permission cơ bản chia đối tượng thành:

- **owner**;
- **group**;
- **others**.

Và ba quyền cơ bản:

- `r` — read;
- `w` — write;
- `x` — execute.

Ví dụ:

```bash
-rwxr-x---
```

Có thể đọc thành:

```text
owner:  rwx
group:  r-x
others: ---
```

Tức owner có read/write/execute, group có read/execute, người khác không có quyền.

### Numeric permissions

Giá trị:

```text
r = 4
w = 2
x = 1
```

Do đó:

```text
7 = rwx
6 = rw-
5 = r-x
4 = r--
```

Ví dụ:

```bash
chmod 750 deploy.sh
```

có nghĩa:

```text
owner = 7 = rwx
group = 5 = r-x
other = 0 = ---
```

### Ownership

Kiểm tra:

```bash
ls -l
```

Thay đổi:

```bash
sudo chown appuser:appgroup application.conf
```

### Permission trên directory

Một điểm dễ bị hỏi:

Đối với directory:

- `r`: xem danh sách tên file;
- `w`: tạo/xóa entry trong directory;
- `x`: traverse/access vào directory.

Vì vậy `x` trên directory đặc biệt quan trọng.

---

## Sudo và least privilege

`sudo` cho phép chạy command với quyền của user khác, thường là root.

Không nên:

```bash
sudo su -
```

rồi làm mọi thứ bằng root nếu không cần thiết.

Nên chạy đúng command cần privilege:

```bash
sudo systemctl restart tomcat
```

thay vì duy trì root shell dài.

Nguyên tắc:

> **Minimum privilege, minimum time, minimum scope.**

Trước lệnh nguy hiểm phải kiểm tra target.

Ví dụ trước:

```bash
sudo rm -rf ...
```

phải đặc biệt xác minh path. Trong operations thực tế, destructive command cần cực kỳ thận trọng.

---

# Câu 2. Quy trình tạo user/group và SSH key. Vì sao production không nên phụ thuộc password login?

Ví dụ cần tạo service/operator account.

### Bước 1 — Tạo group

```bash
sudo groupadd appops
```

### Bước 2 — Tạo user

```bash
sudo useradd -m -s /bin/bash operator1
```

Hoặc trên Ubuntu thường dùng:

```bash
sudo adduser operator1
```

### Bước 3 — Add group

```bash
sudo usermod -aG appops operator1
```

Cần nhớ `-aG`.

Không dùng thiếu `-a` một cách bất cẩn vì có thể làm thay đổi supplementary group membership ngoài ý muốn.

Verify:

```bash
id operator1
groups operator1
```

---

## SSH key

SSH public-key authentication dùng một cặp:

```text
private key
public key
```

Private key phải được giữ bí mật ở client.

Public key được đưa lên server:

```text
~/.ssh/authorized_keys
```

Permissions thường cần chặt:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### Authentication flow

Client chứng minh rằng mình sở hữu private key tương ứng với public key server cho phép.

Private key **không được copy lên server để đăng nhập**.

---

## Vì sao production ưu tiên SSH key hơn password?

Password có rủi ro:

- brute force;
- password reuse;
- credential stuffing;
- phishing;
- password yếu;
- bị chia sẻ giữa người dùng.

SSH key cung cấp credential có entropy cao hơn và có thể quản lý theo từng user/key.

Trong production còn có thể kết hợp:

- disable direct root login;
- MFA hoặc bastion/SSM tùy architecture;
- restricted source IP;
- key rotation;
- centralized identity.

**Supplementary / Beyond explicit syllabus:** Không nên nói “SSH key tuyệt đối an toàn”. Nếu private key bị lộ và không được bảo vệ phù hợp thì vẫn là security incident.

---

# Câu 3. Quản lý systemd service và recover service bị fail

`systemd` là service manager phổ biến trên nhiều Linux distribution hiện đại.

Các command cơ bản:

```bash
systemctl status tomcat
systemctl start tomcat
systemctl stop tomcat
systemctl restart tomcat
```

Enable start at boot:

```bash
sudo systemctl enable tomcat
```

Kiểm tra:

```bash
systemctl is-active tomcat
systemctl is-enabled tomcat
```

### Restart khác reload

`restart` dừng rồi khởi động service lại.

`reload`, nếu service hỗ trợ, yêu cầu process reload configuration mà không thực hiện full restart.

Không được mặc định cho rằng service nào cũng hỗ trợ reload.

---

## Khi service failed

Không nên ngay lập tức:

```bash
sudo systemctl restart tomcat
```

liên tục.

Thay vào đó:

### 1. Xem trạng thái

```bash
systemctl status tomcat
```

### 2. Đọc log

```bash
journalctl -u tomcat
```

Hoặc gần nhất:

```bash
journalctl -u tomcat -n 100
```

### 3. Kiểm tra config/recent change

Các nguyên nhân thường gặp:

- config syntax sai;
- file permission sai;
- binary/path sai;
- port conflict;
- environment variable thiếu;
- disk full;
- Java không tồn tại;
- dependency fail.

### 4. Kiểm tra port

```bash
ss -lntp
```

### 5. Fix root cause trong scope

Sau đó:

```bash
sudo systemctl restart tomcat
systemctl status tomcat
```

### 6. Validate application

```bash
curl -v http://localhost:8080/
```

Đó mới là recovery đầy đủ.

---

# Câu 4. `ip`, `ss`, `ping`, `traceroute`, `curl` dùng thế nào để troubleshoot network? Storage kiểm tra ra sao?

Em sẽ gắn chúng vào từng layer thay vì ghi nhớ riêng từng command.

### `ip`

Kiểm tra network interface và address:

```bash
ip addr
```

Route:

```bash
ip route
```

Trả lời:

- máy có IP không?
- interface up không?
- default route là gì?

---

### `ping`

```bash
ping <destination>
```

Dùng để kiểm tra reachability bằng ICMP.

Nhưng:

> Ping fail không có nghĩa server chắc chắn down.

ICMP có thể bị firewall/Security Group chặn.

---

### `traceroute`

```bash
traceroute <destination>
```

Giúp quan sát path qua các hop.

Có ích khi nghi routing/network path.

---

### `ss`

Ví dụ:

```bash
ss -lntp
```

Ý nghĩa thường dùng:

- `l` = listening;
- `n` = numeric;
- `t` = TCP;
- `p` = process.

Ví dụ Tomcat expected port 8080 nhưng:

```bash
ss -lntp | grep 8080
```

không có output thì ứng dụng có thể chưa listen.

Đây là evidence quan trọng ở Transport layer.

---

### `curl`

Ví dụ:

```bash
curl -v http://localhost:8080/health
```

Có thể kiểm tra:

- TCP connection;
- HTTP request;
- HTTP status;
- response headers/body;
- TLS với HTTPS ở mức cơ bản.

Điểm mạnh của `curl` là kiểm tra gần Application layer.

---

## Cách kết hợp

Ví dụ user báo:

> Không truy cập được Java application.

Em không restart ngay.

Em kiểm tra:

```bash
ip addr
ip route
ping <target>
ss -lntp | grep 8080
curl -v http://localhost:8080/
```

Nếu local curl thành công nhưng remote không truy cập được thì app có thể bình thường; focus chuyển sang firewall, SG, NACL, routing hoặc bind address.

---

## Storage troubleshooting

Các lệnh quan trọng:

```bash
df -h
```

Kiểm tra filesystem usage.

```bash
df -i
```

Kiểm tra inode usage.

```bash
du -sh /path/*
```

Tìm directory tiêu tốn space.

```bash
lsblk
```

Xem block devices.

```bash
findmnt
```

Kiểm tra mount.

Ví dụ filesystem 100% có thể làm app không ghi được log, PostgreSQL gặp lỗi ghi dữ liệu hoặc service start thất bại.

---

# Câu 5. `journalctl`, `grep`, `awk`, `sed`, Bash backup, tar/rsync và cron

## journalctl

Dùng với systemd journal.

Ví dụ:

```bash
journalctl -u tomcat
```

Log gần nhất:

```bash
journalctl -u tomcat -n 100
```

Từ boot hiện tại:

```bash
journalctl -b
```

Có thể search lỗi:

```bash
journalctl -u tomcat | grep -i error
```

---

## grep

Tìm pattern:

```bash
grep -i "error" application.log
```

Recursive:

```bash
grep -R "connection refused" /var/log/
```

---

## awk

Rất hữu ích với structured text.

Ví dụ:

```bash
awk '{print $1}' access.log
```

Lấy field đầu tiên.

Ví dụ disk:

```bash
df -h | awk '{print $1, $5, $6}'
```

---

## sed

Thường dùng transform text.

Ví dụ:

```bash
sed 's/old/new/g' file.txt
```

Trong production cần thận trọng với `sed -i` vì nó sửa trực tiếp file.

Nên:

```bash
cp config.conf config.conf.bak
```

trước.

---

# Bash backup

Một script backup đơn giản về tư duy có thể là:

```bash
#!/bin/bash

SOURCE="/opt/app/config"
DEST="/backup"
DATE=$(date +%F_%H-%M-%S)

tar -czf "$DEST/config-$DATE.tar.gz" "$SOURCE"

if [ $? -ne 0 ]; then
    echo "Backup failed"
    exit 1
fi

echo "Backup successful"
exit 0
```

Điểm cần giải thích:

- shebang chọn interpreter;
- variable tránh hard-code lặp lại;
- `tar -czf` tạo gzip-compressed archive;
- `$?` chứa exit status command trước;
- `exit 0` biểu thị success theo convention Unix;
- non-zero thể hiện failure hoặc trạng thái khác tùy chương trình.

Trong script tốt hơn có thể dùng:

```bash
set -euo pipefail
```

nhưng đây là **Supplementary / Beyond explicit syllabus**, và phải hiểu hành vi trước khi sử dụng.

---

## rsync

Ví dụ:

```bash
rsync -av /opt/app/config/ /backup/config/
```

`rsync` phù hợp synchronization/incremental file copy hơn là tạo archive như `tar`.

---

## cron

Kiểm tra:

```bash
crontab -l
```

Edit:

```bash
crontab -e
```

Ví dụ:

```cron
0 2 * * * /opt/scripts/backup.sh >> /var/log/backup.log 2>&1
```

Nghĩa là 02:00 mỗi ngày.

Một lỗi thường gặp là script chạy tay được nhưng cron fail do:

- `PATH` khác;
- working directory khác;
- permission;
- environment variables không tồn tại.

Vì vậy nên dùng absolute paths khi phù hợp và ghi log.

---

# Câu 6. Exit code khác 0 nghĩa là gì và xử lý thế nào?

Convention phổ biến của Unix/Linux:

```text
0     → success
non-0 → failure hoặc trạng thái đặc biệt
```

Nhưng ý nghĩa chính xác của từng non-zero code phụ thuộc command/program.

Kiểm tra command trước:

```bash
echo $?
```

Nếu script trả non-zero, em không chạy tiếp một cách mù quáng.

Quy trình:

1. xác định step nào fail;
2. ghi lại exit code;
3. đọc stdout/stderr;
4. kiểm tra log;
5. xác định dependency;
6. đánh giá có an toàn retry không;
7. fix trong scope hoặc escalate;
8. chạy lại;
9. verify exit code và expected result.

Ví dụ:

```bash
tar ...
RC=$?

if [ "$RC" -ne 0 ]; then
    echo "Backup failed with code $RC"
    exit "$RC"
fi
```

Không được nói:

> Non-zero lúc nào cũng bằng 1.

Điều đó sai. Có nhiều exit codes khác nhau.

---

# UNIT 3 — AWS CLOUD FUNDAMENTALS

## Thời lượng mục tiêu: khoảng 12 phút

Guide yêu cầu Shared Responsibility, IAM/MFA/CLI/tagging, VPC networking, EC2 lifecycle, EBS snapshot/restore và CloudWatch/Budget/cost.

---

# Câu 1. AWS Shared Responsibility Model

AWS chia security responsibility thành hai phần:

## Security **of** the cloud — AWS chịu trách nhiệm

AWS bảo vệ infrastructure chạy các AWS services, ví dụ:

- physical data centers;
- physical hosts;
- underlying networking infrastructure;
- hardware;
- virtualization infrastructure thuộc trách nhiệm AWS.

## Security **in** the cloud — Customer chịu trách nhiệm

Tùy service, khách hàng chịu trách nhiệm cho những thứ như:

- IAM;
- user permissions;
- OS patching của EC2;
- application configuration;
- Security Groups;
- data;
- secrets;
- encryption choices;
- network configuration.

AWS mô tả đây là security **of** the cloud và security **in** the cloud.

Ví dụ:

AWS chịu trách nhiệm bảo vệ physical server chạy EC2.

Nhưng nếu em mở:

```text
SSH 22 → 0.0.0.0/0
```

và server bị attack, configuration đó thuộc responsibility của customer.

Điểm quan trọng:

> Dùng cloud không có nghĩa AWS chịu trách nhiệm cho mọi security configuration của chúng ta.

---

# Câu 2. IAM user, role, policy, MFA, CLI profile và tagging

## IAM user

Đại diện cho identity trong AWS account. Có thể có console/CLI access tùy cấu hình.

## IAM policy

Policy định nghĩa permission.

Ví dụ tư duy:

```text
Effect
Action
Resource
Condition
```

Policy trả lời:

> Identity này được hoặc không được thực hiện action nào trên resource nào và trong điều kiện nào?

## IAM role

Role cũng là AWS identity nhưng thường được **assume** để nhận temporary credentials thay vì gắn permanent long-lived credential như cách truyền thống của IAM user.

Ví dụ EC2 cần đọc S3:

Tốt hơn là gắn IAM role phù hợp cho EC2 thay vì hard-code AWS access key vào application config.

---

## Least privilege

Không nên:

```text
Action: *
Resource: *
```

nếu workload chỉ cần một số quyền nhỏ.

Phải cấp minimum permissions cần cho task.

---

## MFA

MFA bổ sung authentication factor.

Password bị lộ chưa đủ để attacker đăng nhập nếu factor bổ sung vẫn được bảo vệ.

Đặc biệt quan trọng với privileged identities.

---

## AWS CLI profile

Có thể kiểm tra:

```bash
aws configure list
aws sts get-caller-identity
```

Lệnh `get-caller-identity` rất quan trọng trước khi lab vì giúp xác minh:

- account;
- identity;
- ARN.

Tức là trước khi tạo/xóa resource phải biết:

> Em đang thao tác bằng identity nào và account nào?

---

## Tagging

Ví dụ:

```text
Name=java-demo
Environment=lab
Owner=<team>
Project=L15
CostCenter=training
```

Tag quan trọng cho:

- ownership;
- inventory;
- cost allocation;
- automation;
- cleanup;
- operations;
- identifying environment.

Nếu thấy EC2 không có tag thì khó biết ai sở hữu và có được terminate hay không.

---

# Câu 3. VPC, public/private subnet, route table, IGW, Security Group và NACL

## VPC

VPC là logically isolated network của chúng ta trong AWS.

Trong VPC có các subnet.

---

## Subnet

Một subnet là một phần CIDR của VPC và thuộc một Availability Zone.

### Public subnet

Theo AWS networking model, một subnet được xem là public khi route table của nó có route đến Internet Gateway; ví dụ:

```text
0.0.0.0/0 → igw-...
```

AWS documentation cũng xác nhận public subnet có route đến IGW, còn private subnet không có direct route tới IGW.

Nhưng chỉ có public subnet chưa đủ để EC2 IPv4 trực tiếp giao tiếp Internet: instance còn cần public IPv4/EIP và security rules phù hợp.

### Private subnet

Không có direct route tới Internet Gateway.

Nếu private instance cần outbound Internet, một architecture phổ biến là đi qua NAT Gateway ở public subnet.

---

# Route table

Route table quyết định traffic đi đâu.

Mỗi route gồm:

```text
destination → target
```

Ví dụ:

```text
10.0.0.0/16 → local
0.0.0.0/0   → igw-123...
```

AWS quy định mỗi subnet phải liên kết với route table, trực tiếp hoặc thông qua main route table.

---

# Internet Gateway

IGW được attach vào VPC để cung cấp một thành phần kết nối Internet theo routing configuration phù hợp.

Không phải:

> Gắn IGW là tất cả EC2 tự động public.

Còn phụ thuộc:

- route table;
- public IP;
- Security Group;
- NACL;
- host firewall;
- listening service.

---

# Security Group vs Network ACL

## Security Group

- áp dụng ở network interface/resource level;
- allow rules;
- **stateful**.

Nếu inbound traffic được allow thì response traffic được state tracking xử lý phù hợp.

## Network ACL

- áp dụng ở subnet boundary;
- có allow và deny rules;
- **stateless**;
- inbound và outbound cần được xem xét độc lập.

AWS documentation xác nhận Security Groups là stateful còn NACL là stateless.

### Cách nhớ

```text
Security Group → resource/ENI → stateful
NACL           → subnet       → stateless
```

---

## Troubleshooting VPC bằng OSI Mindset

Ví dụ SSH EC2 fail.

Em kiểm tra:

1. EC2 Running không?
2. đúng IP không?
3. route table?
4. IGW?
5. public IP?
6. SG port 22?
7. NACL?
8. local firewall?
9. `ss -lntp` có sshd listen?
10. credentials/key đúng?

Không nên chỉ thấy timeout rồi tạo một SG rule `0.0.0.0/0`.

---

# Câu 4. EC2 lifecycle, user data, SSH và key pair

## Launch

Khi tạo EC2 cần xác định:

- AMI;
- instance type;
- subnet;
- Security Group;
- IAM role;
- storage;
- key/access mechanism;
- tags.

## Bootstrap bằng user data

User data thường được dùng để bootstrap instance, ví dụ:

- install package;
- create file;
- configure service;
- start application.

Một nguyên tắc quan trọng:

> User data không nên chứa plaintext secrets nếu có thể tránh.

Sau launch phải verify bootstrap chứ không mặc định user data chắc chắn chạy thành công.

---

## SSH access

Ví dụ:

```bash
ssh -i key.pem ubuntu@<host>
```

Phải đảm bảo:

- đúng username;
- private key permission phù hợp;
- SG cho phép SSH từ approved source;
- network route đúng;
- sshd đang listen.

---

## Key pair

AWS giữ public-key side/metadata cần thiết trong workflow; private key phải được người dùng bảo vệ.

Không upload private key lên Git.

Không gửi private key qua chat/email.

---

## EC2 lifecycle

Các trạng thái quan trọng về vận hành:

```text
launch
→ running
→ stop/start
→ terminate
```

`stop` và `terminate` khác hoàn toàn.

Terminate thường mang tính destructive đối với instance; data persistence phụ thuộc storage configuration.

L1.5 không được terminate resource production nếu chưa có approval.

---

# Câu 5. EBS: attach, mount, snapshot, restore và verify backup

Amazon EBS cung cấp block storage cho EC2. AWS mô tả EBS volumes có thể attach vào EC2 và snapshots là point-in-time backups tồn tại độc lập với volume.

## Flow sử dụng volume

### 1. Create EBS volume

Volume và EC2 phải có AZ compatibility phù hợp để attach.

### 2. Attach vào EC2

Sau attach, kiểm tra:

```bash
lsblk
```

### 3. Nếu volume mới thì tạo filesystem

Ví dụ lab:

```bash
sudo mkfs.ext4 /dev/...
```

**Cảnh báo:** `mkfs` là destructive nếu chạy nhầm volume đã chứa dữ liệu.

Phải kiểm tra `lsblk`, filesystem và device identity trước.

### 4. Mount

```bash
sudo mkdir -p /data
sudo mount /dev/... /data
```

Verify:

```bash
findmnt /data
df -h
```

Nếu cần persistent mount qua reboot thì phải cấu hình `/etc/fstab` cẩn thận, ưu tiên UUID và kiểm tra trước reboot.

---

# Snapshot

EBS snapshot là point-in-time backup.

AWS hiện mô tả EBS snapshots là incremental ở backend: sau snapshot đầu, các snapshot tiếp theo chỉ cần lưu block thay đổi cần thiết trong snapshot chain, trong khi mỗi snapshot vẫn có đủ thông tin logic để restore dữ liệu tại thời điểm đó.

Snapshot creation là asynchronous và có thể ở trạng thái `pending` trước khi complete.

---

# Restore

Flow:

```text
snapshot
→ create new EBS volume
→ attach EC2
→ identify device
→ mount
→ verify data
```

### Backup verification

Không được coi:

> Snapshot status = completed

là bằng chứng duy nhất rằng recovery thực tế chắc chắn đúng.

Tốt hơn phải restore thử.

Ví dụ trước backup:

```bash
sha256sum /data/test.txt
```

Sau restore:

```bash
sha256sum /restore/test.txt
```

Nếu hash, contents và expected structure phù hợp thì mới có evidence mạnh hơn về recovery.

> Backup chưa được test restore thì mức độ tin cậy operational còn hạn chế.

---

# Câu 6. CloudWatch alarm, Budget control và cost khi quên resource

## CloudWatch alarm

CloudWatch alarm theo dõi metric/expression và thay đổi state theo condition đã cấu hình.

Ví dụ:

```text
CPUUtilization > threshold
```

trong khoảng evaluation nhất định.

Alarm dùng để:

- phát hiện bất thường;
- cảnh báo operator;
- kích hoạt response/automation tùy configuration.

Không nên hiểu CloudWatch alarm là:

> CPU cao thì AWS tự sửa server.

Alarm là detection/control signal; remediation là bước khác.

---

# AWS Budgets

AWS Budgets dùng để theo dõi cost/usage so với budget và có thể gửi notification khi actual hoặc forecasted cost/usage vượt threshold đã định.

Một nuance vận hành rất quan trọng: budget data/notification **không phải real-time tuyệt đối**; AWS ghi nhận có độ trễ giữa việc phát sinh usage/charge và notification.

Vì vậy:

> Có Budget không có nghĩa em có thể quên cleanup resource.

---

# Nếu quên tắt EC2/public IPv4 thì sao?

Có thể tiếp tục phát sinh cost từ nhiều thành phần tùy architecture, ví dụ:

- compute của running EC2;
- EBS volumes;
- snapshots;
- public IPv4;
- NAT Gateway nếu dùng;
- data transfer;
- các service khác vẫn đang tồn tại.

Do đó sau lab phải thực hiện cost cleanup checklist.

Ví dụ:

```text
EC2 → stop/terminate theo yêu cầu lab
unused EBS → review/delete nếu được phép
snapshots → review retention
public IPv4/EIP → review
NAT Gateway → review
load balancer → review
```

Không được xóa resource nếu chưa xác định owner hoặc retention requirement.

---

# UNIT 4 — JAVA APPLICATION CONFIGURATION

## Thời lượng mục tiêu: khoảng 7 phút

Guide yêu cầu runtime architecture, externalized configuration, database secrets/TLS và safe restart/rollback.

---

# Câu 1. Java app trên Tomcat: JAR, WAR, Tomcat layout và systemd

## Java runtime

Java source code được compile thành bytecode, chạy trên JVM.

Trong syllabus, Java runtime là một thành phần của application stack chứ không chỉ “cài Java rồi chạy”.

Kiểm tra:

```bash
java -version
```

---

## JAR

JAR — Java Archive — thường đóng gói Java classes/resources.

Một số Java application có thể chạy trực tiếp:

```bash
java -jar app.jar
```

nếu được đóng gói phù hợp.

---

## WAR

WAR — Web Application Archive — là packaging truyền thống cho Java web application được deploy vào servlet container như Tomcat.

Tomcat có thể deploy WAR trong Host application base như `webapps`. Apache Tomcat documentation xác nhận WAR có thể được deploy thành web application context.

Ví dụ:

```text
myapp.war
```

có thể tương ứng context:

```text
/myapp
```

theo configuration/deployment rules.

---

# Tomcat layout

Các khái niệm quan trọng:

```text
bin/      scripts
conf/     configuration
logs/     logs
webapps/  deployed applications
work/     runtime/work files
temp/     temporary files
```

Cần phân biệt:

```text
CATALINA_HOME
CATALINA_BASE
```

Tomcat documentation dùng `$CATALINA_BASE` làm base directory cho nhiều đường dẫn runtime và cho phép tách instance-specific content khỏi installation.

---

# systemd integration

Trong Linux, Tomcat có thể được quản lý bằng systemd:

```bash
systemctl status tomcat
systemctl restart tomcat
```

Service definition thường khai báo:

- service user;
- Java path/environment;
- startup command;
- shutdown behavior;
- restart policy.

Không nên chạy Tomcat bằng root nếu không có lý do đặc biệt.

Nên dùng dedicated service account.

---

# Câu 2. Externalized configuration là gì và vì sao không hard-code?

Externalized configuration nghĩa là tách configuration thay đổi theo environment ra khỏi application binary/source code.

Ví dụ:

- database URL;
- environment/profile;
- application port;
- context path;
- log level;
- JVM memory;
- endpoint;
- secrets.

Có thể cung cấp thông qua:

- environment variables;
- config files;
- application profiles;
- runtime options;
- secret management mechanism.

---

## Environment variables

Ví dụ conceptual:

```bash
export DB_HOST=db.internal
export APP_ENV=prod
```

Không nên hard-code:

```text
jdbc:postgresql://192.0.2.10/proddb
password=SuperSecret...
```

trong source code.

### Lợi ích externalization

Cùng một artifact:

```text
app.war
```

có thể được deploy dev/test/prod với config khác nhau.

Điều này:

- giảm rebuild;
- giảm accidental secret exposure;
- giúp automation;
- làm rollback predictable;
- tách application code và environment state.

---

# Profiles

Ví dụ:

```text
dev
test
prod
```

Mỗi environment có config riêng.

Cần tránh chạy production bằng dev profile.

---

# Port và context path

Ví dụ:

```text
Tomcat HTTP connector: 8080
application context: /myapp
```

Endpoint:

```text
http://host:8080/myapp
```

Khi troubleshoot phải phân biệt:

```text
host
port
context path
application endpoint
```

---

# JVM memory options

Ví dụ:

```text
-Xms
-Xmx
```

- `-Xms`: initial heap;
- `-Xmx`: maximum heap.

Không được tăng `-Xmx` tùy ý.

Phải xem:

- host RAM;
- other processes;
- approved baseline;
- monitoring evidence.

Nếu đặt JVM heap quá cao có thể khiến host thiếu memory.

---

# Câu 3. Kết nối database an toàn: secrets và TLS

Application cần các thông tin như:

```text
DB host
port
database
username
password/credential
```

Không nên:

- commit password vào Git;
- ghi password vào screenshot;
- đưa password vào ticket;
- copy secrets vào history;
- chia sẻ secrets qua chat.

Nên sử dụng approved secret storage/injection mechanism của environment.

Database account phải theo **least privilege**.

Application không nên kết nối bằng PostgreSQL superuser nếu chỉ cần:

```text
SELECT
INSERT
UPDATE
DELETE
```

trên application schema.

---

# TLS

TLS cung cấp bảo vệ cho data in transit và giúp xác thực endpoint khi được cấu hình đúng.

Khi validation TLS có thể kiểm tra certificate/connection bằng công cụ như:

```bash
openssl s_client ...
```

nhưng phải hiểu trust chain, hostname verification và certificate validity theo configuration thực tế.

Không được giải quyết certificate error bằng cách tắt validation một cách tùy tiện trong production.

---

# Câu 4. Trước khi restart Tomcat production cần làm gì? Deploy lỗi rollback thế nào?

Đây là câu rất quan trọng.

## Trước restart

Em sẽ kiểm tra:

### 1. Có approval/change window không?

Không restart production vì “em nghĩ restart sẽ hết”.

### 2. Tình trạng hiện tại

```bash
systemctl status tomcat
```

### 3. Logs

```bash
journalctl -u tomcat
```

và application/Tomcat logs theo environment.

### 4. Capacity

```bash
df -h
free -h
```

### 5. Current application health

```bash
curl <health-endpoint>
```

### 6. Backup config/artifact

Phải biết phiên bản hiện tại là gì.

### 7. Rollback plan

Phải xác định rõ:

```text
Nếu release mới fail, artifact/config nào sẽ restore?
```

### 8. Impact

Có traffic không? Có maintenance window không? Có dependency/batch đang chạy không?

---

## Restart

Sau approval:

```bash
sudo systemctl restart tomcat
```

---

## Validate

Không được dừng ở:

```text
systemctl status = active
```

Phải kiểm tra cả:

```bash
ss -lntp
curl <health-endpoint>
journalctl -u tomcat
```

và business/application validation nếu runbook yêu cầu.

---

# Rollback nếu deploy lỗi

Flow tốt:

```text
Detect failure
→ preserve evidence
→ stop further change
→ assess rollback criteria
→ restore previous known-good artifact/config
→ restart/reload theo procedure
→ validate service
→ validate endpoint
→ inspect logs
→ update ticket/evidence
```

Không overwrite artifact cũ mà không giữ version.

Một rollback tốt phải đưa hệ thống về **known-good state**, chứ không phải chỉ “copy file cũ và hy vọng”.

---

# UNIT 5 — DATABASE & MONITORING FOUNDATIONS

## Thời lượng mục tiêu: khoảng 7 phút

Guide yêu cầu PostgreSQL least privilege/`pg_hba.conf`, backup/restore, session/lock/EXPLAIN, Prometheus/Grafana alerting và L1.5 escalation.

---

# Câu 1. PostgreSQL roles, least privilege và pg_hba.conf

PostgreSQL dùng role model.

Có thể tạo login role cho application, ví dụ về mặt khái niệm:

```sql
CREATE ROLE app_user LOGIN PASSWORD '...';
```

Trong thực tế password phải được quản lý an toàn, không hard-code trong script/history.

Sau đó chỉ grant quyền cần thiết.

Ví dụ:

```sql
GRANT CONNECT ON DATABASE appdb TO app_user;
```

và quyền schema/table phù hợp với application.

Không nên:

```sql
ALTER ROLE app_user SUPERUSER;
```

chỉ vì permission error.

Đó là phá vỡ least privilege.

---

# pg_hba.conf

`pg_hba.conf` kiểm soát client authentication theo các yếu tố như:

- connection type;
- database;
- user/role;
- client address;
- authentication method.

PostgreSQL documentation xác nhận address có thể được khai báo bằng host/IP/CIDR để match client source.

Ví dụ conceptual:

```text
host  appdb  app_user  10.0.2.0/24  scram-sha-256
```

Nó có nghĩa đại ý:

> Cho connection phù hợp từ network range đó tới database/user tương ứng với authentication method được chỉ định.

Không nên mở:

```text
0.0.0.0/0
```

một cách tùy tiện chỉ để connection “chạy được”.

---

# Câu 2. PostgreSQL backup/restore và health check

Logical backup thường dùng:

```bash
pg_dump
```

Ví dụ concept:

```bash
pg_dump -Fc appdb > appdb.dump
```

Restore custom-format thường dùng:

```bash
pg_restore
```

Tùy loại backup có thể sử dụng `psql` hoặc `pg_restore`.

Điểm quan trọng không phải thuộc một command duy nhất mà là hiểu:

```text
backup type → restore tool → validation
```

---

# Backup validation

Không coi file tồn tại là đủ.

Kiểm tra:

- command exit status;
- backup file tồn tại và size hợp lý;
- logs;
- restore test nếu có môi trường phù hợp;
- object/table/data sau restore.

Một backup chỉ đáng tin hơn khi đã chứng minh được restore.

---

# Database health check

Ví dụ:

```bash
pg_isready
```

và:

```sql
SELECT 1;
```

Có thể kiểm tra:

- DB process;
- port;
- connection;
- simple query;
- application DB access.

Phân biệt:

```text
PostgreSQL process running
```

không đồng nghĩa:

```text
application có thể login/query thành công.
```

---

# Câu 3. Sessions, locks và EXPLAIN

Khi slow query hoặc request bị treo, không nên restart PostgreSQL ngay.

## Session

Có thể xem hoạt động thông qua PostgreSQL system views, điển hình:

```sql
SELECT * FROM pg_stat_activity;
```

Ta quan tâm:

- session nào active;
- query nào đang chạy;
- thời gian chạy;
- wait state;
- client/user/database.

---

## Locks

Có thể kiểm tra `pg_locks` kết hợp session information.

Mục tiêu:

```text
Ai đang chờ?
Ai đang giữ lock?
Resource nào bị lock?
```

Không nên lập tức terminate session.

Phải hiểu:

- session đó là gì;
- transaction nào;
- business impact;
- killing session có rollback transaction lớn không.

---

# EXPLAIN

Ví dụ:

```sql
EXPLAIN
SELECT ...
```

Nó cho execution plan mà optimizer dự kiến.

Ta có thể thấy:

- sequential scan;
- index scan;
- join strategies;
- estimated rows;
- cost.

`EXPLAIN ANALYZE` thực sự execute query để đo runtime, vì vậy trong production phải cẩn trọng, đặc biệt với expensive hoặc data-changing statements.

**Supplementary / Beyond explicit syllabus:** L1.5 nên chủ yếu thu thập plan/evidence; quyết định index hoặc rewrite complex SQL thường cần DBA/developer/L2 review.

---

# Câu 4. Prometheus, node_exporter, Grafana và alert rule

## node_exporter

`node_exporter` expose host metrics, ví dụ:

- CPU;
- memory;
- filesystem;
- network.

Default/common endpoint:

```text
:9100/metrics
```

Prometheus official guide minh họa scraping node_exporter trên port `9100`.

---

# Prometheus

Prometheus định kỳ scrape metrics target.

Concept:

```text
node_exporter
      ↓
metrics endpoint
      ↓
Prometheus
      ↓
time-series data
      ↓
PromQL / alerts / Grafana
```

Ví dụ `prometheus.yml` concept:

```yaml
scrape_configs:
  - job_name: node
    static_configs:
      - targets:
          - localhost:9100
```

Prometheus documentation dùng cấu trúc tương tự trong node_exporter guide.

---

# Grafana

Grafana dùng data source như Prometheus để query và visualize metrics trên dashboard.

Ví dụ dashboard:

- CPU utilization;
- memory usage;
- disk usage;
- network traffic;
- service/application signals.

---

# Alert

Alert rule gồm:

```text
signal/query
→ condition
→ evaluation
→ alert state
→ notification
```

Ví dụ concept:

```text
disk usage > 90% trong một khoảng thời gian
```

Grafana documentation mô tả alert evaluation theo chu kỳ, sau đó alert có thể fire và được gửi tới configured contact point/notification route.

Quan trọng:

> Dashboard có graph không chứng minh alerting hoạt động.

Phải failure-inject/test condition và có notification evidence.

---

# Câu 5. Khi nào L1.5 xử lý DB/query, khi nào escalate L2?

## L1.5 có thể xử lý khi

- lỗi nằm trong known runbook;
- credential/config sai rõ ràng và được phép sửa;
- service down có recovery procedure;
- connectivity bị config sai trong scope;
- disk capacity issue có approved cleanup;
- restart/retry được runbook cho phép;
- thu thập session/log/lock evidence;
- restore lab hoặc approved procedure.

## Phải escalate khi

- nghi data corruption;
- cần schema change;
- cần index/query optimization phức tạp;
- unknown lock chain/business transaction;
- long-running transaction cần kill nhưng impact không rõ;
- replication/recovery issue vượt runbook;
- permission cần superuser escalation;
- production data loss;
- restore decision có business impact;
- root cause chưa rõ và safe remediation không tồn tại.

Khi escalate không được chỉ nói:

> Database lỗi, nhờ L2 kiểm tra.

Phải gửi evidence:

```text
Symptom
Start time
Scope
Recent changes
DB/service state
Connectivity result
Relevant logs
Sessions/locks
Query/EXPLAIN if safe
Actions already performed
Current impact
```

---

# UNIT 6 — OPERATIONS CONTROLS & INTEGRATED PRACTICE

## Thời lượng mục tiêu: khoảng 4 phút

Unit cuối yêu cầu batch/release flow, integrated architecture/security/cost và OSI troubleshooting.

---

# Câu 1. Batch scheduling, dependency, exit-code và release/change flow

Một batch job không chỉ là script chạy theo giờ.

Flow em cần kiểm soát:

```text
Schedule
→ prerequisites
→ dependencies
→ execute
→ exit code
→ validate output
→ evidence
→ downstream decision
```

Ví dụ Job B phụ thuộc Job A.

Không được chạy B chỉ vì “đến giờ”.

Phải kiểm tra Job A:

```text
completed?
exit code = success?
output/data ready?
```

Nếu dependency fail thì phải stop chain hoặc xử lý đúng runbook.

---

# Change/release flow

Một change đúng chuẩn:

```text
Ticket
→ scope
→ risk/impact
→ implementation plan
→ validation plan
→ rollback plan
→ approval gate
→ execution
→ evidence
→ validation
→ closure
→ handover
```

Approval phải xảy ra **trước** action yêu cầu approval.

Không được:

```text
thay đổi trước → tạo ticket sau.
```

Rollback cũng phải chuẩn bị trước khi deployment.

---

# Câu 2. Xây dựng integrated Linux + AWS + Java + PostgreSQL + Monitoring và security/cost checklist

Em nhìn hệ thống như một flow end-to-end:

```text
Client
  ↓
DNS/network
  ↓
AWS VPC / route / SG / NACL
  ↓
EC2 Linux
  ↓
Tomcat / JVM
  ↓
Java application
  ↓
PostgreSQL
```

Monitoring side:

```text
Linux/node_exporter
        ↓
    Prometheus
        ↓
     Grafana
        ↓
   Alert/notification
```

Mỗi layer phải có configuration baseline.

Ví dụ:

### AWS

- correct VPC/subnet;
- minimum SG access;
- IAM least privilege;
- tags.

### Linux

- controlled users;
- SSH key;
- permissions;
- service account;
- disk/capacity.

### Java

- approved Java;
- Tomcat service;
- externalized config;
- JVM baseline.

### PostgreSQL

- least-privilege role;
- network restriction;
- backup.

### Monitoring

- scrape targets;
- dashboards;
- alert rule;
- notification test.

---

# Security checklist

Em sẽ hỏi:

```text
Có secret nào hard-code không?
Có port nào mở 0.0.0.0/0 không cần thiết không?
Application có chạy root không?
DB user có quyền quá cao không?
IAM có wildcard permission không?
Backup có được bảo vệ không?
```

---

# Cost checklist

Em sẽ kiểm tra:

```text
EC2 nào đang running?
EBS nào unused?
Snapshot nào cần giữ?
Public IPv4 nào đang sử dụng?
NAT Gateway/load balancer/resource nào còn chạy?
Resource có tag owner/environment không?
```

Sau lab cleanup theo approved scope.

---

# Câu 3. Gặp service/port/permission/disk/DB/alert/deployment lỗi thì áp dụng OSI Mindset như thế nào?

Đây là câu kết nối toàn khóa.

Nguyên tắc của em là:

> Không đoán root cause. Không restart ngẫu nhiên. Em xác định symptom, chọn hướng kiểm tra, thu thập evidence và dừng ở layer tìm thấy failure.

---

## Bottom-up

Dùng khi nghi infrastructure/network.

Ví dụ EC2 không SSH được:

```text
Layer 1 → EC2 Running?
Layer 2/3 → IP/network/routing?
Layer 3/4 → SG/NACL?
Layer 4 → port 22?
Layer 7 → sshd?
```

---

## Top-down

Dùng khi user báo application error rõ ràng.

Ví dụ:

```text
GET /app trả 500
```

Em bắt đầu:

```text
Application log
↓
Tomcat
↓
DB dependency
↓
port/network
↓
infrastructure nếu cần
```

---

# Một ví dụ tổng hợp

User báo:

> Java application không truy cập được.

Em xác định expected endpoint trước.

Sau đó:

```bash
curl -v http://localhost:8080/app
```

Nếu connection refused:

```bash
ss -lntp | grep 8080
```

Nếu không listen:

```bash
systemctl status tomcat
journalctl -u tomcat
```

Nếu log cho thấy:

```text
database connection refused
```

thì chuyển xuống dependency:

```text
PostgreSQL running?
5432 listening?
network reachable?
pg_hba.conf phù hợp?
credential đúng?
```

Nếu vượt scope, preserve evidence và escalate.
