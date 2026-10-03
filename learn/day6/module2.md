# Day 6 — Module 2: Bash Scripting, Exit Codes & Cron Automation

## 1. Syllabus Alignment

Module 2 tiếp tục đúng phạm vi **Day 6 — Linux Fundamentals** trong MASTER_SYLLABUS:

> **Concept/Lecture:** Logs with `journalctl`, `grep/awk/sed`, **Bash scripting, exit codes, cron**, backup using `tar/rsync`, and Linux consolidation.  
> **Assignment/Lab:** Build a **health-check and backup script**, **schedule it**, inject a failure, and document escalation evidence. :chatgpt-content-reference{index="0"}

Module 1 vừa học giải quyết câu hỏi:

```text
"Chuyện gì đang xảy ra trên Linux?"
```

Module 2 sẽ giải quyết câu hỏi tiếp theo:

```text
"Làm sao biến những bước kiểm tra thủ công đó
thành một chương trình nhỏ có logic,
trả kết quả rõ ràng,
và tự động chạy theo lịch?"
```

Mental model của cả module:

```text
Linux commands
     │
     ▼
Bash script
     │
     ├── variables
     ├── conditions
     ├── functions
     ├── loops
     ├── command execution
     │
     ▼
Exit status
     │
     ├── 0     → expected success
     └── != 0  → failure / special condition
     │
     ▼
Health decision
     │
     ▼
Evidence / log
     │
     ▼
cron
     │
     ▼
Scheduled unattended execution
```

Đây là bước chuyển rất quan trọng từ:

```text
"biết Linux commands"
```

sang:

```text
"biết vận hành Linux"
```

---

# 2. Phạm vi Mandatory và Supplementary

Phần **Mandatory của syllabus** trong Module 2 là Bash scripting, exit codes, cron và sử dụng chúng để xây health-check script có thể schedule.

Tôi cũng sẽ dạy thêm một số nội dung thực tế như `set -u`, `pipefail`, `trap`, locking, `bash -n`, debugging, timeout awareness, cron hardening và idempotency. Những phần đó sẽ được đánh dấu:

> **Supplementary / Beyond the explicit syllabus**

Chúng không thay thế kiến thức bắt buộc.

Các ví dụ Bash dưới đây dựa trên GNU Bash hiện hành; tài liệu GNU hiện tại là Bash Reference Manual 5.3. :chatgpt-content-reference{index="1"}

---

# 3. Mục tiêu sau Module 2

Sau module này, bạn phải có khả năng nhìn một chuỗi commands như:

```bash
hostname
uptime
df -P /
systemctl is-active ssh.service
ss -lnt
journalctl -p err -b
```

và biến chúng thành một script có thể trả lời có cấu trúc:

```text
HOST       : lab01
SERVICE    : OK
DISK       : OK
PORT       : OK
LOG CHECK  : WARNING
OVERALL    : WARNING
```

sau đó:

```text
script
   ↓
exit status
   ↓
cron
   ↓
log/evidence
```

Quan trọng hơn, khi script fail bạn phải có khả năng trả lời:

```text
Script không chạy?
Hay chạy nhưng check thất bại?

Cron không trigger?
Hay cron trigger nhưng environment khác?

Command không tồn tại?
Hay permission denied?

Pipeline có command fail nhưng exit code bị che mất?

Script chạy trùng nhau?
Hay script treo?
```

---

# 4. Bash là gì và tại sao Admin cần Bash?

Bash là một shell đồng thời là một command language interpreter. Ngoài chạy commands, Bash cung cấp variables, quoting, control flow, functions, redirection và những construct giúp kết hợp commands thành chương trình. :chatgpt-content-reference{index="2"}

Ví dụ bạn làm thủ công:

```bash
df -P /
```

nhìn:

```text
Use% = 92%
```

rồi tự quyết định:

```text
92 >= 80
→ WARNING
```

Script sẽ biến quá trình suy nghĩ đó thành:

```text
command
   ↓
capture value
   ↓
compare threshold
   ↓
print status
   ↓
return exit code
```

Đây là automation.

---

# 5. Shell script đầu tiên

Tạo workspace an toàn:

```bash
mkdir -p "$HOME/day6-module2"
cd "$HOME/day6-module2"
```

Tạo file:

```bash
nano hello.sh
```

Nội dung:

```bash
#!/bin/bash

printf 'Hello from Bash\n'
hostname
date
```

Cho executable permission:

```bash
chmod 750 hello.sh
```

Kiểm tra:

```bash
ls -l hello.sh
```

Chạy:

```bash
./hello.sh
```

---

# 6. Shebang là gì?

Dòng:

```bash
#!/bin/bash
```

được gọi thông thường là **shebang**.

Nó cho kernel biết interpreter nào nên được dùng khi file được execute trực tiếp:

```bash
./hello.sh
```

Mental model:

```text
./hello.sh
    │
    ▼
kernel sees #!/bin/bash
    │
    ▼
/bin/bash hello.sh
```

Nhưng nếu bạn chạy:

```bash
bash hello.sh
```

thì bạn đã trực tiếp yêu cầu Bash đọc file; execute permission và shebang không có vai trò giống trường hợp `./hello.sh`.

---

# 7. Đừng mặc định `/bin/bash` tồn tại ở mọi Unix

Trên hầu hết Linux distro phổ biến:

```bash
command -v bash
```

có thể trả:

```text
/usr/bin/bash
```

và `/bin/bash` có thể tồn tại trực tiếp hoặc thông qua filesystem layout/symlink.

Hãy kiểm tra:

```bash
command -v bash
ls -l /bin/bash 2>/dev/null
```

Bạn cũng sẽ thấy scripts dùng:

```bash
#!/usr/bin/env bash
```

Ưu điểm là tìm `bash` qua `PATH`.

Nhưng với cron, nơi `PATH` có thể khác interactive shell, đây cũng là một dependency.

Cho operations script được kiểm soát trên host cụ thể, việc biết chính xác interpreter path và dùng absolute interpreter thường dễ reasoning hơn.

---

# 8. Bash script không phải text file chứa commands một cách ngẫu nhiên

Một script production-minded thường có conceptual structure:

```text
interpreter
     ↓
constants/config
     ↓
functions
     ↓
validation/prechecks
     ↓
main logic
     ↓
summary
     ↓
exit status
```

Ví dụ:

```bash
#!/bin/bash

DISK_WARN=80

check_disk() {
    ...
}

main() {
    check_disk
}

main "$@"
```

Chúng ta sẽ xây dần đến cấu trúc này.

---

# 9. Comments

Trong Bash:

```bash
# this is a comment
```

Ví dụ:

```bash
#!/bin/bash

# Disk threshold expressed as percentage.
DISK_WARN=80
```

Comment tốt giải thích:

```text
why
assumption
operational reason
```

Comment kém:

```bash
# Set DISK_WARN to 80
DISK_WARN=80
```

Code đã nói điều đó rồi.

---

# 10. Variables

Assignment cơ bản:

```bash
name="server01"
```

Không có spaces quanh `=`.

Đúng:

```bash
name="server01"
```

Sai:

```bash
name = "server01"
```

Bash coi command trên như attempts để execute command `name`.

GNU Bash định nghĩa assignment dạng:

```text
name=value
```

và một variable đã được assignment thì được coi là set, kể cả khi value là empty string. :chatgpt-content-reference{index="3"}

---

# 11. Đọc variable

```bash
host="server01"

printf '%s\n' "$host"
```

Hoặc:

```bash
printf '%s\n' "${host}"
```

Braces đặc biệt hữu ích khi adjacent characters xuất hiện:

```bash
file="${host}_health.log"
```

Nếu viết:

```bash
file="$host_health.log"
```

Bash sẽ tìm variable có tên:

```text
host_health
```

chứ không phải `host`.

GNU Bash mô tả `${parameter}` là form rõ ràng của parameter expansion, đặc biệt cần thiết khi characters sau variable có thể bị hiểu là một phần tên variable. :chatgpt-content-reference{index="4"}

---

# 12. Quoting — một trong những kiến thức Bash quan trọng nhất

So sánh:

```bash
printf '%s\n' $file
```

và:

```bash
printf '%s\n' "$file"
```

Giả sử:

```bash
file="/tmp/My Evidence.txt"
```

Không quote:

```bash
rm $file
```

shell có thể split thành:

```text
/tmp/My
Evidence.txt
```

Thậm chí filename glob characters như:

```text
*
?
[
```

có thể bị pathname expansion.

Quote:

```bash
rm -- "$file"
```

giữ value như một argument.

GNU Bash xác nhận quoting dùng để loại bỏ ý nghĩa đặc biệt của characters và ngăn một số expansions/interpretations của shell. :chatgpt-content-reference{index="5"}

Quy tắc thực hành rất mạnh:

```text
Nếu expansion đại diện cho string/path/user input,
mặc định hãy nghĩ "$variable".
```

---

# 13. Single quotes và double quotes

Single quote:

```bash
printf '%s\n' '$HOME'
```

Output:

```text
$HOME
```

Double quote:

```bash
printf '%s\n' "$HOME"
```

Output có thể:

```text
/home/alice
```

Mental model:

```text
'...' → phần lớn coi literal

"..." → vẫn cho parameter expansion,
        command substitution,
        một số shell processing
```

Ví dụ:

```bash
name="Alice"

printf '%s\n' 'Hello $name'
```

Output:

```text
Hello $name
```

Trong khi:

```bash
printf '%s\n' "Hello $name"
```

Output:

```text
Hello Alice
```

---

# 14. Tại sao tôi ưu tiên `printf` hơn `echo` trong scripts?

Bạn hoàn toàn có thể dùng:

```bash
echo "healthy"
```

Nhưng `printf` cho formatting predictable hơn:

```bash
printf 'STATUS=%s\n' "$status"
```

GNU Bash builtin `printf` hỗ trợ format strings rõ ràng giống `printf` family truyền thống. :chatgpt-content-reference{index="6"}

Ví dụ:

```bash
printf '%-12s : %s\n' "Disk" "OK"
printf '%-12s : %s\n' "Service" "FAILED"
```

Output:

```text
Disk         : OK
Service      : FAILED
```

---

# 15. Command substitution

Bạn cần lấy output command đưa vào variable.

Ví dụ:

```bash
current_host=$(hostname)
```

Sau đó:

```bash
printf 'Host: %s\n' "$current_host"
```

Modern form:

```bash
$(command)
```

nên được ưu tiên hơn old backtick:

```bash
`command`
```

Bash thực thi command substitution và thay expression bằng stdout của command; trailing newlines bị loại bỏ. :chatgpt-content-reference{index="7"}

Ví dụ:

```bash
current_time=$(date '+%Y-%m-%d %H:%M:%S')
```

---

# 16. Command substitution không tự nói command có thành công hay không

Đây là chỗ fresher rất hay sai:

```bash
data=$(some-command)
```

Bạn thấy variable `data`.

Nhưng điều bạn phải quan tâm nữa là:

```text
some-command có exit status gì?
```

