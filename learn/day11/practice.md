# Day 11 — Module Thực hành

## ServiceNow ITSM + Privileged Access + AWS IAM + PalletControl + Audit Evidence + Operational Handover

Module Thực hành Day 11 là nơi toàn bộ lý thuyết về **ITSM, ServiceNow, privileged identity/access, PalletControl, asset/CMDB reconciliation và audit evidence** được ghép thành một operational workflow hoàn chỉnh.

Theo syllabus, bạn phải thực hiện hoặc mô phỏng có kiểm soát ba workflow ServiceNow — **Incident, Service Request và approved Change**; thực hiện các privileged-access scenarios trên **PS/Gateway, AccessCheck và AWS IAM**; xác minh identity, entitlement, approval, expiry, revocation và audit trail; reconcile PalletControl với server/endpoint/CMDB; điều tra missing/duplicate/stale asset; liên kết server logs với ServiceNow ticket; và tạo maintenance/handover evidence. Infrastructure_Operations_14_Da…

Điểm đặc biệt của Day 11 là bạn không được đánh giá “Pass” chỉ vì thao tác kỹ thuật thành công. Bạn phải chứng minh được chuỗi:

```text
Business / Operational Need
          ↓
Incident / Request
          ↓
Correct CI / Asset
          ↓
Ownership + Priority
          ↓
Investigation
          ↓
Change if required
          ↓
Approval
          ↓
Privileged Access
          ↓
Authorized Technical Action
          ↓
Technical Verification
          ↓
Log / Audit Correlation
          ↓
Privilege Revocation
          ↓
Asset / CMDB Reconciliation
          ↓
Resolution / Closure
          ↓
Shift Handover
          ↓
Evidence Manifest
```

---

# 1. Mục tiêu của Module Thực hành Day 11

Sau khi hoàn thành module này, bạn phải có khả năng thực hiện một scenario như một Infrastructure Operations engineer chứ không đơn giản như một system administrator.

Bạn phải chứng minh được:

- tiếp nhận và xử lý một Incident;
- xác định đúng affected CI;
- xác định assignment/ownership;
- đánh giá Impact, Urgency, Priority theo matrix được cung cấp;
- ghi Work Notes có chất lượng operational;
- tạo/xử lý một Service Request;
- tạo/xử lý một approved Change;
- xác minh approval trước khi action;
- xác minh maintenance window;
- xây implementation plan;
- xây backout plan;
- xác định stop conditions;
- request privileged access;
- kiểm tra approval và entitlement;
- thực hiện authorized privileged session;
- xác minh effective access;
- thực hiện một controlled maintenance action;
- thu technical evidence;
- kiểm tra expiry/revocation;
- chứng minh quyền thực sự bị revoke;
- extract ServiceNow evidence;
- distinguish fields/journal/approval/audit/history;
- reconstruct joined timeline;
- reconcile PalletControl/CMDB/server inventory;
- xử lý missing/duplicate/stale asset;
- stage authorized correction;
- thu before/after evidence;
- resolve/close ticket đúng cách;
- tạo shift handover;
- tạo evidence manifest có hash/count/traceability.

Đây chính là các deliverables mà syllabus yêu cầu. Infrastructure_Operations_14_Da…

---

# 2. Phân loại mức độ Hands-on

Ngay đầu lab phải ghi rõ bạn thực hiện activity ở mức nào.

Syllabus dùng bốn mức:

| Code  | Ý nghĩa                                             |
| ----- | --------------------------------------------------- |
| **H** | Bạn tự thực hiện trên actual authorized product lab |
| **S** | Bạn thực hiện bằng simulation hoặc supplied fixture |
| **D** | Instructor demonstration                            |
| **T** | Theory/reference only                               |

Ngoài ra nên ghi:

```text
Independent
```

hoặc:

```text
Assisted
```

Ví dụ:

```text
ServiceNow Incident workflow: H / Independent
AWS IAM access review: H / Assisted
PS/Gateway: S / Independent
AccessCheck: S / Independent
PalletControl: H / Assisted
```

Không được ghi một simulation thành actual product experience. D/T cũng không được tính như hands-on. Đây là completion control chính thức trong syllabus. Infrastructure_Operations_14_Da…

---

# 3. Lab Scenario tổng hợp

Chúng ta dùng một synthetic enterprise scenario xuyên suốt.

Giả sử có application:

```text
Business Service:
LAB-Order-Service

Application server:
LAB-RHEL11-APP01

Application service:
d11-demo.service

Environment:
LAB / NON-PRODUCTION
```

Một monitoring alert báo:

```text
LAB-Order-Service degraded
```

Infrastructure Operations kiểm tra và phát hiện:

```text
d11-demo.service
```

đang ở abnormal state.

Để restore service, engineer có thể cần:

1. Incident;
2. troubleshooting;
3. Change;
4. privileged access;
5. controlled service operation;
6. verification;
7. audit evidence.

Trong quá trình đó bạn còn phát hiện asset record của server trong PalletControl không khớp CMDB.

Ta sẽ dùng cùng một scenario để luyện gần như toàn bộ Day 11.

---

# 4. Logical Topology

```text
                  ┌──────────────────┐
                  │   ServiceNow     │
                  │ ITSM / CMDB      │
                  └────────┬─────────┘
                           │
      Incident / Request / Change / Approval
                           │
                           ▼
               ┌─────────────────────┐
               │ Privileged Access   │
               │ PS/Gateway /        │
               │ AccessCheck / AWS   │
               └─────────┬───────────┘
                         │
                         ▼
                ┌────────────────┐
                │ LAB Server     │
                │ RHEL/Windows   │
                └───────┬────────┘
                        │
                        │ logs
                        ▼
                ┌────────────────┐
                │ Evidence Repo  │
                └────────────────┘

                        ▲
                        │ inventory
                        │
                ┌───────┴────────┐
                │ PalletControl  │
                │ Asset Records  │
                └────────────────┘
```

Không phải tất cả system nhất thiết được integrated trực tiếp. Diagram trên mô tả **operational relationship**, không khẳng định product architecture cụ thể.

---

# 5. Prerequisites

Trước lab cần có:

### ServiceNow

Một training instance hoặc fixture chứa:

```text
Incident
Request
Change
CI
Approval
Audit/history sample
```

### Server

Một authorized lab server:

```text
LAB-RHEL11-APP01
```

hoặc Windows Server tương đương.

### Privileged access

Ít nhất một trong:

```text
PS/Gateway actual lab
AccessCheck actual lab
AWS sandbox IAM
```

Nếu product thực không có, instructor phải cung cấp:

```text
version-labelled screenshots
exports
audit records
fault packets
expected results
approved runbook extract
```

### PalletControl

Actual lab hoặc fixture dataset.

### Evidence repository

Ví dụ:

```text
Day11/
├── 00_Control/
├── 01_ServiceNow/
├── 02_PrivilegedAccess/
├── 03_Server/
├── 04_PalletControl/
├── 05_Timeline/
├── 06_Handover/
└── 99_Manifest/
```

---

# 6. Safety Notes bắt buộc

Đây là lab/non-production/authorized environment.

Không sử dụng:

```text
real production credentials
real MFA secrets
private keys
access tokens
AWS secret access keys
real customer data
```

Dùng placeholders:

```text
<USER_ID>
<TICKET_ID>
<ROLE_NAME>
<TARGET_HOST>
```

Đối với mutating action:

- phải có approval;
- phải đúng scope;
- phải đúng maintenance window;
- phải có rollback;
- phải verify sau action;
- phải ghi evidence.

Không tự ý:

```text
approve own access
extend expired privilege
reset shared credential
edit audit record
delete asset
close ticket to stop SLA
disable logging
```

---

# 7. Evidence Contract áp dụng cho toàn lab

Mỗi evidence item phải có tối thiểu:

```text
Evidence ID
Source system/product/version
Environment
Asset/account identifier
Related ticket
Stable record key
Collector
Collection timestamp + timezone
Requested interval
Exact filters/commands
Export format
Encoding
File path/name
File size
Record count
SHA-256
Access/retention classification
Parent evidence ID
Transformation/script version
Limitations/errors
```

