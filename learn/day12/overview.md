# Day 12 — Operations Automation & Internal Tool Maintenance

Day 12 thuộc Module 7 — **Operations Automation, IAM Audit & Integrated Practice**. Đây là ngày chuyển trọng tâm từ vận hành từng platform riêng lẻ sang khả năng **đọc, sửa, kiểm thử và vận hành an toàn các internal operations tools** dùng Excel/VBA, Python, Windows Batch, PowerShell và Outlook/MAPI. Nội dung Day 12 cũng tạo nền trực tiếp cho **Day 13 — 90-day IAM review, multi-sheet reconciliation và automation testing**. Infrastructure_Operations_14_Da…

Theo syllabus, Day 12 không phải khóa lập trình tổng quát. Mục tiêu là đạt mức **operational automation readiness**: hiểu một script/tool hiện hữu đang làm gì, xác định dependency và risk, sửa có kiểm soát, xử lý dữ liệu chính xác, test regression, bảo vệ credential/evidence và có khả năng rollback. Infrastructure_Operations_14_Da…

Tôi sẽ chia toàn bộ Day 12 thành đúng hai module lớn như bạn yêu cầu.

---

# Module 1 — LÝ THUYẾT

## Operations Automation, Data Processing & Safe Tool Maintenance

Module này sẽ cung cấp toàn bộ nền tảng cần hiểu trước khi sửa hoặc chạy các operations tools.

### 1. Enterprise operations tool maintenance

Đây là tư duy cốt lõi của Day 12.

| Nội dung                 | Cần học                                                                           |
| ------------------------ | --------------------------------------------------------------------------------- |
| Existing script analysis | Cách tiếp cận một VBA/Python/Batch/PowerShell script chưa quen                    |
| Code reading             | Xác định entry point, input, output, function, control flow, dependency           |
| Debugging                | Phân biệt syntax error, runtime error, logic error, data error, environment error |
| Modification             | Sửa đúng phạm vi requirement, tránh thay đổi ngoài scope                          |
| Regression testing       | Chứng minh sửa lỗi A không làm hỏng behavior B                                    |
| Change control           | Có change record, before/after, test result, rollback                             |
| Dependency mapping       | File, folder, module, executable, mailbox, worksheet, environment variable        |
| Failure analysis         | Hard-coded path, missing file, bad encoding, invalid input, permission failure    |
| Rollback readiness       | Giữ version cũ và có khả năng phục hồi nhanh                                      |

Điểm quan trọng là không được coi việc “script chạy xong” đồng nghĩa với “script đúng”. Output phải được kiểm chứng với **expected result/reference dataset**. Syllabus yêu cầu mỗi tool phải có before/after code, tests và reproducible run instruction. Infrastructure_Operations_14_Da…

---

## 2. Microsoft Excel cho Infrastructure Operations

Excel ở Day 12 được sử dụng như một **data reconciliation tool**, không chỉ để nhập dữ liệu thủ công.

Bạn sẽ học:

| Chủ đề                        | Ý nghĩa operational                                  |
| ----------------------------- | ---------------------------------------------------- |
| Excel formulas                | Tính toán và kiểm tra condition                      |
| Lookup functions              | Match ID giữa các dataset                            |
| Cross-sheet reconciliation    | So sánh dữ liệu giữa nhiều worksheet                 |
| Cross-workbook reconciliation | So sánh dữ liệu giữa nhiều workbook                  |
| Duplicate detection           | Phát hiện ID/account/asset bị lặp                    |
| Missing-record identification | Phát hiện record chỉ tồn tại ở một source            |
| Conditional formatting        | Highlight exception                                  |
| Data validation               | Giới hạn giá trị nhập                                |
| Excel Tables                  | Structured data range phục vụ filtering/formulas/VBA |

Một nguyên tắc cực kỳ quan trọng của Day 12 là:

**ID phải được xử lý như text.**

Ví dụ:

```text
00001234
```

không được vô tình trở thành:

```text
1234
```

Tương tự, long identifier không được để Excel tự chuyển sang scientific notation hoặc mất precision. Syllabus đặc biệt yêu cầu bảo toàn **leading zeros, long identifiers và original values**. Infrastructure_Operations_14_Da…

