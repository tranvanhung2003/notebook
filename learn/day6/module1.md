# Day 6 — Module 1: Linux Logs & Text Processing for Troubleshooting

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 6 bắt buộc bao gồm:

> `Logs with journalctl, grep/awk/sed, Bash scripting, exit codes, cron, backup using tar/rsync, and Linux consolidation.`

Assignment cuối ngày là xây dựng health-check + backup script, schedule nó, inject failure và lập escalation evidence. :chatgpt-content-reference{index="0"}

Trong **Module 1**, chúng ta tập trung chính xác vào phần:

**`journalctl` + `grep` + `awk` + `sed` + log analysis/evidence-driven troubleshooting.**

Bash scripting, exit codes và cron thuộc Module 2; `tar`, `rsync` và integrated capstone thuộc Module 3. Tuy nhiên, tôi sẽ giới thiệu một vài khái niệm pipeline/redirection cần thiết để Module 1 có thể thực hành được.

### Mandatory của Module 1

| Nội dung                                       | Mức độ                             |
| ---------------------------------------------- | ---------------------------------- |
| Linux logging mental model                     | Bắt buộc để hiểu `journalctl`      |
| `systemd-journald` / journal                   | Bắt buộc                           |
| `journalctl`                                   | Bắt buộc                           |
| `grep`                                         | Bắt buộc                           |
| `awk`                                          | Bắt buộc                           |
| `sed`                                          | Bắt buộc                           |
| Kết hợp công cụ thành troubleshooting pipeline | Bắt buộc trong Linux consolidation |
| Thu thập evidence                              | Cần thiết cho Assignment Day 6     |

### Supplementary / Beyond the explicit syllabus

Regex chi tiết hơn mức căn bản, journal structured fields, persistence/retention của journald, evidence integrity, một số security considerations và cách thiết kế pipeline robust hơn không được syllabus viết riêng thành từng mục, nhưng tôi sẽ bổ sung vì chúng rất quan trọng cho vận hành thực tế.

---

# 2. Environment assumptions

Các ví dụ dưới đây giả định bạn đang dùng Linux sử dụng:

```text
systemd
systemd-journald
journalctl
GNU grep
GNU awk/gawk hoặc POSIX-compatible awk
GNU sed
Bash-compatible shell
```

Ví dụ phù hợp nhất với Ubuntu/Debian, RHEL/Rocky/AlmaLinux/Amazon Linux và các distro systemd-based tương tự.

Bạn nên kiểm tra môi trường trước:

```bash
cat /etc/os-release
systemctl --version
journalctl --version
grep --version
awk --version 2>/dev/null || awk -W version
sed --version
```

Không nên mặc định mọi Linux giống nhau.

Ví dụ macOS/BSD có thể dùng implementation khác cho `grep`, `sed`, `awk`; macOS cũng không dùng `systemd`, vì vậy `journalctl` không áp dụng.

---

# 3. Mục tiêu cuối Module 1

Sau khi hoàn thành Module 1, bạn không chỉ cần nhớ command.

Bạn phải nhìn một symptom như:

```text
Application không start sau lúc 14:32.
```

và tự xây dựng investigation:

```text
Symptom
   ↓
Xác định unit
   ↓
Xác định boot hiện tại hay boot cũ
   ↓
Xác định time window
   ↓
Thu thập journal
   ↓
Lọc noise
   ↓
Xác định relevant messages
   ↓
Trích field/value
   ↓
Correlation giữa các events
   ↓
Hypothesis
   ↓
Safe verification
   ↓
Root cause hoặc escalation evidence
```

Đó mới là mục tiêu của Module 1.

---

# 4. Mental Model: Linux log thực sự là gì?

Một fresher thường nghĩ:

> Log = file `.log` trong `/var/log`.

Điều đó không còn đầy đủ trên Linux hiện đại.

Với systemd, một luồng phổ biến là:

```text
Application / Service
      │
      ├── stdout
      ├── stderr
      ├── syslog
      └── systemd logging APIs
             │
             ▼
      systemd-journald
             │
             ▼
        systemd journal
             │
             ▼
         journalctl
             │
             ▼
     grep / awk / sed
             │
             ▼
    Operational evidence
```

Output của systemd-managed services thường có thể được journal thu thập và truy vấn bằng `journalctl`. Journal không chỉ là một text file khổng lồ; mỗi entry có thể chứa structured metadata như unit, PID, UID, boot ID, priority cùng `MESSAGE`. `journalctl` có thể filter trực tiếp trên metadata đó. :chatgpt-content-reference{index="1"}

Điều này tạo ra một khác biệt cực kỳ quan trọng:

```text
Traditional thinking:
open file → grep text

Better systemd thinking:
select structured logs → narrow scope → then process text
```

Ví dụ, thay vì:

```bash
grep ERROR /var/log/*
```

bạn nên nghĩ:

```bash
journalctl \
  -u myapp.service \
  -b \
  --since "14:30" \
  --until "14:40"
```

rồi mới tiếp tục lọc.

---

# 5. `systemd-journald` và `journalctl`

## 5.1 `journald` khác `journalctl`

Hai khái niệm này tuyệt đối không được nhầm.

```text
systemd-journald
```

là daemon/service nhận và quản lý journal.

Còn:

```text
journalctl
```

là client/query tool dùng để **đọc và truy vấn journal**.

Mental model:

```text
journald = database/log collector side

journalctl = query/read side
```

Bạn không dùng `journalctl` để “chạy logging service”.

---

# 6. Journal entry không chỉ có `MESSAGE`

Một entry về mặt conceptual có thể giống:

```text
MESSAGE=Failed to start application
PRIORITY=3
_PID=1247
_UID=1001
_SYSTEMD_UNIT=myapp.service
_BOOT_ID=abc123...
_HOSTNAME=server01
```

Bạn thường chỉ nhìn thấy dạng human-friendly:

```text
Oct 01 14:32:17 server01 myapp[1247]: Failed to start application
```

nhưng phía dưới còn metadata.

Đây là lý do:

```bash
journalctl -u myapp.service
```

thường tốt hơn:

```bash
journalctl | grep myapp
```

Cách đầu tiên filter theo metadata của systemd.

Cách thứ hai chỉ tìm string `"myapp"` trong text được render ra.

Hai thứ **không tương đương**.

---

# 7. Command căn bản nhất: `journalctl`

```bash
journalctl
```

Không có argument, nó hiển thị các journal entries mà user hiện tại có quyền đọc, theo thứ tự từ cũ đến mới. Output mặc định thường được đưa qua pager như `less`. :chatgpt-content-reference{index="2"}

Trong production system có nhiều log, command này thường quá rộng.

Đây là ví dụ của một command hợp lệ nhưng **không phải investigation strategy tốt**.

---

# 8. Pager và `--no-pager`

Khi chạy:

```bash
journalctl
```

bạn có thể rơi vào `less`.

Một số phím hữu ích:

```text
q       quit
/word   search forward
n       next occurrence
G       cuối output
g       đầu output
```

Trong automation hoặc khi redirect sang file, nên rõ ràng hơn:

```bash
journalctl --no-pager
```

Ví dụ:

```bash
journalctl -u ssh.service --no-pager
```

Điều này đặc biệt quan trọng khi sau này chúng ta viết script.

---

# 9. Filter theo systemd unit: `-u`

Đây là một trong những command quan trọng nhất toàn khóa Linux:

```bash
journalctl -u <unit>
```

Ví dụ:

```bash
journalctl -u ssh.service
```

Hoặc tùy distro:

```bash
journalctl -u sshd.service
```

`-u` tương đương:

```text
--unit=
```

Official systemd documentation mô tả rằng `--unit=` filter messages liên quan tới systemd unit cụ thể và còn xử lý một số message systemd/coredump liên quan đến unit đó. :chatgpt-content-reference{index="3"}

Điều này tốt hơn đáng kể so với:

```bash
journalctl | grep ssh
```

---

# 10. `systemctl status` và `journalctl` không giống nhau

Day 4 bạn đã học `systemctl`.

Đây là một distinction quan trọng.

```bash
systemctl status myapp.service
```

trả lời chủ yếu:

```text
Unit tồn tại không?
Loaded thế nào?
Enabled không?
Active/failed?
Main PID?
Một số log gần đây?
```

Trong khi:

```bash
journalctl -u myapp.service
```

là công cụ log investigation.

Mental model:

```text
systemctl status
       ↓
"State hiện tại là gì?"

journalctl
       ↓
"Chuyện gì đã xảy ra?"
```

Không nên chỉ nhìn `systemctl status` rồi kết luận root cause.

---

# 11. Filter theo boot với `-b`

Đây là command bạn phải hình thành reflex:

```bash
journalctl -b
```

Ý nghĩa:

> Chỉ xem logs thuộc boot hiện tại.

Tại sao quan trọng?

Giả sử server từng fail hôm qua:

```text
Yesterday boot:
14:30 ERROR cannot bind port

Today boot:
09:00 application started normally
```

Bạn chạy:

```bash
journalctl -u app.service | grep ERROR
```

và nhìn thấy lỗi hôm qua.

Bạn có thể kết luận sai rằng lỗi đó đang xảy ra hiện tại.

Do đó trong troubleshooting current incident:

```bash
journalctl -u app.service -b
```

thường là điểm khởi đầu rất tốt.

---

# 12. Boot trước: `-b -1`

Ví dụ server reboot sau incident.

Bạn cần biết trước reboot đã xảy ra chuyện gì.

```bash
journalctl -b -1
```

Nghĩa là:

```text
previous boot
```

Hai boot trước:

```bash
journalctl -b -2
```

Current boot:

```bash
journalctl -b
```

hoặc:

```bash
journalctl -b 0
```

`journalctl` còn có:

```bash
journalctl --list-boots
```

Official documentation xác nhận `--boot` dùng `_BOOT_ID` để giới hạn các records thuộc một boot cụ thể, và `--list-boots` liệt kê các boot mà journal còn giữ được. :chatgpt-content-reference{index="4"}

---

# 13. Vì sao `-b -1` đôi khi không có dữ liệu?

Bạn chạy:

```bash
journalctl -b -1
```

nhưng không thấy previous boot.

Không nhất thiết command sai.

Có khả năng journal của máy đang dùng **volatile storage**, nên journal cũ biến mất sau reboot.

`journald` hỗ trợ các storage mode như:

```text
volatile
persistent
auto
none
```

Với volatile storage, data nằm dưới:

```text
/run/log/journal
```

và mất khi reboot.

Persistent journal sử dụng:

```text
/var/log/journal
```

khi cấu hình và filesystem cho phép. Official `journald.conf` documentation mô tả chính xác sự khác nhau này. :chatgpt-content-reference{index="5"}

Kiểm tra:

```bash
grep -R '^[[:space:]]*Storage=' \
    /etc/systemd/journald.conf \
    /etc/systemd/journald.conf.d 2>/dev/null
```

Nhưng lưu ý:

Không thấy dòng `Storage=` **không có nghĩa persistent hoặc volatile cụ thể**, vì default/configuration/drop-in/environment vẫn phải được xét.

Có thể dùng:

```bash
journalctl --disk-usage
```

để xem journal hiện sử dụng bao nhiêu disk.

---

# 14. Filter theo thời gian: `--since`

Đây là một trong những kỹ năng investigation quan trọng nhất.

```bash
journalctl --since "2026-10-01 14:30:00"
```

Ví dụ:

```bash
journalctl -u myapp.service \
  --since "2026-10-01 14:30:00"
```

Hoặc relative time:

```bash
journalctl --since "10 minutes ago"
```

Có thể dùng:

```bash
journalctl --since today
```

Official `journalctl` documentation hỗ trợ absolute timestamps, `today`, `yesterday`, `now` và relative timestamps. :chatgpt-content-reference{index="6"}

---

# 15. Giới hạn cả hai đầu: `--since` + `--until`

Ví dụ incident được báo:

```text
14:31 → 14:36
```

Thay vì lấy toàn bộ log hôm nay:

```bash
journalctl -u myapp.service \
  --since "2026-10-01 14:30:00" \
  --until "2026-10-01 14:40:00"
```

Bạn cố ý lấy thêm vài phút trước và sau incident.

Tại sao?

Root cause thường xuất hiện **trước symptom**.

Ví dụ:

```text
14:31:01 disk nearly full
14:31:48 write failed
14:32:02 application unhealthy
14:32:10 monitoring alert
```

Nếu chỉ lấy từ 14:32:10, bạn bỏ mất causal evidence.

---

# 16. Điều tra tốt bắt đầu từ timeline

Một fresher thường tìm:

```text
ERROR
```

Senior operations engineer thường hỏi trước:

```text
Khi nào?
Sau thay đổi nào?
Chỉ service này hay cả host?
Trước failure 5 phút có gì?
Sau failure có auto-recovery không?
```

Log investigation trước hết là **timeline reconstruction**.

---

# 17. Xem những dòng gần nhất với `-n`

```bash
journalctl -n 50
```

nghĩa là xem 50 entries gần nhất.

Theo unit:

```bash
journalctl -u myapp.service -n 100
```

Current boot:

```bash
journalctl -u myapp.service -b -n 100
```

Đây là command nhanh khi bạn vừa xảy ra failure.

Nhưng không nên nghĩ:

> 100 lines gần nhất chắc chắn chứa root cause.

Nếu service tạo log rất nhanh, root cause có thể nằm xa hơn.

Time window vẫn đáng tin hơn.

---

# 18. Follow log realtime: `-f`

Tương tự mental model của:

```bash
tail -f file.log
```

với journal:

```bash
journalctl -f
```

Theo service:

```bash
journalctl -u myapp.service -f
```

Official systemd docs định nghĩa `-f/--follow` là tiếp tục in entries mới khi journal nhận chúng. :chatgpt-content-reference{index="7"}

Use case:

Terminal A:

```bash
journalctl -u myapp.service -f
```

Terminal B:

```bash
sudo systemctl restart myapp.service
```

Bạn quan sát boot sequence realtime.

### Operational caution

Không restart một service production chỉ để “xem log”.

Restart là state-changing operation và có thể:

```text
gây downtime
mất transient evidence
reset counters
đóng connections
làm incident khó reproduce
```

Trong lab thì được.

Production phải theo quyền hạn/change procedure.

---

# 19. Priority / severity

Syslog-style priorities thường được biểu diễn:

| Numeric | Name      | Ý nghĩa khái quát      |
| ------: | --------- | ---------------------- |
|       0 | `emerg`   | System unusable        |
|       1 | `alert`   | Cần action ngay        |
|       2 | `crit`    | Critical               |
|       3 | `err`     | Error                  |
|       4 | `warning` | Warning                |
|       5 | `notice`  | Significant but normal |
|       6 | `info`    | Informational          |
|       7 | `debug`   | Debug                  |

Bạn có thể filter:

```bash
journalctl -p err
```

Nhưng có một nuance rất quan trọng.

`-p err` không chỉ có **exactly `err`**.

Nó bao gồm severity `err` **và các mức nghiêm trọng hơn**, tức numeric thấp hơn. Official systemd docs mô tả single priority theo logic này. :chatgpt-content-reference{index="8"}

Để chỉ exact `err`:

```bash
journalctl -p err..err
```

Một range:

```bash
journalctl -p warning..alert
```

Trong thực tế thường dùng:

```bash
journalctl -u myapp.service -b -p warning
```

nghĩa là lấy warning và nghiêm trọng hơn trong current boot.

---

# 20. Không được coi severity là root cause

Một log:

```text
ERROR connection failed
```

có thể là consequence.

Root cause có thể xuất hiện trước đó:

```text
WARN DNS lookup slow
ERROR database connection failed
ERROR application startup failed
```

Vì vậy:

```bash
journalctl -p err
```

hữu ích để triage nhưng không đủ để root-cause analysis.

---

# 21. Output format

Mặc định journalctl sử dụng human-readable format.

Để timestamp rõ ràng hơn:

```bash
journalctl -o short-iso
```

Hoặc:

```bash
journalctl -o short-full
```

`short-full` hữu ích vì timestamp đầy đủ và format tương thích với input của `--since` / `--until`. systemd cũng hỗ trợ nhiều format structured hơn như `verbose`, JSON và `export`. :chatgpt-content-reference{index="9"}

Trong evidence pack tôi thường ưu tiên:

```bash
journalctl \
  -u myapp.service \
  --since "2026-10-01 14:30:00" \
  --until "2026-10-01 14:40:00" \
  -o short-iso \
  --no-pager
```

Lý do:

```text
có timestamp
có unit/process context
dễ đọc
dễ attach
dễ grep
```

---

# 22. `-o cat`

```bash
journalctl -u myapp.service -o cat
```

`cat` chỉ in message body rất ngắn gọn, bỏ metadata như timestamp. Official docs mô tả `cat` là output mode rất terse. :chatgpt-content-reference{index="10"}

Rất tiện khi muốn xử lý text:

```bash
journalctl -u myapp.service -o cat | grep ERROR
```

nhưng nguy hiểm nếu lưu làm evidence duy nhất vì mất context.

Ví dụ:

```text
Connection refused
```

không cho biết:

```text
khi nào?
host nào?
process nào?
boot nào?
```

Vì vậy:

```text
-o cat
```

rất tốt cho processing;

nhưng:

```text
short-iso / short-full
```

tốt hơn cho evidence.

---

# 23. Structured journal fields

Một tính năng mạnh:

```bash
journalctl -o verbose
```

Bạn có thể thấy các field kiểu:

```text
MESSAGE=...
PRIORITY=...
_PID=...
_UID=...
_GID=...
_HOSTNAME=...
_SYSTEMD_UNIT=...
_BOOT_ID=...
```

Sau đó có thể filter trực tiếp:

```bash
journalctl _SYSTEMD_UNIT=myapp.service
```

Hoặc kết hợp:

```bash
journalctl \
  _SYSTEMD_UNIT=myapp.service \
  _PID=1234
```

Với different fields, điều kiện về cơ bản được kết hợp như AND; nhiều value cho cùng một field được xử lý như alternatives. Đây là behavior được systemd documentation mô tả trực tiếp. :chatgpt-content-reference{index="11"}

Mental model:

```text
_SYSTEMD_UNIT=myapp.service
AND
_PID=1234
```

