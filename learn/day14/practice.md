# Day 14 — Module Thực hành

## Integrated Enterprise Operations Capstone Lab

Module Thực hành Day 14 là **capstone lab** của toàn bộ chương trình. Đây không phải một lab riêng lẻ kiểu “restart một service” hay “configure một VLAN”, mà là bài thực hành tích hợp trong đó bạn phải vận hành một environment có nhiều dependency, nhận ticket, xác định phạm vi, thực hiện health check, điều tra nhiều loại fault, thực hiện corrective action/rollback được phê duyệt, kiểm tra backup/recovery, chạy automation, reconcile IAM/vulnerability, cập nhật ITSM/asset records, thu thập audit evidence, peer review và cuối cùng thực hiện shift handover.

Theo syllabus, Day 14 bắt buộc phải dùng approved build sheet, runbooks, environment inventory và ServiceNow tickets; phải verify Linux/Windows, AWS, Nutanix, HPE, networking, FortiGate, BIG-IP và backup protection; đồng thời phải điều tra các fault thuộc service, permission, routing, firewall, SSL certificate, VM availability, storage capacity, backup, privileged access và logging. Sau đó phải khôi phục service trong authorized boundary, thực hiện backup/recovery/rollback khi phù hợp, correlate logs, chạy VBA/Python/Batch/PowerShell, làm IAM/access review rút gọn, vulnerability reconciliation, update ServiceNow và asset inventory, peer/mentor review, handover và final acceptance. Infrastructure_Operations_14_Da…

Additive requirements còn yêu cầu **mỗi learner trực tiếp thực hiện** ít nhất một server-log extraction, một approved corrective operation hoặc rollback, một isolated recovery check, một IAM exception decision và một automation regression run. Các scenario còn lại có thể tổ chức dưới dạng rotating stations/fault packets, nhưng vẫn phải được review trong Day 14. Infrastructure_Operations_14_Da…

---

# 1. Mục tiêu tổng thể của Capstone Lab

Sau khi hoàn thành module này, bạn phải có khả năng thực hiện end-to-end operational workflow:

```text
Receive Ticket
    ↓
Confirm Scope / Priority / Authorization
    ↓
Identify Environment and Assets
    ↓
Establish Baseline
    ↓
Perform Health Checks
    ↓
Understand Symptom
    ↓
Localize Fault Domain
    ↓
Collect Evidence
    ↓
Form and Test Hypothesis
    ↓
Determine Root Cause
    ↓
Check Change Authorization
    ↓
Perform Controlled Corrective Action
    ↓
Verify Service Recovery
    ↓
Rollback if Required
    ↓
Correlate Logs
    ↓
Update ServiceNow
    ↓
Update Asset / IAM / Vulnerability Records
    ↓
Run Automation Regression
    ↓
Peer Review
    ↓
Shift Handover
    ↓
Final Acceptance
```

Bạn không được đánh giá chỉ bằng việc “service chạy lại”.

Bạn được đánh giá trên cả:

```text
Correctness
Safety
Least privilege
Verification quality
Troubleshooting reasoning
Evidence quality
Rollback readiness
Escalation judgment
Audit traceability
Reproducibility
Handover quality
```

---

# 2. Theo syllabus — Environment phải chuẩn bị

Day 14 giả định instructor đã provision trước environment; setup environment không được tính là “ẩn thêm training day”. Syllabus yêu cầu có RHEL và Windows VMs, isolated AWS sandbox, Excel/VBA, Python, PowerShell, Batch, compatible classic Outlook/test mailbox, sanitized data và các product labs phù hợp. Nếu enterprise product không khả dụng thì phải dùng version-labelled exports, screenshots, configs và fault packets, đồng thời ghi nhận hands-on gap. Infrastructure_Operations_14_Da…

Một topology tham khảo cho capstone:

```text
                    ┌──────────────────────┐
                    │      Client/Test     │
                    │      Workstation     │
                    └──────────┬───────────┘
                               │
                               ▼
                        Enterprise Network
                               │
             ┌─────────────────┴──────────────────┐
             │                                    │
             ▼                                    ▼
      Cisco Catalyst/Nexus                  FortiGate
          L2 / L3                               │
                                                ▼
                                         BIG-IP LTM
                                                │
                              ┌─────────────────┴──────────────┐
                              │                                │
                              ▼                                ▼
                       Linux/RHEL VM                    Windows Server VM
                              │                                │
                              └──────────────┬─────────────────┘
                                             │
                                             ▼
                                   Nutanix AHV/AOS
                                             │
                                      Prism / CVM
                                             │
                                             ▼
                                       HPE Hardware
                                          / iLO

                    AWS Sandbox
       ┌───────────┬───────────┬────────────┐
       │           │           │            │
      IAM         VPC         EC2          EBS
                                            │
                                           S3
                                            │
                              CloudWatch / CloudTrail

                     Backup / DR
                 Nutanix ─────── HYCU
                 AWS ─────────── S3

                  Centralized Logging
       Windows / RHEL / Cisco / FortiGate / AWS
                        ↓
              EventLog Analyzer
              Firewall Analyzer

                    Operations Plane
        ServiceNow / PalletControl / IAM Review
            PS/Gateway / AccessCheck fixtures

                    Automation Plane
          Excel/VBA / Python / Batch /
          PowerShell / Outlook-MAPI
```

Đây là **logical topology**, không yêu cầu mọi organization phải có kiến trúc chính xác như vậy.

---

# 3. Prerequisites trước khi bắt đầu

Trước lab phải có ít nhất:

```text
Approved lab account
Approved privileged account nếu cần
ServiceNow-style tickets
Environment inventory
Build sheet
Network diagram
Asset inventory
Configuration baseline
Runbooks
Known-good test criteria
Evidence repository
Synthetic IAM dataset
Vulnerability dataset
Automation source files
Golden expected results
Backup recovery points
Fault packets
```

Không dùng:

```text
real password
real private key
production token
MFA secret
production customer data
```

Ví dụ credential trong script luôn dùng placeholder hoặc secret mechanism của lab.

---

# 4. Safety Rules

## 4.1 Environment classification

Trước command đầu tiên phải xác nhận:

```text
LAB
DEV
TEST
STAGING
PRODUCTION
```

Nếu bạn không biết mình đang ở environment nào:

**không thực hiện configuration-changing operation.**

---

## 4.2 Read-only và state-changing phải phân biệt

Ví dụ Linux:

Read-only:

```bash
hostnamectl
uptime
systemctl status nginx
ss -lntp
ip route
df -h
journalctl -u nginx
```

Changing:

```bash
sudo systemctl restart nginx
```

AWS:

Read-only:

```bash
aws sts get-caller-identity
aws ec2 describe-instances
```

Changing:

```bash
aws ec2 stop-instances ...
```

Network:

Read-only:

```text
show interfaces
show vlan
show ip route
```

Changing configuration phải qua approved change.

---

# 5. Evidence Repository Structure

Ngay đầu lab nên tạo evidence repository.

Ví dụ:

```text
DAY14/
├── 00_Control/
├── 01_Tickets/
├── 02_Baseline/
├── 03_Server/
│   ├── Linux/
│   └── Windows/
├── 04_AWS/
├── 05_HPE_Nutanix/
├── 06_Network/
├── 07_FortiGate/
├── 08_BIGIP/
├── 09_Backup/
├── 10_Logging/
├── 11_IAM/
├── 12_Vulnerabilities/
├── 13_Automation/
├── 14_ServiceNow_Assets/
├── 15_PeerReview/
├── 16_Handover/
└── 99_FinalPackage/
```

Không bắt buộc tên folder chính xác như vậy; quan trọng là reproducibility.

---

# 6. Evidence Naming Convention

Ví dụ:

```text
D14-EV-001_RHEL01_systemctl_20261022T020500Z.txt
D14-EV-002_WIN01_SystemLog_20261022T021000Z.evtx
D14-EV-003_FGT01_route_20261022T021500Z.txt
D14-EV-004_BIGIP_pool-status_20261022T022000Z.json
```

Tên nên giúp reviewer nhận ra:

```text
Evidence ID
Asset
Content
Timestamp
```

---

# 7. Evidence Manifest

Syllabus yêu cầu mỗi evidence item có metadata như:

```text
Evidence ID
Source system/product/version
Environment
Asset/account identifier
Ticket/stable record key
Collector
Collection timestamp/timezone
Requested time interval
Exact command/filter/query
Export format
Encoding
Filename/path
File size
Record count
SHA-256
Access classification
Retention classification
Parent evidence
Transformation/script version
Limitations/errors
```

