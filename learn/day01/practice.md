# Day 1 — Module Thực hành

## Infrastructure Operations + Linux/RHEL Administration Foundation Lab

Module này biến toàn bộ lý thuyết Day 1 thành một **controlled Infrastructure Operations lab**. Theo syllabus, bạn phải chuẩn bị administrative workstation, approved access checklist, Linux/RHEL lab environment, evidence repository và personal learning baseline; sau đó thực hành CLI navigation, file operations, searching/text processing, users/groups, ownership/permissions, least-privilege `sudo`, SSH keys, authorized/denied access; cuối cùng phải nộp system inventory, access verification và configuration evidence. Day 1 còn bắt buộc tạo shared evidence manifest, hash một source artifact, sửa working copy, phát hiện hash mismatch mà không thay thế original. Infrastructure_Operations_14_Da…

Syllabus không quy định chính xác tên VM, username, directory hay ticket ID phải dùng. Vì vậy các tên như `rhel-lab01`, `labalice`, `LAB-D01-001` bên dưới là **lab design bổ sung** để triển khai đúng yêu cầu syllabus, không phải naming convention bắt buộc của tài liệu nguồn.

---

# 1. Mục tiêu của Module Thực hành

Kết thúc Module Thực hành Day 1, bạn phải tự thực hiện được workflow sau trong **authorized non-production environment**:

```text
Confirm authorization
        ↓
Confirm correct host/environment
        ↓
Collect baseline
        ↓
Perform read-only inspection
        ↓
Perform controlled configuration
        ↓
Verify expected access
        ↓
Verify denied access
        ↓
Troubleshoot bounded failures
        ↓
Rollback/cleanup
        ↓
Collect evidence
        ↓
Hash/preserve evidence
        ↓
Document result and gaps
```

Điểm đánh giá không chỉ là:

> "Command có chạy được hay không?"

Mà phải là:

> "Bạn có xác định đúng target, đúng authorization, thực hiện minimum required change, verify positive/negative case, giữ được evidence và biết rollback/escalate hay không?"

---

# 2. Lab topology đề xuất

Một topology tối thiểu:

```text
┌───────────────────────────────┐
│ Administrative Workstation    │
│                               │
│ SSH client                    │
│ Terminal                      │
│ Text editor                   │
│ Evidence repository           │
│ SSH private key               │
└───────────────┬───────────────┘
                │
                │ SSH
                │
                ▼
┌───────────────────────────────┐
│ rhel-lab01                    │
│ Environment: LAB             │
│ Linux/RHEL VM                │
│                               │
│ learner/admin account         │
│ labalice                      │
│ labbob                        │
│ day1ops group                 │
│ day1sudo group                │
└───────────────────────────────┘
```

Nếu bạn chỉ có một Linux VM và làm trực tiếp trên console thì vẫn có thể hoàn thành hầu hết bài tập. Tuy nhiên phần SSH authentication cần ít nhất một SSH client và một SSH server reachable.

Syllabus yêu cầu instructor chuẩn bị và validate lab access trước delivery, bao gồm RHEL VM; Day 1 cũng là thời điểm review learner prerequisites. Infrastructure_Operations_14_Da…

---

# 3. Naming dùng trong bài lab

Tôi sẽ dùng các giá trị mẫu sau:

```text
Server:
rhel-lab01

Environment:
LAB

Ticket:
LAB-D01-001

Lab users:
labalice
labbob

Lab groups:
day1ops
day1sudo

Shared directory:
/srv/day1-share

Sudo test file:
/var/tmp/day1-admin-check

Evidence directory:
~/infra-ops-lab/Day01
```

Bạn có thể thay chúng theo environment thật.

Không dùng:

- production username thật;
- production password;
- private key thật;
- access token thật;
- confidential production data.

---

# 4. Command classification trong lab

Từ đây tôi sẽ phân biệt:

### `[READ-ONLY]`

Command chỉ inspect state hoặc đọc data.

Ví dụ:

```bash
hostname
id
ls -l
stat file
```

### `[CHANGE]`

Command thay đổi system state.

Ví dụ:

```bash
useradd
chmod
chown
```

### `[DESTRUCTIVE]`

Command có thể xóa data hoặc object.

Ví dụ:

```bash
rm
userdel
```

### `[PRIVILEGED]`

Cần elevated privilege.

Ví dụ:

```bash
sudo useradd ...
```

Một command có thể thuộc nhiều category.

Ví dụ:

```bash
sudo userdel -r labalice
```

là:

```text
CHANGE
+
PRIVILEGED
+
DESTRUCTIVE
```

---

# 5. Safety Rules trước khi bắt đầu

Toàn bộ bài lab này mặc định:

```text
LAB / sandbox
NON-PRODUCTION
AUTHORIZED
```

Trước bất kỳ privileged/configuration change nào, phải trả lời được:

```text
1. Tôi đang ở host nào?
2. Environment nào?
3. Tôi đang login bằng account nào?
4. Ticket/authorization nào cho phép thao tác?
5. Object nào sẽ bị thay đổi?
6. Expected result là gì?
7. Verification là gì?
8. Rollback là gì?
```

Một pre-flight check tối thiểu:

```bash
hostname
whoami
id
pwd
date --iso-8601=seconds
```

Expected conceptual result:

```text
hostname → rhel-lab01
user     → authorized learner/admin
pwd      → expected directory
time     → correct/reasonable timestamp
```

**STOP ngay** nếu hostname/environment không đúng.

---

# LAB PHASE 1 — Administrative Workstation Preparation

## 6. Objective

Chuẩn bị workstation dùng để thực hiện lab và lưu evidence.

Syllabus yêu cầu rõ:

> Prepare the administrative workstation. Infrastructure_Operations_14_Da…

Tài liệu nguồn không quy định phải dùng Windows, Linux hay macOS workstation; điều bắt buộc về mặt outcome là bạn phải có administrative environment đủ để thực hiện lab.

---

# 7. Workstation readiness checklist

Bạn cần có tối thiểu:

```text
Terminal
SSH client
Text editor
Ability to create files/directories
Evidence storage
Clock/timezone visible
Network route to lab server
Approved lab account
```

Optional nhưng hữu ích:

```text
sha256sum or equivalent SHA-256 tool
script session recording utility
secure file transfer utility
```

---

# 8. Tạo working directory

Trên Linux/macOS workstation hoặc Linux administrative VM:

### `[CHANGE]`

```bash
mkdir -p "$HOME/infra-ops-lab/Day01"
cd "$HOME/infra-ops-lab/Day01"
```

Verify:

### `[READ-ONLY]`

```bash
pwd
```

Expected:

```text
/home/<user>/infra-ops-lab/Day01
```

hoặc home path tương ứng.

---

# 9. Kiểm tra SSH client

### `[READ-ONLY]`

```bash
ssh -V
```

Ví dụ output có thể giống:

```text
OpenSSH_...
```

Version thực tế phụ thuộc workstation.

Không cần output giống chính xác ví dụ.

---

# 10. Evidence của Phase 1

Nên lưu:

```text
D01-EVID-001
Administrative workstation readiness

Contains:
- workstation OS
- current user
- SSH client availability
- evidence directory
- timestamp
```

Không cần gửi private workstation information không liên quan.

---

# LAB PHASE 2 — Approved Access Checklist

## 11. Objective

Chứng minh bạn biết **mình đang được phép quản trị cái gì** trước khi bắt đầu configuration.

Đây là practical translation của:

```text
operational safety
change authorization
least privilege
production/non-production separation
```

---

# 12. Access checklist mẫu

Trước lab, tạo một record như:

| Field         | Example                                   |
| ------------- | ----------------------------------------- |
| Ticket        | `LAB-D01-001`                             |
| Environment   | `LAB`                                     |
| Target        | `rhel-lab01`                              |
| Asset ID      | `LAB-LNX-001`                             |
| Learner       | `<approved-id>`                           |
| Access type   | SSH / Console                             |
| Privilege     | Normal account + limited admin            |
| Permitted     | Day 1 Linux lab tasks                     |
| Not permitted | Production access, destructive OS changes |
| Start         | Timestamp                                 |
| End           | Timestamp                                 |
| Escalation    | Instructor/Mentor                         |

Đây là template bổ sung; syllabus yêu cầu approved access checklist nhưng không định nghĩa exact columns.

---

# 13. Access pre-check

Sau khi login:

### `[READ-ONLY]`

```bash
hostname
whoami
id
pwd
date --iso-8601=seconds
```

Capture output.

Nếu expected hostname:

```text
rhel-lab01
```

nhưng nhận:

```text
rhel-prod01
```

thì:

```text
DO NOT CONTINUE
```

Không chạy bất cứ `[CHANGE]` command nào.

---

# LAB PHASE 3 — Evidence Repository

Syllabus yêu cầu tạo evidence repository và shared evidence manifest dùng xuyên 14 ngày. Infrastructure_Operations_14_Da…

## 14. Directory structure

Đây là implementation đề xuất:

```text
Day01/
├── raw/
├── working/
├── deliverable/
├── screenshots/
├── transcripts/
├── configuration/
├── inventory/
└── manifest/
```

Tạo:

### `[CHANGE]`

```bash
mkdir -p raw working deliverable screenshots transcripts configuration inventory manifest
```

Verify:

```bash
find . -maxdepth 1 -type d -print
```

Expected có các directories trên.

---

# 15. Tại sao tách `raw` và `working`?

Không bao giờ analysis trực tiếp trên original nếu có thể tránh được.

Workflow:

```text
Source
   ↓
raw/
   ↓ copy
working/
   ↓ process/redact
deliverable/
```

`raw/`:

```text
untouched original
```

`working/`:

```text
copy used for analysis
```

`deliverable/`:

```text
reviewer-ready derived output
```

Syllabus yêu cầu preserve untouched raw export, xử lý trên copy, và derived/redacted copy phải có traceability riêng. Infrastructure_Operations_14_Da…

---

# LAB PHASE 4 — Start a Command/Activity Record

## 16. Objective

Syllabus yêu cầu:

> document executed commands. Infrastructure_Operations_14_Da…

Không nhất thiết phải dùng một tool cụ thể.

Có hai phương pháp.

---

## 17. Method A — Manual command log

Tạo:

```text
transcripts/day1_commands.txt
```

Ghi mỗi operation theo mẫu:

```text
Timestamp:
Host:
User:
Ticket:
Purpose:
Command:
Expected:
Actual:
Result:
```

Đây là cách portable nhất.

---

# 18. Method B — Terminal transcript

Nếu `script` utility có sẵn và lab policy cho phép:

### `[CHANGE – workstation evidence only]`

```bash
script -q -f transcripts/day1_terminal_session.txt
```

Khi kết thúc:

```bash
exit
```

**Cảnh báo:** terminal transcript có thể capture sensitive information.

Không nhập:

```text
real password
private key
token
secret
```

vào transcript.

Nếu có secret xuất hiện, phải xử lý theo security policy; không đơn giản chỉnh raw transcript rồi gọi đó là original.

---

# LAB PHASE 5 — Personal Learning Baseline

Syllabus yêu cầu `personal learning baseline`, nhưng không chỉ định exact format. Vì vậy phần này là format đề xuất, không phải requirement chi tiết từ source.

## 19. Baseline trước lab

Tự đánh giá:

```text
0 = chưa biết
1 = biết khái niệm
2 = làm được có hướng dẫn
3 = làm độc lập trong lab
```

Các skill:

| Skill                   | Before |
| ----------------------- | -----: |
| `pwd`, `cd`, `ls`       |        |
| absolute/relative paths |        |
| `cp`, `mv`, `rm`        |        |
| `find`, `grep`          |        |
| users/groups            |        |
| permissions             |        |
| `chmod`                 |        |
| `chown`                 |        |
| `umask`                 |        |
| `sudo`                  |        |
| SSH key                 |        |
| evidence hashing        |        |

Sau Day 1 tự đánh giá lại.

Mục tiêu không phải tự cho điểm đẹp mà phát hiện gap.

---

# LAB PHASE 6 — Linux/RHEL System Inventory

## 20. Objective

Thu thập một minimum system baseline trước configuration.

Deliverable này đáp ứng yêu cầu:

> submit system inventory. Infrastructure_Operations_14_Da…

---

# 21. Host identity

### `[READ-ONLY]`

```bash
hostname
```

Expected:

```text
rhel-lab01
```

hoặc hostname lab thực tế.

---

# 22. Operating system

### `[READ-ONLY]`

```bash
cat /etc/os-release
```

Fields thường có:

```text
NAME=
VERSION=
ID=
VERSION_ID=
```

Không assume output phải giống máy khác.

---

# 23. Kernel release

```bash
uname -r
```

Architecture:

```bash
uname -m
```

General system info:

```bash
uname -a
```

Nhớ:

```text
OS distribution version
≠
kernel version
```

---

# 24. Identity

```bash
whoami
id
```

Shell/account entry:

```bash
getent passwd "$USER"
```

---

# 25. Current environment context

```bash
pwd
umask
date --iso-8601=seconds
```

---

# 26. Thu thập inventory thành evidence file

Một approach đơn giản:

```bash
{
  echo "Ticket: LAB-D01-001"
  echo "Collected: $(date --iso-8601=seconds)"
  echo "Hostname: $(hostname)"
  echo "Current user: $(whoami)"
  echo
  echo "=== OS ==="
  cat /etc/os-release
  echo
  echo "=== Kernel ==="
  uname -a
  echo
  echo "=== Identity ==="
  id
  echo
  echo "=== Umask ==="
  umask
} > inventory/system_inventory.txt
```

Đây là `[CHANGE]` đối với evidence repository, nhưng **read-only đối với server configuration**.

Inspect:

```bash
cat inventory/system_inventory.txt
```

---

# 27. Verify inventory quality

Checklist:

```text
Hostname present?
OS present?
Kernel present?
User identity present?
Timestamp present?
Ticket present?
No secret?
```

Nếu thiếu field:

```text
do not silently assume it
```

bổ sung hoặc record gap.

---

# LAB PHASE 7 — CLI Navigation

## 28. Objective

Thành thạo:

```text
pwd
ls
cd
absolute paths
relative paths
.
..
~
```

---

# 29. Tạo navigation sandbox

```bash
mkdir -p "$HOME/day1-cli/navigation/level1/level2"
cd "$HOME/day1-cli"
```

Verify:

```bash
pwd
```

---

# 30. Absolute path navigation

```bash
cd "$HOME/day1-cli/navigation/level1"
pwd
```

Expected:

```text
/home/<user>/day1-cli/navigation/level1
```

---

# 31. Relative navigation

```bash
cd level2
pwd
```

Then:

```bash
cd ..
pwd
```

Then:

```bash
cd ./level2
pwd
```

Then home:

```bash
cd ~
pwd
```

---

# 32. Required understanding

Bạn phải giải thích được:

```text
/       absolute root
.       current directory
..      parent directory
~       current user's home
```

---

# 33. Failure checkpoint

Nếu:

```bash
cd level2
```

trả về:

```text
No such file or directory
```

không tạo directory ngay.

Trước tiên:

```bash
pwd
ls -la
```

Có thể bạn đang sai current directory.

Đây là troubleshooting mindset:

```text
verify context
before modifying context
```

---

# LAB PHASE 8 — File and Directory Operations

## 34. Objective

Thực hành đầy đủ syllabus:

```text
file creation
copying
moving
deletion
```

trong dedicated sandbox.

---

# 35. Create directories

```bash
cd "$HOME/day1-cli"
mkdir files
cd files
```

Verify:

```bash
pwd
```

---

# 36. Create files

```bash
touch report01.txt
touch report02.txt
```

Verify:

```bash
ls -l
```

---

# 37. Write controlled test content

```bash
printf 'Day 1 Infrastructure Operations Lab\n' > report01.txt
printf 'Linux permissions test\n' > report02.txt
```

Inspect:

```bash
cat report01.txt
cat report02.txt
```

---

# 38. Copy

```bash
cp report01.txt report01.copy.txt
```

Verify:

```bash
ls -l report01.txt report01.copy.txt
```

Content verification:

```bash
cmp report01.txt report01.copy.txt
```

Nếu giống nhau, `cmp` thường không in difference.

Một verification khác:

```bash
sha256sum report01.txt report01.copy.txt
```

Expected:

```text
same SHA-256
```

vì bytes giống nhau.

---

# 39. Move/Rename

```bash
mv report02.txt permissions-note.txt
```

Verify:

```bash
ls -l
```

Expected:

```text
permissions-note.txt exists
report02.txt absent
```

---

# 40. Move vào directory khác

```bash
mkdir archive
mv report01.copy.txt archive/
```

Verify:

```bash
ls -l archive/
```

---

# 41. Safe deletion

Chỉ xóa disposable lab object.

```bash
touch delete-me.txt
ls -l delete-me.txt
rm -i delete-me.txt
```

`rm -i` yêu cầu confirmation trên nhiều implementations.

Verify absence:

```bash
ls -l delete-me.txt
```

Expected:

```text
No such file or directory
```

Đây là **expected negative result**, không phải lab failure.

---

# 42. Directory removal

```bash
mkdir empty-dir
rmdir empty-dir
```

Verify:

```bash
ls -ld empty-dir
```

Expected absent.

---

# 43. Destructive safety lesson

Trước:

```bash
rm some-relative-path
```

nên biết:

```bash
pwd
ls -l some-relative-path
```

Không sử dụng:

```bash
rm -rf ...
```

chỉ vì tiện.

Day 1 không cần destructive recursive deletion để đạt syllabus objective.

---

# LAB PHASE 9 — Command Syntax

## 44. Objective

Phân tích command thực tế.

Ví dụ:

```bash
ls -la "$HOME/day1-cli/files"
```

Phải chỉ ra:

```text
Command:
ls

Options:
-l
-a

Argument:
/home/.../day1-cli/files
```

---

# 45. Help verification

```bash
ls --help
```

hoặc:

```bash
man ls
```

Bài thực hành:

Tự tìm:

```text
option nào human-readable file sizes?
option nào reverse sort?
```

Mục tiêu không phải học thuộc tất cả flags mà biết self-service documentation.

---

# LAB PHASE 10 — Pipes và Redirection

## 46. Tạo fixture

```bash
cat > "$HOME/day1-cli/files/events.txt" <<'EOF'
INFO service started
ERROR authentication failed
INFO health check passed
WARNING disk usage elevated
ERROR permission denied
INFO service stopped
EOF
```

Đây là synthetic lab data.

---

# 47. Pipe

```bash
grep ERROR "$HOME/day1-cli/files/events.txt" | wc -l
```

Expected:

```text
2
```

Flow:

```text
grep ERROR
    ↓
2 matching lines
    ↓
wc -l
    ↓
2
```

---

# 48. Redirect stdout

