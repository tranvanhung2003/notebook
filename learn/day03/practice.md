# Day 3 — Module Thực hành: Windows Server Administration Foundations

Module thực hành này chuyển toàn bộ kiến thức lý thuyết Day 3 thành một **end-to-end Windows Server operations lab**. Theo syllabus, bạn phải provision một Windows Server VM được phê duyệt; xác định version, edition, installation mode và installed roles; cấu hình local users/groups, NTFS/share permissions, secure administrative access, firewall và remote management; kiểm tra services, scheduled tasks, storage, networking, updates và Windows Event Logs bằng cả GUI lẫn PowerShell; thực hiện authorized service restart; tạo scheduled health-check; inspect domain/Group Policy nếu có; mô phỏng service hoặc permission failure; extract event records; và hoàn thành Windows Server configuration/maintenance checklist. Phần additive yêu cầu thêm **Server Core/Desktop Experience comparison, bounded System/Application/Security log extraction, native EVTX, CSV summary, negative tests cho unavailable channel/denied access/empty result**, đồng thời phải giữ Event ID, Record ID, Provider, host, timestamp và hash. Infrastructure_Operations_14_Da…

Toàn bộ lab mặc định chạy trong **authorized non-production lab**. Không áp dụng trực tiếp các configuration-changing command lên production nếu chưa có ticket, approval, maintenance window, backup/recovery plan và rollback.

---

# 1. Mục tiêu cuối cùng của Module Thực hành

Sau khi hoàn thành lab, bạn không chỉ cần chứng minh rằng mình biết chạy một số PowerShell commands. Bạn phải thực hiện được toàn bộ operational workflow:

```text
Provision
   ↓
Identify
   ↓
Baseline
   ↓
Configure
   ↓
Verify
   ↓
Operate
   ↓
Inject controlled failure
   ↓
Troubleshoot
   ↓
Recover
   ↓
Collect evidence
   ↓
Rollback / Cleanup
   ↓
Document / Escalate
```

Một submission tốt phải giúp reviewer trả lời được:

```text
Server nào được thao tác?
Version/edition/build nào?
Ai thực hiện?
Thời điểm nào?
Ticket/change nào?
Configuration ban đầu là gì?
Đã thay đổi gì?
Đã verify bằng cách nào?
Failure nào được tạo?
Root cause xác định ra sao?
Recovery có thành công không?
Evidence gốc ở đâu?
Hash là gì?
Còn gap hoặc unresolved issue nào?
```

Đây cũng phù hợp với evidence contract chung của chương trình: evidence cần gắn với source system, environment/asset, collector, timestamp/timezone, exact query/command, output format, record count, hash, ticket và các limitation/error; raw evidence phải được giữ nguyên, còn working/derived copies phải tách riêng. Infrastructure_Operations_14_Da…

---

# 2. Topology đề xuất

Một topology lý tưởng cho Day 3:

```text
                    ┌─────────────────────────┐
                    │ Management Workstation  │
                    │ Windows 10/11           │
                    │ PowerShell / RDP         │
                    └────────────┬────────────┘
                                 │
                      Management Network
                                 │
                    ┌────────────▼────────────┐
                    │ WIN-SRV-D3-01           │
                    │ Windows Server          │
                    │ Standard/Datacenter     │
                    │ Core hoặc Desktop       │
                    └────────────┬────────────┘
                                 │
             ┌───────────────────┴───────────────────┐
             │                                       │
   ┌─────────▼─────────┐                  ┌──────────▼─────────┐
   │ Optional DC/DNS   │                  │ Optional second VM │
   │ AD DS / GPO       │                  │ Server Core/Client │
   └───────────────────┘                  └────────────────────┘
```

Lab chính nên dùng **member server**, không dùng Domain Controller làm máy thử users/groups/local policy vì nhiều behavior về local account khác trên Domain Controller.

Server chính có thể là Windows Server 2022 hoặc 2025 tùy lab. Các cmdlet chính trong bài được chọn để hoạt động trên modern Windows Server, nhưng exact feature availability vẫn phải verify theo version/edition cụ thể.

---

# 3. Lab safety model

Trong toàn bộ Day 3, hãy phân biệt ba loại thao tác:

| Mức                    | Ý nghĩa                                        | Ví dụ                                                                 |
| ---------------------- | ---------------------------------------------- | --------------------------------------------------------------------- |
| Read-only              | Chỉ đọc state/configuration                    | `Get-Service`, `Get-NetIPConfiguration`, `Get-WinEvent`               |
| Controlled change      | Thay đổi nhưng reversible                      | tạo local group, firewall lab rule, Scheduled Task                    |
| Potentially disruptive | Có thể gây service interruption hoặc data loss | restart service, disk format, disable firewall, registry modification |

Các thao tác như `Format-Volume`, `Initialize-Disk`, disable Windows Firewall, clear Event Logs, registry mass change hoặc production reboot **không nằm trong normal Day 3 execution** trừ khi instructor cung cấp disposable environment và explicit approval.

Đặc biệt:

```text
wevtutil cl ...
```

là thao tác clear Event Log và **không được sử dụng** trong lab này.

---

# 4. Chuẩn bị evidence repository

Ngay đầu lab, tạo cấu trúc evidence:

```powershell
New-Item -ItemType Directory -Path C:\Day3Lab -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence\Inventory -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence\Identity -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence\Permissions -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence\Network -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence\Services -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence\Storage -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence\Tasks -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence\Events\Raw -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence\Events\Derived -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence\FailureTests -Force
```

Verify:

```powershell
Get-ChildItem C:\Day3Lab\Evidence
```

Expected result là tất cả folders tồn tại.

Bạn nên dùng naming convention:

```text
D3_<Hostname>_<Artifact>_<YYYYMMDD-HHMMSS>
```

Ví dụ:

```text
D3_WIN-SRV-D3-01_SystemEvents_20261020-143000.evtx
```

Timestamp nên ghi timezone trong manifest, ví dụ:

```text
2026-10-20 14:30:00 +07:00
```

---

# 5. Lab 1 — Provision và xác nhận Windows Server VM

## Objective

Có một Windows Server VM hoàn chỉnh, boot thành công, administrative access hoạt động và đủ điều kiện thực hiện Day 3.

## Pre-check

Trước khi bật VM, ghi nhận:

| Item                | Giá trị cần record            |
| ------------------- | ----------------------------- |
| VM name             | Ví dụ `WIN-SRV-D3-01`         |
| Environment         | `LAB`                         |
| Hypervisor          | Hyper-V/VMware/AHV/cloud...   |
| vCPU                | Theo build sheet              |
| RAM                 | Theo build sheet              |
| OS disk             | Theo build sheet              |
| Network             | Lab VLAN/network              |
| Snapshot/checkpoint | Nếu platform/runbook cho phép |
| Ticket              | `CHG-D3-001` hoặc fixture     |

Không tự ý dựa vào VM snapshot như một backup guarantee; snapshot chỉ được sử dụng theo platform runbook.

Sau khi boot:

```powershell
hostname
whoami
Get-Date
```

Expected:

```text
WIN-SRV-D3-01
WIN-SRV-D3-01\Administrator
10/20/2026 ...
```

Hoặc domain identity nếu lab đã join domain.

Tiếp tục:

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name, Manufacturer, Model,
                  NumberOfLogicalProcessors,
                  TotalPhysicalMemory
```

Evidence nên lưu:

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name, Manufacturer, Model,
                  NumberOfLogicalProcessors,
                  TotalPhysicalMemory |
    Export-Csv C:\Day3Lab\Evidence\Inventory\ComputerSystem.csv `
        -NoTypeInformation -Encoding UTF8
```

---

# 6. Lab 2 — Xác định version, edition, build và installation mode

Đây là requirement bắt buộc của syllabus.

## Method 1 — `Get-ComputerInfo`

```powershell
Get-ComputerInfo |
    Select-Object WindowsProductName,
                  WindowsVersion,
                  OsBuildNumber,
                  WindowsInstallationType
```

Expected output có dạng:

```text
WindowsProductName       : Windows Server ...
WindowsVersion           : ...
OsBuildNumber            : ...
WindowsInstallationType  : Server
```

hoặc Server Core tùy build.

## Method 2 — Registry read-only verification

```powershell
Get-ItemProperty `
    'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' |
    Select-Object ProductName,
                  EditionID,
                  DisplayVersion,
                  CurrentBuild,
                  UBR,
                  InstallationType
```

Điểm quan trọng ở đây không phải thuộc output, mà biết **cross-check hai sources**.

Lưu evidence:

```powershell
Get-ComputerInfo |
    Select-Object WindowsProductName,
                  WindowsVersion,
                  OsBuildNumber,
                  WindowsInstallationType |
    Export-Csv C:\Day3Lab\Evidence\Inventory\OS_Inventory.csv `
        -NoTypeInformation -Encoding UTF8
```

Sau đó hash:

```powershell
Get-FileHash `
    C:\Day3Lab\Evidence\Inventory\OS_Inventory.csv `
    -Algorithm SHA256