ServiceNow export phải giữ stable identifiers như `sys_id` **where authorized**, đồng thời document việc dùng display values hay internal values.

Phải giữ raw export nguyên vẹn, xử lý trên copy, verify hash sau transfer, dùng record count/known event để đánh giá completeness. Screenshot chỉ là supplementary evidence nếu structured export có thể lấy được. Infrastructure_Operations_14_Da…

Ví dụ manifest:

```text
Evidence ID: D11-EV-001
Source: ServiceNow Training Instance
Record: INC00XXXX
CI: LAB-RHEL11-APP01
Collected: 2026-10-21T10:15:00+07:00
Collector: student01
Window: 2026-10-21 09:00–11:00 ICT
Format: CSV
Record Count: 1
SHA-256: <HASH>
Related Change: CHG00XXXX
Limitations: Audit field X unavailable under learner ACL
```

---

# LAB 1 — Pre-flight & Operational Baseline

## Objective

Xác minh environment và authorization trước khi thực hiện bất kỳ action nào.

---

## Step 1 — Xác minh environment

Ghi:

```text
Environment: LAB
ServiceNow Instance: <training instance>
Server: LAB-RHEL11-APP01
PalletControl: <product/version or SIMULATION>
PS/Gateway: <product/version or SIMULATION>
AccessCheck: <product/version or SIMULATION>
AWS Account: <sandbox account>
```

Nếu exact vendor/version của PS/Gateway, AccessCheck hoặc PalletControl chưa được xác định:

```text
Product-specific workflow status:
UNVERIFIED — follow generic simulation only.
```

Syllabus yêu cầu exact vendor/product/version/runbook phải được xác minh trước product-specific operation. Infrastructure_Operations_14_Da…

---

## Step 2 — Xác minh current identity

Trên Linux:

```bash
whoami
id
hostnamectl
date --iso-8601=seconds
```

Read-only.

Expected:

```text
student01
uid=...
LAB-RHEL11-APP01
<timestamp>
```

Trên Windows nếu dùng Windows lab:

```powershell
whoami
hostname
Get-Date -Format o
```

---

## Step 3 — AWS identity verification

Trong AWS sandbox nếu CLI được phép:

```bash
aws sts get-caller-identity
```

Read-only.

Expected structure:

```json
{
  "UserId": "...",
  "Account": "...",
  "Arn": "..."
}
```

Không ghi temporary credentials vào evidence.

Mục đích:

> xác nhận **đúng AWS account và identity** trước khi thực hiện access workflow.

---

## Step 4 — Time synchronization awareness

Ghi timezone từng system:

```text
ServiceNow:
Gateway:
Server:
AWS:
PalletControl:
Workstation:
```

Không cần tất cả cùng timezone, nhưng timeline phải normalize được.

---

## Evidence

Thu:

```text
D11-EV-001 workstation identity
D11-EV-002 server identity
D11-EV-003 AWS caller identity
D11-EV-004 environment/version sheet
```

---

# LAB 2 — ServiceNow Incident Workflow

## Objective

Xử lý một Incident từ intake đến resolution theo operational discipline.

---

# 8. Incident fixture

Symptom:

```text
Monitoring Alert:
LAB-Order-Service health check failed.

Affected CI:
LAB-RHEL11-APP01

Reported time:
10:00 ICT
```

---

# 9. Incident Creation / Intake

Tạo hoặc mở supplied Incident.

Các fields cần verify theo training instance:

```text
Number
Short description
Description
Affected CI
Caller/requester
Assignment Group
Assigned To
Impact
Urgency
Priority
State
Opened timestamp
```

Không nhất thiết tên field giống nhau ở mọi ServiceNow implementation.

---

# 10. Short Description

Bad:

```text
Server issue
```

Better:

```text
LAB-Order-Service unavailable on LAB-RHEL11-APP01
```

Short description phải giúp triage mà không cần mở ticket.

---

# 11. Description

Ví dụ:

```text
Monitoring detected failed application health checks for
LAB-Order-Service at 10:00 ICT.

Affected CI:
LAB-RHEL11-APP01

Observed impact:
Training application unavailable to lab users.

No configuration change has been performed at intake.
```

---

# 12. Xác minh CI

Không chỉ nhận hostname.

Kiểm tra:

```text
CI = LAB-RHEL11-APP01
Environment = LAB
Service relationship = LAB-Order-Service
```

Nếu CI không tồn tại:

```text
Do not create arbitrary production-like CI.
Record discrepancy and follow CMDB/asset workflow.
```

---

# 13. Impact / Urgency / Priority

Bạn phải sử dụng matrix được cung cấp trong lab.

Ví dụ training matrix:

```text
Impact = Medium
Urgency = Medium
Priority = P3
```

Đây chỉ là **fixture**, không phải universal ServiceNow priority rule.

Trong evidence phải ghi:

```text
Priority determined according to LAB priority matrix v1.
```

---

# 14. Assignment

Ví dụ:

```text
Assignment Group:
Linux Operations Lab

Assigned To:
student01
```

Xác minh:

```text
Current ownership is explicit.
```

---

# 15. Initial Work Note

Ví dụ:

```text
10:05 ICT — Incident acknowledged.
Confirmed affected CI LAB-RHEL11-APP01.
Beginning read-only service and log verification.
No configuration-changing action performed.
```

Đây là Work Note tốt vì có:

- timestamp;
- affected CI;
- activity;
- risk boundary.

---

# 16. Technical Investigation

## Linux read-only checks

```bash
hostname
date --iso-8601=seconds
systemctl status d11-demo.service --no-pager
```

Không thay đổi configuration.

Nếu service đang failed:

```text
Active: failed
```

---

# 17. Log extraction

Ví dụ:

```bash
journalctl -u d11-demo.service \
  --since "2026-10-21 09:45:00" \
  --until "2026-10-21 10:15:00" \
  --no-pager
```

Đây là read-only evidence extraction.

Nếu hệ thống lab lưu UTC thì sử dụng UTC theo runbook.

Ghi chính xác:

```text
Command
Timezone
Start time
End time
Host
```

---

# 18. Không restart ngay

Đây là checkpoint quan trọng.

Bạn đã xác định service failed.

Không có nghĩa:

```bash
sudo systemctl restart d11-demo.service
```

được phép ngay.

Hỏi:

```text
Is restart covered by Incident runbook?

OR

Does it require Change?
```

Trong scenario Day 11 này ta giả định configuration-changing/service-impacting action cần approved Change.

---

# 19. Update Incident Work Notes

Ví dụ:

```text
10:10 ICT — Read-only verification completed.

Host: LAB-RHEL11-APP01
Service: d11-demo.service
Observed state: failed

Relevant logs captured for 09:45–10:15 ICT.

Restoration requires an approved maintenance action.
No restart performed pending Change approval.
```

Đây là decision boundary đúng.

---

# Verification

Bạn phải chứng minh:

```text
Correct Incident
Correct CI
Correct ownership
Correct priority rationale
Technical failure verified
No unauthorized action performed
```

---

# Evidence

```text
D11-EV-010 Incident export
D11-EV-011 initial ticket screenshot
D11-EV-012 systemctl status output
D11-EV-013 bounded journal export
D11-EV-014 Work Notes evidence
```

---

# LAB 3 — Service Request Workflow

Syllabus yêu cầu cả Incident và Service Request, nên không được bỏ phần này.

## Scenario

Để maintenance, engineer cần temporary privileged access.

Tạo:

```text
Service Request:
Temporary privileged access to LAB-RHEL11-APP01
```

---

# 20. Request scope

Request phải xác định:

```text
Requester:
student01

Target:
LAB-RHEL11-APP01

Environment:
LAB

Requested Access:
Temporary approved administrative capability

Purpose:
Execute CHGxxxxx associated with INCxxxxx

Start:
<window start>

Expiry:
<window end>
```

Không request:

```text
full admin forever
```

