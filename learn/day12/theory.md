# Day 12 — Module Lý thuyết

## Operations Automation, Data Processing & Safe Internal-Tool Maintenance

Day 12 nằm trong phần **Operations Automation, IAM Audit & Integrated Practice**. Theo syllabus, phạm vi lý thuyết bao gồm maintenance đối với existing operations tools; Excel operational workbooks; VBA; Python; Windows Batch; PowerShell; TXT/CSV/encoding/string/Regex processing; file integrity; secure password generation; controlled credential handling; safe script execution; và Outlook classic COM/MAPI automation. Phần additive của syllabus nhấn mạnh thêm việc bảo toàn ID, encoding fidelity, spreadsheet formula injection, cryptographically secure password generation, raw-evidence preservation và khả năng maintain/repair existing tools thay vì chỉ viết script mới. Infrastructure_Operations_14_Da…

Trong toàn bộ nội dung dưới đây, tôi sẽ phân biệt hai lớp kiến thức. **“Theo syllabus”** là những competency bắt buộc của Day 12. **“Kiến thức bổ sung”** là phần tôi mở rộng để giúp bạn hiểu cách những competency đó được áp dụng trong Infrastructure Operations; phần bổ sung không thay thế hoặc mở rộng scope đánh giá của syllabus một cách tùy tiện.

---

# 1. Tư duy trung tâm của Day 12: Automation trong Infrastructure Operations khác Software Development như thế nào?

Day 12 không đặt mục tiêu biến Infrastructure Operations engineer thành application developer. Trọng tâm là khả năng **safely maintain operational tooling**.

Một Infrastructure Operations engineer thường gặp những tool đã tồn tại từ trước:

```text
Excel workbook
        │
        ├── VBA macro
        │
        ├── CSV exports
        │
        └── manual review
                │
                ▼
           Python parser
                │
                ▼
          Batch wrapper
                │
                ▼
        PowerShell extractor
                │
                ▼
       Operational report
                │
                ▼
       Outlook draft / ticket
```

Tool có thể đã được viết cách đây nhiều năm, không có documentation đầy đủ, sử dụng hard-coded path, giả định một encoding cụ thể, không có regression test hoặc xử lý lỗi rất yếu.

Trong operations, vấn đề quan trọng không chỉ là:

> “Code chạy hay không?”

mà phải là:

> “Code có chạy đúng trên đúng input, dưới đúng identity, trong đúng environment, không làm mất dữ liệu, không lộ secret, không gây side effect ngoài scope và tạo output có thể audit hay không?”

Đó là khác biệt nền tảng.

---

# 2. Enterprise operations tool maintenance

Theo syllabus, Day 12 bắt buộc bao phủ:

```text
existing script analysis
code reading
debugging
modification
regression testing
change control
```

Infrastructure_Operations_14_Da…

Đây là competency quan trọng nhất của toàn bộ Day 12.

---

## 2.1 Existing script analysis là gì?

`Existing script analysis` là quá trình hiểu một tool có sẵn trước khi sửa hoặc chạy nó.

Bạn không nên mở một script 2.000 dòng rồi bắt đầu thay code ngay. Trước tiên cần xây dựng mental model.

Một tool nên được nhìn theo mô hình:

```text
Inputs
  │
  ▼
Validation
  │
  ▼
Transformation
  │
  ▼
Business Logic
  │
  ▼
Output
  │
  ▼
Side Effects
```

Ví dụ:

```text
users.csv
   │
   ▼
Python parser
   │
   ├── normalize ID
   ├── detect duplicate
   ├── lookup inventory
   └── classify exception
   │
   ▼
exceptions.csv
```

Khi phân tích, cần trả lời được những câu hỏi như:

| Area             | Câu hỏi cần trả lời                                            |
| ---------------- | -------------------------------------------------------------- |
| Entry point      | Tool bắt đầu ở đâu?                                            |
| Input            | Nó đọc file/API/workbook/log nào?                              |
| Output           | Nó tạo hoặc sửa cái gì?                                        |
| Dependencies     | Cần module, executable, workbook, mailbox hay environment nào? |
| Permissions      | Account chạy cần quyền gì?                                     |
| State            | Tool có lưu state giữa các lần chạy không?                     |
| Side effects     | Có delete, overwrite, send mail, create task không?            |
| Logging          | Thành công/thất bại được ghi ở đâu?                            |
| Failure behavior | Fail giữa chừng thì hệ thống ở trạng thái nào?                 |
| Security         | Có đọc secret hoặc privileged data không?                      |
| Rollback         | Làm sao quay về trạng thái cũ?                                 |

Đây được gọi là **dependency mapping** và **behavior mapping**.

---

# 3. Code reading — đọc code theo góc nhìn Operations

Code reading không có nghĩa là phải hiểu mọi dòng ngay lập tức.

Một phương pháp tốt là đọc theo nhiều pass.

### Pass đầu tiên: architecture

Tìm:

```text
main entry
functions
modules
input files
output files
external commands
configuration
```

### Pass thứ hai: control flow

Theo dõi:

```text
IF
ELSE
FOR
WHILE
TRY
CATCH
RETURN
EXIT
```

### Pass thứ ba: data flow

Theo dõi một field quan trọng.

Ví dụ `UserID`:

```text
CSV UserID
     ↓
read
     ↓
strip()
     ↓
upper()
     ↓
dictionary lookup
     ↓
exception classification
     ↓
Excel report
```

Cách này giúp phát hiện một vấn đề rất phổ biến:

```text
Raw ID
00012345
```

bị convert thành:

```text
12345
```

ngay từ bước đầu.

Nếu chỉ đọc business logic cuối cùng, bạn sẽ không thấy lỗi bắt nguồn từ conversion trước đó.

---

# 4. Debugging — phải phân biệt loại lỗi

Không phải lỗi nào cũng là “code bug”.

Trong Infrastructure Operations, ít nhất nên phân biệt:

| Loại lỗi            | Ví dụ                                 |
| ------------------- | ------------------------------------- |
| Syntax error        | Thiếu dấu `)`                         |
| Runtime error       | File không tồn tại                    |
| Logic error         | Duplicate được phân loại là normal    |
| Data-quality error  | ID bị blank                           |
| Encoding error      | CP932 file bị decode bằng UTF-8       |
| Permission error    | Service account không đọc được folder |
| Dependency error    | Python module chưa install            |
| Environment error   | Production path khác lab path         |
| Configuration error | Wrong mailbox/folder                  |
| Integration error   | External command trả non-zero         |
| Security control    | Macro bị blocked                      |
| Operational error   | Chạy nhầm dataset                     |

Điểm khó nhất là **logic error** vì chương trình có thể exit code `0` nhưng output sai.

Ví dụ:

```text
Program status: SUCCESS
Rows processed: 10,000
```

nhưng 300 leading-zero IDs đã bị hỏng.

Đó vẫn là failure.

---

# 5. Modification — sửa đúng requirement, không rewrite tùy tiện

Trong enterprise environment, một lỗi nhỏ không nhất thiết cho phép bạn “refactor toàn bộ”.

Giả sử requirement là:

> Batch wrapper không preserve Python exit code.

Change hợp lý có thể chỉ là sửa propagation của exit code.

Không nên nhân cơ hội thay:

```text
folder structure
logging system
naming standard
Python version
argument syntax
```

nếu chúng không thuộc approved change.

Nguyên tắc này gọi là:

**minimum necessary change**.

Nó giúp giảm:

```text
regression risk
review scope
rollback complexity
test scope
change-window duration
```

---

# 6. Regression testing

`Regression testing` kiểm tra xem thay đổi mới có phá những behavior đã hoạt động trước đó không.

Ví dụ bạn sửa parser để hỗ trợ:

```text
CP932
```

Sau khi sửa, không chỉ test CP932.

Bạn vẫn phải test lại:

```text
UTF-8
UTF-8 BOM
UTF-16
previous valid CSV fixtures
previous invalid fixtures
```

Syllabus yêu cầu existing tools được repair/modification nhưng phải có **before/after code, expected/actual tests và reproducible run instruction**. Infrastructure_Operations_14_Da…

Một fix làm case mới PASS nhưng khiến 5 case cũ FAIL là regression.

---

# 7. Change control đối với script

Script cũng là infrastructure change.

Một misconception phổ biến là:

> “Chỉ sửa code thôi, không phải server config nên không cần change control.”

Điều đó sai nếu script có thể:

```text
modify operational data
change accounts
generate compliance reports
create files
move evidence
send mail
restart services
run privileged commands
```

Một script 20 dòng có thể tạo impact lớn hơn một configuration change đơn giản.

Change record lý tưởng liên kết:

```text
Requirement
   ↓
Change
   ↓
Code diff
   ↓
Test cases
   ↓
Test result
   ↓
Deployment
   ↓
Verification
   ↓
Rollback
```