```

Hash không chứng minh source tuyệt đối đúng, nhưng cho phép detect modification sau khi baseline được recorded.

---

# 7. Lab 3 — Xây dựng edition/release matrix

Phần này không chỉ làm trên VM hiện tại. Bạn tạo một matrix từ documentation/instructor material, với lab hiện tại được đánh dấu `Hands-on`.

Ví dụ:

| Release     | Edition         | Install option    | Lab status         | Notes                     |
| ----------- | --------------- | ----------------- | ------------------ | ------------------------- |
| Server 2025 | Standard        | Core/Desktop      | Reference/Hands-on | theo lab                  |
| Server 2025 | Datacenter      | Core/Desktop      | Reference          | chưa deploy               |
| Server 2022 | Standard        | Core/Desktop      | Hands-on/Reference | tùy environment           |
| Server 2022 | Datacenter      | Core/Desktop      | Reference          | edition-specific features |
| Legacy      | Essentials/etc. | version dependent | Reference-only     | lifecycle awareness       |

Không ghi:

```text
"Tested all editions"
```

nếu thực tế chỉ có một VM.

Syllabus yêu cầu reviewer nhìn được entry nào là **hands-on, demonstrated hoặc reference-only**. Infrastructure_Operations_14_Da…

---

# 8. Lab 4 — Server Core vs Desktop Experience

Nếu có VM thứ hai, dùng một Core và một Desktop Experience machine.

Nếu không có, instructor demonstration cũng đáp ứng phần so sánh, nhưng phải ghi `D`/demonstrated thay vì hands-on.

Trên cả hai:

```powershell
Get-ComputerInfo |
    Select-Object WindowsProductName,
                  OsBuildNumber,
                  WindowsInstallationType
```

Trên Core, thử:

```powershell
sconfig
```

Quan sát management model.

Bảng so sánh evidence nên có:

| Aspect                   | Desktop Experience | Server Core     |
| ------------------------ | ------------------ | --------------- |
| Full GUI                 | Có                 | Không           |
| Local MMC experience     | Có                 | Hạn chế/remote  |
| PowerShell               | Có                 | Có              |
| SConfig                  | Có thể dùng        | Rất quan trọng  |
| Remote Server Manager    | Có                 | Có              |
| Windows Admin Center     | Có thể quản trị    | Có thể quản trị |
| Local resource footprint | Lớn hơn            | Nhỏ hơn         |
| Workload compatibility   | rộng               | cần kiểm tra    |

Không cố “convert” installation option trong lab.

---

# 9. Lab 5 — Inspect Roles and Features

Read-only:

```powershell
Get-WindowsFeature |
    Where-Object Installed -eq $true |
    Select-Object DisplayName, Name, InstallState
```

Lưu:

```powershell
Get-WindowsFeature |
    Where-Object Installed -eq $true |
    Select-Object DisplayName, Name, InstallState |
    Export-Csv C:\Day3Lab\Evidence\Inventory\InstalledFeatures.csv `
        -NoTypeInformation -Encoding UTF8
```

Bạn phải trả lời được:

```text
Server này đang có role gì?
Role đó có nằm trong intended build sheet không?
Có feature nào unexpected không?
```

Nếu thấy một role không thuộc expected baseline, **không remove ngay**. Ghi discrepancy và escalate/review build sheet.

---

# 10. Lab 6 — Local users

Đây là configuration-changing lab nhưng reversible.

Kiểm tra trước:

```powershell
Get-LocalUser
```

Nếu command không hoạt động vì server là Domain Controller, đó là dấu hiệu topology sai cho exercise này; chuyển sang member server.

Tạo user:

```powershell
$password = Read-Host "Enter disposable lab password" -AsSecureString

New-LocalUser `
    -Name "lab_ops01" `
    -Password $password `
    -Description "Day 3 controlled lab account"
```

Không hard-code:

```powershell
-Password (ConvertTo-SecureString "Password123!" -AsPlainText -Force)
```

trong evidence hoặc script thật.

Verify:

```powershell
Get-LocalUser -Name lab_ops01 |
    Select-Object Name, Enabled, Description, LastLogon
```

Disable/enable test:

```powershell
Disable-LocalUser -Name lab_ops01
Get-LocalUser -Name lab_ops01

Enable-LocalUser -Name lab_ops01
Get-LocalUser -Name lab_ops01
```

Expected: `Enabled` thay đổi phù hợp.

Rollback cuối lab:

```powershell
Remove-LocalUser -Name lab_ops01
```

Chỉ chạy sau khi các permission labs đã hoàn tất.

---

# 11. Lab 7 — Local groups và least privilege

Tạo hai groups:

```powershell
New-LocalGroup `
    -Name "Lab_File_Readers" `
    -Description "Day 3 read-only file access"

New-LocalGroup `
    -Name "Lab_File_Editors" `
    -Description "Day 3 modify file access"
```

Thêm user:

```powershell
Add-LocalGroupMember `
    -Group "Lab_File_Readers" `
    -Member "lab_ops01"
```

Verify:

```powershell
Get-LocalGroupMember -Group Lab_File_Readers
```

Evidence:

```powershell
Get-LocalGroupMember -Group Lab_File_Readers |
    Select-Object Name, ObjectClass, PrincipalSource |
    Export-Csv `
      C:\Day3Lab\Evidence\Identity\Lab_File_Readers.csv `
      -NoTypeInformation -Encoding UTF8
```

Operational lesson:

```text
User → Group → Permission
```

tốt hơn:

```text
User → Permission trực tiếp
```

vì dễ audit/revoke hơn.

---

# 12. Lab 8 — NTFS permissions

Tạo resource:

```powershell
New-Item `
    -ItemType Directory `
    -Path C:\Day3Lab\Data `
    -Force
```

Kiểm tra current ACL:

```powershell
Get-Acl C:\Day3Lab\Data | Format-List
```

Với `icacls`:

```cmd
icacls C:\Day3Lab\Data
```

Không nên phá inheritance ngay nếu syllabus không yêu cầu.

Cấp `Read & Execute` cho Reader group:

```cmd
icacls C:\Day3Lab\Data /grant "Lab_File_Readers:(OI)(CI)RX"
```

Cấp Modify cho Editors:

```cmd
icacls C:\Day3Lab\Data /grant "Lab_File_Editors:(OI)(CI)M"
```

Trong đó:

```text
OI = Object Inherit
CI = Container Inherit
RX = Read & Execute
M  = Modify
```

Verify:

```cmd
icacls C:\Day3Lab\Data
```

Tạo test file:

```powershell
"Day 3 permission test" |
    Set-Content C:\Day3Lab\Data\Test.txt
```

Khi test bằng `lab_ops01`, expected:

```text
Read file → Allowed
Modify/create file → Denied
```

nếu user chỉ thuộc Reader group.

Đây là phần cực kỳ quan trọng: bạn phải chứng minh **allowed case và denied case**.

---

# 13. Kiểm tra effective access

Nếu có Desktop Experience, dùng:

```text
Properties
→ Security
→ Advanced
→ Effective Access
```

Nhập user:

```text
WIN-SRV-D3-01\lab_ops01
```

So sánh với manual reasoning từ group memberships.

Nếu actual access khác expected:

```text
Check group membership
→ inheritance
→ explicit Deny
→ another group
→ token refresh
```

Nếu vừa thay group membership, user có thể cần logoff/logon để nhận token mới.

---

# 14. Lab 9 — SMB share

Tạo share:

```powershell
New-SmbShare `
    -Name "Day3Data" `
    -Path "C:\Day3Lab\Data" `
    -ReadAccess "$env:COMPUTERNAME\Lab_File_Readers" `
    -ChangeAccess "$env:COMPUTERNAME\Lab_File_Editors"
```

Verify:

```powershell
Get-SmbShare -Name Day3Data
```

```powershell
Get-SmbShareAccess -Name Day3Data
```

NTFS vẫn phải verify:

```cmd
icacls C:\Day3Lab\Data
```

Mental model:

```text
Client
  ↓
\\server\Day3Data
  ↓
Share permission
  ↓
NTFS permission
  ↓
File
```

Test:

```text
\\WIN-SRV-D3-01\Day3Data
```

với Reader user.

Expected:

```text
Read → success
Create/write → denied
```

Nếu NTFS cho Modify nhưng share chỉ Read thì access qua network vẫn bị hạn chế.

Rollback:

```powershell
Remove-SmbShare -Name Day3Data -Force
```

Lưu ý `Remove-SmbShare` chỉ remove share definition, không tự xóa directory/data.

---

# 15. Lab 10 — Kiểm tra administrative membership

Read-only:

```powershell
Get-LocalGroupMember -Group Administrators
```

Bạn cần identify:

```text
Local Administrator accounts
Domain admin/support groups nếu có
Unexpected memberships
```

Không remove unexpected account ngay trong training environment nếu chưa có approved instruction.

Ghi:

```text
Expected?
Business owner?
Approval?
Privilege justified?
```

---

# 16. Lab 11 — RDP baseline

