# Day 7 — Nutanix AOS/AHV Operations & HYCU Backup/Recovery

Theo syllabus, **Day 7 là ngày vận hành chuyên sâu hơn trên nền Nutanix và HYCU**, tiếp nối foundation về HPE/Nutanix của Day 6. Trọng tâm chuyển từ việc “nhận diện architecture, inventory, health và dependency” sang **thực hiện operational administration có kiểm soát, maintenance rehearsal, troubleshooting, backup/restore và recovery assurance**. Infrastructure_Operations_14_Da…

Day 7 thuộc Module **HPE Infrastructure, Nutanix Virtualization & HYCU Backup** của chương trình; các competency cần tạo ra gồm Prism/Prism Central operations, AOS operations, AHV VM/host operations và HYCU protection/recovery. Infrastructure_Operations_14_Da… Infrastructure_Operations_14_Da…

Ở giai đoạn này, tôi sẽ chỉ dựng **bản đồ đầy đủ của kiến thức Day 7 thành hai module Lý thuyết và Thực hành**. Khi bạn yêu cầu tiếp, tôi sẽ đi sâu từng module theo kiểu foundation → operational workflow → verification → troubleshooting → evidence → rollback → escalation.

---

# MODULE 1 — LÝ THUYẾT

Module Lý thuyết có thể chia thành **10 khối kiến thức chính**. Toàn bộ các khối dưới đây đều xuất phát trực tiếp từ Concept/Lecture và Additive Concept/Lecture của Day 7. Infrastructure_Operations_14_Da…

## 1. Nutanix AOS và AHV operational administration

Đây là nền tảng của Day 7.

Bạn cần hiểu mối quan hệ giữa:

| Thành phần                           | Vai trò cần hiểu                                                     |
| ------------------------------------ | -------------------------------------------------------------------- |
| **AOS — Acropolis Operating System** | Distributed storage và các cluster services chính của Nutanix        |
| **AHV — Acropolis Hypervisor**       | Hypervisor chạy VM                                                   |
| **CVM — Controller VM**              | Thành phần Nutanix control/storage services trên từng node           |
| **Prism Element**                    | Quản trị một Nutanix cluster                                         |
| **Prism Central**                    | Quản trị tập trung nhiều cluster và các chức năng management mở rộng |
| **Guest VM**                         | Workload thực tế chạy trên AHV                                       |
| **Storage Container**                | Logical storage được VM sử dụng                                      |
| **Virtual Network/VLAN**             | Kết nối network cho VM                                               |

Không chỉ cần biết chúng là gì mà phải hiểu dependency:

**Physical Node → AHV → CVM/AOS services → Storage/Network → VM workload**

Và phải biết khi có sự cố thì đang hỏng ở **layer nào**, tránh xử lý sai layer.

---

## 2. Prism Element vs Prism Central

Đây là một phần bắt buộc và quan trọng của Day 7.

Bạn cần phân biệt rõ:

| Prism Element                                | Prism Central                                                                |
| -------------------------------------------- | ---------------------------------------------------------------------------- |
| Cluster-level management                     | Centralized/multi-cluster management                                         |
| Theo dõi local cluster                       | Có phạm vi management lớn hơn tùy deployment                                 |
| VM, host, storage, alerts, tasks của cluster | Cross-cluster visibility và các centralized capabilities tùy version/license |
| RBAC theo scope tương ứng                    | RBAC có thể khác về scope                                                    |

Phần này không được học theo kiểu “Prism Central tốt hơn Prism Element”.

Bạn phải hiểu:

- object nào quản lý ở đâu;
- operation nào thuộc cluster scope;
- RBAC scope;
- event/task visibility;
- alert ownership;
- maintenance responsibility;
- khi nào cần chuyển từ Prism Central xuống Prism Element để điều tra.

Syllabus đặc biệt yêu cầu xem xét **Prism Element/Central RBAC và scope** theo đúng version đang triển khai. Infrastructure_Operations_14_Da…

---

## 3. AHV Virtual Machine lifecycle

