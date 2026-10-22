# Day 12 — Module Thực hành

## Operations Automation, Data Processing & Safe Internal-Tool Maintenance Lab

Phần thực hành Day 12 là nơi toàn bộ kiến thức về Excel/VBA, Python, Windows Batch, PowerShell, TXT/CSV/encoding, Regex, secure password generation và Outlook classic COM/MAPI được ghép thành một **controlled enterprise operations workflow**. Theo syllabus, bạn không chỉ viết script mới mà phải **đọc và sửa tool hiện hữu**, xác định dependency và unsafe assumption, thực hiện controlled modification, test positive/negative/encoding cases, kiểm tra hash/record count, so sánh với golden dataset, tạo evidence và chuẩn bị rollback-ready version. Infrastructure_Operations_14_Da…

Syllabus còn đặt acceptance rất rõ: reconciliation phải khớp golden answer; input gốc không thay đổi; encoding failure phải explicit; password evidence phải masked; Outlook chỉ được tạo **verified draft**, không gửi; và VBA/Python/Batch/PowerShell phải có before/after code, expected/actual tests cùng reproducible run instruction. Infrastructure_Operations_14_Da…

---

# 1. Mục tiêu tổng thể của Lab Day 12

Sau khi hoàn thành module thực hành, bạn phải có khả năng nhận một bộ internal operations tools tương tự môi trường enterprise:

```text
Excel Workbook + VBA
        │
        ▼
Raw CSV/TXT/log files
        │
        ▼
Python Parser
        │
        ▼
Batch Wrapper
        │
        ▼
PowerShell Event Extraction
        │
        ▼
Structured Reports
        │
        ▼
Excel Reconciliation
        │
        ▼
Exception Report
        │
        ▼
Outlook Draft
```

và thực hiện đầy đủ lifecycle:

```text
Understand
    ↓
Baseline
    ↓
Preserve originals
    ↓
Identify defects
    ↓
Create change record
    ↓
Modify
    ↓
Positive tests
    ↓
Negative tests
    ↓
Encoding tests
    ↓
Regression tests
    ↓
Verify counts/hashes
    ↓
Compare golden answer
    ↓
Collect evidence
    ↓
Prepare rollback
```

Đây mới là mục tiêu thật của Day 12.

---

# 2. Lab Safety Boundary

Toàn bộ lab phải được thực hiện trong:

```text
LAB / NON-PRODUCTION / AUTHORIZED ENVIRONMENT
```

Không dùng:

```text
production mailbox
production credential
real privileged password
real customer data
real HR/IAM data
production scheduled task
production administrative account
```

Sử dụng:

```text
synthetic accounts
synthetic assets
disposable credentials
test mailbox
test Windows VM
sanitized CSV/TXT/log fixtures
lab workbook
```

Syllabus quy định environment cần được chuẩn bị trước với Excel có VBA phù hợp, Python, PowerShell, Batch, classic Outlook tương thích, test mailbox và sanitized datasets. Infrastructure_Operations_14_Da…

---

# 3. Lab Topology

Một topology phù hợp:

```text
+--------------------------------------------------------+
| Windows Lab VM                                         |
|                                                        |
|  Excel + VBA                                           |
|  Python                                                |
|  cmd.exe / Batch                                       |
|  PowerShell                                            |
|  classic Outlook                                      |
|                                                        |
|  C:\Day12\                                             |
|      ├── source_original\                              |
|      ├── working\                                      |
|      ├── scripts_original\                             |
|      ├── scripts_modified\                             |
|      ├── datasets\                                     |
|      ├── expected\                                     |
|      ├── actual\                                       |
|      ├── evidence\                                     |
|      ├── logs\                                         |
|      ├── hashes\                                       |
|      ├── diff\                                         |
|      └── rollback\                                     |
+-------------------------+------------------------------+
                          |
                          |
                          ▼
                 Test Outlook Profile
                          |
                          ▼
                     Test Mailbox
```

Có thể sử dụng một Windows Server/Windows client lab VM miễn là toolchain và quyền được instructor phê duyệt.

---

# 4. Bộ dữ liệu Golden Dataset

Day 12 yêu cầu một dataset được thiết kế cố ý để chứa nhiều edge cases. Infrastructure_Operations_14_Da…

Ví dụ `Identity_A.csv`:

```csv
UserID,Name,Department
00000123,Nguyen Van An,IT
00000124,Trần Thị Mai,Finance
00000125,山田太郎,Operations
00000126,"Nguyen, Binh",Security
00000127,"Le ""Minh"" Anh",HR
00000128,"Tran
Thanh",Infrastructure
,Blank User,Unknown
00000123,Duplicate User,IT
```

`Identity_B.csv`:

```csv
UserID,Status,Owner
00000123,Active,ops-team
00000124,Active,finance-team
00000125,Disabled,operations-team
00000129,Active,security-team
00000130,Active,
```

Các fixture nên cố ý chứa:

```text
leading-zero IDs
long identifiers
duplicate IDs
duplicate composite keys
blank IDs
case differences
leading/trailing whitespace
Japanese text
Vietnamese text
embedded commas
embedded quotes
embedded newlines
UTF-8
UTF-8 BOM
UTF-16
CP932
CRLF
LF
malformed encoding
missing columns
paths containing spaces
```

Đây đúng với minimum automation testing contract của chương trình. Infrastructure_Operations_14_Da…

---

# 5. Quy tắc quan trọng: Raw Dataset không được sửa

Trước khi bắt đầu:

```text
source_original\
```

phải trở thành immutable baseline về mặt operational procedure.

Bạn không cần biến NTFS thành write-protected bắt buộc trong lab, nhưng tuyệt đối không process trực tiếp trên original nếu workflow có khả năng ghi lại file.

Flow:

```text
source_original
      │
      ├──── SHA-256
      │
      ▼
working copy
      │
      ▼
processing
```

Syllabus yêu cầu bảo tồn untouched raw export, process working copy và dùng hash/count để kiểm tra integrity/completeness. Infrastructure_Operations_14_Da…

---

# 6. Lab 0 — Establish Baseline

## Objective

Trước khi sửa bất cứ tool nào, tạo baseline để biết:

```text
Tool hiện tại là version nào?
Input là gì?
Output là gì?
Expected behavior là gì?
Existing failure là gì?
```

---

## Step 1 — Inventory Lab Environment

PowerShell read-only checks:

```powershell
$PSVersionTable
```

Purpose:

```text
Xác định PowerShell runtime.
```

Python:

```cmd
python --version
```

Kiểm tra Python executable:

```cmd
where python
```

Kiểm tra Outlook/Excel thực hiện bằng approved GUI inventory hoặc application version information.

Evidence:

```text
EV-D12-001_PowerShellVersion.txt
EV-D12-002_PythonVersion.txt
EV-D12-003_ExcelVersion.png
EV-D12-004_OutlookVersion.png
```

Không cần screenshot nếu command/export đã đủ; screenshot chỉ bổ sung.

---

# 7. Step 2 — Hash toàn bộ Starter Artifacts

PowerShell:

```powershell
Get-FileHash `
  -Path "C:\Day12\scripts_original\parser.py" `
  -Algorithm SHA256
```

Expected output:

```text
Algorithm : SHA256
Hash      : <64-character SHA-256>
Path      : C:\Day12\scripts_original\parser.py
```

Có thể hash cả folder:

```powershell
Get-ChildItem "C:\Day12\scripts_original" -File |
    Get-FileHash -Algorithm SHA256
```

Purpose:

```text
Establish byte-level baseline.
```

Nhưng phải nhớ:

> SHA-256 chỉ cho bạn biết artifact có khác baseline byte sequence hay không. Nó không chứng minh source đúng, query đầy đủ hoặc business logic chính xác.

Đây là distinction syllabus yêu cầu. Infrastructure_Operations_14_Da…

---

# 8. Step 3 — Copy sang Working Area

