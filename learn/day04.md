# Day 4 — Linux Processes, Signals, systemd Services, Packages, Environment Variables & Service Startup

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 4 thuộc phần **Linux Fundamentals** và có phạm vi bắt buộc chính xác như sau:

**Concept/Lecture:** `Processes`, `signals`, `systemd services`, `packages`, `environment variables`, và `service startup`.

**Assignment/Lab:** cài đặt một package, tạo và quản lý một `systemd service`, inspect processes, và khôi phục một service bị lỗi. 00_MASTER_SYLLABUS_Java_Fresher…

Đây là một ngày cực kỳ quan trọng vì Day 4 tạo nền móng trực tiếp cho Tomcat/Java ở Day 12–14, PostgreSQL Day 15–16, monitoring Day 17 và troubleshooting tích hợp Day 20. Sau hôm nay, mục tiêu không chỉ là biết chạy `systemctl restart`; bạn cần hiểu **process nào đang chạy, systemd khởi động nó như thế nào, environment ở đâu ra, signal nào đang được gửi, tại sao service fail, và bằng chứng nào chứng minh đã recovery thành công**.

### Môi trường lab mình sẽ dùng

Phần thực hành chính giả định:

```text
Linux: Ubuntu 24.04 LTS / Debian 12+ hoặc tương đương
Init/service manager: systemd
Shell: bash
Quyền: user thông thường có sudo
```

Với RHEL 9/Fedora, phần `systemd`, `ps`, `kill` gần như giống nhau; package manager đổi từ APT sang DNF.

---

# 2. Mục tiêu Day 4

Sau Day 4, bạn cần tự mình giải thích và thực hiện được toàn bộ chuỗi:

```text
package
   ↓
executable / script
   ↓
systemd unit
   ↓
service startup
   ↓
process
   ↓
PID / PPID / UID / environment
   ↓
signals
   ↓
exit / failure
   ↓
systemd detects state
   ↓
diagnose
   ↓
recover
   ↓
validate
```

Quan trọng hơn, bạn cần phân biệt được những cặp rất hay bị nhầm:

| Khái niệm           | Không phải                    |
| ------------------- | ----------------------------- |
| Program             | Process                       |
| Process             | Service                       |
| `systemctl start`   | `systemctl enable`            |
| `systemctl restart` | `systemctl reload`            |
| `systemctl reload`  | `systemctl daemon-reload`     |
| Shell variable      | Exported environment variable |
| `SIGTERM`           | `SIGKILL`                     |
| `inactive`          | `failed`                      |
| Package             | Repository                    |
| `apt update`        | `apt upgrade`                 |

---

# 3. Mental Model tổng thể

Hãy tưởng tượng bạn có một Java application trong tương lai.

File JAR nằm trên disk chỉ là **program/file**:

```text
/opt/myapp/myapp.jar
```

Khi chạy:

```bash
java -jar /opt/myapp/myapp.jar
```

kernel tạo một **process**.

Process có các thuộc tính như:

```text
PID
PPID
UID/GID
memory
CPU usage
open files
working directory
environment variables
process state
exit status
```

Nhưng production không muốn admin SSH vào rồi tự chạy Java bằng tay.

Ta muốn:

```text
system boot
    ↓
systemd PID 1
    ↓
myapp.service
    ↓
java process
```

`systemd` trở thành supervisor quản lý lifecycle của process đó.

Nó có thể:

```text
start
stop
restart
monitor
send signals
run service as a specific user
provide environment variables
restart after failure
start service at boot
collect service state
```

Đó chính là nền tảng mà sau này bạn sẽ dùng với:

```text
tomcat.service
postgresql.service
node_exporter.service
prometheus.service
grafana-server.service
```

---

# 4. Processes — nền tảng đầu tiên

## 4.1 Program khác Process như thế nào?

**Program** là code/file nằm trên storage.

Ví dụ:

```bash
/usr/bin/sleep
/usr/bin/java
/usr/bin/bash
```

**Process** là một instance đang thực thi của program.

Ví dụ:

```bash
sleep 600
```

Nếu chạy ba lần:

```bash
sleep 600 &
sleep 600 &
sleep 600 &
```

bạn có một executable `/usr/bin/sleep`, nhưng ba process khác nhau với ba PID khác nhau.

---

# 5. Những thuộc tính quan trọng của Process

| Thuộc tính  | Ý nghĩa                       |
| ----------- | ----------------------------- |
| PID         | Process ID                    |
| PPID        | Parent Process ID             |
| UID/GID     | User/group thực thi process   |
| CMD/COMMAND | Program/command               |
| CWD         | Current working directory     |
| ENV         | Environment                   |
| FD          | File descriptors              |
| CPU         | CPU consumption               |
| RSS         | Physical memory đang resident |
| STAT        | Process state                 |
| Exit code   | Kết quả khi process kết thúc  |

