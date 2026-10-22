# Day 13 — Module Lý thuyết

## Identity Governance, 90-Day IAM Review, Vulnerability Classification, Multi-Sheet Reconciliation & Automation Assurance

Day 13 thuộc **Operations Automation, IAM Audit & Integrated Practice**. Nếu Day 12 tập trung nhiều vào việc đọc, sửa, chạy và kiểm thử Excel/VBA, Python, Batch, PowerShell cùng các vấn đề CSV/TXT/encoding, thì Day 13 đặt những kỹ năng đó vào một bài toán enterprise thực sự: **periodic IAM certification**, đặc biệt là **90-day IAM review**, kết hợp nhiều nguồn identity/access data, đưa ra reviewer decision có thể audit, đồng thời quản lý vulnerability findings và chứng minh automation tạo kết quả chính xác, repeatable và traceable. Infrastructure_Operations_14_Da…

Phần dưới đây chỉ tập trung vào **Module Lý thuyết**. Chúng ta chưa xây workbook hay chạy lab; thay vào đó, mục tiêu là hiểu đủ sâu để khi sang Module Thực hành, bạn biết **tại sao từng sheet, từng field, từng control và từng test case lại tồn tại**.

---

# 1. Syllabus coverage checklist — Module Lý thuyết Day 13

Toàn bộ các chủ đề phải cover theo syllabus có thể ánh xạ như sau. Infrastructure_Operations_14_Da…

| Nhóm                  | Nội dung bắt buộc                                                                             |
| --------------------- | --------------------------------------------------------------------------------------------- |
| Identity Governance   | identity governance, periodic IAM certification, 90-day IAM review objectives                 |
| Review boundaries     | review scope, cutoff dates, as-of time, timezone, inclusive/exclusive boundary                |
| Account population    | user accounts, privileged accounts, ownership, active/disabled/expired                        |
| Activity evidence     | last sign-in, last-used permission evidence, unknown evidence                                 |
| IAM exceptions        | dormant users, stale entitlements, orphaned accounts, duplicate IDs                           |
| Role & privilege      | role changes, excessive permissions, least privilege, separation of duties                    |
| Special identities    | service accounts, break-glass accounts                                                        |
| Evidence sources      | AWS IAM credential/access reports, privileged-ID exports, PS/Gateway, AccessCheck, ServiceNow |
| Reconciliation        | cross-system identity reconciliation, source-to-target matching, composite keys               |
| Reviewer decisions    | keep, revoke, modify, investigate, rationale, owner, reviewer, due date, evidence             |
| Vulnerability         | CVE/CVSS, exploitability, exposure, asset criticality, business impact                        |
| Risk handling         | severity, remediation priority, exception acceptance, compensating controls                   |
| Remediation lifecycle | ownership, remediation tracking, retest evidence                                              |
| Excel design          | multi-sheet automation, Control/Raw/Reconciliation/Exceptions/Decisions/etc.                  |
| Data controls         | normalization, lookup consistency, exception classification                                   |
| Evidence integrity    | timestamps, source metadata, SHA-256, version control, reproducibility                        |
| Automation testing    | positive, negative, boundary, malformed, incomplete, duplicate, inconsistent                  |
| Advanced testing      | regression, idempotency, dry-run, result comparison                                           |
| Operations assurance  | error logging, defect tracking, operational acceptance criteria                               |

Ngoài ra, syllabus quy định một evidence contract chung: raw evidence phải được giữ nguyên, transformations phải traceable, hashes và counts phải được ghi nhận, và matching hash không đồng nghĩa source ban đầu chắc chắn truthful hoặc complete. Infrastructure_Operations_14_Da…

---

# 2. Identity Governance là gì?

## What

**Identity Governance** là tập hợp các governance process và technical control dùng để đảm bảo đúng identity có đúng access vào đúng resource, vì đúng business purpose, trong đúng thời gian, được đúng người approve và có thể chứng minh được toàn bộ lifecycle đó sau này.

Có thể hình dung:

```text
Person / Workload
        │
        ▼
     Identity
        │
        ▼
     Account
        │
        ▼
      Role
        │
        ▼
   Entitlement
        │
        ▼
      Asset
        │
        ▼
 Business Purpose
        │
        ▼
 Approval / Review
```

Trong Infrastructure Operations, câu hỏi không chỉ là:

```text
"Account này có tồn tại không?"
```

mà phải mở rộng thành:

```text
Account thuộc về ai?
Tại sao account tồn tại?
Account dùng cho system nào?
Được cấp những quyền gì?
Ai đã approve?
Approval còn phù hợp không?
User còn giữ business role đó không?
Access thực tế có giống access được approve không?
Có evidence chứng minh không?
```

Đó chính là tư duy governance.

## Why

Trong enterprise, access thường tích tụ theo thời gian. Một employee có thể chuyển department, thay role, được cấp temporary privileged access, sau đó access cũ không được cleanup. Service account có thể tồn tại sau khi application bị retire. Project administrator có thể vẫn giữ Production permission dù project đã kết thúc.

Kết quả là xuất hiện **privilege accumulation** và **access drift**.

Ví dụ:

```text
Ngày 1:
User A = System Engineer
Access = Server Admin

6 tháng sau:
User A = Project Coordinator

Actual Access:
Server Admin vẫn còn
```

Identity Governance tìm và xử lý những drift kiểu này.

## How

Governance thường sử dụng ba loại control:

**Preventive control** cố ngăn access sai ngay từ đầu, ví dụ approval workflow, least-privilege role design và SoD checks.

**Detective control** tìm access đã sai hoặc trở nên sai, ví dụ periodic IAM certification.

**Corrective control** xử lý sau khi finding được xác nhận, ví dụ revoke, modify hoặc reassign owner.

Day 13 tập trung mạnh nhất vào detective control thông qua periodic review, nhưng reviewer phải hiểu nó liên quan cả preventive và corrective control.

## Security implications

Identity governance kém có thể dẫn đến:

```text
Dormant privileged account
Orphaned admin account
Excessive privilege
Unauthorized access persistence
Untracked service account
SoD violation
Approval expired but access remains
```

Các trường hợp này không nhất thiết đã là security incident, nhưng đều có thể làm tăng attack surface.

---

# 3. Periodic IAM Certification

## What

**IAM certification** hay **access certification/access review** là quá trình định kỳ trong đó một population identity/access được đưa cho reviewer có thẩm quyền để xác nhận access nào:

```text
Keep
Revoke
Modify
Investigate
```

Mục tiêu không phải chỉ tạo spreadsheet rồi ký tên.

Một certification tốt phải tạo ra một chuỗi traceability:

```text
Source account
    ↓
Current entitlement
    ↓
Identity / owner
    ↓
Business role
    ↓
Approval
    ↓
Reviewer decision
    ↓
Remediation task
    ↓
Effective-access recheck
```

## Periodic nghĩa là gì?

Review có thể được tổ chức theo policy với cadence nhất định:

```text
Monthly
Quarterly
Every 90 days
Semi-annually
Annually
```

Day 13 dùng **90-day IAM review** làm scenario chính.

Điểm syllabus nhấn mạnh rất mạnh là:

> **“review every 90 days” và “inactive for 90 days” là hai rule khác nhau.**

Infrastructure_Operations_14_Da…

Ví dụ policy có thể là:

```text
Review frequency = every 90 days
Dormancy threshold = 60 days
```

hoặc:

```text
Review frequency = every 90 days
Dormancy threshold = 120 days
```

Không được suy luận threshold chỉ từ review cadence.

---

# 4. Mục tiêu của 90-Day IAM Review

Một review đúng nghĩa thường cố trả lời bốn câu hỏi lớn.

Thứ nhất:

```text
Access population có complete không?
```

Tức reviewer có nhận được đúng population cần review không.

Thứ hai:

```text
Access có appropriate không?
```

Có phù hợp current role/business purpose không?

Thứ ba:

```text
Access có được approved không?
```

Không chỉ tồn tại, mà phải có authorization hợp lệ.

Thứ tư:

```text
Reviewer decision có được thực thi và verify không?
```

Nếu reviewer chọn `Revoke`, ticket đóng chưa đủ. Effective access phải thực sự bị loại bỏ.

Đây là distinction giữa:

```text
Governance record state
```

và:

```text
Actual technical state
```

Hai thứ có thể khác nhau.

---

# 5. Review Scope

## What

**Review scope** xác định review này bao phủ cái gì.

Ví dụ scope có thể là:

```text
AWS IAM privileged identities
Windows local administrators
Linux sudo-enabled accounts
PS/Gateway privileged IDs
AccessCheck entitlements
Production service accounts
Break-glass accounts
```

Một report có thể rất chính xác về dữ liệu được import nhưng vẫn sai governance nếu population bị thiếu.

Ví dụ:

```text
Total AWS accounts thực tế = 500
Export chỉ có 350
Reconciliation của 350 record = 100% chính xác
```

Review vẫn incomplete.

## Vì sao scope là control quan trọng?

Nếu không declare scope, auditor không thể phân biệt:

```text
"No exceptions found"
```

với:

```text
"Important systems were never included"
```

Do đó review phải nói rõ:

```text
Included systems
Excluded systems
Account types
Environments
Time boundary
Known limitations
```

## Common mistake

Một lỗi rất phổ biến là bắt đầu bằng data:

```text
"Đây là file IAM.csv, hãy tìm account không hoạt động."
```

Thứ tự đúng về governance phải là:

```text
Policy
→ Scope
→ Time boundary
→ Population
→ Evidence
→ Analysis
```

chứ không phải ngược lại.

---

# 6. Review period, As-of Time và Cutoff

Đây là một trong những phần quan trọng nhất của Day 13.

## As-of Time

**As-of time** là thời điểm reference mà tại đó bạn đánh giá trạng thái.

Ví dụ:

```text
As-of:
2026-10-01 00:00:00 UTC
```

Mọi calculation liên quan dormancy, expiration hoặc age phải reference tới cùng điểm này.

Nếu analyst A chạy report lúc 08:00 Bangkok còn analyst B chạy lúc 17:00 UTC mà không có as-of cố định, kết quả có thể khác.

## Cutoff

Giả sử inactivity threshold là:

```text
90 days
```

thì cutoff được tính từ as-of.

Conceptually:

```text
Cutoff = AsOfTime - 90 days
```

Nhưng business rule vẫn phải định nghĩa comparator:

```text
LastActivity < Cutoff
```

hay:

```text
LastActivity <= Cutoff
```

Hai rule này khác nhau tại boundary.

Syllabus yêu cầu declare rõ inclusive/exclusive boundaries. Infrastructure_Operations_14_Da…

---

# 7. Boundary reasoning

Hãy giả sử:

```text
Cutoff = 2026-07-03 00:00:00 UTC
```

Ba fixture quan trọng:

```text
A = 2026-07-02 23:59:59
B = 2026-07-03 00:00:00
C = 2026-07-03 00:00:01
```

Nếu policy là:

```text
Inactive when LastActivity <= Cutoff
```

