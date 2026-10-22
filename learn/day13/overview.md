# Day 13 — Tổng quan

Day 13 nằm trong **Module 7 — Operations Automation, IAM Audit & Integrated Practice** và là ngày chuyển từ “viết/chỉnh sửa automation” của Day 12 sang **governance + audit + reconciliation + testing + defensible evidence**. Trọng tâm không còn chỉ là script chạy được, mà là chứng minh một quy trình **90-day IAM review** có thể được thực hiện chính xác, lặp lại, audit được và tạo ra quyết định access có căn cứ. Infrastructure_Operations_14_Da…

Có thể chia toàn bộ Day 13 thành đúng **hai module lớn**:

1. **Module Lý thuyết — IAM Governance, Vulnerability Classification, Reconciliation & Automation Assurance**
2. **Module Thực hành — 90-Day IAM Review, Multi-Sheet Reconciliation, Vulnerability Register & Automation Testing**

Tôi sẽ trình bày ở đây ở mức **bản đồ kiến thức đầy đủ**, chưa đi sâu từng command, công thức Excel, VBA/Python implementation hay từng test case. Khi bạn yêu cầu tiếp, chúng ta sẽ đào sâu riêng từng module.

---

# MODULE 1 — LÝ THUYẾT

## 1. Identity Governance và periodic IAM certification

Phần đầu tiên phải hiểu là **Identity Governance** không chỉ là tạo/xóa account. Nó là quá trình kiểm soát toàn bộ lifecycle của identity và entitlement:

- Ai sở hữu account?
- Account có còn cần thiết không?
- User có đúng role hiện tại không?
- Quyền được cấp có còn phù hợp không?
- Có quyền thừa không?
- Approval còn hiệu lực không?
- Privileged access có được kiểm soát chặt hơn normal access không?
- Có account nào không xác định được owner?
- Có account tồn tại nhưng không có business justification?
- Quyết định review có bằng chứng hay chỉ dựa vào phỏng đoán?

Day 13 tập trung vào **periodic IAM certification**, đặc biệt là một **90-day IAM review**. Infrastructure_Operations_14_Da…

Điểm cực kỳ quan trọng cần học là:

> **“Review thực hiện mỗi 90 ngày” không đồng nghĩa với “account không hoạt động 90 ngày thì phải revoke”.**

Đây là hai policy concept độc lập:

- **Review frequency**: bao lâu tổ chức tiến hành certification một lần.
- **Inactivity threshold**: sau bao lâu không sử dụng thì account/access được xem là dormant/stale.

Hai giá trị có thể đều là 90 days, nhưng không được mặc định chúng là cùng một rule. Infrastructure_Operations_14_Da…

---

## 2. Review scope, review period và cutoff logic

Một IAM review defensible phải xác định rõ:

- review scope;
- system nào được đưa vào review;
- account type nào được đưa vào;
- review start time;
- review end time;
- **as-of time**;
- timezone;
- cutoff date/time;
- boundary là inclusive hay exclusive.

Ví dụ, nếu policy nói:

```text
Inactive for >= 90 days as of 2026-10-01 00:00:00 UTC
```

thì record ở:

```text
2026-07-03 00:00:00 UTC
```

có thể có kết quả khác record ở:

```text
2026-07-03 00:00:01 UTC
```

tùy cách policy định nghĩa.

Vì thế Day 13 yêu cầu test cả:

- immediately before cutoff;
- exactly on cutoff;
- immediately after cutoff;
- timezone boundary.

Infrastructure_Operations_14_Da…

---

## 3. User account và privileged account inventory

Bạn phải biết xây dựng inventory cho nhiều loại identity:

- normal user;
- privileged user;
- administrative account;
- AWS IAM identity;
- service account;
- break-glass account;
- disabled account;
- expired account;
- account đang active nhưng entitlement đã hết approval.

Các thuộc tính cần quan tâm thường bao gồm:

```text
Account ID
User ID
System
Account Type
Status
Owner
Manager
Role
Entitlement
Approval
Approval Expiry
Creation Date
Disable/Expiry Date
Last Sign-In
Last Permission Use
Source System
```

Mục tiêu không phải chỉ “liệt kê account”, mà phải trả lời được **ai sở hữu nó, vì sao tồn tại, được quyền gì và evidence nào chứng minh quyền đó hợp lệ**.

---

## 4. Account state: active, disabled, expired

Cần phân biệt rõ:

- **Active**: account hiện có thể được sử dụng.
- **Disabled**: account bị vô hiệu hóa.
- **Expired**: account vượt quá validity period hoặc entitlement expiration.
- **Revoked entitlement**: approval hoặc assignment đã bị thu hồi.
- **Account deleted**: object không còn tồn tại.