Trước khi thay đổi RDP, kiểm tra state.

Registry read:

```powershell
Get-ItemProperty `
  'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' `
  -Name fDenyTSConnections
```

Interpretation thường:

```text
0 = RDP connection allowed by this setting
1 = denied by this setting
```

Nhưng vẫn còn dependency khác.

Kiểm tra service:

```powershell
Get-Service TermService
```

Firewall:

```powershell
Get-NetFirewallRule |
    Where-Object Service -eq 'TermService' |
    Select-Object DisplayName, Enabled, Direction, Action, Profile
```

Từ management workstation:

```powershell
Test-NetConnection WIN-SRV-D3-01 -Port 3389
```

Expected nếu RDP listener/firewall/network cho phép:

```text
TcpTestSucceeded : True
```

Điều này vẫn chưa chứng minh authentication thành công.

---

# 17. Authorized RDP enablement

Chỉ thực hiện nếu RDP đang disabled và lab yêu cầu bật.

Trước tiên capture original:

```powershell
$OriginalRdp = (Get-ItemProperty `
  'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' `
  -Name fDenyTSConnections).fDenyTSConnections
```

Enable:

```powershell
Set-ItemProperty `
  'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' `
  -Name fDenyTSConnections `
  -Value 0
```

Firewall rule names có thể khác theo localization. Trên English installation:

```powershell
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
```

Verify:

```powershell
Get-Service TermService
Test-NetConnection localhost -Port 3389
```

Sau đó remote-test từ workstation.

Rollback phải khôi phục **original state**, không đơn giản luôn disable.

Nếu `$OriginalRdp` là `1`, cuối lab:

```powershell
Set-ItemProperty `
  'HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server' `
  -Name fDenyTSConnections `
  -Value 1
```

---

# 18. Lab 12 — Windows Defender Firewall inventory

Read-only:

```powershell
Get-NetFirewallProfile |
    Select-Object Name, Enabled,
                  DefaultInboundAction,
                  DefaultOutboundAction
```

Lưu:

```powershell
Get-NetFirewallProfile |
    Select-Object Name, Enabled,
                  DefaultInboundAction,
                  DefaultOutboundAction |
    Export-Csv `
      C:\Day3Lab\Evidence\Network\FirewallProfiles.csv `
      -NoTypeInformation -Encoding UTF8
```

Không bao giờ giải quyết troubleshooting bằng:

```powershell
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled False
```

trong lab thông thường.

---

# 19. Lab 13 — Tạo narrow temporary firewall rule

Một failure-safe exercise là cho phép ICMP echo từ management workstation cụ thể.

Giả sử management IP:

```text
10.10.10.50
```

Tạo rule:

```powershell
New-NetFirewallRule `
    -DisplayName "DAY3-LAB-ICMP-ECHO" `
    -Direction Inbound `
    -Protocol ICMPv4 `
    -IcmpType 8 `
    -RemoteAddress 10.10.10.50 `
    -Action Allow
```

Verify:

```powershell
Get-NetFirewallRule `
    -DisplayName "DAY3-LAB-ICMP-ECHO"
```

Từ workstation:

```text
ping <server-ip>
```

Nếu ping vẫn fail, không kết luận firewall rule sai ngay. Có thể upstream network/filter không cho ICMP.

Rollback:

```powershell
Remove-NetFirewallRule `
    -DisplayName "DAY3-LAB-ICMP-ECHO"
```

Verify removal:

```powershell
Get-NetFirewallRule `
    -DisplayName "DAY3-LAB-ICMP-ECHO" `
    -ErrorAction SilentlyContinue
```

Expected: no result.

---

# 20. Lab 14 — Network inventory

Chạy:

```powershell
Get-NetAdapter |
    Select-Object Name, InterfaceDescription,
                  Status, LinkSpeed, MacAddress
```

```powershell
Get-NetIPConfiguration
```

```powershell
Get-NetIPAddress -AddressFamily IPv4 |
    Select-Object InterfaceAlias,
                  IPAddress,
                  PrefixLength,
                  AddressState
```

```powershell
Get-DnsClientServerAddress |
    Select-Object InterfaceAlias,
                  AddressFamily,
                  ServerAddresses
```

```powershell
Get-NetRoute -AddressFamily IPv4 |
    Sort-Object RouteMetric |
    Select-Object DestinationPrefix,
                  NextHop,
                  InterfaceAlias,
                  RouteMetric
```

Lưu toàn bộ baseline:

```powershell
Get-NetIPConfiguration |
    Format-List * |
    Out-File `
      C:\Day3Lab\Evidence\Network\IPConfiguration.txt `
      -Encoding utf8
```

---

# 21. Network verification path

Giả sử cần kiểm tra connection tới DNS server và application target.

Bắt đầu local stack:

```powershell
Test-NetConnection 127.0.0.1
```

Sau đó gateway:

```powershell
Test-NetConnection <default-gateway>
```

DNS:

```powershell
Resolve-DnsName <known-hostname>
```

Target TCP:

```powershell
Test-NetConnection <target> -Port 443
```

Phải phân biệt:

```text
Ping success
≠
TCP port success
≠
Application success
```

Nếu TCP 443 success, điều đó chỉ chứng minh transport connection có thể thiết lập, chưa chứng minh HTTP application healthy.

---

# 22. Lab 15 — WinRM baseline

Không thay đổi gì ngay.

```powershell
Get-Service WinRM
```

```powershell
Test-WSMan localhost
```

```cmd
winrm enumerate winrm/config/listener
```

```powershell
Get-PSSessionConfiguration
```

Record:

```text
Service status
Listener
Transport
Address
Firewall availability
PowerShell endpoints
```

Windows PowerShell remoting sử dụng WinRM trên Windows; Microsoft hiện vẫn document `Test-WSMan`, remoting requirements và các related mechanisms cho Windows Server. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-7.6&utm_source=chatgpt.com)

---

# 23. Enable PowerShell Remoting — chỉ khi cần

Nếu lab VM chưa configured và instructor cho phép:

```powershell
Enable-PSRemoting -Force
```

Đây là **configuration-changing command**. Nó có thể:

```text
Start WinRM
Set startup behavior
Create listeners/configuration
Modify firewall rules
Enable session configurations
```

nên phải capture state trước.

Sau đó:

```powershell
Test-WSMan localhost
```

Từ management workstation trong domain/trusted environment:

```powershell
Invoke-Command `
    -ComputerName WIN-SRV-D3-01 `
    -ScriptBlock {
        hostname
        whoami
        Get-Date
    }
```

Expected output phải chứng minh command chạy trên remote host.

Không đặt:

```text
TrustedHosts = *
```

như universal workaround trong workgroup environment.

Nếu authentication không phù hợp, dừng và dùng instructor-approved method, preferably domain authentication hoặc appropriately configured HTTPS.

---

# 24. Lab 16 — Services inventory

Read-only:

```powershell
Get-Service |
    Sort-Object Status, Name
```

Lưu:

```powershell
Get-Service |
    Select-Object Name, DisplayName, Status |
    Export-Csv `
      C:\Day3Lab\Evidence\Services\Services.csv `
      -NoTypeInformation -Encoding UTF8
```

Chọn **instructor-approved low-impact lab service**:

```text
<LAB_SERVICE>
```

Không tự chọn critical service như AD DS, DNS, cluster, database hay antivirus.

Query:

```powershell
Get-Service -Name <LAB_SERVICE>
```

Chi tiết:

```powershell
Get-CimInstance Win32_Service `
    -Filter "Name='<LAB_SERVICE>'" |
    Select-Object Name,
                  DisplayName,
                  State,
                  StartMode,
                  StartName,
                  ProcessId,
                  PathName
```

Điều này cho bạn:

```text
Service
→ service account
→ executable
→ PID
```

---

# 25. Lab 17 — Authorized service restart

Trước restart:

```powershell
Get-Service -Name <LAB_SERVICE>
```

```powershell
Get-CimInstance Win32_Service `
    -Filter "Name='<LAB_SERVICE>'" |
    Select-Object Name, State, ProcessId, StartMode
```

Record timestamp:

```powershell
Get-Date -Format o
```

Thực hiện:

```powershell
Restart-Service `
    -Name <LAB_SERVICE> `
    -ErrorAction Stop
```

Verify:

```powershell
Get-Service -Name <LAB_SERVICE>
```

Nếu service có port/function:

```powershell
Test-NetConnection localhost -Port <LAB_PORT>
```

hoặc application-specific functional check.

Sau đó Event Log:

```powershell
$start = (Get-Date).AddMinutes(-10)

Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    StartTime = $start
} |
Where-Object {
    $_.ProviderName -eq 'Service Control Manager'
} |
Select-Object TimeCreated, Id, RecordId,
              ProviderName, MachineName, Message
```

Không được coi `Running` là toàn bộ post-check nếu application layer vẫn broken.

---

# 26. Lab 18 — Processes

Read:

```powershell
Get-Process |
    Sort-Object CPU -Descending |
    Select-Object -First 20 `
        Name, Id, CPU, WorkingSet, StartTime
