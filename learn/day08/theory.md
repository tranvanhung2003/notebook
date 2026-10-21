# Day 8 — Module Lý thuyết

## Enterprise Networking, Cisco Catalyst, Cisco Nexus và L2/L3 Operations

Day 8 nằm trong **Network, Firewall & Load Balancer Operations**. Theo syllabus, phạm vi lý thuyết bắt buộc bao gồm: enterprise network architecture, OSI/TCP-IP, IPv4/subnetting/default gateway, ARP, MAC learning, switching/routing, segmentation, VLAN, access/trunk, 802.1Q, inter-VLAN routing, static route/routing table, ACL, STP/RSTP, EtherChannel/port-channel, interface health, speed/duplex/errors, LLDP/CDP, Catalyst + IOS/IOS XE, Nexus + NX-OS, management VRF, vPC awareness, configuration backup, firmware precautions, change control và L2/L3 troubleshooting. Ngoài ra, syllabus yêu cầu **Catalyst và Nexus phải được xem như hai platform vận hành riêng**, không được giả định syntax, defaults hay management behavior giống nhau. Infrastructure_Operations_14_Da…

Phần dưới đây chỉ dạy **Module Lý thuyết**. Tôi sẽ chưa biến nó thành bài lab từng bước; tuy nhiên các kiến thức sẽ được trình bày theo góc nhìn Infrastructure Operations: **What → Why → How → Verification → Failure modes → Troubleshooting → Evidence → Security → Escalation**.

---

# PHẦN I — MENTAL MODEL CỦA ENTERPRISE NETWORK

## 1. Enterprise network thực chất đang làm gì?

Ở mức đơn giản nhất, network thực hiện ba nhiệm vụ:

```text
Connect
Forward
Control
```

**Connect** nghĩa là cung cấp connectivity giữa endpoint, server, network appliance và external network.

**Forward** nghĩa là quyết định frame/packet phải đi đâu.

**Control** nghĩa là quyết định traffic nào được phép, topology nào đang active, route nào được sử dụng, link nào dự phòng, device nào được quản trị bằng đường nào.

Một enterprise network thường không phải:

```text
PC ─── Switch ─── Server
```

mà gần hơn với:

```text
User
 |
Access Switch
 |
Distribution/Core
 |
Firewall
 |
Load Balancer
 |
Data Center Switching
 |
Application Server
 |
Database
```

Ngoài ra còn có:

```text
Management network
Backup network
Storage network
Monitoring
Out-of-band management
Cloud connectivity
WAN
Internet
```

Infrastructure Operations engineer cần hiểu **dependency chain** này.

Nếu user báo:

> "Application không truy cập được."

không được mặc định application lỗi.

Có thể lỗi ở:

```text
Endpoint
  ↓
Access port
  ↓
VLAN
  ↓
Trunk
  ↓
STP
  ↓
Port-channel
  ↓
Default gateway
  ↓
Routing
  ↓
ACL
  ↓
Firewall
  ↓
Load balancer
  ↓
Server
  ↓
Application
```

Day 8 chủ yếu đào sâu đoạn:

```text
Physical
→ Layer 2
→ Layer 3
```

Day 9 mới tiếp tục firewall và load balancer.

---

# 2. Enterprise network layers

Mô hình campus truyền thống thường được giải thích theo ba tầng:

```text
Core
 ↑
Distribution
 ↑
Access
```

## Access layer

Là nơi endpoint kết nối.

Ví dụ:

- workstation;
- printer;
- IP phone;
- wireless AP;
- server trong một số topology.

Các chức năng thường gặp:

- access VLAN;
- port security awareness;
- 802.1X awareness;
- PoE awareness;
- edge STP controls;
- uplinks.

## Distribution layer

Aggregation nhiều access switches.

Có thể cung cấp:

- Layer 3 boundary;
- inter-VLAN routing;
- redundancy;
- policy;
- route summarization;
- first-hop gateway.

## Core layer

Mục tiêu chính:

> forward traffic nhanh, ổn định và resilient giữa những phần lớn của network.

Core không nhất thiết phải chứa nhiều policy phức tạp.

---

# 3. Data Center architecture awareness

Cisco Nexus thường liên quan nhiều hơn tới Data Center.

Một Data Center hiện đại có thể dùng:

```text
Spine
 / | \
Leaf Leaf Leaf
 |    |    |
Server / Hypervisor
```

Tuy nhiên Day 8 không yêu cầu bạn thiết kế VXLAN EVPN hay Data Center fabric nâng cao.

Điều cần hiểu là:

> Catalyst và Nexus có thể đều làm L2/L3 switching, nhưng deployment context, operating model và feature set có thể khác đáng kể.

---

# 4. North-South và East-West traffic

## North-South

Traffic đi giữa Data Center/internal network với external network.

Ví dụ:

```text
Internet
   |
Firewall
   |
Load Balancer
   |
Server
```

## East-West

Traffic giữa workloads bên trong Data Center.

Ví dụ:

```text
Web Server
     |
Application Server
     |
Database
```

Trong modern Data Center, East-West traffic có thể rất lớn.

Điều này giải thích tại sao Data Center switching chú trọng:

- low latency;
- high bandwidth;
- redundancy;
- port-channel;
- multi-path;
- vPC;
- leaf/spine architecture.

---

# 5. Physical topology và logical topology

Hai khái niệm này phải tách biệt.

Physical topology:

```text
SW1 Gi1/0/48
       |
       |
SW2 Gi1/0/47
```

Logical topology có thể là:

```text
VLAN 10
VLAN 20
VLAN 30
```

trên chính link đó.

Thậm chí một physical topology có loop:

```text
SW1 ----- SW2
 |         |
 |         |
SW3 -------
```

nhưng STP có thể làm logical forwarding topology trở thành loop-free.

Vì vậy:

> "Có cable/link" không đồng nghĩa "traffic đang forwarding qua link đó".

---

# PHẦN II — OSI VÀ TCP/IP DƯỚI GÓC NHÌN TROUBLESHOOTING

# 6. OSI model

OSI có 7 layers:

| Layer | Tên          | Ví dụ                                       |
| ----- | ------------ | ------------------------------------------- |
| 7     | Application  | HTTP, DNS                                   |
| 6     | Presentation | encoding, TLS-related presentation concepts |
| 5     | Session      | session control                             |
| 4     | Transport    | TCP, UDP                                    |
| 3     | Network      | IPv4, routing                               |
| 2     | Data Link    | Ethernet, VLAN, MAC, STP                    |
| 1     | Physical     | cable, optic, electrical/optical signal     |

Day 8 tập trung mạnh vào:

```text
Layer 1
Layer 2
Layer 3
```

và dùng Layer 4 chủ yếu để biết rằng:

> IP reachability thành công không đảm bảo application port hoạt động.

---

# 7. TCP/IP model

Có nhiều cách biểu diễn, phổ biến:

```text
Application
Transport
Internet
Network Access
```

Mapping gần đúng:

```text
OSI 7/6/5 → Application
OSI 4     → Transport
OSI 3     → Internet
OSI 2/1   → Network Access
```

Bạn không cần tranh luận mô hình nào "đúng hơn".

Trong Operations, mục tiêu là dùng model để khoanh vùng lỗi.

---

# 8. Troubleshooting theo layer

Một workflow tốt:

```text
Layer 1
↓
Layer 2
↓
Layer 3
↓
Layer 4+
```

Ví dụ server không reachable.

### Layer 1

Kiểm tra:

```text
Interface up?
Cable/optic?
Link?
Errors?
```

### Layer 2

Kiểm tra:

```text
Correct VLAN?
Correct trunk?
MAC learned?
STP forwarding?
Port-channel healthy?
```

### Layer 3

Kiểm tra:

```text
Correct IP?
Correct mask?
Gateway?
ARP?
Route?
Return route?
```

### Layer 4+

Sau khi L3 verified:

```text
TCP port?
Firewall?
Application?
TLS?
```

Một lỗi phổ biến của junior engineer là nhảy thẳng đến:

```text
ping
```

rồi kết luận.

Ping chỉ kiểm tra một phần của connectivity.

---

# PHẦN III — ETHERNET, MAC VÀ LAYER 2 SWITCHING

# 9. Ethernet frame

Trong Layer 2 Ethernet, switch chủ yếu quan tâm:

```text
Source MAC
Destination MAC
```

Một Ethernet frame về concept có:

```text
+------------------+
| Destination MAC  |
+------------------+
| Source MAC       |
+------------------+
| EtherType/Length |
+------------------+
| Payload          |
+------------------+
| FCS              |
+------------------+
```

Khi 802.1Q tagging được sử dụng, VLAN tag được thêm vào frame.

Không cần thuộc từng bit của frame để làm Day 8, nhưng phải hiểu:

> Switch Layer 2 forward dựa chủ yếu trên MAC address, không phải destination IP.

---

# 10. MAC address

MAC address thường dài 48 bits.

Biểu diễn ví dụ:

```text
00:11:22:33:44:55
```

hoặc Cisco-style:

```text
0011.2233.4455
```

MAC dùng trong một Layer 2 domain.

MAC không giống IP.

IP có ý nghĩa Layer 3 và thường gắn với logical network/subnet.

MAC phục vụ local frame delivery.

---

# 11. Unicast, broadcast và unknown unicast

## Known unicast

Switch biết destination MAC nằm ở đâu.

```text
MAC-B → Gi1/0/10
```

Frame chỉ được gửi ra interface đó.

## Broadcast

Destination:

```text
ff:ff:ff:ff:ff:ff
```

Switch flood broadcast trong VLAN tương ứng.

ARP Request là ví dụ điển hình.

## Unknown unicast

Switch chưa có destination MAC trong MAC address table.

Nó phải flood frame trong VLAN đó, trừ ingress port.

Khi destination trả lời, switch học location của MAC.

---

# 12. MAC address learning

Đây là cơ chế nền tảng của Ethernet switching.

Giả sử:

```text
PC-A
MAC AA
 |
Gi1/0/1
 |
Switch
```

PC-A gửi một frame.

Switch nhìn:

```text
Source MAC = AA
Ingress = Gi1/0/1
VLAN = 10
```

và học:

```text
VLAN 10
AA → Gi1/0/1
```

Điểm cực kỳ quan trọng:

> Switch học từ **source MAC**, không phải destination MAC.

---

# 13. MAC address table

Conceptual table:

```text
VLAN   MAC             Interface
----   --------------  ---------
10     AAAA.AAAA.AAAA  Gi1/0/1
10     BBBB.BBBB.BBBB  Gi1/0/2
20     CCCC.CCCC.CCCC  Gi1/0/10
```

Operationally đây là một trong những bảng quan trọng nhất.

Nếu biết IP của server nhưng không biết nó nằm switch port nào:

```text
IP
 ↓
ARP table
 ↓
MAC
 ↓
MAC table
 ↓
Interface
```

Sau đó nếu interface là uplink:

```text
LLDP/CDP
↓
Next switch
```

Lặp lại.

---

# 14. MAC aging

Dynamic MAC entries không tồn tại vĩnh viễn.

Nếu không còn traffic, sau aging period entry có thể bị loại.

Vì vậy:

> "MAC không có trong table" không nhất thiết chứng minh device chưa bao giờ kết nối.

Có thể đơn giản là không có traffic gần đây.

Khi troubleshoot, nên tạo traffic hợp lệ rồi quan sát learning lại nếu được phép.

---

# 15. MAC flapping

MAC flapping là khi cùng một MAC xuất hiện qua lại trên nhiều interface.

Ví dụ:

```text
MAC AA:
Gi1/0/1
→ Gi1/0/48
→ Gi1/0/1
→ Gi1/0/48
```

Có thể liên quan:

- Layer 2 loop;
- incorrectly connected devices;
- virtualization behavior;
- incorrect port-channel;
- topology issue.

Không được lập tức kết luận loop chỉ từ một MAC move đơn lẻ; cần correlate topology và logs.

---

# PHẦN IV — SWITCHING VÀ ROUTING

# 16. Switching

Layer 2 switch quyết định forwarding dựa trên:

```text
Destination MAC
```

Simplified process:

```text
Frame arrives
     ↓
Learn source MAC
     ↓
Look up destination MAC
     ↓
Known?
 ┌───┴────┐
Yes       No
 ↓         ↓
Forward   Flood in VLAN
```

---

# 17. Routing

Router/Layer 3 switch forward packet dựa trên:

```text
Destination IP
```

Simplified:

```text
Packet arrives
     ↓
Destination IP
     ↓
Routing table lookup
     ↓
Longest Prefix Match
     ↓
Next-hop / exit interface
```

Đây là khác biệt nền tảng:

```text
Switching → MAC
Routing   → IP
```

---

# 18. Router không đơn giản "chuyển tiếp Ethernet frame"

Một concept rất quan trọng.

Khi packet đi qua router:

```text
IP packet
```

được giữ về source/destination IP trong trường hợp không có NAT, nhưng Layer 2 frame được thay đổi cho mỗi routed hop.

Ví dụ:

```text
Host A
10.10.10.10
MAC A

Gateway
10.10.10.1
MAC G1

Router outgoing MAC G2

Server
10.20.20.20
MAC S
```

Host gửi:

```text
L2:
Source MAC      = A
Destination MAC = G1

L3:
Source IP       = 10.10.10.10
Destination IP  = 10.20.20.20
```

Router bỏ Ethernet header cũ, route packet rồi tạo Layer 2 header mới:

```text
L2:
Source MAC      = G2
Destination MAC = S

L3:
Source IP       = 10.10.10.10
Destination IP  = 10.20.20.20
```

Điều này giải thích vì sao:

> Destination MAC không phải end-to-end identifier xuyên qua router.

---

# PHẦN V — IPv4 ADDRESSING

# 19. IPv4

IPv4 address dài:

```text
32 bits
```

Ví dụ:

```text
192.168.10.25
```

Binary:

```text
11000000.10101000.00001010.00011001
```

Mỗi octet:

```text
8 bits
0–255
```

---

# 20. Network portion và host portion

CIDR xác định bao nhiêu bits thuộc network.

Ví dụ:

```text
10.20.30.25/24
```

`/24` nghĩa là:

```text
24 bits network
8 bits host
```

Subnet mask:

```text
255.255.255.0
```

Network:

```text
10.20.30.0
```

Broadcast:

```text
10.20.30.255
```

---

# 21. CIDR và subnet mask

Một số prefix cần nhận ra nhanh:

| CIDR | Mask            |             Tổng addresses |
| ---- | --------------- | -------------------------: |
| /8   | 255.0.0.0       |                 16,777,216 |
| /16  | 255.255.0.0     |                     65,536 |
| /24  | 255.255.255.0   |                        256 |
| /25  | 255.255.255.128 |                        128 |
| /26  | 255.255.255.192 |                         64 |
| /27  | 255.255.255.224 |                         32 |
| /28  | 255.255.255.240 |                         16 |
| /29  | 255.255.255.248 |                          8 |
| /30  | 255.255.255.252 |                          4 |
| /31  | 255.255.255.254 | special point-to-point use |
| /32  | 255.255.255.255 |          single host route |

Công thức tổng số addresses thông thường:

```text
2^(32-prefix)
```

Ví dụ `/26`:

```text
2^(32-26)
= 2^6
= 64
```

---

# 22. Subnetting example — /26

Cho:

```text
192.168.10.0/24
```

chia thành `/26`.

Block size:

```text
64
```

Các networks:

```text
192.168.10.0/26
192.168.10.64/26
192.168.10.128/26
192.168.10.192/26
```

Subnet đầu:

```text
Network:
192.168.10.0

Usable:
192.168.10.1
...
192.168.10.62

Broadcast:
192.168.10.63
```

Subnet thứ hai:

```text
192.168.10.64/26

Hosts:
65–126

Broadcast:
127
```

Subnetting không phải bài toán thi cử đơn thuần. Nếu subnet mask sai, forwarding decision cũng sai.

---

# 23. Same subnet vs different subnet

Host lấy:

```text
Destination IP
```

và subnet mask để xác định destination local hay remote.

Ví dụ:

```text
Host:
10.10.10.20/24

Destination:
10.10.10.50
```

Same subnet.

Host ARP destination trực tiếp.

Nhưng:

```text
Destination:
10.10.20.50
```

khác subnet.

Host không ARP `10.10.20.50`.

Nó ARP:

```text
default gateway
```

---

# 24. Default gateway

Default gateway là Layer 3 next hop mà host dùng cho destinations ngoài local subnet.

Ví dụ:

```text
Host:
10.10.10.20/24

Gateway:
10.10.10.1
```

Traffic tới:

```text
8.8.8.8
```

Host gửi Ethernet frame tới:

```text
MAC của 10.10.10.1
```

nhưng IP destination vẫn:

```text
8.8.8.8
```

Đây là concept phải thật rõ.

---

# 25. Lỗi subnet mask

Ví dụ:

Server thật ra phải là:

```text
10.10.10.50/24
```

nhưng cấu hình nhầm:

```text
10.10.10.50/16
```

Nó có thể nghĩ rằng:

```text
10.10.20.20
```

là local.

Thay vì gửi gateway, server ARP trực tiếp cho destination.

Nếu destination không cùng L2 domain, ARP thất bại.

Symptom có thể rất khó hiểu:

- một số destinations hoạt động;
- một số không;
- route trên switch/router hoàn toàn đúng.

Root cause lại nằm ở endpoint subnet mask.

---

# PHẦN VI — ARP

# 26. ARP là gì?

ARP = **Address Resolution Protocol**.

Vai trò:

```text
IPv4 address
→
MAC address
```

trong local Layer 2 segment.

Ví dụ PC cần gửi gateway:

```text
10.10.10.1
```

nhưng chưa biết MAC.

PC gửi broadcast:

```text
Who has 10.10.10.1?
```

Gateway reply bằng MAC của nó.

Sau đó PC cache mapping:

```text
10.10.10.1
→
0011.2233.4455
```

---

# 27. ARP Request

ARP Request thường là Layer 2 broadcast.

```text
Destination MAC:
ff:ff:ff:ff:ff:ff
```

Tất cả devices trong VLAN nhận frame.

Chỉ owner của requested IP nên reply bình thường.

---

# 28. ARP Reply

ARP Reply thường unicast về requester.

Sau đó requester lưu entry vào ARP cache.

---

# 29. ARP table khác MAC table

Cần tuyệt đối phân biệt.

### ARP table

```text
IP → MAC
```

### MAC table

```text
MAC → switch interface
```

Do đó:

```text
IP
↓
ARP
MAC
↓
MAC table
Port
```

Đây là một trong những correlation workflows mạnh nhất của Day 8.

---

# 30. ARP failure

Nếu gateway không có ARP cho directly connected endpoint, điều tra:

```text
Endpoint alive?
Correct IP?
Correct VLAN?
Correct subnet?
SVI up?
L2 path intact?
STP?
Trunk VLAN?
Port-channel?
```

Không được kết luận ngay:

> ARP lỗi.

ARP thường là **symptom của L2 issue**.

---

# PHẦN VII — NETWORK SEGMENTATION VÀ VLAN

# 31. Network segmentation

Nếu đặt hàng nghìn devices trong cùng một Layer 2 domain:

- broadcast domain quá lớn;
- operational blast radius lớn;
- troubleshooting khó;
- security isolation yếu;
- topology phức tạp.

Vì vậy chia network thành segments.

Ví dụ:

```text
VLAN 10  USERS
VLAN 20  SERVERS
VLAN 30  MANAGEMENT
VLAN 40  BACKUP
VLAN 50  VOICE
```

Thường mỗi VLAN ánh xạ với một IP subnet.

Ví dụ:

```text
VLAN 10 → 10.10.10.0/24
VLAN 20 → 10.10.20.0/24
VLAN 30 → 10.10.30.0/24
```

Không phải mathematical requirement rằng VLAN luôn đúng một subnet trong mọi design, nhưng đây là design model rất phổ biến và dễ vận hành.

---

# 32. VLAN là gì?

VLAN = **Virtual Local Area Network**.

VLAN tạo một logical Layer 2 broadcast domain.

Hai interfaces trên cùng physical switch nhưng khác VLAN:

```text
Gi1/0/1 → VLAN 10
Gi1/0/2 → VLAN 20
```

không trực tiếp Layer 2 communicate như cùng VLAN.

Muốn giao tiếp cần Layer 3 routing.

---

# 33. VLAN ID

802.1Q VLAN ID usable thông thường:

```text
1–4094
```

nhưng VLAN ranges/reservations/feature behavior phụ thuộc platform và software.

Operations không nên tự giả định một VLAN ID đang available.

Trước change:

```text
Check existing VLAN database
Check local standards
Check ticket/design
Check reserved VLANs
```

---

# 34. Access port

