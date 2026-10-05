Được. Tôi sẽ thiết kế môi trường này theo hướng **lab dùng để thi/demo**, tức là đủ thành phần để bạn thực hành Linux → AWS → Tomcat/Java → PostgreSQL → Prometheus/Grafana, nhưng vẫn giữ **blast radius nhỏ, security hợp lý và chi phí thấp**.

Theo syllabus, phần này trực tiếp phục vụ Day 8 (VPC/subnet/route table/IGW/SG), Day 9 (EC2/key pair/user data/SSH), Day 12 (Java/Tomcat), Day 15 (PostgreSQL), Day 17 (Prometheus/node_exporter/Grafana) và Day 19 (integrated environment). :chatgpt-content-reference{index="0"} :chatgpt-content-reference{index="1"} :chatgpt-content-reference{index="2"} :chatgpt-content-reference{index="3"}

Demo guide cũng yêu cầu bạn phải có sẵn AWS lab chứa EC2 + Tomcat + PostgreSQL + Prometheus/Grafana trước khi bắt đầu phần practical. :chatgpt-content-reference{index="4"}

---

# 1. Kiến trúc chúng ta sẽ dựng

Tôi đề xuất baseline sau:

```text
Windows 11
└── WSL Ubuntu
    └── SSH private key
          |
          | TCP/22
          v
Internet
    |
    v
Internet Gateway
    |
    v
VPC: 10.10.0.0/16
|
+-- Public Subnet: 10.10.10.0/24
|     |
|     +-- Public Route Table
|     |      10.10.0.0/16 -> local
|     |      0.0.0.0/0    -> Internet Gateway
|     |
|     +-- EC2 Ubuntu 24.04 LTS
|            |
|            +-- Java 17
|            +-- Tomcat 10       :8080
|            +-- PostgreSQL 16   :5432
|            +-- Prometheus      :9090
|            +-- node_exporter   :9100
|            +-- Grafana         :3000
|
+-- Private Subnet: 10.10.20.0/24
      |
      +-- Private Route Table
             10.10.0.0/16 -> local
             NO internet route
```

Security Group:

| Port | Service       | Source                     |
| ---: | ------------- | -------------------------- |
|   22 | SSH           | **Your public IP/32 only** |
| 8080 | Tomcat        | **Your public IP/32 only** |
| 3000 | Grafana       | **Your public IP/32 only** |
| 9090 | Prometheus    | **Your public IP/32 only** |
| ICMP | ping testing  | **Your public IP/32 only** |
| 5432 | PostgreSQL    | **NOT exposed**            |
| 9100 | node_exporter | **NOT exposed**            |

PostgreSQL và node_exporter không có lý do gì phải expose ra Internet trong kiến trúc 1 EC2 này.

Security Groups là **stateful**: traffic response cho một connection hợp lệ tự động được phép quay trở lại. :chatgpt-content-reference{index="5"}

---

# 2. Environment/version assumptions

Tôi sẽ dùng:

- AWS Region ví dụ: **Singapore `ap-southeast-1`**
- Ubuntu Server **24.04 LTS**
- Architecture: `x86_64`
- Java: OpenJDK **17**
- Tomcat package: `tomcat10`
- PostgreSQL: **16**
- Prometheus: Ubuntu package
- node_exporter: Ubuntu package
- Grafana OSS: official Grafana APT repository
- EC2: `t3.medium`
- Root EBS: `20–30 GiB gp3`

Ubuntu 24.04 hiện có các package `tomcat10`, PostgreSQL 16, Prometheus và prometheus-node-exporter. :chatgpt-content-reference{index="6"}

Grafana hiện hỗ trợ cài trên Ubuntu/Debian qua official APT repository. :chatgpt-content-reference{index="7"}

> **Supplementary / Beyond explicit syllabus:** Tomcat 10 sử dụng Jakarta namespace. Nếu sau này trainer đưa cho bạn một WAR cũ dùng `javax.*`, phải kiểm tra compatibility trước khi deploy. Đừng mặc định WAR nào cũng deploy được trên Tomcat 10.

---

# 3. Bước 0 — chuẩn bị SSH key ngay trong WSL

Tôi khuyên **không tạo private key trên AWS rồi copy về**, mà tạo key trực tiếp trong WSL. Như vậy private key không rời khỏi máy bạn.

AWS hỗ trợ import public key RSA hoặc ED25519; AWS chỉ nhận **public key**, không nhận private key. :chatgpt-content-reference{index="8"}

Mở WSL Ubuntu:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Kiểm tra:

```bash
ls -ld ~/.ssh
```

Sau đó tạo key:

```bash
ssh-keygen -t ed25519 -a 100 \
  -f ~/.ssh/java-fresher-lab \
  -C "java-fresher-lab"
```

Ý nghĩa:

```text
ssh-keygen
```

tạo SSH key pair.

```text
-t ed25519
```

chọn thuật toán ED25519.

```text
-a 100
```

tăng số vòng KDF dùng để bảo vệ private key nếu bạn đặt passphrase.

```text
-f ~/.ssh/java-fresher-lab
```

private key:

```text
~/.ssh/java-fresher-lab
```

public key:

```text
~/.ssh/java-fresher-lab.pub
```

```text
-C "java-fresher-lab"
```

thêm comment nhận diện key.

Kiểm tra:

```bash
ls -l ~/.ssh/java-fresher-lab*
```

Private key nên có permission:

```bash
chmod 600 ~/.ssh/java-fresher-lab
```

Xem public key:

```bash
cat ~/.ssh/java-fresher-lab.pub
```

**Không bao giờ gửi cho tôi hoặc paste ra chat file:**

```text
~/.ssh/java-fresher-lab
```

Public key `.pub` thì không phải secret.

---

# 4. Lấy public IP thực tế của máy bạn

Đây là điểm nhiều người nhầm.

**Không dùng IP của WSL như `172.x.x.x`.**

AWS Security Group phải thấy **public egress IP của Internet connection**.

Trong WSL:

```bash
curl -4 https://checkip.amazonaws.com
```

Ví dụ:

```text
14.191.25.80
```

Thì Security Group source sẽ là:

```text
14.191.25.80/32
```

`/32` nghĩa là **chính xác một IPv4 address**.

Không dùng:

```text
0.0.0.0/0
```

cho SSH.

---

# 5. Import SSH public key vào EC2

Nếu AWS CLI trong WSL đã được cấu hình:

```bash
aws sts get-caller-identity
```

Bạn phải thấy account/ARN hợp lệ.

Import key:

```bash
aws ec2 import-key-pair \
  --key-name java-fresher-wsl-key \
  --public-key-material fileb://$HOME/.ssh/java-fresher-lab.pub \
  --region ap-southeast-1
```

AWS document chính thức cũng sử dụng `import-key-pair` theo cách này. :chatgpt-content-reference{index="9"}

Verify:

```bash
aws ec2 describe-key-pairs \
  --key-names java-fresher-wsl-key \
  --region ap-southeast-1
```