nếu chỉ cần một maintenance task.

---

# 21. Business justification

Bad:

```text
Need admin.
```

Good:

```text
Temporary access required to execute approved maintenance
for INCxxxx / CHGxxxx on LAB-RHEL11-APP01.
Access limited to maintenance window.
```

---

# 22. Validate owner

Phải biết:

```text
Who owns target system?
Who approves this level of access?
```

Không assume requester có quyền tự approve.

---

# 23. Approval

Verify:

```text
Request status
Approver
Decision
Approval timestamp
Scope
Expiry
```

Một screenshot “Approved” không đủ nếu không biết nó approve cho scope nào.

---

# 24. Evidence

```text
D11-EV-020 Service Request export
D11-EV-021 Approval record
D11-EV-022 Target/CI reference
```

---

# LAB 4 — Approved Change Workflow

Đây là central workflow của Day 11.

## Objective

Chuẩn bị và execute một controlled change với pre-check, implementation, verification và rollback.

---

# 25. Change Record

Change fixture:

```text
Change:
Restore LAB-Order-Service on LAB-RHEL11-APP01

Related Incident:
INCxxxxx

Affected CI:
LAB-RHEL11-APP01
```

---

# 26. Scope

Ví dụ:

```text
Scope:
Restart d11-demo.service following approved lab runbook.

No package installation.
No firewall modification.
No OS reboot.
No unrelated configuration modification.
```

Scope phải nói cả:

```text
What is included?
What is not included?
```

---

# 27. Risk Assessment

Ví dụ lab:

```text
Business Impact:
LAB application unavailable during restart.

Blast Radius:
One lab service on one lab host.

Reversibility:
High, provided configuration baseline remains unchanged.

Risk:
Low/Medium according to LAB matrix.
```

Đừng tự gắn `Low` nếu chưa dùng matrix.

---

# 28. Maintenance Window

Ví dụ:

```text
Window:
10:30–11:00 ICT
```

Tại 10:55 mà procedure còn cần 20 phút:

```text
STOP
```

không nên bắt đầu chỉ vì ticket Approved.

---

# 29. Pre-check Plan

Ví dụ:

```text
1. Confirm Incident.
2. Confirm Change Approved.
3. Confirm maintenance window active.
4. Confirm privileged access approved.
5. Confirm correct hostname.
6. Capture service state.
7. Capture relevant logs.
8. Confirm rollback path.
```

---

# 30. Implementation Plan

Ví dụ:

```text
1. Open approved privileged session.
2. Verify hostname and identity.
3. Restart d11-demo.service.
4. Check service state.
5. Verify local application health.
6. Verify monitoring status.
7. Monitor service for agreed observation window.
```

---

# 31. Verification Plan

Không chỉ:

```text
systemctl says active
```

Cần nhiều layer.

Ví dụ:

```text
Process/service state
Listening endpoint if applicable
Application response
Monitoring
Error log
```

---

# 32. Backout Plan

Nếu change chỉ là restart:

```text
If service fails after restart:
1. Do not perform additional unauthorized configuration changes.
2. Capture failed state and logs.
3. Follow approved recovery runbook if available.
4. If no approved recovery step exists, stop and escalate.
```

Nếu lab có configuration fixture:

```text
Restore backed-up configuration from approved pre-change copy.
Restart service.
Repeat health checks.
```

---

# 33. Stop Conditions

Ví dụ:

```text
STOP if:

Approval cannot be verified.

Maintenance window is not active.

Target hostname differs from ticket.

Pre-check shows an unexpected configuration difference.

Privileged access scope does not cover required action.

Rollback method is unavailable.

Critical unrelated alert appears.
```

---

# 34. Approval Gate

Trước implementation phải capture:

```text
Change state = Approved/authorized according to LAB workflow
Approval reference
Approval timestamp
Implementation window
```

Nếu state:

```text
Awaiting Approval
```

→ không execute.

---

# LAB 5 — Privileged Access: PS/Gateway

Phần này phải thực hiện theo actual runbook nếu có.

Nếu không có actual product:

```text
Classification = S
```

và dùng supplied fixture.

---

# 35. Request Lifecycle Checklist

Phải đi qua:

```text
Request
    ↓
Approval
    ↓
Authorized Access
    ↓
Privileged Session
    ↓
Expiry / Revocation
    ↓
Post-revocation Verification
    ↓
Audit Evidence
```

---

# 36. Verify request

Trước access:

```text
Correct user?
Correct target?
Correct account/role?
Correct expiry?
Correct ticket?
Correct approval?
```

---

# 37. Gateway Session

Nếu product hỗ trợ session ID theo actual runbook, capture:

```text
Session ID
User
Target
Start time
Ticket reference
```

Nếu capability không tồn tại:

```text
Do not invent session ID.
```

Ghi:

```text
Session reference unavailable in supplied product/version/runbook.
```

---

# 38. Credential handling

Nếu workflow dùng credential checkout:

Không capture:

```text
Password
Token
Secret
MFA seed
```

Evidence chỉ nên cho thấy:

```text
Credential checkout authorized
Time
User
Target
Result
```

---

# 39. Authorized Access Verification

Bạn phải chứng minh:

```text
Access actually works
```

chứ không chỉ approval exists.

Ví dụ read-only:

```bash
whoami
id
hostname
```

Nếu access được broker qua gateway thì follow runbook.

---

# 40. Entitlement Verification

Xác minh user có đúng entitlement.

Không chỉ:

```text
Login successful
```

Có thể user login nhưng không có required authorization.

---

# LAB 6 — AccessCheck Workflow

Tương tự, nhưng phải có **checklist riêng**, vì syllabus yêu cầu separate PS/Gateway và AccessCheck lifecycle checklist. Infrastructure_Operations_14_Da…

Không merge hai product thành:

```text
Privileged access completed
```

Bạn phải document riêng:

```text
PS/Gateway:
Request → Approval → Access → Expiry → Revocation → Audit

AccessCheck:
Request → Approval → Access → Expiry → Revocation → Audit
```

Nếu một product chỉ được simulation:

ghi:

```text
S — simulated using approved fixture
```

---

# LAB 7 — AWS IAM Privileged Access Review

AWS component tập trung vào identity/access governance.

## Step 1 — Verify caller

```bash
aws sts get-caller-identity
```

Read-only.

---

# 41. Confirm account

Đối chiếu:

```text
Expected AWS account
Actual AWS account
```

Nếu mismatch:

```text
STOP
```

Đây là stop condition cực kỳ quan trọng.

---

# 42. Confirm identity type

Theo environment, xác định:

```text
IAM user
Assumed role
Federated identity
Identity Center session
```

Không tự assume.

---

# 43. Entitlement Review

Nếu IAM user được sử dụng trong sandbox và permission cho phép read:

```bash
aws iam list-groups-for-user --user-name <USER_NAME>
```

Read-only.

Có thể kiểm tra attached policies theo approved scope:

```bash
aws iam list-attached-user-policies \
  --user-name <USER_NAME>
```

Nhưng nếu environment dùng role/federation, workflow phải theo identity model đó thay vì cố dùng IAM-user commands.

---

# 44. Role Verification

Quan trọng:

```text
Role name
AWS account
Approval
Session duration
Target scope
```

Không dựa chỉ vào role name để suy ra permission.

---

# 45. Allowed Action Test

Dùng một **non-destructive action** được instructor định nghĩa.

Ví dụ đọc resource metadata.

Expected:

```text
Success
```

Evidence:

```text
Command/API operation
Timestamp
Result
Identity
```

---

# 46. Denied Action Test

Instructor phải cung cấp một action an toàn nhưng ngoài entitlement.

Expected:

```text
AccessDenied
```

Điều này chứng minh boundary hoạt động.

Không chọn destructive action chỉ để test deny.

---

# 47. AWS Evidence

```text
D11-EV-030 caller identity
D11-EV-031 role/entitlement evidence
D11-EV-032 allowed action
D11-EV-033 denied action
D11-EV-034 access-request approval reference
```

Không chứa secret.

