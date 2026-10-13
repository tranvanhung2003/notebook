Được. Với tình huống **sắp phỏng vấn và syllabus vẫn là trọng tâm**, bạn không nên cố học sâu toàn bộ stack này. Mục tiêu hợp lý là: **nghe tên biết nó là gì → hiểu nó dùng để làm gì → mô tả được flow vận hành → biết vài keyword quan trọng → gặp câu hỏi scenario thì trả lời theo tư duy operations/evidence/rollback/escalation**.

Danh sách JD này mở rộng trực tiếp từ AWS Day 7–11 và Operations Controls Day 18–21 trong syllabus của bạn. 00_MASTER_SYLLABUS_Java_Fresher… 00_MASTER_SYLLABUS_Java_Fresher…

# 1. Thứ tự ưu tiên học gấp

Nếu chỉ còn ít thời gian, học theo thứ tự này:

| Priority | Skill                                    | Mức cần đạt                                |
| -------- | ---------------------------------------- | ------------------------------------------ |
| **P0**   | AWS architecture & operations            | Phải trả lời ổn                            |
| **P0**   | AWS RDS                                  | Phải hiểu architecture + vận hành          |
| **P0**   | AWS CloudWatch                           | Metrics / Logs / Alarm / troubleshooting   |
| **P0**   | IAM / IAM review 90 ngày                 | Rất dễ hỏi vì liên quan security           |
| **P0**   | Release/change management                | Phải mô tả được flow                       |
| **P0**   | ServiceNow                               | Biết ticket lifecycle                      |
| **P0**   | Audit evidence / log extraction          | Rất hợp role Administration/Operations     |
| **P1**   | Jenkins                                  | Biết CI/CD pipeline, Jenkinsfile           |
| **P1**   | Terraform / CloudFormation               | Biết IaC concept + workflow                |
| **P1**   | Active Directory                         | Domain/User/Group/DC/GPO cơ bản            |
| **P1**   | Privileged ID/PAM                        | Hiểu privileged access workflow            |
| **P1**   | Excel/VBA automation                     | Hiểu mục đích + Dictionary/CSV/multi-sheet |
| **P1**   | Outlook Object Model                     | Biết object hierarchy + automation         |
| **P2**   | opswitch                                 | Biết AWS operation automation              |
| **P2**   | PalletControl / PS-Gateway / AccessCheck | Chỉ cần mental model nếu là internal tool  |

Một nguyên tắc quan trọng trong phỏng vấn: **đừng giả vờ đã dùng tool mà bạn chưa dùng**. Cách trả lời tốt hơn là:

> “Em chưa có kinh nghiệm vận hành trực tiếp sản phẩm X, nhưng em hiểu mục đích của nó trong operational workflow là … Nếu tiếp nhận công việc, em sẽ dựa vào runbook, ticket, permission boundary, validation và evidence để thao tác.”

Câu này an toàn hơn rất nhiều so với bịa chức năng.

---

# 2. AWS Architecture and Operations

Đây là phần quan trọng nhất.

## Bạn phải hiểu AWS architecture theo mental model này

```text
User
  |
DNS
  |
Load Balancer
  |
VPC
  |
+-----------------------------+
| Public Subnet               |
|   ALB / NAT Gateway         |
|                             |
| Private Subnet              |
|   EC2 / Application         |
|          |                  |
| Private DB Subnet           |
|   RDS                       |
+-----------------------------+
          |
      CloudWatch
          |
       Alarms
```

Đừng tập trung nhớ hàng trăm AWS services. Với fresher operations, cần hiểu các thành phần sau.

### Region và Availability Zone

**Region** là khu vực địa lý AWS, ví dụ Tokyo, Singapore.

Một Region có nhiều **Availability Zones – AZs**.

AZ là các location độc lập về hạ tầng trong cùng Region nhằm hỗ trợ high availability.

Ví dụ:

```text
ap-southeast-1
    ├── AZ-a
    ├── AZ-b
    └── AZ-c
```

Interview:

**Q: Tại sao deploy multi-AZ?**

Trả lời:

> Để tránh single point of failure. Nếu workload chỉ nằm trong một AZ thì failure của AZ đó có thể làm mất availability. Production architecture thường phân bố các resource quan trọng giữa nhiều AZ khi service hỗ trợ.

---

## VPC

**VPC – Virtual Private Cloud** là isolated virtual network trong AWS.

Bạn quyết định:

- CIDR/IP range
- subnet
- route
- network connectivity
- security controls

Ví dụ:

```text
VPC: 10.0.0.0/16

Public subnet:
10.0.1.0/24

Private app subnet:
10.0.2.0/24

Private DB subnet:
10.0.3.0/24
```

---

## Public subnet vs Private subnet

Không phải cứ subnet có public IP thì gọi public subnet.

Điểm quan trọng là **routing**.

Public subnet thường có route:

```text
0.0.0.0/0 -> Internet Gateway
```

Private subnet không có direct route tới Internet Gateway.

Private server cần outbound Internet có thể đi:

```text
Private EC2
   ↓
NAT Gateway
   ↓
Internet Gateway
   ↓
Internet
```

---

# 3. Security Group và NACL

Đây là câu AWS interview cực phổ biến.

## Security Group

SG là firewall ở resource/ENI level.

Ví dụ EC2 chạy Tomcat:

```text
Inbound:
TCP 8080
Source: ALB Security Group
```

RDS PostgreSQL:

```text
Inbound:
TCP 5432
Source: Application Security Group
```

Đừng cấu hình:

```text
5432 from 0.0.0.0/0
```

trừ trường hợp lab đặc biệt.

### Keyword phải nhớ

**Security Group = stateful.**

Traffic response của connection đã được cho phép được xử lý theo state của connection.

---

## NACL

NACL hoạt động ở subnet level.

**NACL = stateless.**

Có cả:

```text
ALLOW
DENY
```

rules.

Trong troubleshooting, nên kiểm tra:

```text
DNS
↓
Route
↓
NACL
↓
Security Group
↓
Port/listener
↓
Application
```

---

# 4. AWS operations nghĩa là làm gì?

“Operate AWS” không có nghĩa đơn giản là biết bấm Console.

Một operator thường làm:

```text
Receive request
      ↓
Understand requirement
      ↓
Confirm target/resources
      ↓
Assess impact/risk
      ↓
Get approval
      ↓
Pre-check
      ↓
Execute change
      ↓
Validate
      ↓
Collect evidence
      ↓
Close ticket / handover
```

Đây chính là tư duy rất phù hợp với Day 18 của syllabus.

Ví dụ request:

> Restart EC2 application server production lúc 22:00.

Bạn không nên ngay lập tức restart.

Phải xác nhận:

```text
Which AWS account?
Which Region?
Which EC2 instance?
Production or staging?
Reason?
Maintenance window?
Expected impact?
Application owner?
Approval obtained?
Rollback/recovery?
```

Sau đó pre-check:

```bash
uptime
df -h
free -m
systemctl status tomcat
ss -lntp
```

AWS:

```text
EC2 instance state
CloudWatch alarm state
CPU
network
status checks
```

Sau change:

```text
EC2 running?
Tomcat running?
Port listening?
HTTP health endpoint returns expected result?
CloudWatch alarm OK?
```

---

# 5. AWS RDS

**Amazon RDS – Relational Database Service** là managed relational database service.

AWS quản lý nhiều công việc infrastructure/database administration như provisioning, backup, patching và failure detection/recovery; hiện RDS hỗ trợ các engine như PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, Db2. [Tài liệu AWS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/?utm_source=chatgpt.com)

Interview answer khoảng 30 giây:

> “Amazon RDS là managed relational database service của AWS. Thay vì tự cài database lên EC2 rồi tự quản lý OS, patching và backup infrastructure, RDS giúp AWS quản lý nhiều phần operational đó. Khi vận hành em vẫn cần quan tâm network connectivity, security group, credentials, parameter configuration, storage, performance metrics, backup và availability.”

Đây là câu rất ổn.

---

## RDS DB Instance

**DB instance** là database environment chạy trong RDS.

Application connect bằng:

```text
endpoint
port
database name
username/password
```

Ví dụ PostgreSQL:

```text
mydb.xxx.ap-southeast-1.rds.amazonaws.com
5432
appdb
app_user
```

Bạn **không SSH trực tiếp vào host OS của standard RDS** như một EC2 thông thường. AWS xác nhận direct host access không được cung cấp cho standard RDS, ngoại trừ mô hình RDS Custom. [Tài liệu AWS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.DBInstance.html?utm_source=chatgpt.com)

---

# 6. RDS Multi-AZ vs Read Replica

Rất dễ bị hỏi.

## Multi-AZ

Mục tiêu:

**High Availability.**

Mental model:

```text
Primary RDS
   |
synchronous replication
   |
Standby in another AZ
```

Nếu primary gặp failure, AWS có thể failover.

Đừng trả lời:

> Multi-AZ dùng để tăng read performance.

Không phải mục đích chính.

---

## Read Replica

Mục tiêu chính:

**Read scalability.**

```text
             Primary
            /       \
           ↓         ↓
Read Replica 1    Read Replica 2
```

Application có thể đưa read query sang replicas.

Câu nhớ:

> **Multi-AZ = availability. Read replica = read scaling.**

---

# 7. RDS backup

Bạn nên biết ba khái niệm:

### Automated backup

AWS tự động backup theo cấu hình retention.

### Manual snapshot

Bạn chủ động tạo snapshot.

Ví dụ trước một change lớn:

```text
Change RDS configuration
      ↓
Create snapshot
      ↓
Verify snapshot available
      ↓
Perform change
```

### Restore

Restore thường tạo ra DB environment mới từ backup/snapshot thay vì “undo” theo kiểu Ctrl+Z.

Interview nên nói:

> “Backup chỉ có giá trị khi restore được kiểm chứng.”

Đây là mindset operations rất tốt.

---

# 8. Troubleshooting RDS connection

Scenario:

> Application cannot connect to PostgreSQL RDS.

Đừng trả lời ngay “restart database”.

Đi theo layer:

```text
1. Application configuration
2. DNS / endpoint
3. Port
4. Route / VPC
5. Security Group
6. DB status
7. Credentials
8. Database privilege
9. Connection limit
10. Database logs/metrics
```

Ví dụ:

```bash
nslookup mydb.xxx.rds.amazonaws.com
```

Network check:

```bash
nc -vz mydb.xxx.rds.amazonaws.com 5432
```

Nếu network OK nhưng login fail:

```text
authentication / password / user / pg_hba-equivalent configuration
```

Nếu timeout:

```text
route
SG
NACL
DB availability
DNS/network
```

Nếu:

```text
password authentication failed
```

thì không nên mất thời gian điều tra SG trước vì connection đã đến database rồi.

Đây chính là evidence-driven troubleshooting.

---

# 9. AWS CloudWatch

Một câu nhớ:

> **CloudWatch = monitoring/observability. CloudTrail = AWS API activity/audit.**

Đừng nhầm hai cái.

CloudWatch cung cấp metrics, alarms, dashboards, logs và các chức năng observability cho AWS resources/applications. [Tài liệu AWS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/?utm_source=chatgpt.com)

---

## CloudWatch Metrics

Metric = time-series numerical data.

Ví dụ:

```text
EC2 CPUUtilization
RDS CPUUtilization
RDS DatabaseConnections
EBS metrics
ALB RequestCount
```

AWS mô tả metric là một tập các datapoint theo thời gian. [Tài liệu AWS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html?utm_source=chatgpt.com)

Mental model:

```text
Metric
  |
Datapoints
  |
Threshold
  |
Alarm
  |
SNS / Action
```

---

# 10. CloudWatch Alarm

Alarm theo dõi metric và đánh giá threshold.

Ba state rất nên nhớ:

```text
OK
ALARM
INSUFFICIENT_DATA
```

AWS định nghĩa đúng ba trạng thái metric alarm này. [Tài liệu AWS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation.html?utm_source=chatgpt.com)

Ví dụ:

```text
RDS CPUUtilization > 80%
for 5 minutes
        ↓
      ALARM
        ↓
       SNS
        ↓
Email / notification
```

Một operator không chỉ thấy ALARM rồi restart.

Phải hỏi:

```text
Alarm nào?
Metric nào?
Resource nào?
Threshold?
Duration?
Recent change?
Correlated metrics?
Application impact?
```

---

# 11. CloudWatch Logs

CloudWatch Logs dùng để:

- collect logs
- centralize logs
- search
- analyze
- retain
- create metric/filter/alarm

Concept:

```text
Log Group
    |
Log Stream
    |
Log Events
```

Ví dụ:

```text
/aws/lambda/app
/application/tomcat
```

CloudWatch Logs cũng hỗ trợ Logs Insights để query logs. [Tài liệu AWS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html?utm_source=chatgpt.com)

---

# 12. CloudWatch Agent

Một câu hỏi phổ biến:

> EC2 CloudWatch có memory/disk metric không?

Bạn nên biết rằng host-level metrics/logs bổ sung thường có thể thu thập bằng **CloudWatch Agent**.

AWS mô tả CloudWatch Agent có thể collect system metrics như CPU, memory, disk và network cũng như application logs từ EC2 hoặc server. [Tài liệu AWS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html?utm_source=chatgpt.com)

---

# 13. CloudWatch vs CloudTrail

Cực kỳ nên thuộc bảng này.

|                                | CloudWatch             | CloudTrail                                      |
| ------------------------------ | ---------------------- | ----------------------------------------------- |
| Focus                          | Monitoring             | Audit/API activity                              |
| CPU metric                     | Yes                    | No                                              |
| Application logs               | Yes                    | Không phải chức năng chính                      |
| Alarm                          | Yes                    | Có thể kết hợp nhưng không phải chức năng chính |
| “Who deleted EC2?”             | Không phải nguồn chính | **CloudTrail**                                  |
| “CPU > 90%?”                   | **CloudWatch**         | No                                              |
| “Who modified Security Group?” | CloudTrail             | **Yes**                                         |

Cách nhớ:

```text
CloudWatch = WHAT is happening operationally?

CloudTrail = WHO did WHAT through AWS API/control plane?
```

---

# 14. Jenkins / Release Management

Bạn không cần học Jenkins administrator sâu.

Chỉ cần hiểu:

> Jenkins là automation server thường dùng cho CI/CD.

