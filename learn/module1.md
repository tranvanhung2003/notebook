# Module 1 — Java Application Runtime, Configuration & Deployment

## Day 12–14: Java runtime → Tomcat deployment → configuration → database/TLS → release & rollback

## 1. Syllabus Alignment

Module này bám trực tiếp vào ba ngày trong MASTER SYLLABUS:

- **Day 12:** Java application runtime architecture, JAR/WAR, Tomcat layout, Java 17, Java 25, deployment flow, service account, systemd integration.
- **Day 13:** externalized configuration, environment variables, profiles, ports, context path, JVM memory options, logs, safe restart.
- **Day 14:** database connectivity, secrets handling, TLS certificate basics, reverse-proxy awareness, deployment validation, rollback và reduced-scope consolidation. 00_MASTER_SYLLABUS_Java_Fresher…

Theo yêu cầu của bạn, phần này **không triển khai Guided Lab**, nhưng vẫn dùng command, config file, Java code, XML, SQL/JDBC URL và troubleshooting examples để bạn hiểu được cách vận hành thực tế.

Một số kiến thức tôi thêm ngoài syllabus, chẳng hạn Jakarta namespace compatibility, connection pool, health endpoint hay một số systemd hardening, sẽ được đánh dấu **Supplementary / Beyond the explicit syllabus**.

---

# PHẦN I — DAY 12

# Java Application Runtime Architecture

## 2. Mental model quan trọng nhất

Một Java web application chạy trên Linux có thể hình dung như sau:

```text
Linux
│
├── systemd
│    │
│    └── manages service
│
├── Java process
│    │
│    └── JVM
│         │
│         ├── Heap
│         ├── Metaspace
│         ├── Threads
│         ├── Garbage Collector
│         └── loaded classes
│
└── Tomcat
     │
     ├── HTTP Connector
     ├── Servlet Container
     └── Application WAR
           │
           ├── business logic
           ├── configuration
           └── JDBC
                 │
                 ▼
             PostgreSQL
```

Nếu thêm reverse proxy:

```text
Client
   |
 HTTPS :443
   |
   v
Nginx / Apache HTTP Server
   |
 HTTP :8080
   |
   v
Tomcat
   |
   v
Java application
   |
 JDBC :5432
   |
   v
PostgreSQL
```

Một fresher administrator phải phân biệt được các layer này.

Ví dụ người dùng báo:

> `https://app.example.com/orders` không vào được.

Không được lập tức kết luận:

> "Java app chết."

Có thể lỗi nằm ở:

```text
DNS
↓
network
↓
firewall / Security Group
↓
reverse proxy
↓
Tomcat port
↓
Tomcat process
↓
application deployment
↓
application code/config
↓
database
```

Đây là mental model sẽ quay lại rất nhiều ở Day 19–20.

---

# 3. JVM, JDK và Java application

## 3.1 JVM là gì?

**JVM — Java Virtual Machine** là runtime thực thi Java bytecode.

Thông thường:

```text
.java source
   |
 javac
   |
   v
.class bytecode
   |
   v
JVM
   |
   v
Machine instructions / OS resources
```

Ví dụ:

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello");
    }
}
```

Compile:

```bash
javac Hello.java
```

Tạo:

```text
Hello.class
```

Run:

```bash
java Hello
```

`java` khởi tạo JVM, JVM load class `Hello`, sau đó gọi:

```java
public static void main(String[] args)
```

---

# 4. JDK vs JVM

## JDK

**JDK — Java Development Kit** chứa runtime và các development/administration tools.

Ví dụ:

```text
java
javac
jar
jcmd
jstack
jmap
keytool
```

Kiểm tra:

```bash
java -version
javac -version
```

Ví dụ:

```text
openjdk version "25..."
```

Đường dẫn thực tế phụ thuộc distro/package:

```bash
which java
readlink -f "$(which java)"
```

Không nên hard-code một `JAVA_HOME` rồi giả định nó đúng trên mọi server.

Ví dụ:

```bash
echo "$JAVA_HOME"
```

---

# 5. Java 17 và Java 25

Syllabus yêu cầu cả **Java 17** và **Java 25**.

Tại thời điểm hiện tại, cả Java 17 và Java 25 đều thuộc nhóm LTS trong roadmap của Oracle; Java 25 được phát hành tháng 9/2025. [Oracle](https://www.oracle.com/java/technologies/java-se-support-roadmap.html?utm_source=chatgpt.com)

Điểm cần nhớ cho exam/operator không phải là thuộc mọi feature language mới mà là:

```text
Application
   +
Build target
   +
Framework/library compatibility
   +
Application server compatibility
   +
JVM version
```

phải tương thích với nhau.

---

## 5.1 Không phải cứ JVM mới hơn là chắc chắn app chạy

Ví dụ application được compile bằng JDK mới nhưng chạy bằng JVM quá cũ có thể gặp:

```text
UnsupportedClassVersionError
```

Mental model:

```text
Compile JDK 25
      |
      | produces bytecode requiring newer runtime
      v
Run Java 17
      |
      X
UnsupportedClassVersionError
```

Tuy nhiên khả năng tương thích cụ thể còn phụ thuộc compiler target:

```bash
javac --release 17 ...
```

Một application được build bằng JDK mới **có thể** target Java cũ nếu source/dependencies cho phép.

---

# 6. Java process

Khi application chạy:

```bash
java -jar application.jar
```

Linux nhìn thấy một process.

Kiểm tra:

```bash
ps -ef | grep java
```

hoặc tốt hơn:

```bash
pgrep -a java
```

Có thể thấy:

```text
tomcat  1842 ... /usr/bin/java -Xms512m -Xmx1024m ...
```

PID:

```text
1842
```

Application lúc này sử dụng:

- CPU
- memory
- file descriptors
- network sockets
- threads
- filesystem
- DNS
- environment variables
- credentials/configuration

Vì vậy Java application **không tách rời Linux administration**.

---

# 7. JAR

## 7.1 JAR là gì?

**JAR — Java Archive**.

Về cơ bản nó là một ZIP-compatible archive chứa:

```text
classes
resources
metadata
```

Xem:

```bash
jar tf app.jar
```

Hoặc:

```bash
unzip -l app.jar
```

Một JAR đơn giản:

```text
app.jar
├── META-INF/
│   └── MANIFEST.MF
├── com/
│   └── example/
│       └── Main.class
└── application.properties
```

---

## 7.2 Executable JAR

Nếu manifest/application packaging hỗ trợ entry point:

```bash
java -jar app.jar
```

Conceptually:

```text
Linux
   |
 java -jar
   |
   v
JVM
   |
   v
application.jar
```

Một số framework đóng gói HTTP server bên trong JAR.

Ví dụ phổ biến:

```text
Executable JAR
├── application
└── embedded web server
```

Application có thể tự listen:

```text
8080
```

Vì vậy **không phải mọi Java web application đều cần external Tomcat**.

---

# 8. WAR

## 8.1 WAR là gì?

**WAR — Web Application Archive**.

Đây là packaging format cho web application được deploy vào Servlet container như Tomcat.

Ví dụ:

```text
myapp.war
├── index.jsp
├── static/
├── META-INF/
└── WEB-INF/
    ├── classes/
    ├── lib/
    └── web.xml
```

Các vùng quan trọng:

### `WEB-INF/classes`

Compiled application classes/resources.

### `WEB-INF/lib`

Application dependencies.

Ví dụ:

```text
postgresql-*.jar
logging-library.jar
application-library.jar
```

### `WEB-INF/web.xml`

Deployment descriptor truyền thống.

Không phải modern application nào cũng cần khai báo mọi thứ trong `web.xml`, nhưng bạn vẫn cần hiểu vai trò của nó.

---

# 9. JAR vs WAR

| JAR                                | WAR                                 |
| ---------------------------------- | ----------------------------------- |
| General Java archive               | Web application archive             |
| Có thể chạy trực tiếp              | Thường deploy vào Servlet container |
| `java -jar app.jar`                | Copy/deploy vào Tomcat              |
| Có thể chứa embedded server        | External Tomcat cung cấp container  |
| Lifecycle có thể do app tự quản lý | Lifecycle nằm trong Tomcat          |

Mental model:

```text
Executable JAR

systemd
  |
  v
