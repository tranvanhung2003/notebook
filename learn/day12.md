# Day 12 — Java Application Runtime, Tomcat Deployment & systemd Service

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 12 thuộc phần **Java Application Configuration**. Phạm vi bắt buộc hôm nay là:

> **Concept/Lecture:** Java application runtime architecture, JAR/WAR, Tomcat layout, Java 17, Java 25, deployment flow, service account, và systemd integration.  
> **Assignment/Lab:** Deploy ứng dụng Java được cung cấp lên Tomcat và quản lý nó như một Linux service. 00_MASTER_SYLLABUS_Java_Fresher…

Đây là bước chuyển rất quan trọng: từ Day 2–6 bạn quản trị Linux, Day 7–11 quản trị AWS; bắt đầu Day 12, Linux/EC2 trở thành **nền tảng chạy application thực tế**.

Trong bài hôm nay, chúng ta sẽ phải đạt được chuỗi năng lực:

```text
Java artifact
    ↓
JDK/JVM
    ↓
Tomcat
    ↓
Linux process
    ↓
systemd service
    ↓
TCP :8080
    ↓
HTTP request
    ↓
Java web application
```

Và khi gặp lỗi, bạn phải có khả năng lần ngược:

```text
HTTP failure
  ↓
port/socket
  ↓
Tomcat process
  ↓
systemd
  ↓
Tomcat log
  ↓
JVM
  ↓
WAR/application
  ↓
permission/config/version compatibility
```

---

# 2. Objectives — Học xong Day 12 bạn phải làm được gì?

Kết thúc Day 12, bạn chưa chỉ cần biết "Tomcat là gì". Bạn phải tự mình:

| Năng lực                                           | Mức cần đạt            |
| -------------------------------------------------- | ---------------------- |
| Giải thích JVM/JDK/Application/Tomcat relationship | Bắt buộc               |
| Phân biệt JAR và WAR                               | Bắt buộc               |
| Biết khi nào chạy `java -jar`, khi nào deploy WAR  | Bắt buộc               |
| Hiểu Java 17 và Java 25                            | Bắt buộc               |
| Kiểm tra Java runtime thực tế                      | Bắt buộc               |
| Hiểu `JAVA_HOME`                                   | Bắt buộc               |
| Hiểu Tomcat process                                | Bắt buộc               |
| Hiểu Tomcat directory layout                       | Bắt buộc               |
| Phân biệt `CATALINA_HOME` và `CATALINA_BASE`       | Bắt buộc               |
| Deploy WAR vào Tomcat                              | Bắt buộc               |
| Hiểu context path                                  | Bắt buộc               |
| Tạo dedicated service account                      | Bắt buộc               |
| Tạo `tomcat.service`                               | Bắt buộc               |
| `enable/start/stop/restart/status` bằng systemctl  | Bắt buộc               |
| Xem log bằng `journalctl` và Tomcat logs           | Bắt buộc               |
| Chứng minh port 8080 đang listen                   | Bắt buộc               |
| Validate ứng dụng bằng `curl`                      | Bắt buộc               |
| Chẩn đoán service start failure                    | Bắt buộc               |
| Chẩn đoán port conflict                            | Bắt buộc               |
| Nhận biết Java/Tomcat/app incompatibility          | Bắt buộc               |
| Biết rollback/recovery cơ bản                      | Operations requirement |

---

# 3. Mental Model — Java Application thực sự chạy như thế nào?

Một lỗi phổ biến của fresher là tưởng:

```text
application.war → Linux chạy application.war
```

Thực tế không phải.

Với ứng dụng WAR:

```text
Linux
│
├── systemd
│      │
│      └── tomcat.service
│             │
│             └── JVM
│                  │
│                  └── Apache Tomcat
│                         │
│                         └── application.war
│                                │
│                                ├── Servlet
│                                ├── Application classes
│                                ├── Libraries
│                                └── Resources
│
└── TCP port 8080
       ↑
       │
HTTP client
curl/browser/load balancer
```

Tomcat bản thân nó cũng là **một Java application**.

Khi bạn start Tomcat:

```text
systemd
   ↓
catalina.sh
   ↓
java
   ↓
org.apache.catalina.startup.Bootstrap
   ↓
Tomcat starts
   ↓
Tomcat opens port 8080
   ↓
Tomcat discovers WAR
   ↓
Tomcat creates web application Context
   ↓
Application becomes available
```

Do đó khi request thất bại, nguyên nhân có thể nằm ở rất nhiều lớp.

---

# 4. JDK, JVM, JRE — cần hiểu đúng

## JVM

**JVM — Java Virtual Machine** là runtime thực thi Java bytecode.

Source:

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

sẽ tạo:

```text
Hello.class
```

`Hello.class` chứa Java bytecode.

Sau đó:

```bash
java Hello
```

JVM load và thực thi bytecode.

Mental model:

```text
.java
  ↓ javac
.class bytecode
  ↓ JVM
machine instructions
  ↓
CPU
```

---

## JDK

**JDK — Java Development Kit** cung cấp JVM/runtime cùng các development/diagnostic tools.

Một số command bạn cần nhận ra:

```text
java
javac
jar
jcmd
jps
jstack
jmap
keytool
```

Trong Operations, ngay cả khi không compile application, JDK vẫn hữu ích vì diagnostic tools.

Kiểm tra:

```bash
java -version
```

và:

```bash
javac -version
```

---

# 5. Java 17 và Java 25

Đây là nội dung syllabus bắt buộc.

Java 17 được phát hành năm 2021 và Java 25 năm 2025. Theo Oracle roadmap hiện tại, cả **17 và 25 đều là LTS releases**; Java 25 có Premier Support của Oracle dự kiến tới September 2030. Oracle Java 17 đã đi qua mốc Premier Support September 2026 và có Extended Support roadmap tới September 2029. Đây là **Oracle support lifecycle**; lifecycle của OpenJDK distribution/vendor khác có thể khác. [Oracle](https://www.oracle.com/europe/java/technologies/java-se-support-roadmap.html?utm_source=chatgpt.com)

Điều quan trọng đối với administrator không phải "25 lớn hơn 17 nên phải dùng 25".

Quyết định runtime phải dựa trên:

```text
Application certification
        +
Framework support
        +
Tomcat compatibility
        +
JDK vendor support
        +
Company standard
        +
Security patch policy
```

Không tự ý upgrade production:

```text
Java 17
   ↓
Java 25
```

chỉ vì Java 25 mới hơn.

---

## Kiểm tra Java đang thực sự được dùng

```bash
java -version
```

Ví dụ output:

```text
openjdk version "17.0.x" ...
OpenJDK Runtime Environment ...
OpenJDK 64-Bit Server VM ...
```

Đây chỉ là **Example output**.

Expected invariant là:

```text
major Java version == version được application/Tomcat/build sheet phê duyệt
```

Kiểm tra binary:

```bash
which java
```

Sau đó:

```bash
readlink -f "$(command -v java)"
```

Ví dụ:

```text
/usr/lib/jvm/java-17-openjdk-amd64/bin/java
```

Từ đó:

```text
JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
```

Không được đặt:

```text
JAVA_HOME=/usr/bin/java
```

vì `JAVA_HOME` phải chỉ tới **JDK/JRE root directory**, không phải executable.

Có thể lấy tự động:

```bash
JAVA_BIN="$(readlink -f "$(command -v java)")"
JAVA_HOME="$(dirname "$(dirname "$JAVA_BIN")")"

echo "$JAVA_HOME"
```

Sau đó kiểm tra:

```bash
"$JAVA_HOME/bin/java" -version
```

---

# 6. Java 17 hay Java 25 cho lab hôm nay?

Tôi khuyến nghị baseline lab hôm nay sử dụng:

```text
Java 17
+
Tomcat 10.1
```

trừ khi provided application/build sheet của bạn quy định phiên bản khác.

Lý do không phải Java 25 không chạy được. Apache hiện ghi nhận Tomcat 10.1 yêu cầu Java 11 trở lên, trong khi Tomcat 11 yêu cầu Java 17 trở lên. Vì vậy Java 17 là baseline phù hợp để học dependency relationship. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/appdev/installation.html?utm_source=chatgpt.com)