Ví dụ:

```powershell
Copy-Item `
  -Path "C:\Day12\scripts_original\*" `
  -Destination "C:\Day12\scripts_modified\" `
  -Recurse
```

Verify:

```powershell
Get-ChildItem "C:\Day12\scripts_modified"
```

Sau đó:

```powershell
Get-FileHash "C:\Day12\scripts_original\parser.py"
Get-FileHash "C:\Day12\scripts_modified\parser.py"
```

Trước modification, hai hash phải giống nhau.

---

# 9. Lab 1 — Analyze Existing Tools

Đây là bước bắt buộc trước repair.

Bạn được cung cấp:

```text
Reconcile.xlsm
parser.py
run_parser.bat
ExtractEvents.ps1
```

Nhiệm vụ chưa phải edit.

Nhiệm vụ là lập:

```text
Dependency Map
Input/Output Map
Failure Map
Security Map
```

---

# 10. Dependency Map

Ví dụ:

| Tool                 | Input                      | Output         | Dependency                  | Side Effect            |
| -------------------- | -------------------------- | -------------- | --------------------------- | ---------------------- |
| `Reconcile.xlsm`     | Sheet `SourceA`, `SourceB` | `Exceptions`   | Excel/VBA                   | Workbook changes       |
| `parser.py`          | CSV/TXT                    | normalized CSV | Python stdlib               | Creates files          |
| `run_parser.bat`     | arguments                  | log/output     | Python executable           | Executes child process |
| `ExtractEvents.ps1`  | Event Log                  | CSV report     | PowerShell/Event Log access | Creates report         |
| Outlook macro/script | report files               | Draft          | classic Outlook profile     | Creates draft          |

---

# 11. Tìm Hard-Coded Paths

Ví dụ Python:

```python
input_file = "C:\\Users\\Admin\\Desktop\\users.csv"
```

Đây là defect vì tool phụ thuộc:

```text
username
desktop location
specific machine
```

Batch:

```bat
python C:\Day12\parser.py C:\Day12\data.csv
```

VBA:

```vb
Workbooks.Open "C:\Users\Alice\Desktop\IAM.xlsx"
```

PowerShell:

```powershell
$Output = "C:\Temp\report.csv"
```

Bạn phải document, chưa nhất thiết sửa tất cả nếu không nằm trong scope.

---

# 12. Tìm Unsafe Assumptions

Ví dụ:

```text
all files are UTF-8
all IDs are numeric
all sheets exist
all column names exist
all folders exist
all attachment names are unique
child process always succeeds
input file is never empty
all rows have Owner
Outlook Inbox contains target folder
```

Mỗi assumption nên có:

```text
Assumption
Risk
Test case
Expected handling
```

---

# 13. Tìm Failure Points

Ví dụ:

```text
File open
Encoding decode
CSV parsing
Regex match
Dictionary insert
Output write
Outlook access
Attachment save
Event Log access
```

Mỗi external boundary là potential failure point.

---

# 14. Change Record trước Modification

Một simplified lab change record:

```text
Change ID: CHG-D12-001
Tool: parser.py
Reason:
  Existing parser assumes UTF-8 and silently drops malformed rows.

Current behavior:
  Invalid CP932 input produces incomplete output without clear failure.

Required behavior:
  Explicitly reject/quarantine malformed input and return non-zero.

Risk:
  Parser output format may change.

Rollback:
  Restore scripts_original\parser.py.

Verification:
  Run TC-ENC-001 through TC-ENC-008.
```

Điểm quan trọng:

```text
Change → Requirement → Tests
```

phải trace được.

---

# 15. Lab 2 — Excel/VBA Reconciliation Workbook

Đây là một lab trọng tâm.

## Objective

Repair workbook để:

```text
reconcile IDs across sheets
preserve IDs as text
detect duplicates
detect missing records
generate exception report
preserve originals
```

---

# 16. Workbook Design

Một workbook Day 12 có thể gồm:

```text
Control
SourceA_Raw
SourceB_Raw
Reconciliation
Exceptions
RunLog
```

Day 13 sẽ mở rộng thành nhiều sheet hơn; Day 12 tập trung kỹ thuật automation.

---

# 17. Raw Sheets

`SourceA_Raw` và `SourceB_Raw` không nên bị macro normalize trực tiếp.

Ví dụ:

| OriginalID | Name          |
| ---------- | ------------- |
| `00000123` | Nguyen Van An |
| `user002`  | Le B          |

Trong `Reconciliation`, tạo:

| OriginalID | NormalizedID |
| ---------- | ------------ |
| `00000123` | `00000123`   |
| `user002`  | `USER002`    |

Raw representation vẫn tồn tại.

---

# 18. VBA: Explicit Variables

Starter macro có thể thiếu:

```vb
Option Explicit
```

Bản repair nên sử dụng:

```vb
Option Explicit
```

Ví dụ:

```vb
Dim wsSourceA As Worksheet
Dim wsSourceB As Worksheet
Dim wsExceptions As Worksheet
Dim lastRow As Long
Dim userId As String
```

UserID là:

```vb
String
```

không phải:

```vb
Long
```

để giữ leading zeros.

---

# 19. Function Normalize ID

Ví dụ đơn giản:

```vb
Private Function NormalizeID(ByVal rawValue As String) As String
    NormalizeID = UCase$(Trim$(rawValue))
End Function
```

Nhưng đây chỉ hợp lệ nếu approved normalization rule thực sự là:

```text
trim surrounding whitespace
case-insensitive / uppercase canonical form
```

Không được tự thêm:

```text
remove hyphens
remove internal spaces
convert full-width automatically
```

nếu rule chưa định nghĩa.

---

# 20. Preserve Blank IDs

Sai:

```vb
If userId = "" Then
    GoTo NextRow
End If
```

nếu không record exception.

Blank ID phải có explicit classification:

```text
MISSING_ID
```

Ví dụ:

```vb
If Len(userId) = 0 Then
    ' Write exception row with MISSING_ID
End If
```

---

# 21. Dictionary-Based Matching

Conceptual VBA:

```vb
Dim dict As Object
Set dict = CreateObject("Scripting.Dictionary")
```

Thêm ID:

```vb
If dict.Exists(userId) Then
    ' duplicate
Else
    dict.Add userId, rowNumber
End If
```

Không nên:

```vb
dict(userId) = rowNumber
```

một cách blindly vì duplicate trước đó có thể bị overwrite.

---

# 22. Duplicate Composite Key

Ví dụ dataset:

```text
UserID      System      Role
000123      AWS         Admin
000123      AWS         ReadOnly
```

Nếu business key:

```text
UserID + System + Role
```

thì hai rows không duplicate.

Composite key conceptual:

```vb
key = userId & "|" & systemName & "|" & roleName
```

Tuy nhiên delimiter collision cũng là consideration; trong lab nhỏ có thể dùng approved known-safe composition.

---

# 23. Missing Record Detection

Giả sử:

```text
Source A = Identity Master
Source B = Application Accounts
```

Nếu:

```text
A exists / B missing
```

classify:

```text
MISSING_IN_B
```

Nếu:

```text
B exists / A missing
```

classify:

```text
MISSING_IN_A
```

Không nên chỉ ghi:

```text
Not found
```

vì không rõ direction.

---

# 24. Exception Report

Một structured exception sheet:

| RunID | OriginalID | NormalizedID | ExceptionType | Source  | Details       |
| ----- | ---------- | ------------ | ------------- | ------- | ------------- |
| R001  | `00000123` | `00000123`   | DUPLICATE     | SourceA | Appears twice |
| R001  | blank      | blank        | MISSING_ID    | SourceA | Row 8         |
| R001  | `00000129` | `00000129`   | MISSING_IN_A  | SourceB | —             |

Machine-readable `ExceptionType` quan trọng hơn color.

---

# 25. Conditional Formatting

Có thể highlight:

```text
DUPLICATE
MISSING_ID
MISSING_IN_A
MISSING_IN_B
```

Nhưng visual format chỉ là presentation aid.

Evidence phải dựa vào actual fields.

---

# 26. Data Validation

Ví dụ `ReviewStatus`:

```text
NEW
INVESTIGATING
RESOLVED
ACCEPTED
```

có thể dùng Excel Data Validation.

Điều này giúp tránh:

```text
resolved
Resolved
Done
OK
```

không nhất quán.

---

# 27. VBA Exception Handling

Starter macro có thể có:

```vb
On Error Resume Next
```

toàn bộ procedure.

Đây là vấn đề.

Safer structure:

```vb
On Error GoTo ErrorHandler