---

# 8. Excel trong Infrastructure Operations

Theo syllabus, Day 12 bao phủ:

```text
formulas
lookup functions
cross-sheet ID reconciliation
cross-workbook ID reconciliation
duplicate detection
missing-record identification
conditional formatting
data validation
Excel Tables
```

Infrastructure_Operations_14_Da…

Excel được xem như một **operational reconciliation platform**, không đơn giản là spreadsheet.

---

# 9. Excel data model: mỗi row phải đại diện cho cái gì?

Trước khi viết formula, phải xác định grain của dataset.

Ví dụ:

```text
One row = one account
```

khác hoàn toàn:

```text
One row = one account + one entitlement
```

Ví dụ account:

```text
UserID     Role
000125     Admin
000125     Auditor
```

Nếu dataset được hiểu là “one account per row”, `000125` trông như duplicate.

Nhưng nếu grain là:

```text
UserID + Role
```

thì đây là hai record hợp lệ.

Do đó duplicate detection không thể tách rời định nghĩa **business key**.

---

# 10. Primary key và composite key

Trong operational data thường không có database-quality primary key.

Bạn có thể gặp:

```text
UserID
Server
Role
Environment
```

Một key có thể cần là:

```text
UserID + System + Role
```

Ví dụ:

```text
000123|AWS|ReadOnly
000123|AWS|Admin
```

Hai row này không phải duplicate.

Khái niệm này trở nên đặc biệt quan trọng ở Day 13 khi thực hiện multi-source IAM reconciliation.

---

# 11. ID phải được coi là text

Đây là yêu cầu additive rõ ràng của syllabus:

> Treat IDs as text, preserve leading zeros and long identifiers.

Infrastructure_Operations_14_Da…

Giả sử ID:

```text
0000012456
```

Nếu Excel convert sang Number:

```text
12456
```

thì data integrity đã bị phá.

Một trường hợp nguy hiểm hơn là identifier dài:

```text
123456789012345678
```

Spreadsheet numeric representation có thể không còn bảo toàn identifier nguyên gốc như mong muốn.

Vì vậy identifier không phải mathematical quantity.

Bạn không:

```text
add UserID
divide UserID
average UserID
```

Nó là text key.

---

# 12. Raw value và normalized value

Một design tốt nên giữ cả:

```text
OriginalID
NormalizedID
```

Ví dụ:

| OriginalID | NormalizedID |
| ---------- | ------------ |
| `USER001`  | `USER001`    |
| `user001`  | `USER001`    |

Nhưng không được overwrite raw source thành normalized value.

Tại sao?

Vì reviewer cần biết:

> “Source thực sự ghi gì?”

Nếu chỉ lưu normalized data, bạn đã mất provenance.

---

# 13. Excel formulas trong operational workbooks

Formula thường phục vụ ba nhóm mục tiêu:

```text
validation
reconciliation
classification
```

Một số function thường gặp về mặt khái niệm gồm:

```text
IF
IFERROR
COUNTIF
COUNTIFS
MATCH
INDEX
VLOOKUP
XLOOKUP
TRIM
LEFT
RIGHT
LEN
```

Khả năng availability và behavior cụ thể có thể phụ thuộc Excel version, do đó workbook enterprise phải ghi rõ compatibility requirement.

Một example:

```excel
=COUNTIF(SourceA!A:A,A2)
```

Có thể dùng để biết `A2` xuất hiện bao nhiêu lần trong `SourceA`.

Nếu:

```text
0
```

có thể là missing.

Nếu:

```text
1
```

có thể là unique match.

Nếu:

```text
>1
```

cần investigate duplicate hoặc one-to-many relationship.

Không được đơn giản coi `>1` luôn là error mà chưa hiểu dataset grain.

---

# 14. Lookup function và reconciliation

Reconciliation thường có dạng:

```text
Source A               Source B
--------               --------
Account ID             Account ID
Owner                  Status
Role                   Department
```

Mục tiêu:

```text
A.ID == B.ID ?
```

Nhưng thực tế phức tạp hơn:

```text
A.ID = "000123"
B.ID = 123
```

Nếu lookup “thành công” bằng coercion nhưng raw representations khác nhau, đây có thể là dấu hiệu upstream data issue.

Do đó Excel reconciliation tốt phải phân biệt:

```text
Exact raw match
Normalized match
Missing
Duplicate
Ambiguous
Rejected
```

---

# 15. Cross-sheet reconciliation

Cross-sheet reconciliation thường là:

```text
Workbook
│
├── Sheet: HR
├── Sheet: IAM
├── Sheet: Assets
└── Sheet: Exceptions
```

Một workflow hợp lý:

```text
HR.UserID
    ↓
normalize
    ↓
lookup IAM.UserID
    ↓
lookup Asset.OwnerID
    ↓
classify
```

Điểm quan trọng là không tạo circular dependency khó audit.

---

# 16. Cross-workbook reconciliation

Cross-workbook tăng thêm risk.

Ví dụ:

```text
HR.xlsx
IAM.xlsx
Asset.xlsx
```

Excel formula có thể tham chiếu external workbook.

Operational issues thường gặp:

```text
source workbook renamed
folder moved
drive mapping changed
network share unavailable
external links stale
wrong version opened
```

Do đó cross-workbook automation cần dependency documentation rõ ràng.

Nếu tool chỉ chạy được vì:

```text
Z:\Shared\Operations\IAM\users.xlsx
```

thì `Z:` là dependency phải được ghi nhận.

---

# 17. Duplicate detection

Duplicate không phải chỉ:

```text
same UserID twice
```

Có nhiều dạng:

```text
exact duplicate
duplicate primary key
duplicate composite key
semantic duplicate
cross-system duplicate
```

Ví dụ:

```text
user001
USER001
 user001
```

Sau normalization có thể trở thành cùng một key.

Nhưng normalization rule phải được approved trước.

---

# 18. Missing-record identification

Missing record cũng cần phân loại.

Ví dụ:

```text
IAM exists / HR missing
```

có thể là orphaned account.

Nhưng:

```text
HR exists / IAM missing
```

có thể chỉ là user chưa được provision.

Context quyết định severity.

Day 12 chỉ xây dựng năng lực automation/reconciliation; Day 13 mới áp dụng logic đó sâu vào 90-day IAM review. Infrastructure_Operations_14_Da…

---

# 19. Conditional Formatting

`Conditional Formatting` giúp reviewer nhìn thấy exception.

Ví dụ:

```text
Duplicate → highlighted
Missing owner → highlighted
Unmatched ID → highlighted
```

Nhưng màu sắc không được là evidence duy nhất.

Một machine-readable field vẫn nên tồn tại:

```text
Status = DUPLICATE
```

chứ không chỉ:

```text
cell màu đỏ
```

Lý do là color:

```text
khó audit tự động
khó export
khó search
khó xử lý bởi Python
```

---

# 20. Data Validation

`Data Validation` kiểm soát dữ liệu được nhập thủ công.

Ví dụ cột:

```text
Decision
```

chỉ cho phép:

```text
KEEP
REVOKE
MODIFY
INVESTIGATE
```

Điều này tốt hơn free text kiểu:

```text
keep
Keep
kept
OK
leave it
```

Standardized values giúp automation reliable hơn.

---

# 21. Excel Tables

`Excel Table` giúp tạo structured dataset với:

```text
headers
filtering
structured references
automatic row expansion
consistent formulas
```

Thay vì VBA giả định:

```text
A1:G500
```

tool có thể xử lý Table mà số row thay đổi.

Tuy nhiên vẫn phải validate header names và schema.

---

# 22. VBA — vai trò trong Day 12

Syllabus bao phủ:

```text
macros
modules
procedures
variables
arrays
loops
dictionaries
file operations
exception handling
macro security
```

Infrastructure_Operations_14_Da…

Mục tiêu không phải học toàn bộ VBA language mà là đủ để maintain Excel-based operational tooling.

---

# 23. Macro là gì?

`Macro` là automation chạy trong Office application.

Ví dụ một Excel macro có thể:

```text
Read Source sheet
     ↓
Normalize UserID
     ↓
Build dictionary
     ↓
Compare Target sheet
     ↓
Write exception rows
     ↓
Format report
```

VBA chạy trong context của Excel process và user identity.

Điều đó có nghĩa nó có thể tiếp cận file và resource mà user có quyền truy cập.

Đây là security consideration quan trọng.

---

# 24. VBA Modules

Một `Module` chứa VBA procedures/functions.

Ví dụ:

```vb
Option Explicit

Sub BuildReport()
    ' ...
End Sub
```

`Option Explicit` là practice hữu ích vì yêu cầu variables được declare, giúp giảm typo-induced bugs.

Ví dụ:

```vb
userId = "000123"
usrId = "000124"
```

nếu typo vô tình tạo variable mới, logic có thể sai mà không obvious.

