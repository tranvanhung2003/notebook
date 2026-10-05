Được. Ta tiếp tục **Phần 2 → Unit 3 – AWS Cloud Fundamentals (25 phút)** trên chính environment đã dựng. Theo guide, Unit 3 phải thể hiện IAM/MFA/CLI profile/tagging; VPC + connectivity allow/block; EC2 + user data + SSH; EBS snapshot/restore; CloudWatch/Budget/cost control. :chatgpt-content-reference{index="0"} Nội dung này tương ứng Day 7–11 trong syllabus, bao gồm cả CloudTrail awareness và kiểm soát chi phí. :chatgpt-content-reference{index="1"}

Tôi giữ đúng convention bạn đã đặt: **AWS resource operations dùng AWS Console; command Linux chỉ chạy sau khi SSH vào EC2; không dùng AWS CLI.**

Có một khác biệt cần nói rõ: guide có mục **CLI profile**, nhưng vì bạn đã chọn Console-only cho AWS operations, ta sẽ **giải thích CLI profile bằng lời nhưng không tạo access key chỉ để demo**. Điều này cũng phù hợp security practice hiện tại: AWS khuyến nghị human users ưu tiên federation/temporary credentials thay vì IAM user với long-term credentials. IAM user dưới đây chỉ được tạo để thực hành đúng syllabus, với quyền hạn chế và MFA. :chatgpt-content-reference{index="2"}

# 1. Kịch bản Unit 3

Tôi đề xuất flow khoảng:

| Khoảng     | Nội dung                                                                      |
| ---------- | ----------------------------------------------------------------------------- |
| 0–5 phút   | IAM user + least privilege + MFA + tagging                                    |
| 5–10 phút  | VPC architecture + preview create VPC/subnet/RT/IGW/SG + SG failure injection |
| 10–13 phút | EC2 lifecycle + user data + SSH/bootstrap validation                          |
| 13–19 phút | EBS attach/mount → snapshot → restore                                         |
| 19–24 phút | CloudWatch Alarm + AWS Budget                                                 |
| 24–25 phút | Cost control + CloudTrail/checklist                                           |

Không cần setup lại environment.

---

# 2. IAM — tạo một user quyền vừa phải

## Mục tiêu

Ta tạo:

```text
IAM User:
java-fresher-demo-ops

IAM Group:
java-fresher-l1-operators

Customer managed policy:
JavaFresherL1OperatorPolicy
```

User này:

```text
ĐƯỢC:
- xem EC2
- xem VPC/subnet/route table/SG/EBS
- xem CloudWatch
- Start/Stop EC2 thuộc chính lab của mình

KHÔNG ĐƯỢC:
- terminate EC2
- tạo/xóa IAM user
- sửa Security Group
- tạo/xóa VPC
- xóa volume/snapshot
- AdministratorAccess
```

Đây là một ví dụ rõ của **least privilege + access boundary**.

AWS có tài liệu chính thức về policy cho phép Start/Stop EC2 dựa trên resource tag; ta sẽ dùng đúng pattern đó. :chatgpt-content-reference{index="3"}

---

# 3. Tag EC2 trước

Trước khi tạo policy, mở:

```text
AWS Console
→ EC2
→ Instances
→ java-fresher-integrated-lab
→ Tags
→ Manage tags
```

Đảm bảo có:

```text
Name        = java-fresher-integrated-lab
Project     = JavaFresherLab
Environment = Lab
Owner       = java-fresher-demo-ops
```

`Owner` quan trọng vì policy phía dưới sẽ dựa vào nó.

Tagging không chỉ để "trang trí"; AWS IAM có thể dùng resource tags làm điều kiện authorization. :chatgpt-content-reference{index="4"}

---

# 4. Tạo customer-managed IAM policy

Vào:

```text
AWS Console
→ IAM
→ Policies
→ Create policy
→ JSON
```