```

Một số system processes không cho đọc `StartTime` nếu permission không đủ; đó là expected access behavior.

Map service sang process:

```powershell
$svc = Get-CimInstance Win32_Service `
    -Filter "Name='<LAB_SERVICE>'"

$svc | Select Name, ProcessId

Get-Process -Id $svc.ProcessId
```

Bạn phải chứng minh understanding:

```text
Service object
→ ProcessId
→ process
```

Nếu PID = 0, service có thể stopped.

Không `Stop-Process -Force` chỉ vì process dùng nhiều CPU.

---

# 27. Lab 19 — Startup configuration

Kiểm tra service startup:

```powershell
Get-CimInstance Win32_Service |
    Select-Object Name, State, StartMode |
    Sort-Object StartMode, Name
```

Kiểm tra startup-related Scheduled Tasks:

```powershell
Get-ScheduledTask |
    Select-Object TaskPath, TaskName, State
```

Registry startup locations có thể inspect read-only:

```powershell
Get-ItemProperty `
  'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run' `
  -ErrorAction SilentlyContinue
```

Không thay đổi registry startup trong lab nếu không có explicit requirement.

---

# 28. Lab 20 — Task Scheduler inspection

Read:

```powershell
Get-ScheduledTask |
    Select-Object TaskPath, TaskName, State
```

Chọn task:

```powershell
Get-ScheduledTaskInfo `
    -TaskName "<TaskName>"
```

Record:

```text
TaskName
TaskPath
State
LastRunTime
LastTaskResult
NextRunTime
```

Đừng assume task success chỉ vì state là `Ready`.

---

# 29. Lab 21 — Viết PowerShell health-check script

Tạo:

```text
C:\Day3Lab\Scripts\Day3-HealthCheck.ps1
```

Tạo folder:

```powershell
New-Item `
    -ItemType Directory `
    -Path C:\Day3Lab\Scripts `
    -Force
```

Script mẫu:

```powershell
$ErrorActionPreference = 'Stop'

$OutputDir = 'C:\Day3Lab\Evidence\Tasks'
$Timestamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$HostName  = $env:COMPUTERNAME
$OutputFile = Join-Path $OutputDir `
    "HealthCheck_${HostName}_${Timestamp}.json"

try {
    $os = Get-CimInstance Win32_OperatingSystem

    $services = Get-Service |
        Where-Object {
            $_.Status -eq 'Running'
        } |
        Select-Object Name, Status

    $volumes = Get-Volume |
        Where-Object DriveLetter |
        Select-Object DriveLetter,
                      FileSystemLabel,
                      FileSystem,
                      Size,
                      SizeRemaining

    $network = Get-NetIPConfiguration |
        Select-Object InterfaceAlias,
                      IPv4Address,
                      IPv4DefaultGateway,
                      DNSServer

    $result = [pscustomobject]@{
        Hostname       = $HostName
        CollectedAt    = (Get-Date).ToString('o')
        LastBootTime   = $os.LastBootUpTime
        OS             = $os.Caption
        Version        = $os.Version
        RunningServiceCount = @($services).Count
        Volumes        = $volumes
        Network        = $network
        Status         = 'SUCCESS'
    }

    $result |
        ConvertTo-Json -Depth 6 |
        Set-Content -Path $OutputFile -Encoding UTF8

    Write-Output "Health check completed: $OutputFile"
    exit 0
}
catch {
    $errorFile = Join-Path $OutputDir `
        "HealthCheck_ERROR_${HostName}_${Timestamp}.log"

    "[{0}] {1}" -f (Get-Date).ToString('o'), $_.Exception.Message |
        Set-Content -Path $errorFile -Encoding UTF8

    Write-Error $_
    exit 1
}
```

Quan trọng ở script này là:

```text
timestamp
hostname
explicit output
error handling
non-zero exit on failure
no credentials
```

Test manual:

```powershell
C:\Day3Lab\Scripts\Day3-HealthCheck.ps1
```

Verify:

```powershell
Get-ChildItem C:\Day3Lab\Evidence\Tasks
```

Đọc JSON:

```powershell
Get-Content `
    (Get-ChildItem C:\Day3Lab\Evidence\Tasks\HealthCheck_*.json |
     Sort-Object LastWriteTime -Descending |
     Select-Object -First 1) |
    ConvertFrom-Json
```

---

# 30. Execution Policy checkpoint

Trước khi chạy script:

```powershell
Get-ExecutionPolicy -List
```

Nếu script bị block, **không set `Bypass` global ngay**.

Kiểm tra:

```text
File origin?
Policy scope?
GPO?
Signing requirement?
Lab runbook?
```

Nếu organization cung cấp approved signing/execution method, dùng method đó.

Day 3 không yêu cầu phá execution controls chỉ để task chạy.

---

# 31. Lab 22 — Register Scheduled Health Check

Microsoft hiện document `New-ScheduledTaskAction` và `Register-ScheduledTask` trên Windows Server, trong đó action xác định command mà Task Scheduler sẽ chạy và registration tạo task trên local machine. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/new-scheduledtaskaction?view=windowsserver2025-ps&utm_source=chatgpt.com)

Tạo action:

```powershell
$action = New-ScheduledTaskAction `
    -Execute "PowerShell.exe" `
    -Argument '-NoProfile -File "C:\Day3Lab\Scripts\Day3-HealthCheck.ps1"' `
    -WorkingDirectory "C:\Day3Lab\Scripts"
```

Working directory quan trọng. Nếu không chỉ định, task thường chạy với working directory khác, một nguồn lỗi phổ biến.

Trigger:

```powershell
$trigger = New-ScheduledTaskTrigger `
    -Once `
    -At (Get-Date).AddMinutes(2)
```

Principal cho disposable lab có thể dùng SYSTEM:

```powershell
$principal = New-ScheduledTaskPrincipal `
    -UserId "SYSTEM" `
    -LogonType ServiceAccount `
    -RunLevel Highest
```

Register:

```powershell
Register-ScheduledTask `
    -TaskName "Day3-HealthCheck" `
    -Action $action `
    -Trigger $trigger `
    -Principal $principal `
    -Description "Day 3 scheduled Windows health check"
```

Verify:

```powershell
Get-ScheduledTask `
    -TaskName Day3-HealthCheck
```

```powershell
Get-ScheduledTaskInfo `
    -TaskName Day3-HealthCheck
```

Sau thời gian trigger:

```powershell
Get-ChildItem C:\Day3Lab\Evidence\Tasks |
    Sort-Object LastWriteTime -Descending
```

Expected:

```text
Task LastRunTime populated
LastTaskResult indicates success
new JSON evidence exists
```

Nhưng phải verify JSON content, không chỉ exit code.

Rollback:

```powershell
Unregister-ScheduledTask `
    -TaskName Day3-HealthCheck `
    -Confirm:$false
```

---

# 32. Lab 23 — Storage inspection

Read-only:

```powershell
Get-Disk |
    Select-Object Number,
                  FriendlyName,
                  OperationalStatus,
                  HealthStatus,
                  PartitionStyle,
                  Size
```

```powershell
Get-Partition |
    Select-Object DiskNumber,
                  PartitionNumber,
                  DriveLetter,
                  Type,
                  Size
```

```powershell
Get-Volume |
    Select-Object DriveLetter,
                  FileSystemLabel,
                  FileSystem,
                  HealthStatus,
                  Size,
                  SizeRemaining
```

Lưu:

```powershell
Get-Volume |
    Select-Object DriveLetter,
                  FileSystemLabel,
                  FileSystem,
                  HealthStatus,
                  Size,
                  SizeRemaining |
    Export-Csv `
      C:\Day3Lab\Evidence\Storage\Volumes.csv `
      -NoTypeInformation -Encoding UTF8
```

Tính free percentage:

```powershell
Get-Volume |
Where-Object {
    $_.DriveLetter -and $_.Size -gt 0
} |
Select-Object DriveLetter,
    @{N='SizeGB';E={[math]::Round($_.Size/1GB,2)}},
    @{N='FreeGB';E={[math]::Round($_.SizeRemaining/1GB,2)}},
    @{N='FreePercent';E={
        [math]::Round(
            ($_.SizeRemaining/$_.Size)*100,
            2
        )
    }}
```

Không chạy:

```powershell
Initialize-Disk
Clear-Disk
Format-Volume
```

trên OS disk hoặc unknown disk.

Nếu instructor muốn volume-provisioning exercise, phải dùng riêng **disposable blank disk** với explicit disk-number validation.

---

# 33. Storage troubleshooting checkpoint

Giả sử `C:` còn 2% free.

Không delete random:

```text
C:\Windows
C:\ProgramData
application data
```

Workflow:

```text
Confirm free space
→ identify major consumers
→ determine data owner
→ check retention policy
→ approved cleanup
→ verify free space
→ monitor regrowth
```

Một quick inspection:

```powershell
Get-ChildItem C:\Day3Lab -Recurse -File |
    Sort-Object Length -Descending |
    Select-Object -First 20 `
      FullName,
      @{N='SizeMB';E={[math]::Round($_.Length/1MB,2)}}
```

Không scan toàn bộ production `C:\` recursively tùy tiện nếu workload lớn; query đó có thể expensive.

---

# 34. Lab 24 — Patch/update inventory

Một basic inventory:

```powershell
Get-HotFix |
    Sort-Object InstalledOn -Descending |
    Select-Object -First 20
```

Lưu:

```powershell
Get-HotFix |
    Sort-Object InstalledOn -Descending |
    Export-Csv `
      C:\Day3Lab\Evidence\Inventory\HotFixes.csv `
      -NoTypeInformation -Encoding UTF8
```

Nhưng cần hiểu `Get-HotFix` **không phải universal view của mọi servicing/package state**.

Có thể cross-check packages:

```powershell
Get-WindowsPackage -Online |
    Where-Object PackageState -eq 'Installed' |
    Select-Object PackageName,
                  PackageState,
                  ReleaseType,
                  InstallTime
```

Không cài patch chỉ vì thấy update.

---

# 35. Pending reboot awareness

Không có một single universal built-in flag bao phủ mọi product, nhưng Windows-specific indicators có thể inspect.

Ví dụ CBS:

```powershell
Test-Path `
 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Component Based Servicing\RebootPending'
```

Windows Update:

```powershell
Test-Path `
 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\WindowsUpdate\Auto Update\RebootRequired'
```

Pending file rename:

```powershell
Get-ItemProperty `
 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager' `
 -Name PendingFileRenameOperations `
 -ErrorAction SilentlyContinue
```

Interpretation:

```text
Indicator present
→ investigate reboot requirement

Indicator absent
≠ absolutely no product requires reboot
```

Application installers có thể có riêng reboot logic.

---

# 36. Lab 25 — Event Viewer manual inspection

Mở:

```text
Event Viewer
→ Windows Logs
```

Inspect:

```text
System
Application
Security
```

Chọn một System event và record:

```text
Log Name
Source/Provider
Event ID
Level
Logged timestamp
Computer
Record ID
Message
```

Sau đó tìm cùng event bằng PowerShell.

---

# 37. Lab 26 — `Get-WinEvent`

Microsoft khuyến nghị `FilterHashtable` như một cách filter hiệu quả với keys như `LogName`, `ID`, `StartTime` và `EndTime`. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-7.6&utm_source=chatgpt.com)

