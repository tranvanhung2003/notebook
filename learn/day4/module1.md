# Day 4 — Module 1: Linux Processes, Signals & Environment Variables

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 4 thuộc Linux Fundamentals và bắt buộc bao phủ:

**Concept/Lecture:** `Processes`, `signals`, `systemd services`, `packages`, `environment variables`, `service startup`.

**Assignment/Lab:** cài một package, tạo/quản lý một `systemd service`, kiểm tra process và phục hồi một service bị lỗi. :chatgpt-content-reference{index="0"}

Trong **Module 1**, chúng ta tập trung sâu vào ba nền tảng bắt buộc của Day 4:

**Processes → Signals → Environment Variables**

`systemd services`, package management và full failure-recovery lab sẽ được nối tiếp ở Module 2–3.

Phần `fork()/execve()`, `/proc`, process groups, shell job control, zombie internals, threads và security implications của environment variables là **Supplementary / Beyond the explicit syllabus**. Tuy nhiên chúng rất quan trọng để hiểu đúng ba chủ đề bắt buộc và chuẩn bị cho Java/Tomcat về sau.

Các mô tả kỹ thuật dưới đây đã được đối chiếu với Linux man-pages hiện hành, procps-ng và GNU Bash documentation. Linux `/proc` hiện được định nghĩa là pseudo-filesystem cung cấp interface tới kernel data structures; mỗi process có một `/proc/<PID>/` tương ứng. :chatgpt-content-reference{index="1"}

---

# 2. Mục tiêu của Module 1

Sau module này, bạn không chỉ cần biết chạy:

```bash
ps
kill
export
```

mà phải có khả năng nhận một tình huống vận hành như:

```text
"Ứng dụng dường như vẫn đang chạy nhưng không hoạt động đúng."
```

và tự đặt được chuỗi câu hỏi:

```text
Process có thực sự tồn tại không?
        ↓
PID chính xác là gì?
        ↓
Process chạy dưới user nào?
        ↓
Parent process là gì?
        ↓
Command/executable thật sự là gì?
        ↓
Process đang ở state nào?
        ↓
Environment của nó là gì?
        ↓
Có gửi signal được không?
        ↓
Signal nào phù hợp?
        ↓
Sau signal, state/PID thay đổi thế nào?
        ↓
Evidence nào chứng minh kết quả?
```

Đây chính là tư duy L1 Operations mà chúng ta sẽ xây dựng.

---

# 3. Mental Model lớn nhất của Module 1

Trước tiên hãy phân biệt bốn khái niệm thường xuyên bị fresher trộn lẫn.

| Khái niệm   | Ý nghĩa                                                                      |
| ----------- | ---------------------------------------------------------------------------- |
| **Program** | Code/executable/script nằm trên filesystem                                   |
| **Process** | Một instance đang thực thi của một program                                   |
| **Job**     | Khái niệm do shell quản lý, thường tương ứng một process hoặc process group  |
| **Service** | Một workload được quản lý như service; có thể bao gồm một hoặc nhiều process |

Ví dụ:

```text
/usr/bin/java
```

là một executable.

Khi chạy:

```bash
java -jar myapp.jar
```

Linux tạo ra một Java **process**.

Nếu Java process đó được quản lý bởi:

```text
myapp.service
```

thì nó thuộc một **systemd service**.

Ba Java application khác nhau hoàn toàn có thể cùng sử dụng `/usr/bin/java`, nhưng chúng là ba process khác nhau:

```text
/usr/bin/java
       │
       ├── PID 4210 → app-a.jar
       ├── PID 5321 → app-b.jar
       └── PID 8472 → app-c.jar
```

Do đó:

```text
program ≠ process
```

và:

```text
process ≠ service
```

Điều này cực kỳ quan trọng khi troubleshoot.

---

# 4. Process thực chất là gì?

Một **process** là một execution context mà Linux kernel đang quản lý.

Nó không chỉ có executable code. Một process còn gắn với nhiều loại state/resource:

```text
Process
│
├── PID
├── PPID
├── credentials
│   ├── UID
│   └── GID
├── virtual address space
├── executable
├── command-line arguments
├── environment
├── current working directory
├── open file descriptors
├── signal state
├── scheduling state
└── one or more threads
```

Đó là lý do troubleshooting process không thể chỉ làm:

```bash
ps aux | grep java
```

rồi kết luận.

Process tồn tại chỉ chứng minh:

> Kernel vẫn có process đó.

Nó **không chứng minh application healthy**.

Ví dụ Java process có thể vẫn sống nhưng:

```text
database connection pool exhausted
HTTP thread pool exhausted
deadlock
application initialization incomplete
wrong environment
wrong configuration
port không listening
disk full
```

Những vấn đề đó sẽ được nối dần qua các Day sau.

---

# 5. PID — Process Identifier

Mỗi process được Linux gán một identifier gọi là:

```text
PID = Process ID
```

Ví dụ:

```bash
ps -ef
```

có thể cho:

```text
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 08:00 ?        00:00:02 /sbin/init
student     4821    4700  0 10:12 pts/0    00:00:00 sleep 300
```

Process:

```text
sleep 300
```

có:

```text
PID = 4821
```

PID được dùng để Linux và administrator xác định target cho rất nhiều operation, trong đó có gửi signal. Linux documentation mô tả PID là identifier được cấp cho process khi process được tạo; PID tiếp tục được giữ nguyên qua `execve()`. :chatgpt-content-reference{index="2"}

---

# 6. PID không phải identity vĩnh viễn

Đây là một điểm vận hành quan trọng.

Giả sử:

```text
10:00
PID 4821 → java app-a
```

Sau đó process kết thúc.

Một thời gian sau kernel có thể tái sử dụng PID:

```text
15:00
PID 4821 → một process hoàn toàn khác
```

Vì vậy, một PID lưu trong notebook từ vài giờ trước **không đủ để chứng minh đó vẫn là process cũ**.

Trước khi thao tác nguy hiểm như:

```bash
kill 4821
```

administrator tốt sẽ kiểm tra lại:

```bash
ps -p 4821 -o pid,ppid,user,lstart,etime,stat,cmd
```

Ví dụ:

```text
PID   PPID USER    STARTED                  ELAPSED STAT CMD
4821  4700 student Tue Sep 29 10:12:04 2026  01:03 S    sleep 300
```

Ta vừa kiểm tra được nhiều thứ hơn chỉ PID.

---

# 7. PPID — Parent Process ID

Process thường tồn tại trong quan hệ:

```text
parent
  │
  └── child
```

`PPID` là:

```text
Parent Process ID
```

Linux `getppid()` trả về PID của process cha; nếu parent ban đầu đã chết, process có thể được reparent sang `init` hoặc một process đóng vai trò `subreaper`. :chatgpt-content-reference{index="3"}

Ví dụ:

```text
bash
PID 4000
 │
 └── sleep 300
     PID 4821
     PPID 4000
```

Ta có thể xem:

```bash
ps -o pid,ppid,user,stat,cmd -p 4821
```

Expected invariant:

```text
PID của sleep = 4821
PPID = PID của shell hoặc process tạo nó
```

---

# 8. Process tree

Các process không tồn tại độc lập thành một danh sách phẳng.

Hãy tưởng tượng Linux process hierarchy:

```text
PID 1
systemd
│
├── sshd
│   └── sshd
│       └── bash
│           ├── ps
│           └── sleep
│
├── cron
│
└── application-service
    └── java
        ├── thread ...
        └── thread ...
```

Một câu hỏi vận hành rất mạnh là:

> Process này được process nào tạo ra?

Thay vì chỉ:

> Process này tên gì?

---

# 9. `pstree`

Nếu có `pstree`:

```bash
pstree
```

hoặc:

```bash
pstree -p
```

Ví dụ:

```text
systemd(1)
 └─sshd(901)
    └─sshd(4010)
       └─bash(4072)
          └─sleep(4821)
```

`-p` hiển thị PID.

Một số distro không cài sẵn `pstree`; nó thường đến từ package `psmisc`. **Không cần cài package ngay trong Module 1 nếu máy chưa có**, vì package management là Module 3.

Ta vẫn có thể dùng `ps`.

---

# 10. Supplementary — Process được tạo ra như thế nào?

Đây là phần cực kỳ hữu ích để hiểu Linux.

Mental model kinh điển là:

```text
parent process
      │
      │ fork()
      ↓
parent + child
             │
             │ execve()
             ↓
       new program image
```

## `fork()`

`fork()` tạo một child process bằng cách duplicate process gọi nó. Parent và child sau đó có memory space riêng. Linux dùng cơ chế copy-on-write để tránh copy toàn bộ memory ngay lập tức. Child có PID riêng và PPID của child trỏ về parent. :chatgpt-content-reference{index="4"}

Conceptually:

```text
bash PID 5000
     │
     │ fork()
     ↓
bash PID 5000
child PID 5001
```

Sau đó child có thể gọi:

```text
execve("/usr/bin/sleep", ...)
```

---

# 11. `execve()` không tạo thêm process mới

Đây là một lỗi hiểu rất phổ biến.

`execve()` không đơn giản là:

> “tạo process khác”.

Nó thay thế program image của process hiện tại bằng program mới.

Linux documentation ghi rõ rằng khi `execve()` thành công, text/data/BSS/stack của calling process được thay thế bởi executable mới. PID vẫn giữ nguyên qua `execve()`. :chatgpt-content-reference{index="5"}

Mental model:

```text
PID 5001
bash child
    │
    │ execve("/usr/bin/sleep")
    ↓
PID 5001
sleep
```

PID không đổi.

Program image đổi.

Đây là lý do process tree có thể hình thành từ:

```text
shell
 ↓ fork
child shell
 ↓ exec
java
```

---

# 12. Không phải mọi command đều tạo external process

Ví dụ:

```bash
cd /tmp
```

`cd` thường là shell builtin.

Tại sao?

Giả sử shell tạo child:

```text
bash parent
   │
   └── child
         cd /tmp
```

Child đổi directory:

```text
child cwd = /tmp
```

nhưng parent vẫn:

```text
parent cwd = /home/student
```

Sau đó child kết thúc.

Kết quả:

```text
bash vẫn ở /home/student
```

Vì vậy `cd` cần tác động vào shell hiện tại.

Bạn có thể kiểm tra:

```bash
type cd
```

Expected:

```text
cd is a shell builtin
```

Tương tự:

```bash
type export
```

thường cho:

```text
export is a shell builtin
```

Điều này sẽ cực kỳ quan trọng ở phần environment variables.

---

# 13. Process owner và credentials

Một process chạy trong security context nhất định.

Khi dùng:

```bash
ps -ef
```

hoặc:

```bash
ps aux
```

bạn thấy user sở hữu process.

Ví dụ:

```text
USER      PID  ...
root      901  /usr/sbin/sshd
postgres 1234  /usr/lib/postgresql/...
tomcat   2345  /usr/bin/java ...
```

Điều này trả lời câu hỏi:

> Process đang chạy với quyền của ai?

Và kéo theo:

```text
process đọc được file nào?
ghi được directory nào?
bind được resource nào?
signal được process nào?
```

Đây là cầu nối trực tiếp từ Day 3:

```text
users/groups/permissions
```

sang Day 4:

```text
process runtime identity
```

---

# 14. Quyền gửi signal cũng phụ thuộc identity

Không phải user nào cũng được:

```bash
kill <PID>
```

với bất kỳ process nào.

Theo Linux `kill(2)`, về cơ bản process gửi signal cần có privilege thích hợp hoặc UID phù hợp với target theo các rule được định nghĩa; privileged process có thể sử dụng capability `CAP_KILL`. :chatgpt-content-reference{index="6"}

Ví dụ user `student` thường có thể signal process của chính mình:

```text
student → student
```

nhưng không tự nhiên được phép:

```text
student → postgres
student → root
```

Nếu không có quyền, có thể gặp:

```text
Operation not permitted
```

Điều quan trọng:

```text
"không kill được"
```

không đồng nghĩa:

```text
"process bị treo"
```

Nó có thể đơn giản là vấn đề permission.

---

# 15. Process state

Một process tồn tại không có nghĩa là nó đang thực sự dùng CPU tại thời điểm bạn kiểm tra.

Process có thể đang:

```text
running
waiting
sleeping
stopped
zombie
```

`ps` của procps-ng định nghĩa các state phổ biến như sau. :chatgpt-content-reference{index="7"}

| State | Ý nghĩa                                     |
| ----- | ------------------------------------------- |
| `R`   | Running hoặc runnable                       |
| `S`   | Interruptible sleep                         |
| `D`   | Uninterruptible sleep, thường liên quan I/O |
| `T`   | Stopped bởi job-control signal              |
| `t`   | Stopped bởi debugger/tracing                |
| `Z`   | Zombie/defunct                              |
| `I`   | Idle kernel thread                          |
| `X`   | Dead; bình thường hầu như không thấy        |

Bạn không cần thuộc lòng mọi ký tự ngay lập tức. Nhưng phải nắm chắc:

```text
R
S
D
T
Z
```

---

# 16. State `R` — Running / Runnable

`R` không nhất thiết có nghĩa:

> CPU đang thực thi process này đúng ngay nanosecond bạn nhìn.

Nó có thể là:

```text
currently running
```

hoặc:

```text
runnable / ready to run
```

tức process đang chờ scheduler cấp CPU.

`top` cũng giải thích các task hiển thị `R` nên được hiểu là đang chạy hoặc ready-to-run. :chatgpt-content-reference{index="8"}

Ví dụ trên máy chỉ có vài CPU nhưng có hàng trăm process runnable:

```text
tất cả không thể thực thi đồng thời
```

Scheduler sẽ phân phối CPU time.

---

# 17. State `S` — Interruptible sleep

`S` rất thường gặp.

Ví dụ:

```bash
sleep 300
```

không sử dụng CPU liên tục 300 giây.

Nó chủ yếu đang chờ timer/event.

Bạn có thể thấy:

```text
STAT
S
```

`S` thường là state hoàn toàn bình thường.

Một server application cũng dành phần lớn thời gian ở trạng thái waiting:

```text
chờ network request
chờ timer
chờ data
chờ event
```

Vì vậy:

```text
S ≠ problem
```

---

# 18. State `D` — Uninterruptible sleep

`D` thường gắn với process đang chờ kernel operation, điển hình là I/O.

Ví dụ conceptual:

```text
process
  ↓
read storage
  ↓
kernel waits for I/O
  ↓
D state
```

Một `D` state ngắn có thể hoàn toàn bình thường.

Nhưng:

```text
process stuck in D rất lâu
```

có thể chỉ ra vấn đề như:

```text
storage issue
NFS/network filesystem issue
device problem
kernel/I/O path issue
```

Đây là điểm cần escalation thận trọng.

Đừng nhìn thấy `D` rồi lập tức:

```bash
kill -9 PID
```

Trong uninterruptible sleep, process có thể chưa xử lý termination cho tới khi kernel operation trả về.

Do đó:

```text
kill -9 không phải "nút thần kỳ"
```

---

# 19. State `T` — Stopped

Process vẫn tồn tại nhưng execution đang bị stop.

Ví dụ chúng ta sẽ thực hành:

```bash
sleep 300 &
PID=$!
kill -STOP "$PID"
```

Kiểm tra:

```bash
ps -p "$PID" -o pid,stat,cmd
```

Expected invariant:

```text
STAT bắt đầu bằng T
```

Sau đó:

```bash
kill -CONT "$PID"
```

state sẽ trở lại trạng thái runnable/sleeping tùy thời điểm.

---

# 20. State `Z` — Zombie

Đây là một trong những khái niệm fresher thường hiểu sai nhất.

Zombie **không phải process đang chạy ngầm và không chịu chết**.

Zombie đã **terminate execution rồi**.

Điều còn lại là một entry tối thiểu trong process table để parent có thể lấy termination information của child.

Linux `wait(2)` mô tả rằng child đã terminate nhưng parent chưa `wait()` sẽ trở thành zombie; kernel giữ lại PID, termination status và một số resource-usage information để parent thu thập sau. :chatgpt-content-reference{index="9"}

Mental model:

```text
parent
  │
  ├── creates child
  │
  ↓
child executes
  │
  ↓
child exits
  │
  ├── parent calls wait()
  │       ↓
  │     cleaned up
  │
  └── parent does NOT wait()
          ↓
        zombie
```

---

# 21. Không `kill -9` zombie

Nếu process đã là zombie:

```text
execution đã kết thúc
```

nên việc gửi thêm signal tới nó không giải quyết root cause.

Vấn đề thật sự là:

```text
parent chưa reap child
```

Remediation phụ thuộc tình huống.

Nếu chỉ có zombie ngắn ngủi:

```text
có thể parent sắp reap
```

Nếu zombie tích tụ:

```text
parent application có thể có bug
```

Bạn cần tìm parent:

```bash
ps -o pid,ppid,stat,cmd -p <zombie_pid>
```

Sau đó investigate parent.

Đây là khác biệt giữa:

```text
symptom
```

và:

```text
root cause
```

---

# 22. Một zombie có làm máy chết ngay không?

Thông thường một zombie đơn lẻ không tiêu thụ CPU như process sống và không giữ toàn bộ address space như trước.

Nhưng zombie vẫn giữ process-table information.

Linux documentation cảnh báo rằng nếu quá nhiều zombie không được reaped và process table cạn, hệ thống có thể không tạo thêm process mới. :chatgpt-content-reference{index="10"}

Do đó:

```text
1 zombie tạm thời
```

và:

```text
hàng nghìn zombie tăng liên tục
```

là hai mức severity hoàn toàn khác nhau.

---

# 23. `ps` — công cụ nền tảng

`ps` nghĩa là process status.

Nó cho một **snapshot**.

Official procps-ng documentation mô tả:

> `ps` hiển thị snapshot của active processes; nếu muốn cập nhật lặp lại thì dùng `top`. :chatgpt-content-reference{index="11"}

Đây là mental distinction:

```text
ps  → snapshot
top → continually refreshed view
```

---

# 24. `ps` mặc định

Chạy:

```bash
ps
```

thường chỉ thấy process liên quan đến shell/terminal hiện tại.

Ví dụ:

```text
    PID TTY          TIME CMD
   3201 pts/0    00:00:00 bash
   4521 pts/0    00:00:00 ps
```

Đừng kết luận:

> “Máy chỉ có hai process.”

Bạn chỉ đang dùng selection mặc định.

---

# 25. `ps -ef`

Một command rất phổ biến:

```bash
ps -ef
```

Conceptually:

```text
-e → select all processes
-f → full-format listing
```

Ví dụ:

```text
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 08:00 ?        00:00:02 /sbin/init
root         900       1  0 08:00 ?        00:00:00 sshd
student     4000     900  0 09:30 pts/0    00:00:00 bash
student     4821    4000  0 10:20 pts/0    00:00:00 sleep 300
```

Bạn nên đọc ít nhất:

```text
UID
PID
PPID
STIME
TTY
TIME
CMD
```

---

# 26. `ps aux`

Bạn cũng thường gặp:

```bash
ps aux
```

Đây là BSD-style syntax.

Output thường chứa:

```text
USER
PID
%CPU
%MEM
VSZ
RSS
TTY
STAT
START
TIME
COMMAND
```

Trong đó đặc biệt hữu ích cho Module 1:

```text
USER
PID
STAT
COMMAND
```

CPU/memory sẽ được dùng sâu hơn ở operations/monitoring.

Một note nhỏ nhưng hữu ích: procps-ng hỗ trợ nhiều style option, và documentation cảnh báo `ps -aux` có thể gây ambiguity với syntax khác; dạng thông dụng là:

```bash
ps aux
```

không phải:

````bash
ps -aux
``` :chatgpt-content-reference{index="12"}


---

# 27. Cách dùng `ps` tốt hơn cho troubleshooting

Thay vì luôn lấy hàng chục cột không cần thiết:

```bash
ps aux
````

hãy học custom output.

Ví dụ:

```bash
ps -eo pid,ppid,user,stat,lstart,etime,cmd
```

Ý nghĩa:

| Field    | Ý nghĩa         |
| -------- | --------------- |
| `pid`    | PID             |
| `ppid`   | Parent PID      |
| `user`   | Owner           |
| `stat`   | Process state   |
| `lstart` | Full start time |
| `etime`  | Elapsed time    |
| `cmd`    | Command         |

Đây là command rất mạnh cho investigation.

---

# 28. Quan sát một PID cụ thể

Nếu đã có PID:

```bash
PID=4821
```

thì:

```bash
ps -p "$PID" -o pid,ppid,user,stat,lstart,etime,cmd
```

Đây tốt hơn rất nhiều so với:

```bash
ps -ef | grep 4821
```

vì ta đã có identifier chính xác.

---

# 29. Tại sao `grep` có thể gây nhiễu?

Ví dụ:

```bash
ps -ef | grep java
```

có thể trả:

```text
tomcat   3000 ... java ...
student  5010 ... grep java
```

Bạn thấy chính command:

```text
grep java
```

xuất hiện.

Fresher đôi lúc tưởng:

```text
có hai Java process
```

trong khi chỉ có một.

`grep` vẫn hữu ích, nhưng cần biết limitation.

---

# 30. `pgrep`

Khi mục tiêu là tìm PID theo process name:

```bash
pgrep sleep
```

Hoặc tốt hơn:

```bash
pgrep -a sleep
```

`-a` hiển thị PID cùng full command line.

Ví dụ:

```text
4821 sleep 300
```

`pgrep` hiện hỗ trợ filter theo nhiều tiêu chí như effective UID, PPID, process group và full command line. :chatgpt-content-reference{index="13"}

---

# 31. `pgrep -f`

