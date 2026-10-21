# Day 8 — Network Switching & Routing Operations: Khung kiến thức cần học

Day 8 thuộc module **Network, Firewall & Load Balancer Operations**, nhưng riêng Day 8 tập trung vào **enterprise network fundamentals, Cisco Catalyst, Cisco Nexus, L2/L3 connectivity và operational troubleshooting**. Theo syllabus, bạn phải vừa hiểu nền tảng networking vừa thực hiện được các thao tác vận hành, verification, troubleshooting, configuration backup và rollback trên cả **IOS/IOS XE** lẫn **NX-OS**. Infrastructure_Operations_14_Da…

Tôi sẽ chia Day 8 thành đúng **2 module lớn** như bạn yêu cầu:

- **Module 1 — Lý thuyết:** hiểu kiến trúc, protocol, switching/routing, Catalyst/Nexus và operational methodology.
- **Module 2 — Thực hành:** xây topology, cấu hình, verify, inject lỗi, troubleshoot, backup/compare/rollback và thu thập evidence.

Đây mới là **bản đồ đầy đủ của Day 8**. Khi bạn yêu cầu tiếp, tôi sẽ triển khai từng module ở mức rất sâu.

---

# MODULE 1 — LÝ THUYẾT

## 1. Enterprise Network Architecture

Phần đầu tiên là hiểu một enterprise network được tổ chức như thế nào, thay vì nhìn từng switch hoặc router độc lập.

Bạn cần hiểu:

- LAN, WAN và Data Center network.
- Access layer, Distribution layer, Core layer.
- Leaf/Spine awareness trong Data Center.
- End-user network, Server network, Management network.
- Production, non-production và management segmentation.
- North-South traffic và East-West traffic.
- Physical topology so với logical topology.
- Redundancy và high availability.
- Single point of failure.
- Control plane, data plane và management plane ở mức operational.
- Dependency giữa endpoint → access switch → uplink → routing → firewall/load balancer → server.

Mục tiêu cuối cùng là khi một application không reachable, bạn không troubleshoot ngẫu nhiên mà biết **traffic phải đi qua những layer nào**.

---

## 2. OSI Model và TCP/IP Model

Bạn cần dùng OSI không phải để học thuộc 7 layer đơn thuần, mà như một **troubleshooting framework**.

Các layer quan trọng nhất trong Day 8:

- Layer 1 — Physical.
- Layer 2 — Data Link.
- Layer 3 — Network.
- Layer 4 — Transport ở mức nhận diện dependency.

Ví dụ:

- Interface down → thường bắt đầu ở Layer 1.
- VLAN/trunk/MAC problem → Layer 2.
- Gateway/route problem → Layer 3.
- Ping được nhưng application TCP port không được → có thể đã vượt ngoài phạm vi L2/L3.

Bạn cần biết cách phân biệt:

> Physical connectivity → L2 adjacency → L3 reachability → upper-layer service.

---

# 3. IPv4 Addressing

Đây là nền móng cho toàn bộ phần routing.

Bạn cần nắm:

- IPv4 address.
- Network address.
- Host address.
- Broadcast address.
- Subnet mask.
- CIDR notation.
- Private IPv4 ranges.
- Same subnet vs different subnet.
- Default gateway.
- Network boundary.

Ví dụ phải đọc được:

```text
10.20.30.25/24
```

và xác định được:

```text
Network:   10.20.30.0
Broadcast: 10.20.30.255
Hosts:     10.20.30.1 - 10.20.30.254
```

---

# 4. Subnetting

Không chỉ tính subnet trên giấy, bạn phải hiểu subnet ảnh hưởng trực tiếp tới packet forwarding.

Cần học:

- CIDR `/8` đến `/32`.
- Binary representation cơ bản.
- Network bits và host bits.
- Subnet increment.
- Usable address range.
- Broadcast address.
- Address overlap.
- Point-to-point subnet awareness.
- Troubleshooting sai subnet mask.

Các lỗi thực tế:

```text
Host A: 10.10.10.10/24
Host B: 10.10.20.10/24
```

Host A sẽ không ARP trực tiếp cho Host B mà gửi traffic đến default gateway.

Ngược lại, subnet mask sai có thể khiến host tưởng rằng remote address đang ở local subnet.

---

# 5. Default Gateway

Bạn cần hiểu rõ host quyết định khi nào gửi trực tiếp và khi nào gửi qua gateway.

Workflow:

```text
Destination IP
       ↓
Compare with local subnet
       ↓
Same subnet? ── Yes → ARP destination
       │
       No
       ↓
ARP default gateway
       ↓
Send frame to gateway MAC
```

Đây là kiến thức cực kỳ quan trọng khi troubleshoot inter-VLAN communication.

---

# 6. ARP

**ARP — Address Resolution Protocol** ánh xạ IPv4 address thành MAC address trong local Layer 2 domain.

Bạn cần hiểu:

- ARP request.
- ARP reply.
- ARP cache/table.
- Dynamic ARP entry.
- Gateway ARP.
- Incomplete ARP.
- Stale ARP awareness.
- ARP dependency trong routed communication.

