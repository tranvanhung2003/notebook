# Day 1 — Module Thực hành: Infrastructure Operations & Linux Administration Foundations

Theo syllabus, Assignment/Lab của Day 1 yêu cầu bạn hoàn thành toàn bộ chuỗi: **chuẩn bị administrative workstation, approved access checklist, Linux/RHEL lab environment, evidence repository và personal learning baseline; thực hành CLI navigation, file creation/copy/move/delete/searching/text processing; tạo và quản lý users/groups; cấu hình ownership, permissions và least-privilege `sudo`; tạo SSH key và thiết lập secure SSH access; kiểm tra cả authorized lẫn denied access; document commands; cuối cùng nộp system inventory, access verification và configuration evidence.** Infrastructure_Operations_14_Da…

Module này sẽ triển khai yêu cầu đó thành một **end-to-end controlled lab**. Mặc định toàn bộ thao tác chỉ được thực hiện trên **lab/non-production/authorized environment**. Không áp dụng trực tiếp lên production.

---

# 1. Mục tiêu cuối cùng của Module Thực hành

Sau khi hoàn thành, bạn phải có khả năng tự thực hiện workflow:

```text
Prepare
  ↓
Confirm authorization
  ↓
Identify target
  ↓
Capture baseline
  ↓
Perform CLI operations
  ↓
Create users/groups
  ↓
Configure ownership/permissions
  ↓
Configure least-privilege sudo
  ↓
Configure SSH public-key access
  ↓
Positive verification
  ↓
Negative verification
  ↓
Collect evidence
  ↓
Cleanup / rollback
  ↓
Final verification
```

Điểm quan trọng là bạn không chỉ chứng minh:

```text
"It works."
```

mà còn phải chứng minh:

```text
Correct identity works.
Incorrect identity is denied.
Correct privilege works.
Unauthorized privilege is denied.
Configuration matches requirement.
Evidence is sufficient.
Rollback is possible.
```

---

# 2. Lab topology

Một topology lý tưởng:

```text
+--------------------------------+
| Administrative Workstation     |
|                                |
| Terminal                       |
| SSH client                     |
| SSH private key                |
| Evidence repository            |
+---------------+----------------+
                |
                | SSH
                |
                v
+--------------------------------+
| Linux / RHEL Lab Server        |
|                                |
| Existing approved admin        |
| sudo available                 |
| SSH server already operational |
| Local users/groups             |
| Lab filesystem objects         |
+--------------------------------+
```

Có hai roles quan trọng.

### Existing administrator

Ví dụ placeholder:

```text
labadmin
```

Account này đã được mentor/lab owner cung cấp và có đủ `sudo` privilege để tạo lab users và configuration.

### Lab-created accounts

Chúng ta sẽ dùng ví dụ:

```text
d1ops
d1viewer
```

và group:

```text
d1opsgrp
```

Tên này chỉ dành cho training. Trong environment thật phải theo organization naming convention.

---

# 3. Assumptions của lab

Lab giả định:

```text
Linux/RHEL VM đã được provision.
SSH server đã hoạt động.
Bạn có approved administrative account.
Administrative account có đủ sudo privilege.
Bạn không cần thay sshd_config.
Bạn không cần thay firewall.
Bạn không cần thay SELinux.
Bạn không cần install package mới.
```

Các chủ đề như `systemd`, networking, `firewalld`, SELinux và package management thuộc Day 2 nên Day 1 **không sửa những subsystem đó**. Infrastructure_Operations_14_Da…

Nếu một vấn đề yêu cầu thay đổi chúng, Day 1 action đúng là:

```text
collect evidence
+
document
+
escalate
```

chứ không mở rộng scope tùy tiện.

---

# 4. Safety Rules bắt buộc

Trước mọi command thay đổi state:

```bash
hostname
whoami
id
pwd
```

Mục tiêu:

```text
hostname → đúng server?
whoami   → đúng account?
id       → đúng privilege/groups?
pwd      → đúng directory?
```

Với destructive operations như:

```bash
rm
userdel
groupdel
chmod -R
chown -R
```

phải kiểm tra target trước.

Không bao giờ dùng trong lab này:

```bash
rm -rf /
chmod -R 777 /
chown -R ...
```

trên system paths.

Không chia sẻ:

```text
Password
SSH private key
Passphrase
Token
Secret
```

trong screenshot, command log hoặc evidence.

---

# 5. Phân loại commands trong lab

## Read-only verification

Ví dụ:

```bash
hostname
whoami
id
pwd
ls
stat
getent
uname
cat /etc/os-release
sudo -l
```

Các command này chủ yếu dùng để inspect state.

## Configuration-changing

Ví dụ:

```bash
mkdir
touch
cp
mv
rm
groupadd
useradd
usermod
chmod
chown
chgrp
ssh-keygen
visudo
```

Đặc biệt:

```text
useradd/usermod/userdel
sudoers change
SSH authorized_keys change
```

phải được xem là controlled administrative changes.

---

# PHASE 1 — ADMINISTRATIVE WORKSTATION PREPARATION

# 6. Objective

Chuẩn bị workstation trước khi đụng tới target server.

Bạn cần đảm bảo:

```text
Terminal works
SSH client works
Evidence location exists
Target information is known
Approved credential exists
No production target is involved
```

---

# 7. Kiểm tra SSH client

Trên workstation:

```bash
ssh -V
```

Expected result dạng:

```text
OpenSSH_...
```

Không cần version cụ thể.

Điều bạn cần chứng minh:

```text
SSH client exists
```

---

# 8. Tạo evidence repository

Ví dụ trên workstation:

```bash
mkdir -p ~/Infrastructure_Operations/Day01/{00_admin,01_inventory,02_cli,03_users_groups,04_permissions,05_sudo,06_ssh,07_verification,08_failures,09_final}
```

Kiểm tra:

```bash
find ~/Infrastructure_Operations/Day01 -maxdepth 1 -type d -print
```

Expected:

```text
Day01/
00_admin
01_inventory
02_cli
...
09_final
```

---

# 9. Ý nghĩa cấu trúc evidence

```text
00_admin
```

chứa authorization checklist, scope và lab notes.

```text
01_inventory
```

chứa system baseline.

```text
02_cli
```

chứa CLI exercise evidence.

```text
03_users_groups
```

chứa user/group configuration.

```text
04_permissions
```

chứa ownership, `chmod`, `umask`.

```text
05_sudo
```

chứa `sudo` evidence.

```text
06_ssh
```

chứa SSH evidence nhưng **không chứa private key contents**.

```text
07_verification
```

chứa positive/negative functional tests.

```text
08_failures
```

chứa Failure Injection evidence.

```text
09_final
```

chứa final report/checklist.

---

# 10. Evidence naming convention

Nên dùng naming convention nhất quán:

```text
YYYYMMDD_HHMM_<host>_<purpose>.txt
```

Ví dụ:

```text
20261018_2200_rhel-lab01_inventory.txt
```

Hoặc trong lab nhỏ:

```text
01_system_identity.txt
02_users_groups.txt
03_permission_before.txt
04_permission_after.txt
```

Consistency quan trọng hơn format cụ thể.

---

# PHASE 2 — APPROVED ACCESS CHECKLIST

# 11. Trước khi SSH

Tạo một checklist như:

```text
Day 1 Access Checklist

Target hostname:
Target IP:
Environment:
System owner:
Purpose:
Approved administrative user:
Authorization/ticket reference:
Allowed operations:
Forbidden operations:
Rollback plan:
Evidence directory:
```

Sau đó check:

```text
[ ] Target is lab/non-production
[ ] Hostname/IP confirmed
[ ] Administrative account approved
[ ] sudo scope confirmed
[ ] SSH method approved
[ ] Lab change scope understood
[ ] Evidence location prepared
[ ] Rollback method understood
[ ] No real credentials will be recorded
```

---

# 12. Vì sao bước này quan trọng?

Nếu bạn bỏ qua bước này, bạn có thể làm hoàn hảo về technical nhưng sai về operations.

Ví dụ:

```text
Correct command
+
wrong production server
=
incident
```

Hay:

