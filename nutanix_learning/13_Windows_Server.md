> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# Windows Server – nền tảng vận hành cho mọi edition/version

## 1. Liên hệ với dự án
**[Nguồn dự án]** Windows xuất hiện rất rộng trong SOW như OS/tool workstation/server context, nhưng nguồn không chỉ rõ Windows Server version/edition của từng hệ thống. Vì vậy tài liệu này dạy nền tảng chung cho Windows Server family và chỉ ra điểm khác nhau cần chú ý.

## 2. Edition/version mental map
Các dòng hiện đại thường gặp:
- Standard.
- Datacenter.
- Datacenter: Azure Edition ở các scenario phù hợp.
- Essentials tồn tại ở một số thế hệ/SMB scenario.
- Server Core vs Desktop Experience là **installation option**, không phải edition.
- Các generation cũ còn có Foundation/Storage Server/OEM variations.

**Không học theo tên edition trước; học theo role/workload.** Edition chủ yếu quyết định licensing/virtualization/features; cách troubleshoot service/network/storage/event logs có nền tảng chung.

## 3. Role quan trọng
- Active Directory Domain Services.
- DNS.
- DHCP.
- File Services/SMB.
- IIS.
- Failover Clustering.
- Hyper-V (nếu có ngoài Nutanix).
- Certificate Services.
- Remote Desktop Services.

Nguồn dự án không xác nhận team này quản các role trên; đây là knowledge foundation.

## 4. PowerShell read-only toolkit
```powershell
Get-ComputerInfo
Get-Service | Where-Object Status -eq 'Stopped'
Get-WinEvent -LogName System -MaxEvents 50
Get-NetIPConfiguration
Get-NetRoute
Get-NetTCPConnection -State Listen
Get-Volume
Get-Disk
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10
```

## 5. Event Viewer / Event Log
Phải hiểu:
- System.
- Application.
- Security.
- Setup.
- Role-specific channels.

Khi troubleshoot, filter theo time window và correlate Event ID/source. Không copy nguyên hàng nghìn event mà không có hypothesis.

## 6. Services
```powershell
Get-Service -Name w32time
Get-CimInstance Win32_Service -Filter "Name='w32time'" | Select Name,StartMode,State,PathName
```
Restart service là change có thể impact; kiểm tra dependency:
```powershell
Get-Service -Name <svc> -DependentServices
```

## 7. Network
```powershell
Get-NetAdapter
Get-NetIPConfiguration
Resolve-DnsName example.internal
Test-NetConnection server01 -Port 443
Get-NetRoute
```
Troubleshoot link/IP/DNS/route/port/TLS/application theo lớp.

## 8. Storage
- Disk/partition/volume.
- NTFS/ReFS.
- VSS.
- SMB.

```powershell
Get-Disk
Get-Partition
Get-Volume
vssadmin list writers
```
VSS writer lỗi có thể làm backup application-consistent fail.

## 9. Time
Kerberos, logs, TLS đều nhạy với time.
```powershell
w32tm /query /status
w32tm /query /source
```

## 10. Security/privilege
- Local Administrator.
- Domain groups.
- UAC.
- GPO.
- Service accounts/gMSA.
- Windows Firewall.
- Defender/EDR.

Không tắt firewall/AV để “test nhanh” trên production nếu chưa có approval.

## 11. Server Core
Quản trị bằng PowerShell/remote tools. Học `sconfig`, PowerShell remoting và Windows Admin Center concepts.

## 12. Backup/restore trên Nutanix/HYCU
Guest Windows có thể phụ thuộc VSS. Khi restore:
- machine SID/domain relationship,
- IP/name conflict,
- application/database consistency,
- service startup,
- backup agent/integration.

## 13. Patch/reboot
Checklist:
- change approval/freeze,
- backup/recovery point,
- disk free space,
- pending reboot,
- service owner,
- cluster/HA role,
- post-reboot event/service/network check.

## 14. Lab
- Windows Server evaluation lab: DNS/IIS basic.
- PowerShell collect health report.
- Simulate service failure và query EventLog.
- Test `Test-NetConnection` và TLS certificate inspect.
- Viết pre/post reboot checklist.