---

# 3. VBA fundamentals cho operations workbook

VBA sẽ được học ở mức đủ để đọc, sửa và duy trì workbook automation.

Các thành phần chính:

| Thành phần             | Nội dung                               |
| ---------------------- | -------------------------------------- |
| Macro                  | Automation được trigger trong workbook |
| Module                 | Nơi chứa VBA code                      |
| Procedure              | `Sub` và `Function`                    |
| Variables              | Lưu state/value                        |
| Arrays                 | Xử lý tập dữ liệu                      |
| Loops                  | `For`, `For Each`, `Do`                |
| Dictionary             | Lookup/matching nhanh theo key         |
| File operations        | Đọc/ghi file                           |
| Exception handling     | `On Error` và cleanup logic            |
| Worksheet automation   | Đọc cell, range, table                 |
| Cross-sheet processing | Reconcile nhiều worksheet              |

Bạn cũng phải hiểu các lỗi thường thấy trong legacy VBA:

```text
Hard-coded path
Hard-coded worksheet name
Implicit type conversion
Silent error handling
Assuming rows are always present
Assuming IDs are numeric
Using VBA Rnd for passwords
Overwriting source data
```

Đặc biệt, syllabus cấm dùng **VBA `Rnd` hoặc predictable seed** để tạo operational password. Infrastructure_Operations_14_Da…

---

# 4. Python fundamentals cho operations automation

Python trong Day 12 tập trung vào xử lý file, logs và structured data.

Các thành phần phải học gồm:

| Python component   | Mục đích                                         |
| ------------------ | ------------------------------------------------ |
| `pathlib`          | Path/file handling an toàn hơn hard-coded string |
| `csv`              | Đọc/ghi CSV đúng quoting/delimiter               |
| `re`               | Regular expressions                              |
| `logging`          | Execution/error logging                          |
| `hashlib`          | SHA-256/integrity checking                       |
| `datetime`         | Timestamp/time processing                        |
| `subprocess`       | Chạy external process                            |
| `json`             | Structured JSON processing                       |
| File I/O           | TXT/log/CSV processing                           |
| Exception handling | Xử lý failure rõ ràng                            |
| Functions          | Viết reusable logic                              |

Ví dụ tư duy xử lý sẽ là:

```text
Input file
   ↓
Validate
   ↓
Detect/decode encoding
   ↓
Normalize fields
   ↓
Parse records
   ↓
Extract IDs
   ↓
Validate records
   ↓
Classify errors/exceptions
   ↓
Write structured output
   ↓
Count + hash + log
```

---

# 5. Windows Batch scripting

Batch không được học như ngôn ngữ lập trình phức tạp mà như một **wrapper/orchestration layer**.

Cần hiểu:

```bat
%VARIABLE%
set
if
for
call
exit /b
errorlevel
```

Cũng cần hiểu:

```text
Environment variables
Command execution
File operations
Quoting
Paths containing spaces
Exit codes
Child process failure
Scheduled execution
```

Một yêu cầu đặc biệt trong Day 12:

> Batch wrapper phải giữ nguyên child-process failure status.

Ví dụ Python fail với non-zero exit code thì Batch không được trả về success một cách sai lệch. Infrastructure_Operations_14_Da…

---

# 6. PowerShell cho Infrastructure Operations

PowerShell tập trung mạnh vào Windows operations.

Bạn cần hiểu:

| Topic                     | Ứng dụng                                             |
| ------------------------- | ---------------------------------------------------- |
| Objects                   | PowerShell pipeline truyền object chứ không chỉ text |
| Pipelines                 | Kết nối cmdlets                                      |
| `Import-Csv`              | Nhập structured data                                 |
| `Export-Csv`              | Xuất report                                          |
| File encoding             | Tránh corrupt dữ liệu                                |
| Event Logs                | Extract Windows events                               |
| Administrative automation | Health checks/reporting                              |

Một lab quan trọng sẽ là viết PowerShell để:

```text
Select Event Log
→ Limit time window
→ Filter event criteria
→ Export structured report
→ Preserve metadata
→ Record execution result
```

Day 12 tiếp nối trực tiếp kỹ năng Windows Event Log từ Day 3 nhưng chuyển sang **automation/reproducible extraction**. Infrastructure_Operations_14_Da…

