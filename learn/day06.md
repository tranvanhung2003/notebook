# Day 6 — Linux Operations Consolidation: Logs, Text Processing, Bash, Exit Codes, Cron & Backup

## 1. Syllabus Alignment

Theo **MASTER SYLLABUS**, Day 6 có đúng hai yêu cầu bắt buộc:

- **Concept/Lecture:** `journalctl`, `grep/awk/sed`, Bash scripting, exit codes, cron, backup bằng `tar/rsync`, và tổng hợp kiến thức Linux.
- **Assignment/Lab:** xây dựng **health-check + backup script**, schedule nó, **inject một failure**, sau đó ghi lại **escalation evidence**. 00_MASTER_SYLLABUS_Java_Fresher…

Đây là ngày kết thúc phần **Linux Fundamentals** trước khi Day 7 chuyển sang AWS. Vì vậy mục tiêu hôm nay không phải biết thêm vài command riêng lẻ, mà là ghép Day 2–5 thành một workflow vận hành:

```text
System / Service
      │
      ▼
 logs + status + metrics
      │
      ▼
journalctl / grep / awk / sed
      │
      ▼
 Bash script
      │
      ├── health checks
      ├── backup
      ├── logging
      └── exit code
      │
      ▼
 cron scheduling
      │
      ▼
 failure detected
      │
      ▼
 evidence → remediation / escalation
```

### Mandatory hôm nay

`journalctl` → `grep/awk/sed` → Bash → exit code → cron → `tar/rsync` → integrated troubleshooting → failure injection → evidence.

### Supplementary / Beyond the explicit syllabus

Tôi sẽ bổ sung một số kiến thức cần thiết trong thực tế như `pipefail`, `trap`, atomic-ish backup bằng `.partial`, backup verification, restore test, cron environment, `umask`, `ShellCheck`, RPO/RTO và systemd timer awareness. Những phần này hỗ trợ vận hành nhưng không thay thế nội dung syllabus.

---

# 2. Objectives và prerequisites

Sau Day 6, bạn cần tự làm được một chu trình như sau:

```text
Có alert / nghi ngờ hệ thống bất thường
        ↓
Xác định thời điểm + phạm vi
        ↓
systemctl / journalctl / df / ss / ps
        ↓
lọc bằng grep / awk / sed
        ↓
xác định condition
        ↓
script hóa check
        ↓
trả exit code chính xác
        ↓
schedule bằng cron
        ↓
backup dữ liệu/config
        ↓
verify backup
        ↓
inject failure
        ↓
thu thập evidence
        ↓
fix trong L1 scope hoặc escalate
```

Bạn nên đã nắm Day 2–5: navigation, file permission, `sudo`, process, `systemctl`, package, networking, port/socket, filesystem, `df`, `du`, mount.

### Environment giả định

Các ví dụ chính dùng:

```text
Linux có systemd
Bash
GNU coreutils
grep / awk / sed
tar
rsync
cron hoặc crond
```

Tên service có thể khác nhau:

| Distro family          | Cron package/service thường gặp |
| ---------------------- | ------------------------------- |
| Ubuntu/Debian          | `cron`                          |
| RHEL/Rocky/Alma/Fedora | `crond`, package `cronie`       |

Không được mặc định rằng production của bạn giống lab. Luôn kiểm tra:

```bash
cat /etc/os-release
bash --version
systemctl --version
grep --version
awk --version 2>/dev/null || awk -W version
sed --version
tar --version
rsync --version
```

---

# 3. Mental Model quan trọng nhất của Day 6

Một Linux administrator không chỉ chạy:

```bash
journalctl
```

rồi đọc hàng nghìn dòng.

Bạn cần biến dữ liệu thô thành thông tin:

```text
raw logs
  ↓
scope theo service/time
  ↓
filter pattern
  ↓
extract field
  ↓
aggregate
  ↓
decide
```

Ví dụ:

```bash
journalctl -u ssh --since "1 hour ago" --no-pager \
  | grep "Failed password" \
  | awk '{print $(NF-3)}'
```

Ý tưởng ở đây quan trọng hơn command cụ thể:

```text
journalctl
    lấy đúng dataset

grep
    giữ lại record cần quan tâm

awk
    lấy / tính field

sed
    transform text nếu cần
```

Đây chính là pattern bạn sẽ dùng sau này với:

- Tomcat logs
- Java stack traces
- PostgreSQL logs
- Cloud-init logs
- deployment logs
- Prometheus exporter logs
- batch logs

---

# 4. `journalctl` — đọc log theo tư duy vận hành

## 4.1 systemd journal là gì?

Trên systemd Linux, `systemd-journald` thu thập log từ nhiều nguồn như:

```text
systemd services stdout/stderr
kernel messages
syslog-compatible sources
boot/service metadata
```

Khác với việc chỉ đọc một flat file, journal chứa **structured metadata** như:

```text
timestamp
boot ID
PID
UID
systemd unit
priority
identifier
message
```

`journalctl` là command để query journal.

