> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# HPE ProLiant và iLO

## 1. Liên hệ với dự án
**[Nguồn dự án]** HPE ProLiant là physical layer của Nutanix. iLO được dùng cho hardware status/IML log; admin iLO dùng cho config change/firmware upgrade. Có task HPE firmware upgrade và issue-driven HPE server config change.

## 2. iLO là gì
Integrated Lights-Out là out-of-band management controller. Nó hoạt động độc lập với OS/hypervisor ở mức lớn, dùng để:
- Power control.
- Remote console.
- Hardware inventory/health.
- Integrated Management Log (IML).
- Firmware management.
- Virtual media tùy license/model.

## 3. Hardware layers cần hiểu
- CPU/socket.
- DIMM/channel.
- RAID/HBA/storage controller.
- NIC/OCP adapter.
- PSU/fan/temperature.
- System ROM/BIOS.
- iLO firmware.

Trong Nutanix appliance/node, firmware/driver compatibility rất quan trọng; không nâng “latest” chỉ vì có bản mới.

## 4. Read-only health checklist
- Overall health.
- Active IML entries.
- Power supply redundancy.
- Fan/temp.
- DIMM degradation.
- Disk/controller state.
- NIC link.
- Firmware inventory.
- Server power state.

## 5. IML troubleshooting
Một hardware event tốt cần ghi:
- Component bay/slot.
- Severity.
- First/last occurrence.
- Repetition.
- Correlation với AOS/AHV alert.
- Có degraded redundancy hay imminent failure không.

Không clear IML trước khi evidence đã được thu thập.

## 6. Power operations
Power reset/cold boot có blast radius lớn với HCI node. Trước thao tác:
- Nutanix maintenance procedure.
- VM evacuation/host state nếu cần.
- Cluster resiliency/headroom.
- Confirmation from owner.

## 7. Firmware upgrade
### Pre-check
- HPE/Nutanix compatibility.
- Target firmware bundle.
- Current inventory.
- Maintenance mode/node sequence.
- Cluster health.
- Backup/config evidence.

### Execution
- One node at a time theo supported process.
- Monitor iLO job/POST.
- Không power cycle ngoài procedure.

### Post-check
- Firmware version.
- IML.
- Sensor health.
- AHV host/CVM rejoin.
- Nutanix cluster health/resiliency.
- VM/network/storage behavior.

## 8. Security
- iLO nằm trên management network; giới hạn source IP.
- Unique credentials/privileged workflow.
- TLS certificate nếu môi trường yêu cầu.
- Update firmware vì iLO là security-sensitive surface.

## 9. Lab/KT
- Tập đọc iLO hardware inventory và IML.
- Map physical node serial → AHV host → CVM → Prism object.
- Viết node maintenance checklist.
- Hỏi model ProLiant, iLO generation, firmware baseline, vendor support path.
