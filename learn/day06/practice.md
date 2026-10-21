# Day 6 — Module Thực hành

## HPE ProLiant / iLO + Nutanix HCI / AOS / AHV / Prism

Module Thực hành Day 6 phải biến kiến thức lý thuyết thành một workflow vận hành thực tế: **inventory → health check → baseline → alert investigation → cross-layer correlation → impact/resiliency assessment → evidence → escalation → maintenance planning → post-check**.

Theo syllabus, bạn phải thực hành kiểm tra một HPE ProLiant server hoặc authorized iLO simulator; xác định hardware inventory, firmware, power, management connectivity và hardware health; thu thập hardware logs; sau đó truy cập Nutanix qua Prism Element/Prism Central để nhận diện nodes, CVMs, AHV hosts, storage containers, networking, health, alerts, events, versions, capacity và hardware dependencies. Cuối cùng phải điều tra một simulated node/hardware warning, xác định đúng failing layer/support owner và nộp combined evidence. Infrastructure_Operations_14_Da…

Điểm quan trọng: Day 6 **không phải lab “bấm thử tất cả các nút”**. Firmware update, physical replacement, power-cycle, CVM restart và AHV host maintenance là disruptive operations; syllabus yêu cầu chúng chủ yếu được demonstration/rehearsal, kèm resiliency/capacity checks, maintenance plan, stop conditions và escalation boundary. Infrastructure_Operations_14_Da…

---

# PHẦN 1 — MỤC TIÊU THỰC HÀNH

Sau Module Thực hành Day 6, bạn phải tự thực hiện được quy trình sau trong authorized lab:

```text
HPE/iLO
   │
   ├── Identify server
   ├── Inventory hardware
   ├── Check firmware
   ├── Check power
   ├── Check health
   └── Inspect IML
          │
          ▼
Nutanix
   │
   ├── Identify cluster
   ├── Map Node ↔ AHV ↔ CVM
   ├── Identify AOS/AHV versions
   ├── Inspect storage/network
   ├── Check health/services
   ├── Inspect alerts/events
   └── Check capacity/resilience
          │
          ▼
Cross-layer correlation
          │
          ▼
Failure layer
          │
          ▼
Impact / risk
          │
          ▼
Correct support owner
          │
          ▼
Evidence + maintenance plan
```

Bạn không chỉ cần chứng minh:

> “Tôi tìm thấy warning.”

Mà phải chứng minh:

> “Warning phát sinh ở physical hardware layer của node X, được ghi nhận trong iLO/IML lúc T1 và được Prism phản ánh ở cluster layer lúc T2. Workload hiện vẫn available nhưng redundancy/resilience bị giảm. Không có authorization cho physical replacement, do đó tôi dừng ở verification/evidence collection và escalate tới hardware/vendor owner.”

Đó mới là kết quả đúng tinh thần Infrastructure Operations.

---

# PHẦN 2 — LAB ENVIRONMENT / TOPOLOGY

Một topology lý tưởng:

```text
                         ┌──────────────────────┐
                         │ Admin Workstation    │
                         │ Browser + SSH        │
                         │ Evidence Repository  │
                         └─────────┬────────────┘
                                   │
                         Management Network
                    ┌──────────────┼──────────────┐
                    │              │              │
                    ▼              ▼              ▼
              ┌──────────┐   ┌──────────┐   ┌────────────┐
              │ iLO N1   │   │ iLO N2   │   │Prism      │
              │          │   │          │   │Central     │
              └────┬─────┘   └────┬─────┘   └─────┬──────┘
                   │              │               │
           ┌───────▼──────┐ ┌────▼─────────┐     │
           │ Nutanix Node1│ │Nutanix Node2 │     │
           │              │ │              │     │
           │ AHV          │ │ AHV          │     │
           │ CVM          │ │ CVM          │     │
           │ Storage      │ │ Storage      │     │
           └───────┬──────┘ └────┬─────────┘     │
                   │              │               │
                   └────────┬─────┘               │
                            │                     │
                       Nutanix Cluster ◄──────────┘
                            │
                      Prism Element
                            │
                      Guest VMs
```

Trong lab thật có thể có 3, 4 hoặc nhiều nodes hơn.

Cũng có thể bạn không có physical hardware. Syllabus cho phép instructor cung cấp:

- authorized simulator;
- screenshots;
- exports;
- configuration files;
- version-labelled fault packets.

Khi đó phải ghi rõ:

```text
Hands-on status: Simulation / Fixture
```

chứ không được khai là đã hands-on actual hardware.

---

# PHẦN 3 — PREREQUISITES

Trước lab phải có tối thiểu:

### Access

- approved iLO read-only/operator account;
- approved Prism Element account;
- Prism Central account nếu lab có;
- CVM SSH account chỉ nếu lab/runbook cho phép;
- không sử dụng production credential.

### Information

- lab asset list;
- node names;
- IP/address mapping;
- cluster name;
- expected node count;
- expected software versions nếu instructor cung cấp baseline;
- ticket/change/incident reference giả lập.

### Tools

- browser;
- SSH client;
- text editor;
- screenshot tool;
- hashing utility;
- evidence directory;
- timestamp/timezone awareness.

### Evidence directory

Ví dụ:

```text
Day06/
├── 01_HPE_iLO/
├── 02_HPE_IML/
├── 03_Nutanix_Inventory/
├── 04_Nutanix_Health/
├── 05_Alerts_Events/
├── 06_Correlation/
├── 07_Maintenance_Plan/
├── 08_Failure_Injection/
└── manifest.csv
```

---

# PHẦN 4 — SAFETY CLASSIFICATION

Trước khi thực hiện bất cứ operation nào, phân loại nó.

## Class A — Read-only / observation

Được ưu tiên trong Day 6:

```text
View inventory
View health
View firmware versions
View IML
View power status
View alerts
View events
View capacity
View host/CVM mapping
Run approved diagnostic health checks
```

Không thay đổi intended configuration.

---

## Class B — Diagnostic nhưng cần kiểm soát

Ví dụ:

```text
NCC health check
SSH vào CVM để chạy approved read-only commands
Remote Console observation
```

Không nhất thiết gây thay đổi configuration nhưng vẫn cần:

- authorization;
- privilege;
- resource awareness;
- logging.

---

## Class C — Configuration-changing

Ví dụ:

```text
Edit iLO settings
Edit BIOS/UEFI
Change Nutanix networking
Change storage
Modify cluster configuration
```

Không tự làm trong Day 6 nếu assignment không yêu cầu.

---

## Class D — Disruptive

Ví dụ:

```text
Power Off
Reset
Power Cycle
CVM restart
AHV host maintenance
Firmware update
Hardware replacement
Cluster stop
```

**Không thực hiện tự do trong Day 6.**

Nutanix hiện hành yêu cầu kiểm tra cluster health và khả năng chịu single-node failure trước những operation như restart CVM/AHV host hoặc đưa AHV host vào maintenance mode; critical alerts phải được giải quyết hoặc escalate trước khi tiếp tục. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

---

# PHẦN 5 — EVIDENCE CONTRACT TRƯỚC KHI LAB

Day 6 kế thừa evidence discipline từ Day 1.

Mỗi evidence item nên chứa:

```text
Evidence ID
Source product/system
Product/version
Environment
Asset / hostname / node
Collection timestamp
Timezone
Collector
Ticket/change reference
Command/filter/query
Filename
Record count nếu phù hợp
File size
SHA-256 nếu export thành file
Limitations/errors
```

