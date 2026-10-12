# Day 7 — AWS Cloud Fundamentals: Global Infrastructure, Shared Responsibility, IAM, MFA, AWS CLI Profile & Tagging

Theo **MASTER_SYLLABUS**, Day 7 mở đầu phần AWS Cloud Fundamentals. Nội dung bắt buộc hôm nay là **AWS global infrastructure, shared responsibility, IAM users/roles/policies, MFA, CLI profile và tagging**. Assignment/Lab bắt buộc là **configure approved IAM access, MFA evidence, CLI access và mandatory resource tags**. 00_MASTER_SYLLABUS_Java_Fresher…

Đây là một Day rất quan trọng vì từ Day 8 trở đi bạn bắt đầu tạo VPC, EC2, EBS, CloudWatch... Nếu Day 7 chưa chắc IAM, credentials, Region và tagging, những ngày sau rất dễ rơi vào hai tình huống nguy hiểm: **không có quyền khi cần** hoặc **có quá nhiều quyền và vô tình thay đổi/xóa resource**.

---

## 1. Syllabus Alignment

### Mandatory — bắt buộc theo syllabus

| Chủ đề                      | Bạn phải nắm được                                                     |
| --------------------------- | --------------------------------------------------------------------- |
| AWS Global Infrastructure   | Region, Availability Zone, phạm vi resource                           |
| Shared Responsibility Model | AWS chịu trách nhiệm gì, customer chịu trách nhiệm gì                 |
| IAM User                    | Human/service identity dạng dài hạn và hạn chế của nó                 |
| IAM Role                    | Temporary credentials, assume role                                    |
| IAM Policy                  | Allow/Deny, Action, Resource, Condition, least privilege              |
| MFA                         | Mục đích, các loại MFA, cách kiểm chứng                               |
| AWS CLI Profile             | Credential/config profile, Region, identity validation                |
| Tagging                     | Key/value, naming convention, ownership, environment, cost/operations |
| Lab                         | Approved IAM access + MFA evidence + CLI + mandatory tags             |

### Supplementary / Beyond the explicit syllabus

Trong bài mình sẽ bổ sung thêm những phần cần thiết cho vận hành thực tế: `root user`, IAM group, AWS STS, temporary credentials, Identity Center/SSO, IAM policy evaluation, ARN, credential troubleshooting, permission boundary/SCP awareness, tagging governance và evidence/rollback.

AWS hiện khuyến nghị **human users sử dụng federation và temporary credentials**, và workloads sử dụng **IAM roles** thay vì phân phối long-lived IAM user credentials. IAM users vẫn phải học vì syllabus yêu cầu và chúng vẫn tồn tại trong AWS, nhưng bạn không nên hiểu rằng “IAM user + access key” là kiến trúc mặc định tốt nhất cho production. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

# 2. Objectives — Học xong Day 7 bạn phải làm được gì?

Cuối buổi, bạn phải có khả năng nhìn một AWS environment và trả lời được:

“Resource này nằm ở Region nào?”, “AZ có phải Region không?”, “AWS hay chúng ta chịu trách nhiệm patch OS?”, “Identity hiện tại của CLI là ai?”, “Tại sao command bị AccessDenied?”, “IAM user khác IAM role ở đâu?”, “Policy này thực sự cho phép gì?”, “MFA bảo vệ loại credential nào?”, “Profile CLI nào đang được dùng?”, “Resource thuộc môi trường nào, owner nào?”, “Tôi có bằng chứng chứng minh configuration đã đúng chưa?”

Đặc biệt, mục tiêu không phải chỉ là “login được AWS Console”. Fresher Administration/Operations cần tư duy:

**Identity → Authentication → Authorization → API request → AWS service → resource → evidence.**

---

# 3. Mental Model quan trọng nhất của Day 7

Hãy hình dung mỗi thao tác AWS như sau:

```text
Human / Application
        |
        | Authentication
        v
AWS Identity
(User / Role / Federated Identity)
        |
        | Credentials
        v
AWS API Request
        |
        | Authorization evaluation
        v
IAM Policies
        |
        +---- Explicit Deny? ---> DENIED
        |
        +---- Allow? ----------> continue
        |
        +---- No Allow? -------> DENIED
        |
        v
AWS Service
        |
        v
Resource
(EC2 / S3 / RDS / ...)
```

Một command rất đơn giản như:

```bash
aws ec2 describe-instances --profile java-fresher-lab
```

thực tế cần rất nhiều thứ đúng đồng thời:

```text
AWS CLI installed
        ↓
Profile exists
        ↓
Credentials can be obtained
        ↓
Credentials are valid/not expired
        ↓
Correct AWS account
        ↓
IAM authorization allows ec2:DescribeInstances
        ↓
Correct Region
        ↓
AWS EC2 API endpoint reachable
        ↓
Response returned
```

Đây chính là cách bạn phải troubleshoot sau này.

---

# 4. AWS Global Infrastructure

## 4.1 Region là gì?

**AWS Region** là một khu vực địa lý độc lập mà AWS vận hành infrastructure.

Ví dụ quen thuộc:

```text
ap-southeast-1    Singapore
ap-northeast-1    Tokyo
us-east-1         N. Virginia
eu-west-1         Ireland
```

Region code rất quan trọng vì CLI, SDK và resource đều thường liên quan đến Region.

AWS hiện mô tả Regions là các geographical areas độc lập. Nhiều resource là **regional resource**: resource được tạo trong một Region thì khi chuyển Console sang Region khác bạn có thể không nhìn thấy nó. AWS không tự động replicate mọi regional resource sang Region khác cho bạn. [AWS Documentation](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-regions-availability-zones.html?utm_source=chatgpt.com)

