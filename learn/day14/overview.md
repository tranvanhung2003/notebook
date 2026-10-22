# Day 14 — Tổng quan: Integrated Infrastructure Operations & Final Technical Readiness

Day 14 là **capstone day** của toàn bộ chương trình 14 ngày. Đây không phải một Day để học thêm một product hoàn toàn mới, mà là ngày **tích hợp, kiểm chứng và đánh giá năng lực vận hành** từ Day 1–13 trong một enterprise operations scenario liên hoàn. Theo syllabus, Day 14 phải bao phủ đồng thời server, cloud, hardware, virtualization, backup, networking, security, monitoring, ITSM, privileged access, IAM governance, vulnerability handling, automation, evidence, rollback, escalation và handover. Infrastructure_Operations_14_Da…

Khối lượng Day 14 được thiết kế khoảng **7.5 giờ**, trong đó chỉ khoảng **0.5 giờ Concept/instructor demonstration**, khoảng **6 giờ guided hands-on/fixture analysis** và **1 giờ evidence review/assessment**. Nghĩa là trọng tâm của Day 14 nằm ở thực hành tích hợp, troubleshooting và chứng minh kết quả bằng evidence, chứ không phải lecture dài như các Day trước. Infrastructure_Operations_14_Da…

Tôi sẽ chia nội dung Day 14 thành đúng **hai module lớn** như bạn yêu cầu:

1. **Module Lý thuyết — Integrated Operations, Decision-Making & Acceptance Framework**
2. **Module Thực hành — Integrated Enterprise Operations Capstone Lab**

Dưới đây mới là **bản đồ kiến thức đầy đủ** của hai module. Khi bạn yêu cầu từng module sau, tôi sẽ đi sâu theo kiểu foundation → workflow → command → verification → troubleshooting → evidence → rollback → escalation.

---

# Module 1 — Lý thuyết

## Integrated Operations, Decision-Making & Acceptance Framework

### 1. Mục tiêu của Module Lý thuyết

Module này giúp bạn hiểu cách các domain đã học trong Days 1–13 **liên kết thành một operational system duy nhất**.

Bạn không còn nhìn từng product riêng lẻ như:

`Linux → Windows → AWS → Nutanix → FortiGate → BIG-IP`

mà phải suy nghĩ theo một service flow kiểu:

**User/Application → DNS/Network → Firewall → Load Balancer → Server/VM → Storage → Backup → Monitoring → Logs → IAM → ITSM → Evidence**

Khi có sự cố, mục tiêu không chỉ là "fix service" mà phải đồng thời biết:

- service đang fail ở layer nào;
- evidence nào chứng minh nhận định đó;
- mình có authorization để thay đổi không;
- thay đổi nào ít rủi ro nhất;
- rollback là gì;
- ảnh hưởng downstream/upstream thế nào;
- ticket/change có đúng trạng thái không;
- sau fix cần verify những gì;
- khi nào phải stop và escalate;
- bàn giao ca sau cần để lại thông tin gì.

---

## 2. Integrated Infrastructure Operations

Đây là concept trung tâm của Day 14.

Bạn cần tổng hợp operational knowledge của toàn bộ infrastructure estate:

### Server layer

Bao gồm:

- Linux/RHEL administration;
- Windows Server administration;
- service/process health;
- CPU/memory/disk/filesystem;
- permissions;
- networking;
- event/log inspection;
- patch/maintenance awareness;
- configuration baseline.

Phải hiểu rằng một application outage có thể không phải do application mà có thể do:

- service stopped;
- port không listen;
- firewall local block;
- filesystem full;
- permissions sai;
- DNS resolution lỗi;
- route lỗi;
- expired certificate;
- backend dependency fail.

---

## 3. AWS Integrated Operations

Day 14 yêu cầu bạn có khả năng nhìn tổng hợp các thành phần AWS đã học:

- IAM;
- VPC;
- subnet;
- route table;
- EC2;
- EBS;
- S3;
- CloudWatch;
- CloudTrail.

Không chỉ kiểm tra từng resource tồn tại mà phải kiểm tra **dependency chain**.

Ví dụ:

`IAM role`
→ cho phép EC2 access S3

`VPC/subnet/route`
→ cho phép EC2 có network path

`Security Group`
→ cho phép traffic cần thiết

`EC2`
→ application chạy

