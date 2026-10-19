# Day 3 — Module Lý thuyết

## Windows Server Administration Foundations

Theo syllabus, toàn bộ **Concept/Lecture của Day 3** bao gồm: Windows Server architecture; supported/legacy generations; edition/licensing; Standard, Datacenter, Datacenter: Azure Edition, Essentials/legacy awareness; Server Core và Desktop Experience; Server Manager; Windows Admin Center; services/processes/startup; Registry; local users/groups/security policy; NTFS/share permissions; AD DS/domain accounts/OU/Group Policy; DNS/DHCP; RDP/WinRM; Windows Defender Firewall; networking; storage; Windows Update; Event Viewer/Event Logs; Task Scheduler; backup/recovery; PowerShell cmdlets, pipelines, remoting, execution policies, scripting; và Windows Server operational maintenance. Infrastructure_Operations_14_Da…

Phần dưới đây chỉ tập trung vào **Module Lý thuyết**. Tôi sẽ dùng command chủ yếu để minh họa và verification; hands-on workflow hoàn chỉnh, failure injection và assignment sẽ để dành cho **Module Thực hành**.

---

# 1. Bức tranh tổng thể của Windows Server

## 1.1. Windows Server là gì?

Windows Server là server operating system của Microsoft, được thiết kế để cung cấp các infrastructure services và application workloads như:

| Nhóm                      | Ví dụ                                                 |
| ------------------------- | ----------------------------------------------------- |
| Identity                  | Active Directory Domain Services                      |
| Name resolution           | DNS Server                                            |
| IP configuration          | DHCP Server                                           |
| File services             | SMB/File Server                                       |
| Web services              | IIS                                                   |
| Virtualization            | Hyper-V                                               |
| Remote management         | WinRM, PowerShell Remoting                            |
| Security                  | Windows Firewall, Local Security Policy, Group Policy |
| Automation                | PowerShell, Task Scheduler                            |
| Storage                   | NTFS, ReFS, Storage Spaces                            |
| Infrastructure management | Server Manager, Windows Admin Center                  |

Điểm quan trọng đối với Infrastructure Operations là:

> Một Windows Server không chỉ là một operating system. Nó là tập hợp của operating system + configuration + identities + services + network + storage + security policies + roles/features + logs + dependencies.

Do đó khi user nói:

> “Server bị lỗi.”

Câu đó chưa cung cấp đủ thông tin.

Operations engineer phải phân tách vấn đề thành các layer:

```text
Hardware / Hypervisor
        ↓
Operating System
        ↓
Driver / Kernel resources
        ↓
Storage / Network
        ↓
Identity / Permissions
        ↓
Windows Services
        ↓
Server Role
        ↓
Application
        ↓
Client connectivity
```

Troubleshooting tốt thường là xác định **layer nào đang fail** trước khi thay đổi configuration.

---

# 2. Windows Server architecture

Đây là phần foundation để hiểu tại sao một server có thể boot được nhưng application vẫn fail.

## 2.1. Kernel mode và User mode

Windows tách execution environment thành hai vùng quan trọng.

### Kernel mode

Kernel mode có mức privilege rất cao.

Các thành phần như kernel, một số executive components và device drivers hoạt động ở đây.

Nếu code trong kernel mode gặp lỗi nghiêm trọng, ảnh hưởng có thể lan ra toàn operating system, ví dụ system crash hoặc bugcheck.

### User mode

Phần lớn applications và nhiều services chạy ở user mode.

Một application crash trong user mode thường không làm cả operating system crash.

Operationally, điều này giải thích vì sao:

```text
Operating System healthy
≠
Application healthy
```

Server có thể ping được, RDP được, CPU bình thường nhưng một business application vẫn down do service/process/configuration của chính application đó.

---

# 3. Các operational layers cần hiểu

Có thể nhìn Windows Server như sau:

```text
+--------------------------------------+
| Applications / Server Roles         |
+--------------------------------------+
| Windows Services / Processes        |
+--------------------------------------+
| Security / Identity / Registry      |
+--------------------------------------+
| Network / Storage / Filesystem      |
+--------------------------------------+
| Windows Kernel / Drivers            |
+--------------------------------------+
| VM / Hypervisor / Physical Hardware |
+--------------------------------------+
```

Một incident có thể đi xuyên nhiều layer.

Ví dụ:

```text
User không truy cập được \\FILE01\Data
```

Root cause có thể là:

```text
DNS failure
→ FILE01 không resolve được

Network failure
→ TCP/445 không reachable

Firewall
→ SMB traffic bị block

Service
→ Server service không hoạt động

Share permission
→ user không được cấp quyền

NTFS ACL
→ filesystem từ chối user

AD
→ domain account/group membership sai

Storage
→ volume chứa Data offline
```

Đây là lý do Infrastructure Operations không được “đoán lỗi” dựa trên symptom duy nhất.

---

# 4. Windows Server generations và lifecycle

## Theo syllabus

Bạn phải nhận biết **supported và legacy Windows Server generations**. Infrastructure_Operations_14_Da…

## Kiến thức bổ sung — trạng thái hiện tại

