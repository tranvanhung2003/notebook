# Day 4 — AWS Cloud Administration: IAM, VPC và EC2

Day 4 là ngày đầu tiên của module **AWS Cloud Administration, Monitoring & Disaster Recovery**. Trọng tâm không phải học toàn bộ AWS, mà là xây dựng nền tảng vận hành AWS theo cách một Infrastructure Operations engineer cần làm: **xác định đúng identity trước khi thao tác, kiểm soát quyền theo least privilege, hiểu network path trong VPC, provision và vận hành EC2 an toàn, verify cả allowed/denied behavior, đồng thời thu thập audit evidence mà không làm lộ credentials**. Sang Day 5 mới mở rộng sang EBS, S3 backup/DR, CloudWatch, CloudTrail và cost control. Infrastructure_Operations_14_Da…

Theo syllabus, Day 4 bao gồm toàn bộ các nhóm kiến thức về **AWS global infrastructure, shared responsibility, AWS account/resource governance, IAM, MFA, temporary credentials, AWS CLI, tagging, VPC networking, EC2, AMI, key pair, instance profile, user data, instance metadata/IMDSv2, SSH, Linux/RHEL bootstrap, security hardening và basic EC2 troubleshooting**. Lab phải thực hành IAM/MFA/CLI, thiết kế VPC, provision EC2, kiểm tra network, metadata protection, provisioning logs, allowed/denied connectivity và thu thập evidence. Infrastructure_Operations_14_Da…

Tôi chia toàn bộ Day 4 thành đúng **2 module lớn: Lý thuyết và Thực hành** như sau.

---

# MODULE 1 — LÝ THUYẾT

Module này trả lời ba câu hỏi lớn:

**Ai đang thao tác? → IAM**  
**Resource nằm ở đâu và traffic đi bằng đường nào? → VPC**  
**Compute workload được tạo, truy cập và bảo vệ thế nào? → EC2**

## 1. AWS Global Infrastructure

Bạn cần hiểu mô hình phân cấp cơ bản:

**AWS → Region → Availability Zone → AWS resources**

### Region

`Region` là một khu vực địa lý độc lập, ví dụ như `ap-southeast-1`.

Cần hiểu:

- resource nào mang tính `Regional`;
- resource nào thuộc một `Availability Zone`;
- tại sao Region ảnh hưởng tới:
  - availability;
  - latency;
  - regulatory/data residency;
  - disaster recovery;
  - service availability;
  - operational scope.

Điểm vận hành quan trọng là luôn biết **mình đang đứng ở account nào và Region nào trước khi thao tác**.

Một lỗi rất thường gặp là engineer nghĩ resource bị mất, nhưng thực tế AWS Console đang được đặt sang Region khác.

### Availability Zone

`Availability Zone (AZ)` là một failure domain bên trong Region.

Bạn cần hiểu:

- một Region có nhiều AZ;
- subnet thuộc **một AZ cụ thể**;
- EC2 instance cũng được launch vào một subnet/AZ;
- thiết kế multi-AZ giúp giảm dependency vào một failure domain.

Day 4 chỉ cần foundation; kiến trúc HA/DR chuyên sâu không phải mục tiêu chính.

---

## 2. AWS Shared Responsibility Model

Đây là một concept bắt buộc trước khi làm AWS operations.

Cần phân biệt:

| Phạm vi                   | AWS chịu trách nhiệm     | Customer chịu trách nhiệm |
| ------------------------- | ------------------------ | ------------------------- |
| Physical infrastructure   | Chủ yếu                  | Không                     |
| Datacenter                | Có                       | Không                     |
| Physical network/hardware | Có                       | Không                     |
| Guest OS trên EC2         | Không                    | Có                        |
| OS patching trên EC2      | Không                    | Có                        |
| IAM permissions           | Platform cung cấp cơ chế | Customer cấu hình         |
| Security Group            | Platform cung cấp        | Customer cấu hình         |
| Application/data          | Không                    | Có                        |

Bạn phải hình thành tư duy:

> AWS cung cấp secure infrastructure, nhưng cấu hình sai IAM, Security Group, operating system hoặc application vẫn là trách nhiệm vận hành của tổ chức.

Concept này liên quan trực tiếp đến incident ownership và escalation.

---

## 3. AWS Account và Resource Governance

Trước khi nói đến VM hay network, cần hiểu AWS operations bắt đầu từ governance.

Các thành phần trọng tâm:

| Thành phần              | Ý nghĩa operational                                        |
| ----------------------- | ---------------------------------------------------------- |
| AWS account             | Security/billing/resource boundary                         |
| Account ID              | Xác minh đang thao tác đúng account                        |
| Region                  | Scope của nhiều resource                                   |
| Resource ARN            | Định danh resource                                         |
| Tags                    | Metadata phục vụ ownership, cost, automation và governance |
| Business owner          | Người/chức năng chịu trách nhiệm resource                  |
| Ticket/change reference | Liên kết thao tác với authorization                        |

