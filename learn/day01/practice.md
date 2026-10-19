# Day 1 — Module Thực hành

## Linux/RHEL Administrative Foundation Lab

Module Thực hành Day 1 được xây trực tiếp từ Assignment/Lab của syllabus. Phạm vi bắt buộc gồm: chuẩn bị `administrative workstation`, `approved access checklist`, Linux/RHEL lab environment, evidence repository và personal learning baseline; thực hành CLI navigation, file creation/copy/move/delete, searching và text processing; tạo/quản lý controlled users và groups; cấu hình ownership/permissions; thiết lập least-privilege `sudo`; tạo và cấu hình SSH keys; kiểm thử cả authorized và denied access; document commands; cuối cùng nộp system inventory, access verification và configuration evidence. Infrastructure_Operations_14_Da…

Toàn bộ phần dưới mặc định chạy trên **lab/non-production/authorized environment**. Syllabus cũng nhấn mạnh rằng các hoạt động có rủi ro cao, privileged operations và production changes phải có authorization, rollback planning và supervision thích hợp. Infrastructure_Operations_14_Da…

---

# 1. Mục tiêu cuối cùng của Module Thực hành

Sau khi hoàn thành toàn bộ lab, bạn không chỉ cần chứng minh rằng:

```text
"Tôi biết chạy Linux commands."
```

Bạn phải chứng minh được workflow:

```text
Prepare
   ↓
Identify
   ↓
Baseline
   ↓
Execute
   ↓
Verify
   ↓
Test allowed behavior
   ↓
Test denied behavior
   ↓
Troubleshoot
   ↓
Collect evidence
   ↓
Rollback / Cleanup
   ↓
Close / Escalate
```

Acceptance level mong muốn là bạn có thể tự thực hiện một task kiểu:

> Một controlled Linux user cần được tạo, thêm vào đúng group, được truy cập đúng resource, bị từ chối với resource không được phép, có một `sudo` privilege rất giới hạn và có SSH public-key authentication; toàn bộ operation phải có evidence và rollback.

---

# 2. Lab Topology

Chúng ta sử dụng topology logic:

```text
+-------------------------------------+
| Administrative Workstation          |
|                                     |
| Terminal / Bash                     |
| OpenSSH client                      |
| SSH private key                     |
| Evidence repository                 |
+------------------+------------------+
                   |
                   | SSH
                   |
                   v
+-------------------------------------+
| rhel-lab01                          |
| Linux / RHEL lab VM                 |
|                                     |
| Existing admin/student account      |
| sudo privilege                      |
|                                     |
| Controlled users:                   |
|   ops1                              |
|   audit1                            |
|                                     |
| Controlled group:                   |
|   day1ops                           |
|                                     |
| Lab data:                           |
|   /srv/day1lab                      |
+-------------------------------------+
```

Tên ở đây chỉ là lab convention.

Nếu environment của bạn khác, thay:

```text
rhel-lab01
student
ops1
audit1
day1ops
```

bằng approved lab names.

Không dùng production usernames hoặc tài khoản người thật để thực hành.

---

# 3. Quy ước risk cho commands

Trong toàn bộ lab, tôi sẽ sử dụng ba mức:

```text
[RO]
Read-only / verification

[CHANGE]
Thay đổi state/configuration

[DESTRUCTIVE]
Có khả năng remove data/account/configuration
```

Ví dụ:

```bash
# [RO]
whoami

# [CHANGE]
chmod 640 file.txt

# [DESTRUCTIVE]
rm file.txt
```

`DESTRUCTIVE` không có nghĩa command bị cấm. Nó có nghĩa bạn phải biết chính xác:

```text
target
scope
impact
rollback
authorization
```

trước khi execute.

---

# 4. LAB 0 — Safety Gate và Lab Readiness

## Objective

Xác minh rằng bạn đang đứng đúng environment trước khi thực hiện bất kỳ configuration change nào.

---

## 4.1 Prerequisites

Bạn cần có:

- một Linux/RHEL VM hoặc authorized Linux lab;
- một existing administrative account;
- `sudo` access thích hợp;
- SSH client;
- network connectivity tới lab;
- không dùng production secrets;
- đủ disk space cho evidence;
- quyền tạo controlled lab users/groups;
- quyền chỉnh `/etc/sudoers.d/` trong lab nếu thực hiện phần `sudo`.

Nếu bạn không có một trong các quyền trên, **không bypass**. Ghi nhận limitation và thực hiện phần có thể làm.

---

# 5. Xác định target trước khi làm gì khác

Trên Linux server:

```bash
# [RO]
hostname
```

Tiếp:

```bash
# [RO]
hostnamectl
```

Kiểm tra current identity:

```bash
# [RO]
whoami
```

```bash
# [RO]
id
```

Kiểm tra current directory:

```bash
# [RO]
pwd
```

Kiểm tra timestamp:

```bash
# [RO]
date -Is
```

`date -Is` thường cho output kiểu:

```text
2026-10-19T09:30:15+07:00
```

Timestamps là phần quan trọng của evidence.

---

# 6. Expected Result

Bạn phải biết rõ:

```text
Hostname:
Environment:
Current user:
UID:
Primary group:
Supplementary groups:
Current directory:
Current timestamp/timezone:
```

Ví dụ conceptual:

```text
Hostname        : rhel-lab01
Environment     : LAB
User            : student
UID             : 1000
Primary group   : student
sudo access     : authorized lab admin
```

Không cần output giống ví dụ.

---

# 7. Safety stop conditions

Không tiếp tục configuration-changing labs nếu phát hiện:

```text
hostname không phải lab target
production marker
account không đúng
không biết environment
sudo privilege không được authorize
system thuộc người/team khác
```

Đây không phải "lab failure".

Đó chính là correct operational behavior.

---

# 8. LAB 1 — Administrative Workstation Preparation

## Objective

Chuẩn bị workstation dùng để:

```text
SSH
evidence collection
command documentation
key storage
lab notes
```

---

# 9. Kiểm tra SSH client

Trên administrative workstation:

```bash
# [RO]
ssh -V
```

Output có thể tương tự:

```text
OpenSSH_...
```

Version cụ thể phụ thuộc operating system.

Không cần cố làm version giống ví dụ.

---

# 10. Kiểm tra user/workstation identity

```bash
# [RO]
whoami
hostname
date -Is
```

Bạn cần phân biệt:

```text
Administrative workstation hostname
```

với:

```text
Target server hostname
```

Đây là hai systems khác nhau.

---

# 11. Tạo evidence repository

Trên workstation:

```bash
# [CHANGE]
mkdir -p ~/day1_evidence/{00_access,01_inventory,02_cli,03_files,04_users_groups,05_permissions,06_sudo,07_ssh,08_failures,09_cleanup}
```

Kiểm tra:

```bash
# [RO]
find ~/day1_evidence -maxdepth 1 -type d | sort
```

Expected structure:

```text
day1_evidence/
├── 00_access
├── 01_inventory
├── 02_cli
├── 03_files
├── 04_users_groups
├── 05_permissions
├── 06_sudo
├── 07_ssh
├── 08_failures
├── 09_cleanup
```

---

# 12. Vì sao evidence phải chia structure?

Nếu bạn lưu:

```text
screenshot1.png
screenshot2.png
output1.txt
final.txt
```

sau vài ngày rất khó biết file nào chứng minh điều gì.

Evidence structure tốt phải cho reviewer hiểu:

```text
what evidence belongs to what control/task
```

---

# 13. Evidence naming convention

Một naming convention hữu ích:

```text
YYYYMMDD_HHMM_<host>_<topic>_<description>.txt
```

Ví dụ:

```text
20261019_0930_rhel-lab01_inventory_os.txt
20261019_0940_rhel-lab01_permission_before.txt
20261019_0942_rhel-lab01_permission_after.txt
```

Không bắt buộc phải dùng format này.

Điểm quan trọng là consistency.

---

# 14. Đừng lưu secrets vào evidence

Không lưu:

```text
private SSH key
password
access token
session token
secret key
recovery code
```

Public SSH key và fingerprint có thể phù hợp với evidence tùy policy.

Private SSH key thì không.

---

# 15. LAB 2 — Approved Access Checklist

## Objective

Chứng minh rằng bạn đã xác minh access trước khi thực hiện administration.

---

# 16. Access checklist cần trả lời

Tạo một note, ví dụ:

```text
Lab ID:
Date:
Operator:
Source workstation:
Target hostname:
Environment:
Approved account:
Expected privilege:
Access method:
Change scope:
Rollback available:
Evidence location:
```

Ví dụ:

```text
Lab ID: DAY1-LAB
Operator: student
Target: rhel-lab01
Environment: non-production lab
Access method: SSH
Privilege: authorized sudo for lab
Scope: Day 1 controlled Linux administration
```

Đây là evidence về operational readiness chứ không phải technical command output.

---

# 17. Verify SSH target

Nếu DNS hostname được dùng:

```bash
# [RO]
getent hosts rhel-lab01
```

Expected:

```text
<IP> rhel-lab01
```

Nếu command không trả về gì, có thể DNS/host resolution chưa đúng.

Đừng ngay lập tức sửa `/etc/hosts` nếu chưa có authorization.

---

# 18. Verify initial SSH

```bash
# [RO]
ssh student@rhel-lab01
```

Lần đầu có thể gặp host-key verification prompt.

Không sử dụng shortcut kiểu:

```text
StrictHostKeyChecking=no
```