`EBS`
→ dữ liệu mount đúng

`CloudWatch`
→ theo dõi operational health

`CloudTrail`
→ phục vụ audit về management activity

Bạn cũng phải phân biệt:

**Monitoring evidence**

và

**Audit evidence**

CloudWatch và CloudTrail không phải interchangeable.

---

# 4. HPE ProLiant / iLO Health Management

Phần này kiểm tra khả năng phân biệt:

- OS/application issue;
- hypervisor issue;
- physical hardware issue;
- out-of-band management issue.

Các nội dung phải biết nhìn lại gồm:

- hardware inventory;
- firmware awareness;
- power status;
- hardware health;
- IML;
- storage/controller;
- temperature/power/fan/memory alerts;
- iLO management plane.

Một nguyên tắc quan trọng của Day 14 là:

> Không xử lý software mãi khi evidence cho thấy root cause nằm ở physical hardware.

Bạn phải biết chuyển ownership đúng team/vendor khi cần.

---

# 5. Nutanix AOS / AHV / Prism Operations

Bạn cần tích hợp lại các khái niệm:

- AOS;
- AHV;
- Prism Element;
- Prism Central;
- CVM;
- AHV host;
- VM;
- storage container;
- virtual network;
- cluster health;
- tasks;
- alerts;
- audit records;
- capacity.

Phải phân biệt được failure domain, ví dụ:

`Guest OS issue`

khác với:

`VM configuration issue`

khác với:

`AHV host issue`

khác với:

`CVM/AOS issue`

khác với:

`physical node issue`.

---

# 6. HYCU Backup & Recovery

Day 14 không chấp nhận tư duy:

> Backup job = Success → hệ thống an toàn.

Bạn phải biết kiểm tra:

- protection policy;
- backup job;
- recovery point;
- source;
- target;
- retention;
- failure reason;
- restore readiness.

Đặc biệt phải hiểu:

**Successful backup ≠ proven recovery.**

Khả năng phục hồi phải được chứng minh bằng:

- isolated recovery;
- file/VM validation;
- integrity checking;
- observed recovery time.

Đây cũng là nguyên tắc chung của syllabus đối với AWS S3/HYCU DR. Infrastructure_Operations_14_Da…

---

# 7. Enterprise Networking Integration

Day 14 yêu cầu vận dụng kiến thức cả:

- Cisco Catalyst;
- Cisco Nexus;
- L2;
- L3.

Bạn cần suy luận được connectivity path qua:

- interface state;
- VLAN;
- access/trunk;
- 802.1Q;
- MAC learning;
- ARP;
- routing table;
- static route;
- port-channel;
- STP;
- default gateway.

Đặc biệt phải tránh troubleshooting theo kiểu nhảy ngay vào configuration change.

Workflow đúng thường là:

`Physical/interface`
→ `L2`
→ `L3`
→ `TCP/UDP`
→ `application`.

---

# 8. FortiGate Firewall Operations

Bạn cần tổng hợp:

- route;
- interfaces;
- zones;
- address objects;
- service objects;
- firewall policies;
- policy order;
- NAT;
- sessions;
- logs;
- allow/deny behavior.

Một request không đi qua firewall có thể do:

- route;
- wrong source/destination;
- policy mismatch;
- service mismatch;
- NAT;
- asymmetric path;
- session state.

Day 14 kiểm tra khả năng chứng minh firewall **có thực sự là fault domain hay không**, thay vì mặc định "network/firewall issue".

---

# 9. BIG-IP Load Balancer & SSL/TLS

Bạn phải nhìn lại đầy đủ:

- virtual server;
- listener;
- pool;
- pool member;
- health monitor;
- persistence;
- SNAT;
- TCP/HTTP profiles;
- client-side SSL;
- server-side SSL;
- certificate;
- certificate chain;
- hostname;
- expiry;
- SNI;
- HTTP/HTTPS validation.

Một nguyên tắc troubleshooting rất quan trọng:

Phải phân biệt:

**TCP failure**

vs

**TLS failure**

vs

**HTTP failure**

vs

**application health failure**.

Không được dùng một triệu chứng kiểu "website down" để kết luận ngay backend server down.

---

# 10. Centralized Logging & Investigation

Bạn phải có khả năng correlate evidence từ nhiều nguồn:

- Linux journal/syslog;
- Windows Event Logs;
- Nutanix events;
- HPE iLO/IML;
- FortiGate logs;
- BIG-IP logs;
- Cisco syslog;
- AWS CloudWatch;
- AWS CloudTrail.

Các khái niệm quan trọng:

- timestamps;
- timezone normalization;
- NTP;
- event severity;
- source identity;
- log collection;
- log retention;
- parsing;
- event correlation;
- evidence completeness.

Không được nhầm:

`No result`

với

`No event`.

Có thể query không thấy gì vì:

- source chưa gửi log;
- collector fail;
- parser fail;
- retention hết;
- query/filter sai;
- timezone/window sai;
- access bị giới hạn.

---

# 11. EventLog Analyzer

Bạn cần kiểm tra:

- source onboarding;
- event ingestion;
- search/filter;
- known-event reconciliation;
- collector/service health;
- ingestion delay;
- retention;
- reporting;
- export.

Day 14 tập trung vào việc dùng analyzer làm **operational investigation tool**, không đơn thuần biết giao diện.

---

# 12. Firewall Analyzer

Bạn cần vận dụng:

- traffic analysis;
- denied traffic;
- source/destination flows;
- rule utilization;
- firewall policy analysis;
- configuration change tracking;
- reports;
- security/compliance evidence.

Quan trọng nhất là liên hệ dữ liệu Analyzer với incident thực tế.

---

# 13. ServiceNow Operations

Day 14 phải sử dụng ServiceNow như **control plane về process**, không chỉ là nơi ghi chú.

Bạn phải hiểu và vận dụng:

- incident;
- request;
- change;
- task;
- approval;
- assignment;
- impact;
- urgency;
- priority;
- SLA awareness;
- implementation plan;
- maintenance window;
- backout plan;
- work notes;
- closure;
- handover.

Mọi configuration-changing activity phải có relationship rõ với authorized work.

---

# 14. PalletControl Asset Maintenance

Nội dung phải liên kết:

**Observed infrastructure**

với

**Asset inventory / CMDB-like records**.

Bạn phải phát hiện:

- missing asset;
- duplicate asset;
- stale asset;
- incorrect ownership;
- inventory freshness problem.

Syllabus đặc biệt yêu cầu không được suy đoán product-specific capability của PalletControl nếu actual implementation chưa được xác nhận. Infrastructure_Operations_14_Da…

---

# 15. Privileged Access Governance

Day 14 tổng hợp:

- privileged identity;
- account ownership;
- access request;
- approval;
- temporary access;
- expiry;
- revocation;
- break-glass;
- audit record;
- separation of duties.

Bạn phải có khả năng trả lời:

> "Người này có technical permission" chưa chắc đồng nghĩa với "người này được authorized thực hiện change này."

---

# 16. Condensed 90-Day IAM Review

Day 14 yêu cầu thực hiện phiên bản rút gọn của Day 13.

Các concept bắt buộc:

- review scope;
- as-of time;
- review cutoff;
- active/disabled/expired accounts;
- owner;
- last sign-in;
- last-used evidence;
- dormant account;
- stale entitlement;
- orphaned account;
- duplicate ID;
- service account;
- break-glass account;
- least privilege;
- separation of duties.

Một điểm cực quan trọng:

**90-day periodic review**

không đồng nghĩa với

**90-day inactivity threshold**.

Hai policy condition phải được xác định riêng.

---

# 17. Vulnerability Handling

Bạn phải tổng hợp lại:

- vulnerability;
- CVE awareness;
- CVSS awareness;
- exposure;
- exploitability;
- asset criticality;
- business impact;
- remediation priority;
- remediation owner;
- accepted exception;
- compensating control;
- retest.

Không được quyết định remediation priority chỉ theo CVSS number.

---

# 18. Operations Automation

Day 14 phải sử dụng những automation tools đã xây dựng/sửa trong Days 12–13:

- Excel/VBA;
- Python;
- Windows Batch;
- PowerShell.

Bạn phải hiểu:

`Input`
→ `normalization`
→ `processing`
→ `validation`
→ `output`
→ `evidence`.

Phải kiểm tra:

- failure propagation;
- exit codes;
- malformed input;
- missing input;
- encoding;
- duplicate records;
- regression;
- repeatability;
- no unintended side effects.

---

# 19. Outlook/MAPI Reporting

Day 14 có thể bao gồm final reporting qua classic Outlook/MAPI workflow.