Java 25 vẫn là phần bắt buộc cần hiểu và bạn nên biết cách verify nó.

**Không được suy luận:**

```text
Tomcat supports Java 25
⇒ application supports Java 25
```

Tomcat compatibility và application compatibility là hai vấn đề khác nhau.

---

# 7. JAR là gì?

**JAR — Java ARchive** về bản chất là ZIP-based archive dùng để đóng gói:

```text
.class files
resources
metadata
libraries hoặc application classes
```

Xem nội dung:

```bash
jar tf application.jar
```

hoặc:

```bash
unzip -l application.jar
```

Một executable JAR thường chứa:

```text
META-INF/
META-INF/MANIFEST.MF
com/example/...
```

Trong manifest có thể tồn tại:

```text
Main-Class: com.example.Main
```

Khi đó:

```bash
java -jar application.jar
```

JVM biết class nào phải chạy đầu tiên.

---

# 8. WAR là gì?

**WAR — Web Application Archive** là package cho Java web application.

Typical structure:

```text
application.war
│
├── META-INF/
│
├── WEB-INF/
│   ├── web.xml
│   ├── classes/
│   └── lib/
│
├── index.jsp
├── static/
└── ...
```

`WEB-INF/classes/`:

```text
compiled application classes
```

`WEB-INF/lib/`:

```text
dependency JARs
```

`WEB-INF/web.xml`:

```text
web application deployment descriptor
```

Không phải mọi modern Java application đều bắt buộc có explicit `web.xml`, vì annotation/configuration có thể thay thế nhiều nội dung.

---

# 9. JAR vs WAR — điểm thi rất quan trọng

| JAR                                          | WAR                                           |
| -------------------------------------------- | --------------------------------------------- |
| Generic Java archive                         | Java web application archive                  |
| Có thể chạy bằng `java -jar`                 | Thường deploy vào Servlet container           |
| Có thể có `Main-Class`                       | Có `WEB-INF` structure                        |
| App có thể tự quản lifecycle                 | Container quản lifecycle                      |
| Spring Boot executable JAR là ví dụ phổ biến | External Tomcat deployment là ví dụ điển hình |

Một executable JAR:

```bash
java -jar myapp.jar
```

Một traditional WAR:

```text
myapp.war
       ↓
Tomcat webapps/
       ↓
Tomcat deploy
       ↓
/myapp
```

**Không làm:**

```bash
java -jar myapp.war
```

trừ khi artifact được project/framework thiết kế đặc biệt để executable.

**Không làm:**

```bash
cp app.jar /opt/tomcat/webapps/
```

và tưởng Tomcat sẽ tự deploy nó như WAR.

---

# 10. Context Path

Giả sử:

```text
webapps/inventory.war
```

Thông thường context sẽ là:

```text
/inventory
```

Request:

```text
http://host:8080/inventory/
```

Nếu:

```text
ROOT.war
```

thì context root:

```text
/
```

Apache Tomcat documentation xác nhận WAR/directory name trong appBase thường xác định context path khi không chỉ định context khác. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/html-manager-howto.html?utm_source=chatgpt.com)

Đây là lý do lỗi sau thường chỉ là sai URL:

```text
Tomcat running
port 8080 listening
application deployed
curl localhost:8080 → 404
```

nhưng:

```text
curl localhost:8080/myapp/
```

lại thành công.

**404 không đồng nghĩa Tomcat chết.**

---

# 11. Tomcat là gì?

Apache Tomcat là Java Servlet/JSP container và HTTP server được dùng để host Java web applications.

Tomcat 10.1 hiện implement Jakarta Servlet 6.0 và Jakarta Pages 3.1. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/?utm_source=chatgpt.com)

Request flow đơn giản:

```text
Client
   ↓
TCP :8080
   ↓
Tomcat Connector
   ↓
Engine
   ↓
Host
   ↓
Context
   ↓
Application
   ↓
Servlet/controller
   ↓
Response
```

Bạn chưa cần trở thành Tomcat developer, nhưng administrator phải hiểu từng layer đủ để troubleshooting.

---

# 12. Cảnh báo cực kỳ quan trọng: `javax.*` vs `jakarta.*`

Tomcat hiện đại có một breaking change quan trọng.

Tomcat 9-era applications thường dùng:

```java
javax.servlet.*
```

Trong Tomcat 10 trở lên:

```java
jakarta.servlet.*
```

Apache cảnh báo rõ ứng dụng chuyển từ Tomcat 9 và cũ hơn sang Tomcat 10/11 có thể phải thay đổi code do namespace migration từ Java EE sang Jakarta EE. [Apache Tomcat](https://tomcat.apache.org/download-11.html?utm_source=chatgpt.com)

Ví dụ app cũ:

```java
import javax.servlet.http.HttpServlet;
```

App mới:

```java
import jakarta.servlet.http.HttpServlet;
```

Nếu deploy sai generation, bạn có thể thấy:

```text
ClassNotFoundException
NoClassDefFoundError
javax.servlet...
```

hoặc deployment failure.

Troubleshooting rule:

```text
Java version OK
Tomcat running
WAR deploy fails
↓
Check application/Tomcat Jakarta generation compatibility
```

Đừng "fix" bằng cách random copy JAR vào Tomcat.

---

# 13. Tomcat Directory Layout

Đây là phần bắt buộc của Day 12.

Một Tomcat installation thường có:

```text
apache-tomcat-10.1.x/
│
├── bin/
├── conf/
├── lib/
├── logs/
├── temp/
├── webapps/
└── work/
```

Apache documentation xác định `bin`, `conf`, `logs`, và `webapps` là các key directories, đồng thời mô tả `work`, `temp`, `lib` và quan hệ với `CATALINA_BASE`. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/introduction.html?utm_source=chatgpt.com)

### `bin/`

Startup/runtime scripts:

```text
catalina.sh
startup.sh
shutdown.sh
version.sh
```

Ví dụ:

```bash
bin/version.sh
```

---

### `conf/`

Tomcat configuration.

Quan trọng nhất:

```text
conf/server.xml
```

Ngoài ra thường gặp:

```text
web.xml
context.xml
logging.properties
tomcat-users.xml
```

Không sửa config production trực tiếp mà không có:

```text
backup
change ticket
syntax/semantic validation
rollback plan
```

---

### `lib/`

Libraries của Tomcat/container.

Không coi đây là nơi "quăng dependency vào để chữa lỗi" một cách tùy tiện.