---

# 25. Sub và Function

`Sub` thực hiện một operation:

```vb
Sub GenerateReport()
End Sub
```

`Function` thường return một value:

```vb
Function NormalizeID(value As String) As String
    NormalizeID = Trim(value)
End Function
```

Tách logic thành reusable functions giúp test và maintain dễ hơn.

---

# 26. Variables và data types

Trong automation xử lý ID, data type là cực kỳ quan trọng.

Một mistake nguy hiểm:

```vb
Dim UserID As Long
```

nếu UserID là:

```text
00001234
```

Thay vào đó, identifier thường nên được coi là:

```vb
Dim UserID As String
```

Đây chính là application của requirement “Treat IDs as text”.

---

# 27. Arrays

`Array` lưu nhiều giá trị trong memory.

Trong Excel automation, arrays thường nhanh hơn việc đọc từng cell hàng chục nghìn lần.

Conceptual workflow:

```text
Worksheet range
     ↓
load into array
     ↓
process in memory
     ↓
write output once
```

Tuy nhiên optimization không nên làm code mất clarity nếu dataset nhỏ và maintainability quan trọng hơn.

---

# 28. Loops

Các loop phổ biến:

```vb
For
For Each
Do While
Do Until
```

Một loop phải có predictable termination.

Ví dụ lỗi nguy hiểm:

```text
While record != found
```

nhưng không có maximum hoặc termination khi file corrupt.

Trong operations, unbounded loop có thể:

```text
consume CPU
hang scheduled job
lock workbook
delay downstream automation
```

---

# 29. Dictionaries

`Dictionary` rất hữu ích cho lookup.

Concept:

```text
key → value
```

Ví dụ:

```text
"000123" → "Active"
"000124" → "Disabled"
```

Dictionary còn có thể dùng để phát hiện duplicate:

```text
If key already exists
    duplicate detected
```

Nhưng phải quyết định behavior khi duplicate xuất hiện.

Không nên:

```text
last row silently wins
```

trừ khi business rule xác định rõ.

---

# 30. VBA File Operations

VBA có thể:

```text
open file
read file
write file
copy file
rename file
delete file
```

Từ góc nhìn operations, `delete` và `overwrite` là high-risk hơn `read`.

Một tool tốt nên phân biệt:

```text
read-only phase
output generation phase
destructive phase
```

và hạn chế destructive operations.

---

# 31. VBA Exception Handling

VBA sử dụng pattern như:

```vb
On Error GoTo ErrorHandler
```

Nhưng một anti-pattern rất nguy hiểm là:

```vb
On Error Resume Next
```

được sử dụng quá rộng.

Nó có thể biến:

```text
File not found
```

thành:

```text
script continues
```

và tạo report incomplete nhưng trông như thành công.

Error handling tốt phải:

```text
identify failure
record context
stop or recover intentionally
return clear status
clean up resources
```

---

# 32. Macro Security

Macro là executable code.

Vì vậy macro có thể:

```text
access files
modify files
connect external resources
launch commands
access Outlook
```

Do đó enterprise environment thường áp dụng macro security controls.

Các concept cần hiểu gồm:

```text
trusted source
signed code
trusted location
macro policy
least privilege
code review
```

Exact security settings phụ thuộc Office version, organization policy và endpoint configuration; Infrastructure Operations không nên hướng dẫn user bypass macro protection chỉ để “cho tool chạy”.

Nếu macro bị blocked, câu hỏi đầu tiên không phải:

> “Làm sao tắt security?”

mà là:

> “Macro có được approved không, source có trusted không, signature/policy có đúng không?”

---

# 33. Python fundamentals cho Operations Automation

Theo syllabus, Day 12 bao gồm:

```text
file handling
pathlib
csv
re
logging
hashlib
datetime
subprocess
JSON
error handling
reusable functions
```

Infrastructure_Operations_14_Da…

Python rất phù hợp với automation vì khả năng xử lý structured/unstructured text, file, logs và APIs.

---

# 34. File handling trong Python

Một program thường có lifecycle:

```text
Locate file
   ↓
Validate file
   ↓
Open file
   ↓
Decode
   ↓
Parse
   ↓
Process
   ↓
Write
   ↓
Close
```

Python context manager:

```python
with open(...) as f:
    ...
```

giúp resource được đóng đúng cách khi exception xảy ra.

---

# 35. pathlib

`pathlib` biểu diễn filesystem path như object.

Ví dụ:

```python
from pathlib import Path

input_file = Path("data") / "users.csv"
```

Điểm mạnh là path logic rõ ràng hơn việc nối string thủ công.

Có thể kiểm tra:

```python
input_file.exists()
input_file.is_file()
```

Nhưng file tồn tại không có nghĩa account đang chạy có quyền đọc.

---

# 36. Python csv module

CSV không phải:

```text
split each line by comma
```

CSV có quoting rules.

Ví dụ:

```csv
ID,Name,Comment
0001,"Nguyen, An","Needs review"
```

Nếu dùng:

```python
line.split(",")
```

sẽ chia `"Nguyen, An"` sai thành hai columns.

Python `csv` module xử lý CSV syntax đúng hơn.

---

# 37. re — Regular Expressions

`re` phục vụ pattern matching.

Ví dụ:

```python
import re

pattern = r"^USR\d{6}$"
```

Pattern trên nghĩa là:

```text
^        start
USR      literal text
\d       digit
{6}      exactly 6 digits
$        end
```

Ví dụ hợp lệ:

```text
USR123456
```

không hợp lệ:

```text
XUSR123456
USR123
USR1234567
```

---

# 38. Regex extraction và Regex validation khác nhau

Extraction:

```text
Find an ID somewhere inside a log line
```

Validation:

```text
Confirm entire value follows ID specification
```

Nếu dùng Regex quá rộng:

```regex
\d+
```

bạn có thể match:

```text
timestamp
port
PID
event ID
user ID
```

và extract nhầm field.

---

# 39. Python logging

`logging` giúp tạo execution trace.

Một log tốt nên có:

```text
timestamp
severity
run ID
operation
file
record count
error class
safe context
```

Không nên log:

```text
password
token
private key
session cookie
full secret
```

Ví dụ conceptual:

```text
2026-10-22T09:01:00+07:00 INFO run=R001 input=users.csv
2026-10-22T09:01:01+07:00 INFO records=2531
2026-10-22T09:01:02+07:00 ERROR row=144 reason=MISSING_ID
```

---

# 40. hashlib

`hashlib` hỗ trợ cryptographic hash như SHA-256.

Ví dụ concept:

```text
file bytes
   ↓
SHA-256
   ↓
digest
```

Hash có thể phát hiện file thay đổi so với known baseline.

Nếu:

```text
hash_before != hash_after
```

file đã khác byte-level.

Nhưng syllabus nhấn mạnh rằng hash **không tự chứng minh source authenticity hoặc completeness**. Infrastructure_Operations_14_Da…

Ví dụ file có thể được hash hoàn hảo nhưng query dùng để export đã bỏ mất 50% events.

Hash chỉ nói:

> Đây có phải cùng byte sequence với artifact đã hash hay không?

---

# 41. datetime

`datetime` cần cho:

```text
run timestamps
log timestamps
time-window filtering
file naming
evidence metadata
```

Một vấn đề operations quan trọng là timezone.

Không nên viết ambiguous timestamp:

```text
10/11/26 08:00
```

Không rõ:

```text
October 11?
November 10?
UTC?
local?
```

Evidence tốt hơn:

```text
2026-10-22T09:15:00+07:00
```

hoặc UTC khi workflow yêu cầu.

---

# 42. subprocess

`subprocess` dùng để chạy external commands.

Ví dụ Python gọi:

```text
PowerShell
command-line utility
another script
```

Điểm quan trọng là phải inspect:

```text
return code
stdout
stderr
timeout
```

Không được chỉ assume:

```text
process launched = process succeeded
```

Nếu external tool exit `5`, parent Python process phải hiểu failure.

---

# 43. subprocess và command injection

Nếu command được tạo từ untrusted input, có risk command injection.

Về nguyên tắc operational safety, nên tránh shell interpretation khi không cần và truyền arguments theo cấu trúc phù hợp thay vì concatenate raw user input thành command string.

Ví dụ unsafe conceptual pattern:

```text
"tool.exe " + user_supplied_value
```

nếu value có shell metacharacters.

Đây là **kiến thức bổ sung về secure automation**, không phải một syntax requirement riêng trong syllabus.

---

# 44. JSON

JSON thường xuất hiện trong:

```text
API responses
configuration files
tool output
metadata
```

Ví dụ:

```json
{
  "userId": "000123",
  "status": "active"
}
```

JSON có:

```text
objects
arrays
strings
numbers
booleans
null
```

Identifier vẫn nên được represented dưới dạng string nếu semantics là identifier.

