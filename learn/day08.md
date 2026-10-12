# Day 8 — AWS VPC Networking Foundations

Theo MASTER_SYLLABUS, **Day 8** bắt buộc gồm:

- **Concept/Lecture:** `VPC`, `public/private subnet concepts`, `route tables`, `internet gateway`, `security groups`, `NACL awareness`, và `basic network flow`.
- **Assignment/Lab:** xây dựng một thiết kế VPC + subnet đơn giản, sau đó **validate allowed và blocked connectivity**. 00_MASTER_SYLLABUS_Java_Fresher…

Tôi đã đối chiếu nội dung này với tài liệu AWS hiện hành. Bài hôm nay sẽ đi từ nền tảng đến mức bạn có thể **thiết kế → tạo → kiểm tra → cố tình làm hỏng → troubleshoot → khôi phục → dọn tài nguyên** một VPC đơn giản.

---

# 1. Syllabus Alignment

## Mandatory — bắt buộc Day 8

Hôm nay chúng ta phải thành thạo:

1. Amazon VPC là gì.
2. IPv4 CIDR và chia subnet cơ bản.
3. Public subnet và private subnet khác nhau ở đâu.
4. Route table:
   - local route;
   - `0.0.0.0/0`;
   - destination;
   - target;
   - association;
   - main vs custom route table.
5. Internet Gateway — IGW.
6. Security Group — SG.
7. Network ACL — NACL ở mức awareness.
8. Luồng packet cơ bản.
9. Xây VPC thực tế.
10. Chứng minh:
    - một kết nối **được phép**;
    - cùng kết nối đó sau khi thay security control bị **chặn**;
    - khôi phục connectivity.

## Supplementary / Beyond the explicit syllabus

Tôi sẽ thêm một số kiến thức phục vụ operations và troubleshooting:

- longest-prefix match;
- AWS-reserved subnet IPs;
- Reachability Analyzer;
- NAT Gateway awareness;
- VPC Flow Logs awareness;
- safe change/rollback methodology;
- một EC2 cực nhỏ làm **test fixture** để kiểm chứng network.

EC2 lifecycle đầy đủ thuộc **Day 9**, nên hôm nay ta chỉ dùng EC2 như một endpoint kiểm tra, không thay thế nội dung Day 9.

---

# 2. Mục tiêu sau Day 8

Kết thúc Day 8, bạn phải có thể nhìn một sơ đồ như:

```text
Internet
   |
   |
Internet Gateway
   |
+---------------------------------------+
| VPC 10.8.0.0/16                      |
|                                       |
|  Public Subnet       Private Subnet   |
|  10.8.1.0/24         10.8.2.0/24     |
|       |                    |          |
|  Public RT              Private RT    |
|  local                  local         |
|  0.0.0.0/0 -> IGW                     |
|       |                               |
|     EC2                               |
|     + SG                              |
+---------------------------------------+
```

và trả lời ngay:

> Instance nào có khả năng đi Internet?  
> Route nào quyết định?  
> SG tác động ở đâu?  
> NACL tác động ở đâu?  
> Nếu `curl` timeout thì kiểm tra layer nào trước?

Đây mới là mục tiêu Day 8.

---

# 3. Mental Model: hãy coi VPC là mạng riêng của bạn trong AWS

Amazon VPC — **Virtual Private Cloud** — là một mạng logic được cô lập mà trong đó bạn triển khai các AWS resources.

Một subnet là một dải địa chỉ IP bên trong VPC và một subnet thuộc **một Availability Zone**. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/?utm_source=chatgpt.com)

Hãy liên hệ với mạng truyền thống:

| Mạng truyền thống      | AWS VPC                                     |
| ---------------------- | ------------------------------------------- |
| Network                | VPC                                         |
| VLAN / IP subnet       | Subnet                                      |
| Router routing table   | VPC Route Table                             |
| Internet-facing router | Internet Gateway                            |
| Host firewall          | Security Group, gần tương đương về mục đích |
| Subnet firewall/ACL    | Network ACL                                 |
| Server NIC             | ENI — Elastic Network Interface             |
| IP range               | CIDR                                        |

Nhưng đừng cố coi VPC giống hoàn toàn một switch/router vật lý. Đây là **software-defined networking do AWS quản lý**.

Mental model quan trọng:

```text
WHO wants to talk
       ↓
WHERE is destination?
       ↓
IS THERE A ROUTE?
       ↓
DO subnet controls allow it?
       ↓
DO resource controls allow it?
       ↓
IS application actually listening?
```

Khi troubleshoot:

```text
Client
 ↓
IP / DNS
 ↓
Route
 ↓
IGW / other network path
 ↓
NACL
 ↓
Security Group
 ↓
ENI / Instance
 ↓
OS firewall
 ↓
Listening port
 ↓
Application
```

Không được nhảy thẳng đến “AWS network lỗi”.

---

# 4. VPC CIDR — nền móng của toàn bộ thiết kế

Ví dụ VPC hôm nay:

```text
10.8.0.0/16
```

CIDR `/16` có nghĩa:

```text
Network bits = 16
Host bits    = 16
```

Số IPv4 addresses:

```text
2^(32 - 16)
= 65,536
```

Ta không cần dùng hết.

Có thể chia:

```text
VPC: 10.8.0.0/16

Public subnet:
10.8.1.0/24

Private subnet:
10.8.2.0/24
```

Hai subnet không được overlap.

AWS hiện cho phép IPv4 subnet CIDR từ `/16` đến `/28`. Với subnet thông thường, AWS giữ lại **5 địa chỉ** trong mỗi subnet. Ví dụ `10.8.1.0/24`: [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-sizing.html?utm_source=chatgpt.com)

```text
10.8.1.0     network address
10.8.1.1     VPC router
10.8.1.2     AWS DNS-related reservation
10.8.1.3     reserved for future use
...
10.8.1.255   reserved; VPC doesn't support broadcast
```

Vậy `/24` có:

```text
256 total addresses
251 normally usable addresses
```

### Một lỗi fresher rất phổ biến

Cho rằng:

```text
10.8.1.0/24
10.8.1.128/25
```

là hai subnet độc lập hợp lệ.

Sai, vì `/25` thứ hai nằm bên trong `/24` đầu tiên → **overlap**.

---

# 5. Public Subnet và Private Subnet

Đây là phần quan trọng nhất Day 8.

Một subnet được coi là **public subnet** khi route table của nó có direct route đến Internet Gateway.

Ví dụ:

```text
Destination      Target
10.8.0.0/16      local
0.0.0.0/0        igw-xxxxxxxx
```

Private subnet không có direct route tới IGW:

```text
Destination      Target
10.8.0.0/16      local
```

AWS định nghĩa public/private subnet dựa trên routing theo đúng nguyên tắc trên. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

### Rất quan trọng

**Public subnet ≠ mọi EC2 trong đó đều public.**

Để EC2 IPv4 trực tiếp giao tiếp với Internet qua IGW, ít nhất phải có:

```text
1. Subnet route → IGW
2. Instance có public IPv4 / Elastic IP
3. Security Group cho phép traffic cần thiết
4. NACL không block
5. OS/application hoạt động
```

Chỉ attach IGW vào VPC chưa đủ. Chỉ cho EC2 public IP cũng chưa đủ. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