Điểm cần hiểu để giải thích trong exam:

> EC2 không giữ private key của tôi. Tôi giữ private key trong WSL. AWS chỉ lưu public key và inject public key đó vào instance lúc launch.

---

# 6. Tạo VPC — không dùng wizard tự động

Vào:

```text
AWS Console
→ VPC
→ Your VPCs
→ Create VPC
```

Tôi khuyên chọn:

```text
Resources to create:
VPC only
```

Không dùng `VPC and more` ở lần này.

Lý do: bạn đang học Administration. Bạn phải hiểu từng object được tạo ra.

Điền:

```text
Name:
java-fresher-vpc

IPv4 CIDR:
10.10.0.0/16

IPv6:
No IPv6 CIDR block

Tenancy:
Default
```

Create VPC.

---

# 7. CIDR `10.10.0.0/16` nghĩa là gì?

Đây là private IPv4 network.

Về mặt conceptual:

```text
10.10.0.0/16
```

bao phủ:

```text
10.10.0.0
...
10.10.255.255
```

Chúng ta lấy hai phần nhỏ hơn:

```text
10.10.10.0/24   Public subnet
10.10.20.0/24   Private subnet
```

Mỗi `/24` có 256 địa chỉ toán học, mặc dù AWS reserve một số địa chỉ trong mỗi subnet nên không phải toàn bộ đều assign cho resource.

---

# 8. Enable DNS settings cho VPC

Select:

```text
java-fresher-vpc
```

sau đó:

```text
Actions
→ Edit VPC settings
```

Đảm bảo bật:

```text
Enable DNS resolution
Enable DNS hostnames
```

Điều này hữu ích để EC2 có DNS behavior bình thường và public hostname khi phù hợp.

---

# 9. Tạo Public Subnet

Đi:

```text
VPC
→ Subnets
→ Create subnet
```

VPC:

```text
java-fresher-vpc
```

Name:

```text
java-fresher-public-a
```

Availability Zone:

```text
ap-southeast-1a
```

hoặc AZ đầu tiên hiện ra trong account bạn.

CIDR:

```text
10.10.10.0/24
```

Create.

---

# 10. Tạo Private Subnet

Tiếp tục:

```text
Name:
java-fresher-private-a

CIDR:
10.10.20.0/24
```

Có thể cùng AZ trong lab:

```text
ap-southeast-1a
```

Ta chưa deploy gì vào đây.

Mục đích là giúp bạn demo được khái niệm:

```text
Public subnet
vs
Private subnet
```

---

# 11. Enable auto-assign public IPv4 cho Public Subnet

Select:

```text
java-fresher-public-a
```

Đi:

```text
Actions
→ Edit subnet settings
```

Bật:

```text
Enable auto-assign public IPv4 address
```

Private subnet:

```text
java-fresher-private-a
```

**không bật**.

Điểm quan trọng:

> Subnet không trở thành "public subnet" chỉ vì tên của nó có chữ public. Nó phải có route tới Internet Gateway và instance muốn trực tiếp giao tiếp IPv4 với Internet còn cần public IPv4/EIP.

---

# 12. Tạo Internet Gateway

Đi:

```text
VPC
→ Internet Gateways
→ Create internet gateway
```

Name:

```text
java-fresher-igw
```

Create.

Lúc này IGW mới tồn tại nhưng **chưa thuộc VPC**.

Select IGW:

```text
Actions
→ Attach to a VPC
```

chọn:

```text
java-fresher-vpc
```

Attach.

---

# 13. Tạo Public Route Table

Đi:

```text
VPC
→ Route tables
→ Create route table
```

Name:

```text
java-fresher-rt-public
```

VPC:

```text
java-fresher-vpc
```

Create.

Bạn sẽ thấy route tự động:

```text
Destination       Target
10.10.0.0/16      local
```

Đây là route intra-VPC.

Không xóa.

---

# 14. Thêm default route tới Internet Gateway

Trong `java-fresher-rt-public`:

```text
Routes
→ Edit routes
→ Add route
```

Destination:

```text
0.0.0.0/0
```

Target:

```text
Internet Gateway
java-fresher-igw
```

Save.

Kết quả:

```text
10.10.0.0/16 -> local
0.0.0.0/0    -> java-fresher-igw
```

AWS xác định Internet-bound IPv4 traffic bằng route `0.0.0.0/0` tới IGW. :chatgpt-content-reference{index="10"}

---

# 15. Associate Public Route Table với Public Subnet

Trong:

```text
java-fresher-rt-public
```

chọn:

```text
Subnet associations
→ Edit subnet associations
```

Tick:

```text
java-fresher-public-a
```

Save.

Đây là bước rất quan trọng.

Mỗi subnet phải sử dụng một route table; nếu không associate explicit thì nó dùng main route table. :chatgpt-content-reference{index="11"}

---

# 16. Tạo Private Route Table

Create:

```text
Name:
java-fresher-rt-private

VPC:
java-fresher-vpc
```

Routes chỉ giữ:

```text
10.10.0.0/16 -> local
```

**Không thêm:**

```text
0.0.0.0/0 -> IGW
```

Associate nó với:

```text
java-fresher-private-a
```

Private subnet này hiện:

```text
cannot directly access Internet over IPv4
```

Chúng ta cũng **không tạo NAT Gateway** vì:

- hiện chưa cần;
- NAT Gateway có chi phí;
- làm lab phức tạp hơn;
- private subnet chỉ dùng để minh họa ở giai đoạn này.

---

# 17. Network ACL

VPC mới có default Network ACL.

Đối với lab ban đầu:

> **Không sửa NACL.**

Giữ default allow.

Ta sẽ dùng **Security Group làm primary filter**.

Sau này khi ôn Day 8 tôi sẽ bắt bạn giải thích:

```text
Security Group = stateful
NACL            = stateless
SG applies ENI/resource
NACL applies subnet
```

Nhưng hiện tại không cần thêm một firewall layer khiến troubleshooting khó hơn.

---

# 18. Tạo Security Group

Đi:

```text
EC2
→ Security Groups
→ Create security group
```

Name:

```text
java-fresher-ec2-sg
```

Description:

```text
Security group for Java Fresher integrated lab
```

VPC:

```text
java-fresher-vpc
```

Giả sử public IP của bạn là:

```text
14.191.25.80
```

thì inbound rules:

```text
SSH
TCP 22
14.191.25.80/32
```

```text
Custom TCP
TCP 8080
14.191.25.80/32
```

```text
Custom TCP
TCP 3000
14.191.25.80/32
```

```text
Custom TCP
TCP 9090
14.191.25.80/32
```

và để phục vụ `ping` troubleshooting:

```text
Echo Request - IPv4
ICMP
14.191.25.80/32
```

Không tạo:

```text
5432 from 0.0.0.0/0
```

Không tạo:

```text
9100 from 0.0.0.0/0
```

Outbound:

