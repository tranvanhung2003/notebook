# Day 6 — Module Lý thuyết

## HPE ProLiant / HPE iLO + Nutanix HCI / AOS / AHV / Prism

Phần này là **toàn bộ Module Lý thuyết Day 6**, bám sát `Concept/Lecture` và phần `Additive skill coverage` của syllabus: physical server/HCI fundamentals, HPE ProLiant architecture, RAID/storage controller, firmware, BIOS/UEFI, hardware health/maintenance, HPE iLO, Remote Console, Virtual Media, power control, IML, hardware alert/escalation; tiếp theo là Nutanix cluster architecture, node, CVM, AOS, AHV, Prism Element, Prism Central, storage pool/container, resilience, virtual networking, cluster services, health/capacity/inventory và operational maintenance boundaries. Infrastructure_Operations_14_Da…

Có hai lớp nội dung:

- **Theo syllabus:** bắt buộc phải nắm trong Day 6.
- **Kiến thức bổ sung:** dùng để giải thích dependency và operational reasoning; không thay đổi mục tiêu syllabus.

Một nguyên tắc cần giữ xuyên suốt Day 6 là:

> Infrastructure Operations không chỉ cần biết “component này là gì”, mà phải xác định được **failure nằm ở layer nào, impact tới đâu, evidence nào chứng minh điều đó, mình được phép làm gì, và khi nào phải escalate**.

---

# PHẦN I — MENTAL MODEL CỦA DAY 6

## 1. Từ application quay xuống physical infrastructure

Trong các Day trước, bạn thường tiếp xúc với:

```text
Application
    ↓
Operating System
    ↓
Virtual Machine / EC2
    ↓
Storage / Network
```

Day 6 đi xuống dưới nữa:

```text
Application
    ↓
Guest OS
    ↓
Virtual Machine
    ↓
Hypervisor — AHV
    ↓
Nutanix platform — AOS / CVM
    ↓
Physical Node
    ↓
CPU / Memory / Disk / NIC / PSU / Fan / Controller
    ↓
HPE iLO / Hardware Management
```

Đây là thay đổi mental model rất quan trọng.

Nếu một application chậm, nguyên nhân có thể nằm ở:

```text
Application
Guest OS
VM resource pressure
AHV host
CVM/AOS service
Cluster storage
Physical disk
Storage controller
NIC
Physical switch
Memory
CPU
Thermal throttling
```

Do đó, Infrastructure Operations phải học cách **localize failure** trước khi thay đổi cấu hình.

---

# PHẦN II — PHYSICAL SERVER FUNDAMENTALS

## 2. Physical server là gì?

Physical server là một computer system được thiết kế để cung cấp compute, memory, storage, networking và hardware reliability cho workload enterprise.

Một server như HPE ProLiant không chỉ gồm:

```text
CPU + RAM + Disk
```

mà là cả một hardware platform:

```text
System board
├── CPU sockets
├── Memory channels / DIMMs
├── PCIe buses
├── Storage controllers
├── NICs
├── Local drives
├── Power supplies
├── Fans / cooling
├── Sensors
├── Firmware
└── Management controller — iLO
```

Đối với Infrastructure Operations, điều quan trọng là hiểu **dependency**, chứ không phải trở thành electronics engineer.

---

# 3. CPU

## What

CPU — Central Processing Unit — thực thi instructions của Operating System, hypervisor, VM và application.

Các term thường gặp:

- socket;
- core;
- thread;
- logical processor;
- clock frequency;
- cache;
- NUMA.

Ví dụ một server có:

```text
2 CPU sockets
×
32 physical cores/socket
=
64 physical cores
```

Nếu SMT/Hyper-Threading tương ứng được sử dụng, Operating System/hypervisor có thể nhìn thấy nhiều logical processors hơn physical cores.

## Why Infrastructure Operations quan tâm?

CPU ảnh hưởng trực tiếp tới:

- VM capacity;
- application performance;
- host utilization;
- workload placement;
- maintenance capacity.

Nếu một Nutanix host cần maintenance, workload có thể phải chạy trên các host còn lại.

Vì vậy câu hỏi không chỉ là:

> Host này có CPU khỏe không?

mà còn:

> Nếu host này rời cluster, các host còn lại có đủ CPU headroom hay không?

---

# 4. Memory / DIMM

Physical server sử dụng DIMM — Dual Inline Memory Module — để cung cấp system memory.

Bạn cần phân biệt:

```text
Installed memory
Usable memory
Allocated memory
Free/available memory
Hardware health
```

Một server có 1 TB installed RAM không đồng nghĩa toàn bộ 1 TB luôn usable.

Có thể xuất hiện:

- failed DIMM;
- predictive failure;
- ECC correction events;
- disabled memory;
- degraded memory configuration.

## ECC

Enterprise server thường sử dụng ECC — Error-Correcting Code memory.

Mục đích là phát hiện và trong những trường hợp hỗ trợ, sửa một số memory errors.

Điểm vận hành quan trọng:

```text
Corrected memory error
≠
không có vấn đề
```

Một lượng nhỏ corrected events có thể chưa gây outage, nhưng sự lặp lại hoặc predictive failure có thể cho thấy DIMM/module/channel cần kiểm tra.

---

# 5. Memory topology

CPU thường có các memory channels riêng.

Do đó DIMM placement không phải tùy ý.

Ví dụ conceptual:

```text
CPU 1
├── Channel A
├── Channel B
├── Channel C
└── Channel D

CPU 2
├── Channel A
├── Channel B
├── Channel C
└── Channel D
```

Vendor có population rules cụ thể.

Sai DIMM population có thể:

- giảm performance;
- làm một phần memory không usable;
- tạo hardware warning;
- ngăn boot trong một số configuration.

Infrastructure Operations không nên tự di chuyển DIMM mà không theo maintenance guide của đúng model.

---

# 6. Physical storage

Server có thể sử dụng:

- HDD;
- SAS HDD;
- SATA HDD;
- SSD;
- SAS SSD;
- SATA SSD;
- NVMe SSD.

Mỗi technology có đặc tính khác nhau về:

- latency;
- throughput;
- endurance;
- interface;
- hot-plug capability;
- failure behavior.

Day 6 không yêu cầu storage engineering chuyên sâu, nhưng phải đủ để đọc inventory và health.

---

# 7. Physical drive ≠ Logical drive

Một lỗi rất phổ biến của fresher là nhầm hai khái niệm này.

Ví dụ:

```text
Physical Disk 1
Physical Disk 2
Physical Disk 3
Physical Disk 4
        ↓
Storage Controller
        ↓
RAID Logical Drive
        ↓
Operating System
```

Operating System có thể nhìn thấy **một logical device**, dù bên dưới có nhiều physical disks.

Nếu một physical disk chết:

```text
OS filesystem vẫn có thể online
```

nếu RAID vẫn còn fault tolerance.

Do đó:

> Service vẫn chạy không đồng nghĩa storage hardware healthy.

---

# PHẦN III — RAID VÀ STORAGE CONTROLLER

# 8. Storage controller là gì?

Storage controller quản lý communication giữa system và drives.

Tùy server/configuration, controller có thể cung cấp:

- disk discovery;
- RAID;
- caching;
- logical drive abstraction;
- health reporting;
- rebuild;
- hot spare;
- monitoring.

HPE có các dòng controller khác nhau, nên syntax/UI/capabilities cần xác nhận theo model thực tế.

---

# 9. RAID là gì?

RAID — Redundant Array of Independent Disks — là kỹ thuật tổ chức nhiều drives thành một logical storage structure.

Các mục tiêu có thể bao gồm:

- redundancy;
- availability;
- performance;
- capacity aggregation.

Nhưng:

> RAID không phải backup.

Nếu data bị:

- accidental deletion;
- corruption;
- ransomware;
- application overwrite;

RAID không tự đưa bạn về phiên bản dữ liệu trước đó.

---

# 10. RAID 0

Concept:

```text
Disk A + Disk B
Data striped across disks
```

Ưu điểm:

- performance;
- tổng capacity.

Nhược điểm:

```text
No redundancy
```

Một disk failure có thể làm logical array mất dữ liệu.

Operational interpretation:

> RAID 0 không phải fault-tolerant RAID.

---

# 11. RAID 1

Concept:

```text
Disk A
  ↕ mirror
Disk B
```

Data được mirrored.

Nếu một disk hỏng, remaining disk có thể tiếp tục cung cấp data tùy controller/state.

Nhưng array lúc đó có thể ở:

```text
Degraded state
```

nghĩa là redundancy đã giảm.

---

# 12. RAID 5

Conceptually:

- striping;
- distributed parity;
- có thể chịu một số failure tương ứng với thiết kế RAID level.

Khi một disk fail, array có thể vẫn available nhưng degraded.

Sau replacement:

```text
Rebuild
```

được thực hiện.

Trong lúc rebuild, risk thường cao hơn vì redundancy chưa hoàn toàn phục hồi.

---

# 13. RAID 6

RAID 6 sử dụng additional parity so với RAID 5 và được thiết kế chịu được nhiều disk failure hơn RAID 5 trong giới hạn RAID configuration.

Điểm operations không phải học thuộc parity math, mà hiểu:

```text
Healthy
→ disk failure
→ degraded
→ replacement
→ rebuild
→ healthy
```

và nhận biết stop condition nếu thêm component fail trong thời gian degraded.

---

# 14. RAID 10

Concept:

```text
Mirroring + Striping
```

thường dùng để kết hợp performance và redundancy.

Tuy nhiên usable capacity thấp hơn raw capacity vì có mirroring overhead.

---

# 15. Degraded array

`Degraded` thường nghĩa là:

> Storage vẫn hoạt động nhưng redundancy/fault tolerance đã giảm.

Đây là trạng thái cực kỳ quan trọng.

Ví dụ:

```text
Before:
Disk 1 OK
Disk 2 OK
Disk 3 OK
Disk 4 OK

After:
Disk 1 OK
Disk 2 FAILED
Disk 3 OK
Disk 4 OK

Logical volume = still accessible
RAID state = degraded
```

Người mới dễ nghĩ:

> Application chưa down → không urgent.

Sai.