Define bounded window:

```powershell
$start = (Get-Date).AddHours(-1)
$end   = Get-Date
```

System:

```powershell
$systemEvents = Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    StartTime = $start
    EndTime   = $end
}
```

Display selected fields:

```powershell
$systemEvents |
    Select-Object TimeCreated,
                  Id,
                  RecordId,
                  ProviderName,
                  MachineName,
                  LevelDisplayName,
                  Message
```

Count:

```powershell
$systemEvents.Count
```

Export CSV:

```powershell
$systemEvents |
    Select-Object TimeCreated,
                  Id,
                  RecordId,
                  ProviderName,
                  MachineName,
                  LevelDisplayName,
                  Message |
    Export-Csv `
      C:\Day3Lab\Evidence\Events\Derived\System_LastHour.csv `
      -NoTypeInformation `
      -Encoding UTF8
```

Record:

```text
StartTime
EndTime
timezone
record count
hostname
collector
```

---

# 38. Security Log extraction

Security log thường cần elevated permissions.

```powershell
$securityEvents = Get-WinEvent -FilterHashtable @{
    LogName   = 'Security'
    StartTime = $start
    EndTime   = $end
} -ErrorAction Stop
```

Nếu Access Denied:

Không thay ACL hoặc take ownership.

Ghi:

```text
Collection status: FAILED
Reason: Access denied
Collector identity: ...
Timestamp: ...
```

Sau đó escalate/request authorized account.

Đây chính là một negative test hợp lệ.

---

# 39. Lab 27 — Native EVTX export

`wevtutil` trên Windows Server có thể query và export Event Logs; `epl` hỗ trợ `/q` XPath filtering. [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/wevtutil?utm_source=chatgpt.com)

Export toàn System log trong simple lab:

```cmd
wevtutil epl System C:\Day3Lab\Evidence\Events\Raw\System.evtx /ow:true
```

Nhưng syllabus yêu cầu bounded window, vì vậy tốt hơn là tạo query phù hợp.

Ví dụ Event ID scoped:

```cmd
wevtutil epl System C:\Day3Lab\Evidence\Events\Raw\ServiceEvents.evtx /q:"*[System[(EventID=7036)]]" /ow:true
```

Đối với time-bounded export, dùng XPath/structured query được xác định theo exact UTC interval của incident. Do syntax của timestamp query dễ sai, trong enterprise nên lưu query trong reviewable `.xml` file thay vì tự gõ ad hoc.

Một structured query conceptual:

```xml
<QueryList>
  <Query Id="0" Path="System">
    <Select Path="System">
      *[System[
        TimeCreated[
          @SystemTime&gt;='2026-10-20T07:00:00.000Z'
          and
          @SystemTime&lt;='2026-10-20T07:30:00.000Z'
        ]
      ]]
    </Select>
  </Query>
</QueryList>
```

Sau đó export dùng approved structured-query workflow.

Điểm quan trọng:

```text
Local ICT time
≠
UTC SystemTime string
```

Bạn phải convert đúng timezone.

---

# 40. Verify EVTX

Sau export:

```powershell
Get-Item `
 C:\Day3Lab\Evidence\Events\Raw\System.evtx
```

Hash:

```powershell
Get-FileHash `
 C:\Day3Lab\Evidence\Events\Raw\System.evtx `
 -Algorithm SHA256
```

Read back:

```powershell
Get-WinEvent `
 -Path C:\Day3Lab\Evidence\Events\Raw\System.evtx `
 -MaxEvents 10 |
Select-Object TimeCreated,
              Id,
              RecordId,
              ProviderName,
              MachineName
```

Đây là bước rất quan trọng.

Không chỉ verify:

```text
file exists
```

mà verify:

```text
file is readable
contains expected events
```

---

# 41. Lab 28 — CSV summary

CSV là derived evidence.

Ví dụ:

```powershell
Get-WinEvent `
  -Path C:\Day3Lab\Evidence\Events\Raw\System.evtx |
Select-Object TimeCreated,
              Id,
              RecordId,
              ProviderName,
              MachineName,
              LevelDisplayName,
              Message |
Export-Csv `
  C:\Day3Lab\Evidence\Events\Derived\System_Summary.csv `
  -NoTypeInformation `
  -Encoding UTF8
```

Hash riêng:

```powershell
Get-FileHash `
 C:\Day3Lab\Evidence\Events\Derived\System_Summary.csv `
 -Algorithm SHA256
```

Không dùng hash CSV thay cho EVTX hash.

Hai artifacts khác nhau:

```text
System.evtx
→ native evidence

System_Summary.csv
→ derived working/report artifact
```

---

# 42. Lab 29 — Negative event extraction tests

Syllabus yêu cầu ba loại test đặc biệt.

## Test A — Empty result

Chọn một impossible/reasonably absent Event ID trong bounded window:

```powershell
$result = Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    Id        = 999999
    StartTime = (Get-Date).AddMinutes(-5)
} -ErrorAction SilentlyContinue
```

Bạn có thể thấy no events hoặc exception tùy query.

Ghi:

```text
Query valid?
Channel readable?
Time range valid?
Result count = 0?
```

Conclusion đúng:

```text
No matching event found for this query.
```

Không viết:

```text
Nothing happened on server.
```

---

# 43. Negative Test B — unavailable/nonexistent channel

Ví dụ:

```powershell
Get-WinEvent `
    -LogName 'Day3-Definitely-Not-A-Real-Log' `
    -MaxEvents 1 `
    -ErrorAction Stop
```

Expected: error.

Ghi:

```text
Collection failed because channel does not exist/unavailable.
```

Đây khác hoàn toàn với zero-event result.

---

# 44. Negative Test C — denied access

Cách tốt nhất là instructor cung cấp account ít quyền hoặc fixture.

Chạy:

```powershell
Get-WinEvent `
  -LogName Security `
  -MaxEvents 10 `
  -ErrorAction Stop
```

nếu account không được phép đọc Security log.

Expected:

```text
Access denied
```

Không elevate bằng credential ngoài scope.

Evidence:

```text
identity
command
timestamp
error
```

---

# 45. Lab 30 — Domain membership inspection

Nếu environment có domain:

```powershell
Get-CimInstance Win32_ComputerSystem |
    Select-Object Name,
                  PartOfDomain,
                  Domain,
                  Workgroup
```

Expected:

```text
PartOfDomain : True
Domain       : corp.example
```

Identity:

```cmd
whoami
```

```cmd
whoami /user
```

```cmd
whoami /groups
```

Không assume account đang dùng là domain account.

---

# 46. OU/GPO inspection

Nếu RSAT/AD tooling available và được phép:

```powershell
Get-Command Get-ADComputer `
    -ErrorAction SilentlyContinue