Đây là câu lý thuyết rất dễ xuất hiện trong kiểm tra.

---

# 6. Internet Gateway — IGW

Internet Gateway là VPC component cho phép VPC giao tiếp với Internet.

AWS mô tả IGW là horizontally scaled, redundant và highly available; bạn attach IGW vào VPC rồi sử dụng nó làm target của route. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

Ví dụ:

```text
VPC
 |
 +--- Internet Gateway
```

Nhưng:

```text
attach IGW
```

không đồng nghĩa:

```text
Internet connectivity automatically works
```

Cần route:

```text
0.0.0.0/0 → igw-xxxx
```

Trong IPv4, Internet Gateway còn thực hiện mapping/NAT giữa public IPv4 của EC2 và private IPv4 tương ứng. EC2 về phía OS chủ yếu thấy private IPv4 của ENI; IGW xử lý mapping cho Internet communication. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

---

# 7. Route Table — “packet này phải đi đâu?”

Route table chứa các rule:

```text
Destination → Target
```

Ví dụ:

```text
Destination      Target
10.8.0.0/16      local
0.0.0.0/0        igw-0123456789abcdef
```

Ý nghĩa:

```text
Traffic tới 10.8.x.x
→ ở trong VPC
→ local

Traffic IPv4 khác
→ IGW
```

Mỗi subnet phải được associate với một route table. Một subnet tại một thời điểm chỉ dùng một subnet route table, nhưng một route table có thể phục vụ nhiều subnet. Nếu không explicit association, subnet sử dụng **main route table**. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-route-tables.html?utm_source=chatgpt.com)

---

# 8. `local` route

Khi tạo VPC:

```text
10.8.0.0/16
```

route table của VPC có route dạng:

```text
10.8.0.0/16 → local
```

Nó cho phép routing giữa các resources thuộc VPC CIDR.

Ví dụ:

```text
EC2 A
10.8.1.10

EC2 B
10.8.2.20
```

Hai subnet khác nhau nhưng cùng VPC:

```text
10.8.1.10
   ↓
10.8.0.0/16 → local
   ↓
10.8.2.20
```

Tuy nhiên:

> Có route không có nghĩa là SG sẽ cho phép.

Routing nói:

> Có đường đi.

Security nói:

> Có được phép đi không?

Hai thứ hoàn toàn khác nhau.

---

# 9. Default route `0.0.0.0/0`

CIDR:

```text
0.0.0.0/0
```

match mọi IPv4 address.

Nó thường có ý nghĩa:

```text
"If no more-specific route matches,
send IPv4 traffic here."
```

Ví dụ:

```text
10.8.0.0/16 → local
0.0.0.0/0   → igw
```

Nếu destination:

```text
10.8.2.5
```

cả hai technically có thể nằm trong phạm vi của `/0`, nhưng:

```text
10.8.0.0/16
```

cụ thể hơn `/0`.

AWS dùng nguyên tắc **longest prefix match** — route cụ thể nhất thắng. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/route-tables-priority.html?utm_source=chatgpt.com)

Vì vậy:

```text
10.8.2.5
→ local

8.8.8.8
→ IGW
```

---

# 10. Main Route Table và Custom Route Table

Khi VPC được tạo, nó có một **main route table**.

Subnet không explicit associate custom route table sẽ dùng main RT.

AWS cũng cho phép tạo custom route tables.

Trong thực tế, một pattern tốt cho lab là:

```text
main RT
  local only

public-rt
  local
  0.0.0.0/0 → IGW

private-rt
  local only
```

Sau đó:

```text
public-subnet  → public-rt
private-subnet → private-rt
```

Điều này làm network intent rõ ràng hơn.

AWS cũng khuyến nghị explicit custom route table associations như một cách giảm accidental routing. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-route-tables.html?utm_source=chatgpt.com)

---

# 11. Security Group — resource-level stateful firewall

Security Group áp dụng cho resources/network interfaces.

SG rules có:

```text
Protocol
Port
Source / Destination
```

Ví dụ web server:

```text
Inbound

TCP 80
Source = YOUR_PUBLIC_IP/32
```

Một security group mới không có inbound rules; outbound mặc định thường là allow all. SG chỉ có **ALLOW**, không có explicit `DENY`. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html?utm_source=chatgpt.com)

## SG là stateful

Giả sử inbound:

```text
Client ephemeral port 53144
     ↓
EC2 TCP/80
```

SG cho phép request TCP/80.

Response:

```text
EC2 TCP/80
     ↓
Client TCP/53144
```

Security Group biết đây là response của established traffic và cho response traffic tương ứng. Đây chính là ý nghĩa quan trọng của **stateful**. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-connection-tracking.html?utm_source=chatgpt.com)

---

# 12. Security Group không phải Route Table

Ví dụ:

```text
Public subnet:
0.0.0.0/0 → IGW

SG:
TCP/80 from your IP
```

TCP/80:

```text
Route exists
+
SG allows
→ possible
```

TCP/22:

```text
Route exists
+
SG does NOT allow
→ blocked
```

Đây chính là failure injection chúng ta sẽ thực hành.

---

# 13. Security Group reference

Một SG rule không nhất thiết phải dùng IP CIDR.

Ví dụ về sau:

```text
sg-web
   |
   | TCP/5432
   v
sg-database
```

DB SG có thể có rule:

```text
TCP 5432
Source = sg-web
```

thay vì:

```text
10.8.1.0/24
```

Đây là pattern rất quan trọng khi sang Java + PostgreSQL.

Security Group hỗ trợ reference đến SG khác trong những tình huống được AWS hỗ trợ. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html?utm_source=chatgpt.com)

---

# 14. Network ACL — NACL

NACL hoạt động ở **subnet level**.

So sánh:

| Feature        | Security Group           | NACL                     |
| -------------- | ------------------------ | ------------------------ |
| Scope          | resource/ENI             | subnet                   |
| Rules          | allow only               | allow + deny             |
| Stateful       | Yes                      | No                       |
| Rule order     | aggregate rules          | lowest rule number first |
| Return traffic | automatically recognized | phải cho phép riêng      |
| Primary use    | resource access          | subnet-level guardrail   |

AWS xem Security Groups là primary access-control mechanism và NACL phù hợp với coarse-grained subnet controls / defense-in-depth. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html?utm_source=chatgpt.com)

---

# 15. NACL là stateless

Đây là điểm dễ sai nhất.

Nếu NACL inbound cho phép:

```text
TCP 80
```

thì không có nghĩa outbound response tự động được phép.

Bạn còn phải đảm bảo outbound NACL cho phép response tới **client ephemeral port**.

AWS documentation nêu rõ NACL là stateless và phải có rules phù hợp cho response traffic. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/custom-network-acl.html?utm_source=chatgpt.com)

Ví dụ:

```text
Client
192.0.2.50:51234
   |
   | TCP
   v
Server
10.8.1.10:80
```

Inbound NACL:

```text
source 192.0.2.50
dest TCP/80
ALLOW
```

Response:

```text
Server :80
→ client :51234
```

Outbound NACL phải cho phép destination ephemeral port tương ứng.

Vì vậy custom NACL configuration rất dễ gây lỗi kiểu:

> inbound đúng mà vẫn timeout.

---

# 16. NACL rule order

NACL có numbered rules.

Ví dụ:

```text
100 DENY TCP 22 0.0.0.0/0
200 ALLOW ALL    0.0.0.0/0
*   DENY ALL
```

SSH match rule 100:

```text
DENY
```

AWS ngừng evaluate ngay sau first matching rule.

Ngược lại:

```text
100 ALLOW ALL
200 DENY TCP 22
```

TCP/22 đã match 100 → allow.

Rule 200 không bao giờ có cơ hội block nó.

AWS evaluate NACL rules từ số nhỏ nhất đến lớn hơn, dừng tại first match. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-network-acls.html?utm_source=chatgpt.com)

---

# 17. Default NACL

Default NACL của một VPC mặc định cho phép inbound/outbound traffic thông qua numbered allow rules, sau đó có immutable `* DENY` fallback. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/default-network-acl.html?trk=article-ssr-frontend-pulse_little-text-block&utm_source=chatgpt.com)

Day 8 lab của chúng ta **không chỉnh NACL mặc định**.

Lý do:

> Hôm nay mục tiêu là hiểu NACL và phân biệt nó với SG; ta không cần tạo thêm một nguồn lỗi stateless không cần thiết trong lab đầu tiên.

---

# 18. Network Flow 1 — Internet → Public EC2

Giả sử:

```text
Your laptop
203.0.113.25

EC2 public IPv4
54.x.x.x

EC2 private IPv4
10.8.1.10

Application
TCP/80
```

Luồng khái niệm:

```text
Laptop
203.0.113.25
       |
       | TCP/80
       v
Internet
       |
       v
Internet Gateway
       |
       | public IPv4 mapped to private address
       v
VPC routing
       |
       v
Public Subnet
       |
       +-- NACL permits?
       |
       v
Security Group permits TCP/80
from 203.0.113.25/32?
       |
       v
ENI: 10.8.1.10
       |
       v
OS
       |
       v
nginx listening TCP/80
```

Nếu bất kỳ layer cần thiết nào fail:

```text
request fails
```

---

# 19. Network Flow 2 — Public EC2 → Internet

Instance:

```text
10.8.1.10
```

muốn đến:

```text
1.1.1.1:443
```

Route lookup:

```text
10.8.0.0/16 → local
0.0.0.0/0   → IGW
```

`1.1.1.1` không thuộc VPC CIDR.

Do đó:

```text
0.0.0.0/0 → IGW
```

Sau đó còn phải đáp ứng public-IP và security requirements.

---

# 20. Network Flow 3 — Private Subnet

Private RT:

```text
Destination     Target
10.8.0.0/16     local
```

Instance muốn tới:

```text
1.1.1.1
```

Route lookup:

```text
10.8.0.0/16
```

không match.

Không có:

```text
0.0.0.0/0
```

Kết quả:

```text
NO ROUTE
```

### Câu hỏi rất quan trọng

Nếu instance private subnet có public IP thì sao?

Vẫn chưa đủ.

Không có route từ subnet tới Internet Gateway thì Internet communication qua IGW không hoạt động. AWS documentation nói rõ ngay cả có public IP, subnet không có route IGW vẫn không giao tiếp Internet qua IGW. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

---

# 21. Supplementary — NAT Gateway

Private subnet thường vẫn có nhu cầu:

```text
dnf update
apt update
download dependency
access external APIs
```

nhưng không muốn Internet initiate connection trực tiếp tới instance.

Một kiến trúc phổ biến:

```text
Private EC2
     |
private RT
0.0.0.0/0
     |
     v
NAT Gateway
(public subnet)
     |
     v
IGW
     |
 Internet
```

Private route:

```text
0.0.0.0/0 → nat-xxxxxxxx
```

Public route:

```text
0.0.0.0/0 → igw-xxxxxxxx
```

AWS documents mô hình này cho private-subnet outbound access. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/route-table-options.html?utm_source=chatgpt.com)

**Không tạo NAT Gateway trong lab hôm nay.**

Nó vượt phạm vi explicit Day 8 và tạo thêm cost.

---

# 22. Kiến trúc Lab Day 8

Ta xây:

```text
                     INTERNET
                         |
                         |
                   +-----+-----+
                   |    IGW    |
                   +-----+-----+
                         |
              +----------+-----------+
              | VPC 10.8.0.0/16      |
              |                      |
     +--------+--------+   +---------+--------+
     | Public Subnet   |   | Private Subnet  |
     | 10.8.1.0/24     |   | 10.8.2.0/24     |
     +--------+--------+   +---------+--------+
              |                      |
       Public Route Table      Private Route Table
              |                      |
     10.8.0.0/16 local       10.8.0.0/16 local
     0.0.0.0/0 -> IGW
              |
           Test EC2
              |
         sg-day8-web
              |
         TCP/80 from
         YOUR_IP/32
```

---

# 23. Lab assumptions

Ta giả định Day 7 đã hoàn tất:

```text
AWS account
approved IAM credentials
MFA where required
AWS CLI configured
approved Region
tagging convention
```

Kiểm tra:

```bash
aws sts get-caller-identity
```

Kiểm tra Region:

```bash
aws configure get region
```

Nếu output rỗng:

```bash
aws configure
```

Không nhập hoặc gửi access key/password/token cho tôi.

---

# 24. Cost & Safety Warning

Lab AWS là thao tác tạo cloud resources.

Trước khi làm:

- dùng lab/non-production account nếu có;
- kiểm tra Region;
- dùng naming prefix riêng;
- không mở SSH `0.0.0.0/0`;
- terminate EC2 sau lab;
- xóa resources khi hoàn thành.

Internet Gateway tự nó không có hourly charge, nhưng data transfer có thể phát sinh; AWS hiện cũng tính phí public IPv4. Trang pricing hiện niêm yết **$0.005/address-hour** cho in-use và idle public IPv4, mặc dù benefit/Free Tier của account cụ thể có thể ảnh hưởng bill thực tế. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

Vì vậy:

> Không để test EC2/public IPv4 chạy quên sau lab.

---

# 25. Bước 1 — xác nhận identity và Region

```bash
aws sts get-caller-identity
```

**Expected invariant:**

Bạn phải thấy đúng AWS account/ARN được phép làm lab.

Sau đó:

```bash
REGION=$(aws configure get region)
echo "$REGION"
```

Nếu bạn muốn kiểm tra bằng CLI:

```bash
aws ec2 describe-regions \
  --query 'Regions[].RegionName' \
  --output table
```

Không chọn Region ngẫu nhiên nếu tổ chức đã quy định Region.

---

# 26. Bước 2 — chọn Availability Zone

```bash
AZ=$(aws ec2 describe-availability-zones \
  --filters Name=state,Values=available \
  --query 'AvailabilityZones[0].ZoneName' \
  --output text)

echo "$AZ"
```

**Example output:**

```text
ap-southeast-1a
```

Đây chỉ là ví dụ.

AZ thực tế phụ thuộc account và Region.

---

# 27. Bước 3 — Create VPC

```bash
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.8.0.0/16 \
  --tag-specifications \
  'ResourceType=vpc,Tags=[{Key=Name,Value=day8-vpc},{Key=Environment,Value=lab},{Key=ManagedBy,Value=student}]' \
  --query 'Vpc.VpcId' \
  --output text)

echo "$VPC_ID"
```

Ví dụ:

```text
vpc-0123456789abcdef0
```

Xác minh:

```bash
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID" \
  --query 'Vpcs[0].[VpcId,CidrBlock,State]' \
  --output table
```

Expected invariant:

```text
CIDR  = 10.8.0.0/16
State = available
```

---

# 28. Bước 4 — tạo public subnet

```bash
PUBLIC_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 10.8.1.0/24 \
  --availability-zone "$AZ" \
  --tag-specifications \
  'ResourceType=subnet,Tags=[{Key=Name,Value=day8-public-subnet},{Key=Environment,Value=lab}]' \
  --query 'Subnet.SubnetId' \
  --output text)

echo "$PUBLIC_SUBNET_ID"
```

---

# 29. Bước 5 — tạo private subnet

```bash
PRIVATE_SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id "$VPC_ID" \
  --cidr-block 10.8.2.0/24 \
  --availability-zone "$AZ" \
  --tag-specifications \
  'ResourceType=subnet,Tags=[{Key=Name,Value=day8-private-subnet},{Key=Environment,Value=lab}]' \
  --query 'Subnet.SubnetId' \
  --output text)

echo "$PRIVATE_SUBNET_ID"
```

Verify:

```bash
aws ec2 describe-subnets \
  --subnet-ids "$PUBLIC_SUBNET_ID" "$PRIVATE_SUBNET_ID" \
  --query 'Subnets[].{SubnetId:SubnetId,CIDR:CidrBlock,AZ:AvailabilityZone}' \
  --output table
```

Expected:

```text
day8-public  → 10.8.1.0/24
day8-private → 10.8.2.0/24
```

---

# 30. Public IP assignment nuance

Nondefault subnets mặc định thường không auto-assign public IPv4. AWS documents default behavior này rõ ràng. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-public-ip.html?utm_source=chatgpt.com)

Cho lab public subnet:

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --map-public-ip-on-launch
```

Không bật cho private subnet.

Verify:

```bash
aws ec2 describe-subnets \
  --subnet-ids "$PUBLIC_SUBNET_ID" "$PRIVATE_SUBNET_ID" \
  --query 'Subnets[].{Subnet:SubnetId,CIDR:CidrBlock,PublicIPv4:MapPublicIpOnLaunch}' \
  --output table
```

Expected:

```text
public subnet  → true
private subnet → false
```

Nhưng nhớ:

> `MapPublicIpOnLaunch=true` không làm subnet trở thành public.

**Route tới IGW mới là yếu tố phân loại public subnet.**

---

# 31. Bước 6 — Create Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications \
  'ResourceType=internet-gateway,Tags=[{Key=Name,Value=day8-igw},{Key=Environment,Value=lab}]' \
  --query 'InternetGateway.InternetGatewayId' \
  --output text)

echo "$IGW_ID"
```

Attach:

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID"
```

Verify:

```bash
aws ec2 describe-internet-gateways \
  --internet-gateway-ids "$IGW_ID" \
  --query 'InternetGateways[0].Attachments' \
  --output table
```

Expected:

```text
VPC ID = your VPC
State  = available
```

---

# 32. Bước 7 — tạo Public Route Table

```bash
PUBLIC_RTB_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
  'ResourceType=route-table,Tags=[{Key=Name,Value=day8-public-rt},{Key=Environment,Value=lab}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

echo "$PUBLIC_RTB_ID"
```

Ngay khi tạo, nó có local route.

Xem:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PUBLIC_RTB_ID" \
  --query 'RouteTables[0].Routes' \
  --output table
```

Expected:

```text
10.8.0.0/16 → local
```

---

# 33. Bước 8 — thêm Internet route

```bash
aws ec2 create-route \
  --route-table-id "$PUBLIC_RTB_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$IGW_ID"
```

AWS official example cũng sử dụng chính cấu hình:

```text
0.0.0.0/0 → Internet Gateway
```

để tạo Internet route cho public subnet. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/create-vpc.html?utm_source=chatgpt.com)

Verify:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PUBLIC_RTB_ID" \
  --query 'RouteTables[0].Routes' \
  --output table
```

Bạn cần nhìn thấy:

```text
10.8.0.0/16 → local
0.0.0.0/0   → igw-...
```

---

# 34. Bước 9 — associate public subnet

```bash
PUBLIC_ASSOC_ID=$(aws ec2 associate-route-table \
  --route-table-id "$PUBLIC_RTB_ID" \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --query 'AssociationId' \
  --output text)

echo "$PUBLIC_ASSOC_ID"
```

Đây là thời điểm subnet trở thành public theo routing definition.

---

# 35. Bước 10 — tạo Private Route Table

```bash
PRIVATE_RTB_ID=$(aws ec2 create-route-table \
  --vpc-id "$VPC_ID" \
  --tag-specifications \
  'ResourceType=route-table,Tags=[{Key=Name,Value=day8-private-rt},{Key=Environment,Value=lab}]' \
  --query 'RouteTable.RouteTableId' \
  --output text)

echo "$PRIVATE_RTB_ID"
```

**Không tạo:**

```text
0.0.0.0/0 → IGW
```

Associate:

```bash
PRIVATE_ASSOC_ID=$(aws ec2 associate-route-table \
  --route-table-id "$PRIVATE_RTB_ID" \
  --subnet-id "$PRIVATE_SUBNET_ID" \
  --query 'AssociationId' \
  --output text)

echo "$PRIVATE_ASSOC_ID"
```

Verify:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PRIVATE_RTB_ID" \
  --query 'RouteTables[0].Routes' \
  --output table
```

Expected:

```text
10.8.0.0/16 → local
```

và **không có**:

```text
0.0.0.0/0 → IGW
```

---

# 36. Bước 11 — kiểm chứng association

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PUBLIC_RTB_ID" "$PRIVATE_RTB_ID" \
  --query 'RouteTables[].{RT:RouteTableId,Routes:Routes,Associations:Associations}' \
  --output json
```

Bạn phải có bằng chứng:

```text
public subnet
   → public RT
   → IGW default route

private subnet
   → private RT
   → local only
```

Đừng chỉ nói:

> “Tôi nghĩ đã cấu hình đúng.”

Operations yêu cầu evidence.

---

# 37. Bước 12 — inspect NACL

Không sửa NACL.

Chỉ quan sát:

```bash
aws ec2 describe-network-acls \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --output table
```

Tìm:

```text
IsDefault = true
```

và inbound/outbound rules.

Mục tiêu là nhận ra:

```text
NACL exists
NACL associates with subnets
NACL != SG
NACL is stateless
```

---

# 38. Bước 13 — tạo Security Group cho test web server

```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name day8-web-sg \
  --description "Day 8 web connectivity validation" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' \
  --output text)

echo "$SG_ID"
```

Tag:

```bash
aws ec2 create-tags \
  --resources "$SG_ID" \
  --tags Key=Name,Value=day8-web-sg Key=Environment,Value=lab
```

AWS security group mới mặc định không có inbound access cho tới khi bạn thêm rule. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/creating-security-groups.html?utm_source=chatgpt.com)

---

# 39. Xác định public IPv4 của máy bạn

Nếu máy có `curl`:

```bash
MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "$MY_IP"
```

Giả sử:

```text
203.0.113.25
```

Ta sẽ dùng:

```text
203.0.113.25/32
```

**Không dùng example address ở trên làm rule thật.**

Phải dùng public IP hiện tại của chính bạn.

---

# 40. Cho phép HTTP chỉ từ máy bạn

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr "${MY_IP}/32"
```

