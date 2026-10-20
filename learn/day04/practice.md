# Day 4 — Module Thực hành

## AWS IAM, VPC, EC2, SSH, IMDSv2, Verification, Troubleshooting, Evidence & Rollback Lab

Module Thực hành Day 4 biến toàn bộ phần Lý thuyết thành một workflow giống công việc Infrastructure Operations thực tế: **nhận authorization → xác minh account/identity → kiểm tra access boundary → xây VPC → kiểm tra network controls → provision EC2 Linux/RHEL → bootstrap service → SSH → verify IMDSv2 → kiểm tra allowed/denied behavior → troubleshoot → thu thập evidence → rollback/cleanup**.

Theo syllabus, Day 4 bắt buộc phải thực hành IAM/MFA và AWS CLI authentication; mandatory tags; VPC với public/private subnet concepts, route tables, internet controls, Security Group và NACL checks; launch EC2 Linux bằng approved AMI, key pair, IAM role và bootstrap script; SSH; IMDS protection; network path; allowed/denied connectivity; provisioning logs; architecture/IAM/EC2/network evidence. Phần additive còn yêu cầu verify active account/identity trước mỗi operation, thực hiện sandbox role-access scenario, test một action được phép và một action bị deny, gắn evidence với ServiceNow-style ticket, đồng thời tuyệt đối không đưa password, MFA seed, token hay temporary credentials vào evidence. Infrastructure_Operations_14_Da…

---

# 1. Mục tiêu cuối cùng của Lab

Sau khi hoàn thành module này, bạn phải có khả năng tự giải thích và thực hiện được workflow:

```text
Approved Ticket
     │
     ▼
Verify Identity
     │
     ▼
Verify AWS Account + Region
     │
     ▼
Validate IAM Boundary
     │
     ├── Allowed action → PASS
     └── Denied action  → PASS
     │
     ▼
Validate mandatory tags
     │
     ▼
Create VPC
     │
     ├── Public Subnet
     └── Private Subnet
     │
     ▼
Configure Routing
     │
     ├── local
     └── Internet Gateway
     │
     ▼
Inspect NACL
     │
     ▼
Configure Security Group
     │
     ▼
Launch EC2
     │
     ├── Approved AMI
     ├── Instance Type
     ├── Key Pair
     ├── IAM Instance Profile
     ├── User Data
     ├── IMDSv2
     └── Tags
     │
     ▼
Verify Instance Health
     │
     ▼
SSH
     │
     ▼
Verify Linux/RHEL
     │
     ▼
Verify Bootstrap Logs
     │
     ▼
Verify IMDSv2
     │
     ▼
Allowed / Denied Connectivity
     │
     ▼
Failure Injection
     │
     ▼
Root Cause + Recovery
     │
     ▼
Evidence Package
     │
     ▼
Rollback / Cleanup
```

Đích của lab **không phải chỉ “EC2 chạy được”**.

Một lab đạt yêu cầu phải chứng minh:

| Khu vực           | Điều phải chứng minh                                           |
| ----------------- | -------------------------------------------------------------- |
| Authorization     | Có ticket/approval và scope                                    |
| Identity          | Biết chính xác principal đang thao tác                         |
| AWS scope         | Đúng Account và Region                                         |
| IAM               | Allowed action thành công, denied action thất bại đúng dự kiến |
| Governance        | Mandatory tags tồn tại                                         |
| VPC               | CIDR/subnets/routes đúng thiết kế                              |
| Network security  | SG/NACL được hiểu và verified                                  |
| EC2               | Đúng AMI/type/subnet/profile/tags                              |
| Bootstrap         | Script thực sự chạy thành công                                 |
| SSH               | Secure connectivity hoạt động                                  |
| IMDS              | IMDSv2 required và hoạt động                                   |
| Negative test     | Unauthorized path bị deny                                      |
| Evidence          | Có pre/post evidence và timestamp                              |
| Credential safety | Không secret nào lọt vào evidence                              |
| Rollback          | Có thể khôi phục configuration sau test                        |
| Cleanup           | Lab resources được xử lý an toàn                               |

---

# 2. Operational Safety Contract

Đây là **lab/non-production/authorized environment**.

Không thực hiện các command bên dưới trong production chỉ vì command giống nhau.

Trước bất kỳ configuration-changing operation nào, phải biết:

```text
AWS Account ID
AWS Region
Current Principal
Ticket / Change / Request
Environment = Lab/Sandbox
Resource ownership
Rollback method
```

Ba loại command cần được phân biệt:

| Loại              | Ví dụ                           | Risk       |
| ----------------- | ------------------------------- | ---------- |
| Read-only         | `describe-*`, `get-*`, `list-*` | Thấp       |
| Controlled change | create VPC, modify SG           | Trung bình |
| Destructive       | terminate EC2, delete VPC       | Cao hơn    |

Nguyên tắc lab:

```text
READ
   ↓
UNDERSTAND
   ↓
CHANGE
   ↓
VERIFY
   ↓
EVIDENCE
```

Không làm:

```text
CHANGE
↓
CHANGE
↓
CHANGE
↓
"Không chạy, reboot thử"
```

---

# 3. Secret Handling Rules

Những thứ sau **không được copy vào chat, ticket, screenshot hay evidence repository**:

```text
AWS Secret Access Key
AWS Session Token
Temporary STS credentials
Password
MFA TOTP seed
Private SSH key
SSO token
Browser session cookie
Authorization header
```

AWS hiện khuyến nghị human access sử dụng temporary credentials thông qua roles/federation/IAM Identity Center khi có thể, thay vì long-term IAM access keys; MFA vẫn là control quan trọng cho identity. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

# 4. Lab Topology

Topology chính:

```text
                     Administrative Workstation
                       AWS CLI + SSH client
                              │
                              │
                 AWS Console / CLI / SSH
                              │
                              ▼
┌───────────────────────────────────────────────────────────────┐
│                    AWS SANDBOX ACCOUNT                        │
│                                                               │
│  Human Identity                                               │
│  IAM Identity Center / Federated / approved IAM user          │
│            │                                                  │
│            ▼                                                  │
│      Day4 Operator Role                                       │
│                                                               │
│                 VPC: 10.40.0.0/16                             │
│                                                               │
│      ┌─────────────────────┐     ┌──────────────────────┐     │
│      │ Public Subnet       │     │ Private Subnet       │     │
│      │ 10.40.10.0/24       │     │ 10.40.20.0/24        │     │
│      │                     │     │                      │     │
│      │  EC2 Linux/RHEL     │     │ No EC2 required      │     │
│      │  + Public IPv4      │     │ for base lab         │     │
│      │  + IAM Role         │     │                      │     │
│      └─────────┬───────────┘     └──────────┬───────────┘     │
│                │                            │                 │
│          Public Route Table            Private RT             │
│                │                            │                 │
│         0.0.0.0/0 → IGW                local only             │
│                │                                              │
│                ▼                                              │
│         Internet Gateway                                      │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

Trong base lab, `NAT Gateway` **không cần tạo**. Syllabus chỉ yêu cầu `NAT gateway awareness`, và NAT là resource có cost. Ta sẽ hiểu route architecture và có optional extension nếu sandbox cho phép.

AWS định nghĩa subnet là public khi route table của nó có route tới Internet Gateway. Private subnet không có direct IGW route. Với IPv4, EC2 trong public subnet còn cần public IPv4/Elastic IP cùng với route/security phù hợp mới giao tiếp Internet được. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

---

# 5. Addressing Plan

Ta dùng:

| Object         | CIDR            |
| -------------- | --------------- |
| VPC            | `10.40.0.0/16`  |
| Public subnet  | `10.40.10.0/24` |
| Private subnet | `10.40.20.0/24` |

Public subnet:

```text
10.40.0.0/16 → local
0.0.0.0/0    → Internet Gateway
```

Private subnet trong base lab:

```text
10.40.0.0/16 → local
```

Optional NAT architecture:

```text
Private subnet
      │
0.0.0.0/0
      │
      ▼
NAT Gateway
      │
      ▼
Internet Gateway
```

AWS hiện có cả NAT architecture truyền thống theo AZ và `Regional NAT Gateway`; Day 4 không yêu cầu đi sâu vào deployment model này. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat.html?utm_source=chatgpt.com)

---

# 6. Prerequisites

Workstation cần:

```text
AWS CLI v2
SSH client
curl
Text editor
Secure evidence directory
Approved AWS sandbox access
Approved AWS Region
Approved Linux/RHEL AMI
Approved IAM role/profile
```

Check AWS CLI:

```bash
aws --version
```

Expected dạng:

```text
aws-cli/2.x.x ...
```

Không cần version giống ví dụ; mục tiêu là dùng supported AWS CLI v2.

---

# 7. Tạo Workspace cho Lab

Trên Linux/macOS/WSL:

```bash
mkdir -p ~/infra-lab/day4/{evidence,config,scripts}
cd ~/infra-lab/day4
```

Kiểm tra:

```bash
pwd
find . -maxdepth 1 -type d -print
```

Expected:

```text
.
./evidence
./config
./scripts
```

Không tạo:

```text
./credentials
./secrets
./private-keys-for-submission
```

Private SSH key phải ở secure location riêng, ví dụ:

```text
~/.ssh/
```

và **không copy vào `evidence/`**.

---

# 8. Evidence Naming Convention

Một naming convention tốt:

```text
D4-E001_identity.txt
D4-E002_role_allowed.txt
D4-E003_role_denied.txt
D4-E004_vpc.json
D4-E005_subnets.json
D4-E006_routes.json
D4-E007_sg.json
D4-E008_nacl.json
D4-E009_ec2.json
D4-E010_metadata-options.json
D4-E011_cloud-init.txt
D4-E012_network-tests.txt
D4-E013_failure-recovery.txt
```

Evidence manifest chung của syllabus yêu cầu traceability tới source/ticket, timestamp/timezone, collector, command/filter, file details, SHA-256, classification và limitations; raw evidence phải được giữ nguyên, transformation nên làm trên copy. Infrastructure_Operations_14_Da…

---

# 9. ServiceNow-style Access Ticket

Day 4 chưa học ServiceNow đầy đủ, nhưng syllabus yêu cầu role access scenario gắn với một `ServiceNow-style access ticket`.

Ví dụ:

| Field           | Sample                               |
| --------------- | ------------------------------------ |
| Ticket          | `REQ-AWS-D4-001`                     |
| Requestor       | `<learner-id>`                       |
| Environment     | `AWS Sandbox`                        |
| Business Owner  | `<approved-owner>`                   |
| AWS Account     | `<account-id>`                       |
| Requested Role  | `Day4OperatorRole`                   |
| Purpose         | Day 4 training lab                   |
| Requested Scope | IAM read test + VPC/EC2 lab          |
| Approval        | `<approval-reference>`               |
| Start           | `<timestamp>`                        |
| Expiry          | `<timestamp>`                        |
| Change Boundary | Sandbox resources tagged `Day4` only |
| Rollback        | Delete/revert lab resources          |
| Secret Handling | No secrets in evidence               |

Điều quan trọng:

```text
Technical permission
+
Business authorization
```

phải reconcile với nhau.

---

# 10. Chuẩn bị Variables

Không hard-code IDs lặp đi lặp lại.

Ví dụ:

```bash
export PROFILE="day4-sandbox"
export AWS_REGION="<approved-region>"
export AZ="<approved-availability-zone>"

