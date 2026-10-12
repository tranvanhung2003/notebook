# Day 19 — Integrated Project Build: Linux + AWS + Java + PostgreSQL + Monitoring

## 1. Syllabus Alignment

Theo MASTER_SYLLABUS, **Day 19** là ngày tích hợp toàn bộ các khối kiến thức đã học từ Day 1 đến Day 18 thành một hệ thống có thể **build → configure → deploy → validate → monitor → document → assess risk/cost**.

Phạm vi bắt buộc của Day 19 là:

> **Concept/Lecture:** Integrated project build: Linux + AWS + Java + PostgreSQL + monitoring, configuration baseline, security and cost checks.  
> **Assignment/Lab:** Build the end-to-end lab from the approved build sheet and submit architecture, configuration inventory, and cost/risk checklist. 00_MASTER_SYLLABUS_Java_Fresher…

Đây là **mandatory scope**. Syllabus xác định tài liệu này là nguồn curriculum chính thức và không được bỏ hoặc tự ý thay thế yêu cầu. 00_MASTER_SYLLABUS_Java_Fresher…

Một điểm cần làm rõ: MASTER_SYLLABUS chỉ nhắc đến **“approved build sheet”** nhưng file hiện tại không chứa nội dung chi tiết của build sheet đó. Vì vậy, phần kiến trúc/lệnh cụ thể dưới đây được mình đánh dấu là **Supplementary / Reference Build Sheet**. Khi bạn có build sheet thực tế của khóa học hoặc công ty, build sheet đó phải thắng các giả định lab của mình.

---

# 2. Day 19 thực chất kiểm tra điều gì?

Day 19 **không phải một ngày học thêm một sản phẩm mới**.

Nó kiểm tra xem bạn có thể ghép những thứ đã học thành một service hoàn chỉnh hay không:

```text
User / Client
      |
      v
AWS Network
VPC → Subnet → Route → Security Group
      |
      v
EC2 / Linux
systemd + users + permissions + disk + network
      |
      v
Java / Tomcat
JVM → Tomcat → WAR → external configuration
      |
      v
PostgreSQL
role → database → authentication → query
      |
      v
Application response

Parallel observability path:

Linux host --------> node_exporter -----\
                                         \
Tomcat/App -------------------------------> Prometheus ---> Grafana
                                         /
Service health --------------------------/
```

Một fresher chưa đạt Day 19 nếu chỉ có thể nói:

> “EC2 running, Tomcat running, PostgreSQL running, Prometheus running.”

Bạn cần chứng minh:

```text
Client request
    ↓
AWS allows traffic
    ↓
Linux receives TCP connection
    ↓
Tomcat is LISTENING
    ↓
Java application is deployed
    ↓
Application reads correct environment configuration
    ↓
Application authenticates to PostgreSQL
    ↓
Database answers correctly
    ↓
Application returns expected response
    ↓
Metrics show system health
    ↓
Grafana presents useful operational state
```

Đây chính là tư duy **end-to-end administration**.

---

# 3. Mục tiêu học tập Day 19

Sau Day 19, bạn phải có khả năng nhận một build sheet và biến nó thành một môi trường vận hành được mà không cấu hình ngẫu hứng.

Bạn phải phân biệt được bốn trạng thái:

| Trạng thái    | Ý nghĩa                                   |
| ------------- | ----------------------------------------- |
| Desired state | Build sheet yêu cầu gì                    |
| Actual state  | Máy thực tế đang cấu hình gì              |
| Evidence      | Bạn chứng minh actual state bằng cách nào |
| Drift         | Actual state khác desired state ở đâu     |

Ví dụ:

| Item       | Desired        | Actual           | Evidence              | Result   |
| ---------- | -------------- | ---------------- | --------------------- | -------- |
| Java       | 17             | 17.0.x           | `java -version`       | PASS     |
| Tomcat     | 10.1.x         | 10.1.60          | `catalina.sh version` | PASS     |
| App port   | 8080           | 8080             | `ss -ltnp`            | PASS     |
| PostgreSQL | localhost only | `127.0.0.1:5432` | `ss -ltnp`            | PASS     |
| SSH        | Admin IP only  | `0.0.0.0/0`      | SG evidence           | **FAIL** |
| Monitoring | Prometheus UP  | node DOWN        | Prometheus Targets    | **FAIL** |

Đây là **configuration baseline management**.

---

# 4. Mental Model quan trọng nhất của Day 19

Khi hệ thống hỏng, **đừng bắt đầu bằng restart**.

Hãy suy luận theo chuỗi dependency:

```text
Client
 ↓
DNS / IP
 ↓
AWS route
 ↓
Internet Gateway
 ↓
Security Group
 ↓
EC2 network interface
 ↓
Linux firewall
 ↓
TCP LISTEN socket
 ↓
systemd service
 ↓
Tomcat/JVM
 ↓
Application
 ↓
Application configuration
 ↓
PostgreSQL socket
 ↓
PostgreSQL authentication
 ↓
Database/schema/data
```

Và observability:

```text
node_exporter
    ↓
Prometheus scrape
    ↓
time series
    ↓
PromQL
    ↓
Grafana
    ↓
Alert
```

Một lỗi ở layer thấp có thể làm layer cao trông giống như bị lỗi.

Ví dụ:

```text
curl timeout
```

không đồng nghĩa:

```text
Tomcat hỏng.
```

Nó có thể là:

```text
Security Group
route
firewall
Tomcat không listen
sai port
Tomcat startup failed
JVM startup failed
application deployment failed
```

Day 19 bắt đầu hình thành cách suy nghĩ mà Day 20 sẽ sử dụng để failure injection.

---

# 5. Configuration Baseline là gì?

Configuration baseline là tập hợp các cấu hình đã được xác nhận/approved mà hệ thống **phải** có.

Một baseline tốt trả lời được:

| Domain      | Baseline cần biết                                 |
| ----------- | ------------------------------------------------- |
| AWS         | Region, VPC, subnet, SG, EC2, EBS, IAM, tags      |
| Linux       | OS, hostname, users, groups, filesystem, packages |
| Java        | JDK version, JAVA_HOME                            |
| Tomcat      | Version, CATALINA_HOME, service user, port        |
| Application | Artifact version, context path, config/profile    |
| PostgreSQL  | Version, hostname, port, DB, role                 |
| Monitoring  | Prometheus targets, node_exporter, Grafana        |
| Security    | Exposed ports, secret locations, privileges       |
| Operations  | logs, backup, restart, rollback                   |
| Cost        | EC2, EBS, IPv4, snapshots, CloudWatch, network    |

Không có baseline thì troubleshooting rất nguy hiểm bởi vì bạn không biết:

> “Đây là lỗi hay vốn dĩ hệ thống được thiết kế như vậy?”

---

# 6. Version awareness — Supplementary / Current verification

Day 19 không có yêu cầu “luôn dùng latest”. Trong operations, **approved version quan trọng hơn latest version**.

Tính đến tháng 10/2026:

Apache Tomcat 10.1.60 là một release hiện hành của dòng 10.1 và Tomcat 10.1 yêu cầu Java 11 trở lên. Tomcat 10 sử dụng Jakarta APIs thay vì namespace `javax.*` truyền thống của các Tomcat cũ, nên compatibility của application phải được kiểm tra chứ không thể chỉ nhìn Java version. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/appdev/installation.html?utm_source=chatgpt.com)