Raw exports phải được giữ nguyên; working copy phải tách riêng. Hash chỉ chứng minh file không thay đổi so với baseline hash, không chứng minh source/query là đúng hoặc complete. Infrastructure_Operations_14_Da…

---

# 8. PHASE 1 — Ticket Intake & Operational Control

Giả sử bạn nhận:

```text
INC001401
```

Symptom:

```text
Users cannot access APP-PROD-like lab service over HTTPS.
Intermittent backup failure was also detected.
A recent approved network change exists.
```

Cần đọc ticket trước khi troubleshooting.

Xác định:

```text
Service
Affected users
Environment
Start time
Impact
Urgency
Priority
CI/assets
Recent changes
Assignment group
Current owner
Maintenance window
Related change
```

Không vội login server.

---

# 9. Ticket Quality Check

Ticket thiếu các trường quan trọng thì ghi lại.

Ví dụ:

```text
Missing:
- exact affected URL
- first observed timestamp
- impacted user scope
```

Bạn có thể tiếp tục read-only investigation nếu phù hợp, nhưng không invent missing facts.

---

# 10. Recent Change Review

Tìm:

```text
Firewall change
Load balancer change
Network change
Server patch
Certificate replacement
IAM modification
VM migration
Storage maintenance
Backup policy change
```

Recent change rất đáng điều tra nhưng **không được tự động coi là root cause**.

---

# 11. Establish Investigation Window

Ví dụ:

```text
Incident first reported:
2026-10-22 09:05 +07:00

Initial investigation window:
2026-10-22 08:45–09:30 +07:00
```

Khi query system dùng UTC:

```text
2026-10-22 01:45–02:30 UTC
```

Phải ghi rõ conversion.

---

# 12. PHASE 2 — Establish Baseline

Trước khi sửa bất kỳ thứ gì, chụp current state.

Baseline gồm:

```text
Servers
AWS
Nutanix
HPE
Network
Firewall
Load balancer
Backup
Monitoring
Logging
IAM
```

Mục tiêu:

> Nếu sau corrective action environment xấu hơn, ta biết trạng thái trước đó là gì.

---

# 13. Linux/RHEL Health Check

## Objective

Xác nhận:

```text
Host identity
OS
Uptime
Service state
Resource status
Network
Storage
Logs
```

---

## 13.1 Identity

```bash
hostnamectl
```

Hoặc:

```bash
hostname
```

Evidence cần lưu:

```text
hostname
OS/release
collection time
```

Không troubleshoot nhầm host.

---

## 13.2 Uptime

```bash
uptime
```

Mục tiêu:

- phát hiện recent reboot;
- kiểm tra load indication sơ bộ.

---

## 13.3 Failed systemd units

```bash
systemctl --failed
```

Không tự restart tất cả failed services.

Xác định service nào relevant.

---

# 14. Critical Service Verification

Ví dụ application:

```bash
systemctl status nginx
```

Nếu:

```text
active (running)
```

chưa dừng tại đó.

Kiểm tra process/listener:

```bash
ss -lntp
```

Local application test:

```bash
curl -v http://127.0.0.1/
```

Nếu HTTPS local endpoint:

```bash
curl -v https://localhost/
```

Không dùng insecure TLS mode để biến test thành pass.

---

# 15. Linux Resource Verification

```bash
free -h
df -h
df -i
```

Có thể bổ sung:

```bash
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
```

Mục tiêu:

```text
CPU pressure?
Memory pressure?
Filesystem full?
Inode exhaustion?
Unexpected process?
```

---

# 16. Linux Network Verification

```bash
ip addr
ip route
```

DNS:

```bash
getent hosts app.example.internal
```

Port/listener:

```bash
ss -lntp
```

Path test nếu permitted:

```bash
ping <gateway>
```

hoặc application-aware:

```bash
curl -v https://target/
```

Không coi `ping` failure là proof network failure vì ICMP có thể bị chặn.

---

# 17. Linux SELinux / Local Firewall

Read-only SELinux check:

```bash
getenforce
```

Nếu relevant:

```bash
ls -Z /path
```

Local firewall:

```bash
firewall-cmd --list-all
```

nếu authorized và tool tồn tại.

Không thực hiện:

```bash
setenforce 0
```

hay disable firewall như “quick fix”.

---

# 18. Linux Log Extraction — Mandatory Individual Task

Mỗi learner phải trực tiếp thực hiện server-log extraction.

Ví dụ service log:

```bash
journalctl -u nginx \
  --since "2026-10-22 01:45:00 UTC" \
  --until "2026-10-22 02:30:00 UTC"
```

Hoặc bounded output:

```bash
journalctl \
  --since "2026-10-22 01:45:00 UTC" \
  --until "2026-10-22 02:30:00 UTC"
```

Phải lưu:

```text
hostname
command
timezone
window
output
count nếu applicable
errors
```

Nếu journal rotation khiến interval thiếu:

**ghi evidence gap**.

---

# 19. Windows Server Health Check

## Identity

PowerShell:

```powershell
$env:COMPUTERNAME
Get-ComputerInfo
```

Không nhất thiết export toàn bộ `Get-ComputerInfo`; có thể select relevant fields để evidence dễ review.

---

# 20. Windows Service Verification

```powershell
Get-Service
```

Specific:

```powershell
Get-Service -Name W32Time
```

Process:

```powershell
Get-Process
```

TCP listeners:

```powershell
Get-NetTCPConnection -State Listen
```

---

# 21. Windows Storage

```powershell
Get-Volume
Get-Disk
```

Kiểm tra:

```text
drive capacity
free space
filesystem
operational status
```

Không format/init disk trong Day 14 trừ khi có isolated destructive storage lab riêng được approved.

---

# 22. Windows Networking

```powershell
Get-NetIPConfiguration
Get-NetRoute
```

DNS test:

```powershell
Resolve-DnsName app.example.internal
```

TCP connectivity:

```powershell
Test-NetConnection app.example.internal -Port 443
```

Đây hữu ích hơn `ping` trong HTTPS troubleshooting.

---

# 23. Windows Firewall Inspection

Read-only:

```powershell
Get-NetFirewallProfile
```

Relevant rules:

```powershell
Get-NetFirewallRule |
    Where-Object Enabled -eq 'True'
```

Trong large environment cần filter chặt hơn; không dump vô hạn dữ liệu.

---

# 24. Windows Event Log Extraction

Ví dụ System events theo time:

```powershell
$Start = Get-Date "2026-10-22 08:45"
$End   = Get-Date "2026-10-22 09:30"

Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    StartTime = $Start
    EndTime   = $End
}
```

Có thể select:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    StartTime = $Start
    EndTime   = $End
} | Select-Object TimeCreated, Id, ProviderName, LevelDisplayName, Message
```

Preserve native EVTX nếu acceptance yêu cầu native evidence.

---

# 25. AWS Health Check

## 25.1 Identity first

```bash
aws sts get-caller-identity
```

Phải verify:

```text
Account
Arn
UserId
```

Không submit secrets.

---

# 26. AWS Region Context

Ví dụ:

```bash
aws configure get region
```

Nếu sử dụng profile:

```bash
aws configure get region --profile <approved-profile>
```

Phải biết resource có ở Region nào.

---

# 27. EC2 Inspection

```bash
aws ec2 describe-instances \
  --instance-ids <INSTANCE_ID>
```

Nên lọc output thay vì lưu rất nhiều metadata.

Quan sát:

```text
instance state
subnet
VPC
security groups
IAM profile
private/public addresses
```

Không stop/reboot instance chỉ vì application fail.

---

# 28. VPC Path Verification

Kiểm tra:

```text
Subnet
Route table
Security Group
NACL
Gateway dependency
```

Sử dụng Console hoặc approved AWS CLI queries.

Bạn phải tạo expected-path worksheet:

```text
Source:
Client subnet

Destination:
EC2 private IP

Protocol:
TCP

Port:
443