Ví dụ:

```bash
output=$(cat /file-that-does-not-exist)
rc=$?

printf 'rc=%d\n' "$rc"
```

`output` có thể empty, nhưng empty output không tự động có nghĩa command success hay failure.

Output và exit status là hai channels khác nhau:

```text
stdout → data

exit status → execution result
```

---

# 17. Arithmetic expansion

Bash hỗ trợ:

```bash
$(( expression ))
```

GNU manual định nghĩa đây là arithmetic expansion. :chatgpt-content-reference{index="8"}

Ví dụ:

```bash
count=3
count=$((count + 1))

printf '%d\n' "$count"
```

Output:

```text
4
```

Một dạng đặc biệt hữu ích trong Bash:

```bash
(( count++ ))
```

Nhưng cần cẩn thận về exit status của arithmetic command; khi mới học, đừng sử dụng nó một cách mù quáng trong scripts có `set -e`.

---

# 18. Positional parameters

Giả sử:

```bash
./health-check.sh ssh.service 80
```

Trong script:

```text
$0 → script name
$1 → ssh.service
$2 → 80
```

Ví dụ:

```bash
#!/bin/bash

service_name="$1"
disk_threshold="$2"

printf 'Service: %s\n' "$service_name"
printf 'Disk threshold: %s\n' "$disk_threshold"
```

Bash cung cấp positional parameters và special parameters như một phần core shell semantics. :chatgpt-content-reference{index="9"}

---

# 19. `"$@"` rất quan trọng

Nếu function hoặc script muốn forward tất cả arguments mà vẫn giữ argument boundaries:

```bash
some_function "$@"
```

Đừng dùng:

```bash
some_function $@
```

một cách tùy tiện.

Và đừng nhầm:

```text
"$@"
```

với:

```text
"$*"
```

Trong double quotes, `"$@"` giữ positional arguments thành các words riêng biệt; `"$*"` gộp chúng thành một word theo separator behavior. :chatgpt-content-reference{index="10"}

---

# 20. Parameter default values

Một pattern cực kỳ hữu ích:

```bash
service_name="${1:-ssh.service}"
```

Ý nghĩa:

```text
Nếu $1 có giá trị phù hợp
→ dùng $1

nếu unset/null
→ dùng ssh.service
```

Ví dụ:

```bash
disk_threshold="${2:-80}"
```

Bây giờ:

```bash
./health-check.sh
```

có defaults.

Còn:

```bash
./health-check.sh nginx.service 90
```

override defaults.

---

# 21. Required parameters

Nếu một parameter bắt buộc:

```bash
service_name="${1:?service name is required}"
```

Nếu `$1` không tồn tại hoặc empty, Bash báo lỗi.

Đây là **Supplementary**, nhưng rất hữu ích khi build admin tooling.

---

# 22. Environment variables

Bạn đã học environment variables ở Day 4.

Trong Bash:

```bash
APP_ENV="dev"
```

chỉ là shell variable của shell hiện tại.

```bash
export APP_ENV="dev"
```

đưa variable vào environment cho child processes.

Mental model:

```text
shell variable
      │
      │ export
      ▼
environment
      │
      ▼
child process
```

Điểm này cực kỳ quan trọng khi đến cron, bởi cron job **không chạy trong interactive shell session của bạn**.

---

# 23. Conditions: `if`

Basic syntax:

```bash
if command; then
    ...
else
    ...
fi
```

Điểm quan trọng:

**Bash `if` xét exit status của command**, không nhất thiết phải là Boolean variable.

Ví dụ:

```bash
if systemctl is-active --quiet ssh.service; then
    printf 'Service OK\n'
else
    printf 'Service NOT OK\n'
fi
```

Đây là shell programming idiom rất tốt.

Bạn không nhất thiết phải:

```bash
systemctl is-active --quiet ssh.service
rc=$?

if [ "$rc" -eq 0 ]; then
    ...
fi
```

nếu bạn chỉ cần branch ngay theo command result.

---

# 24. `test`, `[ ... ]` và `[[ ... ]]`

Bạn thường gặp:

```bash
[ "$x" -eq 5 ]
```

và:

```bash
[[ "$x" -eq 5 ]]
```

`[` là `test`-style conditional command.

`[[ ... ]]` là Bash conditional construct với semantics mạnh hơn và thường an toàn/dễ viết hơn trong Bash-only scripts. Bash documentation xác nhận conditional expressions được dùng bởi `[[`, `test` và `[`. :chatgpt-content-reference{index="11"}

Nếu script đã tuyên bố:

```bash
#!/bin/bash
```

thì dùng `[[ ... ]]` là hợp lý.

Nếu bạn cần POSIX `/bin/sh` portability, đừng dùng Bash-specific syntax.

---

# 25. String tests

Ví dụ:

```bash
if [[ -z "$service_name" ]]; then
    printf 'Service name is empty\n'
fi
```

`-z`:

```text
string length = zero
```

`-n`:

```text
string length != zero
```

So sánh:

```bash
if [[ "$status" == "active" ]]; then
    ...
fi
```

Bash documentation xác nhận các string/file/numeric conditional expressions này. :chatgpt-content-reference{index="12"}

---

# 26. Numeric comparisons

Với `[ ]` hoặc `[[ ]]`, bạn thường gặp:

```text
-eq
-ne
-lt
-le
-gt
-ge
```

Ví dụ:

```bash
if [[ "$disk_usage" -ge 80 ]]; then
    printf 'WARNING\n'
fi
```

Bash xác định các arithmetic binary operators này trong conditional expressions. :chatgpt-content-reference{index="13"}

---

# 27. File tests

Ví dụ:

```bash
[[ -e "$file" ]]
```

file tồn tại.

```bash
[[ -f "$file" ]]
```

regular file.

```bash
[[ -d "$dir" ]]
```

directory.

```bash
[[ -r "$file" ]]
```

readable.

```bash
[[ -w "$file" ]]
```

writable.

```bash
[[ -x "$file" ]]
```

executable.

Đây sẽ cực kỳ hữu ích trong backup script Module 3.

---

# 28. Logical operators

Trong `[[ ... ]]`:

```bash
if [[ "$usage" -ge 80 && "$usage" -lt 90 ]]; then
    ...
fi
```

Hoặc:

```bash
if [[ "$state" == "failed" || "$state" == "inactive" ]]; then
    ...
fi
```

Bash hỗ trợ `&&`, `||` và short-circuit semantics trong conditional expressions. :chatgpt-content-reference{index="14"}

---

# 29. `&&` và `||` giữa commands

Không chỉ trong `[[ ]]`.

Ví dụ:

```bash
mkdir -p "$dir" && printf 'Directory ready\n'
```

Command sau `&&` chỉ chạy nếu command trước success.

```bash
systemctl is-active --quiet ssh.service ||
    printf 'Service unhealthy\n'
```

Command sau `||` chạy khi command trước non-zero.

Nhưng đừng lạm dụng:

```bash
command1 && command2 || command3
```

như một replacement chung cho `if`, vì semantics có thể khó đọc và có edge cases khi `command2` fail.

Trong operational scripts:

```bash
if ...; then
    ...
else
    ...
fi
```

thường dễ maintain hơn.

---

# 30. Loops

Ví dụ `for`:

```bash
for service in ssh.service cron.service; do
    printf 'Checking %s\n' "$service"
done
```

Health check nhiều services:

```bash
for service in ssh.service cron.service; do
    if systemctl is-active --quiet "$service"; then
        printf '%s OK\n' "$service"
    else
        printf '%s FAILED\n' "$service"
    fi
done
```

Loop rất hữu ích nhưng không nhất thiết cần dùng nếu script chỉ check một service.

Đừng thêm abstraction khi chưa có nhu cầu.

---

# 31. Functions

Function giúp đóng gói một unit logic:

```bash
check_service() {
    local service_name="$1"

    if systemctl is-active --quiet "$service_name"; then
        printf 'OK: %s\n' "$service_name"
        return 0
    else
        printf 'FAIL: %s\n' "$service_name"
        return 1
    fi
}
```

Bash functions được thực thi trong current shell context và có thể được gọi giống simple commands. :chatgpt-content-reference{index="15"}

Function này có một property rất quan trọng:

```text
nó vừa in human-readable result
vừa return machine-readable exit status
```

---

# 32. `local`

Trong function:

```bash
local service_name="$1"
```

giới hạn variable ở function scope theo Bash behavior.

Điều này giảm accidental variable collisions:

```bash
status="overall"

check_service() {
    local status="active"
}
```

Function xong, outer `status` không bị overwrite.

Đây là practice nên có trong Bash admin scripts.

---

# 33. `return` và `exit` khác nhau

Trong function:

```bash
return 1
```

kết thúc **function** và trả status.

Trong script:

```bash
exit 1
```

kết thúc **shell/script**.

Ví dụ:

```bash
check_disk() {
    if ...; then
        return 0
    fi

    return 1
}

check_disk
exit $?
```

Đừng vô tình dùng:

```bash
exit 1
```

bên trong helper function nếu bạn vẫn cần script tiếp tục thu thập các health checks khác.

---

# 34. Exit status — trái tim của Module 2

GNU Bash định nghĩa exit status nằm trong phạm vi 0–255. Với shell:

```text
0     → success

non-0 → failure / alternate unsuccessful status
```

Bash cũng dùng một số status đặc biệt: command not found thường là `127`, command found nhưng không executable là `126`, và process chết bởi signal `N` được biểu diễn theo dạng `128+N`. :chatgpt-content-reference{index="16"}

Đây không có nghĩa:

```text
mọi non-zero đều giống nhau.
```

Command-specific documentation vẫn là source of truth cho ý nghĩa từng status.

---

# 35. `$?`

Exit status command gần nhất:

```bash
command
printf '%d\n' "$?"
```

Ví dụ:

```bash
true
printf 'rc=%d\n' "$?"
```

Output:

```text
rc=0
```

```bash
false
printf 'rc=%d\n' "$?"
```

Output:

```text
rc=1
```

`$?` là special parameter chứa exit status của command/pipeline gần nhất. :chatgpt-content-reference{index="17"}

---

# 36. `$?` rất dễ bị overwrite

Sai:

```bash
some-command
printf 'Command completed\n'

if [[ $? -ne 0 ]]; then
    ...
fi
```

Lúc này `$?` là exit status của:

```bash
printf
```

không còn của `some-command`.

Đúng:

```bash
some-command
rc=$?

printf 'Command completed with rc=%d\n' "$rc"

if [[ "$rc" -ne 0 ]]; then
    ...
fi
```

Hoặc tốt hơn khi chỉ cần branch:

```bash
if some-command; then
    ...
else
    ...
fi
```

---

# 37. Không tự định nghĩa mọi exit code non-zero là “error hệ thống”

Nhớ ví dụ Module 1:

```bash
grep 'CRITICAL' logfile
```

GNU grep convention:

```text
0 → có selected line
1 → không có selected line
2 → error
```

Vậy health script tìm `"CRITICAL"` có thể coi:

```text
grep rc=1
```

là **healthy result**, không phải script failure.

Điều này minh họa nguyên tắc:

> Exit status phải được hiểu theo semantics của command, không chỉ theo “0 hay không 0”.

---

# 38. Tự thiết kế exit codes cho script

Một health-check script đơn giản có thể quy ước:

```text
0 → all checks healthy
1 → health warning/failure found
2 → script/configuration/internal execution error
```

Đây là **contract do chúng ta thiết kế**, không phải universal Linux standard.

Ví dụ:

```bash
if service_bad; then
    exit 1
fi

exit 0
```

Nếu hệ thống downstream cần phân biệt:

```text
warning
critical
internal error
```

bạn có thể thiết kế thêm codes, nhưng phải document.

Đừng tạo 30 exit codes mà không ai nhớ semantics.

---

# 39. Exit code aggregation

Health check thường thực hiện nhiều checks:

```text
disk
service
port
logs
```

Nếu disk fail nhưng service pass, final exit vẫn phải phản ánh unhealthy.

Một simple pattern:

```bash
overall_rc=0

if ! check_disk; then
    overall_rc=1
fi

if ! check_service; then
    overall_rc=1
fi

exit "$overall_rc"
```

Điểm quan trọng:

Script **không exit ngay sau first failure**, nhờ đó vẫn collect được các evidence khác.

Đây thường là behavior tốt cho health-check/report scripts.

---

# 40. Pipeline exit status — bẫy cực kỳ quan trọng

Giả sử:

```bash
some-command | grep ERROR
```

Mặc định, Bash lấy exit status của **command cuối pipeline**, trừ khi `pipefail` được enable. :chatgpt-content-reference{index="18"}

Ví dụ conceptual:

```text
some-command → FAILED
grep         → SUCCESS
```

Pipeline có thể trả:

```text
0
```

nếu `grep` là command cuối và success.

Điều này có thể che mất upstream failure.

---

# 41. Ví dụ pipeline failure bị che

```bash
cat /does/not/exist | grep ERROR
```

Bạn có thể chỉ nhìn status của `grep`.

Vấn đề conceptual:

```text
producer failed
      │
      ▼
consumer still determines pipeline status by default
```

Trong automation, đây là rủi ro.

---

# 42. Supplementary — `set -o pipefail`

```bash
set -o pipefail
```

Bash khi đó trả pipeline status bằng rightmost non-zero command status, hoặc zero nếu toàn bộ commands success. :chatgpt-content-reference{index="19"}

Ví dụ:

```bash
set -o pipefail

some-command |
grep ERROR
```

Nếu producer fail, script có cơ hội nhận biết.

Đây là một trong những options tôi thường cân nhắc cho operations scripts.

Nhưng:

```text
pipefail không tự làm script hoàn hảo.
```

Bạn vẫn cần hiểu mỗi command status.

---

# 43. Supplementary — `set -u`

```bash
set -u
```

hoặc:

```bash
set -o nounset
```

khi expansion một unset variable, non-interactive Bash thường báo lỗi và exit. GNU Bash mô tả chính xác behavior này. :chatgpt-content-reference{index="20"}

Ví dụ typo:

```bash
disk_threshold=80

printf '%s\n' "$disk_thresold"
```

Không `set -u`:

```text
có thể trở thành empty
```

Có `set -u`:

```text
unbound variable
```

Trong admin automation, fail loudly thường tốt hơn silently doing wrong thing.

---

# 44. Supplementary — `set -e` không đơn giản như “exit khi lỗi”

Bạn sẽ thường thấy:

```bash
set -e
```

Internet hay giải thích:

> “Exit script ngay khi một command fail.”

Đó là simplification nguy hiểm.

GNU Bash có nhiều exceptions: failures dùng trong `if`, `while`, `until`, nhiều `&&`/`||` constructs, parts của pipeline và các contexts khác không có behavior naïve như mô tả trên. :chatgpt-content-reference{index="21"}

Vì vậy tôi **không muốn bạn học thuộc**:

```bash
set -euo pipefail
```

như một thần chú.

Bạn phải hiểu từng option.

---

# 45. Có nên dùng `set -euo pipefail` không?

Trong một số scripts:

```bash
set -euo pipefail
```

rất hữu ích.

Trong một health-check script khác, bạn cố ý muốn:

```text
check disk fails
nhưng vẫn check service
nhưng vẫn check port
nhưng vẫn write report
```

thì fail-fast bằng `-e` có thể chống lại mục tiêu script.

Do đó trong bài health-check chính của Module 2, chúng ta sẽ ưu tiên:

```text
explicit error handling
```

thay vì phụ thuộc vào `set -e`.

Chúng ta có thể dùng:

```bash
set -u
set -o pipefail
```

sau khi hiểu consequences.

---

# 46. `bash -n` — syntax validation

Trước khi execute:

```bash
bash -n health-check.sh
```

Nó parse syntax mà không chạy normal commands.

Nếu không output và rc=0:

```text
syntax parse passed
```

Nhưng nhớ:

```text
syntax correct ≠ logic correct
```

Ví dụ:

```bash
rm -rf "$something"
```

có thể syntax perfectly valid nhưng operationally catastrophic.

---

# 47. `bash -x` — execution tracing

Debug:

```bash
bash -x health-check.sh
```

Bạn sẽ thấy expanded commands khi Bash execute.

Ví dụ:

```text
+ threshold=80
+ check_disk
+ ...
```

Rất hữu ích.

**Security warning:** `-x` có thể expose variables, command arguments và secrets.

Không bật tracing vô tội vạ trên scripts xử lý:

```text
password
token
database credential
API keys
```

Không attach `bash -x` output vào ticket trước khi review/redact.

---

# 48. Functions + status design

Một reusable check nên có contract rõ.

Ví dụ:

```bash
check_service() {
    local service_name="$1"

    if systemctl is-active --quiet "$service_name"; then
        printf 'OK      service=%s\n' "$service_name"
        return 0
    fi

    printf 'CRITICAL service=%s state=not-active\n' "$service_name"
    return 1
}
```

Function này không restart service.

Nó chỉ:

```text
observe
classify
report
```

Rất phù hợp L1 health check.

---

# 49. Disk check

Từ Day 5:

```bash
df -P /
```

Ta parse:

```bash
check_disk() {
    local mount_point="$1"
    local threshold="$2"
    local usage

    usage=$(
        df -P "$mount_point" |
        awk 'NR==2 {
            gsub(/%/, "", $5)
            print $5
        }'
    )

    if [[ -z "$usage" ]]; then
        printf 'UNKNOWN disk mount=%s reason=no-usage-value\n' "$mount_point"
        return 2
    fi

    if [[ "$usage" -ge "$threshold" ]]; then
        printf 'CRITICAL disk mount=%s usage=%s%% threshold=%s%%\n' \
            "$mount_point" "$usage" "$threshold"
        return 1
    fi

    printf 'OK      disk mount=%s usage=%s%% threshold=%s%%\n' \
        "$mount_point" "$usage" "$threshold"

    return 0
}
```

Có một vấn đề vẫn còn:

Nếu `df` fail thì sao?

---

# 50. Không được chỉ validate output; phải validate command execution

Phiên bản tốt hơn:

```bash
check_disk() {
    local mount_point="$1"
    local threshold="$2"
    local df_output
    local usage

    if ! df_output=$(df -P "$mount_point" 2>&1); then
        printf 'UNKNOWN disk mount=%s reason=df-failed detail=%s\n' \
            "$mount_point" "$df_output"
        return 2
    fi

    usage=$(
        printf '%s\n' "$df_output" |
        awk 'NR==2 {
            gsub(/%/, "", $5)
            print $5
        }'
    )

    if [[ ! "$usage" =~ ^[0-9]+$ ]]; then
        printf 'UNKNOWN disk mount=%s reason=invalid-usage value=%q\n' \
            "$mount_point" "$usage"
        return 2
    fi

    if [[ "$usage" -ge "$threshold" ]]; then
        printf 'CRITICAL disk mount=%s usage=%s%% threshold=%s%%\n' \
            "$mount_point" "$usage" "$threshold"
        return 1
    fi

    printf 'OK      disk mount=%s usage=%s%% threshold=%s%%\n' \
        "$mount_point" "$usage" "$threshold"

    return 0
}
```

Bây giờ script phân biệt:

```text
disk actually high
```

với:

```text
could not inspect disk
```

Đó là sự khác nhau giữa:

```text
health failure
```

và:

```text
monitor/check failure
```

Một operations engineer phải phân biệt hai thứ đó.

---

# 51. Port check bằng `ss`

Từ Day 5:

```bash
ss -lnt
```

Ví dụ kiểm tra port 8080 đang LISTEN:

```bash
check_listen_port() {
    local port="$1"

    if ss -lnt |
        awk '{print $4}' |
        grep -Eq ":${port}$"; then

        printf 'OK      port=%s state=listening\n' "$port"
        return 0
    fi

    printf 'CRITICAL port=%s state=not-listening\n' "$port"
    return 1
}
```

Nhưng pipeline semantics có vấn đề tiềm tàng.

Nếu `ss` fail nhưng `grep` chỉ thấy no match:

```text
script có thể report "port not listening"
```

trong khi thật ra:

```text
health checker failed to run ss.
```

Vì vậy robust version nên capture/validate producer riêng.

---

# 52. Port check robust hơn

```bash
check_listen_port() {
    local port="$1"
    local ss_output

    if ! ss_output=$(ss -lnt 2>&1); then
        printf 'UNKNOWN port=%s reason=ss-failed detail=%s\n' \
            "$port" "$ss_output"
        return 2
    fi

    if printf '%s\n' "$ss_output" |
        awk '{print $4}' |
        grep -Eq ":${port}$"; then

        printf 'OK      port=%s state=listening\n' "$port"
        return 0
    fi

    printf 'CRITICAL port=%s state=not-listening\n' "$port"
    return 1
}
```

Đây là mindset quan trọng:

```text
Before interpreting the result,
make sure your measuring instrument worked.
```

---

# 53. Health-check status model

Ta có thể dùng ba logical states:

| Logical state | Ý nghĩa                                           | Suggested script handling |
| ------------- | ------------------------------------------------- | ------------------------: |
| `OK`          | Check thực hiện được và condition tốt             |                       `0` |
| `CRITICAL`    | Check thực hiện được nhưng health condition xấu   |                       `1` |
| `UNKNOWN`     | Không thể xác định health vì check/tool/input lỗi |                       `2` |

Đây là design cho lab của chúng ta, không phải Linux universal standard.

Điểm mạnh là bạn phân biệt:

```text
service down
```

và:

```text
không đủ quyền để kiểm tra service
```

---

# 54. Main function

Một structure sạch:

```bash
main() {
    local overall_rc=0

    if ! check_service "$SERVICE_NAME"; then
        overall_rc=1
    fi

    if ! check_disk "$MOUNT_POINT" "$DISK_THRESHOLD"; then
        overall_rc=1
    fi

    if ! check_listen_port "$PORT"; then
        overall_rc=1
    fi

    return "$overall_rc"
}

main "$@"
exit $?
```