Trang AWS Global Infrastructure hiện tại liệt kê **39 launched Regions và 123 Availability Zones**. Đây là số liệu động và có thể tiếp tục thay đổi; không nên học thuộc số lượng để làm nền tảng tư duy. [Amazon Web Services, Inc.](https://aws.amazon.com/about-aws/global-infrastructure/)

---

## 4.2 Availability Zone — AZ

Một Region chứa nhiều **Availability Zones**.

Ví dụ:

```text
Region: ap-southeast-1

├── ap-southeast-1a
├── ap-southeast-1b
└── ap-southeast-1c
```

AZ không đơn giản là “một server room”.

AWS mô tả mỗi AZ là một location độc lập trong Region, gồm một hoặc nhiều datacenter riêng biệt với nguồn điện, networking và connectivity dự phòng. Các AZ trong cùng Region kết nối với nhau bằng network có bandwidth cao và latency thấp. [AWS Documentation](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-regions-availability-zones.html?utm_source=chatgpt.com)

AWS hiện cho biết mỗi Region có ít nhất ba Availability Zones. [AWS Documentation](https://docs.aws.amazon.com/global-infrastructure/latest/regions/aws-availability-zones.html?utm_source=chatgpt.com)

Điểm thi rất hay hỏi:

```text
Region != Availability Zone
```

Region:

```text
ap-southeast-1
```

AZ:

```text
ap-southeast-1a
```

---

## 4.3 Tại sao cần nhiều AZ?

Giả sử application chỉ nằm trong:

```text
ap-southeast-1a
```

Nếu AZ này gặp disruption, application có thể down.

Nếu kiến trúc:

```text
             Load Balancer
                  |
        +---------+---------+
        |                   |
       AZ-A                AZ-B
        |                   |
      App 1               App 2
```

thì failure của một AZ không nhất thiết khiến toàn bộ application unavailable.

Đây là foundation cho Day 8, Day 9 và kiến trúc application sau này.

---

## 4.4 Region selection

Khi chọn Region, operator thường phải quan tâm tới latency, compliance/data residency, service availability, cost và disaster-recovery architecture.

Ví dụ, không nên đơn giản nghĩ:

> “Singapore gần Việt Nam nên lúc nào cũng chọn Singapore.”

Production architecture còn phụ thuộc business requirement, regulatory requirement, network path, service availability, budget và DR strategy.

---

## 4.5 Regional, Zonal và Global

Đây là một distinction rất quan trọng.

**Regional resource** tồn tại theo Region.

Ví dụ nhiều EC2/VPC-related resources.

**Zonal resource** liên kết với một AZ cụ thể.

Ví dụ EC2 instance được launch vào một subnet, và subnet thuộc một AZ.

**Global service/resource** có phạm vi rộng hơn Region trong một số chức năng.

IAM thường được xem là global service ở cấp account.

Điểm operations cực kỳ quan trọng là:

> Không thấy EC2 instance chưa chắc instance đã mất; trước tiên hãy kiểm tra **account + Region**.

---

## 4.6 Edge infrastructure

AWS còn có infrastructure phục vụ workloads gần end user như Edge Locations/CloudFront POP, Local Zones, Wavelength Zones và các mô hình hybrid/edge khác.

Đây là **Supplementary / Beyond explicit Day 7 syllabus**.

Bạn hiện chưa cần cấu hình chúng. Chỉ cần phân biệt:

```text
Region
  └── Availability Zones

Edge location
  ≠ Availability Zone
```

AWS hiện còn cung cấp Local Zones và Wavelength Zones để đưa workload hoặc dịch vụ gần người dùng hơn trong những use case latency-sensitive. [Amazon Web Services, Inc.](https://aws.amazon.com/about-aws/global-infrastructure/)

---

# 5. AWS Shared Responsibility Model

Một câu phải thuộc:

> **AWS chịu trách nhiệm security OF the cloud.  
> Customer chịu trách nhiệm security IN the cloud.**

AWS chính thức mô tả AWS chịu trách nhiệm bảo vệ infrastructure chạy AWS Cloud gồm hardware, software, networking và facilities. Customer responsibility phụ thuộc service được lựa chọn. Với EC2, customer phải quản lý guest OS, patches, application software và security-group configuration. [Amazon Web Services, Inc.](https://aws.amazon.com/compliance/shared-responsibility-model/)

---

## 5.1 Ví dụ với EC2

Giả sử bạn chạy:

```text
Java Application
Tomcat
Java 17
Amazon Linux
EC2
AWS physical server
AWS data center
```

Phân chia gần đúng:

| Layer                         | Responsibility |
| ----------------------------- | -------------- |
| Java application config       | Customer       |
| Java patch/config             | Customer       |
| Tomcat                        | Customer       |
| Linux guest OS                | Customer       |
| Linux security updates        | Customer       |
| EC2 Security Group            | Customer       |
| Data                          | Customer       |
| IAM permissions               | Customer       |
| Hypervisor infrastructure     | AWS            |
| Physical server               | AWS            |
| Physical network              | AWS            |
| Data center building/security | AWS            |

AWS explicitly nêu khách hàng chạy EC2 chịu trách nhiệm guest OS, security patches, customer-installed application và security-group firewall configuration. [Amazon Web Services, Inc.](https://aws.amazon.com/compliance/shared-responsibility-model/)

---

## 5.2 Managed service thay đổi responsibility

Shared Responsibility không phải bảng cố định áp dụng giống nhau cho mọi AWS service.

Ví dụ với EC2:

```text
Customer manages OS
```

Nhưng với abstracted services như S3 hoặc DynamoDB, AWS quản lý underlying infrastructure, OS và platform; customer vẫn quản lý data, access controls và security configuration liên quan. [Amazon Web Services, Inc.](https://aws.amazon.com/compliance/shared-responsibility-model/)

Vì vậy câu:

> “AWS lo security nên khách hàng không phải patch gì.”

là **sai**.

Câu:

> “Khách hàng phải tự quản lý toàn bộ hardware AWS.”

cũng **sai**.

---

# 6. IAM — Identity and Access Management

IAM giải quyết hai câu hỏi lớn:

```text
Bạn là ai?
Bạn được phép làm gì?
```

Tên chuyên môn:

```text
Authentication
Authorization
```

Authentication:

```text
"Who are you?"
```

Authorization:

```text
"What are you allowed to do?"
```

---

# 7. Root User

Khi AWS account được tạo, account có **root user**.

Root user có quyền cực lớn trên account.

Tư duy đúng:

```text
Root user
   |
   +-- protect strongly
   +-- MFA
   +-- don't use for daily administration
   +-- use only for tasks that genuinely require root
```

Không nên:

```text
root credentials
       ↓
developer laptop
       ↓
daily AWS CLI
```

AWS khuyến nghị giảm tối đa việc sử dụng root credentials và yêu cầu bảo vệ root bằng MFA. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/security-creds.html?utm_source=chatgpt.com)

---

# 8. IAM User

IAM User là identity trong AWS account.

Ví dụ:

```text
alice
bob
java-fresher-lab
deploy-user
```

IAM user có thể có:

```text
Console credentials
Programmatic credentials
Policies
Groups
Tags
MFA device
```

Nhưng đây là điều cần hiểu thật kỹ:

```text
IAM User ≠ employee account automatically
```

IAM user là AWS identity object.

Trong môi trường hiện đại, AWS khuyến nghị workforce users sử dụng federation/Identity Center và temporary credentials thay vì tạo một IAM user dài hạn cho mỗi người. [docs.aws.amazon.com](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

# 9. IAM Group

IAM Group không ghi trực tiếp trong dòng syllabus nhưng là prerequisite rất quan trọng.

Ví dụ:

```text
IAM Group: Developers
     |
     +--- Alice
     +--- Bob
     +--- Carol
```

Policy:

```text
Developers
   |
   +--- ReadOnlyPolicy
```

thay vì attach cùng policy thủ công cho từng user.

Lưu ý:

```text
IAM Group contains users.
IAM Group does NOT contain IAM roles.
```

---

# 10. IAM Role

IAM Role là một identity có permissions nhưng không hoạt động giống một permanent IAM user.

Mental model:

```text
IAM User
  "I am this identity."

IAM Role
  "I temporarily assume this identity."
```

Ví dụ:

```text
EC2
 |
 | Assume Role
 v
AppServerRole
 |
 +--- s3:GetObject
```

Ứng dụng Java trên EC2 cần đọc S3.

Cách tệ:

```text
Access key
Secret key
    ↓
hard-code trong application.properties
```

Cách đúng:

```text
EC2
 ↓
IAM Role
 ↓
Temporary credentials
 ↓
S3
```

AWS hiện khuyến nghị workloads sử dụng IAM roles và temporary credentials. [docs.aws.amazon.com](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

# 11. Hai phần quan trọng của Role

IAM role thường cần hiểu hai khía cạnh.

### Trust policy

Trả lời:

```text
WHO may assume this role?
```

Ví dụ về mặt ý tưởng:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Service": "ec2.amazonaws.com"
  },
  "Action": "sts:AssumeRole"
}
```

Nghĩa là service principal EC2 có thể assume role.

### Permissions policy

Trả lời:

```text
Once assumed, WHAT can this role do?
```

Ví dụ:

```text
s3:GetObject
```

Đây là lỗi fresher rất thường gặp:

> Trust policy đúng nhưng role không có quyền resource.

hoặc:

> Permissions policy có quyền nhưng principal không được phép assume role.

---

# 12. IAM Policy

IAM policy thường là JSON document mô tả authorization rules.

Một ví dụ:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::example-bucket"
    }
  ]
}
```

Các thành phần quan trọng:

```text
Version
Statement
Effect
Action
Resource
Condition
```

AWS IAM documentation định nghĩa identity-based policies kiểm soát identity có thể thực hiện action nào, trên resource nào và trong conditions nào. [AWS Documentation](https://docs.aws.amazon.com/us_en/IAM/latest/UserGuide/access_policies.html?utm_source=chatgpt.com)

---

# 13. Effect

Có hai giá trị:

```text
Allow
Deny
```

Ví dụ:

```json
"Effect": "Allow"
```

hoặc:

```json
"Effect": "Deny"
```

AWS mặc định deny access; `Allow` tạo permission, còn explicit `Deny` có thể override một `Allow`. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_effect.html?utm_source=chatgpt.com)

---

# 14. Action

Action mô tả API operation.

Ví dụ:

```text
s3:GetObject
s3:ListBucket
ec2:DescribeInstances
ec2:StartInstances
ec2:StopInstances
iam:GetUser
```

Bạn nên đọc:

```text
service:Operation
```

Ví dụ:

```text
ec2:DescribeInstances
```

nghĩa là:

```text
Service = EC2
Operation = DescribeInstances
```

---

# 15. Resource và ARN

Policy có thể giới hạn resource.

AWS thường dùng **ARN — Amazon Resource Name**.

Ví dụ dạng:

```text
arn:aws:s3:::example-bucket
```

ARN format thay đổi tùy service.

IAM documentation xác định `Resource` là object hoặc objects mà policy statement áp dụng, thường thông qua ARN. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_resource.html?utm_source=chatgpt.com)

---

# 16. Condition

`Condition` giúp giới hạn permission thêm.

Ví dụ ý tưởng:

```text
Allow action
ONLY IF
request satisfies condition
```

Có thể dựa trên:

```text
IP
MFA
resource tag
principal tag
transport security
Region
...
```

AWS policy engine so sánh condition key/value với request context. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition.html?trk=article-ssr-frontend-pulse_little-text-block&utm_source=chatgpt.com)

---

# 17. IAM Policy Evaluation — phần rất quan trọng khi troubleshoot

Mental model đơn giản cho fresher:

```text
Start
 |
 v
Implicit Deny
 |
 | Is there an applicable Allow?
 +---- No ----> DENY
 |
 Yes
 |
 | Is there an applicable Explicit Deny?
 +---- Yes ---> DENY
 |
 No
 |
 v
ALLOW
```

Bạn phải nhớ:

```text
Explicit Deny beats Allow.
```

Ví dụ:

Policy A:

```text
Allow ec2:*
```

Policy B:

```text
Deny ec2:TerminateInstances
```

Kết quả:

```text
DescribeInstances       ALLOW
StartInstances          ALLOW
StopInstances           ALLOW
TerminateInstances      DENY
```

AWS policy evaluation documentation xác nhận identity/resource policies có thể cấp Allow, nhưng explicit Deny override Allow. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html?utm_source=chatgpt.com)

---

# 18. Least Privilege

**Least privilege**:

> Identity chỉ có các permissions cần thiết để thực hiện nhiệm vụ, không hơn.

Ví dụ fresher cần xem EC2:

Không nên:

```text
AdministratorAccess
```

nếu chỉ cần:

```text
ec2:Describe*
```

AdminAccess rất tiện trong lab nhưng tạo mental model nguy hiểm.

Trong doanh nghiệp, một operator phải hỏi:

```text
Task là gì?
Required APIs là gì?
Required resources là gì?
Required environment là gì?
Có cần write không?
Có cần destructive actions không?
```

---

# 19. Managed Policy và Inline Policy

AWS chia identity-based policies thành managed và inline.

**AWS managed policy** do AWS tạo/quản lý.

**Customer managed policy** do organization/customer tạo.

**Inline policy** gắn trực tiếp một-to-one vào một identity.

AWS docs lưu ý customer managed policies thường cho khả năng kiểm soát chi tiết hơn AWS managed policies, còn inline policies bị ràng buộc trực tiếp với một user/group/role. [AWS Documentation](https://docs.aws.amazon.com/us_en/IAM/latest/UserGuide/access_policies.html?utm_source=chatgpt.com)

Fresher không nên kết luận:

```text
AWS managed policy = always least privilege
```

Không đúng.

Nó có thể rộng hơn task thực tế của bạn.

---

# 20. MFA — Multi-Factor Authentication

Authentication factor có thể đến từ nhiều loại yếu tố.

Password:

```text
Something you know
```

Security key:

```text
Something you have
```

MFA tăng security vì attacker cần nhiều hơn password.

AWS hiện hỗ trợ MFA bằng passkeys/security keys, virtual authenticator apps và hardware TOTP tokens, đồng thời khuyến nghị phishing-resistant MFA như passkeys/security keys khi có thể. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html?utm_source=chatgpt.com)

---

# 21. MFA không phải authorization

Đây là điểm thi quan trọng.

MFA chứng minh authentication mạnh hơn.

MFA không tự động cho bạn:

```text
AdministratorAccess
```

Nếu user không có:

```text
ec2:StartInstances
```

thì nhập MFA cũng không tự sinh permission đó.

Tuy nhiên policy có thể có `Condition` yêu cầu session phải đáp ứng MFA requirement.

---

# 22. MFA Evidence

Syllabus yêu cầu:

> MFA evidence.

Evidence không nên chứa:

```text
password
TOTP seed
QR-code enrollment secret
access key secret
recovery secret
```

Evidence tốt có thể là screenshot Console cho thấy:

```text
MFA device assigned
Username
Account/environment context
```

nhưng phải che/redact thông tin nhạy cảm nếu policy yêu cầu.

Một administrator phải hiểu:

```text
Evidence proves configuration.
Evidence must not create another security incident.
```

---

# 23. AWS CLI

AWS CLI cho phép tương tác AWS APIs từ command line.

Ví dụ:

```bash
aws sts get-caller-identity
```

```bash
aws ec2 describe-regions
```

```bash
aws iam get-user
```

AWS CLI v2 nên được dùng cho lab mới. Đáng chú ý, AWS CLI v1 đã vào maintenance mode từ **August 5, 2026** và AWS hiện khuyến nghị migrate sang CLI v2. [AWS Documentation](https://docs.aws.amazon.com/cli/v1/reference/sts/get-caller-identity.html?utm_source=chatgpt.com)

---

# 24. Kiểm tra AWS CLI

```bash
aws --version
```

Ví dụ output:

```text
aws-cli/2.x.x Python/... Linux/...
```

Đây là **Example output**.

Expected invariant:

```text
aws command exists
CLI major version = 2
```

Nếu:

```text
aws: command not found
```

thì bạn chưa ở bước IAM troubleshooting.

Đó là local workstation/tooling issue.

---

# 25. AWS CLI Profile

Profile cho phép một workstation giữ nhiều AWS configurations.

Ví dụ:

```text
java-fresher-lab
development
production-readonly
```

AWS CLI documentation nói default profile tên `default`; named profiles có thể có credentials, Region và output configuration riêng. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

Ví dụ:

```bash
aws sts get-caller-identity --profile java-fresher-lab
```

---

# 26. Hai file quan trọng của AWS CLI

Trên Linux/macOS:

```text
~/.aws/config
~/.aws/credentials
```

AWS CLI stores configuration và credential information dưới thư mục `.aws` trong home directory. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

Ví dụ `~/.aws/config`:

```ini
[profile java-fresher-lab]
region = ap-southeast-1
output = json
```

Nếu dùng long-lived credentials trong môi trường lab được phê duyệt, `~/.aws/credentials` có thể có:

```ini
[java-fresher-lab]
aws_access_key_id = ...
aws_secret_access_key = ...
```

**Không bao giờ gửi nội dung thực tế của file này cho mình hoặc paste credential vào chat.**

AWS hiện cảnh báo không nên dùng IAM user long-term credentials cho purpose-built software hoặc real workloads; federation/temporary credentials được ưu tiên. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

---

# 27. `default` profile vs named profile

Không nên vô thức dùng:

```bash
aws ec2 ...
```

nếu máy có nhiều account.

Tốt hơn trong lab:

```bash
aws sts get-caller-identity \
  --profile java-fresher-lab
```

Sau đó mới:

```bash
aws ...
```

Điều này giảm nguy cơ:

```text
intended: LAB
actual: PROD
```

Đây là một trong những lỗi cloud operations nguy hiểm nhất.

---

# 28. `aws sts get-caller-identity`

Đây gần như là command đầu tiên bạn nên chạy sau khi cấu hình credentials.

```bash
aws sts get-caller-identity \
  --profile java-fresher-lab
```

Ví dụ:

```json
{
  "UserId": "AIDAXXXXXXXXX",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/java-fresher"
}
```

Nó cho bạn biết:

```text
Who am I?
Which account?
Which ARN?
```

AWS STS documentation xác nhận `GetCallerIdentity` trả về IAM user hoặc role đang được sử dụng để gọi AWS. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/sts/get-caller-identity.html?utm_source=chatgpt.com)

Trong assumed-role session, ARN có thể giống:

```text
arn:aws:sts::123456789012:assumed-role/OperationsRole/alice
```

Đây là evidence rất giá trị.

---

# 29. AWS Profile inspection

Xem profiles:

```bash
aws configure list-profiles
```

AWS CLI hỗ trợ command này để liệt kê profiles hiện có. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/configure/list-profiles.html?utm_source=chatgpt.com)

Xem effective configuration:

```bash
aws configure list \
  --profile java-fresher-lab
```

Bạn có thể thấy:

```text
profile
access_key
secret_key
region
```

CLI sẽ mask secret values trong output.

AWS docs cũng cho biết `aws configure list` cho biết configuration value đến từ đâu, rất hữu ích khi troubleshooting precedence. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

---

# 30. Profile precedence problem

Giả sử profile nói:

```text
region = ap-southeast-1
```

nhưng shell có:

```bash
export AWS_REGION=us-east-1
```

Bạn có thể thấy behavior khác dự kiến.

Vì vậy command:

```bash
aws configure list --profile java-fresher-lab
```

rất quan trọng.

Không chỉ hỏi:

> “File config ghi gì?”

Mà phải hỏi:

> “CLI effective configuration hiện tại là gì?”

---

# 31. AWS Identity Center / SSO — phương pháp preferred cho human access

**Supplementary / current AWS best practice.**

Nếu organization cung cấp IAM Identity Center:

```bash
aws configure sso
```

Sau đó:

```bash
aws sso login --profile java-fresher-lab
```

và:

```bash
aws sts get-caller-identity \
  --profile java-fresher-lab
```

Với SSO, AWS CLI config chứa SSO session/configuration; file `credentials` không nhất thiết chứa permanent IAM-user keys. AWS CLI sẽ sử dụng temporary credentials. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

Nếu công ty bạn dùng SSO thì **đừng tự tạo access key vì tutorial trên Internet nói `aws configure` cần Access Key ID**.

---

# 32. Tagging

AWS tag thường là:

```text
Key = Value
```

Ví dụ:

```text
Environment = lab
Project     = java-fresher
Owner       = operations-training
ManagedBy   = manual
```

Tags giúp:

```text
identify
organize
search
cost allocation
automation
access control
operations
```

AWS nhấn mạnh tagging nên có format chuẩn, nhất quán, hỗ trợ cost tracking, automation và access control. [AWS Documentation](https://docs.aws.amazon.com/tag-editor/latest/userguide/best-practices-and-strats.html?utm_source=chatgpt.com)

---

# 33. Tag naming

Tags **case-sensitive**.

Ví dụ:

```text
Environment
environment
ENVIRONMENT
```

có thể được coi là khác nhau.

Vì vậy company phải chọn convention.

Ví dụ lab hôm nay:

```text
Project       = java-fresher
Environment   = lab
Owner         = <team-or-approved-id>
Purpose       = training
```

Không cần tranh luận camelCase hay lowercase-hyphen cái nào “đúng tuyệt đối”.

Quan trọng là:

```text
documented + standardized + consistently enforced
```

AWS cũng khuyến nghị một capitalization strategy nhất quán. [AWS Documentation](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_tag-policies-best-practices.html?utm_source=chatgpt.com)

---

# 34. Tuyệt đối không chứa sensitive data trong tags

Không đặt:

```text
Password=...
AccessKey=...
CustomerSSN=...
DatabaseSecret=...
```

AWS explicitly cảnh báo tags không dùng để lưu confidential/sensitive information; tags có thể xuất hiện qua nhiều APIs và billing/management workflows. [AWS Documentation](https://docs.aws.amazon.com/tag-editor/latest/userguide/best-practices-and-strats.html?utm_source=chatgpt.com)

---

# 35. Tag limits awareness

AWS general tagging guidance hiện nêu đa số resource có tối đa **50 user-created tags**, AWS-generated tags có prefix `aws:` và không được user tự sửa/xóa như normal user-defined tags. Service-specific behavior vẫn phải kiểm tra tài liệu của service. [AWS Documentation](https://docs.aws.amazon.com/tag-editor/latest/userguide/best-practices-and-strats.html?utm_source=chatgpt.com)

Không cần học thuộc 50 như mục tiêu Day 7 chính.

Quan trọng hơn là:

```text
Tagging is governance metadata.
```

---

# 36. Guided Lab Day 7

## Safety Gate trước khi bắt đầu

Chỉ thực hành trên:

```text
personal sandbox
training account
approved development account
```

Không tự ý thực hiện trên production.

Trước bất kỳ IAM change nào, phải biết:

```text
AWS Account ID
Environment
Current identity
Change được phê duyệt
Expected permission
Rollback owner
```

Không tạo:

```text
AdministratorAccess
root access key
unapproved IAM users
unapproved permanent access keys
```

---

# 37. Lab Architecture

Lab target:

```text
Local Linux workstation
        |
        | AWS CLI
        v
Named CLI Profile
        |
        | Authentication
        v
Approved AWS Identity
        |
        | MFA / temporary credentials
        v
IAM Authorization
        |
        v
AWS APIs
        |
        +--- Verify identity
        +--- Inspect IAM
        +--- Verify Region
        +--- Verify tags
```

---

# 38. Lab Part A — Workstation pre-check

Run:

```bash
date
```

```bash
whoami
```

```bash
uname -a
```

```bash
aws --version
```

Expected invariant:

```text
Linux shell works
Correct local user
AWS CLI available
AWS CLI v2 preferred
```

Tạo evidence directory:

```bash
mkdir -p ~/java-fresher/day07/evidence
```

```bash
chmod 700 ~/java-fresher/day07/evidence
```

Check:

```bash
ls -ld ~/java-fresher/day07/evidence
```

Expected:

```text
drwx------ ...
```

Không lưu secret/access keys trong evidence folder.

---

# 39. Lab Part B — Xác định approved AWS access method

Nếu instructor/company cung cấp SSO, dùng SSO.

Nếu cung cấp IAM user, dùng IAM user.

Nếu cung cấp assumed role workflow, dùng role.

Không tự đổi authentication architecture.

Flow quyết định:

```text
Approved access?
       |
       +-- Identity Center --> configure SSO
       |
       +-- IAM User --------> MFA + approved CLI method
       |
       +-- Role ------------> configure role profile
```

---

# 40. Lab Part C — MFA

Trong AWS Console, vào security credentials của identity được cấp và xác nhận MFA theo approved procedure.

AWS hiện ưu tiên passkeys/security keys khi có thể vì có khả năng chống phishing tốt hơn TOTP. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html?utm_source=chatgpt.com)

Sau khi hoàn tất, evidence cần chứng minh:

```text
identity
MFA enabled/assigned
account/environment context
date/time if your process requires it
```

Không capture QR-code enrollment secret.

Không capture OTP seed.

Không gửi TOTP code vào evidence.

---

# 41. Lab Part D — Configure CLI Profile

### Preferred SSO path

Ví dụ:

```bash
aws configure sso \
  --profile java-fresher-lab
```

Sau login:

```bash
aws sso login \
  --profile java-fresher-lab
```

Kiểm tra:

```bash
aws configure list-profiles
```

và:

```bash
aws configure list \
  --profile java-fresher-lab
```

AWS CLI supports named profiles và `--profile` để chọn profile cụ thể. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html)

---

## Nếu lab được cấp IAM access key

Chỉ nếu instructor/account owner **explicitly approves** permanent IAM-user programmatic credentials:

```bash
aws configure \
  --profile java-fresher-lab
```

CLI hỏi:

```text
AWS Access Key ID
AWS Secret Access Key
Default region name
Default output format
```

Ví dụ Region:

```text
ap-southeast-1
```

Output:

```text
json
```

Sau đó bảo vệ local files:

```bash
chmod 700 ~/.aws
```

```bash
chmod 600 ~/.aws/credentials 2>/dev/null || true
```

```bash
chmod 600 ~/.aws/config 2>/dev/null || true
```

Không chạy:

```bash
cat ~/.aws/credentials
```

rồi paste vào evidence.

---

# 42. Lab Part E — Verify identity

Đây là checkpoint quan trọng nhất.

```bash
aws sts get-caller-identity \
  --profile java-fresher-lab
```

Expected invariant:

```text
Account = approved account
ARN = approved identity/role
Command succeeds
```

Lưu **non-secret output** làm evidence:

```bash
aws sts get-caller-identity \
  --profile java-fresher-lab \
  > ~/java-fresher/day07/evidence/caller-identity.json
```

Sau đó:

```bash
cat ~/java-fresher/day07/evidence/caller-identity.json
```

`GetCallerIdentity` là command thích hợp để xác nhận caller identity. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/sts/get-caller-identity.html?utm_source=chatgpt.com)

---

# 43. Lab Part F — Verify Region

```bash
aws configure get region \
  --profile java-fresher-lab
```

Ví dụ:

```text
ap-southeast-1
```

Sau đó:

```bash
aws configure list \
  --profile java-fresher-lab
```

Bạn phải biết:

```text
profile
credential source
Region
```

Không được tiếp tục tạo resource nếu account hoặc Region sai.

---

# 44. Lab Part G — Inspect IAM identity/policies

Nếu permissions cho phép và identity là IAM user:

```bash
aws iam get-user \
  --profile java-fresher-lab
```

Có thể inspect attached managed policies:

```bash
aws iam list-attached-user-policies \
  --user-name <APPROVED_USER_NAME> \
  --profile java-fresher-lab
```

Nếu `AccessDenied`, **không cố bypass**.

Đó có thể chính là expected least privilege.

Ghi evidence:

```text
command
timestamp
profile
result
AccessDenied if applicable
```

---

# 45. Lab Part H — Understand policy

Giả sử instructor đưa:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:DescribeInstances", "ec2:DescribeVolumes"],
      "Resource": "*"
    }
  ]
}
```

Bạn phải đọc ra được:

```text
Can describe EC2 instances.
Can describe EBS volumes.
Cannot infer permission to start EC2.
Cannot infer permission to stop EC2.
Cannot infer permission to terminate EC2.
```

Đừng nhìn:

```text
"ec2:"
```

rồi kết luận:

> “Có quyền EC2.”

Phải đọc **Action cụ thể**.

---

# 46. Lab Part I — Mandatory Resource Tags

Giả sử approved resource hoặc IAM identity của lab hỗ trợ tagging.

Tag schema cho Day 7:

```text
Project=java-fresher
Environment=lab
Purpose=training
Owner=<approved-non-sensitive-owner-value>
```

Nếu lab cho phép tag IAM user:

```bash
aws iam tag-user \
  --user-name <APPROVED_USER_NAME> \
  --tags \
    Key=Project,Value=java-fresher \
    Key=Environment,Value=lab \
    Key=Purpose,Value=training \
  --profile java-fresher-lab