Day 4 yêu cầu đặc biệt chú ý **mandatory resource tags**.

Ví dụ:

```text
Environment=Lab
Owner=infra-team
Application=training
CostCenter=LAB001
ManagedBy=InfrastructureOperations
```

Tag không chỉ để resource nhìn đẹp hơn. Trong enterprise, nó có thể phục vụ:

- inventory;
- cost allocation;
- access control;
- automation;
- lifecycle management;
- incident ownership;
- cleanup;
- audit.

---

# 4. AWS IAM Foundations

Đây là một trong ba phần lớn nhất của Day 4.

IAM trả lời:

> **Who can do what on which AWS resource, under what conditions?**

Bạn cần hiểu rõ bốn object chính:

| Object     | Vai trò                                                      |
| ---------- | ------------------------------------------------------------ |
| IAM user   | Identity gắn tương đối lâu dài với một người/workload legacy |
| IAM group  | Gom IAM users để quản lý permissions                         |
| IAM role   | Identity được assume để nhận temporary credentials           |
| IAM policy | JSON document định nghĩa permission                          |

---

## 5. IAM User

`IAM user` đại diện cho một identity bên trong AWS account.

Cần hiểu hai loại access có thể liên quan:

- AWS Console access;
- programmatic/API/CLI access.

Operationally cần biết:

- user tồn tại không đồng nghĩa user đang hoạt động;
- có credential không đồng nghĩa đang có quyền phù hợp;
- có policy không đồng nghĩa mọi request đều được Allow;
- không được dùng shared IAM users;
- long-lived access keys phải được hạn chế.

Syllabus mở rộng yêu cầu phân biệt IAM users với **roles** và **federated/Identity Center access** nếu environment sử dụng các cơ chế này. Infrastructure_Operations_14_Da…

---

# 6. IAM Groups

`IAM group` gom nhiều IAM users để gán permissions theo chức năng.

Ví dụ conceptual:

```text
LinuxAdmins
ReadOnlyAuditors
CloudOperators
BillingViewers
```

Thay vì gắn cùng một policy vào 20 users, policy có thể được gắn vào group.

Cần hiểu:

```text
IAM User
   ↓ member of
IAM Group
   ↓ attached
IAM Policy
```

Role không phải thành viên của group.

---

# 7. IAM Roles

`IAM role` đặc biệt quan trọng trong AWS operations.

Role không phải permanent login identity theo cách IAM user truyền thống hoạt động.

Workflow:

```text
Principal
   ↓
AssumeRole
   ↓
STS
   ↓
Temporary credentials
   ↓
AWS API calls
```

Bạn cần hiểu hai phần khác nhau:

### Trust relationship

Trả lời:

> Ai được phép assume role?

### Permissions policy

Trả lời:

> Sau khi assume role, role được làm gì?

Đây là điểm rất dễ nhầm.

Một role có thể có permissions rất mạnh nhưng user không được trust để assume role → user vẫn không sử dụng được role.

---

# 8. IAM Policies

IAM policy thường là JSON document.

Ví dụ conceptual:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:DescribeInstances"],
      "Resource": "*"
    }
  ]
}
```

Bạn cần hiểu các thành phần:

```text
Effect
Action
Resource
Condition
```

Và phân biệt:

```text
Allow
Explicit Deny
Implicit Deny
```

---

# 9. Permission Evaluation

Một trong những kiến thức quan trọng nhất của IAM.

Không thể suy luận:

> “Tôi thấy Allow policy → chắc chắn request sẽ thành công.”

Phải hiểu effective permission.

Conceptual model:

```text
Request
   ↓
Applicable policies
   ↓
Explicit Deny?
   ├─ Yes → DENY
   └─ No
        ↓
Explicit Allow?
   ├─ Yes → potentially ALLOW
   └─ No → IMPLICIT DENY
```

Operationally, đây là nền tảng để troubleshoot:

```text
AccessDenied
UnauthorizedOperation
not authorized to perform...
```

---

# 10. Least Privilege

`Least privilege` nghĩa là cấp **đúng minimum permissions cần thiết**.

Không phải:

```text
AdministratorAccess
```

cho tất cả users vì dễ vận hành.

Bạn cần biết suy nghĩ theo:

```text
Identity
+
Required Action
+
Required Resource
+
Required Duration
+
Required Conditions
```

Ví dụ Cloud operator chỉ cần inspect EC2:

```text
ec2:Describe*
```

không nhất thiết phải có:

```text
ec2:TerminateInstances
```

---

# 11. MFA

`Multi-Factor Authentication` bổ sung một authentication factor bên cạnh password.

Cần hiểu MFA khác với authorization.

```text
Authentication
= Bạn là ai?