chỉ để bỏ qua validation.

Trong lab được quản lý, bạn nên đối chiếu host fingerprint với source tin cậy nếu available.

---

# 19. Sau khi login

Ngay lập tức:

```bash
hostname
whoami
id
pwd
date -Is
```

Đây là post-login identity verification.

Bạn muốn xác nhận:

```text
Correct host
+
Correct account
+
Correct session
```

---

# 20. Verify sudo scope

```bash
# [RO]
sudo -l
```

Đọc output thay vì chỉ nhìn thấy:

```text
User may run the following commands...
```

Hãy xác định:

```text
Có unrestricted sudo?
Có command-specific sudo?
Có target-user restriction?
```

Không chạy arbitrary privileged command chỉ để "test".

---

# 21. Evidence

Có thể lưu:

```bash
{
    date -Is
    hostname
    whoami
    id
    sudo -l
} | tee ~/day1_access_verification.txt
```

Nếu evidence repository nằm trên workstation thay vì server, copy output sau đó bằng approved method.

Không đặt password trong pipe hoặc command arguments.

---

# 22. LAB 3 — Personal Learning Baseline

Syllabus yêu cầu `personal learning baseline`. Infrastructure_Operations_14_Da…

Đây không phải Linux configuration.

Mục đích là xác định skill level trước lab.

---

# 23. Baseline scoring

Bạn có thể tự chấm:

| Skill                  |         0 |            1 |                2 |      3 |
| ---------------------- | --------: | -----------: | ---------------: | -----: |
| CLI navigation         | chưa biết | biết command | làm có hướng dẫn | tự làm |
| File operations        |         0 |            1 |                2 |      3 |
| Search/text processing |         0 |            1 |                2 |      3 |
| Users/groups           |         0 |            1 |                2 |      3 |
| Permissions            |         0 |            1 |                2 |      3 |
| `sudo`                 |         0 |            1 |                2 |      3 |
| SSH key                |         0 |            1 |                2 |      3 |
| Troubleshooting        |         0 |            1 |                2 |      3 |
| Evidence collection    |         0 |            1 |                2 |      3 |

Không dùng baseline để "lấy điểm".

Mục đích là so sánh trước/sau Day 1.

---

# 24. LAB 4 — System Inventory

## Objective

Thu thập baseline mô tả Linux system trước configuration changes.

---

# 25. Timestamp

Bắt đầu inventory bằng:

```bash
# [RO]
date -Is
```

---

# 26. Host identity

```bash
# [RO]
hostnamectl
```

Quan sát:

```text
Static hostname
Operating System
Kernel
Architecture
```

Không phải mọi Linux distribution sẽ hiển thị giống nhau.

---

# 27. Distribution information

```bash
# [RO]
cat /etc/os-release
```

Bạn phải xác định:

```text
NAME
VERSION
ID
VERSION_ID
```

Ví dụ RHEL có metadata tương ứng của Red Hat Enterprise Linux.

Không dùng:

```bash
uname -r
```

để kết luận OS release.

---

# 28. Kernel

```bash
# [RO]
uname -r
```

Architecture:

```bash
# [RO]
uname -m
```

Có thể xem tổng hợp:

```bash
# [RO]
uname -a
```

---

# 29. CPU

```bash
# [RO]
lscpu
```

Bạn chưa cần phân tích NUMA hay virtualization sâu ở Day 1.

Cần nhận biết:

```text
Architecture
CPU count
Model name
```

---

# 30. Memory

```bash
# [RO]
free -h
```

`-h` giúp human-readable.

Day 1 chỉ cần inventory, chưa troubleshooting memory sâu.

---

# 31. Block devices

```bash
# [RO]
lsblk
```

Có thể thêm filesystem details:

```bash
# [RO]
lsblk -f
```

Không chạy `fdisk`, `mkfs`, `wipefs` hoặc disk-changing command trong Day 1.

Storage administration thuộc Day 2.

---

# 32. Network identity

**Kiến thức bổ sung phục vụ inventory; networking được học sâu ở Day 2.**

```bash
# [RO]
ip -brief address
```

Route overview:

```bash
# [RO]
ip route
```

Không thay IP hoặc route trong Day 1.

---

# 33. Current identity

```bash
# [RO]
id
```

---

# 34. Lưu inventory evidence

Ví dụ:

```bash
{
    echo "=== TIMESTAMP ==="
    date -Is

    echo "=== HOST ==="
    hostnamectl

    echo "=== OS ==="
    cat /etc/os-release

    echo "=== KERNEL ==="
    uname -r
    uname -m

    echo "=== CPU ==="
    lscpu

    echo "=== MEMORY ==="
    free -h

    echo "=== STORAGE ==="
    lsblk -f

    echo "=== NETWORK ==="
    ip -brief address

    echo "=== IDENTITY ==="
    id
} | tee ~/day1_system_inventory.txt
```

Đây là read-only collection.

---

# 35. Verification

Đọc lại:

```bash
less ~/day1_system_inventory.txt
```

Hoặc:

```bash
cat ~/day1_system_inventory.txt
```

Kiểm tra:

```text
Có hostname?
Có timestamp?
Có OS?
Có kernel?
Có identity?
Output có bị truncate không?
```

---

# 36. Troubleshooting

Nếu:

```text
command not found
```

đừng ngay lập tức install package.

Xác định:

```bash
command -v <command>
```

và ghi evidence rằng utility không available.

Package management là Day 2 và việc install package có thể cần authorization.

---

# 37. LAB 5 — CLI Navigation

## Objective

Thành thạo:

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

---

# 38. Tạo workspace

```bash
# [CHANGE]
mkdir -p ~/day1_lab/navigation/a/b/c
```

Verify:

```bash
# [RO]
find ~/day1_lab/navigation -type d
```

---

# 39. Bắt đầu tại home

```bash
cd ~
pwd
```

Expected:

```text
/home/<user>
```

---

# 40. Absolute path navigation

```bash
cd /etc
pwd
```

Expected:

```text
/etc
```

Absolute path luôn bắt đầu từ:

```text
/
```

---

# 41. Relative navigation

Quay về lab:

```bash
cd ~/day1_lab/navigation/a/b/c
pwd
```

Đi lên một level:

```bash
cd ..
pwd
```

Expected:

```text
.../navigation/a/b
```

---

# 42. Current directory

```bash
cd .
```

Location không thay đổi.

Check:

```bash
pwd
```

---

# 43. Home shortcut

```bash
cd ~
pwd
```

---

# 44. Previous directory

Trong nhiều shells:

```bash
cd -
```

chuyển về previous working directory.

Đây là tiện ích hữu ích nhưng đừng dùng nếu bạn không chắc previous location trong destructive workflow.

---

# 45. Navigation challenge

Không dùng GUI.

Đi từ:

```text
/home/student
```

đến:

```text
/home/student/day1_lab/navigation/a/b/c
```

bằng absolute path.

Sau đó về:

```text
/home/student/day1_lab/navigation/a
```

bằng relative path.

Verify mỗi lần:

```bash
pwd
```

---

# 46. Evidence

```bash
pwd | tee ~/day1_evidence_navigation_final.txt
```

Nhưng một output cuối chưa chứng minh toàn bộ exercise.

Lab journal nên ghi các paths đã đi qua.

---

# 47. LAB 6 — File và Directory Operations

## Objective

Thực hành:

```text
mkdir
touch
cp
mv
rm
rmdir
```

theo safe workflow.

---

# 48. Tạo structure

```bash
cd ~/day1_lab
```

```bash
# [CHANGE]
mkdir -p files/{input,processing,output,archive}
```

Verify:

```bash
find files -maxdepth 2 -type d | sort
```

Expected:

```text
files
files/archive
files/input
files/output
files/processing
```

---

# 49. Tạo file

```bash
# [CHANGE]
touch files/input/report01.txt
```

Verify:

```bash
ls -l files/input/report01.txt
```

---

# 50. Ghi test data

```bash
# [CHANGE]
printf '%s\n' \
'INFO application started' \
'WARN disk usage 75%' \
'ERROR connection failed' \
'INFO retry started' \
'ERROR connection failed' \
> files/input/application.log
```

Lưu ý:

```text
>
```

sẽ truncate destination nếu file đã tồn tại.

Vì đây là lab file mới nên được phép.

Trong production cần inspect trước.

---

# 51. Verify

```bash
cat files/input/application.log
```

Expected:

```text
INFO application started
WARN disk usage 75%
ERROR connection failed
INFO retry started
ERROR connection failed
```

---

# 52. Copy

```bash
# [CHANGE]
cp files/input/report01.txt files/processing/report01.txt
```

Verify:

```bash
ls -l files/input/report01.txt files/processing/report01.txt
```

---

# 53. Copy với preservation awareness

Basic Day 1:

```bash
cp source destination
```

Không nên mặc định rằng mọi metadata đều được preserve giống hệt.

Trong production backup/migration cần policy/tool phù hợp.

Day 1 chỉ tập basic operation.

---

# 54. Move

```bash
# [CHANGE]
mv files/processing/report01.txt files/output/report01.txt
```

Verify:

```bash
ls -l files/processing
ls -l files/output
```

Expected:

```text
processing không còn report01.txt
output có report01.txt
```

---

# 55. Rename

```bash
# [CHANGE]
mv files/output/report01.txt files/output/report01_completed.txt
```

Verify:

```bash
ls -l files/output
```

---