---

# LAB 8 — Execute Authorized Maintenance

Đến đây Incident, Request, Change và privilege đều đã được verify.

Đây mới là lúc mutating action được phép.

---

# 48. Final Pre-change Gate

Trước action, trả lời YES cho tất cả:

```text
Correct Incident?
Correct Change?
Approved?
Inside maintenance window?
Correct CI?
Correct host?
Correct privileged identity?
Correct entitlement?
Rollback ready?
Evidence collection ready?
```

Một `NO` quan trọng → STOP.

---

# 49. Pre-change evidence

```bash
date --iso-8601=seconds
hostname
whoami
systemctl status d11-demo.service --no-pager
```

Read-only.

---

# 50. Controlled mutating command

Chỉ trong LAB + approved Change:

```bash
sudo systemctl restart d11-demo.service
```

**Đây là configuration/service-changing administrative action.**

Không chạy trên production ngoài approved workflow.

---

# 51. Capture exit status

Ngay sau command:

```bash
echo $?
```

Expected:

```text
0
```

Nhưng:

```text
exit code 0
```

không đủ chứng minh application healthy.

---

# 52. Service Verification

```bash
systemctl is-active d11-demo.service
```

Expected:

```text
active
```

Sau đó:

```bash
systemctl status d11-demo.service --no-pager
```

---

# 53. Application Health Verification

Nếu demo service có HTTP endpoint:

```bash
curl --fail --show-error \
  --connect-timeout 5 \
  --max-time 10 \
  http://127.0.0.1:<PORT>/health
```

Expected:

```text
HTTP/application success according to lab fixture
```

Không dùng:

```bash
curl -k
```

chỉ để ép HTTPS pass trong một HTTPS lab; TLS verification phải giữ nếu scenario là TLS.

---

# 54. Post-change logs

```bash
journalctl -u d11-demo.service \
  --since "2026-10-21 10:25:00" \
  --until "2026-10-21 10:45:00" \
  --no-pager
```

Ghi exact window/timezone.

---

# 55. Monitoring Verification

Kiểm tra:

```text
Service health
Monitoring alarm
Application check
```

Không đóng Incident ngay khi process vừa active một giây.

Theo lab có thể yêu cầu observation window.

---

# 56. Update Change Work Notes

Ví dụ:

```text
10:35 ICT — Approved maintenance executed on
LAB-RHEL11-APP01.

Action:
Restarted d11-demo.service.

Command result:
Success.

Verification:
systemd state active.
Application health check passed.
No new critical service errors observed during validation.

Privileged session remains open only for required post-checks.
```

---

# LAB 9 — Privileged Access Expiry & Revocation

Đây là một trong những phần quan trọng nhất.

Syllabus yêu cầu verify cả:

```text
access expiration
access revocation
```

không chỉ access grant. Infrastructure_Operations_14_Da…

---

# 57. End Session

Kết thúc privileged session theo runbook.

Capture:

```text
Session end time
Session result
Session/audit reference
```

nếu product cung cấp.

---

# 58. Revoke / expire access

Thực hiện đúng mechanism của:

```text
PS/Gateway
AccessCheck
AWS IAM / role session
```

theo approved runbook.

Không dùng generic command nếu exact product unknown.

---

# 59. Effective Revocation Verification

Đây là phần nhiều người bỏ sót.

Không chỉ nhìn:

```text
Status = Revoked
```

Bạn phải kiểm tra quyền thực sự không còn usable.

Ví dụ với controlled fixture:

```text
Attempt same privileged operation after revocation.
```

Expected:

```text
Denied
```

Không thực hiện destructive operation.

---

# 60. Existing Session Caveat

Hỏi:

```text
Was existing session terminated?
```

Một entitlement bị revoke nhưng active session vẫn usable có thể nghĩa:

```text
effective access remains
```

tùy system behavior.

Nếu không xác định được:

```text
Record evidence gap
Escalate
```

---

# 61. Revocation Proof

Evidence nên có:

```text
Revocation request/reference
Revocation timestamp
Target identity
Target entitlement
Post-revocation denied test
Audit/session record
```

Syllabus coi việc thiếu revocation proof là open gap chứ không được tự coi là hoàn thành. Infrastructure_Operations_14_Da…

---

# LAB 10 — Rotation Failure Fixture

Syllabus yêu cầu kiểm tra một **rotation-failure fixture** theo runbook. Infrastructure_Operations_14_Da…

Scenario:

```text
Privileged credential rotation job = Failed
```

---

# 62. Không reset credential ngay

Đầu tiên collect:

```text
Account ID
Target
Rotation job ID
Failure timestamp
Last successful rotation
Current account status
Affected services
Error reference
Ticket
```

---

# 63. Investigation Questions

Phân tích:

```text
Target reachable?
Account locked?
Privilege sufficient?
Vault/gateway connected?
Password policy changed?
Dependent service exists?
Previous credential still in use?
```

Không khẳng định root cause nếu fixture không cung cấp evidence.

---

# 64. Escalation Record

Ví dụ:

```text
Issue:
Credential rotation failed for LAB service account.

Impact:
No immediate application outage observed.

Known:
Rotation job failed at 11:02 ICT.
Last success at <timestamp>.

Unknown:
Whether target-side policy caused failure.

Action:
No manual credential reset performed.
Escalated to privileged-access platform owner.
```

Đây là response đúng nếu authority boundary đã hết.

---

# LAB 11 — ServiceNow Bounded Extraction

Đây là additive requirement bắt buộc.

Bạn phải extract các sample liên quan:

```text
Incident
Change
Access Request
```

---

# 65. Required fields

Theo syllabus, export cần include khi được phép:

```text
Ticket number
Stable sys_id
CI/account reference
Assignment
State
Timestamps
Approval references
```

Infrastructure_Operations_14_Da…

---

# 66. Không export “all”

Define extraction boundary.

Ví dụ:

```text
Incident:
INC001234

Change:
CHG001234

Access:
REQ001234

Window:
09:30–12:00 ICT

Environment:
LAB
```

---

# 67. Record extraction method

Dùng approved method của training instance.

Có thể là:

```text
filtered list export
approved reporting/export capability
authorized API
```

nhưng exact UI/API workflow phụ thuộc instance/version/ACL.

Do syllabus không cung cấp exact version, không nên ghi nhớ một menu path cụ thể như universal procedure.

---

# 68. Count Verification

Ví dụ query cho Incident trả:

```text
Expected matching records: 1
Exported rows: 1
```

Approval query:

```text
Expected: 2
Exported: 2
```

Nếu:

```text
Expected: 1523
Export: 1000
```

phải investigate:

```text
pagination?
export cap?
filter?
permission?
```

---

# 69. Journal Evidence

Thu:

```text
Work Notes
Comments
```

chỉ trong authorized scope.

Không assume journal export luôn nằm cùng CSV ticket fields.

---

# 70. Audit Evidence

Nếu có quyền:

Thu audit records cho các fields quan trọng như:

```text
State
Assignment
Priority
CI
```

Nếu không có quyền:

```text
Evidence Gap:
Audit records inaccessible under learner ACL.
```

Không cố bypass.

---

# 71. Display History

Có thể chụp/thu History để operational review.

Nhưng trong evidence manifest phải ghi:

```text
History display used as supplementary evidence only.
Not treated as complete audit archive.
```

Điều này đúng với explicit requirement trong syllabus. Infrastructure_Operations_14_Da…

---

# 72. Attachment Evidence

Nếu ticket có attachment:

- xác minh attachment thuộc scope;
- export theo authorized method;
- hash file;
- record attachment filename;
- không mở/execute unsafe file tùy tiện.

---

# LAB 12 — Joined Maintenance Timeline

Một acceptance artifact của Day 11 là **joined timeline**.

Bạn phải ghép:

```text
ServiceNow
Privileged-access system
Server logs
```

---

# 73. Timeline Schema

Một format tốt:

| Time  | Source        | Event             | Identity  | Object   | Ticket | Evidence |
| ----- | ------------- | ----------------- | --------- | -------- | ------ | -------- |
| 10:00 | ServiceNow    | Incident opened   | user01    | APP01    | INC... | EV...    |
| 10:20 | ServiceNow    | Change approved   | approver  | APP01    | CHG... | EV...    |
| 10:29 | Gateway       | Privilege granted | student01 | APP01    | REQ... | EV...    |
| 10:32 | Linux         | admin session     | student01 | APP01    | CHG... | EV...    |
| 10:35 | Linux         | service restarted | student01 | d11-demo | CHG... | EV...    |
| 10:36 | Health check  | service healthy   | —         | APP01    | INC... | EV...    |
| 10:42 | Gateway       | session ended     | student01 | APP01    | REQ... | EV...    |
| 10:45 | Access system | access revoked    | system    | role     | REQ... | EV...    |

---

# 74. Normalize timezone

Nếu:

```text
ServiceNow = ICT
Linux = UTC
Gateway = UTC
AWS = UTC
```

chọn canonical:

```text
UTC
```

hoặc:

```text
ICT
```

và giữ original timestamp.

Ví dụ:

```text
Original:
2026-10-21T03:35:00Z

Normalized:
2026-10-21T10:35:00+07:00
```

Không overwrite raw timestamp.

---

# 75. Correlation Rules

Dùng càng nhiều key càng tốt:

```text
Ticket ID
sys_id
Hostname
CI
User ID
Account ID
Session ID
Role
Timestamp
```

Không correlate chỉ bằng:

```text
display name
```

---

# 76. Identify Gaps

Ví dụ:

```text
10:29 privilege granted
10:35 service restart
```

nhưng server log không identify actor.

Ghi:

```text
Correlation confidence = Partial
```

Không khẳng định:

```text
Gateway user definitely executed restart
```

nếu log không support.

---

# LAB 13 — PalletControl Inventory Reconciliation

Đây là phần thứ ba lớn của Module Thực hành.

## Objective

Reconcile controlled PalletControl inventory với:

```text
server inventory
endpoint inventory
CMDB
```

và classify discrepancies.

---

# 77. Synthetic Dataset

Ví dụ lab dataset:

| Asset ID | PalletControl                | CMDB          | Server/Endpoint |
| -------- | ---------------------------- | ------------- | --------------- |
| A001     | APP01 Active                 | APP01 Active  | Present         |
| A002     | DB01 Active                  | DB01 Active   | Missing         |
| A003     | WEB01 Active                 | Missing       | Present         |
| A004     | OLD01 Active                 | OLD01 Retired | Missing         |
| A005     | APP02 duplicated             | APP02 Active  | Present         |
| A006     | API01 Active, last seen 100d | API01 Active  | Present         |

Đây là **fixture do lab thiết kế**, không phải actual PalletControl schema.

---

# 78. Asset A001 — Normal

```text
PalletControl = Present
CMDB = Present
Server = Present
```

Expected:

```text
No discrepancy detected.
```

Nhưng vẫn kiểm tra freshness.

---

# 79. Asset A002 — Missing server

```text
PalletControl = Active
CMDB = Active
Server inventory = Missing
```

Không kết luận ngay:

```text
Server decommissioned.
```

Investigate:

```text
inventory scope?
collection failure?
renamed?
offline?
decommission?
```

---

# 80. Asset A003 — Missing CMDB

```text
PalletControl = Present
Server = Present
CMDB = Missing
```

Possible classification:

```text
CMDB reconciliation required
```

nhưng phải xác minh rằng asset này thực sự phải có CI.

---

# 81. Asset A004 — Lifecycle Conflict

```text
PalletControl = Active
CMDB = Retired
Server = Missing
```

Investigate:

```text
Was approved decommission completed?
Was PalletControl not updated?
Is server temporarily unavailable?
```

---

# 82. Asset A005 — Duplicate

Hai records có cùng hostname.

Không delete ngay.

Compare:

```text
Asset ID
Serial
UUID
Creation time
Last seen
Lifecycle
```

Có thể một record là historical/rebuilt server.

---

# 83. Asset A006 — Stale

```text
Last seen = 100 days ago
Expected collection = daily
```

Classification:

```text
Stale inventory
```

nhưng root cause vẫn unknown.

---

# 84. Collection / Job / Agent Check

Nếu actual PalletControl implementation cung cấp các concepts này theo runbook, inspect:

```text
Collection status
Job status
Agent status
Last successful inventory
```

Nếu product không có một concept cụ thể:

```text
Do not invent it.
```

Syllabus nói rõ “where implemented”. Infrastructure_Operations_14_Da…

---

# 85. Inventory Freshness

Tạo field:

```text
Last Inventory
Expected Frequency
Age
Fresh/Stale
```

Ví dụ:

```text
APP01:
last inventory = 2h
expected = 24h
status = Fresh
```

```text
API01:
last inventory = 100d
expected = 24h
status = Stale
```

---

# 86. Ownership Reconciliation

So sánh:

```text
PalletControl owner
CMDB support group
Service owner
```

Không coi chúng mặc định cùng nghĩa.

Ví dụ:

```text
Asset owner = Infrastructure
Support group = Linux Operations
Business service owner = E-Commerce
```

có thể hoàn toàn hợp lệ.

---

# 87. Authorized Correction

Giả sử xác định chính xác:

```text
PalletControl lifecycle should be Retired.
```

Không update ngay.

Phải có:

```text
Ticket/reference
Authority
Before evidence
Correction
After evidence
```

---

# 88. Before Evidence

Capture:

```text
Asset ID
Old value
Timestamp
Collector
Source
```

---

# 89. Change/Correction

Thực hiện đúng PalletControl product runbook.

Do exact product/version không có trong syllabus, command/menu cụ thể phải lấy từ organization documentation.

---

# 90. After Evidence

Capture:

```text
Asset ID
New value
Timestamp
User/action reference
```

Sau đó re-run reconciliation.

Expected:

```text
Discrepancy resolved
```

---

# LAB 14 — Link Asset Evidence to ServiceNow

Asset discrepancy không nên tồn tại như isolated spreadsheet.

Tạo/update relevant ticket/task theo lab process.

Ví dụ Work Note:

```text
Asset reconciliation identified lifecycle mismatch for
Asset A004.

PalletControl: Active
CMDB: Retired
Server inventory: Not present

Correction not yet executed.
Validation and authorization required.
```

Sau correction:

```text
Authorized asset correction completed under TASKxxxx.
Before/after evidence:
D11-EV-060 / D11-EV-061.

Reconciliation rerun: no lifecycle mismatch detected.
```

---

# LAB 15 — Incident Resolution

Sau technical recovery:

Verify lại:

```text
Service active
Health check passes
Monitoring healthy
No unresolved critical error
```

---

# 91. Incident Resolution Note

Ví dụ:

```text
Service restored following approved CHGxxxx.

Root symptom:
d11-demo.service failed.

Action:
Controlled restart performed during approved maintenance window.

Verification:
Service active.
Application health check passed.
Monitoring recovered.

Privileged access:
REQxxxx expired/revoked.
Post-revocation access test denied as expected.

Related evidence:
D11-EV-xxx through D11-EV-xxx.
```

---

# 92. Không overclaim root cause

Nếu bạn chỉ biết:

```text
service was failed
```

nhưng chưa biết tại sao:

Không ghi:

```text
Root Cause = Application bug
```

Ghi:

```text
Root cause not established.
Service restoration completed.
Further Problem investigation recommended if recurrence occurs.
```

---

# LAB 16 — Change Closure

Change phải có:

```text
Implementation outcome
Verification
Backout required? yes/no
Unexpected impact
Evidence
Completion time
```

Nếu rollback không dùng:

```text
Backout Plan: Not invoked.
```

Không ghi:

```text
Rollback successful
```

nếu chưa rollback.

---

# LAB 17 — Service Request Closure

Request privileged access chỉ nên closure sau khi:

```text
Access granted
Authorized work completed
Session ended
Access expired/revoked
Revocation verified
```

Không đóng ở:

```text
Access approved
```

---

# LAB 18 — Shift Handover

Ngay cả khi Incident gần hoàn tất, bạn phải luyện handover.

## Handover Template

```text
Shift Handover — Day 11

Ticket:
INCxxxx

Change:
CHGxxxx

Access Request:
REQxxxx

Affected CI:
LAB-RHEL11-APP01

Current Status:
Service restored and stable.

Actions Completed:
- Read-only diagnostics
- Approved service restart
- Health verification
- Privileged session termination
- Access revocation verification

Open Items:
- PalletControl stale record investigation

Risk:
No current service outage.
Asset inventory discrepancy remains open.

Evidence:
D11-EV-001 ... D11-EV-070

Next Action:
Asset Management team to verify inventory collection job.

Owner:
<GROUP>

SLA / Due:
<value>

Escalation:
Not required / <details>
```

---

# 93. Handover phải nói cả điều chưa làm

Ví dụ:

```text
NOT performed:
No package upgrade.
No server reboot.
No firewall change.
```

Điều này giúp shift sau không assume sai.

---

# LAB 19 — Evidence Hashing

Trên Linux:

```bash
sha256sum D11_ServiceNow_INC_raw.csv
```

Expected:

```text
<HASH>  D11_ServiceNow_INC_raw.csv
```

Windows PowerShell:

```powershell
Get-FileHash .\D11_ServiceNow_INC_raw.csv -Algorithm SHA256
```

---

# 94. Working Copy

Ví dụ:

```bash
cp D11_ServiceNow_INC_raw.csv \
   D11_ServiceNow_INC_working.csv
```

Chỉ modify working copy.

Hash raw trước và sau analysis:

```bash
sha256sum D11_ServiceNow_INC_raw.csv
```

Phải giống baseline.

---

# 95. Hash của derived file

Derived copy cũng phải có hash riêng.

Không reuse hash của raw.

Manifest:

```text
D11-EV-080
Parent = D11-EV-010
Transformation = manual filter v1
SHA256 = <derived hash>
```

---

# 96. Record Count

CSV đơn giản có header:

```bash
wc -l D11_ServiceNow_INC_raw.csv
```

Nhưng nhớ:

```text
line count ≠ always record count
```

nếu CSV chứa embedded newlines.

Vì vậy Day 12 sẽ xử lý structured CSV counting chính xác hơn.

Trong Day 11, dùng tool/export count nếu available và document method.

---

# 97. Evidence Naming Convention

Ví dụ:

```text
D11-EV-001_ENV_identity.txt
D11-EV-010_INC_raw.csv
D11-EV-011_INC_worknotes.csv
D11-EV-020_REQ_raw.csv
D11-EV-030_CHG_raw.csv
D11-EV-040_Gateway_session.txt
D11-EV-050_Server_journal.txt
D11-EV-060_PalletControl_raw.csv
D11-EV-070_Timeline.xlsx
D11-EV-080_Handover.txt
D11-MANIFEST.csv
```

---

# FAILURE INJECTION 1 — Access Approved nhưng Access Denied

Instructor inject:

```text
ServiceNow:
Approved

Privileged platform:
Login denied
```

## Bạn phải không làm

Không:

```text
reset password immediately
request global admin
bypass gateway
```

---

## Troubleshooting

Theo thứ tự:

```text
1. Correct requester?
2. Correct request?
3. Approval valid?
4. Approval scope correct?
5. Expired?
6. Entitlement provisioned?
7. Account enabled?
8. Correct target?
9. Gateway policy?
10. Authentication succeeded?
11. Authorization failed?
12. Network path?
13. Target service?
```

---

## Evidence

```text
Approval reference
Error timestamp
Target
Identity
Expected entitlement
Observed result
Audit/session reference
```

---

## Escalation

Escalate nếu:

```text
platform-side policy unknown
entitlement provisioning incomplete
privileged account owner unknown
```

---

# FAILURE INJECTION 2 — Access Expired nhưng Vẫn Dùng Được

Fixture:

```text
Request status: Expired
Privilege: still effective
```

Đây là security exception.

---

# 98. Investigation

Kiểm tra:

```text
Was entitlement actually removed?
Existing active session?
Another role grants same access?
Nested group?
Cached authorization?
```

---

# 99. Required Outcome

Không chỉ update request.

Phải:

```text
record exception
escalate
revoke through approved mechanism
verify effective access gone
collect evidence
```

Nếu không verify:

```text
Open Gap
```

---

# FAILURE INJECTION 3 — Credential Rotation Failure

Fixture:

```text
Rotation job failed
```

Bạn phải:

```text
collect job/audit details
determine target/account
inspect dependency evidence
follow product runbook
escalate if boundary reached
```

Không:

```text
manually set random password
```

nếu chưa được authorization.

---

# FAILURE INJECTION 4 — Stale PalletControl Asset

Fixture:

```text
Status = Active
Last inventory = 120 days
```

Bạn phải phân biệt:

```text
Asset stale
```

khỏi:

```text
Asset decommissioned
```

Investigate collection/agent/job where implemented.

---

# FAILURE INJECTION 5 — Duplicate Asset

Fixture:

```text
Two PalletControl records:
hostname = LAB-RHEL11-APP01
```

Một record có:

```text
Asset ID A101
Serial ABC
```

record kia:

```text
Asset ID A455
Serial XYZ
```

Đây có thể là two different systems từng dùng same hostname.

Không merge/delete chỉ dựa hostname.

---

# FAILURE INJECTION 6 — Missing ServiceNow Audit Evidence

Fixture:

Bạn nhìn History thấy thay đổi assignment, nhưng learner account không access được audit record.

Correct result:

```text
Audit evidence unavailable under current ACL.

History retained as supplementary view.

Complete audit verification requires authorized ServiceNow audit role.
```

Không ghi:

```text
Audit record does not exist.
```

---

# FAILURE INJECTION 7 — Export Count Mismatch

ServiceNow filtered view:

```text
1250 matching records
```

Export:

```text
1000 records
```

Bạn phải investigate:

```text
export limit
pagination
query scope
ACL
```

Không hash 1000 records rồi tuyên bố:

```text
Complete evidence
```

---

# FAILURE INJECTION 8 — Wrong Host

Ticket:

```text
LAB-RHEL11-APP01
```

SSH session:

```text
hostname
LAB-RHEL11-APP02
```

Correct action:

```text
STOP immediately.
```

Không:

```text
“APP02 cũng là lab nên làm luôn.”
```

Capture mismatch và reopen correct connection.

---

# FAILURE INJECTION 9 — Maintenance Window Expired

Change approved nhưng:

```text
Window ended: 11:00
Current time: 11:05
```

Correct:

```text
Do not execute.
```

Approval không automatically extend window.

Update/escalate according to Change process.

---

# FAILURE INJECTION 10 — Pre-check differs from Change baseline

Change says:

```text
Expected service version: v1
```

Actual:

```text
v2
```

Nếu difference có thể ảnh hưởng procedure:

```text
STOP.
```

Không blindly continue.

---

# 100. Troubleshooting Checkpoints tổng quát

Khi lab thất bại, luôn phân loại trước.

### Workflow problem

Ví dụ:

```text
wrong state
missing approval
incorrect assignment
expired window
```

### Identity problem

```text
wrong user
wrong account
disabled account
```

### Authentication problem

```text
credential rejected
MFA failure
session creation failure
```

### Authorization problem

```text
login works
required action denied
```

### Technical target problem

```text
server offline
service unavailable
network failure
```

### Data-quality problem

```text
stale asset
duplicate CI
incorrect owner
```

### Evidence problem

```text
missing logs
ACL restriction
retention gap
partial export
timezone ambiguity
```

Đây giúp tránh jumping directly to high-risk fixes.

---

# 101. Rollback / Cleanup

Sau lab:

### Server

Nếu configuration fixture đã thay đổi:

```text
restore approved baseline
verify service
```

Nếu chỉ restart:

```text
no configuration rollback required
```

nhưng verify final state.

### Privileged access