```

Nếu AD module có:

```powershell
Get-ADComputer `
    -Identity $env:COMPUTERNAME `
    -Properties DistinguishedName |
    Select-Object Name, DistinguishedName
```

Nếu module không có, không install chỉ để complete lab nếu không được approve. Instructor có thể cung cấp demonstration.

GPO effective result:

```cmd
gpresult /r
```

Report:

```cmd
gpresult /h C:\Day3Lab\Evidence\Inventory\GPResult.html /f
```

Bạn cần identify:

```text
Computer GPOs
User GPOs
Denied/filter states nếu visible
```

Không modify GPO trong Day 3.

---

# 47. Lab 31 — Backup/recovery awareness

Day 3 cần thực hành recovery thinking, nhưng exact backup product phụ thuộc environment.

Nếu Windows Server Backup feature đã có:

```cmd
wbadmin get status
```

```cmd
wbadmin get versions
```

Read-only.

Nếu lab không có real backup solution, dùng một **controlled recovery-verification exercise**, không gọi nó là enterprise backup.

Tạo source:

```powershell
New-Item `
    -ItemType Directory `
    -Path C:\Day3Lab\RecoverySource `
    -Force

"Important Day 3 data" |
    Set-Content `
      C:\Day3Lab\RecoverySource\Important.txt
```

Hash source:

```powershell
Get-FileHash `
    C:\Day3Lab\RecoverySource\Important.txt `
    -Algorithm SHA256
```

Copy tới approved backup fixture/location:

```powershell
Copy-Item `
    C:\Day3Lab\RecoverySource\Important.txt `
    C:\Day3Lab\BackupFixture\Important.txt
```

Delete **working source only after recording hash**:

```powershell
Remove-Item `
    C:\Day3Lab\RecoverySource\Important.txt
```

Restore:

```powershell
Copy-Item `
    C:\Day3Lab\BackupFixture\Important.txt `
    C:\Day3Lab\RecoverySource\Important.txt
```

Hash restored:

```powershell
Get-FileHash `
    C:\Day3Lab\RecoverySource\Important.txt `
    -Algorithm SHA256
```

Expected:

```text
Original hash = Restored hash
```

Cần ghi rõ:

> Đây là restore-integrity exercise, không phải bằng chứng enterprise backup/DR solution hoàn chỉnh.

---

# 48. Failure Injection 1 — Service stopped

Đây là controlled safe failure.

Chọn instructor-approved:

```text
<LAB_SERVICE>
```

Baseline:

```powershell
Get-Service <LAB_SERVICE>
```

Record timestamp.

Stop:

```powershell
Stop-Service `
    -Name <LAB_SERVICE> `
    -ErrorAction Stop
```

Đây là configuration/state-changing action.

Sau đó giả lập ticket:

```text
Symptom:
Application function unavailable.
```

Bạn không được “biết trước” root cause khi viết troubleshooting report; phải điều tra theo process.

---

# 49. Service failure troubleshooting

Step 1:

```powershell
Get-Service <LAB_SERVICE>
```

Expected:

```text
Stopped
```

Step 2:

```powershell
Get-CimInstance Win32_Service `
  -Filter "Name='<LAB_SERVICE>'" |
  Select Name,
         State,
         StartMode,
         StartName,
         ProcessId
```

Step 3 — Event Logs:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    StartTime = (Get-Date).AddMinutes(-15)
} |
Where-Object ProviderName -eq 'Service Control Manager' |
Select TimeCreated,
       Id,
       RecordId,
       ProviderName,
       Message
```

Step 4 — dependency:

```powershell
Get-Service <LAB_SERVICE> -DependentServices
```

```powershell
Get-Service <LAB_SERVICE> -RequiredServices
```

Step 5 — approved recovery:

```powershell
Start-Service <LAB_SERVICE>
```

Step 6 — verify:

```powershell
Get-Service <LAB_SERVICE>
```

Step 7 — functional test nếu có.

Evidence phải thể hiện:

```text
Before failure
Failure introduced
Observed symptom
Diagnostic evidence
Root cause
Recovery command
Post-state
```

---

# 50. Failure Injection 2 — NTFS permission issue

Baseline:

```cmd
icacls C:\Day3Lab\Data
```

Save ACL:

```cmd
icacls C:\Day3Lab\Data /save C:\Day3Lab\Evidence\Permissions\Data_ACL_Backup.txt /t
```

Lưu ý `icacls /save` có restoration semantics riêng và cần test trong lab; không coi file text này là universal backup của mọi filesystem metadata.

Controlled change:

Remove Reader group access:

```cmd
icacls C:\Day3Lab\Data /remove "Lab_File_Readers"
```

Test user.

Expected:

```text
Access denied
```

Troubleshooting:

```powershell
Get-LocalGroupMember Lab_File_Readers
```

```cmd
icacls C:\Day3Lab\Data
```

Nếu shared access:

```powershell
Get-SmbShareAccess Day3Data
```

Root cause:

```text
NTFS ACE removed
```

Corrective action:

```cmd
icacls C:\Day3Lab\Data /grant "Lab_File_Readers:(OI)(CI)RX"
```

Retest:

```text
Read succeeds
Write still denied
```

Đây là key point: recovery phải restore **least privilege**, không mở rộng thành Modify/Full Control.

---

# 51. Failure Injection 3 — Firewall rule failure

Chỉ thực hiện nếu lab topology cho phép.

Baseline remote test:

```powershell
Test-NetConnection WIN-SRV-D3-01 -Port 3389
```

Instructor có thể disable **specific lab firewall rule**, không disable firewall profile.

Ví dụ nếu dùng custom test listener/service:

```powershell
Disable-NetFirewallRule `
    -DisplayName "<DAY3_TEST_RULE>"
```

Expected:

```text
Remote port test fails
```

Troubleshooting:

```text
Is service listening?
Is route valid?
Is DNS valid?
Is local firewall blocking?
```

Verify listener:

```powershell
Get-NetTCPConnection `
    -State Listen |
    Where-Object LocalPort -eq <PORT>
```

Re-enable exact rule:

```powershell
Enable-NetFirewallRule `
    -DisplayName "<DAY3_TEST_RULE>"
```

Expected connectivity returns.

Không disable entire Windows Firewall.

---

# 52. Failure Injection 4 — Scheduled Task failure

Safe example: temporarily set wrong script path.

Record current action:

```powershell
(Get-ScheduledTask Day3-HealthCheck).Actions
```

Create broken action in a separate test task:

```powershell
$badAction = New-ScheduledTaskAction `
    -Execute "PowerShell.exe" `
    -Argument '-NoProfile -File "C:\Day3Lab\Scripts\DoesNotExist.ps1"'

$badTrigger = New-ScheduledTaskTrigger `
    -Once `
    -At (Get-Date).AddMinutes(1)

$principal = New-ScheduledTaskPrincipal `
    -UserId "SYSTEM" `
    -LogonType ServiceAccount `
    -RunLevel Highest

Register-ScheduledTask `
    -TaskName "Day3-BrokenTask" `
    -Action $badAction `
    -Trigger $badTrigger `
    -Principal $principal
```

Sau run:

```powershell
Get-ScheduledTaskInfo Day3-BrokenTask
```

Inspect Task Scheduler Operational log nếu enabled:

```powershell
Get-WinEvent `
  -LogName 'Microsoft-Windows-TaskScheduler/Operational' `
  -MaxEvents 30 |
Select TimeCreated, Id, RecordId, Message
```

Root cause:

```text
Script path does not exist.
```

Cleanup:

```powershell
Unregister-ScheduledTask `
    -TaskName Day3-BrokenTask `
    -Confirm:$false
```

---

# 53. Failure Injection 5 — Empty/unavailable log query

Đây là evidence-handling failure injection chứ không phải system failure.

Bạn phải chứng minh mình phân biệt:

```text
No matches
```

với:

```text
Cannot query source
```

Đây là kỹ năng rất quan trọng cho Day 10 centralized logging sau này.

---

# 54. Troubleshooting checkpoint — RDP

Nếu user nói:

```text
"Cannot RDP"
```

trình tự nên là:

```text
Name resolution
    ↓
IP reachability / route
    ↓
TCP 3389 connectivity
    ↓
Firewall
    ↓
TermService/listener
    ↓
User identity
    ↓
Remote Desktop authorization
    ↓
GPO/User Rights
    ↓
Authentication/NLA/domain
```

Useful commands:

```powershell
Resolve-DnsName WIN-SRV-D3-01
```

```powershell
Test-NetConnection WIN-SRV-D3-01 -Port 3389
```

Server:

```powershell
Get-Service TermService
```

```powershell
Get-NetTCPConnection `
    -LocalPort 3389 `
    -State Listen `
    -ErrorAction SilentlyContinue
```

Identity:

```cmd
whoami
whoami /groups
```

Do not jump straight to:

```text
Add user to Administrators
```

---

# 55. Troubleshooting checkpoint — Access Denied

Use this sequence:

```text
Exact identity
   ↓
Local vs domain?
   ↓
Group memberships
   ↓
Share permission
   ↓