' normal processing

CleanExit:
    Exit Sub

ErrorHandler:
    ' record safe error context
    ' cleanup
    Resume CleanExit
```

Không log secret vào workbook.

---

# 28. Macro Security Verification

Nếu Excel block macro:

Không làm:

```text
Disable all protections globally.
```

Bạn cần:

```text
verify workbook source
verify approved lab setting
verify trusted/signed configuration
confirm organization policy
```

Nếu policy không cho phép execution, record:

```text
LAB GAP / SECURITY CONTROL
```

thay vì circumvent.

---

# 29. VBA Positive Test

Input:

```text
SourceA:
00000123
00000124

SourceB:
00000123
00000124
```

Expected:

```text
Matched = 2
Exceptions = 0
```

---

# 30. VBA Duplicate Test

Input:

```text
00000123
00000123
```

Expected:

```text
DUPLICATE
```

Không được silently choose first hoặc last record.

---

# 31. VBA Leading-Zero Test

Input:

```text
00000123
```

Expected output:

```text
00000123
```

không:

```text
123
```

Đây là mandatory acceptance concern trong curriculum. Infrastructure_Operations_14_Da…

---

# 32. Excel Formula Injection Test

Synthetic source value:

```text
=TEST_VALUE
```

Raw evidence:

```text
must remain unchanged
```

Presentation export:

```text
must not unintentionally evaluate it as active spreadsheet formula
```

Exact sanitization technique phải phù hợp approved export contract.

Điểm bạn cần chứng minh là:

```text
raw preserved
presentation-safe copy separated
```

---

# 33. VBA Verification

Sau execution:

```text
Source raw sheets unchanged?
Input counts unchanged?
Leading-zero IDs preserved?
Duplicate count correct?
Missing records correct?
Exceptions match golden answer?
No unexpected workbook links?
No plaintext secret?
```

---

# 34. VBA Evidence

Collect:

```text
before VBA code
after VBA code
code diff
workbook hash before
workbook hash after
test cases
expected result
actual result
screenshot/report
RunLog
```

---

# 35. VBA Rollback

Rollback:

```text
Close modified workbook
Restore original .xlsm from rollback package
Verify SHA-256 equals baseline
Open in lab
Run known-good baseline test
```

Rollback validation là một step riêng, không chỉ copy file.

---

# 36. Lab 3 — Python Parser

## Objective

Repair Python parser để:

```text
read CSV/TXT/log
handle explicit encoding
normalize text
extract IDs using Regex
detect malformed records
write structured output
log execution
hash files
return explicit exit codes
```

---

# 37. Python Folder Layout

Ví dụ:

```text
scripts_modified\
    parser.py

datasets\
    input_utf8.csv
    input_utf8_bom.csv
    input_utf16.csv
    input_cp932.csv
    malformed.bin

actual\
logs\
```

---

# 38. Use pathlib

Ví dụ:

```python
from pathlib import Path

input_path = Path(r"C:\Day12\datasets\input_utf8.csv")
output_path = Path(r"C:\Day12\actual\normalized.csv")
```

Nhưng tốt hơn tool nhận arguments thay vì hard-code.

Ví dụ conceptual CLI:

```cmd
python parser.py --input "C:\Day12\datasets\input_utf8.csv" --output "C:\Day12\actual\result.csv" --encoding utf-8
```

---

# 39. Validate File Existence

Python:

```python
if not input_path.exists():
    raise FileNotFoundError(f"Input file not found: {input_path}")
```

Tool không được tự tạo empty input rồi claim success.

---

# 40. Explicit Encoding

Ví dụ:

```python
with input_path.open("r", encoding="utf-8", newline="") as f:
    ...
```

UTF-8 BOM:

```python
encoding="utf-8-sig"
```

CP932 fixture:

```python
encoding="cp932"
```

Không nên:

```python
encoding="utf-8",
errors="ignore"
```

cho audit-sensitive processing.

Vì `ignore` có thể silently drop bytes.

---

# 41. CSV Parsing

Use:

```python
import csv

with input_path.open("r", encoding=encoding, newline="") as f:
    reader = csv.DictReader(f)
```

Không dùng:

```python
line.split(",")
```

vì dataset có quoted comma và embedded newline.

---

# 42. Required Header Validation

Expected:

```python
required = {"UserID", "Name"}
```

Validate:

```python
missing = required - set(reader.fieldnames or [])
```

Nếu missing:

```text
FAILED
Reason: MISSING_REQUIRED_COLUMNS
```

Không tạo misleading output.

---

# 43. Normalize String

Ví dụ:

```python
def normalize_id(value: str) -> str:
    return value.strip().upper()
```

Chỉ dùng nếu rule này được documented.

Không convert:

```python
int(value)
```

vì sẽ làm:

```text
00000123 → 123
```

---

# 44. Regex Extraction

Ví dụ log:

```text
2026-10-22 user=USR000123 action=LOGIN
```

Pattern:

```python
r"\bUSR\d{6}\b"
```

Use:

```python
match = re.search(pattern, line)
```

Nếu no match:

```text
NO_ID_FOUND
```

không nên fabricate.

---

# 45. Long Identifier Test

Input:

```text
123456789012345678901234567890
```

Python string có thể giữ nguyên.

Expected:

```text
Output exactly equal raw identifier
```

No numeric conversion.

---

# 46. Logging

Example:

```python
import logging

logging.basicConfig(
    filename="run.log",
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(message)s"
)
```

Log:

```text
INFO run=R001 input=input_utf8.csv
INFO records_read=100
INFO accepted=96
WARNING rejected=4
```

Không log:

```text
generated_password
token
secret
```

---

# 47. SHA-256 Function

Example:

```python
import hashlib

def sha256_file(path: Path) -> str:
    h = hashlib.sha256()

    with path.open("rb") as f:
        for chunk in iter(lambda: f.read(1024 * 1024), b""):
            h.update(chunk)

    return h.hexdigest()
```

Verify:

```text
input hash recorded before processing
input hash recorded after processing
```

Hai hash phải giống nếu parser được thiết kế read-only với input.

---

# 48. Structured Output

Output:

```csv
OriginalID,NormalizedID,Status,Reason
00000123,00000123,MATCHED,
 user002 ,USER002,MATCHED,
,MISSING_ID,REJECTED,MISSING_ID
```

Tốt hơn free-text report vì downstream automation xử lý được.

---

# 49. Exit Codes

Một lab convention có thể định nghĩa:

```text
0 = success
2 = validation error
3 = encoding error
4 = file/process error
```

Đây là lab contract, không phải universal standard.

Python:

```python
import sys
sys.exit(3)
```

khi encoding failure.

---

# 50. Error Handling

Example:

```python
try:
    ...
except UnicodeDecodeError as exc:
    logging.error("Encoding failure: %s", exc)
    return 3
except FileNotFoundError as exc:
    logging.error("%s", exc)
    return 4
```

Không:

```python
except Exception:
    pass
