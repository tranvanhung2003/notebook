Dưới đây là **bản đồ nội dung Day 6** được chia thành đúng hai module: **Module 1 — Lý thuyết** và **Module 2 — Thực hành**. Tôi bám sát toàn bộ `Concept/Lecture`, `Assignment/Lab`, phần `Additive skill coverage` và `Acceptance/Evidence` của Day 6 trong syllabus; ở bước này tôi chỉ tổ chức lại để bạn thấy rõ phạm vi, chưa đi sâu giảng từng kỹ thuật. Infrastructure_Operations_14_Da…

# Day 6 — HPE ProLiant / iLO + Nutanix HCI / AOS / AHV / Prism

Day 6 là ngày chuyển từ server/cloud fundamentals của Day 1–5 sang **enterprise physical infrastructure và hyperconverged infrastructure**. Hai hệ sinh thái trọng tâm là:

- **HPE ProLiant + HPE iLO**: nhìn từ lớp physical hardware và out-of-band management.
- **Nutanix HCI + AOS + AHV + Prism Element/Prism Central**: nhìn từ lớp cluster, hypervisor, distributed storage và management plane.

Mục tiêu không phải trở thành hardware engineer hay Nutanix expert ngay trong một ngày, mà đạt mức **operational readiness**: biết xác định component, kiểm tra health, đọc alert/log, thiết lập baseline, xác định đúng failure layer, thu thập evidence, nhận biết maintenance risk và biết khi nào cần escalate. Syllabus cũng nhấn mạnh Day 6–7 là cụm kiến thức chung cho **ProLiant, iLO, AOS, AHV, Prism, Prism Central và HYCU**, trong đó Day 6 tập trung vào infrastructure architecture, inventory, health và operational boundaries; Day 7 mới đi sâu hơn vào VM operations, host maintenance, LCM và HYCU backup. Infrastructure_Operations_14_Da…

---

# Module 1 — LÝ THUYẾT

## 1. Physical Server Fundamentals

Phần đầu tiên cần hiểu bản chất của một **physical server** trong môi trường enterprise.

Bạn cần học:

- physical server khác desktop/workstation ở đâu;
- vai trò của:
  - CPU;
  - physical memory;
  - DIMM;
  - system board;
  - power supply;
  - cooling/fan;
  - storage device;
  - NIC;
  - PCIe device;
  - storage controller;
- dependency giữa các component;
- một lỗi physical hardware có thể biểu hiện lên:
  - Operating System;
  - hypervisor;
  - VM;
  - application;
  - monitoring platform
    như thế nào.

Trọng tâm vận hành là học cách suy luận:

**Hardware → Hypervisor → VM → Application**

thay vì thấy application lỗi là lập tức restart service.

Syllabus đặt phần này dưới nhóm **physical server and hyperconverged infrastructure fundamentals**. Infrastructure_Operations_14_Da…

---

## 2. Hyperconverged Infrastructure — HCI Fundamentals

Bạn cần hiểu **HCI — Hyperconverged Infrastructure** là gì và khác mô hình truyền thống như thế nào.

Sẽ cần nắm:

- traditional infrastructure:
  - compute;
  - SAN/storage;
  - network;
  - hypervisor;
- HCI:
  - compute + storage resources nằm trên các cluster nodes;
  - software-defined storage;
  - distributed services;
  - scale-out cluster;
- tại sao failure của một node không nhất thiết đồng nghĩa toàn cluster failure;
- resiliency và redundancy là gì;
- tại sao phải kiểm tra capacity/resiliency trước maintenance.

Điểm quan trọng cho Infrastructure Operations là:

> Không nhìn Nutanix như “một server chạy VM”, mà phải nhìn nó như một distributed platform gồm nhiều layer phụ thuộc lẫn nhau.

---

# 3. HPE ProLiant Server Architecture

Đây là phần hardware chính của Day 6.

Bạn cần nhận diện được ít nhất các nhóm component:

### Compute

- CPU;
- CPU socket;
- cores;
- memory channels;
- DIMM population.

### Memory

- installed memory;
- DIMM health;
- ECC awareness;
- failed/degraded DIMM;
- memory population rules awareness.

### Storage

- HDD;
- SSD;
- NVMe;
- drive bays;
- backplane;
- logical drive;
- physical drive.

### Storage Controller

