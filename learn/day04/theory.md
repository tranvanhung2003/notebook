# Day 4 — Module Lý thuyết

## AWS Cloud Administration: Governance, IAM, VPC và EC2 Operations

Day 4 là phần foundation của AWS operations trong chương trình. Theo syllabus, toàn bộ Concept/Lecture của Day 4 phải bao phủ **AWS global infrastructure, Regions, Availability Zones, shared responsibility model, account/resource governance, IAM, MFA, temporary credentials, AWS CLI profiles, credential safety, tagging, least privilege, VPC/CIDR/subnets/routing/IGW/NAT/Security Group/NACL/network flow, EC2 lifecycle/instance families/AMI/key pair/instance profile/user data/metadata/IMDSv2, SSH, Linux/RHEL bootstrap, security hardening và basic EC2 troubleshooting**. Phần additive còn yêu cầu phân biệt IAM user với role/federated/Identity Center access, liên hệ Console/CLI/role với business owner và approval, đồng thời phân biệt credential state, MFA, entitlement và actual activity evidence. Infrastructure_Operations_14_Da…

Phần dưới đây chỉ là **Module Lý thuyết**. Tôi sẽ giải thích theo hướng Infrastructure Operations: không chỉ “AWS object này là gì”, mà còn phải biết **object liên kết với nhau thế nào, failure xảy ra ở layer nào, verify bằng gì, evidence nào cần giữ, đâu là phạm vi được phép xử lý và đâu là điểm phải escalate**.

---

# I. Bức tranh tổng thể của Day 4

Có thể xem toàn bộ Day 4 dưới dạng một dependency chain:

```text
Business Requirement / Ticket / Approval
                 │
                 ▼
             AWS Account
                 │
                 ▼
       Authentication / Identity
                 │
        ┌────────┴─────────┐
        │                  │
   Console access      CLI/API access
        │                  │
        └────────┬─────────┘
                 ▼
              IAM
   User / Group / Role / Policy
                 │
                 ▼
        Effective Permission
                 │
                 ▼
                VPC
                 │
     ┌───────────┴────────────┐
     │                        │
   Subnet                 Route Table
     │                        │
 SG / NACL           IGW / NAT / local
     │                        │
     └───────────┬────────────┘
                 ▼
                EC2
     AMI / Type / Key / Role
       User Data / IMDS
                 │
                 ▼
          Linux / RHEL OS
                 │
                 ▼
      Service / Application
```

Một Infrastructure Operations engineer cần nhìn được **toàn bộ chain**, vì symptom ở cuối chain thường bắt nguồn từ một layer phía trên.

Ví dụ:

```text
Symptom:
SSH tới EC2 bị timeout
```

Không thể kết luận ngay:

```text
Security Group sai
```

Có thể là:

```text
Wrong AWS Region
Instance stopped
Wrong IP
No public routing
Wrong subnet association
Security Group
NACL
OS firewall
sshd
Network interface
Corporate firewall
Wrong architecture assumption
```

Day 4 xây dựng cách suy nghĩ đó.

---

# II. AWS Global Infrastructure

## 1. AWS Region là gì?

Một `AWS Region` là một khu vực địa lý riêng biệt trong AWS global infrastructure.

Ví dụ Region code:

```text
ap-southeast-1
us-east-1
eu-west-1
```

AWS thiết kế các Region tách biệt với nhau nhằm tạo fault isolation. Phần lớn AWS resources mang tính `Regional`, tức resource bạn tạo ở Region này không tự động xuất hiện ở Region khác. [AWS Documentation](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-regions-availability-zones.html?utm_source=chatgpt.com)

### Operational significance

Trong Infrastructure Operations, Region không chỉ là một dropdown ở góc AWS Console.

Region quyết định hoặc ảnh hưởng tới:

```text
Resource visibility
Service availability
Latency
Data residency
Disaster recovery design
Networking
AMI availability
Resource IDs
Cost
Operational scope
```

Một mistake rất phổ biến:

```text
Engineer:
"EC2 instance biến mất rồi."
```

Thực tế:

```text
Console Region = us-east-1
Instance Region = ap-southeast-1
```

Resource không hề biến mất.

### Verification mindset

Trước một operation cần mentally confirm:

```text
Account?
Region?
Resource?
Identity?
```

Không nên chỉ dựa vào resource `Name` tag.

Ví dụ:

```text
prod-web01
```

có thể tồn tại:

```text
Account A / ap-southeast-1
Account B / ap-southeast-1
Account A / us-east-1
```

Do đó evidence tốt phải có:

```text
Account ID
Region
Resource ID/ARN
Timestamp
```

---

# III. Availability Zone

## 1. AZ là gì?

Mỗi Region gồm nhiều `Availability Zones (AZs)` độc lập.

Một AZ gồm một hoặc nhiều data centers có redundant power, networking và connectivity; các AZ trong một Region được kết nối bằng network có bandwidth cao và latency thấp. [AWS Documentation](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-regions-availability-zones.html?utm_source=chatgpt.com)

Ví dụ conceptual:

```text
Region: ap-southeast-1

├── Availability Zone A
├── Availability Zone B
└── Availability Zone C
```

### Region ≠ AZ

Đây là distinction bắt buộc:

```text
Region
  └── nhiều Availability Zones
```

Không phải:

```text
AZ
  └── nhiều Regions
```

---

## 2. Tại sao AZ quan trọng?

AZ là một `failure domain`.

Giả sử:

```text
Web01 → AZ-A
Web02 → AZ-A
```

Nếu AZ-A có sự cố ảnh hưởng cả hai workload, việc có hai EC2 instances chưa tạo ra multi-AZ resilience.

Thiết kế:

```text
Web01 → AZ-A
Web02 → AZ-B
```

cho phép giảm dependency vào một AZ.

### Tuy nhiên

Day 4 **không yêu cầu** xây HA architecture hoàn chỉnh.

Bạn cần hiểu foundation để sau này đọc architecture:

```text
VPC
├── Public Subnet A → AZ-A
├── Public Subnet B → AZ-B
├── Private Subnet A → AZ-A
└── Private Subnet B → AZ-B
```

---

# IV. AWS Shared Responsibility Model

Đây là concept cực kỳ quan trọng vì nó quyết định:

> Khi có security problem hoặc operational problem, ai chịu trách nhiệm cho layer nào?

AWS gọi đây là mô hình:

```text
Security OF the Cloud
vs
Security IN the Cloud
```

