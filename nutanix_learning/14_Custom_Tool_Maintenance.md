> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# Bảo trì công cụ riêng: VBA/Python/Batch, CSV/log/string/Regex

## 1. Liên hệ với dự án
**[Nguồn dự án]** SOW có Excel `.xlsm` tạo CLI/material cho LB/FW, Python/API tool, email extraction macro, access-log transcription tool, batch `huGetDiskInfo.bat`, password/string generation batch, CSV email → Excel, và task “tạo/chỉnh sửa script cho Nutanix”. Kế hoạch ước tính HCNET/Nutanix có 9 vendor-specific tools, nhưng nguồn chưa liệt kê chính thức đủ 9 tên.

## 2. Mục tiêu khi nhận bàn giao tool cũ
Đừng bắt đầu bằng refactor. Trước tiên tạo **behavioral baseline**:
- Input là gì?
- Output là gì?
- Side effect là gì?
- Dependency/version?
- Credential lấy ở đâu?
- Error được biểu hiện thế nào?
- Ai dùng và ở bước nào trong procedure?

## 3. Bảy bước đọc tool chưa biết
1. Chạy trong lab với fixture nhỏ.
2. Xác định entry point.
3. Trace input parsing.
4. Trace transformation/business rules.
5. Trace external I/O/API/files.
6. Trace error handling/logging.
7. Viết test trước khi sửa.

## 4. VBA/Excel `.xlsm`
### Cần biết
- Workbook/Worksheet/Range object model.
- Module, Sub, Function.
- Event macro: Workbook_Open, Worksheet_Change.
- `Option Explicit`.
- `ThisWorkbook` vs `ActiveWorkbook`.
- `Value2` vs `Value`.
- Error handling `On Error GoTo`.

### Rủi ro thường gặp
- Hard-coded column index/path.
- `Select/Activate` phụ thuộc active sheet.
- Locale date/decimal.
- Hidden sheet/named range.
- Macro security/trusted location.
- String command generation không escape quote.

### Checklist sửa
- Copy file versioned.
- Export modules/source nếu process cho phép.
- Test fixture workbook.
- Compare generated CLI before/after.
- Validate empty/null/duplicate/IPv4/port.
- Không tự động execute command từ Excel nếu current design chỉ generate material.

## 5. Python
### Structure khuyến nghị
```python
from pathlib import Path
import csv
import logging

def parse_input(path: Path): ...
def validate(rows): ...
def transform(rows): ...
def write_output(rows, path: Path): ...

def main(): ...

if __name__ == '__main__':
    main()
```
Tách I/O khỏi logic để test dễ.

### Essentials
- `pathlib` cho path.
- `csv` thay split dấu phẩy thủ công.
- `logging` thay print.
- `argparse` cho CLI.
- `re` cho regex.
- `json` cho API data.
- `requests` nếu môi trường approved; nếu không, stdlib/approved package.

## 6. Windows Batch
Các điểm dễ lỗi:
- `%VAR%` expansion.
- delayed expansion `!VAR!`.
- quoting path có space.
- `%ERRORLEVEL%`.
- `setlocal/endlocal`.
- encoding/code page.
- `for /f` tokenization.

Mẫu:
```bat
@echo off
setlocal
some_command.exe
if errorlevel 1 (
  echo ERROR: command failed 1>&2
  exit /b 1
)
echo OK
exit /b 0
```

## 7. CSV
Không parse CSV bằng `line.split(',')` vì có quoted comma/newline/escape.

Python:
```python
import csv
with open('input.csv', newline='', encoding='utf-8-sig') as f:
    for row in csv.DictReader(f):
        print(row)
```

Cần xác nhận encoding: UTF-8, UTF-8 BOM, Shift-JIS/CP932 có thể gặp trong môi trường Nhật.

## 8. String handling
Luôn phân biệt:
- `None`/empty/whitespace.
- Unicode full-width/half-width.
- CRLF vs LF.
- trailing spaces.
- case sensitivity.
- normalization.

Không tự `.strip()` mọi field nếu trailing space có ý nghĩa.

## 9. Regex
### Core
- `^` / `$`: start/end.
- `\d`, `\s`, `\w`.
- `[]` char class.
- `()` group.
- `(?:)` non-capturing.
- `?`, `*`, `+`, `{m,n}`.
- greedy vs lazy.

Ví dụ parse firewall log tối giản:
```python
pat = re.compile(r'(?P<src>\d+\.\d+\.\d+\.\d+):(?P<srcport>\d+)\s+->\s+(?P<dst>\d+\.\d+\.\d+\.\d+):(?P<dstport>\d+)')
```
Không dùng regex để parse format có parser chuẩn như JSON/XML/CSV.

## 10. Log parsing
Chuẩn hóa:
- timestamp/timezone,
- severity,
- host/device,
- event type,
- correlation ID/ticket,
- raw line.

Không làm mất raw evidence khi transform.

## 11. Testing strategy
### Unit test
Test function thuần: validation, parser, command generation.

### Golden-file test
Input cố định → output text/CLI expected. Rất phù hợp với tool generate material.

### Negative test
- empty input,
- malformed IP,
- invalid port,
- duplicate object,
- encoding lỗi,
- API timeout,
- permission denied.

### Regression test
Mỗi bug production → thêm fixture/test để bug không quay lại.

## 12. Safe change workflow cho tool riêng
1. Freeze baseline version/checksum.
2. Reproduce current behavior.
3. Create test cases.
4. Make smallest change.
5. Run automated tests.
6. Compare output/diff.
7. User acceptance in STG.
8. Version/tag/release note.
9. Rollback package.
10. Update procedure.

## 13. Security
- Không hard-code password/token.
- Không log secret/private key.
- Sanitize command injection nếu input đi vào shell/CLI.
- Validate file path chống ghi nhầm.
- Pin dependency/version theo policy.

## 14. Lab gắn dự án
- CSV request → validate → generate BIG-IP/FortiGate candidate config.
- Regex parse syslog nhưng giữ raw line.
- VBA workbook tạo command, Python test output.
- Batch wrapper gọi tool và trả exit code chuẩn.
- Viết test cho full-width Japanese characters và CP932.
