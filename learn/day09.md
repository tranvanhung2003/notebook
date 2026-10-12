# Day 9 — Amazon EC2: Launch, Bootstrap, Secure SSH & Provisioning Evidence

Hôm nay chúng ta học **toàn bộ Day 9 theo đúng MASTER_SYLLABUS**, với mục tiêu không chỉ “biết tạo EC2”, mà phải hiểu đủ sâu để **launch → bootstrap → connect → validate → troubleshoot → recover → collect evidence → cleanup**.

## 1. Syllabus Alignment

MASTER_SYLLABUS quy định Day 9 chính xác như sau:

> **Concept/Lecture:** EC2 lifecycle, AMI, key pair, user data, instance metadata awareness, SSH access, and Linux bootstrap.  
> **Assignment/Lab:** Launch and bootstrap an EC2 Linux instance, connect securely, and submit provisioning evidence. 00_MASTER_SYLLABUS_Java_Fresher…

Đây là **mandatory scope**. Tôi sẽ không thay thế hoặc bỏ bất kỳ mục nào.

Phần **Supplementary / Beyond the explicit syllabus** hôm nay sẽ bổ sung những nội dung cần thiết cho công việc thực tế: IMDSv2 security, `cloud-init` troubleshooting, EC2 status checks, cost awareness, host-key verification, bootstrap idempotency, CLI provisioning, rollback/rebuild strategy và L1 escalation boundary.

---

# PHẦN I — LÝ THUYẾT

## 2. Objectives + Prerequisites

Kết thúc Day 9, bạn cần tự giải thích và thực hiện được các năng lực sau:

| Năng lực        | Mức cần đạt                                                                          |
| --------------- | ------------------------------------------------------------------------------------ |
| EC2             | Hiểu EC2 instance là gì và các thành phần tạo nên instance                           |
| Lifecycle       | Phân biệt `pending`, `running`, `stopping`, `stopped`, `shutting-down`, `terminated` |
| AMI             | Hiểu AMI là template gì, Region/architecture ảnh hưởng thế nào                       |
| Key pair        | Hiểu public/private key và cách SSH authentication hoạt động                         |
| User data       | Bootstrap Linux tự động bằng `cloud-init`/shell script                               |
| Metadata        | Biết IMDS là gì, truy vấn bằng IMDSv2 và hiểu rủi ro security                        |
| SSH             | Kết nối Linux instance an toàn                                                       |
| Linux bootstrap | Tự động cài package, cấu hình service và tạo baseline                                |
| Validation      | Chứng minh bootstrap hoàn thành thay vì “nhìn có vẻ chạy”                            |
| Troubleshooting | Phân biệt lỗi networking, SSH, key, username, `sshd`, `cloud-init`                   |
| Operations      | Hiểu reboot/stop/start/terminate và impact                                           |
| Evidence        | Thu thập provisioning evidence đủ cho ticket/handover                                |
| Cost            | Biết lúc nào resource vẫn phát sinh tiền                                             |

### Prerequisites từ Day 7–8

Bạn cần nhớ mental model từ hai ngày trước:

```text
AWS Account
   │
   ├── Region
   │
   └── VPC
        │
        ├── Subnet
        │    └── EC2 Instance
        │
        ├── Route Table
        │       └── 0.0.0.0/0 → Internet Gateway
        │
        ├── Security Group
        │       └── TCP/22 from your IP
        │
        └── Internet Gateway
```

Một EC2 có public IPv4 **không tự động đồng nghĩa SSH được**.

Đường đi đầy đủ còn phải hợp lệ:

```text
Your Laptop
   │
   │ TCP/22
   ▼
Internet
   │
   ▼
Internet Gateway
   │
   ▼
Route Table
   │
   ▼
Public Subnet
   │
   ▼
Security Group
   │
   ▼
EC2 Network Interface
   │
   ▼
Linux TCP/22
   │
   ▼
sshd
   │
   ▼
Linux user + ~/.ssh/authorized_keys
   │
   ▼
Private-key authentication
```

Đây là mental model troubleshooting rất quan trọng.

---

# 3. Mental Model — EC2 thực chất là gì?

Amazon EC2 — Elastic Compute Cloud — cung cấp **virtual server** trong AWS.

Khi launch một instance, bạn đang kết hợp nhiều configuration objects:

```text
                 +----------------------+
AMI ------------>|                      |
Instance type --->|                      |
Subnet ---------->|                      |
Security Group -->|      EC2 Instance    |
Key Pair -------->|                      |
User Data ------->|                      |
IAM Role -------->|                      |
EBS Volume ------>|                      |
Tags ------------>|                      |
                 +----------------------+
```

Có thể hình dung:

```text
AMI
 ↓
"Ổ đĩa khởi đầu"
 ↓
EC2 instance boot
 ↓
cloud-init
 ↓
network + SSH key + user data
 ↓
Linux operational
 ↓
Application/service bootstrap
```

AWS định nghĩa AMI là image chứa software cần để setup và boot instance. Khi launch EC2, bạn bắt buộc phải chọn một AMI. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html?utm_source=chatgpt.com)

### EC2 không phải chỉ là VM

Trong operations, một instance ít nhất gắn với:

```text
Compute
CPU / RAM
      │
      ├── Network interface
      ├── Private IP
      ├── Optional public IP
      ├── Security Groups
      ├── EBS root volume
      ├── AMI ancestry
      ├── Key-pair association
      ├── Metadata configuration
      ├── Optional IAM instance profile
      └── Tags
```

Vì thế câu:

> “EC2 bị lỗi.”

chưa phải chẩn đoán.

Bạn phải hỏi:

```text
Instance không running?
OS không boot?
Network không reachable?
Port 22 không mở?
sshd không listen?
Authentication fail?
cloud-init fail?
Application fail?
```

Đó là cách tư duy của Administrator.

---

# 4. Deep Theory

## 4.1 EC2 Lifecycle

AWS hiện định nghĩa các state chính:

| State           | Ý nghĩa                      | EC2 instance usage billing |
| --------------- | ---------------------------- | -------------------------: |
| `pending`       | Đang chuẩn bị start          |                      Không |
| `running`       | VM đang chạy                 |                         Có |
| `stopping`      | Đang stop                    |         Thông thường không |
| `stopped`       | VM đã tắt                    |         Không tính compute |
| `shutting-down` | Đang terminate               |                      Không |
| `terminated`    | Instance đã bị xóa vĩnh viễn |                      Không |

EBS và một số tài nguyên liên quan có thể tiếp tục phát sinh phí dù instance đã `stopped`. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html?utm_source=chatgpt.com)

Mental model:

```text
Launch
   │
   ▼
pending
   │
   ▼
running
   │
   ├──────── reboot ──────────┐
   │                          │
   │                          ▼
   │                       running
   │
   ├── stop → stopping → stopped
   │                         │
   │                         │ start
   │                         ▼
   │                      pending
   │                         │
   │                         ▼
   │                      running
   │
   └── terminate → shutting-down → terminated
```

### Reboot

`reboot` gần giống restart operating system.

Thông thường:

```text
Instance ID     giữ nguyên
Private IPv4    giữ nguyên
Public IPv4     giữ nguyên
EBS             giữ nguyên
Instance store  giữ nguyên qua reboot
```

AWS khuyến nghị có thể dùng EC2 reboot operation thay vì chỉ phụ thuộc OS command. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html?utm_source=chatgpt.com)