Access port thường phục vụ một VLAN.

Concept:

```text
PC
 |
untagged
 |
Access Port
 |
VLAN 10
```

Cisco-style concept:

```text
switchport mode access
switchport access vlan 10
```

Configuration-changing commands này chỉ nên dùng trong authorized lab/change.

---

# 35. Access port failure modes

Các lỗi phổ biến:

### Wrong VLAN

Expected:

```text
VLAN 10
```

Actual:

```text
VLAN 20
```

Physical interface vẫn có thể:

```text
up/up
```

nhưng endpoint sẽ vào sai broadcast domain.

### VLAN does not exist

Port config có thể chỉ reference VLAN nhưng operational behavior có thể không như mong đợi.

### Endpoint addressing mismatch

Port VLAN đúng nhưng host:

```text
10.20.20.x
```

trong khi VLAN đó phải là:

```text
10.10.10.0/24
```

Đây không phải switch configuration issue.

---

# 36. Trunk

Trunk dùng để mang nhiều VLAN trên cùng link.

Ví dụ:

```text
SW1
 |
 | VLAN 10
 | VLAN 20
 | VLAN 30
 |
SW2
```

Một trunk là logical multiplexing nhiều Layer 2 networks qua một physical/logical link.

Tài liệu Catalyst hiện hành vẫn mô tả trunk là point-to-point link mang traffic của nhiều VLAN, trong khi Nexus hiện hành cũng định nghĩa access port là single-VLAN và trunk là multi-VLAN interface dùng IEEE 802.1Q. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr2-fwd/vlan/vlan-configuration-guide/configure-vlan-trunks.html?utm_source=chatgpt.com)

---

# 37. IEEE 802.1Q

802.1Q thêm VLAN identification vào Ethernet frame.

Conceptually:

```text
Normal Ethernet:
[DMAC][SMAC][Type][Payload]

802.1Q:
[DMAC][SMAC][802.1Q Tag][Type][Payload]
```

VLAN tag cho receiving switch biết frame thuộc VLAN nào.

---

# 38. Tagged và untagged traffic

Trên trunk, phần lớn VLAN traffic được tagged.

Một VLAN có thể được xử lý là **native VLAN**, trong đó untagged behavior áp dụng theo platform/configuration.

Operational consequence:

> Native VLAN mismatch có thể gây connectivity hoặc STP inconsistency.

Do đó trunk verification phải xem:

```text
Mode
Native VLAN
Allowed VLANs
Operational VLANs
STP
```

chứ không chỉ:

```text
interface up
```

---

# 39. Allowed VLAN list

Trunk có thể về physical là:

```text
up/up
```

nhưng một VLAN cụ thể không đi được.

Ví dụ:

```text
Allowed:
10,20,30
```

Traffic VLAN 40 sẽ không traverse trunk.

Đây là failure mode rất phổ biến.

---

# 40. Trunk mismatch

SW1:

```text
Allowed VLAN:
10,20,30
```

SW2:

```text
Allowed VLAN:
10,20
```

VLAN 30 có asymmetric/missing forwarding behavior.

Một lỗi khác:

```text
SW1 → trunk
SW2 → access
```

Điều này phải được phát hiện bằng operational state, không chỉ reading one side's config.

---

# 41. Security của VLAN/trunk

Least-privilege networking cũng áp dụng ở Layer 2.

Không nên:

```text
allow all VLANs everywhere
```

nếu không có design justification.

Nguyên tắc tốt:

```text
Only carry required VLANs
```

Lợi ích:

- giảm blast radius;
- giảm accidental propagation;
- giảm attack surface;
- giảm operational ambiguity.

Không được xem VLAN là security boundary tuyệt đối. Policy enforcement thường cần thêm ACL/firewall/other controls.

---

# PHẦN VIII — INTER-VLAN ROUTING VÀ SVI

# 42. Tại sao VLAN khác nhau cần routing?

Cho:

```text
VLAN 10:
10.10.10.0/24

VLAN 20:
10.10.20.0/24
```

Host:

```text
10.10.10.10
```

muốn tới:

```text
10.10.20.20
```

Destination không local.

Host gửi packet đến default gateway.

Một Layer 3 device nhận packet và route sang VLAN 20.

---

# 43. SVI

SVI = **Switch Virtual Interface**.

Ví dụ concept:

```text
interface Vlan10
 ip address 10.10.10.1/24
```

Nó là Layer 3 interface đại diện cho VLAN trên multilayer switch.

Thường:

```text
Host VLAN10 gateway = SVI VLAN10
```

Nexus documentation cũng mô tả VLAN interface/SVI là virtual routed interface nối VLAN với Layer 3 routing engine; trên Nexus 9000, việc sử dụng SVI có những feature/prerequisite cụ thể cần kiểm tra theo release. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x/b-cisco-nexus-9000-nx-os-interfaces-configuration-guide-93x_chapter_0101.html?utm_source=chatgpt.com)

---

# 44. SVI up/down không chỉ phụ thuộc configuration

Một interface VLAN tồn tại trong config chưa chắc operational up.

Tùy platform và environment, nó có thể cần:

- VLAN tồn tại;
- VLAN active;
- relevant Layer 2 ports operational;
- interface không shutdown;
- feature enablement phù hợp.

Vì vậy:

```text
Configured
≠
Operational
```

---

# 45. Inter-VLAN forwarding flow

Ví dụ:

```text
PC-A
10.10.10.10/24
GW 10.10.10.1

Server-B
10.10.20.20/24
GW 10.10.20.1
```

Switch có:

```text
SVI VLAN10
10.10.10.1

SVI VLAN20
10.10.20.1
```

Flow:

```text
PC-A
 ↓
ARP 10.10.10.1
 ↓
Send frame to VLAN10 gateway MAC
 ↓
L3 switch receives packet
 ↓
Route lookup: 10.10.20.0/24 connected
 ↓
ARP for 10.10.20.20 if necessary
 ↓
Build new Ethernet frame
 ↓
Forward VLAN20
 ↓
Server-B
```

Return traffic làm quá trình ngược lại.

---

# PHẦN IX — ROUTING TABLE VÀ STATIC ROUTES

# 46. Routing table

Routing table trả lời:

> Destination IP này phải forward về đâu?

Conceptual output:

```text
10.10.10.0/24 directly connected
10.10.20.0/24 directly connected
10.20.0.0/16 via 10.10.30.2
0.0.0.0/0 via 10.10.30.254
```

---

# 47. Route components

Một route có thể gồm:

```text
Destination prefix
Prefix length
Route source
Administrative information
Metric
Next hop
Exit interface
```

Tùy OS/output.

Ở Day 8, quan trọng nhất:

```text
Does a matching route exist?
Which route wins?
What is next hop?
Can next hop be reached?
What is exit interface?
```

---

# 48. Longest Prefix Match

Đây là rule cực kỳ quan trọng.

Routing table:

```text
10.0.0.0/8       → A
10.10.0.0/16     → B
10.10.10.0/24    → C
```

Destination:

```text
10.10.10.50
```

match cả ba.

Route được chọn:

```text
10.10.10.0/24
```

vì `/24` specific hơn.

Không phải route được thêm trước.

Không phải route có network "trông gần nhất".

---

# 49. Connected route

Khi Layer 3 interface operational với:

```text
10.10.10.1/24
```

device thường có connected route:

```text
10.10.10.0/24
```

Nếu expected connected route mất, cần nghĩ tới:

```text
Interface down?
SVI down?
IP removed?
VRF mismatch?
Feature state?
```

---

# 50. Static route

Static route được administrator chỉ định.

Concept:

```text
Destination
→ Next hop
```

Cisco IOS-style:

```text
ip route 10.20.0.0 255.255.0.0 10.10.30.2
```

Ý nghĩa:

```text
Muốn tới 10.20.0.0/16
hãy gửi qua 10.10.30.2
```

Syntax chính xác cần xác minh theo platform/release.

---

# 51. Default route

IPv4 default route:

```text
0.0.0.0/0
```

Nó match mọi destination nhưng có specificity thấp nhất.

Dùng khi không có route specific hơn.

Mental model:

```text
No specific route?
↓
Use default route if present
↓
Otherwise drop / unreachable
```

---

# 52. Missing route

Symptom:

```text
Same subnet works
Gateway works
Remote subnet fails
```

Check routing table.

Nếu destination không match route phù hợp:

```text
routing failure
```

Nhưng vẫn phải kiểm tra cả return path.

---

# 53. Wrong next hop

Ví dụ:

Correct:

```text
10.20.0.0/16 via 10.10.30.2
```

Actual:

```text
10.20.0.0/16 via 10.10.30.3
```

Route tồn tại nên junior engineer có thể tưởng routing đúng.

Phải xem:

```text
What next hop?
Is it correct?
Reachable?
Does it know destination?
```

---

# 54. Return route

Connectivity luôn cần suy nghĩ hai chiều.

Forward:

```text
A
→ B
→ C
```

Reply phải có đường:

```text
C
→ B
→ A
```

Forward route đúng không chứng minh return route đúng.

Asymmetric routing cũng không nhất thiết là lỗi trong mọi architecture, nhưng có thể gây problem với stateful devices.

---

# PHẦN X — ACL FUNDAMENTALS

# 55. ACL là gì?

ACL = **Access Control List**.

Ở mức fundamental, ACL dùng rules để:

```text
permit
```

hoặc:

```text
deny
```

traffic matching criteria.

Criteria có thể gồm:

- source IP;
- destination IP;
- protocol;
- TCP/UDP port;
- direction/context.

---

# 56. Route và ACL là hai việc khác nhau

Routing trả lời:

> Traffic nên đi đâu?

ACL trả lời:

> Traffic matching policy có được phép hay không?

Do đó:

```text
Route exists
```

không đồng nghĩa:

```text
Traffic permitted
```

---

# 57. Rule order

ACL thường được evaluated theo thứ tự.

Concept:

```text
Rule 10
Rule 20
Rule 30
...
```

Một rule rộng ở trên có thể làm rule specific phía dưới không bao giờ hit.

Ví dụ:

```text
deny IP from subnet A to subnet B
```

đặt trước:

```text
permit TCP 443 A → server B
```

thì permit có thể trở nên unreachable tùy ACL semantics.

---

# 58. Implicit deny awareness

Nhiều ACL models có implicit deny ở cuối.

Nếu traffic không match permit nào:

```text
deny
```

Đây là concept phải kiểm tra theo platform/feature cụ thể trước khi change.