Bạn cần hiểu:

- mailbox/folder access;
- message handling;
- attachments;
- filename collision;
- controlled reporting;
- draft generation.

Theo syllabus, compatible **classic Outlook** mới là target của COM/MAPI exercise; new Outlook cần được đánh giá riêng. Infrastructure_Operations_14_Da…

Và trong lab, mục tiêu là **verified draft**, không tự động gửi email.

---

# 20. Configuration Baselines

Bạn phải biết baseline của một system/resource gồm những gì, ví dụ:

- OS/version;
- services;
- network;
- firewall;
- IAM;
- storage;
- application state;
- cluster state;
- backup protection;
- monitoring.

Baseline cho phép trả lời:

> Điều gì đã thay đổi so với trạng thái được chấp thuận?

---

# 21. Monitoring & Alert Verification

Không được chỉ nhìn "green dashboard".

Phải kiểm tra:

- metric;
- threshold;
- alarm;
- source;
- ingestion;
- notification;
- time range;
- expected alert;
- resolved state.

Một monitor không phát alert có thể do:

- system healthy;

hoặc:

- monitoring pipeline broken.

Day 14 yêu cầu biết phân biệt hai trường hợp.

---

# 22. Backup / Restore Testing

Module Lý thuyết phải củng cố chuỗi:

**Backup → Recovery Point → Restore → Validation → RPO/RTO evidence**

Không thể kết luận DR readiness nếu chưa restore và verify.

---

# 23. Security Controls

Security được tích hợp xuyên suốt chứ không phải một bước riêng.

Bao gồm:

- least privilege;
- privileged access;
- IAM;
- MFA awareness;
- firewall controls;
- segmentation;
- secure administration;
- TLS validation;
- credential protection;
- auditability;
- approved scope.

---

# 24. Audit Evidence

Đây là một trong những competence quan trọng nhất Day 14.

Mỗi evidence item về nguyên tắc phải có:

- evidence ID;
- source product/system/version;
- asset/account;
- environment;
- related ticket;
- collector;
- collection timestamp;
- timezone;
- query/time window;
- exact filters/commands;
- format/encoding;
- size;
- record count;
- SHA-256;
- classification;
- parent evidence;
- transformation/script version;
- limitations/errors.

Raw source phải được giữ nguyên; derived copy phải trace được về original. Infrastructure_Operations_14_Da…

**Hash matching** chỉ chứng minh data không thay đổi so với baseline hash đã ghi, không tự động chứng minh source là đúng hoặc query là complete.

---

# 25. Incident Escalation

Bạn phải biết boundary của L1/L2.

Có thể xử lý khi:

- scope rõ;
- runbook cho phép;
- authorization hợp lệ;
- change reversible;
- impact trong phạm vi.

Phải stop/escalate khi ví dụ:

- evidence mâu thuẫn;
- blast radius chưa biết;
- cần privilege ngoài scope;
- hardware fault;
- cluster resilience không đủ;
- backup recovery không đảm bảo;
- vendor-level lỗi;
- security incident nghiêm trọng;
- change không có approved rollback;
- production safety bị đe dọa.

---

# 26. Operational Handover

Handover phải chuyển giao được operational state cho người không tham gia trực tiếp incident.

Ít nhất cần thể hiện:

- issue;
- impact;
- timeline;
- actions performed;
- current status;
- unresolved items;
- risks;
- monitoring needed;
- next action;
- owner;
- ticket/change;
- evidence location.

---

# 27. Final Technical Readiness

Điểm cuối cùng của Module Lý thuyết là bạn hiểu rằng chương trình này đánh giá **foundation knowledge + supervised operational readiness**, không tuyên bố advanced mastery trên mọi enterprise product. Infrastructure_Operations_14_Da…

Kết quả cuối cùng phải phân loại competence thành:

- **H — Hands-on actual product lab**
- **S — Simulation/fixture**
- **D — Demonstration**
- **T — Theory/reference**

và đồng thời ghi nhận:

- independent hay assisted;
- evidence ID;
- assessment outcome;
- reviewer.

Các gap chưa thực hành thực tế **không được tự động coi là hoàn thành**. Infrastructure_Operations_14_Da…

---

# Module 2 — Thực hành

## Integrated Enterprise Operations Capstone Lab

Đây là phần trọng tâm của Day 14.