Một lỗi audit phổ biến là thấy ServiceNow ticket ghi “revoked” rồi cho rằng quyền thực tế đã mất.

Day 13 yêu cầu phát hiện cả tình huống:

> **revoked-but-still-active access**

tức business workflow nói revoke nhưng effective access trên source system vẫn còn. Infrastructure_Operations_14_Da…

---

## 5. Last sign-in và last-used permission evidence

Đây là một trong những phần khó nhất.

Bạn phải phân biệt:

### Last sign-in

Lần cuối identity authenticate.

### Last-used permission / last-accessed evidence

Lần cuối một service, permission hoặc entitlement được quan sát là đã sử dụng.

Hai loại evidence này không giống nhau.

Một user có thể:

- đăng nhập gần đây nhưng không sử dụng privileged entitlement;
- không có interactive sign-in nhưng service account vẫn hoạt động;
- có blank last-used vì source không hỗ trợ hoặc dữ liệu không đầy đủ.

Syllabus yêu cầu nguyên tắc:

> blank hoặc unsupported last-used value = **unknown evidence**, không phải tự động là “inactive”.

Infrastructure_Operations_14_Da…

---

## 6. Dormant users

**Dormant user/account** thường là identity không có activity theo threshold policy.

Nhưng Day 13 yêu cầu bạn không được đơn giản làm:

```text
LastLogin blank → Dormant → Revoke
```

Phải xem thêm:

- account age;
- service/break-glass exception;
- source coverage;
- last-used permission evidence;
- approval;
- owner;
- workload dependency;
- logging gap;
- retention window.

---

## 7. Stale entitlements

Một account có thể active hợp lệ nhưng entitlement của nó không còn hợp lệ.

Ví dụ:

```text
User: EMP00125
Current role: Finance Analyst
Entitlement: Domain Admin
Approval: role trước đây
```

Account không orphaned, user vẫn active, nhưng entitlement có khả năng **stale** hoặc excessive.

Do đó phải review cả:

```text
Identity → Role → Entitlement → Approval
```

chứ không chỉ:

```text
Identity → Account status
```

---

## 8. Orphaned accounts

Một account được xem là **orphaned** khi không thể liên hệ hợp lệ tới một owner/business identity.

Ví dụ:

```text
AWS IAM user = svc-old-report
HR/IdentityMaster = no match
Owner = blank
ServiceNow approval = none
```

Không được lập tức xóa.

Kết quả phù hợp hơn thường là:

```text
Investigate
```

sau đó xác định:

- owner;
- dependency;
- application;
- service;
- ticket;
- security impact.

---

## 9. Duplicate IDs

Duplicate có thể xuất hiện dưới nhiều dạng:

```text
same ID in same system
same person with multiple account IDs
same ID reused across systems
duplicate source row
duplicate privileged account
```

Điểm quan trọng là phải định nghĩa **business key** đúng.

Không được mặc định:

```text
Display Name = unique identity
```

Syllabus yêu cầu sử dụng documented **account/system/ID composite keys**, không join bằng display name alone. Infrastructure_Operations_14_Da…

Ví dụ:

```text
System + AccountID
```

hoặc:

```text
Environment + System + AccountID
```

---

## 10. Role change và access mismatch

Khi employee chuyển role:

```text
System Administrator
→
Project Coordinator
```

quyền cũ có thể vẫn tồn tại.

Đây là source phổ biến của:

- stale entitlement;
- excessive permission;
- SoD conflict;
- privilege accumulation.

Day 13 yêu cầu reconciliation phải có khả năng phát hiện những sai lệch dạng này.

---

## 11. Excessive permissions

**Excessive permissions** là entitlement vượt quá mức business requirement.

Đây là application trực tiếp của:

**least privilege**

Bạn cần hiểu cách xác định excessive access dựa trên:

- role;
- business requirement;
- approved request;
- entitlement baseline;
- permission-use evidence;
- segregation-of-duties rule.

Không thể chỉ nói “Admin = excessive”.

Có trường hợp Admin là đúng role và có approval hợp lệ.

---

## 12. Service accounts

Service account không thể được đánh giá giống human user.

Ví dụ:

```text
svc_backup
svc_monitor
svc_app_sync
```

Một service account có thể không có interactive login trong 90 days nhưng vẫn hoàn toàn hợp lệ.

Cần thêm:

- application owner;
- service dependency;
- credential owner;
- rotation policy;
- non-interactive activity;
- approval;
- purpose.

