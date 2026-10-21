Dưới đây là **bản đồ kiến thức Day 9** được chia đúng thành hai module **Lý thuyết** và **Thực hành**. Tôi bám sát syllabus và chưa đi sâu tới mức “giảng bài từng command”; mục tiêu của phần này là giúp bạn thấy **toàn bộ phạm vi phải học, mối liên hệ giữa các thành phần, những kỹ năng vận hành phải đạt và evidence phải tạo ra** trước khi chúng ta đào sâu từng module.

Day 9 nằm trong module **Network, Firewall & Load Balancer Operations**, tiếp nối Day 8 về switching/routing. Trọng tâm chuyển từ L2/L3 infrastructure sang hai lớp kiểm soát traffic quan trọng trong enterprise: **FortiGate firewall** và **F5 BIG-IP load balancer**, bao gồm firewall policy/NAT/session troubleshooting, load balancing, health monitoring, SSL/TLS và automation thông qua REST API/cURL. Infrastructure_Operations_14_Da…

# Day 9 — Tổng quan

Có thể hình dung Day 9 theo một data flow điển hình:

**Client → Network → FortiGate → BIG-IP Virtual Server → Pool → Pool Member/Application**

Bạn cần học cách trả lời các câu hỏi vận hành như:

“Packet có đến firewall không?”, “Firewall policy nào xử lý connection?”, “NAT có đúng không?”, “Session có được tạo không?”, “Traffic đã đến BIG-IP chưa?”, “Virtual Server có listener đúng không?”, “Pool có member nào UP?”, “Health monitor đang kiểm tra gì?”, “TLS certificate có hợp lệ không?”, “Client-side SSL và server-side SSL khác nhau ở đâu?”, “API trả về 401, 403, 5xx hay malformed JSON thì phải hiểu như thế nào?”, “Làm sao chứng minh bằng evidence rằng lỗi nằm ở firewall, TLS, load balancer hay application?”

Syllabus yêu cầu cả **configuration understanding**, **read-only verification**, **controlled troubleshooting**, **safe corrective changes**, **rollback**, **API automation**, **SSL/TLS validation** và **audit-quality evidence**. Infrastructure_Operations_14_Da…

---

# Module 1 — Lý thuyết

Module Lý thuyết gồm hai khối lớn: **FortiGate Firewall Operations** và **F5 BIG-IP Load Balancer / SSL / API Operations**.

## 1. Fortinet FortiGate / FortiOS