export OWNER="<learner-id>"
export TICKET="REQ-AWS-D4-001"

export VPC_CIDR="10.40.0.0/16"
export PUBLIC_CIDR="10.40.10.0/24"
export PRIVATE_CIDR="10.40.20.0/24"

export ADMIN_CIDR="<approved-public-ip>/32"

export INSTANCE_TYPE="<approved-instance-type>"
export AMI_ID="<approved-rhel-or-linux-ami-id>"
```

Ví dụ:

```text
AWS_REGION=ap-southeast-1
```

chỉ dùng nếu sandbox policy cho phép Region đó.

Không tự chọn Region chỉ vì “gần Việt Nam”.

---

# 11. Phase 1 — Verify AWS Identity

## Mục tiêu

Trước **mọi operation**, trả lời:

```text
Who am I?
Which AWS account?
Which session/role?
Which Region?
```

Command quan trọng nhất:

```bash
aws sts get-caller-identity \
  --profile "$PROFILE"
```

AWS `GetCallerIdentity` trả identity đang dùng credentials hiện tại và là một trong những cách tốt nhất để tránh thao tác nhầm principal/account. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/sts/get-caller-identity.html?utm_source=chatgpt.com)

Expected sanitized form:

```json
{
  "UserId": "AROAXXXXXXXXX:session-name",
  "Account": "123456789012",
  "Arn": "arn:aws:sts::123456789012:assumed-role/Day4OperatorRole/session-name"
}
```

### Bạn cần đọc output thế nào?

```text
Account
```

phải match approved sandbox account.

```text
Arn
```

phải match expected user/role/session.

Nếu bạn mong:

```text
Day4OperatorRole
```

mà thấy:

```text
ProductionAdministrator
```

thì:

```text
STOP
```

Không chạy tiếp.

---

# 12. Capture Identity Evidence

Nếu output không chứa secret:

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ" \
  | tee evidence/D4-E001_timestamp.txt

aws sts get-caller-identity \
  --profile "$PROFILE" \
  | tee evidence/D4-E001_identity.json
```

Hash:

```bash
sha256sum evidence/D4-E001_identity.json
```

Giữ hash trong evidence manifest.

---

# 13. Verify Region

```bash
aws configure get region \
  --profile "$PROFILE"
```

Nếu trống, command có thể đang dựa vào environment hoặc explicit `--region`.

Check:

```bash
env | grep '^AWS_' | sed '/SECRET/d;/TOKEN/d;/ACCESS_KEY/d'
```

Lưu ý: không dump environment nguyên trạng vì có thể lộ credential environment variables.

Safer approach:

```bash
printf 'AWS_REGION=%s\n' "$AWS_REGION"
printf 'AWS_PROFILE=%s\n' "$PROFILE"
```

---

# 14. Nếu Environment dùng IAM Identity Center

AWS CLI hiện hỗ trợ:

```bash
aws configure sso --profile day4-sandbox
```

Sau configuration:

```bash
aws sso login --profile day4-sandbox
```

AWS CLI sẽ dùng IAM Identity Center session để obtain temporary AWS credentials cho assigned account/role. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html?utm_source=chatgpt.com)

Sau login:

```bash
aws sts get-caller-identity \
  --profile day4-sandbox
```

Không screenshot:

```text
authorization code
SSO token
session cache
```

---

# 15. Nếu Environment dùng IAM User

Đây là compatibility/sandbox path, không phải workforce model ưu tiên.

IAM user lab chỉ nên dùng khi:

```text
instructor explicitly provisions it
or
environment specifically requires it
```

MFA phải được configured theo approved method.

Không lưu:

```text
MFA seed QR code
TOTP seed
recovery secret
```

Evidence có thể chỉ ghi:

```text
Identity: learner-day4
MFA state: Enabled
Verified at: ...
Evidence screenshot: masked/no QR code
```

AWS best practice hiện ưu tiên roles/federation/temporary credentials cho human access, và IAM users chỉ nên tồn tại khi use case thực sự cần long-term IAM identity. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

# 16. Phase 2 — Verify Role Access Boundary

Ta cần hai tests:

```text
Expected Allow
Expected Deny
```

Điểm quan trọng là denied test nên là **read-only operation**, để nếu IAM configuration vô tình rộng hơn dự kiến cũng không gây thay đổi.

Giả sử role được thiết kế để inspect EC2 nhưng không inspect IAM directory.

---

# 17. Allowed Action Test

```bash
aws ec2 describe-instances \
  --max-results 5 \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Expected:

```text
command succeeds
```

Không quan trọng account hiện có EC2 hay không.

Có thể output:

```json
{
  "Reservations": []
}
```

vẫn là successful authorization.

Evidence:

```bash
aws ec2 describe-instances \
  --max-results 5 \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  > evidence/D4-E002_allowed-ec2.json
```

---

# 18. Denied Action Test

Ví dụ:

```bash
aws iam list-users \
  --max-items 1 \
  --profile "$PROFILE"
```

Expected:

```text
AccessDenied
```

Nếu role được thiết kế không có IAM directory permissions.

Điểm quan trọng:

```text
AccessDenied
```

ở đây là:

```text
EXPECTED RESULT
```

và vì vậy:

```text
SECURITY TEST = PASS
```

AWS evaluation mặc định deny request không được explicitly allowed; applicable explicit `Deny` luôn thắng `Allow`. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-denyallow.html?utm_source=chatgpt.com)

---

# 19. Nếu Denied Action Lại Thành Công

Không tiếp tục nghĩ:

> “Hay quá, quyền rộng hơn nên dễ làm lab.”

Kết quả đúng là:

```text
Expected: DENY
Actual: ALLOW
Status: FAIL
```

Potential issue:

```text
Excessive entitlement
Wrong role
Wrong profile
Wrong permission set
Incorrect policy
```

Bạn phải record:

```text
Principal ARN
Expected scope
Unexpected action
Timestamp
Ticket
```

và escalate theo IAM/access governance.

---

# 20. Phase 3 — Mandatory Tag Guardrail

Syllabus yêu cầu:

```text
enforce mandatory resource tags
```

Lab nên có ít nhất:

```text
Environment
Owner
Ticket
Name
```

Ví dụ:

```text
Environment=Lab
Owner=<learner-id>
Ticket=REQ-AWS-D4-001
```

Không đặt secret vào tag; AWS lưu ý tags xuất hiện trong nhiều API responses và không nên chứa confidential data. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html?utm_source=chatgpt.com)

---

# 21. Enforce Tags theo IAM

AWS hỗ trợ policy conditions:

```text
aws:RequestTag/<key>
aws:TagKeys
```

để kiểm soát tags trong resource creation requests. Với EC2, tag-on-create còn liên quan `ec2:CreateTags`; AWS cung cấp chính thức examples yêu cầu tags khi chạy `RunInstances`. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ExamplePolicies_EC2.html?utm_source=chatgpt.com)

Trong enterprise lab, policy này nên được **instructor/admin provision**.

Learner không cần Administrator permission chỉ để tự cấp guardrail cho chính mình.

---

# 22. Kiểm tra Guardrail bằng `--dry-run`

Một ưu điểm lớn của EC2 CLI là:

```text
--dry-run
```

kiểm tra authorization mà **không thực sự chạy operation**.

Nếu authorized:

```text
DryRunOperation
```

Nếu unauthorized:

````text
UnauthorizedOperation
``` :chatgpt-content-reference{index="11"}


Do `RunInstances` có nhiều parameters, dry-run test chỉ nên chạy sau khi các resource dependencies cần thiết đã có. Ta sẽ quay lại test mandatory tags ngay trước Phase Launch.

---

# 23. Phase 4 — Create VPC

Trước change:

```bash
aws sts get-caller-identity \
  --profile "$PROFILE"
````

Sau đó:

```bash
VPC_ID=$(
  aws ec2 create-vpc \
    --cidr-block "$VPC_CIDR" \
    --tag-specifications \
      "ResourceType=vpc,Tags=[
        {Key=Name,Value=day4-vpc},
        {Key=Environment,Value=Lab},
        {Key=Owner,Value=$OWNER},
        {Key=Ticket,Value=$TICKET}
      ]" \
    --query 'Vpc.VpcId' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)

echo "$VPC_ID"
```

Expected:

```text
vpc-xxxxxxxxxxxxxxxxx
```

---

# 24. Wait VPC

```bash
aws ec2 wait vpc-available \
  --vpc-ids "$VPC_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Sau đó:

```bash
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

---

# 25. Verification VPC

Bạn phải verify:

```text
VpcId
State
CidrBlock
Tags
```

Expected:

```text
State = available
CIDR = 10.40.0.0/16
Environment = Lab
Owner = expected learner
Ticket = expected ticket
```

Capture:

```bash
aws ec2 describe-vpcs \
  --vpc-ids "$VPC_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  > evidence/D4-E004_vpc.json
```

---

# 26. Troubleshooting Checkpoint — VPC Creation Fail

Nếu:

```text
UnauthorizedOperation
```

check:

```text
identity
role
permission boundary
SCP
ticket scope
```

Nếu:

```text
InvalidVpc.Range
```

check:

```text
CIDR syntax
allowed CIDR policy
overlap standards
```

Nếu organizational policy deny:

```text
do not bypass
```

Escalate.

---

# 27. Phase 5 — Create Public Subnet

```bash
PUBLIC_SUBNET_ID=$(
  aws ec2 create-subnet \
    --vpc-id "$VPC_ID" \
    --cidr-block "$PUBLIC_CIDR" \
    --availability-zone "$AZ" \
    --tag-specifications \
      "ResourceType=subnet,Tags=[
        {Key=Name,Value=day4-public-subnet},
        {Key=Environment,Value=Lab},
        {Key=Owner,Value=$OWNER},
        {Key=Ticket,Value=$TICKET}
      ]" \
    --query 'Subnet.SubnetId' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)
```

Expected:

```text
subnet-xxxxxxxxxxxxxxxxx
```

---

# 28. Public IPv4 Assignment

Đối với lab SSH trực tiếp:

```bash
aws ec2 modify-subnet-attribute \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --map-public-ip-on-launch \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

