> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# Nutanix: Prism / Prism Central / AOS / AHV

## 1. Liên hệ trực tiếp với dự án
**[Nguồn dự án]** Đây là lõi của phạm vi. Nghiệp vụ gồm VM create/change/delete/start/stop, Volume Group, image, capacity, alert, snapshot inventory, AsyncDR, Nutanix configuration change do issue, script, AOS/AHV/Prism software/firmware upgrade. File quyền tách Prism Central read/operator/admin, Prism Element cho AsyncDR/upgrade, và CVM privileged ID để SSH/ncli/acli.

## 2. Bốn khái niệm không được nhầm
### Prism Element (PE)
UI/management gắn với một cluster. Dùng cho nhiều thao tác cluster-local và một số data-protection/upgrade path tùy version.

### Prism Central (PC)
Management plane tập trung nhiều cluster: inventory, VM operations, alerts, policies và các tính năng quản trị tập trung tùy license/version.

### AOS
Nutanix Acropolis Operating System: data/services layer của HCI, chạy qua Controller VM (CVM) trên mỗi node và cung cấp distributed storage, cluster services, data protection, resiliency.

### AHV
Hypervisor của Nutanix, dựa trên KVM. AHV chạy VM workload; AOS/CVM cung cấp storage/control services cho cluster.

Mental model ngắn: **AHV chạy VM; CVM/AOS cung cấp HCI data services; Prism quản trị.**

## 3. Node và CVM
Mỗi physical node thường chạy AHV và một CVM. CVM giao tiếp để tạo distributed storage fabric. Khi troubleshoot:
- Không coi CVM như VM ứng dụng bình thường.
- Không reboot/power-off CVM tùy tiện.
- Dùng health/maintenance procedure được Nutanix hỗ trợ.

## 4. Storage concepts
- Storage pool/container.
- vDisk.
- Replication factor/resiliency.
- Data locality/placement do AOS quản lý.
- Capacity phải xem cả physical, usable và resiliency overhead.

Không suy luận “còn 20% disk = an toàn”; HCI cần headroom cho failure/rebuild/upgrade.

## 5. VM lifecycle
### Create
Cần xác nhận:
- Name/project/category/tag.
- vCPU/vRAM.
- image/template.
- disk/container.
- NIC/network/VLAN.
- IP/DNS do hệ thống nào cấp.
- guest OS customization.
- backup/monitoring.

### Change
Rủi ro tùy loại:
- vCPU/vRAM có thể yêu cầu reboot tùy guest/policy.
- disk resize cần guest filesystem expansion.
- NIC/VLAN có thể gây mất kết nối.
- power-off change cần downtime approval.

### Delete
Xác nhận backup/retention, owner, dependency, snapshot, IP/DNS, monitoring và CMDB trước khi xóa.

## 6. Volume Group
Volume Group cung cấp block devices cho VM/use case phù hợp. Cần hiểu:
- VG vs ordinary VM vDisk.
- attachment mapping.
- multipath/iSCSI behavior tùy thiết kế.
- create/change/delete và dependency.

**[Nguồn dự án]** VG create/change/delete là nghiệp vụ riêng và có admin permission Prism Central; một số change/delete thiếu procedure đầy đủ nên được yêu cầu chuẩn bị trước KT.

## 7. Image
Image dùng làm nguồn disk/ISO/template. Quy trình tốt:
- Verify source checksum/version.
- Naming convention.
- Import/create.
- Test boot/deploy.
- Record owner/date.
- Không xóa image còn referenced mà chưa kiểm tra dependency.

## 8. Snapshot và AsyncDR
Snapshot local không tự động là offsite backup. AsyncDR trong source được quản lý qua Prism Element và có flow setup/snapshot/restore/delete.

Phải biết:
- Protection policy/schedule.
- Local/remote target.
- RPO.
- Restore granularity.
- Consistency/application quiescing nếu có.
- Replication health.
- Failover/failback process.

## 9. Alerts/capacity
Read-only checklist:
- Cluster health.
- Critical alerts.
- Node/CVM status.
- CPU/memory/storage utilization.
- Replication/resiliency state.
- Failed disk/node.
- Recent tasks/events.

**[Nguồn dự án]** Có task xóa alert không cần thiết, capacity management, CVM Mem available check và snapshot inventory. Hãy phân biệt “acknowledge/resolve/delete alert” với “fix root cause”.

## 10. CLI mental map
### `ncli`
Nutanix CLI truyền thống cho cluster entities/settings.

### `acli`
AHV/VM-oriented CLI, thường dùng cho VM/network/image operations tùy use case.

### Linux shell trên CVM
Dùng theo procedure/support guidance. Không chỉnh file/service nội bộ tùy tiện.

Ví dụ chỉ để học read-only:
```bash
# Trên CVM, tùy version/privilege
cluster status
ncli cluster info
acli vm.list
```
**[Cần xác nhận trong KT]** exact command set và tài khoản cho version thực tế.

## 11. LCM và upgrade AOS/AHV/Prism
### Trước upgrade
- Compatibility matrix.
- Release notes/known issues.
- Cluster health/NCC.
- Capacity/headroom.
- Backup/DR status.
- Firmware/driver dependency.
- Maintenance window/freeze.
- Vendor support case nếu policy yêu cầu.
- Rollback/abort point; nhiều firmware/AOS upgrade không phải “downgrade một click”.

### Trong upgrade
- Theo dõi stage per node/component.
- Không chạy unrelated change.
- Ghi timestamp/error code.
- Nếu fail, thu log trước khi retry tùy tiện.

### Sau upgrade
- Version inventory.
- Health/NCC.
- VM availability.
- Storage/resiliency.
- Network.
- Alerts/tasks.
- HYCU backup.
- Monitoring/log forwarding.

**[Nguồn dự án]** Work schedule có AOS/AHV upgrade thực tế trên STG; plan transition yêu cầu học từ hồ sơ upgrade trước.

## 12. Troubleshooting theo lớp
### VM không reachable
1. Power state.
2. Console/guest boot.
3. vNIC/network mapping.
4. IP/route/DNS.
5. FortiGate/LB path.
6. AHV host/cluster health.

### VM chậm
1. Guest process/load.
2. vCPU ready/contention concept.
3. memory pressure.
4. disk latency/IOPS.
5. cluster capacity/health.
6. backup/snapshot activity.

### Cluster alert
Không chỉ đọc message; xác định component, severity, repeat count, correlated event và recent change.

## 13. Lab
- Dựng CE/lab nếu được phép: tạo VM, clone, snapshot, image.
- Thực hành PC vs PE navigation.
- Viết table mapping object: Cluster → Host → CVM → VM → vDisk → Container → Network.
- Simulate VM delete runbook có backup/rollback decision.
- Viết pre/post checklist cho AOS/AHV upgrade.

## 14. Câu hỏi KT trọng yếu
- Version AOS/AHV/PC hiện tại và target?
- Bao nhiêu cluster, prod/STG/PoC?
- PC có quản lý toàn bộ cluster không?
- AsyncDR topology/RPO?
- VG dùng cho workload nào?
- NCC/LCM policy?
- CVM access workflow?
- Snapshot retention/inventory rule?
- Khi incident, vendor support/escalation path thế nào?
