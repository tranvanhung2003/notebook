# Day 3 — Module Thực hành

## Windows Server Administration & Maintenance Lab

Theo syllabus, phần **Assignment/Lab của Day 3** yêu cầu bạn phải: provision một Windows Server VM được phê duyệt; xác định version, edition, installation mode và installed roles; cấu hình local users/groups, NTFS/share permissions, secure administrative access, firewall và remote management; inspect services, scheduled tasks, storage, networking, updates và Event Logs bằng GUI lẫn PowerShell; thực hiện authorized service restart; tạo scheduled health-check script; inspect domain/Group Policy nếu environment hỗ trợ; mô phỏng service hoặc permission failure; extract event evidence; và cuối cùng nộp **Windows Server configuration and maintenance checklist**. Infrastructure_Operations_14_Da…

Toàn bộ lab dưới đây mặc định chạy trong **authorized lab/non-production environment**. Không áp dụng trực tiếp lên production server nếu chưa có change authorization, backup/recovery plan, maintenance window và rollback được phê duyệt.

---

# 1. Mục tiêu của Module Thực hành Day 3

Sau khi hoàn thành module này, mục tiêu không phải chỉ là “biết click Windows Server”.

Bạn phải có khả năng nhận một server tương đối xa lạ và thực hiện được workflow:

```text
Identify
   ↓
Baseline
   ↓
Inspect
   ↓
Configure safely
   ↓
Verify
   ↓
Troubleshoot
   ↓
Collect evidence
   ↓
Rollback / recover
   ↓
Escalate when required
   ↓
Document
```

Bạn cần chứng minh được bốn năng lực riêng biệt:

| Năng lực        | Bạn phải làm được                                  |
| --------------- | -------------------------------------------------- |
| Administration  | Thực hiện configuration đúng                       |
| Verification    | Chứng minh configuration thực sự có hiệu lực       |
| Troubleshooting | Xác định layer/root cause thay vì đoán             |
| Operations      | Có evidence, rollback, authorization và escalation |

---

# 2. Lab environment đề xuất

## 2.1. Topology tối thiểu

Lab tối thiểu:

```text
+-------------------------+
| Admin Workstation       |
| Windows 10/11           |
| PowerShell              |
| RDP client              |
+------------+------------+
             |
             | Lab network
             |
+------------v------------+
| SRV-D3-01               |
| Windows Server          |
| Standard/Datacenter     |
| Desktop Experience      |
| hoặc Server Core        |
+-------------------------+
```

Nếu có AD lab:

```text
                         +----------------------+
                         | DC01                 |
                         | AD DS + DNS          |
                         +----------+-----------+
                                    |
                                    |
+------------------+      +---------+---------+
| Admin Workstation|------| SRV-D3-01         |
+------------------+      | Member Server      |
                          +-------------------+
```

Nếu không có domain environment thì phần AD/GPO sẽ được thực hiện dưới dạng inspection/awareness.

---

# 3. Naming convention dùng trong lab

Để tránh command mơ hồ, chúng ta giả định:

```text
Server:
SRV-D3-01

Lab folder:
C:\Day3Lab

Data folder:
C:\Day3Lab\Data

Evidence:
C:\Day3Lab\Evidence

Scripts:
C:\Day3Lab\Scripts

Reports:
C:\Day3Lab\Reports

Local user:
lab_user01

Local group:
LAB_File_Modify

SMB share:
Day3Data

Firewall test rule:
DAY3-Allow-TestPort
```

Nếu lab của bạn có tên khác, thay các giá trị trên bằng environment thực tế.

---

# 4. Phân loại command trước khi thực hiện

Trong module này tôi sẽ dùng ba classification.

### READ-ONLY

Không chủ động thay đổi configuration.

Ví dụ:

```powershell
Get-Service
Get-NetIPConfiguration
Get-Volume
Get-WinEvent
```

### CONFIG CHANGE

Có thay đổi state/configuration.

Ví dụ:

```powershell
New-LocalUser
New-SmbShare
New-NetFirewallRule
Restart-Service
```

### HIGHER-RISK / APPROVAL REQUIRED

Có thể ảnh hưởng rộng hoặc gây outage.

Ví dụ:

```text
Reboot production server
Change IP/default gateway
Modify domain GPO
Disable firewall
Format disk
Delete production data
Change Domain Controller configuration
Install major patches
Restore production volume
```

Day 3 lab không yêu cầu bạn thực hiện destructive operations.

---

# 5. Evidence repository

Trước khi làm bất kỳ configuration nào, tạo nơi lưu evidence.

**CONFIG CHANGE — safe lab operation**

```powershell
New-Item -ItemType Directory -Path C:\Day3Lab -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Evidence -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Scripts -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Reports -Force
New-Item -ItemType Directory -Path C:\Day3Lab\Data -Force
```

Verify:

```powershell
Get-ChildItem C:\Day3Lab
```

Expected:

```text
Data
Evidence
Reports
Scripts
```

Evidence nên được lưu với timestamp hoặc tên có ý nghĩa:

```text
01-system-baseline.txt
02-local-users.txt
03-network.txt
04-storage.txt
05-services-before.txt
06-services-after.txt
...
```

Một naming convention tốt hơn trong enterprise có thể là:

```text
SRV-D3-01_20261019_093000_NetworkBaseline.txt
```

---

# 6. Lab 1 — Provision và xác nhận Windows Server VM

## Objective

Provision hoặc sử dụng một Windows Server VM được phép dùng cho lab.

Syllabus chỉ nói **approved Windows Server virtual machine**, không quy định hypervisor cụ thể. Vì vậy lab có thể nằm trên:

```text
Hyper-V
VMware
Nutanix AHV
Azure
AWS
VirtualBox trong môi trường cá nhân
```

miễn là được phép.

## Pre-check

Trước khi thay đổi server, ghi nhận:

```text
VM name
vCPU
Memory
Disk
NIC
Operating System
Snapshot/checkpoint policy
Current power state
```

### Safety

Nếu hypervisor hỗ trợ snapshot/checkpoint và lab policy cho phép, tạo restore point trước lab.

Nhưng cần nhớ:

> Hypervisor snapshot không phải universal replacement cho backup.

---

# 7. Xác nhận hostname

**READ-ONLY**

```powershell
hostname
```

Hoặc:

```powershell
$env:COMPUTERNAME
```

Expected:

```text
SRV-D3-01
```

Nếu hostname không đúng so với build sheet, không nên tự rename production/server domain member mà chưa kiểm tra dependency.

Trong lab có thể rename nếu assignment yêu cầu.

---

# 8. Thu thập system inventory bằng PowerShell

Microsoft hiện cung cấp `Get-ComputerInfo` để lấy consolidated system và operating-system properties. [Microsoft Learn](https://learn.microsoft.com/vi-vn/powershell/module/microsoft.powershell.management/get-computerinfo?view=powershell-7.7&utm_source=chatgpt.com)

**READ-ONLY**

```powershell
Get-ComputerInfo
```

Output khá lớn.

Nên chọn các property quan trọng:

```powershell
Get-ComputerInfo |
Select-Object `
    CsName,
    WindowsProductName,
    WindowsVersion,
    OsName,
    OsVersion,
    OsBuildNumber,
    CsDomain,
    CsDomainRole
```

Expected information:

```text
Hostname
OS product
Version
Build
Domain/workgroup context
```

Lưu evidence:

```powershell
Get-ComputerInfo |
Out-File C:\Day3Lab\Evidence\01-ComputerInfo.txt
```

---

# 9. Xác định edition

Một cách phổ biến:

```powershell
Get-ComputerInfo |
Select-Object WindowsProductName
```

Hoặc:

```powershell
Get-CimInstance Win32_OperatingSystem |
Select-Object Caption, Version, BuildNumber
```

Ví dụ output:

```text
Caption                  Version       BuildNumber
-------                  -------       -----------
Microsoft Windows Server ... ...
```

Bạn cần ghi vào checklist:

```text
Version/Generation:
Edition:
Build:
Installation mode:
```

Không ghi chung chung:

```text
Windows Server
```

vì chưa đủ làm baseline.

---

# 10. Xác định Server Core hay Desktop Experience

Có thể suy luận bằng GUI presence, nhưng operationally nên dùng command/configuration information.

Một check hữu ích:

```powershell
Get-WindowsFeature Server-Gui-Shell, Server-Gui-Mgmt-Infra
```

Tuy nhiên exact feature behavior khác nhau giữa generation, và ở releases hiện đại installation option được quyết định trong installation process.

Trong lab, bạn phải ghi rõ:

```text
Installation mode:
Server Core
hoặc
Server with Desktop Experience
```

Nếu VM có Explorer/Desktop shell đầy đủ thì đang dùng Desktop Experience.

---

# 11. Installed Roles and Features

**READ-ONLY**

```powershell
Get-WindowsFeature
```

Chỉ xem installed:

```powershell
Get-WindowsFeature |
Where-Object InstallState -eq 'Installed'
```

Evidence:

```powershell
Get-WindowsFeature |
Where-Object InstallState -eq 'Installed' |
Out-File C:\Day3Lab\Evidence\02-InstalledFeatures.txt
```

Bạn cần biết server có role nào như:

```text
File Server
DNS Server
DHCP Server
Web Server (IIS)
AD DS
Hyper-V
```

Không được cài thêm role chỉ để “thử” nếu syllabus không yêu cầu.

---

# 12. Baseline trước thay đổi

Đây là điểm rất quan trọng.

Trước configuration lab, collect baseline:

```powershell
Get-Service |
Sort-Object Name |
Out-File C:\Day3Lab\Evidence\03-Services-Before.txt

Get-NetIPConfiguration |
Format-List * |
Out-File C:\Day3Lab\Evidence\04-Network-Before.txt

Get-Volume |
Format-Table -AutoSize |
Out-File C:\Day3Lab\Evidence\05-Volumes-Before.txt

Get-NetFirewallProfile |
Out-File C:\Day3Lab\Evidence\06-FirewallProfiles-Before.txt
```

Microsoft documentation hiện tại xác nhận `Get-NetIPConfiguration` trả network configuration gồm usable interfaces, IP addresses và DNS servers; `Get-Volume` trả các volume objects; `Get-NetFirewallProfile` cho thấy per-profile firewall configuration. [Microsoft Learn](https://learn.microsoft.com/bs-latn-ba/powershell/module/nettcpip/get-netipconfiguration?view=windowsserver2025-ps&utm_source=chatgpt.com)

---

# 13. Lab 2 — Local users và groups

## Objective

Tạo một controlled local user và local group theo least privilege.

Không dùng real credential.

---

# 14. Inspect existing local users

**READ-ONLY**

```powershell
Get-LocalUser
```

Microsoft `Get-LocalUser` trả local user accounts. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/get-localuser?view=powershell-5.1&utm_source=chatgpt.com)

Có thể chọn properties:

```powershell
Get-LocalUser |
Select-Object Name, Enabled, LastLogon, PasswordRequired
```

Evidence:

```powershell
Get-LocalUser |
Out-File C:\Day3Lab\Evidence\07-LocalUsers-Before.txt
```

---

# 15. Inspect local groups

**READ-ONLY**

```powershell
Get-LocalGroup
```

Check Administrators membership:

```powershell
Get-LocalGroupMember -Group Administrators
```

Microsoft xác nhận `Get-LocalGroupMember` lấy các thành viên của local group. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/get-localgroupmember?utm_source=chatgpt.com)

Đây là một security evidence cực kỳ quan trọng.

---

# 16. Tạo local group

**CONFIG CHANGE**

```powershell
New-LocalGroup `
    -Name 'LAB_File_Modify' `
    -Description 'Day 3 lab controlled file access group'
```

Microsoft `New-LocalGroup` tạo local security group trong Security Account Manager. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/new-localgroup?view=powershell-5.1&utm_source=chatgpt.com)

Verify:

```powershell
Get-LocalGroup -Name 'LAB_File_Modify'
```

Expected:

```text
LAB_File_Modify
```

---

# 17. Tạo local user an toàn

Không hard-code password như:

```powershell
$Password = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
```

Trong training, command này thường xuất hiện vì tiện nhưng tạo thói quen xấu.

Thay vào đó:

```powershell
$Password = Read-Host 'Enter lab password' -AsSecureString
```

Sau đó:

```powershell
New-LocalUser `
    -Name 'lab_user01' `
    -Password $Password `
    -Description 'Day 3 training user'
```

Microsoft documentation cho `New-LocalUser` hỗ trợ `SecureString` password và tạo local account. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.localaccounts/new-localuser?view=powershell-5.1&utm_source=chatgpt.com)