| Nhóm kiến thức                      | Nội dung cần nắm                                                                                                                                                    |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **FortiGate architecture**          | Vai trò của FortiGate trong enterprise network; data plane/control-management concepts; quan hệ giữa interface, route, policy, NAT, session và security inspection. |
| **FortiOS**                         | FortiOS là operating system của FortiGate; hiểu GUI/CLI administration và sự phụ thuộc vào version/configuration.                                                   |
| **Administrative access**           | Cách administrator truy cập FortiGate; management interface; authentication; privilege/administrator profile awareness; nguyên tắc least privilege.                 |
| **FortiGate CLI**                   | Cấu trúc CLI; phân biệt command dùng để xem trạng thái, diagnostic command và configuration-changing command.                                                       |
| **Interfaces**                      | Physical/logical interface, IP addressing, interface status, administrative state, VLAN/interface association và role trong packet flow.                            |
| **Zones**                           | Nhóm nhiều interface theo logical security boundary; ảnh hưởng đến firewall policy design.                                                                          |
| **Routing**                         | Routing table, connected/static/dynamic route awareness, default route, next hop và route lookup trong troubleshooting.                                             |
| **Address objects**                 | Host/subnet/range objects dùng để tái sử dụng source/destination trong policies.                                                                                    |
| **Service objects**                 | Representation của protocol/port như TCP/443, UDP/53; custom services và service matching.                                                                          |
| **Firewall policies**               | Cấu trúc policy: incoming interface/zone, outgoing interface/zone, source, destination, service, schedule, action, NAT, security profiles, logging.                 |
| **Policy ordering**                 | Vì policy evaluation phụ thuộc thứ tự nên policy đúng nhưng nằm sai vị trí vẫn có thể không bao giờ match.                                                          |
| **NAT**                             | Source NAT/PAT và các NAT concepts cần thiết để hiểu client-to-server traffic; quan hệ giữa policy và translated address.                                           |
| **Security profiles**               | Awareness về các inspection/security profiles được gắn vào policy; hiểu rằng “policy allow” không luôn đồng nghĩa application traffic sẽ thành công.                |
| **Session table**                   | Stateful firewall behavior; session chứa state của connection đã được firewall xử lý; session inspection là nguồn evidence rất quan trọng.                          |
| **VPN awareness**                   | Nhận biết VPN là một possible traffic path/security boundary, nhưng Day 9 không yêu cầu triển khai VPN chuyên sâu.                                                  |
| **Traffic logs**                    | Dùng log để xác định source, destination, service, action, policy ID và các thông tin cần cho investigation.                                                        |
| **Security events**                 | Nhận diện sự kiện security/inspection và phân biệt với lỗi routing hoặc firewall policy.                                                                            |
| **Policy hit analysis**             | Xem policy nào đang được dùng, policy nào không match, phục vụ troubleshooting và policy review.                                                                    |
| **Configuration revision / backup** | Backup configuration, revision awareness, before/after evidence và cơ sở cho rollback.                                                                              |
| **HA awareness**                    | Hiểu High Availability ở mức operational awareness; tránh thực hiện thay đổi hoặc troubleshoot một node mà không biết HA context.                                   |
| **Firewall troubleshooting**        | Xây dựng troubleshooting flow từ interface → route → policy → NAT → session → log → packet/debug evidence.                                                          |

Đây chính là phạm vi FortiGate mà syllabus yêu cầu: architecture, FortiOS, administrative access, CLI, interfaces/zones/routing, objects, policy/order/NAT/security profiles, sessions, VPN awareness, logs, configuration backup/revision, HA và troubleshooting. Infrastructure_Operations_14_Da…

Một nguyên tắc rất quan trọng của Day 9 là không được nhìn firewall theo kiểu “port 443 open hay closed”. Bạn phải suy luận theo **packet-processing chain**. Một connection có thể fail vì interface/down, route sai, source/destination object sai, service không match, wrong policy order, NAT sai, existing stale session, security profile block hoặc vấn đề nằm hoàn toàn phía sau firewall.

---

## 2. F5 BIG-IP architecture và Local Traffic Manager

BIG-IP được học chủ yếu ở góc độ **Local Traffic Manager — LTM**, tức nhận client connection thông qua một **Virtual Server**, lựa chọn backend trong **Pool**, kiểm tra backend thông qua **Health Monitor**, sau đó quản lý TCP/HTTP/TLS behavior.

| Object/Concept            | Ý nghĩa cần hiểu                                                                                         |
| ------------------------- | -------------------------------------------------------------------------------------------------------- |
| **BIG-IP architecture**   | BIG-IP nằm giữa client và application/backend để điều phối traffic.                                      |
| **LTM**                   | Module chịu trách nhiệm local traffic management/load balancing.                                         |
| **Node**                  | Backend IP/address object đại diện server.                                                               |
| **Pool**                  | Nhóm backend destinations mà Virtual Server có thể gửi traffic tới.                                      |
| **Pool Member**           | Kết hợp backend address/node với service port cụ thể.                                                    |
| **Health Monitor**        | Cơ chế probe backend để xác định member UP/DOWN/available/unavailable tùy configuration.                 |
| **Virtual Server**        | Front-end listener nhận client traffic trên destination IP/port cụ thể.                                  |
| **Listener**              | IP/port/protocol mà BIG-IP nhận connection.                                                              |
| **Load-balancing method** | Algorithm lựa chọn pool member như các phương pháp phân phối connection mà environment hỗ trợ/configure. |
| **Persistence**           | Giữ client/session liên kết với backend phù hợp trong những use case cần state affinity.                 |
| **SNAT**                  | Source address translation ở phía BIG-IP nhằm bảo đảm return path hoặc đáp ứng network design.           |
| **TCP profile**           | Điều chỉnh TCP behavior phía client/server.                                                              |
| **HTTP profile**          | Cho phép BIG-IP hiểu/xử lý HTTP traffic thay vì chỉ TCP pass-through.                                    |