Day 13 cố tình đưa service accounts vào boundary test. Infrastructure_Operations_14_Da…

---

## 13. Break-glass accounts

**Break-glass account** là emergency access account.

Nó thường:

- ít sử dụng;
- có quyền rất cao;
- được bảo vệ và monitor đặc biệt.

Do đó:

```text
unused 90 days
```

không đồng nghĩa với:

```text
delete account
```

Phải kiểm tra:

- emergency-access policy;
- owner;
- credential protection;
- MFA;
- rotation;
- alerting;
- test procedure;
- review evidence.

---

## 14. Separation of Duties — SoD

**Separation of Duties (SoD)** kiểm soát việc một identity có tổ hợp quyền tạo ra conflict.

Ví dụ:

```text
Create Vendor
+
Approve Vendor Payment
```

hoặc:

```text
Developer
+
Production Approval
```

Day 13 yêu cầu IAM review không chỉ hỏi quyền có nhiều hay không, mà còn hỏi:

> **Tổ hợp quyền đó có vi phạm control không?**

---

## 15. Least-privilege compliance

Reviewer phải kiểm tra:

```text
Required Access
vs
Actual Access
```

Kết quả có thể là:

```text
Compliant
Excessive
Missing justification
Stale
Unknown
Exception approved
```

Đây là một trong các core decision areas của Day 13.

---

# 16. Nguồn evidence cho IAM review

Theo syllabus, Day 13 phải kết hợp evidence từ nhiều hệ thống, không dựa trên một source duy nhất. Infrastructure_Operations_14_Da…

Các nguồn chính gồm:

### AWS IAM

- credential/access reports;
- identity information;
- permissions;
- last-accessed information.

### Privileged-ID management exports

Evidence về privileged account/entitlement.

### PS/Gateway

Access history theo runbook thực tế của organization.

### AccessCheck

Entitlement/access history theo implementation thực tế.

### ServiceNow

- request;
- approval;
- change/access ticket;
- owner;
- timestamps.

### PalletControl / Asset inventory

Dùng để map identity hoặc finding tới actual infrastructure asset.

### Server inventory

Dùng để kiểm chứng account/asset relationship.

---

# 17. Cross-system identity reconciliation

Đây là core technical skill của Day 13.

Ví dụ:

```text
IdentityMaster
      │
      ├── AWS IAM
      ├── Privileged Access
      ├── ServiceNow
      ├── Asset Inventory
      └── Server Inventory
```

Bạn phải giải quyết các vấn đề:

```text
different ID formats
leading zero
case differences
whitespace
missing fields
duplicate records
many-to-one mapping
one-to-many mapping
system-specific IDs
```

Từ đó xây dựng identity lineage:

```text
Person
 ↓
Account
 ↓
System
 ↓
Entitlement
 ↓
Approval
 ↓
Asset
 ↓
Reviewer Decision
```

---

# 18. Reviewer decision model

Mỗi exception cuối cùng phải dẫn đến một decision có thể audit.

Syllabus chỉ rõ các quyết định:

```text
Keep
Revoke
Modify
Investigate
```

Mỗi decision cần:

- rationale;
- owner;
- reviewer;
- due date;
- evidence;
- linked task/ticket.

Infrastructure_Operations_14_Da…

Một spreadsheet chỉ highlight màu đỏ nhưng không có decision record là chưa đủ.

---

# 19. Vulnerability assessment và classification

Day 13 không chỉ làm IAM.

Phần thứ hai là **vulnerability assessment và classification**. Infrastructure_Operations_14_Da…

Bạn cần hiểu:

```text
Vulnerability
CVE
CVSS
Exploitability
Exposure
Asset Criticality
Severity
Business Impact
Remediation Priority
Exception
Compensating Control
Retest
```

---

## 20. CVE awareness

**CVE — Common Vulnerabilities and Exposures** là identifier cho public vulnerability.

Ví dụ format:

```text
CVE-2026-XXXXX
```

Day 13 không yêu cầu trở thành vulnerability researcher, nhưng cần biết cách associate finding với:

- affected product;
- affected version;
- CVE nếu applicable;
- scoring source.

---

## 21. CVSS awareness

**CVSS — Common Vulnerability Scoring System** cung cấp technical severity score.

Nhưng:

```text
CVSS != Business Risk
```

Ví dụ một CVSS cao trên isolated lab host có thể có remediation priority khác CVSS thấp hơn trên internet-facing production authentication server.

Do đó phải kết hợp:

```text
CVSS
+
Exposure
+
Exploitability
+
Asset Criticality
+
Business Impact
```

---

# 22. Vulnerability severity vs remediation priority

Day 13 yêu cầu không đánh đồng:

```text
Severity
```

với:

```text
Remediation Priority
```

Ví dụ:

```text
Severity: High
Internet Exposure: No
Asset Criticality: Low
Compensating Control: Strong
```

priority có thể khác một vulnerability có severity tương tự nhưng:

```text
Internet-facing
Critical production asset
Known exploitation
No compensating control
```

---

# 23. Exception acceptance và compensating controls

Không phải vulnerability nào cũng sửa được ngay.

Có thể tồn tại:

- technical constraint;
- application compatibility issue;
- vendor dependency;
- maintenance-window restriction.

Khi chưa remediate được cần quản lý qua:

```text
Risk Exception
Owner
Approval
Expiry
Compensating Control
Retest Date
```

---

# 24. Remediation tracking

Finding phải có lifecycle:

```text
Detected
→ Classified
→ Assigned
→ Remediation planned
→ Remediated
→ Retested
→ Closed
```

Không được dừng ở:

```text
High vulnerability found
```

---

# 25. IAM exception và vulnerability phải tách riêng

Đây là một yêu cầu đặc biệt của syllabus:

> **IAM exception không được trộn với CVE/misconfiguration vulnerability classification.**

Infrastructure_Operations_14_Da…

Ví dụ:

```text
Dormant admin account
```

là IAM/control exception.

Trong khi:

```text
CVE-xxxx on Windows Server
```

là vulnerability finding.

Chúng có thể liên quan risk nhưng classification workflow khác nhau.

---

# 26. Excel multi-sheet automation

Day 13 yêu cầu thiết kế workbook có kiểm soát.

Các sheet bắt buộc theo syllabus:

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

---

## 27. Chức năng của từng sheet

### `Control`

Chứa:

- run parameters;
- review date;
- cutoff;
- timezone;
- source versions;
- automation version.

### `IdentityMaster`

Canonical identity data.

### `IAM_Raw`

Untouched IAM import.

### `Privileged_Raw`

Untouched privileged-access export.

### `ServiceNow_Raw`

Approval/ticket source.

### `Asset_Raw`

Asset inventory.

### `Reconciliation`

Join results.

### `Exceptions`

Detected control exceptions.

### `Decisions`

Reviewer actions.

### `Vulnerabilities`

Vulnerability register.

### `TestResults`

Automation test results.

### `RunLog`

Execution metadata.

---

# 28. Preserve Raw Data

Một core principle:

> Raw source import phải được giữ nguyên.

Không:

```text
import → sửa trực tiếp → save
```

Mà nên:

```text
Raw Source
    ↓
Working / Normalized Copy
    ↓
Reconciliation
    ↓
Report
```

Điều này liên kết trực tiếp với evidence handling từ các Day trước. Evidence source phải có thể trace ngược. Infrastructure_Operations_14_Da…

---

# 29. Source-to-target ID matching

Automation phải match:

```text
source identity
→ target account
→ entitlement
→ approval
→ asset
```

Mỗi join rule phải documented.

Không được join chỉ bằng:

```text
"Nguyen Van A"
```

vì display name:

- không unique;
- có thể đổi;
- có spelling differences.

---

# 30. Composite keys

Composite key là key tạo từ nhiều fields.

Ví dụ:

```text
AWS|Production|A000123
```

từ:

```text
System
Environment
AccountID
```

Mục tiêu:

- giảm ambiguous match;
- tránh cross-system collision;
- đảm bảo deterministic reconciliation.

---

# 31. Lookup consistency

Các lookup phải xử lý nhất quán:

```text
Leading zeros
Case
Whitespace
Blank
Duplicates
Unexpected values
```

Ví dụ:

```text
001234
```

không được tự biến thành:

```text
1234
```

nếu ID là textual identifier.

---

# 32. Exception classification

Automation không chỉ tìm “không match”.

Phải phân loại được:

```text
Missing
Duplicate
Orphaned
Stale
Unknown owner
Missing approval
Expired approval
Excessive privilege
Role mismatch
Unknown last use
Revoked-but-active
Data quality error
Rejected source row
```

---

# 33. Audit trail preservation

Output phải cho reviewer đi ngược được:

```text
Decision
 ↓
Exception
 ↓
Reconciliation row
 ↓
Raw source row
 ↓
Source file
 ↓
Source system
```

Syllabus yêu cầu **every decision traces to a source row and ticket**. Infrastructure_Operations_14_Da…

---