Nhưng vẫn có một issue:

`UNKNOWN=2` đang bị biến thành overall 1.

Ta có thể thiết kế aggregation tốt hơn.

---

# 55. Aggregate severity

Contract:

```text
0 = OK
1 = CRITICAL
2 = UNKNOWN/check error
```

Một approach:

```bash
overall_rc=0

run_check() {
    "$@"
    local rc=$?

    if [[ "$rc" -gt "$overall_rc" ]]; then
        overall_rc="$rc"
    fi
}
```

Sau đó:

```bash
run_check check_service "$SERVICE_NAME"
run_check check_disk "$MOUNT_POINT" "$DISK_THRESHOLD"
run_check check_listen_port "$PORT"
```

Tuy nhiên numerical ordering chỉ hợp lý vì **chúng ta chủ động thiết kế codes như vậy**.

Không được giả định mọi external command có higher code = more severe.

---

# 56. Health-check script hoàn chỉnh — phiên bản học tập

Trước khi sử dụng, xác định service thật:

```bash
systemctl list-units --type=service --state=running
```

Và port thật nếu cần:

```bash
ss -lnt
```

Script:

```bash
#!/bin/bash

set -u
set -o pipefail

SERVICE_NAME="${SERVICE_NAME:-ssh.service}"
MOUNT_POINT="${MOUNT_POINT:-/}"
DISK_THRESHOLD="${DISK_THRESHOLD:-80}"
PORT="${PORT:-22}"

overall_rc=0

timestamp() {
    date '+%Y-%m-%dT%H:%M:%S%z'
}

record_rc() {
    local rc="$1"

    if [[ "$rc" -gt "$overall_rc" ]]; then
        overall_rc="$rc"
    fi
}

check_service() {
    local service_name="$1"

    if systemctl is-active --quiet "$service_name"; then
        printf '%s %-8s check=service service=%s state=active\n' \
            "$(timestamp)" "OK" "$service_name"
        return 0
    fi

    printf '%s %-8s check=service service=%s state=not-active\n' \
        "$(timestamp)" "CRITICAL" "$service_name"
    return 1
}

check_disk() {
    local mount_point="$1"
    local threshold="$2"
    local df_output
    local usage

    if ! df_output=$(df -P "$mount_point" 2>&1); then
        printf '%s %-8s check=disk mount=%s reason=df-failed detail=%q\n' \
            "$(timestamp)" "UNKNOWN" "$mount_point" "$df_output"
        return 2
    fi

    usage=$(
        printf '%s\n' "$df_output" |
        awk 'NR==2 {
            gsub(/%/, "", $5)
            print $5
        }'
    )

    if [[ ! "$usage" =~ ^[0-9]+$ ]]; then
        printf '%s %-8s check=disk mount=%s reason=invalid-usage value=%q\n' \
            "$(timestamp)" "UNKNOWN" "$mount_point" "$usage"
        return 2
    fi

    if [[ "$usage" -ge "$threshold" ]]; then
        printf '%s %-8s check=disk mount=%s usage=%s%% threshold=%s%%\n' \
            "$(timestamp)" "CRITICAL" \
            "$mount_point" "$usage" "$threshold"
        return 1
    fi

    printf '%s %-8s check=disk mount=%s usage=%s%% threshold=%s%%\n' \
        "$(timestamp)" "OK" \
        "$mount_point" "$usage" "$threshold"

    return 0
}

check_port() {
    local port="$1"
    local ss_output

    if ! ss_output=$(ss -lnt 2>&1); then
        printf '%s %-8s check=port port=%s reason=ss-failed detail=%q\n' \
            "$(timestamp)" "UNKNOWN" "$port" "$ss_output"
        return 2
    fi

    if printf '%s\n' "$ss_output" |
        awk '{print $4}' |
        grep -Eq ":${port}$"; then

        printf '%s %-8s check=port port=%s state=listening\n' \
            "$(timestamp)" "OK" "$port"
        return 0
    fi

    printf '%s %-8s check=port port=%s state=not-listening\n' \
        "$(timestamp)" "CRITICAL" "$port"
    return 1
}

main() {
    local rc

    printf '%s INFO     event=health-check-start host=%s\n' \
        "$(timestamp)" "$(hostname)"

    check_service "$SERVICE_NAME"
    rc=$?
    record_rc "$rc"

    check_disk "$MOUNT_POINT" "$DISK_THRESHOLD"
    rc=$?
    record_rc "$rc"

    check_port "$PORT"
    rc=$?
    record_rc "$rc"

    printf '%s INFO     event=health-check-end exit_code=%d\n' \
        "$(timestamp)" "$overall_rc"

    return "$overall_rc"
}

main "$@"
exit $?
```

---

# 57. Đừng copy script rồi chạy ngay

Trước hết inspect values:

```bash
printf 'SERVICE=%s\n' "ssh.service"
printf 'PORT=%s\n' "22"
systemctl status ssh.service --no-pager
ss -lnt
```

Trên RHEL-family SSH unit thường có thể là:

```text
sshd.service
```

Trên Debian/Ubuntu thường gặp:

```text
ssh.service
```

Không hard-code từ bài học mà không kiểm tra machine của bạn.

---

# 58. Syntax validation

Lưu thành:

```text
$HOME/day6-module2/health-check.sh
```

Sau đó:

```bash
bash -n "$HOME/day6-module2/health-check.sh"
```

Check rc:

```bash
printf 'syntax_rc=%d\n' "$?"
```

Expected invariant:

```text
syntax_rc=0
```

---

# 59. Permission

```bash
chmod 750 "$HOME/day6-module2/health-check.sh"
```

Kiểm tra:

```bash
ls -l "$HOME/day6-module2/health-check.sh"
```

Ví dụ:

```text
-rwxr-x--- ...
```

Không cần:

```bash
chmod 777
```

`777` thường là dấu hiệu fresher đang giải permission issue bằng cách mở quyền quá rộng.

---

# 60. Chạy foreground

```bash
"$HOME/day6-module2/health-check.sh"
```

Sau đó **ngay lập tức**:

```bash
rc=$?
printf 'script_rc=%d\n' "$rc"
```

Expected nếu tất cả healthy:

```text
script_rc=0
```

Nếu một condition unhealthy:

```text
script_rc=1
```

Nếu inspection itself fail:

```text
script_rc=2
```

theo contract lab của chúng ta.

---

# 61. Override configuration mà không sửa script

Ví dụ máy dùng:

```text
sshd.service
```

chạy:

```bash
SERVICE_NAME=sshd.service \
PORT=22 \
DISK_THRESHOLD=85 \
"$HOME/day6-module2/health-check.sh"
```

Điều này giới thiệu **externalized configuration mindset**, sẽ rất quan trọng ở Java Day 13.

Code một nơi; environment-specific values được truyền từ ngoài.

---

# 62. Validation phải có positive và negative case

Nếu chỉ chạy lúc mọi thứ OK:

```text
bạn mới chứng minh happy path.
```

Bạn chưa chứng minh:

```text
script có phát hiện failure thật không?
```

Thay vì stop service, ta có thể inject failure **an toàn** bằng configuration.

Ví dụ:

```bash
SERVICE_NAME=definitely-not-a-real.service \
"$HOME/day6-module2/health-check.sh"
```

Expected:

```text
CRITICAL check=service ...
```

và script rc phải non-zero.

Không cần phá service thật.

---

# 63. Inject disk-check failure an toàn

Không cần fill disk.

Dùng mount path không tồn tại:

```bash
MOUNT_POINT=/definitely-not-a-real-mount \
"$HOME/day6-module2/health-check.sh"
```

Expected:

```text
UNKNOWN check=disk
reason=df-failed
```

Đây là **checker failure/input failure**, không phải disk capacity warning.

Bạn vừa test một failure path mà không phá filesystem.

---

# 64. Inject port failure an toàn

Chọn port không listen, ví dụ sau khi verify:

```bash
ss -lnt
```

Rồi:

```bash
PORT=65432 \
"$HOME/day6-module2/health-check.sh"
```

Expected:

```text
CRITICAL check=port port=65432 state=not-listening
```

Nhưng hãy verify trước port đó thực sự không listen.

---

# 65. Failure injection phải có expected behavior

Một failure injection chuyên nghiệp không phải:

```text
"Tôi đổi đại cái gì đó để script lỗi."
```

Mà phải có:

| Injection     | Expected check result | Expected exit |
| ------------- | --------------------- | ------------: |
| Fake service  | `CRITICAL service`    |      non-zero |
| Invalid mount | `UNKNOWN disk`        |      non-zero |
| Closed port   | `CRITICAL port`       |      non-zero |

Nếu actual result khác expectation:

```text
đó chính là troubleshooting exercise.
```

---

# 66. Evidence output

Redirect:

```bash
"$HOME/day6-module2/health-check.sh" \
    > "$HOME/day6-module2/health-check.log" \
    2>&1

rc=$?

printf 'rc=%d\n' "$rc"
```

Sau đó:

```bash
cat "$HOME/day6-module2/health-check.log"
```

Tốt hơn khi append nhiều runs:

```bash
"$HOME/day6-module2/health-check.sh" \
    >> "$HOME/day6-module2/health-check.log" \
    2>&1
```

Nhưng lúc này cần timestamp trong mỗi record, vì file chứa nhiều executions.

Script của chúng ta đã làm điều đó.

---

# 67. Tại sao log line nên machine-friendly?

So sánh:

```text
Everything seems fine with disk.
```

với:

```text
2026-10-01T20:31:00+0700 OK check=disk mount=/ usage=37% threshold=80%
```

Dòng thứ hai dễ:

```text
grep
awk
parse
correlate
```

Đây là nền móng cho monitoring sau Day 17.

---

# 68. Cron mental model

Bây giờ script chạy thủ công được.

Ta chuyển sang scheduling:

```text
crontab
    │
    ▼
cron daemon
    │
    ▼
time match
    │
    ▼
launch command
    │
    ▼
non-interactive environment
    │
    ▼
health-check.sh
    │
    ▼
exit code + output
```

Cronie documentation mô tả crontab như các instructions “run this command at this time/date”; mỗi user's crontab chạy commands dưới account sở hữu crontab. :chatgpt-content-reference{index="22"}

---

# 69. Cron daemon names khác nhau giữa distro

Bạn có thể gặp:

Debian/Ubuntu:

```bash
systemctl status cron.service
```

RHEL/Rocky/AlmaLinux/Cronie:

```bash
systemctl status crond.service
```

Kiểm tra thay vì đoán:

```bash
systemctl list-unit-files |
grep -E '^(cron|crond)\.service'
```

Cronie documentation cũng ghi nhận systemd unit `crond.service` trên hệ thống sử dụng Cronie. :chatgpt-content-reference{index="23"}

---

# 70. User crontab

Xem:

```bash
crontab -l
```

Edit:

```bash
crontab -e
```

Không nên trực tiếp sửa files trong spool directory của cron.