Nếu fault tolerance đang giảm, additional failure có thể dẫn tới serious outage/data loss.

---

# 16. Rebuild

Sau khi faulty drive được replace đúng quy trình, controller có thể reconstruct data lên replacement drive.

HPE mô tả rebuild là quá trình controller dùng fault-tolerance information trên các drives còn lại để reconstruct data. HPE cũng cảnh báo rằng nếu fault tolerance đã bị mất và thêm drive failure xảy ra trong quá trình rebuild, hậu quả có thể nghiêm trọng. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=a00106490en_us&docLocale=en_US&page=GUID-ED5471EB-7B70-45D7-A51E-6D4CD5F46E56.html&utm_source=chatgpt.com)

Operationally:

```text
Drive replaced
     ↓
Rebuild starts
     ↓
Array still vulnerable
     ↓
Rebuild progresses
     ↓
Fault tolerance restored
```

Do đó không được xem:

```text
Replacement complete
```

là:

```text
Incident resolved
```

Phải verify rebuild và final health.

---

# 17. RAID caveat khi học Nutanix

Đây là một điểm rất quan trọng.

Syllabus đưa **RAID and storage controller awareness** trong phần HPE physical server fundamentals, nhưng bạn **không được suy luận rằng mọi Nutanix data disk đều được đặt sau một conventional RAID logical volume**.

Nutanix platform/hardware design có validated storage configuration riêng.

Vì vậy:

> Học RAID để hiểu HPE/general server storage, nhưng khi thao tác Nutanix node phải theo đúng Nutanix/HPE validated architecture của cluster đang dùng.

Không tự ý tạo RAID array trên data disks của Nutanix chỉ vì bạn biết RAID.

---

# PHẦN IV — POWER, COOLING VÀ HARDWARE HEALTH

# 18. Power Supply Unit — PSU

Enterprise server thường có redundant PSU configuration.

Ví dụ:

```text
PSU 1 ─── Power Feed A
PSU 2 ─── Power Feed B
```

Mục tiêu có thể là tránh single power failure làm toàn server mất điện.

Một PSU fail có thể tạo tình huống:

```text
Server still running
but redundancy lost
```

Đây lại là ví dụ:

> Availability hiện tại không đồng nghĩa infrastructure healthy.

---

# 19. Cooling và fan

Server có:

- fans;
- temperature sensors;
- thermal control;
- airflow requirements.

Nếu cooling không đủ:

- component temperature tăng;
- fan speed tăng;
- CPU có thể throttle;
- hardware alert xuất hiện;
- trong tình huống nghiêm trọng system có thể thực hiện protective action tùy platform.

Do đó performance degradation đôi khi bắt nguồn từ hardware thermal condition.

---

# 20. Hardware sensors

iLO có thể thu thập/hiển thị thông tin từ nhiều hardware sensors.

Các nhóm thường gồm:

- temperature;
- fan;
- power;
- CPU;
- memory;
- storage;
- controller;
- system board.

Không nên chỉ nhìn một dòng `Overall Health`.

Một health investigation tốt luôn hỏi:

```text
Overall state?
↓
Which subsystem?
↓
Which exact component?
↓
Current or historical?
↓
Redundancy lost?
↓
Service impact?
```

---

# PHẦN V — FIRMWARE

# 21. Firmware là gì?

Firmware là software chạy gần hardware hơn Operating System.

Một HPE ProLiant có thể có nhiều firmware components:

```text
System ROM / UEFI firmware
iLO firmware
Storage controller firmware
NIC firmware
Drive firmware
Other device firmware
```

Vì vậy câu:

> “Server firmware version là bao nhiêu?”

thường quá mơ hồ.

Cần hỏi:

> Firmware của component nào?

---

# 22. Firmware khác OS patch như thế nào?

OS patch tác động tới:

```text
Operating System / package / service
```

Firmware tác động tới:

```text
Hardware behavior / initialization / device logic
```

Một firmware update có thể yêu cầu:

- iLO restart;
- device reset;
- system reboot;
- maintenance window;
- compatibility verification.

Tùy component và version.

---

# 23. Tại sao firmware update nguy hiểm hơn tưởng tượng?

Firmware update có thể thất bại do:

- wrong image;
- incompatible version;
- interrupted update;
- unexpected power loss;
- dependency mismatch;
- unsupported upgrade path.

HPE cảnh báo không được interrupt firmware update; một số update cần system reset/reboot để có hiệu lực. HPE cũng khuyến nghị bảo đảm nguồn điện ổn định trong firmware maintenance. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00005613en_us&docLocale=en_US&page=GUID-1D845C92-69C7-498D-AC32-02E89431FF79.html&utm_source=chatgpt.com)

Vì vậy workflow phải là:

```text
Identify exact hardware
→ identify current firmware
→ check approved target
→ check compatibility
→ check prerequisites
→ maintenance approval
→ prechecks
→ perform update
→ required restart/reboot
→ postchecks
```

Không phải:

```text
New firmware available → Update
```

---

# 24. “Latest” không đồng nghĩa “approved”

Trong enterprise:

```text
Latest vendor release
```

và

```text
Organization-approved version
```

không nhất thiết giống nhau.

Ví dụ organization có thể pin version vì:

- compatibility;
- validation;
- Nutanix support matrix;
- application constraints;
- known issue;
- staged rollout.

Infrastructure Operations phải theo:

```text
Approved baseline
```

không theo cảm tính “newer is better”.

---

# PHẦN VI — BIOS VÀ UEFI

# 25. BIOS / UEFI là gì?

BIOS/UEFI chịu trách nhiệm platform initialization trước khi OS/hypervisor boot.

Nó liên quan tới:

- hardware initialization;
- CPU configuration;
- memory initialization;
- boot order;
- boot mode;
- security configuration;
- virtualization options;
- device initialization.

---

# 26. Boot flow

Simplified flow:

```text
Power On
   ↓
Hardware initialization
   ↓
POST
   ↓
BIOS / UEFI
   ↓
Boot device selection
   ↓
Hypervisor / OS
   ↓
Services
```

Nếu failure xảy ra trước hypervisor:

```text
SSH to OS
```

không còn hữu ích.

Lúc đó out-of-band management như iLO trở nên đặc biệt quan trọng.

---

# 27. POST

POST — Power-On Self-Test — kiểm tra các platform components trong boot phase.

POST errors có thể liên quan:

- memory;
- storage;
- CPU;
- configuration;
- firmware.

Remote Console cho phép Infrastructure Operations quan sát các thông tin này từ xa.

---

# 28. BIOS/UEFI operational boundary

Thay đổi BIOS/UEFI có thể ảnh hưởng toàn host.

Ví dụ:

```text
virtualization setting
boot mode
device configuration
power profile
```

Do đó đây thường là **configuration-changing privileged operation**.

Read-only inspection và configuration change phải tách biệt rõ.

---

# PHẦN VII — HPE iLO

# 29. iLO là gì?

iLO — Integrated Lights-Out — là HPE out-of-band management technology.

Điểm cốt lõi:

```text
iLO ≠ Windows
iLO ≠ Linux
iLO ≠ AHV
```

Nó là một management controller/platform riêng phục vụ server management.

---

# 30. In-band vs Out-of-band

## In-band management

Bạn quản lý server thông qua system đang chạy.

Ví dụ:

```text
Administrator
     ↓
Production Network
     ↓
SSH
     ↓
Linux OS
```

Nếu OS:

- crash;
- network stack fail;
- kernel panic;
- boot failure;

in-band path có thể mất.

---

## Out-of-band management

Concept:

```text
Administrator
       ↓
Management Network
       ↓
iLO
       ↓
Physical Server
```

Đường này độc lập hơn với host OS.

Bạn có thể dùng iLO để quan sát hardware/boot trong nhiều tình huống host OS không reachable.

Đây là lý do enterprise server luôn rất coi trọng out-of-band management.

---

# 31. Management network separation

Một thiết kế tốt thường tách management plane khỏi normal workload traffic.

Concept:

```text
User / Application Network
             |
          Workloads


Admin / Management Network
             |
            iLO
```

Mục đích:

- reduce attack surface;
- control administrator access;
- limit exposure;
- retain emergency management path.

Không nên public expose iLO tùy tiện.

---

# 32. iLO capabilities

Theo syllabus, phải hiểu:

- remote management;
- Remote Console;
- Virtual Media;
- power controls;
- hardware inventory;
- health;
- firmware;
- IML.

HPE documentation hiện hành tiếp tục mô tả các nhóm capability này trên iLO 7; môi trường cũ hơn có thể dùng iLO 5/6 nên UI và feature availability cần được kiểm tra theo server generation/version. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00005342en_us&docLocale=en_US&utm_source=chatgpt.com)

---

# 33. Remote Console

Remote Console cho phép admin tương tác với:

```text
server screen
keyboard
mouse
```

qua iLO.

Điều này khác RDP.

RDP:

```text
Windows OS phải hoạt động đủ để cung cấp RDP
```

Remote Console:

```text
quan sát host console từ platform management layer
```

Do đó có thể thấy:

- POST;
- boot loader;
- BIOS/UEFI;
- hypervisor boot;
- OS boot screen;
- crash screen.

---

# 34. Tại sao Remote Console là high privilege?

Người có Remote Console có thể tương tác như đang ngồi trước physical machine.

HPE documentation thậm chí lưu ý rằng privilege này có thể cho phép user đi vào BIOS và thực hiện host-related tasks tùy quyền/configuration. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00002007en_us&docLocale=en_US&page=GUID-99266123-6796-4FAB-9F9C-24E0B3A756EF.html&utm_source=chatgpt.com)

Do đó Remote Console phải được cấp theo:

```text
least privilege
```

và access phải được audit.

---

# 35. Virtual Media

Virtual Media cho phép admin attach remote media tới server.

Concept:

```text
Admin workstation
     |
   ISO file
     |
     v
iLO Virtual Media
     |
     v
Server sees virtual device
```

Ứng dụng:

- OS install;
- rescue/recovery;
- diagnostics;
- boot media;
- firmware/service workflows.

---

# 36. Security implication của Virtual Media

Virtual Media là quyền nhạy cảm.