---

# 45. Python Error Handling

Typical construct:

```python
try:
    ...
except SomeException:
    ...
```

Không nên viết broad handler kiểu:

```python
except:
    pass
```

vì nó biến failure thành silence.

Một automation tool nên có explicit outcome như:

```text
SUCCESS
PARTIAL
FAILED
REJECTED_INPUT
ENCODING_ERROR
DEPENDENCY_ERROR
```

tùy design.

---

# 46. Reusable Functions

Thay vì viết một script dài:

```text
300 lines top-to-bottom
```

nên tách logic:

```python
read_input()
normalize_id()
validate_record()
hash_file()
write_output()
```

Điều này giúp:

```text
testing
code review
reuse
debugging
change isolation
```

---

# 47. Windows Batch scripting

Theo syllabus:

```text
environment variables
command execution
exit codes
file operations
scheduled execution
```

Infrastructure_Operations_14_Da…

Batch thường đóng vai trò orchestration/wrapper trên Windows.

Ví dụ:

```text
Scheduler
   ↓
run_etl.bat
   ↓
python parser.py
   ↓
PowerShell export
   ↓
archive output
```

---

# 48. Environment variables trong Batch

Ví dụ:

```bat
set INPUT_DIR=C:\Ops\Input
```

Reference:

```bat
%INPUT_DIR%
```

Environment variables có thể chứa:

```text
paths
environment names
non-secret configuration
runtime options
```

Không nên mặc định dùng environment variable như secret store an toàn chỉ vì nó không hard-code trong script.

---

# 49. Paths with spaces

Một lỗi Batch kinh điển:

```text
C:\Program Files\...
```

Nếu không quote đúng, command có thể hiểu:

```text
C:\Program
```

là executable.

Operational script phải test path chứa spaces vì syllabus yêu cầu case này cụ thể. Infrastructure_Operations_14_Da…

---

# 50. Exit code

Windows process trả exit status.

Convention phổ biến:

```text
0     success
non-0 failure
```

nhưng exact meanings phụ thuộc từng tool.

Batch wrapper phải preserve child failure.

Ví dụ:

```text
Batch
  ↓
Python exits 2
  ↓
Batch incorrectly exits 0
```

Scheduled system sẽ nghĩ job thành công.

Đây là **false success**, một trong những lỗi operational nguy hiểm nhất.

---

# 51. `%ERRORLEVEL%`

Batch có thể inspect process status.

Conceptually:

```bat
python parser.py
if errorlevel 1 (
    exit /b %ERRORLEVEL%
)
```

Khi viết thực tế phải cẩn thận timing và variable expansion, nhưng concept bắt buộc là:

> Parent wrapper không được che giấu child failure.

---

# 52. Batch file operations

Batch có thể:

```text
copy
move
rename
delete
create directories
```

Thao tác `del` hoặc overwrite phải được kiểm soát.

Ví dụ không nên:

```text
delete old output
then run parser
```

nếu parser có thể fail và để lại không có output nào.

Một safer pattern có thể là:

```text
generate temporary output
verify
rename/promote
archive old version
```

tùy requirement.

---

# 53. Scheduled execution

Batch thường được chạy bởi Windows Task Scheduler.

Điểm cần hiểu:

Interactive execution và scheduled execution không giống nhau.

Scheduled task có thể khác:

```text
user identity
current working directory
mapped drives
environment variables
profile
permissions
network credentials
```

Một script chạy tốt khi double-click chưa chắc chạy tốt dưới Task Scheduler.

---

# 54. PowerShell trong Day 12

Syllabus bao phủ:

```text
objects
pipelines
CSV import/export
file encoding
event-log extraction
administrative automation
```

Infrastructure_Operations_14_Da…

PowerShell đặc biệt quan trọng vì nó là object-oriented shell rất mạnh trong Windows administration.

---

# 55. PowerShell Objects

Một điểm nền tảng:

PowerShell pipeline không chỉ truyền text.

Ví dụ conceptual:

```powershell
Get-Process
```

trả objects với properties như:

```text
Name
Id
CPU
StartTime
```

Sau đó:

```powershell
Get-Process | Where-Object ...
```

đang xử lý objects, không parse một table text đơn thuần.

Đây là khác biệt lớn so với shell text-processing truyền thống.

---

# 56. Pipeline

Pipeline:

```text
Producer
  |
  v
Filter
  |
  v
Select
  |
  v
Export
```

Ví dụ:

```powershell
Get-Something |
    Where-Object ... |
    Select-Object ... |
    Export-Csv ...
```

Mỗi stage nên có clear responsibility.

---

# 57. Import-Csv

`Import-Csv` biến CSV rows thành PowerShell objects.

Ví dụ conceptual:

```powershell
$data = Import-Csv -Path "users.csv"
```

Nếu headers là:

```text
UserID,Status
```

mỗi row có properties:

```powershell
$row.UserID
$row.Status
```

Nhưng delimiter và encoding phải phù hợp input.

---

# 58. Export-Csv

`Export-Csv` tạo structured report.

Phải kiểm soát:

```text
encoding
delimiter
field selection
output path
overwrite behavior
```

Nếu output dùng làm evidence hoặc downstream input, schema phải ổn định.

---

# 59. PowerShell encoding

Encoding behavior có thể khác theo PowerShell generation/version và cmdlet, vì vậy operational script không nên dựa vào implicit defaults nếu fidelity quan trọng.

Nguyên tắc:

> Khi encoding là requirement, hãy declare và test encoding thay vì assume default.

Điều này đặc biệt quan trọng với:

```text
Vietnamese
Japanese
UTF-16
CP932
```

---

# 60. Event-log extraction

PowerShell rất phù hợp để automate Windows Event Log extraction.

Một event record có thể bao gồm:

```text
ProviderName
Id
Level
TimeCreated
RecordId
MachineName
Message
```

Operational extraction phải xác định:

```text
which channel?
which host?
what time window?
which Event IDs/providers?
what timezone?
```

Không nên export “mọi thứ” nếu ticket chỉ yêu cầu một bounded incident window.

---

# 61. Administrative automation

PowerShell có thể thực hiện privileged operations.

Do đó cùng một script:

```text
read-only query
```

có risk thấp hơn:

```text
restart service
modify registry
change ACL
create user
```

Operational runbook nên phân loại rõ:

```text
read-only
state-changing
destructive
```

---

# 62. TXT/CSV processing — một trong các chủ đề quan trọng nhất Day 12

Syllabus yêu cầu:

```text
TXT/CSV processing
UTF-8
UTF-8 BOM
UTF-16
Shift-JIS/CP932 awareness
delimiters
quoting
line endings
string normalization
regular expressions
filename conventions
data integrity
```

Infrastructure_Operations_14_Da…

Phần additive còn nhấn mạnh:

> Separate CSV syntax from encoding and locale.

Infrastructure_Operations_14_Da…

Đây là điểm rất quan trọng.

---

# 63. Encoding là gì?

Encoding là rule mapping giữa characters và bytes.

Ví dụ chữ:

```text
A
```

phải được biểu diễn bằng byte sequence nào đó.

Các encoding khác nhau có thể map characters khác nhau.

Nếu decode bằng encoding sai:

```text
bytes
   ↓
wrong decoder
   ↓
garbled text / error
```

---

# 64. UTF-8

`UTF-8` là Unicode encoding phổ biến.

Nó có thể biểu diễn Vietnamese, Japanese và hầu hết scripts.

ASCII characters có representation tương thích ở byte-level.

UTF-8 file không bắt buộc có BOM.

---

# 65. UTF-8 BOM

Một UTF-8 file có thể bắt đầu bằng `BOM` — Byte Order Mark.

Trong UTF-8 nó thường được dùng như signature hơn là để xác định byte order.

Một parser không xử lý BOM đúng có thể đọc header đầu tiên thành kiểu:

```text
﻿UserID
```

thay vì:

```text
UserID
```

Khi đó code tìm column:

```python
row["UserID"]
```

có thể fail dù file “trông đúng” trong text editor.

---

# 66. UTF-16

UTF-16 thường có endianness consideration và BOM có thể giúp biểu thị encoding variant.

Operationally, điều quan trọng không phải thuộc byte theory sâu mà là:

```text
UTF-16 file ≠ UTF-8 file
```

Nếu đọc sai encoding, output có thể fail hoặc corrupt.

---

# 67. Shift-JIS và CP932

Đây là vùng rất dễ nhầm.

`Shift-JIS` là một Japanese character encoding family/convention.

`CP932`, thường liên quan tới Microsoft Windows Japanese code page, có relationship với Shift-JIS nhưng không nên coi chúng luôn hoàn toàn interchangeable.

Một dataset được ghi rõ `CP932` nên được xử lý theo đúng contract đó thay vì assume generic Shift-JIS.

