# Day 7 — Module Lý thuyết

## Nutanix AOS/AHV Operational Administration & HYCU Backup/Recovery

Day 7 tiếp nối Day 6 nhưng thay đổi trọng tâm khá rõ. Day 6 chủ yếu giúp bạn hiểu **HPE/Nutanix infrastructure, architecture, inventory, health, dependency và operational boundary**; Day 7 đưa bạn sang mức **vận hành Nutanix AOS/AHV và HYCU như một Infrastructure Operations engineer**: provision VM, quản lý resource, theo dõi cluster, chuẩn bị maintenance, đọc Tasks/Alerts/Audit, đánh giá NCC/LCM, troubleshooting, bảo vệ workload bằng HYCU và quan trọng nhất là **chứng minh recovery thực sự hoạt động**. Đây chính xác là phạm vi Concept/Lecture và phần mở rộng của Day 7 trong syllabus. Infrastructure_Operations_14_Da…

Mình sẽ trình bày theo hai lớp rõ ràng:

- **Theo syllabus:** những kiến thức bắt buộc Day 7.
- **Kiến thức bổ sung/đối chiếu vendor:** giải thích sâu hơn để biến các topic đó thành operational understanding. Các chi tiết phụ thuộc version sẽ không được coi là universal behavior; khi thao tác thực tế phải đối chiếu đúng AOS, AHV, Prism Central, NCC, LCM và HYCU version của môi trường.

---

# PHẦN I — Bức tranh tổng thể của Nutanix operations

Trước khi học từng feature, bạn phải hình thành được một mental model của toàn platform.

Một Nutanix AHV environment đơn giản có thể hình dung:

```text
                     Management plane
        ┌──────────────────────────────────────┐
        │            Prism Central             │
        │     Multi-cluster / Central Mgmt     │
        └──────────────────┬───────────────────┘
                           │
                           │ manages / monitors
                           ▼
        ┌──────────────────────────────────────┐
        │             Prism Element            │
        │        Cluster-level management      │
        └──────────────────┬───────────────────┘
                           │
        ───────────────────┼────────────────────
                           │
                    Nutanix Cluster
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
     Node 1             Node 2             Node 3
   ┌────────┐         ┌────────┐         ┌────────┐
   │  AHV   │         │  AHV   │         │  AHV   │
   │        │         │        │         │        │
   │  CVM   │         │  CVM   │         │  CVM   │
   │        │         │        │         │        │
   │  VMs   │         │  VMs   │         │  VMs   │
   └────────┘         └────────┘         └────────┘
        │                  │                  │
        └──────── Distributed AOS ────────────┘
                   Storage / Services
```

Mental model quan trọng nhất là:

```text
Physical hardware
      ↓
AHV host
      ↓
CVM / AOS distributed services
      ↓
Storage + Network
      ↓
Guest VM
      ↓
Guest OS
      ↓
Application
```

Nếu application mất kết nối, bạn không được mặc định đó là “VM issue”. Nó có thể nằm ở bất kỳ layer nào.

Ngược lại, một Nutanix cluster alert không có nghĩa mọi application đều down.

Đây là tư duy **layer isolation** sẽ theo bạn xuyên suốt Day 7.

---

# PHẦN II — AOS, AHV, CVM và Nutanix cluster

## 1. AOS là gì?

**AOS — Acropolis Operating System** là software platform cốt lõi của Nutanix, cung cấp các distributed services quan trọng cho cluster, đặc biệt là storage và các dịch vụ quản lý/resiliency liên quan.

Điểm cần nhớ là AOS không nên được hình dung đơn giản giống một traditional OS cài trên từng server rồi hoạt động độc lập.

Trong Nutanix HCI, các node phối hợp thành một distributed system.

Ví dụ cluster có:

```text
Node A
Node B
Node C
Node D
```

Thay vì coi mỗi node có một storage silo riêng:

```text
Node A → Disk A
Node B → Disk B
Node C → Disk C
```

Nutanix cung cấp một distributed storage architecture để workload sử dụng storage ở mức cluster.

Đối với Infrastructure Operations, điều này dẫn đến một nguyên tắc cực kỳ quan trọng:

> Khi troubleshooting storage, không được chỉ nhìn một physical disk hoặc một VM disk. Phải nhìn toàn bộ dependency từ VM → vDisk → container → AOS services → cluster/node → physical storage.

---

# 2. AHV là gì?

**AHV — Acropolis Hypervisor** là hypervisor của Nutanix.

AHV chịu trách nhiệm chạy các virtual machines.

Ở tầng operations, AHV phải được hiểu qua các object như:

```text
AHV Host
   │
   ├── VM-01
   ├── VM-02
   ├── VM-03
   └── CVM
```

Mỗi AHV host cung cấp compute resources như:

- CPU;
- memory;
- networking;
- VM execution environment.

Nhưng AHV không nên được học tách khỏi AOS.

Ví dụ:

```text
AHV = compute / virtualization layer
AOS = distributed platform / storage services
```

Một VM có thể chạy trên AHV nhưng vẫn gặp storage problem nếu AOS/storage layer có vấn đề.

Ngược lại, AOS cluster có thể healthy nhưng một VM vẫn unavailable vì guest OS hoặc application bị lỗi.

---

# 3. CVM — Controller VM

**CVM — Controller VM** là một object rất quan trọng trong Nutanix architecture.

Mỗi Nutanix node thường có CVM tương ứng, và các CVM phối hợp để cung cấp các Nutanix services.

Mental model:

```text
Physical Node
│
├── AHV
│
├── CVM
│
├── Guest VM A
├── Guest VM B
└── Guest VM C
```

CVM không phải một application VM thông thường mà Infrastructure Operations có thể shutdown tùy tiện.

Nó là một phần của platform.

Vì vậy phải phân biệt:

```text
Guest VM operation
≠
CVM operation
≠
AHV host operation
≠
Cluster operation
```

Mức độ blast radius tăng dần.

Restart một application process trong guest thường khác hoàn toàn với restart CVM hoặc AHV host.

---

# 4. Cluster services

Các CVM cùng tham gia các distributed Nutanix services.

Ở mức foundation, bạn chưa cần thuộc lòng toàn bộ internal service architecture, nhưng phải hiểu:

```text
Cluster healthy
    ↓
CVMs healthy
    ↓
Required cluster services healthy
    ↓
Storage/data services healthy
    ↓
AHV workload access healthy
```

Một cluster có nhiều internal services và các dependency lẫn nhau.

Vì vậy kiểu troubleshooting:

```text
Service X down
→ restart X
```

là tư duy nguy hiểm.

Phải hỏi:

```text
Tại sao service down?
Service down ở một CVM hay nhiều CVM?
Cluster vẫn có resiliency không?
Có maintenance đang chạy không?
Có storage/network dependency không?
Có alert hoặc NCC finding không?
Restart service có được runbook/vendor cho phép không?
```

---

# PHẦN III — Prism Element và Prism Central

Đây là một topic bắt buộc của Day 7.

Syllabus yêu cầu hiểu **Prism Element versus Prism Central management scope**, đồng thời phần mở rộng yêu cầu review cả **RBAC và scope** của hai platform. Infrastructure_Operations_14_Da…

---

# 5. Prism Element

**Prism Element**, thường viết tắt là **PE**, về bản chất là management interface gắn với một Nutanix cluster.

Mental model:

```text
Cluster A
   │
   └── Prism Element A

Cluster B
   │
   └── Prism Element B

Cluster C
   │
   └── Prism Element C
```

PE đặc biệt quan trọng cho cluster-local operational work.

Bạn thường quan tâm các object như:

- cluster;
- hosts;
- CVMs;
- VMs;
- storage;
- networks;
- health;
- alerts;
- tasks;
- events;
- upgrades;
- cluster configuration.

PE giúp bạn trả lời những câu hỏi như:

> Cluster này có healthy không?

> Host nào đang có vấn đề?

> VM này đang nằm ở cluster nào?

> Cluster này có alert gì?

> Storage capacity còn bao nhiêu?

> Operation vừa thực hiện thành công hay thất bại?

---

# 6. Prism Central

**Prism Central**, thường viết tắt **PC**, cung cấp centralized management trên các registered clusters.

Tài liệu Nutanix hiện hành mô tả Prism Central Infrastructure Dashboard là nơi cung cấp dynamic summary của các registered clusters; Prism Central cũng có centralized IAM/RBAC và management capabilities trên nhiều cluster. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Prism-Central-Guide-vpc_7_6%3Amul-home-dashboard-pc-r.html&utm_source=chatgpt.com)

Mental model:

```text
               Prism Central
                    │
       ┌────────────┼────────────┐
       │            │            │
       ▼            ▼            ▼
 Cluster A      Cluster B      Cluster C
   PE-A           PE-B           PE-C
```

PC hữu ích khi organization có nhiều clusters.

Ví dụ một enterprise có:

```text
Cluster-BKK-01
Cluster-BKK-02
Cluster-HN-01
Cluster-DR-01
Cluster-DEV-01
```

Infrastructure Operations cần một management view centralized thay vì đăng nhập từng PE chỉ để xem overall state.

---

# 7. Không được hiểu PE vs PC theo kiểu “PC thay thế PE”

Đây là một lỗi tư duy phổ biến.

Không nên nghĩ:

```text
Prism Central = phiên bản lớn hơn của Prism Element
```

Chính xác hơn:

```text
Prism Element
→ cluster-level operational management

Prism Central
→ centralized / multi-cluster management
```

Một số feature hoặc workflow có thể xuất hiện ở cả hai, nhưng:

- scope khác nhau;
- UI có thể khác;
- RBAC khác;
- feature availability phụ thuộc version/license;
- một số operation vẫn cần hiểu cluster-local state.

Do đó trong incident bạn có thể bắt đầu từ PC:

```text
Prism Central
→ thấy Cluster-B có alert
→ drill down
→ Prism Element / cluster-specific evidence
→ host/CVM/VM investigation
```

---

# 8. Operational rule: luôn biết mình đang thao tác ở scope nào

Trước khi bấm một operation, phải xác nhận:

```text
Environment?
Cluster?
VM?
Host?
Tenant/project?
Network?
User scope?
```

Một lỗi rất đơn giản nhưng nguy hiểm:

```text
Bạn nghĩ đang thao tác LAB-CLUSTER
nhưng UI đang ở PROD-CLUSTER.
```

Do đó production workflow nên luôn có bước identity verification:

```text
Environment
→ Prism instance
→ Cluster
→ Object
→ Ticket
→ Action
```

---

# PHẦN IV — Virtual Machine provisioning và resource allocation

Syllabus Day 7 yêu cầu:

- VM provisioning;
- CPU/memory/storage allocation;
- virtual disks;
- network interfaces;
- VLAN integration;
- power operations;
- console;
- migration/placement awareness;
- snapshots.

Infrastructure_Operations_14_Da…

---

# 9. VM là một tập hợp nhiều dependency, không phải chỉ “một server ảo”

Một AHV VM có thể được hình dung:

```text
VM
│
├── Identity
│   ├── Name
│   └── UUID
│
├── Compute
│   ├── vCPU
│   └── Memory
│
├── Storage
│   ├── vDisk 1
│   └── vDisk 2
│
├── Network
│   ├── vNIC 1
│   └── Network/VLAN
│
├── Firmware / boot configuration
│
└── Guest OS
```

Muốn VM hoạt động đúng, tất cả dependency phải hợp lệ.

---

# 10. vCPU

**vCPU** là compute resource expose cho guest VM.

Khi provision VM, câu hỏi không phải đơn giản là:

> “Có thể cấp 16 vCPU không?”

Mà phải hỏi:

```text
Workload requirement là gì?
Approved sizing là gì?
Host/cluster headroom thế nào?
Workload thực tế sử dụng bao nhiêu?
Việc tăng vCPU có thực sự giải quyết bottleneck không?
```

Một lỗi phổ biến là **overprovisioning theo cảm tính**.

Ví dụ:

```text
Application chậm
→ tăng từ 4 vCPU lên 16 vCPU
```

nhưng root cause thực tế có thể là:

- storage latency;
- memory pressure;
- guest OS problem;
- database lock;
- application dependency;
- network latency.

Resource addition không phải universal fix.

---

# 11. Memory allocation

Memory phải được đánh giá ở ít nhất hai góc:

```text
Configured memory
vs
Actual utilization
```

Ví dụ:

```text
VM-A = 64 GB RAM configured
Actual usage ≈ 10 GB
```

Không thể ngay lập tức kết luận 54 GB “wasted”, nhưng đó là tín hiệu cần capacity review.

Ngược lại:

```text
VM-B = 8 GB
Sustained utilization rất cao
Guest swapping
Application latency
```

có thể chỉ ra undersizing.

Infrastructure Operations cần tránh cả hai cực đoan:

```text
Under-provision
→ performance problem

Over-provision
→ capacity waste
→ reduced cluster headroom
→ maintenance difficulty
```

---

# 12. Resource headroom

**Headroom** là capacity dự phòng giúp cluster chịu được:

- workload growth;
- VM migration;
- node maintenance;
- node failure;
- temporary load spikes.

Ví dụ đơn giản:

```text
Cluster total memory = 1 TB
Current allocation/utilization rất gần 1 TB
```

Dù chưa có outage, đây có thể là operational risk.

Vì khi cần maintenance một host, workload trên host đó phải có nơi khác để chạy.

Do đó capacity planning không chỉ hỏi:

> “Hôm nay cluster còn đủ không?”

Mà còn phải hỏi:

> “Nếu mất/maintenance một node thì cluster còn đủ không?”

Nutanix hiện cũng yêu cầu kiểm tra cluster có thể chịu single-node failure trước các operation như restart CVM/AHV host hoặc host maintenance. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

---

# 13. Virtual disk

VM không sử dụng physical disk trực tiếp trong mô hình thông thường.

Mental model:

```text
Guest filesystem
      ↓
Guest virtual disk
      ↓
AHV VM vDisk
      ↓
Storage container
      ↓
AOS distributed storage
      ↓
Physical storage resources
```

Điều đó giải thích vì sao:

```text
Windows C: drive full
```

không nhất thiết nghĩa:

```text
Nutanix cluster storage full.
```

Ngược lại:

```text
Nutanix storage capacity critical
```

không nhất thiết nghĩa guest C: hiện tại đã full.

Hai layer khác nhau.

---

# 14. Storage Container

**Storage Container** là logical storage construct quan trọng trong AOS.

Khi provision VM, vDisk sẽ liên hệ với storage configuration/container theo environment.

Infrastructure Operations cần biết ít nhất:

- container nào đang được dùng;
- capacity;
- policy/characteristics phù hợp;
- cluster health;
- workload dependency.

Không được chọn container chỉ vì tên “có vẻ đúng”.

Ví dụ:

```text
PROD-CONT
BACKUP-CONT
TEST-CONT
```

Tên object không phải security control.

Phải đối chiếu build sheet/change ticket.

---

# PHẦN V — Virtual networking và VLAN integration

---

# 15. VM networking có nhiều layer

Một network flow thường có thể nhìn như:

```text
Application
    ↓
Guest OS TCP/IP
    ↓
Guest vNIC
    ↓
AHV virtual network
    ↓
VLAN
    ↓
Physical NIC/uplink
    ↓
Physical switch
    ↓
Gateway / router
    ↓
Firewall
    ↓
Destination
```

Do đó symptom:

```text
ping destination failed
```

không tự động kết luận được root cause.

---

# 16. vNIC

Một VM có thể có một hoặc nhiều **virtual NIC — vNIC**.

Mỗi vNIC phải được gắn vào network phù hợp.

Ví dụ:

```text
VM-APP01
│
├── NIC1 → PROD_APP_VLAN
└── NIC2 → BACKUP_VLAN
```

Nếu NIC1 bị attach nhầm:

```text
DEV_VLAN
```

VM có thể:

- boot bình thường;
- Prism vẫn báo VM Running;
- disk vẫn accessible;

nhưng application network sẽ fail.

---

# 17. VLAN integration

Day 7 chỉ yêu cầu VLAN ở mức integration với Nutanix VM, không thay thế toàn bộ networking theory của Day 8.

Bạn cần hiểu:

**VLAN** tạo logical Layer-2 segmentation.

Ví dụ:

```text
VLAN 10 = Management
VLAN 20 = Application
VLAN 30 = Database
VLAN 40 = Backup
```

Khi VM được attach vào một AHV network mapped với VLAN phù hợp, upstream physical switching cũng phải transport VLAN đó đúng.

Nếu Nutanix configuration đúng nhưng upstream trunk không cho phép VLAN:

```text
VM
↓
AHV network
↓
VLAN 20
↓
Physical uplink
↓
Switch trunk
   X VLAN 20 missing
```

VM vẫn có thể hiện trạng thái Running nhưng không giao tiếp được.

---

# 18. Troubleshooting VM connectivity

Không nên bắt đầu bằng việc thay đổi network.

Workflow an toàn hơn:

```text
1. VM có Running không?
2. vNIC có attached không?
3. vNIC attach đúng network không?
4. Guest OS có thấy NIC không?
5. IP/subnet mask đúng không?
6. Default gateway đúng không?
7. Local route đúng không?
8. ARP/neighbour resolution?
9. Same-subnet connectivity?
10. Gateway connectivity?
11. Upstream VLAN/trunk?
12. Routing?
13. Firewall?
14. Application port?
```

Nguyên tắc:

> Thu thập evidence trước khi configuration change.

---

# PHẦN VI — VM power operations và console access

---

# 19. Power operations không tương đương nhau

Có sự khác biệt rất lớn giữa:

```text
Guest shutdown
Power off
Reset
Power cycle
```

### Graceful shutdown

Guest OS nhận shutdown process phù hợp.

Application có cơ hội:

- stop service;
- flush writes;
- close database;
- release resources.

Đây thường là lựa chọn an toàn hơn.

### Force power off

Tương tự cắt điện khỏi machine.

Có thể gây:

- incomplete writes;
- filesystem recovery;
- application inconsistency;
- database recovery requirement.

Do đó:

> “VM bị treo” không có nghĩa bạn được quyền force power off ngay.

---

# 20. Console access

VM console rất hữu ích khi network management path không còn hoạt động.

Ví dụ:

```text
SSH → failed
RDP → failed
```

Console cho phép xác định:

- guest có boot không;
- kernel/OS có crash không;
- NIC configuration có lỗi không;
- login prompt có xuất hiện không.