thì A và B có thể bị classify dormant.

Nếu policy là:

```text
Inactive when LastActivity < Cutoff
```

thì chỉ A.

Một automation có thể xử lý hàng triệu record đúng về syntax nhưng sai comparator một ký tự, từ:

```text
<
```

thành:

```text
<=
```

và tạo hàng nghìn incorrect decisions.

Đó là lý do **boundary testing** là mandatory.

---

# 8. Timezone normalization

Một review enterprise có thể nhận dữ liệu từ nhiều system:

```text
AWS
Windows
Linux
ServiceNow
Privileged Access product
Asset Management
```

Timestamp có thể được lưu dưới:

```text
UTC
Local timezone
Timezone offset
Date-only
Locale-specific text
```

Nếu analyst không normalize, cùng một event có thể bị tính ở ngày khác.

Ví dụ:

```text
2026-07-03 00:30 ICT
```

tương đương:

```text
2026-07-02 17:30 UTC
```

Nếu cutoff là UTC midnight, classification có thể thay đổi.

### Operational rule

Không được chỉ lưu:

```text
2026-07-03
```

nếu business rule phụ thuộc giờ/phút.

Phải biết:

```text
Timestamp
Timezone
Normalization rule
```

## Verification

Reviewer phải có khả năng trả lời:

```text
Original timestamp là gì?
Original timezone là gì?
Normalized timestamp là gì?
Conversion rule nào được dùng?
```

Nếu không trả lời được, boundary decision có thể không defensible.

---

# 9. Identity, Account và Entitlement không phải cùng một thứ

Đây là foundation quan trọng.

## Identity

Identity đại diện cho principal.

Có thể là:

```text
Human employee
Contractor
Service identity
Application workload
Emergency identity
```

## Account

Account là representation của identity trên một system cụ thể.

Một người có thể có:

```text
Corporate account
AWS account identity
Linux admin account
Windows admin account
Database privileged ID
```

## Entitlement

Entitlement là permission/access assignment.

Ví dụ:

```text
Role
Group membership
Policy
Administrative privilege
Application permission
```

Quan hệ thường là:

```text
1 Identity
   │
   ├── Account A
   │     ├── Entitlement 1
   │     └── Entitlement 2
   │
   └── Account B
         └── Entitlement 3
```

Nếu review chỉ kiểm account mà không kiểm entitlement, excessive access có thể bị bỏ qua.

---

# 10. User Account Inventory

Inventory là population cơ sở của IAM review.

Một conceptual record tốt thường chứa các nhóm field như:

```text
Identity identifier
Account identifier
System
Environment
Account type
Status
Owner
Role
Entitlement
Creation timestamp
Expiration
Last sign-in
Last-use evidence
Approval reference
Source reference
```

Điểm quan trọng là **không field nào tự mình trả lời tất cả câu hỏi**.

Ví dụ:

```text
Status = Active
```

chỉ nói account active, không nói account legitimate.

```text
Owner = EMP001
```

chỉ nói mapping owner, không nói entitlement appropriate.

```text
Last login = Yesterday
```

chỉ nói recent authentication, không chứng minh privileged permission cần thiết.

---

# 11. Privileged Account Inventory

Privileged identity cần review cẩn thận hơn vì blast radius lớn hơn.

Ví dụ privilege có thể liên quan:

```text
Server administration
Cloud administration
Firewall administration
Hypervisor administration
Backup administration
Identity administration
Database administration
```

Không nên dùng một field chung:

```text
Privileged = Yes
```

mà không biết privilege scope.

Một account có thể:

```text
Privileged on Lab
```

nhưng không privileged trên Production.

Do đó context:

```text
System
Environment
Role
Scope
```

rất quan trọng.

---

# 12. Account Ownership

## What

**Account owner** là người hoặc function chịu trách nhiệm business/operational cho account.

Nó không nhất thiết là người đang sử dụng credential.

Ví dụ service account:

```text
Account:
svc_backup_prod

Interactive user:
None

Business/Technical Owner:
Backup Operations Team
```

## Why

Khi reviewer gặp:

```text
Unknown account
Unknown purpose
```

owner là điểm đầu tiên để xác minh.

Nếu account không có owner, đó là governance problem.

## Missing owner

Không nên tự động:

```text
Missing Owner → Revoke
```

Thay vào đó:

```text
Missing Owner
→ Investigate
→ Determine dependency
→ Establish ownership or remove safely
```

Đặc biệt với service account, blind revocation có thể gây outage.

---

# 13. Active, Disabled, Expired và Deleted

Các trạng thái cần phân biệt.

## Active

Account được system cho phép hoạt động.

Không có nghĩa là approved.

## Disabled

Object vẫn tồn tại nhưng authentication/access bị vô hiệu hóa theo implementation.

Không có nghĩa toàn bộ entitlement history đã biến mất.

## Expired

Account hoặc entitlement vượt validity period.

## Deleted

Object đã bị remove khỏi source system.

## Revoked

`Revoked` thường mô tả governance/access decision.

Điểm rất quan trọng:

```text
Approval = Revoked
```

không tự động chứng minh:

```text
Effective access = Removed
```

---

# 14. Revoked-but-still-active Access

Syllabus cố tình đưa trường hợp này vào dataset Day 13. Infrastructure_Operations_14_Da…

Ví dụ:

```text
ServiceNow:
Access Request = Revoked

PS/Gateway:
Privileged entitlement = Active
```

Đây là mismatch giữa governance plane và technical plane.

Có thể do:

```text
Provisioning failure
Manual process missed
Integration failure
Incorrect account mapping
Delayed synchronization
Wrong ticket closed
```

### Root-cause thinking

Không nên chỉ:

```text
"PS/Gateway sai."
```

Phải trace:

```text
Who requested revoke?
When approved?
When implementation expected?
Which technical account?
Which entitlement?
Was provisioning triggered?
Was it successful?
Was effective access rechecked?
```

---

# 15. Last Sign-In không bằng Last Permission Use

Đây là điểm rất hay bị nhầm.

## Last Sign-In

Phản ánh gần nhất identity authenticate hoặc sign in, tùy source definition.

## Last-Used Permission Evidence

Phản ánh evidence cho việc một permission/service/entitlement đã được sử dụng.

User có thể sign in nhưng không dùng admin privilege.

Ví dụ:

```text
Last sign-in:
Yesterday

Domain Admin permission:
Not observed in 180 days
```

Trường hợp này không chứng minh user dormant, nhưng có thể đặt câu hỏi:

```text
Privileged entitlement còn cần không?
```

Ngược lại, service account có thể không có normal interactive sign-in nhưng workload vẫn sử dụng credential/access.

---

# 16. Blank Last-Used Value

Một rule rất quan trọng từ syllabus:

> blank hoặc unsupported last-used value phải được xem là **unknown evidence**, không phải automatic proof of inactivity. Infrastructure_Operations_14_Da…

Ví dụ dữ liệu:

```text
Account: admin-test
LastUsed:
```

Sai:

```text
Blank → Never used → Revoke
```

Đúng hơn:

```text
Blank
→ Determine source semantics
→ Is field supported?
→ Is collection complete?
→ Is account new?
→ Is observation window sufficient?
→ Classify Unknown if unsupported
```

---

# 17. Observation window

Suppose source chỉ giữ activity data 30 days nhưng review hỏi 90 days.

Nếu:

```text
No activity found
```

bạn không thể chứng minh:

```text
No activity for 90 days
```

Bạn chỉ chứng minh được:

```text
No observed activity in available 30-day window
```

Đây là difference giữa:

```text
absence of evidence
```

và:

```text
evidence of absence
```

Identity governance tốt phải hiểu distinction này.

---

# 18. Dormant User

## What

**Dormant account/user** thường là account không có qualifying activity trong một threshold theo policy.

Conceptually:

```text
Activity age >= inactivity threshold
```

Nhưng actual classification phải xét:

```text
Policy
Account type
Evidence availability
Timestamp reliability
Exceptions
```

## Common mistake

```text
Last login > 90 days
→ Dormant
→ Revoke
```

Quá đơn giản.

Phải hỏi:

```text
Human hay service?
New account hay old?
Emergency account không?
Last sign-in có phải appropriate activity metric không?
Evidence window complete không?
Account có approved exception không?
```

---

# 19. Stale Entitlement

Một **stale entitlement** là access assignment không còn phù hợp với current need hoặc lifecycle.

Ví dụ:

```text
User: EMP00421
Old Role: Network Engineer
New Role: Project Manager

Current Entitlements:
VPN
Email
Firewall Administrator
```

Firewall Administrator có thể trở thành stale entitlement.

Notice rằng user vẫn active và login hàng ngày.

Vì thế dormancy detection sẽ không tìm được vấn đề này.

---

# 20. Dormant Account vs Stale Entitlement

Hai concept khác nhau:

```text
Dormant account:
Account itself not appropriately active/recently used.

Stale entitlement:
Account may be active, but some access is no longer appropriate.
```

Ví dụ:

```text
Login yesterday
Admin role unused / no longer required
```

không phải dormant identity; có thể là stale privilege.

---

# 21. Orphaned Account

## Definition

**Orphaned account** là account không thể được map đáng tin cậy tới một valid owner/identity/purpose.

Ví dụ:

```text
System account:
adm_jdoe_old

IdentityMaster:
No matching employee

Owner:
Blank

Approval:
None
```

## Risk

Orphaned privileged accounts đặc biệt nguy hiểm vì accountability thấp.

Tuy nhiên:

```text
Orphaned ≠ safe to delete immediately
```

Có thể account phục vụ hidden application dependency.

### Safe workflow concept

```text
Detect
→ Verify identity mapping
→ Verify owner
→ Search approval/ticket
→ Determine dependency
→ Determine access/activity
→ Assign decision
```

---

# 22. Duplicate IDs

Duplicate là vấn đề vừa về data quality vừa về governance.

Các trường hợp:

```text
Same account exported twice
Same AccountID appears in multiple systems
Multiple identities map to one account
One identity has multiple privileged IDs
Duplicate source data
```

Một duplicate không phải lúc nào cũng là lỗi.

Ví dụ:

```text
AccountID = admin01
System = Firewall-A
```

và:

```text
AccountID = admin01
System = Switch-A
```

có thể hoàn toàn hợp lệ nếu `AccountID` chỉ unique trong từng system.

Do đó business key rất quan trọng.

---

# 23. Natural Key và Composite Key

## Natural Key

Một field intrinsically unique theo source, nếu thật sự guaranteed.

Ví dụ hypothetical:

```text
SystemUniqueAccountID
```

## Composite Key

Khi một field không unique toàn enterprise, kết hợp nhiều fields:

```text
System + Environment + AccountID
```

Ví dụ:

```text
AWS-PROD|prod|A001234
```

Syllabus yêu cầu join bằng documented account/system/ID composite keys, **không dùng display name alone**. Infrastructure_Operations_14_Da…

---

# 24. Vì sao Display Name là Key tệ?