Dependency application thông thường thuộc:

```text
WEB-INF/lib/
```

---

### `logs/`

Tomcat logs.

Ví dụ:

```bash
ls -lah /opt/tomcat/current/logs
```

Có thể gặp:

```text
catalina.*
localhost.*
localhost_access_log.*
```

Tomcat sử dụng JULI/java.util.logging cho internal logging mặc định. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/logging.html?utm_source=chatgpt.com)

---

### `webapps/`

Nơi deploy web applications khi Host sử dụng default appBase.

Ví dụ:

```text
webapps/
├── ROOT/
├── docs/
├── examples/
├── manager/
└── myapp.war
```

Đối với server thực tế, sample/default applications không cần thiết thường được loại bỏ theo security baseline của tổ chức.

---

### `temp/`

Temporary files dùng bởi JVM/Tomcat.

---

### `work/`

Tomcat-generated working files.

Không coi `work/` như nơi lưu data quan trọng.

---

# 14. `CATALINA_HOME` vs `CATALINA_BASE`

Đây là câu hỏi interview/exam rất tốt.

`CATALINA_HOME`:

```text
Tomcat installation/binaries
```

Ví dụ:

```text
/opt/tomcat/apache-tomcat-10.1.60
```

`CATALINA_BASE`:

```text
runtime configuration của một Tomcat instance
```

Nếu chỉ có một simple installation:

```text
CATALINA_HOME == CATALINA_BASE
```

Có thể đều là:

```text
/opt/tomcat/current
```

Nhưng trong architecture sạch:

```text
CATALINA_HOME
  ↓
shared Tomcat binaries

CATALINA_BASE-1
  ↓
instance A configuration/runtime

CATALINA_BASE-2
  ↓
instance B configuration/runtime
```

Apache khuyến khích hiểu sự tách biệt này vì nó cho phép nhiều Tomcat instances dùng chung binaries nhưng có riêng configuration, logs, apps và runtime data. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/introduction.html?utm_source=chatgpt.com)

---

# 15. Deployment Flow

Một controlled WAR deployment có conceptual flow:

```text
Receive artifact
       ↓
identify artifact/version
       ↓
verify integrity
       ↓
verify compatibility
       ↓
backup/current-state evidence
       ↓
place WAR
       ↓
Tomcat deploy
       ↓
check service
       ↓
check port
       ↓
check logs
       ↓
HTTP validation
       ↓
application validation
       ↓
capture evidence
```

Tomcat có thể automatically deploy WAR đặt trong `webapps` tùy Host configuration. Khi `unpackWARs=true`, WAR có thể được exploded thành directory. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/html-manager-howto.html?utm_source=chatgpt.com)

Ví dụ:

```text
myapp.war
```

sau deploy có thể trở thành:

```text
webapps/
├── myapp.war
└── myapp/
```

Không nhầm directory `myapp/` với source code.

---

# 16. Service Account — tại sao không chạy Tomcat bằng root?

Đây là phần cực kỳ quan trọng về security.

Bad:

```text
root
 └── java/Tomcat
```

Nếu application bị compromise:

```text
application compromise
        ↓
Tomcat process compromise
        ↓
root privileges
        ↓
whole server compromised
```

Đúng hơn:

```text
root/systemd
   ↓ starts as
tomcat user
   ↓
Java/Tomcat
```

Service account thường:

```text
username: tomcat
interactive login: disabled
sudo: no
SSH key: no
home: controlled
permissions: minimum required
```

Kiểm tra:

```bash
id tomcat
```

Ví dụ:

```text
uid=997(tomcat) gid=997(tomcat) groups=997(tomcat)
```

Expected invariant:

```text
Tomcat Java process owner = tomcat
```

chứ không phải UID cụ thể.

---

# 17. systemd Integration

Manual start:

```bash
/opt/tomcat/current/bin/catalina.sh run
```

có vấn đề:

```text
SSH disconnect
human lifecycle management
boot persistence
inconsistent environment
weak auditing
harder monitoring
```

Systemd giải quyết:

```text
systemd
  ↓
tomcat.service
  ↓
Tomcat/JVM
```

Bạn có thể:

```bash
sudo systemctl start tomcat
sudo systemctl stop tomcat
sudo systemctl restart tomcat
sudo systemctl status tomcat
```

Boot automatically:

```bash
sudo systemctl enable tomcat
```

Logs:

```bash
sudo journalctl -u tomcat
```

Recent:

```bash
sudo journalctl -u tomcat -n 100 --no-pager
```

Follow:

```bash
sudo journalctl -u tomcat -f
```

---

# 18. Vì sao nên dùng `catalina.sh run` với systemd?

Nhiều tutorial trên Internet viết:

```ini
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
```

Cách đó có thể hoạt động nhưng tạo thêm vấn đề lifecycle tracking vì `startup.sh` khởi động Tomcat theo kiểu background.

Đối với systemd, mental model sạch hơn là:

```text
systemd
   ↓ owns
foreground Tomcat process
```

nên dùng:

```bash
catalina.sh run
```

với:

```ini
Type=simple
```

Systemd theo dõi trực tiếp Java/Tomcat process.

---

# 19. Lab Architecture

Lab hôm nay:

```text
Linux / EC2
│
├── Java 17
│
├── /opt/tomcat/
│   ├── apache-tomcat-10.1.x/
│   └── current -> apache-tomcat-10.1.x
│
├── user: tomcat
│
├── /etc/systemd/system/tomcat.service
│
└── Tomcat :8080
        │
        └── java-fresher.war
```

Client-side validation:

```text
curl
 ↓
127.0.0.1:8080
 ↓
Tomcat
 ↓
/java-fresher
```

---

# 20. Guided Lab — Deploy Java Application to Tomcat

## Lab Safety

**Thực hiện trên lab/non-production host.**

Trước khi thay đổi:

```bash
hostnamectl
whoami
date
df -h
free -h
ss -ltnp
java -version
```

Đây không phải nghi thức thừa. Nó xác định:

```text
đang ở server nào?
đang là user nào?
disk đủ không?
RAM đủ không?
8080 có đang bị dùng không?
Java hiện tại là gì?
```

Đặc biệt:

```bash
ss -ltnp | grep ':8080'
```

Nếu đã có service production trên 8080:

**không tiếp tục chiếm port.**

---

# 21. Step 1 — Verify Operating Environment

```bash
hostnamectl
```

```bash
cat /etc/os-release
```

```bash
uname -m
```

```bash
free -h
```

```bash
df -h
```

Kiểm tra Java:

```bash
java -version
```

Nếu chưa cài:

### Ubuntu/Debian example

```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
```

Sau đó:

```bash
java -version
javac -version
```