Nhưng console không phải “free administrative access”.

Nó vẫn là privileged access.

Phải tuân thủ:

- authorization;
- identity control;
- least privilege;
- audit;
- session handling.

Không copy password/secret vào evidence.

---

# PHẦN VII — VM placement và migration awareness

---

# 21. VM placement

VM phải chạy trên một AHV host.

Ví dụ:

```text
Cluster
├── Host-01 → VM-A, VM-B
├── Host-02 → VM-C
├── Host-03 → VM-D, VM-E
└── Host-04 → VM-F
```

Placement chịu ảnh hưởng bởi:

- available resources;
- policies;
- host state;
- VM constraints;
- cluster conditions.

Infrastructure Operations không cần cố gắng manually place mọi VM trừ khi runbook/policy yêu cầu.

Điều quan trọng là hiểu dependency.

---

# 22. Tại sao placement quan trọng khi maintenance?

Giả sử:

```text
Host-01
├── VM-A
├── VM-B
└── VM-C
```

Bạn muốn maintenance Host-01.

Các workload phải:

- migrate;
- shutdown;
- hoặc được xử lý bằng workflow phù hợp.

Nhưng các host còn lại cần đủ capacity.

Ví dụ:

```text
Host-02 = 85% memory
Host-03 = 90%
Host-04 = 88%
```

Dù cluster hiện tại “Running”, khả năng evacuate Host-01 có thể là vấn đề.

Do đó:

```text
Current health ≠ maintenance readiness
```

---

# 23. VM migration

Ở mức Day 7 bạn cần hiểu concept, không phải mọi edge case.

Migration cho phép workload chuyển từ host này sang host khác khi platform/configuration hỗ trợ.

Use cases:

- host maintenance;
- balancing;
- hardware work;
- upgrade.

Nhưng không phải mọi VM đều luôn migratable.

Một số special configuration có thể tạo constraint, vì vậy precheck phải xác định workload không thể di chuyển tự động. Nutanix upgrade guidance cũng mô tả một số workload có đặc tính khiến live migration không khả thi, cho thấy tại sao maintenance phải có precheck thay vì giả định mọi VM đều movable. [Nutanix Portal](https://portal.nutanix.com/page/solutions/TN-2122-Nutanix-Upgrades%3Aupgrading-third-party-hypervisors-with-life-cycle-manager.html?utm_source=chatgpt.com)

---

# PHẦN VIII — Snapshots và recovery points

Đây là một trong những topic quan trọng nhất Day 7.

Syllabus nhấn mạnh:

> phải **separate VM snapshots from independent backup protection**. Infrastructure_Operations_14_Da…

---

# 24. Snapshot là gì?

Snapshot ghi lại trạng thái dữ liệu tại một point-in-time theo mechanism của platform.

Nó rất hữu ích cho:

- short-term recovery;
- operational checkpoint;
- protection workflow;
- rollback-related scenarios phù hợp.

Nhưng:

```text
Snapshot ≠ automatically a complete backup strategy
```

---

# 25. Vì sao snapshot không được coi mặc định là backup?

Một backup tốt cần xem xét các failure domain.

Giả sử snapshot vẫn nằm trên cùng production environment:

```text
Production cluster
├── VM
└── Snapshot
```

Nếu failure ảnh hưởng cả environment hoặc data domain đó, snapshot có thể không đáp ứng DR objective.

Independent backup thường nhằm có protection khác failure domain:

```text
Production
    │
    ├── Local snapshot
    │
    └── Backup
          ↓
       Backup target
```

HYCU chính là phần Day 7 giúp bạn chuyển từ “có snapshot” sang **managed protection + target + policy + recovery verification**.

---

# 26. Snapshot cũng tiêu thụ resource và có lifecycle

Không nên:

```text
Create snapshot
→ quên
→ giữ vô thời hạn
```

Snapshot lifecycle phải có:

- purpose;
- owner;
- creation timestamp;
- retention expectation;
- cleanup.

Nếu snapshot được tạo phục vụ change:

```text
CHG000123
VM = APP01
Purpose = pre-change recovery
Expiry = after validated change
```

thì sau khi change hoàn tất phải theo runbook để xử lý.

---

# PHẦN IX — Cluster resource utilization và capacity management

---

# 27. Capacity là một operational discipline

Cluster capacity không chỉ gồm:

```text
Storage free %
```

Phải quan sát:

- CPU;
- memory;
- storage;
- IOPS/throughput/latency khi relevant;
- workload growth;
- node balance;
- operational headroom.

---

# 28. Current utilization vs trend

Hai cluster:

```text
Cluster A
Storage used = 70%
Growth = 0.2% / month
```

và:

```text
Cluster B
Storage used = 70%
Growth = 5% / week
```

cùng “70%” nhưng risk hoàn toàn khác.

Vì vậy syllabus yêu cầu xem **capacity trends**, không chỉ snapshot của current usage. Infrastructure_Operations_14_Da…

---

# 29. Capacity alert không phải chỉ để acknowledge

Giả sử alert:

```text
Storage utilization high
```

L1 acknowledgement:

```text
Alert acknowledged.
```

không giải quyết root cause.

Phải điều tra:

```text
Current utilization?
Growth rate?
Top consumers?
Snapshots?
New workloads?
Backup/recovery operations?
Capacity forecast?
Threshold?
Operational headroom?
```

Sau đó mới quyết định:

- clean up;
- reclaim;
- add capacity;
- move workload;
- escalate;
- schedule expansion.

---

# PHẦN X — CVM status và cluster service health

---

# 30. “Cluster is up” chưa đủ

Một health check tốt phải xem nhiều lớp:

```text
Cluster state
CVM state
Cluster service state
Data resiliency
Alerts
NCC
Hosts
Storage
VMs
Networking
```

Ví dụ:

```text
Prism accessible
```

chỉ chứng minh một management path đang hoạt động.

Không chứng minh:

```text
All CVMs healthy
All services healthy
Data resiliency healthy
All VMs available
```

---

# 31. Read-only command awareness

Trong authorized environment, các command thường được dùng cho health verification có thể gồm:

```bash
cluster status
```

và các service-related checks.

Nutanix KB hiện hành dùng `cluster status` để xác nhận các CVM tham gia cluster và xem state của services sau recovery scenario. [Nutanix Portal](https://portal.nutanix.com/page/documents/kbs/details?targetId=kA07V000000LXhTSAW&utm_source=chatgpt.com)

Tuy nhiên:

> Đọc trạng thái cluster là khác hoàn toàn với stop/start cluster services.

Day 7 phải luôn phân biệt **read-only inspection** và **configuration/disruptive action**.

---

# PHẦN XI — NCC: Nutanix Cluster Check

---

# 32. NCC là gì?

**NCC — Nutanix Cluster Check** là bộ health-check tooling của Nutanix được dùng để kiểm tra cluster condition và tìm các vấn đề/configuration findings.

Nutanix mô tả NCC là cluster-resident tooling giúp đánh giá cluster health và các configuration được Nutanix qualified/recommended. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=NCC-Guide-NCC-v4_6%3Ancc-cluster-ncc-upgrade-wc-t.html&utm_source=chatgpt.com)

Một command rất phổ biến là:

```bash
ncc health_checks run_all
```

NCC có thể được chạy qua Prism hoặc command line tùy version/workflow. Nutanix hiện khuyến nghị chạy NCC trước upgrade và xử lý các result ngoài trạng thái phù hợp trước khi tiếp tục. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=NCC-Guide-NCC-v5_1%3Ancc-ncc-checks-running-t.html&utm_source=chatgpt.com)

---

# 33. NCC không phải “fix button”

Sai:

```text
NCC says warning
→ execute random internet command
```

Đúng:

```text
NCC finding
    ↓
Identify check
    ↓
Read finding/details
    ↓
Map affected component
    ↓
Read correct KB / documentation
    ↓
Assess operational impact
    ↓
Correct if within runbook
    ↓
Rerun check
    ↓
Escalate if unresolved
```

---

# 34. PASS / INFO / WARN / FAIL phải được hiểu trong context

Không phải tất cả non-green finding đều có cùng severity.

Bạn phải xem:

- check name;
- condition;
- affected node/object;
- context;
- associated KB;
- maintenance relevance.

Một warning có thể chưa ảnh hưởng runtime nhưng có thể trở thành **maintenance blocker**.

Ví dụ:

```text
Cluster currently serving VMs normally
```

nhưng NCC finding liên quan resiliency khiến host maintenance không an toàn.

Đó là lý do pre-maintenance check quan trọng.

---

# 35. NCC evidence

Nếu NCC được dùng cho maintenance/change, evidence tốt nên có:

```text
NCC version
Cluster
Timestamp
Checks executed
Overall result
Failed/warning check IDs
Related KB/reference
Resolution
Post-fix rerun
```

Không nên chỉ lưu screenshot:

```text
“All checks complete.”
```

mà không có detailed result.

---

# PHẦN XII — AHV host maintenance

Đây là topic rủi ro cao hơn nhiều so với VM administration.

---

# 36. Maintenance mode dùng để làm gì?

Maintenance mode giúp đưa một AHV host ra khỏi normal workload scheduling/operation để thực hiện maintenance phù hợp.

Use cases:

- hardware maintenance;
- firmware activities;
- hypervisor lifecycle work;
- planned host intervention.

Host maintenance không được hiểu là:

```text
Click Maintenance Mode
→ xong
```

Mà là một full change workflow.

---

# 37. Pre-maintenance workflow

Một engineer phải kiểm tra:

```text
1. Change approved?
2. Maintenance window active?
3. Correct host?
4. Current cluster health?
5. Critical alerts?
6. NCC?
7. Data resiliency?
8. Capacity/headroom?
9. Workloads on host?
10. Non-migratable workloads?
11. Backup/recovery readiness?
12. Application owner notification?
13. Rollback/stop criteria?
14. Vendor issue?
```

Nutanix AHV guidance hiện hành yêu cầu xác minh cluster health và single-node failure tolerance trước những operation như restart host hoặc đưa host vào maintenance. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

---

# 38. Stop condition

Một operator chuyên nghiệp phải biết khi nào **không làm tiếp**.

Ví dụ:

```text
Critical cluster alert unresolved
→ STOP
```

```text
Data resiliency degraded
→ STOP
```

```text
Insufficient evacuation capacity
→ STOP
```

```text
NCC critical finding
→ STOP
```

```text
Maintenance window expired
→ STOP
```

```text
Unexpected second host problem
→ STOP / ESCALATE
```

Stop không phải failure.

Trong production operations:

> Dừng đúng lúc thường là hành vi an toàn hơn “cố hoàn thành change”.

---

# 39. Post-maintenance

Sau maintenance không được chỉ xác nhận:

```text
Host is up.
```

Phải xác nhận:

```text
Host state normal
Cluster healthy
CVM healthy
Cluster services healthy
Data resiliency healthy
No new critical alerts
VM workloads healthy
Network healthy
Capacity normal
Application checks pass
```

Một maintenance chỉ hoàn tất khi **service state** được xác nhận, không phải khi hardware đã bật lại.

---

# PHẦ XIII — Lifecycle Manager

---

# 40. LCM là gì?

**LCM — Lifecycle Manager** là công cụ Nutanix dùng để quản lý lifecycle của nhiều software/firmware components.

LCM hiện hỗ trợ các software/firmware update workflows trong cả connected và dark-site scenarios tùy environment/version. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Life-Cycle-Manager-Guide-v3_2%3Atop-lcm_updates_c.html&utm_source=chatgpt.com)

LCM có thể được truy cập từ Prism Central và, tùy workflow/version, Prism Element. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Life-Cycle-Manager-Guide-v3_1%3Atop-lcm-open-pc-t.html&utm_source=chatgpt.com)

---

# 41. LCM không phải chỉ là “Update All”

Operationally, lifecycle management gồm:

```text
Inventory
    ↓
Determine current versions
    ↓
Available updates
    ↓
Compatibility
    ↓
Dependencies
    ↓
Prechecks
    ↓
Change approval
    ↓
Execution
    ↓
Postchecks
```

Day 7 đặc biệt yêu cầu hiểu:

- firmware dependency;
- AOS dependency;
- AHV dependency;
- maintenance window;
- rollback limitations.

Infrastructure_Operations_14_Da…

---

# 42. Inventory

LCM cần biết current component state.

Mental model:

```text
Current inventory

AOS = version X
AHV = version Y
BIOS = version A
BMC/iLO = version B
NIC firmware = version C
Drive firmware = version D
LCM = version E
```

Đừng xây maintenance plan dựa trên memory.

Phải có inventory evidence.

---

# 43. Compatibility

Không được giả định:

```text
Newest firmware
+
Newest AOS
+
Newest AHV
=
supported
```

Enterprise platform thường có compatibility matrix.

Đúng hơn:

```text
Current stack
→ supported target stack
→ supported upgrade path
```

Do đó version dependency phải được xác minh từ vendor documentation cho đúng environment.

---

# 44. Upgrade sequencing

AOS/AHV/firmware có dependency.

Vì vậy:

```text
Upgrade cái nào trước cũng được
```

là cách tư duy sai.

Sequence phải do:

- supported upgrade path;
- LCM recommendation;
- Nutanix compatibility;
- vendor platform dependency;
- runbook;

quyết định.

---

# 45. Precheck

Precheck có mục tiêu phát hiện condition làm upgrade unsafe.

Ví dụ:

- unhealthy cluster;
- insufficient free space;
- dependency issue;
- non-migratable workload;
- component incompatibility;
- service problem.

Nutanix lifecycle/upgrade workflows thực hiện health và compatibility validation trước khi cho phép rolling upgrade đi tiếp. [Nutanix Portal](https://portal.nutanix.com/page/solutions/TN-2122-Nutanix-Upgrades%3Aupgrading-third-party-hypervisors-with-life-cycle-manager.html?utm_source=chatgpt.com)

---

# PHẦN XIV — Rollback limitations

Đây là một điểm syllabus rất muốn người học hiểu.

Day 6 đã đặt nền:

> Không được giả định snapshots hoặc firmware downgrade bảo đảm rollback. Infrastructure_Operations_14_Da…

Day 7 tiếp tục nguyên tắc đó với AOS/AHV/firmware maintenance.

---

# 46. “Có backup” không đồng nghĩa “change có rollback”

Ví dụ bạn upgrade:

```text
Firmware
```

Một VM snapshot không thể tự động rollback firmware.

Tương tự:

```text
AOS upgrade
```

không thể đơn giản được xử lý:

```text
Restore VM snapshot.
```

Hai layer khác nhau.

---

# 47. Backout vs rollback

Phân biệt:

### Rollback

Quay component về trạng thái/version trước.

### Backout

Đưa service trở về trạng thái operational bằng một phương án được định nghĩa.

Backout có thể không phải downgrade.

Ví dụ:

```text
Upgrade fails on one node
→ stop further rollout
→ engage Nutanix Support
→ recover affected node using supported procedure
```

Đó có thể là backout/recovery strategy thay vì “downgrade tất cả”.

---

# 48. Maintenance plan phải có rollback limitations

Một plan tốt ghi:

```text
Rollback:
Supported only until checkpoint X.

After firmware stage Y:
direct downgrade is not assumed.

If precheck fails:
do not start.

If node fails after upgrade:
stop rolling sequence and escalate to Nutanix Support.
```

Nó tốt hơn rất nhiều so với:

```text
Rollback: restore snapshot.
```

---

# PHẦN XV — RBAC và least privilege

---

# 49. RBAC là gì?

**RBAC — Role-Based Access Control** kiểm soát người dùng được làm gì dựa trên role và scope.

Trong Prism Central IAM hiện tại, Nutanix mô tả các thành phần:

```text
Identity
Entity
Operation
Role
Authorization Policy
```

Authorization Policy kết hợp identity, role và entity để enforce granular access. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Security-Guide-v7_6%3Amul-iam-introduction-c.html&utm_source=chatgpt.com)

---

# 50. Identity

Identity có thể đại diện:

- user;
- group;
- service account;

tùy environment.

Ví dụ:

```text
user: operator01
group: Nutanix-L1
```

---

# 51. Role

Role trả lời:

> Identity này được phép thực hiện operation nào?

Ví dụ conceptual:

```text
Viewer
VM Operator
VM Administrator
Infrastructure Administrator
```

Tên và quyền chính xác phải kiểm tra theo version.

Không nên tự suy đoán permission chỉ dựa trên role name.

---

# 52. Scope

Role mạnh nhưng scope nhỏ khác hoàn toàn role mạnh trên toàn environment.

Ví dụ:

```text
VM Admin
Scope = DEV project
```

khác:

```text
VM Admin
Scope = all production clusters
```

Least privilege phải xét:

```text
Privilege × Scope × Duration
```

chứ không chỉ role name.

---

# 53. Separation of duties

Không nên một account duy nhất mặc định có quyền:

```text
Create VM
Delete VM
Modify network
Modify storage
Perform cluster maintenance
Upgrade platform
Change RBAC
Delete audit evidence
```

Enterprise environment thường cần separation.

Ví dụ:

```text
L1
→ read / monitor / basic approved operations

L2
→ scoped corrective operations

Platform Admin
→ infrastructure changes

Security/IAM
→ access policies

Change approver
→ approval authority
```

---

# 54. Privileged access phải có audit trail

Khi thực hiện high-risk operation cần liên kết:

```text
User identity
Ticket
Approval
Timestamp
Target object
Action
Result
```

Nếu audit record chỉ cho biết:

```text
admin performed action
```

nhưng không biết admin là ai hoặc ticket nào, audit quality rất thấp.

---

# PHẦN XVI — Tasks, Events, Alerts và Audit

Đây là phần cực kỳ quan trọng của Nutanix troubleshooting.

---

# 55. Task

**Task** thường đại diện cho một operation có workflow/state.

Ví dụ:

```text
Create VM
Update VM
Snapshot VM
Migrate workload
Run cluster check
Upgrade component
```

Task có thể có:

```text
Started
Running
Succeeded
Failed
```

Task giúp trả lời:

> “Operation có thực sự hoàn thành không?”

---

# 56. Event

**Event** phản ánh một sự kiện xảy ra trong system.

Ví dụ conceptual:

```text
VM powered off
Host disconnected
Configuration changed
Service state changed
```

Event rất hữu ích khi dựng incident timeline.

---

# 57. Alert

