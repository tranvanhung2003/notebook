# Day 9 — Module Thực hành

## FortiGate Firewall Operations + F5 BIG-IP LTM, SSL/TLS & iControl REST

Module Thực hành Day 9 biến toàn bộ kiến thức lý thuyết thành một workflow vận hành hoàn chỉnh:

```text
Baseline
→ Read-only inspection
→ Connectivity test
→ Runtime-state verification
→ Fault localization
→ Bounded diagnostics
→ Controlled correction
→ Re-test
→ Rollback
→ Evidence
→ Escalation
```

Theo syllabus, bạn phải thực hiện hoặc inspect **FortiGate interfaces, routes, address/service objects, firewall policies**, dùng FortiGate CLI để kiểm tra **sessions, policy matches, routing và traffic logs**, test traffic allowed/blocked, xử lý một fault về policy/NAT; sau đó phải inspect/configure **BIG-IP Virtual Server, Pool, Pool Members, Health Monitor và SSL profile**, kiểm tra HTTP/HTTPS, certificate, backend availability, load-balancing behavior, sử dụng authenticated **iControl REST API + cURL**, parse JSON, troubleshoot Health Monitor/TLS và tạo automation checks. Infrastructure_Operations_14_Da…

Syllabus cũng yêu cầu evidence đầu ra cụ thể cho Day 9: **FortiGate CLI route/policy/session diagnosis + allow/deny + rollback**, và **BIG-IP Virtual Server/Pool/Monitor validation + TLS diagnosis + API/cURL assertion results**. Infrastructure_Operations_14_Da…

---

# 1. Mục tiêu thực hành cuối Day 9

Sau Module này, bạn phải có thể được giao một symptom kiểu:

> `https://app.lab.example` không truy cập được.

và không bắt đầu bằng việc restart bất cứ thứ gì.

Bạn phải lần lượt chứng minh:

```text
Client
  ↓
DNS
  ↓
Route
  ↓
FortiGate interface
  ↓
FortiGate route
  ↓
Firewall policy
  ↓
NAT
  ↓
Session
  ↓
BIG-IP Virtual Server
  ↓
Client-side TLS
  ↓
Pool
  ↓
Pool Member
  ↓
Health Monitor
  ↓
BIG-IP → Backend TCP/TLS
  ↓
HTTP/Application
```

Đồng thời phải có khả năng trả lời:

```text
Expected state là gì?
Actual state là gì?
Evidence nào chứng minh?
Failure boundary đầu tiên ở đâu?
Có cần configuration change không?
Nếu cần change thì rollback thế nào?
Nếu không thuộc scope thì escalate cho ai?
```

---

# 2. Lab environment mẫu

Tôi sẽ sử dụng topology chuẩn dưới đây trong toàn bộ module.

Đây là **lab/non-production topology**, không phải IP production.

```text
                         MANAGEMENT NETWORK
                      192.168.100.0/24
                              |
              +---------------+---------------+
              |                               |
      FortiGate Mgmt                  BIG-IP Mgmt
      192.168.100.10                  192.168.100.20
              |                               |
              |                               |
==========================================================

CLIENT NETWORK
10.10.10.0/24

Client01
10.10.10.50
   |
   |
   v
+------------------------------------------------+
|                  FortiGate                     |
|                                                |
| port2 / CLIENT                                 |
| 10.10.10.1/24                                  |
|                                                |
| port3 / APP-DMZ                                |
| 172.16.10.1/24                                 |
+------------------------------------------------+
                     |
                     |
                     v
             BIG-IP External VLAN

BIG-IP Virtual Server:
172.16.10.50:443
Hostname:
app.lab.example

                     |
               BIG-IP LTM
                     |
              Internal VLAN
              10.20.30.0/24
                     |
          +----------+----------+
          |                     |
          v                     v
      Backend01             Backend02
    10.20.30.11           10.20.30.12
      TCP/8443              TCP/8443
```

Tên objects dùng trong ví dụ:

```text
FortiGate:

LAB_CLIENT
LAB_BIGIP_VIP
LAB_HTTPS
LAB_CLIENT_TO_BIGIP_HTTPS

BIG-IP:

VS_APP_HTTPS
POOL_APP_HTTPS
MON_APP_HTTPS
CLIENTSSL_APP
SERVERSSL_APP

Pool Members:

10.20.30.11:8443
10.20.30.12:8443
```

Nếu environment thật của bạn sử dụng VLAN, Zone, VDOM, Central NAT, HA hoặc partition khác `/Common`, hãy thay bằng giá trị thực tế.

---

# 3. Prerequisites

Trước lab, cần có:

| Requirement                                              | Trạng thái                       |
| -------------------------------------------------------- | -------------------------------- |
| Authorized FortiGate lab account                         | Required                         |
| Authorized BIG-IP lab account                            | Required                         |
| Client Linux/Windows host                                | Required                         |
| Two backend test servers hoặc fixtures                   | Recommended                      |
| HTTPS test application                                   | Required                         |
| DNS entry hoặc `/etc/hosts` lab mapping                  | Required                         |
| `curl`                                                   | Required                         |
| `openssl`                                                | Recommended                      |
| JSON parser như `jq`                                     | Recommended                      |
| SSH client                                               | Required                         |
| Evidence folder                                          | Required                         |
| Lab change ticket/reference                              | Required                         |
| Known-good baseline                                      | Required                         |
| FortiGate backup/revision                                | Required trước change            |
| BIG-IP configuration backup/snapshot method theo runbook | Required trước meaningful change |

Không dùng real production credentials, real private keys hoặc tokens trong lab evidence.

---

# 4. Safety rules — bắt buộc trước khi chạy command

Có bốn loại thao tác.

### Class A — Read-only

Ví dụ:

```bash
get system status
get router info routing-table all
diagnose sys session list

tmsh list ltm virtual
tmsh show ltm pool
```

Đây là những command ưu tiên.

### Class B — Diagnostic nhưng có operational cost

Ví dụ:

```bash
diagnose sniffer packet ...
diagnose debug flow ...
```

Không necessarily thay đổi configuration, nhưng có thể tạo nhiều output và CPU/logging load.

Phải:

```text
Filter chặt
Bounded count
Bounded duration
Stop ngay khi đủ evidence
```

Fortinet cảnh báo `diagnose debug flow` có thể sinh lượng data đáng kể, đặc biệt khi output được gửi tới FortiAnalyzer/FortiCloud, vì vậy không được để chạy lâu không giới hạn. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.0/administration-guide/54688/debugging-the-packet-flow?utm_source=chatgpt.com)

### Class C — Configuration-changing

Ví dụ:

```text
Create firewall address
Modify firewall policy
Change NAT
Modify pool member
Change monitor
Replace SSL profile
```

Chỉ chạy trong authorized lab hoặc approved change.

### Class D — High-risk

Ví dụ:

```text
Broad session clear
HA failover
Disable hardware offload
Production certificate replacement
Route modification
Delete policy
Restore entire firewall config
```

Không dùng như bước troubleshooting thông thường.

---

# 5. Evidence folder structure

Tạo repository:

```text
Day09/
├── 00_manifest/
├── 01_baseline/
├── 02_fortigate/
│   ├── interfaces/
│   ├── routes/
│   ├── policies/
│   ├── sessions/
│   ├── packet_capture/
│   └── logs/
├── 03_bigip/
│   ├── virtual_server/
│   ├── pool/
│   ├── monitor/
│   ├── ssl/
│   └── logs/
├── 04_api/
├── 05_failure_injection/
├── 06_rollback/
└── 07_final_report/
```

Mỗi evidence item tối thiểu nên có:

```text
Evidence ID
Device
Product
Version
Hostname
Environment
Collection timestamp
Timezone
Command/query
Source/destination
Ticket/change reference
Expected result
Actual result
Result interpretation
Collector
```

Nếu export thành file:

```text
Filename
File size
Record count nếu có
SHA-256
```

Yêu cầu này kế thừa evidence contract từ Day 1 của syllabus. Infrastructure_Operations_14_Da…

---

# PHASE 0 — BASELINE & PRE-CHECK

# 6. Lab 0.1 — Ghi nhận FortiGate identity/version

Đăng nhập FortiGate bằng approved SSH account.

Chạy:

```bash
get system status
```

Mục tiêu là xác định:

```text
Hostname
FortiOS version
Serial/VM information nếu được phép lưu
Operation mode
HA context nếu hiển thị
```

Evidence mẫu:

```text
EV-D09-FG-001
Device: FGT-LAB01
Product: FortiGate
Version: FortiOS 7.x.x
Environment: LAB
Collection time: ...
```

Không paste license keys hoặc sensitive registration data nếu output có.

---

# 7. Lab 0.2 — BIG-IP identity/version

Trên BIG-IP:

```bash
tmsh show sys version
```

hoặc exact version command theo runbook môi trường.

Ghi nhận:

```text
Hostname
BIG-IP version
Build
Edition/modules relevant
HA/standalone context
```

Điều này rất quan trọng vì iControl REST guide hiện tại của F5 ghi guide 16.1 vẫn applicable cho 17.0.x, 17.1.x và 17.5.x, nhưng API properties và product behavior vẫn phải được xác minh theo deployed version. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/?utm_source=chatgpt.com)

---

# 8. Lab 0.3 — Establish expected application flow

Trước troubleshooting, điền bảng expected flow:

| Field                 | Expected                    |
| --------------------- | --------------------------- |
| Client                | `10.10.10.50`               |
| Application hostname  | `app.lab.example`           |
| VIP                   | `172.16.10.50`              |
| Protocol              | TCP                         |
| Client port           | Ephemeral                   |
| Destination port      | 443                         |
| FortiGate ingress     | `port2`                     |
| FortiGate egress      | `port3`                     |
| Firewall policy       | `LAB_CLIENT_TO_BIGIP_HTTPS` |
| NAT                   | Theo lab design             |
| BIG-IP Virtual Server | `VS_APP_HTTPS`              |
| BIG-IP Pool           | `POOL_APP_HTTPS`            |
| Pool Member 1         | `10.20.30.11:8443`          |
| Pool Member 2         | `10.20.30.12:8443`          |
| Health Monitor        | `MON_APP_HTTPS`             |
| Expected HTTP         | 200                         |
| Expected TLS hostname | `app.lab.example`           |