Một **Pipeline** biểu diễn delivery process, thường từ source code đến build/test/deploy. Jenkins Pipeline thường được lưu dưới dạng **Jenkinsfile**, tức pipeline-as-code. [Jenkins](https://www.jenkins.io/doc/book/pipeline/?utm_source=chatgpt.com)

Basic pipeline:

```text
Git commit
    ↓
Checkout
    ↓
Build
    ↓
Unit Test
    ↓
Package
    ↓
Deploy Staging
    ↓
Smoke Test
    ↓
Approval
    ↓
Deploy Production
    ↓
Validation
```

---

## Jenkins concepts phải biết

### Controller

Quản lý Jenkins.

### Agent/node

Machine thực thi workload.

### Job

Một automation unit.

### Pipeline

Chuỗi stages.

### Stage

Ví dụ:

```text
Build
Test
Deploy
```

### Step

Command cụ thể bên trong stage.

Jenkins official documentation sử dụng chính model Pipeline → Stage → Step này. [Jenkins](https://www.jenkins.io/doc/book/pipeline/?utm_source=chatgpt.com)

---

# 15. Jenkinsfile

Ví dụ bạn chỉ cần đọc hiểu:

```groovy
pipeline {
    agent any

    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploy application'
            }
        }
    }
}
```

Bạn cần giải thích được:

```text
pipeline = toàn bộ pipeline
agent = nơi chạy
stages = danh sách giai đoạn
stage = logical phase
steps = task
sh = execute shell command
```

Jenkins official deployment examples cũng dùng flow tối thiểu Build → Test → Deploy. [Jenkins](https://www.jenkins.io/doc/pipeline/tour/deployment/?utm_source=chatgpt.com)

---

# 16. Release management

Release management không đồng nghĩa CI/CD.

Operations flow:

```text
Release request
      ↓
Change ticket
      ↓
Impact/risk assessment
      ↓
Approval
      ↓
Prepare artifact/version
      ↓
Pre-check
      ↓
Deploy
      ↓
Smoke test
      ↓
Validation
      ↓
Evidence
      ↓
Close / handover
```

Phải luôn nghĩ đến:

```text
Who approved?
What version?
Where deploy?
When?
What validation?
Rollback trigger?
Rollback procedure?
Evidence?
```

---

# 17. Rollback

Ví dụ:

```text
Current:
app-1.2.3.war

Deploy:
app-1.2.4.war
```

Nếu health check fail:

```text
stop deployment
restore 1.2.3
restart safely
validate endpoint
check logs
record rollback
```

Interview:

**Q: Khi release production fail bạn làm gì?**

Trả lời:

> “Em ưu tiên bảo vệ availability. Em sẽ xác nhận symptom và impact, dừng các thay đổi tiếp theo, kiểm tra rollback criteria trong approved change, thực hiện rollback nếu nằm trong L1/runbook scope, sau đó validate service và lưu evidence. Nếu nguyên nhân hoặc remediation ngoài quyền hạn, em escalate với log và timeline đầy đủ.”

Rất đúng kiểu operations.

---

# 18. ServiceNow

ServiceNow là platform rất hay được dùng cho **IT Service Management – ITSM**.

Ở mức phỏng vấn fresher, tập trung:

```text
Incident
Service Request
Change Request
Problem
Task
Configuration Item (CI)
SLA
Work notes
Assignment group
Approval
```

ServiceNow mô tả Incident Management là quá trình restore normal service operation và giảm business impact. [ServiceNow](https://www.servicenow.com/docs/r/it-service-management/incident-management/c_IncidentManagement.html?utm_source=chatgpt.com)

---

# 19. Incident vs Request vs Change

### Incident

Something is broken.

Ví dụ:

```text
Production application unavailable.
```

Mục tiêu:

```text
Restore service.
```

### Service Request

User muốn một service/action tiêu chuẩn.

Ví dụ:

```text
Request IAM access.
Create account.
Reset password.
```

### Change

Thay đổi infrastructure/application.

Ví dụ:

```text
Upgrade Tomcat
Change SG
Deploy new release
Modify RDS parameter
```

Change cần quản lý:

```text
risk
impact
approval
schedule
implementation
validation
rollback
```

ServiceNow cho phép change request liên kết affected CIs, outages và work notes. [ServiceNow](https://www.servicenow.com/docs/r/it-service-management/service-operations-workspace/change-sow.html?utm_source=chatgpt.com)

---

# 20. Ticket quality

Một ticket tốt phải giúp người khác reconstruct được:

```text
What happened?
Who requested?
Who approved?
When?
Which resource?
What was changed?
What evidence?
What result?
Any incident?
Rollback?
```

Đừng ghi:

```text
Done.
```

Hãy ghi kiểu:

```text
2026-xx-xx 22:05 ICT
Change executed on EC2 instance i-xxxx.

Pre-check:
- Instance state: running
- Tomcat: active
- Disk utilization: 51%

Action:
- Restarted approved application service.

Validation:
- systemd service active
- TCP/8080 listening
- /health returned HTTP 200
- CloudWatch alarm state OK

Result:
Successful. No rollback required.
```

Đây là operational maturity.

---

# 21. AWS service request coordination

JD của bạn ghi:

> Kinh nghiệm chủ trì vận hành AWS service, bao gồm tiếp nhận yêu cầu, xác nhận nội dung và điều phối.

Điều họ muốn có thể không phải “AWS genius”.

Họ muốn người có khả năng **control operation**.

Mental model:

```text
Requester
    ↓
Operator
    ↓
Application Team
AWS Team
Network Team
Security Team
DB Team
    ↓
Operator coordinates
    ↓
Validation
    ↓
Requester confirmation
```

Khi receive request, dùng 7 câu hỏi:

```text
WHAT?
WHY?
WHERE?
WHEN?
WHO APPROVED?
IMPACT?
ROLLBACK?
```

Sau đó:

```text
Do I have authority?
Is information complete?
Is change approved?
Is there runbook?
Is this L1-safe?
```

Nếu thiếu:

```text
Do not guess.
Clarify.
```

Đây là câu trả lời phỏng vấn rất có giá trị.

---

# 22. IaC: Terraform / CloudFormation

**Infrastructure as Code – IaC** nghĩa là quản lý infrastructure thông qua code/configuration thay vì click thủ công.

Ưu điểm:

```text
repeatable
version-controlled
reviewable
automatable
consistent
auditable
```

Terraform được HashiCorp định nghĩa là IaC tool giúp build/change/version infrastructure bằng configuration files. [HashiCorp Developer](https://developer.hashicorp.com/terraform/docs?utm_source=chatgpt.com)

---

# 23. Terraform

Mental model:

```text
Terraform code
     ↓
Provider
     ↓
AWS API
     ↓
AWS resources
```

Basic workflow cực kỳ nên thuộc:

```bash
terraform init
terraform validate
terraform plan
terraform apply
```

Sau cùng khi lab:

```bash
terraform destroy
```

Ý nghĩa:

```text
init      -> initialize provider/modules
validate  -> validate configuration
plan      -> preview changes
apply     -> apply changes
destroy   -> remove managed infrastructure
```

HashiCorp mô tả workflow cốt lõi theo các bước Author/Initialize/Plan/Apply và Terraform sử dụng state để theo dõi infrastructure. [HashiCorp Developer](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/infrastructure-as-code?utm_source=chatgpt.com)

---

# 24. Terraform state

Đây là concept interview phổ biến.

Terraform có:

```text
terraform.tfstate
```

Nó mapping configuration với infrastructure thực.

Đừng nghĩ state chỉ là “log”.

State cực kỳ quan trọng.

Không nên:

```text
email state file lung tung
commit sensitive state vào public repo
manually modify tùy tiện
```

Trong team thường có remote state/backend và locking tùy architecture.

---

# 25. CloudFormation

CloudFormation = AWS-native IaC.

Bạn viết:

```text
YAML
hoặc
JSON
```

Template mô tả resources.

CloudFormation tạo:

```text
Stack
```

AWS gọi template là blueprint cho resource và stack là tập resource được provision từ template. [Tài liệu AWS](https://docs.aws.amazon.com/cloudformation/?utm_source=chatgpt.com)

Ví dụ:

```yaml
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
```

Interview comparison:

| Terraform       | CloudFormation                    |
| --------------- | --------------------------------- |
| HashiCorp       | AWS                               |
| Multi-provider  | AWS-centric                       |
| HCL             | YAML/JSON                         |
| Terraform state | CloudFormation stack              |
| `plan/apply`    | Change set / stack update concept |

Không cần tranh luận cái nào “better”.

---

# 26. Python / PowerShell scripting

JD không đòi bạn phải là developer.

Họ muốn automation.

Bạn nên biết script có thể dùng để:

```text
read inventory
call APIs
process CSV
generate reports
validate configuration
rename/copy files
extract logs
compare IDs
produce evidence
send notifications
```

Interview:

> “Em sử dụng scripting để giảm manual repetitive work, nhưng với operations em luôn chú ý input validation, logging, error handling, dry-run nếu có, idempotency và tránh hard-code credentials.”

Câu này mạnh.

---

# 27. Active Directory – AD

AD mà người ta nói thường là **Active Directory Domain Services – AD DS**.

AD là directory service lưu và quản lý objects như:

```text
Users
Groups
Computers
Servers
```

Microsoft mô tả AD DS là hierarchical directory lưu network objects và tích hợp authentication/access control. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview?view=sc-vmm-1801&utm_source=chatgpt.com)

---

## Các thuật ngữ AD bắt buộc

### Domain

Administrative/security boundary.

Ví dụ:

```text
corp.example.com
```

### Domain Controller – DC

Server chạy AD DS.

Nó xử lý:

```text
authentication
directory
replication
policy-related services
```

### User

Account người dùng.

### Group

Nhóm users/computers.

Permission thường nên cấp cho group thay vì từng user.

### Organizational Unit – OU

Logical container.

Ví dụ:

```text
corp.local
└── Employees
    ├── Vietnam
    └── Japan
```

### Group Policy – GPO

Centralized policy configuration.

Ví dụ:

```text
password policy
Windows settings
security policy
software configuration
```

---

# 28. AD authentication concepts

Bạn chỉ cần biết keyword:

```text
LDAP
Kerberos
NTLM
DNS
```

AD phụ thuộc DNS rất nhiều.

Microsoft liệt kê các capability truyền thống như LDAP, Kerberos/NTLM và Group Policy trong domain services. [Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/domain-services/overview?utm_source=chatgpt.com)

Nếu hỏi:

**AD và IAM AWS giống nhau không?**

Không.

Có điểm chung đều liên quan identity/access nhưng architecture khác.

```text
AD:
Windows domain
users/computers
Kerberos/LDAP/GPO

AWS IAM:
AWS identities
roles/policies
AWS API/resource authorization
```

---

# 29. opswitch

Mục này mình đã xác minh được.

**opswitch** của Classmethod là tool hỗ trợ automation AWS operations mà không cần viết code trực tiếp.

AWS Marketplace hiện mô tả các use case như:

````text
schedule EC2/RDS start-stop
backup
resource copy
unused resource checking
multi-account operation
tag-based resource selection
execution logs
notifications
``` :chatgpt-content-reference{index="19"}


Nếu interviewer hỏi:

> opswitch là gì?

Bạn có thể trả lời:

> “Theo hiểu biết của em, opswitch là AWS operations automation tool. Nó giúp chuẩn hóa những operation lặp lại như schedule start/stop EC2 hoặc RDS, backup và quản lý operation across accounts mà không phải tự viết toàn bộ script. Từ góc độ operation em vẫn cần quan tâm account/target selection, permission, execution result, log và failure handling.”

Không cần biết UI sâu nếu chưa dùng.

---

# 30. PalletControl

Tên **PalletControl** trong context bạn gửi không có nguồn public đủ đáng tin để mình khẳng định đây là sản phẩm chuẩn nào.

Vì vậy **không nên học một định nghĩa bịa**.

Khả năng cao đây là:

- company-specific tool
- internal operational control
- automation/control system
- hoặc tên sản phẩm niche.

Nếu bị hỏi:

> Bạn biết PalletControl không?

Trả lời an toàn:

> “Em chưa vận hành trực tiếp PalletControl nên em không muốn khẳng định sai chức năng sản phẩm. Tuy nhiên nếu đây là operational control tool của hệ thống, em sẽ tiếp cận theo runbook: xác định request, target, permission, execution procedure, expected result, evidence, error handling và rollback/escalation.”

Đây là cách trả lời mature hơn “em không biết”.

---

# 31. Privileged ID Management / PAM

JD:

> PS/Gateway, AWS Console IAM, AccessCheck...

Ý tưởng cốt lõi là **Privileged Access Management – PAM**.

Privileged account:

```text
root
Administrator
Domain Admin
AWS admin role
database administrator
service privileged account
````

Risk:

```text
privilege misuse
credential theft
untracked access
excessive permissions
```

Vì vậy privileged access thường có flow:

```text
Request
  ↓
Business justification
  ↓
Approval
  ↓
Temporary access
  ↓
Authenticate/MFA
  ↓
Privileged operation
  ↓
Logging/session recording
  ↓
Revoke/expire
  ↓
Audit
```

Keyword:

```text
least privilege
temporary privilege
just-in-time access
MFA
approval
audit trail
session logging
credential rotation
separation of duties
```

---

# 32. AWS IAM basics

IAM có mental model:

```text
Principal
   +
Policy
   +
Resource
   +
Action
   +
Condition
```

Một IAM policy trả lời:

> Ai được làm gì với resource nào trong điều kiện nào?

Ví dụ:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::example/*"
}
```

Bạn không cần thuộc JSON phức tạp.

---

# 33. User vs Role

### IAM User

Long-lived identity.

Có thể có:

```text
console password
access key
```

### IAM Role

Identity được **assume**.

Thường gắn với temporary credentials.

AWS best practice hiện đại ưu tiên temporary credentials/roles thay vì long-lived static credentials ở những nơi phù hợp.

Câu nhớ:

> **User = identity. Role = assumed permission identity. Policy = permissions.**

---

# 34. IAM review 90 ngày

Mục này rất đáng học vì nó khá cụ thể.

AWS IAM Access Analyzer hiện hỗ trợ **unused access analysis**, gồm:

````text
unused IAM role
unused access key
unused password
unused service permissions
unused action permissions
``` :chatgpt-content-reference{index="20"}


AWS cho phép cấu hình tracking period từ **1–365 ngày**; 90 ngày chính là một ví dụ chính thức trong documentation. :chatgpt-content-reference{index="21"}

Vậy “IAM Review 90 days” có thể hiểu:

```text
Every 90 days
     ↓
Collect IAM inventory
     ↓
Check unused/stale access
     ↓
Check excessive privilege
     ↓
Check external/public exposure
     ↓
Classify findings
     ↓
Owner review
     ↓
Remediate
     ↓
Evidence/report
````

---

# 35. Bạn kiểm tra gì trong IAM review?

Ví dụ checklist:

```text
IAM users still required?
MFA enabled?
Unused console password?
Unused access keys?
Old access keys?
Unused roles?
AdminAccess attached?
Wildcard permissions?
External trust?
Cross-account trust?
Public resource policies?
Unused permissions?
```

Đặc biệt:

```text
Action: "*"
Resource: "*"
```

không tự động nghĩa là vulnerability trong mọi trường hợp, nhưng phải review rất kỹ.

---

# 36. Vulnerability classification

JD có:

> phân loại lỗ hổng

Không cần biến thành cybersecurity pentester.

Bạn có thể classify operational IAM findings theo kiểu:

### Critical

Ví dụ:

```text
public access to highly sensitive resource
compromised admin credential
root access exposed
```

### High

```text
unnecessary AdministratorAccess
external privileged trust
stale privileged access
```

### Medium

```text
excessive but limited permissions
unused non-critical access
```

### Low/Informational

```text
naming/tagging issue
minor hygiene problem
```

Quan trọng:

> Severity phải dựa vào **impact + exploitability/exposure + asset criticality**, không chỉ tên rule.

---

# 37. Multi-sheet automation cho IAM review

Đây nhiều khả năng là Excel automation.

Ví dụ workbook:

```text
Sheet 1: IAM Users
Sheet 2: IAM Roles
Sheet 3: Policies
Sheet 4: Access Keys
Sheet 5: Findings
Sheet 6: Summary
```

Automation có thể:

```text
read raw CSV
↓
normalize data
↓
match IDs
↓
apply rules
↓
classify
↓
generate summary
↓
export evidence
```

---

# 38. VBA Intermediate

Không cần học thành VBA developer.

Cần hiểu các khái niệm:

```text
Workbook
Worksheet
Range
Cells
Loop
If
Function
Sub
Dictionary
FileSystemObject
Error handling
```

Ví dụ:

```vb
Dim ws As Worksheet
Set ws = ThisWorkbook.Worksheets("Inventory")

Dim lastRow As Long
lastRow = ws.Cells(ws.Rows.Count, 1).End(xlUp).Row
```

Chỉ cần đọc hiểu.

---

# 39. Scripting.Dictionary

Rất hữu ích khi:

```text
ID reconciliation
duplicate detection
lookup
inventory matching
```

Ví dụ input:

```text
Expected IDs:
A001
A002
A003

Actual IDs:
A001
A003
A004
```

Output:

```text
A002 -> missing
A004 -> unexpected
```

Dictionary giúp lookup nhanh theo key.

Mental model:

```text
Key -> Value
```

Ví dụ:

```text
A001 -> active
A002 -> disabled
```

---

# 40. CSV / file-folder processing

Phải hiểu các vấn đề thực tế:

```text
delimiter
comma
quote
newline
encoding
header
duplicate record
missing field
file exists?
directory exists?
```

Đừng assume:

```text
split every line by ","
```

vì field có thể là:

```csv
123,"Nguyen, Van A",active
```

---

# 41. Encoding

Cực kỳ dễ gặp trong môi trường Nhật.

Các encoding bạn nên nhận diện:

```text
UTF-8
UTF-8 with BOM
UTF-16
Shift-JIS / CP932
ANSI/code page
```

Lỗi encoding thường biểu hiện:

```text
Japanese characters garbled
?????
mojibake
```

Ví dụ:

```text
UTF-8 file
    ↓
read as Shift-JIS
    ↓
garbled characters
```

Khi troubleshoot:

```text
What is source encoding?
What does receiver expect?
Does file contain BOM?
What encoding does VBA/API use?
```

---

# 42. SG / YML / inventory

Trong JD này, mình hiểu nhiều khả năng:

```text
SG = AWS Security Group
YML = YAML configuration
inventory = resource inventory
```

Không thể chắc 100% vì wording có vẻ internal.

Ví dụ inventory:

```text
Account
Region
Instance ID
Name
Private IP
Security Group
Owner
Environment
```

Tool có thể:

```text
read YAML
↓
read AWS SG inventory
↓
compare expected vs actual
↓
generate Excel report
```

Câu interview tốt:

> “Inventory automation theo em là biến configuration/resource data thành structured dataset để có thể compare baseline, phát hiện drift và tạo audit evidence.”

---

# 43. Outlook Object Model

Outlook có object hierarchy.

Cơ bản:

```text
Application
    ↓
NameSpace
    ↓
Folder
    ↓
Items
    ↓
MailItem
```

Microsoft định nghĩa `MailItem` là object đại diện một email message. [Microsoft Learn](https://learn.microsoft.com/en-us/office/vba/api/outlook.mailitem?utm_source=chatgpt.com)

Ví dụ conceptual VBA:

```vb
Set outlookApp = CreateObject("Outlook.Application")

Set mail = outlookApp.CreateItem(0)

mail.To = "example@example.com"
mail.Subject = "Operation Result"
mail.Body = "Completed"
mail.Display
```

Trong operations có thể dùng để:

```text
create standardized mail
send report
attach evidence
notify operation result
```

---

# 44. Các Outlook objects nên biết

```text
Application
NameSpace
MAPIFolder / Folder
Items
MailItem
Attachment
Recipient
```

`NameSpace` thường đại diện MAPI message store và giúp access các default folders như Inbox. [Microsoft Learn](https://learn.microsoft.com/en-us/office/vba/outlook/how-to/using-visual-basic-to-customize-outlook-forms/about-the-object-environment?utm_source=chatgpt.com)

Không cần memorize toàn bộ API.

---

# 45. Error handling trong VBA

Không nên viết script mà error là crash luôn.

Concept:

```vb
On Error GoTo ErrorHandler
```

Flow:

```text
normal execution
    ↓
error
    ↓
ErrorHandler
    ↓
log Err.Number
log Err.Description
cleanup
exit safely
```

Microsoft khuyến nghị sử dụng error-handling routines và `On Error` để route runtime errors. [Microsoft Learn](https://learn.microsoft.com/en-us/office/vba/access/concepts/error-codes/elements-of-run-time-error-handling?utm_source=chatgpt.com)

Bạn cần phân biệt:

```vb
On Error Resume Next
```

không nên lạm dụng vì có thể **nuốt lỗi**.

---

# 46. Automated regression test

Regression test nghĩa đơn giản:

> Sau khi sửa tool/script, kiểm tra những chức năng trước đây đang đúng vẫn còn đúng.

Ví dụ VBA tool:

Version 1:

```text
reads CSV
generates 3 sheets
detects duplicates
```

Sau khi thêm feature:

```text
support UTF-8
```

Regression test phải đảm bảo:

```text
old CSV still works
duplicate detection still correct
sheet count still correct
output columns unchanged
```

---

## Test data nên có

```text
normal input
empty file
missing column
duplicate ID
Japanese characters
very large file
invalid encoding
missing folder
read-only file
```

Một candidate operations biết test error case sẽ ghi điểm.

---

# 47. Audit evidence

Đây là skill rất quan trọng.

**Audit evidence** = bằng chứng cho phép người khác xác nhận một operation đã được thực hiện đúng.

Ví dụ:

```text
ticket
approval
before-state
command output
after-state
log
screenshot
timestamp
operator
resource ID
result
```

Một evidence pack tốt trả lời:

```text
Who?
What?
When?
Where?
Why?
How?
Result?
```

---

# 48. Evidence không chỉ là screenshot

Screenshot rất hữu ích nhưng không phải evidence duy nhất.

Tốt hơn:

```text
raw logs
command output
CSV export
CloudTrail records
ServiceNow ticket history
configuration export
checksum/hash
```

Ví dụ Linux:

```bash
date
hostname
systemctl status tomcat
journalctl -u tomcat --since "2026-10-13 20:00"
```

Có timestamp + hostname sẽ tốt hơn chỉ copy vài dòng error.

---

# 49. Evidence integrity

Mục:

> tính toàn vẹn của bằng chứng

Nghĩa là phải tránh việc evidence bị sửa mà không biết.

Good practice:

```text
preserve raw file
do not overwrite original
use consistent timestamps/timezone
record source
restrict modification
hash when required
```

Ví dụ:

```bash
sha256sum operation.log
```

Kết quả hash giúp phát hiện file bị thay đổi.

Workflow:

```text
raw evidence
    ↓
SHA-256
    ↓
archive
    ↓
working copy
```

---

# 50. Log extraction từ server

Khi được yêu cầu extract logs:

Đừng chạy:

```bash
cat gigantic.log
```

một cách vô thức.

Xác định:

```text
incident time
timezone
host
application
log path
request ID / error keyword
required period
```

Ví dụ:

```bash
journalctl -u tomcat \
  --since "2026-10-13 20:00" \
  --until "2026-10-13 20:30"
```

Hoặc:

```bash
grep -i "ERROR" app.log
```

Nhưng evidence tốt thường giữ cả context:

```bash
grep -B 5 -A 10 "ERROR" app.log
```

---

# 51. Log extraction từ ServiceNow

Concept:

```text
ticket number
status history
assignment
work notes
approval
change tasks
timestamps
affected CI
attachments
```

Khi audit:

```text
Requirement
   ↓
Approval
   ↓
Implementation
   ↓
Evidence
   ↓
Validation
```

Audit muốn chứng minh chain này không bị đứt.

---

# 52. Password generation

JD có:

> tạo PW

Nếu tool generate password thì bạn cần biết nguyên tắc:

```text
use secure random source
follow policy
avoid predictable patterns
don't log plaintext password unnecessarily
secure delivery channel
rotation/expiration if required
```

Không generate bằng:

```text
username + birthday
Password123
Random based on timestamp only
```

Trong interview nhấn mạnh:

> “Password generation phải follow corporate policy và secrets không được lưu/log plaintext tùy tiện.”

---

# 53. TXT / CSV automation

Bạn nên hiểu operations tool thường:

```text
read input
validate
transform
output
log
```

Ví dụ:

```text
Input CSV
   ↓
Validate IDs
   ↓
Compare master data
   ↓
Generate password
   ↓
Generate TXT
   ↓
Create report
   ↓
Evidence
```

Đây có thể chính là pattern của tool trong JD.

---

# 54. AccessCheck

Có một Windows API tên `AccessCheck`, nhưng **không có đủ bằng chứng rằng AccessCheck trong JD của bạn chính là Windows API đó**.

Windows `AccessCheck()` kiểm tra security descriptor có cấp requested access cho access token hay không. [Microsoft Learn](https://learn.microsoft.com/windows/win32/api/securitybaseapi/nf-securitybaseapi-accesscheck?utm_source=chatgpt.com)

Nhưng context:

> PS/Gateway, AWS Console IAM, AccessCheck

làm mình nghiêng về khả năng đây là **internal privileged-access checking tool/process**, không phải Windows API.

Vì vậy trong interview **không tự động nói AccessCheck là Win32 API**.

Hỏi lại context hoặc nói theo access review workflow.

---

# 55. Privileged access workflow

Đây là thứ bạn nên thuộc.

Scenario:

> Engineer cần admin permission vào production trong 2 giờ.

Good workflow:

```text
Engineer submits request
        ↓
Justification
        ↓
Manager/system owner approval
        ↓
PAM grants temporary privileged identity
        ↓
MFA/authentication
        ↓
Activity recorded
        ↓
Work performed
        ↓
Access expires/revoked
        ↓
Logs reviewed/audited
```

Keyword ghi điểm:

```text
least privilege
time-bound access
approval
MFA
traceability
audit
separation of duties
```

---

# 56. Multi-sheet automation

Nếu interviewer hỏi kinh nghiệm Excel automation, bạn có thể mô tả architecture:

```text
Raw_Data
IAM_Users
IAM_Roles
SG
Findings
Exceptions
Summary
Audit_Log
```

Macro:

```text
Read Raw_Data
     ↓
Validate schema
     ↓
Load into dictionary
     ↓
Cross-reference IDs
     ↓
Apply classification rules
     ↓
Create findings
     ↓
Create Summary
     ↓
Save timestamped output
```

Đây chính xác là level “Intermediate VBA” hợp lý.

---

# 57. ID reconciliation

Giả sử:

System A:

```text
001
002
003
004
```

System B:

```text
001
003
004
005
```

Kết quả:

```text
002 -> only A
005 -> only B
001 -> matched
003 -> matched
004 -> matched
```

Trong account/access operations, đây là cách phát hiện:

```text
orphan account
missing account
unauthorized account
deprovisioning failure
```

---

# 58. Inventory management

Inventory nghĩa là danh sách resource có kiểm soát.

AWS inventory có thể chứa:

```text
AWS Account ID
Region
Service
Resource ID
Name
Environment
Owner
Security Group
IP
Status
Tag
```

Operational inventory rất quan trọng vì trước khi thao tác bạn phải biết:

> “Resource này là cái gì và owner là ai?”

---

# 59. Câu scenario quan trọng nhất: “Bạn nhận một AWS operation request thì làm gì?”

Bạn nên gần như thuộc câu trả lời này:

> “Đầu tiên em xác nhận request có đầy đủ target AWS account, Region, resource ID, environment, mục đích và maintenance window hay chưa. Sau đó em kiểm tra approval, impact, dependency và rollback procedure. Trước khi thực hiện em thu thập pre-change state và xác nhận đúng resource để tránh thao tác nhầm. Trong quá trình thực hiện em follow runbook/change procedure và ghi nhận timestamp/result. Sau thay đổi em validate service bằng AWS state, application health check, logs hoặc CloudWatch metrics. Cuối cùng em lưu evidence, cập nhật ServiceNow và handover/escalate nếu còn issue.”

Đây là một trong những câu bạn nên luyện nhất.

---

# 60. “Nếu operation fail thì sao?”

Câu trả lời:

> “Em sẽ không tiếp tục thao tác theo kiểu trial-and-error. Em xác định symptom, impact và bước vừa thực hiện, thu thập logs/evidence, so sánh expected với actual. Nếu runbook có remediation hoặc rollback nằm trong quyền L1 thì em thực hiện và validate. Nếu không xác định được root cause hoặc vượt permission/scope thì em escalate cho đúng team, kèm ticket ID, resource, timestamp, command/action đã thực hiện, logs và current state.”

Đây chính xác là tư duy syllabus.

---

# 61. “Nếu bạn không biết tool này thì sao?”

Ví dụ interviewer hỏi PalletControl.

Không nói:

> Em không biết.

Và cũng không bịa.

Nói:

> “Em chưa có hands-on với PalletControl. Tuy nhiên nếu đây là internal operational control tool, cách em tiếp cận sẽ là đọc runbook, xác định input/output, target system, permission model, execution log, failure behavior và rollback procedure. Với tool operation em đặc biệt chú ý audit trail và việc thao tác đúng environment.”

Rất an toàn.

---

# 62. Những cặp khái niệm cần thuộc trước phỏng vấn

Đây là phần mình khuyên bạn đọc lại ngay trước giờ vào phòng:

| A               | B              | Điểm khác                                              |
| --------------- | -------------- | ------------------------------------------------------ |
| Multi-AZ        | Read Replica   | HA vs read scaling                                     |
| CloudWatch      | CloudTrail     | Monitoring vs API audit                                |
| SG              | NACL           | Stateful/resource vs stateless/subnet                  |
| IAM User        | IAM Role       | Long-lived identity vs assumed identity                |
| Incident        | Change         | Restore service vs planned modification                |
| Backup          | Restore        | Có backup chưa đủ, phải restore được                   |
| Jenkins         | Git            | Automation pipeline vs source control                  |
| Terraform       | CloudFormation | Multi-provider IaC vs AWS-native IaC                   |
| Metrics         | Logs           | Numeric time-series vs event/text data                 |
| CI              | CD             | Continuous integration vs delivery/deployment          |
| Authentication  | Authorization  | Bạn là ai vs bạn được làm gì                           |
| AD              | AWS IAM        | Windows directory/domain vs AWS identity/authorization |
| Public subnet   | Private subnet | Route to IGW vs no direct IGW route                    |
| Alarm           | Incident       | Monitoring condition vs operational service disruption |
| Service Request | Incident       | Request something vs something broken                  |

---

# 63. 15 câu phỏng vấn mình đánh giá xác suất cao

Bạn nên tự trả lời được toàn bộ các câu này:

1. **RDS là gì? RDS khác database trên EC2 thế nào?**
2. **Multi-AZ và Read Replica khác nhau như thế nào?**
3. **CloudWatch dùng làm gì?**
4. **CloudWatch và CloudTrail khác nhau thế nào?**
5. **Security Group và NACL khác nhau thế nào?**
6. **Bạn xử lý một CloudWatch CPU alarm thế nào?**
7. **IAM User, Role và Policy là gì?**
8. **Bạn sẽ review IAM access 90 ngày như thế nào?**
9. **Terraform là gì? `plan` và `apply` khác nhau thế nào?**
10. **Jenkins Pipeline là gì?**
11. **Release production fail thì xử lý thế nào?**
12. **Incident và Change Request khác nhau thế nào?**
13. **Audit evidence cần những gì?**
14. **Nếu request thiếu thông tin hoặc approval bạn làm gì?**
15. **Nếu được giao một internal tool chưa từng dùng thì bạn tiếp cận thế nào?**

Nếu trả lời chắc được 15 câu này, phần skill bổ sung của JD đã ở mức khá an toàn.

---

# 64. 5 câu trả lời “xương sống” nên thuộc

### AWS Operations

> “Em nhìn AWS operation theo lifecycle request → confirmation → approval → pre-check → execution → validation → evidence → closure/handover, chứ không chỉ là thao tác trên AWS Console.”

### Monitoring

> “Khi nhận alarm em không xử lý chỉ dựa trên một metric. Em xác nhận resource, time window, impact và recent change, sau đó correlate metrics, logs và application health trước khi quyết định remediation hoặc escalation.”

### Security

> “Với access management em ưu tiên least privilege, temporary access khi phù hợp, MFA, approval và audit trail.”

### Release

> “Release phải có approved version, validation criteria và rollback plan trước khi production deployment.”

### Evidence

> “Một operation chưa hoàn tất nếu chỉ thay đổi thành công nhưng không có validation và evidence để chứng minh đã thực hiện đúng.”

Nếu bạn nói được tự nhiên 5 ý này, interviewer sẽ thấy bạn có **operations mindset**, ngay cả khi hands-on tool-specific chưa nhiều.

---

# 65. Chỗ nào trong danh sách này bạn KHÔNG cần cày sâu

Vì syllabus mới là nội dung phỏng vấn chính, hiện tại **không đáng dành thời gian** học sâu:

```text
Terraform modules nâng cao
Terraform provider development
Jenkins Shared Library
Groovy nâng cao
AD replication internals
Kerberos protocol packets
RDS engine internals
CloudWatch OpenTelemetry nâng cao
VBA COM internals
Windows AccessCheck API internals
ServiceNow development
ServiceNow scripting
```

Chỉ cần concept + operation workflow.

---

# 66. Kế hoạch ôn cấp tốc mình đề xuất

Nếu còn khoảng **3–4 giờ**, chia như sau:

**Giờ 1:** AWS architecture → VPC → EC2 → SG/NACL → IAM → RDS → CloudWatch.

**Giờ 2:** Operations flow → ServiceNow → Incident/Change → Jenkins → release → rollback → evidence.

**Giờ 3:** IAM 90-day review → PAM → Terraform/CloudFormation → AD.

**30–60 phút cuối:** VBA/CSV/encoding/Outlook Object Model + opswitch + internal tools, sau đó chỉ luyện **trả lời miệng scenario**.

Phần quan trọng nhất không phải đọc thêm. Đó là tập trả lời:

```text
What is it?
Why is it used?
How does the operation flow work?
How would I verify it?
What would I do if it failed?
When would I escalate?
```

Đây cũng đúng với mục tiêu cuối của syllabus: **configure → validate → monitor → troubleshoot → rollback/recover → document → escalate → hand over**, thay vì chỉ thuộc định nghĩa. 00_MASTER_SYLLABUS_Java_Fresher…

Nếu phải chọn đúng **một bước tiếp theo trước phỏng vấn**, mình khuyên chuyển ngay sang **mock interview 30–40 câu dựa chính xác trên syllabus + danh sách skill này**, vì phần kiến thức bổ sung ở trên đã đủ breadth cho fresher.