Syllabus yêu cầu bạn hiểu relationship giữa các objects này chứ không chỉ nhớ định nghĩa. Infrastructure_Operations_14_Da…

Ví dụ dependency quan trọng:

**Virtual Server**
→ tham chiếu **Pool**  
→ Pool chứa **Pool Members**  
→ Pool/Member được đánh giá bởi **Health Monitor**  
→ Virtual Server có thể gắn **TCP/HTTP/SSL profiles**  
→ traffic có thể được **SNAT**  
→ member selection chịu ảnh hưởng bởi **load-balancing method** và có thể bởi **persistence**.

Vì vậy khi client báo `https://application.example.com` không truy cập được, việc chỉ kiểm tra “Virtual Server exists” hoàn toàn chưa đủ.

---

## 3. SSL/TLS trên BIG-IP

Đây là một trong những phần quan trọng nhất của Day 9.

Bạn phải hiểu hai connection độc lập có thể tồn tại:

**Client → BIG-IP**

và

**BIG-IP → Backend**

Từ đó xuất hiện:

**client-side SSL/TLS** — TLS giữa client và BIG-IP.

**server-side SSL/TLS** — TLS giữa BIG-IP và backend application server.

Syllabus yêu cầu bao phủ **certificate and key management, certificate chains, expiration/renewal, SNI awareness và HTTPS validation**. Infrastructure_Operations_14_Da…

Các kiến thức phải nắm gồm certificate identity, private key relationship, issuer/intermediate/root CA chain, validity period, hostname validation, certificate trust, certificate expiration, certificate renewal/change planning và SNI.

Một operational principle đặc biệt quan trọng là phải phân biệt ít nhất ba loại lỗi:

**Transport/TCP failure → TLS failure → HTTP/application failure.**

Syllabus nhấn mạnh việc không được gom tất cả thành “HTTPS lỗi”. Certificate validation phải kiểm tra ít nhất **trust, chain, hostname và expiry**. Infrastructure_Operations_14_Da…

Ví dụ:

`connection refused` không phải certificate error.

TLS handshake fail chưa chắc HTTP server có vấn đề.

HTTP `503` chưa chắc TLS có vấn đề.

Pool member DOWN do HTTP health monitor chưa chắc backend TCP port bị down.

---

## 4. BIG-IP tmsh

`tmsh` là command-line management interface quan trọng của BIG-IP.

Trong Day 9, bạn cần hiểu cách dùng `tmsh` để inspect các objects như Virtual Server, Pool, Pool Member, monitor, profiles và trạng thái liên quan.

Khi học chi tiết Module Lý thuyết, chúng ta sẽ phân biệt rất rõ:

**read-only inspection**

với

**configuration-changing commands**.

Điều này quan trọng vì syllabus yêu cầu **safe load balancer configuration changes**, không phải tự do sửa production configuration. Infrastructure_Operations_14_Da…

---

## 5. BIG-IP iControl REST API

Day 9 không chỉ yêu cầu GUI/CLI; bạn còn phải tiếp xúc với API.

Các concepts bắt buộc gồm:

| Chủ đề                      | Kiến thức cần đạt                                                                                                   |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| **iControl REST API**       | API interface dùng để query/manage BIG-IP resources.                                                                |
| **HTTP methods**            | Hiểu sự khác biệt operational giữa read-only và mutating operations.                                                |
| **Authentication**          | API request cần authentication phù hợp; credentials/token phải được bảo vệ.                                         |
| **JSON**                    | Đọc và kiểm tra structured API response.                                                                            |
| **cURL**                    | Dùng command-line HTTP client để thực hiện API/HTTPS validation.                                                    |
| **HTTP response codes**     | Phân biệt success, authentication/authorization failure, client error và server-side failure.                       |
| **Timeouts**                | Request phải bounded; automation không được treo vô hạn.                                                            |
| **Retries**                 | Phải giới hạn retry và đặc biệt không blindly retry configuration changes.                                          |
| **Version-specific syntax** | Endpoint/behavior có thể phụ thuộc BIG-IP/FortiOS version, nên phải kiểm tra environment/runbook trước khi áp dụng. |

Syllabus đặc biệt yêu cầu bảo vệ API credentials/token, document read-only vs mutating methods, sử dụng bounded timeout/retry và không được blindly retry changes. Infrastructure_Operations_14_Da…

---

## 6. cURL, JSON assertion và automation mindset

Bạn sẽ không chỉ gọi API rồi “nhìn thấy JSON là xong”.

Một automation check đúng phải kiểm tra:

request có chạy thành công không → TLS validation có pass không → HTTP response code có đúng không → response có parse được thành JSON không → required field có tồn tại không → field value có đúng expected state không → command/script cuối cùng trả về observable success/failure hay không.

Syllabus yêu cầu test các tình huống:

`401/403`

`timeout`

`malformed response`

`invalid/expired/untrusted certificate`

`healthy/unhealthy member`

và mọi failed assertion phải tạo ra **observable non-success result**. Infrastructure_Operations_14_Da…

---

## 7. Operational safety, security và evidence

Day 9 không dừng ở product knowledge. Mọi thao tác đều phải được nhìn dưới góc Infrastructure Operations.

Bạn phải phân biệt:

**Read-only operation** — inspect route, policy, session, pool/member status, certificate details, API GET.

và

**Mutating operation** — sửa firewall policy, NAT, monitor, Pool Member, Virtual Server, SSL profile hoặc certificate.

Real configuration change phải có **approved change**, expected impact, pre-check, implementation plan, verification, backout/rollback và post-check. Syllabus nhấn mạnh rằng real changes phải dùng existing approved change/backout workflow. Infrastructure_Operations_14_Da…

Credentials, secret headers, passwords, tokens và private keys không được xuất hiện trong evidence/transcript.

---

# Module 2 — Thực hành

Module thực hành được thiết kế để biến toàn bộ kiến thức phía trên thành một investigation flow có thể kiểm chứng.

Syllabus yêu cầu evidence thực hành riêng cho **FortiGate CLI route/policy/session diagnosis, allow/deny tests và controlled configuration/rollback**, đồng thời yêu cầu **BIG-IP Virtual Server/Pool/Monitor validation, TLS diagnosis và authenticated API/cURL assertions**. Infrastructure_Operations_14_Da…

## 1. FortiGate Inspection Lab

Bạn sẽ dùng FortiGate GUI/CLI trong lab hoặc environment được authorization để inspect:

interface state/configuration → routing → address/service objects → firewall policy → policy order → NAT → session → traffic logs.

Mục tiêu không chỉ là đọc configuration, mà phải dựng được reasoning:

**“Traffic từ source A đến destination B trên TCP/443 sẽ đi vào interface nào, route ra đâu, match policy nào, bị NAT thành gì, session có tồn tại không và log nói gì?”**

---

## 2. Allowed versus Blocked Traffic Test

Bạn phải thực hiện controlled traffic tests với ít nhất hai expectation:

một flow **expected allow**

và một flow **expected deny/block**.

Sau đó phải chứng minh outcome bằng combination của client test, firewall policy/session/log evidence.

Điều này rất quan trọng vì “ping được” không đủ để kết luận application connectivity đúng.

---

## 3. FortiGate Fault Investigation

Syllabus yêu cầu simulated fault liên quan đến **incorrect firewall policy hoặc NAT configuration**. Infrastructure_Operations_14_Da…

Bạn sẽ phải xác định symptom, collect evidence, locate wrong object/policy/NAT, giải thích root cause, thực hiện correction chỉ trong approved lab scope, re-test và lưu rollback evidence.