Mặc định matching chủ yếu dựa trên process name.

Nếu cần full command line:

```bash
pgrep -af 'myapp.jar'
```

`-f` yêu cầu match full command line. :chatgpt-content-reference{index="14"}

Điều này đặc biệt hữu ích cho Java vì nhiều process đều có executable:

```text
java
```

nhưng command line khác:

```text
java -jar app-a.jar
java -jar app-b.jar
```

---

# 32. Nhưng matching theo tên cũng có nguy hiểm

Đừng lập tức chạy:

```bash
pkill java
```

trên production server.

Bạn có thể terminate:

```text
app A
app B
Tomcat
monitoring agent
batch job
```

nếu chúng cùng là Java.

Workflow đúng hơn:

```text
find
 ↓
inspect
 ↓
confirm identity
 ↓
act
 ↓
verify
```

Ví dụ:

```bash
pgrep -af java
```

sau đó inspect PID chính xác:

```bash
ps -p "$PID" -o pid,ppid,user,stat,lstart,etime,cmd
```

rồi mới cân nhắc signal.

---

# 33. `top`

`top` cho một dynamic view:

```bash
top
```

Bạn sẽ thấy đại khái:

```text
PID
USER
PR
NI
VIRT
RES
SHR
S
%CPU
%MEM
TIME+
COMMAND
```

Ở Module 1, trọng tâm là:

```text
PID
USER
S
%CPU
%MEM
COMMAND
```

Không cần học mọi interactive key ngay.

Điểm cần hiểu:

```text
ps → evidence snapshot dễ copy/save
top → live observation
```

Trong incident, hai công cụ bổ sung cho nhau.

---

# 34. `/proc` — nhìn process qua kernel interface

Đây là phần giúp bạn vượt từ “biết command” sang “hiểu Linux”.

Linux mount pseudo-filesystem:

```text
/proc
```

Mỗi process thường có:

```text
/proc/<PID>/
```

Ví dụ PID `4821`:

```text
/proc/4821/
```

Linux man-pages xác nhận mỗi running process có numerical subdirectory được đặt tên theo PID. :chatgpt-content-reference{index="15"}

---

# 35. Khảo sát `/proc/<PID>`

Giả sử:

```bash
sleep 300 &
PID=$!
```

`$!` trong Bash là PID của background job gần nhất.

Kiểm tra:

```bash
echo "$PID"
```

Sau đó:

```bash
ls "/proc/$PID"
```

Bạn sẽ thấy rất nhiều entry.

Không cần hiểu tất cả.

Trong Day 4 Module 1, những entry rất hữu ích là:

```text
status
cmdline
exe
cwd
environ
fd/
```

---

# 36. `/proc/<PID>/status`

Chạy:

```bash
cat "/proc/$PID/status"
```

Ví dụ:

```text
Name:   sleep
State:  S (sleeping)
Tgid:   4821
Pid:    4821
PPid:   4000
Uid:    1000    1000    1000    1000
Gid:    1000    1000    1000    1000
Threads:        1
...
```

Ngay từ file này bạn lấy được:

```text
process name
state
PID
PPID
UID/GID
thread count
signal-related state
```

`/proc/<PID>/status` còn expose signal masks và capability-related information, mặc dù chúng ta chưa cần đào sâu toàn bộ trong Day 4. :chatgpt-content-reference{index="16"}

---

# 37. `/proc/<PID>/cmdline`

Chạy:

```bash
tr '\0' ' ' < "/proc/$PID/cmdline"
echo
```

Tại sao phải dùng:

```bash
tr '\0' ' '
```

?

Vì command-line arguments trong `/proc/<PID>/cmdline` thường được phân tách bằng NUL byte (`\0`), không phải newline/space thông thường. :chatgpt-content-reference{index="17"}

Ví dụ:

```text
sleep 300
```

Điều này hữu ích khi `ps` output bị truncate hoặc bạn muốn inspect trực tiếp process interface.

---

# 38. `/proc/<PID>/exe`

```bash
readlink "/proc/$PID/exe"
```

Ví dụ:

```text
/usr/bin/sleep
```

`/proc/<PID>/exe` là symbolic link tới pathname của executable đang được process sử dụng. :chatgpt-content-reference{index="18"}

Đây là distinction quan trọng:

```text
command line:
sleep 300
```

vs:

```text
actual executable:
/usr/bin/sleep
```

Sau này với Java:

```text
command:
java -jar /opt/app/app.jar
```

`exe` có thể là:

```text
/usr/lib/jvm/.../bin/java
```

---

# 39. `/proc/<PID>/cwd`

```bash
readlink "/proc/$PID/cwd"
```

Nó cho current working directory của process. Linux documentation định nghĩa `/proc/<PID>/cwd` là symbolic link tới current working directory. :chatgpt-content-reference{index="19"}

Tại sao điều này quan trọng?

Giả sử application mở:

```text
./config/application.properties
```

Relative path phụ thuộc vào:

```text
cwd
```

Process chạy tay từ:

```text
/home/student/app
```

có thể thành công.

Nhưng service chạy từ:

```text
/
```

có thể fail.

Đây chính là một nguyên nhân phổ biến của:

```text
"chạy command trực tiếp thì được,
 chạy bằng systemd thì fail"
```

mà Module 2 sẽ xử lý.

---

# 40. `/proc/<PID>/fd/`

Process dùng file descriptors để truy cập:

```text
files
pipes
sockets
devices
```

Bạn có thể xem:

```bash
ls -l "/proc/$PID/fd"
```

Linux định nghĩa `/proc/<PID>/fd/` chứa một entry cho mỗi open file descriptor; `0`, `1`, `2` thường lần lượt là standard input, output và error. :chatgpt-content-reference{index="20"}

Ví dụ:

```text
0 -> /dev/pts/0
1 -> /dev/pts/0
2 -> /dev/pts/0
```

Mental model:

```text
FD 0 → stdin
FD 1 → stdout
FD 2 → stderr
```

Đây sẽ cực kỳ quan trọng khi học:

```text
logs
redirection
network sockets
Tomcat
```

sau này.

---

# 41. Race conditions khi inspect process

Một điều thực tế:

Giả sử bạn chạy:

```bash
ps -p 4821
```

thấy process.

Ngay sau đó:

```bash
cat /proc/4821/status
```

có thể nhận:

```text
No such file or directory
```

Không nhất thiết hệ thống có lỗi.

Process có thể đơn giản đã exit giữa hai command.

Process state là dynamic.

Do đó operations evidence phải hiểu yếu tố thời gian:

```text
command
timestamp
observation
```

---

# 42. Foreground process và background process

Khi chạy:

```bash
sleep 300
```

terminal bị command chiếm foreground.

Shell chờ process hoàn tất trước khi trả prompt.

Nếu chạy:

```bash
sleep 300 &
```

Bash đặt job vào background và trả prompt ngay.

Ví dụ:

```text
[1] 4821
```

Trong đó:

```text
1    = shell job number
4821 = PID
```

Đây là hai identifier khác nhau.

---

# 43. Job ID không phải PID

Shell có thể hiển thị:

```text
[1] 4821
```

Và cho phép:

```bash
fg %1
```

`%1` là:

```text
job 1
```

không phải:

```text
PID 1
```

Đây là distinction quan trọng.

---

# 44. Process group — Supplementary

Một shell job có thể gồm nhiều process.

Ví dụ pipeline:

```bash
command1 | command2 | command3
```

có thể tạo:

```text
process A
process B
process C
```

nhưng shell coi cả pipeline là một job/process group.

Linux process groups và sessions được thiết kế để hỗ trợ shell job control. :chatgpt-content-reference{index="21"}

Mental model:

```text
Terminal
   │
Session
   │
Process Group / Job
   ├── process A
   ├── process B
   └── process C
```

---

# 45. `Ctrl+C` không phải magic keyboard shortcut

Khi bạn nhấn:

```text
Ctrl+C
```

terminal driver thường tạo:

```text
SIGINT
```

và gửi tới foreground process group. Linux job-control documentation mô tả terminal-generated signals theo process group như vậy. :chatgpt-content-reference{index="22"}

Do đó:

```text
Ctrl+C
```

không trực tiếp có nghĩa:

> kill PID X.

Nó có nghĩa gần hơn với:

> gửi SIGINT cho foreground process group của terminal.

---

# 46. `Ctrl+Z`

Thông thường:

```text
Ctrl+Z
```

gửi:

```text
SIGTSTP
```

đến foreground process group.

Process bị stopped.

Ví dụ:

```bash
sleep 300
```

nhấn `Ctrl+Z`.

Shell có thể báo:

```text
[1]+  Stopped                 sleep 300
```

Sau đó:

```bash
jobs
```

có thể thấy:

```text
[1]+  Stopped  sleep 300
```

Bạn có thể đưa nó trở lại foreground:

```bash
fg %1
```

hoặc background:

```bash
bg %1
```

---

# 47. Signals — Mental Model

Signal là một cơ chế notification/control giữa kernel/processes và processes.

Mental model đơn giản:

```text
sender
  │
  │ signal
  ↓
target process
  │
  ├── default action
  ├── custom handler
  ├── ignore
  └── possibly blocked/pending
```

Một signal **không phải command text được chạy bên trong process**.

Nó là một asynchronous notification có semantics được kernel định nghĩa.

Linux hỗ trợ standard signals và realtime signals; mỗi signal có disposition xác định hành vi khi signal được delivered.

---

# 48. `kill` là tên dễ gây hiểu nhầm

Command:

```bash
kill
```

không có nghĩa:

```text
kill luôn luôn terminate process
```

Bản chất của nó là:

```text
send a signal
```

Ví dụ:

```bash
kill -STOP "$PID"
```

không terminate process.

Nó dừng execution.

```bash
kill -CONT "$PID"
```

không terminate process.

Nó tiếp tục process.

Official `kill` documentation xác nhận command gửi signal tới process/process group; nếu không chỉ định signal thì mặc định là `SIGTERM`.

---

# 49. Các signal quan trọng cho fresher admin

| Signal      | Tên thường dùng | Ý nghĩa vận hành                                                                      |
| ----------- | --------------- | ------------------------------------------------------------------------------------- |
| `SIGTERM`   | `TERM`          | Yêu cầu process terminate, cho phép graceful handling                                 |
| `SIGKILL`   | `KILL`          | Kernel terminate process; process không thể catch/block/ignore                        |
| `SIGINT`    | `INT`           | Interrupt; thường từ `Ctrl+C`                                                         |
| `SIGHUP`    | `HUP`           | Historically terminal hangup; nhiều daemon dùng cho reload nhưng application-specific |
| `SIGSTOP`   | `STOP`          | Stop process; không thể catch/block/ignore                                            |
| `SIGCONT`   | `CONT`          | Continue stopped process                                                              |
| `SIGTSTP`   | `TSTP`          | Terminal stop, thường từ `Ctrl+Z`                                                     |
| `SIGCHLD`   | `CHLD`          | Thông báo parent về state change của child                                            |
| `SIGUSR1/2` | `USR1/USR2`     | Application-defined                                                                   |

