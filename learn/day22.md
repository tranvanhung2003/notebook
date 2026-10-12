# Day 22 — Final Test: Tổng ôn + Final Practical Exam tích hợp

Tôi đã đối chiếu trực tiếp MASTER_SYLLABUS. **Day 22 là Final Test**, không bổ sung một technology domain mới. Phần theory kiểm tra Linux, AWS, Java application configuration, PostgreSQL, monitoring, change controls và escalation; phần practical yêu cầu bạn **deploy, configure, validate, troubleshoot, document và hand over** một integrated scenario. 00_MASTER_SYLLABUS_Java_Fresher…

Vì hiện tại chúng ta đang ở **LEARN MODE**, tôi sẽ dạy Day 22 theo kiểu “final consolidation”: vừa hệ thống hóa lý thuyết, vừa dựng một bài practical đủ sâu để bạn luyện như kỳ thi cuối khóa. Đây cũng là lúc nối tất cả kiến thức Day 1–21 thành một mental model thống nhất.

---

## 1. Syllabus Alignment

### Mandatory — đúng phạm vi Day 22

| Final theory                                | Final practical                   |
| ------------------------------------------- | --------------------------------- |
| Linux administration                        | Deploy                            |
| AWS fundamentals & operations               | Configure                         |
| Java/JVM/Tomcat configuration               | Validate                          |
| PostgreSQL administration & diagnostics     | Troubleshoot                      |
| Prometheus/node_exporter/Grafana monitoring | Document                          |
| Batch/change/release controls               | Hand over                         |
| L1 troubleshooting & escalation             | Làm việc trên integrated scenario |

Điều quan trọng nhất là Day 22 **không kiểm tra từng công nghệ như những hòn đảo riêng biệt**. Bạn phải suy luận được một request đi qua các layer:

```text
User / Client
     |
     v
DNS / Public IP
     |
     v
AWS VPC
Route Table
Internet Gateway
Security Group / NACL
     |
     v
EC2 Network Interface
     |
     v
Linux
Firewall / Port / Socket
     |
     v
systemd
     |
     v
Tomcat / JVM
     |
     v
Java Application
Configuration / Secrets
     |
     v
PostgreSQL
Authentication / SQL / Lock / Capacity
     |
     v
Application Response
     |
     +------> Logs
     |
     +------> Prometheus
                 |
                 v
              Grafana
                 |
                 v
               Alert
```

Một fresher vận hành tốt không nhảy ngay vào restart service. Bạn phải tìm **failing layer** bằng evidence.

---

# 2. Environment assumptions và technical truth hiện tại

Các command Linux bên dưới chủ yếu giả định:

```text
OS: Ubuntu Server 24.04 LTS hoặc Debian-like
Init system: systemd
Java: OpenJDK 17 hoặc OpenJDK 25
Tomcat: Tomcat 10.1.x khi dùng ví dụ hiện đại
Database: PostgreSQL
Monitoring: Prometheus + node_exporter + Grafana
Cloud: AWS EC2/VPC/EBS/CloudWatch
```

Với RHEL/Rocky/AlmaLinux, package manager và một số đường dẫn sẽ khác:

| Ubuntu/Debian                     | RHEL/Rocky/Alma                  |
| --------------------------------- | -------------------------------- |
| `apt`                             | `dnf`                            |
| `www-data` thường gặp             | `apache` thường gặp              |
| `/etc/postgresql/<version>/main/` | thường dưới `/var/lib/pgsql/...` |
| `ufw` có thể dùng                 | `firewalld` phổ biến             |
| `journalctl`                      | `journalctl`                     |

Không bao giờ học thuộc path PostgreSQL mà không xác minh:

```bash
sudo -u postgres psql -Atc 'SHOW config_file;'
sudo -u postgres psql -Atc 'SHOW hba_file;'
sudo -u postgres psql -Atc 'SHOW data_directory;'
```

### Current documentation note

Tại thời điểm hiện tại, Apache Tomcat 10.1.60 yêu cầu Java 11 trở lên, do đó Java 17 và Java 25 nằm trong phạm vi Java version mà Tomcat 10.1 có thể chạy. Tuy nhiên, **Tomcat chạy được trên JDK đó không đồng nghĩa application chắc chắn tương thích**; framework, library và bytecode của application vẫn phải được kiểm tra. Tomcat 10.1 triển khai Jakarta Servlet 6.0, một chi tiết đặc biệt quan trọng khi troubleshooting application cũ dùng namespace `javax.servlet.*`. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/appdev/installation.html?utm_source=chatgpt.com)