```text
All traffic
0.0.0.0/0
```

giữ mặc định cho lab.

---

# 19. Vì sao không expose PostgreSQL?

Application và PostgreSQL cùng EC2.

Java app có thể dùng:

```text
jdbc:postgresql://127.0.0.1:5432/appdb
```

Traffic không cần đi qua Internet.

Security Group port 5432 vì vậy **không cần inbound rule**.

Điều này vừa an toàn hơn vừa giúp exam explanation tốt:

> "DB không phải public-facing service nên em không expose PostgreSQL port ra Internet. Application chạy cùng instance nên dùng loopback/local connection."

---

# 20. Vì sao không expose node_exporter?

Prometheus cũng chạy cùng máy.

Prometheus scrape:

```text
127.0.0.1:9100
```

Do đó cũng không cần:

```text
SG inbound 9100
```

Prometheus UI `9090` thì chúng ta mở `/32` cho máy bạn để demo.

---

# 21. Launch EC2

Đi:

```text
EC2
→ Instances
→ Launch instances
```

Name:

```text
java-fresher-integrated-lab
```

AMI:

```text
Ubuntu Server 24.04 LTS
```

Architecture:

```text
64-bit (x86)
```

Không hard-code AMI ID vì AMI ID thay đổi theo Region và thời điểm.

---

# 22. Instance type

Chọn:

```text
t3.medium
```

Lý do:

```text
2 vCPU
4 GiB RAM
```

Một instance phải cùng lúc chạy:

```text
Ubuntu
Tomcat/JVM
PostgreSQL
Prometheus
node_exporter
Grafana
```

`t3.small` 2 GiB có thể chạy nhưng margin khá thấp, nhất là Prometheus + Grafana + JVM.

Cho demo ổn định tôi chọn:

```text
t3.medium
```

Không dùng Spot Instance cho exam lab.

---

# 23. Chọn Key Pair

Key pair:

```text
java-fresher-wsl-key
```

Đây chính là public key bạn import từ WSL.

Với Ubuntu EC2, default username là:

```text
ubuntu
```

AWS cũng xác nhận Ubuntu AMI dùng username `ubuntu`. :chatgpt-content-reference{index="12"}

---

# 24. Network settings

Chọn:

```text
VPC:
java-fresher-vpc
```

Subnet:

```text
java-fresher-public-a
```

Auto-assign public IP:

```text
Enable
```

Firewall:

```text
Select existing security group
```

chọn:

```text
java-fresher-ec2-sg
```

Hãy kiểm tra lại trước khi launch.

Không vô tình chọn default VPC.

---

# 25. Storage

Tôi đề xuất:

```text
30 GiB
gp3
Delete on termination: Yes
Encrypted: Yes
```

20 GiB cũng đủ cho lab nhỏ, nhưng 30 GiB giúp Prometheus/PostgreSQL có thêm margin.

Không chọn disk quá lớn không cần thiết.

---

# 26. Metadata security

Trong:

```text
Advanced details
```

nếu Console cung cấp Metadata options, chọn:

```text
IMDSv2 required
```

User data của chúng ta không cần gọi Instance Metadata.

Đây là security hardening hợp lý.

---

# 27. IAM role

Hiện tại:

```text
IAM instance profile:
None
```

User data dưới đây **không gọi AWS API**, vì vậy không cần attach quyền AWS.

Nếu user data có AWS CLI/API call thì phải dùng instance role thay vì nhét credentials vào script. AWS cũng khuyến cáo instance profile cho AWS API calls từ user data. :chatgpt-content-reference{index="13"}

---

# 28. User Data hoàn chỉnh

Đi:

```text
Advanced details
→ User data
```

Paste script sau.

**Không sửa từng phần ngẫu nhiên.**

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

LOG_FILE="/var/log/java-fresher-bootstrap.log"
exec > >(tee -a "$LOG_FILE") 2>&1
trap 'rc=$?; echo "[ERROR] bootstrap failed at line $LINENO (exit=$rc)"; exit $rc' ERR

echo "[INFO] bootstrap started: $(date -Is)"
export DEBIAN_FRONTEND=noninteractive

hostnamectl set-hostname java-fresher-lab

apt-get update
apt-get install -y \
  software-properties-common \
  ca-certificates \
  curl \
  wget \
  gnupg \
  jq \
  unzip \
  rsync \
  tar \
  netcat-openbsd \
  traceroute \
  openssl

add-apt-repository -y universe
apt-get update

apt-get install -y \
  openjdk-17-jdk-headless \
  tomcat10 \
  postgresql \
  postgresql-contrib \
  prometheus \
  prometheus-node-exporter

install -d -m 0755 /etc/apt/keyrings

wget -q \
  -O /etc/apt/keyrings/grafana.asc \
  https://apt.grafana.com/gpg-full.key

chmod 0644 /etc/apt/keyrings/grafana.asc

cat > /etc/apt/sources.list.d/grafana.list <<'GRAFANA_REPO'
deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main
GRAFANA_REPO

apt-get update
apt-get install -y grafana

install -d -m 0755 \
  /opt/java-fresher/evidence \
  /opt/java-fresher/backups \
  /opt/java-fresher/scripts

chown -R ubuntu:ubuntu /opt/java-fresher

systemctl enable \
  tomcat10 \
  postgresql \
  prometheus \
  prometheus-node-exporter \
  grafana-server

cp -a \
  /etc/prometheus/prometheus.yml \
  "/etc/prometheus/prometheus.yml.bak.$(date +%Y%m%d%H%M%S)"

cat > /etc/prometheus/prometheus.yml <<'PROMETHEUS_CONFIG'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['127.0.0.1:9090']

  - job_name: node
    static_configs:
      - targets: ['127.0.0.1:9100']
PROMETHEUS_CONFIG

promtool check config /etc/prometheus/prometheus.yml

cat > /etc/grafana/provisioning/datasources/prometheus.yml <<'GRAFANA_DATASOURCE'
apiVersion: 1

datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://127.0.0.1:9090
    isDefault: true
    editable: true
GRAFANA_DATASOURCE

chmod 0644 \
  /etc/grafana/provisioning/datasources/prometheus.yml

systemctl restart postgresql
systemctl restart tomcat10
systemctl restart prometheus-node-exporter
systemctl restart prometheus
systemctl restart grafana-server

sleep 5

systemctl is-active --quiet tomcat10
systemctl is-active --quiet postgresql
systemctl is-active --quiet prometheus-node-exporter
systemctl is-active --quiet prometheus
systemctl is-active --quiet grafana-server

java -version

pg_isready \
  -h 127.0.0.1 \
  -p 5432

curl -fsS http://127.0.0.1:8080/ >/dev/null
curl -fsS http://127.0.0.1:9090/-/healthy
curl -fsS http://127.0.0.1:9100/metrics >/dev/null
curl -fsS http://127.0.0.1:3000/api/health

ss -lntp \
  | tee /opt/java-fresher/evidence/bootstrap-listening-ports.txt