Linux documentation xác nhận `SIGKILL` và `SIGSTOP` không thể bị caught, blocked hoặc ignored.

---

# 50. Xem danh sách signals

```bash
kill -l
```

Bạn có thể thấy:

```text
HUP
INT
QUIT
KILL
TERM
STOP
CONT
...
```

Tên signal rõ nghĩa hơn các magic number.

Nên ưu tiên:

```bash
kill -TERM "$PID"
```

thay vì học thuộc:

```bash
kill -15 "$PID"
```

và:

```bash
kill -KILL "$PID"
```

thay vì reflexively:

```bash
kill -9 "$PID"
```

Signal numbers có thể khác giữa architecture cho một số signals, trong khi symbolic names biểu đạt intent rõ hơn.

---

# 51. SIGTERM — signal ưu tiên cho shutdown

Nếu chạy:

```bash
kill "$PID"
```

signal mặc định thường là:

```text
SIGTERM
```

Tương đương:

```bash
kill -TERM "$PID"
```

SIGTERM cho application cơ hội xử lý graceful shutdown nếu application có handler phù hợp.

Conceptually application có thể:

```text
receive SIGTERM
     ↓
stop accepting new work
     ↓
finish in-flight work
     ↓
flush data/log
     ↓
close sockets/files
     ↓
release resources
     ↓
exit
```

Điều đó không có nghĩa mọi application chắc chắn làm tất cả các bước này.

Nó phụ thuộc application implementation.

---

# 52. SIGTERM có thể bị xử lý khác nhau

Process có thể:

```text
accept default termination
install handler
ignore SIGTERM
temporarily block SIGTERM
```

Do đó:

```bash
kill -TERM PID
```

không đảm bảo process biến mất ngay lập tức.

Administrator phải:

```text
signal
 ↓
wait appropriate interval
 ↓
verify
```

Không được:

```text
kill command returned 0
therefore app cleanly stopped
```

Hai kết luận đó không tương đương.

---

# 53. SIGKILL

```bash
kill -KILL "$PID"
```

yêu cầu kernel terminate target mà không cho process user-space xử lý signal.

Process không thể catch, block hay ignore `SIGKILL`.

Điều này rất mạnh nhưng cũng phá khả năng graceful cleanup của application.

Potential consequences phụ thuộc application:

```text
temporary files not cleaned
transactions/work interrupted
buffers not flushed by application
lock/state cleanup not executed
partial work
harder postmortem evidence
```

Không phải lúc nào những hậu quả đó cũng xảy ra, nhưng đó là lý do `SIGKILL` là escalation, không phải default.

---

# 54. Operational sequence đúng

Thay vì:

```bash
kill -9 1234
```

hãy nghĩ:

```text
1. Confirm target identity
2. Check owner / command / state
3. Understand whether process is service-managed
4. Use supported service/application stop mechanism when available
5. Otherwise request graceful termination
6. Verify
7. Investigate why it did not stop
8. Escalate to SIGKILL only when justified
9. Verify again
10. Preserve evidence
```

Ở Module 2, nếu process thuộc systemd service, normal operational interface thường sẽ là:

```bash
systemctl stop ...
```

thay vì bypass service manager bằng raw `kill`.

---

# 55. `SIGSTOP` và `SIGCONT`

Đây là cặp signal rất tốt để học vì chúng không cần destroy process.

Start:

```bash
sleep 300 &
PID=$!
```

Check:

```bash
ps -p "$PID" -o pid,stat,cmd
```

Expected:

```text
S
```

Stop:

```bash
kill -STOP "$PID"
```

Check:

```bash
ps -p "$PID" -o pid,stat,cmd
```

Expected state:

```text
T
```

Continue:

```bash
kill -CONT "$PID"
```

Check:

```bash
ps -p "$PID" -o pid,stat,cmd
```

Expected:

```text
S
```

Cuối cùng cleanup:

```bash
kill -TERM "$PID"
```

---

# 56. SIGSTOP khác SIGTSTP

Cả hai đều có thể stop process, nhưng có khác biệt quan trọng.

`SIGSTOP`:

```text
cannot be caught
cannot be blocked
cannot be ignored
```

`SIGTSTP` là job-control signal thường gắn với:

```text
Ctrl+Z
```

và process có thể xử lý nó theo rules thông thường hơn.

Đây giống distinction:

```text
SIGKILL vs SIGTERM
```

ở khía cạnh:

```text
forced kernel-level behavior
vs
application-manageable behavior
```

---

# 57. SIGHUP không đồng nghĩa “reload”

Một anti-pattern:

```text
Muốn reload bất kỳ service nào
→ kill -HUP PID
```

Không đúng.

Historically `SIGHUP` liên quan hangup của controlling terminal. Một số daemon chọn convention:

```text
SIGHUP → reload configuration
```

nhưng đó là **application-specific contract**.

Bạn phải kiểm tra documentation của application/service.

Không được assume:

```text
HUP = reload cho mọi process
```

---

# 58. SIGUSR1 và SIGUSR2

Hai signal này được dành cho application-defined behavior.

Một application có thể định nghĩa:

```text
USR1 → dump thread state
```

application khác:

```text
USR1 → reopen logs
```

application khác nữa không cài handler:

```text
USR1 → default termination
```

Linux mặc định liệt kê `SIGUSR1`/`SIGUSR2` với termination disposition nếu application không thay đổi nó.

Do đó tuyệt đối không:

```bash
kill -USR1 <production PID>
```

chỉ vì bạn thấy tutorial nào đó.

Phải đọc application documentation.

---

# 59. `kill -0`

Một trick vận hành hữu ích:

```bash
kill -0 "$PID"
```

Signal `0` không thực sự gửi signal; kernel vẫn thực hiện existence/permission checks.

Ví dụ:

```bash
kill -0 "$PID"
echo $?
```

Nếu có quyền và PID tồn tại:

```text
0
```

Nhưng hãy cẩn thận:

```text
PID exists
```

không đồng nghĩa:

```text
application healthy
```

Nó chỉ là một check nhỏ.

---

# 60. `pkill`

`pkill` tìm process theo criteria rồi gửi signal.

Ví dụ:

```bash
pkill -TERM sleep
```

Nhưng đây rộng hơn:

```bash
kill -TERM "$PID"
```

Nếu có 20 process tên `sleep`:

```text
pkill sleep
```

có thể target tất cả process matching.

Best practice trong lab/operations:

```bash
pgrep -a sleep
```

trước.

Sau khi chắc chắn criteria:

```bash
pkill ...
```

`pgrep`/`pkill` documentation xác nhận `pkill` mặc định gửi `SIGTERM` tới các process match criteria.

---

# 61. Environment Variables — bắt đầu từ shell variable

Đây là phần thứ ba bắt buộc của Module 1.

Trước hết, hãy phân biệt:

```text
shell variable
```

và:

```text
environment variable
```

Trong Bash:

```bash
DAY4_MODE=lab
```

tạo một shell parameter/variable.

Kiểm tra:

```bash
echo "$DAY4_MODE"
```

Output:

```text
lab
```

Nhưng điều đó chưa chắc child process nhận được nó.

---

# 62. `export`

Khi:

```bash
export DAY4_MODE
```

Bash đánh dấu variable để truyền cho các command/process được thực thi sau đó.

Hoặc viết một bước:

```bash
export DAY4_MODE=lab
```

GNU Bash documentation xác nhận `export` đánh dấu variable để truyền cho subsequently executed commands trong environment.

Mental model:

```text
Bash variable
DAY4_MODE=lab
      │
      │ export
      ↓
exported shell variable
      │
      │ execute child
      ↓
child environment
DAY4_MODE=lab
```

---

# 63. Environment bản chất là gì?

Ở mức process interface, environment là tập các string kiểu:

```text
name=value
```

Ví dụ:

```text
PATH=/usr/local/bin:/usr/bin:/bin
HOME=/home/student
LANG=en_US.UTF-8
APP_ENV=production
JAVA_HOME=/usr/lib/jvm/java-17-openjdk
```

Linux `environ(7)` mô tả environment là array các `name=value` strings được cung cấp cho process khi program được start qua `execve()`. Child được tạo bằng `fork()` thừa hưởng một copy của environment của parent.

---

# 64. Parent → child inheritance

Giả sử shell:

```bash
export APP_ENV=dev
```

Sau đó:

```bash
bash -c 'echo "$APP_ENV"'
```

Output:

```text
dev
```

Mental model:

```text
parent Bash
APP_ENV=dev
      │
      │ create/exec child
      ↓
child Bash
APP_ENV=dev
```

Child nhận environment.

---

# 65. Nhưng child không sửa được parent environment

Start:

```bash
export APP_ENV=parent
```

Check:

```bash
echo "$APP_ENV"
```

Output:

```text
parent
```

Run:

```bash
bash -c 'export APP_ENV=child; echo "child sees: $APP_ENV"'
```

Output:

```text
child sees: child
```

Sau đó:

```bash
echo "parent sees: $APP_ENV"
```

Output vẫn:

```text
parent sees: parent
```

Mental model:

```text
parent
APP_ENV=parent
        │
        ↓ inheritance
child
APP_ENV=parent
        │
        ↓ change
child
APP_ENV=child
```

nhưng:

```text
parent
APP_ENV=parent
```

không đổi.

Đây là nền tảng giải thích rất nhiều hành vi Linux.

---

# 66. Shell variable không exported

Thử:

```bash
DAY4_LOCAL=hidden
```

Trong shell:

```bash
echo "$DAY4_LOCAL"
```

Output:

```text
hidden
```

Nhưng:

```bash
bash -c 'echo "$DAY4_LOCAL"'
```

thường chỉ in newline trống.

Vì:

```text
DAY4_LOCAL
```

chưa exported.

Sau đó:

```bash
export DAY4_LOCAL
```

và:

```bash
bash -c 'echo "$DAY4_LOCAL"'
```

Output:

```text
hidden
```

Đây là một bài kiểm tra bạn cần nắm cực chắc.

---

# 67. `printenv`

Để xem environment:

```bash
printenv
```

Hoặc một variable:

```bash
printenv PATH
```

Ví dụ:

```text
/usr/local/bin:/usr/bin:/bin
```

Nếu:

```bash
DAY4_LOCAL=test
```

nhưng chưa export:

```bash
printenv DAY4_LOCAL
```

có thể không output gì.

Sau:

```bash
export DAY4_LOCAL
```

thì:

```bash
printenv DAY4_LOCAL
```

sẽ thấy:

```text
test
```

---

# 68. `env`

Command:

```bash
env
```

cũng có thể hiển thị environment.

Nhưng `env` đặc biệt hữu ích để chạy command với environment được sửa tạm thời.

Ví dụ:

```bash
env APP_ENV=test bash -c 'echo "$APP_ENV"'
```

Output:

```text
test
```

Sau đó parent:

```bash
echo "${APP_ENV:-not-set}"
```

không nhất thiết bị thay đổi.

---

# 69. Temporary environment assignment

Bash cho phép:

```bash
APP_ENV=test command
```

Ví dụ:

```bash
APP_ENV=test bash -c 'echo "$APP_ENV"'
```

Child thấy:

```text
test
```

nhưng assignment này không cần persist vào shell hiện tại.

Đây là pattern cực kỳ hữu ích khi test:

```text
same executable
different environment
```

---

# 70. `unset`

Để xóa variable khỏi shell:

```bash
unset APP_ENV
```

Kiểm tra:

```bash
echo "${APP_ENV:-not-set}"
```

Output:

```text
not-set
```

Nếu variable đã exported, việc `unset` cũng làm nó không còn được truyền cho child tiếp theo.

---

# 71. `export -n`

Trong Bash, có thể:

```bash
export -n APP_ENV
```

Điều này bỏ export attribute nhưng không nhất thiết xóa shell variable.

Ví dụ:

```bash
APP_ENV=lab
export APP_ENV
export -n APP_ENV
```

Sau đó:

```bash
echo "$APP_ENV"
```

vẫn có thể:

```text
lab
```

nhưng child mới không còn nhận nó như exported environment variable. GNU Bash documents `export -n` như việc unexport name.

---

# 72. `set` khác `env`/`printenv`

Trong Bash:

```bash
set
```

có thể hiển thị rất nhiều shell state:

```text
shell variables
functions
...
```

Trong khi:

```bash
env
```

hoặc:

```bash
printenv
```

thích hợp hơn khi câu hỏi là:

> environment nào được exported?

Do đó để troubleshoot environment, đừng assume:

```text
set output = process environment
```

---

# 73. Variable name là case-sensitive

Ví dụ:

```bash
export APP_ENV=dev
export app_env=test
```

Đây là hai name khác nhau:

```bash
echo "$APP_ENV"
```

Output:

```text
dev
```

```bash
echo "$app_env"
```

Output:

```text
test
```

Linux environment convention là case-sensitive.

Convention thường dùng:

```text
UPPER_CASE
```

nhưng đó là convention, không phải mọi variable bắt buộc viết hoa.

---

# 74. Quoting rất quan trọng

Không nên:

```bash
APP_NAME=Java Fresher App
```

Shell sẽ parse thành nhiều token.

Đúng:

```bash
APP_NAME="Java Fresher App"
```

Check:

```bash
echo "$APP_NAME"
```

Output:

```text
Java Fresher App
```

Khi sử dụng variable, practice tốt:

```bash
"$APP_NAME"
```

thay vì:

```bash
$APP_NAME
```

để tránh word splitting/globbing ngoài ý muốn.

Đây nối lại command safety từ Day 2.

---

# 75. Một số environment variables quan trọng

| Variable                       | Vai trò điển hình                                 |
| ------------------------------ | ------------------------------------------------- |
| `PATH`                         | Search path cho executable                        |
| `HOME`                         | Home-directory context                            |
| `USER`                         | User-name information trong nhiều shell/session   |
| `PWD`                          | Working-directory information                     |
| `LANG`                         | Locale                                            |
| `LC_*`                         | Locale category overrides                         |
| `TZ`                           | Timezone behavior của một số program/library      |
| `JAVA_HOME`                    | Convention chỉ Java installation                  |
| Application-specific variables | Config như profile, ports, URLs, feature settings |

Không nên xem tất cả những variable này như authoritative security identity.

Ví dụ:

```bash
USER=hacker
```

không biến process thành Linux user `hacker`.

Security identity đến từ process credentials, UID/GID, capabilities…, không phải chỉ string `$USER`.

---

# 76. `PATH`

`PATH` cực kỳ quan trọng.

Ví dụ:

```bash
echo "$PATH"
```

có thể cho:

```text
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

Khi bạn chạy:

```bash
java
```

shell có thể tìm executable qua các directory trong `PATH`.

Bạn có thể kiểm tra:

```bash
command -v java
```

Ví dụ:

```text
/usr/bin/java
```

---

# 77. Vì sao PATH gây lỗi “works manually, fails as service”?

Terminal của bạn có thể có:

```text
PATH=/home/student/bin:/usr/local/bin:/usr/bin:/bin
```

Trong khi service environment có thể chỉ:

```text
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
```

Nếu application script dựa vào một executable chỉ có trong:

```text
/home/student/bin
```

thì:

```text
interactive shell → success
service → command not found
```

Đây là lý do operations tốt thường ưu tiên absolute paths cho critical service commands khi thích hợp.

Module 2 sẽ quay lại vấn đề này với `ExecStart=`.

---

# 78. `JAVA_HOME`

Ví dụ:

```bash
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```

Một application/build tool có thể dùng:

```text
JAVA_HOME
```

để tìm Java installation.

Nhưng cần hiểu:

```text
JAVA_HOME
```

và:

```text
PATH
```

là hai khái niệm khác nhau.

Có thể:

```bash
java -version
```

chạy được vì `/usr/bin/java` ở `PATH`, trong khi:

```bash
echo "$JAVA_HOME"
```

trống.

Ngược lại `JAVA_HOME` được set nhưng `PATH` không chứa `$JAVA_HOME/bin` thì command:

```bash
java
```

không nhất thiết được shell tìm ra qua `JAVA_HOME`.

---

# 79. `/proc/<PID>/environ`

Ta có thể inspect environment process bằng:

```bash
tr '\0' '\n' < "/proc/$PID/environ"
```

Tương tự `cmdline`, environment entries dùng NUL separators.

Ví dụ:

```text
HOME=/home/student
PATH=/usr/bin:/bin
DAY4_MODE=lab
```

Linux documentation mô tả `/proc/<PID>/environ` chứa environment ban đầu được thiết lập khi program được started qua `execve()`, với entries NUL-separated.

---

# 80. Một caveat rất quan trọng của `/proc/<PID>/environ`

Đây là chi tiết nhiều tutorial bỏ qua.

Theo Linux man-page, `/proc/<PID>/environ` phản ánh **initial environment** khi process image được started qua `execve()`. Nếu program sau đó tự thay đổi environment trong process memory, file này không nhất thiết phản ánh những thay đổi đó.

Vì vậy không được nói tuyệt đối:

```text
/proc/PID/environ luôn luôn là current live application environment
```

Cách nói chính xác hơn:

> Nó là một nguồn evidence rất hữu ích về environment process được start với, nhưng có technical caveats.

Đây là mức hiểu production-quality.

---

# 81. Permission khi đọc `/proc/<PID>`

Bạn có thể thử:

```bash
cat /proc/1/environ
```

và gặp:

```text
Permission denied
```

Đây không nhất thiết là lỗi.

Linux áp dụng access checks cho `/proc/<PID>`; procfs còn có thể được mount với options như `hidepid` để hạn chế process information giữa users.

Điều này phục vụ security.

---

# 82. Environment variables và secrets

Đừng nghĩ:

```text
environment variable = private secret vault
```

Environment có thể bị nhìn thấy trong nhiều context tùy permissions/debugging/runtime.

Ví dụ:

```bash
export DB_PASSWORD='super-secret'
```

rồi:

```bash
printenv
```

sẽ expose nó ngay trong terminal output.

Hoặc nếu bạn ghi:

```bash
printenv > incident-evidence.txt
```

bạn có thể vô tình đưa:

```text
password
token
API key
credential
```

vào evidence package.

Operations rule:

```text
collect enough evidence
but redact/protect secrets
```

Tuyệt đối không paste credentials vào bài tập của chúng ta.

---

# 83. Command-line arguments cũng có security implication

Tương tự, đừng làm:

```bash
some-command --password mySecret123
```

trừ khi tool/documentation thực sự yêu cầu và cơ chế đó được đánh giá an toàn.

Command line có thể xuất hiện trong:

```bash
ps
```

và:

```text
/proc/<PID>/cmdline
```

Linux explicitly exposes process command line qua procfs.

Sau này khi học secrets handling, chúng ta sẽ chọn cơ chế phù hợp từng application/platform.

---

# 84. Threads vs Processes — Supplementary

Một process có thể có nhiều threads.

Conceptually:

```text
Process
│
├── common address space
├── common open resources
├── process-wide environment
│
├── Thread 1
├── Thread 2
├── Thread 3
└── Thread 4
```

Ví dụ Java application rất thường multi-threaded.

`ps` có thể inspect threads:

```bash
ps -L -p "$PID"
```

hoặc:

```bash
ps -T -p "$PID"
```

procps-ng documents `-L`/`-T` thread-display modes.

Day 4 không yêu cầu thread diagnosis sâu, nhưng khi đến Java/Tomcat, bạn cần nhớ:

```text
one Java PID
```

có thể chứa:

```text
hundreds of Java threads
```

---

# 85. Process lifetime

Mental model đầy đủ hơn:

```text
creation
   ↓
initialization
   ↓
running/runnable
   ↕
waiting/sleeping
   ↕
stopped/continued
   ↓
termination
   ↓
parent collects status
   ↓
fully reaped
```

Hoặc exceptional:

```text
termination
   ↓
parent does not collect status
   ↓
zombie
```

Điều này giải thích phần lớn state transitions bạn cần biết ở fresher level.

---

# 86. Process exit và signal termination

Một process có thể kết thúc vì:

```text
normal application exit
application error
received terminating signal
uncaught fatal condition
resource/system failure
```

Day 6 sẽ đào sâu **exit codes** theo đúng syllabus. Trong Module 1, chỉ cần hiểu:

```text
process disappeared
```

chưa đủ để biết:

```text
why
```

Bạn phải tìm evidence.

Sau này:

```text
systemd status
journalctl
exit status
application logs
```

sẽ nối câu chuyện đó.

---

# 87. Anti-pattern: “PID tồn tại = app healthy”

Ví dụ:

```bash
pgrep -f app.jar
```

trả một PID.

Điều duy nhất bạn đã chứng minh chắc chắn là:

```text
một process matching criteria tồn tại tại thời điểm kiểm tra
```

Bạn chưa chứng minh:

```text
port listening
HTTP works
DB reachable
business function works
configuration correct
application ready
```

Điều này sẽ trở thành:

```text
process check
service check
port check
application endpoint check
dependency check
```

trong các Day tiếp theo.

---

# 88. Anti-pattern: “không thấy process = application chưa bao giờ start”

Process có thể:

```text
start
fail instantly
exit
```

Khi bạn chạy:

```bash
ps
```

5 giây sau:

```text
không còn PID
```

Điều đó không chứng minh application không từng start.

Có thể nó đã start rồi crash ngay.

Module 2 sẽ dạy bạn tìm:

```text
service state
exit reason
journal evidence
```

---

# 89. Anti-pattern: “kill command thành công = process đã cleanly shutdown”

`kill` success chủ yếu nói rằng operation gửi signal thành công theo semantics của system call/tool.

Bạn vẫn phải:

```bash
ps -p "$PID"
```

hoặc equivalent để xác minh.

Thậm chí sau khi PID biến mất:

```text
application cleanup có clean hay không
```

còn cần application-specific evidence.

---

# 90. Anti-pattern: `kill -9` đầu tiên

Đây là anti-pattern quan trọng nhất phần signals.

Sai mental model:

```text
problem?
→ kill -9
```

Đúng hơn:

```text
observe
→ identify
→ understand manager/context
→ graceful stop
→ verify
→ investigate
→ force only if required
```

---

# 91. Anti-pattern: signal process không thuộc scope

Ví dụ:

```bash
sudo kill -9 <random PID>
```

là thao tác có blast radius.

PID có thể thuộc:

```text
database
SSH
monitoring
networking
critical system daemon
```

Trong lab này, chúng ta **chỉ signal process do chính bạn tạo ra**.

Không dùng `sudo kill`.

---

# 92. Anti-pattern: Environment của shell = environment của service

Bạn chạy:

```bash
export APP_MODE=prod
```

rồi nghĩ:

```text
mọi process trên máy bây giờ đều có APP_MODE=prod
```

Hoàn toàn sai.

Environment có process scope/inheritance.

Conceptually:

```text
your Bash
APP_MODE=prod
   │
   └── children launched from this environment
