# Day 7 — Module Thực hành

## Nutanix AOS/AHV Operations, Maintenance, Troubleshooting & HYCU Backup/Recovery

Module Thực hành Day 7 là nơi toàn bộ kiến thức về **AOS, AHV, Prism Element, Prism Central, CVM, NCC, LCM, VM operations và HYCU** được chuyển thành operational workflow thực tế.

Theo syllabus, bạn phải thực hiện hoặc mô phỏng có kiểm soát toàn bộ chuỗi sau: provision/configure AHV VM qua Prism; gán network và storage; kiểm tra guest connectivity và utilization; thực hiện controlled power/snapshot operations; thu thập Task/Alert/Audit/cluster-health evidence; xử lý simulated capacity hoặc VM-availability problem; lập maintenance plan; rehearsal AHV host maintenance với pre/post checks; cấu hình hoặc inspect HYCU environment; source/target/policy; chạy và monitor backup; inject backup failure; đọc logs; sửa lỗi có kiểm soát; rerun; isolated VM/file restore; content validation; đo observed recovery time; và chuẩn bị maintenance/recovery/escalation evidence. Infrastructure_Operations_14_Da…

Một yêu cầu quan trọng của toàn curriculum là **backup success không chứng minh DR readiness**. Với HYCU, syllabus đòi hỏi isolated restoration, integrity validation và observed recovery time. Các hoạt động như hyperconverged cluster maintenance hoặc firmware/software lifecycle work cũng phải có authorization, rollback/backout planning và supervision. Infrastructure_Operations_14_Da…

---

# 1. Mục tiêu cuối cùng của Module Thực hành

Sau khi hoàn tất Day 7, mục tiêu không phải chỉ là “biết bấm Prism” hoặc “đã từng chạy một HYCU backup”.

Bạn phải có khả năng đi qua toàn bộ operational loop:

```text
Understand expected state
        ↓
Verify identity/scope
        ↓
Collect baseline
        ↓
Perform authorized operation
        ↓
Observe task/job execution
        ↓
Verify technical outcome
        ↓
Verify service/data outcome
        ↓
Collect evidence
        ↓
Inject/troubleshoot failure
        ↓
Apply lowest-risk correction
        ↓
Retest
        ↓
Rollback / Cleanup
        ↓
Escalate unresolved conditions
```

Với Nutanix:

```text
Prism
  ↓
Cluster
  ↓
AHV Host
  ↓
CVM / AOS
  ↓
Storage + Network
  ↓
VM
  ↓
Guest OS
  ↓
Application
```

Với HYCU:

```text
Source
  ↓
Protection Policy
  ↓
Backup Job
  ↓
Backup Target
  ↓
Recovery Point
  ↓
Restore
  ↓
Guest/Application Validation
  ↓
Data Integrity Validation
```

Đây mới là “hoàn thành Day 7”.

---

# 2. Môi trường Lab đề xuất

Toàn bộ lab mặc định là:

> **LAB / NON-PRODUCTION / AUTHORIZED ENVIRONMENT ONLY**

Một topology phù hợp:

```text
                       ┌──────────────────────────┐
                       │      Prism Central       │
                       │       LAB-PC-01          │
                       └────────────┬─────────────┘
                                    │
                                    │
                       ┌────────────▼─────────────┐
                       │      Prism Element       │
                       │       LAB-PE-01          │
                       └────────────┬─────────────┘
                                    │
                       Nutanix AHV Cluster
                         LAB-NTNX-CL01
                                    │
             ┌──────────────────────┼─────────────────────┐
             │                      │                     │
             ▼                      ▼                     ▼
         AHV-01                 AHV-02                AHV-03
           │                      │                     │
          CVM                    CVM                   CVM
           │                      │                     │
           └──────────── AOS Distributed ──────────────┘
                                    │
                    ┌───────────────┼────────────────┐
                    │               │                │
                    ▼               ▼                ▼
                LAB-VM01         LAB-VM02       HYCU Controller
                                                  │
                                                  ▼
                                              Backup Target
```

Networking có thể có:

```text
VLAN 110  = LAB-MGMT
VLAN 120  = LAB-SERVER
VLAN 130  = LAB-BACKUP
VLAN 199  = LAB-ISOLATED-RESTORE
```

`LAB-ISOLATED-RESTORE` đặc biệt quan trọng. Đây là network không có đường trực tiếp gây conflict với production-equivalent workload.

---

# 3. Prerequisites trước khi làm bất kỳ lab nào

Trước khi thao tác, bạn cần lập **Lab Control Sheet**.

| Item                 | Ví dụ                                          |
| -------------------- | ---------------------------------------------- |
| Environment          | LAB                                            |
| Prism Central        | `LAB-PC-01`                                    |
| Prism Element        | `LAB-PE-01`                                    |
| Cluster              | `LAB-NTNX-CL01`                                |
| AOS version          | `<deployed-version>`                           |
| AHV version          | `<deployed-version>`                           |
| NCC version          | `<deployed-version>`                           |
| LCM version          | `<deployed-version>`                           |
| HYCU version         | `<deployed-version>`                           |
| Test VM              | `D7-RHEL01`                                    |
| Production-like VLAN | `LAB-SERVER`                                   |
| Restore VLAN         | `LAB-ISOLATED-RESTORE`                         |
| Storage container    | `<approved-container>`                         |
| HYCU source          | `<approved-source>`                            |
| HYCU target          | `<approved-target>`                            |
| Ticket               | `CHG-LAB-D7-001`                               |
| Operator             | `<learner-id>`                                 |
| Supervisor           | `<mentor>`                                     |
| Timezone             | `Asia/Bangkok` hoặc timezone lab được quy định |

Lý do phải ghi version là vì Prism, AHV, NCC, LCM và HYCU đều là **version-sensitive**.

Không nên ghi:

```text
Go to menu X → Y → Z
```

như một procedure universal.

Đúng hơn:

```text
Perform operation X using the approved workflow
for Prism/AHV/HYCU version currently deployed.
```

Tài liệu Nutanix hiện hành chẳng hạn cho phép VM creation qua Prism Central và expose các object như cluster, vCPU, memory, storage container, subnet/VLAN và boot configuration; vị trí UI cụ thể thay đổi theo Prism release. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Prism-Central-Guide-vpc_2024_3-ja-JP%3Amul-vm-create-acropolis-pc-t.html&utm_source=chatgpt.com)

---

# 4. Evidence repository

Ngay đầu Day 7, tạo structure:

```text
Day7/
├── 00_Control/
├── 01_Baseline/
├── 02_VM_Provision/
├── 03_VM_Operations/
├── 04_Cluster_Health/
├── 05_Troubleshooting/
├── 06_Host_Maintenance/
├── 07_HYCU_Backup/
├── 08_HYCU_Failure/
├── 09_Restore/
├── 10_PostChecks/
└── 11_Final_Report/
```

Tên file nên có:

```text
<EvidenceID>_<Asset>_<Action>_<Timestamp>.<ext>
```

Ví dụ:

```text
D7-E001_LAB-NTNX-CL01_PreHealth_20261021T133000+0700.txt
D7-E012_D7-RHEL01_VMConfig_20261021T140500+0700.png
D7-E031_HYCU_JOB1234_Failure_20261021T160000+0700.txt
```

Một evidence item tốt cần trace được:

```text
Evidence
→ source system
→ object
→ timestamp
→ operator
→ ticket
→ operation
→ result
```

Syllabus chung còn yêu cầu giữ raw evidence, metadata, timestamp/timezone, source, query/command, hash/count và limitations khi phù hợp. Infrastructure_Operations_14_Da…

---

# LAB 1 — Baseline Nutanix Environment

## 5. Objective

Trước khi tạo VM hoặc thay đổi gì, bạn phải biết cluster đang ở trạng thái nào.

Mục tiêu Lab 1 là trả lời:

```text
Tôi đang ở đúng cluster không?
Cluster có healthy trước khi tôi thay đổi không?
Có critical alert tồn tại sẵn không?
Capacity hiện tại ra sao?
CVM/host state ra sao?
NCC state hiện tại thế nào?
```

Nếu không có baseline, khi sự cố xảy ra bạn không phân biệt được:

```text
Pre-existing problem
```

và

```text
Problem introduced by my change
```

---

# 6. Bước 1 — Identity verification

Trong Prism, ghi nhận:

```text
Prism instance
Cluster name
Cluster UUID nếu được phép
AOS version
AHV version
Number of nodes
CVMs
Storage containers
Networks/subnets
```

Không bắt đầu từ VM.

Bắt đầu từ **environment identity**.

Expected:

```text
Environment = LAB
Cluster = approved cluster
Operator has intended role
```