AWS chịu trách nhiệm bảo vệ underlying cloud infrastructure; customer chịu trách nhiệm cho workload, guest OS, access controls và nhiều configuration mà customer kiểm soát. Mức phân chia cụ thể thay đổi tùy loại AWS service. [AWS Documentation](https://docs.aws.amazon.com/whitepapers/latest/establishing-your-cloud-foundation-on-aws/security-assurance-aws.html?utm_source=chatgpt.com)

---

## 1. AWS chịu trách nhiệm về gì?

Ở mức conceptual:

```text
Physical facilities
Physical servers
Physical networking
Underlying infrastructure
Virtualization infrastructure
```

Ví dụ bạn chạy EC2.

Bạn không:

```text
replace failed physical DIMM
replace failed physical CPU
repair AWS datacenter UPS
patch AWS hypervisor yourself
```

Đó là AWS responsibility.

---

## 2. Customer chịu trách nhiệm về gì?

Với EC2, customer có mức control tương đối lớn.

Customer thường chịu trách nhiệm:

```text
Guest OS
OS patching
Application
IAM
Security Group configuration
Data
SSH configuration
Operating system users
Secrets
Application firewall
Logging/configuration
Backup strategy
```

Ví dụ:

```text
Security Group:
TCP/22
Source = 0.0.0.0/0
```

AWS đã vận hành Security Group service đúng.

Nhưng việc customer mở SSH cho toàn Internet là customer configuration responsibility.

---

## 3. Service type ảnh hưởng Shared Responsibility

Cần tránh suy nghĩ:

> AWS luôn chịu trách nhiệm chính xác cùng một lớp cho mọi service.

Không đúng.

Ví dụ EC2 cho customer nhiều control hơn một fully managed service.

Với EC2:

```text
AWS:
Physical + virtualization infrastructure

Customer:
Guest OS + application + configurations
```

Với managed services, AWS có thể quản lý thêm nhiều layer.

Đây là lý do khi troubleshoot cần biết:

```text
What service model am I operating?
What layer can I configure?
What layer belongs to AWS?
```

---

# V. AWS Account và Governance

## 1. AWS Account là security boundary quan trọng

Một `AWS account` không nên được hiểu đơn giản là “tài khoản login”.

Nó là một boundary chứa:

```text
Resources
IAM configuration
Billing
Quotas
Service settings
Audit context
Security controls
```

Trong enterprise thường có nhiều accounts:

```text
Management
Security
Logging
Shared Services
Development
Test
Production
Sandbox
```

Đây là **kiến thức bổ sung**, không phải Day 4 yêu cầu thiết kế AWS Organizations architecture.

Điểm Day 4 cần học là:

> Trước khi thao tác phải biết chính xác mình đang ở AWS account nào.

---

## 2. Account ID

AWS account có `Account ID`.

Ví dụ placeholder:

```text
123456789012
```

Bạn không nên lấy:

```text
Account alias
Console display name
resource Name tag
```

làm evidence duy nhất cho account identity.

Operational evidence mạnh hơn:

```text
Account ID
Principal ARN
Region
Resource ID
```

---

# VI. Resource Governance

`Resource governance` là cách tổ chức kiểm soát:

```text
Ai tạo resource?
Resource phục vụ việc gì?
Ai sở hữu?
Resource thuộc environment nào?
Có được phép tồn tại không?
Ai trả chi phí?
Khi nào cleanup?
Thay đổi theo ticket nào?
```

Ví dụ một EC2 không có owner:

```text
i-0123456789abcdef
```

có thể đang chạy tốt về technical perspective nhưng lại là governance problem.

Không ai biết:

```text
Ai chịu trách nhiệm?
Có được phép terminate?
Có production impact không?
Ai approve change?
```

---

# VII. Resource Tagging

## 1. Tag là gì?

AWS tag thường là:

```text
Key = Value
```

Ví dụ:

```text
Environment = Lab
Owner       = InfraOps
Application = Training
CostCenter  = LAB001
ManagedBy   = InfrastructureOperations
```

AWS mô tả tags là metadata giúp identify, organize, search và filter resources. Tag keys có tính case-sensitive; AWS cũng khuyến cáo không lưu PII hoặc confidential information trong tags. [AWS Documentation](https://docs.aws.amazon.com/tag-editor/latest/userguide/best-practices-and-strats.html?utm_source=chatgpt.com)

---

## 2. Vì sao tagging quan trọng cho Operations?

Tagging hỗ trợ:

```text
Resource ownership
Cost allocation
Automation
Inventory
Incident response
Change management
Lifecycle management
Access control
Compliance
Cleanup
```

Ví dụ engineer tìm thấy:

```text
Environment=Lab
Owner=student01
ExpirationDate=2026-10-21
```

thì cleanup workflow dễ hơn rất nhiều so với resource hoàn toàn untagged.

---

## 3. Tag consistency

Các key sau:

```text
Environment
environment
ENVIRONMENT
Env
```

không nên được tùy ý coi là cùng một standard.

AWS khuyến nghị dùng standardized, consistent, case-sensitive tagging convention. [AWS Documentation](https://docs.aws.amazon.com/tag-editor/latest/userguide/best-practices-and-strats.html?utm_source=chatgpt.com)

---

## 4. Mandatory tags

Syllabus yêu cầu hiểu `mandatory resource tags`.

Enterprise có thể yêu cầu:

```text
Owner
Environment
Application
CostCenter
DataClassification
ManagedBy
```

Resource thiếu mandatory tag có thể:

```text
violate governance policy
không được tính đúng cost
không xác định ownership
bị automation quarantine
không đạt audit requirement
```

---

## 5. Security implication của tags

Tags đôi khi được dùng trong authorization.

Ví dụ concept:

```text
Allow start/stop EC2
only if:
Environment = Development
```

Đây là dạng `ABAC` — Attribute-Based Access Control.

Nhưng nếu permission cho phép user tự sửa tag unrestricted, họ có thể thay đổi điều kiện mà policy dựa vào.

Vì vậy cần nghĩ cả hai phía:

```text
Who can act based on tag?
AND
Who can modify that tag?
```

AWS cũng cảnh báo rằng thay đổi access-control tags có thể ảnh hưởng tới policies tham chiếu chúng. [AWS Documentation](https://docs.aws.amazon.com/tag-editor/latest/userguide/best-practices-and-strats.html?utm_source=chatgpt.com)

---

# VIII. Identity and Access Management — IAM

IAM là trung tâm security của Day 4.

Câu hỏi cốt lõi:

```text
WHO
can perform
WHAT ACTION
on
WHICH RESOURCE
under
WHAT CONDITIONS?
```

Đó là authorization problem.

---

# IX. Authentication và Authorization

Hai khái niệm này phải tách biệt.

## Authentication

Trả lời:

> Bạn là ai?

Ví dụ:

```text
Password
MFA
Federation
SSO authentication
Access key
Temporary credentials
```

---

## Authorization

Trả lời:

> Identity đã authenticated đó được phép làm gì?

Ví dụ:

```text
ec2:DescribeInstances
ec2:StartInstances
ec2:StopInstances
```

Có authentication thành công không đồng nghĩa authorization thành công.

Ví dụ:

```text
Console login successful
```

nhưng:

```text
AccessDenied: ec2:TerminateInstances
```

hoàn toàn có thể là expected behavior.

---

# X. IAM Principal và IAM Identity

Trong operational reasoning bạn sẽ thường gặp:

```text
Principal
```

Có thể hiểu là entity đang thực hiện request.

Ví dụ:

```text
IAM user
IAM role session
Federated identity
AWS service principal
Root user
```

Đừng đồng nhất:

```text
User
=
Principal duy nhất
```

AWS authorization rộng hơn thế.

---

# XI. IAM User

`IAM user` là identity tồn tại trong một AWS account.

Nó có thể liên quan tới:

```text
Console credentials
Long-term access keys
Permissions
Group membership
```

---

## 1. IAM user không phải workforce identity model ưu tiên hiện nay

Điểm này rất quan trọng về current AWS best practice.

AWS hiện khuyến nghị human users sử dụng **federation** và temporary credentials thay vì tạo long-term IAM users cho mọi người. IAM users vẫn tồn tại và Day 4 vẫn phải học vì syllabus yêu cầu, nhưng không nên suy luận rằng “IAM user là phương pháp mặc định tốt nhất cho enterprise workforce access”. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

## 2. Một IAM user có thể có những trạng thái gì?

Bạn cần tách:

```text
Identity exists?
Console password exists?
MFA registered?
Access key exists?
Access key active?
Policy attached?
Group membership?
Last activity?
```

Một user:

```text
exists = true
access key active = true
```

không chứng minh user đã dùng key.

---

# XII. IAM Groups

`IAM group` là mechanism để gom IAM users.

Ví dụ:

```text
IAM Group: ReadOnlyOperators

├── alice
├── bob
└── charlie
```

Group có thể được attach permissions policies.

Conceptual flow:

```text
User
 │
 └── membership
       │
       ▼
      Group
       │
       ▼
   Permissions Policy
```

---

## IAM Group không phải một login identity

Bạn không:

```text
login as group
```

và group không hoạt động giống IAM role.

Một group là mechanism để organize IAM users và permission assignment.

---

# XIII. IAM Role

`IAM role` là concept cực kỳ quan trọng.

Một role:

```text
có permissions
nhưng không phải một long-term user identity
```

Principal được phép sẽ `assume` role để nhận temporary credentials.

Conceptual:

```text
Engineer
   │
   │ authenticated
   ▼
Source Identity
   │
   │ sts:AssumeRole
   ▼
IAM Role
   │
   ▼
Temporary Role Session
   │
   ▼
AWS API
```

---

# XIV. Trust Policy và Permissions Policy

Đây là distinction phải nhớ thật chắc.

## Trust Policy

Trả lời:

> Ai được phép assume role này?

Ví dụ concept:

```text
Role: NetworkOperatorRole

Trust:
Account / principal A được phép AssumeRole
```

---

## Permissions Policy

Trả lời:

> Sau khi assume role, session của role được phép làm gì?

Ví dụ:

```text
ec2:Describe*
ec2:StartInstances
ec2:StopInstances
```

---

## Đừng nhầm hai thứ

Role có:

```text
Powerful permissions
```

nhưng user không được trust/không được phép call `AssumeRole`

→ user không sử dụng được role đó.

Ngược lại:

```text
User allowed to assume role
```

nhưng role không có permission cần thiết

→ assume thành công, nhưng API call sau đó vẫn bị `AccessDenied`.

Một troubleshooting engineer phải hỏi hai câu riêng:

```text
1. Can I assume the role?
2. What can the assumed role actually do?
```

---

# XV. Federated Access và IAM Identity Center

Đây là phần additive syllabus yêu cầu hiểu.

Không phải mọi enterprise user đều là IAM user.

Một pattern hiện đại:

```text
Corporate Identity Provider
          │
          ▼
   IAM Identity Center
          │
    Permission Set
          │
          ▼
      AWS Account
          │
       IAM Role
          │
          ▼
Temporary credentials
```

AWS IAM Identity Center cung cấp workforce access tới AWS accounts/applications và temporary credentials cho CLI/SDK. IAM Identity Center users khác IAM users; AWS khuyến nghị temporary credentials cho workforce access. [AWS Documentation](https://docs.aws.amazon.com/singlesignon/latest/userguide/howtogetcredentials.html?utm_source=chatgpt.com)

---

## Tại sao Day 4 cần biết?

Khi một ticket nói:

```text
Give John access to AWS
```

Infrastructure Operations không thể tự động hiểu:

```text
Create IAM user John
```

Phải xác định environment sử dụng:

```text
IAM User?
Federation?
IAM Identity Center?
External IdP?
Temporary privileged access?
```

Syllabus cố ý yêu cầu bạn phân biệt các mô hình này.

---

# XVI. MFA

`MFA` = Multi-Factor Authentication.

Ví dụ authentication có thể yêu cầu:

```text
Something you know
+
Something you have / are
```

AWS khuyến nghị sử dụng MFA; với IAM users/root user khi cần sử dụng các identity đó, MFA là security control quan trọng. AWS hiện khuyến nghị phishing-resistant methods như passkeys/security keys khi phù hợp. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

## MFA không phải IAM permission

Đây là misconception phổ biến.

MFA trả lời:

```text
Authentication confidence
```

IAM policy trả lời:

```text
Authorization
```

Có MFA không có nghĩa:

```text
User = Administrator
```

Và user không có permission thì MFA cũng không tạo permission.

---

# XVII. Temporary Credentials

AWS temporary credentials thường được tạo bởi mechanisms như AWS STS/role sessions.

Thông thường chúng gồm:

```text
Access key ID
Secret access key
Session token
Expiration
```

Điểm đặc biệt là có thời hạn.

---

## Temporary credentials tốt ở đâu?

Compared với permanent long-lived credentials:

```text
Shorter lifetime
Reduced exposure window
Good fit for role assumption
Good fit for federation
Good fit for workload identity
```

AWS hiện khuyến nghị human users và AWS workloads sử dụng temporary credentials khi có thể. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

# XVIII. IAM Roles cho EC2 Workloads

Giả sử application chạy trên EC2 cần gọi S3.

Bad design:

```text
EC2
│
├── /opt/app/config
│       AWS_ACCESS_KEY_ID=...
│       AWS_SECRET_ACCESS_KEY=...
│
└── Application → S3
```

Risk:

```text
Credential theft
Long-lived credential leakage
Manual rotation
Secret copied into backup
Secret leaked into Git/log
```

Better pattern:

```text
EC2
 │
 ▼
Instance Profile
 │
 ▼
IAM Role
 │
 ▼
Temporary Credentials
 │
 ▼
AWS Service
```

AWS explicitly recommends using IAM roles and temporary credentials for EC2 workloads instead of embedding long-term access keys in applications. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html?utm_source=chatgpt.com)

---

# XIX. Instance Profile

Đây là term thường gây confusion.

Khi gắn IAM role vào EC2, EC2 sử dụng một `instance profile` như container/interface để đưa role tới instance.

Operational conceptual chain:

```text
EC2 instance
       │
       ▼
Instance Profile
       │
       ▼
IAM Role
       │
       ▼
Role credentials
```

Applications/SDKs trên instance có thể retrieve temporary role credentials thông qua EC2 metadata mechanisms. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-iam.html?utm_source=chatgpt.com)

Đừng nhầm:

```text
IAM Role
```

với:

```text
Security Group
```

Cả hai đều liên quan EC2 nhưng giải quyết hai vấn đề khác nhau:

```text
IAM Role      → AWS API authorization
Security Group → network traffic authorization
```

---

# XX. IAM Policy

IAM permissions thường được định nghĩa bằng JSON policy documents.

Ví dụ conceptual:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:DescribeInstances",
      "Resource": "*"
    }
  ]
}
```

---

# XXI. Policy Elements

Các thành phần cần hiểu:

```text
Version
Statement
Effect
Action
Resource
Condition
Principal   ← sử dụng trong những policy contexts phù hợp
```

Một cách suy nghĩ hữu ích là:

```text
Principal
Action
Resource
Condition
```

AWS gọi mô hình request context này là `PARC`. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-reqcontext.html?utm_source=chatgpt.com)

---

## Effect

```json
"Effect": "Allow"
```

hoặc:

```json
"Effect": "Deny"
```

AWS policy `Effect` xác định statement dẫn tới Allow hay explicit Deny. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_effect.html?utm_source=chatgpt.com)

---

## Action

Ví dụ:

```text
ec2:DescribeInstances
ec2:StartInstances
s3:GetObject
```

Nó biểu diễn operation được phép hoặc bị deny.

---

## Resource

Xác định resource policy áp dụng.

Có action hỗ trợ resource-level scope tốt; có action/service có behavior khác và đôi khi yêu cầu `"Resource": "*"`. Không nên tự suy đoán — phải xem service authorization documentation.

---

## Condition

Cho phép thêm constraints.

Ví dụ conceptual:

```text
Only from certain source
Only with a required tag
Only when MFA context is present
Only under approved attributes
```

`Condition` là một trong những mechanism mạnh để đạt least privilege.

---

# XXII. Permission Evaluation Logic

Đây là phần cần hiểu sâu nhất trong IAM.

Một oversimplified model hữu ích:

```text
Start
 │
 ▼