```text
Correct SSH key
+
unapproved account
=
security violation
```

---

# PHASE 3 — CONNECT VÀ CONFIRM TARGET

# 13. SSH vào server bằng account có sẵn

Ví dụ:

```bash
ssh labadmin@LAB_HOST
```

Thay:

```text
LAB_HOST
```

bằng hostname/IP của authorized lab.

---

# 14. Ngay sau login

Không chạy admin change ngay.

Chạy:

```bash
hostname
whoami
id
pwd
date -Is
```

Expected:

```text
hostname = lab target
whoami   = labadmin
id       = expected UID/GID/groups
pwd      = expected home
timestamp available
```

Nếu hostname không đúng:

> dừng lab.

Không cố tiếp tục.

---

# 15. Lưu initial evidence

Có thể dùng:

```bash
{
    echo "=== Day 1 Initial Context ==="
    date -Is
    hostname
    whoami
    id
    pwd
} | tee ~/day1_initial_context.txt
```

Sau đó copy file evidence về workstation theo approved method nếu cần.

---

# PHASE 4 — SYSTEM INVENTORY

Syllabus yêu cầu nộp system inventory. Infrastructure_Operations_14_Da…

# 16. Basic inventory

Chạy:

```bash
hostname
uname -r
uname -a
cat /etc/os-release
whoami
id
```

Bạn phải biết giải thích output.

---

# 17. `hostname`

```bash
hostname
```

Xác nhận asset identity ở OS level.

Không nhất thiết đủ để xác định toàn bộ CMDB identity, nhưng rất hữu ích.

---

# 18. Kernel

```bash
uname -r
```

Expected:

```text
kernel release string
```

Lưu chính xác output thay vì đoán version.

---

# 19. OS release

```bash
cat /etc/os-release
```

Tìm các fields như:

```text
NAME=
VERSION=
ID=
VERSION_ID=
```

Nếu là RHEL thì output sẽ phản ánh RHEL release của lab.

Không cần thay đổi gì.

---

# 20. Supplemental inventory

Ngoài syllabus core, có thể thu thập basic hardware/resource inventory bằng read-only commands:

```bash
lscpu
```

```bash
free -h
```

```bash
df -hT
```

```bash
lsblk
```

Các command này giúp inventory có:

```text
CPU
Memory
Mounted filesystem capacity
Block devices
```

Đây là **kiến thức bổ sung phục vụ inventory**, không phải phần storage administration của Day 2.

---

# 21. Inventory evidence command

Ví dụ:

```bash
{
    echo "=== Timestamp ==="
    date -Is

    echo
    echo "=== Hostname ==="
    hostname

    echo
    echo "=== Identity ==="
    whoami
    id

    echo
    echo "=== Kernel ==="
    uname -a

    echo
    echo "=== OS Release ==="
    cat /etc/os-release

    echo
    echo "=== CPU ==="
    lscpu

    echo
    echo "=== Memory ==="
    free -h

    echo
    echo "=== Filesystems ==="
    df -hT

    echo
    echo "=== Block Devices ==="
    lsblk
} > ~/day1_system_inventory.txt
```

Verify:

```bash
less ~/day1_system_inventory.txt
```

---

# PHASE 5 — PERSONAL LEARNING BASELINE

Đây là deliverable được syllabus yêu cầu. Infrastructure_Operations_14_Da…

# 22. Self-assessment trước lab

Tự đánh giá mức:

```text
0 = chưa biết
1 = đã nghe nhưng chưa làm được
2 = làm được với hướng dẫn
3 = tự thực hiện và giải thích được
4 = tự troubleshoot cơ bản được
```

Cho các skills:

| Skill                   | Before Day 1 |
| ----------------------- | -----------: |
| `pwd`, `ls`, `cd`       |              |
| Absolute/relative paths |              |
| `cp`, `mv`, `rm`        |              |
| Pipes/redirection       |              |
| `grep`, `find`          |              |
| Users/groups            |              |
| Ownership               |              |
| `chmod`                 |              |
| `umask`                 |              |
| `sudo`                  |              |
| SSH                     |              |
| SSH keys                |              |
| Evidence collection     |              |
| Rollback thinking       |              |

Sau lab bạn chấm lại.

Mục tiêu là đo **progress**, không phải cố ghi điểm cao.

---

# PHASE 6 — CLI NAVIGATION

# 23. Objective

Bạn phải sử dụng được:

```text
pwd
ls
cd
absolute path
relative path
.
..
~
```

và luôn biết current context.

---

# 24. Tạo workspace

Trong home của administrative account:

```bash
mkdir -p ~/day1-lab/{input,working,output,backup}
```

Verify:

```bash
find ~/day1-lab -maxdepth 2 -type d -print
```

---

# 25. Navigation exercise

```bash
cd ~/day1-lab
pwd
```

Expected:

```text
/home/<user>/day1-lab
```

Tiếp:

```bash
cd input
pwd
```

Tiếp:

```bash
cd ..
pwd
```

Tiếp:

```bash
cd ./working
pwd
```

Tiếp:

```bash
cd ../backup
pwd
```

Cuối:

```bash
cd ~
pwd
```

Bạn phải giải thích được mỗi transition.

---

# 26. Absolute path exercise

Ví dụ:

```bash
cd ~/day1-lab
```

Sau đó:

```bash
cd /etc
pwd
```

Đây là absolute path vì bắt đầu `/`.

Quay về:

```bash
cd ~/day1-lab
```

---

# 27. Relative path exercise

Từ:

```text
~/day1-lab
```

chạy:

```bash
cd input
```

hoặc:

```bash
cd ./input
```

Hai command resolve từ current working directory.

---

# 28. Hidden files

Chạy:

```bash
ls
```

sau đó:

```bash
ls -a
```

So sánh.

Các entries:

```text
.
..
```

cũng xuất hiện khi dùng `-a`.

---

# PHASE 7 — FILE CREATION

# 29. `touch`

Trong workspace:

```bash
cd ~/day1-lab/input
touch server01.txt
touch server02.txt
touch notes.txt
```

Verify:

```bash
ls -l
```

---

# 30. Tạo file có content

Dùng:

```bash
printf '%s\n' \
'server=server01' \
'role=web' \
'status=OK' > server01.txt
```

File 2:

```bash
printf '%s\n' \
'server=server02' \
'role=database' \
'status=ERROR' > server02.txt
```

Notes:

```bash
printf '%s\n' \
'Day 1 Infrastructure Operations Lab' > notes.txt
```

Verify:

```bash
cat server01.txt
cat server02.txt
```

---

# 31. Hiểu `>`

Command:

```bash
echo "test" > file.txt
```

có thể overwrite existing file.

Vì vậy trước khi dùng `>` vào file quan trọng:

```bash
ls -l file.txt
```

và nếu cần:

```bash
cp file.txt file.txt.bak
```

Trong lab chúng ta chỉ dùng với test files.

---

# 32. Append bằng `>>`

```bash
echo "owner=infra-team" >> server01.txt
```

Verify:

```bash
cat server01.txt
```

Bạn phải thấy content cũ vẫn còn và line mới được append.

---

# PHASE 8 — COPY, MOVE, RENAME

# 33. Copy

```bash
cp ~/day1-lab/input/server01.txt ~/day1-lab/working/
```

Verify:

```bash
ls -l ~/day1-lab/working
```

So sánh:

```bash
cat ~/day1-lab/input/server01.txt
cat ~/day1-lab/working/server01.txt
```

---

# 34. Rename bằng `mv`

```bash
cd ~/day1-lab/working
mv server01.txt server01_working.txt
```

Verify:

```bash
ls -l
```

---

# 35. Move

```bash
mv server01_working.txt ../output/
```

Verify source:

```bash
ls -l
```

Verify destination:

```bash
ls -l ../output/
```

---

# 36. Backup

```bash
cp ~/day1-lab/output/server01_working.txt ~/day1-lab/backup/
```

Verify:

```bash
ls -l ~/day1-lab/backup/
```

---

# PHASE 9 — DELETE OPERATIONS

# 37. Safety checkpoint

Trước deletion:

```bash
pwd
ls -l TARGET
```