Ví dụ:

```text
Nguyen Van An
```

có thể:

```text
Không unique
Có dấu/không dấu
Thứ tự họ tên khác
Đổi surname
Có typo
Có extra whitespace
```

Do đó:

```text
DisplayName = DisplayName
```

có thể tạo false match.

False match trong IAM nguy hiểm hơn obvious non-match vì nó có vẻ “thành công”.

---

# 25. Role Change

Role change là một trigger quan trọng cho access review.

Ví dụ:

```text
Old:
Database Administrator

New:
Business Analyst
```

Một mature workflow nên kiểm:

```text
Role changed?
Old privileges removed?
New privileges added?
Conflicting legacy privileges remain?
```

Một lỗi phổ biến là access chỉ tăng:

```text
Role A access
+
Role B access
+
Role C access
```

nhưng không bao giờ remove.

Đây là **privilege accumulation**.

---

# 26. Excessive Permissions

## What

Permission được xem là excessive nếu vượt business need hoặc role requirement theo policy.

Tư duy:

```text
Actual Permission
        vs
Required Permission
```

Điểm quan trọng:

```text
Privileged ≠ Automatically excessive
```

Một Production administrator có thể legitimately cần privileged role.

Câu hỏi đúng là:

```text
Privilege này có phù hợp current responsibility không?
```

---

# 27. Least Privilege

**Least privilege** nghĩa là principal chỉ có mức permission tối thiểu cần thiết để thực hiện authorized function.

Trong Day 13, least privilege không chỉ là design principle; nó trở thành review criterion.

Ví dụ:

```text
Required:
Read monitoring dashboard

Actual:
Full Administrator
```

có khả năng violation.

Nhưng reviewer cần evidence:

```text
Role requirement
Approval
Actual entitlement
Business context
```

---

# 28. Separation of Duties — SoD

## Definition

**Separation of Duties** nhằm ngăn một identity nắm đủ các capability xung đột để tự hoàn thành một high-risk workflow mà không cần independent control.

Ví dụ conceptual:

```text
Create privileged account
+
Approve privileged account
```

hoặc:

```text
Create payment
+
Approve payment
```

SoD không chỉ quan tâm permission từng cái riêng lẻ.

Hai permissions có thể individually legitimate, nhưng combination không acceptable.

---

# 29. SoD workflow

Một SoD analysis thường conceptually:

```text
Identity
↓
All entitlements
↓
Conflict matrix
↓
Conflict?
↓
Approved exception / compensating control?
```

Nếu conflict có approved exception, vẫn phải review:

```text
Owner
Reason
Expiry
Compensating control
```

Approved exception không có nghĩa permanent acceptance.

---

# 30. Service Accounts

Service account là một trong những identity type dễ bị xử lý sai nhất.

## What

Service account được workload/process/application sử dụng thay vì normal interactive human workflow.

Ví dụ:

```text
svc_backup
svc_monitoring
svc_batch
svc_integration
```

## Why special?

Human-account rules thường không áp dụng nguyên xi.

Ví dụ:

```text
No interactive login for 90 days
```

có thể hoàn toàn bình thường.

Thứ cần review là:

```text
Owner
Purpose
Workload dependency
Privilege
Credential lifecycle
Approval
Effective activity
```

---

# 31. Service account ownership

Không nên có:

```text
Owner = Service Account
```

vì account không thể tự chịu trách nhiệm.

Nên có accountable human/team role, ví dụ:

```text
Application Owner
Technical Owner
Service Owner
```

Nếu owner đã rời tổ chức, account có thể trở thành governance exception ngay cả khi service vẫn hoạt động.

---

# 32. Service Account Dormancy

Nếu service account không có activity, có thể là:

```text
Retired application
Failover-only function
Dormant legacy integration
Monitoring gap
Wrong activity metric
```

Vì vậy correct classification có thể là:

```text
Investigate
```

trước khi:

```text
Revoke
```

Đặc biệt Production service account không được disable chỉ vì spreadsheet nói inactive.

---

# 33. Break-Glass Accounts

## What

Break-glass account là emergency identity dùng khi normal access mechanism không khả dụng hoặc emergency escalation yêu cầu đặc biệt.

Đặc điểm thường thấy:

```text
Rarely used
Highly privileged
Specially controlled
Continuously monitored
```

Do đó nó có vẻ dormant theo normal rule.

Nhưng dormant không nhất thiết là lỗi.

## Review focus

Đối với break-glass account, câu hỏi phù hợp là:

```text
Account có được policy authorize không?
Owner/control owner có rõ không?
Credential protection có đúng không?
Monitoring có hoạt động không?
Usage có được review không?
Account còn functional không?
Exception có còn valid không?
```

Syllabus đưa break-glass account vào 90-day boundary fixtures để tránh automation naive. Infrastructure_Operations_14_Da…

---

# 34. New Accounts

Một account được tạo hôm qua có thể có:

```text
LastUsed = blank
```

Nếu logic là:

```text
Blank → Dormant
```

thì account mới sẽ bị classify sai.

Do đó rule thường cần xem:

```text
Creation date
Age
Grace period if policy defines one
Last-use support
```

Điểm quan trọng là **policy-specific behavior**. Không invent grace period nếu policy không có.

---

# 35. AWS IAM Evidence

Theo syllabus, Day 13 sử dụng **AWS IAM credential/access reports** cùng các nguồn khác để hỗ trợ review. Syllabus cũng lưu ý credential reports, permission/last-access information và CloudTrail có scope/limitations khác nhau; không được coi chúng là interchangeable evidence. Infrastructure_Operations_14_Da…

File syllabus tham chiếu riêng AWS IAM documentation cho credential reports và last-accessed information. Infrastructure_Operations_14_Da…

Điều cần hiểu về mặt theory là:

```text
Credential state evidence
≠
Entitlement evidence
≠
Activity evidence
≠
Approval evidence
```

Một credential report có thể giúp trả lời một số câu hỏi về credential/account state, nhưng không tự động chứng minh business justification.

Last-access evidence có thể giúp refinement nhưng không thay thế toàn bộ access review.

---

# 36. Identity Evidence vs Activity Evidence

Hãy xem một account:

```text
Account exists          → identity/account evidence
Role = Administrator    → entitlement evidence
Last used = 12 days ago → activity evidence
REQ001 approved         → authorization evidence
Employee active         → identity lifecycle evidence
```

Muốn decision mạnh, phải kết hợp nhiều loại evidence.

Một source thường không đủ.

---

# 37. Privileged-ID Management Exports

Syllabus yêu cầu dùng privileged-ID management exports cùng AWS IAM, PS/Gateway và AccessCheck trong review. Infrastructure_Operations_14_Da…

Các export này conceptually có thể cung cấp:

```text
Privileged account
Entitlement
Assignment
Access state
Usage/access history
Lifecycle information
```

Tuy nhiên syllabus đặc biệt cảnh báo rằng **PS/Gateway và AccessCheck là organization-specific product labels**, nên exact product, version và runbook phải được xác minh; không được invent product behavior. Infrastructure_Operations_14_Da…

Vì vậy Day 13 theory chỉ nên dạy governance pattern, không giả định menu/API cụ thể.

---

# 38. PS/Gateway và AccessCheck — điều phải hiểu

Ta có thể nói chắc về conceptual role:

```text
Request
↓
Approval
↓
Authorized access
↓
Use
↓
Expiry / Revocation
↓
Audit evidence
```

Nhưng không nên khẳng định:

```text
"PS/Gateway luôn có field X"
```

hoặc:

```text
"AccessCheck API endpoint là ..."
```

nếu chưa biết actual implementation.

Đây chính là enterprise discipline: không biến generic understanding thành fabricated product-specific instruction.

---

# 39. ServiceNow Approval Evidence

ServiceNow trong Day 13 đóng vai trò cung cấp governance evidence:

```text
Who requested?
Who approved?
What access?
For which identity?
For which system?
When?
For how long?
Why?
```

Một access record trong source system có thể legitimate về technical state nhưng unauthorized về governance nếu không tìm được supporting approval.

Ngược lại, một ServiceNow ticket nói approved không chứng minh access trên target system được provision chính xác.

Do đó reconciliation cần kiểm cả hai chiều:

```text
Approval → Actual access
```

và:

```text
Actual access → Approval
```

---

# 40. Access without Approval

Ví dụ:

```text
AWS IAM:
Account = admin123
Role = Admin

ServiceNow:
No corresponding approved request
```

Không nên ngay lập tức kết luận malicious.

Có thể do:

```text
Historical migration
Missing export
Old approval system
Incorrect mapping
Data retention gap
Unauthorized provisioning
```

Đúng classification ban đầu thường là:

```text
Missing Approval / Evidence Gap
```

sau đó investigate.

---

# 41. Approval without Access

Chiều ngược lại:

```text
ServiceNow:
Approved admin access

Actual IAM:
No entitlement
```

Có thể:

```text
Request never provisioned
Access expired
Provisioning failed
Request cancelled later
Export timing mismatch
```

Đây không nhất thiết security risk nhưng là governance/process discrepancy.

---

# 42. Cross-System Identity Reconciliation

Đây là trái tim kỹ thuật của Day 13.

Ta có nhiều nguồn:

```text
IdentityMaster
AWS IAM
Privileged Access
ServiceNow
Asset Inventory
Server Inventory
```

Mỗi source nói một phần câu chuyện.

Mục tiêu reconciliation là tạo relationship:

```text
Person / workload
      │
      ▼
Canonical Identity
      │
      ▼
System Account
      │
      ▼
Entitlement
      │
      ▼
Approval
      │
      ▼
Asset / Environment
```

---

# 43. Reconciliation không chỉ là VLOOKUP

Đây là distinction quan trọng.

Một lookup:

```text
Does AccountID exist?
```

chỉ là một primitive operation.

Reconciliation thực sự phải xử lý:

```text
Uniqueness
Normalization
Source quality
Duplicate detection
One-to-many relationship
Many-to-one relationship
Join precedence
Unknown values
Rejected records
```

Vì vậy Excel function hay Python join chỉ là implementation.

Control design mới là phần khó.

---

# 44. Canonical Identity

**Canonical identity** là representation chuẩn được chọn làm reference trong reconciliation.

Ví dụ:

```text
IdentityMaster.EmployeeID = canonical human identity key
```

Các source khác có thể chứa:

```text
Login name
Email
Cloud user name
Privileged ID
```

Sau đó mapping:

```text
EmployeeID
   ├─ CorporateLogin
   ├─ AWSUser
   └─ PrivilegedID
```

Không nhất thiết EmployeeID luôn là key tốt nhất; điều quan trọng là organization phải có documented identity model.

---

# 45. Raw Value và Normalized Value

Syllabus từ Day 12 đã nhấn mạnh preserve originals, và Day 13 áp dụng trực tiếp.

Ví dụ:

```text
Raw:
" 001234 "

Normalized:
"001234"
```

Không overwrite raw.

Tại sao?

Nếu reviewer hỏi:

```text
"Source thực sự ghi cái gì?"
```

bạn phải trả lời được.

---

# 46. Normalization Rules

Normalization phải explicit.

Ví dụ:

```text
Trim leading/trailing whitespace
Normalize case where identifier is case-insensitive
Normalize system names
Normalize approved date format
Preserve leading zeros
```

Không được có hidden transformations.

Ví dụ nguy hiểm:

```text
001234 → 1234
```

Nếu ID là text, hai giá trị có thể khác meaning.

---

# 47. Leading Zeros

Syllabus đặc biệt nhấn mạnh IDs phải được xem như text và preserve leading zeros trong automation training. Infrastructure_Operations_14_Da…

Trong Day 13:

```text
001234
```

không được vô tình biến thành:

```text
1234
```

Excel auto-formatting là source lỗi phổ biến.

Sai một ID có thể khiến account legitimate trở thành:

```text
Unmatched
```

hoặc nguy hiểm hơn:

```text
matched to wrong identity
```

---

# 48. One-to-One, One-to-Many và Many-to-Many

## One-to-One

```text
1 EmployeeID
→ 1 Account
```

đơn giản nhất.

## One-to-Many

```text
1 Administrator
→ 3 Accounts
```

có thể legitimate.

## Many-to-One

```text
Multiple identities
→ Shared account
```

thường cần đặc biệt review vì accountability có thể thấp.

## Many-to-Many

Thường rất nguy hiểm cho automation nếu join uncontrolled.

Syllabus acceptance contract yêu cầu không được có **silent many-to-many join**. Infrastructure_Operations_14_Da…

---

# 49. Silent Many-to-Many Join

Ví dụ:

Source A:

```text
ID       Row
1001     A1
1001     A2
```

Source B:

```text
ID       Row
1001     B1
1001     B2
```

Naive join tạo:

```text
A1-B1
A1-B2
A2-B1
A2-B2
```

từ 2 + 2 records thành 4 relationships.

Nếu analyst không phát hiện, counts inflate và reconciliation sai.

Do đó trước join phải kiểm uniqueness assumptions.

---

# 50. Counting Rule

Day 13 acceptance yêu cầu source totals reconcile được với categories theo documented counting rule. Infrastructure_Operations_14_Da…

Ví dụ:

```text
Source total = 1,000

Matched        = 940
Unmatched      = 30
Duplicate      = 20
Rejected       = 10
--------------------
Total          = 1,000
```

Nhưng điều này chỉ đúng nếu categories mutually exclusive.

Nếu duplicate cũng được tính trong unmatched thì:

```text
Matched + Unmatched + Duplicate
```

sẽ double-count.

Vì thế phải document counting semantics.

---

# 51. Matched

`Matched` không chỉ có nghĩa:

```text
Key found
```

Có thể cần nhiều level:

```text
Identity matched
Account matched
Entitlement matched
Approval matched
Asset matched
```

Một account có identity match nhưng approval miss.

Do đó mature model nên phân biệt match dimensions.

---

# 52. Unmatched

Unmatched có nhiều nguyên nhân:

```text
True orphan
Bad key
Missing source
Wrong normalization
New account
Historical account
Data extraction gap
```

Không nên classify mọi unmatched thành orphaned.

---

# 53. Rejected Record

Record bị `Rejected` khi không thể safely process.

Ví dụ:

```text
Missing required key
Malformed timestamp
Invalid record structure
Corrupt encoding
```

Đặc điểm quan trọng:

```text
Rejected ≠ silently skipped
```

Rejected records phải count và log reason.

---

# 54. Exception Classification

Day 13 cần exception taxonomy.

Một conceptual taxonomy tốt có thể chứa:

```text
Missing owner
Missing approval
Expired approval
Orphaned account
Duplicate identity
Duplicate account
Stale entitlement
Dormant user
Excessive privilege
Role mismatch
Unknown last use
Revoked-but-active
Invalid source record
```

Mỗi type cần deterministic criteria.

Nếu analyst tự gắn label theo intuition, automation không reproducible.

---

# 55. Unknown là trạng thái hợp lệ

Một trong những lessons quan trọng nhất của audit work:

```text
Unknown
```

không phải failure của analyst nếu source thực sự không đủ evidence.

Sai hơn là biến unknown thành false certainty.

Ví dụ:

```text
LastUse = blank
Source does not support field
```

đúng:

```text
Unknown
```

sai:

```text
Never used
```

---

# 56. Reviewer Decision Model

Syllabus yêu cầu mỗi exception đi tới một trong bốn action categories:

```text
Keep
Revoke
Modify
Investigate
```

và phải kèm rationale, owner, reviewer, due date, evidence và linked task. Infrastructure_Operations_14_Da…

Đây là governance layer nằm phía trên technical detection.

Automation có thể detect:

```text
Missing Approval
```

nhưng không nên tự quyết định:

```text
Revoke
```

nếu business policy yêu cầu human review.

---

# 57. Keep

`Keep` nghĩa là access tiếp tục được chấp nhận.

Nhưng `Keep` phải có rationale.

Weak:

```text
Keep — looks okay
```

Strong hơn:

```text
Keep — entitlement aligns with current Production Database Administrator role;
approved request REQ001234 valid through review period.
```

Reviewer phải chịu accountability cho decision.

---

# 58. Revoke

`Revoke` được dùng khi access không còn authorized/required và approved workflow cho phép removal.

Nhưng decision record chỉ là bước đầu.

Workflow phải phân biệt:

```text
Decision = Revoke
```

với:

```text
Technical access successfully revoked
```

Và sau implementation cần recheck effective access.

---

# 59. Modify

`Modify` phù hợp khi account vẫn cần nhưng current privilege không đúng.

Ví dụ:

```text
Current:
Administrator

Required:
ReadOnlyOperator
```

Decision:

```text
Modify
```

thay vì remove account hoàn toàn.

Đây thường là practical application của least privilege.

---

# 60. Investigate

`Investigate` là decision rất quan trọng.

Dùng khi evidence chưa đủ hoặc remediation trực tiếp quá rủi ro.

Ví dụ:

```text
Unknown service account owner
Blank last-use evidence
Unclear dependency
Conflicting approval records
```

Trong production operations, `Investigate` tốt hơn blind destructive action.

---

# 61. Decision Rationale

Rationale phải trả lời:

```text
Why was this decision made?
```

Không nên chỉ:

```text
"Per policy."
```

Tốt hơn:

```text
Account remains required for approved backup workload;
owner confirmed;
privilege aligns with service requirement;
last-use evidence is unavailable but service dependency is independently verified.
```

Rationale giúp reviewer tiếp theo hiểu logic.

---

# 62. Owner, Reviewer và Remediation Owner

Ba role này không nhất thiết giống nhau.

```text
Account Owner
Reviewer
Remediation Owner
```

Ví dụ:

```text
Account Owner:
Network Operations

Reviewer:
Infrastructure Security Manager

Remediation Owner:
IAM Operations
```

Nếu spreadsheet chỉ có `Owner`, trách nhiệm có thể mơ hồ.

---

# 63. Due Date

Exception không có due date rất dễ tồn tại mãi.

Do đó tracked action cần:

```text
Decision
Owner
Due date
Status
```

Nhất là:

```text
Investigate
Modify
Revoke
```

---

# 64. Evidence Linkage

Decision phải traceable về evidence.

Ví dụ:

```text
Decision D-001
   ↓
Exception E-004
   ↓
Reconciliation row 581
   ↓
IAM_Raw row 991
   ↓
Source file hash
   ↓
ServiceNow REQ001234
```

Syllabus acceptance yêu cầu **every decision traces to a source row and ticket**. Infrastructure_Operations_14_Da…

---

# 65. Vulnerability Assessment — vị trí của nó trong Day 13

Day 13 không chỉ là IAM.

Syllabus còn yêu cầu:

```text
Vulnerability assessment
CVE/CVSS awareness
Exploitability
Exposure
Asset criticality
Severity
Remediation priority
Exception acceptance
Compensating controls
Remediation tracking
```

Infrastructure_Operations_14_Da…

Mục tiêu không phải biến learner thành vulnerability researcher mà là biết xử lý finding theo operational governance.

---

# 66. Vulnerability là gì?

Một **vulnerability** là weakness có thể ảnh hưởng confidentiality, integrity hoặc availability khi bị exploited, tùy context.

Nguồn finding có thể đến từ:

```text
Scanner
Vendor advisory
Security assessment
Configuration review
Manual investigation
```

Day 13 dùng supplied findings.

Bạn phải biết chuyển raw finding thành remediation record có owner và evidence.

---

# 67. CVE

**CVE — Common Vulnerabilities and Exposures** cung cấp identifier cho publicly disclosed vulnerabilities khi applicable.

Conceptual format:

```text
CVE-YYYY-NNNNN
```

Không phải mọi finding đều có CVE.

Ví dụ:

```text
Weak local configuration
Unsupported internal setting
Missing owner
IAM exception
```

có thể không có CVE.

Do đó không được ép mọi finding thành CVE.

---

# 68. CVE ID không phải Severity

Một CVE number chỉ là identifier.

Ví dụ:

```text
CVE-20xx-xxxxx
```

tự nó không nói:

```text
Critical
High
Medium
Low
```

Severity phải đến từ scoring/advisory/context.

---

# 69. CVSS

**CVSS — Common Vulnerability Scoring System** là framework chuẩn hóa technical severity.

Trong Day 13 cần hiểu:

```text
CVSS score
```

là một input.

Không phải:

```text
Final remediation priority
```

Syllabus yêu cầu ghi scoring source/version thay vì assume mọi nguồn dùng cùng methodology. Infrastructure_Operations_14_Da…

---

# 70. Severity vs Business Risk

Đây là concept quan trọng.

Một vulnerability có technical severity cao nhưng asset:

```text
Offline lab
No sensitive data
No external exposure
```

có operational priority khác cùng vulnerability trên:

```text
Internet-facing Production authentication server
```

Do đó:

```text
Technical Severity
+
Exposure
+
Exploitability
+
Asset Criticality
+
Business Impact
```

mới giúp xác định remediation priority.

---

# 71. Exploitability

**Exploitability** hỏi:

```text
Lỗ hổng có thể bị khai thác thực tế trong context này đến mức nào?
```

Các yếu tố conceptually có thể gồm:

```text
Required access
Required interaction
Network reachability
Attack complexity
Available exploit evidence
Existing protections
```

Không nên chỉ thấy CVE rồi kết luận “easily exploitable”.

---

# 72. Exposure

Exposure hỏi:

```text
Asset có reachable/exposed theo cách cho phép attack path không?
```

Ví dụ:

```text
Internet-facing
Internal-only
Restricted management segment
Disconnected test asset
```

Exposure khác severity.

Critical CVE trên unreachable asset vẫn cần quản lý, nhưng immediate priority có thể khác.

---

# 73. Asset Criticality

Một vulnerability không tồn tại trong vacuum.

Asset có thể là:

```text
Tier-0 identity system
Production database
Employee workstation
Development VM
Disposable test server
```

Criticality phản ánh business importance và consequence nếu asset bị compromise hoặc unavailable.

---

# 74. Business Impact

Business impact hỏi:

```text
Nếu vulnerability bị khai thác thành công thì business bị ảnh hưởng gì?
```

Có thể ảnh hưởng:

```text
Service availability
Sensitive data
Privileged control
Regulatory obligations
Financial process
Customer-facing operations
```

Infrastructure Operations cần hiểu dependency, không chỉ scanner score.

---

# 75. Severity và Remediation Priority

Hãy coi:

```text
Severity = "How technically serious is the weakness?"
```

và:

```text
Priority = "How urgently should we act in this environment?"
```

Hai giá trị liên quan nhưng không identical.

Ví dụ:

```text
Finding A:
Severity = High
Production critical system
Internet exposed
No compensating control
→ Very high operational priority
```

Finding B:

```text
Severity = High
Isolated lab
No sensitive dependency
Strong isolation
→ Different priority
```

Không có nghĩa Finding B bỏ qua; chỉ khác scheduling/priority.

---

# 76. Vulnerability và Misconfiguration

Một security finding có thể là:

```text
CVE-based software vulnerability
Configuration weakness
IAM control exception
Operational incident
```

Syllabus trước đó yêu cầu classification rõ trước khi gán severity, và Day 13 nhấn mạnh IAM exceptions phải tách khỏi CVE/misconfiguration vulnerability classification. Infrastructure_Operations_14_Da…

Đây là rất quan trọng.

---

# 77. IAM Exception không phải CVE

Ví dụ:

```text
Dormant privileged account
```

đó là governance/security control exception.

Không cần invent:

```text
CVE = none
CVSS = 9.8
```

CVSS không phải universal score cho mọi security issue.

---

# 78. Exception Acceptance

Không phải vulnerability nào cũng có thể remediate ngay.

Có thể có:

```text
Vendor dependency
Application compatibility
Maintenance window
Business constraint
Temporary operational limitation
```

Organization có thể có formal **risk exception**.

Nhưng:

```text
Exception ≠ ignore
```

Exception cần lifecycle.

---

# 79. Risk Exception Lifecycle

Conceptually:

```text
Finding
↓
Risk assessed
↓
Exception requested
↓
Approver accepts
↓
Compensating control
↓
Expiry / Review date
↓
Reassessment
```

Một exception không có expiry/review mechanism dễ trở thành permanent technical debt.

---

# 80. Compensating Controls

**Compensating control** là control khác được dùng để giảm risk khi primary remediation chưa thể thực hiện.

Ví dụ conceptual:

```text
Network isolation
Tighter access restriction
Additional monitoring
Service exposure reduction
Temporary feature disablement
```

Một compensating control phải có:

```text
Purpose
Scope
Owner
Verification
Limitations
```

Không chỉ ghi:

```text
"Firewall protects it."
```

---

# 81. Remediation Tracking

Finding lifecycle:

```text
Detected
→ Validated
→ Classified
→ Assigned
→ Planned
→ Remediated
→ Retested
→ Closed
```

Một finding không được coi closed chỉ vì:

```text
"Patch installed"
```

Cần retest hoặc verification evidence phù hợp.

---

# 82. Vulnerability Owner

Có thể cần phân biệt:

```text
Asset owner
Remediation owner
Risk/exception owner
Reviewer
```

Ví dụ:

```text
Asset Owner:
Application Team

Remediation Owner:
Windows Operations

Risk Exception Owner:
Business Service Owner
```

Clear ownership giảm stalled findings.

---

# 83. Vulnerability Register

Conceptually mỗi row cần hỗ trợ:

```text
Finding ID
Affected asset
Affected version
CVE if applicable
Score source/version
Exposure
Business impact
Severity
Priority
Remediation owner
Exception owner
Status
Retest evidence
```

Syllabus explicitly yêu cầu các elements này. Infrastructure_Operations_14_Da…

---

# 84. Multi-Sheet Excel Architecture

Day 13 yêu cầu workbook với sheet structure rõ ràng:

```text
Control
IdentityMaster
IAM_Raw
Privileged_Raw
ServiceNow_Raw
Asset_Raw
Reconciliation
Exceptions
Decisions
Vulnerabilities
TestResults
RunLog
```

Infrastructure_Operations_14_Da…

Đây không phải chỉ để đẹp.

Mỗi sheet represent một control boundary.

---

# 85. Control Sheet

`Control` lưu execution/review parameters.

Conceptually:

```text
Review ID
As-of time
Timezone
Cutoff rule
Source versions
Automation version
Run mode
Reviewer
```

Tại sao cần?

Nếu cutoff hard-coded sâu trong formula hoặc VBA, reviewer khó biết run này thực sự dùng rule nào.

Control sheet biến hidden configuration thành explicit configuration.

---

# 86. IdentityMaster

`IdentityMaster` chứa canonical identities dùng cho reconciliation.

Nó nên represent truth source hoặc approved consolidated source cho:

```text
Identity ID
Current status
Current role
Owner attributes
```

Nhưng cần nhớ:

```text
IdentityMaster ≠ magically perfect
```

Nó cũng có thể stale.

Nếu mismatch, không tự động assume other source wrong.

---

# 87. IAM_Raw

`IAM_Raw` chứa untouched IAM import.

Principle:

```text
Raw = evidence
```

Không sửa raw để “làm sạch”.

Nếu cần normalization:

```text
Raw
→ Derived normalized fields
→ Reconciliation
```

---

# 88. Privileged_Raw

Tương tự nhưng cho privileged-access source.

Nó cho phép reviewer chứng minh:

```text
"This was the original privileged-access record received."
```

thay vì chỉ thấy transformed output.

---

# 89. ServiceNow_Raw

Chứa approval/workflow source.

Mục đích:

```text
Actual entitlement
↔
Approved entitlement
```

Nếu transform ticket fields, vẫn phải preserve original values.

---

# 90. Asset_Raw

Asset information giúp đặt account hoặc vulnerability vào operational context.

Ví dụ:

```text
Account → Server → Production → Criticality
```

hoặc:

```text
CVE → Host → Business Service
```

Nếu asset mapping sai, priority có thể sai.

---

# 91. Reconciliation Sheet

Đây là analytical layer.

Một row có thể conceptually tập hợp:

```text
Canonical Identity
Source Account
Entitlement
Owner Match
Approval Match
Asset Match
Activity Evidence
Classification
```

Quan trọng là row phải trace được về source.

---

# 92. Exceptions Sheet

Chứa detected deviations.

Nên tách detection khỏi human decision.

Ví dụ:

```text
Exception:
Missing Approval

Decision:
Investigate
```

Nếu merge hai concept thành một, automation dễ overwrite reviewer judgement.

---

# 93. Decisions Sheet

Đây là human-governance layer.

Không nên cho script tùy ý overwrite.

Syllabus acceptance nhấn mạnh rerun không được overwrite reviewer decisions. Infrastructure_Operations_14_Da…

Design tốt phải tách:

```text
Machine-derived fields
```

và:

```text
Reviewer-entered fields
```

---

# 94. Vulnerabilities Sheet

Dùng cho security findings, tách khỏi IAM Exceptions.

Đây là intentional separation.

Nếu trộn:

```text
Dormant admin account
CVE finding
Missing owner
Unpatched package
```

vào một generic “Risk” table, remediation workflow trở nên mơ hồ.

---

# 95. TestResults

Mỗi automation test cần record:

```text
Case ID
Requirement
Fixture version
Expected result
Actual result
Pass/Fail
Defect
Retest status
```

Syllabus common automation contract yêu cầu chính các loại thông tin này. Infrastructure_Operations_14_Da…

---

# 96. RunLog

`RunLog` là execution evidence.

Conceptually:

```text
Run ID
Timestamp
Automation version
Input set
Input hashes
Parameters
Output counts
Warnings
Errors
Status
```

Nếu output sai, RunLog hỗ trợ reproduce.

---

# 97. Audit Trail Preservation

Audit trail trả lời:

```text
How did this number get here?
```

Ví dụ report nói:

```text
37 stale privileged accounts
```

Reviewer phải trace được:

```text
37
↓
Exceptions rows
↓
Reconciliation logic
↓
Source records
↓
Source files
↓
Collection metadata
```

Nếu không trace, result là claim, chưa phải defensible audit evidence.

---

# 98. Evidence Manifest

Common evidence contract của syllabus yêu cầu mỗi evidence item có các thành phần như evidence ID, source system/product/version, environment/asset/account identifier, ticket reference, collector, timestamp/timezone, requested interval, exact filters/commands, export format/encoding, file size, record count, SHA-256, classification và transformation lineage. Infrastructure_Operations_14_Da…

Day 13 phải reuse mindset này.

---

# 99. Raw Evidence vs Working Copy

Pattern đúng:

```text
Source Export
     │
     ├── Original Raw — immutable
     │
     └── Working Copy
            ↓
        Normalize
            ↓
        Reconcile
            ↓
         Report
```

Không:

```text
Original Export
→ Open
→ Edit
→ Save
→ Pretend still original
```

---

# 100. SHA-256

## What

SHA-256 tạo digest từ file content.

Dùng để detect change so với recorded baseline.

Ví dụ:

```text
Original:
Hash = abc123...

Later:
Hash = xyz789...
```

→ file content khác.

## Nó chứng minh điều gì?

Nó có thể giúp chứng minh:

```text
Current bytes match previously hashed bytes
```

## Nó không chứng minh gì?

Syllabus nhấn mạnh hash không tự chứng minh:

```text
Source authentic
Export complete
Query correct
Data truthful
Time window complete
```

Infrastructure_Operations_14_Da…

Đây là distinction rất quan trọng.

---

# 101. File Count và Record Count

Hash chỉ detect byte-level modification.

Bạn cũng cần counts để phát hiện logical completeness issues.

Ví dụ:

```text
Source Export rows = 10,000
Processed rows = 9,980
Rejected = 20
```

tốt vì reconciliation explainable.

Nếu:

```text
Processed = 9,970
Rejected = 20
Missing = 10 unexplained
```

có evidence gap.

---

# 102. Evidence Gap

Một **evidence gap** xảy ra khi bạn không thể thu đủ evidence cần thiết.

Ví dụ:

```text
Retention expired
Permission denied
Source disconnected
Unsupported field
Export truncated
```

Correct response:

```text
Record gap
Describe impact
Escalate/reviewer decision
```

không phải:

```text
Assume missing data means no issue
```

Syllabus yêu cầu limitations/errors phải được ghi rõ. Infrastructure_Operations_14_Da…

---

# 103. Version Control

Automation output phải gắn với code version.

Ví dụ:

```text
reconcile.py v1.4.2
```

hoặc commit/version identifier tương đương.

Nếu report được tạo bởi version khác nhau nhưng không biết version nào, khó reproduce.

Version control hỗ trợ:

```text
Change traceability
Regression investigation
Rollback
Reproducibility
```

---

# 104. Reproducibility

**Reproducibility** nghĩa là với cùng controlled input + same rule + same code version, ta có thể tạo lại same business result.

Syllabus acceptance:

> repeat runs phải tạo identical business results sau khi loại trừ run timestamp/IDs. Infrastructure_Operations_14_Da…

Ví dụ:

```text
Run 1:
Dormant = 12
Missing Approval = 7

Run 2:
Dormant = 12
Missing Approval = 7
```

Expected.

Nếu run 2 cho:

```text
Dormant = 13
```

dù input không đổi, cần investigate nondeterminism.

---

# 105. Determinism

Một automation deterministic phải tạo cùng logical output cho cùng input.

Non-deterministic behavior có thể đến từ:

```text
Current system time used implicitly
Random ordering
Unstable duplicate resolution
External changing lookup
Locale-specific parsing
```

Ví dụ nếu code dùng:

```text
now()
```

thay vì declared `AsOfTime`, cùng input chạy ngày mai có thể classify khác.

Đó là lý do Day 13 dùng explicit as-of.

---

# 106. Automation Test Design

Day 13 yêu cầu formal test design, không phải “chạy thử vài file”.

Các test group của syllabus bao phủ identity matching, file/encoding, process failure, IAM boundaries, API/network/TLS, password generation, Outlook/report và repeatability/regression; riêng Day 13 tập trung mạnh vào identity matching, IAM boundaries và repeatability. Infrastructure_Operations_14_Da…

Core principle:

```text
Test expected behavior
AND
test failure behavior
```

---

# 107. Test Case Anatomy

Mỗi test phải trả lời:

```text
Case ID?
Requirement?
Fixture?
Expected?
Actual?
Pass/Fail?
Defect?
Retest?
```

Nếu chỉ ghi:

```text
"Tested okay"
```

thì không reproducible.

---

# 108. Positive Test

Positive test dùng valid input và expected successful behavior.

Ví dụ:

```text
Identity exists
Account exists
Owner valid
Approval valid
Role aligned
```

Expected:

```text
Matched
No Exception
```

Positive test chứng minh happy path.

---

# 109. Negative Test

Negative test kiểm invalid/undesired condition.

Ví dụ:

```text
Owner missing
```

Expected:

```text
Exception = MissingOwner
```

Automation không được:

```text
Crash
```

hoặc:

```text
Silently mark clean
```

---

# 110. Boundary Test

Rất quan trọng với Day 13.

Ví dụ:

```text
89 days 23:59:59
90 days exactly
90 days + 00:00:01
```

và:

```text
UTC boundary
Local-time boundary
```

Boundary test tìm bug mà normal test thường bỏ qua.

---

# 111. New Account Test

Fixture:

```text
Created yesterday
LastUsed blank
```

Expected result phải theo policy.

Không được automatically dormant chỉ vì blank.

Syllabus yêu cầu new-account fixture trong Day 13. Infrastructure_Operations_14_Da…

---

# 112. Service Account Test

Fixture:

```text
Service account
No interactive sign-in
Application owner valid
Dependency active
```

Automation không được apply human-user dormancy rule naïvely.

---

# 113. Break-Glass Test

Fixture:

```text
Emergency admin
Rarely used
Policy-approved
```

Expected classification phải reflect special account type.

---

# 114. Missing Owner Test

Fixture:

```text
Account valid
Activity recent
Owner blank
```

Expected:

```text
Missing Owner
```

không:

```text
No Exception because account used recently
```

Activity và ownership là independent controls.

---

# 115. Missing Approval Test

Fixture:

```text
Current entitlement exists
No matching approval
```

Expected:

```text
Missing Approval
```

phải traceable.

---

# 116. Revoked-but-Active Test

Fixture:

```text
ServiceNow:
Revoked

Target:
Still Active
```

Expected:

```text
RevokedButActive exception
```

đây là cross-system control test.

---

# 117. Duplicate Test

Input:

```text
same composite key twice
```

Automation phải:

```text
Detect duplicate
```

trước khi uncontrolled join.

Không được arbitrary select first row mà không báo.

---

# 118. Inconsistent Dataset

Ví dụ:

```text
IdentityMaster says:
User disabled

IAM says:
Account active

ServiceNow says:
Access revoked
```

Correct behavior:

```text
Flag inconsistency
```

không chọn một source ngẫu nhiên làm truth trừ khi documented source precedence tồn tại.

---

# 119. Malformed Data Test

Ví dụ:

```text
Invalid timestamp
Missing required header
Broken record
```

Correct behavior:

```text
Reject/quarantine
Log reason
Count rejected row
```

không silent skip.

---

# 120. Incomplete Data Test

Ví dụ:

```text
Missing owner column
```

Có hai khả năng tùy input contract:

```text
Fail entire run
```

hoặc:

```text
Continue with controlled warning
```

Nhưng behavior phải explicit và tested.

---

# 121. Regression Test

**Regression testing** đảm bảo change mới không làm hỏng behavior cũ.

Ví dụ developer sửa cutoff logic.

Phải retest:

```text
Boundary
Duplicates
Missing owner
Approval matching
Service accounts
```

không chỉ test cutoff.

---

# 122. Golden Dataset

Syllabus yêu cầu expected-output dataset hoặc independently reviewed assertions thay vì chỉ so program với chính nó. Infrastructure_Operations_14_Da…

Concept:

```text
Known Input
+
Known Expected Output
```

Automation output được compare với independently established truth.

Nếu dùng:

```text
Version A program
```

để tạo expected output rồi:

```text
Version B program
```

so với output đó mà không independently validate, cùng một logic bug có thể tồn tại trong cả hai.

---

# 123. Result Comparison

Không nên chỉ compare file binary vì timestamp/run IDs có thể khác.

Cần phân biệt:

```text
Business result
```

và:

```text
Execution metadata
```

Ví dụ:

```text
Business output identical
RunID different
Timestamp different
```

vẫn pass reproducibility.

---

# 124. Idempotency

## Definition

Một operation idempotent có thể chạy lại mà không tạo unintended duplicate effect.

Trong Day 13:

```text
Same exception
```

không được tạo:

```text
Task #1
Task #2
Task #3
```

mỗi lần rerun.

Syllabus acceptance đặc biệt yêu cầu:

```text
no duplicate tasks
no overwritten reviewer decisions
```

Infrastructure_Operations_14_Da…

---

# 125. Idempotency vs Repeatability

Hai khái niệm khác nhau.

**Repeatability/Reproducibility** hỏi:

```text
Kết quả business có giống nhau không?
```

**Idempotency** hỏi:

```text
Chạy lại có tạo extra side effect không?
```

Một script có thể repeatable về report nhưng không idempotent về ticket creation.

Ví dụ:

```text
Run 1:
37 exceptions + 37 tickets

Run 2:
37 exceptions + 37 new duplicate tickets
```

Business detection repeatable, side effect không idempotent.

---

# 126. Dry-Run

**Dry-run** là mode simulate planned actions mà không mutate target systems.

Ví dụ:

```text
Would revoke account X
Would reduce role Y
Would create task Z
```

Nhưng không thực hiện.

Dry-run đặc biệt quan trọng với IAM vì remediation có thể lock out admin hoặc break application.

Nó hỗ trợ:

```text
Preview
Review
Approval
Risk reduction
```

---

# 127. Dry-Run không phải Verification

Một dry-run tốt chứng minh:

```text
Automation intends to do X
```

Nó không chứng minh:

```text
X will definitely succeed on target
```

Sau authorized real execution vẫn cần verification.

---

# 128. Error Logging

Automation operational-grade phải log failure đủ để troubleshoot mà không leak secrets.

Một useful error entry có thể gồm:

```text
Run ID
Case / record ID
Stage
Error category
Sanitized message
Timestamp
```

Không nên log:

```text
Password
Token
Private key
Secret header
```

---

# 129. Fail-Open vs Fail-Closed

Đây là supplementary operational concept nhưng rất quan trọng.

Ví dụ automation gặp:

```text
Missing Approval source
```

Nếu vẫn classify:

```text
"Approved"
```

đó là dangerous fail-open behavior.

An toàn hơn có thể là:

```text
Unknown / Unable to Verify
```

và escalate.

Đây không phải lúc nào cũng “abort everything”; behavior phải phù hợp risk và documented acceptance criteria.

---

# 130. Defect Tracking

Failed test không được biến thành passed chỉ vì script hoàn thành.

Syllabus nói rõ failed case không thể relabel `Passed` đơn giản vì program completed. Infrastructure_Operations_14_Da…

Một defect cần lifecycle:

```text
Detected
→ Logged
→ Assigned
→ Fixed
→ Retested
→ Closed
```

---

# 131. Operational Acceptance Criteria

Một automation Day 13 không đạt chỉ vì:

```text
Exit code = 0
```

Nó phải thỏa:

```text
Correct classification
Correct counts
Correct boundary behavior
No silent data loss
No duplicate side effects
No overwritten decisions
Traceability preserved
Errors observable
Inputs preserved
```

Đây là khác biệt giữa developer smoke test và operational acceptance.

---

# 132. Verification Model cho Day 13 Theory

Có thể chia verification thành năm layer.

| Layer                       | Câu hỏi                                                                |
| --------------------------- | ---------------------------------------------------------------------- |
| Source verification         | Dữ liệu lấy từ đúng source, đúng period và đủ population chưa?         |
| Transformation verification | Normalization/join có đúng và reproducible không?                      |
| Control verification        | Exception logic có đúng policy không?                                  |
| Decision verification       | Reviewer decision có rationale và evidence không?                      |
| Remediation verification    | Effective access/finding state có thực sự thay đổi như intended không? |

Một review yếu thường chỉ verify layer 2.

Một review enterprise phải nhìn đủ cả năm.

---

# 133. Troubleshooting Order

Khi reconciliation result bất thường, không nên sửa formula ngay.

Một operational troubleshooting flow ít rủi ro hơn là:

```text
1. Verify scope and policy
      ↓
2. Verify source completeness
      ↓
3. Verify source schema
      ↓
4. Verify key normalization
      ↓
5. Verify uniqueness assumptions
      ↓
6. Verify join logic
      ↓
7. Verify classification rules
      ↓
8. Verify reviewer/action mapping
```

Nếu source thiếu, sửa lookup không giải quyết root cause.

---

# 134. Troubleshooting Example — Unmatched tăng đột ngột

Giả sử review trước:

```text
Unmatched = 20
```

review hiện tại:

```text
Unmatched = 800
```

Không nên lập tức kết luận 780 orphaned accounts mới xuất hiện.

Hãy kiểm:

```text
Source field renamed?
Leading zeros lost?
Case normalization changed?
IdentityMaster incomplete?
Wrong environment selected?
Export truncated?
Composite key version changed?
```

Data-engineering fault có thể masquerade thành security finding.

---

# 135. Troubleshooting Example — Dormant tăng mạnh