Đây là phần VM administration cốt lõi.

Bạn phải nắm toàn bộ lifecycle:

**Provision → Configure → Connect → Operate → Monitor → Snapshot/Protect → Recover → Retire**

Các nội dung cụ thể gồm:

### VM provisioning

Hiểu các object cần quyết định khi tạo VM:

- VM name;
- vCPU;
- CPU topology awareness;
- memory;
- virtual disk;
- storage container;
- NIC;
- virtual network;
- VLAN;
- guest OS;
- boot settings;
- console access.

### CPU / Memory / Storage allocation

Không chỉ biết “gán CPU và RAM”.

Bạn phải hiểu:

- allocated resources;
- actual utilization;
- over-allocation;
- contention;
- headroom;
- capacity planning;
- workload impact.

### Virtual disks

Hiểu:

- VM disk attachment;
- storage container dependency;
- capacity;
- disk lifecycle;
- snapshot/recovery relationship.

### Virtual NIC

Hiểu:

- NIC association;
- network attachment;
- VLAN integration;
- guest-side IP configuration;
- difference giữa hypervisor-side connectivity và guest-side connectivity.

---

# 4. VLAN integration và VM connectivity

VM có thể “Running” nhưng application vẫn không hoạt động vì network.

Bạn cần hiểu đường đi:

**VM NIC → AHV virtual network → VLAN → physical network → gateway → destination**

Các tình huống cần phân biệt:

- VM không có NIC;
- NIC gắn sai network;
- VLAN sai;
- guest IP sai;
- default gateway sai;
- upstream switch/VLAN issue;
- firewall issue;
- application/service issue.

Day 7 không thay thế Day 8 về enterprise networking, nhưng cần đủ kiến thức để xác định:

> Đây là Nutanix/AHV problem hay network problem?

---

# 5. VM power operations, console và migration/placement

Bạn sẽ học các operation như:

- power on;
- graceful shutdown;
- power off;
- restart/reset awareness;
- console access.

Quan trọng nhất là operational safety.

Ví dụ:

**Guest shutdown** và **force power off** không phải cùng một operation về mức độ rủi ro.

Bạn phải hiểu:

- khi nào operation an toàn;
- khi nào cần application owner approval;
- khi nào cần maintenance window;
- cách verify workload sau operation.

### VM placement và migration awareness

Bạn cần hiểu ở mức operations:

- VM đang chạy trên AHV host nào;
- cluster có thể đặt workload ở đâu;
- resource availability;
- maintenance tác động tới workload placement;
- cần kiểm tra capacity trước khi đưa host vào maintenance.

Không được giả định rằng “cluster có nhiều node nên maintenance luôn an toàn”.

---

# 6. Snapshot, recovery point và backup — ba khái niệm phải tách biệt

Đây là một trong những điểm quan trọng nhất Day 7.

Syllabus yêu cầu rõ:

> **VM snapshots phải được tách biệt với independent backup protection.** Infrastructure_Operations_14_Da…

Bạn cần phân biệt:

### Snapshot

Thường dùng cho:

- short-term operational recovery;
- trước change được kiểm soát;
- rollback trong một số scenario phù hợp.

Nhưng không được mặc định snapshot là backup.

### Recovery Point

Một trạng thái dữ liệu có thể được recovery tại một thời điểm cụ thể.

### Independent Backup

Có:

- protection policy;
- backup target;
- retention;
- recovery workflow;
- backup monitoring;
- recovery assurance.

Điểm tư duy bắt buộc:

> “Backup job Successful” ≠ “Disaster Recovery đã được chứng minh”.

Syllabus toàn chương trình yêu cầu **isolated restoration, integrity validation và measured recovery time**, đặc biệt cho HYCU. Infrastructure_Operations_14_Da…

---

# 7. Cluster health, capacity và CVM/service operations

Day 7 yêu cầu bạn học cách đánh giá tình trạng platform chứ không chỉ VM.

Các vùng cần theo dõi gồm:

### Cluster resource utilization

