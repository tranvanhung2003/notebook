# Day 9 — Module Lý thuyết: FortiGate Firewall & F5 BIG-IP Load Balancer Operations

Phần này trình bày **toàn bộ kiến thức lý thuyết Day 9**, chưa đi vào bài lab hoàn chỉnh. Tôi sẽ bám sát syllabus, đồng thời bổ sung các kiến thức nền cần thiết để bạn có thể reasoning như một Infrastructure Operations engineer thay vì chỉ nhớ menu hoặc command.

Theo syllabus, Day 9 bắt buộc bao phủ hai khối chính: **Fortinet FortiGate/FortiOS firewall operations** và **F5 BIG-IP LTM/load balancing/SSL/API operations**. Nội dung bao gồm FortiGate architecture, interfaces, zones, routing, objects, policies, NAT, security profiles, sessions, logs, configuration backup/revision, HA và troubleshooting; phía BIG-IP gồm nodes, pools, pool members, health monitors, Virtual Servers, load-balancing methods, persistence, SNAT, TCP/HTTP profiles, SSL/TLS, certificate/SNI, `tmsh`, iControl REST API, HTTP/JSON/cURL và safe configuration changes. Infrastructure_Operations_14_Da…

> **Version note:** Các concept dưới đây mang tính operational và tương đối ổn định. Cú pháp chính xác, menu, defaults và một số behavior có thể khác giữa FortiOS/BIG-IP releases. Các chi tiết version-sensitive tôi đối chiếu với FortiOS 7.6.x và BIG-IP documentation hiện hành; khi vận hành môi trường thật vẫn phải xác nhận exact product/version/runbook trước khi thay đổi configuration.

---

# 1. Mental model tổng thể của Day 9

Trước khi học từng product, cần xây mental model này:

```text
Client
  |
  | DNS resolves application.example.com
  v
Network / Router
  |
  v
FortiGate
  |
  | Route
  | Firewall Policy
  | Security Profile
  | NAT
  | Stateful Session
  |
  v
F5 BIG-IP Virtual Server
  |
  | TCP/TLS processing
  | HTTP profile
  | Persistence
  | Load-balancing decision
  |
  v
Pool
  |
  +--------------+
  |              |
Pool Member 1  Pool Member 2
10.20.30.11    10.20.30.12
:8443          :8443
  |              |
Application    Application
```

Trong enterprise environment, khi user báo:

> “Website không truy cập được.”

thì câu đó gần như chưa cho bạn thông tin kỹ thuật nào.

Một Infrastructure Operations engineer cần chia failure domain thành các tầng:

```text
DNS
 ↓
IP routing
 ↓
Firewall policy
 ↓
NAT
 ↓
TCP connection
 ↓
TLS handshake
 ↓
BIG-IP Virtual Server
 ↓
Pool selection
 ↓
Health monitor
 ↓
Backend TCP/TLS
 ↓
HTTP
 ↓
Application
```

Nếu không chia layer như vậy, troubleshooting thường trở thành:

```text
ping
→ restart service
→ restart firewall?
→ disable SSL verification?
→ reboot?
```

Đó là cách làm rủi ro và không có evidence.

Day 9 dạy bạn chuyển từ kiểu troubleshooting “thử command” sang **hypothesis-driven troubleshooting**:

```text
Symptom
→ Xác định failure layer
→ Collect read-only evidence
→ Form hypothesis
→ Verify hypothesis
→ Minimal corrective action
→ Re-test
→ Preserve evidence
→ Rollback/escalate nếu cần
```

Syllabus cũng nhấn mạnh phải phân biệt rõ **transport/TLS failure, HTTP failure và application health failure**, đồng thời validation certificate phải xét trust, chain, hostname và expiry. Infrastructure_Operations_14_Da…

---

# 2. Syllabus coverage của Module Lý thuyết

| Khối                 | Nội dung bắt buộc                                                                     |
| -------------------- | ------------------------------------------------------------------------------------- |
| FortiGate foundation | FortiGate architecture, FortiOS, administrative access, CLI                           |
| Network objects      | interfaces, zones, routing, address objects, service objects                          |
| Policy               | firewall policies, policy ordering, NAT, security profiles                            |
| Runtime state        | session tables, VPN awareness                                                         |
| Visibility           | traffic logs, security events, policy hit analysis                                    |
| Safety               | configuration revision, configuration backup, HA awareness                            |
| Troubleshooting      | firewall troubleshooting và FortiGate CLI reasoning                                   |
| BIG-IP foundation    | BIG-IP architecture, LTM                                                              |
| BIG-IP objects       | nodes, pools, pool members, health monitors, Virtual Servers/listeners                |
| Traffic management   | load-balancing methods, persistence, SNAT                                             |
| Profiles             | TCP profile, HTTP profile                                                             |
| SSL/TLS              | Client SSL, Server SSL, certificate/key, chain, expiry/renewal, SNI, HTTPS validation |
| CLI                  | `tmsh`                                                                                |
| API                  | iControl REST, HTTP methods, authentication, JSON, cURL, response codes               |
| Safety               | safe configuration changes                                                            |
| Automation           | timeout, retry limits, assertion behavior, secret protection                          |

Day 9 sau cùng phải giúp bạn tạo được evidence cho cả **FortiGate CLI route/policy/session diagnosis** và **BIG-IP Virtual Server/Pool/Monitor/TLS/API validation**. Infrastructure_Operations_14_Da…

---

# PHẦN I — FORTIGATE / FORTIOS

# 3. FortiGate là gì?

## 3.1 What

**FortiGate** là security appliance/platform của Fortinet chạy **FortiOS**.

Trong scope Day 9, FortiGate cần được hiểu trước hết như một **stateful firewall** đặt giữa các network security zones và chịu trách nhiệm quyết định traffic nào được phép đi qua.

Một cách đơn giản:

```text
Source network
      |
      v
+------------------+
|    FortiGate     |
|                  |
| Interface        |
| Route lookup     |
| Policy match     |
| Security inspect |
| NAT              |
| Session state    |
+------------------+
      |
      v
Destination network
```

Fortinet mô tả firewall policy là thành phần trung tâm của FortiGate forwarding behavior: traffic đi xuyên FortiGate cần được associate với policy phù hợp; policy quyết định traffic đi đâu, được xử lý như thế nào và có được phép đi qua hay không. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/118003/policies?utm_source=chatgpt.com)

---

# 4. FortiGate architecture ở góc nhìn Operations

Bạn không cần trở thành Fortinet platform engineer trong Day 9, nhưng nên chia FortiGate thành ba góc nhìn logical.

| Plane                 | Vai trò                                                               |
| --------------------- | --------------------------------------------------------------------- |
| Management plane      | GUI, CLI, administrator login, configuration, monitoring              |
| Control plane         | routing information, HA/control protocols, policy/configuration logic |
| Data/forwarding plane | xử lý packet/session thực tế                                          |

Điểm quan trọng:

**Management connectivity hoạt động không chứng minh data traffic hoạt động.**

Ví dụ bạn SSH được vào firewall:

```text
Admin PC
   |
   +------ SSH -----> FortiGate Management
```

nhưng application path:

```text
Client
   |
   +------ HTTPS ----> FortiGate ----> BIG-IP
```

vẫn có thể lỗi hoàn toàn.

Do đó khi troubleshooting không được suy luận:

> “Firewall reachable nên firewall chắc không có lỗi.”

Reachable management plane chỉ chứng minh một management path.

---

# 5. FortiOS

**FortiOS** là operating system của FortiGate.

Operationally, bạn tương tác với FortiOS qua:

```text
GUI
CLI
management integrations
logging/monitoring interfaces
```

Trong Day 9, CLI đặc biệt quan trọng vì một số evidence tốt nhất về:

```text
routing table
sessions
packet flow
diagnostic state
```

được lấy trực tiếp từ CLI.

Một nguyên tắc quan trọng là luôn xác định version trước khi sử dụng một runbook.

Ví dụ:

```text
FortiOS 7.0.x
FortiOS 7.2.x
FortiOS 7.4.x
FortiOS 7.6.x
```

không nên mặc định rằng mọi menu, default hoặc syntax đều hoàn toàn giống nhau.

---

# 6. Administrative access

## 6.1 Vì sao management access là security boundary?

Firewall thường là một trong những devices có quyền lực lớn nhất trong infrastructure.

Administrator của firewall có thể ảnh hưởng tới:

```text
network segmentation
production access
Internet access
NAT
VPN
security inspection
logging
service availability
```

Do đó administrative access phải được xem như privileged access.

Các nguyên tắc quan trọng:

| Control                              | Ý nghĩa                                   |
| ------------------------------------ | ----------------------------------------- |
| Least privilege                      | Chỉ cấp permission cần thiết              |
| MFA nếu environment hỗ trợ           | Giảm credential compromise risk           |
| Dedicated management network         | Tách management traffic khỏi user traffic |
| HTTPS/SSH thay vì insecure protocols | Bảo vệ credentials/session                |
| Named account                        | Audit được ai thực hiện                   |
| Approved privileged workflow         | Gắn action với ticket/change              |
| Session logging/audit                | Có traceability                           |

FortiOS cho phép interface được cấu hình để nhận các management protocols nhất định; ví dụ tài liệu hiện tại minh họa `allowaccess ssh` và dùng `show system interface` để xác minh interface configuration. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/901037/connecting-to-the-cli?utm_source=chatgpt.com)

---

# 7. FortiGate CLI

CLI có hai loại command cần phân biệt.

### Read-only / inspection

Ví dụ:

```bash
get system status
show system interface
get router info routing-table all
diagnose sys session list
```

Các command này chủ yếu dùng để xem state/configuration.

### Configuration-changing

Ví dụ dạng:

```bash
config ...
    edit ...
        set ...
    next
end
```

Những command này thay đổi configuration.

Operationally, distinction này cực kỳ quan trọng.

Một L1/L2 engineer thường được phép chạy nhiều read-only diagnostics nhưng **không mặc nhiên được phép modify firewall policy**.

---

# 8. Interfaces

FortiGate interfaces là nơi traffic đi vào hoặc ra firewall.

Một interface có thể mang các thông tin như:

```text
interface name
IP address
subnet
administrative status
link state
VLAN context
role
management access
```

Ví dụ:

```text
port1 = external
port2 = internal
port3 = DMZ
```

Application flow:

```text
Client
  |
port2
  |
FortiGate
  |
port3
  |
BIG-IP
```

Firewall policy thường có:

```text
srcintf = port2
dstintf = port3
```

Nếu traffic thực tế vào `port4` thay vì `port2`, policy có thể không match dù source/destination IP hoàn toàn đúng.

FortiOS firewall policy fields hiện tại bao gồm các yếu tố như `srcintf`, `dstintf`, `srcaddr`, `dstaddr`, `schedule` và `service`. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/656084/firewall-policy?utm_source=chatgpt.com)

---

# 9. Zones

**Zone** là logical grouping của interfaces.

Ví dụ thay vì policy riêng:

```text
LAN1 → DMZ
LAN2 → DMZ
LAN3 → DMZ
```

có thể có design:

```text
LAN-ZONE
 ├── LAN1
 ├── LAN2
 └── LAN3

LAN-ZONE → DMZ-ZONE
```

Zone giúp policy design dễ quản lý hơn, nhưng có một nguy cơ operational:

> Engineer nhìn thấy interface vật lý nhưng quên rằng policy đang reference zone.

Khi troubleshoot, luôn kiểm tra cả:

```text
physical/logical interface
zone membership
policy references
```

Exact intra-zone behavior hoặc GUI presentation có thể phụ thuộc version/configuration, nên không nên đoán.

