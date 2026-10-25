> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# Cisco Nexus & Catalyst – L2/L3 switching

## 1. Liên hệ với dự án
**[Nguồn dự án]** Nexus/Catalyst là network scope; có OS upgrade, read-only alert checks và TAC-PAC collection. File quyền tách show-only và privilege 15 cho upgrade/tac-pac. Nguồn không đưa topology cụ thể, nên mọi nội dung vPC/HSRP/routing dưới đây là kiến thức nền, không phải xác nhận môi trường đang dùng.

## 2. L2 nền tảng
- MAC address table.
- VLAN.
- Access/trunk.
- 802.1Q tagging.
- STP/RSTP/MST.
- Port-channel/LACP.

### Read-only
```text
show interface status
show vlan brief
show interfaces trunk
show mac address-table
show spanning-tree
show etherchannel summary   # Catalyst/IOS style
show port-channel summary   # Nexus/NX-OS style
```

## 3. L3 nền tảng
- SVI.
- Static route.
- OSPF/BGP nếu dùng.
- Default gateway/VRF.
- HSRP/VRRP concept.

```text
show ip interface brief
show ip route
show arp
show ip ospf neighbor
show bgp ...
```
Exact syntax khác IOS XE/NX-OS.

## 4. Nexus-specific concepts
- NX-OS.
- vPC: multi-chassis port-channel.
- Peer-link/keepalive.
- VDC trên một số platform cũ.
- VRF rộng rãi.

vPC troubleshooting phải kiểm tra consistency, peer-link và member state; không shutdown peer-link tùy tiện.

## 5. Catalyst-specific concepts
Catalyst thường chạy IOS/IOS XE tùy generation. Các deployment mới có thể dùng StackWise/StackPower/DNA ecosystem. **[Cần xác nhận trong KT]** model/OS.

## 6. Network troubleshooting method
1. Physical link/err-disable.
2. VLAN/access/trunk.
3. STP/port-channel.
4. MAC learning.
5. ARP/ND.
6. L3 route/VRF.
7. ACL/firewall.
8. Application/LB.

## 7. Upgrade workflow
### Pre
- Model/current image/boot variable.
- Target release compatibility/ROMMON.
- Redundancy/stack/vPC status.
- Config backup.
- Free flash/bootflash.
- Routing/STP/port-channel baseline.
- Maintenance window.

### Post
- Version/uptime.
- Interface errors/up state.
- STP root/blocked ports.
- vPC/port-channel.
- HSRP/route neighbor.
- Syslog/monitoring.

**[Nguồn dự án]** Nexus và Catalyst upgrade được học qua previous upgrade package vì trong transition có thể không có case thật.

## 8. TAC-PAC / support bundle
Source có task lấy Cisco TAC-PAC. Nguyên tắc:
- Thu thập trước khi reboot nếu có thể.
- Ghi timestamp/problem symptom.
- Bảo vệ config/IP/credential data.
- Gắn đúng vendor case.

## 9. Lab
- 2 switch lab: VLAN/trunk/STP/LACP.
- Simulate trunk allowed VLAN thiếu.
- Compare `show` output trước/sau.
- Viết upgrade pre/post checklist chung, sau đó tách NX-OS vs IOS XE.