Manual hiện hành mô tả `journalctl` là công cụ in/query các entry do systemd journal lưu, đồng thời hỗ trợ filter theo structured fields. [man7](https://man7.org/linux/man-pages/man1/journalctl.1%40%40systemd.html?utm_source=chatgpt.com)

---

## 4.2 Đầu tiên: đừng chạy `journalctl` vô điều kiện

Lệnh này:

```bash
journalctl
```

có thể trả ra lượng log rất lớn.

Trong incident, luôn cố thu hẹp theo:

```text
service
time
boot
priority
keyword
```

---

## 4.3 Log của một service

```bash
journalctl -u ssh.service
```

Ví dụ Tomcat sau này:

```bash
journalctl -u tomcat.service
```

Cron:

Ubuntu/Debian:

```bash
journalctl -u cron.service
```

RHEL-like:

```bash
journalctl -u crond.service
```

`-u/--unit` lọc theo systemd unit. [man7](https://www.man7.org/linux/man-pages/man1/journalctl.1%40%40systemd.html?utm_source=chatgpt.com)

---

## 4.4 Log của boot hiện tại

```bash
journalctl -b
```

Boot trước:

```bash
journalctl -b -1
```

Rất quan trọng khi troubleshooting tình huống:

> “Service trước reboot bị lỗi hay sau reboot mới lỗi?”

---

## 4.5 Query theo thời gian

Ví dụ:

```bash
journalctl --since "today"
```

```bash
journalctl --since "1 hour ago"
```

```bash
journalctl \
  --since "2026-10-12 14:00:00" \
  --until "2026-10-12 14:30:00"
```

Trong production incident, **time correlation** cực kỳ quan trọng.

Ví dụ ticket ghi:

```text
14:07 user bắt đầu report HTTP 500
```

Thì không nên đọc log từ cả ngày.

Hãy query quanh khoảng đó:

```bash
journalctl -u myapp.service \
  --since "14:00" \
  --until "14:15"
```

---

# 5. Log priority

Syslog/systemd sử dụng severity:

| Number | Name      | Ý nghĩa                  |
| -----: | --------- | ------------------------ |
|      0 | `emerg`   | system unusable          |
|      1 | `alert`   | immediate action         |
|      2 | `crit`    | critical                 |
|      3 | `err`     | error                    |
|      4 | `warning` | warning                  |
|      5 | `notice`  | significant normal event |
|      6 | `info`    | informational            |
|      7 | `debug`   | debugging                |

Ví dụ:

```bash
journalctl -p err
```

Một điểm thi dễ sai:

> `journalctl -p err` không chỉ có đúng `err`.

Một priority đơn sẽ bao gồm mức đó **và các mức nghiêm trọng hơn**. Current `journalctl` manual xác nhận priority có thể nhận single level hoặc range. [man7](https://man7.org/linux/man-pages/man1/journalctl.1%40%40systemd.html?utm_source=chatgpt.com)

Muốn range:

```bash
journalctl -p warning..err
```

---

# 6. Theo dõi realtime

Tương tự:

```bash
tail -f
```

với journal:

```bash
journalctl -f
```

Cho service:

```bash
journalctl -u myapp.service -f
```

Một workflow troubleshooting rất hay:

Terminal A:

```bash
journalctl -u myapp.service -f
```

Terminal B:

```bash
curl http://localhost:8080/
```

Bạn trigger request ở B và quan sát log tương ứng ở A.

---

# 7. Lấy N dòng gần nhất

```bash
journalctl -n 50
```

Cho service:

```bash
journalctl -u ssh.service -n 100
```

`-n` mặc định hiển thị các event gần nhất; manual hiện hành ghi nhận mặc định là 10 khi không truyền số. [man7](https://www.man7.org/linux/man-pages/man1/journalctl.1%40%40systemd.html?utm_source=chatgpt.com)

---

# 8. Không dùng pager khi script

Interactive:

```bash
journalctl -u ssh
```

thường mở `less`.

Trong script/pipeline:

```bash
journalctl -u ssh --no-pager
```

Nếu không làm vậy, automation có thể gặp hành vi không mong muốn.

---

# 9. Journal persistence — điểm operations quan trọng

Journal có thể sử dụng storage dưới:

```text
/run/log/journal
```

cho volatile runtime storage hoặc:

```text
/var/log/journal
```

cho persistent storage.

Không nên mặc định rằng logs chắc chắn tồn tại sau reboot. Kiểm tra cấu hình thực tế. `journald.conf` hỗ trợ các mode như `volatile`, `persistent`, `auto`, `none`; cách distro build/configure có thể ảnh hưởng hành vi thực tế. [man7](https://www.man7.org/linux/man-pages/man5/journald.conf.5.html?utm_source=chatgpt.com)

Kiểm tra:

```bash
ls -ld /var/log/journal /run/log/journal 2>/dev/null
```

Disk usage của journal:

```bash
journalctl --disk-usage
```

### L1 lesson

Đừng trả lời:

> “Log không có nghĩa là sự kiện không xảy ra.”

Có thể:

```text
log retention hết
volatile journal
service log sang file khác
insufficient permission
wrong boot
wrong unit
wrong time window
```

---

# 10. Log access và least privilege

User thường không nhất thiết xem được toàn bộ system journal.

Current systemd documentation lưu ý quyền đọc full journal có thể được cấp thông qua các group như `systemd-journal`, và một số distro sử dụng `adm` hoặc `wheel`. [man7](https://man7.org/linux/man-pages/man1/journalctl.1%40%40systemd.html?utm_source=chatgpt.com)

Không nên reflexively chạy mọi thứ bằng:

```bash
sudo journalctl ...
```

Hãy hiểu liệu quyền admin có thật sự cần hay không.

---

# 11. `grep` — lọc record

Mental model:

```text
grep = keep lines matching a pattern
```

Ví dụ:

```bash
grep "ERROR" application.log
```

Case insensitive:

```bash
grep -i "error" application.log
```

Line number:

```bash
grep -n "ERROR" application.log
```

Recursive:

```bash
grep -r "database.url" /opt/myapp/
```

Invert:

```bash
grep -v "DEBUG" application.log
```

Fixed string:

```bash
grep -F "java.lang.NullPointerException" app.log
```

Count:

```bash
grep -c "ERROR" app.log
```

ERE:

```bash
grep -E "ERROR|WARN|CRITICAL" app.log
```

---

# 12. Regex vs literal

Lệnh:

```bash
grep "10.0.0.1" file
```

dấu `.` trong regex nghĩa là bất kỳ character.

Nếu muốn literal:

```bash
grep -F "10.0.0.1" file
```

Hoặc escape:

```bash
grep '10\.0\.0\.1' file
```

Trong operations, nếu tìm một literal identifier, IP, exception name:

```bash
grep -F
```

thường an toàn hơn.

---

# 13. Exit code của `grep` — cực kỳ quan trọng

GNU `grep` có semantics:

````text
0 = tìm thấy match
1 = không tìm thấy match
2 = error
``` :chatgpt-content-reference{index="7"}


Do đó:

```bash
grep -q "ERROR" app.log
````

sau đó:

```bash
echo $?
```

Nếu trả `1`:

> Không có match.

Nó **không nhất thiết là command bị lỗi**.

Đây là lý do viết:

```bash
set -e
```

mà không hiểu exit semantics có thể gây script bug.

Ví dụ:

```bash
set -e

grep -q "ERROR" app.log

echo "continue"
```

Nếu không có `ERROR`, `grep` trả 1 và script có thể terminate.

---

# 14. `awk` — xử lý fields và calculations

Mental model:

```text
grep → line-oriented filter
awk  → record + field processing language
```

Giả sử:

```text
alice 82
bob 45
charlie 91
```

In field 1:

```bash
awk '{print $1}' scores.txt
```

Output:

```text
alice
bob
charlie
```

Field 2:

```bash
awk '{print $2}' scores.txt
```

---

## 14.1 Các variable cần thuộc

```text
$0   whole record
$1   field 1
$2   field 2
...
NF   number of fields
NR   current record number
FS   input field separator
OFS  output field separator
```

GNU awk được thiết kế để select record theo pattern rồi thực hiện action trên các record đó. [gnu.org](https://www.gnu.org/software/gawk/manual/gawk.html?utm_source=chatgpt.com)

---

## 14.2 Condition

```bash
awk '$2 >= 80 {print $1, $2}' scores.txt
```

---

## 14.3 Separator

File CSV đơn giản:

```text
alice,82
bob,45
```

Dùng:

```bash
awk -F, '{print $1, $2}' scores.csv
```

Lưu ý: `awk -F,` **không phải CSV parser hoàn chỉnh**.

CSV có quoted comma như:

```text
"Smith, John",82
```

cần parser phù hợp.

---

# 15. `awk` với `df`

Ví dụ:

```bash
df -P /
```

Có thể cho:

```text
Filesystem  1024-blocks    Used Available Capacity Mounted on
/dev/sda1      50000000 8000000  42000000      17% /
```

Lấy phần trăm:

```bash
df -P / | awk 'NR==2 {print $5}'
```

Output:

```text
17%
```

Bỏ `%`:

```bash
df -P / |
awk 'NR==2 {gsub(/%/, "", $5); print $5}'
```

Output:

```text
17
```

Đó chính là pattern sẽ dùng trong health-check script.

GNU `df -P` cung cấp portable-style output và cố đảm bảo filesystem record nằm trên một line, thích hợp hơn cho basic scripting. [gnu.org](https://www.gnu.org/software/coreutils/manual/coreutils.html?utm_source=chatgpt.com)

---

# 16. `sed` — stream transformation

Mental model:

```text
sed = transform text stream
```

Thay text:

```bash
sed 's/dev/prod/' config.txt
```

Lệnh này mặc định **không sửa file**. Nó in kết quả ra stdout.

Global trong mỗi line:

```bash
sed 's/foo/bar/g' file
```

---

## Delete line

```bash
sed '/DEBUG/d' app.log
```

---

## Chỉ print match

```bash
sed -n '/ERROR/p' app.log
```

---

## In-place editing

GNU sed:

```bash
sed -i 's/8080/8081/' application.conf
```

### Cảnh báo

`-i` thay đổi file.

Trước khi dùng production config:

```bash
cp application.conf application.conf.bak
```

Sau đó:

```bash
sed -i 's/8080/8081/' application.conf
```

Validate:

```bash
grep '808' application.conf
```

GNU `sed` current documentation xác định non-zero exit values cho syntax error, inaccessible file hoặc runtime I/O error. [gnu.org](https://www.gnu.org/software/sed/manual/html_node/Exit-status.html?utm_source=chatgpt.com)

---

# 17. Khi nào grep, awk, sed?

Một mental model nên nhớ:

| Tool   | Hỏi câu gì?                |
| ------ | -------------------------- |
| `grep` | “Dòng nào match?”          |
| `awk`  | “Field nào? tính toán gì?” |
| `sed`  | “Biến đổi text thế nào?”   |

Ví dụ:

```bash
journalctl -u myapp --no-pager \
| grep "ERROR" \
| awk '{print $1, $2, $3}' \
| sed 's/^/[APP-ERROR] /'
```

Nhưng đừng tạo pipeline 8–10 commands chỉ để chứng minh mình biết command.

Rule:

> Pipeline phải dễ đọc, dễ debug, dễ xác minh.

---

# 18. Bash scripting foundation

## 18.1 Shebang

```bash
#!/usr/bin/env bash
```

Nó yêu cầu hệ thống tìm `bash` trong `PATH`.

Một lựa chọn khác:

```bash
#!/bin/bash
```

Trong managed production environment, `/bin/bash` có thể cho execution path deterministic hơn nếu đó là standard của hệ thống.

---

# 19. Variable

```bash
name="java-admin"
```

Không:

```bash
name = "java-admin"
```

Dùng:

```bash
echo "$name"
```

Best practice:

```bash
"$variable"
```

thay vì:

```bash
$variable
```

Ví dụ:

```bash
file="/tmp/My File.txt"

cat "$file"
```

Không quote:

```bash
cat $file
```

shell sẽ hiểu thành hai arguments.

---

# 20. Command substitution

```bash
hostname_value=$(hostname)
```

```bash
used=$(df -P / | awk 'NR==2 {print $5}')
```

---

# 21. Conditions

```bash
if [[ -f /etc/passwd ]]; then
    echo "exists"
fi
```

Directory:

```bash
[[ -d /var/log ]]
```

String:

```bash
[[ "$state" == "active" ]]
```

Integer:

```bash
if (( usage >= 80 )); then
    echo "WARNING"
fi
```

---

# 22. Loops

```bash
for service in ssh cron; do
    systemctl is-active "$service"
done
```

While:

```bash
while read -r line; do
    echo "$line"
done < input.txt
```

Luôn ưu tiên:

```bash
read -r
```

để backslash không bị interpret ngoài ý muốn.

---

# 23. Functions

```bash
log_info() {
    printf '%s [INFO] %s\n' "$(date)" "$*"
}
```

Call:

```bash
log_info "backup started"
```

Function giúp:

```text
giảm duplicate code
chia logic
test dễ hơn
đọc dễ hơn
```

---

# 24. stdout, stderr và redirection

Linux process conventionally có:

```text
0 stdin
1 stdout
2 stderr
```

Redirect stdout:

```bash
command > output.log
```

Append:

```bash
command >> output.log
```

stderr:

```bash
command 2> error.log
```

Cả stdout và stderr:

```bash
command > output.log 2>&1
```

Append cả hai:

```bash
command >> output.log 2>&1
```

Điều này đặc biệt quan trọng với cron vì bạn cần evidence.

---

# 25. Pipe

```bash
command1 | command2
```

stdout của command 1 → stdin của command 2.

Ví dụ:

```bash
journalctl -u ssh --no-pager |
grep "Failed"
```

---

# 26. Exit codes — xương sống của automation

Linux không hiểu câu:

> “Output trông có vẻ ổn.”

Automation cần machine-readable status.

Convention:

```text
0       success
non-zero failure / special condition
```

Bash cung cấp exit code của command vừa chạy qua:

```bash
$?
```

Ví dụ:

```bash
ls /etc/passwd
echo $?
```

Bash documentation xác nhận:

- `0`: success
- non-zero: failure/condition
- command not found thường → `127`
- command tìm thấy nhưng không executable → `126`
- termination bởi signal N → `128 + N` [gnu.org](https://www.gnu.org/s/bash/manual/html_node/Exit-Status.html?utm_source=chatgpt.com)

---

# 27. `&&` và `||`

```bash
mkdir backup && echo "created"
```

Command sau chỉ chạy khi command trước trả 0.

```bash
systemctl is-active --quiet myapp ||
echo "service unhealthy"
```

---

# 28. Pipeline exit code trap

Giả sử:

```bash
false | true
echo $?
```

Thông thường pipeline status là status của **last command**:

```text
true → 0
```

Nên bạn có thể vô tình bỏ qua failure.

Bash có:

```bash
set -o pipefail
```

Sau đó pipeline sẽ non-zero nếu một command trong pipeline thất bại theo semantics của `pipefail`. [gnu.org](https://www.gnu.org/s/bash/manual/html_node/The-Set-Builtin.html?utm_source=chatgpt.com)

Đây là kiến thức rất quan trọng cho operations script.

---

# 29. `set -e` không phải magic

Bạn thường thấy:

```bash
set -e
```

hoặc:

```bash
set -euo pipefail
```

Ý nghĩa:

```text
-e   exit trong nhiều trường hợp command non-zero
-u   sử dụng undefined variable → error
-o pipefail
     pipeline phản ánh failure bên trong
```

Nhưng `-e` có nhiều exception trong:

```text
if
while
until
&&
||
pipeline contexts
!
```

Bash manual mô tả những exception này khá chi tiết. [gnu.org](https://www.gnu.org/s/bash/manual/html_node/The-Set-Builtin.html?utm_source=chatgpt.com)

Vì vậy:

> `set -euo pipefail` không thay thế explicit error handling.

Trong script Day 6 bên dưới, tôi cố ý dùng:

```bash
set -u
set -o pipefail
```

và xử lý các condition quan trọng một cách explicit.

---

# 30. Cron — biến script thành scheduled operations

Cron daemon kiểm tra scheduled jobs và thực thi command đúng thời gian cấu hình. Cronie documentation cho biết daemon kiểm tra các crontab định kỳ theo phút và hỗ trợ user crontab cùng `/etc/crontab`, `/etc/cron.d`. [man7](https://www.man7.org/linux/man-pages/man8/crond.8.html?utm_source=chatgpt.com)

User cron:

```bash
crontab -e
```

Xem:

```bash
crontab -l
```

---

# 31. Cron syntax

User crontab:

```text
minute hour day-of-month month day-of-week command
```

Ví dụ:

```text
15 2 * * * /home/admin/backup.sh
```

Nghĩa là:

```text
02:15 mỗi ngày
```

Allowed ranges trong Cronie:

```text
minute        0-59
hour          0-23
day-of-month  1-31
month         1-12
day-of-week   0-7
```

0 và 7 đều có thể biểu diễn Sunday. [man7](https://www.man7.org/linux/man-pages/man5/crontab.5.html?utm_source=chatgpt.com)

---

# 32. Examples

Mỗi 5 phút:

```text
*/5 * * * * command
```

Mỗi giờ:

```text
0 * * * * command
```

02:30 mỗi ngày:

```text
30 2 * * * command
```

Thứ Hai–Thứ Sáu 09:00:

```text
0 9 * * 1-5 command
```

---

# 33. Một cron trap rất hay gặp

```text
*/35 * * * *
```

Không nghĩa là:

> “mỗi 35 phút liên tục”.

Nó match minute:

```text
00
35
```

trong mỗi giờ.

Cronie manual cũng đặc biệt lưu ý skip values hoạt động trong field tương ứng. [man7](https://man7.org/linux/man-pages/man5/crontab.5.html?utm_source=chatgpt.com)

---

# 34. User crontab vs `/etc/crontab`

User:

```bash
crontab -e
```

format:

```text
* * * * * command
```

System `/etc/crontab` hoặc `/etc/cron.d` thường có thêm username:

```text
* * * * * root command
```

Đây là lỗi thi/practical rất phổ biến.

Đừng đặt:

```text
* * * * * username command
```

vào user `crontab -e`.

---

# 35. Cron environment — nguyên nhân “chạy tay được nhưng cron fail”

Interactive shell của bạn có thể có:

```text
PATH
JAVA_HOME
aliases
.bashrc
SDK variables
working directory
```

Cron environment thường tối giản hơn.

Ví dụ chạy tay:

```bash
backup.sh
```

OK.

Cron:

```text
backup.sh: command not found
```

Nguyên nhân có thể script gọi một program không nằm trong cron PATH.

Best practice:

```text
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin
```

và script sử dụng absolute/configured paths khi cần.

Đừng dựa vào:

```bash
cd ~/somewhere
```

mà không kiểm tra.

---

# 36. Cron job luôn cần log

Không nên:

```text
*/5 * * * * script.sh
```

trong lab operations.

Tốt hơn:

```text
*/5 * * * * "$HOME/day6-lab/bin/day6_ops.sh" >> "$HOME/day6-lab/logs/cron.log" 2>&1
```

Nhờ vậy khi cron fail bạn có:

```text
timestamp
stdout
stderr
script exit evidence
```

---

# 37. Cron troubleshooting workflow

Nếu job không chạy:

```bash
crontab -l
```

Sau đó:

Ubuntu:

```bash
systemctl status cron
journalctl -u cron --since "30 minutes ago"
```

RHEL-family:

```bash
systemctl status crond
journalctl -u crond --since "30 minutes ago"
```

Sau đó kiểm tra:

```text
script executable?
shebang?
PATH?
permission?
source/target path?
environment?
timezone?
stdout/stderr?
```

Cronie còn có `crontab -T` để test syntax trên implementation hỗ trợ nó; không nên giả định mọi cron implementation đều có option này. [man7](https://man7.org/linux/man-pages/man5/crontab.5.html?utm_source=chatgpt.com)

---

# 38. Backup với `tar`

Mental model:

```text
tar
  gom nhiều file/directory thành archive

gzip
  compression
```

Tạo archive:

```bash
tar -czf backup.tar.gz data/
```

Ý nghĩa:

```text
-c create
-z gzip
-f archive filename
```

List:

```bash
tar -tzf backup.tar.gz
```

Extract:

```bash
tar -xzf backup.tar.gz
```

GNU tar định nghĩa ba operation chính là `create`, `list`, `extract`. [gnu.org](https://www.gnu.org/software/tar/manual/tar.html?utm_source=chatgpt.com)

---

# 39. Tạo archive không có nghĩa backup thành công

Đây là distinction cực kỳ quan trọng.

Bad:

```text
tar returned 0
→ backup good
```

Better:

```text
tar returned 0
+
archive exists
+
archive can be listed
+
restore test succeeds
+
restored content validated
```

Một backup mà chưa thử restore vẫn chưa được chứng minh là recoverable.

---

# 40. Restore vào staging directory

Không test restore bằng cách extract thẳng vào source.

Tạo:

```bash
mkdir -p restore-test
```

Sau đó:

```bash
tar -xzf backup.tar.gz -C restore-test
```

Validate:

```bash
find restore-test -type f
```

Có thể compare:

```bash
diff -r source-dir restore-test/source-dir
```

### Expected invariant

Nếu source không đổi giữa backup và comparison:

```bash
diff -r ...
```

phải trả:

```text
exit 0
```

---

# 41. `rsync`

`rsync` phù hợp cho synchronization/copy có khả năng incremental.

Basic:

```bash
rsync -av source/ destination/
```

Common:

```text
-a  archive mode
-v  verbose
-n  dry-run
-i  itemize changes
```

Dry run:

```bash
rsync -avni source/ destination/
```

Current rsync manual mô tả `--dry-run/-n` là trial run không thực hiện change. [download.samba.org](https://download.samba.org/pub/rsync/rsync.1.html?utm_source=chatgpt.com)

---

# 42. Trailing slash của rsync

Hai command này khác ý nghĩa:

```bash
rsync -a source/ destination/
```

và:

```bash
rsync -a source destination/
```

Thông thường:

```text
source/
```

nghĩa là copy **contents của source**.

Trong khi:

```text
source
```

có thể tạo:

```text
destination/source
```

Luôn kiểm tra bằng:

```bash
rsync -avni ...
```

trước khi làm thao tác lớn.

---

# 43. `--delete` là high-risk

```bash
rsync -a --delete source/ destination/
```

sẽ có thể xóa file ở destination nếu file đó không tồn tại source.

Official rsync documentation cảnh báo option này có thể nguy hiểm nếu sử dụng không đúng và khuyến nghị trial run trước. [download.samba.org](https://download.samba.org/pub/rsync/rsync.1.html?utm_source=chatgpt.com)

Operations rule:

```text
KHÔNG thêm --delete theo thói quen.
```

Trước:

```bash
rsync -avni --delete source/ destination/
```

Review change list.

Sau đó mới cân nhắc execution nếu change approved.

---

# 44. Tar vs rsync

| Requirement               |                     `tar` |                      `rsync` |
| ------------------------- | ------------------------: | ---------------------------: |
| Single archive file       |                       tốt |                        không |
| Compression               |                        dễ | không phải chức năng cốt lõi |
| Incremental repeated copy |               hạn chế hơn |                          tốt |
| Mirror directories        | không phải mục tiêu chính |                      rất tốt |
| Remote synchronization    | thường cần transport khác |           built-in workflows |
| Snapshot/archive artifact |                       tốt |                   tùy design |

Không có command nào tự động biến thành “backup strategy tốt”.

---

# 45. Important: database backup

Sau này với PostgreSQL:

> Không được mặc định `tar /var/lib/postgresql/...` trong lúc DB đang chạy là logical/consistent database backup.

Database cần DB-aware backup mechanism như `pg_dump` hoặc phương pháp physical backup được hỗ trợ.

Đây sẽ học sâu hơn Day 15.

---

# 46. Guided Lab — Day 6 Mandatory Assignment

Bây giờ ta thực hiện assignment của syllabus.

## Lab objective

Bạn sẽ:

```text
1. tạo health-check + backup script
2. log kết quả
3. dùng exit code
4. tạo tar backup
5. verify backup
6. test rsync
7. schedule bằng cron
8. inject failure
9. collect evidence
10. restore
11. quyết định remediation/escalation
```

### Safety

Lab chỉ thao tác dưới:

```text
$HOME/day6-lab
```

Không thay đổi production service.

Không dùng:

```bash
chmod -R 777
```

Không xóa `/var/log`.

Không fill filesystem để test disk-full.

Không stop SSH.

Không chỉnh firewall.

Không test `rsync --delete` trên dữ liệu thật.

---

# 47. Lab Phase 0 — Environment inspection

```bash
whoami
hostname
cat /etc/os-release
date
timedatectl 2>/dev/null || true
```

Check commands:

```bash
for cmd in bash journalctl grep awk sed tar rsync crontab; do
    command -v "$cmd" || echo "MISSING: $cmd"
done
```

### Nếu `rsync` chưa có

Ubuntu/Debian, chỉ khi được phép:

```bash
sudo apt update
sudo apt install rsync
```

RHEL-family:

```bash
sudo dnf install rsync
```

Cron package nếu lab machine chưa có:

Ubuntu:

```bash
sudo apt install cron
```

RHEL-family:

```bash
sudo dnf install cronie
```

Package installation làm thay đổi system state; chỉ thực hiện trên lab hoặc khi change được phép.

---

# 48. Tạo workspace

```bash
LAB="$HOME/day6-lab"

mkdir -p "$LAB"/{bin,data,backup,logs,evidence,restore}
```

Verify:

```bash
find "$LAB" -maxdepth 1 -type d -print
```

Expected invariant:

```text
bin
data
backup
logs
evidence
restore
```

---

# 49. Tạo sample data

```bash
cat > "$LAB/data/application.conf" <<'EOF'
app.name=day6-demo
app.port=8080
app.environment=lab
EOF
```

```bash
cat > "$LAB/data/operators.txt" <<'EOF'
alice admin
bob operator
charlie viewer
EOF
```

Permissions:

```bash
chmod 600 "$LAB/data/application.conf"
chmod 640 "$LAB/data/operators.txt"
```

Evidence:

```bash
ls -l "$LAB/data" |
tee "$LAB/evidence/initial-files.txt"
```

---

# 50. Journal practice

Recent errors:

```bash
journalctl -p err -n 20 --no-pager
```

Current boot:

```bash
journalctl -b -n 30 --no-pager
```

Current cron service:

Ubuntu:

```bash
journalctl -u cron -n 20 --no-pager
```

RHEL:

```bash
journalctl -u crond -n 20 --no-pager
```

Save evidence:

```bash
journalctl -b -n 50 --no-pager \
  > "$LAB/evidence/journal-current-boot.txt"
```

---

# 51. Text-processing drills

Find port:

```bash
grep -n 'port' "$LAB/data/application.conf"
```

Extract config keys:

```bash
awk -F= '{print $1}' "$LAB/data/application.conf"
```

Output should resemble:

```text
app.name
app.port
app.environment
```

Transform for display:

```bash
sed 's/^/[CONFIG] /' "$LAB/data/application.conf"
```

Do not modify source yet.

---

# 52. Build the integrated operations script

Create:

```bash
nano "$LAB/bin/day6_ops.sh"
```

hoặc editor của bạn.

Paste:

```bash
#!/usr/bin/env bash

set -u
set -o pipefail

umask 077

BASE_DIR="${DAY6_BASE_DIR:-$HOME/day6-lab}"
SOURCE_DIR="${SOURCE_DIR:-$BASE_DIR/data}"
BACKUP_DIR="${BACKUP_DIR:-$BASE_DIR/backup}"
LOG_DIR="${LOG_DIR:-$BASE_DIR/logs}"

DISK_TARGET="${DISK_TARGET:-/}"
DISK_WARN_PCT="${DISK_WARN_PCT:-80}"
INODE_WARN_PCT="${INODE_WARN_PCT:-80}"

# Optional.
# Example:
# CHECK_SERVICE=cron ./day6_ops.sh
CHECK_SERVICE="${CHECK_SERVICE:-}"

mkdir -p "$BACKUP_DIR" "$LOG_DIR" || {
    printf 'ERROR: cannot create backup/log directories\n' >&2
    exit 20
}

LOG_FILE="$LOG_DIR/day6_ops_$(date +%F).log"

rc=0

log() {
    local level="$1"
    shift

    printf '%s [%s] %s\n' \
        "$(date '+%F %T%z')" \
        "$level" \
        "$*" |
        tee -a "$LOG_FILE"
}

set_rc() {
    local new_rc="$1"

    if (( new_rc > rc )); then
        rc="$new_rc"
    fi
}

check_disk() {
    local used
    local inode_used

    if ! used=$(
        df -P "$DISK_TARGET" 2>>"$LOG_FILE" |
        awk 'NR==2 {gsub(/%/, "", $5); print $5}'
    ); then
        log ERROR "df failed for $DISK_TARGET"
        set_rc 20
        return
    fi

    if [[ ! "$used" =~ ^[0-9]+$ ]]; then
        log ERROR "unexpected disk usage value: '$used'"
        set_rc 20

    elif (( used >= DISK_WARN_PCT )); then
        log WARN \
            "disk usage on $DISK_TARGET is ${used}% (threshold ${DISK_WARN_PCT}%)"
        set_rc 10

    else
        log OK "disk usage on $DISK_TARGET is ${used}%"
    fi

    if ! inode_used=$(
        df -Pi "$DISK_TARGET" 2>>"$LOG_FILE" |
        awk 'NR==2 {gsub(/%/, "", $5); print $5}'
    ); then
        log ERROR "inode check failed for $DISK_TARGET"
        set_rc 20
        return
    fi

    if [[ "$inode_used" =~ ^[0-9]+$ ]] &&
       (( inode_used >= INODE_WARN_PCT )); then

        log WARN \
            "inode usage on $DISK_TARGET is ${inode_used}% (threshold ${INODE_WARN_PCT}%)"
        set_rc 10

    else
        log OK "inode usage on $DISK_TARGET is ${inode_used}%"
    fi
}

check_service() {
    if [[ -z "$CHECK_SERVICE" ]]; then
        log INFO "service check skipped (CHECK_SERVICE is empty)"
        return
    fi

    if systemctl is-active --quiet "$CHECK_SERVICE" \
        2>>"$LOG_FILE"; then

        log OK "service $CHECK_SERVICE is active"

    else
        log WARN "service $CHECK_SERVICE is not active"
        set_rc 10
    fi
}

create_backup() {
    local ts
    local parent
    local base
    local archive
    local partial

    if [[ ! -d "$SOURCE_DIR" ]]; then
        log ERROR "backup source does not exist: $SOURCE_DIR"
        set_rc 20
        return
    fi

    ts=$(date '+%Y%m%d_%H%M%S')

    parent=$(dirname -- "$SOURCE_DIR")
    base=$(basename -- "$SOURCE_DIR")

    archive="$BACKUP_DIR/${base}_${ts}.tar.gz"
    partial="${archive}.partial"

    log INFO "creating backup $archive"

    if tar -czf "$partial" \
        -C "$parent" \
        "$base" \
        2>>"$LOG_FILE"; then

        if tar -tzf "$partial" \
            >/dev/null \
            2>>"$LOG_FILE"; then

            mv -- "$partial" "$archive"

            log OK \
                "backup created and archive listing verified: $archive"

        else
            rm -f -- "$partial"

            log ERROR "backup archive verification failed"
            set_rc 20
        fi

    else
        rm -f -- "$partial"

        log ERROR "tar backup failed"
        set_rc 20
    fi
}

main() {
    log INFO "Day 6 operations job started"

    check_disk
    check_service
    create_backup

    log INFO "Day 6 operations job finished with rc=$rc"

    return "$rc"
}

main
exit $?
```

---

# 53. Hiểu script, không chỉ copy

Đây là phần rất quan trọng.

### `umask 077`

```bash
umask 077
```

file mới do script tạo sẽ mặc định hạn chế access cho group/others.

Backup có thể chứa:

```text
config
host information
internal paths
credentials nếu thiết kế tệ
```

nên backup không nên world-readable.

---

### Configuration thông qua environment

```bash
SOURCE_DIR="${SOURCE_DIR:-$BASE_DIR/data}"
```

Nghĩa là:

```text
nếu SOURCE_DIR đã được set → dùng nó
nếu chưa → dùng default
```

Ta có thể test:

```bash
SOURCE_DIR=/tmp/test ./day6_ops.sh
```

mà không sửa source code.

Đây là pattern rất quan trọng trước Day 13 externalized Java configuration.

---

### Exit-code design

Script của chúng ta định nghĩa:

```text
0   all required operations healthy

10  warning / health condition
    ví dụ disk high, service inactive

20  operational failure
    ví dụ df failure, source missing,
    tar failure, verification failure
```

Đây là **custom contract của script**, không phải universal Linux standard.

Điểm quan trọng:

> Document exit-code semantics.

---

### `.partial`

Thay vì tạo thẳng:

```text
data_20261012.tar.gz
```

script tạo:

```text
data_20261012.tar.gz.partial
```

Chỉ sau khi `tar -tzf` pass mới:

```bash
mv
```

thành archive final.

Điều này giúp giảm nguy cơ một failed/incomplete archive bị nhìn nhầm là completed backup.

---

# 54. Validate syntax trước execution

```bash
bash -n "$LAB/bin/day6_ops.sh"
```

Expected:

```text
không có output
exit code 0
```

Check:

```bash
echo $?
```

---

## Supplementary

Nếu máy có `shellcheck`:

```bash
shellcheck "$LAB/bin/day6_ops.sh"
```

`ShellCheck` rất hữu ích nhưng không phải syllabus requirement.

---

# 55. Set permission

```bash
chmod 700 "$LAB/bin/day6_ops.sh"
```

Verify:

```bash
ls -l "$LAB/bin/day6_ops.sh"
```

Không nên:

```bash
chmod 777
```

chỉ vì:

> “permission denied”.

Bạn phải hiểu ai cần execute.

---

# 56. First execution

```bash
"$LAB/bin/day6_ops.sh"
```

Sau đó ngay lập tức:

```bash
echo $?
```

### Example output

Con số thực tế của bạn sẽ khác:

```text
2026-10-12 16:00:00+0700 [INFO] Day 6 operations job started
2026-10-12 16:00:00+0700 [OK] disk usage on / is 34%
2026-10-12 16:00:00+0700 [OK] inode usage on / is 6%
2026-10-12 16:00:00+0700 [INFO] service check skipped (CHECK_SERVICE is empty)
2026-10-12 16:00:00+0700 [INFO] creating backup ...
2026-10-12 16:00:00+0700 [OK] backup created and archive listing verified ...
2026-10-12 16:00:00+0700 [INFO] Day 6 operations job finished with rc=0
```

### Expected invariant

Không quan trọng disk là `34%` hay `52%`.

Quan trọng:

```text
script chạy hoàn tất
backup archive xuất hiện
log được tạo
tar listing thành công
exit code = 0 khi không có condition lỗi
```

---

# 57. Verify artifacts

```bash
find "$LAB" -maxdepth 2 -type f -print
```

Backup:

```bash
ls -lh "$LAB/backup"
```

Log:

```bash
cat "$LAB/logs"/day6_ops_*.log
```

Archive:

```bash
tar -tzf "$LAB"/backup/*.tar.gz
```

---

# 58. Restore test — bắt buộc về tư duy

Chọn archive mới nhất:

```bash
LATEST_BACKUP=$(
    ls -1t "$LAB"/backup/*.tar.gz |
    head -n 1
)
```

Inspect:

```bash
printf '%s\n' "$LATEST_BACKUP"
```

Tạo staging:

```bash
RESTORE_DIR="$LAB/restore/test1"

mkdir -p "$RESTORE_DIR"
```

Extract:

```bash
tar -xzf "$LATEST_BACKUP" \
    -C "$RESTORE_DIR"
```

Verify:

```bash
find "$RESTORE_DIR" -type f -print
```

Compare:

```bash
diff -r \
    "$LAB/data" \
    "$RESTORE_DIR/data"
```

Expected invariant:

```bash
echo $?
```

→

```text
0
```

nếu source chưa thay đổi sau backup.

---

# 59. rsync exercise

Tạo mirror:

```bash
mkdir -p "$LAB/backup/rsync-mirror"
```

Dry-run trước:

```bash
rsync -avni \
    "$LAB/data/" \
    "$LAB/backup/rsync-mirror/"
```

Review output.

Sau đó mới actual:

```bash
rsync -avi \
    "$LAB/data/" \
    "$LAB/backup/rsync-mirror/"
```

Verify:

```bash
diff -r \
    "$LAB/data" \
    "$LAB/backup/rsync-mirror"
```

Expected:

```text
exit 0
```

---

# 60. Service check

Tìm tên cron service.

Ubuntu:

```bash
systemctl status cron --no-pager
```

RHEL:

```bash
systemctl status crond --no-pager
```

Nếu Ubuntu:

```bash
CHECK_SERVICE=cron \
"$LAB/bin/day6_ops.sh"
```

Nếu RHEL:

```bash
CHECK_SERVICE=crond \
"$LAB/bin/day6_ops.sh"
```

Expected:

```text
[OK] service ... is active
```

nếu daemon active.

---

# 61. Cron scheduling

Trước khi thay crontab, backup existing configuration:

```bash
crontab -l \
    > "$LAB/evidence/crontab.before.txt" \
    2>/dev/null || true
```

Mở:

```bash
crontab -e
```

Trong **lab verification only**, chạy mỗi 5 phút:

```text
SHELL=/bin/bash
PATH=/usr/local/bin:/usr/bin:/bin

*/5 * * * * "$HOME/day6-lab/bin/day6_ops.sh" >> "$HOME/day6-lab/logs/cron.log" 2>&1
```

Verify:

```bash
crontab -l
```

Save evidence:

```bash
crontab -l \
    > "$LAB/evidence/crontab.after.txt"
```

Sau khi cron đã chạy:

```bash
cat "$LAB/logs/cron.log"
```

Và:

```bash
ls -lt "$LAB/backup"
```

Bạn phải thấy archive mới được tạo tự động.

---

# 62. Đừng giữ schedule 5 phút mãi

`*/5` chỉ để verify lab nhanh.

Sau khi test, đổi sang ví dụ:

```text
15 2 * * * "$HOME/day6-lab/bin/day6_ops.sh" >> "$HOME/day6-lab/logs/cron.log" 2>&1
```

Hoặc remove cron khi kết thúc lab.

Trong production, schedule phụ thuộc:

```text
business requirement
RPO
backup duration
load window
retention
dependency
maintenance schedule
```

---

# 63. Failure Injection 1 — safe health failure

Không stop service.

Không làm hỏng OS.

Ta inject bằng cách check một service không tồn tại:

```bash
CHECK_SERVICE=definitely-not-a-real-service.service \
"$LAB/bin/day6_ops.sh"
```

Sau đó:

```bash
echo $?
```

Expected:

```text
10
```

và log:

```text
[WARN] service definitely-not-a-real-service.service is not active
```

Điều này chứng minh script truyền health condition ra exit code.

---

# 64. Failure Injection 2 — backup source disappears

Đây là failure injection tốt hơn cho assignment.

Trước:

```bash
ls "$LAB/data"
```

Rename:

```bash
mv "$LAB/data" "$LAB/data.offline"
```

Chạy:

```bash
"$LAB/bin/day6_ops.sh"
```

Ngay sau:

```bash
echo $?
```

Expected invariant:

```text
exit code = 20
```

Log phải có:

```text
[ERROR] backup source does not exist
```

---

# 65. Recovery / rollback failure injection

Khôi phục source:

```bash
mv "$LAB/data.offline" "$LAB/data"
```

Validate:

```bash
ls -l "$LAB/data"
```

Rerun:

```bash
"$LAB/bin/day6_ops.sh"
```

```bash
echo $?
```

Expected:

```text
0
```

Đây mới là full troubleshooting lifecycle:

```text
healthy
↓
inject failure
↓
detect
↓
evidence
↓
root cause
↓
restore
↓
revalidate
```

---

# 66. Evidence-driven troubleshooting

Khi script fail, không sửa lung tung.

Đi theo sequence:

```text
1. Symptom
2. Expected behavior
3. Recent changes
4. Scope
5. Failing layer
6. Evidence
7. Hypotheses
8. Safest test
9. Root cause
10. Remediation
11. Validation
12. Prevention / escalation
```

Ví dụ failure vừa tạo:

### Symptom

```text
Scheduled backup failed.
```

### Expected

```text
Script should create validated .tar.gz and return rc=0.
```

### Evidence

```bash
tail -50 "$LAB/logs"/day6_ops_*.log
```

shows:

```text
backup source does not exist
```

### Scope

```text
health disk = healthy
script starts = yes
failure isolated to backup source
```

### Hypothesis

```text
SOURCE_DIR missing / renamed / unmounted
```

### Safe tests

```bash
ls -ld "$LAB/data"
```

```bash
find "$LAB" -maxdepth 1 -type d -print
```

Finds:

```text
data.offline
```

### Root cause

```text
Expected source path renamed from data → data.offline.
```

### Remediation

```bash
mv "$LAB/data.offline" "$LAB/data"
```

### Validation

```bash
"$LAB/bin/day6_ops.sh"
echo $?
```

→ `0`.

---

# 67. Escalation evidence template

Nếu L1 không được phép sửa root cause, evidence phải đủ để L2 không phải điều tra lại từ đầu.

Một good escalation có dạng:

```text
Incident:
Scheduled Day 6 backup job failed.

Detected:
2026-10-12 16:35 +07

Host:
<hostname>

User:
<service/user>

Expected:
Source $HOME/day6-lab/data should exist.
Backup should produce a validated tar.gz archive.
Expected script rc=0.

Actual:
Source directory is missing.
Script returned rc=20.

Evidence:
- day6_ops log attached
- ls -ld source output
- crontab entry captured
- previous successful backup identified
- filesystem utilization checked

Impact:
New backup cannot be created.
Existing prior backup is still present.

Recent change:
Source directory appears renamed/missing.

Hypothesis:
Source path changed or underlying filesystem unavailable.

Actions performed:
Read-only inspection only.
No destructive changes performed.

Current state:
Backup still failing.

Escalation request:
Confirm intended source path / filesystem state and authorize remediation.

Rollback/recovery:
Previous validated backup available at <path>.
```

Đó là operations evidence, không phải:

```text
"Backup broken, pls check."
```

---

# 68. Những anti-pattern cần tránh

### Anti-pattern 1

```bash
chmod -R 777 /some/path
```

để “fix permission”.

Sai vì:

```text
blast radius lớn
security exposure
che giấu root cause
khó rollback
```

---

### Anti-pattern 2

```bash
sudo ./script.sh
```

chỉ vì user execution fail.

Trước tiên hỏi:

```text
cần privilege nào?
tại sao?
file nào?
command nào?
```

---

### Anti-pattern 3

```bash
grep ERROR huge.log
```

không time-scope hay service-scope.

Tốt hơn:

```bash
journalctl -u service \
  --since "incident start" \
  --until "incident end" \
  --no-pager
```

rồi mới filter.

---

### Anti-pattern 4

```bash
rsync -a --delete ...
```

thử trực tiếp.

Phải:

```bash
rsync -avni --delete ...
```

trước.

---

### Anti-pattern 5

Backup nhưng chưa bao giờ restore.

```text
archive exists ≠ recoverable backup
```

---

### Anti-pattern 6

Cron job phụ thuộc interactive environment:

```bash
java ...
```

nhưng `JAVA_HOME` chỉ ở `.bashrc`.

---

### Anti-pattern 7

Ignore exit code.

```bash
./backup.sh
```

rồi kết luận:

> “Không thấy error trên màn hình nên chắc OK.”

Phải:

```bash
./backup.sh
rc=$?

echo "$rc"
```

---

# 69. Security & least privilege

Script:

```bash
chmod 700 "$LAB/bin/day6_ops.sh"
```

Backup:

```bash
umask 077
```

Không embed secret:

```bash
DB_PASSWORD="super-secret"
```

trong script.

Không log secret:

```bash
echo "$DB_PASSWORD"
```

Không để cron command chứa credentials:

```text
* * * * * app --password MySecret123
```

vì nó có thể xuất hiện trong:

```text
crontab
process arguments
audit/history
```

Luôn quote variables:

```bash
"$SOURCE_DIR"
```

Không:

```bash
$SOURCE_DIR
```

Đặc biệt nguy hiểm với các command thay đổi/xóa dữ liệu.

---

# 70. Backup security

Backup có thể chứa:

```text
config files
user information
application data
internal hostnames
private operational information
possibly secrets
```

Do đó cần:

```text
least privilege
retention
restricted destination
integrity checking
tested restore procedure
```

---

# 71. Logs cũng có thể chứa sensitive data

Application có thể vô tình log:

```text
password
token
Authorization header
session ID
database URL
customer data
```

Vì vậy:

```bash
journalctl
```

không phải dữ liệu “public”.

Không paste log nguyên xi sang ticket/public channel mà chưa review.

---

# 72. Operations / recovery / rollback

Một job tốt phải trả lời được:

```text
Nó chạy lúc nào?
Ai chạy?
Nó làm gì?
Thành công hay fail?
Output ở đâu?
Backup ở đâu?
Restore thế nào?
Nếu schedule sai thì rollback thế nào?
```

Cron rollback:

```bash
crontab -e
```

remove line mới.

Hoặc nếu muốn restore prior crontab, trước tiên **review file**:

```bash
cat "$LAB/evidence/crontab.before.txt"
```

Không overwrite blind vì file có thể trống hoặc stale.

---

# 73. Backup recovery hierarchy

Khi source bị hỏng:

```text
1. Stop further destructive activity.
2. Identify last known-good backup.
3. Verify archive readability.
4. Restore to staging.
5. Validate.
6. Only then plan production restoration.
```

Không:

```bash
tar -xzf backup.tar.gz -C /
```

theo phản xạ.

---

# 74. Linux consolidation — kết nối Day 2 đến Day 6

Đây là lý do Day 6 gọi là consolidation.

Một incident thực tế:

```text
"Java app inaccessible."
```

Bạn không nhảy ngay vào Java.

Flow đúng có thể là:

```text
DNS?
 ↓
IP reachable?
 ↓
port listening?
 ↓
firewall?
 ↓
systemd service?
 ↓
process?
 ↓
journal?
 ↓
disk full?
 ↓
permission?
 ↓
config?
```

Kỹ năng Day 2–6:

```text
Day 2
navigation / files / text

Day 3
users / permissions / sudo / SSH

Day 4
process / systemd / packages / environment

Day 5
network / ports / filesystem / capacity

Day 6
logs / scripting / scheduling / backup / evidence
```

Từ đây bạn bắt đầu có tư duy administrator thay vì chỉ biết command.

---

# 75. L1 vs Escalation Boundary

Một Java fresher admin/L1 thường có thể làm trong approved scope:

```text
status check
read logs
capacity check
port check
process inspection
approved restart
approved backup execution
approved restore verification
collect evidence
confirm configuration
rerun failed approved job
```

Không nên tự ý:

```text
rewrite application code
change DB schema
change production firewall
delete production data
disable security controls
chmod 777
kill arbitrary process
edit unknown configuration
bypass change procedure
```

Nếu root cause nằm ngoài runbook hoặc privilege:

```text
preserve evidence
state what you know
state what you tested
state business impact
escalate
```

---

# 76. Supplementary — RPO và RTO

Không phải explicit Day 6 syllabus nhưng rất quan trọng.

### RPO — Recovery Point Objective

Bạn chấp nhận mất bao nhiêu dữ liệu?

Backup mỗi 24 giờ:

```text
potential loss ≈ gần 24h
```

tùy workload.

### RTO — Recovery Time Objective

Bạn cần restore trong bao lâu?

Một backup:

```text
tạo được trong 5 phút
```

nhưng:

```text
restore cần 7 giờ
```

có thể không đáp ứng operational requirement.

Do đó backup strategy không thể chỉ hỏi:

> “Backup có chạy không?”

Mà cần:

> “Restore được không, mất bao lâu, mất bao nhiêu dữ liệu?”

---

# 77. Supplementary — systemd timers

Modern systemd systems còn có:

```text
systemd timers
```

cho scheduling.

Nhưng Day 6 syllabus yêu cầu **cron**, vì vậy cron vẫn là nội dung bắt buộc hôm nay.

Bạn không được thay bài lab cron bằng timer rồi bỏ cron.

---

# 78. Exam Focus — Theory

Bạn cần trả lời được:

**1. `journalctl -u app.service` làm gì?**

Filter logs của unit.

**2. `journalctl -f` làm gì?**

Follow new journal events.

**3. `grep` exit 1 có luôn là error không?**

Không. Với GNU grep, `1` thường là no match; `2` là error. [gnu.org](https://www.gnu.org/s/grep/manual/html_node/Exit-Status.html?utm_source=chatgpt.com)

**4. `$?` là gì?**

Exit status command gần nhất.

**5. `pipefail` giải quyết vấn đề gì?**

Ngăn failure trong phần trước của pipeline bị che bởi success của command cuối.

**6. `*/5 * * * *` là gì?**

Mỗi 5 phút.

**7. User crontab có username field không?**

Không.

**8. Tại sao cron chạy fail dù manual chạy được?**

Environment/PATH/working directory/permission khác.

**9. `tar -tzf` làm gì?**

List archive contents.

**10. `rsync --delete` nguy hiểm ở đâu?**

Có thể xóa files ở destination không tồn tại source.

---

# 79. Exam Focus — Practical

Nếu đề yêu cầu:

> Create a scheduled health-check and backup job.

Bạn nên nghĩ theo checklist:

```text
inspect environment
↓
create controlled directories
↓
write script
↓
bash -n
↓
permissions
↓
manual run
↓
exit code
↓
verify logs
↓
verify backup
↓
restore test
↓
backup existing crontab
↓
schedule
↓
wait/verify execution
↓
inject safe failure
↓
collect evidence
↓
recover
↓
validate
```

---

# 80. Knowledge Check

Không xem đáp án trước nếu muốn tự kiểm tra.

### Q1

Khác nhau cơ bản giữa:

```bash
grep
awk
sed
```

là gì?

### Q2

Command:

```bash
grep -q ERROR app.log
echo $?
```

trả `1`.

Có nghĩa gì?

### Q3

Tại sao:

```bash
command1 | command2
```

có thể report success dù `command1` fail?

### Q4

`set -o pipefail` giải quyết vấn đề gì?

### Q5

Bạn muốn xem 100 log cuối của `myapp.service`. Command?

### Q6

Bạn muốn xem errors của boot hiện tại. Bạn sẽ kết hợp option nào?

### Q7

Dòng cron này chạy khi nào?

```text
30 1 * * * command
```

### Q8

Tại sao không nên sử dụng `--delete` với rsync ngay từ lần đầu?

### Q9

Có archive `.tar.gz` rồi thì tại sao vẫn cần restore test?

### Q10

Khi backup source mất, L1 nên thu thập evidence nào trước khi escalation?

---

# 81. Independent Practical Challenge

Sau khi hoàn thành guided lab, hãy tự làm mà **không nhìn script bên trên**.

Yêu cầu:

```text
Script: health_backup.sh
```

Phải:

1. Check `/` filesystem utilization.
2. Warning khi >= 75%.
3. Check một systemd service.
4. Backup một config directory.
5. Không để partial backup mang tên final.
6. Validate archive.
7. Log timestamp + severity.
8. `0` khi healthy.
9. Non-zero khi condition/error.
10. Schedule cron.
11. Capture cron evidence.
12. Inject một failure không làm hỏng OS.
13. Recover.
14. Demonstrate successful rerun.
15. Viết escalation note.

### Extra challenge

Cho phép:

```bash
./health_backup.sh health
```

và:

```bash
./health_backup.sh backup
```

và:

```bash
./health_backup.sh all
```

nhưng không duplicate logic.

---

# 82. Mastery Check cho Day 6

Tôi **chưa coi Day 6 là mastered chỉ vì bạn đã đọc bài này**.

Bạn đạt practical mastery khi có thể tự chứng minh:

```text
[ ] Tôi query được journal theo service/time/priority.
[ ] Tôi phân biệt được grep/awk/sed.
[ ] Tôi hiểu grep exit 1.
[ ] Tôi hiểu stdout/stderr.
[ ] Tôi hiểu $? và custom exit code.
[ ] Tôi hiểu pipe và pipefail.
[ ] Tôi tự viết được Bash script có validation.
[ ] Tôi schedule được cron.
[ ] Tôi troubleshoot được cron environment.
[ ] Tôi tạo được tar backup.
[ ] Tôi verify + restore được backup.
[ ] Tôi dùng rsync dry-run trước operation rủi ro.
[ ] Tôi inject failure an toàn.
[ ] Tôi tìm root cause bằng evidence.
[ ] Tôi recover và revalidate.
[ ] Tôi viết được escalation evidence.
```

---

# 83. Phần quan trọng nhất cần ghi nhớ

Day 6 có thể cô đọng thành công thức:

```text
OBSERVE
journalctl / systemctl / df / logs

        ↓

FILTER
grep / awk / sed

        ↓

AUTOMATE
Bash

        ↓

COMMUNICATE STATE
exit codes + stdout/stderr + logs

        ↓

SCHEDULE
cron

        ↓

PROTECT
tar / rsync + verification + restore

        ↓

OPERATE
failure injection → evidence → root cause
→ remediation / rollback / escalation
```

Nếu sau hôm nay bạn chỉ nhớ cú pháp:

```bash
grep -i
tar -czf
crontab -e
```

thì chưa đủ.

Năng lực cần đạt là:

> **Nhìn thấy symptom → thu hẹp scope → lấy evidence → tự động hóa check → trả trạng thái đúng → backup/restore an toàn → phát hiện failure → xác minh root cause → recover hoặc escalation có evidence.**

Đó chính xác là nền tảng Linux operations mà bạn sẽ mang sang **Day 7 AWS**, rồi sau đó dùng lại cho EC2, Tomcat/Java, PostgreSQL và monitoring. [man7](https://man7.org/linux/man-pages/man1/journalctl.1%40%40systemd.html?utm_source=chatgpt.com)
