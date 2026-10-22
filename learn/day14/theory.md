# Day 14 — Module Lý thuyết

## Integrated Infrastructure Operations, Maintenance & Final Technical Readiness

Day 14 là phần **capstone theory** của toàn bộ chương trình. Theo syllabus, phạm vi lý thuyết phải hợp nhất Linux/RHEL, Windows Server, AWS, HPE/iLO, Nutanix, HYCU, Cisco, FortiGate, BIG-IP, centralized logging, ManageEngine analyzers, ServiceNow, PalletControl, privileged access, IAM review, vulnerability handling, automation, Outlook/MAPI, baselines, monitoring, recovery, security, evidence, escalation và handover thành **một mô hình vận hành enterprise thống nhất**. Infrastructure_Operations_14_Da…

Điểm quan trọng nhất: Day 14 **không phải học lại Day 1–13 theo kiểu từng product độc lập**. Mục tiêu là chuyển từ tư duy “biết dùng từng tool” sang tư duy:

```text
Business Service
      ↓
Application
      ↓
Load Balancer
      ↓
Firewall / Network
      ↓
Server / VM
      ↓
Hypervisor / HCI
      ↓
Physical Infrastructure
      ↓
Storage / Backup / DR

      ↕
IAM / Privileged Access
Monitoring / Logging
ITSM / Change Control
Asset / CMDB
Automation
Audit Evidence
```

Khi một business service xảy ra sự cố, Infrastructure Operations engineer phải tìm được **fault domain**, chứng minh bằng evidence, biết mình được phép làm gì, thực hiện corrective action an toàn, verify recovery, rollback nếu cần, và bàn giao/escalate phần còn lại.

Phần dưới đây chia thành hai lớp:

**Theo syllabus:** những năng lực Day 14 bắt buộc phải cover.

**Kiến thức bổ sung:** cách kết nối chúng thành workflow vận hành thực tế. Phần bổ sung không thay đổi mục tiêu syllabus và không giả định product/version-specific behavior chưa được xác minh.

---

# 1. Integrated Infrastructure Operations — tư duy cốt lõi của Day 14

## 1.1 Integrated Operations là gì?

Trong thực tế enterprise, một service hiếm khi chỉ phụ thuộc vào một server.

Ví dụ người dùng truy cập:

```text
https://app.example.internal
```

Có thể tồn tại dependency:

```text
Client
  │
  ├── DNS
  │
  ▼
Network / VLAN / Routing
  │
  ▼
FortiGate
  │
  ▼
BIG-IP Virtual Server
  │
  ▼
Pool / Pool Member
  │
  ▼
Linux / Windows VM
  │
  ▼
Application
  │
  ▼
Database / Storage
```

Nhưng VM lại phụ thuộc:

```text
VM
 ↓
AHV
 ↓
Nutanix AOS
 ↓
CVM
 ↓
Physical Nutanix Node
```

Physical node có thể liên quan:

```text
HPE ProLiant
 ↓
CPU / Memory / Storage Controller / Disk / NIC
 ↓
iLO health
```

Đồng thời toàn bộ service còn phụ thuộc:

```text
IAM
Privileged Access
Monitoring
Centralized Logging
Backup
ServiceNow Change
Asset Inventory
Certificate lifecycle
Automation
```

Do đó symptom:

> “Website không truy cập được”

hoàn toàn chưa phải root cause.

Root cause có thể là:

```text
Application stopped
Wrong filesystem permission
Disk full
Wrong DNS
Wrong route
Missing VLAN
Firewall policy mismatch
NAT issue
BIG-IP pool member down
Health monitor mismatch
Expired certificate
AHV VM powered off
Nutanix capacity problem
Physical hardware fault
Unauthorized IAM change
```

Day 14 dạy bạn **localize** thay vì đoán.

---

# 2. Operational Thinking Model

Một Infrastructure Operations engineer tốt nên suy nghĩ theo chuỗi:

```text
1. What changed?
2. What is the exact symptom?
3. What is the business impact?
4. What is the expected state?
5. What is the actual state?
6. Where does expected diverge from actual?
7. What evidence supports that?
8. What is the least-risk next check?
9. Am I authorized to change anything?
10. How will I verify success?
11. How will I roll back?
12. When must I escalate?
```

Đây là tư duy xuyên suốt toàn bộ Day 14.

Một lỗi phổ biến của fresher là:

```text
Symptom
  ↓
Guess
  ↓
Change configuration
```

Workflow an toàn hơn là:

```text
Symptom
   ↓
Establish scope
   ↓
Read-only verification
   ↓
Collect evidence
   ↓
Form hypothesis
   ↓
Test hypothesis
   ↓
Identify fault domain
   ↓
Authorization check
   ↓
Controlled corrective action
   ↓
Post-check
   ↓
Evidence
   ↓
Close / Escalate
```

---

# 3. Operational Safety và Change Discipline

Day 14 coi **operational safety** là điều kiện trước mọi thay đổi.

Không phải command chạy được nghĩa là bạn được phép chạy.

Cần phân biệt ba khái niệm:

```text
Technical Capability
≠
Permission
≠
Authorization
```

Ví dụ:

Bạn có `sudo`.

Điều đó chứng minh bạn có technical privilege.

Nó **không chứng minh** bạn có approved change để restart production database.

Tương tự:

Bạn là FortiGate administrator.

Không có nghĩa bạn được tự sửa production firewall policy.

High-risk activities trong syllabus như production firewall changes, firmware updates, hyperconverged cluster maintenance và privileged-account operations phải có authorization, rollback planning và supervision thích hợp. Infrastructure_Operations_14_Da…

---

# 4. Read-only First Principle

Trong troubleshooting, luôn ưu tiên:

```text
Observe
→ Verify
→ Correlate
→ Change
```

thay vì:

```text
Change
→ See what happens
```

Ví dụ Linux:

```bash
systemctl status myapp
ss -lntp
ip route
df -h
journalctl -u myapp
```

đều chủ yếu là read-only inspection.

Trong khi:

```bash
systemctl restart myapp
```

là state-changing operation.

Một operational runbook tốt phải phân biệt hai loại command này.

---

# 5. Linux/RHEL Integrated Administration

## 5.1 Vai trò của Linux/RHEL trong Day 14

Ở Day 1–2 bạn đã học từng thành phần. Day 14 yêu cầu tích hợp chúng để trả lời:

> Linux server này có thực sự là nguyên nhân của service degradation hay không?

Health model nên xem xét:

```text
Host
├── OS
├── CPU
├── Memory
├── Process
├── systemd service
├── Port/listener
├── Network
├── DNS
├── Firewall
├── SELinux
├── Storage
├── Filesystem
├── Permissions
└── Logs
```

---

## 5.2 Service health

Bạn cần phân biệt:

```text
Service unit state
Process existence
Port listening
Application responsiveness
```

Ví dụ:

```bash
systemctl status nginx
```

có thể báo:

```text
active (running)
```

nhưng điều đó chưa chứng minh application hoàn toàn healthy.

Tiếp tục có thể cần:

```bash
ss -lntp
curl http://127.0.0.1
```

Workflow:

```text
systemd
   ↓
process
   ↓
socket
   ↓
local application test
   ↓
remote test
```

---

# 6. Linux Resource Troubleshooting

Các resource cần kiểm tra gồm:

```text
CPU
memory
disk capacity
inode
filesystem mount
I/O symptoms
process resource usage
```

Ví dụ rất phổ biến:

```text
Application failure
     ↓
Log says cannot write file
     ↓
df -h shows filesystem 100%
```

Nhưng cũng cần kiểm tra:

```bash
df -i
```

vì filesystem có thể còn disk space nhưng hết inode.

### Common mistake

Chỉ chạy:

```bash
df -h
```

rồi kết luận storage healthy.

---

# 7. Linux Permission và SELinux

Một application có thể fail dù Unix permissions trông đúng vì:

```text
filesystem permissions
+
ownership
+
SELinux context/policy
```

đều ảnh hưởng access.

Không được chữa SELinux problem bằng cách mặc định:

```bash
setenforce 0
```

và coi đó là solution.

Đó chỉ có thể là diagnostic action trong authorized lab nếu runbook cho phép.

Cần tìm root cause:

```text
Incorrect context?
Incorrect policy?
Incorrect directory permission?
Wrong service account?
```

---

# 8. Linux Networking

Kiểm tra theo layer:

```text
Interface
 ↓
IP address
 ↓
Route
 ↓
DNS
 ↓
Port
 ↓
Application
```

Một sequence read-only điển hình:

```bash
ip addr
ip route
ss -lntp
getent hosts example.internal
ping <gateway>
curl -v https://service.example
```

Không phải mọi environment đều cho phép ICMP nên `ping` failure không tự động đồng nghĩa network unavailable.

---

# 9. Linux Logs

Day 14 yêu cầu coi log như **evidence**, không phải chỉ để xem lỗi.

Có thể cần:

```bash
journalctl
journalctl -u <service>
```

và các log files phù hợp môi trường.

Cần ghi nhận:

```text
hostname
timezone
collection time
query start
query end
filter
command
output
```

Một event không tồn tại trong query chưa chắc chứng minh event chưa từng xảy ra. Log có thể đã:

```text
rotate
expire
fail to collect
be permission restricted
fall outside time window
```

---

# 10. Windows Server Integrated Administration

Cách suy nghĩ tương tự Linux nhưng objects khác:

```text
Windows Server
├── Services
├── Processes
├── Event Logs
├── NTFS permissions
├── Share permissions
├── Registry
├── Windows Defender Firewall
├── Network
├── DNS
├── Storage
├── Scheduled Tasks
├── PowerShell
├── RDP / WinRM
└── Update / reboot state
```

---

# 11. Windows Service Health

Một Windows service có thể:

```text
Running
Stopped
Starting
Stopping
Paused
```

Nhưng cũng như Linux, trạng thái `Running` không đủ chứng minh business function healthy.

Bạn cần correlate:

```text
Service state
+
Process
+
Port
+
Application check
+
Event Logs
```

PowerShell read-only example:

```powershell
Get-Service
Get-Process
Get-NetTCPConnection
```

---

# 12. Windows Permission Model

Trong file access scenario phải nhớ:

```text
Effective access
=
identity
+
group membership
+
NTFS permissions
+
share permissions
+
possibly policy/security controls
```

Một user có NTFS permission vẫn có thể fail qua SMB nếu share permission hạn chế hơn.

Operational troubleshooting không được chỉ nhìn một ACL duy nhất.

---

# 13. Windows Event Logs

Các channel phổ biến:

```text
System
Application
Security
```

nhưng availability và auditing phụ thuộc configuration.

Một event cần được đánh giá qua:

```text
Provider
Event ID
Record ID
Computer
Timestamp
Level
Message
```

Không nên screenshot vài dòng Event Viewer rồi xem đó là complete audit evidence nếu native export có thể thực hiện.

---

# 14. Linux và Windows — common troubleshooting framework

Cả hai OS có thể dùng chung mô hình:

```text
Host reachable?
   ↓
OS healthy?
   ↓
Required service running?
   ↓
Required process alive?
   ↓
Required port listening?
   ↓
Local access works?
   ↓
Remote access works?
   ↓
Authentication works?
   ↓
Authorization works?
   ↓
Application works?
```

Điều này giúp tránh jump layer.

---

# 15. AWS Integrated Operations

Theo syllabus Day 14 phải tích hợp:

```text
IAM
VPC
EC2
EBS
S3
CloudWatch
CloudTrail
```

thành một operational dependency model. Infrastructure_Operations_14_Da…

---

# 16. AWS Identity Context First

Trước khi thao tác AWS, câu hỏi đầu tiên nên là:

```text
Which account?
Which identity?
Which role?
Which Region?
```

Một thao tác đúng command nhưng chạy nhầm:

```text
account
role
Region
```

có thể gây incident nghiêm trọng.

AWS CLI identity verification thường dựa trên STS, chẳng hạn:

```bash
aws sts get-caller-identity
```

Nó giúp xác nhận identity context, nhưng không tự động chứng minh change được business-authorized.

---

# 17. AWS IAM Reasoning

IAM operational troubleshooting cần phân biệt:

```text
Authentication
Authorization
Resource policy
Identity policy
Role trust
Temporary credentials
Application behavior
```

Một `AccessDenied` không tự động nghĩa IAM user “không có quyền”.

Có thể liên quan nhiều control khác nhau.

Nguyên tắc:

```text
Denied action
→ identify principal
→ identify resource
→ identify action
→ inspect applicable policy context
→ determine intended authorization
```

Không giải quyết mọi `AccessDenied` bằng cách attach broad administrator permissions.

---

# 18. AWS VPC Dependency

Một EC2 network path có thể phụ thuộc:

```text
ENI
↓
Subnet
↓
Route table
↓
Security Group
↓
NACL
↓
Gateway / destination
```

Ví dụ Internet access:

```text
EC2
 ↓
Subnet route table
 ↓
Internet Gateway
```

hoặc private egress design:

```text
Private EC2
 ↓
Private subnet
 ↓
NAT path
 ↓
Internet
```

Cần phân biệt **routing problem** với **security filtering problem**.

---

# 19. EC2 Operational Health

EC2 check không chỉ là:

```text
Running
```

Bạn cần nghĩ tới:

```text
instance state
status checks
network
guest OS
application
attached IAM role
attached storage
monitoring
bootstrap/user-data outcome
```

Một EC2 `running` nhưng service stopped vẫn là application outage.

---

# 20. EBS Operational Thinking

EBS layer gồm:

```text
Volume
 ↓
Attachment
 ↓
OS block device
 ↓
Partition/filesystem
 ↓
Mount
 ↓
Application path
```

Nếu application báo missing data, đừng mặc định snapshot mất dữ liệu.

Có thể volume:

```text
attached
but not mounted
```

hoặc mount vào sai path.

---

# 21. S3 Operational Thinking

Các control chính đã học:

```text
IAM
bucket policy
Block Public Access
encryption
versioning
lifecycle
retention
replication
backup/recovery
```

Ở Day 14, mục tiêu quan trọng là hiểu:

```text
Object exists
≠
Application can recover it
```

Recovery còn phụ thuộc:

```text
permissions
version selection
encryption key availability
destination
application readability
integrity
```

Syllabus nhấn mạnh S3/HYCU successful backup không tự động chứng minh DR readiness; cần isolated restoration, integrity validation và observed recovery time. Infrastructure_Operations_14_Da…

---

# 22. CloudWatch vs CloudTrail

Đây là distinction quan trọng.

### CloudWatch

Chủ yếu dùng cho operational observability:

```text
metrics
logs
alarms
application/system monitoring
```

### CloudTrail

Chủ yếu dùng để điều tra AWS API/management activity trong phạm vi được cấu hình và retained.

Do đó:

```text
CPU high?
→ CloudWatch
```

trong khi câu hỏi:

```text
Who changed this resource?
```

thường liên quan CloudTrail/audit trail.

Không được coi hai service là interchangeable.

---

# 23. HPE ProLiant / iLO Health Management

Day 14 yêu cầu biết phân biệt **in-band** và **out-of-band management**.

### In-band

Management thông qua OS/hypervisor đang chạy trên server.

### Out-of-band

iLO có management plane riêng.

Do đó ngay cả khi:

```text
OS unreachable
```

iLO vẫn có thể cho biết:

```text
power state
hardware inventory
health status
IML
hardware alerts
```

nếu management plane còn hoạt động.

---

# 24. Hardware fault isolation

Một failure có thể nằm ở:

```text
Application
OS
Hypervisor
NIC
Storage controller
Disk
Memory
Power
Cooling
Physical host
```

Day 14 đòi hỏi Infrastructure Operations engineer không tiếp tục restart software nếu hardware evidence chỉ ra physical fault.

Ví dụ:

```text
VM unavailable
+
host reports storage/controller warning
+
iLO contains hardware alert
```

lúc này hardware escalation có thể phù hợp hơn việc tiếp tục restart guest VM.

---

# 25. Firmware Awareness

Firmware là high-risk domain.

Bạn cần hiểu:

```text
current version
target version
compatibility
maintenance prerequisites
dependency
reboot requirement
vendor support
backup/config export
stop condition
```

Không được suy luận:

> “Version mới hơn = nên upgrade ngay.”

Upgrade phải dựa approved compatibility matrix/runbook cho environment thực tế.

---

# 26. Nutanix Architecture trong Day 14

Các layer cần phân biệt:

```text
Prism Central
     │
Prism Element
     │
AOS / CVM services
     │
AHV
     │
VM
```

cộng:

```text
physical node
storage
virtual networking
```