Người có quyền này về lý thuyết có thể cung cấp bootable media tới server.

Vì vậy:

```text
Virtual Media privilege
```

không nên cấp rộng rãi chỉ vì user cần “xem health”.

HPE tách `Virtual Media`, `Remote Console`, `Virtual Power and Reset`, `Host BIOS`, `Configure iLO Settings` thành các privileges riêng, phù hợp với nguyên tắc least privilege. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00002007en_us&docLocale=en_US&page=GUID-99266123-6796-4FAB-9F9C-24E0B3A756EF.html&utm_source=chatgpt.com)

---

# 37. Power Controls

iLO có thể cung cấp server power/reset operations.

Conceptually có thể bao gồm các operation như:

```text
Power On
Power Off
Reset
Power Cycle
```

feature cụ thể phụ thuộc generation/version.

Điểm quan trọng không phải nhớ button.

Điểm quan trọng là:

> Power privilege là disruptive privilege.

HPE mô tả `Virtual Power and Reset` là quyền có thể power-cycle/reset host và làm gián đoạn availability. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00002007en_us&docLocale=en_US&page=GUID-99266123-6796-4FAB-9F9C-24E0B3A756EF.html&utm_source=chatgpt.com)

---

# 38. Power operation safety

Trước bất kỳ operation nào có thể làm host xuống:

```text
Which server?
Which cluster?
Which workloads?
Are VMs running?
Can workload fail over/migrate?
Is cluster resilient?
Is there an approved change?
Maintenance window?
Stop condition?
```

Nếu chưa trả lời được, không thực hiện.

---

# PHẦN VIII — IML: INTEGRATED MANAGEMENT LOG

# 39. IML là gì?

IML — Integrated Management Log — là hardware/platform event history của HPE server.

HPE mô tả IML lưu historical server events do system ROM và services như iLO drivers tạo ra, bao gồm health/status information, firmware-related events, OS-related information và POST codes. iLO quản lý IML và log có thể hữu ích ngay cả khi host đang powered off. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00002007zh_cn&docLocale=zh_CN&page=GUID-D7147C7F-2016-0901-06D0-000000001A66.html&utm_source=chatgpt.com)

---

# 40. IML khác OS log

Ví dụ:

```text
Linux journal
Windows Event Log
```

chủ yếu thuộc OS/application layer.

Trong khi:

```text
IML
```

tập trung mạnh hơn vào hardware/platform events.

Một investigation tốt có thể phải correlation:

```text
IML
+
Prism alert/event
+
AHV/CVM state
+
Guest OS log
+
Application log
```

---

# 41. Anatomy của một hardware event

Khi đọc IML, không nên chỉ đọc description.

Hãy suy nghĩ theo schema:

```text
Timestamp
Component
Severity
Event type
Current state
Repeated?
Resolved?
Operational impact?
```

Ví dụ:

```text
09:15
Drive Bay X
Warning
Predictive failure
```

Câu hỏi tiếp theo:

```text
Drive nào?
Controller nào?
Array nào?
Node nào?
Cluster resilience hiện tại?
Workload impact?
```

---

# 42. Historical event vs current state

Một IML entry có thể ghi nhận:

```text
một sự kiện đã xảy ra
```

nhưng không nhất thiết phản ánh current state hoàn toàn.

Ví dụ:

```text
Fan temporarily failed
→ event logged
→ fan recovered
```

Do đó:

```text
Historical log
+
Current hardware status
```

phải được đọc cùng nhau.

---

# 43. IML retention consideration

HPE documentation cho biết khi IML đầy, các event mới có thể overwrite event cũ nhất. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00002007zh_cn&docLocale=zh_CN&page=GUID-D7147C7F-2016-0901-06D0-000000001A66.html&utm_source=chatgpt.com)

Do đó evidence collection có ý nghĩa:

Nếu incident quan trọng:

```text
export/capture relevant log
```

thay vì giả định log sẽ tồn tại vô thời hạn.

---

# PHẦN IX — HARDWARE ALERT INTERPRETATION

# 44. Severity không bằng impact

Một alert có thể mang severity cao nhưng Infrastructure Operations vẫn phải đánh giá context.

Ví dụ:

```text
Critical hardware alert
```

có thể xảy ra ở một redundant component.

Server có thể vẫn online.

Ngược lại một seemingly small network problem có thể khiến toàn host isolated.

Do đó:

```text
Severity
≠
Impact
```

Severity là input.

Impact phải được assessment.

---

# 45. Alert interpretation framework

Dùng chuỗi:

```text
1. What component?
2. What changed?
3. Is failure current?
4. Is redundancy affected?
5. Is service impacted?
6. Could a second failure cause outage?
7. What layer owns remediation?
```

Ví dụ:

```text
PSU 2 failed
       ↓
Server still powered
       ↓
PSU 1 active
       ↓
Power redundancy lost
       ↓
No immediate application outage
       ↓
Risk increased
       ↓
Hardware maintenance required
```

Đó là operational reasoning tốt hơn:

> Server vẫn chạy nên bỏ qua.

---

# 46. Symptoms vs root cause

Một physical problem có thể tạo symptom ở higher layer.

Ví dụ:

```text
Disk latency
↓
Storage latency
↓
VM I/O latency
↓
Database response slow
↓
Application timeout
```

Application timeout chỉ là symptom.

Root cause có thể nằm ở physical storage.

---

# PHẦN X — HARDWARE MAINTENANCE

# 47. Hardware maintenance lifecycle

Một maintenance flow tốt:

```text
Detection
   ↓
Validation
   ↓
Impact assessment
   ↓
Evidence collection
   ↓
Compatibility / replacement validation
   ↓
Approval
   ↓
Maintenance window
   ↓
Pre-check
   ↓
Maintenance
   ↓
Post-check
   ↓
Observation
   ↓
Closure
```

---

# 48. Hardware replacement không phải chỉ “thay part”

Ví dụ một failed disk.

Không được suy nghĩ:

```text
Disk bad
→ remove
→ insert new disk
```

Phải hỏi:

```text
Có đúng disk không?
Disk thuộc system nào?
Array/storage role là gì?
Redundancy hiện tại?
Có additional fault không?
Replacement part compatible?
Hot-plug supported?
Vendor procedure?
Backup/recovery status?
Cluster impact?
```

---

# 49. Wrong-disk risk

Trong storage incident, một trong những lỗi nguy hiểm nhất là remove nhầm healthy disk.

Ví dụ array đang degraded:

```text
Disk 2 FAILED
```

nhưng engineer rút:

```text
Disk 3 HEALTHY
```

Có thể làm tình hình nghiêm trọng hơn rất nhiều.

Vì vậy identification cần kết hợp:

- server identity;
- chassis;
- bay;
- serial;
- controller state;
- iLO evidence;
- physical locator/UID nếu procedure hỗ trợ;
- approved vendor procedure.

---

# PHẦN XI — HARDWARE ESCALATION

# 50. L1/L2 boundary

Day 6 không nhằm biến Infrastructure Operations thành HPE repair technician.

Trong nhiều organization, Infrastructure Operations có thể:

```text
inspect
verify
collect evidence
correlate alerts
assess impact
open/route ticket
perform approved runbook actions
```

nhưng không được tùy ý:

```text
remove hardware
flash arbitrary firmware
modify BIOS
power off production server
```

---

# 51. Khi cần escalate hardware

Ví dụ:

- predictive hardware failure;
- repeated DIMM errors;
- storage controller fault;
- disk failure ngoài runbook;
- unknown hardware alarm;
- unsupported replacement;
- firmware compatibility uncertainty;
- degraded redundancy;
- multiple simultaneous failures;
- vendor diagnostic code yêu cầu specialist;
- activity cần physical access mà bạn không được phép thực hiện.

---

# 52. Escalation package tốt

Không chỉ gửi:

> Server lỗi disk.

Mà nên có:

```text
Ticket ID
Server / Asset ID
Model
Serial if allowed
Environment
Timestamp + timezone
Affected component
Current health
IML evidence
Firmware baseline
Cluster/node mapping
Service impact
Actions already performed
Stop conditions
```

Điều này giảm ping-pong giữa teams.

---

# PHẦN XII — HYPERCONVERGED INFRASTRUCTURE

# 53. Traditional infrastructure

Một kiến trúc truyền thống có thể nhìn như:

```text
Compute Servers
      |
      |
Fibre Channel / Ethernet
      |
      v
External SAN
```

Compute và storage thường là các hệ thống tách biệt.

---

# 54. Hyperconverged Infrastructure — HCI

HCI đưa compute và storage resources gần nhau hơn trong các cluster nodes và sử dụng software để cung cấp distributed infrastructure services.

Simplified mental model:

```text
Node 1        Node 2        Node 3
-------       -------       -------
CPU           CPU           CPU
RAM           RAM           RAM
Storage       Storage       Storage
   \            |            /
      Distributed Platform
             |
          Cluster
```

---

# 55. HCI không phải “3 standalone servers”

Đây là điểm rất quan trọng.

Trong HCI:

```text
Node
```

không chỉ là một server độc lập.

Node là participant trong:

- compute cluster;
- storage platform;
- cluster services;
- resilience model;
- management plane.

Do đó một node operation có thể ảnh hưởng cluster-level behavior.

---

# 56. Scale-out

HCI thường áp dụng scale-out thinking:

```text
Need more capacity/resources?
→ add validated node/resources
```

thay vì chỉ scale-up một server.

Nhưng exact expansion rules phụ thuộc version/platform.

---

# PHẦN XIII — NUTANIX HIGH-LEVEL ARCHITECTURE

# 57. Các layer chính

Mental model Day 6:

```text
                    Prism Central
                         |
                 Central Management
                         |
                -------------------
                         |
                    Prism Element
                         |
                   Local Cluster
                         |
        -----------------------------------
        |                |                |
      Node 1           Node 2           Node 3
        |                |                |
       AHV              AHV              AHV
        |                |                |
       CVM              CVM              CVM
        |                |                |
   Local media      Local media      Local media
```

Đây là simplified architecture cho việc học.