# 56. Safe delete workflow

Trước khi delete:

```bash
pwd
```

```bash
ls -l files/output/report01_completed.txt
```

Sau khi xác nhận đúng target:

```bash
# [DESTRUCTIVE]
rm files/output/report01_completed.txt
```

Verify:

```bash
ls -l files/output
```

---

# 57. Không dùng `rm -rf` như phản xạ

Day 1 phải hình thành habit:

```text
Know target first.
```

Không:

```text
I want directory gone
→ rm -rf
```

một cách máy móc.

---

# 58. `rmdir`

Tạo empty directory:

```bash
mkdir files/test-empty
```

Remove:

```bash
# [DESTRUCTIVE]
rmdir files/test-empty
```

Nếu directory không empty, `rmdir` sẽ fail.

Đây là useful safety property.

---

# 59. LAB 7 — Searching

Syllabus yêu cầu `searching exercises` nhưng không quy định exact utility nào. Các command dưới đây là common Linux choices để thực hiện yêu cầu đó. Infrastructure_Operations_14_Da…

---

# 60. Search filename bằng `find`

```bash
# [RO]
find ~/day1_lab -name 'application.log'
```

Case-insensitive:

```bash
find ~/day1_lab -iname 'APPLICATION.LOG'
```

Search regular files:

```bash
find ~/day1_lab -type f
```

---

# 61. Search directories

```bash
find ~/day1_lab -type d
```

---

# 62. Search theo name pattern

```bash
find ~/day1_lab -type f -name '*.log'
```

---

# 63. Operational warning với `find`

`find` có thể kết hợp actions mang tính destructive.

Trong Day 1 chỉ dùng search/read-only forms.

Không cần sử dụng:

```text
-delete
-exec rm
```

để học search.

---

# 64. Search text bằng `grep`

```bash
grep 'ERROR' ~/day1_lab/files/input/application.log
```

Expected:

```text
ERROR connection failed
ERROR connection failed
```

---

# 65. Case-insensitive search

```bash
grep -i 'error' ~/day1_lab/files/input/application.log
```

---

# 66. Hiển thị line number

```bash
grep -n 'ERROR' ~/day1_lab/files/input/application.log
```

Expected dạng:

```text
3:ERROR connection failed
5:ERROR connection failed
```

Line numbers phụ thuộc nội dung file.

---

# 67. LAB 8 — Basic Text Processing

Syllabus yêu cầu `text-processing exercises`, nhưng không chỉ định bộ command cụ thể. Để tránh đi quá sâu sang Day 2, phần này chỉ sử dụng basic utilities; `awk`, `sed` và regular expressions sẽ được học sâu hơn ở Day 2. Infrastructure_Operations_14_Da…

---

# 68. `head`

```bash
head ~/day1_lab/files/input/application.log
```

Với file nhỏ sẽ hiển thị toàn bộ nếu ít hơn default number of lines.

---

# 69. `tail`

```bash
tail ~/day1_lab/files/input/application.log
```

---

# 70. Count lines

```bash
wc -l ~/day1_lab/files/input/application.log
```

Với dữ liệu mẫu:

```text
5
```

---

# 71. Pipeline

```bash
grep 'ERROR' ~/day1_lab/files/input/application.log | wc -l
```

Expected:

```text
2
```

Data flow:

```text
application.log
     ↓
grep ERROR
     ↓
2 matching lines
     ↓
wc -l
     ↓
2
```

---

# 72. Sort

```bash
grep 'ERROR' ~/day1_lab/files/input/application.log | sort
```

---

# 73. Count duplicate messages

```bash
grep 'ERROR' ~/day1_lab/files/input/application.log |
sort |
uniq -c
```

Expected conceptually:

```text
2 ERROR connection failed
```

Tại sao cần `sort`?

`uniq` xử lý adjacent repeated lines; `sort` gom identical lines lại gần nhau.

---

# 74. Redirect filtered output

```bash
# [CHANGE]
grep 'ERROR' ~/day1_lab/files/input/application.log \
> ~/day1_lab/files/output/errors.txt
```

Verify:

```bash
cat ~/day1_lab/files/output/errors.txt
```

---

# 75. Append

```bash
# [CHANGE]
grep 'WARN' ~/day1_lab/files/input/application.log \
>> ~/day1_lab/files/output/errors.txt
```

Verify:

```bash
cat ~/day1_lab/files/output/errors.txt
```

Bạn phải nhìn thấy ERROR lines cũ cùng WARN line append sau đó.

---

# 76. Redirect stderr exercise

```bash
ls /path-that-does-not-exist 2> ~/day1_lab/files/output/error_output.txt
```

Command sẽ fail.

Kiểm tra:

```bash
cat ~/day1_lab/files/output/error_output.txt
```

Đây là controlled negative test.

---

# 77. Verify exit code

Ngay sau failed command:

```bash
echo $?
```

Bạn sẽ nhận non-zero.

Sau command thành công:

```bash
true
echo $?
```

Expected:

```text
0
```

Exit-code handling được học sâu hơn ở Day 2, nhưng Day 1 nên hiểu success/failure concept.

---

# 78. LAB 9 — Controlled Users and Groups

Đây là phần cần `sudo`.

Trước khi thay đổi identity database phải đảm bảo các names chưa tồn tại.

---

# 79. Pre-check users

```bash
# [RO]
getent passwd ops1
```

```bash
# [RO]
getent passwd audit1
```

Nếu output xuất hiện:

> Dừng.

Không dùng account đang tồn tại.

Chọn controlled names khác.

---

# 80. Pre-check group

```bash
# [RO]
getent group day1ops
```

Nếu group tồn tại, không assume rằng bạn được phép reuse.

---

# 81. Tạo group

```bash
# [CHANGE]
sudo groupadd day1ops
```

Verify:

```bash
getent group day1ops
```

Expected dạng:

```text
day1ops:x:<GID>:
```

GID sẽ phụ thuộc environment.

---

# 82. Tạo user `ops1`

```bash
# [CHANGE]
sudo useradd -m -s /bin/bash ops1
```

Các thành phần:

```text
useradd     tạo account
-m          tạo home directory nếu cần
-s          chọn login shell
/bin/bash   Bash shell
ops1        username
```

---

# 83. Tạo `audit1`

```bash
# [CHANGE]
sudo useradd -m -s /bin/bash audit1
```

---

# 84. Verify accounts

```bash
getent passwd ops1
getent passwd audit1
```

```bash
id ops1
id audit1
```

---

# 85. Verify home

```bash
ls -ld /home/ops1 /home/audit1
```

Quan sát:

```text
owner
group
permissions
```

---

# 86. Add `ops1` vào group

```bash
# [CHANGE]
sudo usermod -aG day1ops ops1
```

Hai options rất quan trọng:

```text
-a = append
-G = supplementary groups
```

---

# 87. Common dangerous mistake

Không nên vô tình chạy:

```bash
usermod -G day1ops ops1
```

nếu user đã có supplementary groups quan trọng.

Tùy tình huống, `-G` không đi cùng `-a` có thể replace supplementary group list.

Vì vậy:

> Baseline existing memberships trước khi thay group.

---

# 88. Verify group membership

```bash
id ops1
```

Expected có:

```text
groups=...,day1ops
```

Verify từ group database:

```bash
getent group day1ops
```

---

# 89. Negative verification

`audit1` không nên thuộc `day1ops`:

```bash
id audit1
```

Expected:

```text
day1ops không xuất hiện
```

Đây là evidence rất quan trọng.

Bạn đã tạo một:

```text
authorized identity
```

và một:

```text
unauthorized identity
```

để test access control.

---

# 90. Evidence before moving on

```bash
{
    echo "=== TIMESTAMP ==="
    date -Is
    echo "=== OPS1 ==="
    id ops1
    echo "=== AUDIT1 ==="
    id audit1
    echo "=== GROUP ==="
    getent group day1ops
} | sudo tee /var/tmp/day1_users_groups_evidence.txt >/dev/null
```

Không bắt buộc đặt evidence ở `/var/tmp`; bạn có thể lưu trong home của admin.

---

# 91. LAB 10 — Ownership and Permissions

## Objective

Thiết kế resource sao cho:

```text
root owns data
day1ops group has approved access
ops1 can read
audit1 cannot read
```

---

# 92. Tạo lab path

```bash
# [CHANGE]
sudo mkdir -p /srv/day1lab/permtest
```

---

# 93. Tạo test data

```bash
# [CHANGE]
printf 'Day 1 controlled data\n' |
sudo tee /srv/day1lab/permtest/report.txt >/dev/null
```

---

# 94. Baseline

```bash
# [RO]
ls -ld /srv/day1lab/permtest
ls -l /srv/day1lab/permtest/report.txt
```

Lưu output **trước change**.

---

# 95. Cấu hình directory

Để isolation file-permission test rõ ràng, cho mọi user traverse/list lab directory:

```bash
# [CHANGE]
sudo chown root:root /srv/day1lab/permtest
```

```bash
# [CHANGE]
sudo chmod 0755 /srv/day1lab/permtest
```

Đây là lab directory chứa test data không sensitive.

---

# 96. Cấu hình file ownership

```bash
# [CHANGE]
sudo chown root:day1ops /srv/day1lab/permtest/report.txt
```

Verify:

```bash
ls -l /srv/day1lab/permtest/report.txt
```

---

# 97. Cấu hình file permission

Requirement:

```text
owner: read + write
group: read
other: none
```