java -jar app.jar
  |
  v
embedded server
```

versus:

```text
WAR

systemd
  |
  v
Tomcat
  |
  v
myapp.war
```

Không được nhầm:

```bash
java -jar app.war
```

là phương pháp deploy WAR chuẩn.

---

# 10. Tomcat là gì?

Apache Tomcat là **Servlet/JSP container**, không phải JVM.

Quan hệ:

```text
Linux
↓
JVM
↓
Tomcat
↓
Web applications
```

Tomcat bản thân nó cũng là một Java application.

Do đó khi chạy Tomcat:

```bash
ps -ef | grep java
```

bạn sẽ thấy Java process.

Tomcat 11 yêu cầu Java 17 trở lên. Tài liệu Tomcat hiện tại cũng phân biệt rõ Tomcat 11 với Tomcat 10.1 về mức Java tối thiểu. [Apache Tomcat](https://tomcat.apache.org/migration-11.0.html?utm_source=chatgpt.com)

---

# 11. CATALINA_HOME và CATALINA_BASE

Đây là kiến thức rất quan trọng.

## `CATALINA_HOME`

Nơi chứa **Tomcat binaries/static installation**.

Ví dụ:

```text
/opt/tomcat
```

Có thể chứa:

```text
bin/
lib/
```

## `CATALINA_BASE`

Nơi chứa configuration/runtime data của một Tomcat instance.

Ví dụ:

```text
/var/lib/tomcat-prod
```

Tomcat documentation hiện tại khuyến khích hiểu sự tách biệt này; `CATALINA_BASE` chứa instance-specific configuration, logs, deployed applications, `work`, `temp`, v.v. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/introduction.html?utm_source=chatgpt.com)

Concept:

```text
CATALINA_HOME
Tomcat binaries
      |
      +--------------------+
      |                    |
      v                    v
CATALINA_BASE A       CATALINA_BASE B
instance A            instance B
```

Điều này cho phép nhiều Tomcat instances chia sẻ installation.

---

# 12. Tomcat layout

Một layout điển hình:

```text
$CATALINA_BASE/
├── bin/
├── conf/
├── lib/
├── logs/
├── temp/
├── webapps/
└── work/
```

Không phải mọi distro package Tomcat đều dùng chính xác filesystem path giống tarball installation, nhưng chức năng tương đương.

---

## 12.1 `bin/`

Startup/runtime scripts.

Ví dụ:

```text
catalina.sh
startup.sh
shutdown.sh
```

Có thể kiểm tra:

```bash
ls -l "$CATALINA_HOME/bin"
```

---

## 12.2 `conf/`

Tomcat configuration.

Các file quan trọng:

```text
server.xml
web.xml
context.xml
logging.properties
```

### `server.xml`

Server/container/network configuration.

Ví dụ connector:

```xml
<Connector port="8080"
           protocol="HTTP/1.1"
           connectionTimeout="20000" />
```

Ý nghĩa:

```text
Tomcat
   |
   └── listen TCP 8080
```

---

## 12.3 `webapps/`

Default application deployment directory.

Ví dụ:

```text
webapps/
├── ROOT/
├── manager/
└── orders.war
```

Tomcat gọi đây là `appBase`; mặc định thường là:

```text
$CATALINA_BASE/webapps
```

Tomcat có thể deploy WAR hoặc exploded directory ở đây. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/deployer-howto.html?utm_source=chatgpt.com)

---

## 12.4 `logs/`

Container log files.

Có thể thấy các file như:

```text
catalina.YYYY-MM-DD.log
localhost.YYYY-MM-DD.log
localhost_access_log.YYYY-MM-DD.txt
```

Tùy configuration và cách package/start Tomcat.

Không được giả định:

```text
catalina.out
```

luôn là nguồn log duy nhất.

Nếu Tomcat được chạy trực tiếp dưới systemd foreground, stdout/stderr cũng có thể được journald thu thập.

---

## 12.5 `temp/`

Temporary files.

Không dùng làm permanent application storage.

---

## 12.6 `work/`

Tomcat-generated working files.

Ví dụ JSP compilation/cache/runtime artifacts.

Tomcat documentation xác định `work` và `temp` là instance-specific runtime directories. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/introduction.html?utm_source=chatgpt.com)

---

# 13. Context

Trong Tomcat:

> Một Context đại diện cho một web application.

Ví dụ:

```text
orders.war
```

thường tương ứng:

```text
/orders
```

Request:

```text
http://host:8080/orders/api/products
```

Tomcat chọn application dựa trên context path matching. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/config/context.html?utm_source=chatgpt.com)

---

## 13.1 WAR filename và context path

Ví dụ:

```text
webapps/orders.war
```

→ thông thường:

```text
/orders
```

Tomcat deployment documentation xác nhận nếu không chỉ định context path khác thì tên WAR/directory thường được dùng làm context. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/html-manager-howto.html?utm_source=chatgpt.com)

Đặc biệt:

```text
ROOT.war
```

đại diện root context:

```text
/
```

Ví dụ:

```text
http://server:8080/
```

thay vì:

```text
http://server:8080/myapp/
```

---

# 14. Exploded WAR

Tomcat có thể unpack:

```text
orders.war
```

thành:

```text
orders/
```

Tomcat option:

```xml
unpackWARs="true"
```

Hiện tại mặc định standard Host configuration là `true`. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/config/host.html?utm_source=chatgpt.com)

Concept:

```text
orders.war
    |
 unpack
    v
orders/
├── WEB-INF/
├── META-INF/
...
```

---

# 15. Deployment flow

Đây là workflow bạn cần nhớ.

```text
Build
 ↓
Artifact
 ↓
Transfer
 ↓
Pre-check
 ↓
Backup current version
 ↓
Deploy
 ↓
Restart/redeploy if required
 ↓
Process validation
 ↓
Port validation
 ↓
Application validation
 ↓
Log validation
 ↓
Dependency validation
 ↓
Success / Rollback
```

Ví dụ artifact:

```text
orders-1.4.2.war
```

Không nên deploy kiểu:

```bash
cp orders-1.4.2.war webapps/
```

rồi tuyên bố success.

Deployment mới chỉ là **implementation step**.

Success phải được chứng minh.

---

# 16. Deployment state

Sau copy:

```bash
cp orders.war "$CATALINA_BASE/webapps/"
```

Tomcat có thể:

```text
detect artifact
↓
deploy context
↓
load classes
↓
initialize application
↓
create DB connections
↓
start serving requests
```

Bất kỳ bước nào cũng có thể fail.

Ví dụ:

```text
WAR copied successfully
```

không đồng nghĩa:

```text
Application healthy
```

---

# 17. Deployment failure examples

Có thể gặp:

### Broken WAR

```text
ZipException
```

### Java mismatch

```text
UnsupportedClassVersionError
```

### Missing dependency

```text
ClassNotFoundException
```

hoặc:

```text
NoClassDefFoundError
```

### Configuration missing

```text
Could not resolve placeholder ...
```

### DB failure

```text
Connection refused
```

### Port conflict

```text
Address already in use
```

### Permission problem

```text
Permission denied
```

Một administrator cần đọc **first relevant/root exception**, không chỉ dòng cuối stack trace.

---

# 18. Supplementary — Jakarta compatibility

Đây là vấn đề cực kỳ phổ biến khi deploy WAR hiện đại.

Tomcat 9 và trước chủ yếu gắn với namespace:

```java
javax.servlet.*
```

Tomcat 10+ chuyển sang:

```java
jakarta.servlet.*
```

Apache ghi rõ đây là breaking change khi chuyển từ Tomcat 9 sang Tomcat 10 và application thường phải được migrate/recompile. [Apache Tomcat](https://tomcat.apache.org/migration-10?utm_source=chatgpt.com)

Ví dụ application cũ:

```java
import javax.servlet.http.HttpServlet;
```

Application mới:

```java
import jakarta.servlet.http.HttpServlet;
```

Nếu deployment gặp class-related error sau khi upgrade Tomcat major version, đây là hypothesis rất đáng kiểm tra.

---

# 19. Service account

## 19.1 Không chạy Tomcat bằng root

Production principle:

```text
root
 ≠
