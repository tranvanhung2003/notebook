# Day 13 — Java Application Configuration: Externalized Configuration, Profiles, Ports, JVM Memory, Logs & Safe Restart

## 1. Syllabus Alignment — hôm nay bắt buộc học gì?

Theo **MASTER_SYLLABUS**, Day 13 thuộc phần **Java Application Configuration**. Nội dung bắt buộc chính xác là:

> **Concept/Lecture:** Externalized configuration, environment variables, profiles, ports, context path, JVM memory options, logs, and safe restart.  
> **Assignment/Lab:** Create environment-specific configuration, tune approved JVM settings, restart safely, and validate logs/endpoints. 00_MASTER_SYLLABUS_Java_Fresher…

Vì vậy, hôm nay chúng ta phải đi hết chuỗi:

**artifact Java không đổi → configuration thay đổi theo environment → Tomcat/JVM nhận configuration → restart có kiểm soát → xác minh process/port/context/log/endpoint/JVM settings → rollback nếu lỗi.**

Day 12 đã xây nền về Java runtime, JAR/WAR, Tomcat và systemd. Day 13 **không phải học lại cách cài Tomcat**, mà là học cách vận hành một Java application đã được deploy sao cho cùng một artifact có thể chạy đúng ở DEV/UAT/PROD mà không sửa code hoặc rebuild tùy tiện.

Day 14 mới đi sâu vào database connectivity, secrets, TLS, release validation và rollback. Vì vậy hôm nay ta sẽ **không biến password/database secret thành trọng tâm lab**.

---

# 2. Environment và version assumptions

Các ví dụ dưới đây giả định:

| Thành phần        | Môi trường tham chiếu    |
| ----------------- | ------------------------ |
| OS                | Linux có `systemd`       |
| Java              | Java 17 hoặc Java 25     |
| Servlet container | Apache Tomcat            |
| Deployment        | WAR trên external Tomcat |
| Service           | ví dụ `tomcat.service`   |
| Tomcat base       | ví dụ `/opt/tomcat`      |
| Application       | ví dụ `javafresher.war`  |
| Default connector | ví dụ TCP `8080`         |
| Context path      | ví dụ `/javafresher`     |

**Không được mặc định máy của bạn đúng các path/name trên.** Trước khi thay đổi, phải discovery môi trường thật bằng:

```bash
java -version
systemctl list-unit-files | grep -i tomcat
systemctl cat tomcat
ps -ef | grep '[o]rg.apache.catalina.startup.Bootstrap'
ss -ltnp
```

