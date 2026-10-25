> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# FortiGate Firewall – từ policy đến CLI troubleshooting

## 1. Liên hệ với dự án
**[Nguồn dự án]** Có nghiệp vụ add/change/delete firewall policy, cấu hình object, config management, OS upgrade và issue-driven config change. Tài liệu sử dụng FortiGate WebUI, TeraTerm, WinMerge và `.xlsm` tạo material/config; quyền tách read-only, policy/object admin và OS-upgrade admin.

## 2. Mental model firewall policy
Traffic được quyết định bởi nhiều yếu tố:
- Ingress interface/zone.
- Source address.
- Destination address.
- Service/port.
- Schedule.
- Action.
- NAT.
- Security profiles.
- Route/session state.

Một policy “allow” vẫn có thể không chạy nếu route sai, object sai, policy order không match hoặc return path bất đối xứng.

## 3. VDOM
VDOM chia một FortiGate thành nhiều virtual firewall. Khi đọc/change phải luôn biết đang ở đúng VDOM. Đây là lỗi thao tác rất nguy hiểm khi nhiều môi trường có object tên giống nhau.

## 4. Object model
- Address/address group.
- Service/service group.
- Policy.
- VIP/DNAT nếu có.
- Static/dynamic route.
- Interface/VLAN.

Tạo object reusable tốt hơn nhét IP trực tiếp vào nhiều policy, nhưng phải theo naming standard.

## 5. Policy change workflow
1. Validate request: src/dst/protocol/port/environment.
2. Resolve DNS/IP và owner.
3. Kiểm tra object đã tồn tại chưa.
4. Search existing policy để tránh duplicate/shadow.
5. Xác định policy position.
6. Chuẩn bị config/CLI.
7. Peer review/WinMerge.
8. Backup config.
9. Apply trong window.
10. Test traffic và log.
11. Save evidence/update inventory.

## 6. CLI read-only hữu ích
```text
get system status
get system performance status
show system interface
get router info routing-table all
show firewall policy
show firewall address
show firewall addrgrp
diagnose sys session list
```
Command cụ thể có thể khác theo FortiOS/version/VDOM; ưu tiên `show/get` trước `config`.

## 7. CLI config structure
```text
config firewall address
    edit "OBJ_EXAMPLE"
        set subnet 10.10.10.10 255.255.255.255
    next
end
```
Ví dụ chỉ để học syntax. Production change phải dùng naming/IP/policy ID đã được review.

## 8. Session-based troubleshooting
Firewall là stateful. Nếu change policy nhưng session cũ vẫn tồn tại, behavior có thể chưa phản ánh ngay. Tuy nhiên xóa session có impact; chỉ làm khi runbook/approval cho phép.

Flow debug logic:
1. Có route không?
2. Policy nào match?
3. NAT có áp dụng không?
4. Session được tạo không?
5. Return traffic về cùng path không?
6. UTM/profile có drop không?

## 9. Logs
Cần biết:
- traffic allow/deny log,
- event/system log,
- security log,
- log forwarding sang Analyzer.

**[Nguồn dự án]** Có task định kỳ kiểm tra NW device logs được forward sang EventLogAnalyzer và task lấy communication logs qua FirewallAnalyzer.

## 10. HA
Nếu dùng HA, phải hiểu active/passive hoặc topology cụ thể, config sync và failover status. **[Cần xác nhận trong KT]** mode HA thực tế.

## 11. FortiOS upgrade
- Compatibility path, không nhảy version tùy ý.
- Backup config.
- Read release notes/known issues.
- Check HA sync, disk/log health.
- Post-check routing/policy/session/VPN/logging/HA.
- Validate Analyzer log ingestion.

## 12. Common mistakes
- Sai VDOM.
- IP mask sai (`/32` vs subnet).
- Source/destination đảo.
- Service TCP/UDP sai.
- Policy order sai.
- NAT bật/tắt sai.
- Object duplicate nhưng khác tên.
- Không test return path.

## 13. Lab
- Dựng 2 network, tạo address/service/policy.
- Viết script parser request CSV → proposed CLI nhưng **không auto-apply**.
- Diff before/after config.
- Tạo troubleshooting worksheet cho deny traffic.
