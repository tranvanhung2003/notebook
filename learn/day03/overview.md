Theo syllabus của Project, **Day 3** là toàn bộ module **Windows Server Administration Foundations**. Mục tiêu của Day 3 không phải trở thành Windows Server expert, mà đạt mức **foundation → operational readiness**: hiểu cấu trúc Windows Server, thực hiện được các tác vụ administration phổ biến, biết verify bằng GUI/PowerShell/logs, xử lý các lỗi cơ bản trong phạm vi L1/L2, biết thu thập evidence và biết khi nào cần escalation. Infrastructure_Operations_14_Da…

Tôi sẽ chia toàn bộ Day 3 thành đúng **2 module lớn: Lý thuyết và Thực hành**, để sau đó chúng ta có thể đi cực kỳ chi tiết từng module.

# Day 3 — Module 1: Lý thuyết

## Windows Server Administration & Operations Foundations

Module này bao phủ toàn bộ phần **Concept/Lecture** của Day 3 trong syllabus. Infrastructure_Operations_14_Da…

### 1. Windows Server architecture

Trước tiên cần hiểu Windows Server vận hành ở mức tổng thể: mối quan hệ giữa **kernel, services, processes, drivers, Registry, filesystem, security subsystem, networking stack, roles/features và management interfaces**.

Mục tiêu không phải học sâu Windows internals, mà phải hiểu một Windows Server đang chạy gồm những thành phần nào, thành phần nào có thể gây lỗi service, authentication, storage hoặc networking, và khi troubleshooting nên quan sát ở đâu trước.

Đây cũng là nền tảng để hiểu các công cụ như **Server Manager, Event Viewer, Services, Task Manager, PowerShell và Windows Admin Center**.

---

### 2. Windows Server generations: supported vs legacy

Cần nhận biết các thế hệ Windows Server, phân biệt:

- Windows Server đang còn trong lifecycle hỗ trợ.
- Windows Server đã cũ hoặc thuộc nhóm legacy.
- Khác biệt về feature availability, security capabilities, management tools và patch support giữa các generation.
- Tại sao một Infrastructure Operations engineer không được giả định cùng một command/feature sẽ giống nhau trên mọi release.

Phần này thiên về **operational awareness**, không phải học thuộc mọi version.

Syllabus cũng nhấn mạnh rằng việc training bao phủ nhiều generation/edition, nhưng lab thực tế có thể chỉ sử dụng một số edition được licensed; các feature phụ thuộc edition/release phải được kiểm tra trên hệ thống thực tế. Infrastructure_Operations_14_Da…

---

### 3. Windows Server editions và licensing awareness

Cần hiểu mục đích và khác biệt tổng quát giữa:

**Standard**

Thường phù hợp workload vật lý hoặc môi trường virtualization không quá lớn.

**Datacenter**

Hướng tới infrastructure virtualization/private cloud quy mô lớn và có nhiều capability hơn tùy release.

**Datacenter: Azure Edition**

Edition có các capability gắn chặt hơn với Azure/hybrid scenarios tùy generation.

**Essentials / legacy awareness**

Hiểu Essentials từng phục vụ môi trường nhỏ và cần nhận thức rằng support/model licensing thay đổi theo generation.

Điểm quan trọng với Operations là:

> Không suy luận feature chỉ dựa trên tên “Windows Server”; phải xác định cả **version + edition + installation mode**.

---

### 4. Server Core vs Desktop Experience

Đây là một chủ đề quan trọng trong Day 3.

Cần hiểu hai installation approaches:

**Server Core**

GUI tối giản hoặc không có full Desktop GUI; footprint nhỏ hơn; thường được quản trị qua **PowerShell, remote management, Server Manager, Windows Admin Center**.

**Desktop Experience**

Có GUI đầy đủ hơn, phù hợp khi workload hoặc quy trình vận hành cần graphical management.

Cần hiểu operational impact:

- Management method.
- Attack surface.
- Patching footprint.
- Resource consumption.
- Local vs remote administration.
- Troubleshooting workflow.

---