systemctl --failed --no-pager \
  | tee /opt/java-fresher/evidence/bootstrap-failed-units.txt

cat > /opt/java-fresher/evidence/bootstrap-summary.txt <<SUMMARY
Bootstrap completed: $(date -Is)
Hostname: $(hostname)
Java: $(java -version 2>&1 | head -n 1)
Tomcat: active on TCP/8080
PostgreSQL: ready on TCP/5432 (localhost only by default)
Prometheus: healthy on TCP/9090
node_exporter: metrics on TCP/9100
Grafana: healthy on TCP/3000
Prometheus config: /etc/prometheus/prometheus.yml
Grafana datasource: /etc/grafana/provisioning/datasources/prometheus.yml
Bootstrap log: $LOG_FILE
SUMMARY

chown ubuntu:ubuntu \
  /opt/java-fresher/evidence/bootstrap-*.txt

echo "[INFO] bootstrap completed successfully: $(date -Is)"
```

Tôi đã kiểm tra syntax Bash của script này bằng `bash -n`; script không có syntax error.

---

# 29. Giải thích kỹ từng phần User Data

## Dòng 1

```bash
#!/usr/bin/env bash
```

Shebang.

Nó nói với OS rằng script này phải chạy bằng Bash.

Không dùng:

```bash
#!/bin/sh
```

vì script của chúng ta có:

```bash
>(...)
```

là Bash process substitution.

---

# 30. Strict mode

```bash
set -Eeuo pipefail
```

Đây là một dòng rất quan trọng.

### `-e`

Nếu command trả exit code khác `0`, script dừng thay vì tiếp tục giả vờ mọi thứ ổn.

Ví dụ:

```bash
apt-get install ...
```

fail thì bootstrap dừng.

Đây là hành vi rất tốt cho automation.

---

### `-E`

Giữ `ERR trap` hoạt động trong nhiều context/function/subshell thích hợp.

---

### `-u`

Reference một variable chưa tồn tại → error.

Tránh typo variable bị biến thành empty string.

---

### `-o pipefail`

Ví dụ:

```bash
command1 | command2
```

nếu `command1` fail nhưng `command2` success thì pipeline bình thường có thể che mất lỗi.

`pipefail` giúp pipeline được đánh dấu failure.

---

# 31. File log bootstrap

```bash
LOG_FILE="/var/log/java-fresher-bootstrap.log"
```

Tạo biến:

```text
LOG_FILE
```

trỏ tới:

```text
/var/log/java-fresher-bootstrap.log
```

File này sẽ là evidence cực kỳ hữu ích.

---

# 32. Redirect stdout/stderr

```bash
exec > >(tee -a "$LOG_FILE") 2>&1
```

Đây là dòng hơi nâng cao.

`exec` ở đây thay đổi destination của output cho **toàn bộ phần còn lại của script**.

```bash
tee -a "$LOG_FILE"
```

nghĩa là:

- hiển thị output;
- đồng thời append vào log.

`2>&1`:

```text
stderr -> stdout
```

nên cả lỗi và normal output đều được ghi.

Bạn còn có log mặc định cloud-init:

```text
/var/log/cloud-init-output.log
```

Nên khi bootstrap fail bạn có **hai nguồn evidence**.

---

# 33. ERR trap

```bash
trap 'rc=$?; echo "[ERROR] bootstrap failed at line $LINENO (exit=$rc)"; exit $rc' ERR
```

Nếu command fail:

```text
$?
```

là exit code.

```text
$LINENO
```

là line number.

Ví dụ log có thể hiện:

```text
[ERROR] bootstrap failed at line 47 (exit=100)
```

Rất hữu ích khi troubleshoot.

---

# 34. Start timestamp

```bash
echo "[INFO] bootstrap started: $(date -Is)"
```

`date -Is` tạo ISO timestamp.

Ví dụ:

```text
2026-10-06T01:30:00+00:00
```

Đây là evidence WHEN.

---

# 35. Noninteractive APT

```bash
export DEBIAN_FRONTEND=noninteractive
```

Vì user data chạy tự động, không có người ngồi trả lời prompt.

Nếu package hỏi interactive question thì bootstrap có thể bị treo.

Do đó ta yêu cầu Debian/Ubuntu package system hoạt động noninteractive.

---

# 36. Hostname

```bash
hostnamectl set-hostname java-fresher-lab
```

Đặt hostname Linux:

```text
java-fresher-lab
```

Sau SSH bạn sẽ dễ nhận biết:

```text
ubuntu@java-fresher-lab
```

---

# 37. Refresh package metadata

```bash
apt-get update
```

Không install software ngay từ stale package cache.

`apt-get update` tải metadata package/repository.

Nó **không phải**:

```bash
apt-get upgrade
```

Hai command có mục đích khác nhau.

---

# 38. Install utility packages

```bash
apt-get install -y \
  software-properties-common \
  ca-certificates \
  curl \
  wget \
  gnupg \
  jq \
  unzip \
  rsync \
  tar \
  netcat-openbsd \
  traceroute \
  openssl
```

`-y`:

```text
automatically answer yes
```

Các package:

```text
software-properties-common
```

cho utility như `add-apt-repository`.

```text
ca-certificates
```

CA trust store phục vụ HTTPS/TLS.

```text
curl
```

HTTP/API/endpoint validation.

```text
wget
```

download file.

```text
gnupg
```

GPG repository trust.

```text
jq
```

parse JSON.

```text
unzip
```

archive utility.

```text
rsync
tar
```

phục vụ backup lab Day 6.

```text
netcat-openbsd
```

cung cấp `nc`, dùng test port:

```bash
nc -zv host 8080
```

```text
traceroute
```

network troubleshooting.

```text
openssl
```

TLS/certificate troubleshooting:

```bash
openssl s_client ...
```

---

# 39. Enable Universe

```bash
add-apt-repository -y universe
```

Tomcat/Prometheus packages liên quan có thể nằm trong Ubuntu `universe`.

Sau khi repository thay đổi:

```bash
apt-get update
```

lại lần nữa.

---

# 40. Install Java/Tomcat/PostgreSQL/Monitoring

```bash
apt-get install -y \
  openjdk-17-jdk-headless \
  tomcat10 \
  postgresql \
  postgresql-contrib \
  prometheus \
  prometheus-node-exporter