Package availability khác nhau theo distribution/version. Đối với Java 25, **không assume** package `openjdk-25-jdk` tồn tại trong repo của distro; kiểm tra repo/vendor policy trước. Oracle cũng hỗ trợ JDK 25 Linux qua archive, `.deb`, `.rpm`, và các Oracle Linux packages tùy platform. [docs.oracle.com](https://docs.oracle.com/en/java/javase/25/install/installation-jdk-linux-platforms.html?utm_source=chatgpt.com)

---

# 22. Step 2 — Determine JAVA_HOME

```bash
JAVA_BIN="$(readlink -f "$(command -v java)")"

echo "$JAVA_BIN"
```

Sau đó:

```bash
JAVA_HOME="$(dirname "$(dirname "$JAVA_BIN")")"

echo "$JAVA_HOME"
```

Validation:

```bash
"$JAVA_HOME/bin/java" -version
```

Evidence:

```bash
printf 'JAVA_HOME=%s\n' "$JAVA_HOME"
```

---

# 23. Step 3 — Inspect port 8080 before deployment

```bash
sudo ss -ltnp | grep ':8080' || echo "8080 currently unused"
```

Expected trước khi Tomcat start:

```text
8080 currently unused
```

Nếu port đã listen:

```bash
sudo ss -ltnp | grep ':8080'
```

Tìm PID/process trước.

Không kill process chỉ vì cần port.

---

# 24. Step 4 — Create Tomcat Service Account

Kiểm tra trước:

```bash
getent passwd tomcat
```

Nếu không tồn tại:

```bash
sudo useradd \
  --system \
  --home-dir /opt/tomcat \
  --create-home \
  --shell /usr/sbin/nologin \
  tomcat
```

Một số distro đặt `nologin` ở:

```text
/sbin/nologin
```

Kiểm tra trước bằng:

```bash
command -v nologin
```

Nếu output khác, dùng path thực tế của distro.

Validation:

```bash
id tomcat
```

```bash
getent passwd tomcat
```

Không đặt password.

Không thêm `tomcat` vào sudoers.

---

# 25. Step 5 — Install Tomcat

Tại thời điểm hiện tại, Apache documentation đang hiển thị Tomcat **10.1.60**, phát hành September 9, 2026; Tomcat 11 hiện là **11.0.26**. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/setup.html?utm_source=chatgpt.com)

Lab sẽ minh họa bằng Tomcat 10.1.

Tạo staging directory:

```bash
cd /tmp
```

Download **từ Apache official distribution source theo environment của bạn** và verify checksum/signature theo release process.

Giả sử artifact đã có:

```text
/tmp/apache-tomcat-10.1.60.tar.gz
```

Kiểm tra:

```bash
ls -lh /tmp/apache-tomcat-10.1.60.tar.gz
```

Không unzip một file không rõ nguồn gốc trực tiếp vào production location.

---

# 26. Step 6 — Extract to `/opt/tomcat`

```bash
sudo mkdir -p /opt/tomcat
```

Extract:

```bash
sudo tar -xzf /tmp/apache-tomcat-10.1.60.tar.gz -C /opt/tomcat
```

Kiểm tra:

```bash
ls -lah /opt/tomcat
```

Tạo stable symlink:

```bash
sudo ln -sfn \
  /opt/tomcat/apache-tomcat-10.1.60 \
  /opt/tomcat/current
```

Validate:

```bash
readlink -f /opt/tomcat/current
```

Lợi ích:

```text
systemd
   ↓
/opt/tomcat/current
```

không phụ thuộc version directory cụ thể.

Khi upgrade:

```text
current
   old → apache-tomcat-x
   new → apache-tomcat-y
```

Đây là **Supplementary / Beyond explicit syllabus**, nhưng là pattern operations rất hữu ích.

---

# 27. Step 7 — Set permissions carefully

Không làm:

```bash
sudo chown -R tomcat:tomcat /opt/tomcat
```

một cách mù quáng nếu không hiểu hậu quả.

Nếu service account có quyền sửa cả executable/binaries, khi Tomcat bị compromise attacker có thể persist bằng cách sửa Tomcat files.

Mô hình tốt hơn:

```text
Tomcat binaries/config
      root-owned

runtime writable directories
      tomcat-owned
```

Đặt installation base:

```bash
sudo chown -R root:root \
  /opt/tomcat/apache-tomcat-10.1.60
```

Cho scripts executable:

```bash
sudo chmod +x /opt/tomcat/current/bin/*.sh
```

Các runtime directories cần write:

```bash
sudo chown -R tomcat:tomcat \
  /opt/tomcat/current/logs \
  /opt/tomcat/current/temp \
  /opt/tomcat/current/work \
  /opt/tomcat/current/webapps
```

Config nên readable nhưng không writable bởi Tomcat:

```bash
sudo chown -R root:tomcat /opt/tomcat/current/conf
```

```bash
sudo chmod -R g+rX /opt/tomcat/current/conf
```

---

# 28. Step 8 — Verify Tomcat Version

```bash
sudo -u tomcat \
  /opt/tomcat/current/bin/version.sh
```

Bạn cần quan sát:

```text
Server version
Server number
OS Name
OS Version
Architecture
JVM Version
JVM Vendor
CATALINA_BASE
CATALINA_HOME
```

Đây là một command troubleshooting rất có giá trị.

---

# 29. Step 9 — Create systemd unit

Tìm JAVA_HOME:

```bash
JAVA_HOME="$(dirname "$(dirname "$(readlink -f "$(command -v java)")")")"

echo "$JAVA_HOME"
```

Sau đó:

```bash
sudo tee /etc/systemd/system/tomcat.service >/dev/null <<EOF
[Unit]
Description=Apache Tomcat Java Application Server
After=network.target

[Service]
Type=simple
User=tomcat
Group=tomcat

Environment="JAVA_HOME=$JAVA_HOME"
Environment="CATALINA_HOME=/opt/tomcat/current"
Environment="CATALINA_BASE=/opt/tomcat/current"

ExecStart=/opt/tomcat/current/bin/catalina.sh run

Restart=on-failure
RestartSec=5

UMask=0027

[Install]
WantedBy=multi-user.target
EOF
```

Bây giờ inspect:

```bash
sudo cat /etc/systemd/system/tomcat.service
```

---

# 30. Đọc hiểu unit file

### `After=network.target`

Service start sau basic network target ordering.

Không có nghĩa:

```text
Internet chắc chắn usable
database chắc chắn reachable
DNS chắc chắn resolved
```

---

### `Type=simple`

Systemd coi process của `ExecStart` là main service process.

---

### `User=tomcat`

Critical security control:

```text
Java/Tomcat ≠ root
```

---

### `Environment="JAVA_HOME=..."`

Service của systemd không nhất thiết kế thừa shell environment bạn đang login.

Đó là lý do:

```bash
echo $JAVA_HOME
```

của bạn có thể đúng,

nhưng Tomcat service vẫn fail nếu unit không có environment phù hợp.

---

### `CATALINA_HOME`

Tomcat installation.

### `CATALINA_BASE`

Tomcat instance runtime configuration.

---

### `ExecStart=... catalina.sh run`

Run Tomcat foreground để systemd quản lý process.

---

### `Restart=on-failure`

Nếu process chết bất thường:

```text
systemd → restart
```

Nhưng khi admin:

```bash
systemctl stop tomcat
```

thì không phải failure recovery.

---

### `UMask=0027`

File mới được tạo sẽ hạn chế permission với group/others hơn default permissive configurations.

**Supplementary / security hardening.**

---

# 31. Step 10 — Validate unit file before start