---

# 10. Routing trên FortiGate

Firewall không chỉ “allow hay deny”.

Nó cũng cần biết **traffic phải đi đâu**.

Ví dụ:

```text
Destination: 10.20.30.15
```

FortiGate phải tìm route:

```text
10.20.30.0/24 → port3
```

Nếu route không tồn tại hoặc route sai:

```text
Policy đúng
Service đúng
NAT đúng
```

traffic vẫn fail.

Tài liệu Fortinet hiện dùng:

```bash
get router info routing-table all
```

để xem routing table. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/479509/dynamic-routing?utm_source=chatgpt.com)

Output conceptual:

```text
S* 0.0.0.0/0 via 192.0.2.1, wan1
C  10.10.10.0/24 is directly connected, port2
S  10.20.30.0/24 via 10.10.99.1, port3
```

Trong đó:

```text
C = connected
S = static
O = OSPF
B = BGP
```

tùy routing protocols được triển khai.

---

# 11. Longest Prefix Match

Foundation networking từ Day 8 tiếp tục áp dụng.

Giả sử routing table:

```text
10.0.0.0/8      via Router-A
10.20.0.0/16    via Router-B
10.20.30.0/24   via Router-C
```

Traffic tới:

```text
10.20.30.15
```

sẽ sử dụng route cụ thể hơn:

```text
10.20.30.0/24
```

chứ không phải `/16` hay `/8`.

Do đó một common troubleshooting mistake là:

> “Tôi thấy route 10.0.0.0/8 nên destination có route.”

Có route không đồng nghĩa đó là **route actually selected**.

---

# 12. Address objects

Thay vì viết IP trực tiếp trong mọi policy, firewall dùng **address objects**.

Ví dụ:

```text
APP_CLIENTS
10.10.10.0/24
```

và:

```text
BIGIP_VIP
203.0.113.50/32
```

Policy có thể đọc:

```text
Source: APP_CLIENTS
Destination: BIGIP_VIP
```

thay vì:

```text
10.10.10.0/24 → 203.0.113.50
```

Lợi ích:

```text
reusability
readability
centralized management
auditability
```

Nhưng object abstraction cũng tạo lỗi.

Ví dụ:

```text
Object name:
APP_PROD

Actual content:
10.20.0.0/16
```

Trong khi production clients thực tế:

```text
10.30.0.0/16
```

Tên object có vẻ đúng nhưng content sai.

Do đó troubleshooting phải kiểm tra **object definition**, không chỉ object name.

---

# 13. Address groups

Nhiều address objects có thể được gom thành group.

Ví dụ:

```text
PROD_WEB_SERVERS
 ├── web01
 ├── web02
 └── web03
```

Operational benefit:

```text
Policy change không cần thêm từng server riêng biệt.
```

Operational risk:

```text
thêm nhầm object vào group
xóa nhầm object
nested group khó đọc
scope quá rộng
```

Principle:

> Object abstraction giúp vận hành tốt hơn chỉ khi naming, ownership và review tốt.

---

# 14. Service objects

Service object đại diện cho network service/protocol/port.

Ví dụ:

```text
HTTPS
TCP/443
```

hoặc custom application:

```text
APP_API
TCP/8443
```

Một firewall policy có thể match:

```text
source
destination
protocol/service
interfaces
schedule
```

Fortinet documentation hiện mô tả services dưới dạng các traffic types/protocols và logical destination ports được dùng trong firewall policies. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/656084/firewall-policy?utm_source=chatgpt.com)

Một lỗi phổ biến:

```text
Application:
TCP/8443

Firewall service:
TCP/443
```

Policy có vẻ đúng về source/destination nhưng không match.

---

# 15. Firewall policy

Đây là concept quan trọng nhất phía FortiGate.

Một policy cơ bản có mental model:

```text
Incoming Interface
        AND
Source Address
        AND
Destination Address
        AND
Service
        AND
Schedule
        AND
Outgoing Interface
        ↓
      Action
```

Ví dụ:

```text
Incoming:  LAN
Outgoing:  DMZ
Source:    APP_CLIENTS
Destination: BIGIP_VIP
Service:   HTTPS
Schedule:  always
Action:    ACCEPT
```

Fortinet mô tả traffic phải match policy parameters; nếu không có policy phù hợp, traffic bị deny. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/656084/firewall-policy?utm_source=chatgpt.com)

---

# 16. Stateful firewall

FortiGate là **stateful firewall**.

Điều này nghĩa là firewall không xem mỗi packet hoàn toàn độc lập.

Nó tạo **session state**.

Ví dụ TCP connection:

```text
Client: 10.10.10.50:52134
Server: 203.0.113.50:443
Protocol: TCP
```

FortiGate có thể tạo session đại diện connection đó.

Do session đã được establish, return traffic thường được associated với session hiện hữu thay vì yêu cầu một independent outbound policy cho từng reply packet.

Fortinet mô tả session như communication channel và lưu session information trong session table; CLI session table đặc biệt hữu ích cho troubleshooting. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.4.7/administration-guide/562859/using-a-session-table?utm_source=chatgpt.com)

---

# 17. Policy ordering

Policy order cực kỳ quan trọng.

Giả sử:

```text
Policy 10
Source: 10.10.0.0/16
Destination: ANY
Service: HTTPS
Action: DENY
```

và:

```text
Policy 20
Source: 10.10.10.0/24
Destination: BIGIP_VIP
Service: HTTPS
Action: ALLOW
```

Nếu broader deny rule được kiểm tra trước rule cụ thể allow, traffic có thể bị bắt bởi policy không mong muốn.

FortiOS GUI hiện có các views thể hiện policies **theo thứ tự chúng được kiểm tra cho matching traffic**, và policies có thể được di chuyển trong sequence. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/656084/firewall-policy?utm_source=chatgpt.com)

Đây là nguyên tắc chung:

```text
Correct policy object
+
wrong policy position
=
traffic vẫn fail
```

---

# 18. Implicit deny

Nếu traffic không match một valid allow policy, mặc định phải coi nó là denied.

Operational implication:

Không được hỏi duy nhất:

> “Có deny rule nào không?”

Phải hỏi:

> “Traffic có match một permitted policy hay không?”

Hai câu khác nhau.

Không có explicit deny log cũng chưa chắc traffic được allow.

---

# 19. Policy match dimensions

Trong troubleshooting, hãy xây tuple:

```text
Ingress interface
Source IP
Source port
Destination IP
Destination port
Protocol
Egress interface
```

Ví dụ:

```text
Ingress: port2
Source: 10.10.10.50
Source port: 53100
Destination: 203.0.113.50
Destination port: 443
Protocol: TCP
Egress: port3
```

Sau đó mới đối chiếu policy.

FortiOS cung cấp policy match capabilities và yêu cầu route liên quan phải tồn tại để policy work đúng. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/656084/firewall-policy?utm_source=chatgpt.com)

---

# 20. Policy hit analysis

Policy hit counters giúp trả lời:

```text
Policy này có thực sự được dùng không?
```

Ví dụ:

```text
Policy 100 hits: 2,451,900
Policy 101 hits: 0
```

Nếu bạn kỳ vọng traffic match Policy 101 nhưng counter vẫn 0, đó là signal hữu ích.

Tuy nhiên:

```text
Hit count > 0
```

không chứng minh **traffic của bạn** match.

Có thể traffic khác match policy đó.

Do đó policy hit analysis luôn phải kết hợp với:

```text
timestamps
source/destination
session
logs
```

FortiOS 7.6 documentation hiện hỗ trợ filtering policy statistics bằng metrics như Bytes, Packets, Hit Count và Last Used. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/656084/firewall-policy?utm_source=chatgpt.com)

---

# 21. NAT — Network Address Translation

NAT là một trong những nguồn gây nhầm lẫn lớn nhất trong firewall troubleshooting.

Cần tách:

```text
SNAT = Source NAT
DNAT = Destination NAT
```

---

# 22. SNAT

Source NAT thay đổi **source IP**.

Ví dụ trước NAT:

```text
10.10.10.50:51000
      ↓
203.0.113.50:443
```

Sau SNAT:

```text
198.51.100.10:62001
      ↓
203.0.113.50:443
```

Backend nhìn thấy:

```text
198.51.100.10
```

thay vì:

```text
10.10.10.50
```

SNAT thường dùng để:

```text
Internet access
return-path control
hide internal addresses
avoid asymmetric routing
```

---

# 23. Policy NAT và Central SNAT

Đây là chi tiết rất dễ phụ thuộc FortiOS design.

Một FortiGate có thể sử dụng NAT gắn với firewall policy hoặc **Central SNAT** tùy configuration.

Fortinet hiện ghi rõ rằng khi Central NAT được enabled, SNAT được quản lý qua `central-snat-map`; NAT rules được đọc theo order và Central SNAT được áp dụng sau security policy. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/421028/central-snat?utm_source=chatgpt.com)

Điều này tạo một troubleshooting question cực kỳ quan trọng:

> Environment này đang dùng policy-based NAT hay Central NAT?

Không trả lời câu này thì rất dễ nhìn sai chỗ.

---

# 24. DNAT / Virtual IP

Destination NAT thay đổi **destination IP**.

Ví dụ client gọi:

```text
203.0.113.50:443
```

FortiGate translate thành:

```text
10.20.30.50:443
```

Fortinet dùng **Virtual IP — VIP** objects cho nhiều DNAT use cases. Static VIP có thể map external IP tới internal IP; port-forwarding VIP có thể map specific ports. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/510402/static-virtual-ips?utm_source=chatgpt.com)

Ví dụ:

```text
Public:
203.0.113.50:443

Mapped:
10.20.30.50:443
```

---

# 25. Double NAT scenario với BIG-IP

Một enterprise path có thể có NAT ở nhiều tầng.

Ví dụ:

```text
Client
10.1.1.50
 |
 v
FortiGate DNAT
203.0.113.50 → 172.16.10.50
 |
 v
BIG-IP VIP
172.16.10.50:443
 |
 v
BIG-IP SNAT
172.16.20.100
 |
 v
Backend
10.20.30.11:8443
```

Nếu backend log hiển thị:

```text
172.16.20.100
```

nhưng user là:

```text
10.1.1.50
```

đây có thể là expected architecture chứ không phải spoofing.

Do đó khi correlate logs phải biết NAT boundaries.

---

# 26. NAT và return path

Một trong những use cases lớn của SNAT là đảm bảo reply đi lại qua đúng device.

Ví dụ không SNAT:

```text
Client
10.10.10.50
   ↓
BIG-IP
   ↓
Backend
10.20.30.11
```

Backend default gateway có thể gửi reply theo một route khác:

```text
Backend
   |
   +---- Router khác ----> Client
```

Tạo asymmetric path.

Với SNAT:

```text
Backend sees source = BIG-IP SNAT IP
```

nên reply quay về BIG-IP.

Ý tưởng tương tự cũng xuất hiện tại firewall NAT design.

---

# 27. Security profiles

Firewall policy `ALLOW` không nhất thiết là cuối cùng của processing.

FortiGate có thể áp dụng security inspection profiles.

Examples thường gặp:

```text
IPS
Antivirus
Application Control
Web Filter
SSL inspection
DNS/security inspection
```

Exact licensed features và available profiles phụ thuộc deployment.

Mental model:

```text
Policy says ACCEPT
        ↓
Security inspection says?
        ↓
ALLOW / BLOCK / RESET / INSPECT
```

Do đó symptom:

```text
Policy matched ACCEPT
```

không đủ để kết luận firewall “không phải nguyên nhân”.

---

# 28. Flow-based và proxy-based inspection awareness

