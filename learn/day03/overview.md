# Day 3 — Windows Server Administration Foundations

Day 3 chuyển trọng tâm từ Linux/RHEL sang **Windows Server operations**. Theo syllabus, mục tiêu không phải chỉ biết sử dụng GUI mà phải hình thành cách vận hành Windows Server theo góc nhìn Infrastructure Operations: xác định đúng **Windows Server generation/edition/installation mode**, quản lý identity và permissions, kiểm tra services/network/storage/security/patching, sử dụng **PowerShell** cho administration và evidence collection, đồng thời biết **verification, troubleshooting, rollback, evidence và escalation**. Infrastructure_Operations_14_Da…

Syllabus cũng lưu ý rằng “Windows Server all editions” được hiểu là phải có **edition-aware operational coverage**; không có nghĩa mọi edition đều giống nhau hoặc tất cả đều phải được cài thực hành. Các feature cụ thể cần được kiểm tra theo release/edition được sử dụng trong lab. Infrastructure_Operations_14_Da…

Tôi sẽ chia toàn bộ Day 3 thành đúng **2 module lớn: Lý thuyết và Thực hành**. Đây là bản đồ nội dung trước; khi bạn yêu cầu từng module, tôi sẽ đi sâu theo từng topic với cơ chế, command, expected output, verification, troubleshooting, evidence, rollback và escalation.

---

# Module 1 — LÝ THUYẾT: Windows Server Administration & Operations

Module này bao phủ toàn bộ **Concept/Lecture** của Day 3 và phần **Additive Concept/Lecture** trong syllabus. Infrastructure_Operations_14_Da…

## 1. Windows Server architecture

Phần đầu tiên cần hiểu Windows Server hoạt động như một operating system dành cho enterprise infrastructure, bao gồm quan hệ giữa:

**Hardware → kernel → drivers → services/processes → security subsystem → networking → storage → server roles/features → management interfaces.**

Các khái niệm sẽ học gồm:

- Windows kernel và user mode/kernel mode ở mức vận hành.
- Processes, threads, services và drivers.
- Service Control Manager.
- Windows Registry.
- Event subsystem.
- Windows networking stack.
- NTFS filesystem và storage stack.
- Server Roles và Features.
- Local identity/security subsystem.
- Domain-integrated identity khi server tham gia Active Directory.
- GUI management, PowerShell và remote management.

Mục tiêu không phải học Windows internals chuyên sâu mà phải hiểu đủ để biết **một lỗi đang nằm ở layer nào**.

---

## 2. Windows Server generations, editions và lifecycle awareness

Phải biết cách phân biệt:

- Windows Server generations đang được organization hỗ trợ.
- Legacy Windows Server generations vẫn có thể tồn tại.
- Support status.
- Patch/support implications.
- Feature differences theo release.

Sau đó phân biệt các edition:

- **Windows Server Standard**
- **Windows Server Datacenter**
- **Windows Server Datacenter: Azure Edition**
- **Windows Server Essentials / legacy awareness**

Điểm quan trọng là không được giả định:

> Standard = Datacenter = Azure Edition.

Sẽ học cách xây dựng **release-by-edition matrix**, trong đó ghi:

- Windows Server release.
- Edition.
- Installation option.
- Supported roles/features.
- Management method.
- Patch constraints.
- Backup constraints.
- Vendor support status.
- Lab coverage status.

Mỗi edition/version cần được đánh dấu:

- **Hands-on**
- **Demonstrated**
- **Reference-only**

để không tạo false evidence rằng đã thực hành trên edition chưa từng sử dụng.

---

## 3. Server Core vs Desktop Experience

Đây là một phần rất quan trọng.

### Desktop Experience

Có GUI đầy đủ hơn, thuận tiện:

- Server Manager.
- Event Viewer.
- MMC consoles.
- GUI configuration tools.

### Server Core

Có attack surface và GUI footprint nhỏ hơn, thường được quản trị thông qua:

- PowerShell.
- Windows Admin Center.
- Remote management.
- `sconfig`.
- command-line tools.

Cần hiểu sự khác biệt về:

- Management workflow.
- Resource footprint.
- Attack surface.
- Patch/reboot considerations.
- Troubleshooting.
- Remote administration dependency.

Syllabus yêu cầu có **Server Core/Desktop Experience comparison**, bằng second VM hoặc instructor demonstration. Infrastructure_Operations_14_Da…

