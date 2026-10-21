# Day 8 — Module Thực hành

## Enterprise Switching & Routing Operations với Cisco Catalyst IOS/IOS XE và Cisco Nexus NX-OS

Theo syllabus, Module Thực hành Day 8 phải đưa bạn từ trạng thái **biết concept** sang khả năng thực hiện một workflow Infrastructure Operations hoàn chỉnh: dựng topology Catalyst/Nexus, cấu hình VLAN/access/trunk/L3/inter-VLAN/static route, inspect MAC/ARP/STP/port-channel/routing tables, thực hiện diagnosis riêng trên IOS/IOS XE và NX-OS, inject lỗi VLAN/route/interface, khôi phục connectivity, backup configuration, kiểm tra before/after difference, verify rollback và tạo evidence package. Infrastructure_Operations_14_Da…

Syllabus còn yêu cầu **Catalyst và Nexus phải có evidence set riêng**, ghi rõ platform/version, tested paths, root-cause reasoning, configuration backup và rollback verification; nếu dùng emulator thì phải ghi rõ giới hạn emulation. Infrastructure_Operations_14_Da…

Phần dưới đây xây thành **một integrated lab lớn**. Bạn có thể chạy toàn bộ hoặc tách thành nhiều session.

> **Safety boundary:** toàn bộ configuration-changing command dưới đây chỉ dành cho **lab/non-production/authorized environment**. Trong production, phải có approved change, impact assessment, configuration backup, maintenance window khi cần, stop condition và backout plan.

---

# PHẦN 1 — OBJECTIVE VÀ EXPECTED OUTCOMES

Sau khi hoàn tất Module Thực hành Day 8, bạn phải thực hiện được những việc sau mà không chỉ copy command:

1. Xác định chính xác platform, OS và software release trước khi thao tác.
2. Thu thập một read-only baseline của Catalyst.
3. Thu thập một read-only baseline riêng của Nexus.
4. Tạo và verify VLAN.
5. Cấu hình access port.
6. Cấu hình 802.1Q trunk.
7. Cấu hình EtherChannel/port-channel bằng LACP.
8. Verify port-channel members và trạng thái aggregation.
9. Cấu hình SVI.
10. Enable Layer 3 routing trên Catalyst phù hợp.
11. Cấu hình routed interface giữa Catalyst và Nexus.
12. Thực hiện inter-VLAN routing.
13. Cấu hình static routes.
14. Đọc routing table và dự đoán packet path.
15. Correlate IP → ARP → MAC → interface.
16. Inspect STP/RSTP.
17. Inspect LLDP/CDP neighbors.
18. Inspect interface health và error counters.
19. Nhận diện Catalyst IOS/IOS XE syntax khác NX-OS.
20. Hiểu management VRF trên Nexus.
21. Inspect vPC nếu topology hỗ trợ, nhưng không tự ý thay đổi vPC.
22. Troubleshoot VLAN mismatch.
23. Troubleshoot missing VLAN trên trunk.
24. Troubleshoot incorrect static route.
25. Troubleshoot interface-down.
26. Troubleshoot degraded port-channel.
27. Validate forward path và return path.
28. Backup running configuration.
29. Kiểm tra running vs startup state.
30. Tạo before/after configuration evidence.
31. Thực hiện rollback có kiểm soát.
32. Thu thập audit-quality evidence.
33. Biết khi nào phải stop và escalate.

---

# PHẦN 2 — REFERENCE PLATFORM VÀ VERSION AWARENESS

Các command dưới đây dùng syntax phổ biến được đối chiếu với tài liệu Cisco cho:

- **Catalyst 9300 / IOS XE 17.x**
- **Nexus 9000 / NX-OS 10.x**

Đây là **reference syntax**, không phải khẳng định mọi Catalyst/Nexus model có cùng behavior.

Ví dụ, Catalyst IOS XE documentation xác nhận SVI được tạo bằng `interface vlan`, Layer 3 routed port bằng `no switchport`, và IP routing phải được enable để switch route giữa networks trên những releases/platforms tương ứng. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-13/configuration_guide/rtng/b_1713_rtng_9300_cg/configuring_ip_unicast_routing.html?utm_source=chatgpt.com)

Trên Nexus 9000, một số Layer 3/SVI workflows yêu cầu `feature interface-vlan`; Nexus port-channel dùng NX-OS-specific behavior và LACP cần được enable bằng `feature lacp` trên các releases được tài liệu hóa. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/interfaces/cisco-nexus-9000-series-nx-os-interfaces-configuration-guide-release-105x/m_configuring_layer_3_interfaces_9x.html?utm_source=chatgpt.com)

Vì vậy bước đầu tiên của mọi lab là:

```text
show version
```

Không bỏ qua bước này.

---

# PHẦN 3 — REFERENCE LAB TOPOLOGY

Chúng ta sử dụng topology này:

```text
                   CAMPUS / CATALYST
                   =================

 PC-A
  |
  | VLAN 10
  |
Gi1/0/10
+----------------+
|   CAT-ACCESS   |
| Catalyst IOS-XE|
+----------------+
  || Gi1/0/47-48
  ||
  || LACP Port-channel1
  || 802.1Q trunk
  ||
+----------------+
|    CAT-DIST    |
| Catalyst IOS-XE|
+----------------+
  | SVI VLAN10 = 10.10.10.1/24
  | SVI VLAN20 = 10.10.20.1/24
  |
  | Routed Link
  | 10.255.0.1/30
  |
  | 10.255.0.2/30
+----------------+
|     NX-DC1     |
| Nexus / NX-OS  |
+----------------+
  ||
  || Eth1/3-4
  || LACP Port-channel10
  || VLAN 30,40
  ||
+----------------+
|     NX-DC2     |
| Nexus / NX-OS  |
+----------------+
  |             |
 VLAN30       VLAN40
  |             |
Server-C      Server-D
```

Logical Layer 3 topology:

```text
10.10.10.0/24      VLAN 10 USERS
        |
   10.10.10.1
        |
     CAT-DIST
        |
   10.10.20.1
        |
10.10.20.0/24      VLAN 20 SERVICES


CAT-DIST
10.255.0.1/30
        |
        |
10.255.0.2/30
NX-DC1


NX-DC1
   |
   +--- 10.20.30.1/24 → VLAN 30 SERVER
   |
   +--- 10.20.40.1/24 → VLAN 40 BACKUP/TEST
```

Static route relationship:

```text
CAT-DIST:

10.20.0.0/16
     ↓
10.255.0.2
     ↓
Nexus side
```

Nexus:

```text
10.10.0.0/16
     ↓
10.255.0.1
     ↓
Catalyst side
```

---

# PHẦN 4 — ADDRESSING PLAN

| Device/Object   | Interface/VLAN | IP               |
| --------------- | -------------- | ---------------- |
| PC-A            | VLAN 10        | `10.10.10.10/24` |
| PC-A Gateway    | VLAN 10 SVI    | `10.10.10.1`     |
| Optional PC-B   | VLAN 20        | `10.10.20.10/24` |
| PC-B Gateway    | VLAN 20 SVI    | `10.10.20.1`     |
| CAT-DIST        | routed transit | `10.255.0.1/30`  |
| NX-DC1          | routed transit | `10.255.0.2/30`  |
| Server-C        | VLAN 30        | `10.20.30.10/24` |
| VLAN 30 Gateway | SVI            | `10.20.30.1`     |
| Server-D        | VLAN 40        | `10.20.40.10/24` |
| VLAN 40 Gateway | SVI            | `10.20.40.1`     |

Networks:

```text
10.10.10.0/24
10.10.20.0/24

10.255.0.0/30

10.20.30.0/24
10.20.40.0/24
```

---

# PHẦN 5 — PREREQUISITES

## Infrastructure

Bạn cần một trong các loại environment:

- physical Catalyst + Nexus lab;
- Cisco Modeling Labs;
- EVE-NG với licensed images phù hợp;
- GNS3 với authorized images;
- instructor-provided virtual equivalent;
- prepared topology/fixture nếu enterprise platform không available.

## Access

Tối thiểu:

```text
read-only privileged access
```

cho diagnostic.

Configuration lab yêu cầu:

```text
authorized configuration privileges
```

Không dùng production credential.

## Documentation

Trước lab phải ghi:

```text
Device
Model
OS
Release
Interface naming
Feature availability
Emulator limitations
```

---

# PHẦN 6 — EMULATION LIMITATIONS

Nếu dùng virtual switch image, một số phần có thể không phản ánh physical hardware thực:

```text
Optic health
Actual CRC generation
Transceiver DOM
Physical speed/duplex negotiation
ASIC behavior
Hardware forwarding performance
Real link failures
Fan/power alarms
Some vPC behaviors
Some platform-specific features
```

Do đó evidence phải ghi:

```text
Environment:
Virtual / Emulated

Limitations:
Physical-layer counters are not representative
of a real Catalyst/Nexus chassis.
```

Syllabus yêu cầu phải **mark emulation limits explicitly** thay vì trình bày simulated result như actual-product physical experience. Infrastructure_Operations_14_Da…

---

# PHẦN 7 — SAFETY NOTES

Trước bất kỳ command thay đổi nào:

```text
1. Verify hostname.
2. Verify platform.
3. Verify environment.
4. Verify ticket/lab scope.
5. Capture baseline.
6. Capture configuration backup.
7. Know rollback command.
8. Know management path.
```

Một lỗi operational nguy hiểm:

```text
CAT-DIST#
```

nhưng bạn tưởng mình đang ở:

```text
CAT-LAB#
```

Do đó có thể đặt terminal prompt/hostname rõ ràng.

---

# PHẦN 8 — EVIDENCE DIRECTORY

Tạo cấu trúc evidence:

```text
Day08/
|
+-- 00_manifest/
|
+-- 01_topology/
|
+-- 02_catalyst_baseline/
|
+-- 03_nexus_baseline/
|
+-- 04_configuration/
|
+-- 05_verification/
|
+-- 06_failure_vlan/
|
+-- 07_failure_route/
|
+-- 08_failure_interface/
|
+-- 09_failure_portchannel/
|
+-- 10_backup/
|
+-- 11_rollback/
|
+-- 12_final/
```

Mỗi evidence nên có:

```text
Evidence ID
Device
Platform
Version
Environment
Timestamp + timezone
Command
Ticket/Lab ID
Expected result
Actual result
File/hash where applicable
```

---

# PHẦN 9 — PHASE 0: DEVICE IDENTITY

## Catalyst — READ ONLY

```text
show version
```

Capture:

```text
Hostname
Model
IOS/IOS XE version
Uptime
Boot image
Serial where authorized
```

Có thể thêm:

```text
show inventory
```

nếu supported.

---

## Nexus — READ ONLY

```text
show version
```

Capture:

```text
NX-OS version
Chassis/model
Uptime
Boot image
```

Không bắt đầu config cho đến khi xác nhận mình thực sự đang trên expected device.

---

# PHẦN 10 — PHASE 1: CATALYST BASELINE

Đây là **read-only diagnosis bắt buộc** theo additive assignment.

## Interface summary

```text
show interfaces status
```

Mục tiêu:

- port;
- status;
- VLAN;
- duplex;
- speed;
- type.

Expected healthy example:

```text
Port       Status       Vlan       Duplex Speed
Gi1/0/10   connected    10         a-full a-1000
Gi1/0/47   connected    trunk      a-full a-1000
Gi1/0/48   connected    trunk      a-full a-1000
```

---

# PHẦN 11 — DETAILED INTERFACE BASELINE

```text
show interfaces GigabitEthernet1/0/10
```

Quan sát:

```text
line protocol
MTU
bandwidth
duplex
speed
input rate
output rate
input errors
CRC
output errors
drops
resets
```

Không chỉ chụp trạng thái `up`.

Baseline đúng nên có counters.

Ví dụ:

```text
Evidence T1:
CRC = 0

Evidence T2:
CRC = 0
```

Nếu T1:

```text
CRC 100
```

T2 sau 60 giây:

```text
CRC 500
```

thì delta:

```text
+400/60s
```

quan trọng hơn số absolute.

---

# PHẦN 12 — CATALYST VLAN BASELINE

```text
show vlan brief
```

Check:

```text
VLAN exists?
Name?
Status?
Access ports?
```

Ví dụ:

```text
10   USERS      active
20   SERVICES   active
```

Nếu VLAN không tồn tại, port/trunk configuration phía sau có thể không hoạt động theo intended design.

---

# PHẦN 13 — CATALYST TRUNK BASELINE

```text
show interfaces trunk
```

Hoặc interface-specific trên supported releases:

```text
show interfaces GigabitEthernet1/0/47 trunk
```

Cisco Catalyst IOS XE documentation dùng `show interfaces ... switchport` và `show interfaces ... trunk` để verify trunk configuration. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-16/configuration_guide/vlan/b_1716_vlan_9300_cg/configuring_vlan_trunks.html?utm_source=chatgpt.com)

Quan sát:

```text
Mode
Encapsulation
Native VLAN
Allowed VLANs
Active VLANs
Forwarding VLANs
```

Phải phân biệt:

```text
allowed
```

với:

```text
actually forwarding
```

---

# PHẦN 14 — CATALYST MAC BASELINE

```text
show mac address-table
```

Có thể filter:

```text
show mac address-table vlan 10
```

Mục tiêu:

```text
Which MAC?
Which VLAN?
Which port?
Dynamic/static?
```

---

# PHẦN 15 — CATALYST ARP BASELINE

```text
show ip arp
```

Tìm:

```text
10.10.10.10
```

Nếu CAT-DIST là gateway, expected:

```text
10.10.10.10
→
PC-A MAC
```

---

# PHẦN 16 — CATALYST STP BASELINE

```text
show spanning-tree
```

Hoặc:

```text
show spanning-tree vlan 10
```

Ghi:

```text
Root ID
Bridge ID
Port role
Port state
Cost
Priority
```

Phải xác định:

> Device này có phải root bridge không?

---

# PHẦN 17 — CATALYST ETHERCHANNEL BASELINE

```text
show etherchannel summary
```

Cisco IOS XE examples sử dụng command này để verify channel-group và member state. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-8/configuration_guide/vlan/b_178_vlan_9300_cg.pdf?utm_source=chatgpt.com)

Điểm cần đọc:

```text
Po1(SU)
```

thường thể hiện logical port-channel Layer 2/in-use trong output tương ứng.

Member:

```text
Gi1/0/47(P)
Gi1/0/48(P)
```

`P` trong documented output biểu thị bundled member.

Không được chỉ xem:

```text
Po1 up
```

mà bỏ member state.

---

# PHẦN 18 — CATALYST L3 BASELINE

```text
show ip interface brief
```

Cisco IOS XE documentation mô tả output gồm interface, IP, administrative/operational status. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-16/command_reference/b_1716_9300_cr/interface_and_hardware_commands.html?utm_source=chatgpt.com)

Tiếp theo:

```text
show ip route
```

Check:

```text
Connected routes
Local routes
Static routes
Default route
```

---

# PHẦN 19 — CATALYST NEIGHBOR DISCOVERY

```text
show cdp neighbors
```

và nếu enabled:

```text
show lldp neighbors
```

Dùng để verify physical topology.

Expected:

```text
CAT-ACCESS Gi1/0/47
→ CAT-DIST Gi1/0/47
```

Nếu actual khác diagram, **actual evidence thắng giả định documentation**, nhưng phải ghi discrepancy.

---

# PHẦN 20 — PHASE 2: NEXUS READ-ONLY BASELINE

Syllabus yêu cầu diagnosis riêng cho Nexus, không copy Catalyst checklist rồi đổi hostname.

## Identity

```text
show version
```

---

## Interfaces

```text
show interface status
```

Detailed:

```text
show interface ethernet 1/1
```

Check:

```text
link status
errors
MTU
speed
duplex
input/output
```

---

# PHẦN 21 — NEXUS VLAN

```text
show vlan brief
```

Expected sau này:

```text
30 SERVER
40 BACKUP
```

---

# PHẦN 22 — NEXUS TRUNK

```text
show interface trunk
```

Check:

```text
Native VLAN
Allowed VLAN
STP forwarding state
```

Không assume output giống Catalyst.

---

# PHẦN 23 — NEXUS MAC

```text
show mac address-table
```

Cisco NX-OS command reference hiện hành vẫn sử dụng `show mac address-table` để inspect Layer 2 forwarding entries. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide/m_managing_the_unicast_rib_and_fib.html?utm_source=chatgpt.com)

---

# PHẦN 24 — NEXUS ARP

```text
show ip arp
```

Nếu sử dụng VRF-specific context, phải chạy query trong đúng VRF theo release syntax.

---

# PHẦN 25 — NEXUS ROUTING

```text
show ip route
```

NX-OS current documentation mô tả `show ip route` để hiển thị Unicast RIB và cho thấy route context/next-hop/interface. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide/m_managing_the_unicast_rib_and_fib.html?utm_source=chatgpt.com)

Bạn cần để ý dòng:

```text
IP Route Table for Context ...
```

Điều này nhắc rằng:

```text
routing table
```

phụ thuộc VRF/context.

---

# PHẦN 26 — NEXUS PORT-CHANNEL

```text
show port-channel summary
```

Current NX-OS documentation sử dụng command này để kiểm tra port-channel/LACP membership. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/interfaces/cisco-nexus-9000-series-nx-os-interfaces-configuration-guide-release-105x/m_configuring_port_channels_93x.html?bookSearch=true&utm_source=chatgpt.com)

---

# PHẦN 27 — NEXUS VRF

```text
show vrf
```

Mục tiêu:

```text
default
management
custom VRFs if any
```

Trước khi chạy management reachability test, phải biết traffic xuất phát từ routing context nào.

---

# PHẦN 28 — NEXUS vPC READ-ONLY

Nếu lab có vPC:

```text
show vpc brief
```

NX-OS command reference xác nhận đây là read-only vPC operational command. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/104x/command-reference/show/b_n9k_show_commands_104x/m_v_showcmds.html?utm_source=chatgpt.com)

Có thể inspect:

```text
show vpc consistency-parameters global
```

Không thay đổi vPC trong core Day 8 lab.

Mục tiêu chỉ là:

```text
vPC domain?
Peer status?
Peer-link?
Keepalive?
Consistency?
Member status?
```

---

# PHẦN 29 — BASELINE ACCEPTANCE GATE

Không bắt đầu configuration cho tới khi baseline có:

```text
Catalyst:
✓ version
✓ interface
✓ VLAN
✓ trunk
✓ MAC
✓ ARP
✓ STP
✓ EtherChannel
✓ routes
✓ config

Nexus:
✓ version
✓ interface
✓ VLAN
✓ trunk
✓ MAC
✓ ARP
✓ STP
✓ port-channel
✓ routes
✓ VRF
✓ config
```

Nếu environment đang có unexplained error:

```text
STOP
```

và ghi baseline exception.

---

# PHẦN 30 — PHASE 3: CONFIGURE CATALYST VLANs

## CHANGE — authorized lab only

Trên CAT-ACCESS và CAT-DIST:

```text
configure terminal

vlan 10
 name USERS

vlan 20
 name SERVICES

vlan 99
 name MANAGEMENT-LAB

end
```

### Giải thích

```text
vlan 10
```

tạo VLAN object.

```text
name USERS
```

gán human-readable name.

Tên VLAN không quyết định forwarding nhưng rất quan trọng cho Operations.

---

# PHẦN 31 — VERIFY VLAN

READ ONLY:

```text
show vlan brief
```

Expected:

```text
10   USERS
20   SERVICES
99   MANAGEMENT-LAB
```

Evidence:

```text
D08-CAT-VLAN-001
```

---

# PHẦN 32 — CONFIGURE CATALYST ACCESS PORT

Giả sử PC-A nằm `Gi1/0/10`.

## CHANGE

```text
configure terminal

interface GigabitEthernet1/0/10
 description LAB-PC-A_VLAN10
 switchport
 switchport mode access
 switchport access vlan 10
 no shutdown

end
```

### Giải thích

`description`:

```text
LAB-PC-A_VLAN10
```

tạo operational context.

`switchport`:

đảm bảo interface ở Layer 2 mode nếu platform yêu cầu.

`switchport mode access`:

ép interface thành access mode.

`switchport access vlan 10`:

assign VLAN 10.

`no shutdown`:

enable interface.

> Trong production, `no shutdown` chỉ được chạy nếu ticket xác nhận interface cần active.

---

# PHẦN 33 — VERIFY ACCESS PORT

READ ONLY:

```text
show interfaces GigabitEthernet1/0/10 switchport
```

```text
show interfaces status
```

```text
show vlan brief
```

Expected:

```text
Administrative Mode: static access
Access Mode VLAN: 10
Interface: connected
```

Exact labels thay đổi theo release.

---

# PHẦN 34 — ENDPOINT CONFIGURATION

Ví dụ PC-A:

```text
IP:      10.10.10.10
Mask:    255.255.255.0
Gateway: 10.10.10.1
```

Nếu dùng VPCS:

```text
ip 10.10.10.10/24 10.10.10.1
```

Nếu dùng Linux fixture, cấu hình theo network manager/method được lab cho phép.

Không nên modify production host networking chỉ để luyện switch.

---

# PHẦN 35 — PHASE 4: CATALYST LACP ETHERCHANNEL

Chúng ta bundle:

```text
Gi1/0/47
Gi1/0/48
```

thành:

```text
Port-channel1
```

giữa CAT-ACCESS và CAT-DIST.

Hai đầu phải được cấu hình consistent.

---

# PHẦN 36 — PRECHECK PORT-CHANNEL

READ ONLY:

```text
show interfaces status
```

```text
show etherchannel summary
```

```text
show running-config interface GigabitEthernet1/0/47
```

```text
show running-config interface GigabitEthernet1/0/48
```

Nếu interfaces đang production-used:

```text
STOP
```

---

# PHẦN 37 — CONFIGURE LACP MEMBERS

## CHANGE — cả hai Catalyst

Ví dụ:

```text
configure terminal

interface range GigabitEthernet1/0/47-48
 description LAB_LACP_TO_PEER
 channel-group 1 mode active

exit
```