Như đã lưu ý, **không được biến diagram này thành quy tắc tuyệt đối**: current Nutanix documentation có compute-only node không có local storage/CVM. [nutanix.com](https://www.nutanix.com/content/dam/nutanix/documents/certifications/prism-element-web-console-guide.pdf?utm_source=chatgpt.com)

---

# 58. Bốn thứ phải phân biệt

Nếu chỉ nhớ bốn câu ở Day 6, hãy nhớ:

```text
HPE iLO → Physical/OOB management
AHV     → Hypervisor
CVM/AOS → Nutanix platform/storage services
Prism   → Management/visibility plane
```

Nhầm bốn layer này sẽ dẫn tới troubleshooting sai.

---

# PHẦN XIV — NUTANIX NODE

# 59. Node là gì?

Một Nutanix node là một physical compute unit trong cluster.

Một standard HCI node có thể đóng góp:

- CPU;
- RAM;
- network;
- local storage;
- hypervisor;
- CVM/platform participation.

Nhưng exact design phụ thuộc node role/platform/version.

---

# 60. Node identity

Trong operations, một node có nhiều identifiers.

Có thể gặp:

```text
Hardware serial
BMC/iLO/IPMI address
AHV host IP
CVM IP
Hostname
Node UUID
Cluster membership
```

Không được nhầm các addresses với nhau.

---

# 61. Ví dụ mapping

```text
Physical Node N1
├── iLO / BMC IP: management-layer address
├── AHV Host IP: hypervisor management
├── CVM IP: Nutanix Controller VM
└── Guest VMs
```

Cùng một physical box nhưng nhiều management identities.

Khi ticket nói:

> Node 10.20.30.15 lỗi

phải hỏi:

> IP này thuộc iLO, AHV, CVM hay guest?

---

# PHẦ XV — CVM: CONTROLLER VM

# 62. CVM là gì?

CVM — Controller VM — là một privileged Nutanix infrastructure VM chạy Nutanix platform services trên các applicable nodes.

Điểm cần nhớ:

```text
CVM ≠ normal application VM
```

Không nên treat CVM giống một Ubuntu VM bình thường.

---

# 63. CVM role

Ở mức Day 6, hãy hiểu CVM là nơi nhiều Nutanix distributed infrastructure services hoạt động.

Nó liên quan tới các function như:

- distributed storage;
- metadata/services;
- cluster coordination;
- management interactions.

Chi tiết internal service architecture sâu hơn chưa cần trong Day 6.

---

# 64. Vì sao CVM cần đặc biệt thận trọng?

Nếu bạn restart một application VM:

```text
chủ yếu application đó bị ảnh hưởng
```

Nếu bạn tác động sai vào CVM:

```text
có thể ảnh hưởng Nutanix platform services
```

Do đó CVM access thường privileged và phải hạn chế.

Nutanix cũng khuyến nghị hạn chế CVM SSH access phù hợp với authentication policy. [nutanix.com](https://www.nutanix.com/content/dam/nutanix/documents/certifications/advanced-admin-aos.pdf?utm_source=chatgpt.com)

---

# 65. CVM failure không tự động bằng cluster failure

Vì Nutanix là distributed architecture, một CVM-related problem không nên được kết luận ngay:

```text
Cluster dead
```

Bạn phải kiểm tra:

- other CVMs;
- cluster services;
- fault tolerance;
- current alerts;
- workload state.

Đây là essence của distributed-system troubleshooting.

---

# 66. CVM exception

Không học thuộc:

> Một physical node luôn có CVM.

Official Nutanix documentation hiện hành có AHV compute-only nodes không có CVM/local storage trong một số architecture. [nutanix.com](https://www.nutanix.com/content/dam/nutanix/documents/certifications/prism-element-web-console-guide.pdf?utm_source=chatgpt.com)

Cách nhớ đúng là:

> CVM là Nutanix infrastructure role gắn với storage/platform architecture của cluster; topology phải xác nhận từ deployment thực tế.

---

# PHẦN XVI — AOS

# 67. AOS là gì?

AOS — Acropolis Operating System — là core software platform của Nutanix Cloud Infrastructure cung cấp nhiều distributed infrastructure/storage capabilities.

Đừng hiểu tên `Operating System` rồi nghĩ:

```text
AOS = OS chạy trực tiếp thay AHV
```

Không đúng mental model.

Trong một AHV-based Nutanix environment:

```text
AHV = hypervisor
AOS/CVM = Nutanix distributed platform/services
```

---

# 68. AOS làm gì ở mức khái niệm?

AOS liên quan tới:

- distributed storage;
- data services;
- cluster services;
- resiliency;
- data management;
- cluster management integration.

Chi tiết implementation sâu hơn thuộc Nutanix advanced administration.

---

# 69. AOS version awareness

Version matters vì:

- feature support;
- upgrade path;
- AHV compatibility;
- Prism compatibility;
- hardware/firmware compatibility;
- bug fixes;
- security fixes.

Không được chạy một operation dựa vào guide version khác mà chưa xác nhận.

---

# PHẦ XVII — AHV

# 70. AHV là gì?

AHV — Acropolis Hypervisor — là Nutanix hypervisor.

Mental model:

```text
Physical Hardware
       ↓
      AHV
       ↓
Virtual Machines
```

AHV cung cấp compute virtualization layer.

---

# 71. AHV vs CVM

Đây là câu phỏng vấn thường gặp.

AHV:

```text
hypervisor
```

CVM:

```text
Nutanix infrastructure VM/platform service role
```

AHV chạy VMs.

CVM là một special Nutanix infrastructure VM trong applicable topology.

---

# 72. AHV host

Một AHV host là physical node đang chạy AHV.

Host có resources:

- CPU;
- memory;
- NIC;
- infrastructure access.

VMs được scheduled/running trên hosts.

---

# 73. Host failure

Nếu AHV host failure:

```text
VMs on that host
```

có thể bị ảnh hưởng.

Khả năng recovery phụ thuộc:

- cluster health;
- HA configuration;
- capacity;
- fault tolerance;
- workload characteristics.

Không nên hứa:

> Một host chết VM chắc chắn không downtime.

Enterprise operations không dùng từ “guaranteed” nếu chưa xác nhận architecture/configuration.

---

# PHẦ XVIII — PRISM ELEMENT

# 74. Prism Element là gì?

Prism Element là built-in management interface cho **một Nutanix cluster cụ thể**.

Nutanix documentation mô tả mỗi cluster có một Prism Element instance cho local cluster management; multiple clusters có thể được quản lý thông qua Prism Central. [nutanix.com](https://www.nutanix.com/content/dam/nutanix/documents/certifications/advanced-admin-aos.pdf?utm_source=chatgpt.com)

Đây là distinction cốt lõi.

---

# 75. Prism Element scope

Bạn có thể nghĩ:

```text
Prism Element
     ↓
One cluster
```

Các object thường được quan sát/quản lý có thể gồm:

- hosts;
- VMs;
- storage;
- alerts;
- tasks/events;
- networking;
- health;
- capacity.

Exact menus thay đổi theo release.

---

# 76. Vì sao Prism Element quan trọng trong incident?

Nếu một specific cluster gặp vấn đề, Prism Element là nguồn visibility rất quan trọng để xem:

```text
Cluster health
Host health
Storage health
Alerts
Tasks/events
Capacity
VM state
```

Nhưng Prism Element không thay thế:

- iLO;
- hardware logs;
- network switch logs;
- guest OS logs.

---

# PHẦ XIX — PRISM CENTRAL

# 77. Prism Central là gì?

Prism Central là centralized management plane cho Nutanix environment.

Mental model:

```text
              Prism Central
             /      |       \
            /       |        \
       Cluster A Cluster B Cluster C
          PE       PE        PE
```

Current Nutanix documentation vẫn mô tả Prism Central với infrastructure monitoring/management capabilities; current `pc.7.6` documentation được phát hành trong 2026, nhưng môi trường lab/enterprise của bạn có thể chạy release khác. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Prism-Central-Guide%3Aprism-central-upgrade-procedures-c.html&utm_source=chatgpt.com)

---

# 78. Prism Central không phải Prism Element “bản lớn hơn” đơn giản

Hai system có overlap nhưng scope khác.

Mental model:

```text
Prism Element
→ cluster-local operational scope

Prism Central
→ centralized/multi-cluster scope + centralized capabilities
```

Feature availability còn phụ thuộc:

- Prism Central version;
- AOS version;
- license;
- deployment.

---

# 79. Khi troubleshoot nên bắt đầu ở PE hay PC?

Không có một câu trả lời tuyệt đối.

Ví dụ centralized operations team có thể nhận alert tại Prism Central rồi drill down cluster.

Nếu đang investigate cụ thể cluster:

```text
Prism Element
```

thường là local cluster visibility quan trọng.

Cần hiểu **management scope**, không học theo button location.

---

# 80. Prism Central availability ≠ cluster availability

Một conceptual distinction quan trọng:

```text
Prism Central inaccessible
```

không nhất thiết có nghĩa:

```text
all Nutanix clusters down
```

Tương tự:

```text
Prism Element management issue
```

không nhất thiết có nghĩa:

```text
all workload VMs down
```

Management plane và data/workload plane phải được phân biệt.

---

# PHẦ XX — MANAGEMENT PLANE SEPARATION

Syllabus đặc biệt yêu cầu tách rõ:

> out-of-band iLO management, AHV host operations, CVM services và Prism management planes. Infrastructure_Operations_14_Da…

Đây là core competency của Day 6.

---

# 81. Layer map

```text
┌─────────────────────────────────────┐
│ Prism Central                       │
│ Centralized management plane        │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│ Prism Element                       │
│ Local cluster management plane      │
└─────────────────┬───────────────────┘
                  │
        ┌─────────┴─────────┐
        │                   │
┌───────▼────────┐  ┌───────▼────────┐
│ CVM / AOS      │  │ AHV            │
│ Platform       │  │ Hypervisor     │
│ services       │  │ Compute        │
└───────┬────────┘  └───────┬────────┘
        └─────────┬─────────┘
                  │
┌─────────────────▼───────────────────┐
│ Physical Nutanix/HPE Node           │
│ CPU/RAM/Disk/NIC/Power              │
└─────────────────┬───────────────────┘
                  │
┌─────────────────▼───────────────────┐
│ iLO                             │
│ Out-of-band hardware management    │
└─────────────────────────────────────┘
```

---

# 82. Troubleshooting example: Prism không mở được

Không kết luận ngay cluster down.

Đi từ low-risk checks:

```text
Client connectivity?
DNS?
Management network?
Prism service?
Cluster IP?
CVM?
Cluster services?
Physical node?
```

Trong khi đó check:

```text
Workload VMs có reachable không?
```

Điều này giúp phân biệt:

```text
Management plane outage
```

với:

```text
Service/data plane outage
```

---

# PHẦ XXI — STORAGE POOL VÀ CONTAINER

# 83. Storage Pool

Storage Pool là logical grouping của cluster storage resources.

Simplified hierarchy:

```text
Physical Storage Resources
          ↓
     Storage Pool
          ↓
  Storage Containers
          ↓
        VM Data
```

Đây chỉ là conceptual hierarchy; implementation chi tiết có thể phức tạp hơn.

---

# 84. Storage Container

Container là một logical storage construct nằm trên storage resources.

VM virtual disks/data có thể được provision từ container tùy platform/configuration.

Bạn nên hiểu container giống:

> Một logical storage boundary/policy object mà platform sử dụng để present/manage storage.

Không nên đồng nhất container với:

```text
one physical disk
```

hay:

```text
one RAID LUN
```

---

# 85. Distributed storage

Nutanix storage không nên mental-model như:

```text
VM on Node 1
→ data chỉ nằm trên disk của Node 1
```

AOS là distributed platform; data protection/resilience được cung cấp ở cluster level theo platform configuration.

Do đó physical disk/node failure analysis cần xét:

```text
cluster resilience
```

chứ không chỉ local disk.

---

# PHẦ XXII — REPLICATION VÀ RESILIENCE

# 86. Resilience là gì?

Resilience là khả năng infrastructure tiếp tục cung cấp service hoặc recover trong presence của failures.

Có thể liên quan:

- disk failure;
- node failure;
- component failure;
- network path failure.

Day 6 cần hiểu concept, chưa cần deep storage mathematics.

---

# 87. Replication

Nutanix platform có mechanisms để maintain protected copies/data structures across the cluster theo configuration.

Ý nghĩa operations:

```text
One physical copy fails
```

không nhất thiết có nghĩa:

```text
Data unavailable
```

nhưng health/resilience có thể giảm.

---

# 88. Healthy vs available vs resilient

Ba khái niệm này không giống nhau.

## Available

Service vẫn đang chạy.

## Healthy

Components/services ở state bình thường.

## Resilient

System vẫn còn khả năng chịu thêm defined failures.

Ví dụ:

```text
VM running = yes
Cluster serving workload = yes
One disk failed = yes
Required redundancy reduced = yes
```

Có thể kết luận:

```text
Available: Yes
Fully healthy: No
Risk: Increased
```

Đây là cách suy nghĩ rất quan trọng.

---

# 89. Fault tolerance check trước maintenance

Current Nutanix AHV documentation yêu cầu kiểm tra cluster health/fault tolerance trước các operation như restarting CVM/AHV host hoặc đưa AHV host vào maintenance mode; critical alerts phải được xử lý phù hợp trước disruptive operation. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

Do đó:

```text
Host is healthy
```

chưa đủ để quyết định maintenance.

Phải hỏi:

```text
Cluster can tolerate this host leaving?
```

---

# PHẦ XXIII — VIRTUAL NETWORKING

# 90. Nutanix virtual networking mental model

Trong AHV environment:

```text
VM vNIC
   ↓
Virtual Network
   ↓
AHV Virtual Switch / networking layer
   ↓
Physical NIC
   ↓
Physical Switch
   ↓
Network
```

Do đó VM connectivity phụ thuộc cả virtual và physical networking.

---

# 91. VM network vs management network

Không được đánh đồng:

```text
VM traffic
AHV management traffic
CVM traffic
backplane traffic
iLO traffic
```

Chúng có thể đi qua different logical/physical paths tùy design.

---

# 92. CVM/host networking

Current Nutanix documentation mô tả cả management và backplane traffic models, bao gồm communication giữa CVMs và hosts; network segmentation có thể tách certain cluster traffic khỏi general management traffic. Exact interface names/topology phụ thuộc deployment/version nên không học cứng một interface number cho mọi environment. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Security-Guide-v7_6%3Awc-segmented-unsegmented-networks-difference-c.html&utm_source=chatgpt.com)

---

# 93. Why networking matters for HCI

HCI phụ thuộc network rất mạnh.

Network issue có thể trông giống:

- storage issue;
- CVM issue;
- host issue;
- VM issue.

Ví dụ:

```text
Physical uplink problem
↓
CVM connectivity issue
↓
Cluster communication degradation
```

Do đó Nutanix troubleshooting không được bỏ qua physical network dependency.

---

# 94. Network redundancy

Một production cluster thường cần network redundancy phù hợp architecture.

Concept:

```text
Host
├── Uplink A → Switch A
└── Uplink B → Switch B
```

Mục tiêu là tránh một cable/NIC/switch port failure trở thành single point of failure.

Nutanix security/network documentation cũng nhấn mạnh việc sử dụng redundant uplinks trong fault-tolerant network design. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=Nutanix-Security-Guide%3Aportal-full-page-view-html&utm_source=chatgpt.com)

---

# PHẦ XXIV — CLUSTER SERVICES

# 95. Distributed services mindset

Nutanix cluster không phải một monolithic daemon duy nhất.

Có nhiều platform services phối hợp.

Ở Day 6 không cần thuộc toàn bộ internal service names.

Bạn cần biết:

```text
CVM up
```

chưa chắc:

```text
all cluster services healthy
```

---

# 96. Cluster service health

Mental checklist:

```text
Cluster reachable?
CVMs reachable?
Cluster services UP?
Critical alerts?
Storage healthy?
Hosts healthy?
Network healthy?
```

Current AHV documentation vẫn sử dụng cluster service health verification như một phần operational checks. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide-v11_2%3Aahv-vm-start-cli-t.html&utm_source=chatgpt.com)