```bash
grep ERROR "$HOME/day1-cli/files/events.txt" > "$HOME/day1-cli/files/errors.txt"
```

Verify:

```bash
cat "$HOME/day1-cli/files/errors.txt"
```

Expected:

```text
ERROR authentication failed
ERROR permission denied
```

---

# 49. Demonstrate overwrite safely

```bash
printf 'first\n' > redirect-demo.txt
cat redirect-demo.txt
```

Then:

```bash
printf 'second\n' > redirect-demo.txt
cat redirect-demo.txt
```

Expected final file:

```text
second
```

Lesson:

```text
> overwrites/truncates target
```

---

# 50. Append

```bash
printf 'third\n' >> redirect-demo.txt
cat redirect-demo.txt
```

Expected:

```text
second
third
```

---

# 51. stderr

Tạo expected failure:

```bash
ls /definitely/not/a/real/day1/path 2> error-output.txt
```

Inspect:

```bash
cat error-output.txt
```

Đây là một safe negative test.

---

# 52. stdout + stderr

```bash
{
  echo "normal output"
  ls /definitely/not/a/real/day1/path
} > combined-output.txt 2>&1
```

Inspect:

```bash
cat combined-output.txt
```

Bạn phải nhận biết normal output và diagnostic output đã được merge.

---

# LAB PHASE 11 — Command History và Help

## 53. History

```bash
history
```

Tìm một số commands gần nhất.

Có thể export một bounded snapshot:

```bash
history > "$HOME/infra-ops-lab/Day01/transcripts/history_snapshot.txt"
```

Nhưng **không coi history là authoritative audit log**.

Trước khi submit, inspect:

```bash
less "$HOME/infra-ops-lab/Day01/transcripts/history_snapshot.txt"
```

Nếu có secrets, không gửi raw evidence bừa bãi.

---

# 54. Help exercise

Chạy:

```bash
man chmod
```

Tìm:

```text
symbolic mode
recursive option
reference option
```

Sau đó:

```bash
chmod --help
```

So sánh:

```text
man page → detailed documentation
--help   → quick syntax/reference
```

---

# LAB PHASE 12 — Searching and Text Processing

Syllabus yêu cầu trực tiếp `searching and text-processing exercises`. Infrastructure_Operations_14_Da…

Day 1 nên tập trung foundation; advanced `awk`, `sed` và regular expressions thuộc Day 2.

---

# 55. `find`

Tạo data:

```bash
mkdir -p "$HOME/day1-search/a" "$HOME/day1-search/b"

touch "$HOME/day1-search/a/report.txt"
touch "$HOME/day1-search/b/report.log"
touch "$HOME/day1-search/b/notes.txt"
```

Search:

```bash
find "$HOME/day1-search" -type f -name '*.txt'
```

Expected:

```text
.../a/report.txt
.../b/notes.txt
```

---

# 56. `grep`

Fixture:

```bash
cat > "$HOME/day1-search/access.log" <<'EOF'
alice ALLOWED server01
bob DENIED server01
alice ALLOWED server02
charlie DENIED server03
EOF
```

Search denied:

```bash
grep 'DENIED' "$HOME/day1-search/access.log"
```

Expected 2 lines.

Case count:

```bash
grep 'DENIED' "$HOME/day1-search/access.log" | wc -l
```

Expected:

```text
2
```

---

# 57. `head`

```bash
head "$HOME/day1-search/access.log"
```

Với small file sẽ hiển thị toàn bộ hoặc phần đầu.

---

# 58. `tail`

```bash
tail -n 2 "$HOME/day1-search/access.log"
```

Expected:

```text
alice ALLOWED server02
charlie DENIED server03
```

---

# 59. `sort`

```bash
sort "$HOME/day1-search/access.log"
```

Dùng để quan sát deterministic ordering.

---

# 60. `uniq`

`uniq` thường hoạt động trên adjacent duplicate lines.

Fixture:

```bash
printf '%s\n' alice alice bob bob bob charlie > names.txt
```

Then:

```bash
uniq names.txt
```

Expected:

```text
alice
bob
charlie
```

Count:

```bash
uniq -c names.txt
```

---

# 61. `wc`

```bash
wc -l "$HOME/day1-search/access.log"
```

Expected:

```text
4 ...
```

Day 1 evidence có thể dùng `wc -l` làm record count **chỉ khi bạn đã định nghĩa sample này là one-record-per-line**.

Không được generalize rằng:

```text
line count = record count
```

cho mọi file format.

---

# LAB PHASE 13 — Users and Groups

Đây là phần configuration-changing đầu tiên có ý nghĩa lớn hơn.

## 62. Objective

Tạo controlled lab identities và groups.

Syllabus yêu cầu:

> create and manage controlled users and groups. Infrastructure_Operations_14_Da…

---

# 63. Safety pre-check

Trước khi tạo:

```bash
getent passwd labalice
getent passwd labbob
getent group day1ops
getent group day1sudo
```

Nếu object đã tồn tại:

```text
STOP
```

Không assume đó là object của lab này.

Dùng unique names khác.

---

# 64. Create group

### `[CHANGE][PRIVILEGED]`

```bash
sudo groupadd day1ops
```

Verify:

```bash
getent group day1ops
```

---

# 65. Create lab users

```bash
sudo useradd -m labalice
sudo useradd -m labbob
```

Verify:

```bash
getent passwd labalice
getent passwd labbob
```

Expected account entries với home directories.

---

# 66. Inspect identities

```bash
id labalice
id labbob
```

Record:

```text
UID
primary GID
supplementary groups
```

---

# 67. Add `labalice` to `day1ops`

Use append mode:

```bash
sudo usermod -aG day1ops labalice
```

Verify:

```bash
id labalice
```

Expected supplementary group includes:

```text
day1ops
```

### Tại sao phải có `-a`?

`usermod -G` mà thiếu append semantics có thể thay thế supplementary group set theo implementation/tool behavior và gây unintended access loss.

Trong operational work:

> Group management là security change.

---

# 68. Current session caveat

Nếu `labalice` đang login từ trước khi group membership thay đổi, process/session hiện tại có thể chưa nhận effective supplementary group mới.

Safe verification:

```text
new login session
```

hoặc instructor-approved method.

Đừng conclude configuration thất bại chỉ vì old process context chưa refresh.

---

# LAB PHASE 14 — Ownership and Permissions

## 69. Objective

Thiết kế một shared directory:

```text
labalice → allowed
labbob   → denied
```

theo least privilege.

---

# 70. Create controlled directory

### `[CHANGE][PRIVILEGED]`

```bash
sudo mkdir -p /srv/day1-share
```

Set ownership:

```bash
sudo chown root:day1ops /srv/day1-share
```

Set permissions:

```bash
sudo chmod 0770 /srv/day1-share
```

Verify:

```bash
ls -ld /srv/day1-share
```

Expected conceptual:

```text
drwxrwx--- root day1ops ...
```

Exact display fields vary.

---

# 71. Create shared test file

```bash
echo 'Day 1 controlled access test' | sudo tee /srv/day1-share/team.txt >/dev/null
```

Set owner/group:

```bash
sudo chown root:day1ops /srv/day1-share/team.txt
```

Set mode:

```bash
sudo chmod 0660 /srv/day1-share/team.txt
```

Verify:

```bash
ls -l /srv/day1-share/team.txt
```

Expected:

```text
-rw-rw---- root day1ops ...
```

---

# 72. `stat` verification

```bash
stat /srv/day1-share/team.txt
```

Portable concise output on GNU/Linux:

```bash
stat -c '%U %G %A %a %n' /srv/day1-share/team.txt
```

Expected conceptual:

```text
root day1ops -rw-rw---- 660 /srv/day1-share/team.txt
```

---

# 73. Positive access test

Nếu learner admin được authorized dùng `sudo -u` trong lab:

```bash
sudo -u labalice -- cat /srv/day1-share/team.txt
```

Expected:

```text
Day 1 controlled access test
```

Đây chứng minh `labalice` có effective read access.

---

# 74. Negative access test

```bash
sudo -u labbob -- cat /srv/day1-share/team.txt
```

Expected:

```text
Permission denied
```

Đây là **PASS**, vì expected outcome là denied access.

---

# 75. Tại sao negative test bắt buộc?

Nếu chỉ chứng minh:

```text
labalice succeeds
```

thì configuration có thể là:

```text
0666
```

và mọi user cũng đọc được.

Positive test chứng minh:

```text
required access works
```

Negative test chứng minh:

```text
unrequired access remains blocked
```

Least privilege cần cả hai.

---

# 76. Directory permissions test

Kiểm tra:

```bash
ls -ld /srv/day1-share
ls -l /srv/day1-share
```

Nếu file `0660` đúng nhưng user vẫn bị denied, inspect parent:

```bash
ls -ld /srv
ls -ld /srv/day1-share
```

Directory traverse permission là một dependency quan trọng.

---

# LAB PHASE 15 — `chmod`, `chown`, `chgrp`

## 77. `chmod` symbolic exercise

Tạo file user-owned:

```bash
touch "$HOME/day1-permission-demo.txt"
```

Check:

```bash
ls -l "$HOME/day1-permission-demo.txt"
```

Add user execute:

```bash
chmod u+x "$HOME/day1-permission-demo.txt"
```

Verify:

```bash
ls -l "$HOME/day1-permission-demo.txt"
```

Sau đó remove:

```bash
chmod u-x "$HOME/day1-permission-demo.txt"
```

---

# 78. Numeric `chmod`

```bash
chmod 640 "$HOME/day1-permission-demo.txt"
```