application runtime account
```

Tạo dedicated identity:

```text
tomcat
```

Concept:

```text
User: tomcat
Group: tomcat
```

Service account chỉ cần quyền đủ để:

```text
read application
read configuration
read required certificates
write logs/runtime directories
bind permitted application ports
connect outbound to dependencies
```

Không nên có quyền:

```text
sudo unrestricted
write /etc
write /usr/bin
write arbitrary application binaries
interactive administration access
```

---

# 20. Least privilege

Ví dụ:

```bash
id tomcat
```

Output conceptually:

```text
uid=... tomcat gid=... tomcat
```

Directory permissions nên tách:

```text
application binaries → read-only at runtime
configuration         → read-only
secrets               → restricted
logs                   → writable
temp/work              → writable
```

Không dùng:

```bash
chmod -R 777 /opt/tomcat
```

để "sửa permission".

Đây là anti-pattern rất nghiêm trọng.

---

# 21. systemd integration

Tomcat/application process không nên được administrator chạy thủ công như:

```bash
nohup java ... &
```

rồi bỏ đó.

systemd cung cấp:

```text
start/stop
boot startup
identity
environment
restart policy
logging integration
state tracking
dependency ordering
```

Mental model:

```text
systemd
   |
   └── service unit
           |
           └── Java/Tomcat process
```

---

# 22. systemd service example

Ví dụ **minh họa**, path phải được điều chỉnh theo distro/install:

```ini
[Unit]
Description=Apache Tomcat application server
After=network.target

[Service]
Type=simple

User=tomcat
Group=tomcat

Environment="JAVA_HOME=/usr/lib/jvm/java-25-openjdk"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/var/lib/tomcat-prod"

EnvironmentFile=-/etc/tomcat/tomcat.env

ExecStart=/opt/tomcat/bin/catalina.sh run

Restart=on-failure
RestartSec=5

UMask=0027

[Install]
WantedBy=multi-user.target
```

Quan trọng nhất không phải học thuộc file này mà hiểu từng field.

---

## 22.1 `User=`

```ini
User=tomcat
```

Process chạy dưới account:

```text
tomcat
```

Thực thi:

```bash
ps -o user,pid,cmd -C java
```

để xác nhận.

---

## 22.2 `Group=`

Primary runtime group:

```ini
Group=tomcat
```

---

## 22.3 `Environment=`

Inject environment variable:

```ini
Environment="JAVA_HOME=/usr/lib/jvm/java-25-openjdk"
```

---

## 22.4 `EnvironmentFile=`

Ví dụ:

```ini
EnvironmentFile=-/etc/tomcat/tomcat.env
```

Dấu:

```text
-
```

trước path có ý nghĩa là thiếu file không nhất thiết khiến unit fail ngay tại bước load.

Ví dụ file:

```bash
JAVA_OPTS=-Xms512m -Xmx1024m
APP_PROFILE=prod
```

Nhưng lưu ý quan trọng:

> EnvironmentFile là mechanism truyền configuration; **không tự động biến file đó thành secret vault an toàn**.

---

## 22.5 `ExecStart`

Ví dụ:

```ini
ExecStart=/opt/tomcat/bin/catalina.sh run
```

Với systemd, foreground execution thường dễ supervise hơn daemonize-double-fork tùy application.

---

## 22.6 `Restart=on-failure`

Nếu process crash:

```text
unexpected failure
↓
systemd
↓
restart
```

Nhưng restart policy không phải thuốc chữa mọi lỗi.

Ví dụ:

```text
bad config
↓
process starts
↓
crashes
↓
systemd restarts
↓
crashes
↓
restart loop
```

Khi đó phải sửa root cause.

---

# 23. systemctl lifecycle

Sau khi sửa unit:

```bash
sudo systemctl daemon-reload
```

Enable startup:

```bash
sudo systemctl enable tomcat
```

Start:

```bash
sudo systemctl start tomcat
```

Status:

```bash
systemctl status tomcat
```

Restart:

```bash
sudo systemctl restart tomcat
```

Stop:

```bash
sudo systemctl stop tomcat
```

Không nhầm:

```bash
daemon-reload
```

với:

```bash
restart tomcat
```

`daemon-reload` làm systemd reload unit definitions; nó không đồng nghĩa restart application.

---

# PHẦN II — DAY 13

# Externalized Configuration

## 24. Vấn đề của hard-coded configuration

Bad design:

```java
String dbHost = "10.0.1.17";
String password = "ProdPassword123";
```

Hoặc:

```text
application.war
├── source/config
│    └── production-password
```

Hậu quả:

```text
code == configuration
```

Muốn đổi environment lại phải rebuild artifact.

Mục tiêu tốt hơn:

```text
same artifact
+
different external configuration
=
different environment
```

Ví dụ:

```text
orders.war
     +
DEV configuration
```

hoặc:

```text
orders.war
     +
PROD configuration
```

---

# 25. Externalized configuration

Configuration có thể đến từ:

```text
environment variables
external properties/YAML files
system properties -D...
command-line arguments
secret stores
platform configuration
database/config service
```

Không có một universal precedence rule cho tất cả Java applications.

**Precedence phụ thuộc framework/application.**

Đây là điểm rất quan trọng.

Không được nói:

> Environment variables luôn override configuration file trong Java.

Java runtime không tự định nghĩa quy tắc business-config đó.

Framework/application mới quyết định.

---

# 26. Environment variables

Linux:

```bash
export APP_ENV=prod
export DB_HOST=db.internal
export DB_PORT=5432
```

Check:

```bash
printenv APP_ENV
```

Trong Java:

```java
String environment = System.getenv("APP_ENV");
```

Ví dụ:

```java
String dbHost = System.getenv("DB_HOST");
```

Nếu biến không tồn tại:

```java
System.getenv("DB_HOST")
```

có thể trả:

```text
null
```

nên application cần validation/error handling phù hợp.

---

# 27. Java system properties

Ngoài environment variable, Java có system properties.

Run:

```bash
java -Dapp.environment=prod \
     -Dapp.region=ap-southeast-1 \
     -jar app.jar
```

Java:

```java
String env = System.getProperty("app.environment");
```

Phân biệt:

```text
System.getenv(...)
```

vs:

```text
System.getProperty(...)
```

Không phải một thứ.

---

# 28. Profiles

Profile là cách group configuration theo environment/mode.

Ví dụ:

```text
development
test
staging
production
```

Không có Java SE standard bắt buộc biến profile phải tên gì.

Application có thể thiết kế:

```bash
APP_PROFILE=prod
```

và code:

```java
String profile = System.getenv("APP_PROFILE");
```

---

## Supplementary — Spring profile

Nếu application dùng Spring Boot:

```bash
SPRING_PROFILES_ACTIVE=prod
```

hoặc:

```text
application.properties
application-prod.properties
```

Đây là **Spring-specific**, không phải Java standard.

---

# 29. Config separation

Một model tương đối tốt:

```text
Artifact
orders.war
  |
  +--------------------------------+
                                   |
                        Environment-specific config
                                   |
                  +----------------+----------------+
                  |                |                |
                 DEV              UAT              PROD
```

Application binary nên ideally giống nhau.

Không nên:

```text
orders-dev.war
orders-uat-fixed.war
orders-prod-final-final2.war
```

nếu khác biệt chủ yếu chỉ là environment configuration.

---

# 30. Ports

Port là Layer 4 endpoint identifier.

Ví dụ:

```text
Tomcat HTTP: 8080
PostgreSQL:   5432
HTTPS:         443
```

Java/Tomcat application:

```text
IP address + port
```

tạo listening socket.

Kiểm tra:

```bash
ss -lntp
```

Ví dụ:

```text
LISTEN ... 0.0.0.0:8080 ...
```

hoặc:

```text
127.0.0.1:8080
```

Hai cái này khác nhau đáng kể.

---

# 31. Bind address

```text
127.0.0.1:8080
```

chỉ local interface.

Trong khi:

```text
0.0.0.0:8080
```

nghĩa là IPv4 wildcard, listen trên các applicable interfaces.

Không được thấy:

```text
port=8080
```

rồi kết luận remote client chắc chắn truy cập được.

Còn phụ thuộc:

```text
bind address
firewall
AWS Security Group
route
reverse proxy
application state
```

---

# 32. Port conflict

Hai processes thường không thể cùng bind cùng address/port combination.

Symptom:

```text
Address already in use
```

Check:

```bash
ss -lntp | grep ':8080'
```

hoặc:

```bash
sudo lsof -iTCP:8080 -sTCP:LISTEN
```

Không nên phản ứng bằng cách:

```bash
kill -9 <pid>
```

ngay.

Phải xác định process đó là gì và có được phép stop hay không.

---

# 33. Tomcat connector port

Ví dụ:

```xml
<Connector port="8080"
           protocol="HTTP/1.1"
           connectionTimeout="20000" />