`ps` hỗ trợ hiển thị PID, PPID, UID và rất nhiều thuộc tính process khác. [man7](https://man7.org/linux/man-pages/man1/ps.1.html?utm_source=chatgpt.com)

Ví dụ:

```bash
ps -ef
```

Hoặc dạng thường hữu ích hơn cho troubleshooting:

```bash
ps -eo pid,ppid,user,stat,lstart,etime,%cpu,%mem,cmd
```

Trong đó:

```text
PID      process ID
PPID     parent PID
USER     process owner
STAT     state
LSTART   start time
ETIME    elapsed runtime
%CPU     CPU usage
%MEM     memory percentage
CMD      command line
```

---

# 6. Parent và Child Process

Một process có thể sinh process khác.

Mental model đơn giản:

```text
bash
 PID 3000
    |
    +--- sleep
         PID 3050
         PPID 3000
```

Thử:

```bash
sleep 600 &
PID=$!

echo "$PID"
ps -o pid,ppid,user,stat,cmd -p "$PID"
```

Bạn có thể kiểm tra shell:

```bash
echo $$
```

Sau đó so với PPID của `sleep`.

---

# 7. Process lifecycle: fork → exec → exit → wait

Ở mức fresher, hãy ghi nhớ model:

```text
parent process
      |
     fork()
      |
      +-------------------+
      |                   |
   parent               child
                          |
                        exec()
                          |
                    new program
                          |
                        exit()
                          |
                      exit status
                          |
                       wait()
                          |
                    parent reaps
```

Không phải mọi chương trình đều triển khai chính xác theo cùng một cách ở application level, nhưng đây là model Linux/Unix cực kỳ quan trọng để hiểu process tree.

---

# 8. PID 1 và systemd

Kiểm tra:

```bash
ps -p 1 -o pid,ppid,user,comm,args
```

Trên hệ thống systemd thông thường bạn sẽ thấy dạng:

```text
PID PPID USER COMMAND ARGS
1   0    root systemd /sbin/init
```

PID `1` đặc biệt.

Trong hệ thống đang dùng systemd:

```text
systemd = system manager = PID 1
```

Systemd quản lý service và process theo unit/cgroup thay vì chỉ dựa vào một PID file theo kiểu daemon truyền thống.

---

# 9. Process State

Một số state rất quan trọng:

| STAT | Ý nghĩa               |
| ---- | --------------------- |
| `R`  | Running / runnable    |
| `S`  | Interruptible sleep   |
| `D`  | Uninterruptible sleep |
| `T`  | Stopped/traced        |
| `Z`  | Zombie                |

Ví dụ:

```bash
ps -eo pid,ppid,user,stat,cmd
```

### `R`

Process đang chạy hoặc sẵn sàng chạy.

### `S`

Đây không có nghĩa là process hỏng.

Nó thường chỉ đang chờ:

```text
timer
network
input
event
child process
```

Rất nhiều daemon bình thường nằm ở trạng thái `S` phần lớn thời gian.

### `T`

Process đã bị stopped.

Bạn có thể tạo trạng thái này bằng:

```bash
kill -STOP PID
```

sau đó:

```bash
kill -CONT PID
```

### `D`

Thường là process đang chờ I/O/kernel operation không interruptible.

Một sai lầm fresher:

```text
Process D → kill -9 sẽ luôn giải quyết.
```

Không đúng.

Ngay cả `SIGKILL` cũng có thể chưa được xử lý cho tới khi kernel operation kết thúc.

Nếu production process kẹt lâu ở `D`, cần điều tra storage/NFS/device/kernel thay vì spam `kill -9`.

### `Z` — Zombie

Process đã exit nhưng parent chưa `wait()` để thu exit status.

Zombie không phải process đang thực thi bình thường nữa.

Do đó:

```bash
kill -9 <zombie>
```

không phải cách xử lý root cause.

Cần xem parent:

```bash
ps -o pid,ppid,stat,cmd -p <zombie-pid>
```

và điều tra tại sao parent không reap child.

---

# 10. Inspect một Process

Các lệnh quan trọng:

```bash
ps -ef
```

```bash
ps aux
```

```bash
ps -o pid,ppid,user,group,stat,etime,%cpu,%mem,cmd -p PID
```

```bash
pgrep process-name
```

```bash
pgrep -a process-name
```

```bash
pstree -p
```

Một cách rất hữu ích:

```bash
cat /proc/PID/status
```

Ví dụ:

```bash
cat /proc/1234/status
```

Process filesystem `/proc` còn cho bạn:

```text
/proc/PID/cmdline
/proc/PID/environ
/proc/PID/cwd
/proc/PID/fd/
/proc/PID/status
```

Ví dụ:

```bash
readlink /proc/1234/cwd
```

```bash
tr '\0' ' ' < /proc/1234/cmdline
echo
```

```bash
sudo tr '\0' '\n' < /proc/1234/environ
```

Đây sẽ cực kỳ hữu ích khi troubleshooting Java/Tomcat sau này.

---

# 11. Signals

## 11.1 Signal là gì?

Signal là cơ chế kernel dùng để thông báo một event tới process.

Một signal có thể yêu cầu process:

```text
terminate
stop
continue
reload
handle some application-defined event
```

Nhưng hành vi chính xác phụ thuộc signal và chương trình.

---

# 12. Các signal fresher bắt buộc phải biết

| Signal    | Linux number thường gặp | Ý nghĩa phổ biến                           |
| --------- | ----------------------: | ------------------------------------------ |
| `SIGHUP`  |                       1 | Hangup; nhiều daemon dùng để reload config |
| `SIGINT`  |                       2 | Interactive interrupt, thường từ Ctrl+C    |
| `SIGKILL` |                       9 | Kernel buộc terminate                      |
| `SIGTERM` |                      15 | Yêu cầu terminate gracefully               |
| `SIGSTOP` |                      19 | Stop process                               |
| `SIGCONT` |                      18 | Continue stopped process                   |

Trên Linux, `SIGKILL` và `SIGSTOP` không thể bị process catch, block hoặc ignore. [man7](https://www.man7.org/linux/man-pages/man7/signal.7.html?utm_source=chatgpt.com)

Quan trọng nhất trong operations:

```text
SIGTERM trước
SIGKILL cuối cùng
```

`kill` mặc định gửi `SIGTERM`. Tài liệu hiện hành cũng khuyến nghị ưu tiên TERM thay cho KILL vì application có cơ hội cleanup trước khi exit; `SIGKILL` không thể được catch và không cho process cơ hội cleanup. [man7](https://man7.org/linux/man-pages/man1/kill.1.html?utm_source=chatgpt.com)

---

# 13. Vì sao `kill` là tên dễ gây hiểu lầm?

Lệnh:

```bash
kill PID
```

không có nghĩa:

> “Linux lập tức giết process”.

Thực tế:

```bash
kill PID
```

tương đương:

```bash
kill -TERM PID
```

Tức là gửi signal `SIGTERM`.

Explicit hơn:

```bash
kill -TERM PID
```

hoặc:

```bash
kill -SIGTERM PID
```

---

# 14. Graceful termination

Một application tốt có thể xử lý `SIGTERM` như sau:

```text
SIGTERM
   ↓
stop receiving new work
   ↓
finish current transaction
   ↓
flush buffers
   ↓
close DB connections
   ↓
close files/sockets
   ↓
exit
```

Đây là lý do production operations không nên nhảy ngay tới:

```bash
kill -9 PID
```

---

# 15. SIGKILL

```bash
kill -KILL PID
```

hoặc:

```bash
kill -9 PID
```

Kernel terminate process mà không cho application chạy shutdown handler.

Rủi ro tùy application:

```text
incomplete transaction
temporary files remain
unfinished write
corrupted application-level state
lost diagnostic evidence
```

`SIGKILL` không phải “phiên bản mạnh hơn và tốt hơn” của `SIGTERM`.

Nó là **last resort**.

---

# 16. SIGHUP không mặc định có nghĩa là reload

Điểm rất hay bị hỏi.

Nhiều daemon Unix dùng:

```bash
kill -HUP PID
```

để reload config.

Nhưng Linux không quy định:

```text
SIGHUP = reload configuration
```

Application phải implement hành vi đó.

Nếu application không implement, default action của `SIGHUP` có thể terminate process. [man7](https://www.man7.org/linux/man-pages/man7/signal.7.html?utm_source=chatgpt.com)

Do đó trước production:

```text
consult application documentation
```

Không đoán.

---

# 17. `kill -0`

Một trick rất hữu ích:

```bash
kill -0 PID
```

Signal `0` không thực sự gửi signal terminate; nó có thể dùng để kiểm tra process existence và permission để signal process. [man7](https://man7.org/linux/man-pages/man1/kill.1.html?utm_source=chatgpt.com)

Ví dụ:

```bash
kill -0 1234
echo $?
```

Nếu thành công:

```text
0
```

---

# 18. LAB nhỏ — Process + Signal

**Chỉ thực hành trên process do chính bạn tạo. Không lấy PID production/ngẫu nhiên để thử.**

Khởi chạy:

```bash
sleep 600 &
PID=$!

echo "PID=$PID"
```

Inspect:

```bash
ps -o pid,ppid,user,stat,etime,cmd -p "$PID"
```

Check:

```bash
kill -0 "$PID"
echo $?
```

Stop:

```bash
kill -STOP "$PID"
```

Inspect:

```bash
ps -o pid,ppid,stat,cmd -p "$PID"
```

Expected invariant:

```text
STAT chứa T
```

Continue:

```bash
kill -CONT "$PID"
```

Kiểm tra:

```bash
ps -o pid,ppid,stat,cmd -p "$PID"
```

Terminate gracefully:

```bash
kill -TERM "$PID"
```

Sau đó:

```bash
wait "$PID"
echo $?
```

Bạn có thể thấy exit status như:

```text
143
```

đây là **example output**, không nên học thuộc một cách máy móc.

---

# 19. Environment Variables

## 19.1 Environment là gì?

Mỗi process có một environment.

Về bản chất nó chứa các cặp dạng:

```text
NAME=value
```

Ví dụ:

```text
PATH=/usr/local/bin:/usr/bin
HOME=/home/student
LANG=en_US.UTF-8
APP_ENV=production
JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```

Khi một child được tạo qua `fork()`, nó nhận copy environment của parent; khi program mới được thực thi qua `execve()`, environment được truyền cho program đó. [man7](https://www.man7.org/linux/man-pages/man7/environ.7.html?utm_source=chatgpt.com)

Mental model:

```text
bash
environment:
APP_ENV=prod
    |
    | fork/exec
    v
java process
environment:
APP_ENV=prod
```

---

# 20. Shell Variable khác Environment Variable

Chạy:

```bash
DAY4_TEST=hello
```

Kiểm tra shell hiện tại:

```bash
echo "$DAY4_TEST"
```

Có:

```text
hello
```

Nhưng:

```bash
bash -c 'echo "${DAY4_TEST-unset}"'
```

có thể cho:

```text
unset
```

Bởi vì biến chưa được exported.

Bây giờ:

```bash
export DAY4_TEST
```

Sau đó:

```bash
bash -c 'echo "$DAY4_TEST"'
```

child process sẽ thấy:

```text
hello
```

---

# 21. Những command Environment quan trọng

```bash
env
```

```bash
printenv
```

```bash
printenv PATH
```

```bash
export VAR=value
```

```bash
unset VAR
```

Bạn cũng có thể chỉ đặt environment cho một command:

```bash
APP_ENV=test ./application
```

Ví dụ:

```bash
DAY4_MODE=temporary bash -c 'echo "$DAY4_MODE"'
```

Sau command đó:

```bash
echo "${DAY4_MODE-unset}"
```

sẽ vẫn là:

```text
unset
```

---

# 22. Sai lầm cực kỳ quan trọng: systemd không đơn giản kế thừa shell environment của bạn

Bạn SSH vào server:

```bash
export APP_ENV=production
```

rồi:

```bash
sudo systemctl restart myapp
```

Không được giả định rằng:

```text
myapp.service sẽ có APP_ENV=production
```

System services do system manager quản lý không đơn giản kế thừa environment của interactive shell của bạn. Systemd có cơ chế riêng như:

```ini
Environment=
EnvironmentFile=
```

Systemd documentation mô tả rõ system services và user services có quy tắc environment riêng; `Environment=` và `EnvironmentFile=` là cơ chế chuẩn để cung cấp environment cho service. [manpages.debian.org](https://manpages.debian.org/trixie/systemd/systemd.exec.5.en.html?utm_source=chatgpt.com)

Đây là kiến thức cực kỳ quan trọng cho **Day 13 — externalized Java configuration**.

---

# 23. Environment trong systemd

Ví dụ trực tiếp trong unit:

```ini
[Service]
Environment="APP_ENV=production"
Environment="JAVA_OPTS=-Xms256m -Xmx512m"
```

Hoặc:

```ini
EnvironmentFile=/etc/myapp/myapp.env
```

File:

```text
APP_ENV=production
APP_PORT=8080
```

---

# 24. Thay đổi EnvironmentFile có cần `daemon-reload` không?

Đây là điểm rất đáng học.

Giả sử unit vẫn là:

```ini
EnvironmentFile=/etc/day4-demo.env
```

và bạn chỉ sửa nội dung:

```text
/etc/day4-demo.env
```

Bạn **không cần `systemctl daemon-reload` chỉ vì giá trị environment file thay đổi**.

Nhưng process đang chạy không tự nhận environment mới.

Bạn cần:

```bash
sudo systemctl restart day4-demo
```

Systemd đọc `EnvironmentFile=` trước khi khởi chạy process mới. [GitHub](https://github.com/systemd/systemd/blob/main/man/systemd.exec.xml?utm_source=chatgpt.com)

Ngược lại, nếu bạn sửa:

```text
/etc/systemd/system/day4-demo.service
```

thì cần:

```bash
sudo systemctl daemon-reload
```

---

# 25. Security: Environment Variable không phải secret store

Điểm này cực kỳ quan trọng cho production.

Không nên coi:

```text
DB_PASSWORD
API_TOKEN
PRIVATE_KEY
```

trong environment của systemd như một secret storage an toàn.

Tài liệu systemd hiện hành cảnh báo environment variables không thích hợp để truyền password/key material vì chúng có thể bị expose qua nhiều cơ chế và truyền xuống process tree; systemd cung cấp credential mechanisms cho các use case cần bảo vệ dữ liệu nhạy cảm. [GitHub](https://github.com/systemd/systemd/blob/main/man/systemd.exec.xml?utm_source=chatgpt.com)

Phần secrets sẽ quay lại sâu hơn ở **Day 14**.

---

# 26. Package Management

## 26.1 Package là gì?

Package không chỉ là executable.

Ví dụ một `.deb` hoặc `.rpm` có thể chứa:

```text
executables
libraries
configuration templates
documentation
systemd units
users/groups
metadata
dependencies
install/remove scripts
```

Mental model:

```text
Repository
   |
   | metadata
   v
Package manager
   |
   | dependency resolution
   v
Package
   |
   +-- binaries
   +-- libraries
   +-- configs
   +-- service units
```

---

# 27. Debian/Ubuntu package stack

Bạn thường gặp:

```text
APT
 ↓
dpkg
```

APT giải quyết repository/dependencies ở high level.

`dpkg` quản lý Debian package database và `.deb` ở lower level.

Debian hiện khuyến nghị `apt` cho interactive command-line work và `apt-get` cho scripting/automation. [Debian](https://www.debian.org/doc/manuals/debian-reference/ch02.en.html?utm_source=chatgpt.com)

---

# 28. `apt update` khác `apt upgrade`

```bash
sudo apt update
```

không đồng nghĩa với update toàn bộ software.

Nó refresh package repository metadata.

Simplified flow:

```text
repository
   ↓
Packages metadata
   ↓
apt local metadata
```

Trong khi:

```bash
sudo apt upgrade
```

thực sự có thể thay đổi installed packages.

Debian mô tả `apt update` là lấy archive metadata và cập nhật local package metadata; `apt install` sau đó resolve dependencies, fetch, unpack và chạy package lifecycle scripts. [Debian](https://www.debian.org/doc/manuals/debian-reference/ch02.en.html?utm_source=chatgpt.com)

---

# 29. Các APT command cần biết

```bash
sudo apt update
```

Tìm package:

```bash
apt search tree
```

Inspect:

```bash
apt show tree
```

Candidate version:

```bash
apt-cache policy tree
```

Install:

```bash
sudo apt install tree
```

Check:

```bash
dpkg -s tree
```

hoặc:

```bash
dpkg-query -W tree
```

Remove:

```bash
sudo apt remove tree
```

Purge:

```bash
sudo apt purge tree
```

Debian phân biệt `remove` và `purge`: remove giữ lại một số configuration files của package, còn purge loại bỏ cả package-managed configuration files. [Debian](https://www.debian.org/doc/manuals/debian-reference/ch02.en.html?utm_source=chatgpt.com)

---

# 30. Cảnh báo Package Operations

Trong production, không chạy mù quáng:

```bash
sudo apt upgrade -y
```

hoặc:

```bash
sudo apt autoremove -y
```

Luôn xem transaction.

Package operation có thể:

```text
install dependencies
upgrade libraries
remove dependencies
restart services
replace files
change behavior
```

Đặc biệt đừng coi package rollback đơn giản là:

```bash
apt remove package
```

Remove ≠ rollback.

Rollback có thể cần:

```text
previous package version
repository availability
dependency compatibility
config rollback
service restart
application validation
```

---

# 31. RHEL/Fedora equivalent

Với RHEL 9:

```bash
sudo dnf install tree
```

Query:

```bash
rpm -q tree
```

Remove:

```bash
sudo dnf remove tree
```

DNF tự resolve dependencies khi cài package. Red Hat cũng cảnh báo cần kiểm tra transaction remove cẩn thận vì dependency removal có thể ảnh hưởng hệ thống. [Red Hat Documentation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_software_with_the_dnf_tool/assembly_installing-rhel-9-content_managing-software-with-the-dnf-tool?utm_source=chatgpt.com)

---

# 32. systemd — phần trung tâm Day 4

`systemd` là system/service manager phổ biến trên Linux hiện đại.

Đối với service:

```text
systemd
 PID 1
   |
   +-- ssh.service
   |
   +-- cron.service
   |
   +-- postgres.service
   |
   +-- tomcat.service
```

Một file có suffix:

```text
.service
```

mô tả một process/service mà systemd quản lý. [man7](https://www.man7.org/linux/man-pages/man5/systemd.service.5.html?utm_source=chatgpt.com)

---

# 33. Unit là gì?

Systemd quản lý nhiều loại **unit**.

Ví dụ:

```text
.service
.socket
.target
.timer
.mount
.path
.device
.slice
.scope
```

Day 4 tập trung chủ yếu:

```text
.service
.target
```

---

# 34. Unit File Locations

Vị trí thường gặp:

```text
/etc/systemd/system/
/run/systemd/system/
/usr/lib/systemd/system/
```

Trên một số Debian/Ubuntu bạn cũng có thể thấy:

```text
/lib/systemd/system/
```

Đừng sửa trực tiếp vendor unit trong `/usr/lib/systemd/system` hoặc `/lib/systemd/system` nếu có thể tránh được.

Admin-defined unit hoặc override thường thuộc:

```text
/etc/systemd/system/
```

---

# 35. Cấu trúc systemd service

Ví dụ:

```ini
[Unit]
Description=Example Service

[Service]
Type=exec
ExecStart=/usr/local/bin/myapp
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Ba phần quan trọng:

```text
[Unit]
[Service]
[Install]
```

---

# 36. `[Unit]`

Ví dụ:

```ini
[Unit]
Description=My Application
After=network.target
```

`Description=` chỉ mô tả unit.

`After=` là **ordering relationship**.

Cực kỳ quan trọng:

```ini
After=abc.service
```

không có nghĩa:

```text
systemd tự động start abc.service
```

Để tạo dependency phải dùng:

```ini
Wants=
```

hoặc:

```ini
Requires=
```

Và dependency với ordering là hai khái niệm độc lập. systemd documentation nhấn mạnh `Wants=`/`Requires=` không tự xác định thứ tự; `After=`/`Before=` mới điều khiển ordering. [man7](https://www.man7.org/linux/man-pages/man5/systemd.unit.5.html?utm_source=chatgpt.com)

---

# 37. `Wants=` versus `Requires=`

Conceptually:

```ini
Wants=database.service
```

là dependency yếu hơn.

```ini
Requires=database.service
```

là dependency mạnh hơn.

Nhưng đừng biến mọi dependency thành:

```ini
Requires=
After=
```

một cách máy móc.

Bạn phải hiểu behavior mong muốn khi dependency fail.

---

# 38. `[Service]`

Ví dụ:

```ini
[Service]
Type=exec
User=myapp
Group=myapp
EnvironmentFile=/etc/myapp.env
ExecStart=/opt/myapp/bin/start
Restart=on-failure
RestartSec=5s
```

Đây là nơi định nghĩa cách process chạy.

---

# 39. `ExecStart=`

```ini
ExecStart=/usr/local/bin/day4-demo
```

Một lỗi rất phổ biến là tưởng `ExecStart=` luôn chạy qua shell.

Ví dụ:

```ini
ExecStart=echo hello > /tmp/output
```

không nên được hiểu giống Bash:

```bash
echo hello > /tmp/output
```

Nếu thực sự cần shell syntax, phải explicitly gọi shell:

```ini
ExecStart=/bin/sh -c 'echo hello > /tmp/output'
```

Trong production, nên tránh shell trung gian nếu không cần.

Prefer:

```ini
ExecStart=/absolute/path/to/program
```

---

# 40. `Type=`

Các Type phổ biến:

```text
simple
exec
forking
oneshot
notify
dbus
```

Đối với fresher:

`simple` và `exec` là quan trọng nhất.

Với `Type=simple`, systemd có thể coi service đã start sau khi fork main process, trước khi `execve()` actual executable hoàn tất.

Tài liệu systemd hiện tại khuyến nghị `Type=exec` cho long-running services trong nhiều trường hợp vì setup/exec errors được phản ánh chính xác hơn trong startup result. [man7](https://www.man7.org/linux/man-pages/man5/systemd.service.5.html?utm_source=chatgpt.com)

Trong lab hôm nay chúng ta dùng:

```ini
Type=exec
```

---

# 41. `Restart=on-failure`

Ví dụ:

```ini
Restart=on-failure
RestartSec=2s
```

Nếu application:

```text
exit non-zero
crash
timeout
certain abnormal termination
```

systemd có thể restart.

Systemd hiện khuyến nghị `Restart=on-failure` cho long-running services khi automatic recovery phù hợp. [man7](https://man7.org/linux/man-pages/man5/systemd.service.5%40%40systemd.html?utm_source=chatgpt.com)

Nhưng:

```text
Restart=on-failure
```

không phải phép màu.

Nếu config sai:

```text
process fails
→ restart
→ fails
→ restart
→ fails
→ restart
```

Systemd có start-rate limiting để tránh infinite restart storm.

---

# 42. `[Install]`

Ví dụ:

```ini
[Install]
WantedBy=multi-user.target
```

Đây là thông tin được `systemctl enable` dùng để hook unit vào startup dependency graph.

Mental model:

```text
multi-user.target
       |
       | wants
       v
day4-demo.service
```

---

# 43. `start` khác `enable`

Đây có thể là câu hỏi exam.

```bash
sudo systemctl start day4-demo
```

nghĩa là:

```text
start NOW
```

Nhưng chưa chắc boot sau tự start.

Trong khi:

```bash
sudo systemctl enable day4-demo
```

nghĩa là:

```text
configure activation according to [Install],
thường để service được pull vào boot dependency graph
```

nhưng **không nhất thiết start ngay**.

Systemd documentation mô tả rõ enable và start là orthogonal: unit có thể enabled nhưng chưa started hoặc started nhưng disabled. `enable` tạo symlink theo `[Install]`, trong khi `start` thực sự activate unit. [man7](https://man7.org/linux/man-pages/man1/systemctl.1.html?utm_source=chatgpt.com)

Muốn cả hai:

```bash
sudo systemctl enable --now day4-demo
```

---

# 44. Các state systemd quan trọng

```bash
systemctl status ssh
```

Có thể thấy:

```text
active (running)
inactive (dead)
failed
activating
deactivating
```

`failed` khác `inactive`.

`inactive` có thể hoàn toàn bình thường vì service đã stop cleanly.

`failed` nghĩa là systemd ghi nhận failure như process exit error, crash, timeout hoặc restart limit. [man7](https://man7.org/linux/man-pages/man1/systemctl.1.html?utm_source=chatgpt.com)

---

# 45. Lệnh quản trị systemd cốt lõi

| Command                    | Ý nghĩa                               |
| -------------------------- | ------------------------------------- |
| `systemctl start X`        | Start unit                            |
| `systemctl stop X`         | Stop                                  |
| `systemctl restart X`      | Stop rồi start                        |
| `systemctl reload X`       | Yêu cầu application reload nếu hỗ trợ |
| `systemctl status X`       | Human-readable state                  |
| `systemctl is-active X`    | Kiểm tra runtime state                |
| `systemctl enable X`       | Configure activation/startup          |
| `systemctl disable X`      | Bỏ enable links                       |
| `systemctl is-enabled X`   | Check enablement                      |
| `systemctl cat X`          | Xem unit definition                   |
| `systemctl show X`         | Xem machine-readable properties       |
| `systemctl daemon-reload`  | Reload unit definitions               |
| `systemctl reset-failed X` | Reset failed/start-limit state        |

---

# 46. `restart`, `reload`, `daemon-reload` — phải thuộc thật chắc

### `restart`

```bash
systemctl restart app
```

nghĩa là:

```text
stop process
+
start process
```

Có interruption.

### `reload`

```bash
systemctl reload app
```

nghĩa là:

```text
ask application to reload its own configuration
```

Chỉ hoạt động nếu service hỗ trợ reload.

### `daemon-reload`

```bash
systemctl daemon-reload
```

nghĩa là:

```text
systemd re-read unit files / dependency configuration
```

Nó **không** đồng nghĩa restart application.

Do đó:

```text
edit app.service
↓
daemon-reload
↓
restart app.service
```

thường là workflow cần thiết.

---

# 47. Service Startup Flow

Khi boot:

```text
kernel
  ↓
systemd PID 1
  ↓
targets/dependency graph
  ↓
multi-user.target
  ↓
enabled/wanted units
  ↓
day4-demo.service
  ↓
[Service]
  ↓
User / Group
Environment
ExecStart
  ↓
application process
```

Nếu service chưa enabled:

```text
systemctl start
```

vẫn có thể chạy nó manually.

Nếu enabled nhưng hiện tại stopped:

```text
systemctl is-enabled
```

có thể trả:

```text
enabled
```

trong khi:

```text
systemctl is-active
```

trả:

```text
inactive
```

Không mâu thuẫn.

---

# 48. GUIDED LAB BẮT BUỘC DAY 4

## Safety trước khi thực hành

Lab này sẽ:

```text
install package
create system user
write /usr/local/bin/
write /etc/
write /etc/systemd/system/
start/stop a service
intentionally break executable permissions
```

Chỉ thực hiện trên **lab VM/non-production host**.

Không làm trên shared production server.

Trước mỗi thay đổi, ta sẽ giữ blast radius nhỏ và có cleanup.

---

# 49. Lab Phase A — Baseline và Evidence

Tạo evidence directory:

```bash
mkdir -p "$HOME/day4-evidence"
```

Ghi baseline:

```bash
{
    echo "=== DATE ==="
    date -Is

    echo "=== OS ==="
    cat /etc/os-release

    echo "=== KERNEL ==="
    uname -r

    echo "=== SYSTEMD ==="
    systemctl --version

    echo "=== CURRENT USER ==="
    id

    echo "=== PID 1 ==="
    ps -p 1 -o pid,ppid,user,comm,args
} | tee "$HOME/day4-evidence/baseline.txt"
```

**Expected invariant:** PID 1 là system/service manager của lab system và `systemctl` hoạt động.

---

# 50. Lab Phase B — Install Package

Trên Ubuntu/Debian:

```bash
sudo apt update
```

Inspect trước:

```bash
apt-cache policy tree
```

Cài:

```bash
sudo apt install tree
```

**Đọc transaction summary trước khi bấm `Y`.**

Verify:

```bash
dpkg-query -W -f='${Status} ${Package} ${Version}\n' tree
```

```bash
command -v tree
```

```bash
tree --version
```

Lưu evidence:

```bash
{
    dpkg-query -W -f='${Status} ${Package} ${Version}\n' tree
    command -v tree
    tree --version
} | tee "$HOME/day4-evidence/package.txt"
```

Nếu `tree` đã được cài từ trước, hãy chọn một approved small package khác như `jq` để thực sự hoàn thành yêu cầu **install a package**.

Không remove package đã tồn tại từ trước trong bước cleanup.

---

# 51. Lab Phase C — Process Exercise

Tạo process:

```bash
sleep 600 &
PID=$!
```

Inspect:

```bash
ps -o pid,ppid,user,stat,lstart,etime,cmd -p "$PID"
```

Evidence:

```bash
ps -o pid,ppid,user,stat,lstart,etime,cmd -p "$PID" \
  | tee "$HOME/day4-evidence/process-running.txt"
```

Stop:

```bash
kill -STOP "$PID"
```

Verify:

```bash
ps -o pid,ppid,stat,cmd -p "$PID"
```

Continue:

```bash
kill -CONT "$PID"
```

Terminate:

```bash
kill -TERM "$PID"
```

Verify disappearance:

```bash
ps -p "$PID"
```

Expected invariant:

```text
process không còn running
```

---

# 52. Lab Phase D — Environment Exercise

Shell-only variable:

```bash
DAY4_LOCAL=hello
```

Current shell:

```bash
echo "$DAY4_LOCAL"
```

Child:

```bash
bash -c 'echo "${DAY4_LOCAL-unset}"'
```

Bây giờ export:

```bash
export DAY4_LOCAL
```

Child:

```bash
bash -c 'echo "$DAY4_LOCAL"'
```

One-command environment:

```bash
env DAY4_ONESHOT=one-command \
    bash -c 'echo "$DAY4_ONESHOT"'
```

Cleanup:

```bash
unset DAY4_LOCAL
```

Bạn vừa chứng minh:

```text
shell variable ≠ automatically child environment
exported variable → inherited by child
```

---

# 53. Lab Phase E — Tạo Least-Privilege Service Account

Ta không chạy demo service bằng root.

Kiểm tra trước:

```bash
getent passwd day4svc
```

Nếu chưa có:

```bash
sudo useradd \
  --system \
  --no-create-home \
  --home-dir /nonexistent \
  --shell /usr/sbin/nologin \
  day4svc
```

Verify:

```bash
id day4svc
```

```bash
getent passwd day4svc
```

Đây là **Supplementary / Beyond explicit syllabus**, nhưng trực tiếp áp dụng least privilege từ Day 3.

---

# 54. Lab Phase F — Tạo Application Script

Tạo:

```bash
sudo tee /usr/local/bin/day4-demo >/dev/null <<'EOF'
#!/usr/bin/env bash

set -u

: "${DAY4_MESSAGE:?DAY4_MESSAGE is required}"
interval="${DAY4_INTERVAL:-5}"

shutdown_handler() {
    printf '%s shutdown signal received; exiting cleanly\n' "$(date -Is)"
    exit 0
}

trap shutdown_handler TERM INT

printf '%s service started pid=%s user=%s message=%s interval=%s\n' \
    "$(date -Is)" "$$" "$(id -un)" "$DAY4_MESSAGE" "$interval"

while true; do
    printf '%s pid=%s message=%s\n' \
        "$(date -Is)" "$$" "$DAY4_MESSAGE"

    sleep "$interval" &
    child=$!
    wait "$child" || true
done
EOF
```

Owner:

```bash
sudo chown root:root /usr/local/bin/day4-demo
```

Permissions:

```bash
sudo chmod 0755 /usr/local/bin/day4-demo
```

Verify:

```bash
ls -l /usr/local/bin/day4-demo
```

Điểm security:

```text
service account được execute
service account không được sửa executable
root sở hữu executable
```

---

# 55. Lab Phase G — Environment File

Tạo:

```bash
sudo tee /etc/day4-demo.env >/dev/null <<'EOF'
DAY4_MESSAGE=hello-from-systemd
DAY4_INTERVAL=5
EOF
```

Permissions:

```bash
sudo chown root:root /etc/day4-demo.env
sudo chmod 0644 /etc/day4-demo.env
```

Ở đây dữ liệu không phải secret nên `0644` chấp nhận được cho lab.

Không áp dụng pattern này cho password/token production.

---

# 56. Lab Phase H — Tạo systemd Service

Tạo:

```bash
sudo tee /etc/systemd/system/day4-demo.service >/dev/null <<'EOF'
[Unit]
Description=Day 4 systemd learning service

[Service]
Type=exec
User=day4svc
Group=day4svc
EnvironmentFile=/etc/day4-demo.env
ExecStart=/usr/local/bin/day4-demo
Restart=on-failure
RestartSec=2s

[Install]
WantedBy=multi-user.target
EOF
```

Inspect:

```bash
cat /etc/systemd/system/day4-demo.service
```

---

# 57. Supplementary — Validate Unit trước khi Load

Đây là thói quen production rất tốt:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/day4-demo.service
```

Mục tiêu:

```text
không có fatal syntax/configuration errors liên quan unit
```

`systemd-analyze verify` có thể hiển thị một số warning phụ thuộc distro/system; đừng học thuộc rằng output phải tuyệt đối rỗng. Hãy đọc message.

---

# 58. `daemon-reload`

Vì ta vừa tạo unit file:

```bash
sudo systemctl daemon-reload
```

Sau đó:

```bash
systemctl cat day4-demo.service
```

Bạn cần phân biệt:

```text
file tồn tại trên disk
```

và:

```text
systemd manager đã reload definition
```

---

# 59. Start Service

Trước tiên:

```bash
systemctl is-enabled day4-demo.service
```

Có thể nhận:

```text
disabled
```

Đây là bình thường.

Start:

```bash
sudo systemctl start day4-demo.service
```

Check:

```bash
systemctl status day4-demo.service --no-pager
```

Expected invariant:

```text
Active: active (running)
```

---

# 60. Machine-Friendly Verification

```bash
systemctl is-active day4-demo.service
```

Expected:

```text
active
```

Hoặc automation-friendly:

```bash
systemctl is-active --quiet day4-demo.service
echo $?
```

Expected invariant:

```text
exit code 0
```

Systemctl sử dụng non-zero status khi `is-active` không thỏa điều kiện. [man7](https://man7.org/linux/man-pages/man1/systemctl.1.html?utm_source=chatgpt.com)

---

# 61. Inspect MainPID

```bash
systemctl show day4-demo.service \
  -p MainPID \
  -p ActiveState \
  -p SubState \
  -p User \
  -p Group
```

Lấy PID:

```bash
PID=$(systemctl show \
  --property MainPID \
  --value \
  day4-demo.service)
```

Kiểm tra:

```bash
echo "$PID"
```

Process:

```bash
ps -o pid,ppid,user,group,stat,etime,cmd -p "$PID"
```

Expected invariant:

```text
USER = day4svc
service process đang running
PID trùng MainPID do systemd report
```

---

# 62. Inspect Environment thật sự của Service Process

```bash
sudo tr '\0' '\n' < "/proc/$PID/environ" \
  | grep '^DAY4_'
```

Bạn kỳ vọng:

```text
DAY4_MESSAGE=hello-from-systemd
DAY4_INTERVAL=5
```

Đây là bằng chứng mạnh hơn rất nhiều so với việc chỉ nói:

> “Tôi đã viết EnvironmentFile nên chắc service có biến.”

Bạn đã kiểm tra **actual running process**.

---

# 63. Inspect Process Tree

```bash
pstree -p "$PID"
```

Bạn có thể thấy:

```text
day4-demo(...)---sleep(...)
```

Vì Bash process của chúng ta đang spawn `sleep`.

Đây là lý do operations không nên luôn nghĩ:

```text
service = đúng một process
```

Một service có thể chứa cả process tree/cgroup.

---

# 64. Basic Log Inspection

**Supplementary / Preview Day 6:** `journalctl` được syllabus học sâu ở Day 6, nhưng ở Day 4 ta cần dùng tối thiểu để recovery failed service.

```bash
sudo journalctl \
  -u day4-demo.service \
  -n 20 \
  --no-pager
```

Bạn nên thấy log:

```text
service started ...
pid=...
message=hello-from-systemd
```

Exact timestamps/PID khác nhau; đó là **example output**.

---

# 65. Enable Service Startup

Hiện tại service đang chạy nhưng chưa chắc enabled.

```bash
systemctl is-enabled day4-demo.service
```

Enable:

```bash
sudo systemctl enable day4-demo.service
```

Verify:

```bash
systemctl is-enabled day4-demo.service
```

Expected:

```text
enabled
```

Inspect link:

```bash
ls -l \
  /etc/systemd/system/multi-user.target.wants/day4-demo.service
```

Bạn thường thấy symlink tới:

```text
/etc/systemd/system/day4-demo.service
```

Đây chính là mechanism enable cơ bản: `systemctl enable` tạo symlink dựa trên `[Install]`; nó không đồng nghĩa start service. [man7](https://www.man7.org/linux/man-pages/man1/systemctl.1.html?utm_source=chatgpt.com)

---

# 66. Không cần reboot chỉ để chứng minh enable

Trong production/lab remote, không nên reboot chỉ để “thử cho chắc” nếu chưa chuẩn bị console/recovery.

Evidence:

```bash
systemctl is-enabled day4-demo.service
```

cộng với symlink/dependency inspection đã đủ cho bài Day 4.

Nếu đây là disposable local VM và bạn chủ động muốn test boot startup, bạn có thể reboot sau khi đã bảo đảm có console access. Nhưng đây là **optional** chứ không nên reboot một host tùy tiện.

---

# 67. Environment Change Test

Xem config:

```bash
cat /etc/day4-demo.env
```

Đổi:

```bash
sudo tee /etc/day4-demo.env >/dev/null <<'EOF'
DAY4_MESSAGE=configuration-changed
DAY4_INTERVAL=3
EOF
```

Không chạy `daemon-reload`.

Kiểm tra process hiện tại:

```bash
PID=$(systemctl show -p MainPID --value day4-demo)
sudo tr '\0' '\n' < "/proc/$PID/environ" \
  | grep '^DAY4_'
```

Bạn vẫn thấy environment cũ.

Tại sao?

Vì process environment được thiết lập khi process được start.

Restart:

```bash
sudo systemctl restart day4-demo.service
```

Lấy PID mới:

```bash
NEWPID=$(systemctl show -p MainPID --value day4-demo)
echo "$NEWPID"
```

Inspect:

```bash
sudo tr '\0' '\n' < "/proc/$NEWPID/environ" \
  | grep '^DAY4_'
```

Bây giờ:

```text
DAY4_MESSAGE=configuration-changed
DAY4_INTERVAL=3
```

Bạn vừa chứng minh:

```text
EnvironmentFile content changed
→ no daemon-reload required
→ running process unchanged
→ restart
→ new process receives new environment
```

---

# 68. FAILURE INJECTION — yêu cầu quan trọng của Day 4

Bây giờ ta cố tình phá service.

**Chỉ làm với `day4-demo`; không thực hiện trên SSH, database, networking service.**

Stop trước:

```bash
sudo systemctl stop day4-demo.service
```

Verify:

```bash
systemctl is-active day4-demo.service
```

Có thể thấy:

```text
inactive
```

Bây giờ bỏ execute permission:

```bash
sudo chmod 0644 /usr/local/bin/day4-demo
```

Inspect:

```bash
ls -l /usr/local/bin/day4-demo
```

Bạn sẽ thấy không còn `x`.

---

# 69. Trigger Failure

```bash
sudo systemctl start day4-demo.service
```

Có thể command báo failure ngay hoặc service thử auto-restart do:

```ini
Restart=on-failure
```

Inspect:

```bash
systemctl status day4-demo.service --no-pager
```

Sau vài lần retry, tùy systemd/start-limit configuration, unit có thể trở thành:

```text
failed
```

Expected invariant:

```text
service không thể successfully execute ExecStart
```

Không học thuộc exact error line.

---

# 70. Troubleshooting theo Evidence, không Guess

Flow đúng:

```text
symptom
  ↓
expected state
  ↓
scope
  ↓
evidence
  ↓
hypothesis
  ↓
safe test
  ↓
root cause
  ↓
remediation
  ↓
validation
```

Ở đây symptom:

```text
day4-demo.service không start
```

Expected:

```text
active (running)
```

Scope:

```text
chỉ day4-demo.service
```

---

# 71. Evidence 1 — status

```bash
systemctl status day4-demo.service --no-pager
```

Quan sát:

```text
Loaded
Active
Main PID nếu có
Result
ExecStart
recent messages
```

---

# 72. Evidence 2 — service definition

```bash
systemctl cat day4-demo.service
```

Kiểm tra:

```ini
ExecStart=/usr/local/bin/day4-demo
```

---

# 73. Evidence 3 — File và Permissions

```bash
ls -l /usr/local/bin/day4-demo
```

Có:

```text
-rw-r--r--
```

Trong khi executable cần execute bit.

Có thể kiểm tra cả path:

```bash
namei -l /usr/local/bin/day4-demo
```

Rất hữu ích khi debug permissions trên nhiều directory component.

---

# 74. Evidence 4 — Basic Journal

**Preview Day 6:**

```bash
sudo journalctl \
  -u day4-demo.service \
  -n 50 \
  --no-pager
```

Bạn có thể thấy error liên quan tới execution/permission.

Nếu systemd report một code kiểu:

```text
203/EXEC
```

hãy coi đây là **example output**; điều quan trọng là đọc evidence thực tế của máy bạn.

---

# 75. Evidence 5 — systemd properties

```bash
systemctl show day4-demo.service \
  -p ActiveState \
  -p SubState \
  -p Result \
  -p ExecMainCode \
  -p ExecMainStatus \
  -p NRestarts
```

Đây là cách tốt hơn để automation parse state so với parse màu/chữ trong `systemctl status`.

---

# 76. Root Cause

Ta có:

```text
ExecStart points to:
    /usr/local/bin/day4-demo

Owner:
    root

Service user:
    day4svc

Mode:
    0644

Execute permission:
    absent
```

Root cause:

```text
ExecStart executable lost execute permission.
```

Đây là kết luận dựa trên evidence, không phải đoán.

---

# 77. Remediation

Khôi phục known-good permission:

```bash
sudo chmod 0755 /usr/local/bin/day4-demo
```

Verify:

```bash
ls -l /usr/local/bin/day4-demo
```

Expected:

```text
-rwxr-xr-x
```

---

# 78. `reset-failed`

Nếu service đã hit failure/start-rate limit:

```bash
sudo systemctl reset-failed day4-demo.service
```

Điểm rất quan trọng:

```text
reset-failed ≠ fix root cause
```

Ta chỉ chạy nó **sau khi đã sửa root cause**.

Systemd documentation cho biết `reset-failed` ngoài việc clear failed state còn reset start-rate-limit counter và service restart counter, hữu ích khi unit đã hit start limit. [man7](https://www.man7.org/linux/man-pages/man1/systemctl.1.html?utm_source=chatgpt.com)

Start:

```bash
sudo systemctl start day4-demo.service
```

---

# 79. Validation sau Recovery

Check state:

```bash
systemctl is-active day4-demo.service
```

Expected:

```text
active
```

Check status:

```bash
systemctl status day4-demo.service --no-pager
```

Check process:

```bash
PID=$(systemctl show -p MainPID --value day4-demo)
ps -o pid,ppid,user,group,stat,etime,cmd -p "$PID"
```

Check environment:

```bash
sudo tr '\0' '\n' < "/proc/$PID/environ" \
  | grep '^DAY4_'
```

Check logs:

```bash
sudo journalctl \
  -u day4-demo.service \
  -n 20 \
  --no-pager
```

Check enablement:

```bash
systemctl is-enabled day4-demo.service
```

Chỉ khi tất cả các evidence phù hợp mới kết luận:

```text
Service recovered.
```

---

# 80. Lưu Evidence

```bash
{
    echo "=== STATUS ==="
    systemctl status day4-demo.service --no-pager

    echo
    echo "=== SHOW ==="
    systemctl show day4-demo.service \
      -p ActiveState \
      -p SubState \
      -p MainPID \
      -p Result

    echo
    echo "=== ENABLEMENT ==="
    systemctl is-enabled day4-demo.service

    echo
    echo "=== PROCESS ==="
    PID=$(systemctl show -p MainPID --value day4-demo.service)
    ps -o pid,ppid,user,group,stat,etime,cmd -p "$PID"

} | tee "$HOME/day4-evidence/service-recovery.txt"
```

Bây giờ evidence pack của bạn tối thiểu nên có:

```text
baseline.txt
package.txt
process-running.txt
service-recovery.txt
```

---

# 81. LAB Cleanup

Nếu muốn dọn sạch lab:

```bash
sudo systemctl disable --now day4-demo.service
```

Xóa unit:

```bash
sudo rm /etc/systemd/system/day4-demo.service
```

Reload:

```bash
sudo systemctl daemon-reload
```

Reset stale failure state:

```bash
sudo systemctl reset-failed
```

Xóa application:

```bash
sudo rm /usr/local/bin/day4-demo
```

Xóa environment file:

```bash
sudo rm /etc/day4-demo.env
```

Xóa service account:

```bash
sudo userdel day4svc
```

Nếu `tree` được cài **chỉ vì lab này**:

```bash
sudo apt remove tree
```

Đừng remove nếu package đã có sẵn trước lab hoặc có dependency/use case khác.

---

# 82. Operations Mental Model — Troubleshooting Service thật

Khi gặp:

```text
"Application is down"
```

đừng lập tức:

```bash
sudo systemctl restart app
```

Một fresher tốt cần suy nghĩ:

```text
Is it really down?
       ↓
Is systemd unit active?
       ↓
Does MainPID exist?
       ↓
Is expected process running?
       ↓
Did it exit?
       ↓
What exit/result?
       ↓
Could ExecStart execute?
       ↓
Correct user/group?
       ↓
Correct permissions?
       ↓
Correct environment?
       ↓
Dependency available?
       ↓
Recent config/change?
       ↓
What do logs say?
```

Sau đó mới:

```text
safe remediation
```

---

# 83. Restart không phải Troubleshooting

Nếu chạy:

```bash
systemctl restart app
```

ngay lập tức, bạn có thể:

```text
destroy transient evidence
change PID
clear symptoms
introduce downtime
hide root cause
trigger restart storm
```

Trong production nên:

```text
inspect first
preserve evidence
then remediate
```

trừ khi runbook/incident priority quy định khác.

---

# 84. `systemctl status` không phải toàn bộ Truth

Status rất tốt để bắt đầu.

Nhưng cần kết hợp:

```text
systemctl show
ps
/proc
unit file
permissions
environment
logs
dependency state
```

Ví dụ:

```text
systemd: active
```

không chắc application endpoint hoạt động.

Một Java service có thể:

```text
process running
but DB connection broken
```

hoặc:

```text
process running
but port not listening
```

Day 5 sẽ bổ sung network/port checks.

---

# 85. `active` ≠ Application Healthy

Mental model cực quan trọng:

```text
systemctl is-active
      |
      v
"systemd believes service runtime is active"
```

không có nghĩa:

```text
HTTP endpoint healthy
DB connected
business transaction works
```

Sau này validation phải đi tiếp:

```text
process
→ socket
→ HTTP
→ app
→ DB
→ monitoring
```

---

# 86. Security & Least Privilege

Một service application không nên chạy root chỉ vì “dễ”.

Ví dụ tốt hơn:

```ini
User=myapp
Group=myapp
```

Executable:

```text
root:root
0755
```

Configuration writable chỉ bởi admin nếu service không cần sửa.

Điều này ngăn compromised application tự sửa executable/config quan trọng.

**Supplementary hardening** sau này có thể dùng thêm systemd directives như:

```ini
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=
ProtectHome=
```

Nhưng không bật mù quáng. Hardening phải tương thích với application behavior.

---

# 87. Anti-Patterns phải tránh

| Anti-pattern                                            | Vì sao sai                      |
| ------------------------------------------------------- | ------------------------------- |
| `kill -9` ngay lập tức                                  | Không cho graceful cleanup      |
| Restart trước khi thu evidence                          | Có thể mất root cause           |
| `sudo chmod 777` để sửa permission                      | Over-permissive, che root cause |
| Chạy app service bằng root không cần thiết              | Tăng blast radius               |
| Sửa `/usr/lib/systemd/system/*.service` trực tiếp       | Package update có thể overwrite |
| Nghĩ `enable = start`                                   | Sai lifecycle semantics         |
| Nghĩ `daemon-reload = restart app`                      | Sai                             |
| Nghĩ `export VAR` trong SSH shell sẽ vào system service | Sai model                       |
| Đặt password vào environment tùy tiện                   | Secret leakage risk             |
| `apt upgrade -y` trên production để “thử”               | Blast radius lớn                |
| Parse `systemctl status` trong script                   | Nên dùng `is-active`/`show`     |
| `reset-failed` rồi coi như đã fix                       | Chỉ reset state/counters        |

---

# 88. L1-safe versus Escalation

Một L1 fresher thường có thể an toàn làm những việc đã được runbook/authorization cho phép:

```text
inspect service
inspect process
inspect unit
inspect permissions
inspect basic logs
start/stop/restart approved service
restore known-good config/permissions
reset failed state after fixing cause
validate service state
collect evidence
```

Nên escalate nếu gặp:

```text
repeated unknown crashes
core dumps
kernel/storage D-state
filesystem corruption
package dependency conflicts
security compromise suspicion
unknown third-party repositories
privilege requirement changes
system-wide dependency failure
production data corruption
unapproved config architecture changes
failure persists after known-safe remediation
```

Khi escalate, không chỉ nói:

> “Service failed, please check.”

Một handoff tốt có dạng:

```text
Symptom:
day4-demo failed to start.

Expected:
active/running.

Scope:
only day4-demo affected.

Evidence:
ExecStart=/usr/local/bin/day4-demo
file mode was 0644
systemd reported execution failure.

Recent change:
execute permission removed during test.

Action:
restored 0755, reset failed state, started service.

Validation:
is-active=active
MainPID=<PID>
process owner=day4svc
is-enabled=enabled.

Current status:
recovered.
```

Đó là tư duy operations chuyên nghiệp.

---

# 89. Rollback / Recovery Model

Nếu thay đổi unit:

```text
backup known-good
↓
change
↓
verify syntax
↓
daemon-reload
↓
restart/start
↓
validate
```

Nếu lỗi:

```text
restore known-good unit
↓
daemon-reload
↓
restart
↓
validate
```

Nếu chỉ thay environment file:

```text
backup env
↓
edit
↓
restart
↓
validate
```

Nếu lỗi:

```text
restore env
↓
restart
↓
validate
```

Không cần `daemon-reload` chỉ vì nội dung EnvironmentFile thay đổi.

---

# 90. Exam Focus

Bạn nên có thể trả lời ngay các câu kiểu:

```text
Program và process khác nhau thế nào?

PID và PPID là gì?

SIGTERM khác SIGKILL thế nào?

Tại sao không dùng kill -9 đầu tiên?

SIGSTOP/SIGCONT làm gì?

Zombie là gì?

Environment variable được child process nhận thế nào?

export có tác dụng gì?

systemd service là gì?

[Unit], [Service], [Install] dùng làm gì?

ExecStart là gì?

After= có tự start dependency không?

Wants= khác Requires= thế nào?

start khác enable thế nào?

restart khác reload thế nào?

daemon-reload dùng lúc nào?

EnvironmentFile được load lúc nào?

Tại sao sửa EnvironmentFile cần restart process?

Restart=on-failure có tác dụng gì?

reset-failed có sửa root cause không?

active service có chắc application healthy không?
```

---

# 91. Practical Exam Focus

Nếu giám khảo giao:

> “Tạo service chạy background, start at boot, chạy dưới non-root user và dùng environment configuration.”

Bạn phải nghĩ được ngay:

```text
service account
↓
executable
↓
ownership/permissions
↓
environment file
↓
.service unit
↓
systemd-analyze verify
↓
daemon-reload
↓
start
↓
status/is-active
↓
inspect MainPID
↓
verify user
↓
verify environment
↓
enable
↓
is-enabled
↓
evidence
```

Nếu service fail:

```text
do NOT blindly restart repeatedly

status
↓
unit
↓
ExecStart
↓
user
↓
permissions
↓
environment
↓
logs
↓
root cause
↓
fix
↓
reset-failed if needed
↓
start
↓
validate
```

---

# 92. Knowledge Check

Hãy tự trả lời **không nhìn lại phần trên**:

1. Một file `/usr/bin/java` có phải process không? Khi nào nó trở thành process?
2. PID và PPID khác nhau thế nào?
3. Process có state `Z` nghĩa là gì? Tại sao `kill -9` zombie không phải giải pháp?
4. `SIGTERM` và `SIGKILL` khác nhau về graceful cleanup thế nào?
5. Signal nào không thể bị catch/ignore: `SIGTERM`, `SIGKILL`, hay `SIGHUP`?
6. `DAY4=x` và `export DAY4=x` khác nhau thế nào đối với child process?
7. Tại sao `export DB_HOST=x` trong SSH shell không đảm bảo `systemctl restart app` sẽ truyền biến đó cho system service?
8. `systemctl start nginx` khác `systemctl enable nginx` thế nào?
9. Khi sửa `/etc/systemd/system/app.service`, lệnh nào thường cần chạy?
10. Khi chỉ sửa `/etc/app.env` được tham chiếu bằng `EnvironmentFile=`, có cần `daemon-reload` không? Process đang chạy có tự nhận biến mới không?
11. `After=postgresql.service` có nghĩa systemd sẽ tự start PostgreSQL không?
12. `Restart=on-failure` giải quyết được config sai không?
13. `systemctl reset-failed` sửa lỗi application hay chỉ reset systemd failure state/counters?
14. Vì sao `systemctl status app` báo active chưa đủ kết luận application healthy?
15. Khi service không start, ba evidence đầu tiên bạn sẽ thu là gì?

---

# 93. Independent Practical Challenge

Sau guided lab, bạn nên tự làm lại một service mới tên:

```text
fresher-clock.service
```

Yêu cầu không nhìn solution Day 4:

```text
Executable:
/usr/local/bin/fresher-clock

Run as:
dedicated non-root service account

Environment:
CLOCK_INTERVAL
CLOCK_MESSAGE

Configuration:
/etc/fresher-clock.env

Service:
fresher-clock.service

Runtime:
print timestamp + message periodically

Startup:
enabled for multi-user.target

Recovery challenge:
intentionally introduce one failure,
diagnose only from evidence,
repair,
validate,
save evidence.
```

Bạn phải chứng minh được:

```text
systemctl is-active
systemctl is-enabled
MainPID
ps shows correct user
/proc/PID/environ shows expected variables
service survives/restarts correctly as designed
failure root cause identified
recovery evidence captured
```

**Không được dùng `chmod 777`, không chạy service root, không dùng `kill -9` trừ khi bạn có lý do kỹ thuật rõ ràng.**

---

# 94. Day 4 Mastery Map

Sau bài hôm nay, mental model mà bạn cần giữ là:

```text
                    Linux Host
                        |
                 systemd / PID 1
                        |
            +-----------+-----------+
            |                       |
       unit definition          boot target
            |                       |
      [Unit]/[Service]          enable links
      [Install]                     |
            |                       |
            +-----------+-----------+
                        |
                      start
                        |
              service process
                        |
          +-------------+-------------+
          |             |             |
         PID        environment      UID/GID
          |                           |
        PPID                     permissions
          |
       process state
          |
       signals
          |
       exit/failure
          |
        systemd
          |
   active / inactive / failed
          |
 troubleshooting evidence
          |
      root cause
          |
        recovery
          |
       validation
```

Đây chính là nền tảng mà bạn sẽ dùng lại khi quản trị **Tomcat/Java, PostgreSQL, Prometheus, Grafana và các service production khác**.

## Mastery Check của Day 4

Bạn **chưa nên coi Day 4 là mastered chỉ vì đã đọc hết lý thuyết**. Day 4 chỉ đạt practical mastery khi bạn thực sự tự tay hoàn thành chuỗi:

```text
install package
→ inspect package
→ create process
→ inspect PID/PPID/state
→ send safe signals
→ demonstrate environment inheritance
→ create non-root service account
→ create executable
→ create EnvironmentFile
→ create systemd unit
→ daemon-reload
→ start
→ inspect MainPID/process/environment
→ enable
→ inject failure
→ collect evidence
→ identify root cause
→ recover
→ validate
→ preserve evidence
```

Nếu bạn thực hành theo đúng lab trên, **hãy gửi cho mình output/evidence từng bước hoặc output ở bước bạn gặp lỗi**. Mình sẽ đóng vai mentor vận hành: kiểm tra evidence, không chỉ đưa đáp án, và buộc reasoning theo đúng flow **symptom → evidence → hypothesis → root cause → remediation → validation** trước khi coi phần Day 4 đó hoàn thành.