```

Sau đó verify:

```bash
aws iam list-user-tags \
  --user-name <APPROVED_USER_NAME> \
  --profile java-fresher-lab
```

Expected invariant:

```text
Required keys exist
Values are correct
No sensitive information
```

Nếu organization cung cấp một resource khác để tag thì sử dụng resource đó và API tương ứng, không tự chọn production resource.

---

# 47. Tagging validation mindset

Đừng chỉ nói:

> “Command không báo lỗi nên chắc tag đã có.”

Luôn đọc lại state.

Pattern:

```text
CHANGE
  ↓
READ BACK
  ↓
COMPARE
  ↓
EVIDENCE
```

Đây sẽ là pattern xuyên suốt 22 Days.

---

# 48. Evidence Pack cuối Lab

Evidence folder có thể trông như:

```text
day07/
└── evidence/
    ├── workstation.txt
    ├── aws-cli-version.txt
    ├── caller-identity.json
    ├── cli-config-redacted.txt
    ├── iam-policy-review.txt
    ├── resource-tags.json
    └── mfa-evidence.png
```

Không được chứa:

```text
aws_secret_access_key
session_token
password
TOTP secret
QR enrollment code
private key
```

---

# 49. Failure Injection 1 — Wrong profile

Giả sử:

```bash
aws sts get-caller-identity \
  --profile does-not-exist