```

### Java

```text
openjdk-17-jdk-headless
```

Java 17 runtime + development tools không cần GUI.

Kiểm tra sau:

```bash
java -version
```

---

### Tomcat

```text
tomcat10
```

Ubuntu package tạo systemd service:

```text
tomcat10.service
```

Ubuntu package thực sự cung cấp systemd unit này. :chatgpt-content-reference{index="14"}

Các path quan trọng:

```text
/etc/tomcat10
/var/lib/tomcat10
/usr/share/tomcat10
```

---

### PostgreSQL

```text
postgresql
```

meta package PostgreSQL.

Ubuntu 24.04 hiện cung cấp PostgreSQL 16. :chatgpt-content-reference{index="15"}

```text
postgresql-contrib
```

cung cấp một số extension/additional utilities.

---

### Prometheus

```text
prometheus
```

Prometheus server.

Ubuntu 24.04 hiện có Prometheus package. :chatgpt-content-reference{index="16"}

---

### node_exporter

```text
prometheus-node-exporter
```

Host metrics exporter.

Ubuntu hiện có package này cho Noble. :chatgpt-content-reference{index="17"}

---

# 41. Chuẩn bị Grafana repository key directory

```bash
install -d -m 0755 /etc/apt/keyrings
```

`install -d`:

```text
create directory
```

`-m 0755`:

```text
rwxr-xr-x
```

---

# 42. Download Grafana signing key

```bash
wget -q \
  -O /etc/apt/keyrings/grafana.asc \
  https://apt.grafana.com/gpg-full.key
```

`-q`:

```text
quiet
```

`-O`:

```text
write output to this exact path
```

Grafana official docs dùng chính repository/key approach này. :chatgpt-content-reference{index="18"}

---

# 43. Key permission

```bash
chmod 0644 /etc/apt/keyrings/grafana.asc
```

Permission:

```text
owner: rw-
group: r--
others: r--
```

APT phải đọc được key.

---

# 44. Tạo Grafana APT repository

```bash
cat > /etc/apt/sources.list.d/grafana.list <<'GRAFANA_REPO'
deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main
GRAFANA_REPO
```

Đây là **heredoc**.

Nó tạo:

```text
/etc/apt/sources.list.d/grafana.list
```

với:

```text
https://apt.grafana.com stable main
```

`signed-by=` giới hạn repository này dùng key cụ thể:

```text
/etc/apt/keyrings/grafana.asc
```

Tốt hơn cách legacy `apt-key`.

---

# 45. Cài Grafana

Repository mới → refresh:

```bash
apt-get update
```

Sau đó:

```bash
apt-get install -y grafana
```

Grafana official Debian/Ubuntu installation cũng hỗ trợ `apt-get install grafana`. :chatgpt-content-reference{index="19"}

---

# 46. Evidence directories

```bash
install -d -m 0755 \
  /opt/java-fresher/evidence \
  /opt/java-fresher/backups \
  /opt/java-fresher/scripts
```

Ta tạo baseline:

```text
/opt/java-fresher/
├── evidence/
├── backups/
└── scripts/
```

Đây rất hữu ích cho practical demo.

---

# 47. Ownership

```bash
chown -R ubuntu:ubuntu /opt/java-fresher
```

Ubuntu SSH user có thể lưu evidence mà không cần `sudo` mọi lần.

---

# 48. Enable services at boot

```bash
systemctl enable \
  tomcat10 \
  postgresql \
  prometheus \
  prometheus-node-exporter \
  grafana-server
```

`enable` nghĩa là:

> cấu hình service start tự động khi machine boot.

Không đồng nghĩa hoàn toàn với:

```bash
systemctl start
```

`start` = start now.

`enable` = start at future boot.

Sau đó chúng ta `restart`, nên chúng cũng sẽ được start ngay.

Grafana official docs vẫn sử dụng systemd service:

````text
grafana-server
``` :chatgpt-content-reference{index="20"}


---

# 49. Backup Prometheus config trước khi sửa

```bash
cp -a \
  /etc/prometheus/prometheus.yml \
  "/etc/prometheus/prometheus.yml.bak.$(date +%Y%m%d%H%M%S)"
````

Đây là thói quen cực kỳ quan trọng cho exam.

**BEFORE → ACTION → AFTER → RESULT**

Trước khi sửa config:

```text
backup
```

Ví dụ:

```text
prometheus.yml.bak.20261006013000
```

`-a` giữ metadata tốt hơn simple copy.

---

# 50. Prometheus config

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
```

Prometheus scrape target mỗi:

```text
15 seconds
```

và evaluate rules mỗi:

```text
15 seconds
```

---

# 51. Prometheus tự monitor chính nó

```yaml
- job_name: prometheus
```

Target:

```yaml
targets: ["127.0.0.1:9090"]
```

Prometheus server tự expose metrics của chính nó.

---

# 52. node_exporter target

```yaml
- job_name: node
```

Target:

```text
127.0.0.1:9100
```

node_exporter chạy cùng EC2.

Prometheus không cần public IP để scrape nó.

---

# 53. Validate configuration trước restart

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Đây là một trong những dòng quan trọng nhất.

Quy trình đúng:

```text
backup
→ edit
→ syntax/config validation
→ restart
→ runtime validation
```

Không phải:

```text
edit
→ restart
→ pray
```

Nếu YAML lỗi, `promtool` trả non-zero và do:

```bash
set -e
```

bootstrap dừng.

---

# 54. Provision Grafana Prometheus datasource

File:

```text
/etc/grafana/provisioning/datasources/prometheus.yml
```

Config:

```yaml
name: Prometheus
```

Tên datasource.

```yaml
uid: prometheus
```

stable datasource UID.

```yaml
type: prometheus
```

Grafana hiểu datasource này dùng Prometheus driver.

```yaml
access: proxy
```

Grafana backend kết nối datasource.

```yaml
url: http://127.0.0.1:9090
```

Điểm cực kỳ hay:

Grafana gọi Prometheus **locally**, không đi:

```text
browser → Internet → public Prometheus
```

mà:

```text
Grafana process
    ↓
127.0.0.1:9090
    ↓
Prometheus
```

```yaml
isDefault: true
```

Datasource mặc định.

```yaml
editable: true
```

Bạn vẫn có thể chỉnh trong Grafana UI.

---

# 55. Restart services

```bash
systemctl restart postgresql
```

restart PostgreSQL.

```bash
systemctl restart tomcat10
```

restart Tomcat.

```bash
systemctl restart prometheus-node-exporter
```

start/restart node_exporter.

```bash
systemctl restart prometheus
```

apply `prometheus.yml`.

```bash
systemctl restart grafana-server
```

apply Grafana datasource provisioning.

---

# 56. Vì sao restart Prometheus sau node_exporter?

Thứ tự:

```text
node_exporter
→ Prometheus
→ Grafana
```

có logic dependency:

```text
node_exporter produces metrics
Prometheus scrapes metrics
Grafana queries Prometheus
```

---

# 57. `sleep 5`

```bash
sleep 5
```

Cho các process một khoảng ngắn để bind socket/initialize trước health checks.

Không phải health-check mechanism hoàn hảo cho production, nhưng đủ cho lab bootstrap.

---

# 58. Validate systemd services

Ví dụ:

```bash
systemctl is-active --quiet tomcat10
```

Nếu active:

```text
exit 0
```

Nếu không active:

```text
non-zero
```

Do `set -e`, bất kỳ service nào fail thì bootstrap cũng fail.

Chúng ta kiểm tra:

```text
tomcat10
postgresql
prometheus-node-exporter
prometheus
grafana-server
```

---

# 59. Java validation

```bash
java -version
```

Xác minh Java thực sự executable.

Đây không chỉ là:

```text
package installed
```

mà:

```text
runtime executable works
```

---

# 60. PostgreSQL health

```bash
pg_isready \
  -h 127.0.0.1 \
  -p 5432
```

Test:

```text
127.0.0.1:5432
```

PostgreSQL có đang accept connections hay không.

---

# 61. Tomcat validation

```bash
curl -fsS http://127.0.0.1:8080/ >/dev/null
```

Flags:

```text
-f
```

fail trên HTTP error.

```text
-s
```

silent.

```text
-S
```

nhưng nếu lỗi thì vẫn show error.

Output redirect:

```text
>/dev/null
```

vì ta chỉ quan tâm exit status.

---

# 62. Prometheus health endpoint

```bash
curl -fsS http://127.0.0.1:9090/-/healthy
```

Expected:

Prometheus trả health success.

---

# 63. node_exporter validation

```bash
curl -fsS http://127.0.0.1:9100/metrics >/dev/null
```

Nếu metrics endpoint không hoạt động bootstrap fail.

---

# 64. Grafana validation

```bash
curl -fsS http://127.0.0.1:3000/api/health
```

Kiểm tra Grafana backend.

---

# 65. Capture listening sockets

```bash
ss -lntp \
  | tee /opt/java-fresher/evidence/bootstrap-listening-ports.txt
```

Các flags:

```text
-l = listening
-n = numeric
-t = TCP
-p = process
```

File evidence:

```text
bootstrap-listening-ports.txt
```

Bạn sẽ dùng được trong demo.

---

# 66. Capture failed units

```bash
systemctl --failed --no-pager \
  | tee /opt/java-fresher/evidence/bootstrap-failed-units.txt
```

Nếu tất cả tốt:

thường không có failed service.

Đây là baseline evidence AFTER.

---

# 67. Summary evidence

Script cuối cùng tạo:

```text
/opt/java-fresher/evidence/bootstrap-summary.txt
```

Chứa:

```text
timestamp
hostname
Java
Tomcat
PostgreSQL
Prometheus
node_exporter
Grafana
config paths
bootstrap log path
```

---

# 68. Vì sao User Data không có `sudo`?

Đây là câu hỏi exam rất hay.

EC2 Linux user data/cloud-init shell script chạy với privilege của root trong bootstrap context.

Vì vậy:

```bash
apt-get install
systemctl restart
cp /etc/...
```

không cần:

```bash
sudo
```

AWS cũng mô tả user-data shell scripts là mechanism thực hiện bootstrap trong initial launch; mặc định user data thường chạy ở initial boot chứ không tự chạy mỗi reboot. :chatgpt-content-reference{index="21"}

---

# 69. Vì sao không đặt password PostgreSQL trong User Data?

Đây là security decision có chủ ý.

Tôi **không** viết kiểu:

```bash
DB_PASSWORD="Password123"
```

trong user data.

Vì credentials không nên nằm trong bootstrap configuration/history một cách không cần thiết.

PostgreSQL được **install + start + validate**.

Sau đó ở phần Day 15 chúng ta sẽ tự:

```text
CREATE ROLE
CREATE DATABASE
GRANT
pg_hba.conf
password/secrets handling
```

đúng theo practical requirement.

---

# 70. Launch instance

Sau khi paste User Data:

```text
Launch instance
```

Sau khi instance state thành:

```text
Running
```

**đừng kết luận ngay rằng bootstrap đã xong.**

EC2 `Running` chỉ nói VM đang chạy.

User data/package installation có thể vẫn đang thực thi.

---

# 71. SSH lần đầu từ WSL

Lấy EC2:

```text
Public IPv4 address
```

ví dụ:

```text
18.141.10.25
```

Trong WSL:

```bash
ssh -i ~/.ssh/java-fresher-lab \
  ubuntu@18.141.10.25
```

AWS xác nhận Ubuntu AMI dùng username `ubuntu`. :chatgpt-content-reference{index="22"}

Lần đầu SSH sẽ hỏi host fingerprint:

```text
Are you sure you want to continue connecting?
```

Đừng có thói quen cứ gõ `yes` vô thức trong production.

Trong lab bạn có thể kiểm tra instance/IP rồi mới accept.

---

# 72. Nếu muốn dùng SSH config

Trong WSL:

```bash
nano ~/.ssh/config
```

Thêm:

```text
Host java-fresher-lab
    HostName 18.141.10.25
    User ubuntu
    IdentityFile ~/.ssh/java-fresher-lab
    IdentitiesOnly yes
```

Permission:

```bash
chmod 600 ~/.ssh/config
```

Sau đó:

```bash
ssh java-fresher-lab
```

rất tiện.

Lưu ý:

> auto-assigned public IPv4 có thể thay đổi sau stop/start. Khi đó phải cập nhật `HostName`.

---

# 73. Kiểm tra cloud-init

Ngay sau SSH:

```bash
cloud-init status
```

Hoặc:

```bash
cloud-init status --wait
```

Expected invariant:

```text
status: done
```

Nếu:

```text
running
```

thì user data chưa hoàn thành.

---

# 74. Kiểm tra bootstrap log

```bash
sudo less /var/log/java-fresher-bootstrap.log
```

Và:

```bash
sudo less /var/log/cloud-init-output.log
```

Tìm error:

```bash
sudo grep -iE 'error|failed|failure' \
  /var/log/java-fresher-bootstrap.log
```

Nhưng nhớ:

`grep "error"` không tự động có nghĩa hệ thống hỏng. Phải đọc context.

---

# 75. Xem summary

```bash
cat /opt/java-fresher/evidence/bootstrap-summary.txt
```

Expected dạng:

```text
Bootstrap completed: ...
Hostname: java-fresher-lab
Java: openjdk version "17..."
Tomcat: active on TCP/8080
PostgreSQL: ready on TCP/5432 ...
Prometheus: healthy on TCP/9090
node_exporter: metrics on TCP/9100
Grafana: healthy on TCP/3000
...
```

---

# 76. Kiểm tra OS

```bash
cat /etc/os-release
```

```bash
hostnamectl
```

```bash
uname -a
```

---

# 77. Kiểm tra Java

```bash
java -version
```

```bash
which java
```

Có thể thêm:

```bash
readlink -f "$(which java)"
```

---

# 78. Kiểm tra Tomcat

```bash
systemctl status tomcat10 --no-pager
```

Port:

```bash
sudo ss -lntp | grep ':8080'
```

Local curl:

```bash
curl -v http://127.0.0.1:8080/
```

Logs:

```bash
sudo journalctl -u tomcat10 -n 50 --no-pager
```

Ubuntu Tomcat package có systemd service `tomcat10`. :chatgpt-content-reference{index="23"}

---