Default = DENY
 │
 ▼
Applicable Explicit Deny?
 │
 ├─ YES → DENY
 │
 └─ NO
      │
      ▼
 Applicable Allow?
      │
      ├─ YES → potentially ALLOW
      └─ NO  → IMPLICIT DENY
```

AWS mặc định implicitly denies requests; explicit Allow phải tồn tại theo applicable evaluation logic, trong khi applicable explicit Deny thắng Allow. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-denyallow.html?utm_source=chatgpt.com)

---

# XXIII. Explicit Deny vs Implicit Deny

## Implicit Deny

Không có policy cho phép action.

Ví dụ user chỉ có:

```text
ec2:DescribeInstances
```

nhưng gọi:

```text
ec2:TerminateInstances
```

Không có Allow.

Result:

```text
Implicit Deny
```

---

## Explicit Deny

Có statement:

```json
{
  "Effect": "Deny",
  "Action": "ec2:TerminateInstances",
  "Resource": "*"
}
```

Ngay cả khi policy khác Allow action, applicable explicit Deny vẫn thắng. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-denyallow.html?utm_source=chatgpt.com)

---

# XXIV. Effective Permissions phức tạp hơn một Policy

Trong enterprise, không được troubleshoot bằng:

> “Tôi thấy Administrator-like policy attached nên chắc chắn được phép.”

Effective permissions có thể chịu tác động từ:

```text
Identity-based policy
Resource-based policy
Permissions boundary
Session policy
Service Control Policy (SCP)
Resource Control Policy (RCP)
Role trust policy
Service-specific controls
```

AWS policy evaluation có union/intersection behavior khác nhau tùy policy type; permissions boundaries và Organizations policies có thể giới hạn permissions ngay cả khi identity policy có Allow. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html?utm_source=chatgpt.com)

### Đây là kiến thức bổ sung

Syllabus Day 4 không yêu cầu bạn trở thành chuyên gia AWS Organizations/SCP.

Nhưng khi gặp:

```text
AccessDenied
```

bạn phải biết:

> Attached IAM policy chưa phải toàn bộ câu chuyện.

---

# XXV. Least Privilege

`Least privilege` là nguyên tắc:

> Identity chỉ được cấp các permissions thực sự cần cho task, trên resource cần thiết, trong điều kiện cần thiết và trong thời gian cần thiết.

AWS khuyến nghị giảm permissions về minimum cần thiết và thường xuyên review/remove unused access. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

## Ví dụ xấu

Operator chỉ cần inspect EC2 nhưng được:

```text
AdministratorAccess
```

Problem:

```text
Can terminate EC2
Can alter IAM
Can modify network
Can remove logs
Can create additional access
```

blast radius quá lớn.

---

## Better conceptual design

Task:

```text
Health-check EC2
```

Permissions có thể tập trung vào:

```text
Describe
Read
List
```

thay vì:

```text
Create
Delete
Modify
Terminate
```

---

## Least privilege có nhiều dimension

Không chỉ `Action`.

Có thể giới hạn:

```text
Action
Resource
Region
Tags
Time
Source
Account
Session duration
```

Conceptual:

```text
Least privilege
=
minimal action
+
minimal resource scope
+
minimal duration
+
appropriate conditions
```

---

# XXVI. AWS Console Access vs CLI/API Access

Đây là distinction additive syllabus yêu cầu.

## AWS Management Console

Human interactive interface.

Flow:

```text
User
 ↓
Authentication
 ↓
Console session
 ↓
AWS APIs behind the UI
```

Console không phải một alternate permission universe.

Console cuối cùng vẫn dựa vào permissions.

---

## AWS CLI

CLI gửi API requests dựa trên credentials/configuration được resolve.

Conceptual:

```text
Terminal
  │
  ▼
AWS CLI
  │
  ├── credentials
  ├── Region
  ├── profile
  └── configuration
       │
       ▼
     AWS API
```

---

# XXVII. AWS CLI Profiles

AWS CLI có thể quản lý nhiều profile.

Ví dụ conceptual:

```text
default

lab-readonly

sandbox-admin

prod-readonly
```

Profile có thể chứa hoặc tham chiếu:

```text
Region
Output format
Credential source
Role configuration
SSO configuration
```

AWS CLI sử dụng `default` profile nếu không chỉ định profile khác; named profiles có thể chứa Region và authentication configuration riêng. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html?utm_source=chatgpt.com)

---

## Operational risk của profiles

Giả sử:

```text
lab profile → sandbox
prod profile → production
```

Engineer nghĩ đang dùng:

```text
lab
```

nhưng shell variable:

```text
AWS_PROFILE=prod
```

Result:

```text
valid command
valid permissions
wrong environment
```

Đây là loại incident nguy hiểm vì không phải technical failure.

Command **thành công**, nhưng success là sai.

---

# XXVIII. `get-caller-identity` mindset

Một command Day 4 bạn phải thuộc về mặt operational concept:

```bash
aws sts get-caller-identity
```

Mục tiêu không phải “học command”.

Mục tiêu là trả lời:

```text
Which account?
Which principal?
Which current session?
```

Syllabus additive yêu cầu verify active account và identity trước operation. Infrastructure_Operations_14_Da…

Thói quen nên là:

```text
VERIFY
before
CHANGE
```

---

# XXIX. Credential Safety

Credentials là sensitive security material.

Không đưa vào:

```text
Ticket
Screenshot
Chat
Git
Wiki
Evidence archive
Script
User Data
Plaintext config
Email
```

nếu chúng chứa secret material.

---

## Những loại dữ liệu cần bảo vệ

```text
Password
Secret access key
Session token
MFA seed
Private SSH key
Federation token
Temporary credential
```

Một `Access Key ID` thường là identifier chứ không phải secret theo nghĩa giống Secret Access Key, nhưng vẫn nên tránh phơi bày không cần thiết trong evidence.

---

## AWS CLI credential files

AWS CLI có thể lưu credential information trong files thuộc thư mục `.aws`; named profiles và nhiều authentication methods có precedence/configuration khác nhau. AWS docs lưu ý credentials specified qua `aws configure` có thể nằm trong local plaintext credentials file, do đó filesystem protection rất quan trọng. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html?utm_source=chatgpt.com)

---

# XXX. Credential State, MFA, Entitlement và Activity

Đây là một phần syllabus additive đặc biệt quan trọng và sẽ quay trở lại Day 13.

Bốn khái niệm này **không đồng nghĩa**.

---

## 1. Credential State

Ví dụ:

```text
Access key = Active
Console password = Enabled
```

Nó trả lời:

> Credential có ở trạng thái sử dụng được không?

---

## 2. MFA State

Ví dụ:

```text
MFA device registered
```

Nó trả lời:

> Identity có MFA mechanism configured không?

Không chứng minh mỗi activity đều sử dụng MFA.

---

## 3. Entitlement

Ví dụ:

```text
User can assume NetworkOperatorRole
```

hoặc:

```text
Role can StopInstances
```

Nó trả lời:

> Identity có quyền gì?

---

## 4. Activity Evidence

Ví dụ audit record chỉ ra:

```text
Role session performed StopInstances
```

Nó trả lời:

> Điều gì thực sự đã xảy ra?

---

## Vì sao phải tách?

Giả sử:

```text
Access Key = ACTIVE
```

Không thể kết luận:

```text
Account has been used recently.
```

Tương tự:

```text
Administrator permission exists
```

không chứng minh:

```text
User performed administrator action.
```

Đây là audit reasoning cực kỳ quan trọng.

---

# XXXI. Business Owner và Approval Mapping

Syllabus yêu cầu access phải map được về business owner.

Ví dụ:

```text
Principal:
arn:aws:sts::<account>:assumed-role/NetworkOps/...