### Stop → Start

Đây không giống reboot.

Khi stop/start EBS-backed instance:

```text
Instance ID          giữ nguyên
EBS root volume      giữ nguyên
Private IPv4         giữ nguyên
Public IPv4          thường thay đổi
Host vật lý          có thể thay đổi
RAM                   mất
Instance-store data  mất
```

Do đó nếu ứng dụng/configuration hard-code public IP thì rất dễ gặp sự cố sau stop/start. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html?utm_source=chatgpt.com)

### Terminate

`terminate` là destructive.

```text
running
   ↓
shutting-down
   ↓
terminated
```

Bạn **không thể start lại terminated instance**.

Root EBS volume mặc định thường có `DeleteOnTermination=true`; việc preserve hay delete volume cần kiểm tra chứ không được phỏng đoán. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html?utm_source=chatgpt.com)

### Góc nhìn Operations

Không được học theo kiểu:

```text
Restart thử xem.
```

Mà phải hỏi:

```text
Reboot có giải quyết đúng failing layer không?

Stop/start:
- public IP có đổi không?
- instance-store có dữ liệu không?
- downtime được approve chưa?

Terminate:
- dữ liệu có cần preserve không?
- evidence đã thu chưa?
- resource có thể rebuild không?
```

---

## 4.2 AMI — Amazon Machine Image

### AMI là gì?

AMI là **template để khởi tạo EC2 instance**.

Nó chứa software configuration cần để boot OS và block-device mapping cần thiết. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html?utm_source=chatgpt.com)

Mental model:

```text
             AMI
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
     EC2-A  EC2-B  EC2-C
```

Một AMI có thể launch nhiều EC2.

### AMI ≠ EC2 instance

```text
AMI                    EC2 Instance
----------------       -------------------------
Template                Running machine
Không phải workload     Có CPU/RAM/network runtime
Source để boot          Được tạo từ AMI
Reusable                Lifecycle riêng
```

### AMI phụ thuộc Region

AMI ID không phải global.

Ví dụ tưởng tượng:

```text
ap-southeast-1 → ami-AAA...
us-east-1      → ami-BBB...
```

Cùng một loại Amazon Linux nhưng ID có thể khác Region. AWS xác nhận AMI là Region-specific và AMI ID là unique theo Region. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html?utm_source=chatgpt.com)

Vì vậy một lỗi phổ biến là copy:

```bash
ami-0123456789abcdef0
```

từ blog cũ rồi dùng ở Region khác.

### AMI và architecture

AMI còn liên quan đến processor architecture:

```text
x86_64
arm64
```

Instance type phải tương thích.

Ví dụ:

```text
x86_64 AMI → x86_64 instance family
arm64 AMI  → Graviton-compatible instance
```

### Source trust

Trong production không nên chọn một public AMI ngẫu nhiên chỉ vì tên trông giống Amazon Linux.

Phải kiểm tra:

```text
Publisher/Owner
AMI ID
OS
Architecture
Region
Creation/version
Approved baseline
```

### Lab Day 9

Chúng ta dùng:

```text
Amazon Linux 2023
Architecture: x86_64
Publisher: Amazon
```

**Current-information note:** Thay vì hard-code AMI ID, AWS cung cấp SSM public parameter:

```text
/aws/service/ami-amazon-linux-latest/
al2023-ami-kernel-default-x86_64
```

Parameter này resolve sang AMI hiện hành của Region. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami-parameter-store.html?utm_source=chatgpt.com)

Tại thời điểm hiện tại, AL2023 `kernel-default` cho instance mới đã chuyển sang kernel 6.18. Instance đã chạy trước đó không tự động đổi kernel chỉ vì default AMI thay đổi. [AWS Documentation](https://docs.aws.amazon.com/linux/al2023/ug/ec2.html?utm_source=chatgpt.com)

Đây là lý do trong Operations chúng ta ghi lại **actual AMI ID và `uname -r`**, không chỉ ghi “Amazon Linux 2023”.

---

## 4.3 EC2 Key Pair

SSH key authentication dựa trên một cặp:

```text
Private key               Public key
────────────               ──────────
Giữ tại client             Được đặt trên server
Không chia sẻ              Có thể phân phối
Dùng để chứng minh         Server dùng để verify
quyền sở hữu
```

AWS EC2 hỗ trợ RSA và ED25519 cho Linux; Windows EC2 không hỗ trợ ED25519 theo cơ chế key pair tương ứng. [AWS Documentation](https://docs.aws.amazon.com/he_il/AWSEC2/latest/UserGuide/create-key-pairs.html?utm_source=chatgpt.com)

### Khi launch EC2

Bạn chọn một EC2 key pair.

Sau khi boot, public key tương ứng được đặt vào Linux account, ví dụ:

```text
/home/ec2-user/.ssh/authorized_keys
```

AWS mô tả quá trình này: khi instance boot lần đầu, public key của key pair được đặt vào `~/.ssh/authorized_keys`. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/replacing-key-pair.html?utm_source=chatgpt.com)

Sau đó:

```text
Your laptop
 private key
     │
     │ SSH authentication
     ▼
EC2
 ~/.ssh/authorized_keys
 public key
```

Private key **không phải password gửi lên server**.

### Permission private key

Trên Linux/macOS:

```bash
chmod 400 day9-key.pem
```

AWS yêu cầu private key không được readable quá rộng; nếu permission không đủ chặt SSH client có thể từ chối dùng key. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs-general.html?utm_source=chatgpt.com)

Kiểm tra:

```bash
ls -l day9-key.pem
```

Ví dụ:

```text
-r-------- 1 alice alice ... day9-key.pem
```

### Tuyệt đối không làm

Không:

```bash
cat day9-key.pem
```

rồi gửi nội dung cho tôi.

Không upload private key vào Git.

Không gửi qua chat/team channel.

Không:

```bash
chmod 777 day9-key.pem
```

Không dùng một private key chung cho cả team nếu có cách quản trị access tốt hơn.

### Mất private key

Nếu mất private key, AWS không đơn giản “download lại” cùng private key đó cho bạn.

Bạn cần alternative access path để inject một public key mới hoặc recovery procedure. AWS cũng cảnh báo mất private key có thể làm mất đường truy cập vào các instance sử dụng key đó. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/replacing-key-pair.html?utm_source=chatgpt.com)

---

## 4.4 User Data

`user data` cho phép đưa instructions vào instance lúc launch.

Ví dụ:

```bash
#!/bin/bash

dnf install -y httpd
systemctl enable --now httpd
```

Amazon Linux sử dụng `cloud-init` để xử lý bootstrap. AL2023 dùng `cloud-init` để cấu hình nhiều phần ban đầu, bao gồm account `ec2-user`, SSH public keys và user data. [AWS Documentation](https://docs.aws.amazon.com/linux/al2023/ug/cloud-init.html?utm_source=chatgpt.com)

Flow:

```text
Launch EC2
   │
   ▼
Linux boot
   │
   ▼
cloud-init starts
   │
   ├── configure user
   ├── install SSH public key
   ├── process metadata
   └── execute user data
            │
            ▼
       Bootstrap system
```

### Một đặc điểm cực kỳ quan trọng

Mặc định user data của Linux instance chạy **ở initial launch**, không tự động chạy lại mỗi lần reboot. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html?utm_source=chatgpt.com)

Do đó:

```text
Launch       → user data runs
Reboot       → normally NOT rerun
Stop/start   → normally NOT rerun
```

Đây là câu hỏi exam/troubleshooting rất hay gặp.

### User data chạy với quyền root

AWS user-data shell script chạy với quyền `root`. Vì vậy trong script:

```bash
dnf install -y httpd
```

không cần:

```bash
sudo dnf install ...
```

AWS cũng lưu ý script không chạy interactive, vì vậy commands cần confirmation phải có option như `-y`. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html?utm_source=chatgpt.com)

### Bootstrap script cần non-interactive

Sai:

```bash
dnf install httpd
```

Có thể chờ prompt.

Đúng:

```bash
dnf install -y httpd
```

### Logs

Một log rất quan trọng:

```text
/var/log/cloud-init-output.log
```

AWS chỉ rõ đây là nơi hữu ích để debug output của user-data script. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html?utm_source=chatgpt.com)

Kiểm tra:

```bash
sudo tail -n 100 /var/log/cloud-init-output.log
```

Và:

```bash
sudo cloud-init status --long
```

### Security rule rất quan trọng

**Không đặt secrets vào user data.**

Không:

```bash
DB_PASSWORD=ProductionPassword123
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
```

User data có thể được retrieve thông qua instance metadata nếu được cấu hình, và cloud-init cũng giữ artifacts cục bộ trên instance. AWS tài liệu hóa endpoint `/latest/user-data` và lưu script dưới `/var/lib/cloud/instances/<instance-id>/`. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html?utm_source=chatgpt.com)

Secret management sẽ được học sâu hơn khi chúng ta sang Java/PostgreSQL.

---

## 4.5 Instance Metadata Service — IMDS

### Instance metadata là gì?

Một EC2 instance cần biết thông tin về chính nó, chẳng hạn:

```text
Instance ID
AMI ID
Instance type
Private IP
Availability Zone
IAM role information
User data
```

AWS cung cấp endpoint local/link-local:

```text
169.254.169.254
```

Metadata endpoint IPv4:

```text
http://169.254.169.254/latest/meta-data/
```

Đây **không phải một public Internet website**. Nó được truy cập từ bên trong instance. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html?utm_source=chatgpt.com)

### IMDSv1 và IMDSv2

IMDSv1:

```text
GET metadata directly
```

IMDSv2:

```text
1. PUT → obtain token
2. GET → send token
```

Mental model:

```text
EC2 instance
     │
     │ PUT /latest/api/token
     ▼
   IMDS
     │
     │ token
     ▼
Instance
     │
     │ GET metadata + token
     ▼
   IMDS
     │
     ▼
 metadata
```

AWS cho phép và khuyến nghị cấu hình instance để **require IMDSv2**, tức:

```text
HttpTokens = required
```

Nếu ứng dụng cũ chỉ hỗ trợ IMDSv1, việc enforce IMDSv2 có thể làm ứng dụng đó hỏng, nên production phải test trước. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-options.html?utm_source=chatgpt.com)

### IMDSv2 request

Lấy token:

```bash
TOKEN=$(curl -sS -X PUT \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" \
  http://169.254.169.254/latest/api/token)
```

Lấy instance ID:

```bash
curl -sS \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

Đây là pattern chính thức của AWS. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html?utm_source=chatgpt.com)

### Vì sao metadata nhạy cảm?

Nếu instance có IAM role, metadata có thể liên quan tới temporary AWS credentials.

Vì vậy:

```text
SSRF vulnerability
       │
       ▼
Application can access IMDS
       │
       ▼
Potential access to role credentials
```

là một security concern nghiêm trọng.

IMDSv2 là một phần của defense-in-depth, không phải phép màu làm application hết vulnerability.

---

## 4.6 SSH Access

SSH = Secure Shell.

Một command điển hình:

```bash
ssh -i day9-key.pem ec2-user@<PUBLIC_IP>
```

Amazon Linux dùng username mặc định:

```text
ec2-user
```

Ubuntu thường:

```text
ubuntu
```

AWS xác nhận username phụ thuộc AMI; Amazon Linux dùng `ec2-user`. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/managing-users.html?utm_source=chatgpt.com)

### SSH authentication flow

```text
Client
  │
  │ TCP SYN :22
  ▼
EC2
  │
  │ sshd replies
  ▼
SSH handshake
  │
  ├── host authentication
  │
  └── user authentication
          │
          ▼
   ec2-user authorized_keys
          │
          ▼
 private/public key verification
          │
          ▼
        shell
```

### Điều kiện để SSH thành công

AWS yêu cầu instance đã sẵn sàng/status check phù hợp, security group cho phép TCP/22 từ client IP và bạn có private key/instance information đúng. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-to-linux-instance.html?utm_source=chatgpt.com)

Nhưng trong troubleshooting chúng ta phải suy nghĩ rộng hơn:

```text
Instance running?
       ↓
Status checks pass?
       ↓
Public IPv4 exists?
       ↓
Subnet route → IGW?
       ↓
SG allows source-IP:22?
       ↓
NACL permits traffic?
       ↓
Local network permits outbound SSH?
       ↓
sshd running/listening?
       ↓
Correct username?
       ↓
Correct key?
```

---

## 4.7 Linux Bootstrap

Bootstrap nghĩa là đưa một machine mới từ:

```text
Generic OS
```

sang:

```text
Minimum operational baseline
```

Ví dụ:

```text
Generic AL2023
      │
      ├── hostname
      ├── packages
      ├── users
      ├── directories
      ├── configuration
      ├── service enable/start
      └── bootstrap marker
            │
            ▼
        Ready host
```

Trong production bootstrap có thể phức tạp hơn với configuration management, Launch Templates, Image Builder, Ansible, SSM, etc.

**Supplementary / Beyond explicit syllabus:** Nguyên tắc tốt là bootstrap càng deterministic và repeatable càng tốt. Nếu một server chỉ có thể tạo đúng bằng cách “SSH vào rồi sửa tay 27 thứ”, đó là configuration drift risk.

---

# 5. Commands / Configuration

## 5.1 Các Linux commands quan trọng hôm nay

| Command                    | Mục đích                       |
| -------------------------- | ------------------------------ |
| `hostnamectl`              | Kiểm tra/set hostname          |
| `uname -a`                 | Kernel/system information      |
| `cat /etc/os-release`      | OS release                     |
| `systemctl status sshd`    | Kiểm tra SSH daemon            |
| `ss -lntp`                 | Kiểm tra listening TCP sockets |
| `systemctl status httpd`   | Kiểm tra bootstrap service     |
| `curl`                     | Validate local HTTP/metadata   |
| `cloud-init status --long` | Bootstrap state                |
| `journalctl`               | Service/system logs            |
| `tail`                     | Xem log cuối file              |

### Expected invariant

Sau lab:

```text
sshd is active
TCP/22 is listening
httpd is active
HTTP localhost returns expected page
cloud-init completed
IMDSv2 works
```

Đó mới là evidence.

---

# 6. Practical Example — Bootstrap Script

Chúng ta sẽ dùng script này trong guided lab:

```bash
#!/bin/bash

set -euo pipefail

exec >> /var/log/day9-bootstrap.log 2>&1

echo "=== Day 9 bootstrap started: $(date -Is) ==="

hostnamectl set-hostname day9-ec2

dnf install -y httpd

cat > /var/www/html/index.html <<'EOF'
<!doctype html>
<html>
<head>
  <title>Day 9 EC2 Lab</title>
