# Day 11 — Module Lý thuyết

## ITSM, ServiceNow, Privileged Access, PalletControl & Enterprise Operations Governance

Day 11 nằm ở giao điểm giữa **technical operations** và **operational governance**. Từ Day 1–10, phần lớn công việc tập trung vào việc xác định trạng thái infrastructure, thực hiện administration, troubleshooting, backup/recovery, network/security checks và log investigation. Day 11 bổ sung một lớp cực kỳ quan trọng: **làm thế nào để những hành động kỹ thuật đó được request, authorize, assign, execute, audit, reconcile, close và hand over một cách kiểm soát trong enterprise**.

Theo syllabus, toàn bộ Module Lý thuyết Day 11 bao gồm ba trụ cột: **ServiceNow/ITSM**, **Privileged Identity & Access Management**, và **PalletControl/IT Asset Management**; đồng thời phải hiểu cách liên kết ticket, approval, privileged access, CI/asset, logs và audit evidence thành một chuỗi operations có thể truy vết. Infrastructure_Operations_14_Da…

Tôi sẽ giảng module này theo 4 phần lớn:

1. **ITSM & ServiceNow Operations**
2. **Privileged Identity & Access Management**
3. **PalletControl & IT Asset Management**
4. **Cross-system operational governance, evidence, troubleshooting và escalation**

---

# PHẦN I — IT SERVICE MANAGEMENT & SERVICENOW

## 1. IT Service Management là gì?

### What

**IT Service Management — ITSM** là tập hợp các process, practice, role và control dùng để quản lý việc cung cấp và vận hành IT services.

ITSM không đơn giản là:

> “Có lỗi thì mở ticket.”

Trong enterprise, ITSM giải quyết những câu hỏi lớn hơn:

- Service nào đang bị ảnh hưởng?
- User/business nào bị ảnh hưởng?
- Ai đang chịu trách nhiệm?
- Mức độ ưu tiên là bao nhiêu?
- Technical investigation đã làm gì?
- Có cần approval không?
- Thay đổi infrastructure có được phép không?
- Maintenance window nào được dùng?
- Rollback như thế nào?
- Service đã thực sự recover chưa?
- Evidence nào chứng minh điều đó?
- Có cần escalation?
- Shift tiếp theo cần biết gì?
- Asset/CMDB có cần update không?

Do đó ITSM đóng vai trò **control plane của operational work**.

---

# 2. Technical operation và ITSM operation khác nhau thế nào?

Ví dụ một Linux application service bị down.

Technical perspective:

```text
systemctl status app.service
journalctl -u app.service
systemctl restart app.service
curl https://app.example
```

Nhưng enterprise operations còn yêu cầu:

```text
Incident exists?
        ↓
Correct CI?
        ↓
Assignment group?
        ↓
Impact/Urgency/Priority?
        ↓
Troubleshooting authorized?
        ↓
Restart allowed under runbook?
        ↓
Change required?
        ↓
Privileged access approved?
        ↓
Action executed
        ↓
Service verified
        ↓
Logs/evidence collected
        ↓
Ticket updated
        ↓
Resolution/closure/handover
```

Một engineer có thể sửa technical problem đúng nhưng vẫn tạo ra **operational failure** nếu:

- không có authorization;
- dùng account không được phép;
- không ghi ticket;
- không biết CI nào bị tác động;
- không thu evidence;
- restart ngoài maintenance window;
- không có backout plan;
- đóng Incident khi service chưa thực sự ổn định.

Đây là mindset trung tâm của Day 11.

---

# 3. ServiceNow trong Day 11

Theo syllabus, ServiceNow là nền tảng ITSM chính cần hiểu trong Day 11.

Tuy nhiên, chúng ta phải tách hai khái niệm:

**ServiceNow product knowledge**  
và  
**ITSM operational knowledge**.

Một organization có thể customize:

- form;
- state model;
- priority matrix;
- mandatory fields;
- assignment rules;
- approval flow;
- SLA;
- change workflow;
- ACL;
- retention;
- audit configuration.

Do đó không được nghĩ:

> “Tôi đã thấy ServiceNow ở công ty A, nên công ty B chắc giống vậy.”

Workflow phải được xác minh trên **actual instance + organization runbook**.

---

# 4. Operational mental model của ServiceNow

Để hiểu ServiceNow ở mức Infrastructure Operations, có thể hình dung các lớp:

```text
┌───────────────────────────────────┐
│ User / Operations / Service Desk  │
└────────────────┬──────────────────┘
                 │
                 ▼
┌───────────────────────────────────┐
│ Incident / Request / Change       │
│ Problem / Task / Approval         │
└────────────────┬──────────────────┘
                 │
                 ▼
┌───────────────────────────────────┐
│ Workflow / Assignment / SLA       │
│ Approval / Notifications          │
└────────────────┬──────────────────┘
                 │
                 ▼
┌───────────────────────────────────┐
│ CI / CMDB / Asset / User / Group  │
└────────────────┬──────────────────┘
                 │
                 ▼
┌───────────────────────────────────┐
│ Audit / Journal / Attachments     │
│ Integration / External Systems    │
└───────────────────────────────────┘
```

Đây là **operational model**, không phải claim về internal product architecture cụ thể của mọi ServiceNow release.

---

# 5. Incident

## What

**Incident** là record quản lý một sự cố làm gián đoạn hoặc suy giảm service.

Ví dụ:

- application không truy cập được;
- server service stopped;
- user không SSH được;
- firewall đang block application flow ngoài dự kiến;
- BIG-IP pool member unhealthy;
- backup job unexpectedly failed.

### Mục tiêu của Incident

Mục tiêu chính:

> **Restore normal service càng sớm và an toàn càng tốt.**

Incident investigation không nhất thiết phải tìm ra ultimate root cause ngay.

Ví dụ:

```text
Symptom:
Application unavailable

Immediate restoration:
Restart failed application worker

Root cause:
Memory leak chưa xác định hoàn toàn
```

Incident có thể resolved sau khi service phục hồi, trong khi root-cause work tiếp tục qua Problem process.

---

# 6. Problem

**Problem** tập trung vào nguyên nhân hoặc potential root cause của một hoặc nhiều Incident.

Ví dụ:

```text
INC001:
Application crashed Monday

INC002:
Application crashed Wednesday

INC003:
Application crashed Friday
```

Có thể tạo:

```text
PRB001:
Recurring application worker crash
```

Problem investigation có thể bao gồm:

- pattern analysis;
- root cause analysis;
- known error;
- workaround;
- permanent remediation.

Khác biệt quan trọng:

```text
Incident → restore service
Problem  → understand/remove underlying cause
```

Không phải mọi Incident đều nhất thiết tạo Problem.

---

# 7. Request / Service Request

**Request** là yêu cầu cung cấp một service, access hoặc resource được định nghĩa.

Ví dụ:

- request new VM;
- request software installation;
- request firewall access;
- request temporary privileged access;
- request group membership.

Request khác Incident.

Nếu user nói:

> “Hãy cấp cho tôi admin access vào server.”

Đó không phải Incident chỉ vì user hiện chưa có quyền.

Đó thường là **access/service request**.

---

# 8. Change

## What

**Change** quản lý một modification có chủ đích đối với environment.

Ví dụ:

- update firewall policy;
- modify routing;
- change application configuration;
- restart production cluster theo maintenance plan;
- replace certificate;
- change IAM entitlement;
- update software;
- modify service configuration.

Change Management trả lời:

> “Chúng ta có nên thực hiện thay đổi này không, khi nào, bởi ai, với risk gì, và nếu thất bại thì làm sao?”

---

# 9. Incident và Change có thể tồn tại cùng nhau

Ví dụ:

Production application bị down.

Có Incident:

```text
INC1001 — Production application unavailable
```

Investigation xác định cần sửa production configuration.

Theo policy, modification cần Change:

```text
CHG2001 — Correct production application configuration
```

Mối quan hệ:

```text
Incident
   │
   ├── Troubleshooting
   │
   └── requires production modification
                   │
                   ▼
                Change
                   │
             Approval/Window
                   │
             Implementation
                   │
              Verification
                   │
                   ▼
           Incident resolution
```

Incident không thay thế Change.

Change cũng không thay thế Incident.

---

# 10. Task

**Task** đại diện cho một đơn vị công việc cần được một cá nhân/group thực hiện.

Ví dụ một Change lớn có thể có:

```text
Change
 ├─ Task 1: Network pre-check
 ├─ Task 2: Application shutdown
 ├─ Task 3: Infrastructure change
 ├─ Task 4: Application validation
 └─ Task 5: Business verification
```

Task giúp:

- chia ownership;
- sequencing;
- tracking;
- evidence;
- accountability.

Một lỗi phổ biến là chỉ nhìn parent ticket và không kiểm tra child tasks.

---

# 11. Approval

**Approval** là record quyết định cho phép hoặc từ chối một action/workflow.

Các approval có thể liên quan đến:

- business owner;
- service owner;
- asset owner;
- security;
- change authority;
- manager;
- data owner;
- IAM owner.

Điểm cực kỳ quan trọng:

> **Approval không đồng nghĩa action đã xảy ra.**

Ví dụ:

```text
Request = Approved
```

không có nghĩa:

```text
IAM role assignment = Active
```

Ngược lại:

```text
Access removed in IAM
```

không chứng minh ServiceNow workflow đã được correctly closed.

Cần phân biệt:

```text
Workflow state
Effective technical state
```

---

# 12. Knowledge

**Knowledge** dùng để lưu operational knowledge có thể tái sử dụng.

Ví dụ:

- SOP;
- troubleshooting guide;
- known issue;
- runbook;
- FAQ;
- access instructions;
- escalation procedure.

Knowledge tốt giúp tránh kiểu vận hành:

> “Chỉ một người trong team biết cách xử lý.”

Nhưng engineer phải kiểm tra:

- article có còn valid không;
- applicable version;
- environment;
- owner;
- last review;
- required approval.