Khả năng này là lý do journal mạnh hơn một flat log file.

---

# 24. `journalctl -g` và `grep` khác nhau

Các systemd hiện đại hỗ trợ:

```bash
journalctl -g 'pattern'
```

hoặc:

```bash
journalctl --grep='pattern'
```

Nó filter trường `MESSAGE` bằng PCRE2 regular expression. Feature này được systemd thêm từ version 237. :chatgpt-content-reference{index="12"}

Ví dụ:

```bash
journalctl -u myapp.service -g 'timeout|refused'
```

Tuy nhiên syllabus yêu cầu bạn phải học GNU `grep`, nên không được thay `grep` bằng `journalctl -g`.

Ngoài ra system cũ có thể không hỗ trợ `-g`.

Kiểm tra:

```bash
journalctl --version
journalctl --help | grep -- '--grep'
```

---

# 25. Access permissions với journal

Normal user không nhất thiết đọc được tất cả system logs.

Official documentation ghi rằng quyền đọc system journal phụ thuộc user/group và distro configuration; các group như `systemd-journal`, `adm`, `wheel` có thể có quyền đọc rộng hơn tùy hệ thống. :chatgpt-content-reference{index="13"}

Nếu:

```bash
journalctl -u some.service
```

không hiện đủ log nhưng:

```bash
sudo journalctl -u some.service
```

lại hiện, đó có thể là privilege issue.

Nhưng không nên giải quyết bằng:

```text
cho mọi user sudo/root
```

Đó đi ngược least privilege của Day 3.

---

# 26. `grep` — công cụ tìm và lọc text

Sau khi journalctl thu hẹp đúng dataset, `grep` giúp chọn các dòng phù hợp pattern.

General form:

```bash
grep [options] pattern [file...]
```

GNU documentation mô tả chính xác synopsis này và khuyến nghị pattern thường nên được quote khi dùng trong shell. :chatgpt-content-reference{index="14"}

Ví dụ file:

```bash
grep 'ERROR' app.log
```

Từ stdin:

```bash
journalctl -u myapp.service | grep 'ERROR'
```

Đây là khái niệm cực quan trọng:

```text
grep file
```

và:

```text
producer | grep
```

đều khả thi.

---

# 27. Standard input / standard output

Ví dụ:

```bash
journalctl -u app.service
```

ghi output ra stdout.

Pipe:

```bash
|
```

đưa stdout đó trở thành stdin của command tiếp theo:

```text
journalctl stdout
       │
       ▼
      pipe
       │
       ▼
grep stdin
```

Ví dụ:

```bash
journalctl -u app.service --no-pager |
grep 'ERROR'
```

Không có temporary file.

---

# 28. Case-sensitive và `-i`

Mặc định:

```bash
grep 'error'
```

khác:

```bash
grep 'ERROR'
```

Case insensitive:

```bash
grep -i 'error'
```

sẽ bắt:

```text
error
Error
ERROR
eRrOr
```

Operationally:

```bash
journalctl -u myapp.service |
grep -i 'error'
```

Nhưng keyword search không nên là evidence duy nhất.

Một application có thể báo failure bằng:

```text
FATAL
exception
failed
refused
timeout
unavailable
```

không chứa từ `ERROR`.

---

# 29. Line number với `-n`

Với text file:

```bash
grep -n 'ERROR' app.log
```

Ví dụ:

```text
347:ERROR database connection failed
```

Rất hữu ích khi trao đổi:

```text
app.log line 347
```

Với journal streaming, line number chỉ là vị trí trong output bạn tạo ra, không phải permanent journal identity.

---

# 30. Invert match `-v`

```bash
grep -v 'DEBUG'
```

nghĩa là:

> Giữ lại những dòng không match DEBUG.

Ví dụ:

```bash
journalctl -u app.service -o cat |
grep -v '^DEBUG'
```

Use case là giảm noise.

Nhưng có rủi ro:

Debug lines đôi khi chứa causal information.

Không nên remove rồi vứt source evidence.

Hãy giữ original evidence, filtered view chỉ dùng để phân tích.

---

# 31. Context: `-A`, `-B`, `-C`

Đây là nhóm option cực hữu ích trong troubleshooting.

After:

```bash
grep -A 3 'ERROR' app.log
```

3 dòng sau match.

Before:

```bash
grep -B 3 'ERROR' app.log
```

3 dòng trước.

Context cả hai:

```bash
grep -C 3 'ERROR' app.log
```

Một error message rất hiếm khi tự đủ context.

Ví dụ:

```text
Config loaded
Connecting database
Connection timeout
ERROR startup failed
Cleanup complete
Process exiting
```

Nếu chỉ:

```bash
grep ERROR
```

bạn thấy:

```text
ERROR startup failed
```

nhưng mất nguyên nhân:

```text
Connection timeout
```

---

# 32. Multiple patterns

Với extended regex:

```bash
grep -E 'ERROR|WARN|FATAL'
```

Case insensitive:

```bash
grep -Ei 'error|warn|fatal'
```

Ví dụ:

```bash
journalctl -u app.service -b --no-pager |
grep -Ei 'error|failed|timeout|refused'
```

Đây là triage tool tốt.

Nhưng cần nhớ:

```text
keyword hit ≠ root cause
```

---

# 33. Basic Regex vs Extended Regex

Đây là điểm nên hiểu rõ.

Mặc định:

```bash
grep
```

sử dụng Basic Regular Expressions theo mode thông thường.

```bash
grep -E
```

sử dụng Extended Regular Expressions.

Ví dụ ERE:

```bash
grep -E 'error|failed'
```

hoặc:

```bash
grep -E 'HTTP [45][0-9][0-9]'
```

Nếu bạn muốn literal string chứ không regex:

```bash
grep -F 'a.b[c]'
```

`-F` rất hữu ích khi pattern chứa ký tự regex nhưng bạn chỉ muốn exact text-like matching.

---

# 34. Regex mental model

Một vài ký hiệu bạn cần quen:

| Pattern               | Ý nghĩa                                     |
| --------------------- | ------------------------------------------- | --------------------------- |
| `^ERROR`              | dòng bắt đầu bằng `ERROR`                   |
| `failed$`             | dòng kết thúc bằng `failed`                 |
| `.`                   | một character bất kỳ                        |
| `[0-9]`               | một digit                                   |
| `[^0-9]`              | không phải digit                            |
| `*`                   | previous expression lặp zero hoặc nhiều lần |
| `+` với ERE           | một hoặc nhiều lần                          |
| `?` với ERE           | zero hoặc một lần                           |
| `A\|B` trong BRE / `A | B` trong ERE                                | alternatives tùy regex mode |

Do đó:

```bash
grep -E 'status=[45][0-9][0-9]'
```

match:

```text
status=404
status=500
status=503
```

nhưng không match:

```text
status=200
```

---

# 35. Quote regex

Nên viết:

```bash
grep -E 'ERROR|WARN'
```

thay vì:

```bash
grep -E ERROR|WARN
```

Command thứ hai bị shell hiểu dấu `|` là pipe.

Shell parsing xảy ra **trước** grep.

Đây là lỗi rất phổ biến.

---

# 36. Recursive `grep`

```bash
grep -R 'database.url' /etc/myapp/
```

Hoặc:

```bash
grep -Rn 'database.url' /etc/myapp/
```

Rất hữu ích để tìm configuration.

Nhưng trước khi recursive grep một tree lớn:

```text
hãy nghĩ về scope
```

Ví dụ:

```bash
sudo grep -R password /
```

là một ý tưởng rất tệ về cả performance, privacy lẫn security.

---

# 37. `grep` exit status — preview cực quan trọng cho Module 2

Syllabus chính thức đặt exit codes ở Module 2, nhưng ở đây bạn cần biết trước behavior của `grep`.

GNU grep trả về:

```text
0 = có ít nhất một selected line
1 = không có selected line
2 = error
```

Ngoại lệ liên quan `-q` cũng tồn tại, nên script phải hiểu command semantics chứ không chỉ nghĩ non-zero = một loại lỗi duy nhất. :chatgpt-content-reference{index="15"}

Ví dụ:

```bash
grep 'CRITICAL' healthy.log
```

không thấy CRITICAL:

```text
exit = 1
```

Đó có thể chính xác là condition bạn mong muốn.

Ta sẽ khai thác sâu trong Module 2.

---

# 38. `awk` — không chỉ là “grep nâng cao”

Một mental model rất tốt cho `awk`:

```text
Input
  ↓
records
  ↓
fields
  ↓
pattern
  ↓
action
  ↓
formatted/aggregated output
```

GNU Awk documentation mô tả `awk` như một data-driven language dựa trên các cặp:

````text
pattern { action }
``` :chatgpt-content-reference{index="16"}


Nếu `grep` mạnh về:

```text
"giữ dòng nào?"
````

thì `awk` đặc biệt mạnh về:

```text
"dòng này có các cột gì, cột nào quan trọng,
tính toán hoặc trình bày chúng thế nào?"
```

---

# 39. Record và field

Theo default:

```text
1 input line ≈ 1 record
```

Record hiện tại:

```awk
$0
```

Các field:

```awk
$1
$2
$3
...
```

Số lượng field:

```awk
NF
```