Expected controls:
SG allow
NACL allow
Route exists
```

---

# 29. EBS Check

Kiểm tra:

```text
Volume attached?
Correct instance?
Expected device?
OS sees volume?
Filesystem mounted?
Capacity okay?
```

Cloud-level attachment không đồng nghĩa guest OS mount đúng.

---

# 30. S3 Backup Check

Inspect:

```text
bucket
Block Public Access
versioning
encryption
recovery object/version
```

Không thay retention/Object Lock trong capstone nếu không có explicit isolated lab approval.

---

# 31. CloudWatch Check

Inspect:

```text
EC2 metrics
alarms
application logs nếu configured
alarm history
```

Hỏi:

```text
Did alert trigger?
Was data available?
Was alarm configured correctly?
```

---

# 32. CloudTrail Check

Use relevant management activity to answer questions như:

```text
Was resource modified?
Which principal performed API action?
When?
Which resource?
```

Không coi Event History là universal evidence cho mọi object access.

---

# 33. HPE ProLiant / iLO Health Station

Phần này nên chủ yếu read-only.

Verify:

```text
Server identity
Serial/asset reference
Power state
Overall health
CPU
Memory
Storage controller
Disk state
NIC health
Temperature
Fan/power alerts
Firmware versions
IML entries
```

Evidence:

```text
timestamped health screenshot/export
IML entry
firmware inventory
affected component
```

---

# 34. HPE Stop Condition

Nếu phát hiện:

```text
Failed disk + degraded RAID
Critical memory error
Power supply redundancy lost
Thermal critical warning
```

thì **không tự ý firmware update/reboot**.

Escalate theo hardware runbook.

---

# 35. Nutanix Health Station

Qua Prism Element/Prism Central phù hợp:

Verify:

```text
Cluster health
Node health
CVM state
AHV host state
VM state
Capacity
Alerts
Tasks
Events
Version
```

Không assume Prism Central và Prism Element có cùng scope/menu trên mọi release.

---

# 36. VM Availability Check

Đối với VM:

```text
VM exists?
Powered on?
Correct host?
Correct network?
Correct NIC?
Correct disk?
Console accessible?
Guest reachable?
```

Sau đó mới xuống guest OS troubleshooting.

---

# 37. Nutanix Capacity Check

Quan sát:

```text
storage utilization
capacity alerts
host utilization
cluster resiliency
headroom
```

Một cluster “còn free capacity” vẫn có thể không đủ an toàn cho maintenance.

---

# 38. Nutanix Maintenance Boundary

Không thực hiện production-like:

```text
CVM restart
AOS upgrade
AHV upgrade
LCM firmware operation
host removal
```

trừ khi instructor-provided isolated lab/runbook cho phép.

Trong Day 14 thường đủ:

```text
read-only verification
prepared maintenance simulation
fault packet analysis
```

---

# 39. HYCU Protection Check

Inspect:

```text
Protected source
Policy
Last job
Job status
Recovery point
Target
Capacity
Retention
```

Nếu job `Success`, vẫn phải tiếp tục recovery validation.

---

# 40. Network Baseline — Catalyst/Nexus

Read-only commands phải validate theo actual release.

Common examples:

```text
show version
show interfaces
show interfaces status
show vlan
show mac address-table
show ip arp
show ip route
show spanning-tree
```

Trunk-related example:

```text
show interfaces trunk
```

Port-channel:

```text
show etherchannel summary
```

hoặc NX-OS equivalent theo approved runbook.

Không giả định IOS XE và NX-OS giống hệt nhau.

---

# 41. Network Path Worksheet

Tạo bảng:

| Layer     | Expected              | Actual | Evidence |
| --------- | --------------------- | ------ | -------- |
| Interface | up/up                 | ?      | command  |
| VLAN      | VLAN 120              | ?      | command  |
| Trunk     | VLAN 120 allowed      | ?      | command  |
| ARP       | gateway resolved      | ?      | command  |
| Route     | destination reachable | ?      | command  |
| TCP       | 443 reachable         | ?      | test     |

Điều này ép troubleshooting có trình tự.

---

# 42. FortiGate Baseline

Vì FortiOS syntax có thể khác theo version, command cụ thể phải được đối chiếu approved runbook.

Các read-only categories cần kiểm tra:

```text
system/version
interfaces
routes
policies
address/service objects
sessions
traffic logs
```

Common examples trong nhiều FortiOS releases có thể gồm:

```text
get system status
get router info routing-table all
```

Session inspection phải filter chặt nếu dùng diagnostic command.

Không chạy unrestricted debug trên production-like firewall.

---

# 43. FortiGate Debug Safety

Nếu instructor cho phép packet/debug:

Phải xác định trước:

```text
source
destination
protocol
port
duration
stop command
```

Rule:

```text
Start bounded debug
→ capture required evidence
→ stop debug immediately
```

Không để diagnostic session chạy vô hạn.

---

# 44. BIG-IP Baseline

Validate:

```text
Virtual server
Destination/port
Pool
Pool members
Health monitor
Profiles
SSL profiles
Certificate
```

Common read-only `tmsh` examples, tùy version/runbook:

```text
tmsh show ltm virtual
tmsh show ltm pool
```

Nếu sử dụng iControl REST, ưu tiên GET/read-only.

---

# 45. BIG-IP API Safety

Read-only request concept:

```bash
curl \
  --cacert <TRUSTED_CA_FILE> \
  -H "Authorization: Bearer <TOKEN_PLACEHOLDER>" \
  https://<BIGIP_HOST>/mgmt/...
```

Không lưu real Authorization header vào transcript.

Không dùng:

```bash
-k
```

để biến certificate validation thành “pass”.

---

# 46. Monitoring/Logging Baseline

Check:

```text
EventLog Analyzer collector health
known sources
last event
ingestion delay
retention state
Firewall Analyzer source health
FortiGate log ingestion
known-event visibility
```

Một dashboard không có alerts chưa chứng minh pipeline healthy.

---

# 47. PHASE 3 — Failure Scenario 1: Service Failure

## Injected fault

Ví dụ lab:

```text
Linux web service stopped
```

Symptom:

```text
BIG-IP marks backend member down.
User HTTPS request fails/intermittent.
```

---

## Investigation

Client test:

```bash
curl -v https://app.lab.example/
```

Load balancer:

```text
Virtual server state
Pool state
Member state
Monitor result
```

Server:

```bash
systemctl status nginx
ss -lntp
journalctl -u nginx
```

---

## Expected reasoning

Nếu:

```text
BIG-IP monitor = failed
Server port 443 = no listener
Service = stopped
```

thì fault domain đã localized tới backend service.

---

# 48. Corrective Action

Chỉ sau approval:

```bash
sudo systemctl start nginx
```

hoặc approved equivalent.

Không dùng restart nếu start đủ.

---

# 49. Verification

```bash
systemctl is-active nginx
ss -lntp
curl https://localhost/
```

Sau đó verify:

```text
BIG-IP member healthy
external URL healthy
monitor alert clears
```

---

# 50. Evidence

Collect:

```text
pre-state
service log
approved ticket/change
corrective command
post-state
BIG-IP recovery state
external transaction result
```

---

# 51. Rollback

Nếu start gây unexpected impact:

```text
stop service
restore known-good config nếu config change liên quan
```

theo runbook.

---

# 52. Escalation

Escalate nếu:

```text
service repeatedly crashes
binary/config corruption
dependency unavailable
start causes data integrity concern
unknown application error
```

---

# 53. Failure Scenario 2 — Permission Failure

## Injected fault

Application service account mất permission vào data directory.

Symptom:

```text
Application starts but cannot read/write data.
```

---

# 54. Linux Permission Investigation

```bash
ls -ld /app/data
ls -l /app/data
id <service-user>
```

Nếu SELinux relevant:

```bash
ls -Zd /app/data
getenforce
```

Logs:

```bash
journalctl -u <service>
```

---

# 55. Windows Permission Variant

Inspect:

```powershell
Get-Acl C:\App\Data
```

Nếu network share:

phải xem cả:

```text
NTFS permission
Share permission
Group membership
```

---

# 56. Root Cause Discipline

Không sửa:

```text
chmod -R 777
```

hoặc:

```text
Everyone: Full Control
```

Chỉ cấp permission tối thiểu theo approved baseline.

---

# 57. Corrective Action

Ví dụ conceptual:

```text
Restore approved owner/group/ACL
```

Phải có before/after evidence.

---

# 58. Verification

Dùng chính application/service account hoặc approved functional test.

Không chỉ kiểm tra `ls -l`.

---

# 59. Failure Scenario 3 — Routing Failure

Injected fault:

```text
Incorrect static route fixture
```

Symptom:

```text
Server cannot reach backend service.
```

---

# 60. Investigation

Source:

```bash
ip route
```

Cisco:

```text
show ip route
```

AWS:

inspect relevant route table.

FortiGate:

inspect routing table.

Cần xác định:

```text
source
destination
next hop
forward path
return path
```

---

# 61. Anti-pattern

Thấy traffic fail rồi add route “cho thử”.

Không được.

Trước configuration change phải biết:

```text
what route should exist
why
where
which prefix
which next hop
```

---

# 62. Verification

Sau approved correction:

```text
routing table matches baseline
application path passes
return path works
no unexpected route introduced
```

---

# 63. Failure Scenario 4 — Firewall Failure

Injected fault:

```text
Expected HTTPS flow denied because of incorrect lab policy/object.
```

---

# 64. Investigation

Define tuple:

```text
Source IP
Destination IP
Protocol
Destination port
Time
```

Sau đó:

```text
Does traffic arrive?
Which route?
Which firewall policy matches?
Is session created?
What does traffic log say?
```

---

# 65. Common Wrong Conclusion

```text
curl timeout
→ firewall problem
```

Sai.

Có thể do:

```text
server down
route issue
listener absent
BIG-IP issue
```

Phải có firewall evidence.

---

# 66. Approved Correction

Ví dụ instructor đã seed sai service object.

Change:

```text
restore approved service object/policy
```

Không tạo broad:

```text
any → any → allow
```

---

# 67. Rollback

Trước change phải có:

```text
policy ID/object ID
old value
new value
configuration backup/revision
rollback steps
```

---

# 68. Failure Scenario 5 — SSL/TLS Certificate Failure

Injected fixture:

```text
Expired certificate
Wrong hostname
Missing intermediate
Untrusted CA
```

---

# 69. Investigation Layers

Phân tách:

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

CLI inspection:

```bash
curl -v https://app.lab.example/
```

Hoặc:

```bash
openssl s_client \
  -connect app.lab.example:443 \
  -servername app.lab.example