```

Flow:

```text
TCP :8080
 ↓
Tomcat Connector
 ↓
Request processing
 ↓
Context
 ↓
Servlet/application
```

Sau sửa `server.xml`, thường cần restart Tomcat để thay đổi có hiệu lực.

Trước restart phải:

```text
inspect
backup
validate config where possible
plan rollback
```

---

# 34. Context path

Giả sử:

```text
orders.war
```

Context:

```text
/orders
```

Correct URL:

```text
http://localhost:8080/orders/
```

Nếu kiểm tra:

```bash
curl http://localhost:8080/
```

và nhận 404 thì chưa chắc application hỏng.

Bạn có thể đơn giản đang gọi **sai context path**.

Đây là lỗi troubleshooting rất thường gặp.

---

# 35. HTTP response interpretation

Ví dụ:

```bash
curl -i http://localhost:8080/orders/
```

Có thể trả:

```text
HTTP/1.1 200
```

Application responding.

```text
HTTP/1.1 404
```

Server accessible nhưng resource/context có thể không đúng.

```text
HTTP/1.1 401
```

Resource tồn tại nhưng authentication cần thiết.

```text
HTTP/1.1 403
```

Server hiểu request nhưng deny authorization/access.

```text
HTTP/1.1 500
```

Server-side application error.

```text
Connection refused
```

không phải HTTP status.

Nó thường nghĩa TCP connection chưa được establish tới listening service.

---

# 36. JVM memory model

Java process memory không chỉ có Heap.

Mental model:

```text
Java Process RSS
│
├── Java Heap
│
├── Metaspace
├── Thread stacks
├── Direct/native buffers
├── JVM native structures
├── Code cache
├── GC structures
├── JNI/native libraries
└── other native memory
```

Do đó:

> `-Xmx4g` không có nghĩa Java process tối đa chỉ dùng 4 GB RAM.

`-Xmx` chỉ giới hạn Java heap maximum.

---

# 37. Heap

Heap là vùng chứa phần lớn Java objects.

Ví dụ:

```java
Order order = new Order();
```

Object thường nằm trong heap.

Heap được Garbage Collector quản lý.

---

# 38. `-Xms`

Initial heap size.

Ví dụ:

```bash
-Xms512m
```

Concept:

```text
initial heap target ≈ 512 MB
```

---

# 39. `-Xmx`

Maximum heap size.

Ví dụ:

```bash
-Xmx2g
```

Concept:

```text
Java heap cannot grow beyond configured maximum
```

Oracle Java documentation định nghĩa `-Xmx` là maximum heap size; `-Xms` cấu hình initial heap. [Oracle Docs](https://docs.oracle.com/en/java/javase/17/docs/specs/man/java.html?utm_source=chatgpt.com)

Ví dụ:

```bash
java -Xms512m -Xmx2g -jar app.jar
```

---

# 40. Không đặt Xmx = toàn bộ RAM

Giả sử server:

```text
RAM = 4 GB
```

Không nên đơn giản:

```text
-Xmx4g
```

vì OS và JVM vẫn cần memory ngoài heap:

```text
Linux
Tomcat
Metaspace
threads
direct buffers
native libraries
filesystem cache
monitoring agents
```

Nếu tổng memory demand vượt available RAM:

```text
memory pressure
↓
swap / severe slowdown
↓
OOM killer possible
↓
service outage
```

---

# 41. Java OutOfMemoryError

Ví dụ:

```text
java.lang.OutOfMemoryError: Java heap space
```

Có thể liên quan:

```text
heap too small
memory leak
unexpected load
large request/data
cache growth
```

Không nên phản xạ:

```text
increase Xmx
```

mà không điều tra.

Correct thinking:

```text
Symptom
↓
memory evidence
↓
historical trend
↓
heap setting
↓
load/change
↓
GC/application evidence
↓
root cause
```

---

# 42. JVM option placement

Ví dụ systemd env:

```ini
Environment="JAVA_OPTS=-Xms512m -Xmx1024m"
```

Nhưng cách Tomcat consumes options phụ thuộc startup scripts/package.

Tomcat installations thường có cơ chế `JAVA_OPTS`/`CATALINA_OPTS`; không nên sửa trực tiếp vendor startup script nếu có supported override mechanism.

Ví dụ common tarball approach:

```text
$CATALINA_BASE/bin/setenv.sh
```

Ví dụ:

```bash
CATALINA_OPTS="-Xms512m -Xmx1024m"
```

`CATALINA_OPTS` thường phù hợp hơn cho Tomcat-runtime-specific options vì chúng được dùng khi Tomcat start, trong khi `JAVA_OPTS` có phạm vi rộng hơn trong scripts.

Điểm exam quan trọng:

> Biết **option thực sự được truyền vào JVM hay chưa**.

Có thể kiểm tra process:

```bash
ps -ef | grep '[j]ava'
```

hoặc JDK tools phù hợp:

```bash
jcmd <PID> VM.command_line
```

---

# 43. Logs

Logs là evidence, không chỉ là text để đọc sau sự cố.

Bạn cần phân biệt ít nhất:

```text
OS/service logs
Tomcat/container logs
application logs
HTTP access logs
database logs
```

---

# 44. systemd journal

Ví dụ:

```bash
journalctl -u tomcat
```

Recent:

```bash
journalctl -u tomcat -n 100
```

Follow:

```bash
journalctl -u tomcat -f
```

Current boot:

```bash
journalctl -u tomcat -b
```

Time window:

```bash
journalctl -u tomcat --since "10 minutes ago"
```

---

# 45. Application log

Application có thể có riêng:

```text
/var/log/myapp/application.log
```

Ví dụ:

```bash
tail -f /var/log/myapp/application.log
```

Search:

```bash
grep -i 'error' application.log
```

Context:

```bash
grep -n -C 5 'SQLException' application.log
```

Không chỉ grep:

```text
ERROR
```

vì root cause có thể được ghi:

```text
Caused by:
```

Ví dụ:

```bash
grep -n 'Caused by' application.log
```

---

# 46. Log level

Các level phổ biến:

```text
TRACE
DEBUG
INFO
WARN
ERROR
```

Không phải mọi logging framework đều vận hành giống nhau.

Production thường không bật DEBUG vô hạn vì:

```text
disk growth
performance overhead
possible sensitive data exposure
noise
```

Debug level chỉ nên tăng có mục đích, có thời gian kết thúc và rollback.

---

# 47. Disk và logs

Application có thể vẫn chạy nhưng log filesystem đầy.

Check:

```bash
df -h
```

Inode:

```bash
df -i
```

Log files:

```bash
du -sh /var/log/*
```

Possible chain:

```text
verbose logging
↓
disk fills
↓
application cannot write logs/temp
↓
unexpected failures
↓
service degradation
```

---

# 48. Safe restart

`restart` không chỉ là command.

Operationally:

```text
Pre-check
↓
Confirm scope
↓
Confirm change approval
↓
Check dependencies
↓
Check active condition / traffic impact
↓
Perform controlled restart
↓
Verify process
↓
Verify port
↓
Verify application
↓
Verify dependencies
↓
Review logs
```

---

# 49. Restart vs reload

Không được dùng hai khái niệm như nhau.

### Restart

```text
process stop
↓
process start
```

Có thể gây:

```text
connection interruption
session impact
temporary downtime
cache warm-up
connection pool recreation
```

### Reload

Nếu application/service hỗ trợ:

```text
re-read config
```

mà không full process restart.

Nhưng **không phải mọi Java/Tomcat setting reload được**.

Nếu runbook yêu cầu restart, không tự suy đoán reload đủ.

---

# 50. Safe restart command

Ví dụ:

```bash
sudo systemctl restart tomcat
```

Sau đó **không dừng tại exit code 0**.

Verify:

```bash
systemctl is-active tomcat
```

```bash
systemctl status tomcat
```

```bash
ss -lntp | grep ':8080'
```

```bash
curl -fsS http://127.0.0.1:8080/orders/health
```

và:

```bash
journalctl -u tomcat --since "5 minutes ago"
```

---

# 51. `systemctl restart` thành công không có nghĩa app healthy

Ví dụ:

```text
systemctl restart
exit code 0
```

systemd chỉ có thể biết process đã được start.

Application có thể sau vài giây:

```text
fail DB initialization
return HTTP 500
have broken business function
```

Do đó validation phải xuyên layer.

---

# PHẦN III — DAY 14

# Database Connectivity

## 52. Application-to-database flow

Java application thường sử dụng JDBC.

Architecture:

```text
Java application
      |
      | JDBC API
      |
      v
PostgreSQL JDBC Driver
      |
      | PostgreSQL wire protocol
      |
      v
PostgreSQL :5432
```

pgJDBC là PostgreSQL JDBC driver cho Java và giao tiếp trực tiếp bằng PostgreSQL native protocol. [pgJDBC](https://jdbc.postgresql.org/documentation/head/?utm_source=chatgpt.com)

---

# 53. JDBC URL

Ví dụ:

```text
jdbc:postgresql://db.internal.example:5432/appdb
```

Breakdown:

```text
jdbc:
   postgresql:
        //
        db.internal.example
        :
        5432
        /
        appdb
```

Meaning:

| Component       | Value                 |
| --------------- | --------------------- |
| Driver/protocol | PostgreSQL            |
| Host            | `db.internal.example` |
| Port            | `5432`                |
| Database        | `appdb`               |

Credentials thường đi riêng.

---

# 54. Java JDBC example

Ví dụ đơn giản để hiểu kiến trúc:

```java
import java.sql.Connection;
import java.sql.DriverManager;

public class DbTest {
    public static void main(String[] args) throws Exception {
        String url =
            "jdbc:postgresql://db.internal.example:5432/appdb";

        String user = System.getenv("DB_USER");
        String password = System.getenv("DB_PASSWORD");

        try (Connection conn =
                 DriverManager.getConnection(url, user, password)) {
            System.out.println("Connected");
        }
    }
}
```

Đây chỉ là minh họa.

Production applications thường dùng datasource/connection pool thay vì mở connection kiểu đơn giản cho mỗi request.

---

# 55. Database connectivity prerequisites

Một successful DB connection cần tất cả:

```text
1. DNS resolution
2. route/network path
3. firewall/security rules
4. DB process listening
5. correct port
6. PostgreSQL access policy
7. valid database
8. valid role/user
9. valid authentication
10. sufficient permissions
11. TLS compatibility if required
```

Connection error chỉ nói cho bạn biết **một stage nào đó fail**.

---

# 56. Connection refused

Ví dụ:

```text
Connection refused
```

Có thể gợi ý:

```text
host reachable
but no accepting socket at target endpoint
```

Hypotheses:

```text
wrong port
PostgreSQL stopped
PostgreSQL listening only localhost
firewall active reject
wrong host
```

Không đồng nghĩa:

```text
password wrong
```

---

# 57. Connection timeout

Ví dụ:

```text
Connection timed out
```

Có thể liên quan:

```text
network route
firewall silently dropping
security group
unreachable host
network ACL
```

Không giống:

```text
authentication failed
```

---

# 58. Authentication error

Ví dụ:

```text
password authentication failed for user ...
```

Đây là evidence rằng bạn đã đi xa hơn TCP layer:

```text
network reached DB
↓
PostgreSQL responded
↓
authentication failed
```

Do đó không nên bắt đầu troubleshooting bằng Security Group nữa nếu evidence đã chứng minh DB trả authentication error.

---

# 59. Permission error

Có thể authentication successful nhưng query fail:

```text
permission denied for table orders
```

Architecture:

```text
connect successfully
       |
       v
authenticated
       |
       v
authorization fails
```

Đây là khác biệt:

```text
Authentication = Who are you?
Authorization  = What may you do?
```

---

# 60. Connection pools — Supplementary

Production web apps thường giữ một pool:

```text
Application
     |
     v
Connection Pool
├── DB connection 1
├── DB connection 2
├── DB connection 3
└── DB connection N
```

Thay vì:

```text
request
↓
open connection
↓
query
↓
close
```

mỗi lần.

Advantages:

```text
lower connection overhead
controlled concurrency
reuse connections
```

Nhưng pool cũng tạo failure modes:

```text
pool exhausted
stale connections
DB connection limit exhausted
leaks
timeout mismatch
```

Nội dung sâu hơn sẽ thuộc Module 2 / Day 15–16.

---

# 61. Secrets handling

Secret examples:

```text
database password
API token
private key
keystore password
cloud credentials
```

Không phải:

```text
DB_HOST
APP_PORT
ENVIRONMENT_NAME
```

vì chúng thường chỉ là configuration, không phải secret.

---

# 62. Không hard-code secrets

Sai:

```java
String password = "P@ssw0rd123";
```

Sai:

```properties
db.password=P@ssw0rd123
```

nếu file này được commit vào Git.

Sai:

```bash
git add application-prod.properties
```

khi file chứa production password.

---

# 63. Secret exposure surfaces

Secret có thể bị leak qua:

```text
Git
CI/CD logs
shell history
process command line
environment dumps
log files
support bundles
screenshots
tickets
backups
world-readable config
```

Đó là lý do không nên truyền password kiểu:

```bash
java -Ddb.password=SuperSecret ...
```

nếu process command-line có thể bị đọc bởi người/process không nên thấy nó.

---

# 64. Environment variable có phải secret vault không?

Không.

Environment variable giúp **externalize** secret:

```bash
DB_PASSWORD=...
```

nhưng không tự giải quyết secret lifecycle.

Bạn vẫn phải xử lý:

```text
who can read it?
where is it stored?
who can change it?
how is it rotated?
is it logged?
is it backed up?
```

---

# 65. Better secret model

Concept:

```text
Secret Manager / protected source
            |
            v
       application
```

hoặc least-privilege protected file:

```text
/etc/myapp/secret
owner: app
mode: 600
```

tùy platform/runbook.

Không bao giờ yêu cầu người dùng paste production secret vào troubleshooting ticket/chat.

---

# 66. TLS fundamentals

TLS có ba mục tiêu quan trọng:

```text
Confidentiality
Integrity
Authentication
```

### Confidentiality

Người ngoài không dễ đọc traffic.

### Integrity

Traffic bị thay đổi trên đường có thể bị phát hiện.

### Authentication

Client có thể xác minh server identity thông qua certificate chain/hostname validation.

---

# 67. Certificate basics

Một server certificate về cơ bản liên kết:

```text
Identity / hostname
        +
Public key
        +
CA signature
```

TLS server có:

```text
private key
certificate
certificate chain
```

Client có trust anchors:

```text
trusted CA certificates
```

---

# 68. Certificate chain

Simplified:

```text
Root CA
   |
 signs
   v
Intermediate CA
   |
 signs
   v
Server Certificate
```

Client phải có khả năng xây dựng:

```text
server cert
↓
intermediate
↓
trusted root
```

Nếu chain không hợp lệ:

```text
certificate validation failure
```

---

# 69. Hostname verification

Giả sử certificate hợp lệ cho:

```text
db.example.com
```

nhưng application connect:

```text
10.0.1.10
```

Có thể fail hostname verification vì identity không khớp.

Do đó:

> Certificate trusted chưa đủ; hostname verification cũng quan trọng.

---

# 70. Expired certificate

Certificate có validity interval.

Nếu expired:

```text
current time > Not After
```

TLS validation có thể fail.

Do đó certificate management là operational lifecycle:

```text
issue
deploy
monitor expiry
renew
deploy new cert
verify
retire old cert
```

---

# 71. Java truststore

Java có truststore chứa trusted CA certificates.

Oracle JDK documentation hiện tại đặt default `cacerts` dưới:

```text
$JAVA_HOME/lib/security/cacerts
```

và `keytool` dùng để inspect/import certificate entries. [Oracle Docs](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html?utm_source=chatgpt.com)

List:

```bash
keytool -list -cacerts
```

Inspect certificate:

```bash
keytool -printcert -file server.crt
```

---

# 72. Custom truststore

Ví dụ:

```bash
keytool \
  -importcert \
  -alias internal-ca \
  -file internal-ca.crt \
  -keystore app-truststore.p12
```

Không nhập certificate một cách mù quáng.

Trước khi trust phải xác minh:

```text
issuer
subject
fingerprint
validity
source authenticity
```

Oracle cũng cảnh báo phải xác minh certificate/fingerprint trước khi thêm trust. [Oracle Docs](https://docs.oracle.com/en/java/javase/25/docs/specs/man/keytool.html?utm_source=chatgpt.com)

---

# 73. PostgreSQL over TLS

Ví dụ JDBC:

```text
jdbc:postgresql://db.example.com:5432/appdb?sslmode=verify-full
```

`verify-full`:

```text
encrypt connection
+
validate certificate chain
+
verify hostname
```

pgJDBC documentation khuyến nghị `verify-full` cho security-sensitive environments. [pgJDBC](https://jdbc.postgresql.org/documentation/ssl/?utm_source=chatgpt.com)

---

# 74. `sslmode=require` vs `verify-full`

Đây là điểm rất hay bị nhầm.

Simplified:

### `require`

```text
encrypted?
yes
```

nhưng theo pgJDBC mode này không cung cấp đầy đủ identity validation như `verify-full`. [pgJDBC](https://jdbc.postgresql.org/documentation/ssl/?utm_source=chatgpt.com)

### `verify-ca`

```text
encrypted
+
certificate chain trusted
```

### `verify-full`

```text
encrypted
+
certificate chain trusted
+
hostname verified
```

Security mindset:

```text
Encryption without identity verification
≠
full protection against MITM
```

---

# 75. TLS troubleshooting

Possible errors:

```text
certificate expired
unknown CA
hostname mismatch
missing intermediate certificate
wrong truststore
wrong truststore password
unsupported TLS/cipher configuration
```

Useful diagnostic—not something to leave permanently enabled:

```bash
java -Djavax.net.debug=ssl ...
```

pgJDBC also documents this as a way to get SSL debugging information. [pgJDBC](https://jdbc.postgresql.org/documentation/ssl/?utm_source=chatgpt.com)

Caution: TLS debug output can be extremely verbose and may expose operational detail.

---

# 76. Reverse proxy awareness

A reverse proxy đứng **trước application server**.

Architecture:

```text
Internet
   |
 HTTPS :443
   |
   v
Reverse Proxy
   |
 HTTP :8080
   |
   v
Tomcat
```

Typical products:

```text
Nginx
Apache HTTP Server
cloud load balancer / proxy
```

---

# 77. Forward proxy vs reverse proxy

### Forward proxy

Represent client:

```text
Client
  |
  v
Proxy
  |
  v
Internet
```

### Reverse proxy

Represent server/backend:

```text
Internet client
      |
      v
Reverse Proxy
      |
      v
Backend server
```

---

# 78. Why reverse proxy?

Có thể cung cấp:

```text
TLS termination
public port 443
routing
load balancing
request filtering
header handling
static content
access logging
backend isolation
```

Tomcat có thể chỉ bind:

```text
127.0.0.1:8080
```

trong khi Nginx listen:

```text
0.0.0.0:443
```

Điều này giảm việc expose application port trực tiếp.

---

# 79. Nginx reverse proxy example

Ví dụ tối giản:

```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Nginx documentation xác định `proxy_pass` để forward request tới upstream server và `proxy_set_header` để kiểm soát headers chuyển tiếp. [Nginx](https://nginx.org/en/docs/http/ngx_http_proxy_module.html?utm_source=chatgpt.com)

Architecture:

```text
Client
  |
  | GET /
  v
Nginx :443
  |
  | proxied HTTP
  v
Tomcat :8080
```

---

# 80. Header awareness

Reverse proxy thường cần truyền metadata về original request.

Ví dụ:

```text
Host
X-Forwarded-For
X-Forwarded-Proto
```

Application có thể cần biết:

```text
original scheme = HTTPS
original client IP
original host
```

Nếu proxy/header handling sai có thể sinh:

```text
wrong redirect URL
HTTP instead of HTTPS redirect
incorrect client IP logs
authentication callback issues
```

Configuration chính xác phụ thuộc proxy/framework.

---

# 81. Backend port should not necessarily be public

Nếu architecture:

```text
Internet
  |
 :443
  |
Proxy
  |
 :8080
  |
Tomcat
```

thì thường không cần expose:

```text
Tomcat :8080
```

trực tiếp ra Internet.

Defense-in-depth:

```text
External access
     |
     v
reverse proxy only
     |
     v
internal backend
```

---

# 82. Deployment validation

Đây là phần cực kỳ quan trọng của Day 14.

Deployment validation phải trả lời:

> Artifact đã thực sự trở thành một service usable chưa?

Không chỉ:

```text
cp succeeded
```

---

# 83. Validation layers

Một sequence rất tốt:

```text
1. Artifact
2. Service
3. Process
4. Socket
5. HTTP
6. Application
7. Dependency
8. Logs
9. Business smoke test
10. Monitoring
```

---

# 84. Artifact validation

Ví dụ:

```bash
ls -lh /path/to/orders.war
```

Hash nếu release process cung cấp:

```bash
sha256sum orders.war
```

Mục tiêu:

```text
right artifact
right version
not truncated/corrupted
```

---

# 85. Service validation

```bash
systemctl status tomcat
```

Expected invariant:

```text
service active
```

Nhưng chưa đủ.

---

# 86. Process validation

```bash
pgrep -a java
```

Check:

```text
correct runtime user?
correct Java?
correct JVM options?
correct Tomcat instance?
```

---

# 87. Socket validation

```bash
ss -lntp | grep ':8080'
```

Expected:

```text
correct listener exists
```

Nếu service active nhưng port không listen, phải investigate.

---

# 88. HTTP validation

```bash
curl -i http://127.0.0.1:8080/orders/
```

Hoặc nếu health endpoint:

```bash
curl -fsS http://127.0.0.1:8080/orders/health
```

Không nên chỉ grep process name.

---

# 89. Reverse proxy validation

```bash
curl -I https://app.example.com/
```

So sánh:

```text
direct backend
```

với:

```text
public proxy path
```

Ví dụ:

```bash
curl http://127.0.0.1:8080/orders/
```

works nhưng:

```bash
curl https://app.example.com/orders/
```

fails.

Điều này scope issue về phía:

```text
reverse proxy/TLS/routing/header layer
```

thay vì Tomcat itself.

---

# 90. Database dependency validation

App có thể trả:

```text
200 /
```

nhưng DB function vẫn hỏng.

Do đó smoke validation cần dependency-sensitive endpoint/function.

Ví dụ conceptual:

```text
GET /health
```

có thể chỉ kiểm tra app process.

Trong khi:

```text
GET /orders/123
```

có thể require DB.

Phải hiểu endpoint nào chứng minh điều gì.

---

# 91. Log validation

Sau release:

```bash
journalctl -u tomcat --since "10 minutes ago"
```

và application log.

Tìm:

```text
ERROR
Exception
Caused by
connection
timeout
permission
certificate
```

Nhưng không nên grep mù; cần timestamp/context.

---

# 92. Deployment success criteria

Ví dụ:

```text
Service          active
Java process     running as correct account
Port             listening
Context          deployed
HTTP endpoint    expected response
Database         connected
Logs             no new critical errors
TLS/public URL   valid
Smoke test       passed
Monitoring       normal
```

Chỉ khi đủ acceptance criteria mới gọi release thành công.

---

# 93. Rollback

Rollback là:

> Quay hệ thống từ trạng thái release mới không đạt acceptance criteria về một known-good state.

Không phải:

> Copy đại file cũ vào rồi hy vọng.

---

# 94. Good rollback prerequisite

Trước deployment phải biết:

```text
current version
new version
old artifact location
config changes
DB changes
rollback command
expected recovery state
```

Ví dụ:

```text
current:
orders-1.4.1.war

candidate:
orders-1.4.2.war
```

---

# 95. Versioned artifacts

Better:

```text
/releases/
├── orders-1.4.0.war
├── orders-1.4.1.war
└── orders-1.4.2.war
```

Not:

```text
orders.war
orders-old.war
orders-old2.war
orders-final-good.war
```

Version identity là một phần evidence.

---

# 96. Simple rollback flow

```text
New release validation fails
        |
        v
Stop / isolate new version safely
        |
        v
Restore previous artifact/config
        |
        v
Restart/redeploy
        |
        v
Validate service
        |
        v
Validate application
        |
        v
Validate DB/dependencies
        |
        v
Preserve incident/release evidence
```

---

# 97. Rollback không chỉ là WAR

Release có thể thay đổi:

```text
WAR
configuration
JVM options
certificate
reverse-proxy config
database schema
permissions
service unit
```

Nếu bạn rollback:

```text
WAR only
```

nhưng giữ configuration incompatible:

```text
old WAR + new config
```

hệ thống vẫn fail.

Do đó phải có:

```text
configuration baseline
```

---

# 98. Database changes complicate rollback

Ví dụ release mới thêm DB schema incompatible với old application.

```text
Application v2
     +
Schema v2
```

Sau đó rollback application:

```text
Application v1
     +
Schema v2
```

có thể không chạy.

Đây là lý do application/database release phải thiết kế compatibility và migration/rollback plan cẩn thận.

Chi tiết DB migration vượt explicit Day 14 syllabus, nhưng operational awareness này rất quan trọng.

---

# 99. Rollback trigger

Rollback không nên phụ thuộc cảm giác.

Ví dụ release criteria:

```text
HTTP 5xx rate above threshold
health check fails
database initialization fails
critical business transaction fails
startup exception
unexpected security issue
```

Khi trigger xảy ra:

```text
stop change
rollback
validate
document
```

trừ khi approved recovery procedure khác quy định.

---

# 100. Troubleshooting framework cho Module 1

Bạn nên thuộc flow:

```text
Symptom
↓
Expected behavior
↓
Recent changes
↓
Scope
↓
Failing layer
↓
Evidence
↓
Hypotheses
↓
Safest test
↓
Root cause
↓
Remediation / escalation
↓
Validation
↓
Evidence preservation
```

---

# 101. Scenario: Tomcat service failed

Check:

```bash
systemctl status tomcat
```

Sau đó:

```bash
journalctl -u tomcat -n 100
```

Hypotheses:

```text
bad Java path
bad permissions
bad server.xml
port conflict
invalid JVM option
application initialization failure
memory problem
```

Không restart 20 lần liên tục.

---

# 102. Scenario: service active nhưng website unavailable

Check layer-by-layer:

```bash
systemctl is-active tomcat
```

Then:

```bash
ss -lntp | grep ':8080'
```

Then:

```bash
curl -v http://127.0.0.1:8080/
```

Then context:

```bash
curl -v http://127.0.0.1:8080/orders/
```

Then proxy:

```bash
curl -v https://app.example.com/orders/
```

This yields scope:

```text
Tomcat?
context?
proxy?
network?
```

---

# 103. Scenario: port not listening

Evidence:

```bash
ss -lntp | grep ':8080'
```

returns nothing.

Then:

```text
application never started
Tomcat startup failed
wrong port
wrong bind address
service crashed
```

Logs next.

---

# 104. Scenario: HTTP 404

Do not restart immediately.

Reasoning:

```text
TCP works
HTTP server works
response produced
```

Potential causes:

```text
wrong context path
WAR not deployed
wrong resource URL
proxy path rewrite mismatch
application routing
```

This is already much narrower than "server down".

---

# 105. Scenario: HTTP 500

Means request reached HTTP application/server layer.

Check logs at exact request timestamp.

Potential:

```text
NullPointerException
SQLException
dependency unavailable
config missing
permission error
downstream timeout
```

Focus moves upward in stack.

---

# 106. Scenario: application cannot connect PostgreSQL

Flow:

```text
App
 |
 +--> resolve DB hostname?
 |
 +--> TCP 5432?
 |
 +--> PostgreSQL responds?
 |
 +--> authentication?
 |
 +--> DB exists?
 |
 +--> authorization?
 |
 +--> TLS validation?
```

Useful commands from host:

```bash
getent hosts db.internal.example
```

```bash
nc -vz db.internal.example 5432
```

nếu `nc` có sẵn.

Hoặc PostgreSQL-aware:

```bash
pg_isready -h db.internal.example -p 5432
```

Credential-sensitive tests phải tuân thủ runbook; không paste password vào command line bừa bãi.

---

# 107. Scenario: certificate error

Symptom:

```text
PKIX path building failed
```

Potential:

```text
unknown CA
missing intermediate
wrong truststore
certificate rotated
```

Symptom:

```text
hostname verification failed
```

Potential:

```text
connect using wrong DNS name/IP
wrong certificate SAN
```

Symptom:

```text
certificate expired
```

Potential root cause obvious hơn nhưng vẫn cần validate certificate chain/time.

---

# 108. Scenario: deployment mới fail ngay

Recent change:

```text
WAR v1.4.2 deployed
```

Comparison:

```text
before = healthy
after = unhealthy
```

Strong hypothesis:

```text
release-related
```

Check:

```text
artifact version
config delta
Java/Tomcat compatibility
logs
DB changes
certificate/config changes
```

Nếu nằm trong approved rollback criteria:

```text
rollback known-good version
```

thường an toàn hơn "debug production indefinitely".

---

# 109. L1 remediation vs escalation

L1/fresher operator thường có thể thực hiện **approved, reversible actions**:

```text
inspect status
inspect logs
check process
check port
check disk
verify config presence
run approved endpoint check
restart according to runbook
rollback according to approved procedure
collect evidence
```

Không nên tự ý:

```text
change production DB schema
disable TLS verification
chmod -R 777
run service as root
delete logs
kill unknown process
change firewall broadly
increase JVM memory without analysis/approval
modify application code
reset production credentials
```

Nếu root cause vượt authority:

```text
collect evidence
preserve state
describe impact
describe tests performed
escalate
```

---

# 110. Security summary

Một production-like Java deployment nên tuân thủ:

```text
Dedicated service account
      +
Least privilege
      +
No embedded secrets
      +
Restricted config files
      +
TLS where appropriate
      +
Certificate validation
      +
Backend exposure minimized
      +
Controlled changes
      +
Audit/evidence
```

---

# 111. Các anti-pattern phải nhớ

### Anti-pattern 1

```bash
chmod -R 777
```

để "fix deployment".

### Anti-pattern 2

Run Tomcat:

```text
root
```

không có lý do operational được phê duyệt.

### Anti-pattern 3

Password trong source:

```java
password = "..."
```

### Anti-pattern 4

Hard-code production host:

```java
dbHost = "10.0.1.20";
```

### Anti-pattern 5

Deployment success =:

```text
file copied
```

### Anti-pattern 6

Restart liên tục thay vì đọc logs.

### Anti-pattern 7

`-Xmx = total system RAM`.

### Anti-pattern 8

Tắt TLS verification để hết certificate error.

Ví dụ đặc biệt nguy hiểm:

```text
trust everything
```

không phải proper fix.

### Anti-pattern 9

Upgrade Tomcat major version mà không kiểm tra:

```text
Java requirement
Jakarta compatibility
application compatibility
config differences
```

### Anti-pattern 10

Thay release mà không giữ previous known-good artifact.

---

# 112. Một mental model tích hợp Day 12–14

Bạn cần nhìn được toàn bộ flow sau:

```text
                         CLIENT
                           |
                        HTTPS
                           |
                           v
                    Reverse Proxy
                    :443 / TLS
                           |
                           v
                      Tomcat :8080
                           |
                           v
                    Context /orders
                           |
                           v
                      orders.war
                           |
               +-----------+-----------+
               |                       |
               v                       v
          Configuration             Logs
               |                       |
      env/system props/files           |
               |                       |
               v                       |
            Java/JVM <-----------------+
               |
               |
               v
             JDBC
               |
             TLS
               |
               v
         PostgreSQL :5432
```

Bên dưới tất cả:

```text
Linux
  |
systemd
  |
service account
  |
filesystem permissions
```

---

# 113. Operator validation chain

Nếu cần nhớ duy nhất một sequence cho Module 1, hãy nhớ:

```text
systemd
↓
process
↓
port
↓
HTTP
↓
context
↓
application
↓
configuration
↓
database
↓
logs
↓
public proxy/TLS
```

Ví dụ:

```bash
systemctl status tomcat

pgrep -a java

ss -lntp | grep ':8080'

curl -i http://127.0.0.1:8080/orders/

journalctl -u tomcat -n 100
```

Sau đó mới mở rộng sang dependency/proxy.

---

# 114. Current-version note

Một vài điểm cần cập nhật so với những tutorial Java/Tomcat cũ trên Internet:

- Java 17 và Java 25 đều là LTS theo roadmap hiện tại của Oracle. [Oracle](https://www.oracle.com/java/technologies/java-se-support-roadmap.html?utm_source=chatgpt.com)
- Tomcat 11 yêu cầu Java 17 trở lên. [Apache Tomcat](https://tomcat.apache.org/migration-11.0.html?utm_source=chatgpt.com)
- Tomcat 10.1 yêu cầu Java 11 trở lên. [Apache Tomcat](https://tomcat.apache.org/migration-10.1.html?utm_source=chatgpt.com)
- Tomcat 10+ dùng Jakarta APIs thay vì Java EE-era `javax.*`; WAR cũ có thể cần migration. [Apache Tomcat](https://tomcat.apache.org/migration-10?utm_source=chatgpt.com)
- Tài liệu Apache ngày 12/10/2026 cho thấy Tomcat 11.0.27 là release hiện hành lúc tra cứu này. Đây là thông tin freshness, **không phải thứ cần học thuộc cho exam**. [Apache Tomcat](https://tomcat.apache.org/?utm_source=chatgpt.com)

Exam focus nên là **compatibility principles**, không phải nhớ patch version.

---

# 115. Exam Focus — những thứ phải trả lời được

Sau Module 1, bạn nên có thể tự giải thích không nhìn tài liệu các câu sau:

### Java/runtime

1. JVM khác JDK thế nào?
2. Java source trở thành running Linux process như thế nào?
3. `UnsupportedClassVersionError` gợi ý điều gì?
4. Vì sao Java 17/25 compatibility phải kiểm tra cùng framework và Tomcat?

### Deployment

5. JAR khác WAR thế nào?
6. Executable JAR khác WAR deploy vào Tomcat thế nào?
7. Tomcat có phải JVM không?
8. `CATALINA_HOME` và `CATALINA_BASE` khác nhau thế nào?
9. `webapps`, `conf`, `logs`, `work`, `temp` dùng làm gì?
10. Vì sao `orders.war` thường map `/orders`?
11. `ROOT.war` có ý nghĩa gì?

### Linux/systemd

12. Vì sao Tomcat không nên chạy root?
13. Dedicated service account mang lại lợi ích gì?
14. `systemctl daemon-reload` khác `restart` như thế nào?
15. `Restart=on-failure` có giải quyết bad config không?

### Configuration

16. Externalized configuration giải quyết vấn đề gì?
17. Environment variable khác Java system property thế nào?
18. Profile có phải Java SE standard không?
19. Port với context path khác nhau thế nào?
20. `127.0.0.1:8080` khác `0.0.0.0:8080` thế nào?

### JVM memory

21. `-Xms` và `-Xmx` là gì?
22. Vì sao `-Xmx4g` không có nghĩa Java chỉ dùng tối đa 4 GB?
23. Vì sao không nên đặt `Xmx = toàn bộ RAM`?
24. `OutOfMemoryError` có phải lúc nào cũng giải quyết bằng tăng heap không?

### Logging/restart

25. Vì sao service active chưa chứng minh application healthy?
26. Safe restart gồm những validation gì?
27. `journalctl -u` dùng để kiểm tra gì?
28. Vì sao DEBUG logging production cần kiểm soát?

### Database

29. JDBC URL có những thành phần gì?
30. `connection refused`, timeout và authentication failure khác nhau thế nào?
31. Authentication khác authorization thế nào?

### Security/TLS

32. Secret khác ordinary config thế nào?
33. Vì sao environment variable chưa phải secret manager?
34. TLS cung cấp confidentiality, integrity và authentication như thế nào?
35. Certificate chain là gì?
36. Truststore là gì?
37. Vì sao hostname verification quan trọng?
38. `sslmode=require` khác `verify-full` như thế nào?

### Reverse proxy

39. Reverse proxy khác forward proxy thế nào?
40. Vì sao thường expose 443 mà không expose Tomcat 8080?
41. `proxy_pass` làm gì?
42. Reverse-proxy headers có thể ảnh hưởng application như thế nào?

### Deployment operations

43. Vì sao copy WAR thành công chưa phải deployment success?
44. Một complete deployment validation cần kiểm tra layer nào?
45. Known-good state là gì?
46. Vì sao rollback phải tính cả config chứ không chỉ WAR?
47. Database migration có thể làm application rollback khó thế nào?

---

# 116. Những command cần nhận diện ngay

Bạn không nhất thiết thuộc mọi option, nhưng nên hiểu ngay mục đích của các command này:

```bash
java -version
javac -version

java -jar app.jar

jar tf app.jar

ps -ef
pgrep -a java

systemctl status tomcat
systemctl start tomcat
systemctl stop tomcat
systemctl restart tomcat
systemctl daemon-reload

journalctl -u tomcat
journalctl -u tomcat -f

ss -lntp

curl -i http://127.0.0.1:8080/

df -h
df -i

keytool -list -cacerts
keytool -printcert -file certificate.crt

getent hosts db.example.com

pg_isready -h db.example.com -p 5432
```

Và JVM settings:

```text
-Xms
-Xmx
-Dname=value
```

---

# 117. Master troubleshooting table

| Symptom                           | Layer đầu tiên nên nghi ngờ | Evidence quan trọng       |
| --------------------------------- | --------------------------- | ------------------------- |
| `systemctl` failed                | service/startup             | `status`, `journalctl`    |
| Java process absent               | JVM/service                 | process + logs            |
| `Address already in use`          | socket/port                 | `ss -lntp`                |
| Connection refused                | TCP/listener                | socket/service            |
| Timeout                           | network/firewall            | route/connectivity        |
| HTTP 404                          | context/routing             | URL/context               |
| HTTP 500                          | application/dependency      | app logs                  |
| DB auth failure                   | credentials/auth            | DB error                  |
| DB permission denied              | authorization               | SQL/role error            |
| `UnsupportedClassVersionError`    | Java compatibility          | runtime/build versions    |
| `ClassNotFoundException`          | dependency/classpath        | deployment/logs           |
| `Permission denied`               | Linux filesystem/account    | `id`, `ls -l`, path perms |
| `PKIX path building failed`       | TLS trust                   | chain/truststore          |
| hostname verification error       | TLS identity                | SAN/hostname              |
| OOM                               | JVM/application memory      | memory/GC/config          |
| Public URL fails, localhost works | proxy/network/TLS           | backend vs proxy tests    |
| New release fails                 | deployment/config delta     | before/after evidence     |

---

# 118. Module 1 mastery summary

Ba Day này thực chất dạy một câu chuyện duy nhất:

```text
DAY 12
"Java application được chạy và deploy như thế nào?"

        ↓

DAY 13
"Runtime đó được configure và vận hành thế nào?"

        ↓

DAY 14
"Runtime đó kết nối dependency, được expose an toàn,
validate và rollback thế nào?"
```

Bạn cần hình thành mental model cuối cùng:

```text
Artifact
   ↓
Tomcat/JVM
   ↓
Linux process
   ↓
systemd service
   ↓
Runtime configuration
   ↓
Port/context
   ↓
Application
   ↓
JDBC/database
   ↓
TLS / Reverse Proxy
   ↓
Validation
   ↓
Operations
   ↓
Rollback
```

Điểm quan trọng nhất đối với fresher administrator không phải chỉ biết chạy:

```bash
systemctl restart tomcat
```

mà phải hiểu:

```text
Tôi đang thay đổi layer nào?
Tôi mong đợi trạng thái gì?
Tôi dùng evidence nào để chứng minh?
Nếu fail thì layer nào fail?
Tôi có quyền sửa hay phải escalate?
Tôi rollback về trạng thái nào?
Tôi chứng minh recovery bằng gì?
```

Nếu đạt được cách suy nghĩ đó thì bạn đã nắm đúng mục tiêu của **Day 12–14**, chứ không chỉ nhớ một tập command.