Tại thời điểm hiện tại, Windows Server 2025 là một release được Microsoft hỗ trợ theo Fixed Lifecycle. Windows Server 2022 đã kết thúc Mainstream Support vào ngày 14/10/2026 và còn Extended Support đến 15/10/2031. Windows Server 2019 đang trong Extended Support đến tháng 1/2029. Windows Server 2016 đang trong Extended Support tới tháng 1/2027. Windows Server 2012 R2 đã hết Extended Support từ 2023 và chu kỳ ESU cuối cũng đã kết thúc vào tháng 10/2026. [Microsoft Learn](https://learn.microsoft.com/en-us/lifecycle/products/windows-server-2025?utm_source=chatgpt.com)

| Generation             | Operational view vào 19/10/2026                        |
| ---------------------- | ------------------------------------------------------ |
| Windows Server 2025    | Current, Mainstream Support                            |
| Windows Server 2022    | Extended Support                                       |
| Windows Server 2019    | Extended Support                                       |
| Windows Server 2016    | Extended Support, gần cuối lifecycle                   |
| Windows Server 2012 R2 | Legacy / out of normal support và ESU cuối đã kết thúc |
| Older generations      | Legacy                                                 |

Điều này rất quan trọng trong Operations.

Một server “vẫn chạy tốt” không có nghĩa nó còn phù hợp về:

```text
Security
Compliance
Vendor support
Application certification
Patch availability
Operational risk
```

### Khi gặp legacy server

L1/L2 thường không tự ý:

```text
Upgrade OS
Upgrade application
Change compatibility settings
Replace authentication methods
Migrate production workload
```

Thay vào đó phải xác định:

```text
Server identity
OS version
Patch level
Application dependency
Business owner
Support contract
Known migration plan
Security compensating controls
Change/ticket reference
```

Sau đó escalate cho đúng owner.

---

# 5. Version, edition, build và installation mode

Đây là bốn khái niệm không nên nhầm với nhau.

### Version

Ví dụ:

```text
Windows Server 2019
Windows Server 2022
Windows Server 2025
```

### Edition

Ví dụ:

```text
Standard
Datacenter
Datacenter: Azure Edition
Essentials
```

### Build

Cho biết OS build cụ thể và giúp xác định patch level.

### Installation mode

Ví dụ:

```text
Server Core
Server with Desktop Experience
```

Hai Windows Server có cùng version nhưng khác edition hoặc installation mode có thể có capability và management workflow khác nhau.

---

# 6. Windows Server editions

Microsoft hiện cung cấp các edition như Standard, Datacenter và Datacenter: Azure Edition cho các release liên quan; Windows Server 2025 lifecycle cũng liệt kê Essentials. Microsoft có bảng comparison riêng vì feature availability không giống nhau giữa edition. [Microsoft Learn](https://learn.microsoft.com/en-us/lifecycle/products/windows-server-2025?utm_source=chatgpt.com)

## 6.1. Standard

Standard thường phù hợp hơn với workload có mức virtualization không lớn.

Một điểm licensing quan trọng: trong physical-core licensing model hiện hành, khi server được licensed đầy đủ theo yêu cầu core, Standard có virtualization rights cho hai Operating System Environments; additional licensing có thể cấp quyền cho thêm OSE. [Microsoft](https://www.microsoft.com/licensing/terms/en-US/productoffering/WindowsServerDatacenter/all/Availability/102/?utm_source=chatgpt.com)

Không nên diễn giải thành:

> “Standard chỉ chạy được hai VM.”

Đây là **licensing rights**, không phải giới hạn kỹ thuật đơn giản của Hyper-V.

---

# 7. Datacenter

Datacenter hướng tới workload có mật độ virtualization cao và có thêm một số capability so với Standard tùy release.

Trong physical-core licensing model tương ứng, Datacenter cho phép Windows Server trong unlimited OSEs trên licensed server khi licensing requirements được đáp ứng. [Microsoft](https://www.microsoft.com/licensing/terms/en-US/productoffering/WindowsServerDatacenter/all/Availability/102/?utm_source=chatgpt.com)

Operational implication:

Một infrastructure engineer cần biết edition vì:

```text
Feature availability
Virtualization rights
Architecture decisions
Upgrade decisions
Compliance/licensing
Troubleshooting assumptions
```

đều có thể bị ảnh hưởng.

---

# 8. Datacenter: Azure Edition

Đây là edition có capability nhắm tới Azure/hybrid scenarios.

Không được suy luận rằng:

```text
Datacenter: Azure Edition
=
Datacenter bình thường + một cái tên Azure
```

Một số feature và deployment conditions khác nhau.

Trong enterprise, trước khi khẳng định một feature có hay không, phải kiểm tra:

```text
Windows Server generation
Edition
Deployment environment
Microsoft documentation
Current licensing terms
```

Syllabus cũng yêu cầu rõ rằng edition-specific capability phải được xác minh với release được sử dụng trong lab hoặc enterprise environment. Infrastructure_Operations_14_Da…

---

# 9. Essentials

Essentials historically hướng nhiều hơn tới small-business scenarios.

Với Windows Server 2025, Microsoft licensing guidance liệt kê Essentials theo mô hình OEM-specific thay vì cùng licensing model với Standard/Datacenter. [Microsoft](https://www.microsoft.com/licensing/guidance/Windows-Server-2025?msockid=18a21e20e58f63ef0c8b0898e4d462c9&utm_source=chatgpt.com)

Day 3 chỉ cần đạt **Essentials/legacy awareness**, tức là:

Bạn phải nhận ra edition khi gặp nó nhưng không cần trở thành specialist về toàn bộ lịch sử licensing của Essentials.

---

# 10. Licensing awareness

Infrastructure Operations không phải licensing legal team, nhưng phải biết đủ để tránh technical assumptions sai.

Ví dụ phải phân biệt:

```text
Technical capability
vs
Licensed usage rights
```

Một feature chạy được không tự động đồng nghĩa enterprise có quyền licensing để sử dụng theo cách đó.

Windows Server Standard và Datacenter hiện sử dụng core-based licensing trong các model phổ biến; các chi tiết về CAL, physical core, VM licensing, Software Assurance hoặc subscription có thể thay đổi theo licensing program nên quyết định licensing thực tế phải dựa trên **Microsoft Product Terms** và license owner của tổ chức. [Microsoft](https://www.microsoft.com/licensing/guidance/Core-based-licensing-models?utm_source=chatgpt.com)

---

# 11. Server Core vs Server with Desktop Experience

Đây là một concept cực kỳ quan trọng.

Microsoft hiện hỗ trợ hai installation options chính đối với Standard/Datacenter: **Server Core** và **Server with Desktop Experience**. Microsoft khuyến nghị Server Core trừ khi workload thực sự cần GUI. Với các supported releases hiện hành, không thể đơn giản convert Server Core sang Desktop Experience sau installation; cần clean installation để đổi installation option. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/get-started/install-options-server-core-desktop-experience?source=recommendations&utm_source=chatgpt.com)

## Server Core

Server Core không cài full Desktop GUI.

Administration thường sử dụng:

```text
PowerShell
SConfig
Server Manager remotely
Windows Admin Center
Other remote-management tools
```

### Ưu điểm

Smaller footprint.

Ít GUI components hơn.

Attack surface nhỏ hơn.

Ít thành phần cần service/patch hơn.

Khuyến khích remote administration.

### Hạn chế

Một số GUI-dependent applications hoặc tools có thể không phù hợp.

Engineer phụ thuộc GUI có thể troubleshooting khó hơn nếu PowerShell skill yếu.

---

## Desktop Experience

Có Windows graphical shell và nhiều GUI administration tools.

Dễ tiếp cận hơn đối với fresher.

Phù hợp khi:

```text
Application yêu cầu GUI
Vendor tool yêu cầu local GUI
Operational process chưa hỗ trợ headless administration
```

Tuy nhiên không nên nghĩ:

> “Server có GUI thì tốt hơn.”

Trong production infrastructure, việc giảm unnecessary components thường có lợi về security và maintenance.

---

# 12. Management plane của Windows Server

Một Windows Server có thể được quản trị từ nhiều interface.

```text
Local GUI
        │
Server Manager
        │
MMC consoles
        │
PowerShell
        │
SConfig
        │
WinRM / PowerShell Remoting
        │
Windows Admin Center
        │
Enterprise management platform
```

Một Operations engineer cần tránh phụ thuộc một interface duy nhất.

Ví dụ Server Manager không mở được không có nghĩa server không quản trị được.

---

# 13. Server Manager

Server Manager là một built-in Windows Server management tool.

Nó giúp administrator quản lý:

```text
Server properties
Roles
Features
Events
Services
Performance
Storage
Remote servers
```

Windows Server management documentation hiện vẫn liệt kê Server Manager cùng PowerShell, Windows commands và RSAT trong nhóm local management tools. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/overview?utm_source=chatgpt.com)

### Operational value

Server Manager đặc biệt hữu ích để:

```text
Initial server inspection
Role/feature installation
Server pool management
High-level health view
```

### Nhưng cần nhớ

Server Manager chỉ là **management interface**.

Nếu Server Manager hiển thị service stopped, root cause nằm ở service/application/dependency chứ không phải mặc định do Server Manager.

---

# 14. Windows Admin Center

Windows Admin Center là browser-based remote management solution cho Windows Server chạy physical, virtual, on-premises hoặc cloud environments. Microsoft mô tả nó như modern management experience bổ sung cho các công cụ như Server Manager/MMC. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/overview?utm_source=chatgpt.com)

Nó có thể expose management interfaces cho nhiều component như:

```text
Processes
Services
Events
Storage
Networking
Certificates
Firewall
PowerShell
Updates
Virtual machines
```

Day 3 chỉ yêu cầu **awareness**, nhưng bạn cần hiểu một principle:

> Management UI khác với managed component.

Nếu Windows Admin Center không kết nối được tới Server01, có thể server vẫn hoàn toàn healthy.

Root cause có thể nằm ở:

```text
DNS
Firewall
WinRM
Authentication
Authorization
Certificate
Management gateway
```

---

# 15. Windows Services

Windows service là program/process được thiết kế chạy background và được quản lý bởi **Service Control Manager**.

Một service thường có các attributes quan trọng:

| Attribute    | Ý nghĩa                        |
| ------------ | ------------------------------ |
| Service Name | Internal identifier            |
| Display Name | Tên thân thiện                 |
| Status       | Running, Stopped...            |
| Start Type   | Automatic, Manual, Disabled... |
| Log On As    | Security identity              |
| Dependencies | Services/components phụ thuộc  |
| Binary Path  | Executable                     |
| Recovery     | Hành động sau failure          |

---

# 16. Service startup types

### Automatic

Service được thiết lập để start trong quá trình boot/service startup processing.

### Automatic (Delayed Start)

Service tự động start nhưng có delay để giảm startup contention.

### Manual

Không tự động start như Automatic; có thể được system, dependent component hoặc administrator start khi cần.

### Disabled

Không được phép start cho đến khi configuration được thay đổi.

Operationally:

```text
Stopped + Manual
```

không tự động đồng nghĩa lỗi.

Nhưng:

```text
Stopped + Automatic
+ application unavailable
```

đáng điều tra.

---

# 17. Service account

Service có thể chạy dưới security context khác nhau, ví dụ built-in service accounts hoặc domain-managed/service identities tùy application.

Security implication rất lớn.

Nếu service chạy bằng account có quá nhiều privilege:

```text
Service compromise
→ privilege escalation impact lớn hơn
```

Nếu account thiếu privilege:

```text
Service start
→ Access Denied
→ service fails
```

Nếu domain service account password bị thay đổi nhưng configuration chưa cập nhật:

```text
Service logon failure
```

có thể xuất hiện.

Vì vậy khi troubleshoot service phải kiểm tra cả:

```text
State
Startup Type
Log On account
Dependencies
Executable
Configuration
Events
Network dependency
Filesystem permission
```

---

# 18. Service dependencies

Application service A có thể phụ thuộc service B.

Nếu:

```text
B fails
```

thì:

```text
A may fail
```

Symptom user nhìn thấy có thể nằm ở A nhưng root cause nằm ở B.

Đây là một nguyên tắc troubleshooting phổ biến:

> Không restart dependency chain một cách mù quáng.

Trước tiên phải xác định dependency và impact.

---

# 19. Verification service

Read-only PowerShell:

```powershell
Get-Service
```

Một service cụ thể:

```powershell
Get-Service -Name Spooler
```

Service theo trạng thái:

```powershell
Get-Service | Where-Object Status -eq 'Running'
```

Quan trọng:

`Get-Service` chỉ cho biết service state ở mức service manager.

Nó không chứng minh application fully healthy.

Ví dụ web application có thể:

```text
Service = Running
Port = Listening
HTTP endpoint = 500
```

Do vậy verification thường phải đi xa hơn.

---

# 20. Windows processes

Process là running instance của executable.

Ví dụ:

```text
powershell.exe
svchost.exe
w3wp.exe
lsass.exe
```

Một process có thể có:

```text
PID
CPU usage
Memory usage
Threads
Handles
Security context
Executable path
Network connections
```

### Service khác Process như thế nào?

Không nên đồng nhất hai khái niệm.

Một service là logical service managed bởi Service Control Manager.

Một process là executing program instance.

Nhiều services có thể cùng chạy trong một shared host process, và một application có thể có nhiều process.

---

# 21. Process verification

```powershell
Get-Process
```

Ví dụ:

```powershell
Get-Process | Sort-Object CPU -Descending
```

Nhưng lưu ý:

CPU value cao chưa chắc là lỗi.

Bạn cần context:

```text
CPU cao bao lâu?
Baseline là gì?
Application đang xử lý workload lớn?
Server có saturation không?
Memory pressure?
Disk I/O?
```

Một snapshot đơn lẻ không đủ để kết luận root cause.

---

# 22. Startup configuration

Startup có thể đến từ nhiều nguồn:

```text
Windows Services
Scheduled Tasks
Startup folders
Registry startup entries
Application-specific mechanisms
Group Policy
```

Điều này quan trọng khi điều tra:

```text
“Sau reboot application không tự start.”
```

Đừng chỉ nhìn Task Manager Startup.

Trên server, service configuration và Task Scheduler thường quan trọng hơn.

---

# 23. Windows Registry fundamentals

Windows Registry là hierarchical configuration database.

Các root hives quan trọng gồm:

| Hive                  | Ý nghĩa khái quát                   |
| --------------------- | ----------------------------------- |
| `HKEY_LOCAL_MACHINE`  | Machine-wide configuration          |
| `HKEY_CURRENT_USER`   | Current user's configuration        |
| `HKEY_USERS`          | Loaded user profiles                |
| `HKEY_CLASSES_ROOT`   | Class/file association view         |
| `HKEY_CURRENT_CONFIG` | Current hardware configuration view |

Ví dụ Registry path:

```text
HKLM\SYSTEM\CurrentControlSet\Services
```

nơi chứa nhiều service-related configuration.

---

# 24. Registry operational safety

Registry có thể ảnh hưởng sâu tới operating system.

Không nên có workflow kiểu:

```text
Google lỗi
→ tìm registry key
→ edit
→ reboot
```

Workflow đúng hơn:

```text
Confirm symptom
→ identify documented setting
→ determine config owner
→ backup/export
→ change approval
→ apply smallest required change
→ verify
→ rollback if unsuccessful
```

### Read-only query

```powershell
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Services\Spooler'
```

### Configuration-changing operation

```powershell
Set-ItemProperty ...
```

Operation thứ hai cần change control khi áp dụng vào managed environment.

---

# 25. Security principals, SID và access token

Để hiểu Windows permissions, cần nắm ba concepts bổ sung rất quan trọng.

## Security principal

Entity có thể được authentication/authorization, chẳng hạn:

```text
User
Group
Computer account
Service account
```

## SID — Security Identifier

Windows sử dụng SID làm identifier thực sự cho security principal.

Tên account có thể thay đổi, SID mới là identity quan trọng trong ACL.

## Access token

Sau authentication, Windows tạo access token chứa information như:

```text
User SID
Group SIDs
Privileges
Integrity information
```

Khi user mở một file, hệ thống so access token với ACL để quyết định allow/deny.

Đây là nền tảng để hiểu permission troubleshooting.

---

# 26. Local users

Local account tồn tại trên chính server.

Ví dụ:

```text
SERVER01\opsuser
```

Local user trên SERVER01 không mặc định giống local user cùng tên trên SERVER02.

### Verification

```powershell
Get-LocalUser
```

Một account:

```powershell
Get-LocalUser -Name 'opsuser'
```

Khi Operations tạo user, cần quan tâm:

```text
Purpose
Owner
Expiration
Group membership
Required privilege
Audit trail
Deprovisioning
```

---

# 27. Local groups

Group giúp cấp permission cho nhóm users thay vì trực tiếp cho từng user.

Ví dụ:

```text
Administrators
Remote Desktop Users
Remote Management Users
```

Operational best practice:

```text
Permission
→ Group
→ User membership
```

thường dễ audit hơn:

```text
Permission
→ hàng chục individual users
```

### Verification

```powershell
Get-LocalGroup
```

```powershell
Get-LocalGroupMember -Group 'Administrators'
```

Đây là những read-only checks rất quan trọng trong security review.

---

# 28. Least privilege

Least privilege nghĩa là chỉ cấp privilege cần thiết để hoàn thành nhiệm vụ.

Ví dụ một operator chỉ cần đọc Event Logs thì không nên mặc định cấp:

```text
Local Administrators
```

Một service chỉ cần read một folder thì không nên có:

```text
Full Control toàn drive
```

Mục tiêu của least privilege là giảm:

```text
Accidental change
Attack surface
Blast radius
Credential abuse
Audit exposure
```

---

# 29. Local Security Policy

Local Security Policy có thể điều khiển những area như:

```text
Account Policies
Local Policies
User Rights Assignment
Security Options
Audit Policy
```

Ví dụ User Rights Assignment có thể quy định identity nào được:

```text
Log on locally
Log on as a service
Access this computer from the network
Shut down system
```

Điểm cực kỳ quan trọng:

> Trên domain-joined server, effective configuration có thể đến từ Group Policy, không chỉ Local Security Policy.

Do vậy bạn sửa local policy nhưng sau refresh nó quay lại, có thể đó không phải “Windows tự đổi”, mà GPO đang enforce configuration.

---

# 30. NTFS permissions

NTFS dùng Access Control Lists để quyết định access đến files/directories.

Common permissions gồm:

| Permission           | Ý nghĩa                                      |
| -------------------- | -------------------------------------------- |
| Full Control         | Gần như toàn quyền gồm permission management |
| Modify               | Read/write/delete                            |
| Read & Execute       | Read và execute                              |
| List Folder Contents | List folder                                  |
| Read                 | Read                                         |
| Write                | Write                                        |

---

# 31. Explicit và inherited permissions

### Explicit

Permission được gán trực tiếp cho object.

### Inherited

Permission được kế thừa từ parent folder.

Ví dụ:

```text
D:\Data
   └── Finance
       └── Report.xlsx
```

Nếu `Finance` inherit permissions từ `D:\Data`, access có thể đến từ parent ACL.

Do vậy nhìn riêng ACL của file mà không hiểu inheritance có thể dẫn đến chẩn đoán sai.

---

# 32. Allow và Deny

Windows ACL evaluation có nhiều nuance, nhưng ở mức Day 3 cần đặc biệt cảnh giác với explicit `Deny`.

Một user có thể thuộc:

```text
GRP-Finance-Read
GRP-Contractor-Deny
```

và effective access có thể không giống thứ bạn nghĩ khi chỉ nhìn một group.

Do đó troubleshooting phải xác định:

```text
Actual user
All relevant group memberships
Inherited ACL
Explicit ACL
Effective access
```

Không được chỉ hỏi:

> “User có nằm trong Finance group không?”

---

# 33. NTFS permission verification

Read-only:

```powershell
Get-Acl C:\Data
```

Ví dụ format:

```powershell
(Get-Acl C:\Data).Access
```

Command giúp inspect ACL nhưng vẫn cần hiểu actual access token và inheritance.

---

# 34. Share permissions

Khi folder được expose qua SMB share, network user có thể chịu hai permission layers:

```text
Share permissions
+
NTFS permissions
```

Ví dụ:

```text
Share: Change
NTFS: Read
```

User không tự động có Modify vì NTFS vẫn hạn chế.

Một mental model hữu ích:

> Effective network access không thể vượt qua restriction của permission layer còn lại.

---

# 35. Local access và network access

Nếu user trực tiếp trên server mở:

```text
D:\Data
```

share permission không tham gia như khi truy cập:

```text
\\SERVER01\Data
```

Do đó symptom:

> “Tôi mở được folder khi RDP vào server nhưng từ máy workstation không mở được.”

là một clue rất quan trọng.

Nó giúp thu hẹp investigation về:

```text
Share
SMB
Network
Firewall
Share permissions
Authentication
```

thay vì chỉ NTFS.

---

# 36. Active Directory Domain Services — AD DS

AD DS là directory service lưu trữ và tổ chức information về objects như users, computers và groups trong hierarchical directory. Domain Controllers cùng domain tham gia replication directory data. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview?view=sc-vmm-1801&utm_source=chatgpt.com)

Day 3 chỉ yêu cầu **AD DS basics**, không yêu cầu bạn thiết kế enterprise forest.

Bạn cần hiểu:

```text
Forest
Domain
Domain Controller
User
Group
Computer object
OU
Authentication
Authorization
Group Policy
DNS dependency
```

---

# 37. Domain

Domain là một logical administrative/security boundary trong AD DS.

Ví dụ:

```text
corp.example.com
```

Objects có thể gồm:

```text
Users
Computers
Groups
OUs
```

Server joined domain sẽ có computer account trong AD.

---

# 38. Domain Controller

Domain Controller host AD DS database/services và tham gia authentication, authorization-related workflows, policy discovery và directory services.

Nếu Domain Controller unreachable, effects có thể xuất hiện ở nhiều nơi:

```text
Logon
Group Policy
DNS
Service authentication
Application authentication
Remote administration
```

Tuy nhiên không phải mọi domain-dependent operation lập tức fail giống nhau vì caching và architecture có thể ảnh hưởng behavior.

---

# 39. Local account vs domain account

Ví dụ:

```text
SERVER01\administrator
```

là local context.

Trong khi:

```text
CORP\jdoe
```

là domain identity.

Điều này cực kỳ quan trọng khi troubleshoot:

```text
“User jdoe là Administrator.”
```

Câu hỏi tiếp theo phải là:

```text
Local jdoe?
Domain jdoe?
Thông qua group nào?
Trên server nào?
```

---

# 40. Organizational Unit — OU

OU là container logic trong AD DS.

Ví dụ:

```text
corp.example.com
 └── Servers
     ├── Production
     └── NonProduction
```

OU thường được sử dụng để:

```text
Organize objects
Delegate administration
Scope Group Policy
```

OU không phải security group.

### OU

Tổ chức objects và policy/delegation scope.

### Group

Thường dùng membership để cấp permissions/access.

Không được dùng hai concepts thay thế nhau.

---

# 41. Group Policy

Group Policy cung cấp centralized configuration cho users và computers.

GPO có thể link vào:

```text
Site
Domain
OU
```

và scope/process quyết định GPO nào áp dụng vào object. Microsoft mô tả Group Policy là cơ chế centralized configuration cho operating systems, applications và user/computer settings; processing phụ thuộc site/domain/OU context cùng những controls khác. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-policy/group-policy-overview?trk=article-ssr-frontend-pulse_little-text-block&utm_source=chatgpt.com)

---

# 42. Tại sao Group Policy quan trọng với Operations?

Giả sử bạn thấy:

```text
Windows Firewall rule bị thay đổi
```

Bạn sửa local rule.

Một lúc sau rule quay về như cũ.

Khả năng:

```text
Domain GPO đang enforce firewall configuration
```

Hoặc:

```text
Local policy says X
Effective policy says Y
```

Do đó phải phân biệt:

```text
Local configuration
Configured policy
Effective policy
```

---

# 43. GPO troubleshooting awareness

Một L1/L2 engineer thường cần xác minh:

```powershell
gpresult /r
```

Hoặc report chi tiết:

```powershell
gpresult /h C:\Temp\gpresult.html
```

Nhưng thay đổi domain GPO thường vượt quá scope nếu không được explicitly delegated/authorized.

Operational boundary:

```text
Inspect
Collect evidence
Identify likely GPO
Escalate to AD/GPO owner
```

thường an toàn hơn việc tự edit policy.

---

# 44. DNS role

DNS ánh xạ names sang IP addresses và là một core dependency của Windows networking. Microsoft xác nhận DNS Server là Windows Server role, và khi triển khai AD DS forest/domain mới, DNS thường được cài cùng Active Directory. [Microsoft Learn](https://learn.microsoft.com/windows-server/networking/dns/dns-overview?utm_source=chatgpt.com)

### Ví dụ

```text
app01.corp.example.com
        ↓
10.20.30.40
```

DNS không chỉ “dịch website name”.

Trong Windows enterprise, DNS rất quan trọng cho:

```text
AD service discovery
Domain authentication
Server discovery
Application connectivity
Management tools
```

---

# 45. DNS records awareness

Day 3 nên nhận biết tối thiểu:

```text
A
AAAA
CNAME
PTR
SRV
```

### A

Hostname → IPv4.

### AAAA

Hostname → IPv6.

### CNAME

Alias → another DNS name.

### PTR

Reverse lookup IP → name.

### SRV

Service location; rất quan trọng trong AD environments.

---

# 46. DNS troubleshooting mindset

Khi:

```text
ping 10.20.30.40
```

works nhưng:

```text
ping APP01
```

fails, network path có thể tốt nhưng name resolution có vấn đề.

Read-only checks:

```powershell
Resolve-DnsName APP01
```

```powershell
Get-DnsClientServerAddress
```

Không nên ngay lập tức edit:

```text
C:\Windows\System32\drivers\etc\hosts
```

để “fix” enterprise DNS problem.

Hosts-file workaround có thể che root cause và gây configuration drift.

---

# 47. DHCP role

DHCP Server tự động cấp IP configuration cho clients.

Microsoft mô tả DHCP Server là optional Windows Server networking role có thể quản lý address leases và các related settings. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-top?utm_source=chatgpt.com)

DHCP có thể cung cấp:

```text
IP address
Subnet mask/prefix information
Default gateway
DNS servers
Other DHCP options
```

---

# 48. DHCP scope

Scope là address range/logical configuration dành cho clients trên subnet.

Ví dụ:

```text
Subnet: 10.10.20.0/24

Pool:
10.10.20.50
→
10.10.20.200
```

Scope cũng có thể chứa options như DNS và gateway. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/networking/technologies/dhcp/dhcp-scopes?utm_source=chatgpt.com)