Authorization
= Bạn được phép làm gì?
```

Một identity có MFA thành công nhưng vẫn có thể bị:

```text
AccessDenied
```

do policy.

Day 4 lab yêu cầu **configure approved IAM access và MFA**. Infrastructure_Operations_14_Da…

---

# 12. Temporary Credentials và AWS STS

Temporary credentials thường gồm ba thành phần:

```text
Access key ID
Secret access key
Session token
```

Chúng có expiration.

Trong enterprise, temporary credentials thường an toàn hơn long-lived static credentials vì:

- lifetime giới hạn;
- phù hợp role assumption;
- giảm credential persistence;
- dễ kiểm soát access window hơn.

Syllabus yêu cầu tuyệt đối không để:

- temporary credentials;
- passwords;
- MFA seeds;
- tokens

xuất hiện trong submitted evidence. Infrastructure_Operations_14_Da…

---

# 13. AWS CLI Profiles

AWS CLI cho phép operations từ terminal.

Concept cần hiểu:

```text
AWS CLI
   ↓
Profile / credential source
   ↓
Identity
   ↓
Region
   ↓
AWS API
```

Một profile có thể lưu/configure:

```text
region
output format
credential source
role information
```

Command quan trọng để xác định identity sẽ là:

```bash
aws sts get-caller-identity
```

Bạn sẽ học rất kỹ trong module thực hành sau.

---

# 14. Credential Safety

Day 4 không chỉ yêu cầu login được AWS; phải biết **giữ credential an toàn**.

Không được:

```text
paste AWS secret access key vào ticket
commit credentials vào Git
để credentials trong screenshot
ghi session token vào evidence
hard-code secret trong bootstrap script
```

Evidence cần chứng minh access nhưng không được trở thành credential leak.

---

# 15. IAM Evidence: State ≠ Entitlement ≠ Activity

Additive coverage của syllabus nhấn mạnh một distinction rất quan trọng.

Cần phân biệt:

### Credential state

Credential có tồn tại/enabled/expired không?

### Entitlement

Identity được cấp quyền gì?

### Activity

Identity thực tế đã làm gì?

Ba thứ này không giống nhau.

Ví dụ:

```text
Access key = Active
```

không chứng minh key đã được sử dụng.

Hay:

```text
Administrator policy attached
```

không chứng minh user đã thực hiện administrative operation.

Điểm này sẽ trở thành nền tảng cho **Day 13 — IAM review**. Infrastructure_Operations_14_Da…

---

# 16. VPC Architecture

Phần lớn thứ hai của Day 4 là network.

`Amazon VPC` là isolated virtual network trong AWS.

Conceptual topology:

```text
AWS Region
└── VPC
    ├── Public Subnet
    │   └── EC2
    │
    └── Private Subnet
        └── EC2
```

Bạn cần học các object:

```text
VPC
CIDR
Subnet
Route Table
Internet Gateway
NAT Gateway awareness
Security Group
Network ACL
```

---

# 17. CIDR

Ví dụ:

```text
10.10.0.0/16
```

VPC có thể được chia thành subnet:

```text
10.10.1.0/24
10.10.2.0/24
10.10.10.0/24
10.10.20.0/24
```

Bạn cần đủ khả năng:

- đọc CIDR;
- xác định network range;
- tránh subnet overlap;
- hiểu subnetting ảnh hưởng architecture.

CIDR foundation này sẽ tiếp tục rất nhiều ở Day 8 networking.

---

# 18. Public và Private Subnets

Một misconception cần loại bỏ:

> Public subnet không đơn giản là subnet có public IP.

Operational definition cần nhìn vào **routing**.

Một public subnet thông thường có route:

```text
0.0.0.0/0 → Internet Gateway
```

Private subnet thường không có direct route đến Internet Gateway.

Private workload nếu cần outbound Internet có thể sử dụng NAT architecture.

---

# 19. Route Tables

Route table quyết định packet phải đi next hop nào.

Ví dụ:

```text
Destination       Target
10.10.0.0/16      local
0.0.0.0/0         igw-xxxxx
```

Khi troubleshoot connectivity phải kiểm tra:

```text
Source
→ Security Group
→ subnet
→ Route Table
→ gateway/next hop
→ destination
```

Không nên troubleshoot bằng cách chỉ mở toàn bộ Security Group.

---

# 20. Internet Gateway

`Internet Gateway (IGW)` cho phép VPC có connectivity với Internet khi routing và resource addressing phù hợp.

Cần hiểu IGW chỉ là một dependency.

Có IGW không có nghĩa instance tự động Internet-accessible.

Còn phải xem:

```text
Route Table
Public IPv4
Security Group
Network ACL
Guest OS firewall
Application listening state
```

---

# 21. NAT Gateway Awareness

Day 4 yêu cầu **awareness**, chưa yêu cầu deep design.

Concept:

```text
Private EC2
   ↓