```

---

# 51. Explicit Encoding Failure

Malformed CP932 fixture phải:

```text
FAIL
```

hoặc:

```text
QUARANTINED
```

với reason.

Không:

```text
PASS with replacement characters
```

Syllabus yêu cầu unmappable/malformed input phải rejected hoặc quarantined với reason. Infrastructure_Operations_14_Da…

---

# 52. Python Positive Test

Command:

```cmd
python parser.py --input "C:\Day12\datasets\valid_utf8.csv" --output "C:\Day12\actual\valid_utf8_out.csv" --encoding utf-8
```

Expected:

```text
Exit code: 0
Output exists
Counts reconcile
Vietnamese text intact
```

Verify exit code in cmd:

```cmd
echo %ERRORLEVEL%
```

---

# 53. Python UTF-8 BOM Test

Input header physically contains BOM.

Expected:

```text
UserID recognized correctly
```

Output must not have header:

```text
﻿UserID
```

unless output contract explicitly uses BOM.

---

# 54. Python UTF-16 Test

Input encoded UTF-16.

Expected:

```text
characters preserved
CSV fields correct
no null-looking garbage
```

---

# 55. Python CP932 Test

Fixture contains Japanese text.

Expected:

```text
Japanese characters survive valid decode
```

For invalid bytes:

```text
explicit non-success
```

---

# 56. Python Embedded Comma Test

Input:

```csv
000001,"Nguyen, An"
```

Expected `Name`:

```text
Nguyen, An
```

one field.

---

# 57. Embedded Newline Test

Input:

```csv
000001,"Line one
Line two"
```

Expected:

```text
one logical CSV row
```

Do not equate physical lines with record count.

---

# 58. Empty Input Test

Empty file.

Expected behavior must be defined.

Example:

```text
Exit 2
Reason EMPTY_INPUT
No misleading success report
```

---

# 59. Locked File Test

Create controlled fixture where output/input is locked.

Expected:

```text
observable failure
sanitized error
non-zero exit
```

No partial success.

---

# 60. Permission-Denied Test

Use a controlled folder with restricted access if lab allows.

Expected:

```text
Access failure identified correctly.
```

Do **not** “fix” immediately by running Administrator.

First identify exact required permission.

---

# 61. Python Evidence

Capture:

```text
parser_before.py
parser_after.py
diff
python version
input hash
output hash
record counts
run log
exit codes
test matrix
```

---

# 62. Python Rollback

Restore original:

```powershell
Copy-Item `
 "C:\Day12\rollback\parser.py" `
 "C:\Day12\scripts_modified\parser.py" `
 -Force
```

Hash verify:

```powershell
Get-FileHash "C:\Day12\scripts_modified\parser.py"
```

Compare against original baseline.

---

# 63. Lab 4 — Windows Batch Wrapper

## Objective

Repair Batch wrapper để:

```text
handle paths with spaces
validate inputs
execute Python
preserve child exit status
write clear operational log
return correct final exit code
```

Đây là explicit Day 12 requirement. Infrastructure_Operations_14_Da…

---

# 64. Broken Wrapper Example

```bat
@echo off
python C:\Day12\scripts_modified\parser.py C:\Day12\Data Files\users.csv
echo Completed
exit /b 0
```

Có ít nhất hai lỗi.

Path:

```text
C:\Day12\Data Files\users.csv
```

không được quote.

Và:

```bat
exit /b 0
```

che giấu Python failure.

---

# 65. Safer Wrapper Structure

Example:

```bat
@echo off
setlocal

set "PYTHON=python"
set "SCRIPT=C:\Day12\scripts_modified\parser.py"
set "INPUT=C:\Day12\Data Files\users.csv"
set "OUTPUT=C:\Day12\actual\users_out.csv"

"%PYTHON%" "%SCRIPT%" --input "%INPUT%" --output "%OUTPUT%" --encoding utf-8

set "RC=%ERRORLEVEL%"

if not "%RC%"=="0" (
    echo Child process failed with exit code %RC%
    exit /b %RC%
)

echo Completed successfully.
exit /b 0
```

Điểm quan trọng:

```bat
set "VAR=value"
```

giúp tránh một số vấn đề whitespace/quoting.

---

# 66. Verify Child Exit Status

Force Python test case trả:

```text
3
```

Run Batch:

```cmd
run_parser.bat
```

Then:

```cmd
echo %ERRORLEVEL%
```

Expected:

```text
3
```

Không:

```text
0
```

---

# 67. Path-With-Spaces Test

Use:

```text
C:\Day12\Test Data\UTF 8 File.csv
```

Expected:

```text
processed correctly
```

Nếu Batch thấy:

```text
'C:\Day12\Test' is not recognized...
```

quoting bị lỗi.

---

# 68. Missing Python Test

Controlled test có thể đổi `PYTHON` thành nonexistent executable.

Expected:

```text
non-zero exit
clear message
no false success
```

---

# 69. Missing Input Test

Delete/move working test copy, không xóa raw evidence.

Expected:

```text
Python fails
Batch propagates failure
Output not marked success
```

---

# 70. Partial Prior Output Test

Đây là case rất quan trọng.

Bước 1:

```text
Run success → result.csv exists
```

Bước 2:

```text
Change input to invalid encoding
```

Bước 3:

```text
Run fails
```

Kiểm tra downstream process có vô tình sử dụng old `result.csv` không.

Safer design thường có:

```text
run-specific output
```

ví dụ:

```text
result_R001.csv
result_R002.csv
```

---

# 71. Batch Evidence

Record:

```text
batch_before.bat
batch_after.bat
source diff
test commands
child exit codes
paths-with-spaces result
execution transcript
```

---

# 72. Lab 5 — PowerShell Windows Event Extraction

## Objective

Develop/extend PowerShell tool để:

```text
extract bounded Windows events
retain important metadata
export structured CSV
use explicit time window
handle empty/no-access cases
record operational evidence
```

Day 12 yêu cầu phát triển PowerShell extraction/report automation. Infrastructure_Operations_14_Da…

---

# 73. Read-Only Query First

Ví dụ:

```powershell
Get-WinEvent -LogName System -MaxEvents 10
```

Đây là read-only check.

Bạn chưa export hoặc modify gì.

---

# 74. Bounded Time Window

Example:

```powershell
$Start = (Get-Date).AddHours(-1)
$End   = Get-Date
```

Then:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName   = 'System'
    StartTime = $Start
    EndTime   = $End
}
```

Production/audit workflow nên dùng ticket-defined time window thay vì arbitrary `-1 hour`, nhưng lab có thể dùng fixture window.

---

# 75. Select Relevant Fields

Example:

```powershell
$events |
    Select-Object `
        MachineName,
        ProviderName,
        Id,
        RecordId,
        LevelDisplayName,
        TimeCreated,
        Message
```

Việc giữ `RecordId` hữu ích cho traceability.

---

# 76. Export CSV

```powershell
$events |
    Select-Object `
        MachineName,
        ProviderName,
        Id,
        RecordId,
        LevelDisplayName,
        TimeCreated,
        Message |
    Export-Csv `
        -Path "C:\Day12\actual\system_events.csv" `
        -NoTypeInformation `
        -Encoding UTF8
```

Encoding behavior có thể khác PowerShell generation, nên lab phải record runtime version và explicitly set encoding khi fidelity quan trọng.

---

# 77. Event Count

```powershell
$events.Count
```

Hoặc robust collection:

```powershell
@($events).Count
```

Record count cần ghi vào evidence manifest.

---

# 78. No-Match Test

Time window không có events.

Đây không nhất thiết là failure.

Phải phân biệt:

```text
query succeeded, zero matches
```

với:

```text
query failed
```

Script có thể output:

```text
RESULT=SUCCESS
EVENT_COUNT=0
```

nếu zero events là valid result.

---

# 79. Invalid Log Test

Use controlled invalid channel:

```powershell
Get-WinEvent -LogName "Nonexistent-Test-Log"
```

Expected:

```text
explicit failure
```

Không claim:

```text
No events
```

---

# 80. Permission Test

Một Security log extraction có thể yêu cầu quyền khác System log tùy environment.

Nếu access denied:

```text
record failure
```