Đây là **expected-state contract**.

Không có bảng này, bạn rất dễ troubleshooting dựa trên assumption.

---

# PHASE 1 — FORTIGATE READ-ONLY INSPECTION

# 9. Lab 1.1 — Inspect interfaces

Command:

```bash
show system interface
```

Nếu output quá lớn, sử dụng specific interface:

```bash
show system interface port2
show system interface port3
```

Bạn muốn xác nhận:

```text
Interface exists
Administrative state
IP/subnet
VDOM
VLAN/parent relationship nếu applicable
Zone relationship nếu applicable
Management access nếu relevant
```

Expected:

```text
port2 = client-facing
port3 = BIG-IP-facing
```

Một output conceptual:

```text
edit "port2"
    set vdom "root"
    set ip 10.10.10.1 255.255.255.0
next

edit "port3"
    set vdom "root"
    set ip 172.16.10.1 255.255.255.0
next
```

Không kết luận interface hoạt động chỉ vì configuration tồn tại.

Bạn còn phải correlate:

```text
configuration
+
runtime link/status
+
actual packets
```

---

# 10. Checkpoint: sai interface

Nếu FortiGate policy đang dùng:

```text
srcintf = LAN-ZONE
```

nhưng bạn chỉ tìm:

```text
srcintf = port2
```

có thể tưởng policy sai.

Kiểm tra zone design trước.

Nếu sử dụng VDOM, cũng phải chắc chắn bạn đang ở đúng VDOM context.

---

# 11. Lab 1.2 — Inspect routing

Chạy:

```bash
get router info routing-table all
```

Fortinet hiện document command này để verify routing table khi connectivity có vấn đề. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.5/administration-guide/221343?utm_source=chatgpt.com)

Tìm route tới:

```text
172.16.10.50
```

và nếu cần route reply/client network.

Ví dụ expected:

```text
C>* 172.16.10.0/24 is directly connected, port3
C>* 10.10.10.0/24 is directly connected, port2
```

Nếu destination là remote network:

```text
S>* 172.16.10.0/24 via 192.168.x.x, port3
```

---

# 12. Verification routing

Bạn cần trả lời:

```text
Destination prefix nào match?
Prefix length?
Route source?
Next hop?
Egress interface?
Route actually selected?
```

Đừng chỉ screenshot một route có vẻ liên quan.

Evidence phải chỉ rõ:

```text
172.16.10.50 matches route 172.16.10.0/24
selected egress = port3
```

---

# 13. Lab 1.3 — Inspect address objects

Read-only:

```bash
show firewall address
```

Nếu object known:

```bash
show firewall address LAB_CLIENT
show firewall address LAB_BIGIP_VIP
```

Xác nhận:

```text
LAB_CLIENT
→ 10.10.10.50/32

LAB_BIGIP_VIP
→ 172.16.10.50/32
```

Một common fault:

```text
Object name đúng
Object value sai
```

Ví dụ:

```text
LAB_BIGIP_VIP
→ 172.16.10.51
```

Tên object không phải evidence của IP thực.

---

# 14. Lab 1.4 — Inspect service object

Nếu dùng custom service:

```bash
show firewall service custom LAB_HTTPS
```

Expected conceptual:

```text
TCP port 443
```

Kiểm tra:

```text
Protocol = TCP?
Destination port = 443?
Port range có rộng bất thường không?
```

Một object:

```text
HTTPS
```

nhưng chứa:

```text
TCP/8443
```

là valid object technically nhưng sai requirement.

---

# 15. Lab 1.5 — Inspect firewall policy

Command:

```bash
show firewall policy
```

Hoặc:

```bash
show firewall policy <POLICY_ID>
```

Tìm policy:

```text
LAB_CLIENT_TO_BIGIP_HTTPS
```

Expected conceptual configuration:

```text
Incoming interface: port2
Outgoing interface: port3
Source: LAB_CLIENT
Destination: LAB_BIGIP_VIP
Service: LAB_HTTPS
Schedule: always
Action: ACCEPT
Logging: enabled
NAT: according to design
```

---

# 16. Policy review worksheet

Không chỉ screenshot policy. Điền bảng:

| Criterion          | Expected                | Actual | Result    |
| ------------------ | ----------------------- | ------ | --------- |
| Incoming interface | port2                   | ...    | PASS/FAIL |
| Outgoing interface | port3                   | ...    | PASS/FAIL |
| Source             | LAB_CLIENT              | ...    |           |
| Destination        | LAB_BIGIP_VIP           | ...    |           |
| Protocol/Service   | TCP/443                 | ...    |           |
| Schedule           | always                  | ...    |           |
| Action             | ACCEPT                  | ...    |           |
| NAT                | expected state          | ...    |           |
| Logging            | enabled                 | ...    |           |
| Policy order       | before conflicting rule | ...    |           |

Policy có thể đúng 8/9 criteria nhưng vẫn fail nếu một criterion sai.

---

# 17. Lab 1.6 — NAT mode identification

Trước khi diagnose NAT, xác định environment dùng:

```text
Policy NAT
```

hay:

```text
Central SNAT
```

Nếu policy NAT, inspect policy:

```bash
show firewall policy <POLICY_ID>
```

Tìm NAT setting.

Nếu Central SNAT được enable, inspect Central SNAT table theo deployed version/runbook, thường liên quan:

```bash
show firewall central-snat-map
```

Fortinet hiện mô tả Central SNAT rules được quản lý trong `central-snat-map`, đọc theo thứ tự và áp dụng theo Central NAT workflow. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/54688/debugging-the-packet-flow?utm_source=chatgpt.com)

Nếu DNAT/VIP involved, inspect:

```bash
show firewall vip
```

Không được troubleshoot NAT trước khi biết NAT architecture.

---

# 18. Lab 1.7 — Baseline client test

Từ Client01:

```bash
curl -v \
  --connect-timeout 5 \
  --max-time 10 \
  https://app.lab.example/
```

Nếu CA của lab trusted:

Expected:

```text
DNS resolve
TCP connect
TLS handshake
HTTP response
```

Nếu lab CA riêng:

```bash
curl \
  --cacert ./lab-ca.pem \
  --connect-timeout 5 \
  --max-time 10 \
  -v \
  https://app.lab.example/
```

Không dùng:

```bash
curl -k
```

làm final validation.

Syllabus explicitly yêu cầu không bypass TLS validation để làm test “pass”. Infrastructure_Operations_14_Da…

---

# 19. Lab 1.8 — Generate a fresh session

Để session investigation có ý nghĩa, tạo connection mới.

Ví dụ:

```bash
curl \
  --cacert ./lab-ca.pem \
  --connect-timeout 5 \
  --max-time 10 \
  https://app.lab.example/
```

Ghi timestamp:

```bash
date -Ins
```

hoặc Windows equivalent.

Timestamp này dùng correlate với FortiGate traffic logs và BIG-IP evidence.

---

# 20. Lab 1.9 — FortiGate session filtering

FortiOS hiện hỗ trợ session filtering theo source, destination, ports, policy ID, protocol và NAT information. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/cli-reference/235530229/diagnose-sys?utm_source=chatgpt.com)

Clear old filters:

```bash
diagnose sys session filter clear
```

Set source:

```bash
diagnose sys session filter src 10.10.10.50
```

Destination:

```bash
diagnose sys session filter dst 172.16.10.50
```

Destination port:

```bash
diagnose sys session filter dport 443
```

Protocol TCP:

```bash
diagnose sys session filter proto 6
```

Display:

```bash
diagnose sys session list
```

---

# 21. Session output phải tìm gì?

Tìm:

```text
Source tuple
Destination tuple
Policy ID
Interfaces/direction
NAT information
Session state
Duration
Counters
```

Bạn không cần decode mọi flag trong Day 9.

Mục tiêu là trả lời:

```text
Session có tồn tại?
Policy nào tạo session?
Source/destination đúng không?
NAT có xảy ra không?
Connection có state hợp lý không?
```

---

# 22. Cách reasoning từ session

Case A:

```text
Client test running
Session exists
Policy ID expected
```

→ FortiGate ít nhất đã nhận và classify connection.

Case B:

```text
No matching session
```

Hypotheses:

```text
Traffic chưa tới FortiGate
Filter sai
VDOM/context sai
Traffic offload/diagnostic limitation
Connection không được tạo
```

Không được kết luận ngay:

> Firewall dropped traffic.

---

# 23. Clean up session filters

Sau inspection:

```bash
diagnose sys session filter clear
```

Evidence:

```text
EV-D09-FG-SESSION-001.txt
```

Phải ghi exact filters đã dùng.

---

# PHASE 2 — ALLOW / DENY TESTS

# 24. Lab 2.1 — Positive test

Requirement:

```text
Client01 → app.lab.example TCP/443
Expected = ALLOW
```

Run:

```bash
curl \
  --cacert ./lab-ca.pem \
  --connect-timeout 5 \
  --max-time 10 \
  -sS \
  -o response.txt \
  -w 'HTTP=%{http_code}\n' \
  https://app.lab.example/
```

Expected:

```text
HTTP=200
curl exit code = 0
```

Capture exit status immediately:

```bash
echo $?
```

---

# 25. Positive test evidence chain

Không chỉ lưu Client output.

Cần ít nhất:

```text
Client cURL
FortiGate session
FortiGate traffic log
BIG-IP Virtual Server status
Pool Member status
```

Khi năm data points đồng thuận, evidence mạnh hơn nhiều.

---

# 26. Lab 2.2 — Negative test

Tạo một flow đã được thiết kế expected-deny.

Ví dụ:

```text
Client01 → VIP TCP/444
Expected = DENY
```