---

# 7. TXT/CSV processing

Đây là một phần rất quan trọng và dễ gây lỗi production.

Bạn phải phân biệt ba khái niệm:

```text
Encoding
CSV syntax
Locale
```

Chúng không phải cùng một thứ.

### Encoding cần học

```text
UTF-8
UTF-8 BOM
UTF-16
Shift-JIS
CP932
```

Ngoài ra còn phải hiểu:

```text
CRLF
LF
delimiter
quoting
embedded comma
embedded quote
embedded newline
```

Ví dụ CSV hợp lệ:

```csv
ID,Name,Comment
0000123,"Nguyen, An","Line 1
Line 2"
```

Parser không được đơn giản dùng:

```python
line.split(",")
```

vì comma và newline có thể nằm trong quoted field.

Syllabus yêu cầu đặc biệt test **Japanese/Vietnamese text, embedded commas/quotes/newlines và malformed encoding**. Infrastructure_Operations_14_Da…

---

# 8. String normalization

Reconciliation chỉ chính xác nếu normalization rule được định nghĩa rõ.

Ví dụ:

```text
" USER001 "
"user001"
"USER001"
```

Có thể cùng đại diện cho một account, nhưng không được tự ý assume.

Cần xác định rule cho:

```text
leading/trailing whitespace
case
full-width / half-width characters
Unicode
blank values
leading zeros
special characters
```

Điểm quan trọng:

**Raw/original value phải được giữ lại.**

Normalized data chỉ nên là derived representation.

---

# 9. Regular Expressions — Regex

Regex dùng để extract và validate patterns trong logs/TXT/CSV.

Ví dụ:

```regex
^USR\d{6}$
```

có thể dùng cho ID dạng:

```text
USR123456
```

Bạn sẽ học:

```text
character classes
anchors
groups
quantifiers
escaping
matching
searching
capturing
validation
```

Nhưng quan trọng hơn syntax là **không viết Regex quá rộng dẫn đến false match**.

---

# 10. File naming và integrity

Automation phải tạo artifact có thể audit được.

Cần học:

```text
filename conventions
timestamps
source/derived distinction
record count
file size
SHA-256
run ID
script version
```

Ví dụ:

```text
IAM_RAW_20261022.csv
IAM_NORMALIZED_20261022.csv
IAM_EXCEPTION_20261022.csv
run_20261022_090000.log
```

Raw input phải được giữ nguyên và processing thực hiện trên copy. Evidence contract của syllabus yêu cầu source, timestamp/timezone, filter, format/encoding, size, count, SHA-256 và transformation/script version phải có khả năng truy vết. Infrastructure_Operations_14_Da…

---

# 11. Spreadsheet formula injection

Đây là security issue quan trọng.

Nếu CSV chứa:

```text
=cmd(...)
+...
-...
@...
```

Excel có thể coi content là formula thay vì plain data.

Vì vậy Day 12 yêu cầu **prevent spreadsheet formula injection trong presentation exports**, nhưng đồng thời không được sửa mất raw evidence. Infrastructure_Operations_14_Da…

Tức là:

```text
Raw evidence
       ↓ preserve unchanged

Presentation/report copy
       ↓ sanitize safely
```

---

# 12. Secure password generation

Phần này không nhằm tạo password thật cho production.

Bạn phải hiểu:

```text
cryptographically secure RNG
password policy
length
allowed character classes
invalid policy
generator failure
secret masking
credential lifecycle
```

Không được:

```text
dùng VBA Rnd
dùng predictable seed
ghi plaintext password vào Excel
ghi plaintext password vào CSV
ghi plaintext password vào log
gửi plaintext password qua email
```

Syllabus yêu cầu sử dụng **approved cryptographically secure source**, test bằng disposable credentials và evidence phải masked. Infrastructure_Operations_14_Da…

---

# 13. Outlook classic COM/MAPI

Day 12 bao gồm automation với **classic Outlook**, không phải mọi phiên bản Outlook.

Cần hiểu các object/concept:

```text
Outlook Application
Namespace
MAPI
Mailbox
Folder
MailItem
Attachments
Draft
```

Đặc biệt:

```text
GetNamespace("MAPI")
```

sẽ được sử dụng để truy cập mailbox/folder trong lab.

Workflow lý thuyết:

```text
Open Outlook COM object
        ↓
Get MAPI namespace
        ↓
Select approved test folder
        ↓
Read allowlisted messages
        ↓
Process safe attachments
        ↓
Handle duplicate filenames
        ↓
Prepare report
        ↓
Create draft
        ↓
Verify
        ↓
DO NOT SEND automatically
```

Syllabus ghi rõ lab yêu cầu **classic Outlook**, và **new Outlook không hỗ trợ VBA/macros** theo reference của curriculum. Infrastructure_Operations_14_Da…

---

# 14. Safe script execution

Trước khi chạy operations automation phải đánh giá:

```text
Which environment?
Which account?
Which permissions?
Which files can change?
Which external systems are contacted?
Can email be sent?
Can files be overwritten?
Can credentials appear in logs?
What happens on partial failure?
Can execution be rolled back?
```

Tư duy Day 12 là:

```text
Read
→ Understand
→ Baseline
→ Change
→ Test
→ Compare
→ Verify
→ Evidence
→ Rollback-ready
```

chứ không phải:

```text
Edit
→ Run
→ Hope
```

---

# Module 2 — THỰC HÀNH

## Repair, Extend, Test & Validate Operations Automation

Module thực hành bám trực tiếp vào Assignment/Lab và Additive Assignment của syllabus. Infrastructure_Operations_14_Da…

## 1. Lab environment và starter artifacts

Bạn sẽ cần một authorized lab có tối thiểu:

```text
Microsoft Excel + approved VBA configuration
Python
Windows Batch environment
PowerShell
classic Outlook + test mailbox
Golden datasets
Sample log/TXT/CSV files
Starter scripts
Evidence folder
```

Không sử dụng credential, mailbox hay operational data thật ngoài phạm vi được cấp phép.

---

## 2. Phân tích existing scripts

Bạn sẽ nhận các tool dạng:

```text
Excel/VBA workbook
Python parser
Batch wrapper
PowerShell script
```

Nhiệm vụ đầu tiên chưa phải sửa.

Bạn phải xác định:

```text
Input
Output
Entry point
Dependencies
Hard-coded paths
Expected environment
External commands
File encoding assumptions
Error handling
Logging
Security assumptions
Side effects
Rollback method
```

Sau đó tạo **dependency map**.

---

# 3. Repair Excel/VBA reconciliation workbook

Workbook sẽ xử lý user/asset IDs qua nhiều sheet.

Bạn phải sửa hoặc bổ sung logic để:

```text
Read source IDs
→ preserve them as text
→ normalize according to documented rules
→ match IDs across sheets
→ detect duplicates
→ detect missing records
→ classify exceptions
→ generate exception report
```

Dataset test phải bao gồm:

```text
leading-zero IDs
long IDs
duplicate composite keys
blank IDs
Japanese text
Vietnamese text
embedded commas
embedded quotes
embedded newlines
```

Đây là dataset kiểu “golden dataset” được curriculum yêu cầu cho Day 12. Infrastructure_Operations_14_Da…

---

# 4. Python CSV/TXT/log parser

Bạn sẽ sửa và mở rộng Python tool để:

```text
Read TXT/CSV/log
Validate files
Handle encoding
Normalize strings
Extract IDs using Regex
Detect malformed records
Write structured output
Generate execution log
Calculate record count
Calculate SHA-256
Return meaningful exit code
```

Tool không được silently replace malformed characters.

Ví dụ một CP932 decode failure phải tạo trạng thái kiểu:

```text
FAILED_ENCODING
```

hoặc quarantine record/file, thay vì tự biến character không đọc được thành `?`.

---

# 5. Batch wrapper

Bạn sẽ viết hoặc repair một Batch wrapper để chạy Python/tool chain.

Batch phải xử lý đúng:

```text
paths containing spaces
missing input files
child-process failures
exit codes
log file path
scheduled execution
```

Ví dụ requirement:

```text
Python exit code = 2
```

thì wrapper không được:

```text
exit /b 0
```

Syllabus yêu cầu child-process failure phải observable. Infrastructure_Operations_14_Da…

