# Day 4 — Module 2: `systemd` Services & Service Startup

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 4 bắt buộc học:

> **Concept/Lecture:** Processes, signals, systemd services, packages, environment variables, and service startup.  
> **Assignment/Lab:** Install a package, create/manage a systemd service, inspect processes, and recover a failed service. :chatgpt-content-reference{index="0"}

Module 1 đã xây nền về **process → PID/PPID → state → signals → environment**. Module 2 bây giờ đặt một lớp quản lý phía trên process:

```text
Linux kernel
    │
    ↓
systemd (system manager, PID 1)
    │
    ↓
unit
    │
    ↓
service unit
    │
    ├── execution identity
    ├── working directory
    ├── environment
    ├── startup command
    ├── dependencies
    ├── restart policy
    ├── stop behavior
    └── boot activation
            │
            ↓
         process(es)
```

Đây là phần bắt buộc của syllabus. Những nội dung như `cgroups`, drop-in overrides, `Type=notify`, `systemd-analyze verify`, `mask`, rate limiting và hardening sẽ được đánh dấu là **Supplementary / Beyond the explicit syllabus**, nhưng chúng rất hữu ích cho Java/Tomcat và operations thực tế.

Tài liệu hiện hành của systemd vẫn định nghĩa `.service` là unit mô tả một process được systemd điều khiển và supervise; execution context được bổ sung bởi các directive của `systemd.exec`. :chatgpt-content-reference{index="1"}

---

# 2. Mục tiêu sau Module 2

Sau bài này, bạn phải có thể nhìn một file:

```ini
[Unit]
Description=Example Application
After=network.target

[Service]
Type=exec
User=appuser
WorkingDirectory=/opt/myapp
EnvironmentFile=/etc/myapp.env
ExecStart=/opt/myapp/bin/start-app
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

và giải thích được **từng directive đang thay đổi điều gì trong runtime**.

Bạn cũng phải hiểu chắc các distinction sau:

| Dễ nhầm                            | Thực tế                                                                 |
| ---------------------------------- | ----------------------------------------------------------------------- |
| process vs service                 | process là runtime entity; service là unit/lifecycle do systemd quản lý |
| `start` vs `enable`                | start = chạy bây giờ; enable = cấu hình activation, thường cho boot     |
| active vs enabled                  | hai chiều state độc lập                                                 |
| `reload` vs `daemon-reload`        | reload app config vs reload systemd unit definitions                    |
| `restart` vs reload                | process lifecycle restart vs app config reload                          |
| `Wants=` vs `After=`               | requirement/pull-in vs ordering                                         |
| `Requires=` vs `After=`            | strong requirement vẫn không tự biểu diễn ordering                      |
| `Environment=` vs shell `export`   | service không mặc nhiên kế thừa interactive shell                       |
| `systemctl status` vs health check | service state không chứng minh business health                          |
| failed state vs root cause         | `failed` là kết quả, không phải nguyên nhân                             |

---

# 3. Tại sao cần service manager?

Ở Module 1, bạn có thể chạy:

```bash
java -jar app.jar
```

hoặc:

```bash
java -jar app.jar &
```

Nhưng production operations cần nhiều hơn việc “đưa process xuống background”.

Giả sử server reboot.

Process bạn chạy thủ công:

```text
java -jar app.jar &
```

không tự nhiên xuất hiện lại.

Giả sử process crash.

Ai phát hiện?

Ai restart?

Giả sử cần chạy bằng user `tomcat`, trong directory `/opt/app`, với environment chính xác.

Ai đảm bảo?

Giả sử administrator cần:

```text
start
stop
restart
check status
enable at boot
collect lifecycle evidence
```

Nếu mỗi application tự làm theo cách riêng, operations trở nên không nhất quán.

`systemd` giải quyết vấn đề đó bằng cách biến runtime management thành một **declarative contract**.

Thay vì:

```text
Admin tự nhớ:
run command X
cd directory Y
export variables A/B/C
run as user Z
remember PID
restart if it dies
start it after boot
```

ta khai báo:

```text
Service unit
│
├── User=Z
├── WorkingDirectory=Y
├── Environment=...
├── ExecStart=X
├── Restart=...
└── WantedBy=...
```

Sau đó service manager thực thi contract đó.

---

# 4. `systemd` là gì?

Trên Linux distribution sử dụng systemd, system instance thường chạy là **PID 1** và đóng vai trò init/service manager.

Kiểm tra:

```bash
ps -p 1 -o pid,ppid,comm,args
```

Ví dụ:

```text
PID  PPID COMMAND  COMMAND
1       0 systemd  /sbin/init
```

hoặc:

```text
1  0  systemd  /lib/systemd/systemd --system ...
```

`/sbin/init` đôi khi là symlink tới systemd.

Tài liệu systemd hiện hành mô tả system instance chạy ở PID 1 và chịu trách nhiệm bring up/maintain userspace services; user sessions có thể có các systemd manager riêng. :chatgpt-content-reference{index="2"}

Kiểm tra version:

```bash
systemctl --version
```

Ví dụ:

```text
systemd 255 (...)
```

Version của bạn có thể khác.

Đây là điều quan trọng vì một số directive mới không tồn tại trên systemd quá cũ.

Những directive/lệnh cốt lõi chúng ta học hôm nay như:

```text
ExecStart=
User=
Group=
WorkingDirectory=
Environment=
EnvironmentFile=
Restart=
WantedBy=

systemctl start
systemctl stop
systemctl restart
systemctl status
systemctl enable
systemctl disable
systemctl daemon-reload
```

đã rất ổn định trên các distro systemd hiện đại.

---

# 5. System manager và user manager

Có hai khái niệm cần phân biệt.

## System service

Ví dụ:

```bash
sudo systemctl status ssh
```

được system systemd manager quản lý.

Unit thường nằm ở các đường dẫn như:

```text
/etc/systemd/system/
/usr/lib/systemd/system/
```

hoặc trên một số distro:

```text
/lib/systemd/system/
```

System manager thường là:

```text
PID 1
```

---

## User service

Ví dụ:

```bash
systemctl --user status some.service
```

được per-user systemd manager quản lý.

Không phải PID 1.

User service phù hợp cho workloads thuộc user session.

Trong curriculum này, Java/Tomcat application server và các infrastructure services chủ yếu liên quan tới **system services**, nên đây là trọng tâm Module 2.

Một khác biệt quan trọng: execution environment của system service và user service không giống nhau. Current systemd documentation nói system manager thường **không đơn giản truyền toàn bộ environment của interactive shell**, trong khi user manager có behavior khác và thường thừa hưởng/import thêm user-session environment. :chatgpt-content-reference{index="3"}

---

# 6. Mental model: systemd quản lý `unit`

Đơn vị cơ bản của systemd không phải chỉ là:

```text
service
```

mà là:

```text
unit
```

Systemd hỗ trợ nhiều unit type.

Ví dụ:

```text
sshd.service
multi-user.target
foo.socket
home.mount
backup.timer
dev-sda.device
```

Suffix cho biết unit type.

Trong Day 4, trọng tâm:

```text
.service
```

Nhưng bạn nên nhận ra:

| Unit type  | Mục đích                                                      |
| ---------- | ------------------------------------------------------------- |
| `.service` | Service/process lifecycle                                     |
| `.target`  | Nhóm/synchronization point của units                          |
| `.socket`  | Socket activation                                             |
| `.timer`   | Time-based activation                                         |
| `.mount`   | Filesystem mount                                              |
| `.path`    | Path-based activation                                         |
| `.device`  | Kernel device representation                                  |
| `.slice`   | Resource hierarchy                                            |
| `.scope`   | Processes được systemd quản lý nhưng tạo từ bên ngoài service |

Chúng ta chỉ đào sâu `.service` và một phần `.target`.

---

# 7. Unit không giống process

Ví dụ:

```text
tomcat.service
```

là một unit.

Trong unit có thể có:

```text
MainPID = 4210
```

nhưng service unit có thể chứa cả process tree:

```text
tomcat.service
│
├── java PID 4210
│
├── child PID 4220
└── helper PID 4231
```

Modern systemd sử dụng Linux **control groups — cgroups** để theo dõi processes thuộc unit, thay vì chỉ dựa vào một PID đơn lẻ. systemd documentation mô tả processes mà systemd spawn được đặt vào cgroup của unit, giúp systemd theo dõi cả nhóm process. :chatgpt-content-reference{index="4"}

Đây là một lợi thế lớn so với kiểu quản lý cổ điển:

```text
app.pid
```

---

# 8. Supplementary — cgroup mental model

Nếu service:

```text
day4-demo.service
```

đang chạy, systemd có thể tổ chức:

```text
/system.slice/day4-demo.service/
        │
        ├── PID 8000 bash
        └── PID 8001 sleep
```

Check:

```bash
systemctl status day4-demo.service
```

thường có phần:

```text
CGroup: /system.slice/day4-demo.service
        ├─8000 ...
        └─8001 ...
```

Hoặc:

```bash
systemd-cgls
```

Điều này giải thích vì sao systemd có thể quản lý:

```text
main process
+
child processes
```

tốt hơn script kiểu:

```bash
kill "$(cat app.pid)"
```

---

# 9. `systemctl`

`systemctl` là CLI chính để:

```text
introspect
+
control
```

systemd manager. :chatgpt-content-reference{index="5"}

Mental model:

```text
administrator
     │
     │ systemctl start app.service
     ↓
systemd manager (PID 1)
     │
     ├── load unit
     ├── resolve dependencies
     ├── construct execution context
     ├── spawn process
     └── track lifecycle
```

`systemctl` không phải service process.

Nó là một **client** nói chuyện với service manager.

---

# 10. `systemctl status`

Ví dụ:

```bash
systemctl status ssh.service
```

hoặc tùy distro:

```bash
systemctl status sshd.service
```

Output thường có dạng:

```text
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (...; enabled; preset: enabled)
     Active: active (running) since ...
   Main PID: 1234 (sshd)
      Tasks: ...
     Memory: ...
        CPU: ...
     CGroup: /system.slice/ssh.service
             └─1234 /usr/sbin/sshd ...
```

Ba dòng đặc biệt quan trọng:

```text
Loaded:
Active:
Main PID:
```

---

# 11. `Loaded:` nghĩa là gì?

`Loaded:` không có nghĩa:

```text
service đang chạy
```

Nó nói về unit definition.

Ví dụ:

```text
Loaded: loaded
```

nghĩa systemd load/parse unit được.

Có thể gặp:

```text
not-found
bad-setting
error
masked
```

Current `systemctl` documentation phân biệt:

```text
LOAD
ACTIVE
SUB
```

trong đó `LOAD` phản ánh unit definition có được load đúng không; `ACTIVE` là high-level activation state; `SUB` là low-level state tùy unit type. :chatgpt-content-reference{index="6"}

---

# 12. `Active:` là runtime state

Ví dụ:

```text
Active: active (running)
```

có:

```text
high-level state = active
substate         = running
```

Một service có thể có các trạng thái:

```text
inactive
activating
active
deactivating
failed
```

Ví dụ:

```text
active (running)
active (exited)
inactive (dead)
failed (Result: exit-code)
activating (auto-restart)
```

Không nên chỉ đọc chữ `active` mà bỏ qua substate/context.

---

# 13. `failed` là một state

Service có thể vào:

```text
failed
```

nếu chẳng hạn:

```text
process exits non-zero
process crashes
startup timeout
stop timeout
restart attempts exceed limit
execution setup fails
```

`failed` không phải root cause.

Nó là:

```text
observed state
```

Root cause có thể là:

```text
wrong user
wrong permissions
missing executable
bad configuration
missing environment file
port conflict
dependency failure
application crash
```

Current `systemctl` docs xác nhận `failed` được dùng khi unit gặp crash, error exit, timeout hoặc tương tự; nguyên nhân được giữ/log lại để administrator introspect. :chatgpt-content-reference{index="7"}

---

# 14. `Main PID`

Ví dụ:

```text
Main PID: 8032
```

Systemd đang coi PID `8032` là main process của service.

Ta có thể nối Module 1 ngay lập tức:

```bash
ps -p 8032 -o pid,ppid,user,stat,lstart,etime,cmd
```

Hoặc:

```bash
cat /proc/8032/status
```

Hoặc:

```bash
tr '\0' '\n' < /proc/8032/environ
```

Đây là integration rất quan trọng:

```text
systemd
   ↓
unit
   ↓
MainPID
   ↓
Linux process
   ↓
/proc
```

---

# 15. `systemctl status` không phải full health check

Nếu:

```text
Active: active (running)
```

bạn mới chứng minh:

> systemd coi service đang active theo lifecycle contract.

Bạn chưa chứng minh:

```text
HTTP endpoint OK
port listening
DB connection OK
application ready
business transaction OK
latency OK
dependency healthy
```

Sau này application validation sẽ nối:

```text
systemctl
   ↓
process
   ↓
socket/port
   ↓
HTTP endpoint
   ↓
application
   ↓
database
```

Vì vậy không viết ticket:

> Application healthy because `systemctl status` is green.

Chính xác hơn:

> Service unit is active; application-level validation still required.

---

# 16. `systemctl show`

`status` được thiết kế chủ yếu cho người đọc.

Khi muốn properties rõ ràng:

```bash
systemctl show ssh.service
```

Hoặc chọn:

```bash
systemctl show ssh.service \
  -p LoadState \
  -p ActiveState \
  -p SubState \
  -p UnitFileState \
  -p MainPID