Run:

```bash
curl \
  --connect-timeout 5 \
  --max-time 10 \
  https://app.lab.example:444/
```

Expected có thể là:

```text
timeout
connection refused
policy denial
```

tùy lab design.

Không coi mọi denied flow phải có cùng symptom.

---

# 27. Negative test success criterion

Đây là một concept quan trọng:

Expected:

```text
DENY
```

Actual:

```text
DENY
```

thì testcase:

```text
PASS
```

Một security negative test “fail to connect” có thể là **successful test result**.

Phải phân biệt:

```text
Connectivity result
```

và:

```text
Testcase result
```

---

# 28. Policy hit/log verification

Nếu traffic logging enabled, kiểm tra traffic logs trong FortiGate GUI/logging interface.

Filter theo:

```text
Timestamp
Source IP
Destination IP
Port
Policy ID
```

Không dùng screenshot unfiltered chứa hàng nghìn events.

Evidence tốt:

```text
Window: 14:31:20–14:31:40 ICT
srcip=10.10.10.50
dstip=172.16.10.50
dstport=443
```

Syllabus yêu cầu policy hit/log analysis và traffic-log evidence. Infrastructure_Operations_14_Da…

---

# PHASE 3 — PACKET CAPTURE & BOUNDED DEBUG

# 29. Khi nào mới dùng packet capture?

Không phải bước 1.

Chỉ dùng sau khi đã kiểm tra:

```text
Interface
Route
Policy
Objects
Session
Logs
```

Packet capture phù hợp khi câu hỏi trở thành:

> Packet thực sự đi vào/đi ra interface nào?

---

# 30. Lab 3.1 — FortiGate packet sniffer

FortiOS hỗ trợ:

```bash
diagnose sniffer packet <interface> <filter> <verbose> <count> <timestamp>
```

với filters kiểu tcpdump, count giới hạn và timestamp format. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.4/cli-reference/303299696/diagnose-sniffer?utm_source=chatgpt.com)

Ví dụ bounded capture:

```bash
diagnose sniffer packet any \
'tcp and host 10.10.10.50 and host 172.16.10.50 and port 443' \
4 30 l
```

Ý nghĩa:

```text
any
→ capture relevant interfaces

filter
→ chỉ đúng client/VIP/443

4
→ packet header + interface information trên FortiOS releases hỗ trợ level này

30
→ maximum 30 packets

l
→ local timestamp
```

Exact syntax phải verify theo FortiOS release.

---

# 31. Expected TCP handshake

Bạn kỳ vọng thấy:

```text
Client → VIP: SYN
VIP → Client: SYN/ACK
Client → VIP: ACK
```

Nếu chỉ thấy:

```text
Client → VIP: SYN
Client → VIP: SYN
Client → VIP: SYN
```

không có return traffic, failure boundary cần điều tra.

---

# 32. Packet-capture inference

Scenario 1:

```text
Ingress SYN visible
Egress SYN visible
No SYN/ACK returned
```

Likely downstream issue.

Scenario 2:

```text
Ingress SYN visible
No egress SYN
```

Likely FortiGate forwarding/policy/route/NAT issue.

Scenario 3:

```text
No ingress SYN
```

Issue nằm trước firewall hoặc capture/filter sai.

---

# 33. Lab 3.2 — Bounded `diagnose debug flow`

Fortinet official guidance supports filters như `saddr`, `daddr`, `dport`, `proto`, bounded `trace start <N>` và explicit `trace stop`. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/54688/debugging-the-packet-flow?utm_source=chatgpt.com)

Reset existing debug:

```bash
diagnose debug reset
```

Clear filter:

```bash
diagnose debug flow filter clear
```

Set narrow filters:

```bash
diagnose debug flow filter saddr 10.10.10.50
diagnose debug flow filter daddr 172.16.10.50
diagnose debug flow filter dport 443
diagnose debug flow filter proto 6
```

Enable function names:

```bash
diagnose debug flow show function-name enable
```

Add timestamp if supported by runbook:

```bash
diagnose debug console timestamp enable
```

Enable debug:

```bash
diagnose debug enable
```

Start bounded trace:

```bash
diagnose debug flow trace start 100
```

Generate exactly one or a few client connections.

Then stop:

```bash
diagnose debug flow trace stop
diagnose debug disable
diagnose debug reset
```

---

# 34. Debug cleanup evidence

Không chỉ lưu debug output.

Phải lưu output chứng minh debug đã được stopped/reset.

Syllabus yêu cầu cleanup evidence. Infrastructure_Operations_14_Da…

Ví dụ manifest:

```text
Debug start: 14:42:11 ICT
Debug stop:  14:42:24 ICT
Filter:
src=10.10.10.50
dst=172.16.10.50
dport=443
proto=6

Trace limit:
100
```

---

# 35. NPU offload caveat

Fortinet xác nhận `diagnose debug flow` bình thường chỉ thấy software-processed traffic; sessions offloaded vào NP6/NP7 có thể không xuất hiện đầy đủ. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.0/administration-guide/926361?utm_source=chatgpt.com)

Vì vậy:

```text
No debug-flow output
```

không tự động bằng:

```text
No traffic.
```

Trong production:

**không disable NPU offload chỉ để làm lab troubleshooting**.

Nếu cần diagnostic sâu hơn, follow vendor runbook/escalation.

---

# PHASE 4 — FORTIGATE FAILURE INJECTION

# 36. Failure Injection 1 — Incorrect service object

Instructor chuẩn bị:

```text
LAB_HTTPS
TCP/8443
```

trong khi client cần:

```text
TCP/443
```

Symptom:

```text
Route correct
Interfaces correct
Policy exists
But traffic doesn't match expected service
```

Your investigation:

```text
show firewall service custom LAB_HTTPS
show firewall policy <ID>
```

Root cause:

```text
Service object mismatch
```

Corrective action trong authorized lab:

```text
Restore LAB_HTTPS to TCP/443
```

Sau đó:

```text
Generate new connection
Check session
Check log
Retest
```

---

# 37. Failure Injection 2 — Wrong policy order

Prepared configuration:

```text
Policy 10
CLIENT_NET → BIGIP_VIP
HTTPS
DENY

Policy 20
LAB_CLIENT → BIGIP_VIP
HTTPS
ALLOW
```

Nếu traffic match broader Policy 10 trước:

```text
Policy 20 never receives traffic
```

Evidence cần:

```text
Policy sequence
Traffic log policy ID
Expected policy ID
Actual policy ID
```

Không được chỉ nói:

> Move allow rule higher.

Phải chứng minh vì sao.

---

# 38. Failure Injection 3 — Wrong NAT configuration

Prepared fault:

Expected:

```text
No SNAT
```

Actual:

```text
Unexpected SNAT
```

hoặc ngược lại.

Possible symptom:

```text
BIG-IP sees unexpected source
Return path fails
ACL downstream rejects translated address
```

Investigation:

```text
Policy NAT state
Central SNAT state
Session translation
Packet capture
```

Root-cause evidence phải chỉ ra:

```text
Original tuple
Translated tuple
Expected tuple
```

---

# 39. Controlled correction requirement

Trước fix:

```text
Take baseline
Record policy ID
Record current NAT
Record session
Capture relevant logs
Confirm approval
```

Sau fix:

```text
Re-test original flow
Verify expected session
Verify traffic log
Verify BIG-IP receives connection
```

Sau đó thực hiện rollback exercise nếu syllabus/instructor yêu cầu.

---

# 40. FortiGate configuration backup

Fortinet hiện hỗ trợ configuration backups/revisions trong GUI/CLI và nhấn mạnh việc backup configuration trước khi cần recover. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/administration-guide/702257/configuration-backups-and-reset?utm_source=chatgpt.com)

Trong lab, exact command phải follow approved runbook.

Ví dụ relevant revision operation documented hiện tại:

```bash
execute revision list config
```

Một số environments có thể dùng:

```bash
execute backup config flash <comment>
```

Không dùng restore command như một experiment tùy tiện.

Restore entire configuration có blast radius lớn.

---

# PHASE 5 — BIG-IP BASELINE

# 41. Lab 5.1 — Map LTM object dependency

Trước command, viết dependency:

```text
VS_APP_HTTPS
       |
       v
POOL_APP_HTTPS
       |
       +---------------------+
       |                     |
       v                     v
10.20.30.11:8443       10.20.30.12:8443
       |                     |
       +---------+-----------+
                 |
                 v
           MON_APP_HTTPS
```

Profiles:

```text
VS_APP_HTTPS
    |
    +-- TCP profile
    +-- HTTP profile
    +-- CLIENTSSL_APP
    +-- SERVERSSL_APP
```

Bạn phải biết dependency trước khi troubleshoot.

---

# 42. Lab 5.2 — Inspect Virtual Server configuration

Read-only:

```bash
tmsh list ltm virtual VS_APP_HTTPS
```

Muốn xem properties chi tiết hơn, có thể dùng theo runbook/version:

```bash
tmsh list ltm virtual VS_APP_HTTPS all-properties
```

`tmsh list ... all-properties` là syntax được F5 document để xem toàn bộ object properties. [techdocs.f5.com](https://techdocs.f5.com/content/kb/en-us/products/big-ip_ltm/manuals/product/bigip-tmsh-reference-11-6-0/_jcr_content/pdfAttach/download/file.res/bigip-tmsh-reference-11-6-0.pdf?utm_source=chatgpt.com)

Check:

```text
Destination
Port
IP protocol
Default Pool
Profiles
SNAT/source-address-translation
Persistence
VLAN restrictions nếu có
```

Expected:

```text
destination = 172.16.10.50:443
pool = POOL_APP_HTTPS
```

---

# 43. Lab 5.3 — Virtual Server runtime state

```bash
tmsh show ltm virtual VS_APP_HTTPS
```

Quan sát:

```text
Availability
State
Connections
Bits/packets
Statistics
```

Không cần thuộc hết counters.

Mục tiêu:

```text
Client request có làm counters increment không?
Virtual Server có available/enabled không?
```

Nếu request client chạy nhưng counter không thay đổi:

```text
traffic likely not reaching this VS
```

hoặc bạn đang inspect wrong object/context.

---

# 44. Lab 5.4 — Inspect Pool config

```bash
tmsh list ltm pool POOL_APP_HTTPS
```

Kiểm tra:

```text
Members
Monitor
Load-balancing mode
Optional priority/ratio settings
```

F5 định nghĩa iControl REST `/mgmt/tm/ltm/pool` là load-balancing pool configuration resource và pool có members subcollection. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/APIRef_tm_ltm_pool.html?utm_source=chatgpt.com)