Nếu dormant count từ 5 lên 500, kiểm:

```text
As-of time
Timezone
Cutoff comparator
Last-use source availability
Parsing
Blank handling
Source retention
```

Một empty activity field có thể do source extraction failure, không phải 495 dormant identities mới.

---

# 136. Troubleshooting Example — Duplicate explosion

Nếu duplicates tăng mạnh:

```text
Check source duplicate
Check join cardinality
Check whether key lost system/environment component
Check trimming/case normalization
Check duplicated import
```

Một poorly designed key có thể collapse nhiều identities thành một.

---

# 137. Security Implications của Workbook

Workbook Day 13 có thể chứa sensitive operational data:

```text
Account IDs
Privileged roles
System names
Approval references
Potential vulnerabilities
```

Do đó supplemental operational controls nên gồm:

```text
Access restriction
Controlled storage
Minimal distribution
No real passwords/tokens
Protected evidence repository
```

Không nên email unrestricted workbook chứa privileged inventory tùy tiện.

---

# 138. Data Minimization

Review scope nói review cần data gì.

Đừng export everything chỉ vì có thể.

Ví dụ nếu review không cần secret/token values, chúng không nên xuất hiện.

Day 13 không bao giờ yêu cầu credential plaintext.

Program-wide instructions cũng yêu cầu secrets và credentials không xuất hiện trong evidence.

---

# 139. Least Privilege cho Reviewer

Reviewer cần đủ quyền đọc evidence.

Không có nghĩa cần quyền mutate Production.

Một pattern tốt:

```text
Read-only evidence access
+
Separate approved remediation workflow
```

Điều này hỗ trợ SoD.

Người review không nhất thiết nên có khả năng tự phê duyệt và tự revoke mọi thứ.

---

# 140. Evidence Collection Security

Evidence itself có thể sensitive.

Một audit package có thể reveal:

```text
Privileged accounts
System topology
Vulnerable hosts
Internal IDs
```

Do đó evidence cần:

```text
Access classification
Storage control
Retention
Controlled sharing
```

Program evidence contract cũng yêu cầu access/retention classification. Infrastructure_Operations_14_Da…

---

# 141. L1/L2 Operational Boundary

Đây là kiến thức bổ sung operational để áp dụng syllabus.

### L1 thường có thể

Thực hiện read-only collection, verify source presence, kiểm basic account status, phát hiện obvious mismatch, chạy approved reconciliation tool, thu evidence và escalate.

### L2 thường có thể

Phân tích complex reconciliation, investigate duplicates/orphans, validate entitlement/approval relationships, review automation defects, thực hiện approved sandbox remediation hoặc standard approved production procedure theo runbook.

### Phải escalate khi

Có unknown ownership trên critical privileged account, break-glass anomaly, service-account dependency chưa rõ, possible unauthorized privileged access, mass revocation risk, evidence gap nghiêm trọng, vulnerability ảnh hưởng critical system, automation defect có khả năng đưa ra hàng loạt incorrect decisions, hoặc production remediation nằm ngoài approved change.

Exact L1/L2 boundary luôn phụ thuộc organization/runbook.

---

# 142. Khi nào tuyệt đối không tự xử lý?

Ví dụ:

```text
Unknown Production service account
+
No owner
+
No recent interactive login
```

Không nên:

```text
Disable immediately
```

Có thể nó chạy payment batch mỗi cuối tháng.

Đúng behavior:

```text
Investigate dependency
Collect evidence
Establish owner
Escalate if necessary
```

---

# 143. Mass Remediation Risk

Automation có thể tìm:

```text
750 stale accounts
```

Không nên assume:

```text
750 → bulk disable
```

Trước mass action cần:

```text
Validate policy
Validate source
Sample results
Check false positives
Approval
Dry-run
Backout/restore plan where applicable
Phased execution
Post-check
```

Day 13 lab chỉ cho phép actual remediation trên authorized sandbox. Infrastructure_Operations_14_Da…

---

# 144. Reviewer Independence

Supplemental governance concept:

Nếu cùng một person:

```text
Requests access
Approves access
Implements access
Reviews access
Closes exception
```

thì SoD rất yếu.

Enterprise governance thường phân chia accountability.

Không nhất thiết mọi organization giống nhau, nhưng reviewer phải nhận biết conflict này.

---

# 145. Auditability

Một process được coi là audit-friendly nếu reviewer độc lập có thể trả lời:

```text
What was reviewed?
When?
Using which data?
Which policy?
Which code?
What exceptions were found?
Who decided?
Why?
What changed?
Was the change verified?
```

Day 13 thực chất dạy cách xây process có thể trả lời từng câu này.

---

# 146. Defensible Decision

Một **defensible decision** không có nghĩa “chắc chắn tuyệt đối”.

Nó có nghĩa decision dựa trên evidence đủ phù hợp, methodology rõ và uncertainty được ghi nhận.

Ví dụ tốt:

```text
Investigate — last-use evidence unavailable because source does not retain required 90-day window; account is privileged; owner confirmed but current necessity not yet validated.
```

Tốt hơn nhiều:

```text
Revoke — no login.
```

---

# 147. Unknown Production Facts Must Remain Unknown

Program-wide completion rubric có một nguyên tắc rất quan trọng:

> **Unknown production facts must remain unknown.** Infrastructure_Operations_14_Da…

Đây là mindset cốt lõi cho infrastructure operations.

Không bịa:

```text
"Probably unused."
```

Không biến:

```text
"No data"
```

thành:

```text
"No activity"
```

---

# 148. Evidence Completeness không bằng Evidence Correctness

Có hai dimension.

### Completeness

Bạn có đủ data cần không?

### Correctness

Data/logic có đúng không?

Ví dụ:

```text
Complete export
+
Wrong cutoff logic
```

→ conclusion sai.

Hoặc:

```text
Correct logic
+
Incomplete population
```

→ conclusion cũng sai.

Phải verify cả hai.

---

# 149. Automation Correctness không bằng Governance Correctness

Script có thể:

```text
100% correct theo formula
```

nhưng formula dựa vào wrong policy.

Ví dụ code đúng:

```text
LastLogin <= 90 days
```

nhưng organization policy threshold là 120 days.

Vậy software correct nhưng governance outcome wrong.

Do đó:

```text
Requirement validation
```

phải precede coding.

---

# 150. Data Quality vs Security Finding

Một unmatched account có thể là:

```text
True orphaned account
```

hoặc:

```text
IdentityMaster missing record
```

Một blank approval có thể là:

```text
Unauthorized access
```

hoặc:

```text
ServiceNow export incomplete
```

Analyst phải phân biệt:

```text
Security/control exception
```

và:

```text
Data-quality exception
```

Nếu không, remediation sẽ sai.

---

# 151. Correlation vs Causation

Nếu vulnerability finding và incident xảy ra cùng asset, không tự động kết luận vulnerability gây incident.

Tương tự:

```text
Account unused
+
Employee resigned
```

có vẻ liên quan, nhưng vẫn cần lifecycle evidence.

Infrastructure analyst phải tránh overclaim.

---

# 152. Source of Truth vs Source of Evidence

Một system có thể là authoritative cho một attribute nhưng không cho attribute khác.

Ví dụ conceptual:

```text
IdentityMaster:
authoritative for employee status

IAM:
authoritative for actual entitlement

ServiceNow:
authoritative for approval record
```

Không có một source duy nhất authoritative cho mọi question.

Cross-system reconciliation tồn tại chính vì lý do đó.

---

# 153. Source Precedence

Nếu sources conflict, cần rule:

```text
Which source wins for which attribute?
```

Ví dụ:

```text
Employee status → IdentityMaster
Actual access → IAM target
Approval → ServiceNow
```

Nhưng source precedence phải documented, không hidden trong code.

---

# 154. Effective Access

**Effective access** là permission thực tế identity có thể sử dụng sau khi xét actual policy/group/role relationships.

Đây mới là thứ remediation cuối cùng cần verify.

Ticket:

```text
Revoked
```

chưa đủ.

Expected:

```text
Effective access no longer granted
```

---

# 155. Entitlement Drift

Entitlement drift xảy ra khi actual access diverges khỏi approved/current-state access.

Ví dụ:

```text
Approved = ReadOnly
Actual = Administrator
```

hoặc:

```text
Approved = Revoked
Actual = Still active
```

Day 13 reconciliation là mechanism phát hiện drift.

---

# 156. Identity Drift

Identity drift có thể là:

```text
Employee changed role
Account ownership still old
Account status inconsistent
Manager changed
```

IdentityMaster vs access source reconciliation giúp tìm các mismatch này.

---

# 157. Asset Drift

Asset inventory cũng có thể stale.

Ví dụ:

```text
PalletControl says server Production
CMDB says Decommissioned
```

Nếu vulnerability priority dựa trên wrong asset classification, decision sai.

Do đó asset data cũng cần freshness awareness.

Syllabus lưu ý PalletControl implementation phải được xác minh theo actual organization deployment; generic inventory exercise không được coi là product proficiency. Infrastructure_Operations_14_Da…

---

# 158. Product-Specific Caveat: PalletControl, PS/Gateway, AccessCheck

Syllabus explicitly nói những tên này có thể organization-specific; instructor phải xác nhận exact vendor/product/version/runbook trước khi đưa ra product-specific steps. Infrastructure_Operations_14_Da…

Vì thế trong Day 13 theory, điều chúng ta học chắc chắn là governance relationship:

```text
Identity
Access
Approval
Asset
History
Evidence
```

Còn exact screen, field, API hay menu chỉ được dạy sau khi biết actual deployment.

---

# 159. Common Failure Modes của Day 13

Có thể nhóm failure modes thành năm domain.

| Domain         | Failure điển hình                                                           |
| -------------- | --------------------------------------------------------------------------- |
| Policy         | Wrong cutoff, unclear inactivity rule, wrong scope                          |
| Data           | Missing rows, stale source, lost leading zeros, malformed dates             |
| Reconciliation | Bad key, false match, many-to-many join, duplicate inflation                |
| Governance     | Missing owner, missing approval, wrong decision, unverified revoke          |
| Automation     | Silent error, nondeterminism, duplicated side effect, overwritten decisions |

Troubleshooting hiệu quả phải xác định failure thuộc layer nào trước.

---

# 160. Symptom vs Root Cause

Ví dụ symptom:

```text
500 orphaned accounts
```

Possible root causes:

```text
IdentityMaster export truncated
```

hoặc:

```text
Composite key changed
```

hoặc thật sự:

```text
Offboarding failure
```

Không được xử lý symptom như root cause.

---

# 161. Operational Evidence cần thu thập

Trong theory, hãy nhớ evidence package Day 13 cuối cùng phải hỗ trợ ít nhất:

```text
Review definition
Source inventory
Source hashes/counts
Normalization rules
Composite-key rules
Reconciliation result
Exception list
Reviewer decisions
Vulnerability classifications
Automation version
Test results
Run logs
Ticket linkage
```