Mode:

```text
rw-r-----
```

Numeric:

```text
640
```

Apply:

```bash
# [CHANGE]
sudo chmod 0640 /srv/day1lab/permtest/report.txt
```

---

# 98. Verify state

```bash
ls -l /srv/day1lab/permtest/report.txt
```

Expected conceptually:

```text
-rw-r----- ... root day1ops ... report.txt
```

Chi tiết timestamp/size khác.

---

# 99. Verify bằng `stat`

```bash
stat /srv/day1lab/permtest/report.txt
```

Tìm:

```text
Access:
Uid:
Gid:
```

---

# 100. Positive access test — `ops1`

Trước tiên verify target identity khi impersonating:

```bash
sudo -u ops1 id
```

Sau đó:

```bash
sudo -u ops1 cat /srv/day1lab/permtest/report.txt
```

Expected:

```text
Day 1 controlled data
```

Đây là **positive test**.

---

# 101. Negative access test — `audit1`

```bash
sudo -u audit1 id
```

Verify `day1ops` không xuất hiện.

Test:

```bash
sudo -u audit1 cat /srv/day1lab/permtest/report.txt
```

Expected:

```text
Permission denied
```

Đây là **negative test**.

---

# 102. Tại sao test bằng `sudo -u`?

Trong controlled lab, administrative account có thể execute command dưới identity khác để test permissions.

Nhưng cần hiểu limitation:

```text
sudo -u user command
```

không hoàn toàn mô phỏng mọi aspect của một interactive login/session.

Nó rất hữu ích cho filesystem permission test, nhưng nếu test PAM/session/login behavior thì cần method phù hợp khác.

---

# 103. Test write denial cho `ops1`

`ops1` có group mode `r--`, không có `w`.

Test:

```bash
sudo -u ops1 sh -c \
'echo unauthorized-change >> /srv/day1lab/permtest/report.txt'
```

Expected:

```text
Permission denied
```

Đây là thiết kế đúng.

`ops1` được:

```text
read
```

nhưng không được:

```text
write
```

---

# 104. Verify data không bị thay đổi

```bash
cat /srv/day1lab/permtest/report.txt
```

Expected vẫn chỉ:

```text
Day 1 controlled data
```

---

# 105. Permissions matrix

Kết quả mong muốn:

| Identity             | Read | Write |
| -------------------- | ---: | ----: |
| `root`               |  Yes |   Yes |
| `ops1` via `day1ops` |  Yes |    No |
| `audit1`             |   No |    No |

Đây là một example rất rõ của least privilege.

---

# 106. LAB 11 — Directory Permission Semantics

File permission một mình chưa đủ.

Ta tạo directory khác để chứng minh `x` trên directory quan trọng.

---

# 107. Tạo secure directory

```bash
sudo mkdir -p /srv/day1lab/private
```

```bash
printf 'private lab data\n' |
sudo tee /srv/day1lab/private/private.txt >/dev/null
```

---

# 108. Ownership

```bash
sudo chown -R root:day1ops /srv/day1lab/private
```

Vì đây là tiny controlled lab tree nên recursive change có scope rõ.

Trong production không dùng `-R` mà chưa inventory target.

---

# 109. Directory permissions

```bash
sudo chmod 0750 /srv/day1lab/private
```

File:

```bash
sudo chmod 0640 /srv/day1lab/private/private.txt
```

---

# 110. Positive test

```bash
sudo -u ops1 cat /srv/day1lab/private/private.txt
```

Expected success.

---

# 111. Negative test

```bash
sudo -u audit1 cat /srv/day1lab/private/private.txt
```

Expected:

```text
Permission denied
```

Trong scenario này failure có thể xảy ra ngay ở directory traversal.

---

# 112. Troubleshooting lesson

Nếu chỉ nhìn:

```bash
ls -l /srv/day1lab/private/private.txt
```

bạn có thể bỏ sót permissions của:

```text
/srv
/srv/day1lab
/srv/day1lab/private
```

Khi troubleshoot nested path:

```text
/file/path/object
```

phải nghĩ tới **parent directories**.

---

# 113. LAB 12 — `chmod`, `chown`, `chgrp` Practice

## `chmod`

Symbolic:

```bash
sudo chmod g+r /srv/day1lab/permtest/report.txt
```

Remove:

```bash
sudo chmod o-r /srv/day1lab/permtest/report.txt
```

Reset expected state:

```bash
sudo chmod 0640 /srv/day1lab/permtest/report.txt
```

---

# 114. `chgrp`

Thay group owner:

```bash
sudo chgrp day1ops /srv/day1lab/permtest/report.txt
```

Verify:

```bash
ls -l /srv/day1lab/permtest/report.txt
```

---

# 115. `chown`

Owner + group:

```bash
sudo chown root:day1ops /srv/day1lab/permtest/report.txt
```

Verify:

```bash
stat /srv/day1lab/permtest/report.txt
```

---

# 116. Pre/post evidence pattern

Trước change:

```bash
stat /srv/day1lab/permtest/report.txt \
| tee ~/permission_before.txt
```

Sau change:

```bash
stat /srv/day1lab/permtest/report.txt \
| tee ~/permission_after.txt
```

Compare:

```bash
diff -u ~/permission_before.txt ~/permission_after.txt
```

Lưu ý `stat` có timestamps thay đổi nên diff có thể chứa nhiều differences.

Mục tiêu là tìm:

```text
Mode
UID
GID
```

---

# 117. LAB 13 — `umask`

## Objective

Hiểu thực nghiệm rằng `umask` ảnh hưởng permissions của newly-created objects.

---

# 118. Xem current umask

```bash
# [RO]
umask
```

Lưu giá trị:

```bash
old_umask=$(umask)
```

Đây chỉ là shell variable trong current shell.

---

# 119. Tạo baseline object

```bash
mkdir -p ~/day1_lab/umask
cd ~/day1_lab/umask
```

```bash
touch before_file
mkdir before_dir
```

Check:

```bash
ls -ld before_file before_dir
```

---

# 120. Tạm thời thay umask

```bash
# [CHANGE - current shell only]
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

hoặc representation tương đương.

---

# 121. Tạo objects mới

```bash
touch after_file
mkdir after_dir
```

Check:

```bash
ls -ld after_file after_dir
```

Typical expectation với normal tools:

```text
after_file → 640
after_dir  → 750
```

Tức:

```text
file base 666
mask      027
result    640
```

và:

```text
dir base  777
mask      027
result    750
```

---

# 122. Restore umask

Rất quan trọng.

```bash
# [CHANGE]
umask "$old_umask"
```

Verify:

```bash
umask
```

Phải trở lại value ban đầu.

---

# 123. Troubleshooting

Nếu result không đúng textbook expectation, nhớ rằng:

> Application có thể request initial mode khác.

`umask` loại bỏ bits; nó không đảm bảo mọi program tạo files bằng cùng requested mode.

---

# 124. LAB 14 — Least-Privilege `sudo`

Đây là phần có risk cao hơn các lab trước vì liên quan privileged authorization.

Chỉ thực hiện trên lab VM.

Không copy rule này sang production.

---

# 125. Objective

Thiết kế:

```text
ops1
 │
 │ sudo
 ▼
ONLY one harmless lab command
```

Cho phép:

```text
create /var/tmp/day1-sudo-proof
```

Không cho phép:

```text
arbitrary root commands
```

---

# 126. Chọn command an toàn

Chúng ta dùng:

```text
/usr/bin/touch /var/tmp/day1-sudo-proof
```

Trước hết xác nhận actual path:

```bash
# [RO]
command -v touch
```

Trên nhiều RHEL systems expected:

```text
/usr/bin/touch
```

Nếu path khác, dùng actual path.

Đừng hard-code path không đúng.

---

# 127. Pre-check `ops1`

```bash
id ops1
```

Đảm bảo `ops1` không phải privileged account ngoài mục tiêu lab.

---

# 128. Tạo sudoers candidate file trước

Không viết invalid syntax trực tiếp vào active policy.

Tạo staging file:

```bash
# [CHANGE]
printf '%s\n' \
'ops1 ALL=(root) NOPASSWD: /usr/bin/touch /var/tmp/day1-sudo-proof' \
| sudo tee /tmp/day1-ops1.sudoers >/dev/null
```

`NOPASSWD` ở đây chỉ dùng để cho phép automated negative/positive lab test với controlled exact command.

Trong production, password/authentication requirement phải theo organization policy.

---

# 129. Set safe mode

```bash
sudo chmod 0440 /tmp/day1-ops1.sudoers
```

---

# 130. Validate syntax trước activation

```bash
# [RO]
sudo visudo -cf /tmp/day1-ops1.sudoers
```

Expected:

```text
parsed OK
```

Hoặc equivalent success result.

Nếu validation fail:

> Không install.

Sửa candidate trước.

---

# 131. Install sudoers drop-in

Chỉ sau successful validation:

```bash
# [CHANGE]
sudo install \
-o root \
-g root \
-m 0440 \
/tmp/day1-ops1.sudoers \
/etc/sudoers.d/day1-ops1
```

---

# 132. Validate toàn bộ sudo configuration

```bash
# [RO]
sudo visudo -cf /etc/sudoers
```

Expected no syntax errors.

Nếu lỗi:

> Rollback ngay file vừa tạo.

```bash
sudo rm /etc/sudoers.d/day1-ops1
```

sau đó revalidate.

---

# 133. Positive sudo test

Từ admin account:

```bash
sudo -u ops1 -H bash -lc \
'sudo -n /usr/bin/touch /var/tmp/day1-sudo-proof'
```

Expected không báo permission error.

---

# 134. Verify result

```bash
ls -l /var/tmp/day1-sudo-proof
```

Expected owner:

```text
root
```

Điều này chứng minh command đã được elevated.

---

# 135. Negative sudo test

Cho `ops1` thử command không được grant:

```bash
sudo -u ops1 -H bash -lc \
'sudo -n /usr/bin/id'
```

Expected denial kiểu:

```text
user is not allowed...
```

Exact message tùy `sudo` version/configuration.

Điều quan trọng:

```text
approved exact command → success