Operational commands sẽ liên quan đến:

```text
show ip arp
```

hoặc NX-OS tương ứng.

Bạn phải có khả năng phân biệt:

- Không có ARP.
- ARP sai.
- Có ARP nhưng route sai.
- Có route nhưng interface/VLAN không functioning.

---

# 7. Ethernet và MAC Address

Bạn cần hiểu Layer 2 forwarding dựa trên Ethernet frame và MAC address.

Các khái niệm:

- Source MAC.
- Destination MAC.
- Broadcast MAC.
- Unicast.
- Broadcast.
- MAC address table.
- Dynamic learning.
- Aging.
- Unknown unicast flooding.
- Broadcast domain.

---

# 8. MAC Address Learning

Switch học source MAC trên ingress interface.

Ví dụ:

```text
MAC address A arrives on Gi1/0/5
```

Switch ghi:

```text
MAC A → Gi1/0/5
```

Bạn sẽ học cách kiểm tra bằng các command như:

```text
show mac address-table
```

và dùng nó để trả lời:

- Endpoint đang nằm ở port nào?
- MAC đã được học chưa?
- MAC có xuất hiện ở đúng VLAN không?
- MAC có đang move giữa các port hay không?

---

# 9. Switching vs Routing

Đây là một trong những distinction quan trọng nhất.

### Switching

Dựa chủ yếu trên:

```text
Destination MAC
```

và hoạt động trong Layer 2 domain.

### Routing

Dựa trên:

```text
Destination IP
```

và lựa chọn next hop từ routing table.

Bạn phải hiểu:

```text
Same VLAN/subnet
→ Switching

Different subnet
→ Routing
```

---

# 10. Network Segmentation

Enterprise network không đưa toàn bộ device vào cùng một broadcast domain.

Bạn cần hiểu:

- Logical segmentation.
- Security segmentation.
- Broadcast containment.
- User VLAN.
- Server VLAN.
- Management VLAN.
- Backup VLAN.
- DMZ awareness.

Segmentation được triển khai chủ yếu thông qua:

```text
VLAN
+ subnet
+ routing
+ ACL/firewall controls
```

---

# 11. VLAN

**VLAN — Virtual LAN** chia một physical switching infrastructure thành nhiều logical Layer 2 broadcast domains.

Cần hiểu:

- VLAN ID.
- VLAN database.
- VLAN membership.
- Access VLAN.
- Native VLAN awareness.
- VLAN existence vs VLAN forwarding.
- VLAN/subnet relationship.
- VLAN naming conventions.

Ví dụ:

```text
VLAN 10 USERS
VLAN 20 SERVERS
VLAN 30 MANAGEMENT
```

Một lỗi rất thường gặp:

```text
Port configured VLAN 20
Endpoint expected VLAN 10
```

---

# 12. Access Port

Access port thường mang traffic của một VLAN cho endpoint.

Ví dụ:

```text
switchport mode access
switchport access vlan 10
```

Bạn cần hiểu:

- Access VLAN.
- Endpoint traffic.
- Untagged frame behavior ở mức operational.
- Port state.
- VLAN membership verification.

---

# 13. Trunk Port

Trunk mang nhiều VLAN giữa network devices hoặc switch ↔ virtualization platform.

Các khái niệm:

- Trunk.
- Allowed VLAN list.
- Native VLAN.
- VLAN pruning awareness.
- VLAN consistency.

Một trunk có thể up về physical nhưng vẫn gây outage nếu VLAN cần thiết không nằm trong:

```text
allowed VLAN list
```

---

# 14. IEEE 802.1Q Tagging

Bạn cần hiểu tại sao switch biết một frame trên trunk thuộc VLAN nào.

802.1Q thêm VLAN information vào Ethernet frame.

Cần phân biệt:

- Tagged traffic.
- Untagged/native VLAN traffic.
- Access port vs trunk port.
- Native VLAN mismatch.

Không cần đi quá sâu vào bit-level frame format, nhưng phải hiểu operational behavior.

---

# 15. Inter-VLAN Routing

Hai VLAN không giao tiếp với nhau chỉ bằng Layer 2 switching.

Ví dụ:

```text
VLAN 10 → 10.10.10.0/24
VLAN 20 → 10.10.20.0/24
```

Communication cần Layer 3 device.

Day 8 cần hiểu:

- SVI.
- Routed interface awareness.
- Default gateway nằm trên Layer 3 switch/router.
- `ip routing`.
- Routing giữa VLAN.

Ví dụ logical flow:

```text
Host VLAN10
   ↓
Gateway VLAN10 SVI
   ↓
L3 routing
   ↓
VLAN20 SVI
   ↓
Host VLAN20
```

---

# 16. Layer 3 Interfaces và SVI

Bạn cần hiểu hai khái niệm:

### SVI

```text
interface vlan 10
```

Đại diện Layer 3 interface cho VLAN.

### Routed physical port