Dùng:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadLabInfrastructure",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus",
        "ec2:DescribeInstanceAttribute",
        "ec2:DescribeVolumes",
        "ec2:DescribeSnapshots",
        "ec2:DescribeVpcs",
        "ec2:DescribeSubnets",
        "ec2:DescribeRouteTables",
        "ec2:DescribeInternetGateways",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSecurityGroupRules",
        "ec2:DescribeNetworkAcls",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeAddresses",
        "ec2:DescribeTags",
        "cloudwatch:DescribeAlarms",
        "cloudwatch:DescribeAlarmsForMetric",
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "tag:GetResources",
        "tag:GetTagKeys",
        "tag:GetTagValues"
      ],
      "Resource": "*"
    },
    {
      "Sid": "OperateOwnLabInstance",
      "Effect": "Allow",
      "Action": ["ec2:StartInstances", "ec2:StopInstances"],
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "aws:ResourceTag/Owner": "${aws:username}"
        }
      }
    }
  ]
}
```

AWS Console hiện hỗ trợ tạo customer-managed policy bằng JSON và thực hiện policy validation trước khi tạo. :chatgpt-content-reference{index="5"}

---

# 5. Giải thích policy

Phần đầu:

```json
"Effect": "Allow"
```

cho phép các actions được liệt kê.

Ví dụ:

```json
"ec2:DescribeInstances"
```

cho phép xem EC2.

```json
"ec2:DescribeVolumes"
```

xem EBS.

```json
"ec2:DescribeVpcs"
```

xem VPC.

```json
"ec2:DescribeSecurityGroups"
```

xem Security Groups.

Những `Describe*` ở đây chỉ cho phép **inspect**, không cho phép sửa.

---

Phần:

```json
"Resource": "*"
```

với các Describe actions là bình thường vì nhiều EC2 Describe API không hỗ trợ giới hạn resource-level như các action thay đổi resource. AWS cũng sử dụng pattern này trong các example policies. :chatgpt-content-reference{index="6"}

---

CloudWatch:

```json
"cloudwatch:DescribeAlarms"
"cloudwatch:ListMetrics"
"cloudwatch:GetMetricData"
"cloudwatch:GetMetricStatistics"
```

cho user xem metric và alarm.

Nhưng không có:

```text
cloudwatch:PutMetricAlarm
cloudwatch:DeleteAlarms
```

nên user không được sửa alarm.

---

Phần đặc biệt:

```json
"ec2:StartInstances",
"ec2:StopInstances"
```

cho phép start/stop.

Nhưng có condition:

```json
"aws:ResourceTag/Owner": "${aws:username}"
```

Nếu login bằng:

```text
java-fresher-demo-ops
```

thì EC2 phải có:

```text
Owner=java-fresher-demo-ops
```

user mới được Start/Stop.

Đây là:

```text
identity
+
resource tag
+
least privilege
```

Không có:

```text
ec2:TerminateInstances
```

nên không được terminate.

Không có:

```text
ec2:AuthorizeSecurityGroupIngress
ec2:RevokeSecurityGroupIngress
```

nên cũng không được sửa firewall.

Đó là chủ ý.

---

# 6. Tạo policy

Sau khi paste JSON:

```text
Next
```

Name:

```text
JavaFresherL1OperatorPolicy
```

Description:

```text
Limited read and tagged EC2 start/stop access for Java Fresher L1 lab
```

Review warnings.

Nếu policy validator báo error, **không bấm Create bỏ qua**.

Sau khi hợp lệ:

```text
Create policy
```

---

# 7. Tạo IAM Group

Đi:

```text
IAM
→ User groups
→ Create group
```

Name:

```text
java-fresher-l1-operators
```

Attach:

```text
JavaFresherL1OperatorPolicy
```

Create.

Tại sao không attach policy trực tiếp vào user?

Bạn có thể giải thích:

> "Em quản lý permission thông qua group để nhiều user cùng job function có thể nhận cùng baseline permission. User membership thay đổi dễ hơn việc quản lý nhiều inline policy riêng lẻ."

---

# 8. Tạo IAM User

Đi:

```text
IAM
→ Users
→ Create user
```

User name:

```text
java-fresher-demo-ops
```

Nếu Console hiện:

```text
Provide user access to the AWS Management Console
```

bật lên vì ta muốn demo interactive login.

Bạn có thể để AWS tạo password ban đầu.

**Không hiển thị password trong recording và không lưu password vào `history.log`.**

Nếu có option:

```text
Users must create a new password at next sign-in
```

bật.

AWS hiện vẫn hỗ trợ IAM user console access và khuyến nghị MFA cho user đó. :chatgpt-content-reference{index="7"}

Add user vào:

```text
java-fresher-l1-operators
```

Tags có thể thêm:

```text
Department = Operations
Environment = Lab
```

AWS IAM cũng hỗ trợ tags cho IAM users. :chatgpt-content-reference{index="8"}

Create user.

---

# 9. MFA — bắt buộc nên demo

Vào:

```text
IAM
→ Users
→ java-fresher-demo-ops
→ Security credentials
```

Tìm:

```text
Multi-factor authentication (MFA)
```

Chọn:

```text
Assign MFA device
```

Device:

```text
Authenticator app
```

Scan QR với authenticator của bạn.

AWS sẽ yêu cầu các code MFA để xác minh.

Sau khi hoàn tất:

```text
MFA device = Assigned
```

Bạn nên nói:

> "Password là factor thứ nhất. MFA code là factor thứ hai. Nếu password bị lộ thì attacker vẫn thiếu factor còn lại."

Không quay QR seed lâu trên recording.

---

# 10. Test permission của IAM user

Nên dùng private/incognito browser để không logout admin session.

Login bằng:

```text
java-fresher-demo-ops
```

- password + MFA.

Vào:

```text
EC2 → Instances
```

User phải xem được lab EC2.

Có thể kiểm tra:

```text
Instance state
→ Stop instance
```

button/action có thể khả dụng đối với instance:

```text
Owner=java-fresher-demo-ops
```

Nhưng **không stop ngay**, vì Unit 4–6 còn cần EC2.

Quan trọng hơn, user không có permission sửa SG hay terminate EC2.

Bạn nên giải thích:

> "Đây là access boundary. Em không cấp quyền chỉ vì user có thể cần nó trong tương lai. Khi cần thay đổi Security Group, phải dùng approved elevated access."

---

# 11. CLI profile trong guide — xử lý thế nào?

Guide có:

> IAM access, MFA, CLI profile, tagging. :chatgpt-content-reference{index="9"}

Nhưng theo convention của bạn, chúng ta **không dùng AWS CLI**.

Trong recording nói ngắn:

> "AWS CLI profile là named credential/configuration profile dùng để phân tách AWS identities hoặc environments khi thao tác qua CLI. Trong demo này em sử dụng AWS Console theo scope đã định nên em không tạo long-lived access key chỉ để demo CLI. IAM access hiện được bảo vệ bằng console password và MFA."

Điều này tốt hơn việc tạo access key rồi không cần dùng.

---

# 12. VPC — không tạo thật, chỉ đi qua form

Environment thật của ta đã có:

```text
VPC
10.10.0.0/16