Rất nên dùng:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/tomcat.service
```

Không có output thường nghĩa là không phát hiện lỗi đáng chú ý.

Sau unit change:

```bash
sudo systemctl daemon-reload
```

Sai lầm rất phổ biến:

```text
edit unit
↓
systemctl restart
↓
"tại sao config mới không có tác dụng?"
```

Vì quên:

```bash
systemctl daemon-reload
```

---

# 32. Step 11 — Enable Tomcat

```bash
sudo systemctl enable tomcat
```

Kiểm tra:

```bash
systemctl is-enabled tomcat
```

Expected:

```text
enabled
```

**Enable ≠ start.**

`enable`:

```text
configure boot-time activation
```

`start`:

```text
start right now
```

---

# 33. Step 12 — Start Tomcat

```bash
sudo systemctl start tomcat
```

Không lập tức mở browser rồi đoán.

Kiểm tra control plane trước:

```bash
sudo systemctl status tomcat --no-pager
```

Expected invariant:

```text
Active: active (running)
```

---

# 34. Step 13 — Check logs

```bash
sudo journalctl -u tomcat -n 100 --no-pager
```

Sau đó Tomcat logs:

```bash
sudo ls -lah /opt/tomcat/current/logs
```

Có thể:

```bash
sudo tail -n 100 /opt/tomcat/current/logs/catalina.out
```

Nhưng filename/log routing phụ thuộc Tomcat/systemd configuration.

**Evidence priority:**

```text
systemctl
journalctl
Tomcat logs
```

không phải đoán lỗi.

---

# 35. Step 14 — Check process

```bash
pgrep -af 'org.apache.catalina.startup.Bootstrap'
```

Sau đó:

```bash
ps -ef | grep '[o]rg.apache.catalina.startup.Bootstrap'
```

Bạn cần chứng minh process chạy với:

```text
tomcat
```

không phải:

```text
root
```

---

# 36. Step 15 — Check socket

```bash
sudo ss -ltnp | grep ':8080'
```

Tomcat default HTTP connector thường listen 8080.

Bạn có thể thấy:

```text
LISTEN ...
*:8080
...
java
```

Expected invariant:

```text
Java/Tomcat owns the expected listening socket
```

Không chỉ:

```text
port 8080 open
```

vì process khác cũng có thể chiếm 8080.

---

# 37. Step 16 — Test HTTP locally

```bash
curl -i http://127.0.0.1:8080/
```

Bạn đang kiểm tra:

```text
TCP connection
+
HTTP response
+
Tomcat request processing
```

Đừng chỉ chạy:

```bash
ping localhost
```

Ping thành công không chứng minh HTTP hoạt động.

---

# 38. Step 17 — Inspect Provided Application Before Deployment

Giả sử artifact của assignment là:

```text
/tmp/java-fresher.war
```

Kiểm tra file:

```bash
ls -lh /tmp/java-fresher.war
```

Type:

```bash
file /tmp/java-fresher.war
```

Xem structure:

```bash
jar tf /tmp/java-fresher.war | head -50
```

Tìm:

```text
WEB-INF/
WEB-INF/classes/
WEB-INF/lib/
```

Có thể xem manifest:

```bash
unzip -p \
  /tmp/java-fresher.war \
  META-INF/MANIFEST.MF
```

Nếu có `WEB-INF/web.xml`:

```bash
unzip -p \
  /tmp/java-fresher.war \
  WEB-INF/web.xml | head -100
```

Không sửa WAR trước khi hiểu artifact.

---

# 39. Step 18 — Record artifact checksum

```bash
sha256sum /tmp/java-fresher.war
```

Ví dụ evidence:

```text
<hash>  /tmp/java-fresher.war
```

Điều này cho phép trả lời:

> Bạn deploy artifact nào?

Thay vì:

> Tôi nghĩ là file mới nhất.

Operationally, hai file cùng tên:

```text
java-fresher.war
```

không chứng minh chúng cùng content.

Checksum giúp xác định artifact chính xác.

---

# 40. Step 19 — Deploy the WAR

Vì đây là lần deploy lab đầu tiên, kiểm tra trước:

```bash
ls -lah /opt/tomcat/current/webapps/
```

Copy artifact với tên context mong muốn:

```bash
sudo cp \
  /tmp/java-fresher.war \
  /opt/tomcat/current/webapps/java-fresher.war
```

Ownership:

```bash
sudo chown tomcat:tomcat \
  /opt/tomcat/current/webapps/java-fresher.war
```

Permission:

```bash
sudo chmod 640 \
  /opt/tomcat/current/webapps/java-fresher.war
```

Kiểm tra:

```bash
ls -lh \
  /opt/tomcat/current/webapps/java-fresher.war
```

---

# 41. Step 20 — Observe Deployment

Theo dõi logs:

```bash
sudo journalctl -u tomcat -f
```

hoặc terminal khác:

```bash
sudo tail -f /opt/tomcat/current/logs/catalina.out
```

Tomcat có thể unpack:

```text
java-fresher.war
        ↓
java-fresher/
```

Kiểm tra:

```bash
ls -lah /opt/tomcat/current/webapps/
```

Tomcat documentation xác nhận WAR trong appBase có thể được unpack khi `unpackWARs=true`. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/html-manager-howto.html?utm_source=chatgpt.com)

---

# 42. Step 21 — Validate Application Context

Vì artifact là:

```text
java-fresher.war
```

context thông thường:

```text
/java-fresher
```

Test:

```bash
curl -i \
  http://127.0.0.1:8080/java-fresher/
```

Nếu bạn chỉ cần status code:

```bash
curl -sS \
  -o /dev/null \
  -w 'HTTP %{http_code}\n' \
  http://127.0.0.1:8080/java-fresher/
```

Nhưng:

```text
HTTP 200
```

chỉ chứng minh endpoint trả 200.

Application validation phải dựa vào expected behavior của provided application.

Nếu app có endpoint được tài liệu hóa như:

```text
/health
```

hoặc một business endpoint nào đó, hãy test endpoint **được application specification cung cấp**, không tự giả định.

---

# 43. Full Validation Chain

Sau deployment, chạy:

```bash
sudo systemctl is-active tomcat
```

```bash
sudo systemctl status tomcat --no-pager
```

```bash
pgrep -af 'org.apache.catalina.startup.Bootstrap'
```

```bash
sudo ss -ltnp | grep ':8080'
```

```bash
curl -i http://127.0.0.1:8080/java-fresher/
```

```bash
sudo journalctl -u tomcat -n 50 --no-pager
```

```bash
ls -lah /opt/tomcat/current/webapps/
```

Đây là nhiều layers evidence:

```text
systemd
process
socket
HTTP
application deployment
logs
```

Nếu tất cả đều hợp lệ, confidence cao hơn rất nhiều so với:

```text
systemctl says active
```

---

# 44. Evidence Pack cho Assignment

Bạn nên lưu evidence tương tự:

```text
day12/
├── 01-java-version.txt
├── 02-tomcat-version.txt
├── 03-service-account.txt
├── 04-tomcat-unit.txt
├── 05-service-status.txt
├── 06-process.txt
├── 07-port.txt
├── 08-war-checksum.txt
├── 09-http-validation.txt
└── 10-logs.txt
```

Ví dụ:

```bash
mkdir -p ~/day12-evidence
```

```bash
java -version \
  > ~/day12-evidence/01-java-version.txt 2>&1
```

```bash
sudo -u tomcat /opt/tomcat/current/bin/version.sh \
  > ~/day12-evidence/02-tomcat-version.txt 2>&1
```

```bash
id tomcat \
  > ~/day12-evidence/03-service-account.txt