unapproved command → denied
```

---

# 136. Đây mới là least privilege verification

Chỉ chứng minh:

```text
touch works
```

là chưa đủ.

Bạn phải chứng minh:

```text
other sudo command does NOT work
```

---

# 137. Sudo evidence

Lưu:

```text
sudoers candidate validation
sudo policy listing
successful allowed command
failed unauthorized command
proof-file owner
timestamp
```

Bạn có thể kiểm tra:

```bash
sudo -u ops1 -H bash -lc 'sudo -n -l'
```

Output cần được review cẩn thận.

---

# 138. Sudo rollback

Sau khi hoàn tất lab:

```bash
# [CHANGE]
sudo rm /etc/sudoers.d/day1-ops1
```

Xóa proof file:

```bash
sudo rm /var/tmp/day1-sudo-proof
```

Validate:

```bash
sudo visudo -cf /etc/sudoers
```

Verify `ops1` không còn rule:

```bash
sudo -u ops1 -H bash -lc 'sudo -n -l'
```

Expected controlled rule không còn.

---

# 139. LAB 15 — SSH Key Management

## Objective

Tạo SSH key pair an toàn trên administrative workstation và dùng public key để authenticate vào lab server.

Không gửi private key cho bất kỳ ai.

---

# 140. Pre-check `.ssh`

Trên workstation:

```bash
ls -ld ~/.ssh
```

Nếu chưa tồn tại:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

---

# 141. Generate lab key

Modern OpenSSH environments thường hỗ trợ `ed25519`.

Ví dụ:

```bash
ssh-keygen \
-t ed25519 \
-f ~/.ssh/day1_lab_ed25519 \
-C "day1-lab"
```

Bạn sẽ được hỏi passphrase.

Sử dụng passphrase phù hợp nếu lab/policy cho phép.

Không chia sẻ passphrase.

Nếu environment/security policy, FIPS mode hoặc OpenSSH version không hỗ trợ/không cho phép `ed25519`, phải dùng algorithm được organization approve thay vì tự ý bypass policy.

---

# 142. Files được tạo

```text
~/.ssh/day1_lab_ed25519
~/.ssh/day1_lab_ed25519.pub
```

Phân biệt:

```text
day1_lab_ed25519
= PRIVATE KEY

day1_lab_ed25519.pub
= PUBLIC KEY
```

---

# 143. Verify private key permissions

```bash
ls -l ~/.ssh/day1_lab_ed25519
```

Private key phải được restrictive.

OpenSSH client có thể từ chối sử dụng private key nếu permissions quá open.

---

# 144. Public key fingerprint

```bash
ssh-keygen -lf ~/.ssh/day1_lab_ed25519.pub
```

Output dạng:

```text
256 SHA256:... day1-lab ...
```

Fingerprint là evidence hữu ích.

Không cần chụp private-key contents.

---

# 145. Backup authorized_keys trước modification

Trước khi thêm key vào existing lab account, login bằng existing approved method.

Trên target:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Nếu `authorized_keys` đã tồn tại:

```bash
cp -a ~/.ssh/authorized_keys \
~/.ssh/authorized_keys.pre-day1
```

Lưu ý:

Nếu file backup đã tồn tại từ lab cũ, không overwrite blindly.

Bạn có thể thêm timestamp:

```bash
cp -a ~/.ssh/authorized_keys \
~/.ssh/authorized_keys.pre-day1.$(date +%Y%m%d%H%M%S)
```

---

# 146. Install public key — preferred helper

Nếu existing SSH authentication method cho phép, từ workstation:

```bash
ssh-copy-id \
-i ~/.ssh/day1_lab_ed25519.pub \
student@rhel-lab01
```

Tool sẽ copy **public key**, không phải private key.

---

# 147. Verify server-side authorization file

Trên target:

```bash
ls -ld ~/.ssh
```

```bash
ls -l ~/.ssh/authorized_keys
```

Bạn cần quan tâm:

```text
owner
group
permissions
```

Không paste contents chứa nhiều users' public keys vào public evidence nếu organization coi đó là sensitive operational information.

Fingerprint là evidence tốt hơn.

---

# 148. Positive SSH test

Từ workstation:

```bash
ssh \
-i ~/.ssh/day1_lab_ed25519 \
-o IdentitiesOnly=yes \
student@rhel-lab01 \
'hostname; whoami; id'
```

Expected:

```text
rhel-lab01
student
uid=...
```

Điều cần chứng minh là:

```text
correct key
→ correct host
→ correct account
```

---

# 149. Vì sao dùng `IdentitiesOnly=yes`?

SSH client/agent có thể có nhiều identities.

Nếu bạn không giới hạn identity, connection có thể thành công nhờ một key khác và bạn tưởng rằng lab key hoạt động.

`IdentitiesOnly=yes` làm test rõ ràng hơn.

---

# 150. SSH verbose troubleshooting

Nếu key không work:

```bash
ssh \
-v \
-i ~/.ssh/day1_lab_ed25519 \
-o IdentitiesOnly=yes \
student@rhel-lab01
```

`-v` cung cấp debug information.

Nếu cần sâu hơn có thể dùng additional verbosity nhưng tránh dump unnecessarily large logs hoặc information không cần thiết.

---

# 151. Không thay server config ngay

Nếu SSH key fail, không bắt đầu bằng:

```text
edit sshd_config
restart sshd
disable security
```

Trước tiên kiểm tra:

```text
target hostname
username
key actually offered
public key actually installed
ownership
permissions
account state
```

---

# 152. LAB 16 — Denied SSH Access Test

Syllabus yêu cầu verify **authorized and denied access**. Infrastructure_Operations_14_Da…

Tạo một second key nhưng **không install public key**.

---

# 153. Generate unauthorized test key

```bash
ssh-keygen \
-t ed25519 \
-f ~/.ssh/day1_denied_ed25519 \
-C "day1-denied-test"
```

Đây vẫn là secret private key của lab; không đưa vào evidence.

---

# 154. Confirm fingerprint

```bash
ssh-keygen -lf ~/.ssh/day1_denied_ed25519.pub
```

---

# 155. Negative authentication test

```bash
ssh \
-i ~/.ssh/day1_denied_ed25519 \
-o IdentitiesOnly=yes \
-o BatchMode=yes \
student@rhel-lab01 \
'true'
```

`BatchMode=yes` giúp không fallback vào interactive password prompt trong test automation context.

Expected:

```text
Permission denied ...
```

Exact authentication methods được hiển thị tùy server.

---

# 156. Kiểm tra exit status

Ngay sau failed SSH:

```bash
echo $?
```

Expected:

```text
non-zero
```

Đây là correct result của negative test.

Không phải mọi non-zero trong security test đều là "lab failure".

Ở đây nó chứng minh unauthorized key bị reject.

---

# 157. Positive vs Negative evidence

Bạn phải có hai pieces:

```text
Installed authorized key
→ login success

Uninstalled unauthorized key
→ login failure
```

Nếu chỉ có success test, chưa chứng minh enforcement đầy đủ.

---

# 158. LAB 17 — Secure SSH Key Handling

## Kiểm tra local files

```bash
ls -l ~/.ssh/day1_lab_ed25519*
```

Bạn phải biết file nào là private/public.

---

# 159. Không `cat` private key vào terminal recorder

Không làm:

```text
cat ~/.ssh/day1_lab_ed25519
```

trong evidence session.

Điều đó không giúp verification và tạo risk leakage.

---

# 160. Public key inspection

Nếu cần:

```bash
cat ~/.ssh/day1_lab_ed25519.pub
```

Nhưng trong audit evidence, fingerprint thường gọn hơn:

```bash
ssh-keygen -lf ~/.ssh/day1_lab_ed25519.pub
```

---

# 161. LAB 18 — Command History and Evidence

## Objective

Document executed commands mà không biến shell history thành nơi chứa secrets.

---

# 162. Xem history

```bash
history
```

Không nên coi đây là source duy nhất.

History có thể thiếu:

```text
timestamps
outputs
commands from other sessions
full context
```

---

# 163. Timestamps cho current history display

Trong Bash, có thể tạm thời:

```bash
export HISTTIMEFORMAT='%F %T %z '
```

Sau đó:

```bash
history
```

Lưu ý history entries cũ chỉ có meaningful timestamps nếu shell history data đã lưu time metadata phù hợp.

Không mặc định coi output này là authoritative audit log.

---

# 164. Evidence strategy tốt hơn

Đối với important checks, dùng:

```bash
command | tee evidence.txt
```

Ví dụ:

```bash
id ops1 |
tee ~/day1_evidence/04_users_groups/ops1_id.txt
```

---

# 165. Capture stderr cùng stdout khi cần

```bash
command > evidence.txt 2>&1
```

hoặc:

```bash
command 2>&1 | tee evidence.txt
```

Nhưng nhớ:

> Evidence chứa raw output. Kiểm tra secret trước khi share.

---

# 166. Optional terminal transcript

Utility `script` có thể capture terminal session nếu installed:

```bash
script ~/day1_session.log
```

Khi xong:

```bash
exit
```

Nhưng **không dùng session recording khi bạn có nguy cơ nhập/display secrets**.

Nếu dùng `ssh-keygen` với passphrase, đảm bảo recording approach không capture sensitive input theo cách không mong muốn.

---

# 167. LAB 19 — Integrated Access Validation

Bây giờ kết nối các phần lại.

Desired state:

```text
ops1
 ├─ exists
 ├─ member of day1ops
 ├─ can read approved file
 ├─ cannot write approved file
 ├─ cannot read unauthorized file/resource
 └─ only has explicitly permitted sudo operation