Dùng:

```bash
crontab
```

command interface.

---

# 71. Cron expression — năm field

User crontab:

```text
minute hour day-of-month month day-of-week command
```

Cronie cho allowed ranges:

| Field        |                                   Common range |
| ------------ | ---------------------------------------------: | -------------------------------------- |
| minute       |                                         `0-59` |
| hour         |                                         `0-23` |
| day of month |                                         `1-31` |
| month        |                                         `1-12` |
| day of week  | `0-7`, với `0/7` thường là Sunday trong Cronie | :chatgpt-content-reference{index="24"} |

---

# 72. Đọc cron từ trái sang phải

Ví dụ:

```cron
30 2 * * * /path/job.sh
```

Đọc:

```text
minute = 30
hour = 2
day of month = any
month = any
day of week = any

→ 02:30 every day
```

---

# 73. `*`

```text
*
```

có nghĩa:

```text
mọi giá trị hợp lệ trong field
```

Ví dụ:

```cron
* * * * *
```

match mỗi minute.

Trong lab có thể dùng để nhanh chóng test scheduling.

Production không nên để health job chạy mỗi phút nếu không có lý do, đặc biệt nếu check tốn tài nguyên.

---

# 74. Lists

Ví dụ:

```cron
0 8,12,18 * * *
```

chạy:

```text
08:00
12:00
18:00
```

Cronie hỗ trợ comma-separated lists. :chatgpt-content-reference{index="25"}

---

# 75. Ranges

```cron
0 9-17 * * *
```

chạy mỗi hour từ 09:00 tới 17:00 tại minute 0.

Cronie ranges là inclusive. :chatgpt-content-reference{index="26"}

---

# 76. Steps

```cron
*/5 * * * *
```

nghĩa là every 5 minutes trong minute field.

Nhưng rất quan trọng:

```cron
*/35 * * * *
```

không có nghĩa “chính xác mỗi 35 phút liên tục”.

Cronie giải thích step hoạt động **bên trong field**, nên `*/35` ở minute field sẽ match các points trong 0–59 tương ứng, không tạo một interval clock liên tục xuyên hours. :chatgpt-content-reference{index="27"}

Đây là câu exam/troubleshooting rất hay.

---

# 77. Ví dụ cron expressions

```cron
*/5 * * * *
```

Mỗi 5 phút.

```cron
0 * * * *
```

Đầu mỗi giờ.

```cron
0 2 * * *
```

02:00 mỗi ngày.

```cron
30 1 * * 1
```

01:30 vào Monday theo numbering/name semantics của implementation.

```cron
0 3 1 * *
```

03:00 ngày 1 mỗi tháng.

---

# 78. Day-of-month và day-of-week có nuance

Cronie có behavior đáng chú ý:

Nếu **cả day-of-month và day-of-week đều bị restricted**, Cronie chạy khi **một trong hai** match, thay vì trực giác “cả hai phải match”. :chatgpt-content-reference{index="28"}

Ví dụ Cronie:

```cron
30 4 1,15 * 5
```

có thể nghĩa:

```text
04:30 ngày 1
hoặc ngày 15
hoặc Friday
```

không phải:

```text
chỉ khi ngày 1/15 đồng thời là Friday
```

Đây là lý do không nên viết cron calendar expression phức tạp mà không kiểm chứng semantics của implementation.

---

# 79. User crontab và `/etc/crontab` khác format

User:

```bash
crontab -e
```

entry:

```cron
*/5 * * * * /home/alice/job.sh
```

Không có username field.

Nhưng system crontab `/etc/crontab` và files trong `/etc/cron.d/` thường có username field:

```cron
*/5 * * * * alice /home/alice/job.sh
```

Cronie documentation ghi rõ system cron format có additional username field. :chatgpt-content-reference{index="29"}

Đây là lỗi fresher rất thường mắc.

---

# 80. Đừng thêm username vào `crontab -e`

Sai trong user crontab:

```cron
*/5 * * * * alice /home/alice/job.sh
```

Cron sẽ cố execute:

```text
alice
```

như command hoặc một phần command.

Đúng:

```cron
*/5 * * * * /home/alice/job.sh
```

---

# 81. Cron shell environment

Cronie mặc định set một số environment variables, trong đó `SHELL` mặc định là `/bin/sh`; `HOME` và `LOGNAME` lấy từ account owner, mặc dù details có thể khác giữa implementations. :chatgpt-content-reference{index="30"}

Điều này cực quan trọng.

Crontab entry:

```cron
* * * * * source ~/.bashrc; some-command
```

không nên được giả định sẽ chạy giống Bash interactive shell.

---

# 82. Cron không phải terminal login session của bạn

Khi SSH:

```text
login
  ↓
shell startup files
  ↓
interactive environment
  ↓
PATH
  ↓
aliases
  ↓
functions
  ↓
current working directory
```

Cron:

```text
cron daemon
  ↓
minimal/specific environment
  ↓
scheduled command
```

Đây là nguồn gốc của câu kinh điển:

> “Script chạy tay được nhưng cron không chạy.”

---

# 83. `PATH` issue

Bạn chạy tay:

```bash
my-custom-command
```

success vì:

```bash
echo "$PATH"
```

có:

```text
/home/alice/bin
```

Cron có thể không có path đó.

Do đó trong admin automation:

```text
absolute paths
hoặc
explicit controlled PATH
```

thường an toàn hơn.

Ví dụ crontab:

```cron
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Nhưng trước khi copy PATH này:

```bash
command -v systemctl
command -v df
command -v awk
command -v grep
command -v ss
command -v date
command -v hostname
```

Check machine thật.

---

# 84. Absolute script path

Không:

```cron
*/5 * * * * ./health-check.sh
```

Bạn không nên giả định cron working directory là directory script.

Dùng:

```cron
*/5 * * * * /home/alice/day6-module2/health-check.sh
```

Hoặc explicit interpreter:

```cron
*/5 * * * * /bin/bash /home/alice/day6-module2/health-check.sh
```

---

# 85. Relative paths bên trong script cũng nguy hiểm

Ví dụ script:

```bash
LOG_FILE="./health.log"
```

Chạy trong:

```text
/home/alice/day6-module2
```

thì log ở đúng chỗ.

Cron chạy với working directory khác, `./health.log` có thể nằm chỗ khác hoặc fail.

Tốt hơn:

```bash
BASE_DIR="$HOME/day6-module2"
LOG_FILE="$BASE_DIR/health.log"
```

Hoặc path configured explicitly.

---

# 86. Cron output

Theo traditional/POSIX behavior, stdout/stderr không redirect có thể được gửi bằng mail qua implementation-defined/local mail setup. :chatgpt-content-reference{index="31"}

Nhưng nhiều modern minimal servers không có mail transport bạn thực sự sử dụng.

Vì vậy bạn nên chủ động quyết định output đi đâu.

Lab:

```cron
*/5 * * * * /bin/bash /home/alice/day6-module2/health-check.sh >> /home/alice/day6-module2/cron-health.log 2>&1
```

---

# 87. Nhưng cron line dài không nên chứa business logic

Xấu:

```cron
*/5 * * * * df -P / | awk '...' | grep ... && systemctl ... || ...
```

Tốt:

```cron
*/5 * * * * /bin/bash /opt/ops/health-check.sh >> /var/log/... 2>&1
```

Mental model:

```text
cron decides WHEN

script decides WHAT/HOW
```

Tách responsibilities làm troubleshooting dễ hơn.

---

# 88. `%` trong crontab là một trap

Trong Cronie, unescaped `%` trong command portion có special semantics: nó có thể được biến thành newline và phần sau first `%` được đưa thành stdin. :chatgpt-content-reference{index="32"}

Vì vậy command như:

```cron
* * * * * date '+%Y-%m-%d'
```

có thể không behave như bạn tưởng trên implementations mang semantics đó.

Đây là thêm một lý do để:

```text
đặt logic vào script
```

thay vì cram shell logic vào crontab.

---

# 89. Timezone

Cron scheduling phụ thuộc system/cron timezone và implementation/configuration.

Cronie hỗ trợ:

```text
CRON_TZ
```

trong crontab. :chatgpt-content-reference{index="33"}

Nhưng đừng sử dụng trước khi hiểu environment.

Kiểm tra:

```bash
timedatectl
date
```

Một incident ticket ghi UTC nhưng cron host chạy UTC+7 có thể tạo nhầm timeline.

---

# 90. DST và implementation differences

Với servers ở regions có Daylight Saving Time, scheduled jobs quanh clock-change boundaries cần đặc biệt cẩn thận. Cron implementations có thể có handling khác nhau hoặc extensions khác nhau.

Việt Nam không dùng DST hiện nay, nhưng production servers có thể chạy UTC hoặc timezone khác.

Operational principle:

```text
schedule semantics
+ server timezone
+ application timezone
```

phải rõ ràng.

---

# 91. Cron service validation

Trước khi blame script:

Debian-like:

```bash
systemctl status cron.service --no-pager
```

RHEL/Cronie:

```bash
systemctl status crond.service --no-pager
```

Nếu daemon itself inactive:

```text
crontab đúng vẫn không chạy.
```

---

# 92. Crontab validation

List:

```bash
crontab -l
```

Capture evidence:

```bash
crontab -l > "$HOME/day6-module2/crontab-evidence.txt"
```

Kiểm tra exact entry thay vì nói:

> “Tôi nhớ là đã add cron rồi.”

Operations cần evidence.

Cronie hiện có `crontab -T` để syntax-test crontab trước install, nhưng đây là implementation-specific extension, không được giả định có trên mọi distro. :chatgpt-content-reference{index="34"}

---

# 93. Guided Lab — Schedule health check

Giả sử:

```text
/home/alice/day6-module2/health-check.sh
```

Hãy dùng path thực của bạn:

```bash
realpath "$HOME/day6-module2/health-check.sh"
```

Đảm bảo executable:

```bash
test -x "$HOME/day6-module2/health-check.sh"
printf 'rc=%d\n' "$?"
```

---

# 94. Manual execution trước cron

Bắt buộc:

```bash
/bin/bash "$HOME/day6-module2/health-check.sh" \
    >> "$HOME/day6-module2/manual-test.log" \
    2>&1

rc=$?
printf 'manual_rc=%d\n' "$rc"
```

Đọc:

```bash
cat "$HOME/day6-module2/manual-test.log"
```

Nếu manual execution bằng command **giống cron sẽ dùng** còn fail, đừng add cron.

---

# 95. Test cron mỗi phút trong lab

Tạm thời:

```bash
crontab -e
```

Thêm:

```cron
* * * * * /bin/bash /home/YOURUSER/day6-module2/health-check.sh >> /home/YOURUSER/day6-module2/cron-health.log 2>&1
```

**Không copy `YOURUSER`.**

Lấy actual path bằng:

```bash
realpath "$HOME/day6-module2/health-check.sh"
```

và actual home:

```bash
printf '%s\n' "$HOME"
```

---

# 96. Verify cron bằng evidence

Sau lần execution dự kiến, xem:

```bash
tail -n 30 "$HOME/day6-module2/cron-health.log"
```

Bạn phải thấy timestamp từ script.

Nếu log có nhiều runs:

```bash
grep 'event=health-check-start' \
    "$HOME/day6-module2/cron-health.log"