Verify:

```powershell
Get-LocalUser -Name 'lab_user01'
```

---

# 18. Thêm user vào controlled group

**CONFIG CHANGE**

```powershell
Add-LocalGroupMember `
    -Group 'LAB_File_Modify' `
    -Member 'lab_user01'
```

Verify:

```powershell
Get-LocalGroupMember -Group 'LAB_File_Modify'
```

Expected:

```text
lab_user01
```

Không add user vào `Administrators` chỉ vì thuận tiện.

---

# 19. Least privilege verification

Bạn cần chứng minh:

```powershell
Get-LocalGroupMember -Group Administrators
```

không chứa:

```text
lab_user01
```

trừ khi lab explicitly yêu cầu administrative account.

Evidence:

```powershell
Get-LocalGroupMember -Group 'LAB_File_Modify' |
Out-File C:\Day3Lab\Evidence\08-LabGroupMembership.txt
```

---

# 20. Rollback local identity

Sau lab:

```powershell
Remove-LocalGroupMember `
    -Group 'LAB_File_Modify' `
    -Member 'lab_user01'
```

Sau đó:

```powershell
Remove-LocalUser -Name 'lab_user01'
Remove-LocalGroup -Name 'LAB_File_Modify'
```

Chỉ cleanup sau khi toàn bộ permission tests hoàn thành.

---

# 21. Lab 3 — NTFS permissions

## Objective

Hiểu và thực hành:

```text
ACL
ACE
Allow
Inheritance
Group-based access
Effective permissions
```

Không sử dụng production folders.

---

# 22. Tạo test directory

**CONFIG CHANGE**

```powershell
New-Item `
    -ItemType Directory `
    -Path C:\Day3Lab\Data\Finance `
    -Force
```

Tạo test file:

```powershell
'Day 3 controlled test data' |
Set-Content C:\Day3Lab\Data\Finance\sample.txt
```

---

# 23. Inspect ACL hiện tại

**READ-ONLY**

```powershell
Get-Acl C:\Day3Lab\Data\Finance
```

Chi tiết:

```powershell
(Get-Acl C:\Day3Lab\Data\Finance).Access
```

`Get-Acl` lấy security descriptor và ACL của file/resource. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-acl?view=powershell-7.4&utm_source=chatgpt.com)

Bạn cần nhận diện:

```text
IdentityReference
FileSystemRights
AccessControlType
IsInherited
InheritanceFlags
PropagationFlags
```

---

# 24. Cấu hình NTFS permission

Để lab dễ quan sát, có thể dùng `icacls`.

**CONFIG CHANGE**

Ví dụ:

```cmd
icacls C:\Day3Lab\Data\Finance /grant LAB_File_Modify:(OI)(CI)M
```

Trong đó:

```text
LAB_File_Modify
= principal được cấp quyền

OI
= Object Inherit

CI
= Container Inherit

M
= Modify
```

Ý nghĩa operational:

Group có thể Modify folder và contents theo inheritance configuration.

---

# 25. Verify NTFS ACL

```cmd
icacls C:\Day3Lab\Data\Finance
```

Và:

```powershell
(Get-Acl C:\Day3Lab\Data\Finance).Access |
Format-Table IdentityReference,
            FileSystemRights,
            AccessControlType,
            IsInherited -AutoSize
```

Evidence:

```powershell
(Get-Acl C:\Day3Lab\Data\Finance).Access |
Out-File C:\Day3Lab\Evidence\09-NTFS-ACL.txt
```

---

# 26. Test bằng actual identity

Đây là bước thường bị bỏ qua.

Không đủ để nói:

> ACL nhìn đúng.

Bạn phải test dưới identity được cấp quyền.

Ví dụ đăng nhập/`runas` bằng `lab_user01` trong lab và thử:

```text
Read sample.txt
Create new file
Modify file
Delete controlled test file
```

Nếu permission là Modify thì các operation phù hợp phải hoạt động.

Bạn cũng cần test một user **không** nằm trong group.

---

# 27. Permission troubleshooting workflow

Nếu `lab_user01` không có quyền như expected:

```text
Confirm identity
       ↓
Check group membership
       ↓
Check ACL
       ↓
Check inheritance
       ↓
Check explicit Deny
       ↓
Check token/session
       ↓
Retest
```

Không giải quyết bằng:

```text
Everyone: Full Control
```

---

# 28. Lab 4 — SMB share permissions

## Objective

Hiểu interaction giữa:

```text
Share ACL
+
NTFS ACL
```

---

# 29. Inspect SMB shares

**READ-ONLY**

```powershell
Get-SmbShare
```

Microsoft `Get-SmbShare` lấy SMB shares hiện có trên computer. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbshare?view=windowsserver2022-ps&utm_source=chatgpt.com)

Bạn sẽ thấy các administrative shares như:

```text
C$
ADMIN$
IPC$
```

Không modify/delete chúng trong lab.

---

# 30. Tạo lab SMB share

**CONFIG CHANGE**

```powershell
New-SmbShare `
    -Name 'Day3Data' `
    -Path 'C:\Day3Lab\Data\Finance' `
    -ChangeAccess 'LAB_File_Modify' `
    -FullAccess 'Administrators'
```

`New-SmbShare` expose filesystem folder cho remote SMB clients và hỗ trợ `FullAccess`, `ChangeAccess`, `ReadAccess`, `NoAccess`. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/smbshare/new-smbshare?view=windowsserver2025-ps&utm_source=chatgpt.com)

---

# 31. Verify SMB share

```powershell
Get-SmbShare -Name 'Day3Data'
```

Expected:

```text
Name      Path
----      ----
Day3Data  C:\Day3Lab\Data\Finance
```

---

# 32. Inspect share ACL

```powershell
Get-SmbShareAccess -Name 'Day3Data'
```

Microsoft xác nhận `Get-SmbShareAccess` lấy ACL của SMB share. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/smbshare/get-smbshareaccess?utm_source=chatgpt.com)

Expected principal:

```text
LAB_File_Modify
```

với `Change`.

---

# 33. Kiểm thử từ network

Từ workstation hoặc chính server qua UNC:

```text
\\SRV-D3-01\Day3Data
```

Hoặc PowerShell:

```powershell
Test-Path '\\SRV-D3-01\Day3Data'
```

Để kiểm tra port:

```powershell
Test-NetConnection SRV-D3-01 -Port 445
```

Microsoft documentation cho `Test-NetConnection` hỗ trợ TCP connectivity diagnostics. [Microsoft Learn](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection?utm_source=chatgpt.com)

---

# 34. Phân biệt lỗi SMB và NTFS

Nếu:

```text
Local access works
Network access fails
```

hãy nghĩ tới:

```text
SMB share
Share permission
TCP/445
Firewall
DNS
Authentication
```

Nếu:

```text
Network share opens
nhưng file cannot modify
```

hãy kiểm tra cả:

```text
Share Change?
NTFS Modify?
Deny?
Group membership?
```

---

# 35. Rollback SMB share

**CONFIG CHANGE**

Sau lab:

```powershell
Remove-SmbShare -Name 'Day3Data'
```

Cần confirmation.

Lưu ý:

Remove share không nhất thiết xóa underlying folder/data.

Đây chính là sự khác biệt:

```text
Remove access path
vs
Delete data
```

---

# 36. Lab 5 — Secure administrative access

Phần này gồm:

```text
Console/local access
RDP awareness
WinRM
PowerShell Remoting
Firewall
Authorization
```

---

# 37. Kiểm tra RDP trước khi thay đổi

Một server không nhất thiết cần RDP enabled.

Nếu lab đã bật RDP, test port:

```powershell
Test-NetConnection SRV-D3-01 -Port 3389
```

Nhưng:

```text
TcpTestSucceeded = True
```

chỉ chứng minh TCP reachability.

Nó không chứng minh:

```text
User authorized
Authentication works
Desktop session healthy
```

---

# 38. RDP authorization

Một non-admin account thường phải thuộc appropriate group để được RDP theo policy.

Inspect:

```powershell
Get-LocalGroupMember -Group 'Remote Desktop Users'
```

Không add user vào Administrators chỉ để RDP.

Nếu lab cần:

```powershell
Add-LocalGroupMember `
    -Group 'Remote Desktop Users' `
    -Member 'lab_user01'