Private subnet route
   ↓
NAT Gateway
   ↓
Internet Gateway
   ↓
Internet
```

Mục tiêu thông thường:

```text
outbound Internet access
```

cho private resources mà không expose inbound Internet trực tiếp.

---

# 22. Security Groups

`Security Group` là một trong các network security controls quan trọng nhất.

Bạn cần hiểu:

- gắn với network interface/resource;
- rules kiểm soát inbound/outbound;
- stateful behavior;
- source có thể là:
  - CIDR;
  - Security Group;
  - các source types được AWS hỗ trợ;
- least privilege áp dụng cả network access.

Ví dụ:

```text
SSH TCP/22
Source: approved admin IP/CIDR
```

không nên mặc định:

```text
0.0.0.0/0
```

---

# 23. Network ACL

`Network ACL (NACL)` hoạt động ở subnet level.

Cần phân biệt với Security Group:

| Security Group                           | Network ACL           |
| ---------------------------------------- | --------------------- |
| Resource/ENI-oriented                    | Subnet-oriented       |
| Stateful                                 | Stateless             |
| Allow rules                              | Allow/Deny rules      |
| Không phải cùng operating model với NACL | Rule evaluation riêng |

Khi network lỗi, phải biết Security Group pass chưa chắc NACL pass.

---

# 24. VPC Network Flow

Một Infrastructure Operations engineer phải biết suy luận packet path.

Ví dụ SSH từ admin workstation:

```text
Admin workstation
       ↓
Internet
       ↓
Internet Gateway
       ↓
VPC Route
       ↓
Subnet
       ↓
Network ACL
       ↓
Security Group
       ↓
EC2 ENI
       ↓
Linux firewall
       ↓
sshd
```

Một failure tại bất kỳ layer nào cũng có thể xuất hiện với symptom:

```text
SSH timeout
```

Vì vậy phải troubleshoot từng layer.

---

# 25. EC2 Fundamentals

Phần lớn thứ ba của Day 4 là EC2.

`Amazon EC2` cung cấp virtual compute instances.

Bạn phải hiểu lifecycle cơ bản:

```text
Launch
↓
Pending
↓
Running
↓
Stop / Start
↓
Terminate
```

Đồng thời phải hiểu sự khác biệt operational giữa:

```text
Stop
Reboot
Terminate
```

đặc biệt vì `Terminate` có thể dẫn tới permanent deletion tùy storage/configuration.

---

# 26. EC2 Instance Families

Không cần thuộc lòng toàn bộ AWS instance catalog.

Phải hiểu families phản ánh workload characteristics, ví dụ:

```text
General purpose
Compute optimized
Memory optimized
Storage optimized
Accelerated computing
```

Infrastructure Operations cần biết kiểm tra:

```text
instance type
vCPU
memory
network characteristics
architecture
```

để troubleshoot resource/capacity mismatch.

---

# 27. AMI

`Amazon Machine Image (AMI)` là template dùng để launch EC2 instance.

AMI liên quan tới:

```text
Operating system
Root volume template
Architecture
Launch configuration metadata
```

Day 4 yêu cầu launch Linux instance từ **approved AMI**.

Không nên tùy ý chọn random public image trong enterprise environment.

---

# 28. EC2 Key Pairs

Key pair hỗ trợ secure authentication cho các use cases như SSH vào Linux EC2.

Concept:

```text
Private key
   ↓ kept securely by administrator

Public key
   ↓ available to instance
```

Private key:

```text
không upload vào ticket
không gửi qua chat
không lưu trong public share
không commit vào Git
```

---

# 29. IAM Instance Profile

EC2 workload thường không nên lưu static AWS access keys.

Thay vào đó:

```text
EC2
 ↓
Instance Profile
 ↓
IAM Role
 ↓
Temporary Credentials
 ↓
AWS API
```

Đây là pattern rất quan trọng.

Bạn cần phân biệt:

```text
Human CLI profile
```

và

```text
EC2 IAM role / instance profile
```

---

# 30. User Data

`User data` cho phép bootstrap instance trong launch process.

Ví dụ workload:

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
```

Các use case:

```text
package installation
service configuration
initial configuration
bootstrap
```

Operationally phải biết:

- bootstrap có thể fail;
- user data không phải magic guarantee;
- phải kiểm tra provisioning logs;
- không được hard-code secrets.

---

# 31. Instance Metadata

EC2 instance có thể truy cập metadata service để lấy thông tin về bản thân.

Ví dụ concept:

```text
Instance ID
Region
network information
IAM role information
temporary credentials
```

Vì metadata có thể expose security-sensitive information, syllabus yêu cầu **instance metadata và IMDSv2 awareness**. Infrastructure_Operations_14_Da…

---