FortiOS hiện hỗ trợ inspection mode tùy policy như **Flow-based** hoặc **Proxy-based** trong những contexts phù hợp. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/656084/firewall-policy?utm_source=chatgpt.com)

Day 9 không cần đi sâu engine internals, nhưng phải hiểu:

```text
inspection behavior
performance
session handling
logging
TLS inspection behavior
```

có thể bị ảnh hưởng bởi inspection design.

Khi troubleshoot security inspection, không được tùy tiện disable security profile trong production chỉ để “test”.

Cách an toàn hơn là:

```text
collect evidence
identify profile
review logs
reproduce in lab
obtain approved change if bypass test is required
```

---

# 29. Session table

Session table là một trong những công cụ mạnh nhất của FortiGate troubleshooting.

Fortinet mô tả session table như nơi chứa information của open communications và đặc biệt hữu ích để xác minh connection thực tế. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.4.7/administration-guide/562859/using-a-session-table?utm_source=chatgpt.com)

Một session có thể giúp bạn trả lời:

```text
Traffic có đến FortiGate không?
Policy ID nào được dùng?
Source/destination là gì?
NAT có xảy ra không?
TCP state là gì?
```

---

# 30. Vì sao session quan trọng hơn config đơn thuần?

Configuration nói:

> “Firewall nên làm gì.”

Session nói:

> “Firewall đang xử lý connection cụ thể này như thế nào.”

Đây là difference giữa:

```text
desired/configured state
```

và:

```text
runtime state
```

Một engineer mạnh luôn kiểm tra cả hai.

---

# 31. Existing session và configuration change

Một mistake phổ biến:

```text
Sửa policy
→ refresh browser
→ vẫn lỗi
→ kết luận sửa policy không đúng
```

Nhưng application có thể đang sử dụng connection/session hiện hữu.

Bạn không nên mặc định mọi config change sẽ tạo lại mọi runtime session ngay lập tức.

Correct reasoning:

```text
Check existing session
Create a genuinely new test connection
Compare policy/session
```

Việc clear session là disruptive action.

Không được chạy kiểu:

```text
clear all sessions
```

trên production chỉ để troubleshooting nếu chưa có authorization/impact assessment.

---

# 32. Session filtering

FortiOS hiện hỗ trợ filter session theo nhiều attributes như:

```text
source IP
destination IP
source port
destination port
policy ID
interface
protocol
VDOM
```

và dùng:

```bash
diagnose sys session list
```

để hiển thị sessions. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/cli-reference/235530229/diagnose-sys?utm_source=chatgpt.com)

Ví dụ read-only conceptual workflow:

```bash
diagnose sys session filter clear
diagnose sys session filter src 10.10.10.50
diagnose sys session filter dst 203.0.113.50
diagnose sys session list
```

Exact filter arguments cần kiểm tra release đang chạy.

---

# 33. VPN awareness

Syllabus chỉ yêu cầu **VPN awareness**, không yêu cầu trở thành VPN engineer.

Bạn cần hiểu VPN có thể thay đổi traffic path.

Ví dụ:

```text
Branch client
   |
IPsec tunnel
   |
FortiGate
   |
Firewall policy
   |
BIG-IP
```

Troubleshooting phải hỏi:

```text
Tunnel up không?
Traffic selector đúng không?
Route có đi qua tunnel không?
Firewall policy đúng direction không?
NAT có vô tình applied không?
```

Điểm quan trọng:

> “VPN tunnel up” không chứng minh application traffic chạy.

Control-plane tunnel state và actual traffic forwarding là hai bằng chứng khác nhau.

---

# 34. Traffic logs

Traffic log thường có các fields operationally useful như:

```text
timestamp
source IP
destination IP
source port
destination port
protocol
action
policy
interfaces
bytes
session identifier
NAT information
```

Exact fields phụ thuộc log configuration/version.

Một good evidence record phải cho phép người review trả lời:

```text
Ai?
Từ đâu?
Đến đâu?
Khi nào?
Port/protocol gì?
Policy nào?
Allow hay deny?
```

---

# 35. Security events

Security event khác traffic log thuần.

Traffic log có thể nói:

```text
traffic allowed by policy
```

nhưng security engine có thể tạo event:

```text
IPS signature triggered
application blocked
malware detected
web category blocked
certificate inspection issue
```

Do đó khi application fail:

```text
Firewall policy log
+
security event log
```

phải được xem cùng nhau khi applicable.

---

# 36. Configuration revision và backup

Trước configuration change, phải có khả năng quay về known-good state.

Fortinet hiện nhấn mạnh phải backup configuration; FortiOS hỗ trợ backup qua GUI/CLI và có các options như encrypted backup hoặc password masking tùy format/workflow. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/702257/configuration-backups-and-reset?utm_source=chatgpt.com)

Operational process:

```text
Pre-change backup
        ↓
Record version
        ↓
Record current state
        ↓
Apply approved change
        ↓
Verify
        ↓
Keep post-change evidence
```

Backup không chỉ phục vụ disaster.

Nó cũng phục vụ:

```text
diff
rollback planning
audit
change review
```

---

# 37. Backup không đồng nghĩa rollback guaranteed

Một rất quan trọng:

```text
Backup exists
≠
Rollback guaranteed
```

Vì restore có thể:

```text
require reboot
replace more configuration than intended
interact with HA state
cause traffic impact
depend on exact firmware/version
```

Do đó change plan phải nói rõ **backout method**, không chỉ:

> “Có backup nên rollback được.”

---

# 38. HA awareness

**High Availability — HA** giúp giảm service interruption khi một unit fail.

Fortinet FGCP là một HA solution cho FortiGate clustering. Current documentation mô tả cluster members phải đáp ứng compatibility requirements như model/firmware phù hợp với design. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.2/administration-guide/62403/fgcp?utm_source=chatgpt.com)

Day 9 không yêu cầu bạn build HA cluster, nhưng bạn phải hỏi:

```text
Device standalone hay HA?
Node nào active?
Node nào standby?
Config sync healthy không?
Session synchronization có relevant không?
Mình đang SSH vào member nào?
```

Nếu không hiểu HA, bạn có thể troubleshoot đúng config nhưng trên wrong node/context.

---

# 39. HA và change safety

Trước firewall change trong HA environment:

```text
HA health
Config sync
Active/standby state
Recent failover
Peer reachability
```

phải được xem như pre-checks.

Không được assumption:

> “HA có nên change chắc chắn không downtime.”

HA giảm risk nhưng không loại bỏ change risk.

---

# 40. FortiGate troubleshooting methodology

Đây là flow rất quan trọng.

| Thứ tự | Kiểm tra     | Câu hỏi                                                      |
| ------ | ------------ | ------------------------------------------------------------ |
| 1      | Scope        | Một user hay tất cả users? Một VIP hay nhiều services?       |
| 2      | Interface    | Packet vào đúng interface không? Link/interface UP không?    |
| 3      | Route        | Destination có route không? Egress interface nào?            |
| 4      | Objects      | Source/destination/service objects có đúng không?            |
| 5      | Policy       | Policy nào match? Order đúng không?                          |
| 6      | NAT          | SNAT/DNAT mapping có đúng không? Central NAT hay policy NAT? |
| 7      | Session      | Session tồn tại không? Policy ID/NAT/state là gì?            |
| 8      | Security     | Security profile có block/reset không?                       |
| 9      | Logs         | Traffic/security logs nói gì?                                |
| 10     | Packet/debug | Chỉ dùng khi evidence trước chưa đủ                          |
| 11     | Change       | Chỉ sau approval và root-cause evidence                      |
| 12     | Verify       | Test lại đúng original flow                                  |

---

# 41. FortiGate packet capture

Một packet capture giúp trả lời:

```text
Packet có vào firewall không?
Có ra firewall không?
Reply có quay lại không?
TCP handshake diễn ra đến đâu?
```

Concept:

```text
Ingress:
Client SYN → Firewall

Egress:
Firewall SYN → Server

Return:
Server SYN/ACK → Firewall

Return:
Firewall SYN/ACK → Client
```

Nếu thấy:

```text
SYN enters
no SYN leaves
```

failure domain nằm tại hoặc trước forwarding decision.

Nếu:

```text
SYN enters
SYN leaves
no SYN/ACK returns
```

có thể downstream route/server/firewall/backend issue.

---

# 42. `diagnose debug flow`

`diagnose debug flow` mạnh hơn packet capture vì nó có thể cho insight vào processing path.

Fortinet documentation hiện hướng dẫn sử dụng filters và có thể limit số trace; cũng có explicit command để stop trace. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/54688/debugging-the-packet-flow?utm_source=chatgpt.com)

Safe pattern trong lab:

```bash
diagnose debug reset
diagnose debug flow filter clear
diagnose debug flow filter saddr 10.10.10.50
diagnose debug flow filter daddr 203.0.113.50
diagnose debug flow show function-name enable
diagnose debug enable
diagnose debug flow trace start 100
```

Sau khi đủ evidence:

```bash
diagnose debug flow trace stop
diagnose debug disable
diagnose debug reset
```

Syllabus yêu cầu debug phải **tightly filtered, time-bounded và stop sau khi sử dụng**. Infrastructure_Operations_14_Da…

---

# 43. Hardware offload caveat

Đây là **kiến thức bổ sung quan trọng**.

Một số FortiGate platforms có hardware acceleration/offload. Khi traffic được hardware offload, một số software diagnostic tools có thể không thấy toàn bộ traffic như bạn kỳ vọng.

Fortinet documentation lưu ý packet/debug visibility có thể chịu ảnh hưởng bởi hardware acceleration trên các platforms tương ứng. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/656084/firewall-policy?utm_source=chatgpt.com)

Operational lesson:

> “Debug không thấy packet” không tự động chứng minh packet không tồn tại.

Phải biết limitations của diagnostic tool.

Không được disable hardware acceleration trên production chỉ để “nhìn cho rõ” nếu chưa có approved change.

---

# PHẦN II — F5 BIG-IP / LTM

# 44. F5 BIG-IP là gì?

Trong Day 9, trọng tâm là **BIG-IP Local Traffic Manager — LTM**.

Mental model:

```text
Client
   |
   v
Virtual Server / Listener
   |
   v
Traffic profiles
   |
   v
Persistence decision
   |
   v
Load-balancing decision
   |
   v
Pool
   |
   +------ Pool Member A
   |
   +------ Pool Member B
```

Pool/member relationship được F5 định nghĩa rõ: pool chứa pool members; khi pool được associated với Virtual Server, LTM có thể direct traffic nhận bởi Virtual Server tới member trong pool. [techdocs.f5.com](https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/ltm-concepts-11-5-1/6.html?utm_source=chatgpt.com)

---

# 45. BIG-IP architecture mental model

Operationally, BIG-IP có hai thế giới bạn phải phân biệt:

```text
Management/control
```

và:

```text
traffic processing
```

Bạn có thể:

```text
SSH vào BIG-IP thành công
GUI accessible
API responds
```

nhưng Virtual Server vẫn down.

Do đó management reachability ≠ application service availability.

---

# 46. Virtual Server

**Virtual Server** là một front-end traffic object.

Thường có mental model:

```text
Destination IP
+
Destination port
+
IP protocol
+
profiles
+
pool
```

Ví dụ:

```text
Virtual Server:
APP_PROD_HTTPS

Destination:
203.0.113.50:443

Protocol:
TCP

Pool:
APP_PROD_POOL
```

F5 iControl REST API hiện mô tả `destination` của Virtual Server là virtual address + port nơi Virtual Server listens for connections; Virtual Server cũng có thể reference default pool. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/APIRef_tm_ltm_virtual.html?utm_source=chatgpt.com)