---

# PHẦ XXV — CLUSTER DISCOVERY VÀ INVENTORY

# 97. Inventory là gì?

Inventory trả lời:

> Chúng ta đang vận hành cái gì?

Một useful inventory của Day 6 phải map được physical và logical layers.

Ví dụ:

```text
Cluster: NTNX-LAB-01

Node N1
├── Hardware model
├── Serial / asset reference
├── iLO/BMC identity
├── AHV identity
├── CVM identity
└── Firmware/software versions

Node N2
...
```

---

# 98. Vì sao inventory quan trọng?

Không có inventory, troubleshooting dễ xảy ra:

- log nhầm host;
- reboot nhầm node;
- replace nhầm component;
- firmware apply nhầm model;
- open vendor case thiếu thông tin;
- không biết affected workloads.

Inventory là operational control, không phải giấy tờ hành chính vô nghĩa.

---

# 99. Hardware inventory

Thường cần:

- node/model;
- CPU;
- memory;
- disks;
- controllers;
- NICs;
- PSU;
- firmware.

---

# 100. Software inventory

Có thể gồm:

- AOS version;
- AHV version;
- Prism Central version;
- relevant firmware;
- cluster information.

---

# 101. Baseline

`Baseline` là known/approved reference state.

Ví dụ:

```text
AOS: approved version X
AHV: approved version Y
Firmware bundle: approved baseline Z
Node count: 4
Expected CVMs: ...
Capacity: ...
Critical alerts: 0
```

Sau change:

```text
Before
vs
After
```

so sánh với baseline.

---

# PHẦ XXVI — HEALTH DASHBOARD

# 102. Dashboard là summary, không phải chân lý tuyệt đối

Một dashboard giúp aggregate health.

Nhưng:

```text
Green
```

không chứng minh rằng:

- không có historical fault;
- không có application issue;
- không có latent risk;
- evidence complete.

Tương tự:

```text
Red
```

không tự giải thích root cause.

---

# 103. Health evaluation hierarchy

Tốt hơn nên nghĩ:

```text
Overall Cluster
    ↓
Hosts
    ↓
CVM / platform
    ↓
Storage
    ↓
Network
    ↓
Hardware
    ↓
Workloads
```

---

# PHẦ XXVII — ALERTS, EVENTS VÀ TASKS

# 104. Alert

Alert thường biểu thị condition cần attention.

Ví dụ:

```text
hardware warning
capacity warning
service warning
```

---

# 105. Event

Event là occurrence/change được ghi nhận.

Ví dụ:

```text
node state changed
service restarted
administrator operation
```

Event có thể useful để reconstruct timeline.

---

# 106. Task

Task đại diện cho operation/work item được platform thực hiện hoặc tracking.

Day 7 sẽ đi sâu hơn tasks/audit operations.

Day 6 chỉ cần hiểu distinction.

---

# 107. Acknowledge ≠ resolve

Một lỗi rất thường gặp:

```text
Alert acknowledged
```

rồi ticket đóng.

Sai.

Acknowledgement thường chỉ có nghĩa:

> Có người đã nhìn thấy alert.

Resolution phải chứng minh:

```text
Root cause addressed
+
health restored
+
verification passed
```

---

# PHẦ XXVIII — CAPACITY

# 108. Capacity không chỉ là storage

Cluster capacity gồm nhiều dimensions:

```text
CPU
Memory
Storage
Network
```

Một cluster có storage dư nhiều nhưng memory gần hết vẫn có thể không đủ capacity để evacuate host.

---

# 109. Capacity utilization vs headroom

Utilization:

```text
đang dùng bao nhiêu
```

Headroom:

```text
còn khả năng hấp thụ thêm workload bao nhiêu
```

Maintenance quan tâm headroom.

---

# 110. Ví dụ

Giả sử 4 hosts:

```text
Host A = 70%
Host B = 70%
Host C = 70%
Host D = 70%
```

Nhìn từng host:

> Còn 30%, có vẻ ổn.

Nhưng nếu remove Host D:

workload D phải được absorb bởi A/B/C.

