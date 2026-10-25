> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# Red Hat Enterprise Linux (RHEL) cho vận hành hạ tầng

## 1. Liên hệ với dự án
**[Nguồn dự án]** Work schedule có `Ansible自動化対応_OS払出(RHEL7.9/9.6)`, `RHEL9運用機能検証_バックアップポリシー解除` và `RHEL9運用機能検証_VM廃止`. SOW cũng yêu cầu kiến thức setup ban đầu Windows/Linux ở một số nghiệp vụ. Điều này cho thấy RHEL có thể là **guest OS/workload trên Nutanix**, không phải sản phẩm lõi do team quản lý như AOS/AHV.

## 2. Kiến trúc OS cần biết
- Kernel + user space.
- systemd quản lý service/boot.
- RPM package; `dnf` (RHEL 8/9) và `yum` compatibility.
- SELinux mandatory access control.
- firewalld/nftables.
- NetworkManager.
- LVM/XFS thường gặp.
- journald + rsyslog.
- chrony cho time sync.

## 3. RHEL 7 vs 8/9 – khác biệt nên nhớ
- RHEL 7 dùng systemd nhưng tooling/network/firewall cũ hơn; RHEL 8/9 chuẩn hóa `dnf`, nftables backend và có thay đổi package/module.
- Python/system libraries khác đáng kể; script cũ không nên giả định `/usr/bin/python` luôn tồn tại.
- Network script kiểu `/etc/sysconfig/network-scripts/ifcfg-*` dần bị loại bỏ; RHEL 9 nên ưu tiên NetworkManager/`nmcli`.
- Crypto policies và OpenSSL defaults mới hơn có thể làm client/protocol cũ fail.

## 4. Command read-only cơ bản
```bash
cat /etc/redhat-release
uname -r
hostnamectl
uptime
free -h
lsblk -f
df -hT
ip -br addr
ip route
ss -lntup
systemctl --failed
journalctl -p err..alert --since today
getenforce
firewall-cmd --state
```

## 5. systemd
```bash
systemctl status sshd
systemctl is-enabled sshd
systemctl list-units --type=service --state=failed
journalctl -u sshd --since '1 hour ago'
```
Trước `restart`, phải biết service dependency và impact. Trong production, `systemctl restart` là change chứ không phải troubleshooting read-only.

## 6. Package management
```bash
dnf repolist
dnf check-update
rpm -q <package>
rpm -qi <package>
rpm -V <package>
```
RHEL subscription/repository có thể qua Red Hat Satellite hoặc repo nội bộ; **[Cần xác nhận trong KT]** cách repo của dự án.

## 7. Storage/LVM
Mental model: disk → partition/PV → VG → LV → filesystem → mount.
```bash
lsblk
pvs
vgs
lvs -a -o +devices
findmnt
xfs_info /
```
Cảnh báo: shrink XFS không hỗ trợ như một thao tác online thông thường; không tự resize nếu chưa biết filesystem và backup.

## 8. Network
```bash
nmcli device status
nmcli connection show
ip addr
ip route
resolvectl status  # không có trên mọi version/config
getent hosts example.internal
```
Troubleshoot theo lớp: link → IP → route → DNS → port → TLS → application.

## 9. SELinux
```bash
getenforce
ausearch -m AVC -ts recent
semanage fcontext -l | head
```
Không biến `setenforce 0` thành “fix”. Nếu SELinux block, xác định label/policy đúng.

## 10. Logs
- `journalctl`: systemd journal.
- `/var/log/messages`, `/var/log/secure`, service-specific logs tùy config.
- Kiểm tra timezone bằng `timedatectl` trước khi correlate với Nutanix/HYCU/firewall logs.

## 11. User/privilege
```bash
id <user>
getent passwd <user>
sudo -l -U <user>
last -a | head
```
Không chia sẻ root password; dùng sudo/privileged ID theo policy.

## 12. Guest trên Nutanix: điểm cần liên kết
Khi VM RHEL có vấn đề, phân tách:
- Guest OS: CPU process, memory, filesystem, service, route.
- Hypervisor/VM: vCPU/vRAM, vNIC, virtual disk, snapshot.
- Cluster/storage: AOS health/capacity.
- Network: VLAN/uplink/firewall/LB.
- Backup: HYCU policy/job.

Nếu guest treo nhưng AHV/Prism vẫn thấy VM powered-on, cần console/iLO/serial strategy theo runbook.

## 13. RHEL provisioning/decommission checklist
### Provision
- Hostname/IP/DNS/NTP.
- CPU/RAM/disk.
- OS version/repo.
- user/sudo/SSH.
- security baseline/SELinux/firewalld.
- monitoring/log forwarding.
- backup policy.
- CMDB/tag/owner.

### Decommission
- Owner approval.
- Dependency check.
- Backup/retention decision.
- Disable monitoring/jobs.
- Remove DNS/IP reservation as applicable.
- Remove backup policy only khi đã xác nhận retention.
- Delete VM theo change.
- Update CMDB/ticket.

## 14. Lab
- RHEL 9 VM: tạo service systemd đơn giản, xem journal.
- Tạo filesystem/LVM lab và monitor capacity.
- Dùng `nmcli` tạo secondary connection trong lab.
- Bật SELinux enforcing và sửa lỗi label thay vì disable.
- Viết Ansible playbook read-only collect facts.