### 5. Server Manager

Học vai trò của **Windows Server Manager** trong:

- Local Server management.
- Add/remove Roles and Features.
- Inspect server properties.
- Services.
- Events.
- Performance.
- Storage.
- Remote server management.

Điểm cần hiểu là Server Manager là một **management interface**, không phải bản thân service hay operating system component mà nó quản lý.

---

### 6. Windows Admin Center awareness

Day 3 chỉ yêu cầu mức **awareness**.

Cần biết Windows Admin Center là một web-based management interface cho Windows infrastructure và có thể hỗ trợ quản trị:

- Servers.
- Clusters.
- Services.
- Storage.
- Networking.
- Certificates.
- Events.
- Processes.
- PowerShell.

Không cần đi sâu deployment architecture ở Day 3.

---

### 7. Windows services

Đây là core operational topic.

Cần hiểu:

- Service là gì.
- Service name vs Display Name.
- Running/Stopped/Paused.
- Automatic.
- Automatic (Delayed Start).
- Manual.
- Disabled.
- Service dependencies.
- Log On account.
- Recovery configuration.

Phải biết một service failure có thể đến từ:

- Dependency failure.
- Permission.
- Incorrect service account.
- Missing executable.
- Network dependency.
- Port conflict.
- Configuration corruption.
- Resource exhaustion.

Và phải biết verify bằng GUI lẫn PowerShell.

---

### 8. Processes và startup configuration

Phân biệt rõ:

**Process** là một running instance của executable.

**Service** là một workload được quản lý bởi Service Control Manager.

Một service có thể chạy trong một process riêng hoặc chia sẻ process tùy trường hợp.

Cần hiểu:

- PID.
- CPU.
- Memory.
- Parent/child awareness.
- User context.
- Startup programs.
- Service startup configuration.

Điều quan trọng trong Operations là không kết luận:

> “Process đang tồn tại = application healthy.”

Service health cần được xác minh bằng nhiều layer.

---

### 9. Windows Registry fundamentals

Cần biết **Windows Registry** là centralized hierarchical configuration database.

Các hive thường cần nhận biết ở mức foundation gồm:

- `HKEY_LOCAL_MACHINE`
- `HKEY_CURRENT_USER`
- `HKEY_CLASSES_ROOT`
- `HKEY_USERS`
- `HKEY_CURRENT_CONFIG`

Phải hiểu Registry rất nhạy cảm.

Operational principle:

**Read → verify → backup/export → approved change → verify → rollback nếu cần.**

Không được coi Registry editing là troubleshooting step đầu tiên.

---

### 10. Local users và groups

Học administration của:

- Local user.
- Local group.
- Password/account status.
- Group membership.
- Built-in administrative groups.

Phải hiểu sự khác biệt cơ bản giữa:

**local account**

và

**domain account**.

Đồng thời áp dụng **least privilege** khi cấp quyền.

---

### 11. Local Security Policy

Làm quen với các security control cục bộ như:

- Account policies.
- User Rights Assignment.
- Security Options.
- Audit Policy awareness.

Điểm quan trọng là hiểu Local Security Policy có thể bị **Domain Group Policy** override khi server tham gia domain.

---

### 12. NTFS permissions

Một trong những phần quan trọng nhất.

Cần nắm:

- Full Control.
- Modify.
- Read & Execute.
- List Folder Contents.
- Read.
- Write.
- Advanced/Special permissions awareness.

Đồng thời hiểu:

- Explicit permission.
- Inherited permission.
- Allow.
- Deny.
- User membership.
- Group membership.

Cần tập suy luận **effective access** thay vì chỉ nhìn một ACL entry.

---

### 13. Share permissions

Khi folder được chia sẻ qua SMB, ngoài NTFS permission còn có **share permission**.

Phải hiểu quan hệ giữa:

**Share Permission + NTFS Permission**

để xác định effective access khi truy cập từ network.

Ví dụ một user có NTFS Modify nhưng share permission chỉ Read thì quyền thực tế qua share sẽ bị giới hạn.

---

### 14. Active Directory Domain Services basics