```

Có thể thấy dạng:

```text
The config profile (does-not-exist) could not be found
```

Troubleshooting:

```bash
aws configure list-profiles
```

Root cause:

```text
profile name incorrect
```

Remediation:

```text
use correct approved profile
```

Không cần reinstall AWS CLI.

---

# 50. Failure Injection 2 — Missing credentials

Triệu chứng thường gặp:

```text
Unable to locate credentials
```

Failing layer:

```text
Authentication configuration
```

Không phải:

```text
EC2 failure
VPC failure
Security Group failure
```

Kiểm tra:

```bash
aws configure list \
  --profile java-fresher-lab
```

Nếu SSO:

```bash
aws sso login \
  --profile java-fresher-lab
```

---

# 51. Failure Injection 3 — Expired temporary credentials

Có thể thấy error tương tự:

```text
ExpiredToken
```

Mental model:

```text
Credentials existed
        ↓
Temporary session expired
        ↓
AWS rejects request
```

Nếu SSO:

```bash
aws sso login --profile java-fresher-lab
```

Nếu assumed role, phải renew session thông qua approved source identity/process.

Không biến temporary credential issue thành permanent IAM access key workaround.

---

# 52. Failure Injection 4 — AccessDenied

Ví dụ:

```text
AccessDenied
User is not authorized to perform ec2:StartInstances
```

Đừng lập tức yêu cầu:

```text
AdministratorAccess
```

Evidence-driven analysis:

```text
Identity?
↓
Account?
↓
Requested API?
↓
Resource?
↓
Policy Allows?
↓
Explicit Deny?
↓
Permission boundary/SCP/resource policy?
```

Với fresher L1, thường nhiệm vụ là xác định:

```text
Who
tried What
on Which Resource
in Which Account/Region
at What Time
and received Which Error
```

rồi escalate nếu permission change ngoài scope.

---

# 53. Failure Injection 5 — Wrong Region

Bạn chạy:

```bash
aws ec2 describe-instances \
  --profile java-fresher-lab