---

# 49. DHCP lease và reservation

### Lease

IP được cấp trong một khoảng thời gian.

### Reservation

Mapping được định nghĩa để client cụ thể nhận một predictable IP theo identification mechanism như MAC/client identifier tùy implementation.

Không được nhầm reservation với static IP configured trực tiếp trên client.

---

# 50. DHCP và DNS khác nhau thế nào?

```text
DHCP
→ Tôi nên dùng IP/network settings nào?

DNS
→ Name này tương ứng address/service nào?
```

Một server có thể:

```text
Có IP đúng nhưng DNS sai
```

hoặc:

```text
DNS service healthy nhưng client nhận wrong DNS server từ DHCP
```

Hai role có quan hệ nhưng không phải một.

---

# 51. RDP — Remote Desktop Protocol

RDP cung cấp remote interactive graphical session.

Operational dependencies gồm:

```text
Server reachable
RDP enabled
RDP service healthy
Firewall allows traffic
User authorized
Authentication succeeds
Policy permits access
```

Vì vậy:

> “RDP không được”

không đồng nghĩa:

> “Server down.”

---

# 52. RDP security considerations

Không nên expose RDP broadly nếu architecture không yêu cầu.

Enterprise thường bổ sung controls như:

```text
VPN
Jump host / bastion
Privileged access workflow
Restricted source networks
MFA at upstream control
Session auditing
Least privilege
```