GNU awk mặc định split fields theo whitespace; `$0` là whole record và `$NF` là field cuối. :chatgpt-content-reference{index="17"}

Ví dụ input:

```text
alice 42 active
bob 17 inactive
```

Command:

```bash
awk '{print $1}'
```

output:

```text
alice
bob
```

---

# 40. `$0` và `$1`

Ví dụ:

```bash
awk '{print $0}' file
```

in entire record.

```bash
awk '{print $1}' file
```

in first field.

```bash
awk '{print $1, $3}' file
```

in field 1 và field 3.

Đây là distinction cơ bản nhưng phải chắc.

---

# 41. `NF`

Ví dụ:

```bash
echo 'api01 200 healthy' |
awk '{print NF}'
```

Expected invariant:

```text
3
```

Field cuối:

```bash
echo 'api01 200 healthy' |
awk '{print $NF}'
```

Expected invariant:

```text
healthy
```

---

# 42. `NR`

`NR` là số record đã đọc tổng cộng.

Ví dụ:

```bash
awk '{print NR, $0}' file
```

có thể tạo:

```text
1 first line
2 second line
3 third line
```

Useful cho evidence hoặc parsing.

---

# 43. Pattern trong `awk`

Ví dụ:

```bash
awk '/ERROR/ {print $0}' app.log
```

Đọc như:

```text
Nếu record match ERROR
→ print toàn record
```

Rút gọn:

```bash
awk '/ERROR/'
```

vì default action của một matching pattern là print record.

---

# 44. Conditional numeric comparison

Đây là nơi `awk` vượt xa simple grep.

Ví dụ:

```text
api01 42
api02 97
api03 61
```

Command:

```bash
awk '$2 > 80 {print $1, $2}' data.txt
```

Output:

```text
api02 97
```

Operational use:

```text
lọc filesystem usage
lọc CPU values
lọc latency
lọc process metrics
```

---

# 45. `awk` và filesystem usage

Ví dụ:

```bash
df -P /
```

Có thể cho output dạng:

```text
Filesystem 1024-blocks Used Available Capacity Mounted on
/dev/sda2     20000000  ...      ...      82% /
```

Ta muốn lấy percentage.

Một cách:

```bash
df -P / |
awk 'NR==2 {print $5}'
```

Output:

```text
82%
```

Muốn bỏ `%`:

```bash
df -P / |
awk 'NR==2 {
    gsub(/%/, "", $5)
    print $5
}'
```

Rồi threshold:

```bash
df -P / |
awk 'NR==2 {
    gsub(/%/, "", $5)
    if ($5 >= 80)
        print "WARNING disk usage:", $5 "%"
}'
```

Đây chính là tư duy sẽ được chuyển thành health-check script trong Module 2.

---

# 46. Vì sao dùng `df -P` thay vì parse `df -h` mù quáng?

`df -h` dành cho human reading:

```text
4.7G
832M
```

Parsing/so sánh numeric từ human suffix dễ tạo complexity.

`df -P` còn giúp format predictable hơn theo portable output style.

Trong script health check, hãy ưu tiên nguồn dữ liệu dễ parse ổn định.

Đây là một nguyên tắc operations rất lớn:

> Human-readable output chưa chắc machine-readable tốt.

---

# 47. Field separator `-F`

Nếu input:

```text
alice:1001:dev
bob:1002:ops
```

Bạn có thể:

```bash
awk -F: '{print $1, $3}' file
```

Output:

```text
alice dev
bob ops
```

`-F` đặt input field separator.

GNU Awk cũng cung cấp variable `FS`; `-F` là cách command-line tiện dụng để set nó. :chatgpt-content-reference{index="18"}

---

# 48. `BEGIN` và `END`

Ví dụ:

```bash
awk '
BEGIN {
    print "Health report"
}
{
    count++
}
END {
    print "Records:", count
}' file
```

`BEGIN` chạy trước input processing.

`END` chạy sau khi input đã xử lý.

Đây là nền tảng hữu ích khi sau này tạo report.

---

# 49. Đếm errors bằng `awk`

Ví dụ:

```bash
journalctl -u myapp.service -b -o cat |
awk '
/ERROR/ {errors++}
END {print "ERROR count:", errors+0}
'
```

Giả sử có 3 entries:

```text
ERROR count: 3
```

`grep` có thể làm việc tương tự với:

```bash
grep -c ERROR
```

Điểm quan trọng không phải dùng tool “phức tạp nhất”.

Mà là:

```text
dùng tool đơn giản nhất phù hợp bài toán
```

---

# 50. `grep` hay `awk`?

Một guideline thực tế:

| Task                             | Tool thường phù hợp |
| -------------------------------- | ------------------- |
| Tìm dòng có text                 | `grep`              |
| Loại dòng                        | `grep -v`           |
| Match regex đơn giản             | `grep -E`           |
| Chọn column                      | `awk`               |
| Numeric condition                | `awk`               |
| Aggregation/count/sum            | `awk`               |
| Reformat fields                  | `awk`               |
| Search/replace text              | `sed`               |
| Chọn range hoặc transform stream | `sed`               |

Đừng viết:

```bash
cat file | grep ... | awk ... | sed ...
```

chỉ vì bạn biết cả bốn command.

Pipeline dài không tự động đồng nghĩa pipeline tốt.

---

# 51. `sed` — Stream Editor

`sed` là **stream editor**.

Mental model:

```text
input line
   ↓
sed program
   ↓
pattern space
   ↓
transformation
   ↓
output
```

GNU sed syntax tổng quát:

````bash
sed OPTIONS... [SCRIPT] [INPUTFILE...]
``` :chatgpt-content-reference{index="19"}


Use case operations thường gặp:

```text
replace text
normalize text
select lines
delete unwanted lines
transform stream
redact/sanitize output
````

---

# 52. Substitution với `s`

Syntax quan trọng nhất:

```text
s/regexp/replacement/flags
```

GNU sed documentation gọi `s` là substitution command và mô tả chính xác form này. :chatgpt-content-reference{index="20"}

Ví dụ:

```bash
echo 'status=FAILED' |
sed 's/FAILED/OK/'
```

Output:

```text
status=OK
```

---

# 53. Default chỉ replace occurrence đầu tiên

Ví dụ:

```bash
echo 'error error error' |
sed 's/error/ERROR/'
```

Output:

```text
ERROR error error
```

Muốn replace tất cả trong mỗi line:

```bash
echo 'error error error' |
sed 's/error/ERROR/g'
```

Output:

```text
ERROR ERROR ERROR
```

`g` nghĩa là global trên các matches trong pattern space hiện tại. :chatgpt-content-reference{index="21"}

---

# 54. Separator không nhất thiết là `/`

Ví dụ path:

```text
/var/log/myapp
```

Command này khá khó đọc:

```bash
sed 's/\/var\/log\/myapp/\/backup\/myapp/'
```

Bạn có thể dùng:

```bash
sed 's#/var/log/myapp#/backup/myapp#'
```

GNU sed cho phép dùng delimiter khác `/` miễn nhất quán trong `s` expression. :chatgpt-content-reference{index="22"}

Đây là practice rất hữu ích khi xử lý paths.

---

# 55. `sed -n` và `p`

Mặc định `sed` tự in pattern space sau mỗi processing cycle.

`-n` tắt automatic printing. :chatgpt-content-reference{index="23"}

Ví dụ:

```bash
sed -n '1,5p' app.log
```

in line 1 tới 5.

Tìm và print:

```bash
sed -n '/ERROR/p' app.log
```

Tương tự đơn giản với:

```bash
grep ERROR app.log
```

nên trong trường hợp đó `grep` thường dễ hiểu hơn.

---

# 56. `sed -E`

Extended regex:

```bash
sed -E '...'
```

GNU documentation lưu ý `-E` dùng extended regular expressions và hiện có portability tốt hơn historical `-r`. :chatgpt-content-reference{index="24"}

Ví dụ lấy request ID:

```bash
echo 'level=ERROR request_id=REQ123 status=500' |
sed -En 's/.*request_id=([^ ]+).*/\1/p'
```

Output:

```text
REQ123
```

Trong thực tế, `awk` đôi khi dễ đọc hơn tùy input format.

---

# 57. `sed -i`: phần nguy hiểm nhất của Module 1

Command:

```bash
sed -i 's/old/new/g' config.conf
```

thay đổi file thật.

GNU `sed -i` thực hiện in-place editing bằng cách tạo temporary output và thay thế original; nếu không cung cấp backup suffix thì original bị overwrite mà không giữ backup copy. :chatgpt-content-reference{index="25"}

**Cảnh báo vận hành:** Không dùng `sed -i` trực tiếp trên production configuration chỉ vì command nhìn ngắn.

Workflow đúng nên là:

```text
Inspect
    ↓
Backup
    ↓
Transform preview
    ↓
Diff
    ↓
Syntax validation
    ↓
Apply
    ↓
Validate service
    ↓
Rollback nếu cần
```

Ví dụ preview trước:

```bash
sed 's/old/new/g' config.conf
```

Không `-i`.

Nếu output đúng, backup:

```bash
cp -a config.conf config.conf.bak
```

Rồi:

```bash
sed -i 's/old/new/g' config.conf
```

Hoặc GNU sed backup suffix:

```bash
sed -i.bak 's/old/new/g' config.conf
```

Sau đó:

```bash
diff -u config.conf.bak config.conf
```

---

# 58. Một anti-pattern nguy hiểm với `sed`

Đừng tùy tiện:

```bash
sudo sed -i 's/foo/bar/g' /etc/*
```

Blast radius quá lớn.

Bạn có thể sửa:

```text
file không liên quan
binary/text đặc biệt
security configuration
service configuration
authentication files
```

Admin giỏi không chỉ biết “command có chạy không”.

Admin giỏi luôn hỏi:

```text
Nó sẽ thay đổi cái gì?
Bao nhiêu file?
Rollback thế nào?
Validation thế nào?
```

---

# 59. Pipeline: kết nối `journalctl`, `grep`, `awk`, `sed`

Ví dụ:

```bash
journalctl \
  -u myapp.service \
  -b \
  --since "20 minutes ago" \
  --no-pager |
grep -Ei 'error|warning|failed'
```

Mental model:

```text
journal
   ↓
scope = myapp.service
   ↓
scope = current boot
   ↓
scope = last 20 min
   ↓
render text
   ↓
grep relevant words
```

Điều quan trọng là filter có cấu trúc xảy ra **trước** text matching.

---

# 60. Pipeline tốt vs pipeline xấu

Pipeline này:

```bash
journalctl |
grep myapp |
grep ERROR
```

có nhiều vấn đề:

```text
không giới hạn boot
không giới hạn time
grep text thay vì unit metadata
có thể match unrelated lines
dataset ban đầu quá lớn
```

Tốt hơn:

```bash
journalctl \
  -u myapp.service \
  -b \
  --since "15 minutes ago" \
  --no-pager |
grep -i 'error'
```

Tư duy:

```text
narrow early, process later
```

---

# 61. Thêm `awk`

Giả sử application message dạng:

```text
request_id=R001 status=200 latency_ms=41
request_id=R002 status=503 latency_ms=1002
request_id=R003 status=500 latency_ms=701
```

Tìm status 5xx:

```bash
journalctl \
  -u myapp.service \
  -o cat \
  --since "10 minutes ago" |
awk '
{
    status=""
    for (i=1; i<=NF; i++) {
        if ($i ~ /^status=/) {
            split($i, a, "=")
            status=a[2]
        }
    }

    if (status >= 500 && status < 600)
        print
}'
```

Điểm đáng học ở đây là:

```text
record → fields → find field → extract value → numeric decision
```

Đó là kiểu logic health check mà Module 2 sẽ tiếp tục.

---

# 62. Thêm `sed` để normalize

Ví dụ application in:

```text
password=secret123
```

Bạn cần chia sẻ log ra ngoài team.

Có thể dùng:

```bash
sed -E 's/(password=)[^ ]+/\1REDACTED/g'
```

Nhưng hãy nhớ:

Redaction dựa trên regex có thể bỏ sót secret format khác.

Không nên tin rằng một `sed` expression đơn giản đã làm toàn bộ evidence an toàn.

Security review vẫn cần thiết.

---

# 63. `>` và `>>`

Đây là kiến thức hỗ trợ cho evidence.

```bash
command > file
```

overwrite output file.

```bash
command >> file
```

append.

Ví dụ:

```bash
journalctl \
  -u myapp.service \
  --since "10 minutes ago" \
  --no-pager \
  > evidence.log
```

**Cảnh báo:**

```bash
> evidence.log
```

xóa nội dung cũ trước khi ghi.

Nếu evidence cũ quan trọng, đừng overwrite nhầm.

---

# 64. stderr và `2>`

Conceptual file descriptors:

```text
0 = stdin
1 = stdout
2 = stderr
```

Ví dụ:

```bash
some-command > output.log 2> error.log
```

stdout:

```text
output.log
```

stderr:

```text
error.log
```

Cả hai:

```bash
some-command > all.log 2>&1
```

Phần này sẽ quan trọng hơn trong Bash Module 2.

---

# 65. `tee`

Nếu muốn vừa nhìn output vừa lưu:

```bash
journalctl \
  -u myapp.service \
  --since "10 minutes ago" \
  --no-pager |
tee evidence.log
```

Flow:

```text
command
  ↓
 tee ─────→ terminal
  │
  └───────→ evidence.log
```

Đây là một pattern thực tế rất hữu ích.

---

# 66. Evidence khác filtered output

Giả sử bạn lấy:

```bash
journalctl \
  -u app.service \
  --since "14:30" \
  --until "14:40" \
  -o short-iso \
  --no-pager \
  > raw-evidence.log
```

Sau đó:

```bash
grep -Ei 'error|failed|timeout' raw-evidence.log \
  > relevant.log
```

Tôi khuyến khích giữ cả:

```text
raw-evidence.log
relevant.log
```

Vì filtered evidence có thể loại mất context.

Mental model:

```text
Raw bounded evidence
       │
       ├── preserved
       │
       └── filtered working copy
```

---

# 67. Một investigation workflow chuẩn

Giả sử ticket:

```text
"Java application stopped responding around 14:32."
```

Không chạy ngay:

```bash
grep error ...
```

Ta làm lần lượt.

### Step A — xác định scope

```bash
hostname
date
```

Sau đó:

```bash
systemctl status myapp.service --no-pager -l
```

Bạn cần biết:

```text
đúng host?
đúng time?
unit nào?
current state?
```

### Step B — current boot

```bash
journalctl -u myapp.service -b -n 100 --no-pager
```

### Step C — time window

```bash
journalctl \
  -u myapp.service \
  -b \
  --since "2026-10-01 14:25:00" \
  --until "2026-10-01 14:40:00" \
  -o short-iso \
  --no-pager
```

### Step D — preserve evidence

```bash
journalctl \
  -u myapp.service \
  -b \
  --since "2026-10-01 14:25:00" \
  --until "2026-10-01 14:40:00" \
  -o short-iso \
  --no-pager \
  > myapp-incident.log
```

### Step E — search indicators

```bash
grep -Ein \
  'error|exception|failed|timeout|refused|denied|fatal' \
  myapp-incident.log
```

### Step F — context

Giả sử relevant line 73:

```bash
grep -n -C 5 'Connection refused' myapp-incident.log
```

### Step G — hypothesis

Ví dụ:

```text
14:31:52 DB connection refused
14:31:54 retry
14:31:57 retry
14:32:01 startup failed
```

Hypothesis:

```text
Application failure may be downstream of database connectivity failure.
```

Notice wording:

**may be**.

Đừng biến correlation thành certainty trước khi verify.

---

# 68. Evidence-driven troubleshooting

Bạn phải phân biệt bốn khái niệm:

| Khái niệm   | Ví dụ                                                  |
| ----------- | ------------------------------------------------------ |
| Symptom     | App unavailable                                        |
| Evidence    | `Connection refused` at 14:31:52                       |
| Hypothesis  | DB endpoint unavailable                                |
| Root cause  | Sau validation xác định PostgreSQL process was stopped |
| Remediation | Restore DB service theo runbook                        |

Một log line tự nó thường chưa phải root cause.

---

# 69. “ERROR” không phải lúc nào cũng failure thật

Ví dụ application có thể log:

```text
ERROR optional metrics exporter unavailable
```

nhưng main application vẫn phục vụ bình thường.

Ngược lại một line:

```text
INFO JVM exiting
```

có thể rất quan trọng.

Do đó severity phải kết hợp với:

```text
expected behavior
service state
endpoint behavior
timeline
impact
```

---

# 70. “No log” cũng là evidence

Ví dụ:

```bash
journalctl -u myapp.service \
  --since "14:30" \
  --until "14:40"
```

không có bất kỳ entry nào.

Đừng nghĩ:

> “Không có log nên không biết gì.”

Nó tạo thêm hypotheses:

```text
service chưa chạy?
sai unit?
sai host?
sai time/timezone?
journal không persistent?
permission không đủ?
application log sang file khác?
logging misconfigured?
service chưa emit stdout/stderr?
```

Absence of evidence không tự động chứng minh một hypothesis, nhưng nó giúp định hướng investigation.

---

# 71. Timezone cực kỳ quan trọng

Ticket có thể ghi:

```text
14:32 UTC
```

server dùng:

```text
UTC+7
```

Nếu bạn search 14:30 local:

```bash
journalctl --since "14:30"
```

bạn có thể nhìn sai window hoàn toàn.

Kiểm tra:

```bash
date
timedatectl
```

Trong escalation evidence, nên ghi rõ timestamp/timezone khi ambiguity có thể xảy ra.

---

# 72. `journalctl -x`: dùng cẩn thận

Bạn thường thấy trên Internet:

```bash
journalctl -xe
```

hoặc:

```bash
journalctl -xeu myapp.service
```

`-x` có thể bổ sung explanatory catalog text cho một số messages.

Nó hữu ích cho interactive troubleshooting.

Nhưng official systemd documentation còn lưu ý không nên dùng `-x` khi attach journal output vào bug report, vì phần explanatory text không phải raw journal event gốc. :chatgpt-content-reference{index="26"}

Do đó:

```text
interactive help → có thể dùng -x

preserved evidence → ưu tiên output không thêm explanation
```