**Alert** cho biết một condition cần attention.

Ví dụ:

```text
Storage capacity threshold
Hardware issue
Service degradation
Network issue
Health check finding
```

Alert thường cần:

```text
Triage
Ownership
Investigation
Resolution
```

---

# 58. Acknowledgement không phải resolution

Syllabus ghi rõ phải phân biệt hai trạng thái này. Infrastructure_Operations_14_Da…

Ví dụ:

```text
09:00 — Storage alert generated
09:05 — Operator acknowledged
```

không có nghĩa:

```text
09:05 — Storage problem solved
```

Acknowledgement có nghĩa:

> “Tôi/đội vận hành đã thấy hoặc nhận trách nhiệm xử lý.”

Resolution phải có evidence riêng.

---

# 59. Audit

Audit trả lời các câu hỏi:

```text
Who?
What?
When?
Where?
Which object?
What result?
```

Audit quan trọng cho:

- incident;
- change;
- security review;
- access review;
- compliance;
- root cause analysis.

---

# 60. Correlation

Engineer tốt không đọc một source riêng lẻ.

Ví dụ:

```text
13:02 alert: VM unreachable
13:01 task: NIC update
13:01 audit: operator X
12:58 change ticket starts
```

Khi correlate, bạn có hypothesis:

```text
Recent network change may be related.
```

Nhưng cần tránh:

```text
Correlation = proof.
```

Phải verify configuration.

---

# PHẦ XVII — Nutanix troubleshooting framework

Đây là phần biến knowledge thành operational reasoning.

---

# 61. Quy tắc đầu tiên: xác định scope

Ví dụ user nói:

> Application không truy cập được.

Đừng bắt đầu bằng restart.

Hãy hỏi:

```text
Một user hay tất cả?
Một VM hay nhiều VM?
Một application hay nhiều?
Một VLAN?
Một host?
Một cluster?
Một site?
```

Scope thu hẹp search space.

---

# 62. Establish baseline

Xác định trạng thái bình thường:

```text
VM should be running?
Expected CPU/memory?
Expected network?
Expected IP?
Expected host?
Expected application port?
Backup protection expected?
```

Nếu không biết expected state thì khó xác định abnormal state.

---

# 63. Check recent change

Một trong các câu quan trọng nhất:

> “Có gì thay đổi gần thời điểm incident?”

Kiểm tra:

- Tasks;
- Events;
- Audit;
- Change ticket;
- maintenance activity;
- LCM;
- network change;
- snapshot/restore;
- VM reconfiguration.

---

# 64. Layer-by-layer investigation

Một flow tổng quát:

```text
Application symptom
        ↓
Guest/application
        ↓
Guest network
        ↓
VM configuration
        ↓
AHV host
        ↓
AHV networking
        ↓
AOS/storage
        ↓
CVM/services
        ↓
Cluster
        ↓
Physical infrastructure
```

Không phải incident nào cũng cần đi đến cuối.

Bạn dừng khi đã chứng minh được failing layer.

---

# 65. Low-risk trước high-risk

Thứ tự đúng thường là:

```text
Observe
→ Query
→ Compare
→ Correlate
→ Test non-destructively
→ Correct lowest-risk cause
→ Verify
```

không phải:

```text
Restart
→ reboot
→ power cycle
→ hope
```

---

# 66. Troubleshooting example: VM unavailable

Một investigation có thể đi:

```text
VM exists?
↓
VM power state?
↓
Console accessible?
↓
Guest booted?
↓
NIC attached?
↓
IP configuration?
↓
Network reachability?
↓
Host healthy?
↓
Cluster alerts?
↓
Storage healthy?
↓
Recent task/config change?
```

Nếu VM Running + console healthy + NIC correct nhưng network unavailable:

```text
Likely network path investigation.
```

Nếu nhiều VMs trên cùng host cùng lỗi:

```text
Host/network/shared dependency becomes stronger hypothesis.
```

Nếu nhiều VMs toàn cluster cùng storage issue:

```text
AOS/storage/cluster issue becomes stronger hypothesis.
```

---

# PHẦ XVIII — Escalation boundary trong Nutanix operations

Day 7 không nhằm biến learner thành Nutanix Support engineer.

---

# 67. L1 nên làm gì?

Tùy organization, L1 thường có thể:

- inspect dashboard;
- verify VM state;
- inspect alarms;
- gather Tasks/Events;
- perform approved connectivity checks;
- collect evidence;
- follow known runbook;
- identify scope;
- escalate.

Không nên tự:

- stop cluster services;
- modify low-level storage configuration;
- force cluster operations;
- perform firmware upgrade;
- bypass NCC;
- alter unsupported configurations.

---

# 68. L2 có thể làm gì?

L2 thường có broader runbook scope:

- deeper diagnosis;
- approved VM changes;
- maintenance preparation;
- capacity analysis;
- standard corrective actions;
- NCC analysis;
- change implementation;
- vendor case preparation.

Nhưng vẫn phải escalate khi:

```text
Unknown cluster service failure
Data resiliency risk
Multiple nodes affected
Persistent NCC critical findings
Upgrade failure
Possible data corruption
Unsupported configuration
Hardware dependency
Security compromise
```

---

# PHẦ XIX — HYCU architecture

Nửa thứ hai của Day 7 là HYCU.

Syllabus bắt buộc cover:

- HYCU architecture;
- backup controller;
- source discovery;
- targets;
- protection policies;
- scheduling;
- retention;
- application awareness;
- monitoring;
- failure diagnosis;
- recovery;
- RPO/RTO;
- DR.

Infrastructure_Operations_14_Da…