Nếu thấy cluster production hoặc object không khớp ticket:

```text
STOP
```

---

# 7. Bước 2 — RBAC verification

Bạn cần xác định account có quyền gì.

Mục tiêu là không dùng excessive privilege.

Ví dụ lab cần:

```text
View cluster
View VM
Create/edit approved VM
Power approved VM
Create approved snapshot/recovery point
View Tasks/Alerts
```

Không có lý do tự động cần:

```text
Cluster shutdown
LCM upgrade execution
RBAC modification
Network infrastructure modification
```

Nếu account có quyền quá rộng, vẫn chỉ sử dụng scope được ticket cho phép.

---

# 8. Bước 3 — Cluster health inspection

Kiểm tra trong Prism:

```text
Overall cluster health
Critical alerts
Warnings
Hosts
CVMs
Storage
Capacity
VM state
Tasks
```

Ở đây chỉ **read-only verification**.

Không acknowledge alert ngay chỉ để dashboard “sạch”.

Syllabus Day 7 yêu cầu hiểu rõ:

```text
Alert acknowledgement
≠
Alert resolution
```

Infrastructure_Operations_14_Da…

---

# 9. Bước 4 — CVM/cluster service check

Nếu lab cho SSH read-only vào CVM, một command thường dùng là:

```bash
cluster status
```

Mục đích:

```text
Inspect cluster/CVM service state
```

Đây là read-only check.

Không được biến thành:

```bash
cluster stop
```

hoặc một service restart nào khác ngoài approved runbook.

---

# 10. Bước 5 — Critical alert inspection

Tài liệu AHV 11.2 hiện tại yêu cầu review và resolve critical alerts trước các operation như CVM/AHV restart hoặc host maintenance. Nutanix cũng đưa ví dụ read-only inspection:

````bash
ncli alert ls
``` :chatgpt-content-reference{index="5"}


Ở Day 7, nếu sử dụng command này, mục tiêu là:

```text
collect alerts
````

không phải tự sửa tất cả alerts.

---

# 11. Bước 6 — NCC baseline

Full NCC check có thể chạy từ CVM:

```bash
ncc health_checks run_all
```

Nutanix Support documentation hiện tại vẫn xác nhận command này và cho biết output được lưu thành NCC log sau khi hoàn tất. NCC cũng có thể được kích hoạt từ Prism Health UI. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details/?targetId=kA032000000986SCAQ&utm_source=chatgpt.com)

Đây là một operation health-check, không phải configuration change, nhưng vẫn nên chạy theo lab/runbook vì full NCC có thể mất thời gian và tạo load nhất định.

Bạn cần record:

```text
NCC version
Start time
End time
PASS count
WARN
FAIL
ERROR
Relevant check names
Output location
```

Không được record đơn giản:

```text
NCC = OK
```

---

# 12. Baseline acceptance

Trước Lab 2 phải trả lời được:

```text
Correct environment? YES
Cluster healthy enough for lab? YES
Critical unresolved issue? NO
Capacity suitable? YES
CVMs/services healthy? YES
NCC blocker? NO
```

Nếu một câu quan trọng là `NO`:

```text
DO NOT PROCEED
```

Escalate cho instructor/platform owner.

---

# LAB 2 — Provision AHV VM qua Prism

Đây là yêu cầu trực tiếp của syllabus. Infrastructure_Operations_14_Da…

---

# 13. Objective

Tạo một controlled VM:

```text
D7-RHEL01
```

với:

```text
approved CPU
approved memory
approved virtual disk
approved storage container
approved subnet/network
correct VLAN
```

Sau đó chứng minh guest boot và network connectivity hoạt động.

---

# 14. Build sheet mẫu

Không “ước lượng” resources lúc đang bấm UI.

Chuẩn bị:

| Field       | Lab value                 |
| ----------- | ------------------------- |
| VM Name     | `D7-RHEL01`               |
| vCPU        | 2                         |
| Memory      | 4 GiB                     |
| System Disk | 30 GiB                    |
| Data Disk   | 5 GiB optional            |
| Container   | `<LAB-container>`         |
| Network     | `<LAB-server-network>`    |
| VLAN        | `<approved-VLAN>`         |
| Guest       | RHEL-compatible lab image |
| Boot        | Theo image                |
| IP          | `<lab-IP>` hoặc DHCP      |
| Gateway     | `<lab-gateway>`           |
| DNS         | `<lab-DNS>`               |
| Owner       | `<learner>`               |
| Ticket      | `CHG-LAB-D7-001`          |

Các con số này chỉ là lab example, không phải sizing recommendation cho production.

---

# 15. VM creation workflow

Operational sequence:

```text
Prism
 ↓
Compute / VMs
 ↓
Create VM
 ↓
Verify target cluster
 ↓
Set identity
 ↓
Allocate CPU/memory
 ↓
Attach boot/system disk
 ↓
Select approved storage container
 ↓
Attach network/subnet
 ↓
Verify associated VLAN
 ↓
Review configuration
 ↓
Create
```

Tài liệu Prism hiện tại xác nhận VM creation qua Prism Central có các bước chọn cluster, vCPU, memory, disk/container và subnet; VLAN ID của selected subnet có thể được hiển thị để kiểm tra. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Prism-Central-Guide-vpc_2024_3-ja-JP%3Amul-vm-create-acropolis-pc-t.html&utm_source=chatgpt.com)

Điểm quan trọng:

> **Review trước Create.**

Không chỉ click nhanh qua wizard.

---

# 16. Pre-create verification

So sánh:

```text
Build sheet
          versus
Prism review screen
```

Kiểm từng item.

Ví dụ:

```text
Build sheet:
VLAN 120

Prism:
VLAN 130
```

→ `STOP`.

Không tạo VM rồi mới “sửa sau”.

---

# 17. Sau khi Create

Đừng lập tức Power On.

Trước tiên kiểm:

```text
VM exists
VM name correct
Cluster correct
vCPU correct
Memory correct
Disk correct
Container correct
NIC attached
Network correct
```

Capture:

```text
Task ID / task reference
timestamp
operator
VM configuration
```

---

# LAB 3 — Guest Boot, Console & Connectivity Verification

## 18. Controlled Power On

Sau configuration review:

```text
Power On
```

Đây là **change operation**, nhưng blast radius thấp trong lab nếu VM là disposable test workload.

Observe task.

Expected:

```text
Task succeeds
VM state = powered on
```

Không chỉ xem icon màu xanh.

---

# 19. Console verification

Mở VM console.

Kiểm:

```text
Guest boot completes
No obvious kernel panic
Login prompt available
Expected hostname
Expected OS
```

Console rất hữu ích vì nó độc lập với việc guest network có hoạt động hay không.

Nếu SSH không vào được nhưng console hoạt động:

```text
VM itself may be healthy;
network/guest configuration becomes likely investigation area.
```

---

# 20. Linux guest verification

Trong RHEL/Linux lab guest:

```bash
hostnamectl
```

Mục đích:

```text
Verify hostname and OS identity.
```

```bash
ip addr
```

Mục đích:

```text
Verify interface and IP assignment.
```

```bash
ip route
```

Mục đích:

```text
Verify local routes and default gateway.
```

```bash
cat /etc/resolv.conf
```

hoặc resolver tooling phù hợp với distro.

Mục đích:

```text
Inspect DNS configuration.
```

---

# 21. Connectivity test theo layer

Không test bằng một lệnh duy nhất.

Thứ tự:

```text
NIC exists
 ↓
IP exists
 ↓
Local subnet
 ↓
Default gateway
 ↓
Remote IP
 ↓
DNS resolution
 ↓
Application port
```

Ví dụ:

```bash
ping -c 4 <gateway>
```

Sau đó:

```bash
ping -c 4 <approved-test-ip>
```

Nếu ICMP được phép.

DNS:

```bash
getent hosts <approved-hostname>
```

Service:

```bash
curl -I --connect-timeout 5 http://<approved-test-service>
```

hoặc:

```bash
nc -vz <host> <port>
```

nếu package/tool đã được approved.

---

# 22. Test cả success và denied path

Operations verification không chỉ hỏi:

```text
Có connect được không?
```

Mà còn:

```text
Có connect đúng những nơi được phép không?
```

Ví dụ:

```text
Expected allowed:
VM → DNS
VM → test web endpoint

Expected denied:
VM → unauthorized management network
```

Denied result cũng là valid evidence nếu đúng design.

---

# 23. Windows guest equivalent

Nếu lab dùng Windows VM:

```powershell
Get-NetIPConfiguration
```

```powershell
Get-NetRoute
```

```powershell
Test-NetConnection <host> -Port <port>
```