```

Evidence nên chứng minh:

```text
cron entry exists
script actually executed
timestamp matches schedule
checks executed
exit/result logged
```

---

# 97. Cron daemon logs

Tùy distro:

```bash
journalctl -u cron.service --since "15 minutes ago"
```

hoặc:

```bash
journalctl -u crond.service --since "15 minutes ago"
```

Cronie có thể log qua syslog/system logging infrastructure. :chatgpt-content-reference{index="35"}

Bạn đang kết nối Module 1 với Module 2:

```text
cron failure
   ↓
journalctl
   ↓
grep
   ↓
evidence
```

---

# 98. Sau test phải đổi schedule

`* * * * *` chỉ dùng để lab nhanh.

Nếu requirement là mỗi 15 phút:

```cron
*/15 * * * * /bin/bash /home/.../health-check.sh >> /home/.../cron-health.log 2>&1
```

Sau khi sửa:

```bash
crontab -l
```

để verify.

Đừng quên một test schedule chạy mỗi phút rồi bỏ đó.

---

# 99. Cron failure troubleshooting workflow

Giả sử:

```text
Script chạy tay OK.
Cron không tạo log.
```

Đừng rewrite script ngay.

Investigation:

```text
Expected schedule
        ↓
crontab exists?
        ↓
cron daemon active?
        ↓
cron attempted execution?
        ↓
correct user?
        ↓
path/interpreter?
        ↓
permissions?
        ↓
environment?
        ↓
working directory?
        ↓
stdout/stderr destination?
        ↓
script/application failure?
```

Đây là layer-by-layer troubleshooting.

---

# 100. Failure case: cron command not found

Symptom:

```text
manual OK
cron:
command not found
```

Evidence cần:

```bash
command -v command-name
```

```bash
printf '%s\n' "$PATH"
```

và nếu cần temporary diagnostic cron job trong lab:

```cron
* * * * * /usr/bin/env > /home/alice/day6-module2/cron-env.txt 2>&1
```

Sau đó inspect:

```bash
cat "$HOME/day6-module2/cron-env.txt"
```

Đừng để diagnostic entry tồn tại lâu hơn cần thiết.

---

# 101. Failure case: Permission denied

Phân biệt:

```text
script file not executable

parent directory inaccessible

output log not writable

command needs privilege

SELinux/security control

filesystem mounted noexec
```

Evidence:

```bash
namei -l /path/to/script 2>/dev/null
```

nếu `namei` có sẵn.

Hoặc:

```bash
ls -ld /path /path/to
ls -l /path/to/script
```

Check:

```bash
id
```

Quan trọng:

**Cron job chạy dưới user của crontab**, không dưới user bạn tưởng. Cronie xác nhận user crontab commands chạy với account sở hữu crontab. :chatgpt-content-reference{index="36"}

---

# 102. Không dùng root cron chỉ vì permission problem

Một fresher thấy:

```text
permission denied
```

rồi:

```bash
sudo crontab -e
```

và schedule toàn bộ script dưới root.

Đây là dangerous anti-pattern.

Trước tiên hỏi:

```text
Script thực sự cần privilege gì?

Có thể chỉ đọc world-readable metrics không?

Có thể grant narrow sudo command không?

Có thể đổi evidence directory ownership không?
```

Least privilege từ Day 3 vẫn áp dụng.

---

# 103. Failure case: Script chạy cron nhưng output khác

Có thể do:

```text
PATH
HOME
SHELL
locale
timezone
working directory
environment variables
permissions
non-interactive session
```

Đây là lý do automation phải minimize implicit dependencies.

Một script đáng tin cậy cố giảm assumptions.

---

# 104. Failure case: script overlap

Giả sử job:

```cron
* * * * *
```

nhưng script mất 3 phút.

Timeline:

```text
12:00 run A starts
12:01 run B starts
12:02 run C starts
```

Bây giờ có ba copies cùng chạy.

Health check read-only có thể chỉ tạo noise.

Backup job ở Module 3 có thể nguy hiểm hơn nhiều.

Cron bản thân không đảm bảo rằng previous job đã hoàn thành trước next schedule.

---

# 105. Supplementary — locking với `flock`

Trên Linux systems có util-linux `flock`, một common pattern:

```bash
flock -n /run/user/1000/health-check.lock \
    /path/health-check.sh
```

Hoặc locking trong script.

Nhưng:

```text
flock không thuộc explicit syllabus Day 6
```

và lock location/permissions phải thiết kế cẩn thận.

Bạn chỉ cần hiểu risk:

```text
scheduled execution can overlap.
```

Đến Module 3 điều này càng quan trọng với backup.

---

# 106. Supplementary — `trap`

Bash `trap` cho phép chạy action khi shell nhận một số signals hoặc pseudo-events như `EXIT`. GNU Bash định nghĩa builtin này chính thức. :chatgpt-content-reference{index="37"}

Ví dụ temporary file:

```bash
tmp_file=$(mktemp)

cleanup() {
    rm -f -- "$tmp_file"
}

trap cleanup EXIT
```

Dù script success hay nhiều failure paths, cleanup có cơ hội chạy.

Rất hữu ích trong automation.

Nhưng signal/error handling với `trap` có nhiều subtleties; đừng dùng một complex `ERR` trap trước khi hiểu Bash semantics.

---

# 107. Supplementary — temporary files

Không làm:

```bash
tmp="/tmp/output.tmp"
```

nếu attacker/multiple executions có thể collide.

Ưu tiên:

```bash
tmp=$(mktemp)
```

sau đó cleanup bằng `trap`.

Đây là security/robustness practice ngoài explicit syllabus.

---

# 108. Supplementary — idempotency

Một operation gọi là idempotent theo nghĩa thực hành khi chạy lại không tạo ra unintended accumulating effects.

Ví dụ:

```bash
mkdir -p "$dir"
```

thường dễ rerun hơn:

```bash
mkdir "$dir"
```

Nhưng idempotency không chỉ là thêm `-p`.

Bạn phải xét:

```text
file append?
duplicate users?
duplicate firewall rule?
backup overwrite?
restart?
external API calls?
```

Cron khiến idempotency cực kỳ quan trọng vì job sẽ chạy lặp lại.

---

# 109. Supplementary — timeout

Một command có thể hang:

```text
network request
remote filesystem
DNS
database client
```

Nếu một health check treo mãi, cron overlap có thể xuất hiện.

GNU/Linux thường có:

```bash
timeout 10s some-command
```

nhưng `timeout` là external utility, không phải Bash builtin và không nằm trong explicit Day 6 syllabus.

Khi dùng, bạn cũng phải hiểu exit behavior của nó.

---

# 110. Security — không hard-code secrets

Sai:

```bash
DB_PASSWORD="SuperSecret123"
```

trong:

```text
health-check.sh
```

Vì script có thể:

```text
được readable
được backup
được commit
được attach evidence
được trace bằng bash -x
```

Day 14 sẽ học secrets sâu hơn, nhưng nguyên tắc từ bây giờ:

```text
Không nhúng credentials vào health-check script.
```

---

# 111. Security — command injection

Nếu script nhận argument:

```bash
target="$1"
```

và làm:

```bash
eval "ping -c 1 $target"
```

đây là một pattern nguy hiểm.

`eval` làm shell parse lại string như command.

Với admin scripts chạy privilege cao, misuse có thể dẫn đến command injection.

Không dùng `eval` nếu không có lý do rất rõ.

Trong fresher operations tooling, gần như luôn có cách an toàn hơn.

---

# 112. Security — quote input

Ví dụ:

```bash
grep "$pattern" "$file"
```

tốt hơn unquoted expansions.

Và nếu argument có thể bắt đầu bằng `-`, nhiều commands hỗ trợ:

```bash
command -- "$value"
```

Ví dụ:

```bash
rm -- "$file"
```

`--` báo end-of-options cho commands hỗ trợ convention này.

---

# 113. Security — cron file permissions

Crontabs và scripts scheduled tự động là security-sensitive.

Nếu attacker có thể sửa script cron của privileged user:

```text
attacker effectively controls scheduled code execution.
```

Vì vậy kiểm tra:

```bash
ls -l /path/to/script
ls -ld /path/to/script-directory
```

Một root cron script không nên nằm trong directory writable bởi arbitrary users.

Cronie cũng đặt permission expectations lên crontab files. :chatgpt-content-reference{index="38"}

---

# 114. Security — logging secrets

Đừng:

```bash
printf 'Using token %s\n' "$API_TOKEN"
```

Và cẩn thận với:

```bash
set -x
```

Nếu debugging secret-bearing script, tracing có thể tiết lộ secret vào cron log/journal/ticket.

---

# 115. Operational behavior: stdout vs stderr

Good script nên suy nghĩ:

```text
normal report → stdout

execution/internal error → stderr
```

Ví dụ:

```bash
printf 'OK service=ssh\n'
```

stdout.

Internal error:

```bash
printf 'ERROR: required command ss not found\n' >&2
```

stderr.

Điều này giúp callers redirect riêng.

---

# 116. Dependency prechecks

Health script dùng:

```text
systemctl
df
awk
grep
ss
date
hostname
```

Bạn có thể validate:

```bash
require_command() {
    local command_name="$1"

    if ! command -v "$command_name" >/dev/null 2>&1; then
        printf 'ERROR missing-command=%s\n' "$command_name" >&2
        return 1
    fi
}
```

Sau đó:

```bash
require_command systemctl || exit 2
require_command df || exit 2
require_command awk || exit 2
```

Điều này tạo error rõ hơn:

```text
missing-command=ss
```

thay vì downstream nonsense.

---

# 117. Configuration validation

Nếu:

```bash
DISK_THRESHOLD=banana
```

rồi:

```bash
[[ "$usage" -ge "$DISK_THRESHOLD" ]]
```

behavior không còn đúng với intention.

Validate:

```bash
if [[ ! "$DISK_THRESHOLD" =~ ^[0-9]+$ ]]; then
    printf 'ERROR invalid DISK_THRESHOLD=%q\n' "$DISK_THRESHOLD" >&2
    exit 2
fi
```

Range:

```bash
if (( DISK_THRESHOLD < 1 || DISK_THRESHOLD > 100 )); then
    printf 'ERROR threshold out of range\n' >&2
    exit 2
