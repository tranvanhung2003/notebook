# Day 2 — Tổng quan nội dung cần học

Day 2 tiếp tục phần **Linux Administration Foundations**, nhưng chuyển trọng tâm từ các thao tác nền tảng của Day 1 sang **vận hành một Linux/RHEL server như một Infrastructure Operations engineer**: quản lý package, process, service, network, security controls, storage, logs, scripting, scheduling, backup/restore và troubleshooting. Theo syllabus, toàn bộ phạm vi Concept/Lecture của Day 2 bao gồm từ `dnf/yum/rpm`, `systemd`, SELinux, `firewalld`, networking, storage, logging cho đến Bash, `cron`, backup và operational troubleshooting. Infrastructure_Operations_14_Da…

Tôi sẽ chia Day 2 thành đúng **2 module lớn** như bạn yêu cầu:

1. **Module 1 — Lý thuyết: Linux/RHEL System Administration & Operations**
2. **Module 2 — Thực hành: Linux/RHEL Administration, Troubleshooting & Automation Lab**

Ở bước này tôi chỉ xây dựng **bản đồ kiến thức đầy đủ** để bạn nhìn thấy toàn bộ những gì cần học. Khi bạn yêu cầu từng module sau, chúng ta sẽ đi sâu theo hướng: **What → Why → How → Verification → Troubleshooting → Evidence → Rollback → Escalation**.

---

# MODULE 1 — LÝ THUYẾT

## Linux/RHEL System Administration & Operations

Module này tương ứng với toàn bộ phần **Concept/Lecture của Day 2**. Infrastructure_Operations_14_Da…

### 1. Linux/RHEL system administration và package management

Bạn cần hiểu cách một RHEL/Linux server được quản trị ở mức operating system, đặc biệt là software package.

Phạm vi gồm:

| Nội dung               | Cần nắm                                                              |
| ---------------------- | -------------------------------------------------------------------- |
| `dnf`                  | Package manager chính trên các RHEL generation hiện đại              |
| `yum`                  | Compatibility/legacy awareness và cách sử dụng trong môi trường RHEL |
| `rpm`                  | Package format và low-level package query/install awareness          |
| package lifecycle      | install, update, remove, query, verify                               |
| dependencies           | quan hệ phụ thuộc giữa các package                                   |
| repository             | nguồn cung cấp package                                               |
| repository awareness   | repository nào đang enabled/disabled                                 |
| subscription awareness | nhận thức về RHEL subscription và repository entitlement             |

Điểm quan trọng về Operations không chỉ là “cài được package”, mà phải biết:

- package đến từ repository nào;
- version nào được cài;
- dependency nào bị ảnh hưởng;
- việc update có thể ảnh hưởng service nào;
- cách verify sau thay đổi;
- cần lưu evidence gì cho change/audit.

---

### 2. Environment variables

Bạn cần hiểu **environment variables** là dữ liệu key/value ảnh hưởng đến cách shell, process và application hoạt động.

Các vấn đề cần học bao gồm:

- shell environment;
- user environment;
- process environment;
- cách xem environment variables;
- cách set variable;
- sự khác nhau giữa variable tạm thời và cấu hình persistent;
- biến thường gặp như `PATH`;
- tác động của environment variable đến command và application.

Trong Operations, đây là một nguồn lỗi rất phổ biến:

> Command chạy được với user này nhưng không chạy với user khác.

hoặc:

> Script chạy thủ công thành công nhưng chạy qua `cron` lại thất bại.

Một trong những nguyên nhân có thể là khác biệt về environment.

---

### 3. Processes, process states, signals và resource usage

Đây là phần giúp bạn hiểu Linux đang thực sự chạy workload như thế nào.

Bạn cần học:

- process là gì;
- PID;
- parent/child process;
- foreground/background process;
- process states;
- process ownership;
- process resource usage;
- CPU usage;
- memory usage;
- signals;
- graceful termination versus forced termination;
- cách xác định process đang gây vấn đề.