Operations engineer không nên xử lý RDP problem bằng cách:

```text
Disable firewall
Allow port from Any/Any
Add user to Administrators
```

trừ khi có approved design rất cụ thể.

---

# 53. WinRM — Windows Remote Management

WinRM là Microsoft implementation của WS-Management và hỗ trợ remote management. PowerShell remoting trên Windows thường sử dụng WinRM. [Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/winrm/about-windows-remote-management?utm_source=chatgpt.com)

Conceptually:

```text
Management client
      ↓
PowerShell / management tool
      ↓
WinRM / WS-Management
      ↓
Remote server
```

---

# 54. RDP vs WinRM

| RDP                         | WinRM                                    |
| --------------------------- | ---------------------------------------- |
| Interactive GUI session     | Remote management protocol               |
| Human-driven                | Automation-friendly                      |
| Desktop session             | Command/object-oriented management       |
| Higher interactive overhead | Better suited to scripted administration |
| Useful for GUI tasks        | Useful for PowerShell remoting           |

Một mature Operations environment thường không yêu cầu RDP cho mọi tác vụ.

---

# 55. PowerShell Remoting

Ví dụ interactive:

```powershell
Enter-PSSession -ComputerName SERVER01
```

Remote command:

```powershell
Invoke-Command -ComputerName SERVER01 -ScriptBlock {
    Get-Service
}
```

Microsoft documentation xác nhận Windows PowerShell remoting trên Windows dùng WinRM và Windows Server releases hiện đại đã hỗ trợ remoting configuration mặc định theo server management design, mặc dù effective configuration vẫn cần được xác minh. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements?view=powershell-7.6&utm_source=chatgpt.com)

---

# 56. Authentication và authorization trong remoting

Network reachability không đủ.

Có thể xảy ra:

```text
TCP connection works
+
WinRM listener responds
+
Authentication works
+
User lacks authorization
=
Remote command denied
```

Troubleshooting vì vậy phải phân tách:

```text
Connectivity
Protocol/service
Authentication
Authorization
Endpoint configuration
Command permission
```

---

# 57. Windows Firewall

Syllabus gọi **Windows Defender Firewall**; tài liệu Microsoft hiện hành thường trình bày component dưới tên **Windows Firewall**.

Firewall filter network traffic dựa trên conditions như:

```text
Source IP
Destination IP
Protocol
Port
Application
Service
Interface
Profile
```

Microsoft hiện mô tả default behavior là block unsolicited inbound traffic trừ khi có applicable rule, trong khi outbound traffic mặc định thường allowed trừ khi có blocking rule. [Microsoft Learn](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/?utm_source=chatgpt.com)

---

# 58. Firewall profiles

Ba profiles quan trọng:

```text
Domain
Private
Public
```

Domain profile áp dụng trong domain network context thích hợp; Private/Public đại diện các trust context khác nhau. [Microsoft Learn](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/?utm_source=chatgpt.com)

Một firewall rule có thể đúng port nhưng sai profile, dẫn đến traffic vẫn bị block.

Do đó luôn kiểm tra:

```text
Rule enabled?
Direction?
Action?
Protocol?
Local port?
Remote port?
Source scope?
Destination scope?
Active profile?
```

---

# 59. Inbound và outbound firewall rules

### Inbound

Traffic đi vào server.

Ví dụ client:

```text
Client → TCP/443 → Web server
```

### Outbound

Traffic server initiate ra ngoài.

Ví dụ:

```text
App server → TCP/1433 → Database
```

Không phải mọi application failure đều do inbound firewall.

---

# 60. Firewall troubleshooting

Workflow logic:

```text
Application expected flow
        ↓
Source / destination
        ↓
Protocol
        ↓
Port
        ↓
Is server listening?
        ↓
Route/connectivity
        ↓
Firewall profile
        ↓
Effective rule
        ↓
Application-level response
```

Điểm rất quan trọng:

> Đừng disable toàn bộ firewall để chứng minh “firewall có phải lỗi hay không” nếu có phương pháp read-only hoặc narrow test an toàn hơn.

Microsoft cũng hỗ trợ firewall rules theo program/service, IP, protocol/port và interface conditions. [Microsoft Learn](https://learn.microsoft.com/en-us/windows/security/operating-system-security/network-security/windows-firewall/?utm_source=chatgpt.com)

---

# 61. Windows network configuration

Một server network configuration cơ bản gồm:

```text
NIC
IP address
Prefix/subnet mask
Default gateway
DNS servers
Routes
Network profile
```

### Read-only verification

```powershell
Get-NetAdapter
```

```powershell
Get-NetIPConfiguration
```

```powershell
Get-NetIPAddress
```

```powershell
Get-NetRoute
```

```powershell
Get-DnsClientServerAddress
```

---

# 62. Network troubleshooting theo layer

Ví dụ application `APP01` cần connect `DB01:1433`.

Không nên bắt đầu bằng restart server.

Đi theo chain:

```text
1. Local NIC up?
2. Correct IP?
3. Correct route?
4. Gateway reachable?
5. DB01 resolves correctly?
6. Destination IP reachable?
7. TCP/1433 reachable?
8. DB service listening?
9. Firewall permits?
10. Application authentication works?
```

PowerShell:

```powershell
Test-NetConnection DB01 -Port 1433
```

Kết quả giúp phân biệt:

```text
NameResolutionSucceeded
RemoteAddress
TcpTestSucceeded
```

Nhưng vẫn không chứng minh database query thành công.

---

# 63. Storage fundamentals

Bạn phải phân biệt:

```text
Disk
Partition
Volume
Filesystem
Drive letter / mount point
```

### Disk

Block storage device Windows nhìn thấy.

### Partition

Logical region trên disk.

### Volume

Storage unit được OS quản lý/use.

### Filesystem

Cách files/directories được tổ chức, ví dụ NTFS hoặc ReFS.

### Drive letter

Ví dụ:

```text
C:
D:
E:
```

Không phải mọi volume bắt buộc phải dùng drive letter.

---

# 64. Storage verification

Microsoft Storage PowerShell module cung cấp `Get-Disk` và `Get-Volume` cho việc inspect disk/volume objects. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/storage/get-disk?view=windowsserver2025-ps&utm_source=chatgpt.com)

```powershell
Get-Disk
```

```powershell
Get-Partition
```

```powershell
Get-Volume
```

Bạn phải biết xem:

```text
HealthStatus
OperationalStatus
FileSystem
Size
SizeRemaining
DriveLetter
```

---

# 65. Disk capacity incidents

Disk full có thể gây rất nhiều symptom:

```text
Application cannot write
Database stops
Backup fails
Logs stop
Windows Update fails
Temporary files cannot be created
Service crashes
```

Do đó free space là một core health check.

Nhưng khi disk low:

> Không tự ý delete files chỉ để giải phóng space.

Trước hết xác định:

```text
Folder growth
Application owner
Retention policy
Log rotation
Temporary vs business data
Backup status
Cleanup procedure
```

---

# 66. Windows Update và patch control

Patching là risk-managed operational activity.

Các concept cần hiểu:

```text
Security update
Quality update
Servicing update
Reboot requirement
Maintenance window
Dependency
Application compatibility
Post-patch validation
Rollback/recovery
```

Một infrastructure engineer không nên coi patching là:

```text
Click Install
→ reboot
→ done
```

---

# 67. Enterprise patch workflow

Một conceptual workflow tốt:

```text
Identify required update
        ↓
Assess applicability/risk
        ↓
Check vendor/application compatibility
        ↓
Approved change
        ↓
Confirm backup/recovery readiness
        ↓
Maintenance window
        ↓
Install
        ↓
Reboot if required
        ↓
OS verification
        ↓
Service/application verification
        ↓
Monitoring
        ↓
Evidence / change closure
```

Nếu server là:

```text
Domain Controller
Database server
Cluster node
Critical middleware
```

reboot planning có thể phức tạp hơn rất nhiều.

---

# 68. Pending reboot awareness

Sau updates hoặc role/configuration changes, server có thể cần restart.

Operationally, cần phân biệt:

```text
Update installed
```

với:

```text
Update fully effective and server validated
```

Nếu pending reboot nhưng ticket được đóng là “completed successfully”, evidence chưa hoàn chỉnh.

---

# 69. Event Viewer

Event Viewer là GUI để inspect Windows Event Logs.

Một event thường có context như:

```text
Log name/channel
Provider
Event ID
Level
Timestamp
Computer
User/context
Message
XML details
```

Event Viewer là core troubleshooting interface nhưng không nên sử dụng theo kiểu:

> Tìm một event màu đỏ rồi coi đó là root cause.

---

# 70. Windows Event Logs

Ba classic logs rất quan trọng:

### System

OS components, drivers, services và system-related events.

### Application

Applications/services ghi events.

### Security

Security audit events, tùy audit policy.

Ngoài ra:

```text
Applications and Services Logs
```

có nhiều provider-specific channels.

---

# 71. Event level không bằng business severity

Ví dụ:

```text
Event Level = Error
```

không tự động có nghĩa:

```text
P1 incident
```

Và:

```text
Information
```

không tự động nghĩa event vô nghĩa.

Business severity phụ thuộc:

```text
Service impact
User impact
Scope
Criticality
Duration
Security implications
```

---

# 72. Event ID và Provider

Event ID phải được đọc cùng Provider.

Không nên chỉ search:

```text
Event ID 1000
```

và áp dụng lời giải từ bất kỳ nguồn nào.

Cần ít nhất:

```text
Provider
Event ID
OS/application version
Message
Timestamp
Context
```

Vì cùng number có thể có meaning khác nhau tùy provider.

---

# 73. Timeline correlation

Một troubleshooting workflow tốt:

```text
User reported 14:32
        ↓
Application failure 14:31
        ↓
Service event 14:30
        ↓
Authentication failures 14:29
        ↓
GPO/service account change 14:25
```

Timeline có giá trị hơn việc đọc events ngẫu nhiên.

---

# 74. PowerShell Event Log inspection

Một công cụ rất hữu ích:

```powershell
Get-WinEvent
```

Ví dụ:

```powershell
Get-WinEvent -LogName System -MaxEvents 20
```

Filter level/time/provider cụ thể thường hiệu quả hơn dump toàn log.

Evidence nên giữ:

```text
Server
Timezone
Query/filter
Start/end timestamp
Log
Provider
Event ID
Relevant message
Extraction timestamp
Ticket/change ID
```

---

# 75. Task Scheduler

Task Scheduler cho phép Windows thực thi program/script dựa trên triggers và conditions.

Một scheduled task có các core components:

| Component  | Ý nghĩa               |
| ---------- | --------------------- |
| Trigger    | Khi nào task chạy     |
| Action     | Task làm gì           |
| Principal  | Identity dùng để chạy |
| Conditions | Điều kiện bổ sung     |
| Settings   | Runtime behavior      |
| History    | Execution records     |

Microsoft ScheduledTasks module cung cấp các cmdlet như `Get-ScheduledTask`, `Get-ScheduledTaskInfo`, `Register-ScheduledTask`, `Start-ScheduledTask` và các cmdlet liên quan. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/?view=windowsserver2025-ps&utm_source=chatgpt.com)

---

# 76. Task Scheduler security

Một scheduled task có thể trở thành privilege escalation vector nếu configuration không an toàn.

Ví dụ nguy hiểm:

```text
Task runs as high privilege
+
script path writable by normal users
```

Attacker hoặc accidental user có thể sửa script rồi chờ privileged task chạy.

Do đó phải kiểm tra:

```text
Run As identity
Script permissions
Executable path
Arguments
Working directory
Credential handling
```

---

# 77. Scheduled task troubleshooting

Khi task “không chạy”, đừng chỉ nhìn State.