Syllabus yêu cầu **Shift-JIS/CP932 awareness** và đặc biệt có fixture với CP932/unmappable characters. Infrastructure_Operations_14_Da…

---

# 68. Silent replacement là gì và vì sao nguy hiểm?

Suppose source bytes không decode được.

Một parser có thể thay unknown character bằng:

```text
?
```

hoặc replacement marker.

Script vẫn chạy.

Nhưng data đã thay đổi.

Nếu character nằm trong:

```text
account ID
filename
name
business key
```

reconciliation có thể sai.

Syllabus yêu cầu malformed hoặc unmappable data **không được silently replace**. Infrastructure_Operations_14_Da…

Safer behavior là:

```text
reject
quarantine
explicitly log error
```

tùy use case.

---

# 69. CSV syntax khác encoding

Một file có thể là:

```text
CSV syntax + UTF-8 encoding
```

hoặc:

```text
CSV syntax + CP932 encoding
```

CSV nói về:

```text
fields
delimiter
quoting
records
```

Encoding nói về:

```text
bytes ↔ characters
```

Hai layer độc lập.

---

# 70. Delimiter

CSV tên là comma-separated, nhưng operational datasets đôi khi dùng:

```text
,
;
tab
|
```

Vì vậy parser nên có input contract rõ ràng.

Không nên đoán delimiter một cách tùy tiện trong audit-critical workflow nếu ambiguity có thể làm sai columns.

---

# 71. Quoting

Ví dụ:

```csv
UserID,Name
0001,"Nguyen, An"
```

Comma trong quoted value không phải delimiter.

Parser đúng hiểu:

```text
Column 1 = 0001
Column 2 = Nguyen, An
```

---

# 72. Embedded quotes

CSV có thể chứa quotes trong field.

Ví dụ conceptual:

```text
User said "approved"
```

CSV escaping phải theo syntax phù hợp.

Nếu tự parse bằng simple string split, rất dễ hỏng.

---

# 73. Embedded newline

Một quoted field có thể chứa newline.

Ví dụ:

```csv
ID,Comment
0001,"Line one
Line two"
```

Logical record này chiếm nhiều physical lines.

Do đó:

```text
line count
```

không nhất thiết bằng:

```text
CSV record count
```

Đây là một subtle but important audit issue.

---

# 74. CRLF và LF

Windows thường sử dụng:

```text
CRLF
```

Unix/Linux thường:

```text
LF
```

Modern libraries thường xử lý được cả hai, nhưng custom code có thể không.

Nếu script hard-code:

```text
"\r\n"
```

nó có thể fail trên LF-only data.

Syllabus yêu cầu test cả CRLF/LF. Infrastructure_Operations_14_Da…

---

# 75. Missing columns

Parser không được assume mọi file đều có đúng schema.

Ví dụ expected:

```text
UserID,Status,Owner
```

actual:

```text
UserID,Status
```

Tool nên fail rõ:

```text
MISSING_REQUIRED_COLUMN: Owner
```

thay vì:

```text
Owner = blank
```

và tiếp tục tạo misleading results.

---

# 76. Locale

Locale có thể ảnh hưởng:

```text
date formatting
decimal separators
list separators
case rules
```

Ví dụ date:

```text
01/02/2026
```

có thể ambiguous.

Structured automation nên ưu tiên unambiguous formats.

---

# 77. String normalization

Normalization là biến input thành canonical representation để matching.

Ví dụ:

```text
" USER001 "
"user001"
"USER001"
```

Có thể normalized thành:

```text
USER001
```

nếu policy quy định:

```text
trim surrounding whitespace
case-insensitive
```

Nhưng policy phải được documented.

---

# 78. Không normalization tùy tiện

Bạn không được thấy:

```text
USER-001
USER001
```

rồi tự quyết:

> “Có vẻ cùng một account nên xóa hyphen.”

Điều đó có thể merge hai identifiers khác nhau.

Normalization phải có rule.

---

# 79. Full-width characters

Syllabus đặc biệt yêu cầu định nghĩa **full-width normalization rules**. Infrastructure_Operations_14_Da…

Ví dụ Japanese/full-width data có thể chứa characters visually similar nhưng code points khác nhau.

Việc normalize chúng có thể cần thiết trong một system, nhưng lại incorrect ở system khác.

Do đó:

```text
normalize only according to documented rule
retain original
```

---

# 80. Whitespace

Không phải whitespace nào cũng dễ nhìn.

Có thể có:

```text
space
tab
non-breaking space
Unicode whitespace
```

A simple visual inspection không đủ.

Một ID “trông giống nhau” có thể khác raw bytes/characters.

---

# 81. Case normalization

Ví dụ:

```text
AdminUser
adminuser
```

Có thể same identity trong một source nhưng different objects trong system khác.

Do đó case normalization phải dựa vào identifier semantics.

---

# 82. Filename conventions

Filename tốt giúp traceability.

Ví dụ:

```text
<system>_<artifact>_<timestamp>_<runid>.<ext>
```

Ví dụ:

```text
IAM_Exceptions_20261022T091500_R001.csv
```

Một convention tốt tránh:

```text
final.csv
final2.csv
final_final.csv
newfinal.csv
```

---

# 83. Filename sanitization

Input-controlled filenames có thể chứa characters không hợp lệ hoặc path traversal patterns.

Khi xử lý email attachments hoặc externally supplied filenames, automation không nên blindly concatenate filename vào destination path.

Cần:

```text
resolve target directory
sanitize filename
prevent directory escape
detect collisions
```

Đây là đặc biệt quan trọng với Outlook attachment handling.

---

# 84. Data Integrity

Data integrity trong Day 12 không chỉ là hash.

Phải xem nhiều dimension:

```text
byte integrity
record completeness
field fidelity
schema integrity
transformation correctness
traceability
```

Ví dụ:

```text
SHA-256 correct
```

nhưng:

```text
50 records accidentally filtered out
```

thì integrity ở business level vẫn thất bại.

---

# 85. Evidence lineage

Syllabus định nghĩa evidence contract chi tiết.

Một artifact nên trace được:

```text
Raw Source
   │
   │ SHA-256
   ▼
Working Copy
   │
   │ Script v1.3
   ▼
Normalized Output
   │
   │ reconciliation
   ▼
Exception Report
```

Metadata cần bao gồm source, system/product/version, environment, collector, collection time/timezone, query/filter, format/encoding, size/count/hash và transformation/script version. Infrastructure_Operations_14_Da…

---

# 86. Raw evidence không được overwrite

Nguyên tắc:

```text
raw source → read-only preservation
```

Processing phải dùng:

```text
working copy
```

Nếu bạn normalize raw source trực tiếp, bạn phá evidence lineage.

---

# 87. Spreadsheet Formula Injection

Syllabus yêu cầu:

> Prevent spreadsheet formula injection in presentation exports while preserving raw evidence.

Infrastructure_Operations_14_Da…

Đây là một security issue quan trọng.

Nếu user-controlled value bắt đầu bằng certain spreadsheet-interpreted characters, Excel có thể xem nó như formula thay vì plain text.

Ví dụ conceptual input:

```text
=...
+...
-...
@...
```

Khi được mở trong spreadsheet context, behavior có thể khác plain CSV viewer.

---

# 88. Raw evidence và presentation export phải tách nhau

Bạn không nên sanitize raw source rồi ghi đè.

Safer architecture:

```text
Raw evidence
    │
    ├── unchanged
    │
    ▼
Processing copy
    │
    ▼
Presentation-safe output
```

Presentation output có thể được xử lý theo approved spreadsheet-safety rule.

Raw evidence vẫn giữ nguyên byte content.

---

# 89. Secure password generation

Syllabus yêu cầu:

```text
cryptographically secure password generation
controlled credential handling
```

và additive section nói rõ:

> Never use VBA `Rnd` or a predictable seed for operational password generation.

Infrastructure_Operations_14_Da…

---

# 90. Pseudo-random và cryptographically secure random khác nhau

Không phải random generator nào cũng phù hợp với secret.

General-purpose PRNG có thể được thiết kế cho:

```text
simulation
games
sampling
```

không phải để chống prediction.

Password generation cần nguồn random có security properties phù hợp.

---

# 91. Password entropy — hiểu ở mức Operations

Security không đến từ việc password “trông phức tạp”.

Ví dụ:

```text
P@ssw0rd2026!
```

có nhiều character classes nhưng vẫn predictable.

Password generator cần:

```text
secure randomness
adequate length
policy compliance
```

chứ không chỉ thay:

```text
a → @
o → 0
```

---

# 92. Secure source

Trong Python, standard-library mechanisms dành cho secrets là ví dụ phù hợp hơn general-purpose random.

Trong Windows/.NET environments cũng tồn tại cryptographically secure random facilities.

Syllabus không yêu cầu một implementation duy nhất; requirement là **approved cryptographically secure source**.

---