Syllabus yêu cầu **RAID and storage controller awareness**.

Bạn cần hiểu cơ bản:

- RAID controller làm gì;
- physical disk và logical volume khác nhau thế nào;
- RAID 0/1/5/6/10 về mặt operational concept;
- degraded array;
- failed drive;
- rebuilding;
- controller/cache/battery hoặc capacitor health awareness.

Không yêu cầu biến Day 6 thành storage engineering chuyên sâu, nhưng phải đủ để không đưa ra quyết định nguy hiểm khi thấy một disk failure.

---

# 4. Server Firmware

Một server enterprise có nhiều firmware khác nhau, không chỉ một firmware duy nhất.

Cần nhận biết các nhóm như:

- system ROM/BIOS/UEFI;
- iLO firmware;
- storage controller firmware;
- NIC firmware;
- drive firmware;
- platform firmware packages.

Phải hiểu:

**Firmware update ≠ software package update thông thường.**

Firmware update có thể:

- cần reboot;
- ảnh hưởng hardware initialization;
- có compatibility dependency;
- ảnh hưởng cluster node;
- có downtime;
- có rollback limitation.

Syllabus yêu cầu cả:

- firmware version inspection;
- firmware update precautions;
- supported firmware/software combinations;
- maintenance planning. Infrastructure_Operations_14_Da…

---

# 5. BIOS / UEFI

Cần hiểu BIOS/UEFI ở mức Infrastructure Operations:

- hardware initialization;
- boot configuration;
- boot order;
- virtualization settings awareness;
- CPU/memory related configuration;
- firmware settings baseline;
- operational risks của việc thay đổi BIOS/UEFI.

Không phải mọi warning đều cần thay đổi BIOS.

Trong enterprise, thay đổi BIOS/UEFI thường cần:

**approved change + maintenance window + baseline + recovery plan + post-check.**

---

# 6. Hardware Health Indicators

Đây là phần rất quan trọng.

Bạn cần biết cách đánh giá:

- overall server health;
- CPU health;
- memory health;
- storage health;
- RAID/controller state;
- disk health;
- fan health;
- temperature;
- power supply;
- voltage;
- NIC;
- hardware sensors.

Cần phân biệt trạng thái kiểu:

- Healthy;
- OK;
- Warning;
- Degraded;
- Critical;
- Failed.

Điểm cần học sâu sau này là:

> Alert có ý nghĩa gì, component nào tạo alert, failure domain nào bị ảnh hưởng, và workload có đang chịu impact hay chưa.

---

# 7. Hardware Maintenance

Day 6 không chỉ đọc dashboard.

Bạn phải hiểu maintenance workflow cơ bản:

**Alert → Validate → Assess Impact → Check Redundancy → Collect Evidence → Change Approval → Maintenance → Verification → Closure/Escalation**

Các hoạt động có thể liên quan:

- replace disk;
- replace DIMM;
- replace PSU;
- replace fan;
- update firmware;
- reboot physical server;
- hardware diagnostics.

Tuy nhiên syllabus quy định physical replacement và firmware operations có thể chỉ được **demonstration**, thay vì cho learner tự thực hiện trực tiếp. Infrastructure_Operations_14_Da…

---

# 8. HPE Integrated Lights-Out — iLO

Đây là một topic bắt buộc của Day 6.

**iLO — Integrated Lights-Out** là out-of-band management interface của HPE ProLiant.

Bạn cần hiểu sự khác biệt giữa:

### In-band management

Quản lý server thông qua OS/hypervisor.

Ví dụ:

```text
SSH → Linux
RDP → Windows
Hypervisor management
```

### Out-of-band management

Quản lý trực tiếp management controller của server.

Ví dụ:

```text
Administrator
    |
    v
iLO Management Network
    |
    v
Physical Server Hardware
```

iLO vẫn có thể hữu ích ngay cả khi Operating System không hoạt động bình thường.

---

# 9. iLO Remote Management

Các capability cần hiểu theo syllabus:

- remote management;
- remote console;
- virtual media;
- power controls;
- hardware inventory;
- health monitoring;
- firmware inspection;
- log inspection.

---

# 10. Remote Console

Bạn cần hiểu mục đích của **remote console**:

- quan sát boot process;
- kiểm tra POST;
- xem BIOS/UEFI;
- xử lý OS không reachable qua network;
- hỗ trợ installation/recovery.