Verify:

```bash
stat -c '%A %a %n' "$HOME/day1-permission-demo.txt"
```

Expected:

```text
-rw-r----- 640 ...
```

---

# 79. `chown`

Trong lab controlled file:

```bash
sudo touch /var/tmp/day1-owner-test
sudo chown labalice:day1ops /var/tmp/day1-owner-test
```

Verify:

```bash
stat -c '%U %G %A %n' /var/tmp/day1-owner-test
```

---

# 80. `chgrp`

```bash
sudo chgrp day1ops /var/tmp/day1-owner-test
```

Verify:

```bash
stat -c '%G %n' /var/tmp/day1-owner-test
```

---

# 81. Capture before/after

Before:

```bash
stat -c '%U %G %A %a %n' file
```

Change:

```bash
chmod ...
```

After:

```bash
stat -c '%U %G %A %a %n' file
```

Evidence phải có:

```text
before
change
after
```

không chỉ after-state.

---

# LAB PHASE 16 — `umask`

## 82. Objective

Thấy trực tiếp ảnh hưởng của `umask` tới default permission.

---

# 83. Record original

```bash
original_umask=$(umask)
printf 'Original umask: %s\n' "$original_umask"
```

---

# 84. Set temporary lab `umask`

Trong shell hiện tại:

```bash
umask 027
```

Verify:

```bash
umask
```

Expected:

```text
0027
```

hoặc display tương đương.

---

# 85. Create file và directory

```bash
mkdir -p "$HOME/day1-umask-test"
cd "$HOME/day1-umask-test"

touch testfile
mkdir testdir
```

Inspect:

```bash
stat -c '%A %a %n' testfile testdir
```

Typical result with standard GNU tools:

```text
file      640
directory 750
```

Tuy nhiên application/process có thể request stricter mode; đừng coi đây là universal arithmetic rule cho mọi application.

---

# 86. Restore original `umask`

```bash
umask "$original_umask"
```

Verify:

```bash
umask
```

Expected bằng giá trị đã record.

Đây là một mini rollback.

---

# LAB PHASE 17 — Least-Privilege `sudo`

Đây là một phần quan trọng nhất Day 1.

## 87. Objective

Tạo một account có thể thực hiện **một administrative task được chỉ định**, nhưng không có unrestricted root access.

Syllabus yêu cầu:

> establish least-privilege sudo access. Infrastructure_Operations_14_Da…

---

# 88. Lab design

Ta sẽ tạo một protected file:

```text
/var/tmp/day1-admin-check
```

Normal user không được đọc.

Nhưng member của `day1sudo` được dùng:

```bash
sudo cat /var/tmp/day1-admin-check
```

Không được:

```bash
sudo id
```

hoặc arbitrary root command.

Đây là lab demonstration, không phải production sudo policy recommendation.

---

# 89. Create sudo group

Pre-check:

```bash
getent group day1sudo
```

Nếu absent:

```bash
sudo groupadd day1sudo
```

Add learner lab user:

```bash
sudo usermod -aG day1sudo labalice
```

Verify:

```bash
id labalice
```

---

# 90. Create protected target

```bash
echo 'AUTHORIZED-SUDO-TEST' | sudo tee /var/tmp/day1-admin-check >/dev/null
sudo chown root:root /var/tmp/day1-admin-check
sudo chmod 0600 /var/tmp/day1-admin-check
```

Verify:

```bash
stat -c '%U %G %A %a %n' /var/tmp/day1-admin-check
```

Expected:

```text
root root -rw------- 600 ...
```

---

# 91. Verify normal access is denied

As `labalice` context:

```bash
sudo -u labalice -- cat /var/tmp/day1-admin-check
```

Expected:

```text
Permission denied
```

Đây là pre-change evidence.

---

# 92. Determine exact `cat` path

Sudoers rules use absolute executable path.

Check:

```bash
command -v cat
```

On many RHEL systems:

```text
/usr/bin/cat
```

nhưng hãy dùng actual output.

---

# 93. Create controlled sudoers rule

**Không chỉnh `/etc/sudoers` bằng text editor thông thường trong bài lab.**

Use:

```bash
sudo visudo -f /etc/sudoers.d/day1-lab
```

Example content, assuming `command -v cat` returned `/usr/bin/cat`:

```text
%day1sudo ALL=(root) /usr/bin/cat /var/tmp/day1-admin-check
```

Save.

---

# 94. Syntax validation

```bash
sudo visudo -c
```

Expected:

```text
parsed OK
```

Exact output có thể khác.

Nếu syntax validation fail:

```text
STOP
```

Không tiếp tục test như thể configuration hợp lệ.

---

# 95. `sudo -l`

Login as a fresh `labalice` session hoặc authorized method, rồi:

```bash
sudo -l
```

Expected rule chỉ ra specific permitted command.

Bạn không nên thấy:

```text
(ALL) ALL
```

trừ khi account còn privilege khác được cấp bởi environment.

Nếu có privilege rộng hơn, lab không còn chứng minh được isolated least-privilege condition; phải record limitation.

---

# 96. Positive sudo test

As `labalice`:

```bash
sudo /usr/bin/cat /var/tmp/day1-admin-check
```

Expected:

```text
AUTHORIZED-SUDO-TEST
```

---

# 97. Negative sudo test

```bash
sudo /usr/bin/id
```

Expected:

```text
not allowed
```

hoặc sudo denial tương ứng.

Nếu command thành công, **lab failed least-privilege acceptance** vì `labalice` có privilege rộng hơn thiết kế.

Bạn phải investigate source của additional sudo entitlement.

Không đơn giản ghi:

> "Pass because everything works."

---

# 98. `sudo` evidence

Capture:

```text
Before:
normal read denied

Configuration:
sudoers rule

Verification:
sudo -l

Positive:
approved command succeeds

Negative:
unapproved command denied
```

Đây là một evidence package rất mạnh.

---

# 99. Sudo rollback

Remove dedicated rule:

```bash
sudo rm /etc/sudoers.d/day1-lab
```

Then validate:

```bash
sudo visudo -c
```

Nhưng **chỉ cleanup sau khi lab/evidence đã hoàn thành**, hoặc khi procedure yêu cầu rollback.

Group membership cleanup có thể làm cuối bài.

---

# LAB PHASE 18 — SSH Key Generation

## 100. Objective

Tạo SSH key pair, bảo vệ private key và sử dụng public key để authenticate tới lab account.

---

# 101. Workstation key directory

Trên administrative workstation:

```bash
mkdir -p "$HOME/.ssh"
chmod 700 "$HOME/.ssh"
```

---

# 102. Generate dedicated lab key

```bash
ssh-keygen -t ed25519 -f "$HOME/.ssh/day1_lab_ed25519" -C "day1-lab"
```

Bạn sẽ có:

```text
~/.ssh/day1_lab_ed25519
~/.ssh/day1_lab_ed25519.pub
```

Private:

```text
day1_lab_ed25519
```

Public:

```text
day1_lab_ed25519.pub
```

Nếu organization/FIPS/environment không hỗ trợ algorithm này, phải dùng algorithm approved cho environment. Không tự thay security policy.

---

# 103. Inspect files

```bash
ls -l "$HOME/.ssh/day1_lab_ed25519"*
```

Không:

```bash
cat ~/.ssh/day1_lab_ed25519
```

rồi paste output vào evidence.

Private key content **không bao giờ là lab deliverable**.

---

# 104. Fingerprint

Public key fingerprint:

```bash
ssh-keygen -lf "$HOME/.ssh/day1_lab_ed25519.pub"
```

Fingerprint có thể lưu làm evidence.

Public key fingerprint:

```text
not secret
```

Private key:

```text
secret credential material
```

---

# LAB PHASE 19 — Configure Authorized Key

Có nhiều cách. Syllabus chỉ yêu cầu generate/configure SSH keys, không bắt buộc method cụ thể.

## 105. Preferred lab method nếu password bootstrap được instructor approve

```bash
ssh-copy-id -i "$HOME/.ssh/day1_lab_ed25519.pub" labalice@rhel-lab01
```

Lần bootstrap có thể yêu cầu temporary lab password nếu server cho phép.

Không đưa password vào transcript/evidence.

---

# 106. Alternative server-side method

Nếu instructor/admin cấu hình key server-side, trên server:

```bash
sudo install -d -m 700 -o labalice -g labalice /home/labalice/.ssh
```

Sau đó chỉ install **public key**.

Không copy private key lên server.

`authorized_keys` phải thuộc `labalice` và có permission phù hợp với server policy.

Ví dụ lab fresh account:

```bash
sudo install -m 600 -o labalice -g labalice \
  /tmp/labalice_day1.pub \
  /home/labalice/.ssh/authorized_keys
```

Trong đó:

```text
/tmp/labalice_day1.pub
```

chỉ chứa public key.

Nếu `authorized_keys` đã tồn tại, không dùng command trên vì sẽ overwrite file. Phải theo approved append/management procedure.

---

# 107. Verify server-side SSH files

```bash
sudo ls -ld /home/labalice/.ssh
sudo ls -l /home/labalice/.ssh/authorized_keys
```

Typical safe permissions:

```text
~/.ssh               700
authorized_keys      600
```

và ownership:

```text
labalice:labalice
```

Exact SSH access checks phụ thuộc deployed configuration.

---

# LAB PHASE 20 — SSH Host Identity Verification

## 108. Không blindly accept host key

First connection:

```bash
ssh -i "$HOME/.ssh/day1_lab_ed25519" labalice@rhel-lab01
```