Mục tiêu Operations là có khả năng đi từ symptom:

> “Server chậm.”

đến các câu hỏi cụ thể:

- process nào đang dùng CPU?
- process nào đang dùng memory?
- process có stuck không?
- service liên quan đến process nào?
- có nên terminate process không?
- terminate bằng signal nào an toàn hơn?

---

### 4. `systemd`, units và service lifecycle

Đây là một trong những phần quan trọng nhất của Day 2.

Bạn cần hiểu:

- `systemd`;
- unit;
- service unit;
- unit state;
- active/inactive;
- enabled/disabled;
- failed state;
- dependency;
- service lifecycle.

Các operation cơ bản gồm:

- start;
- stop;
- restart;
- reload;
- enable;
- disable;
- inspect status;
- xác định failed service;
- điều tra service startup failure.

Không chỉ học command, bạn phải hiểu quan hệ:

**systemd unit → process → configuration → dependency → port → log**

Ví dụ khi một web service không hoạt động, bạn không nên chỉ liên tục restart service mà phải kiểm tra:

**service status → process → logs → configuration → network port → firewall/SELinux**

---

### 5. Boot targets, service startup và recovery

Day 2 yêu cầu hiểu cách Linux/RHEL đưa system từ boot đến trạng thái vận hành.

Các nội dung chính:

- boot target;
- `systemd` target;
- service startup;
- automatic startup;
- service dependency;
- failed startup;
- recovery concept.

Bạn cần hiểu tại sao một service có thể:

- chạy được khi start thủ công;
- nhưng không chạy sau reboot;
- hoặc bị dependency failure.

Mục tiêu là phân biệt:

**service đang chạy hiện tại**  
với  
**service được cấu hình để chạy sau boot**.

---

### 6. SELinux

SELinux là security control rất quan trọng trên RHEL.

Theo Day 2 cần nắm:

- SELinux modes;
- SELinux contexts;
- quan hệ SELinux với application access;
- nhận diện SELinux-related failure.

Bạn cần hiểu ba vấn đề riêng biệt:

**Linux permissions**  
≠ **SELinux policy/context**  
≠ **firewall access**

Một application có thể có UNIX permissions chính xác nhưng vẫn bị SELinux chặn.

Mục tiêu Operations là không hình thành thói quen nguy hiểm:

> “Không chạy thì disable SELinux.”

Thay vào đó phải xác định:

- SELinux có thực sự là nguyên nhân không;
- context có đúng không;
- log nào chứng minh việc deny;
- corrective action nào phù hợp.

---

### 7. `firewalld`

Bạn cần hiểu host-based firewall trên RHEL.

Phạm vi bao gồm:

- `firewalld`;
- firewall rules;
- permission cho network traffic;
- kiểm tra traffic được phép hay bị block.

Bạn phải xây dựng tư duy:

**Application listening** chưa đồng nghĩa với **client truy cập được**.

Đường đi có thể là:

**client → network → server NIC → firewall → listening service**

Vì vậy khi service không truy cập được, không được mặc định kết luận application lỗi.

---

### 8. Linux networking fundamentals

Đây là một phần lớn của Day 2.

Theo syllabus cần học:

- IP addressing;
- routing;
- DNS resolution;
- TCP ports;
- UDP ports.

Bạn cần hiểu các dependency:

**IP configuration → route → gateway → DNS → destination → port → service**

Khi troubleshooting connectivity phải biết phân biệt:

- IP problem;
- routing problem;
- DNS problem;
- port problem;
- service problem;
- firewall problem.

---

### 9. Network diagnostic tools

Day 2 yêu cầu sử dụng và hiểu:

`ping`

Kiểm tra basic IP reachability trong phạm vi phù hợp.

`traceroute` / `tracepath`

Quan sát network path/hops.

`curl`

Kiểm tra application connectivity, đặc biệt HTTP/HTTPS hoặc TCP-oriented checks phù hợp.

`ss`

Kiểm tra socket và listening port.

`ip`

Kiểm tra và quản lý Linux network state như:

- interface;
- address;
- route.

Điểm quan trọng là không học các command rời rạc mà hiểu mỗi tool trả lời **một câu hỏi troubleshooting khác nhau**.

Ví dụ:

`ip` → cấu hình local network có đúng không?

`ping` → có basic reachability không?

`tracepath` → path gặp vấn đề ở đâu?

`ss` → service có thực sự listening không?

`curl` → application có phản hồi ở layer phù hợp không?

---

### 10. Disks và partitions

Storage administration là phần quan trọng khác của Day 2.

Bạn cần hiểu:

- disk;
- partition;
- filesystem;
- mount;
- mount point.

Phải phân biệt rõ:

**physical/virtual disk**

→ **partition**

→ **filesystem**

→ **mount point**

→ **user/application data**

Nếu không hiểu chuỗi này, việc troubleshooting storage sẽ rất dễ nhầm.

---

### 11. Filesystems và mounts

Bạn cần hiểu:

- filesystem;
- mount;
- temporary mount;
- persistent mount;
- filesystem availability;
- mount failure.

Đặc biệt phải hiểu một filesystem tồn tại không có nghĩa là application đang sử dụng filesystem đó.

Ví dụ mount thất bại có thể khiến application vô tình ghi dữ liệu vào filesystem khác bên dưới mount point.

---

### 12. `/etc/fstab`

`/etc/fstab` là phần bắt buộc trong Day 2.

Bạn cần học vai trò của nó trong:

- persistent filesystem mounts;
- boot-time mounting;
- mount configuration;
- recovery khi cấu hình mount lỗi.

Đây là khu vực có operational risk vì cấu hình sai có thể ảnh hưởng boot hoặc filesystem availability.

Do đó phải biết:

- backup trước khi thay đổi;
- validate;
- test;
- verify;
- rollback.

---

### 13. Disk capacity và inode capacity

Một Infrastructure Operations engineer phải phân biệt:

**disk space exhaustion**

và

**inode exhaustion**.

Hai tình trạng này đều có thể khiến:

> “Không tạo được file mới.”

nhưng root cause khác nhau.

Bạn cần hiểu:

- filesystem capacity;
- used/free space;
- inode usage;
- capacity threshold;
- storage-related symptoms.

---

### 14. `journalctl`

Bạn cần học `journalctl` như công cụ quan trọng để truy vấn `systemd journal`.

Cần hiểu cách sử dụng log để điều tra:

- service startup;
- service failure;
- boot;
- system event;
- error timeline.

Đây là bước chuyển tư duy rất quan trọng từ:

> “Thử restart xem có được không.”

sang:

> “Thu thập evidence trước, xác định nguyên nhân rồi mới thực hiện corrective action.”

---

### 15. syslog

Bên cạnh `journald`, bạn cần hiểu khái niệm **syslog** trong Linux logging.

Phải nhận biết:

- log source;
- log messages;
- severity awareness;
- nơi application/system có thể ghi log;
- vai trò log trong troubleshooting và audit.

Ở Day 10 bạn sẽ học centralized logging sâu hơn, nên Day 2 chính là foundation.

---

### 16. Log rotation

Log không thể tăng vô hạn.

Bạn cần hiểu:

- vì sao phải rotate log;
- log retention cơ bản;
- hậu quả nếu log không được quản lý;
- quan hệ giữa log growth và disk capacity.

Một lỗi Operations kinh điển:

> Application tạo log quá nhanh → filesystem đầy → service khác bắt đầu lỗi.

---

### 17. Text processing: `grep`, `awk`, `sed`

Đây là bộ công cụ cực kỳ quan trọng để xử lý logs và text.

Bạn cần hiểu vai trò tương đối:

- `grep`: tìm/filter text;
- `awk`: xử lý dữ liệu theo field/pattern;
- `sed`: stream editing và transformation.

Mục tiêu không phải trở thành expert ngay trong Day 2 mà phải đủ khả năng:

**large log → filter → extract → correlate → evidence**

---