# 32. IMDSv2

`IMDSv2` sử dụng session-oriented/token-based request flow thay vì kiểu request đơn giản trước đây.

Bạn cần hiểu về:

```text
Metadata options
HttpTokens
HttpEndpoint
Hop limit
```

Mục tiêu operational:

> xác minh EC2 metadata access được cấu hình phù hợp với security baseline.

Day 4 lab phải **validate instance metadata protections**. Infrastructure_Operations_14_Da…

---

# 33. SSH Access tới EC2

Để SSH thành công cần cả chuỗi dependency:

```text
Instance running
+
correct IP/DNS
+
correct routing
+
Security Group permits TCP/22
+
NACL permits traffic
+
OS firewall permits
+
sshd running
+
correct username
+
correct private key
```

Một lỗi SSH vì vậy không được xử lý bằng cách ngay lập tức mở:

```text
0.0.0.0/0:22
```

---

# 34. Linux/RHEL EC2 Bootstrap

Do Day 1–2 đã học Linux/RHEL, Day 4 tích hợp kiến thức đó vào cloud.

Bạn sẽ cần vận dụng:

```text
SSH
sudo
systemd
dnf
ip
ss
curl
journalctl
filesystem
permissions
firewall awareness
```

để kiểm tra EC2 Linux instance sau khi bootstrap.

Đây là điểm kết nối trực tiếp giữa Day 2 và Day 4.

---

# 35. EC2 Security Hardening

Ở mức Day 4, hardening tập trung vào các nguyên tắc:

```text
Least privilege IAM
Restricted network exposure
Secure SSH access
Approved AMI
No embedded credentials
IMDS protection
OS updates/baseline awareness
Logging
Resource tagging
```

Không phải mục tiêu Day 4 để xây dựng full CIS hardening program.

---

# 36. Basic EC2 Operational Troubleshooting

Bạn cần học cách chia lỗi theo layer.

Ví dụ:

```text
EC2 status/lifecycle
        ↓
Network path
        ↓
Security Group / NACL
        ↓
Guest OS
        ↓
Service
        ↓
IAM
        ↓
Application
```

Các symptom thường gặp:

```text
Instance không start
SSH timeout
Connection refused
Permission denied
AccessDenied từ AWS API
Bootstrap không chạy
Service không listen
Instance không gọi được AWS service
```

Mục tiêu Day 4 không chỉ sửa được lỗi mà còn phải xác định **failure domain**.

---

# 37. AWS Privileged Access Operations

Additive syllabus nhấn mạnh một workflow rất enterprise:

```text
Approved business requirement
        ↓
Access request / ticket
        ↓
Approval
        ↓
Correct account
        ↓
Correct identity / role
        ↓
Temporary authorized access
        ↓
Allowed/denied validation
        ↓
Evidence
        ↓
Expiry / access boundary
```

Bạn phải map được:

```text
Console access
CLI identity
Role trust
Role permissions
Business owner
Ticket/approval
```

với nhau. Infrastructure_Operations_14_Da…

---

# MODULE 2 — THỰC HÀNH

Module thực hành biến toàn bộ knowledge trên thành một controlled AWS sandbox scenario.

Syllabus yêu cầu learner **configure approved IAM access/MFA, verify CLI authentication, enforce tags, provision VPC networking, launch EC2 với AMI/key pair/IAM role/user data, establish SSH, test metadata protection, inspect network paths, test both allowed và denied connectivity, verify provisioning logs và submit architecture/IAM/EC2/network evidence**. Ngoài ra phải có sandbox role-access scenario gắn với ServiceNow-style ticket và không được để credentials/secrets xuất hiện trong evidence. Infrastructure_Operations_14_Da…

## 1. Lab Environment

Topology tổng quát sẽ có dạng:

```text
Administrative Workstation
        |
        | AWS Console / AWS CLI
        |
        v
+-----------------------------------------------+
| AWS Sandbox Account                           |
|                                               |
| IAM User / Federated Identity                 |
|       |                                       |
|       +---- AssumeRole ------------------+    |
|                                          |    |
|                                          v    |
|                                     IAM Role  |
|                                               |
| VPC 10.10.0.0/16                              |
|                                               |
| +-----------------------+                     |
| | Public Subnet         |                     |
| | 10.10.1.0/24          |                     |
| |                       |                     |
| | EC2 Linux/RHEL        |                     |
| | + IAM Instance Role   |                     |
| +-----------+-----------+                     |
|             |                                 |
|        Route Table                            |
|             |                                 |
|       Internet Gateway                        |
|                                               |
| +-----------------------+                     |
| | Private Subnet        |                     |
| | 10.10.10.0/24         |                     |
| | Concept / optional    |                     |
| +-----------------------+                     |
+-----------------------------------------------+
```

---

## 2. Phase 1 — Verify Account và Identity