---

## 4. Windows Server Manager

Học cách sử dụng **Server Manager** như một management console để:

- Xem server inventory.
- Kiểm tra Local Server.
- Add/remove Roles and Features.
- Theo dõi services/events.
- Quản lý multiple Windows Servers.
- Xem warning và operational status.

Đồng thời phải hiểu giới hạn của Server Manager: GUI chỉ là một management interface; khi troubleshoot cần kiểm chứng bằng PowerShell, Event Logs và system state.

---

## 5. Windows Admin Center awareness

Day 3 chỉ yêu cầu mức **awareness**, tức là hiểu Windows Admin Center là web-based management platform dùng cho Windows infrastructure.

Cần biết nó có thể hỗ trợ các workflow như:

- Server administration.
- Services.
- Processes.
- Storage.
- Firewall.
- Certificates.
- Event logs.
- Updates.
- PowerShell.
- Remote management.

Không mặc định rằng organization nào cũng triển khai Windows Admin Center.

---

# 6. Windows services

Đây sẽ là một trong các phần trọng tâm nhất của Day 3.

Cần học:

- Service name vs display name.
- Service status.
- Startup Type:
  - Automatic
  - Automatic (Delayed Start)
  - Manual
  - Disabled
- Service dependencies.
- Service account.
- Service recovery behavior.

Các hoạt động cơ bản:

- Query service.
- Start.
- Stop.
- Restart.
- Inspect startup configuration.
- Kiểm tra dependency.
- Kiểm tra related Event Logs.

PowerShell điển hình:

```powershell
Get-Service
Get-Service -Name Spooler
Restart-Service -Name Spooler
```

Nhưng tư duy vận hành quan trọng hơn command:

```text
Service down
   ↓
Check service status
   ↓
Check dependencies
   ↓
Check Event Logs
   ↓
Check service account / permissions
   ↓
Check port / storage / configuration dependency
   ↓
Authorized corrective action
   ↓
Verify service
   ↓
Verify application
```

Không được restart service như phản xạ đầu tiên mà chưa thu thập evidence nếu incident cần investigation.

---

# 7. Windows processes

Phân biệt rõ:

**Service ≠ Process.**

Một service có thể chạy trong một process; một process cũng có thể host nhiều services.

Nội dung gồm:

- Process ID — PID.
- CPU usage.
- Memory usage.
- Executable path.
- Parent/child relationship ở mức operational.
- Process ownership.
- Hung/high-resource process.

PowerShell:

```powershell
Get-Process
```

Sẽ học cách liên hệ:

```text
Service
→ Process
→ PID
→ Network port
→ Event
→ Application behavior
```

---

# 8. Startup configuration

Không chỉ service mới khởi động cùng Windows.

Cần hiểu những nguồn startup phổ biến như:

- Services.
- Scheduled Tasks.
- Registry startup locations.
- Server Roles/Features.
- Group Policy controlled settings.

Mục tiêu là biết tìm nguyên nhân khi:

- service không tự start sau reboot;
- application không chạy;
- scheduled operation không chạy;
- configuration bị thay đổi sau policy refresh.

---

# 9. Windows Registry fundamentals

Registry là hierarchical configuration database của Windows.

Các root keys cần nhận diện:

```text
HKEY_LOCAL_MACHINE
HKEY_CURRENT_USER
HKEY_CLASSES_ROOT
HKEY_USERS
HKEY_CURRENT_CONFIG
```

Day 3 tập trung vào:

- Registry structure.
- Keys.
- Values.
- Data types.
- Configuration lookup.
- Operational safety.

Registry modification là thao tác có rủi ro.

Nguyên tắc:

```text
Read first
→ Export/backup if modification is authorized
→ Change smallest possible scope
→ Verify
→ Rollback if necessary
```

Không chỉnh Registry theo random Internet fix trong production.

---

# 10. Local users và local groups

Học cách quản lý local identity:

- Create user.
- Disable/enable account.
- Group membership.
- Password policies.
- Administrative group membership.
- Least privilege.

Các group quan trọng cần nhận diện như:

```text
Administrators
Remote Desktop Users
Users
Backup Operators
```

Không đồng nghĩa:

```text
Need RDP access
=
Need local Administrator
```

Đây là một common privilege mistake.

---

# 11. Local Security Policy

Học vai trò của Local Security Policy trong việc kiểm soát:

- Account policies.
- Password policies.
- Account lockout.
- Local rights.
- Security options.
- Audit policy awareness.

Một trong các mục tiêu là hiểu:

```text
Local Security Policy
vs
Domain Group Policy
```

Domain policy có thể override hoặc kiểm soát nhiều local settings.

---

# 12. NTFS permissions

Đây là topic cực kỳ quan trọng.

Các permissions cơ bản:

- Full Control
- Modify
- Read & Execute
- List Folder Contents
- Read
- Write

Cần hiểu:

- Explicit permission.
- Inherited permission.
- Allow.
- Deny.
- User permission.
- Group permission.
- Effective Access.

Ví dụ:

```text
User
↓
Group A → Read
Group B → Modify
↓
Effective NTFS access
```

Troubleshooting permission không thể chỉ nhìn một ACE.

---

# 13. Share permissions

Khi một folder được truy cập qua SMB share, có hai layer:

```text
Share Permission
+
NTFS Permission
```

Effective network access chịu ảnh hưởng bởi cả hai.

Ví dụ:

```text
Share: Read
NTFS: Modify
```

thì user không tự động có Modify qua SMB.

Sẽ học cách phân biệt:

- Local filesystem access.
- Network share access.
- Share permission.
- NTFS permission.
- Effective access.

---

# 14. Active Directory Domain Services basics

Day 3 chỉ ở foundation level nhưng phải hiểu AD DS vì Windows Server enterprise thường không vận hành độc lập.

Các object quan trọng:

- Domain.
- Domain Controller.
- User.
- Computer.
- Group.
- Organizational Unit.
- Group Policy.

Hiểu basic authentication flow và tại sao domain identity khác local identity.

---

# 15. Local account vs domain account

Phải phân biệt:

```text
SERVER01\administrator
```

với:

```text
CONTOSO\administrator
```

hoặc:

```text
administrator@contoso.example
```

Các vấn đề thực tế:

- account scope;
- authentication source;
- domain membership;
- cached/domain authentication;
- administrative rights;
- audit attribution.

---

# 16. Organizational Units — OU

OU dùng để tổ chức Active Directory objects và hỗ trợ:

- Administrative delegation.
- Group Policy targeting.
- Logical organization.

Không nên nhầm:

```text
OU
≠
Security Group
```

OU không đơn giản là một group chứa users để cấp resource permissions.

---

# 17. Group Policy awareness

Học khái niệm:

- Group Policy Object — GPO.
- Computer Configuration.
- User Configuration.
- Policy inheritance.
- Domain/OU targeting.
- Policy refresh.
- Resultant Policy awareness.

Operational issue điển hình:

> Administrator sửa setting local nhưng một thời gian sau setting quay về trạng thái cũ.

Một nguyên nhân cần kiểm tra là **Group Policy**.

---

# 18. DNS role

Windows DNS đặc biệt quan trọng trong Active Directory environment.

Cần hiểu:

- DNS zones.
- Records.
- Forward lookup.
- Reverse lookup awareness.
- A/AAAA records.
- CNAME.
- PTR.
- SRV record awareness.
- DNS client settings.

Một rule rất quan trọng:

> AD problem thường có thể bắt nguồn từ DNS problem.

---

# 19. DHCP role

Học foundation của DHCP:

- Scope.
- Address pool.
- Exclusions.
- Reservations.
- Lease.
- DHCP options.

Các option điển hình:

- Default Gateway.
- DNS Server.
- DNS domain.

Workflow cơ bản:

```text
DHCPDISCOVER
DHCPOFFER
DHCPREQUEST
DHCPACK
```

---

# 20. RDP

Remote Desktop Protocol dùng cho remote administration.

Cần hiểu:

- RDP access requirement.
- Local/domain permissions.
- Remote Desktop Users.
- Firewall dependency.
- Network reachability.
- Authentication.
- Security considerations.

Troubleshooting RDP phải tách:

```text
Network
→ Firewall
→ RDP listener/service
→ Account permission
→ Authentication
→ Policy
```

---

# 21. WinRM

**Windows Remote Management** là nền tảng quan trọng cho remote PowerShell.

Cần biết:

- WinRM service.
- Listener.
- Authentication.
- Firewall.
- PowerShell Remoting dependency.
- TrustedHosts/domain considerations ở mức foundation.

---

# 22. Windows Defender Firewall

Học các firewall profiles:

- Domain
- Private
- Public

Và firewall rule dimensions:

- Inbound / Outbound.
- Protocol.
- Local port.
- Remote address.
- Program/service.
- Profile.
- Allow/Block.

Operational principle:

Không disable toàn bộ firewall chỉ để “test cho nhanh”.

Ưu tiên:

```text
Inspect
→ Create narrow approved rule
→ Test
→ Verify
→ Remove/rollback if temporary
```

---

# 23. Windows Server network configuration

Phải biết inspect:

- Network adapters.
- IPv4/IPv6.
- IP address.
- Prefix/subnet.
- Default Gateway.
- DNS servers.
- Routes.
- Listening ports.

PowerShell awareness:

```powershell
Get-NetAdapter
Get-NetIPAddress
Get-NetIPConfiguration
Get-NetRoute
Get-NetTCPConnection
```

Troubleshooting flow:

```text
Adapter
→ IP
→ Subnet
→ Gateway
→ Route
→ DNS
→ Firewall
→ Target service
```

---

# 24. Storage và volume management

Học Windows storage objects:

```text
Physical Disk
→ Disk
→ Partition
→ Volume
→ Filesystem
→ Drive Letter / Mount Point
```

Các nội dung chính:

- Disk status.
- Partition.
- Volume.
- NTFS.
- Drive letters.
- Free capacity.
- Basic storage operations.
- Storage failure awareness.

PowerShell:

```powershell
Get-Disk
Get-Partition
Get-Volume
```

---

# 25. Windows Update và patch control

Không được coi patching đơn giản là “bấm Update”.

Phải hiểu operational lifecycle:

```text
Change approval
→ Compatibility/prerequisite review
→ Baseline
→ Backup/recovery readiness
→ Maintenance window
→ Install patches
→ Reboot if required
→ Post-check
→ Evidence
```

Kiểm tra:

- Pending updates.
- Installed updates.
- Reboot status.
- Service health sau patch.
- Application health.
- Patch evidence.

---

# 26. Event Viewer

Học Event Viewer như công cụ troubleshooting và evidence collection.

Các log chính:

```text
System
Application
Security
```

Ngoài ra có:

```text
Applications and Services Logs
```

Cần hiểu các trường:

- Provider.
- Event ID.
- Level.
- Timestamp.
- Computer.
- User.
- Message.
- Record ID.

---

# 27. Windows Event Logs

Day 3 không dừng ở đọc Event Viewer.

Syllabus yêu cầu có khả năng **extract audit-quality evidence**.

Cần phân biệt:

```text
View event
vs
Filter event
vs
Export event
vs
Preserve native evidence
```

Đặc biệt cần giữ:

- Event ID.
- Record ID.
- Provider.
- Host.
- Timestamp.

Syllabus yêu cầu export native **EVTX** bằng approved method như `wevtutil`, đồng thời tạo **CSV summary** riêng với encoding được ghi rõ. Infrastructure_Operations_14_Da…

---

# 28. Task Scheduler

Học:

- Task.
- Trigger.
- Action.
- Conditions.
- Settings.
- Run identity.
- Run whether user is logged on.
- History/status.
- Last Run Result.

Các failure phổ biến:

- wrong account;
- missing permission;
- incorrect working directory;
- script execution failure;
- environment difference;
- path with spaces;
- inaccessible network resource.

---

# 29. Backup and recovery

Day 3 cần foundation về:

- Backup purpose.
- Recovery.
- Restore verification.
- Recovery point awareness.
- System/application data distinction.

Quan trọng:

> Backup job “Success” không đồng nghĩa restore chắc chắn thành công.

Cần verify data sau recovery.

---

# 30. PowerShell fundamentals

PowerShell là phần trung tâm của Day 3.

Khác shell text-oriented truyền thống, PowerShell làm việc mạnh với **objects**.

Ví dụ:

```powershell
Get-Service
```

trả về object chứ không chỉ text.

Cần học:

- Cmdlets.
- Verb-Noun syntax.
- Parameters.
- Objects.
- Properties.
- Methods.
- Pipeline.

---

# 31. PowerShell pipelines

Ví dụ:

```powershell
Get-Service |
Where-Object Status -eq 'Running' |
Select-Object Name, Status
```

Sẽ học flow:

```text
Object
→ Pipeline
→ Filter
→ Select
→ Sort
→ Export
```