```

---

# 70. Check Certificate

Quan tâm:

```text
Subject/SAN
Issuer
Validity
Chain
Hostname
SNI
```

Không ghi private key vào evidence.

---

# 71. Root Cause Examples

```text
Certificate expired
Certificate valid but wrong hostname
Intermediate missing
Client does not trust CA
Wrong SSL profile attached
Backend TLS monitor failure
```

---

# 72. Corrective Boundary

Certificate replacement là state-changing security operation.

Cần:

```text
approved certificate
private-key handling controls
change ticket
backup/current profile reference
rollback
maintenance plan
```

Trong Day 14 có thể dùng fixture nếu actual replacement không authorized.

---

# 73. Failure Scenario 6 — VM Availability

Injected fault:

```text
VM powered off
or
incorrect VM network assignment
or
host-side availability fixture
```

---

# 74. Investigation

Prism:

```text
VM state
task history
alerts
host
NIC/network
disk
```

Guest:

```text
console
OS boot
network
services
```

Hardware if needed:

```text
iLO alerts
```

---

# 75. Corrective Action

Nếu approved:

```text
Power on VM
```

hoặc restore expected virtual NIC/network assignment.

Không migrate/reconfigure cluster resource ngẫu nhiên.

---

# 76. Verification

```text
VM running
Guest reachable
Application service healthy
Monitoring recovered
BIG-IP member healthy nếu applicable
```

---

# 77. Failure Scenario 7 — Storage Capacity

Injected fault:

```text
Linux filesystem almost/full
Windows volume low
Nutanix capacity warning
HYCU target capacity low
```

---

# 78. Linux Investigation

```bash
df -h
df -i
du -x -h /path 2>/dev/null | sort -h | tail
```

`du` trên large production filesystem có thể tạo I/O load; trong production phải cân nhắc scope/tooling.

---

# 79. Windows Investigation

```powershell
Get-Volume
```

Không xóa files ngay.

Xác định:

```text
logs?
temp?
application data?
backup staging?
unexpected growth?
```

---

# 80. Capacity Correction

Safe options tùy case:

```text
approved log cleanup
retention correction
known temporary file cleanup
capacity extension
application cleanup
```

Mọi deletion phải có owner/approval.

---

# 81. Verification

```text
capacity recovered
service recovered
monitoring alert clears
no required files deleted
```

---

# 82. Failure Scenario 8 — Backup Failure

Injected fault có thể là:

```text
target unavailable
capacity insufficient
permission denied
source unavailable
```

---

# 83. Investigation

HYCU:

```text
Job ID
Source
Policy
Target
Start/end
Error
Recovery point status
```

AWS/S3:

```text
identity
object/bucket access
versioning
encryption dependency
```

---

# 84. Corrective Action

Chỉ correction thuộc approved scope.

Ví dụ:

```text
restore lab target availability
correct approved sandbox permission
free prepared target capacity
```

Sau đó rerun.

---

# 85. Critical Rule

```text
Backup rerun success
≠
Recovery proven
```

Do đó phải chuyển sang isolated recovery.

---

# 86. PHASE 4 — Mandatory Isolated Recovery Check

Mỗi learner phải thực hiện recovery validation.

Có thể chọn:

```text
HYCU file restore
HYCU VM restore
S3 version restore
other approved backup fixture
```

---

# 87. Recovery Preconditions

Trước restore xác nhận:

```text
Recovery point
Source
Target
Restore destination
Network isolation
Naming/IP conflict
Available credentials
Encryption dependencies
Expected content manifest
```

---

# 88. Recovery Start Timestamp

Ghi:

```text
Recovery initiated:
2026-10-22T10:21:15+07:00
```

Sau khi complete:

```text
Recovery functional validation complete:
2026-10-22T10:34:40+07:00
```

Observed recovery duration:

```text
13m25s
```

Đây không tự động bằng contractual RTO, nhưng là observed recovery metric.

---

# 89. Restore to Isolation

Không restore VM vào production-like network nếu sẽ tạo:

```text
duplicate IP
duplicate hostname
duplicate domain identity
service conflict
```

Dùng isolated network hoặc approved alternate destination.

---

# 90. Content Validation

Nếu file-based:

```text
expected files?
record count?
hash?
application readable?
```

Ví dụ Linux:

```bash
sha256sum restored-file
```

Windows:

```powershell
Get-FileHash .\restored-file -Algorithm SHA256
```

---

# 91. Recovery Acceptance

Phải chứng minh:

```text
Correct recovery point
Correct object/data
Content integrity
Application readability
No conflict
Observed recovery time
```

Syllabus đặc biệt yêu cầu isolated restore, verified content/counts và measured recovery, chứ không chỉ job status. Infrastructure_Operations_14_Da…

---

# 92. Failure Scenario 9 — Privileged Access Problem

Injected scenario:

```text
User has expired approval but entitlement still active
```

hoặc:

```text
Approved user cannot obtain required temporary access
```

---

# 93. Investigation

Check:

```text
Identity
Account
Owner
Requested access
Approval
Start/end
Entitlement
Actual access
Expiry
Revocation
Audit record
```

Không yêu cầu user gửi password/token.

---

# 94. Distinguish Cases

Case A:

```text
Approval absent
+
access absent
```

có thể là correct control behavior.

Case B:

```text
Approval expired
+
access still active
```

là exception.

Case C:

```text
Approval valid
+
access denied
```

có thể là provisioning/role/problem.

---

# 95. Mandatory IAM Exception Decision

Mỗi learner phải quyết định một exception.

Possible dispositions:

```text
KEEP
REVOKE
MODIFY
INVESTIGATE
```

Record:

```text
Account
System
Exception
Evidence
Decision
Rationale
Owner
Reviewer
Due date
Ticket
```

---

# 96. Example

```text
Account: ops-temp-017
Entitlement: Firewall-ReadWrite
Approval expiry: 2026-10-20
Observed effective access: Active
Decision: REVOKE
Reason: Authorization period expired
Owner: Network Operations
Ticket: TASK001417
```

Không thực hiện actual revocation ngoài authorized sandbox.

---

# 97. Post-remediation Verification

Nếu sandbox remediation approved:

```text
revocation performed
```

sau đó **retest effective access**.

Không chỉ tin UI nói `Disabled`.

---

# 98. Failure Scenario 10 — Logging Failure

Injected fault:

```text
RHEL source stops appearing in EventLog Analyzer.
```

---

# 99. Investigation Chain

```text
Known event generated at source?
     ↓
Event exists locally?
     ↓
Forwarding/agent healthy?
     ↓
Network path healthy?
     ↓
Collector receives?
     ↓
Parser processes?
     ↓
