# Day 3 — Module Lý thuyết: Windows Server Administration Foundations

Module này bám trực tiếp toàn bộ **Concept/Lecture** và **Additive Concept/Lecture** của Day 3 trong syllabus: Windows Server architecture, generations/editions/licensing, Server Core/Desktop Experience, Server Manager, Windows Admin Center, services/processes/startup, Registry, local identity/security, NTFS/share permissions, AD DS/OU/GPO, DNS/DHCP, RDP/WinRM/Firewall/networking, storage, patching, Event Logs, Task Scheduler, backup/recovery, PowerShell và operational maintenance. Phần mở rộng bắt buộc là **edition-aware operational coverage** và **audit-quality PowerShell/Windows Event evidence**. Infrastructure_Operations_14_Da…

Tôi sẽ phân biệt hai loại nội dung:

- **Theo syllabus**: kiến thức bắt buộc của Day 3.
- **Kiến thức bổ sung**: nền tảng cần thiết để hiểu đúng hoặc vận hành an toàn, nhưng không mở rộng Day 3 thành khóa Windows chuyên sâu.

---

# I. Mục tiêu sau khi học xong Module Lý thuyết

Sau phần này, bạn cần có khả năng nhìn vào một Windows Server và suy luận theo chuỗi:

```text
Server identity
    ↓
Windows generation / edition / build / installation option
    ↓
Roles & Features
    ↓
Identity / authentication / authorization
    ↓
Services / processes / scheduled tasks
    ↓
Networking / firewall / remote management
    ↓
Storage / filesystem / permissions
    ↓
Patching / backup / recovery
    ↓
Events / logs / evidence
    ↓
PowerShell administration
    ↓
Verification / troubleshooting / escalation
```

Điểm quan trọng là Day 3 **không đào tạo theo tư duy “nhớ vị trí GUI”**. Một Infrastructure Operations engineer phải hiểu **object, dependency và state** phía sau GUI.

Ví dụ:

```text
"RDP không vào được"
```

không phải một root cause.

Nó mới chỉ là symptom. Có thể nguyên nhân nằm ở:

```text
Client
→ DNS
→ Routing
→ Windows Firewall
→ RDP listener
→ Remote Desktop Services
→ User right
→ Group membership
→ NLA/authentication
→ Domain connectivity
→ GPO
```

Tư duy đó sẽ xuyên suốt toàn bộ Day 3.

---

# II. Syllabus coverage checklist — Module Lý thuyết

Theo syllabus, chúng ta phải cover đầy đủ:

1. Windows Server architecture.
2. Supported và legacy Windows Server generations.
3. Edition/licensing differences.
4. Standard.
5. Datacenter.
6. Datacenter: Azure Edition.
7. Essentials/legacy awareness.
8. Server Core.
9. Desktop Experience.
10. Server Manager.
11. Windows Admin Center awareness.
12. Windows services.
13. Processes.
14. Startup configuration.
15. Windows Registry fundamentals.
16. Local users.
17. Local groups.
18. Local Security Policies.
19. NTFS permissions.
20. Share permissions.
21. Active Directory Domain Services basics.
22. Domain accounts.
23. Organizational Units.
24. Group Policy awareness.
25. DNS role.
26. DHCP role.
27. RDP.
28. WinRM.
29. Windows Defender Firewall.
30. Windows network configuration.
31. Storage and volume management.
32. Windows Update và patch control.
33. Event Viewer.
34. Windows Event Logs.
35. Task Scheduler.
36. Backup and recovery.
37. PowerShell cmdlets.
38. PowerShell pipelines.
39. PowerShell remoting.
40. Execution policies.
41. Administrative scripting.
42. Windows Server operational maintenance.
43. Release-by-edition matrix.
44. Edition-specific operational constraints.
45. Native EVTX evidence.
46. `Get-WinEvent` evidence.
47. CSV summary/encoding awareness.
48. Event identity: Event ID, Record ID, Provider, host, timestamp.
49. Evidence hashing.
50. Service/scheduled-task/patch/reboot verification.

Đây là toàn bộ phạm vi chúng ta cần nắm trong module. Infrastructure_Operations_14_Da…

---

# III. Windows Server architecture

## 1. Windows Server là gì?

Windows Server là operating system của Microsoft hướng tới server workloads như:

- File services.
- Directory services.
- DNS/DHCP.
- Web/application hosting.
- Database workloads.
- Virtualization.
- Remote administration.
- Certificate services.
- Infrastructure management.
- Enterprise applications.

Microsoft mô tả Windows Server là nền tảng enterprise server chạy workloads trong on-premises, hybrid và cloud environments. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/get-started/overview?utm_source=chatgpt.com)

Điểm cần nhớ:

> Windows Server không đơn giản là “Windows desktop nhưng chạy trên server”.

Hai sản phẩm chia sẻ rất nhiều technology foundation, nhưng Server có:

- Server Roles/Features.
- Management model khác.
- Licensing khác.
- Support lifecycle khác.
- Workload assumptions khác.
- Security/availability requirements khác.

---

# IV. Architecture layers cần biết

Ở mức Infrastructure Operations, có thể hình dung:

```text
┌──────────────────────────────────────────┐
│ Applications / Server Roles              │
├──────────────────────────────────────────┤
│ Services / Processes / Scheduled Tasks   │
├──────────────────────────────────────────┤
│ Win32 / .NET / PowerShell / APIs         │
├──────────────────────────────────────────┤
│ Windows security / networking / storage  │
├──────────────────────────────────────────┤
│ Windows Executive / Kernel               │
├──────────────────────────────────────────┤
│ Drivers                                  │
├──────────────────────────────────────────┤
│ Hardware / Hypervisor                    │
└──────────────────────────────────────────┘
```

Không cần học Windows kernel internals sâu ở Day 3, nhưng cần hiểu distinction:

### User mode

Phần lớn applications/services chạy trong user mode.

Nếu process user-mode crash, thông thường nó không trực tiếp làm toàn bộ kernel crash.

### Kernel mode

Kernel và nhiều drivers hoạt động ở privileged mode.

Kernel/driver failure có khả năng gây system-wide failure như bugcheck/BSOD.

### Driver

Driver là software component cho phép OS tương tác hardware hoặc virtual hardware.

Trong VM:

```text
Windows Server
↓
Virtual NIC / virtual disk
↓
Hypervisor virtual device
↓
Physical infrastructure
```

Do đó một network problem đôi khi không nằm trong Windows mà ở:

- Hypervisor virtual switch.
- VLAN.
- physical network.
- cloud network.

---

# V. Một số critical Windows components

Không cần thuộc toàn bộ Windows internals, nhưng nên nhận diện:

### `System`

Kernel/system activity.

### `services.exe`

Service Control Manager liên quan lifecycle của Windows services.

### `lsass.exe`

Local Security Authority Subsystem Service.

Liên quan:

- authentication;
- local security policy;
- security tokens;
- credential/security operations.

Đây là security-critical process.

### `svchost.exe`

Service Host.

Nhiều Windows services có thể được host qua `svchost.exe`.

Điều này giải thích vì sao:

```text
Service
≠
Process
```

và đôi khi một process chứa nhiều services.

### `winlogon.exe`

Liên quan interactive logon session.

### Windows Registry

Configuration database.

### Event Log service

Quản lý Windows event infrastructure.

---

# VI. Tư duy dependency

Một application service có thể phụ thuộc:

```text
Application service
   ├── Windows service
   ├── Registry configuration
   ├── NTFS files
   ├── service account
   ├── DNS
   ├── network route
   ├── firewall
   ├── database
   ├── certificate
   └── external API
```

Do đó Infrastructure Operations không troubleshoot bằng cách nhìn một component duy nhất.

---

# VII. Windows Server generations và lifecycle

## 1. Vì sao generation quan trọng?

Một command hoặc feature có thể khác theo:

- Windows Server release.
- Edition.
- Installation option.
- Patch level.
- Role.
- Licensing.
- Organization configuration.

Ví dụ bạn không nên viết runbook:

> “Trên Windows Server thì feature X luôn tồn tại.”

Phải xác định:

```text
Product
Version
Edition
Build
Feature state
```

---

# VIII. Current versus legacy awareness