Đây là nền tảng quan trọng cho Day 12 automation.

---

# 32. PowerShell Remoting

Cần hiểu:

```powershell
Invoke-Command
Enter-PSSession
```

và dependency:

- WinRM.
- Authentication.
- Firewall.
- Authorization.
- Endpoint configuration.

Đồng thời phải phân biệt local command và remote command.

---

# 33. PowerShell execution policies

Học các execution policy concepts và operational implications.

Quan trọng:

**Execution Policy không phải security boundary hoàn chỉnh.**

Nó không thay thế:

- Code signing.
- Application control.
- Least privilege.
- Endpoint security.

Không nên thay execution policy rộng chỉ để chạy một script chưa kiểm chứng.

---

# 34. Administrative PowerShell scripting

Sẽ bắt đầu viết script nhỏ cho Infrastructure Operations:

- Health checks.
- Service checks.
- Network checks.
- Storage checks.
- Event extraction.
- Report generation.

Ví dụ logic:

```text
Collect hostname
Collect uptime
Collect disk capacity
Collect service status
Collect network status
Write structured output
Return useful exit status
```

---

# 35. Windows Server operational maintenance

Đây là phần tổng hợp.

Một maintenance workflow đúng phải gồm:

```text
Ticket/change
↓
Approval
↓
Scope
↓
Pre-check
↓
Baseline/evidence
↓
Backup/recovery readiness
↓
Implementation
↓
Verification
↓
Application validation
↓
Post-check
↓
Evidence
↓
Closure / Escalation
```

Các thao tác như service restart, patch/reboot, permission change hay firewall rule change không được nhìn như isolated technical command mà phải đặt trong operational workflow.

---

# 36. Security và least privilege xuyên suốt Day 3

Security không phải một topic riêng mà xuất hiện trong mọi phần:

- Không cấp local Administrator nếu không cần.
- Không disable firewall để workaround.
- Không dùng shared admin account nếu có thể tránh.
- Không export credential.
- Không sửa Registry thiếu backup.
- Không thay permission rộng hơn cần thiết.
- Không clear Event Logs trong quá trình investigation.
- Không bypass access control để lấy evidence.
- Remote management phải sử dụng authorized access.

---

# 37. Evidence-quality Windows administration

Đây là phần syllabus tăng cường đặc biệt.

Mọi evidence quan trọng cần liên hệ tới:

```text
Hostname
Timestamp/timezone
Windows version
Edition
Installation type
Command/filter
Event source
Ticket/change
Collector
Output file
Record count
Hash
```

Đối với event evidence, phải giữ được:

```text
Event ID
Record ID
Provider
Host
Timestamp
```

và khi có thể:

```text
Native EVTX
+
Human-readable CSV
```

Không được thay native evidence bằng screenshot nếu export được dữ liệu tốt hơn. Infrastructure_Operations_14_Da…

---

# Module 2 — THỰC HÀNH: Windows Server Administration Lab

Module thực hành bám trực tiếp Assignment/Lab và Additive Assignment/Lab của Day 3. Infrastructure_Operations_14_Da…

Toàn bộ thực hành mặc định trên **authorized lab/non-production Windows Server VM**.

---

## Lab 1 — Provision Windows Server VM

Bạn sẽ:

- Provision một Windows Server VM được phép sử dụng.
- Boot và login.
- Kiểm tra hostname.
- Kiểm tra resources.
- Xác nhận network connectivity.
- Xác nhận administrative access.

Evidence:

- VM configuration.
- Hostname.
- CPU/RAM.
- Network.
- Timestamp.

---

# Lab 2 — Identify Windows version, edition và installation mode

Bạn phải xác định chính xác:

```text
Windows Server version
Edition
Build
Installation type
```

Sử dụng cả GUI và PowerShell/system tools.

Ví dụ sẽ làm việc với:

```powershell
Get-ComputerInfo
```

và các nguồn system information khác.

Kết quả cuối:

```text
Server:
Version:
Edition:
Build:
Installation Mode:
```

---

# Lab 3 — Build Windows Server edition matrix

Tạo bảng:

| Release | Edition | Installation | Roles/features | Management | Support | Lab status |
| ------- | ------- | ------------ | -------------- | ---------- | ------- | ---------- |

Các entry phải phân loại:

```text
Hands-on
Demonstrated
Reference-only
```

Đây là deliverable bắt buộc của additive coverage. Infrastructure_Operations_14_Da…