# 34. Evidence completeness

Một audit artifact tốt không chỉ đúng business logic.

Cần có:

```text
Source
Timestamp
Timezone
File
Record count
SHA-256
Automation version
Run ID
Filters
Transformation
Ticket reference
```

Common evidence contract của syllabus còn yêu cầu source system/product/version, file size, access classification và limitations/errors. Infrastructure_Operations_14_Da…

---

# 35. SHA-256 checksum

SHA-256 được dùng để phát hiện source file có thay đổi không.

Ví dụ:

```text
source.csv
↓
SHA-256 = abc...
```

Sau processing:

```text
source.csv vẫn giữ nguyên
working-copy.csv thay đổi
```

Hash giúp chứng minh:

```text
file hiện tại == file đã hash trước đó
```

nhưng **không chứng minh source system ban đầu là đúng hoặc export là complete**. Đây là distinction rất quan trọng trong audit evidence. Infrastructure_Operations_14_Da…

---

# 36. Version control và reproducibility

Bạn cần có khả năng trả lời:

```text
Script version nào tạo report này?
Input nào?
Config nào?
Run khi nào?
Ai chạy?
Kết quả gì?
Có chạy lại được không?
```

Nếu cùng input và cùng business rule thì repeat run phải tạo:

> identical business results, bỏ qua run timestamp/ID.

Infrastructure_Operations_14_Da…

---

# 37. Automation test design

Day 13 nâng automation từ “chạy được” thành “được kiểm thử”.

Các loại test phải học:

```text
Positive Test
Negative Test
Boundary Test
Regression Test
Malformed-input Test
Incomplete-data Test
Duplicate-data Test
Inconsistent-data Test
Repeatability Test
Idempotency Test
Dry-run Test
```

Infrastructure_Operations_14_Da…

---

# 38. Positive tests

Valid input phải tạo expected valid output.

Ví dụ:

```text
Valid Identity
+ Valid IAM account
+ Valid owner
+ Valid approval
```

Expected:

```text
Matched
No Exception
```

---

# 39. Negative tests

Input hoặc condition sai phải bị phát hiện.

Ví dụ:

```text
Missing owner
```

Expected:

```text
Exception = Missing Owner
```

Không phải script crash hoặc im lặng bỏ record.

---

# 40. Boundary tests

Đặc biệt quan trọng với 90-day review:

```text
89d 23h 59m 59s
90d exactly
90d + 1s
```

và timezone boundaries.

---

# 41. Regression tests

Sau khi sửa code, những function từng đúng phải tiếp tục đúng.

Ví dụ sửa logic dormant account không được làm hỏng:

```text
duplicate detection
owner matching
approval matching
```

---

# 42. Idempotency

Nếu automation chạy lại cùng input:

```text
Run 1
Run 2
```

thì không được tạo duplicate operational side effects.

Ví dụ không được:

```text
same exception → create 2 ServiceNow tasks
```

Syllabus yêu cầu repeat run không tạo duplicate tasks hoặc overwrite reviewer decisions. Infrastructure_Operations_14_Da…

---

# 43. Dry-run

**Dry-run** là execution mode kiểm tra automation sẽ làm gì mà không thực hiện remediation thực.

Ví dụ:

```text
Would revoke A00123
Would modify role for A00124
```

nhưng không thực sự thay đổi access.

Đây là control rất quan trọng với IAM.

---

# 44. Error logging và defect tracking

Mỗi failure phải:

- observable;
- logged;
- classified;
- traceable.

Một test case cần ít nhất:

```text
Test Case ID
Requirement
Fixture Version
Expected Result
Actual Result
Pass/Fail
Defect/Ticket
Retest Status
```

Đây cũng là common automation acceptance contract của chương trình. Infrastructure_Operations_14_Da…

---

# MODULE 2 — THỰC HÀNH

Module thực hành của Day 13 là một **simulated enterprise IAM certification exercise**, không phải các bài nhỏ rời rạc. Assignment yêu cầu kết hợp AWS IAM, privileged-access systems, asset inventory, ServiceNow, Excel và automation thành một quy trình audit hoàn chỉnh. Infrastructure_Operations_14_Da…

---

## 1. Lab Objective tổng thể

Bạn phải thực hiện một:

> **Simulated 90-day privileged-access and IAM review**

với datasets từ:

```text
AWS IAM
PS/Gateway
AccessCheck
PalletControl
Server Inventory
ServiceNow
```

Sau đó tạo ra:

```text
90-Day IAM Review Report
Vulnerability Register
Multi-Sheet Reconciliation Workbook
Automation Test Evidence
Audit Trail
```