Không dùng wildcard chưa kiểm tra.

---

# 38. Controlled file deletion

Tạo disposable file:

```bash
touch ~/day1-lab/working/delete-me.txt
```

Verify:

```bash
ls -l ~/day1-lab/working/delete-me.txt
```

Delete:

```bash
rm ~/day1-lab/working/delete-me.txt
```

Verify:

```bash
ls -l ~/day1-lab/working/
```

---

# 39. Directory deletion

Tạo:

```bash
mkdir ~/day1-lab/working/tempdir
```

Nếu empty:

```bash
rmdir ~/day1-lab/working/tempdir
```

Điều này an toàn hơn việc reflexively dùng recursive delete.

---

# 40. `rm -r`

Chỉ dùng với dedicated disposable directory đã verify.

Ví dụ:

```bash
mkdir -p ~/day1-lab/working/test-delete/subdir
touch ~/day1-lab/working/test-delete/subdir/file.txt
```

Inspect:

```bash
find ~/day1-lab/working/test-delete -print
```

Sau khi xác nhận đúng:

```bash
rm -r ~/day1-lab/working/test-delete
```

Verify:

```bash
ls -ld ~/day1-lab/working/test-delete
```

Expected:

```text
No such file or directory
```

---

# PHASE 10 — SEARCHING

Syllabus yêu cầu searching exercises. Infrastructure_Operations_14_Da…

# 41. Search filename với `find`

```bash
find ~/day1-lab -type f -name '*.txt' -print
```

Expected:

Danh sách `.txt` files trong workspace.

---

# 42. Search file cụ thể

```bash
find ~/day1-lab -type f -name 'server01*' -print
```

Giải thích:

```text
~/day1-lab  → starting path
-type f     → regular file
-name       → name condition
server01*   → filename pattern
-print      → display result
```

---

# 43. Search content với `grep`

```bash
grep 'status=ERROR' ~/day1-lab/input/*.txt
```

Expected:

```text
server02.txt:status=ERROR
```

---

# 44. Case-insensitive search

```bash
grep -i 'error' ~/day1-lab/input/*.txt
```

---

# 45. Line numbers

```bash
grep -n 'status' ~/day1-lab/input/server02.txt
```

Output có dạng:

```text
3:status=ERROR
```

---

# PHASE 11 — BASIC TEXT PROCESSING

Day 2 mới đi sâu `grep/awk/sed` và regular expressions. Day 1 chỉ cần foundation về command composition và simple text processing. Infrastructure_Operations_14_Da…

# 46. `cat`

```bash
cat ~/day1-lab/input/server01.txt
```

Dùng cho small text files.

Không dùng `cat` blindly với huge files hoặc sensitive files.

---

# 47. `head`

```bash
head ~/day1-lab/input/server01.txt
```

---

# 48. `tail`

```bash
tail ~/day1-lab/input/server01.txt
```

---

# 49. `wc`

```bash
wc -l ~/day1-lab/input/server01.txt
```

Count lines.

---

# 50. Pipe exercise

```bash
cat ~/day1-lab/input/server02.txt | grep 'status'
```

Lưu ý command có thể viết đơn giản hơn:

```bash
grep 'status' ~/day1-lab/input/server02.txt
```

Nhưng pipe version giúp bạn học data flow:

```text
cat stdout
    ↓
pipe
    ↓
grep stdin
```

---

# 51. `sort`

Tạo:

```bash
printf '%s\n' \
'server03' \
'server01' \
'server02' \
'server01' > ~/day1-lab/input/servers.txt
```

Sort:

```bash
sort ~/day1-lab/input/servers.txt
```

---

# 52. `uniq`

Để `uniq` xử lý duplicates liền nhau một cách hữu dụng:

```bash
sort ~/day1-lab/input/servers.txt | uniq
```

Count:

```bash
sort ~/day1-lab/input/servers.txt | uniq -c
```

---

# 53. Pipeline evidence

```bash
sort ~/day1-lab/input/servers.txt \
| uniq -c \
| tee ~/day1-lab/output/server_counts.txt
```

Verify:

```bash
cat ~/day1-lab/output/server_counts.txt
```

---

# PHASE 12 — USERS VÀ GROUPS

Từ đây bắt đầu administrative changes.

Trước mỗi stage:

```bash
hostname
whoami
sudo -l
```

---

# 54. Requirement

Tạo:

```text
Group:
d1opsgrp

Users:
d1ops
d1viewer
```

Target behavior:

```text
d1ops:
member of d1opsgrp

d1viewer:
not member of d1opsgrp
```

---

# 55. Baseline trước change

Kiểm tra names chưa tồn tại:

```bash
getent passwd d1ops
getent passwd d1viewer
getent group d1opsgrp
```

Expected nếu chưa tồn tại:

```text
no matching output
```

Một non-zero exit code có thể xảy ra.

---

# 56. Tạo group

**Configuration-changing command:**

```bash
sudo groupadd d1opsgrp
```

Verify:

```bash
getent group d1opsgrp
```

Expected:

```text
d1opsgrp:x:<GID>:
```

Exact GID không được đoán.

---

# 57. Tạo `d1ops`

```bash
sudo useradd -m -s /bin/bash d1ops
```

Giải thích:

```text
useradd       create user
-m            create home directory
-s /bin/bash  configure login shell
d1ops         username
```

Verify:

```bash
getent passwd d1ops
```

```bash
id d1ops
```

```bash
ls -ld /home/d1ops
```

---

# 58. Tạo `d1viewer`

```bash
sudo useradd -m -s /bin/bash d1viewer
```

Verify:

```bash
getent passwd d1viewer
id d1viewer
ls -ld /home/d1viewer
```

---

# 59. Add supplementary group

```bash
sudo usermod -aG d1opsgrp d1ops
```

Điểm quan trọng:

```text
-a
```

là append.

```text
-G
```

specify supplementary groups.

Không được bỏ `-a` theo thói quen khi intention là append, vì tùy syntax nó có thể thay thế supplementary group list.

---

# 60. Verify group membership

```bash
id d1ops
```

Expected:

```text
groups=...,d1opsgrp
```

Kiểm tra `d1viewer`:

```bash
id d1viewer
```

Expected:

```text
d1opsgrp not present
```

---

# 61. Evidence

```bash
{
    date -Is
    hostname

    echo "=== Group ==="
    getent group d1opsgrp

    echo "=== d1ops ==="
    getent passwd d1ops
    id d1ops

    echo "=== d1viewer ==="
    getent passwd d1viewer
    id d1viewer
} | tee ~/day1_users_groups.txt
```

---

# PHASE 13 — OWNERSHIP

# 62. Tạo controlled filesystem area

Trong dedicated lab:

```bash
sudo mkdir -p /srv/day1-lab/reports
```

Verify:

```bash
ls -ld /srv/day1-lab
ls -ld /srv/day1-lab/reports
```

---

# 63. Tạo test file

```bash
echo "Day 1 restricted operational report" \
| sudo tee /srv/day1-lab/reports/report.txt >/dev/null
```

Verify:

```bash
ls -l /srv/day1-lab/reports/report.txt
```

---

# 64. Capture BEFORE state

```bash
{
    date -Is
    hostname
    ls -ld /srv/day1-lab/reports
    ls -l /srv/day1-lab/reports/report.txt
    stat /srv/day1-lab/reports/report.txt
} | tee ~/permission_before.txt
```

---

# 65. Change directory ownership

Requirement:

```text
owner = d1ops
group = d1opsgrp
```

Command:

```bash
sudo chown d1ops:d1opsgrp /srv/day1-lab/reports
```

Verify:

```bash
ls -ld /srv/day1-lab/reports
```

---

# 66. Change file ownership

```bash
sudo chown d1ops:d1opsgrp /srv/day1-lab/reports/report.txt
```

Verify:

```bash
ls -l /srv/day1-lab/reports/report.txt
```

---

# 67. `chgrp` exercise

Tạo second test file:

```bash
sudo touch /srv/day1-lab/reports/group-test.txt
```

Check:

```bash
ls -l /srv/day1-lab/reports/group-test.txt
```