```powershell
Resolve-DnsName <hostname>
```

Không cần chạy cả Windows lẫn Linux nếu syllabus environment chỉ cung cấp một VM. Mục tiêu là hiểu workflow.

---

# LAB 4 — Resource Utilization Inspection

## 24. Objective

Chứng minh bạn phân biệt được:

```text
Configured resources
```

và:

```text
Actual utilization
```

---

# 25. Prism-side checks

Ghi:

```text
Configured vCPU
Configured memory
Disk capacity
Host placement
Current CPU utilization
Current memory utilization
Storage usage
```

Không được kết luận chỉ từ instantaneous spike.

Ví dụ:

```text
CPU 90% at one instant
```

không đủ để chứng minh sustained CPU pressure.

Bạn cần trend/window phù hợp.

---

# 26. Guest-side correlation

Linux:

```bash
uptime
```

```bash
free -h
```

```bash
df -h
```

```bash
df -i
```

```bash
top
```

hoặc:

```bash
ps -eo pid,comm,%cpu,%mem --sort=-%cpu | head
```

Mục tiêu:

```text
Prism sees VM CPU high
           ↕
Guest process/utilization
```

Correlation này giúp tránh “Prism CPU high → add CPU ngay”.

---

# 27. Storage differentiation

Ví dụ guest:

```bash
df -h
```

cho:

```text
/ = 95% used
```

nhưng Prism cluster:

```text
Storage = 45% utilized
```

Hai kết quả không mâu thuẫn.

Chúng nói về hai layer:

```text
Guest filesystem capacity
vs
Cluster storage capacity
```

Bạn phải chứng minh mình hiểu distinction này.

---

# LAB 5 — Controlled VM Power Operations

## 28. Objective

Thực hiện power lifecycle có kiểm soát và hiểu blast radius.

Sequence đề xuất:

```text
Guest graceful shutdown
 ↓
Verify Prism state
 ↓
Power on
 ↓
Verify guest
```

Không bắt đầu bằng force power-off.

---

# 29. Graceful shutdown

Linux:

```bash
sudo shutdown -h now
```

nếu learner được approved sudo.

Expected:

```text
Guest shuts down cleanly
Prism eventually shows powered off
```

Capture:

```text
guest command/time
Prism Task
VM state
```

Sau đó Power On qua approved Prism operation.

Verify lại:

```bash
uptime
```

```bash
systemctl --failed
```

và network/application check.

---

# 30. Force power-off

Chỉ làm nếu lab explicitly yêu cầu simulation.

Phải ghi:

```text
Risk:
Equivalent to abrupt loss of power.

Possible impact:
Filesystem/application inconsistency.
```

Không dùng force operation chỉ vì graceful shutdown mất vài giây.

---

# LAB 6 — Controlled Snapshot / Recovery Point Operations

## 31. Objective

Tạo một point-in-time protection object cho VM và hiểu rằng:

```text
Snapshot
≠
Independent backup
```

Syllabus đặc biệt bắt buộc distinction này. Infrastructure_Operations_14_Da…

---

# 32. Chuẩn bị deterministic test data

Trong VM:

```bash
sudo mkdir -p /opt/day7-lab
```

```bash
echo "Day7 baseline $(date -Is)" | sudo tee /opt/day7-lab/baseline.txt
```

Tạo checksum:

```bash
sha256sum /opt/day7-lab/baseline.txt
```

Ví dụ:

```text
a3f...  /opt/day7-lab/baseline.txt
```

Lưu checksum ra evidence repository.

---

# 33. Snapshot workflow

Theo UI/version được triển khai:

```text
Select VM
 ↓
Create snapshot/recovery point
 ↓
Name with ticket/purpose
 ↓
Confirm
 ↓
Observe task
 ↓
Verify object exists
```

Tên ví dụ:

```text
D7-RHEL01_PRE_TEST_CHG-LAB-D7-001
```

Không dùng:

```text
snapshot1
test
temp
```

vì audit quality rất thấp.

---

# 34. Modify data sau snapshot

```bash
echo "Day7 post snapshot $(date -Is)" | sudo tee /opt/day7-lab/after_snapshot.txt
```

Không xóa dữ liệu quan trọng.

Lab chỉ cần chứng minh point-in-time distinction.

---

# 35. Snapshot evidence

Record:

```text
VM
snapshot/recovery-point ID
timestamp
purpose
ticket
Task reference
```

Sau đó ghi rõ:

```text
This snapshot is not accepted as independent HYCU backup evidence.
```

Đây là điểm examiner rất dễ hỏi.

---

# LAB 7 — Prism Tasks, Alerts, Events & Audit Collection

## 36. Objective

Xây một timeline từ các operation bạn vừa thực hiện.

Bạn cần correlate:

```text
Create VM
Power On
Shutdown
Power On
Snapshot
```

với Tasks/Audit.

---

# 37. Task evidence

Với mỗi operation ghi:

| Time  | Action    | Object    | Result  | Task Ref |
| ----- | --------- | --------- | ------- | -------- |
| 14:00 | Create VM | D7-RHEL01 | Success | `<id>`   |
| 14:10 | Power On  | D7-RHEL01 | Success | `<id>`   |
| 14:30 | Shutdown  | D7-RHEL01 | Success | `<id>`   |
| 14:35 | Power On  | D7-RHEL01 | Success | `<id>`   |
| 14:45 | Snapshot  | D7-RHEL01 | Success | `<id>`   |

---

# 38. Audit correlation

Bạn muốn chứng minh:

```text
Who
performed
What
on
Which object
When
```

Nếu audit log cung cấp fields tương ứng, record chúng.

Không đưa password/session token vào transcript.

---

# 39. Alert evidence

Nếu cluster có benign instructor-generated alert:

```text
Alert ID
severity
affected object
creation time
status
description
```

Sau đó phân biệt:

```text
Acknowledged
```

với:

```text
Resolved
```

Nếu instructor cho phép acknowledge:

```text
Acknowledgement evidence
```

không được ghi:

```text
Issue fixed.
```

trừ khi root cause thật sự đã được giải quyết và alert resolution được verify.

---

# LAB 8 — Cluster Health & NCC Evidence

## 40. Objective

Tạo health package có thể dùng trước maintenance.

Workflow:

```text
Alerts
 ↓
Hosts
 ↓
CVM
 ↓
Services
 ↓
Capacity
 ↓
Data resiliency
 ↓
NCC
 ↓
Go/No-Go
```

---

# 41. NCC run

Trong authorized CVM session:

```bash
ncc health_checks run_all
```

Nutanix KB hiện tại chỉ ra command này tạo comprehensive NCC output và cho phép export/download kết quả để support analysis. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details/?targetId=kA032000000986SCAQ&utm_source=chatgpt.com)

Expected không nhất thiết mọi check đều PASS.

Điều cần đánh giá:

```text
Are non-PASS findings understood?
Are any maintenance blockers?
```

---

# 42. NCC finding handling

Nếu có:

```text
FAIL: <check>
```

workflow:

```text
Identify check
 ↓
Read generated recommendation / referenced KB
 ↓
Determine affected layer
 ↓
Determine operational risk
 ↓
Correct only if authorized/runbook-covered
 ↓
Rerun
```

Nếu không biết:

```text
STOP + ESCALATE
```

Nutanix hiện khuyến nghị resolve critical alerts/NCC issues hoặc liên hệ Support nếu không thể resolve trước disruptive operations. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

---

# LAB 9 — Failure Injection: VM Availability

Đây là safe fault scenario.

## 43. Scenario

User reports:

> `D7-RHEL01` unavailable.

Instructor có thể seed một lỗi như:

```text
VM powered off
```

hoặc:

```text
NIC disconnected/wrong lab network
```

hoặc:

```text
guest IP misconfigured
```

Không nói learner root cause trước.

---

# 44. Điều tra đúng thứ tự

```text
1. Confirm incident time.
2. Verify VM exists.
3. Check VM power state.
4. Review recent Task/Audit.
5. Open console.
6. Verify guest boot.
7. Verify NIC attachment.
8. Verify guest IP.
9. Verify route.
10. Test gateway.
11. Check cluster/host health.
12. Correct only confirmed cause.
13. Retest.
```

Đây là một trong số ít chỗ mình dùng numbered troubleshooting list vì thứ tự rất quan trọng.

---

# 45. Ví dụ root cause — NIC attached sai network

Symptom:

```text
VM = Running
Console = accessible
Guest = healthy
No gateway connectivity
```

Prism inspection:

```text
Expected network:
LAB-SERVER

Actual:
LAB-ISOLATED-RESTORE
```

Root cause:

```text
Incorrect vNIC network attachment.
```

Corrective action:

```text
Attach NIC to approved LAB-SERVER network
```

chỉ sau khi authorization của fault scenario cho phép.

Postcheck:

```bash
ip addr
ip route
ping -c 4 <gateway>
```

Record:

```text
Before config
Root cause
Correction
After config
Connectivity result
Task reference
```

---

# LAB 10 — Failure Injection: Capacity Problem

## 46. Scenario

Không cần thực sự làm đầy storage.

Instructor có thể cung cấp:

```text
simulated storage alert
capacity screenshot
trend export
fault packet
```

Syllabus cho phép simulated capacity problem.

---

# 47. Learner phải phân tích gì?

Không chỉ đọc:

```text
Storage 85%
```

Mà cần tìm:

```text
Current used
Current free
Growth trend
Alert threshold
Top consumers
Snapshot/recovery-point contribution
Recent workload provisioning
Expected maintenance headroom
```

Nếu có 15% free nhưng growth rất nhanh:

```text
capacity risk may be urgent.
```

Nếu có 15% free nhưng stable và policy threshold là cảnh báo sớm:

```text
different urgency.
```

---

# 48. Không cleanup bừa

Tuyệt đối không:

```text
delete snapshot
delete VM
delete recovery points
```

chỉ vì storage alert.

Phải xác minh owner và retention/protection dependency.

Capacity incident có thể cần escalate thay vì cleanup.

---

# LAB 11 — Maintenance Plan Preparation

Syllabus yêu cầu **prepare an approved maintenance plan**. Infrastructure_Operations_14_Da…

---

# 49. Maintenance ticket structure

Một maintenance plan nên có:

```text
Change ID
Environment
Cluster
Target host
Business reason
Affected VMs
Potential impact
Owner
Approver
Maintenance window
Prerequisites
Prechecks
Implementation steps
Stop conditions
Backout/recovery strategy
Postchecks
Escalation contacts
Evidence IDs
```

---

# 50. Ví dụ maintenance purpose

```text
Purpose:
Rehearse AHV host maintenance workflow.

Environment:
LAB only.

Target:
AHV-01.

Actual hardware/firmware change:
None.

Expected impact:
VMs should remain available through approved
placement/migration workflow where supported.
```

---

# LAB 12 — AHV Host Maintenance Rehearsal

Đây là một **Additive Assignment bắt buộc** của syllabus. Có thể thực hiện dưới instructor control hoặc documented simulation. Infrastructure_Operations_14_Da…

---

# 51. Risk level

Đây là:

> **Potentially disruptive infrastructure operation.**

Không được tự thực hiện trên production.

Lab cần mentor approval.

---

# 52. Precheck Gate 1 — Change authorization

Check:

```text
Ticket approved
Window active
Target host confirmed
Operator authorized
Supervisor present
```

Nếu thiếu:

```text
NO-GO
```

---

# 53. Gate 2 — Cluster critical alerts

Review:

```text
Prism Alerts
```

hoặc authorized read-only CLI:

```bash
ncli alert ls
```

Nutanix AHV 11.2 hiện yêu cầu xử lý critical alerts trước restart/maintenance workflow. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

---

# 54. Gate 3 — Single-node tolerance

Đây là cực kỳ quan trọng.

Nutanix hiện ghi rõ trước việc restart CVM/AHV host hoặc đưa AHV host vào maintenance mode phải kiểm tra cluster có khả năng chịu **single-node failure** hay không. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

Điều đó có nghĩa:

```text
Cluster currently healthy
```

chưa đủ.

Phải đánh giá:

```text
Can cluster safely operate without target node?
```

---

# 55. Gate 4 — Workload placement

Inventory các VM trên target host:

```text
VM-A
VM-B
VM-C
```

Với từng VM:

```text
Can migrate?
Any special device?
Any dependency?
Critical workload?
Enough resources elsewhere?
```

Không giả định tất cả VM đều movable.

---

# 56. Gate 5 — Remaining cluster capacity

Ví dụ:

```text
AHV-02 memory utilization 45%
AHV-03 memory utilization 52%
```

có vẻ có headroom.

Nhưng:

```text
AHV-02 92%
AHV-03 94%
```

là cảnh báo rõ.

Không làm host maintenance chỉ vì button cho phép.

---

# 57. Gate 6 — NCC

```bash
ncc health_checks run_all
```

Nếu có critical unresolved failure:

```text
NO-GO
```

Nutanix cũng sử dụng NCC full check trong node-shutdown prechecks. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Network-Replacement-Cisco-HCI%3Abre-node-shutdown-precheck-t.html&utm_source=chatgpt.com)

---

# 58. Gate 7 — Recovery readiness

Xác minh:

```text
Critical workloads protected?
Latest backup acceptable?
Known backup failures?
Recovery capability available?
```

Lưu ý:

> Backup không biến host maintenance thành risk-free.

Nó chỉ là một control bổ sung.

---

# 59. Maintenance execution

Ở đây phải theo **exact runbook của deployed Nutanix/AHV version**.

Không nên lấy generic internet CLI rồi chạy.

Sequence conceptual:

```text
Initiate approved maintenance workflow
            ↓
Observe workload placement/migration
            ↓
Verify host reaches expected maintenance state
            ↓
Perform simulated maintenance activity
            ↓
Exit maintenance according to runbook
            ↓
Observe host rejoin/normal state
```

Nếu lab không cho actual maintenance:

```text
documented simulation
```

vẫn hợp syllabus, nhưng phải đánh dấu evidence:

```text
S = Simulation
```

không được ghi `Hands-on actual product maintenance`.

---

# 60. Stop conditions

Đây là phần cực quan trọng.

Dừng ngay nếu:

| Condition                            | Decision        |
| ------------------------------------ | --------------- |
| Critical cluster alert mới xuất hiện | STOP            |
| Data resiliency degraded             | STOP            |
| Second host issue                    | STOP            |
| VM migration fails unexpectedly      | STOP            |
| Insufficient capacity                | STOP            |
| NCC blocker                          | STOP            |
| Unexpected CVM/service issue         | STOP            |
| Maintenance window hết               | STOP            |
| Unsupported state                    | STOP + ESCALATE |
| Possible data integrity risk         | STOP + ESCALATE |

“Cố hoàn thành change” không phải success.

---

# 61. Post-maintenance validation

Phải check lại **cùng loại evidence** với baseline.

```text
Target host normal?
All expected hosts healthy?
CVM healthy?
Cluster services healthy?
Critical alerts?
Data resiliency?
Capacity?
VM placement?
Guest VMs reachable?
Application test?
NCC?
```

Tư duy:

```text
Precheck A
     ↕
Postcheck A
```

Không dùng precheck rất chi tiết nhưng postcheck chỉ:

```text
host green.
```

---

# LAB 13 — LCM Inspection / Upgrade Readiness

Day 7 yêu cầu LCM, compatibility/precheck và firmware/AOS/AHV dependency.

Syllabus không bắt buộc learner phải thực hiện thật một production-style upgrade.

Vì đây là disruptive/high-risk activity, cách lab an toàn hơn là **readiness assessment hoặc supervised fixture/demo**.

---

# 62. Objective

Không bấm `Upgrade`.

Mục tiêu là xác định:

```text
Current stack
Available target
Compatibility
Dependencies
Precheck
Maintenance requirement
Rollback limitation
```

---

# 63. Inventory capture

Record:

```text
AOS version
AHV version
NCC version
LCM version
BIOS/BMC/firmware components where exposed
```

---

# 64. LCM inventory/precheck

Trong LCM:

```text
Run/inspect inventory
Inspect applicable updates
Inspect compatibility/prechecks
```

không execute firmware/AOS/AHV update trừ khi instructor-controlled environment được chuẩn bị riêng.

---

# 65. Upgrade decision

Report phải có:

```text
Ready
Not Ready
Blocked
Requires Vendor Review
```

và reason.

Ví dụ:

```text
NO-GO:
NCC critical finding unresolved.
```

hoặc:

```text
NO-GO:
Target AHV/AOS combination not yet confirmed
against deployed compatibility matrix.
```

---

# 66. Rollback limitation exercise

Instructor hỏi:

> Nếu upgrade AOS/firmware fail thì restore VM snapshot có rollback platform được không?

Đáp án operational:

```text
No.
```

VM snapshot nằm ở workload/data layer.

Firmware/AOS/AHV lifecycle nằm ở platform layer.

Phải dùng vendor-supported recovery/backout strategy.

Syllabus Day 6–7 đặc biệt yêu cầu không giả định snapshot hoặc firmware downgrade luôn bảo đảm rollback. Infrastructure_Operations_14_Da…