Infrastructure_Operations_14_Da…

---

# 2. Phase 1 — Review preparation

Trước processing phải xác định:

```text
Review ID
Review Scope
Review Period
As-of Time
Timezone
Cutoff Rule
Inactivity Policy
Systems Included
Account Types
Reviewer
Approver
Input Files
Automation Version
```

Không được bắt đầu bằng cách mở Excel rồi lookup ngay.

---

# 3. Phase 2 — Source intake

Import datasets tương ứng:

```text
AWS IAM export
PS/Gateway export
AccessCheck export
PalletControl export
Server inventory
ServiceNow approvals
Identity master
Vulnerability findings
```

Raw files phải:

- preserved;
- hashed;
- counted;
- timestamped;
- identified by source/version.

---

# 4. Phase 3 — Build workbook

Tạo workbook gồm:

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

Đây không phải suggested layout mà là layout explicitly yêu cầu trong additive Day 13 lab. Infrastructure_Operations_14_Da…

---

# 5. Phase 4 — Data normalization

Các identity fields cần được normalize có kiểm soát:

```text
Whitespace
Case
ID representation
System names
Environment
Account status
Dates
Timezone
Boolean/status values
```

Nhưng phải giữ original values để audit.

Ví dụ:

```text
RawAccountID        NormalizedAccountID
" 001234 "          "001234"
```

Không được làm mất leading zero.

---

# 6. Phase 5 — Cross-system reconciliation

Thực hiện matching giữa:

```text
IdentityMaster
IAM
Privileged Access
ServiceNow
Asset Inventory
```

với documented composite key.

Ví dụ:

```text
System + AccountID
```

Kết quả phải phân loại thành:

```text
Matched
Unmatched
Duplicate
Rejected
```

và totals phải reconcile theo counting rule đã documented. Infrastructure_Operations_14_Da…

---

# 7. Phase 6 — Detect IAM exceptions

Lab phải tìm được ít nhất các loại:

```text
Missing account
Duplicate account
Orphaned account
Stale account
Dormant account
Missing owner
Missing approval
Expired approval
Excessive permission
Role mismatch
Unknown last-use evidence
Revoked-but-still-active
```

Seeded fixtures còn bao gồm:

```text
Before cutoff
On cutoff
After cutoff
New account
Timezone boundary
Service account
Break-glass account
```

Infrastructure_Operations_14_Da…

---

# 8. Phase 7 — Validate owner và approvals

Mỗi privileged account cần kiểm tra:

```text
Owner exists?
Owner valid?
Business role valid?
Approval exists?
Approval covers this entitlement?
Approval still valid?
ServiceNow reference exists?
```

Không chỉ kiểm tra ticket number “có giá trị”.

---

# 9. Phase 8 — Reviewer decisions

Mỗi exception được đưa vào:

```text
Decisions
```

với:

```text
Keep
Revoke
Modify
Investigate
```

kèm:

```text
Rationale
Owner
Reviewer
Due Date
Evidence
Linked Task
```

---

# 10. Phase 9 — Effective-access verification

Nếu lab cho phép remediation trong sandbox:

```text
Revoke
Modify
```

thì sau thay đổi phải **recheck actual effective access**.

Không được chỉ dựa vào ticket state:

```text
Closed Complete
```

Syllabus chỉ cho phép remediation thực trên **authorized sandbox**. Infrastructure_Operations_14_Da…

---

# 11. Phase 10 — Vulnerability reconciliation

Nhận bộ vulnerability findings và map với asset inventory.

Mỗi record cần chứa:

```text
Affected Asset
Affected Version
CVE if applicable
Scoring Source
Scoring Version
Exposure
Business Impact
Priority
Remediation Owner
Exception Owner if applicable
Retest Evidence
```

Infrastructure_Operations_14_Da…

---

# 12. Phase 11 — Vulnerability classification

Bạn phải đánh giá:

```text
Technical Severity
Exploitability
Exposure
Asset Criticality
Business Impact
```

rồi quyết định:

```text
Remediation Priority
```

và owner.

Không chỉ sort theo CVSS.

---

# 13. Phase 12 — Build remediation records

Mỗi finding cần tracked record:

```text
Finding ID
Asset
CVE
Severity
Priority
Owner
Target Date
Status
Remediation
Exception
Compensating Control
Retest Evidence
```

---

# 14. Phase 13 — Develop/extend automation

Syllabus yêu cầu dùng:

```text
VBA
and/or
Python
```

để thực hiện reconciliation repeatably. Infrastructure_Operations_14_Da…