---

# Lab 4 — Server Core vs Desktop Experience comparison

Sử dụng:

- second VM;
- hoặc instructor demonstration.

So sánh:

- Login experience.
- Local management.
- PowerShell usage.
- Server Manager availability.
- Remote administration.
- Resource footprint.
- Operational workflow.

---

# Lab 5 — Inspect installed Roles and Features

Kiểm tra:

- Installed roles.
- Installed features.
- Management tools.

PowerShell sẽ sử dụng các command như:

```powershell
Get-WindowsFeature
```

Bạn phải giải thích role nào đang present và tại sao.

---

# Lab 6 — Local user management

Thực hiện:

- Create controlled local user.
- Set appropriate account configuration.
- Verify account.
- Disable/enable khi cần.
- Remove sau lab.

Không sử dụng real credential.

Ví dụ user:

```text
lab_ops01
```

---

# Lab 7 — Local group management

Tạo hoặc sử dụng controlled group, ví dụ:

```text
Lab_File_Readers
```

Sau đó:

```text
User
→ Group
→ Resource permission
```

Verify membership bằng GUI và PowerShell.

---

# Lab 8 — NTFS permissions

Tạo folder:

```text
C:\Lab\Data
```

Thiết lập:

- Read.
- Modify.
- Inheritance.
- Group-based access.

Sau đó test:

```text
Allowed operation
Denied operation
```

Phải chứng minh cả hai.

---

# Lab 9 — Shared-folder permissions

Tạo SMB share.

Sau đó cấu hình:

```text
Share permission
+
NTFS permission
```

Test từ authorized client hoặc second session.

Mục tiêu là xác định đúng **effective access**.

---

# Lab 10 — Secure administrative access

Thực hành kiểm tra:

- Administrative membership.
- RDP permission.
- Authentication source.
- Least privilege.

Không cấp Administrator chỉ vì user cần login.

---

# Lab 11 — RDP validation

Kiểm tra toàn bộ chain:

```text
Network
→ Firewall
→ RDP
→ Account
→ Authorization
```

Evidence cần có:

- Connection success.
- User identity.
- Server identity.
- Timestamp.

---

# Lab 12 — Configure Windows Defender Firewall rule

Tạo một rule giới hạn phạm vi trong lab.

Sau đó:

```text
Before
→ connectivity denied

Apply rule
→ connectivity allowed

Rollback
→ original state restored
```

Phải phân biệt read-only inspection và configuration-changing command.

---

# Lab 13 — Configure/validate WinRM

Kiểm tra WinRM và remote management.

Sau đó test PowerShell Remoting khi environment cho phép.

Verify:

```text
Local hostname
vs
Remote hostname
```

để chứng minh command thực sự chạy trên remote server.

---

# Lab 14 — Inspect network configuration

Thu thập:

```text
Adapter
IP
Prefix
Gateway
DNS
Route
Listening ports
```

Bằng GUI và PowerShell.

Các cmdlets sẽ được dùng gồm:

```powershell
Get-NetAdapter
Get-NetIPAddress
Get-NetIPConfiguration
Get-NetRoute
Get-NetTCPConnection
```

---

# Lab 15 — Inspect Windows services

Chọn một safe lab service.

Kiểm tra:

- Name.
- Status.
- Startup Type.
- Dependencies.
- Service account.
- Related events.

---

# Lab 16 — Authorized service restart

Thực hiện một **authorized restart**.

Workflow:

```text
Pre-check
→ Evidence
→ Restart
→ Status verification
→ Functional verification
→ Event verification
```

Không chỉ ghi:

```powershell
Restart-Service
```

rồi coi như hoàn thành.

---

# Lab 17 — Inspect running processes

Dùng GUI và:

```powershell
Get-Process
```

Tìm:

- PID.
- CPU.
- Memory.
- Process name.

Sau đó liên hệ một service với process tương ứng nếu phù hợp.

---

# Lab 18 — Inspect scheduled tasks

Kiểm tra:

- Existing tasks.
- Trigger.
- Action.
- User context.
- Last Run Time.
- Last Run Result.

Phân biệt OS/vendor tasks với learner-created task.

---

# Lab 19 — Create scheduled health-check script

Viết một PowerShell health-check script.

Ví dụ kiểm tra:

```text
Hostname
Uptime
Disk free space
Service state
IP configuration
```