Trước bất kỳ operation nào:

```text
Who am I?
Which AWS account?
Which role/user?
Which Region?
Which ticket authorizes this?
```

CLI verification sẽ xoay quanh:

```bash
aws sts get-caller-identity
```

Evidence cần giữ:

```text
Account ID
Principal/role ARN
Timestamp
Selected Region
Ticket/reference
```

nhưng không giữ secret/token.

Đây là requirement trực tiếp của additive lab: **verify active account and identity before each operation**. Infrastructure_Operations_14_Da…

---

## 3. Phase 2 — IAM và MFA

Thực hành:

```text
Approved identity
↓
MFA
↓
IAM policy/group/role inspection
↓
Least-privilege validation
```

Bạn sẽ phải hiểu evidence nào chứng minh:

```text
identity tồn tại
MFA configured
role available
permission scope
approval/reference
```

---

## 4. Phase 3 — AWS CLI Authentication

Configure approved CLI profile.

Conceptual commands:

```bash
aws configure
```

hoặc organization-specific profile/SSO/role configuration.

Sau đó verify:

```bash
aws sts get-caller-identity
```

và:

```bash
aws configure list
```

Mục tiêu là biết CLI hiện đang lấy:

```text
credentials từ đâu
Region nào
profile nào
identity nào
```

---

## 5. Phase 4 — Allowed Action Test

Ví dụ một read-only action được policy cho phép:

```bash
aws ec2 describe-instances
```

Expected:

```text
Success
```

Evidence:

```text
command
timestamp
identity
successful response subset
```

---

## 6. Phase 5 — Denied Action Test

Syllabus yêu cầu test cả **allowed action và denied action**. Infrastructure_Operations_14_Da…

Ví dụ role được thiết kế read-only.

Một unauthorized operation phải tạo:

```text
AccessDenied
```

Điều quan trọng:

> Denied result có thể là **expected successful security verification**.

Tức là command fail về mặt API nhưng security control hoạt động đúng.

---

# 7. Phase 6 — Resource Tagging

Bạn sẽ tạo/tag resource theo mandatory tagging policy.

Ví dụ:

```text
Name
Environment
Owner
Application
CostCenter
```

Sau đó verify:

```text
resource tồn tại
required tags đầy đủ
giá trị đúng
```

Failure injection có thể là thiếu:

```text
Owner
```

hoặc:

```text
Environment
```

---

# 8. Phase 7 — Design VPC CIDR

Ví dụ thiết kế:

```text
VPC
10.10.0.0/16

Public subnet
10.10.1.0/24

Private subnet
10.10.10.0/24
```

Bạn phải giải thích được:

```text
CIDR không overlap
subnet purpose
AZ placement
public/private intent
```

---

# 9. Phase 8 — Create VPC

Sau khi VPC được tạo, verification không chỉ là thấy status `Available`.

Phải kiểm tra:

```text
VPC ID
CIDR
Region
DNS settings nếu relevant
tags
```

Evidence nên record IDs thay vì chỉ screenshot.

---

# 10. Phase 9 — Create Subnets

Tạo:

```text
Public subnet
Private subnet
```

Sau đó verify:

```text
Subnet ID
CIDR
AZ
VPC association
route association
```

Một subnet không nên được gọi là public/private chỉ dựa vào tên.

---

# 11. Phase 10 — Internet Gateway

Tạo/attach approved `Internet Gateway`.

Verification:

```text
IGW exists
IGW attached to correct VPC
```

Sau đó route table mới quyết định Internet routing.

---

# 12. Phase 11 — Route Tables

Public route table dự kiến:

```text
VPC CIDR → local
0.0.0.0/0 → Internet Gateway
```

Private subnet route phải phản ánh architecture đã định.

Bạn sẽ verify:

```text
routes
subnet associations
default route
next-hop target
```

---

# 13. Phase 12 — Security Group

Security Group cho Linux EC2 có thể cho phép:

```text
TCP/22
Source = approved administrative source
```

Nếu bootstrap web service:

```text
TCP/80
```

có thể được test tùy lab.

Bạn phải biết lý do từng rule tồn tại.

Không chấp nhận:

```text
All traffic
0.0.0.0/0
```

chỉ để “lab chạy được” nếu không có justification.

---

# 14. Phase 13 — Network ACL Check

Không nhất thiết phải xây NACL phức tạp.

Nhưng phải inspect:

```text
Inbound rules
Outbound rules
Subnet association
Rule ordering
```

và hiểu NACL có thể tạo symptom network dù Security Group đúng.

---

# 15. Phase 14 — EC2 AMI Selection

Chọn **approved Linux/RHEL AMI**.

Record:

```text
AMI ID
OS/distribution
architecture
Region
selection reason
```

Không chọn image tùy ý.

---

# 16. Phase 15 — Key Pair