```

Ví dụ:

```text
LoadState=loaded
ActiveState=active
SubState=running
UnitFileState=enabled
MainPID=1234
```

Tài liệu systemd khuyến nghị `show` khi cần output dễ dùng bằng chương trình, trong khi `status` là human-readable. :chatgpt-content-reference{index="8"}

---

# 17. Runtime state và enablement state là hai dimension khác nhau

Đây là câu cực quan trọng.

Một service có thể:

| Runtime  | Enablement | Ý nghĩa                                |
| -------- | ---------- | -------------------------------------- |
| active   | enabled    | đang chạy và configured for activation |
| inactive | enabled    | hiện không chạy nhưng được enabled     |
| active   | disabled   | chạy bây giờ nhưng không enabled       |
| inactive | disabled   | không chạy và không enabled            |

Do đó:

```text
ACTIVE ≠ ENABLED
```

Hãy nhớ bằng mô hình:

```text
start / stop
      ↓
CURRENT RUNTIME

enable / disable
      ↓
FUTURE ACTIVATION CONFIGURATION
```

---

# 18. `systemctl start`

```bash
sudo systemctl start app.service
```

có nghĩa:

```text
activate app.service now
```

Current documentation định nghĩa `start` là start/activate unit. :chatgpt-content-reference{index="9"}

Nó **không tự động có nghĩa**:

```text
enable at boot
```

Sau start:

```bash
systemctl is-active app.service
```

có thể trả:

```text
active
```

nhưng:

```bash
systemctl is-enabled app.service
```

vẫn có thể trả:

```text
disabled
```

Hoàn toàn hợp lệ.

---

# 19. `systemctl stop`

```bash
sudo systemctl stop app.service
```

yêu cầu systemd deactivate service.

Điểm quan trọng:

> Nếu service được systemd quản lý, ưu tiên nói chuyện với service manager thay vì tùy tiện `kill PID`.

Tại sao?

Vì `systemctl stop` cho systemd biết lifecycle transition đang xảy ra.

Systemd có thể thực hiện:

```text
ExecStop=
signal handling
timeout
cleanup
cgroup cleanup
dependency handling
state transition
```

Nếu bạn bypass bằng:

```bash
kill -9 MAINPID
```

service có thể:

```text
auto-restart
enter failed state
leave children/resources
trigger dependency behavior
```

khác với một controlled stop.

---

# 20. Nếu không có `ExecStop=` thì sao?

Service không nhất thiết phải có:

```ini
ExecStop=
```

Nếu không có, systemd có thể signal process theo service kill configuration.

Current service documentation nói nếu không có `ExecStop=`, service process được gửi configured `KillSignal=`; default behavior thường sử dụng `SIGTERM`, và nếu service không kết thúc trong stop timeout thì có thể bị cưỡng chế bằng `SIGKILL`. :chatgpt-content-reference{index="10"}

Điều này nối trực tiếp Module 1:

```text
systemctl stop
        ↓
systemd
        ↓
SIGTERM
        ↓
application graceful shutdown opportunity
```

---

# 21. `systemctl restart`

```bash
sudo systemctl restart app.service
```

về cơ bản là yêu cầu:

```text
stop
 ↓
start
```

Current `systemctl` docs mô tả `restart` là stop rồi start; nếu unit đang inactive thì `restart` vẫn có thể start nó. :chatgpt-content-reference{index="11"}

Đây là distinction:

```text
restart
```

không phải:

```text
"refresh config bằng magic"
```

Process thường bị thay đổi.

Ví dụ:

Trước:

```text
MainPID=8000
```

Sau restart:

```text
MainPID=8122
```

PID mới là evidence rõ rằng process lifecycle đã thay đổi.

---

# 22. `try-restart`

Supplementary nhưng hữu ích:

```bash
sudo systemctl try-restart app.service
```

Khác:

```text
restart:
inactive → start

try-restart:
inactive → do nothing
```

Current docs định nghĩa `try-restart` chỉ restart nếu unit hiện đang running. :chatgpt-content-reference{index="12"}

Trong automated change scripts, distinction này có thể quan trọng.

---

# 23. `reload`

```bash
sudo systemctl reload app.service
```

có nghĩa:

> Yêu cầu **application/service** reload configuration nếu service hỗ trợ reload.

Nó không có nghĩa:

> systemd đọc lại `.service` unit file.

Ví dụ Apache:

```text
systemctl reload apache2
```

có thể yêu cầu Apache reload:

```text
httpd.conf
virtual hosts
```

mà không terminate toàn bộ application process theo cách restart.

Current `systemctl` documentation nói rõ `reload` reload **service-specific configuration**, không phải systemd unit definition. :chatgpt-content-reference{index="13"}

---

# 24. `daemon-reload`

```bash
sudo systemctl daemon-reload
```

hoàn toàn khác.

Nó nói với **systemd manager**:

> Hãy đọc lại unit definitions và dựng lại dependency tree.

Current docs mô tả `daemon-reload` sẽ rerun generators, reload unit files và recreate dependency tree. :chatgpt-content-reference{index="14"}

Mental model:

```text
/etc/systemd/system/app.service
             │
             │ edit
             ↓
disk changed

systemd manager memory
still old definition
             │
             │ daemon-reload
             ↓
systemd reloads unit definitions
```

---

# 25. Quy tắc phải thuộc: edit unit → daemon-reload

Nếu bạn sửa:

```text
/etc/systemd/system/app.service
```

thì pattern:

```bash
sudo systemd-analyze verify /etc/systemd/system/app.service
sudo systemctl daemon-reload
```

sau đó mới:

```bash
sudo systemctl restart app.service
```

hoặc start.

Nếu quên `daemon-reload`, systemd có thể vẫn dùng definition đã load trước đó.

Đây là một trong những lỗi practical exam rất dễ gặp.

---

# 26. `reload` vs `daemon-reload`

Hãy học bảng này thật chắc:

| Command                   | Ai reload?      | Cái gì được reload?                 |
| ------------------------- | --------------- | ----------------------------------- |
| `systemctl reload app`    | Application     | App-specific configuration          |
| `systemctl daemon-reload` | systemd manager | systemd unit files/dependency graph |

Ví dụ:

```text
/etc/myapp/application.conf changed
```

nếu application hỗ trợ reload:

```bash
systemctl reload myapp
```

Nhưng:

```text
/etc/systemd/system/myapp.service changed
```

cần:

```bash
systemctl daemon-reload
```

sau đó thường phải restart/start để execution context mới áp dụng.

---

# 27. `daemon-reload` không restart service

Một misunderstanding khác:

```bash
sudo systemctl daemon-reload
```

không có nghĩa:

```text
running process automatically restarted
```

Giả sử bạn đổi:

```ini
Environment="APP_ENV=old"
```

thành:

```ini
Environment="APP_ENV=new"
```

rồi chạy:

```bash
sudo systemctl daemon-reload
```

Process cũ vẫn đang chạy với environment ban đầu.

Bạn thường cần:

```bash
sudo systemctl restart app.service
```

để tạo process mới với execution context mới.

Đây chính là environment inheritance từ Module 1.

---

# 28. Unit file anatomy

Một `.service` thường có ba section chính:

```ini
[Unit]

[Service]

[Install]
```

Mental model:

| Section     | Câu hỏi nó trả lời                            |
| ----------- | --------------------------------------------- |
| `[Unit]`    | Unit là gì và quan hệ với units khác thế nào? |
| `[Service]` | Process phải chạy thế nào?                    |
| `[Install]` | Enable unit sẽ hook nó vào đâu?               |

Không phải service nào cũng bắt buộc sử dụng mọi section.

---

# 29. `[Unit]`

Ví dụ:

```ini
[Unit]
Description=Day 4 Demo Service
Documentation=https://example.invalid/docs
After=network.target
Wants=network.target
```

Các directive như:

```text
Description=
Documentation=
Wants=
Requires=
After=
Before=
```

nằm ở đây.

`[Unit]` chủ yếu nói về metadata và relationship/dependency.

---

# 30. `Description=`

Ví dụ:

```ini
Description=Day 4 Demo Worker
```

Bạn sẽ thấy nó trong:

```bash
systemctl status day4-demo
```

Description tốt nên nói:

```text
service làm gì
```

chứ không phải:

```text
Service
Application
Process
```

Một description như:

```ini
Description=Order Processing Java API
```

hữu ích hơn:

```ini
Description=Java Service
```

---

# 31. `[Service]`

Đây là phần cốt lõi.

Ví dụ:

```ini
[Service]
Type=exec
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp
EnvironmentFile=/etc/myapp.env
ExecStart=/opt/myapp/bin/myapp
Restart=on-failure
RestartSec=5s
```

Nó định nghĩa execution environment và lifecycle.

---

# 32. `ExecStart=`

Directive quan trọng nhất:

```ini
ExecStart=/opt/myapp/bin/myapp
```

Nó nói:

> Khi service start, execute command này.

Đối với hầu hết service type thông thường, process được start từ `ExecStart=` trở thành **main process** của service. :chatgpt-content-reference{index="15"}

Ví dụ:

```ini
ExecStart=/usr/bin/java -jar /opt/app/app.jar
```

Conceptually:

```text
systemd
   │
   └── exec Java
          │
          └── MainPID
```

---

# 33. `ExecStart=` không chạy qua shell theo mặc định

Đây là lỗi cực kỳ phổ biến.

Ví dụ:

```ini
ExecStart=/usr/bin/java -jar /opt/app/app.jar > /var/log/app.log
```

Fresher có thể tưởng:

```text
">" sẽ redirect như Bash
```

Không đúng.

Systemd command parsing **không phải Bash command line**.

Current systemd docs nói các shell constructs như:

```text
>
>>
<
|
&
```

không được xử lý như shell syntax mặc định. :chatgpt-content-reference{index="16"}

Nếu thật sự cần shell:

```ini
ExecStart=/bin/sh -c 'command | other-command'
```

Nhưng production service nên tránh shell nếu không cần.

---

# 34. Tại sao tránh `/bin/sh -c` không cần thiết?

Ví dụ:

```ini
ExecStart=/bin/sh -c '/usr/bin/java -jar /opt/app/app.jar'
```

thêm một lớp:

```text
systemd
  ↓
shell
  ↓
java
```

trong khi:

```ini
ExecStart=/usr/bin/java -jar /opt/app/app.jar
```

đơn giản hơn:

```text
systemd
  ↓
java
```

Lợi ích:

```text
process identity rõ hơn
ít quoting complexity
ít shell expansion surprise
ít attack surface hơn
signal behavior rõ hơn
```

Chỉ dùng shell khi bạn **thực sự cần shell semantics**.

---

# 35. Executable path trong `ExecStart=`

Current systemd cho phép argument đầu là:

```text
absolute path
```

hoặc simple executable name không có slash; nếu không dùng absolute path thì systemd resolve bằng fixed search path cho standard binary directories. :chatgpt-content-reference{index="17"}

Ví dụ hợp lệ trên modern systemd:

```ini
ExecStart=java -version
```

nhưng trong administrative units, mình khuyến nghị khi thực tế phù hợp:

```ini
ExecStart=/usr/bin/java -version
```

vì execution intent rõ hơn.

Trước khi viết:

```bash
command -v java
```

Ví dụ:

```text
/usr/bin/java
```

---

# 36. Không giả định path

Trên hệ thống A:

```text
/usr/bin/java
```

Hệ thống B:

```text
/usr/lib/jvm/java-17-openjdk/bin/java
```

Hoặc symlink khác.

Luôn kiểm tra:

```bash
command -v java
readlink -f "$(command -v java)"
```

Đây là principle:

```text
inspect first
configure second
```

---

# 37. `Type=`

`Type=` nói với systemd:

> Khi nào service được coi là đã start, và main process/lifecycle cần được hiểu theo model nào?

Các type hiện hành gồm nhiều loại, nhưng fresher cần hiểu sâu ba loại:

```text
simple
exec
oneshot
```

và awareness về:

```text
forking
notify
dbus
```

---

# 38. `Type=simple`

Nếu:

```ini
[Service]
ExecStart=/opt/app/start
```

và không có `Type=`, thông thường default là:

```ini
Type=simple
```

Với `simple`, systemd coi startup tiến triển rất sớm sau khi main process được spawn. Current documentation lưu ý rằng `systemctl start` cho `Type=simple` thậm chí có thể trả success trước khi actual executable invocation failure được phát hiện trong một số setup-stage cases. :chatgpt-content-reference{index="18"}

Mental model:

```text
systemd
  ↓
spawn process
  ↓
service considered started relatively early
  ↓
exec application
```

---

# 39. `Type=exec`

Trên systemd hiện đại:

```ini
Type=exec
```

giống `simple`, nhưng systemd chờ cho đến khi actual service binary được `execve()` thành công trước khi coi startup hoàn thành ở điểm tương ứng.

Current docs nói `Type=exec` thường là lựa chọn tốt hơn `simple` khi phù hợp, vì setup/executable invocation failures được phản ánh chính xác hơn trong start operation. :chatgpt-content-reference{index="19"}

Mental model:

```text
systemd
   ↓
prepare execution context
   ↓
execve(application)
   ↓ success