# 79. Kiểm tra PostgreSQL

```bash
systemctl status postgresql --no-pager
```

Cluster:

```bash
pg_lsclusters
```

Expected gần giống:

```text
Ver Cluster Port Status Owner    Data directory
16  main    5432 online postgres ...
```

Health:

```bash
pg_isready
```

Port:

```bash
sudo ss -lntp | grep ':5432'
```

Bạn đặc biệt cần quan sát PostgreSQL bind address.

Lý tưởng baseline:

```text
127.0.0.1:5432
```

không phải expose Internet.

---

# 80. Kiểm tra Prometheus

```bash
systemctl status prometheus --no-pager
```

```bash
sudo ss -lntp | grep ':9090'
```

```bash
curl http://127.0.0.1:9090/-/healthy
```

Validate config một lần nữa:

```bash
sudo promtool check config /etc/prometheus/prometheus.yml
```

---

# 81. Kiểm tra node_exporter

```bash
systemctl status prometheus-node-exporter --no-pager
```

```bash
sudo ss -lntp | grep ':9100'
```

```bash
curl -s http://127.0.0.1:9100/metrics | head
```

Expected sẽ thấy metric dạng:

```text
node_...
```

---

# 82. Kiểm tra Prometheus đã scrape node_exporter chưa

Từ EC2:

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=up' \
  | jq
```

Bạn muốn thấy target:

```text
job="prometheus"
```

và:

```text
job="node"
```

với:

```text
value ... "1"
```

`up = 1` nghĩa là scrape thành công.

---

# 83. Kiểm tra Grafana

```bash
systemctl status grafana-server --no-pager
```

Port:

```bash
sudo ss -lntp | grep ':3000'
```

Health:

```bash
curl http://127.0.0.1:3000/api/health
```

Grafana official docs dùng port mặc định:

````text
3000
``` :chatgpt-content-reference{index="24"}


---

# 84. Grafana lần đăng nhập đầu

Trên browser Windows:

```text
http://PUBLIC_IP:3000
````

Default initial credentials thường là:

```text
username: admin
password: admin
```

Grafana yêu cầu đổi password sau đăng nhập đầu và documentation cũng khuyến cáo đổi default password. :chatgpt-content-reference{index="25"}

**Đổi ngay.**

Không dùng `admin/admin` làm lâu dài.

---

# 85. Current Grafana note

Hiện Grafana 13 đã loại bỏ legacy executables:

```text
grafana-cli
grafana-server
```

ở CLI binary level và khuyến nghị:

```bash
grafana cli ...
grafana server ...
```

Tuy nhiên package systemd service vẫn tên:

```text
grafana-server.service
```

Đây là hai khái niệm khác nhau. :chatgpt-content-reference{index="26"}

Điểm này rất dễ nhầm.

---

# 86. Kiểm tra tất cả port một lần

```bash
sudo ss -lntp
```

Bạn cần hiểu expected state:

```text
22    SSH
5432  PostgreSQL
8080  Tomcat
9090  Prometheus
9100  node_exporter
3000  Grafana
```

Không chỉ nhìn thấy rồi nói "OK".

Phải map:

```text
port → process → service → function
```

---

# 87. Test từ WSL — rất quan trọng

Giả sử:

```bash
EC2_IP=18.141.10.25
```

SSH:

```bash
ssh -i ~/.ssh/java-fresher-lab ubuntu@$EC2_IP
```

Test TCP SSH:

```bash
nc -zv $EC2_IP 22
```

Tomcat:

```bash
curl -v http://$EC2_IP:8080/
```

Prometheus:

```bash
curl -v http://$EC2_IP:9090/-/healthy
```

Grafana:

```bash
curl -v http://$EC2_IP:3000/api/health
```

---

# 88. PostgreSQL phải FAIL từ WSL

Đây lại là một **expected failure rất tốt**.

```bash
nc -zv $EC2_IP 5432
```

Bạn **không muốn** remote Internet access tới PostgreSQL.

Nếu timeout:

```text
đây có thể là expected security behavior
```

Không phải mọi timeout đều là incident.

---

# 89. node_exporter cũng nên fail từ WSL

```bash
nc -zv $EC2_IP 9100
```

Ta không mở Security Group 9100.

Nhưng trên EC2:

```bash
curl http://127.0.0.1:9100/metrics
```

phải thành công.

Điều này chứng minh:

```text
service healthy
+
external exposure intentionally blocked
```

Đây là một demonstration security rất tốt.

---

# 90. Test browser

Từ Windows browser:

Tomcat:

```text
http://PUBLIC_IP:8080/
```

Prometheus:

```text
http://PUBLIC_IP:9090/
```

Grafana:

```text
http://PUBLIC_IP:3000/
```

Chỉ máy có public IP được allow trong Security Group mới truy cập được.

---

# 91. Verify route architecture trong EC2

Trên instance:

```bash
ip addr
```

Bạn sẽ thấy **private IP**, ví dụ:

```text
10.10.10.25
```

Điều này rất quan trọng:

EC2 OS thường không thấy public IPv4 gắn trực tiếp như một normal interface address.

AWS thực hiện mapping public/private ở infrastructure layer.

Route:

```bash
ip route
```

Expected đại loại:

```text
default via 10.10.10.1 ...
10.10.10.0/24 dev ...
```

---

# 92. Test Internet outbound từ EC2

```bash
curl -I https://aws.amazon.com
```

Hoặc:

```bash
curl -4 https://checkip.amazonaws.com
```

Nếu thành công:

```text
EC2
→ subnet route
→ IGW
→ Internet
```

đang hoạt động.

---

# 93. OSI Mindset nếu SSH fail

Ví dụ:

```bash
ssh ubuntu@$EC2_IP
```

timeout.

**Không restart EC2 ngay.**

Dùng OSI mindset.

### Layer 1 / infrastructure

AWS Console:

```text
Instance State = Running?
Status checks = passed?
```

### Layer 3

Kiểm tra:

```text
EC2 nằm đúng VPC?
đúng public subnet?
public IPv4 có tồn tại?
```

Route table:

```text
0.0.0.0/0 → IGW?
```

IGW:

```text
Attached?
```

### Layer 4

Security Group:

```text
TCP/22 from MY_PUBLIC_IP/32?
```

Public IP máy bạn có đổi không?

```bash
curl -4 https://checkip.amazonaws.com
```

### Application/SSH

Nếu network tới được nhưng auth fail:

```bash
ssh -vvv \
  -i ~/.ssh/java-fresher-lab \
  ubuntu@$EC2_IP