- CPU;
- memory;
- storage;
- capacity trends;
- available headroom.

### Storage capacity alerts

Phải biết phân biệt:

- warning;
- critical condition;
- temporary spike;
- sustained growth;
- actual exhaustion risk.

Không chỉ đọc alert hiện tại, syllabus yêu cầu chú ý tới **capacity trends**. Infrastructure_Operations_14_Da…

### CVM status

Bạn cần hiểu:

- CVM nằm ở đâu;
- tại sao CVM quan trọng;
- service health liên quan tới CVM;
- khi CVM issue xảy ra thì phạm vi ảnh hưởng có thể khác VM issue.

### Cluster service health

Bạn cần biết cách đặt câu hỏi:

> Cluster có đang healthy hay chỉ UI trông bình thường?

Việc đánh giá phải dựa vào nhiều evidence:

- health;
- alerts;
- events;
- tasks;
- service state;
- NCC findings;
- capacity;
- workload state.

---

# 8. NCC — Nutanix Cluster Check awareness

Day 7 yêu cầu **NCC health-check awareness**.

Bạn chưa cần coi NCC như một “fix everything tool”.

Cần hiểu NCC dùng để:

- kiểm tra cluster configuration;
- health assessment;
- detect known conditions;
- hỗ trợ pre-maintenance/pre-upgrade validation.

Đặc biệt phải biết:

> NCC finding cần được **interpret**, không phải cứ thấy finding là tự động sửa.

Bạn cần phân loại:

- informational;
- actionable;
- maintenance blocker;
- potentially false/context-dependent result;
- condition cần vendor escalation.

---

# 9. AHV host maintenance và Lifecycle Manager

Đây là phần có mức operational risk cao hơn.

## AHV host maintenance

Bạn sẽ học workflow tư duy:

**Approval → Pre-check → Capacity check → Workload check → Cluster health → Maintenance operation → Monitor → Post-check → Exit maintenance → Verify**

Trước maintenance phải trả lời được:

- cluster có đủ resource không?
- workload có thể placement/migrate không?
- cluster health hiện tại ra sao?
- đang có unresolved alert không?
- CVM/service có healthy không?
- backup/protection có hợp lệ không?
- maintenance window đã được duyệt chưa?

Syllabus yêu cầu rehearsal workflow này trong instructor-controlled lab hoặc simulation. Infrastructure_Operations_14_Da…

## LCM — Lifecycle Manager

Bạn phải hiểu LCM ở mức operations:

- inventory;
- dependency;
- compatibility;
- prechecks;
- upgrade planning;
- firmware;
- AOS;
- AHV;
- supported upgrade sequence.

Điểm quan trọng:

**Không được xem firmware, AOS và AHV như ba update độc lập không liên quan.**

Phải hiểu dependency chain.

---

# 10. Firmware/AOS/AHV upgrade và rollback limitations

Day 7 không yêu cầu bạn tùy tiện upgrade cluster.

Bạn học cách **prepare và assess** upgrade.

Workflow logic:

**Current versions → Desired versions → Compatibility → Prerequisites → NCC/prechecks → Capacity/resiliency → Backup/recovery readiness → Maintenance window → Approved change → Upgrade sequence → Verification**

Đặc biệt phải hiểu:

### Upgrade không đồng nghĩa với dễ rollback

Syllabus Day 6–7 nhấn mạnh không được giả định:

- snapshot sẽ luôn bảo đảm rollback;
- firmware downgrade luôn có thể thực hiện;
- platform upgrade luôn reversible.

Rollback limitations phải được ghi nhận trước change. Infrastructure_Operations_14_Da…

---

# 11. RBAC và least privilege trên Nutanix

Day 7 yêu cầu **RBAC**.

Bạn cần hiểu:

- user;
- role;
- permission;
- scope;
- cluster access;
- centralized access;
- separation of duties.

Operational principle:

**Read-only diagnosis trước → minimum required privileges → change only with authorization.**

Ví dụ:

Người chỉ cần health-check không nên mặc định có:

- VM delete;
- cluster configuration;
- host maintenance;
- upgrade;
- network modification.

---

# 12. Nutanix Tasks, Events, Alerts và Audit logs

Đây là một phần rất quan trọng cho troubleshooting và audit evidence.

Bạn phải phân biệt:

### Task

Cho biết một operation đã được thực hiện:

- ai;
- khi nào;
- action nào;
- thành công/thất bại.

### Event

Cho biết sự kiện xảy ra trên platform.

### Alert

Condition cần attention.

### Audit

Dùng để trace administrative activity phù hợp với capability của platform/version.

Một distinction quan trọng của syllabus:

> **Alert acknowledgement ≠ Alert resolution.** Infrastructure_Operations_14_Da…

Việc acknowledge chỉ thể hiện đã tiếp nhận/thừa nhận alert.

Không chứng minh root cause đã được khắc phục.

---

# 13. Nutanix troubleshooting methodology

Day 7 không dạy troubleshooting theo kiểu “restart thử”.

Bạn cần học investigation theo layer.

Một troubleshooting tree điển hình:

**User/Application symptom**

↓

**Guest OS / Application**

↓

**VM state/resources**

↓

**AHV host**

↓

**Network**

↓

**Storage**

↓

**CVM/AOS services**

↓

**Cluster health**

↓

**Physical dependency**

Bạn phải học cách xác định:

- scope;
- blast radius;
- recent changes;
- resource condition;
- alert/event/task evidence;
- current cluster health;
- safe corrective boundary.

---

# 14. HYCU architecture for Nutanix

Đây là nửa lớn thứ hai của Day 7.

Bạn cần hiểu các thành phần logic:

**Nutanix Source → HYCU protection workflow → Backup Target → Recovery Point → Restore**

Các concept syllabus yêu cầu gồm:

- HYCU backup architecture;
- backup controller;
- source discovery;
- backup targets;
- protection policies;
- schedule;
- retention;
- application-aware protection;
- monitoring;
- recovery.

---

# 15. HYCU source discovery và backup targets

Bạn phải hiểu:

### Source

Workload hoặc Nutanix environment cần protect.

### Discovery

HYCU cần biết những object nào tồn tại và object nào cần protection.

### Target

Nơi backup data được lưu.

Trước khi cấu hình phải xác minh:

- target supported không;
- connectivity;
- authentication;
- available capacity;
- retention compatibility;
- product/version compatibility.

Syllabus yêu cầu xác minh **HYCU target/support compatibility theo workload và environment thực tế**. Infrastructure_Operations_14_Da…

---

# 16. HYCU protection policy, schedule và retention

Protection policy phải trả lời được:

- protect cái gì;
- backup khi nào;
- giữ bao lâu;
- tạo bao nhiêu recovery point;
- RPO kỳ vọng là gì;
- workload có cần application consistency không.

Bạn phải phân biệt:

**Schedule** = khi nào chạy.

**Retention** = giữ dữ liệu trong bao lâu.

**RPO** = mức data-loss window chấp nhận được.

Ba khái niệm này liên quan nhưng không giống nhau.

---

# 17. Application-aware protection

Một VM snapshot/backup không nhất thiết đồng nghĩa với application-consistent recovery.

Bạn cần hiểu khác biệt cơ bản giữa:

- crash-consistent;
- application-consistent.

Và tại sao workload như:

- database;
- transactional application;
- directory/application services

có thể cần application-aware protection.

Syllabus yêu cầu kiểm tra **application consistency cho workload được chọn**. Infrastructure_Operations_14_Da…

---

# 18. Backup monitoring và failed-job diagnosis

Một Infrastructure Operations engineer không chỉ kiểm tra “green/red”.

Bạn phải đọc:

- job ID;
- source;
- target;
- start/end time;
- selected recovery point;
- failure stage;
- error/error category;
- target state;
- capacity;
- connectivity;
- authentication;
- dependency.

Failure phải được điều tra có hệ thống.

Ví dụ:

**Backup failed**

→ Source available?