Một physical interface có thể vận hành ở Layer 3 thay vì Layer 2 trên platform phù hợp.

Operationally phải biết kiểm tra:

- Interface up/down.
- Protocol state.
- IP address.
- VLAN state.
- Routing table entry.

---

# 17. Static Routes

Static route là route do administrator cấu hình.

Conceptual format:

```text
destination network
→ next-hop
```

Ví dụ Cisco-style:

```text
ip route 10.20.0.0 255.255.0.0 10.10.10.1
```

Cần hiểu:

- Destination network.
- Prefix/mask.
- Next hop.
- Exit interface awareness.
- Default route.
- Recursive lookup awareness.
- Missing route.
- Wrong next hop.
- More-specific route.

---

# 18. Routing Table Interpretation

Đây là kỹ năng operational bắt buộc.

Bạn phải đọc được routing table và trả lời:

- Destination có route không?
- Connected route hay static route?
- Next hop là gì?
- Outgoing interface là gì?
- Có default route không?
- Route nào được chọn khi nhiều prefix match?

Key concept:

> **Longest Prefix Match**

Ví dụ:

```text
10.0.0.0/8
10.10.0.0/16
10.10.10.0/24
```

Destination:

```text
10.10.10.25
```

sẽ match `/24`.

---

# 19. ACL Fundamentals

Day 8 yêu cầu **ACL fundamentals**, chưa phải firewall chuyên sâu như Day 9.

Bạn cần hiểu:

- Standard vs extended ACL awareness.
- Source/destination matching.
- Protocol matching.
- Permit/deny.
- Rule order.
- Implicit deny awareness.
- Inbound vs outbound placement.
- ACL troubleshooting.

Quan trọng:

> Route tồn tại không có nghĩa traffic chắc chắn được phép.

---

# 20. Spanning Tree Protocol — STP/RSTP

STP bảo vệ switched network khỏi Layer 2 loop.

Bạn cần hiểu:

- Layer 2 loop.
- Broadcast storm.
- MAC table instability.
- Duplicate frames.
- Bridge Protocol Data Unit — BPDU.
- Root bridge.
- Root port.
- Designated port.
- Blocking/discarding behavior.
- STP convergence.
- RSTP awareness.

Bạn phải đọc được operational state thay vì chỉ học theory.

---

# 21. Loop Prevention

Phải hiểu tại sao redundant physical path vừa cần thiết vừa nguy hiểm.

Ví dụ:

```text
SW1 ───── SW2
 │         │
 └── SW3 ──┘
```

Nếu tất cả links forward Layer 2 traffic mà không có loop-prevention mechanism, có thể sinh network loop.

STP/RSTP giữ redundancy nhưng block logical forwarding ở path cần thiết.

---

# 22. EtherChannel / Port Channel

Nhiều physical links có thể được bundle thành một logical link.

Cần hiểu:

- Link aggregation.
- Port-channel.
- Member interfaces.
- Bandwidth aggregation.
- Redundancy.
- Load distribution awareness.
- LACP awareness.
- Configuration consistency.

Operational failure:

```text
3 links configured
2 links bundled
1 link suspended
```

Network có thể vẫn hoạt động nhưng ở trạng thái degraded.

---

# 23. Interface Operational State

Bạn cần đọc interface state một cách có phương pháp.

Các trạng thái/metrics:

- administratively down.
- down/down.
- up/down.
- up/up.
- link state.
- input/output packets.
- drops.
- errors.
- CRC errors.
- interface resets.
- speed.
- duplex.

---

# 24. Speed và Duplex

Bạn cần hiểu:

- Auto-negotiation.
- Fixed speed.
- Fixed duplex.
- Full duplex.
- Half duplex awareness.
- Duplex mismatch.

Một link có thể:

```text
up/up
```

nhưng performance rất kém do physical/interface errors.

---

# 25. Link Errors

Các counters cần biết cách diễn giải:

- CRC.
- input errors.
- output errors.
- drops.
- overruns awareness.
- collisions awareness.
- interface resets.

Không được thấy CRC rồi lập tức kết luận switch hỏng.

Phải xem:

```text
transceiver
cable
optic
patch panel
remote interface
speed/duplex
error trend
```

---

# 26. LLDP và CDP

Đây là discovery protocols dùng để xác định neighbor.

### CDP

Cisco proprietary.

### LLDP

IEEE standard.

Có thể giúp xác định:

- Neighbor device.
- Local interface.
- Remote interface.
- Platform.
- Management information.

Đây là công cụ cực kỳ hữu ích khi topology documentation không hoàn toàn chính xác.

---

# 27. Cisco Catalyst Architecture

Bạn cần hiểu Catalyst chủ yếu xuất hiện trong enterprise/campus switching use cases.

Day 8 không yêu cầu thiết kế sâu Catalyst architecture, nhưng cần operational proficiency:

- Interfaces.
- VLANs.
- trunks.
- SVIs.
- routing.
- STP.
- EtherChannel.
- MAC table.
- ARP.
- IOS/IOS XE CLI.
- running-config.
- startup-config.
- configuration persistence.