Điểm operational quan trọng:

Remote console là privileged capability.

Không nên coi nó tương đương việc mở một terminal thông thường.

---

# 11. Virtual Media

**Virtual media** cho phép attach media từ xa cho server.

Ví dụ:

```text
ISO image
   |
   v
iLO Virtual Media
   |
   v
Physical Server
```

Có thể dùng trong:

- OS installation;
- recovery;
- diagnostics;
- firmware workflows.

Nhưng cũng cần hiểu security implications vì người có quyền virtual media có thể tác động trực tiếp tới boot/recovery path.

---

# 12. Power Controls

iLO có thể cung cấp các power operations.

Bạn sẽ cần phân biệt về mặt operational:

- power status inspection;
- graceful operation;
- reset;
- power off;
- power on.

Điểm quan trọng nhất:

> Khả năng bấm Power Off không có nghĩa là Infrastructure Operations được phép bấm Power Off.

Phải xem:

- workload;
- VM placement;
- cluster state;
- approval;
- maintenance window;
- redundancy;
- stop conditions.

---

# 13. Integrated Management Log — IML

Một phần bắt buộc là **IML — Integrated Management Log**.

Bạn cần hiểu IML dùng để ghi nhận hardware/platform events như:

- disk warning;
- memory event;
- temperature;
- fan;
- PSU;
- controller;
- hardware status transition.

Bạn cần học cách đọc một IML entry theo logic:

```text
Timestamp
↓
Component
↓
Severity
↓
Event
↓
Current state
↓
Potential impact
↓
Required action
```

IML sau đó phải được correlation với Nutanix/Prism evidence trong lab.

---

# 14. Hardware Alert Interpretation

Không được chỉ đọc chữ `Critical`.

Bạn phải học suy luận:

**Alert → Component → Failure Layer → Redundancy → Workload Impact → Correct Owner**

Ví dụ:

```text
Physical drive failure
        ↓
RAID / local storage layer
        ↓
Nutanix node
        ↓
Cluster resilience
        ↓
VM / workload risk
```

Từ đó mới quyết định:

- continue monitoring;
- planned maintenance;
- urgent hardware intervention;
- vendor escalation.

---

# 15. Hardware Escalation

Syllabus nhấn mạnh **hardware escalation**.

Bạn cần biết khi nào Infrastructure Operations có thể:

- verify;
- collect logs;
- collect inventory;
- inspect health;
- confirm failure;
- create/route ticket.

Và khi nào cần escalate:

- hardware replacement;
- uncertain failure;
- firmware compatibility issue;
- repeated component failure;
- insufficient cluster resilience;
- vendor-specific hardware fault;
- maintenance vượt quyền được cấp.

---

# 16. Nutanix HCI Architecture

Sau HPE là phần lớn thứ hai của Day 6: **Nutanix architecture**.

Bạn cần xây dựng mental model:

```text
Nutanix Cluster
│
├── Node 1
│   ├── AHV
│   ├── CVM
│   └── Local Storage
│
├── Node 2
│   ├── AHV
│   ├── CVM
│   └── Local Storage
│
└── Node N
    ├── AHV
    ├── CVM
    └── Local Storage
```

Đây là nền tảng để hiểu Day 7.

---

# 17. Nutanix Cluster Nodes

Bạn cần hiểu **node** là gì và một node có những dependency nào.

Cần biết:

- physical node;
- hardware resources;
- AHV host;
- CVM;
- storage contribution;
- network connectivity;
- cluster membership.

Một node warning không đồng nghĩa ngay với cluster outage.

Phải kiểm tra toàn bộ cluster state.

---

# 18. Controller VM — CVM

**CVM — Controller VM** là component rất quan trọng.

Trong Day 6 cần nắm ở mức architecture:

- mỗi Nutanix node thường gắn với một CVM trong thiết kế Nutanix;
- CVM liên quan tới distributed platform/storage services;
- CVM không phải một workload VM thông thường;
- CVM service health khác với AHV host health.

Đây là một trong những separation quan trọng nhất của syllabus.

---

# 19. AOS — Acropolis Operating System

Bạn phải hiểu **AOS** đại diện cho platform software/services của Nutanix.

Các khái niệm liên quan:

- cluster services;
- distributed storage;
- data resiliency;
- storage management;
- cluster health;
- version awareness.

Day 6 tập trung architecture và health.

Day 7 mới đi sâu hơn operational administration.

---

# 20. AHV — Acropolis Hypervisor

**AHV** là hypervisor layer.

Bạn cần phân biệt:

```text
Physical HPE/Nutanix node
        ↓
AHV
        ↓
Virtual Machines
```

và:

```text
CVM
↓
Nutanix platform/storage services
```

AHV host failure và CVM/service failure là hai loại vấn đề khác nhau.

---

# 21. Prism Element

Bạn cần hiểu **Prism Element** ở phạm vi:

- cluster-local management;
- nodes;
- hosts;
- VMs;
- storage;
- alerts;
- events;
- health;
- capacity;
- cluster operations.

---

# 22. Prism Central

Bạn cần phân biệt **Prism Central** với **Prism Element**.

Ở mức Day 6, cần hiểu:

```text
Prism Element
→ quản lý/quan sát cluster cụ thể

Prism Central
→ centralized / multi-cluster management scope
```

Không được mặc định hai interface có cùng scope hoặc cùng chức năng.

---

# 23. Management Plane Separation

Đây là phần additive rất quan trọng.

Phải phân biệt rõ bốn layer:

```text
iLO
↓
Out-of-band physical hardware management

AHV
↓
Hypervisor / compute layer

CVM + AOS
↓
Nutanix distributed platform/services

Prism Element / Prism Central
↓
Management and visibility planes
```

Syllabus yêu cầu learner phải **separate out-of-band iLO management, AHV host operations, CVM services and Prism management planes**. Infrastructure_Operations_14_Da…

Nếu không phân biệt được bốn layer này, troubleshooting Nutanix rất dễ sai hướng.

---

# 24. Cluster Discovery

Bạn cần hiểu cách một Infrastructure Operations engineer nhận biết environment:

- cluster name;
- cluster nodes;
- AHV hosts;
- CVMs;
- management addresses;
- Prism Element;
- Prism Central;
- software versions;
- hardware models.

Mục tiêu là tạo **configuration/inventory baseline**.

---

# 25. Storage Pools

Bạn cần hiểu khái niệm **storage pool**:

- tập hợp storage resources;
- cluster storage capacity;
- relation với distributed storage.

Không cần thiết kế Nutanix storage chuyên sâu ngay Day 6, nhưng phải biết vị trí của object trong architecture.

---

# 26. Storage Containers

Bạn cần phân biệt:

```text
Physical Disks
→ Storage Pool
→ Container
→ VM Storage
```

Container là một object quan trọng sẽ tiếp tục xuất hiện ở Day 7.

---

# 27. Storage Replication and Resilience Concepts

Syllabus yêu cầu **storage replication and resilience concepts**.

Bạn phải hiểu:

- replicated/distributed data;
- redundancy;
- failure tolerance;
- degraded resilience;
- impact của disk/node failure;
- vì sao cluster có thể vẫn hoạt động khi có component failure.

Điểm cực kỳ quan trọng:

> “VM vẫn chạy” không đồng nghĩa “cluster hoàn toàn healthy”.

Cluster có thể đang chạy nhưng resilience đã giảm.

---

# 28. Virtual Networking

Day 6 cần nhận biết Nutanix virtual networking ở mức architecture.

Cần hiểu:

- VM NIC;
- virtual network;
- AHV networking;
- VLAN dependency;
- physical network dependency.

Phần cấu hình VM/network chi tiết sẽ tiếp tục ở Day 7.

---

# 29. Cluster Services

Bạn cần hiểu cluster có nhiều service phụ thuộc lẫn nhau.

Operational mindset:

```text
Cluster healthy?
    |
    +-- Hardware healthy?
    +-- AHV hosts healthy?
    +-- CVMs healthy?
    +-- Cluster services healthy?
    +-- Storage healthy?
    +-- Network healthy?
```

Không troubleshoot Nutanix chỉ bằng một dashboard indicator.

---

# 30. Nutanix Health Dashboard

Phải học cách quan sát:

- overall health;
- node status;
- alert;
- capacity;
- service health;
- hardware health;
- storage state.

Và phải biết:

**green dashboard ≠ tuyệt đối không có issue**.