Không tự elevate ngoài approved scope.

---

# 81. PowerShell Error Handling

Example:

```powershell
try {
    $events = Get-WinEvent -FilterHashtable $Filter -ErrorAction Stop
}
catch {
    Write-Error "Event extraction failed: $($_.Exception.Message)"
    exit 2
}
```

Không expose unnecessary sensitive data.

---

# 82. PowerShell File Hash

After export:

```powershell
Get-FileHash `
  "C:\Day12\actual\system_events.csv" `
  -Algorithm SHA256
```

Record:

```text
file
size
record count
SHA-256
collection time
filter
host
```

---

# 83. Native vs Presentation Evidence

Day 12 focuses operational reports; earlier Windows Day 3 also covered native EVTX preservation.

Trong Day 12, nếu lab chỉ yêu cầu structured CSV report thì CSV là automation output.

Không được mặc nhiên gọi CSV:

```text
native Windows Event Log evidence
```

Nó là derived report.

Nếu native evidence cần thiết theo runbook, phải export/collect riêng.

---

# 84. PowerShell Evidence

Collect:

```text
ExtractEvents_before.ps1
ExtractEvents_after.ps1
source diff
PowerShell version
filter
time window
host
CSV
event count
hash
execution log
exit code
```

---

# 85. Lab 6 — Secure Password Generation

## Objective

Implement một **lab-only disposable password generator** sử dụng cryptographically secure random source, validate policy và bảo đảm không leak plaintext vào evidence.

Syllabus cấm dùng VBA `Rnd` hoặc predictable seed cho operational passwords. Infrastructure_Operations_14_Da…

---

# 86. Python Secure Source

Python có module:

```python
import secrets
```

Ví dụ lab-safe implementation:

```python
import secrets
import string

def generate_password(length: int) -> str:
    if length < 16:
        raise ValueError("Password length below lab policy minimum")

    alphabet = (
        string.ascii_uppercase +
        string.ascii_lowercase +
        string.digits +
        "!@#$%^&*"
    )

    return "".join(secrets.choice(alphabet) for _ in range(length))
```

Đây là example lab; actual enterprise password policy phải theo organizational requirement.

---

# 87. Character Policy Caveat

Nếu policy yêu cầu ít nhất:

```text
1 uppercase
1 lowercase
1 digit
1 symbol
```

việc chọn ngẫu nhiên từ combined alphabet không tự đảm bảo mỗi generated password luôn có đủ nhóm.

Do đó implementation cần validate output hoặc construct theo approved policy rồi securely shuffle.

Lab phải test **policy compliance**, không chỉ độ dài.

---

# 88. Không hiển thị Password trong Evidence

Không:

```python
print(password)
```

trong evidence capture.

Thay vào đó:

```text
Password generation: SUCCESS
Length: 20
Uppercase: PASS
Lowercase: PASS
Digit: PASS
Symbol: PASS
Plaintext logged: NO
```

Nếu operator cần xem disposable password trong lab để verify, việc đó phải ở controlled step và **không được lưu vào evidence**.

---

# 89. Secret Leakage Test

Search output folder:

PowerShell:

```powershell
Select-String `
  -Path "C:\Day12\evidence\*" `
  -Pattern "<known-disposable-test-secret>" `
  -SimpleMatch
```

Expected:

```text
no matches
```

Nhưng chỉ dùng disposable known lab secret.

Không đưa credential thật vào search command/evidence.

---

# 90. Invalid Policy Test

Example:

```text
Requested length = 4
Minimum allowed = 16
```

Expected:

```text
FAILED_POLICY
```

Tool không silently generate weak password.

---

# 91. Secure Random Provider Failure

Trong lab có thể simulate bằng dependency injection/mock fixture hoặc supplied failed result.

Không cần làm hỏng OS random source.

Expected:

```text
generation aborts
no weak fallback
```

Đây là `fail closed`.

Minimum test catalog yêu cầu trường hợp này. Infrastructure_Operations_14_Da…

---

# 92. Không đánh giá Cryptographic Security bằng Uniqueness Sample

Ví dụ bạn generate 1.000 passwords và thấy không duplicate.

Điều đó không chứng minh:

```text
cryptographically secure
```

Curriculum nêu rõ uniqueness sampling không phải bằng chứng về cryptographic security. Infrastructure_Operations_14_Da…

Bạn phải dựa vào approved cryptographically secure generator.

---

# 93. Lab 7 — Outlook Classic COM/MAPI Workflow

## Objective

Trong **classic Outlook test environment**:

```text
access test folder
process allowlisted attachments
handle filename collision
prepare report
create draft
verify
DO NOT SEND
```

Syllabus chỉ định sử dụng `GetNamespace('MAPI')`, test folder, allowlisted attachment processing và draft-only workflow. Infrastructure_Operations_14_Da…

---

# 94. Safety Boundary của Outlook Lab

Không:

```text
Send automatically
process production mailbox
download arbitrary attachments
overwrite files silently
use real confidential emails
```

Phải:

```text
test mailbox
test folder
allowlist
unique output folder
draft only
manual verification
```

---

# 95. Object Model

Conceptual VBA:

```vb
Dim olApp As Object
Dim olNs As Object

Set olApp = CreateObject("Outlook.Application")
Set olNs = olApp.GetNamespace("MAPI")
```

`GetNamespace("MAPI")` là bước syllabus yêu cầu.

---

# 96. Select Test Folder

Pseudo-VBA:

```vb
Dim inbox As Object
Dim testFolder As Object

Set inbox = olNs.GetDefaultFolder(6)
Set testFolder = inbox.Folders("Day12-Test")
```

Exact constants/object behavior có thể tùy implementation; trong actual enterprise lab nên dùng approved Outlook/VBA references/runbook.

Nếu folder không tồn tại:

```text
STOP
FAILED_FOLDER_NOT_FOUND
```

Không silently fallback về toàn Inbox.

---

# 97. Enumerate Messages

Conceptual:

```vb
Dim item As Object

For Each item In testFolder.Items
    ' process controlled test items
Next
```

Nên filter scope thay vì process mọi message.

---

# 98. Attachment Allowlist

Ví dụ allowed lab extensions:

```text
.csv
.txt
.log
```

Pseudo-logic:

```text
If extension is allowlisted:
    inspect/save
Else:
    record rejected attachment
```

Allowlist cụ thể phải theo use case.

---

# 99. Filename Sanitization

Attachment:

```text
report.csv
```

không được blindly trust.

Destination phải luôn nằm trong controlled directory.

Conceptual:

```text
C:\Day12\working\attachments\
```

Không để attacker-controlled path thoát ra:

```text
..\..\somewhere
```

Trong synthetic lab, fixture có thể kiểm tra handling này.

---

# 100. Filename Collision

Message 1:

```text
report.csv
```

Message 2:

```text
report.csv
```

Không overwrite.

Possible convention:

```text
report_001.csv
report_002.csv
```

hoặc:

```text
R001_MSG001_report.csv
```

Mục tiêu:

```text
both preserved
traceable to source message
```

---

# 101. Missing Attachment Test

Target message không có expected attachment.

Expected:

```text
MISSING_ATTACHMENT
```

Không:

```text
process previous attachment
```

---

# 102. Disallowed Attachment Test

Fixture:

```text
unexpected.exe
```

Expected:

```text
REJECTED_DISALLOWED_TYPE
```

Không save/run.

---

# 103. Create Draft

Pseudo-VBA:

```vb
Dim mail As Object
Set mail = olApp.CreateItem(0)

mail.Subject = "Day 12 Test Report"
mail.Body = "Lab report prepared for verification."