Day 3 không yêu cầu trở thành AD administrator, nhưng phải hiểu nền tảng **Active Directory Domain Services (AD DS)**.

Cần biết các khái niệm:

- Domain.
- Domain Controller.
- Domain account.
- Authentication.
- Authorization.
- Directory object.
- Computer account.
- Group.
- Organizational Unit.

Quan trọng nhất là phân biệt:

**local identity**

với

**domain identity**.

---

### 15. Domain accounts

Hiểu cách domain account được dùng để:

- Logon.
- Access resources.
- Join groups.
- Receive Group Policy.
- Authenticate across enterprise systems.

Operations engineer cần biết kiểm tra server đang:

- Workgroup.
- Hay domain-joined.

Và user đang sử dụng:

- Local account.
- Hay domain account.

---

### 16. Organizational Units — OU

Hiểu **Organizational Unit** là logical container trong Active Directory.

OU thường được sử dụng để:

- Organize objects.
- Delegate administration.
- Apply Group Policy.

Không được nhầm:

**OU ≠ security group**.

---

### 17. Group Policy awareness

Day 3 yêu cầu **Group Policy awareness**, chưa phải Group Policy engineering chuyên sâu.

Cần hiểu:

- GPO là gì.
- GPO có thể kiểm soát server configuration thế nào.
- Local Policy và Domain Policy khác nhau thế nào.
- Configuration có thể đến từ Group Policy thay vì local administrator.

Khi troubleshooting cần tránh sửa local configuration liên tục trong khi GPO đang tự động áp lại policy.

---

### 18. DNS Server role

Hiểu Windows Server có thể cung cấp **DNS role**.

Cần biết nền tảng:

- Name resolution.
- Forward lookup.
- Reverse lookup awareness.
- DNS records.
- DNS client configuration.
- Quan hệ đặc biệt quan trọng giữa DNS và Active Directory.

Day 3 chủ yếu yêu cầu role awareness và operational understanding.

---

### 19. DHCP Server role

Hiểu chức năng của **DHCP**:

- Dynamically assign IP configuration.
- Scope.
- Lease.
- Reservation awareness.
- Default gateway.
- DNS server assignment.

Phải phân biệt:

**DHCP cung cấp network configuration**

với

**DNS cung cấp name resolution**.

---

### 20. RDP

Học **Remote Desktop Protocol** ở góc nhìn secure administration.

Cần hiểu:

- RDP dùng để remote interactive session.
- Authentication.
- Authorization.
- Firewall dependency.
- Network connectivity dependency.
- Remote Desktop settings.
- Risks khi expose RDP không kiểm soát.

Không xem việc mở port rộng rãi là phương pháp troubleshooting hợp lệ.

---

### 21. WinRM

**Windows Remote Management (WinRM)** là nền tảng quan trọng cho remote administration và PowerShell remoting.

Cần hiểu:

- Remote management.
- Authentication.
- Listener awareness.
- Firewall dependency.
- Remote PowerShell.

RDP và WinRM phục vụ mục đích khác nhau.

---

### 22. Windows Defender Firewall

Cần học:

- Inbound rules.
- Outbound rules.
- Profiles.
- Port-based rules.
- Program/service rules.
- Allow/block logic.

Phải biết troubleshooting firewall theo cách:

1. Xác định source/destination.
2. Protocol.
3. Port.
4. Direction.
5. Profile.
6. Existing rule.
7. Application listener.

Không nên disable toàn bộ firewall chỉ để “test”.

---

### 23. Windows Server networking

Bao gồm operational understanding của:

- Network adapters.
- IPv4 configuration.
- IP address.
- Prefix/subnet.
- Default gateway.
- DNS server.
- Routing awareness.
- Connectivity verification.

PowerShell sẽ được sử dụng song song với GUI.

---

### 24. Storage và volume management

Cần hiểu:

- Physical/virtual disk awareness.
- Disk.
- Partition.
- Volume.
- Drive letter.
- Filesystem.
- Capacity.
- Free space.

Và các operational risks:

- Disk full.
- Wrong volume.
- Filesystem problem.
- Application consuming unexpected capacity.

Day 3 tập trung administration cơ bản, không phải storage architecture nâng cao.

---

### 25. Windows Update và patch control

Cần hiểu patching không đơn giản là “nhấn Update”.

Operations phải quan tâm:

- Approved updates.
- Change window.
- Service impact.
- Reboot requirement.
- Dependency.
- Backup/restore considerations.
- Post-patch verification.
- Rollback/escalation.

Trong enterprise, patching phải gắn với **change control**.

---

### 26. Event Viewer

**Event Viewer** là một trong các troubleshooting tools quan trọng nhất của Day 3.

Cần biết cách:

- Chọn log.
- Filter.
- Search.
- Identify timestamps.
- Identify Source/Provider.
- Event ID.
- Level.
- User/computer context.
- Export evidence.

Không nên chỉ nhìn dòng chữ màu đỏ rồi kết luận root cause.

---

### 27. Windows Event Logs

Các log nền tảng cần nhận biết gồm:

- Application.
- System.
- Security.

Ngoài ra Windows còn nhiều **Applications and Services Logs** tùy role/component.

Cần học cách correlate:

**symptom → timestamp → Event Log → Event ID/Provider → related events → probable cause**.

---

### 28. Task Scheduler

Hiểu Windows có thể thực thi jobs/scripts tự động bằng **Task Scheduler**.

Cần biết:

- Trigger.
- Action.
- Conditions.
- Settings.
- Run As account.
- Run whether user is logged on or not.
- Last Run Result.
- Execution history.

Đây sẽ được dùng trực tiếp trong lab health-check.

---

### 29. Backup và recovery

Day 3 yêu cầu foundation về backup/recovery.

Không chỉ hiểu “backup completed”, mà phải nhận thức:

- What was backed up?
- Where?
- When?
- Can it be restored?
- Who is authorized?
- What evidence chứng minh recovery thành công?

Chi tiết backup/DR enterprise sẽ xuất hiện mạnh hơn ở các Day sau.

---

### 30. PowerShell fundamentals

Đây là một trong các trọng tâm lớn nhất của Day 3.

Cần hiểu PowerShell khác Command Prompt ở chỗ pipeline của PowerShell truyền **objects**, không chỉ raw text.

Cần học:

- Cmdlet naming.
- `Verb-Noun`.
- Parameters.
- Help.
- Objects.
- Properties.
- Methods awareness.
- Filtering.
- Formatting.
- Pipeline.

Ví dụ các nhóm command sẽ liên quan đến:

- Services.
- Processes.
- Users/groups.
- Network.
- Storage.
- Firewall.
- Event Logs.
- Tasks.
- Windows features.

---

### 31. PowerShell pipelines

Cần hiểu data flow kiểu:

```powershell
Get-Something | Where-Object ... | Select-Object ...
```

và tại sao nó giúp Infrastructure Operations thực hiện:

- Filtering.
- Inventory.
- Verification.
- Evidence extraction.
- Automation.

---

### 32. PowerShell remoting

Cần nắm concept:

- Local execution.
- Remote execution.
- Interactive remote session.
- One-to-many administration awareness.
- WinRM dependency.

Phần này phải luôn đi cùng authorization và least privilege.

---

### 33. PowerShell execution policies

Hiểu mục đích của **Execution Policy**, các scope và tác động tới script execution.

Quan trọng:

Execution Policy **không phải security boundary hoàn chỉnh**.

Không được tùy tiện thay đổi policy toàn hệ thống chỉ để chạy một script không rõ nguồn gốc.

---

### 34. Administrative scripting

Từ các PowerShell fundamentals ở trên, cần hiểu cách dùng script cho operations:

- Health checks.
- Inventory collection.
- Service verification.
- Event extraction.
- Storage checks.
- Network checks.
- Repeatable administration.

Script phải có:

- Predictable output.
- Error handling awareness.
- Logging/evidence.
- Safe permissions.
- Reversible actions khi có configuration change.

---