NTFS permission
   ↓
Inheritance
   ↓
Explicit Deny
   ↓
User token refresh
   ↓
Application behavior
```

Commands:

```cmd
whoami
whoami /groups
```

```powershell
Get-SmbShareAccess Day3Data
```

```cmd
icacls C:\Day3Lab\Data
```

Do not use:

```text
Everyone Full Control
```

as troubleshooting shortcut.

---

# 56. Troubleshooting checkpoint — service won't start

Read state:

```powershell
Get-Service <LAB_SERVICE>
```

Inspect CIM:

```powershell
Get-CimInstance Win32_Service `
 -Filter "Name='<LAB_SERVICE>'"
```

Check dependencies:

```powershell
Get-Service <LAB_SERVICE> -RequiredServices
```

Check System Event Log:

```powershell
Get-WinEvent -FilterHashtable @{
  LogName='System'
  StartTime=(Get-Date).AddMinutes(-30)
}
```

Check disk:

```powershell
Get-Volume
```

Check executable path and service account.

Do not repeatedly `Start-Service` without reading error/evidence.

---

# 57. Troubleshooting checkpoint — PowerShell Remoting

Management client:

```powershell
Test-WSMan WIN-SRV-D3-01
```

Server:

```powershell
Get-Service WinRM
```

```cmd
winrm enumerate winrm/config/listener
```

Firewall:

```powershell
Get-NetFirewallRule |
    Where-Object {
        $_.DisplayGroup -like '*Remote Management*' -or
        $_.Service -eq 'WinRM'
    }
```

Identity/auth:

```cmd
whoami
```

Do not add wildcard TrustedHosts unless approved.

---

# 58. Evidence manifest

Tạo CSV hoặc spreadsheet với columns:

```text
EvidenceID
Hostname
Environment
Source
Collector
CollectedAt
Timezone
Ticket
Command
TimeWindow
Filter
Filename
Format
Encoding
RecordCount
SHA256
ParentEvidenceID
Notes
```

Ví dụ:

| EvidenceID | Source            | File               | Count | SHA256 |
| ---------- | ----------------- | ------------------ | ----: | ------ |
| D3-E001    | Get-ComputerInfo  | OS_Inventory.csv   |     1 | ...    |
| D3-E002    | System Log        | System.evtx        |   432 | ...    |
| D3-E003    | Derived from E002 | System_Summary.csv |   432 | ...    |
| D3-E004    | `Get-Service`     | Services.csv       |   210 | ...    |

Đây là evidence mindset xuyên suốt 14 days. Infrastructure_Operations_14_Da…

---

# 59. Hash toàn bộ submitted evidence

Ví dụ:

```powershell
Get-ChildItem `
    C:\Day3Lab\Evidence `
    -Recurse `
    -File |
ForEach-Object {
    $hash = Get-FileHash $_.FullName -Algorithm SHA256

    [pscustomobject]@{
        File       = $_.FullName
        SizeBytes  = $_.Length
        SHA256     = $hash.Hash
    }
} |
Export-Csv `
  C:\Day3Lab\Evidence\Evidence_Hashes.csv `
  -NoTypeInformation `
  -Encoding UTF8
```

Quan trọng: chạy hash manifest sau khi evidence artifacts không còn được chỉnh sửa.

Nếu chỉnh sửa derived CSV sau đó, phải regenerate hash và record version.

Không modify raw EVTX sau hash.

---

# 60. Final Windows Server maintenance inspection

Cuối lab, chạy một consolidated read-only check.

## Identity

```powershell
hostname
whoami
```

## OS

```powershell
Get-ComputerInfo |
 Select WindowsProductName,
        OsBuildNumber,
        WindowsInstallationType
```

## Roles

```powershell
Get-WindowsFeature |
 Where Installed
```

## Services

```powershell
Get-Service |
 Where Status -ne Running
```

Không phải mọi stopped service là lỗi; cần baseline.

## Network

```powershell
Get-NetIPConfiguration
```

## Firewall

```powershell
Get-NetFirewallProfile
```

## Storage

```powershell
Get-Volume
```

## Scheduled Tasks

```powershell
Get-ScheduledTask
```

## Updates

```powershell
Get-HotFix |
 Sort InstalledOn -Descending |
 Select -First 10
```

## Recent errors

```powershell
Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    Level     = 2
    StartTime = (Get-Date).AddHours(-2)
} -ErrorAction SilentlyContinue
```

`Level=2` thường tương ứng Error trong Windows event level model.

---

# 61. Final maintenance checklist

Submission nên có bảng như sau:

| Category       | Check                       | Expected                 | Actual | Status | Evidence |
| -------------- | --------------------------- | ------------------------ | ------ | ------ | -------- |
| Identity       | Hostname correct            | Build sheet              | ...    | Pass   | E001     |
| OS             | Version/edition correct     | Approved                 | ...    | Pass   | E002     |
| Roles          | Expected roles only         | Baseline                 | ...    | Pass   | E003     |
| Identity       | Lab users/groups            | Least privilege          | ...    | Pass   | E004     |
| NTFS           | Reader read-only            | Read/no write            | ...    | Pass   | E005     |
| SMB            | Share access correct        | Expected                 | ...    | Pass   | E006     |
| RDP            | Authorized only             | Working                  | ...    | Pass   | E007     |
| WinRM          | Approved management         | Working                  | ...    | Pass   | E008     |
| Firewall       | Enabled                     | Yes                      | ...    | Pass   | E009     |
| Network        | IP/DNS/routes valid         | Baseline                 | ...    | Pass   | E010     |
| Services       | Lab service healthy         | Running                  | ...    | Pass   | E011     |
| Storage        | Adequate free space         | Threshold                | ...    | Pass   | E012     |
| Patching       | Status captured             | Yes                      | ...    | Pass   | E013     |
| Scheduled Task | Health task valid           | Exit 0/output            | ...    | Pass   | E014     |
| Events         | EVTX collected              | Valid/readable           | ...    | Pass   | E015     |
| CSV            | Derived summary             | UTF-8                    | ...    | Pass   | E016     |
| Hash           | SHA-256 recorded            | Yes                      | ...    | Pass   | E017     |
| Recovery       | Controlled restore verified | Hash match               | ...    | Pass   | E018     |
| Failure test   | Service recovery            | Successful               | ...    | Pass   | E019     |
| Failure test   | Permission recovery         | Least privilege restored | ...    | Pass   | E020     |

---

# 62. Cleanup / rollback

Cleanup không có nghĩa xóa evidence.

Giữ evidence repository, nhưng remove temporary configuration.

Remove lab share nếu không cần:

```powershell
Remove-SmbShare `
    -Name Day3Data `
    -Force `
    -ErrorAction SilentlyContinue
```

Remove firewall rule:

```powershell
Remove-NetFirewallRule `
    -DisplayName "DAY3-LAB-ICMP-ECHO" `
    -ErrorAction SilentlyContinue
```

Remove Scheduled Tasks:

```powershell
Unregister-ScheduledTask `
    -TaskName Day3-HealthCheck `
    -Confirm:$false `
    -ErrorAction SilentlyContinue
```

```powershell
Unregister-ScheduledTask `
    -TaskName Day3-BrokenTask `
    -Confirm:$false `
    -ErrorAction SilentlyContinue
```

Remove users/groups sau permission tests:

```powershell
Remove-LocalUser `
    -Name lab_ops01 `
    -ErrorAction SilentlyContinue

Remove-LocalGroup `
    -Name Lab_File_Readers `
    -ErrorAction SilentlyContinue

Remove-LocalGroup `
    -Name Lab_File_Editors `
    -ErrorAction SilentlyContinue
```

RDP/WinRM chỉ rollback nếu bạn đã thay đổi chúng và lab/build sheet yêu cầu original state được restore.

Không blindly disable WinRM hoặc RDP nếu chúng đã enabled trong baseline.

Quy tắc:

```text
Rollback to captured original state
```

chứ không:

```text
Rollback to what I assume original state was
```

---

# 63. Post-cleanup verification

Verify temporary objects không còn:

```powershell
Get-SmbShare `
    -Name Day3Data `
    -ErrorAction SilentlyContinue
```

```powershell
Get-NetFirewallRule `
    -DisplayName "DAY3-LAB-ICMP-ECHO" `
    -ErrorAction SilentlyContinue
```

```powershell
Get-ScheduledTask `
    -TaskName Day3-HealthCheck `
    -ErrorAction SilentlyContinue
```

Nhưng evidence files phải còn:

```powershell
Get-ChildItem `
    C:\Day3Lab\Evidence `
    -Recurse `
    -File
```

Hash final evidence.

---

# 64. Escalation boundary trong Day 3 Lab

L1/L2 có thể thực hiện read-only inspection và các controlled operations nằm trong approved runbook. Nhưng phải dừng và escalate nếu gặp tình huống như unauthorized privileged access requirement, unexpected production-like data, Domain Controller problem, unresolved GPO override, boot failure, disk corruption, security compromise indicators, repeated application crash, failed recovery, Event Log tampering hoặc một corrective action yêu cầu thay đổi ngoài ticket.