Client có thể hiện host-key fingerprint.

Bạn phải so sánh với fingerprint đã được instructor/administrator cung cấp qua trusted channel.

Không nên chỉ trả lời:

```text
yes
```

vì thấy warning.

---

# 109. Sau khi trusted host key được lưu

Check known host:

```bash
ssh-keygen -F rhel-lab01
```

Điều này kiểm tra client đã có known-host entry, không tự chứng minh entry đó được trusted đúng cách lúc đầu.

---

# LAB PHASE 21 — Positive SSH Authentication

## 110. Connect with explicit key

```bash
ssh \
  -i "$HOME/.ssh/day1_lab_ed25519" \
  -o IdentitiesOnly=yes \
  labalice@rhel-lab01
```

Sau login:

```bash
hostname
whoami
id
```

Expected:

```text
hostname → rhel-lab01
whoami   → labalice
```

`id` phải phản ánh groups expected.

---

# 111. What success actually proves

SSH login success chứng minh:

```text
network path worked
SSH service responded
host verification completed according to client state
selected identity authenticated
session was authorized
```

Nó **không chứng minh**:

```text
labalice has correct sudo
all filesystem permissions are correct
server configuration is secure
```

Mỗi claim cần verification riêng.

---

# LAB PHASE 22 — Negative SSH Authentication

Syllabus yêu cầu verify **authorized and denied access**. Infrastructure_Operations_14_Da…

Ta không phá server configuration để tạo denied case.

---

# 112. Generate a wrong/disposable key

Trên workstation:

```bash
ssh-keygen \
  -t ed25519 \
  -f "$HOME/.ssh/day1_wrong_ed25519" \
  -C "day1-wrong-key"
```

Không install public key này lên server.

---

# 113. Force public-key-only negative test

```bash
ssh \
  -i "$HOME/.ssh/day1_wrong_ed25519" \
  -o IdentitiesOnly=yes \
  -o PreferredAuthentications=publickey \
  -o PasswordAuthentication=no \
  -o KbdInteractiveAuthentication=no \
  labalice@rhel-lab01
```

Expected:

```text
authentication denied
```

thường có dạng:

```text
Permission denied (publickey...)
```

Exact message phụ thuộc server/client configuration.

---

# 114. Why force `IdentitiesOnly`?

Nếu workstation có nhiều keys loaded, SSH client có thể thử key khác và accidental success.

`IdentitiesOnly=yes` giúp test đúng key được chỉ định.

---

# 115. Why disable fallback password?

Nếu wrong key fail rồi client fallback password, lab có thể trở nên mơ hồ.

Ta muốn chứng minh:

```text
wrong key → denied
```

không phải:

```text
wrong key failed but password succeeded
```

---

# 116. SSH evidence

Nên giữ:

```text
public key fingerprint
target hostname
username
timestamp
successful login result
wrong-key denied result
```

Không giữ:

```text
private key
passphrase
password
secret
```

---

# LAB PHASE 23 — SSH Troubleshooting Checkpoints

## 117. Case A — `Could not resolve hostname`

Investigation:

```bash
hostname
```

trên local server không giải quyết DNS, nhưng first check target spelling.

Nếu networking troubleshooting sâu cần `getent hosts`, `ping`, `ip`, `ss`, `firewall`, đó chủ yếu thuộc Day 2.

Trong Day 1, xác định:

```text
name-resolution layer suspected
```

và escalate/transition theo scope.

---

# 118. Case B — `Connection refused`

Điều này không giống authentication denial.

Concept:

```text
TCP target was reached/rejected
```

hoặc service/listener issue có thể tồn tại.

Day 1 learner nên:

```text
record exact error
confirm target
escalate to Day-2/service troubleshooting boundary
```

thay vì chỉnh `sshd` ngay.

---

# 119. Case C — `Permission denied (publickey)`

Check theo low-risk order:

```text
correct username?
correct key file?
correct hostname?
public key installed?
correct ownership?
reasonable ~/.ssh permissions?
correct authorized_keys?
server policy within scope?
```

Client diagnostic:

```bash
ssh -v \
  -i "$HOME/.ssh/day1_lab_ed25519" \
  -o IdentitiesOnly=yes \
  labalice@rhel-lab01
```

Sanitize output trước khi lưu.

---

# 120. Case D — Host key changed

Nếu client báo host-key mismatch:

```text
STOP
```

Không chạy ngay:

```bash
ssh-keygen -R host
```

chỉ để warning biến mất.

Phải xác minh:

```text
Was server rebuilt?
Was DNS/IP changed?
Was host legitimately rekeyed?
Am I connecting to wrong system?
```

Chỉ update trusted host entry sau validation.

---

# LAB PHASE 24 — Permission Troubleshooting Exercise

## 121. Scenario

`labalice` phải đọc:

```text
/srv/day1-share/team.txt
```

nhưng bị:

```text
Permission denied
```

---

# 122. Step 1 — identity

```bash
id labalice
```

Question:

```text
Is day1ops present?
```

---

# 123. Step 2 — file state

```bash
stat -c '%U %G %A %a %n' /srv/day1-share/team.txt
```

Expected design:

```text
root day1ops ... 660
```

---

# 124. Step 3 — directory

```bash
stat -c '%U %G %A %a %n' /srv/day1-share
```

Expected:

```text
root day1ops ... 770
```

---

# 125. Step 4 — new session issue

Nếu group vừa được add:

```text
old login session
```

có thể chưa reflect it.

Test new authorized session.

---

# 126. Step 5 — only then modify

Không:

```bash
chmod 777 ...
```

Instead identify exact mismatch.

Ví dụ group owner sai:

```text
Current:
root:root

Expected:
root:day1ops
```

Authorized correction:

```bash
sudo chgrp day1ops /srv/day1-share/team.txt
```

Then:

```bash
sudo -u labalice -- cat /srv/day1-share/team.txt
```

---

# LAB PHASE 25 — Configuration Baseline Capture

## 127. Objective

Lưu state của Day 1 configuration.

Một minimum baseline:

```text
Users:
labalice
labbob

Groups:
day1ops
day1sudo

Shared directory:
/srv/day1-share

Permissions:
/srv/day1-share
/srv/day1-share/team.txt

Sudo rule:
/etc/sudoers.d/day1-lab

SSH:
labalice .ssh ownership/modes
authorized public-key fingerprint
```

---

# 128. Generate configuration evidence

Ví dụ:

```bash
{
  echo "Collected: $(date --iso-8601=seconds)"
  echo "Host: $(hostname)"
  echo
  echo "=== labalice ==="
  id labalice
  echo
  echo "=== labbob ==="
  id labbob
  echo
  echo "=== groups ==="
  getent group day1ops
  getent group day1sudo
  echo
  echo "=== shared directory ==="
  stat -c '%U %G %A %a %n' /srv/day1-share
  stat -c '%U %G %A %a %n' /srv/day1-share/team.txt
  echo
  echo "=== SSH permissions ==="
  sudo stat -c '%U %G %A %a %n' /home/labalice/.ssh
  sudo stat -c '%U %G %A %a %n' /home/labalice/.ssh/authorized_keys
} > "$HOME/infra-ops-lab/Day01/configuration/day1_configuration_baseline.txt"
```

Note:

Sudo rule output có thể chứa administrative policy information; chỉ include nếu evidence scope cho phép.

---

# LAB PHASE 26 — Access Verification Matrix

## 129. Objective

Không dùng câu:

> "Permissions look correct."

Thay bằng test matrix.

| Test                                                    | Expected |
| ------------------------------------------------------- | -------- |
| `labalice` reads `/srv/day1-share/team.txt`             | ALLOW    |
| `labbob` reads same file                                | DENY     |
| `labalice` reads protected root file normally           | DENY     |
| `labalice` reads sudo test file via approved `sudo cat` | ALLOW    |
| `labalice` runs unapproved `sudo id`                    | DENY     |
| correct SSH key                                         | ALLOW    |
| wrong SSH key                                           | DENY     |

Đây là core Day 1 acceptance pattern.

---

# LAB PHASE 27 — Evidence Manifest

Syllabus common evidence contract yêu cầu mỗi evidence item có evidence ID, source system/product/version, environment/asset/account identifier, related ticket/stable record key, collector, timestamp/timezone, exact commands/filters, format/encoding, path/name, size/count, SHA-256, classification, parent evidence/transformation và limitations/errors. Infrastructure_Operations_14_Da…

## 130. Manifest template

Ví dụ:

| Field           | Example                   |
| --------------- | ------------------------- |
| Evidence ID     | `D01-EVID-010`            |
| Source          | `rhel-lab01 / RHEL`       |
| Environment     | `LAB`                     |
| Asset ID        | `LAB-LNX-001`             |
| Ticket          | `LAB-D01-001`             |
| Collector       | `<learner-id>`            |
| Collection time | `2026-...+07:00`          |
| Command         | `stat ...`                |
| Format          | UTF-8 text                |
| Filename        | `permission_baseline.txt` |
| File size       | `<bytes>`                 |
| Record count    | `<defined-count>`         |
| SHA-256         | `<hash>`                  |
| Classification  | Internal/Lab              |
| Parent ID       | blank if raw              |
| Limitations     | none / explicit gap       |

Exact classification labels phải theo organization.

---

# LAB PHASE 28 — Hash Evidence Integrity Exercise

Đây là syllabus-mandated additive lab.

## 131. Create source export

```bash
cat > raw/sample_export.txt <<'EOF'
record001 ALLOW labalice
record002 DENY labbob
record003 ALLOW labalice
EOF
```