Health dashboard chỉ là một evidence source.

---

# 31. Alerts và Events

Bạn cần phân biệt operationally:

- alert;
- event;
- current condition;
- historical occurrence;
- acknowledgement;
- actual remediation.

Một alert được acknowledge không có nghĩa root cause đã được fix.

Phần này sẽ được mở rộng mạnh hơn ở Day 7.

---

# 32. Hardware / Software Inventory

Bạn phải biết inventory ít nhất các loại:

### Hardware

- node;
- model;
- serial/asset information khi được phép;
- CPU;
- memory;
- disk;
- NIC.

### Software

- AOS version;
- AHV version;
- Prism-related version awareness;
- firmware versions.

Mục tiêu là tạo **configuration baseline**.

---

# 33. Capacity Utilization

Bạn cần biết quan sát:

- CPU utilization;
- memory utilization;
- storage capacity;
- cluster headroom;
- utilization trend awareness.

Capacity là yếu tố bắt buộc phải kiểm tra trước disruptive maintenance.

---

# 34. Maintenance Boundaries

Đây là một trong những mục tiêu lớn nhất Day 6.

Bạn cần phân biệt:

### Read-only / low-risk

Ví dụ:

- inspect health;
- read logs;
- review alerts;
- inventory;
- version check;
- capacity check.

### Potentially disruptive

Ví dụ:

- power operation;
- firmware update;
- host maintenance;
- hardware replacement;
- reboot;
- storage intervention.

Các thao tác disruptive phải có:

```text
Authorization
+
Maintenance Window
+
Pre-check
+
Resiliency Check
+
Backup / Recovery Awareness
+
Stop Conditions
+
Post-check
+
Escalation Plan
```

Syllabus cũng nhấn mạnh high-risk operations như firmware upgrade và hyperconverged cluster maintenance phải có authorization, rollback planning và supervision phù hợp. Infrastructure_Operations_14_Da…

---

# 35. Hardware Replacement Prerequisites

Trước physical replacement cần suy nghĩ tới:

- xác định đúng faulty component;
- verify redundancy;
- workload dependency;
- cluster capacity;
- cluster health;
- maintenance approval;
- vendor procedure;
- replacement part compatibility;
- post-replacement validation.

Day 6 **không dạy kiểu “disk đỏ thì rút disk”**.

---

# 36. Supported Firmware / Software Combination

Một Nutanix node có nhiều layer:

```text
Hardware
+ Firmware
+ AHV
+ AOS
+ Prism
```

Các version có dependency.

Vì vậy không nên nâng firmware dựa đơn thuần vào tiêu chí “phiên bản mới hơn”.

Phải kiểm tra supported combination theo environment thực tế.

---

# 37. Operational Escalation Boundary

Sau Module Lý thuyết, bạn phải có khả năng xác định:

```text
Hardware problem?
→ HPE / hardware team / vendor

AHV problem?
→ virtualization / Nutanix operations

CVM/AOS problem?
→ Nutanix platform team/vendor

Physical network dependency?
→ network team

Guest OS/application?
→ server/application team
```

Đây chính là yêu cầu **“identify the failing layer and correct support owner”** trong Acceptance/Evidence. Infrastructure_Operations_14_Da…

---

# Module 2 — THỰC HÀNH

Module Thực hành sẽ biến toàn bộ kiến thức trên thành một **health-check + incident triage + maintenance planning exercise**. Assignment chính thức yêu cầu kiểm tra HPE ProLiant/iLO và Nutanix cluster, xây baseline, xử lý simulated warning và nộp combined evidence. Infrastructure_Operations_14_Da…

## 1. HPE ProLiant / iLO Environment Inspection

Bạn sẽ cần truy cập:

- HPE ProLiant server thực tế trong lab;

hoặc:

- authorized iLO simulator;
- instructor-provided screenshots/export nếu physical equipment không khả dụng.

Mục tiêu đầu tiên là xác định:

- server identity;
- hardware model;
- hardware inventory;
- firmware;
- health;
- power;
- management connectivity.

---

# 2. Hardware Inventory Collection

Bạn sẽ thu thập inventory gồm các thành phần phù hợp như:

- model;
- CPU;
- memory;
- disks;
- storage controller;
- NIC;
- power supply;
- fan;
- firmware versions.

Kết quả cần đủ để tạo một **hardware baseline**.