Syllabus yêu cầu giữ untouched raw export riêng với working copies, kiểm tra hashes sau transfer và sử dụng counts/known events để đánh giá completeness. Infrastructure_Operations_14_Da…

Ví dụ naming:

```text
D6-HPE-001_NODE01_iLO_Health_20261021T103000+0700.png
D6-HPE-002_NODE01_IML_20261021T103500+0700.csv
D6-NTNX-001_CLUSTER01_HostInventory_20261021T110000+0700.txt
D6-NTNX-002_CLUSTER01_ClusterStatus_20261021T110500+0700.txt
```

Không dùng:

```text
screenshot1.png
new.txt
final-final2.txt
```

vì không audit-friendly.

---

# LAB 1 — XÁC NHẬN ĐÚNG HPE SERVER / iLO

## Objective

Trước hết phải chứng minh:

> Mình đang truy cập đúng asset trong đúng environment.

Đây là precondition trước mọi health investigation.

---

## Step 1 — Login iLO

Truy cập iLO URL/address được instructor cung cấp.

Không paste credential thật vào:

- notes;
- screenshots;
- evidence;
- chat;
- scripts.

---

## Step 2 — Xác nhận identity

Tìm các thông tin available trong environment như:

```text
Server name
Product/model
Serial number
Asset tag
iLO hostname
iLO firmware version
Server generation
Power state
```

Không phải mọi environment đều expose cùng field.

---

## Verification

So sánh với lab inventory:

```text
Expected asset: NODE01
Observed asset: NODE01

Expected model: <lab baseline>
Observed model: <observed>

Environment: LAB
```

Nếu mismatch:

```text
STOP
```

Không tiếp tục operation.

---

## Evidence

Chụp/capture màn hình chứa:

- identifiable asset;
- iLO/version;
- timestamp nếu platform/display hỗ trợ.

Mask các identifier nếu lab policy yêu cầu.

Evidence ID:

```text
D6-HPE-001
```

---

# LAB 2 — HPE HARDWARE INVENTORY

## Objective

Xây hardware inventory đủ để biết server thực tế gồm những gì.

---

## Step 1 — Processor inventory

Record:

```text
CPU model
CPU count
Core information
Health
```

Không cần screenshot mọi field nếu một structured export/table đã đủ.

---

## Step 2 — Memory inventory

Record:

```text
Total memory
DIMM population
DIMM health
Any degraded/failed module
```

Điều cần kiểm tra không phải chỉ là:

```text
1024 GB installed
```

mà còn:

```text
Are all expected DIMMs healthy?
```

---

## Step 3 — Storage

Record:

```text
Storage controller
Physical drives
Drive type
Drive capacity
Drive health
Logical drive/RAID information nếu applicable
```

Lưu ý: đây là inventory/health check, **không create/delete RAID configuration**.

---

## Step 4 — NIC

Record:

```text
NIC adapters
Ports
Health/state
Firmware if available
```

Không thay đổi interface configuration.

---

## Step 5 — PSU / cooling

Record:

```text
Number of PSUs
PSU health
Redundancy status if exposed
Fan health
Temperature/thermal state
```

---

# HPE INVENTORY TABLE

Bạn nên tạo table dạng:

| Component          | Identifier   | Version/Model      | State   | Notes            |
| ------------------ | ------------ | ------------------ | ------- | ---------------- |
| Server             | NODE01       | ProLiant `<model>` | Healthy | LAB              |
| CPU                | CPU1         | `<model>`          | OK      | —                |
| Memory             | DIMM set     | `<capacity>`       | OK      | —                |
| Storage controller | Controller 1 | `<model>`          | OK      | —                |
| Drive              | Bay X        | `<model/capacity>` | OK      | —                |
| PSU                | PSU1         | —                  | OK      | Redundancy check |
| PSU                | PSU2         | —                  | OK      | Redundancy check |

Không invent field nếu UI không cung cấp.

---

# LAB 3 — FIRMWARE BASELINE

## Objective

Xác định firmware state mà **không update firmware**.

---

## Record ít nhất

Nếu available:

```text
System ROM / UEFI
iLO firmware
Storage controller firmware
NIC firmware
Drive firmware
```

Không phải tất cả platform đều hiển thị mọi component ở cùng menu.

HPE iLO 7 User Guide hiện tại tháng 7/2026 tiếp tục cung cấp inventory, server health và firmware-management capabilities; nhưng iLO 5/6/7 có UI/capability differences, nên lab phải sử dụng documentation tương ứng với đúng iLO generation. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00005342en_us&docLocale=en_US&utm_source=chatgpt.com)

---

## Baseline table

```text
Component           Current version      Baseline     Status
----------------------------------------------------------------
iLO                  x.y                  x.y          MATCH
System ROM           x.y                  x.y          MATCH
Storage Controller   x.y                  x.z          REVIEW
NIC                   x.y                  unknown      UNKNOWN
```

`UNKNOWN` tốt hơn tự đoán.

---

# Quan trọng: không biến version mismatch thành automatic update

Nếu thấy:

```text
Current: 1.20
Latest: 1.24
```

không được kết luận:

```text
ACTION = upgrade
```

Bạn phải hỏi:

```text
Approved baseline?
Hardware compatibility?
Nutanix compatibility?
Maintenance window?
Reboot requirement?
Known issues?
Rollback limitations?
```

---

# LAB 4 — POWER VÀ MANAGEMENT CONNECTIVITY

## Objective

Xác nhận iLO hoạt động và đọc trạng thái power mà không thay đổi trạng thái power.

---

## Verify

```text
iLO accessible = Yes/No
Server power state = On/Off
Management connection stable = Yes/No
```

Có thể thêm:

```text
iLO health
iLO security state
```

nếu available.

Current HPE iLO documentation phân biệt iLO health, iLO security, iLO Event Log và Integrated Management Log trên overview. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00005342en_us&docLocale=en_US&page=GUID-76E5C538-A1FC-4912-97AA-55C63BC3E2B8.html&utm_source=chatgpt.com)

---

# Không làm

Không click:

```text
Reset
Cold Boot
Power Off
Power Cycle
```

chỉ để “thử xem”.

Đây là disruptive operations.

---

# LAB 5 — OVERALL HARDWARE HEALTH

## Objective

Đánh giá hardware health có cấu trúc.

---

## Health domains

Kiểm tra:

```text
Overall Server Health
Processor
Memory
Storage
Controller
Drive
NIC
Power
Cooling
Temperature
Firmware
```

---

# Kết luận phải có scope

Không chỉ viết:

```text
Healthy
```

Tốt hơn:

```text
At 2026-10-21 10:40 +07:00, iLO reported overall server
health as OK. CPU, memory, storage, power and cooling checks
reviewed in the lab showed no active critical fault.
```

Hoặc nếu có limitation:

```text
Drive SMART details were unavailable to the assigned account;
storage health is therefore based on the controller/iLO summary only.
```

Đây là cách viết audit-quality.

---

# LAB 6 — IML COLLECTION

## Objective

Thu thập hardware event history.

HPE mô tả Integrated Management Log là record của historical server events; iLO overview cung cấp direct access tới IML. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00005342en_us&docLocale=en_US&page=GUID-76E5C538-A1FC-4912-97AA-55C63BC3E2B8.html&utm_source=chatgpt.com)

---

## Step 1 — Open IML

Exact navigation phụ thuộc iLO generation/version.

Do đó không học thuộc:

```text
Click menu A → menu B → menu C
```

mà cần nhận biết object:

```text
Integrated Management Log
```

---

## Step 2 — Xác định window

Ví dụ:

```text
From: 2026-10-21 09:00 +07:00
To:   2026-10-21 11:00 +07:00
```

---

## Step 3 — Review events

Với mỗi event đáng chú ý:

```text
Timestamp
Severity
Component
Message
Status
```

---

## Step 4 — Không clear log

Day 6 tuyệt đối không yêu cầu:

```text
Clear IML
```

Việc clear log phá evidence history và có thể làm investigation khó hơn.

---

## Step 5 — Export nếu lab cho phép

Giữ raw export.

Ví dụ:

```text
raw/
  D6-HPE-IML-RAW.csv

working/
  D6-HPE-IML-FILTERED.csv
```

Không edit raw export.

---

# LAB 7 — REMOTE CONSOLE OBSERVATION

## Objective

Hiểu Remote Console operationally nhưng không gây disruption.

---

## Safe exercise

Instructor có thể cho phép:

1. mở Remote Console;
2. xác nhận server console đang hiển thị;
3. xem current screen;
4. đóng console.

Không:

- reboot;
- BIOS change;
- send disruptive key sequence;
- alter boot order.

---

## Verify

Record:

```text
Remote Console access: Available
Observed power state: On
Host screen: Normal / login screen / hypervisor console
Changes performed: None
```

---

# LAB 8 — VIRTUAL MEDIA AWARENESS

Đây nên là demonstration trừ khi instructor có controlled sandbox.

Bạn cần xác định:

```text
Virtual Media function available?
Current media attached?
User has privilege?
```

Không mount arbitrary ISO vào production-like host.

---

# LAB 9 — HPE MAINTENANCE ASSESSMENT

Sau inventory + health + IML, bạn phải đưa ra một trong các status:

```text
No maintenance required
Monitoring required
Planned maintenance required
Urgent maintenance required
Vendor escalation required
Insufficient evidence
```

Không được dùng trạng thái mơ hồ như:

```text
Seems fine.
```

---

# PHẦN HAI CỦA LAB — NUTANIX

# LAB 10 — PRISM ELEMENT ACCESS

## Objective

Xác định đúng cluster và phạm vi quản lý.

---

## Step 1 — Login Prism Element

Dùng approved lab account.

Ngay sau login xác nhận:

```text
Cluster name
Environment
Expected node count
Current health
```

---

## Step 2 — Không nhầm Prism Element với Prism Central

Record:

```text
System: Prism Element
Scope: Local cluster
Cluster: <name>
```

---

# LAB 11 — PRISM CENTRAL ACCESS

Nếu lab có Prism Central:

Record:

```text
System: Prism Central
Managed clusters visible
Target cluster
```

Mục tiêu thực hành không chỉ là login mà phải chứng minh bạn hiểu:

```text
Prism Element = local cluster
Prism Central = centralized management scope
```

---

# LAB 12 — CLUSTER INVENTORY

Tạo cluster inventory:

```text
Cluster Name
Cluster ID/UUID if authorized
AOS version
Hypervisor
AHV version
Node count
Cluster IP/VIP if allowed
Prism Element address
Prism Central registration/status
```

Không cần expose sensitive addresses trong deliverable nếu policy cấm.

---

# LAB 13 — NODE INVENTORY

Trong Prism Element, nhận diện các physical nodes/hosts.

Current Nutanix documentation vẫn sử dụng Prism Element `Hardware` view để xác nhận node membership. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=NIC-Addition-Cisco-HCI%3Abre-node-remove-verify-t.html&utm_source=chatgpt.com)

Tạo mapping:

| Node  | AHV Host | CVM   | Hardware state | Notes       |
| ----- | -------- | ----- | -------------- | ----------- |
| Node1 | AHV01    | CVM01 | Healthy        | —           |
| Node2 | AHV02    | CVM02 | Healthy        | —           |
| Node3 | AHV03    | CVM03 | Warning        | Investigate |

Không giả định số node.

---

# LAB 14 — AHV HOST INVENTORY

Nếu CVM SSH được authorized, một read-only inventory command được official Nutanix documentation sử dụng là:

```bash
acli host.list
```

Current documentation cho thấy output có thể chứa các fields như:

```text
Hypervisor IP
Hypervisor DNS Name
Host UUID
Node state
Connected
Node type
Schedulable
Hypervisor Name
CVM IP
```

tùy version/role. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Security-Guide%3Aportal-full-page-view-html&utm_source=chatgpt.com)

---

## Classification

```text
Command: acli host.list
Purpose: inventory
Risk: read-only observation
```

Không nhầm với:

```bash
acli host.enter_maintenance_mode ...
```

Command thứ hai thay đổi host state và **không phải routine Day 6 exercise**.

---

# LAB 15 — `ncli host ls`

Một inventory command khác xuất hiện trong Nutanix operational documentation:

```bash
ncli host ls
```

Nutanix sử dụng command này để liệt kê host-related information trong các maintenance procedures. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide-v10_0%3Aahv-cluster-shut-down-t.html&utm_source=chatgpt.com)

Trong Day 6 chỉ dùng để:

- map hosts;
- compare với Prism;
- record evidence.

---

# LAB 16 — CVM MAPPING

Bạn phải xác định:

```text
Node ↔ AHV Host ↔ CVM
```

Ví dụ:

```text
Node 01
├── AHV: 10.10.x.x
└── CVM: 10.10.y.y
```

Không cần public/expose real IPs trong report nếu security policy không cho phép.

---

# Cảnh báo quan trọng

Không mặc định mọi node topology đều giống nhau.

Current Nutanix platforms có thể có node roles khác nhau; do đó hãy lấy:

```text
Node Type
CVM mapping
Storage role
```

từ actual environment thay vì tự điền dựa trên giả định.

---

# LAB 17 — AOS / AHV VERSION VERIFICATION

Record:

```text
AOS Version
AHV Version
Prism Central Version nếu applicable
NCC Version nếu được phép xem
```

Không dùng mục đích:

> Có version mới → upgrade.

Day 6 chỉ xây baseline.

---

# LAB 18 — STORAGE INVENTORY

Trong Prism, xác định:

```text
Storage Pool
Containers
Capacity
Used
Available
Health
```

Exact terminology/menu có thể thay đổi theo release.

---

# Không làm trong Day 6

Không:

```text
Create container
Delete container
Change replication/resilience
Resize arbitrary storage configuration
```

trừ khi một lab riêng được instructor phê duyệt.

---

# LAB 19 — VIRTUAL NETWORK INVENTORY

Xác định:

```text
Virtual networks
VLAN IDs nếu visible/authorized
AHV networking objects
Relevant VM networks
```

Không sửa VLAN/network.

Mục tiêu là lập mapping:

```text
VM
 ↓
Virtual network
 ↓
AHV
 ↓
Physical network dependency
```

---

# LAB 20 — CLUSTER HEALTH DASHBOARD

## Objective

Xác nhận current cluster health.

Review:

```text
Overall health
Node health
Host health
Storage health
Critical alerts
Warnings
```

---

# Không chỉ nhìn màu

Nếu dashboard:

```text
GREEN
```

phải vẫn xem:

```text
Active alerts
Recent events
Capacity
Expected nodes
```

Nếu:

```text
RED
```

không tự ý remediate.

---

# LAB 21 — ALERT REVIEW

## Prism Element

Review active alerts.

Với từng relevant alert record:

```text
Alert name
Severity
Entity
Timestamp
Current state
Knowledge/KB reference nếu có
```

Nếu CVM SSH được phép, current Nutanix AHV documentation dùng:

```bash
ncli alert ls
```