```

Sau đó verify:

```powershell
Get-LocalGroupMember -Group 'Remote Desktop Users'
```

---

# 39. WinRM inspection

Trước khi enable gì, kiểm tra trước.

**READ-ONLY**

```powershell
Get-Service WinRM
```

Sau đó:

```powershell
Test-WSMan localhost
```

`Test-WSMan` gửi identification request để xác định WinRM service local/remote có hoạt động hay không. [Microsoft Learn](https://learn.microsoft.com/powershell/module/microsoft.wsman.management/test-wsman?utm_source=chatgpt.com)

Remote:

```powershell
Test-WSMan SRV-D3-01
```

---

# 40. PowerShell Remoting

Trên Windows Server hiện đại, PowerShell remoting thường đã được enabled mặc định, vì vậy không nên chạy `Enable-PSRemoting` theo reflex trước khi kiểm tra. Microsoft documentation hiện tại cũng ghi PowerShell remoting được enabled mặc định trên Windows Server platforms. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting?view=powershell-7.5&utm_source=chatgpt.com)

Nếu environment lab yêu cầu re-enable:

**CONFIG CHANGE**

```powershell
Enable-PSRemoting -Force
```

Command này có thể cấu hình WinRM, listener, firewall exceptions và session endpoints, do đó phải hiểu nó là configuration-changing operation. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/enable-psremoting?view=powershell-7.5&utm_source=chatgpt.com)

---

# 41. Test interactive remoting

Từ authorized admin workstation:

```powershell
Enter-PSSession -ComputerName SRV-D3-01
```

Khi vào session:

```powershell
hostname
```

Expected:

```text
SRV-D3-01
```

Exit:

```powershell
Exit-PSSession
```

---

# 42. Test non-interactive remoting

```powershell
Invoke-Command `
    -ComputerName SRV-D3-01 `
    -ScriptBlock {
        Get-Service WinRM
    }
```

Điểm quan trọng:

Remote command phải trả service information từ **remote machine**, không phải local admin workstation.

---

# 43. Remote-management troubleshooting

Nếu `Enter-PSSession` fail:

```text
Step 1 — DNS
Resolve-DnsName SRV-D3-01

Step 2 — Network
Test-NetConnection SRV-D3-01

Step 3 — WinRM
Test-WSMan SRV-D3-01

Step 4 — Service
Get-Service WinRM

Step 5 — Firewall
Check applicable rule/profile

Step 6 — Authentication
Correct account/domain?

Step 7 — Authorization
Does identity have remote access?

Step 8 — Endpoint configuration
PowerShell session configuration?
```

Không nhảy ngay tới:

```text
Disable firewall
```

---

# 44. Lab 6 — Windows Firewall

## Objective

Tạo một controlled inbound test rule, verify và rollback.

---

# 45. Inspect firewall profiles

**READ-ONLY**

```powershell
Get-NetFirewallProfile
```

Bạn cần biết profile nào đang active.

Có thể dùng:

```powershell
Get-NetConnectionProfile
```

để xem network category/context.

---

# 46. Inspect firewall rules

```powershell
Get-NetFirewallRule |
Where-Object Enabled -eq 'True'
```

Hoặc specific:

```powershell
Get-NetFirewallRule -DisplayName '*Remote Desktop*'
```

`Get-NetFirewallRule` trả firewall rule objects; Microsoft lưu ý port/address conditions nằm ở associated filter objects chứ không phải lúc nào cũng hiện ngay trên rule object. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/netsecurity/get-netfirewallrule?view=windowsserver2025-ps&utm_source=chatgpt.com)

---

# 47. Firewall rule + port filter

Ví dụ:

```powershell
Get-NetFirewallRule -DisplayName '*Remote Desktop*' |
Get-NetFirewallPortFilter
```

Điều này giúp xem:

```text
Protocol
LocalPort
RemotePort
```

---

# 48. Controlled firewall rule

Giả sử lab có một test application lắng nghe TCP/8080.

**CONFIG CHANGE**

```powershell
New-NetFirewallRule `
    -DisplayName 'DAY3-Allow-TestPort' `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 8080 `
    -Action Allow `
    -Profile Private
```

Microsoft `New-NetFirewallRule` hỗ trợ direction, action, profile, addresses, protocol và ports. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule?view=windowsserver2025-ps&viewFallbackFrom=win10-ps&utm_source=chatgpt.com)

---

# 49. Verify rule

```powershell
Get-NetFirewallRule `
    -DisplayName 'DAY3-Allow-TestPort'
```

Port:

```powershell
Get-NetFirewallRule `
    -DisplayName 'DAY3-Allow-TestPort' |
Get-NetFirewallPortFilter
```

---

# 50. Một nguyên tắc quan trọng: rule tồn tại không có nghĩa port mở

Bạn cần phân biệt:

```text
Firewall allows TCP/8080
```

với:

```text
Application actually listening on TCP/8080
```

Check:

```powershell
Get-NetTCPConnection -LocalPort 8080 -State Listen
```

`Get-NetTCPConnection` cung cấp local/remote IP, port và TCP state. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection?view=windowsserver2025-ps&utm_source=chatgpt.com)

Nếu không có listener:

```text
Firewall rule đúng
+
Application không listen
=
Connection vẫn fail
```

---

# 51. Test allowed traffic

Từ client:

```powershell
Test-NetConnection SRV-D3-01 -Port 8080
```

Expected nếu listener hoạt động:

```text
TcpTestSucceeded : True
```

Nếu `False`, investigation:

```text
DNS
Route
Listener
Firewall
Profile
Rule scope
Application
```

---

# 52. Firewall rollback

```powershell
Remove-NetFirewallRule `
    -DisplayName 'DAY3-Allow-TestPort'
```

Verify:

```powershell
Get-NetFirewallRule `
    -DisplayName 'DAY3-Allow-TestPort' `
    -ErrorAction SilentlyContinue
```

No result expected.

---

# 53. Lab 7 — Services

## Objective

Inspect services và thực hiện một **authorized service restart**.

Không chọn critical infrastructure service tùy tiện.

Trong lab nên chọn một service không critical hoặc service do mentor chuẩn bị.

---

# 54. Inspect service state

```powershell
Get-Service |
Sort-Object Status, Name
```

Một service cụ thể:

```powershell
Get-Service -Name Spooler
```

Lưu ý: `Spooler` chỉ nên được dùng nếu safe trong VM lab và không có dependency cần giữ.

---

# 55. Inspect startup configuration sâu hơn

`Get-Service` không expose mọi configuration.

Dùng CIM:

```powershell
Get-CimInstance Win32_Service |
Select-Object Name,
              DisplayName,
              State,
              StartMode,
              StartName
```

Một service:

```powershell
Get-CimInstance Win32_Service `
    -Filter "Name='Spooler'" |
Select-Object *
```

Bạn cần biết:

```text
State
Start mode
Logon account
Path
Process ID
```

---

# 56. Dependency inspection

```powershell
Get-Service -Name Spooler -DependentServices
```

Và:

```powershell
Get-Service -Name Spooler -RequiredServices
```

Trước restart một service, phải hiểu:

```text
What depends on it?
What does it depend on?
```

---

# 57. Pre-change evidence

```powershell
Get-Service -Name Spooler |
Out-File C:\Day3Lab\Evidence\10-Service-Before.txt
```

Và timestamp:

```powershell
Get-Date |
Out-File C:\Day3Lab\Evidence\10-Service-Before-Time.txt
```

---

# 58. Authorized service restart

**CONFIG CHANGE**

```powershell
Restart-Service -Name Spooler
```

Không dùng `-Force` theo thói quen nếu không cần.

---

# 59. Verify service restart

```powershell
Get-Service -Name Spooler
```

Expected:

```text
Status = Running
```

Nhưng chưa đủ.

Nếu đó là application service thì cần:

```text
Service check
+
Port check
+
Application health check
```

Ví dụ:

```powershell
Test-NetConnection localhost -Port <application-port>
```

---

# 60. Check related Event Logs

Sau restart:

```powershell
Get-WinEvent `
    -FilterHashtable @{
        LogName   = 'System'
        StartTime = (Get-Date).AddMinutes(-10)
    } |
Select-Object TimeCreated, Id, ProviderName, LevelDisplayName, Message
```

Bạn đang tìm events liên quan thời gian restart.

---

# 61. Service evidence sau change

```powershell
Get-Service -Name Spooler |
Out-File C:\Day3Lab\Evidence\11-Service-After.txt
```

Ticket/evidence nên trả lời:

```text
What was restarted?
Why?
Who authorized?
When?
Previous state?
Post-state?
Application verified?
Any errors?
```

---

# 62. Lab 8 — Process inspection

## Objective

Phân biệt process state với service/application state.

---

# 63. Liệt kê processes

```powershell
Get-Process
```

Top CPU:

```powershell
Get-Process |
Sort-Object CPU -Descending |
Select-Object -First 10 Name, Id, CPU
```

Top memory:

```powershell
Get-Process |
Sort-Object WorkingSet64 -Descending |
Select-Object -First 10 `
    Name,
    Id,
    @{Name='MemoryMB';Expression={
        [math]::Round($_.WorkingSet64 / 1MB, 2)
    }}