Count:

```bash
wc -l raw/sample_export.txt
```

Expected:

```text
3
```

Trong fixture này ta định nghĩa:

```text
1 line = 1 record
```

---

# 132. Hash original

```bash
sha256sum raw/sample_export.txt
```

Save:

```bash
sha256sum raw/sample_export.txt > raw/sample_export.txt.sha256
```

Inspect:

```bash
cat raw/sample_export.txt.sha256
```

---

# 133. Verify original against recorded hash

```bash
sha256sum -c raw/sample_export.txt.sha256
```

Expected:

```text
raw/sample_export.txt: OK
```

---

# 134. Create working copy

```bash
cp raw/sample_export.txt working/sample_export.txt
```

Verify copies initially identical:

```bash
sha256sum raw/sample_export.txt working/sample_export.txt
```

Expected:

```text
same SHA-256 values
```

---

# 135. Alter working copy only

```bash
printf 'record004 DENY labcharlie\n' >> working/sample_export.txt
```

Do **not** edit:

```text
raw/sample_export.txt
```

---

# 136. Compare hashes

```bash
sha256sum raw/sample_export.txt working/sample_export.txt
```

Expected:

```text
different SHA-256
```

---

# 137. Compare counts

```bash
wc -l raw/sample_export.txt working/sample_export.txt
```

Expected:

```text
raw     = 3
working = 4
```

---

# 138. Verify original again

```bash
sha256sum -c raw/sample_export.txt.sha256
```

Expected:

```text
OK
```

This proves:

```text
original bytes remain unchanged
```

---

# 139. What have we actually proven?

We have proven:

```text
raw file still matches recorded hash
working file differs from raw
```

We have **not** proven:

```text
source was authentic
source contained every event
source query was correct
three original records represented the whole system
```

Syllabus explicitly warns that hash detects changes against a trusted recorded baseline but does not independently prove source authenticity or completeness. Infrastructure_Operations_14_Da…

---

# 140. Record transformation

Manifest for working copy:

```text
Evidence ID:
D01-EVID-021

Parent:
D01-EVID-020

Transformation:
Appended synthetic record004 for integrity test

Original modified?
No

Working SHA-256:
...

Record count:
4
```

This is the expected evidence lineage.

---

# LAB PHASE 29 — Evidence Classification and Secret Review

## 141. Before submission

Search deliverables for sensitive material.

Do not submit:

```text
private SSH key
password
token
secret
MFA seed
production credential
/etc/shadow content
```

Public SSH key is not secret, nhưng vẫn chỉ include nếu relevant.

---

# 142. Evidence review questions

Trước submit từng artifact:

```text
Can reviewer identify source?
Can reviewer identify host?
Can reviewer identify environment?
Is collection time present?
Is timezone clear?
Is ticket/reference present?
Is exact command/query known?
Is hash recorded?
Is raw original preserved?
Are limitations recorded?
Does it contain secrets?
```

---

# LAB PHASE 30 — Failure Injection

Tất cả failure injection dưới đây chỉ dùng trong lab.

---

## Failure 1 — Wrong Current Directory

### Injection

Bạn được yêu cầu inspect:

```text
$HOME/day1-cli/files
```

nhưng đang ở:

```text
$HOME/day1-search
```

và chạy relative command:

```bash
cat events.txt
```

Result:

```text
No such file or directory
```

### Expected troubleshooting

```bash
pwd
ls -la
```

Then locate:

```bash
find "$HOME" -path '*/day1-cli/files/events.txt' -print
```

### Root cause

Wrong working directory / wrong relative path.

### Corrective action

Use correct path:

```bash
cat "$HOME/day1-cli/files/events.txt"
```

### Lesson

Không tạo lại file chỉ vì relative lookup fail.

---

## Failure 2 — Missing Group Membership

### Injection

Authorized instructor removes `labalice`:

```bash
sudo gpasswd -d labalice day1ops
```

Fresh session/access test fails.

### Symptom

```text
Permission denied
```

### Troubleshooting

```bash
id labalice
stat -c '%U %G %A %a %n' /srv/day1-share
stat -c '%U %G %A %a %n' /srv/day1-share/team.txt
```

### Root cause

`labalice` no longer belongs to `day1ops`.

### Rollback

```bash
sudo usermod -aG day1ops labalice
```

Then new login/session and retest.

---

## Failure 3 — Wrong File Group

### Injection

```bash
sudo chgrp root /srv/day1-share/team.txt
```

### Symptom

`labalice` may be denied depending on mode.

### Troubleshooting

```bash
stat -c '%U %G %A %a %n' /srv/day1-share/team.txt
id labalice
```

### Expected

Discover:

```text
expected group: day1ops
actual group: root
```

### Corrective action

```bash
sudo chgrp day1ops /srv/day1-share/team.txt
```

Verify positive + negative cases again.

---

## Failure 4 — Wrong SSH Key

### Injection

Use:

```text
day1_wrong_ed25519
```

instead of authorized key.

### Expected symptom

Authentication denial.

### Troubleshooting

Client:

```bash
ssh -v \
  -i "$HOME/.ssh/day1_wrong_ed25519" \
  -o IdentitiesOnly=yes \
  -o PreferredAuthentications=publickey \
  -o PasswordAuthentication=no \
  labalice@rhel-lab01
```

Do not change server configuration.

### Root cause

Presented key does not correspond to authorized public key.

### Recovery

Use:

```text
day1_lab_ed25519
```

and retest.

---

## Failure 5 — Evidence Hash Mismatch

### Injection

Already performed by modifying:

```text
working/sample_export.txt
```

### Expected

```text
working hash != raw hash
```

### Correct response

Do **not** overwrite raw.

Document:

```text
intentional derived modification
new hash
parent ID
record-count difference
```

### Lesson

Hash mismatch does not always mean malicious tampering.

It means:

```text
bytes differ
```

Context determines whether difference is legitimate.

---

# LAB PHASE 31 — Rollback and Cleanup

Rollback phải được xem như một phần của lab, không phải việc "dọn cho đẹp".

## 143. Before cleanup

Đảm bảo evidence đã captured:

```text
system inventory
users/groups
permissions
sudo allowed/denied
SSH allowed/denied
hash exercise
manifest
```

---

# 144. Remove sudo rule

Nếu lab yêu cầu restore server:

```bash
sudo rm /etc/sudoers.d/day1-lab
sudo visudo -c
```

Verify user no longer has lab sudo entitlement:

```bash
sudo -u labalice -- sudo -l
```

Behavior có thể phụ thuộc authentication/environment, nên instructor có thể verify trong fresh session.

---

# 145. Remove protected sudo fixture

```bash
sudo rm /var/tmp/day1-admin-check
```

Verify:

```bash
ls -l /var/tmp/day1-admin-check
```

Expected absent.

---

# 146. Remove shared lab directory

**Only if `/srv/day1-share` was created exclusively for this lab.**

Pre-check:

```bash
sudo ls -la /srv/day1-share
```

Then:

```bash
sudo rm -r /srv/day1-share
```

Đây là destructive command.

Nếu bất cứ content nào ngoài lab fixture xuất hiện:

```text
STOP
```

Không xóa.

---

# 147. Remove lab users

Only dedicated lab accounts:

```bash
sudo userdel -r labalice
sudo userdel -r labbob
```

`-r` có thể remove home/mail-related files according to implementation, nên đây là destructive cleanup.

Verify first:

```bash
getent passwd labalice
getent passwd labbob
```

và confirm account ownership.

---

# 148. Remove groups

Sau users:

```bash
sudo groupdel day1ops
sudo groupdel day1sudo
```

Verify:

```bash
getent group day1ops
getent group day1sudo
```

Expected no result.

---

# 149. SSH key cleanup

Nếu key chỉ dùng cho disposable Day 1 lab:

```bash
rm "$HOME/.ssh/day1_wrong_ed25519" \
   "$HOME/.ssh/day1_wrong_ed25519.pub"
```

Đối với authorized Day 1 key, cleanup phải theo lab policy.

Nếu key cần dùng Day 2:

```text
retain securely
```

Nếu không:

```text
revoke public key server-side
then securely remove unused private credential according to policy
```

Không upload private key vào evidence repository.

---

# 150. Cleanup verification

Không đủ để nói:

> "Tôi đã cleanup."

Phải verify.

Ví dụ:

```bash
getent passwd labalice
getent group day1ops
ls -ld /srv/day1-share
ls -l /var/tmp/day1-admin-check
```

Expected:

```text
objects absent
```

nếu full cleanup là target.

---

# LAB PHASE 32 — Complete Evidence Package

Một Day 1 submission tốt nên có cấu trúc như:

```text
Day01/
│
├── raw/
│   ├── sample_export.txt
│   └── sample_export.txt.sha256
│
├── working/
│   └── sample_export.txt
│
├── inventory/
│   └── system_inventory.txt
│
├── configuration/
│   ├── users_groups_baseline.txt
│   ├── permissions_baseline.txt
│   ├── sudo_verification.txt
│   └── ssh_verification.txt
│
├── transcripts/
│   ├── day1_commands.txt
│   └── day1_terminal_session.txt
│
├── screenshots/
│   └── <only useful UI/context screenshots>
│
├── deliverable/
│   └── day1_summary.txt
│
└── manifest/
    └── evidence_manifest.csv
```

Đây là suggested organization, không phải exact folder structure mandated bởi syllabus.

---

# 151. Required deliverables mapping