fi
```

Admin tooling phải validate config trước khi tin nó.

---

# 118. Health-check script tốt không nên tự remediate mặc định

Một tempting design:

```text
if service down
→ automatically restart
```

Nhưng Day 6 requirement là health check/scheduling/troubleshooting, không phải autonomous remediation.

Health-check script ban đầu nên:

```text
observe
report
exit appropriately
```

chứ không:

```text
detect → mutate system
```

Nếu auto-remediation được yêu cầu sau này, cần runbook, approval, retry limits, blast-radius considerations và rollback/escalation.

---

# 119. L1 boundary

Script có thể báo:

```text
CRITICAL service=app.service state=not-active
```

L1 operator không nên tự động suy ra:

```bash
sudo systemctl restart app.service
```

Nếu runbook không authorize.

Bước tiếp theo nên là evidence:

```bash
systemctl status app.service --no-pager -l
journalctl -u app.service -b --since "..."
```

Module 1 + Module 2 kết nối tại đây.

---

# 120. Integrated troubleshooting scenario

Symptom:

```text
Health-check cron log stopped updating.
```

Bạn nên chia layer:

```text
Layer 1: cron scheduling
Layer 2: shell execution
Layer 3: script dependencies
Layer 4: health commands
Layer 5: target service/system
Layer 6: evidence output
```

Ví dụ:

```text
cron daemon active?
      ↓
crontab entry correct?
      ↓
cron attempted?
      ↓
script executable/interpreter valid?
      ↓
PATH/permission?
      ↓
script started?
      ↓
which check failed?
      ↓
target actually unhealthy?
```

Không nhảy thẳng tới:

```text
"cron hỏng."
```

---

# 121. Guided Failure Injection 1 — bad service

Run:

```bash
SERVICE_NAME=not-real-day6.service \
"$HOME/day6-module2/health-check.sh" \
    > "$HOME/day6-module2/failure-service.log" \
    2>&1

rc=$?

printf 'rc=%d\n' "$rc"
```

Expected invariant:

```text
service check must report not-active/nonhealthy
overall rc must be non-zero
```

Evidence:

```bash
cat "$HOME/day6-module2/failure-service.log"
```

---

# 122. Guided Failure Injection 2 — bad mount

```bash
MOUNT_POINT=/not-a-real-day6-mount \
"$HOME/day6-module2/health-check.sh" \
    > "$HOME/day6-module2/failure-disk.log" \
    2>&1

rc=$?

printf 'rc=%d\n' "$rc"
```

Expected:

```text
disk check UNKNOWN
```

Không phải:

```text
disk usage critical
```

Hai failure types khác nhau.

---

# 123. Guided Failure Injection 3 — cron environment

Trong lab, tạo script:

```bash
cat > "$HOME/day6-module2/show-env.sh" <<'EOF'
#!/bin/bash

printf '%s\n' "DATE=$(date '+%Y-%m-%dT%H:%M:%S%z')"
printf '%s\n' "USER=${USER:-unset}"
printf '%s\n' "HOME=${HOME:-unset}"
printf '%s\n' "SHELL=${SHELL:-unset}"
printf '%s\n' "PATH=${PATH:-unset}"
printf '%s\n' "PWD=$(pwd)"
EOF
```

Validate:

```bash
bash -n "$HOME/day6-module2/show-env.sh"
chmod 750 "$HOME/day6-module2/show-env.sh"
```

Manual:

```bash
"$HOME/day6-module2/show-env.sh" \
    > "$HOME/day6-module2/manual-env.txt"
```

Temporarily cron:

```cron
* * * * * /bin/bash /home/YOU/day6-module2/show-env.sh > /home/YOU/day6-module2/cron-env.txt 2>&1
```

Sau khi chạy, compare:

```bash
diff -u \
    "$HOME/day6-module2/manual-env.txt" \
    "$HOME/day6-module2/cron-env.txt"
```

Đây là lab cực kỳ hữu ích để **tự nhìn thấy** cron environment khác thế nào thay vì chỉ học lý thuyết.

---

# 124. Cleanup diagnostic cron entries

Sau lab:

```bash
crontab -e
```

Xóa `show-env.sh` diagnostic entry.

Sau đó:

```bash
crontab -l
```

để chứng minh cleanup đúng.

Không dùng:

```bash
crontab -r
```

một cách tùy tiện.

`crontab -r` có thể remove toàn bộ user's crontab tùy implementation/options.

Đây là destructive action.

---

# 125. Cron rollback

Nếu bạn vừa deploy schedule mới và thấy vấn đề:

Rollback đơn giản có thể là:

```text
restore previous crontab entry
```

Vì vậy trước change:

```bash
crontab -l > "$HOME/day6-module2/crontab.before"
```

Sau change:

```bash
crontab -l > "$HOME/day6-module2/crontab.after"
```

Compare:

```bash
diff -u \
    "$HOME/day6-module2/crontab.before" \
    "$HOME/day6-module2/crontab.after"
```

Đây là change-evidence mindset.

---

# 126. Nhưng backup crontab có thể chứa sensitive commands

Nếu cron entries chứa:

```text
internal paths
tokens embedded incorrectly
customer identifiers
backup locations
```

file evidence cũng nhạy cảm.

Bảo vệ:

```bash
chmod 600 "$HOME/day6-module2/crontab.before"
```

Và tốt hơn hết: đừng embed secrets trong crontab.

---

# 127. Supplementary — Cron vs systemd timers

Trên systemd systems, `systemd.timer` là một alternative mạnh cho scheduling.

Nó có thể tích hợp với:

```text
systemd units
journal
dependencies
state
```

Nhưng **Day 6 syllabus yêu cầu cron**, nên chúng ta không thay cron bằng systemd timers.

Bạn cần học cron đầy đủ.

systemd timers chỉ là supplementary awareness.

---

# 128. Common anti-pattern: `sh script.sh` với Bash syntax

Script:

```bash
#!/bin/bash

[[ "$x" == "yes" ]]
```

Bạn chạy:

```bash
sh script.sh
```

Bây giờ shebang bị bỏ qua vì bạn đã explicitly invoke `sh`.

Nếu `/bin/sh` là `dash` hoặc shell khác:

```text
Bash-specific syntax có thể fail.
```

Nếu script là Bash:

```bash
bash script.sh
```

hoặc:

```bash
./script.sh
```

với proper shebang.

---

# 129. Common anti-pattern: extension quyết định interpreter

Tên:

```text
health-check.sh
```

không đảm bảo Bash.

Interpreter được quyết định bởi:

```text
how it is invoked
+
shebang when directly executed
```

`.sh` chỉ là naming convention.

---

# 130. Common anti-pattern: unquoted variables

Nguy hiểm:

```bash
rm $file
cp $source $dest
grep $pattern $file
```

Better:

```bash
rm -- "$file"
cp -- "$source" "$dest"
grep -- "$pattern" "$file"
```

subject to command-specific option support.

---

# 131. Common anti-pattern: parse `ls`

Không làm:

```bash
for file in $(ls /some/path); do
    ...
done
```

Filename có spaces/newlines/globs làm logic fragile.

Shell filesystem processing là một chủ đề lớn; Module 2 chỉ cần nhớ:

```text
don't treat ls output as a stable machine data interface.
```

---

# 132. Common anti-pattern: `cat file | grep`

Không phải luôn “sai”, nhưng:

```bash
grep pattern file
```

đơn giản hơn:

```bash
cat file | grep pattern
```

Pipeline nên có lý do.

Càng nhiều process, càng nhiều failure points.

---

# 133. Common anti-pattern: overwrite log

Cron:

```cron
* * * * * script > health.log
```

Mỗi lần chạy:

```text
old evidence overwritten
```

Có lúc đó chính là intention.

Nhưng nếu muốn history:

```cron
* * * * * script >> health.log 2>&1
```

sau đó bạn lại phải nghĩ:

```text
log rotation
disk growth
retention
```

Automation luôn tạo thêm operational responsibilities.

---

# 134. Common anti-pattern: log tăng mãi

Một job mỗi phút:

```text
1440 runs/day
```

Nếu mỗi run ghi 10 KB:

```text
~14 MB/day
```

Theo thời gian log có thể fill filesystem.

Trong Module 3 khi backup và consolidation, chúng ta sẽ xét retention rõ hơn.

Tạm thời hãy nhận thức:

```text
evidence itself consumes capacity.
```

---

# 135. Common anti-pattern: cron every minute forever

Lab:

```cron
* * * * *
```

rất tiện.

Production:

```text
có thật sự cần 60 checks/hour không?
```

Frequency phải dựa trên:

```text
business requirement
failure detection objective
system load
downstream costs
log volume
```

Không chọn cadence chỉ vì syntax dễ.

---

# 136. Common anti-pattern: script tự `sudo`

Ví dụ:

```bash
sudo systemctl status ...
```

trong cron có thể fail vì:

```text
no terminal
password prompt
sudo policy
```

Và hard-coding privileged behavior trong script gây security issues.

Nếu automation cần privilege:

```text
design privilege explicitly
```

không mong `sudo` interactive magically hoạt động.

---

# 137. Common anti-pattern: silent failure

Cron:

```cron
* * * * * /path/job >/dev/null 2>&1
```

Ngay từ đầu lab thì đây là ý tưởng tệ.

Bạn vừa vứt:

```text
stdout
stderr
evidence
```

Khi đã có monitoring/alerting mature, suppress/route outputs có thể hợp lý.

Nhưng lúc học và initial rollout, hãy giữ evidence.

---

# 138. Common anti-pattern: success message unconditional

Sai:

```bash
backup-command
printf 'SUCCESS\n'
```

Nếu backup command fail, script vẫn print SUCCESS.

Đúng:

```bash
if backup-command; then
    printf 'SUCCESS\n'
else
    printf 'FAILED\n' >&2
    exit 1