```

---

# 64. Liên hệ process với service

Có thể xem PID từ `Win32_Service`:

```powershell
Get-CimInstance Win32_Service |
Where-Object State -eq 'Running' |
Select-Object Name, ProcessId
```

Sau đó:

```powershell
Get-Process -Id <PID>
```

Mục tiêu là hiểu:

```text
Service
→ Process
→ PID
→ CPU / Memory
→ Network connections
```

---

# 65. Không kill process tùy tiện

`Stop-Process -Force` có thể bypass normal service shutdown behavior.

Do đó với Windows service:

```text
Prefer service control
```

thay vì:

```text
Kill process
```

nếu service vẫn responsive.

Kill process chỉ nên dùng khi runbook/incident condition cho phép.

---

# 66. Lab 9 — Network inspection

Đây là một trong những phần quan trọng nhất của Windows Operations.

---

# 67. Network adapter

```powershell
Get-NetAdapter
```

Quan sát:

```text
Name
InterfaceDescription
Status
LinkSpeed
MAC address
```

Nếu status:

```text
Down
```

không nên tiếp tục DNS troubleshooting trước khi hiểu network link.

---

# 68. IP configuration

```powershell
Get-NetIPConfiguration
```

Microsoft xác nhận cmdlet này lấy network configuration gồm usable interfaces, addresses và DNS servers. [Microsoft Learn](https://learn.microsoft.com/bs-latn-ba/powershell/module/nettcpip/get-netipconfiguration?view=windowsserver2025-ps&utm_source=chatgpt.com)

Chi tiết:

```powershell
Get-NetIPConfiguration -Detailed
```

---

# 69. IP address

```powershell
Get-NetIPAddress
```

IPv4:

```powershell
Get-NetIPAddress `
    -AddressFamily IPv4 |
Where-Object IPAddress -notlike '169.254*'
```

Microsoft `Get-NetIPAddress` trả IPv4/IPv6 configuration và interface association. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-netipaddress?view=windowsserver2025-ps&utm_source=chatgpt.com)

---

# 70. APIPA awareness

Nếu bạn thấy:

```text
169.254.x.x
```

thường đây là automatic private address behavior, một strong clue rằng expected DHCP/static configuration có vấn đề.

Không kết luận ngay DHCP down.

Cần kiểm tra:

```text
Was interface supposed to use DHCP?
Is cable/vNIC connected?
DHCP scope?
VLAN?
DHCP relay?
```

---

# 71. Default route

```powershell
Get-NetRoute `
    -DestinationPrefix '0.0.0.0/0'
```

Bạn cần xác định:

```text
NextHop
Interface
RouteMetric
```

Không có default route không nhất thiết là lỗi nếu server intentionally isolated.

---

# 72. DNS client settings

```powershell
Get-DnsClientServerAddress
```

Xác nhận:

```text
Correct DNS servers?
Wrong public DNS on domain member?
Unexpected DNS?
```

Đặc biệt domain member không nên tùy tiện đổi DNS sang public resolver nếu enterprise AD DNS architecture yêu cầu internal DNS.

---

# 73. DNS resolution

```powershell
Resolve-DnsName SRV-D3-01
```

Hoặc target:

```powershell
Resolve-DnsName DC01
```

Bạn cần ghi:

```text
Queried name
Resolved IP
DNS server
Timestamp
```

---

# 74. Connectivity ladder

Không chỉ dùng `ping`.

Test theo layer.

### Local stack

```powershell
Test-NetConnection 127.0.0.1
```

### Gateway

```powershell
Test-NetConnection <gateway-ip>
```

### Remote server

```powershell
Test-NetConnection <remote-ip>
```

### DNS name

```powershell
Test-NetConnection SERVER02
```

### Application port

```powershell
Test-NetConnection SERVER02 -Port 443
```

---

# 75. Networking evidence

```powershell
Get-NetIPConfiguration -Detailed |
Out-File C:\Day3Lab\Evidence\12-Network.txt
```

```powershell
Get-NetRoute |
Out-File C:\Day3Lab\Evidence\13-Routes.txt
```

```powershell
Get-DnsClientServerAddress |
Out-File C:\Day3Lab\Evidence\14-DNS-Client.txt
```

---

# 76. Lab 10 — Storage

## Objective

Inspect disk → partition → volume → capacity.

---

# 77. Disk inventory

```powershell
Get-Disk
```

Microsoft `Get-Disk` lấy disks visible to OS. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/storage/get-disk?view=windowsserver2025-ps&utm_source=chatgpt.com)

Bạn cần quan sát:

```text
Number
FriendlyName
OperationalStatus
HealthStatus
Size
PartitionStyle
```

---

# 78. Partitions

```powershell
Get-Partition
```

Bạn cần liên hệ:

```text
Disk 0
  ↓
Partition
  ↓
Volume C:
```

---

# 79. Volumes

```powershell
Get-Volume
```

Microsoft `Get-Volume` trả volume objects và thông tin như filesystem, HealthStatus, Size, SizeRemaining. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/storage/get-volume?view=windowsserver2025-ps&utm_source=chatgpt.com)

Focus:

```text
DriveLetter
FileSystem
HealthStatus
Size
SizeRemaining
```

---

# 80. Capacity percentage

```powershell
Get-Volume |
Where-Object DriveLetter |
Select-Object DriveLetter,
    FileSystemLabel,
    HealthStatus,
    @{Name='SizeGB';Expression={
        [math]::Round($_.Size / 1GB, 2)
    }},
    @{Name='FreeGB';Expression={
        [math]::Round($_.SizeRemaining / 1GB, 2)
    }},
    @{Name='FreePercent';Expression={
        if ($_.Size -gt 0) {
            [math]::Round(
                ($_.SizeRemaining / $_.Size) * 100,
                2
            )
        }
    }}
```

Đây sẽ là một phần của health-check script.

---

# 81. Storage troubleshooting

Nếu volume gần đầy:

```text
1. Confirm free capacity
2. Identify largest directories/files
3. Determine data ownership
4. Check recent growth
5. Check log retention
6. Check backup/temp files
7. Consult application owner
8. Perform approved cleanup only
```

Không bắt đầu bằng:

```text
Delete *.log
```

vì log có thể là evidence.

---

# 82. Storage evidence

```powershell
Get-Disk |
Out-File C:\Day3Lab\Evidence\15-Disks.txt

Get-Partition |
Out-File C:\Day3Lab\Evidence\16-Partitions.txt

Get-Volume |
Out-File C:\Day3Lab\Evidence\17-Volumes.txt
```

---

# 83. Lab 11 — Windows Update và patch inspection

Assignment yêu cầu inspect updates, không bắt buộc bạn phải install production-style patches.

## Safety

Trong lab có thể inspect freely.

Không install/reboot production chỉ để hoàn thành training.

---

# 84. OS build và patch context

```powershell
Get-ComputerInfo |
Select-Object WindowsProductName,
              WindowsVersion,
              OsBuildNumber
```

Installed hotfix view:

```powershell
Get-HotFix |
Sort-Object InstalledOn -Descending
```

Lưu ý:

`Get-HotFix` không phải universal inventory cho mọi loại modern servicing component, vì vậy đừng xem nó là single source of truth cho entire Windows Update state.

---

# 85. GUI inspection

Desktop Experience:

```text
Settings
→ Windows Update
```

hoặc enterprise management solution nếu organization sử dụng.

Bạn cần ghi nhận:

```text
Update status
Recent successful update
Failed update?
Reboot pending?
Management source nếu biết
Maintenance window
```

---

# 86. Patch verification mindset

Một patch task chỉ complete khi:

```text
Update installed
+
required reboot completed
+
OS healthy
+
critical services healthy
+
application verified
+
monitoring normal
```

Không chỉ dựa vào installer result.

---

# 87. Lab 12 — Event Viewer và Windows Event Logs

Đây là lab troubleshooting/evidence quan trọng nhất.

---

# 88. GUI workflow

Mở:

```text
Event Viewer
```

Đi qua:

```text
Windows Logs
 ├── Application
 ├── Security
 └── System
```

Bạn cần biết cách:

```text
Filter Current Log
Find
Save Filtered Log File
Properties
Event Details
XML view
```

---

# 89. Get-WinEvent

Microsoft `Get-WinEvent` đọc Windows Event Logs và ETL/event tracing sources trên local hoặc remote computers. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.diagnostics/get-winevent?view=powershell-7.6&utm_source=chatgpt.com)

Latest System events:

```powershell
Get-WinEvent `
    -LogName System `
    -MaxEvents 20
```

---

# 90. FilterHashtable

Đây là cách rất hữu ích.

Ví dụ 30 phút gần nhất:

```powershell
$StartTime = (Get-Date).AddMinutes(-30)

Get-WinEvent `
    -FilterHashtable @{
        LogName   = 'System'
        StartTime = $StartTime
    }
```

---

# 91. Error và Critical events

```powershell
Get-WinEvent `
    -FilterHashtable @{
        LogName   = 'System'
        Level     = 1,2
        StartTime = (Get-Date).AddHours(-2)
    } |
Select-Object TimeCreated,
              Id,
              ProviderName,
              LevelDisplayName,
              Message
```

Trong Event Log level:

```text
1 = Critical
2 = Error
```

Nhưng nhớ:

Event severity không bằng business incident severity.

---

# 92. Provider và Event ID

Evidence phải capture cả:

```text
ProviderName
Event ID
```

Ví dụ:

```powershell
Get-WinEvent `
    -LogName System `
    -MaxEvents 50 |
Select-Object TimeCreated,
              ProviderName,
              Id,
              LevelDisplayName,
              Message
```

Không ghi ticket:

```text
“Event ID 1000 found.”
```

mà không ghi Provider.

---

# 93. Export events

Có thể export textual evidence:

```powershell
Get-WinEvent `
    -FilterHashtable @{
        LogName   = 'System'
        StartTime = (Get-Date).AddHours(-1)
    } |
Select-Object TimeCreated,
              ProviderName,
              Id,
              LevelDisplayName,
              Message |
Export-Csv `
    C:\Day3Lab\Evidence\18-SystemEvents.csv `
    -NoTypeInformation `
    -Encoding UTF8