# 93. Fail closed

Nếu secure generator không khả dụng:

```text
do not fallback to weak generator
```

Ví dụ sai:

```text
secure generator failed
   ↓
use timestamp as password
```

Safer outcome:

```text
generation FAILED
```

---

# 94. Password policy test

Tool có thể cần kiểm tra:

```text
minimum length
required character sets
forbidden characters
maximum constraints
```

Nhưng policy là organization-specific.

Không được assume một universal rule kiểu:

```text
12 chars + uppercase + lowercase + number + symbol
```

nếu enterprise policy khác.

---

# 95. Generated-password evidence phải masked

Ví dụ evidence đúng:

```text
Password generated: YES
Length: 20
Policy validation: PASS
Secret displayed: NO
```

Không đúng:

```text
Password = V7!...
```

Syllabus yêu cầu disposable credentials và masked evidence; operational passwords không được lưu trong workbooks, CSVs hoặc mail. Infrastructure_Operations_14_Da…

---

# 96. Credential handling

Credential có thể gồm:

```text
password
API token
access key
private key
session cookie
OAuth token
```

Operational automation phải tránh:

```text
hard-code secret
echo secret
log secret
put secret into CSV
put secret into workbook
attach secret to email
commit secret into source
```

---

# 97. Least privilege cho automation identity

Một script chỉ cần:

```text
read Event Log
```

không nên chạy bằng identity có:

```text
Domain Admin
```

nếu không cần.

Automation account nên chỉ có rights cần thiết.

Điều này giảm blast radius nếu script lỗi hoặc bị compromise.

---

# 98. Safe Script Execution

Syllabus yêu cầu **safe script execution** như một explicit topic. Infrastructure_Operations_14_Da…

Safe execution bắt đầu trước khi chạy command.

Bạn phải biết:

```text
Which environment?
Which host?
Which account?
Which input?
Which version?
What permissions?
What side effects?
What rollback?
```

---

# 99. Read-only vs state-changing

Đây là classification rất quan trọng.

Ví dụ:

```powershell
Get-Service
```

là read-oriented.

Còn:

```powershell
Restart-Service
```

thay đổi state.

Automation design cần biết boundary này.

---

# 100. Idempotency

`Idempotency` là property khi chạy operation nhiều lần không tạo uncontrolled repeated side effects.

Ví dụ:

```text
Ensure folder exists
```

thường có thể idempotent.

Nhưng:

```text
Append same record to CSV
```

mỗi lần chạy có thể tạo duplicate.

Repeatability tests của syllabus liên quan trực tiếp đến vấn đề này. Infrastructure_Operations_14_Da…

---

# 101. Dry-run

Một tool có thể hỗ trợ `dry-run` để mô phỏng changes.

Ví dụ:

```text
Would disable account A
Would remove role B
Would create file C
```

mà không thực hiện.

Syllabus Day 13 nhấn mạnh dry-run testing; Day 12 xây nền tảng tư duy này. Infrastructure_Operations_14_Da…

---

# 102. Partial failure

Một trong những vấn đề khó nhất trong automation:

```text
100 operations planned
67 succeeded
68th failed
```

Tool làm gì?

Có thể:

```text
rollback first 67
```

hoặc:

```text
stop and preserve partial state
```

hoặc:

```text
continue safe independent records
```

Không có universal answer.

Nhưng behavior phải explicit và documented.

---

# 103. File locking

Tool có thể fail vì file đang mở.

Ví dụ:

```text
Excel workbook locked
CSV opened exclusively
output directory read-only
```

Không nên transform:

```text
Permission denied
```

thành generic:

```text
Something went wrong
```

Diagnostic context phải đủ để operator hành động.

---

# 104. Working directory

Một script dùng:

```text
.\input.csv
```

phụ thuộc current working directory.

Khi chạy từ terminal:

```text
C:\Ops
```

nó thành:

```text
C:\Ops\input.csv
```

Nhưng Task Scheduler có thể có working directory khác.

Do đó explicit path resolution rất quan trọng.

---

# 105. Hard-coded paths

Hard-coded path:

```text
C:\Users\Alice\Desktop\data.csv
```

là operational smell.

Nó có thể fail khi:

```text
different user
different server
service account
scheduler
DR environment
```

Better design thường đưa path vào:

```text
configuration
argument
known relative directory
```

---

# 106. Hard-coded assumptions

Không chỉ path.

Ví dụ:

```text
Sheet1 always exists
column 4 is always UserID
mail folder is always Inbox\Test
all IDs are numeric
all text is UTF-8
all runs are interactive
```

Day 12 yêu cầu bạn nhận diện các unsafe assumptions này. Infrastructure_Operations_14_Da…

---

# 107. Outlook classic COM/MAPI

Theo syllabus, Day 12 bao gồm:

```text
Outlook classic COM/MAPI object model
mailbox/folder access
message processing
attachment processing
automated report preparation
security prompts
compatibility awareness with new Outlook
```

Infrastructure_Operations_14_Da…

Đây là phần đặc biệt vì technology phụ thuộc Outlook client architecture.

---

# 108. Classic Outlook Object Model

Conceptual hierarchy:

```text
Outlook.Application
        │
        ▼
    NameSpace
        │
        ▼
     Folders
        │
        ▼
      Items
        │
        ├── MailItem
        ├── AppointmentItem
        └── ...
```

Attachments nằm dưới message/item liên quan.

---

# 109. `GetNamespace("MAPI")`

Syllabus additive explicitly yêu cầu lab sử dụng:

```text
GetNamespace("MAPI")
```

để truy cập test folder qua Outlook object model. Infrastructure_Operations_14_Da…

Conceptually:

```text
Application
    ↓
MAPI namespace
    ↓
mail store
    ↓
folder
    ↓
messages
```

---

# 110. MAPI là gì trong context này?

`MAPI` = Messaging Application Programming Interface.

Trong classic Outlook automation context, MAPI namespace cho phép Outlook Object Model truy cập mailbox/folder hierarchy theo capabilities của client/profile.

Bạn không nên đồng nhất:

```text
MAPI
Microsoft Graph
IMAP
SMTP
```

Chúng là các mechanisms/interfaces khác nhau.

---

# 111. Mailbox/folder access

Automation phải chọn đúng folder.

Ví dụ:

```text
Mailbox
└── Inbox
    └── Operations
        └── Evidence
```

Hard-code folder name có risk:

```text
localized names
folder renamed
multiple mailboxes
missing folder
```

Nên validate folder existence trước processing.

---

# 112. Message filtering

Không nên process toàn mailbox nếu requirement chỉ là:

```text
approved test folder
```

Filter có thể dựa trên:

```text
folder
subject
sender
timestamp
attachment criteria
```

Nhưng criteria phải được documented.

---

# 113. Attachment Processing

Attachment processing tạo nhiều risks:

```text
malicious file
unexpected extension
duplicate filename
path traversal
oversized file
overwrite existing file
```

Syllabus yêu cầu allowlisted attachment processing và filename-collision handling. Infrastructure_Operations_14_Da…

---

# 114. Allowlist

Thay vì:

```text
save everything
```

workflow có thể chỉ cho phép expected types, tùy approved requirement.

Ví dụ conceptual:

```text
.csv
.txt
.evtx
```

nhưng allowlist thực tế phải theo use case.

---

# 115. Filename collision

Hai messages có thể cùng attachment:

```text
report.csv
```

Nếu save lần hai bằng same path:

```text
overwrite
```

evidence thứ nhất mất.

Safer strategies gồm:

```text
unique run directory
timestamp suffix
collision counter
message ID-derived name
```

tùy design.

---

# 116. Automated report preparation

Day 12 yêu cầu Outlook workflow cuối cùng chỉ tạo:

```text
verified draft
```

không auto-send. Infrastructure_Operations_14_Da…

Điều này là operational safety boundary rất quan trọng.

Workflow:

```text
Generate report
   ↓
Validate report
   ↓
Create Draft
   ↓
Attach artifacts
   ↓
Validate recipients
   ↓
Validate attachment names
   ↓
STOP
```

Human/operator có cơ hội review trước khi gửi.

---

# 117. Security prompts

Classic Outlook/Office automation có thể chịu ảnh hưởng từ security policy, trust settings, antivirus integration hoặc programmatic-access controls.

Nếu automation bị security prompt/block, không nên circumvent security control không được phê duyệt.

Bạn cần xác định:

```text
expected control?
client configuration?
organization policy?
unsupported automation method?
```

---

# 118. New Outlook compatibility

Syllabus ghi rõ classic Outlook là dependency và new Outlook không hỗ trợ VBA/macros cho workflow này. Infrastructure_Operations_14_Da…