</head>
<body>
  <h1>Day 9 EC2 bootstrap successful</h1>
  <p>Managed through EC2 user data.</p>
</body>
</html>
EOF

systemctl enable --now httpd

install -d -m 0755 /var/local

cat > /var/local/day9-bootstrap-complete <<EOF
bootstrap_status=success
completed_at=$(date -Is)
EOF

echo "=== Day 9 bootstrap completed: $(date -Is) ==="
```

### Phân tích

`#!/bin/bash`

Xác định interpreter.

```bash
set -euo pipefail
```

Tăng độ an toàn cho script.

`-e`: command lỗi có thể làm script stop.

`-u`: dùng undefined variable được coi là lỗi.

`pipefail`: pipeline phản ánh failure tốt hơn.

Ta **không dùng `set -x`** vì trong real operations nó có thể vô tình log dữ liệu nhạy cảm.

```bash
exec >> /var/log/day9-bootstrap.log 2>&1
```

Redirect `stdout` và `stderr` sang log riêng.

```bash
hostnamectl set-hostname day9-ec2
```

Thiết lập hostname.

```bash
dnf install -y httpd
```

Cài Apache HTTP Server non-interactively.

```bash
systemctl enable --now httpd
```

Hai hành vi:

```text
enable → start automatically on future boots
--now  → start ngay bây giờ
```

Marker:

```text
/var/local/day9-bootstrap-complete
```

cho phép chúng ta chứng minh bootstrap đã đi đến cuối.

---

# PHẦN II — GUIDED LAB DAY 9

# 7. Guided Lab — Launch, Bootstrap, SSH, Validate

## 7.1 Objective

Bạn sẽ xây:

```text
                    Internet
                       │
                       │ SSH/22 from YOUR IP only
                       ▼
               Internet Gateway
                       │
                       ▼
                Public Subnet
                       │
                       ▼
              +----------------+
              | EC2 AL2023     |
              |                |
              | sshd :22       |
              | httpd :80      |
              |                |
              | User Data      |
              | cloud-init     |
              +----------------+
```

HTTP port 80 **không cần public inbound** trong lab này. Chúng ta validate Apache từ bên trong host bằng localhost.

Điều này giảm attack surface.

---

## 7.2 Assumptions

Lab environment:

| Setting        | Giá trị                                 |
| -------------- | --------------------------------------- |
| OS             | Amazon Linux 2023                       |
| Architecture   | x86_64                                  |
| VPC            | VPC Day 8 hoặc lab VPC được phép        |
| Subnet         | Public subnet                           |
| Internet route | `0.0.0.0/0 → IGW`                       |
| SSH            | TCP/22 từ **your current public IP/32** |
| Metadata       | IMDSv2 required                         |
| Public IPv4    | Enabled cho lab SSH                     |
| Key            | Dedicated Day 9 key pair                |
| IAM role       | Không cần cho lab cơ bản                |
| Workload       | Apache chỉ validate locally             |

---

## 7.3 COST & SAFETY WARNING

**Lab này tạo AWS resources có thể phát sinh chi phí.**

AWS Free Tier hiện phụ thuộc ngày tạo account và account eligibility. Với account tạo từ July 15, 2025 trở đi, Free Tier model khác các tài liệu AWS cũ và dựa trên eligibility/credits. Không được mặc định rằng `t2.micro` hay một instance bất kỳ luôn miễn phí. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html?utm_source=chatgpt.com)

Chọn instance type được AWS Console đánh dấu phù hợp với Free Tier/credits của **account của bạn**, nếu có.

Ngoài compute:

```text
EBS storage
Public IPv4
Snapshots
Elastic IP
Data transfer
```

có thể tạo cost riêng.

AWS hiện công bố public IPv4 address charge là **$0.005/giờ/address** đối với in-use hoặc idle public IPv4 trong phạm vi pricing đó. [Amazon Web Services, Inc.](https://aws.amazon.com/vpc/pricing/?trk=public_post_comment-text&utm_source=chatgpt.com)

Sau khi lab xong chúng ta sẽ **terminate** resource lab và kiểm tra cleanup.

---

# 8. Lab Step-by-Step

## Step A — Pre-flight inspection

Trước khi tạo instance, xác nhận:

```text
Region đúng
VPC đúng
Subnet đúng
Subnet có route → IGW
SG không mở SSH 0.0.0.0/0
Key location an toàn
Budget/cost awareness
```

Bạn có thể ghi baseline:

```text
Region:
VPC ID:
Subnet ID:
Route Table ID:
Security Group ID:
Your public IP:
```

Không ghi private key vào evidence.

---

## Step B — Security Group

Tạo hoặc dùng security group riêng:

```text
Name:
day9-ec2-sg
```

Inbound:

| Type | Protocol | Port | Source                |
| ---- | -------- | ---: | --------------------- |
| SSH  | TCP      |   22 | `<YOUR_PUBLIC_IP>/32` |

Không sử dụng:

```text
0.0.0.0/0
```

cho SSH nếu không có lý do được approve.

Outbound cho lab có thể giữ default outbound policy để instance truy cập package repositories.

### Why `/32`?

Ví dụ:

```text
203.0.113.25/32
```

có nghĩa chỉ đúng một IPv4 source.

---

## Step C — Key Pair

EC2 Console:

```text
EC2
→ Network & Security
→ Key Pairs
→ Create key pair
```

Ví dụ:

```text
Name: day9-lab-key
Type: ED25519
Format: .pem
```

Linux hỗ trợ ED25519. [AWS Documentation](https://docs.aws.amazon.com/he_il/AWSEC2/latest/UserGuide/create-key-pairs.html?utm_source=chatgpt.com)

Lưu private key ở nơi an toàn.

Linux/macOS:

```bash
chmod 400 ~/keys/day9-lab-key.pem
```

Validate:

```bash
stat ~/keys/day9-lab-key.pem
```

hoặc:

```bash
ls -l ~/keys/day9-lab-key.pem
```

Không gửi file này cho bất kỳ ai.

---

## Step D — Launch Instance

EC2 Console:

```text
EC2
→ Instances
→ Launch instances
```

Tên:

```text
day9-al2023-lab
```

AMI:

```text
Amazon Linux 2023
Publisher: Amazon
Architecture: x86_64
```

Đừng chỉ nhìn chữ “Amazon Linux”; kiểm tra publisher.

Instance type:

```text
Chọn loại phù hợp lab/account/Free Tier eligibility.
```

Key pair:

```text
day9-lab-key
```

Network:

```text
VPC:      Day 8 VPC
Subnet:   public subnet
Public IP: enabled
SG:       day9-ec2-sg
```

Storage:

Giữ lab root volume nhỏ/hợp lý theo policy/console mặc định được phê duyệt. Day 10 sẽ học EBS sâu.

---

## Step E — Metadata Options

Trong **Advanced details**, tìm Instance Metadata options.

Cho lab:

```text
Metadata accessible:
Enabled

Metadata version:
V2 only / token required
```

Ở AWS CLI tương đương về nguyên tắc:

```text
HttpEndpoint=enabled
HttpTokens=required
```

AWS hỗ trợ enforce IMDSv2 ngay lúc launch và document `HttpTokens=required` cho use case này. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-IMDS-new-instances.html?utm_source=chatgpt.com)

Không tắt metadata endpoint trong lab vì mục tiêu syllabus yêu cầu metadata awareness.

---

## Step F — User Data

Trong Advanced details → User data, paste:

```bash
#!/bin/bash

set -euo pipefail

exec >> /var/log/day9-bootstrap.log 2>&1

echo "=== Day 9 bootstrap started: $(date -Is) ==="

hostnamectl set-hostname day9-ec2

dnf install -y httpd

cat > /var/www/html/index.html <<'EOF'
<!doctype html>
<html>
<head>
  <title>Day 9 EC2 Lab</title>
</head>
<body>
  <h1>Day 9 EC2 bootstrap successful</h1>
  <p>Managed through EC2 user data.</p>
</body>
</html>
EOF

systemctl enable --now httpd

install -d -m 0755 /var/local

cat > /var/local/day9-bootstrap-complete <<EOF
bootstrap_status=success
completed_at=$(date -Is)
EOF

echo "=== Day 9 bootstrap completed: $(date -Is) ==="
```

Không thêm passwords, access keys hoặc DB secrets.

---

## Step G — Tags

Day 7 đã học tagging nên hãy giữ discipline.

Ví dụ:

```text
Name         = day9-al2023-lab
Environment  = lab
Purpose      = java-fresher-day9
Owner        = <approved identifier>
```

Không cho secret vào tag.

---

## Step H — Launch

Chọn:

```text
Launch instance
```

Observe lifecycle:

```text
pending
   ↓
running
```

**Không SSH ngay lập tức rồi kết luận lỗi.**

Instance có thể đã `running` nhưng:

```text
cloud-init
package installation
service setup
```

vẫn chưa hoàn tất.

AWS lưu ý user-data tasks có thể làm boot kéo dài thêm và cần chờ chúng hoàn thành trước khi test. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html?utm_source=chatgpt.com)