```

Đây là reviewer-friendly output.

---

# 94. Native `.evtx` evidence

Trong actual incident, `.evtx` export có thể bảo tồn event structure tốt hơn screenshot/text.

Có thể dùng Event Viewer GUI để:

```text
Save All Events As...
```

hoặc `wevtutil` trong approved workflow.

Day 3 cần hiểu:

```text
Screenshot
≠
Full event evidence
```

Screenshot hữu ích cho visual proof, nhưng machine-readable/exported log tốt hơn cho deeper investigation.

---

# 95. Lab 13 — Scheduled Tasks inspection

## Objective

Inspect existing tasks rồi tạo scheduled health check.

---

# 96. Inspect tasks

```powershell
Get-ScheduledTask
```

Microsoft `Get-ScheduledTask` trả task definition objects. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/get-scheduledtask?view=windowsserver2025-ps&utm_source=chatgpt.com)

Filter:

```powershell
Get-ScheduledTask |
Where-Object State -ne 'Disabled' |
Select-Object TaskPath, TaskName, State
```

---

# 97. Runtime info

```powershell
Get-ScheduledTaskInfo `
    -TaskName '<task-name>'
```

Microsoft `Get-ScheduledTaskInfo` trả last runtime information của scheduled task. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/get-scheduledtaskinfo?view=windowsserver2025-ps&utm_source=chatgpt.com)

Quan sát:

```text
LastRunTime
LastTaskResult
NextRunTime
NumberOfMissedRuns
```

---

# 98. Inspect action

```powershell
$Task = Get-ScheduledTask -TaskName '<task-name>'

$Task.Actions
```

Inspect trigger:

```powershell
$Task.Triggers
```

Principal:

```powershell
$Task.Principal
```

Operationally rất quan trọng.

Một task:

```text
Ready
```

chưa chứng minh last execution successful.

---

# 99. Lab 14 — Tạo PowerShell health-check script

Syllabus yêu cầu **scheduled health-check script**.

Script này sẽ chỉ thực hiện read-only checks.

Tạo:

```text
C:\Day3Lab\Scripts\Day3-HealthCheck.ps1
```

Nội dung:

```powershell
$Timestamp = Get-Date
$Computer  = $env:COMPUTERNAME
$OutputDir = 'C:\Day3Lab\Reports'

if (-not (Test-Path $OutputDir)) {
    New-Item -ItemType Directory -Path $OutputDir -Force |
        Out-Null
}

$ReportFile = Join-Path `
    $OutputDir `
    ("HealthCheck_{0}_{1}.txt" -f `
        $Computer,
        $Timestamp.ToString('yyyyMMdd_HHmmss'))

"Day 3 Windows Server Health Check" |
    Out-File $ReportFile

"Timestamp: $($Timestamp.ToString('o'))" |
    Out-File $ReportFile -Append

"Computer: $Computer" |
    Out-File $ReportFile -Append

"`r`n=== OS ===" |
    Out-File $ReportFile -Append

Get-ComputerInfo |
Select-Object `
    WindowsProductName,
    WindowsVersion,
    OsBuildNumber |
Format-List |
Out-File $ReportFile -Append

"`r`n=== Critical Services ===" |
    Out-File $ReportFile -Append

Get-Service |
Where-Object {
    $_.StartType -eq 'Automatic'
} |
Select-Object Name, Status, StartType |
Format-Table -AutoSize |
Out-File $ReportFile -Append

"`r`n=== Network ===" |
    Out-File $ReportFile -Append

Get-NetIPConfiguration |
Format-List InterfaceAlias,
            IPv4Address,
            IPv4DefaultGateway,
            DNSServer |
Out-File $ReportFile -Append

"`r`n=== Volumes ===" |
    Out-File $ReportFile -Append

Get-Volume |
Where-Object DriveLetter |
Select-Object `
    DriveLetter,
    FileSystem,
    HealthStatus,
    @{Name='SizeGB';Expression={
        [math]::Round($_.Size / 1GB, 2)
    }},
    @{Name='FreeGB';Expression={
        [math]::Round($_.SizeRemaining / 1GB, 2)
    }},
    @{Name='FreePercent';Expression={
        if ($_.Size -gt 0) {
            [math]::Round(
                ($_.SizeRemaining / $_.Size) * 100,
                2
            )
        }
    }} |
Format-Table -AutoSize |
Out-File $ReportFile -Append

"`r`n=== Recent System Errors ===" |
    Out-File $ReportFile -Append

$StartTime = (Get-Date).AddHours(-1)

Get-WinEvent `
    -FilterHashtable @{
        LogName   = 'System'
        Level     = 1,2
        StartTime = $StartTime
    } `
    -ErrorAction SilentlyContinue |
Select-Object `
    TimeCreated,
    Id,
    ProviderName,
    LevelDisplayName |
Format-Table -AutoSize |
Out-File $ReportFile -Append

"`r`n=== Completed ===" |
    Out-File $ReportFile -Append

"Health check completed successfully."

exit 0
```

---

# 100. Hiểu script thay vì copy

Phần đầu:

```powershell
$Timestamp = Get-Date
```

capture execution time.

```powershell
$env:COMPUTERNAME
```

capture hostname.

Timestamp dùng ISO-like format trong report:

```powershell
$Timestamp.ToString('o')
```

rất hữu ích cho evidence.

---

# 101. Output directory validation

```powershell
if (-not (Test-Path $OutputDir)) {
```

kiểm tra folder tồn tại.

Nếu chưa có:

```powershell
New-Item ...
```

Đây là ví dụ đơn giản của defensive scripting.

---

# 102. Automatic services check

```powershell
Get-Service |
Where-Object {
    $_.StartType -eq 'Automatic'
}
```

Mục tiêu là phát hiện automatic service đang stopped.

Nhưng lưu ý:

Không phải mọi stopped automatic service chắc chắn là incident.

Một số service behavior phụ thuộc trigger/service architecture.

Health-check chỉ là **signal**, không thay root-cause analysis.

---

# 103. Storage check

Script tính:

```text
SizeGB
FreeGB
FreePercent
```

Tốt hơn chỉ nhìn:

```text
FreeGB
```

vì:

```text
10 GB free trên 20 GB volume
```

khác rất nhiều:

```text
10 GB free trên 4 TB volume
```

---

# 104. Recent errors

Script xem Critical/Error trong System log một giờ gần nhất.

Một health check mature hơn sẽ filter theo known providers/services để giảm noise, nhưng Day 3 chỉ cần foundation.

---

# 105. Manual script test

Trước khi schedule:

```powershell
& 'C:\Day3Lab\Scripts\Day3-HealthCheck.ps1'
```

Expected:

```text
Health check completed successfully.
```

Check:

```powershell
Get-ChildItem C:\Day3Lab\Reports
```

Mở latest report:

```powershell
Get-Content `
    (Get-ChildItem C:\Day3Lab\Reports |
     Sort-Object LastWriteTime -Descending |
     Select-Object -First 1).FullName
```

---

# 106. Execution Policy check

Trước khi thay execution policy:

```powershell
Get-ExecutionPolicy -List
```

Nếu script bị block, bạn phải xác định:

```text
MachinePolicy?
UserPolicy?
CurrentUser?
LocalMachine?
Downloaded-file Mark-of-the-Web?
```

Không chạy reflexively:

```powershell
Set-ExecutionPolicy Bypass -Scope LocalMachine
```

Trong enterprise, execution policy có thể bị GPO quản lý.

---

# 107. Tạo scheduled task

Một safe pattern trong lab là dùng built-in identity thích hợp hoặc dedicated controlled account tùy yêu cầu.

Ví dụ daily health check:

```powershell
$Action = New-ScheduledTaskAction `
    -Execute 'powershell.exe' `
    -Argument "-NoProfile -File `"C:\Day3Lab\Scripts\Day3-HealthCheck.ps1`""

$Trigger = New-ScheduledTaskTrigger `
    -Daily `
    -At 9:00AM
```

Register:

```powershell
Register-ScheduledTask `
    -TaskName 'Day3-HealthCheck' `
    -Action $Action `
    -Trigger $Trigger `
    -Description 'Day 3 Windows Server health check lab' `
    -User 'SYSTEM'
```

`Register-ScheduledTask` đăng ký scheduled task definition trên local computer. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/scheduledtasks/register-scheduledtask?view=windowsserver2025-ps&utm_source=chatgpt.com)

### Security note

`SYSTEM` có privilege rất lớn.

Trong real enterprise:

> Không chọn SYSTEM chỉ vì “dễ chạy”.

Identity phải phù hợp với least privilege và organizational design.

Ở controlled lab, nó giúp tránh lưu password.

---

# 108. Verify scheduled task

```powershell
Get-ScheduledTask `
    -TaskName 'Day3-HealthCheck'
```

Inspect:

```powershell
Get-ScheduledTask `
    -TaskName 'Day3-HealthCheck' |
Select-Object TaskName, State
```

---

# 109. Manual task run

```powershell
Start-ScheduledTask `
    -TaskName 'Day3-HealthCheck'
```

Sau một lúc:

```powershell
Get-ScheduledTaskInfo `
    -TaskName 'Day3-HealthCheck'
```

Quan sát:

```text
LastRunTime
LastTaskResult
NextRunTime
```

Check report:

```powershell
Get-ChildItem C:\Day3Lab\Reports |
Sort-Object LastWriteTime -Descending
```

---

# 110. Scheduled Task verification không dừng ở State

Bạn cần chứng minh:

```text
Task exists
+
Trigger correct
+
Action correct
+
Principal correct
+
Task executed
+
Output report created
+
Report contains expected content
```

Đó mới là full verification.

---

# 111. Scheduled Task rollback

```powershell
Unregister-ScheduledTask `
    -TaskName 'Day3-HealthCheck'
```

Xác nhận removal:

```powershell
Get-ScheduledTask `
    -TaskName 'Day3-HealthCheck' `
    -ErrorAction SilentlyContinue
```