Public subnet
10.10.10.0/24

Private subnet
10.10.20.0/24

Public route:
0.0.0.0/0 → Internet Gateway
```

Ta không duplicate.

---

# 13. Preview tạo VPC

Vào:

```text
VPC Console
→ Your VPCs
→ Create VPC
```

Chọn:

```text
VPC only
```

Điền thử:

```text
Name:
demo-preview-vpc

IPv4 CIDR:
10.20.0.0/16

IPv6:
No IPv6 CIDR block

Tenancy:
Default
```

Tới đây dừng.

Bạn nói:

> "`/16` là address space của VPC. Subnet sau đó phải lấy CIDR nằm bên trong address space này."

**Không nhấn Create VPC.**

Cancel.

---

# 14. Preview Public Subnet

Vào:

```text
VPC
→ Subnets
→ Create subnet
```

Giả định VPC preview tồn tại, bạn giải thích các field:

```text
VPC:
demo-preview-vpc

Subnet name:
demo-public-a

Availability Zone:
ap-southeast-1a

IPv4 CIDR:
10.20.10.0/24
```

Không Create.

Bạn nói:

> "Subnet không tự trở thành public chỉ vì tên là public. Để public IPv4 traffic ra Internet, subnet cần route table có default route tới Internet Gateway và instance cần public IPv4."

Cancel.

---

# 15. Preview Private Subnet

Cũng form đó:

```text
Name:
demo-private-a