Đây là distinction rất chuyên nghiệp.

---

# 73. `journalctl -k`

Kernel messages:

```bash
journalctl -k
```

Official systemd docs cho biết `-k/--dmesg` filter kernel transport và implied current boot. :chatgpt-content-reference{index="27"}

Useful khi Day 5 issue liên quan:

```text
disk
filesystem
NIC
OOM
device
kernel
```

Ví dụ:

```bash
journalctl -k -p warning
```

Không phải mọi app failure đều nằm ở application log.

Integrated troubleshooting cần biết khi nào phải xuống layer kernel.

---

# 74. Một ví dụ cross-layer

Symptom:

```text
Java app stopped.
```

App log:

```text
Cannot write data
```

Nếu chỉ search app:

```bash
journalctl -u myapp.service
```

bạn có thể dừng ở:

```text
application write failure
```

Nhưng kernel/system logs có thể cho thấy:

```text
filesystem I/O error
```

hoặc:

```text
No space left on device
```

Đó là lý do troubleshooting phải đi theo layers.

---

# 75. Journal retention

Journal không phải kho lưu trữ vô hạn.

`journald` có controls liên quan disk usage/retention, và journal files có thể rotate/vacuum. Official `journald.conf` docs mô tả các controls như `SystemMaxUse`, `RuntimeMaxUse`, `SystemKeepFree` và `RuntimeKeepFree`. :chatgpt-content-reference{index="28"}

Điều này có operational implication:

> “Không tìm thấy log tháng trước” không nhất thiết nghĩa event không xảy ra.

Có thể data đã rotate khỏi retention window.

---

# 76. Đừng thay đổi journal retention chỉ để làm bài lab

Các thay đổi:

```text
/etc/systemd/journald.conf
journal vacuum
journal storage mode
```

có thể ảnh hưởng forensic evidence hoặc disk usage.

Trong lab Module 1 hiện tại, **không cần thay đổi journald configuration**.

Chúng ta chủ yếu read/query.

Đó là cách giới hạn blast radius.

---

# 77. Security: logs có thể chứa secrets

Log có thể vô tình chứa:

```text
password
access token
API key
session cookie
Authorization header
database connection string
personal data
internal IP/hostnames
customer identifiers
```

Vì vậy command kiểu:

```bash
journalctl > incident.log
```

không có nghĩa file đó được phép gửi cho bất kỳ ai.

Evidence handling phải có:

```text
need-to-know
least privilege
secure storage
redaction where appropriate
approved sharing path
```

Đặc biệt không paste toàn bộ production journal lên public forum.

---

# 78. Security: tránh log injection confusion

Nếu application nhận user input và log trực tiếp:

```text
username=<user supplied text>
```

attacker hoặc malformed input có thể tạo text trông giống:

```text
ERROR
SUCCESS
admin
```

Nên khi điều tra, structured metadata và trusted source context đáng tin hơn việc chỉ grep keyword trong message.

---

# 79. Anti-pattern: `sudo journalctl | grep...` cho mọi việc

`sudo` có thể cần để đọc privileged logs, nhưng đừng biến nó thành phản xạ.

Trước:

```bash
sudo journalctl ...
```

hãy thử:

```bash
journalctl ...
```

Nếu permission không đủ, mới dùng access được approve.

Đây là application trực tiếp của least privilege Day 3.

---

# 80. Anti-pattern: restart trước khi thu evidence

Incident:

```text
service failed
```

Fresher:

```bash
sudo systemctl restart app
```

Sau đó mới xem log.

Có thể service recovery, nhưng bạn đã:

```text
thay state
tạo thêm log
có thể che symptom
mất transient state
```

Better:

```text
inspect
capture evidence
then remediate
```

trừ trường hợp runbook/availability requirement bắt buộc immediate recovery.

---

# 81. Anti-pattern: chỉ search keyword `ERROR`

Ví dụ root cause:

```text
Bind failed: Address already in use
```

không nhất thiết có chữ:

```text
ERROR
```

Hoặc:

```text
Permission denied
No space left on device
Connection refused
Killed process
Out of memory
```

Vì vậy troubleshooting phải dựa vào domain knowledge, không phải một keyword duy nhất.

---

# 82. Anti-pattern: pipeline càng dài càng “pro”

Ví dụ:

```bash
cat file |
grep X |
awk '{print $0}' |
sed 's/foo/foo/' |
grep Y
```

Có thể technically chạy nhưng rất khó maintain.

Đừng dùng tool nếu nó không tạo value.

Ví dụ:

```bash
grep X file
```

tốt hơn:

```bash
cat file | grep X
```

trừ khi `cat` có mục tiêu cụ thể.

---

# 83. Anti-pattern: parse human output mà không kiểm tra assumptions

Ví dụ:

```bash
df -h |
awk '{print $5}'
```

Bạn giả định:

```text
mọi line đều filesystem data
column layout không đổi
header đã xử lý
mount path không gây complication
```

Trước khi scripting phải inspect actual input.

Operations automation luôn phải biết:

```text
input contract là gì?
```

---

# 84. Anti-pattern: sửa production config bằng `sed -i` ngay lập tức

Sai:

```bash
sudo sed -i 's/8080/9090/' /etc/myapp/app.conf
sudo systemctl restart myapp
```

Chưa:

```text
backup
diff
validate syntax
review scope
understand duplicate 8080
prepare rollback
```

`sed -i` là tool, không phải change-management process.

---

# 85. L1 troubleshooting boundary

Trong Day 6, bạn phải bắt đầu phát triển boundary:

```text
L1 safe investigation
vs
state-changing remediation
vs
escalation
```

Những hành động thường tương đối an toàn:

```text
systemctl status
journalctl
grep
awk
sed without -i
ss
df
free
ps
read-only inspection
```

Những hành động cần cẩn trọng/approval hơn:

```text
restart
kill
modify config
chmod/chown
delete files
vacuum journal
change firewall
unmount
format filesystem
sed -i on production files
```

L1 không phải:

> Tôi biết command nên tôi được phép chạy.

L1 phải là:

> Tôi hiểu scope, runbook, approval và blast radius.

---

# 86. Khi nào nên escalation?

Ví dụ bạn thu được:

```text
14:31:17 application DB timeout
14:31:20 retry
14:31:25 retry
14:31:30 startup failed
```

Bạn kiểm tra trong allowed scope và thấy:

```text
DB host là remote production database
không thuộc quyền quản trị của bạn
```

Đừng cố:

```text
SSH vào DB
restart DB
modify network
```

nếu đó ngoài scope.

Thay vào đó escalation evidence nên thể hiện:

```text
host
unit
incident time
impact
exact error
time window
commands đã chạy
findings
tests đã thực hiện
current state
reason escalation required
```

---

# 87. Guided Lab — Module 1

Lab này được thiết kế **an toàn**, không cần phá service production.

Chúng ta dùng `logger` tạo synthetic log records để thực hành journal analysis.

`logger` gửi message vào system logging infrastructure. Trên systemd distro, các messages thường đi vào journal.

Không có destructive operation trong phần chính của lab.

---

## Phase A — kiểm tra environment

Chạy:

```bash
hostname
date
timedatectl
systemctl --version
journalctl --version
grep --version
sed --version
```

Với `awk`:

```bash
awk -W version 2>&1 | head
```

Hoặc distro khác:

```bash
awk --version 2>&1 | head
```

### Expected invariant

Bạn phải xác định được:

```text
hostname
current timestamp
timezone
systemd availability/version
grep/sed/awk implementation
```

Không cần output giống máy tôi.

---

# 88. Phase B — tạo synthetic events

Đặt một tag dễ tìm:

```bash
LAB_TAG="day6-m1-$USER"
echo "$LAB_TAG"
```

Tạo events:

```bash
logger -p user.info \
  -t "$LAB_TAG" \
  'component=api request_id=R001 status=200 latency_ms=42 message=request_ok'
```

```bash
logger -p user.warning \
  -t "$LAB_TAG" \
  'component=db request_id=R002 status=503 latency_ms=1200 message=connection_slow'
```

```bash
logger -p user.err \
  -t "$LAB_TAG" \
  'component=db request_id=R003 status=500 latency_ms=3000 message=connection_refused'
```

```bash
logger -p user.info \
  -t "$LAB_TAG" \
  'component=api request_id=R004 status=200 latency_ms=39 message=recovered'
```

Lưu ý:

Đây là **simulated log**, không phải real incident.

---

# 89. Phase C — truy vấn bằng identifier

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "5 minutes ago" \
  --no-pager
```

`-t` tương đương filter theo syslog identifier, được systemd hỗ trợ trực tiếp. :chatgpt-content-reference{index="29"}

### Expected invariant

Bạn thấy các message vừa tạo.

Nếu không thấy, kiểm tra:

```bash
logger -t "$LAB_TAG" "test"
journalctl --since "2 minutes ago" --no-pager |
grep "$LAB_TAG"
```

Nếu normal user không đọc được system journal, bạn có thể cần quyền phù hợp theo lab environment.

---

# 90. Phase D — timestamp rõ ràng

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "5 minutes ago" \
  -o short-iso \
  --no-pager
```

Quan sát:

```text
timestamp
hostname
identifier
message
```

Đừng chỉ nhìn message body.

---