Tổng remaining headroom có thể không đủ.

Do đó:

> Maintenance capacity là cluster-level question.

---

# 111. Storage capacity risk

Distributed storage cũng cần free capacity/headroom để:

- normal writes;
- rebuild/re-protection;
- maintenance;
- data movement.

Nếu cluster gần full:

```text
disk/node failure
```

có thể khó recover hơn.

---

# PHẦ XXIX — HARDWARE DEPENDENCIES

# 112. Nutanix software vẫn phụ thuộc hardware

HCI là software-defined nhưng không phải hardware-independent.

Chain:

```text
HPE PSU
↓
Physical Node
↓
AHV
↓
CVM / AOS
↓
VM
↓
Application
```

Một PSU/fan/controller/NIC failure có thể xuất hiện lên Prism dưới dạng infrastructure warning.

---

# 113. Correlation iLO ↔ Prism

Đây là mục bắt buộc của syllabus.

Ví dụ:

```text
10:01 iLO IML
Drive warning Node 2

10:02 Prism
Hardware warning Node 2

10:03 Prism
Cluster remains available
```

Từ đây suy luận:

```text
Root/failing layer likely physical hardware
Prism alert is higher-level visibility
Correct evidence source includes iLO/IML
```

---

# 114. Không dùng correlation giả

Hai event gần thời gian chưa chắc causal.

Bạn cần match:

```text
timestamp
asset/node identity
component
event type
state
```

Correlation phải có evidence.

---

# PHẦ XXX — SECURITY VÀ LEAST PRIVILEGE

# 115. iLO là privileged management plane

Nếu attacker có privileged iLO access, họ có thể có khả năng:

- Remote Console;
- power operations;
- Virtual Media;
- BIOS configuration;
- firmware operations,

tùy privileges.

Vì vậy iLO phải được bảo vệ nghiêm ngặt.

---

# 116. Separate privileges

HPE documentation hiện hành tách các privileges như:

- Login;
- Remote Console;
- Virtual Power and Reset;
- Virtual Media;
- Host BIOS;
- Configure iLO Settings;
- Administer User Accounts;
- Host NIC;
- Host Storage. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00002007en_us&docLocale=en_US&page=GUID-99266123-6796-4FAB-9F9C-24E0B3A756EF.html&utm_source=chatgpt.com)

Điều này minh họa least privilege rất tốt.

Một monitoring operator không nhất thiết cần:

```text
Power Reset
```

chỉ để xem health.

---

# 117. Prism/CVM privileged access

Tương tự Nutanix:

- read-only visibility;
- VM operator;
- cluster administrator;
- CVM SSH;
- host root;

không nên cấp cùng mức cho tất cả users.

---

# 118. Management credentials

Không được:

- share passwords trong ticket;
- paste secrets vào screenshots;
- lưu credential trong evidence;
- dùng shared administrator account khi policy cấm.

Evidence phải chứng minh identity/access mà không expose secret.

---

# PHẦ XXXI — DAILY OPERATIONAL HEALTH MODEL

# 119. HPE physical health

Mental health check:

```text
Management reachable?
Power state correct?
Overall hardware health?
CPU?
Memory?
Storage?
Controller?
PSU?
Fan/thermal?
NIC?
IML?
Firmware baseline?
```

---

# 120. Nutanix health

Mental health check:

```text
Prism reachable?
Cluster state?
Expected nodes present?
Hosts healthy?
CVM/platform healthy?
Cluster services healthy?
Critical alerts?
Storage capacity?
Hardware alerts?
Network dependencies?
Software versions?
```

---

# PHẦ XXXII — SAFE TROUBLESHOOTING MODEL

Syllabus yêu cầu Infrastructure Operations phải investigate theo thứ tự low-risk → higher-risk.

Đây là một trong những principles quan trọng nhất.

---

# 121. Level 1 — Observe

Read-only:

```text
Dashboard
Alerts
Events
Inventory
Health
Logs
Capacity
Version
```

Không thay đổi system.

---

# 122. Level 2 — Correlate

So sánh:

```text
iLO
Prism
AHV/CVM
OS/application
network
```

Tạo timeline.

---

# 123. Level 3 — Validate

Dùng safe checks:

```text
Is issue reproducible?
Is component currently failed?
Are other nodes healthy?
Is there redundancy?
```

---

# 124. Level 4 — Approved corrective action

Chỉ khi:

```text
runbook
+
authorization
+
known impact
+
rollback/recovery understanding
```

---

# 125. Level 5 — Disruptive operation

Ví dụ:

- reboot;
- maintenance mode;
- firmware;
- power operation;
- hardware replacement.

Phải có stronger controls.

---

# PHẦ XXXIII — FAILURE MODES VÀ TRIỆU CHỨNG

# 126. Memory failure

Possible symptoms:

- iLO memory warning;
- POST warning;
- reduced usable memory;
- host instability;
- Prism hardware alert.

Investigate:

```text
iLO health
IML
affected DIMM
current host state
cluster capacity
```

---

# 127. Disk failure

Possible symptoms:

```text
IML event
storage controller warning
Prism hardware alert
reduced resilience
I/O performance issue
```

Do not immediately remove disk.

---

# 128. PSU failure

Possible:

```text
server stays online
but redundancy lost
```

Need assess:

- second PSU state;
- power feed;
- node/cluster risk;
- maintenance.

---

# 129. Fan/temperature problem

Symptoms:

```text
fan warning
temperature warning
performance degradation
high fan speed
hardware protection action
```

Check physical/environmental factors before blaming application.

---

# 130. AHV host issue

Symptoms:

```text
host unreachable
VM availability changes
Prism host alert
```

Need distinguish:

```text
AHV issue
vs
physical node issue
vs
network issue
```

---

# 131. CVM/platform issue

Symptoms may include:

- cluster service health warning;
- Prism alert;
- storage/platform behavior;
- CVM reachability problem.

Do not restart CVM casually.

---

# 132. Prism issue

If Prism inaccessible:

- management plane issue possible;
- cluster/workloads may still be alive.

Always verify service plane separately.

---

# PHẦ XXXIV — OPERATIONAL MAINTENANCE BOUNDARIES

# 133. Read-only operations

Generally lower risk:

```text
view inventory
view health
view alerts
read IML
view versions
view capacity
```

Still subject to authorization and data classification, but usually non-disruptive.

---

# 134. Configuration-changing operations

Examples:

```text
BIOS setting change
iLO configuration
network configuration
cluster configuration
```

Need change control according to organization.

---

# 135. Disruptive operations

Examples:

```text
Power Off
Reset
AHV host maintenance
CVM restart
firmware update
hardware replacement
```

Highest scrutiny.

Syllabus explicitly requires high-risk operations such as firmware and hyperconverged cluster maintenance to use authorization, rollback/recovery planning and appropriate supervision. Infrastructure_Operations_14_Da…

---

# PHẦ XXXV — PRE-MAINTENANCE THINKING

# 136. Identity check

First:

```text
Am I on the correct system?
```

Confirm:

- environment;
- cluster;
- host;
- node;
- asset.

---

# 137. Health check

Never start with:

```text
Cluster already degraded
```

unless the approved recovery procedure specifically requires intervention.

Ask:

```text
Any critical alert?
Any node down?
Storage healthy?
Network healthy?
```

---

# 138. Capacity check

If host leaves:

```text
Can remaining cluster absorb workload?
```

---

# 139. Resilience check

Ask:

```text
Can cluster tolerate planned component removal?
```

Current Nutanix guidance specifically calls for cluster health/failure-tolerance checks before restart/maintenance operations. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

---

# 140. Compatibility check

For firmware/software:

```text
Hardware
↕
Firmware
↕
AHV
↕
AOS
↕
Prism
```

Versions cannot be treated independently.

---

# 141. Impact check

Identify:

```text
VMs
Applications
Owners
Users
Dependencies
```

---

# 142. Backup/recovery awareness

Maintenance may fail.

Need know:

```text
What is the recovery path?
```

Nhưng rất quan trọng:

> Không được giả định snapshot luôn là rollback hoàn hảo.

Và:

> Không được giả định firmware luôn có thể downgrade an toàn.

Syllabus nêu rõ maintenance plan không được dựa trên giả định rằng snapshots hay firmware downgrade bảo đảm rollback. Infrastructure_Operations_14_Da…

---

# PHẦ XXXVI — STOP CONDITIONS

# 143. Stop condition là gì?

Là điều kiện buộc engineer:

```text
STOP
```

không tiếp tục maintenance.

Ví dụ:

```text
Unexpected second node failure
New critical alert
Capacity below threshold
Loss of required resilience
Unexpected workload outage
Incorrect target identified
Firmware validation failure
```

---

# 144. Tại sao phải định nghĩa trước?

Trong incident/change, human bias thường khiến người thực hiện nghĩ:

> Đã làm nửa chừng rồi, cố tiếp.

Stop conditions giúp tránh escalation của failure.

---

# PHẦ XXXVII — POST-MAINTENANCE VERIFICATION

# 145. “Command succeeded” không bằng “change succeeded”

Ví dụ update tool nói:

```text
SUCCESS
```

Chưa đủ.

Phải verify:

```text
Firmware version
Hardware health
AHV status
CVM/services
Cluster health
Alerts
VMs
Application
```

---

# 146. Compare before vs after

Tư duy:

```text
Baseline before
       ↓
Change
       ↓
Observed after
       ↓
Compare
```

Nếu không có precheck, bạn rất khó biết issue sau maintenance là:

- new;
- pre-existing.

---

# PHẦ XXXVIII — CONFIGURATION BASELINE

# 147. Baseline content

Day 6 configuration baseline nên hiểu conceptually gồm:

```text
Asset identity
Hardware model
Hardware inventory
Firmware versions
Power/health
Cluster identity
Node mapping
AHV version
AOS version
Prism scope/version
Storage/capacity
Active alerts
```

---

# 148. Baseline vs backup

Hai thứ khác nhau.

Baseline:

```text
record of expected/known configuration
```

Backup:

```text
recoverable copy/data/configuration
```

Bạn có thể có baseline mà không có full backup.

---

# PHẦ XXXIX — EVIDENCE