để xem cluster alerts trước disruptive operations. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

---

# Expected workflow

```text
Alert
  ↓
Identify affected entity
  ↓
Determine physical/logical layer
  ↓
Check corresponding source
  ↓
Check impact
  ↓
Check resiliency
```

---

# LAB 22 — EVENTS / TASKS REVIEW

Review recent:

```text
Events
Tasks
```

Mục đích là tạo chronology.

Ví dụ:

```text
10:02 Node warning appeared
10:03 Hardware alert generated
10:05 Administrator opened alert
10:09 iLO IML reviewed
```

Không confuse:

```text
Alert acknowledgement
```

với:

```text
Problem resolution
```

---

# LAB 23 — BASIC CLUSTER SERVICE STATUS

Nếu SSH access được authorized, một standard current Nutanix diagnostic command là:

```bash
cluster status
```

Nutanix current support documentation sử dụng command này để xác nhận CVMs và cluster services. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

---

## Expected reasoning

Output phải được đọc theo:

```text
Expected CVMs present?
Services UP?
Any unexpected STOPPED/DOWN state?
```

Không cần thuộc lòng toàn bộ service names ở Day 6.

---

# Filter nhanh

Official recovery guidance cũng sử dụng:

```bash
cluster status | grep -v UP
```

để nhanh chóng tìm entries không ở state `UP`. Nếu output trống trong context tương ứng, không có service entry bất thường theo filter đó. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

---

## Evidence

```bash
cluster status > D6_Nutanix_ClusterStatus.txt
```

Lệnh shell redirection chỉ ghi output command sang file trong working context; hãy đảm bảo đường dẫn được phép và không overwrite evidence cũ.

Tốt hơn dùng timestamped filename:

```bash
cluster status > D6_CLUSTER01_ClusterStatus_20261021T113000.txt
```

---

# LAB 24 — NCC HEALTH CHECK

## NCC là gì trong lab?

NCC — Nutanix Cluster Check — cung cấp health checks cho cluster.

Current Nutanix documentation cho phép chạy NCC từ Prism Element Health dashboard hoặc CLI. Official guidance hiện tại sử dụng:

```bash
ncc health_checks run_all
```

và yêu cầu investigate statuses khác `INFO`/`PASS` trước upgrade/maintenance. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=NCC-Guide-NCC-v5_1%3Ancc-ncc-checks-running-t.html&utm_source=chatgpt.com)

---

# Safety

Dù đây là diagnostic command:

- chạy trong authorized lab;
- xác nhận đúng NCC version;
- không tự chạy tùy tiện trên production nếu runbook/change không cho phép;
- lưu output.

---

# Expected result

Không nhất thiết mọi training environment đều trả toàn PASS.

Mục tiêu là biết phân loại:

```text
PASS
INFO
WARN
FAIL
ERROR
```

và investigate.

Không được sửa output để “đẹp”.

---

# LAB 25 — SERVICE-SPECIFIC NCC CHECK

Ví dụ current Nutanix support guidance có:

```bash
ncc health_checks system_checks cluster_services_down_check
```

để kiểm tra services bị down. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

Day 6 có thể dùng nếu instructor đã chuẩn bị simulated service warning.

---

# LAB 26 — HARDWARE NCC CHECK

Current Nutanix recovery guidance có hardware check:

```bash
ncc health_checks hardware_checks power_supply_check
```

để đánh giá PSU-related condition. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

Đây đặc biệt hữu ích trong Day 6 vì có thể correlation:

```text
NCC
↕
Prism alert
↕
HPE iLO / IML
```

---

# LAB 27 — DATA RESILIENCY / FAULT-TOLERANCE CHECK

Một trong những command quan trọng nhất về maintenance reasoning:

```bash
ncli cluster get-domain-fault-tolerance-status type=node
```

Current Nutanix guidance dùng nó để xác nhận khả năng chịu node failure trước disruptive maintenance. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

---

# Không đọc command theo kiểu Yes/No máy móc

Nếu result cho thấy cluster không đủ fault tolerance:

```text
STOP planned node maintenance
```

và investigate/escalate.

Không tiếp tục chỉ vì:

> VM đang chạy.

---

# LAB 28 — VM POWER-STATE AWARENESS

Day 6 không provisioning VM nhưng phải hiểu workloads nào phụ thuộc cluster.

Trong Prism Element:

```text
VMs → Table
```

review expected VM power states.

Nutanix current guidance coi Prism Element VM table là cách hypervisor-independent để xác nhận expected VM power states. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

Trên AHV, authorized read-only CLI:

```bash
acli vm.list power_state=on
```

có thể hỗ trợ inventory. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

---

# LAB 29 — CAPACITY CHECK

Record ít nhất:

```text
CPU utilization
Memory utilization
Storage utilization
Available/headroom indication
```

Không chỉ lấy số.

Phải trả lời:

> Nếu NODE02 cần maintenance, cluster hiện có dấu hiệu capacity đủ hay có risk?

Day 6 không yêu cầu bạn tính HA scheduling model sâu; nếu không có đủ data, conclusion đúng là:

```text
Insufficient evidence to authorize maintenance.
```

---

# LAB 30 — CONFIGURATION BASELINE

Đây là một deliverable chính.

Tạo baseline:

```text
Day 6 Configuration Baseline
============================

Environment:
LAB

HPE:
Asset:
Model:
iLO generation/version:
Power:
Overall hardware health:
Firmware:
Storage controller:
Active IML concern:

Nutanix:
Cluster:
Node count:
AOS:
AHV:
Prism Element:
Prism Central:
Expected CVMs:
Storage:
Networks:
Critical alerts:
Capacity summary:
Fault-tolerance status:
```

---

# Ý nghĩa của baseline

Sau maintenance/incident bạn có:

```text
BEFORE
  ↓
ACTION
  ↓
AFTER
```

Nếu không capture `BEFORE`, post-change comparison yếu hơn rất nhiều.

---

# PHẦN 6 — DAILY HPE HEALTH CHECK

Một Day 6 learner nên có khả năng chạy checklist kiểu:

```text
HPE Daily Health

[ ] Correct asset verified
[ ] iLO reachable
[ ] Power state expected
[ ] Overall hardware health reviewed
[ ] CPU health reviewed
[ ] Memory health reviewed
[ ] Storage/controller health reviewed
[ ] PSU/redundancy reviewed
[ ] Fan/thermal state reviewed
[ ] NIC health reviewed
[ ] New IML events reviewed
[ ] Firmware baseline known
[ ] Unresolved warning documented
[ ] Evidence timestamp recorded
[ ] Escalation created if required
```

Không có checkbox:

```text
[ ] Everything looks okay
```

vì quá mơ hồ.

---

# PHẦN 7 — DAILY NUTANIX HEALTH CHECK

```text
Nutanix Daily Health

[ ] Correct cluster verified
[ ] Prism Element reachable
[ ] Prism Central status reviewed where applicable
[ ] Expected nodes present
[ ] AHV hosts accounted for
[ ] CVMs accounted for
[ ] Cluster services reviewed
[ ] Critical alerts reviewed
[ ] Recent relevant events reviewed
[ ] Hardware warnings reviewed
[ ] Storage capacity reviewed
[ ] CPU/memory capacity reviewed
[ ] Fault-tolerance/resiliency status considered
[ ] Versions recorded
[ ] Open issues mapped to correct owner
```

---

# PHẦN 8 — FAILURE INJECTION 1: PSU FAILURE

Đây là một safe simulated scenario rất phù hợp.