---

# 112. Lab 15 — Active Directory/domain inspection

Phần này chỉ thực hiện nếu domain environment available, đúng như syllabus quy định. Infrastructure_Operations_14_Da…

Không tạo forest/domain mới cho Day 3 nếu environment không có.

---

# 113. Domain membership

```powershell
Get-ComputerInfo |
Select-Object CsDomain, CsDomainRole
```

Hoặc:

```powershell
(Get-CimInstance Win32_ComputerSystem).Domain
```

Bạn cần xác định:

```text
Workgroup
vs
Domain-joined
```

---

# 114. Current identity

```powershell
whoami
```

Groups:

```cmd
whoami /groups
```

Bạn phải phân biệt:

```text
SRV-D3-01\user
```

và:

```text
CORP\user
```

---

# 115. Domain Controller discovery awareness

Nếu appropriate:

```cmd
nltest /dsgetdc:<domain-name>
```

Ví dụ:

```cmd
nltest /dsgetdc:corp.example.com
```

Mục tiêu là inspection.

Không thay đổi DC configuration.

---

# 116. Group Policy inspection

Read-only:

```cmd
gpresult /r
```

Detailed report:

```cmd
gpresult /h C:\Day3Lab\Evidence\GPResult.html
```

Sau đó review:

```text
Applied Group Policy Objects
Computer settings
User settings
Security filtering
```

---

# 117. GPO troubleshooting principle

Nếu local setting khác expected:

```text
1. Read local configuration
2. Read effective configuration
3. Check gpresult
4. Identify applicable GPO
5. Record evidence
6. Escalate to AD/GPO owner if change required
```

Không mở `Group Policy Management Console` và edit production GPO ngoài scope.

---

# 118. Lab 16 — Backup/recovery awareness

Day 3 chỉ yêu cầu foundation; Day 5/Day 7 sẽ đi sâu backup/DR.

Trong lab này hãy backup controlled file/folder.

---

# 119. Controlled backup set

Tạo:

```text
C:\Day3Lab\Data\BackupDemo
```

```powershell
New-Item `
    -ItemType Directory `
    -Path C:\Day3Lab\Data\BackupDemo `
    -Force

'Version 1 - Day 3 backup test' |
Set-Content C:\Day3Lab\Data\BackupDemo\important.txt
```

---

# 120. Hash trước backup

```powershell
Get-FileHash `
    C:\Day3Lab\Data\BackupDemo\important.txt `
    -Algorithm SHA256
```

Lưu:

```powershell
Get-FileHash `
    C:\Day3Lab\Data\BackupDemo\important.txt `
    -Algorithm SHA256 |
Export-Csv `
    C:\Day3Lab\Evidence\19-OriginalHash.csv `
    -NoTypeInformation
```

Hash giúp verify integrity sau restore.

---

# 121. Backup controlled file

Day 3 không quy định backup product.

Có thể dùng simple copy cho conceptual test:

```powershell
New-Item `
    -ItemType Directory `
    -Path C:\Day3Lab\Backup `
    -Force

Copy-Item `
    C:\Day3Lab\Data\BackupDemo\important.txt `
    C:\Day3Lab\Backup\
```

Trong production, đây **không phải substitute cho enterprise backup solution**.

---

# 122. Controlled restore test

Để không phá original, restore sang isolated folder:

```powershell
New-Item `
    -ItemType Directory `
    -Path C:\Day3Lab\Restore `
    -Force

Copy-Item `
    C:\Day3Lab\Backup\important.txt `
    C:\Day3Lab\Restore\
```

---

# 123. Verify restored integrity

```powershell
Get-FileHash `
    C:\Day3Lab\Restore\important.txt `
    -Algorithm SHA256
```

So sánh hash.

```powershell
$Original = (
    Get-FileHash `
    C:\Day3Lab\Data\BackupDemo\important.txt `
    -Algorithm SHA256
).Hash

$Restored = (
    Get-FileHash `
    C:\Day3Lab\Restore\important.txt `
    -Algorithm SHA256
).Hash

$Original -eq $Restored
```

Expected:

```text
True
```

Đây là proof tốt hơn:

> “Tôi thấy file nằm đó.”

---

# 124. Failure Injection 1 — Controlled service failure

Đây là phần quan trọng nhất để chuyển administration sang troubleshooting.

Chỉ dùng safe service/lab application.

---

# 125. Baseline trước failure

```powershell
Get-Service -Name '<LabService>'
```

Capture:

```powershell
Get-Date
```

Evidence:

```text
Service = Running
Application test = Pass
Time = ...
```

---

# 126. Inject service failure

**CONFIG CHANGE — LAB ONLY**

```powershell
Stop-Service -Name '<LabService>'
```

Không chọn các critical services như:

```text
LSASS-related components
RPC core services
Domain Controller critical services
Hyper-V cluster services
```

nếu không có dedicated simulation.

---

# 127. Observe symptom

Đầu tiên kiểm tra user/application symptom.

Ví dụ:

```powershell
Test-NetConnection `
    localhost `
    -Port <LabPort>
```

Có thể fail nếu application listener biến mất.

---

# 128. Diagnose service failure

```powershell
Get-Service -Name '<LabService>'
```

Expected:

```text
Stopped
```

Inspect configuration:

```powershell
Get-CimInstance Win32_Service `
    -Filter "Name='<LabService>'"
```

Check events:

```powershell
Get-WinEvent `
    -FilterHashtable @{
        LogName   = 'System'
        StartTime = (Get-Date).AddMinutes(-15)
    } |
Select-Object TimeCreated,
              ProviderName,
              Id,
              LevelDisplayName,
              Message
```

---

# 129. Restore service

```powershell
Start-Service -Name '<LabService>'
```

Verify:

```powershell
Get-Service -Name '<LabService>'
```

Then application-level check:

```powershell
Test-NetConnection `
    localhost `
    -Port <LabPort>
```

Nếu applicable.

---

# 130. Failure Injection 1 — Lessons

Bạn phải có thể nói:

```text
Symptom:
Application unreachable

Observation:
Service stopped

Evidence:
Service state + timestamp + related logs

Recovery:
Service started

Verification:
Service running + port/application test passes

Root cause:
Controlled training stop, not natural failure

Rollback:
Restored original Running state
```

Điểm quan trọng:

Không được viết:

> “Root cause: service stopped.”

`Stopped` là state/symptom.

Trong controlled failure, root cause là deliberate administrative action.

Trong real incident, bạn còn phải tìm **vì sao service stopped**.

---

# 131. Failure Injection 2 — NTFS permission failure

Scenario:

`lab_user01` trước đó Modify được `Finance`.

Ta deliberate remove permission để tạo failure.

---

# 132. Baseline

```powershell
(Get-Acl C:\Day3Lab\Data\Finance).Access
```

Save:

```powershell
(Get-Acl C:\Day3Lab\Data\Finance).Access |
Export-Csv `
    C:\Day3Lab\Evidence\20-ACL-BeforeFailure.csv `
    -NoTypeInformation
```

---

# 133. Inject permission failure

**CONFIG CHANGE — LAB ONLY**

Ví dụ remove controlled group ACE:

```cmd
icacls C:\Day3Lab\Data\Finance /remove LAB_File_Modify
```

Verify:

```cmd
icacls C:\Day3Lab\Data\Finance
```

---

# 134. Reproduce symptom

Dùng actual test identity.

Thử:

```text
Create file
Modify file
```

Expected:

```text
Access denied
```

Nếu vẫn có access, kiểm tra:

```text
Another group gives permission?
Administrator?
Inherited permission?
Existing handle/session?
```

Đây là chính lý do phải test effective access.

---

# 135. Diagnose permission failure

Sequence:

```text
Identity
 ↓
whoami
 ↓
Group membership
 ↓
Get-LocalGroupMember
 ↓
NTFS ACL
 ↓
Get-Acl / icacls
 ↓
Share ACL nếu qua UNC
 ↓
Get-SmbShareAccess
 ↓
Inheritance/Deny
 ↓
Effective result
```

---

# 136. Restore permission

```cmd
icacls C:\Day3Lab\Data\Finance /grant LAB_File_Modify:(OI)(CI)M
```

Verify ACL:

```powershell
(Get-Acl C:\Day3Lab\Data\Finance).Access
```

Retest actual user.

Expected:

```text
Create/modify succeeds
```

---

# 137. Failure Injection 3 — Firewall failure

Đây là optional nhưng rất hữu ích.

Prerequisite:

Một lab service đang listen trên test port.

Baseline:

```powershell
Test-NetConnection SRV-D3-01 -Port <LabPort>
```

Expected:

```text
True
```

Inject một narrow lab block rule, chỉ áp dụng đúng test port/source cần thiết.

Không block RDP/WinRM connection bạn đang dùng để quản trị server nếu không có console fallback.

---

# 138. Golden rule với firewall failure injection

Trước khi firewall experiment qua remote connection:

```text
Have console access
or
Have out-of-band access
```

Nếu không, bạn có thể tự lock yourself out.

Đây là một ví dụ quan trọng của:

> rollback capability phải tồn tại trước change.

---

# 139. Failure Injection 4 — Scheduled task failure

Controlled scenario:

Tạo task với invalid script path:

```text
C:\Day3Lab\Scripts\DoesNotExist.ps1
```

Sau đó manual run task.

Observe:

```powershell
Get-ScheduledTaskInfo `
    -TaskName '<FailureTask>'
```

Điều tra:

```text
Task definition
Action path
Principal
History
Output absent
```

Fix path và rerun.

Điểm học:

> Task = Ready không có nghĩa action thực thi thành công.

---

# 140. Failure Injection 5 — DNS/name-resolution failure concept

Chỉ thực hiện nếu lab architecture cho phép an toàn.

Không nên thay DNS server của domain member nếu có nguy cơ phá toàn domain connectivity.

Một safer simulation có thể là lookup nonexistent hostname:

```powershell
Resolve-DnsName definitely-not-a-real-day3-host.lab `
    -ErrorAction SilentlyContinue