---

# 6. PowerShell Windows Event extraction

Bạn sẽ phát triển PowerShell automation để:

```text
Define time range
Select Event Log
Apply filter
Extract records
Export report
Handle encoding
Count events
Log execution
Return success/failure clearly
```

Expected evidence có thể chứa:

```text
Computer
ProviderName
Id
RecordId
LevelDisplayName
TimeCreated
Message
```

tùy phạm vi lab.

---

# 7. Encoding test lab

Bạn phải chạy cùng parser với các fixture:

```text
UTF-8
UTF-8 BOM
UTF-16
CP932
CRLF
LF
```

và các failure fixtures:

```text
malformed bytes
missing column
wrong delimiter
quoted delimiter
embedded newline
unsupported/unmappable characters
```

Minimum automation test catalog của curriculum yêu cầu malformed/unmappable data phải **reject hoặc quarantine với lý do rõ ràng**, không silently corrupt dữ liệu. Infrastructure_Operations_14_Da…

---

# 8. Positive / Negative / Boundary testing

Không chỉ test “happy path”.

Bạn phải có ít nhất các nhóm:

| Test type  | Ví dụ                |
| ---------- | -------------------- |
| Positive   | File đúng format     |
| Negative   | Missing file         |
| Negative   | Permission denied    |
| Negative   | Missing header       |
| Boundary   | Empty input          |
| Boundary   | Very long ID         |
| Boundary   | Leading-zero ID      |
| Data error | Duplicate ID         |
| Encoding   | Malformed CP932      |
| File path  | Path contains spaces |
| Process    | Child script fails   |

Syllabus yêu cầu expected result và actual result phải được record riêng. Infrastructure_Operations_14_Da…

---

# 9. Password-generation test

Trong lab, bạn sẽ implement một secure password-generation function bằng approved cryptographically secure source.

Test sẽ kiểm tra:

```text
required length
allowed character policy
invalid policy
generator failure
masked reporting
absence of plaintext secrets
```

Ví dụ evidence chỉ được hiển thị:

```text
Password generated: **************
Length: 16
Policy check: PASS
```

Không lưu actual password trong evidence package.

---

# 10. Outlook/MAPI automation lab

Trong classic Outlook test environment:

```text
GetNamespace("MAPI")
        ↓
Select test mailbox/folder
        ↓
Enumerate approved messages
        ↓
Inspect attachments
        ↓
Allowlist attachment types
        ↓
Save safely
        ↓
Handle filename collision
        ↓
Prepare evidence attachment/report
        ↓
Create draft message
        ↓
Verify recipients/content/attachments
        ↓
STOP
```

Không auto-send.

Lab cũng phải test:

```text
missing folder
missing attachment
duplicate filename
disallowed attachment path/type
unavailable Outlook profile
```

Curriculum yêu cầu kết quả cuối cùng của Outlook workflow chỉ là **verified draft**. Infrastructure_Operations_14_Da…

---

# 11. Hash và data-integrity verification

Input và output phải có kiểm tra integrity.

Ví dụ:

```text
input SHA-256
record count
output SHA-256
output record count
```

Nhưng cần hiểu:

```text
Hash matching ≠ proof data source was truthful
```

Hash chủ yếu chứng minh file không thay đổi so với baseline hash đã ghi nhận.

Raw input phải giữ nguyên. Derived file phải có hash riêng và liên kết đến parent evidence. Infrastructure_Operations_14_Da…

---

# 12. Compare output against golden dataset

Đây là bước quyết định tool có đúng hay không.

Không được test bằng cách:

```text
script A output
vs
script A output
```

Mà phải:

```text
Actual output
vs
Independently verified expected output
```

Curriculum quy định automation tests phải dựa trên trusted expected-output dataset hoặc independently reviewed assertions. Infrastructure_Operations_14_Da…

---

# 13. Regression test

Sau khi repair tool:

```text
Run old known-good cases
Run new fixed cases
Run failure cases
Run same input twice
Compare business results
```

Phải đảm bảo:

```text
không tạo duplicate side effects
không overwrite input
không thay đổi reviewer decisions
không làm hỏng cases từng PASS
```