---

# 9. Verification — AWS Layer

## 9.1 State

Console phải hiển thị:

```text
Instance state:
Running
```

## 9.2 Status checks

Kiểm tra Status and alarms.

AWS hiện có system, instance, attached EBS và opt-in application status checks. System/instance/EBS checks chạy tự động; system status liên quan underlying AWS host/infrastructure, còn instance status tập trung vào guest/network configuration của instance. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html?utm_source=chatgpt.com)

Đối với lab, trước SSH chúng ta muốn:

```text
System status:   Passed / OK
Instance status: Passed / OK
```

Một điểm quan trọng:

> **Running ≠ fully healthy.**

Instance có thể:

```text
State: running
Status check: impaired
```

---

# 10. Secure SSH Connection

Lấy:

```text
Public IPv4 address
```

Không paste private key lên server.

Từ client:

```bash
ssh -i ~/keys/day9-lab-key.pem ec2-user@<PUBLIC_IP>
```

Ví dụ hình thức:

```bash
ssh -i ~/keys/day9-lab-key.pem ec2-user@203.0.113.10
```

Amazon Linux dùng `ec2-user`. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/managing-users.html?utm_source=chatgpt.com)

### First connection

Có thể thấy:

```text
The authenticity of host ... can't be established.
...
Are you sure you want to continue connecting?
```

Đây là **server host-key verification**, khác với EC2 key pair dùng để authenticate user.

Trong production không nên hình thành thói quen:

```text
"thấy prompt là gõ yes"
```

AWS có hướng dẫn verification fingerprint để giảm nguy cơ man-in-the-middle. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs-general.html?utm_source=chatgpt.com)

---

# 11. Verification — Inside Linux

Sau khi login:

```bash
whoami
```

Expected invariant:

```text
ec2-user
```

OS:

```bash
cat /etc/os-release
```

Kernel:

```bash
uname -r
```

Hostname:

```bash
hostnamectl
```

Expected:

```text
day9-ec2
```

Uptime:

```bash
uptime
```

Network:

```bash
ip addr
```

Routes:

```bash
ip route
```

SSH listener:

```bash
sudo ss -lntp | grep ':22'
```

Expected invariant:

```text
a LISTEN socket exists for TCP/22
```

Service:

```bash
sudo systemctl status sshd --no-pager
```

Expected invariant:

```text
active (running)
```

---

# 12. Verify cloud-init / User Data

Status:

```bash
sudo cloud-init status --long
```

Bootstrap log:

```bash
sudo cat /var/log/day9-bootstrap.log
```

Marker:

```bash
sudo cat /var/local/day9-bootstrap-complete
```

Expected:

```text
bootstrap_status=success
completed_at=...
```

AWS cloud-init output:

```bash
sudo tail -n 100 /var/log/cloud-init-output.log
```

AWS tài liệu hóa `/var/log/cloud-init-output.log` là nơi cần kiểm tra khi user-data không làm đúng những gì bạn dự kiến. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html?utm_source=chatgpt.com)

### Apache

```bash
sudo systemctl status httpd --no-pager
```

Expected:

```text
active (running)
```

Check enabled:

```bash
sudo systemctl is-enabled httpd
```

Expected:

```text
enabled
```

Listener:

```bash
sudo ss -lntp | grep ':80'
```

HTTP:

```bash
curl -i http://127.0.0.1/
```

Expected invariant:

```text
HTTP response successful
```

Body phải chứa:

```text
Day 9 EC2 bootstrap successful
```

Lưu ý:

> HTTP localhost thành công **không chứng minh Internet client truy cập port 80 được**.

Nó chỉ chứng minh:

```text
httpd running
+
local TCP stack works
+
application responds
```

Đây là distinction troubleshooting quan trọng.

---

# 13. Verify IMDSv2

Lấy token:

```bash
TOKEN=$(curl -sS -X PUT \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" \
  http://169.254.169.254/latest/api/token)
```

Không cần print token.

Lấy instance ID:

```bash
curl -sS \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

AMI ID:

```bash
curl -sS \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/ami-id
```

Instance type:

```bash
curl -sS \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-type
```

Private IPv4:

```bash
curl -sS \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/local-ipv4
```

Availability Zone:

```bash
curl -sS \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone
```

AWS document pattern `PUT /latest/api/token` rồi gửi `X-aws-ec2-metadata-token` trên GET request. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html?utm_source=chatgpt.com)

### Test rằng IMDSv1 không được dùng

Bạn có thể thử request metadata **không kèm token**:

```bash
curl -sS \
  http://169.254.169.254/latest/meta-data/instance-id
```

Nếu instance được cấu hình V2-only thì request tokenless không được phép thành công.

Đừng chỉ nhìn metadata hoạt động; hãy chứng minh configuration đúng security requirement.

---

# 14. Failure Injection & Troubleshooting

Đây là phần rất quan trọng của Day 9.

Mental model chuẩn:

```text
Symptom
   ↓
Expected behavior
   ↓
Recent changes
   ↓
Scope
   ↓
Failing layer
   ↓
Evidence
   ↓
Hypotheses
   ↓
Safest test
   ↓
Root cause
   ↓
Remediation
   ↓
Validation
   ↓
Evidence / prevention
```

## Case 1 — SSH timeout

Symptom:

```text
ssh: connect to host ... port 22: Connection timed out
```

Không được kết luận:

```text
"SSH key sai."
```

Key authentication chưa chắc đã xảy ra.

Think layer-by-layer:

```text
Client
 ↓
Internet
 ↓
Public IP
 ↓
IGW
 ↓
Route table
 ↓
Subnet
 ↓
SG
 ↓
NACL
 ↓
EC2
 ↓