---

# 3. Firmware Baseline

Bạn sẽ ghi nhận:

- firmware component;
- current version;
- health/status;
- update requirement nếu được cung cấp;
- dependency/compatibility cần kiểm tra.

Ở Day 6 không mặc định update firmware thật.

Syllabus cho phép firmware operation ở dạng demonstration để tránh disruptive risk.

---

# 4. Power Status Verification

Phải xác định:

- current server power state;
- iLO connectivity;
- remote management availability.

Quan trọng là phân biệt:

**verify power status**

với

**perform power operation**.

Phần đầu thường read-only; phần sau có thể disruptive.

---

# 5. Hardware Health Check

Bạn sẽ phải review:

- overall health;
- CPU;
- memory;
- disk;
- storage controller;
- cooling;
- temperature;
- PSU;
- network/hardware sensors.

Kết quả không chỉ là screenshot.

Bạn phải đưa ra conclusion:

```text
Healthy
Degraded
Critical
Unknown / Insufficient evidence
```

---

# 6. IML Collection

Bạn sẽ truy xuất **Integrated Management Log**.

Cần lưu evidence về:

- timestamp;
- event;
- severity;
- affected component;
- status;
- probable impact.

---

# 7. Maintenance Requirement Assessment

Từ hardware evidence, bạn phải xác định:

- không cần action;
- monitoring;
- planned maintenance;
- urgent maintenance;
- vendor escalation.

Không được tự suy luận “critical = reboot”.

---

# 8. Nutanix Prism Element Access

Phần tiếp theo là Nutanix lab.

Bạn sẽ đăng nhập **Prism Element** và nhận diện:

- cluster;
- nodes;
- AHV hosts;
- CVMs;
- storage;
- network;
- alerts;
- events;
- health.

---

# 9. Prism Central Access

Nếu lab có Prism Central, bạn phải truy cập và phân biệt:

- Prism Central scope;
- Prism Element scope.

Mục tiêu là hiểu mình đang quản lý:

- một cluster;
- hay centralized environment.

---

# 10. Identify Nutanix Cluster

Phải thu thập các thông tin cơ bản:

- cluster identity;
- node count;
- AOS version;
- AHV version;
- overall health.

---

# 11. Identify Nodes

Với từng node cần nhận biết ít nhất:

- node identity;
- hardware dependency;
- AHV host;
- CVM;
- state/health.

---

# 12. Identify CVMs

Bạn cần xác nhận:

- CVM tương ứng;
- CVM state;
- relation với node;
- cluster service dependency.

---

# 13. Identify AHV Hosts

Bạn cần xác nhận:

- host list;
- host state;
- hardware node mapping;
- cluster membership.

---

# 14. Identify Storage Containers

Bạn sẽ inspect:

- storage pool/container objects;
- utilization;
- capacity.

Không nhất thiết cấu hình lại container trong Day 6.

---

# 15. Identify Virtual Networking

Bạn sẽ xác định các virtual networking objects được lab cung cấp và hiểu relation với:

- AHV;
- VM;
- VLAN;
- physical network.

---

# 16. Nutanix Health Inspection

Phải review:

- cluster health;
- node health;
- hardware status;
- CVM/service status;
- alerts;
- capacity.

---

# 17. Alert Inspection

Bạn sẽ xem:

- active alerts;
- severity;
- source;
- affected entity;
- timestamp.

Sau đó phải xác định:

```text
What is failing?
Where is it failing?
What is affected?
Who owns the next action?
```

---

# 18. Events Inspection

Review event history để xây timeline:

```text
T0 → warning detected
T1 → health status changed
T2 → alert created
T3 → administrator investigated
```

Timeline rất quan trọng cho incident/evidence sau này.

---

# 19. Software Version Inventory

Bạn sẽ ghi lại:

- AOS version;
- AHV version;
- management platform/version information khi available;
- relevant firmware/software baseline.

---

# 20. Capacity Inspection

Phải đánh giá:

- CPU;
- memory;
- storage;
- headroom;
- cluster utilization.

Mục tiêu không phải capacity planning chuyên sâu mà trả lời:

> Cluster có đủ capacity/resiliency để xem xét maintenance hay không?

---

# 21. Hardware Dependency Inspection

Nutanix VM/service phụ thuộc vào physical hardware.