HYCU hiện vẫn mô tả Nutanix protection theo mô hình policy-driven, automatic discovery, VM/application/file recovery và Nutanix-native integration. [HYCU](https://www.hycu.com/resources/datasheet-hycu-nutanix?utm_source=chatgpt.com)

---

# 69. Mental model của HYCU

Hãy hình dung:

```text
              HYCU
        Backup Controller
              │
       ┌──────┴───────┐
       │              │
       ▼              ▼
   Nutanix Source   Policy
       │              │
       └──────┬───────┘
              ↓
         Backup Job
              ↓
       Recovery Point
              ↓
        Backup Target
              ↓
           Restore
```

Đây là logic quan trọng hơn nhớ UI menu.

---

# 70. Backup Controller

**Backup Controller** là management/control component của HYCU environment.

Nó chịu trách nhiệm orchestration các workflow như:

- source registration/discovery;
- policy;
- jobs;
- monitoring;
- restore operations.

HYCU onboarding material hiện vẫn dùng thuật ngữ Backup Controller cho Nutanix deployment. [HYCU](https://www.hycu.com/customer-onboarding?utm_source=chatgpt.com)

---

# 71. Control plane vs backup data

Bạn nên phân biệt conceptual:

```text
Control plane
→ quyết định backup cái gì, khi nào, policy nào

Data protection path
→ dữ liệu được snapshot/copy/store theo workflow
```

Đây là distinction hữu ích trong troubleshooting.

Ví dụ UI/controller inaccessible không nhất thiết đồng nghĩa toàn bộ backup data đã mất.

Ngược lại controller healthy không chứng minh target có thể restore.

---

# PHẦ XX — Source discovery

---

# 72. Source là gì?

Trong context Day 7, **source** là Nutanix environment/workload mà HYCU cần protect.

Có thể là VM hoặc application, tùy protection scope/version.

Workflow:

```text
Register/Connect source
       ↓
Discover inventory
       ↓
Identify workloads
       ↓
Assign protection
```

HYCU hiện hỗ trợ automated workload/application discovery trong Nutanix-oriented protection workflows. [HYCU](https://www.hycu.com/resources/datasheet-hycu-nutanix?utm_source=chatgpt.com)

---

# 73. Discovery không đồng nghĩa protection

Đây là distinction cực kỳ quan trọng.

Có thể:

```text
HYCU sees VM
```

nhưng:

```text
VM has no policy.
```

Do đó:

```text
Discovered ≠ Protected
```

Tương tự:

```text
Policy assigned ≠ Recent successful recovery point
```

và:

```text
Recent recovery point ≠ Restore tested.
```

Hãy nhớ chain:

```text
Discovered
↓
Policy assigned
↓
Backup successful
↓
Recovery point available
↓
Restore successful
↓
Recovered data validated
```

Chỉ bước cuối mới cung cấp recovery assurance mạnh.

---

# PHẦ XXI — Backup target

---

# 74. Target là gì?

**Backup target** là destination dùng lưu protected data theo architecture/configuration được hỗ trợ.

Một target cần đánh giá nhiều hơn chỉ:

```text
Free capacity.
```

Phải xem:

- support compatibility;
- connectivity;
- credentials/access;
- capacity;
- performance;
- retention;
- immutability nếu áp dụng;
- encryption;
- recoverability.

Syllabus yêu cầu explicit validation về **HYCU target/support compatibility**. Infrastructure_Operations_14_Da…

---

# 75. Failure domain

Một good backup architecture phải hỏi:

> Nếu source infrastructure fail, target có fail cùng không?

Ví dụ:

```text
Source VM
      │
      └── backup copy
            nằm cùng failure domain
```

khả năng resilience thấp hơn:

```text
Source production environment
      │
      └──── protected backup
                 ↓
          separate target
```

Tùy architecture, có thể bổ sung:

- immutable storage;
- Object Lock/WORM;
- separate administrative domain;
- remote target.

HYCU hiện quảng bá immutable/WORM-capable backup designs trong Nutanix protection portfolio, nhưng implementation cụ thể phải dựa trên target và version thực tế. [HYCU](https://www.hycu.com/resources/datasheet-hycu-nutanix?utm_source=chatgpt.com)

---

# PHẦ XXII — Protection Policy

Đây là trái tim của policy-based backup.

---

# 76. Protection Policy trả lời điều gì?

Một policy phải giúp xác định:

```text
What?
→ workload nào

How often?
→ RPO / backup cadence

Where?
→ backup target

How long?
→ retention

How fast should recovery be?
→ RTO target

How consistent?
→ VM/application consistency

What additional copies?
→ nếu policy/version hỗ trợ
```

---

# 77. RPO trong policy

**RPO — Recovery Point Objective** trả lời:

> Có thể chấp nhận mất tối đa bao nhiêu dữ liệu tính theo thời gian?

Ví dụ:

```text
RPO = 4 hours
```

Nghĩa là organization đặt objective để recovery point không quá cũ so với tolerated data-loss window.

Không có nghĩa:

```text
Backup chạy đúng 4 tiếng một lần
```

trong mọi implementation.

RPO là business/recovery objective; scheduling là mechanism để đáp ứng objective.

---

# 78. RTO trong policy

**RTO — Recovery Time Objective** trả lời:

> Sau disruption, service cần được phục hồi trong khoảng thời gian mục tiêu bao lâu?

Ví dụ:

```text
RTO = 2 hours
```

Không có nghĩa backup software đã đảm bảo tuyệt đối production service sẽ trở lại trong 2 giờ.

RTO end-to-end còn phụ thuộc:

- detection;
- decision;
- operator access;
- target;
- network;
- restore throughput;
- guest startup;
- database recovery;
- application validation;
- dependencies.

---

# 79. Retention

**Retention** trả lời:

> Recovery point được giữ trong bao lâu?

Ví dụ:

```text
Backup every 4h
Retention 14d
```

hai con số trả lời hai câu hỏi khác nhau.

HYCU User Guide hiện có các policy field về RPO, RTO, retention và backup target configuration, nhưng chính xác options phải kiểm tra đúng installed version. [download.hycu.com](https://download.hycu.com/ec/v5.2.0/help/en/HYCU_UserGuide.pdf?utm_source=chatgpt.com)

---

# 80. RPO ≠ retention

Ví dụ:

```text
RPO = 4 hours
Retention = 30 days
```

nghĩa conceptual:

- cần recovery points đủ thường xuyên để đáp ứng tolerated data loss;
- giữ các protected recovery points theo policy trong 30 ngày.

Không phải:

```text
RPO = 30 days.
```

---

# PHẦ XXIII — Scheduling

---

# 81. Backup scheduling cần cân bằng nhiều yếu tố

Backup không chạy trong vacuum.

Nó tiêu thụ:

- source resources;
- storage resources;
- network;
- target throughput;
- concurrency slots;
- operational window.

Do đó schedule phải cân bằng:

```text
RPO
×
workload impact
×
backup window
×
target capacity
×
concurrency
×
business peak time
```

---

# 82. Backup window

Nếu workload peak:

```text
09:00–18:00
```

và backup nặng chạy:

```text
10:00
```

có thể tạo contention.

Nhưng chuyển backup ra:

```text
01:00
```

không tự động đúng nếu:

```text
RPO = 4 hours.
```

Do đó business requirement và technical schedule phải align.

---

# PHẦ XXIV — Application-aware protection

---

# 83. Crash-consistent vs application-consistent

Đây là concept cực kỳ quan trọng.

### Crash-consistent

Dữ liệu giống trạng thái machine sau một sudden power loss.

Filesystem/application có thể phải recovery khi restore.

### Application-consistent

Protection workflow phối hợp với application để dữ liệu ở trạng thái nhất quán phù hợp hơn khi backup.

Điều này đặc biệt quan trọng với:

- databases;
- transactional systems;
- messaging systems;
- directory/application services.

HYCU hiện tiếp tục nhấn mạnh application-consistent backup và automated application discovery trên Nutanix workloads. [download.hycu.com](https://download.hycu.com/ec/v3.0.0/help/en/HYCU_UserGuide.pdf?utm_source=chatgpt.com)

---

# 84. Tại sao VM backup successful vẫn có thể application restore fail?

Ví dụ database:

```text
VM disks captured
```

nhưng transaction vẫn đang in-flight.

Sau restore:

```text
OS boots
```

nhưng database có thể:

- perform recovery;
- have consistency issue;
- need log replay;
- fail application validation.

Do đó:

```text
VM recoverable ≠ application proven recoverable
```

---

# 85. Application credentials

Nếu backup product cần application-aware integration, credentials có thể cần thiết tùy workload/workflow.

Security principle:

- dùng dedicated account;
- minimum required privilege;
- protect secrets;
- rotate;
- không expose trong screenshots/logs;
- không hard-code credential.

Nếu application discovery fail, đừng lập tức tăng quyền admin toàn phần.

Kiểm tra:

```text
Connectivity
Credential validity
Privilege
Guest/application prerequisites
Version support
```

---

# PHẦ XXV — Backup monitoring

---

# 86. “Green dashboard” chưa đủ

Backup operations phải đánh giá:

```text
Protection coverage
Last successful job
Last recovery point
Job duration
Failure trend
Target capacity
Policy compliance
Restore test
```

Ví dụ:

```text
100 jobs succeeded
```

không giúp gì nếu critical VM:

```text
DB-PROD-01
```

không có policy.

---

# 87. Protection coverage

Một useful metric là:

```text
Protected workloads
-------------------
Total required workloads
```

Ví dụ:

```text
98 / 100 protected
```

98% nghe tốt.

Nhưng nếu 2 workload chưa protected là:

```text
Payment DB
Identity service
```

risk vẫn critical.

Do đó percentage phải kết hợp business criticality.

---

# PHẦ XXVI — Failed backup job diagnosis

Đây là skill bắt buộc Day 7.

---

# 88. Không retry mù quáng

Một lỗi backup:

```text
FAILED
```

không nên dẫn tới:

```text
Retry
Retry
Retry
Retry
```

Nếu cause là:

```text
Target full
```

retry chỉ tạo thêm noise.

---

# 89. Diagnostic hierarchy

Một failed backup nên được điều tra theo các nhóm:

```text
Source
↓
Controller
↓
Policy
↓
Connectivity
↓
Authentication
↓
Target
↓
Capacity
↓
Application consistency
↓
Platform dependency
```

---

# 90. Source-side failures

Hỏi:

```text
VM exists?
VM available?
Nutanix source reachable?
Object discovered?
Application state healthy?
Required snapshot operation works?
```

---

# 91. Target-side failures

Hỏi:

```text
Target reachable?
Credential valid?
Free capacity sufficient?
Target service healthy?
Object Lock/retention issue?
Latency/throughput problem?
```

Syllabus đặc biệt đề xuất các controlled failure như:

- unavailable target;
- insufficient target capacity.

Infrastructure_Operations_14_Da…

---

# 92. Authentication failure

Symptoms có thể bao gồm:

```text
Access denied
Unauthorized
Credential expired
Permission insufficient
```

Không khắc phục bằng:

```text
Give account full admin.
```

Đúng hơn:

```text
Identify required permission
Compare effective access
Fix least-privilege permission
Retest
```

---

# 93. Application consistency failure

Nếu infrastructure snapshot thành công nhưng application step fail:

```text
Backup job may fail/degrade
```

Điều tra:

- application service;
- credentials;
- integration requirement;
- guest communication;
- database state;
- timeout.

Không được bỏ application-aware processing mà không đánh giá recovery consequence.

---

# PHẦ XXVII — Recovery Point

---

# 94. Recovery Point là gì?

**Recovery Point** là trạng thái dữ liệu có thể chọn để restore.

Ví dụ:

```text
10:00
14:00
18:00
22:00
```

Nếu incident xảy ra:

```text
21:35
```

recovery point gần nhất hợp lệ có thể là:

```text
18:00
```

---

# 95. “Newest” không phải lúc nào cũng “đúng”

Giả sử malware xuất hiện lúc:

```text
18:30
```

Recovery points:

```text
18:00 clean
20:00 contaminated
22:00 contaminated
```

Chọn newest:

```text
22:00
```

sẽ restore malware state.

Recovery-point selection cần incident context.

---

# PHẦ XXVIII — VM-level và file-level recovery

---

# 96. Full VM restore

Dùng khi cần phục hồi full virtual machine hoặc khi scope failure rộng.

Có thể phục hồi:

- VM configuration;
- disks;
- workload state theo recovery point.

Nhưng full VM restore có operational risks:

- IP conflict;
- hostname conflict;
- identity conflict;
- duplicate application instance;
- unintended network activity.

---

# 97. File-level restore

Nếu user chỉ mất:

```text
/report/quarterly.xlsx
```

không nên mặc định restore cả server.

File-level recovery có thể:

- nhanh hơn;
- ít blast radius;
- ít disruption hơn.

Operational principle:

> Chọn minimum recovery scope đủ để giải quyết incident.

---

# PHẦ XXIX — Isolated restore

Syllabus yêu cầu restore vào isolated network để tránh identity/IP conflict. Infrastructure_Operations_14_Da…

Đây là một trong những nguyên tắc quan trọng nhất của Day 7.

---

# 98. Tại sao phải isolated?

Giả sử production VM:

```text
Hostname: AD01
IP: 10.10.10.10
```

Bạn restore clone cùng configuration trực tiếp vào production network.

Có thể tạo:

```text
duplicate IP
duplicate hostname
directory conflict
DNS conflict
application conflict
unexpected traffic
```

Vì vậy recovery verification thường nên dùng:

```text
Isolated recovery network
```

trước.

---

# 99. Isolated recovery flow

```text
Choose recovery point
        ↓
Restore workload
        ↓
Attach isolated network
        ↓
Boot
        ↓
Guest validation
        ↓
Application validation
        ↓
Data validation
        ↓
Record results
```

Chỉ khi recovery được xác minh mới xem xét cutover/promotion theo approved DR plan.

---

# PHẦ XXX — Recovery verification

Đây là nơi nhiều backup programs thất bại về mặt tư duy.

---

# 100. “Restore completed” chưa đủ

Có ít nhất bốn tầng thành công:

```text
Level 1
Backup job succeeded

Level 2
Recovery point exists

Level 3
Restore operation succeeded

Level 4
Recovered workload/data is usable
```

Day 7 muốn bạn đạt Level 4.

---

# 101. Infrastructure validation

Sau restore kiểm tra:

```text
VM powers on?
Disk present?
NIC correct?
Guest OS boots?
No critical boot error?
```

---

# 102. Application validation

Tiếp theo:

```text
Application service running?
Expected ports listening?
Database opens?
Application login works?
Dependencies reachable?
```

---

# 103. Data validation

Cuối cùng:

```text
Expected files exist?
File count correct?
Record count correct?
Known data readable?
Hash/checksum matches reference where appropriate?
```

Syllabus yêu cầu compare recovered content với source/reference và lưu recovery evidence. Infrastructure_Operations_14_Da…

---

# PHẦ XXXI — RPO sâu hơn

---

# 104. Planned RPO vs observed recovery point age

Giả sử:

```text
Incident time        = 15:00
Latest usable backup = 13:00
```

Observed recoverable data gap:

```text
15:00 - 13:00 = 2 hours
```

Nếu objective:

```text
RPO ≤ 4 hours
```

thì recovery point về mặt age phù hợp objective.

Nhưng nếu latest usable recovery point là:

```text
08:00
```

thì gap:

```text
7 hours
```

RPO objective bị miss.

---

# 105. RPO không phải backup frequency một cách máy móc

Ví dụ backup schedule mỗi 4 giờ nhưng một job fail:

```text
08:00 success
12:00 failed
16:00 incident
```

Recovery point gần nhất:

```text
08:00
```

Actual data-loss exposure:

```text
8 hours
```

Dù config nói:

```text
every 4h
```

thực tế RPO không đạt.

Do đó cần monitor **successful recovery points**, không chỉ policy definition.

---

# PHẦ XXXII — RTO sâu hơn

---

# 106. RTO end-to-end

RTO không chỉ tính từ lúc bấm Restore.

Một realistic recovery timeline:

```text
10:00 Incident detected
10:08 Incident confirmed
10:15 Recovery approved
10:20 Restore started
10:40 VM restored
10:48 OS booted
10:55 Database recovered
11:05 Application validated
```

Nếu business RTO tính từ disruption:

```text
10:00 → 11:05 = 65 minutes
```

không phải:

```text
10:20 → 10:40 = 20 minutes.
```

---

# 107. Observed recovery time

Day 7 yêu cầu đo observed recovery time.

Cần xác định rõ measurement boundaries.

Ví dụ lab có thể định nghĩa:

```text
Recovery Start:
operator clicks approved Restore action

Recovery End:
restored application/data passes validation
```

Sau đó:

```text
Observed recovery time =
Recovery End - Recovery Start
```

Phải ghi rõ definition để con số có ý nghĩa.

---

# PHẦ XXXIII — Disaster Recovery

---

# 108. Backup và DR không giống nhau

**Backup** tập trung bảo vệ/recover data.

**Disaster Recovery** tập trung khôi phục khả năng cung cấp service sau disruption lớn.

Một organization có thể có nhiều backups nhưng DR yếu nếu:

- target unavailable;
- credential mất;
- runbook lỗi;
- network chưa chuẩn bị;
- DNS dependency chưa biết;
- application dependencies thiếu;
- restore chưa từng test.

---

# 109. DR dependency map

Một application thường không phải chỉ một VM.

Ví dụ:

```text
Web Server
    ↓
Application Server
    ↓
Database
    ↓
Identity
    ↓
DNS
```

Restore Web VM trước không có nghĩa service usable.

Do đó recovery phải hiểu application dependency.

---

# 110. DR workflow

Một simplified operational process:

```text
Incident
   ↓
Assess
   ↓
Determine recovery scope
   ↓
Authorization / declaration
   ↓
Identify clean recovery point
   ↓
Verify target/dependencies
   ↓
Restore isolated
   ↓
Validate infrastructure
   ↓
Validate application
   ↓
Validate data
   ↓
Promote/cutover if approved
   ↓
Monitor
   ↓
Document
```

---

# PHẦ XXXIV — Snapshot vs HYCU backup: distinction bắt buộc phải thuộc

Hãy ghi nhớ bảng sau.

| Aspect                      | VM Snapshot                      | Independent HYCU protection               |
| --------------------------- | -------------------------------- | ----------------------------------------- |
| Primary use                 | Point-in-time operational state  | Managed data protection/recovery          |
| Policy                      | Có thể limited/platform-specific | Protection policy                         |
| Schedule                    | Có thể có                        | Policy-driven                             |
| Retention                   | Snapshot lifecycle               | Backup retention                          |
| Target                      | Có thể cùng platform             | Có thể dùng independent supported target  |
| Failure-domain independence | Không nên mặc định               | Có thể thiết kế independent               |
| Application awareness       | Phụ thuộc mechanism              | HYCU supports application-aware workflows |
| File recovery               | Không phải mục đích chung        | Có thể hỗ trợ granular recovery           |
| Monitoring                  | Platform-specific                | Backup-job monitoring                     |
| Recovery assurance          | Chưa đủ                          | Cần restore + validation                  |
| DR proof                    | Không                            | Cũng không, trừ khi recovery được test    |

Nói ngắn gọn:

```text
Snapshot = recovery mechanism/component.

Backup = data protection system.

DR = business/service recovery capability.
```

---

# PHẦ XXXV — Security implications của HYCU

---

# 111. Backup infrastructure là security-critical

Backup chứa:

- server data;
- databases;
- identity data;
- configuration;
- potentially sensitive files.

Do đó compromise backup platform có thể nghiêm trọng ngang production compromise.

---

# 112. Least privilege

Backup operator không nhất thiết cần:

```text
Global infrastructure admin
+
domain admin
+
storage admin
+
security admin
```

Phải dùng minimum rights theo integration requirements.

---

# 113. Credential isolation

Không:

```text
export screenshots with passwords
store backup admin password in Excel
hard-code credential in script
paste access token into ticket
```

Evidence phải mask secret.

---

# 114. Immutability

**Immutable backup** là backup mà attacker/operator không thể dễ dàng sửa/xóa trước retention constraints theo mechanism được cấu hình.

Nó hữu ích trước:

- ransomware;
- malicious deletion;
- compromised administrative account.

HYCU hiện hỗ trợ integrations cho immutable/Object Lock/WORM-style protection trên một số supported targets. [HYCU](https://www.hycu.com/resources/datasheet-hycu-nutanix?utm_source=chatgpt.com)

Nhưng phải nhớ:

```text
Immutability
≠
complete security.
```

Vẫn cần:

- credential security;
- MFA;
- network isolation;
- monitoring;
- restore tests.

---

# PHẦ XXXVI — Evidence model của Day 7

Syllabus Day 7 yêu cầu rất cụ thể evidence:

- Prism tasks/audit references;
- pre/post health;
- stop/escalation decisions;
- HYCU job ID;
- source;
- target;
- recovery point;
- failure cause;
- restore logs;
- content validation;
- measured recovery time.

Infrastructure_Operations_14_Da…

---

# 115. Nutanix evidence

Một Nutanix operational record tốt:

```text
Evidence ID: D7-NTNX-001

Product:
Nutanix AOS/AHV

Cluster:
LAB-NTNX-01

Timestamp:
2026-10-21T10:30:00+07:00

Operation:
VM health verification

VM:
LAB-RHEL-01

Task reference:
<task-id>

Alert state:
No critical alerts

Cluster health:
Healthy

NCC reference:
<ncc-output-id>

Ticket:
CHG-LAB-007
```

---

# 116. HYCU evidence

```text
Evidence ID: D7-HYCU-005

Job ID:
<job-id>

Source:
LAB-RHEL-01

Policy:
LAB-GOLD

Target:
LAB-BACKUP-TARGET

Start:
10:00

End:
10:17

Result:
Success

Recovery Point:
<recovery-point-id>

Restore Test:
Passed

Restore Start:
11:00

Validation End:
11:14

Observed Recovery Time:
14 minutes
```

---

# 117. Failed-job evidence

Ví dụ:

```text
Job ID:
HYCU-JOB-102

Expected:
Backup succeeds

Actual:
Failed

Error:
Target unavailable

Root Cause:
Backup target connectivity unavailable

Correction:
Approved connectivity restored

Retest Job:
HYCU-JOB-103

Retest:
Passed
```

Điều quan trọng:

> Không xóa evidence của failure sau khi fix.

Failure → investigation → correction → retest chính là audit trail giá trị.

---

# PHẦ XXXVII — Common failure modes Day 7

Bạn cần có khả năng nhận diện các nhóm failure sau.

| Symptom                         | Possible layer                          |
| ------------------------------- | --------------------------------------- |
| VM cannot boot                  | VM/guest/storage                        |
| VM boots but no network         | Guest/vNIC/VLAN/upstream                |
| Many VMs on one host affected   | Host/shared dependency                  |
| Cluster storage alert           | Capacity/AOS/storage                    |
| CVM/service issue               | Platform                                |
| Maintenance cannot proceed      | Capacity/health/workload constraint     |
| NCC failure                     | Cluster/configuration/hardware/software |
| LCM precheck fails              | Health/compatibility/dependency         |
| Backup job fails                | Source/controller/policy/target         |
| Application backup fails        | Application consistency/integration     |
| Restore succeeds but app fails  | Guest/application/data consistency      |
| Restored VM conflicts with prod | Recovery-network design                 |
| RPO missed                      | Successful recovery-point cadence       |
| RTO missed                      | Restore/dependency/validation duration  |

---

# PHẦ XXXVIII — Operational decision tree tích hợp Nutanix + HYCU

Hãy xem một scenario tổng hợp:

> User báo `APP01` unavailable. Prism cho thấy VM có vấn đề và backup đêm qua cũng failed.

Một Infrastructure Operations engineer Day 7 nên suy luận:

```text
START
 │
 ├─ Determine scope
 │
 ├─ Check VM state
 │     │
 │     ├─ OFF → Why?
 │     │        Task? Audit? Host event?
 │     │
 │     └─ ON → Console?
 │
 ├─ Guest healthy?
 │
 ├─ vNIC/network correct?
 │
 ├─ Host healthy?
 │
 ├─ Cluster alerts?
 │
 ├─ CVM/services healthy?
 │
 ├─ Capacity?
 │
 ├─ NCC relevant?
 │
 └─ Recent changes?
```

Song song với recovery readiness:

```text
HYCU
 │
 ├─ Workload discovered?
 │
 ├─ Policy assigned?
 │
 ├─ Last successful job?
 │
 ├─ Why latest job failed?
 │
 ├─ Target available?
 │
 ├─ Valid recovery point?
 │
 └─ Recovery tested?
```

Sau đó mới ra quyết định:

```text
Repair current VM
       or
Recover from backup
       or
Escalate
```

Đây chính là mindset Day 7 muốn xây dựng.

---

# PHẦ XXXIX — Những điều Day 7 tuyệt đối không được hiểu sai

### 1. VM Running ≠ Application Healthy

```text
VM green
```

không chứng minh application usable.

### 2. Cluster accessible ≠ Cluster fully healthy

Prism login được không chứng minh toàn bộ distributed services healthy.

### 3. Alert acknowledged ≠ Alert resolved

Syllabus explicitly yêu cầu distinction này. Infrastructure_Operations_14_Da…

### 4. Snapshot ≠ Backup

Đây là core lesson.

### 5. Backup Successful ≠ Restore Successful

Phải test.

### 6. Restore Successful ≠ Application Healthy

Phải validate.

### 7. Backup available ≠ DR Ready

Phải có runbook, dependency và tested recovery.

### 8. Current cluster healthy ≠ Safe for maintenance

Phải check headroom/resiliency/NCC/workload constraints.

### 9. Upgrade available ≠ Upgrade approved

Còn compatibility, change, maintenance window, precheck.

### 10. Upgrade success ≠ Automatic rollback exists

Rollback limitations phải được biết trước.

### 11. Admin access ≠ Authorized access

RBAC và ticket approval vẫn áp dụng.

### 12. Retry backup ≠ Troubleshooting

Phải tìm root cause.

---

# PHẦ XL — Mental model hoàn chỉnh Day 7

Nếu chỉ ghi nhớ một diagram, hãy ghi nhớ diagram này:

```text
┌──────────────────────────────────────────────┐
│               MANAGEMENT PLANE               │
│                                              │
│   Prism Central            Prism Element     │
│   Multi-cluster            Cluster scope     │
│                                              │
│   RBAC / Tasks / Events / Alerts / Audit     │
└───────────────────────┬──────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────┐
│               NUTANIX CLUSTER                │
│                                              │
│     AHV Host     AHV Host     AHV Host       │
│        │            │            │           │
│       CVM          CVM          CVM           │
│        └────── AOS Services ─────┘           │
│                                              │
│          Distributed Storage                 │
│                                              │
│       VM     VM     VM     VM                │
└───────────────────────┬──────────────────────┘
                        │
           Compute / Storage / Network
                        │
                        ▼
                 Guest Workload
                        │
                        ▼
                  Application
```

Protection side:

```text
Application / VM
       │
       ▼
Nutanix Source
       │
       ▼
HYCU Discovery
       │
       ▼
Protection Policy
 RPO / RTO / Retention
       │
       ▼
Backup Job
       │
       ▼
Recovery Point
       │
       ▼
Backup Target
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
Data Validation
       │
       ▼
Measured Recovery Time
```

Operational control bao quanh tất cả:

```text
Authorization
      ↓
Least Privilege
      ↓
Pre-check
      ↓
Operation
      ↓
Verification
      ↓
Evidence
      ↓
Rollback / Recovery
      ↓
Escalation
```

---

# Day 7 — Module Lý thuyết: chuẩn đầu ra

Sau khi nắm chắc toàn bộ module này, bạn phải tự giải thích được những câu sau mà không học thuộc máy móc:

**Architecture:** AOS, AHV, CVM, host, cluster và guest VM liên hệ như thế nào?

**Management:** Prism Element khác Prism Central ở management scope nào?

**VM:** Một AHV VM phụ thuộc CPU, memory, vDisk, storage container, vNIC và VLAN như thế nào?

**Networking:** Vì sao VM Running vẫn có thể mất connectivity?

**Placement:** Vì sao host maintenance cần capacity/headroom assessment?

**Snapshot:** Vì sao snapshot không thể được xem mặc định là independent backup?

**Health:** Cluster health cần đánh giá qua alerts, CVM/services, resiliency, capacity và NCC như thế nào?

**NCC:** NCC giúp gì, và vì sao NCC finding không nên được xử lý bằng random fix?

**Maintenance:** Các precheck, stop condition và postcheck của AHV host maintenance là gì?

**LCM:** Tại sao firmware/AOS/AHV upgrade phải xem compatibility và dependency?

**Rollback:** Tại sao snapshot không đảm bảo rollback cho platform/firmware upgrade?

**Security:** RBAC, scope và least privilege áp dụng thế nào cho Prism operations?

**Observability:** Task, Event, Alert và Audit khác nhau ở điểm nào?

**HYCU:** Source, Backup Controller, Target, Policy, Job và Recovery Point liên hệ thế nào?

**Policy:** RPO, RTO, schedule và retention khác nhau thế nào?

**Consistency:** Crash-consistent và application-consistent khác nhau ra sao?

**Backup troubleshooting:** Failed job cần kiểm tra source → policy → connectivity → target → capacity → application dependency như thế nào?

**Recovery:** Khi nào dùng VM restore, khi nào file restore?

**Isolation:** Vì sao restore validation nên vào isolated network?

**Assurance:** Vì sao `Backup Success` không đủ chứng minh DR readiness?

**RPO/RTO:** Làm sao phân biệt configured objective với observed recovery result?

**Evidence:** Một maintenance/backup/restore operation cần lưu Task ID, Job ID, timestamps, recovery point, health evidence và validation evidence gì?

**Escalation:** Khi nào L1/L2 phải dừng và escalate thay vì tiếp tục corrective action?

Nếu bạn thực sự hiểu và giải thích được toàn bộ các câu trên theo dependency và operational workflow, thì phần **Module Lý thuyết Day 7** theo syllabus đã được cover đầy đủ: AOS/AHV administration, Prism Element/Prism Central, VM lifecycle, compute/storage/network allocation, VLAN, power/console, placement/migration, snapshots, health/capacity, CVM/services, NCC, LCM/upgrades, maintenance/rollback limitations, RBAC, tasks/events/audit/troubleshooting và toàn bộ HYCU protection → monitoring → failure diagnosis → recovery → RPO/RTO → DR. Infrastructure_Operations_14_Da…

Bước tiếp theo của Day 7 là **Module Thực hành**, nơi toàn bộ lý thuyết trên sẽ được chuyển thành một chuỗi lab enterprise-style: **AHV VM provisioning → VLAN/connectivity verification → Prism health/evidence → controlled snapshot/power operations → simulated capacity/availability incident → AHV host-maintenance rehearsal → HYCU policy/backup → failure injection → failed-job diagnosis → isolated VM/file restore → content validation → measured recovery time → rollback/escalation evidence package**.