Business reason:
Emergency troubleshooting

Business owner:
Infrastructure Manager

Approval:
CHG001234

Scope:
Read network configuration

Expiry:
22:00 UTC
```

Một permission technical hợp lệ nhưng không có business authorization vẫn có thể là governance problem.

---

# XXXII. VPC — Virtual Private Cloud

Bây giờ chuyển sang network layer.

`Amazon VPC` là virtual network boundary nơi bạn thiết kế IP addressing, subnets, routing và network controls cho resources.

Conceptual:

```text
AWS Region
└── VPC: 10.10.0.0/16
    │
    ├── Subnet A
    ├── Subnet B
    │
    ├── Route Tables
    ├── Security Groups
    ├── Network ACLs
    └── Gateways
```

---

# XXXIII. CIDR

`CIDR` = Classless Inter-Domain Routing.

Ví dụ:

```text
10.10.0.0/16
```

Prefix `/16` cho biết số bit network prefix.

Một VPC lớn:

```text
10.10.0.0/16
```

có thể được chia:

```text
10.10.1.0/24
10.10.2.0/24
10.10.10.0/24
10.10.20.0/24
```

---

## Vì sao Infrastructure Operations phải hiểu CIDR?

Vì CIDR xuất hiện trong:

```text
VPC
Subnet
Route
Security Group
NACL
VPN
Peering
Firewall rule
Corporate network
```

Nếu không hiểu subnetting, bạn sẽ không troubleshoot network path được.

---

## CIDR overlap

Ví dụ:

```text
VPC A: 10.0.0.0/16
VPC B: 10.0.0.0/16
```

Sau này muốn route giữa chúng sẽ tạo architecture complications.

Do đó IP planning là governance decision, không chỉ technical input field.

---

# XXXIV. Subnet

Một VPC được chia thành subnets.

Mỗi subnet thuộc một AZ.

Ví dụ:

```text
VPC 10.10.0.0/16

AZ-A:
  10.10.1.0/24

AZ-B:
  10.10.2.0/24
```

Subnet tạo ra một placement và routing boundary quan trọng.

---

# XXXV. Public Subnet và Private Subnet

Đây là nơi rất nhiều người học AWS hiểu sai.

## Public subnet

Theo AWS, subnet được gọi là public khi route table associated với subnet có route tới an Internet Gateway. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

Ví dụ:

```text
Destination      Target

10.10.0.0/16     local
0.0.0.0/0        igw-1234
```

---

## Private subnet

Subnet không có route trực tiếp tới Internet Gateway thường được gọi là private subnet.

Ví dụ:

```text
Destination      Target

10.10.0.0/16     local
0.0.0.0/0        nat-xxxx
```

hoặc isolated subnet:

```text
10.10.0.0/16     local
```

---

## Public subnet không có nghĩa mọi EC2 đều public

Một instance trong public subnet muốn IPv4 Internet connectivity còn cần addressing phù hợp như public IPv4/Elastic IP cùng với routing và security controls. AWS explicitly notes rằng route tới IGW và public addressing đều liên quan. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

Do đó:

```text
"Instance is in a public subnet"
```

không đồng nghĩa:

```text
"Instance is reachable from everywhere."
```

---

# XXXVI. Route Table

`Route table` quyết định traffic destination được gửi tới target nào.

AWS VPC có implicit router, và mỗi subnet phải associated với một route table; nếu không explicitly associate custom table, subnet dùng main route table. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html?utm_source=chatgpt.com)

---

## Route anatomy

Một route có:

```text
Destination
Target
```

Ví dụ:

```text
10.10.0.0/16  → local
0.0.0.0/0     → igw-xxxx
```

---

## Local route

VPC route table có local route cho communication trong VPC.

Concept:

```text
VPC = 10.10.0.0/16

Route:
10.10.0.0/16 → local
```

---

## Longest prefix match

Khi có nhiều routes phù hợp, VPC routing sử dụng route cụ thể nhất phù hợp destination (`longest prefix match`). [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/subnet-route-tables.html?utm_source=chatgpt.com)

Ví dụ:

```text
0.0.0.0/0        → IGW
10.20.0.0/16      → another target
10.20.30.0/24     → more specific target
```

Traffic tới:

```text
10.20.30.15
```

sẽ match `/24` cụ thể hơn `/16` và `/0`.

---

# XXXVII. Internet Gateway

`Internet Gateway (IGW)` là VPC component cho Internet communication.

Nó phải được:

```text
created
attached to VPC
referenced by appropriate routes
```

AWS mô tả IGW là horizontally scaled, redundant và highly available VPC component. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

---

## Có IGW chưa đủ

Đây là một checklist tư duy:

```text
IGW exists?
        │
        ▼
IGW attached?
        │
        ▼
Route table has route?
        │
        ▼
Subnet uses that table?
        │
        ▼
Instance has suitable address?
        │
        ▼
SG allows traffic?
        │
        ▼
NACL allows traffic?
        │
        ▼
OS/service accepts traffic?
```

Bỏ qua một layer cũng đủ làm connectivity fail.

---

# XXXVIII. NAT Gateway Awareness

Syllabus chỉ yêu cầu **awareness**, vì vậy Day 4 cần hiểu purpose trước.

Một common architecture:

```text
Internet
   │
   ▼
Internet Gateway
   │
   ▼
NAT Gateway
   │
   ▼
Private Subnet
   │
   ▼
EC2
```

Theo AWS, NAT device có thể cho resources trong private subnets initiate connectivity ra external networks mà không cho unsolicited inbound connections quay vào theo cách trực tiếp. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat.html?utm_source=chatgpt.com)

---

## Use case

Private EC2 cần:

```text
download package
reach external repository
call public API
```

nhưng không muốn EC2 có direct inbound exposure.

---

## Kiến thức bổ sung hiện hành

AWS documentation hiện cũng có `Regional NAT Gateway`, bên cạnh kiến trúc NAT Gateway theo AZ truyền thống. Regional mode có behavior và deployment model khác. Đây **không phải requirement chính của Day 4**; khi làm lab ta sẽ bám environment được provision và architecture được phê duyệt, không tự đổi topology. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateways-regional.html?utm_source=chatgpt.com)

Điều quan trọng cho Day 4:

```text
Understand why NAT exists
Understand route dependency
Understand that NAT ≠ inbound Internet gateway
```

---

# XXXIX. Security Group

`Security Group (SG)` là virtual firewall control associated với supported resources/network interfaces như EC2 ENI.

Rules mô tả traffic allowed vào/ra resource. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html?utm_source=chatgpt.com)

---

## Inbound rule

Ví dụ:

```text
Protocol: TCP
Port: 22
Source: 203.0.113.10/32
```

Ý nghĩa:

```text
allow SSH
from one approved source
```

---

## Outbound rule

Ví dụ:

```text
Protocol: HTTPS
Port: 443
Destination: approved range
```

---

# XL. Security Groups là Stateful

Đây là kiến thức bắt buộc.

AWS Security Groups sử dụng connection tracking và có stateful behavior. Nếu traffic khởi tạo được allow, response traffic của connection được xử lý dựa trên connection state thay vì yêu cầu một independent reverse rule giống stateless ACL logic. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html?utm_source=chatgpt.com)

Simplified:

```text
Client ──allowed request──▶ EC2
Client ◀──response──────── EC2
```

Response được connection tracking xử lý.

---

# XLI. Security Group không có explicit Deny rules kiểu NACL

Security Group tập trung vào allow rules.

Nếu traffic không match allow:

```text
not allowed
```

Không giống NACL, nơi rules có:

```text
ALLOW
DENY
```

---

# XLII. Security Group Reference

Thay vì source CIDR, trong nhiều architectures SG có thể reference another Security Group.

Concept:

```text
Web-SG
   │
   ▼
TCP/3306
   │
   ▼
DB-SG
```

Điều này hỗ trợ identity-by-resource-grouping tốt hơn hardcoded IPs trong một số architectures.

---

# XLIII. Network ACL

`Network ACL (NACL)` hoạt động ở subnet boundary.

Mỗi rule có:

```text
Rule number
Protocol
Port
Source/Destination
Allow or Deny
```

AWS đánh giá rules theo rule number tăng dần và dừng tại rule đầu tiên match. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/nacl-rules.html?utm_source=chatgpt.com)

---

# XLIV. NACL là Stateless

Khác Security Group:

```text
Security Group → stateful
NACL           → stateless
```

Với NACL, return traffic phải được allow phù hợp; NACL không ghi nhớ connection state như SG. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/nacl-examples.html?utm_source=chatgpt.com)

---

## Operational consequence

Bạn cho phép inbound:

```text
TCP/22
```

nhưng outbound ephemeral response traffic bị NACL chặn.

Symptom có thể là:

```text
SSH does not work
```

Mặc dù:

```text
Security Group looks correct.
```

Đây là một troubleshooting trap phổ biến.

---

# XLV. Security Group vs NACL

| Thuộc tính     | Security Group                 | Network ACL                     |
| -------------- | ------------------------------ | ------------------------------- |
| Scope          | Resource/ENI related           | Subnet                          |
| Behavior       | Stateful                       | Stateless                       |
| Rules          | Allow                          | Allow + Deny                    |
| Return traffic | Connection tracking            | Must be permitted appropriately |
| Typical use    | Resource-level traffic control | Subnet-level guardrail          |
| Rule priority  | Không giống ordered NACL model | Rule-number ordered             |

AWS documents xác nhận stateful SG và stateless NACL model. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html?utm_source=chatgpt.com)

---

# XLVI. VPC Network Flow

Một operator tốt phải có khả năng trace packet.

Giả sử:

```text
Laptop
  │
  │ SSH :22
  ▼
Internet
  │
  ▼
Internet Gateway
  │
  ▼
VPC Routing
  │
  ▼