```

Output:

```text
Reservations: []
```

Fresher có thể kết luận:

> “Không có EC2.”

Sai.

Trước hết:

```bash
aws configure get region \
  --profile java-fresher-lab
```

và nếu được phép:

```bash
aws ec2 describe-instances \
  --region ap-southeast-1 \
  --profile java-fresher-lab
```

Resource có thể ở Region khác.

---

# 54. Failure Injection 6 — Wrong AWS Account

Đây nghiêm trọng hơn wrong Region.

Luôn:

```bash
aws sts get-caller-identity \
  --profile java-fresher-lab
```

Nếu:

```text
Expected Account: 111111111111
Actual Account:   222222222222
```

**STOP.**

Không tiếp tục change.

Đây là operational safety gate.

---

# 55. Failure Injection 7 — Tags missing

Symptom:

```text
Resource exists
but Environment/Owner/Project tags missing
```

Questions:

```text
Did creation request include tags?
Do you have tagging permission?
Were tag keys spelled correctly?
Case mismatch?
Did policy deny TagResource/CreateTags?
Is organization enforcing a tag policy?
```

Không “fix” bằng cách đặt random tag values.

Phải theo approved tagging standard.

---

# 56. Troubleshooting Framework Day 7

Khi AWS command fail, hãy đi theo thứ tự:

```text
1. Symptom
        ↓