### 35. Windows Server operational maintenance

Đây là phần kết nối tất cả nội dung của Module Lý thuyết.

Một Infrastructure Operations engineer phải nhìn server theo workflow:

**Inventory → Baseline → Health Check → Identify Change → Authorization → Execute → Verify → Evidence → Rollback/Escalate → Documentation.**

Day 3 vì vậy không đơn thuần là học Windows GUI hay PowerShell, mà là học cách **vận hành Windows Server an toàn trong enterprise environment**.

---

# Day 3 — Module 2: Thực hành

## Windows Server Administration & Maintenance Lab

Module này chuyển toàn bộ kiến thức ở trên thành các bài hands-on tương ứng với **Assignment/Lab của syllabus**. Infrastructure_Operations_14_Da…

### Lab 1 — Provision Windows Server VM

Bạn sẽ chuẩn bị một **approved Windows Server virtual machine** trong lab/non-production environment.

Sau khi provision phải xác nhận server thực sự usable trước khi thay đổi configuration.

Evidence tối thiểu sẽ gồm:

- VM identity.
- Hostname.
- Boot status.
- Network connectivity.
- Timestamp.
- Administrative access validation.

---

### Lab 2 — Windows Server inventory

Xác định chính xác:

**Windows Server version**

**Edition**

Ví dụ loại information như Standard hoặc Datacenter tùy lab.

**Installation mode**

- Server Core.
- Desktop Experience.

**Installed Roles/Features**

Mục đích là tạo **configuration baseline** trước khi thực hiện các thao tác khác.

---

### Lab 3 — Local user/group administration

Bạn sẽ:

- Tạo controlled local user.
- Tạo hoặc sử dụng local group.
- Thêm/xóa membership theo yêu cầu.
- Verify account.
- Verify group membership.
- Kiểm tra allowed/denied access.

Tất cả đều phải tuân thủ **least privilege**.

---

### Lab 4 — NTFS permission configuration

Tạo test folder/file và cấu hình permission có kiểm soát.

Ví dụ scenario:

```text
C:\LabData
```

Phải kiểm tra ít nhất:

- Administrator access.
- Authorized user access.
- Unauthorized user denied access.
- Inheritance.
- Effective permission.

Quan trọng là **test permission bằng actual identity**, không chỉ nhìn ACL.

---

### Lab 5 — Shared-folder permissions

Tạo controlled SMB share trong lab.

Sau đó kiểm tra interaction giữa:

**NTFS permission**

và

**Share permission**.

Bạn phải chứng minh được:

- User nào Read.
- User nào Modify.
- User nào bị denied.

---

### Lab 6 — Secure administrative access

Thiết lập hoặc kiểm tra administrative access qua các phương thức được cấp phép.

Có thể bao gồm:

- Local console.
- RDP.
- PowerShell/WinRM.

Phải verify authentication, authorization và firewall dependency.

Không expose administrative protocols ra network không cần thiết.

---

### Lab 7 — Windows Defender Firewall

Tạo hoặc kiểm tra một **approved firewall rule**.

Bạn phải thực hiện cả hai test:

**Allowed traffic**

và

**Denied traffic**.

Sau đó lưu evidence cho:

- Rule.
- Profile.
- Protocol.
- Port.
- Direction.
- Test result.

---

### Lab 8 — Remote management

Cấu hình hoặc kiểm tra remote administration bằng Windows-native tooling.

Trọng tâm:

- WinRM.
- PowerShell remoting.
- Authentication.
- Authorization.
- Firewall.
- Remote command verification.

RDP có thể được sử dụng khi lab yêu cầu interactive administration.

---

### Lab 9 — Services inspection

Kiểm tra Windows services bằng cả:

**GUI**

và

**PowerShell**.

Phải nhận biết:

- Status.
- Startup Type.
- Service account.
- Dependencies.
- Relevant events.

---

### Lab 10 — Authorized service restart

Chọn một **safe lab service** được phép restart.

Workflow phải là:

```text
Pre-check
→ Record current state
→ Authorized restart
→ Check service state
→ Check dependent functionality
→ Check Event Log
→ Collect evidence
```

Không được dùng destructive production-style experimentation.

---

### Lab 11 — Process inspection

Sử dụng Task Manager hoặc PowerShell để kiểm tra:

- Process.
- PID.
- CPU.
- Memory.
- User context.

Sau đó liên hệ process với service/application tương ứng khi phù hợp.

---

### Lab 12 — Scheduled Tasks inspection

Kiểm tra các scheduled tasks hiện có:

- Trigger.
- Action.
- Account.
- Status.
- Last Run Time.
- Last Run Result.

Sau đó sẽ tạo một task riêng cho health-check script.

---

### Lab 13 — Storage inspection

Kiểm tra:

- Disks.
- Partitions.
- Volumes.
- Drive letters.
- Filesystem.
- Used/free space.

Mục tiêu là biết phát hiện capacity issue trước khi application bị ảnh hưởng.

---

### Lab 14 — Network inspection

Kiểm tra server network configuration bằng GUI và PowerShell.

Bạn phải verify:

```text
NIC
→ IP address
→ subnet/prefix
→ default gateway
→ DNS
→ local TCP/IP stack
→ gateway/network connectivity
→ DNS resolution
→ application/service connectivity
```

Đây là workflow troubleshooting rất quan trọng.

---

### Lab 15 — Windows Update inspection

Kiểm tra update/patch status của server.

Lab tập trung vào **inspection và operational control**, không nhất thiết cài mọi update.

Phải biết xác định:

- Current status.
- Pending reboot awareness.
- Patch/change implications.
- Maintenance requirements.

---

### Lab 16 — Windows Event Logs

Dùng Event Viewer và PowerShell để:

- Inspect System events.
- Inspect Application events.
- Inspect relevant Security events.
- Filter theo time.
- Filter theo Event ID/Provider khi phù hợp.
- Export relevant records.

Đây sẽ là nguồn evidence chính cho phần failure simulation.

---

### Lab 17 — PowerShell administration

Thực hiện các tác vụ inspection bằng PowerShell thay vì chỉ GUI.

Mục tiêu là có khả năng lấy configuration dưới dạng structured objects và xây dựng các pipeline phục vụ:

- Inventory.
- Verification.
- Troubleshooting.
- Evidence collection.

---

### Lab 18 — Scheduled health-check script

Viết một PowerShell health-check script đơn giản để kiểm tra một số thành phần như:

```text
Hostname
OS information
Service status
Disk free space
Network configuration
Timestamp
```

Script sau đó được chạy bằng **Task Scheduler**.

Phải verify cả:

- Script output.
- Task status.
- Execution time.
- Last Run Result.

---

### Lab 19 — Active Directory / Group Policy inspection

Nếu lab có domain environment, bạn sẽ thực hiện **inspection**, không tự ý sửa Domain Controller hoặc GPO.

Mục tiêu có thể gồm:

- Verify domain membership.
- Identify current user/domain.
- Inspect relevant Group Policy.
- Hiểu một configuration đang đến từ local policy hay domain policy.

Syllabus ghi rõ phần này thực hiện **“where available”**, nên nếu lab standalone thì không giả định có AD environment. Infrastructure_Operations_14_Da…

---

### Lab 20 — Failure Simulation: Service failure

Tạo một **controlled service failure** trên lab workload.

Sau đó troubleshoot theo hướng:

```text
Observe symptom
→ Check service state
→ Check dependencies
→ Check process
→ Check Event Log
→ Check configuration
→ Restore service
→ Verify application
→ Collect evidence
```

Mục tiêu không phải chỉ “làm service chạy lại”, mà phải xác định được probable cause.

---

### Lab 21 — Failure Simulation: Permission failure

Tạo một permission scenario trong test folder/share khiến controlled user không truy cập được.

Sau đó điều tra:

- Identity.
- Local/domain account.
- Group membership.
- NTFS ACL.
- Inheritance.
- Share permission.
- Effective access.

Sau khi sửa phải verify lại bằng chính user bị ảnh hưởng.

