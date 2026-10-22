# Day 13 — Module Thực hành

## 90-Day IAM Review, Cross-System Reconciliation, Vulnerability Register, Automation Testing & Audit Evidence

Module Thực hành Day 13 là nơi toàn bộ kiến thức Identity Governance, IAM certification, data reconciliation và automation assurance được ghép thành **một workflow enterprise hoàn chỉnh**. Đây không phải bài Excel độc lập, cũng không phải bài Python độc lập. Bạn đóng vai Infrastructure Operations analyst thực hiện một **simulated 90-day privileged-access and IAM review** bằng dữ liệu từ AWS IAM, PS/Gateway, AccessCheck, PalletControl, server inventory và ServiceNow; sau đó phải tạo exception, reviewer decision, vulnerability register, automation tests và một audit package có thể trace từ kết luận cuối cùng ngược về raw source. Infrastructure_Operations_14_Da…

Syllabus quy định rất rõ các control của lab: workbook phải có 12 sheets cụ thể; raw imports phải được giữ nguyên; join phải dùng documented account/system/ID composite key thay vì display name; phải có các fixture ở trước/đúng/sau 90-day cutoff, new account, timezone boundary, service/break-glass account, missing owner/approval, revoked-but-still-active và unknown last use; mỗi exception phải có decision `Keep/Revoke/Modify/Investigate`; remediation thật chỉ được thực hiện trong authorized sandbox; cuối cùng phải recheck effective access. Infrastructure_Operations_14_Da…

---

# A. Mục tiêu cuối cùng của Module Thực hành

Sau lab, bạn phải có khả năng thực hiện toàn bộ chuỗi:

```text
Policy / Review Requirement
          │
          ▼
Define Review Scope
          │
          ▼
Declare As-of / Cutoff / Timezone
          │
          ▼
Collect Source Exports
          │
          ▼
Hash + Count + Preserve Raw Evidence
          │
          ▼
Import as Text / Normalize Derived Data
          │
          ▼
Build Composite Keys
          │
          ▼
Cross-System Reconciliation
          │
          ├── Identity
          ├── Account
          ├── Entitlement
          ├── Owner
          ├── Approval
          └── Asset
          │
          ▼
Exception Classification
          │
          ├── Missing
          ├── Duplicate
          ├── Orphaned
          ├── Stale
          ├── Missing Owner
          ├── Missing Approval
          ├── Unknown Last Use
          └── Revoked-but-Active
          │
          ▼
Reviewer Decision
          │
   ┌──────┼────────┬─────────┐
   ▼      ▼        ▼         ▼
 Keep   Revoke   Modify   Investigate
          │
          ▼
Authorized Sandbox Remediation
          │
          ▼
Effective Access Recheck
```

Song song với IAM workflow là:

```text
Vulnerability Finding
        │
        ▼
Map to Asset / Version
        │
        ▼
CVE / Score Source if applicable
        │
        ▼
Exposure + Exploitability
        │
        ▼
Asset Criticality
        │
        ▼
Business Impact
        │
        ▼
Severity / Priority
        │
        ▼
Remediation / Exception Owner
        │
        ▼
Retest Evidence
```

Và toàn bộ workflow phải được bao phủ bởi:

```text
Automation
+
Tests
+
Run Logs
+
Hashes
+
Counts
+
Audit Traceability
```

---

# B. Environment / Topology của Lab

## 1. Lab phải là authorized non-production environment

Mặc định Day 13 sử dụng:

- prepared/sanitized datasets;
- synthetic user/account data;
- authorized AWS sandbox nếu có;
- Excel;
- Python;
- optional VBA;
- evidence repository;
- ticket/reference fixtures;
- answer key do instructor giữ.

Chương trình cũng quy định datasets 90-day được chuẩn bị trước; learner không chờ thực tế 90 ngày mới làm review. Infrastructure_Operations_14_Da…

Topology logic:

```text
                        ┌─────────────────┐
                        │ IdentityMaster  │
                        └────────┬────────┘
                                 │
               ┌─────────────────┼──────────────────┐
               │                 │                  │
               ▼                 ▼                  ▼
        ┌────────────┐    ┌──────────────┐   ┌─────────────┐
        │  AWS IAM   │    │ PS/Gateway / │   │ ServiceNow  │
        │   Export   │    │ AccessCheck  │   │   Export    │
        └──────┬─────┘    └──────┬───────┘   └──────┬──────┘
               │                 │                  │
               └─────────────────┼──────────────────┘
                                 │
                                 ▼
                         ┌───────────────┐
                         │Reconciliation │
                         └───────┬───────┘
                                 │
                ┌────────────────┼─────────────────┐
                ▼                ▼                 ▼
          Exceptions         Decisions       Audit Evidence


        PalletControl / Server Inventory
                    │
                    ▼
               Asset_Raw
                    │
             ┌──────┴─────────┐
             ▼                ▼
      IAM Asset Context   Vulnerability
                            Mapping
```

---

# C. Prerequisites

Trước khi bắt đầu, bạn cần có ít nhất:

| Requirement              | Mục đích                                         |
| ------------------------ | ------------------------------------------------ |
| Excel                    | Workbook, reconciliation, reviewer-facing report |
| Python 3                 | Repeatable reconciliation/testing                |
| Optional VBA             | Workbook-side automation                         |
| Hash utility             | SHA-256 evidence verification                    |
| Sanitized source exports | Input dataset                                    |
| Evidence repository      | Lưu raw/working/output/test evidence             |
| Review policy fixture    | Định nghĩa threshold/cutoff                      |
| Ticket/change reference  | Audit linkage                                    |
| Test answer key          | Expected results                                 |
| Authorized AWS sandbox   | Chỉ nếu lab có AWS live access                   |

Bạn không cần và không được đưa vào lab:

```text
real password
private key
secret access key
session token
MFA seed
production credential
```

---

# D. Safety Notes — cực kỳ quan trọng

## Read-only và configuration-changing phải tách biệt

Phần lớn lab nên là:

```text
Read
Export
Hash
Normalize
Compare
Classify
Report
```

không phải:

```text
Disable
Delete
Detach policy
Change production permission
```

Nếu thực hành remediation:

```text
Revoke
Modify
Disable
```

chỉ được thực hiện trong **authorized sandbox** và phải có:

```text
Approval
Pre-check
Expected impact
Rollback/restore path
Post-check
Evidence
```

Syllabus quy định rõ actual remediation trong Day 13 chỉ nên chạy trong authorized sandbox và phải recheck effective access sau đó. Infrastructure_Operations_14_Da…

---

# E. Deliverables bắt buộc

Đến cuối Day 13, bạn phải tạo được tối thiểu:

1. `90-Day IAM Review Report`
2. `IAM_Reconciliation.xlsx`
3. `Vulnerability_Register`
4. `Exceptions Register`
5. `Reviewer Decisions`
6. VBA/Python automation source
7. Test dataset/fixtures
8. `TestResults`
9. `RunLog`
10. Evidence manifest
11. Source SHA-256 inventory
12. Raw source copies
13. Derived/normalized working copies
14. Execution logs
15. Final audit trail

Đó chính là output mà syllabus yêu cầu: complete IAM review report, vulnerability register, multi-sheet reconciliation workbook, test evidence và audit trail. Infrastructure_Operations_14_Da…

---

# F. Cấu trúc Evidence Repository

Một layout lab phù hợp:

```text
Day13_IAM_Review/
│
├── 00_Control/
│   ├── review_scope.txt
│   ├── review_policy.txt
│   └── change_ticket_reference.txt
│
├── 01_Raw/
│   ├── IdentityMaster.csv
│   ├── AWS_IAM.csv
│   ├── PrivilegedAccess.csv
│   ├── ServiceNow.csv
│   ├── AssetInventory.csv
│   └── Vulnerabilities.csv
│
├── 02_Working/
│   ├── normalized_identity.csv
│   ├── normalized_iam.csv
│   └── reconciliation.csv
│
├── 03_Automation/
│   ├── reconcile.py
│   ├── config.json
│   └── optional_vba.bas
│
├── 04_Output/
│   ├── IAM_Reconciliation.xlsx
│   ├── IAM_Review_Report.xlsx
│   └── vulnerability_register.csv
│
├── 05_Test/
│   ├── fixtures/
│   ├── expected/
│   └── TestResults.csv
│
├── 06_Logs/
│   └── run_*.log
│
└── 07_Evidence/
    ├── evidence_manifest.csv
    ├── sha256_manifest.txt
    └── screenshots/
```

Điểm quan trọng:

```text
01_Raw
```

không được dùng như working folder.

Raw phải bất biến.

Syllabus yêu cầu preserve untouched source export và chỉ process copy; redacted copy cũng phải có hash riêng và redaction note. Infrastructure_Operations_14_Da…

---