## 1. Objective tổng thể

Bạn sẽ đóng vai Infrastructure Operations engineer xử lý một enterprise environment có nhiều system liên kết.

Bạn phải thực hiện toàn bộ lifecycle:

**Receive ticket**

→ **Understand scope**

→ **Verify authorization**

→ **Establish baseline**

→ **Perform health checks**

→ **Identify symptom**

→ **Localize fault domain**

→ **Collect evidence**

→ **Troubleshoot**

→ **Perform approved corrective action**

→ **Verify recovery**

→ **Rollback nếu cần**

→ **Correlate logs**

→ **Update ticket/assets**

→ **Review IAM/vulnerabilities**

→ **Run automation**

→ **Peer review**

→ **Handover**

→ **Final acceptance**

Đây chính là Assignment/Lab chính thức của Day 14. Infrastructure_Operations_14_Da…

---

# 2. Lab Inputs

Bạn phải làm việc dựa trên các artifact được chuẩn bị trước:

- approved build sheet;
- runbooks;
- environment inventory;
- ServiceNow tickets;
- cumulative artifacts từ Days 1–13;
- account/asset datasets;
- automation scripts;
- backup/recovery points;
- fault packets;
- known-good baselines.

Không được tự ý xử lý ngoài scope.

---

# 3. Infrastructure Health Verification

Đầu tiên phải thực hiện health checks trên toàn stack.

### Linux/RHEL

Kiểm tra:

- host;
- uptime;
- resources;
- service;
- networking;
- disk;
- logs.

### Windows Server

Kiểm tra:

- services;
- processes;
- storage;
- networking;
- Windows Event Logs;
- scheduled tasks;
- update/reboot state khi relevant.

### AWS

Kiểm tra:

- active identity;
- IAM;
- EC2;
- VPC path;
- security controls;
- EBS;
- S3;
- CloudWatch;
- relevant CloudTrail events.

### HPE/iLO

Kiểm tra:

- hardware health;
- IML;
- power;
- storage;
- firmware baseline.

### Nutanix

Kiểm tra:

- cluster;
- host;
- CVM;
- VM;
- capacity;
- alerts;
- tasks.

### HYCU

Kiểm tra:

- protection status;
- latest job;
- recovery points;
- target.

### Network

Kiểm tra:

- link;
- VLAN;
- trunk;
- MAC;
- ARP;
- routes;
- L2/L3 reachability.

### FortiGate

Kiểm tra:

- interfaces;
- routes;
- policy;
- session;
- logs.

### BIG-IP

Kiểm tra:

- virtual server;
- pool;
- members;
- health monitors;
- SSL/TLS.

---

# 4. Failure Scenarios

Syllabus yêu cầu bạn phải investigate các scenario thuộc nhiều layer.

Ít nhất phải review các nhóm fault sau:

- service failure;
- permission failure;
- routing failure;
- firewall failure;
- SSL certificate failure;
- VM availability failure;
- storage capacity failure;
- backup failure;
- privileged-access issue;
- logging issue.

Mỗi scenario phải được xử lý theo logic:

`Symptom`
→ `Hypothesis`
→ `Read-only checks`
→ `Evidence`
→ `Root cause`
→ `Authorized correction`
→ `Verification`
→ `Rollback/escalation`.

---

# 5. Server Log Extraction

Additive Day 14 quy định mỗi learner phải trực tiếp thực hiện ít nhất một **server-log extraction**.

Bạn phải chứng minh được:

- đúng host;
- đúng timezone;
- đúng time window;
- đúng filter;
- đúng source;
- đúng output;
- count;
- hash;
- ticket linkage.

---

# 6. Approved Corrective Operation hoặc Rollback

Mỗi learner phải thực hiện một thao tác corrective hoặc rollback có kiểm soát.

Ví dụ conceptually có thể là:

- restart approved service;
- restore configuration;
- correct lab permission;
- revert safe firewall fixture;
- restore known-good configuration;
- recover failed automation configuration.

Quan trọng không phải độ "hoành tráng" của change mà là:

- authorization;
- precheck;
- execution;
- postcheck;
- evidence;
- rollback readiness.

---

# 7. Isolated Recovery Check

Mỗi learner phải thực hiện ít nhất một recovery test.

Có thể liên quan:

- HYCU;
- S3;
- VM/file;
- backup dataset.

Bắt buộc phải:

- chọn đúng recovery point;
- restore vào isolated destination nếu cần;
- tránh identity/IP conflict;
- verify content;
- compare source/manifest;
- record recovery duration;
- preserve evidence.

---

# 8. IAM Exception Decision

Bạn phải xử lý ít nhất một IAM exception.

Ví dụ fixture có thể là:

- stale privileged entitlement;
- missing owner;
- missing approval;
- revoked-but-active account;
- dormant user;
- service account;
- break-glass account;
- unknown last-used evidence.

Decision không chỉ "pass/fail".

Các disposition có thể gồm:

- keep;
- revoke;
- modify;
- investigate.

Mỗi quyết định phải có:

- rationale;
- owner;
- reviewer;
- due date;
- evidence;
- linked task/ticket.

---

# 9. Automation Regression Run

Mỗi learner phải chạy ít nhất một regression run đối với automation từ Day 12–13.

Có thể gồm:

- VBA;
- Python;
- Batch;
- PowerShell.

Phải verify:

- source inputs unchanged;
- expected outputs;
- record counts;
- exception classification;
- exit status;
- repeatability;
- no duplicate side effects;
- evidence hashes.

---

# 10. Condensed IAM / Access Certification

Bạn sẽ chạy phiên bản rút gọn của 90-day review.

Workflow gồm:

`Import datasets`

→ `normalize identities`

→ `join by documented keys`

→ `detect unmatched/duplicate/stale/orphan`

→ `review entitlement`

→ `make reviewer decision`

→ `link evidence`

→ `record exception`.

Không được match identity bằng display name đơn thuần.

---

# 11. Vulnerability Reconciliation

Bạn phải reconcile supplied vulnerability findings với asset inventory.

Phải xác định:

- affected asset;
- version;
- CVE nếu có;
- severity source;
- exposure;
- business criticality;
- remediation priority;
- remediation owner;
- exception;
- retest status.

---

# 12. Centralized Log Correlation

Bạn phải correlate server/network/cloud/security records.

Ví dụ incident timeline có thể kết hợp:

`Windows/Linux event`

→ `FortiGate traffic`

→ `BIG-IP pool status`

→ `CloudWatch metric`

→ `CloudTrail management event`

→ `ServiceNow change`.

Mục tiêu là xây dựng **timeline có chứng cứ**, không đoán theo correlation cảm tính.

---

# 13. ServiceNow Updates

Trong suốt lab phải cập nhật:

- investigation;
- evidence references;
- approval;
- change implementation;
- rollback;
- status;
- unresolved issue;
- assignment;
- closure/handover.

Ticket phải phản ánh được điều gì thực sự xảy ra.

---

# 14. Asset Inventory Update

Nếu phát hiện discrepancy, phải:

- identify discrepancy;
- verify source of truth;
- document authorized correction;
- capture before/after evidence;
- link ticket.

Không được tự sửa CMDB/asset record chỉ để "khớp dữ liệu".

---

# 15. Peer / Mentor Review

Day 14 yêu cầu peer hoặc mentor review:

- configuration;
- security;
- evidence;
- automation outputs;
- source-to-report traceability.

Một peer phải có khả năng lấy một transformation đã chọn và **reproduce** được kết quả từ source input.

Đây là kiểm tra rất quan trọng về auditability.

---

# 16. Rotating Stations / Fault Packets

Không nhất thiết mỗi learner tự tay recreate toàn bộ mọi fault trong Day 14.

Syllabus cho phép dùng:

- rotating stations;
- prepared fault packets;
- fixture-based scenarios.

Nhưng **mọi original scenario phải được review** trong Day 14.

Điều này giúp kiểm tra rộng toàn bộ curriculum mà không giả vờ rằng 6 giờ lab có thể thực hiện độc lập hàng chục enterprise systems thực tế.

---

# 17. Final Evidence Package

Đầu ra Day 14 không chỉ là "lab completed".

Bạn phải có một final acceptance package gồm:

- final architecture;
- configuration inventory;
- runbooks;
- operational checklist;
- test results;
- audit evidence;
- risk register;
- acceptance package;
- signed skill matrix;
- evidence manifest;
- residual-gap register;
- reviewer decisions;
- test summary.

Đây là deliverable chính thức của Day 14. Infrastructure_Operations_14_Da…

---

# 18. Evidence Quality Requirements