`mode active` dùng LACP.

---

# PHẦN 38 — CONFIGURE PORT-CHANNEL TRUNK

```text
interface Port-channel1
 description LAB_TRUNK_TO_CATALYST_PEER
 switchport
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
 no shutdown

end
```

Cisco Catalyst documentation xác nhận `switchport mode trunk` đặt interface ở trunk mode và `switchport trunk allowed vlan` kiểm soát VLAN list trên trunk. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-16/configuration_guide/vlan/b_1716_vlan_9300_cg/configuring_vlan_trunks.html?utm_source=chatgpt.com)

---

# PHẦN 39 — VERIFY ETHERCHANNEL

READ ONLY:

```text
show etherchannel summary
```

Expected conceptual state:

```text
Po1
Protocol LACP
Gi1/0/47 bundled
Gi1/0/48 bundled
```

Tiếp:

```text
show interfaces Port-channel1
```

và:

```text
show interfaces trunk
```

Phải thấy VLAN:

```text
10
20
99
```

được allowed.

---

# PHẦN 40 — VERIFY STP TREATMENT

```text
show spanning-tree vlan 10
```

Bạn nên thấy:

```text
Port-channel1
```

được STP xem như logical interface thay vì independent members cho forwarding topology.

---

# PHẦN 41 — PHASE 5: CATALYST SVI

Trên CAT-DIST:

## CHANGE

```text
configure terminal

interface Vlan10
 description USERS_GATEWAY
 ip address 10.10.10.1 255.255.255.0
 no shutdown

interface Vlan20
 description SERVICES_GATEWAY
 ip address 10.10.20.1 255.255.255.0
 no shutdown

end
```

Catalyst documentation định nghĩa SVI là Layer 3 representation của VLAN dùng cho routing/host connectivity. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-16/configuration_guide/int_hw/b_1716_int_and_hw_9300_cg/configuring_interface_characteristics.html?utm_source=chatgpt.com)

---

# PHẦN 42 — ENABLE IP ROUTING

Trên reference Catalyst IOS XE L3 switch:

## CHANGE

```text
configure terminal
ip routing
end
```

Cisco IOS XE Catalyst documentation ghi nhận IP routing phải được enabled trước khi routed forwarding giữa Layer 3 interfaces/SVIs có thể diễn ra trên các documented releases. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-13/configuration_guide/rtng/b_1713_rtng_9300_cg/configuring_ip_unicast_routing.html?utm_source=chatgpt.com)

---

# PHẦN 43 — VERIFY SVI

READ ONLY:

```text
show ip interface brief
```

Expected:

```text
Vlan10   10.10.10.1   up   up
Vlan20   10.10.20.1   up   up
```

Nếu:

```text
Vlan10 up/down
```

hoặc:

```text
down/down
```

không ngay lập tức recreate SVI.

Kiểm tra:

```text
VLAN exists?
Any active port in VLAN?
Trunk carrying VLAN?
Interface shutdown?
STP?
```

---

# PHẦN 44 — VERIFY CONNECTED ROUTES

```text
show ip route
```

Expected có connected networks:

```text
10.10.10.0/24
10.10.20.0/24
```

Nếu SVI up nhưng route missing, phải investigate actual interface/IP/routing state.

---

# PHẦN 45 — FIRST INTER-VLAN TEST

PC-A:

```text
10.10.10.10
```

PC-B:

```text
10.10.20.10
```

Test:

```text
PC-A → ping 10.10.10.1
```

sau đó:

```text
PC-A → ping 10.10.20.1
```

rồi:

```text
PC-A → ping 10.10.20.10
```

Không nhảy thẳng test cuối.

---

# PHẦN 46 — CORRELATE ARP

CAT-DIST:

```text
show ip arp
```

Expected:

```text
10.10.10.10 → PC-A MAC
10.10.20.10 → PC-B MAC
```

---

# PHẦN 47 — CORRELATE MAC

CAT-DIST/CAT-ACCESS:

```text
show mac address-table vlan 10
```

Find PC-A MAC.

Expected logical chain:

```text
10.10.10.10
     ↓
show ip arp
     ↓
AAAA.BBBB.CCCC
     ↓
show mac address-table
     ↓
Port-channel1 / access interface
```

Đây là một core operational skill.

---

# PHẦN 48 — PHASE 6: ROUTED LINK CATALYST → NEXUS

Chúng ta tạo Layer 3 point-to-point transit:

```text
CAT-DIST
10.255.0.1/30

NX-DC1
10.255.0.2/30
```

---

# PHẦN 49 — CATALYST ROUTED PORT

Ví dụ interface:

```text
GigabitEthernet1/0/24
```

## CHANGE

```text
configure terminal

interface GigabitEthernet1/0/24
 description L3_TRANSIT_TO_NX-DC1
 no switchport
 ip address 10.255.0.1 255.255.255.252
 no shutdown

end
```

`no switchport` chuyển physical port thành Layer 3 routed port trên relevant Catalyst platforms. Cisco IOS XE documentation mô tả routed port theo cách này. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-13/configuration_guide/rtng/b_1713_rtng_9300_cg/configuring_ip_unicast_routing.html?utm_source=chatgpt.com)

---

# PHẦN 50 — VERIFY CATALYST TRANSIT

```text
show ip interface brief
```

Expected:

```text
Gi1/0/24  10.255.0.1  up up
```

Routing:

```text
show ip route 10.255.0.0
```

Expected connected:

```text
10.255.0.0/30
```

---

# PHẦN 51 — NEXUS ROUTED PORT

Giả sử NX-DC1:

```text
Ethernet1/1
```

## CHANGE

```text
configure terminal

interface Ethernet1/1
 description L3_TRANSIT_TO_CAT-DIST
 no switchport
 ip address 10.255.0.2/30
 no shutdown

end
```

---

# PHẦN 52 — VERIFY NEXUS TRANSIT

```text
show interface Ethernet1/1
```

```text
show ip route
```

Expected connected network:

```text
10.255.0.0/30
```

Test Nexus:

```text
ping 10.255.0.1
```

CAT-DIST:

```text
ping 10.255.0.2
```

Nếu fail, không cấu hình static route tiếp.

Transit phải healthy trước.

---

# PHẦN 53 — TROUBLESHOOT TRANSIT CHECKPOINT

Nếu ping transit fail:

```text
1. interface up?
2. no switchport both ends?
3. IP correct?
4. mask /30 correct?
5. cable/link correct?
6. same VRF?
7. connected route exists?
8. ARP/adjacency?
```

Không blame static route vì static route chưa liên quan đến directly connected transit.

---

# PHẦN 54 — PHASE 7: NEXUS L2 FEATURES

Trên NX-DC1/NX-DC2:

## CHANGE

Nếu platform/release yêu cầu:

```text
configure terminal
feature lacp
feature interface-vlan
end
```

NX-OS 10.5 documentation mô tả `feature lacp` là prerequisite để sử dụng LACP; documentation cho Nexus 9000 cũng yêu cầu `feature interface-vlan` cho SVI trên các platform có hỗ trợ, với một số platform exception phải kiểm tra riêng. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/interfaces/cisco-nexus-9000-series-nx-os-interfaces-configuration-guide-release-105x/m_configuring_port_channels_93x.html?utm_source=chatgpt.com)

---

# PHẦN 55 — CREATE NEXUS VLANs

```text
configure terminal

vlan 30
 name SERVER

vlan 40
 name BACKUP-TEST

end
```

Verify:

```text
show vlan brief
```

---

# PHẦN 56 — CREATE NX-OS PORT-CHANNEL

Giữa NX-DC1/NX-DC2:

```text
Ethernet1/3
Ethernet1/4
```

→

```text
port-channel10
```

## CHANGE

```text
configure terminal

interface Ethernet1/3
 switchport
 channel-group 10 mode active
 no shutdown

interface Ethernet1/4
 switchport
 channel-group 10 mode active
 no shutdown

interface port-channel10
 description LAB_TRUNK_TO_NX_PEER
 switchport
 switchport mode trunk
 switchport trunk allowed vlan 30,40
 no shutdown

end
```

NX-OS documentation uses `channel-group ... mode active` for LACP port-channel members and `show port-channel summary` for verification. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/interfaces/cisco-nexus-9000-series-nx-os-interfaces-configuration-guide-release-105x/m_configuring_port_channels_93x.html?bookSearch=true&utm_source=chatgpt.com)

---

# PHẦN 57 — VERIFY NEXUS PORT-CHANNEL

```text
show port-channel summary
```

Check:

```text
Po10 up?
LACP?
Eth1/3 bundled?
Eth1/4 bundled?
```

Sau đó:

```text
show interface trunk
```

Check:

```text
30
40
```

được carried.

---

# PHẦN 58 — NEXUS ACCESS PORTS

NX-DC2:

Server-C:

```text
Ethernet1/10
```

Server-D:

```text
Ethernet1/11
```

## CHANGE

```text
configure terminal

interface Ethernet1/10
 description SERVER-C_VLAN30
 switchport
 switchport mode access
 switchport access vlan 30
 no shutdown

interface Ethernet1/11
 description SERVER-D_VLAN40
 switchport
 switchport mode access
 switchport access vlan 40
 no shutdown

end
```

---

# PHẦN 59 — VERIFY NEXUS ACCESS PORT

```text
show interface status
```

```text
show vlan brief
```

```text
show running-config interface Ethernet1/10
```

Expected:

```text
Eth1/10 → VLAN30
Eth1/11 → VLAN40
```

---

# PHẦN 60 — PHASE 8: NEXUS SVI

Trên NX-DC1:

```text
configure terminal

interface vlan 30
 description SERVER_GATEWAY
 ip address 10.20.30.1/24
 no shutdown

interface vlan 40
 description BACKUP_TEST_GATEWAY
 ip address 10.20.40.1/24
 no shutdown

end
```

Nexus 9000 documentation mô tả VLAN interface/SVI là mechanism dùng cho Layer 3 inter-VLAN routing, với `feature interface-vlan` requirement trên relevant Nexus 9000 releases. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/interfaces/cisco-nexus-9000-series-nx-os-interfaces-configuration-guide-release-105x/m_configuring_layer_3_interfaces_9x.html?utm_source=chatgpt.com)

---

# PHẦN 61 — VERIFY NEXUS SVI

```text
show interface vlan 30
```

```text
show interface vlan 40
```

Expected:

```text
Vlan30 up
Vlan40 up
```

Routing:

```text
show ip route
```

Expected:

```text
10.20.30.0/24 connected
10.20.40.0/24 connected
```

---

# PHẦN 62 — ENDPOINTS NEXUS SIDE

Server-C:

```text
10.20.30.10/24
GW 10.20.30.1
```

Server-D:

```text
10.20.40.10/24
GW 10.20.40.1
```

---

# PHẦN 63 — TEST NEXUS INTER-VLAN

Server-C:

```text
ping 10.20.30.1
```

Sau đó:

```text
ping 10.20.40.1
```

Cuối cùng:

```text
ping 10.20.40.10
```

Nếu gateway ping fail:

troubleshoot L2 trước.

Nếu gateway works nhưng remote VLAN fails:

troubleshoot L3/routing.

---

# PHẦN 64 — NEXUS ARP CORRELATION

NX-DC1:

```text
show ip arp
```

Expected:

```text
10.20.30.10 → Server-C MAC
10.20.40.10 → Server-D MAC
```

NX-DC2:

```text
show mac address-table
```

Locate those MACs.

---

# PHẦN 65 — PHASE 9: STATIC ROUTE ON CATALYST

Catalyst phải biết:

```text
10.20.0.0/16
```

ở phía Nexus.

## CHANGE

```text
configure terminal

ip route 10.20.0.0 255.255.0.0 10.255.0.2

end
```

Ý nghĩa:

```text
Destination: 10.20.0.0/16
Next-hop:    10.255.0.2
```

---

# PHẦN 66 — VERIFY CATALYST STATIC ROUTE

```text
show ip route 10.20.30.10
```

Expected route phải resolve qua:

```text
10.255.0.2
```

Không chỉ chạy:

```text
show running-config | include ip route
```

vì configured route chưa chắc usable trong RIB nếu next-hop/context có vấn đề.

---

# PHẦN 67 — STATIC ROUTE ON NEXUS

Nexus phải biết Catalyst networks:

```text
10.10.0.0/16
```

## CHANGE

```text
configure terminal

ip route 10.10.0.0/16 10.255.0.1

end
```

NX-OS 10.5 static routing documentation sử dụng `ip route <prefix> <next-hop/interface>` và `show ip static-route`/`show ip route` để verify. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/unicast-routing-configuration/cisco-nexus-9000-series-nx-os-unicast-routing-configuration-guide/m_configuring_static_routing.html?utm_source=chatgpt.com)

---

# PHẦN 68 — VERIFY NEXUS STATIC ROUTE

```text
show ip static-route
```

và:

```text
show ip route 10.10.10.10
```

Expected next hop:

```text
10.255.0.1
```

---

# PHẦN 69 — WHY BOTH ROUTES ARE REQUIRED

Forward path:

```text
PC-A
10.10.10.10
   ↓
CAT-DIST
   ↓
10.255.0.2
   ↓
NX-DC1
   ↓
Server-C
10.20.30.10
```

Return path:

```text
Server-C
10.20.30.10
   ↓
NX-DC1
   ↓
10.255.0.1
   ↓
CAT-DIST
   ↓
PC-A
```

Nếu Nexus thiếu route về `10.10.0.0/16`:

request có thể đi tới Server-C nhưng reply không quay về.

---

# PHẦN 70 — PHASE 10: END-TO-END TEST

Không test một bước duy nhất.

Từ PC-A:

### Test 1 — local gateway

```text
ping 10.10.10.1
```

### Test 2 — Catalyst transit

```text
ping 10.255.0.1
```

### Test 3 — Nexus transit

```text
ping 10.255.0.2
```

### Test 4 — Nexus SVI

```text
ping 10.20.30.1
```

### Test 5 — Server

```text
ping 10.20.30.10
```

Mỗi hop giúp khoanh vùng failure domain.

---

# PHẦN 71 — PACKET WALK EXERCISE

Bạn phải tự giải thích traffic PC-A → Server-C.

PC-A:

```text
10.10.10.10/24
```

Destination:

```text
10.20.30.10
```

PC-A nhận ra destination remote.

Do đó ARP:

```text
10.10.10.1
```

Không ARP Server-C.

CAT-DIST nhận packet và routing table lookup:

```text
10.20.30.10
matches
10.20.0.0/16
via 10.255.0.2
```

CAT-DIST ARP/adjacency resolution cho:

```text
10.255.0.2
```

Nexus nhận packet.

Nexus route:

```text
10.20.30.0/24 connected
```

Nexus ARP:

```text
10.20.30.10
```

Sau đó frame đi qua VLAN30 trunk đến Server-C.

Đây là evidence-backed packet walk.

---

# PHẦN 72 — MAC/ARP CORRELATION EXERCISE

Bạn được cho:

```text
IP = 10.20.30.10
```

Mục tiêu tìm physical/logical location.

NX-DC1:

```text
show ip arp
```

Lấy MAC:

```text
0050.56aa.bbcc
```

Sau đó:

```text
show mac address-table address 0050.56aa.bbcc
```

Nếu output:

```text
VLAN30 → Po10
```

thì endpoint không local trên NX-DC1.

Chuyển NX-DC2:

```text
show mac address-table address 0050.56aa.bbcc
```

Expected:

```text
Eth1/10
```

Kết luận:

```text
10.20.30.10
→ MAC 0050.56aa.bbcc
→ NX-DC2 Eth1/10
```

Đây là loại reasoning rất thường dùng trong real Operations.

---

# PHẦN 73 — PHASE 11: STP INSPECTION

Catalyst:

```text
show spanning-tree vlan 10
```

Bạn phải ghi:

```text
Root bridge?
Root port?
Designated port?
Forwarding/blocking?
```

Nexus:

```text
show spanning-tree vlan 30
```

Ghi tương tự.

---

# PHẦN 74 — STP FAILURE INTERPRETATION

Giả sử:

```text
Port-channel1
Role: Alternate
State: Blocking/Discarding
```

Không lập tức sửa.

Hỏi:

```text
Is another redundant path active?
Is this expected by topology?
Who is root?
What does the design say?
```

STP block có thể là normal.

---

# PHẦN 75 — PHASE 12: LLDP/CDP TOPOLOGY VERIFICATION

Catalyst:

```text
show cdp neighbors detail
```

hoặc:

```text
show lldp neighbors detail
```

Nexus:

```text
show cdp neighbors detail
```

hoặc:

```text
show lldp neighbors detail
```

Tạo bảng:

| Local      | Local Port | Neighbor | Neighbor Port |
| ---------- | ---------- | -------- | ------------- |
| CAT-ACCESS | Gi1/0/47   | CAT-DIST | Gi1/0/47      |
| CAT-ACCESS | Gi1/0/48   | CAT-DIST | Gi1/0/48      |
| NX-DC1     | Eth1/3     | NX-DC2   | Eth1/3        |
| NX-DC1     | Eth1/4     | NX-DC2   | Eth1/4        |

Nếu mismatch:

```text
Document first.
Do not silently "correct" topology.
```

---

# PHẦN 76 — PHASE 13: INTERFACE HEALTH CHECK

Catalyst:

```text
show interfaces GigabitEthernet1/0/47
```

Nexus:

```text
show interface Ethernet1/3
```

Capture:

```text
Speed
Duplex
MTU
Input packets
Output packets
CRC
Input errors
Output errors
Drops
```

Chụp T1.

Generate traffic.

Chụp T2.

Calculate:

```text
delta = T2 - T1
```

---

# PHẦN 77 — HEALTH INTERPRETATION

## Healthy

```text
up
expected speed
expected duplex
no growing errors
```

## Degraded

```text
up
but increasing CRC
```

hoặc:

```text
port-channel missing a member
```

## Failed

```text
down
```

hoặc:

```text
administratively down unexpectedly
```

Đây là phân biệt quan trọng:

```text
Availability ≠ Health
```

---

# PHẦN 78 — PHASE 14: vPC AWARENESS EXERCISE

Không cần cấu hình full vPC trong core Day 8.

Nếu instructor cung cấp NX-DC1 + NX-DC2 đã configured vPC:

READ ONLY:

```text
show vpc brief
```

Cisco documents `show vpc brief` để inspect vPC state, peer-link và individual vPC status. [Cisco](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/104x/command-reference/show/b_n9k_show_commands_104x/m_v_showcmds.html?utm_source=chatgpt.com)

Ghi:

```text
vPC domain
Peer adjacency
Peer-link
Keepalive status
Consistency
vPC member
```

---

# PHẦN 79 — STANDALONE PORT-CHANNEL VS vPC

Bạn phải tự vẽ được:

Standalone:

```text
Server
 ||||
 ||||
NX-DC1
```

vPC:

```text
        Server
        |    |
        |    |
     NX-DC1 NX-DC2
        \    /
       vPC peers
```

Standalone:

```text
all physical members terminate on one chassis
```

vPC:

```text
member links terminate on two vPC peer switches
```

Không nói:

> "vPC chỉ là EtherChannel lớn hơn."

Đó là oversimplification.

---

# PHẦN 80 — FAILURE INJECTION METHODOLOGY

Mỗi injected fault phải theo workflow:

```text
Healthy baseline
↓
Record expected result
↓
Inject ONE fault
↓
Observe symptom
↓
Do not reveal cause initially
↓
Read-only diagnosis
↓
Identify root cause
↓
Authorized correction
↓
Verify
↓
Collect evidence
↓
Return to baseline
```

Không inject nhiều lỗi cùng lúc khi đang học foundation.

---

# PHẦN 81 — FAILURE INJECTION 1: WRONG ACCESS VLAN

Baseline:

```text
PC-A → VLAN10
```

Healthy:

```text
PC-A → 10.10.10.1 PASS
```

## Instructor CHANGE

Trên CAT-ACCESS:

```text
configure terminal

interface GigabitEthernet1/0/10
 switchport access vlan 20

end
```

Đây là intentional lab fault.

---

# PHẦN 82 — SYMPTOM

PC-A:

```text
ping 10.10.10.1
```

fails.

Physical interface:

```text
up/up
```

Điều này chứng minh:

```text
Link up ≠ correct connectivity
```

---

# PHẦN 83 — DIAGNOSIS ORDER

READ ONLY:

```text
show interfaces status
```

Interface up.

Tiếp:

```text
show interfaces GigabitEthernet1/0/10 switchport
```

Observe:

```text
Access VLAN: 20
```

Expected:

```text
10
```

Root cause:

```text
access VLAN mismatch
```

---

# PHẦN 84 — CORRECTION

Approved lab change:

```text
configure terminal

interface GigabitEthernet1/0/10
 switchport access vlan 10

end
```

---

# PHẦN 85 — VERIFICATION

```text
show interfaces GigabitEthernet1/0/10 switchport
```

Expected VLAN10.

Then:

```text
ping 10.10.10.1
```

PASS.

Collect:

```text
before
fault
root-cause evidence
after
ping result
```

---

# PHẦN 86 — FAILURE INJECTION 2: VLAN REMOVED FROM TRUNK

Healthy trunk:

```text
10,20,99
```

Instructor changes:

```text
configure terminal

interface Port-channel1
 switchport trunk allowed vlan 20,99

end
```

VLAN10 removed.

---

# PHẦN 87 — EXPECTED SYMPTOM

PC-A access port:

```text
up
```

VLAN:

```text
10
```

Correct.

But gateway across trunk:

```text
unreachable
```

This is important:

```text
Access configuration correct
but path still fails
```

---

# PHẦN 88 — DIAGNOSIS

```text
show interfaces trunk
```

Observe:

```text
Allowed VLAN:
20,99
```

Expected:

```text
10,20,99
```

Potential additional evidence:

CAT-ACCESS:

```text
show mac address-table vlan 10
```

may know PC MAC.

CAT-DIST may not learn that MAC via Po1 due VLAN10 path failure.

---

# PHẦN 89 — CORRECT TRUNK