Automation phải:

```text
Read source
Validate input
Normalize
Build keys
Reconcile
Detect exceptions
Generate output
Record metadata
Log errors
Preserve input
```

---

# 15. Phase 14 — Execution metadata

Mỗi automation run cần ghi nhận ít nhất:

```text
Run ID
Execution Time
Timezone
Input Files
Input Hashes
Input Counts
Automation Version
Parameters
Cutoff Date
Output Files
Output Counts
Errors
Exit Status
```

Đây là phần tạo **reproducibility**.

---

# 16. Phase 15 — Automation test dataset

Lab không chỉ chạy production-like dataset.

Bạn phải test:

```text
Normal dataset
Malformed dataset
Incomplete dataset
Duplicate dataset
Inconsistent dataset
```

Infrastructure_Operations_14_Da…

---

# 17. Phase 16 — IAM boundary testing

Các fixture bắt buộc gồm:

```text
Before 90-day cutoff
Exactly on cutoff
After cutoff
New account
Unknown last-use
Service account
Break-glass account
Stale approval
Missing owner
Revoked-but-active
```

Điểm quan trọng là expected outcome phải theo **policy**, không theo intuition. Infrastructure_Operations_14_Da…

---

# 18. Phase 17 — Duplicate testing

Cần chứng minh script xử lý đúng:

```text
Duplicate source row
Duplicate account
Cross-system collision
Duplicate composite key
```

và không thực hiện silent many-to-many join.

Common acceptance contract đặc biệt yêu cầu điều này. Infrastructure_Operations_14_Da…

---

# 19. Phase 18 — Missing/incomplete data testing

Các case như:

```text
Missing owner
Missing approval
Blank last-use
Missing asset
Missing column
Missing ID
```

phải tạo explicit exception/error.

Không được silently discard.

---

# 20. Phase 19 — Malformed input testing

Ví dụ:

```text
Invalid date
Unexpected delimiter
Corrupted row
Wrong data type
Bad header
```

Automation phải fail hoặc quarantine record theo cách observable.

---

# 21. Phase 20 — Regression testing

Sau bất kỳ sửa đổi nào của VBA/Python:

```text
Known-good tests
```

phải chạy lại.

Test result phải ghi:

```text
Expected
Actual
Pass/Fail
```

---

# 22. Phase 21 — Repeatability test

Chạy cùng dataset ít nhất hai lần:

```text
Run A
Run B
```

Expected:

```text
Business results identical
```

ngoại trừ:

```text
Run ID
Timestamp
```

Infrastructure_Operations_14_Da…

---

# 23. Phase 22 — Idempotency test

Đặc biệt kiểm tra:

```text
No duplicate task
No duplicate exception
No duplicated side effect
No overwritten reviewer decision
```

khi script chạy lại.

---

# 24. Phase 23 — Verify data integrity

Kiểm tra:

```text
Raw source unchanged
SHA-256 unchanged
Record counts accounted for
Rejected rows documented
Output count explainable
```

Một acceptance quan trọng là:

> source total phải reconcile được thành matched + unmatched + duplicate + rejected theo counting rule đã xác định.

Infrastructure_Operations_14_Da…

---

# 25. Phase 24 — Traceability test

Reviewer phải chọn bất kỳ một decision và đi ngược được:

```text
Decision
→ Exception
→ Reconciliation
→ Raw Row
→ Source File
→ Source System
→ Ticket
```

Nếu không làm được thì audit trail chưa đạt.

---

# 26. Phase 25 — Final review package

Deliverables chính của Day 13 gồm:

### 1. 90-Day IAM Review Report

Tóm tắt:

```text
Scope
Cutoff
Population
Matches
Exceptions
Decisions
Open actions
Evidence gaps
```

### 2. Multi-Sheet Reconciliation Workbook

Bao gồm toàn bộ sheet structure.

### 3. Vulnerability Register

Gồm finding, asset, severity, priority, remediation và retest.

### 4. Automation

VBA/Python source cùng version.

### 5. Test Evidence

Positive, negative, malformed, incomplete, duplicate, inconsistent, boundary và regression tests.

### 6. Run/Audit Trail

Execution logs, hashes, counts và metadata.

### 7. Reviewer Decisions

Có rationale và ticket linkage.

Infrastructure_Operations_14_Da…

---

# Tiêu chuẩn hoàn thành Day 13

Để có thể coi Day 13 hoàn thành đúng syllabus, bạn phải chứng minh được đồng thời bốn năng lực.