AWS documentation xác nhận nondefault subnets không mặc định auto-assign public IPv4; attribute này kiểm soát behavior đó. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/working-with-ip-addresses.html?utm_source=chatgpt.com)

---

# 29. Create Private Subnet

```bash
PRIVATE_SUBNET_ID=$(
  aws ec2 create-subnet \
    --vpc-id "$VPC_ID" \
    --cidr-block "$PRIVATE_CIDR" \
    --availability-zone "$AZ" \
    --tag-specifications \
      "ResourceType=subnet,Tags=[
        {Key=Name,Value=day4-private-subnet},
        {Key=Environment,Value=Lab},
        {Key=Owner,Value=$OWNER},
        {Key=Ticket,Value=$TICKET}
      ]" \
    --query 'Subnet.SubnetId' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)
```

Không enable public IPv4 auto assignment cho private subnet.

---

# 30. Verify Subnets

```bash
aws ec2 describe-subnets \
  --subnet-ids "$PUBLIC_SUBNET_ID" "$PRIVATE_SUBNET_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  --query 'Subnets[*].{
    ID:SubnetId,
    CIDR:CidrBlock,
    AZ:AvailabilityZone,
    PublicIP:MapPublicIpOnLaunch,
    VPC:VpcId
  }' \
  --output table
```

Expected conceptual:

```text
Public:
10.40.10.0/24
MapPublicIpOnLaunch=True

Private:
10.40.20.0/24
MapPublicIpOnLaunch=False
```

Capture full JSON separately:

```bash
aws ec2 describe-subnets \
  --subnet-ids "$PUBLIC_SUBNET_ID" "$PRIVATE_SUBNET_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  > evidence/D4-E005_subnets.json
```

---

# 31. Phase 6 — Internet Gateway

Create:

```bash
IGW_ID=$(
  aws ec2 create-internet-gateway \
    --tag-specifications \
      "ResourceType=internet-gateway,Tags=[
        {Key=Name,Value=day4-igw},
        {Key=Environment,Value=Lab},
        {Key=Owner,Value=$OWNER},
        {Key=Ticket,Value=$TICKET}
      ]" \
    --query 'InternetGateway.InternetGatewayId' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)
```

Attach:

```bash
aws ec2 attach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

IGW phải được attach vào VPC và route table phải reference nó trước khi public subnet có Internet path. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html?utm_source=chatgpt.com)

---

# 32. Verify IGW

```bash
aws ec2 describe-internet-gateways \
  --internet-gateway-ids "$IGW_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Expected attachment:

```text
VpcId = $VPC_ID
State = available
```

---

# 33. Phase 7 — Public Route Table

Create:

```bash
PUBLIC_RTB_ID=$(
  aws ec2 create-route-table \
    --vpc-id "$VPC_ID" \
    --tag-specifications \
      "ResourceType=route-table,Tags=[
        {Key=Name,Value=day4-public-rt},
        {Key=Environment,Value=Lab},
        {Key=Owner,Value=$OWNER},
        {Key=Ticket,Value=$TICKET}
      ]" \
    --query 'RouteTable.RouteTableId' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)
```

---

# 34. Add Default Internet Route

```bash
aws ec2 create-route \
  --route-table-id "$PUBLIC_RTB_ID" \
  --destination-cidr-block "0.0.0.0/0" \
  --gateway-id "$IGW_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Expected:

```text
true
```

---

# 35. Associate Public Subnet

```bash
PUBLIC_ASSOC_ID=$(
  aws ec2 associate-route-table \
    --route-table-id "$PUBLIC_RTB_ID" \
    --subnet-id "$PUBLIC_SUBNET_ID" \
    --query 'AssociationId' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)
```

---

# 36. Create Private Route Table

```bash
PRIVATE_RTB_ID=$(
  aws ec2 create-route-table \
    --vpc-id "$VPC_ID" \
    --tag-specifications \
      "ResourceType=route-table,Tags=[
        {Key=Name,Value=day4-private-rt},
        {Key=Environment,Value=Lab},
        {Key=Owner,Value=$OWNER},
        {Key=Ticket,Value=$TICKET}
      ]" \
    --query 'RouteTable.RouteTableId' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)
```

Associate:

```bash
PRIVATE_ASSOC_ID=$(
  aws ec2 associate-route-table \
    --route-table-id "$PRIVATE_RTB_ID" \
    --subnet-id "$PRIVATE_SUBNET_ID" \
    --query 'AssociationId' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)
```

Base lab **không thêm**:

```text
0.0.0.0/0
```

vào private RT.

---

# 37. Verify Routes

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PUBLIC_RTB_ID" "$PRIVATE_RTB_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Public expected:

```text
10.40.0.0/16 → local
0.0.0.0/0    → igw-...
```

Private expected:

```text
10.40.0.0/16 → local
```

Mỗi subnet phải associated với một route table; nếu không explicit associate custom table, nó dùng main route table. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html?utm_source=chatgpt.com)

Capture:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PUBLIC_RTB_ID" "$PRIVATE_RTB_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  > evidence/D4-E006_routes.json
```

---

# 38. Checkpoint — Chứng minh Public/Private bằng Configuration

Bạn phải có khả năng chứng minh:

```text
day4-public-subnet
    ↓
associated route table
    ↓
0.0.0.0/0
    ↓
Internet Gateway
```

và:

```text
day4-private-subnet
    ↓
associated route table
    ↓
NO direct IGW route
```

Tên `public-subnet` không phải evidence.

Route table mới là evidence.

---

# 39. Phase 8 — Network ACL Inspection

Mục tiêu Day 4 là:

```text
NACL checks
```

không bắt buộc tự thiết kế production-like custom NACL.

Xem NACL trong VPC:

```bash
aws ec2 describe-network-acls \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Capture:

```bash
aws ec2 describe-network-acls \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  > evidence/D4-E008_nacl.json
```

---

# 40. NACL Verification Questions

Từ output phải xác định:

```text
NACL ID?
Default/custom?
Subnet association?
Inbound rules?
Outbound rules?
Rule order?
Allow/Deny?
```

Network ACL đánh giá rule từ rule number thấp lên cao và dừng khi có match; NACL là stateless nên response traffic cũng cần được allow phù hợp. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/create-network-acl.html?utm_source=chatgpt.com)

---

# 41. Không Modify Default NACL tùy tiện

Day 4 base lab ưu tiên:

```text
READ-ONLY NACL inspection
```

Không cố tạo complex ephemeral-port rule chỉ để “thực hành firewall”.

Reason:

```text
Incorrect NACL
→ SSH loss
→ HTTP loss
→ package repository failure
→ difficult troubleshooting
```

NACL failure injection sẽ là optional/instructor-controlled.

---

# 42. Phase 9 — Security Group

Create:

```bash
SG_ID=$(
  aws ec2 create-security-group \
    --group-name "day4-ec2-sg" \
    --description "Day 4 lab EC2 security group" \
    --vpc-id "$VPC_ID" \
    --query 'GroupId' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)

echo "$SG_ID"
```

Tag:

```bash
aws ec2 create-tags \
  --resources "$SG_ID" \
  --tags \
    Key=Name,Value=day4-ec2-sg \
    Key=Environment,Value=Lab \
    Key=Owner,Value="$OWNER" \
    Key=Ticket,Value="$TICKET" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

---

# 43. Restrict SSH

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr "$ADMIN_CIDR" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Không dùng:

```text
0.0.0.0/0
```

cho SSH nếu lab không có explicit justification.

AWS cũng khuyến nghị SSH/RDP ingress chỉ được mở cho specific IP ranges cần thiết. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html?utm_source=chatgpt.com)

---

# 44. Allow HTTP từ Admin Source

Bootstrap service sẽ dùng Apache/httpd.

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr "$ADMIN_CIDR" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Ta giới hạn HTTP vào admin source để tránh expose lab web server ra toàn Internet.

---

# 45. Security Group Behavior

New custom SG ban đầu không có inbound rules; outbound thường bắt đầu với allow-all outbound rule. Security Group là stateful và chỉ có allow rules, khác NACL. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html?utm_source=chatgpt.com)

Đối với Day 4:

```text
Inbound 22 → ADMIN_CIDR
Inbound 80 → ADMIN_CIDR
```

Default outbound được giữ để bootstrap có thể:

```text
DNS
package repository
software download
```

Trong production, egress policy phải theo organizational standard.

---

# 46. Verify Security Group

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  > evidence/D4-E007_sg.json
```

Review:

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  --query 'SecurityGroups[0].{
    GroupId:GroupId,
    Inbound:IpPermissions,
    Outbound:IpPermissionsEgress
  }'
```

---

# 47. Phase 10 — EC2 Key Pair

Hai lựa chọn hợp lệ:

```text
Existing approved key pair
```

hoặc:

```text
Generate local key pair
and import public key
```

Cách thứ hai có ưu điểm private key không bao giờ được AWS trả về workstation.

---

# 48. Generate Local SSH Key

Ví dụ:

```bash
ssh-keygen \
  -t ed25519 \
  -f ~/.ssh/day4-ec2
```

Tool sẽ hỏi passphrase.

Trong môi trường enterprise, dùng passphrase theo approved policy.

Files:

```text
~/.ssh/day4-ec2
~/.ssh/day4-ec2.pub
```

Private:

```text
~/.ssh/day4-ec2
```

Public:

```text
~/.ssh/day4-ec2.pub
```

---

# 49. Import Public Key

```bash
export KEY_NAME="day4-ec2-key"
export KEY_FILE="$HOME/.ssh/day4-ec2"
```

Import:

```bash
aws ec2 import-key-pair \
  --key-name "$KEY_NAME" \
  --public-key-material "fileb://${KEY_FILE}.pub" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

AWS lưu public key; private key phải do operator bảo vệ. Nếu private key mất, EC2 không có copy để download lại. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html?utm_source=chatgpt.com)

---

# 50. Private Key Permissions

```bash
chmod 400 "$KEY_FILE"
```

AWS SSH prerequisites cũng yêu cầu restrictive permissions cho private key trên Linux/macOS clients. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connection-prereqs-general.html?utm_source=chatgpt.com)

Check:

```bash
ls -l "$KEY_FILE"
```

Expected kiểu:

```text
-r-------- ...
```

---

# 51. Không Submit Private Key

Evidence chỉ ghi:

```text
KeyPairName=day4-ec2-key
Fingerprint=<non-secret fingerprint if needed>
PrivateKeySubmitted=NO
```

Không:

```bash
cat ~/.ssh/day4-ec2 > evidence/key.txt
```

---

# 52. Phase 11 — IAM Role / Instance Profile

EC2 phải launch với IAM role như syllabus yêu cầu.

Learner không nhất thiết cần permission để tự tạo IAM role.

Preferred:

```text
Instructor provisions:
Day4Ec2Role
Day4Ec2InstanceProfile
```

Role có **minimum permissions cần cho lab**.

Base lab thậm chí không cần EC2 application gọi thêm AWS services.

Mục tiêu Day 4 là hiểu:

```text
EC2
  ↓