---

# 45. Lab 5.5 — Inspect Pool runtime state

```bash
tmsh show ltm pool POOL_APP_HTTPS
```

Hoặc member-specific output theo version:

```bash
tmsh show ltm pool POOL_APP_HTTPS members
```

F5 documentation/release references use `tmsh show ltm pool <pool> members` để inspect member state. [techdocs.f5.com](https://techdocs.f5.com/kb/en-us/products/big-ip_ltm/releasenotes/related/relnote-supplement-bigip-15-1-3-1.html?utm_source=chatgpt.com)

Expected:

```text
10.20.30.11:8443 → available/up
10.20.30.12:8443 → available/up
```

---

# 46. Configured state vs runtime state

Ví dụ:

```text
tmsh list ltm pool
```

shows:

```text
10.20.30.11:8443
```

Điều đó chỉ chứng minh:

```text
Member exists in config.
```

Trong khi:

```text
tmsh show ltm pool
```

mới giúp trả lời:

```text
Member currently healthy/available?
```

Đây là distinction rất quan trọng.

---

# 47. Lab 5.6 — Health Monitor inspection

Inspect Pool configuration để xác định monitor attached.

Ví dụ:

```bash
tmsh list ltm pool POOL_APP_HTTPS
```

Nếu monitor custom:

```bash
tmsh list ltm monitor https MON_APP_HTTPS
```

hoặc:

```bash
tmsh list ltm monitor http MON_APP_HTTPS
```

tùy monitor type.

Check:

```text
Monitor type
Interval
Timeout
Destination/alias
Send string
Receive expectation
SSL settings
```

F5 current documentation mô tả health monitor là mechanism xác định availability/performance của resource; nếu resource không đáp ứng trong timeout hoặc health condition fail, BIG-IP có thể chuyển traffic sang resource khác. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-16-0-0/big-ip-local-traffic-manager-monitors-reference/monitors-concepts.html?utm_source=chatgpt.com)

---

# 48. Monitor reasoning worksheet

Ví dụ:

```text
Type:
HTTPS

Target:
member port 8443

Request:
GET /health HTTP/1.1
Host: app.lab.example

Expected:
HTTP 200 / expected body
```

Questions:

```text
Backend port đúng?
Path đúng?
Host header đúng?
TLS/SNI required?
Expected body realistic?
Interval/timeout reasonable?
```

---

# PHASE 6 — BACKEND DIRECT VALIDATION

# 49. Tại sao test backend trực tiếp?

Nếu Pool Member DOWN, cần phân biệt:

```text
Backend broken
```

và:

```text
BIG-IP monitor broken.
```

Từ một authorized host có network path tương đương, test:

```bash
curl \
  --connect-timeout 5 \
  --max-time 10 \
  https://10.20.30.11:8443/health
```

Nhưng nếu backend requires hostname/SNI, direct IP test có thể sai.

Better:

```bash
curl \
  --resolve app.lab.example:8443:10.20.30.11 \
  --cacert ./backend-ca.pem \
  --connect-timeout 5 \
  --max-time 10 \
  https://app.lab.example:8443/health
```

---

# 50. Why `--resolve` matters

Command:

```bash
--resolve app.lab.example:8443:10.20.30.11
```

giữ:

```text
Hostname = app.lab.example
SNI = app.lab.example
```

nhưng kết nối tới:

```text
10.20.30.11
```

Do đó test gần hơn với actual HTTPS behavior.

---

# 51. Compare backend01/backend02

```bash
for ip in 10.20.30.11 10.20.30.12
do
  curl \
    --resolve app.lab.example:8443:$ip \
    --cacert ./backend-ca.pem \
    --connect-timeout 5 \
    --max-time 10 \
    -sS \
    -o /dev/null \
    -w "$ip HTTP=%{http_code}\n" \
    https://app.lab.example:8443/health
done
```

Expected:

```text
10.20.30.11 HTTP=200
10.20.30.12 HTTP=200
```

---

# 52. Interpretation

Case:

```text
Backend direct test = 200
BIG-IP monitor = DOWN
```

Investigate:

```text
Monitor request
Monitor source
TLS settings
Host header
Receive string
Firewall path between BIG-IP and backend
```

Case:

```text
Backend direct test = fail
BIG-IP monitor = DOWN
```

Monitor may actually be working correctly.

Do not “fix” monitor to mark broken application UP.

---

# PHASE 7 — LOAD BALANCING TEST

# 53. Prepare distinguishable backends

Backend responses should include server identity.

Backend01:

```text
backend01
```

Backend02:

```text
backend02
```

Example endpoint:

```text
/whoami
```

Responses:

```text
backend01
```

or:

```text
backend02
```

---

# 54. Lab 7.1 — Repeated requests

```bash
for i in $(seq 1 10)
do
  curl \
    --cacert ./lab-ca.pem \
    --connect-timeout 5 \
    --max-time 10 \
    -sS \
    https://app.lab.example/whoami
  echo
done
```

Possible output:

```text
backend01
backend02
backend01
backend02
...
```

Không nhất thiết phải 50/50 chính xác.

---

# 55. Nếu tất cả requests vào một backend

Không kết luận load balancing broken ngay.

Investigate:

```text
Persistence
Connection reuse
Pool member availability
Load-balancing method
HTTP client behavior
Existing sessions
```

Nếu cookie persistence configured, cùng client có thể intentionally đi về cùng member.

---

# 56. Test connection reuse

`curl` process mới mỗi lần thường tạo connections khác với một persistent HTTP client.

Do đó test design phải được document:

```text
One curl process per request?
Same TCP session?
Cookies retained?
Persistence configured?
```

Một test không kiểm soát các yếu tố này rất dễ cho kết luận sai.

---

# PHASE 8 — TLS / CERTIFICATE VALIDATION

# 57. Lab 8.1 — cURL TLS validation

Run:

```bash
curl \
  --cacert ./lab-ca.pem \
  --connect-timeout 5 \
  --max-time 10 \
  -v \
  https://app.lab.example/
```

Check:

```text
Resolved IP
Connected IP/port
TLS version
Certificate verification result
HTTP status
```

---

# 58. Lab 8.2 — OpenSSL handshake

```bash
openssl s_client \
  -connect 172.16.10.50:443 \
  -servername app.lab.example \
  -showcerts
```

Important:

```text
-connect
→ destination IP/port

-servername
→ SNI hostname

-showcerts
→ certificates presented
```

Check:

```text
Leaf certificate
Intermediate certificates
Issuer
SAN
Validity
Verify return code
```

---

# 59. Certificate metadata

Bạn có thể pipe certificate sang:

```bash
openssl x509 -noout \
  -subject \
  -issuer \
  -dates \
  -ext subjectAltName
```

Expected:

```text
SAN includes app.lab.example
Not Before valid
Not After future
Issuer expected
```

Do not save private key.

---

# 60. TLS acceptance criteria

Một HTTPS endpoint chỉ pass TLS validation nếu:

```text
TCP connection succeeds
AND
TLS handshake succeeds
AND
trust validation succeeds
AND
certificate chain acceptable
AND
hostname matches
AND
certificate time validity acceptable
```

Không chỉ:

```text
Certificate has not expired.
```

---

# 61. Client SSL verification on BIG-IP

Inspect Virtual Server profiles:

```bash
tmsh list ltm virtual VS_APP_HTTPS
```

Identify attached Client SSL profile.

Then:

```bash
tmsh list ltm profile client-ssl CLIENTSSL_APP
```

F5 BIG-IP 17.5.1 documentation confirms Client SSL profile handles client-side TLS, decrypting incoming TLS and encrypting responses; certificates/keys are attached through the profile. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-17-5-1/big-ip-system-ssl-administration/managing-client-side-http-traffic-using-a-ca-signed-rsa-certificate.html?utm_source=chatgpt.com)

Inspect:

```text
Certificate references
Key references
CA/trust options
SNI-related settings if applicable
```

Do not display or export private-key content.

---

# 62. Server SSL verification

If architecture is:

```text
Client HTTPS
→ BIG-IP
→ Backend HTTPS
```

identify Server SSL profile:

```bash
tmsh list ltm virtual VS_APP_HTTPS
```

then:

```bash
tmsh list ltm profile server-ssl SERVERSSL_APP
```

Questions:

```text
Is Server SSL enabled?
Does backend require TLS?
Does backend require SNI?
Does BIG-IP validate backend certificate?
Which CA is trusted?
```

Do not assume Client SSL success proves Server SSL success.

---

# PHASE 9 — HEALTH MONITOR FAILURE

# 63. Failure Injection 4 — Backend health endpoint broken

Instructor changes Backend02:

```text
GET /health
→ HTTP 500
```

but `/` might still return:

```text
HTTP 200
```

Expected BIG-IP result:

```text
Backend02 Pool Member → DOWN
Backend01 → UP
```

---

# 64. Investigation sequence

Do not immediately edit monitor.

First:

```bash
tmsh show ltm pool POOL_APP_HTTPS
```

Then:

```bash
tmsh list ltm pool POOL_APP_HTTPS
```

Identify monitor.

Then test backend directly:

```bash
curl \
  --resolve app.lab.example:8443:10.20.30.12 \
  --cacert ./backend-ca.pem \
  -i \
  https://app.lab.example:8443/health
```

If actual:

```text
HTTP/1.1 500
```