Verify:

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --query 'SecurityGroups[0].IpPermissions' \
  --output json
```

Bạn cần thấy:

```text
TCP
80
YOUR_PUBLIC_IP/32
```

Không cần:

```text
0.0.0.0/0
```

cho lab này.

---

# 41. Minimal EC2 test fixture

Đây là một **preview nhỏ của Day 9**, chỉ để có thứ nhận packet.

Trong EC2 Console:

```text
EC2 → Instances → Launch instance
```

Chọn:

```text
Name:
day8-web-test

AMI:
Amazon Linux 2023

Instance type:
smallest approved / Free Tier eligible option shown for your account

Network:
day8-vpc

Subnet:
day8-public-subnet

Auto-assign public IP:
Enable

Security Group:
existing day8-web-sg
```

Không cần mở SSH nếu bạn chỉ sử dụng endpoint để test HTTP.

---

# 42. User Data cho test fixture

Trong Advanced details → User data:

```bash
#!/bin/bash
dnf install -y nginx
echo 'Day 8 VPC connectivity works' > /usr/share/nginx/html/index.html
systemctl enable --now nginx
```

Sau khi boot:

```text
nginx
→ TCP/80
```

Đây chỉ là test fixture. User data và EC2 bootstrap sẽ được học đúng scope ở Day 9.

---

# 43. VALIDATION 1 — Allowed Connectivity

Lấy public IPv4 của instance.

Ví dụ:

```text
54.x.x.x
```

Từ máy local:

```bash
curl -v --connect-timeout 5 http://PUBLIC_IP/
```

Expected application response:

```text
Day 8 VPC connectivity works
```

Quan trọng hơn nội dung HTML là:

```text
TCP connection established
HTTP response received
```

### Evidence cần lưu

```text
VPC ID
Public subnet ID
Private subnet ID
IGW ID
Public route table
Private route table
SG inbound rule
EC2 public IPv4
curl success
```

---

# 44. Giải thích tại sao test thành công

Hãy tự trace:

```text
curl
 ↓
PUBLIC_IP:80
 ↓
Internet
 ↓
Internet Gateway
 ↓
public subnet route architecture
 ↓
NACL allows
 ↓
SG allows TCP/80 from MY_IP/32
 ↓
EC2
 ↓
nginx listens :80
 ↓
HTTP 200
```

Đừng chỉ ghi:

> `curl works`.

Phải giải thích **tại sao** works.

---

# 45. FAILURE INJECTION 1 — Security Group block

Bây giờ cố tình remove HTTP rule.

Trước change:

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --output json
```

Sau đó:

```bash
aws ec2 revoke-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr "${MY_IP}/32"
```

Verify rule đã biến mất:

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --query 'SecurityGroups[0].IpPermissions' \
  --output json
```

---

# 46. VALIDATION 2 — Blocked Connectivity

```bash
curl -v --connect-timeout 5 http://PUBLIC_IP/
```

Expected invariant:

```text
Không thiết lập được HTTP TCP connection như trước.
```

Thông thường biểu hiện sẽ là timeout.

Ta vừa tạo controlled experiment:

```text
Before:
network path + SG allow
→ success

After:
same client
same EC2
same nginx
same route
same IP
SG rule removed
→ failure
```

Đây là troubleshooting methodology rất tốt vì chỉ thay đổi **một variable**.

---

# 47. Root Cause Statement

Không viết:

> “Network failed.”

Viết:

> TCP/80 connectivity từ `MY_IP/32` tới `day8-web-test` bị chặn sau khi inbound TCP/80 rule của `day8-web-sg` bị revoke. Route, public IPv4 và application endpoint không thay đổi. Root cause là Security Group không còn rule cho phép TCP/80 từ source CIDR đã kiểm thử.

Đây là cách viết incident evidence chuyên nghiệp.

---

# 48. Recovery

Restore rule:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr "${MY_IP}/32"
```

Verify:

```bash
curl -v --connect-timeout 5 http://PUBLIC_IP/
```

Expected:

```text
Day 8 VPC connectivity works
```

Hoàn tất:

```text
failure injection
→ diagnosis
→ remediation
→ validation
```

---

# 49. FAILURE INJECTION 2 — Route failure

Đây là bài rất quan trọng.

Trước hết inspect:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PUBLIC_RTB_ID" \
  --output table
```

Bạn đang có:

```text
0.0.0.0/0 → IGW
```

Trong lab/non-production, có thể cố tình remove route:

```bash
aws ec2 delete-route \
  --route-table-id "$PUBLIC_RTB_ID" \
  --destination-cidr-block 0.0.0.0/0
```

**Cảnh báo:** thao tác này làm mất Internet path của mọi resource sử dụng route table đó. Chỉ thực hiện trong VPC lab.

Test:

```bash
curl -v --connect-timeout 5 http://PUBLIC_IP/
```

Kỳ vọng:

```text
failure
```

Bây giờ SG vẫn đúng.

Application vẫn chạy.

Nhưng:

```text
routing intent bị phá
```

---

# 50. Recovery route

```bash
aws ec2 create-route \
  --route-table-id "$PUBLIC_RTB_ID" \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id "$IGW_ID"
```

Verify:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PUBLIC_RTB_ID" \
  --output table
```

Sau đó:

```bash
curl -v --connect-timeout 5 http://PUBLIC_IP/
```

Expected:

```text
success again
```

Đây là một Day 8 lab rất giá trị vì bạn đã chứng minh:

```text
route failure
≠
Security Group failure
```

---

# 51. Evidence cho private subnet

Inspect:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PRIVATE_RTB_ID" \
  --query 'RouteTables[0].Routes' \
  --output table
```

Expected:

```text
10.8.0.0/16 → local
```

Không được có:

```text
0.0.0.0/0 → IGW
```

Đó là network-design evidence rằng subnet của bạn **không có direct IPv4 Internet route**.

---

# 52. Supplementary — Reachability Analyzer

AWS có **Reachability Analyzer**, một static configuration analysis tool.

Nó không gửi real packet giống `curl`. Nó phân tích configuration và có thể chỉ ra component đang block path, ví dụ:

```text
route table
security group
NACL
...
```

AWS cho phép source/destination như:

```text
EC2 instance
network interface
internet gateway
...
```

và trả về:

```text
Reachable
```

hoặc:

````text
Not reachable
+ blocking component
``` :chatgpt-content-reference{index="23"}


AWS thậm chí có example phân tích:

```text
Internet Gateway → EC2 instance
TCP/22
``` :chatgpt-content-reference{index="24"}


Lưu ý: Reachability Analyzer **có pricing per analysis run**, nên đừng chạy lặp vô hạn. :chatgpt-content-reference{index="25"}

---

# 53. Troubleshooting Framework cho Day 8

Khi user nói:

> “Server không truy cập được.”

Không sửa ngay SG.

Đi theo quy trình:

```text
1. Symptom là gì?
2. Expected behavior là gì?
3. Source là gì?
4. Destination là gì?
5. Protocol/port?
6. Recent changes?
7. Resource nằm subnet nào?
8. Subnet dùng route table nào?
9. Route có phù hợp destination không?
10. IGW/NAT/etc có tồn tại và attached không?
11. Public/private IP đúng không?
12. NACL có block không?
13. SG có allow không?
14. OS/application có listen không?
15. Re-test
16. Preserve evidence
````

---

# 54. Scenario: Public EC2 không vào được TCP/80

Triệu chứng:

```bash
curl http://PUBLIC_IP
```

timeout.

## Hypothesis A — không có public IPv4

Check:

```bash
aws ec2 describe-instances \
  --instance-ids INSTANCE_ID \
  --query 'Reservations[0].Instances[0].[PrivateIpAddress,PublicIpAddress]' \
  --output table
```

Nếu PublicIpAddress:

```text
None
```

đó là candidate root cause.

---

# 55. Hypothesis B — wrong route table association

Check:

```bash
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=$PUBLIC_SUBNET_ID" \
  --output table
```

Bạn cần xác định subnet thực sự associate route table nào.

Không được chỉ nhìn một route table có tên:

```text
day8-public-rt
```

rồi kết luận subnet đang dùng nó.

**Association là evidence quan trọng hơn Name tag.**

---

# 56. Hypothesis C — missing IGW route

Check:

```bash
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=$PUBLIC_SUBNET_ID" \
  --query 'RouteTables[].Routes' \
  --output table
```

Tìm:

```text
0.0.0.0/0 → igw-...
```

Nếu không có:

```text
no Internet default route
```

---

# 57. Hypothesis D — IGW không attach đúng VPC

```bash
aws ec2 describe-internet-gateways \
  --internet-gateway-ids "$IGW_ID" \
  --output json
```

Check:

```text
Attachments[].VpcId
```

phải là:

```text
$VPC_ID
```

---

# 58. Hypothesis E — SG wrong source

Ví dụ laptop public IP thay đổi từ:

```text
198.51.100.20
```

sang:

```text
198.51.100.33
```

nhưng SG vẫn:

```text
198.51.100.20/32
```

Kết quả:

```text
timeout
```

Refresh:

```bash
curl -s https://checkip.amazonaws.com
```

rồi compare với SG rule.

---

# 59. Hypothesis F — SG đúng nhưng app không listen

Đây là một distinction cực kỳ quan trọng:

```text
Network path reachable
```

không đồng nghĩa:

```text
application available
```

Trên Linux server:

```bash
sudo ss -lntp
```

Tìm:

```text
:80
```

Nếu không có process listen TCP/80:

```text
SG cannot fix the application.
```

Một SG rule chỉ cho phép packet tới server.

Nó không tự tạo web server.

---

# 60. Hypothesis G — NACL block

Nếu custom NACL tồn tại:

```bash
aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=$PUBLIC_SUBNET_ID" \
  --output json
```

Check cả:

```text
Ingress
Egress
```

NACL là stateless, nên đừng chỉ kiểm tra inbound TCP/80.

Response ephemeral traffic cũng cần hợp lệ. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/custom-network-acl.html?utm_source=chatgpt.com)

---

# 61. `ping` không phải universal network test

Một fresher thường làm:

```bash
ping SERVER
```

Không reply rồi kết luận:

> Network down.

Sai.

Ping sử dụng ICMP.

Ứng dụng có thể dùng:

```text
TCP/80
TCP/443
TCP/8080
TCP/5432
```

SG có thể:

```text
allow TCP/443
deny/no-rule ICMP
```

Khi đó:

```text
ping fails
HTTPS works
```

Test đúng protocol của application:

```bash
curl -v http://host:8080
```

hoặc:

```bash
nc -vz host 5432
```

nếu `nc` có sẵn.

---

# 62. `curl` failure cũng phải đọc đúng

Ví dụ:

```text
Connection refused
```

thường gợi ý khác với:

```text
Connection timed out
```

Mental model:

```text
timeout
→ packet/path/security filtering là candidate mạnh

connection refused
→ destination reachable nhưng không có listener hoặc active reject là candidate mạnh
```

Không coi đây là tuyệt đối 100%; nhưng nó giúp hình thành hypothesis.

---

# 63. Security & Least Privilege

Không tạo:

```text
SSH TCP/22
0.0.0.0/0
```

chỉ vì:

> “Cho dễ lab.”

Nên dùng:

```text
YOUR_PUBLIC_IP/32
```

Nếu application phải Internet-facing:

```text
TCP/443 from 0.0.0.0/0
```

có thể là business requirement hợp lệ.

Nhưng management port:

```text
22
3389
5432
3306
```

không nên mở Internet-wide nếu không có kiến trúc/approval phù hợp.

---

# 64. Security Group ≠ nơi để DENY một IP

SG không có deny rules.

Nếu:

```text
allow everyone except 198.51.100.50
```

không thể trực tiếp diễn đạt bằng SG kiểu:

```text
ALLOW 0.0.0.0/0
DENY 198.51.100.50
```

vì SG không hỗ trợ explicit deny.

NACL có allow + deny nhưng phải cẩn thận về ordering và stateless response behavior. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html?utm_source=chatgpt.com)

---

# 65. Anti-pattern: “Public subnet = unsafe”

Không chính xác.

Public subnet chỉ nói:

```text
subnet route table has a route to IGW
```

Việc resource có reachable từ Internet hay không còn phụ thuộc:

```text
public IP
security group
NACL
application
other controls
```

Vì vậy:

```text
PUBLIC
```

là routing property, không phải một judgment “secure/insecure”.

---

# 66. Anti-pattern: đặt database public chỉ vì app cần kết nối

Ví dụ architecture:

```text
Internet
   |
Web/App
   |
PostgreSQL
```

App và DB cùng VPC thì chúng có thể giao tiếp thông qua private networking/local routes.

Không cần:

```text
PostgreSQL public IP
```

chỉ để Java app gọi DB.

Sau này ta hướng tới:

```text
App subnet
   |
private address
   |
DB subnet
```

với SG restricted.

---

# 67. Anti-pattern: `0.0.0.0/0` ở đâu cũng nguy hiểm như nhau

Không hoàn toàn.

Trong route table:

```text
0.0.0.0/0 → IGW
```

là default route.

Trong Security Group inbound:

```text
TCP/22 from 0.0.0.0/0
```

là cho phép cả Internet thử SSH vào port 22.

Cùng string:

```text
0.0.0.0/0
```

nhưng ý nghĩa phụ thuộc context.

---

# 68. Anti-pattern: Route Table như firewall

Route table không phải security firewall.

Route table trả lời:

> Traffic destination này đi target nào?

SG/NACL trả lời:

> Traffic này có được phép không?

Thiếu route:

```text
no network path
```

Có route nhưng SG block:

```text
path exists
authorization blocks it
```

Đây là hai failure layer khác nhau.

---

# 69. Anti-pattern: sửa nhiều thứ cùng lúc

Giả sử HTTP fail.

Fresher làm:

```text
change SG
change NACL
change route
restart EC2
restart nginx
change DNS
```

Sau đó hoạt động.

Bạn không biết root cause là gì.

Trong operations, hãy thay **một hypothesis một lần**:

```text
Evidence
→ hypothesis
→ safest test
→ result
→ next hypothesis
```

Failure injection lúc nãy chính là ví dụ chuẩn.

---

# 70. Operations Evidence Pack

Sau lab, bạn nên có một folder Day 8 chứa:

```text
01-identity.txt
02-vpc.txt
03-subnets.txt
04-igw.txt
05-public-route-table.txt
06-private-route-table.txt
07-nacl.txt
08-security-group.txt
09-http-allowed.txt
10-http-blocked.txt
11-http-recovered.txt
12-cleanup.txt
```

Ví dụ:

```bash
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID" \
  > 02-vpc.json