Set group:

```bash
sudo chgrp d1opsgrp /srv/day1-lab/reports/group-test.txt
```

Verify:

```bash
ls -l /srv/day1-lab/reports/group-test.txt
```

---

# PHASE 14 — FILE PERMISSIONS

# 68. Requirement

Directory:

```text
d1ops: rwx
d1opsgrp: r-x
others: ---
```

Mode:

```text
750
```

File:

```text
d1ops: rw-
d1opsgrp: r--
others: ---
```

Mode:

```text
640
```

---

# 69. Directory permission

```bash
sudo chmod 750 /srv/day1-lab/reports
```

Verify:

```bash
ls -ld /srv/day1-lab/reports
```

Expected:

```text
drwxr-x---
```

---

# 70. File permission

```bash
sudo chmod 640 /srv/day1-lab/reports/report.txt
```

Verify:

```bash
ls -l /srv/day1-lab/reports/report.txt
```

Expected:

```text
-rw-r-----
```

---

# 71. Numeric interpretation

For:

```text
750
```

```text
7 = rwx
5 = r-x
0 = ---
```

For:

```text
640
```

```text
6 = rw-
4 = r--
0 = ---
```

Bạn phải tính được chứ không chỉ nhớ.

---

# 72. Symbolic mode exercise

Tạo:

```bash
sudo touch /srv/day1-lab/reports/symbolic-test.txt
sudo chown d1ops:d1opsgrp /srv/day1-lab/reports/symbolic-test.txt
```

Đặt initial mode:

```bash
sudo chmod 600 /srv/day1-lab/reports/symbolic-test.txt
```

Add group read:

```bash
sudo chmod g+r /srv/day1-lab/reports/symbolic-test.txt
```

Verify:

```bash
ls -l /srv/day1-lab/reports/symbolic-test.txt
```

Expected:

```text
-rw-r-----
```

Remove owner write:

```bash
sudo chmod u-w /srv/day1-lab/reports/symbolic-test.txt
```

Verify:

```bash
ls -l /srv/day1-lab/reports/symbolic-test.txt
```

Sau exercise, restore:

```bash
sudo chmod 640 /srv/day1-lab/reports/symbolic-test.txt
```

---

# PHASE 15 — POSITIVE FILE ACCESS TEST

Configuration trông đúng chưa đủ.

Phải functional test.

# 73. Test `d1ops`

```bash
sudo -u d1ops cat /srv/day1-lab/reports/report.txt
```

Expected:

```text
Day 1 restricted operational report
```

Đây là positive access test.

---

# 74. Test write by owner

```bash
sudo -u d1ops bash -c \
'echo "verified-by-d1ops" >> /srv/day1-lab/reports/report.txt'
```

Verify:

```bash
tail /srv/day1-lab/reports/report.txt
```

Expected line:

```text
verified-by-d1ops
```

---

# PHASE 16 — NEGATIVE FILE ACCESS TEST

# 75. `d1viewer`

`d1viewer` không thuộc `d1opsgrp`.

Test:

```bash
sudo -u d1viewer cat /srv/day1-lab/reports/report.txt
```

Expected:

```text
Permission denied
```

Nếu đọc được, control không đúng.

Không được nói:

```text
"Permission configured rồi nên chắc chắn secure."
```

Functional denial mới là evidence mạnh.

---

# 76. Capture negative result

Có thể:

```bash
sudo -u d1viewer cat /srv/day1-lab/reports/report.txt \
> ~/d1viewer_read_test.txt 2>&1
```

Sau đó:

```bash
cat ~/d1viewer_read_test.txt
```

Expected:

```text
Permission denied
```

---

# 77. Parent directory importance

Nếu file mode là:

```text
640
```

nhưng parent directory là:

```text
700
```

group user vẫn không access được.

Kiểm tra path:

```bash
ls -ld /srv
ls -ld /srv/day1-lab
ls -ld /srv/day1-lab/reports
```

Đây là troubleshooting checkpoint rất quan trọng.

---

# PHASE 17 — UMASK

# 78. Kiểm tra current `umask`

Dưới current account:

```bash
umask
```

Có thể thấy:

```text
0022
```

hoặc giá trị khác.

Không được giả định.

---

# 79. Controlled test với `d1ops`

Chạy:

```bash
sudo -u d1ops bash -c '
    umask 027
    touch /home/d1ops/umask-file.txt
    mkdir /home/d1ops/umask-dir
'
```

Verify:

```bash
ls -ld /home/d1ops/umask-dir
ls -l /home/d1ops/umask-file.txt
```

Expected với `umask 027`:

File:

```text
640
rw-r-----
```

Directory:

```text
750
rwxr-x---
```

---

# 80. Verify numeric mode

Có thể dùng:

```bash
stat -c '%A %a %U %G %n' \
/home/d1ops/umask-file.txt \
/home/d1ops/umask-dir
```

Expected pattern:

```text
-rw-r----- 640 ...
drwxr-x--- 750 ...
```

---

# 81. Vì sao file không thành `750`?

File base mode thường không yêu cầu execute:

```text
666
```

Directory base:

```text
777
```

Với:

```text
umask 027
```

kết quả:

```text
file → 640
directory → 750
```

---

# PHASE 18 — LEAST-PRIVILEGE SUDO

Đây là một trong những phần quan trọng nhất của Day 1.

Chúng ta sẽ tạo một privilege cực nhỏ, chỉ cho phép `d1ops` đọc **một root-owned lab status file**.

Không cấp full root.

---

# 82. Tạo protected file

```bash
sudo mkdir -p /srv/day1-lab/admin
```

```bash
echo "DAY1_ADMIN_STATUS=OK" \
| sudo tee /srv/day1-lab/admin/status.txt >/dev/null
```

Set ownership:

```bash
sudo chown root:root /srv/day1-lab/admin/status.txt
```

Set permission:

```bash
sudo chmod 600 /srv/day1-lab/admin/status.txt
```

Directory:

```bash
sudo chown root:root /srv/day1-lab/admin
sudo chmod 700 /srv/day1-lab/admin
```

---

# 83. Prove normal user cannot read

```bash
sudo -u d1ops cat /srv/day1-lab/admin/status.txt
```

Expected:

```text
Permission denied
```

Đây là BEFORE evidence.

---

# 84. Xác định executable path

Chạy:

```bash
command -v cat
```

Ví dụ có thể là:

```text
/usr/bin/cat
```

**Phải sử dụng path thực tế của lab.**

Không hard-code nếu output khác.

Trong các command dưới đây tôi giả định:

```text
/usr/bin/cat
```

---

# 85. Tạo sudoers fragment

Dùng:

```bash
sudo visudo -f /etc/sudoers.d/day1-d1ops
```

Trong lab có thể đặt:

```text
d1ops ALL=(root) NOPASSWD: /usr/bin/cat /srv/day1-lab/admin/status.txt
```

`NOPASSWD` ở đây chỉ được dùng để tránh yêu cầu tạo/chia sẻ password cho account training và chỉ grant **một exact read-only lab command**.

Không suy rộng pattern này thành production policy.

Production phải theo organization security policy.

---

# 86. Validate syntax

Sau khi save:

```bash
sudo visudo -cf /etc/sudoers.d/day1-d1ops
```

Expected:

```text
parsed OK
```

Nếu không:

> sửa syntax trước khi test.

---

# 87. Verify sudoers fragment metadata

```bash
sudo ls -l /etc/sudoers.d/day1-d1ops
```

Bạn không cần dump toàn bộ production sudo policy vào evidence.

Chỉ capture lab fragment/context đủ dùng.

---

# 88. Check from `d1ops`

```bash
sudo -u d1ops sudo -l
```

Output phải thể hiện exact command được permit.

---

# 89. Positive sudo test

```bash
sudo -u d1ops sudo -n \
/usr/bin/cat /srv/day1-lab/admin/status.txt
```

Expected:

```text
DAY1_ADMIN_STATUS=OK
```

`-n` nghĩa là non-interactive; nếu password bị yêu cầu, command fail thay vì prompt.

---

# 90. Negative sudo test

Test một command không được grant:

```bash
sudo -u d1ops sudo -n \
/usr/bin/cat /etc/shadow
```

Expected:

```text
not allowed
```

hoặc policy-denied message tương đương.

Điểm quan trọng:

```text
/etc/shadow must NOT be displayed
```

Nếu nó hiển thị, account có privilege rộng hơn dự kiến và lab phải dừng để review.

---

# 91. Đây là chứng minh least privilege

Requirement:

```text
Allowed:
cat /srv/day1-lab/admin/status.txt
```

Denied:

```text
cat /etc/shadow
```

Vậy evidence phải có cả hai.

---

# PHASE 19 — SSH KEY GENERATION

Day 1 yêu cầu generate và configure SSH keys. Infrastructure_Operations_14_Da…

SSH key phải được tạo trên **administrative workstation**, không phải để copy private key lên server.

---

# 92. Chuẩn bị local `.ssh`

Trên workstation:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

---

# 93. Algorithm selection

Phải tuân thủ organization/platform policy.

Trong một lab thông thường, nếu environment cho phép có thể dùng:

```bash
ssh-keygen -t ed25519 \
-f ~/.ssh/day1_rhel_lab \
-C "day1-lab"
```

Nếu môi trường có compliance/FIPS hoặc algorithm policy riêng, dùng algorithm mà policy yêu cầu thay vì ép dùng ví dụ trên.

Khi được hỏi passphrase:

> dùng một local passphrase phù hợp policy và **không chia sẻ nó cho tôi hoặc đưa vào evidence**.

---

# 94. Files được tạo

Ví dụ:

```text
~/.ssh/day1_rhel_lab
```

là private key.

```text
~/.ssh/day1_rhel_lab.pub
```

là public key.

---

# 95. Verify permissions

```bash
ls -l ~/.ssh/day1_rhel_lab*
```

Private key phải được bảo vệ.

Nếu cần:

```bash
chmod 600 ~/.ssh/day1_rhel_lab
```

Public key có thể readable hơn vì không phải secret theo cùng nghĩa.

---

# 96. Fingerprint

```bash
ssh-keygen -lf ~/.ssh/day1_rhel_lab.pub
```

Evidence nên lưu **fingerprint** thay vì private key content.

Ví dụ evidence:

```text
Key purpose: Day 1 Lab
Algorithm: <actual output>
Fingerprint: SHA256:...
Private key: NOT INCLUDED
```

---

# PHASE 20 — CONFIGURE SSH PUBLIC KEY

Có nhiều bootstrap methods tùy environment.

Không thay `sshd_config` ở Day 1.

---

# 97. Method A — `ssh-copy-id`

Nếu server cho phép authorized bootstrap authentication và procedure cho phép:

```bash
ssh-copy-id \
-i ~/.ssh/day1_rhel_lab.pub \
d1ops@LAB_HOST
```

Phương thức này có thể yêu cầu existing authentication.

Nếu password login bị disable thì method này có thể không phù hợp.

---

# 98. Method B — Admin-controlled key installation

Nếu bạn đã SSH vào server bằng existing `labadmin`, public key có thể được provision theo approved procedure.

Trên server, tạo:

```bash
sudo install -d \
-m 700 \
-o d1ops \
-g d1ops \
/home/d1ops/.ssh
```

Sau đó public key được đưa vào:

```text
/home/d1ops/.ssh/authorized_keys
```

bằng approved secure method.

Không paste private key.

---

# 99. Set ownership

```bash
sudo chown -R d1ops:d1ops /home/d1ops/.ssh
```

Ở đây recursive operation chỉ tác động dedicated `.ssh` directory vừa tạo cho lab account.

Trong production, recursive `chown` luôn cần review scope.

---

# 100. Directory permission

```bash
sudo chmod 700 /home/d1ops/.ssh
```

---

# 101. `authorized_keys`

```bash
sudo chmod 600 /home/d1ops/.ssh/authorized_keys
```

Verify:

```bash
sudo ls -ld /home/d1ops/.ssh
sudo ls -l /home/d1ops/.ssh/authorized_keys
```

Expected pattern:

```text
drwx------ d1ops d1ops .ssh
-rw------- d1ops d1ops authorized_keys
```

Exact timestamps/sizes khác nhau.

---

# PHASE 21 — POSITIVE SSH VERIFICATION

Trên workstation:

```bash
ssh \
-i ~/.ssh/day1_rhel_lab \
d1ops@LAB_HOST
```

Nếu host key prompt xuất hiện lần đầu, phải xác minh fingerprint theo approved lab information trước khi accept.

Không auto-accept một hostname lạ.

---

# 102. Sau login

Chạy:

```bash
whoami
hostname
id
pwd
```

Expected:

```text
whoami → d1ops
hostname → lab server
id → expected UID/groups including d1opsgrp
pwd → /home/d1ops
```

---

# 103. One-command verification

Có thể từ workstation:

```bash
ssh \
-i ~/.ssh/day1_rhel_lab \
d1ops@LAB_HOST \
'whoami; hostname; id'
```

Output là evidence rất tốt vì chứng minh:

```text
SSH authentication succeeded
+
correct identity
+
correct host
+
correct group membership
```

---

# PHASE 22 — NEGATIVE SSH VERIFICATION

Syllabus yêu cầu verify **authorized and denied access**. Infrastructure_Operations_14_Da…

Ta sẽ không phá public key hợp lệ.

Thay vào đó tạo một second key chưa được authorize.

---

# 104. Generate unapproved key

Trên workstation:

```bash
ssh-keygen -t ed25519 \
-f ~/.ssh/day1_unapproved \
-C "day1-unapproved-test"
```

Nếu environment policy yêu cầu algorithm khác, dùng policy-approved algorithm.

Private key vẫn phải được bảo vệ.

---

# 105. Force client chỉ sử dụng unapproved key

```bash
ssh \
-o IdentitiesOnly=yes \
-o PreferredAuthentications=publickey \
-o PasswordAuthentication=no \
-o KbdInteractiveAuthentication=no \
-i ~/.ssh/day1_unapproved \
d1ops@LAB_HOST
```

Expected:

```text
Permission denied
```

Thông điệp cụ thể có thể là:

```text
Permission denied (publickey)
```

hoặc server có danh sách methods khác.

Điểm cần chứng minh:

```text
Unapproved key cannot authenticate.
```

---

# 106. Tại sao dùng `IdentitiesOnly=yes`?

Nếu không, SSH agent có thể thử một authorized key khác.

Lúc đó negative test có thể:

```text
unexpectedly succeed
```

mặc dù key bạn muốn test không được authorize.

`IdentitiesOnly=yes` giúp test đúng identity file được chỉ định.

---

# PHASE 23 — SSH DEBUGGING

Nếu positive authentication fail:

```bash
ssh -v \
-o IdentitiesOnly=yes \
-i ~/.ssh/day1_rhel_lab \
d1ops@LAB_HOST
```

Không vội sửa configuration.

Phân tích stage.

---

# 107. Nếu thấy timeout

Có thể:

```text
Network / route / firewall / port
```

Không phải key issue.

Day 1:

```text
collect evidence
escalate if network change required
```

---

# 108. Nếu thấy host-key mismatch

Không chạy ngay:

```bash
ssh-keygen -R LAB_HOST
```

chỉ để bỏ warning.

Phải xác minh:

```text
server rebuild?
IP reused?
host key rotated?
wrong server?
```

Nếu chưa xác minh:

> escalate.

---

# 109. Nếu thấy key offered but rejected

Kiểm tra:

```bash
sudo ls -ld /home/d1ops
sudo ls -ld /home/d1ops/.ssh
sudo ls -l /home/d1ops/.ssh/authorized_keys
```

Kiểm tra ownership:

```text
d1ops:d1ops
```

Kiểm tra đúng public key/fingerprint.

Không chỉnh `sshd_config` Day 1.

---

# PHASE 24 — COMMAND DOCUMENTATION

Syllabus yêu cầu document executed commands. Infrastructure_Operations_14_Da…

# 110. Minimum command record

Mỗi meaningful operation nên có:

```text
Timestamp
Hostname
User
Purpose
Command
Expected result
Actual result
Verification
Status
Ticket/change reference
```

---

# 111. Ví dụ record

```text
Timestamp:
2026-10-18T22:15:00+07:00

Host:
rhel-lab01

Operator:
labadmin

Purpose:
Grant d1ops read-only sudo access to lab status file

Change:
Created /etc/sudoers.d/day1-d1ops

Verification:
visudo -cf /etc/sudoers.d/day1-d1ops

Positive Test:
d1ops can read /srv/day1-lab/admin/status.txt via sudo

Negative Test:
d1ops cannot read /etc/shadow via sudo

Result:
PASS
```

---

# 112. Capturing command output

Pattern:

```bash
{
    echo "Timestamp:"
    date -Is

    echo "Hostname:"
    hostname

    echo "User:"
    whoami

    echo "Verification:"
    id d1ops
} | tee evidence.txt
```

---

# 113. stderr evidence

Đối với negative test:

```bash
command > evidence.txt 2>&1
```

sẽ capture stdout và stderr.

Ví dụ:

```bash
sudo -u d1viewer cat /srv/day1-lab/reports/report.txt \
> ~/negative_permission_test.txt 2>&1
```

---

# PHASE 25 — COMPLETE ACCESS VERIFICATION MATRIX

Bạn nên có matrix cuối cùng.

| Test                            | Expected | Result |
| ------------------------------- | -------- | ------ |
| `d1ops` exists                  | Yes      |        |
| `d1viewer` exists               | Yes      |        |
| `d1opsgrp` exists               | Yes      |        |
| `d1ops` member of `d1opsgrp`    | Yes      |        |
| `d1viewer` member of `d1opsgrp` | No       |        |
| `d1ops` reads report            | Allowed  |        |
| `d1viewer` reads report         | Denied   |        |
| `d1ops` approved sudo command   | Allowed  |        |
| `d1ops` unapproved sudo command | Denied   |        |
| Approved SSH key                | Allowed  |        |
| Unapproved SSH key              | Denied   |        |

Nếu bất kỳ negative test nào unexpectedly succeeds:

> lab security objective = FAIL cho đến khi root cause được xác định.

---

# PHASE 26 — CONFIGURATION EVIDENCE

# 114. Users/groups

```bash
getent passwd d1ops
getent passwd d1viewer
getent group d1opsgrp
id d1ops
id d1viewer
```

---

# 115. Ownership/permissions

```bash
ls -ld /srv/day1-lab
ls -ld /srv/day1-lab/reports
ls -l /srv/day1-lab/reports/report.txt
stat /srv/day1-lab/reports/report.txt
```

---

# 116. `umask`

Capture:

```bash
sudo -u d1ops bash -c '
    umask 027
    umask
'
```

và creation result.

---

# 117. `sudo`

```bash
sudo -u d1ops sudo -l
```

plus positive/negative tests.

Không capture toàn bộ confidential enterprise sudo configuration nếu không cần.

---

# 118. SSH

Capture:

```text
Public-key fingerprint
authorized_keys ownership
authorized_keys mode
positive login identity
negative login result
```

Không capture:

```text
private key
passphrase
```

---

# PHASE 27 — FAILURE INJECTION 1: WRONG FILE PERMISSION

Failure Injection chỉ áp dụng trong lab.

# 119. Inject

Hiện report:

```bash
ls -l /srv/day1-lab/reports/report.txt
```

Backup current mode evidence.

Sau đó intentionally remove group read:

```bash
sudo chmod 600 /srv/day1-lab/reports/report.txt
```

---

# 120. Symptom

Test một group-authorized identity nếu bạn có một separate group reader.

Với current setup `d1ops` là owner nên vẫn đọc được.

Để tạo group-reader scenario an toàn, add `d1viewer` tạm thời vào group:

```bash
sudo usermod -aG d1opsgrp d1viewer
```

Fresh test:

```bash
sudo -u d1viewer cat /srv/day1-lab/reports/report.txt
```

Với mode `600`, expected:

```text
Permission denied
```

---

# 121. Troubleshoot

Inspect:

```bash
id d1viewer
```

```bash
ls -ld /srv/day1-lab/reports
```

```bash
ls -l /srv/day1-lab/reports/report.txt
```

Reasoning:

```text
Group membership correct
Parent traverse correct
File group correct
But group read bit absent
→ root cause identified
```

---

# 122. Correct

Restore:

```bash
sudo chmod 640 /srv/day1-lab/reports/report.txt
```

Retest:

```bash
sudo -u d1viewer cat /srv/day1-lab/reports/report.txt
```

Expected:

```text
success
```

Sau Failure Injection, remove temporary membership để trở lại baseline:

```bash
sudo gpasswd -d d1viewer d1opsgrp
```

Verify:

```bash
id d1viewer
```

Retest expected denial.

---

# PHASE 28 — FAILURE INJECTION 2: MISSING GROUP MEMBERSHIP

# 123. Inject

Requirement giả định:

```text
d1viewer should temporarily have group access
```

Add:

```bash
sudo usermod -aG d1opsgrp d1viewer
```

Verify success.

Sau đó intentionally remove:

```bash
sudo gpasswd -d d1viewer d1opsgrp
```

---

# 124. Symptom

```bash
sudo -u d1viewer cat /srv/day1-lab/reports/report.txt
```

Expected:

```text
Permission denied
```

---

# 125. Investigation order

Đừng chạy `chmod 777`.

Chạy:

```bash
id d1viewer
```

Sau đó:

```bash
ls -l /srv/day1-lab/reports/report.txt
```

Bạn sẽ thấy:

```text
file group = d1opsgrp
group permission = r--
d1viewer not member of d1opsgrp
```

Root cause:

```text
identity/group membership
```

không phải file mode.

---

# 126. Fix

Nếu requirement cần access:

```bash
sudo usermod -aG d1opsgrp d1viewer
```

Verify:

```bash
id d1viewer
```

Retest.

Sau Failure Injection hoàn tất, restore intended Day 1 baseline bằng cách remove membership.

---

# PHASE 29 — FAILURE INJECTION 3: BAD SUDO POLICY CANDIDATE

Không inject syntax error trực tiếp vào active `/etc/sudoers`.

Ta kiểm tra một **offline candidate file**.

# 127. Tạo invalid candidate

```bash
cat > ~/bad-sudoers-test <<'EOF'
this is not valid sudoers syntax
EOF
```

Validate:

```bash
sudo visudo -cf ~/bad-sudoers-test
```

Expected:

```text
syntax error
```

---

# 128. Lesson

Correct operator response:

```text
Validation failed
→ Do not deploy
```

Sai response:

```text
Copy it into /etc/sudoers.d and see what happens
```

Đây chính là operational safety.

---

# PHASE 30 — FAILURE INJECTION 4: UNAUTHORIZED SSH KEY

Negative SSH key test ở trên chính là một failure injection an toàn.

Expected:

```text
Approved key → success
Unapproved key → authentication failure
```

Troubleshooting phải chứng minh:

```text
Network works
Host identified
Username correct
But selected public key is not authorized
```

Không cần phá server configuration.

---

# PHASE 31 — FAILURE INJECTION 5: WRONG RELATIVE PATH

# 129. Scenario

Từ:

```bash
cd ~/day1-lab/input
```

thử:

```bash
cat output/server_counts.txt
```

Expected:

```text
No such file or directory
```

Vì shell resolve thành:

```text
~/day1-lab/input/output/server_counts.txt
```

không phải:

```text
~/day1-lab/output/server_counts.txt
```

---

# 130. Troubleshoot

```bash
pwd
```

```bash
find ~/day1-lab -name server_counts.txt -print
```

Correct relative path:

```bash
cat ../output/server_counts.txt
```

Lesson:

> path problems thường là context problems, không phải permission problems.

---

# PHASE 32 — TROUBLESHOOTING CHECKPOINTS

## Permission denied

Investigation order:

```text
Correct user?
    ↓
Correct groups?
    ↓
Correct object?
    ↓
Correct owner/group?
    ↓
Correct file mode?
    ↓
Correct parent directory mode?
    ↓
Other control outside Day 1?
```

