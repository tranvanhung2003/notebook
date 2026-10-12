# Day 2 — Linux Fundamentals: Architecture, CLI, Files & Text Processing

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 2 thuộc phần **Linux Fundamentals** và yêu cầu bắt buộc chính xác:

> **Concept/Lecture:** Linux architecture, distributions, CLI navigation, files/directories, text processing, help, and command safety.  
> **Assignment/Lab:** Complete a command-line navigation and file-manipulation task set with a command log. 00_MASTER_SYLLABUS_Java_Fresher…

Tôi sẽ coi toàn bộ các phần trên là **Mandatory**. Những nội dung như `stdin/stdout/stderr`, pipeline, quoting, globbing, evidence collection, troubleshooting workflow và các kỹ thuật tránh xóa nhầm sẽ được đánh dấu là **Supplementary / Beyond the explicit syllabus** khi chúng mở rộng hơn câu chữ của syllabus, nhưng chúng rất quan trọng cho các Day sau.

Về môi trường, bài học giả định bạn đang sử dụng một Linux distribution phổ biến với **Bash** và GNU utilities, ví dụ Ubuntu/Debian hoặc RHEL/Rocky/AlmaLinux. GNU Coreutils hiện hành là 9.11; Bash manual hiện hành mô tả Bash 5.3. Cú pháp cơ bản chúng ta dùng hôm nay ổn định trên hầu hết Linux distribution. [gnu.org](https://www.gnu.org/software/coreutils/manual/coreutils.html?utm_source=chatgpt.com)

---

# 2. Objectives + Prerequisites

Sau Day 2, mục tiêu không phải chỉ là “biết vài command”. Bạn cần hình thành được mental model để khi SSH vào một Linux server sau này, bạn có thể tự trả lời:

**Tôi đang ở đâu? Tôi đang thao tác lên file nào? Command này đọc gì, ghi gì, có thay đổi hệ thống không? Nếu command thất bại thì evidence ở đâu? Nếu command nguy hiểm thì cách kiểm tra trước khi chạy là gì?**

Kết thúc bài, bạn cần có khả năng:

| Năng lực           | Kết quả cần đạt                                                             |
| ------------------ | --------------------------------------------------------------------------- |
| Linux architecture | Giải thích Kernel, user space, shell, utilities và hardware liên hệ thế nào |
| Distribution       | Hiểu distribution khác Linux kernel như thế nào                             |
| CLI                | Phân biệt terminal, shell, prompt, command                                  |
| Navigation         | Dùng thành thạo `pwd`, `cd`, `ls`, absolute/relative path                   |
| Files/directories  | Tạo, copy, move, rename, inspect, delete an toàn                            |
| Text               | Xem, lọc, đếm, sort và nối command                                          |
| Help               | Tự tra cứu `man`, `--help`, `help`, `apropos`, `type`                       |
| Command safety     | Nhận biết overwrite/delete/globbing/root risks                              |
| Troubleshooting    | Đọc error và xác định sai command, sai path hay sai assumption              |
| Lab evidence       | Hoàn thành lab và lưu command log                                           |

### Prerequisites

Bạn chỉ cần một Linux shell. **Không cần root và không cần `sudo` trong Day 2.**

Điều này rất quan trọng: thực hành filesystem cơ bản bằng root sẽ tạo ra blast radius không cần thiết.

---

# 3. Mental Model — Linux thực sự là gì?

Một mental model đơn giản nhưng rất hữu ích:

```text
+------------------------------------------------+
| Application                                    |
| Java / Tomcat / PostgreSQL / curl / ls / grep |
+------------------------------------------------+
                     |
              User-space APIs
                     |
+------------------------------------------------+
| Shell / CLI                                    |
| bash, sh, zsh                                  |
+------------------------------------------------+
                     |
                System calls
                     |
+------------------------------------------------+
| Linux Kernel                                   |
| CPU scheduling | memory | filesystem | network|
| processes      | drivers| security   | devices|
+------------------------------------------------+
                     |
+------------------------------------------------+
| Hardware                                       |
| CPU | RAM | disk | NIC | devices               |
+------------------------------------------------+
```

Linux kernel cung cấp các chức năng nền tảng như multitasking, virtual memory, networking, filesystem và hardware interaction. Tài liệu chính thức của kernel cũng tách rõ các interface dành cho user space và interface quản trị hệ thống. [Kernel](https://www.kernel.org/doc/html/latest/?utm_source=chatgpt.com)

Điểm rất quan trọng:

**Linux không đồng nghĩa với Ubuntu.**

Linux chính xác hơn là **kernel**. Ubuntu, Debian, Fedora, Rocky Linux… là **Linux distributions** gồm Linux kernel cộng với user-space utilities, libraries, package-management system, init/service-management system, configuration defaults và release policy.

---

# 4. Deep Theory

## 4.1 Kernel space và User space

Kernel chạy ở vùng đặc quyền cao, chịu trách nhiệm quản lý tài nguyên.

Application thông thường chạy trong **user space**.

Ví dụ khi chạy:

```bash
cat notes.txt
```

Không phải `cat` tự trực tiếp nói chuyện với SSD.

Mental flow gần đúng là:

```text
bash
  ↓
execute cat
  ↓
cat requests file access
  ↓
system call
  ↓
Linux kernel
  ↓
filesystem / cache / storage driver
  ↓
data returned
  ↓
cat writes output
  ↓
terminal
```

Điều này sẽ cực kỳ quan trọng về sau khi troubleshooting:

```text
Application
    ↓
OS API
    ↓
Kernel
    ↓
Filesystem/network/device
```

Bạn sẽ học sâu hơn về process ở Day 4, network/storage ở Day 5.

---

# 4.2 Distribution là gì?

Một distribution thường đóng gói:

```text
Linux kernel
+
GNU/Linux utilities
+
libraries
+
package manager
+
init/service system
+
configuration defaults
+
repositories
+
security/update policy
```

Ví dụ:

| Family  | Distribution ví dụ                   | Package ecosystem |
| ------- | ------------------------------------ | ----------------- |
| Debian  | Debian, Ubuntu                       | `.deb`, `apt`     |
| Red Hat | RHEL, Rocky Linux, AlmaLinux, Fedora | `.rpm`, `dnf`     |
| SUSE    | SLES, openSUSE                       | `.rpm`, `zypper`  |
| Arch    | Arch Linux                           | `pacman`          |

Package management sẽ học sâu hơn vào **Day 4**; hôm nay chỉ cần hiểu distribution có thể làm command/configuration hơi khác nhau.

Ví dụ cùng một mục tiêu “install package” nhưng command có thể khác.

```bash
apt install ...
```

so với:

```bash
dnf install ...
```

Nhưng những command nền tảng như:

```bash
pwd
cd
ls
cp
mv
rm
cat
grep
```

phần lớn có cách sử dụng tương tự giữa các distro Linux phổ biến.

---

# 4.3 Terminal ≠ Shell ≠ Command

Đây là câu hỏi fresher rất hay nhầm.

### Terminal

Terminal là interface mà bạn nhập và nhìn output.

Ví dụ:

```text
GNOME Terminal
Windows Terminal
iTerm2
SSH terminal session
```

### Shell

Shell là program nhận command của bạn và thực thi/điều phối chúng.

Ví dụ:

```text
bash
sh
zsh
fish
```

Trong server Linux, Bash cực kỳ phổ biến.

Kiểm tra shell hiện tại:

```bash
echo "$SHELL"
```

Có thể thấy:

```text
/bin/bash
```

Kiểm tra process shell hiện tại:

```bash
ps -p $$ -o comm=
```

Ví dụ:

```text
bash
```

### Prompt

Ví dụ:

```text
student@linux01:~$
```

Prompt chỉ là giao diện hiển thị trạng thái.

Thông thường:

```text
student     user
linux01     hostname
~           working directory
$           normal user
```

Root thường có prompt kết thúc bằng:

```text
#
```

Nhưng **không được dựa vào `$` hoặc `#` như cơ chế security tuyệt đối** vì prompt có thể tùy chỉnh.

---

# 4.4 Shell builtin và external command

Một command có thể là shell builtin hoặc external executable.

Ví dụ:

```bash
type cd
```

Có thể trả về:

```text
cd is a shell builtin
```

Trong khi:

```bash
type ls
```

có thể trả về:

```text
ls is aliased to `ls --color=auto'
```

hoặc:

```text
ls is /usr/bin/ls
```

Tại sao `cd` thường phải là builtin?

Bởi vì `cd` cần thay đổi **current working directory của chính shell hiện tại**. Một process con không thể chạy xong rồi thay đổi working directory của parent shell. Bash documentation cũng nêu `cd` là một trong các builtin cần thao tác trực tiếp trên shell environment. [gnu.org](https://www.gnu.org/s/bash/manual/bash.html?utm_source=chatgpt.com)

Command rất hữu ích:

```bash
type command
```

Ví dụ:

```bash
type cd
type pwd
type ls
type grep
```

---

# 4.5 Command anatomy

Command thường có dạng:

```text
command [options] [arguments]
```

Ví dụ:

```bash
ls -l /var/log
```

Trong đó:

```text
ls         command
-l         option
/var/log   argument
```

Hoặc:

```bash
cp -v source.txt backup.txt
```

Ở đây:

```text
cp           command
-v           verbose option
source.txt   source
backup.txt   destination
```

GNU Bash mô tả simple command theo mô hình command + arguments được phân cách bằng shell syntax. [gnu.org](https://www.gnu.org/s/bash/manual/html_node/Shell-Commands.html?utm_source=chatgpt.com)

---

# 4.6 Linux filesystem là một cây duy nhất

Linux filesystem không dùng kiểu:

```text
C:
D:
E:
```

như Windows.

Mental model:

```text
/
├── etc
├── home
│   └── student
├── var
│   └── log
├── usr
├── tmp
├── proc
├── sys
└── dev
```

Tất cả bắt đầu từ:

```text
/
```

gọi là **root directory**.

Không được nhầm:

```text
/
```

với:

```text
/root
```

`/` là filesystem root.

`/root` thường là home directory của user `root`.

Linux `hier(7)` mô tả filesystem tree bắt đầu tại `/`, đồng thời mô tả `/dev`, `/proc`, `/run`, `/sys`, `/tmp`, `/opt`, `/root` và các directory chuẩn khác. [man7.org](https://man7.org/linux/man-pages/man7/hier.7.html?utm_source=chatgpt.com)

---

# 4.7 Những directory quan trọng

| Path       | Ý nghĩa cơ bản                                               |
| ---------- | ------------------------------------------------------------ |
| `/`        | filesystem root                                              |
| `/home`    | home của regular users                                       |
| `/root`    | home của root                                                |
| `/etc`     | system/application configuration                             |
| `/var`     | variable data                                                |
| `/var/log` | log files truyền thống                                       |
| `/tmp`     | temporary files                                              |
| `/usr`     | phần lớn user-space programs/data                            |
| `/bin`     | essential command location hoặc symlink trên distro hiện đại |
| `/sbin`    | system utilities hoặc symlink trên distro hiện đại           |
| `/dev`     | device files                                                 |
| `/proc`    | virtual filesystem chứa kernel/process information           |
| `/sys`     | kernel/device information qua sysfs                          |
| `/run`     | runtime state kể từ khi boot                                 |
| `/opt`     | optional/add-on software                                     |
| `/mnt`     | thường dùng temporary mount point                            |
| `/boot`    | boot-related files                                           |

### `/proc` không phải directory bình thường trên disk

Ví dụ:

```bash
cat /proc/cpuinfo
```

hoặc:

```bash
cat /proc/meminfo
```

`/proc` là **pseudo-filesystem** cung cấp interface tới kernel/process information; phần lớn entries là read-only nhưng có một số writable kernel interfaces. [man7.org](https://www.man7.org/linux/man-pages/man5/procfs.5.html?utm_source=chatgpt.com)

Day 2 chỉ cần nhận biết điều này. Đừng tự ý ghi vào `/proc/sys/...`.

---

# 4.8 Current Working Directory

Mỗi shell process có một current working directory.

Kiểm tra:

```bash
pwd
```

Ví dụ:

```text
/home/student
```

`pwd` = **print working directory**.

Đây nên là một trong những command đầu tiên bạn chạy trước thao tác có nguy cơ thay đổi file.

---

# 4.9 Absolute path và Relative path

Đây là kiến thức bắt buộc phải thật chắc.

## Absolute path

Bắt đầu bằng:

```text
/
```

Ví dụ:

```text
/etc/hosts
/home/student/report.txt
/var/log
```

Nó không phụ thuộc current directory.

## Relative path

Không bắt đầu bằng `/`.

Ví dụ:

```text
report.txt
logs/app.log
../backup
```

Nó được giải từ current working directory.

Ví dụ hiện tại:

```text
/home/student
```

Command:

```bash
cat notes/today.txt
```

được hiểu là:

```text
/home/student/notes/today.txt
```

Linux pathname resolution bắt đầu tại filesystem root khi pathname bắt đầu bằng `/`; nếu không, resolution bắt đầu từ current working directory. [man7.org](https://man7.org/linux/man-pages/man7/path_resolution.7.html?utm_source=chatgpt.com)

---

# 4.10 Những ký hiệu path quan trọng

```text
.     current directory
..    parent directory
~     current user's home directory
/     root directory
```

Ví dụ:

```bash
cd .
```

Không thay đổi vị trí.

```bash
cd ..
```

Đi lên một directory.

```bash
cd ~
```

Về home.

Có thể viết đơn giản:

```bash
cd
```

Bash cũng đưa bạn về home.

Ví dụ:

```bash
cd /var/log
pwd
```

Output:

```text
/var/log
```

Sau đó:

```bash
cd ..
pwd
```

Output:

```text
/var
```

---

# 4.11 `ls` — nhìn trước khi thao tác

Cơ bản:

```bash
ls
```

Chi tiết:

```bash
ls -l
```

Hidden files:

```bash
ls -a
```

Thường dùng:

```bash
ls -la
```

Human-readable size:

```bash
ls -lh
```

Directory itself thay vì contents:

```bash
ls -ld /var/log
```

GNU `ls` mặc định hoạt động trên current directory nếu không truyền argument và mặc định không hiển thị filename bắt đầu bằng `.`. [gnu.org](https://www.gnu.org/software/coreutils/manual/html_node/ls-invocation.html?utm_source=chatgpt.com)

Ví dụ:

```text
-rw-r--r-- 1 student student 128 Oct 12 10:00 notes.txt
```

Day 2 chỉ cần nhận biết rằng dòng này chứa metadata.

Permissions/ownership:

```text
-rw-r--r--
student student
```

sẽ được học sâu vào **Day 3**.

---

# 4.12 Hidden files

Trên Linux, filename bắt đầu bằng:

```text
.
```

thường được xem là hidden.

Ví dụ:

```text
.bashrc
.profile
.ssh
```

Không có special “hidden attribute” bắt buộc như Windows.

Command:

```bash
ls
```

có thể không thấy chúng.

Command:

```bash
ls -la
```

sẽ thấy.

---

# 4.13 File name là case-sensitive

Thông thường Linux filesystem phân biệt:

```text
Report.txt
report.txt
REPORT.txt
```

thành ba filename khác nhau.

Đây là lỗi rất phổ biến khi người mới chuyển từ Windows sang Linux.

Ví dụ:

```bash
cat App.log
```

không tương đương:

```bash
cat app.log
```

---

# 4.14 Tạo directory — `mkdir`

Tạo một directory:

```bash
mkdir reports
```

Tạo nhiều directory:

```bash
mkdir logs backups data
```

Nested directory:

```bash
mkdir -p app/log/archive
```

`-p` cho phép tạo parent directories cần thiết.

Verification:

```bash
ls -ld app app/log app/log/archive
```

---

# 4.15 Tạo file — `touch`

Ví dụ:

```bash
touch notes.txt
```

Nếu file chưa tồn tại, file rỗng được tạo.

Kiểm tra:

```bash
ls -l notes.txt
```

Nếu file đã tồn tại, `touch` chủ yếu update timestamps theo mặc định.

Vì vậy đừng nghĩ `touch existing-file` nghĩa là “recreate/empty file”.

Nó **không truncate file**.

---

# 4.16 Xem loại file — `file`

Ví dụ:

```bash
file /etc/hosts
```

Có thể thấy:

```text
/etc/hosts: ASCII text
```

Hoặc:

```bash
file /bin/ls
```

thường trả về thông tin ELF executable.

Đây là command troubleshooting rất hữu ích.

Tên file có `.txt` không đảm bảo nó thực sự là text file.

---

# 4.17 Copy — `cp`

Cơ bản:

```bash
cp source.txt destination.txt
```

Copy vào directory:

```bash
cp report.txt backup/
```

Verbose:

```bash
cp -v report.txt backup/
```

Directory recursion:

```bash
cp -R project backup/
```

GNU Coreutils xác nhận `cp` tạo một copy độc lập; việc copy directory yêu cầu recursive option như `-R`, `-r` hoặc `-a`. [gnu.org](https://www.gnu.org/s/coreutils/manual/html_node/cp-invocation.html?utm_source=chatgpt.com)

### Cảnh báo

Command:

```bash
cp source.txt destination.txt
```

có thể **overwrite destination**.

Trước thao tác quan trọng:

```bash
ls -l -- source.txt destination.txt
```

Sau đó có thể sử dụng:

```bash
cp -i source.txt destination.txt
```

để có interactive protection trong lab.

Production workflow tốt hơn không nên dựa hoàn toàn vào confirmation prompt; phải hiểu chính xác source/destination và rollback.

---

# 4.18 Move và rename — `mv`

Rename:

```bash
mv old.txt new.txt
```

Move:

```bash
mv report.txt archive/
```

Move + rename:

```bash
mv report.txt archive/report-2026.txt
```

Kiểm tra trước:

```bash
ls -l old.txt new.txt
```

Interactive:

```bash
mv -i old.txt new.txt
```

`mv` cũng có nguy cơ overwrite destination.

---

# 4.19 Remove file — `rm`

## Đây là command nguy hiểm đầu tiên của Day 2.

Xóa file:

```bash
rm test.txt
```

Không được hình thành thói quen:

```bash
rm -rf ...
```

cho mọi trường hợp.

GNU Coreutils xác nhận `rm` mặc định xóa files, không xóa directory nếu không sử dụng recursive option. `-I` có thể yêu cầu confirmation cho một số recursive hoặc multi-file operations. [gnu.org](https://www.gnu.org/s/coreutils/manual/html_node/rm-invocation.html?utm_source=chatgpt.com)

Safe lab pattern:

```bash
pwd
ls -la
rm -i test.txt
ls -la
```

---

# 4.20 Xóa directory

Empty directory:

```bash
rmdir empty-dir
```

Nếu không empty:

```text
Directory not empty
```

Recursive delete:

```bash
rm -r directory
```

Đây là destructive operation.

Trước khi chạy:

```bash
pwd
ls -la directory
```

Nếu có wildcard:

```bash
printf '%s\n' directory/*
```

để xem shell sẽ expand thành những gì.

Trong Day 2 lab, **không sử dụng `sudo rm -rf`**.

---

# 4.21 `--` — một safety detail rất đáng học

Giả sử bạn có filename:

```text
-f
```

Command:

```bash
rm -f
```

sẽ được interpret là option chứ không phải filename.

Dùng:

```bash
rm -- -f
```

`--` thường đánh dấu:

```text
end of options
```

Sau đó các argument còn lại được xử lý như operands.

Safe habit:

```bash
rm -- "$filename"
cp -- "$source" "$destination"
```

đặc biệt hữu ích trong scripting sau này.

**Supplementary / Beyond explicit syllabus**, nhưng rất có giá trị cho command safety.

---

# 4.22 Quoting — cực kỳ quan trọng

Giả sử file:

```text
monthly report.txt
```

Sai:

```bash
cat monthly report.txt
```

Shell nhìn thấy hai arguments:

```text
monthly
report.txt
```

Đúng:

```bash
cat "monthly report.txt"
```

hoặc:

```bash
cat 'monthly report.txt'
```

## Single quotes

```bash
echo '$HOME'
```

Output:

```text
$HOME
```

## Double quotes

```bash
echo "$HOME"
```

Output có thể là:

```text
/home/student
```

Mental rule ban đầu:

> Khi truyền variable chứa filename/path, mặc định hãy nghĩ tới `"double quotes"`.

Ví dụ:

```bash
file="$HOME/my reports/report.txt"
cat "$file"
```

Không nên:

```bash
cat $file
```

Quoting và shell expansion được Bash xác định là thành phần cốt lõi của shell syntax. [gnu.org](https://www.gnu.org/software/bash/manual/html_node/Basic-Shell-Features.html?utm_source=chatgpt.com)

---

# 4.23 Globbing / wildcard

Shell có thể expand patterns.

```text
*      zero or more characters
?      exactly one character
[...]  character set/range
```

Ví dụ:

```bash
ls *.log
```

Có thể expand thành:

```bash
ls app.log access.log error.log
```

**Quan trọng:** thông thường chính shell expand `*.log` trước, không phải `ls`.

Do đó:

```bash
rm *.log
```

có thể rất nguy hiểm.

Inspect trước:

```bash
printf '%s\n' *.log
```

sau đó mới quyết định remove.

Ví dụ:

```bash
ls report?.txt
```

có thể match:

```text
report1.txt
reportA.txt
```

nhưng không match:

```text
report10.txt
```

---

# 4.24 Text processing mental model

Unix/Linux CLI rất mạnh vì command thường xử lý **text streams**.

Mental model:

```text
INPUT
  ↓
command 1
  ↓
text stream
  ↓
command 2
  ↓
text stream
  ↓
OUTPUT
```

Ví dụ:

```bash
cat servers.txt | sort
```

rồi:

```bash
cat servers.txt | sort | uniq
```

và:

```bash
cat servers.txt | sort | uniq | wc -l
```

Đây là triết lý compose các tool nhỏ.

---

# 4.25 stdin, stdout, stderr

**Supplementary / Beyond the explicit syllabus**, nhưng cần biết để thực sự hiểu text processing.

Mỗi command thường có ba standard streams:

| Stream | File descriptor | Ý nghĩa                 |
| ------ | --------------: | ----------------------- |
| stdin  |               0 | input                   |
| stdout |               1 | normal output           |
| stderr |               2 | error/diagnostic output |

Ví dụ:

```bash
cat /etc/hosts
```

normal output đi ra:

```text
stdout
```

Nếu:

```bash
cat /does/not/exist
```

error message đi ra:

```text
stderr
```

---

# 4.26 Redirection

### stdout → file

```bash
echo "hello" > output.txt
```

**Cảnh báo:** `>` tạo file nếu chưa tồn tại nhưng sẽ **truncate/overwrite** nội dung file hiện có.

Bash documentation xác nhận output redirection `>` mở file để ghi và file hiện có sẽ bị truncate theo hành vi mặc định. [gnu.org](https://www.gnu.org/s/bash/manual/html_node/Redirections.html?utm_source=chatgpt.com)

Append:

```bash
echo "second line" >> output.txt
```

Input:

```bash
sort < names.txt
```

stderr:

```bash
command 2> error.log
```

stdout:

```bash
command > output.log
```

Cả stdout + stderr trong Bash:

```bash
command > all.log 2>&1
```

Thứ tự redirection có ý nghĩa. Bash xử lý redirections từ trái sang phải. [gnu.org](https://www.gnu.org/s/bash/manual/html_node/Redirections.html?utm_source=chatgpt.com)

---

# 4.27 Pipeline `|`

Ví dụ:

```bash
cat app.log | grep ERROR
```

`stdout` của command bên trái trở thành `stdin` của command bên phải.

Bash định nghĩa pipeline là một chuỗi command nối bởi `|` hoặc `|&`, trong đó output của command trước được nối vào input của command tiếp theo. [gnu.org](https://www.gnu.org/software/bash/manual/html_node/Pipelines?utm_source=chatgpt.com)

Ví dụ:

```bash
grep ERROR app.log | wc -l
```

Mental flow:

```text
app.log
   ↓
grep ERROR
   ↓
matching lines
   ↓
wc -l
   ↓
number of matching lines
```

---

# 4.28 `cat`

Hiển thị file nhỏ:

```bash
cat notes.txt
```

Có thể concatenate:

```bash
cat part1.txt part2.txt
```

Không nên dùng `cat` để dump file vài GB vào terminal.

Với file lớn, dùng:

```bash
less huge.log
```

---

# 4.29 `less`

```bash
less /var/log/some.log
```

Các key quan trọng:

```text
Space     next page
b         previous page
/search   search
n         next match
q         quit
```

`less` rất quan trọng khi đọc logs.

---

# 4.30 `head` và `tail`

Đầu file:

```bash
head file.txt
```

10 lines đầu mặc định.

Chọn số:

```bash
head -n 5 file.txt
```

Cuối file:

```bash
tail file.txt
```

20 lines cuối:

```bash
tail -n 20 file.txt
```

Theo dõi appended log:

```bash
tail -f application.log
```

`tail -f` sẽ trở nên cực kỳ quan trọng khi học Tomcat, PostgreSQL và application logs.

Dừng:

```text
Ctrl+C
```

---

# 4.31 `wc`

Word count:

```bash
wc file.txt
```

Thông thường cung cấp line/word/byte information.

Đếm lines:

```bash
wc -l file.txt
```

Ví dụ:

```bash
grep ERROR app.log | wc -l
```

---

# 4.32 `sort`

```bash
sort names.txt
```

Reverse:

```bash
sort -r names.txt
```

Numeric:

```bash
sort -n numbers.txt
```

Chú ý:

```bash
sort
```

mặc định là lexical/text-oriented ordering, không phải lúc nào cũng numeric.

Ví dụ:

```text
1
10
2
```

Có thể khác mong đợi nếu không dùng:

```bash
sort -n
```

---

# 4.33 `uniq`

Ví dụ file:

```text
server1
server1
server2
server1
```

Command:

```bash
uniq file.txt
```

không loại bỏ tất cả duplicate globally.

`uniq` xử lý **adjacent duplicate lines**.

Pattern thường gặp:

```bash
sort file.txt | uniq
```

Đếm:

```bash
sort file.txt | uniq -c
```

Đây là câu hỏi practical/exam rất tốt.

---

# 4.34 `grep`

Syllabus Day 6 có nhắc riêng `grep/awk/sed`, vì vậy hôm nay chúng ta chỉ dùng **basic grep** cho text processing; phần sâu sẽ để Day 6.

Search:

```bash
grep ERROR app.log
```

Line number:

```bash
grep -n ERROR app.log
```

Case insensitive:

```bash
grep -i error app.log
```

Invert:

```bash
grep -v DEBUG app.log
```

Fixed string:

```bash
grep -F 'ERROR [DB]' app.log
```

`-F` hữu ích khi bạn không muốn pattern bị interpret như regular expression.

---

# 4.35 `cut`

Ví dụ:

```text
alice:developer
bob:admin
carol:tester
```

Command:

```bash
cut -d: -f1 users.txt
```

Output:

```text
alice
bob
carol
```

Trong đó:

```text
-d:     delimiter :
-f1     field 1
```

Ví dụ real Linux:

```bash
cut -d: -f1 /etc/passwd
```

Chỉ đọc; an toàn.

---

# 4.36 `tee`

Command:

```bash
some_command | tee output.txt
```

cho phép vừa:

```text
hiển thị terminal
```

vừa:

```text
ghi vào file
```

Append:

```bash
some_command | tee -a output.txt
```

Rất hữu ích cho evidence.

---

# 4.37 `echo` và `printf`

`echo`:

```bash
echo "hello"
```

`printf` predictable hơn khi formatting:

```bash
printf '%s\n' "hello"
```

Tạo sample data:

```bash
printf '%s\n' server1 server2 server1 > servers.txt
```

Khi scripting sau này, `printf` thường đáng tin cậy hơn `echo` cho formatting phức tạp.

---

# 4.38 Help system — kỹ năng bắt buộc

Một administrator tốt **không phải người thuộc hết options**.

Người tốt là người biết tìm đúng documentation.

## `command --help`

Ví dụ:

```bash
ls --help
```

```bash
cp --help
```

```bash
sort --help
```

Thường phù hợp khi cần tìm nhanh option.

---

## `man`

```bash
man ls
```

Search trong man:

```text
/pattern
```

Next:

```text
n
```

Quit:

```text
q
```

Ví dụ:

```bash
man cp
man rm
man bash
```

---

# 4.39 Man sections

Một tên có thể tồn tại trong nhiều manual section.

Phổ biến:

```text
1  executable programs / shell commands
2  system calls
3  library calls
4  devices
5  file formats/config formats
7  conventions / miscellaneous
8  administration commands
```

Ví dụ:

```bash
man 5 passwd
```

khác với:

```bash
man 1 passwd
```

Đây là kỹ năng Linux administration rất quan trọng.

---

# 4.40 `help` cho Bash builtin

Vì `cd` là Bash builtin:

```bash
help cd
```

thường phù hợp hơn:

```bash
cd --help
```

Có thể:

```bash
help pwd
help type
help history
```

---

# 4.41 `apropos`

Không biết command nhưng biết ý tưởng?

Ví dụ:

```bash
apropos "copy files"
```

`apropos` search tên và short description của manual pages. [man7.org](https://www.man7.org/linux/man-pages/man1/apropos.1.html?utm_source=chatgpt.com)

Ví dụ khác:

```bash
apropos filesystem
apropos directory
```

---

# 4.42 `type` trước `which`

Để biết shell thực sự sẽ chạy cái gì:

```bash
type ls
type cd
type grep
```

`type` có thể phát hiện:

```text
alias
function
builtin
executable
```

nên trong Bash troubleshooting thường hữu ích hơn chỉ dùng `which`.

---

# 5. Command Reference — trạng thái và mức độ nguy hiểm

| Command | Purpose                       | Thay đổi state? | Risk          |
| ------- | ----------------------------- | --------------: | ------------- |
| `pwd`   | current directory             |              No | Very low      |
| `ls`    | list files                    |              No | Very low      |
| `cd`    | đổi cwd của shell             |     Shell state | Low           |
| `file`  | inspect file type             |              No | Low           |
| `cat`   | display/concatenate           |  No nếu chỉ đọc | Low           |
| `less`  | inspect text                  |              No | Low           |
| `head`  | first lines                   |              No | Low           |
| `tail`  | last lines                    |              No | Low           |
| `grep`  | search text                   |              No | Low           |
| `sort`  | sort stream                   |   No nếu stdout | Low           |
| `uniq`  | adjacent duplicate processing |              No | Low           |
| `wc`    | count                         |              No | Low           |
| `mkdir` | create directory              |             Yes | Low           |
| `touch` | create/update timestamp       |             Yes | Low           |
| `cp`    | create/overwrite destination  |             Yes | Medium        |
| `mv`    | rename/move/overwrite         |             Yes | Medium        |
| `>`     | create/truncate destination   |             Yes | **High**      |
| `>>`    | append                        |             Yes | Medium        |
| `rmdir` | remove empty directory        |             Yes | Medium        |
| `rm`    | remove files                  |             Yes | **High**      |
| `rm -r` | recursive delete              |             Yes | **Very high** |

---

# 6. Command Safety — phần quan trọng nhất hôm nay

Trước một destructive command, hãy tự ép mình chạy mental checklist:

```text
WHERE am I?
WHAT exactly will be affected?
DO I need privilege?
CAN the target already contain important data?
WHAT is my rollback?
HOW will I verify afterward?
```

Một pattern cực kỳ tốt:

```bash
pwd
ls -la
```

rồi mới:

```bash
cp ...
mv ...
rm ...
```

Nếu dùng wildcard:

```bash
printf '%s\n' *.log
```

rồi mới:

```bash
rm -- *.log
```

nếu thật sự đúng mục tiêu.

---

## Đặc biệt nguy hiểm: `rm -rf`

`-r`:

```text
recursive
```

`-f`:

```text
force
```

Combination này làm giảm nhiều safety barriers.

Bạn **không cần `rm -rf` cho hầu hết Day 2 lab**.

Không hình thành reflex:

```bash
sudo rm -rf ...
```

khi directory không xóa được.

Nếu operation fail, hãy điều tra lý do.

---

# 7. Guided Lab — Day 2 Mandatory Assignment

Bây giờ là phần thực hành chính thức của hôm nay.

Mục tiêu:

```text
CLI navigation
+
file/directory manipulation
+
text processing
+
help
+
safe deletion
+
command log
+
verification evidence
```

## 7.1 Safety assumptions

Lab chỉ làm trong:

```text
~/labs/day2
```

Không sử dụng:

```text
sudo
```

Không sửa:

```text
/etc
/usr
/var
/proc
/sys
```

Chỉ đọc một số file system khi cần.

---

# 7.2 Tạo workspace

Chạy:

```bash
cd ~
mkdir -p ~/labs/day2
cd ~/labs/day2
pwd
```

**Expected invariant:**

Output cuối phải kết thúc bằng:

```text
/labs/day2
```

Ví dụ:

```text
/home/student/labs/day2
```

---

# 7.3 Bắt đầu command/session log

Assignment yêu cầu **command log**.

Một cách rất hữu ích trên Linux là utility `script`.

Kiểm tra:

```bash
command -v script
```

Nếu có output tương tự:

```text
/usr/bin/script
```

thì chạy:

```bash
mkdir -p evidence
script -q evidence/terminal-session.log
```

Từ thời điểm này terminal session sẽ được record.

Khi hoàn tất lab, dùng:

```bash
exit
```

để đóng recording.

Nếu máy không có `script`, cuối lab có thể dùng Bash:

```bash
history > evidence/command-history.txt
```

`script` tốt hơn cho evidence vì có thể giữ cả command và terminal output.

**Supplementary / Beyond explicit syllabus:** đây là kỹ thuật evidence collection để thực hiện đúng yêu cầu “command log”.

---

# 7.4 Xác minh context

```bash
whoami
pwd
printf 'Shell: %s\n' "$SHELL"
uname -s
```

Expected invariants:

```text
whoami    -> normal lab user
pwd       -> ~/labs/day2
uname -s  -> Linux
```

Không cần output giống hoàn toàn ví dụ.

---

# 7.5 Tạo directory structure

```bash
mkdir -p workspace/{input,output,archive,backup}
```

Kiểm tra:

```bash
find workspace -maxdepth 2 -type d
```

Nếu muốn chỉ dùng command core cơ bản:

```bash
ls -la workspace
```

Expected structure:

```text
workspace/
├── archive/
├── backup/
├── input/
└── output/
```

`find` ở đây là **Supplementary**; chúng ta sẽ chưa đi sâu syntax.

---

# 7.6 Tạo sample text

```bash
printf '%s\n' \
'INFO application started' \
'ERROR database unavailable' \
'INFO retry started' \
'ERROR database unavailable' \
'WARN disk usage high' \
'INFO application ready' \
> workspace/input/app.log
```

Tạo server list:

```bash
printf '%s\n' \
'app02' \
'app01' \
'db01' \
'app01' \
'app03' \
'db01' \
> workspace/input/servers.txt
```

Kiểm tra:

```bash
ls -lh workspace/input
```

---

# 7.7 Inspect file

```bash
file workspace/input/app.log
wc -l workspace/input/app.log
cat workspace/input/app.log
```

Expected invariant:

```text
6 lines
```

Không cần file-size output giống máy khác.

---

# 7.8 Head/tail

```bash
head -n 2 workspace/input/app.log
```

Expected:

```text
INFO application started
ERROR database unavailable
```

Sau đó:

```bash
tail -n 2 workspace/input/app.log
```

Expected:

```text
WARN disk usage high
INFO application ready
```

---

# 7.9 Basic grep

```bash
grep ERROR workspace/input/app.log
```

Expected two lines.

Đếm:

```bash
grep ERROR workspace/input/app.log | wc -l
```

Expected invariant:

```text
2
```

Line numbers:

```bash
grep -n ERROR workspace/input/app.log
```

Expected:

```text
2:ERROR database unavailable
4:ERROR database unavailable
```

---

# 7.10 Redirect filtered result

```bash
grep ERROR workspace/input/app.log > workspace/output/errors.txt
```

Verification:

```bash
cat workspace/output/errors.txt
wc -l workspace/output/errors.txt
```

Expected:

```text
2
```

---

# 7.11 Append

```bash
printf '%s\n' 'ERROR timeout contacting service' >> workspace/output/errors.txt
```

Verify:

```bash
tail -n 1 workspace/output/errors.txt
wc -l workspace/output/errors.txt
```

Expected count:

```text
3
```

Đây là lúc cần hiểu khác biệt:

```text
>     overwrite/truncate
>>    append
```

---

# 7.12 Sort và uniq

Xem input:

```bash
cat workspace/input/servers.txt
```

Sort:

```bash
sort workspace/input/servers.txt
```

Unique:

```bash
sort workspace/input/servers.txt | uniq
```

Count:

```bash
sort workspace/input/servers.txt | uniq -c
```

Bạn nên thấy đại ý:

```text
2 app01
1 app02
1 app03
2 db01
```

Thứ tự có thể phụ thuộc locale trong các tình huống text phức tạp; GNU `ls` và các tool sorting liên quan cũng có thể chịu ảnh hưởng locale. [gnu.org](https://www.gnu.org/software/coreutils/manual/html_node/ls-invocation.html?utm_source=chatgpt.com)

---

# 7.13 Tạo unique server inventory

```bash
sort workspace/input/servers.txt |
uniq > workspace/output/unique-servers.txt
```

Verify:

```bash
cat workspace/output/unique-servers.txt
wc -l workspace/output/unique-servers.txt
```

Expected:

```text
4
```

---

# 7.14 Copy

```bash
cp -v workspace/output/unique-servers.txt workspace/backup/
```

Verification:

```bash
ls -l workspace/backup
cat workspace/backup/unique-servers.txt
```

---

# 7.15 Rename

```bash
mv workspace/backup/unique-servers.txt \
   workspace/backup/server-inventory.txt
```

Verify:

```bash
ls -l workspace/backup
```

Bạn phải thấy:

```text
server-inventory.txt
```

và không còn:

```text
unique-servers.txt
```

trong directory đó.

---

# 7.16 Relative navigation

```bash
cd workspace/input
pwd
```

Sau đó:

```bash
cd ../output
pwd
```

Sau đó:

```bash
cd ../backup
pwd
```

Sau đó:

```bash
cd ../..
pwd
```

Bạn phải quay về:

```text
~/labs/day2
```

Đây là bài kiểm tra quan trọng về `..`.

---

# 7.17 Absolute navigation

```bash
cd /tmp
pwd
```

Sau đó:

```bash
cd ~/labs/day2
pwd
```

Bạn cần giải thích được tại sao:

```bash
cd /tmp
```

không phụ thuộc current directory, còn:

```bash
cd workspace
```

có phụ thuộc.

Ubuntu documentation cũng dùng chính loại exercise này để minh họa absolute và relative paths. [Ubuntu](https://ubuntu.com/desktop/docs/en/latest/tutorial/the-linux-command-line-for-beginners/?utm_source=chatgpt.com)

---

# 7.18 Filename có space

```bash
touch "workspace/input/monthly report.txt"
```

Check:

```bash
ls -l workspace/input
```

Đọc:

```bash
cat "workspace/input/monthly report.txt"
```

Copy:

```bash
cp "workspace/input/monthly report.txt" \
   "workspace/backup/monthly report.txt"
```

Verify:

```bash
ls -l workspace/backup
```

Mục tiêu ở đây là chứng minh bạn hiểu quoting.

---

# 7.19 Hidden file

```bash
touch workspace/input/.lab-metadata
```

Compare:

```bash
ls workspace/input
```

với:

```bash
ls -la workspace/input
```

Bạn phải giải thích được tại sao command thứ nhất không hiển thị `.lab-metadata`.

---

# 7.20 Help task

Không copy từ bài giảng.

Tự tìm hiểu option `-i` của `rm`:

```bash
rm --help
```

hoặc:

```bash
man rm
```

Tìm hiểu `-n` của `head`:

```bash
man head
```

Kiểm tra `cd`:

```bash
help cd
```

Kiểm tra command type:

```bash
type cd
type ls
type grep
```

---

# 7.21 Safe deletion exercise

Tạo file:

```bash
touch workspace/output/delete-me.tmp
```

Trước khi xóa:

```bash
pwd
ls -l workspace/output/delete-me.tmp
```

Xóa interactive:

```bash
rm -i workspace/output/delete-me.tmp
```

Confirm.

Verify:

```bash
ls -l workspace/output/delete-me.tmp
```

Expected result:

```text
No such file or directory
```

Ở đây error này **là evidence thành công** vì mục tiêu là file không còn tồn tại.

---

# 7.22 Empty-directory removal

```bash
mkdir workspace/output/empty-dir
rmdir workspace/output/empty-dir
```

Verify:

```bash
ls -ld workspace/output/empty-dir
```

Expected:

```text
No such file or directory
```

---

# 7.23 Tạo evidence summary

```bash
{
    printf '%s\n' '=== DAY 2 EVIDENCE ==='
    printf '%s\n' '--- Current directory ---'
    pwd
    printf '%s\n' '--- Workspace ---'
    find workspace -maxdepth 2 -print
    printf '%s\n' '--- Error count ---'
    grep ERROR workspace/input/app.log | wc -l
    printf '%s\n' '--- Unique server count ---'
    wc -l workspace/output/unique-servers.txt
} | tee evidence/verification.txt
```

`find` và grouped command block là supplementary techniques, nhưng evidence rất hữu ích.

Expected invariants:

```text
ERROR count = 2
unique server count = 4
workspace exists
backup/server-inventory.txt exists
```

---

# 7.24 Save command history

Nếu đang dùng `script`, kết thúc:

```bash
exit
```

Sau đó:

```bash
cd ~/labs/day2
ls -lh evidence
```

Bạn nên có:

```text
terminal-session.log
verification.txt
```

Có thể bổ sung:

```bash
history > evidence/command-history.txt
```

---

# 8. Verification Checklist

Day 2 lab **chưa đạt** chỉ vì command “không báo lỗi”.

Bạn cần chứng minh state cuối.

Chạy:

```bash
cd ~/labs/day2

test -d workspace/input &&
echo "PASS: input exists"

test -f workspace/input/app.log &&
echo "PASS: app.log exists"

test -f workspace/output/errors.txt &&
echo "PASS: errors.txt exists"

test -f workspace/output/unique-servers.txt &&
echo "PASS: unique server list exists"

test -f workspace/backup/server-inventory.txt &&
echo "PASS: backup inventory exists"
```

Kiểm tra dữ liệu:

```bash
grep ERROR workspace/input/app.log | wc -l
```

Expected:

```text
2
```

```bash
wc -l < workspace/output/errors.txt
```

Expected:

```text
3
```

```bash
wc -l < workspace/output/unique-servers.txt
```

Expected:

```text
4
```

`test` command là **Supplementary / Beyond explicit syllabus**, nhưng đây là kiểu validation rất quan trọng trong operational scripting.

---

# 9. Failure Injection & Troubleshooting

Đây là phần biến việc “biết command” thành kỹ năng administrator.

Mental workflow chuẩn của project:

```text
Symptom
→ expected behavior
→ recent change
→ scope
→ failing layer
→ evidence
→ hypotheses
→ safest test
→ root cause
→ remediation
→ validation
→ prevention/escalation
```

---

## Scenario 1 — `No such file or directory`

Chạy cố ý:

```bash
cat workspace/input/ap.log
```

Expected:

```text
No such file or directory
```

Không sửa ngay bằng phỏng đoán.

Evidence:

```bash
pwd
ls -la workspace/input
```

Hypothesis:

```text
filename typo
```

Bạn sẽ thấy:

```text
app.log
```

không phải:

```text
ap.log
```

Root cause:

```text
incorrect pathname
```

Remediation:

```bash
cat workspace/input/app.log
```

---

# Scenario 2 — Relative path đúng ở chỗ này nhưng sai ở chỗ khác

Đang ở:

```text
~/labs/day2
```

thì:

```bash
cat workspace/input/app.log
```

đúng.

Chuyển:

```bash
cd /tmp
```

rồi chạy lại:

```bash
cat workspace/input/app.log
```

sẽ fail.

Evidence:

```bash
pwd
```

Root cause:

```text
relative path resolved from wrong cwd
```

Remediation:

```bash
cat ~/labs/day2/workspace/input/app.log
```

hoặc:

```bash
cd ~/labs/day2
```

Đây là một trong những lỗi Linux fresher phổ biến nhất.

---

# Scenario 3 — Copy directory thất bại

Thử:

```bash
cp workspace/input workspace/backup/input-copy
```

Thông thường sẽ báo đại ý:

```text
-r not specified; omitting directory...
```

Evidence:

```bash
file workspace/input
```

Root cause:

```text
source is a directory
```

Remediation:

```bash
cp -R workspace/input workspace/backup/input-copy
```

Verification:

```bash
ls -la workspace/backup/input-copy
```

---

# Scenario 4 — `rmdir` thất bại

Tạo:

```bash
mkdir workspace/output/not-empty
touch workspace/output/not-empty/file.txt
```

Thử:

```bash
rmdir workspace/output/not-empty
```

Expected:

```text
Directory not empty
```

**Không phản xạ bằng `rm -rf` ngay.**

Inspect:

```bash
ls -la workspace/output/not-empty
```

Nếu file thật sự chỉ là lab data:

```bash
rm workspace/output/not-empty/file.txt
rmdir workspace/output/not-empty
```

Đây là workflow an toàn hơn.

---

# Scenario 5 — `uniq` không loại duplicate như mong muốn

Tạo:

```bash
printf '%s\n' \
A \
B \
A \
> /tmp/day2-uniq-demo.txt
```

Chạy:

```bash
uniq /tmp/day2-uniq-demo.txt
```

Output vẫn:

```text
A
B
A
```

Hypothesis:

```text
uniq chỉ loại adjacent duplicates
```

Test:

```bash
sort /tmp/day2-uniq-demo.txt | uniq
```

Output:

```text
A
B
```

Root cause không phải `uniq` lỗi.

Root cause là:

```text
wrong assumption about uniq semantics
```

---

# Scenario 6 — Accidentally truncated file

Đây là failure cực kỳ quan trọng.

Giả sử:

```bash
printf '%s\n' one two three > demo.txt
```

Sau đó người vận hành muốn append nhưng chạy:

```bash
printf '%s\n' four > demo.txt
```

Kiểm tra:

```bash
cat demo.txt
```

Bạn chỉ còn:

```text
four
```

Root cause:

```text
used > instead of >>
```

Trong production, recovery phụ thuộc backup/copy/versioning/application recovery capability. Shell không có universal “undo”.

Đây chính là lý do:

```text
inspection + backup + command review
```

quan trọng.

---

# 10. Command Safety Drill

Trước command:

```bash
rm -r some-directory
```

hãy tập tư duy:

```text
1. pwd
2. ls -ld -- some-directory
3. ls -la -- some-directory
4. Confirm target
5. Confirm data disposable/backup available
6. Use least destructive command
7. Remove
8. Verify
```

Không cần memorize như ritual máy móc; phải hiểu mục đích từng bước.

---

# 11. Mistakes / Anti-Patterns

Các lỗi quan trọng nhất cần tránh hôm nay là:

| Anti-pattern                             | Tại sao nguy hiểm          | Thay thế                       |
| ---------------------------------------- | -------------------------- | ------------------------------ |
| Chạy command mà không biết `pwd`         | thao tác nhầm path         | `pwd` trước destructive action |
| `sudo` mọi command                       | tăng blast radius          | dùng normal user               |
| `rm -rf` theo thói quen                  | xóa recursive không safety | inspect → minimal deletion     |
| Không quote filename                     | word splitting/globbing    | `"$path"`                      |
| Dùng `>` khi muốn append                 | truncate file              | `>>`                           |
| Nghĩ `uniq` deduplicate toàn file        | logic sai                  | `sort \| uniq`                 |
| Nghĩ Linux không case-sensitive          | wrong file                 | verify exact names             |
| Dùng relative path nhưng không biết cwd  | sai target                 | `pwd` hoặc absolute path       |
| Thấy error rồi thử command ngẫu nhiên    | phá evidence               | inspect first                  |
| Copy/move mà không check destination     | overwrite                  | inspect destination            |
| Học thuộc options nhưng không dùng `man` | dễ sai/version-dependent   | tra documentation              |

---

# 12. Security & Least Privilege

Day 3 mới học sâu về users/groups/permissions/sudo/SSH, nhưng Day 2 cần hình thành một nguyên tắc:

> **Không dùng privilege cao hơn mức cần thiết.**

Nếu:

```bash
mkdir test
```

làm được thì không dùng:

```bash
sudo mkdir test
```

Nếu:

```bash
cat file
```

làm được thì không dùng:

```bash
sudo cat file
```

Tại sao?

Bởi vì một typo với normal user thường bị giới hạn bởi permissions.

Một typo bằng root có thể trở thành outage.

Ví dụ:

```text
normal user mistake → lab directory damaged
root mistake        → system directory damaged
```

Đây chính là **blast-radius control**.

---

# 13. Operations / Recovery / Rollback

File operations không có universal transaction hoặc Undo.

### `cp`

Rollback thường đơn giản nếu destination mới:

```bash
rm copied-file
```

nhưng chỉ khi bạn chắc đó chính xác là copy mới.

Nếu overwrite existing file:

```text
rollback requires backup/versioned copy/etc.
```

### `mv`

Nếu chỉ rename:

```bash
mv new-name old-name
```

có thể rollback.

Nhưng nếu overwrite destination thì recovery khó hơn.

### `rm`

Không nên coi recovery là guaranteed.

Operationally:

```text
rm = destructive state change
```

nên prevention tốt hơn recovery.

### `>`

Có thể truncate file tức thì.

Do đó với config quan trọng sau này:

```text
inspect
→ backup
→ edit
→ validate
→ deploy/restart
→ verify
→ rollback if needed
```

Đây là tư duy bạn sẽ dùng xuyên suốt Java/Tomcat/PostgreSQL.

---

# 14. L1 vs Escalation

Ở Day 2, L1-safe action thường gồm:

```text
inspect path
inspect file existence
read logs/text
verify current directory
correct obvious typo
retry safe read-only commands
work inside authorized lab directory
```

Bạn nên dừng và escalate khi:

```text
target nằm ngoài phạm vi được giao
cần root mà runbook không cho phép
file quan trọng đã bị delete/overwritten
system configuration có nguy cơ hỏng
không biết blast radius
recovery procedure không rõ
```

Một operator giỏi không phải người “sửa tất cả”.

Một operator giỏi biết:

```text
what I know
what I changed
what evidence I have
what is safe for L1
when to escalate
```

---

# 15. Exam Focus

Các dạng câu hỏi lý thuyết rất có khả năng phù hợp với syllabus Day 2 gồm:

**Linux vs Linux distribution:** Linux kernel khác Ubuntu/RHEL ở điểm nào?

**Terminal vs shell:** terminal có phải Bash không?

**Filesystem:** `/` và `/root` khác nhau thế nào?

**Paths:** absolute và relative path khác nhau thế nào?

**Navigation:** `.` `..` `~` nghĩa là gì?

**Hidden file:** vì sao `.bashrc` không xuất hiện với `ls` mặc định?

**File manipulation:** `cp`, `mv`, `rm`, `rmdir` khác nhau thế nào?

**Text:** `head`, `tail`, `grep`, `sort`, `uniq`, `wc`.

**Pipeline:** `|` làm gì?

**Redirection:** khác biệt giữa `>` và `>>`.

**Help:** khi nào dùng `man`, `--help`, `help`, `apropos`.

**Safety:** vì sao `rm -rf` nguy hiểm?

Practical exam dạng Day 2 có thể đưa bạn vào một directory tree và yêu cầu:

```text
navigate
find correct path
inspect data
copy
rename
filter text
save output
delete temporary artifacts safely
prove final state
```

---

# 16. Knowledge Check

Hãy tự trả lời trước khi xem đáp án.

1. Linux và Ubuntu có phải cùng một khái niệm không?
2. Kernel nằm giữa application và hardware như thế nào?
3. `cd` tại sao là shell builtin?
4. `/` khác `/root` thế nào?
5. Path nào là absolute: `etc/hosts` hay `/etc/hosts`?
6. `..` có nghĩa gì?
7. Vì sao `ls` không hiện `.bashrc` mặc định?
8. `touch file.txt` có xóa nội dung file cũ không?
9. `cp fileA fileB` có nguy cơ gì nếu `fileB` đã tồn tại?
10. `rmdir` khác `rm -r` thế nào?
11. Tại sao `"monthly report.txt"` cần quote?
12. `*.log` thường được ai expand?
13. `>` khác `>>` thế nào?
14. File descriptor của stderr là số mấy?
15. `|` làm gì?
16. `uniq` có remove mọi duplicate bất kể vị trí không?
17. Tại sao thường cần `sort | uniq`?
18. Dùng gì để tra help của `cd`?
19. Dùng gì để xác định `ls` là alias/builtin/executable?
20. Trước `rm -r`, hai command đầu tiên bạn nên nghĩ tới là gì?

### Đáp án ngắn

```text
1. Không. Linux là kernel; Ubuntu là distribution.
2. Application dùng OS/kernel interfaces để truy cập tài nguyên.
3. Nó phải thay đổi cwd của shell hiện tại.
4. / là filesystem root; /root là home của root.
5. /etc/hosts.
6. Parent directory.
7. Tên bắt đầu bằng . bị ẩn khỏi listing mặc định.
8. Không; touch chủ yếu create/update timestamp.
9. Có thể overwrite fileB.
10. rmdir chỉ empty dir; rm -r recursive.
11. Space nếu không quote sẽ phân tách arguments.
12. Shell.
13. > truncate; >> append.
14. 2.
15. Nối stdout command trước vào stdin command sau.
16. Không.
17. Để duplicate trở thành adjacent.
18. help cd.
19. type ls.
20. pwd và ls/ls -la target.
```

---

# 17. Independent Practical Challenge

Sau khi hoàn thành Guided Lab, hãy làm bài này **không nhìn lại command mẫu**.

Bạn đang ở:

```text
~/labs/day2-challenge
```

Yêu cầu tạo:

```text
day2-challenge/
├── raw/
├── processed/
├── backup/
└── evidence/
```

Trong `raw/access.log`, tạo dữ liệu:

```text
200 /health
500 /api/orders
200 /login
404 /missing
500 /api/orders
503 /database
200 /health
```

Nhiệm vụ:

1. Xác nhận chính xác current working directory.
2. Tạo directory structure.
3. Tạo `access.log`.
4. Xác nhận file có đúng 7 lines.
5. Tạo `processed/errors.txt` chỉ chứa `500` và `503`.
6. Không lấy dòng `404`.
7. Chứng minh errors file có 3 lines.
8. Tạo danh sách unique status codes được sort.
9. Copy danh sách đó vào `backup/`.
10. Rename backup thành `status-codes.baseline`.
11. Tạo một hidden file `.completed`.
12. Tạo file có tên `Day 2 Notes.txt`.
13. Copy nó mà không gặp lỗi do spaces.
14. Tạo một temporary file rồi xóa an toàn.
15. Dùng help system để xác định option interactive của `rm`.
16. Ghi command/session evidence vào `evidence/`.
17. Không dùng `sudo`.
18. Không dùng `rm -rf`.
19. Cuối bài chứng minh toàn bộ required files tồn tại.
20. Giải thích bằng lời absolute path và relative path đã được dùng ở đâu.

Đây mới là bài kiểm tra xem bạn **thực sự nắm Day 2** hay chỉ đọc hiểu.

---

# 18. Mastery Check

Day 2 không nên được đánh dấu “mastered” chỉ vì bạn đã đọc bài này.

Mức **Theory Ready** khi bạn tự giải thích được:

```text
Linux architecture
distribution
terminal/shell
filesystem tree
absolute/relative paths
stdin/stdout/stderr
pipe/redirection
basic text commands
command safety
```

Mức **Practical Ready** khi không nhìn hướng dẫn bạn có thể:

```text
navigate
create
inspect
copy
rename
filter
sort
deduplicate
redirect
delete safely
validate
produce evidence
```

Mức **Operational Ready** khi trước một destructive command bạn tự nhiên nghĩ:

```text
pwd
→ inspect
→ understand target
→ understand blast radius
→ perform minimum change
→ verify
→ preserve evidence
```

---

## Day 2 — điều quan trọng nhất cần mang sang Day 3

Nếu chỉ nhớ một mental model hôm nay, hãy nhớ:

```text
KNOW WHERE YOU ARE
        ↓
KNOW WHAT THE COMMAND WILL TOUCH
        ↓
INSPECT BEFORE MODIFYING
        ↓
USE THE LEAST DESTRUCTIVE ACTION
        ↓
VERIFY THE RESULT
        ↓
KEEP EVIDENCE
```

Đó không chỉ là cách dùng Linux CLI. Đó là cách một **administrator/operator** phải làm việc.

Day 2 theo syllabus yêu cầu command-line navigation, file manipulation và command log; phần Guided Lab ở trên bao phủ đầy đủ yêu cầu đó. Tuy nhiên, tôi **chưa coi Day 2 của bạn là hoàn thành** cho đến khi bạn thực sự chạy lab hoặc chứng minh được practical reasoning/evidence. 00_MASTER_SYLLABUS_Java_Fresher…

Tài liệu hiện hành cũng xác nhận các nguyên tắc kỹ thuật chính chúng ta dùng hôm nay: filesystem bắt đầu từ `/`; pathname bắt đầu bằng `/` được resolve từ root; `ls` mặc định không liệt kê dotfiles; `cp` hỗ trợ recursive directory copy; `rm` là destructive file removal; Bash pipeline nối output của command trước với input của command sau; và `>` có thể truncate file hiện có. [man7.org](https://man7.org/linux/man-pages/man7/hier.7.html?utm_source=chatgpt.com)