Bắt buộc:

```text
end sessions
revoke temporary privilege
verify denied access
```

### AWS

Remove/revoke temporary lab entitlement theo instructor/runbook nếu lab đã provision entitlement.

### ServiceNow

Không delete evidence-bearing tickets chỉ để “cleanup”.

Giữ trạng thái theo training workflow.

### PalletControl

Revert temporary synthetic correction nếu lab design yêu cầu reset fixture.

### Evidence

Preserve:

```text
raw export
hash
manifest
derived reports
```

---

# 102. “Cleanup” không có nghĩa delete evidence

Operational cleanup:

```text
remove temporary privilege
restore test state
close sessions
```

khác với:

```text
delete audit trail
delete logs
delete ticket history
```

Không được xóa evidence để lab “sạch”.

---

# 103. Acceptance Criteria — ServiceNow

Bạn đạt phần ServiceNow nếu reviewer có thể xác minh:

```text
Incident lifecycle complete
Request lifecycle complete
Change lifecycle complete
Correct ownership
Priority rationale
Impact recorded
Troubleshooting documented
Approval linked
Implementation documented
Backout plan present
Verification recorded
Closure/handover present
```

---

# 104. Acceptance Criteria — Privileged Access

Phải chứng minh:

```text
Correct identity
Correct target
Approved entitlement
Authorization
Access result
Session/audit reference
Expiry
Revocation
Post-revocation verification
```

Một request “Closed” không đủ.

---

# 105. Acceptance Criteria — PalletControl

Phải có:

```text
Product/version/runbook record
Inventory export
Freshness analysis
Ownership reconciliation
Missing/duplicate/stale findings
Authorized correction if applicable
Before evidence
After evidence
Reconciliation rerun
```

---

# 106. Acceptance Criteria — Evidence

Phải có:

```text
Raw exports preserved
Working copies separated
Record counts
Hashes
Timezone
Filters
Source IDs
Ticket IDs
Limitations
Evidence gaps
```

Syllabus yêu cầu matching hash không được dùng để tuyên bố rằng query/source/window ban đầu chắc chắn đầy đủ hoặc chính xác. Infrastructure_Operations_14_Da…

---

# 107. Deliverable 1 — ServiceNow Extraction Recipe

Bạn phải tạo một procedure mà người khác có thể reproduce.

Ví dụ structure:

```text
ServiceNow Extraction Recipe — Day 11

Environment:
LAB ServiceNow

Records:
INCxxxx
CHGxxxx
REQxxxx

Filters:
Exact ticket numbers

Required Fields:
number
sys_id where authorized
state
assignment
CI
timestamps
approval references

Journal:
Collected separately

Audit:
Collected where permitted

Attachments:
Approved scope only

Export Counts:
<values>

Timezone:
ICT

Limitations:
<values>

Hash:
See evidence manifest
```

---

# 108. Deliverable 2 — Joined Timeline

Phải correlate:

```text
ServiceNow
Gateway/Access
Server
```

Không chỉ copy/paste screenshots.

---

# 109. Deliverable 3 — Evidence Manifest

Có đầy đủ metadata theo syllabus. Infrastructure_Operations_14_Da…

---

# 110. Deliverable 4 — PalletControl Discrepancy Record

Ví dụ:

```text
Finding ID:
D11-PC-001

Asset:
A004

Observed:
PalletControl Active
CMDB Retired
Server not found

Classification:
Lifecycle discrepancy

Root Cause:
Not yet established

Owner:
Asset Management

Correction:
Pending authorization

Evidence:
D11-EV-060
```

Không overclaim.

---

# 111. Deliverable 5 — PalletControl Maintenance Record

Nếu actual correction được làm:

```text
Before State
Ticket
Authorization
Correction
After State
Verification
Timestamp
Operator
Evidence
```

---

# 112. Deliverable 6 — PS/Gateway Lifecycle Checklist

Ví dụ:

```text
[ ] Request
[ ] Identity verified
[ ] Target verified
[ ] Entitlement verified
[ ] Approval
[ ] Access start
[ ] Session/audit reference
[ ] Authorized work
[ ] Session ended
[ ] Expiry/revocation
[ ] Post-revocation test
[ ] Evidence
[ ] Failure/escalation scenario
```

---

# 113. Deliverable 7 — AccessCheck Lifecycle Checklist

Tạo riêng checklist tương đương.

Không copy một checklist và giả định product behavior giống nhau; các fields generic có thể giống, nhưng product-specific mapping phải theo actual runbook.

---

# 114. Deliverable 8 — AWS IAM Access Evidence

Bao gồm:

```text
AWS account
Identity ARN / authorized identifier
Role/account reference
Approval ticket
Allowed read test
Denied test
Expiry/revocation evidence where applicable
```

Mask/redact sensitive data theo policy.

---

# 115. Deliverable 9 — Maintenance Record

Phải cho reviewer trace:

```text
Incident
→ Change
→ Approval
→ Privileged Access
→ Action
→ Verification
→ Revocation
```

---

# 116. Deliverable 10 — Handover Record

Phải đủ để shift khác tiếp tục xử lý without repeating risky actions.

---

# 117. Suggested Evidence Index

Một final Day 11 package có thể trông như sau:

```text
Day11/
│
├── 00_Control/
│   ├── Environment_Baseline.txt
│   └── Day11_Scope.txt
│
├── 01_ServiceNow/
│   ├── INC_raw.csv
│   ├── INC_worknotes.csv
│   ├── CHG_raw.csv
│   ├── REQ_raw.csv
│   ├── Approvals_raw.csv
│   └── Audit_or_Gap_Record.txt
│
├── 02_PrivilegedAccess/
│   ├── PS_Gateway_Checklist.xlsx
│   ├── AccessCheck_Checklist.xlsx
│   ├── AWS_Identity.txt
│   ├── Access_Test.txt
│   ├── Revocation_Test.txt
│   └── Rotation_Failure_Record.txt
│
├── 03_Server/
│   ├── PreCheck.txt
│   ├── Service_Status_Pre.txt
│   ├── Journal_Pre.txt
│   ├── Implementation.txt
│   ├── Service_Status_Post.txt
│   └── Journal_Post.txt
│
├── 04_PalletControl/
│   ├── Inventory_raw.csv
│   ├── Reconciliation.xlsx
│   ├── Discrepancies.xlsx
│   └── Correction_Before_After.txt
│
├── 05_Timeline/
│   └── Joined_Timeline.xlsx
│
├── 06_Handover/
│   └── Shift_Handover.txt
│
└── 99_Manifest/
    ├── Evidence_Manifest.xlsx
    └── SHA256SUMS.txt
```

Đây là suggested lab organization, không phải mandatory ServiceNow/PalletControl directory standard.

---

# 118. Practical Reasoning Scenario — Ticket đúng, Action sai

Giả sử:

```text
Change Approved:
Restart service on APP01
```

Engineer thực hiện:

```text
Restart APP01
Modify configuration
Install package
```

Chỉ restart nằm trong scope.

Package install và config modification là:

```text
scope expansion
```

dù technical result cuối cùng tốt.

Correct handling:

```text
STOP before additional changes
Update investigation
Obtain appropriate authorization
```

---

# 119. Scenario — Approval đúng nhưng Target sai

```text
Approval:
APP01

Session:
APP02
```

Không được dựa vào:

```text
same application cluster
```

để tiếp tục.

Approval scope không cover APP02 nếu ticket không nói vậy.

---

# 120. Scenario — Change thành công nhưng access không revoke

```text
Service restored = PASS

Temporary privilege still active = FAIL
```

Day 11 chưa hoàn thành.

Đây là lý do syllabus coi missing revocation proof là open gap. Infrastructure_Operations_14_Da…

---

# 121. Scenario — Asset correction đúng nhưng không có ticket

Technical data có thể đã chính xác, nhưng auditability yếu.

Cần:

```text
authorized correction reference
```

đặc biệt nếu asset record là controlled operational data.

---

# 122. Scenario — ServiceNow export có hash đúng nhưng thiếu 20%