# 91. Phase E — lọc severity native bằng journalctl

Error và mức nghiêm trọng hơn:

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "5 minutes ago" \
  -p err \
  -o short-iso \
  --no-pager
```

### Expected invariant

Record `connection_refused` phải xuất hiện.

Warning không nên xuất hiện với `-p err`.

---

# 92. Phase F — dùng `grep`

Lấy toàn bộ synthetic messages:

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "5 minutes ago" \
  -o cat \
  --no-pager |
grep -E 'status=5[0-9][0-9]'
```

### Expected invariant

Bạn tìm được các 5xx events:

```text
status=503
status=500
```

Không được phụ thuộc chính xác vào order nếu môi trường logging có khác biệt, nhưng dataset lab này thông thường sẽ giữ chronological order.

---

# 93. Phase G — tìm error indicators

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "5 minutes ago" \
  -o cat \
  --no-pager |
grep -Ei 'slow|refused|failed|error'
```

Bạn phải hiểu:

Đây là **text filtering**, không phải priority filtering.

Record warning `connection_slow` và err `connection_refused` đều có thể match.

---

# 94. Phase H — dùng `awk` lấy request IDs

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "5 minutes ago" \
  -o cat \
  --no-pager |
awk '
{
    for (i=1; i<=NF; i++) {
        if ($i ~ /^request_id=/)
            print $i
    }
}'
```

Expected invariant:

```text
request_id=R001
request_id=R002
request_id=R003
request_id=R004
```

---

# 95. Phase I — lấy status bằng `awk`

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "5 minutes ago" \
  -o cat \
  --no-pager |
awk '
{
    for (i=1; i<=NF; i++) {
        if ($i ~ /^status=/)
            print $i
    }
}'
```

Bạn đang biến unstructured-looking message thành field processing.

---

# 96. Phase J — threshold với `awk`

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "5 minutes ago" \
  -o cat \
  --no-pager |
awk '
{
    status=""
    for (i=1; i<=NF; i++) {
        if ($i ~ /^status=/) {
            split($i, a, "=")
            status=a[2]
        }
    }

    if (status >= 500)
        print
}'
```

Expected relevant lines:

```text
R002 status=503
R003 status=500
```

Đừng học thuộc đoạn awk.

Hãy hiểu logic:

```text
read record
→ scan fields
→ find status=
→ split key/value
→ numeric compare
→ print matching record
```

---

# 97. Phase K — `sed`

Lấy request IDs:

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "5 minutes ago" \
  -o cat \
  --no-pager |
sed -En 's/.*request_id=([^ ]+).*/\1/p'
```

Expected:

```text
R001
R002
R003
R004
```

Bạn vừa giải cùng một vấn đề bằng cách khác.

Câu hỏi quan trọng:

> `awk` hay `sed` dễ maintain hơn cho format này?

Ở đây tôi thường ưu tiên `awk` nếu tiếp tục cần nhiều key/value và numeric logic.

---

# 98. Phase L — preserve raw evidence

Tạo workspace:

```bash
mkdir -p "$HOME/day6-module1"
```

Lưu evidence:

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "10 minutes ago" \
  -o short-iso \
  --no-pager \
  > "$HOME/day6-module1/raw-journal.log"
```

Kiểm tra:

```bash
wc -l "$HOME/day6-module1/raw-journal.log"
```

```bash
sed -n '1,20p' "$HOME/day6-module1/raw-journal.log"
```

---

# 99. Phase M — tạo filtered working copy

```bash
grep -Ei \
  'status=5[0-9][0-9]|slow|refused' \
  "$HOME/day6-module1/raw-journal.log" \
  > "$HOME/day6-module1/relevant.log"
```

Kiểm tra:

```bash
cat "$HOME/day6-module1/relevant.log"
```

Bây giờ có:

```text
raw-journal.log
relevant.log
```

Đó là cách làm tốt hơn chỉ lưu filtered lines.

---

# 100. Phase N — validation

Bạn phải chứng minh được:

```bash
grep 'R001' "$HOME/day6-module1/raw-journal.log"
```

```bash
grep 'R003' "$HOME/day6-module1/raw-journal.log"
```

```bash
grep 'connection_refused' \
  "$HOME/day6-module1/relevant.log"
```

Và:

```bash
journalctl \
  -t "$LAB_TAG" \
  --since "10 minutes ago" \
  -p err \
  --no-pager
```

Nếu các evidence không phù hợp với expectation, investigation lab chưa hoàn thành.

---

# 101. Cleanup của lab

Các file evidence do bạn tạo có thể xóa sau khi đã học xong:

```bash
rm -r "$HOME/day6-module1"
```

**Trước khi chạy `rm -r`, kiểm tra path:**

```bash
ls -ld "$HOME/day6-module1"
```

Synthetic entries đã vào journal thì **không cần và không nên cố xóa riêng** chỉ để cleanup lab.

Journal sẽ được quản lý theo retention policy của hệ thống.

Đừng dùng journal vacuum chỉ để xóa vài dòng lab.

---

# 102. Lab thực tế hơn: investigation một systemd service

Sau synthetic lab, chọn một service an toàn đã tồn tại trên máy.

Ví dụ có thể là:

```text
ssh.service
sshd.service
cron.service
crond.service
```

Tên phụ thuộc distro.

Trước tiên:

```bash
systemctl list-units --type=service --state=running
```

Chọn `<unit>`.

Giả sử:

```text
<unit>=ssh.service
```

Investigation:

```bash
systemctl status ssh.service --no-pager -l
```

Sau đó:

```bash
journalctl \
  -u ssh.service \
  -b \
  -n 50 \
  -o short-iso \
  --no-pager
```

Time filter:

```bash
journalctl \
  -u ssh.service \
  -b \
  --since "30 minutes ago" \
  -o short-iso \
  --no-pager
```

Search:

```bash
journalctl \
  -u ssh.service \
  -b \
  --since "30 minutes ago" \
  --no-pager |
grep -Ei 'error|failed|denied|invalid|disconnect'
```

**Không inject failure vào SSH đang dùng để remote vào máy.**

Đó là một operational safety rule rất quan trọng.

---

# 103. Failure investigation example

Giả sử bạn thấy:

```text
Oct 01 14:31:04 app01 myapp[1220]: Connecting database
Oct 01 14:31:05 app01 myapp[1220]: Connection refused
Oct 01 14:31:10 app01 myapp[1220]: Retrying connection
Oct 01 14:31:20 app01 myapp[1220]: Connection refused
Oct 01 14:31:21 app01 systemd[1]: myapp.service: Main process exited
Oct 01 14:31:21 app01 systemd[1]: myapp.service: Failed with result 'exit-code'
```

Fresher conclusion:

```text
Root cause = systemd service failed.
```

Sai.

`Failed with result 'exit-code'` mô tả outcome ở service layer.

Earlier events cho thấy:

```text
database connection refused
```

nhưng ta vẫn chưa chứng minh root cause của connection refusal.

Có thể là:

```text
database process down
wrong host
wrong port
firewall
routing
service listening only localhost
database startup still in progress
```

Module 1 dạy bạn không overclaim từ log.

---

# 104. Correlation thay vì single-line diagnosis

Giả sử:

```text
14:31:00 config loaded
14:31:01 database endpoint=db01:5432
14:31:03 connection refused
14:31:05 retry
14:31:10 connection refused
14:31:11 process exiting
14:31:11 systemd reports service failed
```

Timeline cho phép hypothesis:

```text
app exits because database connectivity could not be established
```

Nhưng root cause của DB connectivity vẫn cần Day 5 network evidence và sau này Day 15–16 PostgreSQL evidence.

Đây là integrated thinking mà khóa học hướng tới.

---

# 105. Khi nào dùng `grep`, `awk`, `sed` sau `journalctl`?

Hãy hình dung ba cấp:

```text
Level 1 — Query source correctly
journalctl

Level 2 — Select interesting records
grep

Level 3 — Extract/transform values
awk / sed
```

Ví dụ:

```bash
journalctl \
  -u myapp.service \
  -b \
  --since "30 minutes ago" \
  -o cat \
  --no-pager |
grep 'latency_ms=' |
awk '
{
    for(i=1;i<=NF;i++)
        if ($i ~ /^latency_ms=/) {
            split($i,a,"=")
            print a[2]
        }
}'
```

Flow:

```text
correct service/time
        ↓
messages containing latency
        ↓
extract numeric latency
```

---

# 106. Tool composition và maintainability

Đừng cố làm mọi thứ bằng một regex khổng lồ.

Ví dụ một command ngắn không đồng nghĩa với code tốt.

Operational scripts nên ưu tiên:

```text
readable
predictable
testable
safe
easy to troubleshoot
```

Nếu một pipeline chỉ mình bạn hiểu, nó không phải operations-quality automation.

Ngày 18 bạn còn phải làm handover/change evidence; readability sẽ càng quan trọng.

---

# 107. Supplementary — `journalctl -o json`

Journal bản chất structured, nên có thể:

```bash
journalctl -u myapp.service -o json
```

Một record có thể chứa structured fields.

Nếu môi trường có `jq`, structured parsing bằng JSON có thể robust hơn việc grep rendered human text.

Ví dụ conceptual:

```bash
journalctl -u myapp.service -o json |
jq '.MESSAGE'
```

Nhưng:

**`jq` không nằm trong explicit Day 6 syllabus**, nên đây là Supplementary. Bạn vẫn phải thành thạo `grep/awk/sed`.

---

# 108. Supplementary — environment/locale ảnh hưởng text tools

`grep`, `awk` và regex có thể chịu ảnh hưởng bởi locale, đặc biệt character classes/sorting/case interpretation.

GNU grep documentation ghi rõ các environment variables như:

```text
LC_ALL
LC_COLLATE
LC_CTYPE
LANG
```

có thể ảnh hưởng matching/character interpretation. :chatgpt-content-reference{index="30"}

Trong automation cần deterministic text behavior, đôi khi bạn sẽ thấy:

```bash
LC_ALL=C
```

nhưng không nên thêm vào mọi script một cách mù quáng vì nó cũng thay đổi locale/encoding behavior.

Ta sẽ nói kỹ hơn ở Module 2.

---

# 109. Supplementary — integrity của evidence

Trong investigation quan trọng, một practice tốt là lưu evidence rồi tạo hash:

```bash
sha256sum incident.log
```

Ví dụ:

```text
incident.log
incident.log.sha256
```

Điều này có thể giúp chứng minh file evidence không bị vô tình thay đổi sau lúc capture.

Đây **không phải requirement rõ ràng của Day 6 syllabus**, nhưng là practice tốt trong môi trường cần audit/forensics/change evidence.

---

# 110. Phương pháp troubleshooting chuẩn của Module 1

Đến cuối Module 1, quy trình bạn nên internalize là:

```text
1. Confirm symptom
        ↓
2. Confirm host + clock + timezone
        ↓
3. Identify affected unit/process
        ↓
4. Establish boot
        ↓
5. Establish time window
        ↓
6. Capture bounded raw journal
        ↓
7. Filter relevant records
        ↓
8. Extract useful fields
        ↓
9. Build timeline
        ↓
10. Form hypothesis
        ↓
11. Cross-check with other evidence
        ↓
12. Root cause OR escalate
        ↓
13. Preserve evidence
```

Đừng đảo thành:

```text
guess
→ change something
→ restart
→ see if it works
```

Đó là trial-and-error administration.

---

# 111. Exam Focus — những gì bạn cần trả lời được

| Chủ đề               | Bạn cần hiểu                                   |
| -------------------- | ---------------------------------------------- |
| journald             | Collector/storage role                         |
| journalctl           | Query journal                                  |
| `-u`                 | Filter systemd unit                            |
| `-b`                 | Current/specific boot                          |
| `-b -1`              | Previous boot                                  |
| `--since/--until`    | Time window                                    |
| `-n`                 | Recent entries                                 |
| `-f`                 | Follow new events                              |
| `-p`                 | Severity filtering                             |
| `-o`                 | Output formatting                              |
| `--no-pager`         | Non-interactive output                         |
| `grep`               | Select lines by pattern                        |
| `grep -i`            | Case insensitive                               |
| `grep -E`            | Extended regex                                 |
| `grep -F`            | Fixed string                                   |
| `grep -v`            | Invert match                                   |
| `grep -C`            | Context                                        |
| `awk`                | Record/field processing                        |
| `$0`, `$1`, `$NF`    | Record/fields                                  |
| `NR`, `NF`           | Record count/field count                       |
| `-F`                 | Field separator                                |
| `pattern { action }` | Core awk model                                 |
| `sed`                | Stream transformation                          |
| `s///`               | Substitute                                     |
| `g`                  | All matches per pattern space                  |
| `-n` + `p`           | Explicit selection/printing                    |
| `-E`                 | Extended regex                                 |
| `-i`                 | In-place change — risky                        |
| Pipeline             | stdout → stdin                                 |
| Evidence             | Raw bounded source before aggressive filtering |

---

# 112. Knowledge Check

Trước khi xem câu trả lời, hãy tự giải thích các tình huống sau:

| Câu hỏi                                                                                 | Điều bạn phải reasoning được                       |
| --------------------------------------------------------------------------------------- | -------------------------------------------------- |
| Tại sao `journalctl \| grep app` kém hơn `journalctl -u app.service`?                   | Text match vs structured unit filter               |
| Vì sao nên dùng `-b` khi xử lý incident hiện tại?                                       | Tránh log boot cũ                                  |
| `journalctl -p err` có chỉ exact `err` không?                                           | Không; gồm mức nghiêm trọng hơn                    |
| Tại sao `grep` trả exit 1 chưa chắc là command error?                                   | No selected lines                                  |
| `$0` trong awk là gì?                                                                   | Entire record                                      |
| `$NF` là gì?                                                                            | Last field                                         |
| Khi nào dùng `awk` thay grep?                                                           | Field/numeric/aggregation logic                    |
| `sed -i` nguy hiểm ở điểm nào?                                                          | Mutates original                                   |
| Vì sao không lưu chỉ output của `grep ERROR`?                                           | Mất surrounding/contextual evidence                |
| Vì sao root cause không nhất thiết là dòng có severity cao nhất?                        | Error có thể chỉ là downstream symptom             |
| Nếu `journalctl -b -1` không có log thì có kết luận previous boot không lỗi được không? | Không; persistence/retention có thể không giữ data |
| Vì sao phải kiểm tra timezone?                                                          | Incident timestamp có thể nằm ở timezone khác      |

---

# 113. Independent Practical Challenge

Không cần sửa bất kỳ service nào.

Chọn một running systemd service trên lab Linux rồi tự tạo evidence package có cấu trúc:

```text
Host identity
Current date/time/timezone
Service current state
Current-boot logs
Last-30-minute logs
Warnings/errors if any
Filtered interesting events
At least one awk extraction
At least one sed transformation
Your timeline interpretation
Whether evidence suggests healthy / suspicious / insufficient information
What additional evidence you would collect next
```

Quan trọng:

Không được invent root cause nếu log không chứng minh nó.

Ví dụ conclusion đúng có thể là:

```text
"No failure observed in the selected 30-minute window.
Service is currently active.
Available evidence is insufficient to infer historical incidents
outside this retention/time window."
```

Đó tốt hơn rất nhiều so với bịa:

```text
"Service has no problems."
```

---

# 114. Mastery Check Module 1

Tôi chỉ coi bạn **nắm Module 1 về mặt kiến thức** khi bạn có thể nhìn command:

```bash
journalctl \
  -u app.service \
  -b \
  --since "20 minutes ago" \
  -o short-iso \
  --no-pager |
grep -Ei -C 3 'failed|timeout|refused'
```

và giải thích từng phần:

```text
nguồn dữ liệu nào
scope unit nào
boot nào
time range nào
format nào
vì sao no-pager
pipe làm gì
grep đang dùng regex mode gì
case sensitivity thế nào
context bao nhiêu
những information nào đã bị loại
command này có thay đổi system state không
output có đủ làm raw evidence hay chỉ là filtered evidence
```

Và khi thấy:

```bash
sed -i ...
```

phản xạ của bạn phải là:

```text
"Dừng lại: file nào sẽ bị thay đổi?
Đã inspect chưa?
Backup đâu?
Diff đâu?
Validation đâu?
Rollback đâu?"
```

Đó chính là tư duy administration/operations mà Day 6 muốn hình thành.

---

# 115. Mental model cuối cùng cần nhớ

Nếu chỉ nhớ một sơ đồ của toàn Module 1, hãy nhớ:

```text
                    INCIDENT
                        │
                        ▼
            ┌─────────────────────┐
            │ Scope the problem   │
            │ host / time / unit  │
            │ boot / impact       │
            └──────────┬──────────┘
                       │
                       ▼
            ┌─────────────────────┐
            │     journalctl      │
            │ retrieve evidence   │
            │ with correct scope  │
            └──────────┬──────────┘
                       │
                       ▼
              bounded raw logs
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
           grep       awk       sed
         select      parse     transform
             │         │         │
             └─────────┼─────────┘
                       ▼
                  TIMELINE
                       │
                       ▼
                  HYPOTHESIS
                       │
                       ▼
              SAFE VERIFICATION
                  /           \
                 /             \
                ▼               ▼
          ROOT CAUSE        INSUFFICIENT /
          + REMEDY          OUT OF SCOPE
                                │
                                ▼
                           ESCALATION
                           + EVIDENCE
```

Module 1 vì vậy không phải bốn command rời rạc. Nó là nền tảng của **evidence-driven Linux troubleshooting**.

Tài liệu systemd hiện hành xác nhận `journalctl` hỗ trợ structured filtering theo unit, boot, time, priority, identifier và nhiều output formats; GNU documentation hiện hành cũng xác nhận các semantics quan trọng của `grep`, `awk` và `sed` mà chúng ta vừa dùng. :chatgpt-content-reference{index="31"}

**Bước thực hành hợp lý tiếp theo của Day 6 là bạn tự chạy Guided Lab Module 1 ở trên và gửi cho tôi command + output/evidence. Khi đó tôi có thể review theo đúng PRACTICE workflow: kiểm tra command, phát hiện assumption sai, yêu cầu evidence còn thiếu và đánh giá bạn đã đủ readiness để sang Module 2 hay chưa.**