Instance Profile
  ↓
IAM Role
  ↓
Temporary Credentials
```

AWS khuyến nghị workload dùng IAM roles/temporary credentials thay vì hard-coded access keys. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html?utm_source=chatgpt.com)

---

# 53. Verify Instance Profile Before Launch

Giả sử:

```bash
export INSTANCE_PROFILE="Day4Ec2InstanceProfile"
```

Nếu learner có permission:

```bash
aws iam get-instance-profile \
  --instance-profile-name "$INSTANCE_PROFILE" \
  --profile "$PROFILE"
```

Expected:

```text
InstanceProfile
  → Role
      → Day4Ec2Role
```

Không cần có broad application permission.

---

# 54. Phase 12 — Approved AMI Verification

Không launch random marketplace/public image.

```bash
aws ec2 describe-images \
  --image-ids "$AMI_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  --query 'Images[0].{
    ImageId:ImageId,
    Name:Name,
    State:State,
    OwnerId:OwnerId,
    Architecture:Architecture,
    PlatformDetails:PlatformDetails,
    RootDeviceType:RootDeviceType
  }'
```

Review:

```text
ImageId
Name
OwnerId
State
Architecture
PlatformDetails
```

AMI phải ở đúng Region và compatible với selected instance type. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ami-lifecycle.html?utm_source=chatgpt.com)

---

# 55. RHEL vs Other Linux AMI

Preferred:

```text
approved RHEL AMI
```

vì Day 1–2 đã học RHEL.

Nếu lab không có RHEL license/image và instructor dùng:

```text
Amazon Linux
```

thì evidence phải ghi:

```text
Hands-on OS = Amazon Linux
RHEL-specific execution = not performed
```

Không được ghi:

```text
RHEL hands-on completed
```

khi thực tế không chạy RHEL.

---

# 56. Phase 13 — Bootstrap Script

Tạo:

```bash
cat > scripts/day4-user-data.sh <<'EOF'
#!/bin/bash
set -euo pipefail

echo "=== Day 4 bootstrap start ==="
date -u

dnf -y install httpd

cat > /var/www/html/index.html <<'HTML'
<!doctype html>
<html>
<head><title>AWS Day 4 Lab</title></head>
<body>
<h1>AWS Day 4 Lab</h1>
<p>Bootstrap completed successfully.</p>
</body>
</html>
HTML

systemctl enable --now httpd

echo "=== httpd state ==="
systemctl is-enabled httpd
systemctl is-active httpd

echo "=== listening sockets ==="
ss -lntp

echo "=== local HTTP check ==="
curl --fail --silent --show-error http://127.0.0.1/

echo "=== Day 4 bootstrap end ==="
date -u
EOF
```

---

# 57. Giải thích Bootstrap Script

```bash
#!/bin/bash
```

chọn Bash interpreter.

```bash
set -euo pipefail
```

giúp script fail rõ hơn:

```text
-e
stop khi command fail

-u
error khi dùng undefined variable

pipefail
pipeline fail nếu component fail
```

Không dùng:

```bash
set -x
```

một cách vô thức trong production bootstrap vì có thể ghi sensitive command arguments vào logs.

---

# 58. `dnf -y install httpd`

```bash
dnf -y install httpd
```

phù hợp RHEL-family systems có repository/configuration thích hợp.

Nếu fail:

```text
do not immediately edit security
```

investigate:

```text
DNS
route
internet path
repository
entitlement/subscription/RHUI
package metadata
```

---

# 59. `systemctl enable --now`

```bash
systemctl enable --now httpd
```

nghĩa:

```text
enable → start on future boot
--now  → also start immediately
```

Verify riêng biệt:

```bash
systemctl is-enabled httpd
systemctl is-active httpd
```

---

# 60. User Data Security

Không đặt:

```bash
export AWS_SECRET_ACCESS_KEY=...
export DB_PASSWORD=...
```

trong script.

User Data có thể được retrieve từ instance metadata, vì vậy nó không phải secret store. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html?utm_source=chatgpt.com)

---

# 61. User Data Execution Behavior

AWS Linux User Data có thể chạy shell scripts/cloud-init directives; scripts mặc định chạy trong initial launch context và output có thể được xem ở:

```text
/var/log/cloud-init-output.log
```

AWS docs cũng lưu ý User Data script chạy với root privileges, vì vậy không cần `sudo` bên trong script. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html?icmpid=docs_ec2_console&utm_source=chatgpt.com)

---

# 62. Phase 14 — Mandatory Tag Dry-Run Test

Trước launch thật, kiểm tra guardrail.

Giả sử policy yêu cầu:

```text
Environment=Lab
Owner=<learner>
```

Ta thử **missing tags** bằng `--dry-run`.

Command phải dùng resource IDs đã chuẩn bị:

```bash
aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --security-group-ids "$SG_ID" \
  --key-name "$KEY_NAME" \
  --iam-instance-profile "Name=$INSTANCE_PROFILE" \
  --metadata-options \
    "HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=1" \
  --dry-run \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Nếu mandatory-tag IAM guardrail đúng và tags là bắt buộc:

```text
UnauthorizedOperation
```

là expected.

---

# 63. Positive Tag Dry Run

Bây giờ thêm required tags:

```bash
aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --security-group-ids "$SG_ID" \
  --key-name "$KEY_NAME" \
  --iam-instance-profile "Name=$INSTANCE_PROFILE" \
  --metadata-options \
    "HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=1" \
  --tag-specifications \
    "ResourceType=instance,Tags=[
      {Key=Name,Value=day4-rhel},
      {Key=Environment,Value=Lab},
      {Key=Owner,Value=$OWNER},
      {Key=Ticket,Value=$TICKET}
    ]" \
    "ResourceType=volume,Tags=[
      {Key=Environment,Value=Lab},
      {Key=Owner,Value=$OWNER},
      {Key=Ticket,Value=$TICKET}
    ]" \
  --dry-run \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Expected:

```text
DryRunOperation
```

Meaning:

````text
request would have been authorized
but no instance was created
``` :chatgpt-content-reference{index="24"}


---

# 64. Guardrail Acceptance

| Test | Expected | PASS |
|---|---|---|
| Missing mandatory tags | `UnauthorizedOperation` | Yes |
| Required tags supplied | `DryRunOperation` | Yes |
| Missing tags unexpectedly allowed | `DryRunOperation` | No |
| Correct tags still unauthorized | `UnauthorizedOperation` | No; investigate |

Nếu organizational sandbox không có mandatory-tag IAM guardrail, không giả vờ đã enforce.

Ghi:

```text
Tag guardrail = manual validation only
IAM enforcement = unavailable in lab
Gap = OPEN
````

Điều này phù hợp philosophy của syllabus: không claim capability không thực sự được lab.

---

# 65. Phase 15 — Launch EC2 thật

Sau final pre-check:

```bash
aws sts get-caller-identity \
  --profile "$PROFILE"
```

Launch:

```bash
INSTANCE_ID=$(
  aws ec2 run-instances \
    --image-id "$AMI_ID" \
    --instance-type "$INSTANCE_TYPE" \
    --subnet-id "$PUBLIC_SUBNET_ID" \
    --security-group-ids "$SG_ID" \
    --key-name "$KEY_NAME" \
    --iam-instance-profile "Name=$INSTANCE_PROFILE" \
    --associate-public-ip-address \
    --user-data file://scripts/day4-user-data.sh \
    --metadata-options \
      "HttpTokens=required,HttpEndpoint=enabled,HttpPutResponseHopLimit=1,InstanceMetadataTags=disabled" \
    --tag-specifications \
      "ResourceType=instance,Tags=[
        {Key=Name,Value=day4-rhel},
        {Key=Environment,Value=Lab},
        {Key=Owner,Value=$OWNER},
        {Key=Ticket,Value=$TICKET}
      ]" \
      "ResourceType=volume,Tags=[
        {Key=Environment,Value=Lab},
        {Key=Owner,Value=$OWNER},
        {Key=Ticket,Value=$TICKET}
      ]" \
    --query 'Instances[0].InstanceId' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)

echo "$INSTANCE_ID"
```

Expected:

```text
i-xxxxxxxxxxxxxxxxx
```

Current AWS CLI supports `--metadata-options`, `--iam-instance-profile`, `--user-data`, tag specifications và `--dry-run` trên `run-instances`. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/ec2/run-instances.html?utm_source=chatgpt.com)

---

# 66. Metadata Configuration Giải thích

```text
HttpTokens=required
```

bắt buộc IMDSv2.

```text
HttpEndpoint=enabled
```

IMDS endpoint vẫn available.

```text
HttpPutResponseHopLimit=1
```

phù hợp simple non-container lab.

```text
InstanceMetadataTags=disabled
```

không expose instance tags qua IMDS.

AWS cho phép cấu hình metadata ở instance/account/AMI levels, với precedence rules; container workloads có thể cần hop limit khác, vì vậy production phải theo deployed workload baseline. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-options.html?utm_source=chatgpt.com)

---

# 67. Wait for EC2 Running

```bash
aws ec2 wait instance-running \
  --instance-ids "$INSTANCE_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Sau đó inspect:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

---

# 68. Extract Runtime Details

```bash
PUBLIC_IP=$(
  aws ec2 describe-instances \
    --instance-ids "$INSTANCE_ID" \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)

PRIVATE_IP=$(
  aws ec2 describe-instances \
    --instance-ids "$INSTANCE_ID" \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' \
    --output text \
    --profile "$PROFILE" \
    --region "$AWS_REGION"
)

echo "INSTANCE_ID=$INSTANCE_ID"
echo "PUBLIC_IP=$PUBLIC_IP"
echo "PRIVATE_IP=$PRIVATE_IP"
```

IP addresses không phải credentials, nhưng evidence vẫn phải được classified phù hợp organizational policy.

---

# 69. EC2 Inventory Evidence

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  > evidence/D4-E009_ec2.json
```

Bạn phải verify:

```text
InstanceId
ImageId
InstanceType
State
SubnetId
VpcId
PrivateIpAddress
PublicIpAddress
SecurityGroups
IamInstanceProfile
MetadataOptions
Tags
```

---

# 70. EC2 Status Checks

Running chưa đồng nghĩa healthy.

```bash
aws ec2 describe-instance-status \
  --instance-ids "$INSTANCE_ID" \
  --include-all-instances \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

EC2 hiện phân biệt system, instance, attached EBS và optional application status checks. System status thiên về underlying AWS infrastructure; instance status thiên về guest/network configuration của instance. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html?utm_source=chatgpt.com)

Base acceptance:

```text
SystemStatus = ok
InstanceStatus = ok
```

sau khi startup ổn định.

---