```text
configure terminal

interface Port-channel1
 switchport trunk allowed vlan 10,20,99

end
```

---

# PHẦN 90 — VERIFY TRUNK RECOVERY

```text
show interfaces trunk
```

Then:

```text
show mac address-table vlan 10
```

Then:

```text
ping 10.10.10.1
```

PASS.

---

# PHẦN 91 — FAILURE INJECTION 3: WRONG STATIC ROUTE

Healthy Catalyst route:

```text
10.20.0.0/16
via 10.255.0.2
```

Inject:

```text
configure terminal

no ip route 10.20.0.0 255.255.0.0 10.255.0.2
ip route 10.20.0.0 255.255.0.0 10.255.0.3

end
```

Use only if `10.255.0.3` is safe/nonexistent fixture.

---

# PHẦN 92 — EXPECTED SYMPTOM

PC-A can reach:

```text
10.10.10.1
```

and perhaps:

```text
10.255.0.1
```

but not:

```text
10.20.30.10
```

This indicates:

```text
Local L2/L3 healthy
Remote routing suspect
```

---

# PHẦN 93 — DIAGNOSIS

First:

```text
show ip route 10.20.30.10
```

Observe:

```text
next-hop 10.255.0.3
```

Expected:

```text
10.255.0.2
```

Then test:

```text
ping 10.255.0.2
```

PASS.

Potential:

```text
ping 10.255.0.3
```

FAIL.

Root cause:

```text
incorrect static-route next hop
```

---

# PHẦN 94 — ROUTE CORRECTION

```text
configure terminal

no ip route 10.20.0.0 255.255.0.0 10.255.0.3
ip route 10.20.0.0 255.255.0.0 10.255.0.2

end
```

---

# PHẦN 95 — VERIFY ROUTE

```text
show ip route 10.20.30.10
```

Then:

```text
ping 10.20.30.10
```

PASS.

Also test:

```text
10.20.30.10 → 10.10.10.10
```

to verify return path.

---

# PHẦN 96 — FAILURE INJECTION 4: INTERFACE ADMINISTRATIVELY DOWN

Choose lab-only endpoint port.

Instructor:

```text
configure terminal

interface GigabitEthernet1/0/10
 shutdown

end
```

---

# PHẦN 97 — SYMPTOM

```text
show interfaces status
```

may indicate disabled/notconnect depending output.

Detailed:

```text
show interfaces GigabitEthernet1/0/10
```

shows:

```text
administratively down
```

---

# PHẦN 98 — CRITICAL OPERATIONAL LESSON

Trong lab:

```text
no shutdown
```

là expected correction.

Trong production:

**không được làm như vậy chỉ vì bạn thấy administratively down**.

Bạn cần biết:

```text
Why is it shutdown?
Ticket?
Security action?
Maintenance?
Unused port?
Known fault?
```

---

# PHẦN 99 — LAB CORRECTION

Sau khi instructor confirms:

```text
configure terminal

interface GigabitEthernet1/0/10
 no shutdown

end
```

Verify:

```text
show interfaces status
```

và connectivity.

---

# PHẦN 100 — FAILURE INJECTION 5: PORT-CHANNEL MEMBER FAILURE

Healthy:

```text
Po10
Eth1/3
Eth1/4
```

both bundled.

Inject on one Nexus member:

```text
configure terminal

interface Ethernet1/4
 shutdown

end
```

---

# PHẦN 101 — EXPECTED RESULT

Potentially service remains reachable because:

```text
Eth1/3 remains active
```

But:

```text
redundancy reduced
capacity reduced
```

Therefore:

```text
Ping PASS
```

does not mean:

```text
System Healthy
```

---

# PHẦN 102 — DIAGNOSIS

NX-DC1:

```text
show port-channel summary
```

Expected:

```text
Eth1/3 bundled
Eth1/4 down/not bundled
```

Then:

```text
show interface Ethernet1/4
```

identify administrative state.

---

# PHẦN 103 — RESTORE

Lab authorized:

```text
configure terminal

interface Ethernet1/4
 no shutdown

end
```

Verify:

```text
show port-channel summary
```

Both members must return.

---

# PHẦN 104 — OPTIONAL FAILURE: LACP MISMATCH

Một more realistic fault:

one side:

```text
channel-group 10 mode active
```

other side misconfigured or no corresponding channel-group.

Symptoms:

```text
member not bundled
port suspended / individual
```

depending platform/release.

Diagnosis:

```text
show port-channel summary
```

```text
show running-config interface Ethernet1/3
```

Compare both ends.

Do not randomly remove/recreate whole port-channel.

---

# PHẦN 105 — OPTIONAL FAILURE: NATIVE VLAN MISMATCH

Lab only.

One trunk:

```text
native VLAN 99
```

other:

```text
native VLAN 1
```

Potential symptoms depend traffic and STP behavior.

Diagnosis must compare both ends.

Never "fix" by disabling STP.

---

# PHẦN 106 — TROUBLESHOOTING FLOW: LAYER 1

Always begin:

```text
Interface state
↓
Admin state
↓
Speed/duplex
↓
Errors
↓
Physical neighbor
```

Questions:

```text
Is link physically present?
Is local side enabled?
Is remote side enabled?
Are errors growing?
```

---

# PHẦN 107 — TROUBLESHOOTING FLOW: LAYER 2

Then:

```text
Correct VLAN?
↓
Access/trunk?
↓
Allowed VLAN?
↓
STP forwarding?
↓
Port-channel healthy?
↓
MAC learning?
```

Do not inspect routes until Layer 2 path makes sense.

---

# PHẦN 108 — TROUBLESHOOTING FLOW: LAYER 3

Then:

```text
IP
↓
Mask
↓
Gateway
↓
ARP
↓
SVI
↓
Connected route
↓
Static route
↓
Next hop
↓
VRF
↓
Return route
```

Then policy:

```text
ACL
```

---

# PHẦN 109 — CATALYST READ-ONLY TROUBLESHOOTING RUNBOOK

Suggested order:

```text
show version
show interfaces status
show interfaces <interface>
show interfaces <interface> switchport
show vlan brief
show interfaces trunk
show etherchannel summary
show spanning-tree
show mac address-table
show ip arp
show ip interface brief
show ip route
show cdp neighbors
show lldp neighbors
show running-config
```

Điểm quan trọng:

Bạn không nhất thiết chạy tất cả mọi lần.

Runbook giúp bạn chọn đúng category.

---

# PHẦN 110 — NEXUS READ-ONLY TROUBLESHOOTING RUNBOOK

```text
show version
show interface status
show interface <interface>
show vlan brief
show interface trunk
show port-channel summary
show spanning-tree
show mac address-table
show ip arp
show interface vlan <id>
show ip route
show vrf
show cdp neighbors
show lldp neighbors
show vpc brief
show running-config
```

Một lần nữa:

```text
IOS XE command familiarity
≠
permission to assume NX-OS behavior
```

---

# PHẦN 111 — MANAGEMENT VRF PRACTICE

Trên Nexus:

```text
show vrf
```

Xác định:

```text
management
default
```

Nếu management interface:

```text
mgmt0
```

thuộc management VRF, reachability query phải xem đúng context.

Một failure scenario:

```text
Default VRF cannot reach management server
```

không tự động nghĩa:

```text
Management network broken
```

Có thể bạn đang test sai VRF/source.

---

# PHẦN 112 — CONFIGURATION BACKUP: PRE-CHANGE

Đây là syllabus requirement bắt buộc.

Trước major lab change:

Catalyst:

```text
show running-config
```

```text
show startup-config
```

Nexus:

```text
show running-config
```

```text
show startup-config
```

Lưu thành:

```text
CAT-DIST_running_before.txt
CAT-DIST_startup_before.txt

NX-DC1_running_before.txt
NX-DC1_startup_before.txt
```

---

# PHẦN 113 — RUNNING VS STARTUP

Hai trạng thái khác nhau:

```text
running-config
=
current active state
```

```text
startup-config
=
configuration intended to load during next normal startup
```