Runbook cũ đôi khi nguy hiểm hơn không có runbook.

---

# 13. Asset

**Asset** đại diện cho một resource mà organization cần quản lý về ownership và lifecycle.

Ví dụ:

- physical server;
- laptop;
- storage appliance;
- licensed software;
- endpoint.

Asset management thường quan tâm:

```text
What is it?
Who owns it?
Where is it?
What is its lifecycle state?
Is it managed?
Is inventory current?
```

Chúng ta sẽ đi sâu phần này trong PalletControl.

---

# 14. Configuration Item — CI

**Configuration Item — CI** là một thành phần cần được quản lý về configuration và relationship vì nó hỗ trợ một service.

Ví dụ:

```text
Application
   │
   ▼
Virtual Server
   │
   ▼
VM
   │
   ▼
AHV Cluster
   │
   ▼
Physical Node
```

Hoặc:

```text
Web Application
      │
      ▼
BIG-IP Virtual Server
      │
      ▼
Backend Pool
      │
      ▼
Application Servers
```

Các object có thể được model thành CIs và relationships.

CI giúp Incident responder trả lời:

> “Đây không chỉ là host `srv01`; host này đang phục vụ service nào?”

---

# 15. CMDB

**Configuration Management Database — CMDB** lưu CIs cùng relationships và attributes liên quan.

Một CMDB tốt cho phép operational reasoning:

```text
Affected Server
       ↓
Hosted Application
       ↓
Business Service
       ↓
Business Owner
```

hoặc:

```text
Network Device
       ↓
Dependent Links
       ↓
Dependent Systems
```

CMDB đặc biệt hữu ích trong:

- impact assessment;
- Incident triage;
- Change planning;
- outage correlation;
- ownership;
- audit;
- asset reconciliation.

---

# 16. Asset và CI không phải cùng một khái niệm

Đây là điểm dễ nhầm.

### Asset view

Tập trung vào:

- ownership;
- procurement;
- lifecycle;
- inventory;
- financial/administrative tracking.

### CI view

Tập trung vào:

- technical configuration;
- relationship;
- service dependency;
- operational impact.

Ví dụ một physical server có thể vừa là:

```text
Asset record
```

vừa có:

```text
Server CI
```

Nhưng hai record phục vụ mục đích khác nhau.

---

# 17. Ticket lifecycle

Syllabus yêu cầu hiểu **ticket lifecycle**.

Một operational lifecycle khái quát:

```text
Create
  ↓
Categorize
  ↓
Prioritize
  ↓
Assign
  ↓
Acknowledge
  ↓
Investigate
  ↓
Take approved action
  ↓
Verify
  ↓
Resolve
  ↓
Close
```

Tuy nhiên exact states khác nhau giữa organizations.

Ví dụ organization có thể có:

```text
New
Assigned
In Progress
Pending
Resolved
Closed
```

Hoặc workflow khác.

Không được ghi nhớ một state sequence rồi coi đó là universal ServiceNow behavior.

---

# 18. Assignment Group

**Assignment Group** đại diện team chịu trách nhiệm xử lý công việc.

Ví dụ:

- Windows Operations;
- Linux Operations;
- Network Operations;
- Cloud Operations;
- Security Operations;
- Database Team;
- Application Support.

Có hai mức ownership thường cần quan sát:

```text
Assignment Group
       +
Assignee
```

### Assignment Group

Ai chịu trách nhiệm ở team level?

### Assignee

Engineer cụ thể nào đang xử lý?

---

# 19. Ownership quan trọng thế nào?

Ticket không có clear ownership dễ xảy ra:

```text
Team A → nghĩ Team B xử lý
Team B → nghĩ Team C xử lý
Team C → không biết ticket
```

Kết quả:

- SLA breach;
- incident kéo dài;
- duplicated troubleshooting;
- escalation chậm;
- audit ambiguity.

Operational rule:

> Một ticket có thể có nhiều collaborator, nhưng accountability phải rõ.

---

# 20. Impact

**Impact** phản ánh phạm vi/mức độ ảnh hưởng của vấn đề đối với users/services/business.

Ví dụ:

```text
Impact thấp:
1 test user

Impact cao:
Production authentication service toàn company
```

Impact không chỉ là “bao nhiêu máy lỗi”.

Phải xem:

- user population;
- business criticality;
- production/non-production;
- service dependency;
- security implications;
- revenue/regulatory impact.

---

# 21. Urgency

**Urgency** phản ánh mức độ cần xử lý nhanh.

Ví dụ:

Một certificate:

```text
Expires in 50 days
```

khác:

```text
Expires in 30 minutes
```

Impact có thể giống nhau nhưng Urgency khác.

---

# 22. Priority

**Priority** thường được xác định từ combination của:

```text
Impact + Urgency
```

nhưng exact priority matrix là organization-specific.

Ví dụ conceptual:

| Impact | Urgency | Possible result    |
| ------ | ------- | ------------------ |
| High   | High    | Critical/very high |
| High   | Low     | Medium/high        |
| Low    | High    | Medium             |
| Low    | Low     | Low                |

Không được tự áp matrix trên vào production nếu organization có matrix khác.

---

# 23. Một lỗi phổ biến: severity = priority

Không luôn đúng.

Security finding:

```text
CVSS = High
```

không tự động có nghĩa:

```text
Incident priority = P1
```

Infrastructure Operations phải xem:

- exposure;
- asset criticality;
- actual exploitation;
- affected service;
- compensating control;
- business impact.

Day 10 đã giới thiệu distinction giữa severity và business risk; Day 11 áp tư duy đó vào ticket governance.

---

# 24. SLA Awareness

**Service Level Agreement — SLA** trong operational context có thể đặt target đối với:

- response;
- acknowledgement;
- resolution;
- fulfillment;
- other milestones.

Điểm cần hiểu không phải là nhớ một con số SLA.

Phải biết SLA có thể phụ thuộc:

- Priority;
- service;
- support schedule;
- business calendar;
- pause conditions;
- ticket state;
- contractual rules.

Ví dụ không được giả định:

> “P2 lúc nào cũng phải resolve trong 4 giờ.”

Đó là policy-specific.

---

# 25. SLA breach không phải chỉ là timer problem

Khi ticket gần breach cần hỏi:

```text
Is ownership correct?
Is priority correct?
Is ticket waiting for approval?
Is vendor engaged?
Is customer input pending?
Is SLA legitimately paused?
Is escalation required?
```

Không nên thay state chỉ để “stop SLA clock” nếu không đúng thực tế.

Đó là audit/control violation.

---

# 26. Work Notes và Customer-visible Comments

Syllabus đặc biệt yêu cầu phân biệt hai loại communication.

## Work Notes

Thường dùng cho internal operational detail:

- investigation;
- commands/results;
- internal coordination;
- escalation;
- risk;
- diagnostic reasoning.

## Customer-visible Comments

Dùng để giao tiếp với requester/end user theo appropriate level.

Ví dụ customer comment:

```text
The service has been restored and validation is in progress.
```

Internal work note có thể chứa:

```text
Restarted application worker after confirming CHG approval.
Health check HTTP 200 from nodes A/B.
Monitoring stable for 15 minutes.
```

Visibility thực tế vẫn phụ thuộc instance configuration/ACL.

---

# 27. Work Note tốt phải trả lời được gì?

Một useful work note nên cho engineer tiếp theo biết:

```text
When?
Who?
What was observed?
What was checked?
What was changed?
Under what authorization?
What was the result?
What is next?
```

Ví dụ kém:

```text
Checked server. Looks OK.
```

Ví dụ tốt hơn:

```text
2026-10-21 13:20 ICT:
Verified app01 CPU/memory/disk healthy.
Service api.service was inactive.
Reviewed journal for 13:00–13:20 ICT.
No configuration change performed.
Escalating to Application Support due repeated startup failure.
```

Điểm cốt lõi là reproducibility và handover.

---

# 28. Approval Gate

**Approval gate** là một điểm workflow yêu cầu approval trước khi tiếp tục.

Conceptual flow:

```text
Request
   ↓
Technical review
   ↓
Approval Gate
   ├── Reject → Stop
   │
   └── Approve
          ↓
     Implementation
```

Approval gate giúp ngăn:

- unauthorized access;
- unauthorized change;
- uncontrolled production work;
- SoD violations.

---

# 29. Approval phải phù hợp scope

Một approval cho:

```text
Read-only diagnostic access
```

không có nghĩa engineer được:

```text
Modify firewall policy
```

Một approval cho:

```text
server01
```

không tự động cover:

```text
server02
```

Một approval valid hôm qua có thể không valid hôm nay nếu:

- expired;
- ticket closed;
- maintenance window passed;
- scope changed.

---

# 30. Maintenance Window

**Maintenance window** là time window được chấp thuận để thực hiện một activity có potential service impact.

Tại sao cần?

- stakeholder awareness;
- resource availability;
- reduced business impact;
- support team coordination;
- rollback time;
- vendor availability.

Không nên bắt đầu risky operation ngay cuối window nếu không còn đủ rollback time.

---

# 31. Change Risk

Change risk không đơn giản là:

```text
Low / Medium / High
```

Mà cần reasoning về:

- blast radius;
- reversibility;
- technical complexity;
- dependency;
- service criticality;
- previous failure history;
- maintenance duration;
- rollback confidence;
- monitoring readiness;
- available support.

Ví dụ:

Changing one lab VM:

```text
low blast radius
```

Changing production firewall shared policy:

```text
potentially high blast radius
```

---

# 32. Implementation Plan

Một implementation plan tốt phải đủ cụ thể để engineer khác hiểu đúng sequence.

Ví dụ structure:

```text
1. Confirm approval.
2. Confirm maintenance window.
3. Verify current service health.
4. Record current configuration.
5. Execute authorized change.
6. Validate expected configuration.
7. Perform technical health checks.
8. Perform application health checks.
9. Monitor for defined period.
10. Complete evidence.
```