audit1
 ├─ exists
 ├─ not member of day1ops
 └─ cannot read protected file

student
 └─ can SSH using approved lab key
```

---

# 168. Identity validation

```bash
id ops1
id audit1
```

Check exactly:

```text
ops1 ∈ day1ops
audit1 ∉ day1ops
```

---

# 169. Ownership validation

```bash
stat /srv/day1lab/permtest/report.txt
```

Expected conceptual:

```text
owner = root
group = day1ops
mode = 640
```

---

# 170. Authorized read

```bash
sudo -u ops1 cat /srv/day1lab/permtest/report.txt
```

Expected success.

---

# 171. Unauthorized read

```bash
sudo -u audit1 cat /srv/day1lab/permtest/report.txt
```

Expected denial.

---

# 172. Unauthorized write

```bash
sudo -u ops1 sh -c \
'echo TEST >> /srv/day1lab/permtest/report.txt'
```

Expected denial.

---

# 173. Authorized sudo

Nếu sudo rule chưa cleanup:

```bash
sudo -u ops1 -H bash -lc \
'sudo -n /usr/bin/touch /var/tmp/day1-sudo-proof'
```

Expected success.

---

# 174. Unauthorized sudo

```bash
sudo -u ops1 -H bash -lc \
'sudo -n /usr/bin/id'
```

Expected denial.

---

# 175. Authorized SSH

Từ workstation:

```bash
ssh \
-i ~/.ssh/day1_lab_ed25519 \
-o IdentitiesOnly=yes \
student@rhel-lab01 \
'hostname; whoami'
```

Expected success.

---

# 176. Unauthorized SSH key

```bash
ssh \
-i ~/.ssh/day1_denied_ed25519 \
-o IdentitiesOnly=yes \
-o BatchMode=yes \
student@rhel-lab01 \
'true'
```

Expected fail.

---

# 177. Acceptance Matrix

| Control          | Positive Test             | Negative Test            |
| ---------------- | ------------------------- | ------------------------ |
| Group membership | `ops1` has `day1ops`      | `audit1` does not        |
| File read        | `ops1` reads              | `audit1` denied          |
| File write       | root can modify           | `ops1` denied            |
| `sudo`           | approved command succeeds | unrelated command denied |
| SSH              | approved key works        | unknown key denied       |

Nếu bạn có đủ five control outcomes này cùng evidence, bạn đã thực hiện đúng tinh thần Assignment Day 1.

---

# 178. FAILURE INJECTION 1 — Wrong Group Membership

## Inject

Tạm remove `ops1` khỏi supplementary group `day1ops`.

Cách thực hiện phụ thuộc distro/tooling; trên Linux có thể dùng appropriate group-management command, nhưng không nên làm nếu bạn không chắc existing memberships.

Trong lab này safest learning pattern là thay ownership của test file sang một temporary group không chứa `ops1`, thay vì phá user memberships.

Ví dụ tạo:

```bash
sudo groupadd day1wrong
```

Thay group file:

```bash
sudo chgrp day1wrong /srv/day1lab/permtest/report.txt
```

---

# 179. Symptom

```bash
sudo -u ops1 cat /srv/day1lab/permtest/report.txt
```

Expected:

```text
Permission denied
```

---

# 180. Troubleshooting

Không sửa ngay.

Collect:

```bash
id ops1
```

```bash
ls -l /srv/day1lab/permtest/report.txt
```

Compare:

```text
user groups = day1ops
file group   = day1wrong
```

Root cause:

```text
group ownership mismatch
```

---

# 181. Rollback

```bash
sudo chgrp day1ops /srv/day1lab/permtest/report.txt
```

Verify:

```bash
sudo -u ops1 cat /srv/day1lab/permtest/report.txt
```

Success phải trở lại.

---

# 182. FAILURE INJECTION 2 — Missing Directory Traverse Permission

Baseline:

```bash
sudo chmod 0750 /srv/day1lab/private
```

Để tạo controlled failure cho group:

```bash
sudo chmod 0740 /srv/day1lab/private
```

Group có:

```text
r--
```

nhưng không có:

```text
x
```

---

# 183. Symptom

```bash
sudo -u ops1 cat /srv/day1lab/private/private.txt
```

Expected:

```text
Permission denied
```

Mặc dù file itself có thể có:

```text
640 root:day1ops
```

---

# 184. Investigation

```bash
ls -ld /srv/day1lab/private
```

```bash
ls -l /srv/day1lab/private/private.txt
```

Bạn phải phát hiện:

```text
file permission looks fine
BUT
directory group lacks x
```

Đây là classic Linux permission troubleshooting scenario.

---

# 185. Rollback

```bash
sudo chmod 0750 /srv/day1lab/private
```

Verify positive access lại.

---

# 186. FAILURE INJECTION 3 — File Mode Too Restrictive

Inject:

```bash
sudo chmod 0600 /srv/day1lab/permtest/report.txt
```

File vẫn:

```text
root:day1ops
```

nhưng group mất read.

---

# 187. Symptom

```bash
sudo -u ops1 cat /srv/day1lab/permtest/report.txt
```

Fail.

---

# 188. Diagnosis

```bash
id ops1
```

Group membership đúng.

```bash
ls -l /srv/day1lab/permtest/report.txt
```

Shows:

```text
-rw-------
```

Root cause:

```text
group read bit missing
```

---

# 189. Rollback

```bash
sudo chmod 0640 /srv/day1lab/permtest/report.txt
```

Verify again.

---

# 190. FAILURE INJECTION 4 — Invalid `sudoers` Candidate

Đây là cách an toàn để học lỗi sudoers mà **không làm hỏng active sudo configuration**.

Tạo invalid temporary file:

```bash
printf '%s\n' \
'THIS IS NOT VALID SUDOERS SYNTAX' \
| sudo tee /tmp/day1-invalid.sudoers >/dev/null
```

Validate:

```bash
sudo visudo -cf /tmp/day1-invalid.sudoers
```

Expected failure.

---

# 191. Correct operational behavior

Không copy file vào:

```text
/etc/sudoers.d/
```

Evidence:

```text
syntax validation failed
candidate rejected
active configuration unchanged
```

Đây là perfect example của:

```text
pre-change verification preventing incident
```

Cleanup:

```bash
sudo rm /tmp/day1-invalid.sudoers
```

---

# 192. FAILURE INJECTION 5 — Unauthorized SSH Key

Bạn đã thực hiện ở SSH negative test:

```text
Generate second key
Do not authorize it
Attempt login
```

Expected:

```text
authentication denied
```

Troubleshooting exercise:

So sánh fingerprint:

```bash
ssh-keygen -lf ~/.ssh/day1_lab_ed25519.pub
```

với:

```bash
ssh-keygen -lf ~/.ssh/day1_denied_ed25519.pub
```

Chúng khác nhau.

Chỉ authorized public key phải được server chấp nhận.

---

# 193. Troubleshooting Workflow — File Permission

Khi gặp:

```text
Permission denied
```

đi theo thứ tự ít rủi ro:

```text
1. whoami / id
       ↓
2. Confirm exact path
       ↓
3. ls -ld parent directories
       ↓
4. ls -l target
       ↓
5. stat target
       ↓
6. Compare owner/group
       ↓
7. Compare required operation: read/write/execute
       ↓
8. Check group membership
       ↓
9. Consider ACL / SELinux if traditional mode correct
       ↓