fi
```

Đây là một trong những lý do exit codes nằm trong syllabus Day 6.

---

# 139. Common anti-pattern: final command accidentally determines script status

Script:

```bash
some-critical-command
printf 'Script completed\n'
```

Nếu critical command fail nhưng `printf` success, script cuối cùng có thể exit theo last command status và thành `0` nếu bạn không explicitly handle status.

GNU Bash mặc định script returns status của last executed command nếu không có explicit exit/syntax-error condition liên quan. :chatgpt-content-reference{index="39"}

Do đó final script contract phải deliberate.

---

# 140. Operational logging pattern

Một simple log helper:

```bash
log() {
    local level="$1"
    shift

    printf '%s %-8s %s\n' \
        "$(date '+%Y-%m-%dT%H:%M:%S%z')" \
        "$level" \
        "$*"
}
```

Usage:

```bash
log INFO "event=start"
log OK "check=disk usage=35%"
log CRITICAL "check=service state=down"
```

Lưu ý `"$*"` ở đây là intentional formatting design, không phải general argument-forwarding pattern.

---

# 141. Structured functions nên có contracts

Ví dụ documentation ngay trên function:

```bash
# check_service SERVICE
# Returns:
#   0 - service active
#   1 - service not active
#   2 - unable to determine
check_service() {
    ...
}
```

Điều này cực kỳ có ích khi Day 18 bạn làm operations handover.

---

# 142. Evidence pack cho Module 2

Một evidence pack tốt nên có artifacts thể hiện:

| Evidence                         | Chứng minh                           |
| -------------------------------- | ------------------------------------ |
| `bash --version`                 | Interpreter context                  |
| `bash -n health-check.sh` result | Syntax validation                    |
| `ls -l health-check.sh`          | Ownership/permissions                |
| Normal run output                | Happy path                           |
| Normal run exit code             | Machine contract                     |
| Failure injection output         | Failure detection                    |
| Failure run exit code            | Correct propagation                  |
| `crontab -l`                     | Schedule installed                   |
| Cron-produced log                | Scheduled execution happened         |
| cron/crond journal excerpt       | Scheduler evidence nếu cần           |
| Before/after crontab diff        | Change evidence                      |
| Cleanup evidence                 | Lab does not leave unwanted schedule |

Đây không chỉ là “screenshot để nộp”.

Nó dạy bạn chứng minh operational state.

---

# 143. Troubleshooting case study 1

Symptom:

```text
cron-health.log không được tạo.
```

Hypotheses:

```text
cron daemon down
entry missing
path wrong
directory permission
interpreter path wrong
job chưa tới schedule
wrong user's crontab
redirection destination unwritable
```

Safe tests:

```bash
crontab -l
```

```bash
systemctl status cron.service
```

hoặc:

```bash
systemctl status crond.service
```

```bash
ls -ld "$HOME/day6-module2"
```

```bash
ls -l "$HOME/day6-module2/health-check.sh"
```

```bash
/bin/bash "$HOME/day6-module2/health-check.sh"
```

Sau đó scheduler logs.

---

# 144. Troubleshooting case study 2

Symptom:

```text
cron-health.log có:
ss: command not found
```

Evidence cho biết:

```text
scheduler worked
script started
failure inside runtime environment
```

Do đó bạn đã loại trừ:

```text
cron schedule parsing
cron daemon inactivity
script not launched
```

Bây giờ tập trung:

```text
PATH / executable location / dependency
```

Đó là evidence-driven narrowing.

---

# 145. Troubleshooting case study 3

Symptom:

```text
health script exit=1
```

Bạn không được kết luận:

```text
script broken
```

Theo contract của chúng ta:

```text
1 = health condition critical
2 = checker/internal problem
```

Đầu tiên đọc output:

```text
CRITICAL service=...
```

Có thể script đang hoạt động hoàn hảo bằng cách phát hiện target failure.

Automation failure và detected infrastructure failure là hai khái niệm khác nhau.

---

# 146. Troubleshooting case study 4

Symptom:

```text
script chạy mỗi phút dù tôi muốn mỗi 10 phút
```

Crontab:

```cron
* */10 * * * ...
```

Sai mental model.

`*/10` ở **hour field** là mỗi 10 giờ trong hour range, còn minute field vẫn `*`.

Nếu mỗi 10 phút:

```cron
*/10 * * * *
```

Luôn xác định field positions trước khi đọc expression.

---

# 147. Troubleshooting case study 5

Symptom:

```text
job chạy giờ khác dự kiến
```

Evidence:

```bash
date
timedatectl
```

Inspect crontab:

```bash
crontab -l
```

Check `CRON_TZ` nếu Cronie/config có sử dụng.

Đừng sửa expression cho tới khi biết schedule đang interpreted trong timezone nào.

---

# 148. Recovery/Rollback

Bash script deployment rollback có thể là:

```text
restore previous known-good script
```

Vì vậy trước edit:

```bash
cp -a health-check.sh health-check.sh.bak
```

Sau edit:

```bash
diff -u health-check.sh.bak health-check.sh
bash -n health-check.sh
```

Nếu new version bad:

```bash
cp -a health-check.sh.bak health-check.sh
```

Nhưng trước overwrite:

```text
confirm path
confirm backup integrity
```

---

# 149. Better operational deployment pattern

Trong managed environment, thường tốt hơn:

```text
version-controlled source
        ↓
review
        ↓
deploy
        ↓
validate
        ↓
rollback known version
```

thay vì:

```text
ssh vào server
nano script
save
hope
```

Version control nằm ngoài explicit syllabus nhưng mindset này quan trọng.

---

# 150. Exam Focus — Bash

Bạn cần giải thích được:

```bash
#!/bin/bash
```

làm gì.

Bạn cần hiểu khác biệt:

```bash
bash script.sh
```

với:

```bash
./script.sh
```

Bạn phải hiểu:

```text
variable assignment
parameter expansion
quoting
command substitution
positional parameters
if
numeric/string/file tests
functions
return vs exit
loops cơ bản
stdout/stderr
```

---

# 151. Exam Focus — exit codes

Bạn phải giải thích:

```text
0
```

là success convention.

```text
non-zero
```

không tự động cùng một meaning cho mọi command.

Bạn phải hiểu:

```bash
$?
```

chỉ giữ status command gần nhất.

Bạn phải nhận ra bug:

```bash
command
echo done
rc=$?
```

`rc` là status `echo`, không phải command ban đầu.

---

# 152. Exam Focus — pipelines

Bạn phải biết mặc định:

```bash
command1 | command2
```

pipeline status thường đến từ command cuối trong Bash nếu `pipefail` off. :chatgpt-content-reference{index="40"}

Và hiểu vì sao:

```bash
set -o pipefail
```

có thể hữu ích.

Nhưng không được nói:

> “pipefail làm mọi pipeline an toàn.”

Nó chỉ thay đổi status semantics.

---

# 153. Exam Focus — cron

Bạn phải đọc được:

```cron
*/5 * * * *
```

```cron
0 2 * * *
```

```cron
30 1 * * 1
```

và phân biệt:

```text
user crontab
```

với:

```text
/etc/crontab / /etc/cron.d
```

Bạn cũng phải biết tại sao manual success không chứng minh cron success.

---

# 154. Knowledge Check

Tự trả lời trước khi nhìn lại bài:

| Câu hỏi                                                  | Điểm cần reasoning                                    |
| -------------------------------------------------------- | ----------------------------------------------------- |
| Tại sao phải quote `"$file"`?                            | Word splitting/globbing/safe argument boundaries      |
| `$(command)` cho bạn gì?                                 | Captured stdout, không thay thế việc check status     |
| `$?` chứa gì?                                            | Status command/pipeline gần nhất                      |
| `return` khác `exit` thế nào?                            | Function vs script/shell                              |
| Vì sao `if command; then` thường tốt?                    | Branch trực tiếp theo command status                  |
| Exit 1 của `grep` có luôn là execution error không?      | Không                                                 |
| Pipeline mặc định lấy status command nào?                | Last command trong Bash nếu không `pipefail`          |
| `set -e` có đơn giản là “fail nào cũng exit” không?      | Không                                                 |
| Vì sao cron script nên dùng absolute paths?              | Cron environment/working-directory differences        |
| User crontab có username field không?                    | Không                                                 |
| `/etc/crontab` có thể có username field không?           | Có                                                    |
| Vì sao cron mỗi phút có thể overlap?                     | Previous execution có thể chưa hoàn thành             |
| Vì sao không redirect mọi thứ vào `/dev/null` ngay?      | Mất evidence                                          |
| Health-check có nên auto-restart service mặc định không? | Không nếu chưa có explicit remediation design/runbook |
| Script rc=1 có chắc script bug không?                    | Không; có thể health condition detected               |

---

# 155. Independent Practical Challenge

Không sửa script mẫu trực tiếp trong lúc chạy challenge.

Hãy tự xây một script mới:

```text
linux-health.sh
```

Nó phải kiểm tra ít nhất:

```text
hostname/time
one systemd service
root filesystem usage
one listening TCP port
```

Yêu cầu behavior:

```text
OK condition       → clear output
health failure     → clear CRITICAL output
inspection failure → clear UNKNOWN output
final exit code    → reflects overall result
```

Sau đó bạn phải test:

```text
normal case
fake service
invalid mount
non-listening port
```

Cuối cùng schedule script bằng cron trong lab và chứng minh execution bằng timestamped log.

Không stop SSH nếu bạn đang SSH vào lab machine.

---

# 156. Mastery Check — bạn chưa hoàn thành Module 2 chỉ vì script chạy

Tôi chỉ coi Module 2 đạt practical mastery khi bạn có thể giải thích toàn bộ flow này:

```text
           CRONTAB
              │
              │ time matches
              ▼
          cron daemon
              │
              │ execution environment
              ▼
          /bin/bash
              │
              ▼
       health-check.sh
              │
      ┌───────┼────────┐
      ▼       ▼        ▼
   service   disk     port
    check    check    check
      │       │        │
      └───────┼────────┘
              ▼
        individual rc
              │
              ▼
         overall rc
              │
       ┌──────┴──────┐
       ▼             ▼
    stdout         stderr
       │             │
       └──────┬──────┘
              ▼
         evidence log
```

Và nếu output không xuất hiện, bạn phải troubleshoot:

```text
schedule
   ↓
daemon
   ↓
user
   ↓
environment
   ↓
interpreter
   ↓
permissions
   ↓
dependencies
   ↓
script logic
   ↓
target system
   ↓
evidence destination
```

thay vì thử sửa lung tung.

---

# 157. Mối liên hệ với các Day sau

Những gì bạn đang học ở đây không dừng tại Linux.

Day 12–14, bạn sẽ dùng cùng tư duy để:

```text
start Java/Tomcat
check service
check endpoint
validate deployment
safe restart
rollback
```

Day 17 monitoring:

```text
health signals
alerts
failure detection
```

Day 18 operations controls đặc biệt quay lại:

```text
batch scheduling
dependency checks
exit-code checks
evidence
rollback
handover
```

Master syllabus ghi rõ **Day 18 có dependency và exit-code checks**, nên exit codes hôm nay chính là nền móng cho operations workflow sau này. :chatgpt-content-reference{index="41"}

---

# 158. Mental model cuối cùng của Module 2

Nếu chỉ giữ lại một chuỗi tư duy, hãy giữ:

```text
Manual command
      ↓
Understand its inputs/output/status
      ↓
Wrap it in a function
      ↓
Validate prerequisites
      ↓
Handle success/failure explicitly
      ↓
Aggregate results
      ↓
Return meaningful exit code
      ↓
Produce timestamped evidence
      ↓
Test happy path
      ↓
Inject safe failure
      ↓
Verify detection
      ↓
Run non-interactively
      ↓
Schedule with cron
      ↓
Prove cron actually ran
      ↓
Troubleshoot by layer
      ↓
Preserve evidence / escalate
```

Và nguyên tắc quan trọng nhất của cả Module 2 là:

> **Automation không làm một quy trình sai trở nên đúng. Nó chỉ làm quy trình đó chạy nhanh hơn và lặp lại nhiều hơn.**

Trước khi tự động hóa một command, bạn phải hiểu command đó làm gì, thay đổi state gì, exit status nghĩa gì, failure mode là gì, evidence ở đâu và rollback/escalation thế nào.

**Module 2 đến đây đã bao phủ đầy đủ phần kiến thức Bash scripting + exit codes + cron của Day 6. Module 3 sẽ nối trực tiếp từ đây sang `tar`/`rsync`, backup verification/restore, hoàn thiện health-check + backup automation, scheduling, failure injection và Linux consolidation capstone.**