```

Không tự động inject vào:

```text
existing processes
other SSH sessions
other users
systemd services
```

Đây sẽ là trọng tâm khi sang Module 2.

---

# 93. Anti-pattern: Parent environment thay đổi child đã chạy

Giả sử:

```bash
export APP_MODE=old
sleep 300 &
PID=$!
```

Sau đó parent chạy:

```bash
export APP_MODE=new
```

Không được assume process `sleep` đang chạy tự động nhận:

```text
APP_MODE=new
```

Environment inheritance xảy ra khi child được launched/execed.

Nó không phải live shared configuration channel giữa parent và child.

Để process nhận configuration mới, application phải có cơ chế cụ thể như:

```text
restart
reload
config API
signal handler
watch file
```

tùy application.

---

# 94. Guided Lab — Process, Signal và Environment

Lab này không cần root và không thay đổi system configuration.

**Safety rule:** chỉ thao tác với process mà bạn tự tạo trong lab. Không signal process hệ thống, SSH, database hay Java application thật.

Assumption:

```text
Linux
Bash
procps tools: ps
standard kill builtin/tool
/proc mounted
```

`pgrep`, `top`, `pstree` có thể phụ thuộc package availability.

---

# 95. Lab Bước 1 — Xác nhận shell hiện tại

Chạy:

```bash
echo "Shell PID: $$"
```

Ví dụ:

```text
Shell PID: 7312
```

Sau đó:

```bash
ps -p $$ -o pid,ppid,user,stat,lstart,etime,cmd
```

Quan sát:

```text
PID
PPID
USER
STAT
CMD
```

Expected invariant:

```text
PID phải bằng $$
```

Đây là evidence đầu tiên.

---

# 96. Lab Bước 2 — Tạo child process

Chạy:

```bash
sleep 600 &
```

Bash có thể in:

```text
[1] 7450
```

Lưu PID:

```bash
LAB_PID=$!
```

Check:

```bash
echo "$LAB_PID"
```

Sau đó:

```bash
ps -p "$LAB_PID" -o pid,ppid,user,stat,lstart,etime,cmd
```

Bạn nên thấy gần giống:

```text
PID   PPID USER     STAT STARTED                     ELAPSED CMD
7450  7312 student  S    Tue Sep 29 16:... 2026       00:05 sleep 600
```

Expected invariants:

```text
process exists
PID = $LAB_PID
command = sleep 600
user = current user
PPID gần như shell PID trong case đơn giản này
state thường S
```

---

# 97. Lab Bước 3 — Chứng minh parent-child

Run:

```bash
echo "Shell: $$"
echo "Child: $LAB_PID"
```

Sau đó:

```bash
ps -o pid,ppid,user,stat,cmd -p "$$", "$LAB_PID"
```

Nếu implementation của `ps` không chấp nhận syntax đó, dùng:

```bash
ps -p "$$" -p "$LAB_PID" -o pid,ppid,user,stat,cmd
```

Expected relationship:

```text
bash PID 7312
  ↓
sleep PID 7450
PPID=7312
```

---

# 98. Lab Bước 4 — Quan sát `/proc`

```bash
test -d "/proc/$LAB_PID" && echo "proc entry exists"
```

Expected:

```text
proc entry exists
```

Check state:

```bash
grep -E '^(Name|State|Pid|PPid|Uid|Gid|Threads):' "/proc/$LAB_PID/status"
```

Example output:

```text
Name:   sleep
State:  S (sleeping)
Pid:    7450
PPid:   7312
Uid:    1000    1000    1000    1000
Gid:    1000    1000    1000    1000
Threads:        1
```

---

# 99. Lab Bước 5 — Executable, cwd và command line

Executable:

```bash
readlink "/proc/$LAB_PID/exe"
```

Expected dạng:

```text
/usr/bin/sleep
```

Current working directory:

```bash
readlink "/proc/$LAB_PID/cwd"
```

Expected:

```text
directory mà shell đã đứng khi start sleep
```

Command line:

```bash
tr '\0' ' ' < "/proc/$LAB_PID/cmdline"
echo
```

Expected:

```text
sleep 600
```

Bạn vừa chứng minh ba thứ khác nhau:

```text
Executable → /usr/bin/sleep
Arguments  → sleep 600
CWD        → current directory at launch
```

---

# 100. Lab Bước 6 — Signal STOP

Trước:

```bash
ps -p "$LAB_PID" -o pid,stat,cmd
```

Sau đó:

```bash
kill -STOP "$LAB_PID"
```

Recheck:

```bash
ps -p "$LAB_PID" -o pid,stat,cmd
```

Expected:

```text
STAT bắt đầu T
```

Điểm cần hiểu:

```text
process vẫn tồn tại
```

nhưng:

```text
execution stopped
```

---

# 101. Lab Bước 7 — Signal CONT

```bash
kill -CONT "$LAB_PID"
```

Verify:

```bash
ps -p "$LAB_PID" -o pid,stat,cmd
```

Expected thường:

```text
S
```

Process tiếp tục chờ timer.

Đây là bằng chứng thực nghiệm rằng:

```text
kill command ≠ always kill process
```

---

# 102. Lab Bước 8 — `kill -0`

```bash
kill -0 "$LAB_PID"
echo "rc=$?"
```

Expected:

```text
rc=0
```

Tại thời điểm đó PID tồn tại và bạn được phép signal nó.

Nhắc lại:

```text
rc=0
```

không phải health check của application.

---

# 103. Lab Bước 9 — SIGTERM

Trước tiên collect evidence:

```bash
ps -p "$LAB_PID" -o pid,ppid,user,stat,lstart,etime,cmd
```

Then:

```bash
kill -TERM "$LAB_PID"
```

Verify:

```bash
ps -p "$LAB_PID" -o pid,stat,cmd
```

Expected:

```text
không còn process
```

Bash cũng có thể báo:

```text
[1]+  Terminated              sleep 600
```

Check:

```bash
test -d "/proc/$LAB_PID" && echo exists || echo gone
```

Expected:

```text
gone
```

---

# 104. Lab Bước 10 — Shell variable vs exported environment

Set shell-only variable:

```bash
DAY4_LOCAL="shell-only"
```

Set exported variable:

```bash
export DAY4_EXPORTED="visible-to-child"
```

Parent:

```bash
printf 'local=%s\n' "$DAY4_LOCAL"
printf 'exported=%s\n' "$DAY4_EXPORTED"
```

Expected:

```text
local=shell-only
exported=visible-to-child
```

Child:

```bash
bash -c '
printf "child local=<%s>\n" "$DAY4_LOCAL"
printf "child exported=<%s>\n" "$DAY4_EXPORTED"
'
```

Expected invariant:

```text
DAY4_LOCAL không được inherited
DAY4_EXPORTED được inherited
```

---

# 105. Lab Bước 11 — Export local variable

```bash
export DAY4_LOCAL
```

Run child again:

```bash
bash -c '
printf "child local=<%s>\n" "$DAY4_LOCAL"
printf "child exported=<%s>\n" "$DAY4_EXPORTED"
'
```

Bây giờ child phải thấy cả hai.

Đây là core skill bắt buộc.

---

# 106. Lab Bước 12 — Child không sửa parent

Parent:

```bash
export DAY4_ROLE="parent"
```

Child:

```bash
bash -c '
export DAY4_ROLE="child"
printf "Inside child: %s\n" "$DAY4_ROLE"
'
```

Output:

```text
Inside child: child
```

Parent:

```bash
printf 'Back in parent: %s\n' "$DAY4_ROLE"
```

Expected:

```text
Back in parent: parent
```

Giải thích:

```text
child inherited a copy
```

không phải:

```text
parent and child share one mutable environment table
```

---

# 107. Lab Bước 13 — Temporary environment

```bash
TEMP_MODE=test bash -c 'echo "child TEMP_MODE=$TEMP_MODE"'
```

Expected:

```text
child TEMP_MODE=test
```

Parent:

```bash
echo "parent TEMP_MODE=${TEMP_MODE:-unset}"
```

Expected:

```text
parent TEMP_MODE=unset
```

---

# 108. Lab Bước 14 — Inspect environment qua `/proc`

Start process bằng explicit environment:

```bash
DAY4_MARKER="started-with-this" sleep 600 &
ENV_PID=$!
```

Check PID:

```bash
echo "$ENV_PID"
```

Inspect:

```bash
tr '\0' '\n' < "/proc/$ENV_PID/environ" | grep '^DAY4_MARKER='
```

Expected:

```text
DAY4_MARKER=started-with-this
```

Điều này nối trực tiếp:

```text
shell environment
        ↓
exec process
        ↓
/proc/PID/environ
```

Cleanup:

```bash
kill -TERM "$ENV_PID"
```

Verify:

```bash
ps -p "$ENV_PID"
```

Expected process gone.

---

# 109. Lab Bước 15 — PATH

Check:

```bash
printf '%s\n' "$PATH"
```

Find executables:

```bash
command -v bash
command -v ps
command -v sleep
```

Ví dụ:

```text
/usr/bin/bash
/usr/bin/ps
/usr/bin/sleep
```

Sau đó thử environment tối giản:

```bash
env -i PATH=/usr/bin:/bin bash -c '
echo "PATH=$PATH"
command -v sleep
'
```

Điều này cho bạn thấy process có thể chạy trong environment rất khác shell login thông thường.

Đây chính là điều sẽ xảy ra ở một mức nào đó khi chuyển sang service management.

---

# 110. Lab Cleanup

Cleanup variables:

```bash
unset DAY4_LOCAL
unset DAY4_EXPORTED
unset DAY4_ROLE
unset DAY4_MARKER
unset LAB_PID
unset ENV_PID
```

Kiểm tra xem còn accidental `sleep 600` của chính lab không:

```bash
pgrep -a sleep
```

**Không kill những `sleep` không chắc chắn thuộc lab của bạn.**

Nếu còn PID bạn ghi nhận rõ là lab process, inspect trước:

```bash
ps -p <PID> -o pid,ppid,user,stat,lstart,cmd
```

rồi mới cleanup đúng PID.

---

# 111. Failure Injection — Process bị stopped nhưng người vận hành tưởng bị hang

Scenario:

```bash
sleep 600 &
FAIL_PID=$!
kill -STOP "$FAIL_PID"
```

User report:

```text
"Process vẫn tồn tại nhưng không chạy nữa."
```

Đừng đoán.

Expected troubleshooting:

```text
symptom:
process exists but appears inactive