Không được dùng vague instruction:

```text
Make configuration change.
```

---

# 33. Backout Plan / Rollback Plan

**Backout plan** mô tả cách trở về safe previous state nếu implementation thất bại.

Phải trả lời:

- Trigger nào bắt đầu rollback?
- Restore cái gì?
- Dùng source/config nào?
- Ai quyết định rollback?
- Thời gian cần bao lâu?
- Sau rollback verify cái gì?

Ví dụ:

```text
If health check fails after policy update:
1. Restore previous policy configuration.
2. Confirm policy ordering.
3. Clear only authorized test session if required.
4. Re-run connectivity validation.
5. Record rollback evidence.
```

---

# 34. “Có backup” không đồng nghĩa “có rollback”

Ví dụ:

Bạn có configuration backup.

Nhưng nếu:

- backup không restore được;
- version incompatible;
- dependencies thay đổi;
- rollback mất 3 giờ nhưng window còn 10 phút;

thì backup không tạo ra valid rollback plan.

Điều này liên hệ tư duy từ Nutanix, firewall, AWS và backup/DR các Day trước.

---

# 35. Stop Condition

Một change plan tốt phải có **stop condition**.

Ví dụ:

```text
STOP if:
- Pre-check fails.
- Approval cannot be verified.
- Current configuration differs from documented baseline.
- Unexpected critical alarm appears.
- Redundancy is degraded.
- Rollback path is unavailable.
```

Infrastructure Operations maturity thể hiện rất rõ ở khả năng:

> Biết khi nào **không nên tiếp tục**.

---

# 36. Resolution

**Resolution** nghĩa issue đã được xử lý đến mức ticket có thể chuyển sang resolved state theo policy.

Nhưng resolution phải dựa trên evidence.

Ví dụ:

```text
service process running
```

chưa chắc đủ.

Cần có thể verify:

```text
Service running
Port listening
Application response valid
Monitoring healthy
User path confirmed where required
```

---

# 37. Resolution Code

**Resolution Code** dùng để classify cách ticket được giải quyết.

Các exact values phụ thuộc organization.

Có thể gặp categories dạng:

- fixed;
- workaround;
- duplicate;
- user error;
- no fault found;
- configuration corrected.

Không được chọn resolution code tùy tiện chỉ để close ticket.

Nó ảnh hưởng:

- reporting;
- trend analysis;
- Problem identification;
- audit;
- KPI.

---

# 38. Resolved và Closed

Hai state này không nên coi là đồng nghĩa.

Conceptually:

```text
Resolved
   ↓
Verification / observation / user acceptance
   ↓
Closed
```

Tùy workflow organization, process có thể khác.

Điểm cần nhớ:

> **Resolved** là technical/process outcome; **Closed** là end-state của lifecycle theo governance.

---

# 39. Shift Handover

**Shift handover** cực kỳ quan trọng trong 24×7 operations.

Engineer bàn giao phải giúp shift tiếp theo tiếp tục mà không cần điều tra lại từ đầu.

Một handover nên truyền đạt:

```text
Ticket
Affected service/CI
Business impact
Current state
Timeline
Diagnostics completed
Changes performed
Evidence
Pending actions
Pending approvals
Risk
SLA status
Escalation status
Next recommended step
Owner
```

---

# 40. Handover failure

Các lỗi thường gặp:

### “Issue ongoing”

Quá ít thông tin.

### Không ghi negative result

Shift sau lại chạy cùng troubleshooting.

### Không ghi failed action

Shift sau lặp lại action rủi ro.

### Không ghi approval/window

Shift sau có thể thực hiện unauthorized work.

### Không ghi current owner

Ticket bị abandoned.

---

# 41. ServiceNow record types: Field, Journal, Approval, Attachment, Audit, History

Đây là additive coverage bắt buộc của syllabus.

Phải phân biệt ít nhất:

```text
Ticket field
Journal / work-note entry
Approval record
Attachment
Audit record
Display History
```

Infrastructure_Operations_14_Da…

---

# 42. Ticket Field

Ticket fields chứa structured state.

Ví dụ conceptual:

```text
Number
State
Assignment Group
Assigned To
Priority
Impact
Urgency
CI
Opened
Updated
Resolved
Closed
```

Field value là một snapshot của current state.

Ví dụ:

```text
Assignment Group = Linux Operations
```

chỉ nói assignment hiện tại.

Nó không nhất thiết cho bạn biết previous assignments nếu không xem history/audit.

---

# 43. Journal Entry

Journal-style information ghi chronology.

Ví dụ:

```text
Work Notes
Comments
```

Journal record trả lời:

> “Người xử lý đã ghi nhận gì trong quá trình ticket tồn tại?”

Nhưng journal không phải replacement cho technical evidence.

Ví dụ:

```text
"Firewall was updated successfully."
```

không mạnh bằng:

```text
Change reference
Configuration before/after
Connectivity test
Firewall logs
Timestamp
```

---

# 44. Approval Record

Approval record cung cấp evidence về:

- approver;
- decision;
- state;
- timestamp;
- related request/change.

Nhưng cần kiểm tra:

```text
Approval applies to which scope?
```

và:

```text
Was implementation after approval?
```

Timeline matters.

Ví dụ:

```text
Implementation: 10:00
Approval:       10:15
```

Rõ ràng approval không thể retroactively authorize action 10:00 trừ khi có documented emergency process.

---

# 45. Attachment

Attachment có thể chứa:

- change plan;
- log export;
- screenshot;
- report;
- configuration;
- approval support documentation.

Nhưng phải quan tâm:

- scope;
- confidentiality;
- malware/content controls;
- access permissions;
- retention;
- file integrity.

Không upload secret chỉ vì “ticket private”.

---

# 46. Audit Record và Display History

Đây là điểm quan trọng nhất của phần ServiceNow evidence.

Syllabus cảnh báo:

> History không được mặc định xem như complete audit archive. Infrastructure_Operations_14_Da…