---

# 59. Security và ACL

Nguyên tắc:

```text
Least privilege
```

nghĩa là chỉ permit traffic business-required.

Không nên sửa outage bằng cách:

```text
permit any any
```

chỉ để "xem có chạy không" trong production.

Nếu cần troubleshoot policy, phải dùng approved controlled test.

---

# PHẦN XI — STP/RSTP VÀ LOOP PREVENTION

# 60. Vì sao Layer 2 loop nguy hiểm?

Ethernet broadcast frame không có Layer 3 TTL tương tự IP packet.

Nếu topology có Layer 2 loop:

```text
SW1 ---- SW2
 |        |
 |        |
SW3 ------
```

một broadcast có thể circulate.

Hậu quả:

- broadcast storm;
- duplicate frames;
- MAC table instability;
- high CPU;
- congestion;
- widespread outage.

---

# 61. Tại sao vẫn cần redundant links?

Không redundancy:

```text
SW1 ----- SW2
```

link chết:

```text
outage
```

Có redundancy:

```text
SW1 ----- SW2
 \         /
   --SW3--
```

nhưng tạo potential loop.

Giải pháp:

> Có physical redundancy nhưng logical Layer 2 topology phải loop-free.

Đó là vai trò của STP.

---

# 62. STP

STP = **Spanning Tree Protocol**.

STP chọn một loop-free forwarding tree.

High-level:

```text
Elect root bridge
       ↓
Calculate best paths to root
       ↓
Select port roles
       ↓
Block/discard redundant forwarding paths
```

Catalyst documentation hiện hành vẫn cung cấp PVST+/rapid-PVST+ operating models trên Catalyst families; exact mode/default phải kiểm tra trên actual platform/release. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr2-fwd/stp/stp-configuration-guide/m-stp.html?utm_source=chatgpt.com)

---

# 63. Root bridge

STP cần một reference point gọi là:

```text
Root Bridge
```

Election sử dụng Bridge ID.

Conceptually Bridge ID liên quan:

```text
Bridge priority
+
system/MAC-related identifier
```

Lowest Bridge ID thắng.

Trong per-VLAN variants, election có thể theo từng VLAN.

---

# 64. Root Port

Trên non-root switch, Root Port là port cung cấp best path về root bridge.

Thông thường mỗi STP instance trên non-root bridge có một Root Port.

---

# 65. Designated Port

Trên mỗi Layer 2 segment, một port được chọn làm Designated Port cho forwarding direction tương ứng.

Root bridge ports thường là designated trong topology đơn giản.

---

# 66. Alternate Port

RSTP có Alternate Port cung cấp alternative path.

Nó có thể ở discarding state nhưng sẵn sàng thay thế path chính khi topology thay đổi.

---

# 67. STP states

Classic STP thường được giảng:

```text
Blocking
Listening
Learning
Forwarding
Disabled
```

RSTP simplified state model:

```text
Discarding
Learning
Forwarding
```

Không được nhầm:

```text
Physical interface up
```

với:

```text
STP forwarding
```

Port có thể link-up nhưng STP đang không forward user frames.

---

# 68. RSTP

RSTP = **Rapid Spanning Tree Protocol**.

Mục tiêu lớn:

```text
faster convergence
```

so với classic STP trong nhiều scenarios.

Day 8 không yêu cầu bạn thành expert STP timer tuning.

Bạn phải biết operationally:

```text
Root?
Port role?
State?
Why blocked/discarding?
Topology changes?
```

---

# 69. Không tự ý "fix" STP bằng cách disable STP

Một lỗi cực nguy hiểm:

> "Port bị STP block nên tôi disable STP."

STP có thể đang bảo vệ network khỏi loop.

Nếu một redundant link đang blocked đúng thiết kế, đó không phải failure.

Cisco cũng cảnh báo việc vô hiệu STP không phù hợp trên native VLAN/trunk có thể tạo loop. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr2-fwd/vlan/vlan-configuration-guide/configure-vlan-trunks.html?utm_source=chatgpt.com)

---

# 70. STP evidence

Khi có suspicion L2 loop hoặc blocked path, evidence cần:

```text
STP root identity
Local bridge identity
Port role
Port state
Cost
Topology change info
Affected VLAN
Timestamp
Physical topology
```

Không chỉ screenshot dòng:

```text
Blocking
```

---

# PHẦN XII — ETHERCHANNEL / PORT-CHANNEL / LINK AGGREGATION

# 71. Vấn đề của một link đơn

Giả sử:

```text
SW1
 |
10 Gbps
 |
SW2
```

Hai vấn đề:

1. link failure làm mất connectivity;
2. throughput bị giới hạn bởi link đó.

Có thể dùng nhiều links:

```text
SW1
 ||||
 ||||
SW2
```

nhưng nếu để independent L2 links, STP có thể block một số để tránh loop.

---

# 72. EtherChannel / port-channel

Một nhóm physical links được kết hợp thành một logical interface:

```text
Eth1
Eth2
Eth3
Eth4
 \ | | /
Port-Channel10
```

Lợi ích:

- link redundancy;
- aggregate capacity;
- logical simplification;
- STP nhìn logical bundle thay vì từng member trong nhiều scenarios.

---

# 73. EtherChannel vs port-channel

Trong Cisco terminology:

**EtherChannel** thường mô tả technology/bundle.

**Port-channel** thường là logical interface.

Ví dụ:

```text
Port-channel10
```

là logical interface đại diện cho members.

---

# 74. LACP

LACP = **Link Aggregation Control Protocol**.

Standardized protocol dùng để negotiate link aggregation.

Common modes:

```text
active
passive
```

Concept:

```text
active + active  → negotiate
active + passive → negotiate
passive + passive → không chủ động negotiate
```

Exact platform behavior/configuration phải verify.

---

# 75. PAgP awareness

PAgP là Cisco-specific aggregation protocol, chủ yếu liên quan IOS/Catalyst operational knowledge.

Bạn không nên assume Nexus dùng cùng workflow.

Đây là ví dụ điển hình cho lý do syllabus bắt buộc:

> separate IOS/IOS XE and NX-OS runbooks.

---

# 76. Configuration consistency

Members trong bundle phải compatible.

Có thể cần consistency ở:

- Layer 2 vs Layer 3 mode;
- access/trunk;
- access VLAN;
- native VLAN;
- allowed VLANs;
- speed;
- other interface parameters.

Cisco Catalyst documentation hiện hành nhấn mạnh các Layer 2 EtherChannel members phải có compatible VLAN/trunk configuration; allowed VLAN ranges không đồng nhất có thể ngăn bundle hình thành đúng. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr2-fwd/etherchannel/etherchannel-configuration-guide/etherchannels.html?utm_source=chatgpt.com)

---

# 77. Port-channel không có nghĩa một flow dùng tổng toàn bộ bandwidth

Đây là misconception phổ biến.

Nếu có:

```text
4 x 10 Gbps
```

aggregate capacity về lý thuyết:

```text
40 Gbps
```

nhưng một single flow thường được hash lên một member, không tự động chia packet của một flow đều trên bốn links.

Do đó:

```text
aggregate throughput ≠ single-flow throughput
```

---

# 78. Degraded port-channel

Ví dụ:

```text
Po10
Members:
Eth1 → bundled
Eth2 → bundled
Eth3 → suspended
Eth4 → bundled
```

Connectivity vẫn chạy.

Nhưng:

- redundancy giảm;
- capacity giảm;
- risk tăng.

Do đó trạng thái:

```text
service is up
```

không đồng nghĩa:

```text
infrastructure is healthy
```

---

# PHẦN XIII — INTERFACE HEALTH, SPEED, DUPLEX VÀ ERRORS

# 79. Interface operational states

Một interface thường có administrative state và operational state.

Ví dụ:

```text
administratively down
```

nghĩa là configuration đã disable interface.

Khác với:

```text
down
```

do physical/link issue.

---

# 80. Up/up

Simplified:

```text
Physical/link = up
Protocol = up
```

thường là healthy sign.

Nhưng không đủ.

Một link có thể:

```text
up/up
```

mà có:

```text
millions of CRC errors
```

và performance rất kém.

---

# 81. Administratively down

Điều này thường có nghĩa interface bị shutdown bằng configuration.

Trong production:

> Không được thấy `administratively down` rồi lập tức `no shutdown`.

Có thể interface intentionally disabled vì:

- security;
- unused port;
- maintenance;
- failed equipment;
- change freeze;
- reserved connection.

Cần ticket/topology/owner confirmation.

---

# 82. Speed

Ví dụ:

```text
1 Gbps
10 Gbps
25 Gbps
40 Gbps
100 Gbps
400 Gbps
```

Tùy hardware.

Mismatch hoặc unsupported optic/cabling có thể gây:

- link down;
- negotiation failure;
- degraded behavior.

---

# 83. Duplex

Ethernet modern enterprise links thường full duplex.

Historically:

```text
half duplex
full duplex
```

Duplex mismatch có thể gây:

- low throughput;
- errors;
- retransmissions;
- collisions-related symptoms trên legacy scenarios.

Với modern high-speed Ethernet, problem thường liên quan optics/cabling/FEC/platform compatibility hơn, nhưng Day 8 vẫn cần hiểu duplex fundamentals.

---

# 84. CRC errors

CRC/FCS-related receive errors nghĩa là frames bị phát hiện corrupt.

Potential causes:

```text
Bad cable
Bad optic
Dirty fiber
Physical signal issue
Remote-side problem
Hardware issue
Some mismatch/compatibility problem
```

Không được kết luận:

```text
CRC → bad switch
```

Cần correlate counters ở cả hai ends.

---

# 85. Input errors

`input errors` thường là aggregate/category counter tùy platform.

Nó có thể bao gồm nhiều error types.

Do đó:

> Không chỉ đọc total counter; phải xem sub-counters và trend.

---

# 86. Output drops

Output drops có thể xuất hiện khi:

```text
traffic arrives faster than egress can transmit
```

hoặc queue/resource pressure.

Có thể liên quan congestion chứ không phải cable failure.

---

# 87. Counter delta quan trọng hơn absolute count

Ví dụ interface uptime:

```text
3 years
```

CRC:

```text
15
```

có thể ít đáng lo.

Nhưng:

```text
CRC 15 → 5000
```

trong 30 giây là critical signal.

Do đó evidence nên capture:

```text
T1 counter
T2 counter
Delta
Interval
```

---

# PHẦN XIV — LLDP VÀ CDP