# 149. Evidence là gì?

Evidence là artifact cho phép người khác xác minh:

- bạn đã quan sát gì;
- lúc nào;
- ở đâu;
- bằng cách nào;
- kết luận dựa trên gì.

Syllabus yêu cầu Day 6 giữ timestamped hardware/cluster evidence, configuration baseline và maintenance plan. Infrastructure_Operations_14_Da…

---

# 150. Good evidence

Ví dụ:

```text
Evidence ID: D6-HPE-001
Source: iLO
Asset: NODE-02
Timestamp: 2026-10-21 10:15 +07:00
Observation: Drive warning
Ticket: CHGxxxx / INCxxxx
```

---

# 151. Screenshot alone is weak

Screenshot có ích nhưng thiếu context nếu:

- không có timestamp;
- không có asset identity;
- cropped mất environment;
- không có ticket;
- không biết filter.

Evidence tốt phải reproducible/traceable.

---

# 152. Evidence manifest

Common evidence contract của syllabus yêu cầu các field như evidence ID, source/product/version, asset/account identifier, ticket, collector, collection timestamp/timezone, filter/query, file metadata/count/hash và limitations khi phù hợp. Infrastructure_Operations_14_Da…

Đây sẽ tiếp tục được dùng ở Day 10–14.

---

# PHẦ XL — FAILURE-LAYER IDENTIFICATION

# 153. Đây là competency quan trọng nhất của Day 6

Khi gặp incident, mục tiêu đầu tiên không phải fix.

Mục tiêu là:

```text
Where is the failure?
```

---

# 154. Hardware layer

Evidence:

```text
iLO critical health
IML hardware fault
physical sensor
controller fault
```

Likely owner:

```text
hardware/platform team/vendor
```

---

# 155. AHV layer

Evidence:

```text
host-specific hypervisor condition
VM scheduling/host issue
```

Likely owner:

```text
virtualization/Nutanix operations
```

---

# 156. CVM/AOS layer

Evidence:

```text
cluster services
storage/platform alerts
CVM-related condition
```

Likely owner:

```text
Nutanix platform team/vendor
```

---

# 157. Prism layer

Evidence:

```text
management UI/API issue
while workloads remain available
```

Likely:

```text
management plane/platform issue
```

---

# 158. Network layer

Evidence:

```text
lost uplink
VLAN
switch port
routing
management network
```

Likely owner:

```text
network team
```

---

# 159. Guest layer

Evidence:

```text
VM healthy at hypervisor level
guest OS service failing
```

Likely owner:

```text
server/application team
```

---

# PHẦ XLI — ROOT-CAUSE TREE

Một practical reasoning tree:

```text
Application unavailable
        |
        v
VM reachable?
 |             |
No            Yes
 |             |
 v             v
VM state?    OS/service issue?
 |
 v
Host healthy?
 |             |
No            Yes
 |             |
 v             v
AHV/hardware? Network/storage?
 |
 v
Physical node healthy?
 |             |
No            Yes
 |             |
 v             v
iLO/IML       CVM/AOS/network
```

Không phải rigid procedure, nhưng giúp tránh nhảy ngay tới reboot.

---

# PHẦ XLII — COMMON MISTAKES

## 160. Mistake: thấy Critical là reboot

Sai vì chưa biết failure layer.

---

## 161. Mistake: server ping được nên hardware healthy

Sai.

PSU/disk/memory redundancy có thể đang degraded.

---

## 162. Mistake: Prism green nghĩa toàn bộ stack healthy

Sai.

Prism không thay thế mọi evidence source.

---

## 163. Mistake: acknowledge alert = fixed

Sai.

---

## 164. Mistake: một disk fail thì rút ngay

Nguy hiểm nếu:

- wrong disk;
- array state khác dự kiến;
- platform-specific storage;
- additional failure.

---

## 165. Mistake: update latest firmware

Sai nếu chưa kiểm tra supported/approved baseline.

---

## 166. Mistake: restart CVM như application VM

Sai operational boundary.

---

## 167. Mistake: Prism Central down = all clusters down

Sai management-plane reasoning.

---

## 168. Mistake: CVM down = toàn cluster chắc chắn down

Sai distributed-system reasoning.

---

## 169. Mistake: snapshot = backup = rollback

Ba khái niệm không đồng nhất.

---

## 170. Mistake: service available = resilience intact

Sai.

System có thể đang:

```text
available but degraded
```

---

# PHẦ XLIII — VERSION-SENSITIVE REALITY

Đây là phần rất quan trọng để không học cứng.

## HPE