### 18. Regular expressions

Regular expressions hỗ trợ:

- tìm pattern;
- lọc log;
- extract dữ liệu;
- input validation;
- scripting.

Bạn cần hiểu các pattern cơ bản để sử dụng cùng:

- `grep`;
- `awk`;
- `sed`;
- Bash.

Phần này sẽ trở lại ở Day 12–13 khi học automation và data reconciliation.

---

### 19. Bash variables

Day 2 bắt đầu đưa bạn từ command line administration sang automation.

Bạn cần hiểu:

- variable assignment;
- variable expansion;
- quoting awareness;
- command output sử dụng trong script;
- input/output.

Bash variable là foundation cho health-check và backup script trong lab.

---

### 20. Bash control flow

Bạn cần nắm các cấu trúc logic để script có thể ra quyết định thay vì chỉ chạy command nối tiếp.

Ví dụ conceptual:

**check condition**

→ đúng → thực hiện action A  
→ sai → thực hiện action B.

Đây là cơ sở cho:

- health checks;
- validation;
- error handling;
- automation.

---

### 21. Exit codes

Exit code là nội dung rất quan trọng trong operational scripting.

Bạn cần hiểu:

- command success/failure;
- exit status;
- script exit status;
- downstream decision.

Một script Operations tốt không chỉ in ra:

> Backup completed.

mà còn phải có cách để system khác xác định:

> Backup thực sự thành công hay thất bại?

Exit code chính là một phần của cơ chế đó.

---

### 22. Error handling

Bạn cần hiểu script không nên giả định mọi command đều thành công.

Workflow tốt phải có dạng:

**execute → verify result → handle error → record evidence → exit appropriately**

Day 2 yêu cầu health-check và backup script có **exit-code handling**, nên phần này trực tiếp phục vụ lab.

---

### 23. `cron` scheduling

Automation chỉ thực sự hữu ích khi có thể chạy theo lịch.

Bạn cần hiểu:

- scheduled execution;
- cron job;
- execution timing;
- environment differences;
- output/error handling.

Một vấn đề thực tế cần nhớ:

> Script chạy thủ công thành công không chứng minh nó sẽ chạy thành công qua `cron`.

---

### 24. File compression

Bạn cần học compression trong context Operations:

- giảm kích thước dữ liệu;
- chuẩn bị archive;
- backup;
- log collection;
- evidence packaging.

Phải phân biệt conceptual:

**archive**

và

**compression**.

Hai khái niệm có liên quan nhưng không hoàn toàn giống nhau.

---

### 25. `tar` backup

Bạn sẽ sử dụng `tar` để:

- đóng gói files/directories;
- tạo backup archive;
- phục vụ restore.

Quan trọng nhất là:

> tạo được archive chưa đủ — phải kiểm tra nội dung và thực hiện restore validation.

---

### 26. `rsync` backup

Bạn cần hiểu `rsync` trong workflow copy/synchronize dữ liệu.

Các yếu tố cần quan tâm:

- source;
- destination;
- file differences;
- repeated execution;
- verification.

Ở góc độ Operations, phải đặc biệt cẩn thận xác định đúng:

**source → destination**

để tránh thao tác ngược hoặc làm mất dữ liệu.

---

### 27. Basic restore procedures

Backup không được xem là hoàn chỉnh nếu chưa chứng minh restore được.

Bạn cần hiểu workflow:

**backup → verify backup → restore → verify restored data**

Day 2 đặt nền móng cho tư duy này; Day 5 và Day 7 sẽ mở rộng sang AWS backup/DR và HYCU.

---

### 28. Health checks

Bạn cần học cách xây dựng một Linux health check có cấu trúc.

Các nhóm kiểm tra thường liên quan trực tiếp đến các chủ đề Day 2 như:

- service state;
- process state;
- resource usage;
- network;
- storage;
- filesystem;
- log/error indication.

Mục tiêu là chuyển từ reactive troubleshooting sang systematic verification.

---

### 29. Operational troubleshooting