sshd
```

Evidence:

```text
Instance running?
Status checks passed?
Public IPv4 correct?
SG TCP/22 source correct?
Route 0.0.0.0/0 → IGW?
Local network blocks outbound 22?
```

Một lỗi cực phổ biến:

```text
SG source = old public IP /32
```

nhưng ISP/client public IP đã thay đổi.

Remediation:

Cập nhật rule theo change/scope được approve.

Không:

```text
temporarily open 22 to 0.0.0.0/0 forever
```

---

## Case 2 — Connection refused

Symptom:

```text
Connection refused
```

Điểm khác timeout:

```text
timeout
≈ connection không đi đến nơi / bị drop

refused
≈ target reachable nhưng port không có listener
   hoặc có explicit rejection
```

Trên EC2 nếu có alternate access path:

```bash
sudo systemctl status sshd
```

```bash
sudo ss -lntp | grep ':22'
```

Logs:

```bash
sudo journalctl -u sshd --no-pager
```

---

## Case 3 — `Permission denied (publickey)`

Symptom:

```text
Permission denied (publickey).
```

Điều này thường cho thấy network/port/sshd đã đi xa hơn timeout.

Investigate:

```text
Correct username?
Correct private key?
Correct instance?
Was this key selected during launch?
Is authorized_keys correct?
Permissions correct?
```

Ví dụ:

Sai:

```bash
ssh -i day9-key.pem root@...
```

Amazon Linux thường:

```bash
ssh -i day9-key.pem ec2-user@...
```

Hoặc bạn đang dùng:

```text
day8-key.pem
```

trong khi instance được launch bằng:

```text
day9-key.pem
```

---

## Case 4 — `UNPROTECTED PRIVATE KEY FILE`

SSH có thể báo private key permissions quá rộng.

Inspect:

```bash
ls -l day9-lab-key.pem
```

Fix Linux/macOS:

```bash
chmod 400 day9-lab-key.pem
```

AWS hướng dẫn sử dụng permission chặt tương ứng. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs-general.html?utm_source=chatgpt.com)

---

## Case 5 — Host key changed

Có thể gặp:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

Không được phản xạ ngay:

```bash
ssh-keygen -R ...
```

rồi tiếp tục.

Có thể chỉ là lab instance cũ bị terminate và public IP được reuse cho instance mới.

Nhưng cũng có thể là:

```text
wrong host
DNS/IP issue
MITM concern
```

Quy trình:

```text
Verify target identity
→ confirm instance replacement/change
→ verify fingerprint
→ only then update known_hosts
```

---

## Case 6 — Instance `running` nhưng user data chưa hoàn thành

Inspect:

```bash
sudo cloud-init status --long
```

```bash
sudo tail -n 200 /var/log/cloud-init-output.log
```

```bash
sudo cat /var/log/day9-bootstrap.log
```

Look for:

```text
package download failure
DNS resolution failure
permission/path error
invalid command
repository unavailable
syntax error
```

AWS nhấn mạnh `running` có thể xảy ra trước khi toàn bộ bootstrap workload của bạn hoàn thành. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html?utm_source=chatgpt.com)

---

## Case 7 — `dnf install httpd` fail

Hypotheses:

```text
No Internet route
No public IP/NAT
DNS issue
Outbound SG restrictions
Repository problem
Package name wrong for distro
```

Evidence:

```bash
ip route
```

```bash
cat /etc/resolv.conf
```

```bash
curl -I https://aws.amazon.com
```

và cloud-init log.

Không ngay lập tức modify random networking settings.

---

## Case 8 — httpd installed nhưng không chạy

Inspect:

```bash
sudo systemctl status httpd --no-pager
```

```bash
sudo journalctl -u httpd -n 100 --no-pager
```

```bash
sudo ss -lntp | grep ':80'
```

Distinguish:

```text
Package installed
≠
service running
≠
service listening
≠
application responds
≠
remote client can access
```

Đây là một chain rất quan trọng:

```text
Installed
   ↓
Started
   ↓
Listening
   ↓
Local response
   ↓
Network reachable
   ↓
Remote response
```

---

## Case 9 — Metadata request không hoạt động

Nếu request không token bị fail nhưng token request hoạt động:

```text
GOOD:
IMDSv2 is being enforced.
```

Nếu token request fail:

Investigate:

```text
Metadata endpoint enabled?
URL exact?
IMDS disabled?
Firewall/local routing issue?
```

Nếu chạy application trong container và metadata access có vấn đề thì IMDS hop limit có thể liên quan; AWS đặc biệt document hop-limit considerations cho container environments. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-IMDS-new-instances.html?utm_source=chatgpt.com)

Đây là supplementary detail; lab hiện tại không cần container.

---

## Case 10 — EC2 status check fail

### System status check fail

Có thể liên quan:

```text
AWS host/network/power/infrastructure
```

AWS nói system status failures là các vấn đề underlying system thường cần AWS xử lý hoặc có thể được workaround bằng stop/start trên EBS-backed instance. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html?utm_source=chatgpt.com)

### Instance status check fail

Có thể liên quan:

```text
guest OS
startup config
memory exhaustion
filesystem
kernel
network config
```

AWS phân biệt đây là các vấn đề thường cần operator can thiệp. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html?utm_source=chatgpt.com)

### Không stop/start mù quáng

Trước stop/start:

```text
Is this production?
Public IPv4 impact?
Instance-store data?
Downtime approved?
Evidence collected?
```

---

# 15. Mistakes / Anti-Patterns

| Anti-pattern                       | Vì sao sai                                         |
| ---------------------------------- | -------------------------------------------------- |
| SSH `0.0.0.0/0`                    | Tăng attack surface                                |
| Share `.pem`                       | Private-key compromise                             |
| `chmod 777 key.pem`                | Key readable quá rộng                              |
| Hard-code random AMI ID            | Region/version mismatch                            |
| Chọn public AMI chỉ theo tên       | Supply-chain/trust risk                            |
| User data chứa password            | Secret exposure risk                               |
| User data interactive              | Có thể treo bootstrap                              |
| Assume user data runs every reboot | Mặc định không đúng                                |
| `running` = ready                  | `cloud-init` có thể chưa xong                      |
| Test service chỉ bằng `ps`         | Process tồn tại chưa chứng minh endpoint hoạt động |
| Thấy timeout rồi đổi key           | Không đúng failing layer                           |
| Delete `known_hosts` warning ngay  | Có thể bỏ qua MITM signal                          |
| Stop/start để “fix mọi thứ”        | Có downtime/IP/data implications                   |
| Terminate trước khi lấy evidence   | Mất khả năng điều tra                              |
| Không cleanup lab                  | Cost/risk                                          |

---

# 16. Security & Least Privilege

Security baseline Day 9:

```text
Internet exposure
        │
        ▼
TCP/22 only
        │
        ▼
My public IP /32
        │
        ▼
SSH key authentication
        │
        ▼
ec2-user
        │
        ▼