→ Target reachable?

→ Target capacity sufficient?

→ Credentials valid?

→ Policy valid?

→ Workload/application dependency?

→ Product/service issue?

---

# 19. VM restore và file-level restore

Bạn phải hiểu hai lớp recovery.

### VM recovery

Khôi phục cả VM hoặc workload.

### File recovery

Khôi phục object/file cụ thể.

Không phải scenario nào cũng nên restore cả VM.

Bạn cần chọn recovery method dựa trên:

- failure scope;
- RPO;
- RTO;
- recovery risk;
- identity/network conflict;
- data validation requirements.

---

# 20. Isolated restore và recovery verification

Đây là nội dung bắt buộc.

Syllabus yêu cầu restore vào **isolated network** để tránh:

- duplicate IP;
- duplicate hostname;
- service identity conflict;
- application collision;
- production traffic contamination.

Infrastructure_Operations_14_Da…

Sau restore không được chỉ nhìn trạng thái “Success”.

Phải validate:

**Backup → Restore → Boot/Open → Data validation → Compare source/reference → Record recovery time**

---

# 21. RPO và RTO

Bạn cần hiểu operationally:

### RPO — Recovery Point Objective

Bạn có thể mất bao nhiêu dữ liệu tính theo thời gian.

### RTO — Recovery Time Objective

Mất bao lâu để phục hồi service/workload.

Day 7 còn yêu cầu đo **observed recovery time**, không chỉ đọc RTO trên policy.

Ví dụ logic:

- recovery bắt đầu 10:00;
- service validated 10:27;

→ observed recovery time ≈ 27 phút.

Con số này là evidence thực nghiệm của lab, không tự động đồng nghĩa với production RTO guarantee.

---

# 22. DR operational procedure

Day 7 cần nối backup với Disaster Recovery thinking.

Một DR workflow cơ bản:

**Incident**

→ Assess scope

→ Declare recovery requirement

→ Select valid recovery point

→ Verify target/data availability

→ Restore isolated

→ Validate

→ Decide promotion/cutover

→ Document results

→ Escalate unresolved dependencies

Điểm quan trọng:

> Recovery phải được chứng minh bằng dữ liệu recover được và sử dụng được, không phải bằng trạng thái “backup completed”.

---

# MODULE 2 — THỰC HÀNH

Module thực hành Day 7 không phải một bài VM creation đơn giản. Theo syllabus, bạn phải hoàn thành một chuỗi lab tích hợp Nutanix + HYCU + maintenance + troubleshooting + evidence collection. Infrastructure_Operations_14_Da…

Tôi sẽ cấu trúc thành **8 practical labs**.

---

## Lab 1 — Provision một AHV VM

### Mục tiêu

Provision VM qua Prism với configuration được phê duyệt.

Bạn sẽ thực hiện:

**Prism → Create VM → CPU/RAM → Disk → Network → Power On → Console → Guest verification**

Các nội dung phải verify:

| Object       | Verification             |
| ------------ | ------------------------ |
| VM identity  | Đúng name/cluster        |
| vCPU         | Đúng approved allocation |
| RAM          | Đúng approved allocation |
| Disk         | Đúng size/storage        |
| NIC          | Đúng virtual network     |
| VLAN         | Đúng network             |
| Power        | VM Running               |
| Guest        | Boot thành công          |
| Connectivity | Test network phù hợp     |

---

# Lab 2 — Controlled VM operations

Thực hành:

- power operation;
- console access;
- resource inspection;
- snapshot creation;
- snapshot verification;
- controlled recovery/snapshot operation nếu lab cho phép.

Mục tiêu không chỉ là thao tác được mà phải ghi:

**Before state → Action → Expected result → Actual result → Verification**

---

# Lab 3 — Nutanix operational health inspection

Bạn sẽ thu thập và phân tích:

- cluster health;
- VM utilization;
- host utilization;
- storage capacity;
- alerts;
- events;
- tasks;
- audit records;
- CVM status;
- NCC findings nếu environment cho phép.