---

## 4. FortiGate packet/debug demonstration

Additive lab yêu cầu dùng approved FortiGate runbook để kiểm tra interface, route, policy và session, đồng thời thực hiện **tightly filtered, time-bounded packet/debug demonstration** khi được authorization. Debug phải được **stop sau khi thu thập đủ evidence**. Infrastructure_Operations_14_Da…

Đây là operational discipline rất quan trọng: diagnostic tool mạnh nhưng phải bounded để tránh tạo unnecessary load hoặc excessive output.

---

## 5. BIG-IP Object Validation Lab

Bạn sẽ inspect/configure trong lab:

Virtual Server → listener → default Pool → Pool → Pool Members → Health Monitor → TCP/HTTP profiles → SSL profiles.

Sau đó xác nhận backend availability và load-balancer state.

Bạn phải có khả năng giải thích được vì sao member là `UP`, `DOWN` hoặc unavailable dựa trên monitor behavior chứ không chỉ đọc icon GUI.

---

## 6. Load-balancing Behavior Test

Tạo nhiều client requests có kiểm soát để quan sát request distribution.

Tùy environment, bạn sẽ xem ảnh hưởng của load-balancing method và persistence.

Mục tiêu không phải benchmark performance mà là xác minh rằng **Virtual Server thực sự gửi traffic đến expected backend members**.

---

## 7. Health Monitor Failure Injection

Một backend hoặc monitor fixture sẽ được làm unhealthy có kiểm soát.

Bạn phải phân biệt:

backend network/TCP failure

với

TLS problem

với

HTTP/application health check failure.

Syllabus coi đây là capability trọng tâm của Day 9. Infrastructure_Operations_14_Da…

---

## 8. SSL/TLS Validation Lab

Bạn sẽ kiểm tra HTTPS endpoint và certificate theo các dimension:

trust → chain → hostname → expiration → SNI-related behavior.

Negative fixtures có thể bao gồm expired certificate hoặc untrusted certificate.

Quan trọng nhất: **không được dùng cách bypass TLS verification chỉ để làm test “pass”**. Đây là requirement được ghi trực tiếp trong syllabus. Infrastructure_Operations_14_Da…

---

## 9. BIG-IP iControl REST API Lab

Bạn sẽ thực hiện **authenticated read-only requests** tới BIG-IP iControl REST API bằng `cURL`.

Các request ban đầu phải tập trung vào inspection, ví dụ trạng thái Virtual Server/Pool/Pool Member.

Bạn phải lưu:

request objective, endpoint/path thích hợp, HTTP status, cURL exit status, selected non-secret JSON fields và assertion result.

Không lưu Authorization header/token/password.

---

## 10. JSON Parsing và Automated Assertions

Response API sẽ được parse để xác định trạng thái có đúng expectation hay không.

Ví dụ logic:

Expected: Pool Member = healthy  
Actual API response: unhealthy  
Assertion = FAIL  
Script/tool exit result = non-success.

Không được coi automation thành công chỉ vì `curl` lấy được response.

---

## 11. Negative API Tests

Syllabus yêu cầu thực hành các fixtures như:

`401` — authentication problem.

`403` — authenticated nhưng không authorized hoặc access bị policy từ chối.

`timeout` — connectivity/service response problem.

`malformed response` — response không đúng structured format expectation.

Những test này dạy bạn viết operational automation theo nguyên tắc **fail visibly**, không silent success. Infrastructure_Operations_14_Da…

---

## 12. Integrated Firewall → BIG-IP Troubleshooting

Đây là phần nối FortiGate với BIG-IP thành một application path.

Ví dụ:

**Client → FortiGate → BIG-IP Virtual Server → Pool → Member**

Bạn sẽ troubleshoot theo layer:

client connectivity → routing → firewall policy/NAT/session → BIG-IP listener → TCP/TLS → Virtual Server → Pool → member health → application HTTP response.

Đây là cách tránh “tool hopping” ngẫu nhiên.