Đây là kết quả trực tiếp của syllabus Day 13 và common evidence contract. Infrastructure_Operations_14_Da… Infrastructure_Operations_14_Da…

---

# 162. Mental Model hoàn chỉnh của 90-Day Review

Bạn nên ghi nhớ pipeline sau:

```text
POLICY
  │
  ▼
Define Review Scope
  │
  ▼
Declare As-of / Cutoff / Timezone
  │
  ▼
Collect Raw Sources
  │
  ▼
Hash / Count / Preserve
  │
  ▼
Normalize Derived Data
  │
  ▼
Build Composite Keys
  │
  ▼
Reconcile Identities / Accounts / Entitlements
  │
  ▼
Match Approval / Owner / Asset
  │
  ▼
Classify Exceptions
  │
  ▼
Reviewer Decision
  │
  ├── Keep
  ├── Revoke
  ├── Modify
  └── Investigate
  │
  ▼
Authorized Remediation
  │
  ▼
Effective-Access Verification
  │
  ▼
Evidence / Audit Trail
```

Đây chính là logic core của Day 13.

---

# 163. Mental Model hoàn chỉnh của Vulnerability Workflow

```text
RAW FINDING
    │
    ▼
Validate Finding
    │
    ▼
Identify Asset / Version
    │
    ▼
CVE / Scoring Source if applicable
    │
    ▼
Evaluate Exploitability
    │
    ▼
Evaluate Exposure
    │
    ▼
Asset Criticality
    │
    ▼
Business Impact
    │
    ▼
Severity + Remediation Priority
    │
    ▼
Owner / Due Date
    │
    ├── Remediate
    │
    └── Approved Exception + Compensating Control
    │
    ▼
Retest
    │
    ▼
Close / Continue Tracking
```

---

# 164. Mental Model hoàn chỉnh của Automation Assurance

```text
REQUIREMENT
   │
   ▼
Documented Rule
   │
   ▼
Automation Logic
   │
   ▼
Known Test Fixture
   │
   ├── Positive
   ├── Negative
   ├── Boundary
   ├── Duplicate
   ├── Malformed
   ├── Incomplete
   └── Inconsistent
   │
   ▼
Expected vs Actual
   │
   ▼
Pass / Defect
   │
   ▼
Regression Test
   │
   ▼
Repeatability Test
   │
   ▼
Idempotency Test
   │
   ▼
Operational Acceptance
```

---

# 165. Những anti-pattern quan trọng phải tránh

Một anti-pattern là:

```text
"Last login blank nên revoke."
```

Một anti-pattern khác:

```text
"Display name giống nhau nên cùng identity."
```

Hoặc:

```text
"Ticket ghi revoked nên access đã mất."
```

Hoặc:

```text
"CVSS cao nhất nên luôn xử lý đầu tiên."
```

Hoặc:

```text
"Hash match nên evidence chắc chắn complete."
```

Hoặc:

```text
"Script chạy không crash nên test passed."
```

Hoặc:

```text
"Workbook output đẹp nên reconciliation đúng."
```

Toàn bộ Day 13 được thiết kế để loại bỏ kiểu suy luận này.

---

# 166. Day 13 Theory — Operational reasoning scenario

Giả sử bạn thấy:

```text
Account:
svc-prod-report

Status:
Active

Owner:
Blank

Last sign-in:
Blank

Privilege:
Database Read

ServiceNow approval:
Found, approved 2 years ago

Application inventory:
Reporting service still active
```

Người mới có thể nói:

```text
Blank owner + blank login → Dormant → Revoke
```

Infrastructure Operations reasoning tốt hơn:

```text
1. Đây là service account, không dùng normal interactive-login logic.
2. Missing owner là governance exception.
3. Approval rất cũ cần kiểm validity/current requirement.
4. Application còn active cho thấy dependency có thể tồn tại.
5. Last sign-in blank không chứng minh inactivity.
6. Correct initial decision likely Investigate.
7. Need establish owner and verify workload use.
8. Production revoke requires approved change and dependency validation.
```

Đây chính là level reasoning mà Day 13 hướng tới.

---

# 167. Scenario thứ hai — Revoked-but-Active

```text
Employee:
Left company 20 days ago

IdentityMaster:
Disabled

AWS access:
Privileged role still assigned

ServiceNow:
Termination access revoke completed

Last activity:
5 days ago
```

Đây là serious discrepancy.

Phân tích:

```text
Identity lifecycle says disabled.
Governance workflow says revoke complete.
Effective entitlement says access remains.
Activity suggests continued use.
```

Lúc này không chỉ là stale data.

Cần:

```text
Preserve evidence
Validate identity/account mapping
Escalate according to security/incident procedure
Avoid destroying evidence
Revoke only through authorized emergency/standard process
```

Đây có thể vượt routine L1 review.

---

# 168. Scenario thứ ba — Vulnerability Priority

```text
Asset A:
CVSS high
Internal development
No production data

Asset B:
CVSS medium
Internet-facing authentication service
Critical business dependency
```

Sai:

```text
Always fix A before B because CVSS higher
```

Đúng hơn:

```text
Evaluate both technical severity and operational/business risk.
```

Asset B có thể có higher remediation priority.

---

# 169. Scenario thứ tư — Wrong Join

```text
IdentityMaster:
00123

IAM export after Excel conversion:
123
```

Lookup fails.

Analyst thấy:

```text
Unmatched IAM Account
```

và classify:

```text
Orphaned
```

Root cause thật:

```text
Leading zero lost during data handling.
```

Đây là lý do data integrity và IAM governance không tách rời nhau.

---

# 170. Scenario thứ năm — Repeat Run

Run 1:

```text
47 exceptions
47 remediation tasks
```

Run 2 cùng input:

```text
47 exceptions
94 total remediation tasks
```

Detection reproducible nhưng workflow không idempotent.

Automation chưa đạt operational acceptance.

---

# 171. Tiêu chuẩn hiểu Module Lý thuyết

Sau khi học xong phần này, bạn phải có thể nhìn một IAM review và tự hỏi đúng các câu:

| Area             | Câu hỏi bạn phải biết đặt ra                   |
| ---------------- | ---------------------------------------------- |
| Scope            | Population có đúng và complete không?          |
| Time             | As-of, cutoff, timezone và comparator là gì?   |
| Identity         | Account thuộc identity nào?                    |
| Ownership        | Ai chịu trách nhiệm?                           |
| Entitlement      | User có quyền gì thực tế?                      |
| Approval         | Quyền đó có authorization còn hiệu lực không?  |
| Activity         | Evidence nói gì và không nói gì?               |
| Special accounts | Có service/break-glass semantics không?        |
| Reconciliation   | Key nào được dùng, uniqueness ra sao?          |
| Decision         | Keep/Revoke/Modify/Investigate dựa trên gì?    |
| Vulnerability    | Severity khác priority như thế nào?            |
| Evidence         | Có trace ngược được source không?              |
| Automation       | Có boundary/regression/idempotency test không? |

---

# 172. Day 13 Theory Cheat Sheet

| Concept           | Điểm phải nhớ                                                  |
| ----------------- | -------------------------------------------------------------- |
| 90-day review     | Review cadence không đồng nghĩa inactivity threshold           |
| As-of             | Reference timestamp cố định của review                         |
| Cutoff            | Phải định nghĩa comparator và timezone                         |
| Blank last-use    | Unknown evidence, không mặc định inactive                      |
| Dormant           | Activity-based, policy-specific                                |
| Stale entitlement | Access không còn phù hợp dù account vẫn active                 |
| Orphaned          | Không map đáng tin cậy tới owner/identity/purpose              |
| Duplicate         | Phải kiểm business key/cardinality                             |
| Composite key     | Dùng account/system/ID context, không display name alone       |
| Service account   | Không áp human login logic một cách máy móc                    |
| Break-glass       | Rare use có thể expected; review control state                 |
| SoD               | Quan tâm conflict giữa combination of permissions              |
| Least privilege   | Actual access phải align required access                       |
| ServiceNow        | Approval evidence, không phải proof of actual access           |
| Effective access  | Technical permission thực tế                                   |
| IAM decision      | Keep / Revoke / Modify / Investigate                           |
| CVE               | Identifier, không phải severity                                |
| CVSS              | Technical severity input, không phải business priority         |
| Priority          | Phải xét exposure, exploitability, criticality, impact         |
| Exception         | Không phải ignore; cần owner/expiry/control/review             |
| Raw data          | Không chỉnh sửa source evidence                                |
| SHA-256           | Detect file change; không chứng minh source truth/completeness |
| Reproducibility   | Same inputs/rules → same business results                      |
| Idempotency       | Rerun không tạo duplicate side effects                         |
| Dry-run           | Preview intended changes, không phải proof of success          |
| Regression        | Change mới không được phá behavior cũ                          |
| Unknown           | Giữ unknown nếu evidence không đủ                              |

---

# 173. Kết luận Module Lý thuyết Day 13

Day 13 không thực chất là một bài “Excel lookup nâng cao”. Nó là bài học về cách biến một tập source records không hoàn hảo thành **defensible operational decisions**.

Bạn phải đồng thời hiểu bốn tầng:

```text
Governance
    ↓
Data Reconciliation
    ↓
Automation Assurance
    ↓
Audit Evidence
```

Nếu chỉ biết Governance mà không biết reconciliation, bạn không xử lý được thousands of records.

Nếu chỉ biết Excel/Python mà không hiểu Governance, bạn có thể tự động hóa một business rule sai.

Nếu logic đúng nhưng evidence không traceable, reviewer không thể chứng minh kết quả.

Nếu tất cả đúng nhưng rerun tạo duplicate tickets hoặc overwrite human decisions, automation vẫn chưa operationally safe.

Syllabus đặt acceptance rất rõ: source totals phải reconcile theo counting rule; seeded boundary cases phải đúng; mỗi reviewer decision phải trace được về source row và ticket; repeat runs phải cho cùng business result ngoài run-specific metadata; và không được tạo duplicate tasks hay overwrite reviewer decisions. Infrastructure_Operations_14_Da…

**Module Lý thuyết Day 13 đến đây đã cover toàn bộ phạm vi Concept/Lecture và additive theory trong syllabus**, bao gồm Identity Governance, 90-day review boundaries, user/privileged account lifecycle, activity evidence, dormant/stale/orphaned/duplicate handling, service/break-glass accounts, least privilege/SoD, AWS IAM/privileged-access/ServiceNow evidence, cross-system reconciliation, reviewer decisions, vulnerability assessment, CVE/CVSS, risk prioritization, multi-sheet control design, evidence integrity và automation testing/acceptance. Infrastructure_Operations_14_Da…

Bước tiếp theo phù hợp là **Module Thực hành Day 13**: chúng ta sẽ biến toàn bộ lý thuyết trên thành một simulated enterprise 90-day IAM review với đúng 12 sheets, source datasets, composite-key design, reconciliation logic, seeded exceptions, vulnerability register, VBA/Python automation, boundary/regression/idempotency tests, hash/count verification và final audit package.
