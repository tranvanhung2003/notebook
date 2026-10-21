Dưới đây là **bản đồ nội dung Day 11** được chia đúng thành **2 module: Lý thuyết và Thực hành**. Tôi bám sát syllabus của Project, chưa đi sâu theo kiểu bài giảng từng khái niệm; mục đích là để bạn thấy **toàn bộ phạm vi Day 11 phải học và phải làm**, trước khi chúng ta triển khai từng module thật chi tiết. Day 11 thuộc nhóm **Monitoring, ITSM, Asset Management & Privileged Access Operations**, nối tiếp Day 10 về monitoring/log investigation và chuẩn bị dữ liệu, workflow, evidence cho Day 12–13 về automation và IAM review. Infrastructure_Operations_14_Da…

# Day 11 — Tổng quan

Trọng tâm của Day 11 là vận hành ba nhóm năng lực enterprise có quan hệ rất chặt với nhau:

**ITSM / ServiceNow** quản lý _công việc vận hành_ — ai đang xử lý, việc gì xảy ra, mức độ ảnh hưởng, approval nào đã được cấp, thay đổi nào đã được thực hiện, rollback ra sao và ticket được đóng thế nào.

**Privileged Identity & Access Management** quản lý _quyền truy cập đặc quyền_ — ai được quyền truy cập hệ thống nào, tại sao được cấp quyền, ai approve, thời hạn bao lâu, session nào đã diễn ra, quyền đã hết hạn/revoke chưa và audit trail ở đâu.

**Asset Management / PalletControl** quản lý _đối tượng hạ tầng_ — server, endpoint, software/hardware asset nào đang tồn tại, owner là ai, trạng thái lifecycle là gì, inventory có chính xác không và CMDB/asset database có khớp thực tế không.

Ba mảng này kết hợp thành một operational chain điển hình:

> **Asset/CI → Incident/Request/Change → Approval → Privileged Access → Maintenance → Verification → Logs/Evidence → Asset/CMDB Update → Closure → Handover**

Syllabus yêu cầu không chỉ biết giao diện hoặc định nghĩa, mà phải có khả năng **thực hiện workflow có kiểm soát, giữ audit evidence, reconcile dữ liệu, xác minh quyền, xử lý discrepancy và biết khi nào phải escalate**. Infrastructure_Operations_14_Da…

---

# MODULE 1 — LÝ THUYẾT

Module này gồm khoảng **8 khối kiến thức lớn**.

## 1. Enterprise IT Service Management và ServiceNow ITSM architecture

Bạn cần hiểu ITSM không đơn thuần là “ticketing system”, mà là framework để kiểm soát toàn bộ lifecycle của operational work.

Phạm vi cần học gồm:

- IT service management fundamentals.
- ServiceNow ITSM architecture ở mức Infrastructure Operations.
- Quan hệ giữa user, assignment group, task, ticket, CI, approval, SLA và audit history.
- Cách ServiceNow hỗ trợ incident handling, request fulfillment, change control, knowledge management, asset/CMDB integration và operational handover.

Bạn phải hình dung được ServiceNow là nơi liên kết:

**người yêu cầu → business/service → CI → operational task → approval → implementation → evidence → closure**.

---

## 2. Các record/object chính trong ServiceNow

Day 11 yêu cầu phân biệt rõ:

- **Incident**
- **Problem**
- **Request**
- **Task**
- **Change**
- **Approval**
- **Knowledge**
- **Asset**
- **CMDB / Configuration Item**

Đây là phần đặc biệt quan trọng vì người mới thường nhầm:

**Incident ≠ Request ≠ Change ≠ Problem.**

Ví dụ:

- Server bị service down → thường bắt đầu bằng **Incident**.
- User xin access → **Request**.
- Cần restart production service theo maintenance window → có thể cần **Change**.
- Incident lặp lại nhiều lần và cần root-cause investigation → **Problem**.
- Server/VM/firewall/load balancer → có thể tồn tại dưới dạng **CI** trong CMDB.

Bạn cũng cần hiểu ticket có thể sinh ra các subordinate tasks và approval records khác nhau.

---

## 3. Ticket lifecycle và operational ownership

Syllabus yêu cầu nắm toàn bộ lifecycle của ticket, không chỉ mở và đóng ticket.

Các nội dung gồm:

- ticket creation/intake;
- categorization;
- assignment;
- ownership;
- reassignment;
- investigation;
- work notes;
- approvals;
- implementation;
- validation;
- resolution;
- closure;
- audit history;
- shift handover.

Bạn phải biết trách nhiệm của **assignment group**, individual assignee và các nhóm hỗ trợ khác.

Một trọng tâm lớn là tránh tình trạng:

> “Ticket đang tồn tại nhưng không rõ ai chịu trách nhiệm.”

Operational ownership phải luôn xác định được.

---

## 4. Impact, Urgency, Priority và SLA

Bạn cần hiểu:

- **Impact**: mức độ ảnh hưởng.
- **Urgency**: mức độ cần xử lý nhanh.
- **Priority**: mức ưu tiên được tính hoặc xác định từ Impact/Urgency theo policy.
- **SLA**: thời gian hoặc target mà service/support organization cam kết.

Ví dụ phải suy luận được sự khác nhau giữa:

- 1 user không truy cập test environment;
- 300 users không truy cập production application;
- privileged administrative access request chưa được approve;
- backup monitoring warning nhưng chưa có service impact.

Không được chỉ nhìn “severity” mà bỏ qua business impact.

---

## 5. Work notes, customer-visible comments và audit trail

Đây là phần rất quan trọng cho enterprise operations.

Bạn cần phân biệt:

- **Work notes**: thông tin nội bộ cho support/operations.
- **Customer-visible comments**: nội dung người yêu cầu có thể thấy.
- ticket fields;
- journal entries;
- attachments;
- approvals;
- audit records;
- display/history information.

Syllabus đặc biệt cảnh báo rằng **ServiceNow history không được mặc định xem là audit archive đầy đủ**. History sets được tạo từ audit data nhưng có lifecycle riêng; phải xem actual audit configuration, table/field ACL và retention configuration của instance. Infrastructure_Operations_14_Da…

Điều này dẫn đến một nguyên tắc vận hành:

> Không thấy record trong History không đồng nghĩa event chưa từng xảy ra.

Phải kiểm tra audit configuration, permissions, retention và extraction scope.

---

## 6. Change Management

Một phần lớn Day 11 liên quan operational change.

Bạn phải hiểu:

- approval gates;
- change risk;
- maintenance window;
- implementation plan;
- verification plan;
- backout plan;
- change ownership;
- closure criteria.

Một change đúng không phải chỉ viết:

> “Restart server.”

Mà cần ít nhất xác định:

**Why → Scope → Risk → Dependencies → Pre-check → Implementation → Verification → Stop condition → Rollback → Post-check → Evidence → Approval**.

Bạn cũng cần hiểu tại sao một change có thể technically successful nhưng operationally failed nếu:

- thiếu approval;
- thực hiện ngoài maintenance window;
- không verify service;
- không thu evidence;
- không update ticket;
- không chứng minh rollback readiness.

---

## 7. Resolution, closure và shift handover

Bạn phải phân biệt:

**Resolved** và **Closed**.

Operational closure cần trả lời được:

- symptom đã biến mất chưa?
- service đã recover chưa?
- root cause đã xác định chưa?
- workaround hay permanent fix?
- user/business đã được thông báo chưa?
- evidence đã được attach/reference chưa?
- asset/CI có cần update không?
- có open risk hay follow-up task không?

**Shift handover** cũng là syllabus topic bắt buộc.

Một handover tốt phải truyền đủ:

- ticket/reference;
- current status;
- affected CI;
- observed symptoms;
- actions đã thực hiện;
- actions không nên lặp lại;
- pending approvals;
- risk;
- next troubleshooting step;
- owner;
- deadline/SLA;
- escalation state.

---

# 8. Privileged Identity & Access Management

Đây là khối lý thuyết thứ hai rất lớn của Day 11.

Bạn cần học **privileged ID lifecycle**:

**Request → Identity verification → Business justification → Owner → Approval → Provision/Assign → Access → Audit → Expiry → Revocation → Review**

Các concept bắt buộc gồm:

- privileged identity;
- privileged ID;
- account ownership;
- authorization;
- access request;
- approval workflow;
- session/gateway awareness;
- credential checkout;
- credential rotation;
- access expiration;
- revocation;
- audit records.

Mục tiêu không phải học password mà là học **governance xung quanh privileged access**.

---

## 9. PS/Gateway và AccessCheck