# 71. Phase 16 — Verify Metadata Options từ Control Plane

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].MetadataOptions' \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Expected:

```json
{
  "State": "applied",
  "HttpTokens": "required",
  "HttpPutResponseHopLimit": 1,
  "HttpEndpoint": "enabled",
  "InstanceMetadataTags": "disabled"
}
```

Exact additional fields có thể khác theo platform/version.

Capture:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].MetadataOptions' \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  > evidence/D4-E010_metadata-options.json
```

---

# 72. Phase 17 — SSH Prerequisite Check

Trước SSH, kiểm tra chain:

```text
Instance = running?
Status checks = OK?
Public IPv4 exists?
Public subnet route = IGW?
Security Group TCP/22 = ADMIN_CIDR?
NACL = permits?
Correct key?
Correct OS username?
```

AWS SSH guidance cũng yêu cầu instance status checks, SG SSH access, correct private key và đúng instance details trước khi connect. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/connect-to-linux-instance.html?utm_source=chatgpt.com)

---

# 73. Determine SSH Username

Không đoán.

Dựa vào approved AMI documentation.

RHEL-family thường gặp:

```text
ec2-user
```

Amazon Linux:

```text
ec2-user
```

Ubuntu:

```text
ubuntu
```

AWS lưu ý username phụ thuộc AMI; `Permission denied (publickey)` thường phải check username và key trước. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstancesConnecting.html?utm_source=chatgpt.com)

Define:

```bash
export SSH_USER="<approved-ami-user>"
```

---

# 74. SSH

```bash
ssh \
  -i "$KEY_FILE" \
  "${SSH_USER}@${PUBLIC_IP}"
```

Ở lần đầu, SSH host fingerprint prompt xuất hiện.

Không blindly type `yes` trong production.

Lab nên kiểm tra source/fingerprint theo approved workflow nếu có.

---

# 75. Verify Host Identity Sau Login

Trên EC2:

```bash
whoami
hostname
uname -a
cat /etc/os-release
```

Purpose:

```text
whoami
→ đúng Linux user?

hostname
→ đúng host?

os-release
→ đúng distribution?

uname
→ đúng kernel/architecture?
```

---

# 76. Verify Network trên Guest

```bash
ip addr
```

```bash
ip route
```

Expected:

```text
private IP belongs to 10.40.10.0/24
```

Default route thường thông qua VPC router address trong subnet.

Do not expect guest OS route table to show:

```text
Internet Gateway
```

IGW là VPC routing object, không phải Linux gateway trực tiếp hiển thị như AWS object trong guest.

---

# 77. Verify DNS

```bash
getent hosts example.com
```

hoặc approved test hostname.

Nếu fail:

```text
don't immediately blame Security Group
```

Check:

```text
DNS
VPC DNS
network
resolver configuration
```

---

# 78. Verify `httpd`

```bash
systemctl status httpd --no-pager
```

```bash
systemctl is-active httpd
```

Expected:

```text
active
```

```bash
systemctl is-enabled httpd
```

Expected:

```text
enabled
```

---

# 79. Verify Listening Port

```bash
sudo ss -lntp
```

Expected entry:

```text
*:80
```

hoặc equivalent address.

Nếu `httpd` active nhưng không thấy listener:

```text
service configuration issue
```

---

# 80. Local HTTP Test

```bash
curl --fail --silent --show-error http://127.0.0.1/
```

Expected:

```html
<h1>AWS Day 4 Lab</h1>
...
```

Đây là crucial isolation test.

Nếu local curl thành công nhưng external curl fail:

```text
Application likely healthy
Network path likely failing
```

---

# 81. External HTTP Test

Từ workstation:

```bash
curl \
  --connect-timeout 5 \
  --max-time 10 \
  "http://${PUBLIC_IP}/"
```

Expected:

```text
AWS Day 4 Lab
Bootstrap completed successfully.
```

Bây giờ ta đã chứng minh:

```text
Application
+
Guest OS
+
Security Group
+
Route
+
IGW
+
Public addressing
+
Client path
```

đều đủ để HTTP hoạt động.

---

# 82. Phase 18 — Verify User Data / Provisioning Logs

SSH vào EC2:

```bash
sudo tail -n 100 /var/log/cloud-init-output.log
```

AWS xác định `/var/log/cloud-init-output.log` là primary output log hữu ích để debug Linux User Data/cloud-init bootstrap. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html?icmpid=docs_ec2_console&utm_source=chatgpt.com)

Expected có:

```text
=== Day 4 bootstrap start ===
...
active
...
AWS Day 4 Lab
...
=== Day 4 bootstrap end ===
```

---

# 83. Check `cloud-init`

Nếu image hỗ trợ:

```bash
sudo cloud-init status --long
```

Possible:

```text
status: done
```

Exact output phụ thuộc distro/cloud-init version.

Nếu không có `cloud-init` command nhưng bootstrap mechanism khác được image sử dụng, document difference.

---

# 84. Evidence Provisioning Log

Không blindly copy toàn log trước khi review sensitive material.

Trong lab script hiện tại không có secrets.

Capture bounded output:

```bash
sudo tail -n 200 /var/log/cloud-init-output.log
```

Save từ workstation bằng approved method hoặc copy sanitized output.

Evidence phải có:

```text
instance ID
timestamp
log path
command
bounded output
```

---

# 85. Phase 19 — Verify IMDSv2 Inside Instance

Trên EC2:

```bash
TOKEN=$(
  curl --fail --silent --show-error \
    -X PUT \
    "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"
)
```

**Không echo token vào evidence.**

Token bản thân là security-sensitive session material.

---

# 86. Retrieve Instance ID bằng IMDSv2

```bash
curl \
  --fail \
  --silent \
  --show-error \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  "http://169.254.169.254/latest/meta-data/instance-id"
```

Expected:

```text
i-xxxxxxxxxxxxxxxxx
```

Nó phải match:

```bash
echo "$INSTANCE_ID"
```

AWS IMDSv2 sử dụng `PUT /latest/api/token` để obtain token, rồi token được cung cấp trong metadata GET requests. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html?utm_source=chatgpt.com)

---

# 87. Verify AMI ID qua Metadata

```bash
curl \
  --fail \
  --silent \
  --show-error \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  "http://169.254.169.254/latest/meta-data/ami-id"
```

Compare với:

```text
$AMI_ID
```

Đây là cross-layer verification:

```text
AWS control plane
vs
instance self-observation
```

---

# 88. Verify IAM Role Name qua IMDS

Có thể query:

```bash
curl \
  --fail \
  --silent \
  --show-error \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
```

Expected:

```text
Day4Ec2Role
```

**Dừng tại role name.**

Không submit endpoint response:

```text
/iam/security-credentials/Day4Ec2Role
```

vì response có temporary credentials.

---

# 89. Negative IMDSv1 Test

Bây giờ thử request không token:

```bash
curl \
  --silent \
  --output /dev/null \
  --write-out '%{http_code}\n' \
  "http://169.254.169.254/latest/meta-data/instance-id"
```

Expected khi `HttpTokens=required`:

```text
401
```

AWS xác nhận token-required configuration trả `401 Unauthorized` đối với metadata request không có valid token. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html?utm_source=chatgpt.com)

Interpretation:

```text
IMDSv2 request with token → PASS
IMDSv1-style request without token → DENIED
```

Đây là positive + negative security verification.

---

# 90. IMDS Evidence

Không lưu token.

Evidence chỉ cần:

```text
Instance ID
HttpTokens=required
HttpEndpoint=enabled
IMDSv2 metadata query = success
Tokenless query HTTP status = 401
Timestamp
```

Ví dụ:

```text
D4-E010
Instance: i-...
HttpTokens: required
IMDSv2 GET instance-id: PASS
Tokenless GET result: HTTP 401
Token captured in evidence: NO
```

---

# 91. Phase 20 — Allowed Connectivity Test

Allowed path:

```text
Admin Workstation
       │
       │ TCP/80
       ▼
Internet
       │
       ▼
IGW
       │
       ▼
Public Subnet
       │
       ▼
NACL
       │
       ▼
Security Group
       │
       ▼
EC2
       │
       ▼
httpd
```

Test:

```bash
curl \
  --connect-timeout 5 \
  --max-time 10 \
  "http://${PUBLIC_IP}/"
```

Expected:

```text
HTTP connection succeeds
```

---

# 92. Capture Allowed Test

```bash
{
  date -u +"%Y-%m-%dT%H:%M:%SZ"
  curl \
    --connect-timeout 5 \
    --max-time 10 \
    --write-out '\nHTTP_CODE=%{http_code}\n' \
    "http://${PUBLIC_IP}/"
} | tee evidence/D4-E012_http-allowed.txt
```

Expected:

```text
HTTP_CODE=200
```

assuming Apache response config như lab.

---

# 93. Phase 21 — Denied Connectivity Test

Ta cần chứng minh denied behavior mà không phá SSH access.

Do đó ta dùng HTTP/80.

Trước test:

```text
local HTTP = healthy
external HTTP = healthy
```

Sau đó temporarily remove **HTTP rule only**.

Không remove TCP/22.

---

# 94. Capture SG Baseline Before Change

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  > evidence/D4-E012_sg-before-negative-test.json
```

Change reference:

```text
CHG-D4-HTTP-NEGATIVE-TEST
```

---

# 95. Revoke HTTP Rule

```bash
aws ec2 revoke-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr "$ADMIN_CIDR" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Security Group changes apply automatically tới associated resources. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-security-group-rules.html?utm_source=chatgpt.com)

---

# 96. Verify Application vẫn Healthy

Trên instance:

```bash
curl \
  --fail \
  --silent \
  --show-error \
  http://127.0.0.1/
```

Expected:

```text
SUCCESS
```

Điều này chứng minh:

```text
httpd không bị stop
```

---

# 97. External Denied Test

Từ workstation:

```bash
curl \
  --connect-timeout 5 \
  --max-time 10 \
  "http://${PUBLIC_IP}/"
```

Expected:

```text
timeout / unable to connect
```

Exact client error có thể khác.

Quan trọng:

```text
Expected = FAIL
Actual = FAIL
Security test = PASS
```

---

# 98. Troubleshooting Reasoning

Ta biết:

```text
local HTTP = PASS
```

nhưng:

```text
external HTTP = FAIL
```

Ta cũng biết vừa change:

```text
SG inbound port 80 removed
```

Root cause:

```text
Security Group intentionally blocks inbound TCP/80
```

Không phải:

```text
httpd failure
```

---

# 99. Rollback HTTP Rule

Ngay sau negative test:

```bash
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr "$ADMIN_CIDR" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

---

# 100. Verify Rollback

```bash
curl \
  --connect-timeout 5 \
  --max-time 10 \
  "http://${PUBLIC_IP}/"
```

Expected:

```text
SUCCESS again
```