systemd continues
```

Trong lab này chúng ta có thể dùng:

```ini
Type=exec
```

nếu systemd của bạn đủ mới.

Nếu dùng distro rất cũ không hỗ trợ, dùng:

```ini
Type=simple
```

---

# 40. `Type=oneshot`

Dành cho action chạy xong rồi exit.

Ví dụ:

```ini
[Service]
Type=oneshot
ExecStart=/usr/local/sbin/prepare-app-data
```

Mental model:

```text
start
 ↓
command runs
 ↓
command exits
 ↓
action complete
```

Không phải long-running daemon.

Current docs nói `oneshot` chờ command hoàn tất; nếu không có `RemainAfterExit=yes`, nó sẽ chuyển trở lại inactive sau khi action hoàn thành. :chatgpt-content-reference{index="20"}

Ví dụ use case:

```text
one-time initialization
filesystem preparation
configuration generation
cleanup task
```

---

# 41. `RemainAfterExit=yes`

Với `oneshot`:

```ini
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/local/bin/setup
ExecStop=/usr/local/bin/teardown
```

Sau khi `setup` exit thành công:

```text
không có long-running process
```

nhưng unit vẫn có thể được coi:

```text
active (exited)
```

Điều này thường gây confusion:

```text
active
```

không luôn đồng nghĩa:

```text
có process đang chạy
```

---

# 42. `Type=forking`

Awareness only.

Traditional daemon đôi khi tự daemonize:

```text
parent starts
  ↓
forks
  ↓
parent exits
  ↓
child continues in background
```

Đó là model:

```ini
Type=forking
```

Current systemd guidance khuyến nghị modern projects tránh kiểu `Type=forking` nếu có thể; các modern types như simple/exec/notify thường dễ quản lý đáng tin cậy hơn. :chatgpt-content-reference{index="21"}

Java/Tomcat service chúng ta về sau nên ưu tiên foreground process dưới systemd, không cố tự background process bằng `&`.

---

# 43. Anti-pattern quan trọng: background trong `ExecStart`

Sai pattern thường gặp:

```ini
ExecStart=/bin/sh -c '/usr/bin/java -jar app.jar &'
```

Bạn đang:

```text
systemd
  ↓
shell
  ↓
background java
  ↓
shell exits
```

Systemd lifecycle tracking trở nên phức tạp/không đúng như mong muốn.

Systemd đã là supervisor.

Application nên chạy foreground:

```ini
ExecStart=/usr/bin/java -jar /opt/app/app.jar
```

Mental model:

> **Đừng daemonize thủ công một process mà service manager được sinh ra để quản lý.**

---

# 44. `Type=notify`

Supplementary.

Một sophisticated daemon có thể nói cho systemd:

```text
"I have completed initialization and I am ready."
```

qua systemd notification protocol.

Khi đó:

```ini
Type=notify
```

có thể phù hợp.

Khác với:

```text
process exists
```

nó có readiness signaling ở lifecycle level.

Tomcat mặc định không tự nhiên trở thành `Type=notify` chỉ vì là Java service; cần integration thích hợp.

---

# 45. `User=`

Ví dụ:

```ini
User=tomcat
```

có nghĩa process được execute với Unix identity `tomcat`.

Current systemd documentation định nghĩa `User=` và `Group=` là user/group mà system-service processes được execute dưới; nếu user không tồn tại tại startup, invocation sẽ fail. :chatgpt-content-reference{index="22"}

Mental model:

```text
systemd PID 1/root
        │
        │ prepares credentials
        ↓
service process
UID = tomcat
```

---

# 46. Tại sao `User=` quan trọng?

Nếu bỏ `User=` trong system service, nhiều system services mặc định có thể chạy với root identity.

Đối với Java application không cần root:

```text
running as root
```

tăng blast radius rất lớn.

Nếu application compromise:

```text
attacker → application privileges
```

Do đó:

```ini
User=myapp
Group=myapp
```

là foundation của least privilege.

Sau này Tomcat sẽ dùng dedicated service account vì lý do này.

---

# 47. `Group=`

Ví dụ:

```ini
Group=tomcat
```

đặt primary group.

Nếu không set `Group=`, systemd có thể sử dụng default group của user theo user database rules. :chatgpt-content-reference{index="23"}

Bạn vẫn phải hiểu Day 3:

```text
owner
group
mode bits
```

vì:

```text
User=tomcat
```

mà:

```text
/opt/app/app.jar
```

không readable bởi `tomcat` thì service fail.

---

# 48. Đây là nơi Day 3 và Day 4 gặp nhau

Giả sử:

```text
-rw------- root root app.jar
```

Service:

```ini
User=tomcat
ExecStart=/usr/bin/java -jar /opt/app/app.jar
```

Java process không thể đọc JAR.

Root cause:

```text
filesystem permissions
```

không phải:

```text
systemd bug
```

Troubleshooting cần check:

```bash
namei -l /opt/app/app.jar
ls -l /opt/app/app.jar
id tomcat
```

và xác định path traversal + file permissions.

---

# 49. Không dùng `sudo` bên trong `ExecStart`

Anti-pattern:

```ini
ExecStart=/usr/bin/sudo -u tomcat /usr/bin/java ...
```

Thông thường không cần.

Systemd đã có:

```ini
User=tomcat
Group=tomcat
```

Đúng model:

```text
systemd
  ↓ User=tomcat
java process
```

không phải:

```text
systemd
  ↓
sudo
  ↓
tomcat
  ↓
java
```

---

# 50. `WorkingDirectory=`

Ví dụ:

```ini
WorkingDirectory=/opt/myapp
```

Systemd sẽ set current working directory của executed process.

Current docs nói system services nếu không set `WorkingDirectory=` mặc định thường bắt đầu relative to system root; user services có default khác. :chatgpt-content-reference{index="24"}

Đây là nguyên nhân kinh điển của:

> “Chạy bằng tay được, chạy service fail.”

---

# 51. Ví dụ lỗi relative path

Application code:

```text
open("./config/app.conf")
```

Chạy tay:

```bash
cd /opt/myapp
./bin/app
```

thì:

```text
./config/app.conf
→ /opt/myapp/config/app.conf
```

Service nếu không set working directory:

```text
cwd = /
```

application tìm:

```text
/config/app.conf
```

và fail.

Fix có thể là:

```ini
WorkingDirectory=/opt/myapp
```

hoặc tốt hơn application dùng explicit config path.

---

# 52. Validation `WorkingDirectory`

Sau khi service chạy:

```bash
MAINPID=$(systemctl show -p MainPID --value day4-demo.service)
```

Check:

```bash
readlink "/proc/$MAINPID/cwd"
```

Expected:

```text
/opt/day4-systemd-lab
```

Đây là evidence tuyệt vời:

```text
unit directive
        ↓
runtime process state
        ↓
/proc evidence
```

---

# 53. `Environment=`

Ví dụ:

```ini
Environment="APP_ENV=production"
Environment="JAVA_HOME=/usr/lib/jvm/java-17-openjdk"
```

Systemd đưa các variable này vào environment của executed process. :chatgpt-content-reference{index="25"}

Khác Bash:

```bash
export APP_ENV=production
```

là environment của shell/session và descendants.

System service phải có environment từ chính execution context của service.

---

# 54. `Environment=` không làm shell expansion như bạn tưởng

Ví dụ:

```ini
Environment="BASE=/opt/app"
Environment="CONFIG=$BASE/config"
```

Đừng mặc định kỳ vọng:

```text
CONFIG=/opt/app/config
```

Current systemd docs nói `Environment=` không thực hiện shell-style variable expansion trong value; `$` không có shell meaning ở directive đó. :chatgpt-content-reference{index="26"}

Do đó config rõ ràng hơn:

```ini
Environment="BASE=/opt/app"
Environment="CONFIG=/opt/app/config"
```

---

# 55. `EnvironmentFile=`

Ví dụ:

```ini
EnvironmentFile=/etc/myapp.env
```

File:

```text
APP_ENV=production
APP_PORT=8080
LOG_LEVEL=INFO
```

Systemd đọc environment assignments từ file đó.

Current docs mô tả `EnvironmentFile=` chứa newline-separated variable assignments; blank/comment lines được bỏ qua. :chatgpt-content-reference{index="27"}

Điều này rất hữu ích vì tách:

```text
service lifecycle definition
```

khỏi:

```text
environment-specific configuration
```

---

# 56. Environment separation cho Java

Ví dụ service file ổn định:

```ini
[Service]
ExecStart=/usr/bin/java -jar /opt/myapp/myapp.jar
EnvironmentFile=/etc/myapp/myapp.env
```

DEV:

```text
APP_PROFILE=dev
DB_HOST=dev-db
```

PROD:

```text
APP_PROFILE=prod
DB_HOST=prod-db
```

Bạn không cần copy/paste toàn bộ `.service` cho mỗi environment.

Đây là foundation cho Day 13:

```text
Externalized configuration
environment variables
profiles
```

---

# 57. Security warning: `Environment=` không phải secret vault

Systemd documentation hiện hành cảnh báo rõ rằng environment variables **không phù hợp để truyền secret** như passwords/key material, do khả năng exposure và propagation. :chatgpt-content-reference{index="28"}

Do đó không nên coi:

```ini
Environment="DB_PASSWORD=superSecret"
```

là một secret-management design tốt chỉ vì file service permission chặt.

Ngày 14 chúng ta sẽ học secrets handling sâu hơn.

Trong Day 4 lab:

```text
chỉ dùng non-sensitive demonstration values
```

---

# 58. Inspect environment thực tế của service

Lấy Main PID:

```bash
MAINPID=$(systemctl show day4-demo.service -p MainPID --value)
```

Sau đó:

```bash
sudo tr '\0' '\n' < "/proc/$MAINPID/environ"
```

Hoặc chỉ lấy safe fields:

```bash
sudo tr '\0' '\n' < "/proc/$MAINPID/environ" |
grep -E '^(DAY4_|PATH=)'
```

Đây nối kiến thức Module 1.

Nhớ caveat đã học:

`/proc/PID/environ` chủ yếu phản ánh environment process được exec với, không phải secret-safe interface.

---

# 59. `PATH` của service khác shell

Interactive SSH shell có thể:

```text
PATH=/home/student/.local/bin:/usr/local/bin:/usr/bin:/bin
```

System systemd manager có environment hạn chế/predictable hơn.

Current systemd documentation mô tả một standard search environment/path cho system services thay vì mặc nhiên import toàn bộ interactive shell environment. :chatgpt-content-reference{index="29"}

Do đó:

```text
works in terminal
```

không chứng minh:

```text
works in service
```

---

# 60. `[Install]`

Ví dụ:

```ini
[Install]
WantedBy=multi-user.target
```

Section này chủ yếu được sử dụng khi:

```bash
systemctl enable ...
```

Điểm rất quan trọng:

```text
[Install]
```

không có nghĩa:

```text
service đang active
```

Nó mô tả cách unit được **installed into dependency structure** khi enable.

---

# 61. `systemctl enable`

```bash
sudo systemctl enable day4-demo.service
```

Nó **không mặc định start service ngay**.

Current systemd documentation nói `enable` tạo symlinks theo `[Install]`; enablement và starting là hai operation độc lập. :chatgpt-content-reference{index="30"}

Nếu unit có:

```ini
[Install]
WantedBy=multi-user.target
```

enable thường tạo dạng:

```text
/etc/systemd/system/multi-user.target.wants/day4-demo.service
    ↓
/etc/systemd/system/day4-demo.service
```

Bạn có thể inspect:

```bash
ls -l /etc/systemd/system/multi-user.target.wants/
```

---

# 62. `WantedBy=` bị hiểu sai rất nhiều

Không nên học:

```text
WantedBy=multi-user.target
=
run AFTER multi-user.target
```

Sai mental model.

`WantedBy=` khi enable tạo reverse `Wants=` relationship từ target tới service. Systemd documentation xác nhận `WantedBy=` tạo symlink trong target's `.wants/` directory. :chatgpt-content-reference{index="31"}

Conceptually:

```text
multi-user.target
       │
       │ Wants
       ↓
day4-demo.service
```

Nó nói:

> Khi target được activated, target muốn kéo service này vào transaction.

Ordering là khái niệm khác.

---

# 63. Target là gì?

`.target` có thể hiểu gần như:

```text
logical synchronization/grouping point
```

Ví dụ:

```text
multi-user.target
network.target
graphical.target
```

Systemd documentation nói target units dùng để group units và tạo synchronization points trong boot/shutdown. :chatgpt-content-reference{index="32"}

`multi-user.target` có vai trò tương tự một normal multi-user system state trong classic init concepts.

---

# 64. `enable` và boot

Mental model:

```text
[Install]
WantedBy=multi-user.target
           │
           │ systemctl enable
           ↓
symlink under:
multi-user.target.wants/
           │
           │ next boot
           ↓
systemd activates multi-user.target
           │
           │ target Wants service
           ↓
day4-demo.service scheduled for activation
```

Đây là service startup requirement trong syllabus.

---

# 65. `systemctl enable --now`

Nếu muốn:

```text
enable
+
start now
```

có thể:

```bash
sudo systemctl enable --now day4-demo.service
```

`--now` kết hợp enablement change với immediate runtime action.

Nhưng khi học, nên ban đầu thực hiện riêng:

```bash
sudo systemctl enable day4-demo.service
systemctl is-enabled day4-demo.service