---

# 27. Prism Element vs Prism Central

Operationally cần hiểu management scope.

Khái niệm chung:

**Prism Element** thiên về quản lý cluster cụ thể.

**Prism Central** cung cấp management tập trung ở deployment sử dụng nó.

Tuy nhiên feature chính xác phải được xác minh theo deployed version/license.

Day 14 không yêu cầu nhớ mọi menu. Quan trọng là xác định:

```text
Which management plane owns the object I am investigating?
```

---

# 28. AHV

AHV là virtualization layer.

Các object cần nhìn:

```text
VM
vCPU
memory
virtual disk
NIC
network
host placement
power state
```

Một guest outage có thể nằm ở:

```text
guest OS
VM config
network assignment
host
cluster
storage
```

Không được mặc định VM “down” là guest OS issue.

---

# 29. AOS/CVM

CVM/AOS liên quan storage và cluster services.

Khi troubleshoot phải phân biệt:

```text
VM workload issue
vs
AHV host issue
vs
CVM/service issue
vs
cluster capacity/resiliency issue
```

Đây chính là fault-domain thinking mà Day 14 muốn kiểm tra.

---

# 30. Nutanix Capacity

Capacity không chỉ hỏi:

```text
Free GB còn bao nhiêu?
```

Cần nghĩ:

```text
current utilization
trend
resiliency requirement
maintenance headroom
workload placement
```

Một cluster còn free capacity chưa chắc đủ safe capacity cho host maintenance.

---

# 31. HYCU Backup & Recovery

Backup architecture phải hiểu qua:

```text
Source
 ↓
Protection policy
 ↓
Backup job
 ↓
Backup target
 ↓
Recovery point
 ↓
Restore
 ↓
Validation
```

---

# 32. Backup Success không bằng Recovery Success

Giả sử dashboard cho thấy:

```text
Backup job: Success
```

Bạn vẫn chưa biết:

```text
recovery point usable?
credentials valid?
target readable?
encryption dependencies available?
restore works?
application can read restored content?
```

Do đó Day 14 nhấn mạnh **recovery assurance**.

---

# 33. RPO và RTO

### RPO — Recovery Point Objective

Liên quan lượng dữ liệu có thể mất theo time.

Ví dụ:

```text
Latest recoverable point = 09:00
Incident = 09:30
```

potential data gap khoảng:

```text
30 minutes
```

nhưng policy-specific RPO phải được định nghĩa trước.

### RTO — Recovery Time Objective

Liên quan thời gian mục tiêu để restore service.

Day 14 quan tâm không chỉ target RTO mà cả:

```text
observed recovery time
```

trong controlled restore.

---

# 34. Cisco Catalyst/Nexus L2/L3 Connectivity

Day 14 không yêu cầu biến bạn thành Network Engineer chuyên sâu; mục tiêu là Infrastructure Operations có khả năng localize L2/L3 fault.

Flow:

```text
Physical/link
   ↓
Interface
   ↓
VLAN
   ↓
MAC learning
   ↓
ARP
   ↓
Gateway
   ↓
Routing
   ↓
Destination
```

---

# 35. Layer 2 Thinking

Các thành phần:

```text
VLAN
access port
trunk
802.1Q
MAC table
STP
port-channel
```

Nếu hai host cùng subnet không communicate, kiểm tra L2 trước khi bắt đầu thay static route.

Common root causes:

```text
wrong VLAN
trunk VLAN missing
port down
port-channel issue
STP state
incorrect access VLAN
```

---

# 36. Layer 3 Thinking

Các thành phần:

```text
IP
subnet
default gateway
ARP
SVI/routed interface
routing table
static route
next hop
```

Một route tồn tại không tự động chứng minh end-to-end path hoạt động.

Phải xem:

```text
forward path
+
return path
```

---

# 37. Catalyst vs Nexus

Điểm syllabus muốn giữ là **không giả định IOS/IOS XE và NX-OS giống nhau**.

Concept L2/L3 tương tự, nhưng:

```text
syntax
defaults
features
management behavior
release-specific details
```

có thể khác.

Day 14 cần giữ platform awareness và version awareness.

---

# 38. FortiGate Firewall Operations

Một packet path qua firewall có thể được suy nghĩ như:

```text
Incoming interface
      ↓
Routing / policy context
      ↓
Firewall policy
      ↓
Service/object match
      ↓
NAT
      ↓
Outgoing path
      ↓
Return traffic/session
```

Operationally cần xác định:

```text
Does packet reach firewall?
Which route?
Which policy?
Is policy matched?
Is session created?
Is NAT expected?
What does traffic log show?
```

---

# 39. Firewall policy troubleshooting

Sai lầm phổ biến:

> “Firewall đang block.”

nhưng chưa có evidence.

Bạn phải chứng minh bằng:

```text
expected source
expected destination
expected protocol/port
policy
session/log
routing
```

Nếu firewall không hề thấy packet thì root cause có thể nằm upstream.

---

# 40. Policy Ordering

Với platform dùng ordered policy evaluation, placement của rule có thể ảnh hưởng behavior.

Do đó không chỉ hỏi:

```text
Is there an allow rule?
```

mà còn:

```text
Is the expected rule actually selected?
```

Cần kiểm tra behavior thực tế qua policy/session/log phù hợp.

---

# 41. NAT

Một service failure có thể phát sinh vì:

```text
wrong source NAT
wrong destination translation
unexpected address
return path problem
```

NAT troubleshooting nên được thực hiện sau khi đã hiểu expected traffic flow.

Không thay NAT chỉ vì address trên backend khác address client truy cập.

---

# 42. BIG-IP Load Balancing

Một BIG-IP service path điển hình:

```text
Client
 ↓
Virtual Server
 ↓
Profiles / SSL
 ↓
Pool
 ↓
Pool Member
 ↓
Backend Application
```

---

# 43. Virtual Server

Virtual server thường đại diện listener mà client kết nối.

Cần hiểu:

```text
destination
port
protocol
profiles
pool association
```

Nếu client không reach virtual server thì chưa nên bắt đầu sửa backend application.

---

# 44. Pool và Pool Member

Pool là nhóm backend targets.

Pool member health có thể là:

```text
available
unavailable
unknown
```

tùy platform/version/UI representation.

Quan trọng hơn tên trạng thái là lý do:

```text
health monitor failed?
network unreachable?
service closed?
application returns unexpected result?
```

---

# 45. Health Monitor

Health monitor failure không nhất thiết server chết.

Có thể:

```text
backend alive
but expected URL wrong
```

hoặc:

```text
HTTP status unexpected
```

hoặc:

```text
TLS handshake fails
```

hoặc:

```text
monitor timeout too short
```

Do đó phải xác định monitor đang test **cái gì**.

---

# 46. TLS Troubleshooting

Một HTTPS failure nên tách:

```text
DNS
 ↓
TCP
 ↓
TLS
 ↓
HTTP
 ↓
Application
```

Đừng gộp tất cả thành “SSL problem”.

TLS verification cần xem:

```text
certificate validity
hostname
expiry
chain
trust
SNI
protocol compatibility
```

---

# 47. Client-side và Server-side TLS

Trong load-balancer design, có thể tồn tại:

```text
Client
  │ TLS
  ▼
BIG-IP
  │ HTTP hoặc TLS
  ▼
Backend
```

Do đó certificate problem có thể xảy ra ở:

```text
client-facing side
```

hoặc:

```text
backend-facing side
```

Hai failure domain khác nhau.

---

# 48. Không bypass TLS validation để “làm cho pass”

Một anti-pattern rất quan trọng:

```bash
curl -k ...
```

rồi kết luận HTTPS healthy.

`-k`/insecure mode bỏ qua certificate validation.

Nó có thể hữu ích rất hạn chế trong controlled diagnosis để phân biệt trust issue, nhưng **không được dùng như evidence rằng TLS validation pass**.

Syllabus yêu cầu retained TLS validation trong automation test và negative TLS cases phải tạo failed assertion rõ ràng. Infrastructure_Operations_14_Da…

---

# 49. Centralized Log Investigation

Centralized logging là lớp nối nhiều domain lại với nhau.

Các nguồn Day 14 có thể liên quan:

```text
Linux
Windows
AWS
FortiGate
BIG-IP
Cisco
Nutanix
HPE
ServiceNow
ManageEngine
```

Mục tiêu không phải gom thật nhiều logs mà là tạo một **defensible timeline**.