```

```bash
sudo systemctl cat tomcat \
  > ~/day12-evidence/04-tomcat-unit.txt
```

```bash
sudo systemctl status tomcat --no-pager \
  > ~/day12-evidence/05-service-status.txt
```

```bash
pgrep -af 'org.apache.catalina.startup.Bootstrap' \
  > ~/day12-evidence/06-process.txt
```

```bash
sudo ss -ltnp \
  > ~/day12-evidence/07-port.txt
```

```bash
sha256sum /tmp/java-fresher.war \
  > ~/day12-evidence/08-war-checksum.txt
```

```bash
curl -i http://127.0.0.1:8080/java-fresher/ \
  > ~/day12-evidence/09-http-validation.txt 2>&1
```

```bash
sudo journalctl -u tomcat -n 100 --no-pager \
  > ~/day12-evidence/10-logs.txt
```

---

# 45. Failure Injection 1 — Port 8080 Already in Use

**Chỉ làm trong lab. Stop Tomcat trước.**

```bash
sudo systemctl stop tomcat
```

Confirm:

```bash
sudo ss -ltnp | grep ':8080' || echo "8080 free"
```

Nếu Python 3 có sẵn:

```bash
python3 -m http.server 8080 \
  --bind 127.0.0.1 >/tmp/day12-port-test.log 2>&1 &
```

Lưu PID:

```bash
TEST_PID=$!
echo "$TEST_PID"
```

Confirm:

```bash
ss -ltnp | grep ':8080'
```

Bây giờ:

```bash
sudo systemctl start tomcat
```

Check:

```bash
sudo systemctl status tomcat --no-pager
```

Logs:

```bash
sudo journalctl -u tomcat -n 100 --no-pager
```

Bạn nên tìm evidence tương tự:

```text
BindException
Address already in use
```

Mental model:

```text
Tomcat wants :8080
       ↓
kernel
       ↓
:8080 already owned
       ↓
bind() fails
       ↓
Tomcat startup fails
```

Không chữa bằng:

```bash
kill -9 random_pid
```

Phải xác định process nào đang giữ port và nó có được phép dừng hay không.

Cleanup:

```bash
kill "$TEST_PID"
```

Validate:

```bash
ss -ltnp | grep ':8080' || echo "8080 free again"
```

Start Tomcat:

```bash
sudo systemctl start tomcat
```

Validate:

```bash
sudo systemctl status tomcat --no-pager
```

```bash
sudo ss -ltnp | grep ':8080'
```

```bash
curl -i http://127.0.0.1:8080/java-fresher/
```

---

# 46. Failure Injection 2 — Wrong JAVA_HOME

**Lab only.**

Trước tiên lưu unit:

```bash
sudo cp \
  /etc/systemd/system/tomcat.service \
  /etc/systemd/system/tomcat.service.day12.bak
```

Sai configuration có thể khiến application unavailable.

Thay đổi `JAVA_HOME` sang invalid path, reload rồi start sẽ khiến service không khởi động đúng.

Điều quan trọng không phải phá service mà là học workflow:

```text
symptom
Tomcat won't start

↓ expected
systemctl active

↓ recent change
JAVA_HOME changed

↓ evidence
systemctl status
journalctl

↓ hypothesis
Java runtime path invalid

↓ safe test
/path/bin/java -version

↓ root cause
wrong JAVA_HOME

↓ remediation
restore known-good unit

↓ validation
daemon-reload
restart
status
port
curl
```

Khôi phục:

```bash
sudo cp \
  /etc/systemd/system/tomcat.service.day12.bak \
  /etc/systemd/system/tomcat.service
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart tomcat
```

---

# 47. Common Failure: `systemctl status` says failed

Không ngay lập tức reinstall.

Chạy:

```bash
sudo systemctl status tomcat --no-pager
```

Sau đó:

```bash
sudo journalctl -u tomcat -b --no-pager
```

Kiểm tra:

```bash
"$JAVA_HOME/bin/java" -version
```

```bash
sudo -u tomcat test -x \
  /opt/tomcat/current/bin/catalina.sh \
  && echo OK
```

```bash
sudo -u tomcat test -r \
  /opt/tomcat/current/conf/server.xml \
  && echo OK
```

```bash
sudo ss -ltnp | grep ':8080'
```

Evidence trước, remediation sau.

---

# 48. Common Failure: Permission denied

Symptom có thể:

```text
Permission denied
cannot create log
cannot create temp file
cannot deploy WAR
```

Điều tra component-by-component:

```bash
namei -l /opt/tomcat/current/logs
```

```bash
namei -l /opt/tomcat/current/webapps/java-fresher.war
```

```bash
sudo -u tomcat test -r \
  /opt/tomcat/current/webapps/java-fresher.war \
  && echo readable
```

```bash
sudo -u tomcat test -w \
  /opt/tomcat/current/logs \
  && echo writable
```

Không chữa bằng:

```bash
chmod -R 777 /opt/tomcat
```

`777` thường chỉ che giấu root cause đồng thời làm xấu security.

---

# 49. Common Failure: `404 Not Found`

Đầu tiên đừng kết luận application down.

Xác định:

```text
Is Tomcat running?
```

```bash
systemctl is-active tomcat
```

```text
Is port open?
```

```bash
ss -ltnp | grep ':8080'
```

```text
Does Tomcat answer?
```

```bash
curl -i http://127.0.0.1:8080/
```

```text
What is deployed?
```

```bash
ls -lah /opt/tomcat/current/webapps/
```

Nếu có:

```text
java-fresher.war
```

test:

```bash
curl -i http://127.0.0.1:8080/java-fresher/
```

404 thường có thể là:

```text
wrong context path
wrong endpoint
deployment failed
application route not defined
```

---

# 50. Common Failure: Connection refused

```bash
curl http://127.0.0.1:8080
```

returns:

```text
Connection refused
```

Mental model:

```text
request reached local TCP stack
       ↓
nothing accepting that socket
```

Kiểm tra:

```bash
ss -ltnp | grep ':8080'
```

Nếu không có output:

```text
Tomcat down
OR
Tomcat listening different port/address
OR
startup failed
```

Check:

```bash
systemctl status tomcat
journalctl -u tomcat
```

---

# 51. Common Failure: Timeout

Timeout khác connection refused.

```text
Connection refused
≈ destination reachable but no listener / reject

Timeout
≈ packets/request not receiving expected response
```

Nếu remote request timeout nhưng localhost works:

```bash
curl localhost:8080/...  → works
remote curl              → timeout
```

Application/Tomcat có thể hoàn toàn khỏe.

Investigate:

```text
bind address
Linux firewall
AWS Security Group
NACL
routing
client path
```

Đây chính là integration với Day 5 và Day 8–11.

---

# 52. Common Failure: Tomcat active nhưng application deploy fail

Systemd:

```text
active (running)
```

không chứng minh mọi WAR healthy.

Một Tomcat JVM có thể chạy trong khi một Context deploy thất bại.

Check logs:

```bash
sudo journalctl -u tomcat -n 200 --no-pager
```

```bash
grep -RniE \
  'SEVERE|ERROR|Exception|Caused by' \
  /opt/tomcat/current/logs/