---

# 47. Listener

“Listener” trong operational language nghĩa là:

```text
IP + port + protocol
```

mà load balancer nhận traffic.

Ví dụ:

```text
203.0.113.50:443/TCP
```

Nếu DNS resolve đúng nhưng Virtual Server không listen trên expected destination, request fail trước khi load balancing xảy ra.

---

# 48. Virtual Address versus Virtual Server

Hai khái niệm dễ nhầm.

```text
Virtual Address
= IP address concept
```

Trong khi:

```text
Virtual Server
= traffic listener/service definition
```

Ví dụ một IP có thể liên quan nhiều services:

```text
203.0.113.50:80
203.0.113.50:443
```

tùy configuration.

F5 API documentation cũng tách object `/ltm/virtual-address` khỏi `/ltm/virtual`. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/APIRef_tm_ltm_virtual.html?utm_source=chatgpt.com)

---

# 49. Node

**Node** thường đại diện backend system/IP ở LTM.

Ví dụ:

```text
Node web01
IP 10.20.30.11
```

Node không nhất thiết đại diện service port cụ thể.

---

# 50. Pool Member

**Pool Member** thường là:

```text
Node/IP + service port
```

Ví dụ:

```text
10.20.30.11:8443
```

Trong cùng server:

```text
10.20.30.11:8080
10.20.30.11:8443
```

là hai service endpoints khác nhau.

Một node có thể reachable nhưng member cụ thể down:

```text
ICMP ping works
```

nhưng:

```text
TCP/8443 closed
```

Do đó:

> “Server ping được” không chứng minh Pool Member healthy.

---

# 51. Pool

**Pool** là collection của Pool Members.

Ví dụ:

```text
APP_POOL
 ├── 10.20.30.11:8443
 ├── 10.20.30.12:8443
 └── 10.20.30.13:8443
```

F5 cho phép pool có health monitors, load-balancing method và các behavior khác. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-14-1-0/big-ip-local-traffic-management-basics-14-1-0/about-pools.html?utm_source=chatgpt.com)

---

# 52. Object dependency

Đây là dependency map bạn phải nhớ:

```text
Virtual Server
      |
      v
     Pool
      |
      +----------+
      |          |
      v          v
 Member A     Member B
      |
      v
    Node
```

Ngoài ra:

```text
Virtual Server
   |
   +---- TCP profile
   +---- HTTP profile
   +---- Client SSL profile
   +---- Server SSL profile
   +---- Persistence profile
```

Vì vậy “Virtual Server UP” không phải toàn bộ health story.

---

# 53. Health Monitor

**Health Monitor** kiểm tra availability của backend resource.

F5 mô tả health monitors là cơ chế kiểm tra availability của devices/services; nếu monitored resource không đáp ứng phù hợp trong timeout hoặc bị xác định unhealthy, traffic có thể được redirect tới resource khác. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-15-0-0/big-ip-local-traffic-manager-monitors-reference/monitors-concepts.html?utm_source=chatgpt.com)

Health monitor có thể kiểm tra nhiều level.

```text
ICMP
TCP
HTTP
HTTPS
application-specific
```

---

# 54. TCP monitor versus HTTP monitor

TCP monitor có thể chứng minh:

```text
Port accepts connection
```

nhưng không chứng minh:

```text
Application healthy
```

Ví dụ server:

```text
TCP/8080 accepts
```

nhưng application trả:

```text
HTTP 500
```

TCP monitor vẫn có thể coi service reachable trong khi HTTP-aware monitor có thể xác định unhealthy tùy configuration.

---

# 55. HTTP monitor

Một HTTP monitor có thể conceptually gửi:

```http
GET /health HTTP/1.1
Host: app.example.com
```

và kỳ vọng response phù hợp.

Nếu backend trả:

```http
HTTP/1.1 200 OK
```

và expected body match, monitor có thể xem member healthy.

Nhưng nếu:

```http
HTTP/1.1 503 Service Unavailable
```

hoặc body không match expected string, member có thể bị đánh dấu unhealthy tùy monitor config.

---

# 56. HTTPS monitor

HTTPS monitor thêm một layer nữa:

```text
TCP
 ↓
TLS handshake
 ↓
HTTP request
 ↓
HTTP response
```

Do đó failure có thể ở:

```text
TCP
TLS
certificate
SNI
HTTP
application
```

Đây là lý do syllabus bắt buộc bạn phân biệt transport/TLS/application health.

---

# 57. Health monitor source

Một backend có thể cho phép client traffic nhưng block health monitor source.

Ví dụ:

```text
Client data path source:
BIG-IP SNAT 10.10.20.100

Monitor source:
BIG-IP Self IP 10.10.20.10
```

Backend firewall:

```text
allow 10.10.20.100
deny 10.10.20.10
```

Kết quả:

```text
Application có thể technically work
Health Monitor = DOWN
BIG-IP không gửi production traffic tới member
```

Current BIG-IP documentation cũng lưu ý health monitor source behavior và các modern versions có thêm SNAT Pool options cho một số monitor types. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-21-1-0/big-ip-local-traffic-manager-monitors-reference/monitors-tasks.html?utm_source=chatgpt.com)

---

# 58. Monitor timeout / interval

Hai concepts quan trọng:

```text
Interval
Timeout
```

**Interval** thường liên quan frequency monitor probe.

**Timeout** liên quan thời gian/resource failure threshold.

Không nên tùy tiện đặt monitor quá aggressive.

Nếu monitor quá sensitive:

```text
temporary latency
→ member flapping
→ traffic redistribution
→ load spike
→ wider incident
```

Nếu quá lenient:

```text
failed backend
→ vẫn nhận traffic lâu
```

Monitor là operational control, không chỉ check box.

---

# 59. Load balancing

Load balancing quyết định member nào nhận new connection/request tùy algorithm và profile/configuration.

Các concepts quan trọng gồm:

| Method            | Mental model                                  |
| ----------------- | --------------------------------------------- |
| Round Robin       | Lần lượt qua available members                |
| Ratio             | Phân phối theo weight                         |
| Least Connections | Ưu tiên resource có ít active connections hơn |
| Weighted variants | Kết hợp active load với configured capacity   |

F5 documentation mô tả load-balancing method là algorithm dùng để chọn backend; Round Robin, Ratio và Least Connections là những examples tiêu biểu. [techdocs.f5.com](https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/ltm-basics-11-6-0/4.html?utm_source=chatgpt.com)

Exact available algorithms/features phải verify theo BIG-IP version.

---

# 60. Load balancing không đồng nghĩa requests luôn chia đều

Giả sử:

```text
Round Robin
2 members
100 requests
```

Bạn không nhất thiết luôn thấy chính xác:

```text
50 / 50
```

vì còn:

```text
connection reuse
persistence
member availability
OneConnect/application behavior
existing sessions
```

Đặc biệt nếu persistence tồn tại, traffic có thể cố ý quay về một backend.

---

# 61. Persistence

**Persistence** hay session affinity giúp giữ một client/session với same backend khi application yêu cầu.

Ví dụ shopping cart lưu local memory:

```text
Request 1 → Server A
Request 2 → Server B
```

có thể làm session mất.

Persistence tạo behavior:

```text
Client X
   ↓
Server A
   ↓
Server A
   ↓
Server A
```

F5 documentation giải thích rằng persistence profile có thể khiến returning client tiếp tục được gửi tới cùng pool member thay vì load-balancing algorithm chọn lại. [techdocs.f5.com](https://techdocs.f5.com/content/kb/en-us/products/big-ip_ltm/manuals/product/ltm-profiles-reference-12-1-0/_jcr_content/pdfAttach/download/file.res/BIG-IP_Local_Traffic_Management__Profiles_Reference.pdf?utm_source=chatgpt.com)

---

# 62. Persistence versus load balancing

Một điểm rất quan trọng:

```text
Load-balancing method
```

và:

```text
Persistence
```

không phải cùng một thứ.

Load balancing thường trả lời:

> Member nào nên nhận connection mới?

Persistence trả lời:

> Connection/request này có cần quay lại member trước không?

Persistence có thể override normal load-balancing choice trong appropriate context.

---

# 63. Persistence side effects

Persistence misconfiguration có thể tạo symptom:

```text
Một user luôn lỗi
User khác bình thường
```

Ví dụ user A persist vào:

```text
Member 1
```

Member 1 có application bug.

User B persist vào:

```text
Member 2
```

và không gặp lỗi.

Nếu engineer chỉ test từ máy mình:

```text
works for me
```

sẽ bỏ sót issue.

---

# 64. SNAT trên BIG-IP

BIG-IP có thể thay source IP trước khi gửi traffic tới backend.

Mental model:

```text
Client
192.0.2.100
   |
   v
BIG-IP
   |
SNAT
   |
   v
Backend sees
10.20.30.100
```

F5 Virtual Server API hiện có `source-address-translation` properties phục vụ SNAT behavior. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/APIRef_tm_ltm_virtual.html?utm_source=chatgpt.com)

---

# 65. Vì sao BIG-IP SNAT thường được dùng?

Một mục tiêu phổ biến:

```text
ensure symmetric return path
```

Không SNAT:

```text
Backend
→ default gateway
→ router
→ client
```

thay vì:

```text
Backend
→ BIG-IP
→ client
```

Nếu client-side TCP connection terminate/proxy qua BIG-IP, bypassing return traffic có thể phá connection.

---

# 66. SNAT và log visibility

Tradeoff:

Without SNAT:

```text
Backend sees real client IP
```

With SNAT:

```text
Backend sees BIG-IP address
```

Application có thể cần other mechanisms để preserve client identity ở higher layer, tùy design.

Operational lesson:

Khi correlate backend logs với firewall logs, phải biết SNAT design.

---

# 67. TCP profiles

F5 **TCP profile** là configuration object kiểm soát TCP behavior.

BIG-IP documentation mô tả TCP profiles như tools dùng để quản lý TCP network traffic; các profiles được associated với Virtual Server và là dependency của nhiều traffic-management functions khác. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-21-1-0/big-ip-local-traffic-management-profiles-reference/protocol-profiles.html?utm_source=chatgpt.com)

Day 9 không yêu cầu tuning chuyên sâu.

Bạn cần hiểu:

```text
TCP behavior is not necessarily transparent pass-through.
```

BIG-IP có thể terminate/manage TCP sessions tùy Virtual Server/profile design.

---

# 68. Two TCP connections

Một load balancer/proxy architecture có thể tạo hai logical TCP connections:

```text
Client-side TCP:
Client ↔ BIG-IP
```

và:

```text
Server-side TCP:
BIG-IP ↔ Backend
```

Do đó:

```text
Client → BIG-IP TCP succeeds
```

không chứng minh:

```text
BIG-IP → Backend TCP succeeds
```

Đây là một insight cực kỳ quan trọng.

---

# 69. HTTP profile

HTTP profile làm BIG-IP aware ở Layer 7 HTTP.

F5 documentation mô tả HTTP profiles như groups of settings điều khiển cách BIG-IP xử lý HTTP traffic; profiles có thể ảnh hưởng đến headers và application-layer processing. [techdocs.f5.com](https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/ltm-profiles-reference-13-0-0/2.html?utm_source=chatgpt.com)

Nếu không có appropriate HTTP processing context, BIG-IP có thể chỉ xử lý traffic ở lower layer.

---

# 70. Layer 4 versus Layer 7 thinking

Layer 4:

```text
IP
TCP
port
```

Layer 7:

```text
HTTP method
Host header
URI
cookies
headers
response code
```

Ví dụ:

```text
TCP/443 = OPEN
```

nhưng:

```http
GET /api
Host: wrong.example.com

HTTP/1.1 404 Not Found
```

Network works.

Application routing does not.

---

# 71. SSL/TLS foundation

Đây là phần cực kỳ quan trọng.

TLS cung cấp các properties như:

```text
encryption
integrity
server authentication
optionally client authentication
```

High-level handshake:

```text
ClientHello
   ↓
ServerHello
Certificate
   ↓
Certificate validation
Key agreement
   ↓
Encrypted application traffic
```

Không cần học crypto mathematics trong Day 9, nhưng phải phân biệt các failure stages.

---

# 72. Client-side SSL

**Client-side SSL**:

```text
Client
   ⇄ TLS
BIG-IP
```

BIG-IP đóng vai TLS endpoint phía client.

F5 documentation hiện mô tả **Client SSL profile** là profile cho phép BIG-IP terminate/decrypt client-side SSL traffic và encrypt response trở lại client. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-17-0-0/big-ip-system-ssl-administration/ssl-traffic-management.html?utm_source=chatgpt.com)

---

# 73. Server-side SSL

**Server-side SSL**:

```text
BIG-IP
   ⇄ TLS
Backend
```

Nếu Server SSL profile được áp dụng, BIG-IP có thể re-encrypt request tới backend.

F5 hiện mô tả Server SSL profile là cơ chế cho phép BIG-IP thực hiện SSL processing ở server side. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-17-0-0/big-ip-system-ssl-administration/ssl-traffic-management.html?utm_source=chatgpt.com)

---

# 74. TLS termination

Scenario:

```text
Client
   |
 HTTPS
   |
BIG-IP
   |
 HTTP
   |
Backend
```

BIG-IP terminates TLS.

Backend nhận clear-text HTTP trên internal network.

Ưu điểm:

```text
certificate centralized
backend CPU offload
Layer 7 visibility
```

Risk/design consideration:

```text
traffic BIG-IP → backend không encrypted
```

Nếu security requirement bắt buộc encryption end-to-end, design khác cần được dùng.

---

# 75. TLS re-encryption / bridging

Scenario:

```text
Client
   |
 HTTPS
   |
BIG-IP
   |
 HTTPS
   |
Backend
```

BIG-IP xử lý:

```text
client-side TLS
```

và một independent:

```text
server-side TLS
```

Do đó có thể tồn tại hai independent certificate validation domains.

---

# 76. Một TLS problem có thể ở hai phía

Symptom:

```text
HTTPS application unavailable
```

Possible cases:

```text
Client → BIG-IP TLS broken
```

hoặc:

```text
BIG-IP → Backend TLS broken
```

Hai failure hoàn toàn khác nhau.

Client-side example:

```text
public certificate expired
```

Server-side example:

```text
backend certificate not trusted
```

---

# 77. Certificate

Digital certificate thường binding:

```text
identity
+
public key
```

Certificate chứa các metadata như:

```text
Subject
Issuer
Validity
Public key
SAN
Signature
```

Trong HTTPS, hostname validation ngày nay chủ yếu dựa vào **Subject Alternative Name — SAN**.

---

# 78. Private key

Certificate public key và corresponding private key phải pair đúng.

Concept:

```text
Certificate:
contains public key

Private key:
secret
```

Private key phải được bảo vệ nghiêm ngặt.

Evidence package **không bao giờ** nên chứa private key.

---

# 79. Certificate chain

Simplified chain:

```text
Server Certificate
      |
      v
Intermediate CA
      |
      v
Root CA
```

Client cần xây trust path tới trusted CA.

Nếu server gửi incomplete chain:

```text
Leaf
 |
missing Intermediate
```

một số clients có thể fail.

Do đó certificate tồn tại và chưa expire vẫn chưa đủ.

---

# 80. Certificate trust

Certificate validation phải hỏi:

```text
Certificate signed by trusted issuer?
Chain complete?
```

Self-signed certificate có thể technically encrypt traffic nhưng client chưa trust nó.

Encryption ≠ trusted identity.

---

# 81. Certificate hostname validation

Giả sử certificate SAN:

```text
app.example.com
```

client connect:

```text
https://api.example.com
```

certificate có thể:

```text
valid dates
trusted CA
good chain
```

nhưng vẫn fail:

```text
hostname mismatch
```

Vì vậy syllabus yêu cầu check **trust + chain + hostname + expiry**. Infrastructure_Operations_14_Da…

---

# 82. Certificate expiry

Certificate có:

```text
Not Before
Not After
```

Nếu current time > `Not After`:

```text
expired
```

Nếu clock sai, một certificate valid có thể bị hiểu sai.

Do đó TLS troubleshooting đôi khi liên quan tới:

```text
NTP/time synchronization
```

chứ không chỉ certificate file.

---

# 83. Certificate renewal

Renewal workflow enterprise thường gồm:

```text
inventory
expiry monitoring
CSR/key decision
CA issuance
certificate import
chain import
profile update
change approval
verification
rollback
```

Đừng đợi tới ngày expiry mới xử lý.

Certificate renewal là planned operational change.

---

# 84. Certificate/key mismatch

Nếu certificate và private key không match, SSL profile có thể không hoạt động đúng.

Không được “fix” bằng cách copy private keys lung tung giữa systems.

Private keys phải follow approved security handling.

---

# 85. SNI

**Server Name Indication — SNI** cho phép client gửi hostname mong muốn trong TLS ClientHello.

Điều này rất quan trọng khi nhiều hostnames share cùng IP.

Ví dụ:

```text
203.0.113.50
```

phục vụ:

```text
app1.example.com
app2.example.com
```

SNI cho phép BIG-IP chọn certificate/profile phù hợp trước khi HTTP Host header được đọc.

F5 hiện mô tả SNI chính xác theo mechanism này: client gửi domain name trong TLS negotiation để BIG-IP chọn appropriate certificate/profile; có thể cấu hình default SNI profile khi no match/no SNI. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-17-5-1/big-ip-system-ssl-administration/additional-ssl-profile-configuration-options.html?utm_source=chatgpt.com)

---

# 86. SNI failure symptom

Giả sử client gửi:

```text
SNI = app2.example.com
```

nhưng BIG-IP chỉ có profile cho:

```text
app1.example.com
```

có thể xuất hiện:

```text
wrong certificate
TLS failure
hostname mismatch
unexpected default certificate
```

Do đó test bằng IP trực tiếp:

```bash
curl https://203.0.113.50
```

không nhất thiết tương đương test bằng hostname:

```bash
curl https://app2.example.com
```

Đây là common mistake.

---

# 87. TLS test phải preserve hostname/SNI

Nếu DNS chưa đúng nhưng muốn test đúng hostname, có thể trong lab dùng mechanism như:

```bash
curl --resolve app.example.com:443:203.0.113.50 \
     https://app.example.com/
```

`--resolve` giúp request vẫn sử dụng:

```text
hostname = app.example.com
SNI = app.example.com
```

nhưng connect tới IP chỉ định.

Đây thường tốt hơn:

```bash
curl https://203.0.113.50
```

khi bạn cần kiểm tra SNI/hostname behavior.

---

# 88. Không dùng `-k` để biến test fail thành pass

Một anti-pattern:

```bash
curl -k https://app.example.com
```

`-k` / `--insecure` bypass certificate verification.

Nó có thể hữu ích trong một số controlled diagnostic experiments, nhưng **không được coi kết quả đó là successful TLS validation**.

Syllabus nêu rõ không bypass TLS validation chỉ để đạt trạng thái pass. Infrastructure_Operations_14_Da…

Proper test:

```bash
curl --cacert /path/to/trusted-ca.pem \
     https://app.example.com/
```

hoặc dùng trusted system CA store.

---

# 89. Transport failure versus TLS failure versus HTTP failure

Hãy nhớ bảng này.

| Layer       | Example symptom                     |
| ----------- | ----------------------------------- |
| DNS         | name not resolved                   |
| TCP         | timeout / connection refused        |
| TLS         | handshake error / certificate error |
| HTTP        | 404 / 500 / 503                     |
| Application | HTTP 200 nhưng response data sai    |

Nếu nhận:

```http
HTTP/1.1 503 Service Unavailable
```

thì ít nhất:

```text
DNS probably resolved
TCP probably established
TLS probably completed nếu HTTPS
HTTP exchange occurred
```

Do đó đừng troubleshoot certificate đầu tiên khi đã có valid HTTP response.

---

# 90. `connection refused`

`connection refused` thường nghĩa:

```text
IP reachable enough to return TCP reset
but no listener/service accepting connection
```

Có thể do:

```text
Virtual Server disabled
port wrong
backend service not listening
firewall/device actively rejecting
```

Không giống:

```text
timeout
```

---

# 91. Timeout

Timeout có thể do:

```text
packet drop
route problem
firewall silent deny
backend nonresponse
health monitor path
network congestion
```

Timeout không tự nó cho biết root cause.

Bạn phải correlate across layers.

---

# 92. HTTP 503

`503 Service Unavailable` có thể xuất hiện khi application/load balancer không có usable backend, nhưng exact reason phải kiểm tra.

Trong BIG-IP context, hypotheses có thể bao gồm:

```text
all Pool Members unhealthy
application returning 503
traffic policy/iRule behavior
upstream dependency failure
```

Không được thấy `503` rồi tự động kết luận “F5 lỗi”.

---

# 93. Profiles trên BIG-IP

F5 profiles là reusable configuration objects điều khiển traffic behavior.

Ví dụ:

```text
TCP profile
HTTP profile
Client SSL profile
Server SSL profile
Persistence profile
```

Profiles được attached tới Virtual Server.

F5 docs xác nhận một Virtual Server có thể associate multiple profiles như TCP, SSL và HTTP. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-15-1-0/big-ip-local-traffic-management-profiles-reference/introduction-to-local-traffic-profiles.html?utm_source=chatgpt.com)

---

# 94. BIG-IP `tmsh`

`tmsh` = **Traffic Management Shell**.

Đây là CLI chính để quản lý/inspect BIG-IP configuration và state.

Một distinction quan trọng:

```text
list
```

thường dùng để xem configuration.

```text
show
```

thường dùng để xem status/statistics.

F5 `tmsh` documentation mô tả `show` là command hiển thị status/statistics cho components. [techdocs.f5.com](https://techdocs.f5.com/content/kb/en-us/products/big-ip_ltm/manuals/product/bigip-tmsh-11-5-1/_jcr_content/pdfAttach/download/file.res/bigip-tmsh-11-5-1.pdf?utm_source=chatgpt.com)

---

# 95. `tmsh list`

Ví dụ:

```bash
tmsh list ltm virtual
```

có thể hiển thị Virtual Server configuration.

Ví dụ conceptual:

```text
ltm virtual app_https {
    destination 203.0.113.50:https
    ip-protocol tcp
    pool app_pool
    profiles {
        clientssl { }
        http { }
        tcp { }
    }
}
```

F5 documentation có examples sử dụng `tmsh list ltm virtual` để inspect Virtual Server config. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-15-0-1/big-ip-system-configuring-geo-redundancy/configuring-geo-redundancy-on-a-big-ip-system.html?utm_source=chatgpt.com)

---

# 96. `tmsh show`

Status:

```bash
tmsh show ltm pool
```

có thể dùng để xem pool statistics/status.

Detailed:

```bash
tmsh show ltm pool detail
```