sudo only when needed
```

Metadata:

```text
IMDS endpoint enabled
+
IMDSv2 token required
```

Secrets:

```text
No secret in:
user data
tags
shell history
screenshots
evidence pack
Git
chat
```

IAM:

Lab không cần application AWS API access, vậy **không attach broad IAM role chỉ để “cho tiện”**.

Nếu sau này application cần access AWS:

```text
EC2 IAM Role
+
least-privilege policy
```

thay vì đặt:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

trên server.

---

# 17. Operations / Recovery / Rollback

## Reboot

Dùng khi có lý do guest-OS/service hợp lý:

```bash
sudo reboot
```

Nhưng production thường cần change/incident procedure tương ứng.

## Stop/start

Potential impact:

```text
Downtime
Public IPv4 change
Host change
RAM lost
Instance-store lost
```

Do đó stop/start là operation có blast radius lớn hơn reboot.

## Terminate

Destructive:

```text
Instance cannot be started again.
```

Trước terminate production phải xem:

```text
Data retention
EBS DeleteOnTermination
Evidence
Backup
Recovery path
Approval
```

### Rebuild strategy

Nếu Day 9 lab bị bootstrap sai nghiêm trọng, một cách sạch là:

```text
Bad user data
   ↓
Preserve evidence
   ↓
Correct user data
   ↓
Terminate disposable lab instance
   ↓
Launch fresh instance
   ↓
Validate again
```

Điều này thường tốt hơn:

```text
SSH in
→ manually fix many things
→ forget what was changed
→ configuration drift
```

---

# 18. Provisioning Evidence — Mandatory Lab Deliverable

Syllabus yêu cầu **submit provisioning evidence**. 00_MASTER_SYLLABUS_Java_Fresher…

Một evidence pack tốt có thể được tổ chức như bảng sau:

| Evidence                      | Cần chứng minh                        |
| ----------------------------- | ------------------------------------- |
| Instance details              | Instance ID, AMI, type, Region/AZ     |
| Tags                          | Resource được quản lý đúng            |
| VPC/Subnet                    | Instance nằm đúng network             |
| Security Group                | SSH chỉ cho approved source           |
| Metadata config               | IMDSv2 required                       |
| Status checks                 | EC2 health checks passed              |
| `cat /etc/os-release`         | OS thực tế                            |
| `uname -r`                    | Kernel thực tế                        |
| `hostnamectl`                 | Bootstrap hostname                    |
| `cloud-init status --long`    | cloud-init completion                 |
| `/var/log/day9-bootstrap.log` | Bootstrap execution                   |
| Bootstrap marker              | Script reached successful end         |
| `systemctl status httpd`      | Service active                        |
| `ss -lntp`                    | Ports/listeners                       |
| `curl 127.0.0.1`              | HTTP functional validation            |
| IMDSv2 query                  | Metadata functional                   |
| SSH login                     | Secure access succeeded               |
| Cleanup                       | Instance terminated when lab complete |

### Evidence không được chứa

```text
Private key
Access key
Secret key
Password
Session token
Sensitive user data
```

Evidence tốt phải trả lời:

```text
What was provisioned?
With what configuration?
Did it boot correctly?
Did bootstrap complete?
Was secure access validated?
Was functionality validated?
Was security configuration checked?
Was cleanup completed?
```

---

# 19. Supplementary — AWS CLI Version

Console lab là đủ cho mandatory Day 9, nhưng vì Day 7 có AWS CLI nên đây là phần rất hữu ích.

Check identity trước:

```bash
aws sts get-caller-identity
```

**Không tiếp tục nếu account/role không phải lab account/approved identity.**

Set context:

```bash
export AWS_REGION=ap-southeast-1
```

Ví dụ launch:

```bash
aws ec2 run-instances \
  --region "$AWS_REGION" \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type <INSTANCE_TYPE> \
  --key-name day9-lab-key \
  --subnet-id <SUBNET_ID> \
  --security-group-ids <SG_ID> \
  --associate-public-ip-address \
  --metadata-options "HttpEndpoint=enabled,HttpTokens=required" \
  --user-data file://day9-user-data.sh \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=day9-al2023-lab},{Key=Environment,Value=lab},{Key=Purpose,Value=java-fresher-day9}]'
```

AWS hỗ trợ dùng:

```text
resolve:ssm:/aws/service/ami-amazon-linux-latest/...
```

để resolve AMI hiện tại thay vì hard-code AMI ID. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/finding-an-ami-parameter-store.html?utm_source=chatgpt.com)

Describe:

```bash
aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --instance-ids <INSTANCE_ID>
```

Status:

```bash
aws ec2 describe-instance-status \
  --region "$AWS_REGION" \
  --instance-ids <INSTANCE_ID>
```

Đây là **Supplementary / Beyond the explicit Day 9 lab**, nhưng rất hữu ích cho automation và exam practical thinking.

---

# 20. Cleanup — Bắt buộc sau lab

## Trước cleanup

Thu evidence trước.

Sau đó verify:

```text
No data required
No assignment evidence still missing
This is disposable lab resource
Correct account
Correct Region
Correct instance ID
```

### Terminate instance

Console:

```text
EC2
→ Instances
→ select day9-al2023-lab
→ Instance state
→ Terminate instance
```

Hoặc CLI:

```bash
aws ec2 terminate-instances \
  --region "$AWS_REGION" \
  --instance-ids <INSTANCE_ID>
```

Sau đó verify state chuyển:

```text
shutting-down
→
terminated
```

### Kiểm tra leftovers

Đừng chỉ terminate EC2 rồi kết luận cleanup xong.

Check:

```text
EBS volumes
Elastic IPs
Public IPv4 resources
Unused SG if dedicated
Dedicated key-pair record
Other accidental resources
```

AWS nhấn mạnh stopped instance không còn compute usage charge nhưng EBS và các resource khác vẫn có thể phát sinh phí. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html?utm_source=chatgpt.com)

Nếu key pair chỉ tạo cho lab, sau khi chắc chắn không còn instance nào phụ thuộc nó, có thể remove EC2 key-pair resource và xử lý local private key theo policy của lab.

---

# 21. L1 Troubleshooting vs Escalation

| Situation                        | L1 action                                          |
| -------------------------------- | -------------------------------------------------- |
| Wrong SSH source IP              | Verify/change nếu thuộc approved scope             |
| Instance not running             | Inspect lifecycle/state                            |
| Wrong username                   | Correct connection command                         |
| Wrong key selected locally       | Use approved matching key                          |
| `httpd` stopped                  | Inspect log → start nếu safe/approved              |
| User data syntax error           | Collect evidence, correct/rebuild lab              |
| SG missing approved rule         | Fix within approved change boundary                |
| cloud-init failure               | Inspect logs/root cause                            |
| System status failure            | Preserve evidence; approved stop/start or escalate |
| Host-key mismatch unexplained    | **Do not bypass; escalate/verify**                 |
| IAM `AccessDenied` outside role  | Escalate to IAM owner; don't request admin         |
| NACL/VPC change outside scope    | Escalate network owner                             |
| Lost key with no recovery route  | Escalate/recovery procedure                        |
| Production destructive operation | Approval/change/incident process                   |
| Suspected credential exposure    | Security escalation                                |

Important operational principle:

```text
L1 does not mean:
"try anything until it works."
```

L1 nghĩa:

```text
Diagnose safely
+
operate within approved boundary
+
preserve evidence
+
escalate correctly
```

---

# 22. Exam Focus

Bạn cần nhớ chắc các distinctions sau.

### Question: AMI là gì?

Strong answer:

> AMI là Region-specific machine image dùng làm template để launch EC2 instances. Nó cung cấp OS/software configuration cần để boot cùng block-device mapping và phải tương thích với instance architecture/type.

### Question: Stop khác terminate?

```text
Stop:
instance vẫn tồn tại
có thể start lại
EBS giữ
compute usage stops
public IPv4 thường thay đổi after start