HPE ProLiant estate có thể chứa nhiều generations và tương ứng có thể gặp các iLO generations khác nhau. Tài liệu HPE hiện hành năm 2026 có cả iLO 5, iLO 6 và iLO 7 documentation; iLO 7 documentation hiện tại tiếp tục được cập nhật cho supported newer systems. [support.hpe.com](https://support.hpe.com/hpesc/public/docDisplay?docId=sd00005342en_us&page=GUID-96BF049D-2906-44C2-A25B-F8106BFBDCCA.html&utm_source=chatgpt.com)

Do đó trong production:

```text
Check server generation
→ check iLO generation
→ check firmware version
→ use matching documentation
```

---

## Nutanix

Current Nutanix documents trong 2026 có các product releases như:

```text
AOS 7.6
AHV 11.2
Prism Central pc.7.6
```

nhưng đây **không có nghĩa lab hoặc production của bạn đang chạy các version này**. [Nutanix Portal](https://portal.nutanix.com/page/documents/details?targetId=AHV-Admin-Guide%3Aahv-cluster-health-verify-t.html&utm_source=chatgpt.com)

Do đó:

```text
Never copy a command/workflow
solely because it is from latest docs.
```

Always match deployed version.

---

# PHẦ XLIV — HPE/NUTANIX OPERATIONAL DEPENDENCY MAP

Một architecture map đầy đủ hơn:

```text
Users
  |
Application
  |
Guest OS
  |
VM
  |
AHV Hypervisor
  |
Physical Nutanix/HPE Node
  |-------------------------|
  | CPU / RAM               |
  | NIC                     |
  | Drives                  |
  | Storage Controller      |
  | PSU / Fans              |
  |-------------------------|
       |
      iLO
       |
Hardware Management Network


Parallel Nutanix Platform Path:

CVM
 |
AOS Distributed Services
 |
Storage / Cluster Services
 |
Prism Element
 |
Prism Central
```

Khi incident xảy ra, bạn xác định path nào degraded.

---

# PHẦ XLV — OPERATIONAL SCENARIOS ĐỂ HIỂU LÝ THUYẾT

## 171. Scenario 1 — Một PSU fail

State:

```text
Node running
VMs running
Prism mostly healthy
iLO reports PSU 2 failure
```

Bad reasoning:

> Không có outage nên đóng alert.

Good reasoning:

```text
Physical failure identified
↓
Power redundancy reduced
↓
Additional PSU/feed failure could cause outage
↓
Collect iLO/IML
↓
Assess cluster/workload impact
↓
Plan hardware maintenance
```

---

# 172. Scenario 2 — Disk warning

```text
IML: predictive failure
Prism: hardware warning
VMs: operational
```

Correct interpretation:

```text
Available now
but physical storage component unhealthy
```

Do not immediately:

```text
reboot VM
```

vì không liên quan root cause.

---

# 173. Scenario 3 — Prism Central inaccessible

But:

```text
Prism Element Cluster A reachable
Applications reachable
VMs healthy
```

Likely first hypothesis:

```text
Central management-plane issue
```

not:

```text
Nutanix infrastructure completely down
```

---

# 174. Scenario 4 — Host disappears from Prism

Possible causes include:

```text
physical power
hardware fault
AHV problem
network issue
management connectivity
```

Need iLO cross-check.

If iLO says:

```text
Server Powered Off
```

direction differs greatly from:

```text
Server Powered On / hardware healthy
```

---

# 175. Scenario 5 — Application slow

Prism indicates:

```text
cluster CPU normal
memory normal
```

but iLO shows:

```text
thermal issue
```

Need investigate hardware thermal behavior rather than only VM CPU.

---

# PHẦ XLVI — ESCALATION BOUNDARY

## L1 / junior Infrastructure Operations

Expected capability:

- inspect dashboards;
- verify inventory;
- capture hardware health;
- inspect IML;
- identify alerts;
- map node/CVM/AHV;
- collect evidence;
- basic correlation;
- follow approved runbook;
- escalate correctly.

---

## L2 / experienced Operations

May additionally:

- deeper correlation;
- approved host/CVM operational actions;
- maintenance preparation;
- capacity/resilience assessment;
- vendor case preparation;
- controlled remediation according to runbook.

Exact permissions depend organization.

---

## Platform/vendor engineering

Examples:

- complex AOS issue;
- multi-node service degradation;
- unsupported hardware failure;
- firmware defect;
- data integrity concern;
- advanced cluster repair;
- unresolved NCC/health problems;
- unusual storage conditions.

---

# PHẦ XLVII — KHI NÀO KHÔNG ĐƯỢC “CỐ FIX”

Phải stop/escalate nếu:

```text
failure layer uncertain
multiple components degraded
cluster resilience insufficient
unexpected data/storage issue
no approved runbook
no change authorization
production impact expanding
firmware compatibility unknown
required evidence unavailable
vendor support required
```

Professional operation không phải người sửa được mọi thứ.

Professional operation là người **biết boundary**.

---

# PHẦ XLVIII — SỰ KHÁC NHAU GIỮA SNAPSHOT, BACKUP VÀ HARDWARE ROLLBACK

Đây là knowledge bổ sung quan trọng vì syllabus cảnh báo rollback assumption.

## Snapshot

Snapshot có thể cung cấp recovery point ở virtualization/storage layer tùy implementation.

Nhưng snapshot:

```text
không làm failed DIMM sống lại
không rollback faulty PSU
không đảm bảo firmware revert
```

---

## Backup

Backup là independent protection copy theo backup architecture.

Day 7 sẽ học HYCU.

---

## Firmware rollback

Firmware downgrade có thể:

- unsupported;
- blocked;
- có compatibility issue;
- không revert configuration/state hoàn hảo.

Do đó “backout plan = downgrade firmware” chỉ hợp lệ nếu vendor/runbook xác nhận.

---

# PHẦ XLIX — WHY DAY 6 PREPARES DAY 7

Day 6 trả lời:

```text
What is the infrastructure?
Is it healthy?
Which layer is failing?
```

Day 7 sẽ tiến tới:

```text
How do I operate it?
```

bao gồm:

- VM provisioning;
- CPU/memory/storage allocation;
- virtual disks;
- NICs/VLAN;
- power operations;
- snapshots;
- recovery points;
- migration;
- host maintenance;
- NCC;
- LCM;
- upgrade dependencies;
- RBAC;
- audits;
- HYCU backup/recovery.

Vì vậy nếu Day 6 không chắc:

```text
Node
CVM
AOS
AHV
Prism Element
Prism Central
```

Day 7 sẽ rất khó.

---

# PHẦ L — NHỮNG RELATIONSHIP PHẢI THUỘC

Bạn nên có thể giải thích ngay:

```text
HPE ProLiant
= physical server platform

iLO
= out-of-band physical server management

AHV
= Nutanix hypervisor

CVM
= Nutanix infrastructure Controller VM

AOS
= Nutanix distributed platform/storage software

Prism Element
= local cluster management

Prism Central
= centralized/multi-cluster management
```

---

# PHẦ LI — HEALTH REASONING PHẢI THUỘC

Không nói:

```text
"It's green."
```

Mà phải có khả năng nói:

```text
Hardware health: Normal
AHV hosts: Expected hosts present
CVM/platform services: Healthy
Storage: No active critical alert
Cluster capacity: Within approved operating range
Critical alerts: None observed in checked window
Evidence timestamp: ...
```

Đó mới là operational statement.

---

# PHẦ LII — TROUBLESHOOTING REASONING PHẢI THUỘC

Flow:

```text
Symptom
  ↓
Scope
  ↓
Impact
  ↓
Layer
  ↓
Current health
  ↓
Historical events
  ↓
Correlate evidence
  ↓
Identify likely root cause
  ↓
Check operational boundary
  ↓
Remediate or escalate
  ↓
Verify
```

---

# PHẦ LIII — SECURITY REASONING PHẢI THUỘC

Day 6 security không tập trung firewall rules mà tập trung **infrastructure management privilege**.

Câu hỏi luôn là:

```text
Does this person only need visibility?
```

Nếu có:

```text
Power/reset
Virtual Media
Host BIOS
firmware
CVM SSH
cluster administration
```

thì quyền đã tăng mạnh.

Least privilege phải áp dụng.

---

# PHẦ LIV — MAINTENANCE REASONING PHẢI THUỘC

Trước disruptive change:

```text
1. Correct asset?
2. Approved ticket?
3. Maintenance window?
4. Current cluster healthy?
5. Capacity sufficient?
6. Resilience sufficient?
7. Workloads identified?
8. Supported compatibility?
9. Evidence captured?
10. Recovery/backout understood?
11. Stop conditions defined?
12. Escalation contact ready?
```

Không đủ một mục quan trọng → không nên proceed tùy tiện.

---

# PHẦ LV — EVIDENCE REASONING PHẢI THUỘC

Không chỉ lấy screenshot.

Phải bảo đảm:

```text
WHAT
WHERE
WHEN
HOW
WHO
TICKET
RESULT
```

Ví dụ:

```text
WHAT:
PSU failure

WHERE:
NODE02

WHEN:
2026-10-21T10:15:00+07:00

SOURCE:
HPE iLO / IML

RELATED SYSTEM:
Nutanix Cluster NTNX01

RESULT:
Node remains online; power redundancy degraded
```

---

# PHẦ LVI — MỘT INCIDENT HOÀN CHỈNH VỀ MẶT LÝ THUYẾT

Giả sử monitoring báo:

```text
Nutanix Node 03 hardware warning
```

Một Infrastructure Operations engineer có tư duy đúng sẽ không lập tức reboot.

### Phase 1 — Scope

Xác nhận:

```text
Cluster?
Node?
Environment?
```

### Phase 2 — Current state

Check:

```text
Cluster health
Node status
VM impact
Critical alerts
```

### Phase 3 — Hardware correlation

Check iLO:

```text
Overall health
IML
Physical component
```

Phát hiện:

```text
Power Supply 2 Failed
```

### Phase 4 — Impact

Node:

```text
still powered
```

VM:

```text
running
```

Nhưng:

```text
power redundancy lost
```

### Phase 5 — Risk

Nếu remaining PSU/power feed fail:

```text
node outage
```

### Phase 6 — Ownership

Root failing layer:

```text
physical hardware
```

Correct owner:

```text
HPE/hardware maintenance team
```

không phải:

```text
application team
```

### Phase 7 — Evidence

Capture:

```text
iLO health
IML
Prism node health
Cluster status
Affected workloads
Timestamp
```

### Phase 8 — Maintenance planning

Before PSU replacement:

```text
verify redundancy design
vendor procedure
cluster health
capacity
approved change
maintenance window
```

### Phase 9 — Postcheck

After replacement:

```text
PSUs healthy
hardware healthy
Prism healthy
no new alerts
workloads normal
```

Đây chính là full operational thought process Day 6 đang muốn bạn học.

---

# PHẦ LVII — “THỰC TẾ ENTERPRISE” MÀ DAY 6 MUỐN HÌNH THÀNH

Một fresher thường nhìn dashboard rồi hỏi:

> Nút nào để fix?

Một Infrastructure Operations engineer cần hỏi:

> Failure nằm ở layer nào?

Sau đó:

> Evidence nào chứng minh?

Sau đó:

> Business/service impact là gì?

Sau đó:

> Resilience còn không?

Sau đó:

> Tôi có authority để xử lý không?

Sau đó mới:

> Corrective action nào phù hợp?

Đó là sự khác nhau rất lớn.

---

# PHẦ LVIII — TỔNG HỢP MENTAL MODEL DAY 6

Bạn có thể hình dung toàn bộ Day 6 bằng một stack:

```text
┌───────────────────────────────────────┐
│ Application                           │
├───────────────────────────────────────┤
│ Guest OS                              │
├───────────────────────────────────────┤
│ VM                                    │
├───────────────────────────────────────┤
│ AHV Hypervisor                        │
├───────────────────────────────────────┤
│ CVM / AOS Distributed Services        │
├───────────────────────────────────────┤
│ Nutanix Cluster                       │
├───────────────────────────────────────┤
│ Physical Node                         │
│ CPU / RAM / Disk / NIC                │
│ Controller / PSU / Fan                │
├───────────────────────────────────────┤
│ Firmware / BIOS / UEFI                │
├───────────────────────────────────────┤
│ iLO Out-of-Band Management            │
└───────────────────────────────────────┘

Management:

Prism Element → Local Cluster
Prism Central → Centralized Management

Evidence:

iLO / IML
Prism Health
Alerts
Events
Inventory
Capacity
Versions
Ticket/change references
```

---

# Day 6 — Cheat Sheet Lý thuyết

| Term                   | Ý nghĩa operational                                            |
| ---------------------- | -------------------------------------------------------------- |
| **HPE ProLiant**       | Physical server platform                                       |
| **CPU**                | Compute resource                                               |
| **DIMM**               | Physical memory module                                         |
| **RAID**               | Disk organization/fault-tolerance mechanism; không phải backup |
| **Storage Controller** | Controller quản lý physical/logical storage                    |
| **Firmware**           | Low-level software của hardware component                      |
| **BIOS/UEFI**          | Platform initialization/configuration layer                    |
| **iLO**                | Out-of-band hardware management                                |
| **Remote Console**     | Remote keyboard/video/mouse console tới physical server        |
| **Virtual Media**      | Remote attachment của boot/install/recovery media              |
| **IML**                | HPE Integrated Management Log                                  |
| **HCI**                | Hyperconverged Infrastructure                                  |
| **Node**               | Physical cluster member/resource role                          |
| **AHV**                | Nutanix hypervisor                                             |
| **CVM**                | Nutanix Controller VM / infrastructure service VM              |
| **AOS**                | Nutanix distributed infrastructure/storage platform            |
| **Prism Element**      | Local Nutanix cluster management                               |
| **Prism Central**      | Centralized/multi-cluster management                           |
| **Storage Pool**       | Logical aggregation của storage resources                      |
| **Storage Container**  | Logical storage construct                                      |
| **Resilience**         | Khả năng chịu/recover từ failure                               |
| **Degraded**           | Service có thể vẫn chạy nhưng health/redundancy đã giảm        |
| **Baseline**           | Known/approved reference state                                 |
| **Stop Condition**     | Điều kiện buộc dừng maintenance                                |
| **Escalation**         | Chuyển đúng owner khi vượt scope/quyền/chuyên môn              |

---

## Những điều bạn phải nắm chắc trước khi chuyển sang Module Thực hành

Sau toàn bộ Module Lý thuyết Day 6, bạn phải có thể tự giải thích được chuỗi sau mà không nhầm layer:

```text
HPE ProLiant physical hardware
        ↓
CPU / RAM / Disk / NIC / PSU / Fan
        ↓
Firmware / BIOS / UEFI
        ↓
iLO + IML
        ↓
Nutanix Physical Node
        ↓
AHV
        ↓
CVM / AOS
        ↓
Nutanix Cluster
        ↓
Prism Element
        ↓
Prism Central
        ↓
VM / Guest OS / Application
```

Và khi có warning, tư duy phải là:

```text
Symptom
→ Scope
→ Failure Layer
→ Impact
→ Redundancy / Resilience
→ Evidence
→ Operational Boundary
→ Remediate or Escalate
→ Verify
```

Đó là **năng lực lý thuyết cốt lõi của Day 6** mà syllabus yêu cầu trước khi sang lab: nhận diện chính xác HPE/Nutanix infrastructure, phân biệt các management/compute/platform/hardware planes, hiểu health và resilience, biết maintenance risk, biết evidence cần giữ, và đặc biệt **xác định đúng failing layer cùng correct support owner** thay vì xử lý theo phỏng đoán. Infrastructure_Operations_14_Da…