Search/index contains?
```

---

# 100. Generate/Identify Known Event

Trong authorized lab có thể tạo harmless known event hoặc chọn existing known event.

Record exact:

```text
timestamp
host
event identifier
message
```

Sau đó search centralized platform.

---

# 101. “No results” handling

Nếu analyzer không thấy event:

Không kết luận ngay:

```text
event did not occur
```

Ghi:

```text
source evidence exists
central evidence absent
ingestion gap suspected
```

---

# 102. Logging Gap Escalation

Escalation package:

```text
Source
Last known event
Known missing event
Collector
Time window
Ingestion status
Search filter
Affected audit coverage
```

---

# 103. PHASE 5 — Correlated Incident Timeline

Sau khi xử lý technical faults, dựng timeline.

Ví dụ:

| Time  | Source            | Event                    |
| ----- | ----------------- | ------------------------ |
| 09:01 | ServiceNow        | Change started           |
| 09:03 | FortiGate         | Policy updated           |
| 09:04 | Firewall Analyzer | New deny flow            |
| 09:05 | Monitoring        | HTTPS alert              |
| 09:06 | BIG-IP            | Pool member marked down  |
| 09:07 | RHEL              | Backend access error     |
| 09:18 | ServiceNow        | Investigation update     |
| 09:27 | Change            | Approved correction      |
| 09:30 | FortiGate         | Expected policy restored |
| 09:31 | BIG-IP            | Member healthy           |
| 09:32 | Monitoring        | Alert cleared            |
| 09:35 | Operator          | Functional test passed   |

Không claim causation chỉ vì timestamp gần nhau; timeline là evidence để support reasoning.

---

# 104. Timeline Timezone Rule

Mọi timestamp phải normalize hoặc annotate.

Ví dụ:

```text
Source timestamp:
2026-10-22 02:03 UTC

Normalized operational time:
2026-10-22 09:03 +07:00
```

Không silently convert.

---

# 105. PHASE 6 — Approved Corrective Operation / Rollback Mandatory Task

Mỗi learner phải thực hiện ít nhất một.

Có thể là:

```text
restart/start approved service
restore approved file permission
revert safe firewall fixture
restore route
restore VM network assignment
rollback script version
restore configuration
```

---

# 106. Corrective Action Record

Trước action:

```text
Ticket:
CHG001408

Asset:
RHEL-APP-01

Current state:
nginx stopped

Expected state:
nginx active

Approved action:
start nginx

Rollback:
stop nginx if unexpected behavior

Success criteria:
service active
port listening
local HTTP OK
BIG-IP member healthy
external HTTPS OK
```

Đây là mindset chuẩn.

---

# 107. Pre-change Evidence

Phải giữ:

```text
configuration/current state
service state
relevant logs
monitoring
related dependency states
```

Nếu không, bạn khó chứng minh improvement.

---

# 108. Execute Once, Observe

Không:

```text
restart
restart
restart
restart
```

Nếu first corrective action fail:

```text
capture failure
reassess hypothesis
```

Repeated changes có thể phá evidence và tăng blast radius.

---

# 109. Post-change Verification

Luôn verify từ **nhiều layer**.

Ví dụ backend service:

```text
systemd
→ listener
→ local transaction
→ BIG-IP health
→ client transaction
→ monitoring
```

---

# 110. Rollback Trigger

Ví dụ:

```text
Service fails to start
Unexpected error increases
Second dependent service fails
Health monitor remains down
New critical alert appears
```

Trigger phải được định nghĩa trước.

---

# 111. Rollback Verification

Rollback không kết thúc khi command rollback chạy.

Phải verify:

```text
original configuration restored
old operational state restored
no residual partial state
monitoring understood
```

---

# 112. PHASE 7 — Automation Regression Lab

Syllabus bắt buộc chạy VBA/Python/Batch/PowerShell utilities trên operational datasets và kiểm tra results/integrity. Infrastructure_Operations_14_Da…

Minimum test catalog còn bao gồm identity matching, encoding, process failures, IAM boundaries, API/TLS, password handling, Outlook/report và repeatability. Infrastructure_Operations_14_Da…

---

# 113. Automation Inputs

Ví dụ:

```text
IdentityMaster.csv
IAM_Raw.csv
Privileged_Raw.csv
ServiceNow_Raw.csv
Asset_Raw.csv
Vulnerabilities.csv
```

Preserve originals.

Không edit raw CSV trong Excel rồi save đè.

---

# 114. Hash Inputs

Linux:

```bash
sha256sum *.csv
```

Windows:

```powershell
Get-FileHash .\IAM_Raw.csv -Algorithm SHA256
```

Store hash in manifest.

---

# 115. Python Regression Run

Conceptual:

```bash
python reconcile.py \
  --input ./working \
  --output ./output
```

Capture:

```text
script version
command
exit code
start/end time
stdout
stderr
```

---

# 116. Batch Wrapper

Example:

```bat
@echo off
python reconcile.py --input "C:\Day14 Working" --output "C:\Day14 Output"
set RC=%ERRORLEVEL%

echo Child exit code: %RC%
exit /b %RC%
```

Critical concept:

```text
child script failure
→ wrapper must not return success
```

---

# 117. PowerShell Report Run

Could extract bounded event records:

```powershell
.\Export-Events.ps1 `
    -ComputerName "WIN01" `
    -StartTime "2026-10-22T08:45:00+07:00" `
    -EndTime "2026-10-22T09:30:00+07:00" `
    -OutputPath ".\output"
```

Exact parameters depend on your Day 12 script.

---

# 118. VBA Workbook Regression

Open controlled workbook.

Do not enable unknown macro content.

Input:

```text
identity data
asset data
access records
```

Run approved macro.

Expected outputs:

```text
matched IDs
unmatched IDs
duplicates
exceptions
summary
```

Verify leading-zero IDs remain text.

---

# 119. Required Automation Test — Identity Matching

Fixtures:

```text
001234
1234
ABC001
abc001
" USER01 "
duplicate composite keys
same display name/different IDs
```

Acceptance:

```text
No accidental numeric conversion
No silent many-to-many joins
Deterministic exceptions
```

---

# 120. Encoding Test

Fixtures:

```text
UTF-8
UTF-8 BOM
UTF-16
CP932 fixture
Vietnamese characters
Japanese characters
embedded comma
quoted newline
CRLF
LF
malformed bytes
```

Expected behavior:

```text
correct round-trip
```

hoặc:

```text
explicit rejection/quarantine
```

Không silent replacement.

---

# 121. File Failure Test

Test:

```text
Missing input
Locked file
Permission denied
Empty file
Missing header
Path with spaces
```

Expected:

```text
non-success status
clear sanitized error
no misleading report
```

---

# 122. API/TLS Negative Test

Prepared fixture:

```text
401
403
timeout
server error
malformed JSON
invalid certificate
unhealthy member
```

Script phải:

```text
bound timeout
retain TLS verification
detect failure
return failed assertion
sanitize credentials
```

---

# 123. Repeatability Test

Run same inputs twice.

Compare business results.

Allowed differences:

```text
Run ID
Timestamp
Temporary working path
```

Not allowed:

```text
different IAM decisions
different unmatched counts
duplicate tasks
overwritten reviewer decisions
```

---

# 124. Automation Reconciliation

Example count rule:

```text
Raw rows:          1,000
Matched:             930
Unmatched:            30
Duplicates:           15
Rejected malformed:   25
                     ----
Total:              1,000
```

Nếu:

```text
930 + 30 + 15 + 25 != 1000
```

investigate.

Không sign off unexplained discrepancy.

---

# 125. PHASE 8 — Condensed IAM / Access Certification

Day 14 không lặp lại full Day 13 nhưng phải thực hiện condensed review.

Define:

```text
Review as-of:
2026-10-22 09:00 +07:00

Review period:
defined by lab policy

Inactivity threshold:
defined separately
```

---

# 126. Data Sources

Có thể dùng:

```text
IAM_Raw
Privileged_Raw
ServiceNow_Raw
Asset_Raw
IdentityMaster
```

Join bằng:

```text
system
account ID
employee/identity ID
approved composite key
```

Không join display name alone.

---

# 127. IAM Checks

Inspect:

```text
Active accounts
Disabled accounts
Expired accounts
Missing owner
Missing approval
Stale entitlement
Dormant evidence
Unknown last-use
Service accounts
Break-glass accounts
Revoked-but-active access
Duplicate identity
```

---

# 128. Boundary Test

Fixture:

```text
last activity one second before threshold
exactly on threshold
one second after threshold
```

Classification phải theo documented inclusive/exclusive rule.

---

# 129. Unknown Evidence

Nếu:

```text
last_used = blank
```

không tự động classify:

```text
Dormant
```

Correct classification có thể là:

```text
Unknown / Investigate
```

---

# 130. IAM Decisions

For every exception:

```text
KEEP
REVOKE
MODIFY
INVESTIGATE
```

Document rationale.

Example:

```text
Exception:
Break-glass account has no recent use.

Decision:
KEEP

Reason:
Expected low-use emergency account;
owner and controls validated.