sudo systemctl start day4-demo.service
systemctl is-active day4-demo.service
```

để hiểu hai dimension.

---

# 66. `systemctl disable`

```bash
sudo systemctl disable day4-demo.service
```

gỡ các enablement links liên quan.

Điều cực kỳ quan trọng:

```text
disable ≠ stop
```

Service có thể:

```text
active + disabled
```

sau command.

Current systemd docs cũng nhấn mạnh enable/disable không đồng nghĩa immediate runtime start/stop. :chatgpt-content-reference{index="33"}

Nếu thật sự muốn cả hai:

```bash
sudo systemctl disable --now day4-demo.service
```

Nhưng hiểu rõ blast radius trước khi dùng trên production.

---

# 67. `is-active`

```bash
systemctl is-active day4-demo.service
```

Ví dụ:

```text
active
```

Exit status cũng phù hợp cho scripting.

Current docs định nghĩa `is-active` trả thành công nếu unit active. :chatgpt-content-reference{index="34"}

Ví dụ:

```bash
if systemctl is-active --quiet day4-demo.service; then
    echo "service active"
fi
```

Không nên parse màu/output của:

```bash
systemctl status
```

trong script.

---

# 68. `is-enabled`

```bash
systemctl is-enabled day4-demo.service
```

Có thể output:

```text
enabled
disabled
static
masked
indirect
...
```

Không nên coi world chỉ có:

```text
enabled
disabled
```

Current systemd có nhiều unit-file states. :chatgpt-content-reference{index="35"}

---

# 69. `static`

Một service có thể:

```text
static
```

Điều đó không có nghĩa:

```text
broken
```

Nó thường có nghĩa unit không cung cấp normal `[Install]` enablement information.

Nó vẫn có thể:

```text
được start thủ công
được dependency khác kéo vào
được socket/timer kích hoạt
```

Vì vậy:

```text
static ≠ disabled-with-error
```

---

# 70. Supplementary — `mask`

`mask` mạnh hơn `disable`.

```bash
sudo systemctl mask app.service
```

thường làm unit không thể được started theo cách bình thường, kể cả bằng dependency/manual start, bằng cách đặt mask trong unit load path.

Để phục hồi:

```bash
sudo systemctl unmask app.service
```

Mental model:

```text
disable:
"don't hook this into normal enablement"