---

## 13. Corrective Action và Rollback

Khi fault fixture thực sự yêu cầu configuration change, lab phải có:

before-state evidence → approved change → minimal correction → verification → post-state evidence → rollback procedure → rollback verification.

Real production change không nằm trong phạm vi tự do thực hành. Syllabus yêu cầu high-risk firewall/load-balancer work phải đi qua authorization và backout workflow. Infrastructure_Operations_14_Da…

---

## 14. Evidence Package

Day 9 không hoàn thành chỉ vì service hoạt động trở lại.

Evidence cuối cùng phải chứng minh được:

**Expected state → Observed failure → Diagnostic evidence → Root cause → Corrective action → Verification → Rollback/cleanup status.**

Đặc biệt với API/cURL, syllabus yêu cầu giữ **expected/actual results, HTTP status, tool exit status, selected non-secret response fields và cleanup evidence**. Infrastructure_Operations_14_Da…

---

# Quan hệ giữa hai Module

Điểm quan trọng nhất của Day 9 là **Module Lý thuyết và Module Thực hành không tách rời**.

Bạn học **policy ordering** để sau đó chứng minh wrong policy match.

Bạn học **session table** để biết một firewall decision thực tế đã xảy ra thế nào.

Bạn học **Health Monitor** để phân biệt “member down” với “application lỗi”.

Bạn học **certificate chain/SNI** để không coi mọi TLS error là certificate expiry.

Bạn học **HTTP status + cURL exit code + JSON assertion** để xây automation check có ý nghĩa vận hành.

Bạn học **rollback/evidence** để một technical fix trở thành một **controlled enterprise operation**.

Day 9 cũng được syllabus xác định rõ là nơi phải tạo evidence cho cả **FortiGate CLI** và **BIG-IP API/cURL/SSL**, sau đó các kỹ năng này còn được tái sử dụng trong integrated operations ở các Day sau. Infrastructure_Operations_14_Da…

## Kết quả cuối cùng cần đạt sau Day 9

Sau khi hoàn tất cả hai module, mục tiêu không phải chỉ là “biết FortiGate và BIG-IP”, mà là có thể nhìn một lỗi application connectivity và suy luận có hệ thống qua:

**Network → Firewall → NAT/Session → TCP → TLS → BIG-IP Virtual Server → Pool → Health Monitor → Backend → HTTP/Application**

đồng thời biết **verify bằng CLI/API/log**, thực hiện safe troubleshooting, không expose secrets, không bypass TLS validation, biết giới hạn L1/L2, biết khi nào cần change approval/escalation và tạo được evidence đủ để người khác review/reproduce.

Đây cũng phù hợp với mục tiêu tổng thể của chương trình: accelerated operational readiness với hands-on lab, verification evidence, controlled troubleshooting và integrated operations practice. Infrastructure_Operations_14_Da…

**Phạm vi Day 9 đã được chia thành hai module để học sâu tiếp theo:**

**Module 1 — Lý thuyết:** FortiGate/FortiOS → firewall packet flow → policy/NAT/session/logging → BIG-IP/LTM objects → load balancing/persistence/SNAT → SSL/TLS/certificate/SNI → `tmsh` → iControl REST → HTTP/JSON/cURL → safe operations và troubleshooting methodology.

**Module 2 — Thực hành:** FortiGate inspection + allow/deny + policy/NAT fault → bounded packet/debug → BIG-IP Virtual Server/Pool/Monitor inspection → load-balancing test → monitor failure → TLS failure diagnosis → read-only iControl REST/cURL → JSON assertions → 401/403/timeout/malformed response → integrated FortiGate-to-BIG-IP troubleshooting → rollback + evidence package.

Khi bạn bảo **“Trình bày Module Lý thuyết Day 9”**, tôi sẽ bắt đầu giảng toàn bộ Module 1 từ foundation đến operational troubleshooting, bao gồm **What / Why / How / workflow / security / verification / troubleshooting / evidence / escalation / commands/examples**, nhưng chưa sang lab.