Instructor cung cấp:

```text
IML:
Power Supply 2 Failure

Prism:
Hardware warning on Node02

VMs:
Running

Cluster:
Otherwise healthy
```

---

## Nhiệm vụ của bạn

### 1. Confirm scope

```text
Cluster?
Node?
Environment?
```

### 2. Check iLO

Verify:

```text
PSU state
Overall power health
Redundancy state nếu available
IML timestamp
```

### 3. Check Prism

Verify:

```text
Node state
Hardware warning
Cluster availability
```

### 4. Assess impact

Correct answer không phải:

```text
No impact because VMs run.
```

Tốt hơn:

```text
No observed workload outage at this time.
Physical power redundancy is degraded, increasing the risk
of node outage if the remaining power path fails.
```

### 5. Escalation

Likely primary owner:

```text
Hardware/HPE support
```

Nutanix operations vẫn có thể tham gia vì node là cluster member.

---

# PHẦN 9 — FAILURE INJECTION 2: DISK WARNING

Fixture:

```text
iLO IML:
Predictive drive failure

Prism:
Node hardware warning

Cluster:
Available
```

---

## Investigation

Check:

```text
Correct node?
Correct bay/device?
Controller health?
Other drive faults?
Cluster health?
Resilience?
Capacity?
```

---

# Không làm

Không rút disk.

Không tự replace.

Không tạo RAID.

Không reboot host.

---

# Expected decision

```text
Physical storage warning confirmed.
Cluster currently available.
Replacement requires approved platform/vendor procedure and
pre-maintenance resiliency checks.
```

---

# PHẦN 10 — FAILURE INJECTION 3: CVM/SERVICE WARNING

Fixture:

```text
Prism:
Service warning

iLO:
Hardware healthy
```

---

## Investigation path

```text
Hardware healthy?
       ↓
AHV healthy?
       ↓
CVM expected/present?
       ↓
cluster status
       ↓
NCC service check
```

---

## Key learning

Nếu iLO fully healthy nhưng Nutanix services unhealthy:

```text
Do not escalate immediately as physical hardware failure.
```

Primary failing layer có thể nằm ở:

```text
CVM/AOS platform
```

---

# PHẦN 11 — FAILURE INJECTION 4: PRISM CENTRAL UNAVAILABLE

Fixture:

```text
Prism Central unreachable
Prism Element reachable
Cluster healthy
VMs reachable
```

---

## Expected reasoning

```text
Central management plane issue
≠
cluster outage
```

Investigate:

```text
PC availability
management network
DNS/access
PC-specific alerts
```

Không reboot AHV nodes để “fix Prism Central”.

---

# PHẦN 12 — FAILURE INJECTION 5: CAPACITY WARNING

Fixture:

```text
Node01 healthy
Node02 healthy
Node03 healthy

Storage capacity: high utilization
or
Memory headroom: low
```

A maintenance request asks:

```text
Take Node03 offline.
```

---

## Expected action

Không chỉ check:

```text
Node03 healthy?
```

Check:

```text
Remaining capacity?
Cluster resilience?
Critical alerts?
Workload placement?
```

Nếu không đủ headroom:

```text
STOP / DEFER / ESCALATE
```

Đây là một trong những lessons quan trọng nhất Day 6.

---

# PHẦN 13 — ILO ↔ PRISM CORRELATION EXERCISE

Syllabus yêu cầu trực tiếp việc correlate hardware alert/IML với Prism health evidence. Infrastructure_Operations_14_Da…

Bạn nên tạo timeline:

| Time  | Source   | Entity      | Observation               |
| ----- | -------- | ----------- | ------------------------- |
| 10:02 | iLO IML  | Node02 PSU2 | PSU failure               |
| 10:03 | Prism    | Node02      | Hardware warning          |
| 10:04 | Prism    | Cluster     | Cluster remains available |
| 10:05 | VM check | VM-A        | Running                   |

---

# Từ timeline phải suy ra

```text
Earliest authoritative hardware signal:
iLO/IML

Higher-level reflection:
Prism hardware alert

Current service impact:
No outage observed

Risk:
Power redundancy degraded

Primary failure layer:
Physical hardware

Primary escalation:
Hardware/HPE
```

---

# Không được suy luận quá evidence

Nếu không biết PSU feed topology:

Không viết:

> Power Feed B failed.

Chỉ viết:

> PSU2 reported failed; upstream power-feed state was not independently verified.

Đó là evidence discipline.

---

# PHẦN 14 — FAILURE-LAYER MATRIX

Trong investigation, lập bảng:

| Evidence                  | Hardware | AHV | CVM/AOS | Prism | Network | Guest |
| ------------------------- | -------: | --: | ------: | ----: | ------: | ----: |
| iLO IML hardware fault    |        ✓ |     |         |       |         |       |
| AHV host abnormal         |        ? |   ✓ |         |       |       ? |       |
| `cluster status` issue    |          |     |       ✓ |       |       ? |       |
| PC only inaccessible      |          |     |         |     ✓ |       ? |       |
| Guest service failed only |          |     |         |       |       ? |     ✓ |

`?` nghĩa là cần thêm evidence.

Một strong operator không force every symptom thành một root cause quá sớm.

---

# PHẦN 15 — TROUBLESHOOTING ORDER

Day 6 nên luôn dùng thứ tự ít rủi ro → nhiều rủi ro:

```text
1. Verify target/environment
        ↓
2. Observe dashboard
        ↓
3. Inspect alerts/events
        ↓
4. Inspect hardware/iLO/IML
        ↓
5. Check inventory/mapping
        ↓
6. Check cluster/service health
        ↓
7. Check capacity/resilience
        ↓
8. Correlate evidence
        ↓
9. Identify probable failing layer
        ↓
10. Consult approved runbook
        ↓
11. Escalate or approved remediation
```

Không:

```text
Alert
 ↓
Restart
```

---

# PHẦN 16 — TROUBLESHOOTING COMMAND SET

Các command dưới đây chỉ được chạy khi CVM SSH được instructor cho phép.

## 1. Host inventory

```bash
acli host.list
```

Purpose:

```text
Map hypervisor hosts, UUID/state/role/CVM information
depending on deployed version.
```

Read-only.

---

## 2. Alternative host inventory

```bash
ncli host ls
```

Purpose:

```text
List cluster host information.
```

Read-only. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide-v10_0%3Aahv-cluster-shut-down-t.html&utm_source=chatgpt.com)

---

## 3. Cluster service health

```bash
cluster status
```

Purpose:

```text
Check CVM/cluster service status.
```

Read-only diagnostic. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

---

## 4. Filter abnormal service state

```bash
cluster status | grep -v UP
```

Purpose:

```text
Quickly identify entries not containing UP.
```

Không dùng filter này như complete health proof; NCC và context vẫn cần thiết. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

---

## 5. Alerts

```bash
ncli alert ls
```

Purpose:

```text
List cluster alerts.
```

Current AHV maintenance guidance sử dụng nó trước disruptive operation. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

---

## 6. NCC

```bash
ncc health_checks run_all
```

Purpose:

```text
Execute NCC cluster health checks.
```

Diagnostic nhưng cần approved use. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=NCC-Guide-NCC-v5_1%3Ancc-ncc-checks-running-t.html&utm_source=chatgpt.com)

---

## 7. Down-service check

```bash
ncc health_checks system_checks cluster_services_down_check
```

Purpose:

````text
Identify cluster service-down conditions.
``` :chatgpt-content-reference{index="25"}


---

## 8. PSU check

```bash
ncc health_checks hardware_checks power_supply_check
````