2. Expected behavior
        ↓
3. Recent change
        ↓
4. Correct workstation/tool?
        ↓
5. Correct CLI profile?
        ↓
6. Correct credentials?
        ↓
7. Correct identity/account?
        ↓
8. Correct Region?
        ↓
9. Correct IAM permission?
        ↓
10. Service/resource state?
        ↓
11. Safe remediation or escalation
        ↓
12. Validation
        ↓
13. Evidence
```

Đừng nhảy trực tiếp từ:

```text
AccessDenied
```

sang:

```text
Network is broken
```

---

# 57. L1 Remediation vs Escalation

Một fresher/L1 thường có thể an toàn thực hiện những việc đã được phê duyệt như xác nhận profile, đăng nhập lại SSO, kiểm tra identity, kiểm tra Region, đọc error, kiểm tra tags, verify assigned permissions và thu thập evidence.

Bạn nên escalate thay vì tự sửa nếu vấn đề đòi hỏi thay đổi production IAM policy, tạo AdministratorAccess, sửa SCP, permission boundary, cross-account trust, root credentials, security baseline hoặc bất kỳ permission change nào chưa được phê duyệt.

---

# 58. Security Anti-Patterns

Các pattern sau phải coi là red flag:

```text
Root user for daily tasks
Shared IAM account
One IAM user used by whole team
AdministratorAccess for convenience
Credentials committed to Git
Secret key pasted into ticket/chat
Access keys stored in Java properties
No MFA
One default CLI profile for all accounts
No verification of Account ID before change
Production and lab credentials mixed together
Sensitive information stored in tags
```

---

# 59. Operations / Recovery / Rollback

Giả sử hôm nay bạn cấu hình sai CLI profile.

Rollback local:

```text
identify wrong profile
backup config if required
remove/correct only affected profile
re-run identity check
```

Nếu IAM access key bị lộ:

```text
Treat as credential compromise.
```

Không chỉ:

```text
delete terminal history
```

Credential phải được deactivate/revoked/rotated theo organization process, scope impact phải được đánh giá và audit evidence phải được bảo toàn.

Nếu policy attach nhầm:

```text
Record current state
Identify change
Use approved rollback
Detach/revert affected policy
Verify effective access
Preserve evidence
```

Không tự rollback production IAM nếu bạn không có change authority.

---

# 60. Cost Awareness

Day 7 chủ yếu xoay quanh identity/access/configuration nên **không cần launch EC2, EBS, NAT Gateway hay RDS** để hoàn thành core lab.

Vì vậy không nên tạo compute/network resources chỉ để “có cái test”.

Nếu instructor cung cấp existing resource, dùng resource đó.

Nếu bạn phải tạo resource ở các Day sau:

```text
know resource
know Region
know pricing model
tag it
record it
clean it up
```

Cost management sẽ được học sâu hơn ở Day 11.

---

# 61. Cleanup Day 7

Cuối lab, cleanup **chỉ các artifact được phép cleanup**.

Local temporary shell override:

```bash
unset AWS_PROFILE
```

Nếu lab yêu cầu xóa local experimental profile, thực hiện theo instructor procedure.

Nếu SSO session chỉ cần kết thúc:

```bash
aws sso logout
```

Nếu instructor yêu cầu retain profile để học Day 8–11 thì không xóa.

Không tự delete:

```text
IAM user
IAM role
company access key
MFA device
managed policy
```

nếu chúng được organization quản lý.

---

# 62. Day 7 Operational Runbook mẫu

Trước change:

```text
Confirm ticket/approval
Confirm AWS account
Confirm profile
Confirm Region
Confirm identity
Confirm scope
```

Trong change:

```text
Use least privilege
Perform one controlled change
Capture result
Read back state
```

Sau change:

```text
Validate
Capture evidence
Check mandatory tags
Record deviations
Rollback if unsuccessful
Escalate if outside L1 scope
```

---

# 63. Exam Focus

### Theory

Bạn phải giải thích được bằng lời của mình:

```text
Region vs AZ
Why multi-AZ exists
Security OF vs IN the cloud
IAM user vs IAM role
Authentication vs authorization
Allow vs explicit Deny
Action vs Resource
Role trust policy vs permissions policy
Least privilege
Why MFA matters
Why temporary credentials are preferred
What AWS CLI profile does
Why get-caller-identity matters
Why tagging matters
```

### Practical

Bạn nên làm được mà không cần tra từng bước:

```bash
aws --version
```

```bash
aws configure list-profiles
```

```bash
aws configure list --profile java-fresher-lab
```

```bash
aws sts get-caller-identity --profile java-fresher-lab
```

```bash
aws configure get region --profile java-fresher-lab
```

và biết cách verify mandatory tags trên resource được cấp.

---

# 64. Knowledge Check

1. Một EC2 instance nằm trong `ap-southeast-1a`. `ap-southeast-1a` là Region hay AZ?

2. AWS data center mất nguồn điện vật lý. Responsibility chính thuộc AWS hay customer?

3. Amazon Linux EC2 chưa được security patch. Responsibility chính thuộc AWS hay customer?

4. IAM user có `Allow ec2:*` nhưng một applicable policy khác có `Deny ec2:TerminateInstances`. User có terminate instance được không?

5. MFA có tự động làm user thành Administrator không?

6. IAM role khác IAM user quan trọng nhất ở điểm nào?

7. `aws sts get-caller-identity` dùng để chứng minh điều gì?

8. Vì sao trước một destructive AWS command phải kiểm tra cả Account và Region?

9. `Environment=prod` và `environment=prod` có chắc là cùng một tag key không?

10. Vì sao không được lưu password hoặc access key trong AWS tags?

11. CLI báo `ExpiredToken`; layer nào nên được điều tra đầu tiên?

12. CLI `describe-instances` trả về empty. Có được kết luận ngay “account không có EC2” không?

13. Trust policy của role trả lời câu hỏi gì?

14. Permissions policy của role trả lời câu hỏi gì?

15. Vì sao AWS hiện ưu tiên temporary credentials cho human/workload access?

---

# 65. Independent Practical Challenge

Sau khi học phần trên, bạn nên tự thực hiện challenge này trên **approved lab AWS account**:

```text
Stage 1
Identify your AWS access mechanism.