mask:
"do not allow this unit to start"
```

Không mask production service nếu không có change/rollback plan.

---

# 71. Unit file locations

Đối với **system units**, important paths thường gồm:

```text
/etc/systemd/system/
/run/systemd/system/
/usr/lib/systemd/system/
```

Trên một số distro có:

```text
/lib/systemd/system/
```

Administrator-created system units thuộc:

```text
/etc/systemd/system/
```

Theo current systemd unit-load-path documentation, `/etc/systemd/system` là location dành cho system units do administrator tạo và có precedence cao hơn vendor unit directories. :chatgpt-content-reference{index="36"}

---

# 72. Vendor unit vs administrator unit

Ví dụ package cài:

```text
/usr/lib/systemd/system/tomcat.service
```

Admin muốn thay đổi memory option.

Anti-pattern:

```bash
sudo vim /usr/lib/systemd/system/tomcat.service
```

Tại sao?

Package upgrade có thể thay file vendor.

Tốt hơn dùng:

```text
/etc/systemd/system/
```

override/drop-in.

---

# 73. `systemctl cat`

Trước khi sửa service:

```bash
systemctl cat ssh.service
```

hoặc:

```bash
systemctl cat tomcat.service
```

Command này giúp nhìn effective unit fragments/drop-ins.

Đây là inspection-before-change.

Đừng thấy:

```text
/usr/lib/systemd/system/foo.service
```

rồi vội sửa file đó.

---

# 74. Supplementary — drop-in override

Ví dụ vendor unit:

```text
/usr/lib/systemd/system/myapp.service
```

Admin override:

```text
/etc/systemd/system/myapp.service.d/override.conf
```

Ví dụ:

```ini
[Service]
Environment="APP_ENV=production"
```

Systemd merge drop-in vào main unit.

Current documentation xác nhận `foo.service.d/*.conf` được merged vào main unit; `/etc` drop-ins có precedence cao hơn `/run` và `/usr/lib`. :chatgpt-content-reference{index="37"}

---

# 75. `systemctl edit`

Preferred workflow cho vendor service:

```bash
sudo systemctl edit myapp.service
```

sau đó viết override.

Ví dụ:

```ini
[Service]
EnvironmentFile=/etc/myapp/myapp.env
```

Ưu điểm:

```text
vendor unit remains package-owned
local change explicit
upgrade safer
rollback easier
```

---

# 76. Dependency: hai câu hỏi hoàn toàn khác nhau

Đây là phần khó nhưng cực kỳ quan trọng.

Giả sử:

```text
app.service depends on database.service
```

Có hai câu hỏi khác nhau:

### Câu hỏi A

> Khi start app, database có được kéo vào start transaction không?

Đây là:

```text
requirement dependency
```

ví dụ:

```text
Wants=
Requires=
```

### Câu hỏi B

> Nếu cả hai đang được start, cái nào phải start trước?

Đây là:

```text
ordering dependency
```

ví dụ:

```text
After=
Before=
```

Hai khái niệm **orthogonal**.

Current systemd documentation nói rõ requirement dependencies và ordering dependencies là độc lập. :chatgpt-content-reference{index="38"}

---

# 77. `After=` không tự start unit kia

Ví dụ:

```ini
[Unit]
After=postgresql.service
```

Không có nghĩa chắc chắn:

```text
systemd sẽ start PostgreSQL vì app cần nó
```

`After=` chỉ nói:

> Nếu cả app và PostgreSQL đều nằm trong transaction, order app sau PostgreSQL.

Mental model:

```text
After=
=
ORDERING
```

không phải:

```text
PULL IN
```

---

# 78. `Wants=`

Ví dụ:

```ini
Wants=postgresql.service
```

nghĩa:

```text
khi unit này được activated,
hãy cố kéo postgresql.service vào nữa
```

Đây là relationship tương đối weak.

Nếu wanted unit fail, wanting unit có thể vẫn tiếp tục tùy ordering/situation.

Current documentation gọi `Wants=` là recommended loose coupling trong nhiều trường hợp. :chatgpt-content-reference{index="39"}

---

# 79. `Requires=`

```ini
Requires=postgresql.service
```

mạnh hơn `Wants=`.

Nếu app được activated:

```text
PostgreSQL cũng được pulled in
```

và failure/stop relationships có semantics mạnh hơn.

Nhưng:

```text
Requires=postgresql.service
```

**không tự nhiên có nghĩa**:

```text
wait for PostgreSQL before app
```

Cần ordering nếu đó là intention:

```ini
Requires=postgresql.service
After=postgresql.service
```

Current docs nói requirement dependencies không quyết định startup ordering; `After=`/`Before=` phải được cấu hình riêng. :chatgpt-content-reference{index="40"}

---

# 80. `After=` không có nghĩa “healthy”

Ngay cả:

```ini
Requires=postgresql.service
After=postgresql.service
```

không đảm bảo:

```text
database has accepted all connections
schema ready
application credentials valid
migration complete
```

Nó chỉ đảm bảo lifecycle ordering theo systemd's definition of unit startup completion.

Đây là distinction:

```text
dependency/order
```

vs:

```text
application readiness
```

Rất quan trọng cho distributed systems.

---

# 81. `Before=`

Nếu:

```ini
Before=other.service
```

và cả hai units được start:

```text
current unit starts before other.service
```

`Before=` là inverse ordering của `After=`. :chatgpt-content-reference{index="41"}

Đừng viết cả:

```ini
After=foo.service
Before=foo.service
```

cho cùng relationship — đó là contradiction/order cycle territory.

---

# 82. Ví dụ dependency đúng mental model

```ini
[Unit]
Description=Order API
Wants=network-online.target
After=network-online.target
```

Interpretation:

```text
Wants=
try to pull target into activation

After=
order app after target
```

Hai directives trả lời hai câu hỏi khác nhau.

Lưu ý: `network-online.target` cũng không phải universal proof rằng mọi remote network dependency của bạn reachable; network readiness là một chủ đề có nhiều nuance, và Day 5 sẽ học networking.

---

# 83. `WantedBy=multi-user.target`

Khi enable:

```text
multi-user.target
      │
      └── Wants app.service
```

Vì target units có default ordering behavior liên quan dependencies, việc activation có thể được ordering phù hợp quanh target; nhưng tuyệt đối không diễn giải simplistically:

```text
WantedBy=X
=
After=X
```

Đây là hai mechanism khác nhau. :chatgpt-content-reference{index="42"}

---

# 84. Service startup at boot — mental model đầy đủ

Một simplified boot flow:

```text
Kernel boots
    │
    ↓
systemd becomes PID 1
    │
    ↓
systemd resolves default target/dependency graph
    │
    ↓
targets pull required/wanted units
    │
    ↓
enabled service enters transaction
    │
    ↓
dependencies/order evaluated
    │
    ↓
execution context created
    │
    ├── UID/GID
    ├── cwd
    ├── environment
    └── resource/security settings
    │
    ↓
ExecStart
    │
    ↓
main process
    │
    ↓
systemd tracks lifecycle
```

Đó mới là nghĩa sâu của:

```text
service startup
```

---

# 85. `Restart=`

Ví dụ:

```ini
Restart=on-failure
```

Systemd có thể tự động restart service khi process fail theo policy.

Current systemd docs định nghĩa các policy như:

```text
no
always
on-success
on-failure
on-abnormal
on-abort
on-watchdog
```

và hiện khuyến nghị `on-failure` cho nhiều long-running services để tăng reliability. :chatgpt-content-reference{index="43"}

---

# 86. `Restart=no`

Default:

```ini
Restart=no
```

Nếu process crash:

```text
service → failed/inactive
```

systemd không tự start lại chỉ vì service tồn tại.

---

# 87. `Restart=on-failure`

Ví dụ:

```ini
Restart=on-failure
```

phù hợp với nhiều long-running service.

Concept:

```text
unexpected failure
      ↓
systemd observes process exit
      ↓
restart policy says recover
      ↓
wait RestartSec
      ↓
start new process
```

Nhưng controlled:

```bash
systemctl stop service
```

không bị hiểu như application failure để lập tức undo stop. Current docs explicitly exclude service-manager-requested stop/restart from normal auto-restart behavior. :chatgpt-content-reference{index="44"}

---

# 88. `Restart=always`

```ini
Restart=always
```

rộng hơn.

Nếu application cleanly exits:

```text
systemd có thể start lại
```

Điều này phù hợp cho một số daemon nhưng có thể không phù hợp nếu process được phép tự kết thúc hợp lệ.

Không chọn:

```text
always
```

chỉ vì:

> “restart nhiều hơn chắc an toàn hơn.”

Policy phải phản ánh application semantics.

---

# 89. `RestartSec=`

Ví dụ:

```ini
RestartSec=5s
```

đặt delay trước restart attempt.

Current systemd supports time values như:

````text
5s
30s
2min
``` :chatgpt-content-reference{index="45"}


Tại sao không restart ngay lập tức liên tục?

Nếu app failure do:

```text
DB down
bad config
missing file
port occupied
````

restart 100 lần/giây:

```text
không sửa root cause
```

chỉ tạo:

```text
log storm
CPU/process churn
noise
```

---

# 90. Restart loop và start rate limit

Systemd có rate limiting để ngăn một broken service restart vô hạn ở tốc độ cao.

Bạn có thể thấy:

```text
Start request repeated too quickly
```

hoặc unit hit start limit.

`reset-failed` cũng reset relevant restart/start rate counters. :chatgpt-content-reference{index="46"}

Important operational principle:

> Nếu service restart loop, đừng chỉ tăng limits để che lỗi. Tìm root cause.

---

# 91. `reset-failed`

Suppose:

```text
day4-demo.service → failed
```

Sau khi fix:

```bash
sudo systemctl reset-failed day4-demo.service
```

có thể clear recorded failed state/rate-limit counters.

Current docs nói failed state và recorded failure info được giữ để introspection cho tới khi stop/restart/reset; `reset-failed` cũng reset per-unit start/restart counters. :chatgpt-content-reference{index="47"}

Nhưng:

```text
reset-failed ≠ fix
```

Nếu executable vẫn missing:

```text
reset-failed
start
→ fails again
```

---

# 92. `ExecStartPre=` và `ExecStartPost=`

Supplementary nhưng rất hữu ích.

Ví dụ:

```ini
ExecStartPre=/usr/bin/test -r /etc/myapp/app.conf
ExecStart=/usr/bin/java -jar /opt/myapp/app.jar
ExecStartPost=/usr/local/bin/register-health-check
```

Flow:

```text
ExecStartPre
   ↓ success
ExecStart
   ↓
ExecStartPost
```

Current docs nói multiple `ExecStartPre=`/`ExecStartPost=` lines chạy serially; nếu non-ignored pre command fail, startup fails và main `ExecStart=` không chạy. :chatgpt-content-reference{index="48"}

---

# 93. Không dùng `ExecStartPre=` cho long-running daemon

Ví dụ sai:

```ini
ExecStartPre=/opt/helper-daemon &
ExecStart=/opt/main-app
```

`ExecStartPre=` được thiết kế cho pre-start action, không phải background worker lâu dài.

Current docs explicitly nói long-running processes không nên được started từ `ExecStartPre=`; processes còn lại từ pre-start commands sẽ bị cleanup trước khi next service process được run. :chatgpt-content-reference{index="49"}

---

# 94. `ExecStop=`

Có thể define:

```ini
ExecStop=/opt/myapp/bin/graceful-stop
```

Nó nên là command:

```text
request clean termination
```

không phải arbitrary post-cleanup logic.

Postmortem cleanup phù hợp hơn với:

```ini
ExecStopPost=
```

theo current systemd guidance. :chatgpt-content-reference{index="50"}

---

# 95. Main process và child process

Suppose:

```text
myapp.service
│
├── MainPID 9000
└── helper PID 9005
```

`systemctl stop` không đơn giản chỉ có:

```text
kill 9000
```

Systemd có unit/cgroup-level process management.

Điều này là lý do systemd lifecycle management tốt hơn việc admin thủ công giữ một PID.

---

# 96. `systemctl list-units`

```bash
systemctl list-units --type=service
```

trả các loaded service units.

Điểm quan trọng:

```text
list-units
```

không phải comprehensive catalog của mọi unit file installed trên disk.

Current docs phân biệt loaded units với installed unit files. :chatgpt-content-reference{index="51"}

---

# 97. `systemctl list-unit-files`

Muốn xem unit files:

```bash
systemctl list-unit-files --type=service
```

Output kiểu:

```text
UNIT FILE             STATE
ssh.service           enabled
foo.service           disabled
bar.service           static
```

Mental distinction:

```text
list-units
→ runtime/load view

list-unit-files
→ installed unit definitions + enablement
```

---

# 98. `systemctl is-failed`

```bash
systemctl is-failed app.service
```

Nếu:

```text
failed
```

thì exit status phù hợp cho scripting.

Cũng có:

```bash
systemctl --failed
```

để xem failed units.

Trong incident:

```bash
systemctl --failed
```

có thể giúp xác định scope rộng hơn:

```text
một service?
nhiều services?
```

---

# 99. `systemctl daemon-reload` khi nào cần?

Thông thường sau:

```text
create unit file
edit unit file
create/remove manual drop-in
manual dependency symlink changes
```

Ví dụ:

```bash
sudo vim /etc/systemd/system/myapp.service
sudo systemctl daemon-reload
```

Current documentation explicitly warns on-disk unit changes có thể không khớp manager view cho tới khi reload. :chatgpt-content-reference{index="52"}

---

# 100. `systemd-analyze verify`

**Supplementary / Beyond explicit syllabus**, nhưng rất nên dùng trước khi activate unit mới.

```bash
sudo systemd-analyze verify /etc/systemd/system/myapp.service
```

Nó có thể phát hiện:

```text
unknown directive
wrong section
missing referenced dependency
ExecStart executable missing/not executable
```

Current documentation mô tả `systemd-analyze verify FILE...` dùng để load/validate units và report nhiều loại lỗi cấu hình. :chatgpt-content-reference{index="53"}

Operational pattern:

```text
edit
 ↓
syntax/static validation
 ↓
daemon-reload
 ↓
start/restart
 ↓
runtime validation
```

Đừng đảo thành:

```text
edit
 ↓
restart production
 ↓
see what happens
```

---

# 101. Status validation: human-readable

```bash
systemctl status day4-demo.service --no-pager
```

Bạn cần đọc:

```text
Loaded
Active
Main PID
Tasks
CGroup
recent messages
```

Không chỉ nhìn:

```text
green dot
```

---

# 102. State validation: machine-oriented

```bash
systemctl show day4-demo.service \
  -p LoadState \
  -p ActiveState \
  -p SubState \
  -p UnitFileState \
  -p MainPID \
  -p ExecMainStatus
```

Ví dụ:

```text
LoadState=loaded
ActiveState=active
SubState=running
UnitFileState=enabled
MainPID=8201
ExecMainStatus=0
```

Đây là excellent evidence.

---

# 103. Minimal `journalctl` preview

`journalctl` chính thức nằm ở **Day 6**, không phải Day 4. Syllabus đặt logging với `journalctl` vào Day 6. :chatgpt-content-reference{index="54"}

Nhưng syllabus Day 4 yêu cầu **recover a failed service**, nên chúng ta cần một preview tối thiểu:

```bash
sudo journalctl -u day4-demo.service -n 30 --no-pager
```

Ý nghĩa:

```text
-u
→ unit

-n 30
→ latest 30 entries

--no-pager
→ print directly
```

Chúng ta chưa học journal architecture/filtering/time ranges đầy đủ cho tới Day 6.

---

# 104. Vì sao systemd logs stdout/stderr hữu ích?

Service không có terminal interactive như bạn.

Nếu application viết:

```text
stdout
stderr
```

default service configuration trên normal systemd systems thường đưa chúng vào journald infrastructure.

Do đó:

```bash
journalctl -u service
```

trở thành source evidence rất quan trọng.

Sau này Java/Tomcat còn có application-specific log files.

---

# 105. Guided Lab — tạo systemd service thực sự

Lab này **thay đổi system-level configuration**:

```text
/opt/day4-systemd-lab
/etc/day4-demo.env
/etc/systemd/system/day4-demo.service
```

và sử dụng `sudo`.

Chỉ thực hiện trên **lab VM / disposable Linux host**, không làm trên production.

Chúng ta sẽ không cài package trong lab này vì package management thuộc Module 3.

---

# 106. Lab prerequisites

Trước tiên:

```bash
ps -p 1 -o pid,comm,args
```

Expected invariant:

```text
PID 1 phải là systemd
```

Tiếp:

```bash
systemctl --version
```

Sau đó:

```bash
systemctl is-system-running
```

Có thể output:

```text
running
```

Trong container hoặc WSL không bật systemd, lab có thể không phù hợp.

---

# 107. Lab safety pre-check

Xác nhận unit chưa tồn tại:

```bash
systemctl status day4-demo.service --no-pager
```

Expected có thể:

```text
Unit day4-demo.service could not be found
```

Kiểm tra paths chưa có:

```bash
ls -ld /opt/day4-systemd-lab /etc/day4-demo.env \
  /etc/systemd/system/day4-demo.service 2>/dev/null
```

Nếu các paths đã chứa dữ liệu của lab cũ, **không overwrite mù quáng**.

Inspect trước.

---

# 108. Tạo lab directory

```bash
sudo install -d -m 0755 /opt/day4-systemd-lab
```

Verify:

```bash
ls -ld /opt/day4-systemd-lab
```

Expected invariant:

```text
directory exists
permissions allow service process to traverse/read
```

---

# 109. Tạo worker script

```bash
sudo tee /opt/day4-systemd-lab/day4-worker.sh >/dev/null <<'EOF'
#!/usr/bin/env bash

set -u

trap 'echo "day4-demo: received termination signal, exiting"; exit 0' TERM INT

echo "day4-demo: started pid=$$ user=$(id -un) mode=${DAY4_MODE:-unset} source=${DAY4_SOURCE:-unset}"

while true; do
    echo "day4-demo: heartbeat pid=$$ mode=${DAY4_MODE:-unset} message=${DAY4_MESSAGE:-unset}"
    sleep "${DAY4_INTERVAL:-10}" &
    wait $!
done
EOF
```

Make executable:

```bash
sudo chmod 0755 /opt/day4-systemd-lab/day4-worker.sh
```

Inspect:

```bash
ls -l /opt/day4-systemd-lab/day4-worker.sh
```

Syntax test:

```bash
bash -n /opt/day4-systemd-lab/day4-worker.sh
```

Expected:

```text
no output
exit code 0
```

Check:

```bash
echo $?
```

Expected:

```text
0
```

---

# 110. Vì sao dùng script này?

Nó làm ba việc có giá trị học tập.

Thứ nhất:

```text
long-running
```

nên systemd có process để manage.

Thứ hai:

```text
prints environment
```

để kiểm tra `Environment=` và `EnvironmentFile=`.

Thứ ba:

```text
handles TERM/INT
```

để nối kiến thức signals từ Module 1.

---

# 111. Tạo EnvironmentFile

```bash
sudo tee /etc/day4-demo.env >/dev/null <<'EOF'
DAY4_MODE=from-environment-file
DAY4_INTERVAL=10
DAY4_MESSAGE="hello from EnvironmentFile"
EOF
```

Permissions:

```bash
sudo chmod 0644 /etc/day4-demo.env
```

Inspect:

```bash
cat /etc/day4-demo.env
```

Không có secret trong file này.

---

# 112. Chọn service identity

Chúng ta sẽ dùng **user hiện tại** để tránh phải tạo thêm OS account trong lab này.

Lấy:

```bash
LAB_USER="$(id -un)"
LAB_GROUP="$(id -gn)"

printf 'User=%s\nGroup=%s\n' "$LAB_USER" "$LAB_GROUP"
```

Ví dụ:

```text
User=student
Group=student
```

Đây vẫn tốt hơn chạy worker bằng root.

Trong production, application thường nên có **dedicated service account**, ví dụ:

```text
tomcat
myapp
postgres
```

---

# 113. Tạo service unit

Chạy:

```bash
LAB_USER="$(id -un)"
LAB_GROUP="$(id -gn)"

sudo tee /etc/systemd/system/day4-demo.service >/dev/null <<EOF
[Unit]
Description=Day 4 systemd learning service

[Service]
Type=exec
User=$LAB_USER
Group=$LAB_GROUP
WorkingDirectory=/opt/day4-systemd-lab
Environment="DAY4_MODE=from-unit-file"
Environment="DAY4_SOURCE=Environment-directive"
EnvironmentFile=/etc/day4-demo.env
ExecStart=/opt/day4-systemd-lab/day4-worker.sh
Restart=on-failure
RestartSec=3s

[Install]
WantedBy=multi-user.target
EOF
```

Inspect:

```bash
sudo cat /etc/systemd/system/day4-demo.service
```

---

# 114. Nếu `Type=exec` không được hỗ trợ

Nếu bạn đang dùng một systemd rất cũ và verify báo:

```text
Unknown service type
```

đổi:

```ini
Type=exec
```

thành:

```ini
Type=simple
```

Trên modern Ubuntu/RHEL/Debian installations, `exec` thường được hỗ trợ.

---

# 115. Phân tích unit chúng ta vừa tạo

```ini
[Unit]
Description=Day 4 systemd learning service
```

Metadata.

```ini
Type=exec
```

Systemd chờ successful `exec` point.

```ini
User=<lab-user>
Group=<lab-group>
```

Least-privileged execution context.

```ini
WorkingDirectory=/opt/day4-systemd-lab
```

Set `cwd`.

```ini
Environment="DAY4_MODE=from-unit-file"
```

Set environment direct.

```ini
EnvironmentFile=/etc/day4-demo.env
```

Load file environment.

```ini
ExecStart=/opt/day4-systemd-lab/day4-worker.sh
```

Start long-running process.

```ini
Restart=on-failure
RestartSec=3s
```

Automatic failure recovery.

```ini
WantedBy=multi-user.target
```

Enablement hook.

---

# 116. Một experiment tinh tế về environment precedence

Unit đặt:

```text
DAY4_MODE=from-unit-file
```

Environment file đặt:

```text
DAY4_MODE=from-environment-file
```

Current systemd environment assembly rules place `EnvironmentFile=` values after `Environment=` values; nếu cùng variable được đặt từ nhiều sources, later source theo precedence rules thắng. :chatgpt-content-reference{index="55"}

Ta dự đoán:

```text
DAY4_MODE=from-environment-file
```

sẽ là effective value.

Nhưng đừng tin prediction.

Ta sẽ verify runtime.

---

# 117. Validate unit trước activation

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/day4-demo.service
```

Nếu không có warning/error:

```text
static validation passed
```

Không có output thường là tín hiệu tốt.

Nhưng:

```text
verify passed
```

không đồng nghĩa:

```text
application runtime guaranteed success
```

Nó chỉ là một lớp validation.

---

# 118. `daemon-reload`

Bây giờ:

```bash
sudo systemctl daemon-reload
```

Điều này cho systemd manager biết unit mới.

Check:

```bash
systemctl status day4-demo.service --no-pager
```

Có thể thấy:

```text
Loaded: loaded (...; disabled; ...)
Active: inactive (dead)
```

Đây là trạng thái rất đáng học:

```text
loaded
disabled
inactive
```

Unit hoàn toàn hợp lệ, nhưng chưa start và chưa enable.

---

# 119. Start nhưng chưa enable

```bash
sudo systemctl start day4-demo.service
```

Check active:

```bash
systemctl is-active day4-demo.service
```

Expected:

```text
active
```

Check enabled:

```bash
systemctl is-enabled day4-demo.service
```

Expected ở bước này:

```text
disabled
```

Bạn vừa chứng minh thực nghiệm:

```text
running ≠ enabled
```

---

# 120. Inspect status

```bash
systemctl status day4-demo.service --no-pager
```

Tìm:

```text
Loaded:
Active:
Main PID:
CGroup:
```

Ví dụ:

```text
Loaded: loaded (...; disabled; ...)
Active: active (running)
Main PID: 21345
CGroup: /system.slice/day4-demo.service
```

PID thật của bạn sẽ khác.

---

# 121. Inspect service properties

```bash
systemctl show day4-demo.service \
  -p LoadState \
  -p ActiveState \
  -p SubState \
  -p UnitFileState \
  -p MainPID
```

Expected invariants:

```text
LoadState=loaded
ActiveState=active
SubState=running
UnitFileState=disabled
MainPID > 0
```

---

# 122. Nối sang Module 1

Lấy PID:

```bash
MAINPID="$(systemctl show day4-demo.service -p MainPID --value)"
echo "$MAINPID"
```

Inspect:

```bash
ps -p "$MAINPID" \
  -o pid,ppid,user,group,stat,lstart,etime,cmd
```

Bạn phải nhận ra:

```text
PID
PPID
USER
GROUP
STAT
CMD
```

không còn là khái niệm mới.

Systemd chỉ cung cấp layer quản lý ở trên.

---

# 123. Kiểm tra working directory

```bash
sudo readlink "/proc/$MAINPID/cwd"
```

Expected:

```text
/opt/day4-systemd-lab
```

Điều này chứng minh:

```ini
WorkingDirectory=/opt/day4-systemd-lab
```

đã trở thành actual process state.

---

# 124. Kiểm tra executable

```bash
sudo readlink "/proc/$MAINPID/exe"
```

Vì script dùng:

```text
#!/usr/bin/env bash
```

main executable cuối cùng có thể là Bash path.

Ví dụ:

```text
/usr/bin/bash
```

Đây là một bài học:

```text
ExecStart path
```

không luôn bằng:

```text
/proc/PID/exe
```

nếu target là interpreted script với shebang.

---

# 125. Kiểm tra command line

```bash
sudo tr '\0' ' ' < "/proc/$MAINPID/cmdline"
echo
```

Bạn có thể thấy Bash/script path tương ứng.

Quan sát actual environment/runtime thay vì suy đoán.

---

# 126. Verify User/Group

```bash
ps -p "$MAINPID" -o user,group,pid,cmd
```

Expected:

```text
user/group = account bạn dùng trong User=/Group=
```

Không phải root.

Đây là least privilege evidence.

---

# 127. Verify environment

```bash
sudo tr '\0' '\n' < "/proc/$MAINPID/environ" |
grep '^DAY4_'
```

Expected:

```text
DAY4_SOURCE=Environment-directive
DAY4_MODE=from-environment-file
DAY4_INTERVAL=10
DAY4_MESSAGE=hello from EnvironmentFile
```

Order có thể khác.

Điểm quan trọng là value.

Bạn vừa chứng minh:

```text
systemd unit/config
       ↓
process environment
```

---

# 128. Xem minimal logs

```bash
sudo journalctl -u day4-demo.service \
  -n 20 \
  --no-pager
```

Bạn nên thấy:

```text
day4-demo: started pid=...
day4-demo: heartbeat pid=... mode=from-environment-file ...
```

Đây là **example output**, không phải literal invariant.

Expected invariant:

```text
service emitted heartbeat
runtime environment matches expected values
```

---

# 129. Enable service

Bây giờ:

```bash
sudo systemctl enable day4-demo.service
```

Check:

```bash
systemctl is-enabled day4-demo.service
```

Expected:

```text
enabled
```

Check active:

```bash
systemctl is-active day4-demo.service
```

Expected:

```text
active
```

---

# 130. Inspect enablement symlink

```bash
ls -l \
  /etc/systemd/system/multi-user.target.wants/day4-demo.service
```

Expected dạng:

```text
... day4-demo.service -> /etc/systemd/system/day4-demo.service
```

Bạn vừa nhìn thấy **physical implementation** của `enable`.

Đây không còn là lệnh magic.

---

# 131. Chứng minh `disable ≠ stop`

Disable:

```bash
sudo systemctl disable day4-demo.service
```

Check:

```bash
systemctl is-enabled day4-demo.service
```

Expected:

```text
disabled
```

Nhưng:

```bash
systemctl is-active day4-demo.service
```

Expected:

```text
active
```

Service vẫn đang chạy.

Rất quan trọng.

Enable lại để tiếp tục:

```bash
sudo systemctl enable day4-demo.service
```

---

# 132. Test controlled stop

Record PID:

```bash
OLD_PID="$(systemctl show day4-demo.service -p MainPID --value)"
echo "$OLD_PID"
```

Stop:

```bash
sudo systemctl stop day4-demo.service
```

Check:

```bash
systemctl is-active day4-demo.service
```

Expected:

```text
inactive
```

Check old PID:

```bash
ps -p "$OLD_PID"
```

Expected invariant:

```text
old process is gone
```

---

# 133. Observe graceful termination

Xem:

```bash
sudo journalctl -u day4-demo.service -n 20 --no-pager
```

Bạn có thể thấy:

```text
received termination signal, exiting
```

Đây nối trực tiếp:

```text
systemctl stop
       ↓
systemd stop lifecycle
       ↓
SIGTERM
       ↓
Bash trap
       ↓
clean exit
```

Module 1 và Module 2 đã nối thành một mental model duy nhất.

---

# 134. Start lại

```bash
sudo systemctl start day4-demo.service
```

New PID:

```bash
NEW_PID="$(systemctl show day4-demo.service -p MainPID --value)"

printf 'old=%s\nnew=%s\n' "$OLD_PID" "$NEW_PID"
```

Expected invariant:

```text
NEW_PID != OLD_PID
```

PID reuse theoretically possible theo thời gian, nhưng ngay trong immediate lab sequence bình thường PID sẽ khác.

---

# 135. Test `restart`

Record:

```bash
BEFORE_PID="$NEW_PID"
```

Restart:

```bash
sudo systemctl restart day4-demo.service
```

After:

```bash
AFTER_PID="$(systemctl show day4-demo.service -p MainPID --value)"

printf 'before=%s\nafter=%s\n' \
  "$BEFORE_PID" "$AFTER_PID"
```

Expected:

```text
new process lifecycle
```

Đây là concrete proof rằng restart không giống reload.

---

# 136. Thay environment nhưng chưa restart

Edit:

```bash
sudo sed -i \
  's/DAY4_MESSAGE=.*/DAY4_MESSAGE="configuration changed"/' \
  /etc/day4-demo.env