---

# LAB 14 — HYCU Environment Baseline

Bây giờ chuyển từ Nutanix operational side sang independent data protection.

---

# 67. Objective

Trước khi chạy backup, xác định:

```text
Backup Controller
Source
Target
Policy
Protected object
RPO/RTO
Retention
Last job
Recovery points
```

HYCU documentation hiện tại vẫn tổ chức environment theo sources, targets và policies; User Guide 5.1/5.2 có riêng các phần thêm Nutanix source, target setup, backup strategy, VM protection và recovery. [download.hycu.com](https://download.hycu.com/ec/v5.1.0-2980/help/en/HYCU_UserGuide.pdf?utm_source=chatgpt.com)

---

# 68. Backup Controller verification

Ghi:

```text
Controller identity
HYCU version
Management address
Time/timezone
Health state
Role used
```

Không capture password.

HYCU onboarding hiện tại vẫn gọi component này là **HYCU Backup Controller**. [HYCU](https://www.hycu.com/customer-onboarding?utm_source=chatgpt.com)

---

# 69. Source inspection

Source:

```text
LAB-NTNX-CL01
```

Verify:

```text
Source registered?
Reachable?
Correct cluster?
Correct credentials configured?
Inventory discovered?
D7-RHEL01 visible?
```

Discovery:

```text
VM visible
```

không có nghĩa:

```text
VM protected
```

---

# 70. Target inspection

Ghi:

```text
Target name
Target type
Reachability
Capacity
Supported status
Encryption/immutability controls if applicable
```

HYCU 5.1 documentation liệt kê nhiều target types như NFS, SMB, Nutanix, Nutanix Objects, iSCSI, Azure, S3/S3-compatible và Data Domain; target nào được phép trong lab phải dựa đúng compatibility của installed version. [download.hycu.com](https://download.hycu.com/ec/v5.1.0-2980/help/en/HYCU_UserGuide.pdf?utm_source=chatgpt.com)

Không tự chọn một target chỉ vì HYCU UI có thể nhập URL vào được.

---

# LAB 15 — HYCU Protection Policy

## 71. Objective

Gán một protection policy cho:

```text
D7-RHEL01
```

Policy phải có documented:

```text
RPO
RTO
schedule/window
retention
target
application-aware requirements
```

HYCU User Guide hiện hành vẫn có custom-policy workflow và policy options liên quan backup strategy. [download.hycu.com](https://download.hycu.com/ec/v3.0.0/help/en/HYCU_UserGuide.pdf?utm_source=chatgpt.com)

---

# 72. Policy worksheet

Ví dụ lab:

| Field                 | Example                    |
| --------------------- | -------------------------- |
| Policy                | `D7-LAB-POLICY`            |
| Workload              | `D7-RHEL01`                |
| Target                | `<LAB-HYCU-TARGET>`        |
| RPO                   | 4h                         |
| RTO                   | 1h                         |
| Retention             | 7d                         |
| Backup Window         | Lab-approved               |
| Application awareness | Based on selected workload |

Đây chỉ là sample; không phải enterprise recommendation.

---

# 73. Application-aware decision

Nếu workload chỉ là simple file server lab:

```text
application-aware features may not be required.
```

Nếu instructor cung cấp SQL/database workload:

```text
application-aware protection should be inspected/tested.
```

Syllabus bắt buộc **validate application consistency for selected workload**. Infrastructure_Operations_14_Da…

---

# LAB 16 — Prepare Source Data & Integrity Manifest

Đây là một lab cực kỳ quan trọng vì sau này restore phải có dữ liệu để so sánh.

---

# 74. Tạo controlled dataset

Trong Linux VM:

```bash
sudo mkdir -p /opt/day7-recovery-test
```

```bash
echo "alpha" | sudo tee /opt/day7-recovery-test/file1.txt
```

```bash
echo "beta" | sudo tee /opt/day7-recovery-test/file2.txt
```

```bash
date -Is | sudo tee /opt/day7-recovery-test/created_at.txt
```

---

# 75. Source manifest

```bash
find /opt/day7-recovery-test -type f -print0 \
  | sort -z \
  | xargs -0 sha256sum
```

Ví dụ output:

```text
<hash1> /opt/day7-recovery-test/created_at.txt
<hash2> /opt/day7-recovery-test/file1.txt
<hash3> /opt/day7-recovery-test/file2.txt
```

Lưu:

```text
source_manifest.sha256
```

---

# 76. Count

```bash
find /opt/day7-recovery-test -type f | wc -l
```

Ví dụ:

```text
3
```

Bây giờ bạn có hai controls:

```text
File count
+
SHA-256
```

Sau restore có thể chứng minh dữ liệu giống source.

---

# LAB 17 — Execute & Monitor HYCU Backup

## 77. Objective

Chạy một controlled backup và chứng minh:

```text
policy applied
job ran
job succeeded
recovery point exists
```

---

# 78. Pre-backup check

Xác minh:

```text
Source reachable
Target reachable
VM exists
Policy attached
No active conflicting lab operation
Enough target capacity
Source data manifest saved
```

---

# 79. Start backup

Dùng approved HYCU workflow của installed version.

Record **before clicking Run**:

```text
Start timestamp
Source
Policy
Target
Expected RPO
Ticket
```

---

# 80. Monitor job

Không đóng UI sau khi bấm.

Theo dõi:

```text
Job ID
Job type
Source
State
Start time
Duration
Data transferred where exposed
Warnings
Errors
Final state
```

---

# 81. Backup success verification

Giả sử:

```text
Job = Success
```

chưa đủ.

Check:

```text
Recovery point appears?
Correct timestamp?
Correct VM?
Correct target?
Policy state compliant?
```

Mới có thể ghi:

```text
Backup execution verified.
```

Chưa được ghi:

```text
Recovery verified.
```

---

# LAB 18 — Controlled HYCU Failure Injection

Syllabus yêu cầu inject một backup failure. Additive section đưa hai ví dụ:

```text
Unavailable target
Insufficient capacity
```

Infrastructure_Operations_14_Da…

---

# 82. Safety requirement

Không phá target production.

Cách hợp lệ:

```text
Instructor-provided fault fixture
isolated target
test-only credential
controlled target disable
prepared insufficient-capacity condition
```

Không:

```text
delete arbitrary backup data
fill production filesystem
block enterprise firewall
revoke shared service credentials
```

---

# 83. Failure scenario A — Target unavailable

Baseline:

```text
Job success.
```

Inject fault:

```text
Lab target unavailable
```

Run controlled backup.

Expected:

```text
Job fails or reports target connectivity failure.
```

Capture:

```text
Job ID
start/end
failure stage
error text/code
target
source
```

---

# 84. Diagnosis

Không retry ngay.

Check:

```text
Source still healthy?
HYCU Controller healthy?
Policy correct?
Target registered?
Target reachable?
Target service available?
Authentication?
```

Nếu:

```text
Source healthy
Controller healthy
Target unavailable
```

root-cause confidence tăng mạnh.

---

# 85. Correction

Instructor khôi phục lab target.

Bạn verify target state trước.

Sau đó:

```text
Rerun
```

Record job mới.

Không overwrite job cũ.

Evidence phải cho thấy:

```text
Failed Job ID
        ↓
Root cause
        ↓
Correction
        ↓
New Job ID
        ↓
Success
```

Đây mới là troubleshooting evidence.

---

# 86. Failure scenario B — Insufficient capacity

Nếu instructor dùng fixture:

```text
Target capacity insufficient
```

Learner phải xác định:

```text
Target
required capacity
available capacity
job error
```

Không khắc phục bằng xóa random backups.

Corrective options phải theo policy:

```text
approved cleanup
expand target
change test target
retention correction
```

Chỉ instructor/platform owner quyết định nếu có data deletion.

---

# LAB 19 — Failed Job Log Analysis

## 87. Objective

Từ một failed job, tạo incident record đủ để người khác reproduce reasoning.

Template:

```text
Job ID:
Source:
Target:
Policy:
Started:
Failed:
Stage:
Error:
Expected:
Actual:
Hypothesis:
Checks performed:
Root cause:
Correction:
Retest Job:
Retest result:
```

---

# 88. Error message không phải root cause

Ví dụ log:

```text
Cannot write backup data.
```

Đây chỉ là symptom.

Root cause có thể:

```text
Target full
Network unreachable
Permission denied
Object-store policy
Target service failure
```

Report phải phân biệt:

```text
Observed error
vs
Root cause
```

---

# LAB 20 — Recovery Point Selection

Trước restore, chọn đúng recovery point.

---

# 89. Không mặc định chọn newest

Instructor có thể seed:

```text
RP-1 10:00 clean
RP-2 14:00 clean
RP-3 18:00 modified/bad test data
```

Incident:

```text
Data corruption occurred 17:15.
```

Newest recovery point `18:00` có thể không phù hợp.

Bạn phải chọn:

```text
14:00
```

nếu evidence cho thấy đó là latest known-good state.

---

# 90. Record recovery-point decision

```text
Selected RP:
<id>

Timestamp:
14:00

Reason:
Last known-good point before controlled corruption.

Rejected newer RP:
18:00

Reason:
Captured after corruption.
```

Đây là DR reasoning quan trọng.

---

# LAB 21 — Isolated Full VM Restore

Đây là phần trọng tâm nhất Day 7.

Syllabus yêu cầu isolated restore để tránh IP/identity conflicts. Infrastructure_Operations_14_Da…

HYCU 5.2 documentation hiện vẫn hỗ trợ chọn restore point và `Clone VM` tới destination, gồm chọn destination source và storage container. [download.hycu.com](https://download.hycu.com/ec/v5.2.0/help/en/HYCU_UserGuide.pdf?utm_source=chatgpt.com)

---

# 91. Không overwrite original trong recovery assurance lab

Mục tiêu Day 7 là chứng minh restore.

Safe pattern:

```text
Original:
D7-RHEL01

Restored:
D7-RHEL01-RESTORE
```

Đừng restore-overwrite original trừ khi lab chuyên biệt yêu cầu.

---

# 92. Restore-start timestamp

Ngay trước restore:

```text
T0 = YYYY-MM-DDThh:mm:ss+07:00
```

Đây là điểm bắt đầu đo **observed recovery time** theo lab contract.

---

# 93. Restore configuration

Select:

```text
Recovery point
Clone/restore operation
Destination cluster
Storage container
New VM name
Network
```

Network phải là:

```text
LAB-ISOLATED-RESTORE
```

hoặc VM được restore power-off rồi attach isolated network trước power-on.

---

# 94. Vì sao isolated?

Nếu restore cùng:

```text
hostname
IP
MAC-related config
application identity
```

vào live network, bạn có thể gây:

```text
Duplicate IP
Duplicate hostname
DNS conflict
Domain conflict
Application split-brain
Unexpected outbound traffic
```

Do đó isolation không phải nice-to-have.

Nó là safety control.

---

# 95. Power-on restored VM

Sau restore:

```text
Verify virtual hardware
Verify isolated NIC
THEN
Power On
```

Không power on trước khi network verification.

---

# 96. Guest validation

Console:

```text
Guest boots
Filesystem mounts
Expected hostname
Expected disk
```

Linux:

```bash
hostnamectl
```

```bash
lsblk
```

```bash
df -h
```

```bash
systemctl --failed
```

---

# LAB 22 — Restored Data Integrity Validation

## 97. File count

```bash
find /opt/day7-recovery-test -type f | wc -l
```

Compare:

```text
Source expected = 3
Restore actual  = 3
```

---

# 98. Hash validation

Trong restored VM:

```bash
find /opt/day7-recovery-test -type f -print0 \
  | sort -z \
  | xargs -0 sha256sum
```

Compare với pre-backup manifest.

Expected:

```text
source hash == restored hash
```

Nếu không:

```text
Restore cannot be marked fully validated.
```

---

# 99. Application-readable test

Checksum alone chứng minh bytes của selected files.

Nó không chứng minh application usable.

Nếu lab có application:

```text
start app
query data
perform read-only transaction
verify expected record
```

Nếu chỉ có Linux file workload:

```bash
cat /opt/day7-recovery-test/file1.txt
```

Expected:

```text
alpha
```

---

# 100. Recovery end timestamp

Chỉ khi:

```text
VM booted
+
guest healthy
+
required data validated
```

mới ghi:

```text
T1 = recovery validation complete
```

---

# 101. Observed recovery time

Công thức:

```text
Observed Recovery Time = T1 - T0
```

Ví dụ:

```text
Restore start      = 16:00:00
Validation complete = 16:17:42

Observed Recovery Time = 17m 42s
```

Không làm tròn thành:

```text
about 15 min
```

nếu lab yêu cầu audit-quality evidence.

---

# 102. So sánh với RTO

Nếu policy/business target:

```text
RTO = 30 min
```

và:

```text
Observed = 17m42s
```

lab result:

```text
Observed recovery completed within the stated lab RTO.
```

Không được viết:

```text
HYCU guarantees 17m42s RTO.
```

Một lần lab không phải production guarantee.

---

# LAB 23 — File-Level Restore

Full VM restore không phải lúc nào cũng hợp lý.

HYCU documentation hiện tại cũng mô tả individual-file recovery từ VM filesystem, cho phép chỉ phục hồi file cần thiết thay vì toàn bộ VM. [download.hycu.com](https://download.hycu.com/ec/v3.0.0/help/en/HYCU_UserGuide.pdf?utm_source=chatgpt.com)

---

# 103. Scenario

User vô tình xóa:

```text
/opt/day7-recovery-test/file2.txt
```

Trong original lab VM:

```bash
sudo rm /opt/day7-recovery-test/file2.txt
```

Chỉ làm với controlled lab file.

---

# 104. Verify loss

```bash
ls -l /opt/day7-recovery-test/
```

Expected:

```text
file2.txt missing
```

---

# 105. Restore minimum scope

Thay vì restore entire VM:

```text
Select known-good recovery point
        ↓
Choose file-level recovery
        ↓
Select file2.txt
        ↓
Restore to approved destination
```

Safe variant:

```text
restore to alternate lab directory
```

trước:

```text
/tmp/day7-restore-check/
```

để tránh overwrite existing data.

---

# 106. Validate file

```bash
sha256sum /tmp/day7-restore-check/file2.txt
```

Compare original manifest.

Expected:

```text
hash matches.
```

Sau approval mới có thể đưa về original path nếu lab yêu cầu.

---

# LAB 24 — RPO Measurement

Day 7 yêu cầu hiểu và thực hành RPO/RTO.

---

# 107. Observed RPO concept

Giả sử incident:

```text
17:00
```

Latest usable recovery point:

```text
15:30
```

Observed recoverable data age:

```text
17:00 - 15:30 = 1h30m
```

Nếu RPO objective:

```text
≤ 4h
```

lab đáp ứng.

---

# 108. Failed backup impact

Ví dụ:

```text
12:00 success
16:00 failed
18:00 incident
```

Latest usable RP:

```text
12:00
```

Observed gap:

```text
6h
```

Nếu target RPO:

```text
4h
```

thì:

```text
RPO missed.
```

Đây là lý do “schedule every 4h” không đồng nghĩa với “RPO luôn 4h”.

---

# LAB 25 — DR Mini-Rehearsal

Bây giờ kết hợp các lab thành một incident.

---

# 109. Scenario

Instructor đưa incident:

> `D7-RHEL01` bị corrupted sau một change. Current VM vẫn boot nhưng application/test data không còn đáng tin cậy. HYCU có nhiều recovery points, trong đó recovery point mới nhất được tạo sau corruption.

Learner phải quyết định:

```text
Do not blindly repair
Do not blindly restore newest
```

Mà phải:

```text
Establish corruption time
 ↓
Identify last-known-good RP
 ↓
Confirm recovery approval
 ↓
Restore isolated
 ↓
Validate data
 ↓
Measure recovery
 ↓
Prepare promotion recommendation
```

---

# 110. DR decision report

Report cuối nên ghi:

```text
Incident time
Corruption window
Selected recovery point
Reason
Restore destination
Isolation control
Data validation
Application validation
Observed recovery time
Expected RPO
Expected RTO
Outstanding risks
Recommendation
```

---

# LAB 26 — Failure Injection Catalogue

Để đảm bảo learner thực sự troubleshoot chứ không chỉ follow happy path, Day 7 nên có 5 controlled faults.

| Failure                           | Expected reasoning                           |
| --------------------------------- | -------------------------------------------- |
| VM accidentally powered off       | Power state → Task/Audit → restore           |
| Wrong VM network                  | VM healthy → NIC/network → connectivity      |
| Capacity warning                  | utilization → trend → consumers → escalation |
| HYCU target unavailable           | source/controller/policy → target            |
| HYCU insufficient target capacity | capacity → retention/expansion decision      |

Có thể thêm:

```text
wrong recovery point selection
```

như một examiner scenario, nhưng không cần gây data loss thực tế.

---

# 111. Một lỗi không nên inject bằng cách nào?

Không:

```text
kill CVMs randomly
stop AOS services
corrupt storage
remove physical disk without runbook
break production networking
run unapproved firmware downgrade
delete real backup chains
disable security controls
```

Day 7 dạy troubleshooting, không dạy phá platform.

---

# 112. Troubleshooting Checkpoint — VM unavailable

Mental flow:

```text
VM visible?
  │
  ├─ No → inventory/management issue
  │
  └─ Yes
      ↓
Power state?
      │
      ├─ Off → Task/Audit/reason
      │
      └─ On
          ↓
Console healthy?
          │
          ├─ No → guest/boot/storage
          │
          └─ Yes
              ↓
NIC correct?
              ↓
IP/route?
              ↓
Network path?
              ↓
Application?
```

---

# 113. Troubleshooting Checkpoint — multiple VMs affected

Nếu:

```text
VM-A
VM-B
VM-C
```

đều lỗi cùng lúc, đừng troubleshoot ba guest riêng trước.

Hỏi:

```text
Same host?
Same VLAN?
Same storage container?
Same cluster?
Same upstream dependency?
```

Shared failure domain giúp thu hẹp root cause.

---

# 114. Troubleshooting Checkpoint — HYCU failure

Mental flow:

```text
JOB FAILED
    │
    ▼
Source available?
    │
    ▼
Controller healthy?
    │
    ▼
Policy valid?
    │
    ▼
Application preparation?
    │
    ▼
Network/connectivity?
    │
    ▼
Authentication?
    │
    ▼
Target reachable?
    │
    ▼
Capacity?
    │
    ▼
Target-side/service condition?
```

Không bỏ qua layer.

---

# 115. Troubleshooting Checkpoint — restore success nhưng application fail

Đây là scenario rất quan trọng.

```text
Restore Job = SUCCESS
VM = ON
Application = DOWN
```

Không được conclude:

```text
HYCU restore is good; application team's problem.
```

Trước tiên cần validate:

```text
Correct recovery point?
All expected disks?
Filesystem healthy?
Application service?
Database consistency?
Required dependency?
Network isolation effect?
Credentials?
DNS?
```

Sau đó xác định support boundary.

---

# 116. Escalation boundary — Nutanix

L1/L2 phải escalate khi gặp:

```text
Data resiliency concern
Multiple-node failure
Unknown CVM/service failure
Persistent critical NCC failure
Potential storage corruption
Unsupported configuration
LCM/upgrade failure outside runbook
Hardware failure needing replacement
Cluster instability
Security incident
```

Evidence package nên có:

```text
Cluster
AOS/AHV/NCC versions
Timestamp
Alert IDs
Task IDs
NCC output
Affected VMs
Recent changes
What was tested
What was NOT changed
```

---

# 117. Escalation boundary — HYCU

Escalate khi:

```text
Repeated job failure after known-safe checks
Suspected backup-chain corruption
Target compatibility uncertainty
Restore failure
Application consistency failure outside runbook
Backup Controller failure
Unsupported combination
Credential/integration problem requiring higher privilege
Potential security compromise
```

Package:

```text
HYCU version
Job ID
Source
Target
Policy
Recovery point
Timestamp
Error
Relevant logs
Previous successful job
Correction attempted
Retest result
```

---

# 118. Cleanup & Rollback

Một lab không kết thúc khi screenshot xong.

Phải cleanup controlled resources.

---

# 119. VM cleanup

Nếu VM chỉ dùng lab:

```text
Power off gracefully
```

Sau đó delete chỉ khi:

```text
Instructor approves
Evidence complete
No remaining restore dependency
No backup lab dependency
```

Không delete nếu HYCU lab sau vẫn cần source.

---

# 120. Snapshot cleanup

Snapshot/recovery-point được tạo cho operational test phải:

```text
retain according to lab requirement
```

hoặc:

```text
delete through approved cleanup
```

Không để orphaned snapshots chỉ vì “lab xong rồi”.

---

# 121. Restore cleanup

Restored VM:

```text
D7-RHEL01-RESTORE
```

phải được:

```text
powered off
verified no longer needed
removed through approved workflow
```

Restore network không được reconnect production-equivalent network vô tình.

---

# 122. HYCU cleanup

Không xóa backup chain chỉ để dọn lab nếu retention đang được dùng cho assessment.

Cleanup có thể gồm:

```text
remove temporary test policy assignment
remove temporary restore VM
restore target condition after failure injection
document remaining recovery points
```

---

# 123. Post-lab Nutanix health check

Lặp lại baseline:

```text
Alerts
Hosts
CVM
Cluster services
Capacity
VM state
NCC
```

Đây là cách chứng minh:

```text
Lab did not leave cluster degraded.
```

---

# 124. Pre/Post evidence comparison

Một bảng rất quan trọng:

| Check           | Before    | After            | Result |
| --------------- | --------- | ---------------- | ------ |
| Cluster health  | Healthy   | Healthy          | PASS   |
| Hosts           | 3 healthy | 3 healthy        | PASS   |
| CVMs            | Healthy   | Healthy          | PASS   |
| Critical alerts | 0         | 0                | PASS   |
| NCC blockers    | 0         | 0                | PASS   |
| Test VM         | N/A       | cleaned/expected | PASS   |
| Backup target   | Healthy   | Healthy          | PASS   |
| Restore VM      | N/A       | removed          | PASS   |

Nếu After khác Before:

```text
explain why.
```

Không che giấu discrepancy.

---

# 125. Evidence Package bắt buộc

Theo acceptance chính xác của syllabus, Day 7 phải có ít nhất:

| Category    | Evidence                  |
| ----------- | ------------------------- |
| Prism       | Task references           |
| Prism       | Audit references          |
| Health      | Pre-health                |
| Health      | Post-health               |
| Maintenance | Go/No-Go decision         |
| Maintenance | Stop/escalation decisions |
| HYCU        | Job ID                    |
| HYCU        | Source                    |
| HYCU        | Target                    |
| HYCU        | Protection policy         |
| Recovery    | Recovery point            |
| Failure     | Failure cause             |
| Failure     | Logs                      |
| Retest      | New successful job        |
| Restore     | Restore logs              |
| Validation  | File/content comparison   |
| Timing      | Measured recovery time    |
| RPO/RTO     | Observed assessment       |
| Cleanup     | Final-state verification  |

Đây không phải mình tự thêm acceptance criteria; syllabus Day 7 ghi trực tiếp Prism task/audit references, pre/post health, stop/escalation decisions, HYCU job ID, source/target, recovery point, failure cause, restore logs, content validation và measured recovery time là required evidence. Infrastructure_Operations_14_Da…

---

# 126. Evidence Manifest mẫu

```text
Evidence ID:
D7-E045

Source System:
HYCU

Version:
<version>

Environment:
LAB

Asset:
D7-RHEL01

Job ID:
<job-id>

Related Ticket:
CHG-LAB-D7-001

Collector:
<learner>

Collection Time:
2026-10-21T17:25:00+07:00

Operation:
Isolated VM restore validation

Recovery Point:
<rp-id>

Restore Target:
LAB-NTNX-CL01

Restore Network:
LAB-ISOLATED-RESTORE

Result:
PASS

File Count Expected:
3

File Count Actual:
3

Hash Validation:
PASS

Recovery Start:
17:02:10+07:00

Validation Complete:
17:18:43+07:00

Observed Recovery Time:
00:16:33

Limitations:
Lab workload only; no production application validation.
```

Đây là evidence có thể reviewer audit lại.

---

# 127. Final Maintenance Record mẫu

```text
CHANGE:
CHG-LAB-D7-001

TARGET:
AHV-01 / LAB-NTNX-CL01

PRECHECK:
PASS

CRITICAL ALERTS:
None

NCC:
No maintenance-blocking findings

CAPACITY:
Adequate for simulated host evacuation

WORKLOAD REVIEW:
Completed

GO/NO-GO:
GO

MAINTENANCE:
Instructor-controlled rehearsal

POSTCHECK:
PASS

NEW ALERTS:
None

VM VALIDATION:
PASS

ROLLBACK:
Not required

ESCALATION:
None
```

Nếu simulation:

```text
Execution Classification:
S — Simulation/Fixture
```

không được ghi:

```text
H — Actual hands-on maintenance
```

nếu không thực sự thao tác product.

Syllabus toàn chương trình yêu cầu phân biệt rõ Hands-on, Simulation, Demonstration và Theory. Infrastructure_Operations_14_Da…

---

# 128. Final HYCU Recovery Record mẫu

```text
SOURCE:
D7-RHEL01

POLICY:
D7-LAB-POLICY

TARGET:
<target>

BACKUP JOB:
<job-id-success>

FAILURE JOB:
<job-id-failed>

FAILURE:
Target unavailable

ROOT CAUSE:
Controlled target unavailability

CORRECTION:
Target restored by instructor

RETEST:
<job-id-retest>

RETEST RESULT:
PASS

RECOVERY POINT:
<rp-id>

RESTORE:
Clone to D7-RHEL01-RESTORE

NETWORK:
LAB-ISOLATED-RESTORE

FILE COUNT:
Expected 3
Actual 3

HASH:
PASS

APPLICATION/DATA READ:
PASS

OBSERVED RECOVERY:
16m33s

RPO:
Met / Missed / Not assessable

RTO:
Met / Missed / Not assessable

CLEANUP:
PASS
```

---

# 129. Acceptance Criteria của toàn Module Thực hành

Bạn chưa hoàn thành Day 7 nếu chỉ làm được:

```text
Create VM
+
Run backup
```

Bạn phải chứng minh chuỗi:

```text
VM provisioning
✓

CPU/memory/storage verified
✓

Correct virtual network/VLAN
✓

Guest connectivity verified
✓

Resource utilization inspected
✓

Controlled power operation
✓

Snapshot/recovery point operation
✓

Prism Tasks collected
✓

Alerts/audit inspected
✓

Cluster health baseline
✓

NCC evidence
✓

Capacity/VM availability fault investigated
✓

Maintenance plan prepared
✓

AHV host-maintenance rehearsal
✓

Pre/post maintenance checks
✓

HYCU source verified
✓

HYCU target verified
✓

Protection policy applied
✓

Backup executed
✓

Backup monitored
✓

Failure injected safely
✓

Logs investigated
✓

Root cause identified
✓

Approved correction applied
✓

Backup rerun successful
✓

Known recovery point selected
✓

VM/file restore performed
✓

Restore isolated
✓

Guest/application/data validation
✓

Source vs restored content compared
✓

Observed recovery time measured
✓

RPO/RTO assessed
✓

Evidence package complete
✓

Cleanup complete
✓

Escalation boundary understood
✓
```

---

# 130. Những lỗi thực hành thường gặp

| Sai lầm                                 | Vì sao sai                          |
| --------------------------------------- | ----------------------------------- |
| Tạo VM mà không check target cluster    | Có thể thao tác nhầm environment    |
| Chỉ check VM `Running`                  | Không chứng minh guest/app healthy  |
| Ping được là kết luận network tốt       | Chưa kiểm application path          |
| Thấy CPU spike là tăng vCPU             | Chưa tìm root cause                 |
| Force power off trước                   | Rủi ro data/application consistency |
| Snapshot xong gọi là backup             | Sai failure-domain concept          |
| Acknowledge alert rồi coi resolved      | Sai operational semantics           |
| NCC FAIL nhưng vẫn maintenance          | Có thể tăng blast radius            |
| Maintenance chỉ check host              | Không verify cluster/workload       |
| LCM thấy update là install              | Chưa assess compatibility/change    |
| HYCU thấy VM là coi protected           | Discovery ≠ protection              |
| Backup Success là coi DR hoàn tất       | Chưa restore                        |
| Retry failed backup liên tục            | Không tìm root cause                |
| Xóa old backup để chữa capacity         | Có thể phá retention                |
| Restore VM trực tiếp production network | Duplicate identity/IP risk          |
| Restore job Success là coi dữ liệu tốt  | Chưa validate content               |
| Chọn newest recovery point              | Có thể restore corruption           |
| Chỉ screenshot dashboard                | Evidence thiếu traceability         |
| Fix xong xóa failure evidence           | Mất audit trail                     |
| Không cleanup restore VM                | Tạo security/capacity risk          |

---

# 131. Troubleshooting philosophy phải hình thành sau Day 7

Khi có incident, bạn cần suy nghĩ theo pattern:

```text
SYMPTOM
   ↓
SCOPE
   ↓
EXPECTED STATE
   ↓
CURRENT STATE
   ↓
RECENT CHANGE
   ↓
SHARED DEPENDENCY
   ↓
EVIDENCE
   ↓
HYPOTHESIS
   ↓
LOW-RISK TEST
   ↓
ROOT CAUSE
   ↓
AUTHORIZED CORRECTION
   ↓
VERIFICATION
   ↓
EVIDENCE
```

Không phải:

```text
SYMPTOM
   ↓
REBOOT
```

---

# 132. Operational flow Nutanix hoàn chỉnh

Sau Module Thực hành, flow Nutanix bạn nên thuộc theo logic:

```text
Ticket / Incident
       │
       ▼
Verify Environment & Identity
       │
       ▼
Prism Health / Alerts / Capacity
       │
       ▼
VM / Host / CVM / AOS Scope
       │
       ▼
Tasks / Events / Audit
       │
       ▼
NCC if relevant
       │
       ▼
Determine failing layer
       │
       ├─────────► Guest/Application
       │
       ├─────────► VM
       │
       ├─────────► Network
       │
       ├─────────► AHV Host
       │
       ├─────────► Storage/AOS
       │
       └─────────► Hardware
       │
       ▼
Authorized Correction
       │
       ▼
Post-health
       │
       ▼
Evidence / Escalation
```

---

# 133. Operational flow HYCU hoàn chỉnh

```text
Protection Requirement
       │
       ▼
Source Discovery
       │
       ▼
Target Compatibility
       │
       ▼
Protection Policy
       │
       ▼
Application Consistency
       │
       ▼
Backup Job
       │
       ▼
Monitor
       │
       ├──── Failed
       │       │
       │       ▼
       │   Diagnose
       │       │
       │       ▼
       │   Correction
       │       │
       │       ▼
       │     Rerun
       │
       ▼
Recovery Point
       │
       ▼
Incident / Restore Test
       │
       ▼
Select Known-Good Point
       │
       ▼
Isolated Restore
       │
       ▼
Infrastructure Validation
       │
       ▼
Application Validation
       │
       ▼
Content/Hash Validation
       │
       ▼
Observed RPO/RTO
       │
       ▼
Evidence / Cleanup
```

HYCU hiện vẫn nhấn mạnh policy-driven backup, auto-discovery và granular point-in-time recovery cho Nutanix workloads, nhưng recovery capability phải luôn được xác minh theo exact HYCU version, target và workload được triển khai. [HYCU](https://www.hycu.com/resources/datasheet-hycu-nutanix?utm_source=chatgpt.com)

---

# 134. Day 7 — Practical Cheat Sheet

### Nutanix read-only/health

```bash
cluster status
```

Dùng kiểm tra cluster/service state theo approved runbook.

```bash
ncli alert ls
```

Inspect alerts; tài liệu AHV hiện tại sử dụng nó trong cluster-health verification. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

```bash
ncc health_checks run_all
```

Full NCC health check. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details/?targetId=kA032000000986SCAQ&utm_source=chatgpt.com)

Không tự suy rộng các command trên thành quyền:

```text
cluster stop
service restart
host reboot
```

Những operation đó có blast radius hoàn toàn khác.

### Linux guest

```bash
hostnamectl
ip addr
ip route
free -h
df -h
df -i
systemctl --failed
```

Network:

```bash
ping -c 4 <gateway>
getent hosts <hostname>
curl -I --connect-timeout 5 http://<test-endpoint>
```

Integrity:

```bash
sha256sum <file>
```

Dataset:

```bash
find /opt/day7-recovery-test -type f | wc -l
```

Manifest:

```bash
find /opt/day7-recovery-test -type f -print0 \
  | sort -z \
  | xargs -0 sha256sum
```

---

# 135. Kết luận Module Thực hành Day 7

Nếu hoàn thành đúng module này, bạn không chỉ biết “Nutanix và HYCU dùng như thế nào”.

Bạn đã thực hành một operational chain gần với cách enterprise infrastructure được vận hành:

```text
Provision
→ Configure
→ Verify
→ Monitor
→ Baseline
→ Troubleshoot
→ Maintain
→ Protect
→ Fail
→ Diagnose
→ Correct
→ Backup
→ Restore
→ Validate
→ Measure
→ Document
→ Cleanup
→ Escalate
```

Điểm quan trọng nhất Day 7 là ba distinction:

```text
Snapshot
≠
Independent Backup
```

```text
Backup Success
≠
Recovery Success
```

và:

```text
Recovery Success
≠
Business/Application Recovery
```

Bạn chỉ có **recovery assurance** khi một known-good recovery point được restore trong môi trường an toàn, workload boot được, application/data đọc được, nội dung recovered được kiểm chứng, recovery time được đo, evidence được lưu và environment được cleanup/post-check đúng quy trình. Đây chính là mức acceptance mà syllabus Day 7 hướng tới. Infrastructure_Operations_14_Da…