Syllabus yêu cầu học operational workflow đối với:

- **PS/Gateway**
- **AccessCheck**

Nhưng đây là các tên product/environment-specific trong organization. Syllabus **không cho phép tự suy đoán vendor, command, feature hoặc workflow**.

Trước khi làm product-specific lab phải xác định:

- exact product/vendor;
- version;
- deployed architecture;
- authentication flow;
- request/approval workflow;
- runbook;
- permitted operations;
- audit/session recording capability;
- credential rotation process.

Syllabus nói rõ PS/Gateway và AccessCheck phải theo organization's approved runbooks và real configuration; không được giả định undocumented capabilities. Infrastructure_Operations_14_Da…

Do đó khi học sâu module này, tôi sẽ phân biệt:

**generic privileged-access principles** và **organization-specific product workflow**.

---

# 10. Privileged session / gateway model

Bạn cần hiểu khái niệm session mediation:

User không nhất thiết lấy permanent privileged credential trực tiếp.

Một gateway có thể kiểm soát:

**User → Authentication → Approval/Policy → Gateway → Target system**

Và có thể tạo evidence như:

- request ID;
- approval;
- session ID;
- login timestamp;
- target system;
- command/session metadata;
- disconnect timestamp.

Mục tiêu là tăng accountability và giảm uncontrolled privileged credentials.

---

# 11. Credential checkout và rotation controls

Bạn cần hiểu:

- khi nào credential được checkout;
- người nào được checkout;
- checkout duration;
- expiry;
- automatic/manual rotation;
- failed rotation;
- emergency handling.

Một credential rotation failure không nên được “fix” bằng cách tùy tiện đặt password mới ngoài workflow.

Cần xác định:

- account owner;
- dependency;
- target system;
- application/service đang dùng credential;
- rotation job;
- last successful rotation;
- impact;
- escalation owner.

---

# 12. AWS Console IAM access trong Day 11

AWS IAM đã được giới thiệu ở Day 4; Day 11 tập trung vào **governance và privileged access operations**.

Bạn cần ôn và mở rộng:

- AWS Console identity;
- IAM user;
- IAM role;
- permission assignment;
- approval linkage;
- access expiry;
- effective access;
- audit evidence.

Các câu hỏi operations cần trả lời được:

> Người này là ai?

> Họ đang dùng IAM user, role hay federated identity?

> Ai approve?

> Role/permission nào đã được assign?

> Permission đã hết hạn chưa?

> Quyền đã revoke thực sự chưa?

> Có evidence nào chứng minh access đã xảy ra?

---

# 13. Access Review

Bạn cần học cách review privileged access theo các câu hỏi:

- account có legitimate owner không?
- business need còn tồn tại không?
- approval còn valid không?
- permission có excessive không?
- account có active không?
- account có dormant không?
- account có orphaned không?
- role assignment có phù hợp không?
- access có expired nhưng vẫn active không?

Đây chính là nền tảng trực tiếp cho **Day 13 — 90-day IAM review**.

---

# 14. Segregation of Duties

**Segregation of Duties (SoD)** có nghĩa các quyền nhạy cảm không nên tập trung không kiểm soát vào một người.

Ví dụ cần nhận biết:

Một người vừa:

- request access;
- approve request;
- cấp quyền;
- sử dụng quyền;
- xác nhận audit result

→ tạo ra governance weakness.

Bạn cần học cách nhận diện SoD conflict và escalation, không phải tự ý revoke access nếu không nằm trong authority boundary.

---

# 15. Dormant accounts

**Dormant account** là account không có activity trong một khoảng thời gian theo policy.

Điểm cần lưu ý:

- “Không thấy activity” chưa chắc bằng “không được sử dụng”.
- Phải hiểu data source và retention.
- Có thể thiếu audit record.
- Service account có usage model khác human account.

Day 13 sẽ đào sâu hơn việc cutoff 90 ngày; Day 11 chủ yếu xây foundation.

---

# 16. Orphaned accounts

**Orphaned account** thường là account không còn xác định được valid owner hoặc tồn tại ngoài expected lifecycle.

Ví dụ:

- employee đã rời organization;
- team không còn tồn tại;
- asset bị decommission nhưng privileged account vẫn active;
- service account không có owner;
- account tồn tại trong privileged-access system nhưng không có record tương ứng trong identity source.