↓
evidence:
ps -p PID -o pid,ppid,user,stat,cmd

↓
observation:
STAT=T

↓
hypothesis:
process has been stopped

↓
safe test:
confirm exact PID and state

↓
remediation:
kill -CONT PID

↓
validation:
STAT no longer T
```

Run:

```bash
ps -p "$FAIL_PID" -o pid,ppid,user,stat,cmd
```

Expected:

```text
T
```

Recover:

```bash
kill -CONT "$FAIL_PID"
```

Validate:

```bash
ps -p "$FAIL_PID" -o pid,stat,cmd
```

Cleanup:

```bash
kill -TERM "$FAIL_PID"
```

Đây là một failure-recovery flow nhỏ nhưng đúng methodology.

---

# 112. Supplementary Failure Injection — tạo zombie an toàn

Phần này chỉ làm nếu có `python3`:

```bash
command -v python3
```

Nếu không có thì bỏ qua. **Không cài package chỉ để làm bài này.**

Start:

```bash
python3 -c '
import os, time

pid = os.fork()
if pid == 0:
    os._exit(0)

time.sleep(120)
' &
ZPARENT=$!
```

Parent Python vẫn sống khoảng 120 giây.

Child exits ngay.

Check children:

```bash
ps --ppid "$ZPARENT" -o pid,ppid,stat,cmd
```

Bạn có thể thấy dạng:

```text
PID   PPID STAT CMD
8124  8123 Z    [python3] <defunct>
```

Đây là zombie.

Thử:

```bash
ps -p 8124 -o pid,ppid,stat,cmd
```

State:

```text
Z
```

Root cause:

```text
child exited
parent chưa wait()
```

Không phải:

```text
child đang chạy và bị hung
```

Cleanup parent:

```bash
kill -TERM "$ZPARENT"
```

Sau khi parent biến mất, zombie sẽ được reparented/reaped theo hệ thống process-reaping behavior và thường biến mất nhanh chóng.

Đây chỉ là lab để hiểu lifecycle, **không phải cách tạo zombie trên production**.

---

# 113. Troubleshooting Framework cho Process

Khi nhận ticket:

```text
"App process không ổn"
```

không bắt đầu bằng remediation.

Bắt đầu bằng evidence.

| Giai đoạn             | Câu hỏi                                           |
| --------------------- | ------------------------------------------------- |
| Symptom               | Người dùng thực sự quan sát thấy gì?              |
| Expected behavior     | Process đáng lẽ phải tồn tại hay không?           |
| Recent changes        | Có restart/deploy/config/package change không?    |
| Scope                 | Một process, một user, một host hay nhiều host?   |
| Identity              | PID, owner, executable, args là gì?               |
| State                 | `R/S/D/T/Z`?                                      |
| Parent                | Ai tạo process?                                   |
| Environment           | Process được start với environment gì?            |
| Evidence              | `ps`, `/proc`, logs/service state?                |
| Hypothesis            | Crash? stopped? wrong env? permission? stuck I/O? |
| Safe test             | Test nhỏ nhất để reject/confirm hypothesis        |
| Remediation           | Trong L1 scope hay phải escalate?                 |
| Validation            | Process state/behavior sau fix                    |
| Evidence preservation | Command/output/timestamp                          |
| Prevention            | Service management/config/runbook improvement     |

Đây là workflow bạn nên internalize.

---

# 114. Troubleshooting Case A — “Không thấy process”

Bạn chạy:

```bash
pgrep -af myapp
```

không có output.

Đừng lập tức kết luận:

```text
package chưa cài
```

Các hypotheses có thể gồm:

```text
process chưa start
process đã crash
search pattern sai
executable/command name khác
process thuộc namespace/container context khác
permission/process visibility restriction
```

L1 investigation cần context tiếp theo.

Với systemd workload, Module 2 sẽ bổ sung:

```bash
systemctl status ...
```

và minimal journal evidence.

---

# 115. Troubleshooting Case B — Process tồn tại nhưng state `T`

Evidence:

```text
PID 5321
STAT T
```

Hypothesis mạnh:

```text
process stopped
```

Kiểm tra recent operator actions/job control.

Nếu đúng target và L1 scope cho phép:

```bash
kill -CONT 5321
```

Validate:

```bash
ps -p 5321 -o pid,stat,cmd
```

Nếu state không còn `T`, recovery ở process-state level đã thành công.

Vẫn cần validate application behavior.

---

# 116. Troubleshooting Case C — Process `Z`

Evidence:

```text
PID 6001
PPID 5900
STAT Z
```

Không làm:

```bash
kill -9 6001
```

Investigation tập trung vào:

```text
PPID 5900
```

Ví dụ:

```bash
ps -p 5900 -o pid,ppid,user,stat,lstart,cmd
```

Nếu zombie tăng liên tục, collect:

```text
parent identity
zombie count
start time
rate of accumulation
logs
recent changes
```

rồi escalation tới application owner nếu nằm ngoài L1.

---

# 117. Troubleshooting Case D — Process state `D`

Nếu một critical process liên tục `D` trong thời gian dài:

```text
do not repeatedly kill
```

Collect:

```text
PID
state
duration
host scope
storage/network symptoms
recent mount/storage changes
```

Day 5 sẽ học:

```text
disk
filesystem
mount
capacity
network
```

nên lúc đó chúng ta sẽ nối investigation sâu hơn.

Một `D` state bền vững có thể vượt L1 app restart scope.

---

# 118. Troubleshooting Case E — “Chạy tay được, service không chạy”

Module 1 chỉ xây hypothesis:

```text
manual shell context
        ≠
service context
```

Possible difference:

```text
USER
GROUP
PATH
HOME
working directory
environment variables
permissions
stdin/stdout
resource limits
```

Đây sẽ là một case trung tâm của Module 2.

---

# 119. Security & Least Privilege

Process management cũng là security.

Không dùng:

```bash
sudo
```

chỉ để “cho chắc”.

Nếu process thuộc user bạn:

```text
normal user privilege
```

thường đủ để inspect và signal nó.

Nếu signal bị:

```text
Operation not permitted
```

câu hỏi đầu tiên không nên là:

```text
"làm sao bypass?"
```

mà là:

```text
"Tôi có thực sự được phép quản lý process này không?"
```

Day 3 least privilege được áp dụng trực tiếp ở đây.

---

# 120. Security — Không expose full environment vô tội vạ

Command:

```bash
printenv
```

hoặc:

```bash
cat /proc/PID/environ
```

có thể expose configuration nhạy cảm.

Trong evidence package, tốt hơn chỉ lấy key cần thiết:

```bash
tr '\0' '\n' < "/proc/$PID/environ" |
grep -E '^(JAVA_HOME|APP_ENV|PATH)='
```

Nhưng kể cả vậy cũng phải biết variable nào sensitive.

Không bao giờ yêu cầu/paste:

```text
PASSWORD
TOKEN
SECRET
AWS_SECRET_ACCESS_KEY
private key
```

vào bài học.

---

# 121. Security — Không dùng `.` trong PATH một cách tùy tiện

Ví dụ:

```bash
PATH=".:$PATH"
```

có thể khiến shell ưu tiên executable từ current directory trước trusted directories.

Nếu attacker có thể đặt executable tên:

```text
ls
sudo-like-helper
java
```

trong directory bạn đang đứng, PATH unsafe có thể dẫn tới thực thi file không mong muốn.

Vì vậy:

```text
predictable PATH
absolute paths where appropriate
least privilege
```

là practices quan trọng cho service administration.

---

# 122. Operations — Process inspection trước remediation

Một junior admin rất dễ tập trung vào:

```text
"fix nhanh"
```

Senior operational thinking tập trung vào:

```text
prove first
```

Ví dụ trước khi terminate:

```bash
date
ps -p "$PID" -o pid,ppid,user,stat,lstart,etime,cmd
```

Nếu cần:

```bash
readlink "/proc/$PID/exe"
readlink "/proc/$PID/cwd"
```

Rồi mới act.

Tại sao?

Vì sau:

```bash
kill
```

process biến mất.

Một phần evidence cũng biến mất.

---

# 123. Evidence preservation

Trước remediation, hãy nghĩ:

```text
What evidence disappears after I fix this?
```

Ví dụ process đang sai environment.

Nếu bạn restart ngay:

```text
old PID disappears
/proc/old_PID disappears
old environment evidence disappears
```

Vì vậy incident workflow tốt thường là:

```text
inspect
record
remediate
validate
```

không phải:

```text
restart first
investigate later
```

---

# 124. L1 boundary

Trong Day 4, những việc thường phù hợp L1 lab scope là:

```text
identify process
inspect PID/PPID/user/state
inspect known-safe environment fields
send approved TERM/STOP/CONT to owned lab process
verify termination
recognize zombie
identify parent
recognize suspicious D state
collect evidence
```

Những việc nên escalation hoặc cần runbook/approval trong môi trường thật gồm:

```text
killing unknown production processes
SIGKILL on database/application
repeated process crashes
large zombie accumulation
persistent D-state
root-owned critical daemon
unknown signal semantics
security-sensitive process
```

Đây không phải vì command khó, mà vì **blast radius và ownership**.

---

# 125. Command Reference của Module 1

| Command                  | Mục đích                              |      Thay đổi state? | Privilege              |
| ------------------------ | ------------------------------------- | -------------------: | ---------------------- |
| `ps`                     | Snapshot processes                    |                Không | Thường không           |
| `ps -ef`                 | Full system-style process listing     |                Không | Thường không           |
| `ps aux`                 | BSD user-oriented listing             |                Không | Thường không           |
| `ps -p PID -o ...`       | Inspect exact PID                     |                Không | Thường không           |
| `pgrep -a NAME`          | Tìm PID/name                          |                Không | Thường không           |
| `pgrep -af PATTERN`      | Match full command line               |                Không | Thường không           |
| `pstree -p`              | Process hierarchy                     |                Không | Thường không           |
| `top`                    | Live process observation              |   Không, nếu chỉ xem | Thường không           |
| `kill -TERM PID`         | Send SIGTERM                          |     Có thể terminate | Cần signal permission  |
| `kill -STOP PID`         | Stop execution                        |                   Có | Cần signal permission  |
| `kill -CONT PID`         | Resume                                |                   Có | Cần signal permission  |
| `kill -KILL PID`         | Force terminate                       |      Có, destructive | Cần signal permission  |
| `kill -0 PID`            | Existence/permission check            |                Không | Permission rules apply |
| `printenv`               | Inspect environment                   |                Không | Không                  |
| `env`                    | Inspect/construct command environment |     Không tới parent |
| `export NAME=value`      | Export shell variable                 | Có với shell context | Không                  |
| `unset NAME`             | Remove shell variable                 | Có với shell context | Không                  |
| `readlink /proc/PID/exe` | Executable path                       |                Không | Access-dependent       |
| `readlink /proc/PID/cwd` | Working directory                     |                Không | Access-dependent       |
| `cat /proc/PID/status`   | Kernel process state                  |                Không | Access-dependent       |

---

# 126. Expected output vs Expected invariant

Trong lab operations, đừng thuộc lòng literal output.

Ví dụ tôi có thể đưa:

```text
PID 7450
```

nhưng máy bạn có thể là:

```text
PID 12983
```

Do đó phân biệt:

**Example output**

```text
7450
```

với:

**Expected invariant**

```text
PID phải là một PID hợp lệ của process bạn vừa tạo
```

Tương tự:

```text
/usr/bin/sleep
```

có thể khác trên môi trường khác, nhưng expected invariant là:

```text
/proc/PID/exe phải trỏ tới executable thực tế của process sleep
```

Đây là cách làm lab đúng.

---

# 127. Exam Focus — những điều phải trả lời được

Nếu theory exam hỏi:

> Process khác program thế nào?

Bạn phải trả lời được:

```text
Program là executable/code;
process là một runtime instance với PID, memory, credentials,
environment và các resources/state do kernel quản lý.
```

Nếu hỏi:

> PID và PPID?

Bạn phải trả lời được:

```text
PID định danh process;
PPID định danh parent process.
```

Nếu hỏi:

> `kill PID` gửi signal nào mặc định?

```text
SIGTERM
```

theo standard Linux utility behavior.

Nếu hỏi:

> Signal nào không thể caught/blocked/ignored?

```text
SIGKILL
SIGSTOP
```

---

# 128. Exam Focus — process states

Phải nhận ra:

```text
R → running/runnable
S → interruptible sleep
D → uninterruptible sleep
T → stopped
Z → zombie
```

Và đặc biệt:

```text
Zombie đã terminate execution.
```

Không trả lời:

> Zombie là process chạy ngầm và không thể kill.

Sai.

---

# 129. Exam Focus — environment

Bạn phải giải thích:

```bash
FOO=bar
```

khác:

```bash
export FOO=bar
```

ở chỗ exported variable được đưa vào environment của subsequently executed child commands.

Bạn cũng phải giải thích:

```text
child environment change
```

không tự động sửa:

```text
parent environment
```

---

# 130. Exam Focus — practical

Nếu được giao:

```text
Find process, identify owner/state/parent,
pause it, resume it, then terminate it safely
```

workflow hợp lý:

```bash
pgrep -a <name>