PostgreSQL hiện có các major version được support là 14–18; PostgreSQL 18.6 là current release tại thời điểm kiểm tra. PostgreSQL 14 sẽ hết support vào **12 November 2026**, nên một môi trường thực tế đang dùng PostgreSQL 14 cần có upgrade plan. [PostgreSQL](https://www.postgresql.org/support/versioning/?utm_source=chatgpt.com)

Prometheus download page hiện liệt kê 3.15.0 là latest release và 3.13.4 là một LTS release; node_exporter 1.12.1 là release mới nhất được liệt kê. Grafana download page hiện liệt kê Grafana 13.2.2. [prometheus.io](https://prometheus.io/download/?utm_source=chatgpt.com)

**Exam/operations rule:** nếu approved build sheet yêu cầu Java 17 + Tomcat X + PostgreSQL Y, bạn **không tự nâng version trong lúc build** chỉ vì upstream có version mới hơn.

---

# 7. Kiến trúc Reference Lab cho Day 19

## Supplementary / Reference Build Sheet

Để bạn có thể thực hành Day 19 ngay cả khi chưa có build sheet chính thức, ta dùng mô hình **single-EC2 integrated lab**.

```text
                          Internet
                             |
                     Security Group
                             |
             +---------------+---------------+
             |                               |
          SSH 22                         App 8080
       Admin-IP/32                     Admin-IP/32
             |                               |
             +---------------+---------------+
                             |
                       EC2 Instance
                     Ubuntu / systemd
                             |
        +--------------------+---------------------+
        |                    |                     |
     Tomcat                PostgreSQL         node_exporter
      :8080             127.0.0.1:5432      127.0.0.1:9100
        |                                          |
        |                                        scrape
        |                                          |
        +--------------------------------> Prometheus
                                         127.0.0.1:9090
                                               |
                                               v
                                            Grafana
                                         127.0.0.1:3000
```

Grafana sẽ được truy cập qua SSH tunnel:

```bash
ssh -L 3000:127.0.0.1:3000 ubuntu@PUBLIC_IP
```

Sau đó browser:

```text
http://localhost:3000
```

### Vì sao chọn single-node?

Vì nó rẻ hơn và giúp tập trung vào integration.

Nhưng đây **không phải kiến trúc production lý tưởng**.

Risk phải ghi rõ:

```text
Single point of failure
No workload isolation
DB shares CPU/RAM/disk with application
Monitoring disappears if EC2 dies
No high availability
No private-tier separation
```

Việc nhận ra các hạn chế này chính là một phần Day 19.

---

# 8. AWS layer — những gì phải hiểu

## 8.1 VPC

VPC tạo logical network boundary.

Ví dụ:

```text
10.20.0.0/16
```

Không phải tất cả IP trong VPC đều internet-accessible.

---

## 8.2 Subnet

Reference lab:

```text
Public subnet:
10.20.1.0/24
```

Một subnet trở thành “public subnet” về mặt kiến trúc khi route table có route tới Internet Gateway, chẳng hạn:

```text
0.0.0.0/0 → igw-xxxxxxxx
```

Nhưng instance vẫn cần public addressing và Security Group phù hợp mới nhận internet traffic trực tiếp.

---

## 8.3 Security Group

Security Group là **stateful firewall** của AWS.

Nếu inbound request được phép, return traffic của connection đó có thể được trả về mà không cần một inbound/outbound rule đối xứng theo kiểu NACL. AWS xác nhận security groups sử dụng connection tracking và có tính stateful. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-connection-tracking.html?utm_source=chatgpt.com)

Reference lab:

|     Port | Source              | Purpose                          |
| -------: | ------------------- | -------------------------------- |
|   TCP 22 | `YOUR_PUBLIC_IP/32` | SSH                              |
| TCP 8080 | `YOUR_PUBLIC_IP/32` | temporary lab application access |
|     5432 | **Not exposed**     | PostgreSQL local                 |
|     9100 | **Not exposed**     | node_exporter local              |
|     9090 | **Not exposed**     | Prometheus local                 |
|     3000 | **Not exposed**     | Grafana through SSH tunnel       |

Đặc biệt không mở:

```text
22     0.0.0.0/0
5432   0.0.0.0/0
3000   0.0.0.0/0
9090   0.0.0.0/0
9100   0.0.0.0/0
```

AWS Security Hub cũng xem unrestricted ingress tới các cổng nhạy cảm như 22, 3000, 5432 và 8080 là cấu hình cần đặc biệt tránh. [AWS Documentation](https://docs.aws.amazon.com/securityhub/latest/userguide/ec2-controls.html?utm_source=chatgpt.com)

---

# 9. Linux baseline

Day 19 phải tích hợp lại Day 2–6.

Bạn cần biết trạng thái sau:

```bash
hostnamectl
cat /etc/os-release
uname -r

whoami
id

df -hT
df -i
free -h

ip addr
ip route

systemctl --failed

ss -ltnp
```

### Expected invariant

Không cần output giống nhau từng byte.

Điều phải invariant là:

```text
Filesystem còn đủ space
inode không cạn
RAM không ở trạng thái bất thường
default route đúng
không có unexpected failed units
expected services listening đúng port
```

Đây là khác biệt giữa:

**Example output**

và

**Expected invariant**.

Trong troubleshooting, invariant quan trọng hơn việc nhớ output mẫu.

---

# 10. Service accounts

Không chạy mọi thứ bằng root.

Reference baseline:

| Service       | Account         |
| ------------- | --------------- |
| Tomcat        | `tomcat`        |
| PostgreSQL    | `postgres`      |
| Prometheus    | `prometheus`    |
| node_exporter | `node_exporter` |
| Grafana       | `grafana`       |

Một service account thường:

```text
không dùng để SSH login
không cần interactive shell
chỉ có quyền trên file/directory cần thiết
```

Ví dụ:

```bash
getent passwd tomcat
getent passwd postgres
getent passwd prometheus
getent passwd node_exporter
getent passwd grafana
```

Không nên thấy mọi service chạy:

```text
root
```

---

# 11. systemd integration

Khi bạn chạy một application production-like, bạn không nên yêu cầu operator chạy:

```bash
./startup.sh
```

bằng tay sau mỗi reboot.

Bạn muốn:

```text
systemd
  ↓
service definition
  ↓
service account
  ↓
environment
  ↓
Java/Tomcat
```

Các command cơ bản:

```bash
sudo systemctl daemon-reload

sudo systemctl enable tomcat
sudo systemctl start tomcat

systemctl status tomcat
systemctl is-active tomcat
systemctl is-enabled tomcat
```

Điều cần hiểu:

```text
start   → chạy ngay
stop    → dừng ngay
restart → stop + start
reload  → reload nếu service hỗ trợ
enable  → cấu hình start khi boot
disable → bỏ automatic start
```

`enable` không có nghĩa service đang chạy.

`active` không có nghĩa service sẽ chạy lại sau reboot.

---

# 12. Java/Tomcat baseline

Ví dụ baseline:

```text
Java:              17
Tomcat:            10.1.x
Service account:   tomcat
CATALINA_HOME:     /opt/tomcat
Application:       app.war
Context:           /app
Tomcat port:       8080
```

Verification:

```bash
java -version

readlink -f "$(command -v java)"

/opt/tomcat/bin/catalina.sh version

ps -ef | grep '[j]ava'

ss -ltnp | grep ':8080'
```

Bạn cần phân biệt:

```text
Java installed
```

với:

```text
Tomcat actually uses that Java.
```

Một máy có thể chứa Java 17 và Java 25 cùng lúc.

Do đó evidence phải bao gồm process/runtime config chứ không chỉ:

```bash
java -version
```

---

# 13. PostgreSQL baseline

Application connectivity đi theo flow:

```text
Java application
       |
       | JDBC
       v
postgresql://HOST:5432/DB
       |
       v
PostgreSQL listener
       |
       v
pg_hba authentication
       |
       v
Role
       |
       v
Database / schema
```

Một JDBC URL điển hình:

```text
jdbc:postgresql://127.0.0.1:5432/appdb
```

Nhưng **tên environment/config key của application phụ thuộc chính application**.

Không được tự giả định app luôn dùng:

```text
SPRING_DATASOURCE_URL
```

nếu build sheet không nói đó là Spring Boot.

---

# 14. PostgreSQL verification

```bash
psql --version

sudo -u postgres pg_isready

sudo -u postgres psql -Atc 'show port;'

sudo -u postgres psql -Atc 'show listen_addresses;'

sudo -u postgres psql -Atc 'show hba_file;'
```

Socket:

```bash
sudo ss -ltnp | grep ':5432'
```

Với single-node lab, desired state tốt là:

```text
127.0.0.1:5432
```

chứ không phải:

```text
0.0.0.0:5432
```

Nếu application và database cùng host thì không có lý do hợp lý để mở PostgreSQL ra Internet.

---

# 15. Monitoring baseline

Luồng:

```text
Linux kernel
    ↓
node_exporter
    ↓
/metrics
    ↓
Prometheus
    ↓
PromQL
    ↓
Grafana
```

node_exporter cung cấp hardware/kernel/OS metrics cho Prometheus. Prometheus documentation cũng mô tả `scrape_configs` và `static_configs` là cơ chế chuẩn để cấu hình static targets. [prometheus.io](https://prometheus.io/docs/guides/node-exporter/?utm_source=chatgpt.com)

Reference endpoints:

```text
node_exporter:
127.0.0.1:9100

Prometheus:
127.0.0.1:9090

Grafana:
127.0.0.1:3000
```

Prometheus must see:

```text
up{job="node"} = 1
```

Đây là một invariant rất quan trọng.

---

# 16. Security baseline

Day 19 security không phải chỉ là Security Group.

Bạn phải kiểm tra security trên nhiều layer:

| Layer          | Check                                          |
| -------------- | ---------------------------------------------- |
| IAM            | Không dùng quyền rộng hơn cần thiết            |
| EC2 metadata   | Prefer IMDSv2                                  |
| Security Group | Least-privilege ingress                        |
| SSH            | key authentication, source restricted          |
| Linux          | service accounts                               |
| Files          | secrets không world-readable                   |
| PostgreSQL     | DB không public nếu không cần                  |
| Application    | secrets externalized                           |
| Monitoring     | admin interfaces không public                  |
| TLS            | dùng TLS khi traffic cần bảo vệ                |
| Evidence       | không chụp/đính kèm password/token/private key |

EC2 cho phép cấu hình instance metadata để **require IMDSv2**, và AWS có setting tương ứng cho new instances. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html?utm_source=chatgpt.com)

Kiểm tra metadata option từ workstation có AWS CLI phù hợp:

```bash
aws ec2 describe-instances \
  --instance-ids i-xxxxxxxxxxxxxxxxx \
  --query 'Reservations[0].Instances[0].MetadataOptions'
```

Desired:

```text
HttpTokens = required
```

---

# 17. Cost awareness

Một fresher administrator phải biết rằng “resource đang không được sử dụng” **không đồng nghĩa cost = 0**.

Các cost drivers của lab:

| Resource                    | Cost concern                               |
| --------------------------- | ------------------------------------------ |
| EC2                         | running instance time                      |
| EBS                         | allocated storage                          |
| EBS snapshots               | stored snapshot data                       |
| Public IPv4                 | hourly charge                              |
| Data transfer               | dependent on direction/destination         |
| NAT Gateway                 | hourly + data processing if deployed       |
| CloudWatch                  | detailed/custom metrics, logs, alarms etc. |
| Idle Elastic IP/public IPv4 | can still incur cost                       |

AWS hiện tính phí public IPv4 theo giờ; trang pricing hiện ghi **$0.005/hour cho mỗi in-use hoặc idle public IPv4 address**. Region/service eligibility vẫn phải được kiểm tra khi tính bill thật. [Amazon Web Services, Inc.](https://aws.amazon.com/vpc/pricing//?utm_source=chatgpt.com)

Công thức phải biết:

```text
IPv4 cost
=
number_of_public_IPs
× usage_hours
× IPv4_hourly_rate
```

Không cần thuộc lòng toàn bộ bảng giá EC2.

Bạn phải biết **tìm cost driver và kiểm tra calculator/billing**.

CloudWatch cũng cần chú ý: EC2 basic monitoring được cung cấp mặc định cho nhiều EC2 metrics, còn detailed monitoring/custom metrics có thể phát sinh charge. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-metrics-basic-detailed.html?utm_source=chatgpt.com)

---

# 18. Guided Lab Day 19

> **Cảnh báo:** Đây là lab có thể tạo AWS cost và có các bước thay đổi service/configuration. Chỉ làm trong lab/non-production hoặc môi trường đã được approve. Kiểm tra AWS region, account, tags, Security Group và backup trước khi thay đổi. Không gửi password, AWS keys, DB password hoặc SSH private key vào chat.

Dưới đây là **một workflow build hoàn chỉnh**. Đây sẽ là list chính duy nhất; hãy làm theo thứ tự vì thứ tự dependency rất quan trọng.

1. **Provision AWS foundation.** Tạo hoặc sử dụng VPC/subnet/route đã được approve. Launch một EC2 Linux với EBS-backed root volume, security group chỉ cho phép SSH từ public IP của bạn và application port từ phạm vi đã approve. Thêm tags như `Project=JavaFresherDay19`, `Environment=lab`, `Owner=<alias>` và `ExpiryDate=<date>`. Kiểm tra IMDSv2 được require nếu build sheet cho phép. Ghi lại instance ID, private IP, public IP, SG ID và volume ID làm evidence.

2. **Establish Linux baseline.** Sau khi SSH vào instance, chạy:

   ```bash
   hostnamectl
   cat /etc/os-release
   uname -r
   id
   ip addr
   ip route
   df -hT
   df -i
   free -h
   systemctl --failed
   ss -ltnp
   ```

   Lưu evidence trước khi cài đặt. Mục tiêu là biết trạng thái ban đầu và phát hiện unexpected conditions.

3. **Install base packages.** Với Ubuntu reference environment:

   ```bash
   sudo apt update

   sudo apt install -y \
     openjdk-17-jdk \
     postgresql \
     postgresql-contrib \
     curl \
     wget \
     tar \
     ca-certificates
   ```

   Sau đó:

   ```bash
   java -version
   psql --version

   systemctl status postgresql --no-pager
   ```

   Không nâng toàn hệ thống bằng một lệnh lớn chỉ để “cho mới nhất” nếu build sheet không yêu cầu.

4. **Install Tomcat under dedicated account.** Reference version:

   ```bash
   sudo useradd \
     --system \
     --home /opt/tomcat \
     --create-home \
     --shell /usr/sbin/nologin \
     tomcat
   ```

   Tải Tomcat:

   ```bash
   TOMCAT_VERSION=10.1.60

   cd /tmp

   wget \
     "https://dlcdn.apache.org/tomcat/tomcat-10/v${TOMCAT_VERSION}/bin/apache-tomcat-${TOMCAT_VERSION}.tar.gz"

   wget \
     "https://dlcdn.apache.org/tomcat/tomcat-10/v${TOMCAT_VERSION}/bin/apache-tomcat-${TOMCAT_VERSION}.tar.gz.sha512"
   ```

   Verify checksum:

   ```bash
   sha512sum -c \
     "apache-tomcat-${TOMCAT_VERSION}.tar.gz.sha512"
   ```

   Install:

   ```bash
   sudo tar \
     -xzf "apache-tomcat-${TOMCAT_VERSION}.tar.gz" \
     -C /opt/tomcat \
     --strip-components=1

   sudo chown -R tomcat:tomcat /opt/tomcat

   sudo chmod +x /opt/tomcat/bin/*.sh
   ```

   Verify:

   ```bash
   /opt/tomcat/bin/catalina.sh version
   ```

5. **Integrate Tomcat with systemd.** Xác định `JAVA_HOME`:

   ```bash
   dirname "$(dirname "$(readlink -f "$(command -v java)")")"
   ```

   Trên reference Ubuntu/amd64 thường là:

   ```text
   /usr/lib/jvm/java-17-openjdk-amd64
   ```

   nhưng phải verify thay vì đoán. Tạo:

   ```bash
   sudoedit /etc/systemd/system/tomcat.service
   ```

   Reference unit:

   ```ini
   [Unit]
   Description=Apache Tomcat
   After=network.target

   [Service]
   Type=forking

   User=tomcat
   Group=tomcat

   Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64"
   Environment="CATALINA_HOME=/opt/tomcat"
   Environment="CATALINA_BASE=/opt/tomcat"
   EnvironmentFile=-/etc/java-fresher/app.env

   ExecStart=/opt/tomcat/bin/startup.sh
   ExecStop=/opt/tomcat/bin/shutdown.sh

   Restart=on-failure
   RestartSec=5
   UMask=0027

   [Install]
   WantedBy=multi-user.target
   ```

   Sau đó:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable tomcat
   sudo systemctl start tomcat

   systemctl status tomcat --no-pager
   systemctl is-enabled tomcat
   systemctl is-active tomcat

   sudo ss -ltnp | grep ':8080'
   ```

6. **Provision PostgreSQL application access.** Đầu tiên inspect trạng thái:

   ```bash
   sudo -u postgres pg_isready

   sudo -u postgres psql -Atc 'show port;'

   sudo -u postgres psql -Atc 'show listen_addresses;'

   sudo -u postgres psql -Atc 'show hba_file;'
   ```

   Trong single-node lab, giữ PostgreSQL local nếu có thể. Tạo application role/database bằng interactive `psql` để password không xuất hiện trong shell history:

   ```bash
   sudo -u postgres psql
   ```

   Ví dụ:

   ```sql
   CREATE ROLE app_runtime
       LOGIN
       NOSUPERUSER
       NOCREATEDB
       NOCREATEROLE
       NOREPLICATION;

   \password app_runtime

   CREATE DATABASE appdb OWNER app_runtime;
   ```

   Đây là baseline đơn giản cho lab. Ở môi trường production-like hơn, nên tách schema/migration owner khỏi runtime role để runtime không cần quyền sở hữu database. Sau đó test:

   ```bash
   psql \
     -h 127.0.0.1 \
     -U app_runtime \
     -d appdb \
     -W
   ```

   Trong `psql`:

   ```sql
   SELECT current_user;
   SELECT current_database();
   SELECT now();
   ```

   Check socket:

   ```bash
   sudo ss -ltnp | grep ':5432'
   ```

   Nếu DB và app cùng node, mục tiêu là không cần public port 5432.

7. **Externalize application configuration.** Chuẩn bị directory:

   ```bash
   sudo install \
     -d \
     -o root \
     -g tomcat \
     -m 0750 \
     /etc/java-fresher

   sudo install \
     -o root \
     -g tomcat \
     -m 0640 \
     /dev/null \
     /etc/java-fresher/app.env
   ```

   Edit:

   ```bash
   sudoedit /etc/java-fresher/app.env
   ```

   **Không copy ví dụ dưới đây nếu application không dùng các key tương ứng.** Mapping phải lấy từ build sheet/application documentation.

   ```text
   APP_DB_URL=jdbc:postgresql://127.0.0.1:5432/appdb
   APP_DB_USER=app_runtime
   APP_DB_PASSWORD=<local-secret>
   ```

   Kiểm tra permission:

   ```bash
   sudo ls -l /etc/java-fresher/app.env

   sudo stat \
     -c '%U %G %a %n' \
     /etc/java-fresher/app.env
   ```

   Desired:

   ```text
   root tomcat 640 ...
   ```

   Không đưa file này vào evidence pack nếu nó chứa password.

8. **Deploy Java application.** Giả sử approved artifact có tên:

   ```text
   app.war
   ```

   Dừng Tomcat trước khi controlled deployment:

   ```bash
   sudo systemctl stop tomcat
   ```

   Nếu đang replace release cũ:

   ```bash
   sudo mkdir -p /opt/releases/backup

   sudo cp \
     /opt/tomcat/webapps/app.war \
     "/opt/releases/backup/app.war.$(date +%Y%m%d_%H%M%S)"
   ```

   Chỉ chạy backup command nếu old artifact tồn tại. Deploy:

   ```bash
   sudo install \
     -o tomcat \
     -g tomcat \
     -m 0640 \
     /path/to/approved/app.war \
     /opt/tomcat/webapps/app.war

   sudo systemctl start tomcat
   ```

   Validation:

   ```bash
   systemctl status tomcat --no-pager

   sudo journalctl \
     -u tomcat \
     -n 100 \
     --no-pager

   sudo tail -n 100 \
     /opt/tomcat/logs/catalina.out
   ```

   Kiểm tra socket:

   ```bash
   sudo ss -ltnp | grep ':8080'
   ```

   HTTP:

   ```bash
   curl -i http://127.0.0.1:8080/app/
   ```

   Nếu application có approved health endpoint, dùng endpoint đó. Đừng tự mặc định `/actuator/health` nếu build sheet không xác nhận Spring Boot Actuator.

9. **Install node_exporter.** Reference current binary:

   ```bash
   NODE_EXPORTER_VERSION=1.12.1

   cd /tmp

   wget \
     "https://github.com/prometheus/node_exporter/releases/download/v${NODE_EXPORTER_VERSION}/node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"
   ```

   Verify current SHA256 from official release:

   ```bash
   echo \
   "b51d8a76aa2a9156a55d501aca6276fae09e262259a5e4e831d2c2222f084e63  node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz" \
   | sha256sum -c -
   ```

   Account:

   ```bash
   sudo useradd \
     --system \
     --no-create-home \
     --shell /usr/sbin/nologin \
     node_exporter
   ```

   Install binary:

   ```bash
   tar \
     -xzf \
     "node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64.tar.gz"

   sudo install \
     -o root \
     -g root \
     -m 0755 \
     "node_exporter-${NODE_EXPORTER_VERSION}.linux-amd64/node_exporter" \
     /usr/local/bin/node_exporter
   ```

   systemd:

   ```bash
   sudoedit /etc/systemd/system/node_exporter.service
   ```

   ```ini
   [Unit]
   Description=Prometheus Node Exporter
   After=network-online.target

   [Service]
   User=node_exporter
   Group=node_exporter
   Type=simple

   ExecStart=/usr/local/bin/node_exporter \
     --web.listen-address=127.0.0.1:9100

   Restart=on-failure
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```

   Activate:

   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now node_exporter

   systemctl status node_exporter --no-pager

   curl -fsS \
     http://127.0.0.1:9100/metrics \
     | head
   ```

10. **Install Prometheus.** Reference current release:

    ```bash
    PROM_VERSION=3.15.0

    cd /tmp

    wget \
      "https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz"
    ```

    Checksum:

    ```bash
    echo \
    "2a542df32eac02ee17b9d844fb2aa1de00dafa5476579ba8a3ba862e9d572ea0  prometheus-${PROM_VERSION}.linux-amd64.tar.gz" \
    | sha256sum -c -
    ```

    Account/directories:

    ```bash
    sudo useradd \
      --system \
      --no-create-home \
      --shell /usr/sbin/nologin \
      prometheus

    sudo mkdir -p \
      /etc/prometheus/rules \
      /var/lib/prometheus

    sudo chown \
      prometheus:prometheus \
      /var/lib/prometheus
    ```

    Install:

    ```bash
    tar \
      -xzf \
      "prometheus-${PROM_VERSION}.linux-amd64.tar.gz"

    cd "prometheus-${PROM_VERSION}.linux-amd64"

    sudo install \
      -m 0755 \
      prometheus \
      promtool \
      /usr/local/bin/
    ```

    Config:

    ```bash
    sudoedit /etc/prometheus/prometheus.yml
    ```

    ```yaml
    global:
      scrape_interval: 15s

    rule_files:
      - /etc/prometheus/rules/*.yml

    scrape_configs:
      - job_name: prometheus
        static_configs:
          - targets:
              - 127.0.0.1:9090

      - job_name: node
        static_configs:
          - targets:
              - 127.0.0.1:9100
    ```

    Validate **before restart**:

    ```bash
    promtool check config \
      /etc/prometheus/prometheus.yml
    ```

    systemd:

    ```bash
    sudoedit /etc/systemd/system/prometheus.service
    ```

    ```ini
    [Unit]
    Description=Prometheus
    Wants=network-online.target
    After=network-online.target

    [Service]
    User=prometheus
    Group=prometheus
    Type=simple

    ExecStart=/usr/local/bin/prometheus \
      --config.file=/etc/prometheus/prometheus.yml \
      --storage.tsdb.path=/var/lib/prometheus \
      --storage.tsdb.retention.time=7d \
      --web.listen-address=127.0.0.1:9090

    Restart=on-failure
    RestartSec=5

    [Install]
    WantedBy=multi-user.target
    ```

    Start:

    ```bash
    sudo systemctl daemon-reload
    sudo systemctl enable --now prometheus

    systemctl status prometheus --no-pager

    curl -fsS \
      http://127.0.0.1:9090/-/ready
    ```

    Query:

    ```bash
    curl -s \
      'http://127.0.0.1:9090/api/v1/query?query=up'
    ```

    Bạn muốn thấy node target ở trạng thái `1`.

11. **Configure a basic integrated alert rule.** Tạo:

    ```bash
    sudoedit /etc/prometheus/rules/day19.yml
    ```

    ```yaml
    groups:
      - name: day19
        rules:
          - alert: NodeExporterDown
            expr: up{job="node"} == 0
            for: 2m
            labels:
              severity: warning
            annotations:
              summary: "Node exporter target is down"
    ```

    Validate:

    ```bash
    promtool check rules \
      /etc/prometheus/rules/day19.yml

    promtool check config \
      /etc/prometheus/prometheus.yml
    ```

    Reload safely with:

    ```bash
    sudo systemctl reload prometheus
    ```

    nếu unit/package của bạn hỗ trợ reload; nếu không, dùng approved controlled restart:

    ```bash
    sudo systemctl restart prometheus
    ```

    Sau đó kiểm tra:

    ```bash
    systemctl is-active prometheus

    curl -fsS \
      http://127.0.0.1:9090/-/ready
    ```

    Nếu Day 17 environment của bạn có Alertmanager thì integrate notification path theo cấu hình đó. Không tự tạo một notification channel ngoài approved scope.

12. **Install Grafana.** Tài liệu Grafana hiện hỗ trợ Debian/Ubuntu và systemd, và current download page tại thời điểm kiểm tra đưa ra Grafana 13.2.2. [Grafana Labs](https://grafana.com/docs/grafana/latest/setup-grafana/installation/?utm_source=chatgpt.com)
    Reference:

    ```bash
    cd /tmp

    wget \
      https://dl.grafana.com/grafana/release/13.2.2/grafana_13.2.2_34846740809_linux_amd64.deb

    sudo apt-get install -y \
      adduser \
      libfontconfig1 \
      musl

    sudo dpkg -i \
      grafana_13.2.2_34846740809_linux_amd64.deb
    ```

    Start:

    ```bash
    sudo systemctl enable --now grafana-server

    systemctl status grafana-server --no-pager
    ```

    Với lab này, tốt hơn là chỉ cho Grafana listen local. Sau khi cấu hình `[server]` trong `/etc/grafana/grafana.ini` theo approved config, verify:

    ```bash
    sudo ss -ltnp | grep ':3000'
    ```

    Desired:

    ```text
    127.0.0.1:3000
    ```

    Từ workstation:

    ```bash
    ssh \
      -L 3000:127.0.0.1:3000 \
      ubuntu@EC2_PUBLIC_IP
    ```

    Browser:

    ```text
    http://localhost:3000
    ```

    Add Prometheus data source:

    ```text
    URL = http://127.0.0.1:9090
    ```

    Dashboard tối thiểu nên cho bạn quan sát CPU, memory, disk/filesystem và host availability.

13. **Execute end-to-end validation.** Đây là bước quan trọng nhất Day 19:

    ```bash
    systemctl --failed
    ```

    Services:

    ```bash
    for svc in \
      postgresql \
      tomcat \
      node_exporter \
      prometheus \
      grafana-server
    do
      echo "===== $svc ====="
      systemctl is-active "$svc"
      systemctl is-enabled "$svc"
    done
    ```

    Ports:

    ```bash
    sudo ss -ltnp
    ```

    Expected topology:

    ```text
    :8080             Tomcat
    127.0.0.1:5432    PostgreSQL
    127.0.0.1:9100    node_exporter
    127.0.0.1:9090    Prometheus
    127.0.0.1:3000    Grafana
    ```

    DB:

    ```bash
    sudo -u postgres pg_isready
    ```

    App:

    ```bash
    curl -i \
      http://127.0.0.1:8080/app/
    ```

    node_exporter:

    ```bash
    curl -fsS \
      http://127.0.0.1:9100/metrics \
      >/dev/null \
      && echo OK
    ```

    Prometheus:

    ```bash
    curl -fsS \
      http://127.0.0.1:9090/-/ready
    ```

    Grafana:

    ```bash
    curl -fsS \
      http://127.0.0.1:3000/api/health
    ```

    Capacity:

    ```bash
    df -hT
    df -i
    free -h
    ```

    Logs:

    ```bash
    sudo journalctl \
      -u tomcat \
      -n 50 \
      --no-pager

    sudo journalctl \
      -u postgresql \
      -n 50 \
      --no-pager

    sudo journalctl \
      -u node_exporter \
      -n 50 \
      --no-pager

    sudo journalctl \
      -u prometheus \
      -n 50 \
      --no-pager

    sudo journalctl \
      -u grafana-server \
      -n 50 \
      --no-pager
    ```

14. **Produce the required Day 19 deliverables and clean up.** Nộp architecture diagram, configuration inventory và cost/risk checklist. Sau khi đã giữ evidence cần thiết, thực hiện cleanup đúng policy. Nếu terminate EC2, nhớ rằng termination là hành động permanent; EBS volume có thể bị delete tùy `DeleteOnTermination`. AWS khuyến nghị kiểm tra/backup dữ liệu cần giữ trước termination. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html?utm_source=chatgpt.com)

---

# 19. Architecture deliverable phải như thế nào?

Một architecture submission tốt không cần đẹp như solution architect diagram.

Nó phải đủ để operator hiểu:

```text
Internet
   |
   | SSH 22 from Admin-IP/32
   | App 8080 from Admin-IP/32
   v
Security Group: sg-day19
   |
   v
EC2: java-fresher-day19
Private IP: 10.20.1.x
   |
   +--- Tomcat :8080
   |       |
   |       +--- app.war
   |       |
   |       +--- JDBC → localhost:5432
   |
   +--- PostgreSQL :5432 localhost-only
   |
   +--- node_exporter :9100 localhost-only
   |
   +--- Prometheus :9090 localhost-only
   |       |
   |       +--- scrape node_exporter
   |
   +--- Grafana :3000 localhost-only
           |
           +--- Prometheus datasource

Admin
   |
   +--- SSH tunnel → Grafana
```

Architecture phải thể hiện **traffic flow**, không chỉ boxes.

---

# 20. Configuration Inventory — deliverable bắt buộc

Bạn có thể dùng format sau:

| Domain     | Item          | Desired state     | Actual state | Evidence                | Status |
| ---------- | ------------- | ----------------- | ------------ | ----------------------- | ------ |
| AWS        | Region        | approved region   | ...          | CLI/console             | PASS   |
| AWS        | EC2 ID        | approved resource | ...          | EC2 describe            | PASS   |
| AWS        | SG SSH        | Admin-IP/32       | ...          | SG rules                | PASS   |
| AWS        | Public IPv4   | required only     | ...          | EC2 network             | PASS   |
| AWS        | IMDS          | v2 required       | ...          | MetadataOptions         | PASS   |
| Linux      | OS            | Ubuntu approved   | ...          | `/etc/os-release`       | PASS   |
| Linux      | root FS       | healthy           | ...          | `df -hT`                | PASS   |
| Java       | Java          | 17                | ...          | `java -version`         | PASS   |
| Tomcat     | Version       | approved          | ...          | `catalina.sh version`   | PASS   |
| Tomcat     | Port          | 8080              | ...          | `ss -ltnp`              | PASS   |
| App        | Artifact      | approved WAR      | ...          | checksum/release record | PASS   |
| App        | Context       | `/app`            | ...          | HTTP request            | PASS   |
| DB         | Port          | 5432              | ...          | `show port`             | PASS   |
| DB         | Exposure      | localhost         | ...          | `ss -ltnp`              | PASS   |
| DB         | Runtime user  | least privilege   | ...          | `\du` evidence          | PASS   |
| Monitoring | node_exporter | UP                | ...          | Prometheus target       | PASS   |
| Monitoring | Prometheus    | READY             | ...          | `/-/ready`              | PASS   |
| Monitoring | Grafana       | healthy           | ...          | `/api/health`           | PASS   |
| Operations | failed units  | none expected     | ...          | `systemctl --failed`    | PASS   |
| Capacity   | disk          | below threshold   | ...          | `df -hT`                | PASS   |

### Quan trọng

Không đưa vào inventory:

```text
actual DB password
AWS secret key
SSH private key
Grafana password
API token
```

Inventory ghi:

```text
Secret stored at: protected env file
Permissions: 0640 root:tomcat
```

chứ không ghi secret value.

---

# 21. Cost Checklist

Day 19 submission phải chứng minh bạn **đã suy nghĩ về cost**, không nhất thiết phải dự đoán bill tuyệt đối chính xác.

| Check          | Evidence / Question                              |
| -------------- | ------------------------------------------------ |
| EC2            | Instance type có lớn quá lab không?              |
| Runtime        | Có cần chạy 24/7 không?                          |
| EBS            | Volume size có hợp lý không?                     |
| Snapshot       | Snapshot nào cần giữ?                            |
| Public IPv4    | Có bao nhiêu address?                            |
| NAT Gateway    | Có thực sự cần không?                            |
| CloudWatch     | Có bật detailed monitoring/custom metrics không? |
| Data transfer  | Có transfer lớn cross-AZ/Internet không?         |
| Stopped EC2    | EBS vẫn còn phải trả không?                      |
| Terminated EC2 | orphaned volume/snapshot có còn không?           |
| Tags           | Có Owner/Project/ExpiryDate không?               |
| Cleanup        | Ai chịu trách nhiệm xóa resource?                |

Một administrator tốt không chỉ nói:

> “Tôi stop EC2 nên chắc chắn hết charge.”

Bạn phải kiểm tra các resource phụ thuộc còn lại.

---

# 22. Risk Checklist

| Risk                  | Impact                    | Current control            | Remaining risk            |
| --------------------- | ------------------------- | -------------------------- | ------------------------- |
| SSH public            | unauthorized access       | source `/32`               | IP changes                |
| App on 8080           | plaintext transport       | temporary lab restriction  | no TLS                    |
| DB compromise         | data exposure             | localhost-only             | same-host compromise      |
| Secrets in env file   | credential exposure       | `0640 root:tomcat`         | privileged admin can read |
| Single EC2            | complete outage           | backup/rebuild docs        | SPOF remains              |
| Disk full             | app/DB failure            | monitoring                 | reactive if no alert      |
| Tomcat OOM            | service failure           | JVM limits/monitoring      | workload spike            |
| Bad WAR               | deployment outage         | previous artifact retained | rollback time             |
| Monitoring co-located | monitoring dies with host | none                       | visibility loss           |
| AWS runaway resources | unexpected cost           | tagging/cleanup            | manual oversight          |

Điều quan trọng là không phải “không có risk”.

Điều quan trọng là:

```text
Know the risk
→ quantify impact
→ identify control
→ identify residual risk
→ accept / remediate / escalate
```

---

# 23. Evidence pack

Một evidence pack tốt có thể có cấu trúc:

```text
day19-evidence/
├── 01-architecture/
│   └── architecture.txt
│
├── 02-aws/
│   ├── instance.txt
│   ├── security-group.txt
│   └── tags.txt
│
├── 03-linux/
│   ├── os.txt
│   ├── filesystem.txt
│   ├── memory.txt
│   └── ports.txt
│
├── 04-java-tomcat/
│   ├── java-version.txt
│   ├── tomcat-version.txt
│   ├── tomcat-status.txt
│   └── app-http-validation.txt
│
├── 05-postgresql/
│   ├── pg-version.txt
│   ├── pg-ready.txt
│   └── db-connectivity.txt
│
├── 06-monitoring/
│   ├── services.txt
│   ├── prometheus-up.txt
│   └── grafana-health.txt
│
├── 07-inventory/
│   └── configuration-inventory.csv
│
└── 08-risk-cost/
    ├── cost-checklist.txt
    └── risk-register.txt
```

Không lưu password/private key vào đây.

---

# 24. Validation chain — phần bạn cần thuộc tư duy, không phải thuộc lệnh

Khi application hoạt động, bạn nên có evidence ở từng layer:

| Layer         | Validation                    |
| ------------- | ----------------------------- |
| AWS instance  | running                       |
| SG            | expected ingress only         |
| Linux         | healthy                       |
| Java          | correct runtime               |
| Tomcat        | service active                |
| Network       | `:8080` listening             |
| HTTP          | request succeeds              |
| DB            | `pg_isready` succeeds         |
| JDBC          | app confirms DB interaction   |
| node_exporter | metrics endpoint responds     |
| Prometheus    | `up=1`                        |
| Grafana       | data source healthy           |
| Disk          | sufficient                    |
| Logs          | no unexpected critical errors |

Điều này tạo ra một **proof chain**.

---

# 25. Troubleshooting integration

Giả sử:

```text
User says: application doesn't work.
```

Không được lập tức:

```bash
sudo systemctl restart tomcat
```

Workflow đúng:

```text
Symptom
  ↓
Expected behavior
  ↓
Scope
  ↓
Recent change
  ↓
Failing layer
  ↓
Evidence
  ↓
Hypothesis
  ↓
Safest test
  ↓
Root cause
  ↓
Remediation
  ↓
Validation
  ↓
Evidence
```

---

## Scenario A — Connection timeout

Client:

```bash
curl http://PUBLIC_IP:8080/app/
```

Timeout.

Đầu tiên kiểm tra:

```bash
systemctl is-active tomcat
```

Sau đó:

```bash
sudo ss -ltnp | grep ':8080'
```

Nếu Tomcat listen local và internal curl được:

```bash
curl http://127.0.0.1:8080/app/
```

thì khả năng lớn failure nằm ở:

```text
AWS SG
routing
public addressing
host firewall
binding address
```

Không phải database.

---

# 26. Scenario B — Connection refused

```text
curl: connection refused
```

Thông thường có nghĩa client đã reach host nhưng không có process accepting connection trên port đó hoặc host actively rejected.

Check:

```bash
sudo ss -ltnp | grep ':8080'
```

Nếu không có output:

```bash
systemctl status tomcat --no-pager

sudo journalctl \
  -u tomcat \
  -n 100 \
  --no-pager
```

---

# 27. Scenario C — HTTP 500

HTTP 500 rất khác timeout.

Network/TCP/Tomcat có thể đã hoạt động.

Layer nghi ngờ cao hơn:

```text
application
configuration
database
permissions
dependency
```

Evidence:

```bash
sudo tail -n 200 \
  /opt/tomcat/logs/catalina.out
```

Tìm:

```text
SQLException
authentication failed
connection refused
permission denied
NoSuchMethodError
ClassNotFoundException
OutOfMemoryError
```

Không grep mỗi `"ERROR"` rồi kết luận.

Cần đọc context quanh timestamp.

---

# 28. Scenario D — PostgreSQL authentication failed

Ví dụ log:

```text
password authentication failed
```

Điều này cho biết rất nhiều:

```text
TCP connection tới PostgreSQL có thể đã thành công.
PostgreSQL đã nhận được authentication attempt.
```

Vậy chưa nên sửa:

```text
route
Security Group
Tomcat port
```

Kiểm tra:

```bash
psql \
  -h 127.0.0.1 \
  -U app_runtime \
  -d appdb \
  -W
```

Nếu manual login cũng fail:

```text
role/password/pg_hba/database
```

Nếu manual login thành công nhưng app fail:

```text
app config
secret escaping
environment loading
wrong profile
stale process environment
```

---

# 29. Scenario E — Tomcat active nhưng app không tồn tại

```bash
systemctl is-active tomcat
```

returns:

```text
active
```

Nhưng:

```text
HTTP 404
```

Đây là ví dụ kinh điển:

> Service health ≠ application health.

Kiểm tra:

```bash
ls -l /opt/tomcat/webapps/

sudo tail -n 200 \
  /opt/tomcat/logs/catalina.out
```

Possible causes:

```text
wrong WAR name
wrong context path
deployment failure
incompatible WAR
permission problem
Jakarta/javax incompatibility
```

Tomcat 10 trở lên chuyển từ Java EE `javax.*` sang Jakarta `jakarta.*`, vì vậy một WAR cũ có thể gặp compatibility problem dù Tomcat process vẫn chạy. [Apache Tomcat](https://tomcat.apache.org/download-10.cgi?utm_source=chatgpt.com)

---

# 30. Scenario F — Prometheus không có node metrics

Query:

```promql
up{job="node"}
```

returns:

```text
0
```

Check target itself:

```bash
curl -fsS \
  http://127.0.0.1:9100/metrics \
  | head
```

Nếu fail:

```bash
systemctl status node_exporter --no-pager

sudo journalctl \
  -u node_exporter \
  -n 100 \
  --no-pager
```

Nếu exporter works nhưng Prometheus `up=0`:

```text
wrong target
wrong port
wrong hostname
Prometheus config not reloaded
YAML issue
```

Check:

```bash
promtool check config \
  /etc/prometheus/prometheus.yml
```

---

# 31. Scenario G — Grafana “No data”

Đừng kết luận Grafana hỏng.

Chain:

```text
Grafana
 ↓
Prometheus datasource
 ↓
Prometheus
 ↓
target
 ↓
node_exporter
```

Check bottom-up:

```bash
curl -fsS http://127.0.0.1:9100/metrics >/dev/null
```

then:

```bash
curl -fsS http://127.0.0.1:9090/-/ready
```

then query:

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=up'
```

Nếu Prometheus có data nhưng Grafana không có:

```text
datasource URL
Grafana datasource authentication
dashboard query
time range
variables
```

---

# 32. Scenario H — Disk full

```bash
df -hT
df -i
```

Phải kiểm tra cả:

```text
bytes
```

và:

```text
inodes
```

Một filesystem có thể còn GB nhưng hết inode.

Ảnh hưởng:

```text
PostgreSQL cannot write
Tomcat cannot write logs/temp
Prometheus TSDB cannot persist
Grafana DB cannot write
system services fail
```

Không chạy:

```bash
rm -rf /var/log/*
```

trong hoảng loạn.

Cần xác định data:

```bash
sudo du -xhd1 /var | sort -h
```

rồi tiếp tục drill-down một cách có kiểm soát.

---

# 33. Rollback mindset

Mọi approved change nên có:

```text
Before state
Change
Validation
Rollback trigger
Rollback procedure
After validation
```

Ví dụ deployment:

```text
Current:
app-v1.war

Change:
app-v2.war

Rollback artifact:
app-v1.war
```

Nếu new deployment không đạt validation:

```bash
sudo systemctl stop tomcat
```

Khôi phục known-good artifact:

```bash
sudo cp \
  /opt/releases/backup/<known-good-war> \
  /opt/tomcat/webapps/app.war
```

Correct ownership:

```bash
sudo chown \
  tomcat:tomcat \
  /opt/tomcat/webapps/app.war
```

Start:

```bash
sudo systemctl start tomcat
```

Validation lại toàn chain.

---

# 34. Configuration rollback

Trước khi thay file quan trọng:

```bash
sudo cp \
  /etc/prometheus/prometheus.yml \
  "/etc/prometheus/prometheus.yml.$(date +%Y%m%d_%H%M%S).bak"
```

Sau chỉnh sửa:

```bash
promtool check config \
  /etc/prometheus/prometheus.yml
```

Đây là pattern quan trọng:

```text
backup
→ edit
→ syntax validation
→ apply
→ functional validation
```

không phải:

```text
edit
→ restart
→ pray
```

---

# 35. AWS recovery considerations

Trước khi terminate EC2, phải biết:

```text
Is data backed up?
Which EBS volumes have DeleteOnTermination?
Do I need a snapshot?
Are there logs/evidence I need?
Are there Elastic IPs?
Are there orphan volumes?
```

AWS xác nhận EBS-backed instance có `DeleteOnTermination`; root volumes thường có default behavior khác các data volumes, nên operator phải kiểm tra actual setting chứ không đoán. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/preserving-volumes-on-termination.html?utm_source=chatgpt.com)

Đây chính là kiến thức Day 10 được tích hợp vào Day 19.

---

# 36. L1 boundary

Một junior/L1 administrator thường có thể làm những việc đã được runbook/approval cho phép như:

```text
inspect status
inspect logs
check ports
check disk
validate configuration
test DB connectivity
restart approved service
reapply known-good config
rollback known release
collect evidence
```

Nhưng nên escalate khi gặp:

| Situation                          | Why escalate             |
| ---------------------------------- | ------------------------ |
| suspected database corruption      | data integrity risk      |
| unknown security compromise        | incident response        |
| leaked private key/password        | credential incident      |
| unexplained data loss              | high impact              |
| architecture/security rule change  | blast radius             |
| production IAM policy modification | privilege risk           |
| repeated unexplained JVM crash     | deeper engineering issue |
| irreversible DB change             | recovery risk            |
| unknown network routing change     | multi-service impact     |

L1 tốt không phải người “sửa được mọi thứ”.

L1 tốt là người:

```text
phân loại đúng
thu evidence đúng
không làm blast radius lớn hơn
fix đúng phần được phép
escalate đúng thời điểm
```

---

# 37. Anti-patterns Day 19

| Anti-pattern                             | Vì sao nguy hiểm                             |
| ---------------------------------------- | -------------------------------------------- |
| Mở tất cả SG để “test cho dễ”            | che mất network/security issue               |
| `chmod -R 777`                           | phá least privilege                          |
| chạy Tomcat bằng root                    | unnecessary privilege                        |
| hardcode DB password vào WAR             | secret leakage                               |
| paste credential vào ticket              | security incident                            |
| restart service trước khi lấy log        | destroy useful evidence                      |
| upgrade mọi package khi troubleshooting  | tạo thêm variables                           |
| install latest version ngoài build sheet | configuration drift                          |
| expose PostgreSQL to Internet            | unnecessary attack surface                   |
| expose Prometheus/node_exporter          | information exposure                         |
| chỉ check `systemctl active`             | service may be active but application broken |
| chỉ check HTTP                           | backend DB may still be degraded             |
| chỉ check dashboard                      | monitoring path itself can fail              |
| terminate EC2 ngay sau lab               | có thể mất evidence/data                     |
| stop EC2 rồi nghĩ cost = zero            | EBS/other resources can remain               |

---

# 38. Security check trước khi PASS Day 19

Bạn chưa nên đánh dấu build PASS nếu:

```text
SSH is 0.0.0.0/0
PostgreSQL is public
Grafana is publicly exposed unnecessarily
secrets are world-readable
services unnecessarily run as root
IMDS settings were ignored
unknown ports are listening
unused AWS resources remain
```

Useful audit:

```bash
sudo ss -lntup
```

Sau đó tự hỏi từng socket:

```text
What process owns it?
Why is it listening?
Who can reach it?
Does it need to be public?
```

Đây là một habit operations rất mạnh.

---

# 39. Final integrated health-check

Một operator mới nhận shift có thể chạy:

```bash
date

hostname

uptime

df -hT

free -h

systemctl --failed

systemctl is-active tomcat

systemctl is-active postgresql

systemctl is-active node_exporter

systemctl is-active prometheus

systemctl is-active grafana-server

sudo ss -ltnp

sudo -u postgres pg_isready

curl -sS \
  -o /dev/null \
  -w 'APP_HTTP=%{http_code}\n' \
  http://127.0.0.1:8080/app/

curl -fsS \
  http://127.0.0.1:9090/-/ready

curl -fsS \
  http://127.0.0.1:3000/api/health
```

Nhưng lưu ý:

**Health-check script không thay thế troubleshooting.**

Nó chỉ giúp phát hiện:

```text
expected vs actual
```

nhanh hơn.

---

# 40. Exam Focus — Day 19 Theory

Bạn nên có khả năng trả lời được các câu dạng:

**“Configuration baseline là gì?”**

Desired state đã được approve để so sánh với actual state và phát hiện drift.

**“Tại sao service active chưa chứng minh application healthy?”**

Vì process/service có thể chạy nhưng deployment, dependency, DB hoặc business endpoint vẫn lỗi.

**“Security Group stateful nghĩa là gì?”**

Response traffic của allowed connection được connection tracking xử lý tương ứng, không cần thiết kế rule theo kiểu stateless NACL. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-connection-tracking.html?utm_source=chatgpt.com)

**“Tại sao không public port 5432?”**

Không expose unnecessary attack surface; app và DB nên giao tiếp qua trusted/private/local path.

**“Tại sao Prometheus không nên public mặc định?”**

Prometheus có thể tiết lộ infrastructure/service topology và metrics; admin interface không cần public Internet để phục vụ lab.

**“Tại sao phải inventory?”**

Để biết exact runtime state, reproduce environment, review security, troubleshoot drift, handover và audit.

**“Tại sao cần cost checklist?”**

AWS resources có independent billing lifecycle; stopped application không đồng nghĩa mọi resource không còn charge.

---

# 41. Exam Focus — Practical

Trong practical exam, họ có thể không nói:

> “Chạy lệnh X.”

Thay vào đó bạn có thể nhận:

```text
Deploy application.
Connect to DB.
Expose approved endpoint.
Enable monitoring.
Validate.
Provide evidence.
```

Bạn phải tự chuyển yêu cầu thành dependency graph:

```text
AWS
 ↓
Linux
 ↓
Java/Tomcat
 ↓
configuration
 ↓
database
 ↓
application
 ↓
monitoring
 ↓
evidence
```

Nếu deployment lỗi, điểm quan trọng không chỉ là cuối cùng có chạy hay không.

Evaluator có thể nhìn cách bạn:

```text
inspect
reason
avoid destructive actions
validate
document
rollback
```

---

# 42. Knowledge Check

Hãy tự trả lời trước khi xem lại bài:

| Question                                       | Bạn cần giải thích được                         |
| ---------------------------------------------- | ----------------------------------------------- |
| Configuration baseline khác inventory thế nào? | desired state vs recorded actual state          |
| Vì sao `systemctl active` chưa đủ?             | process ≠ end-to-end health                     |
| Timeout khác connection refused thế nào?       | network/filter vs listener/reject implications  |
| HTTP 500 giúp narrow scope thế nào?            | HTTP stack reached, higher-layer failure likely |
| PostgreSQL auth error nói gì?                  | DB was reached far enough to authenticate       |
| Vì sao không mở 5432 public?                   | least privilege                                 |
| `up{job="node"}=0` nghĩa gì?                   | Prometheus scrape failed                        |
| Grafana No data có chắc Grafana hỏng?          | no                                              |
| Disk full có ảnh hưởng những service nào?      | potentially all                                 |
| Tại sao cần check inode?                       | filesystem can exhaust inode independently      |
| Tại sao phải retain previous WAR?              | rollback                                        |
| Vì sao không upgrade random packages?          | drift and additional variables                  |
| IMDSv2 giúp ở layer nào?                       | EC2 metadata security                           |
| Stop EC2 có đủ để cleanup cost?                | no                                              |
| Risk và issue khác nhau thế nào?               | potential future vs current problem             |

---

# 43. Independent Practical Challenge

Sau khi hoàn thành guided lab, hãy thử một lần **không nhìn hướng dẫn**.

Bạn được giao:

```text
Build one approved EC2 environment hosting:

Java 17
Tomcat
approved WAR
PostgreSQL
node_exporter
Prometheus
Grafana

Requirements:

Application must connect to PostgreSQL.
PostgreSQL must not be Internet-accessible.
Monitoring must confirm host health.
Grafana admin interface must not be publicly exposed.
SSH must use least-privilege source restriction.
System must survive reboot.
Produce architecture, inventory, security review,
cost review and validation evidence.
```

Success criteria:

```text
EC2 running
Linux healthy
services enabled
application HTTP works
application DB path works
PostgreSQL local/private
node_exporter UP
Prometheus ready
Grafana healthy
no unexpected failed services
no unnecessary public ports
inventory complete
cost/risk checklist complete
rollback path documented
```

Không coi task hoàn thành chỉ vì browser mở được application.

---

# 44. Mastery Check Day 19

Day 19 chỉ có thể coi là **practical mastery** khi bạn thực sự chứng minh được các trạng thái sau:

| Capability                 | Explanation alone | Practical evidence |
| -------------------------- | ----------------: | -----------------: |
| Understand architecture    |                 ✓ |                    |
| Provision/configure        |                   |                  ✓ |
| Deploy WAR                 |                   |                  ✓ |
| Configure DB               |                   |                  ✓ |
| Validate JDBC path         |                   |                  ✓ |
| Validate ports             |                   |                  ✓ |
| Configure monitoring       |                   |                  ✓ |
| Read logs                  |                   |                  ✓ |
| Security review            |                   |                  ✓ |
| Cost review                |                   |                  ✓ |
| Architecture documentation |                   |                  ✓ |
| Configuration inventory    |                   |                  ✓ |
| Risk checklist             |                   |                  ✓ |
| Rollback readiness         |                   |                  ✓ |

Đọc hết bài **không có nghĩa đã master Day 19**.

Mastery của ngày hôm nay phải là:

```text
I can receive a build sheet
        ↓
understand architecture
        ↓
provision safely
        ↓
configure Linux
        ↓
configure Java/Tomcat
        ↓
connect PostgreSQL
        ↓
deploy application
        ↓
monitor the system
        ↓
validate every dependency
        ↓
identify security/cost risks
        ↓
produce evidence
        ↓
rollback/recover if required
        ↓
hand over the environment
```

---

# 45. Những gì Day 19 đã tích hợp từ các ngày trước

Day 19 gần như là một **mini-capstone trước troubleshooting day**:

```text
Day 2–6
Linux fundamentals + systemd + network + disk + logs

Day 7–11
AWS IAM + VPC + EC2 + EBS + monitoring/cost controls

Day 12–14
Java + JVM + Tomcat + external config + DB/TLS + rollback

Day 15–16
PostgreSQL + diagnostics

Day 17
Prometheus + node_exporter + Grafana + alerts

Day 18
change/release controls + evidence + rollback + handover

                    ↓

                  DAY 19
           Integrated system build

                    ↓

                  DAY 20
          Failure injection/troubleshooting
```

Đó là lý do Day 19 rất quan trọng: **Day 20 sẽ cố tình phá những dependency mà Day 19 yêu cầu bạn build đúng.**

---

## Kết luận Day 19

Ba artifact syllabus yêu cầu bạn phải hoàn thành là:

| Mandatory artifact          | Nội dung                                               |
| --------------------------- | ------------------------------------------------------ |
| **Architecture**            | hệ thống gồm những layer nào, traffic đi đâu           |
| **Configuration Inventory** | desired/actual version, port, service, resource, state |
| **Cost/Risk Checklist**     | cost drivers, exposure, security risks, cleanup        |

Còn kỹ năng cốt lõi cần mang sang Day 20 là:

```text
Expected state
      +
Actual evidence
      +
Dependency reasoning
      =
Operational troubleshooting
```

Một Day 19 đạt chuẩn không phải là **“tôi đã cài Java, PostgreSQL và Grafana”**, mà là **“tôi có thể chứng minh toàn bộ Linux → AWS → Java/Tomcat → PostgreSQL → monitoring đang tạo thành một hệ thống nhất quán, secure enough cho lab, có cost awareness, có rollback path và có evidence để người khác tiếp quản.”**