F5 `tmsh` reference có ví dụ `show / ltm pool` và detailed pool statistics. [techdocs.f5.com](https://techdocs.f5.com/content/kb/en-us/products/big-ip_ltm/manuals/product/bigip-tmsh-11-5-1/_jcr_content/pdfAttach/download/file.res/bigip-tmsh-11-5-1.pdf?utm_source=chatgpt.com)

---

# 97. Configuration versus state trên BIG-IP

Giống FortiGate:

```text
list = configured state
show = operational state
```

Conceptually.

Ví dụ:

```text
Pool configuration contains member:
10.20.30.11:8443
```

không chứng minh:

```text
member currently UP
```

Bạn phải xem operational status.

---

# 98. iControl REST API

BIG-IP cung cấp **iControl REST API**.

F5 mô tả iControl REST là REST-based interface dùng để programmatically interact với BIG-IP configuration/services. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/?utm_source=chatgpt.com)

API giúp:

```text
monitoring
automation
inventory
configuration management
validation
reporting
```

Day 9 tập trung chủ yếu vào **authenticated read-only requests**.

---

# 99. REST concept

REST thường sử dụng:

```text
resource
URI
HTTP method
request headers
request body
response status
response body
```

Example resource:

```text
/mgmt/tm/ltm/virtual
```

F5 API documents endpoint này là Virtual Server configuration collection. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/APIRef_tm_ltm_virtual.html?utm_source=chatgpt.com)

---

# 100. HTTP methods

| Method  | Typical semantic              |
| ------- | ----------------------------- |
| GET     | Retrieve/read                 |
| POST    | Create/action tùy API         |
| PUT     | Replace/update resource       |
| PATCH   | Partial update                |
| DELETE  | Delete resource               |
| OPTIONS | Discover supported operations |

Exact behavior phải đọc endpoint documentation.

Điểm vận hành quan trọng:

```text
GET
```

thường là lựa chọn Day 9 cho read-only validation.

Trong F5 Virtual Server endpoint, collection supports read operations như GET, trong khi resource endpoint còn hỗ trợ mutating methods như PUT/PATCH/DELETE/POST. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/APIRef_tm_ltm_virtual.html?utm_source=chatgpt.com)

---

# 101. Read-only versus mutating APIs

Điều này phải được phân biệt rõ:

```http
GET /mgmt/tm/ltm/virtual
```

read-only.

Trong khi một request như:

```http
PATCH ...
```

có thể change production configuration.

Do đó API client có thể dangerous không kém CLI.

Least privilege API account cực kỳ quan trọng.

---

# 102. API authentication

iControl REST yêu cầu authentication.

F5 documentation hiện nêu support cho:

```text
HTTP Basic Authentication
Token-based authentication
```

với caveats liên quan local versus external authentication. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-soap/Authentication_with_the_F5_REST_API.html?utm_source=chatgpt.com)

Token-based pattern có thể dùng header:

```http
X-F5-Auth-Token: <token>
```

Credentials/token phải được treat như secrets.

---

# 103. Không log secrets

Một cURL command kiểu:

```bash
curl -H "X-F5-Auth-Token: abcdef123456..."
```

nếu paste vào:

```text
ticket
chat
terminal transcript
evidence file
```

sẽ leak secret.

Better operational practice:

```bash
export F5_TOKEN='<secret>'
```

rồi:

```bash
-H "X-F5-Auth-Token: ${F5_TOKEN}"
```

Nhưng ngay cả environment variable cũng là credential handling issue, nên phải follow enterprise-approved mechanism.

Evidence chỉ nên ghi:

```text
Authentication: token-based
Result: authenticated
Token: REDACTED
```

---

# 104. Basic authentication risk

Example:

```bash
curl -u admin:Password123 ...
```

có thể expose password trong:

```text
shell history
process list
transcript
screenshare
ticket
```

Không dùng real credentials trong lab notes.

Trong course examples luôn dùng placeholder:

```text
<USERNAME>
<PASSWORD>
<TOKEN>
```

---

# 105. JSON

API response thường ở dạng JSON.

Example simplified:

```json
{
  "name": "app_https",
  "destination": "/Common/203.0.113.50:443",
  "pool": "/Common/app_pool",
  "enabled": true
}
```

JSON có:

```text
objects
arrays
keys
values
strings
numbers
booleans
null
```

Bạn cần đọc được cấu trúc để không chỉ nói:

> “API trả response.”

Mà phải nói:

> “API trả HTTP 200, JSON parse thành công và field `pool` bằng expected value.”

---

# 106. Assertion

Automation check tốt phải có **assertion**.

Bad check:

```text
curl command completed.
PASS.
```

Good check:

```text
HTTP status expected = 200
HTTP status actual   = 200

JSON parse expected  = valid
JSON parse actual    = valid

Pool expected        = /Common/app_pool
Pool actual          = /Common/app_pool

Assertion            = PASS
```

Nếu:

```text
actual != expected
```

script phải tạo observable failure.

Syllabus nêu rõ failed assertion phải tạo non-success result. Infrastructure_Operations_14_Da…

---

# 107. cURL

`curl` là command-line HTTP client.

Basic structure:

```bash
curl [options] URL
```

Read-only BIG-IP pattern:

```bash
curl \
  --cacert /path/to/trusted-ca.pem \
  --connect-timeout 5 \
  --max-time 10 \
  -sS \
  -H "X-F5-Auth-Token: ${F5_TOKEN}" \
  "https://<BIGIP-MGMT>/mgmt/tm/ltm/virtual"
```

Mục đích:

```text
--cacert
→ verify server certificate against approved CA

--connect-timeout
→ bound connection phase

--max-time
→ bound total execution

-sS
→ reduce noise nhưng vẫn show errors

-H
→ authentication header
```

Không dùng:

```bash
-k
```

để giả vờ TLS valid.

---

# 108. cURL exit status versus HTTP status

Đây là điểm rất quan trọng.

Có hai lớp status.

### Tool-level

```text
curl exit code
```

Ví dụ failure có thể là:

```text
DNS resolution
TCP timeout
TLS verification
```

### HTTP-level

Server có thể trả:

```text
200
401
403
404
500
503
```

`curl` có thể technically communicate thành công nhưng HTTP operation lại fail.

Do đó phải capture cả:

```text
tool exit code
HTTP status
```

Syllabus yêu cầu chính xác điều này. Infrastructure_Operations_14_Da…

---

# 109. HTTP response codes

Operational classification:

| Code family | Ý nghĩa khái quát                 |
| ----------- | --------------------------------- |
| 2xx         | Request successful                |
| 3xx         | Redirect                          |
| 4xx         | Client/request/auth-related error |
| 5xx         | Server-side/API service error     |

Day 9 đặc biệt chú ý:

```text
401
403
timeout
5xx
malformed response
```

---

# 110. 401 Unauthorized

`401` thường liên quan authentication.

Hypotheses:

```text
credential invalid
token expired
auth method incorrect
missing auth header
```

Không nên respond bằng:

> “Use admin account.”

Proper approach:

```text
check credential workflow
check token validity
check identity source
do not increase privilege until root cause established
```

---

# 111. 403 Forbidden

`403` thường có implication:

```text
identity known
but access not permitted
```

Hypotheses:

```text
role insufficient
resource permission denied
authorization policy
partition/scope restrictions
```

401 và 403 không phải cùng một problem.

---

# 112. 404

Could mean:

```text
wrong API path
resource not found
wrong partition/path
version endpoint difference
```

Không nên ngay lập tức conclude service down.

---

# 113. 5xx

5xx thường chỉ server/API processing problem.

Có thể cần:

```text
BIG-IP service logs
API daemon status
request validity
device resource health
```

Syllabus chỉ yêu cầu awareness/troubleshooting, không yêu cầu sửa internals không có approval.

---

# 114. Timeout

Automation không được request vô hạn.

Syllabus yêu cầu **bounded timeouts**.

Ví dụ:

```bash
--connect-timeout 5
--max-time 10
```

Values thực tế phải follow runbook/SLA/environment.

Timeout bảo vệ:

```text
scheduled jobs
monitoring loops
CI pipelines
operator terminal
```

khỏi hanging indefinitely.

---

# 115. Retry

Retry có thể hữu ích cho transient failures.

Nhưng:

```text
retry GET
```

và:

```text
retry DELETE/PATCH/POST
```

có risk rất khác nhau.

Read-only GET thường dễ retry safely hơn.

Mutating request có thể gây duplicate/change ambiguity.

Syllabus nói rõ:

> **do not blindly retry changes.** Infrastructure_Operations_14_Da…

---

# 116. Idempotency awareness

**Idempotent** theo high-level API semantics nghĩa là lặp operation không tạo thêm cumulative side effects ngoài intended final state.

Tuy nhiên actual API behavior phải được xác minh.

Không nên suy luận chỉ từ method name.

Trong automation operations:

```text
read
validate
dry-run nếu có
change once
verify
```

an toàn hơn blind retries.

---

# 117. API malformed response

Giả sử expected:

```json
{
  "state": "up"
}
```

nhưng nhận:

```text
<html>Error</html>
```

HTTP connection có thể thành công nhưng parser fail.

Correct result:

```text
FAIL
```

không phải:

```text
PASS because response received
```

---

# 118. BIG-IP logs

Ở mức awareness, hai log locations thường useful gồm:

```text
/var/log/ltm
/var/log/restjavad.0.log
```