then Health Monitor may be behaving correctly.

---

# 65. Correct operational response

Wrong fix:

```text
Change monitor from /health to /
```

just to make Pool Member green.

Correct response:

```text
Monitor expected condition documented
Backend violates expected condition
Escalate to application owner
```

unless change ticket explicitly says monitor requirement was wrong.

---

# PHASE 10 — TLS FAILURE INJECTION

# 66. Failure Injection 5 — Expired/untrusted fixture

Use prepared lab endpoint with:

```text
expired certificate
```

hoặc:

```text
certificate signed by untrusted lab CA
```

Run:

```bash
curl \
  --connect-timeout 5 \
  --max-time 10 \
  https://badcert.lab.example/
```

Expected:

```text
Non-zero curl exit status
TLS validation error
```

---

# 67. Anti-pattern

Do not change test to:

```bash
curl -k https://badcert.lab.example/
```

then write:

```text
PASS
```

That only proves:

```text
Application responds when certificate validation is disabled.
```

It does not prove TLS trust is correct.

Syllabus specifically prohibits using bypass validation as a “pass”. Infrastructure_Operations_14_Da…

---

# 68. Valid diagnostic use of insecure test

In a controlled investigation, `-k` might answer:

> Nếu bỏ certificate verification thì application layer có trả response không?

That can isolate:

```text
Certificate validation failure
```

from:

```text
Application failure
```

Nhưng result phải được recorded:

```text
Diagnostic-only test
TLS verification bypassed
NOT a production acceptance pass
```

---

# PHASE 11 — BIG-IP iCONTROL REST API

# 69. API scope của Day 9

Theo syllabus, lab yêu cầu **authenticated, read-only API requests**.

Không cần modify BIG-IP qua REST trong Day 9.

Các API resource chính:

```text
/mgmt/tm/ltm/virtual
/mgmt/tm/ltm/pool
/mgmt/tm/ltm/pool/<pool>/members
```

F5 API reference hiện xác nhận collection endpoints cho Virtual Servers, Pools và Pool Members đều hỗ trợ `GET`; object endpoints còn có mutating methods, vì vậy account/scripting discipline rất quan trọng. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/APIRef_tm_ltm_pool_members.html?utm_source=chatgpt.com)

---

# 70. Authentication approach

BIG-IP supports Basic Authentication và token-based authentication; F5 training material lưu ý token-based authentication thường giảm việc gửi credentials lặp lại trong mỗi request. [F5 Cloud Docs](https://clouddocs.f5.com/training/community/automation/html/class01/module1/lab2.html?utm_source=chatgpt.com)

Trong lab, ưu tiên approved token workflow nếu môi trường hỗ trợ.

Không hard-code credentials vào script.

---

# 71. Environment variables

Ví dụ:

```bash
export BIGIP_HOST="bigip-lab.example"
export BIGIP_TOKEN="<TOKEN_FROM_APPROVED_AUTH_FLOW>"
export CA_FILE="./bigip-mgmt-ca.pem"
```

Không commit:

```text
BIGIP_TOKEN
password
client secret
```

vào Git/evidence.

---

# 72. Lab 11.1 — GET Virtual Servers

```bash
curl \
  --cacert "$CA_FILE" \
  --connect-timeout 5 \
  --max-time 10 \
  -sS \
  -H "X-F5-Auth-Token: ${BIGIP_TOKEN}" \
  "https://${BIGIP_HOST}/mgmt/tm/ltm/virtual"
```

Expected:

```json
{
  "items": [
    ...
  ]
}
```

Không cần lưu toàn bộ response nếu chứa unrelated environment information.

Use minimal evidence fields.

---

# 73. Pretty-print JSON

Nếu có `jq`:

```bash
curl ... |
jq .
```

Filter selected fields:

```bash
curl ... |
jq '.items[] | {name,destination,pool}'
```

Expected conceptual:

```json
{
  "name": "VS_APP_HTTPS",
  "destination": "/Common/172.16.10.50:443",
  "pool": "/Common/POOL_APP_HTTPS"
}
```

F5 defines `/mgmt/tm/ltm/virtual` as the Virtual Server configuration collection. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/APIRef_tm_ltm_virtual.html?utm_source=chatgpt.com)

---

# 74. Lab 11.2 — GET Pools

```bash
curl \
  --cacert "$CA_FILE" \
  --connect-timeout 5 \
  --max-time 10 \
  -sS \
  -H "X-F5-Auth-Token: ${BIGIP_TOKEN}" \
  "https://${BIGIP_HOST}/mgmt/tm/ltm/pool"
```

Filter:

```bash
... | jq '.items[] | {name,monitor,loadBalancingMode}'
```

---

# 75. Lab 11.3 — GET specific Pool Members

BIG-IP iControl paths encode partition/name in the resource ID format used by API.

Conceptual example:

```bash
curl \
  --cacert "$CA_FILE" \
  --connect-timeout 5 \
  --max-time 10 \
  -sS \
  -H "X-F5-Auth-Token: ${BIGIP_TOKEN}" \
  "https://${BIGIP_HOST}/mgmt/tm/ltm/pool/~Common~POOL_APP_HTTPS/members"
```

F5 documents this pool-member collection structure under `/mgmt/tm/ltm/pool/~resource id/members`. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/APIRef_tm_ltm_pool_members.html?utm_source=chatgpt.com)

Output fields can include information such as:

```text
name
address
state
session
ratio
```

according to API/version.

---

# 76. HTTP status capture

Do not rely only on response body.

Example:

```bash
HTTP_CODE=$(
  curl \
    --cacert "$CA_FILE" \
    --connect-timeout 5 \
    --max-time 10 \
    -sS \
    -o response.json \
    -w '%{http_code}' \
    -H "X-F5-Auth-Token: ${BIGIP_TOKEN}" \
    "https://${BIGIP_HOST}/mgmt/tm/ltm/virtual"
)

CURL_RC=$?
```

Now you have:

```text
CURL_RC
HTTP_CODE
response.json
```

Three separate evidence layers.

---

# 77. Why tool exit code matters

Scenario:

```text
TLS verification failed
```

Could yield:

```text
curl RC != 0
HTTP status unavailable
```

Scenario:

```text
API returns 403
```

could yield:

```text
curl networking/TLS succeeded
HTTP = 403
```

Thus:

```text
Tool success
≠
API operation success
```

---

# PHASE 12 — JSON ASSERTIONS

# 78. Basic assertion principle

Suppose expected:

```text
Virtual Server name:
VS_APP_HTTPS

Pool:
POOL_APP_HTTPS
```

Query:

```bash
VS_POOL=$(
  jq -r \
    '.items[] |
     select(.name=="VS_APP_HTTPS") |
     .pool' \
    response.json
)
```

Check:

```bash
if [ "$VS_POOL" = "/Common/POOL_APP_HTTPS" ]; then
    echo "PASS"
else
    echo "FAIL"
    exit 1
fi
```

Now failure becomes machine-observable.

---

# 79. Assertion must fail closed

Bad:

```bash
jq ... || true
echo PASS
```

Good:

```bash
if ! jq -e '...' response.json >/dev/null; then
    echo "ASSERTION FAILED"
    exit 1
fi
```

The syllabus specifically requires failed assertions to yield an observable non-success result. Infrastructure_Operations_14_Da…

---

# 80. Full read-only API check example

```bash
#!/usr/bin/env bash

set -u

BIGIP_HOST="${BIGIP_HOST:?BIGIP_HOST is required}"
BIGIP_TOKEN="${BIGIP_TOKEN:?BIGIP_TOKEN is required}"
CA_FILE="${CA_FILE:?CA_FILE is required}"

VS_NAME="VS_APP_HTTPS"
EXPECTED_POOL="/Common/POOL_APP_HTTPS"

TMP_FILE="$(mktemp)"
trap 'rm -f "$TMP_FILE"' EXIT

HTTP_CODE="$(
    curl \
      --cacert "$CA_FILE" \
      --connect-timeout 5 \
      --max-time 10 \
      -sS \
      -o "$TMP_FILE" \
      -w '%{http_code}' \
      -H "X-F5-Auth-Token: ${BIGIP_TOKEN}" \
      "https://${BIGIP_HOST}/mgmt/tm/ltm/virtual"
)"
CURL_RC=$?

if [ "$CURL_RC" -ne 0 ]; then
    echo "FAIL: curl exit code=$CURL_RC"
    exit 10
fi

if [ "$HTTP_CODE" != "200" ]; then
    echo "FAIL: HTTP status=$HTTP_CODE"
    exit 20
fi

if ! jq -e . "$TMP_FILE" >/dev/null 2>&1; then
    echo "FAIL: malformed JSON"
    exit 30
fi

ACTUAL_POOL="$(
    jq -r \
      --arg vs "$VS_NAME" \
      '.items[] | select(.name==$vs) | .pool' \
      "$TMP_FILE"
)"

if [ "$ACTUAL_POOL" != "$EXPECTED_POOL" ]; then
    echo "FAIL: expected pool=$EXPECTED_POOL actual=$ACTUAL_POOL"
    exit 40
fi

echo "PASS: virtual=$VS_NAME pool=$ACTUAL_POOL"
exit 0
```

---

# 81. Script design review

Script trên phân biệt failure classes:

```text
10
→ transport/TLS/cURL failure

20
→ HTTP/API status failure

30
→ malformed JSON

40
→ business assertion mismatch

0
→ PASS
```

Đây là operational automation tốt hơn:

```text
"curl returned something, therefore success"
```

---

# 82. Secret handling caveat

Script không `echo` token.

Không chạy:

```bash
set -x
```

khi command line chứa secret header.

Không submit shell history chứa:

```text
X-F5-Auth-Token: <real token>
```

Nếu evidence cần show request:

```text
X-F5-Auth-Token: <REDACTED>
```

---

# PHASE 13 — API NEGATIVE TESTS

# 83. Negative Test A — 401

Use instructor-provided invalid/expired fixture.