# 88. Neighbor discovery

Trong thực tế documentation có thể outdated.

Bạn cần xác minh physical topology.

Hai protocols rất hữu ích:

```text
CDP
LLDP
```

---

# 89. CDP

CDP = **Cisco Discovery Protocol**.

Cisco proprietary Layer 2 discovery protocol.

Có thể cung cấp:

- neighbor device ID;
- local interface;
- remote port;
- platform;
- capabilities;
- management information tùy configuration.

Cisco vẫn duy trì CDP trong current IOS XE documentation. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr2-fwd/cdp-lldp-mac-udld/cdp-lldp-mac-udld-configuration-guide/c-configure-cdp.html?utm_source=chatgpt.com)

---

# 90. LLDP

LLDP = **Link Layer Discovery Protocol**.

IEEE standard, hữu ích trong multi-vendor environment.

Nó có thể cho biết:

```text
Local port
Neighbor
Neighbor port
System name
Capabilities
Management information
```

Cisco hiện hành hỗ trợ LLDP trên Catalyst và mô tả việc enable/verify theo platform/release. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/c9000/lyr2-fwd/cdp-lldp-mac-udld/cdp-lldp-mac-udld-configuration-guide/c-configure-lldp.html?utm_source=chatgpt.com)

---

# 91. LLDP/CDP trong troubleshooting

Suppose topology says:

```text
CAT1 Gi1/0/48
→ CAT2 Gi1/0/48
```

Nhưng CDP cho:

```text
CAT1 Gi1/0/48
→ CAT3 Gi1/0/47
```

Bạn vừa phát hiện:

```text
documentation mismatch
```

Không nên sửa configuration dựa trên diagram cũ mà bỏ qua actual neighbor evidence.

---

# 92. Security implication

Discovery protocols có thể leak:

- hostname;
- platform;
- management information;
- topology metadata.

Vì vậy organization có thể disable chúng trên untrusted-facing ports.

Không tự ý enable CDP/LLDP chỉ để troubleshoot nếu policy không cho phép.

---

# PHẦN XV — CISCO CATALYST VÀ IOS/IOS XE

# 93. Catalyst là gì?

Cisco Catalyst là family switch/router-oriented enterprise networking products, thường thấy nhiều trong:

- campus;
- branch;
- enterprise aggregation/core;
- access switching;
- wired infrastructure.

Không nên coi toàn bộ Catalyst models giống nhau.

Một Catalyst 9200, 9300, 9400, 9500, 9600 có:

- hardware architecture khác;
- interface capabilities khác;
- supported features khác;
- license khác;
- software releases khác.

Vì vậy syllabus yêu cầu version-aware operations.

---

# 94. IOS vs IOS XE

Cisco IOS là network operating system truyền thống.

IOS XE là architecture mới hơn và hiện được dùng rộng trên modern Catalyst platforms.

CLI có tính quen thuộc rất cao với IOS-style syntax.

Nhưng:

> "IOS-like syntax" không có nghĩa feature behavior identical trên mọi platform/release.

---

# 95. CLI hierarchy

Typical IOS/IOS XE:

```text
Switch>
```

User EXEC.

Sau:

```text
Switch#
```

Privileged EXEC.

Configuration:

```text
Switch(config)#
```

Interface:

```text
Switch(config-if)#
```

Ví dụ:

```text
Switch> enable
Switch# configure terminal
Switch(config)# interface GigabitEthernet1/0/10
Switch(config-if)#
```

---

# 96. Read-only vs changing commands

Read-only examples:

```text
show version
show interfaces
show vlan brief
show ip route
show spanning-tree
```

Configuration-changing:

```text
configure terminal
interface ...
switchport ...
shutdown
no shutdown
ip route ...
```

Infrastructure Operations phải luôn nhận thức mình đang ở mode nào.

Prompt:

```text
Switch#
```

khác:

```text
Switch(config)#
```

---

# 97. Catalyst command categories cần biết

Bạn cần xây mental command map:

```text
Identity
Interfaces
VLAN
Trunk
MAC
ARP
STP
EtherChannel
Routing
Neighbors
Configuration
```

Không cần học mọi command Cisco.

Mục tiêu:

> biết mình cần kiểm tra object nào.

---

# PHẦN XVI — CISCO NEXUS VÀ NX-OS

# 98. Nexus là gì?

Cisco Nexus là switching family thường tập trung vào Data Center environments.

Common operational areas:

- high-speed Ethernet;
- Data Center switching;
- server connectivity;
- virtualization integration;
- port-channel;
- vPC;
- Layer 2/3;
- management isolation.

---

# 99. NX-OS

NX-OS là network operating system dùng trên nhiều Nexus platforms.

CLI nhìn quen với IOS nhưng không được coi như IOS clone.

Cisco vPC documentation hiện hành thậm chí explicit cảnh báo người quen IOS rằng NX-OS commands cho feature này có thể khác IOS. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/interfaces/cisco-nexus-9000-series-nx-os-interfaces-configuration-guide-release-105x/m_configuring_vpcs_9x.html?utm_source=chatgpt.com)

---

# 100. NX-OS feature awareness

Một số functionality trên NX-OS có thể cần feature enablement.

Ví dụ Nexus 9000 documentation mô tả SVI functionality gắn với `interface-vlan` feature trong các releases liên quan. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/configuration/interfaces/cisco-nexus-9000-nx-os-interfaces-configuration-guide-103x/b-cisco-nexus-9000-nx-os-interfaces-configuration-guide-93x_chapter_0101.html?utm_source=chatgpt.com)

Điều này minh họa nguyên tắc:

> Không copy Catalyst config sang Nexus vì command trông gần giống.

---

# 101. Catalyst vs Nexus operational thinking

Không nên tạo một runbook:

```text
Cisco Switch Commands
```

rồi dùng chung.

Nên có:

```text
Catalyst IOS/IOS XE Runbook

Nexus NX-OS Runbook
```

Mỗi runbook ghi:

```text
Platform
Model
Release
Feature
Read-only command
Expected output
Failure indicators
Change syntax
Rollback
Known limitations
```

Đây là requirement explicit của syllabus. Infrastructure_Operations_14_Da…

---

# PHẦN XVII — VRF VÀ MANAGEMENT VRF

# 102. VRF là gì?

VRF = **Virtual Routing and Forwarding**.

Một physical router/switch có thể duy trì nhiều independent routing tables.

Concept:

```text
Device
├── VRF BLUE
│   └── Routing Table BLUE
│
├── VRF RED
│   └── Routing Table RED
│
└── VRF MANAGEMENT
    └── Routing Table MANAGEMENT
```

Hai VRFs có thể thậm chí sử dụng overlapping IP spaces tùy design.

---

# 103. Tại sao cần VRF?

Use cases:

- tenant separation;
- management separation;
- service separation;
- overlapping address spaces;
- control of routing domains.

---

# 104. Nexus management VRF

Đây là knowledge quan trọng Day 8.

Current Nexus 9000 documentation mô tả:

```text
default VRF
management VRF
```

là các routing contexts riêng; management VRF dùng cho management purposes, và `mgmt0` thuộc management context trên documented Nexus 9000 behavior. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/103x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide-release-103x/m_configuring_layer_3_virtualization.html?utm_source=chatgpt.com)

Do đó:

```text
Data-plane route works
```

không đồng nghĩa:

```text
Management-plane route works
```

---

# 105. Classic management troubleshooting trap

Administrator:

```text
ping 10.100.0.10
```

fails.

Kết luận:

> "Network unreachable."

Nhưng destination có thể cần được looked up trong:

```text
management VRF
```

Trong khi default ping dùng default VRF.

Do đó trước test:

```text
Which source interface?
Which VRF?
Which routing table?
```

Exact NX-OS command syntax phải verify theo deployed release.

---

# 106. VRF không phải firewall

VRF tạo routing separation.

Nhưng không nên suy diễn:

```text
VRF = security firewall
```

Traffic leaking/inter-VRF routing có thể được configured.

Security policy vẫn cần controls thích hợp.

---

# PHẦN XVIII — vPC

# 107. vPC là gì?

vPC = **virtual PortChannel** trên Cisco Nexus.

Một downstream device có physical links tới hai Nexus switches:

```text
        Server/Switch
          |     |
          |     |
       Nexus-A Nexus-B
```

nhưng downstream nhìn chúng như một logical port-channel.

Cisco Nexus 9000 current documentation định nghĩa vPC chính xác theo mental model này và ghi nhận các lợi ích như Layer 2 multipathing, redundancy và use of parallel links. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/106x/configuration/interfaces/cisco-nexus-9000-series-nx-os-interfaces-configuration-guide-release-106x/m_configuring_vpcs_9x.html?utm_source=chatgpt.com)

---

# 108. Standalone port-channel vs vPC

## Standalone

```text
Server
 ||||
Nexus-A
```

Tất cả members terminate trên cùng một switch.

## vPC

```text
Server
 |  |
 |  |
Nexus-A    Nexus-B
```

Members terminate trên hai vPC peers.

Đây là difference cốt lõi.

---

# 109. vPC building blocks awareness

Bạn cần biết terms:

```text
vPC domain
vPC peer
peer-link
peer-keepalive
vPC member port
```

Day 8 chưa yêu cầu expert-level vPC failure recovery.

Nhưng phải biết chúng có vai trò khác nhau.

---

# 110. Peer-link

Peer-link là critical connectivity giữa vPC peers cho coordination và traffic/functions liên quan.

Không được coi peer-link như một ordinary user port-channel.

---

# 111. Peer-keepalive

Peer-keepalive giúp peers detect peer state theo cơ chế vPC.

Nó không đơn giản thay thế peer-link.

Trong production troubleshooting, mất peer-link/keepalive có thể có substantial consequences, nên ngoài basic read-only diagnosis thường cần escalation/runbook.

---

# 112. vPC consistency

Một số parameters giữa peers phải compatible.

Mismatch có thể làm:

- vPC inconsistent;
- member suspended;
- traffic impact.

Do đó vPC issue không nên "fix" bằng random config synchronization.

Cần xem:

```text
vPC status
Consistency parameters
Peer-link
Keepalive
Member state
Related STP/VLAN state
```

---

# PHẦN XIX — CONFIGURATION STATE: RUNNING VS STARTUP

# 113. Running configuration

`running-config` đại diện current active configuration.

Các configuration changes thường ảnh hưởng running configuration.

---

# 114. Startup configuration

`startup-config` được sử dụng khi boot/reload theo normal operating flow.