Đây là high-risk finding và thường cần review/escalation.

---

# 17. Break-glass access

Bạn cần hiểu:

**break-glass account/access** dành cho emergency access khi normal access path không hoạt động.

Không được hiểu break-glass là “admin password dùng chung”.

Bạn phải học các controls như:

- strict ownership;
- emergency use criteria;
- limited distribution;
- strong authentication;
- usage logging;
- post-use review;
- credential rotation;
- incident/change linkage.

---

# 18. Privileged activity auditing

Mọi privileged action quan trọng phải có khả năng truy vết đến:

**Who → What → Which account/role → Which system → When → Why → Which approval/ticket → Result**

Evidence có thể đến từ:

- PS/Gateway;
- AccessCheck;
- AWS IAM/CloudTrail;
- operating system logs;
- ServiceNow;
- application audit records.

Day 11 yêu cầu học **correlation**, không xem từng log source tách biệt.

---

# 19. PalletControl IT Asset Management

Khối thứ ba của Day 11 là asset management.

Bạn cần học:

- hardware inventory;
- software inventory;
- managed endpoints;
- asset ownership;
- lifecycle status;
- asset reconciliation;
- inventory freshness;
- data quality;
- controlled maintenance of asset records.

PalletControl trong syllabus phải được học theo **actual organization deployment**. Không được lấy một generic asset database rồi tuyên bố đã thành thạo PalletControl. Syllabus yêu cầu exact product/version/runbook và actual implementation phải được xác nhận. Infrastructure_Operations_14_Da…

---

# 20. Asset vs CI / CMDB

Một khái niệm quan trọng cần phân biệt:

**Asset** thiên về quản lý vòng đời, ownership, inventory, procurement/financial/physical tracking.

**CI — Configuration Item** thiên về operational relationship và service configuration trong CMDB.

Trong thực tế có thể có mapping như:

**Physical server Asset ↔ Server CI**

Nhưng hai record không nhất thiết giống hệt nhau.

Bạn cần hiểu lý do mismatch có thể xuất hiện:

- discovery lag;
- manual update;
- duplicate records;
- renamed hostname;
- VM cloning;
- decommission chưa sync;
- ownership change;
- agent offline;
- stale data.

---

# 21. Asset reconciliation

Đây là năng lực thực tế quan trọng.

Bạn cần đối chiếu:

**PalletControl ↔ Server Inventory ↔ Endpoint Data ↔ CMDB ↔ ServiceNow**

Các loại discrepancy phải nhận diện:

- Missing
- Duplicate
- Stale
- Conflicting
- Unknown owner
- Incorrect status
- Old hostname
- Wrong environment
- Incorrect lifecycle state

Không được tự sửa record ngay; phải xác minh source of truth và quyền được chỉnh sửa.

---

# 22. Inventory freshness và data quality

Một inventory entry tồn tại không có nghĩa nó còn đáng tin.

Bạn cần xem:

- last inventory timestamp;
- last agent check-in;
- collector/job status;
- data source;
- asset owner;
- expected reporting frequency;
- discrepancy duration.

Ví dụ:

Server vẫn “Active” nhưng agent 90 ngày không check-in.

Kết luận đúng không phải:

> Server đã decommission.

Mà là:

> Inventory data đang stale; cần xác minh bằng authoritative source.

---

# MODULE 2 — THỰC HÀNH

Syllabus Day 11 yêu cầu một chuỗi lab tổng hợp, không phải ba bài lab hoàn toàn tách biệt. Infrastructure_Operations_14_Da…

Có thể chia Module Thực hành thành **8 exercise lớn**.

---

## Lab 1 — Simulated ServiceNow Incident Workflow

Bạn sẽ thực hiện một Incident từ đầu đến cuối.

Cần làm được:

**Create/receive Incident → Verify CI → Set assignment → Determine Impact/Urgency/Priority → Troubleshoot → Record work notes → Collect evidence → Resolve → Close/Handover**

Phải chứng minh:

- ticket owner;
- assignment group;
- affected CI;
- priority;
- timestamps;
- investigation steps;
- technical evidence;
- resolution;
- closure information.

Không được ghi work note kiểu:

> “Checked issue. Fixed.”

Mà phải đủ khả năng audit lại.

---

## Lab 2 — Service Request Workflow

Thực hiện một service/access request.

Ví dụ:

User yêu cầu temporary administrative access.