Purpose:

````text
Evaluate underlying PSU-related condition.
``` :chatgpt-content-reference{index="26"}


---

## 9. Fault-tolerance status

```bash
ncli cluster get-domain-fault-tolerance-status type=node
````

Purpose:

```text
Assess node-domain fault-tolerance status.
```

Very important before host maintenance. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

---

## 10. VM power-state awareness

```bash
acli vm.list power_state=on
```

Purpose:

```text
List powered-on AHV VMs.
```

Read-only. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

---

# PHẦN 17 — COMMANDS KHÔNG PHẢI ROUTINE DAY 6

Bạn có thể thấy trong vendor documentation:

```bash
acli host.enter_maintenance_mode <hypervisor-IP>
```

hoặc:

```bash
cluster stop
```

hoặc:

```bash
cluster start
```

Đây **không phải các command để chạy thử trong Day 6**.

Chúng thay đổi operational state.

Current Nutanix documentation thậm chí yêu cầu không tiếp tục nếu host không enter maintenance mode đúng cách và có additional restrictions đối với cluster configuration/HA. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Security-Guide%3Aportal-full-page-view-html&utm_source=chatgpt.com)

Chúng ta sẽ chỉ sử dụng chúng trong:

- instructor-controlled demonstration;
- dedicated Day 7 host-maintenance exercise;
- authorized sandbox.

---

# PHẦN 18 — MAINTENANCE REHEARSAL

Syllabus yêu cầu draft maintenance plan, không bắt buộc thực hiện disruptive maintenance thật.

Giả sử:

```text
Node02 has failed PSU.
Hardware team requests replacement.
```

Bạn phải tạo maintenance plan.

---

# 18.1 Scope

```text
Affected cluster:
NTNX-LAB-01

Affected node:
Node02

Affected component:
PSU2

Related AHV:
AHV02

Related CVM:
CVM02

Potentially affected VMs:
<list/reference>
```

---

# 18.2 Business/technical impact

Ví dụ:

```text
Current observed impact:
No workload outage.

Risk:
Power redundancy is degraded.

Expected maintenance impact:
Dependent on replacement procedure and platform design.
Node shutdown may be required if hot replacement is not
supported/approved.
```

Không đoán `zero downtime`.

---

# 18.3 Preconditions

```text
[ ] Correct asset verified
[ ] Approved change
[ ] Maintenance window
[ ] Current hardware evidence captured
[ ] Prism cluster healthy
[ ] Critical alerts reviewed
[ ] Capacity reviewed
[ ] Fault tolerance reviewed
[ ] Workloads identified
[ ] Replacement part validated
[ ] HPE procedure available
[ ] Nutanix procedure available if node disruption required
[ ] Escalation contact available
```

---

# 18.4 Stop conditions

Rất quan trọng.

```text
STOP if:

- another node becomes unhealthy;
- new critical storage alert appears;
- required fault tolerance is unavailable;
- capacity is insufficient;
- target node identity cannot be conclusively verified;
- unexpected VM outage occurs;
- replacement part does not match;
- vendor procedure conflicts with planned sequence;
- cluster service health degrades unexpectedly.
```

---

# 18.5 Implementation sequence

Vì Day 6 chỉ rehearsal:

```text
1. Confirm approval.
2. Run prechecks.
3. Confirm correct hardware.
4. Hand off to authorized hardware engineer/vendor.
5. Physical replacement performed per exact HPE procedure.
6. Observe hardware recovery.
7. Run postchecks.
```

Không invent physical replacement instructions nếu chưa biết exact ProLiant model.

---

# PHẦN 19 — POST-MAINTENANCE CHECKLIST

Sau theoretical/simulated maintenance:

### HPE

```text
[ ] Component now healthy
[ ] Overall health normal
[ ] Power redundancy restored
[ ] No unexpected IML events
[ ] Firmware/state unchanged unless planned
```

### Nutanix

```text
[ ] Node present
[ ] AHV healthy
[ ] CVM expected/present
[ ] Cluster services healthy
[ ] Critical alerts reviewed
[ ] Storage healthy
[ ] Capacity normal
[ ] Resilience restored
[ ] Expected VMs available
```

### Evidence

```text
[ ] Post-maintenance timestamp
[ ] Before/after comparison
[ ] Ticket updated
[ ] Outstanding issue recorded
```

---

# PHẦN 20 — ROLLBACK VS RECOVERY

Một trong những phần quan trọng nhất của lab.

## Configuration change

Có thể có backout:

```text
restore previous setting
```

nếu vendor/runbook cho phép.

## Firmware

Không được mặc định:

```text
install old firmware = safe rollback
```

## Hardware replacement

Nếu replacement part fail:

```text
reinstall old failed component
```

không nhất thiết là useful rollback.

## Cluster maintenance

Nếu health deteriorates:

```text
STOP
→ stabilize
→ escalate
```

thường quan trọng hơn việc cố “reverse every step”.

Syllabus đặc biệt yêu cầu maintenance plan không giả định snapshot hoặc firmware downgrade guarantee rollback. Infrastructure_Operations_14_Da…

---

# PHẦN 21 — EVIDENCE COLLECTION WORKFLOW

Đây là phần không được bỏ qua.

## Raw

```text
Day06/raw/
```

Chứa original exports.

Không chỉnh sửa.

---

## Working

```text
Day06/working/
```

Có thể:

- filter;
- annotate;
- correlate.

---

## Deliverable

```text
Day06/deliverable/
```

Có redaction nếu cần.

---

# Hashing trên Linux workstation

```bash
sha256sum D6-HPE-IML-RAW.csv
```

Expected shape:

```text
<64-character-sha256>  D6-HPE-IML-RAW.csv
```

---

# Hashing trên PowerShell

```powershell
Get-FileHash `
  -Path '.\D6-HPE-IML-RAW.csv' `
  -Algorithm SHA256
```

Không nói:

> Matching SHA-256 proves the evidence is authentic.

Nó chỉ chứng minh copy đang kiểm tra giống bytes của baseline đã hash.

Syllabus cũng nhấn mạnh hash không tự chứng minh source authenticity/completeness. Infrastructure_Operations_14_Da…

---

# PHẦN 22 — EVIDENCE MANIFEST

Ví dụ:

| Evidence ID | Source  | Asset     | Time      | Ticket    | Artifact          | Result       |
| ----------- | ------- | --------- | --------- | --------- | ----------------- | ------------ |
| D6-HPE-001  | iLO     | Node02    | 10:02 +07 | INC-D6-01 | Health screenshot | Warning      |
| D6-HPE-002  | IML     | Node02    | 10:05 +07 | INC-D6-01 | IML export        | PSU2 failure |
| D6-NTNX-001 | Prism   | Cluster01 | 10:08 +07 | INC-D6-01 | Alert export      | Node warning |
| D6-NTNX-002 | CVM CLI | Cluster01 | 10:11 +07 | INC-D6-01 | `cluster status`  | Services UP  |

Sau đó hash artifacts nếu exportable.

---

# PHẦN 23 — INCIDENT TIMELINE

Bạn cần biến evidence rời rạc thành timeline:

```text
10:02  IML records PSU2 fault
10:03  Prism hardware alert generated
10:05  Operator acknowledges investigation
10:08  Cluster health checked
10:10  No VM outage observed
10:12  Fault tolerance reviewed
10:15  Hardware escalation initiated
```

Timeline cho phép reviewer hiểu:

- what happened;
- when;
- what you knew at each point.

---

# PHẦN 24 — ESCALATION RECORD