Public Subnet
  │
  ▼
Network ACL
  │
  ▼
Security Group
  │
  ▼
EC2 ENI
  │
  ▼
Linux Firewall
  │
  ▼
sshd
```

---

# XLVII. Control Flow và Data Flow

Hai khái niệm nên tách.

## Control Flow

Configuration plane:

```text
IAM
Route Table
Security Group
NACL
EC2 configuration
```

## Data Flow

Packet thực sự:

```text
Source → destination
```

Một configuration nhìn hợp lý không chứng minh packet thực sự tới destination.

Do đó operational verification thường cần cả:

```text
Configuration evidence
+
Runtime behavior evidence
```

---

# XLVIII. EC2 Fundamentals

`Amazon EC2` cung cấp compute instances.

Đối với một Operations engineer, EC2 không chỉ là “VM trên AWS”.

Một EC2 instance có dependencies:

```text
AMI
Instance Type
VPC
Subnet
ENI
IP addresses
Security Groups
Storage
IAM Role
Key Pair / access mechanism
User Data
Metadata options
Guest OS
```

Khi instance fail, phải inspect dependencies thay vì chỉ nhìn state.

---

# XLIX. EC2 Lifecycle

Các states/operations quan trọng bao gồm:

```text
Launch
Pending
Running
Reboot
Stop
Start
Terminate
```

---

## Reboot

AWS mô tả reboot EC2 tương đương OS reboot ở nhiều mặt; instance vẫn trên cùng host và giữ private IPv4/public DNS cùng instance-store data theo documented behavior. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html?utm_source=chatgpt.com)

Operational use:

```text
service/OS recovery
kernel-related reboot requirement
controlled maintenance
```

Nhưng reboot không nên là bước troubleshooting đầu tiên nếu chưa thu thập evidence.

---

## Stop / Start

Đối với supported EBS-backed instances, stop/start khác reboot.

Stop/start có thể dẫn instance sang underlying host khác; private IPv4 được giữ, nhưng auto-assigned public IPv4 thường có thể thay đổi khi start lại nếu không sử dụng stable addressing mechanism phù hợp. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html?utm_source=chatgpt.com)

Do đó:

```text
Stop/start
≠
Reboot
```

---

## Terminate

`Terminate` nghĩa là loại bỏ instance.

AWS documentation lưu ý default behavior thường delete root EBS volume và preserve non-root EBS volumes, nhưng actual deletion behavior phụ thuộc block-device configuration. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html?utm_source=chatgpt.com)

Vì thế trong enterprise:

```text
terminate
```

là destructive operation.

Không phải troubleshooting shortcut.

---

# L. Stop ≠ Terminate

Đây là distinction cần nhớ:

```text
STOP
Instance vẫn tồn tại
Có thể start lại

TERMINATE
Instance bị loại bỏ
Không "start lại" chính instance đã terminated
```

Trước terminate:

```text
Approval
Dependency check
Backup/recovery consideration
Evidence
Data persistence assessment
Cleanup plan
```

---

# LI. EC2 Instance Families và Types

EC2 có nhiều instance families tối ưu cho workload khác nhau.

AWS hiện phân nhóm rộng như:

```text
General purpose
Compute optimized
Memory optimized
Storage optimized
Accelerated computing
HPC / specialized families
```

Instance types khác nhau về CPU, memory, storage và networking capacity. [AWS Documentation](https://docs.aws.amazon.com/ec2/latest/instancetypes/ec2-instance-type-specifications.html?utm_source=chatgpt.com)

---

## Infrastructure Operations cần biết gì?

Không cần học thuộc toàn bộ catalog.

Cần hiểu:

```text
Family
Generation
Size
CPU architecture
vCPU
Memory
Networking
EBS performance
Instance storage
Compatibility
```

---

## Ví dụ naming

Ví dụ conceptual:

```text
m7i.large
```

Có thể đọc như:

```text
m    → family/series
7    → generation
i    → processor/platform option
large → size
```

AWS instance type naming conventions encode family/generation/options/size. [AWS Documentation](https://docs.aws.amazon.com/ec2/latest/instancetypes/instance-type-names.html?utm_source=chatgpt.com)

---

## Operational failure mode

Application:

```text
high memory usage
OOM kills
```

Root cause có thể là:

```text
wrong instance sizing
```

không phải:

```text
network
```

Hoặc instance type không tương thích architecture của selected AMI.

---

# LII. AMI — Amazon Machine Image

AMI là image dùng để setup và boot EC2 instance.

AWS AMI chứa software cần cho instance boot và block-device mapping; AMI cũng gắn với các characteristics như Region, OS, processor architecture, root volume type và virtualization type. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html?utm_source=chatgpt.com)

---

## AMI không chỉ là ISO

AMI là launch artifact.

Concept:

```text
AMI
 │
 ├── OS image
 ├── root volume layout
 ├── architecture
 └── launch-related attributes
      │
      ▼
    EC2 Instance
```

---

# LIII. Approved AMI

Enterprise không nên chọn random public AMI chỉ vì tên trông đúng.

Cần biết:

```text
Publisher
AMI ID
Region
OS
Version
Architecture
Image provenance
Patch baseline
Hardening status
```

AWS hiện còn có `Allowed AMIs` feature để organizations control AMIs phù hợp tiêu chí, nhưng đây là **kiến thức bổ sung**, không phải requirement cốt lõi Day 4. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-allowed-amis.html?utm_source=chatgpt.com)

---

# LIV. AMI là Regional

AMI phải phù hợp Region.

Một AMI ID ở Region A không nên giả định có cùng ID/availability ở Region B.

Do đó documentation/evidence cần record:

```text
AMI ID
+
Region
```

chứ không chỉ:

```text
"RHEL image"
```

---

# LV. EC2 Key Pairs

Key pair gồm:

```text
Public Key
Private Key
```

Với Linux EC2, public key được đưa tới instance và private key dùng để chứng minh identity khi SSH. AWS không giữ bản private key để bạn download lại sau khi mất. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html?utm_source=chatgpt.com)

---

## Security model

```text
Public key
→ EC2 instance

Private key
→ administrator
```

Private key phải được bảo vệ.

Nếu attacker có private key và network path + username phù hợp:

```text
SSH compromise
```

có thể xảy ra.

---

## Private key không phải evidence

Không được gửi:

```text
.pem contents
private key screenshot
private key attachment
```

để chứng minh bạn đã dùng key.

Evidence chỉ cần:

```text
Key pair identifier
Successful authorized login evidence
Fingerprint where appropriate
Ticket/reference
```

---

# LVI. SSH Authentication Dependency

SSH success không chỉ phụ thuộc key.

Full chain:

```text
EC2 running?
      ↓
Correct address?
      ↓
Correct route?
      ↓
SG permits?
      ↓
NACL permits?
      ↓
OS firewall permits?
      ↓
sshd listening?
      ↓
Correct username?
      ↓
Correct key?
      ↓
Permissions correct?
```

---

## Symptom taxonomy

### Timeout

Thường nghiêng về:

```text
routing
SG
NACL
network path
instance unreachable
```

nhưng không kết luận tuyệt đối.

### Connection refused

Thường cho thấy:

```text
network reached host
but no listener / rejection
```

### Permission denied

Thường xem:

```text
username
key
authorized_keys
file permissions
sshd config
```

Troubleshooting bắt đầu từ symptom classification.

---

# LVII. EC2 Instance Profile và Workload Identity

Quay lại IAM-EC2 integration.

Giả sử application trên EC2 cần:

```text
Read from S3
```

Correct conceptual architecture:

```text
Application
   │
   ▼
AWS SDK / CLI
   │
   ▼
Credential Provider
   │
   ▼
EC2 Role Credentials
   │
   ▼
AWS API
```

Không nên:

```text
hard-code static access keys
```

AWS SDKs/CLI có thể automatically use credentials made available through the instance role credential chain. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/securing_access-keys.html?utm_source=chatgpt.com)

---

# LVIII. User Data

`EC2 User Data` thường được dùng cho bootstrap.

Ví dụ:

```bash
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
```

Conceptual flow:

```text
Launch EC2
    │
    ▼
Guest OS boots
    │
    ▼
cloud-init / platform mechanism
    │
    ▼
User Data execution
    │
    ▼
Package/config/service bootstrap
```

---

# LIX. User Data không phải configuration guarantee

User Data tồn tại không chứng minh:

```text
Bootstrap succeeded
```

Script có thể fail vì:

```text
Repository unavailable
DNS failure
Route failure
Package missing
Permission issue
Syntax error
Wrong OS command
Dependency failure
Service start failure
```

Do đó Operations phải verify **result**, không chỉ verify `User Data` field tồn tại.

---

# LX. Không để secrets trong User Data

Rất quan trọng.

Không nên:

```bash
AWS_SECRET_ACCESS_KEY=...
DB_PASSWORD=...
PRIVATE_TOKEN=...
```

trong User Data.

Metadata mechanisms có thể expose user-data content từ bên trong instance tùy configuration/use case; AWS documents an IMDS path for retrieving user data. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html?utm_source=chatgpt.com)

Nguyên tắc:

```text
Bootstrap configuration
≠
Secret store
```

---

# LXI. EC2 Instance Metadata Service — IMDS

EC2 cung cấp Instance Metadata Service để instance truy cập metadata liên quan chính nó.

Common information có thể liên quan:

```text
Instance identity
Network information
IAM role credentials
User Data
```

IPv4 endpoint historically:

```text
169.254.169.254
```

AWS cũng hỗ trợ IPv6 metadata endpoint trong điều kiện phù hợp. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html?utm_source=chatgpt.com)

---

# LXII. Vì sao metadata security-sensitive?

Nếu instance có IAM role:

```text
EC2
 │
 ▼
IMDS
 │
 ▼
Temporary role credentials
```

Do đó một process/application có access tới metadata có thể potentially retrieve role credentials subject to the relevant controls.

Đây là lý do metadata protection rất quan trọng.

---

# LXIII. IMDSv1 và IMDSv2

Day 4 yêu cầu `IMDSv2 awareness`.

### IMDSv1

Simpler request model.

### IMDSv2

Token/session-oriented model.

Simplified:

```text
1. PUT request
      │
      ▼