---

# 28. Cisco IOS / IOS XE Operational CLI

Phải hiểu operational command hierarchy:

```text
User EXEC
Privileged EXEC
Global Configuration
Interface Configuration
```

Ví dụ:

```text
Switch>
Switch#
Switch(config)#
Switch(config-if)#
```

Bạn phải phân biệt rõ:

### Read-only verification

Ví dụ:

```text
show interfaces
show vlan brief
show ip route
```

với:

### Configuration-changing commands

Ví dụ:

```text
configure terminal
interface ...
switchport ...
```

---

# 29. Cisco Nexus Architecture

Nexus tập trung nhiều vào Data Center networking.

Day 8 cần hiểu:

- Nexus operational model.
- NX-OS.
- VLAN/trunk.
- Layer 3 interfaces.
- port-channel.
- management plane.
- management VRF.
- vPC awareness.
- operational differences với Catalyst.

Không được giả định:

```text
IOS XE syntax == NX-OS syntax
```

Syllabus đặc biệt yêu cầu giữ **Catalyst và Nexus runbook riêng biệt**. Infrastructure_Operations_14_Da…

---

# 30. NX-OS Operational CLI

Bạn sẽ học các NX-OS command tương ứng với những verification task chính:

- interface state.
- VLAN.
- trunk.
- MAC learning.
- ARP.
- routing.
- spanning tree.
- port-channel.
- configuration.
- neighbor discovery.

Điểm quan trọng không phải học một bảng command bằng trí nhớ, mà phải biết:

> Tôi muốn kiểm tra object gì → trên platform này dùng command nào?

---

# 31. IOS XE vs NX-OS

Một nội dung bắt buộc của Day 8 là so sánh operational behavior và command reference giữa hai platform.

Bạn cần có mental model:

| Area                    | Catalyst                                    | Nexus                            |
| ----------------------- | ------------------------------------------- | -------------------------------- |
| OS family               | IOS/IOS XE                                  | NX-OS                            |
| Common role             | Enterprise/campus                           | Data Center                      |
| Management              | Platform-specific                           | Management VRF thường quan trọng |
| L2/L3                   | Có                                          | Có                               |
| Port aggregation        | EtherChannel/port-channel                   | port-channel                     |
| Multi-chassis awareness | Stack/other technologies depending platform | vPC                              |
| Command syntax          | IOS-family                                  | NX-OS-specific                   |

Các feature/default phải verify theo actual release, không suy diễn giữa platforms. Đây cũng là yêu cầu explicit của syllabus. Infrastructure_Operations_14_Da…

---

# 32. Management VRF

Đặc biệt quan trọng trên Nexus.

Bạn cần hiểu:

**VRF — Virtual Routing and Forwarding**

cho phép duy trì routing table riêng biệt.

Management interface có thể thuộc management VRF, vì vậy:

```text
ping destination
```

và

```text
ping destination vrf management
```

có thể cho kết quả khác nhau tùy NX-OS release/configuration.

Điểm operational:

> Management path và production/data path không nhất thiết dùng cùng routing table.

---

# 33. vPC Awareness

**vPC — virtual PortChannel** là concept rất quan trọng của Nexus.

Day 8 chỉ yêu cầu awareness và phân biệt với standalone port-channel.

Standalone port-channel:

```text
Device
  ││
 Switch A
```

vPC conceptual:

```text
       Device
       │    │
   Nexus-A  Nexus-B
       \    /
        vPC
```

Bạn cần hiểu:

- Hai Nexus peers.
- Peer relationship.
- Logical multi-chassis connectivity.
- Redundancy.
- Không đồng nhất vPC với một port-channel thông thường.

---

# 34. Configuration Backup

Một Infrastructure Operations engineer không được thay đổi network configuration mà không có baseline/backup phù hợp.

Bạn cần học:

- `running-config`.
- `startup-config` ở những platform/mode áp dụng.
- Save/persist configuration.
- External configuration backup.
- Timestamp.
- Device hostname.
- Platform/version.
- Ticket/change reference.
- Integrity/retention awareness.

---

# 35. Running Configuration vs Saved Configuration

Syllabus yêu cầu rõ phải kiểm tra:

> **saved-versus-running configuration**. Infrastructure_Operations_14_Da…

Bạn cần hiểu vấn đề:

```text
Running config = đang hoạt động
Saved config   = cấu hình sẽ được dùng sau reboot
```

Nếu thay đổi đang chạy nhưng chưa được persist đúng cách, reboot có thể làm mất cấu hình.

Cách lưu cụ thể phải theo platform/version/runbook.

---

# 36. Configuration Difference

Trước và sau một approved change cần biết chính xác:

```text
What changed?
```

Bạn cần có khả năng:

- Capture before-state.
- Capture after-state.
- Compare.
- Verify only approved lines changed.
- Link change to ticket.
- Verify restored state after rollback.

---

# 37. Firmware / Software Upgrade Precautions

Day 8 không yêu cầu thực hiện production firmware upgrade, nhưng yêu cầu hiểu precautions.