ps -p <PID> -o pid,ppid,user,stat,lstart,etime,cmd

kill -STOP <PID>

ps -p <PID> -o pid,stat,cmd

kill -CONT <PID>

ps -p <PID> -o pid,stat,cmd

kill -TERM <PID>

ps -p <PID>
```

Nhưng trong practical exam, **đừng copy PID mù quáng**. Phải inspect target identity.

---

# 131. Knowledge Check

Hãy tự trả lời trước khi nhìn đáp án ở phần sau.

| #   | Câu hỏi                                                                               |
| --- | ------------------------------------------------------------------------------------- |
| 1   | Program và process khác nhau thế nào?                                                 |
| 2   | PID và PPID biểu diễn gì?                                                             |
| 3   | `ps` và `top` khác nhau ở mental model nào?                                           |
| 4   | State `S` có nhất thiết là lỗi không?                                                 |
| 5   | State `Z` nghĩa là gì?                                                                |
| 6   | Vì sao `kill -9` zombie không giải quyết root cause?                                  |
| 7   | `kill PID` mặc định gửi signal gì?                                                    |
| 8   | Vì sao ưu tiên SIGTERM hơn SIGKILL?                                                   |
| 9   | Hai signal nào process không thể catch/block/ignore?                                  |
| 10  | `kill -0 PID` làm gì?                                                                 |
| 11  | Shell variable khác exported environment variable thế nào?                            |
| 12  | Child có thể `export` variable để thay đổi parent không?                              |
| 13  | Tại sao shell environment và systemd service environment không nhất thiết giống nhau? |
| 14  | `/proc/PID/exe` cho biết gì?                                                          |
| 15  | `/proc/PID/cwd` có ích trong troubleshooting nào?                                     |
| 16  | `/proc/PID/environ` có caveat gì?                                                     |
| 17  | Process tồn tại có chứng minh application healthy không?                              |
| 18  | Tại sao phải collect evidence trước restart/kill?                                     |

---

# 132. Đáp án Knowledge Check

**1.** Program là executable/code; process là runtime instance của program với PID, memory, runtime state và resources.

**2.** PID là Process ID; PPID là PID của parent process.

**3.** `ps` cho snapshot; `top` cung cấp refreshed/live view.

**4.** Không. `S` thường là interruptible sleep và hoàn toàn bình thường với process đang chờ event.

**5.** Process đã terminate nhưng parent chưa collect/reap termination status.

**6.** Vì zombie không còn execution để kill; vấn đề là parent chưa reap nó.

**7.** `SIGTERM`.

**8.** Vì process có thể handle SIGTERM và thực hiện graceful cleanup; SIGKILL không cho user-space handler chạy.

**9.** `SIGKILL` và `SIGSTOP`.

**10.** Không gửi signal thật nhưng thực hiện existence và permission checks.

**11.** Shell variable chỉ tồn tại trong shell state; exported variable được truyền cho subsequently executed child process.

**12.** Không. Child thay đổi environment của chính nó, không sửa ngược parent.

**13.** Environment là process-specific/inherited context; systemd không đơn giản thừa hưởng interactive SSH shell environment.

**14.** Symlink tới executable pathname của process.

**15.** Xác định working directory, rất hữu ích với relative paths/configuration.

**16.** Nó chủ yếu phản ánh initial environment được set khi program được exec; application modifications về sau có technical caveats và không nhất thiết hiện ở đó.

**17.** Không. PID tồn tại chỉ là process-level evidence, không phải application-health evidence.

**18.** Vì process termination/restart có thể làm mất `/proc` state, old PID/environment và các runtime clues cần để tìm root cause.

---

# 133. Independent Practical Challenge

Không nhìn lại Guided Lab, hãy tự thực hiện scenario sau trên **lab machine**, không dùng `sudo`.

Bạn cần tạo một process với:

```text
DAY4_ENV=practice
```

và để process chạy đủ lâu để inspect.

Sau đó chứng minh bằng evidence:

```text
PID
PPID
owner
state
command line
actual executable
current working directory
DAY4_ENV trong startup environment
```

Tiếp theo bạn phải:

```text
STOP process
prove state T
CONT process
prove process resumed
TERM process
prove process disappeared
```

Evidence cuối cùng nên đủ để một mentor đọc và hiểu:

```text
process nào được tạo
nó chạy trong context nào
signal nào đã gửi
state chuyển đổi ra sao
cleanup thành công chưa
```

---

# 134. Reference Solution cho challenge

Một solution hợp lệ:

```bash
DAY4_ENV=practice sleep 900 &
PID=$!

echo "PID=$PID"

ps -p "$PID" -o pid,ppid,user,stat,lstart,etime,cmd

readlink "/proc/$PID/exe"

readlink "/proc/$PID/cwd"

tr '\0' '\n' < "/proc/$PID/environ" |
grep '^DAY4_ENV='
```

Expected invariant:

```text
DAY4_ENV=practice
```

Stop:

```bash
kill -STOP "$PID"

ps -p "$PID" -o pid,ppid,user,stat,cmd
```

Expected:

```text
STAT begins with T
```

Continue:

```bash
kill -CONT "$PID"

ps -p "$PID" -o pid,ppid,user,stat,cmd
```

Expected:

```text
state no longer T
```

Terminate gracefully:

```bash
kill -TERM "$PID"
```

Validate:

```bash
ps -p "$PID"
```

và:

```bash
test -d "/proc/$PID" &&
echo "STILL EXISTS" ||
echo "PROCESS GONE"
```

Expected invariant:

```text
PROCESS GONE
```

---

# 135. Mastery Check

Sau Module 1, mental model bạn cần giữ trong đầu là:

```text
PROGRAM ON DISK
      │
      │ execution
      ↓
PROCESS
├── PID
├── PPID
├── owner / credentials
├── executable
├── command line
├── working directory
├── open file descriptors
├── environment
├── process state
├── signal handling
└── threads
      │
      ├── R = running/runnable
      ├── S = interruptible sleep
      ├── D = uninterruptible sleep
      ├── T = stopped
      └── Z = zombie
```

Và signal model:

```text
administrator/kernel/process
            │
            │ signal
            ↓
          target
            │
            ├── TERM → graceful opportunity
            ├── INT  → interrupt
            ├── STOP → forced stop
            ├── CONT → resume
            ├── HUP  → application-specific use
            └── KILL → forced termination
```

Environment model:

```text
parent process
export APP_ENV=prod
        │
        │ child creation + exec
        ↓
child process
APP_ENV=prod
        │
        │ child changes APP_ENV
        ↓
APP_ENV=test

parent remains:
APP_ENV=prod
```

Và operational model:

```text
symptom
   ↓
expected behavior
   ↓
scope
   ↓
PID / PPID / USER
   ↓
state
   ↓
executable / cmdline / cwd
   ↓
environment
   ↓
hypothesis
   ↓
safest test
   ↓
approved remediation
   ↓
validation
   ↓
preserve evidence
```

## Điều quan trọng nhất phải mang sang Module 2

Module 1 cho bạn hiểu **process**.

Module 2 sẽ đặt thêm một lớp quản lý phía trên:

```text
systemd
   ↓
service unit
   ↓
execution context
   ├── User=
   ├── WorkingDirectory=
   ├── Environment=
   ├── EnvironmentFile=
   └── ExecStart=
           ↓
         process
           ├── PID
           ├── environment
           ├── state
           └── signals
```

Vì vậy khi sang `systemd`, những thứ như `ExecStart=`, `User=`, `Environment=`, `Restart=` hay `systemctl stop` sẽ không còn là những directive/lệnh phải học thuộc. Bạn đã có mental model phía dưới để hiểu **systemd đang tạo, quan sát và điều khiển process nào, trong execution context nào, bằng lifecycle nào**. Đó chính là nền tảng để học Module 2 ở độ sâu production operations.