Obtain token
      │
2. Use token
      │
      ▼
Metadata GET
```

AWS IMDSv2 token có TTL tối đa 21,600 seconds (6 hours), là instance-specific, và token phải được cung cấp cho metadata request. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html?utm_source=chatgpt.com)

---

# LXIV. `HttpTokens`

Important metadata option:

```text
HttpTokens = optional
```

nghĩa tokenless/IMDSv1-style access có thể vẫn được support theo configuration.

```text
HttpTokens = required
```

nghĩa require IMDSv2 token.

AWS documents that `httpTokens=required` represents IMDSv2-only token-required operation. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-options.html?utm_source=chatgpt.com)

---

# LXV. Metadata Hop Limit

`HttpPutResponseHopLimit` kiểm soát IP hop limit cho IMDSv2 token response.

Containerized environments có thể cần careful consideration.

Không nên:

```text
increase blindly
```

chỉ để “application chạy”.

Thay đổi metadata settings phải dựa vào:

```text
architecture
application requirement
security baseline
approved design
```

---

# LXVI. IMDS Verification Mindset

Khi review instance:

```text
Metadata endpoint enabled?
HttpTokens required?
Hop limit expected?
IAM role attached?
Application dependent on IMDS?
```

Điểm quan trọng:

> Security configuration phải vừa secure vừa compatible với workload.

Nếu bắt buộc IMDSv2 khiến legacy application fail, response không phải:

```text
disable protection immediately
```

mà phải:

```text
collect evidence
identify compatibility issue
review baseline
escalate/change application where required
```

---

# LXVII. Linux/RHEL EC2 Bootstrap

Day 4 cố ý nối kiến thức AWS với Day 1–2.

Bạn đã học Linux/RHEL:

```text
systemd
dnf
permissions
networking
SSH
journalctl
firewalld
SELinux
filesystem
```

Khi Linux chạy trong EC2, những kiến thức này vẫn áp dụng.

Cloud không làm guest OS biến mất.

---

# LXVIII. Layered Security

Một EC2 Linux có thể có nhiều layer:

```text
AWS Network Layer
    Security Group
    NACL
    Routing

Guest OS Layer
    firewalld
    sshd
    SELinux
    users/groups
    sudo

Application Layer
    service config
    TLS
    authentication
```

Một service fail có thể xảy ra ở bất kỳ layer nào.

---

# LXIX. EC2 Security Hardening

Day 4 không phải full CIS benchmark course.

Syllabus muốn foundation operational hardening.

Core principles:

```text
Approved AMI
Least privilege IAM
Temporary credentials
Restricted Security Groups
Controlled SSH
Protected private keys
IMDSv2 protection
No embedded secrets
Patch awareness
Limited exposed services
Resource tagging
Logging/evidence awareness
```

---

# LXX. Secure SSH

Bad practice:

```text
22/tcp
0.0.0.0/0
```

trong mọi environment.

Better:

```text
restricted approved source
```

hoặc alternative access mechanisms theo enterprise design.

AWS cũng hỗ trợ mechanisms như Systems Manager Session Manager, nhưng **Session Manager không nằm trong core Day 4 syllabus**, nên ở đây chỉ cần biết nó tồn tại như một alternative operational method; không thay đổi mục tiêu SSH lab. AWS EC2 documentation cũng nêu Session Manager như một alternative to key-pair connection methods. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html?utm_source=chatgpt.com)

---

# LXXI. Basic EC2 Troubleshooting Framework

Một operator không troubleshoot random.

Hãy chia theo layers.

```text
Layer 1: Scope
Account
Region
Resource

Layer 2: EC2 Control Plane
State
Status
Configuration

Layer 3: VPC
Subnet
Route
IGW/NAT

Layer 4: Security Controls
Security Group
NACL

Layer 5: Guest Network
Interface
IP
Route
DNS
Firewall

Layer 6: OS
CPU
Memory
Disk
systemd

Layer 7: Service
Listener
Config
Logs

Layer 8: Identity
SSH identity
IAM role
Permissions

Layer 9: Application
Application behavior
Dependencies
```

---

# LXXII. Troubleshooting phải đi từ ít rủi ro tới nhiều rủi ro

Bad troubleshooting:

```text
SSH fail
→ open all ports
→ reboot
→ replace route
→ terminate/rebuild
```

Better:

```text
Observe
   ↓
Verify scope
   ↓
Collect read-only evidence
   ↓
Identify failing layer
   ↓
Form hypothesis
   ↓
Test safely
   ↓
Approved correction
   ↓
Verify
   ↓
Collect post-change evidence
```

---

# LXXIII. Scenario: SSH Timeout

Symptom:

```text
ssh: connect to host ... port 22: Connection timed out
```

Potential categories:

```text
Wrong IP
Wrong Region/resource
Instance not Running
No IGW route
No public addressing
SG blocks 22
NACL blocks traffic
Corporate firewall
Network path
Guest firewall
```

Do **not** start with:

```text
regenerate key
```

vì key authentication thường xảy ra sau khi TCP connectivity tồn tại.

---

# LXXIV. Scenario: SSH Permission Denied

Symptom:

```text
Permission denied (publickey)
```

Network path đã tiến xa hơn timeout case.

Focus shift tới:

```text
Wrong username
Wrong private key
Public key mismatch
authorized_keys
File permissions
SSH daemon configuration
```

Đây là example của **symptom-driven troubleshooting**.

---

# LXXV. Scenario: EC2 Không Ra Internet

Potential path:

```text
EC2
 │
 ▼
Guest route
 │
 ▼
Subnet route
 │
 ▼
NAT or IGW
 │
 ▼
Security controls
 │
 ▼
DNS / target
```

Public subnet case và private subnet case khác nhau.

Public instance có thể cần:

```text
Public addressing
+
IGW route
```

Private architecture có thể cần:

```text
NAT
+
correct route
```

AWS NAT troubleshooting guidance cũng nhấn mạnh kiểm tra NAT status, route tables, SG/NACL và public-subnet/IGW dependencies. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html?utm_source=chatgpt.com)

---

# LXXVI. Scenario: `AccessDenied`

Đừng troubleshoot IAM bằng cách attach AdministratorAccess ngay.

Flow:

```text
Which principal?
      ↓
Which account?
      ↓
Which action?
      ↓
Which resource?
      ↓
Which policies?
      ↓
Any explicit Deny?
      ↓
Boundary/session/SCP?
      ↓
Correct role assumed?
      ↓
Trust policy?
```

Mục tiêu là **find authorization root cause**, không bypass policy.

---

# LXXVII. Scenario: User Data Không Chạy Đúng

Possible causes:

```text
Wrong shebang
Wrong package manager
DNS
Internet path
Repository
Permissions
Script error
Service error
OS mismatch
User Data execution behavior
```

Verification phải đi từ:

```text
Was bootstrap mechanism invoked?
        ↓
What did logs say?
        ↓
Which line failed?
        ↓
Did package install?
        ↓
Did service start?
        ↓
Is port listening?
```

Không chỉ:

```text
"I don't see the website."
```

---

# LXXVIII. Verification Philosophy

Day 4 yêu cầu mindset:

> Configuration existence ≠ functional verification.

Ví dụ:

```text
Security Group rule exists
```

không chứng minh:

```text
SSH works
```

Tương tự:

```text
IAM policy attached
```

không chứng minh:

```text
effective permission allows request
```

Hay:

```text
User Data exists
```

không chứng minh:

```text
bootstrap succeeded
```

---

# LXXIX. Positive và Negative Verification

Một security control phải verify cả hai mặt.

Ví dụ IAM:

```text
Expected allowed action
→ succeeds

Expected denied action
→ fails
```

Network:

```text
TCP/22 from approved source
→ allowed