CIDR:
10.20.20.0/24
```

Private subnet của design này sẽ **không có**:

```text
0.0.0.0/0 → Internet Gateway
```

Không Create.

---

# 16. Preview Internet Gateway

Đi:

```text
VPC
→ Internet gateways
→ Create internet gateway
```

Name:

```text
demo-preview-igw
```

Giải thích:

> "Create IGW chưa đủ. Sau khi tạo còn phải attach IGW vào VPC."

Không Create.

---

# 17. Preview Route Table

Đi:

```text
VPC
→ Route tables
→ Create route table
```

Name:

```text
demo-preview-public-rt
```

Chọn VPC preview về mặt conceptual.

Giải thích sau khi tạo phải có:

```text
10.20.0.0/16 → local
0.0.0.0/0    → IGW
```

và associate với public subnet.

Không Create.

---

# 18. Preview Security Group

Đi:

```text
VPC
→ Security groups
→ Create security group
```

Name:

```text
demo-preview-sg
```

Ví dụ inbound:

```text
22    SSH         MY_PUBLIC_IP/32
8080  Tomcat      MY_PUBLIC_IP/32
3000  Grafana     MY_PUBLIC_IP/32
9090  Prometheus  MY_PUBLIC_IP/32
```

Không có:

```text
5432 from Internet
9100 from Internet
```

Giải thích:

> "Security Group chỉ có allow rules, không có explicit deny rule. Nếu inbound không được allow thì traffic đó không được phép đi vào." :chatgpt-content-reference{index="10"}

Không Create.

---

# 19. Security Group failure injection thật

Đây là phần quan trọng nhất của VPC demo.

**Cảnh báo: không đụng rule SSH TCP/22.**

Nếu xóa nhầm rule SSH `/32`, bạn có thể mất remote access.

Ta chỉ test:

```text
TCP/8080 Tomcat
```

---

# 20. BEFORE — chứng minh Tomcat đang healthy

Bạn đang SSH trên EC2.

Chạy:

```bash
systemctl is-active tomcat10
```

Expected:

```text
active
```

Socket:

```bash
sudo ss -lntp | grep ':8080'
```

Endpoint local:

```bash
curl -fsS http://127.0.0.1:8080/ >/dev/null
echo "tomcat_local_exit=$?"
```

Expected:

```text
tomcat_local_exit=0
```

Sau đó trên Windows browser:

```text
http://EC2_PUBLIC_IP:8080/
```

phải truy cập được.

Ta vừa có BEFORE state:

```text
service healthy
port listening
local HTTP works
external HTTP works
```

---

# 21. ACTION — block TCP/8080 bằng Security Group

AWS Console:

```text
EC2
→ Security Groups
→ java-fresher-ec2-sg
→ Inbound rules
→ Edit inbound rules
```

Trước khi delete, ghi nhớ chính xác rule:

```text
Type: Custom TCP
Protocol: TCP
Port: 8080
Source: YOUR_PUBLIC_IP/32
Description: Tomcat from workstation
```

Chỉ delete rule:

```text
8080
```

Không delete:

```text
22
```

Save rules.

AWS cho phép add/remove/edit Security Group rules sau khi EC2 đã launch. :chatgpt-content-reference{index="11"}

---

# 22. AFTER — chứng minh external connectivity bị block

Refresh Windows browser:

```text
http://EC2_PUBLIC_IP:8080/
```

Expected:

```text
connection fails / times out
```

Bây giờ **không restart Tomcat**.

Trên EC2:

```bash
systemctl is-active tomcat10
```

vẫn:

```text
active
```

```bash
sudo ss -lntp | grep ':8080'
```

vẫn listen.

```bash
curl -fsS http://127.0.0.1:8080/ >/dev/null
echo $?
```

vẫn:

```text
0
```

Đây là evidence cực đẹp:

```text
Layer 7 application:
OK

Tomcat process:
OK

Local TCP/HTTP:
OK

External TCP connection:
FAIL
```

Vừa thay đổi duy nhất:

```text
Security Group inbound rule
```

Root cause:

```text
AWS Security Group
→ Layer 4 access control
```

Bạn nên nói:

> "Em không restart application vì evidence cho thấy application vẫn healthy. Recent change là Security Group, local connectivity vẫn OK nhưng external connectivity fail, nên root cause nằm ở AWS network access control."

Đây chính xác là OSI Mindset mà guide yêu cầu. :chatgpt-content-reference{index="12"}

---

# 23. ROLLBACK — restore rule

AWS Console:

```text
Security Group
→ Inbound rules
→ Edit inbound rules
→ Add rule
```

Restore **chính xác**:

```text
Custom TCP
TCP
8080
YOUR_PUBLIC_IP/32
```

Không dùng:

```text
0.0.0.0/0
```

Save.

Browser:

```text
http://EC2_PUBLIC_IP:8080/
```

phải hoạt động trở lại.

Flow demo của bạn lúc này rất đẹp:

```text
BEFORE
external works
        ↓
ACTION
remove SG/8080
        ↓
AFTER
external fails
local works
        ↓
ROOT CAUSE
Security Group
        ↓
ROLLBACK
restore /32 rule
        ↓