Tính đến tháng 10/2026, Tomcat 11.0.x hiện hỗ trợ Java 17 trở lên; Tomcat 10.1.x hỗ trợ Java 11 trở lên. Vì syllabus yêu cầu Java 17 và Java 25, cả Java 17/25 đều phù hợp với Tomcat 11 về yêu cầu Java tối thiểu, nhưng **không có nghĩa bạn được tự ý nâng major version Tomcat trong Day 13**. [Apache Tomcat](https://tomcat.apache.org/whichversion.html?utm_source=chatgpt.com)

---

# 3. Mental Model quan trọng nhất của Day 13

Hãy tưởng tượng application của chúng ta gồm ba lớp:

```text
                 ┌────────────────────────┐
                 │     CLIENT / CURL      │
                 └────────────┬───────────┘
                              │
                        HTTP :8080
                              │
                 ┌────────────▼───────────┐
                 │       TOMCAT           │
                 │ Connector / Context    │
                 │ JVM process            │
                 └────────────┬───────────┘
                              │
                 ┌────────────▼───────────┐
                 │   JAVA APPLICATION     │
                 │ WAR + config/profile   │
                 │ business logic         │
                 └────────────────────────┘

Configuration đến từ bên ngoài:
    systemd Environment / EnvironmentFile
                 │
                 ▼
           startup process
                 │
                 ▼
         JAVA / CATALINA_OPTS
                 │
                 ├── JVM heap
                 ├── Java system properties
                 └── application properties
```

Điều quan trọng là phân biệt:

| Loại cấu hình       | Ai sở hữu?                 | Ví dụ                                   |
| ------------------- | -------------------------- | --------------------------------------- |
| OS/service          | Linux/systemd              | service user, environment               |
| JVM                 | Java runtime               | `-Xms`, `-Xmx`, `-D...`                 |
| Tomcat              | Servlet container          | Connector port, Context                 |
| Application         | Java application/framework | profile, log level, feature config      |
| Runtime observation | operations                 | logs, health endpoint, listening socket |

Nếu không phân biệt các layer này, bạn rất dễ sửa đúng syntax nhưng **sai layer**, dẫn đến configuration không có tác dụng.

---

# 4. Externalized Configuration

## 4.1 Configuration là gì?

Configuration là những giá trị điều khiển hành vi runtime mà thông thường không nên hard-code vào application binary.

Ví dụ:

```text
Environment name
API base URL
Timeout
Thread count
Log level
Feature flag
Port
Context path
Database endpoint
```

Một anti-pattern:

```java
String apiUrl = "https://prod-api.company.com";
```

DEV muốn gọi endpoint khác thì developer phải sửa source code → rebuild → tạo artifact mới.

Cách tốt hơn:

```text
artifact:
    javafresher.war

DEV:
    API_URL=https://dev-api.example

UAT:
    API_URL=https://uat-api.example

PROD:
    API_URL=https://api.example
```

Artifact có thể giữ nguyên; environment cung cấp configuration.

Đây chính là **externalized configuration**.

---

## 4.2 Tại sao Operations cần externalized configuration?

Mục tiêu là tách:

```text
Application artifact
        ≠
Environment configuration
```

Một release có thể có checksum:

```text
javafresher-1.5.2.war
SHA256: abcdef...
```

DEV, UAT và PROD cùng dùng artifact đó nhưng khác configuration.

Điều này giúp:

```text
build once
      ↓
test artifact
      ↓
promote same artifact
      ↓
apply environment configuration
```

Thay vì:

```text
dev.war
uat.war
prod.war
```

với code khác nhau không rõ lý do.

Đây là một nguyên tắc vận hành cực kỳ quan trọng.

---

# 5. Các nguồn configuration

Một Java application có thể nhận configuration qua nhiều nguồn:

| Nguồn                          | Ví dụ                               |
| ------------------------------ | ----------------------------------- |
| config packaged trong artifact | `application.properties`            |
| external config file           | `/etc/myapp/application.properties` |
| environment variable           | `APP_ENV=prod`                      |
| Java system property           | `-Dapp.env=prod`                    |
| JVM options                    | `-Xmx512m`                          |
| command-line arguments         | `--server.port=8081`                |
| container configuration        | Tomcat `server.xml`, Context XML    |

Không được mặc định precedence giữa các nguồn là giống nhau cho mọi framework.

**Precedence là behavior của application/framework.**

---

# 6. Environment Variables

## 6.1 Environment variable là gì?

Environment variable là một cặp:

```text
KEY=VALUE
```

Ví dụ:

```bash
APP_ENV=lab
LOG_LEVEL=INFO
JAVA_HOME=/usr/lib/jvm/java-25
```

Trong shell:

```bash
export APP_ENV=lab

echo "$APP_ENV"
```

Nhưng có một bẫy rất quan trọng:

```bash
export APP_ENV=lab
```

trong SSH shell **không có nghĩa `tomcat.service` tự động nhận nó**.

Application chạy dưới systemd có runtime environment riêng.

Đây là lỗi fresher rất thường gặp:

```text
echo $APP_ENV
→ lab

Application:
→ vẫn chạy prod/default
```

Vì bạn đang kiểm tra **environment của shell**, không phải environment của Java process.

---

# 7. Environment configuration qua systemd

Một cách quản lý sạch là cho systemd đọc một `EnvironmentFile`.

Ví dụ reference design:

```text
/etc/javafresher/
├── javafresher.env
└── application-lab.properties
```

Environment file:

```bash
sudo install -d -o root -g tomcat -m 0750 /etc/javafresher
sudo editor /etc/javafresher/javafresher.env
```

Ví dụ:

```text
APP_ENV=lab
CATALINA_OPTS="-Xms256m -Xmx512m"
```

Sau đó service unit hoặc drop-in có thể tham chiếu:

```ini
[Service]
EnvironmentFile=/etc/javafresher/javafresher.env
```

Không chỉnh vendor service file tùy tiện nếu có thể sử dụng drop-in:

```bash
sudo systemctl edit tomcat
```

---

## 7.1 Khi nào cần `daemon-reload`?

Đây là điểm exam rất hay hỏi.

Nếu bạn thay:

```text
/etc/systemd/system/tomcat.service
```

hoặc:

```text
/etc/systemd/system/tomcat.service.d/override.conf
```

thì:

```bash
sudo systemctl daemon-reload
```

rồi mới restart.

Nhưng nếu unit đã có:

```ini
EnvironmentFile=/etc/javafresher/javafresher.env
```

và bạn **chỉ sửa nội dung `javafresher.env`**, thông thường không cần `daemon-reload`; restart process để environment mới được sử dụng là đủ.

Mental model:

```text
unit definition changed?
       │
       ├── YES → daemon-reload → restart
       │
       └── NO
            │
     EnvironmentFile content only?
            │
            └── restart
```

---

# 8. Làm thế nào chứng minh Java process thực sự nhận environment?

Đầu tiên lấy PID:

```bash
systemctl show tomcat -p MainPID
```

hoặc:

```bash
pgrep -af 'org.apache.catalina.startup.Bootstrap'
```

Ví dụ:

```text
MainPID=2468
```

Kiểm tra process:

```bash
ps -p 2468 -o pid,ppid,user,lstart,cmd
```

Trong lab không chứa secret, có thể kiểm tra một biến cụ thể:

```bash
sudo tr '\0' '\n' < /proc/2468/environ | grep '^APP_ENV='
```

Expected invariant:

```text
APP_ENV=lab
```

### Cảnh báo bảo mật

Không làm:

```bash
sudo tr '\0' '\n' < /proc/2468/environ
```

rồi paste toàn bộ output vào ticket/chat, vì environment có thể chứa credential hoặc token.

Chỉ lọc key không nhạy cảm:

```bash
grep '^APP_ENV='
```

Day 14 sẽ đi sâu hơn về secrets.

---

# 9. Profiles

## 9.1 Profile giải quyết vấn đề gì?

Profile là một tập configuration ứng với một environment hoặc một runtime mode.

Ví dụ:

```text
default
dev
test
uat
prod
```

Concept:

```text
base configuration
       +
profile-specific overrides
       =
effective runtime configuration
```

Ví dụ:

```text
application.properties
        +
application-prod.properties
        =
effective prod configuration
```

Profile **không đồng nghĩa hoàn toàn với environment**, nhưng đây là cách dùng phổ biến.

---

# 10. Supplementary / Beyond the explicit syllabus — Spring Boot profiles

Syllabus không bắt buộc Spring Boot, vì vậy phần implementation này là **Supplementary**, nhưng rất hữu ích nếu provided Java application của bạn dùng Spring Boot.

Spring Boot hỗ trợ externalized configuration qua property files, YAML, environment variables, Java system properties và command-line arguments. Nó cũng hỗ trợ profile-specific files như:

```text
application.properties
application-dev.properties
application-prod.properties
```

Profile-specific configuration có thể override base configuration. [Home](https://docs.spring.io/spring-boot/reference/features/external-config.html?utm_source=chatgpt.com)

Ví dụ:

```properties
# application.properties
app.banner=Java Fresher
logging.level.root=INFO
```

```properties
# application-lab.properties
app.environment=LAB
logging.level.root=DEBUG
```

Activate:

```bash
SPRING_PROFILES_ACTIVE=lab
```

Hoặc Java property:

```bash
-Dspring.profiles.active=lab
```

---

# 11. External Spring Boot configuration directory

Một production-like approach tốt hơn việc sửa file trong WAR là external directory:

```text
/etc/javafresher/
├── application.properties
└── application-lab.properties
```

Ví dụ systemd environment:

```text
SPRING_PROFILES_ACTIVE=lab
SPRING_CONFIG_ADDITIONAL_LOCATION=file:/etc/javafresher/
```

Spring Boot sẽ xét các external configuration locations và profile-specific variants theo rules của framework. [Home](https://docs.spring.io/spring-boot/reference/features/external-config.html?utm_source=chatgpt.com)

Điểm vận hành cần nhớ:

**Không sửa WAR chỉ để thay environment.**

---

# 12. Ports — đừng nhầm application port và Tomcat port

Đây là một phần cực kỳ quan trọng của Day 13.

Có hai mô hình khác nhau.

## Mô hình A — standalone executable JAR

```text
java -jar app.jar
        │
        ▼
Spring Boot
embedded Tomcat
        │
        ▼
port 8080
```

Ở đây application sở hữu web server.

Với Spring Boot embedded container, một property như:

```properties
server.port=8081
```

có thể cấu hình listen port của embedded server. [Home](https://docs.spring.io/spring-boot/3.5/reference/web/servlet.html?utm_source=chatgpt.com)

---

## Mô hình B — WAR deployed vào external Tomcat

```text
systemd
   │
   ▼
Tomcat
Connector :8080
   │
   ├── app1.war
   └── javafresher.war
```

Ở đây **external Tomcat sở hữu listening connector**.

Port thường được cấu hình trong:

```text
$CATALINA_BASE/conf/server.xml
```

Ví dụ:

```xml
<Connector port="8080"
           protocol="HTTP/1.1" />
```

Đây là container-wide setting.

Nếu Tomcat đang chứa 5 ứng dụng:

```text
/app1
/app2
/app3
/app4
/javafresher
```

thay Connector:

```text
8080 → 8081
```

có thể tác động **tất cả năm ứng dụng**.

Đó là blast radius.

---

# 13. Kiểm tra port

Trước thay đổi:

```bash
sudo ss -ltnp
```

Hoặc:

```bash
sudo ss -ltnp | grep ':8080'
```

Ví dụ output:

```text
LISTEN 0 100 *:8080 *:* users:(("java",pid=2468,fd=55))
```

Đây là **example output**, không phải output bắt buộc.

Expected invariant là:

```text
TCP 8080 ở trạng thái LISTEN
và process sở hữu socket là Java/Tomcat mong đợi.
```

Test HTTP:

```bash
curl -v http://127.0.0.1:8080/
```

---

# 14. Port troubleshooting mental model

Nếu:

```bash
curl http://127.0.0.1:8080
```

trả:

```text
Connection refused
```

thì đừng bắt đầu bằng application code.

Đi theo layer:

```text
Is service active?
    ↓
Does Java process exist?
    ↓
Is 8080 listening?
    ↓
Which PID owns 8080?
    ↓
Did Tomcat Connector initialize?
    ↓
Application deployed?
```

Commands:

```bash
systemctl status tomcat
pgrep -af java
ss -ltnp | grep ':8080'
journalctl -u tomcat --since '-10 min'
```

---

# 15. Context Path

Port trả lời:

> Tôi kết nối tới process nào?

Context path trả lời:

> Trong Tomcat đó tôi muốn application nào?

Ví dụ:

```text
http://server:8080/javafresher/users
                     └──────────┘
                      context
```

---

# 16. Context path khi deploy WAR

Trong automatic deployment của Tomcat, context path thường được suy ra từ filename.

Ví dụ:

| WAR               | Context path   |
| ----------------- | -------------- |
| `javafresher.war` | `/javafresher` |
| `orders.war`      | `/orders`      |
| `ROOT.war`        | `/`            |
| `foo#bar.war`     | `/foo/bar`     |

Tomcat documentation hiện hành mô tả rõ quan hệ giữa WAR filename, context name và context path. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/config/context.html?utm_source=chatgpt.com)

Vì vậy nếu có:

```text
$CATALINA_BASE/webapps/javafresher.war
```

thì request thông thường:

```bash
curl http://127.0.0.1:8080/javafresher/
```

Không phải:

```bash
curl http://127.0.0.1:8080/
```

trừ khi app được deploy tại root context.

---

# 17. Phân biệt lỗi port với lỗi context path

Đây là một diagnostic rule rất giá trị.

### Case A

```text
curl: (7) Failed to connect
Connection refused
```

Bạn chưa vào được HTTP listener.

Investigate:

```text
process/socket/port/service
```

### Case B

```text
HTTP/1.1 404
```

TCP + HTTP server đã hoạt động.

Investigate:

```text
context path
application deployment
endpoint mapping
```

Nói cách khác:

```text
Connection refused
→ lower layer

HTTP 404
→ web layer đã sống, nhưng resource/context không khớp
```

---

# 18. JVM Memory — `-Xms` và `-Xmx`

Đây là phần rất quan trọng cho cả exam lẫn Operations.

Hai options nền tảng:

```text
-Xms
-Xmx
```

Ví dụ:

```bash
-Xms256m
-Xmx512m
```

`-Xms` điều khiển initial/minimum heap sizing còn `-Xmx` đặt maximum Java heap size. Oracle cũng nhấn mạnh rằng tổng RAM khả dụng là yếu tố lớn đối với GC và heap không nên được xem tách rời khỏi tài nguyên hệ thống. [docs.oracle.com](https://docs.oracle.com/en/java/javase/25/gctuning/factors-affecting-garbage-collection-performance.html?utm_source=chatgpt.com)

---

# 19. Heap không phải toàn bộ JVM memory

Đây là một bẫy quan trọng.

Giả sử:

```text
-Xmx512m
```

không có nghĩa Java process sẽ tối đa chỉ dùng 512 MB RAM.

JVM còn cần memory cho:

```text
Java Heap
+
Metaspace
+
Thread stacks
+
Code cache
+
Direct/native buffers
+
JVM native structures
+
GC structures
+
libraries
```

Và host còn cần:

```text
Linux kernel
systemd
Tomcat native components
other applications
filesystem cache
monitoring
PostgreSQL...
```

Do đó:

```text
Host RAM = 1 GB

-Xmx1g
```

là một thiết kế đáng ngờ.

Không còn headroom cho OS/native memory.

---

# 20. Trước khi tune JVM phải inspection

Không bắt đầu bằng:

```bash
-Xmx4g
```

vì “4 GB nghe có vẻ nhanh”.

Trước tiên:

```bash
free -h
```

```bash
ps -eo pid,user,rss,vsz,cmd --sort=-rss | head
```

```bash
systemctl show tomcat -p MemoryMax
```

Nếu có cgroup/container limit thì phải xem giới hạn đó nữa.

Question phải trả lời:

```text
Host có bao nhiêu RAM?
Tomcat đang dùng bao nhiêu?
Có app/database khác không?
cgroup limit là bao nhiêu?
Current Xmx là gì?
Application load là bao nhiêu?
Có OOM hay GC pressure thật không?
```

---

# 21. `CATALINA_OPTS` versus `JAVA_OPTS`

Đây là kiến thức Tomcat rất quan trọng.

Tomcat khuyến nghị Java options dùng riêng cho runtime Tomcat như heap, GC, JMX... nên đặt trong:

```bash
CATALINA_OPTS
```

Ví dụ:

```bash
CATALINA_OPTS="-Xms256m -Xmx512m"
```

Trong khi:

```bash
JAVA_OPTS
```

được áp dụng cho nhiều Tomcat commands khác, bao gồm startup/stop/version tùy invocation.

Apache Tomcat còn ghi rõ **không nên đặt memory limits vào `JAVA_OPTS`; heap settings phù hợp với `CATALINA_OPTS`**. [GitHub](https://github.com/apache/tomcat/blob/main/RUNNING.txt?utm_source=chatgpt.com)

Đây là câu exam rất đáng nhớ:

```text
Tomcat runtime-only JVM options
→ CATALINA_OPTS

Generic options needed by multiple Tomcat commands
→ JAVA_OPTS
```

---

# 22. `setenv.sh`

Nếu Tomcat được khởi động bằng standard Tomcat scripts:

```text
startup.sh
catalina.sh
```

một nơi chuẩn để custom environment là:

```text
$CATALINA_BASE/bin/setenv.sh
```

Ví dụ:

```bash
CATALINA_OPTS="-Xms256m -Xmx512m"
```

Tomcat khuyến nghị `setenv.sh` để giữ customization tách khỏi distributed startup scripts. [GitHub](https://github.com/apache/tomcat/blob/main/RUNNING.txt?utm_source=chatgpt.com)

Không sửa trực tiếp:

```text
bin/catalina.sh
```

chỉ để thêm `-Xmx`.

Vì upgrade Tomcat có thể overwrite file đó và làm mất customization.

---

# 23. Nhưng có một bẫy lớn với `setenv.sh`

`setenv.sh` chỉ có tác dụng nếu startup path thật sự đi qua Tomcat startup scripts.

Ví dụ unit:

```ini
ExecStart=/opt/tomcat/bin/catalina.sh run
```

thì hợp lý.

Nhưng nếu systemd unit chạy:

```ini
ExecStart=/usr/bin/java ...
```

trực tiếp thì:

```text
setenv.sh
```

có thể không bao giờ được đọc.

Do đó đừng hỏi:

> Tôi đã sửa `setenv.sh`, sao `-Xmx` không đổi?

Trước hết:

```bash
systemctl cat tomcat
```

xem **ExecStart thực tế**.

Đây chính là evidence-driven troubleshooting.

---

# 24. Cách chứng minh `-Xmx` thực sự active

Không tin mỗi config file.

Kiểm tra process runtime.

Lấy PID:

```bash
PID=$(systemctl show -p MainPID --value tomcat)
echo "$PID"
```

Nếu dùng JDK có `jcmd`:

```bash
sudo -u tomcat jcmd "$PID" VM.command_line
```

```bash
sudo -u tomcat jcmd "$PID" VM.flags
```

Có thể xem heap:

```bash
sudo -u tomcat jcmd "$PID" GC.heap_info
```

Hoặc tối thiểu:

```bash
ps -p "$PID" -o pid,user,args
```

Expected invariant:

```text
-Xms256m
-Xmx512m
```

xuất hiện hoặc effective JVM settings tương ứng được xác nhận.

**Configuration file tồn tại không phải evidence đủ. Runtime state mới là evidence mạnh hơn.**

---

# 25. Logs — ba loại log cần phân biệt

Một Java/Tomcat environment thường có ít nhất ba nguồn.

| Source           | Nội dung                                |
| ---------------- | --------------------------------------- |
| systemd journal  | lifecycle của service, stdout/stderr    |
| Tomcat logs      | Catalina/container/deployment           |
| application logs | business/application/framework messages |

Bạn phải biết mình đang troubleshoot layer nào.

---

# 26. systemd journal

Ví dụ:

```bash
journalctl -u tomcat
```

Last 100 entries:

```bash
journalctl -u tomcat -n 100 --no-pager
```

Khoảng thời gian:

```bash
journalctl -u tomcat --since '-10 minutes' --no-pager
```

Live:

```bash
journalctl -fu tomcat
```

`journalctl -u` filter log theo unit và là công cụ chuẩn để đọc journal của systemd service. [freedesktop.org](https://www.freedesktop.org/software/systemd/man/255/journalctl.html?utm_source=chatgpt.com)

---

# 27. Tomcat logs

Thông thường có:

```text
$CATALINA_BASE/logs/
```

Ví dụ có thể gặp:

```text
catalina.YYYY-MM-DD.log
localhost.YYYY-MM-DD.log
localhost_access_log....
```

Exact files phụ thuộc logging configuration/version.

Tomcat JULI `FileHandler` mặc định có thể ghi rotated log files dưới `${catalina.base}/logs`. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/api/org/apache/juli/FileHandler.html?utm_source=chatgpt.com)

Kiểm tra:

```bash
ls -lah "$CATALINA_BASE/logs"
```

```bash
tail -n 100 "$CATALINA_BASE/logs/catalina."*.log
```

Không assume một filename duy nhất.

---

# 28. `catalina.out` không phải chân lý duy nhất

Nếu Tomcat chạy qua một startup mechanism redirect stdout/stderr vào file thì có thể có:

```text
catalina.out
```

Nhưng nếu chạy foreground:

```bash
catalina.sh run
```

dưới systemd, stdout/stderr có thể đi thẳng vào journal.

Do đó câu:

> “Tomcat không có `catalina.out`, chắc không có log.”

là sai.

Hãy xác định logging path thực tế.

---

# 29. Log level

Các level phổ biến:

```text
TRACE
DEBUG
INFO
WARN
ERROR
```

Production không nên bật:

```text
DEBUG
```

vĩnh viễn nếu không có lý do vì:

```text
log volume tăng
disk usage tăng
noise tăng
có thể expose dữ liệu
performance có thể bị ảnh hưởng
```

Nếu cần debug:

```text
approval
↓
enable temporarily
↓
capture evidence
↓
restore normal level
```

---

# 30. Safe Restart — đây là kỹ năng cốt lõi của Day 13

`systemctl restart tomcat` là một command.

**Safe restart là một procedure.**

Không giống nhau.

Một safe restart phải có:

```text
pre-check
→ backup
→ syntax/config validation
→ blast-radius assessment
→ restart
→ service validation
→ port validation
→ application validation
→ log validation
→ rollback decision
→ evidence
```

---

# 31. Trước restart phải trả lời gì?

Bạn phải biết:

| Câu hỏi                           | Tại sao?           |
| --------------------------------- | ------------------ |
| Tôi đang restart service nào?     | tránh wrong target |
| Nó chứa bao nhiêu application?    | blast radius       |
| Ai đang sử dụng nó?               | availability       |
| Configuration nào vừa đổi?        | traceability       |
| Backup ở đâu?                     | rollback           |
| Expected port/context là gì?      | validation         |
| Expected endpoint là gì?          | validation         |
| Restart window được approve chưa? | operation control  |

Trong production, “restart để thử xem” là anti-pattern.

---

# 32. Baseline trước thay đổi

Giả sử service là `tomcat`.

```bash
date
hostname
```

```bash
systemctl status tomcat --no-pager -l
```

```bash
systemctl show tomcat \
  -p MainPID \
  -p ActiveState \
  -p SubState
```

```bash
sudo ss -ltnp | grep ':8080'
```

```bash
curl -i http://127.0.0.1:8080/javafresher/
```

```bash
journalctl -u tomcat -n 30 --no-pager
```

Đây chính là **before-change evidence**.

---

# 33. Backup configuration trước change

Ví dụ:

```bash
sudo cp -a \
  /etc/javafresher/javafresher.env \
  /etc/javafresher/javafresher.env.bak-$(date +%Y%m%d-%H%M%S)
```

Nếu thay `server.xml`:

```bash
sudo cp -a \
  /opt/tomcat/conf/server.xml \
  /opt/tomcat/conf/server.xml.bak-$(date +%Y%m%d-%H%M%S)
```

Không backup WAR chỉ vì đổi environment file nếu WAR không đổi.

Backup phải tương ứng với **change scope**.

---

# 34. Validate config trước restart

Nếu sửa Tomcat XML có thể chạy:

```bash
/opt/tomcat/bin/catalina.sh configtest
```

Nếu đơn giản muốn check XML syntax và có `xmllint`:

```bash
xmllint --noout /opt/tomcat/conf/server.xml
```

Nhưng cần hiểu khác nhau:

```text
xmllint
→ XML syntax

Tomcat configtest
→ Tomcat startup/config interpretation
```

XML hợp lệ chưa chắc Tomcat configuration semantics đúng.

---

# 35. Safe restart sequence

Trong lab:

```bash
sudo systemctl restart tomcat
```

Ngay lập tức:

```bash
systemctl is-active tomcat
```

Expected:

```text
active
```

Nhưng `active` **chưa đủ**.

Tiếp tục:

```bash
systemctl status tomcat --no-pager -l
```

```bash
PID=$(systemctl show -p MainPID --value tomcat)
echo "$PID"
```

```bash
sudo ss -ltnp | grep ':8080'
```

```bash
journalctl -u tomcat --since '-5 minutes' --no-pager
```

Sau đó HTTP validation.

---

# 36. Endpoint validation

Không chỉ:

```bash
curl localhost:8080
```

mà phải gọi đúng application/context.

Ví dụ:

```bash
curl -i \
  http://127.0.0.1:8080/javafresher/
```

Nếu application có health endpoint documented:

```bash
curl -i \
  http://127.0.0.1:8080/javafresher/health
```

Nếu là Spring Boot và Actuator thực sự đã được enable/expose:

```bash
curl -i \
  http://127.0.0.1:8080/javafresher/actuator/health
```

Nhưng **không được assume `/actuator/health` tồn tại** chỉ vì application là Java.

---

# 37. Validation hierarchy

Một restart thành công phải được chứng minh từng layer:

```text
systemd active
      ↓
Java PID exists
      ↓
expected TCP port LISTEN
      ↓
Tomcat startup successful
      ↓
WAR/context deployed
      ↓
HTTP endpoint responds
      ↓
business/health response correct
      ↓
no unexpected ERROR in logs
```

Nếu bạn chỉ có:

```text
systemctl status → active
```

thì chưa đủ chứng minh application healthy.

---

# 38. LAB DAY 13 — Environment-specific Configuration + JVM Tuning + Safe Restart

Đây là phần thực hành bắt buộc theo syllabus.

## Mục tiêu

Sau lab bạn phải chứng minh được:

```text
1. biết current environment
2. tạo external environment configuration
3. activate đúng profile/config
4. set approved JVM heap options
5. restart Tomcat an toàn
6. chứng minh JVM nhận settings
7. chứng minh port/context hoạt động
8. đọc logs
9. inject một failure
10. rollback thành công
11. lưu evidence
```

---

# 39. Lab Safety

**Chỉ thực hiện change/failure injection trên dedicated lab VM/non-production.**

Đặc biệt không tự ý:

```text
đổi production Tomcat port
giảm production Xmx
restart production service
sửa systemd unit
```

mà không có approval/runbook/rollback.

---

# 40. Lab Step 1 — Discover môi trường thật

```bash
hostnamectl
```

```bash
java -version
```

```bash
systemctl cat tomcat
```

Tìm:

```text
User=
ExecStart=
Environment=
EnvironmentFile=
```

Sau đó:

```bash
systemctl status tomcat --no-pager -l
```

```bash
ps -ef | grep '[o]rg.apache.catalina.startup.Bootstrap'
```

```bash
sudo ss -ltnp
```

Xác định:

```text
SERVICE_NAME=
SERVICE_USER=
CATALINA_HOME=
CATALINA_BASE=
CONNECTOR_PORT=
WAR_NAME=
CONTEXT_PATH=
HEALTH_OR_TEST_ENDPOINT=
```

Đây là inventory đầu lab.

---

# 41. Step 2 — Evidence baseline

Ví dụ:

```bash
mkdir -p ~/day13-evidence
```

```bash
date | tee ~/day13-evidence/00-date.txt
```

```bash
java -version \
  2>&1 | tee ~/day13-evidence/01-java-version.txt
```

```bash
systemctl status tomcat --no-pager -l \
  | tee ~/day13-evidence/02-before-status.txt
```

```bash
sudo ss -ltnp \
  | tee ~/day13-evidence/03-before-ports.txt
```

```bash
journalctl -u tomcat -n 50 --no-pager \
  | tee ~/day13-evidence/04-before-journal.txt
```

---

# 42. Step 3 — Xác định current JVM configuration

```bash
PID=$(systemctl show -p MainPID --value tomcat)

echo "$PID"
```

```bash
ps -p "$PID" -o pid,user,args
```

Nếu có `jcmd`:

```bash
sudo -u tomcat jcmd "$PID" VM.command_line
```

```bash
sudo -u tomcat jcmd "$PID" VM.flags
```

Save evidence:

```bash
sudo -u tomcat jcmd "$PID" VM.flags \
  > ~/day13-evidence/05-before-jvm-flags.txt
```

---

# 43. Step 4 — Capacity check trước JVM change

```bash
free -h
```

```bash
df -h
```

```bash
ps -eo pid,user,rss,vsz,cmd --sort=-rss | head -20
```

Không set heap trước khi biết RAM.

Reference lab-only target:

```text
-Xms256m
-Xmx512m
```

Đây **không phải universal production recommendation**.

Nó chỉ là value minh họa cho lab có đủ RAM.

---

# 44. Step 5 — Tạo environment-specific configuration

```bash
sudo install -d \
  -o root \
  -g tomcat \
  -m 0750 \
  /etc/javafresher
```

Tạo:

```bash
sudo editor /etc/javafresher/javafresher.env
```

Ví dụ tối thiểu:

```text
APP_ENV=lab
CATALINA_OPTS="-Xms256m -Xmx512m"
```

Nếu application là Spring Boot:

```text
SPRING_PROFILES_ACTIVE=lab
SPRING_CONFIG_ADDITIONAL_LOCATION=file:/etc/javafresher/
CATALINA_OPTS="-Xms256m -Xmx512m"
```

Permissions:

```bash
sudo chown root:tomcat /etc/javafresher/javafresher.env
sudo chmod 0640 /etc/javafresher/javafresher.env
```

Check:

```bash
sudo ls -l /etc/javafresher/javafresher.env
```

---

# 45. Step 6 — Spring Boot profile file, nếu app hỗ trợ

**Supplementary Spring Boot implementation.**

```bash
sudo editor /etc/javafresher/application-lab.properties
```

Ví dụ không chứa secret:

```properties
app.environment=lab
logging.level.root=INFO
```

Permissions:

```bash
sudo chown root:tomcat \
  /etc/javafresher/application-lab.properties

sudo chmod 0640 \
  /etc/javafresher/application-lab.properties
```

Không tự invent property application-specific như:

```properties
app.environment
```

nếu provided application không support nó.

Hãy dùng property contract/runbook của provided application.

---

# 46. Step 7 — Kết nối EnvironmentFile với systemd

Kiểm tra trước:

```bash
systemctl cat tomcat
```

Nếu chưa có, tạo drop-in:

```bash
sudo systemctl edit tomcat
```

Nội dung:

```ini
[Service]
EnvironmentFile=/etc/javafresher/javafresher.env
```

Sau khi **unit definition** đổi:

```bash
sudo systemctl daemon-reload
```

Check:

```bash
systemctl cat tomcat
```

Phải thấy drop-in mới.

---

# 47. Step 8 — Pre-restart validation

```bash
sudo test -r /etc/javafresher/javafresher.env \
  && echo "environment file readable"
```

Nếu service account là `tomcat`:

```bash
sudo -u tomcat \
  test -r /etc/javafresher/javafresher.env \
  && echo "tomcat can read environment file"
```

Tomcat config test:

```bash
sudo -u tomcat \
  /opt/tomcat/bin/catalina.sh configtest
```

Path phải đổi theo môi trường thực tế.

---

# 48. Step 9 — Safe restart

Ghi thời điểm:

```bash
date
```

Restart:

```bash
sudo systemctl restart tomcat
```

Ngay lập tức:

```bash
systemctl is-active tomcat
```

```bash
systemctl status tomcat --no-pager -l
```

Nếu không active:

**không restart liên tục.**

Đi đọc evidence.

```bash
journalctl -u tomcat --since '-5 min' --no-pager
```

---

# 49. Step 10 — Verify PID và port

```bash
PID=$(systemctl show -p MainPID --value tomcat)

echo "$PID"
```

PID phải khác 0.

```bash
ps -p "$PID" -o pid,user,lstart,args
```

Port:

```bash
sudo ss -ltnp | grep ':8080'
```

Expected invariant:

```text
Tomcat Java process exists.
Expected connector port is LISTEN.
```

---

# 50. Step 11 — Verify environment

Chỉ check non-sensitive variables:

```bash
sudo tr '\0' '\n' < "/proc/$PID/environ" \
  | grep '^APP_ENV='
```

Nếu Spring Boot:

```bash
sudo tr '\0' '\n' < "/proc/$PID/environ" \
  | grep '^SPRING_PROFILES_ACTIVE='
```

Expected:

```text
APP_ENV=lab
```

hoặc:

```text
SPRING_PROFILES_ACTIVE=lab
```

---

# 51. Step 12 — Verify JVM heap runtime

```bash
sudo -u tomcat jcmd "$PID" VM.command_line
```

```bash
sudo -u tomcat jcmd "$PID" VM.flags
```

```bash
sudo -u tomcat jcmd "$PID" GC.heap_info
```

Bạn phải xác minh effective heap tương ứng với:

```text
-Xms256m
-Xmx512m
```

Chứ không chỉ:

> “Tôi nhìn trong config file thấy vậy.”

---

# 52. Step 13 — Verify endpoint

Ví dụ:

```bash
curl -i \
  http://127.0.0.1:8080/javafresher/
```

Hoặc documented health endpoint:

```bash
curl -i \
  http://127.0.0.1:8080/javafresher/health
```

Có thể lấy HTTP code:

```bash
curl -sS \
  -o /dev/null \
  -w 'HTTP %{http_code}\n' \
  http://127.0.0.1:8080/javafresher/
```

Expected invariant phải dựa vào application:

```text
HTTP 200
```

hoặc một response documented khác.

Không phải mọi endpoint đều phải trả 200.

---

# 53. Step 14 — Validate startup logs

```bash
journalctl -u tomcat \
  --since '-10 min' \
  --no-pager
```

Tìm lỗi:

```bash
journalctl -u tomcat \
  --since '-10 min' \
  --no-pager \
  | grep -Ei 'error|exception|fatal|failed'
```

Tomcat logs:

```bash
ls -lah /opt/tomcat/logs
```

Sau đó:

```bash
grep -RniE \
  'error|exception|fatal|failed' \
  /opt/tomcat/logs
```

Lưu ý `grep error` có thể có false positives.

Phải đọc context.

---

# 54. Step 15 — Save after-change evidence

```bash
systemctl status tomcat --no-pager -l \
  > ~/day13-evidence/10-after-status.txt
```

```bash
sudo ss -ltnp \
  > ~/day13-evidence/11-after-ports.txt
```

```bash
journalctl -u tomcat --since '-10 min' --no-pager \
  > ~/day13-evidence/12-after-journal.txt
```

```bash
sudo -u tomcat jcmd "$PID" VM.flags \
  > ~/day13-evidence/13-after-jvm-flags.txt
```

HTTP:

```bash
curl -i http://127.0.0.1:8080/javafresher/ \
  > ~/day13-evidence/14-endpoint.txt
```

---

# 55. Failure Injection — JVM configuration error

Đây là bài troubleshooting quan trọng.

**Chỉ dedicated lab.**

Backup:

```bash
sudo cp -a \
  /etc/javafresher/javafresher.env \
  /etc/javafresher/javafresher.env.good
```

Tạm thay:

```text
CATALINA_OPTS="-Xms256m -XmxBAD"
```

Đây là intentional invalid JVM option.

Restart:

```bash
sudo systemctl restart tomcat
```

Dự kiến service fail.

Không panic, không restart 10 lần.

---

# 56. Troubleshooting failure

Symptom:

```bash
systemctl status tomcat
```

có thể cho thấy:

```text
failed
```

Evidence:

```bash
journalctl -u tomcat -n 100 --no-pager
```

Bạn phải tìm JVM startup error liên quan tới invalid heap option.

Reasoning:

```text
Recent change:
CATALINA_OPTS changed

Service:
fails before normal Tomcat startup

Evidence:
JVM rejects -XmxBAD

Hypothesis:
invalid JVM memory syntax

Safest test:
compare environment file with backup
```

```bash
sudo diff -u \
  /etc/javafresher/javafresher.env.good \
  /etc/javafresher/javafresher.env
```

Root cause:

```text
Invalid -Xmx argument.
```

---

# 57. Rollback

Restore:

```bash
sudo cp -a \
  /etc/javafresher/javafresher.env.good \
  /etc/javafresher/javafresher.env
```

Restart:

```bash
sudo systemctl restart tomcat
```

Validate:

```bash
systemctl is-active tomcat
```

```bash
sudo ss -ltnp | grep ':8080'
```

```bash
curl -i \
  http://127.0.0.1:8080/javafresher/
```

```bash
journalctl -u tomcat \
  --since '-5 min' \
  --no-pager
```

Runtime JVM flags:

```bash
PID=$(systemctl show -p MainPID --value tomcat)

sudo -u tomcat jcmd "$PID" VM.flags
```

Đây mới là **complete rollback**.

Copy file cũ về nhưng không restart/validate chưa được coi là recovery hoàn tất.

---

# 58. Troubleshooting matrix Day 13

| Symptom                               | Evidence nên kiểm tra     | Nguyên nhân thường gặp              | Hướng xử lý                   |
| ------------------------------------- | ------------------------- | ----------------------------------- | ----------------------------- |
| `tomcat.service failed`               | `journalctl -u tomcat`    | invalid JVM/config                  | rollback config               |
| Active nhưng connection refused       | `ss -ltnp`                | port không listen                   | check Connector/startup       |
| Port listen nhưng HTTP 404            | `curl -i`, WAR            | context path sai                    | check WAR/context             |
| `-Xmx` không đổi                      | `jcmd`, `systemctl cat`   | config không được startup path dùng | inspect `ExecStart`           |
| Profile không active                  | process env + startup log | environment không tới JVM/app       | inspect EnvironmentFile       |
| Config file ignored                   | app logs/config path      | sai location/precedence             | verify effective config       |
| Service fail “address already in use” | `ss -ltnp`                | port collision                      | identify owner, rollback      |
| Application cannot read config        | log + `ls -l`             | permission/ownership                | correct least privilege       |
| Restart OK rồi OOM                    | logs/jcmd/memory          | Xmx quá thấp hoặc leak              | rollback/escalate             |
| Old behavior remains                  | PID, runtime settings     | chưa restart hoặc sửa sai instance  | identify actual CATALINA_BASE |

---

# 59. Một scenario rất quan trọng: `-Xmx` không có tác dụng

Bạn sửa:

```text
/opt/tomcat/bin/setenv.sh
```

thành:

```bash
CATALINA_OPTS="-Xms256m -Xmx512m"
```

restart xong:

```bash
jcmd PID VM.flags
```

vẫn thấy heap cũ.

Đừng tiếp tục sửa random.

Evidence:

```bash
systemctl cat tomcat
```

Nếu:

```ini
ExecStart=/usr/bin/java -Xmx1g ...
```

thì root cause gần như rõ:

```text
systemd invokes Java directly
↓
Tomcat catalina.sh not involved
↓
setenv.sh not consumed
```

Đây là ví dụ điển hình của:

> **configuration file ≠ effective configuration.**

---

# 60. Scenario: service active nhưng ứng dụng vẫn lỗi

Giả sử:

```bash
systemctl is-active tomcat
```

trả:

```text
active
```

và:

```bash
ss -ltnp | grep 8080
```

thấy Java listen.

Nhưng:

```bash
curl http://localhost:8080/javafresher/
```

trả:

```text
404
```

Lúc này không restart Linux.

Investigate deployment:

```bash
ls -lah /opt/tomcat/webapps
```

Nếu bạn thấy:

```text
orders.war
```

chứ không có:

```text
javafresher.war
```

thì `/javafresher` không thể magically tồn tại.

Hoặc WAR thật là:

```text
fresher.war
```

thì context mặc định có thể là:

```text
/fresher
```

Tomcat naming rules liên kết context path với WAR/context filename. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/config/context.html?utm_source=chatgpt.com)

---

# 61. Scenario: Port conflict

Tomcat fail với thông tin tương đương:

```text
Address already in use
```

Không kill Java process ngẫu nhiên.

Check:

```bash
sudo ss -ltnp | grep ':8080'
```

Ví dụ:

```text
LISTEN ... users:(("java",pid=1111,...))
```

Xác định PID:

```bash
ps -fp 1111
```

Question:

```text
PID 1111 là application hợp lệ?
Service nào sở hữu?
Port assignment có sai không?
Recent change là gì?
```

Trong production, việc kill process khác để “giải phóng port” có thể tạo outage thứ hai.

---

# 62. Scenario: JVM `OutOfMemoryError`

Nếu log:

```text
java.lang.OutOfMemoryError: Java heap space
```

không tự động kết luận:

```text
increase Xmx
```

Có ít nhất hai nhóm khả năng:

```text
Heap legitimately undersized

hoặc

Memory leak / abnormal workload
```

Oracle troubleshooting documentation cũng lưu ý `OutOfMemoryError` có thể liên quan tới application giữ reference không còn cần thiết, tức memory leak. [docs.oracle.com](https://docs.oracle.com/en/java/javase/25/troubleshoot/troubleshooting-memory-leaks.html?utm_source=chatgpt.com)

L1 nên thu evidence:

```bash
free -h
jcmd PID GC.heap_info
jcmd PID VM.flags
ps ...
application logs
request/load timeline
recent changes
```

Nếu cần heap dump/code analysis thì thường đã vượt khỏi basic L1 scope và cần escalation.

---

# 63. Mistakes / Anti-patterns phải tránh

1. Sửa config nhưng không biết service startup path.

2. `export SPRING_PROFILES_ACTIVE=prod` trong SSH session rồi nghĩ systemd service tự nhận.

3. Sửa trực tiếp WAR để thay DEV/UAT/PROD config.

4. Hard-code production endpoint trong source.

5. Dùng `JAVA_OPTS` cho `-Xmx` mà không hiểu Tomcat startup semantics.

6. Set `-Xmx` bằng toàn bộ RAM host.

7. Chỉ chạy `systemctl restart`, không validation.

8. Chỉ thấy `active` rồi kết luận app healthy.

9. Thấy HTTP `404` rồi troubleshoot firewall.

10. Thấy connection refused rồi troubleshoot controller code.

11. Thay port external Tomcat mà không nhận ra mọi WAR trên cùng Tomcat đều chịu ảnh hưởng.

12. Copy toàn bộ `/proc/PID/environ` vào ticket, vô tình leak secret.

13. Bật DEBUG logging vô thời hạn.

14. Restart liên tục để “xem có tự hết không”.

15. Không backup known-good configuration trước change.

---

# 64. Security & Least Privilege

Tomcat không nên chạy bằng `root` chỉ vì bạn muốn tiện truy cập file.

Dùng dedicated service account:

```text
tomcat
```

Application configuration nên có ownership/permission phù hợp:

```text
root:tomcat
0640
```

Ví dụ:

```bash
sudo chown root:tomcat /etc/javafresher/application-lab.properties
sudo chmod 0640 /etc/javafresher/application-lab.properties
```

Nếu file chứa non-sensitive configuration, đây là reasonable lab pattern.

Secrets sẽ được học sâu hơn Day 14.

Ngoài ra không mở remote debug/JMX public chỉ để troubleshoot nhanh. Debug ports có thể tạo security exposure nghiêm trọng.

---

# 65. Operations / Recovery / Rollback

Một configuration change tốt nên có record dạng:

| Field            | Giá trị                                        |
| ---------------- | ---------------------------------------------- |
| Change           | JVM heap + application profile                 |
| Before           | `Xmx=...`, profile=...                         |
| After            | `Xms256m`, `Xmx512m`, profile=lab              |
| Service          | `tomcat.service`                               |
| Impact           | Tomcat restart                                 |
| Backup           | path của known-good config                     |
| Validation       | systemd + PID + port + HTTP + logs             |
| Rollback trigger | startup fail / health fail / unexpected errors |
| Rollback         | restore config + restart + revalidate          |
| Evidence         | files trong `day13-evidence`                   |

Đây là cách Operations biến “tôi đã sửa” thành “tôi có thể chứng minh change được kiểm soát”.

---

# 66. L1 boundary và escalation

### L1-safe work thường bao gồm

Inspection:

```bash
systemctl status
journalctl
ss
curl
ps
free
df
```

Known approved configuration change.

Approved restart.

Rollback về known-good config.

Endpoint validation.

Evidence collection.

### Escalate khi

```text
Memory leak suspected
Unknown JVM crash
Repeated OOM after rollback
Application code exception requiring code fix
Unclear business configuration
No known-good rollback
Production port collision involving another service
Security/secret exposure
Change requires architecture decision
Heap dump/code profiling required
```

Fresher tốt không phải người “tự sửa mọi thứ”.

Fresher tốt là người biết:

```text
scope
evidence
safe action
rollback
escalation boundary
```

---

# 67. Exam Focus — Day 13

Bạn cần đặc biệt chắc các điểm sau:

| Question                                     | Phải trả lời được                                       |
| -------------------------------------------- | ------------------------------------------------------- |
| Externalized config là gì?                   | tách environment config khỏi artifact/code              |
| Vì sao cần?                                  | same artifact, different environment                    |
| Env var khác shell var thế nào?              | process environment/scope                               |
| Profile là gì?                               | tập config theo runtime/environment                     |
| Port standalone JAR nằm ở đâu?               | embedded server/application                             |
| Port WAR external Tomcat nằm ở đâu?          | Tomcat Connector                                        |
| Context path là gì?                          | URI prefix chọn application                             |
| `app.war` thường thành context gì?           | `/app`                                                  |
| `ROOT.war`?                                  | `/`                                                     |
| `-Xms`?                                      | initial/minimum heap setting                            |
| `-Xmx`?                                      | maximum heap                                            |
| Heap có bằng process RAM không?              | không                                                   |
| Tomcat heap nên đặt đâu?                     | `CATALINA_OPTS`                                         |
| Khi nào daemon-reload?                       | unit/drop-in definition changed                         |
| Active service có đủ chứng minh app healthy? | không                                                   |
| Safe restart gồm gì?                         | precheck→backup→restart→multi-layer validation→rollback |
| 404 vs connection refused?                   | web-layer mismatch vs listener/connectivity             |
| Evidence quan trọng?                         | status, PID, port, logs, endpoint, JVM flags            |

---

# 68. Knowledge Check có đáp án

| Câu hỏi                                                                                | Đáp án / reasoning                                                         |
| -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| Bạn chạy `export APP_ENV=prod`; Tomcat systemd có chắc nhận không?                     | Không. Shell environment khác service environment.                         |
| `tomcat.service` active nhưng `curl localhost:8080` connection refused. Healthy không? | Không. Phải kiểm tra socket/port/startup.                                  |
| `curl :8080/app` trả 404. Port có đang hoạt động không?                                | Có khả năng rất cao HTTP listener hoạt động; investigate context/endpoint. |
| WAR tên `billing.war` mặc định context thường là gì?                                   | `/billing`.                                                                |
| Muốn root context bằng WAR naming convention?                                          | `ROOT.war`.                                                                |
| `-Xmx512m` có nghĩa process chỉ dùng ≤512MB RAM?                                       | Không. JVM còn native/non-heap memory.                                     |
| Đặt heap trong `JAVA_OPTS` hay `CATALINA_OPTS` với Tomcat standard scripts?            | `CATALINA_OPTS`.                                                           |
| Sửa drop-in systemd thì làm gì trước restart?                                          | `systemctl daemon-reload`.                                                 |
| Chỉ sửa nội dung EnvironmentFile thì có luôn cần daemon-reload?                        | Không; thường restart service để process đọc environment mới.              |
| Config file nói `-Xmx512m`; evidence mạnh nhất là gì?                                  | Verify effective runtime JVM flags bằng `jcmd`/process command line.       |
| Restart xong service active là hoàn tất change chưa?                                   | Chưa; cần port, endpoint, logs, runtime config validation.                 |
| Invalid `-Xmx` làm Tomcat fail. L1 action tốt?                                         | Capture evidence → restore known-good config → restart → validate.         |

---

# 69. Independent Practical Challenge

Không nhìn lại procedure chi tiết, hãy tự hoàn thành scenario sau trên lab machine:

```text
Application: Java WAR on Tomcat
Current environment: default
Desired environment: lab
Approved JVM:
    Xms = 256 MB
    Xmx = 512 MB

Requirements:
    externalize environment configuration
    configure lab profile where supported
    do not modify application WAR
    safely restart Tomcat
    verify service PID
    verify listening port
    verify correct context path
    verify HTTP endpoint
    verify effective JVM memory
    inspect logs
    retain before/after evidence

Failure:
    inject an invalid JVM memory argument
    prove root cause from logs
    rollback
    demonstrate recovery
```

Acceptance criteria:

| Criterion                  | Evidence                    |
| -------------------------- | --------------------------- |
| Configuration externalized | config/env file             |
| Artifact unchanged         | WAR checksum/path unchanged |
| JVM settings effective     | `jcmd VM.flags`             |
| Service healthy            | `systemctl`                 |
| Correct Java process       | PID/process command         |
| Correct socket             | `ss -ltnp`                  |
| Correct context            | `curl`                      |
| Logs clean after recovery  | journal/Tomcat logs         |
| Failure diagnosed          | failure log evidence        |
| Rollback successful        | restored endpoint + status  |
| Operational evidence       | complete evidence directory |

Nếu bạn chỉ giải thích được mà chưa thực hiện thì **chưa coi là mastery thực hành**.

---

# 70. Day 13 troubleshooting algorithm phải thuộc

Khi Java application có vấn đề sau configuration change, hãy tư duy:

```text
SYMPTOM
   ↓
WHAT SHOULD BE WORKING?
   ↓
WHAT CHANGED?
   ↓
WHAT IS THE SCOPE?
   ↓
systemd service?
   ↓
Java process?
   ↓
JVM flags/environment?
   ↓
listening port?
   ↓
Tomcat context?
   ↓
application endpoint?
   ↓
logs?
   ↓
hypothesis
   ↓
safest test
   ↓
root cause
   ↓
L1 fix OR escalation
   ↓
validation
   ↓
evidence
```

Đừng bắt đầu bằng:

```text
restart
restart
restart
```

---

# 71. Day 13 Mastery Map

Sau hôm nay, mental model bạn cần giữ là:

```text
              EXTERNAL CONFIG
                    │
          ┌─────────┴──────────┐
          │                    │
   environment vars       config files
          │                    │
          └─────────┬──────────┘
                    │
               systemd
                    │
                    ▼
              TOMCAT/JVM
        ┌───────────┼───────────┐
        │           │           │
      port       JVM heap     profile
        │        Xms/Xmx        │
        │           │           │
        └───────────┴───────────┘
                    │
                    ▼
             Java application
                    │
              context path
                    │
                    ▼
                 HTTP
                    │
                    ▼
               validation

Validation:
systemd
 → PID
 → JVM settings
 → socket
 → context
 → endpoint
 → logs
```

---

## Kết luận Day 13

Day 13 không đơn thuần dạy bạn “set environment variable” hay “set `-Xmx`”. Mục tiêu thực sự là học **configuration operations lifecycle**:

**inspect → externalize → configure → protect → validate configuration → restart safely → prove effective runtime state → test endpoint → inspect logs → diagnose failure → rollback → preserve evidence.**

Ba distinction quan trọng nhất bạn nên thuộc ngay hôm nay là:

**`config file` ≠ `effective runtime configuration`; `service active` ≠ `application healthy`; và `restart command` ≠ `safe restart procedure`.**

Đây cũng là nền trực tiếp cho **Day 14**, khi chúng ta thêm PostgreSQL connectivity, secrets, TLS, deployment validation và rollback vào chính pipeline này.