```

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PUBLIC_RTB_ID" "$PRIVATE_RTB_ID" \
  > 05-06-route-tables.json
```

Evidence cần chứng minh state, không chỉ chụp một màn hình không có context.

---

# 71. Rollback Principle

Giả sử change ticket nói:

> Add TCP/80 inbound from office CIDR.

Pre-change:

```text
capture existing SG
```

Change:

```text
add approved rule
```

Validation:

```text
curl TCP/80
```

Nếu unintended exposure:

```text
revoke exactly newly-added rule
```

Verify state trở về baseline.

Đây là **rollback**.

Không phải:

> restart server rồi hy vọng.

---

# 72. Cleanup Lab

Đây là bắt buộc vì AWS resource có thể phát sinh cost.

## 72.1 Terminate EC2

Trong EC2 Console:

```text
Instances
→ day8-web-test
→ Instance state
→ Terminate instance
```

Chờ:

```text
terminated
```

Không xóa SG trước khi ENI dependency được giải phóng.

---

# 73. Delete Security Group

Sau khi EC2 đã terminate và ENI không còn phụ thuộc:

```bash
aws ec2 delete-security-group \
  --group-id "$SG_ID"
```

---

# 74. Disassociate custom Route Tables

```bash
aws ec2 disassociate-route-table \
  --association-id "$PUBLIC_ASSOC_ID"
```

```bash
aws ec2 disassociate-route-table \
  --association-id "$PRIVATE_ASSOC_ID"
```

---

# 75. Delete Route Tables

```bash
aws ec2 delete-route-table \
  --route-table-id "$PUBLIC_RTB_ID"
```

```bash
aws ec2 delete-route-table \
  --route-table-id "$PRIVATE_RTB_ID"
```

Main route table của VPC không được delete như custom RT.

---

# 76. Detach và Delete Internet Gateway

```bash
aws ec2 detach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID"
```

Sau đó:

```bash
aws ec2 delete-internet-gateway \
  --internet-gateway-id "$IGW_ID"
```

---

# 77. Delete Subnets

```bash
aws ec2 delete-subnet \
  --subnet-id "$PUBLIC_SUBNET_ID"
```

```bash
aws ec2 delete-subnet \
  --subnet-id "$PRIVATE_SUBNET_ID"
```

---

# 78. Delete VPC

```bash
aws ec2 delete-vpc \
  --vpc-id "$VPC_ID"
```

Verify không còn:

```bash
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID"
```

AWS sẽ trả lỗi kiểu resource không tồn tại nếu VPC đã được xóa thành công.

Lưu evidence cleanup.

---

# 79. Nếu delete VPC thất bại

Đừng thử đi thử lại một cách mù quáng.

Thông thường vẫn còn dependency:

```text
subnet
ENI
security group dependency
internet gateway attachment
route table association
EC2
NAT gateway
other VPC resource
```

Method:

```text
delete child resources
        ↓
detach dependencies
        ↓
delete VPC last
```

Đây là một operational principle bạn sẽ gặp rất nhiều trong AWS.

---

# 80. L1 Troubleshooting Boundary

Một fresher/L1 có thể an toàn thực hiện khi runbook/approval cho phép:

```text
describe resources
inspect route tables
inspect associations
inspect SG
inspect NACL
confirm public/private IP
check application listening port
collect curl/nc output
compare current vs baseline
restore an explicitly-approved SG rule
restore known route from approved configuration
```

Không tự ý:

```text
open 0.0.0.0/0 to production DB
change enterprise network architecture
modify Transit Gateway
change organization-wide controls
disable security controls
change corporate CIDRs
bypass Network Firewall
```

Những trường hợp này phải escalate.

---

# 81. Integrated Troubleshooting Example

Ticket:

> `https://app.example.com` timeout.

Đừng chỉ kiểm tra SG.

Hãy suy nghĩ:

```text
Client
 ↓
DNS resolves?
 ↓
Correct public endpoint?
 ↓
IGW/load balancer path?
 ↓
Route table?
 ↓
NACL?
 ↓
Security Group?
 ↓
EC2?
 ↓
Linux firewall?
 ↓
Port 443 listening?
 ↓
Java/Tomcat?
 ↓
Application?
 ↓
Database downstream?
```

Day 8 chỉ tập trung đoạn:

```text
VPC
subnet
route
IGW
NACL
SG
```

Nhưng về Day 19–20, bạn phải trace toàn chain này.

---

# 82. Exam Focus — các câu bắt buộc phải trả lời được

### Q1. Điều gì làm một subnet trở thành public?

Không phải Name tag `"public"`.

Mà là:

```text
route table associated with subnet
contains direct route to Internet Gateway
```

---

### Q2. Attach IGW có tự làm EC2 Internet-reachable không?

Không.

Còn cần:

````text
appropriate route
public IPv4/IPv6
security controls
application
``` :chatgpt-content-reference{index="28"}


---

### Q3. Security Group stateful hay stateless?

```text
Stateful
````

---

### Q4. NACL stateful hay stateless?

```text
Stateless
```

---

### Q5. SG có DENY rule không?

```text
No
```

Chỉ allow rules. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html?utm_source=chatgpt.com)

---

### Q6. NACL có DENY không?

```text
Yes
```

---

### Q7. NACL evaluate rule thế nào?

```text
lowest rule number first
first match wins
```

---

### Q8. `10.8.0.0/16 → local` để làm gì?

Routing traffic cho VPC CIDR bên trong VPC.

---

### Q9. `0.0.0.0/0` là gì?

Default IPv4 CIDR:

```text
every IPv4 destination
```

subject to more-specific route matching.

---

### Q10. Nếu có:

```text
10.8.0.0/16 → local
0.0.0.0/0 → IGW
```

traffic tới `10.8.2.50` chọn route nào?

```text
10.8.0.0/16 → local
```

do longest prefix match. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/route-tables-priority.html?utm_source=chatgpt.com)

---

# 83. Knowledge Check

Hãy thử tự trả lời trước khi xem đáp án.

### 1

VPC:

```text
10.10.0.0/16
```

Subnet:

```text
10.10.1.0/24
```

RT:

```text
10.10.0.0/16 → local
```

Đây là public hay private subnet?

**Đáp án:** private theo routing definition vì không có direct route tới IGW.

### 2

Thêm:

```text
0.0.0.0/0 → IGW
```

Subnet bây giờ?

**Đáp án:** public.

### 3

EC2 trong public subnet nhưng không có public IPv4.

Có thể laptop Internet connect trực tiếp đến private IP của nó không?

**Đáp án:** không qua Internet thông thường.

### 4

EC2 có public IP, subnet có IGW route, nhưng SG không có inbound TCP/443.

HTTPS Internet vào được không?

**Đáp án:** không.

### 5