Vì vậy:

```text
running ≠ startup
```

là tình huống có thể tồn tại.

Cisco IOS XE và current NX-OS documentation đều phân biệt hai configuration states này. [spg.xgslb-v3.cisco.com](https://spg.xgslb-v3.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/syst-mgmt/b-system-management/m_cm-config-files-0.html?utm_source=chatgpt.com)

---

# 115. Tại sao khác biệt này nguy hiểm?

Giả sử:

```text
Tuesday:
fix trunk config
```

Connectivity restored.

Nhưng change không persisted.

Sunday switch reload.

Device boot từ old startup configuration.

Outage quay lại.

Do đó verification phải hỏi:

```text
Is the running state correct?
Is intended configuration persisted?
```

---

# 116. Nhưng cũng không được tự động save mọi thay đổi

Có trường hợp temporary troubleshooting change purposely chỉ nằm trong running configuration.

Vì vậy hành động:

```text
copy running-config startup-config
```

là **configuration-changing/persistence action**.

Không chạy chỉ vì thấy difference.

Phải xem:

- change ticket;
- approval;
- intended final state;
- rollback plan.

Current Cisco IOS XE và NX-OS documentation đều dùng `copy running-config startup-config` để persist running state vào startup state. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9600/software/release/26-x/configuration_guide/sys_mgmt/b_26x_sys_mgmt_9600_cg.pdf?utm_source=chatgpt.com)

---

# PHẦN XX — CONFIGURATION BACKUP

# 117. Backup không giống save

Đây là distinction quan trọng.

### Save/persist

```text
running
→
startup
```

### Backup

```text
device configuration
→
separate trusted storage
```

Backup dùng cho:

- audit;
- baseline;
- change comparison;
- disaster recovery;
- rollback support.

---

# 118. Một useful configuration backup phải có context

Không chỉ file:

```text
switch.cfg
```

Nên có:

```text
Hostname
Asset ID
Platform/model
Software version
Environment
Timestamp/timezone
Ticket/change ID
Running/startup indicator
Collector
File hash
Storage classification
```

Điều này phù hợp common evidence contract của toàn chương trình. Infrastructure_Operations_14_Da…

---

# 119. Raw evidence

Configuration backup gốc nên:

```text
preserved untouched
```

Nếu cần redact secrets:

```text
Raw original
↓
Working copy
↓
Redacted copy
```

không overwrite original.

---

# 120. Sanitized configuration

Một số modern NX-OS releases có capability tạo sanitized running configuration theo documentation, nhưng availability phụ thuộc release. Không được assume một command/path này tồn tại trên mọi Nexus. [Team Development](https://team-development.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/106x/configuration/fundamentals/cisco-nexus-9000-series-nx-os-fundamentals-configuration-guide-release-106x/m-working-with-configuration-files.html?utm_source=chatgpt.com)

Điều quan trọng:

> Không đưa plaintext credentials/secrets vào training evidence.

---

# PHẦN XXI — FIRMWARE / NETWORK OS UPGRADE PRECAUTIONS

# 121. Tại sao switch upgrade là high risk?

Network switch có thể là dependency của:

```text
hundreds/thousands endpoints
servers
hypervisors
firewalls
storage
other switches
```

Sai upgrade có blast radius lớn.

Day 8 chỉ yêu cầu hiểu precautions, không yêu cầu tự ý upgrade production.

---

# 122. Pre-upgrade questions

Trước upgrade phải biết:

```text
Current platform?
Current version?
Target version?
Supported upgrade path?
Bootloader/ROM requirements?
Hardware compatibility?
Feature compatibility?
Licensing?
Known defects?
Expected downtime?
Redundant peer?
Traffic impact?
```

---

# 123. Prechecks

Một generic safe precheck set:

```text
Device health
CPU/memory
Interface status
Port-channel
STP
Routing
Neighbors
Redundancy
Power/fans
Storage space
Current image
Current boot settings
Configuration backup
Logs/alarms
```

Không upgrade unhealthy switch mà chưa hiểu condition.

---

# 124. Rollback limitation

Không được nói:

> "Nếu lỗi thì downgrade."

Rollback software đôi khi:

- unsupported;
- disruptive;
- incompatible với changed configuration;
- require multiple steps;
- alter database/state;
- impossible within outage budget.

Vì vậy rollback phải dựa vendor-supported release-specific procedure.

---

# 125. Maintenance window

Upgrade production cần:

```text
Approved change
Maintenance window
Stakeholders
Expected impact
Console/out-of-band access
Backout plan
Stop conditions
Post-check
Escalation contacts
```

Không có các prerequisites này thì operator nên stop.

---

# PHẦN XXII — NETWORK CHANGE CONTROL

# 126. Network change là gì?

Examples:

- VLAN addition;
- access VLAN change;
- trunk allowed VLAN change;
- static route;
- SVI;
- ACL;
- STP priority;
- port-channel;
- interface shutdown/no shutdown;
- software upgrade.

Ngay cả một dòng:

```text
switchport access vlan 20
```

có thể disconnect production server.

---

# 127. Minimum change lifecycle

```text
Request
  ↓
Impact assessment
  ↓
Approval
  ↓
Baseline
  ↓
Backup
  ↓
Implementation
  ↓
Verification
  ↓
Success?
 ┌──┴───┐
Yes     No
 ↓       ↓
Post    Rollback
checks   ↓
        Verify
```

---

# 128. Pre-check

Phải chứng minh:

```text
service/network is in known state before change
```

Ví dụ:

```text
Interface currently up
VLAN exists
Trunk current VLAN list
Route current state
STP current root
Port-channel members healthy
```

Nếu không có pre-check, sau change rất khó chứng minh issue mới phát sinh do change hay đã tồn tại.

---

# 129. Stop conditions

Một change plan tốt phải trả lời:

> Khi nào tôi dừng và rollback?

Ví dụ:

```text
Unexpected loss of management
Unexpected STP root change
Unexpected multiple VLAN outage
Port-channel loses redundancy
Routing table changes outside scope
Critical alarms appear
```

---

# PHẦN XXIII — SECURITY VÀ LEAST PRIVILEGE TRONG NETWORK OPERATIONS

# 130. Read-only trước, change sau

Troubleshooting tốt bắt đầu bằng:

```text
show
inspect
compare
collect
```

trước:

```text
configure
clear
restart
shutdown
reload
```

Đây là risk-minimization principle.

---

# 131. Least-privilege administrative access

Không phải network engineer nào cũng cần unrestricted privileged access.

Có thể tách:

```text
Read-only NOC
Change operator
Network admin
Security admin
Platform admin
```

Tùy enterprise RBAC model.

---

# 132. Management plane

Network device management nên được protect:

- SSH thay vì insecure cleartext protocol;
- AAA;
- RBAC;
- management ACL;
- management VRF/network;
- centralized logging;
- NTP;
- audit;
- approved jump host.

Day 8 không yêu cầu thiết kế AAA, nhưng phải có management-plane security awareness.

---

# 133. Management VRF không thay thế authentication

Tách management routing table hữu ích.

Nhưng vẫn cần:

```text
Identity
Authentication
Authorization
Logging
```

---

# 134. Configuration evidence có thể chứa sensitive data

`show running-config` có thể chứa:

- usernames;
- community strings;
- pre-shared material;
- hashes;
- management IPs;
- topology details.

Vì vậy:

> Không paste full production config vào public ticket/chat nếu không được phép.

---

# PHẦN XXIV — L2 TROUBLESHOOTING METHODOLOGY

# 135. Nguyên tắc lớn

Không change-first.

Workflow:

```text
Observe
↓
Scope
↓
Hypothesize
↓
Collect evidence
↓
Test low-risk hypothesis
↓
Identify root cause
↓
Authorized correction
↓
Verify
```

---

# 136. Step 1 — Scope

Hỏi:

```text
One user?
One port?
One VLAN?
One switch?
One site?
All VLANs?
Both directions?
Intermittent or constant?
```

Scope giúp khoanh layer.

---

# 137. Step 2 — Physical/interface

Check:

```text
Interface status
Administrative state
Speed
Duplex
Error counters
Transceiver/cable indicators
Neighbor
```

Nếu Layer 1 fail thì chưa cần route analysis.

---

# 138. Step 3 — VLAN

Check:

```text
VLAN exists?
Port in expected VLAN?
VLAN active?
```

---

# 139. Step 4 — Trunk

Check:

```text
Trunk operational?
Correct native VLAN?
Required VLAN allowed?
Required VLAN actually forwarding?
Both ends consistent?
```

---

# 140. Step 5 — STP

Check:

```text
Port forwarding?
Blocked/discarding?
Correct root?
Unexpected topology change?
```

---

# 141. Step 6 — Port-channel

Check:

```text
Bundle up?
All expected members bundled?
Suspended member?
Config consistency?
```

---

# 142. Step 7 — MAC learning

Check:

```text
Expected MAC learned?
Correct VLAN?
Correct interface?
MAC flapping?
```

Nếu MAC không học được, investigate lower layers before routing.

---

# PHẦN XXV — L3 TROUBLESHOOTING METHODOLOGY

# 143. IP configuration

Verify:

```text
IP address
Subnet mask
Gateway
```

Không tin ticket description tuyệt đối.

---

# 144. Local subnet reachability

Can endpoint reach:

```text
itself
local peer
gateway
```

Mỗi result có diagnostic value khác nhau.

---

# 145. ARP

Check:

```text
Does host know gateway MAC?
Does gateway know host MAC?
```

Incomplete/missing ARP often points back toward Layer 2.

---

# 146. SVI/L3 interface

Verify:

```text
Configured IP
Operational state
Correct VLAN
Correct VRF
```

---

# 147. Routing table

For failing destination:

```text
Find matching route
↓
Determine longest-prefix match
↓
Identify next hop
↓
Validate next-hop reachability
```

---

# 148. Return path

On destination-side routing device:

```text
Is there route back to source?
```

Nếu không, forward request có thể đến nhưng reply không quay về.

---

# 149. ACL/policy

Only after route/path reasonably established:

```text
ACL?
Firewall?
Other policy?
```

Tránh blame firewall trước khi chứng minh Layer 2/3.

---

# PHẦN XXVI — FAILURE SCENARIO 1: VLAN MISMATCH

Giả sử:

```text
PC-A
IP 10.10.10.20/24
Gateway 10.10.10.1
```

Expected port:

```text
VLAN 10
```

Actual:

```text
VLAN 20
```

Physical:

```text
up/up
```

Symptom:

```text
Cannot reach gateway
```

Reason:

PC broadcasts ARP trong VLAN 20.

Gateway `10.10.10.1` nằm VLAN 10.

ARP không tới được gateway.

Troubleshooting chain:

```text
Interface up
✓

Correct IP
✓

ARP gateway
✗

Check VLAN
↓
Wrong VLAN
```

---

# PHẦN XXVII — FAILURE SCENARIO 2: VLAN MISSING FROM TRUNK

Topology:

```text
PC-A
 |
SW1
 |
trunk
 |
SW2
 |
Gateway
```

VLAN 10 được cấu hình access ở SW1 và gateway ở SW2.

Nhưng trunk:

```text
Allowed VLANs:
20,30
```

VLAN 10 không traverse.

Symptoms:

- local MAC learning có thể xuất hiện SW1;
- interface up;
- trunk up;
- gateway unreachable;
- SW2 không learn endpoint MAC through trunk.

Đây là lý do cần correlate:

```text
Access VLAN
Trunk allowed list
MAC table both sides
STP
```

---

# PHẦN XXVIII — FAILURE SCENARIO 3: WRONG STATIC ROUTE

Routing table:

```text
10.20.0.0/16 via 10.10.30.3
```

Correct next hop phải:

```text
10.10.30.2
```

Symptom:

```text
Gateway reachable
Local networks reachable
Remote subnet fails
```

Troubleshooting:

```text
L2 healthy
↓
Gateway healthy
↓
Route exists
↓
But next hop incorrect
```

Lesson:

> "Route exists" chưa đủ.

---

# PHẦN XXIX — FAILURE SCENARIO 4: INTERFACE DOWN

Possible states:

```text
administratively down
```

hoặc:

```text
physical down
```

Các root cause khác nhau.

### Administrative

Possible:

- intentional shutdown;
- incomplete change;
- security action;
- configuration mistake.

### Physical

Possible:

- cable;
- optic;
- remote interface;
- power;
- hardware;
- unsupported media.

Không dùng cùng remediation cho cả hai.

---

# PHẦN XXX — FAILURE SCENARIO 5: PORT-CHANNEL DEGRADED

Expected:

```text
Po10:
Eth1
Eth2
Eth3
Eth4
```

Actual:

```text
Eth1 bundled
Eth2 bundled
Eth3 suspended
Eth4 bundled
```

Service vẫn reachable.

Nếu monitoring chỉ hỏi:

```text
Can I ping?
```

result:

```text
PASS
```

nhưng operational health:

```text
DEGRADED
```

Infrastructure Operations phải phân biệt:

```text
Availability
vs
Health
vs
Redundancy
```

---

# PHẦN XXXI — CATALYST READ-ONLY COMMAND MAP

Đây là command-oriented theory để chuẩn bị cho Module Thực hành. Exact availability/output vẫn phải kiểm tra platform/release.

### Identity/version

```text
show version
```

### Interface overview

```text
show interfaces status
```

### Detailed interface

```text
show interfaces <interface>
```

### VLAN

```text
show vlan brief
```

### Trunk

```text
show interfaces trunk
```

### MAC

```text
show mac address-table
```

### ARP

Common IOS/IOS XE forms include:

```text
show ip arp
```

### STP

```text
show spanning-tree
```

### EtherChannel

```text
show etherchannel summary
```

### Routing

```text
show ip route
```

### CDP

```text
show cdp neighbors
show cdp neighbors detail
```

### LLDP

```text
show lldp neighbors
show lldp neighbors detail
```

### Running config

```text
show running-config
```

### Startup config

```text
show startup-config
```

Các command `show` trên là mục tiêu chủ yếu read-only, dù một số outputs có thể chứa sensitive data.

---

# PHẦN XXXII — NEXUS READ-ONLY COMMAND MAP

Common NX-OS categories:

### Version

```text
show version
```

### Interface status

```text
show interface status
```

### Detailed interface

```text
show interface ethernet 1/1
```

### VLAN

```text
show vlan brief
```

### Trunk

```text
show interface trunk
```

### MAC

```text
show mac address-table
```

Current NX-OS documentation vẫn sử dụng `show mac address-table` để inspect forwarding entries. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/102x/command-reference/show/b_n9k_show_commands_1021/m_m_showcmds.html?utm_source=chatgpt.com)