Tại thời điểm hiện tại, Microsoft ghi nhận **Windows Server 2025 là current LTSC release**. Windows Server cũng có LTSC và Annual Channel nhưng Day 3 chủ yếu tập trung estate-style Windows Server administration. [Microsoft Learn](https://learn.microsoft.com/th-th/windows/release-health/windows-server-release-info?utm_source=chatgpt.com)

Trong một enterprise estate, bạn có thể gặp:

```text
Windows Server 2025
Windows Server 2022
Windows Server 2019
Windows Server 2016
Windows Server 2012 R2
hoặc cũ hơn
```

Nhưng:

> Server vẫn tồn tại ≠ server vẫn được support.

---

# IX. Support lifecycle awareness

Ở thời điểm tháng 10/2026:

| Release             | Operational interpretation                                    |
| ------------------- | ------------------------------------------------------------- |
| Windows Server 2025 | Current LTSC                                                  |
| Windows Server 2022 | Mainstream support đã kết thúc, Extended Support còn tiếp tục |
| Windows Server 2019 | Extended Support                                              |
| Windows Server 2016 | Extended Support nhưng gần cuối lifecycle                     |
| Older generations   | Phải kiểm tra lifecycle/ESU/organization exception cụ thể     |

Microsoft hiện ghi Windows Server 2025 extended support đến 2034; Windows Server 2022 đến 2031; Windows Server 2019 đến 2029; Windows Server 2016 đến tháng 1/2027. [Microsoft Learn](https://learn.microsoft.com/en-us/lifecycle/products/windows-server-2025?utm_source=chatgpt.com)

Operationally, phải phân biệt:

```text
Technically running
        ≠
Vendor-supported
        ≠
Organization-supported
        ≠
Security-approved
```

Một Windows Server cũ vẫn chạy application không đồng nghĩa được phép tiếp tục triển khai server mới cùng release đó.

---

# X. Editions: Standard, Datacenter, Datacenter: Azure Edition, Essentials

Microsoft hiện vẫn cung cấp comparison theo Standard, Datacenter và Datacenter: Azure Edition, nhưng feature availability phụ thuộc release. [Microsoft Learn](https://learn.microsoft.com/windows-server/get-started/editions-comparison-windows-server-2022?utm_source=chatgpt.com)

## 1. Windows Server Standard

Thường nhắm tới workload có virtualization/containerization requirements thấp hơn Datacenter.

Infrastructure Operations cần biết:

- feature set;
- virtualization licensing;
- role availability;
- activation;
- support status.

Không nên quyết định edition dựa trên tên “Standard” mà phải theo:

- architecture requirement;
- licensing requirement;
- virtualization density;
- security/storage feature requirement.

---

# XI. Windows Server Datacenter

Datacenter hướng tới highly virtualized/datacenter environments và có các capabilities không giống Standard.

Operational mistake phổ biến:

> thấy feature hoạt động trên Datacenter lab → viết runbook giả định Standard cũng có.

Không được làm vậy.

Phải ghi:

```text
Validated on:
Windows Server <version>
<edition>
<build>
```

---

# XII. Datacenter: Azure Edition

Azure Edition có các capabilities gắn với Azure/hybrid scenarios mà Standard/Datacenter truyền thống không nhất thiết giống.

Day 3 không yêu cầu học toàn bộ Azure Edition internals.

Bạn chỉ cần hiểu:

- Đây là separate edition.
- Không được giả định feature parity.
- Deployment/support constraints phải tra đúng release.
- Runbook phải xác định edition trước khi apply procedure.

---

# XIII. Essentials

Syllabus yêu cầu **Essentials/legacy awareness**.

Nghĩa là:

- hiểu edition này tồn tại ở một số Windows Server generations;
- có thể gặp trong smaller-business/legacy estates;
- không giả định lifecycle, licensing hay feature set giống Standard.

Microsoft lifecycle hiện vẫn liệt kê Essentials trong Windows Server 2025 lifecycle scope, nhưng actual licensing/distribution conditions phải xác minh theo agreement và release cụ thể. [Microsoft Learn](https://learn.microsoft.com/en-us/lifecycle/products/windows-server-2025?utm_source=chatgpt.com)

---

# XIV. Release-by-edition matrix

Đây là requirement quan trọng của additive syllabus.

Một Operations team nên có matrix dạng:

| Field               | Ý nghĩa                              |
| ------------------- | ------------------------------------ |
| Release             | 2019/2022/2025...                    |
| Edition             | Standard/Datacenter/...              |
| Build               | Patch/build level                    |
| Installation option | Core/Desktop                         |
| Roles               | Roles được dùng                      |
| Management          | WAC/PowerShell/Server Manager/etc.   |
| Backup              | Approved backup method               |
| Patch               | Approved patch method                |
| Vendor status       | Support lifecycle                    |
| Organization status | Supported/exception/decommission     |
| Lab coverage        | H/S/D/T hoặc hands-on/demo/reference |

Điều quan trọng là:

> “All editions” trong training không có nghĩa learner đã cài và test mọi edition.

Syllabus yêu cầu phải ghi rõ **hands-on, demonstrated hoặc reference-only**. Infrastructure_Operations_14_Da…

---

# XV. Server Core vs Server with Desktop Experience

Đây là một foundational distinction.

Microsoft hiện xác nhận Server Core và Server with Desktop Experience vẫn là hai installation options của Standard/Datacenter đối với Windows Server 2016–2025. Server Core không có full Desktop Experience và được quản trị chủ yếu qua PowerShell, SConfig hoặc remote tools. Microsoft cũng khuyến nghị Server Core trừ khi workload có lý do cụ thể cần GUI. [Microsoft Learn](https://learn.microsoft.com/en-gb/windows-server/get-started/getting-started-with-server-core?utm_source=chatgpt.com)

---

# XVI. Server Core

## What

Minimal installation option.

Không có traditional Windows desktop shell đầy đủ.

Management thường qua:

```text
PowerShell
cmd
SConfig
RSAT
Server Manager remotely
Windows Admin Center
other remote tools
```

## Why

Smaller installed footprint có thể mang lại:

- ít components hơn;
- smaller attack surface;
- ít GUI dependencies;
- potentially fewer components requiring servicing.

## Common misconception

> Server Core = crippled Windows Server.

Sai.

Server Core vẫn hỗ trợ nhiều server roles quan trọng.

---

# XVII. Desktop Experience

Có standard graphical user interface và GUI management tools.

Phù hợp khi:

- application yêu cầu GUI components;
- local GUI administration là requirement;
- third-party software không hỗ trợ Core;
- team/runbook có dependency phù hợp.

Trade-off:

```text
More GUI components
→ larger footprint
→ more local management convenience
```

---

# XVIII. Có chuyển Core ↔ Desktop Experience được không?

Đối với các current generations được Microsoft nêu trong tài liệu hiện hành, Microsoft nói không thể đơn giản convert giữa Server Core và Desktop Experience sau installation; muốn đổi installation option cần clean installation. [Microsoft Learn](https://learn.microsoft.com/en-gb/windows-server/get-started/getting-started-with-server-core?utm_source=chatgpt.com)

Đây là lý do installation option là **design decision**, không phải cosmetic setting.

---

# XIX. Server Manager

## What

Server Manager là built-in Windows Server management console.

Nó hỗ trợ:

- Local Server overview.
- Role/Feature management.
- Server inventory.
- Services.
- Events.
- multi-server management.
- role-specific management entry points.

## Why

Có GUI overview nhanh và phù hợp với initial administration.

## Operational caution

Không nên hiểu:

```text
Server Manager shows green
=
Application definitely healthy
```

Một server có thể không có Windows warning nhưng application vẫn failure.

Verification luôn cần workload-level check.

---

# XX. Windows Admin Center

Microsoft mô tả Windows Admin Center là browser-based server administration tool, có thể triển khai local và quản trị Windows Server/cluster environments; nó bổ sung chứ không đơn giản “xóa bỏ” Server Manager/MMC. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/overview?utm_source=chatgpt.com)

Day 3 cần mức awareness.

Bạn nên hiểu:

```text
Admin workstation/browser
        ↓ HTTPS
Windows Admin Center Gateway
        ↓ management protocols
Managed server
```

Nó có thể hỗ trợ:

- Overview.
- Certificates.
- Devices.
- Events.
- Firewall.
- Local users/groups.
- Network.
- PowerShell.
- Processes.
- Registry.
- Services.
- Storage.
- Updates.

Nhưng actual functions phụ thuộc:

- WAC version.
- OS version.
- extensions.
- permissions.
- deployment architecture.

---

# XXI. Management-plane principle

Một critical concept:

> GUI, PowerShell và remote tooling chỉ là các management surfaces khác nhau đối với cùng underlying system state.

Ví dụ:

```text
services.msc
Get-Service
Server Manager
Windows Admin Center
```

đều có thể quan sát aspects của Windows services.

Do đó khi GUI lỗi, bạn vẫn phải biết cách verify bằng alternative interface.

---

# XXII. Windows services

Đây là một trong những topics quan trọng nhất.

## What

Windows service là long-running background component được Service Control Manager quản lý.

Service thường:

- không cần interactive user;
- start cùng OS hoặc theo demand;
- cung cấp system/application functionality.

---

# XXIII. Service properties quan trọng

Mỗi service có thể có:

```text
Service Name
Display Name
Status
Startup Type
Service Account
Binary Path
Dependencies
Recovery configuration
```

Ví dụ:

```text
Display Name: Print Spooler
Service Name: Spooler
```

Script/runbook thường nên dùng stable service name.

---

# XXIV. Service states

Bạn thường gặp:

```text
Stopped
Start Pending
Stop Pending
Running
Continue Pending
Pause Pending
Paused
```

Không nên chỉ nghĩ:

```text
Running / Stopped
```

Ví dụ service stuck ở `StartPending` có thể là symptom quan trọng.

---

# XXV. Startup Types

Các loại phổ biến:

### Automatic

Start trong boot/service startup process.

### Automatic (Delayed Start)

Start tự động nhưng delayed.

### Manual

Start khi được yêu cầu hoặc dependency/trigger khởi động.

### Disabled

Không được start cho tới khi configuration thay đổi.

Quan trọng:

> Manual không có nghĩa service luôn phải Stopped.

---

# XXVI. Service dependencies

Ví dụ conceptual:

```text
Application Service
      ↓
Database service
      ↓
Network
```

Nếu dependency failed:

```text
Application Service failed
```

chưa chắc root cause là application service.

Troubleshooting phải đi dependency tree.

---

# XXVII. Service account

Service có thể chạy dưới:

```text
LocalSystem
LocalService
NetworkService
virtual service account
managed service account
domain/service account
```

Không nên tùy ý đổi service account.

Đổi service identity có thể ảnh hưởng:

- filesystem access;
- registry access;
- network authentication;
- database permissions;
- certificate private-key access;
- SPN/Kerberos;
- secrets.

---

# XXVIII. Restart service — vì sao cần thận trọng?

`Restart-Service` nghe đơn giản nhưng có thể gây outage.

Before restart cần biết:

```text
Is this production?
Is restart authorized?
What depends on it?
Are there active transactions?
Is it clustered?
What is expected downtime?
Do we need logs before restart?
What is rollback/escalation?
```

L1/L2 không nên restart critical service chỉ vì “thử xem có hết lỗi không”.

---

# XXIX. Troubleshooting service chuẩn

Recommended order:

```text
1. Xác định exact symptom
2. Xác định service
3. Query current state
4. Kiểm tra startup type
5. Kiểm tra dependencies
6. Kiểm tra service account
7. Kiểm tra Event Logs
8. Kiểm tra resource/network/storage dependency
9. Kiểm tra recent change
10. Thực hiện corrective action được phê duyệt
11. Verify service
12. Verify application
```

Read-only verification trước configuration-changing action.

---

# XXX. Process fundamentals

## Process là gì?

Process là executing instance của executable.

Có:

```text
Process Name
PID
Executable
Threads
Handles
Memory
CPU
Owner
Start time
```

Một process có thể:

- là application;
- host service;
- host nhiều services.

---

# XXXI. Service vs Process

Ví dụ:

```text
Service
 ↓
svchost.exe
 ↓
PID 1234
```

Một `svchost.exe` không phải lúc nào cũng tương ứng duy nhất một service.

Khi troubleshoot high CPU:

```text
High CPU process
→ determine PID
→ identify hosted service(s)
→ correlate Event Logs
```

Không kill `svchost.exe` bừa.

---

# XXXII. Process troubleshooting

Symptoms:

- high CPU;
- high memory;
- hung application;
- process absent;
- process repeatedly crashing.

Workflow:

```text
Confirm symptom
→ identify PID
→ inspect resource usage
→ identify owner/service
→ event correlation
→ check dependency
→ obtain application-specific evidence
→ authorized recovery
```

Termination là state-changing/destructive action và có thể mất transaction/data.

---

# XXXIII. Startup configuration

Windows startup không chỉ là Startup folder.

Một server workload có thể khởi động thông qua:

- Services.
- Scheduled Tasks.
- application-specific startup.
- Registry-based startup mechanisms.
- Group Policy.
- server role behavior.

Nếu application “không tự chạy sau reboot”, phải hỏi:

```text
Nó được thiết kế start bằng mechanism nào?
```

Không mặc định service.

---

# XXXIV. Windows Registry fundamentals

## What

Registry là hierarchical configuration database của Windows và applications.

Các root hives thường gặp:

```text
HKEY_LOCAL_MACHINE
HKEY_CURRENT_USER
HKEY_CLASSES_ROOT
HKEY_USERS
HKEY_CURRENT_CONFIG
```

---

# XXXV. HKEY_LOCAL_MACHINE

Machine-wide configuration.

Hay viết tắt:

```text
HKLM
```

Có configuration liên quan:

- OS.
- services.
- software.
- hardware.
- security.

---

# XXXVI. HKEY_CURRENT_USER

Configuration của current logged-on user:

```text
HKCU
```

Điều này rất quan trọng khi troubleshoot:

> “Tại sao setting hoạt động khi Administrator login nhưng Scheduled Task lại không thấy setting đó?”

Có thể vì process đang chạy dưới account khác, do đó `HKCU` khác.

---

# XXXVII. Registry values

Common data types:

```text
REG_SZ
REG_EXPAND_SZ
REG_DWORD
REG_QWORD
REG_MULTI_SZ
REG_BINARY
```

Không nên sửa value khi chưa biết application expectation.

---

# XXXVIII. Registry operational safety

Registry modification có thể gây:

- service failure;
- boot issues;
- application failure;
- security weakening.

Workflow:

```text
Identify exact key/value
→ Verify source/runbook
→ Record original value
→ Export/backup relevant configuration
→ Change only approved value
→ Verify
→ Rollback if required
```

Không sử dụng random Internet registry fix trong production.

---

# XXXIX. Users và groups

Windows có hai identity contexts quan trọng trong Day 3:

```text
Local identity
Domain identity
```

---

# XL. Local user

Local account tồn tại trong Security Accounts Manager của host.

Ví dụ:

```text
SERVER01\labops
```

Scope chủ yếu là server đó.

---

# XLI. Domain account

Ví dụ:

```text
CORP\alice
```

hoặc UPN:

```text
alice@corp.example
```

Domain identity được centrally managed qua Active Directory Domain Services.

---

# XLII. Local vs domain account

Hai account có username text giống nhau vẫn là **different security principals**.

Ví dụ:

```text
SERVER01\admin
CORP\admin
```

không phải cùng account.

Do đó evidence luôn nên ghi fully-qualified identity.

---

# XLIII. Groups

Groups cho phép role-based permission assignment.

Thay vì:

```text
User Alice → folder
User Bob → folder
User Carol → folder
```

nên:

```text
Users
  ↓
Security Group
  ↓
Permission
```

Ví dụ:

```text
CORP\App_Readers
    ↓
NTFS Read
```

Giúp:

- audit dễ hơn;
- join/leave process rõ;
- least privilege;
- access review.

---

# XLIV. Built-in groups

Ví dụ phổ biến:

```text
Administrators
Users
Remote Desktop Users
Backup Operators
```

Membership có security impact lớn.

Đặc biệt:

```text
Administrators
```

không nên được dùng như solution mặc định cho mọi access issue.

---

# XLV. Least privilege

Quy tắc:

> Cấp đúng permission cần thiết, cho đúng identity, trong đúng thời gian, đúng scope.

Ví dụ user chỉ cần RDP:

Không nên:

```text
Add-LocalGroupMember Administrators
```

chỉ để RDP hoạt động.

Có thể requirement chỉ là Remote Desktop access plus application permissions.

---

# XLVI. Local Security Policy

Local Security Policy chứa nhiều settings liên quan:

```text
Account Policies
Local Policies
Security Options
User Rights Assignment
Audit settings
```

---

# XLVII. Password policy

Các concepts:

- Minimum password length.
- complexity.
- history.
- maximum/minimum age.

Trong domain environment, effective policy có thể do domain/GPO control.

---

# XLVIII. Account lockout

Settings có thể bao gồm:

```text
Lockout threshold
Lockout duration
Reset counter interval
```

Operational caution:

Account lockout có thể đến từ stale credential trong:

- service;
- Scheduled Task;
- mapped drive;
- application;
- another workstation.

Không reset password liên tục trước khi tìm authentication source.

---

# XLIX. User Rights Assignment

Khác với filesystem permission.

Ví dụ rights như:

```text
Log on locally
Log on as a service
Log on as a batch job
Allow log on through Remote Desktop Services
```

Một account có NTFS permission nhưng vẫn không nhất thiết được phép RDP hoặc run service.

---

# L. NTFS permissions

NTFS authorization là core topic.

Common permissions:

```text
Full Control
Modify
Read & Execute
List Folder Contents
Read
Write
```

---

# LI. Permission concepts

Cần hiểu:

### Security principal

User/group/service identity nhận permission.

### ACE

Access Control Entry.

### ACL

Access Control List.

### Inheritance

Child object nhận permission từ parent.

### Explicit permission

Permission configured directly trên object.

---

# LII. Allow và Deny

General rule:

- Allow grants rights.
- Deny explicitly denies rights.

Nhưng effective access có thể phức tạp vì:

- nhiều groups;
- inheritance;
- explicit ACE;
- share permissions;
- ownership;
- application behavior.

Operational recommendation:

> Tránh dùng explicit Deny trừ khi policy/design thực sự cần.

Deny thường làm troubleshooting khó.

---

# LIII. Effective permissions

Ví dụ:

```text
Alice
 ├─ Finance_Readers → Read
 └─ Finance_Editors → Modify
```

Effective NTFS access thường kết hợp permissions từ relevant group memberships, subject to Deny/inheritance rules.

Do đó khi user báo Access Denied:

Không chỉ kiểm tra ACL trên user.

Phải xem:

```text
Identity
→ Token/group membership
→ ACL
→ Inheritance
→ Deny
→ path
→ application access mode
```

---

# LIV. Ownership

Owner có special control relationship với securable object.

Nhưng:

> Ownership không đồng nghĩa luôn được phép đọc dữ liệu.

Permission và ownership là related nhưng separate concepts.

Take Ownership cũng là privileged action, không phải standard troubleshooting shortcut.

---

# LV. Share permissions

Khi folder được share qua SMB:

```text
Client
  ↓
SMB share
  ↓
Share permission
  ↓
NTFS permission
  ↓
File
```

Effective network access chịu cả Share và NTFS permissions.

Rule-of-thumb:

> Kết quả qua SMB bị giới hạn bởi layer restrict hơn giữa Share và NTFS.

---

# LVI. Ví dụ NTFS + Share

Giả sử:

```text
Share: Read
NTFS: Modify
```

Qua SMB, user không có full Modify chỉ vì NTFS là Modify.

Ngược lại:

```text
Share: Full Control
NTFS: Read
```

NTFS vẫn giới hạn user thành Read.

Nếu truy cập local trực tiếp:

```text
C:\Data
```

share permission không tham gia; NTFS là principal filesystem permission layer.

---

# LVII. Common permission mistakes

1. `Everyone: Full Control` để chữa Access Denied.
2. Cấp permission trực tiếp cho nhiều users thay vì groups.
3. Phá inheritance không có lý do.
4. Add user vào Administrators.
5. Không phân biệt local vs domain identity.
6. Không refresh logon token sau group membership change.
7. Không kiểm tra Share + NTFS cùng lúc.
8. Dùng Deny quá rộng.

---

# LVIII. Active Directory Domain Services basics

AD DS cung cấp centralized directory và identity services.

Các object cơ bản:

```text
Domain
User
Computer
Group
Organizational Unit
Group Policy Object
Domain Controller
```

---

# LIX. Domain

Domain là administrative/security boundary trong AD DS architecture ở mức cơ bản.

Ví dụ:

```text
corp.example
```

Có directory chứa users, computers, groups và other objects.

---

# LX. Domain Controller

Domain Controller chạy AD DS role và xử lý nhiều functions như:

- authentication;
- directory queries;
- policy-related operations;
- replication với DC khác.

Trong production, không nên coi DC như generic application server.

---

# LXI. Domain joining

Khi Windows Server join domain:

```text
Server
↓
AD computer object
↓
Domain trust relationship
```

Nó có thể:

- authenticate domain users;
- receive GPOs;
- use domain DNS integration;
- participate centralized management.

---

# LXII. Authentication vs Authorization

Rất quan trọng:

```text
Authentication
= Bạn là ai?

Authorization
= Bạn được phép làm gì?
```

Ví dụ:

```text
User đăng nhập thành công
```

chỉ chứng minh authentication.

Không chứng minh user có quyền access folder/application.

---

# LXIII. Kerberos/NTLM awareness — Kiến thức bổ sung

Trong Active Directory environments, Kerberos thường là primary authentication protocol, với NTLM còn tồn tại trong certain scenarios.

Day 3 không yêu cầu sâu về Kerberos tickets/SPNs.

Nhưng cần hiểu:

```text
DNS
Time synchronization
Domain connectivity
```

có ảnh hưởng rất lớn đến domain authentication.

---

# LXIV. Organizational Unit — OU

OU dùng để:

- organize AD objects;
- delegate administration;
- target/link policies theo design.

Ví dụ:

```text
corp.example
│
├── Servers
│   ├── Production
│   └── NonProduction
│
└── Users
```

---

# LXV. OU không phải Security Group

Đây là một interview question phổ biến.

### OU

Organization/delegation/GPO targeting.

### Security Group

Authorization/access assignment.

Không dùng OU thay group để cấp NTFS permission.

---

# LXVI. Group Policy awareness

Group Policy centrally controls configuration.

GPO có:

```text
Computer Configuration
User Configuration
```

GPO có thể tác động:

- security;
- firewall;
- Windows Update;
- scripts;
- audit settings;
- RDP;
- services;
- registry;
- password/security behavior;
- many OS settings.

---

# LXVII. GPO processing concept

Conceptual:

```text
Local
→ Site
→ Domain
→ OU hierarchy
```

Actual precedence còn chịu:

- inheritance;
- Enforced;
- Block Inheritance;
- security filtering;
- WMI filtering;
- loopback scenarios.

Day 3 chỉ cần awareness.

---

# LXVIII. Vì sao GPO quan trọng trong troubleshooting?

Scenario:

Administrator thay local firewall rule.

Một thời gian sau setting quay lại.

Possible cause:

```text
Group Policy refresh
```

Scenario khác:

User được add local permission nhưng RDP vẫn denied.

Có thể effective User Rights Assignment bị GPO control.

Do đó phải phân biệt:

```text
Local configuration
vs
Effective domain-managed configuration
```

---

# LXIX. DNS role

DNS maps names và other records.

Basic examples:

```text
server01.corp.example
→ 10.10.10.20
```

Record types cần biết:

```text
A
AAAA
CNAME
PTR
SRV
```

---

# LXX. A record

Hostname → IPv4.

```text
server01 → 10.20.30.40
```

### AAAA

Hostname → IPv6.

### CNAME

Alias → canonical name.

### PTR

IP → hostname trong reverse lookup context.

### SRV

Service location.

Rất quan trọng trong Active Directory service discovery.

---

# LXXI. DNS trong Active Directory

Domain environment phụ thuộc DNS rất mạnh.

Nếu client dùng wrong DNS server, có thể xảy ra:

- domain join failure;
- logon issue;
- GPO failure;
- DC discovery failure;
- application name resolution failure.

Một common mistake:

> Domain-joined server dùng public DNS trực tiếp làm preferred DNS.

Trong AD environment, DNS architecture phải theo organization design.

---

# LXXII. DNS troubleshooting order

```text
1. Kiểm tra configured DNS servers
2. Kiểm tra hostname/FQDN
3. Query DNS
4. Kiểm tra record
5. Kiểm tra DNS suffix
6. Kiểm tra network connectivity tới DNS server
7. Kiểm tra DNS service/server nếu thuộc scope
8. Correlate với AD/application
```

Không nên sửa `hosts` file để che DNS problem trừ controlled diagnostic/runbook.

---

# LXXIII. DHCP role

DHCP tự động cung cấp IP configuration.

Classic DORA:

```text
DHCPDISCOVER
↓
DHCPOFFER
↓
DHCPREQUEST
↓
DHCPACK
```

---

# LXXIV. DHCP scope

Scope bao gồm address range cho một subnet.

Có thể có:

```text
Address Pool
Exclusion
Reservation
Lease duration
Options
```

---

# LXXV. DHCP Options

Common options:

```text
Default Gateway
DNS Servers
DNS Domain
```

Incorrect DHCP option có thể gây widespread problem dù DHCP vẫn “running”.

Ví dụ wrong DNS option:

```text
Clients receive IP successfully
but cannot find domain resources
```

---

# LXXVI. Reservation

Reservation thường map:

```text
Client identifier / MAC-related identity
→ specific IP
```

Không đồng nghĩa static IP configured manually trên host.

---

# LXXVII. RDP — Remote Desktop Protocol

RDP cung cấp graphical remote session.

Common default service port:

```text
TCP 3389
```

Modern RDP có thể sử dụng UDP 3389 cho performance-related transport scenarios, nhưng operational validation phải dựa vào actual configuration/policy.

Không được giả định port mặc định nếu organization customizes it.

---

# LXXVIII. RDP dependency chain

Để RDP thành công:

```text
Client
 ↓
DNS/name resolution
 ↓
Network route
 ↓
Firewall
 ↓
RDP listener
 ↓
Remote Desktop Services
 ↓
Authentication
 ↓
User right
 ↓
Session creation
```

Một failure ở bất kỳ layer nào đều có thể cho symptom:

```text
Cannot RDP
```

---

# LXXIX. RDP authorization

User có thể cần:

- membership/group permission phù hợp;
- `Allow log on through Remote Desktop Services`;
- không bị Deny user right;
- server cho phép remote access.

Adding to local Administrators là overprivilege trong nhiều trường hợp.

---

# LXXX. Network Level Authentication — bổ sung

NLA thực hiện authentication sớm hơn trước khi full RDP session được tạo.

Không disable NLA như default troubleshooting workaround.

Nếu NLA failure:

Investigate:

- identity;
- domain;
- time;
- credential;
- certificate/TLS context;
- policy.

---

# LXXXI. WinRM

Windows Remote Management là Microsoft implementation của WS-Management và là foundation cho traditional Windows PowerShell remoting.

Microsoft xác nhận Windows PowerShell remoting trên Windows sử dụng WinRM; Windows Server từ Server 2012 trở lên thường được enabled for remoting by default, mặc dù organization policy có thể thay đổi configuration. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements?view=powershell-7.6&utm_source=chatgpt.com)

---

# LXXXII. WinRM architecture

```text
Administrator
    ↓
PowerShell
    ↓
WinRM client
    ↓
Network
    ↓
WinRM listener
    ↓
Remote endpoint
    ↓
PowerShell session
```

Dependencies:

- service;
- listener;
- firewall;
- authentication;
- authorization;
- endpoint configuration.

---

# LXXXIII. WinRM ports awareness

Common defaults:

```text
HTTP 5985
HTTPS 5986
```

Nhưng environment có thể customize.

HTTPS không tự động có nghĩa application payload security model đơn giản hơn; actual authentication/encryption behavior phải hiểu theo deployment.

Không gửi password/token trong evidence.

---

# LXXXIV. RDP vs WinRM

### RDP

Interactive GUI session.

### WinRM/PowerShell Remoting

Command/object-based remote administration.

Operationally, PowerShell remoting thường:

- reproducible hơn;
- scriptable;
- dễ record commands;
- phù hợp multi-server administration.

---

# LXXXV. Windows Defender Firewall

Windows Firewall là host firewall.

Nó kiểm soát traffic dựa trên:

- direction;
- profile;
- protocol;
- ports;
- application/service;
- IP addresses;
- interface characteristics.

Microsoft hiện ghi rõ Windows Firewall có ba network profiles: Domain, Private và Public. [Microsoft Learn](https://learn.microsoft.com/windows/security/operating-system-security/network-security/windows-firewall/?utm_source=chatgpt.com)

---

# LXXXVI. Domain Profile

Áp dụng khi system domain-joined nhận diện domain network theo Windows networking/domain detection.

Không phải “network ở office nên tự nhiên là Domain”.

---

# LXXXVII. Private Profile

Network được classified là trusted/private theo configuration.

---

# LXXXVIII. Public Profile

Dùng cho untrusted/public network context, thường restrictive hơn.

---

# LXXXIX. Firewall Rule components

Rule có thể gồm:

```text
Direction
Action
Protocol
Local Port
Remote Port
Local Address
Remote Address
Program
Service
Profile
Interface
```

---

# XC. Inbound vs Outbound

### Inbound

Traffic đi vào server.

### Outbound

Traffic originating từ server.

Nếu application không kết nối được external database, inbound rule trên app server có thể không phải nơi đầu tiên cần sửa.

Phải xác định flow direction.

---

# XCI. Firewall troubleshooting principle

Không:

```text
Disable firewall
→ test
→ quên bật lại
```

Preferred:

```text
Inspect profiles
→ inspect rules
→ verify listening service
→ test path
→ create narrow authorized exception if required
→ verify
```

---

# XCII. Network configuration

Bạn cần hiểu objects:

```text
NIC
IP address
Subnet prefix
Default gateway
DNS
Route
TCP/UDP port
Connection
```

---

# XCIII. NIC

Network Interface Card/Adapter.

Trong VM có thể là virtual NIC.

State có thể:

```text
Up
Down
Disabled
Disconnected
```

---

# XCIV. IP address và prefix

Ví dụ:

```text
10.10.20.15/24
```

Network:

```text
10.10.20.0/24
```

Phải hiểu enough subnetting để biết target cùng hay khác subnet.

---

# XCV. Default gateway

Used khi destination không nằm trong directly connected route và routing table chọn default route.

Wrong gateway có thể tạo symptom:

```text
Can reach same subnet
Cannot reach other networks
```

---

# XCVI. Routing table

Windows quyết định packet destination dựa vào routes.

Không nên chỉ kiểm tra `ping`.

Một server có thể có:

- multiple NICs;
- multiple gateways;
- static routes;
- VPN routes;
- virtual adapters.

Route selection có thể là root cause.

---

# XCVII. DNS client configuration

Network stack có thể tốt nhưng DNS sai.

Troubleshooting phải tách:

```text
IP connectivity
vs
Name resolution
```

Ví dụ:

```text
Test-NetConnection 10.1.1.20 -Port 443
```

success nhưng hostname thất bại → nghi DNS/name resolution.

---

# XCVIII. TCP/UDP port

Port number không phải service guarantee.

Ví dụ:

```text
TCP 443 listening
```

không chứng minh application healthy.

Nó chỉ chứng minh có listener/connection possibility ở transport layer.

---

# XCIX. Network troubleshooting hierarchy

Một flow tốt:

```text
1. Adapter state
2. IP configuration
3. Same-subnet connectivity
4. Gateway/routes
5. DNS
6. Target IP
7. Target port
8. Local firewall
9. Remote firewall/network firewall
10. Application/service
```

---

# C. Storage architecture

Windows storage có nhiều layers:

```text
Physical / Virtual Disk
       ↓
Disk
       ↓
Partition
       ↓
Volume
       ↓
Filesystem
       ↓
Drive Letter / Mount Point
       ↓
Files/Directories
```

---

# CI. Disk

OS-visible storage device.

Trong VM có thể là virtual disk backed by hypervisor storage.

Do đó Windows `Disk Healthy` không chứng minh physical array healthy.

Layering rất quan trọng.

---

# CII. Partition

Subdivision của disk.

Common partition styles:

```text
GPT
MBR
```

Modern systems thường dùng GPT nhiều hơn.

---

# CIII. Volume

Logical storage area formatted with filesystem và mounted for use.

Có thể mount bằng:

```text
Drive letter
```

ví dụ:

```text
D:
```

hoặc mount point.

---

# CIV. Filesystems

Day 3 tập trung chủ yếu NTFS awareness.

Có thể gặp ReFS tùy workload/edition/use case, nhưng syllabus không yêu cầu ReFS sâu.

Filesystem quyết định:

- permissions;
- metadata;
- files/directories;
- supported features.

---

# CV. Storage capacity

Phải phân biệt:

```text
Disk capacity
Partition size
Volume size
Free space
Filesystem free space
```

Một physical datastore có capacity nhưng guest volume vẫn có thể full.

---

# CVI. Low disk symptoms

Disk space exhaustion có thể gây:

- service failure;
- database failure;
- Event Log issue;
- update failure;
- backup failure;
- temp-file failure;
- OS instability.

Tuy nhiên không nên delete random files.

Correct flow:

```text
Measure
→ Identify growth
→ Determine ownership
→ Apply retention/cleanup runbook
→ Verify
```

---

# CVII. Windows Update và patch control

Patch management là operational workflow, không phải chỉ chạy update.

Goals:

- remediate vulnerabilities;
- fix bugs;
- maintain vendor support;
- satisfy compliance.

Risks:

- reboot;
- service interruption;
- compatibility issues;
- failed update;
- boot failure.

---

# CVIII. Patch lifecycle

Một production-style process:

```text
1. Change/ticket
2. Scope identification
3. Patch applicability
4. Compatibility assessment
5. Backup/recovery readiness
6. Maintenance window
7. Pre-check
8. Patch deployment
9. Reboot if required
10. Post-check
11. Application verification
12. Monitoring
13. Evidence
14. Change closure
```

---

# CIX. Reboot awareness

Patch installed không đồng nghĩa patch fully effective nếu pending reboot required.

Operations phải record:

```text
Patch installed
Reboot required?
Reboot completed?
OS returned?
Services returned?
Application returned?
Monitoring healthy?
```

---

# CX. Patch failure troubleshooting

Check:

- Windows Update history.
- relevant Event Logs.
- disk space.
- reboot status.
- servicing stack/component state.
- network/update source.
- WSUS/management tooling if used.
- error code.

Không repeatedly retry blindly.

---

# CXI. Event Viewer

Event Viewer là GUI interface để xem Windows Event Logs.

Common logs:

```text
Application
Security
System
```

Ngoài ra:

```text
Applications and Services Logs
```

---

# CXII. System log

Thường chứa events từ:

- drivers;
- Windows services;
- operating-system components.

Useful cho:

- service issues;
- driver failures;
- boot;
- networking/system events.

---

# CXIII. Application log

Application hoặc framework components có thể write event tại đây.

Không phải mọi application đều dùng Application log.

---

# CXIV. Security log

Chứa security/audit events nếu audit policy tương ứng được enabled.

Important distinction:

> Không thấy security event không chứng minh activity không xảy ra.

Có thể:

- audit category không enabled;
- event đã rotate;
- collection gap;
- insufficient permissions;
- wrong timeframe.

---

# CXV. Event fields

Critical fields:

```text
LogName
Provider
Event ID
Level
TimeCreated
Record ID
Machine/Computer
User/Security context
Message
```

---

# CXVI. Event ID

Event ID giúp classify event type nhưng không đủ một mình.

Same numeric Event ID từ different providers có thể mang meaning khác.

Nên correlate:

```text
Provider + Event ID
```

---

# CXVII. Record ID

Record ID identify record position/id trong log channel context.

Rất hữu ích để trace exact event.

Syllabus yêu cầu bảo tồn Record ID trong evidence. Infrastructure_Operations_14_Da…

---

# CXVIII. Timestamp

Timestamp phải xem cùng:

```text
Timezone
Clock synchronization
Requested incident window
```

Nếu server time sai, correlation với network/cloud logs sẽ sai.

---

# CXIX. Event correlation

Một service incident có thể có:

```text
10:02 service error
10:02 application dependency timeout
10:03 SCM reports service termination
10:04 monitoring alert
10:06 administrator login
10:08 service restart
```

Một investigator cần reconstruct timeline thay vì chụp một screenshot duy nhất.

---

# CXX. Native EVTX evidence

Syllabus yêu cầu export native Windows event data, ví dụ EVTX.

Why?

EVTX giữ richer event structure hơn CSV/screenshot.

Concept:

```text
Native EVTX
= preservation/review artifact

CSV
= analysis/report artifact

Screenshot
= visual supplemental artifact
```

`wevtutil` có khả năng query, export và archive logs trên supported Windows Server versions. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/wevtutil?utm_source=chatgpt.com)

---

# CXXI. Không clear logs

`wevtutil` có capability clear log, nhưng trong Day 3 investigation/evidence workflow:

> Không clear Event Logs.

Clearing có thể:

- destroy evidence;
- violate policy;
- hinder incident investigation.

Syllabus explicitly yêu cầu negative tests mà **không clearing logs**. Infrastructure_Operations_14_Da…

---

# CXXII. Empty results vs collection failure

Đây là critical audit distinction.

Nếu query trả zero:

Có thể là:

```text
A. Không có event matching
B. Wrong log
C. Wrong Event ID
D. Wrong timezone
E. Wrong timeframe
F. Permission denied
G. Channel unavailable
H. Query malformed
I. Logs rotated
```

Do đó:

```text
0 records
```

không tự động có nghĩa:

```text
No incident occurred
```

---

# CXXIII. Task Scheduler

Task Scheduler cho phép execution theo:

- time;
- schedule;
- boot;
- logon;
- event;
- other triggers.

Task gồm:

```text
Trigger
Action
Principal
Conditions
Settings
History
```

---

# CXXIV. Trigger

Khi nào task chạy.

Ví dụ:

```text
Daily 02:00
At startup
At logon
On event
```

---

# CXXV. Action

Task thực hiện gì:

```text
powershell.exe
script
program
command
```

---

# CXXVI. Principal

Identity chạy task.

Đây là source của nhiều failures.

Ví dụ interactive test:

```text
C:\Scripts\health.ps1
```

chạy thành công dưới admin.

Scheduled Task chạy bằng service account lại fail vì:

- account khác;
- permission khác;
- profile khác;
- mapped drive không tồn tại;
- execution policy context;
- working directory;
- network credentials.

---

# CXXVII. Scheduled Task failure symptoms

Check:

```text
Task status
Last Run Time
Last Run Result
Task History
PowerShell/application logs
Output file
Exit code
```

Không kết luận success chỉ từ việc task trigger đã fire.

---

# CXXVIII. Backup và recovery

Backup là tạo recoverable copy/protection point.

Recovery/restore là sử dụng backup đó để khôi phục.

Quan trọng:

```text
Backup successful
≠
Recovery proven
```

---

# CXXIX. Backup questions

Một backup design phải trả lời:

```text
What is protected?
Where is backup stored?
How often?
Retention?
Who can restore?
Encryption?
Application consistency?
How is restore tested?
```

---

# CXXX. File-level vs system/application recovery

Có nhiều layers:

- file/folder restore;
- application data restore;
- system state;
- full server/VM recovery.

Day 3 chỉ cần foundation; exact backup product phụ thuộc environment.

---

# CXXXI. Recovery validation

Sau restore cần verify:

```text
File exists?
Correct size?
Correct content?
Application can open it?
Permissions correct?
Timestamp expected?
```

Nếu có hash baseline:

```text
Hash restored data
→ compare expected hash
```

---

# CXXXII. PowerShell fundamentals

PowerShell là management shell, scripting language và automation framework.

Một khác biệt rất quan trọng:

> PowerShell pipeline truyền **objects**, không chỉ raw text.

---

# CXXXIII. Cmdlet naming

Convention:

```text
Verb-Noun
```

Ví dụ:

```powershell
Get-Service
Get-Process
Get-EventLog
Get-WinEvent
Get-ComputerInfo
Get-NetAdapter
Get-Volume
```

Common verbs:

```text
Get
Set
New
Remove
Start
Stop
Restart
Test
Export
Import
```

Tên command thường cho clue về risk:

```text
Get-*    → thường read-only
Set-*    → change
Remove-* → potentially destructive
Restart-* → state-changing
```

Không tuyệt đối nhưng rất hữu ích.

---

# CXXXIV. Discoverability

PowerShell hỗ trợ self-discovery:

```powershell
Get-Command
Get-Help
Get-Member
```

Ví dụ:

```powershell
Get-Help Get-Service -Full
```

Operationally, đọc help tốt hơn copy command từ Internet không hiểu parameters.

---

# CXXXV. PowerShell Objects

Ví dụ:

```powershell
Get-Service
```

trả service objects.

Bạn có thể xem properties:

```powershell
Get-Service | Get-Member
```

Điều này khác với parsing textual table output.

---

# CXXXVI. Pipeline

Ví dụ conceptual:

```powershell
Get-Service |
    Where-Object Status -eq 'Running' |
    Select-Object Name, Status
```

Flow:

```text
Get objects
→ filter objects
→ select properties
→ output
```

---

# CXXXVII. Filtering

Tốt nhất filter càng sớm và đúng source càng tốt.

Ví dụ event collection:

Nếu có native filter support:

```powershell
Get-WinEvent -FilterHashtable ...
```

thường tốt hơn lấy hàng triệu events rồi `Where-Object`.

Lý do:

- performance;
- memory;
- clearer query;
- evidence reproducibility.

---

# CXXXVIII. Formatting vs data

Một PowerShell principle rất quan trọng:

```powershell
Format-Table
```

dành cho display.

Không nên format data rồi mới export để machine-process.

Preferred conceptual:

```text
Objects
→ Select properties
→ Export-Csv
```

không:

```text
Objects
→ Format-Table
→ Export-Csv
```

---

# CXXXIX. PowerShell remoting

Common commands:

```powershell
Enter-PSSession
Invoke-Command
New-PSSession
```

### `Enter-PSSession`

Interactive remote shell.

### `Invoke-Command`

Execute commands remotely, phù hợp scripting/multiple hosts.

---

# CXL. Remote identity awareness

Trong evidence cần biết:

```text
Command executed from where?
Command executed on which server?
As which identity?
At what time?
```

Vì:

```powershell
hostname
```

trên management workstation khác kết quả command chạy qua remoting.

---

# CXLI. PowerShell execution policies

Microsoft định nghĩa execution policy là safety feature kiểm soát điều kiện script/config files được chạy, nhưng **không phải security boundary**. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.5&utm_source=chatgpt.com)

Common policies:

```text
Restricted
RemoteSigned
AllSigned
Unrestricted
Bypass
Undefined
```

---

# CXLII. Policy scopes

Scopes gồm:

```text
MachinePolicy
UserPolicy
Process
CurrentUser
LocalMachine
```

Group Policy-configured policies có precedence cao. [Microsoft Learn](https://learn.microsoft.com/en-gb/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.3&utm_source=chatgpt.com)

---

# CXLIII. Operational mistake: Set-ExecutionPolicy Bypass

Không nên gặp error rồi:

```powershell
Set-ExecutionPolicy Bypass -Scope LocalMachine
```

như một generic fix.

Bạn phải hỏi:

```text
Script source trusted?
Organization policy?
Is script signed?
Why is execution blocked?
Does GPO control it?
Can we use approved narrower scope?
```

---

# CXLIV. PowerShell security

Execution Policy không thay thế:

- least privilege;
- code review;
- code signing;
- application control;
- endpoint protection;
- script logging;
- secrets management.

---

# CXLV. Administrative scripting

Day 3 scripting ở mức foundation.

Một admin script tốt nên có:

```text
Clear inputs
Read-only default where possible
Validation
Error handling
Meaningful exit result
Logging
No embedded secrets
Deterministic output
Timestamp
Hostname
```

---

# CXLVI. Health-check script concept

Ví dụ:

```text
Host
├── Hostname
├── OS version
├── Uptime
├── Critical service state
├── Disk space
├── IP configuration
└── Timestamp
```

Output có thể dùng:

- console;
- CSV;
- JSON;
- log file.

---

# CXLVII. Error handling awareness

Một dangerous script:

```powershell
Restart-Service SomeService
"Success"
```

Nếu command fail nhưng script vẫn in Success thì evidence sai.

Cần kiểm tra:

- terminating/non-terminating errors;
- try/catch;
- `$?`;
- `$LASTEXITCODE` khi gọi native command;
- exit code;
- actual postcondition.

Day 12 sẽ đi sâu hơn.

---

# CXLVIII. Windows Server operational maintenance

Đây là điểm tổng hợp toàn bộ Day 3.

Một operations engineer không chỉ “biết command”.

Bạn phải biết workflow.

---

# CXLIX. Pre-maintenance

Trước change:

```text
Ticket/change ID
Scope
Asset/hostname
Environment
Owner
Maintenance window
Expected impact
Dependencies
Backup/recovery readiness
Rollback/backout
Approvals
```

---

# CL. Pre-check

Capture baseline:

```text
OS/version
Uptime
Services
Application state
CPU/memory
Storage
Network
Events
Backup state
Monitoring state
```

Không baseline → khó chứng minh change gây hay không gây issue.

---

# CLI. Implementation

Chỉ làm approved scope.

Nếu ticket nói:

```text
Restart Service A
```

không tự tiện:

```text
Reboot server
```

vì “restart không được”.

Scope escalation cần được tôn trọng.

---

# CLII. Post-check

Không chỉ check technical action.

Ví dụ service restart:

```text
Service Running
```

chưa đủ.

Cần:

```text
Service Running
+
Correct port listening
+
Application functional
+
No new critical event
+
Monitoring recovered
```

---

# CLIII. Rollback vs recovery

Hai khái niệm liên quan nhưng không đồng nhất.

### Rollback/backout

Đưa configuration/change về previous approved state.

### Recovery

Khôi phục service/data/function sau failure.

Ví dụ patch đôi khi không có simple rollback. Có thể recovery cần restore/rebuild/vendor procedure.

---

# CLIV. Escalation boundary

L1/L2 thường có thể:

- read status;
- collect evidence;
- execute approved runbook;
- restart approved noncritical service;
- validate connectivity;
- apply authorized known remediation.

Phải escalate khi:

- unknown root cause;
- privileged access ngoài scope;
- Domain Controller-sensitive issue;
- data-loss risk;
- boot failure;
- repeated service crash;
- registry/security change ngoài runbook;
- patch failure requiring servicing repair;
- production firewall/GPO change;
- security incident;
- backup cannot restore;
- evidence suggests compromise.

---

# CLV. Operational safety: Read-only vs changing commands

Đây là skill nên hình thành ngay.

### Mostly read-only examples

```powershell
Get-Service
Get-Process
Get-ComputerInfo
Get-NetAdapter
Get-NetIPConfiguration
Get-NetRoute
Get-Disk
Get-Volume
Get-WinEvent
```

### State/config changing examples

```powershell
Restart-Service
Stop-Service
Set-Service
New-LocalUser
Add-LocalGroupMember
Set-NetFirewallRule
New-NetFirewallRule
Initialize-Disk
Format-Volume
Set-ExecutionPolicy
```

Không được đánh đồng mức risk.

---

# CLVI. Security implications xuyên suốt Day 3

Mọi Windows administration phải xem identity và privilege.

Nguyên tắc:

```text
Authenticate
→ authorize
→ execute
→ audit
```

Không dùng:

- shared admin password;
- plaintext credentials trong script;
- blanket Full Control;
- firewall disabled;
- local Administrators như troubleshooting shortcut.

---

# CLVII. UAC — Kiến thức bổ sung cần biết

User Account Control tách standard user context và elevated administrative operations.

Một account thuộc Administrators không có nghĩa mọi process luôn chạy unrestricted admin token.

Do đó có thể gặp:

```text
Command works "Run as administrator"
but fails in normal shell
```

Đây là expected security behavior chứ không nhất thiết là bug.

---

# CLVIII. Local admin ≠ Domain Admin

Rất quan trọng:

```text
Local Administrator
```

quản trị host.

```text
Domain Admin
```

là highly privileged domain identity/group context.

Không dùng Domain Admin làm daily server administration nếu không cần.

---

# CLIX. Evidence handling trong Windows Server operations

Day 3 additive syllabus yêu cầu evidence có chất lượng audit.

Không chỉ:

> “Tôi đã chạy command.”

Mà phải chứng minh:

```text
What
Where
When
Who
Why
How
Result
```

---

# CLX. Minimum evidence metadata

Một evidence item lý tưởng có:

```text
Evidence ID
Hostname / Asset ID
Environment
Windows version/build
Edition
Collector
Timestamp
Timezone
Ticket/change ID
Command/query
Time window
Filter
Record count
Output filename
SHA-256
Limitations/errors
```

Điều này nối với evidence contract chung của toàn chương trình. Infrastructure_Operations_14_Da…

---

# CLXI. Raw vs derived evidence

Ví dụ:

```text
System.evtx
```

là native/source-style export.

Sau đó tạo:

```text
System_Filtered.csv
```

CSV là derived artifact.

Không overwrite EVTX bằng filtered CSV.

Structure:

```text
Original/native
      ↓
Working copy
      ↓
Filtered/derived report
```

Mỗi derived artifact có thể có riêng:

- hash;
- transformation;
- script version;
- parent evidence ID.

---

# CLXII. Hash có chứng minh evidence là thật không?

Không.

SHA-256 chứng minh:

> file hiện tại giống/khác baseline hash đã recorded.

Nó **không** tự chứng minh:

- source đúng;
- query đúng;
- timeframe đầy đủ;
- log không thiếu;
- event authentic trước lúc export.

Syllabus nhấn mạnh distinction này xuyên suốt chương trình. Infrastructure_Operations_14_Da…

---

# CLXIII. `Get-WinEvent` evidence model

Khi extract events, nên giữ tối thiểu:

```text
TimeCreated
Id
RecordId
ProviderName
MachineName
LevelDisplayName
Message
```

Tùy audit requirement còn có:

- UserId;
- LogName;
- Process/thread metadata;
- raw XML.

---

# CLXIV. Time-bounded extraction

Thay vì:

```text
Export entire Security log
```

cho incident 10 phút, nên define:

```text
Start time
End time
Timezone
Event IDs
Provider
Hostname
```

Benefits:

- scope;
- privacy;
- faster analysis;
- reproducibility.

---

# CLXV. Native EVTX + CSV

Best mental model:

```text
EVTX = forensic/audit-friendly native event container
CSV  = human/data analysis derivative
```

Không dùng CSV để giả vờ rằng toàn bộ original event metadata luôn được preserved.

---

# CLXVI. CSV encoding

Syllabus yêu cầu explicit encoding.

Why?

Một CSV có:

```text
Vietnamese
Japanese
Unicode usernames
application messages
```

có thể bị corruption nếu encoding không được xác định.

Day 12 sẽ học UTF-8/BOM/UTF-16/CP932 sâu hơn.

Trong Day 3 chỉ cần hiểu:

> “CSV file” chưa mô tả đủ format; phải biết encoding.

---

# CLXVII. Negative evidence tests

Syllabus yêu cầu hiểu ba cases:

### Unavailable channel

Log channel không có/không enabled.

### Access denied

Collector không có quyền đọc.

### Empty result

Query valid nhưng zero matches.

Ba tình huống này **không giống nhau**.

Một report tốt phải phân biệt.

---

# CLXVIII. Service verification evidence

Nếu restart service:

Evidence nên có:

```text
Before:
service = Running/Stopped
timestamp

Action:
ticket reference
authorized command

After:
service = Running
PID if relevant
application check
related event
timestamp
```

Không chỉ screenshot “Running”.

---

# CLXIX. Scheduled Task evidence

Capture:

```text
Task name
Principal
Trigger
Action
Last Run Time
Last Run Result
Output
Timestamp
```

Nếu task health-check script:

Need both:

```text
Task executed
and
Script result valid
```

---

# CLXX. Patch/reboot evidence

Record:

```text
Pre-build
Applicable updates
Installed update/KB
Installation result
Reboot required
Reboot time
Post-build
Service health
Application health
```

---

# CLXXI. Common Windows Server failure patterns

Bây giờ kết nối các concepts.

## Pattern 1 — Service won't start

Possible causes:

```text
Disabled startup
Dependency down
Bad service account/password
Missing file
Port conflict
Permission
Registry/config error
Disk full
Application failure
```

Do not immediately reinstall.

---

# CLXXII. Pattern 2 — Access Denied

Potential layers:

```text
Wrong account
Group membership
Share permission
NTFS permission
Deny ACE
UAC
User Rights Assignment
GPO
Application ACL
```

“Access Denied” không tự chứng minh filesystem permission problem.

---

# CLXXIII. Pattern 3 — Server reachable by IP but not hostname

Likely investigate:

```text
DNS
DNS suffix
record
client resolver config
```

Không focus firewall first nếu IP path works.

---

# CLXXIV. Pattern 4 — RDP timeout

Investigate:

```text
DNS
Route
Firewall
Port/listener
RDP service
```

Timeout khác authentication denied.

---

# CLXXV. Pattern 5 — RDP says user not allowed

Investigate:

```text
Identity
Remote Desktop Users
User Rights Assignment
Deny right
GPO
```

Không ưu tiên network nếu connection đã tới authentication stage.

---

# CLXXVI. Pattern 6 — Scheduled Task works manually but fails scheduled

Likely:

```text
Different user context
Path
working directory
permissions
mapped drive
execution policy
credential
environment variable
```

---

# CLXXVII. Pattern 7 — Patch installed but application down

Check:

```text
Reboot complete?
Service status?
Dependencies?
Port?
Certificate?
Application logs?
Windows events?
Configuration change?
```

Do not close change merely because update UI says Success.

---

# CLXXVIII. Pattern 8 — Disk full

Investigate:

```text
Which volume?
What grew?
Logs?
Temp?
Database?
Backup?
Crash dump?
Application output?
```

Never delete unknown data blindly.

---

# CLXXIX. Pattern 9 — GPO overrides local setting

Symptoms:

```text
Admin fixes setting
↓
Works
↓
Later reverts
```

Investigate effective policy.

---

# CLXXX. Pattern 10 — Security log query has zero results

Need distinguish:

```text
No event
Wrong filter
Wrong timezone
Auditing not enabled
Log rotated
Insufficient access
```

Never state “nothing happened” solely from an empty result.

---

# CLXXXI. Operational troubleshooting methodology

For almost every Windows issue:

```text
1. Define symptom precisely
2. Define affected asset/user/application
3. Define start time
4. Check recent changes
5. Check basic dependencies
6. Collect read-only evidence
7. Narrow failure layer
8. Form hypothesis
9. Test lowest-risk hypothesis
10. Apply authorized remediation
11. Verify technical state
12. Verify business/application function
13. Document
14. Rollback/escalate if required
```

---

# CLXXXII. Evidence-before-change principle

Trong nhiều incidents:

```text
Collect evidence
BEFORE
Restart/Reboot
```

vì restart có thể:

- reset process state;
- change PIDs;
- clear transient symptoms;
- rotate/log new events;
- destroy useful memory/runtime clues.

Không phải lúc nào cũng trì hoãn recovery, nhưng phải cân bằng:

```text
Service restoration urgency
vs
Evidence preservation
```

theo incident/runbook.

---

# CLXXXIII. When to restart server?

Reboot là broad-impact action.

Không nên:

```text
Unknown issue
→ reboot
```

như first troubleshooting step.

Reboot hợp lý khi:

- patch requires it;
- runbook calls for it;
- approved recovery plan;
- known OS condition;
- vendor procedure.

Nếu production:

- approval;
- maintenance/incident authority;
- dependency awareness;
- monitoring;
- post-check.

---

# CLXXXIV. Escalation examples

### Escalate tới AD team khi

- domain trust issue;
- GPO requires modification;
- DC health issue;
- domain-level identity problem.

### Escalate tới Network team khi

- route/VLAN/upstream firewall evidence indicates external network layer.

### Escalate Security khi

- suspicious authentication;
- unauthorized privilege change;
- malware/compromise indicators;
- unexpected Security log manipulation.

### Escalate Application team khi

Windows service and infrastructure healthy nhưng application-specific failures persist.

### Escalate Storage/Virtualization team khi

Guest sees disk/I/O issue suspected below Windows layer.

---

# CLXXXV. Windows Server operational mindset

Bạn nên học Day 3 theo năm questions.

## 1. “What object am I dealing with?”

```text
User?
Group?
Service?
Process?
NIC?
Disk?
Volume?
Event?
Scheduled Task?
GPO?
```

## 2. “What state is it in?”

```text
Running?
Stopped?
Enabled?
Disabled?
Connected?
Full?
Denied?
Healthy?
```

## 3. “What dependencies does it have?”

```text
DNS?
Network?
Permission?
Storage?
Account?
Certificate?
Service?
```

## 4. “What evidence proves my conclusion?”

```text
Command output?
Event?
Log?
Test result?
Configuration?
```

## 5. “What is the safest authorized next action?”

```text
Read more?
Test?
Restart?
Change?
Rollback?
Escalate?
```

---

# CLXXXVI. Connection giữa Linux Day 1–2 và Windows Day 3

Rất nhiều concepts tương đương, dù tools khác.

| Linux/RHEL             | Windows Server                                 |
| ---------------------- | ---------------------------------------------- |
| `systemd`              | Service Control Manager / Windows Services     |
| `journalctl`           | Windows Event Logs / `Get-WinEvent`            |
| users/groups           | local/domain users/groups                      |
| `chmod/chown`          | NTFS ACL/ownership                             |
| `sudo`                 | administrative elevation/UAC/privileged groups |
| `cron`                 | Task Scheduler                                 |
| `ip`                   | NetTCPIP PowerShell cmdlets                    |
| `ss`                   | `Get-NetTCPConnection`                         |
| `firewalld`            | Windows Defender Firewall                      |
| `/etc/*` configuration | Registry + files + policy                      |
| SSH                    | RDP/WinRM/PowerShell Remoting                  |
| shell scripts          | PowerShell scripts                             |

Nhưng không nên cố map 1:1.

Ví dụ:

```text
sudo
```

không hoàn toàn tương đương UAC.

`systemd` unit model cũng không giống hoàn toàn Windows services.

---

# CLXXXVII. Day 3 edition-awareness requirement

Syllabus đặc biệt nhấn mạnh rằng Windows training phải tránh claim:

> “Tôi biết mọi Windows Server edition.”

Thay vào đó:

```text
Windows Server 2025 Standard:
reference/demonstrated/hands-on

Windows Server 2025 Datacenter:
...

Windows Server 2022:
...
```

Điều này phản ánh real enterprise operations: version/edition matters. Infrastructure_Operations_14_Da…

---

# CLXXXVIII. Current Microsoft management position

Microsoft hiện coi Windows Admin Center, local management tools, System Center và Azure-related management là các management approaches của Windows Server. Server Manager/MMC vẫn tồn tại, còn Windows Admin Center là modern browser-based management surface. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/overview?utm_source=chatgpt.com)

Vì vậy skill target của Day 3 không phải:

```text
“Biết click Server Manager”
```

mà là:

```text
Can administer Windows Server
through multiple management surfaces
and verify underlying state.
```

---

# CLXXXIX. Ports/protocol awareness của Day 3

Không cần memorise mọi port nhưng nên biết core ones:

| Function           | Protocol/Port thường gặp  |
| ------------------ | ------------------------- |
| DNS                | TCP/UDP 53                |
| DHCP server/client | UDP 67/68                 |
| SMB                | TCP 445                   |
| RDP                | TCP/UDP 3389 commonly     |
| WinRM HTTP         | TCP 5985                  |
| WinRM HTTPS        | TCP 5986                  |
| Kerberos           | TCP/UDP 88                |
| LDAP               | TCP/UDP 389 depending use |
| LDAPS              | TCP 636                   |

Lưu ý:

- Environment có thể customize.
- AD có nhiều additional ports/dynamic RPC dependencies.
- Table này chỉ là foundation, không phải complete firewall matrix.

---

# CXC. Core PowerShell commands cần nhận diện về mặt lý thuyết

Không cần lab ngay, nhưng phải biết command family:

```powershell
Get-ComputerInfo

Get-Service
Get-Process

Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember

Get-NetAdapter
Get-NetIPAddress
Get-NetIPConfiguration
Get-NetRoute
Get-NetTCPConnection
Get-NetFirewallProfile
Get-NetFirewallRule

Get-Disk
Get-Partition
Get-Volume

Get-WinEvent

Get-ScheduledTask

Get-WindowsFeature

Get-ExecutionPolicy
Get-PSSessionConfiguration
```

Nguyên tắc:

```text
Get-*
```

thường phù hợp phase discovery/read-only.

Sau đó mới tới `Set/New/Remove/Restart`.

---

# CXCI. `Get-WinEvent` vs Event Viewer

Event Viewer:

- visual;
- useful manual exploration.

`Get-WinEvent`:

- repeatable;
- scriptable;
- filterable;
- exportable;
- stronger for evidence workflow.

Một operator tốt dùng được cả hai.

---

# CXCII. GUI vs PowerShell verification

Ví dụ Service:

```text
services.msc says Running
```

Có thể cross-check:

```powershell
Get-Service
```

Network:

```text
GUI adapter configuration
```

cross-check:

```powershell
Get-NetIPConfiguration
```

Storage:

```text
Disk Management
```

cross-check:

```powershell
Get-Disk
Get-Partition
Get-Volume
```

Multiple evidence sources giúp phát hiện misunderstanding.

---

# CXCIII. Common beginner mistakes — toàn Day 3

### Mistake 1

Dùng Administrator cho mọi task.

### Mistake 2

Fix permission bằng Full Control.

### Mistake 3

Disable firewall.

### Mistake 4

Restart/reboot trước khi collect evidence.

### Mistake 5

Nhầm local và domain account.

### Mistake 6

Không kiểm tra GPO.

### Mistake 7

Nhìn service Running và kết luận application healthy.

### Mistake 8

Nhìn patch installed và kết luận maintenance successful.

### Mistake 9

Nhìn backup Success và kết luận restore possible.

### Mistake 10

Screenshot Event Viewer nhưng không lưu Event ID/Record ID/timezone.

### Mistake 11

Export CSV mà không biết encoding.

### Mistake 12

Empty query = “không có sự kiện”.

### Mistake 13

Change Registry không capture original state.

### Mistake 14

Run PowerShell script không review.

### Mistake 15

Set execution policy Bypass như universal fix.

---

# CXCIV. Security checklist về mặt lý thuyết

Trước bất kỳ administrative change nào, tự hỏi:

```text
1. Tôi có authorization không?
2. Tôi có đúng account không?
3. Privilege có nhỏ nhất cần thiết không?
4. Change có audit được không?
5. Có secret nào bị log không?
6. Có rollback không?
7. Tôi đã baseline chưa?
8. Tôi biết blast radius chưa?
```

---

# CXCV. Verification hierarchy

Verification tốt thường có nhiều layer:

```text
Configuration verification
        ↓
State verification
        ↓
Connectivity verification
        ↓
Application verification
        ↓
User/business verification
```

Ví dụ RDP rule:

```text
Firewall rule exists
```

không bằng:

```text
Firewall rule enabled
```

cũng không bằng:

```text
port reachable
```

và chưa bằng:

```text
authorized user can RDP successfully
```

---

# CXCVI. Evidence hierarchy

Tương tự:

```text
Screenshot
<
Structured command output
<
Native export + metadata
```

Không phải screenshot vô dụng. Screenshot có thể tốt để:

- UI state;
- dashboard;
- visual configuration.

Nhưng với Event Logs, native EVTX thường mạnh hơn screenshot.

---

# CXCVII. Day 3 escalation mindset

L1/L2 nên biết giới hạn.

Một engineer tốt không phải người cố tự fix mọi thứ.

Một escalation tốt gồm:

```text
Symptom
Start time
Affected host
Impact
What changed
What was checked
Evidence
Hypothesis
What was attempted
Result
Current state
Requested next action
```

Bad escalation:

> “Server lỗi, check giúp.”

Good escalation:

```text
SERVER01 application unavailable since 14:32 ICT.
Windows host reachable and RDP accessible.
Service AppSvc repeatedly stops within 20 seconds.
System Event 7031 and application provider event 101 observed.
Disk has 42 GB free; network dependency reachable.
No configuration change performed.
Restart attempted once under INC001234 per runbook and failure reproduced.
EVTX and service-state outputs attached.
Escalating to Application L2 for crash analysis.
```

Đây mới là operational readiness.

---

# CXCVIII. Mental model tổng hợp Day 3

Khi nhận một Windows Server hoàn toàn mới, sequence lý tưởng:

```text
WHO IS IT?
    ↓
Hostname / asset / environment

WHAT IS IT?
    ↓
Release / edition / build / Core-or-Desktop

WHAT DOES IT DO?
    ↓
Roles / applications / services

WHO CAN ACCESS IT?
    ↓
Local/domain users / groups / rights

HOW IS IT REACHED?
    ↓
NIC / IP / DNS / routes / firewall / RDP / WinRM

WHERE IS ITS DATA?
    ↓
Disks / partitions / volumes / NTFS / shares

HOW IS IT MAINTAINED?
    ↓
Updates / scheduled tasks / backup

HOW DO I KNOW ITS HEALTH?
    ↓
Services / processes / events / monitoring

HOW DO I PROVE WHAT HAPPENED?
    ↓
PowerShell / EVTX / timestamps / hashes / ticket

WHAT IF SOMETHING GOES WRONG?
    ↓
Troubleshoot / rollback / recover / escalate
```

Nếu bạn có được mental model này, bạn đã hiểu trọng tâm thật sự của Windows Server Infrastructure Operations.

---

# CXCIX. Phần “Theo syllabus” và phần “Kiến thức bổ sung”

## Theo syllabus — bắt buộc

Bạn phải hiểu được:

- Windows Server architecture.
- Versions/generations.
- Standard/Datacenter/Azure Edition/Essentials awareness.
- Core/Desktop Experience.
- Server Manager/WAC.
- Services/processes/startup.
- Registry.
- local users/groups/security policy.
- NTFS/share.
- AD/domain/OU/GPO.
- DNS/DHCP.
- RDP/WinRM/Firewall/networking.
- Storage.
- updates.
- events.
- scheduler.
- backup.
- PowerShell.
- maintenance.
- edition matrix.
- EVTX/CSV/event evidence.

Đó chính xác là phạm vi Day 3. Infrastructure_Operations_14_Da…

## Kiến thức bổ sung tôi đã dùng để làm rõ

Các phần như:

- UAC.
- authentication vs authorization.
- high-level Kerberos awareness.
- NLA.
- detailed troubleshooting layering.
- evidence-before-change.
- GUI vs PowerShell cross-verification.

Đây là nền tảng operational để hiểu đúng syllabus, không thay đổi mục tiêu Day 3.

---

# CC. Cheat Sheet lý thuyết — Day 3

## Core management

```text
Server Manager
Windows Admin Center
PowerShell
MMC tools
Remote management
```

## Identity

```text
Local user
Domain user
Security groups
Least privilege
Local Security Policy
GPO
```

## Permissions

```text
NTFS ACL
Share permission
Inheritance
Allow/Deny
Effective Access
```

## Remote access

```text
RDP
WinRM
PowerShell Remoting
```

## Networking

```text
NIC
IP
Prefix
Gateway
DNS
Routes
Firewall
Ports
```

## Storage

```text
Disk
Partition
Volume
Filesystem
Free space
```

## Operations

```text
Services
Processes
Scheduled Tasks
Updates
Backup/Restore
Event Logs
```

## PowerShell

```text
Verb-Noun
Objects
Pipeline
Get-*
Filter
Select
Export
Remoting
Execution Policy
```

## Evidence

```text
Hostname
Timestamp + timezone
Event ID
Record ID
Provider
Native EVTX
CSV
Command/filter
Ticket
Count
SHA-256
```

## Critical rule

```text
Read
→ Understand
→ Baseline
→ Change
→ Verify
→ Document
→ Rollback/Escalate
```

---

# CCI. Điều kiện được xem là “hiểu Module Lý thuyết Day 3”

Bạn chưa cần thuộc mọi command. Nhưng bạn cần tự giải thích được:

1. Tại sao phải biết Windows release, edition và installation mode trước khi làm maintenance.
2. Server Core khác Desktop Experience ở đâu.
3. Service và process khác nhau thế nào.
4. Vì sao restart service không nên là troubleshooting step đầu tiên.
5. Local account khác domain account thế nào.
6. Authentication khác authorization thế nào.
7. NTFS và Share permissions kết hợp ra sao.
8. OU khác Security Group như thế nào.
9. Vì sao GPO có thể làm local setting quay lại trạng thái cũ.
10. Vì sao Active Directory phụ thuộc DNS.
11. RDP failure phải troubleshoot theo layers nào.
12. WinRM liên quan PowerShell Remoting ra sao.
13. Vì sao không disable Windows Firewall để troubleshooting.
14. Disk, partition và volume khác nhau thế nào.
15. Vì sao “patch installed” chưa đủ để close maintenance.
16. Event ID, Provider và Record ID có vai trò gì.
17. Native EVTX khác CSV summary thế nào.
18. Vì sao empty `Get-WinEvent` result không chứng minh “không có event”.
19. Vì sao Scheduled Task chạy manual được nhưng scheduled lại fail.
20. Vì sao successful backup không chứng minh successful recovery.
21. PowerShell object pipeline khác traditional text pipeline ở điểm nào.
22. Execution Policy có phải security boundary hay không.
23. Vì sao phải preserve original evidence.
24. SHA-256 chứng minh điều gì và không chứng minh điều gì.
25. Khi nào L1/L2 phải dừng và escalate.

Nếu bạn giải thích được 25 điểm trên bằng reasoning thay vì học thuộc lòng, phần foundation của **Module Lý thuyết Day 3** đã đạt yêu cầu.

Module Lý thuyết này bao phủ toàn bộ scope Day 3 trong syllabus. Bước kế tiếp hợp lý là **Module Thực hành Day 3**, nơi toàn bộ concepts trên sẽ được chuyển thành một Windows Server lab hoàn chỉnh: từ provision VM → inventory/version detection → users/groups → NTFS/share → RDP/WinRM/firewall → services/processes → storage → patch inspection → scheduled PowerShell health check → `Get-WinEvent`/EVTX/CSV evidence → failure injection → troubleshooting → rollback → final maintenance checklist.