```

Một số root causes:

```text
WAR corrupt
incompatible Java version
javax/jakarta mismatch
missing application dependency
bad web.xml
permission error
application startup exception
database/config dependency failure
```

Day 14 sẽ đi sâu hơn phần external dependency/deployment validation.

---

# 53. Java version mismatch

Ví dụ class được compile bằng Java mới hơn runtime.

Bạn có thể gặp:

```text
UnsupportedClassVersionError
```

Mental model:

```text
compiled with newer bytecode level
           ↓
older JVM attempts loading
           ↓
cannot understand class version
```

Troubleshooting:

```bash
java -version
```

Sau đó đối chiếu application build requirement.

Không tự nâng Java production trước khi xác nhận compatibility.

---

# 54. Tomcat 10/11 Compatibility

Current official requirements:

| Tomcat      | Minimum Java |
| ----------- | -----------: | -------------------------------------- |
| Tomcat 10.1 |      Java 11 |
| Tomcat 11   |      Java 17 | :chatgpt-content-reference{index="13"} |

Do đó:

```text
Tomcat 11 + Java 11
```

không phải supported runtime combination.

Nhưng:

```text
Tomcat 10.1 + Java 17
```

phù hợp yêu cầu runtime của Tomcat.

Và Java 25 cũng vượt minimum runtime requirement, nhưng application compatibility vẫn phải verify riêng.

---

# 55. `restart` không phải universal fix

Anti-pattern:

```text
application problem
↓
restart Tomcat
↓
works temporarily
↓
ticket closed
```

Bạn chưa tìm root cause.

Correct:

```text
symptom
↓
capture evidence
↓
scope
↓
logs/metrics/process
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
↓
prevention
```

Restart có thể:

```text
clear temporary state
release memory
rebuild connection pools
reload config
```

nhưng cũng xóa một số runtime evidence và gây outage.

Day 13 sẽ đi sâu **safe restart**.

---

# 56. Service Account Security

Bạn cần kiểm tra:

```bash
ps -eo user,pid,ppid,cmd | grep '[j]ava'
```

Bad:

```text
root  ... java ...
```

Expected:

```text
tomcat ... java ...
```

Service account không cần:

```text
interactive shell
sudo
SSH login
write access /etc
write access Tomcat binaries
write access application artifact repository
```

Principle:

```text
read what it needs
write only runtime state
execute only required binaries
```

---

# 57. Không expose Tomcat Manager tùy tiện

Tomcat distribution có thể chứa management web applications như `manager`.

Không:

```text
open manager to 0.0.0.0/0
+
weak password
```

Không dùng Manager UI là điều kiện bắt buộc để deploy Day 12.

Manual controlled WAR placement + systemd + evidence là đủ để hiểu architecture.

---

# 58. Không chạy Tomcat bằng root để bind port 80

Sai:

```text
Need :80
↓
run Tomcat as root
```

Đây là escalation of privilege không cần thiết.

Pattern thông thường:

```text
Internet
   ↓
reverse proxy / load balancer :80/:443
   ↓
Tomcat :8080
```

Reverse proxy/TLS thuộc nội dung tiếp theo của Day 14.

---

# 59. Operations — Stop / Start / Restart

Stop:

```bash
sudo systemctl stop tomcat
```

Verify:

```bash
systemctl is-active tomcat
```

và:

```bash
ss -ltnp | grep ':8080'
```

Start:

```bash
sudo systemctl start tomcat
```

Verify lại:

```bash
systemctl status tomcat --no-pager
ss -ltnp | grep ':8080'
curl -i http://127.0.0.1:8080/java-fresher/
```

Restart:

```bash
sudo systemctl restart tomcat
```

Không coi command return code `0` là toàn bộ application validation.

---

# 60. Basic Rollback Concept

Day 14 sẽ học rollback đầy đủ hơn, nhưng Day 12 phải biết nguyên tắc.

Trước deployment update:

```text
old WAR
 ↓ backup
deployment
 ↓
validate
```

Nếu failure:

```text
stop service / controlled undeploy
↓
restore previous artifact
↓
start
↓
validate
↓
capture rollback evidence
```

Một simple lab backup:

```bash
sudo cp \
  /opt/tomcat/current/webapps/java-fresher.war \
  /opt/tomcat/current/webapps/java-fresher.war.bak
```

Nhưng trong real release management tốt hơn nên lưu artifact immutable/versioned ngoài active `webapps`.

---

# 61. L1 vs Escalation

L1 Administrator có thể an toàn xử lý:

| Tình huống                          | L1 action                      |
| ----------------------------------- | ------------------------------ |
| Service stopped                     | inspect → approved restart     |
| Port conflict từ known lab process  | resolve safely                 |
| Wrong permission do approved change | restore approved permission    |
| Wrong `JAVA_HOME`                   | restore approved configuration |
| WAR missing                         | redeploy approved artifact     |
| Wrong context URL                   | correct validation             |
| Logs available                      | collect/preserve evidence      |

Nên escalate khi:

| Tình huống                                     | Lý do                                 |
| ---------------------------------------------- | ------------------------------------- |
| Application throws internal business exception | Developer/app owner domain            |
| Unknown JVM crash                              | deeper JVM analysis                   |
| Repeated OutOfMemoryError                      | Day 13/JVM tuning + app investigation |
| Unknown Java compatibility issue               | application owner/build team          |
| `javax`/`jakarta` migration required           | code/build change                     |
| Production change outside ticket               | approval boundary                     |
| Security compromise suspected                  | security incident process             |
| Database dependency problem outside L1 scope   | DB/app escalation                     |

Fresher giỏi không phải người "sửa được mọi thứ".

Fresher giỏi biết:

```text
what is safe
what is approved
what evidence exists
when to stop
when to escalate
```

---

# 62. Integrated Troubleshooting Model

Khi user nói:

> “Ứng dụng Java không truy cập được.”

Không restart ngay.

Đi theo chuỗi:

```text
1. What URL?
        ↓
2. DNS/IP correct?
        ↓
3. Network path reachable?
        ↓
4. SG/firewall permits traffic?
        ↓
5. Host reachable?
        ↓
6. Port listening?
        ↓
7. Which process owns port?
        ↓
8. systemd service running?
        ↓
9. JVM alive?
        ↓
10. Tomcat started successfully?
        ↓
11. WAR deployed?
        ↓
12. Correct context?
        ↓
13. App startup exception?
        ↓