Stage 2
Verify MFA.

Stage 3
Configure a named profile:
java-fresher-lab

Stage 4
Prove:
AWS CLI version
profile
identity
account
Region

Stage 5
Inspect your assigned IAM permissions
without changing them.

Stage 6
Apply/verify mandatory approved tags
on the designated lab resource.

Stage 7
Inject one safe failure:
wrong profile name OR expired SSO session.

Stage 8
Diagnose and recover.

Stage 9
Produce evidence pack.

Stage 10
Document:
symptom
root cause
remediation
validation
evidence
```

Tuyệt đối không inject failure bằng cách xóa IAM user, remove MFA, disable production credentials hoặc tự sửa organization IAM policy.

---

# 66. Mastery Check Day 7

Bạn **chưa nên xem Day 7 là hoàn thành chỉ vì đã đọc bài này**.

Day 7 chỉ đạt practical mastery khi bạn thực sự chứng minh được flow:

```text
Approved Access
     ↓
MFA
     ↓
CLI Profile
     ↓
Credentials
     ↓
STS Identity Verification
     ↓
Correct AWS Account
     ↓
Correct Region
     ↓
IAM Authorization
     ↓
Tagged Resource
     ↓
Evidence
```

Nếu một mắt xích fail, bạn phải có khả năng tìm layer lỗi thay vì thử command ngẫu nhiên.

Điểm quan trọng nhất cần mang sang Day 8–11 là:

```text
NEVER make an AWS change
until you know:

WHO you are
WHICH account
WHICH Region
WHAT permission
WHAT resource
WHAT rollback
```

Đó là tư duy Administration/Operations đúng hơn rất nhiều so với chỉ biết click AWS Console.

AWS Shared Responsibility xác định rõ AWS bảo vệ infrastructure của cloud, trong khi customer vẫn chịu trách nhiệm nhiều lớp configuration, identity, data, OS/application tùy service. [Amazon Web Services, Inc.](https://aws.amazon.com/compliance/shared-responsibility-model/) AWS cũng hiện khuyến nghị temporary credentials, MFA và least privilege cho identity management. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

**Phần tiếp theo hợp lý nhất của Day 7 là thực hành theo dạng guided lab thực chiến:** bạn chạy từng command trên AWS lab của mình, gửi lại **output đã che thông tin nhạy cảm**, và mình sẽ đóng vai senior operator kiểm tra từng evidence, cho failure injection và chỉ chuyển bước khi configuration của bạn thực sự đúng.