### ARP

```text
show ip arp
```

### STP

```text
show spanning-tree
```

### Port-channel

```text
show port-channel summary
```

### Routing

```text
show ip route
```

### VRF

Common inspection includes:

```text
show vrf
```

và VRF-specific routing checks theo release syntax.

### vPC

```text
show vpc brief
```

### Running config

```text
show running-config
```

### Startup config

```text
show startup-config
```

### Neighbors

```text
show cdp neighbors
show lldp neighbors
```

Exact command options/output fields phải lấy từ deployed NX-OS documentation.

---

# PHẦN XXXIII — CATALYST VS NEXUS: ĐIỀU PHẢI NHỚ

| Area                      | Catalyst                               | Nexus                               |
| ------------------------- | -------------------------------------- | ----------------------------------- |
| Typical context           | Campus/enterprise                      | Data Center                         |
| Common OS                 | IOS/IOS XE                             | NX-OS                               |
| L2 switching              | Yes                                    | Yes                                 |
| L3 switching              | Platform-dependent capabilities        | Platform-dependent capabilities     |
| VLAN/trunk                | Yes                                    | Yes                                 |
| Link aggregation          | EtherChannel/port-channel              | port-channel                        |
| LACP                      | Yes                                    | Yes                                 |
| vPC                       | Không phải general Catalyst equivalent | Major Nexus technology              |
| Management VRF            | Platform/config dependent              | Important Nexus operational concept |
| CLI                       | IOS-style                              | NX-OS                               |
| Feature enablement        | Platform specific                      | Important for some NX-OS features   |
| Configuration persistence | running/startup distinction            | running/startup distinction         |
| Runbook                   | Separate                               | Separate                            |

Không dùng bảng này để suy luận mọi feature. Syllabus yêu cầu **actual release-specific verification**. Infrastructure_Operations_14_Da…

---

# PHẦN XXXIV — OPERATIONAL EVIDENCE

Day 8 không chỉ yêu cầu "biết command".

Bạn cần biết evidence nào chứng minh kết luận của mình.

## Device identity evidence

```text
Hostname
Model
Serial/asset reference if authorized
OS
Version
Timestamp
```

## Layer 1

```text
Interface state
Speed
Duplex
Error counters
Transceiver evidence if needed
```

## Layer 2

```text
VLAN state
Trunk state
MAC table
STP
Port-channel
LLDP/CDP
```

## Layer 3

```text
IP interface
ARP
Routing table
VRF
Reachability test
```

## Configuration

```text
Relevant running config
Startup config
Backup
Before/after diff
```

## Troubleshooting

```text
Symptom
Expected state
Actual state
Hypothesis
Evidence
Root cause
Correction
Verification
```

Syllabus yêu cầu riêng **một Catalyst evidence set và một Nexus evidence set**, có platform/version, tested paths, root-cause reasoning, configuration backup và rollback verification. Infrastructure_Operations_14_Da…

---

# PHẦN XXXV — ROOT CAUSE VS SYMPTOM

Infrastructure Operations phải phân biệt hai khái niệm.

Ví dụ:

```text
Symptom:
Server cannot ping gateway
```

Không phải root cause.

Evidence:

```text
ARP incomplete
```

vẫn chưa chắc root cause.

Further evidence:

```text
Server MAC learned VLAN 30
Expected VLAN 20
```

Root cause:

```text
Access interface assigned to wrong VLAN
```

Một incident report tốt phải đi tới mức này.

---

# PHẦN XXXVI — TROUBLESHOOTING ORDER: LOW RISK → HIGH RISK

## Level 1 — Read-only

```text
show commands
ping
traceroute where authorized
logs
monitoring
topology review
ticket review
```

## Level 2 — Controlled state inspection

Ví dụ:

- compare configs;
- capture counters twice;
- inspect neighboring device;
- test alternate known-good endpoint.

## Level 3 — Reversible configuration correction

Chỉ sau:

```text
Root cause confidence
Approval
Backup
Rollback
```

## Level 4 — Disruptive action

Ví dụ:

```text
interface bounce
reload
software upgrade
major STP change
vPC modification
```

thường phải escalation/change window.

---

# PHẦN XXXVII — ESCALATION BOUNDARIES

## L1/L2 thường có thể làm

Tùy organization/runbook:

```text
Read-only status checks
Collect interface counters
Inspect VLAN/trunk
Inspect MAC/ARP
Inspect routing
Inspect STP
Inspect port-channel
Ping approved endpoints
Collect config/log evidence
Correlate topology
Identify likely root cause
```

## Có thể làm correction nếu explicitly authorized

Ví dụ lab hoặc documented runbook:

```text
Correct access VLAN
Restore approved trunk VLAN
Correct approved static route
Restore intentionally disabled test interface
```

## Escalate khi

```text
Production core/distribution impact
Unknown blast radius
STP instability
Possible Layer 2 loop
vPC peer/peer-link failure
Multiple network devices affected
Hardware/optic replacement required
Firmware defect suspected
Upgrade/downgrade needed
Unauthorized ACL/routing scope
Loss of management access
Configuration baseline unreliable
No tested rollback
Security incident suspected
```

---

# PHẦN XXXVIII — MENTAL MODEL HOÀN CHỈNH CỦA MỘT PACKET

Đây là phần quan trọng nhất để kết nối toàn bộ Day 8.

Giả sử:

```text
PC-A
IP: 10.10.10.10/24
MAC: AAAA
GW: 10.10.10.1

Server-B
IP: 10.20.20.20/24
MAC: BBBB
GW: 10.20.20.1
```

Topology:

```text
PC-A
 |
Access VLAN10
 |
SW1
 |
802.1Q Trunk
 |
L3 Switch
 |
VLAN20
 |
Server-B
```

PC-A evaluates:

```text
10.20.20.20
```

against:

```text
10.10.10.0/24
```

Different subnet.

Therefore use:

```text
10.10.10.1
```

PC-A checks ARP:

```text
10.10.10.1 → ?
```

Nếu missing:

```text
ARP Request
```

broadcast trong VLAN10.

Switch learns:

```text
AAAA → PC port, VLAN10
```

Gateway replies.

PC builds frame:

```text
Destination MAC = Gateway MAC
Source MAC      = AAAA
```