---

# 50. Timestamp và Timezone

Một incident có thể có:

```text
Linux → UTC
Windows → local time
Firewall → local timezone
AWS → UTC
ServiceNow → displayed user timezone
```

Nếu không normalize time, bạn có thể suy luận sai sequence.

Ví dụ:

```text
10:05 firewall event
09:06 server event
```

không thể so trực tiếp nếu timezone khác nhau.

Evidence nên luôn ghi:

```text
timestamp
timezone
```

---

# 51. Correlation không có nghĩa causation

Nếu hai event xảy ra gần nhau:

```text
Firewall policy change at 10:00
Application failure at 10:01
```

đó là correlation mạnh để investigate, nhưng chưa tự động chứng minh policy change gây outage.

Phải kiểm tra thêm:

```text
traffic path
policy hit
session/log
rollback result
```

---

# 52. “No Logs” không bằng “No Activity”

Đây là một concept cực kỳ quan trọng.

Không tìm thấy event có thể do:

```text
No event occurred
Source disconnected
Collector failed
Parser failed
Wrong query
Wrong timezone
Retention expired
Insufficient permission
Wrong log channel
```

Do đó:

```text
No matching records
```

là observation.

Không nên biến observation đó thành:

```text
User definitely did nothing
```

---

# 53. EventLog Analyzer

Day 14 dùng EventLog Analyzer theo hướng operational investigation.

Phải hiểu ba layer:

```text
Source generated event?
      ↓
Collector received event?
      ↓
Analyzer indexed/searchable event?
```

Nếu search không thấy event, cần phân biệt ba trạng thái trên.

---

# 54. Ingestion Health

Một log source “đã onboard” chưa chắc đang healthy.

Cần nhìn:

```text
last event time
expected event
ingestion delay
collector health
source identity
parsing
retention
```

Một silent source có thể nghiêm trọng vì monitoring/audit blind spot.

---

# 55. Firewall Analyzer

Firewall Analyzer trong Day 14 được sử dụng để hiểu:

```text
traffic
deny events
flows
rule usage
configuration changes
reports
```

Nhưng report tool không thay thế firewall ground truth.

Nếu report và FortiGate real-time observation mâu thuẫn, phải investigate:

```text
ingestion delay?
filter?
time range?
source mapping?
```

---

# 56. ServiceNow như operational control plane

ServiceNow không chỉ là “nơi viết ticket”.

Trong Day 14, ServiceNow giúp liên kết:

```text
Business request
      ↓
Authorization
      ↓
Technical action
      ↓
Evidence
      ↓
Result
      ↓
Closure / Handover
```

---

# 57. Incident vs Change

### Incident

Mục tiêu chính:

```text
restore normal service
```

### Change

Mục tiêu:

```text
control modification of environment
```

Một incident có thể dẫn tới change.

Ví dụ:

```text
Incident:
website unavailable
```

Investigation xác định:

```text
incorrect firewall configuration
```

Corrective action có thể phải đi qua approved change process.

---

# 58. Work Notes và Customer-visible Communication

Operational records phải phân biệt:

```text
technical investigation notes
```

và:

```text
customer-facing communication
```

Technical notes có thể bao gồm:

```text
commands
hypotheses
logs
internal escalation
```

Customer comments thường nên tập trung:

```text
impact
status
ETA only when established
resolution information
```

Không expose sensitive configuration/credentials.

---

# 59. Implementation Plan và Backout Plan

Một change tốt cần trả lời:

```text
What will change?
Who performs it?
When?
What are prechecks?
What proves success?
What triggers rollback?
How is rollback performed?
What proves rollback succeeded?
```

Backout plan:

```text
"Restore if problem"
```

là quá mơ hồ.

---

# 60. PalletControl Asset Maintenance

Theo syllabus, PalletControl phải được hiểu trong context organization's implementation.

Không được giả định:

```text
specific menu
specific agent behavior
specific API
specific inventory feature
```

nếu actual deployment chưa xác minh. Infrastructure_Operations_14_Da…

Core concept là **asset reconciliation**.

---

# 61. Asset Inventory vs Observed Reality

Bạn có thể có ba nguồn:

```text
PalletControl
CMDB
Actual server/cloud inventory
```

và chúng có thể không khớp.

Ví dụ:

```text
Actual server: srv-app-01 exists
CMDB: exists
PalletControl: missing
```

Không tự động kết luận asset phải thêm.

Có thể:

```text
collection issue
agent issue
asset decommission in progress
naming mismatch
duplicate identity
```

Phải investigate trước.

---

# 62. Asset data quality

Các vấn đề phổ biến:

```text
Missing asset
Duplicate asset
Stale asset
Wrong owner
Wrong status
Old hostname
Wrong environment
Outdated software information
```

Asset record có operational impact vì nó liên quan:

```text
ownership
incident routing
patching
vulnerability
IAM
cost
audit
```

---

# 63. Privileged-access Governance

Privileged access gồm nhiều lớp:

```text
Identity
     ↓
Approval
     ↓
Entitlement
     ↓
Access
     ↓
Session/activity
     ↓
Expiry/revocation
     ↓
Audit
```

Day 14 không chỉ hỏi “login được không”.

---

# 64. Least Privilege

Least privilege nghĩa là cấp:

```text
minimum required access
for required scope
for required duration
```

Không phải:

```text
everyone gets read-only
```

và cũng không phải:

```text
administrator forever because operations may need it someday
```

---

# 65. Entitlement vs Activity

Cần phân biệt:

```text
User CAN perform action
```

với:

```text
User DID perform action
```

IAM permission chứng minh entitlement.

Audit/activity evidence mới liên quan actual use.

Đây là distinction quan trọng khi review privileged accounts.

---

# 66. Temporary Privilege

Một access workflow tốt có thể gồm:

```text
request
approval
grant
use
expiry
revoke
verify revocation
```

Việc `expiry time` được ghi trong ticket chưa chứng minh access thực sự bị revoke.

Cần verification.

---

# 67. Break-glass Account

Break-glass account tồn tại cho emergency scenario.

Nó thường cần controls đặc biệt:

```text
strict ownership
restricted usage
monitoring
credential protection
post-use review
```

Không nên classify là dormant normal account chỉ vì ít được dùng.

---

# 68. 90-Day IAM Review

Syllabus Day 14 yêu cầu condensed IAM/access certification dựa trên các nguyên tắc Day 13.

Cần hiểu:

```text
review cadence
≠
inactivity threshold
```

Ví dụ:

```text
Review every 90 days
```

không có nghĩa:

```text
disable every account unused for 90 days
```

trừ khi policy định nghĩa như vậy.

---

# 69. Review boundary

Một IAM review defensible cần định nghĩa:

```text
as-of timestamp
timezone
review start
review end
inclusive/exclusive rules
policy threshold
```

Nếu không, account hoạt động ngay đúng boundary có thể bị classify không nhất quán.

---

# 70. Unknown last-used evidence

Một field last-used blank không đồng nghĩa:

```text
never used
```

Nó có thể nghĩa:

```text
data unavailable
unsupported
outside evidence scope
not collected
```

Do đó classification đúng có thể là:

```text
Unknown
```

thay vì:

```text
Dormant
```

---

# 71. IAM Exception Categories

Day 14 có thể gặp:

```text
orphaned account
missing owner
stale entitlement
expired approval
privilege mismatch
duplicate identity
revoked-but-active access
unknown activity
```

Mỗi exception cần decision như:

```text
Keep
Revoke
Modify
Investigate
```

với rationale.

---

# 72. Vulnerability Handling

Vulnerability handling không chỉ là đọc CVSS.

Risk reasoning cần xem:

```text
Technical severity
       +
Exploitability
       +
Exposure
       +
Asset criticality
       +
Business impact
       +
Existing controls
```

---

# 73. CVE và CVSS

### CVE

Identifier cho một publicly known vulnerability khi applicable.

### CVSS

Scoring framework giúp thể hiện technical severity.

Nhưng:

```text
CVSS
≠
final business remediation priority
```

Ví dụ một high-severity vulnerability trên:

```text
isolated non-production host
```

có thể có business risk khác một medium-severity vulnerability trên:

```text
internet-facing critical service
```

---

# 74. Vulnerability vs Misconfiguration

Không phải security finding nào cũng là CVE.

Có thể là:

```text
CVE vulnerability
Misconfiguration
IAM exception
Operational incident
Policy violation
```

Classification đúng rất quan trọng để assign ownership đúng.

---

# 75. Compensating Controls

Nếu chưa remediation ngay, có thể tồn tại compensating control như:

```text
network isolation
access restriction
WAF/firewall control
service disablement
enhanced monitoring
```

Tuy nhiên compensating control cần được đánh giá và approved, không được tự xem là equivalent fix.

---

# 76. Remediation Verification

“Ticket resolved” không chứng minh vulnerability fixed.

Phải retest:

```text
Before:
vulnerable state
```

sau change:

```text
After:
fixed/not exposed/mitigated state
```

và giữ evidence.

---

# 77. VBA/Python/Batch/PowerShell Automation

Day 14 không chỉ kiểm tra script output.

Nó kiểm tra **operational reliability**.

Mô hình automation:

```text
Input
 ↓
Validation
 ↓
Normalization
 ↓
Business Logic
 ↓
Output
 ↓
Verification
 ↓
Evidence
```

---

# 78. Input Contract

Script phải biết input kỳ vọng:

```text
required columns
encoding
delimiter
path
data types
ID rules
```

Nếu input sai, script nên:

```text
fail clearly
```

hoặc:

```text
quarantine invalid records
```

thay vì âm thầm produce output sai.

---

# 79. IDs phải được xử lý như text

Ví dụ ID:

```text
001234
```

nếu Excel/script biến thành:

```text
1234
```

identity reconciliation có thể sai.

Tương tự long IDs có thể bị scientific notation hoặc precision issue nếu xử lý numeric.

Do đó:

```text
identifiers ≠ quantities
```

---

# 80. Normalization

Normalization có thể xử lý:

```text
leading/trailing whitespace
case
full-width/half-width rules
line endings
approved Unicode rules
```

Nhưng phải giữ original data.

Đừng sửa source evidence trực tiếp.

---

# 81. Duplicate handling

Duplicate không nên được xử lý bằng:

```text
remove duplicates
```

mà không phân tích.

Cần hỏi:

```text
Duplicate identity?
Duplicate row?
Multiple legitimate entitlements?
Many-to-many join?
Data source duplication?
```

Silent deduplication có thể làm mất audit evidence.

---

# 82. Automation Exit Status

Script không nên print error rồi exit thành công.

Ví dụ Batch wrapper gọi Python:

```text
Python failed
```

nhưng wrapper trả:

```text
exit code 0
```

sẽ khiến scheduler/report nghĩ job successful.

Day 14 yêu cầu observable failure.

---

# 83. Regression Testing

Regression test trả lời:

> Sau khi sửa script, chức năng cũ có còn đúng không?

Minimum test catalog trong syllabus bao gồm identity matching, file/encoding, file/process failures, IAM boundaries, API/network/TLS, password handling, Outlook/report và repeatability/regression. Infrastructure_Operations_14_Da…

---

# 84. Expected vs Actual

Một test case cần ít nhất:

```text
Case ID
Requirement
Input/fixture
Expected result
Actual result
Pass/Fail
Defect
Retest status
```

Script:

```text
executed without exception
```

không có nghĩa:

```text
Pass
```

nếu result sai.

---

# 85. Idempotency

Operational automation lý tưởng nên biết repeated run gây gì.

Ví dụ chạy hai lần không được tự động:

```text
create duplicate tickets
duplicate user accounts
overwrite reviewer decisions
duplicate output rows
```

trừ khi đó là intentionally defined behavior.

---

# 86. PowerShell

Trong Day 14, PowerShell đặc biệt phù hợp cho:

```text
Windows administration
object pipelines
event extraction
CSV/report generation
automation
```

Quan trọng là hiểu PowerShell truyền **objects**, không chỉ text như nhiều shell pipeline truyền thống.

---

# 87. Python

Python thích hợp cho:

```text
CSV/TXT parsing
regex
JSON
hashing
file automation
normalization
structured reconciliation
logging
```

Nhưng production-quality script cần:

```text
error handling
input validation
safe path handling
logging
exit codes
tests
```

---

# 88. Batch

Batch thường dùng như wrapper:

```text
set environment
call script
handle paths
propagate exit code
schedule execution
```

Nó đặc biệt dễ lỗi với:

```text
paths containing spaces
quoting
environment variables
child exit status
```

---

# 89. VBA

VBA trong syllabus liên quan:

```text
Excel reconciliation
multi-sheet automation
exception report
legacy operational tools
```

Security concern:

```text
macro trust
signed/approved source
unsafe workbook
embedded secrets
```

Không chạy macro không rõ nguồn chỉ vì workbook “nội bộ”.

---

# 90. Outlook/MAPI Reporting

Day 14 tiếp tục tư duy Day 12:

```text
Collect evidence
 ↓
Validate attachments
 ↓
Prepare report
 ↓
Create draft
 ↓
Human verification
```

Syllabus target là compatible **classic Outlook** cho COM/MAPI workflow; new Outlook cần được đánh giá riêng. Infrastructure_Operations_14_Da…

---

# 91. Email automation safety

Automation không nên:

```text
blindly send
```

trong lab.

Phải verify:

```text
recipient
subject
attachment names
attachment paths
sensitive content
duplicate files
```

trước delivery.

---

# 92. Configuration Baselines

Baseline là documented expected state của system tại một thời điểm.

Ví dụ server baseline:

```text
hostname
OS/version
installed roles
critical services
IP
routes
firewall
storage
monitoring
backup
owners
```

Network baseline:

```text
interfaces
VLANs
trunks
routes
ACL/policy expectations
```

Cloud baseline:

```text
resource IDs
tags
security controls
roles
monitoring
backup state
```

---

# 93. Baseline không đồng nghĩa “mọi thứ đang đúng”

Baseline có thể đơn giản là:

```text
known approved state
```

Một approved baseline vẫn có thể chứa technical debt.

Do đó:

```text
matches baseline
```

không đồng nghĩa:

```text
perfectly secure
```

Nó chỉ trả lời:

```text
current state differs from approved reference?
```

---

# 94. Configuration Drift

Drift là difference giữa current state và expected/approved state.

Ví dụ:

```text
Expected SSH port configuration
Actual differs
```

hoặc:

```text
Expected firewall rule absent
```

Drift phải được:

```text
identified
classified
authorized for correction
documented
```

Không phải mọi drift đều sửa ngay.

---

# 95. Monitoring

Monitoring trả lời:

```text
What is happening now?
Is performance/availability outside expected limits?
```

Các thành phần:

```text
metric
threshold
condition
alert
notification
response
```

---

# 96. Monitoring Pipeline

Không chỉ system bị monitor có thể fail.

Monitoring itself có thể fail.

Mô hình:

```text
Source metric/event
      ↓
Agent/collector
      ↓
Transport
      ↓
Monitoring platform
      ↓
Rule
      ↓
Notification
      ↓
Human response
```

Nếu không thấy alert, có hai possibility lớn:

```text
Nothing happened
```

hoặc:

```text
Monitoring failed
```

---

# 97. Alert Verification

Một alert test nên trả lời:

```text
Was known condition generated?
Was it collected?
Did rule evaluate?
Was alert created?
Was notification delivered?
Did it clear correctly?
```

Điều này khác với chỉ kiểm tra:

```text
dashboard is green
```

---

# 98. Backup/Restore Theory

Backup workflow:

```text
Select data
 ↓
Protect
 ↓
Store
 ↓
Retain
 ↓
Monitor
 ↓
Restore
 ↓
Validate
```

Backup operational maturity thực sự được đo ở:

```text
Can we restore the correct data
within expected time
with verified integrity?
```

---

# 99. Recovery Isolation

Restore vào isolated location thường giúp tránh:

```text
overwrite production
duplicate hostname
duplicate IP
application identity conflict
domain/service conflict
```

Đặc biệt quan trọng với VM restore.

---

# 100. Integrity Validation

Recovery validation không chỉ:

```text
files exist
```

mà có thể cần:

```text
file count
hash
application readability
version
expected structure
```

Nếu source có manifest:

```text
SHA-256
```

restore có thể được so với manifest khi phù hợp.

---

# 101. Security Controls như cross-cutting concern

Security không nằm ở một box riêng.

Nó ảnh hưởng:

```text
Linux permissions
Windows ACL
AWS IAM
Firewall
TLS
Privileged access
Backup access
Logs
Automation
ServiceNow
Asset data
```