Cần xem:

```text
Enabled?
Trigger reached?
Principal valid?
Password/service logon rights?
Action path correct?
Working directory?
Script execution policy?
Network dependency?
Last Run Result?
History/events?
```

Read-only:

```powershell
Get-ScheduledTask
```

```powershell
Get-ScheduledTaskInfo -TaskName 'TaskName'
```

---

# 78. Backup fundamentals

Backup không phải chỉ là “copy data sang nơi khác”.

Một usable backup process phải trả lời:

```text
What?
Where?
When?
Retention?
Protection?
Who can access?
How to restore?
Was restore tested?
```

Mục đích là recovery.

Do đó:

> Backup success ≠ Recovery readiness.

Nguyên tắc này sẽ tiếp tục xuất hiện rất mạnh ở Day 5 và Day 7.

---

# 79. Backup scope

Có thể tồn tại các scope khác nhau:

```text
File/folder backup
Volume backup
System state
Application-aware backup
VM-level backup
Image-level backup
```

Bạn không được mặc định:

> “Có backup server” nghĩa là mọi dữ liệu/application được recover.

Phải xác định protected scope.

---

# 80. Recovery thinking

Một recovery operation phải quan tâm:

```text
Restore point
Data consistency
Application consistency
Permissions
Dependencies
Recovery location
Validation
Business acceptance
```

Một file restore thành công về mặt copy không đồng nghĩa application phục hồi hoàn chỉnh.

---

# 81. RPO và RTO awareness

Syllabus đi sâu RPO/RTO hơn ở các Day sau, nhưng Day 3 nên hiểu foundation.

### RPO — Recovery Point Objective

Chấp nhận mất bao nhiêu dữ liệu theo thời gian.

### RTO — Recovery Time Objective

Mục tiêu về thời gian phục hồi service.

Ví dụ:

```text
Backup mỗi 24 giờ
```

không phù hợp nếu business yêu cầu:

```text
RPO = 1 giờ
```

---

# 82. PowerShell — foundation

PowerShell là command shell + scripting environment + automation framework.

Điểm khác biệt quan trọng so với shell xử lý plain text truyền thống là PowerShell làm việc mạnh với **objects**.

Microsoft mô tả object trong PowerShell chứa type, properties và methods, và objects có thể đi qua pipeline. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_objects?view=powershell-7.6&utm_source=chatgpt.com)

---

# 83. Cmdlet naming

PowerShell cmdlet thường theo convention:

```text
Verb-Noun
```

Ví dụ:

```powershell
Get-Service
Get-Process
Get-Volume
Get-NetAdapter
Restart-Service
Stop-Service
New-LocalUser
```

Verb giúp mô tả action.

Noun mô tả resource/object.

---

# 84. Get-Command

Khi không nhớ command:

```powershell
Get-Command
```

Ví dụ:

```powershell
Get-Command *Service*
```

Đây tốt hơn việc đoán cmdlet name.

---

# 85. Get-Help

```powershell
Get-Help Get-Service
```

Chi tiết hơn:

```powershell
Get-Help Get-Service -Full
```

Examples:

```powershell
Get-Help Get-Service -Examples
```

Trong enterprise, biết đọc built-in help/documentation quan trọng hơn thuộc hàng trăm commands.

---

# 86. PowerShell objects

Ví dụ:

```powershell
Get-Service
```

không chỉ trả “text table”.

Nó trả objects.

Bạn có thể inspect:

```powershell
Get-Service | Get-Member
```

Bạn sẽ thấy properties/methods của object.

Đây là foundation của PowerShell automation.

---

# 87. Properties

Ví dụ:

```powershell
Get-Service | Select-Object Name, Status
```

Ở đây:

```text
Name
Status
```

là properties của service objects.

Bạn không parsing text bằng vị trí column như nhiều shell workflow truyền thống.

---

# 88. Pipeline

Pipeline dùng `|`.

Microsoft định nghĩa pipeline là chuỗi commands trong đó output object của command trước được chuyển cho command tiếp theo. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_pipelines?view=powershell-7.6&utm_source=chatgpt.com)

Ví dụ:

```powershell
Get-Service |
    Where-Object Status -eq 'Running' |
    Select-Object Name, Status
```

Flow:

```text
Get-Service
    ↓
Service objects
    ↓
Where-Object
    ↓
Filtered service objects
    ↓
Select-Object
    ↓
Selected properties
```

---

# 89. Where-Object

Filtering:

```powershell
Get-Service |
Where-Object Status -eq 'Stopped'
```

Bạn phải hiểu:

```text
Filter as early as practical
```

để giảm unnecessary data, đặc biệt trong remote/large-scale queries.

---

# 90. Select-Object

```powershell
Get-Process |
Select-Object Name, Id, CPU
```

Mục đích:

```text
Select useful properties
Create operational output
Prepare export/report
```

---

# 91. Sort-Object

```powershell
Get-Process |
Sort-Object CPU -Descending
```

Có thể giúp tìm processes đáng chú ý.

Nhưng một lần nữa:

> High CPU process ≠ root cause automatically.

---

# 92. Format-\* cmdlets

Ví dụ:

```powershell
Get-Service |
Format-Table Name, Status
```

`Format-Table` dành cho presentation.

Không nên thường xuyên dùng nó giữa pipeline khi bạn còn cần object data cho processing/export.

Một common beginner mistake:

```powershell
Get-Service |
Format-Table |
Export-Csv
```

Output sẽ không như mong muốn vì formatting objects đã thay data structure.

Better conceptual flow:

```text
Get
→ Filter
→ Select
→ Export
```

Formatting nên chủ yếu ở output-to-human stage.

---

# 93. PowerShell remoting

Như đã nói, trên Windows, PowerShell remoting thường sử dụng WinRM. Nó hỗ trợ temporary commands, interactive sessions và persistent sessions. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements?view=powershell-7.6&utm_source=chatgpt.com)

Ví dụ:

```powershell
Invoke-Command -ComputerName SERVER01 -ScriptBlock {
    Get-Service
}
```

Và:

```powershell
Enter-PSSession SERVER01
```

---

# 94. Why remoting matters

Nếu quản trị 100 servers, workflow:

```text
RDP từng server
```

không scale tốt.

PowerShell remoting cho phép:

```text
Repeatability
Consistency
Automation
Central evidence collection
Reduced GUI dependency
```

Nhưng phải được kiểm soát bởi:

```text
Authorization
Network restrictions
Endpoint configuration
Logging
Credential protection
```

---

# 95. Execution Policy

PowerShell Execution Policy kiểm soát điều kiện mà PowerShell dùng để load configuration files và chạy scripts.

Microsoft nhấn mạnh đây là một **safety feature**, không nên xem như một complete security boundary. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7&utm_source=chatgpt.com)

Common values gồm:

```text
Restricted
RemoteSigned
AllSigned
Unrestricted
Bypass
Undefined
```

---

# 96. Execution Policy scopes

Execution Policy có thể tồn tại ở các scopes khác nhau như:

```text
MachinePolicy
UserPolicy
Process
CurrentUser
LocalMachine
```

Check:

```powershell
Get-ExecutionPolicy -List
```

Một policy có thể bị Group Policy chi phối.

Do vậy nếu script không chạy, không nên ngay lập tức:

```powershell
Set-ExecutionPolicy Bypass
```

ở LocalMachine scope.

Đó là một classic unsafe workaround.

---

# 97. Administrative scripting

Một operations script tốt không chỉ “chạy được”.

Nó nên có properties như:

```text
Predictable
Readable
Logged
Repeatable
Safe
Parameterised where appropriate
Error-aware
Least privilege
No embedded real secrets
```

---

# 98. Idempotency awareness

Một concept bổ sung hữu ích:

Nếu script chạy nhiều lần, nó không nên phá state hoặc tạo duplicates một cách không kiểm soát.

Ví dụ script “ensure folder exists” tốt hơn script luôn cố tạo folder và fail khi folder đã tồn tại.

Day 12 sẽ đi sâu hơn vào automation testing, nhưng mindset nên bắt đầu từ Day 3.

---

# 99. Error handling awareness

PowerShell commands có thể tạo:

```text
Success output
Non-terminating errors
Terminating errors
```

Trong administrative automation, cần hiểu rằng:

```text
Command ran
```

không đồng nghĩa:

```text
Desired state achieved
```

Mọi change cần verification.

---

# 100. Output và evidence

PowerShell rất hữu ích cho evidence collection.

Ví dụ:

```powershell
Get-ComputerInfo
```

```powershell
Get-Service
```

```powershell
Get-NetIPConfiguration
```

```powershell
Get-Volume
```

```powershell
Get-WinEvent
```

Output có thể được:

```text
Recorded
Exported
Attached to ticket
Compared before/after
Timestamped
```

Nhưng khi export phải tránh vô tình thu thập:

```text
Passwords
Tokens
Private keys
Sensitive personal data
Secrets
```

---

# 101. Windows Server operational maintenance

Đây là phần kết nối toàn bộ module.

Infrastructure Operations không phải:

```text
Log in
→ click things
→ server works
```