Unapproved traffic
→ denied
```

Điểm rất quan trọng:

```text
A failed unauthorized request
```

có thể chính là:

```text
PASS
```

của security control.

---

# LXXX. Evidence trong AWS Operations

Evidence cần chứng minh:

```text
Who
What
Where
When
Under which authorization
Expected result
Actual result
```

---

## Identity evidence

Ví dụ:

```text
AWS Account ID
Principal ARN
Role name
Session identity
Region
Timestamp
```

---

## IAM evidence

```text
Relevant role
Relevant policy
MFA state where authorized
Scope
Approval
Expiry
Allowed test
Denied test
```

---

## Network evidence

```text
VPC ID
Subnet ID
CIDR
AZ
Route Table ID
Relevant routes
Security Group ID/rules
NACL association/rules
```

---

## EC2 evidence

```text
Instance ID
AMI ID
Instance Type
Subnet
IAM profile
Security Groups
Metadata options
Tags
Instance state
```

---

# LXXXI. Evidence phải tránh secrets

Syllabus explicitly yêu cầu submitted evidence không chứa:

```text
Temporary credentials
Passwords
MFA seeds
Tokens
```

và về operational practice cũng không được đưa private key/secret key vào evidence. Infrastructure_Operations_14_Da…

Điểm acceptance không phải:

> “Tôi screenshot mọi thứ.”

Mà là:

> “Tôi giữ đủ bằng chứng để reviewer tái dựng operation nhưng không làm lộ credential.”

---

# LXXXII. Screenshot không phải Evidence duy nhất

Một screenshot có thể giúp:

```text
visual context
configuration state
UI-specific evidence
```

nhưng thường thiếu:

```text
structured fields
complete output
machine-readable details
exact context
repeatability
```

Vì vậy evidence tốt có thể gồm:

```text
CLI output
structured export
screenshot
ticket
timestamp
architecture diagram
```

tùy use case.

---

# LXXXIII. Read-only Verification vs Configuration Change

Day 4 phải hình thành discipline rất rõ.

### Read-only

Ví dụ concept:

```text
Describe
Get
List
Inspect
```

Mục tiêu:

```text
understand current state
```

### Configuration-changing

Ví dụ:

```text
Create
Modify
Attach
Detach
Start
Stop
Terminate
Change route
Change policy
```

Mục tiêu:

```text
change current state
```

Trong incident:

```text
Read first
Change later
```

trừ emergency runbook được approval rõ ràng.

---

# LXXXIV. Common Mistakes tổng hợp

| Mistake                                              | Vì sao nguy hiểm                                  |
| ---------------------------------------------------- | ------------------------------------------------- |
| Không verify Account                                 | Có thể change nhầm environment                    |
| Không verify Region                                  | Có thể chẩn đoán nhầm hoặc tạo duplicate resource |
| Dùng root user daily                                 | Excessive privilege                               |
| Dùng IAM users + long-lived keys không cần thiết     | Credential exposure                               |
| Attach AdministratorAccess để sửa AccessDenied       | Bypass least privilege                            |
| Không hiểu role trust vs role permission             | Troubleshoot sai layer                            |
| Hard-code secret trong EC2                           | Credential leak                                   |
| Mở `0.0.0.0/0` để “test”                             | Excessive network exposure                        |
| Gọi subnet là public chỉ vì tên                      | Sai network reasoning                             |
| Chỉ check SG mà bỏ NACL/route                        | Miss root cause                                   |
| Treat NAT như inbound gateway                        | Sai architecture                                  |
| Dùng random AMI                                      | Supply-chain/configuration risk                   |
| Gửi private key trong evidence                       | Critical credential exposure                      |
| Chỉ thấy User Data rồi kết luận bootstrap thành công | Không verify runtime                              |
| Reboot trước khi collect logs                        | Mất volatile troubleshooting context              |
| Terminate để “rebuild cho nhanh”                     | Destructive change                                |
| Credential active → kết luận user active             | Audit reasoning sai                               |
| Policy attached → kết luận action occurred           | Entitlement ≠ activity                            |
| Screenshot token/secret                              | Data leak                                         |

---

# LXXXV. L1/L2 Escalation Boundary

Day 4 muốn bạn bắt đầu suy nghĩ như Infrastructure Operations chứ không phải unrestricted cloud administrator.

## L1 có thể thường làm trong authorized runbook

Tùy organization:

```text
Read-only health checks
Verify instance state
Verify tags
Inspect route/SG
Check approved IAM identity
Gather logs/evidence
Run known connectivity tests
Follow pre-approved restart procedure
Create/escalate incident
```

---

## L2 có thể thường xử lý sâu hơn

Tùy delegated permission:

```text
Authorized SG correction
Approved route correction
Role/policy troubleshooting
EC2 operational changes
Bootstrap diagnosis
OS/service remediation
Controlled IAM/access investigation
```

Nhưng vẫn phải theo:

```text
ticket
approval
change scope
runbook
```

---

## Escalate khi nào?

Ví dụ:

```text
Production impact outside runbook
IAM/SCP/organization-level restrictions
Unclear business owner
Privileged-access conflict
Suspected credential compromise
Cross-account issue
Network architecture change
Large CIDR/routing impact
Security control must be weakened
Unknown dependency
Need destructive terminate/rebuild
Possible AWS service/platform issue
Evidence indicates security incident
```

---

# LXXXVI. Khi nào Escalate tới Security/IAM Team?

Nếu thấy:

```text
Unexpected privileged role
Unknown access key
Credential exposure
MFA issue
Privilege escalation
Unapproved administrator entitlement
Suspicious role trust
Unexpected cross-account access
```

Không tự ý:

```text
delete identity
rotate unrelated credentials
change SCP
detach policies globally
```

nếu không được authorized.

---

# LXXXVII. Khi nào Escalate tới Network Team?

Nếu vấn đề vượt VPC scope được giao, ví dụ:

```text
Transit Gateway
Direct Connect
Site-to-Site VPN
Corporate routing
Central inspection firewall
Hybrid DNS
Shared VPC architecture
```

Day 4 chỉ tập trung VPC foundation.

---

# LXXXVIII. Khi nào Escalate tới AWS Support?

Không phải mọi vấn đề EC2 đều là AWS issue.

Trước khi escalate nên có:

```text
Account
Region
Resource ID
Timestamp/timezone
Observed symptom
Expected behavior
Relevant logs
Network/IAM checks
Reproduction result
Changes already attempted
Business impact
```

Điều này giúp tránh ticket:

```text
"EC2 doesn't work, please check."
```

---

# LXXXIX. Mental Model quan trọng nhất của IAM

Bạn nên thuộc flow này:

```text
Who am I?
   │
   ▼
How was I authenticated?
   │
   ▼
Which account?
   │
   ▼
Which role/session?
   │
   ▼
What action?
   │
   ▼
Which resource?
   │
   ▼
What policy types apply?
   │
   ▼
Any Deny?
   │
   ▼
Expected Allow?
   │
   ▼
Actual API result
```

---

# XC. Mental Model quan trọng nhất của VPC

```text
Source
 │
 ▼
Source security
 │
 ▼
Routing
 │
 ▼
Gateway / target
 │
 ▼
Subnet boundary
 │
 ▼
NACL
 │
 ▼
Security Group
 │
 ▼
ENI
 │
 ▼
Guest OS
 │
 ▼
Listening service
```

Nếu bạn không biết packet đang ở layer nào, đừng change configuration ngẫu nhiên.

---

# XCI. Mental Model quan trọng nhất của EC2

```text
Control Plane
Instance state/configuration
        │
        ▼
Network Plane
VPC/Subnet/Route/SG/NACL
        │
        ▼
Guest OS
Linux/RHEL
        │
        ▼
Service
systemd/process/socket
        │
        ▼
Application
actual business service
```

Một green EC2 state chỉ chứng minh:

```text
EC2 control plane sees instance running
```

không chứng minh:

```text
application healthy
```

---

# XCII. Mental Model về Identity và Network

Một distinction rất quan trọng:

### IAM answers

```text
Can principal call AWS API?
```

Ví dụ:

```text
Can I call:
ec2:StopInstances?
```

### Security Group/NACL answer

```text
Can network traffic flow?
```

Ví dụ:

```text
Can TCP/22 reach EC2?
```

Không được nhầm:

```text
IAM permission
```

với:

```text
network permission
```

---

# XCIII. Scenario tổng hợp 1 — Console xem được EC2 nhưng CLI không xem được

Possible reasoning:

```text
Console identity?
CLI identity?
CLI profile?
Account?
Region?
Role?
Credentials expired?
Effective permission?
```

Không kết luận AWS CLI lỗi.

Console và CLI có thể đang sử dụng **hai principals khác nhau**.

---

# XCIV. Scenario tổng hợp 2 — CLI Describe được nhưng Stop không được

Observation:

```text
DescribeInstances → success
StopInstances     → AccessDenied
```

Likely conclusion:

```text
Authentication works.
CLI reaches AWS.
Account/credentials are at least valid.
Authorization differs by Action.
```

Đây là useful narrowing.

---

# XCV. Scenario tổng hợp 3 — EC2 có Public IP nhưng SSH Timeout

Không được suy luận:

```text
Public IP → Internet reachable
```

Cần kiểm tra:

```text
Subnet route → IGW?
SG?
NACL?
Corporate source path?
Instance state?
OS firewall?
```

---

# XCVI. Scenario tổng hợp 4 — EC2 Private Subnet Không `dnf update`

Possible dependencies:

```text
DNS?
Route?
NAT?
NAT route?
NACL?
Outbound SG?
Repository?
Proxy?
```

Không nên move instance sang public subnet chỉ để “fix”.

---

# XCVII. Scenario tổng hợp 5 — EC2 Role Có Permission Nhưng Application AccessDenied

Check:

```text
Correct instance profile?
Correct role?
Application using role credentials?
Old environment credentials overriding chain?
Resource policy?
Boundary/SCP?
Region/resource?
```

Một application có environment variables chứa old credentials có thể không dùng identity bạn nghĩ nó đang dùng.

Đây là example về:

```text
effective runtime identity
≠
assumed intended identity
```

---

# XCVIII. Scenario tổng hợp 6 — Bootstrap Service Không Chạy

Architecture:

```text
User Data
→ dnf install
→ systemctl enable
→ systemctl start
```

Nếu site fail, investigate sequentially:

```text
Did User Data execute?
Did DNS/network work?
Did package install?
Does unit exist?
Did service start?
Is port listening?
Does local curl work?
Does external route work?
```

Đây là reasoning từ inside-out.

---

# XCIX. Scenario tổng hợp 7 — Expected Deny

Suppose role:

```text
EC2ReadOnlyOperator
```

Allowed:

```text
DescribeInstances
```

Denied:

```text
TerminateInstances
```

Nếu test result:

```text
Describe → Success
Terminate → AccessDenied
```

security outcome:

```text
PASS
```

Không phải:

```text
Partial failure
```

Điều này rất quan trọng khi viết lab report.

---

# C. Scenario tổng hợp 8 — IAM Role Assume Fail

Phải tách:

```text
Source identity permission
```

và:

```text
Target role trust
```

Nếu trust không cho principal:

```text
AssumeRole fails
```

dù role bên trong có `AdministratorAccess`.

---

# CI. Scenario tổng hợp 9 — Credentials Hết Hạn

Temporary credentials có expiration.

Symptom có thể là authentication/session errors.

Correct solution không phải:

```text
paste secret mới vào script permanently
```

mà phải xác định intended authentication workflow:

```text
SSO refresh
role session refresh
credential provider
instance role
```

---

# CII. Scenario tổng hợp 10 — Audit hỏi “User này có hoạt động không?”

Không trả lời bằng:

```text
Access key = Active
```

Vì đó chỉ là credential state.

Cần tách:

```text
Credential state
Entitlement
Last-used evidence
Audit activity
```

Đây là bridge rất quan trọng từ Day 4 sang Day 13 IAM review.

---

# CIII. Kiến thức bổ sung — Root User

Syllabus không liệt kê root user thành topic riêng, nhưng phải biết boundary.

AWS account `root user` có special full account authority và AWS khuyến cáo không dùng root cho daily tasks; thay vào đó sử dụng appropriately permissioned identities. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_identity-management.html?utm_source=chatgpt.com)

Infrastructure operator không nên nghĩ:

```text
AccessDenied?
→ use root.
```

Đó là anti-pattern.

---

# CIV. Kiến thức bổ sung — Permission Boundary và SCP

Không cần master Day 4.

Nhưng phải biết chúng tồn tại.

Concept:

```text
IAM identity policy:
"Allow X"