Tôi đã kiểm tra lại tài liệu Microsoft hiện hành: Microsoft vẫn ghi nhận **VBA Macros không được hỗ trợ trong new Outlook**, trong khi classic Outlook hỗ trợ chúng. Microsoft cũng ghi `Outlook Object Model (OOM)` và `MAPI` là available trong classic Outlook nhưng không được hỗ trợ theo cùng mô hình trong new Outlook. [Microsoft Learn](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/vba-alternatives?utm_source=chatgpt.com)

Do đó Day 12 phải xác định rõ:

```text
classic Outlook compatible lab
```

chứ không assume workflow COM/MAPI/VBA sẽ hoạt động trên new Outlook.

---

# 119. Alternatives không thuộc lab chính nhưng cần nhận thức

Microsoft nêu các hướng thay thế cho certain new Outlook integration scenarios như:

````text
Power Automate
Microsoft Graph
Office.js / web add-ins
``` :chatgpt-content-reference{index="32"}


Đây là **kiến thức bổ sung**, không có nghĩa Day 12 thay lab classic Outlook/MAPI bằng Microsoft Graph. Syllabus vẫn xác định classic Outlook/MAPI là lab target.

---

# 120. Cross-language automation architecture

Một operations workflow hiếm khi chỉ có một technology.

Bạn có thể thấy:

```text
Task Scheduler
      │
      ▼
   Batch
      │
      ▼
   Python
      │
      ├─────► CSV
      │
      ▼
 PowerShell
      │
      ▼
   Excel/VBA
      │
      ▼
 Outlook draft
````

Điều quan trọng là **error propagation** giữa các layer.

---

# 121. Error propagation

Ví dụ:

```text
Python fails
   ↓
Batch ignores exit code
   ↓
PowerShell runs anyway
   ↓
Excel report generated from stale file
   ↓
Outlook creates draft
```

End user thấy một draft “thành công”.

Nhưng entire process đã fail từ stage đầu.

Do đó mỗi stage phải có:

```text
clear success condition
clear failure condition
validated input
validated output
```

---

# 122. Stale output là một risk lớn

Một common failure:

```text
Yesterday's output exists.
Today's parser fails.
Report process reads yesterday's output.
```

Nếu không kiểm tra timestamp/run ID, report có vẻ valid.

Mitigation concept:

```text
run-specific output
run ID
timestamp verification
expected hash/count
```

---

# 123. Contract giữa tools

Mỗi tool nên có input/output contract.

Ví dụ Python parser:

```text
Input:
UTF-8 CSV
Required headers: UserID,Status

Output:
UTF-8 CSV
Headers: UserID,NormalizedID,Result

Exit:
0 success
2 validation failure
3 encoding failure
```

Sau đó Batch wrapper biết cách handle từng outcome.

---

# 124. Record counts

Một useful reconciliation:

```text
Input records
=
Accepted
+ Rejected
```

hoặc:

```text
Input
=
Matched
+ Unmatched
+ Duplicate
+ Rejected
```

tùy counting rule.

Nếu:

```text
Input = 10,000
Output accounted = 9,997
```

thì 3 records đang mất.

Tool không được coi success cho tới khi discrepancy được giải thích.

---

# 125. Golden Dataset

Syllabus yêu cầu test bằng **golden dataset**.

Golden dataset là test data có expected answer đã biết độc lập.

Ví dụ:

```text
100 input rows
```

answer key:

```text
80 matched
5 unmatched
6 duplicates
4 blank IDs
5 malformed rows
```

Tool phải tạo kết quả đúng.

---

# 126. Không test program bằng chính program

Một anti-pattern:

```text
Program output = 95 exceptions

Expected output = copy of program output
```

Test như vậy không chứng minh gì.

Syllabus nói tests phải sử dụng trusted expected-output dataset hoặc independently reviewed assertions. Infrastructure_Operations_14_Da…

---

# 127. Positive testing

Positive case hỏi:

> Tool có xử lý input hợp lệ đúng không?

Ví dụ:

```text
well-formed UTF-8 CSV
valid IDs
all mandatory columns present
```

Expected:

```text
SUCCESS
```

---

# 128. Negative testing

Negative case intentionally đưa condition không hợp lệ:

```text
missing file
wrong permission
missing header
malformed bytes
```

Tool phải:

```text
fail clearly
not corrupt data
not claim success
```

---

# 129. Boundary testing

Boundary test kiểm tra edge conditions.

Ví dụ:

```text
empty file
one row
very long ID
ID exactly maximum length
zero records
path containing spaces
```

---

# 130. Encoding testing

Syllabus yêu cầu:

```text
UTF-8 with BOM
UTF-8 without BOM
UTF-16
CP932
quoted fields
embedded newline
CRLF/LF
malformed bytes
```

Infrastructure_Operations_14_Da…

Đây không phải optional “nice-to-have”.

Nó là minimum automation test catalog của curriculum.

---

# 131. File/process failure testing

Curriculum cũng yêu cầu:

```text
missing file
locked file
permission denied
empty input
missing header
path with spaces
child-script failure
```

Acceptance là failure phải observable, log phải hữu ích và script không được tạo misleading success output. Infrastructure_Operations_14_Da…

---

# 132. Password tests

Minimum catalog còn yêu cầu:

```text
valid password policy
invalid policy
secure random provider failure
secret leakage test
```

Acceptance:

```text
approved secure generator
fail-closed behavior
no plaintext secret in report/log/workbook
```

Infrastructure_Operations_14_Da…

---

# 133. Outlook tests

Curriculum yêu cầu test:

```text
missing folder
missing attachment
duplicate filename
disallowed attachment path/type
unavailable profile
```

và không auto-send. Infrastructure_Operations_14_Da…

---

# 134. Repeatability

Một tool tốt phải chạy lại được.

Nếu input giống nhau và business rules giống nhau thì business result nên stable.

Có thể khác:

```text
run timestamp
run ID
log filename
```

nhưng classification không được random thay đổi.

---

# 135. Determinism

Ví dụ:

```text
same 100 accounts
same policy
same source state
```

không nên lần đầu:

```text
10 exceptions
```

lần sau:

```text
13 exceptions
```

nếu không có lý do.

Nondeterminism làm audit khó defend.

---

# 136. Rollback-ready versions

Đây là khác biệt giữa “script development” và “operations maintenance”.

Trước modification:

```text
original script
```

phải được preserved.

Sau modification:

```text
modified script
```

phải có version/diff.

Rollback package phải cho phép:

```text
restore original
rerun known validation
confirm behavior
```

---

# 137. Source Diff

`Diff` cho thấy chính xác thay đổi.

Ví dụ:

```diff
- Open file using default encoding
+ Open file using configured encoding
```

Reviewer có thể nhanh chóng đánh giá risk.

Một 2-line fix nên có 2-line diff, không phải 500-line formatting change.

---

# 138. Verification khác Testing

Hai term dễ bị nhầm.

`Testing` hỏi:

> Tool có hoạt động đúng trên test cases không?

`Verification` hỏi:

> Sau execution/deployment, expected operational state có thực sự đạt không?

Ví dụ test:

```text
parser generated report
```

Verification:

```text
record count reconciles
hash recorded
exceptions equal golden answer
source unchanged
```

---

# 139. Troubleshooting workflow

Operations troubleshooting nên đi từ low-risk/read-only đến higher-risk.

Một flow tốt:

```text
Confirm symptom
       ↓
Confirm input/version/environment
       ↓
Check logs
       ↓
Check exit code
       ↓
Check dependency
       ↓
Check permissions
       ↓
Check encoding/schema
       ↓
Reproduce with safe fixture
       ↓
Inspect code
       ↓
Apply controlled change
       ↓
Regression test
```

Không nên bắt đầu bằng:

```text
rewrite script
disable security
run as Administrator
```

---

# 140. “Run as Administrator” không phải universal fix

Nếu tool fail access:

```text
permission denied
```

thì chạy elevated có thể làm nó chạy.

Nhưng điều đó không chứng minh quyền elevated là đúng.

Bạn cần xác định:

```text
What permission is required?
Why?
To which object?
For how long?
```

Đó là least privilege.

---

# 141. Logging và secret leakage

Error path cũng phải mask secrets.

Developer thường nhớ mask success log:

```text
Token = ********
```

nhưng exception có thể dump:

```text
request headers
environment
command line
stack trace with secret
```

Curriculum nhấn mạnh secrets phải masked trong cả successful lẫn error paths. Infrastructure_Operations_14_Da…

---

# 142. Execution metadata

Một reproducible run cần biết:

```text
run ID
script version
input version/hash
start time
end time
host
account/collector
parameters
result
output files
```

Không nhất thiết mọi tool phải log tất cả fields vào một file duy nhất, nhưng evidence package phải đủ để reconstruct run.

---

# 143. Reproducibility

Một reviewer khác phải có khả năng hiểu:

```text
Which script?
Which input?
Which parameters?
Which environment?
Which result?
```