Packet inside:

```text
Destination IP = 10.20.20.20
Source IP      = 10.10.10.10
```

Switches forward VLAN10 frame based on MAC.

Trunk transports VLAN10 using 802.1Q.

L3 switch receives packet at VLAN10 SVI.

Routing table lookup:

```text
10.20.20.0/24
```

selects connected VLAN20 route.

L3 switch resolves Server-B MAC via ARP if necessary.

New Ethernet frame:

```text
Source MAC      = VLAN20 SVI MAC
Destination MAC = BBBB
```

IP packet remains logically:

```text
10.10.10.10 → 10.20.20.20
```

Server receives it.

Return reply:

```text
10.20.20.20 → 10.10.10.10
```

uses its gateway:

```text
10.20.20.1
```

and routing occurs back toward VLAN10.

Nếu bạn hiểu trọn flow này, bạn có foundation rất tốt để troubleshoot phần lớn Day 8.

---

# PHẦN XXXIX — MỘT TROUBLESHOOTING DECISION TREE HOÀN CHỈNH

Khi endpoint A không reach endpoint B:

```text
START
 │
 ├─ Interface physically up?
 │     ├─ No → Layer 1 / admin state
 │     └─ Yes
 │
 ├─ Errors increasing?
 │     ├─ Yes → physical/link investigation
 │     └─ No
 │
 ├─ Correct access VLAN?
 │     ├─ No → VLAN root cause
 │     └─ Yes
 │
 ├─ Required VLAN crosses trunks?
 │     ├─ No → trunk root cause
 │     └─ Yes
 │
 ├─ STP forwarding?
 │     ├─ Unexpected no → STP/topology investigation
 │     └─ Yes
 │
 ├─ Port-channel healthy?
 │     ├─ No → aggregation investigation
 │     └─ Yes
 │
 ├─ Expected MAC learned?
 │     ├─ No → return to L1/L2
 │     └─ Yes
 │
 ├─ Same subnet?
 │     ├─ Yes → ARP/direct L2
 │     └─ No
 │
 ├─ Correct gateway?
 │
 ├─ Gateway ARP works?
 │
 ├─ SVI/L3 interface operational?
 │
 ├─ Correct VRF?
 │
 ├─ Destination route exists?
 │
 ├─ Correct longest-prefix match?
 │
 ├─ Correct next hop?
 │
 ├─ Return route exists?
 │
 ├─ ACL/policy blocking?
 │
 └─ If L3 verified → escalate/continue Layer4+ investigation
```

Đây là troubleshooting logic bạn nên internalize thay vì memorizing random commands.

---

# PHẦN XL — COMMON MISTAKES CẦN TRÁNH

### 1. Thấy `up/up` rồi kết luận network tốt

Sai vì VLAN/STP/routing vẫn có thể lỗi.

### 2. Ping fail rồi blame firewall

Sai vì lower layers chưa được verified.

### 3. Route tồn tại = routing đúng

Sai vì next hop/prefix/VRF/return route có thể sai.

### 4. Không có ARP = ARP protocol lỗi

Sai vì root cause có thể là VLAN/trunk.

### 5. Port STP blocking = problem

Không nhất thiết. Có thể STP đang hoạt động đúng.

### 6. Port-channel up = tất cả members tốt

Sai. Bundle có thể degraded.

### 7. `running-config` đúng = reboot-safe

Sai nếu intended state chưa persisted.

### 8. Nexus giống Catalyst

Sai và syllabus explicitly cấm assumption này. Infrastructure_Operations_14_Da…

### 9. Management ping fail = production routing fail

Sai nếu VRF/source context khác.

### 10. Configuration backup = copy/paste vài dòng

Không đủ cho audit/rollback.

### 11. `no shutdown` vì thấy port down

Nguy hiểm nếu shutdown là intentional control.

### 12. Disable STP vì port blocked

Có thể tạo catastrophic Layer 2 loop.

### 13. Allow every VLAN để khắc phục trunk

Vi phạm least privilege và có thể mở rộng blast radius.

### 14. Save running-config ngay sau temporary troubleshooting

Có thể persist một state không intended.

---

# PHẦN XLI — KIẾN THỨC BỔ SUNG NGOÀI MỨC TỐI THIỂU CỦA SYLLABUS

Các phần sau hỗ trợ hiểu syllabus nhưng không phải các advanced topics bắt buộc của Day 8:

### Control plane

Các protocols và processes quyết định network behavior:

```text
STP
routing protocols
LACP control
ARP/control functions
```

### Data plane

Thực hiện actual forwarding:

```text
MAC forwarding
IP forwarding
```

### Management plane

Administrator quản trị device thông qua:

```text
SSH
API
SNMP
syslog
NTP
management interface
```

Mental model này rất hữu ích.

Một network có thể:

```text
Data plane healthy
Management plane broken
```

hoặc ngược lại.

---

# PHẦN XLII — DAY 8 LÝ THUYẾT: CHEAT SHEET

## L2

```text
Ethernet → MAC
Switch learns source MAC
MAC table → MAC → interface
VLAN → Layer 2 broadcast domain
Access → usually one VLAN
Trunk → multiple VLANs
802.1Q → VLAN tagging
STP/RSTP → loop prevention
EtherChannel/port-channel → link aggregation
LLDP/CDP → neighbor discovery
```

## L3

```text
IPv4 → 32 bits
Subnet → local vs remote decision
Gateway → next hop for remote subnet
ARP → IPv4 → MAC
SVI → Layer 3 interface for VLAN
Routing table → destination forwarding
Longest prefix match → most specific route wins
Static route → administrator-defined path
Default route → 0.0.0.0/0
ACL → permit/deny policy
```

## Catalyst

```text
IOS/IOS XE
Campus/enterprise focus
Separate runbook
EtherChannel terminology
running/startup config
```

## Nexus

```text
NX-OS
Data Center focus
Separate runbook
VRF awareness
management VRF
port-channel
vPC
running/startup config
```

## Troubleshooting

```text
Physical
↓
Interface
↓
VLAN
↓
Trunk
↓
STP
↓
Port-channel
↓
MAC
↓
IP/mask/gateway
↓
ARP
↓
SVI
↓
VRF
↓
Route
↓
Return route
↓
ACL
↓
Upper layers
```

## Change discipline

```text
Approval
↓
Baseline
↓
Backup
↓
Change
↓
Verify
↓
Persist if intended
↓
Post-check
↓
Evidence
```

Nếu failed:

```text
Stop
↓
Rollback
↓
Verify rollback
↓
Escalate if needed
```

---

# Module Lý thuyết Day 8 — Completion Checklist

Sau phần này, về mặt lý thuyết bạn cần có thể tự giải thích được toàn bộ các mục sau:

- [ ] Enterprise network architecture và dependency chain.
- [ ] Access / Distribution / Core và Data Center awareness.
- [ ] OSI và TCP/IP theo troubleshooting perspective.
- [ ] Ethernet frame, MAC, broadcast, unknown unicast.
- [ ] Cơ chế MAC learning.
- [ ] MAC address table.
- [ ] Switching vs routing.
- [ ] IPv4 addressing.
- [ ] CIDR và subnetting.
- [ ] Same-subnet vs remote-subnet decision.
- [ ] Default gateway.
- [ ] ARP Request/Reply.
- [ ] ARP table vs MAC table.
- [ ] Network segmentation.
- [ ] VLAN.
- [ ] Access port.
- [ ] Trunk.
- [ ] 802.1Q.
- [ ] Native VLAN awareness.
- [ ] Allowed VLANs.
- [ ] SVI.
- [ ] Inter-VLAN routing.
- [ ] Routing table.
- [ ] Longest Prefix Match.
- [ ] Connected route.
- [ ] Static route.
- [ ] Default route.
- [ ] Next hop.
- [ ] Return path.
- [ ] ACL fundamentals.
- [ ] STP/RSTP.
- [ ] Root Bridge.
- [ ] Root/Designated/Alternate ports.
- [ ] STP/RSTP states.
- [ ] Layer 2 loop symptoms.
- [ ] EtherChannel/port-channel.
- [ ] LACP.
- [ ] PAgP awareness.
- [ ] Link aggregation limitations.
- [ ] Interface status.
- [ ] Speed/duplex.
- [ ] CRC/errors/drops.
- [ ] Counter trend/delta.
- [ ] LLDP/CDP.
- [ ] Catalyst role và IOS/IOS XE.
- [ ] Nexus role và NX-OS.
- [ ] Tại sao không dùng chung Catalyst/Nexus runbook.
- [ ] VRF.
- [ ] Nexus management VRF.
- [ ] vPC vs standalone port-channel.
- [ ] vPC peer / peer-link / peer-keepalive awareness.
- [ ] running-config vs startup-config.
- [ ] Save/persist vs configuration backup.
- [ ] Firmware/software upgrade precautions.
- [ ] Network change control.
- [ ] Read-only vs configuration-changing operations.
- [ ] Least privilege và management-plane security.
- [ ] L2 troubleshooting sequence.
- [ ] L3 troubleshooting sequence.
- [ ] VLAN mismatch diagnosis.
- [ ] Trunk VLAN failure diagnosis.
- [ ] Incorrect-route diagnosis.
- [ ] Interface-down diagnosis.
- [ ] Port-channel degraded-state diagnosis.
- [ ] Evidence collection.
- [ ] Root cause vs symptom.
- [ ] Rollback thinking.
- [ ] Escalation boundaries.

Đó là **toàn bộ Module Lý thuyết Day 8 theo scope của syllabus**, với phần bổ sung chỉ nhằm làm rõ cơ chế và operational reasoning. Phần syllabus gốc của Day 8 yêu cầu sau cùng người học phải chuyển các kiến thức này thành năng lực riêng trên **Catalyst IOS/IOS XE** và **Nexus NX-OS**, đặc biệt là interface errors, VLAN/trunk, L2 learning, L3 reachability, configuration backup/difference và rollback verification. Infrastructure_Operations_14_Da…

Bước tiếp theo phù hợp là **Module Thực hành Day 8**: chúng ta sẽ dựng topology Catalyst + Nexus, tạo VLAN/access/trunk/SVI/static route/port-channel, lập baseline, chạy verification riêng trên IOS XE và NX-OS, sau đó inject lần lượt **VLAN mismatch, incorrect route, interface down và port-channel/trunk faults**, troubleshoot theo đúng flow ở trên, cuối cùng thực hiện configuration backup, before/after diff, rollback và evidence package.