Additional action:
Confirm monitoring and credential control.
```

---

# 131. PHASE 9 — Vulnerability Reconciliation

Input:

```text
Vulnerability scanner/export fixture
Asset inventory
CMDB/PalletControl
Business criticality
```

---

# 132. Required Fields

For finding:

```text
Asset
Version
CVE if applicable
Scoring source/version
Severity
Exposure
Business impact
Priority
Owner
Exception
Due date
Retest evidence
```

---

# 133. Classification

Distinguish:

```text
CVE vulnerability
Configuration weakness
IAM exception
Operational incident
```

Không đưa tất cả vào một “Vulnerability” bucket.

---

# 134. Example

```text
Finding:
Unsupported TLS configuration

Asset:
APP-LB-01

CVE:
N/A

Type:
Configuration weakness

Exposure:
Internal only

Business criticality:
High

Priority:
Medium/High per lab policy

Owner:
Load Balancer Team
```

---

# 135. Remediation Tracking

Không chỉ ghi:

```text
Fixed
```

Phải có:

```text
Task
Owner
Action
Due date
Change
Post-change test
Retest result
Evidence ID
```

---

# 136. PHASE 10 — ServiceNow Update

Trong lab, ticket phải phản ánh actual timeline.

Work notes nên ghi:

```text
observation
action
result
evidence reference
next step
```

Ví dụ:

```text
09:18 +07 — BIG-IP pool member APP01 observed down.
Local RHEL check found nginx stopped.
No configuration change performed at this stage.
Evidence: D14-EV-021, D14-EV-022.
```

---

# 137. Change Update

Implementation:

```text
09:27 +07 — Approved CHG001408 implementation started.
Action: Start nginx service on RHEL-APP-01.
```

Postcheck:

```text
09:31 +07 — Service active, TCP/443 listening,
BIG-IP member healthy, external HTTPS test passed.
```

---

# 138. Sensitive Information

Không paste:

```text
password
session token
private key
full confidential payload
```

vào ticket.

---

# 139. ServiceNow Evidence

Nếu export permitted:

Preserve:

```text
Ticket number
sys_id where authorized
State
Assignment
Timestamps
Approval
Work notes/audit reference
```

Syllabus yêu cầu stable identifiers và source-to-ticket traceability. Infrastructure_Operations_14_Da…

---

# 140. PHASE 11 — PalletControl / Asset Reconciliation

Compare:

```text
Observed infrastructure
vs
PalletControl
vs
ServiceNow/CMDB or supplied inventory
```

---

# 141. Detect

```text
Missing asset
Duplicate asset
Stale record
Wrong owner
Wrong lifecycle status
Stale inventory timestamp
```

---

# 142. Do Not Immediately Correct

Ví dụ:

```text
VM exists in Prism
but not in PalletControl
```

Could mean:

```text
inventory collection delay
new asset not yet onboarded
collection job failed
asset deliberately excluded
```

Investigate source.

---

# 143. Controlled Correction

Nếu approved:

```text
before evidence
authorized correction
after evidence
ticket linkage
```

Nếu actual PalletControl unavailable:

use fixture and classify:

```text
S = simulation
```

không:

```text
H = hands-on
```

---

# 144. PHASE 12 — Centralized Evidence Correlation

Tạo incident package với:

```text
Server evidence
Network evidence
Firewall evidence
Load balancer evidence
Cloud evidence
Backup evidence
Monitoring evidence
ServiceNow evidence
IAM evidence
```

---

# 145. Correlation Example

Suppose root cause = incorrect FortiGate object.

Supporting evidence:

```text
D14-EV-014
Network route healthy

D14-EV-017
FortiGate expected policy not matching

D14-EV-018
Firewall deny event

D14-EV-022
Backend service healthy

D14-EV-024
BIG-IP member recovered after correction

D14-EV-026
External HTTPS transaction passed
```

Đây mạnh hơn chỉ một screenshot firewall.

---

# 146. PHASE 13 — Peer Review

Additive Day 14 bắt buộc peer kiểm tra source-to-report traceability và reproduce một transformation. Infrastructure_Operations_14_Da…

Peer không chỉ đọc report.

Peer phải chọn:

```text
one output row
```

và trace ngược:

```text
Report row
 ↓
Transformation
 ↓
Script/version
 ↓
Working input
 ↓
Raw source row
 ↓
Evidence/ticket
```

---

# 147. Reproduction Test

Peer receives:

```text
Raw input
Script
Version
Run command
Rules
```

Peer chạy lại.

Expected:

```text
same business results
```

Differences allowed:

```text
timestamp
run ID
```

---

# 148. Peer Review Questions

Reviewer nên kiểm tra:

```text
Were raw files changed?
Do counts reconcile?
Are hashes recorded?
Are secrets absent?
Are decisions traceable?
Does report match evidence?
Was rollback defined?
Were unknowns marked unknown?
```

---

# 149. Peer Review Finding

Ví dụ:

```text
Finding PR-03:
Derived CSV contains 997 rows but source reconciliation
accounts for only 995.

Status:
OPEN

Severity:
Acceptance blocker

Required action:
Identify two unaccounted rows and rerun.
```

Không sign off.

---

# 150. PHASE 14 — Security Review

Perform final review across:

```text
Accounts
Permissions
IAM
Firewall
TLS
Credentials
Evidence
Automation
Backups
```

---

# 151. Secret Leakage Check

Search working outputs for known placeholder patterns or test secrets.

Do not create a report containing actual production secret just để test.

Check:

```text
log files
CSV
Excel
PowerShell transcripts
Python logs
curl transcripts
ServiceNow attachments
Outlook draft
```

---

# 152. Excessive Permission Review

Look for:

```text
chmod 777
Everyone Full Control
AdministratorAccess
any-any firewall rule
unrestricted privileged role
permanent temporary access
```

Nếu fixture contains them, record finding.

---

# 153. TLS Review

Confirm:

```text
No permanent insecure certificate bypass
Correct trust path
Expected hostname
Expiry acceptable
```

---

# 154. PHASE 15 — Outlook/MAPI Reporting

Chỉ dùng compatible **classic Outlook** nếu lab được chuẩn bị như syllabus yêu cầu. New Outlook không được tự giả định có COM/MAPI/VBA behavior tương tự. Infrastructure_Operations_14_Da…

Objective:

```text
Prepare report draft
Do not automatically send
```

---

# 155. Attachment Allowlist

Chỉ attach:

```text
approved evidence summary
sanitized report
manifest
test summary
```

Không attach:

```text
raw password file
private keys
tokens
unredacted sensitive source unless authorized
```

---

# 156. Filename Collision

Nếu output already exists:

Không silent overwrite.

Behavior:

```text
rename deterministically
or
fail with explicit message
```

---

# 157. Draft Verification

Before completion:

```text
Correct recipient placeholder/test mailbox
Correct subject
Correct body
Correct attachments
No secrets
No accidental production recipient
```

Sau đó giữ ở `Draft`.

---

# 158. PHASE 16 — Shift Handover Rehearsal

Handover phải đủ để một operator mới tiếp quản mà không phải hỏi lại toàn bộ history.

Suggested format:

```text
Shift Handover — Day 14

Primary incident:
INC001401

Service:
APP-LAB

Current status:
Restored / Monitoring

Root cause:
...

Actions completed:
...

Changes:
CHG001408

Verification:
...

Open risks:
...

Open gaps:
...

Monitoring required:
...

Next action:
...

Owner:
...

Evidence package:
...
```

---

# 159. Handover — Open Issues

Mỗi open issue phải có:

```text
Owner
Next action
Due date/window
Risk
Evidence
Escalation path
```

Không ghi:

```text
Please check later.
```

---

# 160. Handover — Product Gap

Ví dụ:

```text
HYCU actual product access unavailable.

Performed:
fixture-based recovery analysis.

Competency classification:
S

Remaining gap:
Actual HYCU isolated restore not independently executed.

Owner:
Training Infrastructure Team.
```

Đây là cách đúng.

---

# 161. PHASE 17 — Competency Matrix

Mỗi skill row phải đánh:

```text
H
S
D
T
```

Trong đó syllabus định nghĩa:

```text
H = actual product hands-on
S = simulation/fixture
D = instructor demonstration
T = theory/reference
```

và phải ghi assisted/independent, evidence ID, outcome và reviewer. Infrastructure_Operations_14_Da…

---

# 162. Example Skill Matrix

| Skill                        | Mode | Independence | Evidence | Result |
| ---------------------------- | ---- | ------------ | -------- | ------ |
| RHEL service troubleshooting | H    | Independent  | EV-021   | Pass   |
| Windows event extraction     | H    | Independent  | EV-031   | Pass   |
| FortiGate investigation      | S    | Assisted     | EV-044   | Pass   |
| HYCU recovery                | S    | Independent  | EV-055   | Pass   |
| HPE firmware operation       | D    | N/A          | EV-060   | Gap    |
| IAM review                   | H/S  | Independent  | EV-071   | Pass   |

Không inflate capability.

---

# 163. PHASE 18 — Residual Gap Register

Example:

| Gap                                    | Reason              | Risk   | Owner      | Action                  |
| -------------------------------------- | ------------------- | ------ | ---------- | ----------------------- |
| Actual HYCU restore not performed      | Product unavailable | Medium | Lab Team   | Schedule product lab    |
| HPE firmware workflow demo only        | No spare hardware   | Medium | Infra Lead | Supervised lab          |
| AccessCheck revocation simulation only | No account          | High   | IAM Team   | Validate in test tenant |

---

# 164. PHASE 19 — Risk Register

Examples:

```text
RISK-01:
Backup recovery not independently tested