Terminate:
instance bị xóa
không start lại được
root EBS thường delete theo DeleteOnTermination
```

### Question: Reboot khác stop/start?

```text
Reboot:
same instance runtime lifecycle
public/private IP retained
same host normally

Stop/start:
host may change
public IPv4 usually changes
instance store lost
```

### Question: EC2 key pair hoạt động thế nào?

> EC2 giữ/đưa public key vào instance; operator giữ private key. SSH chứng minh possession của private key để authenticate với public key trong `authorized_keys`.

### Question: User data chạy khi nào?

> Linux EC2 user-data/cloud-init directives mặc định chạy trong initial launch boot cycle, không mặc định chạy mỗi reboot.

### Question: User data chạy bằng user nào?

> Shell user data trên Amazon Linux chạy dưới `root`.

### Question: Debug user data ở đâu?

```text
/var/log/cloud-init-output.log
```

và có thể dùng:

```bash
cloud-init status --long
```

### Question: IMDSv2 khác IMDSv1?

> IMDSv2 yêu cầu session token: trước hết `PUT /latest/api/token`, sau đó gửi token trong metadata requests. Có thể configure EC2 để require token và từ chối IMDSv1.

### Question: SSH timeout và publickey error khác nhau?

```text
Timeout:
thường điều tra network path/SG/route/IP/service reachability

Permission denied (publickey):
network/sshd thường đã reachable;
điều tra username/key/authorized_keys
```

---

# 23. Knowledge Check

|   # | Câu hỏi                                                                                      |
| --: | -------------------------------------------------------------------------------------------- |
|   1 | Tại sao `running` không có nghĩa bootstrap hoàn tất?                                         |
|   2 | Stop/start ảnh hưởng public IPv4 như thế nào?                                                |
|   3 | Vì sao không hard-code AMI ID lấy từ blog?                                                   |
|   4 | AMI và EC2 instance khác nhau thế nào?                                                       |
|   5 | Public key và private key được đặt ở đâu?                                                    |
|   6 | Tại sao `chmod 400` cho `.pem`?                                                              |
|   7 | Default username của Amazon Linux là gì?                                                     |
|   8 | User data mặc định chạy lại khi reboot không?                                                |
|   9 | User-data shell script chạy với privilege nào?                                               |
|  10 | Log chính để debug cloud-init user-data là gì?                                               |
|  11 | Vì sao không đặt secrets trong user data?                                                    |
|  12 | `169.254.169.254` có vai trò gì?                                                             |
|  13 | IMDSv2 token được lấy bằng GET hay PUT?                                                      |
|  14 | `HttpTokens=required` có ý nghĩa gì?                                                         |
|  15 | SSH timeout và `Permission denied (publickey)` chỉ ra những failing layer khác nhau thế nào? |
|  16 | Vì sao không nên mở SSH `0.0.0.0/0`?                                                         |
|  17 | Nếu system status check fail, phạm vi nghi ngờ khác instance status check thế nào?           |
|  18 | Tại sao terminate phải được xem là destructive operation?                                    |
|  19 | Tại sao evidence phải được thu trước cleanup?                                                |
|  20 | Vì sao HTTP localhost success chưa chứng minh public HTTP connectivity?                      |

Nếu bạn trả lời được 20 câu này bằng lý do chứ không chỉ thuộc lòng, phần theory của Day 9 đã khá chắc.

---

# 24. Independent Practical Challenge

Sau guided lab, hãy tự làm lại **không nhìn từng bước** với yêu cầu:

```text
Amazon Linux 2023
IMDSv2 required
Dedicated key pair
SSH only from your IP
Public subnet
User data bootstrap
Install/start one service
Create bootstrap marker
Connect via SSH
Validate OS
Validate service
Validate listener
Validate application
Query metadata with IMDSv2
Collect evidence
Terminate lab safely
```

Failure injection bắt buộc:

```text
Temporarily remove SSH inbound rule
→ attempt SSH
→ classify symptom
→ prove root cause
→ restore approved /32 rule
→ validate SSH
→ preserve before/after evidence
```

Điều cần học không phải:

> “Xóa rule nên SSH hỏng.”

Mà là reasoning:

```text
Symptom:
SSH timeout

Expected:
TCP/22 reachable

Recent change:
SG SSH rule removed

Scope:
network connectivity, not authentication

Evidence:
instance healthy
correct public IP
route exists
SG lacks TCP/22 source rule

Hypothesis:
SG is dropping inbound TCP/22

Safe remediation:
restore approved source-IP/32 rule

Validation:
SSH succeeds

Root cause:
missing security-group rule
```

Đó chính là troubleshooting standard bạn sẽ cần ở Day 20.

---

# 25. Day 9 Mastery Check

Bạn **chưa nên tự đánh dấu Day 9 là mastered chỉ vì đã đọc bài này**.

Mức độ có thể đánh giá như sau:

| Level        | Evidence                                             |
| ------------ | ---------------------------------------------------- |
| Understand   | Giải thích lifecycle, AMI, key, user data, IMDS, SSH |
| Configure    | Launch EC2 đúng network/security                     |
| Automate     | Bootstrap bằng user data                             |
| Validate     | Status, logs, listener, endpoint, metadata           |
| Troubleshoot | Phân biệt network/auth/bootstrap failure             |
| Operate      | Reboot/stop/start/terminate hiểu impact              |
| Secure       | `/32`, protected key, IMDSv2, no secrets             |
| Recover      | Rebuild disposable failed instance                   |
| Document     | Provisioning evidence đầy đủ                         |
| Mastery      | Thực hiện toàn bộ mà không cần step-by-step          |

---

# 26. Tóm tắt Day 9

Mental model cuối cùng bạn nên ghi nhớ:

```text
                 AMI
                  │
                  ▼
            Launch EC2
                  │
                  ▼
               pending
                  │
                  ▼
               running
                  │
        ┌─────────┴─────────┐
        │                   │
        ▼                   ▼
    cloud-init          Status checks
        │
        ▼
    User Data
        │
        ├── hostname
        ├── packages
        ├── config
        └── services
        │
        ▼
   Linux bootstrapped
        │
        ├──── SSH ← Key Pair
        │           │
        │           └── SG TCP/22 from /32
        │
        ├──── IMDSv2
        │       │
        │       └── token required
        │
        └──── Validation
                │
                ├── cloud-init
                ├── systemctl
                ├── ss
                ├── curl
                └── logs
                       │
                       ▼
                  Evidence Pack
                       │
                       ▼
                    Cleanup
```

Day 9 bắt buộc từ syllabus vì thế đã được bao phủ đầy đủ ở cả **Concept/Lecture: EC2 lifecycle, AMI, key pair, user data, instance metadata awareness, SSH access, Linux bootstrap** và **Assignment/Lab: launch + bootstrap EC2 Linux + secure connection + provisioning evidence**. 00_MASTER_SYLLABUS_Java_Fresher…

Điểm quan trọng nhất về mindset: **đừng dừng ở “EC2 đã chạy”**. Một Administrator phải chứng minh từng lớp:

```text
Provisioned
→ booted
→ healthy
→ bootstrapped
→ securely reachable
→ service running
→ port listening
→ endpoint responding
→ metadata secured
→ evidence collected
→ resource safely cleaned up
```

Đó mới là mức Day 9 hướng tới.