F5 training documentation hiện ghi `/var/log/ltm` cho LTM events và `/var/log/restjavad.0.log` cho REST API-related events trong relevant deployments. [F5 Cloud Docs](https://clouddocs.f5.com/training/community/automation/html/class01/module1/module1.html?utm_source=chatgpt.com)

Exact logging behavior/rotation/access phải verify theo version/environment.

---

# 119. Load-balancer troubleshooting methodology

| Layer          | Câu hỏi                                           |
| -------------- | ------------------------------------------------- |
| Virtual Server | Exists? Enabled? Correct IP/port/protocol?        |
| Profiles       | Correct TCP/HTTP/SSL profiles attached?           |
| Pool           | Correct default pool?                             |
| Members        | Correct IP/port?                                  |
| Monitor        | Correct health monitor?                           |
| Availability   | Members healthy?                                  |
| Routing        | BIG-IP có route tới backend không?                |
| SNAT           | Return path correct không?                        |
| TLS            | Client-side/server-side handshake đúng không?     |
| HTTP           | Status code/body đúng không?                      |
| Persistence    | Client có bị pinned vào problematic member không? |
| Logs           | LTM/API/TLS logs có evidence gì?                  |

---

# 120. Integrated FortiGate + BIG-IP flow

Bây giờ nối hai products.

Scenario:

```text
Client:
10.10.10.50

Application hostname:
app.example.com

DNS:
app.example.com → 203.0.113.50
```

Path:

```text
Client
10.10.10.50
    |
    | TCP 443
    v
FortiGate
    |
    | Policy 101
    | Allow HTTPS
    | maybe NAT
    v
BIG-IP
Virtual Server
203.0.113.50:443
    |
    | Client SSL
    | HTTP
    | Load balancing
    v
APP_POOL
    |
    +---- 10.20.30.11:8443
    |
    +---- 10.20.30.12:8443
```

---

# 121. Failure scenario A — Firewall policy

Client:

```text
curl: connection timed out
```

FortiGate log:

```text
Denied
policy=implicit
```

BIG-IP:

```text
No connection increment
```

Reasoning:

```text
Traffic never reaches BIG-IP.
```

Do not start debugging pool monitor.

Root domain:

```text
Firewall policy/path.
```

---

# 122. Failure scenario B — Route

FortiGate:

```text
Packet enters port2
```

No correct route to BIG-IP network.

Debug:

```text
route lookup fails / wrong egress
```

Policy may exist perfectly.

Root cause:

```text
routing
```

not:

```text
policy.
```

---

# 123. Failure scenario C — BIG-IP member down

Firewall:

```text
allow
session established
```

BIG-IP:

```text
Virtual Server receives traffic
Pool members all DOWN
```

Client gets:

```text
503 or connection failure
```

depending configuration.

Root domain:

```text
load balancer/backend health.
```

---

# 124. Failure scenario D — Client-side certificate expired

Firewall:

```text
Allow
```

BIG-IP listener reachable:

```text
TCP handshake OK
```

TLS:

```text
certificate expired
```

Backend health:

```text
UP
```

Root domain:

```text
client-side TLS certificate
```

Not backend application.

---

# 125. Failure scenario E — Server-side TLS

Client certificate:

```text
valid
```

Client→BIG-IP TLS:

```text
success
```

Pool backend:

```text
TCP reachable
```

BIG-IP→backend TLS:

```text
certificate trust/handshake failure
```

Root domain:

```text
Server SSL/backend TLS.
```

---

# 126. Failure scenario F — SNI

Client A:

```text
https://app1.example.com
```

works.

Client B:

```text
https://app2.example.com
```

fails certificate validation.

Both resolve:

```text
203.0.113.50
```

Likely hypothesis:

```text
SNI / SSL profile / certificate mapping.
```

Not IP reachability.

---

# 127. Failure scenario G — Health endpoint

Backend:

```text
https://10.20.30.11:8443/
```

works.

Monitor:

```text
GET /health
```

returns:

```text
500
```

Pool Member:

```text
DOWN
```

Engineer manually browsing `/` và thấy 200 có thể kết luận sai.

Health monitor tests **configured health condition**, không nhất thiết root page.

---

# 128. Failure scenario H — Wrong Host header

Backend virtual hosting requires:

```http
Host: app.example.com
```

Health monitor sends incorrect request.

Backend returns:

```http
404
```

Member is marked down.

Network and TLS may both be healthy.

Root cause:

```text
application-layer monitor configuration.
```

---

# 129. Failure scenario I — SNAT return path

BIG-IP sends:

```text
Client source preserved
```

Backend default route bypasses BIG-IP.

Result:

```text
SYN reaches backend
SYN/ACK bypasses BIG-IP
client connection fails
```

Changing health monitor will not fix it.

Investigate:

```text
routing/SNAT.
```

---

# 130. Failure scenario J — Persistence

Pool:

```text
Member A healthy-ish but application bug
Member B healthy
```

Client X pinned:

```text
A
```

Client Y:

```text
B
```

Only X reports issue.

Need inspect persistence/session, not assume random network failure.

---

# 131. Troubleshooting theo least-risk order

Đây là operational philosophy của Day 9:

```text
Read-only evidence
        ↓
Low-impact tests
        ↓
Bounded diagnostics
        ↓
Controlled reproduction
        ↓
Approved change
        ↓
Verify
        ↓
Rollback if needed
```

Không đảo thành:

```text
restart
disable security
clear sessions
change routes
```

rồi mới collect evidence.

---

# 132. Evidence model

Một strong Day 9 evidence set phải trả lời được:

| Evidence  | Cần ghi                                   |
| --------- | ----------------------------------------- |
| Identity  | hostname/device/product/version           |
| Time      | timestamp + timezone                      |
| Flow      | source/destination/protocol/port          |
| Firewall  | route, policy, NAT, session               |
| BIG-IP    | Virtual Server, pool, member, monitor     |
| TLS       | hostname, issuer, expiry, chain result    |
| API       | endpoint, HTTP status, cURL exit status   |
| Change    | ticket/change ID                          |
| Result    | expected versus actual                    |
| Cleanup   | debug stopped, temporary changes reverted |
| Integrity | file/hash nếu export thành artifact       |

Day 1 evidence contract của syllabus còn yêu cầu source owner, asset ID, query/window, collector, ticket reference, size/count/SHA-256 và separation giữa raw evidence với derived/redacted copies. Infrastructure_Operations_14_Da…

---

# 133. Không thu secrets làm evidence

Không lưu:

```text
password
API token
Authorization header
private key
MFA seed
session secret
```

Nếu transcript chứa:

```text
Authorization: Bearer ...
```

evidence đã trở thành security incident candidate.

Sanitize trước khi submit.

---

# 134. Expected versus actual

Mọi troubleshoot nên có format reasoning:

```text
Expected:
TCP/443 should be allowed.

Actual:
FortiGate log shows policy 105 deny.

Evidence:
timestamp ...
source ...
destination ...
policy ...
action ...

Conclusion:
Traffic blocked at firewall policy layer.
```

Không viết:

> “Firewall lỗi.”

Đó không phải technical evidence.

---

# 135. Common mistakes — FortiGate

| Mistake                             | Vì sao sai                                              |
| ----------------------------------- | ------------------------------------------------------- |
| “Ping được nên firewall OK”         | ICMP không chứng minh TCP/443                           |
| “Policy exists nên traffic allowed” | Có thể sai order/object/service                         |
| “No deny log nên allowed”           | Logging có thể disabled hoặc traffic không tới device   |
| “Route exists”                      | Có thể route khác thắng                                 |
| “Session table empty = no traffic”  | Filter/context/offload/tool limitation có thể ảnh hưởng |
| “NAT enabled = NAT correct”         | Có thể wrong IP pool/mode/central NAT                   |
| Clear sessions ngay                 | Disruptive và mất runtime evidence                      |
| Disable security profile để thử     | Có thể tạo security exposure                            |
| Run unlimited debug                 | Tạo noisy output/operational risk                       |

---

# 136. Common mistakes — BIG-IP

| Mistake                                  | Vì sao sai                                |
| ---------------------------------------- | ----------------------------------------- |
| “Virtual Server exists nên service OK”   | Pool/member/TLS vẫn có thể fail           |
| “Pool Member pingable nên healthy”       | Service port/application có thể fail      |
| “TCP monitor UP nên application healthy” | Layer 7 chưa được kiểm tra                |
| “Certificate not expired nên valid”      | Trust/chain/hostname cũng phải đúng       |
| `curl -k` pass = TLS healthy             | Verification đã bị bypass                 |
| HTTP 503 = network failure               | TCP/TLS/HTTP đã có thể hoạt động          |
| 401 và 403 giống nhau                    | Authentication khác authorization         |
| Retry every API call                     | Mutating requests có thể có side effects  |
| API response received = success          | Status/JSON/assertion có thể fail         |
| Backend sees SNAT IP = attack            | Có thể là expected load-balancer behavior |

---

# 137. Security implications và least privilege

FortiGate và BIG-IP đều nằm ở critical control points.

Một unauthorized firewall change có thể:

```text
expose internal network
block production
bypass security segmentation
```

Một unauthorized load-balancer change có thể:

```text
take application offline
route users to wrong backend
break TLS
expose private keys
```

Do đó Day 9 phải luôn giữ nguyên tắc:

```text
read-only first
least privilege
approved change
pre-check
backup
minimal change
post-check
backout
evidence
```

Syllabus yêu cầu real configuration changes phải tuân existing approved change/backout workflow. Infrastructure_Operations_14_Da…

---

# 138. L1/L2 escalation boundary

Một L1/L2 engineer thường có thể an toàn thực hiện trong authorized scope:

| Task                              | Typical boundary                         |
| --------------------------------- | ---------------------------------------- |
| Verify interface/status           | L1/L2                                    |
| Verify route                      | L1/L2                                    |
| Inspect firewall policy           | L1/L2                                    |
| Check sessions                    | L1/L2                                    |
| Review logs                       | L1/L2                                    |
| Verify Virtual Server/pool/member | L1/L2                                    |
| Validate certificate              | L1/L2                                    |
| Run read-only API GET             | L1/L2 nếu authorized                     |
| Bounded packet capture/debug      | L2 hoặc controlled runbook               |
| Modify firewall policy            | Change approval / network-security owner |
| Change NAT                        | Change approval                          |
| Replace production certificate    | Approved change                          |
| Modify SSL profile                | BIG-IP/application owner/change          |
| Clear broad session table         | Escalate/approval                        |
| Force HA failover                 | High-risk/change                         |
| Disable security inspection       | Security/change approval                 |
| Change routing in production      | Network owner/change                     |

Exact organizational boundary luôn do enterprise RBAC/runbook quyết định.

---

# 139. Khi nào phải escalate?

Escalation cần thiết khi:

```text
required action exceeds your RBAC
root cause lies outside assigned system
production change required
HA state abnormal
configuration drift unexplained
certificate/private-key issue requires security owner
traffic loss remains unexplained after read-only diagnostics
security event indicates compromise
vendor defect suspected
required rollback is disruptive
evidence conflicts across systems
```

Escalation tốt không phải:

> “Không biết, chuyển team khác.”

Escalation tốt là:

```text
Impact:
All users cannot access APP-PROD.

Start time:
14:32 ICT.

Expected:
Client 10.10.10.50 should reach VIP 203.0.113.50:443.

Verified:
DNS correct.
FortiGate policy 101 allows flow.
FortiGate session present.
BIG-IP VS app_https receives connections.
Both pool members marked DOWN.
Monitor is HTTPS /health.

Backend manual test:
/health returns HTTP 500.

Suspected layer:
Application health endpoint.

No configuration changed.

Evidence:
EV-0901 ... EV-0906.
```

Đó là escalation có giá trị.

---

# 140. Read-only command cheat sheet của phần Lý thuyết

## FortiGate

| Mục tiêu            | Command                               |
| ------------------- | ------------------------------------- |
| Version/system      | `get system status`                   |
| Interface config    | `show system interface`               |
| Specific interface  | `show system interface <interface>`   |
| Routing table       | `get router info routing-table all`   |
| Sessions            | `diagnose sys session list`           |
| Session filter      | `diagnose sys session filter ...`     |
| Reset debug state   | `diagnose debug reset`                |
| Set flow filter     | `diagnose debug flow filter ...`      |
| Enable debug        | `diagnose debug enable`               |
| Start bounded trace | `diagnose debug flow trace start <N>` |
| Stop trace          | `diagnose debug flow trace stop`      |
| Disable debug       | `diagnose debug disable`              |

Current Fortinet documentation confirms the routing, session and debug-flow command families above. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/cli-reference/235530229/diagnose-sys?utm_source=chatgpt.com)

## BIG-IP

| Mục tiêu              | Example                     |
| --------------------- | --------------------------- |
| Virtual Server config | `tmsh list ltm virtual`     |
| Pool config           | `tmsh list ltm pool`        |
| Pool runtime status   | `tmsh show ltm pool`        |
| Detailed pool state   | `tmsh show ltm pool detail` |
| Virtual Server state  | `tmsh show ltm virtual`     |