Capture SG after rollback:

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION" \
  > evidence/D4-E012_sg-after-rollback.json
```

Acceptance:

```text
Baseline      → PASS
Failure test  → expected FAIL
Rollback      → PASS
```

---

# 101. Phase 22 — Network Path Inspection

Network troubleshooting không chỉ là `ping`.

Collect:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].{
    PrivateIp:PrivateIpAddress,
    PublicIp:PublicIpAddress,
    SubnetId:SubnetId,
    VpcId:VpcId,
    SG:SecurityGroups
  }' \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Then route:

```bash
aws ec2 describe-route-tables \
  --filters \
    "Name=association.subnet-id,Values=$PUBLIC_SUBNET_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Then NACL:

```bash
aws ec2 describe-network-acls \
  --filters \
    "Name=association.subnet-id,Values=$PUBLIC_SUBNET_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Then SG:

```bash
aws ec2 describe-security-groups \
  --group-ids "$SG_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

---

# 102. Guest-side Network Inspection

```bash
ip addr
ip route
ss -lntp
```

Application:

```bash
curl http://127.0.0.1/
```

Then external:

```bash
curl "http://${PUBLIC_IP}/"
```

Tư duy:

```text
Control Plane
+
Guest Network
+
Service
+
End-to-End test
```

---

# 103. Phase 23 — Security Group vs NACL Verification

Bạn phải chứng minh được:

```text
Security Group = stateful
NACL           = stateless
```

Security Groups aggregate allow rules và automatically permit response traffic của allowed connections theo connection tracking; NACL response traffic phải được allow phù hợp. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/nacl-examples.html?utm_source=chatgpt.com)

Điều này giải thích vì sao:

```text
SG seems correct
```

vẫn chưa đủ để conclude network healthy.

---

# 104. Phase 24 — Validate Private Subnet Concept

Không cần EC2 thứ hai.

Inspect private route table:

```bash
aws ec2 describe-route-tables \
  --route-table-ids "$PRIVATE_RTB_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Expected:

```text
10.40.0.0/16 → local
```

Không có:

```text
0.0.0.0/0 → IGW
```

Acceptance statement:

```text
Private subnet has no direct internet gateway route.
```

Đó là sufficient Day 4 base-lab evidence.

---

# 105. Optional NAT Gateway Exercise

Chỉ thực hiện khi:

```text
sandbox approved
cost approved
instructor explicitly requires it
```

Conceptual flow:

```text
Private RT
0.0.0.0/0
     │
     ▼
NAT Gateway
     │
     ▼
Internet
```

Trong traditional zonal deployment, NAT Gateway cho private resources outbound connectivity và unsolicited inbound connections không đi ngược vào private EC2 theo cách Internet-exposed resource. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat.html?utm_source=chatgpt.com)

AWS troubleshooting guidance cho traditional NAT architecture nhấn mạnh NAT status, public-subnet IGW route, private-subnet route tới NAT, SG outbound và NACL rules. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateway-troubleshooting.html?utm_source=chatgpt.com)

Không bắt buộc provision cho acceptance Day 4.

---

# 106. Phase 25 — Verify EC2 IAM Association

Control plane:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].IamInstanceProfile' \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Expected:

```text
ARN
ID
```

Guest:

```bash
TOKEN=$(
  curl --fail --silent \
    -X PUT \
    "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 300"
)

curl \
  --fail \
  --silent \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  "http://169.254.169.254/latest/meta-data/iam/security-credentials/"
```

Expected role name.

---

# 107. Do Not Extract Role Credentials for Evidence

Không chạy:

```text
GET /iam/security-credentials/<role>
```

rồi:

```text
copy JSON to ticket
```

Response có thể chứa:

```text
AccessKeyId
SecretAccessKey
Token
Expiration
```

Syllabus explicitly yêu cầu temporary credentials/tokens không xuất hiện trong submitted logs. Infrastructure_Operations_14_Da…

---

# 108. Phase 26 — EC2 Lifecycle Operational Check

Xem state:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Expected:

```text
running
```

---

# 109. Reboot vs Stop/Start vs Terminate

Trong lab troubleshooting:

```text
reboot
```

không phải bước đầu tiên.

Trước reboot:

```text
collect logs
capture service state
capture network state
```

Stop/start có consequences khác reboot, bao gồm underlying host migration trong nhiều trường hợp và auto-assigned public IPv4 có thể thay đổi. Terminate là destructive lifecycle transition. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/how-ec2-instance-stop-start-works.html?utm_source=chatgpt.com)

---

# 110. Basic Health Collection

Trên Linux:

```bash
date -u
hostname
uptime
whoami
```

```bash
free -h
```

```bash
df -h
df -i
```

```bash
ip addr
ip route
```

```bash
systemctl --failed
```

```bash
systemctl status httpd --no-pager
```

```bash
ss -lntp
```

```bash
sudo journalctl -u httpd --since "-15 minutes" --no-pager
```

Đây là bridge giữa Day 2 Linux operations và Day 4 AWS operations.

---

# 111. Failure Injection Framework

Mỗi failure phải theo:

```text
Baseline PASS
     ↓
Introduce one controlled fault
     ↓
Observe symptom
     ↓
Collect evidence
     ↓
Form hypothesis
     ↓
Read-only checks
     ↓
Identify root cause
     ↓
Authorized repair
     ↓
Retest
     ↓
Rollback verification
```

Không inject nhiều faults đồng thời ở lần đầu.

Nếu bạn thay cả:

```text
Route
SG
service
```

thì lab mất giá trị troubleshooting.

---

# 112. Failure Injection 1 — Security Group Blocks HTTP

Fault:

```text
remove TCP/80 inbound
```

Symptom:

```text
external curl timeout
```

But:

```text
local curl PASS
```

Root cause domain:

```text
AWS network security
```

Repair:

```text
restore TCP/80 ADMIN_CIDR rule
```

Verification:

```text
external HTTP works again
```

Evidence:

```text
before SG
failure SG
local HTTP result
external fail result
after rollback SG
external recovered result
```

---

# 113. Failure Injection 2 — Stop `httpd`

Trên EC2:

```bash
sudo systemctl stop httpd
```

Expected:

```bash
systemctl is-active httpd
```

returns:

```text
inactive
```

Local:

```bash
curl \
  --connect-timeout 3 \
  http://127.0.0.1/
```

fails.

External also fails.

---

# 114. Diagnose Failure 2

Inspect:

```bash
systemctl status httpd --no-pager
```

```bash
sudo journalctl \
  -u httpd \
  --since "-10 minutes" \
  --no-pager
```

```bash
ss -lntp
```

Reasoning:

```text
SG unchanged
Route unchanged
Instance reachable via SSH
Port 80 not listening
httpd inactive

→ Service layer
```

---

# 115. Recover Failure 2

```bash
sudo systemctl start httpd
```

Verify:

```bash
systemctl is-active httpd
```

```bash
ss -lntp
```

```bash
curl http://127.0.0.1/
```

Then workstation:

```bash
curl "http://${PUBLIC_IP}/"
```

Root cause:

```text
service stopped
```

Không phải network.

---

# 116. Failure Injection 3 — Incorrect Route Association

Chỉ thực hiện trong sandbox và phải giữ SSH recovery plan.

Safer implementation là **không thay public route table trong live learner instance nếu bạn không có out-of-band access**.

Instructor có thể cung cấp fixture:

```text
Public subnet
associated accidentally with private RT
```

Learner phải diagnose bằng:

```bash
aws ec2 describe-route-tables \
  --filters \
    "Name=association.subnet-id,Values=$PUBLIC_SUBNET_ID"
```

Expected finding:

```text
No 0.0.0.0/0 → IGW
```

Root cause:

```text
incorrect route-table association
```

Repair, nếu authorized:

```text
associate correct public RT
```

Do not practice route failure on a production subnet.

---

# 117. Failure Injection 4 — IMDSv1 Attempt

Không cần change configuration.

Instance vẫn có:

```text
HttpTokens=required
```

Attempt:

```bash
curl \
  --silent \
  --output /dev/null \
  --write-out '%{http_code}\n' \
  http://169.254.169.254/latest/meta-data/instance-id
```

Expected:

```text
401
```

Then IMDSv2:

```bash
TOKEN=$(
  curl --fail --silent \
  -X PUT \
  http://169.254.169.254/latest/api/token \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300"
)

curl \
  --fail \
  --silent \
  -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

Expected:

```text
instance ID
```

Security control works correctly. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html?utm_source=chatgpt.com)

---

# 118. Failure Injection 5 — Bad Bootstrap Fixture

Không rebuild working instance ngay.

Instructor can provide a user-data fixture:

```bash
#!/bin/bash
dnf -y install package-that-does-not-exist
systemctl enable --now httpd
```

Expected failure:

```text
package installation error
```

Investigate:

```bash
sudo tail -n 200 /var/log/cloud-init-output.log
```

Then answer:

```text
Did User Data execute?
Which line failed?
Was network working?
Did dnf contact repository?
Did script stop due set -e?
Was httpd ever started?
```

---

# 119. Troubleshooting Scenario — SSH Timeout

Use sequence:

```text
Correct account?
      ↓
Correct Region?
      ↓
Correct instance?
      ↓
State running?
      ↓
Public IPv4?
      ↓
Public subnet?
      ↓
0.0.0.0/0 → IGW?
      ↓
SG TCP/22 from correct ADMIN_CIDR?
      ↓
NACL permits?
      ↓
Corporate/local network permits?
      ↓
Guest sshd?
```

Do **not** start by replacing SSH key.

Timeout occurs before key authentication in many cases.

---

# 120. Troubleshooting Scenario — `Permission denied (publickey)`

Network path likely progressed further.

Check:

```text
Correct username?
Correct key?
Correct AMI?
Public key installed?
Private key file?
sshd config?
authorized_keys?
```

AWS connection troubleshooting specifically recommends checking correct AMI username and correct private key for this symptom. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/TroubleshootingInstancesConnecting.html?utm_source=chatgpt.com)

---

# 121. Troubleshooting Scenario — `Connection refused`

Usually indicates host/network path reached destination enough to receive rejection.

Check:

```bash
sudo ss -lntp | grep ':22'
```

```bash
sudo systemctl status sshd
```

Could also involve local firewall rejection.

Do not assume SG.

SG drop behavior commonly manifests differently from an actively refused socket.

---

# 122. Troubleshooting Scenario — HTTP External Fails but Local Works

Known facts:

```text
httpd = active
port 80 = listening
local curl = success
```

Investigate outside-in:

```text
Security Group
NACL
Route table
IGW
Public IPv4
Client path
```

This is classic Infrastructure Operations layer isolation.

---

# 123. Troubleshooting Scenario — Local HTTP Fails

Focus shifts inside:

```text
httpd state
port listener
configuration
SELinux
filesystem
application logs
```