Theo Day 1 syllabus, tối thiểu bạn cần chứng minh các outcome sau. Infrastructure_Operations_14_Da…

| Syllabus requirement       | Evidence                           |
| -------------------------- | ---------------------------------- |
| Administrative workstation | readiness record                   |
| Approved access checklist  | access checklist                   |
| Linux/RHEL environment     | system inventory                   |
| Evidence repository        | repository structure               |
| Personal learning baseline | before/after baseline              |
| CLI navigation             | command transcript                 |
| File creation              | output/state                       |
| Copy                       | original/copy verification         |
| Move                       | before/after state                 |
| Delete                     | safe deletion evidence             |
| Searching                  | `find` output                      |
| Text processing            | `grep`, `wc`, `sort`, etc.         |
| Users/groups               | `getent`, `id`                     |
| Ownership                  | `stat`, `ls -l`                    |
| Permissions                | mode evidence                      |
| `chmod/chown/chgrp`        | before/change/after                |
| `umask`                    | original/test/restored             |
| Least-privilege sudo       | `sudo -l` + allow/deny             |
| SSH keys                   | public fingerprint, no private key |
| Authorized access          | successful test                    |
| Denied access              | failed test                        |
| Executed commands          | transcript/run log                 |
| System inventory           | inventory file                     |
| Access verification        | access matrix                      |
| Configuration evidence     | baseline file                      |
| Evidence manifest          | manifest                           |
| SHA-256 exercise           | original hash + mismatch           |
| Untouched original         | original re-verification           |

---

# 152. Day 1 Acceptance Criteria

Một reviewer phải có thể trả lời được:

### System

```text
Which host was used?
Which OS?
Which kernel?
Which environment?
```

### Authorization

```text
Which ticket?
Which learner/admin account?
What actions were approved?
```

### Linux access

```text
Which users/groups were created?
What ownership/modes were configured?
Who was allowed?
Who was denied?
```

### Sudo

```text
Which command was allowed?
Which unauthorized command was denied?
Was unrestricted root access accidentally granted?
```

### SSH

```text
Which account?
Which public-key fingerprint?
Did correct key work?
Did wrong key fail?
```

### Evidence

```text
Which item is raw?
Which item is derived?
What is the SHA-256?
Was original preserved?
Can derived item trace back to parent?
```

Syllabus's explicit Acceptance/Evidence condition is: reviewer phải trace được evidence item về source và ticket, original phải giữ nguyên, và modified copy phải fail comparison với original hash. Infrastructure_Operations_14_Da…

---

# 153. Common Practical Mistakes

## Mistake 1 — Không check hostname

Bạn login nhầm server rồi bắt đầu:

```bash
sudo useradd ...
```

Đây là operational failure dù command syntax đúng.

---

## Mistake 2 — Dùng `chmod 777`

Symptom:

```text
Permission denied
```

Action:

```bash
chmod 777 file
```

Không có root-cause reasoning.

Đây là bad practice.

---

## Mistake 3 — Test only success

Nếu:

```text
labalice allowed
```

nhưng:

```text
labbob also allowed
```

policy đã fail.

---

## Mistake 4 — `usermod -G` sai

Có thể unintentionally replace supplementary groups.

Use approved method và verify before/after.

---

## Mistake 5 — Group changed nhưng old session chưa refresh

Bạn tưởng configuration fail và sửa thêm permissions.

Kết quả có thể over-permission system.

---

## Mistake 6 — Store private key trong evidence

Đây là credential exposure, không phải "evidence tốt".

---

## Mistake 7 — Blindly accept SSH host-key warning

Bạn bỏ qua server identity validation.

---

## Mistake 8 — Delete known-host entry ngay khi warning

Điều này xóa warning, không xác minh root cause.

---

## Mistake 9 — Edit raw evidence

Khi reviewer hash lại:

```text
mismatch
```

nhưng không có transformation record.

Evidence lineage fail.

---

## Mistake 10 — Hash rồi tuyên bố evidence complete

Hash chỉ chứng minh byte integrity relative to baseline.

---

## Mistake 11 — Gửi `history` như audit log

History không phải authoritative audit mechanism.

---

## Mistake 12 — Cleanup trước khi evidence capture

Bạn mất before/after proof.

---

# 154. Practical Troubleshooting Flow

Khi một test không đạt, dùng flow:

```text
Confirm expected result
        ↓
Confirm actual result
        ↓
Confirm identity
        ↓
Confirm hostname/environment
        ↓
Inspect current configuration
        ↓
Compare with baseline
        ↓
Identify smallest possible cause
        ↓
Run read-only test
        ↓
Change only if authorized
        ↓
Retest
        ↓
Capture evidence
```

---

# 155. Permission Troubleshooting Flow

```text
User cannot access file
        ↓
id <user>
        ↓
Correct group?
        ↓
Inspect directory ownership/mode
        ↓
Can user traverse directory?
        ↓
Inspect file ownership/mode
        ↓
Expected owner/group?
        ↓
Old session?
        ↓
Other controls?
        ↓
Fix minimum required item
        ↓
Positive test
        ↓
Negative test
```

`SELinux` troubleshooting sâu được dành cho Day 2.

---

# 156. SSH Troubleshooting Flow

```text
SSH fails
  │
  ├─ Correct hostname?
  │
  ├─ Correct username?
  │
  ├─ Correct server identity?
  │
  ├─ TCP/session reached SSH?
  │
  ├─ Correct private key selected?
  │
  ├─ Public key authorized?
  │
  ├─ .ssh ownership/mode reasonable?
  │
  └─ Server policy outside Day 1 scope?
         ↓
       Escalate / Day 2
```

---

# 157. Sudo Troubleshooting Flow

```text
sudo command fails
      ↓
id
      ↓
sudo -l
      ↓
correct group?
      ↓
correct sudoers rule?
      ↓
exact executable path?
      ↓
exact arguments?
      ↓
visudo -c
      ↓
new session if group recently changed
```

Do not fix with:

```text
ALL=(ALL) ALL
```

unless that is explicitly authorized requirement, which it is not in this Day 1 lab design.

---

# 158. Evidence Troubleshooting Flow

```text
Hash mismatch
      ↓
Which file?
      ↓
Raw or derived?
      ↓
Was transformation intended?
      ↓
Is parent evidence known?
      ↓
Was original modified?
      ↓
Compare size/count/hash
      ↓
Document explanation
```

If raw original changed unexpectedly:

```text
STOP
preserve current state
do not overwrite
record discrepancy
escalate to reviewer
```

---

# 159. Escalation Boundaries trong Lab

Bạn nên escalate thay vì tiếp tục nếu:

```text
Target turns out to be production
Unknown account already exists
Unknown group already exists
Existing authorized_keys would be overwritten
SSH host-key unexpectedly changed
sudo -l shows unexplained broad privilege
visudo validation fails
Permission fix may affect real application
Directory contains non-lab data
Private key or credential appears exposed
Evidence original unexpectedly changed
Lab state differs materially from documented topology
```

Đây không phải weakness.

Đó là đúng operational behavior.

---

# 160. Practical Operations Scenarios

Dưới đây là các scenarios bạn nên tự suy luận trước khi xem bất kỳ solution nào.

## Scenario 1 — Wrong host

Bạn được giao ticket cho:

```text
rhel-lab01
```

nhưng sau SSH:

```bash
hostname
```

trả về:

```text
rhel-prod01
```

Bạn phải giải thích:

```text
What do you do?
What evidence do you retain?
Do you run any configuration command?
Who do you notify?
```

---

## Scenario 2 — User still denied after group change

Bạn vừa:

```bash
sudo usermod -aG day1ops labalice
```

`id labalice` từ admin context cho thấy group đúng, nhưng SSH session cũ của `labalice` vẫn không có group.

Bạn cần xác định nguyên nhân có thể là gì trước khi thay permission.

---

## Scenario 3 — File is `660`, user still denied

State:

```text
file:
root day1ops 660

user:
member of day1ops
```

nhưng user vẫn denied.

Bạn phải kiểm tra:

```text
parent directory permissions
```

thay vì đổi file thành `666`.

---

## Scenario 4 — `sudo -l` unexpectedly shows `ALL`

Lab policy chỉ yêu cầu `cat /var/tmp/day1-admin-check`, nhưng account có:

```text
(ALL) ALL
```

Bạn có được coi least-privilege lab là pass không?

Không.

Phải record access gap và investigate existing entitlement.

---

## Scenario 5 — Correct SSH key appears to fail

Client has many identities.

Bạn chạy:

```bash
ssh labalice@rhel-lab01
```

và authentication fail.

Bước useful:

```bash
ssh -v -i ~/.ssh/day1_lab_ed25519 -o IdentitiesOnly=yes ...
```

để kiểm tra đúng identity được offer.

---

## Scenario 6 — Host key changed

Không được giải quyết bằng việc xóa known-host record ngay lập tức.

Phải verify host identity qua trusted process.

---

## Scenario 7 — Evidence hash mismatch

Raw hash:

```text
ABC
```

Working hash:

```text
XYZ
```

Nếu working copy đã intentionally redacted, mismatch là expected.

Nếu raw current hash đổi từ:

```text
ABC
```

thành:

```text
XYZ
```

không có documented transformation, đó là evidence-integrity problem.

---

## Scenario 8 — Cleanup finds unexpected file

Bạn định:

```bash
sudo rm -r /srv/day1-share
```

nhưng:

```bash
ls -la /srv/day1-share
```

cho thấy một file không thuộc lab.

Correct response:

```text
STOP
```

không tiếp tục cleanup destructive operation.

---

# 161. Practical Knowledge Check

Không có đáp án ở đây; nếu bạn trả lời sau, tôi có thể chấm từng câu theo `Correct / Partially Correct / Incorrect / Missing Evidence`.

1. Trước khi chạy `sudo useradd`, 5 checks quan trọng nhất là gì?
2. Vì sao `hostname`, `whoami`, `pwd` nên được kiểm tra trước risky change?
3. `cp` thành công có đủ chứng minh destination content giống source không?
4. Vì sao `chmod 777` là một cách xử lý permission kém?
5. Vì sao file `660` vẫn có thể inaccessible với member của group?
6. Sau `usermod -aG`, vì sao session hiện tại có thể chưa thấy group mới?
7. Negative access testing chứng minh điều gì mà positive testing không chứng minh?
8. `sudo -l` dùng để verify gì?
9. Vì sao sudoers command cần exact executable path?
10. Vì sao không copy private SSH key lên server?
11. `authorized_keys` chứa private hay public key?
12. Nếu SSH host key thay đổi, tại sao không nên xóa warning ngay?
13. `Permission denied (publickey)` khác `Connection timed out` như thế nào về troubleshooting layer?
14. Vì sao `IdentitiesOnly=yes` hữu ích trong SSH test?
15. `umask 027` thường ảnh hưởng file/directory mới như thế nào?
16. SHA-256 mismatch chứng minh điều gì?
17. SHA-256 match không chứng minh điều gì?
18. Tại sao raw evidence và working copy phải tách biệt?
19. Khi nào screenshot không đủ làm evidence?
20. Khi cleanup thấy data không thuộc lab, action đúng là gì?

---

# 162. Day 1 Practical Completion Checklist

Bạn chỉ nên đánh dấu Day 1 practical hoàn thành khi tất cả các item phù hợp đã được verify.

### Environment

- [ ] Administrative workstation ready.
- [ ] SSH client verified.
- [ ] Correct LAB environment confirmed.
- [ ] Correct hostname confirmed.
- [ ] Authorized account confirmed.
- [ ] Ticket/reference exists.

### Evidence

- [ ] Evidence repository created.
- [ ] `raw/` separated from `working/`.
- [ ] Evidence manifest exists.
- [ ] Timestamp/timezone recorded.
- [ ] No secrets included.

### Inventory

- [ ] OS identified.
- [ ] Kernel identified.
- [ ] Architecture identified.
- [ ] Hostname identified.
- [ ] Current identity recorded.
- [ ] Baseline evidence stored.

### CLI

- [ ] `pwd`.
- [ ] `ls`.
- [ ] `cd`.
- [ ] absolute path.
- [ ] relative path.
- [ ] `.`.
- [ ] `..`.
- [ ] `~`.

### File operations

- [ ] File created.
- [ ] Directory created.
- [ ] File copied.
- [ ] Copy verified.
- [ ] File renamed/moved.
- [ ] Safe deletion performed.
- [ ] Deletion verified.

### Search/text

- [ ] `find`.
- [ ] `grep`.
- [ ] `head`.
- [ ] `tail`.
- [ ] `wc`.
- [ ] `sort`/basic processing.
- [ ] pipe.
- [ ] stdout redirection.
- [ ] stderr redirection.

### Users/groups

- [ ] Controlled users created.
- [ ] Controlled groups created.
- [ ] Group membership assigned.
- [ ] `id` verification completed.
- [ ] `getent` verification completed.

### Permissions

- [ ] Ownership configured.
- [ ] Group ownership configured.
- [ ] Numeric `chmod` used.
- [ ] Symbolic `chmod` used.
- [ ] `chown` verified.
- [ ] `chgrp` verified.
- [ ] `umask` tested.
- [ ] Original `umask` restored.
- [ ] Allowed filesystem access verified.
- [ ] Denied filesystem access verified.

### Sudo

- [ ] Protected lab target created.
- [ ] Normal access denied.
- [ ] Dedicated least-privilege rule created.
- [ ] `visudo` validation passed.
- [ ] `sudo -l` reviewed.
- [ ] Authorized sudo command succeeds.
- [ ] Unauthorized sudo command fails.
- [ ] No unexplained unrestricted sudo entitlement.

### SSH

- [ ] Dedicated key pair generated.
- [ ] Private key protected.
- [ ] Public fingerprint recorded.
- [ ] Public key installed appropriately.
- [ ] Server identity validated.
- [ ] Correct-key login succeeds.
- [ ] Wrong-key login fails.
- [ ] No private key included in evidence.

### Evidence integrity

- [ ] Raw sample generated/exported.
- [ ] Raw SHA-256 captured.
- [ ] Raw verification passes.
- [ ] Working copy created.
- [ ] Working copy modified.
- [ ] Working hash differs.
- [ ] Original raw hash still passes.
- [ ] Transformation documented.
- [ ] Parent evidence ID documented.
- [ ] Record counts documented.

### Cleanup/Rollback

- [ ] Required evidence captured before cleanup.
- [ ] Lab sudo rule removed if required.
- [ ] Protected fixture removed if required.
- [ ] Test directory removed if required.
- [ ] Lab users/groups removed if required.
- [ ] SSH access revoked if required.
- [ ] Cleanup state verified.

---

# 163. Practical Cheat Sheet

### Environment verification

```bash
hostname
whoami
id
pwd
date --iso-8601=seconds
```

### OS/kernel

```bash
cat /etc/os-release
uname -r
uname -m
uname -a
```

### Navigation

```bash
pwd
ls
ls -la
cd /absolute/path
cd relative/path
cd ..
cd ~
```

### Files

```bash
touch file
mkdir directory
mkdir -p path
cp source destination
mv source destination
rm file
rmdir empty-directory
```

### Search/text

```bash
find PATH -type f -name 'pattern'
grep 'text' file
head file
tail file
wc -l file
sort file
uniq file
```

### Pipes/redirection

```bash
command1 | command2
command > file
command >> file
command 2> errors
command > all 2>&1
```

### Identity

```bash
whoami
id
id username
getent passwd username
getent group groupname
```

### Users/groups

```bash
sudo groupadd group
sudo useradd -m user
sudo usermod -aG group user
```

### Ownership

```bash
ls -l
stat file
chown user file
chown user:group file
chgrp group file
```

### Permissions

```bash
chmod 640 file
chmod u+x file
stat -c '%U %G %A %a %n' file
```

### `umask`

```bash
umask
umask 027
```

### Sudo

```bash
sudo -l
sudo visudo -f /etc/sudoers.d/file
sudo visudo -c
```

### SSH

```bash
ssh-keygen -t ed25519
ssh-keygen -lf key.pub
ssh-copy-id -i key.pub user@host
ssh -i key -o IdentitiesOnly=yes user@host
ssh -v -i key -o IdentitiesOnly=yes user@host
ssh-keygen -F hostname
```

### Hashing

```bash
sha256sum file
sha256sum file > file.sha256
sha256sum -c file.sha256
```

---

# 164. End-to-End Day 1 Operational Workflow

Nếu gom toàn bộ module lại thành một workflow duy nhất, Day 1 phải trông như sau:

```text
Receive lab authorization
        ↓
Verify workstation
        ↓
Verify target environment
        ↓
hostname / whoami / id / pwd
        ↓
Create evidence repository
        ↓
Start command/activity record
        ↓
Collect RHEL/Linux system inventory
        ↓
Practice CLI navigation
        ↓
Create/copy/move/delete lab files
        ↓
Search/filter/process controlled data
        ↓
Create controlled users/groups
        ↓
Configure ownership/permissions
        ↓
Positive filesystem access test
        ↓
Negative filesystem access test
        ↓
Test umask
        ↓
Create least-privilege sudo rule
        ↓
sudo -l
        ↓
Allowed sudo test
        ↓
Denied sudo test
        ↓
Generate SSH key pair
        ↓
Protect private key
        ↓
Install public key
        ↓
Validate host identity
        ↓
Correct-key SSH succeeds
        ↓
Wrong-key SSH fails
        ↓
Collect configuration baseline
        ↓
Create evidence manifest
        ↓
Hash raw evidence
        ↓
Create working copy
        ↓
Modify working copy
        ↓
Detect hash mismatch
        ↓
Verify raw original remains unchanged
        ↓
Record transformation
        ↓
Review evidence for secrets/gaps
        ↓
Rollback/cleanup
        ↓
Verify cleanup
        ↓
Submit evidence package
```

Đó chính là tinh thần của Day 1: bạn không chỉ chứng minh rằng mình biết `cd`, `chmod`, `sudo` hoặc `ssh`. Bạn phải chứng minh rằng mình có thể thực hiện một operational task theo chuỗi **authorization → baseline → controlled execution → positive/negative verification → troubleshooting → rollback → evidence**.

Quan trọng nhất, Day 1 đặt chuẩn evidence áp dụng cho toàn khóa: evidence phải trace được về source/ticket, raw artifact phải được giữ nguyên, derived artifact phải có lineage, hash/count phải được ghi nhận và mọi limitation/error phải được nói rõ thay vì che giấu. Infrastructure_Operations_14_Da…

Nếu bạn hoàn thành được toàn bộ module trên bằng chính tay mình và có evidence đầy đủ, bạn đã đạt đúng practical foundation mà Day 1 yêu cầu để chuyển sang **Day 2 — Linux/RHEL System Administration, services, networking, storage, logs, Bash, backup/restore và operational troubleshooting**.