Expected:

```text
HTTP 401
Assertion FAIL
```

Evidence:

```text
CURL_RC
HTTP status
Sanitized response
Timestamp
```

Interpretation:

```text
Transport reachable
TLS works
API service reachable
Authentication failed
```

Do not escalate privilege.

---

# 84. Negative Test B — 403

Use valid identity with intentionally insufficient authorization.

Expected:

```text
HTTP 403
```

Interpretation:

```text
Identity authenticated or recognized
Operation forbidden by authorization/scope
```

Action:

```text
Verify RBAC/request scope
```

Not:

```text
Use administrator account.
```

---

# 85. Negative Test C — Timeout

Point request at prepared timeout fixture or network blackhole within lab.

Example:

```bash
curl \
  --connect-timeout 2 \
  --max-time 4 \
  ...
```

Expected:

```text
non-zero curl exit
bounded completion
```

Acceptance:

Script must not hang indefinitely.

---

# 86. Negative Test D — Malformed JSON

Prepared mock endpoint returns:

```html
<html>
  error
</html>
```

HTTP may even be:

```text
200
```

JSON assertion must fail.

Correct:

```text
HTTP 200
JSON parse FAIL
Overall FAIL
```

Not:

```text
HTTP 200 = PASS
```

---

# 87. Negative Test E — Server error

Fixture returns:

```text
HTTP 500
```

Your script should output:

```text
API HTTP failure
```

Do not retry indefinitely.

For read-only GET, limited retry could be acceptable if runbook permits, but must be bounded.

For mutating operations, syllabus explicitly says **do not blindly retry changes**. Infrastructure_Operations_14_Da…

---

# PHASE 14 — INTEGRATED TROUBLESHOOTING LAB

# 88. Scenario

User ticket:

```text
INC-LAB-D09-001

Reported:
Users cannot access https://app.lab.example.

Start:
14:30 ICT

Scope:
Multiple users

Expected:
HTTPS application should return HTTP 200.
```

Bạn không được biết fault nằm ở đâu.

---

# 89. Step 1 — Client-side baseline

Run:

```bash
date -Ins

getent hosts app.lab.example
```

hoặc appropriate OS DNS utility.

Then:

```bash
curl \
  --cacert ./lab-ca.pem \
  --connect-timeout 5 \
  --max-time 10 \
  -v \
  https://app.lab.example/
```

Record exact symptom.

---

# 90. Possible symptom classification

### DNS error

```text
Could not resolve host
```

Stop.

Do not troubleshoot FortiGate policy yet.

### TCP timeout

Proceed to:

```text
Network
Route
FortiGate
Virtual Server listener
```

### TLS certificate error

Proceed to:

```text
Certificate
Chain
Hostname
SNI
Client SSL
```

### HTTP 503

Network/TLS likely progressed much further.

Proceed:

```text
Virtual Server
Pool
Pool Members
Health Monitor
Backend
```

### HTTP 500

Investigate application/backend rather than immediately firewall.

---

# 91. Step 2 — FortiGate route

```bash
get router info routing-table all
```

Record:

```text
Expected route exists?
Correct egress?
```

If fail:

```text
First failing boundary = routing
```

Do not move deeper without documenting.

---

# 92. Step 3 — Firewall policy

Inspect:

```bash
show firewall policy
```

Compare source/destination/service/interfaces/order.

If mismatch:

```text
First failing boundary = policy classification
```

---

# 93. Step 4 — Session

Generate new client request.

Then:

```bash
diagnose sys session filter clear
diagnose sys session filter src 10.10.10.50
diagnose sys session filter dst 172.16.10.50
diagnose sys session filter dport 443
diagnose sys session list
```

Interpret:

```text
No session
```

versus:

```text
Session created with expected policy
```

---

# 94. Step 5 — Packet evidence if needed

If previous state inconclusive:

```bash
diagnose sniffer packet any \
'tcp and host 10.10.10.50 and host 172.16.10.50 and port 443' \
4 20 l
```

Only then bounded debug if justified.

---

# 95. Step 6 — BIG-IP Virtual Server

```bash
tmsh list ltm virtual VS_APP_HTTPS
tmsh show ltm virtual VS_APP_HTTPS
```

Questions:

```text
Correct destination?
Enabled?
Expected Pool?
Counters increasing?
```

If FortiGate forwards successfully but VS counters do not move:

```text
investigate path between FortiGate and BIG-IP
```

---

# 96. Step 7 — Pool

```bash
tmsh list ltm pool POOL_APP_HTTPS
tmsh show ltm pool POOL_APP_HTTPS
```

If:

```text
both members DOWN
```

narrow fault domain to:

```text
BIG-IP → backend / health/application
```

---

# 97. Step 8 — Backend direct test

Backend01:

```bash
curl \
  --resolve app.lab.example:8443:10.20.30.11 \
  --cacert ./backend-ca.pem \
  --connect-timeout 5 \
  --max-time 10 \
  -i \
  https://app.lab.example:8443/health
```

Backend02 similar.

Compare with Health Monitor requirement.

---

# 98. Step 9 — TLS

Client-side:

```bash
openssl s_client \
  -connect 172.16.10.50:443 \
  -servername app.lab.example
```

If server-side SSL relevant, perform equivalent authorized backend test.

Determine:

```text
Client-side or server-side?
Trust?
Chain?
Hostname?
Expiry?
SNI?
```

---

# 99. Step 10 — API independent validation

Use read-only API:

```bash
GET /mgmt/tm/ltm/virtual
GET /mgmt/tm/ltm/pool
GET /mgmt/tm/ltm/pool/~Common~POOL_APP_HTTPS/members
```

Compare API data with `tmsh`.

This gives cross-interface consistency check.

---

# 100. First-failing-boundary table

Final incident worksheet:

| Boundary          | Result        |
| ----------------- | ------------- |
| DNS               | PASS          |
| Client TCP        | PASS/FAIL     |
| FortiGate route   | PASS/FAIL     |
| FortiGate policy  | PASS/FAIL     |
| FortiGate session | PASS/FAIL     |
| BIG-IP VS         | PASS/FAIL     |
| Client TLS        | PASS/FAIL     |
| Pool              | PASS/FAIL     |
| Member health     | PASS/FAIL     |
| Server-side TLS   | PASS/FAIL/N/A |
| HTTP health       | PASS/FAIL     |
| Application       | PASS/FAIL     |

Stop root-cause search at **first meaningful failure**, then prove causality.

---

# PHASE 15 — APPROVED CONFIGURATION CHANGE

# 101. Change gate

Before any fix:

```text
Incident/ticket exists?
Change approval exists if required?
Current config captured?
Rollback defined?
Risk known?
Affected traffic known?
Maintenance window needed?
HA status checked?
```

If any mandatory control missing:

```text
STOP / ESCALATE
```

---

# 102. FortiGate lab-only policy template

Nếu Assignment yêu cầu actual configuration, một isolated lab policy có thể conceptually được tạo như sau.

**Không copy trực tiếp vào production.**

```bash
config firewall address
    edit "LAB_CLIENT"
        set subnet 10.10.10.50 255.255.255.255
    next

    edit "LAB_BIGIP_VIP"
        set subnet 172.16.10.50 255.255.255.255
    next
end
```

Custom service:

```bash
config firewall service custom
    edit "LAB_HTTPS"
        set tcp-portrange 443
    next
end
```

Policy template:

```bash
config firewall policy
    edit 0
        set name "LAB_CLIENT_TO_BIGIP_HTTPS"
        set srcintf "port2"
        set dstintf "port3"
        set srcaddr "LAB_CLIENT"
        set dstaddr "LAB_BIGIP_VIP"
        set action accept
        set schedule "always"
        set service "LAB_HTTPS"
        set logtraffic all
        set nat disable
    next
end
```

Caveats:

```text
Interface names vary.
Zones may replace interfaces.
VDOM context may differ.
NAT design may require NAT enabled.
Central NAT changes behavior.
Security profiles may be mandatory.
Policy order matters.
```

Therefore exact production syntax must follow approved FortiOS-version runbook.

---

# 103. Verify newly created policy

Immediately:

```bash
show firewall policy
```

Then generate controlled request.

Check:

```bash
diagnose sys session filter clear
diagnose sys session filter src 10.10.10.50
diagnose sys session filter dst 172.16.10.50
diagnose sys session list
```

Verify traffic log.

Do not assume CLI returned no error = service fixed.

---

# 104. FortiGate rollback

Rollback method should be defined **before change**.

For a newly created lab policy/object, rollback may mean:

```text
Delete only objects created for change
Restore previous object values
Restore policy order
Verify resulting traffic
```

Do not use whole-device restore when a small targeted backout is sufficient.

Whole configuration restore can have much larger impact.

---

# PHASE 16 — BIG-IP LAB CONFIGURATION

# 105. Configuration sequence

In isolated lab, dependency order normally follows:

```text
Monitor
→ Pool
→ Pool Members
→ SSL Profiles
→ Virtual Server
```

or variations based on pre-existing objects.

For Day 9, if environment is preconfigured, inspection is sufficient where Assignment permits.