Syllabus yêu cầu reproducible run instruction cho từng language/tool. Infrastructure_Operations_14_Da…

---

# 144. Security implications tổng hợp

Day 12 có security surface rất lớn vì automation có thể nối nhiều systems.

Risk landscape:

| Area                | Security risk                          |
| ------------------- | -------------------------------------- |
| Excel               | Formula injection, macro execution     |
| VBA                 | File/system access                     |
| Python              | Command/file/API abuse                 |
| Batch               | Injection, quoting, elevated execution |
| PowerShell          | Administrative changes                 |
| CSV                 | Malicious/ambiguous input              |
| Password generation | Predictable secret                     |
| Logs                | Credential leakage                     |
| Outlook             | Data exfiltration, unintended send     |
| Attachments         | Malicious files/path traversal         |
| Scheduler           | Overprivileged service account         |

Do đó automation phải được coi là security-sensitive infrastructure.

---

# 145. Least privilege model

Có bốn privilege dimensions nên nghĩ tới:

```text
Identity privilege
File privilege
System privilege
Network/application privilege
```

Ví dụ Outlook report generator chỉ cần:

```text
read test folder
create draft
```

không nhất thiết cần:

```text
mailbox-wide delete
automatic send
```

---

# 146. Common failure modes của Day 12

Các failure patterns xuyên suốt syllabus có thể quy về một số nhóm chính.

| Failure                | Symptom                 | Typical root cause             |
| ---------------------- | ----------------------- | ------------------------------ |
| Leading zeros lost     | IDs không match         | Numeric conversion             |
| Garbled text           | Japanese/Vietnamese lỗi | Wrong encoding                 |
| Column not found       | Parser exception        | BOM/missing header             |
| CSV rows shifted       | Wrong columns           | Naive delimiter parsing        |
| False success          | Scheduler says success  | Exit code swallowed            |
| Stale report           | Wrong data date         | Old output reused              |
| Duplicate loss         | Missing records         | Dictionary overwrite           |
| Workbook corruption    | Source modified         | No raw/working separation      |
| Secret leakage         | Password visible        | Unsafe logging                 |
| Attachment overwrite   | Missing evidence        | Filename collision             |
| Macro blocked          | Tool won't run          | Policy/trust issue             |
| Scheduled job fails    | Works manually          | Different identity/environment |
| Excel formula executes | Unexpected behavior     | Formula injection              |
| Infinite/slow run      | Process hangs           | Loop/data assumptions          |
| Partial output         | Counts mismatch         | Silent exception               |

---

# 147. Troubleshooting principle: symptom không phải root cause

Ví dụ:

```text
Lookup returns no match
```

Root cause có thể là:

```text
different ID
leading zero lost
case difference
whitespace
wrong sheet
wrong workbook version
encoding corruption
duplicate ambiguity
```

Do đó không nên ngay lập tức sửa lookup formula.

---

# 148. Evidence cần thu thập khi troubleshoot

Một investigation tốt thường cần:

```text
input hash
input row count
output hash
output row count
script version
command
parameters
exit code
log extract
error message
environment
timestamp
```

Nhưng phải tránh collect secret.

---

# 149. Escalation boundary

L1/L2 Infrastructure Operations có thể thường xử lý trong approved scope:

```text
verify inputs
check files
check encoding
check permissions
reproduce known failure
collect logs
apply approved low-risk fix
rollback approved change
```

Nên escalate khi:

```text
business rule ambiguous
production data at risk
unknown secret exposure
unapproved privilege required
code architecture requires major redesign
security control would need bypass
data corruption cannot be bounded
mail was unintentionally sent externally
evidence integrity cannot be established
```

Exact support boundary vẫn phải theo organization runbook.

---

# 150. Mối liên hệ Day 12 → Day 13

Day 12 dạy **tooling mechanics**.

Day 13 áp dụng chúng vào:

```text
90-day IAM review
cross-system identity reconciliation
multi-sheet automation
vulnerability tracking
audit trail
test design
```

Syllabus Day 13 yêu cầu workbook gồm nhiều source sheets và reconciliation/decision layers, đồng thời source totals phải reconcile và seeded exceptions phải đúng answer key. Infrastructure_Operations_14_Da…

Nếu Day 12 chưa vững:

```text
ID handling
encoding
duplicate logic
error propagation
evidence integrity
regression testing
```

thì Day 13 rất dễ tạo một audit report nhìn đẹp nhưng sai dữ liệu.

---

# 151. Mental model tổng hợp của toàn bộ Module Lý thuyết

Toàn bộ Day 12 có thể gom vào một end-to-end control flow:

```text
                    REQUIREMENT
                        │
                        ▼
              Understand Existing Tool
                        │
           ┌────────────┴────────────┐
           ▼                         ▼
     Dependency Map             Risk Analysis
           │                         │
           └────────────┬────────────┘
                        ▼
                     INPUT
                        │
             Preserve Raw Source
                        │
                        ▼
                 Validate Schema
                        │
                        ▼
                Decode Encoding
                        │
                        ▼
             Normalize According
               to Explicit Rules
                        │
                        ▼
               Process / Reconcile
                        │
            ┌───────────┴───────────┐
            ▼                       ▼
       Valid Records            Exceptions
            │                       │
            └───────────┬───────────┘
                        ▼
                    OUTPUT
                        │
           Count + Hash + Metadata
                        │
                        ▼
                    TESTING
         ┌──────────────┼──────────────┐
         ▼              ▼              ▼
      Positive       Negative       Boundary
         │              │              │
         └──────────────┼──────────────┘
                        ▼
                 Regression Test
                        │
                        ▼
              Golden-data Compare
                        │
               ┌────────┴────────┐
               ▼                 ▼
             PASS              FAIL
               │                 │
               ▼                 ▼
          Verification       Troubleshoot
               │
               ▼
          Evidence Package
               │
               ▼
           Change Record
               │
               ▼
        Rollback-ready State
```

Đây chính là tư duy mà bạn cần mang sang Day 13 và Day 14.

---

# 152. Những nguyên tắc phải nhớ sau Module Lý thuyết Day 12

Nếu phải rút toàn bộ module xuống thành các operational rules cốt lõi thì chúng là:

| Principle                         | Ý nghĩa                                      |
| --------------------------------- | -------------------------------------------- |
| Preserve raw evidence             | Không overwrite source                       |
| Treat identifiers as text         | Không làm mất leading zeros/precision        |
| Normalize explicitly              | Không tự đoán business rules                 |
| Encoding is a contract            | Không assume UTF-8                           |
| CSV syntax ≠ encoding             | Hai layer khác nhau                          |
| Never silently corrupt data       | Reject/quarantine malformed input            |
| Propagate failure                 | Child failure không được biến thành success  |
| Test negative paths               | Happy path chưa đủ                           |
| Compare against independent truth | Golden dataset/answer key                    |
| Protect secrets                   | Không log/workbook/mail plaintext credential |
| Use secure randomness             | Không dùng predictable PRNG cho password     |
| Least privilege                   | Automation chỉ có quyền cần thiết            |
| Make side effects explicit        | Biết tool thay đổi gì                        |
| Verify counts and hashes          | Hash một mình chưa đủ                        |
| Keep rollback version             | Change phải reversible                       |
| Draft before send                 | Outlook lab không auto-send                  |
| Test repeatability                | Same input phải cho stable business result   |
| Document dependencies             | Tool phải reproducible                       |
| Escalate ambiguity                | Không invent business rules                  |

---

# 153. Tiêu chuẩn hiểu bài của Module Lý thuyết Day 12

Sau khi nắm chắc phần lý thuyết, bạn phải có thể nhìn một operations tool và giải thích được toàn bộ chuỗi:

```text
What does it read?
What does it change?
What does it produce?
Which identity runs it?
Which permissions does it need?
Which encoding does it expect?
What happens to IDs?
How are duplicates handled?
What happens on malformed input?
How are failures propagated?
Where are logs?
Could secrets leak?
How is correctness tested?
How are counts reconciled?
How is integrity checked?
What is the rollback?
When should operations stop and escalate?
```

Nếu bạn chưa trả lời được những câu hỏi đó thì chưa nên chạy tool trên một operational dataset thực.

Day 12 về bản chất không dạy bạn “viết vài macro hoặc script”. Nó dạy bạn cách **maintain automation như một production operational component**: hiểu dependency, giữ nguyên raw evidence, kiểm soát data transformation, xử lý encoding và identifiers chính xác, propagate failure, kiểm thử deterministic behavior, bảo vệ credential, tạo evidence và luôn có rollback.

Phạm vi này bám đầy đủ vào Concept/Lecture và Additive Concept/Lecture của Day 12 trong syllabus, đồng thời chuẩn bị trực tiếp cho multi-sheet IAM reconciliation và automation testing của Day 13. Infrastructure_Operations_14_Da… Infrastructure_Operations_14_Da…