Đây là chủ đề tổng hợp của Module Lý thuyết.

Bạn phải bắt đầu hình thành một troubleshooting flow có kỷ luật:

**Observe symptom**

→ **Collect evidence**

→ **Identify affected layer**

→ **Perform read-only verification**

→ **Form hypothesis**

→ **Apply authorized corrective action**

→ **Verify recovery**

→ **Collect post-change evidence**

→ **Document**

→ **Escalate nếu vượt boundary**

Day 2 không chỉ dạy Linux commands; mục tiêu lớn hơn là tạo **operational troubleshooting mindset**.

---

# MODULE 2 — THỰC HÀNH

## Linux/RHEL Administration, Troubleshooting & Automation Lab

Module này tương ứng với toàn bộ **Assignment/Lab của Day 2** trong syllabus. Infrastructure_Operations_14_Da…

Khi học chi tiết Module Thực hành, chúng ta sẽ xây dựng một RHEL/Linux lab xuyên suốt thay vì làm từng command rời rạc.

---

### Lab Track 1 — Package installation và verification

Bạn sẽ:

- kiểm tra package hiện tại;
- kiểm tra repository phù hợp;
- install approved packages;
- xác nhận package được cài;
- xác định package version;
- thu thập installation evidence.

Mục tiêu là chứng minh:

**requested package → approved repository → installed successfully → verified**

---

### Lab Track 2 — `systemd` service management

Bạn sẽ thực hành:

- inspect service;
- start/stop/restart;
- enable/disable khi phù hợp;
- kiểm tra service lifecycle;
- kiểm tra startup behavior;
- verify service state.

Không chỉ nhìn thấy `active (running)` mà còn phải kiểm tra behavior thực tế của service.

---

### Lab Track 3 — Process inspection

Bạn sẽ:

- xem running processes;
- xác định PID;
- xác định process owner;
- quan sát resource usage;
- liên hệ process với service;
- thực hiện safe process troubleshooting.

Mục tiêu là phân biệt:

**service state**

với

**underlying process state**.

---

### Lab Track 4 — Environment variables

Bạn sẽ:

- xem environment;
- kiểm tra một số variables;
- tạo/test controlled variable;
- quan sát ảnh hưởng của environment tới command/script.

Phần này sau đó sẽ được liên kết với `cron`.

---

### Lab Track 5 — Deliberately failed service recovery

Syllabus yêu cầu cố tình tạo một failed service trong lab rồi phục hồi.

Workflow sẽ là:

**inject controlled failure**

→ observe symptom

→ inspect `systemd`

→ inspect logs

→ determine root cause

→ corrective action

→ service recovery

→ verification

→ evidence collection.

Đây là một trong những bài lab quan trọng nhất của Day 2.

---

### Lab Track 6 — Network configuration và connectivity testing

Bạn sẽ kiểm tra hoặc cấu hình trong authorized lab:

- IP;
- interface;
- route;
- DNS;
- network connectivity.

Sau đó sử dụng:

- `ip`;
- `ping`;
- `tracepath`/`traceroute`;
- `ss`;
- `curl`.

Mục tiêu là xác định được failure nằm ở:

**local host → route → DNS → network → port → service**

---

### Lab Track 7 — DNS verification

Bạn phải chứng minh được khác biệt giữa:

**IP connectivity**

và

**DNS resolution**.

Một host có thể reachable bằng IP nhưng application vẫn lỗi vì name resolution.

Bạn sẽ học cách tách hai lớp vấn đề này khi troubleshooting.

---

### Lab Track 8 — Firewall permissions

Bạn sẽ cấu hình/test firewall permission trong môi trường lab.

Bạn phải chứng minh được cả:

**allowed path**

và

**denied path**.

Evidence phải giúp trả lời:

- traffic nào được allow;
- traffic nào bị deny;
- port/service nào liên quan;
- rule nào quyết định behavior.

---

### Lab Track 9 — SELinux-related access

Bạn sẽ tạo hoặc quan sát một scenario mà application access liên quan tới SELinux.