Hash chỉ chứng minh:

```text
export file unchanged
```

không chứng minh:

```text
export complete
```

Phải kiểm tra query count, page/export limits và missing intervals. Đây là explicit evidence principle của syllabus. Infrastructure_Operations_14_Da…

---

# 123. L1 Practical Boundary

Trong lab, learner ở L1-like scope nên thành thạo:

```text
ticket intake
read-only verification
priority/assignment checks
approved evidence extraction
known runbook steps
basic access verification
asset discrepancy identification
handover
escalation
```

Không tự thực hiện high-risk unapproved actions.

---

# 124. L2 Practical Boundary

L2-like scope có thể:

```text
deeper troubleshooting
approved service action
approved configuration maintenance
privileged session
evidence correlation
controlled rollback
```

nhưng vẫn không có nghĩa:

```text
self-approve
bypass policy
create own emergency authority
```

---

# 125. Escalation Package Template

Khi cần escalate:

```text
Ticket:
INCxxxx

Affected CI:
LAB-RHEL11-APP01

Business/Service Impact:
<description>

Observed:
<facts>

Expected:
<expected state>

Verified:
<facts>

Not Verified:
<unknowns>

Actions Performed:
<authorized actions>

Actions NOT Performed:
<important boundaries>

Evidence:
D11-EV-...

Current Risk:
<risk>

Requested Escalation:
<team/product owner>

Reason:
Authority / knowledge / product boundary exceeded.
```

---

# 126. Day 11 Practical Completion Checklist

## ServiceNow

- [ ] Incident được xử lý.
- [ ] Correct CI.
- [ ] Assignment rõ.
- [ ] Impact/Urgency/Priority được justify.
- [ ] Troubleshooting documented.
- [ ] Service Request được xử lý.
- [ ] Change được xử lý.
- [ ] Approval được verify.
- [ ] Maintenance window được verify.
- [ ] Implementation plan hoàn chỉnh.
- [ ] Backout plan hoàn chỉnh.
- [ ] Stop conditions rõ.
- [ ] Resolution evidence có.
- [ ] Closure/handover có.
- [ ] Bounded export hoàn thành.
- [ ] Audit/history distinction được ghi đúng.

## Privileged Access

- [ ] Identity verified.
- [ ] Account/target verified.
- [ ] Entitlement verified.
- [ ] Approval verified.
- [ ] Access thành công.
- [ ] Authorized session evidence.
- [ ] Expiry/revocation thực hiện.
- [ ] Effective revocation verified.
- [ ] PS/Gateway checklist riêng.
- [ ] AccessCheck checklist riêng.
- [ ] AWS IAM scenario hoàn thành.
- [ ] Rotation failure scenario được xử lý/escalate.
- [ ] Không lộ credential.

## PalletControl

- [ ] Product/version/runbook được ghi hoặc marked unknown.
- [ ] Inventory export.
- [ ] Freshness check.
- [ ] Ownership check.
- [ ] Missing asset investigation.
- [ ] Duplicate investigation.
- [ ] Stale asset investigation.
- [ ] Lifecycle mismatch investigation.
- [ ] Authorized correction nếu applicable.
- [ ] Before/after evidence.
- [ ] Reconciliation rerun.

## Evidence

- [ ] Raw exports giữ nguyên.
- [ ] Working copies tách riêng.
- [ ] File counts.
- [ ] Record counts.
- [ ] SHA-256.
- [ ] Timezone.
- [ ] Exact query/window.
- [ ] Ticket references.
- [ ] Stable IDs where permitted.
- [ ] Evidence gaps documented.
- [ ] Joined timeline.
- [ ] Evidence manifest.
- [ ] Shift handover.

---

# 127. Khi nào Day 11 được coi là chưa hoàn thành?

Ngay cả khi service cuối cùng hoạt động, Day 11 vẫn **chưa hoàn thành** nếu:

```text
privileged access revocation chưa verify
```

hoặc:

```text
critical evidence missing mà không được ghi gap
```

hoặc:

```text
simulated product được khai là actual hands-on
```

hoặc:

```text
ServiceNow extraction không có count/scope rõ
```

hoặc:

```text
PalletControl discrepancy bị “sửa” mà không có authorization/evidence
```

hoặc:

```text
unknown facts bị biến thành assumed facts
```

Syllabus yêu cầu mọi mandatory acceptance check phải pass trên lab/fixture có sẵn và unknown production facts phải tiếp tục được ghi là unknown. Infrastructure_Operations_14_Da…

---

# 128. Practical Cheat Sheet

```text
BEFORE ACTION
------------
Ticket?
Correct CI?
Correct environment?
Approval?
Maintenance window?
Privilege?
Rollback?
Stop conditions?


DURING ACTION
-------------
Correct host?
Correct user?
Stay in scope?
Timestamp?
Capture result?
No secret leakage?


AFTER ACTION
------------
Technical health?
Application health?
Monitoring?
Logs?
Ticket updated?
Session ended?
Privilege revoked?
Revocation verified?
Asset/CMDB reconciled?
Evidence hashed?
Handover complete?
```

---

# 129. Day 11 Operational Golden Rule

Có bốn trạng thái bạn luôn phải so sánh:

```text
1. Requested state
2. Approved state
3. Effective technical state
4. Evidenced state
```

Ví dụ:

```text
Requested:
Temporary admin until 11:00

Approved:
Temporary admin until 11:00

Effective at 10:30:
Admin active

Effective at 11:05:
Admin still active

Evidence:
Access test succeeds
```

Kết luận:

```text
Access-control exception.
```

Không phải:

```text
Request closed successfully.
```

Ngược lại:

```text
Requested:
Temporary admin until 11:00

Approved:
Temporary admin until 11:00

Effective at 11:05:
Denied

Audit:
Revoked 11:00

Evidence:
Post-revocation test denied
```

→ lifecycle có evidence tốt.

---

# 130. Kết quả cuối cùng bạn phải đạt được

Khi thực sự thành thạo Module Thực hành Day 11, bạn phải có khả năng nhận một scenario kiểu:

> “Production-like application gặp sự cố; engineer cần privilege để thực hiện maintenance; ServiceNow có Incident/Change; asset data có discrepancy.”

và suy luận theo đúng trình tự:

```text
Tôi chưa thay đổi gì.

Tôi xác minh đúng ticket và CI trước.

Tôi xác minh Impact/Urgency/Priority và ownership.

Tôi thực hiện read-only investigation.

Tôi xác định action nào cần Change.

Tôi kiểm tra implementation/backout/stop conditions.

Tôi xác minh approval và maintenance window.

Tôi xác minh privileged-access request.

Tôi xác minh identity, entitlement và expiry.

Tôi thực hiện đúng approved scope.

Tôi verify technical service bằng nhiều signal.

Tôi thu log và audit evidence.

Tôi kết thúc session.

Tôi xác minh access thực sự đã revoke.

Tôi correlate ServiceNow, privileged-access và server timeline.

Tôi reconcile asset/CMDB.

Tôi không tự sửa discrepancy nếu chưa có authority.

Tôi giữ raw evidence, count, timestamp và SHA-256.

Tôi ghi mọi unknown/evidence gap.

Tôi resolve/close đúng workflow.

Tôi bàn giao những gì còn mở cho shift tiếp theo.
```

Đó chính là năng lực mà Day 11 muốn xây dựng: **không chỉ biết thao tác ServiceNow, AWS IAM, PalletControl hay privileged-access products riêng lẻ, mà biết vận hành chúng như một chuỗi enterprise operations có authorization, least privilege, rollback, verification, evidence, audit traceability và escalation boundary**. Theo syllabus, output cuối Day 11 phải bao gồm **ServiceNow extraction recipe, joined timeline/evidence manifest, PalletControl discrepancy/maintenance record, cùng separate PS/Gateway và AccessCheck lifecycle checklists**; simulation chỉ chứng minh workflow understanding, còn actual product proficiency cần actual product lab và instructor sign-off. Infrastructure_Operations_14_Da…