| Năng lực                   | Bạn cần làm được                                                                                     |
| -------------------------- | ---------------------------------------------------------------------------------------------------- |
| **IAM Governance**         | Xác định review scope/cutoff, reconcile identity/access, tìm exception và đưa ra defensible decision |
| **Vulnerability Handling** | Map vulnerability tới asset, đánh giá severity/risk/priority và tracking remediation                 |
| **Automation & Data**      | Xây multi-sheet workbook và VBA/Python reconciliation có reproducibility                             |
| **Testing & Audit**        | Boundary/negative/regression/idempotency testing, hash/count verification và complete traceability   |

Acceptance quan trọng nhất gồm: source totals reconcile hợp lý; seeded boundary/exception cases cho đúng expected outcome; mọi decision trace được về source row và ticket; repeat run tạo cùng business result; không tạo duplicate task hoặc overwrite reviewer decisions. Infrastructure_Operations_14_Da…

---

# Quan hệ Day 12 → Day 13 → Day 14

**Day 12** dạy nền tảng để đọc, sửa và kiểm thử operational automation bằng Excel/VBA, Python, Batch, PowerShell, xử lý CSV/TXT/encoding và evidence. Infrastructure_Operations_14_Da…

**Day 13** dùng những kỹ năng đó để giải quyết một business-control problem thực tế: **90-day IAM certification + vulnerability reconciliation + automated audit evidence**. Infrastructure_Operations_14_Da…

**Day 14** sẽ tích hợp kết quả Day 13 vào enterprise operations scenario tổng thể, trong đó bạn phải thực hiện condensed IAM/access certification, vulnerability reconciliation và automation regression run cùng các mảng Linux, Windows, AWS, Nutanix, networking, security, backup và ITSM. Infrastructure_Operations_14_Da…

---

## Bản đồ Day 13

```text
DAY 13
│
├── MODULE 1 — LÝ THUYẾT
│   │
│   ├── Identity Governance
│   ├── 90-Day IAM Review
│   ├── Scope / Cutoff / Timezone
│   ├── Identity & Privileged Account Inventory
│   ├── Dormant / Stale / Orphaned / Duplicate
│   ├── Service / Break-glass Accounts
│   ├── Role Change / Excessive Permissions
│   ├── Least Privilege / SoD
│   ├── AWS IAM / PS-Gateway / AccessCheck
│   ├── ServiceNow Approval Evidence
│   ├── Cross-System Reconciliation
│   ├── Keep / Revoke / Modify / Investigate
│   │
│   ├── Vulnerability Assessment
│   ├── CVE / CVSS
│   ├── Exploitability / Exposure
│   ├── Asset Criticality / Business Impact
│   ├── Severity vs Remediation Priority
│   ├── Exceptions / Compensating Controls
│   │
│   ├── Excel Multi-Sheet Automation
│   ├── Composite Keys / Lookup Consistency
│   ├── Exception Classification
│   ├── Evidence / SHA-256 / Versioning
│   └── Testing / Regression / Idempotency
│
└── MODULE 2 — THỰC HÀNH
    │
    ├── Prepare 90-Day Review
    ├── Preserve & Hash Raw Sources
    ├── Build 12-Sheet Workbook
    ├── Normalize Identities
    ├── Build Composite Keys
    ├── Reconcile IAM / Privileged / ITSM / Assets
    ├── Detect IAM Exceptions
    ├── Validate Owners & Approvals
    ├── Reviewer Decisions
    ├── Effective Access Verification
    │
    ├── Vulnerability → Asset Mapping
    ├── Severity / Priority Classification
    ├── Remediation Register
    │
    ├── VBA/Python Automation
    ├── Execution Metadata / RunLog
    ├── Positive Tests
    ├── Negative Tests
    ├── Boundary Tests
    ├── Malformed / Incomplete Tests
    ├── Duplicate / Inconsistent Tests
    ├── Regression Tests
    ├── Repeatability / Idempotency Tests
    │
    ├── Hash / Count Verification
    ├── Source-to-Decision Traceability
    └── Final Audit Package
```

Đây là toàn bộ phạm vi Day 13 theo syllabus; về mặt chương trình, kỹ năng này được xem là phần **90-day IAM review, vulnerability classification, multi-sheet automation và test/audit trail** của Module 7. Infrastructure_Operations_14_Da…

Khi bạn yêu cầu tiếp theo **“Module Lý thuyết”**, tôi sẽ triển khai toàn bộ Module 1 theo kiểu foundation → operational workflow → security → failure modes → verification → troubleshooting → evidence → escalation, đi sâu từng topic và không bỏ sót mục nào trong phạm vi trên.