NX-OS official documentation xác nhận hai files có thể khác nhau và `copy running-config startup-config` được dùng để persist current state. [spg.xgslb-v3.cisco.com](https://spg.xgslb-v3.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/fundamentals/cisco-nexus-9000-series-nx-os-fundamentals-configuration-guide-release-105x/m-working-with-configuration-files.html?utm_source=chatgpt.com)

Catalyst documentation cũng dùng:

```text
copy running-config startup-config
```

để persist intended configuration. [Cisco](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/17-13/configuration_guide/vlan/b_1713_vlan_9300_cg.pdf?utm_source=chatgpt.com)

---

# PHẦN 114 — DO NOT AUTO-SAVE

Sai operational behavior:

```text
Made change
↓
Immediately save
```

Correct thinking:

```text
Made approved change
↓
Verify
↓
Confirm intended final state
↓
Persist only if change plan requires
```

Temporary troubleshooting state không nên vô tình trở thành reboot-persistent state.

---

# PHẦN 115 — SAVE CONFIGURATION

Khi approved:

Catalyst:

```text
copy running-config startup-config
```

Nexus:

```text
copy running-config startup-config
```

Đây là **configuration-changing/persistence action**.

Không phải read-only command.

---

# PHẦN 116 — EXTERNAL BACKUP

NX-OS official documentation cũng hỗ trợ copy configuration file tới remote server theo supported schemes, dùng cho backup. [spg.xgslb-v3.cisco.com](https://spg.xgslb-v3.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/fundamentals/cisco-nexus-9000-series-nx-os-fundamentals-configuration-guide-release-105x/m-working-with-configuration-files.html?utm_source=chatgpt.com)

Trong enterprise, backup destination có thể là:

```text
approved configuration management server
SCP/SFTP repository
network automation platform
version-controlled config archive
```

Không tự dựng insecure TFTP service trong production chỉ để lấy backup nếu policy không cho phép.

---

# PHẦN 117 — CONFIGURATION DIFF

Trước:

```text
running_before.txt
```

Sau:

```text
running_after.txt
```

Compare.

Bạn phải trả lời:

```text
Which lines changed?
Were they approved?
Any unexpected changes?
```

Ví dụ expected:

```text
+ vlan 10
+ vlan 20
+ interface Vlan10
+ ip route ...
```

Nếu thấy unrelated:

```text
- aaa ...
+ username ...
```

STOP và investigate.

---

# PHẦN 118 — WHY CONFIG DIFF MATTERS

Nếu connectivity restored nhưng unintended config changed:

```text
Change ≠ successful
```

Success yêu cầu:

```text
Expected service result
+
Expected configuration state
+
No unintended side effect
```

---

# PHẦN 119 — ROLLBACK DESIGN

Không nên định nghĩa rollback là:

```text
Restore entire config blindly
```

Trong production điều đó có thể overwrite legitimate concurrent changes.

Preferred rollback:

```text
undo exact authorized change
```

Ví dụ VLAN fault:

Before:

```text
switchport access vlan 10
```

Change:

```text
switchport access vlan 20
```

Rollback:

```text
switchport access vlan 10
```

---

# PHẦN 120 — STATIC ROUTE ROLLBACK

Change:

```text
ip route 10.20.0.0/16 10.255.0.3
```

Rollback NX-OS example:

```text
no ip route 10.20.0.0/16 10.255.0.3
ip route 10.20.0.0/16 10.255.0.2
```

On Catalyst corresponding mask-style syntax:

```text
no ip route 10.20.0.0 255.255.0.0 10.255.0.3
ip route 10.20.0.0 255.255.0.0 10.255.0.2
```

Always verify exact platform syntax first.

---

# PHẦN 121 — TRUNK ROLLBACK

Suppose change:

```text
switchport trunk allowed vlan 10,20,30,99
```

but approved baseline:

```text
10,20,99
```

Rollback must restore:

```text
10,20,99
```

Không dùng:

```text
switchport trunk allowed vlan all
```

vì đó không phải rollback; đó là expanding scope.

---

# PHẦN 122 — ROLLBACK VERIFICATION

Sau rollback:

```text
show running-config interface ...
```

then:

```text
show interfaces trunk
```

then:

```text
show mac address-table
```

then:

```text
show ip route
```

then connectivity.

Rollback không complete chỉ vì command accepted.

---

# PHẦN 123 — CONFIG PERSISTENCE VERIFICATION

Sau approved final state:

```text
show running-config
```

và:

```text
show startup-config
```

Compare relevant sections.

Goal:

```text
Intended active state
=
Intended persisted state
```

nếu change policy yêu cầu persistence.

---

# PHẦN 124 — CHANGE CONTROL RECORD

Một Day 8 lab change record nên có:

```text
Change ID:
CHG-D08-001

Purpose:
Create test VLAN10/VLAN20 and inter-VLAN routing.

Environment:
LAB

Devices:
CAT-ACCESS
CAT-DIST
NX-DC1
NX-DC2

Risk:
Temporary loss of test connectivity.

Pre-check:
Interfaces healthy.
No errors.
Configuration backed up.

Implementation:
...

Verification:
...

Stop conditions:
Unexpected management loss
Unexpected STP change
Unexpected non-lab interface affected

Rollback:
...

Evidence IDs:
...
```

---

# PHẦN 125 — STOP CONDITIONS

Đây là kỹ năng thường bị bỏ quên.

Stop nếu:

```text
Management session unexpectedly lost.
Unexpected production VLAN appears affected.
STP root unexpectedly changes outside scope.
Multiple unrelated interfaces flap.
Port-channel peer becomes inconsistent.
Routing table changes beyond intended prefixes.
Device CPU/health deteriorates.
You cannot identify the device/version.
Rollback command is uncertain.
```

Trong lab có thể tiếp tục theo instructor.

Trong production:

```text
STOP → assess → escalate
```

---

# PHẦN 126 — EVIDENCE PACKAGE: CATALYST

Tối thiểu:

```text
D08-CAT-001_show-version.txt
D08-CAT-002_interfaces-status.txt
D08-CAT-003_vlan.txt
D08-CAT-004_trunk.txt
D08-CAT-005_mac.txt
D08-CAT-006_arp.txt
D08-CAT-007_stp.txt
D08-CAT-008_etherchannel.txt
D08-CAT-009_routes.txt
D08-CAT-010_config-before.txt
D08-CAT-011_config-after.txt
D08-CAT-012_config-rollback.txt
```

---

# PHẦN 127 — EVIDENCE PACKAGE: NEXUS

```text
D08-NX-001_show-version.txt
D08-NX-002_interface-status.txt
D08-NX-003_vlan.txt
D08-NX-004_trunk.txt
D08-NX-005_mac.txt
D08-NX-006_arp.txt
D08-NX-007_stp.txt
D08-NX-008_portchannel.txt
D08-NX-009_routes.txt
D08-NX-010_vrf.txt
D08-NX-011_vpc.txt
D08-NX-012_config-before.txt
D08-NX-013_config-after.txt
D08-NX-014_rollback.txt
```

Syllabus yêu cầu chính xác tinh thần này: Catalyst và Nexus có platform-specific evidence set, tested path, root-cause reasoning, config backup và rollback verification. Infrastructure_Operations_14_Da…

---

# PHẦN 128 — EVIDENCE PACKAGE CHO MỖI FAILURE

Ví dụ VLAN mismatch:

```text
Incident:
FAIL-D08-001

Symptom:
PC-A cannot reach gateway.

Expected:
Gi1/0/10 VLAN10.

Actual:
Gi1/0/10 VLAN20.

Root cause:
Incorrect access VLAN.

Evidence:
show interface ... switchport

Correction:
Restored VLAN10.

Verification:
Gateway ping PASS.
MAC learned in VLAN10.

Rollback:
Not required beyond returning baseline.
```

---

# PHẦN 129 — ROOT-CAUSE REASONING TEMPLATE

Không viết:

```text
Ping failed because network issue.
```

Viết:

```text
Symptom:
PC-A could not reach 10.10.10.1.

Lower-layer evidence:
Gi1/0/10 was physically up with no increasing errors.

L2 evidence:
Gi1/0/10 was assigned to VLAN20; design expected VLAN10.

Root cause:
Access VLAN mismatch.

Correction:
Restored access VLAN to VLAN10 under lab change.

Verification:
Interface remained up, PC-A MAC was learned in VLAN10,
ARP resolution succeeded and gateway ping passed.
```

Đây mới là Operations-grade reasoning.

---

# PHẦN 130 — TEST MATRIX

Bạn nên có bảng:

| Test                        | Expected                                       |
| --------------------------- | ---------------------------------------------- |
| PC-A → CAT VLAN10 gateway   | PASS                                           |
| PC-B → VLAN20 gateway       | PASS                                           |
| PC-A → PC-B                 | PASS                                           |
| CAT-DIST → NX transit       | PASS                                           |
| NX → CAT transit            | PASS                                           |
| Server-C → VLAN30 gateway   | PASS                                           |
| Server-C → Server-D         | PASS                                           |
| PC-A → Server-C             | PASS                                           |
| Server-C → PC-A             | PASS                                           |
| VLAN mismatch injected      | FAIL as expected                               |
| VLAN restored               | PASS                                           |
| Wrong route injected        | FAIL as expected                               |
| Route restored              | PASS                                           |
| Interface shutdown          | FAIL as expected                               |
| Interface restored          | PASS                                           |
| Single LACP member shutdown | Connectivity survives, degraded state detected |

Một lab tốt test cả:

```text
positive behavior
```

và:

```text
negative behavior
```

---

# PHẦN 131 — PERMITTED PATH VALIDATION

Syllabus nói validate permitted network paths.

Trong Day 8 context, bạn có thể define:

```text
PC-A VLAN10 → Server-C VLAN30 = expected reachable
```

và:

```text
PC-A VLAN10 → unsupported/unconfigured test subnet = expected unreachable
```

Không cần firewall policy Day 9.

Mục tiêu là xác minh routing/topology behaves as designed.

---

# PHẦN 132 — TESTING A NEGATIVE PATH

Ví dụ:

```text
10.99.99.99
```

không có route.

Check:

```text
show ip route 10.99.99.99
```

Nếu no default route:

expected:

```text
No matching route → unreachable
```

Negative test này giúp chứng minh routing không accidentally rộng hơn intended.

---

# PHẦN 133 — MANAGEMENT PLANE TEST

Nexus:

```text
show vrf
```

Xác định management context.

Nếu instructor cung cấp management target:

```text
Mgmt server = 192.0.2.10
```

kiểm tra route trong proper VRF.

Mục tiêu:

```text
Do not confuse management reachability
with default VRF/data-plane reachability.
```

---

# PHẦN 134 — ACL FUNDAMENTALS PRACTICE

Day 8 chỉ cần fundamental.

Nếu instructor cung cấp existing lab ACL, ưu tiên **inspect-only**:

Catalyst examples có thể gồm:

```text
show access-lists
```

hoặc:

```text
show ip access-lists
```

Mục tiêu:

```text
Which ACL?
Which rule?
Permit/deny?
Direction?
Applied interface?
```

Không cần xây complex ACL lab nếu chưa thuộc assignment.

---

# PHẦN 135 — WHY NOT `permit ip any any`

Nếu connectivity fails và bạn sửa bằng:

```text
permit ip any any
```

thì bạn chưa troubleshoot.

Bạn chỉ đã bypass policy.

Day 8 phải giữ principle:

```text
Routing problem
≠
ACL problem
```

và least privilege.

---

# PHẦN 136 — READ-ONLY VS CHANGE COMMAND TABLE

| Type               | Examples                             |
| ------------------ | ------------------------------------ |
| Read-only          | `show version`                       |
| Read-only          | `show interfaces status`             |
| Read-only          | `show vlan brief`                    |
| Read-only          | `show interfaces trunk`              |
| Read-only          | `show mac address-table`             |
| Read-only          | `show ip arp`                        |
| Read-only          | `show spanning-tree`                 |
| Read-only          | `show ip route`                      |
| Read-only          | `show etherchannel summary`          |
| Read-only          | `show port-channel summary`          |
| Read-only          | `show vpc brief`                     |
| Change             | `configure terminal`                 |
| Change             | `vlan 10`                            |
| Change             | `switchport access vlan 10`          |
| Change             | `switchport mode trunk`              |
| Change             | `channel-group ...`                  |
| Change             | `ip address ...`                     |
| Change             | `ip route ...`                       |
| Change             | `shutdown/no shutdown`               |
| Persistence change | `copy running-config startup-config` |

Một `show running-config` thường read-only nhưng output có thể chứa sensitive material.

---

# PHẦN 137 — DON'T CLEAR STATE TOO EARLY

Các command kiểu:

```text
clear arp
clear mac address-table
clear counters
```

có thể làm mất useful evidence hoặc alter operational state.

Vì vậy troubleshooting sequence:

```text
Observe first.
Capture first.
Clear only if approved and necessary.
```

Không cần dùng các command này để hoàn thành Day 8 core lab.

---

# PHẦN 138 — DON'T BOUNCE INTERFACE AS FIRST STEP

Một common bad habit:

```text
Port problem?
shutdown
no shutdown
```

Điều này có thể:

- disrupt service;
- erase transient symptoms;
- trigger reconvergence;
- hide root cause;
- violate change process.

Đúng hơn:

```text
inspect status
inspect errors
inspect config
inspect remote side
inspect logs
```

rồi mới decide.

---

# PHẦN 139 — PHYSICAL ERROR SCENARIO

Nếu physical hardware lab cho phép:

T1:

```text
CRC: 100
```

T2:

```text
CRC: 5000
```

Check both ends.

Possible investigation:

```text
Cable
Optic
Patch panel
Remote transceiver
Speed
FEC if applicable
Hardware
```

Không replace production optic without authorization/change.

---

# PHẦN 140 — ESCALATION BOUNDARY

L1/L2 learner có thể:

```text
inspect
collect
correlate
test approved connectivity
identify likely root cause
execute pre-approved lab correction
```

Escalate nếu:

```text
Possible hardware failure
STP instability
Possible L2 loop
Core/distribution-wide impact
Multiple VLAN outage
vPC peer-link problem
NX-OS defect suspected
IOS XE defect suspected
Firmware upgrade required
Management plane lost
No safe rollback
Unknown configuration ownership
Security event suspected
```

---

# PHẦN 141 — CATALYST VS NEXUS COMMAND COMPARISON

| Goal               | Catalyst IOS/IOS XE            | Nexus NX-OS                  |
| ------------------ | ------------------------------ | ---------------------------- |
| Version            | `show version`                 | `show version`               |
| Interface summary  | `show interfaces status`       | `show interface status`      |
| Detailed interface | `show interfaces Gi...`        | `show interface Ethernet...` |
| VLAN               | `show vlan brief`              | `show vlan brief`            |
| Trunk              | `show interfaces trunk`        | `show interface trunk`       |
| MAC                | `show mac address-table`       | `show mac address-table`     |
| ARP                | `show ip arp`                  | `show ip arp`                |
| STP                | `show spanning-tree`           | `show spanning-tree`         |
| Aggregation        | `show etherchannel summary`    | `show port-channel summary`  |
| Routing            | `show ip route`                | `show ip route`              |
| VRF                | platform-dependent command set | `show vrf`                   |
| vPC                | N/A as Nexus vPC               | `show vpc brief`             |
| Running            | `show running-config`          | `show running-config`        |
| Startup            | `show startup-config`          | `show startup-config`        |

Tên interface và exact arguments phụ thuộc platform/release.

---

# PHẦN 142 — PRACTICAL TROUBLESHOOTING EXAMPLE A

Problem:

```text
PC-A cannot ping Server-C.
```

Bad approach:

```text
Change route.
```

Correct approach:

```text
PC-A → gateway?
```

PASS.

```text
CAT-DIST → NX transit?
```

PASS.

Check:

```text
show ip route 10.20.30.10
```

Correct.

NX:

```text
show ip route 10.10.10.10
```

No matching route.

Root cause:

```text
missing return route
```

Fix:

add approved route.

Evidence:

```text
before route table
change
after route table
bidirectional ping
```

---

# PHẦN 143 — PRACTICAL TROUBLESHOOTING EXAMPLE B

Problem:

```text
Server-C gateway unreachable.
```

Check NX-DC1:

```text
show interface vlan 30
```

up.

Check NX-DC2:

```text
show vlan brief
```

VLAN30 exists.

Check:

```text
show interface trunk
```

VLAN30 missing from Po10.

Root cause:

```text
trunk allowed VLAN omission
```

Không phải routing.

---

# PHẦN 144 — PRACTICAL TROUBLESHOOTING EXAMPLE C

Problem:

```text
Network is slow but ping succeeds.
```

Check:

```text
show interface ...
```

Find CRC increasing.

This indicates possible Layer 1 issue.

Do not change routes.

This illustrates:

```text
symptom performance
→ Layer 1 evidence
```

---

# PHẦN 145 — PRACTICAL TROUBLESHOOTING EXAMPLE D

Problem:

```text
Server reachable but redundancy alarm.
```

Check:

```text
show port-channel summary
```

Only one of two members bundled.

Root cause may be:

```text
member down
LACP mismatch
member config mismatch
physical issue
```

Service availability does not remove urgency of degraded redundancy.

---

# PHẦN 146 — PRACTICAL TROUBLESHOOTING EXAMPLE E

Problem:

```text
Nexus can reach production test network
but cannot reach management collector.
```

Do not immediately add default route.

First:

```text
show vrf
```

Determine:

```text
management vs default context
```

Then inspect routes in correct context.

---

# PHẦN 147 — FINAL HEALTH CHECK: CATALYST

Before declaring success:

```text
show version
show interfaces status
show interfaces trunk
show vlan brief
show etherchannel summary
show spanning-tree
show mac address-table
show ip arp
show ip interface brief
show ip route
```

Compare with pre-change.

---

# PHẦN 148 — FINAL HEALTH CHECK: NEXUS

```text
show version
show interface status
show interface trunk
show vlan brief
show port-channel summary
show spanning-tree
show mac address-table
show ip arp
show ip route
show vrf
show vpc brief
```

`show vpc brief` chỉ nếu vPC applicable.

---

# PHẦN 149 — FINAL END-TO-END PATH

Must pass:

```text
PC-A
→ CAT-ACCESS
→ Port-channel1
→ CAT-DIST
→ 10.255.0.2
→ NX-DC1
→ Port-channel10
→ NX-DC2
→ Server-C
```

and return:

```text
Server-C
→ NX-DC2
→ NX-DC1
→ CAT-DIST
→ CAT-ACCESS
→ PC-A
```

---

# PHẦN 150 — EXPECTED ROUTING TABLE LOGIC

CAT-DIST should conceptually know:

```text
10.10.10.0/24 → connected
10.10.20.0/24 → connected
10.255.0.0/30 → connected
10.20.0.0/16  → 10.255.0.2
```

NX-DC1:

```text
10.20.30.0/24 → connected
10.20.40.0/24 → connected
10.255.0.0/30 → connected
10.10.0.0/16  → 10.255.0.1
```

Bạn phải có thể viết bảng này **trước khi nhìn CLI**.

Đó là dấu hiệu bạn hiểu routing thay vì phụ thuộc command output.

---

# PHẦN 151 — EXPECTED L2 LOGIC

Catalyst:

```text
PC-A MAC
↓
Gi1/0/10 on CAT-ACCESS
↓
Po1 toward CAT-DIST
```

Nexus:

```text
Server-C MAC
↓
Eth1/10 on NX-DC2
↓
Po10 toward NX-DC1
```

---

# PHẦN 152 — CLEANUP OPTION A: RETAIN FINAL LAB

Nếu environment dành cho Day 9/Day 14 reuse:

```text
Do not tear down.
```

Thay vào đó:

1. verify healthy state;
2. save approved configuration;
3. archive evidence;
4. document topology;
5. mark asset as reusable lab baseline.

---

# PHẦN 153 — CLEANUP OPTION B: FULL ROLLBACK

Nếu phải trả lab về initial state:

Reverse changes theo controlled order.

Ví dụ:

```text
Static routes
↓
SVIs
↓
Access assignments
↓
Port-channels
↓
VLANs
```

Nhưng chỉ remove objects do lab tạo.

Không dùng kiểu:

```text
write erase
```

hoặc:

```text
erase startup-config
```

trong shared/enterprise lab trừ khi instructor explicitly yêu cầu.

---

# PHẦN 154 — EXAMPLE CATALYST CLEANUP

Nếu known-safe standalone lab:

```text
configure terminal

no ip route 10.20.0.0 255.255.0.0 10.255.0.2

interface GigabitEthernet1/0/10
 no switchport access vlan 10

interface Port-channel1
 no switchport trunk allowed vlan 10,20,99

...
```

Tuy nhiên exact cleanup phải dựa original baseline.

Không copy blindly block này.

---

# PHẦN 155 — WHY BASELINE IS REQUIRED FOR CLEANUP

Bạn không thể biết:

```text
What should I restore?
```

nếu không biết:

```text
What existed before?
```

Do đó config backup không chỉ phục vụ catastrophic recovery.

Nó còn phục vụ:

```text
precise rollback
```

---

# PHẦN 156 — FINAL ACCEPTANCE EVIDENCE

Syllabus acceptance yêu cầu bạn chứng minh:

### Catalyst

```text
Platform/version recorded
Interface errors inspected
VLAN/trunk inspected
MAC learning inspected
L3 reachability tested
Config backup retained
Rollback verified
```

### Nexus

```text
Platform/version recorded
Interface errors inspected
VLAN/trunk inspected
MAC learning inspected
L3 reachability tested
Port-channel inspected
VRF awareness demonstrated
vPC vs standalone explained
Config backup retained
Rollback verified
```

### Failures

```text
VLAN mismatch diagnosed
Incorrect route diagnosed
Interface-down diagnosed
```

### Change

```text
Before
After
Diff
Rollback
Verification
```

Đúng với acceptance criteria của Day 8 trong syllabus. Infrastructure_Operations_14_Da…

---

# PHẦN 157 — PRACTICAL ASSESSMENT: KHÔNG ĐƯỢC CHỈ NÓI "PASS"

Ví dụ không đạt:

```text
Ping worked.
```

Đạt:

```text
PC-A 10.10.10.10 successfully reached Server-C
10.20.30.10 after the static route was restored.

Catalyst RIB selected 10.20.0.0/16 via 10.255.0.2.
NX-DC1 had the return route 10.10.0.0/16 via 10.255.0.1.
ARP resolution was present on both directly connected segments.
Server-C MAC was learned in VLAN30 through Po10.
```

Đó là verification có reasoning.

---

# PHẦN 158 — TROUBLESHOOTING MATRIX

| Symptom                                    | First checks          | Likely domain    |
| ------------------------------------------ | --------------------- | ---------------- |
| Interface down                             | admin/link/remote     | L1               |
| Up but CRC increasing                      | counters/cable/optic  | L1               |
| Gateway unreachable                        | VLAN/trunk/MAC/ARP    | L2               |
| Same VLAN fails                            | MAC/STP/trunk         | L2               |
| Same VLAN works, remote fails              | gateway/route         | L3               |
| Route exists but remote fails              | next-hop/return route | L3               |
| Some VLANs cross trunk                     | allowed VLAN list     | L2               |
| Port-channel up with fewer members         | member state/LACP     | L2               |
| Management only fails                      | VRF/management route  | Management plane |
| Connectivity returns after VLAN correction | VLAN mismatch         | L2               |
| One direction works                        | return route/policy   | L3/policy        |

---

# PHẦN 159 — KEY OPERATIONAL PRINCIPLE: VERIFY FROM BOTH ENDS

Một interface problem không nên chỉ kiểm tra local side.

Nếu:

```text
CAT-DIST Gi1/0/24
up
```

nhưng remote Nexus:

```text
Eth1/1
down
```

cần reconcile.

Trunk cũng vậy:

```text
CAT side config
+
peer side config
```

Port-channel:

```text
local LACP
+
peer LACP
```

Route:

```text
forward
+
return
```

Operations thường thất bại khi chỉ nhìn một đầu.

---

# PHẦN 160 — KEY OPERATIONAL PRINCIPLE: EXPECTED VS ACTUAL

Mọi troubleshooting nên có:

```text
Expected:
Gi1/0/10 = VLAN10

Actual:
Gi1/0/10 = VLAN20
```

Không chỉ:

```text
show interface says VLAN20
```

Không có expected baseline thì không thể gọi VLAN20 là "wrong".

---

# PHẦN 161 — KEY OPERATIONAL PRINCIPLE: ONE CHANGE AT A TIME

Nếu bạn đồng thời sửa:

```text
VLAN
route
trunk
port-channel
```

và service hồi phục, bạn không biết root cause thực sự.

Trong foundation troubleshooting:

```text
one hypothesis
→ one controlled change
→ verify
```

là phương pháp tốt hơn.

---

# PHẦN 162 — KEY OPERATIONAL PRINCIPLE: VERIFY THE FIX DID NOT BREAK SOMETHING ELSE

Ví dụ thêm VLAN10 vào trunk.

Positive:

```text
VLAN10 restored.
```

Nhưng phải check:

```text
Existing VLAN20 still works?
Port-channel healthy?
STP unchanged unexpectedly?
```

Đây là regression check.

---

# PHẦN 163 — DAY 8 PRACTICAL EVIDENCE MANIFEST EXAMPLE

```text
Evidence ID: D08-CAT-TRUNK-001
Source: CAT-ACCESS
Platform: Cisco Catalyst
OS: IOS XE
Version: <actual>
Environment: LAB
Timestamp: 2026-10-21T20:00:00+07:00
Command: show interfaces trunk
Purpose: Verify VLAN10/20/99 trunk forwarding
Expected: VLAN10,20,99 allowed
Actual: VLAN10,20,99 allowed
Ticket: LAB-D08-001
Collector: <learner>
Limitations: Virtual interface; no physical optic validation
```

---

# PHẦN 164 — HASHING CONFIG EXPORTS

Nếu config/evidence được export thành files:

```text
SHA-256
```

nên được record theo common evidence contract của chương trình. Syllabus toàn chương trình yêu cầu evidence có source, asset, collection timestamp/timezone, filter/command, file size/count/hash và giữ untouched raw export riêng với working copies. Infrastructure_Operations_14_Da…

Hash chứng minh:

```text
file bytes unchanged relative to recorded hash
```

Hash không chứng minh:

```text
configuration was correct
source device was trustworthy
collection was complete
```

---

# PHẦN 165 — DO NOT EXPOSE SECRETS

Running config có thể chứa sensitive configuration.

Evidence submission phải:

```text
Raw protected copy
+
Sanitized/redacted deliverable where required
```

Không gửi:

```text
password
private key
secret
token
real production credential
```

---

# PHẦN 166 — PRACTICAL ESCALATION RECORD EXAMPLE

```text
Observed:
Po10 remains up with one member suspended.

Impact:
Service reachable; redundancy degraded.

Read-only evidence:
show port-channel summary
show interface Ethernet1/4
show running-config interface Ethernet1/4

Finding:
Local member configuration matches expected baseline.
Physical state remains down.

Action:
No further configuration changes performed.

Escalation:
Physical/network platform team for cable/optic/remote-side validation.

Reason:
Potential Layer 1 fault outside current authorized scope.
```

Đây là một kết quả tốt.

Bạn **không bắt buộc phải tự sửa mọi lỗi**.

---

# PHẦN 167 — COMMAND CHEAT SHEET: CATALYST

## Identity

```text
show version
```

## Interface

```text
show interfaces status
show interfaces <interface>
```

## Switchport

```text
show interfaces <interface> switchport
```

## VLAN

```text
show vlan brief
```

## Trunk

```text
show interfaces trunk
```

## MAC

```text
show mac address-table
```

## ARP

```text
show ip arp
```

## STP

```text
show spanning-tree
```

## EtherChannel

```text
show etherchannel summary
```

## IP interfaces

```text
show ip interface brief
```

## Routing

```text
show ip route
```

## Neighbors

```text
show cdp neighbors
show lldp neighbors
```

## Config

```text
show running-config
show startup-config
```

---

# PHẦN 168 — COMMAND CHEAT SHEET: NEXUS

## Identity

```text
show version
```

## Interfaces

```text
show interface status
show interface Ethernet1/1
```

## VLAN

```text
show vlan brief
```

## Trunk

```text
show interface trunk
```

## MAC

```text
show mac address-table
```

## ARP

```text
show ip arp
```

## STP

```text
show spanning-tree
```

## Port-channel

```text
show port-channel summary
```

## Routes

```text
show ip route
show ip static-route
```

## VRF

```text
show vrf
```

## vPC

```text
show vpc brief
```

## Config

```text
show running-config
show startup-config
```

---

# PHẦN 169 — CONFIGURATION CHEAT SHEET: CATALYST LAB

VLAN:

```text
vlan 10
 name USERS
```

Access:

```text
interface Gi1/0/10
 switchport mode access
 switchport access vlan 10
```

Trunk:

```text
interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,99
```

LACP:

```text
interface range Gi1/0/47-48
 channel-group 1 mode active
```

SVI:

```text
interface Vlan10
 ip address 10.10.10.1 255.255.255.0
```

Routing:

```text
ip routing
```

Static route:

```text
ip route 10.20.0.0 255.255.0.0 10.255.0.2
```

---

# PHẦN 170 — CONFIGURATION CHEAT SHEET: NEXUS LAB

Features where applicable:

```text
feature lacp
feature interface-vlan
```

VLAN:

```text
vlan 30
 name SERVER
```

Access:

```text
interface Ethernet1/10
 switchport
 switchport mode access
 switchport access vlan 30
 no shutdown
```

Port-channel:

```text
interface Ethernet1/3
 channel-group 10 mode active
```

Trunk:

```text
interface port-channel10
 switchport
 switchport mode trunk
 switchport trunk allowed vlan 30,40
```

SVI:

```text
interface vlan 30
 ip address 10.20.30.1/24
 no shutdown
```

Static route:

```text
ip route 10.10.0.0/16 10.255.0.1
```

---

# PHẦN 171 — FINAL LAB WALKTHROUGH TÓM TẮT

Toàn bộ Day 8 practical có thể nhớ theo sequence:

```text
1. Identify devices
       ↓
2. Record versions
       ↓
3. Collect baseline
       ↓
4. Backup config
       ↓
5. Create VLANs
       ↓
6. Configure access
       ↓
7. Build LACP trunks
       ↓
8. Verify STP/MAC
       ↓
9. Create Catalyst SVIs
       ↓
10. Enable inter-VLAN routing
       ↓
11. Create Catalyst–Nexus routed link
       ↓
12. Create Nexus VLANs/port-channel
       ↓
13. Create Nexus SVIs
       ↓
14. Configure static routes
       ↓
15. Validate end-to-end path
       ↓
16. Correlate ARP/MAC/routes
       ↓
17. Inspect interface health
       ↓
18. Inspect management VRF/vPC
       ↓
19. Inject faults
       ↓
20. Diagnose read-only first
       ↓
21. Correct one root cause
       ↓
22. Verify forward + return path
       ↓
23. Diff configuration
       ↓
24. Rollback exercise
       ↓
25. Verify rollback
       ↓
26. Final health check
       ↓
27. Archive evidence
```

---

# PHẦN 172 — DAY 8 PRACTICAL COMPLETION CHECKLIST

## Environment

- [ ] Lab/non-production confirmed.
- [ ] Device hostnames confirmed.
- [ ] Catalyst model/version recorded.
- [ ] Nexus model/version recorded.
- [ ] Emulator limitations documented.
- [ ] Evidence directory created.
- [ ] Ticket/lab ID created.

## Catalyst read-only diagnosis

- [ ] `show version`.
- [ ] Interface state.
- [ ] Interface error counters.
- [ ] VLAN state.
- [ ] Trunk state.
- [ ] MAC learning.
- [ ] ARP.
- [ ] STP.
- [ ] EtherChannel.
- [ ] IP interfaces.
- [ ] Routing table.
- [ ] CDP/LLDP.
- [ ] Running/startup config.

## Nexus read-only diagnosis

- [ ] `show version`.
- [ ] Interface state.
- [ ] Error counters.
- [ ] VLAN.
- [ ] Trunk.
- [ ] MAC.
- [ ] ARP.
- [ ] STP.
- [ ] Port-channel.
- [ ] Routing.
- [ ] VRF.
- [ ] vPC inspected where applicable.
- [ ] Running/startup config.

## L2 configuration

- [ ] VLAN10 created.
- [ ] VLAN20 created.
- [ ] VLAN30 created.
- [ ] VLAN40 created.
- [ ] Access ports configured.
- [ ] Catalyst LACP bundle configured.
- [ ] Nexus LACP bundle configured.
- [ ] 802.1Q trunks configured.
- [ ] Required VLAN list verified.
- [ ] STP state verified.
- [ ] MAC learning verified.

## L3 configuration

- [ ] Catalyst VLAN10 SVI.
- [ ] Catalyst VLAN20 SVI.
- [ ] Catalyst inter-VLAN routing.
- [ ] Catalyst–Nexus transit.
- [ ] Nexus VLAN30 SVI.
- [ ] Nexus VLAN40 SVI.
- [ ] Nexus inter-VLAN routing.
- [ ] Catalyst static route.
- [ ] Nexus return static route.
- [ ] Connected/static routes verified.

## Connectivity

- [ ] Endpoint → local gateway.
- [ ] VLAN10 → VLAN20.
- [ ] Catalyst → Nexus transit.
- [ ] VLAN30 → VLAN40.
- [ ] PC-A → Server-C.
- [ ] Server-C → PC-A.
- [ ] Forward path documented.
- [ ] Return path documented.

## Failure Injection

- [ ] Wrong access VLAN diagnosed.
- [ ] Missing trunk VLAN diagnosed.
- [ ] Incorrect static route diagnosed.
- [ ] Interface shutdown diagnosed.
- [ ] Degraded port-channel diagnosed.
- [ ] Root cause recorded.
- [ ] Correction verified.
- [ ] No unrelated configuration change.

## Configuration control

- [ ] Pre-change running config captured.
- [ ] Pre-change startup config captured.
- [ ] Post-change config captured.
- [ ] Before/after diff reviewed.
- [ ] Intended state persisted if required.
- [ ] Rollback performed at least once.
- [ ] Rollback verified operationally.

## Evidence

- [ ] Catalyst evidence set.
- [ ] Nexus evidence set.
- [ ] Topology.
- [ ] IP/VLAN table.
- [ ] Tested paths.
- [ ] Failure evidence.
- [ ] Root-cause reasoning.
- [ ] Before/after configuration.
- [ ] Rollback evidence.
- [ ] Timestamp/timezone.
- [ ] Ticket/change reference.
- [ ] File hash where appropriate.
- [ ] No secrets exposed.

---

# PHẦN 173 — TIÊU CHUẨN ĐỂ COI MODULE THỰC HÀNH DAY 8 LÀ HOÀN THÀNH

Bạn chưa hoàn thành Day 8 nếu chỉ:

```text
configure được VLAN
```

hoặc:

```text
ping được server
```

Bạn chỉ nên coi Module Thực hành hoàn thành khi có thể chứng minh toàn bộ chuỗi:

```text
Topology understood
        ↓
Platform/version identified
        ↓
Baseline collected
        ↓
Configuration implemented
        ↓
L2 state verified
        ↓
L3 state verified
        ↓
End-to-end flow proven
        ↓
Failure intentionally introduced
        ↓
Fault localized systematically
        ↓
Root cause demonstrated with evidence
        ↓
Correction authorized and applied
        ↓
Service restored
        ↓
Configuration compared
        ↓
Rollback demonstrated
        ↓
Rollback verified
        ↓
Evidence archived
        ↓
Escalation boundary understood
```

Đó mới chính xác là mục tiêu operational của syllabus Day 8: không chỉ biết **Cisco networking commands**, mà phải chứng minh được khả năng vận hành **Catalyst IOS/IOS XE và Nexus NX-OS như hai platform riêng biệt**, troubleshoot L2/L3 có phương pháp, thực hiện safe change, verify, backup, rollback và tạo evidence có thể review/audit được. Infrastructure_Operations_14_Da…