Một escalation tốt nên có dạng:

```text
Asset:
WIN-SRV-D3-01

Impact:
Day3LabService unavailable.

First observed:
2026-10-20 14:12 +07:00

Findings:
Service stopped.
Startup type Automatic.
Required services running.
System Event ID xxx captured.
Disk and network healthy.

Actions:
Start attempted once under CHG-D3-001.
Service stopped again after 12 seconds.

Evidence:
D3-E021.evtx
D3-E022.csv
D3-E023-service.txt

Current state:
Service unavailable.

Request:
Escalation to Application L2 for application crash analysis.
```

Không escalate:

```text
"Server lỗi, nhờ check."
```

---

# 65. Expected evidence package cuối Day 3

Một package tốt có thể có:

```text
Day3_WindowsServer/
│
├── 00_Manifest/
│   ├── Evidence_Manifest.csv
│   └── Evidence_Hashes.csv
│
├── 01_Inventory/
│   ├── OS_Inventory.csv
│   ├── InstalledFeatures.csv
│   ├── EditionMatrix.xlsx/csv
│   └── Core_vs_Desktop.txt
│
├── 02_Identity/
│   ├── LocalUsers.csv
│   ├── Groups.csv
│   └── AdminMembership.csv
│
├── 03_Permissions/
│   ├── NTFS_Before.txt
│   ├── NTFS_After.txt
│   └── SMB_Access.csv
│
├── 04_Network/
│   ├── IPConfiguration.txt
│   ├── Routes.csv
│   ├── DNS.txt
│   └── FirewallProfiles.csv
│
├── 05_RemoteManagement/
│   ├── RDP_State.txt
│   └── WinRM_State.txt
│
├── 06_Services/
│   ├── Services.csv
│   ├── Service_Before.txt
│   └── Service_After.txt
│
├── 07_Storage/
│   └── Volumes.csv
│
├── 08_Updates/
│   └── HotFixes.csv
│
├── 09_Tasks/
│   ├── Day3-HealthCheck.ps1
│   ├── HealthCheck_*.json
│   └── TaskStatus.txt
│
├── 10_Events/
│   ├── Raw/
│   │   ├── System.evtx
│   │   └── Security.evtx
│   └── Derived/
│       ├── System_Summary.csv
│       └── Security_Summary.csv
│
├── 11_FailureTests/
│   ├── ServiceFailure/
│   ├── PermissionFailure/
│   └── EventQueryNegativeTests/
│
├── 12_Recovery/
│   └── Restore_Hash_Comparison.txt
│
└── 13_Final/
    ├── Windows_Server_Maintenance_Checklist.xlsx/csv
    ├── Issues_and_Escalations.txt
    └── Cleanup_Verification.txt
```

Syllabus yêu cầu cuối cùng phải verify service, scheduled-task và patch/reboot status, đồng thời preserve Event ID, Record ID, Provider, host, timestamp và hashes. Infrastructure_Operations_14_Da…

---

# 66. Day 3 Module Thực hành — Acceptance criteria

Bạn được xem là hoàn thành tốt module khi reviewer có thể độc lập xác nhận rằng bạn đã provision hoặc sử dụng đúng Windows Server lab; identify đúng release, edition, build và installation mode; biết version/edition nào hands-on và version nào chỉ reference; tạo và quản lý controlled local identity theo least privilege; demonstrate NTFS và SMB effective permissions bằng cả allow và deny case; validate RDP/WinRM mà không mở quyền quá mức; inspect firewall/network/storage/services/processes/tasks/updates; tạo PowerShell health check; schedule và verify script; inspect domain/GPO nếu available; execute controlled service và permission failure; recover đúng root cause thay vì workaround rộng; collect bounded event evidence; preserve native EVTX và derived CSV riêng; hash evidence; distinguish empty result, unavailable channel và access denied; rollback temporary changes; và hoàn thành maintenance/escalation record.

Điểm quan trọng nhất của module không phải số commands bạn nhớ, mà là khả năng thực hiện quy trình:

```text
OBSERVE
   ↓
UNDERSTAND
   ↓
CHANGE ONLY IF AUTHORIZED
   ↓
VERIFY
   ↓
COLLECT EVIDENCE
   ↓
RECOVER / ROLLBACK
   ↓
DOCUMENT
```

---

# 67. Day 3 Practical Cheat Sheet

| Mục đích            | Command                                         |
| ------------------- | ----------------------------------------------- |
| OS info             | `Get-ComputerInfo`                              |
| Host/domain         | `Get-CimInstance Win32_ComputerSystem`          |
| Roles               | `Get-WindowsFeature`                            |
| Users               | `Get-LocalUser`                                 |
| Groups              | `Get-LocalGroup`, `Get-LocalGroupMember`        |
| NTFS                | `Get-Acl`, `icacls`                             |
| SMB                 | `Get-SmbShare`, `Get-SmbShareAccess`            |
| Services            | `Get-Service`, `Get-CimInstance Win32_Service`  |
| Processes           | `Get-Process`                                   |
| Network             | `Get-NetIPConfiguration`                        |
| IP                  | `Get-NetIPAddress`                              |
| Routes              | `Get-NetRoute`                                  |
| DNS                 | `Resolve-DnsName`                               |
| Connectivity        | `Test-NetConnection`                            |
| Firewall            | `Get-NetFirewallProfile`, `Get-NetFirewallRule` |
| WinRM               | `Test-WSMan`                                    |
| Remoting            | `Invoke-Command`                                |
| Storage             | `Get-Disk`, `Get-Partition`, `Get-Volume`       |
| Updates             | `Get-HotFix`, `Get-WindowsPackage -Online`      |
| Tasks               | `Get-ScheduledTask`, `Get-ScheduledTaskInfo`    |
| Events              | `Get-WinEvent`                                  |
| Native Event export | `wevtutil epl`                                  |
| Hash                | `Get-FileHash -Algorithm SHA256`                |
| GPO result          | `gpresult /r`, `gpresult /h ...`                |

Một practical rule đáng nhớ hơn toàn bộ bảng trên là:

```text
Get-* / inspect
       ↓
collect baseline
       ↓
smallest authorized change
       ↓
Get-* / inspect again
       ↓
functional test
       ↓
evidence
```

---

# 68. Day Completion Checklist — Module Thực hành

- [ ] Windows Server VM provisioned/accessible trong authorized lab.
- [ ] Version, edition, build và installation mode được xác định.
- [ ] Release-by-edition matrix được tạo và phân biệt Hands-on/Demonstrated/Reference-only.
- [ ] Server Core/Desktop Experience comparison được hoàn thành.
- [ ] Installed Roles/Features được inventoried.
- [ ] Controlled local user/group được tạo và verify.
- [ ] Least-privilege group-based permission được demonstrate.
- [ ] NTFS Read/Modify behavior được kiểm tra.
- [ ] SMB Share + NTFS effective permission được kiểm tra.
- [ ] Administrative group membership được review.
- [ ] RDP được verify an toàn.
- [ ] Windows Defender Firewall được inventory và temporary rule được rollback.
- [ ] IP/DNS/routes/listeners được baseline.
- [ ] WinRM/PowerShell Remoting được inspect/test khi available.
- [ ] Service inventory được thu thập.
- [ ] Authorized service restart được thực hiện với pre/post-check.
- [ ] Process/service/PID relationship được kiểm tra.
- [ ] Storage/volume capacity được inspect.
- [ ] Patch/update/reboot indicators được inspect.
- [ ] Scheduled Tasks được inspect.
- [ ] PowerShell health-check script chạy thành công.
- [ ] Health-check được scheduled và actual output được verify.
- [ ] Domain membership/GPO được inspect nếu lab có AD.
- [ ] Controlled restore/integrity verification được thực hiện.
- [ ] System/Application/Security Event Logs được inspect.
- [ ] `Get-WinEvent` bounded queries được thực hiện.
- [ ] Native EVTX được export và đọc lại thành công.
- [ ] Separate UTF-8 CSV summary được tạo.
- [ ] Event ID, Record ID, Provider, host và timestamp được giữ.
- [ ] Empty-result test được hoàn thành.
- [ ] Unavailable-channel test được hoàn thành.
- [ ] Access-denied event test được hoàn thành hoặc có fixture.
- [ ] Service failure injection được troubleshoot và recover.
- [ ] Permission failure injection được troubleshoot và least privilege được restore.
- [ ] Temporary lab configuration được cleanup/rollback.
- [ ] Evidence manifest và SHA-256 hashes được hoàn tất.
- [ ] Final Windows Server maintenance checklist được hoàn thành.
- [ ] Remaining gaps/unresolved issues được ghi rõ thay vì đánh dấu giả là Pass.

Nếu toàn bộ checklist này đạt, bạn đã hoàn thành **Module Thực hành Day 3** đúng mục tiêu syllabus: không chỉ biết thao tác Windows Server, mà còn biết **baseline → change → verify → troubleshoot → evidence → rollback → escalation** trong một workflow có thể audit được.