Commands:

```bash
systemctl status httpd --no-pager
```

```bash
journalctl -u httpd --since "-15 minutes" --no-pager
```

```bash
ss -lntp
```

```bash
ls -lZ /var/www/html/
```

RHEL SELinux knowledge từ Day 2 tiếp tục áp dụng.

---

# 124. Troubleshooting Scenario — `dnf install` Fail trong User Data

Collect:

```bash
sudo tail -n 200 /var/log/cloud-init-output.log
```

```bash
sudo journalctl -b --no-pager
```

```bash
ip route
```

```bash
getent hosts <repository-host-if-known>
```

```bash
dnf repolist
```

Potential domains:

```text
no Internet route
DNS failure
repository issue
subscription/RHUI
package typo
certificate/time problem
proxy requirement
```

Không sửa random.

---

# 125. Troubleshooting Scenario — `AccessDenied`

Capture:

```text
principal
account
action
resource
timestamp
error message
```

Check first:

```bash
aws sts get-caller-identity \
  --profile "$PROFILE"
```

Then reason:

```text
Wrong role?
Missing Allow?
Explicit Deny?
Permissions boundary?
Session policy?
SCP/RCP?
Resource policy?
```

AWS policy evaluation có thể bao gồm nhiều policy types; attached identity policy không nhất thiết phản ánh toàn effective permission. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html?utm_source=chatgpt.com)

---

# 126. Troubleshooting Scenario — Instance Running nhưng Status Check Fail

Check:

```bash
aws ec2 describe-instance-status \
  --instance-ids "$INSTANCE_ID" \
  --include-all-instances \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

`SystemStatus` failure nghiêng về underlying AWS-side infrastructure; `InstanceStatus` failure thường nghiêng về guest OS/startup/network configuration mà customer cần xử lý. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring-system-instance-status-check.html?utm_source=chatgpt.com)

Escalation decision khác nhau.

---

# 127. Troubleshooting Scenario — IMDSv2 App Breaks

Nếu:

```text
HttpTokens=required
```

và legacy software chỉ biết IMDSv1:

```text
application may fail
```

AWS explicitly cảnh báo việc enforce IMDSv2 có thể break software/agents chỉ hỗ trợ IMDSv1. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-options.html?utm_source=chatgpt.com)

Không immediately set:

```text
HttpTokens=optional
```

trong production.

Correct workflow:

```text
identify dependency
collect evidence
update compatible SDK/agent
test
approve change
then enforce secure baseline
```

---

# 128. Evidence Package — Identity

`D4-E001_identity.json`

Nội dung:

```text
Account ID
Principal ARN
Session identity
Timestamp
```

Không có:

```text
access key
secret
token
```

---

# 129. Evidence Package — IAM

`D4-E002_allowed-ec2.json`

chứng minh:

```text
allowed read action
```

`D4-E003_denied-iam.txt`

chứng minh:

```text
expected denied action
```

Record:

```text
Expected=DENY
Actual=DENY
Result=PASS
```

---

# 130. Evidence Package — Governance

Record mandatory tags:

```text
Name
Environment
Owner
Ticket
```

và tag enforcement dry-run outcomes:

```text
missing tag request
→ UnauthorizedOperation

valid tag request
→ DryRunOperation
```

Nếu guardrail không có:

```text
Gap explicitly recorded
```

---

# 131. Evidence Package — Architecture

Architecture artifact:

```text
VPC
├── Public Subnet
│   ├── Public RT
│   ├── IGW route
│   ├── SG
│   └── EC2
└── Private Subnet
    └── Private RT
```

Phải ghi:

```text
VPC ID
Subnet IDs
Route Table IDs
IGW ID
SG ID
NACL ID
EC2 ID
```

Không chỉ screenshot diagram.

---

# 132. Evidence Package — EC2

Evidence record:

```text
Instance ID
AMI ID
Instance Type
State
VPC/Subnet
Private IP
Public IP
Security Group
IAM Instance Profile
Metadata Options
Tags
```

---

# 133. Evidence Package — Linux/RHEL

Capture:

```text
/etc/os-release
hostname
uname
ip addr
ip route
systemctl status httpd
ss -lntp
local curl result
```

Các outputs này chứng minh guest-level configuration.

---

# 134. Evidence Package — Bootstrap

Capture bounded:

```text
/var/log/cloud-init-output.log
```

với:

```text
timestamp
instance ID
collection command
log range
```

Không submit raw full log nếu nó có information ngoài approved scope.

---

# 135. Evidence Package — Network Test

Matrix:

| Test                         | Expected | Actual   | Result |
| ---------------------------- | -------- | -------- | ------ |
| SSH from admin               | Allow    | Allow    | PASS   |
| HTTP from admin before fault | Allow    | Allow    | PASS   |
| HTTP after SG rule removed   | Deny     | Deny     | PASS   |
| Local HTTP during SG deny    | Allow    | Allow    | PASS   |
| HTTP after rollback          | Allow    | Allow    | PASS   |
| IMDSv2 with token            | Allow    | Allow    | PASS   |
| IMDS without token           | Deny     | Deny/401 | PASS   |

Đây là rất mạnh vì evidence chứng minh cả positive và negative behavior.

---

# 136. Evidence Package — Failure Injection

Record theo format:

```text
Fault ID
Timestamp
Baseline
Injected fault
Observed symptom
Hypothesis
Commands executed
Evidence collected
Root cause
Correction
Rollback
Post-check
Result
```

Ví dụ:

```text
Fault ID: D4-F01
Fault: Remove SG TCP/80
Symptom: External curl timeout
Local curl: PASS
Root Cause: Missing SG ingress rule
Correction: Restore TCP/80 ADMIN_CIDR
Post-check: HTTP 200
Result: PASS
```

---

# 137. Hash Evidence

Sau khi evidence files finalized:

```bash
find evidence -type f -print0 \
  | sort -z \
  | xargs -0 sha256sum \
  > evidence/SHA256SUMS.txt
```

Verify:

```bash
sha256sum -c evidence/SHA256SUMS.txt
```

Lưu ý `SHA256SUMS.txt` tự chứa hashes của files trước thời điểm nó được generate; không nên cố self-hash vào chính nó theo cách recursive.

---

# 138. Evidence Manifest

Một row có thể như:

| Field          | Value                                    |
| -------------- | ---------------------------------------- |
| Evidence ID    | `D4-E010`                                |
| Source         | AWS EC2                                  |
| Environment    | Sandbox                                  |
| Account        | `<account>`                              |
| Region         | `<region>`                               |
| Resource       | `i-...`                                  |
| Ticket         | `REQ-AWS-D4-001`                         |
| Collector      | `<learner>`                              |
| Timezone       | UTC                                      |
| Timestamp      | `...Z`                                   |
| Command        | `describe-instances ... MetadataOptions` |
| Format         | JSON                                     |
| Classification | Internal/Lab                             |
| SHA-256        | `<hash>`                                 |
| Secret Review  | PASS                                     |
| Limitation     | None                                     |

---

# 139. Pre-Cleanup Verification

Trước cleanup, đảm bảo đã thu đủ:

```text
Identity evidence
IAM allowed/denied
Tag enforcement
VPC
Subnet
Route
SG
NACL
EC2
Metadata
SSH/Linux
Bootstrap log
Connectivity tests
Failure/recovery
Architecture
Manifest
Hashes
```

Không terminate EC2 rồi mới nhớ:

> “Chưa lấy cloud-init log.”

---

# 140. Cleanup vs Rollback

Hai khái niệm khác nhau.

`Rollback`:

```text
undo một test/change
để restore previous known-good state
```

Ví dụ:

```text
re-add TCP/80 rule
```

`Cleanup`:

```text
remove lab resources
khi lab kết thúc
```

Ví dụ:

```text
terminate EC2
delete VPC
```

---

# 141. Cleanup Step 1 — Final Identity Check

Trước destructive cleanup:

```bash
aws sts get-caller-identity \
  --profile "$PROFILE"
```

Compare:

```text
Account
Role
Region
Ticket
Resource IDs
```

Đây là pre-destruction safety gate.

---

# 142. Cleanup Step 2 — Terminate EC2

Only approved lab instance:

```bash
aws ec2 terminate-instances \
  --instance-ids "$INSTANCE_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Wait:

```bash
aws ec2 wait instance-terminated \
  --instance-ids "$INSTANCE_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Do not:

```bash
aws ec2 terminate-instances --instance-ids <ID copied from memory>
```

without verifying tag/ID/account.

---

# 143. Verify Termination

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Expected:

```text
terminated
```

EC2 root EBS deletion behavior phụ thuộc block-device settings; default behavior commonly deletes root volume on termination, nên termination phải được coi là destructive. [AWS Documentation](https://docs.aws.amazon.com/us_en/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html?utm_source=chatgpt.com)

---

# 144. Cleanup Step 3 — Delete Security Group

Sau EC2/ENI dependencies đã gone:

```bash
aws ec2 delete-security-group \
  --group-id "$SG_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Nếu:

```text
DependencyViolation
```

không force random deletes.

Find resource still referencing SG.

---

# 145. Cleanup Step 4 — Delete Imported Key Pair

AWS-side key pair:

```bash
aws ec2 delete-key-pair \
  --key-name "$KEY_NAME" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Local private key:

```text
do not delete blindly
```

Nếu key được tạo riêng cho disposable lab và retention policy yêu cầu removal:

```bash
rm ~/.ssh/day4-ec2 ~/.ssh/day4-ec2.pub
```

chỉ sau khi xác nhận không resource nào khác sử dụng nó.

Private key không thuộc audit evidence.

---

# 146. Cleanup Step 5 — Route Tables

Disassociate custom associations:

```bash
aws ec2 disassociate-route-table \
  --association-id "$PUBLIC_ASSOC_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

```bash
aws ec2 disassociate-route-table \
  --association-id "$PRIVATE_ASSOC_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Delete:

```bash
aws ec2 delete-route-table \
  --route-table-id "$PUBLIC_RTB_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

```bash
aws ec2 delete-route-table \
  --route-table-id "$PRIVATE_RTB_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

---

# 147. Cleanup Step 6 — Internet Gateway

Detach:

```bash
aws ec2 detach-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --vpc-id "$VPC_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Delete:

```bash
aws ec2 delete-internet-gateway \
  --internet-gateway-id "$IGW_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

---

# 148. Cleanup Step 7 — Subnets

```bash
aws ec2 delete-subnet \
  --subnet-id "$PUBLIC_SUBNET_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

```bash
aws ec2 delete-subnet \
  --subnet-id "$PRIVATE_SUBNET_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

---

# 149. Cleanup Step 8 — VPC

```bash
aws ec2 delete-vpc \
  --vpc-id "$VPC_ID" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Nếu:

```text
DependencyViolation
```