Đây là phần tạo operational baseline.

---

# Lab 4 — Nutanix troubleshooting scenario

Syllabus yêu cầu ít nhất một simulated condition liên quan tới:

- capacity; hoặc
- VM availability.

Bạn phải đi theo workflow:

**Symptom → Scope → Evidence → Layer isolation → Root cause → Corrective boundary → Verify → Escalate nếu cần**

Ví dụ scenario:

> VM unavailable.

Bạn không được ngay lập tức restart cluster service.

Phải xác định trước:

- VM off?
- guest issue?
- AHV host issue?
- resource exhaustion?
- network issue?
- storage issue?
- cluster alert?

---

# Lab 5 — AHV host-maintenance rehearsal

Đây là additive lab bắt buộc.

Syllabus cho phép:

- instructor-controlled hands-on;
- hoặc documented simulation nếu không có quyền maintenance thực tế.

Bạn sẽ thực hiện tư duy workflow:

**Change authorization**

→ Workload inventory

→ Cluster health

→ Capacity/headroom

→ Protection status

→ Migration/placement readiness

→ Maintenance operation/simulation

→ Monitor

→ Post-health check

→ Verify workload

→ Maintenance completion evidence

Lab này phải có **stop conditions**.

Ví dụ:

Không tiếp tục maintenance nếu:

- cluster đã degraded;
- insufficient capacity;
- unresolved critical NCC finding;
- critical backup unavailable;
- workload migration không khả thi;
- change approval thiếu.

---

# Lab 6 — HYCU protection workflow

Bạn sẽ cấu hình hoặc inspect:

- Nutanix source;
- backup target;
- protection policy;
- schedule;
- retention;
- protected VM;
- application-aware option nếu applicable.

Sau đó:

**Run Backup → Monitor Job → Verify Recovery Point**

Evidence phải bao gồm ít nhất:

- job ID;
- source;
- target;
- policy;
- timing;
- result;
- recovery point.

---

# Lab 7 — Controlled backup failure injection

Syllabus đưa ví dụ rõ:

- unavailable target;
- insufficient capacity.

Infrastructure_Operations_14_Da…

Workflow lab sẽ là:

**Healthy baseline**

→ Inject controlled fault

→ Run/observe job

→ Capture failure

→ Analyze logs

→ Identify root cause

→ Apply approved correction

→ Rerun

→ Verify success

Mục tiêu quan trọng là chứng minh:

> Bạn có thể giải thích **vì sao job fail**, chứ không chỉ làm nó chạy lại.

---

# Lab 8 — Isolated VM/File Restore

Đây là lab quyết định backup có thực sự usable không.

Workflow:

**Choose Recovery Point**

→ Select VM/File restore

→ Restore to isolated destination/network

→ Avoid identity/IP conflict

→ Verify restored object

→ Compare content

→ Measure recovery time

→ Record evidence

Syllabus yêu cầu compare dữ liệu recovered với source/reference và đo recovery time. Infrastructure_Operations_14_Da…

---

# Evidence bắt buộc của Day 7

Evidence của Day 7 không nên chỉ là screenshot.

Theo acceptance của Day 7, tối thiểu phải có:

| Evidence                | Nội dung                         |
| ----------------------- | -------------------------------- |
| Prism Task references   | Các operation đã thực hiện       |
| Prism Audit references  | Trace administrative actions     |
| Pre-maintenance health  | Cluster health trước maintenance |
| Post-maintenance health | Cluster health sau maintenance   |
| Maintenance decision    | Go/No-Go/Stop/Escalate           |
| HYCU Job ID             | Backup job cụ thể                |
| Source                  | Workload được protect            |
| Target                  | Backup destination               |
| Recovery Point          | Điểm khôi phục đã chọn           |
| Failure evidence        | Error/log của injected failure   |
| Root cause              | Nguyên nhân được xác định        |
| Restore logs            | Evidence của restore             |
| Content validation      | So sánh dữ liệu recovered        |
| Recovery time           | Thời gian recovery đo được       |
| Escalation record       | Nếu vượt L1/L2 boundary          |