Một operational workflow trưởng thành hơn là:

```text
Observe
    ↓
Identify asset
    ↓
Establish baseline
    ↓
Verify symptom
    ↓
Determine scope/impact
    ↓
Collect read-only evidence
    ↓
Identify likely component
    ↓
Check authorization/change
    ↓
Perform smallest safe action
    ↓
Verify technical result
    ↓
Verify service result
    ↓
Monitor
    ↓
Document evidence
    ↓
Rollback or close/escalate
```

---

# 102. Configuration baseline

Baseline trả lời:

> “Normal của server này là gì?”

Có thể gồm:

```text
Hostname
OS version
Edition
Build
Roles/features
IP configuration
DNS
Volumes
Critical services
Firewall profile
Domain membership
Patch state
Scheduled tasks
Backup status
```

Không có baseline, bạn khó phân biệt:

```text
Normal
vs
Drift
vs
Incident
```

---

# 103. Configuration drift

Configuration drift xảy ra khi state hiện tại khác expected/approved baseline.

Ví dụ:

```text
Service changed from Automatic → Disabled
Firewall rule added
DNS changed
Local Administrator membership changed
Scheduled task disappeared
Disk added without inventory update
```

Không phải mọi drift đều malicious.

Có thể do:

```text
Approved change chưa document
Manual emergency fix
GPO
Automation
Application installer
Human error
```

Operations cần tìm **change evidence** trước khi sửa.

---

# 104. Read-only verification trước configuration change

Một discipline quan trọng:

```text
Read-only first
```

Ví dụ khi service fail:

Đầu tiên:

```powershell
Get-Service
Get-Process
Get-WinEvent
```

Không phải:

```powershell
Restart-Service
```

ngay lập tức.

Restart có thể:

```text
Temporarily hide root cause
Destroy transient evidence
Impact dependent applications
Create new events
```

---

# 105. “Restart fixes it” không phải root-cause analysis

Restart có thể là approved recovery action.

Nhưng cần phân biệt:

```text
Recovery action
vs
Root cause
```

Ví dụ:

```text
Memory leak
→ restart service
→ service recovers
```

Restart giải quyết symptom tạm thời.

Root cause vẫn là memory/resource behavior.

Ticket tốt phải ghi cả hai.

---

# 106. Troubleshooting từ ít rủi ro tới nhiều rủi ro

Một hierarchy tốt:

```text
Observe
↓
Read configuration
↓
Read logs
↓
Test connectivity
↓
Compare baseline
↓
Restart isolated service nếu authorized
↓
Change configuration
↓
Restart server
↓
Rollback/restore
```

Không phải incident nào cũng cần đi hết chain.

Mục tiêu là:

> smallest safe action that restores service while preserving evidence.

---

# 107. Evidence collection

Một Windows Server incident evidence package tốt thường có:

| Evidence              | Ví dụ                                     |
| --------------------- | ----------------------------------------- |
| Asset identity        | Hostname, VM/asset ID                     |
| Time                  | Timestamp + timezone                      |
| OS state              | Version, edition, build                   |
| Service state         | `Get-Service` output                      |
| Network               | IP/DNS/route                              |
| Storage               | `Get-Volume`                              |
| Event evidence        | Log, Provider, Event ID                   |
| Security context      | Relevant group/policy, không chứa secrets |
| Change reference      | Change/ticket ID                          |
| Before/after          | State trước và sau                        |
| Recovery verification | Application/user-facing test              |
| Outstanding risk      | Những gì chưa giải quyết                  |

Evidence phải đủ để người khác reconstruct được:

```text
What happened?
When?
Where?
Who/what changed?
What was checked?
What was changed?
Did it recover?
```

---

# 108. Timestamp và timezone

Một detail nhỏ nhưng cực kỳ quan trọng.

Nếu server log dùng:

```text
UTC
```

nhưng ticket viết:

```text
Vietnam local time
```

mà không ghi timezone, timeline investigation có thể lệch bảy giờ.

Khi collect evidence nên ghi rõ:

```text
Timestamp
Timezone
Source system
```

Day 10 sẽ đi sâu hơn vào centralized logging và timestamp normalization.

---

# 109. Security implications xuyên suốt Day 3

Mọi Windows Server operation đều có security implications.

### Accounts

Không cấp unnecessary admin privileges.

### Services

Không dùng overprivileged service accounts.

### Files

Không cấp `Everyone: Full Control` chỉ để “hết lỗi permission”.

### Firewall

Không mở `Any → Any` nếu chỉ cần một source subnet/port.

### RDP

Không expose không kiểm soát.

### WinRM

Chỉ cho authorized administrators/endpoints.

### PowerShell

Không disable controls để chạy unknown scripts.

### Registry

Không modify undocumented keys.

### Scheduled Tasks

Bảo vệ script path và credentials.

### Backups

Backup chứa production data cũng là sensitive asset.

---

# 110. Common operational anti-patterns

Đây là những lỗi tư duy bạn phải tránh ngay từ Day 3.

| Anti-pattern                             | Vì sao nguy hiểm                    |
| ---------------------------------------- | ----------------------------------- |
| Restart ngay khi thấy lỗi                | Mất evidence, không biết root cause |
| Disable firewall để test                 | Mở attack surface                   |
| Add user vào Administrators              | Bypass permission design            |
| Give Everyone Full Control               | Security exposure                   |
| `ExecutionPolicy Bypass` toàn máy        | Làm yếu safety controls             |
| Edit Registry theo forum                 | Configuration risk                  |
| Delete logs để free disk                 | Mất audit evidence                  |
| Reboot production không change           | Service impact                      |
| Assume Event ID = root cause             | Thiếu context                       |
| Assume Ping = application healthy        | Chỉ kiểm tra một phần connectivity  |
| Assume service Running = service healthy | Không verify application            |
| Assume backup success = restore ready    | Chưa chứng minh recovery            |

---

# 111. L1/L2 operational boundaries

Syllabus đặt Day 3 trong overall Infrastructure Operations model, nên phải giữ boundary từ Day 1. Infrastructure_Operations_14_Da…

Một practical conceptual boundary:

### L1 thường có thể

Trong approved runbook:

```text
Check health
Inspect logs
Verify service state
Verify connectivity
Collect evidence
Run approved health checks
Restart explicitly authorized low-risk service
Update ticket
Escalate
```

### L2 thường có thể

Tùy organization:

```text
Deeper log correlation
Permission investigation
Service configuration analysis
Network/firewall verification
Approved configuration corrections
PowerShell diagnostics
Change implementation
Rollback
```

### Escalate

Khi cần:

```text
Domain-wide GPO change
AD architecture change
Security incident response
Unknown privileged access
Production firewall architecture change
Critical Registry changes
Storage corruption
Data recovery with business impact
Unsupported OS remediation
Major patch failure
Application/vendor-specific deep changes
```

Boundary thực tế luôn theo:

```text
Organization policy
Access rights
Runbook
Change approval
Service ownership
```

---

# 112. Một complete troubleshooting example

Giả sử incident:

> User không thể truy cập `\\FILE01\Finance`.

Một fresher dễ làm:

```text
Restart FILE01
```

Nhưng một Infrastructure Operations engineer nên suy luận:

```text
User identity?
        ↓
Can FILE01 resolve?
        ↓
Can FILE01 be reached?
        ↓
SMB TCP/445 reachable?
        ↓
Server service healthy?
        ↓
Share exists?
        ↓
Share ACL?
        ↓
NTFS ACL?
        ↓
AD group membership?
        ↓
Recent GPO/change?
        ↓
Relevant Event Logs?
```

Nếu:

```text
\\10.20.1.50\Finance works
\\FILE01\Finance fails
```

focus chuyển sang:

```text
DNS/name resolution
```

Nếu:

```text
Administrator works
User fails
```

focus chuyển sang:

```text
Identity/authorization/permissions
```

Nếu mọi user fail và:

```text
TCP/445 unavailable
```

focus chuyển sang:

```text
Service/firewall/network
```

Đây là tư duy Day 3 quan trọng hơn việc thuộc GUI menu.

---

# 113. Một complete service troubleshooting example

Incident:

> Application không hoạt động sau reboot.

Reasoning:

```text
OS booted?
    ↓
Critical app service exists?
    ↓
Service Running?
    ↓
Startup Type?
    ↓
Dependencies healthy?
    ↓
Log On account valid?
    ↓
Executable/config present?
    ↓
Relevant System/Application events?
    ↓
Required port listening?
    ↓
Backend dependency reachable?
    ↓
Application health check?
```

Ví dụ service `Running` nhưng application vẫn fail:

```text
Service
Running
   ↓
TCP port
Listening
   ↓
HTTP endpoint
500
```

Kết luận:

```text
Windows service layer healthy enough to run
but application layer unhealthy
```

Không tiếp tục restart service mù quáng.

---

# 114. Một complete permission troubleshooting example

User:

```text
CORP\Alice
```

không thể Modify:

```text
\\FILE01\Projects
```

Bạn phải kiểm tra:

```text
Is Alice authenticating as CORP\Alice?
        ↓
Relevant group memberships?
        ↓
Share ACL?
        ↓
NTFS ACL?
        ↓
Inheritance?
        ↓
Explicit Deny?
        ↓
Effective access?
        ↓
Group token refreshed?
```

Một common issue:

Admin vừa add user vào group nhưng existing logon token chưa reflect membership.

Vì vậy:

```text
AD membership changed
```

không phải lúc nào existing session lập tức có expected token.

---

# 115. Một complete remote-management troubleshooting example

PowerShell:

```powershell
Enter-PSSession SERVER01
```

fails.

Không nên kết luận WinRM “hỏng”.

Chain:

```text
Does SERVER01 resolve?
        ↓
Network reachable?
        ↓
WinRM service available?
        ↓
Firewall permits?
        ↓
Listener configured?
        ↓
Authentication successful?
        ↓
User authorized for endpoint?
        ↓
Policy/configuration issue?
```

PowerShell remoting có system requirements, WinRM dependency và user/endpoint authorization requirements riêng. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_remote_requirements?view=powershell-7.6&utm_source=chatgpt.com)

---

# 116. GUI vs PowerShell

Bạn cần biết cả hai.

### GUI tốt cho

```text
Discovery
Visualization
Learning relationships
Occasional administration
Manual inspection
```

### PowerShell tốt cho

```text
Repeatability
Automation
Remote operations
Filtering
Bulk inspection
Evidence collection
Scale
```

Một capable Windows Operations engineer không phải người “không bao giờ dùng GUI”.

Cũng không phải người “chỉ dùng GUI”.

Mục tiêu là:

> Chọn interface an toàn và hiệu quả nhất cho task.

---

# 117. Read-only command map cho Day 3

Bảng này chưa phải lab; nó là command vocabulary bạn cần nhận biết.

| Mục đích            | Read-only command            |
| ------------------- | ---------------------------- |
| OS information      | `Get-ComputerInfo`           |
| Hostname            | `hostname`                   |
| Services            | `Get-Service`                |
| Processes           | `Get-Process`                |
| Local users         | `Get-LocalUser`              |
| Local groups        | `Get-LocalGroup`             |
| Group members       | `Get-LocalGroupMember`       |
| File ACL            | `Get-Acl`                    |
| Network adapters    | `Get-NetAdapter`             |
| IP config           | `Get-NetIPConfiguration`     |
| IP addresses        | `Get-NetIPAddress`           |
| Routes              | `Get-NetRoute`               |
| DNS client          | `Get-DnsClientServerAddress` |
| DNS lookup          | `Resolve-DnsName`            |
| Connectivity        | `Test-NetConnection`         |
| Firewall            | `Get-NetFirewallRule`        |
| Disks               | `Get-Disk`                   |
| Partitions          | `Get-Partition`              |
| Volumes             | `Get-Volume`                 |
| Events              | `Get-WinEvent`               |
| Scheduled tasks     | `Get-ScheduledTask`          |
| Task runtime info   | `Get-ScheduledTaskInfo`      |
| Group Policy result | `gpresult /r`                |
| PowerShell policy   | `Get-ExecutionPolicy -List`  |

Day 3 cần ưu tiên các command **read-only** trước.

Configuration-changing commands sẽ được dùng có kiểm soát trong Module Thực hành.

---

# 118. Mental model cần ghi nhớ

Bạn có thể gói toàn bộ lý thuyết Day 3 vào mô hình:

```text
WINDOWS SERVER
│
├── Platform
│   ├── Version
│   ├── Edition
│   ├── Build
│   └── Core/Desktop Experience
│
├── Management
│   ├── Server Manager
│   ├── Windows Admin Center
│   ├── PowerShell
│   └── WinRM
│
├── Compute
│   ├── Processes
│   └── Services
│
├── Configuration
│   ├── Registry
│   ├── Local Policy
│   └── Group Policy
│
├── Identity
│   ├── Local users/groups
│   └── AD domain identities
│
├── Access Control
│   ├── NTFS
│   └── Share permissions
│
├── Infrastructure Roles
│   ├── AD DS
│   ├── DNS
│   └── DHCP
│
├── Connectivity
│   ├── IP
│   ├── DNS
│   ├── Routes
│   ├── Firewall
│   ├── RDP
│   └── WinRM
│
├── Storage
│   ├── Disk
│   ├── Partition
│   ├── Volume
│   └── Filesystem
│
├── Maintenance
│   ├── Updates
│   ├── Backup
│   └── Scheduled Tasks
│
└── Observability
    ├── Event Viewer
    ├── Event Logs
    └── PowerShell evidence
```

---

# 119. Day 3 Theory — Operational reasoning model

Mỗi khi gặp Windows Server incident, hãy tự đặt chuỗi câu hỏi:

```text
WHAT?
Symptom thật sự là gì?

WHERE?
Server, service, network, storage hay identity?

WHEN?
Bắt đầu lúc nào?

WHO?
User/service account nào bị ảnh hưởng?

SCOPE?
Một user, một server hay toàn environment?

CHANGE?
Có patch/change/GPO/deployment nào gần thời điểm lỗi?

VERIFY?
Evidence nào chứng minh symptom?

DEPENDENCY?
DNS, AD, network, storage, backend nào liên quan?

SAFE ACTION?
Read-only check nào làm trước?

RECOVERY?
Action nào ít risk nhất?

VALIDATION?
Làm sao chứng minh service thật sự restored?

EVIDENCE?
Cần giữ log/output gì?

ESCALATION?
Đã vượt quyền/runbook/expertise chưa?
```

Đây chính là cách biến kiến thức Windows Server từ “biết command” thành **Infrastructure Operations skill**.

---

# 120. Module Lý thuyết Day 3 — Completion Checklist

Sau khi học kỹ module này, bạn cần tự giải thích được toàn bộ những câu sau:

| Năng lực         | Bạn phải trả lời được                                                                  |
| ---------------- | -------------------------------------------------------------------------------------- |
| Architecture     | Kernel/user mode, services/processes/configuration liên quan thế nào?                  |
| Version          | Server đang chạy generation/build nào?                                                 |
| Lifecycle        | Supported hay legacy?                                                                  |
| Edition          | Standard/Datacenter/Azure Edition/Essentials khác nhau ở đâu về operational awareness? |
| Installation     | Core hay Desktop Experience?                                                           |
| Management       | Khi nào dùng Server Manager, WAC, PowerShell, WinRM?                                   |
| Services         | Service state, startup, account, dependencies là gì?                                   |
| Processes        | Process khác service thế nào?                                                          |
| Registry         | Registry dùng làm gì và vì sao high risk?                                              |
| Identity         | Local account khác domain account thế nào?                                             |
| Security         | Least privilege áp dụng ở đâu?                                                         |
| NTFS             | ACL/inheritance/Allow/Deny hoạt động thế nào?                                          |
| SMB              | NTFS và share permission phối hợp ra sao?                                              |
| AD DS            | Domain/DC/user/group/computer/OU là gì?                                                |
| Group Policy     | Vì sao local setting có thể bị override?                                               |
| DNS              | Name resolution hoạt động ở mức nào?                                                   |
| DHCP             | Scope/lease/options là gì?                                                             |
| RDP              | RDP dependency và risk là gì?                                                          |
| WinRM            | Tại sao WinRM quan trọng cho remote administration?                                    |
| Firewall         | Direction/profile/protocol/port/scope là gì?                                           |
| Network          | IP/gateway/DNS/route/connectivity phải check theo thứ tự nào?                          |
| Storage          | Disk/partition/volume/filesystem khác nhau thế nào?                                    |
| Updates          | Vì sao patch cần change/reboot/post-check?                                             |
| Events           | Provider/Event ID/timestamp/context quan trọng thế nào?                                |
| Tasks            | Trigger/action/principal/Last Run Result là gì?                                        |
| Backup           | Backup success khác recovery readiness thế nào?                                        |
| PowerShell       | Cmdlet/object/property/pipeline hoạt động ra sao?                                      |
| Remoting         | PowerShell remoting phụ thuộc gì?                                                      |
| Execution Policy | Nó là gì và không phải gì?                                                             |
| Scripting        | Script operations tốt cần đặc tính nào?                                                |
| Operations       | Verification/evidence/rollback/escalation được đặt ở đâu?                              |

Nếu bạn hiểu và có thể giải thích được bảng này bằng lời của mình, bạn đã có **theoretical foundation đầy đủ theo syllabus của Day 3**. Program Summary của syllabus cũng xác định đúng Day 3 tập trung vào **Windows Server editions, AD, services, events, security và PowerShell**. Infrastructure_Operations_14_Da…

Bước tiếp theo phù hợp nhất là **Module Thực hành Day 3**: tôi sẽ biến toàn bộ lý thuyết trên thành một lab Windows Server hoàn chỉnh theo đúng Assignment của syllabus, từ provision VM → baseline → users/groups → NTFS/SMB → firewall/WinRM → services/processes → storage/network/update → Event Logs → PowerShell health-check → Task Scheduler → failure injection → evidence → rollback/escalation → final maintenance checklist.