Final package phải bảo đảm:

**Raw evidence preserved**

→ **working copy identifiable**

→ **transformation traceable**

→ **counts reconciled**

→ **hash recorded**

→ **ticket relationship known**

→ **reviewer can reproduce**.

Screenshot chỉ nên bổ sung cho evidence; không được âm thầm thay thế exportable records nếu không có lý do. Infrastructure_Operations_14_Da…

---

# 19. Automation Acceptance

Automation phải được kiểm tra ít nhất theo các nhóm:

- identity matching;
- encoding/file handling;
- file/process failure;
- IAM boundary;
- API/network/TLS failures;
- password-generation safeguards nếu applicable;
- Outlook/report handling;
- repeatability/regression.

Một script "run hết không crash" **không đồng nghĩa test pass**.

Acceptance phải so sánh:

**Expected result ↔ Actual result**. Infrastructure_Operations_14_Da…

---

# 20. Safety & Stop Conditions

Day 14 phải tuân thủ nguyên tắc:

**read-only first → reversible action → configuration change only when authorized.**

Các hoạt động high-risk như:

- production firewall change;
- firmware update;
- Nutanix cluster maintenance;
- privileged-account operation;

đều cần authorization, rollback planning và supervision phù hợp. Infrastructure_Operations_14_Da…

---

# 21. Final Acceptance / Sign-off

Day 14 chỉ được coi là đạt khi:

- original assignments đã được account đầy đủ;
- mọi skill row có evidence hoặc explicit gap;
- mandatory acceptance checks pass;
- seeded critical faults được phát hiện hoặc escalate đúng;
- no unexplained evidence discrepancy;
- no unresolved critical integrity issue;
- mentor/reviewer sign-off theo scope.

Đặc biệt, syllabus nói rõ các trường hợp sau **không được im lặng bỏ qua**:

- missing privileged-access revocation proof;
- backup restore chưa verify;
- critical automation tests chưa làm;
- integrity checks chưa hoàn thành.

Chúng phải giữ ở trạng thái **open gap** cho đến khi có owner/action rõ ràng. Infrastructure_Operations_14_Da…

---

# Bản đồ Day 14 cô đọng

Có thể hình dung toàn bộ Day 14 bằng chuỗi sau:

```text
                     DAY 14 CAPSTONE

        ┌────────────────────────────────────┐
        │ Approved Ticket / Change / Access  │
        └────────────────┬───────────────────┘
                         │
                         ▼
              Environment / Baseline
                         │
         ┌───────────────┼────────────────┐
         ▼               ▼                ▼
   Server/Cloud      Network/Security   HCI/Backup
 Linux / Windows      Cisco/FortiGate    HPE/Nutanix
 AWS                  BIG-IP             HYCU
         └───────────────┼────────────────┘
                         ▼
                 Symptom / Incident
                         │
                         ▼
             Read-only Verification
                         │
                         ▼
               Evidence Collection
                         │
                         ▼
              Root Cause Isolation
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
     Authorized Correction       Escalation
              │
              ▼
         Recovery / Rollback
              │
              ▼
        Post-change Verification
              │
              ▼
      Logs + IAM + Vulnerability
              │
              ▼
     Automation Regression Test
              │
              ▼
   ServiceNow / Asset Reconciliation
              │
              ▼
     Peer Review / Reproducibility
              │
              ▼
          Shift Handover
              │
              ▼
     Final Evidence & Acceptance
```

Vì vậy, **Module Lý thuyết** của Day 14 sẽ dạy bạn _cách suy nghĩ và ra quyết định khi vận hành hệ thống tích hợp_, còn **Module Thực hành** sẽ bắt bạn chứng minh khả năng đó bằng một chuỗi incident/change/recovery/IAM/automation/evidence thực tế, có verification, rollback, escalation và handover đầy đủ. Đây chính xác là mục tiêu final technical readiness của syllabus Day 14. Infrastructure_Operations_14_Da…

Khi bạn yêu cầu **“Trình bày chi tiết Module Lý thuyết Day 14”**, tôi sẽ bắt đầu đào sâu toàn bộ Module 1 theo từng domain, dependency flow, troubleshooting logic, verification, evidence, security và escalation boundary. Sau đó **Module Thực hành Day 14** sẽ được xây thành một capstone lab hoàn chỉnh theo kiểu enterprise runbook.