```

Sau đó so sánh với valid target.

Mục tiêu là phân biệt:

```text
Name resolution failure
```

khỏi:

```text
IP routing failure
```

---

# 141. Evidence extraction sau Failure Injection

Syllabus explicitly yêu cầu extract relevant event records. Infrastructure_Operations_14_Da…

Không phải failure nào cũng sinh event đúng như bạn mong đợi.

Nếu controlled NTFS denial không có Security event:

> Không bịa event evidence.

Security audit event phụ thuộc audit policy/SACL.

Bạn có thể ghi:

```text
No relevant Security audit event observed because required auditing was not configured.
```

Đó là chính xác hơn.

---

# 142. Evidence fields chuẩn

Mỗi event evidence nên có:

```text
Hostname
Log Name
Timestamp
Timezone
Provider
Event ID
Level
Message
Related test
Extraction time
Ticket/change reference
```

---

# 143. PowerShell evidence extraction mẫu

```powershell
$Start = (Get-Date).AddMinutes(-30)

Get-WinEvent `
    -FilterHashtable @{
        LogName   = 'System'
        StartTime = $Start
    } |
Select-Object `
    MachineName,
    TimeCreated,
    ProviderName,
    Id,
    LevelDisplayName,
    Message |
Export-Csv `
    C:\Day3Lab\Evidence\21-FailureEvents.csv `
    -NoTypeInformation `
    -Encoding UTF8
```

---

# 144. Tạo “before vs after evidence”

Một pattern tốt:

```text
Before
↓
Change
↓
After
```

Ví dụ service:

```text
service_before.txt
change_timestamp.txt
service_after.txt
events_after.csv
application_test_after.txt
```

Nhờ vậy reviewer có thể xác minh chính xác change.

---

# 145. Không thu secrets trong evidence

Trước khi attach evidence vào ticket:

Check xem có:

```text
Password
Credential
Token
Private key
Sensitive connection string
PII
Secret
```

không.

Nếu có, redact/mask đúng procedure.

Không screenshot password prompt với credential.

---

# 146. Operational troubleshooting checkpoints

Khi gặp bất kỳ lỗi nào trong Day 3 lab, sử dụng sequence này.

## Checkpoint 1 — Reproduce

Bạn có tái tạo được symptom không?

Nếu không:

```text
Intermittent?
Already recovered?
Wrong user?
Wrong server?
Wrong time?
```

---

# 147. Checkpoint 2 — Scope

Một user?

Một machine?

Một subnet?

Một service?

Toàn server?

Toàn domain?

Scope là một trong những clues mạnh nhất.

---

# 148. Checkpoint 3 — Recent change

Kiểm tra:

```text
Patch
Reboot
GPO
Permission
Deployment
Firewall
Service account
Network
Storage
Scheduled task
```

gần thời điểm incident.

---

# 149. Checkpoint 4 — Read-only state

Thu thập trước:

```powershell
Get-Service
Get-Process
Get-NetIPConfiguration
Get-NetRoute
Get-NetFirewallRule
Get-Volume
Get-WinEvent
```

Không thay đổi ngay.

---

# 150. Checkpoint 5 — Dependency

Ví dụ:

```text
Application
├── Service
├── DNS
├── Network
├── Firewall
├── Storage
├── AD
└── Backend database
```

Root cause có thể không nằm trên server đang báo lỗi.

---

# 151. Checkpoint 6 — Smallest safe action

Ưu tiên:

```text
Start one failed service
```

thay vì:

```text
Reboot server
```

nếu root cause/scope cho phép.

Ưu tiên:

```text
Fix one ACL
```

thay vì:

```text
Everyone Full Control
```

---

# 152. Checkpoint 7 — Technical verification

Sau fix:

```text
Service running?
Port listening?
DNS works?
ACL correct?
Task ran?
Disk healthy?
```

---

# 153. Checkpoint 8 — Business/application verification

Không dừng ở technical layer.

Ví dụ:

```text
Get-Service says Running
```

nhưng user vẫn không access application.

Thì incident chưa resolved.

---

# 154. Checkpoint 9 — Monitor

Sau recovery:

```text
Does service remain stable?
Does disk continue filling?
Do new errors appear?
Does scheduled task repeat successfully?
```

Recovery tức thời chưa đủ chứng minh stable.

---

# 155. Checkpoint 10 — Evidence và closure

Ticket phải nói:

```text
Symptom
Scope
Cause/probable cause
Checks performed
Change performed
Verification
Evidence
Outstanding issue
Rollback status
Escalation if any
```

---

# 156. Escalation boundaries trong Day 3 lab

Bạn nên escalate nếu phát hiện tình huống tương tự:

| Situation                                           | Typical action                              |
| --------------------------------------------------- | ------------------------------------------- |
| Unknown Domain GPO enforcing configuration          | AD/GPO team                                 |
| Domain Controller issue                             | AD specialist                               |
| Repeated service crash with application stack trace | Application/vendor team                     |
| Disk/file-system corruption                         | Storage/OS specialist                       |
| Suspected security compromise                       | Security/IR                                 |
| Unauthorized Administrator account                  | Security + IAM                              |
| Patch causes production application failure         | Change/Application/Microsoft/vendor support |
| RDP/WinRM blocked by enterprise security policy     | Security/network/endpoint management        |
| OS unsupported/legacy                               | Platform owner/security/risk management     |
| Production restoration required                     | Backup/application/business owner           |

L1/L2 boundary luôn phụ thuộc organizational authorization.

---

# 157. Rollback matrix cho toàn module

| Change                 | Rollback                                   |
| ---------------------- | ------------------------------------------ |
| Local user created     | `Remove-LocalUser`                         |
| Local group created    | `Remove-LocalGroup`                        |
| Group membership       | `Remove-LocalGroupMember`                  |
| NTFS ACE added         | Remove specific ACE / restore saved ACL    |
| SMB share created      | `Remove-SmbShare`                          |
| RDP group membership   | Remove test user                           |
| Firewall rule created  | `Remove-NetFirewallRule`                   |
| WinRM reconfigured     | Restore approved previous configuration    |
| Service changed        | Restore original startup/state             |
| Scheduled task created | `Unregister-ScheduledTask`                 |
| Test files/folders     | Delete only controlled lab artifacts       |
| Backup test            | Remove test backup after evidence complete |

Một rollback tốt phải biết **original state**.

Do đó baseline không phải paperwork; baseline là prerequisite của rollback.

---

# 158. Cleanup không phải `Remove-Item C:\Day3Lab -Recurse -Force` ngay lập tức

Nếu xóa toàn bộ lab folder trước khi nộp evidence, bạn có thể mất:

```text
Reports
Logs
Hashes
Screenshots
Scripts
Before/after configuration
```

Workflow đúng:

```text
Complete lab
↓
Verify evidence
↓
Copy evidence to approved repository
↓
Verify copied evidence
↓
Perform cleanup
↓
Verify environment returned to expected state
```

---

# 159. Final Windows Server configuration checklist

Đây là deliverable trực tiếp từ syllabus.

Bạn nên có report tương tự:

| Category        | Evidence cần có                  |
| --------------- | -------------------------------- |
| Server identity | Hostname                         |
| Version         | Windows Server generation        |
| Edition         | Standard/Datacenter/etc.         |
| Build           | OS build                         |
| Installation    | Core/Desktop Experience          |
| Roles/Features  | Installed list                   |
| Domain          | Domain/workgroup state           |
| Local users     | Controlled account inventory     |
| Local groups    | Relevant memberships             |
| Privilege       | Administrators membership        |
| NTFS            | ACL verification                 |
| SMB             | Share + share ACL                |
| RDP             | Access status/authorization      |
| WinRM           | Service/Test-WSMan/remoting      |
| Firewall        | Profiles/rules/test              |
| Services        | Critical state/startup           |
| Processes       | Resource/PID inspection          |
| Networking      | NIC/IP/gateway/DNS/routes        |
| Storage         | Disk/partition/volume/free space |
| Updates         | Patch/update status              |
| Events          | Relevant exported records        |
| Scheduled Tasks | Task definition/runtime          |
| Health script   | Script + generated report        |
| AD/GPO          | `gpresult` if available          |
| Backup          | Controlled restore validation    |
| Failure tests   | Service/permission etc.          |
| Rollback        | Final restored state             |
| Escalation      | Any unresolved findings          |

---

# 160. Evidence package đề xuất

Một hoàn chỉnh `Evidence` directory có thể trông như:

```text
C:\Day3Lab\Evidence\
│
├── 01-ComputerInfo.txt
├── 02-InstalledFeatures.txt
├── 03-Services-Before.txt
├── 04-Network-Before.txt
├── 05-Volumes-Before.txt
├── 06-FirewallProfiles-Before.txt
├── 07-LocalUsers-Before.txt
├── 08-LabGroupMembership.txt
├── 09-NTFS-ACL.txt
├── 10-Service-Before.txt
├── 11-Service-After.txt
├── 12-Network.txt
├── 13-Routes.txt
├── 14-DNS-Client.txt
├── 15-Disks.txt
├── 16-Partitions.txt
├── 17-Volumes.txt
├── 18-SystemEvents.csv
├── 19-OriginalHash.csv
├── 20-ACL-BeforeFailure.csv
├── 21-FailureEvents.csv
├── GPResult.html
└── Final-Day3-Checklist.txt
```

Không phải syllabus bắt buộc chính xác filename này; đây là **kiến thức bổ sung về evidence organization**.

---

# 161. Final health-check command set

Đây là nhóm read-only commands bạn nên có khả năng chạy mà không cần tra lại từng cái:

```powershell
hostname

Get-ComputerInfo

Get-WindowsFeature |
Where-Object InstallState -eq 'Installed'

Get-LocalUser

Get-LocalGroup

Get-LocalGroupMember -Group Administrators

Get-Service

Get-Process

Get-NetAdapter

Get-NetIPConfiguration

Get-NetIPAddress

Get-NetRoute

Get-DnsClientServerAddress

Resolve-DnsName <hostname>

Test-NetConnection <hostname> -Port <port>

Get-NetFirewallProfile

Get-NetFirewallRule

Get-Disk

Get-Partition

Get-Volume

Get-ScheduledTask

Get-ScheduledTaskInfo -TaskName '<task>'

Get-WinEvent -LogName System -MaxEvents 20

Get-ExecutionPolicy -List

Test-WSMan <server>

Get-SmbShare

Get-SmbShareAccess -Name '<share>'

Get-Acl '<path>'

gpresult /r
```

Những cmdlet về SMB, firewall, WinRM, storage, network, Scheduled Tasks và Event Logs ở trên đều tương ứng với các management modules được Microsoft document cho Windows/Windows Server hiện hành. [Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/smbshare/new-smbshare?view=windowsserver2025-ps&utm_source=chatgpt.com)

---

# 162. Các command bạn phải nhận biết là configuration-changing

```powershell
New-LocalUser
New-LocalGroup
Add-LocalGroupMember

New-SmbShare
Grant-SmbShareAccess

New-NetFirewallRule
Remove-NetFirewallRule

Enable-PSRemoting

Start-Service
Stop-Service
Restart-Service

Register-ScheduledTask
Unregister-ScheduledTask
```

Nhìn thấy các verbs:

```text
New
Set
Add
Remove
Enable
Disable
Start
Stop
Restart
Register
Unregister
```

bạn nên mặc định nghĩ:

> “Command này có thể thay đổi state. Tôi đã kiểm tra authorization và rollback chưa?”

---

# 163. Day 3 practical troubleshooting flow tổng hợp

Khi server/application gặp lỗi:

```text
             INCIDENT
                 │
                 ▼
        Identify exact symptom
                 │
                 ▼
          Determine scope
                 │
                 ▼
         Check recent change
                 │
                 ▼
        Collect baseline/state
                 │
      ┌──────────┼───────────┐
      ▼          ▼           ▼
   Service     Network     Identity
      │          │           │
      ▼          ▼           ▼
   Process      DNS       Permissions
      │          │           │
      └──────┬───┴─────┬─────┘
             ▼         ▼
          Storage    Firewall
             │         │
             └────┬────┘
                  ▼
               Events
                  │
                  ▼
          Probable root cause
                  │
                  ▼
        Authorized safe action
                  │
                  ▼
        Technical verification
                  │
                  ▼
        Application verification
                  │
                  ▼
             Monitoring
                  │
                  ▼
             Evidence
                  │
          ┌───────┴────────┐
          ▼                ▼
       Resolve          Escalate
```

---

# 164. Một scenario tổng hợp cuối lab

Giả sử ticket:

> Finance user không thể truy cập `\\SRV-D3-01\Day3Data` sau maintenance.

Bạn phải điều tra theo thứ tự.

### 1. Verify server

```powershell
Test-NetConnection SRV-D3-01
```

### 2. Verify DNS

```powershell
Resolve-DnsName SRV-D3-01
```

### 3. Verify SMB

```powershell
Test-NetConnection SRV-D3-01 -Port 445
```

### 4. Verify share

```powershell
Get-SmbShare -Name Day3Data
```

### 5. Verify share ACL

```powershell
Get-SmbShareAccess -Name Day3Data
```

### 6. Verify NTFS ACL

```powershell
Get-Acl C:\Day3Lab\Data\Finance
```

### 7. Verify identity

```powershell
Get-LocalGroupMember `
    -Group LAB_File_Modify
```

### 8. Verify related events

```powershell
Get-WinEvent `
    -FilterHashtable @{
        LogName   = 'System'
        StartTime = (Get-Date).AddMinutes(-30)
    }
```

### 9. Compare recent change

```text
Was ACL changed?
Share recreated?
Firewall changed?
User removed from group?
```

### 10. Apply smallest corrective action

Ví dụ user bị remove khỏi access group:

```powershell
Add-LocalGroupMember `
    -Group LAB_File_Modify `
    -Member lab_user01
```

### 11. Retest actual user

Không chỉ check membership.

### 12. Record evidence

```text
Before state
Change
After state
User verification
Timestamp
```

Đây là một **Infrastructure Operations workflow**, không phải chỉ là Windows command exercise.

---

# 165. Những lỗi fresher thường mắc trong Module Thực hành

### Chỉ làm GUI, không biết command

Khi Server Core hoặc remote automation xuất hiện sẽ bị kẹt.

### Chỉ dùng PowerShell, không hiểu object/state

Copy command mà không biết command đang thay đổi gì.

### Không baseline

Không biết original configuration để rollback.

### Test bằng Administrator

Permission test mất ý nghĩa.

### Chỉ nhìn ACL

Không test actual effective access.

### “Port open” = application healthy

Sai.

### “Service running” = application healthy

Sai.

### Restart trước khi collect logs

Mất transient evidence.

### Disable firewall để troubleshoot

Quá rộng và không an toàn.

### Add Administrators để sửa permission issue

Vi phạm least privilege.

### Hard-code passwords trong script

Credential exposure.

### Backup nhưng không restore test

Chưa chứng minh recovery.

### Chụp screenshot thay vì export logs

Evidence nghèo, khó query và correlate.

---

# 166. Day 3 Module Thực hành — Completion Checklist

Khi kết thúc, bạn phải thực hiện hoặc hiểu đầy đủ các hạng mục sau theo Assignment của syllabus. Infrastructure_Operations_14_Da…

### Environment

- Windows Server VM đã được provision/approved.
- Hostname được xác minh.
- Version được xác minh.
- Edition được xác minh.
- Installation mode được xác minh.
- Installed roles/features được inventory.

### Identity

- Local user được tạo có kiểm soát.
- Local group được tạo.
- Group membership được xác minh.
- Least privilege được chứng minh.
- Local Administrators được inspect.

### Permissions

- NTFS ACL được inspect.
- Controlled NTFS permission được cấu hình.
- Actual user access được test.
- SMB share được tạo.
- Share ACL được kiểm tra.
- Share + NTFS effective behavior được hiểu.

### Secure administration

- RDP status được inspect.
- WinRM được inspect.
- `Test-WSMan` được sử dụng.
- PowerShell remoting được kiểm tra nếu available.
- Authorization được phân biệt với network connectivity.

### Firewall/network

- Firewall profiles được inspect.
- Firewall rules được inspect.
- Controlled rule được tạo/test/rollback.
- NIC được inspect.
- IP/subnet được inspect.
- Default route được inspect.
- DNS client được inspect.
- DNS resolution được test.
- TCP connectivity được test.

### Services/processes

- Services được inventory.
- Startup type/account/dependencies được hiểu.
- Authorized service restart được thực hiện.
- Before/after evidence được lưu.
- Process/PID/resource usage được inspect.
- Application verification được thực hiện sau service action.

### Storage

- Disks được inspect.
- Partitions được inspect.
- Volumes được inspect.
- Filesystem/capacity/free space được xác minh.
- Low-capacity troubleshooting process được hiểu.

### Updates

- OS build được ghi nhận.
- Installed updates/hotfix information được inspect.
- Reboot/post-patch verification concept được hiểu.
- Không thực hiện unauthorized patching.

### Event Logs

- Event Viewer được sử dụng.
- System/Application/Security logs được hiểu.
- `Get-WinEvent` được sử dụng.
- Provider + Event ID + timestamp được capture.
- Relevant events được export.

### Task Scheduler

- Existing tasks được inspect.
- Task action/trigger/principal được inspect.
- Scheduled health-check được tạo.
- Manual run được thực hiện.
- Last run information được verify.
- Output report được verify.

### PowerShell scripting

- Health-check script chạy thành công.
- OS information được collect.
- Service state được collect.
- Network information được collect.
- Volume/free-space information được collect.
- Recent events được collect.
- Output có timestamp.

### AD/GPO

Nếu available:

- Domain membership được xác định.
- Current identity/domain được xác định.
- `gpresult` được thực hiện.
- Effective GPO được inspect.
- Không unauthorized edit domain policy.

### Backup/recovery

- Controlled data được backup.
- Isolated restore được thực hiện.
- SHA-256 hash được so sánh.
- Recovery được verify.

### Failure Injection

- Controlled service failure được troubleshoot.
- Controlled permission failure được troubleshoot.
- Có thể bổ sung firewall/task failure.
- Root cause và symptom được phân biệt.
- Recovery được verify.

### Evidence

- Before/after configuration có evidence.
- Timestamp được lưu.
- Commands/outputs quan trọng được lưu.
- Event logs được export.
- Không chứa real credentials.
- Final maintenance checklist hoàn chỉnh.

### Cleanup/Rollback

- Temporary firewall rule được remove.
- Test share được remove nếu assignment kết thúc.
- Scheduled test task được remove nếu cần.
- Test accounts/groups được cleanup nếu policy yêu cầu.
- Original service state được restore.
- Evidence đã được preserve trước cleanup.
- Final server health được verify.

---

# 167. Chuẩn đầu ra cuối Day 3

Nếu hoàn thành đúng module này, bạn không chỉ có thể nói:

> “Tôi biết Windows Server.”

Mà phải có khả năng nói chính xác hơn:

> “Tôi có thể baseline một Windows Server, xác định OS/edition/roles, inspect identity và least-privilege membership, cấu hình và troubleshoot NTFS/SMB permissions, verify RDP/WinRM, kiểm tra Windows Firewall và network paths, inspect services/processes/storage/update status, tìm và export Windows Event Logs, xây dựng PowerShell health-check chạy qua Task Scheduler, inspect domain/GPO khi được phép, thực hiện controlled service/permission failure diagnosis, verify backup recovery bằng integrity check, lưu audit evidence, rollback configuration và biết khi nào phải escalate.”

Đó chính là **operational readiness** mà syllabus Day 3 hướng tới; syllabus cũng nhấn mạnh toàn chương trình là accelerated foundation và operational readiness với hands-on lab, verification evidence, controlled troubleshooting, không phải tuyên bố advanced product mastery. Infrastructure_Operations_14_Da…