# G. Phase 0 — Xác định Review Contract trước khi đụng dữ liệu

Đây là bước rất hay bị bỏ qua.

Bạn chưa được mở CSV rồi tự đoán rule.

Trước tiên phải tạo review contract.

Ví dụ:

```text
Review ID:
IAM-2026-Q3-001

Environment:
LAB

Review Type:
90-Day Periodic IAM Certification

Review Start:
2026-10-01T00:00:00Z

Review End:
2026-10-05T23:59:59Z

As-of:
2026-10-01T00:00:00Z

Timezone:
UTC

Inactivity Threshold:
90 days

Inactive Comparator:
LastActivity <= Cutoff

Cutoff:
2026-07-03T00:00:00Z

Included:
AWS IAM users
Privileged accounts
Service accounts
Break-glass accounts

Excluded:
Training-only identities explicitly listed in scope

Reviewer:
REVIEWER-001
```

Điểm quan trọng:

```text
Review every 90 days
```

không được tự động suy ra:

```text
Inactive threshold = 90 days
```

Threshold phải được policy nói rõ. Infrastructure_Operations_14_Da…

---

# H. Phase 1 — Verify Source Inventory

Trước processing, lập bảng:

| Source ID | Source               | File                 | Expected purpose                |
| --------- | -------------------- | -------------------- | ------------------------------- |
| SRC-001   | IdentityMaster       | IdentityMaster.csv   | Canonical identity              |
| SRC-002   | AWS IAM              | AWS_IAM.csv          | Cloud accounts/credential state |
| SRC-003   | Privileged Access    | PrivilegedAccess.csv | Privileged IDs                  |
| SRC-004   | ServiceNow           | ServiceNow.csv       | Approval/ticket records         |
| SRC-005   | Asset inventory      | AssetInventory.csv   | Asset/business context          |
| SRC-006   | Vulnerability source | Vulnerabilities.csv  | Security findings               |

Bạn phải biết:

```text
Which system?
Which export?
Which version?
Which time?
Who collected?
How many rows?
```

---

# I. Optional AWS live collection trong sandbox

Nếu lab cho phép AWS read-only activity, trước hết verify active identity:

```bash
aws sts get-caller-identity
```