```

AWS troubleshooting docs cũng nhấn mạnh phải kiểm tra đúng username, đúng key và inbound SG cho SSH. :chatgpt-content-reference{index="27"}

---

# 94. Nếu `Permission denied (publickey)`

Khác hoàn toàn timeout.

```text
Connection timed out
```

nghi:

```text
network / route / SG / IP
```

Trong khi:

```text
Permission denied (publickey)
```

nghĩa là bạn đã tới SSH server rồi.

Layer 3/4 gần như đã hoạt động.

Kiểm tra:

```text
wrong username?
wrong private key?
key không match public key?
```

Ubuntu username:

```text
ubuntu
```

không phải:

```text
ec2-user
```

---

# 95. Nếu User Data fail

Đừng terminate ngay.

Evidence:

```bash
cloud-init status
```

```bash
sudo tail -n 200 /var/log/cloud-init-output.log
```

```bash
sudo tail -n 200 /var/log/java-fresher-bootstrap.log
```

```bash
sudo systemctl --failed
```

```bash
sudo journalctl -p err -b --no-pager
```

Sau đó xác định bước fail.

Ví dụ:

```text
APT repository
DNS
Internet route
package unavailable
Grafana repository key
Prometheus config
service startup
```

Đây mới là troubleshooting có evidence.

---

# 96. Không rerun toàn bộ User Data bừa bãi

AWS user data mặc định chủ yếu chạy trong initial launch boot cycle; nó không phải script bạn mặc định mong sẽ chạy lại mỗi reboot. :chatgpt-content-reference{index="28"}

Nếu fail giữa chừng, trước tiên:

```text
inspect
→ identify failed command
→ fix exact root cause
→ rerun only appropriate safe step
```

Đừng:

```bash
sudo bash user-data.sh
```

một cách mù quáng.

Vì script bootstrap có thể:

```text
rewrite config
restart service
change current state
```

---

# 97. Evidence bạn nên chụp ngay sau setup

Vì demo guide đặc biệt coi trọng evidence, hãy lưu ít nhất:

```text
01-vpc.png
02-subnets.png
03-route-public.png
04-route-private.png
05-igw.png
06-security-group.png
07-ec2-details.png
08-cloud-init-status.txt
09-ss-listening.txt
10-systemctl-services.txt
11-prometheus-targets.png
12-grafana-health.png
13-tomcat.png
14-postgresql-health.txt
```

Ngoài screenshot, lưu command output.

Ví dụ:

```bash
mkdir -p ~/exam-evidence/bootstrap
```

```bash
ip addr > ~/exam-evidence/bootstrap/ip-addr.txt
```

```bash
ip route > ~/exam-evidence/bootstrap/ip-route.txt
```

```bash
sudo ss -lntp \
  > ~/exam-evidence/bootstrap/ss-lntp.txt
```

```bash
systemctl --failed \
  > ~/exam-evidence/bootstrap/systemctl-failed.txt
```

```bash
java -version \
  > ~/exam-evidence/bootstrap/java-version.txt 2>&1
```

---

# 98. Ghi history

Demo guide bắt buộc bạn giải thích được history của chính mình. :chatgpt-content-reference{index="29"}

Sau buổi setup:

```bash
history > ~/exam-evidence/bootstrap/history.log
```

Đừng chỉ lưu history.

Bạn phải hiểu:

```text
command
option
input
output
side effect
verification
rollback
```

của từng command.

---

# 99. Kiểm tra environment cuối cùng

Tôi muốn bạn chạy nguyên block này sau khi bootstrap hoàn thành:

```bash
echo '=== HOST ==='
hostnamectl

echo '=== OS ==='
cat /etc/os-release

echo '=== JAVA ==='
java -version

echo '=== FAILED SERVICES ==='
sudo systemctl --failed

echo '=== TOMCAT ==='
systemctl is-active tomcat10
curl -I http://127.0.0.1:8080/

echo '=== POSTGRESQL ==='
systemctl is-active postgresql
pg_isready

echo '=== PROMETHEUS ==='
systemctl is-active prometheus
curl http://127.0.0.1:9090/-/healthy

echo '=== NODE EXPORTER ==='
systemctl is-active prometheus-node-exporter
curl -s http://127.0.0.1:9100/metrics | head

echo '=== GRAFANA ==='
systemctl is-active grafana-server
curl http://127.0.0.1:3000/api/health

echo '=== SOCKETS ==='
sudo ss -lntp

echo '=== DISK ==='
df -h

echo '=== MEMORY ==='
free -h
```

Đây sẽ là baseline health-check rất tốt trước khi chúng ta bắt đầu cấu hình application.

---

# 100. Expected invariant cuối cùng

Đừng học thuộc exact output. Hãy nhớ **invariants**:

```text
EC2 = Running
Ubuntu = reachable by SSH from WSL

Public subnet:
0.0.0.0/0 → IGW

Private subnet:
no Internet default route

Security Group:
22/8080/3000/9090 only from your public /32

Java:
17 installed

Tomcat:
active
TCP/8080 reachable locally and from your allowed IP

PostgreSQL:
active
TCP/5432 available locally
NOT publicly exposed

node_exporter:
active
TCP/9100 locally reachable
NOT publicly exposed

Prometheus:
active
TCP/9090
job prometheus = UP
job node = UP

Grafana:
active
TCP/3000
Prometheus datasource provisioned

systemctl --failed:
no unexpected failed unit
```

---

# 101. Cost-control — bắt buộc cho syllabus

Sau mỗi buổi lab:

```text
EC2
→ Instance state
→ Stop instance
```

Không để instance chạy qua đêm nếu không cần.

Hiện AWS tính phí public IPv4 ở mức **$0.005/IP/hour** cho cả public IPv4 đang dùng và idle public IPv4 theo pricing hiện hành. :chatgpt-content-reference{index="30"}

Với auto-assigned public IPv4:

- stop EC2 → IP thường được release;
- start lại → thường nhận public IP mới;
- vì vậy SSH config có thể cần update.

EBS vẫn có thể tiếp tục phát sinh storage charge khi instance stopped.

Không allocate Elastic IP chỉ để tránh đổi IP trừ khi thực sự cần; idle public IPv4 cũng bị tính phí. :chatgpt-content-reference{index="31"}

---

## Baseline chúng ta vừa thiết kế

Sau bước này bạn sẽ có đúng nền tảng cần cho practical:

```text
AWS networking
├── VPC
├── Public subnet
├── Private subnet
├── IGW
├── Public route table
├── Private route table
└── Security Group

EC2 Ubuntu
├── SSH từ WSL
├── Java 17
├── Tomcat 10 :8080
├── PostgreSQL :5432 local
├── Prometheus :9090
├── node_exporter :9100 local
└── Grafana :3000
```

Và quan trọng hơn, cấu hình này giúp bạn **giải thích được architecture**, thay vì chỉ có một EC2 chứa một đống service mà không biết traffic đi như thế nào.

**Bước hợp lý tiếp theo sau khi bạn dựng xong baseline này** sẽ là cấu hình PostgreSQL `appdb/appuser` theo least privilege → cấu hình Tomcat/JVM → deploy một Java WAR → externalized DB configuration → validate end-to-end → sau đó mới tạo Grafana dashboard/alert và failure injection. Đó cũng chính là flow sát practical Day 12–17 nhất.