Cần xem xét:

- Current version.
- Target version.
- Compatibility.
- Release notes.
- Hardware support.
- Boot image.
- Storage space.
- Redundancy.
- Maintenance window.
- Backup.
- Console/out-of-band access.
- Expected reboot.
- Upgrade path.
- Rollback limitations.
- Post-upgrade validation.

Đây là thao tác high-risk nên chỉ được thực hiện trong authorized environment/runbook.

---

# 38. Network Change Control

Bất kỳ configuration change nào trên enterprise switch cũng cần operational discipline.

Một change tốt phải có:

```text
Purpose
Impact
Affected devices
Interfaces/VLANs/routes
Prerequisites
Pre-check
Implementation
Verification
Stop conditions
Rollback
Post-check
Evidence
Approvals
```

---

# 39. L2 Connectivity Troubleshooting

Bạn sẽ học workflow từ ít rủi ro tới nhiều rủi ro.

Ví dụ:

```text
1. Interface state
2. Error counters
3. VLAN membership
4. Trunk status
5. Allowed VLANs
6. MAC learning
7. STP state
8. Port-channel state
9. Neighbor information
10. Configuration comparison
```

Đây là phần cực kỳ quan trọng của Day 8.

---

# 40. L3 Connectivity Troubleshooting

Sau khi L2 được xác nhận, tiếp tục:

```text
IP configuration
↓
Subnet mask
↓
Default gateway
↓
ARP
↓
SVI/L3 interface
↓
Routing table
↓
Static route
↓
Return route
↓
ACL
```

Bạn phải tránh kiểu troubleshoot:

> “Ping không được → route sai.”

Không được bỏ qua các layer trước đó.

---

# 41. VLAN Mismatch Troubleshooting

Một trong các failure scenario bắt buộc của Assignment.

Ví dụ:

```text
Endpoint expected VLAN 20

Switch port configured:
switchport access vlan 30
```

Hoặc trunk không cho VLAN cần thiết đi qua.

Bạn phải:

- detect,
- identify root cause,
- correct trong approved scope,
- verify path,
- retain before/after evidence.

---

# 42. Incorrect Route Troubleshooting

Failure scenario thứ hai của syllabus.

Có thể gồm:

- Missing static route.
- Wrong next hop.
- Incorrect prefix.
- Wrong default route.
- No return route.

Bạn phải trace traffic cả hai chiều:

```text
Forward path
AND
Return path
```

---

# 43. Interface-Down Troubleshooting

Failure scenario bắt buộc thứ ba.

Phải phân biệt:

```text
administratively down
physical down
protocol down
err-disabled awareness
port-channel member problem
remote-side issue
```

Và không tự ý bật interface nếu chưa biết tại sao nó bị shutdown trong production.

---

# 44. Operational Evidence

Mọi troubleshooting của Day 8 phải thu thập được evidence.

Tối thiểu gồm:

- Device hostname.
- Platform.
- OS/version.
- Timestamp/timezone.
- Interface status.
- VLAN status.
- Trunk status.
- MAC table.
- ARP table.
- Routing table.
- STP state.
- Port-channel state.
- Relevant configuration.
- Ping/path verification.
- Before/after config.
- Config backup.
- Ticket/change reference.
- Root cause.
- Corrective action.
- Rollback result.

Syllabus yêu cầu riêng một evidence set cho **Catalyst** và một evidence set cho **Nexus**, có platform/version, tested paths, root-cause reasoning, configuration backup và rollback verification. Infrastructure_Operations_14_Da…

---

# MODULE 2 — THỰC HÀNH

Assignment Day 8 yêu cầu xây dựng một **simulated enterprise switching topology sử dụng Cisco Catalyst và Cisco Nexus hoặc approved virtual equivalents**, sau đó cấu hình L2/L3, verify các operational tables, inject faults, restore connectivity và thu thập evidence. Infrastructure_Operations_14_Da…

Module thực hành sau này sẽ được triển khai thành lab hoàn chỉnh theo các phần dưới đây.

---

## 1. Lab Environment Preparation

Bạn sẽ chuẩn bị một topology lab với tối thiểu:

```text
Client-A
   |
Catalyst-1
   |
Catalyst-2 / L3
   |
Nexus-1
   |
Server-B
```

Phiên bản đầy đủ hơn có thể sử dụng:

```text
PC-A
 |
CAT-SW1
 |\
 | \
 | CAT-SW2
 |
NEXUS-1 ===== NEXUS-2
     |
   Server
```

Tất cả thao tác configuration-changing phải nằm trong:

```text
Lab / simulator / authorized environment
```

---

# 2. Baseline Collection

Trước khi thay đổi:

- Identify device.
- Record hostname.
- Record model.
- Record OS.
- Record IOS XE/NX-OS version.
- Record current interface state.
- Record existing VLANs.
- Record routing table.
- Record STP state.
- Record current config.
- Save evidence.

Mục tiêu:

> Không troubleshoot hoặc thay đổi một environment mà không biết trạng thái ban đầu.