đó là useful operational signal.

Không random delete.

Inventory remaining dependencies.

---

# 150. Nếu Optional NAT Đã Được Tạo

Cleanup order phải xử lý NAT trước.

Conceptual:

```text
delete NAT Gateway
     ↓
wait deletion
     ↓
remove NAT routes
     ↓
release associated disposable EIP if approved
     ↓
continue subnet/VPC cleanup
```

NAT cleanup đặc biệt quan trọng vì lab resource có thể tiếp tục phát sinh cost.

---

# 151. IAM Cleanup

Nếu role/profile do instructor/shared lab provision:

```text
DO NOT DELETE
```

Nếu bạn được explicitly yêu cầu tạo disposable IAM role riêng:

```text
detach policies
remove role from instance profile
delete instance profile
delete role
```

theo approved cleanup runbook.

Không delete shared IAM resource chỉ vì tên có chữ `Lab`.

---

# 152. Final Resource Search

Có thể inspect VPC-scoped resources:

```bash
aws ec2 describe-vpcs \
  --filters "Name=tag:Ticket,Values=$TICKET" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Tương tự:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Ticket,Values=$TICKET" \
  --profile "$PROFILE" \
  --region "$AWS_REGION"
```

Và other resource types nếu tagging policy hỗ trợ.

Goal:

```text
No unexpected running lab compute
No unwanted exposed SG
No forgotten NAT
No unexpected EIP
```

---

# 153. Cleanup Evidence

Record:

```text
Instance terminated
SG deleted
Key pair deleted/reviewed
Route tables deleted
IGW detached/deleted
Subnets deleted
VPC deleted
IAM resources retained/deleted according to ownership
```

Final evidence should answer:

> “Bạn đã để lại resource gì?”

Không:

> “Chắc là xóa hết rồi.”

---

# 154. Operational Troubleshooting Flow phải thuộc

Khi EC2 application unreachable:

```text
1. Verify Account
        ↓
2. Verify Region
        ↓
3. Verify Instance ID
        ↓
4. Verify EC2 State
        ↓
5. Verify Status Checks
        ↓
6. Verify IP Address
        ↓
7. Verify Subnet
        ↓
8. Verify Route Table
        ↓
9. Verify IGW/NAT architecture
        ↓
10. Verify NACL
        ↓
11. Verify Security Group
        ↓
12. Verify Guest IP/route
        ↓
13. Verify Guest firewall
        ↓
14. Verify Service
        ↓
15. Verify Port Listener
        ↓
16. Verify Application
```

Order này giảm risk vì phần đầu chủ yếu read-only.

---

# 155. IAM Troubleshooting Flow phải thuộc

```text
1. aws sts get-caller-identity
        ↓
2. Correct AWS Account?
        ↓
3. Correct Principal?
        ↓
4. Correct Role Session?
        ↓
5. Which API Action?
        ↓
6. Which Resource?
        ↓
7. Relevant Allow?
        ↓
8. Explicit Deny?
        ↓
9. Permissions Boundary?
        ↓
10. Session Policy?
        ↓
11. SCP/RCP?
        ↓
12. Resource/Trust Policy?
```

Không sửa IAM bằng:

```text
attach AdministratorAccess
```

rồi coi issue solved.

---

# 156. SSH Troubleshooting Flow phải thuộc

```text
Timeout
  ↓
Network path first

Connection refused
  ↓
Listener/service/firewall

Permission denied (publickey)
  ↓
Username/key/authentication
```

Đây là symptom classification cực kỳ hữu ích trong operations.

---

# 157. User Data Troubleshooting Flow phải thuộc

```text
Did cloud-init/user-data execute?
        ↓
Check cloud-init-output.log
        ↓
Which command failed?
        ↓
Was network available?
        ↓
Was DNS available?
        ↓
Was package available?
        ↓
Did service install?
        ↓
Did service start?
        ↓
Is port listening?
        ↓
Does local request work?
        ↓
Does external request work?
```

---

# 158. L1/L2 Operational Boundary

Một L1/L2 Infrastructure Operations engineer trong lab này nên biết distinction:

| Tình huống                      | Thao tác                               |
| ------------------------------- | -------------------------------------- |
| Wrong Region selected           | Correct context                        |
| Instance stopped unexpectedly   | Verify ticket/dependency before start  |
| SG known rule missing           | Correct only if authorized             |
| `httpd` stopped                 | Restart if runbook permits             |
| Wrong IAM role                  | Stop operation and correct access path |
| AccessDenied unexplained        | Gather evidence/escalate               |
| SCP/organization policy         | Escalate IAM/cloud governance          |
| Suspected credential compromise | Escalate Security immediately          |
| Corporate route/VPN issue       | Escalate network ownership             |
| AWS system status impairment    | Follow EC2/AWS support procedure       |
| Destructive rebuild required    | Change approval required               |

---

# 159. Stop Conditions

Bạn phải dừng lab change nếu gặp:

```text
Unexpected production Account ID
Unknown role
Unapproved Region
Unknown resource ownership
Resource lacks expected Ticket/Owner tag
Credential exposed
Unexpected Administrator access
Unexpected IAM denied/allowed behavior
Potential production route
Unclear cleanup dependency
Possible security incident
```

Đây không phải “lab bị fail”.

Biết dừng đúng lúc là một Infrastructure Operations skill.

---

# 160. Acceptance Package

Final package nên có:

| Artifact          | Requirement                      |
| ----------------- | -------------------------------- |
| Ticket            | Approval/scope/expiry            |
| Identity          | Account + ARN + timestamp        |
| IAM Allow         | Successful permitted operation   |
| IAM Deny          | Expected denial                  |
| Tags              | Mandatory tag evidence           |
| Architecture      | VPC/subnets/routes               |
| VPC evidence      | VPC ID/CIDR                      |
| Subnet evidence   | CIDR/AZ/public-IP property       |
| Route evidence    | Public/private route distinction |
| SG evidence       | 22/80 restricted source          |
| NACL evidence     | Association/rules                |
| EC2 evidence      | AMI/type/subnet/profile/tags     |
| Metadata          | IMDSv2 required                  |
| SSH evidence      | Successful secure login          |
| OS evidence       | Linux/RHEL identity              |
| Bootstrap         | cloud-init output                |
| Service           | `httpd` active/listening         |
| Positive network  | HTTP success                     |
| Negative network  | HTTP denied                      |
| Rollback          | HTTP restored                    |
| Failure injection | Diagnosis + root cause           |
| Cleanup           | Resource disposition             |
| Manifest          | Traceability                     |
| SHA-256           | Integrity hashes                 |
| Secret check      | PASS                             |

---

# 161. Day 4 Practical Completion Criteria

Bạn chỉ nên coi Module Thực hành Day 4 hoàn thành khi có thể tự làm và giải thích được chain sau:

```text
Ticket
  ↓
Approved identity
  ↓
MFA / temporary session
  ↓
CLI profile
  ↓
GetCallerIdentity
  ↓
Allowed IAM test
  ↓
Denied IAM test
  ↓
Mandatory tag validation
  ↓
VPC
  ↓
CIDR
  ↓
Public subnet
  ↓
Private subnet
  ↓
Route tables
  ↓
IGW
  ↓
NACL inspection
  ↓
Security Group
  ↓
Approved AMI
  ↓
Key Pair
  ↓
IAM Instance Profile
  ↓
User Data
  ↓
EC2 launch
  ↓
Status checks
  ↓
SSH
  ↓
Linux/RHEL verification
  ↓
httpd
  ↓
Provisioning logs
  ↓
IMDSv2 token request
  ↓
Tokenless denial
  ↓
Allowed connectivity
  ↓
Denied connectivity
  ↓
Fault isolation
  ↓
Rollback
  ↓
Evidence
  ↓
Cleanup
```

Nếu bạn chỉ launch EC2 và SSH được nhưng không thể giải thích:

```text
vì sao packet đi được,
vì sao IAM action được phép,
vì sao một action khác bị deny,
vì sao subnet là public,
vì sao IMDSv1 bị block,
vì sao HTTP fail khi SG rule bị remove,
```

thì chưa đạt objective của Day 4.

---

# 162. Cheat Sheet Thực hành Day 4

| Objective        | Command / Check                                |
| ---------------- | ---------------------------------------------- |
| Identity         | `aws sts get-caller-identity`                  |
| Region           | `aws configure get region`                     |
| VPC              | `aws ec2 describe-vpcs`                        |
| Subnets          | `aws ec2 describe-subnets`                     |
| Routes           | `aws ec2 describe-route-tables`                |
| SG               | `aws ec2 describe-security-groups`             |
| NACL             | `aws ec2 describe-network-acls`                |
| EC2              | `aws ec2 describe-instances`                   |
| EC2 health       | `aws ec2 describe-instance-status`             |
| AMI              | `aws ec2 describe-images`                      |
| Dry-run          | `aws ec2 run-instances --dry-run ...`          |
| Linux IP         | `ip addr`                                      |
| Linux routes     | `ip route`                                     |
| Listeners        | `ss -lntp`                                     |
| Service          | `systemctl status httpd`                       |
| Logs             | `journalctl`, `/var/log/cloud-init-output.log` |
| Local HTTP       | `curl http://127.0.0.1/`                       |
| External HTTP    | `curl http://<public-ip>/`                     |
| IMDSv2 token     | `PUT /latest/api/token`                        |
| IMDS instance ID | `/latest/meta-data/instance-id`                |
| Hash             | `sha256sum`                                    |

AWS Security Group là `stateful`, NACL là `stateless`; public subnet được xác định bởi route tới IGW; `HttpTokens=required` bắt buộc IMDSv2; EC2 `--dry-run` là một trong những công cụ rất hữu ích để kiểm tra authorization mà không tạo resource. [AWS Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html?utm_source=chatgpt.com)

---

## Kết quả cuối cùng bạn phải đạt

Một Infrastructure Operations engineer hoàn thành Day 4 không nên nói đơn giản:

> “Tôi biết tạo VPC và EC2.”

Bạn phải có thể nói:

> “Tôi xác minh đúng AWS Account, Region và principal trước khi thao tác; sử dụng approved temporary/federated access và MFA; kiểm tra allowed/denied IAM behavior; xác minh mandatory tagging; thiết kế và provision VPC với public/private routing rõ ràng; hiểu SG và NACL; launch EC2 từ approved AMI với key pair, IAM instance profile, User Data và IMDSv2; verify từ AWS control plane xuống Linux service và end-to-end connectivity; phân biệt application failure với network/IAM failure; thu thập evidence không chứa secrets; thực hiện rollback và cleanup có kiểm soát; và biết khi nào vấn đề vượt L1/L2 boundary để escalate.”

Đó mới là **Day 4 practical operational readiness** mà syllabus yêu cầu. Infrastructure_Operations_14_Da…