RISK-02:
Privileged account owner missing

RISK-03:
Certificate expires in 20 days

RISK-04:
Centralized logging source silent

RISK-05:
Nutanix capacity below internal headroom
```

Each must have:

```text
impact
likelihood
owner
treatment
due date
status
```

---

# 165. PHASE 20 — Final Evidence Integrity Check

Before submission:

Calculate hashes for final evidence items.

Linux:

```bash
sha256sum <file>
```

Windows:

```powershell
Get-FileHash .\file -Algorithm SHA256
```

Update manifest.

---

# 166. Transfer Verification

Nếu files được copy sang final repository:

Recalculate hash.

Expected:

```text
source hash
=
destination hash
```

Nếu mismatch:

```text
STOP
investigate transfer/change
```

Không sign off.

---

# 167. Counts

For CSV/report datasets:

```text
source records
working records
output records
rejected
filtered
duplicates
```

reconcile.

---

# 168. Redacted Evidence

Nếu tạo redacted copy:

```text
Raw:
EV-100

Redacted:
EV-100-R1

Parent:
EV-100

Reason:
Removed employee email addresses

Redacted hash:
...
```

Không replace raw.

---

# 169. Screenshots

Screenshots nên có:

```text
visible asset identity
timestamp context nếu possible
relevant state only
```

Avoid capture:

```text
password fields
tokens
unrelated sensitive tabs
```

Screenshot supplementary, không thay native export nếu native record có sẵn mà không có explanation. Infrastructure_Operations_14_Da…

---

# 170. PHASE 21 — Final Acceptance Review

Reviewer kiểm tra:

```text
Were all original Day 14 scenarios accounted for?
Was individual mandatory work done?
Were critical faults found/escalated?
Are evidence items traceable?
Do counts reconcile?
Are secrets absent?
Were changes authorized?
Was backup recovery actually verified?
Did automation regression pass?
Are IAM decisions traceable?
Are open gaps documented?
```

---

# 171. Automatic Sign-off Blockers

Syllabus đặc biệt không chấp nhận:

```text
silent data loss
exposed secrets
unapproved changes
unexplained hash discrepancy
unexplained count discrepancy
```

Infrastructure_Operations_14_Da…

Ngoài ra completion rubric yêu cầu privileged-access revocation proof, backup recovery verification, critical automation tests và integrity checks nếu applicable; thiếu chúng phải giữ thành open gap. Infrastructure_Operations_14_Da…

---

# 172. Full Troubleshooting Workflow để dùng ở mọi Station

Dùng pattern:

```text
1. Read ticket.
2. Confirm environment.
3. Confirm asset.
4. Confirm authorization.
5. Capture baseline.
6. Reproduce symptom safely.
7. Identify expected state.
8. Start read-only checks.
9. Localize layer.
10. Collect relevant logs.
11. Form hypothesis.
12. Run targeted test.
13. Confirm root cause or reject hypothesis.
14. Check change approval.
15. Capture pre-change evidence.
16. Execute smallest corrective action.
17. Capture actual result.
18. Run postchecks.
19. Check monitoring.
20. Check dependent services.
21. Roll back if criteria fail.
22. Correlate evidence.
23. Update ticket.
24. Record residual risks/gaps.
25. Handover/close.
```

Đây nên trở thành muscle memory.

---

# 173. Rotating Station Design

Vì Day 14 không thể thực hiện full deployment của hàng chục enterprise products từ đầu, syllabus cho phép rotating stations và prepared fault packets.

Một cách tổ chức:

```text
Station A
Linux/Windows

Station B
AWS

Station C
Nutanix/HPE/HYCU

Station D
Cisco/FortiGate

Station E
BIG-IP/TLS

Station F
Logging/Analyzers

Station G
IAM/Vulnerability

Station H
Automation/Reporting
```

Mỗi learner không nhất thiết có actual hands-on trên mọi proprietary tool nhưng phải review tất cả scenario.

---

# 174. Fault Packet Contents

Một fault packet tốt nên chứa:

```text
Scenario ID
Business impact
Ticket
Topology excerpt
Known baseline
Available evidence
Seeded fault
Allowed actions
Forbidden actions
Success criteria
Expected escalation
Answer key for instructor
```

Không cho learner answer key.

---

# 175. Failure Injection Safety

Các lỗi nên inject:

```text
stopped service
wrong test-file permission
wrong lab route
wrong isolated firewall fixture
expired test certificate
powered-off lab VM
prepared full test filesystem
unavailable test backup target
expired synthetic privilege
silent log source fixture
```

Không inject:

```text
production routing loops
real credential lockout
production certificate replacement
destructive disk operations
cluster quorum loss
real ransomware-like behavior
```

---

# 176. Troubleshooting Checkpoint 1 — Before Change

Bạn phải trả lời được:

```text
What exactly is wrong?
Which layer?
What evidence?
What is expected?
What change is proposed?
Who approved it?
What could break?
What is rollback?
```

Nếu không trả lời được, chưa nên change.

---

# 177. Troubleshooting Checkpoint 2 — After Change

Phải trả lời:

```text
Did the state change?
Did the symptom disappear?
Did dependent layers recover?
Any new alert?
Was rollback needed?
What evidence proves recovery?
```

---

# 178. Troubleshooting Checkpoint 3 — Before Closure

Phải trả lời:

```text
Is business function restored?
Is monitoring normal?
Is backup protection normal?
Is security posture acceptable?
Are open issues assigned?
Is evidence complete?
Has ServiceNow been updated?
Can another operator reproduce understanding?
```

---

# 179. L1/L2 Escalation Boundary

Escalate khi:

```text
required privilege exceeds authorization
root cause unresolved after runbook
hardware failure suspected
cluster resilience compromised
security compromise suspected
backup recovery uncertain
vendor defect suspected
high-risk config change required
rollback unavailable
business impact increasing
evidence contradictory
```

Không cố “fix cho xong”.

---

# 180. Escalation Record

Example:

```text
Escalation:
Nutanix L3 / Vendor

Reason:
Repeated CVM service alert after approved checks.

Impact:
2 application VMs at risk.

Completed checks:
- cluster health
- capacity
- host state
- VM placement
- recent tasks

No corrective action performed because:
Runbook requires vendor guidance.