Bạn phải nối được chuỗi:

```text
HPE hardware
↓
Nutanix node
↓
AHV
↓
CVM/AOS
↓
VM
↓
Application
```

---

# 22. Establish Configuration Baseline

Đây là deliverable bắt buộc.

Baseline có thể bao gồm:

- server identity;
- hardware inventory;
- firmware;
- node list;
- cluster information;
- AOS/AHV versions;
- health state;
- storage/capacity;
- alerts;
- management access.

Baseline là cơ sở cho:

- future maintenance;
- troubleshooting;
- before/after comparison.

---

# 23. Simulated Node / Hardware Warning

Lab sẽ có một simulated failure hoặc warning.

Ví dụ dạng an toàn:

```text
Disk warning
Node hardware warning
Power supply warning
Memory warning
Cluster/node health warning
```

Bạn phải điều tra mà không thực hiện destructive action.

---

# 24. Failure Layer Identification

Đây là bài kiểm tra tư duy quan trọng.

Bạn phải quyết định warning thuộc layer nào:

```text
Physical hardware
iLO
AHV
CVM
AOS
Prism
Network
Guest VM
Application
```

---

# 25. HPE IML ↔ Prism Correlation

Phần additive của syllabus yêu cầu:

> correlate a prepared hardware alert/IML entry with Prism health evidence. Infrastructure_Operations_14_Da…

Ví dụ tư duy:

```text
13:02 iLO IML
Physical disk warning

13:03 Prism
Node hardware warning

13:04 Prism
Cluster remains available
but resilience degraded
```

Bạn phải chứng minh hai evidence có liên quan bằng:

- timestamp;
- node;
- component;
- alert;
- health state.

---

# 26. Daily HPE Health Checklist

Bạn sẽ xây hoặc hoàn thiện checklist dạng:

```text
[ ] iLO reachable
[ ] Server power normal
[ ] Overall health normal
[ ] CPU healthy
[ ] Memory healthy
[ ] Storage healthy
[ ] PSU healthy
[ ] Cooling healthy
[ ] No unexpected IML critical event
[ ] Firmware baseline recorded
```

---

# 27. Daily Nutanix Health Checklist

Ví dụ scope:

```text
[ ] Prism reachable
[ ] Cluster healthy
[ ] All expected nodes present
[ ] AHV hosts healthy
[ ] CVMs healthy
[ ] Cluster services healthy
[ ] No unresolved critical alert
[ ] Storage capacity acceptable
[ ] Hardware warnings reviewed
[ ] Versions recorded
```

---

# 28. Determine Escalation Ownership

Từ warning đã điều tra, bạn phải chọn đúng owner.

Ví dụ:

```text
Physical DIMM failure
→ Hardware / HPE

AHV host state abnormal
→ Nutanix virtualization operations

CVM/AOS service problem
→ Nutanix platform support

Network path problem
→ Network team
```

Không chỉ ghi “escalate L2”.

Phải giải thích **vì sao**.

---

# 29. Maintenance Ticket Draft

Phần additive yêu cầu bạn draft một maintenance ticket.

Ticket phải chứa ít nhất:

### Affected scope

- nodes;
- hosts;
- VMs;
- service/application dependency.

### Impact

- expected impact;
- possible impact.

### Approved sequence

Các bước thực hiện theo đúng thứ tự.

### Prechecks

Ví dụ:

- cluster health;
- capacity;
- resiliency;
- affected workloads;
- evidence captured.

### Stop conditions

Ví dụ:

- additional node becomes unhealthy;
- insufficient capacity;
- new critical alert;
- cluster loses required resilience.

### Postchecks

Ví dụ:

- node healthy;
- cluster healthy;
- services normal;
- alerts cleared/resolved appropriately.

### Vendor escalation artifacts

Ví dụ:

- logs;
- IML;
- screenshots;
- serial/model;
- firmware;
- timestamp;
- cluster health evidence.

Đây là một deliverable chính thức của Day 6. Infrastructure_Operations_14_Da…

---

# 30. Physical Replacement Demonstration

Nếu có physical lab, instructor có thể demonstration:

- hardware identification;
- replacement prerequisites;
- maintenance sequence;
- vendor procedure.

Learner không được mặc định tự tháo/thay hardware trong production.

---

# 31. Firmware Operation Demonstration