Permissions boundary:
does not permit X

Effective:
DENIED
```

Tương tự AWS Organizations `SCP` có thể giới hạn actions đối với accounts thuộc organization. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html?utm_source=chatgpt.com)

Operational takeaway:

```text
AccessDenied
≠
always missing user policy
```

---

# CV. Kiến thức bổ sung — Temporary Elevated Access

Enterprise có thể áp dụng:

```text
Request
→ Approval
→ temporary privileged role
→ session
→ expiry
```

thay vì permanent admin rights.

AWS IAM Identity Center documentation cũng mô tả temporary elevated/just-in-time access như mô hình request, approve và track sensitive permissions trong khoảng thời gian xác định. [AWS Documentation](https://docs.aws.amazon.com/singlesignon/latest/userguide/temporary-elevated-access.html?utm_source=chatgpt.com)

Điều này khớp trực tiếp operational mindset của syllabus.

---

# CVI. Day 4 Theory — Security Decision Tree

Khi chuẩn bị operation:

```text
1. Có ticket/approval không?
             │
             ▼
2. Đúng AWS account không?
             │
             ▼
3. Đúng Region không?
             │
             ▼
4. Đúng identity/role không?
             │
             ▼
5. Permission có đúng scope không?
             │
             ▼
6. Resource có đúng owner/tags không?
             │
             ▼
7. Operation read-only hay change?
             │
             ▼
8. Change có reversible không?
             │
             ▼
9. Evidence plan có chưa?
             │
             ▼
10. Rollback/escalation boundary?
```

Nếu không trả lời được một checkpoint quan trọng, không nên tiếp tục blindly.

---

# CVII. Day 4 Theory — Connectivity Decision Tree

Khi một EC2 connection fail:

```text
Correct account/Region/resource?
        │
        ▼
Instance Running?
        │
        ▼
Correct destination IP/DNS?
        │
        ▼
Correct subnet?
        │
        ▼
Correct route?
        │
        ▼
IGW/NAT as architecture requires?
        │
        ▼
NACL permits?
        │
        ▼
Security Group permits?
        │
        ▼
Guest firewall permits?
        │
        ▼
Service listening?
        │
        ▼
Authentication correct?
```

Điều này nên trở thành reflex.

---

# CVIII. Day 4 Theory — IAM Decision Tree

Khi API call fail:

```text
Who is caller?
      │
      ▼
Credentials valid?
      │
      ▼
Correct account?
      │
      ▼
Correct role/session?
      │
      ▼
Correct Action?
      │
      ▼
Correct Resource?
      │
      ▼
Explicit Allow?
      │
      ▼
Explicit Deny?
      │
      ▼
Boundary/session/SCP?
      │
      ▼
Resource policy/trust?
```

Không escalate IAM ticket chỉ với:

```text
"AccessDenied"
```

Cần có:

```text
Principal
Action
Resource
Time
Expected permission
Actual error
```

và **không có secret**.

---

# CIX. Những thứ phải nhớ tuyệt đối trước khi sang Lab

| Topic                 | Điều phải nhớ                                                                              |
| --------------------- | ------------------------------------------------------------------------------------------ |
| Region                | Geographic operational boundary                                                            |
| AZ                    | Failure domain trong Region                                                                |
| Shared Responsibility | AWS bảo vệ cloud infrastructure; customer bảo vệ workload/configuration thuộc phạm vi mình |
| Account               | Security/resource/governance boundary quan trọng                                           |
| Tag                   | Metadata cho ownership, cost, automation, governance                                       |
| IAM User              | Long-lived IAM identity; không phải workforce model duy nhất                               |
| IAM Group             | Gom IAM users để manage permissions                                                        |
| IAM Role              | Assumable identity sử dụng temporary credentials                                           |
| Trust Policy          | Ai được assume role                                                                        |
| Permissions Policy    | Role/user được làm gì                                                                      |
| MFA                   | Authentication factor, không phải permission                                               |
| Temporary Credentials | Credentials có expiration                                                                  |
| Least Privilege       | Chỉ cấp minimum required access                                                            |
| CLI Profile           | Có thể làm bạn operate bằng identity/account/Region khác                                   |
| IAM Evaluation        | Default deny; explicit Deny thắng Allow                                                    |
| VPC                   | Virtual network boundary                                                                   |
| Subnet                | AZ-scoped network segment                                                                  |
| Public Subnet         | Route table có path tới IGW                                                                |
| Private Subnet        | Không có direct IGW route                                                                  |
| Route Table           | Destination → target                                                                       |
| IGW                   | Internet connectivity component                                                            |
| NAT                   | Outbound connectivity pattern cho private resources                                        |
| SG                    | Stateful resource-level traffic control                                                    |
| NACL                  | Stateless subnet-level allow/deny control                                                  |
| AMI                   | EC2 launch image, Region/architecture sensitive                                            |
| Instance Type         | Compute/memory/network/storage characteristics                                             |
| Key Pair              | Public/private SSH authentication material                                                 |
| Instance Profile      | EC2 mechanism liên kết role tới instance                                                   |
| User Data             | Bootstrap input; success phải được verified                                                |
| IMDS                  | Instance-local metadata service                                                            |
| IMDSv2                | Token-based metadata access model                                                          |
| SSH                   | Requires complete network + OS + identity chain                                            |
| Hardening             | Least privilege + reduced exposure + secure credential handling                            |
| Troubleshooting       | Diagnose layer-by-layer, read-only first                                                   |
| Evidence              | Reproducible facts, no secrets                                                             |

---

# CX. Concept Map cuối Module Lý thuyết Day 4

Toàn bộ kiến thức hôm nay có thể nén lại thành architecture map này:

```text
                         BUSINESS / GOVERNANCE
              ┌──────────────────────────────────┐
              │ Ticket / Approval / Owner / Tags │
              └─────────────────┬────────────────┘
                                │
                                ▼
                         AWS ACCOUNT
                                │
               ┌────────────────┴───────────────┐
               │                                │
               ▼                                ▼
        AUTHENTICATION                    AUTHORIZATION
      MFA / Federation                   IAM Policies
      IAM Identity Center                Least Privilege
      CLI Credentials                    Effective Access
               │                                │
               └────────────────┬───────────────┘
                                ▼
                         IAM PRINCIPAL
                User / Role / Role Session
                                │
                                ▼
                              VPC
     ┌──────────────────────────┼───────────────────────────┐
     │                          │                           │
     ▼                          ▼                           ▼
   CIDR                     Subnets                    Routing
                         Public / Private          RT / IGW / NAT
     │                          │                           │
     └──────────────────────────┼───────────────────────────┘
                                ▼
                      NETWORK SECURITY
                   Security Group + NACL
                                │
                                ▼
                              EC2
           ┌────────────────────┼─────────────────────┐
           │                    │                     │
           ▼                    ▼                     ▼
          AMI              Instance Type           Key Pair
           │                    │                     │
           └────────────────────┼─────────────────────┘
                                ▼
                       Instance Profile
                                │
                             IAM Role
                                │
                                ▼
                    Temporary Credentials
                                │
                                ▼
                    User Data / Bootstrap
                                │
                                ▼
                     Instance Metadata
                           IMDSv2
                                │
                                ▼
                         LINUX / RHEL
       SSH → Users → sudo → firewalld → systemd → service
                                │
                                ▼
                         APPLICATION
                                │
                                ▼
                         VERIFICATION
                                │
               ┌────────────────┼───────────────┐
               ▼                ▼               ▼
           Functional        Security         Evidence
             test          allow + deny      no secrets
```

---

# Kết luận Module Lý thuyết Day 4

Sau phần lý thuyết này, mục tiêu không phải chỉ nhớ định nghĩa `IAM`, `VPC` hay `EC2`. Bạn phải bắt đầu tư duy theo ba chuỗi dependency chính.

**Chuỗi Identity:**

```text
Business approval
→ Authentication
→ Principal
→ Role
→ Policy evaluation
→ Effective permission
→ AWS API result
```

**Chuỗi Network:**

```text
Source
→ Route
→ Gateway
→ Subnet
→ NACL
→ Security Group
→ ENI
→ Guest OS
→ Service
```

**Chuỗi EC2:**

```text
AMI
→ Instance Type
→ VPC/Subnet
→ IAM Role
→ Key/Access
→ User Data
→ IMDS
→ Linux/RHEL
→ Service
→ Verification
```

Và ở góc nhìn Infrastructure Operations, mọi operation hoàn chỉnh phải có thêm một vòng bao quanh ba chuỗi trên:

```text
Authorization
      ↓
Pre-check
      ↓
Operation
      ↓
Verification
      ↓
Negative Verification
      ↓
Evidence
      ↓
Rollback / Cleanup
      ↓
Escalation if unresolved
```

Đây cũng chính là lý do Day 4 không dạy AWS như một bài “click Console để tạo EC2”. Syllabus yêu cầu bạn hình thành **operational readiness**: biết mình đang thao tác bằng identity nào, trong account/Region nào, permission đến từ đâu, packet phải đi qua layer nào, EC2 phụ thuộc những component nào, vì sao một control Allow/Deny, làm sao xác nhận result, và làm sao chứng minh operation bằng evidence mà không làm lộ credentials. Infrastructure_Operations_14_Da…

Phần **Module Thực hành Day 4** tiếp theo sẽ chuyển toàn bộ theory này thành một controlled AWS sandbox lab: từ **IAM/MFA/CLI identity verification → role allowed/denied test → mandatory tags → VPC/CIDR/subnets → route/IGW/NAT awareness → SG/NACL → EC2/AMI/key pair/instance profile → User Data → IMDSv2 → SSH/Linux verification → troubleshooting → evidence → cleanup/rollback**.