mail.Save
```

Quan trọng:

```vb
mail.Save
```

không:

```vb
mail.Send
```

Lab acceptance yêu cầu verified draft only. Infrastructure_Operations_14_Da…

---

# 104. Attach Report

Concept:

```vb
mail.Attachments.Add "C:\Day12\actual\exception_report.csv"
```

Trước attach cần verify:

```text
file exists
expected hash/count
correct run ID
correct classification
```

Không attach stale report.

---

# 105. Draft Verification

Operator kiểm tra:

```text
Subject correct?
Recipients expected?
Body correct?
Attachments correct?
Attachment count correct?
Run ID correct?
No plaintext secret?
No production destination?
Still Draft?
```

Evidence:

```text
Draft screenshot
message metadata
attachment filenames
```

Không cần lưu sensitive mailbox content ngoài scope.

---

# 106. Missing Folder Test

Rename test folder hoặc dùng fixture path không tồn tại.

Expected:

```text
explicit failure
no fallback
no draft
```

---

# 107. Unavailable Profile Test

Nếu Outlook profile unavailable, automation phải:

```text
fail clearly
```

Không attempt to create arbitrary profile or request credential outside approved scope.

---

# 108. Security Prompt Test

Nếu security control blocks automation:

```text
record exact symptom
stop
escalate according to policy
```

Không:

```text
disable endpoint security
disable macro controls
```

---

# 109. Classic vs New Outlook Boundary

Theo syllabus, Day 12 COM/MAPI/VBA workflow yêu cầu compatible **classic Outlook**; nếu environment chỉ có new Outlook thì đây phải được ghi là environment/practice gap, không giả vờ hands-on đã hoàn thành. Infrastructure_Operations_14_Da…

---

# 110. Lab 8 — Integrated End-to-End Run

Sau khi từng tool hoạt động riêng, ghép chúng.

Workflow:

```text
Raw fixture files
       │
       ▼
Batch wrapper
       │
       ▼
Python parser
       │
       ▼
Normalized CSV
       │
       ▼
Excel/VBA reconciliation
       │
       ▼
Exception report
       │
       ├───────────────┐
       │               │
       ▼               ▼
PowerShell        Evidence hash
Event report           │
       │               │
       └───────┬───────┘
               ▼
          Outlook Draft
               │
               ▼
          Manual Verify
```

---

# 111. End-to-End Run ID

Create:

```text
R20261022-001
```

Mọi output nên liên kết run ID khi practical.

Ví dụ:

```text
R20261022-001_normalized.csv
R20261022-001_exceptions.csv
R20261022-001_events.csv
R20261022-001_run.log
```

Điều này giúp tránh stale-output problem.

---

# 112. Pre-Run Checklist

Trước run:

```text
Correct lab VM?
Correct user?
Correct script version?
Correct input dataset?
Raw hashes recorded?
Output folder clean/new?
Test mailbox selected?
No production destinations?
Change record approved?
Rollback package exists?
```

Nếu bất kỳ point critical nào không rõ:

```text
STOP
```

---

# 113. Run Python through Batch

Example:

```cmd
"C:\Day12\scripts_modified\run_parser.bat"
```

Check:

```cmd
echo %ERRORLEVEL%
```

Expected:

```text
0
```

nếu valid run.

---

# 114. Verify Python Output

Check:

```powershell
Get-Item "C:\Day12\actual\R20261022-001_normalized.csv"
```

Hash:

```powershell
Get-FileHash `
  "C:\Day12\actual\R20261022-001_normalized.csv" `
  -Algorithm SHA256
```

Count records bằng parser-aware method; không blindly dùng line count nếu embedded newline tồn tại.

---

# 115. Reconcile in VBA

Import/copy processed output vào working workbook.

Run approved macro.

Verify:

```text
matched
unmatched
duplicates
rejected
```

against golden answer.

---

# 116. Run PowerShell Extraction

Example:

```powershell
.\ExtractEvents.ps1 `
    -OutputPath "C:\Day12\actual\R20261022-001_events.csv"
```

If script supports parameters.

Capture:

```text
start time
end time
filter
event count
exit code
hash
```

---

# 117. Create Outlook Draft

Only after:

```text
Python PASS
Batch PASS
Excel reconciliation PASS
PowerShell PASS
Evidence hashes recorded
```

Create test draft.

This prevents stale/failed artifacts from being distributed.

---

# 118. End-to-End Verification

You should be able to reconcile:

```text
Raw Input
  ↓
Accepted + Rejected
  ↓
Matched + Unmatched + Duplicate
```

Exact equation depends on counting rules.

Ví dụ:

```text
Raw rows            = 100
Rejected            = 5
Accepted            = 95

Accepted:
Matched             = 80
Unmatched           = 10
Duplicate category  = 5
```

Nhưng beware: duplicate counting rule must be defined.

Ví dụ duplicate group of 3 rows có thể count:

```text
3 duplicate rows
```

hoặc:

```text
1 duplicate group + 2 additional occurrences
```

Do đó counting rule phải documented.

---

# 119. Golden Answer Comparison

Expected:

```text
MATCHED          80
MISSING_IN_A     4
MISSING_IN_B     6
DUPLICATE        5
MISSING_ID       3
REJECTED_ENCODING 2
```

Actual output phải được compare independently.

Không lấy actual result rồi copy thành expected.

Curriculum yêu cầu trusted expected dataset hoặc independently reviewed assertion. Infrastructure_Operations_14_Da…

---

# 120. Lab 9 — Positive Testing

Mỗi tool ít nhất cần happy-path case.

Example Test Case:

```text
ID: TC-POS-001
Requirement:
  Parse valid UTF-8 CSV.

Input:
  valid_utf8.csv

Expected:
  Exit 0.
  100 logical records.
  Vietnamese text preserved.
  Output hash produced.
  No rejected record.
```

Actual:

```text
Exit: 0
Records: 100
Rejected: 0
Text: preserved
PASS
```

---

# 121. Lab 10 — Negative Testing

Examples:

```text
missing file
permission denied
missing column
invalid encoding
missing Outlook folder
disallowed attachment
child process failure
```

Negative test thành công khi tool:

```text
correctly fails
```

không phải khi tool “không gặp lỗi”.

Ví dụ:

```text
Expected: Exit 3
Actual: Exit 3
Result: PASS
```

---

# 122. Lab 11 — Boundary Testing

Required examples:

```text
empty file
single record
very long ID
blank ID
max/min password length
zero Event Log result
path with spaces
```

Purpose:

```text
verify behavior near constraints
```

---

# 123. Lab 12 — Encoding Test Matrix

Một recommended table:

| Case   | Encoding / Format | Expected        |
| ------ | ----------------- | --------------- |
| ENC-01 | UTF-8 no BOM      | PASS            |
| ENC-02 | UTF-8 BOM         | PASS            |
| ENC-03 | UTF-16            | PASS            |
| ENC-04 | valid CP932       | PASS            |
| ENC-05 | malformed CP932   | FAIL/QUARANTINE |
| ENC-06 | CRLF              | PASS            |
| ENC-07 | LF                | PASS            |
| ENC-08 | quoted comma      | PASS            |
| ENC-09 | embedded quote    | PASS            |
| ENC-10 | embedded newline  | PASS            |
| ENC-11 | missing header    | FAIL            |

Điều này bám trực tiếp minimum automation test catalog. Infrastructure_Operations_14_Da…

---

# 124. Lab 13 — Repeatability Test

Run same valid input twice:

```text
R001
R002
```

Expected business output:

```text
same matched count
same duplicate count
same exception classifications
same normalized values
```

Allowed differences:

```text
run ID
timestamp
output filename
log timestamps
```

Curriculum yêu cầu stable business results và không duplicate side effects. Infrastructure_Operations_14_Da…

---

# 125. Repeatability Hash Caveat

Nếu output có timestamp:

```text
R001 hash != R002 hash
```

có thể hoàn toàn bình thường.

Do đó không yêu cầu identical file hashes nếu metadata intentionally changes.

Instead compare:

```text
business rows after excluding run metadata
```

---

# 126. Lab 14 — Regression Testing

Sau mỗi repair:

```text
new defect case
old known-good cases
old known-failure cases
```