Đây chính là các acceptance artifacts mà syllabus Day 7 yêu cầu. Infrastructure_Operations_14_Da…

Ngoài ra, evidence toàn chương trình phải hướng đến traceability: source/product/version, environment/asset, ticket, collector, timestamp/timezone, commands/filter, count/hash và limitation/error khi phù hợp; raw evidence phải được bảo tồn thay vì ghi đè bằng bản đã xử lý. Infrastructure_Operations_14_Da…

---

# Operational safety xuyên suốt Day 7

Day 7 có một số thao tác có thể ảnh hưởng cả cluster nên phải phân loại rõ.

| Thao tác                 | Mức độ                     |
| ------------------------ | -------------------------- |
| View VM state            | Read-only                  |
| View alerts/events/tasks | Read-only                  |
| View cluster utilization | Read-only                  |
| Health inspection        | Chủ yếu read-only          |
| VM console               | Access-sensitive           |
| Power operation          | Change                     |
| Snapshot operation       | Change                     |
| VM provisioning          | Change                     |
| Host maintenance         | Potentially disruptive     |
| AOS/AHV upgrade          | High-risk                  |
| Firmware upgrade         | High-risk                  |
| Backup policy change     | Change                     |
| Restore                  | Controlled recovery change |

Các hoạt động high-risk như hyperconverged cluster maintenance phải có **authorization, rollback planning và appropriate supervision**. Infrastructure_Operations_14_Da…

---

# Troubleshooting mindset cần đạt sau Day 7

Sau Day 7, khi gặp một incident kiểu:

> “VM không truy cập được và backup đêm qua cũng fail.”

Bạn không nên xử lý bằng trial-and-error.

Bạn phải có khả năng hình thành flow:

**1. Xác định symptom và blast radius**

→ một VM hay nhiều VM?

**2. Check Nutanix platform**

→ VM / AHV / CVM / cluster health?

**3. Check resource**

→ CPU / RAM / storage capacity?

**4. Check network**

→ NIC / VLAN / guest connectivity?

**5. Check recent Tasks/Events/Audit**

→ vừa có change gì?

**6. Check HYCU**

→ source, target, policy, job?

**7. Check recovery readiness**

→ usable recovery point còn không?

**8. Decide**

→ fix trong scope / stop / escalate / recover.

Đây mới là operational competence mà Day 7 hướng tới.

---

# Day 7 — Hai module chúng ta sẽ học sâu

Khi học chi tiết sau, tôi đề xuất giữ đúng hai module sau:

### **Module Lý thuyết — Nutanix AOS/AHV Operations & HYCU Backup/DR**

Sẽ học sâu toàn bộ:

**AOS/AHV architecture → Prism Element/Prism Central → VM lifecycle → resource allocation → networking/VLAN → power/console → placement/migration → snapshot → cluster capacity → CVM/services → NCC → host maintenance → LCM/upgrades → RBAC → Tasks/Events/Alerts/Audit → troubleshooting → HYCU architecture → source/target → policy → application consistency → failed jobs → restore → RPO/RTO → DR.**

### **Module Thực hành — Nutanix/HYCU Operational Lab**

Sẽ làm theo chuỗi:

**Provision VM → Network verification → Resource/health inspection → Controlled VM/snapshot operation → Prism evidence collection → Capacity/availability troubleshooting → Maintenance rehearsal → HYCU protection → Backup execution → Failure injection → Log diagnosis → Corrective action → Rerun → Isolated restore → Content validation → RPO/RTO/recovery-time measurement → Evidence package → Rollback/Escalation.**

Như vậy **không có topic nào thuộc Day 7 trong syllabus bị bỏ sót**, đồng thời vẫn giữ ranh giới rõ: **Lý thuyết giải thích hệ thống và cách suy luận; Thực hành chứng minh bạn vận hành, verify, troubleshoot, recover và thu thập evidence được trong môi trường lab có kiểm soát.** Infrastructure_Operations_14_Da…