---

# 3. Catalyst Read-Only Diagnosis

Bạn sẽ thực hiện một diagnosis hoàn toàn **read-only** trên Catalyst.

Nhóm checks:

```text
device identity
interface state
interface errors
VLAN state
trunk state
MAC learning
ARP
STP
EtherChannel
routing
neighbor discovery
configuration
```

Đây là yêu cầu riêng trong additive assignment. Infrastructure_Operations_14_Da…

---

# 4. Nexus Read-Only Diagnosis

Sau đó lặp lại cùng mục tiêu trên NX-OS, nhưng **không copy command blindly từ Catalyst**.

Bạn sẽ verify:

```text
interface
VLAN
trunk
MAC
ARP
STP
port-channel
route
VRF
neighbor
running configuration
```

và ghi nhận platform/version.

---

# 5. VLAN Configuration Lab

Bạn sẽ tạo một số VLAN mẫu như:

```text
VLAN 10 USERS
VLAN 20 SERVERS
VLAN 30 MANAGEMENT
```

Sau đó verify:

- VLAN exists.
- VLAN active.
- Correct ports belong to correct VLAN.
- Expected MAC addresses appear in correct VLAN.

---

# 6. Access Port Configuration

Bạn sẽ cấu hình endpoint-facing port.

Ví dụ mục tiêu:

```text
PC-A
→ VLAN 10
```

Verification:

```text
interface operational status
access mode
VLAN membership
MAC learning
endpoint IP
ping gateway
```

---

# 7. Trunk Configuration

Tiếp theo xây trunk giữa switches.

Bạn phải verify:

- trunk operational.
- VLAN tagging.
- allowed VLANs.
- native VLAN.
- both ends consistent.

Không chỉ thấy interface `up/up` rồi kết luận trunk tốt.

---

# 8. 802.1Q Verification

Bạn sẽ xác nhận VLAN traffic thực sự được carry qua trunk.

Ví dụ:

```text
VLAN 10 → permitted
VLAN 20 → permitted
VLAN 30 → intentionally not permitted
```

Sau đó test connectivity để thấy effect của allowed VLAN list.

---

# 9. SVI Configuration

Bạn sẽ tạo Layer 3 gateway cho VLAN.

Ví dụ:

```text
VLAN 10
Gateway 10.10.10.1/24

VLAN 20
Gateway 10.10.20.1/24
```

Verify:

- SVI present.
- Line protocol.
- IP correct.
- VLAN active.
- Connected route installed.

---

# 10. Inter-VLAN Routing Lab

Bạn sẽ làm cho:

```text
Client VLAN10
```

giao tiếp với:

```text
Server VLAN20
```

và trace:

```text
Client
→ access port
→ VLAN10
→ SVI10
→ routing
→ SVI20
→ VLAN20
→ server
```

Sau đó test return path.

---

# 11. Static Routing Lab

Topology sẽ có ít nhất hai Layer 3 segments/devices.

Bạn sẽ:

- Create static route.
- Verify route installation.
- Test next-hop reachability.
- Test destination.
- Verify return route.

Sau đó intentionally cấu hình một route sai trong safe lab để troubleshoot.

---

# 12. Routing Table Inspection

Bạn phải đọc routing table trước khi ping.

Mục tiêu:

```text
Destination?
Route source?
Prefix?
Next-hop?
Exit interface?
```

Sau đó dự đoán packet path.

---

# 13. ARP Table Inspection

Bạn sẽ:

1. Clear/test trong phạm vi lab khi phù hợp.
2. Generate traffic.
3. Observe ARP learning.
4. Map IP → MAC.
5. Cross-check MAC table.

Operational correlation:

```text
IP
 ↓ ARP
MAC
 ↓ MAC table
Switch interface
```

Đây là một trong những workflow hữu ích nhất trong network troubleshooting.

---

# 14. MAC Address Table Inspection

Bạn sẽ dùng MAC address để xác định endpoint location.

Workflow:

```text
IP address
↓
ARP table
↓
MAC address
↓
MAC address table
↓
Switch interface
↓
LLDP/CDP / next switch
```

Có thể lặp lại qua nhiều switches.

---

# 15. STP Lab

Bạn sẽ xem:

- Root bridge.
- Root port.
- Designated port.
- Forwarding port.
- Alternate/blocking/discarding port.

Sau đó giải thích:

> Link nào đang bị STP không cho forwarding và tại sao.

---

# 16. EtherChannel / Port-Channel Lab

Bạn sẽ bundle nhiều interfaces thành logical channel trong lab phù hợp.

Verify:

```text
Port-channel exists
Member count
Member state
Protocol
VLAN/trunk consistency
Traffic continuity
```

---

# 17. vPC Awareness Exercise

Nếu Nexus lab hỗ trợ, bạn sẽ inspect một vPC topology.

Nếu lab không hỗ trợ, syllabus cho phép topology/fixture demonstration.

Bạn phải giải thích được khác biệt giữa:

```text
Standalone port-channel
```

và:

```text
vPC
```