F5 current implementation documentation still follows a model where Pools contain backend members and Virtual Servers reference Pools/profiles. [techdocs.f5.com](https://techdocs.f5.com/en-us/bigip-17-5-1/big-ip-system-ssl-administration/managing-client-side-http-traffic-using-a-ca-signed-rsa-certificate.html?utm_source=chatgpt.com)

---

# 106. Example Pool creation — lab only

Conceptual `tmsh`:

```bash
tmsh create ltm pool POOL_APP_HTTPS \
    members add {
        10.20.30.11:8443 {
            address 10.20.30.11
        }
        10.20.30.12:8443 {
            address 10.20.30.12
        }
    }
```

Exact monitor attachment should follow the deployed monitor design.

Before running:

```bash
tmsh list ltm pool POOL_APP_HTTPS
```

to make sure object does not already exist.

---

# 107. Example Virtual Server — lab only

Conceptual:

```bash
tmsh create ltm virtual VS_APP_HTTPS \
    destination 172.16.10.50:443 \
    ip-protocol tcp \
    pool POOL_APP_HTTPS
```

Real HTTPS configuration generally additionally requires appropriate:

```text
TCP profile
HTTP profile
Client SSL profile
Possibly Server SSL profile
SNAT
VLAN restriction
Persistence
```

Do not create incomplete production Virtual Servers from a training snippet.

---

# 108. BIG-IP configuration persistence

On BIG-IP, running config and saved config are operationally important distinctions.

Changes may require:

```bash
tmsh save sys config
```

according to approved runbook.

Do not save experimental/broken configuration before validation unless the runbook specifically calls for it.

Rollback strategy must account for:

```text
current running config
saved config
HA/config-sync context
```

---

# 109. BIG-IP HA caveat

If device is part of a Device Service Cluster, before change verify:

```text
Active/Standby
Config Sync
Which device you are modifying
Traffic group state
```

Day 9 does not require HA configuration, but operational awareness is mandatory.

Do not assume:

```text
"GUI opened on device"
```

means:

```text
"this is the correct device to change."
```

---

# PHASE 17 — BIG-IP ROLLBACK

# 110. Minimal rollback principle

Suppose change:

```text
Changed Pool monitor:
OLD_MONITOR → MON_APP_HTTPS
```

Rollback:

```text
MON_APP_HTTPS → OLD_MONITOR
```

not:

```text
restore whole BIG-IP
```

Suppose:

```text
new Pool Member created
```

rollback may be:

```text
remove only new member
```

after verifying dependencies.

---

# 111. Post-rollback verification

Rollback is not complete because command executed.

Verify:

```bash
tmsh list ltm virtual VS_APP_HTTPS
tmsh list ltm pool POOL_APP_HTTPS
tmsh show ltm pool POOL_APP_HTTPS
```

Then run application test.

Rollback acceptance:

```text
Config == pre-change expected state
Runtime health == expected
Application behavior == expected
No temporary test artifacts remain
```

---

# PHASE 18 — AUTOMATED CONNECTIVITY CHECKS

# 112. End-to-end HTTPS check

A simple bounded check:

```bash
#!/usr/bin/env bash

URL="https://app.lab.example/"
CA="./lab-ca.pem"

HTTP_CODE=$(
    curl \
      --cacert "$CA" \
      --connect-timeout 5 \
      --max-time 10 \
      -sS \
      -o /dev/null \
      -w '%{http_code}' \
      "$URL"
)
RC=$?

if [ "$RC" -ne 0 ]; then
    echo "FAIL transport_or_tls curl_rc=$RC"
    exit 10
fi

if [ "$HTTP_CODE" != "200" ]; then
    echo "FAIL http_status=$HTTP_CODE"
    exit 20
fi

echo "PASS http_status=200"
exit 0
```

---

# 113. Why this is better than ping

`ping` validates at most:

```text
ICMP path
```

The script validates:

```text
DNS
TCP
TLS
HTTP
```

assuming DNS is used and certificate verification remains enabled.

Thus it represents application path more accurately.

---

# 114. Combined app + BIG-IP API check

An advanced Day 9 validation could produce:

```text
CHECK-1 Application HTTPS
PASS

CHECK-2 BIG-IP REST
PASS

CHECK-3 Virtual Server mapping
PASS

CHECK-4 Pool Member count
PASS

CHECK-5 All required members healthy
PASS
```

Only then:

```text
OVERALL PASS
```

This introduces Day 12 automation mindset early without exceeding Day 9 scope.

---

# PHASE 19 — FAILURE INJECTION MATRIX

Syllabus asks controlled failures. Một useful Day 9 set gồm:

| Fault                          | Expected symptom                              | Primary evidence             |
| ------------------------------ | --------------------------------------------- | ---------------------------- |
| Wrong FortiGate service        | TCP/443 not matching policy                   | Policy + session/debug       |
| Wrong policy order             | Wrong policy hit/deny                         | Logs + policy IDs            |
| Wrong NAT                      | Unexpected source/return failure              | Session NAT + capture        |
| BIG-IP member unhealthy        | Pool member DOWN                              | `tmsh show ltm pool`         |
| Health Monitor wrong Host/path | Member DOWN though server otherwise reachable | Monitor config + direct curl |
| Expired/untrusted cert         | TLS verification fail                         | cURL/OpenSSL                 |
| Wrong SNI                      | Wrong certificate/handshake behavior          | OpenSSL `-servername`        |
| API invalid credential         | 401                                           | HTTP status                  |
| API insufficient role          | 403                                           | HTTP status                  |
| API timeout                    | bounded nonzero result                        | cURL exit                    |
| malformed JSON                 | JSON assertion fail                           | parser exit                  |

---

# 120. Troubleshooting checkpoint — FortiGate

Khi bạn không biết đi tiếp thế nào, sử dụng flow:

```text
1. Correct client/source?
        ↓
2. Correct destination/VIP?
        ↓
3. Interface state/config?
        ↓
4. Route selected?
        ↓
5. Objects correct?
        ↓
6. Service/port correct?
        ↓
7. Policy match/order?
        ↓
8. NAT expected?
        ↓
9. Session created?
        ↓
10. Traffic/security log?
        ↓
11. Bounded packet capture?
        ↓
12. Bounded debug flow?
        ↓
13. Approved correction?
```

Không đi thẳng từ 1 → 13.

---

# 121. Troubleshooting checkpoint — BIG-IP

```text
1. Virtual Server correct IP/port?
        ↓
2. VS enabled/available?
        ↓
3. Correct profiles?
        ↓
4. Correct default Pool?
        ↓
5. Pool Members correct IP/ports?
        ↓
6. Members available?
        ↓
7. Monitor correct?
        ↓
8. Backend network reachable?
        ↓
9. Backend TCP reachable?
        ↓
10. Server-side TLS healthy?
        ↓
11. HTTP health endpoint healthy?
        ↓
12. Persistence/SNAT relevant?
        ↓
13. Approved correction?
```

---

# 122. Troubleshooting checkpoint — TLS

```text
TCP reachable?
    |
    +-- NO → Network/firewall/listener
    |
    +-- YES
          ↓
TLS handshake?
    |
    +-- NO → protocol/cipher/SNI/profile
    |
    +-- YES
          ↓
Certificate trusted?
    |
    +-- NO → CA/chain
    |
    +-- YES
          ↓
Hostname matches?
    |
    +-- NO → SAN/SNI/certificate mapping
    |
    +-- YES
          ↓
Not expired/not-before valid?
    |
    +-- NO → certificate/time
    |
    +-- YES
          ↓
HTTP investigation
```

---

# 123. Troubleshooting checkpoint — API

```text
Can resolve/connect?
      ↓
TLS verification?
      ↓
curl exit code 0?
      ↓
HTTP status?
      ↓
Authentication?
      ↓
Authorization?
      ↓
JSON valid?
      ↓
Required fields exist?
      ↓
Expected values match?
```

This sequence prevents confusing:

```text
401
```

with:

```text
TLS failure
```

or:

```text
JSON parsing failure.
```

---

# PHASE 20 — EVIDENCE PACKAGE

# 124. Mandatory FortiGate evidence

Submit at minimum:

```text
EV-FG-01 System/version
EV-FG-02 Interfaces
EV-FG-03 Routing
EV-FG-04 Address objects
EV-FG-05 Service objects
EV-FG-06 Firewall policy/order
EV-FG-07 NAT configuration/state
EV-FG-08 Allowed-flow session
EV-FG-09 Denied-flow evidence
EV-FG-10 Traffic logs
EV-FG-11 Packet/debug output if used
EV-FG-12 Corrective action
EV-FG-13 Post-change verification
EV-FG-14 Rollback evidence
EV-FG-15 Debug cleanup
```

---

# 125. Mandatory BIG-IP evidence

```text
EV-F5-01 Product/version
EV-F5-02 Virtual Server config
EV-F5-03 Virtual Server runtime
EV-F5-04 Pool config
EV-F5-05 Pool Member states
EV-F5-06 Health Monitor config
EV-F5-07 Direct backend validation
EV-F5-08 Load-balancing result
EV-F5-09 Client-side TLS
EV-F5-10 Server-side TLS if applicable
EV-F5-11 Certificate validation
EV-F5-12 Corrective action
EV-F5-13 Post-change verification
EV-F5-14 Rollback
```

---

# 126. Mandatory API evidence

```text
EV-API-01 Endpoint tested
EV-API-02 Authentication type, secret redacted
EV-API-03 TLS validation result
EV-API-04 cURL exit status
EV-API-05 HTTP status
EV-API-06 Selected JSON fields
EV-API-07 Assertion result
EV-API-08 401 testcase
EV-API-09 403 testcase
EV-API-10 timeout testcase
EV-API-11 malformed JSON testcase
```

Syllabus specifically requires expected/actual results, HTTP and tool exit status, selected non-secret fields và cleanup evidence. Infrastructure_Operations_14_Da…

---

# 127. Evidence quality — Bad example

```text
F5 was down.
Fixed monitor.
Everything OK.
```

Không acceptable.

---

# 128. Evidence quality — Good example

```text
Ticket:
INC-LAB-D09-001

Expected:
VS_APP_HTTPS should serve HTTPS using two healthy members.

Observed:
At 2026-10-21 14:33:10 +07,
Client curl returned HTTP 503.

FortiGate:
Route to 172.16.10.50 present.
Policy LAB_CLIENT_TO_BIGIP_HTTPS matched.
Session established.

BIG-IP:
VS_APP_HTTPS available.
POOL_APP_HTTPS:
10.20.30.11:8443 UP
10.20.30.12:8443 DOWN

Monitor:
MON_APP_HTTPS checks /health.

Direct Backend02 test:
HTTP 500.

Root cause:
Backend02 health endpoint failure.

Action:
No BIG-IP configuration changed.
Escalated to application team.

Verification:
Backend01 remained healthy and serving traffic.

Evidence IDs:
EV-FG-...
EV-F5-...
```

Đây là Infrastructure Operations quality.

---

# PHASE 21 — ESCALATION BOUNDARIES

# 129. L1/L2 có thể làm gì?

Tùy RBAC/runbook, các task thường phù hợp:

| Activity                     | Typical handling    |
| ---------------------------- | ------------------- |
| Run client connectivity test | L1/L2               |
| Inspect route                | L1/L2               |
| Inspect firewall policy      | L1/L2               |
| Inspect session              | L1/L2               |
| Read traffic logs            | L1/L2               |
| Narrow packet capture        | L2                  |
| Narrow bounded debug         | L2/runbook          |
| Inspect Virtual Server       | L1/L2               |
| Inspect Pool/Member          | L1/L2               |
| Validate certificate         | L1/L2               |
| Read-only REST GET           | L1/L2 if authorized |
| Parse JSON                   | L1/L2               |
| Application backend test     | L2                  |

---

# 130. Escalate/change approval required

Examples:

```text
Production firewall policy modification
Production NAT change
Production route change
Security profile bypass
Broad session clearing
HA failover
Virtual Server modification
Pool member removal
Monitor behavior modification
Certificate/key replacement
Client SSL/Server SSL change
Private-key operation
BIG-IP HA/config-sync issue
Suspected vendor defect
```

Do not cross boundary because:

> “Fix looks easy.”

---

# PHASE 22 — CLEANUP

# 131. FortiGate cleanup checklist

Ensure:

```text
Debug disabled
Debug filters reset
Sniffer stopped
Temporary test policy removed if required
Temporary objects removed if required
Policy order restored
NAT restored
No unexpected test sessions/actions
Configuration state verified
```

---

# 132. BIG-IP cleanup checklist

Ensure:

```text
Test member restored
Monitor restored
Temporary test objects removed
Temporary profiles removed
No test certificate/private key left unintentionally
Expected Pool health restored
VS status verified
Configuration sync checked if relevant
```

---

# 133. API cleanup

Ensure:

```text
Temporary token expired/revoked according to workflow
Token absent from transcript
Token absent from shell script
Token absent from shell history if environment permits controlled cleanup
Temporary JSON responses sanitized before submission
Temporary files deleted if sensitive
```

---

# 134. Rollback validation

A rollback result should be:

```text
ROLLBACK EXECUTED
```

plus:

```text
ROLLBACK VERIFIED
```

Those are different.

Example:

```text
Action:
Policy NAT restored to previous setting.

Verification:
Fresh session shows expected source tuple.
HTTPS endpoint returns HTTP 200.
No additional config drift identified.
```

---

# PHASE 23 — PRACTICAL ACCEPTANCE TEST

Để được coi hoàn thành Module Thực hành Day 9, bạn phải demonstrate toàn bộ chain sau.

### FortiGate

Bạn phải tự làm được:

```text
Identify FortiOS version
Identify interfaces
Verify route
Verify address/service objects
Verify firewall policy
Understand policy order
Identify NAT model
Filter session
Correlate traffic log
Perform allow test
Perform deny test
Use bounded packet capture/debug
Diagnose prepared policy/NAT fault
Correct within authorized lab
Verify
Rollback
Collect evidence
```

### BIG-IP

Bạn phải:

```text
Identify BIG-IP version
Locate Virtual Server
Identify listener
Locate Pool
Locate Pool Members
Identify Health Monitor
Determine Member health
Validate backend directly
Observe load balancing
Identify persistence/SNAT considerations
Inspect Client SSL
Inspect Server SSL if applicable
Validate certificate chain/hostname/expiry
Troubleshoot a Health Monitor failure
Troubleshoot TLS failure
```

### API

Bạn phải:

```text
Authenticate safely
Issue read-only GET
Validate TLS
Capture cURL exit code
Capture HTTP status
Parse JSON
Select non-secret fields
Build assertion
Return nonzero result on failed assertion
Handle 401
Handle 403
Handle timeout
Handle malformed JSON
Protect secrets
```

---

# 135. Practical interview-level scenario

Một interviewer hoặc senior engineer có thể hỏi:

> User không truy cập được application qua FortiGate và BIG-IP. Bạn làm gì?

Một answer tốt không phải:

> “Tôi kiểm tra firewall và restart F5.”

Một operational answer tốt:

```text
1. Xác định scope, timestamp và exact hostname/port.
2. Reproduce bằng bounded cURL.
3. Phân loại DNS/TCP/TLS/HTTP failure.
4. Verify client route nếu relevant.
5. Trên FortiGate:
   interface → route → policy → NAT → fresh session → traffic logs.
6. Chỉ dùng filtered packet capture/debug nếu evidence chưa đủ.
7. Nếu traffic tới BIG-IP:
   Virtual Server → counters → profiles → Pool → Pool Members → Monitor.
8. Nếu Member DOWN:
   direct backend TCP/TLS/HTTP health test.
9. Nếu TLS issue:
   trust → chain → hostname → expiry → SNI, tách client-side/server-side.
10. Dùng read-only iControl REST để cross-check VS/Pool/Member state.
11. Xác định first failing boundary.
12. Chỉ thực hiện minimal approved change.
13. Re-test original user path.
14. Rollback nếu acceptance fail.
15. Lưu evidence và escalate nếu root cause ngoài scope.
```

Đó chính là tư duy Day 9.

---

# 136. Day 9 Practical Cheat Sheet

| Need                 | Check                                           |
| -------------------- | ----------------------------------------------- |
| FortiGate version    | `get system status`                             |
| Interface config     | `show system interface`                         |
| Route                | `get router info routing-table all`             |
| Address objects      | `show firewall address`                         |
| Service objects      | `show firewall service custom`                  |
| Firewall policy      | `show firewall policy`                          |
| Sessions             | `diagnose sys session list`                     |
| Clear session filter | `diagnose sys session filter clear`             |
| Source filter        | `diagnose sys session filter src <IP>`          |
| Destination filter   | `diagnose sys session filter dst <IP>`          |
| Port filter          | `diagnose sys session filter dport <PORT>`      |
| Packet capture       | `diagnose sniffer packet ...`                   |
| Reset debug          | `diagnose debug reset`                          |
| Flow filters         | `diagnose debug flow filter ...`                |
| Start bounded flow   | `diagnose debug flow trace start <N>`           |
| Stop flow            | `diagnose debug flow trace stop`                |
| Disable debug        | `diagnose debug disable`                        |
| BIG-IP VS config     | `tmsh list ltm virtual <VS>`                    |
| BIG-IP VS runtime    | `tmsh show ltm virtual <VS>`                    |
| Pool config          | `tmsh list ltm pool <POOL>`                     |
| Pool state           | `tmsh show ltm pool <POOL>`                     |
| Pool members         | `tmsh show ltm pool <POOL> members`             |
| Client SSL           | `tmsh list ltm profile client-ssl <PROFILE>`    |
| Server SSL           | `tmsh list ltm profile server-ssl <PROFILE>`    |
| HTTPS client test    | `curl --cacert ...`                             |
| SNI/TLS test         | `openssl s_client -connect ... -servername ...` |
| REST Virtual Servers | `GET /mgmt/tm/ltm/virtual`                      |
| REST Pools           | `GET /mgmt/tm/ltm/pool`                         |
| REST Pool Members    | `GET /mgmt/tm/ltm/pool/~Common~<POOL>/members`  |

FortiOS 7.6.x official documentation confirms the current session-filter, packet-sniffer and bounded debug-flow command families used above. [Fortinet Documentation](https://docs.fortinet.com/document/fortigate/7.6.6/cli-reference/235530229/diagnose-sys?utm_source=chatgpt.com) F5's current API references confirm the Virtual Server, Pool and Pool Member REST resources used in the lab, while current BIG-IP 17.5.1 documentation confirms the Client SSL model used for HTTPS termination. [F5 Cloud Docs](https://clouddocs.f5.com/api/icontrol-rest/APIRef_tm_ltm_pool_members.html?utm_source=chatgpt.com)

---

# 137. Module Thực hành Day 9 — Definition of Done

Bạn chỉ nên đánh dấu **Day 9 Practical Complete** khi có thể chứng minh end-to-end:

```text
CLIENT
   |
   | Test + timestamp
   v
FORTIGATE
   |
   | Interface verified
   | Route verified
   | Objects verified
   | Policy verified
   | NAT verified
   | Session verified
   | Log verified
   | Debug bounded
   v
BIG-IP
   |
   | Virtual Server verified
   | TLS verified
   | Pool verified
   | Members verified
   | Monitor verified
   | Load balancing verified
   v
BACKEND
   |
   | TCP/TLS verified
   | HTTP health verified
   v
APPLICATION
```

và song song:

```text
BIG-IP
   |
iControl REST
   |
TLS verification
   |
Authentication
   |
HTTP status
   |
JSON parse
   |
Assertions
   |
Explicit PASS / FAIL
```

Cuối cùng, bạn phải có:

```text
Baseline
+ fault evidence
+ root-cause reasoning
+ approved corrective action
+ verification
+ rollback evidence
+ sanitized logs
+ timestamps
+ ticket reference
+ cleanup evidence
```

Điểm quan trọng nhất của **Module Thực hành Day 9** không phải là nhớ nhiều `FortiGate CLI` hay `tmsh` commands nhất. Kỹ năng cần đạt là có thể nhận một sự cố application connectivity phức tạp và **chứng minh từng boundary theo thứ tự**, từ firewall đến load balancer đến TLS đến backend, dùng read-only verification trước, diagnostic có giới hạn sau, configuration change cuối cùng; đồng thời không expose secrets, không bypass TLS để tạo false pass, không thay đổi production vượt quyền và luôn giữ khả năng rollback/escalation. Đó chính là operational readiness mà Assignment/Lab và Acceptance/Evidence của syllabus Day 9 yêu cầu. Infrastructure_Operations_14_Da…