Do đó mỗi operational action phải hỏi:

```text
Who can do this?
Why?
For how long?
Is it logged?
Can we prove approval?
```

---

# 102. Least-privilege Troubleshooting

Không tăng privilege ngay để “xem có chạy không”.

Ví dụ access denied không nên chữa bằng:

```text
Administrator
root
Full Control
Allow Any
```

Nếu làm vậy, bạn mất khả năng biết permission nào thực sự cần.

Correct workflow:

```text
identify required action
identify required object
inspect effective permission
make smallest authorized adjustment
retest
```

---

# 103. Secrets Handling

Evidence không được chứa:

```text
password
private key
session token
MFA seed
API secret
plaintext privileged credential
```

Log sanitization phải áp dụng cả success path và error path.

Syllabus final acceptance nói rõ không chấp nhận exposed secrets. Infrastructure_Operations_14_Da…

---

# 104. Audit Evidence — phần quan trọng nhất của Day 14

Day 14 không chỉ hỏi:

> “Bạn đã làm đúng chưa?”

mà còn hỏi:

> “Một reviewer độc lập có chứng minh được bạn đã làm gì không?”

Syllabus đưa ra một evidence contract rất cụ thể. Infrastructure_Operations_14_Da…

Một evidence record tốt phải có các trường như:

```text
Evidence ID
Source system/product/version
Environment
Asset/account ID
Related ticket
Stable record key
Collector
Collection timestamp
Timezone
Requested time interval
Exact query/filter/command
Export format
Encoding
Filename/path
Size
Record count
SHA-256
Access classification
Retention classification
Parent evidence ID
Transformation/script version
Limitations/errors
```

---

# 105. Raw Evidence vs Working Copy

Nguyên tắc:

```text
Raw export
   │
   ├── immutable / untouched
   │
   ▼
Working copy
   │
   ▼
Filtered / transformed analysis
   │
   ▼
Reviewer deliverable
```

Không nên:

```text
export.csv
```

rồi mở, edit, delete rows và save đè.

Bạn sẽ mất original evidence.

---

# 106. Hashing đúng cách

Hash giúp phát hiện file thay đổi so với recorded baseline.

Ví dụ:

```text
Raw export SHA-256:
ABC...
```

sau đó file bị chỉnh:

```text
SHA-256:
DEF...
```

ta biết bytes đã thay đổi.

Nhưng:

```text
Hash match
```

không chứng minh:

```text
source truthful
query complete
time window correct
collector trustworthy
```

Syllabus nói rõ distinction này. Infrastructure_Operations_14_Da…

---

# 107. Record Count

Hash không thay record count.

Hai controls phục vụ mục đích khác:

```text
Hash
→ integrity of bytes
```

```text
Record count
→ completeness/reconciliation indicator
```

Nếu source query nói:

```text
105 records
```

nhưng export chỉ:

```text
100
```

phải investigate pagination/export limit/filter.

---

# 108. Screenshots

Screenshot hữu ích cho:

```text
GUI-only state
visual status
configuration context
```

nhưng thường không nên thay native exports nếu native record có sẵn.

Native records thường preserve được:

```text
IDs
timestamps
metadata
structured fields
searchability
```

tốt hơn screenshot.

---

# 109. Evidence Gap

Nếu evidence không lấy được vì:

```text
retention
permission
collector failure
unsupported field
source unavailable
```

không được invent data.

Phải ghi:

```text
Evidence gap
Reason
Impact
Owner
Reviewer decision
```

Một nguyên tắc quan trọng cuối syllabus:

> Unknown production facts must remain unknown. Infrastructure_Operations_14_Da…

---

# 110. Incident Escalation

L1/L2 không được đánh giá bằng việc “cố fix cho bằng được”.

Một engineer tốt biết **khi nào dừng**.

Có thể tiếp tục trong scope khi:

```text
known failure
known runbook
authorized action
understood blast radius
reversible change
healthy rollback path
```

---

# 111. Escalation Boundary

Nên escalate khi chẳng hạn:

```text
required access exceeds scope
root cause uncertain and change risky
critical hardware fault
cluster resiliency insufficient
possible security compromise
backup recovery uncertain
vendor-level defect suspected
production outage blast radius expanding
rollback unavailable
evidence conflicts
```

Escalation không phải failure.

Escalate đúng thời điểm là operational competence.

---

# 112. Escalation Package

Đừng gửi:

```text
"Server broken. Please check."
```

Một escalation tốt nên có:

```text
Ticket
Impact
Start time
Affected assets
Expected state
Observed state
Checks completed
Evidence
Recent changes
Hypothesis
Actions attempted
Results
Current risk
Exact help required
```

Nhờ đó L3/vendor không phải điều tra lại từ đầu.

---

# 113. Stop Conditions

Trước maintenance phải định nghĩa stop condition.

Ví dụ:

```text
If cluster health deteriorates → stop
If redundancy lost → stop
If unexpected node fault appears → stop
If backup validation fails → stop
If rollback cannot be executed → stop
```

Stop condition nên được xác định **trước change**, không phải sau khi incident xảy ra.

---

# 114. Rollback Philosophy

Rollback không phải magic undo.

Có những change dễ reversible:

```text
config file change with known-good copy
```

nhưng có change rollback rất phức tạp:

```text
firmware
schema change
cluster upgrade
data mutation
```

Do đó phải hiểu:

```text
rollback capability
```

khác:

```text
rollback assumption
```

---

# 115. Post-change Verification

“Command succeeded” không phải success criteria.

Ví dụ:

```text
systemctl restart app
```

returns success.

Bạn vẫn phải verify:

```text
service state
port
health endpoint
dependency
user transaction
monitor
logs
```

---

# 116. Operational Handover

Handover là việc truyền **operational state**, không phải kể lại lịch sử dài dòng.

Người nhận ca cần biết:

```text
What is happening?
What has been done?
What remains?
What is the risk?
Who owns it?
What should happen next?
```

---

# 117. Good Handover Structure

Một handover tốt thường chứa:

```text
Incident/change ID
Service
Impact
Timeline
Current state
Actions performed
Evidence
Outstanding faults
Workarounds
Monitoring required
Next action
Owner
Deadline/window
Escalation contact
Risk
```

---

# 118. Handover những điều chưa biết

Rất quan trọng:

Không biến:

```text
Unknown
```

thành:

```text
Assumed healthy
```

Ví dụ không access được HYCU vì lab limitation:

Handover phải ghi:

```text
HYCU recovery validation:
Not independently executed
Reason: product lab unavailable
Status: simulation only
Gap remains open
```

không được ghi:

```text
HYCU verified
```

---

# 119. Competency Matrix

Additive Day 14 yêu cầu sử dụng competency matrix để final acceptance. Infrastructure_Operations_14_Da…

Syllabus định nghĩa bốn mức evidence:

```text
H = learner executed on actual product lab
S = simulation / fixture
D = instructor demonstration
T = theory / reference only
```

Ngoài ra phải chỉ rõ:

```text
independent
or
assisted
```

và liên kết:

```text
evidence ID
assessment outcome
reviewer
```

Infrastructure_Operations_14_Da…

---

# 120. H/S/D/T không giống nhau

Ví dụ:

Bạn xem instructor configure FortiGate.

Đó là:

```text
D
```

Không phải:

```text
H
```

Bạn giải fixture exported configuration:

```text
S
```

Bạn tự thao tác actual authorized FortiGate lab:

```text
H
```

Sự phân biệt này ngăn course completion trở thành false claim về hands-on experience.

---

# 121. Product/version gaps

Day 14 phải carry forward những gap như:

```text
no PalletControl product lab
no AccessCheck access
no physical HPE hardware
only FortiGate fixture
specific Windows edition not available
specific BIG-IP release not available
```

Gap phải được ghi lại, không che giấu.

Syllabus nói rõ product/version/runbook phải được xác nhận trước khi tuyên bố product-specific proficiency. Infrastructure_Operations_14_Da…

---

# 122. Practical Evidence Map của toàn chương trình

Day 14 final review quay lại competency matrix gồm 24 skill families, bao gồm evidence về:

```text
server/ServiceNow extraction
Excel/VBA
PalletControl
ServiceNow
privileged access
IAM review
S3 recovery
RHEL
Windows Server
Prism
AOS
AHV
HYCU
HPE/iLO
FortiGate
BIG-IP
Nexus
Catalyst
EventLog Analyzer
Firewall Analyzer
Nutanix maintenance
internal automation tools
BIG-IP API/cURL/SSL
Outlook/MAPI/PowerShell
```