Một escalation tốt:

```text
Issue:
PSU2 failure on NODE02.

Environment:
LAB / Cluster NTNX-LAB-01.

First observed:
2026-10-21 10:02 +07:00.

Evidence:
D6-HPE-002 IML reports PSU2 failure.
D6-NTNX-001 Prism reports hardware warning on same node.

Impact:
No workload outage observed at time of investigation.

Risk:
Power redundancy reduced.

Cluster:
Services operational; capacity/resiliency evidence attached.

Actions performed:
Read-only health checks only.
No power/reset/replacement actions executed.

Requested action:
Hardware replacement assessment according to the supported HPE
procedure and Nutanix maintenance prerequisites.
```

Đây là L1/L2-quality escalation.

---

# PHẦN 25 — COMMON FAILURE: iLO KHÔNG REACHABLE

Bạn phải phân biệt:

```text
iLO unavailable
```

với:

```text
physical server down
```

Investigation:

```text
1. Correct address?
2. Client network?
3. DNS if using name?
4. Management network path?
5. iLO management interface?
6. Other management evidence?
```

Nếu host OS/VMs vẫn hoạt động:

```text
iLO management-path problem
```

có thể xảy ra mà server vẫn online.

Không power-cycle physical server để thử khôi phục iLO nếu chưa có approved procedure.

---

# PHẦN 26 — COMMON FAILURE: PRISM KHÔNG REACHABLE

Flow:

```text
Prism inaccessible
        ↓
Is network path okay?
        ↓
Is VIP/name resolution okay?
        ↓
Can individual management endpoint be reached?
        ↓
CVM state?
        ↓
cluster status?
        ↓
workload availability?
```

Một Prism UI problem không tự chứng minh cluster outage.

---

# PHẦN 27 — COMMON FAILURE: `cluster status` CÓ SERVICE DOWN

Không lập tức:

```bash
cluster start
```

hoặc restart service.

Đầu tiên:

```text
Which service?
One CVM or cluster-wide?
Related alert?
NCC finding?
Recent event?
Hardware/network issue?
```

Nếu ngoài runbook:

```text
collect evidence → escalate
```

---

# PHẦN 28 — COMMON FAILURE: NCC WARN/FAIL

Nếu:

```text
WARN / FAIL
```

đừng suppress hoặc rerun đến khi PASS.

Procedure:

```text
1. Record exact check.
2. Record affected entity.
3. Open associated vendor KB/runbook.
4. Determine impact.
5. Determine whether maintenance must stop.
6. Escalate if outside scope.
```

Current Nutanix NCC guidance yêu cầu resolve findings khác `INFO/PASS` trước maintenance/upgrade hoặc contact Nutanix Support nếu không thể xử lý. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=NCC-Guide-NCC-v5_1%3Ancc-ncc-checks-running-t.html&utm_source=chatgpt.com)

---

# PHẦN 29 — COMMON FAILURE: HARDWARE WARNING NHƯNG PRISM NORMAL

Có thể do:

- historical/resolved IML event;
- alert propagation delay;
- platform visibility differences;
- warning không được Prism surface theo cách bạn mong đợi.

Không kết luận một source sai.

Làm:

```text
Compare timestamps
Check current hardware state
Check active vs historical
Check node identity
Check Prism events
Record discrepancy
```

---

# PHẦN 30 — COMMON FAILURE: PRISM WARNING NHƯNG iLO HEALTHY

Possible categories:

```text
AHV
CVM/AOS
network
storage/platform logic
historical condition
```

Do đó chuyển investigation lên logical layers.

---

# PHẦN 31 — COMMON FAILURE: CAPACITY HIGH

Không tự:

- delete VM;
- delete snapshot;
- remove data;
- resize container.

Day 6 action:

```text
verify
quantify
identify owner
assess maintenance risk
escalate/capacity-plan
```

Actual cleanup thuộc approved change/runbook.

---

# PHẦN 32 — SECURITY CHECK TRONG LAB

Bạn phải kiểm tra security behavior của chính mình.

### Không chụp

- password;
- token;
- secret;
- private key.

### Không lưu

```text
admin_password.txt
```

### Không dùng shared privileged account nếu individual identity có sẵn.

### Không yêu cầu access cao hơn chỉ để xem health.

Nếu read-only role đủ:

```text
use read-only role
```

---

# PHẦN 33 — PRIVILEGE BOUNDARY

HPE iLO có privileges riêng cho các capability như Remote Console, Virtual Power/Reset, Virtual Media và BIOS/configuration functions; do đó account để health-check không mặc định cần disruptive privileges. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00005342en_us&docLocale=en_US&utm_source=chatgpt.com)

Nutanix cũng phải áp dụng cùng nguyên tắc:

```text
Read-only monitoring
≠
CVM SSH
≠
AHV root
≠
cluster administrator
```

---

# PHẦN 34 — REPORTING LIMITATIONS

Một professional report phải ghi:

```text
Limitations:
- Physical replacement was demonstrated only.
- No firmware update was executed.
- Prism Central was unavailable in this lab.
- Hardware sensor X was not visible to assigned role.
```

Không che gap.

Syllabus toàn khóa yêu cầu phân loại work là actual hands-on, simulation, demonstration hoặc theory/reference; simulation không được khai thành product hands-on thực tế. Infrastructure_Operations_14_Da…

---

# PHẦN 35 — FINAL DAY 6 DELIVERABLE PACKAGE

Bạn nên kết thúc Day 6 với một package tương tự:

```text
Day06_Final/
│
├── 00_README.txt
│
├── 01_Inventory/
│   ├── HPE_Hardware_Inventory.xlsx/csv
│   └── Nutanix_Cluster_Inventory.xlsx/csv
│
├── 02_HPE/
│   ├── Health/
│   ├── Firmware/
│   └── IML/
│
├── 03_Nutanix/
│   ├── Prism_Health/
│   ├── Hosts_CVMs/
│   ├── Alerts_Events/
│   ├── Storage/
│   └── Capacity/
│
├── 04_Correlation/
│   └── Incident_Timeline.xlsx/csv
│
├── 05_Troubleshooting/
│   └── D6_Investigation_Record.md
│
├── 06_Maintenance/
│   └── D6_Maintenance_Plan.md
│
├── 07_Escalation/
│   └── D6_Escalation_Record.md
│
└── manifest.csv
```

File types cụ thể có thể thay đổi theo environment.

---

# PHẦN 36 — ACCEPTANCE CRITERIA

Syllabus yêu cầu cuối Day 6 bạn phải chứng minh được:

### 1. Hardware inventory

```text
PASS:
Correctly identifies HPE hardware/components and firmware baseline.
```

### 2. iLO operational understanding

```text
PASS:
Can verify management connectivity, power and health without
performing unauthorized disruptive actions.
```

### 3. IML

```text
PASS:
Can locate/capture relevant hardware events with timestamps.
```

### 4. Nutanix inventory

```text
PASS:
Can identify cluster, nodes, AHV hosts, CVMs, storage and networks.
```

### 5. Prism scope

```text
PASS:
Correctly distinguishes Prism Element and Prism Central.
```

### 6. Health

```text
PASS:
Checks cluster, alerts, events, service health and capacity.
```

### 7. Correlation

```text
PASS:
Correlates HPE hardware evidence with Nutanix evidence.
```

### 8. Failure-layer identification

```text
PASS:
Identifies physical vs AHV vs CVM/AOS vs management/network/guest.
```

### 9. Escalation ownership

```text
PASS:
Routes problem to correct team/vendor with sufficient evidence.
```