đều phải chạy lại.

Ví dụ Python CP932 fix:

Before:

```text
UTF-8 PASS
CP932 FAIL
```

After expected:

```text
UTF-8 PASS
CP932 PASS
Malformed CP932 FAIL explicitly
```

Nếu UTF-8 giờ FAIL:

```text
regression
```

---

# 127. Regression Baseline

Maintain test catalog:

```text
TC-001 valid UTF-8
TC-002 UTF-8 BOM
TC-003 CP932
TC-004 missing file
TC-005 duplicate ID
...
```

Mỗi change chạy lại relevant baseline.

---

# 128. Lab 15 — Failure Injection

Day 12 đặc biệt phù hợp với safe failure injection.

Không phá host hoặc production.

---

## Failure Injection 1 — Wrong Encoding

Fixture:

```text
CP932 file
```

nhưng run as:

```text
UTF-8
```

Expected symptom:

```text
UnicodeDecodeError / explicit decoding error
```

Correct response:

```text
confirm file contract
confirm encoding
do not use errors=ignore
rerun with approved encoding
```

Evidence:

```text
error log
exit code
fixture hash
corrected run
```

---

## Failure Injection 2 — Lost Leading Zeros

Broken VBA converts:

```text
00000123
```

to:

```text
123
```

Symptoms:

```text
false unmatched IDs
```

Troubleshooting:

```text
inspect raw
inspect normalized value
inspect data type
trace Excel/VBA conversion
```

Root cause:

```text
identifier treated as numeric
```

Corrective action:

```text
treat ID as text
```

Regression test all long/leading-zero fixtures.

---

## Failure Injection 3 — Batch False Success

Python returns:

```text
3
```

Batch returns:

```text
0
```

Symptom:

```text
scheduler says success but output missing
```

Root cause:

```text
child exit code not propagated
```

Corrective action:

```text
capture %ERRORLEVEL%
return child status
```

---

## Failure Injection 4 — Duplicate Attachment Filename

Two messages:

```text
report.csv
report.csv
```

Broken workflow:

```text
second overwrites first
```

Correct behavior:

```text
unique filenames
both preserved
source-message traceability
```

---

## Failure Injection 5 — Stale Output

Prior valid:

```text
result.csv
```

Current parser fails.

Downstream Excel still reads old:

```text
result.csv
```

Symptom:

```text
report appears valid despite failed current run
```

Correct design:

```text
run-specific output
successful-run marker
timestamp/run ID validation
```

---

# 129. Troubleshooting Checkpoints

Khi lab fail, luôn đi từ low-risk đến high-risk.

### Checkpoint 1 — Confirm Environment

```text
Correct host?
Correct account?
Correct tool version?
Correct working directory?
```

### Checkpoint 2 — Confirm Input

```text
Correct filename?
Correct hash?
Correct encoding?
Correct schema?
```

### Checkpoint 3 — Check Exit Status

Python:

```cmd
echo %ERRORLEVEL%
```

PowerShell:

```powershell
$LASTEXITCODE
```

hoặc script-specific result.

### Checkpoint 4 — Review Sanitized Logs

Tìm:

```text
first meaningful error
```

không chỉ last cascading failure.

### Checkpoint 5 — Dependency

```text
Python available?
Folder exists?
Outlook profile?
Worksheet?
Event Log channel?
```

### Checkpoint 6 — Permissions

Xác nhận minimum required access.

Không tự chạy elevated trước khi hiểu requirement.

### Checkpoint 7 — Reproduce with Golden Fixture

Nếu production-like dataset fail:

```text
reproduce with sanitized fixture
```

### Checkpoint 8 — Code Investigation

Chỉ sau đó mới sửa code.

---

# 130. Verification Philosophy

Một lab Day 12 chỉ được coi là PASS nếu bốn lớp đều đúng:

```text
Execution
Data
Security
Auditability
```

### Execution

```text
Tool completes correctly.
```

### Data

```text
Records and content are correct.
```

### Security

```text
No secret leakage or unauthorized action.
```

### Auditability

```text
Another reviewer can reconstruct what happened.
```

---

# 131. Không được coi Exit Code 0 là đủ

Ví dụ:

```text
Exit 0
```

nhưng:

```text
Input rows: 1000
Output rows: 930
Rejected: 0
```

70 rows disappeared.

Result:

```text
FAIL
```

---

# 132. Record Count Verification

Một example control:

```text
Input logical records = 100

Accepted = 93
Rejected = 7

93 + 7 = 100
```

Then:

```text
Accepted = Matched + Unmatched + DuplicateCategory
```

theo documented counting rule.

No unexplained loss.

---

# 133. Hash Verification

For raw source:

```text
SHA_before == SHA_after
```

Expected.

For working/output:

```text
new SHA recorded
```

Do not require output hash equal input hash.

---

# 134. Evidence Manifest

Syllabus yêu cầu evidence item có metadata rất cụ thể, bao gồm source, environment, collection timestamp/timezone, command/filter, format/encoding, file path/name, size, count, SHA-256, ticket, transformation/script version và limitations/errors. Infrastructure_Operations_14_Da…

Một Day 12 row có thể là:

```text
Evidence ID:
EV-D12-023

Artifact:
R001_normalized.csv

Source:
valid_utf8.csv

Tool:
parser.py v1.2

Run:
R001

Collected:
2026-10-22T14:00:00+07:00

Encoding:
UTF-8

Input count:
100

Output count:
100

SHA-256:
<hash>

Parent evidence:
EV-D12-005

Ticket:
CHG-D12-001

Limitations:
None
```

---

# 135. Before/After Evidence

Mỗi language bắt buộc có:

```text
BEFORE
AFTER
DIFF
```

Ví dụ:

```text
parser_before.py
parser_after.py
parser.diff
```

VBA:

```text
Module1_before.bas
Module1_after.bas
```

Batch:

```text
run_before.bat
run_after.bat
```

PowerShell:

```text
ExtractEvents_before.ps1
ExtractEvents_after.ps1
```

Đây là explicit acceptance requirement. Infrastructure_Operations_14_Da…

---

# 136. Source Diff

Git không bắt buộc trong syllabus, nhưng có thể dùng nếu lab cho phép.

Simple diff example:

```diff
- user_id = int(row["UserID"])
+ user_id = row["UserID"].strip()
```

Reviewer nhìn thấy ngay root cause/fix.

Tránh formatting toàn file nếu chỉ sửa một issue vì làm diff khó review.

---

# 137. Reproducible Run Instruction

Ví dụ:

```text
Tool:
parser.py v1.2

Runtime:
Python 3.x as recorded in EV-D12-002

Command:
python parser.py ^
  --input "C:\Day12\datasets\valid_utf8.csv" ^
  --output "C:\Day12\actual\valid_utf8_out.csv" ^
  --encoding utf-8

Expected:
Exit code 0.
100 records.
No rejected rows.

Verification:
Compare output to expected\valid_utf8_expected.csv.
```

Reviewer phải có thể reproduce.

---

# 138. Test Evidence Schema

Curriculum yêu cầu record:

```text
Case ID
Requirement
Fixture version
Expected result
Actual result
Pass/fail
Defect/ticket
Retest status
```

Infrastructure_Operations_14_Da…

Một example:

| Case      | Requirement     | Expected      | Actual        | Status |
| --------- | --------------- | ------------- | ------------- | ------ |
| TC-ENC-04 | Valid CP932     | Parse success | Parse success | PASS   |
| TC-ENC-05 | Malformed CP932 | Explicit fail | Exit 3        | PASS   |
| TC-ID-02  | Leading zeros   | Preserve      | Preserved     | PASS   |
| TC-BAT-03 | Child fail      | Propagate 3   | Returned 3    | PASS   |

---

# 139. Failed Test không được đổi nhãn PASS

Nếu:

```text
Expected:
Exit 3
```

Actual:

```text
Exit 0
```

thì:

```text
FAIL
```

Ngay cả khi script:

```text
completed
```

Syllabus explicitly cấm việc coi execution completion là bằng chứng test passed. Infrastructure_Operations_14_Da…

---

# 140. Defect Record

Example:

```text
DEF-D12-004

Test:
TC-BAT-003

Expected:
Batch returns Python exit 3.

Actual:
Batch returned 0.

Root cause:
Hard-coded exit /b 0.

Fix:
Capture and propagate child ERRORLEVEL.

Retest:
PASS.
```

---

# 141. Rollback Package

Recommended:

```text
rollback\
├── Reconcile_original.xlsm
├── parser_original.py
├── run_parser_original.bat
├── ExtractEvents_original.ps1
└── rollback_manifest.txt
```

Rollback manifest:

```text
File
Baseline SHA-256
Restore destination
Validation test
```

---

# 142. Rollback Procedure

Ví dụ parser:

```text
1. Stop scheduled/test execution.
2. Archive modified version.
3. Restore original version.
4. Verify SHA-256.
5. Execute baseline smoke test.
6. Confirm known original behavior.
7. Record rollback evidence.
```

Rollback không chỉ là:

```text
copy old file back
```

mà phải verify restored state.

---

# 143. Stop Conditions

Trong lab, dừng ngay nếu:

```text
Production mailbox accidentally selected
Production dataset discovered
Real secret exposed
Original source modified
Unexpected outbound email attempted
Tool requires unauthorized privilege
Large unexpected file deletion occurs
Hash mismatch cannot be explained
Output counts do not reconcile
```

Sau đó preserve evidence và escalate.

---

# 144. Escalation Boundary

Bạn có thể xử lý trong L1/L2 lab scope:

```text
wrong path
missing file
known schema issue
known encoding issue
approved code defect
quoting error
known exit-code propagation issue
known Excel normalization issue
```

Escalate khi:

```text
business matching rule ambiguous
production behavior differs
security policy needs bypass
credential exposure suspected
unknown data corruption
Outlook unexpectedly sends
macro security policy conflicts
privileged access needed without approval
major redesign required
```

---

# 145. Full Day 12 Evidence Package

Một final submission có thể có cấu trúc:

```text
Day12_Submission/
│
├── 01_Change/
│   ├── CHG-D12-001.txt
│   ├── DependencyMap.xlsx
│   └── RiskAssessment.txt
│
├── 02_Original/
│   ├── Reconcile_original.xlsm
│   ├── parser_original.py
│   ├── run_parser_original.bat
│   └── ExtractEvents_original.ps1
│
├── 03_Modified/
│   ├── Reconcile_modified.xlsm
│   ├── parser_modified.py
│   ├── run_parser_modified.bat
│   └── ExtractEvents_modified.ps1
│
├── 04_Diff/
│   ├── VBA.diff
│   ├── Python.diff
│   ├── Batch.diff
│   └── PowerShell.diff
│
├── 05_Input/
│   ├── valid_utf8.csv
│   ├── utf8_bom.csv
│   ├── utf16.csv
│   ├── cp932.csv
│   └── malformed_fixture.bin
│
├── 06_Expected/
│   └── golden_answers/
│
├── 07_Actual/
│   ├── normalized.csv
│   ├── exceptions.csv
│   └── events.csv
│
├── 08_TestResults/
│   ├── TestMatrix.xlsx
│   └── RegressionResults.csv
│
├── 09_Logs/
│   ├── python.log
│   ├── batch.log
│   └── powershell.log
│
├── 10_Outlook/
│   └── DraftEvidence/
│
├── 11_Hashes/
│   └── manifest.csv
│
├── 12_Rollback/
│   └── original_versions/
│
└── 13_Runbook/
    └── Reproducible_Run_Instructions.txt
```

Không bắt buộc chính xác folder names này; đây là implementation phù hợp với syllabus.

---

# 146. Acceptance Criteria cuối Module Thực hành

Bạn chỉ nên đánh dấu **Day 12 Practical = Completed** khi tất cả điều sau được chứng minh:

### Existing Tool Analysis

```text
☐ Input/output identified
☐ Dependencies documented
☐ Hard-coded paths identified
☐ Unsafe assumptions documented
☐ Failure points identified
```

### Excel/VBA

```text
☐ IDs treated as text
☐ Leading zeros preserved
☐ Long IDs preserved
☐ Raw originals retained
☐ Duplicates detected
☐ Missing records detected
☐ Exception report generated
☐ Formula-injection concern handled in presentation output
☐ Before/after VBA captured
```

### Python

```text
☐ CSV parsing correct
☐ UTF-8 tested
☐ UTF-8 BOM tested
☐ UTF-16 tested
☐ CP932 tested
☐ Malformed encoding explicit
☐ Regex extraction tested
☐ Logging implemented
☐ SHA-256 collected
☐ Exit codes meaningful
```

### Batch

```text
☐ Paths with spaces work
☐ Missing input fails correctly
☐ Child process exit propagated
☐ No false success
```

### PowerShell

```text
☐ Event Log query bounded
☐ Important fields retained
☐ CSV encoding explicit
☐ Event count recorded
☐ No-match differentiated from failure
☐ Output hashed
```

### Password Generation

```text
☐ Cryptographically secure generator used
☐ Invalid policy rejected
☐ Secure-provider failure fails closed
☐ Password evidence masked
☐ No plaintext secret in workbook/log/report/mail
```

### Outlook/MAPI

```text
☐ classic Outlook test environment
☐ GetNamespace("MAPI") workflow
☐ Test folder only
☐ Allowlisted attachments
☐ Filename collisions handled
☐ Missing attachment tested
☐ Missing folder tested
☐ Draft created
☐ Draft verified
☐ No automatic send
```

### Testing

```text
☐ Positive tests
☐ Negative tests
☐ Boundary tests
☐ Encoding tests
☐ Regression tests
☐ Repeatability tests
☐ Expected vs actual recorded
☐ Golden answer comparison passed
```

### Evidence / Rollback

```text
☐ Raw inputs unchanged
☐ Input hashes recorded
☐ Output hashes recorded
☐ Record counts reconcile
☐ Before/after code preserved
☐ Source diff retained
☐ Execution logs retained
☐ Test evidence retained
☐ Rollback package created
☐ Rollback procedure verified
☐ Reproducible run instructions completed
```

Những tiêu chí này trực tiếp phản ánh Assignment/Lab, Additive Assignment và Acceptance/Evidence của Day 12, cộng với common evidence/automation acceptance contract của curriculum. Infrastructure_Operations_14_Da… Infrastructure_Operations_14_Da…

---

# 147. Kết quả cuối cùng bạn phải đạt được

Sau Module Thực hành Day 12, mục tiêu không phải đơn giản là nói:

> “Tôi biết VBA, Python, Batch và PowerShell.”

Bạn phải chứng minh được một capability cao hơn:

```text
Tôi có thể nhận một operations tool hiện hữu
              │
              ▼
       hiểu dependency của nó
              │
              ▼
      bảo vệ source/evidence
              │
              ▼
      xác định defect và risk
              │
              ▼
       sửa minimum scope
              │
              ▼
       test cả success/failure
              │
              ▼
      kiểm tra encoding/data
              │
              ▼
      reconcile với golden data
              │
              ▼
        kiểm tra hash/count
              │
              ▼
          bảo vệ secret
              │
              ▼
       thu evidence đầy đủ
              │
              ▼
       rollback được change
```

Đó chính là competency thực tế mà Day 12 chuẩn bị cho **Day 13**: multi-sheet IAM reconciliation, 90-day access review, exception classification, audit trail và repeatable automation. Nếu Day 12 không đạt độ chính xác về ID, encoding, data integrity, error propagation và regression testing, thì automation của Day 13 dù tạo ra report đẹp vẫn có thể đưa ra **audit conclusion sai**. Infrastructure_Operations_14_Da…