Infrastructure_Operations_14_Da…

Day 14 không có nghĩa thực hiện lại toàn bộ 24 full labs, nhưng final acceptance phải biết mỗi competency đã được chứng minh ở mức nào.

---

# 123. Final Technical Readiness

Đây là mục tiêu cuối cùng.

**Readiness** không đồng nghĩa:

```text
advanced expert
```

Syllabus xác định chương trình là accelerated operational-readiness program, không phải tuyên bố mastery ở tất cả products. Infrastructure_Operations_14_Da…

Technical readiness có nghĩa bạn có khả năng:

```text
understand environment
follow runbook
verify state
diagnose common faults
operate inside authorization
collect evidence
perform safe correction
verify recovery
rollback
escalate
handover
```

---

# 124. Sự khác nhau giữa “biết command” và “operationally ready”

### Người biết command

Có thể nhớ:

```bash
systemctl restart nginx
```

### Người operationally ready

Trước command sẽ hỏi:

```text
Is nginx the failing layer?
Which host?
Which environment?
Is restart approved?
What sessions will be affected?
What is pre-change state?
How do I verify success?
What if restart fails?
Which evidence should I retain?
```

Đó mới là mục tiêu Day 14.

---

# 125. End-to-End Troubleshooting Framework

Hãy ghi nhớ framework sau vì nó tổng hợp gần như toàn bộ Module Lý thuyết:

```text
                 INCIDENT
                     │
                     ▼
          Identify Business Impact
                     │
                     ▼
          Confirm Scope / Ticket
                     │
                     ▼
          Check Recent Changes
                     │
                     ▼
         Establish Expected State
                     │
                     ▼
          Read-only Observations
                     │
                     ▼
             Localize Layer
                     │
      ┌──────────────┼───────────────┐
      ▼              ▼               ▼
  Application      Network       Infrastructure
      │              │               │
   Service         VLAN/Route       VM/Host
   Process         Firewall         Storage
   Logs            BIG-IP           Hardware
      └──────────────┼───────────────┘
                     ▼
               Form Hypothesis
                     │
                     ▼
             Validate Evidence
                     │
                     ▼
              Identify Cause
                     │
                     ▼
             Authorization Check
                     │
          ┌──────────┴─────────┐
          ▼                    ▼
     Authorized              Outside
        scope                 scope
          │                    │
          ▼                    ▼
  Controlled Change        Escalation
          │
          ▼
      Verification
          │
          ▼
   Monitoring / Logs
          │
          ▼
 Evidence / Ticket Update
          │
          ▼
   Closure / Handover
```

---

# 126. Cross-Layer Example — Application Outage

Giả sử:

```text
Users report HTTPS service unavailable.
```

Một fresher có thể nghĩ:

```text
Restart server.
```

Day 14 reasoning:

```text
1. Resolve DNS.
2. Confirm destination.
3. Test TCP reachability.
4. Validate TLS handshake.
5. Inspect BIG-IP virtual server.
6. Inspect pool.
7. Inspect member health.
8. Inspect firewall path.
9. Inspect network routing.
10. Inspect server listener.
11. Inspect application/service.
12. Inspect OS resources.
13. Inspect relevant logs.
14. Check recent approved changes.
```

Nếu BIG-IP member is down nhưng server application responds locally, investigate:

```text
monitor definition
network path
TLS/backend behavior
```

thay vì reboot server.

---

# 127. Cross-Layer Example — Permission Failure

Symptom:

```text
Backup job receives Access Denied.
```

Possible domains:

```text
HYCU authorization
AWS IAM
filesystem ACL
share permission
S3 bucket policy
encryption key access
expired privileged access
```

Correct reasoning:

```text
What resource?
Which identity?
Which exact denied operation?
What policy/control applies?
Was access previously valid?
Did entitlement expire?
What change occurred?
```

Không chữa bằng broad admin privilege.

---

# 128. Cross-Layer Example — VM unavailable

Possible chain:

```text
Guest service
 ↓
Guest OS
 ↓
VM power
 ↓
AHV
 ↓
Host
 ↓
AOS/CVM
 ↓
Physical hardware
```

Evidence có thể từ:

```text
guest logs
Prism tasks
cluster alerts
CVM status
HPE/iLO
```

Day 14 yêu cầu correlate layers thay vì chỉ xem guest console.

---

# 129. Cross-Layer Example — Backup Failure

Sequence:

```text
Backup failed
   ↓
Check job details
   ↓
Check source availability
   ↓
Check target
   ↓
Check capacity
   ↓
Check permissions
   ↓
Check network
   ↓
Check policy
   ↓
Correct approved fault
   ↓
Rerun
   ↓
Perform recovery validation
```

Không kết luận fixed chỉ vì rerun becomes green.

---

# 130. Cross-Layer Example — Logging Failure

Symptom:

```text
EventLog Analyzer reports no events from server.
```

Hypotheses:

```text
No events generated
Source forwarding failed
Collector unavailable
Network blocked
Parser issue
Time/filter mismatch
Retention problem
```

Test:

```text
Generate/identify known source event
       ↓
verify source log
       ↓
verify transmission/collector
       ↓
search analyzer
       ↓
reconcile timestamp/count
```

---

# 131. Cross-Layer Example — Certificate Failure

Client reports:

```text
certificate warning
```

Possible root causes:

```text
expired certificate
wrong hostname
missing intermediate
wrong certificate attached
SNI mismatch
untrusted CA
backend TLS problem
client clock problem
```

Never solve by permanently disabling validation.

---

# 132. Evidence-driven Troubleshooting

Một nguyên tắc nên trở thành thói quen:

```text
Observation
≠
Interpretation
≠
Conclusion
```

Ví dụ:

```text
Observation:
curl timed out.
```

Interpretation:

```text
No application response received within timeout.
```

Không nên nhảy ngay tới:

```text
Conclusion:
Firewall is blocking.
```

Firewall là một hypothesis trong nhiều possibilities.

---

# 133. Hypothesis-driven Troubleshooting

Thay vì chạy hàng chục command ngẫu nhiên:

```text
Hypothesis:
backend app is listening locally but unreachable through BIG-IP.
```

Tests:

```text
Local curl
Backend port state
Pool member state
Health monitor result
BIG-IP-to-backend path
```

Kết quả nhanh hơn và evidence rõ hơn.

---

# 134. Least-risk Troubleshooting Order

Một general order tốt:

```text
Documentation/ticket
        ↓
Monitoring
        ↓
Read-only commands
        ↓
Logs
        ↓
Targeted tests
        ↓
Safe reversible action
        ↓
Configuration change
        ↓
Disruptive maintenance
```

Càng xuống thấp, risk càng tăng.

---

# 135. Dependency Awareness

Trước change phải hỏi:

```text
What depends on this object?
```

Ví dụ restart one VM:

```text
Is it standalone?
Is it in pool?
Does load balancer drain traffic?
Does application have clustering?
Is maintenance approved?
```

Không đánh giá action chỉ theo server đó.

---

# 136. Blast Radius

Blast radius là phạm vi ảnh hưởng tiềm năng.

Ví dụ:

```text
Restart one test VM
```

blast radius nhỏ hơn:

```text
change core firewall policy
```

hoặc:

```text
restart Nutanix cluster service
```

Decision threshold phải tăng tương ứng với blast radius.

---

# 137. Prechecks

Trước change, prechecks nhằm xác nhận:

```text
environment currently known
dependencies healthy
rollback possible
backup/config saved
approval valid
maintenance window active
monitoring available
```

Không có precheck, bạn không biết post-change failure là mới hay đã tồn tại trước.

---

# 138. Postchecks

Postchecks phải mirror prechecks.

Ví dụ trước:

```text
service active
port 443
health endpoint 200
pool member healthy
```

sau change cũng kiểm tra cùng set.

Nhờ vậy before/after có thể so sánh.

---

# 139. Peer Review

Additive Day 14 yêu cầu peer kiểm tra source-to-report traceability và reproduce một transformation. Infrastructure_Operations_14_Da…

Đây là test rất mạnh.

Nếu colleague không thể từ:

```text
Raw file
+
Script version
+
Documented parameters
```

tạo lại report tương tự, process chưa reproducible.