Repeatability/regression là một phần bắt buộc trong automation acceptance contract. Infrastructure_Operations_14_Da…

---

# 14. Change record và source diff

Mỗi tool được sửa phải có:

```text
Change ID
Reason
Affected file/tool
Original version
Modified version
Dependencies
Before behavior
After behavior
Source diff
Test cases
Expected/actual result
Known limitations
Rollback method
```

Syllabus yêu cầu tạo **small source diff và rollback package** cho từng tool được sửa. Infrastructure_Operations_14_Da…

---

# 15. Rollback package

Ví dụ cấu trúc:

```text
Day12/
├── source_original/
├── source_modified/
├── test_data/
├── expected/
├── actual/
├── logs/
├── hashes/
├── diff/
└── rollback/
```

Rollback phải có thể đưa tool về trạng thái trước change mà không phụ thuộc vào việc “nhớ mình đã sửa những gì”.

---

# 16. Evidence package cuối Day 12

Theo syllabus, deliverable cuối cùng phải chứng minh được cả **coding correctness** lẫn **operational control**.

Tối thiểu sẽ gồm:

```text
Updated VBA workbook
Updated Python parser
Updated Batch wrapper
Updated PowerShell extraction script
Outlook/MAPI draft evidence

Before/after source code
Source diff
Dependency map
Change records

Golden datasets
Test cases
Expected results
Actual results
Pass/fail results
Regression results

Execution logs
Record counts
SHA-256 hashes

Encoding test evidence
Masked password-generation evidence

Rollback-ready copies
Reproducible run instructions
```

Acceptance quan trọng nhất là: reconciliation phải khớp golden answer; source input không bị thay đổi; encoding failures phải explicit; secrets phải masked; Outlook chỉ tạo draft; và mỗi language/tool phải có before/after, expected/actual tests và reproducible run instruction. Infrastructure_Operations_14_Da…

---

# Bức tranh tổng thể Day 12

Có thể hình dung Day 12 thành một pipeline duy nhất:

```text
                 DAY 12
                   │
          Existing Operations Tool
                   │
                   ▼
             Read / Analyze
                   │
       ┌───────────┼────────────┐
       ▼           ▼            ▼
     VBA         Python      Batch/PowerShell
       │           │            │
       └───────────┼────────────┘
                   ▼
             Data Processing
                   │
       CSV / TXT / Logs / Excel
                   │
        Encoding + Normalization
                   │
             Regex / Matching
                   │
                   ▼
             Modify / Repair
                   │
                   ▼
       Positive / Negative Tests
                   │
          Encoding / Boundary
                   │
          Regression Testing
                   │
                   ▼
        Compare with Golden Data
                   │
             PASS / FAIL
                   │
       ┌───────────┴────────────┐
       ▼                        ▼
   Evidence                  Rollback
       │
       ▼
 Change record + hashes + logs
```

## Kết quả mong đợi sau Day 12

Kết thúc Day 12, bạn chưa cần trở thành software developer. Bạn cần có khả năng **nhận một operations tool đang tồn tại, hiểu input/output và dependency của nó, sửa một lỗi hoặc requirement nhỏ có kiểm soát, xử lý CSV/TXT/Excel đúng encoding và ID semantics, sử dụng VBA/Python/Batch/PowerShell đúng vai trò, kiểm thử positive/negative/regression, bảo vệ secrets, tạo evidence, và rollback được thay đổi**. Đây chính là skill family “internal-tool maintenance” mà syllabus yêu cầu trước khi sang Day 13 để thực hiện IAM reconciliation và audit automation ở quy mô lớn hơn. Infrastructure_Operations_14_Da… Infrastructure_Operations_14_Da…

Khi bạn bảo **“Trình bày chi tiết Module Lý thuyết Day 12”**, tôi sẽ đi lần lượt từng phần theo format **What → Why → How → operational workflow → security → failure modes → verification → troubleshooting → evidence → rollback/escalation**, kèm code/example cụ thể. Khi bạn bảo **“Trình bày chi tiết Module Thực hành Day 12”**, tôi sẽ xây dựng toàn bộ hands-on lab từ environment, datasets, VBA/Python/Batch/PowerShell/Outlook tasks cho tới test matrix, failure injection, verification, evidence package và rollback.