RESULT
external works again
```

---

# 24. CloudTrail awareness sau SG change

Đây là phần syllabus Day 11 dù practical guide không tách thành một mục riêng.

AWS Console:

```text
CloudTrail
→ Event history
```

Bạn có thể tìm các event liên quan:

```text
RevokeSecurityGroupIngress
AuthorizeSecurityGroupIngress
```

Ý nghĩa:

```text
ai thay đổi
khi nào
API/action nào
resource liên quan
```

Điểm cần nói:

> "Khi troubleshoot production, em không chỉ nhìn trạng thái hiện tại mà còn kiểm tra recent changes. CloudTrail giúp truy vết AWS API activity."

---

# 25. EC2 lifecycle + User Data

AWS Console:

```text
EC2
→ Instances
→ java-fresher-integrated-lab
```

Chỉ cho người xem:

```text
Instance ID
Instance state
Instance type
AMI
VPC ID
Subnet ID
Availability Zone
Private IPv4
Public IPv4
Key pair
Security Group
```

Giải thích lifecycle:

```text
launch
→ pending
→ running
→ stop
→ stopped
→ start
→ running
→ terminate
→ terminated
```

`terminate` khác `stop`.

`Stop` có thể start lại.

`Terminate` là destruction của instance.

---

# 26. User Data đã làm gì?

User data của lab chúng ta đã bootstrap:

```text
Ubuntu
→ packages
→ Java 17
→ Tomcat
→ PostgreSQL
→ Prometheus
→ node_exporter
→ Grafana
→ service enable/start
→ health validation
```

AWS EC2 hỗ trợ Linux user data dưới dạng shell scripts hoặc cloud-init directives để tự động bootstrap instance sau launch. :chatgpt-content-reference{index="13"}

Trên EC2:

```bash
cloud-init status --long
```

Expected invariant:

```text
status: done
```

Bootstrap log riêng của chúng ta:

```bash
sudo less /var/log/java-fresher-bootstrap.log
```

Cloud-init output:

```bash
sudo less /var/log/cloud-init-output.log
```

Bạn không cần đọc toàn bộ.

Dùng:

```bash
sudo grep -Ei \
'bootstrap started|bootstrap completed|error|failed' \
/var/log/java-fresher-bootstrap.log
```

---

# 27. Validate kết quả bootstrap

```bash
java -version
```

```bash
for svc in \
  tomcat10 \
  postgresql \
  prometheus \
  prometheus-node-exporter \
  grafana-server
do
    printf '%-30s ' "$svc"
    systemctl is-active "$svc"
done
```

Expected:

```text
active
active
active
active
active
```

Bạn có thể nói:

> "EC2 Running không chứng minh bootstrap thành công. Em validate cloud-init, package/runtime và systemd services."

---

# 28. SSH security

Console → Security Group:

```text
TCP/22
Source:
MY_PUBLIC_IP/32
```

Không:

```text
0.0.0.0/0
```

Key pair authentication được dùng thay cho public password login.

Không bao giờ hiển thị:

```text
private key
```

trên recording.

---

# 29. EBS lab — cảnh báo trước

Phần này có lệnh:

```text
mkfs
```

**có tính destructive** nếu chạy nhầm device.

Trước khi format:

```text
xác định volume ID
→ xác định size
→ xác định device mới
→ kiểm tra FSTYPE
→ đảm bảo không phải root disk
→ rồi mới mkfs
```

Không đoán:

```text
/dev/nvme1n1
```

chỉ vì lần trước nó có tên đó.

Nitro EC2 có thể expose EBS volume dưới `/dev/nvmeXn1`, và tên NVMe có thể khác với `/dev/sdf` bạn nhập khi attach. AWS khuyến nghị dùng `lsblk`/volume ID để nhận diện. :chatgpt-content-reference{index="14"}

---

# 30. Tạo EBS volume bằng Console

Trước hết EC2 Console → instance:

ghi lại:

```text
Availability Zone
```

ví dụ:

```text
ap-southeast-1a
```

Volume phải cùng AZ với EC2 mới attach được. :chatgpt-content-reference{index="15"}

Đi:

```text
EC2
→ Elastic Block Store
→ Volumes
→ Create volume
```

Điền:

```text
Volume type:
gp3

Size:
2 GiB

Availability Zone:
SAME AZ AS EC2

Encryption:
Enabled
```

Tag:

```text
Name        = java-fresher-ebs-lab
Project     = JavaFresherLab
Environment = Lab
```

Create volume.

AWS Console hiện dùng `gp3` làm default cho volume tạo bằng console. :chatgpt-content-reference{index="16"}

---

# 31. Attach EBS

Khi volume:

```text
State = Available
```

chọn:

```text
Actions
→ Attach volume
```

Instance:

```text
java-fresher-integrated-lab
```

Device name có thể nhập:

```text
/dev/sdf
```

Attach.

---

# 32. Nhận diện disk trên EC2

SSH EC2:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,SERIAL
```