Sau đó dùng Task Scheduler chạy định kỳ.

Verify:

```text
Task executed
+
Output created
+
Exit/result inspected
```

---

# Lab 20 — Storage inspection

Inspect:

```powershell
Get-Disk
Get-Partition
Get-Volume
```

Xác định:

- Disk.
- Partition.
- Filesystem.
- Drive letter.
- Capacity.
- Free space.

Không thực hiện destructive disk operation nếu không nằm trong lab được chuẩn bị.

---

# Lab 21 — Windows Update inspection

Kiểm tra:

- Patch/update status.
- Installed update information.
- Reboot considerations.
- Patch baseline.

Không tự ý patch production-style system ngoài approved workflow.

---

# Lab 22 — Windows Event Viewer investigation

Tạo hoặc sử dụng known event.

Sau đó tìm bằng:

- Event Viewer.
- Filter.
- Event ID.
- Timestamp.
- Provider.

Phải xác nhận đúng event thay vì chỉ chụp màn hình log.

---

# Lab 23 — Get-WinEvent

Dùng PowerShell để query event logs.

Conceptual pattern:

```powershell
Get-WinEvent ...
```

Sẽ thực hành:

- Specific log.
- Specific time window.
- Filter.
- Event ID.
- Provider.
- Structured output.

Đây là kỹ năng quan trọng cho Day 10 và Day 12.

---

# Lab 24 — Export native EVTX

Syllabus yêu cầu export **authorized System/Application/Security event window** và giữ native event evidence.

Một tool điển hình:

```text
wevtutil
```

Output:

```text
*.evtx
```

Sau đó verify:

- File exists.
- Correct log source.
- Correct event/time range.
- File hash.
- Record/count information khi có thể.

---

# Lab 25 — Produce CSV event summary

Từ event query, tạo một **CSV summary** riêng.

Các field điển hình:

```text
TimeCreated
Id
RecordId
ProviderName
MachineName
LevelDisplayName
Message
```

Encoding phải được ghi rõ.

Native EVTX và CSV có mục đích khác nhau:

```text
EVTX
→ preservation/native evidence

CSV
→ analysis/reporting
```

---

# Lab 26 — Negative evidence extraction tests

Syllabus yêu cầu cố ý test các tình huống:

### Unavailable channel

Query một channel không tồn tại hoặc unavailable.

### Denied access

Dùng controlled scenario để quan sát access-denied behavior.

### Empty results

Query một khoảng thời gian/filter không có event.

Điểm cực kỳ quan trọng:

```text
No result
≠
Collection succeeded and nothing happened
```

Cần xác định query thực sự hợp lệ.

Không được clear logs để tạo test case. Infrastructure_Operations_14_Da…

---

# Lab 27 — Domain inspection

Nếu lab có Active Directory:

- Check domain membership.
- Identify domain.
- Inspect domain user.
- Inspect relevant groups.
- Inspect OU placement.

Nếu AD lab không có, phần này có thể là demonstration/reference theo syllabus.

---

# Lab 28 — Group Policy inspection

Inspect:

- Applied policies.
- Computer policy awareness.
- User policy awareness.
- Resultant policy.

Không sửa enterprise GPO ngoài authorized scope.

---

# Lab 29 — Simulated service failure

Tạo một **safe service failure**.

Ví dụ:

```text
Service intentionally stopped
```

Sau đó điều tra:

```text
Symptom
→ Service status
→ Dependencies
→ Event Log
→ Root cause
→ Authorized recovery
→ Verification
```

Evidence phải thể hiện cả before/after.

---

# Lab 30 — Simulated permission failure

Tạo controlled permission issue:

```text
User expected to access folder
but receives Access Denied
```

Phải điều tra:

```text
User identity
→ Group membership
→ Share permission
→ NTFS permission
→ Inheritance
→ Effective access
```

Sau đó sửa **smallest required permission**.

Không giải quyết bằng `Everyone: Full Control`.

---

# Lab 31 — Event correlation cho failure

Sau service hoặc permission failure, xác định related Windows Event records.

Thu thập:

```text
Event ID
Record ID
Provider
Hostname
Timestamp
Message
```

Sau đó nối thành timeline.

---

# Lab 32 — Backup/recovery exercise

Thực hiện controlled backup/recovery phù hợp environment.

Phải verify:

```text
Backup exists
→ Restore executed
→ Recovered data readable
→ Expected data matches
```