`tmsh list` và `show` command families là các fundamental tools để inspect LTM configuration/state. [techdocs.f5.com](https://techdocs.f5.com/content/kb/en-us/products/big-ip_ltm/manuals/product/bigip-tmsh-11-5-1/_jcr_content/pdfAttach/download/file.res/bigip-tmsh-11-5-1.pdf?utm_source=chatgpt.com)

---

# 141. BIG-IP read-only REST mental model

Conceptual request:

```http
GET /mgmt/tm/ltm/virtual
```

Flow:

```text
cURL
  |
TLS verification
  |
Authentication
  |
HTTP GET
  |
iControl REST
  |
JSON response
  |
Parse
  |
Assertion
  |
Exit success/failure
```

Mỗi stage đều có thể fail.

---

# 142. API success criteria

Không dùng:

```text
“curl ran”
```

làm success criterion.

Một robust validation có dạng:

```text
DNS/connection succeeded
AND
TLS verification succeeded
AND
authentication succeeded
AND
HTTP status expected
AND
JSON valid
AND
required field exists
AND
field value matches expected state
```

Chỉ khi tất cả required conditions pass mới gọi check đó là success.

---

# 143. Một automation result mẫu

```text
Check ID: BIGIP-VS-001

Target:
bigip-lab.example.com

Endpoint:
/mgmt/tm/ltm/virtual/~Common~app_https

Expected:
HTTP 200
name = app_https
enabled = true
pool = /Common/app_pool

Actual:
HTTP 200
name = app_https
enabled = true
pool = /Common/app_pool

TLS validation:
PASS

JSON parsing:
PASS

Assertion:
PASS

cURL exit:
0
```

Nếu pool trả:

```text
/Common/old_pool
```

thì:

```text
Assertion:
FAIL
```

dù HTTP status vẫn 200.

---

# 144. Negative result mẫu

```text
Expected:
HTTP 200

Actual:
HTTP 403

TLS:
PASS

Authentication:
Identity recognized

Authorization:
Denied

Assertion:
FAIL

Interpretation:
API connectivity and TLS are functional.
Investigate RBAC/authorization scope.
Do not substitute a more privileged account without approval.
```

Đây là đúng operational reasoning.

---

# 145. Data flow versus control evidence

Khi troubleshoot Day 9, evidence nên lấy từ nhiều vantage points:

```text
Client
FortiGate
BIG-IP
Backend
```

Ví dụ:

```text
Client:
SYN sent

FortiGate:
session exists / allowed

BIG-IP:
connection count increments

Backend:
no SYN received
```

Bây giờ failure domain thu hẹp tới:

```text
BIG-IP → backend path
```

Thay vì:

> “Website lỗi.”

---

# 146. Correlation bằng timestamp

Nếu systems có time zones khác:

```text
FortiGate: UTC
BIG-IP: local
Backend: UTC+7
```

bạn có thể correlate sai event.

Evidence phải lưu:

```text
timestamp
timezone
```

và ideally NTP/time sync status.

Đây là lý do timestamp normalization sẽ tiếp tục quan trọng ở Day 10 centralized logging.

---

# 147. Một troubleshooting equation rất hữu ích

Hãy nghĩ service availability như dependency chain:

```text
Application availability
=
DNS
×
Network route
×
Firewall permission
×
NAT correctness
×
TCP connectivity
×
TLS correctness
×
BIG-IP Virtual Server
×
Pool availability
×
Backend health
×
Application response
```

Nếu một component = failure:

```text
overall service = failure
```

Bạn không cần chứng minh mọi component lỗi.

Bạn cần tìm **component đầu tiên trong flow không còn đáp ứng expected state**.

---

# 148. “First failing boundary”

Đây là một trong những troubleshooting techniques quan trọng nhất.

Suppose:

```text
Client → FortiGate     PASS
FortiGate → BIG-IP     PASS
BIG-IP TLS             PASS
BIG-IP → backend TCP   PASS
BIG-IP → backend TLS   FAIL
HTTP                   NOT REACHED
```

First failing boundary:

```text
server-side TLS
```

Vậy không cần troubleshoot HTTP response nữa.

Nó chưa bao giờ xảy ra.

---

# 149. Không nhảy layer

Nếu:

```text
TCP timeout
```

đừng bắt đầu bằng:

```text
HTTP headers
certificate SAN
```

Nếu:

```text
HTTP 500
```

đừng bắt đầu bằng:

```text
ARP troubleshooting
```

Layered troubleshooting tiết kiệm rất nhiều thời gian.

---

# 150. Day 8 liên hệ Day 9

Day 8 đã học:

```text
VLAN
trunk
routing
ARP
MAC
L2/L3
static route
```

Day 9 dùng toàn bộ nền tảng đó.

Ví dụ:

```text
Pool Member DOWN
```

có thể là:

```text
BIG-IP monitor issue
```

nhưng cũng có thể là:

```text
wrong VLAN
wrong route
gateway failure
ARP issue
```

Vì vậy load balancer troubleshooting không tách khỏi networking.

---

# 151. Liên hệ Day 10

Day 10 sẽ học centralized logs.

Evidence Day 9 trở thành input cho Day 10:

```text
FortiGate traffic logs
FortiGate security logs
BIG-IP LTM logs
server logs
```

Bạn sẽ bắt đầu correlate:

```text
same flow
same timestamp
multiple systems
```

---

# 152. Liên hệ Day 11

Day 11 sẽ đưa operational action vào:

```text
ServiceNow incident
change
approval
backout
audit history
```

Day 9 dạy kỹ thuật.

Day 11 dạy governance quanh kỹ thuật.

Production operation phải có cả hai.

---

# 153. Liên hệ Day 12–14

Day 12 automation:

```text
cURL
PowerShell
Python
JSON
exit codes
```

sẽ sử dụng mindset API/assertion của Day 9.

Day 14 integrated scenario có thể là:

```text
Application unavailable
→ FortiGate
→ BIG-IP
→ SSL
→ backend
→ logs
→ ticket
→ rollback
```

Day 9 vì vậy là một trong những Day quan trọng nhất của course.

---

# 154. Operational decision tree tổng hợp

```text
User cannot access HTTPS application
              |
              v
      DNS resolves correctly?
        /            \
      No              Yes
      |                |
 DNS investigation     v
                  TCP reachable?
                   /       \
                 No         Yes
                 |           |
       Route/firewall/NAT     v
                       TLS succeeds?
                        /       \
                      No         Yes
                      |           |
           Trust/chain/SNI/etc    v
                         HTTP response?
                           /       \
                         No         Yes
                         |           |
               BIG-IP/backend      HTTP code/body
                    health             |
                                       v
                                Application analysis
```

Ở nhánh “TCP unreachable”:

```text
Client
→ route
→ FortiGate interface
→ policy
→ NAT
→ session
→ packet flow
→ BIG-IP listener
```

Ở nhánh “TLS fail”:

```text
certificate
chain
trust
hostname
expiry
SNI
Client SSL
Server SSL
```

Ở nhánh “HTTP fail”:

```text
Virtual Server
Pool
Health Monitor
Host header
URI
backend application
```

---

# 155. Mental model cuối Module Lý thuyết

Nếu chỉ nhớ một diagram của Day 9, hãy nhớ cái này:

```text
CLIENT
  |
  | DNS
  | TCP SYN
  v
+---------------------------+
|         FORTIGATE         |
|                           |
| Interface                 |
| Route                     |
| Firewall Policy           |
| Security Profile          |
| NAT                       |
| Stateful Session          |
| Traffic/Security Logs     |
+---------------------------+
              |
              | TCP/443
              v
+---------------------------+
|        F5 BIG-IP LTM      |
|                           |
| Virtual Server            |
| TCP Profile               |
| Client SSL                |
| HTTP Profile              |
| Persistence               |
| Load Balancing            |
| SNAT                      |
| Server SSL                |
+---------------------------+
              |
              v
           POOL
        /        \
       /          \
      v            v
 MEMBER A       MEMBER B
  |                |
 Monitor         Monitor
  |                |
  v                v
APP SERVER      APP SERVER
```

Mỗi box phải trả lời được bốn câu:

```text
What is configured?
What is runtime state?
How do I verify it?
What evidence proves it?
```

Nếu bạn trả lời được bốn câu đó cho từng layer, bạn đã bắt đầu suy luận đúng kiểu Infrastructure Operations.

---

# 156. Module Lý thuyết Day 9 — Completion Checklist

| Kiến thức                 | Bạn cần có khả năng giải thích                           |
| ------------------------- | -------------------------------------------------------- |
| FortiGate                 | Vai trò firewall và FortiOS                              |
| Interface                 | Ingress/egress relationship                              |
| Zone                      | Logical interface grouping                               |
| Routing                   | Route selection và dependency với policy                 |
| Address object            | Abstraction của IP/subnet                                |
| Service object            | Protocol/port abstraction                                |
| Firewall policy           | Match criteria, action, sequence                         |
| Policy ordering           | Tại sao correct rule có thể không match                  |
| NAT                       | SNAT vs DNAT, policy NAT vs Central NAT awareness        |
| Security profiles         | Vì sao ACCEPT chưa chắc end result                       |
| Session                   | Runtime state và policy/NAT evidence                     |
| Traffic logs              | Flow evidence                                            |
| Security events           | Inspection/security evidence                             |
| VPN                       | Traffic path awareness                                   |
| Backup/revision           | Pre-change/rollback evidence                             |
| HA                        | Active/standby và operational awareness                  |
| FortiGate troubleshooting | Interface → route → policy → NAT → session → log → debug |
| BIG-IP LTM                | Vai trò load balancer                                    |
| Virtual Server            | Front-end listener                                       |
| Node                      | Backend IP object                                        |
| Pool Member               | Backend IP + service port                                |
| Pool                      | Backend collection                                       |
| Health Monitor            | Backend availability determination                       |
| Load balancing            | Member-selection algorithm                               |
| Persistence               | Session affinity                                         |
| SNAT                      | Source translation/return-path control                   |
| TCP profile               | TCP behavior                                             |
| HTTP profile              | Layer-7 HTTP behavior                                    |
| Client SSL                | Client ↔ BIG-IP TLS                                      |
| Server SSL                | BIG-IP ↔ backend TLS                                     |
| Certificate               | Identity/public key                                      |
| Chain                     | Leaf → intermediate → root                               |
| TLS validation            | Trust + chain + hostname + expiry                        |
| SNI                       | Hostname during TLS negotiation                          |
| `tmsh`                    | Configuration/status inspection                          |
| iControl REST             | Programmatic BIG-IP interface                            |
| HTTP methods              | GET vs mutating operations                               |
| Authentication            | Basic/token awareness                                    |
| JSON                      | Structured response parsing                              |
| cURL                      | HTTPS/API validation                                     |
| HTTP codes                | 2xx/4xx/5xx, đặc biệt 401/403                            |
| Timeouts                  | Bounded automation                                       |
| Retries                   | Không blindly retry mutating changes                     |
| Assertions                | Expected versus actual                                   |
| Evidence                  | Status + output + timestamp + non-secret fields          |
| Escalation                | Biết khi nào dừng L1/L2 và chuyển owner                  |

---

## Kết luận Module Lý thuyết Day 9

Day 9 không thực sự là bài học về “hai sản phẩm FortiGate và BIG-IP”. Nó là bài học về **end-to-end application traffic operations**.

Bạn cần nhìn một HTTPS request như chuỗi:

```text
DNS
→ routing
→ firewall policy
→ NAT
→ stateful session
→ TCP
→ TLS
→ Virtual Server
→ profiles
→ persistence
→ pool
→ health monitor
→ backend TLS/TCP
→ HTTP
→ application
```

Và tại mỗi layer bạn phải biết:

```text
Expected state là gì?
Runtime state là gì?
Read-only verification bằng gì?
Failure symptom trông như thế nào?
Evidence nào chứng minh?
Change nào là risky?
Rollback ra sao?
Khi nào phải escalate?
```

Đặc biệt, bốn distinction phải thuộc rất chắc trước khi sang thực hành:

```text
Firewall configuration ≠ runtime session state

TCP success ≠ TLS success

TLS success ≠ HTTP/application success

API request completed ≠ assertion passed
```

Đó chính là foundation cần có trước khi bước sang **Module Thực hành Day 9**, nơi chúng ta sẽ trực tiếp áp dụng FortiGate CLI, route/policy/session analysis, bounded debug, BIG-IP `tmsh`, Pool/Monitor checks, TLS validation, `curl`, iControl REST, JSON assertion, negative tests, failure injection, verification, evidence collection và rollback theo đúng Assignment/Lab trong syllabus.