```

Check file:

```bash
cat /etc/day4-demo.env
```

Bây giờ **đừng restart**.

Inspect process environment:

```bash
MAINPID="$(systemctl show day4-demo.service -p MainPID --value)"

sudo tr '\0' '\n' < "/proc/$MAINPID/environ" |
grep '^DAY4_MESSAGE='
```

Bạn sẽ thấy old value.

Tại sao?

Vì existing process không tự nhiên inherit lại changed EnvironmentFile.

---

# 137. Có cần `daemon-reload` khi chỉ sửa EnvironmentFile?

Đây là nuance quan trọng.

`EnvironmentFile=` được đọc khi process được executed; thay đổi **nội dung file được tham chiếu** không giống thay đổi unit definition itself.

Bạn cần new process để đọc new values:

```bash
sudo systemctl restart day4-demo.service
```

Bạn không nhất thiết cần `daemon-reload` chỉ vì nội dung external EnvironmentFile đổi, miễn đường dẫn/directive của unit không đổi.

Sau restart:

```bash
MAINPID="$(systemctl show day4-demo.service -p MainPID --value)"

sudo tr '\0' '\n' < "/proc/$MAINPID/environ" |
grep '^DAY4_MESSAGE='
```

Expected:

```text
DAY4_MESSAGE=configuration changed
```

Đây là một practical distinction rất quan trọng.

---

# 138. Thay unit file

Bây giờ giả sử sửa:

```ini
RestartSec=3s
```

thành:

```ini
RestartSec=5s
```

Đây là actual unit-file change:

```bash
sudo sed -i \
  's/RestartSec=3s/RestartSec=5s/' \
  /etc/systemd/system/day4-demo.service
```

Validate:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/day4-demo.service
```

Sau đó:

```bash
sudo systemctl daemon-reload
```

Đây là correct workflow.

---

# 139. Kiểm tra systemd nhận config mới chưa

```bash
systemctl show day4-demo.service -p RestartUSec
```

Có thể output normalized:

```text
RestartUSec=5s
```

Hoặc format tương tự tùy version.

`systemctl show` hiển thị normalized runtime properties, nên property name đôi khi khác directive spelling. :chatgpt-content-reference{index="56"}

---

# 140. Failure Injection 1 — executable mất execute permission

**Đây chỉ thực hiện trên file lab.**

Trước failure:

```bash
systemctl is-active day4-demo.service
```

Collect baseline:

```bash
systemctl status day4-demo.service --no-pager
```

Stop:

```bash
sudo systemctl stop day4-demo.service
```

Remove executable permission:

```bash
sudo chmod 0644 /opt/day4-systemd-lab/day4-worker.sh
```

Verify:

```bash
ls -l /opt/day4-systemd-lab/day4-worker.sh
```

Expected:

```text
not executable
```

---

# 141. Static validation sẽ bắt được gì?

Run:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/day4-demo.service
```

Current systemd-analyze can detect Exec-style commands that are missing or not executable. :chatgpt-content-reference{index="57"}

Đây minh họa:

```text
pre-change validation
```

có thể ngăn incident.

Nhưng để học failure recovery, ta sẽ cố start.

---

# 142. Start failed service

```bash
sudo systemctl start day4-demo.service
```

Command có thể return error.

Check:

```bash
systemctl status day4-demo.service --no-pager
```

Expected:

```text
Active: failed
```

Không cần học thuộc exact numeric status trong Day 4.

---

# 143. Evidence-first troubleshooting

Không sửa ngay.

Collect:

```bash
systemctl show day4-demo.service \
  -p LoadState \
  -p ActiveState \
  -p SubState \
  -p Result \
  -p MainPID \
  -p ExecMainStatus
```

Sau đó:

```bash
sudo journalctl -u day4-demo.service \
  -n 30 \
  --no-pager
```

Rồi:

```bash
ls -l /opt/day4-systemd-lab/day4-worker.sh
```

Evidence chain:

```text
service failed
        ↓
ExecStart target identified
        ↓
target exists
        ↓
target not executable
        ↓
root cause confirmed
```

---

# 144. Recovery

Restore:

```bash
sudo chmod 0755 /opt/day4-systemd-lab/day4-worker.sh
```

Verify:

```bash
test -x /opt/day4-systemd-lab/day4-worker.sh &&
echo "executable"
```

Expected:

```text
executable
```

Static verify:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/day4-demo.service
```

Then:

```bash
sudo systemctl reset-failed day4-demo.service
sudo systemctl start day4-demo.service
```

---

# 145. Recovery validation

Check:

```bash
systemctl is-active day4-demo.service
```

Expected:

```text
active
```

Then:

```bash
systemctl status day4-demo.service --no-pager
```

Then PID:

```bash
MAINPID="$(systemctl show day4-demo.service -p MainPID --value)"

ps -p "$MAINPID" \
  -o pid,ppid,user,stat,lstart,etime,cmd
```

Then logs:

```bash
sudo journalctl -u day4-demo.service \
  -n 20 \
  --no-pager
```

Recovery chưa hoàn chỉnh chỉ vì:

```text
systemctl start returned 0
```

Cần validation evidence.

---

# 146. Failure Injection 2 — wrong User

Conceptual scenario.

Unit:

```ini
User=day4-user-does-not-exist
```

Khi systemd cố tạo execution context:

```text
cannot resolve/apply user identity
```

service start fail.

Troubleshooting:

```bash
systemctl status ...
systemctl show ...
id day4-user-does-not-exist
```

Không cần đoán:

> systemd corrupted.

Root cause:

```text
invalid execution identity
```

Current systemd docs xác nhận specified static `User=` phải tồn tại trước service startup, nếu không invocation fail. :chatgpt-content-reference{index="58"}

---

# 147. Failure Injection 3 — WorkingDirectory missing

Unit:

```ini
WorkingDirectory=/opt/does-not-exist
```

Service may fail trong execution setup trước application chạy.

Evidence:

```bash
systemctl status service
test -d /opt/does-not-exist
```

Root cause:

```text
working directory precondition invalid
```

Đây là case điển hình:

```text
application process never truly started
```

nhưng ticket có thể report:

> Application won't start.

---

# 148. Failure Injection 4 — missing EnvironmentFile

Unit:

```ini
EnvironmentFile=/etc/myapp.env
```

File bị xóa:

```text
/etc/myapp.env → missing
```

Startup có thể fail.

Nếu file thực sự optional, systemd supports syntax:

```ini
EnvironmentFile=-/etc/myapp.env
```

dấu `-` trước path yêu cầu ignore missing file.

Nhưng chỉ dùng optional semantics nếu application **thực sự có safe defaults**.

Đừng thêm `-` chỉ để làm lỗi biến mất.

---

# 149. Failure Injection 5 — application exits immediately

Suppose:

```ini
ExecStart=/opt/myapp/start
Restart=on-failure
```

`start` chạy rồi:

```text
exit 1
```

systemd:

```text
process exits
    ↓
failure recognized
    ↓
Restart=on-failure
    ↓
wait RestartSec
    ↓
retry
```

Nếu root cause persistent:

```text
repeat
```

Cuối cùng có thể hit start-rate limits.

---

# 150. Restart loop là symptom, không phải fix

Nếu status liên tục:

```text
activating (auto-restart)
```

đừng lập tức:

```text
increase RestartSec
increase StartLimitBurst
```

Đó chỉ thay behavior.

Phải investigate:

```text
why process exits?
```

Possible layers:

```text
binary
permissions
environment
config
port
filesystem
database
dependency
application bug
```

---

# 151. Troubleshooting workflow chuẩn

Khi nhận:

> `myapp.service` không chạy.

Tư duy:

```text
SYMPTOM
service unavailable
        ↓
EXPECTED
should be active/running
        ↓
SCOPE
one unit? many units? whole host?
        ↓
RECENT CHANGE
unit/config/package/permissions/reboot?
        ↓
UNIT STATE
Loaded / Active / SubState / Result
        ↓
UNIT DEFINITION
ExecStart/User/WorkingDirectory/Environment
        ↓
PROCESS
MainPID exists?
        ↓
EVIDENCE
status / journal / filesystem / /proc
        ↓
HYPOTHESES
        ↓
SAFEST TEST
        ↓
ROOT CAUSE
        ↓
L1 SAFE REMEDIATION
        ↓
RESTART/RECOVERY
        ↓
VALIDATION
        ↓
PRESERVE EVIDENCE
```

Đây là workflow bạn phải hình thành thành phản xạ.

---

# 152. Case: `Unit ... could not be found`

Symptom:

```text
Unit myapp.service could not be found.
```

Possible hypotheses:

```text
wrong unit name
unit file does not exist
unit file placed wrong directory
new unit not daemon-reloaded
typo in suffix
```

Evidence:

```bash
systemctl list-unit-files | grep myapp
```

```bash
find /etc/systemd/system \
     /usr/lib/systemd/system \
     /lib/systemd/system \
     -maxdepth 1 \
     -name 'myapp*' 2>/dev/null
```

Nếu file vừa tạo:

```bash
sudo systemctl daemon-reload
```

sau khi validate.

---

# 153. Case: `Loaded: bad-setting`

Đây thường là unit parsing/config problem.

Check:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/myapp.service
```

Common causes:

```text
directive misspelled
directive placed wrong section
invalid value
invalid quoting
unsupported directive on old systemd
```

Không restart liên tục trước khi sửa config.

---

# 154. Case: active nhưng app inaccessible

Status:

```text
Active: active (running)
```

nhưng user:

```text
"website down"
```

Đừng restart ngay chỉ vì user says app down.

Process/service layer đã pass.

Chuyển xuống stack:

```text
port listening?
network?
HTTP endpoint?
application logs?
database dependency?
```

Day 5 sẽ thêm:

```text
ss
curl
DNS
firewall
```

Day 14–16 thêm DB/application diagnosis.

---

# 155. Case: process chạy tay được nhưng systemd fail

Đây là một trong những case quan trọng nhất toàn khóa học.

Compare contexts:

| Interactive shell            | systemd service                     |
| ---------------------------- | ----------------------------------- |
| your login user              | `User=`                             |
| your groups                  | `Group=`                            |
| current `$PWD`               | `WorkingDirectory=`                 |
| shell `$PATH`                | service environment                 |
| exported vars                | `Environment=` / `EnvironmentFile=` |
| terminal attached            | usually no interactive TTY          |
| shell initialization         | usually not sourced                 |
| your ulimits/session context | service-specific context            |

Troubleshooting phải compare context, không compare mỗi command text.

---

# 156. `.bashrc` không phải service configuration

Suppose:

```bash
echo 'export APP_ENV=prod' >> ~/.bashrc
```

Bạn login:

```bash
echo "$APP_ENV"
```

works.

System service:

```text
APP_ENV missing
```

Không surprising.

Systemd không được thiết kế để chạy service bằng cách:

```text
interactive login
→ source .bashrc
→ execute command
```

Service config phải explicit.

---

# 157. Service không nên phụ thuộc interactive login

Production service phải hoạt động:

```text
sau reboot
không ai SSH vào
không shell profile được source
```

Nếu app chỉ chạy sau khi admin login rồi:

```bash
source ~/.bashrc
systemctl restart app
```

đó là configuration smell.

Execution context cần declarative và reproducible.

---

# 158. Service startup và HOME

Nếu application giả định:

```text
$HOME
```

phải kiểm tra service context.

`User=` không có nghĩa mọi login-shell initialization/state sẽ giống interactive session.

Java application nên ưu tiên explicit:

```text
config path
data path
log path
temporary path
```

thay vì hidden assumptions.

---

# 159. Security — service không nên chạy root nếu không cần

Bad:

```ini
[Service]
ExecStart=/usr/bin/java -jar /opt/app/app.jar
```

nếu không có `User=`, và system service default identity là root.

Better:

```ini
User=myapp
Group=myapp
```

sau đó cấp đúng filesystem/network permissions.

Principle:

```text
minimum privileges necessary
```

không phải:

```text
run root to avoid permission problems
```

---

# 160. Permission denied là evidence, không phải lý do dùng root

Nếu service account không đọc được:

```text
/opt/app/config
```

bad reaction:

```text
Change User=root
```

Better investigation:

```text
Which path?
Which operation?
Who owns it?
What permissions are required?
Can ownership/group/mode be corrected narrowly?
```

Fix root cause, không bypass access control.

---

# 161. Security — unit files cũng là privileged configuration

Nếu attacker sửa được:

```text
/etc/systemd/system/root-owned.service
```

họ có thể thay:

```ini
ExecStart=
```

và khiến PID 1 execute arbitrary command với service privileges.

Do đó unit files cho system services thường phải:

```text
root-owned
not writable by untrusted users
```

Check:

```bash
ls -l /etc/systemd/system/day4-demo.service
```

Expected lab:

```text
root root
```

---

# 162. Security — EnvironmentFile permission

Nếu EnvironmentFile kiểm soát:

```text
command arguments
paths
profiles
connections
```

nó cũng là config asset.

Trong production:

```text
ownership
permissions
change control
```

phải phù hợp.

Nhưng nhớ:

```text
chmod 600
```

không tự biến environment variable thành ideal secret store.

---

# 163. Supplementary — basic service hardening

Systemd có rất nhiều hardening controls:

```text
NoNewPrivileges=
PrivateTmp=
ProtectSystem=
ProtectHome=
CapabilityBoundingSet=
RestrictAddressFamilies=
SystemCallFilter=
```

Chúng ta **không áp dụng hàng loạt hôm nay**, vì hardening sai có thể làm application fail theo cách khó debug.

Nguyên tắc:

```text
understand application
baseline functionality
add one restriction
validate
observe
rollback if needed
```

Không copy-paste “maximum security” unit từ Internet.

---

# 164. Supplementary — `systemd-analyze security`

Có thể:

```bash
systemd-analyze security service.service
```

để xem một security exposure analysis.

Nhưng output là heuristic của systemd's configured sandboxing controls, không phải proof:

```text
service is secure
```

`security` command hiện có trong `systemd-analyze`. :chatgpt-content-reference{index="59"}

---

# 165. Operations — change discipline

Trước khi sửa unit production:

```text
1. Identify exact unit
2. Read existing unit and drop-ins
3. Record current state
4. Record MainPID/version
5. Back up/change through controlled override
6. Validate syntax
7. Understand restart impact
8. Prepare rollback
9. Apply change
10. Validate unit + process + application
```

Trong fresher role, nếu không có approved change:

```text
do not modify production service definitions
```

---

# 166. Snapshot current effective configuration

Useful:

```bash
systemctl cat myapp.service
```

Store evidence:

```bash
systemctl show myapp.service \
  -p LoadState \
  -p ActiveState \
  -p SubState \
  -p UnitFileState \
  -p MainPID
```

Nếu approved:

```bash
sudo cp \
  /etc/systemd/system/myapp.service \
  /etc/systemd/system/myapp.service.bak
```

Nhưng với vendor units, preferred model thường là:

```text
drop-in override
```

thay vì copying vendor unit unnecessarily.

---

# 167. Rollback mental model

Suppose change:

```text
RestartSec=5
→ RestartSec=30
```

và application operations degrade.

Rollback:

```text
restore previous unit/drop-in
        ↓
systemd-analyze verify
        ↓
daemon-reload
        ↓
restart if required/approved
        ↓
validate state/process/app
```

Rollback không chỉ là:

```text
copy old file back
```

vì systemd manager cần nhận config, và runtime process có thể cần restart.

---

# 168. Unit config vs runtime config

Hãy giữ ba lớp riêng:

```text
DISK
/etc/systemd/system/app.service

        ↓ daemon-reload

SYSTEMD MANAGER STATE
loaded unit definition

        ↓ start/restart

PROCESS RUNTIME
PID
UID
cwd
environment
open files
sockets
```

Một thay đổi ở lớp trên **không tự động** lan xuống process đã chạy.

Đây là một mental model cực mạnh.

---

# 169. Ví dụ: đổi `User=`

Disk:

```ini
User=olduser
```

service đang:

```text
PID 5000 → UID olduser
```

Bạn edit:

```ini
User=newuser
```

rồi:

```bash
systemctl daemon-reload
```

Process PID 5000 vẫn:

```text
UID olduser
```

Để new execution identity áp dụng cần lifecycle change:

```text
restart/new process
```

---

# 170. Ví dụ: đổi `WorkingDirectory=`

Same.

Disk changed:

```ini
WorkingDirectory=/new/path
```

`daemon-reload`:

```text
manager learns new config
```

existing process:

```text
cwd remains old path
```

Restart:

```text
new PID gets new cwd
```

---

# 171. Ví dụ: đổi `EnvironmentFile`

File contents changed:

```text
APP_ENV=v1
→
APP_ENV=v2
```

Existing process:

```text
still v1
```

Restart:

```text
new process gets v2
```

Environment không phải shared live-memory config.

---

# 172. `systemctl restart` và availability

Restart có availability impact.

Even if:

```text
restart takes only 2 seconds
```

traffic có thể fail trong window.

Trước production restart phải biết:

```text
load balancer?
multiple instances?
maintenance window?
active requests?
session impact?
database transactions?
rollback?
```

Day 18 sẽ formalize change/release controls.

---

# 173. `reload` có thể giảm downtime nhưng không phải universal

Nếu application supports safe reload:

```text
reload
```

có thể giữ process/lifecycle continuity tốt hơn.

Nhưng:

```text
not every application supports reload
```

và:

```text
not every setting is reloadable
```

Ví dụ changing Java JVM heap:

```text
-Xmx
```

thường yêu cầu new JVM process.

Không thể assume:

```text
systemctl reload
```

sẽ apply mọi setting.

---

# 174. Unit dependencies không thay thế health checks

Ví dụ:

```ini
After=postgresql.service
Requires=postgresql.service
```

PostgreSQL unit:

```text
active
```

nhưng DB có thể:

```text
reject credentials
database missing
schema unavailable
connection slots full
network problem
```

Systemd chỉ giải quyết service dependency/lifecycle level.

Application-layer validation vẫn cần.

---

# 175. L1 vs escalation

Một L1 fresher thường nên có khả năng:

| L1-safe khi có runbook/approval   | Có thể cần escalation                       |
| --------------------------------- | ------------------------------------------- |
| `status` / `show`                 | rewrite vendor service architecture         |
| `is-active` / `is-enabled`        | arbitrary root service changes              |
| inspect unit                      | change production service identity          |
| inspect MainPID                   | alter advanced dependencies                 |
| inspect safe environment fields   | change security sandboxing blindly          |
| controlled start/stop/restart     | repeated forced kills                       |
| daemon-reload after approved edit | persistent restart loops with unknown cause |
| recover obvious lab config error  | boot-order cycle                            |
| collect minimal journal evidence  | system-wide PID1/systemd instability        |

Command đơn giản không có nghĩa risk nhỏ.

---

# 176. Common mistake: sửa unit nhưng quên daemon-reload

Symptom:

```text
"I changed ExecStart but restart still seems to use old configuration."
```

Check:

```bash
systemctl status service
```

Systemd đôi khi cảnh báo unit changed on disk.

Correct:

```bash
sudo systemd-analyze verify /etc/systemd/system/service.service
sudo systemctl daemon-reload
sudo systemctl restart service.service
```

theo approved procedure.

---

# 177. Common mistake: enable rồi tưởng đang chạy

```bash
sudo systemctl enable app
```

Admin:

> “Đã enable, app running rồi.”

Sai.

Check:

```bash
systemctl is-enabled app
systemctl is-active app
```

Hai câu hỏi khác nhau.

---

# 178. Common mistake: disable rồi tưởng đã dừng

```bash
sudo systemctl disable app
```

Admin:

> “Service stopped.”

Sai.

Check:

```bash
systemctl is-active app
```

nó có thể vẫn:

```text
active
```

---

# 179. Common mistake: `After=` được hiểu như dependency

Unit:

```ini
After=postgresql.service
```

Admin:

> “Postgres chắc chắn sẽ được start.”

Sai.

Need pull-in:

```text
Wants=
Requires=
```

tùy semantics.

---

# 180. Common mistake: `Requires=` được hiểu như ordering

Unit:

```ini
Requires=postgresql.service
```

Admin:

> “App chắc chắn start sau Postgres.”

Không đúng.

Nếu cần ordering:

```ini
After=postgresql.service
```

thường kết hợp.

---

# 181. Common mistake: thêm mọi thứ vào `After=`

Có người viết:

```ini
After=network.target
After=postgresql.service
After=redis.service
After=sshd.service
After=cron.service
```

không dựa trên actual dependency.

Kết quả:

```text
unnecessary coupling
complex boot graph
harder troubleshooting
possible cycles
```

Dependencies phải reflect real requirements.

---

# 182. Common mistake: `Restart=always` cho mọi service

Một batch/oneshot process được thiết kế:

```text
do job
exit 0
```

Nếu:

```ini
Restart=always
```

bạn có thể biến nó thành:

```text
run forever repeatedly
```

Policy phải dựa lifecycle semantics.

---

# 183. Common mistake: service script tự background

```bash
java -jar app.jar &
exit 0
```

Systemd sees wrapper exit.

Behavior có thể không phù hợp với main-process model.

Better long-running app:

```text
remain foreground
systemd supervises it
```

---

# 184. Common mistake: dùng `nohup` dưới systemd

Ví dụ:

```ini
ExecStart=/usr/bin/nohup /usr/bin/java -jar app.jar
```

Thường không cần.

`nohup` là tool phù hợp cho shell/session scenarios.

Systemd đã có:

```text
lifecycle
stdout/stderr handling
process tracking
signals
restart
boot activation
```

Đừng chồng các process-management mechanisms không cần thiết.

---

# 185. Common mistake: chạy application bằng root để fix permission

Symptom:

```text
Permission denied
```

Bad fix:

```ini
User=root
```

Correct diagnosis:

```text
what resource is denied?
what minimum privilege is needed?
```

Rồi chỉnh:

```text
owner/group/mode
directory
ACL
capability
port architecture
```

phù hợp.

---

# 186. Common mistake: `systemctl status` là bằng chứng duy nhất

Service:

```text
active
```

không đồng nghĩa app healthy.

Service:

```text
failed
```

cũng không cho root cause nếu không inspect deeper.

Good evidence pack:

```text
timestamp
unit state
unit definition
MainPID/process
logs
filesystem/config evidence
validation result
```

---

# 187. Practical command matrix

| Command                       | Mục đích                        |       Có thay runtime? |
| ----------------------------- | ------------------------------- | ---------------------: |
| `systemctl status UNIT`       | Human-readable state/evidence   |                  Không |
| `systemctl show UNIT`         | Detailed properties             |                  Không |
| `systemctl is-active UNIT`    | Runtime active check            |                  Không |
| `systemctl is-enabled UNIT`   | Enablement state                |                  Không |
| `systemctl is-failed UNIT`    | Failed-state check              |                  Không |
| `systemctl start UNIT`        | Activate now                    |                     Có |
| `systemctl stop UNIT`         | Deactivate now                  |                     Có |
| `systemctl restart UNIT`      | Stop + start                    |                     Có |
| `systemctl reload UNIT`       | Request app config reload       |                 Có thể |
| `systemctl daemon-reload`     | Reload manager unit definitions |         Manager config |
| `systemctl enable UNIT`       | Configure enablement links      | Boot/activation config |
| `systemctl disable UNIT`      | Remove enablement links         | Boot/activation config |
| `systemctl reset-failed UNIT` | Clear failure/rate-limit state  |          Manager state |
| `systemctl cat UNIT`          | View effective unit fragments   |                  Không |
| `systemd-analyze verify FILE` | Static validation               |                  Không |

---

# 188. Risk matrix

| Operation                 | Risk                                              |
| ------------------------- | ------------------------------------------------- |
| `status`, `show`, `cat`   | Low/read-only                                     |
| `is-active`, `is-enabled` | Low/read-only                                     |
| `daemon-reload`           | Usually low but system-wide manager config reread |
| `enable/disable`          | Changes future activation                         |
| `start`                   | Starts workload/resources                         |
| `stop`                    | Availability impact                               |
| `restart`                 | Availability + process-state change               |
| edit unit                 | Potential startup/security impact                 |
| `mask`                    | Strong prevention of activation                   |
| changing `User=`          | Permission/security/data access impact            |
| changing dependencies     | Boot/start-order impact                           |

Không dùng cùng một mức caution cho tất cả commands.

---

# 189. Verification layers sau một service change

Một professional validation không dừng ở:

```bash
systemctl start app
```

Nên nghĩ:

```text
Layer 1 — unit definition
Loaded correctly?