Firmware operation cũng nên được thực hiện dưới dạng:

- demonstration;
- approved simulation;
- version-labelled documentation.

Trọng tâm là biết workflow an toàn hơn là “bấm Update”.

---

# 32. Pre-maintenance Checks

Bạn phải học checklist trước disruptive work:

```text
Authorization?
Maintenance window?
Correct node?
Cluster healthy?
Capacity sufficient?
Resilience sufficient?
Workloads identified?
Current alerts understood?
Firmware/software supported?
Evidence captured?
Vendor/runbook available?
Stop conditions declared?
```

---

# 33. Post-maintenance Checks

Sau maintenance, cần verify:

```text
Hardware health
Node status
AHV status
CVM status
Cluster health
Storage health
Capacity
Alerts/events
Workload availability
```

Không được kết thúc change chỉ vì action command đã thành công.

---

# 34. Evidence Collection

Syllabus yêu cầu **timestamped hardware/cluster evidence**. Infrastructure_Operations_14_Da…

Evidence package Day 6 cần có các nhóm chính:

### HPE evidence

- hardware inventory;
- firmware versions;
- power state;
- health indicators;
- IML;
- warning details.

### Nutanix evidence

- cluster inventory;
- nodes;
- AHV hosts;
- CVMs;
- storage;
- networking;
- versions;
- health;
- alerts;
- events;
- capacity.

### Operational evidence

- configuration baseline;
- health checklist;
- alert correlation;
- troubleshooting notes;
- maintenance ticket;
- escalation decision.

---

# 35. Evidence Integrity

Day 6 vẫn phải tuân theo evidence contract chung của toàn khóa.

Mỗi evidence item nên có:

- evidence ID;
- source product/version;
- asset/node/cluster identifier;
- timestamp + timezone;
- collection method;
- ticket reference;
- filename;
- size;
- record count khi phù hợp;
- SHA-256 khi export thành file;
- limitations/errors.

Raw evidence phải được giữ nguyên; working copy và redacted copy phải được quản lý riêng. Infrastructure_Operations_14_Da…

---

# 36. Final Day 6 Acceptance

Sau khi hoàn thành hai module, bạn phải chứng minh được bốn năng lực lớn.

**Thứ nhất — Identify the failing layer.**

Bạn phải phân biệt được issue thuộc:

```text
Hardware
iLO
AHV
CVM/AOS
Prism
Network
Guest/Application
```

**Thứ hai — Identify the correct support owner.**

Không escalate mơ hồ.

**Thứ ba — Produce timestamped evidence + configuration baseline.**

Evidence phải đủ để reviewer hiểu bạn đã quan sát gì và kết luận dựa trên đâu.

**Thứ tư — Produce a safe maintenance plan.**

Syllabus đặc biệt cảnh báo không được giả định:

```text
snapshot = guaranteed rollback
```

hoặc:

```text
firmware downgrade = guaranteed rollback
```

Maintenance plan phải phản ánh đúng recovery limitation. Infrastructure_Operations_14_Da…

---

## Cấu trúc chúng ta sẽ học chi tiết tiếp theo

Khi bạn yêu cầu **“Module Lý thuyết Day 6”**, tôi sẽ đi sâu theo luồng:

**Physical Server → HPE ProLiant → RAID/Controller → Firmware/BIOS/UEFI → iLO → IML → HCI → Nutanix Node → CVM → AOS → AHV → Prism Element → Prism Central → Storage → Virtual Networking → Health/Capacity → Management Plane Separation → Maintenance & Escalation.**

Khi bạn yêu cầu **“Module Thực hành Day 6”**, tôi sẽ xây dựng một lab hoàn chỉnh theo luồng:

**HPE/iLO inventory & health check → IML investigation → Nutanix inventory → Prism Element/Central inspection → node/CVM/AHV/storage/network mapping → health/capacity baseline → simulated fault investigation → IML/Prism correlation → escalation → maintenance ticket → evidence package → cleanup/acceptance.**

Như vậy, **Module Lý thuyết** sẽ giúp bạn hiểu chính xác từng layer và dependency; còn **Module Thực hành** sẽ buộc bạn chứng minh khả năng vận hành bằng **verification, troubleshooting, evidence, maintenance planning và escalation**, đúng mục tiêu Day 6 của syllabus.