Mục tiêu không phải disable SELinux.

Mục tiêu là:

**reproduce → inspect → identify SELinux involvement → correct safely → verify**

Đây là bài tập quan trọng để tránh một anti-pattern rất phổ biến trong Linux administration.

---

### Lab Track 10 — Disk, filesystem và mount

Bạn sẽ thao tác với storage trong controlled lab:

**disk/partition**

→ filesystem

→ mount point

→ mount validation.

Bạn phải biết kiểm tra:

- storage device;
- mount status;
- filesystem availability;
- data accessibility.

---

### Lab Track 11 — Persistent mount với `/etc/fstab`

Bạn sẽ cấu hình persistent mounting.

Lab phải bao gồm:

- backup configuration;
- modification;
- validation;
- mount test;
- persistence verification;
- rollback plan.

Đây là một configuration-changing lab nên chúng ta sẽ làm theo operational safety procedure.

---

### Lab Track 12 — Disk và inode capacity

Bạn sẽ kiểm tra:

- disk capacity;
- filesystem utilization;
- inode utilization.

Sau đó phân tích storage symptom.

Mục tiêu là khi gặp:

> `No space left on device`

bạn không chỉ nhìn disk GB mà còn nghĩ đến inode.

---

### Lab Track 13 — Log extraction

Bạn sẽ sử dụng:

- `journalctl`;
- syslog-related logs;
- log files.

để extract event liên quan tới:

- service;
- system;
- failure;
- troubleshooting scenario.

Không chỉ đọc log mà phải xác định đúng:

**timestamp + host + service/process + event + severity/context**

để làm evidence.

---

### Lab Track 14 — Log filtering với `grep`, `awk`, `sed` và regex

Bạn sẽ lấy một lượng log tương đối lớn và từng bước:

**raw logs**

→ `grep`

→ regex

→ `awk`

→ `sed`

→ useful troubleshooting evidence.

Đây cũng là foundation cho automation về sau.

---

### Lab Track 15 — Bash health-check script

Bạn sẽ phát triển một Bash script kiểm tra system health.

Script phải có khả năng kiểm tra một số trạng thái hệ thống, xử lý kết quả và trả về status phù hợp.

Trọng tâm bao gồm:

- variables;
- conditions;
- command execution;
- exit codes;
- error handling;
- readable output.

Đây là sản phẩm automation chính đầu tiên trong syllabus.

---

### Lab Track 16 — Bash backup script

Bạn sẽ viết một controlled backup script sử dụng các công cụ phù hợp như:

- compression;
- `tar`;
- `rsync`.

Script cần kiểm tra:

**source**

→ backup operation

→ backup result

→ verification

→ exit status.

Không được coi command chạy xong là backup đã thành công.

---

### Lab Track 17 — `cron` scheduling

Bạn sẽ schedule script chạy bằng `cron`.

Bạn phải verify:

- cron entry;
- thời điểm chạy;
- script thực sự được execute;
- output/result;
- error nếu có.

Điểm quan trọng là kiểm tra differences giữa manual execution và scheduled execution.

---

### Lab Track 18 — Backup validation

Sau khi tạo backup, bạn phải xác nhận:

- archive/output tồn tại;
- nội dung đúng;
- dữ liệu backup phù hợp;
- backup có thể sử dụng.

Đây là bước **verification**, không phải optional.

---

### Lab Track 19 — Restore validation

Bạn sẽ thực hiện controlled restore.

Workflow:

**create source data**

→ backup

→ thay đổi/xóa controlled test copy

→ restore

→ compare restored data

→ confirm integrity.

Syllabus yêu cầu rõ **validate backup and restore results**, do đó Day 2 không kết thúc ở bước “backup succeeded”.

---

### Lab Track 20 — Failure Injection: service failure

Một lỗi được cố tình đưa vào lab.

Bạn phải:

- detect;
- diagnose;
- collect logs;
- determine root cause;
- recover;
- verify;
- document.

---

### Lab Track 21 — Failure Injection: network failure