ServiceNow documentation được syllabus tham chiếu giải thích rằng trong release **Xanadu**, `Audit [sys_audit]`, `History Sets [sys_history_set]` và `History [sys_history_line]` phục vụ mục đích khác nhau; History Set/History được tạo từ Audit data khi cần và chỉ chứa một subset lịch sử gần đây thay vì toàn bộ audit history. [servicenow.com](https://www.servicenow.com/docs/r/xanadu/platform-security/c_DiffBtwnAuditHistSets.html)

Quan trọng: đó là tài liệu release-specific được syllabus dùng để minh họa distinction. Khi vận hành instance thật phải kiểm tra:

- actual ServiceNow release;
- auditing configuration;
- table/field auditing;
- ACL;
- retention;
- archive behavior.

Không lấy retention number của một release rồi áp vào mọi instance.

---

# 47. ACL — Access Control List

**ACL** kiểm soát ai có thể access data/action.

Trong ServiceNow context, ACL có thể ảnh hưởng:

- table access;
- record access;
- field access;
- export availability.

Do đó:

```text
“I cannot see audit field”
```

không có nghĩa:

```text
“Audit field does not exist.”
```

Có thể là permission issue.

Đây là general troubleshooting principle:

```text
Not visible ≠ Not present
```

---

# 48. “No evidence found” là một kết luận nguy hiểm

Khi không thấy event phải phân biệt:

```text
Event did not happen
```

với:

```text
Not audited
Retention expired
ACL denies access
Wrong time range
Wrong timezone
Wrong record
Wrong table
Export incomplete
History view incomplete
```

Đây là continuation trực tiếp của Day 10:

> **No result ≠ No event.**

---

# PHẦN II — PRIVILEGED IDENTITY & ACCESS MANAGEMENT

Day 11 không chỉ nói IAM chung chung. Syllabus yêu cầu **privileged identity lifecycle**, account ownership, authorization, access request/approval, session/gateway, credential checkout/rotation, AWS IAM, access review, SoD, dormant/orphaned accounts, break-glass và privileged activity auditing. Infrastructure_Operations_14_Da…

---

# 49. Identity, Account, Credential và Entitlement

Bốn term này phải tách rõ.

### Identity

Người hoặc workload là ai.

```text
Alice Nguyen
Backup Service
Automation Runner
```

### Account

Technical identity tồn tại trên system.

```text
alice.admin
svc_backup
IAM user
```

### Credential

Thứ được dùng để authenticate.

Ví dụ:

- password;
- SSH private key;
- certificate;
- token;
- temporary credentials.

### Entitlement

Quyền được cấp.

Ví dụ:

```text
Administrators group
sudo privilege
IAM role
Database role
Firewall admin role
```

Quan hệ:

```text
Human Identity
      ↓
Account
      ↓
Credential
      ↓
Entitlements
      ↓
Target Resource
```

---

# 50. Privileged Identity

**Privileged Identity** là identity có khả năng thực hiện action có ảnh hưởng cao.

Ví dụ:

- OS administrator;
- domain administrator;
- cloud administrator;
- firewall administrator;
- network administrator;
- database administrator.

Privilege không chỉ có nghĩa “root”.

Một account có thể rất dangerous nếu có quyền:

```text
Modify IAM policies
```

dù không đăng nhập được Linux root.

---

# 51. Privileged ID Lifecycle

Syllabus yêu cầu nắm toàn bộ lifecycle:

```text
Need identified
      ↓
Request
      ↓
Identity verification
      ↓
Ownership verification
      ↓
Authorization
      ↓
Approval
      ↓
Provision / assign
      ↓
Use
      ↓
Audit
      ↓
Periodic review
      ↓
Expiry / revoke
```

Không được tập trung chỉ vào:

```text
“User login được chưa?”
```

---

# 52. Account Ownership

Mỗi privileged account phải có clear owner.

Owner có thể là:

- human owner;
- application owner;
- service owner;
- technical team.

Đặc biệt với service account:

```text
svc_backup
```

phải biết:

- business/technical purpose;
- responsible owner;
- systems using it;
- credential management method;
- rotation owner;
- lifecycle status.

---

# 53. Authorization

**Authentication** và **Authorization** khác nhau.

Authentication:

> “Bạn là ai?”

Authorization:

> “Bạn được phép làm gì?”

Ví dụ:

```text
Login successful
```

không có nghĩa:

```text
Allowed to modify firewall policies.
```

Infrastructure Operations troubleshooting phải xác định failure xảy ra ở đâu:

```text
Identity
Authentication
Authorization
Target configuration
Network path
```

---

# 54. Privileged Access Request

Một good access request phải đủ thông tin:

```text
Who?
Why?
Target?
Privilege level?
Duration?
Environment?
Ticket?
Business owner?
Approver?
```

Ví dụ không tốt:

```text
Need admin ASAP.
```

Ví dụ controlled:

```text
Requester: Engineer A
Target: prod-app01
Privilege: temporary sudo for service restart runbook
Reason: INC-1001
Duration: maintenance window only
Approval: Application owner + Operations lead
```

---

# 55. Least Privilege

**Least privilege**:

> Chỉ cấp quyền tối thiểu cần thiết để hoàn thành authorized task.

Ví dụ task là:

```text
Restart one service
```

Không nhất thiết cần:

```text
unrestricted root
```

nếu platform hỗ trợ scoped privilege.

Least privilege giảm:

- blast radius;
- accidental change;
- credential theft impact;
- insider risk;
- audit ambiguity.

---

# 56. Just-in-Time và Temporary Access

Một pattern thường gặp là quyền được cấp trong limited timeframe:

```text
Access Granted
      ↓
Maintenance
      ↓
Expiry
      ↓
Revoke
```

Lợi ích:

- giảm standing privilege;
- giới hạn attack window;
- align quyền với approved work.

Nhưng temporary access chỉ tốt nếu expiry/revocation thực sự hoạt động.

---

# 57. Approval Workflow trong privileged access

Conceptual workflow:

```text
Requester
    ↓
Business Justification
    ↓
Manager / Owner Approval
    ↓
Security / System Approval
    ↓
Provision
    ↓
Use
    ↓
Expiry/Revoke
```

Exact approval chain phụ thuộc organization.

Không được assume:

> Manager approval luôn đủ.

Một high-risk system có thể cần additional owner/security approval.

---

# 58. Privileged Session / Gateway

Một **privileged access gateway** có thể đóng vai trò mediation layer giữa administrator và target.

Mental model:

```text
Admin
  │
  ▼
Authentication
  │
  ▼
Approval/Policy Check
  │
  ▼
Privileged Gateway
  │
  ▼
Target System
```

Gateway có thể giúp giảm việc user trực tiếp sở hữu permanent privileged credentials.

---

# 59. Session Evidence

Một privileged session có thể cần evidence như:

```text
Session ID
User identity
Privileged account
Target
Start time
End time
Request/ticket reference
Outcome
```

Capabilities chi tiết như session recording hoặc command capture phụ thuộc product/version/configuration.

Không được tự giả định PS/Gateway hay AccessCheck có feature đó.

---

# 60. Credential Checkout

**Credential checkout** là workflow cấp quyền sử dụng credential trong controlled period.

Conceptually:

```text
Credential stored securely
      ↓
Approved user requests
      ↓
Checkout authorized
      ↓
Credential/session available temporarily
      ↓
Use
      ↓
Check-in / expiry
```

Control objectives:

- restrict who gets credential;
- restrict duration;
- record use;
- rotate where policy requires.

---

# 61. Credential Rotation

**Credential rotation** là thay credential theo policy hoặc event.

Có thể trigger bởi:

- scheduled policy;
- checkout/check-in;
- personnel change;
- exposure;
- emergency use;
- compromise.

Điểm quan trọng:

> Rotation thành công phải được xác minh ở target system.

Một vault nói:

```text
Rotation Successful
```

nhưng application dùng old credential và bị outage thì technically chưa ổn.

---

# 62. Rotation Dependency

Service account password rotation đặc biệt rủi ro.

Ví dụ:

```text
svc_app
   ├── Windows Service
   ├── Scheduled Task
   └── Application Connector
```

Nếu rotate password mà chỉ update một location:

```text
Application failure
```

Do đó trước rotation phải hiểu dependency.

---

# 63. Rotation Failure

Khi rotation fail, không được phản ứng:

> “Đặt password thủ công cho nhanh.”

Phải kiểm tra:

```text
Which account?
Which target?
Last successful rotation?
Current account state?
Credential store state?
Network path?
Permissions?
Dependency?
Ticket?
Runbook?
```

Rồi mới quyết định authorized remediation/escalation.

---

# 64. PS/Gateway và AccessCheck

Syllabus đưa tên:

- **PS/Gateway**
- **AccessCheck**

nhưng không xác định vendor/product architecture cụ thể.

Đây là giới hạn nguồn rất quan trọng.

Syllabus yêu cầu phải xác minh **exact vendor, product/version và approved runbook** trước khi dạy hoặc thực hiện product-specific steps; không được infer undocumented capabilities. Infrastructure_Operations_14_Da…

Vì vậy ở Module Lý thuyết chúng ta chỉ khẳng định generic workflow:

```text
Request
Approval
Authorized access
Access use
Expiry/revocation
Audit/reference
Failed-access escalation
Rotation handling
```

Menu, CLI, API, screen name, session recording capability... phải chờ environment thật.

---

# 65. Một nguyên tắc quan trọng về product proficiency

Nếu bạn chỉ làm:

```text
fixture / screenshot / simulated workflow
```

thì bạn chứng minh:

```text
workflow understanding
```

chứ không chứng minh actual product hands-on proficiency.

Syllabus quy định rõ actual product hands-on cần actual lab và instructor sign-off. Infrastructure_Operations_14_Da…

---

# 66. AWS Console IAM Identity Access

AWS IAM đã được học ở Day 4. Day 11 nhìn AWS dưới góc **privileged governance**.

Cần phân biệt:

```text
Identity
Credential
Role
Permission
Session
Approval
Activity
```

Day 11 không chỉ hỏi:

> “User login AWS được không?”

Mà hỏi:

> “User này có business authorization cho permission hiện tại không?”

---

# 67. IAM User vs Role

Conceptually:

### IAM user

Persistent IAM identity trong account.

### IAM role

Identity có permissions nhưng được assumed để có session/temporary credentials.

Trong enterprise hiện đại cũng có thể có federation/Identity Center, nhưng actual implementation phải xác minh.

Operational focus:

```text
Who is the human?
Which AWS identity/session?
Which role?
Which account?
Which approval?
Which permissions?
How long?
```

---

# 68. Role Assignment

Role assignment phải được kiểm soát vì role có thể mang privilege cao.

Review phải xem:

- role scope;
- trust relationship;
- effective permissions;
- requester;
- approval;
- duration;
- owner.

Không chỉ nhìn:

```text
Role name = Support
```

vì name không chứng minh actual permissions.

---

# 69. Permission vs Activity

Một identity có permission:

```text
ec2:TerminateInstances
```

không có nghĩa user đã thực hiện termination.

Ngược lại user không hoạt động gần đây không có nghĩa permission không tồn tại.

Phải tách:

```text
Entitlement evidence
```

và:

```text
Activity evidence
```

Điểm này sẽ cực kỳ quan trọng trong Day 13 IAM review.

---

# 70. Access Review

**Access Review** kiểm tra xem existing access còn appropriate không.

Review questions:

```text
Does the account still exist legitimately?
Is ownership valid?
Is access still required?
Is approval valid?
Is privilege excessive?
Is expiry correct?
Is account dormant?
Is account orphaned?
Is role appropriate?
Has access been revoked where required?
```

---

# 71. Access Review không chỉ là spreadsheet exercise

Kết luận:

```text
REMOVE
```

trong spreadsheet chưa có nghĩa quyền đã được remove.

Phải phân biệt:

```text
Reviewer decision
Remediation task
Technical execution
Effective access verification
```

Chuỗi đầy đủ:

```text
Review finding
     ↓
Decision
     ↓
Approved remediation
     ↓
Technical revoke
     ↓
Verify effective revoke
     ↓
Evidence
```

---

# 72. Segregation of Duties — SoD

**Segregation of Duties** ngăn một cá nhân có quá nhiều control trên cùng một critical process.

Ví dụ undesirable pattern:

```text
Person A:
Request access
Approve access
Provision access
Use access
Review own access
```

Điều đó tạo conflict of interest.

Một stronger model:

```text
Requester       → Person A
Approver        → Person B
Provisioner     → Team C
Reviewer        → Person D
```

Không phải mọi operation cần 4 người; exact SoD policy tùy organization.

---

# 73. SoD trong Change Management

SoD cũng có thể xuất hiện ở:

```text
Developer
   ↓
writes production change
   ↓
approves own change
   ↓
deploys own change
   ↓
validates own change
```

Risk cao hơn vì không có independent review.

Infrastructure Operations phải nhận diện conflict và follow policy, không tự “bypass để nhanh”.

---

# 74. Dormant Account

**Dormant account** là account không có observed activity trong một period được policy định nghĩa.

Điểm quan trọng:

```text
No observed activity
        ≠
Proof of inactivity
```

Lý do:

- incomplete audit;
- retention expired;
- activity tracked ở source khác;
- last-use signal khác last-login;
- service account activity pattern khác human user.

Day 13 sẽ đi sâu vào cutoff dates và 90-day review.

---

# 75. Dormant không đồng nghĩa disable ngay

Workflow đúng thường là:

```text
Finding
   ↓
Validate evidence
   ↓
Confirm owner
   ↓
Confirm business need
   ↓
Decision
   ↓
Approved action
```

Không:

```text
No login 90 days → immediately disable everything
```

unless policy specifically authorizes that automatic control.

---

# 76. Orphaned Account

**Orphaned account** là account không còn legitimate/known ownership hoặc không match expected identity lifecycle.

Ví dụ:

- employee left organization;
- project ended;
- owner unknown;
- system decommissioned;
- application replaced;
- service account tồn tại nhưng team không ai nhận ownership.

Orphaned privileged account là high-risk finding.

---

# 77. Duplicate Accounts

Một user có nhiều accounts không tự động là lỗi.

Ví dụ:

```text
alice
alice-admin
```

có thể là deliberate separation giữa standard account và privileged account.

Nhưng phải xác định:

```text
Purpose
Owner
Approval
Privilege
Lifecycle
```

Duplicate ID không được đánh giá chỉ bằng string matching.

---

# 78. Break-glass Access

**Break-glass access** dành cho emergency situation khi normal access path không usable hoặc không đủ.

Ví dụ:

- primary identity provider unavailable;
- privileged gateway outage;
- critical recovery scenario.

Break-glass account phải được quản lý chặt hơn ordinary account, không lỏng hơn.

---

# 79. Break-glass Controls

Typical controls cần hiểu:

```text
Restricted ownership
Strong credential protection
Clearly defined emergency conditions
Limited distribution
Alerting
Usage logging
Post-use review
Credential rotation after use
Ticket/incident linkage
```

Break-glass không có nghĩa:

> “Password root chung đặt trong Excel.”

---

# 80. Privileged Activity Auditing

Goal:

> Có thể truy vết privileged action đến identity, authorization và technical effect.

Một complete chain:

```text
Human Identity
      ↓
Request
      ↓
Approval
      ↓
Privileged Account / Role
      ↓
Session
      ↓
Target System
      ↓
Action
      ↓
System Log
      ↓
Verification
```

---

# 81. Correlation Key

Để correlate, cần stable identifiers.

Ví dụ:

```text
Ticket number
sys_id
User ID
Privileged account ID
Session ID
Hostname
Asset ID
AWS account ID
Role
Timestamp
```

Nếu chỉ dựa vào display name:

```text
John Smith
```

rất dễ mismatch.

---

# 82. Timeline Correlation

Giả sử:

```text
10:00 Change approved
10:02 Access granted
10:04 Gateway session starts
10:06 Admin logs into server
10:08 Service restarted
10:10 Health check passes
10:12 Session ends
10:15 Access revoked
```

Timeline này mạnh hơn rất nhiều so với mỗi hệ thống chỉ có một screenshot riêng biệt.

Nó trả lời:

```text
Who?
When?
Under which approval?
On which system?
What happened?
Was access removed afterward?
```

---

# 83. Timezone

Correlation chỉ đáng tin nếu timestamps cùng reference.

Ví dụ:

```text
ServiceNow      → UTC
Server          → ICT
Gateway         → UTC
CloudTrail      → UTC
```

Phải normalize hoặc document timezone.

Nếu không:

```text
10:00
```

ở system A và:

```text
10:00
```

ở system B có thể cách nhau 7 giờ.

Day 10 đã tạo foundation này; Day 11 áp vào governance timeline.

---

# PHẦN III — PALLETCONTROL & IT ASSET MANAGEMENT

Theo syllabus, Day 11 cần học PalletControl trong context của:

- hardware/software inventory;
- managed endpoints;
- asset reconciliation;
- lifecycle status;
- inventory data quality;
- controlled maintenance of asset records. Infrastructure_Operations_14_Da…

---

# 84. Giới hạn kiến thức về PalletControl

Syllabus không xác định đầy đủ vendor/version/architecture của **PalletControl**.

Do đó chúng ta không được bịa:

- agent name;
- ports;
- database;
- API;
- menus;
- job commands;
- scanner behavior.

Syllabus yêu cầu dùng organization's actual documentation và procedures nếu implementation khác generic asset inventory workflow. Infrastructure_Operations_14_Da…

Vì vậy phần lý thuyết này tập trung vào **IT Asset Management concepts** mà PalletControl phải hỗ trợ trong lab theo actual deployment.

---

# 85. IT Asset Management — ITAM

**IT Asset Management** quản lý lifecycle và authoritative information về technology assets.

Mục tiêu:

```text
Know what exists
Know who owns it
Know its status
Know where it is
Know whether records are current
Control changes to inventory
```

---

# 86. Hardware Inventory

Hardware inventory có thể gồm:

```text
Asset ID
Serial number
Manufacturer
Model
Hostname
Location
Owner
Environment
Lifecycle state
```

Exact schema tùy implementation.

Hardware inventory giúp:

- support;
- patch planning;
- warranty;
- replacement;
- vulnerability remediation;
- audit.

---

# 87. Software Inventory

Software inventory có thể giúp biết:

- software name;
- installed version;
- endpoint;
- installation state;
- license status;
- unsupported versions.

Use cases:

```text
Find endpoints running vulnerable software
Find unsupported version
License reconciliation
Patch planning
```

---

# 88. Managed Endpoint

**Managed endpoint** là device nằm trong organization's management scope.

Ví dụ:

- workstation;
- server;
- laptop;
- VM.

Operational questions:

```text
Is endpoint managed?
Is agent reporting?
When was last inventory?
Is owner correct?
Is endpoint active?
```

---

# 89. Collection / Job / Agent Concept

Syllabus additive coverage yêu cầu, **where implemented**, phải inspect:

- collection status;
- job status;
- agent status.

Đây là generic model:

```text
Endpoint
   │
   ▼
Agent / Collector
   │
   ▼
Inventory Job
   │
   ▼
Asset Repository
```

Nếu data stale, lỗi có thể ở nhiều layer.

Không nên kết luận:

```text
No current record → server offline
```

Có thể chỉ là collector problem.

---

# 90. Lifecycle Status

Một asset có thể trải qua lifecycle kiểu:

```text
Planned
   ↓
Procured
   ↓
Received
   ↓
Deployed
   ↓
Active
   ↓
Maintenance
   ↓
Retired
   ↓
Disposed
```

Tên state cụ thể phụ thuộc tool/policy.

Điểm cần hiểu là:

> Asset record phải phản ánh real-world lifecycle ở mức được policy yêu cầu.

---

# 91. Lifecycle Mismatch

Ví dụ:

```text
PalletControl:
Status = Active

CMDB:
Status = Retired

Server monitoring:
No heartbeat 6 months
```

Bạn không được chọn ngay một source và overwrite hai source còn lại.

Phải điều tra:

```text
Which is authoritative?
When were records updated?
Was decommission approved?
Does physical/virtual resource still exist?
Is monitoring complete?
```

---

# 92. Inventory Freshness

Freshness trả lời:

> Record này được xác nhận/cập nhật gần nhất khi nào?

Ví dụ:

```text
Asset A:
Last collection = 5 minutes ago

Asset B:
Last collection = 120 days ago
```

Cả hai đều có record, nhưng confidence khác nhau.

---

# 93. Freshness ≠ Existence

Nếu:

```text
Last seen = 180 days ago
```

không đủ để kết luận:

```text
Asset deleted
```

Possible causes:

- agent broken;
- endpoint disconnected;
- network segmentation;
- collection disabled;
- inventory job failed;
- actual decommission.

Phải investigate.

---

# 94. Data Quality

Data quality có nhiều dimensions:

### Completeness

Required fields có đủ không?

### Accuracy

Value có đúng không?

### Consistency

Các systems có agree không?

### Uniqueness

Có duplicate không?

### Timeliness

Data có fresh không?

### Ownership

Có accountable owner không?

---

# 95. Asset Reconciliation

**Asset reconciliation** là so sánh records giữa các sources để phát hiện mismatch.

Day 11 có thể reconcile:

```text
PalletControl
      ↕
CMDB
      ↕
Server Inventory
      ↕
Endpoint Inventory
```

Các systems không nhất thiết có cùng purpose, nên reconciliation không phải simple Excel `VLOOKUP` rồi delete unmatched records.

---

# 96. Matching Key

Ideal matching có thể dùng:

- stable Asset ID;
- serial number;
- system UUID;
- cloud instance ID;
- managed device ID.

Hostname đôi khi helpful nhưng có thể thay đổi.

Display name là key yếu.

Ví dụ:

```text
server01
```

có thể đã được:

- rebuilt;
- renamed;
- cloned;
- reused.

---

# 97. Missing Record

Ví dụ:

```text
Server inventory = Present
PalletControl     = Missing
```

Possible causes:

- onboarding incomplete;
- agent missing;
- collection failure;
- synchronization delay;
- scope exclusion;
- wrong matching key.

Không nhất thiết là tool bug.

---

# 98. Duplicate Record

Ví dụ:

```text
PalletControl:
Asset 1001 = srv01
Asset 1844 = srv01
```

Need determine:

- same physical asset?
- rebuilt server?
- duplicate ingestion?
- old retired object?
- naming collision?

Không delete record dựa chỉ trên hostname.

---

# 99. Stale Record

Stale có nghĩa data không được cập nhật đủ gần theo expected freshness policy.

Ví dụ:

```text
Last inventory:
120 days ago

Expected:
Daily
```

Đây là meaningful discrepancy.

Nhưng root cause chưa xác định.

---

# 100. Ownership Mismatch

Ví dụ:

```text
PalletControl owner = Team A
CMDB support group  = Team B
ServiceNow ticket   = Team C
```

Cần biết các fields có cùng semantic meaning không.

Có thể:

```text
Asset owner ≠ Support group ≠ Service owner
```

Nên mismatch có thể legitimate.

---

# 101. Controlled Maintenance of Asset Records

Không nên trực tiếp edit inventory để “cho match”.

Process đúng:

```text
Detect discrepancy
     ↓
Validate source
     ↓
Identify authoritative data
     ↓
Document finding
     ↓
Obtain authorization
     ↓
Apply correction
     ↓
Re-run reconciliation
     ↓
Capture before/after evidence
```

---

# 102. Source of Truth

Một common mistake:

> “CMDB luôn là source of truth.”

Không nhất thiết.

Một organization có thể quy định:

```text
HR → Human identity authority
Cloud API → AWS instance state authority
Asset tool → Procurement lifecycle authority
CMDB → Service relationship authority
```

Phải biết **source of truth cho field nào**, không chỉ “source of truth system nào”.

---

# 103. Discovery Data và Authoritative Data

Discovery có thể biết:

```text
Hostname
IP
OS
Installed software
```

Nhưng không nhất thiết biết authoritative:

```text
Business owner
Approval
Financial ownership
Disposition status
```

Do đó inventory reconciliation cần semantic understanding.

---

# PHẦN IV — CROSS-SYSTEM OPERATIONAL GOVERNANCE

Đây là phần giúp ghép toàn bộ Day 11 lại.

---

# 104. Golden Operational Chain

Một enterprise maintenance scenario lý tưởng:

```text
Incident / Request
       ↓
Identify affected CI
       ↓
Assess Impact/Urgency/Priority
       ↓
Assign owner
       ↓
Technical investigation
       ↓
Determine required Change
       ↓
Approval
       ↓
Maintenance window
       ↓
Privileged access request
       ↓
Access approval
       ↓
Temporary privileged access
       ↓
Implementation
       ↓
Technical verification
       ↓
Audit/log correlation
       ↓
Access revocation
       ↓
Asset/CMDB update if needed
       ↓
Resolution
       ↓
Closure
       ↓
Shift handover
```

Đây gần như là “xương sống” của Day 11.

---

# 105. Workflow Evidence vs Technical Evidence

Hai loại evidence khác nhau.

### Workflow Evidence

Chứng minh authorization/process:

```text
Incident
Change
Approval
Access request
Assignment
Maintenance window
```

### Technical Evidence

Chứng minh system state/action:

```text
Server log
Command output
Configuration
IAM record
Gateway session
Application test
```

Một audit-quality package cần cả hai.

---

# 106. Vì sao chỉ screenshot ServiceNow không đủ?

Screenshot:

```text
Change = Completed
```

chỉ chứng minh UI hiển thị state tại thời điểm screenshot.

Nó không trực tiếp chứng minh:

```text
Service healthy
Configuration correct
Access revoked
```

Phải correlate với technical evidence.

---

# 107. Vì sao chỉ server log không đủ?

Log:

```text
User admin restarted service
```

có thể chứng minh action.

Nhưng không chứng minh:

```text
Action authorized
Change approved
Maintenance window valid
```

Cần workflow evidence.

---

# 108. Evidence Chain

Ideal evidence chain:

```text
Ticket
  │
  ├─ Approval
  │
  ├─ CI
  │
  ├─ Privileged Access Request
  │       │
  │       └─ Session Reference
  │
  ├─ Server / Device Logs
  │
  ├─ Verification Result
  │
  └─ Closure/Handover
```

---

# 109. Evidence Manifest

Day 11 tiếp tục evidence discipline từ Day 1.

Common evidence contract của syllabus yêu cầu evidence item ghi những thông tin như:

- Evidence ID
- source system/product/version
- environment/asset/account
- ticket reference
- stable record key
- collector
- collection timestamp/timezone
- query/time window
- filters/commands
- export format/encoding
- file size
- record count
- SHA-256
- transformation lineage
- limitations/errors. Infrastructure_Operations_14_Da…

---

# 110. Raw Evidence vs Working Copy

Phải tách:

```text
RAW
Untouched original export
```

và:

```text
WORKING
Filtered/normalized/correlated copy
```

Ví dụ:

```text
INC1001_raw.csv
INC1001_analysis.xlsx
INC1001_redacted.csv
```

Mỗi derived copy nên có traceability về parent source.

---

# 111. Hash có ý nghĩa gì?

Ví dụ:

```text
SHA-256(raw.csv) = ABC...
```

Sau đó hash vẫn:

```text
ABC...
```

→ file contents match recorded baseline.

Nhưng hash **không chứng minh**:

- query đúng;
- source system đúng;
- export đầy đủ;
- time window đúng;
- source data truthful.

Đây là distinction bắt buộc xuyên suốt chương trình. Infrastructure_Operations_14_Da…

---

# 112. Bounded Extraction

Evidence extraction nên có boundary rõ.

Ví dụ:

```text
Ticket:
INC0012345

Window:
2026-10-21 02:00–04:00 UTC

Fields:
Number
State
Assignment
CI
Timestamps
Approval references
```

Không:

```text
Export toàn bộ ServiceNow database.
```

Bounded extraction hỗ trợ:

- least privilege;
- privacy;
- audit scope;
- reproducibility;
- manageable evidence.

---

# 113. Stable ID vs Display Value

Một display name có thể đổi.

Ví dụ:

```text
Display:
John Smith
```

hoặc:

```text
Network Team
```

Stable internal identifier khi được phép có thể giúp correlation mạnh hơn.

Syllabus yêu cầu giữ stable `sys_id` trong ServiceNow export **where permitted**. Infrastructure_Operations_14_Da…

Không có nghĩa phải dump mọi internal ID bất kể scope/security.

---

# 114. Page Limits và Export Limits

Một export cho thấy:

```text
1000 rows
```

không có nghĩa dataset có đúng 1000 records.

Có thể tool giới hạn export/page.

Phải xác minh:

```text
Total matching records
Rows exported
Pagination
Export limit
Excluded records
```

Điều này quan trọng cho audit completeness.

---

# 115. Evidence Gap

Khi evidence không obtainable:

Không được giả vờ rằng nó không cần thiết.

Ghi:

```text
Evidence Gap:
Security audit field unavailable due ACL.

Impact:
Cannot independently verify exact permission-change actor.

Action:
Escalated to authorized ServiceNow audit administrator.
```

Một documented unknown tốt hơn một fabricated conclusion.

---

# 116. Least Privilege trong evidence collection

Collector chỉ nên có quyền cần thiết.

Không nên yêu cầu:

```text
ServiceNow admin
```

chỉ để lấy vài approved ticket records nếu scoped role đủ.

Tương tự không dùng:

```text
AWS AdministratorAccess
```

chỉ để read một IAM state.

---

# 117. Secret Handling

Evidence tuyệt đối không nên chứa unnecessary secrets:

- password;
- private key;
- MFA seed;
- token;
- secret header;
- session secret;
- temporary AWS credentials.

Syllabus yêu cầu privileged-access evidence phải chứng minh identity, entitlement, approval, expiry, revocation và audit record **mà không expose credentials**. Infrastructure_Operations_14_Da…

---

# 118. Troubleshooting Model — Ticket có vẻ sai

Nếu ServiceNow record không như expected, điều tra theo low-risk order:

```text
1. Confirm correct ticket.
2. Confirm current state.
3. Confirm assignment.
4. Confirm timestamps/timezone.
5. Confirm approvals.
6. Confirm CI/reference.
7. Check journal entries.
8. Check audit/history if authorized.
9. Check ACL/visibility.
10. Escalate before modifying workflow data manually.
```

Không sửa ticket fields chỉ để “làm đẹp data”.

---

# 119. Troubleshooting Model — Access Approved nhưng Login Denied

Đi từ ít rủi ro đến nhiều rủi ro:

```text
1. Confirm requester identity.
2. Confirm approval state.
3. Confirm validity/expiry.
4. Confirm target system.
5. Confirm requested role/entitlement.
6. Confirm entitlement actually provisioned.
7. Confirm account active.
8. Confirm authentication path.
9. Confirm gateway/session policy.
10. Confirm network/service reachability.
11. Inspect authorized audit evidence.
12. Escalate according to owning layer.
```

Không reset password trước khi xác định layer lỗi.

---

# 120. Troubleshooting Model — Access Expired nhưng vẫn hoạt động

Đây là security issue.

Phải xác định:

```text
Was expiry only workflow metadata?
Was entitlement removed?
Is existing session still active?
Was role cached?
Is there another role granting equivalent access?
```

Đây là distinction giữa:

```text
Intended Access
```

và:

```text
Effective Access
```

---

# 121. Effective Access

**Effective access** là quyền thực sự user có sau khi tính toàn bộ policies, groups, roles, inherited privileges và sessions.

Ví dụ:

Bạn remove:

```text
Role A
```

nhưng user vẫn có quyền qua:

```text
Group B → Role C
```

Vậy revocation incomplete.

---

# 122. Troubleshooting Model — Inventory Missing

```text
1. Confirm correct Asset ID.
2. Search alternate stable IDs.
3. Confirm endpoint exists.
4. Confirm endpoint in management scope.
5. Check last collection.
6. Check agent/collector if applicable.
7. Check inventory job.
8. Check reconciliation delay.
9. Check duplicate/renamed records.
10. Escalate if source pipeline malfunctioning.
```

Không create duplicate asset ngay.

---

# 123. Troubleshooting Model — Duplicate Asset

```text
1. Compare stable IDs.
2. Compare serial/system UUID.
3. Compare creation dates.
4. Check rebuild history.
5. Check lifecycle state.
6. Check CI relationship.
7. Check authoritative source.
8. Stage correction only after ownership/approval.
```

---

# 124. Security Implications của Day 11

Day 11 touches extremely sensitive controls:

### Unauthorized Change

Service impact.

### Excessive Privilege

Compromise blast radius.

### Missing Audit

Không thể reconstruct action.

### Stale Asset Data

Vulnerability/patch gaps.

### Orphaned Account

Unowned privilege.

### Broken SoD

Fraud/abuse/control failure.

### Exposed Credentials

Direct security compromise.

### Incorrect Closure

Hidden operational risk.

---

# 125. L1/L2 Boundary — Incident

Typical L1:

- ticket intake;
- verify symptoms;
- use approved runbook;
- gather read-only evidence;
- basic triage;
- route/escalate.

Typical L2:

- deeper troubleshooting;
- controlled service operations;
- authorized configuration work;
- correlation;
- implementation/rollback within approved scope.

Nhưng actual organizational boundary luôn ưu tiên runbook/policy.

---

# 126. L1/L2 Boundary — Change

L1 thường không tự:

- approve change;
- expand scope;
- modify production configuration ngoài runbook;
- waive maintenance window.

L2 có thể implement approved changes nhưng không mặc định có quyền:

- self-approve;
- bypass CAB/change authority;
- alter risk classification.

---

# 127. L1/L2 Boundary — Privileged Access

L1/L2 có thể:

- validate request;
- collect evidence;
- identify entitlement mismatch;
- execute approved workflow.

Không tự:

- create unauthorized privileged account;
- extend expired access;
- bypass approval;
- share credential;
- disable audit;
- use break-glass vì convenience.

---

# 128. Khi nào cần Escalate privileged-access issue?

Escalate khi:

- owner unknown;
- approval ambiguous;
- suspected unauthorized access;
- orphaned privileged account;
- expired entitlement still active;
- credential rotation failure impacts services;
- gateway/session audit unavailable;
- break-glass used unexpectedly;
- suspected credential exposure;
- SoD violation.

---

# 129. Khi nào cần Escalate Asset issue?

Escalate khi:

- source of truth unclear;
- mass duplicates;
- collection pipeline fails;
- widespread stale inventory;
- asset owner unknown;
- decommission mismatch;
- critical asset absent;
- unauthorized record modifications;
- inventory discrepancy affects security/compliance.

---

# 130. Common Mistake: Ticket = Truth

Sai:

```text
ServiceNow says server is retired → therefore server no longer exists.
```

Đúng hơn:

```text
ServiceNow/CMDB says retired.
Now verify against authoritative technical/asset source.
```

---

# 131. Common Mistake: Technical System = Truth cho mọi field

Sai:

```text
Server is online → therefore asset should be Active.
```

Có thể server đang online tạm thời trong decommission process.

Mỗi source có semantic authority khác nhau.

---

# 132. Common Mistake: Approved = Provisioned

Không.

```text
Approved
```

là workflow state.

Phải kiểm tra:

```text
Effective entitlement.
```

---

# 133. Common Mistake: Revocation Requested = Revoked

Không.

Cần:

```text
Request
   ↓
Execution
   ↓
Effective verification
```

Syllabus thậm chí coi missing privileged-access revocation proof là open completion gap. Infrastructure_Operations_14_Da…

---

# 134. Common Mistake: History = Complete Audit

Không.

Syllabus đặc biệt thêm requirement này bởi đây là một audit mistake rất dễ mắc. Infrastructure_Operations_14_Da…

ServiceNow Xanadu documentation cũng cho thấy History Sets/History phục vụ display/use khác Audit và chỉ là subset được generated từ Audit data. [servicenow.com](https://www.servicenow.com/docs/r/xanadu/platform-security/c_DiffBtwnAuditHistSets.html)

---

# 135. Common Mistake: Access Denied = IAM Problem

Access path có nhiều layers:

```text
Identity
Authentication
Authorization
Gateway
Network
Target service
Target policy
```

Login denied không tự động nghĩa IAM entitlement sai.

---

# 136. Common Mistake: “No Last Login” = Dormant

Không.

Có thể:

- unsupported evidence;
- missing data;
- retention issue;
- API limitation;
- service account behavior.

Day 13 sẽ formalize cách xử lý unknown last-use evidence.

---

# 137. Common Mistake: Asset Missing = Create New Record

Nếu actual record tồn tại dưới old hostname/stable ID khác, bạn sẽ tạo duplicate.

Điều tra trước.

---

# 138. Common Mistake: Asset Duplicate = Delete One

Bạn có thể xóa historical legitimate record.

Phải xác định lifecycle và stable identity trước.

---

# 139. Common Mistake: Screenshot = Audit Package

Screenshot useful nhưng thường không đủ cho:

- count reconciliation;
- complete field extraction;
- repeatable query;
- machine processing;
- raw evidence integrity.

Syllabus nói screenshots nên supplement exportable records chứ không mặc định thay thế chúng. Infrastructure_Operations_14_Da…

---

# 140. Một scenario tổng hợp

Giả sử:

> Production server `APP01` bị service failure. Engineer cần temporary privileged access để restart service và thay một approved configuration parameter.

Operational reasoning:

```text
1. Incident exists.
2. Verify APP01 CI.
3. Determine Impact/Urgency/Priority.
4. Assign Application/Linux Operations.
5. Perform read-only diagnosis.
6. Determine configuration change required.
7. Link/create Change.
8. Confirm Change risk.
9. Confirm implementation plan.
10. Confirm backout plan.
11. Confirm maintenance window.
12. Obtain approval.
13. Request privileged access.
14. Verify access approval.
15. Obtain time-bounded privilege.
16. Execute approved implementation.
17. Verify service.
18. Collect host/application logs.
19. End privileged session.
20. Revoke/verify access.
21. Update Asset/CI if configuration/lifecycle data changed.
22. Update Change.
23. Resolve Incident.
24. Produce handover if shift changes.
25. Preserve evidence.
```

Đây chính là cách Day 11 biến technical engineer thành Infrastructure Operations engineer.

---

# 141. Nếu bước 13 xảy ra trước bước 7 thì sao?

Không tự động là violation.

Có thể read-only access đã được provisioned theo standing approved support model.

Nhưng nếu privileged access được dùng để perform change, cần kiểm tra authorization chain phù hợp.

Điểm quan trọng:

> Không áp timeline cứng nhắc nếu organization có different approved process; nhưng mọi action quan trọng phải traceable và authorized.

---

# 142. Evidence Correlation Scenario

Giả sử chúng ta có:

```text
ServiceNow:
CHG000100 → Approved 13:00 UTC

Gateway:
Session S-987 → User A → APP01 → 13:05 UTC

Linux:
sudo event → User A → 13:07 UTC

Application:
Restart → 13:08 UTC

Health monitor:
Healthy → 13:09 UTC

Gateway:
Session closed → 13:12 UTC

Access system:
Privilege revoked → 13:15 UTC
```

Bạn có thể xây auditable timeline:

```text
Approval
   ↓
Authorized access
   ↓
Privileged action
   ↓
Technical result
   ↓
Session termination
   ↓
Privilege revocation
```

Đây là evidence quality tốt.

---

# 143. Nhưng correlation không phải proof tuyệt đối

Cùng timestamp và same user không tự động chứng minh causality.

Ví dụ:

```text
Gateway session active
```

không nhất thiết session đó đã thực hiện Linux command nếu account shared.

Do đó cần càng nhiều stable correlation key càng tốt.

---

# 144. Read-only Verification vs Configuration-changing Action

Day 11 phải giữ distinction này.

### Read-only examples

- view ticket;
- inspect approval;
- view IAM role;
- inspect asset record;
- view audit records;
- compare inventory.

### Mutating examples

- approve ticket;
- change ticket state;
- assign IAM role;
- revoke role;
- modify asset;
- update CI;
- rotate credential.

Mutating action cần stronger authorization/evidence.

---

# 145. Security Principle: Never use evidence collection to expand access

Ví dụ engineer được permission xem:

```text
Incident
```

nhưng không được:

```text
Security audit records
```

Không nên tìm cách bypass ACL để “hoàn thành lab”.

Correct behavior:

```text
Document evidence gap
Escalate to authorized owner
```

---

# 146. Audit Evidence và Privacy

Audit records có thể chứa:

- usernames;
- user activity;
- system details;
- potentially sensitive fields.

Principles:

```text
Minimum necessary scope
Approved storage
Restricted access
Retention classification
Redaction where required
```

Không share raw privileged logs rộng rãi chỉ vì là “training evidence”.

---

# 147. Operational Record Quality

Một ticket tốt phải giúp một engineer khác reconstruct:

```text
What happened?
What was affected?
What did we know at the time?
What did we do?
Why were we allowed?
What result did we get?
What remains open?
```

Nếu ticket không trả lời được các câu này, documentation quality còn yếu.

---

# 148. Data Quality và Operational Risk

Bad asset data gây risk thực tế.

Ví dụ:

```text
Owner incorrect
```

→ vulnerability remediation gửi nhầm team.

```text
Lifecycle = Retired
but server still active
```

→ có thể bỏ patch.

```text
Duplicate
```

→ compliance count sai.

```text
Missing endpoint
```

→ security agent coverage gap.

Do đó Asset Management không phải administrative paperwork.

---

# 149. ITSM và Security không tách rời

Một security incident cũng cần operational governance:

```text
Detection
  ↓
Incident
  ↓
Owner
  ↓
Impact
  ↓
Privileged investigation
  ↓
Approved containment
  ↓
Change
  ↓
Evidence
  ↓
Closure
```

Security control mà không có operational workflow thường khó audit và khó reproduce.

---

# 150. ITSM và IAM không tách rời

Access request nên connect:

```text
Who needs access?
Why?
For which ticket/business need?
Who approved?
For how long?
Was it revoked?
```

IAM chỉ nhìn technical permissions mà không nhìn business approval sẽ thiếu governance context.

---

# 151. ITSM và Asset Management không tách rời

Ticket nên reference correct CI/asset.

Nếu không:

```text
Incident = APP01
Evidence = APP10
```

có thể investigation sai system.

Correct asset/CI linkage giúp:

- ownership;
- impact;
- troubleshooting;
- audit;
- maintenance planning.

---

# 152. Asset Management và IAM cũng liên quan

Privileged access vào:

```text
srv-123
```

cần biết:

```text
Asset owner?
Environment?
Criticality?
Lifecycle?
```

Access vào decommissioned/orphaned asset có thể là security anomaly.

---

# 153. Day 11 và Day 10

Day 10 dạy:

```text
Logs tell you what systems observed.
```

Day 11 thêm:

```text
Tickets/approvals tell you what operations were authorized/intended.
```

Correlation hai nguồn giúp trả lời:

```text
Observed technical action
        +
Approved operational intent
```

---

# 154. Day 11 và Day 12

Day 12 automation sẽ xử lý:

- CSV;
- IDs;
- logs;
- reconciliations;
- reports.

Day 11 cung cấp business semantics:

```text
What does account ID mean?
What does asset mismatch mean?
What is valid approval?
What should be reported?
```

Nếu không hiểu Day 11, automation Day 12 có thể chạy rất nhanh... nhưng đưa ra kết luận sai.

---

# 155. Day 11 và Day 13

Day 13 90-day IAM review dựa trực tiếp vào Day 11:

```text
Account ownership
Approval
Entitlement
Dormancy
Orphan account
SoD
Privilege
Access evidence
Revocation
```

Day 11 là foundation của governance reasoning.

---

# 156. Operational State vs Intended State

Đây là một mental model cực kỳ hữu ích:

```text
Intended State
    │
    │ defined by:
    │ ticket / approval / CMDB / policy
    │
    ▼
Effective State
    │
    │ observed from:
    │ system / IAM / asset / logs
    │
    ▼
Compare
```

Nếu khác:

```text
Exception / discrepancy
```

Ví dụ:

```text
Approval:
Access expired

Effective State:
Role still active
```

→ exception.

---

# 157. Record State vs Reality Matrix

| Workflow/Record | Technical Reality | Ý nghĩa                 |
| --------------- | ----------------- | ----------------------- |
| Approved        | Access active     | có thể đúng             |
| Approved        | Access absent     | fulfillment issue       |
| Expired         | Access absent     | expected                |
| Expired         | Access active     | security exception      |
| Asset Active    | Host active       | likely consistent       |
| Asset Active    | Host gone         | investigate             |
| Asset Retired   | Host active       | investigate urgently    |
| Ticket Resolved | Service healthy   | consistent              |
| Ticket Resolved | Service down      | closure quality problem |

Đây là mindset rất quan trọng cho Infrastructure Operations.

---

# 158. Evidence State cũng phải được kiểm tra

Có ba tầng:

```text
Declared state
Observed state
Evidence state
```

Ví dụ:

```text
Declared:
Access revoked.

Observed:
User cannot access.

Evidence:
Revocation audit record available.
```

Confidence cao hơn.

Nếu:

```text
Declared:
Access revoked.

Observed:
Unknown.

Evidence:
None.
```

Không nên kết luận chắc chắn.

---

# 159. Escalation Boundary Model

Khi gặp issue, hãy hỏi:

```text
Do I understand the issue?
        ↓
Do I have authority?
        ↓
Is there an approved runbook?
        ↓
Is the action reversible?
        ↓
Is impact bounded?
        ↓
Can I verify success?
```

Nếu một trong các điều quan trọng là “No/Unknown”, khả năng cao nên stop/escalate.

---

# 160. Khi L1/L2 nên dừng

Ví dụ:

- unclear approval;
- unknown privileged account owner;
- break-glass required nhưng criteria không rõ;
- asset correction affects financial records;
- audit data inaccessible;
- broad privileged access discrepancy;
- multiple production assets inconsistent;
- mass access revocation required;
- suspicious privileged activity;
- evidence indicates possible compromise.

---

# 161. Evidence cần chuẩn bị cho Escalation

Good escalation không phải:

> “Access doesn't work.”

Mà nên bao gồm:

```text
Ticket:
REQ123

User:
user01

Target:
server01

Expected:
Temporary admin access until 15:00 UTC

Actual:
Authentication succeeds, authorization denied

Verified:
Approval present
Account active
Network path reachable

Not verified:
Effective group membership

Evidence:
Screenshots/export/log references

Impact:
Maintenance blocked

Requested escalation:
Privileged-access platform owner
```

Như vậy L3/vendor có thể tiếp tục nhanh hơn.

---

# 162. Theory Checklist — ServiceNow / ITSM

Sau Module Lý thuyết bạn phải tự giải thích được:

- ITSM khác technical troubleshooting ở đâu.
- Incident là gì.
- Problem là gì.
- Request khác Incident thế nào.
- Change dùng khi nào.
- Task dùng để làm gì.
- Approval chứng minh cái gì và không chứng minh cái gì.
- Knowledge có vai trò gì.
- Asset khác CI thế nào.
- CMDB dùng để làm gì.
- ticket lifecycle hoạt động ra sao.
- assignment group và assignee khác nhau thế nào.
- Impact, Urgency, Priority khác nhau thế nào.
- SLA là gì và tại sao không được assume timer.
- Work Notes khác customer-visible comments.
- approval gate là gì.
- maintenance window để làm gì.
- change risk phải đánh giá những gì.
- implementation plan cần gì.
- backout plan cần gì.
- stop condition là gì.
- resolution khác closure thế nào.
- resolution code để làm gì.
- shift handover cần gì.
- field/journal/approval/attachment/audit/history khác nhau thế nào.
- tại sao History không được coi là complete audit archive.
- ACL ảnh hưởng evidence extraction như thế nào.

---

# 163. Theory Checklist — Privileged Access

Bạn phải giải thích được:

- Identity khác account.
- Account khác credential.
- Credential khác entitlement.
- Privileged Identity là gì.
- privileged ID lifecycle.
- account ownership.
- authentication vs authorization.
- access request.
- least privilege.
- temporary/JIT privilege.
- approval workflow.
- session/gateway model.
- credential checkout.
- credential rotation.
- rotation dependency.
- AWS IAM identity/role trong access governance.
- access review.
- effective access.
- role assignment.
- SoD.
- dormant account.
- orphaned account.
- duplicate account.
- break-glass.
- privileged activity auditing.
- expiry vs actual revocation.
- workflow evidence vs technical evidence.

---

# 164. Theory Checklist — PalletControl / ITAM

Bạn phải giải thích được:

- IT Asset Management.
- hardware inventory.
- software inventory.
- managed endpoint.
- collection/job/agent concept.
- lifecycle status.
- inventory freshness.
- data quality.
- reconciliation.
- stable matching key.
- missing record.
- duplicate record.
- stale record.
- ownership discrepancy.
- source of truth.
- authoritative field.
- controlled asset correction.
- Asset vs CI.
- Asset/CMDB relationship.

---

# 165. Mental Model quan trọng nhất của Day 11

Nếu chỉ nhớ một diagram của toàn Module Lý thuyết, hãy nhớ:

```text
                 BUSINESS / USER NEED
                         │
                         ▼
                INCIDENT / REQUEST
                         │
                         ▼
                OWNER / PRIORITY
                         │
                         ▼
               CHANGE IF REQUIRED
                         │
                         ▼
                     APPROVAL
                         │
                         ▼
             PRIVILEGED ACCESS
                         │
                         ▼
               TECHNICAL ACTION
                         │
                         ▼
                   VERIFICATION
                         │
           ┌─────────────┴─────────────┐
           ▼                           ▼
      SYSTEM LOGS                ACCESS AUDIT
           │                           │
           └─────────────┬─────────────┘
                         ▼
                  ASSET / CI CHECK
                         │
                         ▼
                EVIDENCE CORRELATION
                         │
                         ▼
              RESOLUTION / CLOSURE
                         │
                         ▼
                     HANDOVER
```

---

# 166. Cheat Sheet — Module Lý thuyết Day 11

| Area                  | Critical concepts                                                           |
| --------------------- | --------------------------------------------------------------------------- |
| ITSM                  | Incident, Problem, Request, Change, Task                                    |
| ServiceNow governance | Assignment, Approval, Impact, Urgency, Priority, SLA                        |
| Documentation         | Work Notes, Comments, Attachments, Knowledge                                |
| Change control        | Risk, Maintenance Window, Implementation Plan, Backout Plan, Stop Condition |
| Closure               | Resolution, Resolution Code, Closure, Handover                              |
| Audit                 | Fields, Journal, Approval, Audit, History, ACL, Retention                   |
| Privileged IAM        | Identity, Account, Credential, Entitlement                                  |
| Access lifecycle      | Request → Approval → Provision → Use → Expire/Revoke → Audit                |
| PAM concepts          | Gateway, Session, Credential Checkout, Rotation                             |
| Governance            | Least Privilege, SoD, Access Review                                         |
| Risk accounts         | Dormant, Orphaned, Break-glass                                              |
| AWS                   | Identity, Role, Effective Permission, Approval, Activity                    |
| ITAM                  | Hardware/Software Inventory, Managed Endpoint                               |
| PalletControl         | Collection/Job/Agent where implemented, Freshness, Reconciliation           |
| Data quality          | Missing, Duplicate, Stale, Ownership/Lifecycle mismatch                     |
| CMDB                  | CI, relationships, service impact                                           |
| Evidence              | Ticket + Approval + Session + Technical Log + Verification                  |
| Integrity             | Raw export, working copy, count, SHA-256                                    |
| Operations rule       | **Declared state must be checked against effective state**                  |

Theo traceability matrix của syllabus, Day 11 cuối cùng phải xây đủ knowledge để bạn có thể xử lý **ServiceNow Incident/Request/Change lifecycle, maintenance/backout, approvals, closure/handover, audit extraction; PalletControl inventory freshness/reconciliation; và separate privileged-access lifecycle cho PS/Gateway, AWS IAM và AccessCheck**. Infrastructure_Operations_14_Da…

**Đến đây chúng ta đã cover toàn bộ Module Lý thuyết Day 11 trong syllabus.** Chưa chuyển sang thao tác GUI/product-specific hay lab; đặc biệt **PS/Gateway, AccessCheck và PalletControl sẽ không được giả định menu/command/function** cho đến khi có exact product/version/runbook, đúng với giới hạn mà syllabus đặt ra. Bước tiếp theo của Day 11 là **Module Thực hành**, nơi toàn bộ lý thuyết này sẽ được ghép thành một scenario xuyên suốt từ ServiceNow ticket → approval → privileged access → maintenance → evidence → PalletControl/CMDB reconciliation → closure và shift handover.