Đây là requirement trực tiếp của Day 8. Infrastructure_Operations_14_Da…

---

# 18. Interface Health Lab

Bạn sẽ inspect:

```text
link status
line protocol
speed
duplex
CRC
input errors
output errors
drops
```

Mục tiêu là phân loại:

```text
Healthy
Degraded
Down
Misconfigured
Possibly physical fault
```

---

# 19. LLDP/CDP Neighbor Discovery Lab

Bạn sẽ xây lại topology từ device evidence.

Ví dụ:

```text
CAT1 Gi1/0/48
→ CDP
→ CAT2 Gi1/0/47
```

Kết quả là một connectivity map có thể đối chiếu với topology dự kiến.

---

# 20. Failure Injection 1 — VLAN Mismatch

Instructor/lab sẽ deliberately tạo lỗi:

```text
Wrong access VLAN
```

hoặc:

```text
Missing allowed VLAN on trunk
```

Bạn phải troubleshoot từ evidence mà không được cho ngay đáp án.

---

# 21. Failure Injection 2 — Incorrect Route

Một static route sẽ:

- missing,
- có wrong next hop,
- có wrong destination,
- hoặc return route thiếu.

Bạn cần xác định root cause bằng:

```text
routing table
next-hop reachability
ARP
interface status
path test
```

---

# 22. Failure Injection 3 — Interface Down

Một link cần thiết sẽ down.

Bạn phải phân biệt:

```text
shutdown?
physical?
remote side?
port-channel?
configuration mismatch?
```

Sau đó restore chỉ trong approved scope.

---

# 23. Optional Failure Injection — Trunk Problem

Có thể mô phỏng:

```text
Allowed VLAN mismatch
Native VLAN mismatch
Wrong switchport mode
```

Sau đó kiểm tra:

```text
show interface trunk
VLAN state
STP
MAC learning
```

---

# 24. Optional Failure Injection — Port-Channel Problem

Một member có thể không join bundle vì configuration mismatch.

Bạn phải xác định:

- Which member failed.
- Why.
- Whether redundancy remains.
- Risk before correction.

---

# 25. L2 Troubleshooting Workflow

Bạn sẽ thực hành theo flow cố định:

```text
Physical
↓
Interface
↓
Errors
↓
Access/Trunk
↓
VLAN
↓
STP
↓
Port-channel
↓
MAC learning
↓
Neighbor
```

---

# 26. L3 Troubleshooting Workflow

Sau đó:

```text
IP
↓
Subnet mask
↓
Gateway
↓
ARP
↓
SVI
↓
Route
↓
Next-hop
↓
Return route
↓
ACL
```

---

# 27. Catalyst vs Nexus Command Comparison

Bạn sẽ xây một mini-runbook riêng cho hai platform.

Ví dụ categories:

```text
Device/version
Interfaces
VLAN
Trunk
MAC
ARP
STP
Port-channel
Routing
Neighbors
Configuration
```

Không chỉ ghi command mà phải ghi:

```text
Purpose
Expected output
Important fields
Failure indicators
Platform/version
```

---

# 28. Running vs Saved Configuration Check

Bạn sẽ xác nhận configuration đang active có được lưu/persist đúng cách theo platform và approved procedure hay chưa.

Phải phân biệt rõ:

```text
Active state
```

với:

```text
Persisted state
```

---

# 29. Configuration Backup

Trước một safe lab change, bạn sẽ export/capture config.

Evidence nên gồm:

```text
hostname
platform
version
timestamp
ticket/lab ID
configuration file
hash where used
```

---

# 30. Before/After Configuration Comparison

Sau correction:

```text
Before
vs
After
```

Bạn phải xác định chính xác:

- Approved lines changed.
- Unexpected lines?
- Connectivity restored?
- Config persisted?
- No new alarms/errors?

---

# 31. Rollback Exercise

Một change không chỉ cần triển khai được mà phải **back out được**.

Bạn sẽ:

1. Record baseline.
2. Apply safe change.
3. Verify.
4. Roll back.
5. Verify original behavior/configuration.
6. Capture evidence.

Syllabus yêu cầu rollback verification rõ ràng cho cả Catalyst/Nexus evidence set. Infrastructure_Operations_14_Da…

---

# 32. Connectivity Path Validation

Không chỉ `ping`.

Bạn sẽ phải trả lời:

```text
Source device?
Source VLAN?
Default gateway?
Layer 2 path?
Layer 3 hop?
Destination VLAN?
Return path?
```

và lưu expected/actual results.

---

# 33. Change Control Exercise

Một mini change record cho Day 8 sẽ bao gồm:

```text
Change purpose
Affected device(s)
Affected interface(s)
VLAN/routes
Risk
Pre-check
Configuration
Verification
Rollback
Post-check
Evidence
```

---

# 34. Evidence Collection

Final lab package phải chứa ít nhất:

```text
Network topology
Catalyst platform/version
Nexus platform/version

Catalyst:
- Interface output
- VLAN/trunk output
- MAC table
- ARP
- STP
- EtherChannel
- Routing table

Nexus:
- Interface output
- VLAN/trunk output
- MAC
- ARP
- STP
- Port-channel
- Routing table
- Relevant VRF evidence

Failure:
- Symptom
- Evidence
- Root cause
- Corrective action
- Verification

Change:
- Before configuration
- After configuration
- Backup
- Rollback verification
```

Đây chính là loại evidence mà syllabus dùng làm acceptance criteria cho Day 8. Infrastructure_Operations_14_Da…

---

# 35. Escalation Exercise

Bạn cũng phải biết khi nào **không nên tiếp tục sửa**.

Ví dụ nên escalate nếu:

- Hardware/optic fault suspected.
- Production uplink impact.
- Unknown trunk carrying many production VLANs.
- STP topology change có blast radius lớn.
- vPC peer problem ngoài runbook.
- Firmware/software defect suspected.
- Change requires unauthorized privilege.
- Configuration baseline không đủ.
- Rollback condition không bảo đảm.
- Multiple dependent systems affected.

---

# Kết quả mong đợi sau Day 8

Sau khi học xong cả hai module, bạn cần đạt được ít nhất 5 năng lực.

**Thứ nhất**, bạn có thể giải thích packet đi từ một endpoint sang một subnet khác bằng chuỗi:

```text
IP decision
→ ARP
→ Ethernet frame
→ VLAN
→ switching
→ gateway
→ routing
→ destination VLAN
→ destination host
```

**Thứ hai**, bạn có thể đọc và correlate:

```text
Interface table
VLAN table
Trunk state
MAC table
ARP table
STP state
Port-channel state
Routing table
```

để xác định lỗi nằm ở đâu.

**Thứ ba**, bạn vận hành được cả **Cisco Catalyst IOS/IOS XE** và **Cisco Nexus NX-OS** ở mức foundation/intermediate operations, đồng thời không giả định hai platform có command/default/management behavior giống nhau. Syllabus coi đây là hai skill riêng biệt trong competency matrix. Infrastructure_Operations_14_Da…

**Thứ tư**, bạn troubleshoot có phương pháp:

```text
Layer 1
→ Layer 2
→ Layer 3
→ policy/upper layer
```

thay vì thử command ngẫu nhiên.

**Thứ năm**, mọi thay đổi phải có:

```text
Baseline
→ Authorization
→ Backup
→ Change
→ Verification
→ Evidence
→ Rollback capability
→ Post-check
```

---

## Bản đồ Day 8 rút gọn

```text
DAY 8
│
├── MODULE 1 — LÝ THUYẾT
│   ├── Enterprise network architecture
│   ├── OSI / TCP-IP
│   ├── IPv4 / subnetting / gateway
│   ├── ARP / MAC
│   ├── Switching vs routing
│   ├── Segmentation / VLAN
│   ├── Access / trunk / 802.1Q
│   ├── Inter-VLAN routing / SVI
│   ├── Static routes / routing table
│   ├── ACL fundamentals
│   ├── STP / RSTP
│   ├── EtherChannel / port-channel
│   ├── Interface health / errors
│   ├── LLDP / CDP
│   ├── Catalyst / IOS XE
│   ├── Nexus / NX-OS
│   ├── Management VRF
│   ├── vPC awareness
│   ├── Configuration backup
│   ├── Firmware precautions
│   ├── Change control
│   └── L2/L3 troubleshooting
│
└── MODULE 2 — THỰC HÀNH
    ├── Build topology
    ├── Collect baseline
    ├── Catalyst read-only diagnosis
    ├── Nexus read-only diagnosis
    ├── Configure VLAN
    ├── Configure access ports
    ├── Configure trunk
    ├── Configure SVI
    ├── Configure inter-VLAN routing
    ├── Configure static routes
    ├── Inspect MAC / ARP / VLAN
    ├── Inspect STP
    ├── Inspect EtherChannel / port-channel
    ├── Inspect interfaces / errors
    ├── LLDP / CDP topology discovery
    ├── Compare IOS XE / NX-OS
    ├── Failure: VLAN mismatch
    ├── Failure: incorrect route
    ├── Failure: interface down
    ├── Restore connectivity
    ├── Backup running configuration
    ├── Running vs saved config check
    ├── Before/after diff
    ├── Rollback
    ├── Validate network paths
    └── Submit topology + commands + troubleshooting
        + change + backup/rollback evidence
```

Như vậy, **toàn bộ Day 8 trong syllabus đã được map vào hai module mà không bỏ topic nào**: phần Lecture nằm ở Module Lý thuyết và toàn bộ Assignment/Additive Assignment/Acceptance requirements được chuyển thành Module Thực hành. Infrastructure_Operations_14_Da…

Khi bạn nói **“Trình bày Module Lý thuyết Day 8”**, tôi sẽ bắt đầu giảng Module 1 từ foundation → packet flow → VLAN/STP/routing → Catalyst/Nexus → troubleshooting, với **What / Why / How / workflow / commands / verification / failure modes / evidence / escalation** cho từng phần.