---

### Lab 22 — Event evidence extraction

Từ các failure simulation ở trên, tìm và export relevant Event Logs.

Evidence phải cho thấy tối thiểu:

```text
Server
Timestamp
Event Log
Provider/Source
Event ID
Severity
Message/context
Related operation
```

Qua đó tập kỹ năng **evidence-driven troubleshooting**.

---

### Lab 23 — Backup/recovery awareness

Thực hiện hoặc inspect backup/recovery capability có sẵn trong lab.

Trọng tâm không chỉ là backup state mà còn là tư duy:

```text
Backup
→ Verify backup
→ Identify restore point
→ Controlled recovery
→ Verify restored data/service
→ Record evidence
```

Không thực hiện destructive restore trên dữ liệu ngoài lab.

---

### Lab 24 — Windows Server configuration & maintenance checklist

Đây là deliverable cuối cùng của Day 3.

Bạn cần tổng hợp kết quả thành checklist thể hiện ít nhất:

```text
Server identity
Version
Edition
Installation mode
Installed roles/features

Local users/groups
NTFS permissions
Share permissions
Administrative access

Firewall configuration
Remote management

Services
Processes
Scheduled tasks

Network
Storage
Updates

Event Logs
PowerShell verification

AD/Group Policy inspection nếu available

Failure simulation
Troubleshooting result

Evidence collected
Outstanding issue
Rollback status
Escalation requirement
```

Đây chính là artifact chứng minh bạn hoàn thành Day 3 theo hướng **enterprise operations**, thay vì chỉ thực hiện một số command rời rạc.

---

# Mối quan hệ giữa hai Module

Có thể hình dung toàn bộ Day 3 như sau:

```text
MODULE 1 — LÝ THUYẾT
Windows Server platform
        ↓
Identity & Permissions
        ↓
AD / Group Policy awareness
        ↓
Services & Processes
        ↓
Networking & Firewall
        ↓
Storage & Updates
        ↓
Logs & Scheduling
        ↓
PowerShell & Remoting
        ↓
Operational Maintenance

                │
                ▼

MODULE 2 — THỰC HÀNH
Provision VM
        ↓
Inventory / Baseline
        ↓
Users & Groups
        ↓
NTFS / Share Permissions
        ↓
Secure Remote Administration
        ↓
Firewall / Network
        ↓
Services / Processes
        ↓
Storage / Update Inspection
        ↓
Event Logs
        ↓
PowerShell Administration
        ↓
Scheduled Health Check
        ↓
AD/GPO Inspection
        ↓
Failure Injection
        ↓
Troubleshooting
        ↓
Recovery / Verification
        ↓
Evidence
        ↓
Maintenance Checklist
```

Đây cũng phù hợp với Program Summary của syllabus, trong đó **Day 3 = Windows Server editions, AD, services, events, security và PowerShell**. Infrastructure_Operations_14_Da…

## Kết quả cần đạt sau Day 3

Sau khi hoàn thành cả hai module, bạn phải có khả năng nhìn một Windows Server mới và tự trả lời được: **đây là server version/edition/mode nào, có những roles gì, identity và permissions đang hoạt động thế nào, service/process/network/storage/firewall/update có khỏe không, log nào chứng minh điều đó, PowerShell command nào dùng để verify, lỗi nằm ở layer nào, thay đổi nào L1/L2 được phép thực hiện, evidence nào cần giữ và khi nào phải rollback hoặc escalate**.

Đặc biệt, mọi lab của chương trình được hiểu là **authorized lab/non-production environment**, còn các thao tác production hoặc high-risk phải tuân thủ change authorization, backup/rollback planning và supervision phù hợp. Infrastructure_Operations_14_Da…

Khi bạn nói **“Module Lý thuyết Day 3”**, tôi sẽ bắt đầu triển khai toàn bộ Module 1 từ foundation đến operational troubleshooting, theo từng topic với **What → Why → How → Where/When → Security → Verification → Troubleshooting → Evidence → Escalation**, kèm PowerShell/GUI examples.