10. Escalate outside Day 1 scope
```

Không bắt đầu bằng:

```bash
chmod 777
```

---

# 194. Troubleshooting Workflow — SSH

Khi SSH fail:

```text
Exact error first.
```

Sau đó classify.

### `Permission denied`

Focus:

```text
username
key
account
authentication policy
file permissions
```

### `Connection refused`

Khả năng liên quan:

```text
service/listening state
port
```

### Timeout

Khả năng liên quan:

```text
network path
firewall
routing
target state
```

Day 2 sẽ học networking/firewall/service investigation sâu hơn.

---

# 195. Troubleshooting SSH Key

Client side:

```bash
ssh -v \
-i ~/.ssh/day1_lab_ed25519 \
-o IdentitiesOnly=yes \
student@rhel-lab01
```

Check:

```text
Is correct key being offered?
Is target user correct?
Is target hostname correct?
```

Server-side authorized account:

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

Check:

```text
owner
permissions
authorized public key
```

Không tự ý:

```text
disable authentication protections
enable root login
disable host-key checking
```

để "fix".

---

# 196. Troubleshooting `sudo`

Nếu permitted command fail:

```bash
sudo -u ops1 -H bash -lc 'sudo -n -l'
```

Check:

```text
Is rule installed?
Exact command path?
Exact command arguments?
Target user?
Syntax valid?
```

Sudoers command matching có thể rất precise.

Rule:

```text
/usr/bin/touch /var/tmp/day1-sudo-proof
```

không có nghĩa user được phép:

```text
/usr/bin/touch /some/other/file
```

nếu rule thực sự restrict arguments theo exact configured command form.

Đó là mục tiêu của least privilege.

---

# 197. LAB 20 — Evidence Collection Package

Syllabus yêu cầu cuối Day 1 submit:

```text
system inventory
access verification
configuration evidence
```

cùng documentation của executed commands. Infrastructure_Operations_14_Da…

Một evidence package tốt có thể là:

```text
Day1_Evidence/
│
├── 00_access/
│   ├── access_checklist.txt
│   └── initial_identity.txt
│
├── 01_inventory/
│   ├── hostname.txt
│   ├── os_release.txt
│   ├── kernel.txt
│   └── system_inventory.txt
│
├── 02_cli/
│   └── navigation_notes.txt
│
├── 03_files/
│   ├── file_operations.txt
│   ├── search_results.txt
│   └── text_processing.txt
│
├── 04_users_groups/
│   ├── ops1_id.txt
│   ├── audit1_id.txt
│   └── day1ops_group.txt
│
├── 05_permissions/
│   ├── permission_baseline.txt
│   ├── permission_final.txt
│   ├── positive_test.txt
│   └── negative_test.txt
│
├── 06_sudo/
│   ├── sudoers_validation.txt
│   ├── allowed_test.txt
│   └── denied_test.txt
│
├── 07_ssh/
│   ├── key_fingerprint.txt
│   ├── authorized_test.txt
│   └── denied_test.txt
│
├── 08_failures/
│   ├── failure01_group.txt
│   ├── failure02_directory.txt
│   └── failure03_permission.txt
│
└── 09_cleanup/
    └── cleanup_verification.txt
```

Không cần filename chính xác giống vậy.

Structure và traceability mới quan trọng.

---

# 198. Mỗi evidence nên trả lời gì?

Một useful evidence record phải trả lời:

```text
WHEN?
WHO?
WHERE?
WHAT?
EXPECTED?
ACTUAL?
RESULT?
```

Ví dụ:

```text
Timestamp:
2026-10-19T10:15:00+07:00

Host:
rhel-lab01

Operator:
student

Test:
Verify ops1 can read group-authorized report

Command:
sudo -u ops1 cat /srv/day1lab/permtest/report.txt

Expected:
Read succeeds

Observed:
Day 1 controlled data

Result:
PASS
```

---

# 199. Screenshots vs command output

Screenshot hữu ích cho context nhưng text output thường:

```text
searchable
copyable
diffable
smaller
```

Nếu organization yêu cầu screenshot thì thu cả screenshot.

Nhưng screenshot không nên chứa:

```text
private key
password
secret
unrelated confidential data
```

---

# 200. Evidence integrity

Sau khi thu evidence, không edit raw file để "làm đẹp".

Nếu cần explanation:

```text
raw evidence
+
separate analyst note
```

tốt hơn:

```text
modify raw evidence
```

---

# 201. LAB 21 — Cleanup và Rollback

Cleanup là một phần của lab, không phải việc tùy chọn.

Thứ tự hợp lý là remove dependencies từ trên xuống.

---

# 202. Remove sudo rule trước

Nếu còn:

```bash
sudo rm /etc/sudoers.d/day1-ops1
```

Validate:

```bash
sudo visudo -cf /etc/sudoers
```

---

# 203. Remove sudo proof

```bash
sudo rm -f /var/tmp/day1-sudo-proof
```

`-f` ở đây được dùng trên exact known lab file.

Không extrapolate thành habit cho arbitrary paths.

---

# 204. SSH key cleanup

Nếu key chỉ dành cho lab và phải revoke:

1. Identify exact Day 1 public key bằng fingerprint/comment.
2. Backup current `authorized_keys`.
3. Remove **chính xác entry của lab key**.
4. Không overwrite các authorized keys khác.
5. Verify denied access sau removal.

Đừng đơn giản restore một old backup nếu trong thời gian đó có legitimate keys khác được thêm.

---

# 205. Verify SSH revocation

Sau khi remove Day 1 authorized key:

```bash
ssh \
-i ~/.ssh/day1_lab_ed25519 \
-o IdentitiesOnly=yes \
-o BatchMode=yes \
student@rhel-lab01 \
'true'
```

Nếu không còn alternative authorization cho key đó, expected denied.

---

# 206. Local private key cleanup

Nếu training policy yêu cầu remove test keys:

```bash
rm ~/.ssh/day1_denied_ed25519
rm ~/.ssh/day1_denied_ed25519.pub
```

Với primary lab key chỉ remove nếu không còn cần.

Không remove existing real administrative key.

---

# 207. Remove lab directory

Pre-check:

```bash
sudo find /srv/day1lab -maxdepth 3 -ls
```

Đảm bảo chỉ có controlled test data.

Sau đó:

```bash
# [DESTRUCTIVE]
sudo rm -r /srv/day1lab
```

Không thêm `-f` nếu không cần.

Verify:

```bash
ls -ld /srv/day1lab
```

Expected:

```text
No such file or directory
```

---

# 208. Remove users

Trước hết:

```bash
id ops1
id audit1
```

Xác nhận đây chính là lab-created accounts.

Sau đó:

```bash
# [DESTRUCTIVE]
sudo userdel -r ops1
```

```bash
# [DESTRUCTIVE]
sudo userdel -r audit1
```

`-r` có thể remove home/mail data liên quan user.

Chỉ dùng vì đây là controlled lab user.

Không chạy với production account.

---

# 209. Verify users removed

```bash
getent passwd ops1
```

```bash
getent passwd audit1
```

Expected no records.

---

# 210. Remove groups

Sau khi users không còn dependency:

```bash
# [DESTRUCTIVE]
sudo groupdel day1ops
```

Nếu tạo failure-injection group:

```bash
sudo groupdel day1wrong
```

---

# 211. Verify

```bash
getent group day1ops
getent group day1wrong
```

Expected no output.

---

# 212. Không remove evidence

Cleanup operational objects không có nghĩa xóa evidence.

Evidence repository nên giữ:

```text
before state
changes
positive tests
negative tests
cleanup
```

theo training/audit requirement.

---

# 213. Cleanup verification evidence

Thu:

```bash
{
    echo "=== CLEANUP TIMESTAMP ==="
    date -Is

    echo "=== USERS ==="
    getent passwd ops1 || true
    getent passwd audit1 || true

    echo "=== GROUPS ==="
    getent group day1ops || true
    getent group day1wrong || true

    echo "=== LAB DIRECTORY ==="
    ls -ld /srv/day1lab 2>&1 || true

    echo "=== SUDO RULE ==="
    ls -l /etc/sudoers.d/day1-ops1 2>&1 || true
} | tee ~/day1_cleanup_verification.txt
```

Expected là controlled objects không còn.

---

# 214. Practical Troubleshooting Record

Một Infrastructure Operations engineer phải có khả năng document incident/failure.

Ví dụ:

```text
Issue:
ops1 cannot read /srv/day1lab/permtest/report.txt

Timestamp:
...

Expected:
ops1 has read access through day1ops

Observed:
Permission denied

Evidence:
id ops1
ls -ld parent
ls -l file
stat file

Finding:
File group ownership changed from day1ops to day1wrong

Root Cause:
Incorrect group ownership

Corrective Action:
chgrp day1ops report.txt

Verification:
ops1 successfully read file

Rollback:
Not required; corrective action restored approved baseline