Evidence:
EV-081–EV-087
```

Đây là một **successful operational decision**, không phải fail.

---

# 181. Final Architecture Artifact

Final package cần architecture view.

Nó phải đủ để reviewer hiểu:

```text
server placement
cloud components
network path
firewall
load balancer
virtualization
backup
monitoring
ITSM
```

Không cần diagram đẹp như enterprise design document; cần chính xác và traceable.

---

# 182. Configuration Inventory

Example fields:

```text
Asset ID
Hostname
Environment
Platform
Version
IP
Role
Owner
Monitoring
Backup
Critical dependency
Evidence reference
```

---

# 183. Runbook Package

Runbooks có thể gồm:

```text
Server health check
Network connectivity diagnosis
Firewall investigation
BIG-IP health check
Backup recovery verification
Log extraction
IAM exception handling
Automation execution
Handover
```

Mỗi runbook nên phân biệt:

```text
read-only
changing
approval requirement
rollback
escalation
```

---

# 184. Operational Checklist

Example:

```text
[ ] Ticket validated
[ ] Environment confirmed
[ ] Authorization confirmed
[ ] Baseline captured
[ ] Server health verified
[ ] AWS health verified
[ ] Nutanix/HPE verified
[ ] Network verified
[ ] Firewall verified
[ ] BIG-IP verified
[ ] Backup verified
[ ] Logging verified
[ ] Corrective operation completed
[ ] Recovery tested
[ ] IAM decision completed
[ ] Vulnerability review completed
[ ] Automation regression completed
[ ] Evidence reconciled
[ ] ServiceNow updated
[ ] Assets updated
[ ] Peer review completed
[ ] Handover completed
```

---

# 185. Test Results Package

Mỗi test:

```text
Test ID
Requirement
Fixture/version
Expected
Actual
Pass/Fail
Defect
Retest
Evidence
```

Syllabus nói rõ một case không thể được đánh `Pass` chỉ vì script chạy xong nếu expected result không đạt. Infrastructure_Operations_14_Da…

---

# 186. Reviewer Decisions

Một reviewer decision phải rõ:

```text
Accepted
Accepted with gap
Rejected
Retest required
Escalation required
```

Không dùng từ mơ hồ:

```text
Looks fine.
```

---

# 187. Final Handover Scenario

Bạn nên thực hành 5 phút handover bằng lời.

Structure:

```text
1. Service/incident
2. Business impact
3. Root cause/current hypothesis
4. Actions completed
5. Current state
6. Open issues
7. Risk
8. Next actions
9. Owners
10. Evidence/ticket references
```

Không đọc toàn bộ ticket.

---

# 188. Mentor Challenge Questions

Trong final review mentor có thể hỏi:

**“Tại sao bạn kết luận firewall là root cause?”**

Bạn phải trỏ vào evidence.

**“Nếu backup job success thì tại sao vẫn restore?”**

Bạn phải nói backup execution không chứng minh recoverability.

**“Tại sao không disable SELinux?”**

Vì đó không phải least-risk root-cause correction.

**“Tại sao bạn không reboot VM?”**

Vì root cause chưa localize hoặc reboot có blast radius.

**“Hash match chứng minh gì?”**

Chứng minh bytes match trusted recorded baseline, không chứng minh source/query completeness.

**“Blank last-used nghĩa dormant?”**

Không. Có thể là unknown evidence.

**“Tại sao scenario FortiGate của bạn là S chứ không phải H?”**

Vì dùng version-labelled fixture thay vì actual product lab.

---

# 189. Final Practical Competence Model

Sau Day 14, mục tiêu không phải:

```text
"I have used every enterprise product."
```

mà là:

```text
"I can safely operate inside a defined scope,
verify state,
troubleshoot systematically,
collect defensible evidence,
make controlled corrections,
verify recovery,
know when to stop,
and hand over accurately."
```

Syllabus cũng nói rõ chương trình là accelerated operational-readiness program; actual mastery/hands-on phải được phân biệt với simulation/demonstration/theory. Infrastructure_Operations_14_Da…

---

# 190. Day 14 Practical Completion Checklist

## Operational control

- [ ] Environment xác định đúng.
- [ ] Ticket/scope đọc và hiểu.
- [ ] Authorization được xác minh.
- [ ] Recent changes được review.
- [ ] Maintenance/change boundary rõ.
- [ ] Rollback và stop conditions rõ.

## Infrastructure health

- [ ] Linux/RHEL health verified.
- [ ] Windows Server health verified.
- [ ] AWS identity/resources/network verified.
- [ ] HPE/iLO health reviewed.
- [ ] Nutanix cluster/VM state reviewed.
- [ ] HYCU protection reviewed.
- [ ] Catalyst/Nexus L2/L3 path reviewed.
- [ ] FortiGate path/policy reviewed.
- [ ] BIG-IP virtual/pool/member reviewed.
- [ ] Monitoring/logging health reviewed.

## Mandatory individual work

- [ ] Server-log extraction.
- [ ] Approved corrective operation hoặc rollback.
- [ ] Isolated recovery validation.
- [ ] IAM exception decision.
- [ ] Automation regression run.

## Failure scenarios

- [ ] Service.
- [ ] Permission.
- [ ] Routing.
- [ ] Firewall.
- [ ] SSL certificate.
- [ ] VM availability.
- [ ] Storage capacity.
- [ ] Backup.
- [ ] Privileged access.
- [ ] Logging.

## Governance

- [ ] Condensed IAM certification.
- [ ] Vulnerability reconciliation.
- [ ] ServiceNow updated.
- [ ] Asset inventory reconciliation.
- [ ] Risk register updated.
- [ ] Residual gaps recorded.

## Evidence

- [ ] Raw exports preserved.
- [ ] Working copies separate.
- [ ] Evidence IDs assigned.
- [ ] Timestamps/timezones recorded.
- [ ] Exact filters/commands recorded.
- [ ] Counts reconciled.
- [ ] SHA-256 recorded where applicable.
- [ ] Secret leakage checked.
- [ ] Evidence gaps documented.

## Automation

- [ ] Expected vs actual recorded.
- [ ] Identity matching tests pass.
- [ ] Encoding tests pass.
- [ ] Failure tests behave correctly.
- [ ] TLS/API negative tests handled safely where applicable.
- [ ] Repeatability verified.
- [ ] No silent data loss.
- [ ] No duplicate side effects.

## Acceptance

- [ ] Peer source-to-report traceability check.
- [ ] Selected transformation reproduced.
- [ ] H/S/D/T correctly recorded.
- [ ] Assisted/independent status recorded.
- [ ] Mentor/reviewer decisions recorded.
- [ ] Shift handover rehearsed.
- [ ] Final package complete.
- [ ] Open gaps have owner/action.

---

# 191. Final Deliverables

Kết thúc Day 14, final package phải bao gồm tối thiểu:

```text
Final architecture
Configuration inventory
Runbooks
Operational checklist
Server/network/cloud/security evidence
Backup/recovery evidence
Automation outputs
Test results
IAM review
Vulnerability register
ServiceNow records
Asset reconciliation
Evidence manifest
Risk register
Residual-gap register
Competency/skill matrix
Reviewer decisions
Peer review notes
Shift handover
Acceptance summary
```

Đây chính là package mà syllabus yêu cầu để kết thúc integrated capstone. Infrastructure_Operations_14_Da…

---

# 192. Điều kiện PASS

Bạn có thể được đánh giá **PASS trong phạm vi đã thực hiện** khi:

```text
All required scenarios accounted for
Mandatory individual tasks completed
Critical faults found or correctly escalated
Changes authorized
Recovery demonstrated
Automation correctly tested
Counts/hashes reconciled
Evidence traceable
No secret exposure
Open gaps explicitly recorded
H/S/D/T classifications honest
Handover complete
```

---

# 193. Điều kiện PARTIALLY CORRECT / GAP

Ví dụ:

```text
Technical root cause correct
but evidence incomplete
```

hoặc:

```text
Backup job fixed
but no restore performed
```

hoặc:

```text
IAM decision reasonable
but no ticket/evidence link
```

hoặc:

```text
FortiGate workflow understood
but only demonstration observed
```

Không được nâng những trường hợp này thành full hands-on completion.

---

# 194. Điều kiện FAIL / SIGN-OFF BLOCKER

Ví dụ nghiêm trọng:

```text
Unapproved production-like change
Plaintext secrets in evidence
Silent record loss
Fabricated evidence
Unexplained count/hash mismatch
Claimed restore without actual verification
Claimed revocation without checking effective access
Ignored critical security/hardware fault
Changed firewall broadly to make test pass
Disabled TLS verification and claimed success
Destroyed raw evidence
```

Những vấn đề này không thể giải quyết bằng việc “report trông đẹp”.

---

# 195. Mental Runbook cuối cùng

Khi bạn thực sự đi làm Infrastructure Operations, hãy ghi nhớ workflow này:

```text
TICKET
  ↓
SCOPE
  ↓
AUTHORIZATION
  ↓
BASELINE
  ↓
OBSERVE
  ↓
VERIFY
  ↓
LOCALIZE
  ↓
EVIDENCE
  ↓
HYPOTHESIS
  ↓
TARGETED TEST
  ↓
ROOT CAUSE
  ↓
CHANGE GATE
  ↓
MINIMUM SAFE ACTION
  ↓
POSTCHECK
  ↓
RECOVERY TEST
  ↓
MONITORING
  ↓
AUDIT EVIDENCE
  ↓
TICKET / ASSET / IAM UPDATE
  ↓
PEER REVIEW
  ↓
ESCALATE OR CLOSE
  ↓
HANDOVER
```

Đây chính là điểm cuối của **Module Thực hành Day 14**: không phải chứng minh bạn nhớ nhiều commands, mà chứng minh bạn có thể **vận hành một environment tích hợp một cách an toàn, có kiểm chứng, có rollback, có evidence, có audit trail, biết giới hạn quyền hạn và biết escalation/handover đúng chuẩn enterprise**. Theo syllabus, final sign-off không được chấp nhận silent data loss, exposed secrets, unapproved changes hoặc unexplained evidence discrepancies; đồng thời những hands-on chưa thực hiện phải tiếp tục được ghi là residual gap chứ không biến mất chỉ vì khóa học đã kết thúc. Infrastructure_Operations_14_Da… Infrastructure_Operations_14_Da…