PostgreSQL documentation hiện tại là dòng PostgreSQL 18; tài liệu chính thức tại thời điểm kiểm tra ghi PostgreSQL 18.6 là current và 17/16/15/14 vẫn được hỗ trợ. **Syllabus không yêu cầu PostgreSQL 18**, vì vậy exam/lab phải dùng version được build sheet hoặc môi trường cung cấp, không tự ý upgrade chỉ vì đó là version mới nhất. [PostgreSQL](https://www.postgresql.org/docs/current/index.html?utm_source=chatgpt.com)

AWS hiện tiếp tục khuyến nghị temporary credentials/IAM roles, MFA và least privilege thay vì cấp quyền rộng hoặc dựa lâu dài vào access keys. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

# 3. Mental Model quan trọng nhất của Day 22

Bạn cần phân biệt bốn khái niệm:

| Khái niệm     | Câu hỏi cần trả lời                                       |
| ------------- | --------------------------------------------------------- |
| Configuration | Hệ thống **được thiết lập như thế nào?**                  |
| State         | Hệ thống **đang ở trạng thái nào?**                       |
| Evidence      | Bạn **chứng minh** trạng thái đó bằng gì?                 |
| Change        | Bạn đã **thay đổi** cái gì, với approval và rollback nào? |

Ví dụ:

```bash
systemctl is-enabled tomcat
```

trả lời startup configuration.

```bash
systemctl is-active tomcat
```

trả lời runtime state.

```bash
systemctl status tomcat --no-pager
journalctl -u tomcat --since "-10 min"
```

là evidence.

Còn:

```bash
sudo systemctl restart tomcat
```

là một **change**.

Đây chính là khác biệt giữa người “biết command” và người có tư duy operations.

---

# 4. FINAL THEORY — Linux

## 4.1 Linux architecture

Mental model cơ bản:

```text
Hardware
   ↓
Linux Kernel
   ↓
System Calls
   ↓
Processes / Services
   ↓
Libraries / Runtime
   ↓
Applications
   ↓
Users / Shell
```

Kernel quản lý CPU scheduling, memory, filesystems, devices, networking và process isolation.

Shell như Bash không phải kernel. `systemd` cũng không phải kernel. Tomcat cũng không phải OS service “tự nhiên”; Tomcat chỉ trở thành managed service khi được tích hợp vào systemd hoặc init mechanism.

---

## 4.2 Filesystem

Các path phải nhận diện:

| Path       | Ý nghĩa                           |
| ---------- | --------------------------------- |
| `/etc`     | system/application configuration  |
| `/var/log` | traditional logs                  |
| `/var/lib` | persistent application state      |
| `/opt`     | optional/vendor software          |
| `/usr`     | installed programs/libraries      |
| `/home`    | normal user home                  |
| `/root`    | root home                         |
| `/tmp`     | temporary files                   |
| `/proc`    | kernel/process virtual filesystem |
| `/sys`     | kernel/device virtual filesystem  |
| `/run`     | runtime state                     |

Các command nền tảng:

```bash
pwd
ls -lah
cd /path
find /var/log -type f -name '*.log'
stat /path/to/file
file /path/to/file
du -sh /var/log/*
df -hT
```

Phải hiểu:

```bash
df
```

đo filesystem capacity.

```bash
du
```

đo disk blocks mà file/directory đang sử dụng.

Nếu `df` báo full nhưng `du` không tìm thấy tương ứng, nghĩ đến **deleted-but-open files**:

```bash
sudo lsof +L1
```

Đây là tình huống operations rất thực tế.

---

# 5. Users, groups, permissions và least privilege

Linux permission:

```text
r = read
w = write
x = execute / traverse
```

Ví dụ:

```text
-rw-r----- 1 tomcat appops application.properties
```

nghĩa là:

```text
owner tomcat  = rw-
group appops  = r--
others        = ---
```

Octal:

```text
r = 4
w = 2
x = 1

7 = rwx
6 = rw-
5 = r-x
4 = r--
```

Command:

```bash
id
groups
getent passwd tomcat
getent group appops

ls -ld /opt/tomcat
namei -l /opt/tomcat/webapps/myapp.war

sudo chown tomcat:tomcat file
sudo chmod 640 file
```

Một lỗi rất phổ biến:

```text
file = 644
```

nhưng directory cha là:

```text
700 root root
```

Tomcat vẫn không thể traverse.

Dùng:

```bash
namei -l /path/to/file
```

để kiểm tra permission trên **toàn bộ path**.

---

# 6. sudo và privilege boundaries

Không làm:

```bash
sudo su -
```

rồi thực hiện hàng chục command không lưu dấu vết.

Ưu tiên:

```bash
sudo systemctl restart tomcat
sudo journalctl -u tomcat
```

Phải biết:

```bash
sudo -l
```

để xác minh quyền được delegated.

Operations principle:

```text
Need root?
     |
    No ------> Không dùng sudo
     |
    Yes
     |
Có approved command/runbook?
     |
    Yes ------> sudo specific-command
     |
    No -------> Escalate / obtain approval
```

---

# 7. SSH

Permission private key thường cần chặt:

```bash
chmod 400 lab-key.pem
```

Kết nối:

```bash
ssh -i lab-key.pem ubuntu@SERVER_IP
```

Không bao giờ:

```text
paste private key
email private key
commit private key to Git
store private key inside application directory
```

Troubleshooting SSH phải đi theo layer:

```text
DNS/IP
 → route
 → AWS Security Group
 → OS firewall
 → sshd listening?
 → user?
 → key?
 → ~/.ssh permission?
 → sshd logs?
```

Kiểm tra:

```bash
ss -ltnp | grep ':22'
systemctl status ssh
journalctl -u ssh --since "-10 min"
```

Tên service có thể là `ssh` hoặc `sshd` tùy distribution.

---

# 8. Processes và signals

Command:

```bash
ps aux
ps -ef
pgrep -af java
top
free -h
```

Signals quan trọng:

```text
SIGTERM = yêu cầu process shutdown graceful
SIGKILL = kernel terminate process ngay
SIGHUP  = thường được service dùng để reload, tùy application
```

Ưu tiên:

```bash
kill PID
```

tức SIGTERM.

Chỉ dùng:

```bash
kill -9 PID
```

khi thực sự cần thiết.

`SIGKILL` không cho application cleanup, flush state hoặc graceful shutdown.

---

# 9. systemd

Một service có ba trạng thái khác nhau cần phân biệt:

```bash
systemctl is-enabled tomcat
systemctl is-active tomcat
systemctl status tomcat
```

`enabled` ≠ `running`.

Service có thể:

```text
enabled + inactive
disabled + active
enabled + failed
```

Command quan trọng:

```bash
sudo systemctl start tomcat
sudo systemctl stop tomcat
sudo systemctl restart tomcat
sudo systemctl reload tomcat
sudo systemctl enable tomcat

systemctl cat tomcat
systemctl show tomcat
systemctl list-dependencies tomcat
```

Sau khi chỉnh unit:

```bash
sudo systemctl daemon-reload
```

Không phải:

```bash
systemctl daemon-reload
```

là restart service. Hai operation hoàn toàn khác nhau.

---

# 10. Logs và journalctl

Câu hỏi đầu tiên sau khi service fail:

```bash
systemctl status tomcat --no-pager
```

Sau đó:

```bash
journalctl -u tomcat -n 100 --no-pager
journalctl -u tomcat --since "-10 min"
journalctl -u tomcat -p warning
```

Theo dõi realtime:

```bash
journalctl -fu tomcat
```

Text filtering:

```bash
grep -i error app.log
grep -Ein 'error|exception|failed|denied' app.log
```

`awk`:

```bash
awk '{print $1, $9}' access.log
```

`sed`:

```bash
sed -n '100,150p' app.log
```

Nhưng không được grep từ khóa rồi kết luận root cause. Cần đọc context:

```bash
grep -n -B 5 -A 10 'ERROR' app.log
```

---

# 11. Networking

Phải phân biệt:

```text
IP address
DNS name
Port
Protocol
Socket
Listening socket
Connection
```

Kiểm tra local listener:

```bash
ss -ltnp
```

Tomcat:

```bash
ss -ltnp | grep ':8080'
```

PostgreSQL:

```bash
ss -ltnp | grep ':5432'
```

Prometheus:

```bash
ss -ltnp | grep ':9090'
```

node_exporter:

```bash
ss -ltnp | grep ':9100'
```

Grafana:

```bash
ss -ltnp | grep ':3000'
```

HTTP validation:

```bash
curl -v http://127.0.0.1:8080/
curl -I http://127.0.0.1:8080/
```

DNS:

```bash
getent hosts example.com
```

Connectivity:

```bash
ping HOST
traceroute HOST
```

Nhưng nhớ rằng ping sử dụng ICMP. Một host không trả lời ping **không chứng minh TCP service down**.

Test TCP:

```bash
nc -vz HOST 5432
```

nếu `nc` đã được cài đặt.

---

# 12. Disk, filesystem và mounts

Kiểm tra:

```bash
lsblk -f
df -hT
findmnt
mount
```

Nếu thêm EBS volume, **không bao giờ lập tức chạy `mkfs` dựa trên device name bạn đoán**.

Trước tiên:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,UUID
sudo file -s /dev/DEVICE
```

`mkfs` là destructive.

Sau khi chắc chắn volume mới và trống:

```bash
sudo mkfs.ext4 /dev/DEVICE
```

Mount:

```bash
sudo mkdir -p /data
sudo mount /dev/DEVICE /data
df -hT /data
```

Trước sửa `/etc/fstab`:

```bash
sudo cp -a /etc/fstab /etc/fstab.before-day22
```

Tìm UUID:

```bash
sudo blkid /dev/DEVICE
```

Sau khi chỉnh:

```bash
sudo mount -a
findmnt /data
```

Nếu `mount -a` báo lỗi, sửa trước reboot.

---

# 13. Bash, exit code, cron, backup

Mọi command trả exit status:

```bash
command
echo $?
```

Theo convention:

```text
0     = success
non-0 = failure
```

Script operations nên dùng:

```bash
#!/usr/bin/env bash
set -euo pipefail
```

Nhưng phải hiểu `set -e` có edge cases; không xem nó như substitute cho error handling.

Ví dụ:

```bash
if curl -fsS http://127.0.0.1:8080/health >/dev/null; then
    echo "Application OK"
else
    echo "Application FAILED" >&2
    exit 1
fi
```

Cron phải quan tâm:

```text
PATH
working directory
permissions
environment variables
logging
exit code
lock/concurrency
```

Backup:

```bash
tar -czf backup.tar.gz /path
rsync -a source/ destination/
```

`rsync --delete` có blast radius lớn. Không dùng nếu chưa hiểu direction và chưa dry-run.

---

# 14. FINAL THEORY — AWS

## Shared Responsibility Model

AWS chịu trách nhiệm **security of the cloud**.

Customer chịu trách nhiệm **security in the cloud**, mức độ cụ thể tùy service.

Với EC2, bạn vẫn chịu trách nhiệm lớn về:

```text
guest OS patching
users
SSH access
application configuration
firewall/security groups
data
secrets
Tomcat
Java
PostgreSQL
monitoring
```

---

# 15. IAM

Phải phân biệt:

| Thành phần | Ý nghĩa                                   |
| ---------- | ----------------------------------------- |
| IAM User   | long-lived identity trong một số use case |
| IAM Role   | assumable identity, temporary credentials |
| Policy     | tập permission                            |
| MFA        | second factor                             |
| Access Key | programmatic credential                   |
| Root user  | account-level privileged identity         |

Current AWS guidance ưu tiên federated access/roles và temporary credentials, MFA và least privilege. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

Một policy phải suy nghĩ theo:

```text
Effect
Action
Resource
Condition
```

Principle:

```text
Allow only what is required
on only required resources
under required conditions
for required duration.
```

Không dùng:

```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```

chỉ vì “cho dễ làm lab”.

---

# 16. VPC networking

Mental model:

```text
VPC
 ├── Subnet
 │     ├── EC2
 │     └── ENI
 │
 ├── Route Table
 ├── Internet Gateway
 ├── Security Groups
 └── Network ACL
```

Một **public subnet** về mặt routing là subnet có route phù hợp đến Internet Gateway. Nhưng EC2 muốn trực tiếp giao tiếp Internet còn cần public IPv4/EIP và network/security configuration phù hợp.

Ví dụ route:

```text
Destination      Target
10.22.0.0/16     local
0.0.0.0/0        igw-xxxxxxxx
```

Security Group là **stateful**. Response traffic thuộc connection được theo dõi có thể được cho phép mà không cần mirror rule theo cách NACL hoạt động. AWS documentation hiện xác nhận security groups dùng connection tracking và stateful filtering. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-connection-tracking.html?utm_source=chatgpt.com)

NACL là subnet-level và stateless; phải suy nghĩ cả direction và ephemeral ports.

---

# 17. Security Group design

Không nên:

```text
SSH 22     0.0.0.0/0
Postgres   5432 0.0.0.0/0
Grafana    3000 0.0.0.0/0
Tomcat     8080 0.0.0.0/0
```

AWS cũng khuyến nghị không unrestricted ingress với các administrative/database/application ports như 22, 5432, 3000, 8080. [AWS Documentation](https://docs.aws.amazon.com/controltower/latest/controlreference/ec2-rules.html?utm_source=chatgpt.com)

Cho lab:

```text
22/tcp     YOUR_PUBLIC_IP/32
8080/tcp   YOUR_PUBLIC_IP/32   only if needed
3000/tcp   YOUR_PUBLIC_IP/32   only if needed
5432       no public rule
9100       no public rule
9090       preferably local/private only
```

Tốt hơn nữa dùng SSH tunnel thay vì public monitoring ports.

---

# 18. EC2

Lifecycle:

```text
launch
 ↓
pending
 ↓
running
 ↓
stop/start
 ↓
terminate
```

`stop` khác `terminate`.

Stop thường giữ EBS-backed root volume nhưng compute ngừng; terminate thường xóa instance và volume có `DeleteOnTermination=true`.

Không học thuộc assumption. Kiểm tra resource configuration.

Metadata nên được tiếp cận với IMDSv2.

**Supplementary / Beyond the explicit syllabus:** Ví dụ an toàn để xem instance ID:

```bash
TOKEN=$(curl -fsS -X PUT \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300" \
  http://169.254.169.254/latest/api/token)

curl -fsS \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

Không lấy IAM credential từ metadata để paste vào evidence.

---

# 19. Public IPv4 và cost

AWS hiện tính phí public IPv4 address; tài liệu EC2 cũng ghi rõ public IPv4 gắn với running instance và Elastic IP đều có charge. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html?utm_source=chatgpt.com)

Do đó lab Day 22 phải nghĩ:

```text
Do I need this public IPv4?
Do I need this instance running?
Do I need this unattached EBS?
Do I need this snapshot?
Do I need this NAT Gateway?
```

AWS cleanup là một phần operations, không phải việc phụ.

---

# 20. EBS

EBS là block storage.

Flow:

```text
create/attach volume
       ↓
OS discovers block device
       ↓
filesystem
       ↓
mount
       ↓
application data
```

Snapshot:

```text
EBS Volume
    |
    v
EBS Snapshot
    |
    v
New EBS Volume
    |
    v
attach
    |
    v
mount
    |
    v
validate data
```

Một snapshot thành công chưa đủ. Phải **restore test**.

Đối với database, filesystem snapshot không tự động đồng nghĩa application-consistent backup. PostgreSQL-native backup/restore vẫn là kỹ năng quan trọng.

---

# 21. CloudWatch, CloudTrail, Budgets

Không nhầm:

| Service            | Dùng để                           |
| ------------------ | --------------------------------- |
| CloudWatch Metrics | numerical telemetry               |
| CloudWatch Logs    | centralized logs                  |
| CloudWatch Alarms  | evaluate metrics / trigger action |
| CloudTrail         | AWS API activity/audit            |
| AWS Budgets        | cost/budget notification          |

EC2 default CloudWatch metrics có CPU nhưng **host memory và filesystem usage không phải default EC2 metric**. Muốn memory/disk OS metrics thường cần CloudWatch Agent hoặc monitoring stack khác.

Do course dùng Prometheus/node_exporter, đây là lý do node_exporter hữu ích.

---

# 22. FINAL THEORY — Java, JVM và Tomcat

Mental model:

```text
Linux
  ↓
JDK / JVM
  ↓
Tomcat
  ↓
WAR
  ↓
Java Web Application
  ↓
External Configuration
  ↓
PostgreSQL
```

Hoặc executable application:

```text
Linux
  ↓
JDK / JVM
  ↓
java -jar application.jar
```

---

# 23. JAR vs WAR

JAR:

```text
Java ARchive
```

Có thể là library hoặc executable application.

Kiểm tra:

```bash
jar tf application.jar | head
```

WAR:

```text
Web Application ARchive
```

thường deploy vào servlet container như Tomcat.

```bash
jar tf application.war | head
```

Tomcat deployment thường liên quan:

```text
CATALINA_HOME
CATALINA_BASE
conf/
logs/
webapps/
work/
temp/
bin/
```

`CATALINA_HOME` là Tomcat installation.

`CATALINA_BASE` cho phép tách instance-specific configuration/runtime state.

---

# 24. Java version

Kiểm tra:

```bash
java -version
javac -version
readlink -f "$(command -v java)"
```

Tìm `JAVA_HOME` trên nhiều Linux environment:

```bash
dirname "$(dirname "$(readlink -f "$(command -v java)")")"
```

Ví dụ output có thể là:

```text
/usr/lib/jvm/java-17-openjdk-amd64
```

Nhưng đừng copy path đó vào mọi server.

---

# 25. Tomcat service account

Không chạy Tomcat bằng root.

Ví dụ service user:

```bash
sudo useradd \
  --system \
  --home /opt/tomcat \
  --shell /usr/sbin/nologin \
  tomcat
```

Mục tiêu:

```text
Tomcat compromised
       ↓
Attacker gets tomcat privileges
NOT
root privileges
```

Đây chính là blast-radius control.

---

# 26. systemd integration cho Tomcat

Ví dụ lab:

```ini
[Unit]
Description=Apache Tomcat
After=network.target

[Service]
Type=simple
User=tomcat
Group=tomcat
EnvironmentFile=/etc/tomcat/tomcat.env
ExecStart=/opt/tomcat/bin/catalina.sh run
Restart=on-failure
RestartSec=5
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

Ví dụ environment file:

```bash
JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
CATALINA_HOME=/opt/tomcat
CATALINA_BASE=/opt/tomcat
CATALINA_OPTS=-Xms256m -Xmx512m
```

Đây là example. `JAVA_HOME`, heap sizing và paths phải được xác minh cho environment thật.

Sau thay đổi:

```bash
sudo systemctl daemon-reload
sudo systemctl start tomcat
systemctl status tomcat --no-pager
```

Validation:

```bash
pgrep -af java
ss -ltnp | grep ':8080'
curl -fsS http://127.0.0.1:8080/
```

Ba evidence này kiểm tra ba layer khác nhau:

```text
process
socket
application protocol
```

---

# 27. JVM memory

Quan trọng nhất:

```text
-Xms = initial heap
-Xmx = maximum heap
```

Ví dụ:

```text
-Xms256m
-Xmx512m
```

`-Xmx512m` **không đồng nghĩa Java process chỉ sử dụng tối đa 512 MB RAM**.

Process còn sử dụng:

```text
Metaspace
thread stacks
native libraries
direct buffers
JIT/code cache
JVM structures
```

Do đó:

```text
RSS > Xmx
```

không nhất thiết là memory leak.

Không được thấy OOM rồi tăng `-Xmx` tùy tiện. Trước tiên:

```bash
free -h
ps -o pid,rss,vsz,cmd -p PID
journalctl -k | grep -i oom
journalctl -u tomcat
```

---

# 28. Externalized configuration

Không hard-code:

```text
DB_PASSWORD
API_TOKEN
environment-specific URLs
production hostname
```

vào source hoặc WAR.

Có thể dùng:

```text
environment variables
external properties files
secret management system
systemd EnvironmentFile
```

Nhưng environment variables cũng không phải magical secret vault.

Permission file:

```bash
sudo chown root:tomcat /etc/myapp/app.env
sudo chmod 640 /etc/myapp/app.env
```

Kiểm tra:

```bash
sudo -u tomcat test -r /etc/myapp/app.env
echo $?
```

Đừng chạy:

```bash
cat /etc/myapp/app.env
```

rồi copy secret vào evidence.

---

# 29. Profiles, ports và context path

Environment profile:

```text
dev
test
uat
prod
```

Mục đích là tách behavior/config giữa environment.

Context path:

```text
myapp.war
```

thường có thể dẫn đến:

```text
http://host:8080/myapp/
```

còn:

```text
ROOT.war
```

thường:

```text
http://host:8080/
```

Nhưng application/container configuration có thể override.

Không đoán endpoint; kiểm tra deployment configuration.

---

# 30. Safe restart

Safe restart không phải:

```bash
restart
```

rồi “chắc là xong”.

Flow đúng:

```text
Confirm approved change
      ↓
Capture baseline
      ↓
Validate configuration
      ↓
Confirm dependencies
      ↓
Restart
      ↓
Check service
      ↓
Check port
      ↓
Check logs
      ↓
Check endpoint
      ↓
Check DB dependency
      ↓
Check monitoring
```

Ví dụ:

```bash
systemctl status tomcat --no-pager
curl -fsS http://127.0.0.1:8080/health

sudo systemctl restart tomcat

systemctl is-active tomcat
journalctl -u tomcat --since "-2 min"
ss -ltnp | grep ':8080'
curl -fsS http://127.0.0.1:8080/health
```

---

# 31. Deployment và rollback

WAR release:

```text
release-42.war
       ↓
pre-check
       ↓
backup/current artifact reference
       ↓
deploy
       ↓
restart/reload
       ↓
smoke test
       ↓
logs
       ↓
monitoring
```

Nếu validation fail:

```text
stop further change
       ↓
preserve evidence
       ↓
restore previous known-good artifact
       ↓
restart safely
       ↓
validate
       ↓
document rollback
```

Không debug production hàng giờ trên một release đang gây outage nếu rollback là safe approved path.

---

# 32. TLS và reverse proxy

TLS cung cấp:

```text
confidentiality
integrity
server authentication
```

Certificate không phải encryption key đơn giản; bạn thường có:

```text
private key
certificate
certificate chain
trusted CA
```

Private key phải được bảo vệ.

Kiểm tra certificate:

```bash
openssl x509 -in server.crt -noout -subject -issuer -dates
```

Kết nối:

```bash
openssl s_client -connect host:443 -servername host
```

Reverse proxy architecture:

```text
Client
  ↓ HTTPS :443
Reverse Proxy
  ↓ HTTP/private network
Tomcat :8080
```

Tomcat 8080 không nhất thiết phải public.

---

# 33. FINAL THEORY — PostgreSQL

Mental model:

```text
PostgreSQL Server / Cluster
        |
        +--- Database
               |
               +--- Schema
                      |
                      +--- Tables
                      +--- Sequences
                      +--- Views
```

Role ≠ database.

Schema ≠ database.

---

# 34. PostgreSQL service và connectivity

Kiểm tra:

```bash
systemctl status postgresql
pg_isready
```

Ubuntu/Debian:

```bash
pg_lsclusters
```

SQL connection:

```bash
sudo -u postgres psql
```

TCP:

```bash
psql -h 127.0.0.1 -U app_user -d appdb
```

Socket:

```bash
psql -U app_user -d appdb
```

Hai connection path có thể match `pg_hba.conf` rules khác nhau.

---

# 35. pg_hba.conf

Đây là authentication access policy.

Các field chính:

```text
TYPE  DATABASE  USER  ADDRESS  METHOD
```

Ví dụ:

```text
host  appdb  app_user  10.22.1.0/24  scram-sha-256
```

Rule order cực kỳ quan trọng.

PostgreSQL xử lý rules theo thứ tự; rule đầu tiên match được sử dụng và **không fall-through sang rule sau nếu authentication của rule đó thất bại**. Sau khi thay đổi trên Unix-like system, config thường cần reload; tài liệu hiện tại cũng cung cấp `pg_hba_file_rules` để kiểm tra lỗi rule. [PostgreSQL](https://www.postgresql.org/docs/18/auth-pg-hba-conf.html?utm_source=chatgpt.com)

Reload:

```bash
sudo -u postgres psql -c "SELECT pg_reload_conf();"
```

Validation bổ sung:

```sql
SELECT line_number, type, database, user_name, address, auth_method, error
FROM pg_hba_file_rules;
```

**Supplementary / Beyond the explicit syllabus:** `pg_hba_file_rules` rất hữu ích trước khi bạn kết luận rằng application fail vì password.

---

# 36. PostgreSQL least privilege

Runtime application không nên dùng `postgres`.

Một design tốt hơn:

```text
postgres
   |
administration only

app_owner / migration role
   |
DDL/schema ownership

app_runtime
   |
SELECT/INSERT/UPDATE/DELETE required by application
```

Ví dụ conceptual:

```sql
CREATE ROLE app_owner NOLOGIN;

CREATE ROLE app_runtime LOGIN;
```

Password nên đặt qua secure interactive process như:

```text
\password app_runtime
```

thay vì hard-code credential trong command history.

Database:

```sql
CREATE DATABASE appdb OWNER app_owner;
```

Runtime permissions tùy application contract.

Không blindly grant:

```sql
GRANT ALL PRIVILEGES ...
```

---

# 37. Sessions

Kiểm tra:

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    state,
    wait_event_type,
    wait_event,
    query_start,
    left(query, 200) AS query
FROM pg_stat_activity
WHERE pid <> pg_backend_pid();
```

Các state cần hiểu:

```text
active
idle
idle in transaction
```

`idle in transaction` kéo dài có thể gây vấn đề như giữ transaction/resource lâu.

Không terminate session chỉ vì nhìn thấy nó “idle”.

---

# 38. Locks

Tìm blocked sessions:

```sql
SELECT
    pid,
    pg_blocking_pids(pid) AS blocked_by,
    state,
    wait_event_type,
    wait_event,
    left(query, 200) AS query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

Flow:

```text
Slow request
   ↓
Is DB query active?
   ↓
Waiting?
   ↓
Lock?
   ↓
Who blocks?
   ↓
What transaction owns blocker?
   ↓
Safe to terminate?
```

Fresher L1 không được tự động:

```sql
SELECT pg_terminate_backend(...);
```

trên production nếu runbook/approval không cho phép.

---

# 39. EXPLAIN

Safe first step:

```sql
EXPLAIN
SELECT ...
```

`EXPLAIN` mô tả execution plan dự kiến.

`EXPLAIN ANALYZE` **thực sự execute query**.

Vì vậy:

```sql
EXPLAIN (ANALYZE, BUFFERS)
DELETE FROM ...
```

có thể rất nguy hiểm.

Trên production, L1 nên bắt đầu với normal `EXPLAIN`, query statistics và evidence; chỉ chạy `ANALYZE` nếu safe và approved.

---

# 40. Capacity

Database size:

```sql
SELECT pg_size_pretty(pg_database_size(current_database()));
```

Table size:

```sql
SELECT
    relname,
    pg_size_pretty(pg_total_relation_size(relid))
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

Host capacity:

```bash
df -hT
df -ih
free -h
```

Cả block capacity lẫn inode capacity đều quan trọng:

```bash
df -i
```

Filesystem có thể còn GB trống nhưng hết inode.

---

# 41. Backup và restore

Backup logical:

```bash
sudo install -d -o postgres -g postgres -m 700 \
  /var/lib/postgresql/backups
```

Backup:

```bash
sudo -u postgres pg_dump \
  -Fc \
  -d appdb \
  -f /var/lib/postgresql/backups/appdb.dump
```

Kiểm tra dump:

```bash
sudo -u postgres pg_restore \
  --list \
  /var/lib/postgresql/backups/appdb.dump | head
```

Restore test:

```bash
sudo -u postgres createdb appdb_restore
```

```bash
sudo -u postgres pg_restore \
  -d appdb_restore \
  /var/lib/postgresql/backups/appdb.dump
```

Sau đó chạy validation query.

Backup chỉ được xem là đáng tin khi có khả năng restore.

---

# 42. FINAL THEORY — Monitoring

Architecture:

```text
Linux Host
   |
node_exporter :9100
   |
   v
Prometheus :9090
   |
   v
Grafana :3000

Prometheus
   |
Alert Rule
   |
Alertmanager / notification flow
```

`node_exporter` **exposes metrics**.

Prometheus **scrapes/stores/queries metrics**.

Grafana **visualizes metrics**.

Không nhầm ba vai trò.

---

# 43. Prometheus scrape

Ví dụ:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets:
          - "127.0.0.1:9100"
```

Validation:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

node_exporter:

```bash
curl -fsS http://127.0.0.1:9100/metrics | head
```

Prometheus readiness:

```bash
curl -fsS http://127.0.0.1:9090/-/ready
```

---

# 44. Metrics và PromQL

Check target:

```promql
up
```

Specific:

```promql
up{job="node"}
```

CPU activity thường suy luận từ counter:

```promql
100 * (
  1 -
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

Filesystem:

```promql
node_filesystem_avail_bytes
```

Memory:

```promql
node_memory_MemAvailable_bytes
```

Đừng alert chỉ vì metric “cao”. Phải hiểu semantic của metric.

---

# 45. Prometheus alerts

Ví dụ:

```yaml
groups:
  - name: host-alerts
    rules:
      - alert: NodeExporterDown
        expr: up{job="node"} == 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "node_exporter target is unavailable"
```

Kiểm tra:

```bash
promtool check rules /etc/prometheus/rules/host-alerts.yml
```

Prometheus official docs xác nhận `for:` tạo pending duration trước khi alert được coi là firing. [prometheus.io](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/?utm_source=chatgpt.com)

Điều này tránh transient spike làm alert ngay lập tức.

---

# 46. Grafana

Grafana flow:

```text
Data source
   ↓
Query
   ↓
Panel
   ↓
Dashboard
```

Grafana Alerting có thể evaluate Prometheus-backed rules và route notification qua contact point/policy tùy configuration. [Grafana Labs](https://grafana.com/docs/grafana/latest/datasources/prometheus/alerting/?utm_source=chatgpt.com)

Dashboard không chứng minh monitoring hoàn chỉnh.

Bạn cần:

```text
metrics exist
targets healthy
dashboard query works
alert rule evaluates
failure is injected
alert reaches pending/firing
recovery happens
evidence captured
```

---

# 47. FINAL THEORY — Batch, Change, Release

Change management không phải bureaucracy vô nghĩa. Nó giảm blast radius.

Flow:

```text
Request
  ↓
Scope
  ↓
Risk assessment
  ↓
Implementation plan
  ↓
Validation plan
  ↓
Rollback plan
  ↓
Approval
  ↓
Execution
  ↓
Evidence
  ↓
Validation
  ↓
Closure / rollback
  ↓
Handover
```

Một ticket tốt phải trả lời:

| Nội dung         | Ví dụ                           |
| ---------------- | ------------------------------- |
| What             | Deploy `myapp-1.4.2.war`        |
| Why              | approved release                |
| Where            | `app-prod-01`                   |
| When             | maintenance window              |
| Who              | operator + approver             |
| Risk             | service interruption            |
| Pre-check        | current health OK               |
| Implementation   | exact runbook                   |
| Validation       | endpoint + DB + logs + metrics  |
| Rollback trigger | health check fails              |
| Rollback         | restore previous WAR            |
| Evidence         | command/log/timestamp/reference |
| Outcome          | success / rolled back           |

---

# 48. Batch scheduling

Batch correctness không chỉ là “cron chạy”.

Phải xác minh:

```text
Was job started?
Was dependency ready?
What exit code?
Was expected output produced?
Was downstream step executed?
Did monitoring detect failure?
```

Ví dụ:

```bash
/usr/local/bin/daily-job.sh
rc=$?

printf 'exit_code=%s\n' "$rc"

if [ "$rc" -ne 0 ]; then
    exit "$rc"
fi
```

Exit code phải propagate đúng.

---

# 49. FINAL THEORY — L1 Troubleshooting & Escalation

Đây là skill quan trọng nhất.

Framework:

```text
SYMPTOM
   ↓
EXPECTED BEHAVIOR
   ↓
RECENT CHANGES
   ↓
SCOPE
   ↓
FAILING LAYER
   ↓
EVIDENCE
   ↓
HYPOTHESES
   ↓
SAFEST DISCRIMINATING TEST
   ↓
ROOT CAUSE
   ↓
L1-SAFE REMEDIATION
   ↓
VALIDATION
   ↓
PREVENTION / ESCALATION
```

Ví dụ user nói:

```text
"Website không truy cập được."
```

Không restart Tomcat ngay.

Đi lần lượt:

```bash
curl -v https://service.example/
```

Nếu remote fail:

```text
DNS?
IP?
route?
AWS SG?
```

Trên EC2:

```bash
curl -v http://127.0.0.1:8080/app/
```

Nếu local cũng fail:

```bash
ss -ltnp | grep ':8080'
systemctl status tomcat
journalctl -u tomcat
```

Nếu Tomcat up nhưng app lỗi:

```text
application logs
configuration
database
```

DB:

```bash
pg_isready
```

Monitoring:

```promql
up
```

Đó là evidence-driven troubleshooting.

---

# 50. Khi nào L1 nên remediate và khi nào escalate?

| Situation                                                 | Typical response                        |
| --------------------------------------------------------- | --------------------------------------- |
| Service stopped, known cause, approved restart            | L1 restart + validate                   |
| Wrong permission caused by approved deployment            | Correct according to runbook            |
| Known config typo in approved change                      | rollback/correct per ticket             |
| Disk filled by known temporary logs with approved cleanup | cleanup safely                          |
| Unknown data corruption                                   | Escalate                                |
| Security compromise suspected                             | Preserve evidence + security escalation |
| Unknown repeated JVM OOM                                  | Evidence + escalate                     |
| Production database destructive action required           | Escalate                                |
| Need broad IAM privilege change                           | Escalate                                |
| Schema/database corruption                                | DBA escalation                          |
| Application code exception requiring code fix             | Developer escalation                    |
| Unauthorized change discovered                            | stop/change-control escalation          |

“Có root” không đồng nghĩa “được phép sửa”.

---

# 51. PHẦN THỰC HÀNH DAY 22 — Integrated Final Practical

Bây giờ ta dựng một scenario sát Final Practical.

### Scenario

Bạn nhận ticket:

```text
Deploy Java web application "FresherAdmin"
onto an AWS EC2 Linux host.

Application must:
- run under Tomcat
- use PostgreSQL
- start through systemd
- expose its HTTP endpoint
- be monitored by Prometheus/node_exporter
- have a Grafana dashboard
- include an operational alert
- use least-privilege configuration
- provide evidence
- survive an injected failure
- support rollback
- finish with handover documentation
```

Không dùng production. AWS resources có thể phát sinh phí. Terminate/delete resources sau lab nếu không còn cần.

---

# 52. Phase 0 — Pre-change inspection

Trước deploy phải thu baseline:

```bash
date -Is
hostnamectl
uname -a
cat /etc/os-release

ip addr
ip route

df -hT
df -ih
free -h

systemctl --failed
ss -ltnp
```

Expected invariant:

```text
host reachable
root filesystem has safe free capacity
no unexpected failed critical services
network configuration understood
```

Evidence:

```bash
mkdir -p ~/day22-evidence
```

```bash
{
  date -Is
  hostnamectl
  df -hT
  free -h
  systemctl --failed
  ss -ltnp
} > ~/day22-evidence/00-baseline.txt
```

Review before submission:

```bash
grep -Ein 'password|secret|token|access.?key' \
  ~/day22-evidence/00-baseline.txt
```

Evidence pack không được chứa credentials.

---

# 53. Phase 1 — AWS architecture

Cost-efficient architecture:

```text
Internet
   |
   v
Internet Gateway
   |
   v
Public Subnet
   |
   v
Security Group
   |
   v
EC2
 ├── Linux
 ├── Tomcat
 ├── PostgreSQL
 ├── Prometheus
 ├── node_exporter
 └── Grafana
```

Một EC2 đủ cho lab tích hợp và giảm chi phí. Production architecture thực tế có thể tách database/monitoring, nhưng đó không phải mandatory requirement của syllabus.

Security Group:

```text
SSH     :22    YOUR_IP/32
App     :8080  YOUR_IP/32
Grafana :3000  YOUR_IP/32 if required

No public 5432
No public 9100
No public 9090 unless specifically required
```

Tag:

```text
Name=day22-integrated
Environment=lab
Purpose=training
Owner=<approved identifier>
```

---

# 54. Phase 2 — EC2 bootstrap

Kết nối:

```bash
ssh -i lab-key.pem ubuntu@PUBLIC_IP
```

Update package metadata:

```bash
sudo apt update
```

Inspect trước upgrade hàng loạt. Trong final practical, không tự ý:

```bash
sudo apt full-upgrade -y
```

trên một assigned system nếu ticket không yêu cầu.

Install utilities nếu approved:

```bash
sudo apt install -y curl unzip jq net-tools
```

`net-tools` không bắt buộc; `ss` từ iproute2 nên là tool chính.

---

# 55. Phase 3 — Java validation

Kiểm tra trước:

```bash
java -version
command -v java
```

Nếu assignment yêu cầu Java 17 và chưa có:

```bash
sudo apt install -y openjdk-17-jdk
```

Validation:

```bash
java -version
javac -version
```

Capture:

```bash
java -version 2>&1 |
  tee ~/day22-evidence/10-java-version.txt
```

Không tự upgrade Java 17 → 25 nếu build sheet yêu cầu Java 17.

---

# 56. Phase 4 — Tomcat

Nếu exam đã cung cấp Tomcat, sử dụng artifact được cung cấp.

Nếu tự cài manual, phải lấy release từ trusted Apache source, verify checksum/signature theo lab procedure và không hard-code một download URL/version cũ chỉ vì từng học nó.

Ví dụ directory:

```bash
sudo mkdir -p /opt/tomcat
```

Service account:

```bash
sudo useradd \
  --system \
  --home-dir /opt/tomcat \
  --shell /usr/sbin/nologin \
  tomcat
```

Ownership:

```bash
sudo chown -R tomcat:tomcat /opt/tomcat
```

Kiểm tra:

```bash
id tomcat
ls -ld /opt/tomcat
```

---

# 57. Phase 5 — PostgreSQL

Install nếu environment chưa có:

```bash
sudo apt install -y postgresql postgresql-contrib
```

Validate:

```bash
systemctl status postgresql --no-pager
pg_isready
```

Version:

```bash
psql --version
```

Discover paths:

```bash
sudo -u postgres psql -Atc 'SHOW config_file;'
sudo -u postgres psql -Atc 'SHOW hba_file;'
sudo -u postgres psql -Atc 'SHOW data_directory;'
```

Đây là bằng chứng tốt hơn việc học thuộc path.

---

# 58. Phase 6 — Database roles

Vào psql:

```bash
sudo -u postgres psql
```

Ví dụ:

```sql
CREATE ROLE fresher_owner NOLOGIN;
CREATE DATABASE fresherdb OWNER fresher_owner;

CREATE ROLE fresher_app LOGIN;
```

Đặt password interactively:

```text
\password fresher_app
```

Không paste password vào evidence.

Connect:

```text
\c fresherdb
```

Runtime access phải dựa trên schema/application requirement.

Ví dụ nếu application đã có table do migration owner tạo:

```sql
GRANT CONNECT ON DATABASE fresherdb TO fresher_app;
GRANT USAGE ON SCHEMA public TO fresher_app;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA public
TO fresher_app;

GRANT USAGE, SELECT
ON ALL SEQUENCES IN SCHEMA public
TO fresher_app;
```

Nếu future objects được tạo bởi `fresher_owner`:

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE fresher_owner
IN SCHEMA public
GRANT SELECT, INSERT, UPDATE, DELETE
ON TABLES TO fresher_app;
```

```sql
ALTER DEFAULT PRIVILEGES
FOR ROLE fresher_owner
IN SCHEMA public
GRANT USAGE, SELECT
ON SEQUENCES TO fresher_app;
```

Đây là design tốt hơn dùng superuser cho application.

---

# 59. Phase 7 — Database connectivity test

Không cần app trước.

```bash
psql -h 127.0.0.1 \
     -U fresher_app \
     -d fresherdb
```

Sau connect:

```sql
SELECT current_database(), current_user;
```

Expected invariant:

```text
current_database = fresherdb
current_user     = fresher_app
```

Nếu fail:

```text
connection refused
    → server/listener

timeout
    → routing/firewall/SG

password authentication failed
    → user/password/HBA

no pg_hba.conf entry
    → HBA rule

permission denied
    → SQL privilege
```

Không gom tất cả thành “database down”.

---

# 60. Phase 8 — Application configuration

Vì syllabus không cung cấp cụ thể tên property của application, ta **không được invent** `spring.datasource.*` hay một config key cụ thể và giả vờ nó thuộc application.

Bạn phải lấy configuration contract từ artifact/build sheet.

Pattern chung:

```text
DB host     = 127.0.0.1
DB port     = 5432
DB database = fresherdb
DB username = fresher_app
DB password = secret source
```

Có thể đặt environment file:

```bash
sudo install -d -o root -g tomcat -m 750 /etc/fresheradmin
```

Tạo:

```text
/etc/fresheradmin/app.env
```

Permissions:

```bash
sudo chown root:tomcat /etc/fresheradmin/app.env
sudo chmod 640 /etc/fresheradmin/app.env
```

Verify:

```bash
sudo -u tomcat test -r /etc/fresheradmin/app.env
echo $?
```

Không capture file content nếu chứa secret.

---

# 61. Phase 9 — Deploy WAR

Pre-change:

```bash
ls -lh application.war
sha256sum application.war
jar tf application.war | head
```

Ghi checksum vào evidence:

```bash
sha256sum application.war |
  tee ~/day22-evidence/20-artifact-sha256.txt
```

Backup current artifact nếu đang upgrade:

```bash
sudo cp -a \
  /opt/tomcat/webapps/fresheradmin.war \
  /opt/tomcat/releases/fresheradmin.war.before-change
```

Chỉ chạy nếu file tồn tại và release layout của bạn dùng pattern đó.

Deploy:

```bash
sudo install \
  -o tomcat \
  -g tomcat \
  -m 640 \
  application.war \
  /opt/tomcat/webapps/fresheradmin.war
```

---

# 62. Phase 10 — systemd

Environment file có thể được reference:

```ini
EnvironmentFile=/etc/fresheradmin/app.env
```

Unit:

```ini
[Unit]
Description=FresherAdmin Tomcat
After=network.target postgresql.service

[Service]
Type=simple
User=tomcat
Group=tomcat
EnvironmentFile=/etc/fresheradmin/app.env
ExecStart=/opt/tomcat/bin/catalina.sh run
Restart=on-failure
RestartSec=5
SuccessExitStatus=143

[Install]
WantedBy=multi-user.target
```

Sau đó:

```bash
sudo systemctl daemon-reload
sudo systemctl enable tomcat
sudo systemctl start tomcat
```

Validate:

```bash
systemctl is-enabled tomcat
systemctl is-active tomcat
systemctl status tomcat --no-pager
```

---

# 63. Phase 11 — Five-level application validation

Đây là pattern rất nên thuộc.

### Level 1 — Process

```bash
pgrep -af java
```

### Level 2 — Socket

```bash
ss -ltnp | grep ':8080'
```

### Level 3 — HTTP

```bash
curl -v http://127.0.0.1:8080/fresheradmin/
```

### Level 4 — Logs

```bash
journalctl -u tomcat --since "-5 min" --no-pager
```

Nếu application log riêng:

```bash
tail -n 100 /path/to/application.log
```

### Level 5 — Dependency

Test endpoint có sử dụng database hoặc kiểm tra application health theo contract.

Không tuyên bố deploy successful chỉ vì:

```bash
systemctl status tomcat
```

hiện `active`.

Tomcat có thể active trong khi WAR deployment fail.

---

# 64. Phase 12 — Monitoring deployment

node_exporter phải được reachable bởi Prometheus.

Kiểm tra:

```bash
curl -fsS http://127.0.0.1:9100/metrics | head
```

Prometheus scrape:

```yaml
scrape_configs:
  - job_name: node
    static_configs:
      - targets:
          - "127.0.0.1:9100"
```

Validate:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Sau apply/restart theo service procedure:

```bash
curl -fsS http://127.0.0.1:9090/-/ready
```

PromQL:

```promql
up{job="node"}
```

Expected invariant:

```text
1
```

---

# 65. Phase 13 — Dashboard

Grafana datasource:

```text
Prometheus URL:
http://127.0.0.1:9090
```

Dashboard nên ít nhất thể hiện:

| Signal                                                      | Meaning                    |
| ----------------------------------------------------------- | -------------------------- |
| Host CPU                                                    | saturation/load clue       |
| Memory available                                            | capacity                   |
| Filesystem free                                             | disk risk                  |
| `up`                                                        | scrape target availability |
| Application availability nếu exposed metric/blackbox exists | service signal             |

Day 22 không yêu cầu một dashboard đẹp. Nó yêu cầu **monitoring usable for operations**.

---

# 66. Phase 14 — Alert

Rule:

```yaml
groups:
  - name: day22
    rules:
      - alert: NodeExporterDown
        expr: up{job="node"} == 0
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "node_exporter unavailable"
```

Syntax check:

```bash
promtool check rules /etc/prometheus/rules/day22.yml
```

Sau khi apply, xem:

```text
inactive
→ pending
→ firing
```

---

# 67. FAILURE INJECTION 1 — Service failure

Trong lab:

```bash
sudo systemctl stop tomcat
```

Không coi đây là command production nếu không được approved.

Symptom:

```text
application unavailable
```

Evidence:

```bash
systemctl status tomcat --no-pager
ss -ltnp | grep ':8080'
curl -v http://127.0.0.1:8080/
journalctl -u tomcat --since "-5 min"
```

Expected:

```text
service inactive
no listener 8080
curl connection refused
```

Recover:

```bash
sudo systemctl start tomcat
```

Validate lại toàn bộ five levels.

---

# 68. FAILURE INJECTION 2 — Port conflict

Lab-only approach:

Trước tiên xác minh 8080 đang dùng bởi ai:

```bash
sudo ss -ltnp | grep ':8080'
```

Nếu một process khác bind 8080, Tomcat có thể báo dạng:

```text
Address already in use
BindException
```

Workflow:

```text
Tomcat failed
  ↓
journal
  ↓
BindException
  ↓
ss -ltnp
  ↓
identify PID/process
  ↓
determine whether conflicting process is legitimate
```

Không:

```bash
kill -9 RANDOM_PID
```

chỉ để chiếm port.

---

# 69. FAILURE INJECTION 3 — Permission failure

Lab-only:

Giả sử Tomcat phải đọc config nhưng permission sai.

Evidence:

```bash
ls -l /etc/fresheradmin/app.env
namei -l /etc/fresheradmin/app.env
sudo -u tomcat test -r /etc/fresheradmin/app.env
echo $?
```

Logs có thể hiện:

```text
Permission denied
```

Root cause:

```text
owner/group/mode hoặc directory traversal permission
```

Remediation theo baseline:

```bash
sudo chown root:tomcat /etc/fresheradmin/app.env
sudo chmod 640 /etc/fresheradmin/app.env
```

Validation:

```bash
sudo -u tomcat test -r /etc/fresheradmin/app.env
```

---

# 70. FAILURE INJECTION 4 — Disk full

Không fill `/` đến 100% trên machine quan trọng.

Trong lab có thể dùng một small dedicated test filesystem.

Symptoms:

```text
No space left on device
logs stop writing
deployment fails
DB may malfunction
```

Evidence:

```bash
df -hT
df -ih
du -xhd1 /var | sort -h
sudo lsof +L1
```

Phải phân biệt:

```text
blocks exhausted
inode exhausted
deleted-open files
```

Remediation phụ thuộc root cause.

Không:

```bash
rm -rf /var/log/*
```

---

# 71. FAILURE INJECTION 5 — PostgreSQL authentication

Ví dụ intentionally incorrect password/config trong lab.

Application log:

```text
password authentication failed
```

Workflow:

```bash
pg_isready
```

Nếu DB reachable:

```bash
psql -h 127.0.0.1 -U fresher_app -d fresherdb
```

Sau đó kiểm tra:

```text
username
database
credential source
pg_hba rule
role LOGIN capability
```

Không reset password ngay trước khi biết application đang sử dụng credential nào.

---

# 72. FAILURE INJECTION 6 — PostgreSQL unreachable

Nếu application báo connection refused:

```bash
pg_isready -h 127.0.0.1 -p 5432
ss -ltnp | grep ':5432'
systemctl status postgresql
```

Nếu remote DB:

```text
route
Security Group
NACL
OS firewall
listen_addresses
pg_hba.conf
```

`pg_hba.conf` không quyết định process có listen trên interface hay không. `listen_addresses` và listener phải đúng trước khi HBA có ý nghĩa.

---

# 73. FAILURE INJECTION 7 — Alert failure

Symptom:

```text
node_exporter stopped
but no alert
```

Đi theo pipeline:

```text
node_exporter actually down?
       ↓
Prometheus target shows down?
       ↓
up == 0?
       ↓
rule loaded?
       ↓
expression correct?
       ↓
for duration elapsed?
       ↓
alert firing?
       ↓
notification route/contact point?
```

Commands:

```bash
systemctl status node_exporter
curl -v http://127.0.0.1:9100/metrics

promtool check config /etc/prometheus/prometheus.yml
promtool check rules /etc/prometheus/rules/day22.yml
```

Nếu `for: 2m`, đừng mong alert firing sau 3 giây.

---

# 74. FAILURE INJECTION 8 — Bad deployment và rollback

Scenario:

```text
new WAR deployed
Tomcat running
endpoint returns HTTP 500
```

Evidence:

```bash
systemctl status tomcat
ss -ltnp | grep ':8080'
curl -v http://127.0.0.1:8080/fresheradmin/
journalctl -u tomcat --since "-10 min"
```

Nếu failure gắn trực tiếp với approved new release và rollback criteria đạt:

```text
stop service
preserve bad-release logs/checksum
restore known-good WAR
start service
validate
document rollback
```

Điểm exam rất quan trọng:

```text
Tomcat active
```

không đồng nghĩa:

```text
release successful
```

---

# 75. Troubleshooting Matrix tổng hợp

| Symptom                     | Evidence đầu tiên           | Possible layer |
| --------------------------- | --------------------------- | -------------- |
| SSH timeout                 | SG/routes/port 22           | AWS/network    |
| SSH permission denied       | ssh log/user/key            | identity       |
| HTTP connection refused     | `ss`, service               | process/socket |
| HTTP 404                    | context path/deployment     | Tomcat/app     |
| HTTP 500                    | application logs            | app/dependency |
| DB connection refused       | `pg_isready`, `ss`          | DB listener    |
| DB password failure         | HBA/role/credential         | authentication |
| Permission denied           | `namei -l`, `ls -l`         | filesystem     |
| `Address already in use`    | `ss -ltnp`                  | port conflict  |
| `No space left`             | `df -h`, `df -i`            | storage        |
| JVM killed                  | kernel/OOM logs             | memory         |
| Metric absent               | exporter `/metrics`, target | monitoring     |
| Alert not firing            | rule/expression/`for`       | alerting       |
| Deployment bad after change | logs + previous version     | release        |

Đây là table bạn nên có khả năng tái tạo bằng tư duy, không phải học thuộc từng dòng.

---

# 76. Rollback philosophy

Rollback tốt phải được thiết kế **trước change**, không phải sau khi failure xảy ra mới nghĩ.

Ví dụ deployment:

```text
Before:
fresheradmin-1.3.0.war = known good

Change:
deploy 1.4.0

Rollback trigger:
health endpoint non-2xx
DB operation fails
critical error in logs
monitoring degradation

Rollback:
restore 1.3.0
restart
validate
```

Evidence cần ghi:

```text
release before
release attempted
failure
decision time
rollback executed
post-rollback validation
```

---

# 77. Security review trước khi “pass” Day 22

System có chạy được nhưng security kém vẫn không pass về mặt operational competence.

Phải kiểm tra:

```bash
id tomcat
ls -l /etc/fresheradmin/
ss -ltnp
sudo -l
```

AWS:

```text
No unnecessary 0.0.0.0/0
No public PostgreSQL
No exposed node_exporter
IAM least privilege
MFA
No long-lived credentials embedded on server
resource tagging
```

Database:

```text
application not superuser
HBA narrowly scoped
credentials protected
backup protected
```

Monitoring:

```text
Grafana not anonymously/publicly exposed without need
alert destinations protected
dashboards do not reveal credentials
```

Logs:

```text
no passwords
no access keys
no tokens
```

---

# 78. Evidence pack — một phần của practical exam

Một cấu trúc tốt:

```text
day22-evidence/
├── 00-baseline.txt
├── 10-java-version.txt
├── 11-tomcat-service.txt
├── 12-listeners.txt
├── 20-artifact-sha256.txt
├── 30-postgresql-health.txt
├── 31-db-validation.txt
├── 40-application-validation.txt
├── 50-prometheus-target.txt
├── 51-alert-validation.txt
├── 60-failure-investigation.txt
├── 61-recovery-validation.txt
├── 70-rollback.txt
└── 80-handover.txt
```

Không cần file naming giống hệt. Điểm cốt lõi là evidence có chronology và đủ chứng minh.

Evidence tốt:

```text
2026-10-12T...
command/action
result
interpretation
```

Evidence xấu:

```text
"It works."
```

---

# 79. Handover

Một handover chuyên nghiệp phải đủ để người ca sau tiếp tục vận hành mà không phải hỏi lại mọi thứ.

Ví dụ nội dung:

```text
Service:
FresherAdmin

Host:
day22-app-01

Current release:
1.4.0

Status:
Application healthy

Endpoint:
approved application endpoint

Tomcat:
active

Database:
reachable; runtime account validated

Monitoring:
Prometheus target UP
Grafana dashboard validated
NodeExporterDown rule validated

Changes:
WAR deployed
external config updated
Tomcat restarted

Validation:
HTTP smoke test passed
DB operation passed
logs reviewed
no unexpected critical errors

Known issues:
none / details

Rollback:
previous WAR location/reference documented

Pending:
none / next action

Evidence:
ticket attachments/reference

Security:
no credentials included in handover
```

Đây không chỉ là paperwork. Handover là một operational control.

---

# 80. Final Practical — bạn được đánh giá thực sự ở đâu?

Examiner không chỉ nhìn:

```text
Did you make it work?
```

Mà về mặt competency, Day 22 đòi hỏi bạn suy nghĩ được:

```text
Did you inspect before changing?
Did you understand blast radius?
Did you protect secrets?
Did you use least privilege?
Did you validate every layer?
Did you preserve evidence?
Did you troubleshoot from evidence?
Did you know when NOT to change something?
Did you have rollback?
Did you verify rollback?
Did you hand over clearly?
```

Đây chính là sự khác nhau giữa configuration exercise và operations exam.

---

# 81. Một sequence nên thuộc lòng cho practical

Khi được giao một machine mới, mental sequence của bạn nên là:

```text
Understand ticket
        ↓
Confirm scope and approval
        ↓
Inspect environment
        ↓
Capture baseline
        ↓
Verify network/AWS
        ↓
Verify storage
        ↓
Verify Java
        ↓
Verify/install Tomcat
        ↓
Verify PostgreSQL
        ↓
Configure least privilege
        ↓
Externalize app config
        ↓
Deploy artifact
        ↓
Start through systemd
        ↓
Process validation
        ↓
Socket validation
        ↓
HTTP validation
        ↓
Database validation
        ↓
Logs validation
        ↓
Monitoring validation
        ↓
Alert validation
        ↓
Failure injection
        ↓
Troubleshoot
        ↓
Recover
        ↓
Rollback test when applicable
        ↓
Evidence
        ↓
Handover
        ↓
AWS cleanup / cost control
```

Nếu bạn có thể thực hiện chuỗi này độc lập, bạn đã rất gần mục tiêu cuối của syllabus.

---

# 82. Những anti-pattern Day 22 rất dễ mất điểm

| Anti-pattern                             | Vì sao sai                         |
| ---------------------------------------- | ---------------------------------- |
| Restart ngay khi có incident             | phá evidence, chưa biết root cause |
| `chmod 777` để “fix permission”          | bỏ least privilege                 |
| chạy Tomcat root                         | blast radius                       |
| dùng PostgreSQL `postgres` cho app       | excessive privilege                |
| mở `5432 0.0.0.0/0`                      | database exposure                  |
| mở mọi port SG                           | security failure                   |
| hard-code password trong WAR/config repo | secret exposure                    |
| dùng `kill -9` đầu tiên                  | non-graceful                       |
| dùng `rm -rf` để xử lý disk full         | destructive                        |
| xem `active` là app healthy              | incomplete validation              |
| backup không restore test                | unproven recovery                  |
| `EXPLAIN ANALYZE` bừa trên production    | executes query                     |
| sửa config không backup                  | poor rollback                      |
| thay đổi không có ticket/approval        | change control violation           |
| không timestamp evidence                 | khó reconstruct incident           |
| handover có password/token               | security incident                  |
| tạo AWS resource rồi bỏ quên             | cost/operations failure            |

---

# 83. Supplementary / Beyond the explicit syllabus — Tomcat 10 migration trap

Nếu application cũ được build cho Java EE/Tomcat 9 và import:

```java
javax.servlet.*
```

rồi deploy vào Tomcat 10.1, có thể failure vì Tomcat 10.x sử dụng Jakarta APIs:

```java
jakarta.servlet.*
```

Tomcat 10.1 hiện implement Jakarta Servlet 6.0. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/?utm_source=chatgpt.com)

Nếu thấy:

```text
ClassNotFoundException
NoClassDefFoundError
javax.servlet...
```

đừng kết luận Java version sai ngay.

Có thể là API migration problem.

Đây là ví dụ tốt về:

```text
symptom
≠
root cause
```

---

# 84. Supplementary — PostgreSQL authentication hiện đại

Current PostgreSQL documentation ghi MD5-encrypted password authentication đang deprecated và dự kiến được loại bỏ trong future release. Vì vậy với lab/config mới nên hiểu và ưu tiên `scram-sha-256` khi environment/application hỗ trợ. [PostgreSQL](https://www.postgresql.org/docs/18/auth-pg-hba-conf.html?utm_source=chatgpt.com)

Điều này **không thay đổi syllabus requirement**; nó là current operational knowledge bổ sung.

---

# 85. Knowledge Check — Final Theory

Hãy thử trả lời bằng miệng trước khi đọc đáp án ngắn trong cột cuối.

| Question                                          | Expected core answer                                        |
| ------------------------------------------------- | ----------------------------------------------------------- |
| `systemctl is-enabled` khác `is-active`?          | boot configuration vs runtime state                         |
| SG stateful hay stateless?                        | stateful                                                    |
| NACL?                                             | stateless subnet-level filtering                            |
| Public subnet chỉ cần IGW là đủ cho EC2 Internet? | không; cần routing + public addressing + security config    |
| `-Xmx` là total Java RAM?                         | không, chỉ max Java heap                                    |
| Tomcat running nghĩa application healthy?         | không                                                       |
| `pg_hba.conf` dùng để làm gì?                     | client authentication rules                                 |
| HBA có fall-through sau failed auth không?        | không                                                       |
| `EXPLAIN ANALYZE` có execute query không?         | có                                                          |
| `pg_dump` tạo ra gì?                              | logical database backup                                     |
| Backup thành công đã đủ chưa?                     | chưa, cần restore verification                              |
| node_exporter làm gì?                             | expose host metrics                                         |
| Prometheus làm gì?                                | scrape/store/query time-series                              |
| Grafana làm gì?                                   | visualize/query and possibly alert                          |
| `for: 5m` trong Prometheus alert?                 | condition phải tồn tại đủ thời gian trước firing            |
| CloudTrail vs CloudWatch?                         | API audit vs monitoring/observability                       |
| SIGTERM vs SIGKILL?                               | graceful request vs immediate kill                          |
| `df` vs `du`?                                     | filesystem utilization vs file/directory blocks             |
| `chmod 777` để fix permission?                    | anti-pattern                                                |
| App dùng DB superuser?                            | violation of least privilege                                |
| Khi unknown DB corruption?                        | preserve evidence + escalate                                |
| Handover có password?                             | không                                                       |
| Rollback plan viết khi nào?                       | trước change                                                |
| Change success xác nhận bằng gì?                  | defined validation evidence                                 |
| Root cause khác symptom?                          | symptom là observable effect; root cause là nguyên nhân nền |

Nếu những câu này phải suy nghĩ rất lâu, đó là khu vực cần revision thêm.

---

# 86. Independent Practical Challenge

Sau khi đọc xong bài này, bài luyện Day 22 thực sự nên làm là:

```text
Build one EC2 Linux lab.

Install/validate:
Java
Tomcat
PostgreSQL
Prometheus
node_exporter
Grafana

Deploy one WAR.

Run Tomcat under a dedicated service account.

Externalize configuration.

Create least-privilege PostgreSQL access.

Validate:
process
port
HTTP
database
logs
metrics
dashboard
alert

Inject:
one service failure
one permission failure
one database connectivity/authentication failure
one monitoring failure
one bad deployment

For every incident:
capture symptom
identify expected state
identify failing layer
collect evidence
form hypotheses
perform safe tests
find root cause
recover
revalidate
document

Finally:
perform rollback
produce evidence pack
write handover
clean AWS resources.
```

Đây là **một bài practical**, không phải năm bài rời rạc.

---

# 87. Mastery Check Day 22

Bạn chưa nên tự đánh dấu “mastered” chỉ vì hiểu bài giải thích trên. Với Day 22, mastery thực tế chỉ nên được công nhận khi bạn có thể độc lập thực hiện:

```text
understand
→ configure
→ deploy
→ validate
→ monitor
→ troubleshoot
→ recover
→ rollback
→ document
→ escalate
→ hand over
```

và quan trọng hơn, biết **khi nào không nên tự ý thao tác**.

---

## Kết luận Day 22

Day 22 là bài kiểm tra xem kiến thức Day 1–21 đã trở thành **operational thinking** hay chưa. MASTER_SYLLABUS yêu cầu theory trên toàn bộ Linux, AWS, Java configuration, PostgreSQL, monitoring, change controls và escalation; practical yêu cầu tích hợp deploy/configure/validate/troubleshoot/document/handover. 00_MASTER_SYLLABUS_Java_Fresher…

Mental model quan trọng nhất bạn cần giữ là:

```text
Client
→ Network/AWS
→ EC2/Linux
→ Process
→ Port
→ Tomcat/JVM
→ Application
→ Configuration
→ PostgreSQL
→ Logs/Metrics
→ Alert

Symptom
→ Evidence
→ Failing layer
→ Hypothesis
→ Safe test
→ Root cause
→ Safe remediation / escalation
→ Validation
→ Evidence
→ Handover
```

**Bước học tốt nhất tiếp theo cho Day 22 là làm practical thật, không đọc thêm theory.** Bạn có thể gửi tôi từng command/output/evidence khi thực hành; theo **PRACTICE MODE**, tôi sẽ đóng vai mentor/examiner, kiểm tra từng bước, chỉ ra lỗi vận hành, bắt bạn troubleshooting bằng evidence và không cho “pass” chỉ vì service cuối cùng chạy được.