Escalation:
Not required; within lab/L2 scope
```

Đây là loại thinking bạn cần tập từ Day 1.

---

# 215. Khi nào không nên tự sửa?

Giả sử bạn thấy:

```text
/etc/ssh/sshd_config
```

khác với expectation.

Không đồng nghĩa bạn được phép sửa.

Nếu:

```text
không có baseline
không biết policy
không có change approval
service impact không rõ
```

thì correct action là:

```text
capture evidence
document discrepancy
escalate
```

không phải:

```text
vi /etc/ssh/sshd_config
```

rồi restart service.

---

# 216. Escalation Boundary Trong Lab

Một practical operator nên escalate khi:

```text
access không được authorize
unknown root-owned object xuất hiện
unknown privileged account
unknown SSH key
sudo rule khác baseline
server có production marker
security control phải bị weaken mới "fix được"
data loss risk
account name collision với existing environment
```

Escalation phải kèm evidence.

---

# 217. Practical Acceptance Test — Task A

Không nhìn tài liệu, bạn phải làm được:

```text
1. Xác định current user
2. Xác định hostname
3. Xác định OS
4. Xác định kernel
5. Tạo directory tree
6. Tạo/copy/move/remove test file
7. Search file
8. Search text
9. Pipe output
10. Redirect output
```

Không cần nhớ mọi option.

Bạn phải biết dùng:

```bash
man
--help
```

khi quên.

---

# 218. Practical Acceptance Test — Task B

Bạn phải có thể thực hiện:

```text
Create controlled group
Create two controlled users
Add one user to group
Verify memberships
Configure root:group ownership
Set 640 file permissions
Authorized user reads successfully
Unauthorized user is denied
Authorized user cannot write
```

---

# 219. Practical Acceptance Test — Task C

Bạn phải giải thích và demonstrate:

```text
Current umask
Temporary change
Effect on newly-created file
Effect on newly-created directory
Restore original umask
```

---

# 220. Practical Acceptance Test — Task D

Bạn phải có thể thiết kế one-command `sudo` delegation.

Acceptance:

```text
Allowed sudo command → PASS
Other sudo command   → DENIED
sudoers syntax       → VALID
```

---

# 221. Practical Acceptance Test — Task E

Bạn phải có thể:

```text
Generate SSH key pair
Identify private/public key
Protect private key
Install public key
Verify fingerprint
Login with explicit key
Test unapproved key
Document allowed + denied result
Revoke lab key if required
```

---

# 222. Practical Acceptance Test — Task F

Cho bạn symptom:

```text
ops1 gets Permission denied
```

Bạn không được thay đổi permissions trong 30 giây đầu.

Bạn phải inspect:

```bash
id ops1
ls -ld <parent>
ls -l <file>
stat <file>
```

rồi mới đưa hypothesis.

Đây là test operational discipline.

---

# 223. Evidence Acceptance Criteria

Day 1 chưa hoàn thành nếu bạn chỉ nói:

```text
"Em đã làm xong."
```

Bạn phải chứng minh.

Tối thiểu evidence package cần thể hiện:

| Area              | Evidence                           |
| ----------------- | ---------------------------------- |
| System            | hostname, OS, kernel, timestamp    |
| Account           | `id`/`getent`                      |
| Group             | membership                         |
| Ownership         | owner/group                        |
| Permissions       | mode                               |
| Authorized access | successful test                    |
| Denied access     | failed test                        |
| `sudo`            | allowed + denied                   |
| SSH               | authorized + unauthorized key test |
| Cleanup           | lab objects removed/restored       |
| Commands          | execution record/journal           |

---

# 224. Một số lỗi thực hành phổ biến

### Chạy command trước khi kiểm tra hostname

Risk:

```text
wrong server
```

### Tạo user mà không kiểm tra existing user

Risk:

```text
naming collision
modify existing identity
```

### Dùng `usermod -G` mà không hiểu `-a`

Risk:

```text
remove existing supplementary memberships
```

### `chmod 777`

Risk:

```text
over-permission
```

### Chỉ test successful access

Risk:

```text
không chứng minh denied control hoạt động
```

### Dùng broad `sudo`

Risk:

```text
privilege escalation scope quá lớn
```

### Copy private SSH key lên server

Risk:

```text
credential compromise
```

### Disable SSH host verification

Risk:

```text
server identity không còn được xác minh
```

### Sửa `/etc/sudoers` trực tiếp mà không validate

Risk:

```text
sudo configuration break
```

### Xóa lab accounts nhưng không giữ evidence

Risk:

```text
không còn audit trail
```

---

# 225. Day 1 Practical Workflow hoàn chỉnh

Bạn nên thuộc workflow này hơn là thuộc từng command:

```text
┌─────────────────────────────┐
│ 1. Confirm authorization    │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 2. Confirm target/environment│
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 3. Prepare evidence repo    │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 4. Capture inventory        │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 5. CLI/file exercises       │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 6. Create controlled IDs    │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 7. Configure group access   │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 8. Configure permissions    │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 9. Test allow + deny        │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 10. Configure minimal sudo  │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 11. Test allow + deny sudo  │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 12. Generate SSH keys       │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 13. Configure public key    │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 14. Test allow + deny SSH   │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 15. Failure injection       │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 16. Troubleshoot/evidence   │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 17. Rollback/Cleanup        │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ 18. Verify clean state      │
└─────────────────────────────┘
```

---

# 226. Day 1 Practical Cheat Sheet

### Identity

```bash
whoami
id
id <user>
groups <user>

getent passwd <user>
getent group <group>
```

### System

```bash
hostname
hostnamectl
cat /etc/os-release
uname -r
uname -m
date -Is
```

### Navigation

```bash
pwd
ls
ls -l
ls -la
cd
```

### Files

```bash
touch
mkdir
cp
mv
rm
rmdir
stat
```

### Search

```bash
find
grep
```

### Text processing

```bash
head
tail
wc
sort
uniq
```

### Pipes/redirection

```text
|
>
>>
2>
```

### Users/groups

```bash
groupadd
useradd
usermod
userdel
groupdel
```

### Ownership

```bash
chown
chgrp
```

### Permissions

```bash
chmod
umask
```

### `sudo`

```bash
sudo -l
visudo -cf <file>
```

### SSH

```bash
ssh
ssh-keygen
ssh-copy-id
```

Useful verification:

```bash
ssh-keygen -lf <public-key>
```

---

# 227. Final Day 1 Practical Completion Checklist

Bạn chỉ nên coi Module Thực hành Day 1 hoàn thành khi có thể tick được:

### Environment preparation

- [ ] Tôi xác minh đúng lab target.
- [ ] Tôi xác minh current identity.
- [ ] Tôi biết mình có authorization gì.
- [ ] Tôi có evidence repository.
- [ ] Tôi đã ghi personal learning baseline.

### System inventory

- [ ] Hostname.
- [ ] OS distribution.
- [ ] OS version.
- [ ] Kernel.
- [ ] Architecture.
- [ ] CPU overview.
- [ ] Memory overview.
- [ ] Storage overview.
- [ ] Current identity.
- [ ] Timestamp.

### CLI

- [ ] `pwd`.
- [ ] `ls`.
- [ ] `cd`.
- [ ] absolute path.
- [ ] relative path.
- [ ] `.`
- [ ] `..`
- [ ] `~`.

### Files

- [ ] Create file.
- [ ] Create directory.
- [ ] Copy.
- [ ] Move.
- [ ] Rename.
- [ ] Delete controlled file.
- [ ] Remove controlled directory.
- [ ] Verify before/after.

### Search/text

- [ ] `find`.
- [ ] `grep`.
- [ ] pipe.
- [ ] stdout redirection.
- [ ] stderr redirection.
- [ ] append.
- [ ] simple counting/filtering.

### Users/groups

- [ ] Pre-check names.
- [ ] Create controlled group.
- [ ] Create controlled users.
- [ ] Add correct user to group.
- [ ] Verify membership.
- [ ] Verify unauthorized user is not member.

### Ownership/permissions

- [ ] `chown`.
- [ ] `chgrp`.
- [ ] `chmod`.
- [ ] `stat`.
- [ ] Positive read test.
- [ ] Negative read test.
- [ ] Negative write test.
- [ ] Parent directory permission test.

### `umask`

- [ ] Read current value.
- [ ] Change temporarily.
- [ ] Create new file/directory.
- [ ] Explain result.
- [ ] Restore original value.

### `sudo`

- [ ] Design exact least-privilege rule.
- [ ] Validate candidate sudoers syntax.
- [ ] Install controlled drop-in.
- [ ] Authorized sudo action succeeds.
- [ ] Unauthorized sudo action fails.
- [ ] Rollback sudo rule.
- [ ] Revalidate configuration.

### SSH

- [ ] Generate key pair.
- [ ] Know private vs public.
- [ ] Protect private key.
- [ ] Verify fingerprint.
- [ ] Install public key.
- [ ] Login with explicitly selected key.
- [ ] Unauthorized key fails.
- [ ] Understand host-key verification.
- [ ] Revoke lab key when required.

### Failure Injection

- [ ] Wrong group ownership.
- [ ] Directory traverse failure.
- [ ] Incorrect file mode.
- [ ] Invalid sudoers candidate.
- [ ] Unauthorized SSH key.
- [ ] Root cause identified before remediation.
- [ ] Corrective action verified.

### Evidence/Cleanup

- [ ] System inventory retained.
- [ ] Access verification retained.
- [ ] Configuration evidence retained.
- [ ] Positive tests retained.
- [ ] Negative tests retained.
- [ ] Failure evidence retained.
- [ ] Cleanup verified.
- [ ] No password/private key/token in evidence.

---

# 228. Tiêu chuẩn "Pass" thực sự của Day 1

Day 1 không nên được đánh giá bằng việc bạn nhớ:

```bash
chmod 640
```

hay:

```bash
useradd
```

Điểm quan trọng hơn là bạn có thể giải thích:

> Tại sao cần `640`, identity nào cần access, identity nào phải bị denied, current state là gì, bạn thay đổi gì, dùng quyền nào, verification nào chứng minh kết quả, evidence nào chứng minh operation, và rollback ra sao.

Một Infrastructure Operations fresher đạt yêu cầu Day 1 phải có tư duy:

```text
Never start with change.

Start with:
target
identity
scope
current state
authorization
```

rồi mới:

```text
change
```

và luôn kết thúc bằng:

```text
verification
+
evidence
+
cleanup/rollback
```

Đó chính là lý do Assignment Day 1 kết hợp `CLI`, `users/groups`, `permissions`, `sudo`, `SSH`, authorized/denied testing và evidence collection trong cùng một ngày: mục tiêu không phải học các commands rời rạc mà xây một **secure, controlled, auditable Linux administration workflow**. Infrastructure_Operations_14_Da…

Nếu bạn muốn, tôi có thể tra và đối chiếu thêm với **official Red Hat/OpenSSH documentation hiện hành** để xác minh các behavior/version-sensitive của `sudo`, `OpenSSH`, SSH key algorithms và RHEL trước khi bạn chạy lab trên một RHEL version cụ thể.