Command này trả thông tin identity đang dùng để gọi AWS API, giúp tránh thao tác nhầm account/context. AWS hiện vẫn document `aws sts get-caller-identity` cho mục đích xác định caller identity. [AWS Docs](https://docs.aws.amazon.com/cli/latest/userguide/cli_sts_code_examples.html?utm_source=chatgpt.com)

Credential report có thể được tạo bằng:

```bash
aws iam generate-credential-report
```

và retrieve bằng:

```bash
aws iam get-credential-report
```

AWS document rằng credential report chứa IAM users và trạng thái các IAM-managed credentials; quyền cần để tạo/tải report lần lượt là `iam:GenerateCredentialReport` và `iam:GetCredentialReport`. Report không bao phủ mọi loại service-specific credential, nên không được coi nó là complete universal credential inventory. [AWS Docs](https://docs.aws.amazon.com/us_en/IAM/latest/UserGuide/id_credentials_getting-report.html?utm_source=chatgpt.com)

### Security

Không ghi vào transcript:

```text
SecretAccessKey
SessionToken
Password
MFA seed
```

Nếu output chứa sensitive field ngoài scope, sanitize/redact working copy và giữ raw theo approved repository policy.

---

# J. Phase 2 — Hash và preserve source evidence

Trước khi mở/edit file:

## Linux

```bash
sha256sum IdentityMaster.csv
sha256sum AWS_IAM.csv
sha256sum PrivilegedAccess.csv
```

Expected concept:

```text
<64-character SHA-256>  IdentityMaster.csv
```

## PowerShell

```powershell
Get-FileHash .\IdentityMaster.csv -Algorithm SHA256
```

Ví dụ output:

```text
Algorithm Hash                                                             Path
--------- ----                                                             ----
SHA256    88A4...                                                          IdentityMaster.csv
```

Hash phải lưu vào evidence manifest.

Nhưng nhớ:

```text
Matching SHA-256
```

chỉ chứng minh bytes hiện tại match với file đã hash trước đó.

Nó **không chứng minh**:

```text
export đầy đủ
source truthful
query đúng
time window đúng
```

Đây là yêu cầu evidence rất quan trọng của syllabus. Infrastructure_Operations_14_Da…

---

# K. Phase 3 — Count source records

Hash chưa đủ.

Ví dụ:

```text
IdentityMaster.csv     1,002 rows
AWS_IAM.csv              305 rows
PrivilegedAccess.csv     128 rows
ServiceNow.csv           142 rows
AssetInventory.csv       417 rows
Vulnerabilities.csv       35 rows
```

Ghi lại:

```text
Header count
Data record count
Blank row count
Rejected row count
```

Nếu tool import chỉ load:

```text
AWS_IAM = 299
```

trong khi raw có:

```text
305
```

phải giải thích 6 records còn lại.

Không được silent loss.

---

# L. Phase 4 — Tạo workbook 12 sheets

Theo syllabus, workbook phải có đúng các functional sheets:

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

# M. Sheet 1 — Control

Đây là configuration và audit context của toàn run.

Recommended fields:

| Field                   | Example              |
| ----------------------- | -------------------- |
| ReviewID                | IAM-2026-Q3-001      |
| Environment             | LAB                  |
| ReviewFrequency         | 90 Days              |
| AsOfTime                | 2026-10-01T00:00:00Z |
| Timezone                | UTC                  |
| InactivityThresholdDays | 90                   |
| CutoffTime              | 2026-07-03T00:00:00Z |
| BoundaryRule            | `<= Cutoff`          |
| AutomationVersion       | 1.0.0                |
| RunMode                 | DryRun               |
| Reviewer                | REVIEWER-001         |
| ApprovalTicket          | CHG-LAB-001          |
| SourceManifest          | EV-001               |

### Verification

Reviewer phải nhìn `Control` và biết:

```text
Automation đã dùng rule nào?
As-of nào?
Timezone nào?
Version nào?
```

Nếu các giá trị này chỉ nằm hard-coded trong Python, workbook audit-friendly kém hơn.

---

# N. Sheet 2 — IdentityMaster

Đây là canonical identity population.

Suggested fields:

```text
IdentityID
EmployeeID
IdentityType
DisplayName
Status
CurrentRole
Department
ManagerID
OwnerID
JoinDate
EndDate
SourceRowID
```

Ví dụ:

| IdentityID | IdentityType | Status   | CurrentRole      |
| ---------- | ------------ | -------- | ---------------- |
| 000123     | Human        | Active   | NetworkEngineer  |
| 000124     | Human        | Disabled | FormerEmployee   |
| SVC001     | Service      | Active   | ReportingService |
| BG001      | BreakGlass   | Active   | EmergencyAdmin   |

## Quan trọng

IDs phải import dưới dạng **text**.

```text
000123
```

không được biến thành:

```text
123
```

Nếu đã mất leading zero ở bước import, downstream formula không thể biết chắc original value là bao nhiêu.

---

# O. Sheet 3 — IAM_Raw

Đây là raw cloud/access source.

Ví dụ schema:

```text
SourceRowID
System
Environment
AccountID
UserName
AccountStatus
OwnerID
Role
Entitlement
CreatedTime
LastSignIn
LastPermissionUse
CredentialState
```

Không thêm classification trực tiếp vào raw nếu có thể tránh.

Raw:

```text
source evidence
```

Derived:

```text
Reconciliation
```

---

# P. Sheet 4 — Privileged_Raw

Có thể bao gồm:

```text
SourceRowID
PrivilegedSystem
Environment
AccountID
IdentityID
PrivilegeRole
Status
ValidFrom
ValidTo
LastAccess
OwnerID
```

Đối với PS/Gateway và AccessCheck, exact field name phải lấy từ actual runbook/product export. Syllabus cảnh báo không được giả định generic workflow là product-specific proficiency cho các tên sản phẩm organization-specific này. Infrastructure_Operations_14_Da…

Vì vậy trong lab simulation có thể normalize về schema chung, nhưng vẫn giữ original raw columns.

---

# Q. Sheet 5 — ServiceNow_Raw

Suggested fields:

```text
SourceRowID
TicketNumber
SysID
RequestType
AccountID
IdentityID
RequestedEntitlement
ApprovalState
Approver
RequestedDate
ApprovedDate
ValidUntil
State
```

Nếu `sys_id` được authorized export, giữ stable ID.

Không chỉ dùng display ticket number nếu stable key tốt hơn có sẵn.

---

# R. Sheet 6 — Asset_Raw

Suggested:

```text
AssetID
Hostname
System
Environment
BusinessService
Criticality
Owner
LifecycleStatus
Version
SourceRowID
```

Asset mapping sẽ được dùng cả:

```text
IAM context
```

và:

```text
Vulnerability prioritization
```

---

# S. Phase 5 — Import đúng cách

Đây là bước rất dễ phá dữ liệu.

## Không double-click CSV một cách mù quáng

Excel có thể tự interpret:

```text
001234 → 1234
```

hoặc:

```text
1-2 → date
```

hoặc long identifiers thành scientific notation.

Thay vào đó, khi import phải explicitly xác định:

```text
AccountID = Text
IdentityID = Text
TicketID = Text
```

Timestamp phải giữ đủ timezone information nếu source cung cấp.

---

# T. Phase 6 — Normalization Strategy

Raw giữ nguyên.

Reconciliation tạo derived normalized fields.

Ví dụ raw:

```text
System = " aws-prod "
AccountID = " 001234 "
Environment = "Prod"
```

Derived:

```text
System_Norm      = "AWS-PROD"
AccountID_Norm   = "001234"
Environment_Norm = "PROD"
```

---

# U. Excel normalization examples

Giả sử Excel Table:

```text
tblIAM
```

### Trim whitespace

```excel
=TRIM([@AccountID])
```

### Normalize case cho field được policy xác định là case-insensitive

```excel
=UPPER(TRIM([@System]))
```

### Composite key

```excel
=UPPER(TRIM([@System])) & "|" &
 UPPER(TRIM([@Environment])) & "|" &
 TRIM([@AccountID])
```

Kết quả:

```text
AWS|PROD|001234
```

Không được normalize case một cách tùy ý nếu source identifier thực tế case-sensitive.

---

# V. Source-to-target Composite Keys

Một join key tốt phải phản ánh uniqueness scope.

Sai:

```text
DisplayName
```

Tốt hơn:

```text
System + Environment + AccountID
```

Ví dụ:

```text
AWS|PROD|admin001
```

và:

```text
WINDOWS|PROD|admin001
```

không bị coi là cùng account.

---

# W. Kiểm tra Composite Key uniqueness trước khi join

Ví dụ Excel:

```excel
=COUNTIF(Reconciliation[CompositeKey],[@CompositeKey])
```

Nếu:

```text
Count = 1
```

key unique trong dataset.

Nếu:

```text
Count > 1
```

phải điều tra duplicate/cardinality.

Không được cứ chạy `XLOOKUP` lấy record đầu tiên.

---

# X. Silent many-to-many join — lỗi rất nguy hiểm

Source A:

| Key  | Row |
| ---- | --- |
| X001 | A1  |
| X001 | A2  |

Source B:

| Key  | Row |
| ---- | --- |
| X001 | B1  |
| X001 | B2  |

Naive database merge tạo:

```text
A1-B1
A1-B2
A2-B1
A2-B2
```

Hai records mỗi bên biến thành bốn combinations.

Syllabus yêu cầu identity matching không được có accidental numeric conversion hoặc silent many-to-many join. Infrastructure_Operations_14_Da…

---

# Y. Phase 7 — Xây Reconciliation Sheet

Một schema hữu ích:

```text
ReconID
CompositeKey
System
Environment
AccountID
IdentityID
IdentityMatch
AccountStatus
IdentityStatus
OwnerSource
OwnerMaster
OwnerMatch
Role
Entitlement
ApprovalTicket
ApprovalMatch
ApprovalState
LastSignIn
LastPermissionUse
LastUseEvidenceState
AssetID
AssetMatch
DuplicateFlag
ExceptionType
SourceIAMRow
SourcePrivilegedRow
SourceServiceNowRow
SourceAssetRow
```

Điều cần đạt:

```text
Every result
→ source row
```

---

# Z. Match dimensions phải tách biệt

Không dùng một field duy nhất:

```text
MATCHED = TRUE
```

vì record có thể:

```text
IdentityMatch = Yes
OwnerMatch = Yes
ApprovalMatch = No
AssetMatch = Yes
```

Kết luận lúc này không phải clean match.

Nên có các dimension rõ:

```text
IdentityMatched
AccountMatched
OwnerMatched
ApprovalMatched
AssetMatched
```

---

# AA. Phase 8 — Counting Rule

Syllabus yêu cầu source totals phải reconcile thành:

```text
Matched
Unmatched
Duplicate
Rejected
```

theo một counting rule được documented. Infrastructure_Operations_14_Da…

Ví dụ policy lab:

```text
Matched:
Record has valid unique key and expected target association.

Unmatched:
Valid input record but no corresponding target association.

Duplicate:
Key violates required uniqueness.

Rejected:
Input cannot be evaluated because required schema/data is invalid.
```

Ví dụ:

```text
IAM source total       305

Matched                275
Unmatched               14
Duplicate               10
Rejected                 6
--------------------------
Total                   305
```

Acceptance:

```text
275 + 14 + 10 + 6 = 305
```

Nếu không bằng, bạn phải tìm missing/double-counted rows.

---

# AB. Duplicate và Unmatched phải được định nghĩa mutually exclusive nếu bạn dùng phép cộng tổng

Nếu duplicate record vừa được tính:

```text
Duplicate = 10
```

vừa nằm trong:

```text
Unmatched = 14
```

thì:

```text
Matched + Unmatched + Duplicate + Rejected
```

sẽ double count.

Bạn phải document rule.

Ví dụ:

```text
Classification precedence:

1. Rejected
2. Duplicate
3. Matched
4. Unmatched
```

Mỗi source record nhận đúng một population status.

Sau đó IAM exception có thể là một dimension khác.

---

# AC. Phase 9 — Phân biệt Population Status và IAM Exception

Ví dụ record:

```text
PopulationStatus = Matched
```

nhưng:

```text
ExceptionType = MissingApproval
```

Không mâu thuẫn.

`Matched` nói:

```text
source-to-target relationship exists
```

`MissingApproval` nói:

```text
governance control failed
```

Đây là distinction rất quan trọng.

---

# AD. Phase 10 — Detect Missing Account

Ví dụ:

```text
IdentityMaster:
Identity 000123 is active

Expected Account:
AWS PROD

IAM:
No matching account
```

Có thể classify:

```text
MissingAccount
```

nhưng phải xem requirement có thực sự nói identity phải có account đó hay không.

Không được suy luận mọi active employee đều phải có AWS account.

---

# AE. Detect Duplicate Account

Ví dụ:

```text
AWS|PROD|admin001
AWS|PROD|admin001
```

Nếu uniqueness contract nói mỗi composite key chỉ một record:

```text
Exception = DuplicateAccount
```

Evidence nên lưu:

```text
both source row IDs
```

---

# AF. Detect Orphaned Account

Candidate:

```text
IAM Account exists
IdentityMaster match = None
Owner match = None
```

Không nên ngay lập tức gắn final:

```text
ConfirmedOrphan
```

an toàn hơn:

```text
CandidateOrphan
```

sau investigation.

Bạn phải kiểm:

```text
Service account?
Break-glass account?
Historical system?
Identity source gap?
Incorrect key?
```

---

# AG. Detect Dormant Account

Giả sử policy lab nói:

```text
Threshold = 90 days
AsOf = 2026-10-01T00:00Z
Comparator = LastActivity <= Cutoff
```

Chỉ được classify dormant nếu:

```text
Activity evidence is supported
AND
Timestamp valid
AND
Policy applies to account type
AND
Boundary result satisfies rule
```

Không làm:

```text
blank → dormant
```

Syllabus nói blank/unsupported last-used là **unknown evidence**, không phải proof of inactivity. Infrastructure_Operations_14_Da…

---

# AH. Boundary fixtures

Lab bắt buộc nên có:

```text
CASE-BOUND-01
LastUse = 2026-07-02 23:59:59Z

CASE-BOUND-02
LastUse = 2026-07-03 00:00:00Z

CASE-BOUND-03
LastUse = 2026-07-03 00:00:01Z
```

Nếu rule:

```text
LastUse <= Cutoff
```

expected:

```text
BOUND-01 = Dormant
BOUND-02 = Dormant
BOUND-03 = Not Dormant
```

Nếu policy khác, expected cũng phải thay đổi.

Không có universal answer ngoài policy.

---

# AI. Timezone boundary test

Fixture:

```text
Raw Local Time:
2026-07-03 00:30:00 +07:00
```

Normalized UTC:

```text
2026-07-02 17:30:00Z
```

Nếu cutoff:

```text
2026-07-03 00:00:00Z
```

thì normalized value nằm trước cutoff.

Nếu automation bỏ timezone và chỉ lấy local date:

```text
2026-07-03
```

classification có thể sai.

---

# AJ. New Account fixture

Ví dụ:

```text
Created:
2026-09-29

LastUse:
blank
```

Sai:

```text
Blank last use → Dormant
```

Expected:

```text
LastUseEvidence = Unknown / Not Yet Observed
```

và classification dựa trên policy dành cho new account.

Không invent grace period nếu policy fixture không định nghĩa.

---

# AK. Service Account fixture

```text
AccountID:
svc_backup_prod

IdentityType:
Service

LastInteractiveLogin:
blank

Owner:
BackupOperations

Workload:
Active

Approval:
Valid
```

Expected:

```text
Do not apply normal human interactive-login dormancy rule.
```

Phải review:

```text
Owner
Purpose
Technical activity
Entitlement
Approval
Dependency
```

---

# AL. Break-Glass Account fixture

```text
AccountID:
bg-prod-admin

Type:
BreakGlass

LastUse:
220 days ago

Owner:
Security Operations

Policy:
Emergency account

Monitoring:
Active

Approval/control:
Valid
```

Không được tự động:

```text
220 days → Revoke
```

Expected classification có thể là:

```text
SpecialAccountReview
```

rồi kiểm break-glass controls.

---

# AM. Missing Owner fixture

```text
Account:
prod-db-admin-02

Identity:
matched

Owner:
blank

Activity:
recent
```

Expected:

```text
MissingOwner
```

Recent activity không giải quyết ownership failure.

---

# AN. Missing Approval fixture

```text
Actual entitlement:
Administrator

Approval source:
No match
```

Expected:

```text
MissingApproval
```

Nhưng initial disposition nên là:

```text
Investigate
```

cho tới khi xác định:

```text
Unauthorized?
Historical approval unavailable?
Mapping error?
Retention gap?
```

---

# AO. Stale Approval fixture

```text
Entitlement:
Active

Approval ValidUntil:
2026-08-31

AsOf:
2026-10-01
```

Potential exception:

```text
StaleApproval
```

hoặc:

```text
ExpiredAuthorization
```

theo taxonomy của lab.

---

# AP. Revoked-but-Still-Active fixture

Đây là case cực kỳ quan trọng:

```text
ServiceNow:
REVOKED

Privileged target:
ACTIVE
```

Expected:

```text
ExceptionType = RevokedButActive
```

Không được chỉ nhìn ticket state và cho pass.

Root-cause candidates:

```text
Provisioning failure
Wrong account mapping
Manual revoke missed
Synchronization failure
Ticket closed incorrectly
```

---

# AQ. Unknown Last Use

Fixture:

```text
LastUse = blank
SourceCapability = unsupported
```

Expected:

```text
LastUseEvidenceState = Unknown
```

Không:

```text
Inactive
```

Reviewer action có thể là:

```text
Investigate
```

tùy account/risk context.

---

# AR. Role Change fixture

IdentityMaster:

```text
CurrentRole:
BusinessAnalyst
```

IAM:

```text
Entitlement:
ProductionAdministrator
```

Previous data:

```text
OldRole:
SystemsEngineer
```

Potential classification:

```text
RoleMismatch
StaleEntitlement
PotentialExcessivePrivilege
```

Phải validate approval/current responsibility trước final decision.

---

# AS. Least-Privilege fixture

Ví dụ:

```text
Required role:
ReadOnly

Actual:
Administrator
```

Expected:

```text
ExcessivePrivilegeCandidate
```

Reviewer có thể quyết định:

```text
Modify
```

nếu evidence xác nhận chỉ cần ReadOnly.

---

# AT. SoD fixture

Ví dụ:

```text
Identity:
EMP001

Entitlement A:
CreatePrivilegedUser

Entitlement B:
ApprovePrivilegedUser
```

Nếu conflict matrix của organization cấm combination này:

```text
Exception = SoDConflict
```

Nhưng phải xem:

```text
Approved exception?
Compensating control?
Expiry?
```

---

# AU. Phase 11 — Exceptions Sheet

Recommended schema:

```text
ExceptionID
ReconID
ExceptionType
Severity
DetectedDate
AccountID
IdentityID
System
Environment
Description
EvidenceRef
SourceRowRef
TicketRef
Status
```

Ví dụ:

```text
EX-0007
REC-0144
RevokedButActive
High
admin-012
000123
AWS
PROD
Approved access was revoked in governance system but technical entitlement remains active.
EV-0042
IAM_Raw!219
REQ001889
Open
```

---

# AV. Không nhét Reviewer Decision trực tiếp vào Exceptions logic

Automation có thể tạo:

```text
ExceptionType
```

nhưng reviewer nhập:

```text
Decision
Rationale
```

ở `Decisions`.

Nếu rerun automation xóa/recreate Exceptions sheet, reviewer decision vẫn phải được bảo tồn.

Syllabus acceptance đặc biệt cấm overwrite reviewer decisions. Infrastructure_Operations_14_Da…

---

# AW. Phase 12 — Decisions Sheet

Schema:

```text
DecisionID
ExceptionID
Decision
Rationale
Reviewer
DecisionDate
Owner
DueDate
EvidenceRef
LinkedTask
ImplementationStatus
VerificationStatus
```

Allowed decisions theo syllabus:

```text
Keep
Revoke
Modify
Investigate
```

---

# AX. Ví dụ decision — Keep

```text
Decision:
Keep

Rationale:
Service account remains required by active backup workload.
Owner confirmed as Backup Operations.
Privilege aligns with documented workload.
Approval remains valid.
```

`Keep` không được là:

```text
Looks okay.
```

---

# AY. Ví dụ decision — Revoke

```text
Decision:
Revoke

Rationale:
User left the organization.
Current identity state is disabled.
Entitlement has no current business requirement.
Approved removal ticket exists.
```

Nhưng `Decision=Revoke` chưa chứng minh access đã thực sự mất.

Phải có:

```text
ImplementationStatus
VerificationStatus
```

---

# AZ. Ví dụ decision — Modify

```text
Current:
Administrator

Required:
ReadOnlyOperator

Decision:
Modify
```

Đây là least-privilege remediation.

---

# BA. Ví dụ decision — Investigate

```text
Account:
svc_legacy01

Owner:
Unknown

Activity evidence:
Unavailable

Dependency:
Not established
```

An toàn:

```text
Investigate
```

không:

```text
Delete
```

---

# BB. Phase 13 — Sandbox remediation

Chỉ thực hiện trên account fixture hoặc authorized sandbox.

Ví dụ workflow:

```text
1. Capture current entitlement
2. Capture ticket/approval
3. Confirm target account
4. Dry-run if automation supports it
5. Execute approved change
6. Re-read effective access
7. Compare before/after
8. Store evidence
```

---

# BC. Verification sau Revoke

Không chỉ kiểm:

```text
ServiceNow state = Complete
```

Mà phải kiểm:

```text
Technical system no longer grants entitlement
```

Ví dụ:

```text
Before:
Account admin-012
Role = Administrator

After:
Role absent
```

Sau đó recorded:

```text
EffectiveAccessVerification = Passed
```

---

# BD. Rollback cho Modify/Revoke

Rollback không phải lúc nào cũng:

```text
undo automatically
```

Với access remediation, rollback chỉ được thực hiện nếu:

```text
Decision sai
Change gây outage
Approved rollback criteria triggered
```

Phải biết:

```text
Original entitlement
Original scope
Original expiry
```

Không được phục hồi privilege rộng hơn baseline cũ.

---

# BE. Phase 14 — Vulnerability Register

Tạo `Vulnerabilities` sheet.

Recommended fields:

```text
FindingID
AssetID
Hostname
Environment
AssetVersion
FindingType
CVE
ScoreSource
ScoreVersion
BaseScore
Exposure
Exploitability
AssetCriticality
BusinessImpact
Severity
RemediationPriority
RemediationOwner
ExceptionOwner
TargetDate
Status
CompensatingControl
RetestEvidence
Ticket
```

Syllabus yêu cầu ít nhất affected asset/version, CVE nếu applicable, scoring source/version, exposure, business impact, priority, owner và retest evidence. Infrastructure_Operations_14_Da…

---

# BF. Map Vulnerability → Asset

Raw finding có thể:

```text
Hostname:
web01
```

Asset inventory có:

```text
AssetID:
AST-00017

Hostname:
web01

Environment:
Production

Criticality:
Critical

BusinessService:
Customer Portal
```

Reconciliation:

```text
Finding
→ Host
→ AssetID
→ Business Context
```

Nếu asset match không unique:

```text
do not guess
```

Flag:

```text
AmbiguousAssetMatch
```

---

# BG. Không dùng CVSS làm Priority duy nhất

Ví dụ:

```text
Finding A:
CVSS 9.x
Isolated lab
No sensitive dependency

Finding B:
Lower technical score
Internet-facing production authentication
Critical business service
```

Không tự động nói:

```text
A phải luôn fix trước B
```

Priority phải theo approved organization risk matrix.

Day 13 yêu cầu kết hợp:

```text
Exploitability
Exposure
Asset Criticality
Business Impact
```

---

# BH. Vulnerability Exception

Nếu chưa thể remediate:

```text
FindingID:
VUL-001

Status:
RiskException

Reason:
Vendor compatibility dependency

ExceptionOwner:
ApplicationOwner

Expiry:
<approved date>

CompensatingControl:
Restricted network exposure

Retest:
required
```

Không:

```text
Accepted → forget forever
```

---

# BI. Retest Evidence

Sau remediation:

```text
Before:
Finding present

Change:
approved patch/configuration

After:
Finding absent or condition corrected
```

Retest evidence có thể là:

```text
scanner result
configuration output
version evidence
```

phù hợp với finding.

---

# BJ. Phase 15 — Automation Architecture

Automation tốt nên chia thành các stage:

```text
load_config()
      ↓
validate_sources()
      ↓
hash_sources()
      ↓
read_raw()
      ↓
validate_schema()
      ↓
normalize()
      ↓
detect_duplicates()
      ↓
reconcile()
      ↓
classify_exceptions()
      ↓
validate_counts()
      ↓
generate_reports()
      ↓
write_runlog()
```

Không nên viết một script 1,000 lines trong một function.

---

# BK. Python — mẫu normalization function

Ví dụ training:

```python
def normalize_text(value):
    if value is None:
        return ""
    return str(value).strip()


def normalize_case_insensitive(value):
    return normalize_text(value).upper()


def build_account_key(system, environment, account_id):
    system_n = normalize_case_insensitive(system)
    env_n = normalize_case_insensitive(environment)
    account_n = normalize_text(account_id)

    if not system_n or not env_n or not account_n:
        raise ValueError("Composite key field is missing")

    return f"{system_n}|{env_n}|{account_n}"
```

Điểm đáng chú ý:

```text
account_id
```

được giữ dưới dạng string.

Không:

```python
int(account_id)
```

vì có thể làm:

```text
001234 → 1234
```

---

# BL. Python — explicit timestamp parsing

Concept:

```python
from datetime import datetime, timezone

AS_OF = datetime(2026, 10, 1, 0, 0, tzinfo=timezone.utc)
```

Không nên:

```python
datetime.now()
```

cho business cutoff nếu review contract đã declare fixed `AsOfTime`.

Nếu script dùng current system clock, rerun ngày hôm sau có thể ra classification khác dù input không đổi.

---

# BM. Python — unknown last use

Không viết:

```python
if not last_use:
    dormant = True
```

Mà:

```python
if not last_use:
    last_use_state = "UNKNOWN"
    dormant = None
```

Sau đó reviewer/business rule xử lý `UNKNOWN`.

---

# BN. Python — boundary logic

Ví dụ policy:

```python
if last_use <= cutoff:
    classification = "DORMANT"
else:
    classification = "ACTIVE_WITHIN_THRESHOLD"
```

Test bắt buộc phải có:

```text
cutoff - 1 second
cutoff exactly
cutoff + 1 second
```

---

# BO. Python — reject invalid records

Pattern:

```python
required = ["system", "environment", "account_id"]

missing = [field for field in required if not row.get(field)]

if missing:
    rejected.append({
        "source_row": row_number,
        "reason": f"Missing required fields: {','.join(missing)}"
    })
    continue
```

Không:

```python
except:
    pass
```

`except: pass` có thể gây silent data loss.

---

# BP. Python — logging

Ví dụ:

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)

logging.info("Starting reconciliation run")
logging.warning("Missing owner for record %s", record_id)
logging.error("Rejected row %s: malformed timestamp", row_id)
```

Không log:

```text
password
token
secret key
```

---

# BQ. Run ID

Mỗi run có identifier:

```text
RUN-20261001-001
```

RunLog:

```text
RunID
AutomationVersion
StartTime
EndTime
AsOf
Cutoff
SourceHashes
SourceCounts
MatchedCount
UnmatchedCount
DuplicateCount
RejectedCount
ExceptionCount
ExitStatus
```

---

# BR. Excel formula cho Duplicate

Ví dụ:

```excel
=IF(COUNTIF(Reconciliation[CompositeKey],[@CompositeKey])>1,
    "DUPLICATE",
    "")
```

Nhưng Excel formula này chỉ useful nếu:

```text
CompositeKey definition
```

đã đúng.

Automation không thể sửa một business key sai.

---

# BS. Lookup

Nếu Excel build hỗ trợ lookup hiện đại, có thể sử dụng lookup function tương ứng; nếu không, dùng approved alternative của environment.

Quan trọng hơn exact function là:

```text
lookup key phải documented
duplicate phải detect trước
not-found phải explicit
```

Một lookup trả first matching value khi duplicate tồn tại có thể che giấu data problem.

---

# BT. VBA trong Day 13 nên làm gì?

VBA có thể hỗ trợ:

```text
Refresh reconciliation
Validate mandatory sheets
Validate Control parameters
Generate exception report
Write RunLog
Protect raw sheets from accidental edits
```

Nhưng VBA không nên tự động:

```text
delete source
revoke Production access
send report externally
```

trừ khi có approved workflow riêng, vốn ngoài scope mặc định Day 13.

---

# BU. Mẫu VBA validation đơn giản

Ví dụ conceptual:

```vb
Sub ValidateRequiredSheets()

    Dim requiredSheets As Variant
    requiredSheets = Array( _
        "Control", _
        "IdentityMaster", _
        "IAM_Raw", _
        "Privileged_Raw", _
        "ServiceNow_Raw", _
        "Asset_Raw", _
        "Reconciliation", _
        "Exceptions", _
        "Decisions", _
        "Vulnerabilities", _
        "TestResults", _
        "RunLog" _
    )

    ' Validate each required worksheet exists.
    ' Record failure instead of silently continuing.

End Sub
```

Trong lab đầy đủ, mỗi failure phải được surfaced vào `RunLog/TestResults`.

---

# BV. Phase 16 — Automation Testing

Day 13 không được coi automation là hoàn thành chỉ vì script chạy.

Syllabus minimum test catalog yêu cầu:

- identity matching;
- leading zeros;
- long IDs;
- whitespace/case;
- duplicates;
- cross-system ID collision;
- missing owner;
- file/encoding failures;
- missing/locked file;
- missing header;
- IAM boundary cases;
- new account;
- unknown last-use;
- service/break-glass;
- stale approval;
- repeatability/regression.

Infrastructure_Operations_14_Da…

---

# BW. TestResults schema

```text
TestCaseID
Requirement
FixtureVersion
TestType
Input
ExpectedResult
ActualResult
PassFail
DefectID
RetestStatus
RunID
EvidenceRef
```

Không:

```text
Tested = Yes
```

---

# BX. Core Test Catalog cho Day 13

| ID   | Test                            | Expected concept                      |
| ---- | ------------------------------- | ------------------------------------- |
| T001 | Valid identity/account/approval | Matched, no exception                 |
| T002 | Leading-zero ID                 | ID preserved                          |
| T003 | Long ID                         | No numeric truncation                 |
| T004 | Leading/trailing spaces         | Normalize per rule                    |
| T005 | Case difference                 | Match only if field case-insensitive  |
| T006 | Duplicate composite key         | Explicit duplicate                    |
| T007 | Cross-system same AccountID     | No false cross-system match           |
| T008 | Missing owner                   | MissingOwner                          |
| T009 | Missing approval                | MissingApproval                       |
| T010 | Before cutoff                   | Expected policy boundary              |
| T011 | Exactly cutoff                  | Expected inclusive/exclusive behavior |
| T012 | After cutoff                    | Expected policy boundary              |
| T013 | New account                     | No naïve dormant classification       |
| T014 | Blank last-use                  | Unknown                               |
| T015 | Service account                 | Apply service-account rule            |
| T016 | Break-glass                     | Apply special control rule            |
| T017 | Stale approval                  | Exception generated                   |
| T018 | Revoked-but-active              | Exception generated                   |
| T019 | Malformed date                  | Rejected/quarantined                  |
| T020 | Missing required column         | Explicit failure                      |
| T021 | Duplicate source import         | Detected                              |
| T022 | Same run twice                  | Same business result                  |
| T023 | Existing reviewer decision      | Not overwritten                       |
| T024 | Existing remediation task       | No duplicate task                     |

---

# BY. Positive test

Input:

```text
Identity = valid
Account = unique
Owner = valid
Approval = current
Entitlement = correct
```

Expected:

```text
PopulationStatus = Matched
Exception = None
```

---

# BZ. Negative test

Input:

```text
Owner = blank
```

Expected:

```text
Exception = MissingOwner
```

Không được:

```text
Script crashes
```

hoặc:

```text
Row silently disappears
```

---

# CA. Boundary test

Ba fixtures:

```text
Cutoff - 1 sec
Cutoff
Cutoff + 1 sec
```

Acceptance:

```text
exactly matches policy/answer key
```

---

# CB. Malformed-data test

Ví dụ:

```text
LastUse = "banana"
```

Không được auto-coerce thành blank rồi nói:

```text
Unknown
```

tốt hơn:

```text
Rejected / InvalidTimestamp
```

vì blank evidence và malformed evidence không giống nhau.

---

# CC. Missing Header test

Expected input:

```text
AccountID
```

nhưng source thiếu column.

Behavior tốt:

```text
Run fails validation
or
affected source quarantined
```

với explicit error.

Không generate misleading clean report.

---

# CD. Locked File test

Nếu working input bị locked:

```text
InputAccessError
```

phải observable.

Không dùng stale previous copy mà không báo.

---

# CE. Empty File test

`0 records` có thể legitimate hoặc extraction failure.

Automation không được mặc định:

```text
No accounts = No exceptions
```

Cần kiểm:

```text
Was zero expected?
Source health?
Export completed?
```

---

# CF. File Encoding tests

Program-wide automation contract yêu cầu xử lý:

```text
UTF-8
UTF-8 BOM
UTF-16
CP932 fixture
quoted delimiter
embedded newline
CRLF/LF
malformed bytes
```

Infrastructure_Operations_14_Da…

Trong Day 13 mục tiêu không phải trở thành encoding expert, mà bảo đảm identity không bị corrupt.

Ví dụ:

```text
Nguyễn Văn A
```

không được biến thành mojibake rồi lookup fail.

---

# CG. Cross-system collision test

Input:

```text
AWS|PROD|admin01
WINDOWS|PROD|admin01
```

Expected:

```text
two distinct accounts
```

nếu key bao gồm System.

Nếu join chỉ bằng:

```text
admin01
```

test phải fail.

---

# CH. Repeatability test

Run cùng input hai lần.

Expected:

```text
RunID differs
Timestamp differs

BUT:

MatchedCount same
UnmatchedCount same
DuplicateCount same
RejectedCount same
Exceptions same
Business decisions preserved
```

Syllabus yêu cầu repeat runs tạo identical business results sau khi loại bỏ run timestamp/ID. Infrastructure_Operations_14_Da…

---

# CI. Idempotency test

Giả sử:

```text
EX-001
```

đã map tới:

```text
TASK-1001
```

Rerun không được tạo:

```text
TASK-1002
```

cho cùng logical exception trừ khi workflow intentionally yêu cầu.

Có thể dùng stable exception key:

```text
System|AccountID|ExceptionType|ReviewID
```

để detect existing task.

---

# CJ. Reviewer Decision preservation test

Run 1:

```text
EX-001
Decision = Investigate
Rationale = Owner dependency pending
```

Run 2:

Automation regenerate technical data.

Expected:

```text
Decision remains Investigate
Rationale remains intact
```

Không:

```text
Decision blank
```

hoặc:

```text
script guesses Revoke
```

---

# CK. Modified-input regression test

Run baseline:

```text
Fixture v1
```

Sau bug fix:

```text
Fixture v1 again
```

Known-good cases vẫn phải pass.

Sau đó:

```text
Fixture v2
```

với controlled change.

Chỉ expected affected result được thay đổi.

---

# CL. Golden Dataset

Test suite phải có known expected output.

Ví dụ:

```text
fixture_accounts.csv
fixture_approvals.csv
expected_exceptions.csv
```

Không tạo expected result chỉ bằng chính script đang test.

Syllabus yêu cầu trusted expected-output dataset hoặc independently reviewed assertions. Infrastructure_Operations_14_Da…

---

# CM. Phase 17 — Result Comparison

Không binary compare toàn file vì:

```text
RunID
Timestamp
File metadata
```

có thể khác.

Hãy compare business columns:

```text
CompositeKey
PopulationStatus
ExceptionType
DecisionLink
```

Exclude:

```text
RunID
ExecutionTimestamp
```

khi test reproducibility.

---

# CN. Phase 18 — Error Logging

Một useful error record:

```text
Timestamp:
2026-10-01T10:05:12Z

RunID:
RUN-001

Stage:
SOURCE_VALIDATION

Record:
IAM_Raw row 112

Error:
Invalid timestamp format

SensitiveData:
None
```

Không log toàn row nếu row chứa secret ngoài approved scope.

---

# CO. Failure không được biến thành success

Ví dụ script:

```text
Processed 299 of 305 rows
Exit code = 0
```

nhưng 6 rows biến mất.

Đây **không phải success**.

Correct result:

```text
Processed: 299
Rejected: 6
SourceTotal: 305
CountReconciliation: PASS
```

hoặc:

```text
UnexplainedLoss: 6
OverallStatus: FAIL
```

---

# CP. Phase 19 — Audit Traceability Test

Chọn ngẫu nhiên một decision:

```text
DEC-0012
```

Bạn phải trace:

```text
DEC-0012
   ↓
EX-0049
   ↓
REC-000219
   ↓
IAM_Raw Row 142
   ↓
AWS_IAM.csv
   ↓
SHA-256
```

và song song:

```text
DEC-0012
   ↓
REQ-001889
   ↓
ServiceNow_Raw Row 37
   ↓
ServiceNow.csv
```

Nếu reviewer không thể trace:

```text
decision
→ evidence
```

audit package chưa đạt.

Syllabus acceptance yêu cầu mọi decision phải trace về source row và ticket. Infrastructure_Operations_14_Da…

---

# CQ. Evidence Manifest

Suggested columns:

```text
EvidenceID
SourceSystem
ProductVersion
Environment
AssetOrAccountID
TicketReference
Collector
CollectionTimestamp
Timezone
RequestedWindow
FilterOrCommand
ExportFormat
Encoding
FilePath
FileSize
RecordCount
SHA256
Classification
ParentEvidenceID
TransformationVersion
Limitations
Errors
```

Đây gần như trực tiếp từ common evidence contract của syllabus. Infrastructure_Operations_14_Da…

---

# CR. Parent/Derived lineage

Ví dụ:

```text
EV-001
AWS_IAM.csv
RAW
```

derived:

```text
EV-011
normalized_iam.csv
Parent = EV-001
ScriptVersion = 1.2.0
```

report:

```text
EV-021
IAM_Review_Report.xlsx
Parents = EV-011 + EV-014 + EV-018
```

Điều này giúp chứng minh transformation lineage.

---

# CS. Screenshot chỉ là supplemental evidence

Nếu source export được:

```text
prefer structured/native export
```

Screenshot có thể bổ sung:

```text
UI state
configuration view
reviewer context
```

nhưng không nên thay toàn bộ export mà không giải thích.

Syllabus cũng nói screenshots supplement exportable records thay vì mặc định thay thế chúng. Infrastructure_Operations_14_Da…

---

# CT. Phase 20 — Final 90-Day IAM Review Report

Executive/report summary nên có:

```text
Review ID
Review Scope
As-of
Cutoff
Timezone
Source Systems
Source Counts
Population Counts
Exceptions by Category
Reviewer Decisions
Open Investigations
Remediation Status
Evidence Gaps
Automation Version
Test Status
```

Ví dụ:

```text
Source IAM Records:       305

Matched:                  275
Unmatched:                 14
Duplicate:                 10
Rejected:                   6

IAM Exceptions:            23

Keep:                       8
Revoke:                     4
Modify:                     3
Investigate:                8
```

Nhưng số liệu chỉ có ý nghĩa nếu counting rule rõ.

---

# CU. Report phải nói về Evidence Gaps

Ví dụ:

```text
Evidence Gap EG-001

Source:
Privileged Access

Issue:
Last-use field unavailable for historical period.

Impact:
Dormancy cannot be established for 12 accounts.

Decision:
Classified as UnknownLastUse and sent for owner review.
```

Không được viết:

```text
12 accounts inactive
```

nếu source không chứng minh được.

---

# CV. Vulnerability Summary trong final package

Ví dụ:

```text
Total Findings:        35
Critical:                2
High:                   10
Medium:                 18
Low:                     5

Open Remediation:       17
Approved Exceptions:     3
Retest Pending:           5
Closed:                  10
```

Nhưng `Severity` và `RemediationPriority` nên được report riêng.

Ví dụ:

```text
High severity / Moderate priority
```

vẫn có thể tồn tại.

---

# CW. Failure Injection 1 — Lost Leading Zero

Instructor fixture:

```text
IdentityMaster:
001234
```

Working import vô tình thành:

```text
1234
```

### Symptom

```text
Unmatched identity
Potential orphan
```

### Troubleshooting

Kiểm:

```text
Raw ID
Working ID
Excel data type
Normalization rule
```

### Root cause

```text
Identifier incorrectly converted to numeric
```

### Corrective action

Re-import source as text.

Không manually edit:

```text
1234 → 001234
```

mà không có documented source evidence.

---

# CX. Failure Injection 2 — Wrong Cutoff Comparator

Code:

```python
last_use < cutoff
```

Policy:

```text
last_use <= cutoff
```

### Symptom

Account exactly on cutoff cho kết quả sai.

### Detection

Boundary test `T011`.

### Corrective action

Fix comparator.

### Regression

Chạy lại:

```text
before
on
after
```

và full regression.

---

# CY. Failure Injection 3 — Duplicate Key Explosion

Remove `System` khỏi composite key.

Old:

```text
System|Environment|AccountID
```

Fault:

```text
Environment|AccountID
```

### Symptom

Cross-system accounts collide.

### Detection

Duplicate spike.

### Troubleshooting

Compare key definition giữa:

```text
Control
Code
Workbook
```

### Corrective action

Restore documented key version.

---

# CZ. Failure Injection 4 — ServiceNow Revoke nhưng Access Active

Fixture:

```text
ServiceNow = Revoked
Target = Active
```

### Expected

```text
RevokedButActive
```

### Wrong action

```text
Trust ticket and mark clean
```

### Correct action

```text
Capture evidence
Investigate/proceed via approved remediation
Recheck effective access
```

---

# DA. Failure Injection 5 — Empty Last-Use Field

Fixture:

```text
LastUse = blank
```

### Faulty code

```python
if not last_use:
    dormant = True
```

### Expected test result

```text
FAIL
```

### Correct result

```text
UnknownLastUse
```

---

# DB. Failure Injection 6 — Rerun destroys Decisions

Automation writes:

```text
Decisions sheet = new blank table
```

### Symptom

Reviewer decisions disappear.

### Severity

Operationally serious because human governance record was overwritten.

### Correct design

Separate:

```text
machine-generated records
```

from:

```text
reviewer-maintained records
```

and merge using stable IDs.

---

# DC. Troubleshooting flow tổng thể

Khi kết quả bất thường, đừng bắt đầu bằng sửa formula.

Dùng sequence:

```text
1. Verify Review Contract
      │
2. Verify As-of / Cutoff / Timezone
      │
3. Verify Raw Source Hash
      │
4. Verify Source Count
      │
5. Verify Schema
      │
6. Verify ID Type / Encoding
      │
7. Verify Normalization
      │
8. Verify Composite Key
      │
9. Verify Uniqueness / Cardinality
      │
10. Verify Join
      │
11. Verify Classification
      │
12. Verify Decisions
      │
13. Verify Remediation
```

---

# DD. Troubleshooting — Unmatched count tăng đột ngột

Ví dụ:

```text
Previous:
12 unmatched

Current:
312 unmatched
```

Đừng nói ngay:

```text
300 orphaned accounts
```

Check:

```text
IdentityMaster row count
Source export completeness
Leading zeros
Column mapping
System/environment normalization
Composite key version
Encoding corruption
```

---

# DE. Troubleshooting — Duplicate count tăng đột ngột

Check:

```text
Source imported twice?
Header treated as row?
Composite key field removed?
System name normalized incorrectly?
Same file concatenated twice?
Join changed to many-to-many?
```

---

# DF. Troubleshooting — Dormant count tăng đột ngột

Check:

```text
As-of changed?
Timezone changed?
Cutoff changed?
Comparator changed?
Last-use source unavailable?
Blank converted to old date?
Timestamp parsing failed?
```

---

# DG. Troubleshooting — Missing Approval hàng loạt

Có thể root cause:

```text
Actual unauthorized access
```

nhưng cũng có thể:

```text
ServiceNow export truncated
Ticket key changed
Old tickets outside retention
Wrong join field
Wrong environment
```

Do đó trước mass revoke phải verify source completeness.

---

# DH. Troubleshooting — Source count mismatch

Raw:

```text
305
```

Processed:

```text
299
```

Checklist:

```text
6 blank lines?
6 malformed?
6 filtered?
Header count issue?
Encoding issue?
Parser skipped records?
Quoted newline?
```

Nếu không giải thích được:

```text
FAIL acceptance
```

---

# DI. Troubleshooting — Python succeeds nhưng output sai

`exit code 0` chỉ nói process không báo fatal error.

Không nói business logic đúng.

Kiểm:

```text
Expected vs actual
Boundary fixtures
Golden dataset
Count reconciliation
Exception totals
```

---

# DJ. Troubleshooting — Workbook và Python khác nhau

Ví dụ:

```text
Excel = 22 exceptions
Python = 23
```

Không chọn tool “mình tin hơn”.

Trace record difference.

Compare:

```text
Normalization
Whitespace handling
Case handling
Null handling
Date parser
Boundary comparator
Duplicate rule
```

Mục tiêu cuối cùng là deterministic rule, không phải ép numbers giống nhau bằng tay.

---

# DK. Cleanup / Rollback

Lab cleanup phải đảm bảo:

```text
Raw evidence preserved
Sandbox changes restored if required
Temporary test accounts removed according to runbook
Temporary test entitlements removed
Temporary output clearly marked
Secrets absent
Final evidence hashes recorded
```

Không xóa:

```text
Raw source
RunLog
TestResults
Decision evidence
```

chỉ để folder “đẹp”.

---

# DL. Rollback của automation

Trước khi sửa script:

```text
reconcile_v1.0.py
```

giữ:

```text
reconcile_v1.0.py
reconcile_v1.1.py
```

hoặc dùng approved version-control workflow.

Không overwrite known-good copy rồi mất ability reproduce old report.

Day 12/13 yêu cầu rollback-ready automation và version traceability. Infrastructure_Operations_14_Da…

---

# DM. Evidence to Collect theo từng Phase

| Phase             | Evidence                       |
| ----------------- | ------------------------------ |
| Review setup      | Scope, policy, as-of, cutoff   |
| Source collection | Filename, source, timestamp    |
| Integrity         | SHA-256, size                  |
| Completeness      | Record counts                  |
| Import            | Encoding/data type notes       |
| Normalization     | Rule version                   |
| Reconciliation    | Source/target row refs         |
| Exception         | Exception ID + evidence        |
| Decision          | Reviewer + rationale + ticket  |
| Remediation       | Before/after effective access  |
| Vulnerability     | Asset mapping + classification |
| Automation        | Source version + run ID        |
| Test              | Expected/actual/pass/fail      |
| Final             | Report + manifest + gaps       |

---

# DN. Operational Acceptance Criteria

Không phải:

```text
"Script ran successfully."
```

Mà là toàn bộ:

### Data integrity

```text
Raw source unchanged.
Hashes recorded.
Counts reconcile.
```

### Reconciliation

```text
No silent many-to-many joins.
No accidental numeric conversion.
```

### IAM

```text
Boundary cases correct.
Blank last-use remains unknown.
Special account types handled appropriately.
```

### Decisions

```text
Every exception receives traceable decision/status.
Reviewer decisions preserved.
```

### Automation

```text
Same input → same business result.
Rerun → no duplicate side effects.
```

### Evidence

```text
Decision → Exception → Reconciliation → Raw row → Source file
```

must work.

Syllabus defines exactly these acceptance expectations. Infrastructure_Operations_14_Da…

---

# DO. Một record hoàn chỉnh nên trace như thế nào?

Giả sử:

```text
Account:
prod-admin-01
```

Source:

```text
IAM_Raw Row 109
```

Reconciliation:

```text
REC-109
```

Exception:

```text
EX-033
MissingApproval
```

Decision:

```text
DEC-033
Investigate
```

Ticket:

```text
TASK-5011
```

Evidence:

```text
EV-IAM-001
EV-SNOW-008
```

Lineage:

```text
prod-admin-01
      │
      ▼
 IAM_Raw 109
      │
      ▼
    REC-109
      │
      ▼
     EX-033
      │
      ▼
    DEC-033
      │
      ▼
   TASK-5011
```

Reviewer phải đi xuôi và ngược được.

---

# DP. Example — Full IAM Case

### Raw identity

```text
IdentityID: 000371
Status: Active
Role: ApplicationSupport
```

### Actual access

```text
System: AWS
Environment: PROD
AccountID: prod-app-371
Entitlement: Administrator
```

### Approval

```text
RequestedEntitlement: ReadOnlySupport
Approval: Approved
```

### Result

```text
IdentityMatch: Yes
OwnerMatch: Yes
ApprovalMatch: Partial / Mismatch
```

### Exception

```text
ExcessivePrivilege
```

### Decision

Có thể:

```text
Modify
```

### Remediation

```text
Administrator
→
ReadOnlySupport
```

### Verification

```text
effective entitlement reread
```

### Evidence

```text
before state
ticket
after state
run ID
reviewer
```

Đây là một end-to-end Day 13 case.

---

# DQ. Example — Full Service Account Case

```text
Account:
svc-report-prod

Type:
Service

Owner:
Business Intelligence Operations

LastInteractiveLogin:
blank

Technical activity:
Observed

Approval:
Valid

Privilege:
Database Read
```

Naive rule:

```text
Blank login → dormant
```

sai.

Correct result:

```text
No human-login dormancy conclusion.
Service ownership valid.
Business purpose valid.
Entitlement appropriate.
Keep.
```

Nếu `Owner=blank`:

```text
MissingOwner
```

dù service active.

---

# DR. Example — Full Unknown Evidence Case

```text
Account:
admin-old

LastUse:
blank

Source documentation:
Last-use field unavailable for this historical account

Owner:
valid

Approval:
valid
```

Correct:

```text
LastUseEvidence = Unknown
```

không:

```text
Dormant
```

Decision có thể:

```text
Investigate
```

nếu privilege cao.

Evidence gap phải được recorded.

---

# DS. Example — Full Vulnerability Case

```text
Finding:
VUL-007

Asset:
WEB-PROD-01

AssetVersion:
Version X

CVE:
Applicable

Technical Severity:
High

Exposure:
Internet-facing

Criticality:
Critical

Business Impact:
Customer authentication

Remediation Owner:
Platform Team
```

Priority không được copy mù từ CVSS.

Phải đi qua organization risk policy.

Sau remediation:

```text
RetestEvidence:
EV-VUL-007-RETEST
```

---

# DT. Escalation Boundary

L1/L2 có thể tự xử lý trong lab/runbook:

```text
Read-only export verification
Hash/count verification
Known reconciliation issues
Known fixture classification
Standard data-quality correction in working copy
Approved automation rerun
```

Nên escalate khi gặp:

```text
Unknown privileged account owner
Possible unauthorized Production privilege
Revoked-but-active critical account
Break-glass control failure
Service account dependency unknown
Mass incorrect classification
Evidence source incomplete
Potential compromise indicators
Automation bug impacting many accounts
Critical vulnerability outside standard remediation
```

---

# DU. Không được dùng automation để auto-revoke hàng loạt trong Day 13

Nếu script phát hiện:

```text
300 potentially stale accounts
```

correct workflow:

```text
Detect
→ Validate
→ Review
→ Approve
→ Dry-run
→ Remediate authorized subset
→ Verify
```

Không:

```text
detect → disable 300
```

Đây là nguyên tắc least-risk và change control.

---

# DV. Day 13 Practical Completion Checklist

Bạn chỉ nên coi Module Thực hành hoàn thành khi:

### Review definition

- [ ] Review scope documented
- [ ] As-of time documented
- [ ] Timezone documented
- [ ] Review start/end documented
- [ ] Inactivity threshold documented
- [ ] Inclusive/exclusive cutoff rule documented

### Sources

- [ ] Identity source collected
- [ ] IAM source collected
- [ ] Privileged source collected
- [ ] ServiceNow source collected
- [ ] Asset source collected
- [ ] Vulnerability source collected

### Evidence

- [ ] Raw files untouched
- [ ] SHA-256 captured
- [ ] Record counts captured
- [ ] File size captured
- [ ] Source/version recorded
- [ ] Evidence manifest created

### Workbook

- [ ] `Control`
- [ ] `IdentityMaster`
- [ ] `IAM_Raw`
- [ ] `Privileged_Raw`
- [ ] `ServiceNow_Raw`
- [ ] `Asset_Raw`
- [ ] `Reconciliation`
- [ ] `Exceptions`
- [ ] `Decisions`
- [ ] `Vulnerabilities`
- [ ] `TestResults`
- [ ] `RunLog`

### Data processing

- [ ] IDs imported as text
- [ ] Leading zeros preserved
- [ ] Raw values preserved
- [ ] Normalization rules documented
- [ ] Composite key documented
- [ ] Duplicate detection implemented
- [ ] Many-to-many join protected

### IAM cases

- [ ] Missing account
- [ ] Duplicate
- [ ] Candidate orphan
- [ ] Dormant
- [ ] Stale entitlement
- [ ] Missing owner
- [ ] Missing approval
- [ ] Stale approval
- [ ] Unknown last use
- [ ] New account
- [ ] Service account
- [ ] Break-glass
- [ ] Revoked-but-active
- [ ] Role change
- [ ] Excessive privilege

### Decisions

- [ ] `Keep`
- [ ] `Revoke`
- [ ] `Modify`
- [ ] `Investigate`
- [ ] Rationale
- [ ] Owner
- [ ] Reviewer
- [ ] Due date
- [ ] Evidence
- [ ] Linked task

### Vulnerabilities

- [ ] Asset mapping
- [ ] Version
- [ ] CVE where applicable
- [ ] Scoring source/version
- [ ] Exposure
- [ ] Business impact
- [ ] Severity
- [ ] Priority
- [ ] Remediation owner
- [ ] Exception owner
- [ ] Retest evidence

### Automation

- [ ] Version identified
- [ ] Run ID generated
- [ ] Input hashes captured
- [ ] Counts captured
- [ ] Errors logged
- [ ] Raw sources never overwritten

### Testing

- [ ] Positive
- [ ] Negative
- [ ] Boundary
- [ ] New account
- [ ] Unknown last-use
- [ ] Service account
- [ ] Break-glass
- [ ] Duplicate
- [ ] Cross-system collision
- [ ] Malformed input
- [ ] Incomplete input
- [ ] Regression
- [ ] Repeatability
- [ ] Idempotency
- [ ] Reviewer decision preservation

### Final acceptance

- [ ] Source totals reconcile
- [ ] Seeded exceptions match answer key
- [ ] Boundary cases match policy
- [ ] Every decision traces to source
- [ ] Every decision traces to ticket where applicable
- [ ] No unexplained dropped records
- [ ] Rerun gives same business result
- [ ] No duplicate remediation task
- [ ] No reviewer decision overwritten
- [ ] Evidence gaps explicitly documented

Đây chính là completion standard mà syllabus đặt ra cho Day 13. Infrastructure_Operations_14_Da…

---

# DW. Cheat Sheet — Practical Day 13

```text
FIRST:
Policy → Scope → As-of → Cutoff → Timezone

THEN:
Source → Hash → Count → Preserve

NEVER:
Edit raw evidence directly
Join on display name alone
Convert identity IDs to numbers
Treat blank last-use as inactive
Trust ticket state as proof of effective access
Trust CVSS as final business priority
Silently discard bad records
Overwrite reviewer decisions

KEY:
System + Environment + AccountID

POPULATION:
Matched
Unmatched
Duplicate
Rejected

IAM EXCEPTIONS:
MissingOwner
MissingApproval
StaleApproval
Dormant
StaleEntitlement
CandidateOrphan
Duplicate
RoleMismatch
ExcessivePrivilege
UnknownLastUse
RevokedButActive

DECISIONS:
Keep
Revoke
Modify
Investigate

TEST:
Positive
Negative
Boundary
Malformed
Incomplete
Duplicate
Inconsistent
Regression
Repeatability
Idempotency

EVIDENCE:
Source
Timestamp
Timezone
Count
SHA-256
RunID
ScriptVersion
Ticket
SourceRow
Decision
```

---

# Kết luận Module Thực hành Day 13

Điểm khó nhất của Day 13 không phải viết một công thức Excel hay một đoạn Python. Điểm khó là bảo đảm toàn bộ chuỗi **source → transformation → finding → decision → remediation → verification** không bị đứt ở bất kỳ đâu.

Một kết quả như:

```text
23 privileged accounts need remediation
```

chưa có nhiều giá trị nếu bạn không chứng minh được:

```text
23 đến từ population nào?
Rule nào tạo ra 23?
Cutoff nào?
Timezone nào?
Source nào?
Có record nào bị drop không?
Composite key nào?
Có duplicate không?
Blank last-use xử lý thế nào?
Reviewer nào quyết định?
Ticket nào?
Sau revoke effective access còn hay không?
```

Day 13 vì thế đặt ra bốn chuẩn đồng thời:

```text
Correctness
+
Completeness
+
Repeatability
+
Traceability
```

Raw evidence phải giữ nguyên; mỗi transformation phải explainable; seeded boundary cases phải đúng; unknown phải được giữ là unknown; source totals phải reconcile; reviewer decisions không được automation ghi đè; rerun không tạo duplicate side effects; và mọi kết luận quan trọng phải lần ngược được về source evidence. Đây chính là điều phân biệt một **ad-hoc spreadsheet exercise** với một **enterprise-grade IAM certification workflow**. Infrastructure_Operations_14_Da…

Với phần này, **toàn bộ Module Thực hành Day 13 theo syllabus đã được cover**, gồm 90-day IAM review, AWS/privileged access/ServiceNow/asset reconciliation, workbook 12 sheets, composite keys, exception classification, defensible reviewer decisions, sandbox remediation và effective-access verification, vulnerability register, VBA/Python automation, boundary/negative/regression/idempotency testing, hashes/counts, RunLog và audit trail. Infrastructure_Operations_14_Da…