Phải kiểm tra:

- requester identity;
- target system;
- requested privilege;
- business justification;
- approval;
- expiry;
- fulfillment;
- verification;
- closure.

Lab này sẽ liên kết trực tiếp sang privileged-access workflow.

---

## Lab 3 — Approved Change Workflow

Bạn sẽ xử lý một change có kiểm soát.

Phải chuẩn bị:

- scope;
- risk;
- affected CI;
- approval;
- maintenance window;
- implementation plan;
- pre-checks;
- rollback/backout plan;
- validation;
- post-check;
- closure.

Phải ghi rõ **stop condition**.

Ví dụ:

> Nếu health check thất bại sau bước X thì dừng implementation và thực hiện rollback.

---

# 4. ServiceNow Evidence Extraction

Đây là phần additive rất quan trọng.

Bạn phải biết cách xuất một bounded sample từ ServiceNow, bao gồm khi được phép:

- ticket number;
- stable `sys_id`;
- CI/account reference;
- assignment;
- state;
- timestamps;
- approval references.

Ngoài ra còn phải thu riêng khi authorized:

- journal/work-note evidence;
- audit evidence;
- attachment evidence.

Nếu không lấy được phải ghi:

**Unavailable due to ACL / retention / missing auditing / export limitation**

chứ không được âm thầm bỏ qua. Infrastructure_Operations_14_Da…

---

# 5. Reconstruct Maintenance Timeline

Bạn phải dựng lại một maintenance timeline từ nhiều source.

Ví dụ:

**10:00 Change approved**

**10:03 Privileged access granted**

**10:06 Admin login**

**10:08 Service stopped**

**10:10 Configuration changed**

**10:12 Service started**

**10:13 Health check passed**

**10:15 Access revoked**

**10:18 Change completed**

Timeline phải đối chiếu giữa:

- ServiceNow;
- privileged-access records;
- server/application logs.

Mục tiêu là chứng minh các event có thể reconcile về mặt thời gian và ownership.

---

# 6. Privileged Access Lifecycle Lab

Thực hiện hoặc mô phỏng trên:

- PS/Gateway;
- AccessCheck;
- AWS IAM.

Đối với từng system cần đi qua:

**Request → Approval → Access → Verify → Expire/Revoke → Audit**

Bạn phải kiểm tra:

- user identity;
- entitlement;
- account/role;
- approval;
- access duration;
- expiry;
- revocation;
- denied access after revocation;
- audit/session reference.

Không được đưa password, MFA secret, token hoặc secret header vào evidence.

---

# 7. Failed Privileged Access / Rotation Scenario

Day 11 còn yêu cầu rehearsal failure scenarios.

Ví dụ:

### Scenario A — Request approved nhưng login vẫn denied

Phải phân tích:

- approval state;
- entitlement;
- gateway policy;
- target account;
- account status;
- network path;
- session audit.

### Scenario B — Access hết hạn nhưng permission vẫn tồn tại

Phải xác minh effective access và escalate.

### Scenario C — Credential rotation failure

Phải xác định:

- rotation target;
- last successful rotation;
- dependency;
- error;
- impact;
- runbook action;
- escalation.

Không tự đổi credential ngoài approved workflow.

---

# 8. PalletControl Inventory Reconciliation Lab

Bạn sẽ lấy controlled asset dataset và so với:

- server inventory;
- endpoint inventory;
- CMDB;
- ServiceNow.

Phải tìm:

- missing assets;
- duplicate assets;
- stale assets;
- ownership mismatch;
- lifecycle mismatch.

Sau đó phân loại từng discrepancy.

Ví dụ:

| PalletControl | CMDB    | Server     | Kết luận                      |
| ------------- | ------- | ---------- | ----------------------------- |
| Present       | Present | Present    | likely valid                  |
| Present       | Missing | Present    | CMDB reconciliation needed    |
| Present       | Present | Missing    | verify server/decommission    |
| Duplicate     | One CI  | One server | duplicate asset investigation |

Mục tiêu là **điều tra**, không phải “cleanup bằng mọi giá”.

---

# 9. PalletControl Maintenance Check

Theo additive requirement, nếu implementation hỗ trợ, bạn phải kiểm tra:

- collection status;
- job status;
- agent status;
- inventory freshness;
- ownership.

Nếu có authorized discrepancy correction, phải lưu:

**Before → Approval/Ticket → Correction → After → Verification**

Syllabus đặc biệt yêu cầu product/version/runbook của PalletControl phải được xác định; generic inventory lab chỉ được coi là simulation nếu không có actual product lab. Infrastructure_Operations_14_Da…

---

# 10. Cross-system Evidence Correlation

Đây là phần rất quan trọng để hoàn thành Day 11.

Bạn phải liên kết:

**ServiceNow Ticket**

↕

**Server Logs**

↕

**Privileged Access Record**

↕

**Asset/CI**

Một reviewer phải có khả năng hỏi:

> “Action này do ai thực hiện?”

và bạn truy ra được:

**Ticket → Approval → Privileged session → Host → Server log → CI/Asset**

---

# 11. Evidence Manifest

Evidence Day 11 không chỉ là screenshot.

Theo common evidence contract của chương trình, evidence item cần có các thuộc tính như:

- evidence ID;
- source system/product/version;
- environment/asset/account identifier;
- related ticket;
- stable record key;
- collector;
- timestamp/timezone;
- time interval;
- filters/commands;
- export format;
- encoding;
- file name/path;
- file size;
- record count;
- SHA-256;
- parent evidence/transformation metadata;
- limitations/errors.

Phải giữ **untouched raw export** và xử lý trên working copy. Matching hash chỉ chứng minh file không thay đổi so với baseline được ghi nhận, không chứng minh source/query/window ban đầu đã đầy đủ hay đúng. Infrastructure_Operations_14_Da…

---

# 12. Day 11 Deliverables

Khi hoàn thành Module Thực hành, bạn phải có tối thiểu các nhóm artifact sau:

1. **ServiceNow incident workflow evidence**
2. **Service request workflow evidence**
3. **Approved change + implementation/backout plan**
4. **ServiceNow extraction recipe**
5. **Ticket/journal/audit/approval evidence**
6. **Joined maintenance timeline**
7. **PS/Gateway lifecycle checklist**
8. **AccessCheck lifecycle checklist**
9. **AWS IAM privileged-access scenario**
10. **Denied/expired/revoked access verification**
11. **Privileged session/audit references**
12. **Credential rotation failure investigation**
13. **PalletControl inventory reconciliation**
14. **Missing/duplicate/stale asset discrepancy report**
15. **PalletControl maintenance/freshness record**
16. **Before/after authorized correction evidence**
17. **Evidence manifest**
18. **Shift handover record**

Syllabus đặt yêu cầu đầu ra Day 11 là **ServiceNow extraction recipe, joined timeline/evidence manifest, PalletControl discrepancy/maintenance record, và separate PS/Gateway + AccessCheck lifecycle checklists**. Một simulation chỉ chứng minh bạn hiểu workflow; để ghi nhận actual hands-on product proficiency phải có product lab thật và instructor sign-off. Infrastructure_Operations_14_Da…

---

# Cấu trúc học Day 11 mà chúng ta sẽ dùng

Khi bạn yêu cầu tôi đi sâu, tôi đề xuất chia đúng thành hai module lớn như sau:

**Module Lý thuyết** sẽ đi sâu lần lượt từ **ServiceNow/ITSM → ticket lifecycle → Incident/Problem/Request/Change → priority/SLA → approvals/change/rollback → audit/history/extraction → privileged identity lifecycle → PS/Gateway/AccessCheck → AWS IAM → dormant/orphaned/break-glass/SoD → privileged audit → PalletControl → Asset/CI/CMDB → reconciliation/data quality → cross-system evidence**.

**Module Thực hành** sẽ dựng một scenario xuyên suốt: **ServiceNow Incident/Request/Change → privileged-access request/approval → authorized maintenance → server log correlation → expiry/revocation → PalletControl/CMDB reconciliation → ServiceNow extraction → evidence manifest → closure/handover**, kèm các failure scenarios về denied access, expired privilege, rotation failure, stale/duplicate/missing asset và incomplete ServiceNow evidence.

Điểm quan trọng nhất của Day 11 là: **không chỉ biết dùng ServiceNow, PalletControl hay privileged-access tools riêng lẻ; bạn phải biết nối chúng thành một auditable enterprise operations workflow từ request/incident đến approval, access, implementation, verification, evidence, asset reconciliation, closure và handover.** Infrastructure_Operations_14_Da…