### 10. Maintenance planning

```text
PASS:
Includes impact, prechecks, stop conditions and postchecks without
assuming guaranteed snapshot/firmware rollback.
```

Đây chính là acceptance requirement của syllabus: xác định failing layer và correct support owner, giữ timestamped hardware/cluster evidence cùng configuration baseline, và có maintenance plan an toàn. Infrastructure_Operations_14_Da…

---

# PHẦN 37 — PRACTICAL TROUBLESHOOTING FLOW PHẢI THUỘC

Đây là flow quan trọng nhất Module Thực hành Day 6:

```text
              ALERT / SYMPTOM
                     │
                     ▼
          Verify correct environment
                     │
                     ▼
             Determine scope
                     │
                     ▼
             Check current impact
                     │
                     ▼
       ┌──── Hardware alert? ────┐
       │                         │
      Yes                        No
       │                         │
       ▼                         ▼
    iLO/IML               Prism/AHV/CVM
       │                         │
       └────────────┬────────────┘
                    ▼
             Correlate evidence
                    │
                    ▼
           Check health/services
                    │
                    ▼
          Check capacity/resilience
                    │
                    ▼
           Identify failing layer
                    │
                    ▼
         Within approved scope?
              │             │
             Yes            No
              │             │
              ▼             ▼
       Approved action    Escalate
              │             │
              └──────┬──────┘
                     ▼
                  Verify
                     │
                     ▼
             Collect evidence
                     │
                     ▼
          Update ticket / handover
```

Nếu bạn áp dụng được flow này mà không nhảy thẳng tới restart, bạn đã bắt đầu hình thành đúng operational mindset.

---

# PHẦN 38 — DAY 6 PRACTICAL CHEAT SHEET

| Check                | Primary method              | Expected evidence             |
| -------------------- | --------------------------- | ----------------------------- |
| Correct HPE server   | iLO identity/inventory      | Asset/model screenshot/export |
| Power                | iLO                         | Power state                   |
| Hardware health      | iLO                         | Health summary                |
| Hardware event       | IML                         | Timestamped IML record        |
| Firmware             | iLO inventory               | Version baseline              |
| Nutanix cluster      | Prism Element               | Cluster identity              |
| Nodes                | Prism / `acli host.list`    | Node/host map                 |
| AHV                  | Prism/CLI                   | AHV host/version              |
| CVM                  | Prism/CLI mapping           | CVM mapping                   |
| Services             | `cluster status`            | Service-state output          |
| Alerts               | Prism / `ncli alert ls`     | Alert evidence                |
| NCC                  | `ncc health_checks run_all` | NCC report                    |
| Resilience           | approved Prism/NCLI check   | Fault-tolerance evidence      |
| VM awareness         | Prism / `acli vm.list`      | Expected workload state       |
| Capacity             | Prism                       | CPU/memory/storage evidence   |
| Hardware correlation | iLO + Prism                 | Timeline                      |
| Escalation           | ticket                      | Owner + evidence              |
| Maintenance          | change plan                 | precheck/stop/postcheck       |

---

# PHẦN 39 — DAY 6 PRACTICAL COMPLETION CHECKLIST

Trước khi coi Module Thực hành hoàn tất, bạn phải tự tick được:

### HPE / iLO

```text
[ ] Tôi xác nhận được đúng physical server.
[ ] Tôi thu được hardware inventory.
[ ] Tôi xác định được firmware baseline.
[ ] Tôi xác nhận power state.
[ ] Tôi review CPU/memory/storage/controller health.
[ ] Tôi review PSU/fan/thermal health.
[ ] Tôi biết mở và đọc IML.
[ ] Tôi phân biệt current health với historical event.
[ ] Tôi hiểu Remote Console privilege.
[ ] Tôi hiểu Virtual Media privilege.
[ ] Tôi không thực hiện unauthorized power/firmware operations.
```

### Nutanix

```text
[ ] Tôi xác định đúng cluster.
[ ] Tôi phân biệt Prism Element và Prism Central.
[ ] Tôi nhận diện nodes.
[ ] Tôi map được AHV hosts.
[ ] Tôi map được CVMs.
[ ] Tôi xác định AOS/AHV versions.
[ ] Tôi kiểm tra storage/container.
[ ] Tôi kiểm tra virtual networking ở mức inventory.
[ ] Tôi review alerts/events.
[ ] Tôi verify cluster service health.
[ ] Tôi hiểu cách dùng NCC.
[ ] Tôi review capacity.
[ ] Tôi đánh giá resilience trước maintenance.
```

### Troubleshooting

```text
[ ] Tôi điều tra từ read-only trước.
[ ] Tôi không reboot theo phản xạ.
[ ] Tôi correlate iLO và Prism.
[ ] Tôi xác định failing layer.
[ ] Tôi phân biệt symptom và root cause.
[ ] Tôi xác định correct support owner.
[ ] Tôi biết khi nào STOP.
[ ] Tôi biết khi nào escalate.
```

### Maintenance

```text
[ ] Tôi biết kiểm tra asset identity.
[ ] Tôi kiểm tra approval.
[ ] Tôi kiểm tra maintenance window.
[ ] Tôi kiểm tra current cluster health.
[ ] Tôi kiểm tra capacity.
[ ] Tôi kiểm tra resilience.
[ ] Tôi xác định workloads.
[ ] Tôi kiểm tra compatibility.
[ ] Tôi định nghĩa stop conditions.
[ ] Tôi định nghĩa postchecks.
[ ] Tôi không giả định firmware downgrade là guaranteed rollback.
[ ] Tôi không giả định snapshot là guaranteed rollback.
```

### Evidence

```text
[ ] Mỗi evidence có ID.
[ ] Có timestamp + timezone.
[ ] Có source/system.
[ ] Có asset/node/cluster identity.
[ ] Có ticket reference.
[ ] Raw evidence không bị sửa.
[ ] Working copy được tách riêng.
[ ] Export quan trọng có hash khi phù hợp.
[ ] Tôi ghi rõ limitation/gap.
[ ] Tôi có incident correlation timeline.
[ ] Tôi có final configuration baseline.
[ ] Tôi có escalation/maintenance record.
```

---

# Kết quả cuối cùng mà Day 6 yêu cầu ở bạn

Sau Module Thực hành, khi một Nutanix node báo warning, bạn **không nên phản xạ bằng “restart cái gì?”**.

Bạn phải có thể thực hiện tư duy sau:

```text
1. Tôi đang ở đúng cluster/node chưa?
2. Workload hiện có impact không?
3. Warning bắt nguồn từ physical hardware,
   AHV, CVM/AOS, Prism hay network?
4. iLO/IML nói gì?
5. Prism nói gì?
6. Hai nguồn có cùng node/timestamp/component không?
7. Cluster services còn healthy không?
8. Capacity và resilience còn đủ không?
9. Tôi được phép làm gì?
10. Nếu cần maintenance, prechecks là gì?
11. Stop conditions là gì?
12. Ai là correct support owner?
13. Tôi đã thu đủ evidence chưa?
14. Sau action sẽ verify những gì?
```

Nếu bạn trả lời được đủ 14 câu trên bằng evidence thay vì phỏng đoán, **Module Thực hành Day 6 đã đạt đúng mục tiêu operational readiness của syllabus**. Day 7 sau đó mới hợp lý để đi sâu vào VM provisioning, AHV operational administration, snapshots, host maintenance, NCC/LCM, upgrades và HYCU backup/recovery. Infrastructure_Operations_14_Da…