Tạo hoặc sử dụng approved EC2 key pair.

Private key cần có secure handling.

Ví dụ Linux workstation:

```bash
chmod 600 <private-key>
```

Không đưa private key vào evidence repository.

Evidence chỉ cần chứng minh:

```text
key pair name/reference
associated instance
secure-use verification
```

---

# 17. Phase 16 — IAM Role cho EC2

Tạo/sử dụng một approved role.

Sau đó attach qua `instance profile`.

Verification:

```text
instance
↓
IAM instance profile
↓
role
↓
policy
```

Không lưu AWS secret key bên trong EC2 để workload gọi AWS API.

---

# 18. Phase 17 — User Data Bootstrap

Ví dụ:

```bash
#!/bin/bash

dnf install -y httpd
systemctl enable --now httpd

echo "AWS Day 4 Lab" > /var/www/html/index.html
```

Mục tiêu không chỉ chạy script mà phải verify:

```text
package installed?
service running?
service enabled?
port listening?
page readable?
bootstrap log successful?
```

---

# 19. Phase 18 — Launch EC2

Launch configuration phải được kiểm tra toàn bộ:

```text
AMI
instance type
subnet
Security Group
key pair
IAM role
user data
metadata options
tags
```

Đây là dạng pre-flight checklist rất quan trọng trong production.

---

# 20. Phase 19 — EC2 Lifecycle Verification

Sau khi launch:

```text
Instance state = running
```

nhưng vẫn chưa đủ.

Phải xem thêm:

```text
status checks
IP addressing
subnet
Security Group
IAM profile
metadata settings
tags
```

---

# 21. Phase 20 — SSH Access

Từ administrative workstation:

```bash
ssh -i <approved-private-key> <user>@<instance-address>
```

Verification:

```bash
hostname
whoami
ip addr
ip route
```

Bạn cần chứng minh mình đang login đúng host, đúng user.

---

# 22. Phase 21 — Validate Network Path

Từ bên trong instance sẽ kiểm tra bằng các Linux commands đã học Day 2:

```bash
ip addr
ip route
ss -lntp
curl
```

Có thể kiểm tra:

```text
local service
outbound connectivity
routing
listening ports
```

---

# 23. Phase 22 — Allowed Connectivity Test

Ví dụ:

```text
approved admin source → TCP/22 → EC2
```

Expected:

```text
Allowed
```

Evidence cần record:

```text
source
destination
port
protocol
expected
actual
timestamp
```

---

# 24. Phase 23 — Denied Connectivity Test

Cũng phải chứng minh security control từ chối đúng traffic không được phép.

Ví dụ:

```text
Unapproved port → EC2
```

Expected:

```text
Denied
```

Đây là một phần quan trọng của security verification:

```text
Allowed traffic works
AND
Forbidden traffic fails
```

Chỉ chứng minh happy path là chưa đủ.

---

# 25. Phase 24 — Instance Metadata / IMDSv2 Validation

Bạn sẽ inspect metadata options.

Mục tiêu là xác minh instance tuân thủ metadata protection expected của lab.

Sẽ phải phân biệt:

```text
IMDS enabled?
IMDSv2 required?
token-required behavior?
```

Chi tiết command tôi sẽ trình bày ở module Thực hành đầy đủ.

---

# 26. Phase 25 — Provisioning Log Verification

Nếu user data không thành công, không được sửa mù.

Investigation sẽ kiểm tra:

```text
cloud-init
system logs
package operation
service logs
exit/failure symptoms
```

Ví dụ trên Linux cloud instance có thể cần kiểm tra:

```bash
journalctl
```

cùng các cloud-init-related logs tùy AMI.

Mục tiêu là tìm root cause.

---

# 27. Phase 26 — Linux/RHEL Verification

Sau bootstrap:

```bash
rpm -q httpd
```

hoặc package-equivalent phù hợp AMI.

```bash
systemctl status httpd
```

```bash
ss -lntp
```

```bash
curl http://localhost
```

Kết quả phải correlate:

```text
package
→ service
→ socket
→ application response
```

---

# 28. Phase 27 — Basic EC2 Failure Injection

Day 4 phù hợp với các fault simulation an toàn như:

```text
Incorrect Security Group rule
Wrong route/subnet association
Missing mandatory tag
IAM explicit/implicit denial
Failed user-data command
Wrong SSH user/key
```

Mục tiêu không phải phá lab mà tập:

```text
observe symptom
↓
collect facts
↓
isolate layer
↓
form hypothesis
↓
read-only verification
↓
authorized correction
↓
retest
```

---

# 29. Phase 28 — Troubleshooting Sequence

Một troubleshooting flow rất quan trọng sẽ là:

```text
1. Correct AWS account?
2. Correct Region?
3. Correct resource?
4. Correct instance state?
5. Correct subnet?
6. Correct route table?
7. Correct Security Group?
8. Correct NACL?
9. Correct IP/DNS?
10. Guest OS reachable?
11. Service running?
12. Port listening?
13. Authentication correct?
14. IAM permission correct?
```

Tư duy này quan trọng hơn việc nhớ riêng từng command.

---

# 30. Phase 29 — ServiceNow-style Access Ticket

Additive lab yêu cầu sandbox role-access scenario phải được gắn với một **ServiceNow-style access ticket**. Infrastructure_Operations_14_Da…

Record sẽ cần liên hệ:

```text
Requestor
Business owner
AWS account
Role
Requested scope
Approval
Start time
Expiry
Allowed result
Denied result
Evidence IDs
```

Day 11 mới học ServiceNow đầy đủ; Day 4 chỉ cần hình thành discipline về traceability.

---

# 31. Phase 30 — Evidence Collection

Một Day 4 evidence package tốt sẽ chứa ít nhất các nhóm artifact sau:

| Evidence group  | Nội dung                                    |
| --------------- | ------------------------------------------- |
| Identity        | Account, ARN/role, Region                   |
| Authorization   | Ticket/approval/access scope                |
| IAM             | Relevant user/group/role/policy/MFA state   |
| CLI             | `get-caller-identity`, profile verification |
| VPC             | VPC/subnet/route topology                   |
| Security        | Security Group/NACL configuration           |
| EC2             | Instance/AMI/type/subnet/profile            |
| Metadata        | IMDS configuration                          |
| Bootstrap       | User data result/log                        |
| Linux           | service/network verification                |
| Connectivity    | allowed test                                |
| Connectivity    | denied test                                 |
| Tags            | mandatory tags                              |
| Troubleshooting | symptom → diagnosis → correction            |
| Final           | architecture diagram/checklist              |

Các screenshots chỉ nên supplement evidence, không thay thế structured/exportable records nếu có.

---

# 32. Secrets phải loại khỏi Evidence

Day 4 explicitly yêu cầu submitted evidence **không được chứa**:

```text
AWS password
AWS secret access key
session token
temporary credential
MFA seed
private SSH key
```

Acceptance requirement của syllabus là account/role identifiers, approvals, scope, expiry và access results phải reconcile, trong khi secrets/tokens phải hoàn toàn vắng mặt khỏi logs/evidence. Infrastructure_Operations_14_Da…

---

# 33. Cleanup / Rollback

Day 4 cũng phải hình thành thói quen cleanup.

Sau lab cần review:

```text
EC2 instance
public IPv4 exposure
Security Group rules
temporary access
IAM role assignments
unused key pairs
Internet Gateway/network resources
tags
lab artifacts
```

Không được terminate/delete resource chỉ vì “lab đã xong” trước khi:

```text
collect evidence
verify dependency
confirm cleanup scope
```

---

# 34. Kết quả phải đạt sau Day 4

Sau khi hoàn thành đầy đủ hai module, bạn phải có khả năng nhìn một AWS scenario và suy luận theo chuỗi:

```text
BUSINESS AUTHORIZATION
        ↓
AWS ACCOUNT
        ↓
IDENTITY
        ↓
AUTHENTICATION
        ↓
IAM AUTHORIZATION
        ↓
VPC
        ↓
SUBNET
        ↓
ROUTE
        ↓
SG / NACL
        ↓
EC2
        ↓
GUEST OS
        ↓
SERVICE
        ↓
APPLICATION
```

Đồng thời phải hiểu một operation hoàn chỉnh không dừng ở:

```text
“I launched EC2 successfully.”
```

mà phải là:

```text
Authorized
+
Correct account/identity
+
Least privilege
+
Correct network path
+
Secure EC2 configuration
+
Successful bootstrap
+
Allowed path verified
+
Denied path verified
+
Metadata protection verified
+
Evidence collected
+
Secrets excluded
+
Cleanup/escalation understood
```

Đó chính xác là operational mindset mà Day 4 muốn xây dựng. Day 4 thuộc module AWS Days 4–5; Day 5 sẽ tiếp tục từ compute/network foundation này sang **EBS, S3 backup/DR, RPO/RTO, CloudWatch, CloudTrail, AWS Budgets và resource cleanup/cost controls**. Infrastructure_Operations_14_Da…

Khi bạn bảo **“Module Lý thuyết”**, tôi sẽ triển khai toàn bộ Module 1 theo từng topic từ foundation → operational workflow → security → verification → troubleshooting → evidence → escalation, không bỏ sót nội dung syllabus. Khi bạn bảo **“Module Thực hành”**, tôi sẽ biến Module 2 thành một lab hoàn chỉnh từng bước, gồm topology, prerequisites, safety, AWS Console/AWS CLI commands, expected output, verification, failure injection, troubleshooting, evidence và cleanup/rollback.