---

# 140. Reproducibility

Reproducibility nghĩa:

```text
Same approved input
+
Same logic/version
+
Same rules
≈
Same business result
```

Không nhất thiết metadata như:

```text
run timestamp
run ID
```

giống nhau.

Nhưng classification/decision logic không được thay đổi ngẫu nhiên.

---

# 141. Silent Data Loss

Day 14 final acceptance đặc biệt cấm silent data loss. Infrastructure_Operations_14_Da…

Ví dụ:

Input:

```text
1000 rows
```

Output:

```text
997 rows
```

nhưng script báo:

```text
Success
```

đây là serious failure nếu ba row biến mất không có explanation.

Correct behavior:

```text
Matched: 950
Exceptions: 30
Rejected: 20
Total: 1000
```

và reconcile được.

---

# 142. Unexplained Hash/Count Discrepancy

Nếu source:

```text
100 records
```

derived report:

```text
96
```

điều đó có thể đúng nếu:

```text
4 intentionally filtered
```

nhưng phải trace được.

Nếu không:

```text
unexplained discrepancy
```

là sign-off blocker theo syllabus. Infrastructure_Operations_14_Da…

---

# 143. Risk Register

Final Day 14 theory cần hiểu risk register không chỉ dành cho security.

Có thể ghi:

```text
Risk ID
Description
Affected service
Likelihood
Impact
Existing control
Residual risk
Owner
Action
Due date
Status
```

Examples:

```text
Backup restore never independently tested
Unknown owner for privileged account
Certificate expires soon
Logging gap
Capacity risk
Unsupported firmware
```

---

# 144. Residual Gap Register

Gap khác incident.

Gap có thể là:

```text
No actual BIG-IP lab access
AccessCheck simulation only
Unable to verify hardware replacement workflow
ServiceNow audit field unavailable
```

Mục tiêu là minh bạch:

```text
What was not proven?
Why?
Who owns follow-up?
```

---

# 145. Acceptance Package

Final acceptance package theo Day 14 gồm các artifacts lớn như:

```text
Architecture
Configuration inventory
Runbooks
Operational checklist
Test results
Audit evidence
Risk register
Signed skill matrix
Evidence manifest
Residual-gap register
Reviewer decisions
Test summary
```

Đây là outcome cuối của chương trình. Infrastructure_Operations_14_Da…

---

# 146. Điều kiện không được xem là “complete”

Theo completion rubric, các gap như sau phải vẫn open nếu chưa chứng minh:

```text
privileged-access revocation proof missing
backup restore not verified
critical automation tests missing
integrity checks missing
```

Completion của course không tự động biến unperformed hands-on thành completed competence. Infrastructure_Operations_14_Da…

---

# 147. Mental Model cuối cùng của Module Lý thuyết

Bạn nên kết thúc Module Lý thuyết Day 14 với mental model này:

```text
                    BUSINESS SERVICE
                          │
                          ▼
                 Understand Impact
                          │
                          ▼
                  ITSM Authorization
                          │
                          ▼
                  Expected Baseline
                          │
                          ▼
                 Observe Current State
                          │
                          ▼
 ┌──────────────────────────────────────────────────────────┐
 │                TECHNICAL DEPENDENCIES                    │
 │                                                          │
 │ Linux / Windows                                          │
 │ AWS                                                      │
 │ HPE / iLO                                                │
 │ Nutanix AOS / AHV / Prism                                │
 │ HYCU                                                     │
 │ Catalyst / Nexus                                         │
 │ FortiGate                                                │
 │ BIG-IP                                                   │
 │ Storage / Backup                                         │
 └──────────────────────────────────────────────────────────┘
                          │
                          ▼
                Monitoring & Logging
                          │
                          ▼
                  Fault Localization
                          │
                          ▼
                  Security / IAM
                          │
                          ▼
              Authorized Corrective Work
                          │
             ┌────────────┴────────────┐
             ▼                         ▼
          Success                    Failure
             │                         │
             ▼                         ▼
      Verify Recovery           Rollback / Escalate
             │                         │
             └────────────┬────────────┘
                          ▼
                    Collect Evidence
                          │
                          ▼
                   Update ServiceNow
                          │
                          ▼
                Update Asset / Risks
                          │
                          ▼
               Peer Review / Reproduce
                          │
                          ▼
                       Handover
                          │
                          ▼
                 Final Acceptance
```

---

# 148. Những câu bạn phải trả lời được sau Module Lý thuyết

Sau phần lý thuyết này, bạn phải có khả năng giải thích được những vấn đề sau mà không chỉ đọc thuộc command:

**Một:** Vì sao `service running` chưa đủ chứng minh application healthy?

**Hai:** Vì sao một website outage có thể xuất phát từ BIG-IP, FortiGate, routing, backend OS, certificate hoặc hypervisor?

**Ba:** Cách phân biệt L2, L3, firewall, TCP, TLS, HTTP và application failure.

**Bốn:** Vì sao backup success không chứng minh restore readiness?

**Năm:** Vì sao IAM entitlement không chứng minh actual user activity?

**Sáu:** Vì sao blank `last used` không tự động chứng minh dormant account?

**Bảy:** Vì sao `No log found` không chứng minh `No event occurred`?

**Tám:** Vì sao matching SHA-256 không chứng minh query/source completeness?

**Chín:** Vì sao screenshot không phải lúc nào cũng đủ làm audit evidence?

**Mười:** Khi nào L1/L2 phải stop và escalate?

**Mười một:** Vì sao rollback phải được thiết kế trước change?

**Mười hai:** Vì sao successful command execution không đồng nghĩa successful maintenance?

**Mười ba:** Vì sao current state phải được so với approved baseline?

**Mười bốn:** Vì sao product/version gap phải xuất hiện trong final handover?

**Mười lăm:** Vì sao `S`, `D`, `T` không được khai báo như actual hands-on `H`?

---

# 149. Cheat Sheet tư duy — Module Lý thuyết Day 14

Có thể cô đọng toàn bộ lý thuyết Day 14 thành mười nguyên tắc:

```text
1. Verify identity, environment and authorization first.

2. Understand the dependency chain before troubleshooting.

3. Start read-only and low-risk.

4. Separate symptom, observation, hypothesis and root cause.

5. Troubleshoot layer-by-layer instead of random changes.

6. A change is not successful until post-verification passes.

7. Backup is not recovery; restore must be proven.

8. Evidence must be traceable, complete enough, timestamped,
   counted, hashed where applicable and linked to a ticket.

9. Unknown remains unknown; gaps must be documented.

10. Safe escalation and handover are part of successful operations,
    not signs of failure.
```

---

## Kết luận Module Lý thuyết Day 14

Điểm cốt lõi của Day 14 không phải ghi nhớ nhiều product hơn. Đó là chuyển toàn bộ kiến thức Days 1–13 thành **operational judgment**.

Một Infrastructure Operations engineer đạt mục tiêu Day 14 phải có khả năng nhìn một incident và suy nghĩ:

```text
What is impacted?
What should the architecture look like?
Which dependency could explain this symptom?
What evidence do I already have?
What read-only check gives me the most information?
What layer owns the fault?
What am I authorized to change?
What is the safest corrective action?
How do I prove recovery?
What is my rollback?
What evidence must I preserve?
What remains unknown?
Should I close, escalate, or hand over?
```

Đó cũng là lý do final acceptance của syllabus yêu cầu không chỉ technical result mà còn **skill matrix, evidence manifest, reviewer decisions, test results và residual gaps**, đồng thời không chấp nhận silent data loss, exposed secrets, unapproved changes hoặc unexplained hash/count discrepancies. Infrastructure_Operations_14_Da…

**Module Lý thuyết Day 14 đến đây đã cover toàn bộ Concept/Lecture trong syllabus:** integrated infrastructure operations, Linux/RHEL, Windows Server, AWS, HPE/iLO, Nutanix AOS/AHV/Prism Central, HYCU, Cisco Nexus/Catalyst, FortiGate, BIG-IP/SSL/TLS, centralized logging, EventLog Analyzer, Firewall Analyzer, ServiceNow, PalletControl, privileged access, 90-day IAM review, vulnerability handling, VBA/Python/Batch/PowerShell, Outlook/MAPI, configuration baseline, monitoring/alerting, backup/recovery, security, audit evidence, escalation, handover và final technical readiness. Infrastructure_Operations_14_Da…