Không coi “backup completed successfully” là bằng chứng recovery.

---

# Lab 33 — PowerShell pipeline practice

Ví dụ workflow:

```text
Get objects
→ Filter
→ Select
→ Sort
→ Export
```

Áp dụng với:

- Services.
- Processes.
- Network.
- Events.
- Storage.

---

# Lab 34 — PowerShell remoting practice

Nếu environment hỗ trợ:

```text
Management workstation
      ↓
    WinRM
      ↓
Windows Server
      ↓
PowerShell command
```

Thu thập evidence chứng minh remote execution.

---

# Lab 35 — Maintenance checklist

Bạn sẽ tạo một Windows Server maintenance checklist hoàn chỉnh.

Ít nhất phải có:

```text
Asset identity
Windows version
Edition
Installation mode
Roles
Services
Processes
Network
Firewall
Storage
Updates
Scheduled Tasks
Event Logs
Remote management
Security
Backup/recovery
Domain/GPO
Evidence
Outstanding issues
Escalation
```

Đây là deliverable chính của Day 3. Infrastructure_Operations_14_Da…

---

# Lab 36 — Evidence package

Cuối lab phải có một evidence package có khả năng reviewer kiểm tra lại.

Ví dụ:

```text
Day3/
├── 01_System_Inventory/
├── 02_Edition_Matrix/
├── 03_Users_Groups/
├── 04_Permissions/
├── 05_Network/
├── 06_Firewall/
├── 07_Services/
├── 08_Storage/
├── 09_Patching/
├── 10_ScheduledTasks/
├── 11_EventLogs/
│   ├── Native_EVTX/
│   └── CSV/
├── 12_Failure_Test/
├── 13_Backup_Recovery/
├── 14_PowerShell/
└── 15_Maintenance_Checklist/
```

Evidence phải giữ các yếu tố mà syllabus yêu cầu như event IDs, record IDs, provider, host, timestamps và hashes. Infrastructure_Operations_14_Da…

---

# Hai module sẽ kết nối với nhau như thế nào?

Day 3 nên được học theo flow:

```text
MODULE 1 — LÝ THUYẾT
        │
        ├─ Windows architecture
        ├─ Editions / Core / Desktop
        ├─ Identity / AD / GPO
        ├─ NTFS / Share
        ├─ Services / Processes
        ├─ Network / Firewall / RDP / WinRM
        ├─ Storage
        ├─ Patch
        ├─ Event Logs
        ├─ Task Scheduler
        ├─ Backup
        └─ PowerShell
               │
               ▼
MODULE 2 — THỰC HÀNH
        │
        ├─ Provision VM
        ├─ Inventory
        ├─ Users/groups
        ├─ Permissions
        ├─ Remote access
        ├─ Firewall
        ├─ Network
        ├─ Services
        ├─ Storage
        ├─ Updates
        ├─ Scheduled script
        ├─ Event investigation
        ├─ EVTX/CSV extraction
        ├─ Failure injection
        ├─ Recovery
        └─ Evidence package
```

Sau khi hoàn thành cả hai module, mục tiêu của Day 3 là bạn có thể nhìn một Windows Server và thực hiện được vòng vận hành cơ bản:

```text
Identify
→ Baseline
→ Inspect
→ Administer
→ Secure
→ Change
→ Verify
→ Troubleshoot
→ Recover
→ Collect Evidence
→ Rollback / Escalate
```

Đây cũng chính là năng lực Windows Server mà syllabus yêu cầu ở traceability matrix: **release/edition coverage matrix, selected-edition operational lab, Core/GUI comparison và EVTX/PowerShell evidence**. Infrastructure_Operations_14_Da…

**Bước tiếp theo:** khi bạn nói **“Trình bày Module Lý thuyết Day 3”**, tôi sẽ bắt đầu Module 1 từ foundation và đi thật sâu từng phần theo format **What → Why → How → Where/When → Components → Security → Failure modes → Verification → Troubleshooting → Evidence → Escalation**, kèm PowerShell/GUI examples. Khi bạn nói **“Trình bày Module Thực hành Day 3”**, tôi sẽ biến Module 2 thành một hands-on lab hoàn chỉnh có **Objective → Topology → Prerequisites → Safety → từng bước → Commands → Expected Result → Verification → Failure Injection → Evidence → Cleanup/Rollback → Troubleshooting checkpoints**.