Layer 2 — lifecycle
Active/running?

Layer 3 — process
Correct PID/user/command?

Layer 4 — execution context
Correct cwd/environment?

Layer 5 — service resource
Correct socket/port/files?

Layer 6 — application
Endpoint/functionality healthy?

Layer 7 — dependency
DB/downstream healthy?

Layer 8 — monitoring
No new alerts/errors?
```

Day 4 tập trung Layer 1–4.

Các Day sau xây tiếp 5–8.

---

# 190. Exam Focus — phải hiểu bằng reasoning

### Câu hỏi: `start` và `enable` khác gì?

Đáp án tốt:

```text
start activates the unit now.
enable installs the unit into systemd's configured activation dependency
structure, commonly so it is pulled in during boot.
They are independent operations.
```

Không chỉ:

```text
start = start
enable = boot
```

mà nên biết enable thường tạo symlink từ `[Install]`.

---

# 191. Exam Focus — `reload` vs `daemon-reload`

Đáp án:

```text
systemctl reload app.service
asks the application/service to reload its own configuration if supported.

systemctl daemon-reload
asks the systemd manager to re-read unit files and dependency definitions.
```

Đây là câu rất dễ thi.

---

# 192. Exam Focus — `After=` vs `Requires=`

Đáp án:

```text
After= defines ordering.
Requires= defines a requirement/pull-in relationship.

Neither directive replaces the other.
```

Ví dụ:

```ini
Requires=database.service
After=database.service
```

có cả dependency và ordering.

---

# 193. Exam Focus — `WantedBy=multi-user.target`

Không trả lời:

> Service starts after multi-user.target.

Trả lời đúng hơn:

```text
WantedBy= is installation information.
When the service is enabled, systemctl typically creates a symlink under
multi-user.target.wants/, establishing a Wants relationship from the target
to this service.
```

Đó là mức trả lời mạnh.

---

# 194. Exam Focus — service environment

Nếu hỏi:

> Tôi `export APP_ENV=prod` trong SSH rồi `sudo systemctl start app`. Service có chắc nhận được không?

Trả lời:

```text
No.

A system service does not generally inherit arbitrary environment variables
from the administrator's interactive shell. Configure service environment
explicitly with mechanisms such as Environment= or EnvironmentFile=.
```

---

# 195. Exam Focus — `daemon-reload`

Nếu sửa:

```text
/etc/systemd/system/app.service
```

workflow:

```text
validate
→ daemon-reload
→ start/restart as appropriate
→ verify
```

Đừng chỉ:

```text
edit → restart
```

rồi assume manager saw definition mới.

---

# 196. Exam Focus — failure

Nếu `systemctl start` fail:

Không làm ngay:

```bash
chmod 777 ...
kill -9 ...
run as root ...
```

Thay vào đó:

```text
state
→ unit definition
→ MainPID/result
→ logs
→ filesystem/user/environment evidence
→ root cause
→ minimal remediation
→ validation
```

Đây là thứ người đánh giá practical muốn thấy hơn khả năng “thử random commands”.

---

# 197. Knowledge Check

Hãy tự trả lời các câu sau trước khi nhìn đáp án:

|   # | Câu hỏi                                                                |
| --: | ---------------------------------------------------------------------- |
|   1 | `systemd` chạy ở PID nào khi là system manager?                        |
|   2 | Unit khác process thế nào?                                             |
|   3 | `.service` mô tả cái gì?                                               |
|   4 | `Loaded:` khác `Active:` như thế nào?                                  |
|   5 | `active` có chứng minh application healthy không?                      |
|   6 | `systemctl start` có enable service không?                             |
|   7 | `systemctl enable` có start service không?                             |
|   8 | `disable` có stop service không?                                       |
|   9 | `[Install] WantedBy=` dùng khi nào?                                    |
|  10 | `After=` có tự kéo dependency vào start không?                         |
|  11 | `Requires=` có tự định nghĩa ordering không?                           |
|  12 | `reload` khác `daemon-reload` ra sao?                                  |
|  13 | Tại sao interactive shell `export` không phải service config đáng tin? |
|  14 | `User=` ảnh hưởng gì?                                                  |
|  15 | `WorkingDirectory=` ảnh hưởng gì?                                      |
|  16 | Tại sao không nên `nohup ... &` trong `ExecStart=`?                    |
|  17 | `Restart=on-failure` dùng để làm gì?                                   |
|  18 | `reset-failed` có sửa root cause không?                                |
|  19 | Tại sao nên dùng `systemd-analyze verify` trước activation?            |
|  20 | Vì sao không nên sửa vendor unit trực tiếp?                            |

---

# 198. Đáp án Knowledge Check

**1.** System manager thông thường chạy là PID 1.

**2.** Unit là management/configuration abstraction; process là runtime execution entity do kernel quản lý.

**3.** `.service` mô tả process/service lifecycle được systemd control/supervise.

**4.** `Loaded` nói unit definition; `Active` nói runtime activation state.

**5.** Không. Nó mới chứng minh lifecycle state ở systemd layer.

**6.** Không mặc định.

**7.** Không mặc định; dùng `--now` mới kết hợp immediate start.

**8.** Không.

**9.** Nó cung cấp installation/enablement information, ví dụ tạo Wants symlink vào target khi enable.

**10.** Không. `After=` chỉ ordering.

**11.** Không. Requirement và ordering là riêng.

**12.** `reload` yêu cầu application reload app config; `daemon-reload` yêu cầu systemd reload unit definitions.

**13.** System services có execution environment riêng, không đơn giản inherit arbitrary interactive shell state.

**14.** Chỉ định Unix account service process chạy dưới.

**15.** Set process current working directory.

**16.** Vì systemd đã là supervisor; self-backgrounding làm lifecycle/process tracking khó và không cần thiết.

**17.** Tự restart service sau qualifying failures.

**18.** Không. Nó chỉ reset recorded failed/rate-limit state.

**19.** Để phát hiện configuration/static execution errors trước khi ảnh hưởng runtime.

**20.** Vì package upgrade có thể overwrite; administrator override/drop-in dễ maintain/rollback hơn.

---

# 199. Independent Practical Challenge

Không nhìn lab solution ở trên, hãy tự xây một service tên:

```text
fresher-worker.service
```

với các invariant sau:

```text
service chạy bằng non-root user
working directory riêng
EnvironmentFile riêng
long-running process
Restart=on-failure
5-second restart delay
enable qua multi-user.target
```

Bạn phải cung cấp evidence chứng minh:

```text
unit loaded
service active
service enabled
MainPID > 0
process owner đúng
working directory đúng
environment đúng
process nằm trong service cgroup
stop làm process biến mất
start tạo process mới
```

Sau đó inject một failure **chỉ trong lab** bằng cách làm `ExecStart` target non-executable hoặc dùng một invalid working directory.

Bạn phải recovery theo flow:

```text
detect
→ preserve evidence
→ determine root cause
→ restore configuration/permissions
→ validate
→ restart
→ prove healthy at service/process layer
```

Không được dùng:

```text
chmod 777
User=root
kill -9
reboot
```

như shortcut.

---

# 200. Mastery Check — mental model cuối Module 2

Hãy giữ mô hình này trong đầu:

```text
                /etc/systemd/system/app.service
                             │
                             │ unit definition
                             ↓
                      systemd manager
                          PID 1
                             │
             ┌───────────────┼────────────────┐
             │               │                │
          [Unit]          [Service]        [Install]
             │               │                │
      dependencies       execution         enablement
       + ordering          context           hooks
             │               │                │
       Wants/After       User/Group       WantedBy
                            │
                     WorkingDirectory
                            │
                       Environment
                            │
                        ExecStart
                            │
                         Restart
                            ↓
                        PROCESS
                            │
             ┌──────────────┼──────────────┐
             │              │              │
            PID            UID            cwd
             │              │              │
           PPID          groups       environment
             │
          signals
             │
           cgroup
```

Và lifecycle:

```text
                 ENABLEMENT
                    │
        enable ─────┼──── disable
                    │
                    ↓
              boot activation
                    │
                    ↓

inactive ── start ──→ activating ──→ active
   ↑                                  │
   │                                  │
   └──────────── stop ────────────────┘

active ───────── restart ─────────→ new process

failure
   ↓
failed
   │
   ├── Restart= policy → auto-recovery attempt
   │
   └── root cause fix → reset/start → validation
```

Và ba config/runtime layers:

```text
UNIT FILE ON DISK
      │
      │ daemon-reload
      ↓
SYSTEMD MANAGER CONFIG
      │
      │ start/restart
      ↓
RUNNING PROCESS
PID / UID / cwd / environment
```

Nếu bạn hiểu thật chắc ba diagram này thì phần lớn command `systemctl` sẽ không còn là thứ phải học thuộc.

**Module 2 hoàn thành về mặt kiến thức khi bạn có thể giải thích và thực hành được toàn bộ chain:**

```text
create unit
→ validate
→ daemon-reload
→ start
→ inspect
→ verify process context
→ enable
→ distinguish active/enabled
→ stop/restart safely
→ inject failure
→ collect evidence
→ diagnose
→ recover
→ validate
```

Đó chính là nền tảng trực tiếp để sang **Module 3: Package Management + Full Day-4 Integrated Operations & Failure-Recovery Lab**, nơi chúng ta sẽ kết hợp package installation, package ownership/files, systemd lifecycle, processes, environment và recovery thành một bài thực hành Day 4 hoàn chỉnh.