SG cho TCP/443 inbound nhưng nginx/Tomcat không listen 443.

SG có sửa được không?

**Đáp án:** không. Đây là application/socket layer problem.

### 6

NACL inbound allow TCP/443 nhưng outbound response traffic bị deny.

HTTPS có thể hoạt động bình thường không?

**Đáp án:** không, vì NACL stateless.

### 7

SG có:

```text
TCP 22 from 10.0.0.0/8
```

và:

```text
TCP 22 from 192.168.1.0/24
```

Rules được aggregate hay first-match?

**Đáp án:** aggregate. SG không dùng first-match ordered evaluation như NACL.

### 8

NACL:

```text
100 DENY TCP 22
200 ALLOW ALL
```

TCP/22?

**Đáp án:** deny.

### 9

NACL:

```text
100 ALLOW ALL
200 DENY TCP 22
```

TCP/22?

**Đáp án:** allow vì rule 100 match trước.

### 10

`ping` fail có chứng minh HTTPS fail không?

**Đáp án:** không. ICMP và TCP/443 là traffic khác nhau.

---

# 84. Independent Practical Challenge

Sau guided lab, bạn nên tự làm lại **không copy từng command**.

Yêu cầu:

```text
VPC:
10.80.0.0/16

Public subnet:
10.80.10.0/24

Private subnet:
10.80.20.0/24
```

Thiết kế:

```text
Public RT:
10.80.0.0/16 → local
0.0.0.0/0 → IGW

Private RT:
10.80.0.0/16 → local
```

Security policy:

```text
TCP/80 only from YOUR_PUBLIC_IP/32
```

Phải chứng minh:

```text
1. HTTP allowed
2. SG rule removed → HTTP blocked
3. SG restored → HTTP works
4. Internet route removed → HTTP fails
5. route restored → HTTP works
6. private RT has no IGW route
7. resources fully cleaned up
```

Evidence phải đủ để một mentor khác nhìn vào có thể xác nhận những kết luận trên.

---

# 85. Master Troubleshooting Table

| Symptom                                | Layer nghi ngờ             | Evidence                        |
| -------------------------------------- | -------------------------- | ------------------------------- |
| Public EC2 không Internet              | route/public IP/IGW        | RT, instance IP, IGW attachment |
| HTTP timeout                           | SG/NACL/route              | SG rules, NACL, route           |
| HTTP refused                           | listener/service           | `ss -lntp`, service status      |
| Private subnet không Internet          | có thể expected behavior   | private RT                      |
| Added SG rule vẫn fail                 | route/NACL/app/source CIDR | inspect whole chain             |
| SG looks correct but user IP changed   | source CIDR                | current public IP               |
| Một subnet hoạt động, subnet khác fail | RT/NACL association        | association evidence            |
| Responses fail với custom NACL         | ephemeral ports            | inbound + outbound NACL         |
| Ping fail, HTTP works                  | ICMP not allowed           | protocol-specific rules         |

---

# 86. Day 8 Command Cheat Sheet

```bash
# identity
aws sts get-caller-identity

# VPC
aws ec2 create-vpc
aws ec2 describe-vpcs
aws ec2 delete-vpc

# subnet
aws ec2 create-subnet
aws ec2 describe-subnets
aws ec2 modify-subnet-attribute
aws ec2 delete-subnet

# Internet Gateway
aws ec2 create-internet-gateway
aws ec2 attach-internet-gateway
aws ec2 describe-internet-gateways
aws ec2 detach-internet-gateway
aws ec2 delete-internet-gateway

# route tables
aws ec2 create-route-table
aws ec2 create-route
aws ec2 delete-route
aws ec2 associate-route-table
aws ec2 disassociate-route-table
aws ec2 describe-route-tables
aws ec2 delete-route-table

# Security Groups
aws ec2 create-security-group
aws ec2 authorize-security-group-ingress
aws ec2 revoke-security-group-ingress
aws ec2 describe-security-groups
aws ec2 delete-security-group

# NACL
aws ec2 describe-network-acls
```

Đừng học thuộc command theo kiểu ký tự.

Hãy nhớ object model:

```text
create object
→ obtain ID
→ attach/associate
→ modify rules
→ describe
→ validate
→ detach/disassociate
→ delete
```

---

# 87. Những invariant bạn phải nhớ

Một thiết kế Day 8 đúng phải thỏa:

```text
VPC CIDR không overlap với subnet layout của chính nó
```

```text
Each subnet → one active subnet route table
```

```text
Public subnet
→ direct route to IGW
```

```text
Private subnet
→ no direct route to IGW
```

```text
Route determines path
```

```text
Security Group controls permitted resource traffic
```

```text
NACL controls permitted subnet traffic
```

```text
Security Group = stateful
```

```text
NACL = stateless
```

```text
SG = allow only
```

```text
NACL = allow + deny
```

```text
NACL = numbered first match
```

```text
Public subnet ≠ automatically public EC2
```

---

# 88. Sơ đồ tư duy Day 8 cần thuộc

```text
                        Internet
                           |
                           |
                          IGW
                           |
                 0.0.0.0/0 route
                           |
            +--------------+--------------+
            |          VPC CIDR           |
            |         10.8.0.0/16         |
            |                             |
       Public Subnet                 Private Subnet
       10.8.1.0/24                   10.8.2.0/24
            |                             |
          NACL                          NACL
            |                             |
            |                             |
           ENI                           ENI
            |                             |
           SG                            SG
            |                             |
           EC2                           EC2
```

Nhưng packet không phải chỉ nhìn sơ đồ vật lý. Phải luôn hỏi:

```text
Destination?
→ route?

Subnet?
→ NACL?

Resource?
→ SG?

Host?
→ service listening?
```

---

# 89. Day 8 Mastery Check

Tôi **chưa coi Day 8 là mastered chỉ vì bạn đã đọc hết bài này**.

Bạn đạt mức lý thuyết khi có thể tự giải thích không nhìn tài liệu:

```text
VPC
CIDR
subnet
public vs private
route table
local route
default route
IGW
SG
NACL
stateful/stateless
basic packet flow
```

Bạn đạt mức practical khi có thể tự:

```text
create VPC
→ create public/private subnet
→ attach IGW
→ configure route tables
→ configure SG
→ validate allowed connectivity
→ inject SG failure
→ diagnose it
→ recover
→ inject route failure
→ diagnose it
→ recover
→ prove private subnet routing intent
→ clean everything up
```

Đó mới là đúng yêu cầu Assignment/Lab của Day 8. 00_MASTER_SYLLABUS_Java_Fresher…

---

## Kết luận Day 8

Nếu chỉ nhớ một framework hôm nay, hãy nhớ:

```text
               ROUTING
        "Có đường đi không?"
                 |
                 v
             SECURITY
        "Có được phép không?"
                 |
                 v
            APPLICATION
        "Có service nhận không?"
```

Và với AWS VPC:

```text
VPC
 ↓
Subnet
 ↓
Route Table
 ↓
Gateway/path
 ↓
NACL
 ↓
Security Group
 ↓
Instance
 ↓
Listening service
```

Đây chính là nền tảng network troubleshooting bạn sẽ dùng tiếp ở **Day 9 EC2**, sau đó là Tomcat/Java, PostgreSQL, monitoring và cuối cùng các integrated failure scenarios của Day 19–20.