Commands:

```bash
whoami
id USER
ls -ld /path
ls -l /path/file
stat /path/file
```

Nếu tất cả DAC state đúng mà vẫn deny:

```text
Do not disable security controls.
Collect evidence.
Escalate to investigate ACL/SELinux/other policy.
```

---

# 131. SSH troubleshooting checkpoint

```text
Correct host?
 ↓
TCP connection succeeds?
 ↓
Host key valid?
 ↓
Correct username?
 ↓
Correct private key selected?
 ↓
Public key authorized?
 ↓
Correct ownership?
 ↓
Correct permissions?
 ↓
Server-side policy?
```

Day 1 dừng trước server-wide policy modification.

---

# 132. `sudo` troubleshooting checkpoint

```text
Correct user?
 ↓
sudo available?
 ↓
sudo -l shows rule?
 ↓
Exact executable path?
 ↓
Exact arguments?
 ↓
Syntax valid?
 ↓
Positive test?
 ↓
Negative test?
```

Không giải quyết bằng:

```text
grant ALL=(ALL) ALL
```

chỉ vì rule cụ thể chưa chạy.

---

# PHASE 33 — EVIDENCE PACKAGE

Final evidence package nên có cấu trúc:

```text
Day01/
├── 00_admin/
│   ├── access_checklist.txt
│   └── learning_baseline.txt
│
├── 01_inventory/
│   └── system_inventory.txt
│
├── 02_cli/
│   ├── navigation.txt
│   ├── file_operations.txt
│   ├── searching.txt
│   └── text_processing.txt
│
├── 03_users_groups/
│   └── users_groups.txt
│
├── 04_permissions/
│   ├── before.txt
│   ├── after.txt
│   ├── access_positive.txt
│   ├── access_negative.txt
│   └── umask.txt
│
├── 05_sudo/
│   ├── sudo_rule_summary.txt
│   ├── sudo_positive.txt
│   └── sudo_negative.txt
│
├── 06_ssh/
│   ├── key_fingerprint.txt
│   ├── ssh_permissions.txt
│   ├── ssh_positive.txt
│   └── ssh_negative.txt
│
├── 08_failures/
│   ├── permission_failure.txt
│   ├── group_failure.txt
│   ├── sudo_validation_failure.txt
│   └── ssh_key_failure.txt
│
└── 09_final/
    ├── day1_summary.txt
    └── completion_checklist.txt
```

Không đặt private key vào repository này.

---

# PHASE 34 — EVIDENCE QUALITY CHECK

Mỗi artifact nên trả lời:

```text
Which host?
Which user?
When?
What was tested?
What command?
What result?
Was result expected?
PASS or FAIL?
```

Một file chỉ chứa:

```text
Permission denied
```

là evidence yếu.

Tốt hơn:

```text
Timestamp:
Host:
Identity:
Test:
Expected:
Actual:
Result:
```

---

# PHASE 35 — CLEANUP / ROLLBACK

Cleanup cũng là một phần của operations.

Không được làm "lab xong rồi để nguyên".

---

# 133. Trước cleanup

Capture final configuration evidence trước.

Sau đó xác nhận:

```bash
getent passwd d1ops
getent passwd d1viewer
getent group d1opsgrp
sudo ls -ld /srv/day1-lab
```

---

# 134. Remove sudo rule

Trước:

```bash
sudo visudo -cf /etc/sudoers.d/day1-d1ops
```

Nếu lab kết thúc và rule chỉ dành cho lab:

```bash
sudo rm /etc/sudoers.d/day1-d1ops
```

Verify:

```bash
sudo -u d1ops sudo -l
```

Expected:

lab grant không còn.

Lưu ý account có thể còn các privileges khác nếu environment đã cấu hình sẵn; không được assume.

---

# 135. Remove lab filesystem

Chỉ sau khi xác nhận path:

```bash
sudo ls -la /srv/day1-lab
```

Nếu đúng dedicated lab directory:

```bash
sudo rm -r /srv/day1-lab
```

Không dùng:

```bash
sudo rm -rf /srv
```

---

# 136. Check user processes

Trước khi delete account:

```bash
pgrep -u d1ops
pgrep -u d1viewer
```

Nếu có active process:

> xác định trước khi xóa account.

Không kill blindly.

---

# 137. Delete lab users

Chỉ với accounts được tạo riêng cho lab:

```bash
sudo userdel -r d1ops
```

```bash
sudo userdel -r d1viewer
```

`-r` có thể xóa home/mail data của account.

Do đó command này chỉ an toàn khi bạn đã xác nhận account/home là disposable lab objects.

---

# 138. Verify deletion

```bash
getent passwd d1ops
getent passwd d1viewer
```

Expected:

```text
no result
```

---

# 139. Delete group

Sau khi users không còn:

```bash
sudo groupdel d1opsgrp
```

Verify:

```bash
getent group d1opsgrp
```

Expected:

```text
no result
```

---

# 140. Remove temporary SSH keys

Nếu keys chỉ dùng cho lab và policy yêu cầu cleanup:

Trên workstation:

```bash
rm ~/.ssh/day1_rhel_lab
rm ~/.ssh/day1_rhel_lab.pub
rm ~/.ssh/day1_unapproved
rm ~/.ssh/day1_unapproved.pub
```

Chỉ chạy sau khi:

```bash
ls -l ~/.ssh/day1_*
```

và xác nhận đúng files.

Không dùng wildcard nếu có nguy cơ match unrelated keys.

Ví dụ an toàn hơn:

```bash
rm ~/.ssh/day1_rhel_lab ~/.ssh/day1_rhel_lab.pub
```

---

# 141. Evidence không xóa

Giữ:

```text
fingerprints
sanitized command output
configuration evidence
test results
```

nhưng không giữ:

```text
private key
passphrase
password
```

---

# PHASE 36 — FINAL VERIFICATION

Sau cleanup:

```bash
getent passwd d1ops
getent passwd d1viewer
getent group d1opsgrp
```

Lab path:

```bash
ls -ld /srv/day1-lab
```

Expected:

```text
No such file or directory
```

Sudo rule:

```bash
sudo ls -l /etc/sudoers.d/day1-d1ops
```

Expected:

```text
No such file or directory
```

Chỉ coi cleanup complete sau verification.

---

# PHASE 37 — ROLLBACK PHILOSOPHY

Trong lab này, mỗi change phải có rollback rõ.

| Change                           | Rollback                                       |
| -------------------------------- | ---------------------------------------------- |
| Create file                      | remove lab file                                |
| Create directory                 | remove dedicated lab directory                 |
| Change `chmod`                   | restore recorded previous mode                 |
| Change `chown`                   | restore recorded previous owner/group          |
| Add user to group                | remove membership                              |
| Create user                      | `userdel` after verifying disposable account   |
| Create group                     | `groupdel`                                     |
| Add sudo fragment                | remove dedicated fragment                      |
| Add SSH public key               | remove only corresponding authorized key entry |
| Generate temporary local SSH key | remove temporary key when no longer required   |

Điểm quan trọng:

> rollback không nên dựa vào trí nhớ.

Bạn phải biết **before state**.

---

# PHASE 38 — ESCALATION BOUNDARIES

Trong Day 1, bạn nên dừng và escalate nếu gặp các trường hợp như:

### SSH

```text
Host key unexpectedly changed
Need sshd_config modification
Need firewall modification
Need network route change
Need PAM change
```

### Permissions

```text
Recursive permission change affects production tree
Shared filesystem involved
Application owner unclear
ACL or SELinux suspected
Current baseline unknown
```

### Users

```text
Existing production account would be modified
UID/GID conflict
Central identity source involved
Account ownership unclear
```

### Sudo

```text
Required rule would effectively grant unrestricted root
Complex wildcard required
Shell-capable command would be privileged
Production sudoers policy unclear
```

### Evidence/security

```text
Credential exposure
Private key exposure
Unexpected sensitive data discovered
```

---

# PHASE 39 — LAB TROUBLESHOOTING RECORD TEMPLATE

Khi có lỗi, document:

```text
Incident/Exercise ID:

Timestamp:

Host:

Environment:

User:

Expected behavior:

Actual behavior:

Exact error:

Recent change:

Read-only checks:

Observed state:

Hypothesis:

Controlled test:

Result:

Corrective action:

Verification:

Rollback available:

Escalation required:

Evidence files:
```

Đây là habit rất quan trọng cho những Day sau.

---

# PHASE 40 — WHAT YOU SHOULD NOT DO

Day 1 lab không yêu cầu bạn:

```text
disable SELinux
disable firewall
edit sshd_config
restart sshd randomly
reboot server to "fix" access
set chmod 777
grant unrestricted sudo
use shared root credentials
copy private key to server
put passwords in scripts
delete system users
modify production files
```

Nếu một tutorial trên Internet bảo làm như vậy để "cho chạy", không áp dụng mù quáng.

---

# PHASE 41 — DAY 1 PRACTICAL TROUBLESHOOTING MINDSET

Khi một test fail, tư duy:

```text
Symptom
  ↓
Context
  ↓
Identity
  ↓
Configuration
  ↓
Dependency
  ↓
Expected behavior
  ↓
Difference
  ↓
Root cause hypothesis
  ↓
Smallest approved correction
  ↓
Verification
  ↓
Evidence
```

Không:

```text
Symptom
 ↓
Google random command
 ↓
sudo everything
 ↓
reboot
```

---

# PHASE 42 — EXPECTED FINAL SKILLS

Sau lab, bạn phải tự làm được:

### Workstation

- kiểm tra SSH client;
- tổ chức evidence repository;
- bảo vệ SSH private key;
- thu thập fingerprint.

### Linux CLI

- navigate directory tree;
- phân biệt absolute/relative path;
- create/copy/move/delete files;
- tìm files;
- tìm text;
- compose simple pipelines;
- redirect output.

### Identity

- query users/groups;
- tạo lab user/group;
- assign supplementary group;
- verify UID/GID/group membership.

### Permissions

- read `ls -l`;
- sử dụng `chmod`;
- sử dụng `chown`;
- sử dụng `chgrp`;
- hiểu file vs directory permission;
- test authorized/denied access;
- sử dụng `umask`.

### Privilege

- đọc `sudo -l`;
- hiểu exact limited sudo command;
- validate sudoers bằng `visudo`;
- prove permitted command;
- prove prohibited command is denied.

### SSH

- generate key pair;
- protect private key;
- install public key;
- verify `.ssh` ownership/mode;
- authenticate bằng key;
- debug bằng `ssh -v`;
- prove unapproved key is denied.

### Operations

- capture before/after state;
- collect evidence;
- rollback;
- cleanup;
- know when to escalate.

---

# PHASE 43 — PRACTICAL COMPLETION CHECKLIST

Bạn chỉ nên đánh dấu Day 1 Practical complete khi toàn bộ nội dung dưới đây đã có evidence:

```text
ADMINISTRATION
[ ] Workstation prepared
[ ] Approved target confirmed
[ ] Access checklist complete
[ ] Evidence repository created
[ ] Personal learning baseline recorded

INVENTORY
[ ] Hostname captured
[ ] OS captured
[ ] Kernel captured
[ ] User identity captured
[ ] Basic system inventory captured

CLI
[ ] pwd
[ ] ls
[ ] cd
[ ] absolute paths
[ ] relative paths
[ ] touch
[ ] mkdir
[ ] cp
[ ] mv
[ ] rm
[ ] find
[ ] grep
[ ] pipe
[ ] >
[ ] >>
[ ] basic text processing

USERS/GROUPS
[ ] Controlled group created
[ ] Controlled users created
[ ] Group membership configured
[ ] Membership verified

PERMISSIONS
[ ] Ownership verified
[ ] chown tested
[ ] chgrp tested
[ ] chmod numeric tested
[ ] chmod symbolic tested
[ ] umask tested
[ ] Positive file access verified
[ ] Negative file access verified

SUDO
[ ] Baseline access tested
[ ] Least-privilege rule configured
[ ] sudoers validated
[ ] Approved sudo command succeeds
[ ] Unapproved sudo command denied

SSH
[ ] Approved key pair generated
[ ] Private key protected
[ ] Fingerprint documented
[ ] Public key installed
[ ] authorized_keys owner verified
[ ] authorized_keys permission verified
[ ] Positive SSH authentication verified
[ ] Unapproved key denied
[ ] No private key included in evidence

TROUBLESHOOTING
[ ] Permission failure investigated
[ ] Group membership failure investigated
[ ] Invalid sudo candidate safely detected
[ ] SSH negative test investigated
[ ] Path failure investigated

EVIDENCE
[ ] Before-state evidence
[ ] After-state evidence
[ ] Positive verification evidence
[ ] Negative verification evidence
[ ] Commands documented
[ ] Timestamps included
[ ] Hostname included
[ ] Identity included
[ ] No credentials exposed

CLEANUP
[ ] Temporary sudo rule removed
[ ] Lab directory removed
[ ] Lab users removed if required
[ ] Lab group removed if required
[ ] Temporary SSH keys removed if required
[ ] Final state verified
```

---

# PHASE 44 — DELIVERABLE CUỐI DAY 1

Theo đúng syllabus, ba nhóm deliverable quan trọng nhất phải là **system inventory, access verification và configuration evidence**. Infrastructure_Operations_14_Da…

Một final submission tốt có thể tóm tắt:

```text
DAY 1 LAB REPORT

1. Environment
   - Host
   - OS
   - Kernel
   - Operator
   - Lab scope

2. Administrative Readiness
   - Access authorization
   - Workstation readiness
   - Evidence repository

3. CLI Exercises
   - Navigation
   - Files/directories
   - Copy/move/delete
   - Search
   - Text processing

4. Identity Configuration
   - Users
   - Groups
   - Membership

5. Filesystem Security
   - Ownership
   - Permissions
   - umask
   - Positive/negative tests

6. Privileged Access
   - sudo requirement
   - Rule
   - Syntax validation
   - Allowed test
   - Denied test

7. SSH
   - Public-key fingerprint
   - Server-side authorization
   - Positive login
   - Negative login

8. Failure Injection
   - Symptoms
   - Investigation
   - Root cause
   - Correction
   - Verification

9. Cleanup
   - Objects removed
   - Final verification

10. Evidence Index
   - Artifact filenames
   - Timestamps
   - Test status
```

---

# Day 1 — Operational Flow cần thuộc về tư duy

Nếu chỉ nhớ **một workflow** từ toàn bộ Module Thực hành Day 1, hãy nhớ:

```text
              REQUEST
                 │
                 ▼
         Confirm Authorization
                 │
                 ▼
        Confirm Host / Identity
                 │
                 ▼
          Capture Baseline
                 │
                 ▼
         Read-only Inspection
                 │
                 ▼
        Smallest Safe Change
                 │
                 ▼
      Configuration Verification
                 │
                 ▼
       Functional Positive Test
                 │
                 ▼
       Functional Negative Test
                 │
                 ▼
          Collect Evidence
                 │
          ┌──────┴──────┐
          ▼             ▼
       Success        Failure
          │             │
          ▼             ▼
       Document      Troubleshoot
          │             │
          │        Within scope?
          │        ┌────┴────┐
          │       Yes        No
          │        │          │
          │        ▼          ▼
          │     Correct     Escalate
          │        │
          └────────┘
              │
              ▼
       Cleanup / Rollback
              │
              ▼
       Final Verification
              │
              ▼
             CLOSE
```

Đây mới là mục tiêu thật sự của Day 1: không phải chỉ học `chmod`, `useradd`, `sudo` hay `ssh-keygen`, mà hình thành khả năng **thực hiện một administrative change có kiểm soát từ đầu đến cuối**, gồm authorization, baseline, implementation, security verification, troubleshooting, evidence, rollback và escalation. Đây chính là foundation để bước sang Day 2, nơi cùng operational discipline này sẽ được áp dụng cho packages, processes, `systemd`, SELinux, `firewalld`, networking, storage, logs, Bash, backup và troubleshooting Linux sâu hơn. Infrastructure_Operations_14_Da…