Bạn sẽ được đặt vào một connectivity scenario bị lỗi.

Mục tiêu là troubleshoot theo layer thay vì thử command ngẫu nhiên.

Flow dự kiến:

**interface/IP**

→ route

→ DNS

→ firewall

→ port

→ application.

---

### Lab Track 22 — Failure Injection: storage failure

Một controlled storage problem sẽ được tạo.

Ví dụ ở mức category theo syllabus:

- mount;
- capacity;
- storage availability.

Bạn phải xác định symptom, root cause và cách recovery trong authorized scope.

---

### Lab Track 23 — Diagnostic evidence collection

Mỗi sự cố phải có evidence.

Bạn sẽ học cách thu thập tối thiểu các loại artifact phù hợp như:

- command output;
- service status;
- relevant logs;
- network state;
- storage state;
- script output;
- timestamp;
- before/after verification.

Mục tiêu là người khác có thể đọc evidence và hiểu bạn đã làm gì.

---

### Lab Track 24 — Linux troubleshooting record

Sản phẩm cuối cùng của lab không chỉ là một server “đã sửa xong”.

Bạn phải xây dựng một troubleshooting record thể hiện:

**symptom**

→ initial state

→ diagnostics

→ evidence

→ root cause

→ action

→ result

→ verification.

---

### Lab Track 25 — Escalation record

Syllabus yêu cầu **troubleshooting and escalation record**.

Bạn phải biết khi nào vấn đề:

- nằm trong scope của operator;
- cần L2/L3;
- cần application team;
- cần network/security/storage owner;
- hoặc cần change approval trước khi tiếp tục.

Một escalation tốt không chỉ viết:

> “Service lỗi, nhờ kiểm tra.”

Mà phải cung cấp evidence đủ để team nhận escalation tiếp tục điều tra mà không phải bắt đầu lại từ đầu.

---

# Mối liên hệ giữa hai module

Hai module không tách biệt hoàn toàn.

Luồng học đúng của Day 2 sẽ là:

**Theory**

Package → Process → `systemd` → SELinux/Firewall → Network → Storage → Logs → Bash → Scheduling → Backup → Troubleshooting

sau đó chuyển thành:

**Practice**

Configure → Verify → Inject Failure → Diagnose → Recover → Re-verify → Collect Evidence → Rollback/Cleanup → Escalation Record.

Đó chính là mô hình vận hành xuyên suốt syllabus: chương trình không chỉ yêu cầu kiến thức Linux mà nhấn mạnh **hands-on labs, verification evidence, controlled troubleshooting và integrated operations practice**. Infrastructure_Operations_14_Da…

---

# Day 2 — Kết quả cuối cùng bạn cần đạt

Sau khi hoàn thành cả hai module, về mặt thực tế bạn phải đủ khả năng nhìn một RHEL/Linux server và thực hiện một workflow kiểu:

> “Service không truy cập được.”

Sau đó không đoán mò mà lần lượt xác định:

**Service có chạy không?**  
→ Process có tồn tại không?  
→ Service log nói gì?  
→ Port có listening không?  
→ IP/interface có đúng không?  
→ Route có đúng không?  
→ DNS có resolve không?  
→ `firewalld` có cho traffic qua không?  
→ SELinux có deny application không?  
→ Storage có đầy hoặc mount lỗi không?  
→ Sau corrective action đã verify thực sự chưa?  
→ Evidence đã đủ chưa?  
→ Nếu chưa giải quyết được thì escalation cần gửi gì?

Nếu làm được toàn bộ flow đó một cách có kiểm soát, bạn đã đạt đúng tinh thần **operational readiness** mà syllabus đặt ra cho Day 2.

Khi bạn muốn bắt đầu học sâu, chỉ cần nói **“Module Lý thuyết Day 2”** hoặc **“Module Thực hành Day 2”**; tôi sẽ triển khai toàn bộ module đó thật chi tiết, đầy đủ, theo đúng syllabus và theo hướng enterprise Operations.