14. External dependency failure?
```

Đây chính là tư duy integrated operations mà curriculum hướng tới.

---

# 63. Một command không đủ để chứng minh health

`ps`:

```text
process exists
```

không chứng minh HTTP works.

`systemctl status`:

```text
systemd sees process
```

không chứng minh app deployed.

`ss`:

```text
socket listens
```

không chứng minh application endpoint works.

`curl`:

```text
HTTP endpoint responded
```

nhưng chưa chắc business dependency healthy.

Proper validation:

```text
systemd
+
process
+
socket
+
logs
+
HTTP
+
application-specific behavior
```

---

# 64. Mistakes / Anti-Patterns phải tránh

1. Chạy Tomcat bằng `root`.
2. `chmod -R 777` để sửa permission.
3. Thấy lỗi là restart ngay.
4. Dùng `kill -9` làm shutdown thông thường.
5. Không kiểm tra `java -version`.
6. Nhầm `JAVA_HOME` với `/usr/bin/java`.
7. Nhầm JAR với WAR.
8. Copy JAR vào `webapps` rồi chờ Tomcat deploy.
9. Nghĩ `systemctl active` nghĩa application healthy.
10. Nghĩ port 8080 listen nghĩa đúng Tomcat đang listen.
11. Không xem logs.
12. Không biết context path.
13. Không backup trước update.
14. Deploy artifact không checksum/version.
15. Upgrade Java/Tomcat không kiểm tra compatibility.
16. Bỏ qua `javax` → `jakarta`.
17. Edit systemd unit nhưng quên `daemon-reload`.
18. Dùng `startup.sh` thủ công rồi đồng thời để systemd quản lý, tạo lifecycle confusion.
19. Mở 8080 public chỉ để "test nhanh".
20. Lưu credentials trực tiếp vào WAR/unit file.

---

# 65. Supplementary / Beyond the explicit syllabus — systemd Hardening

Sau khi baseline chạy đúng, có thể nghiên cứu thêm:

```ini
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true
ProtectSystem=full
```

Nhưng không copy security directives mù quáng.

Ví dụ `ProtectSystem=` hoặc write restrictions có thể khiến Tomcat không ghi được:

```text
logs/
temp/
work/
webapps/
```

Operations principle:

```text
baseline first
↓
understand required filesystem access
↓
apply hardening
↓
restart
↓
validate
```

Không:

```text
copy "secure Tomcat systemd" từ Internet
↓
service stops working
```

---

# 66. Exam Focus

Bạn phải có khả năng trả lời ngắn gọn nhưng chính xác các câu dạng:

**JDK khác JVM thế nào?**

JVM thực thi Java bytecode; JDK cung cấp Java runtime cùng development/diagnostic tooling.

**JAR khác WAR?**

JAR là Java archive tổng quát và có thể executable; WAR là package cho Java web application thường được Servlet container như Tomcat deploy.

**Tomcat dùng làm gì?**

Host và execute Java Servlet-based web applications, xử lý HTTP request thông qua container architecture.

**`CATALINA_HOME`?**

Tomcat installation root.

**`CATALINA_BASE`?**

Runtime configuration root của một Tomcat instance.

**Tại sao dùng service account?**

Least privilege và giới hạn blast radius nếu Tomcat/application bị compromise.

**Tại sao dùng systemd?**

Lifecycle management, startup at boot, process supervision, consistent environment và centralized operational control.

**Tomcat active nhưng app 404 — làm gì?**

Check context path, deployed WAR/exploded context và logs; không kết luận Tomcat down.

**Tomcat không start — check gì trước?**

`systemctl status`, `journalctl`, Java runtime/JAVA_HOME, permission, port conflict, Tomcat logs.

**Tại sao Java 25 không tự động tốt hơn Java 17?**

Runtime phải phù hợp application certification, framework/Tomcat compatibility và organization support policy.

---

# 67. Knowledge Check

Không nhìn lại phần trên, hãy tự trả lời 15 câu sau:

1. JVM đóng vai trò gì trong runtime architecture?
2. JDK khác JVM ở đâu?
3. `java -jar app.jar` thực hiện điều gì?
4. WAR khác JAR thế nào?
5. `inventory.war` mặc định thường tạo context path nào?
6. `ROOT.war` thường đại diện context nào?
7. `CATALINA_HOME` là gì?
8. `CATALINA_BASE` là gì?
9. Tại sao không chạy Tomcat bằng root?
10. `systemctl enable tomcat` khác `systemctl start tomcat` như thế nào?
11. Tại sao `systemctl status` = active chưa chứng minh application healthy?
12. `curl` trả `Connection refused` gợi ý layer nào?
13. Tomcat start thất bại với `Address already in use` thì bước evidence đầu tiên là gì?
14. `javax.servlet` và `jakarta.servlet` liên quan gì tới Tomcat compatibility?
15. Bạn cần những evidence nào để chứng minh deployment Day 12 thành công?

---

# 68. Independent Practical Challenge

Sau guided lab, hãy tự thực hiện challenge mà không nhìn solution:

```text
Starting state:
- Linux server
- Java installed
- Tomcat installed
- provided WAR available

Required end state:
- dedicated tomcat service account
- Tomcat controlled by systemd
- starts automatically at boot
- runs as non-root
- Java version proven
- WAR artifact checksum recorded
- WAR deployed
- correct context identified
- application responds locally
- port ownership proven
- logs clean enough for acceptance
- evidence directory completed
```

Sau đó tự inject:

```text
port 8080 conflict
```

và phải chứng minh bạn có thể:

```text
detect
→ gather evidence
→ identify root cause
→ recover
→ validate
```

mà không dùng `chmod 777`, `kill -9` hay reinstall Tomcat.

---

# 69. Day 12 Mastery Check

Day 12 **chưa được coi là mastered chỉ vì bạn đọc xong bài này**.

Mức hiểu lý thuyết:

```text
JVM/JDK
JAR/WAR
Tomcat
Java 17/25
layout
deployment
systemd
service account
```

Mức practical mastery chỉ đạt khi bạn thực sự chứng minh được:

```text
$ java -version

$ sudo -u tomcat /opt/tomcat/current/bin/version.sh

$ id tomcat

$ systemctl status tomcat

$ pgrep -af org.apache.catalina.startup.Bootstrap

$ ss -ltnp | grep 8080

$ curl http://127.0.0.1:8080/<context>/

$ journalctl -u tomcat

$ sha256sum <application>.war
```

và hiểu **mỗi command đang chứng minh layer nào**.

---

# 70. Bản đồ kiến thức Day 12 cần ghi nhớ

```text
                         DAY 12
                           │
       ┌───────────────────┼────────────────────┐
       │                   │                    │
     Java                Tomcat              Linux
       │                   │                    │
   JDK/JVM              layout              service user
   Java 17              bin                 permission
   Java 25              conf                process
       │                logs                   │
       │                webapps                │
       │                work                   │
       │                temp                   │
       │                   │                    │
       └───────── WAR ─────┤                    │
                           │                    │
                     deployment            systemd
                           │                    │
                        Context             start
                           │                stop
                           │                restart
                           │                enable
                           │                    │
                           └──────────┬─────────┘
                                      │
                                  validation
                                      │
                     systemd → process → port
                              → log → HTTP
```

Một kiến thức current-state quan trọng để bạn giữ đúng mental model: Tomcat 10.1 hiện yêu cầu Java 11+, Tomcat 11 yêu cầu Java 17+, trong khi Java 17 và Java 25 đều thuộc nhóm LTS theo Oracle roadmap. Điều đó **không thay thế requirement của application**: version được deploy phải là combination đã được application/build specification chấp nhận. [Oracle](https://www.oracle.com/de/java/technologies/java-se-support-roadmap.html?utm_source=chatgpt.com)

**Phần mandatory của Day 12 hôm nay đã được bao phủ đầy đủ về lý thuyết và guided lab.** Bước quan trọng nhất bây giờ là thực hiện lab thật: tạo service account → dựng Tomcat → viết systemd unit → deploy WAR → thu evidence → inject port failure → recovery. Day 13 sau đó mới đi sâu vào **externalized configuration, environment variables, profiles, ports, context path, JVM memory options, logs và safe restart**, đúng thứ tự syllabus. 00_MASTER_SYLLABUS_Java_Fresher…