Có thể thấy:

```text
nvme0n1    30G ... /
nvme1n1     2G
```

Root:

```text
nvme0n1
```

New EBS:

```text
nvme1n1
```

**Tên thực tế của bạn có thể khác.**

Xác nhận disk mới không có filesystem:

```bash
sudo blkid /dev/nvme1n1
```

Nếu không output hoặc FSTYPE trống → blank volume.

Có thể kiểm tra:

```bash
sudo file -s /dev/nvme1n1
```

---

# 33. Format volume

Chỉ sau khi chắc chắn đúng device:

```bash
sudo mkfs.ext4 \
  -L java-fresher-data \
  /dev/nvme1n1
```

`mkfs.ext4` tạo ext4 filesystem.

`-L` đặt filesystem label.

Verify:

```bash
sudo blkid /dev/nvme1n1
```

---

# 34. Mount

```bash
sudo mkdir -p /mnt/labdata
```

```bash
sudo mount \
  /dev/nvme1n1 \
  /mnt/labdata
```

Verify:

```bash
findmnt /mnt/labdata
```

và:

```bash
df -h /mnt/labdata
```

---

# 35. Tạo recovery evidence

```bash
sudo tee /mnt/labdata/recovery-proof.txt >/dev/null <<EOF
Java Fresher EBS recovery test
Created: $(date -Is)
Host: $(hostname)
EOF
```

Đọc:

```bash
cat /mnt/labdata/recovery-proof.txt
```

Có thể tạo thêm checksum:

```bash
sha256sum /mnt/labdata/recovery-proof.txt
```

Lưu output để so sánh sau restore.

Sau đó:

```bash
sync
```

---

# 36. Snapshot

AWS Console:

```text
EC2
→ Volumes
→ java-fresher-ebs-lab
→ Actions
→ Create snapshot
```

Description:

```text
Java Fresher Unit 3 recovery snapshot
```

Tags:

```text
Name        = java-fresher-ebs-snapshot
Project     = JavaFresherLab
Environment = Lab
```

Create snapshot.

AWS hỗ trợ tạo EBS snapshot trực tiếp từ volume trong EC2 console. :chatgpt-content-reference{index="17"}

---

# 37. Restore snapshot

Khi bạn đã có một snapshot ở trạng thái usable/completed:

```text
EC2
→ Snapshots
→ java-fresher-ebs-snapshot
→ Actions
→ Create volume from snapshot
```

Chọn:

```text
Volume type:
gp3

Availability Zone:
SAME AZ AS EC2
```

Tag:

```text
Name=java-fresher-ebs-restore
```

Create.

Volume tạo từ snapshot là replica của data trong source snapshot. :chatgpt-content-reference{index="18"}

Attach volume restore:

```text
Actions
→ Attach volume
→ java-fresher-integrated-lab
```

Device:

```text
/dev/sdg
```

---

# 38. Nhận diện restore volume

Trên EC2:

```bash
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINTS,SERIAL
```

Ví dụ:

```text
nvme0n1   root
nvme1n1   original EBS mounted /mnt/labdata
nvme2n1   restored EBS
```

**Không chạy `mkfs` trên restored volume.**

Nếu bạn format nó, bạn vừa phá data mình đang cố chứng minh đã restore.

---

# 39. Mount restored volume

```bash
sudo mkdir -p /mnt/restore
```

Dùng **device path đã xác định thực tế**:

```bash
sudo mount \
  /dev/nvme2n1 \
  /mnt/restore
```

Verify:

```bash
findmnt /mnt/restore
```

Đọc file:

```bash
cat /mnt/restore/recovery-proof.txt
```

Checksum:

```bash
sha256sum /mnt/restore/recovery-proof.txt
```

So với checksum source.

Nếu giống nhau:

```text
snapshot exists
+
volume restored
+
filesystem mounts
+
actual file exists
+
contents match
```

Đó mới là **restore verification**.

Chỉ nói:

> "Snapshot completed"

chưa chứng minh restore thành công.

---

# 40. Cleanup restore volume

Sau verify:

```bash
sudo umount /mnt/restore
```

Verify:

```bash
findmnt /mnt/restore
```

không còn mount.

Sau đó Console:

```text
EC2
→ Volumes
→ java-fresher-ebs-restore
→ Actions
→ Detach volume
```

Khi:

```text
Available
```

và chắc chắn đó là **restored test volume**, không phải source:

```text
Delete volume
```

Cẩn thận kiểm tra:

```text
Volume ID
Name tag
Attachment
```

trước khi delete.

---

# 41. CloudWatch alarm

Ta tạo alarm thực tế.

AWS Console:

```text
CloudWatch
→ Alarms
→ All alarms
→ Create alarm
```

Choose metric:

```text
EC2
→ Per-Instance Metrics
```

Chọn:

```text
InstanceId = EC2 lab
MetricName = CPUUtilization
```

AWS chính thức hướng dẫn tạo CPU alarm theo flow này. :chatgpt-content-reference{index="19"}

Condition tôi đề xuất cho lab:

```text
Statistic:
Average

Period:
5 minutes

Threshold:
CPUUtilization >= 70%

Datapoints:
1 out of 1
```

Tên:

```text
java-fresher-high-cpu
```

---

# 42. Notification

Nếu muốn demo đầy đủ:

```text
Notification
→ Create new SNS topic
```

Name:

```text
java-fresher-alerts
```

Email:

```text
email của chính bạn
```

Bạn không cần gửi email đó cho tôi.

Sau khi tạo subscription, email đó phải được confirm thì SNS mới gửi notification tới nó.

Nếu không muốn lộ email trong recording, có thể tạo alarm không có notification action và giải thích notification separately; nhưng để exam mạnh hơn thì notification thật tốt hơn.

---

# 43. Cách giải thích CloudWatch Alarm

Không nói đơn giản:

> "CloudWatch monitor CPU."

Nên nói:

> "CloudWatch thu thập metric. Alarm đánh giá metric theo statistic, period và threshold. Ở đây Average CPUUtilization trong khoảng 5 phút được so với threshold 70%. Khi condition đủ evaluation periods thì alarm chuyển state, ví dụ từ OK sang ALARM, và có thể trigger SNS notification hoặc action khác."

CloudWatch hỗ trợ SNS notification khi alarm state thay đổi. :chatgpt-content-reference{index="20"}

---

# 44. Budget control

AWS Console:

```text
Billing and Cost Management
→ Budgets
→ Create budget
```

Chọn:

```text
Customize (advanced)
→ Cost budget
```

AWS hiện hướng dẫn Cost Budget theo flow này. :chatgpt-content-reference{index="21"}

Ví dụ lab:

```text
Budget name:
java-fresher-monthly-budget

Period:
Monthly

Budget amount:
$5
```

Threshold:

```text
80% of budget
Actual
```

Email:

```text
email của bạn
```

Điều này nghĩa là:

```text
Budget = $5

80% =
$4 actual spending
```

thì alert condition được thỏa.

---

# 45. Actual vs Forecasted budget alert

Bạn cần giải thích:

```text
Actual
```

dựa trên chi phí đã phát sinh.

```text
Forecasted
```

dựa trên AWS forecast mức chi tiêu cuối period.

Một chi tiết hiện hành đáng nhớ: AWS nói forecast-based budget alerts cần đủ historical usage để tạo forecast; tài liệu hiện ghi AWS cần khoảng **5 tuần usage data** để tạo budget forecasts. Vì account lab mới có thể chưa đủ history, alarm `Actual 80%` là lựa chọn chắc chắn hơn để demo. :chatgpt-content-reference{index="22"}

---

# 46. Cost control

Ở Unit 3 đừng stop EC2 giữa buổi.

Unit 4–6 còn cần:

```text
Tomcat
PostgreSQL
Prometheus
Grafana
```

Nếu bạn stop EC2 bây giờ thì tự phá phần demo còn lại.

Trong Unit 3 nói:

> "Em đã kiểm tra đường dẫn Instance state → Stop instance, nhưng em chưa stop vì EC2 hiện đang là dependency cho Unit 4–6. Sau khi hoàn thành toàn bộ demo em sẽ stop instance."

Đây là decision đúng về dependency/change control.

Sau khi **toàn bộ buổi demo** kết thúc:

```text
EC2
→ Instances
→ java-fresher-integrated-lab
→ Instance state
→ Stop instance
```

---

# 47. Vì sao phải quan tâm Public IPv4?

AWS hiện tính:

```text
$0.005 / public IPv4 / hour
```

cho cả public IPv4 in-use và idle public IPv4 thuộc account/resource của bạn. :chatgpt-content-reference{index="23"}

Nên:

```text
EC2 compute
+
EBS storage
+
snapshots
+
public IPv4
```

đều phải được xem xét trong cost-control.

Không chỉ EC2 compute.

---

# 48. Cleanup những resource tạm của Unit 3

Sau rehearsal hoặc sau khi không cần nữa, kiểm tra:

```text
IAM:
java-fresher-demo-ops
java-fresher-l1-operators
JavaFresherL1OperatorPolicy

EBS:
java-fresher-ebs-lab
java-fresher-ebs-restore

Snapshots:
java-fresher-ebs-snapshot

CloudWatch:
java-fresher-high-cpu

Budget:
java-fresher-monthly-budget
```

Đừng delete blindly.

Nếu còn cần cho final demo thì giữ.

Sau final exam/rehearsal cuối mới cleanup.

---

# 49. Kịch bản nói khi demo Security Group

Đây là đoạn bạn nên tập thật chắc:

> "Trước khi thay đổi em xác nhận Tomcat đang active, port 8080 đang listen và local curl thành công. Từ browser bên ngoài em cũng truy cập được. Sau đó em remove duy nhất inbound rule TCP/8080 từ IP workstation của em. External connection bị chặn nhưng trên EC2 Tomcat vẫn active, socket vẫn listen và localhost curl vẫn thành công. Vì recent change chỉ là Security Group và Application layer vẫn healthy, em xác định lỗi nằm ở Transport/access-control layer của AWS. Em không restart Tomcat. Sau đó em restore chính xác rule `/32`, validate browser lại và xác nhận service phục hồi."

Nếu bạn trình bày được flow này, bạn đang thể hiện cùng lúc:

```text
VPC understanding
Security Group
OSI Mindset
evidence
recent-change analysis
safe remediation
rollback
validation
```

Đây là một trong những phần mạnh nhất của Unit 3.

---

# 50. Kịch bản nói về IAM

Bạn nên diễn đạt kiểu:

> "Em không cấp AdministratorAccess. Em tạo IAM user riêng cho L1 operator, attach permission qua group. User được xem EC2/VPC/EBS/CloudWatch và chỉ được Start/Stop instance có `Owner` tag khớp username. User không được terminate instance hoặc thay đổi Security Group. Đây là least privilege và access boundary. Em cũng bật MFA để giảm rủi ro nếu password bị lộ. Trong môi trường enterprise hiện đại, human access nên ưu tiên federation/temporary credentials; IAM user này là lab theo yêu cầu chương trình."

Đó là câu trả lời tốt cả về syllabus lẫn technical truth hiện tại.

---

# 51. Kịch bản nói về EBS

> "Em không coi snapshot tồn tại là đủ chứng minh backup. Em attach volume, xác định chính xác block device, tạo filesystem, mount và ghi file test. Sau đó em tạo snapshot, tạo một volume mới từ snapshot, attach, mount volume restored mà không format lại, rồi đọc file và so checksum. Chỉ khi data đọc lại đúng thì em mới kết luận recovery thành công."

Đây là tư duy backup/restore đúng.

---

# 52. Checklist cuối Unit 3

Trước khi sang Unit 4, state cuối nên là:

```text
IAM
✓ limited IAM user created
✓ group-based policy
✓ MFA assigned
✓ no AdministratorAccess
✓ EC2 Owner tag configured

VPC
✓ existing VPC/subnets/RT/IGW explained
✓ create screens demonstrated but Cancelled
✓ no duplicate VPC resources created

Security
✓ Tomcat 8080 initially reachable
✓ inbound 8080 removed
✓ external blocked
✓ localhost still healthy
✓ rule restored
✓ external connectivity restored
✓ SSH/22 never touched

EC2
✓ Running
✓ user-data bootstrap explained
✓ cloud-init status done
✓ SSH key access understood

EBS
✓ lab volume attached
✓ filesystem created safely
✓ mounted
✓ data created
✓ snapshot created
✓ new volume restored
✓ recovered data validated
✓ temporary restored volume cleaned up

Monitoring/Cost
✓ CloudWatch CPU alarm
✓ monthly AWS Budget
✓ public IPv4 cost understood
✓ EC2 deliberately left Running because Unit 4–6 depend on it
```

Đặc biệt, **đừng stop EC2 cuối Unit 3 trong full exam**. Việc stop thật nên để sau Unit 6/Self Q&A, vì guide tiếp theo còn yêu cầu trực tiếp Tomcat, PostgreSQL và monitoring trên chính lab đó. Đây là điểm mà việc “làm đúng checklist” phải nhường cho dependency/change-control hợp lý.

Unit 3 như trên bao phủ toàn bộ yêu cầu Day 7–11 và practical Unit 3, đồng thời tận dụng chính VPC/EC2/User Data chúng ta đã dựng thay vì tạo resource trùng lặp.
