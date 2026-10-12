# Day 14 — Database Connectivity, Secrets, TLS, Release Validation & Rollback

## 1. Syllabus Alignment

Theo `MASTER_SYLLABUS`, đây là nguồn curriculum bắt buộc của toàn khóa; các yêu cầu trong syllabus không được bỏ qua hoặc tự ý thay đổi. 00_MASTER_SYLLABUS_Java_Fresher…

**Day 14 bắt buộc gồm:**

> **Concept/Lecture:** Database connectivity, secrets handling, TLS certificate basics, reverse-proxy awareness, deployment validation, rollback, and reduced-scope consolidation.  
> **Assignment/Lab:** Connect the application to PostgreSQL, apply a basic TLS configuration, perform release validation, and execute rollback. 00_MASTER_SYLLABUS_Java_Fresher…

Ngày hôm nay có thể hiểu là ngày hoàn thiện khối **Java Application Configuration** trước khi Day 15 chuyển trọng tâm sang PostgreSQL administration.

| Phạm vi           | Day 14                                                       |
| ----------------- | ------------------------------------------------------------ |
| **Mandatory**     | Application → PostgreSQL connectivity                        |
| **Mandatory**     | Secrets handling                                             |
| **Mandatory**     | TLS / certificate fundamentals                               |
| **Mandatory**     | Reverse proxy awareness                                      |
| **Mandatory**     | Deployment/release validation                                |
| **Mandatory**     | Rollback                                                     |
| **Mandatory**     | Reduced-scope consolidation của Day 12–14                    |
| **Supplementary** | PostgreSQL TLS `verify-full`                                 |
| **Supplementary** | Nginx reverse proxy configuration example                    |
| **Supplementary** | systemd credentials / secret manager                         |
| **Supplementary** | Certificate rotation, mTLS, advanced production architecture |

---

# 2. Mục tiêu Day 14

Sau Day 14, bạn cần có khả năng nhìn một Java application dưới góc độ operations như chuỗi:

```text
Client
   |
   | HTTP / HTTPS
   v
[Reverse Proxy / Load Balancer]     <- optional
   |
   | HTTP / HTTPS
   v
[Tomcat + Java application]
   |
   | JDBC
   | TCP 5432
   | optionally TLS
   v
[PostgreSQL]
```

Và khi một release mới được triển khai, bạn phải chứng minh được:

```text
Artifact đúng
    ↓
Configuration đúng
    ↓
Secrets đúng và không bị lộ
    ↓
Tomcat start thành công
    ↓
Port listen
    ↓
HTTPS/TLS hoạt động
    ↓
Application health tốt
    ↓
Database connection tốt
    ↓
Business function hoạt động
    ↓
Logs không có lỗi bất thường
```

Nếu bất kỳ bước quan trọng nào thất bại:

```text
STOP
 ↓
Collect evidence
 ↓
Decide L1 remediation vs escalation
 ↓
Rollback to known-good version
 ↓
Validate again
```

Đây chính là tư duy mà một fresher Operations/Admin cần hình thành.

---

# 3. Prerequisites và assumptions của lab

Day 14 phụ thuộc trực tiếp Day 12 và Day 13.

Bạn cần đã biết:

- Java application chạy trên Tomcat.
- Tomcat được quản lý bằng `systemd`.
- Biết `CATALINA_HOME` và `CATALINA_BASE`.
- Biết deploy WAR.
- Biết externalized configuration.
- Biết environment variables.
- Biết safe restart.
- Biết `systemctl`, `journalctl`, `ss`, `curl`.
- Biết Linux ownership/permissions.

Một điểm rất quan trọng:

**Day 15 mới là ngày PostgreSQL provisioning, users/roles, `pg_hba.conf`, backup/restore.** Syllabus xác định rõ phần đó ở Day 15. 00_MASTER_SYLLABUS_Java_Fresher…

Vì vậy lab Day 14 sẽ giả định rằng:

```text
PostgreSQL server     : đã được instructor/lab cung cấp
Database              : đã tồn tại
Database role         : đã tồn tại
Network access        : đã được cho phép
Application WAR       : đã có từ Day 12
Tomcat service        : đã có từ Day 12
External config       : đã thực hành Day 13
```

Chúng ta tập trung vào **application-side connectivity và operations**, không biến Day 14 thành Day 15.

---

# 4. Environment assumptions

Ví dụ xuyên suốt bài:

```text
Application service : tomcat.service
Tomcat HTTP port    : 8080
Tomcat HTTPS port   : 8443

Application context : /myapp
Health endpoint     : /myapp/health

PostgreSQL:
host                : db.lab.internal
port                : 5432
database            : appdb
user                : appuser
```

**Những tên này là lab convention, không phải giá trị bắt buộc trong syllabus.**

Trong máy của bạn phải thay chúng bằng thông tin thật.

Đặc biệt:

```text
<APP_CONTEXT>
<HEALTH_PATH>
<DB_HOST>
<DB_NAME>
<DB_USER>
```

không được copy nguyên xi nếu environment thực tế khác.

---

# PHẦN I — DATABASE CONNECTIVITY

# 5. Java application kết nối PostgreSQL như thế nào?

Mental model quan trọng nhất:

```text
Java application
      |
      | JDBC API
      v
PostgreSQL JDBC Driver (pgJDBC)
      |
      | DNS resolution
      v
DB hostname → IP
      |
      | TCP
      v
IP:5432
      |
      | optional TLS negotiation
      v
PostgreSQL
      |
      | Authentication
      v
DB role
      |
      | Authorization
      v
database/schema/table permissions
      |
      v
SQL query
```

Một câu đơn giản như:

> "Application không connect được DB."

thực tế có thể là lỗi ở **ít nhất bảy layer khác nhau**.

---

# 6. JDBC là gì?

JDBC = **Java Database Connectivity**.

JDBC cung cấp API chuẩn để Java application tương tác với relational database.

Với PostgreSQL, application thường sử dụng **pgJDBC**, là PostgreSQL JDBC driver viết bằng Java và giao tiếp trực tiếp với PostgreSQL native protocol. [jdbc.postgresql.org](https://jdbc.postgresql.org/documentation/head/?utm_source=chatgpt.com)

Một JDBC URL cơ bản:

```text
jdbc:postgresql://db.lab.internal:5432/appdb
```

Phân tích:

```text
jdbc:
  └── JDBC URL

postgresql:
  └── database driver/protocol

db.lab.internal
  └── database hostname

5432
  └── PostgreSQL TCP port

appdb
  └── database
```

---

# 7. Các thông tin application thường cần

Tối thiểu:

```text
Database hostname
Database port
Database name
Database username
Database password
```

Có thể có thêm:

```text
TLS settings
connection timeout
socket timeout
connection pool size
schema
application name
read-only setting
```

Ví dụ JDBC:

```text
jdbc:postgresql://db.lab.internal:5432/appdb
```

Với TLS verification:

```text
jdbc:postgresql://db.lab.internal:5432/appdb?sslmode=verify-full&sslrootcert=/etc/myapp/certs/postgres-ca.crt
```

pgJDBC hỗ trợ các `sslmode`:

```text
disable
allow
prefer
require
verify-ca
verify-full
```

Trong tài liệu pgJDBC hiện hành, default là `prefer`; `verify-ca` xác minh certificate chain, còn `verify-full` xác minh cả certificate chain **và hostname**. [jdbc.postgresql.org](https://jdbc.postgresql.org/documentation/use/?utm_source=chatgpt.com)

Trong môi trường security-sensitive, `verify-full` là lựa chọn được khuyến nghị. [jdbc.postgresql.org](https://jdbc.postgresql.org/documentation/ssl/?utm_source=chatgpt.com)

---

# 8. Đừng nhầm encryption với authentication

Ví dụ:

```text
sslmode=require
```

có nghĩa là:

```text
Traffic phải được TLS encryption
```

nhưng theo pgJDBC, `require` **không phải hostname/certificate identity verification đầy đủ**. [jdbc.postgresql.org](https://jdbc.postgresql.org/documentation/ssl/?utm_source=chatgpt.com)

Trong khi:

```text
sslmode=verify-full
```

thực hiện cả:

```text
Encrypted connection
+
Trusted CA verification
+
Hostname verification
```

Đây là sự khác biệt cực kỳ hay xuất hiện trong câu hỏi theory.

---

# 9. Các layer cần kiểm tra khi DB connection fail

## Layer 1 — Configuration

Application có đúng:

```text
DB host?
DB port?
DB database?
DB username?
```

Không bao giờ bắt đầu troubleshooting bằng:

```bash
sudo systemctl restart tomcat
```

một cách vô thức.

Restart không sửa được hostname sai.

---

## Layer 2 — DNS

Kiểm tra:

```bash
getent hosts db.lab.internal
```

hoặc:

```bash
getent ahosts db.lab.internal
```

Expected invariant:

```text
Hostname resolve được thành đúng IP.
```

Không cần output giống hệt một ví dụ cố định.

---

## Layer 3 — TCP connectivity

Nếu có `nc`:

```bash
nc -vz db.lab.internal 5432
```

Possible success:

```text
Connection to db.lab.internal 5432 port [tcp/postgresql] succeeded!
```

Nếu `nc` không cài, Bash có thể test TCP:

```bash
timeout 3 bash -c '</dev/tcp/db.lab.internal/5432'
echo $?
```

`0` thường biểu thị TCP connect được.

### Nhưng nhớ:

TCP connection thành công chỉ chứng minh:

```text
client → host:5432 reachable
```

Nó **không chứng minh**:

```text
password đúng
database tồn tại
role đủ quyền
TLS trust đúng
query chạy được
application hoạt động
```

---

# 10. JDBC driver

Nếu application không có PostgreSQL JDBC driver, bạn có thể gặp:

```text
ClassNotFoundException: org.postgresql.Driver
```

hoặc:

```text
No suitable driver
```

Nếu application là WAR, kiểm tra:

```bash
jar tf myapp.war | grep -i postgresql
```

Có thể thấy dạng:

```text
WEB-INF/lib/postgresql-42.x.x.jar
```

Nhưng không phải application nào cũng bundle JDBC driver.

Một số architecture đặt driver ở Tomcat shared library:

```text
$CATALINA_BASE/lib/
```

hoặc:

```text
$CATALINA_HOME/lib/
```

### Anti-pattern

Thấy thiếu JDBC driver rồi ngay lập tức download một random JAR từ Internet và copy vào Tomcat.

Không làm vậy trong production.

Bạn cần biết:

```text
application dependency model
approved driver version
release artifact baseline
vendor/project build specification
```

---

# 11. Database connection pool

Production Java application hiếm khi mở một TCP connection mới cho mọi request.

Thường dùng:

```text
Connection Pool
```

Mental model:

```text
                     ┌─ DB connection 1
Application ─ Pool ──┼─ DB connection 2
                     ├─ DB connection 3
                     └─ ...
```

Lợi ích:

- giảm connection setup overhead;
- kiểm soát số DB connections;
- reuse connection;
- timeout và health-check tốt hơn.

Nhưng pool tạo ra failure mode mới:

```text
pool exhausted
connection leak
too many database connections
stale connections
wrong timeout
```

Day 16 sẽ đi sâu hơn sessions/locks/capacity.

Hôm nay bạn chỉ cần hiểu layer này tồn tại.

---

# PHẦN II — SECRETS HANDLING

# 12. Secret là gì?

Trong Day 14, secret có thể là:

```text
Database password
API token
Private key
Keystore password
Client certificate private key
Cloud credential
```

Không phải secret:

```text
DB hostname
DB port
certificate public portion
application port
context path
```

Certificate thường là **public information**.

Private key mới là bí mật.

---

# 13. Secret lifecycle

Không chỉ cần "giấu password".

Một secret có lifecycle:

```text
Creation
  ↓
Distribution
  ↓
Storage
  ↓
Access
  ↓
Use
  ↓
Rotation
  ↓
Revocation
  ↓
Deletion
```

Security tốt phải kiểm soát toàn bộ vòng đời này.

---

# 14. Những nơi KHÔNG nên đặt secret

Không đặt password trong:

```text
Git repository
application source code
README
shell scripts committed to Git
systemd unit committed/publicized
release ticket
screenshots
monitoring dashboard
email/chat
command-line argument
logs
```

Đặc biệt nguy hiểm:

```bash
java -Ddb.password=SuperSecret123 ...
```

vì command line có thể xuất hiện trong:

```bash
ps
/proc
diagnostics
process inventory
```

---

# 15. Environment variables có phải secret manager không?

Không.

Environment variables rất tiện cho **externalized configuration**, nhưng không nên đồng nhất:

```text
environment variable = secure secret store
```

Đây là hai thứ khác nhau.

Environment variables vẫn có thể bị:

- inherited bởi child processes;
- lộ qua debug dump;
- lộ qua application diagnostics;
- lộ nếu admin dùng sai tooling;
- vô tình log ra.

Vì vậy:

```text
env var = configuration transport mechanism
```

không phải:

```text
env var = full secrets-management solution
```

---

# 16. Baseline lab secret handling

Một mô hình tương đối an toàn cho lab:

```text
/etc/myapp/app.env
```

Ownership:

```text
root:tomcat
```

Permission:

```text
0640
```

Nghĩa là:

```text
root      read/write
tomcat    read
others    no access
```

Tạo directory:

```bash
sudo install -d \
  -o root \
  -g tomcat \
  -m 0750 \
  /etc/myapp
```

Tạo/edit file bằng:

```bash
sudoedit /etc/myapp/app.env
```

Ví dụ cấu trúc:

```text
APP_DB_URL=jdbc:postgresql://db.lab.internal:5432/appdb
APP_DB_USERNAME=appuser
APP_DB_PASSWORD=<enter-secret-interactively>
```

**Không đưa password thật vào screenshot hoặc evidence.**

Sau đó:

```bash
sudo chown root:tomcat /etc/myapp/app.env
sudo chmod 0640 /etc/myapp/app.env
```

Verify:

```bash
stat -c '%U %G %a %n' /etc/myapp/app.env
```

Expected invariant:

```text
root tomcat 640 /etc/myapp/app.env
```

---

# 17. Kết nối EnvironmentFile với systemd

Có thể tạo drop-in:

```bash
sudo systemctl edit tomcat
```

Ví dụ:

```ini
[Service]
EnvironmentFile=/etc/myapp/app.env
```

Sau đó:

```bash
sudo systemctl daemon-reload
```

Kiểm tra:

```bash
sudo systemctl cat tomcat
```

### Rất quan trọng

Tên environment variable mà application hiểu phụ thuộc application.

Ví dụ một **Spring Boot application** thường có thể sử dụng:

```text
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
```

nhưng syllabus **không nói application bắt buộc là Spring Boot**.

Vì vậy trước khi cấu hình phải kiểm tra:

```text
README
deployment documentation
application.properties
application.yml
existing environment
systemd unit
runbook
```

Không được tự đoán config key.

---

# 18. Supplementary — production-grade secret handling

Ngoài syllabus trực tiếp, production có thể dùng:

```text
AWS Secrets Manager
AWS Systems Manager Parameter Store
HashiCorp Vault
Kubernetes Secrets + external secret manager
systemd credentials
```

Điểm quan trọng là:

```text
Application lấy secret khi runtime
+
secret không nằm trong release artifact
+
access control riêng
+
audit
+
rotation
```

Nếu secret bị lộ, hành động đúng không phải chỉ là:

```text
rm app.env
```

mà phải:

```text
1. Revoke/rotate secret.
2. Determine exposure scope.
3. Update approved secret store.
4. Redeploy/restart safely.
5. Validate.
6. Preserve security evidence.
7. Escalate according to incident procedure.
```

---

# PHẦN III — TLS & CERTIFICATES

# 19. TLS giải quyết vấn đề gì?

TLS cung cấp ba thuộc tính chính:

```text
Confidentiality
Integrity
Authentication
```

### Confidentiality

Người nghe lén network không đọc được nội dung plaintext.

### Integrity

Traffic bị thay đổi sẽ bị phát hiện.

### Authentication

Client có thể kiểm tra server thực sự là server mong muốn thông qua certificate chain và hostname.

---

# 20. TLS KHÔNG giải quyết mọi security problem

HTTPS không có nghĩa application tự động an toàn.

TLS không:

- sửa SQL injection;
- sửa weak password;
- sửa authorization;
- encrypt database at rest;
- sửa XSS;
- ngăn app logging password;
- bảo vệ server đã bị compromise.

TLS chủ yếu bảo vệ **data in transit**.

---

# 21. Public key và private key

Một TLS server có:

```text
Private key
Public key
Certificate
```

Mental model:

```text
private key
    ^
    | SECRET
    |
certificate contains public key
    |
    | PUBLIC
    v
client
```

**Private key tuyệt đối không gửi cho client.**

---

# 22. Certificate chứa gì?

Một certificate thường chứa:

```text
Subject
Issuer
Public key
Serial number
Validity period
Signature
Subject Alternative Name (SAN)
Key usage
Extended key usage
```

Kiểm tra:

```bash
openssl x509 \
  -in app.crt \
  -noout \
  -subject \
  -issuer \
  -serial \
  -dates
```

Kiểm tra SAN:

```bash
openssl x509 \
  -in app.crt \
  -noout \
  -ext subjectAltName
```

Fingerprint:

```bash
openssl x509 \
  -in app.crt \
  -noout \
  -fingerprint \
  -sha256
```

---

# 23. CN và SAN

Ví dụ user truy cập:

```text
https://app.example.com
```

Certificate phải hợp lệ cho:

```text
app.example.com
```

Ngày nay hostname validation chủ yếu dựa vào:

```text
Subject Alternative Name (SAN)
```

Ví dụ:

```text
DNS:app.example.com
DNS:www.app.example.com
```

Nếu certificate chỉ hợp lệ cho:

```text
server01.internal
```

nhưng user truy cập:

```text
app.example.com
```

thì client đúng chuẩn phải báo:

```text
hostname mismatch
```

PostgreSQL `verify-full` cũng kiểm tra hostname theo certificate. [PostgreSQL](https://www.postgresql.org/docs/18/libpq-ssl.html?utm_source=chatgpt.com)

---

# 24. Certificate chain

Production thường:

```text
Root CA
   |
   v
Intermediate CA
   |
   v
Server certificate
```

Client đã trust Root CA.

Server gửi thường:

```text
Leaf certificate
+
Intermediate certificate(s)
```

Client xây chain:

```text
Leaf
 ↓
Intermediate
 ↓
Trusted Root
```

Nếu intermediate bị thiếu, browser/JVM/client có thể báo trust failure.

---

# 25. Truststore và keystore

Hai khái niệm rất hay nhầm.

## Keystore

Thường chứa:

```text
private key
+
certificate
```

Server dùng.

## Truststore

Thường chứa:

```text
trusted CA certificates
```

Client dùng để quyết định:

> Tôi tin certificate do CA nào ký?

Mental model:

```text
Server Keystore
    |
private key + certificate
    |
    v
TLS handshake
    |
    v
Client Truststore
trusted CA
```

---

# 26. Self-signed certificate

Self-signed:

```text
certificate được ký bằng chính private key tương ứng
```

Nó không có trust chain đến public CA.

Rất hữu ích cho:

```text
training
isolated lab
testing
development
```

nhưng **không phù hợp production Internet-facing**. Apache Tomcat cũng lưu ý self-signed certificates hữu ích cho testing nhưng không phù hợp production use. [Apache Tomcat](https://tomcat.apache.org/tomcat-11.0-doc/ssl-howto.html?utm_source=chatgpt.com)

---

# 27. TLS handshake — simplified

Một cách hiểu đủ cho fresher:

```text
Client
  |
  | ClientHello
  | supported TLS versions/ciphers
  v
Server
  |
  | ServerHello
  | Certificate
  v
Client

Client verifies:
  - certificate validity?
  - trusted issuer?
  - hostname matches?
  - certificate usable?

Then:
  key agreement
     ↓
  session keys established
     ↓
  encrypted application traffic
```

Nếu certificate verification fail:

```text
TLS connection should fail
```

Chứ không nên:

```text
"disable verification để cho chạy"
```

---

# PHẦN IV — BASIC TLS CONFIGURATION ON TOMCAT

# 28. Tomcat có thể terminate TLS trực tiếp

Architecture:

```text
Browser
   |
   | HTTPS :8443
   v
Tomcat
```

Apache Tomcat hỗ trợ TLS Connector trong `server.xml`; current Tomcat 10.1 documentation mô tả `SSLEnabled="true"` và certificate configuration qua `SSLHostConfig` / `Certificate`. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/ssl-howto.html?utm_source=chatgpt.com)

Đây sẽ là mô hình lab đơn giản của Day 14.

---

# 29. Nhưng production thường có reverse proxy

Architecture thường gặp hơn:

```text
Browser
   |
   | HTTPS :443
   v
Nginx / Apache / Load Balancer
   |
   | HTTP :8080
   | or HTTPS
   v
Tomcat
```

Tomcat documentation cũng lưu ý rằng khi Tomcat đứng sau một web server/reverse proxy, TLS thường được terminate ở web server phía trước. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/ssl-howto.html?utm_source=chatgpt.com)

Day 14 yêu cầu **reverse-proxy awareness**, không yêu cầu bắt buộc bạn phải build full reverse proxy stack.

---

# PHẦN V — REVERSE PROXY AWARENESS

# 30. Reverse proxy là gì?

Forward proxy:

```text
Client → Proxy → Internet
```

Reverse proxy:

```text
Internet → Proxy → Backend server
```

Ví dụ:

```text
client
   |
   v
nginx:443
   |
   v
Tomcat:8080
```

Client chỉ biết Nginx endpoint.

---

# 31. Reverse proxy thường làm gì?

Có thể xử lý:

- TLS termination;
- public port `443`;
- certificate lifecycle;
- load balancing;
- access logging;
- routing;
- compression;
- request limits;
- security headers;
- timeout control;
- backend isolation.

Tomcat không cần trực tiếp expose:

```text
8080
```

cho Internet.

---

# 32. Forwarded headers

Giả sử:

```text
Client IP = 203.0.113.100
```

Nginx kết nối Tomcat từ:

```text
127.0.0.1
```

Nếu không forward metadata, Tomcat có thể nghĩ client là:

```text
127.0.0.1
```

Reverse proxy thường forward:

```http
Host: app.example.com
X-Forwarded-For: 203.0.113.100
X-Forwarded-Proto: https
```

Một Nginx example:

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;

    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

`proxy_pass` và `proxy_set_header` là các directives chính thức của Nginx proxy module. [nginx.org](https://nginx.org/en/docs/http/ngx_http_proxy_module.html?utm_source=chatgpt.com)

### Supplementary / Beyond explicit syllabus

Configuration này chỉ minh họa reverse proxy awareness. Day 14 không bắt buộc triển khai Nginx.

---

# 33. Security issue với forwarded headers

Không được blind trust:

```text
X-Forwarded-For
X-Forwarded-Proto
```

Nếu client trực tiếp truy cập Tomcat, client có thể tự gửi:

```http
X-Forwarded-Proto: https
```

Do đó trust boundary phải là:

```text
Untrusted client
      |
      v
Trusted reverse proxy
      |
 overwrites forwarding headers
      |
      v
Backend trusts ONLY known proxy
```

Nếu cấu hình sai có thể dẫn đến:

- spoofed client IP;
- incorrect secure-cookie behavior;
- authentication/authorization assumptions sai;
- incorrect redirect URL.

---

# 34. Reverse proxy failure symptoms

### Redirect loop

```text
HTTPS client
 → proxy
 → HTTP Tomcat
 → app thinks connection HTTP
 → redirects HTTPS
 → repeat
```

Có thể do application không nhận đúng forwarded scheme.

---

### Wrong absolute URLs

Application sinh:

```text
http://localhost:8080/login
```

thay vì:

```text
https://app.example.com/login
```

Potential cause:

```text
Host / X-Forwarded-Proto handling
```

---

### Wrong client IP

Logs chỉ có:

```text
127.0.0.1
```

Potential cause:

```text
missing proxy headers
or backend not configured to trust proxy
```

---

# PHẦN VI — DEPLOYMENT VALIDATION

# 35. Deploy thành công ≠ release thành công

Đây là một trong các điểm quan trọng nhất Day 14.

Bạn chạy:

```bash
systemctl restart tomcat
```

và thấy:

```text
active (running)
```

Điều đó **không chứng minh release thành công**.

Nó chỉ chứng minh:

```text
systemd process đang chạy
```

Application vẫn có thể:

```text
HTTP 500
DB disconnected
wrong environment
wrong certificate
missing dependency
partial startup
broken business functionality
```

---

# 36. Release validation ladder

Một release nên được kiểm tra theo thứ tự:

```text
1. Baseline
2. Artifact
3. Configuration
4. Service
5. Socket
6. TLS
7. HTTP
8. Application health
9. Database functionality
10. Logs
11. Business smoke test
12. Evidence
```

---

# 37. Pre-deployment baseline

Trước thay đổi:

```bash
date -Is

systemctl status tomcat --no-pager

ss -ltnp

curl -fsS \
  http://127.0.0.1:8080/myapp/health
```

Capture Java:

```bash
java -version
```

Tomcat:

```bash
"$CATALINA_HOME/bin/version.sh"
```

Artifact checksum:

```bash
sha256sum "$CATALINA_BASE/webapps/myapp.war"
```

Mục tiêu:

> Có một known-good baseline để so sánh sau release và rollback.

---

# 38. Backup trước khi thay đổi

**Cảnh báo:** Đây là thay đổi ảnh hưởng service availability và application artifact. Thực hành trên lab/non-production trước.

Tạo rollback directory:

```bash
TS=$(date +%Y%m%d_%H%M%S)

sudo install \
  -d \
  -o root \
  -g root \
  -m 0700 \
  "/var/backups/myapp/$TS"
```

Backup WAR:

```bash
sudo cp -a \
  "$CATALINA_BASE/webapps/myapp.war" \
  "/var/backups/myapp/$TS/"
```

Backup server config:

```bash
sudo cp -a \
  "$CATALINA_BASE/conf/server.xml" \
  "/var/backups/myapp/$TS/"
```

Nếu backup secret config:

```bash
sudo cp -a \
  /etc/myapp/app.env \
  "/var/backups/myapp/$TS/"
```

Backup directory phải giữ permission chặt vì giờ nó cũng chứa secret.

---

# 39. Verify backup

Không chỉ copy rồi tin.

```bash
sudo ls -l "/var/backups/myapp/$TS"
```

Checksums:

```bash
sudo sha256sum \
  "$CATALINA_BASE/webapps/myapp.war" \
  "/var/backups/myapp/$TS/myapp.war"
```

Expected invariant:

```text
hai SHA-256 phải giống nhau
```

---

# PHẦN VII — GUIDED LAB DAY 14

# 40. Lab objective

Hoàn thành đủ syllabus assignment:

```text
1. Connect Java application → PostgreSQL
2. Handle DB secret safely enough for lab
3. Configure basic TLS on Tomcat
4. Deploy/restart safely
5. Validate release
6. Inject/observe failure
7. Roll back
8. Validate rollback
9. Produce evidence
```

---

# LAB PHASE A — DISCOVERY

## 41. Không đoán Tomcat paths

Đầu tiên:

```bash
sudo systemctl cat tomcat
```

Xem:

```text
User=
Group=
Environment=
EnvironmentFile=
ExecStart=
```

Process:

```bash
ps -ef | grep '[o]rg.apache.catalina'
```

Ports:

```bash
sudo ss -ltnp
```

Tìm:

```text
8080
8443
```

Java:

```bash
java -version
```

Tomcat:

```bash
"$CATALINA_HOME/bin/version.sh"
```

Nếu `$CATALINA_HOME` chưa có trong shell, xác định nó từ:

```text
systemd unit
process command line
existing documentation
```

---

# LAB PHASE B — BASELINE

## 42. Application baseline

```bash
curl -i \
  http://127.0.0.1:8080/myapp/health
```

Nếu app không có `/health`, dùng endpoint do application documentation chỉ định.

Expected invariant:

```text
HTTP success
application currently functional
```

Logs:

```bash
sudo journalctl \
  -u tomcat \
  --since '10 minutes ago' \
  --no-pager
```

Ghi nhận baseline.

---

# LAB PHASE C — DATABASE NETWORK VALIDATION

## 43. DNS

```bash
getent hosts db.lab.internal
```

Nếu fail:

```text
Do not continue by changing DB password.
```

Đó là sai troubleshooting layer.

Bạn đang ở DNS layer.

---

## 44. TCP

```bash
nc -vz db.lab.internal 5432
```

Nếu:

```text
Connection refused
```

thường khác với:

```text
Connection timed out
```

### Connection refused

Thường ám chỉ:

```text
host reachable
but target port rejects/no listener
```

Potential causes:

```text
PostgreSQL not running
not listening on expected address
wrong port
firewall actively rejects
```

### Timeout

Potential causes:

```text
routing
security group
NACL
firewall drop
wrong IP
network path
```

Không được coi hai lỗi này như nhau.

---

# LAB PHASE D — DATABASE CONFIGURATION

## 45. Create protected config directory

```bash
sudo install \
  -d \
  -o root \
  -g tomcat \
  -m 0750 \
  /etc/myapp
```

---

## 46. Create secret-bearing environment file

```bash
sudoedit /etc/myapp/app.env
```

**Ví dụ generic lab convention:**

```text
APP_DB_URL=jdbc:postgresql://db.lab.internal:5432/appdb
APP_DB_USERNAME=appuser
APP_DB_PASSWORD=<SECRET>
```

Nhắc lại:

> `APP_DB_*` là tên minh họa. Application thật phải dùng config keys mà app hỗ trợ.

Spring Boot example:

```text
SPRING_DATASOURCE_URL=jdbc:postgresql://db.lab.internal:5432/appdb
SPRING_DATASOURCE_USERNAME=appuser
SPRING_DATASOURCE_PASSWORD=<SECRET>
```

---

## 47. Apply permissions

```bash
sudo chown root:tomcat /etc/myapp/app.env
sudo chmod 0640 /etc/myapp/app.env
```

Verify:

```bash
stat -c '%U:%G %a %n' /etc/myapp/app.env
```

Expected invariant:

```text
root:tomcat 640
```

### Failure injection

Temporarily imagine permission:

```text
600 root:root
```

Tomcat user sẽ không đọc được.

Possible symptom:

```text
application starts with missing DB variable
or environment file load/service failure
```

Đây là connection problem nhìn giống application problem nhưng root cause là **filesystem permission**.

---

# LAB PHASE E — SYSTEMD CONFIGURATION

## 48. Add EnvironmentFile

```bash
sudo systemctl edit tomcat
```

Add:

```ini
[Service]
EnvironmentFile=/etc/myapp/app.env
```

Reload manager:

```bash
sudo systemctl daemon-reload
```

Inspect:

```bash
sudo systemctl cat tomcat
```

Không paste `app.env` vào evidence.

---

# LAB PHASE F — RESTART AND DATABASE CONNECTION TEST

## 49. Safe restart

Ghi timestamp:

```bash
CHANGE_TIME=$(date -Is)
echo "$CHANGE_TIME"
```

Restart:

```bash
sudo systemctl restart tomcat
```

Immediately verify:

```bash
sudo systemctl status tomcat --no-pager
```

Sau đó logs:

```bash
sudo journalctl \
  -u tomcat \
  --since "$CHANGE_TIME" \
  --no-pager
```

---

# 50. Tìm DB-related errors

Ví dụ:

```bash
sudo journalctl \
  -u tomcat \
  --since "$CHANGE_TIME" \
  --no-pager |
grep -Ei \
'jdbc|postgres|sqlstate|datasource|connection|exception|error'
```

Đừng chỉ grep `ERROR`.

Đọc surrounding context nếu phát hiện vấn đề.

---

# 51. Validate application

```bash
curl -i \
  http://127.0.0.1:8080/myapp/health
```

Nếu app có DB health check, tốt.

Nếu không, phải thực hiện một **approved functional operation có sử dụng database**, ví dụ:

```text
GET một record
create một disposable test record
read configuration persisted in DB
```

Không chạy destructive business transactions chỉ để test.

---

# 52. Database error taxonomy

## `password authentication failed`

Layer:

```text
PostgreSQL authentication
```

Kiểm tra:

```text
username
secret source
secret rotation/change
configuration precedence
```

Không cần đổi firewall.

---

## `database "... " does not exist`

Layer:

```text
DB target configuration
```

Kiểm tra:

```text
database name
environment
JDBC URL
```

---

## `no pg_hba.conf entry`

Application đã tới PostgreSQL.

Root cause nằm gần:

```text
PostgreSQL client authentication policy
```

Day 14 fresher nên:

```text
collect source IP
DB host
username
database
exact timestamp
error
```

sau đó follow approved procedure / escalate.

Không tự tiện mở `pg_hba.conf` rộng:

```text
0.0.0.0/0 trust
```

Đây là anti-pattern nghiêm trọng.

---

## `Connection refused`

Điều tra:

```text
DB process/listener
wrong host
wrong port
```

---

## Timeout

Điều tra:

```text
DNS
route
firewall
AWS SG
NACL
network ACL/path
```

---

## `PKIX path building failed`

Đây thường là Java TLS trust failure.

Investigate:

```text
CA trust
certificate chain
truststore
```

---

## Hostname verification failure

Ví dụ app connect:

```text
db.lab.internal
```

certificate SAN:

```text
postgres01.example.com
```

Nếu `verify-full`, connection phải fail.

Đó là **security feature hoạt động đúng**, không phải lý do để disable TLS verification.

---

# LAB PHASE G — TLS

# 53. Safety trước TLS change

Bạn sắp chỉnh:

```text
server.xml
private key
certificate
Tomcat port
```

Potential blast radius:

```text
Tomcat fails to start
HTTPS unavailable
private key exposed
wrong certificate
port conflict
```

Trước khi chỉnh:

```bash
sudo cp -a \
  "$CATALINA_BASE/conf/server.xml" \
  "$CATALINA_BASE/conf/server.xml.pre-day14"
```

Verify:

```bash
sudo ls -l \
  "$CATALINA_BASE/conf/server.xml" \
  "$CATALINA_BASE/conf/server.xml.pre-day14"
```

---

# 54. Create TLS directory

```bash
sudo install \
  -d \
  -o root \
  -g tomcat \
  -m 0750 \
  /etc/tomcat/tls
```

---

# 55. Generate lab self-signed certificate

**Lab/non-production only.**

```bash
sudo openssl req \
  -x509 \
  -newkey rsa:2048 \
  -sha256 \
  -days 30 \
  -nodes \
  -keyout /etc/tomcat/tls/app.key \
  -out /etc/tomcat/tls/app.crt \
  -subj "/CN=app.lab.local" \
  -addext "subjectAltName=DNS:app.lab.local,IP:127.0.0.1"
```

`-nodes` ở đây tạo private key không mã hóa bằng passphrase.

Điều đó có nghĩa:

> file permissions trở thành cực kỳ quan trọng.

Set:

```bash
sudo chown root:tomcat \
  /etc/tomcat/tls/app.key \
  /etc/tomcat/tls/app.crt
```

Private key:

```bash
sudo chmod 0640 /etc/tomcat/tls/app.key
```

Certificate:

```bash
sudo chmod 0644 /etc/tomcat/tls/app.crt
```

Certificate public có thể readable; private key phải restricted.

---

# 56. Inspect certificate

```bash
openssl x509 \
  -in /etc/tomcat/tls/app.crt \
  -noout \
  -subject \
  -issuer \
  -dates
```

SAN:

```bash
openssl x509 \
  -in /etc/tomcat/tls/app.crt \
  -noout \
  -ext subjectAltName
```

Expected:

```text
DNS:app.lab.local
IP Address:127.0.0.1
```

---

# 57. Configure Tomcat TLS Connector

Edit:

```bash
sudoedit "$CATALINA_BASE/conf/server.xml"
```

Một minimal current Tomcat-style connector:

```xml
<Connector
    port="8443"
    protocol="org.apache.coyote.http11.Http11NioProtocol"
    SSLEnabled="true"
    scheme="https"
    secure="true">

    <SSLHostConfig>
        <Certificate
            certificateFile="/etc/tomcat/tls/app.crt"
            certificateKeyFile="/etc/tomcat/tls/app.key"
            type="RSA" />
    </SSLHostConfig>

</Connector>
```

Current Tomcat documentation hỗ trợ certificate/key file hoặc keystore-based configuration. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/ssl-howto.html?utm_source=chatgpt.com)

### Important

Tomcat version khác nhau có thể có chi tiết config khác.

Luôn xác nhận:

```bash
"$CATALINA_HOME/bin/version.sh"
```

và đối chiếu documentation đúng major/minor version.

---

# 58. Config test trước restart

Nếu installation hỗ trợ:

```bash
sudo -u tomcat \
  "$CATALINA_HOME/bin/catalina.sh" \
  configtest
```

Expected:

```text
configuration parses successfully
```

Nếu fail:

```text
DO NOT restart.
```

Đây là nguyên tắc operations:

```text
validate before activate
```

---

# 59. Restart

```bash
CHANGE_TIME=$(date -Is)

sudo systemctl restart tomcat
```

Status:

```bash
sudo systemctl status tomcat --no-pager
```

Socket:

```bash
sudo ss -ltnp | grep ':8443'
```

Expected invariant:

```text
Tomcat listening on TCP 8443
```

---

# 60. Initial TLS diagnostic

```bash
curl -vk \
  https://127.0.0.1:8443/myapp/health
```

`-k` bỏ qua certificate verification.

Vì vậy:

```text
curl -k succeeds
```

chỉ chứng minh đại khái:

```text
TLS endpoint reachable
HTTP application responds
```

**Nó không chứng minh certificate trust/hostname validation đúng.**

Không sử dụng `-k` làm final acceptance evidence.

---

# 61. Proper TLS validation

Vì lab certificate self-signed, ta explicitly trust certificate cho test:

```bash
curl \
  --cacert /etc/tomcat/tls/app.crt \
  --resolve app.lab.local:8443:127.0.0.1 \
  https://app.lab.local:8443/myapp/health
```

Lúc này client xác minh:

```text
trust
+
hostname
+
TLS
+
HTTP response
```

Đây là evidence mạnh hơn rất nhiều.

---

# 62. Inspect live TLS handshake

```bash
openssl s_client \
  -connect 127.0.0.1:8443 \
  -servername app.lab.local \
  </dev/null
```

Xem:

```text
subject
issuer
certificate chain
protocol
cipher
verification
```

Không cần thuộc toàn bộ output.

Bạn cần biết đọc các phần quan trọng.

---

# PHẦN VIII — SUPPLEMENTARY DB TLS

# 63. Secure application → PostgreSQL

Phần này vượt tối thiểu lab, nhưng rất hữu ích.

Architecture:

```text
Tomcat
  |
  | JDBC + TLS
  v
PostgreSQL
```

JDBC URL:

```text
jdbc:postgresql://db.lab.internal:5432/appdb?sslmode=verify-full&sslrootcert=/etc/myapp/certs/postgres-ca.crt
```

`verify-full` thực hiện server certificate verification và hostname verification. [jdbc.postgresql.org](https://jdbc.postgresql.org/documentation/use/?utm_source=chatgpt.com)

Create cert directory:

```bash
sudo install \
  -d \
  -o root \
  -g tomcat \
  -m 0750 \
  /etc/myapp/certs
```

CA certificate:

```text
/etc/myapp/certs/postgres-ca.crt
```

Permission:

```bash
sudo chown root:tomcat /etc/myapp/certs/postgres-ca.crt
sudo chmod 0644 /etc/myapp/certs/postgres-ca.crt
```

CA certificate không phải private key.

---

# PHẦN IX — RELEASE VALIDATION

# 64. Validation matrix

Một Day 14 release tốt nên có bảng evidence tương tự:

| Check          | Command/evidence        | Pass condition          |
| -------------- | ----------------------- | ----------------------- |
| Artifact       | `sha256sum`             | Approved hash/version   |
| systemd        | `systemctl status`      | `active (running)`      |
| HTTP socket    | `ss -ltnp`              | Expected port listening |
| HTTPS socket   | `ss -ltnp`              | `8443` listening        |
| HTTP health    | `curl`                  | Expected 2xx            |
| TLS trust      | `curl --cacert ...`     | No certificate error    |
| Certificate    | `openssl x509`          | Correct SAN/validity    |
| Database       | App DB-backed operation | Successful              |
| Logs           | `journalctl`            | No new fatal errors     |
| Secrets        | `stat`                  | Restricted permissions  |
| Business smoke | Approved function       | Correct result          |

---

# 65. What is an Expected Invariant?

Ví dụ:

```bash
systemctl status tomcat
```

Example output có thể khác theo distro/version.

Không nên học thuộc:

```text
Active: active (running) since ...
Main PID: 12345
```

Điều cần nhớ là invariant:

```text
unit loaded
active/running
no crash/restart loop
expected main process
```

Đây là cách học operations đúng.

---

# 66. Log validation

Sau release:

```bash
sudo journalctl \
  -u tomcat \
  --since "$CHANGE_TIME" \
  --no-pager
```

Search:

```bash
sudo journalctl \
  -u tomcat \
  --since "$CHANGE_TIME" \
  --no-pager |
grep -Ei \
'exception|error|failed|fatal|sqlstate|jdbc|ssl|certificate'
```

Nhưng:

```text
grep no output ≠ application definitely healthy
```

Bạn vẫn phải health check và functional test.

---

# 67. Release validation ≠ chỉ positive test

Bạn cũng cần negative/security assertions.

Ví dụ:

```text
Private key world-readable?       NO
Secret visible in ticket?         NO
Unexpected port exposed?          NO
TLS hostname invalid?             NO
Old release accidentally running? NO
```

---

# PHẦN X — ROLLBACK

# 68. Rollback là gì?

Rollback không đơn giản là:

```text
copy old WAR
```

Rollback nghĩa là:

> đưa system trở lại **known-good state** đã được xác minh.

State có thể gồm:

```text
application artifact
application configuration
environment
TLS configuration
database compatibility
systemd configuration
```

---

# 69. Khi nào rollback?

Typical triggers:

```text
Application does not start
Health endpoint fails
HTTP 5xx spike
Database connection broken
TLS endpoint broken
Critical regression
Major functional smoke test fails
Security issue introduced
```

Không tiếp tục sửa production vô hạn trong khi outage kéo dài nếu approved rollback đã tồn tại.

---

# 70. Rollback decision mental model

```text
Release failure
     |
     v
Can L1 safely correct immediately
within approved change?
     |
   yes ──> fix → validate
     |
    no
     v
Rollback available?
     |
   yes ──> rollback → validate
     |
    no
     v
Escalate + protect evidence
```

---

# LAB PHASE H — SIMULATED FAILED RELEASE

# 71. Failure injection

Trong lab, ta cố tình tạo lỗi an toàn.

Ví dụ chỉnh DB port thành:

```text
5433
```

trong app configuration.

**Không làm trên production.**

Restart:

```bash
CHANGE_TIME=$(date -Is)

sudo systemctl restart tomcat
```

Observe:

```bash
sudo journalctl \
  -u tomcat \
  --since "$CHANGE_TIME" \
  --no-pager
```

Bạn có thể thấy:

```text
connection refused
timeout
datasource initialization failure
```

tùy application behavior.

Health test:

```bash
curl -i \
  http://127.0.0.1:8080/myapp/health
```

Expected:

```text
release acceptance fails
```

---

# 72. Evidence trước rollback

Record:

```bash
date -Is
sudo systemctl status tomcat --no-pager
sudo ss -ltnp
```

Logs:

```bash
sudo journalctl \
  -u tomcat \
  --since "$CHANGE_TIME" \
  --no-pager
```

Evidence cần trả lời:

```text
What changed?
When?
What failed?
What evidence supports it?
What is rollback target?
```

---

# 73. Execute rollback

Giả sử:

```text
/var/backups/myapp/$TS/
```

là known-good backup.

Stop service:

```bash
sudo systemctl stop tomcat
```

Verify:

```bash
sudo systemctl is-active tomcat
```

Expected:

```text
inactive
```

Restore environment:

```bash
sudo cp -a \
  "/var/backups/myapp/$TS/app.env" \
  /etc/myapp/app.env
```

Nếu artifact thay đổi:

```bash
sudo cp -a \
  "/var/backups/myapp/$TS/myapp.war" \
  "$CATALINA_BASE/webapps/myapp.war"
```

Nếu TLS/Tomcat configuration là phần failed release:

```bash
sudo cp -a \
  "/var/backups/myapp/$TS/server.xml" \
  "$CATALINA_BASE/conf/server.xml"
```

Validate config nếu applicable:

```bash
sudo -u tomcat \
  "$CATALINA_HOME/bin/catalina.sh" \
  configtest
```

Start:

```bash
ROLLBACK_TIME=$(date -Is)

sudo systemctl start tomcat
```

---

# 74. Rollback chưa hoàn thành cho đến khi validate

Check:

```bash
sudo systemctl status tomcat --no-pager
```

Sockets:

```bash
sudo ss -ltnp
```

Health:

```bash
curl -fsS \
  http://127.0.0.1:8080/myapp/health
```

HTTPS nếu expected:

```bash
curl \
  --cacert /etc/tomcat/tls/app.crt \
  --resolve app.lab.local:8443:127.0.0.1 \
  https://app.lab.local:8443/myapp/health
```

Logs:

```bash
sudo journalctl \
  -u tomcat \
  --since "$ROLLBACK_TIME" \
  --no-pager
```

Functional DB-backed operation:

```text
PASS
```

Chỉ sau đó mới kết luận:

```text
Rollback successful.
```

---

# 75. Tomcat exploded WAR caveat

Nếu Tomcat dùng:

```text
unpackWARs="true"
```

WAR có thể được explode thành:

```text
webapps/myapp/
```

Khi rollback WAR, generated exploded directory có thể gây confusion tùy deployment state/timestamps.

Trong lab có thể cần:

```text
stop Tomcat
verify directory is generated deployment content
restore WAR
remove ONLY corresponding generated exploded directory
start Tomcat
```

**Cảnh báo:** không chạy một lệnh như:

```bash
rm -rf webapps/*
```

Đó là destructive và blast radius quá lớn.

Phải biết chính xác directory nào do WAR sinh ra trước khi remove.

---

# 76. Database migrations làm rollback khó hơn

Suppose:

```text
Release v2
   +
DROP COLUMN customer.email
```

Sau đó bạn restore:

```text
v1.war
```

nhưng v1 vẫn cần:

```text
customer.email
```

Kết quả:

```text
application rollback fails
```

Đây gọi là:

```text
application/database compatibility problem
```

Day 14 reduced-scope lab **không nên dùng destructive DB migrations**.

Trong production, DB migrations phải có:

```text
backward compatibility strategy
backup/recovery plan
migration rollback/forward-fix plan
DBA approval where required
```

---

# PHẦN XI — TROUBLESHOOTING PLAYBOOK

# 77. Chuỗi troubleshooting chuẩn

Không troubleshooting kiểu random commands.

Dùng:

```text
Symptom
  ↓
Expected behavior
  ↓
Recent change
  ↓
Scope
  ↓
Failing layer
  ↓
Evidence
  ↓
Hypotheses
  ↓
Safest discriminating test
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

# 78. Scenario 1 — Application không connect DB

Symptom:

```text
Application HTTP 500
```

Recent change:

```text
DB configuration changed
```

Đừng nhảy ngay tới PostgreSQL config.

Kiểm tra:

```bash
sudo journalctl \
  -u tomcat \
  --since '10 minutes ago'
```

Giả sử log:

```text
Connection refused db.lab.internal:5432
```

Test DNS:

```bash
getent hosts db.lab.internal
```

Test TCP:

```bash
nc -vz db.lab.internal 5432
```

Nếu TCP refused:

```text
Failure layer = network/service listener
```

Không phải application code.

---

# 79. Scenario 2 — Authentication failure

Log:

```text
password authentication failed for user "appuser"
```

Evidence đã chứng minh:

```text
Network path works
PostgreSQL responds
Authentication rejected
```

Hypotheses:

```text
wrong password
wrong username
stale secret
secret rotated
application reading wrong config source
```

Check config source precedence.

Không paste password vào troubleshooting ticket.

---

# 80. Scenario 3 — TLS certificate hostname mismatch

Client:

```bash
curl \
  --cacert app.crt \
  https://127.0.0.1:8443/
```

Certificate:

```text
SAN=DNS:app.lab.local
```

Potential result:

```text
certificate subject name does not match target hostname
```

Đây là đúng.

Correct test:

```bash
curl \
  --cacert app.crt \
  --resolve app.lab.local:8443:127.0.0.1 \
  https://app.lab.local:8443/
```

Không sửa bằng:

```bash
curl -k
```

rồi tuyên bố production ready.

---

# 81. Scenario 4 — Tomcat fails after TLS change

Status:

```bash
sudo systemctl status tomcat
```

Then:

```bash
sudo journalctl \
  -u tomcat \
  -b \
  --no-pager
```

Potential causes:

```text
XML syntax error
certificate path wrong
private key unreadable
key/certificate mismatch
port already in use
unsupported configuration
```

Check certificate files:

```bash
sudo stat \
  /etc/tomcat/tls/app.crt \
  /etc/tomcat/tls/app.key
```

Port:

```bash
sudo ss -ltnp | grep ':8443'
```

Config test:

```bash
sudo -u tomcat \
  "$CATALINA_HOME/bin/catalina.sh" \
  configtest
```

---

# 82. Scenario 5 — TLS private key permission failure

Suppose:

```text
app.key = root:root 0600
```

Tomcat runs:

```text
User=tomcat
```

Then Tomcat cannot read key.

Evidence:

```bash
sudo -u tomcat test -r /etc/tomcat/tls/app.key

echo $?
```

Expected if inaccessible:

```text
non-zero
```

Safe remediation under approved baseline:

```bash
sudo chown root:tomcat /etc/tomcat/tls/app.key
sudo chmod 0640 /etc/tomcat/tls/app.key
```

Validate again before restart.

---

# 83. Scenario 6 — Port conflict

Tomcat tries 8443 but another process owns it.

Check:

```bash
sudo ss -ltnp | grep ':8443'
```

Possible failure:

```text
Address already in use
```

Do **not** kill random PID.

First identify:

```bash
ps -fp <PID>
```

Find owner/service:

```bash
systemctl status <service>
```

Port conflict may be legitimate architecture.

Escalate/change design rather than arbitrarily killing production service.

---

# PHẦN XII — L1 BOUNDARIES

# 84. Những gì L1/fresher thường có thể làm nếu runbook cho phép

- inspect service status;
- inspect logs;
- check DNS;
- test TCP port;
- check file permissions;
- validate approved application configuration;
- restore approved known-good config;
- restart according to procedure;
- health check;
- run approved smoke test;
- collect release evidence;
- execute documented rollback;
- verify certificate metadata;
- report DB authentication/network symptoms.

---

# 85. Khi nào phải escalate?

Escalate khi cần:

- sửa production `pg_hba.conf` ngoài runbook;
- tạo/drop DB roles;
- reset shared production credentials;
- thay DB privileges không được phê duyệt;
- thay schema/data;
- mở rộng firewall/SG/NACL;
- disable TLS verification;
- issue/revoke production CA certificates;
- investigate private key compromise;
- resolve suspicious security event;
- recover corrupted database;
- handle unknown schema migration failure;
- troubleshoot DB saturation sâu;
- thay architecture không nằm trong approved change.

L1 không có nghĩa:

> "Không được làm gì."

L1 nghĩa:

> Làm đầy đủ những gì nằm trong **approved operational boundary**, và escalate với evidence tốt khi vượt boundary.

---

# PHẦN XIII — SECURITY & ANTI-PATTERNS

# 86. Anti-pattern: password trong JDBC URL

Ví dụ không nên:

```text
jdbc:postgresql://db/appdb?user=appuser&password=SuperSecret
```

Nó có thể xuất hiện trong:

```text
logs
config dumps
error output
process diagnostics
tickets
```

Giữ credentials separate nếu application hỗ trợ.

---

# 87. Anti-pattern: world-readable secret

Sai:

```bash
chmod 644 /etc/myapp/app.env
```

Ai trên server cũng có thể đọc.

Better:

```bash
chmod 640 /etc/myapp/app.env
chown root:tomcat /etc/myapp/app.env
```

---

# 88. Anti-pattern: private key permission 0644

Sai:

```text
-rw-r--r-- app.key
```

Private key không phải certificate.

Nó phải restrictive.

---

# 89. Anti-pattern: dùng `curl -k` làm acceptance test

`-k`:

```text
disable certificate verification
```

Nó hữu ích như diagnostic test nhưng không chứng minh TLS identity đúng.

---

# 90. Anti-pattern: expose Tomcat management/public ports không cần thiết

Production architecture thường không cần Internet trực tiếp tới:

```text
8080
8443
Tomcat Manager
```

Nếu reverse proxy tồn tại:

```text
public :443
backend :8080 restricted
```

là architecture thường hợp lý hơn.

---

# 91. Anti-pattern: restart-first troubleshooting

Symptom:

```text
DB authentication failed
```

Reaction:

```bash
systemctl restart tomcat
systemctl restart tomcat
systemctl restart tomcat
```

Không giải quyết root cause.

Nó còn:

- xóa một phần transient evidence;
- gây thêm downtime;
- tạo noise;
- có thể làm connection storm.

Evidence first.

---

# PHẦN XIV — RELEASE / ROLLBACK EVIDENCE

# 92. Evidence pack nên có gì?

Một evidence pack Day 14 tốt có:

```text
Change timestamp
Application version
Artifact SHA-256
Java version
Tomcat version
Service status
Port/listener evidence
Sanitized DB connectivity result
HTTP health result
TLS certificate metadata
HTTPS validation result
Sanitized logs
Functional smoke test
Rollback target
Rollback execution record
Post-rollback validation
Final state
```

Không chứa:

```text
password
access token
private key
full secret environment dump
```

---

# 93. Một mini handover record

Ví dụ:

```text
Change:
Deploy myapp v1.4.0 and enable HTTPS lab connector.

Baseline:
v1.3.0 healthy at 2026-10-12T...

Validation:
- tomcat.service active
- HTTP 8080 healthy
- HTTPS 8443 healthy
- TLS hostname validation passed
- PostgreSQL-backed operation passed
- no fatal startup errors

Failure:
v1.4.0 test configuration pointed to incorrect DB port.

Action:
Rollback initiated according to Day14 runbook.

Rollback:
Restored previous environment and application artifact.

Final state:
v1.3.0 healthy.
DB operation passed.
No remaining critical error.

Evidence:
journal excerpt, checksums, health output, TLS output.
```

Đây là operations mindset rất quan trọng cho Day 18 về change/release controls sau này.

---

# PHẦN XV — REDUCED-SCOPE CONSOLIDATION

# 94. Day 12 → Day 14 liên kết thế nào?

Ba ngày tạo thành một block.

```text
DAY 12
Java runtime
JAR/WAR
Tomcat
systemd
deployment
      |
      v
DAY 13
externalized config
profiles
ports
context path
JVM memory
logs
safe restart
      |
      v
DAY 14
database connectivity
secrets
TLS
reverse proxy
release validation
rollback
```

Mental model tổng hợp:

```text
                CLIENT
                   |
                HTTPS
                   |
                   v
        [Reverse Proxy]
                   |
             HTTP/HTTPS
                   |
                   v
+--------------------------------------+
| Linux                                |
|                                      |
| systemd                              |
|   ↓                                  |
| Tomcat                               |
|   ↓                                  |
| Java Application                     |
|   ↓                                  |
| External Configuration               |
|   ↓                                  |
| Secrets                              |
+--------------------------------------+
                   |
                  JDBC
                   |
               TCP 5432
                   |
                   v
             PostgreSQL
```

Nếu system fail:

```text
Client
 ↓
DNS
 ↓
Network
 ↓
Port/socket
 ↓
Tomcat
 ↓
Application config
 ↓
JDBC
 ↓
PostgreSQL authentication
 ↓
DB authorization
 ↓
SQL/data
```

Đây chính là integrated troubleshooting chain mà từ Day 15 trở đi chúng ta sẽ tiếp tục mở rộng.

---

# PHẦN XVI — OPERATIONS / RECOVERY

# 95. Certificate expiry

Certificates có:

```text
Not Before
Not After
```

Check:

```bash
openssl x509 \
  -in /etc/tomcat/tls/app.crt \
  -noout \
  -dates
```

Production cần monitor expiry.

Một service hoàn toàn healthy hôm nay có thể fail trong tương lai khi certificate hết hạn.

Certificate lifecycle là operational concern, không chỉ security concern.

---

# 96. Secret rotation

Một safe rotation pattern:

```text
Prepare new secret
        ↓
Update server/auth side
        ↓
Update approved secret store
        ↓
Restart/reload app safely
        ↓
Validate new connection
        ↓
Revoke old secret
        ↓
Audit/evidence
```

Thứ tự cụ thể phụ thuộc hệ thống.

Không tự ý rotate production DB password nếu multiple consumers đang dùng cùng credential.

Đó là lý do service accounts nên có ownership rõ ràng và least privilege.

---

# 97. Certificate/private-key mismatch

Có thể kiểm tra public keys tương ứng mà không in private key ra màn hình.

Certificate public key fingerprint:

```bash
openssl x509 \
  -in app.crt \
  -pubkey \
  -noout |
openssl pkey \
  -pubin \
  -outform DER |
sha256sum
```

Private key-derived public key:

```bash
openssl pkey \
  -in app.key \
  -pubout \
  -outform DER |
sha256sum
```

Hai hash phải giống nhau.

Nếu không:

```text
certificate and private key do not belong together
```

---

# PHẦN XVII — EXAM FOCUS

# 98. Theory bạn phải trả lời được

Bạn cần tự giải thích được:

1. JDBC nằm ở đâu trong Java → PostgreSQL flow?
2. `Connection refused` khác `timeout` thế nào?
3. Tại sao TCP 5432 open chưa chứng minh app kết nối DB được?
4. Secret khác configuration thường thế nào?
5. Certificate có phải secret không?
6. Private key có phải secret không?
7. TLS cung cấp gì?
8. `sslmode=require` khác `verify-full` ở điểm nào?
9. CA chain là gì?
10. SAN dùng để làm gì?
11. Keystore và truststore khác gì?
12. Reverse proxy là gì?
13. TLS termination nghĩa là gì?
14. Vì sao cần `X-Forwarded-Proto`?
15. Tại sao `systemctl active` chưa đủ để kết luận release tốt?
16. Release validation cần những layer nào?
17. Rollback khác restore WAR đơn thuần thế nào?
18. Tại sao database migrations có thể phá rollback?
19. Khi nào L1 nên escalate?
20. Evidence package không được chứa gì?

---

# 99. Practical exam skills

Bạn cần có thể tự tay:

```bash
systemctl status
systemctl restart
systemctl cat

journalctl -u

getent hosts

nc -vz

ss -ltnp

curl
curl --cacert
curl --resolve

openssl x509
openssl s_client

sha256sum
stat
chmod
chown
```

Và quan trọng hơn:

> Biết **tại sao** mình chạy command đó và command chứng minh điều gì.

---

# PHẦN XVIII — KNOWLEDGE CHECK

Hãy thử tự trả lời trước khi nhìn phần đáp án.

### Q1

App báo:

```text
password authentication failed
```

Bạn có cần kiểm tra Security Group đầu tiên không?

### Q2

Port 5432 connect được bằng `nc`. Có kết luận DB username/password đúng được không?

### Q3

Certificate:

```text
SAN=DNS:app.example.com
```

Client truy cập:

```text
https://10.0.1.20
```

`verify-full` nên pass hay fail?

### Q4

`curl -k https://...` thành công. Certificate có được xác minh không?

### Q5

`tomcat.service` active nhưng `/health` trả HTTP 500. Release có pass không?

### Q6

Password DB vô tình xuất hiện trong Git. Chỉ xóa commit hiện tại có đủ không?

### Q7

Tomcat không đọc được `/etc/tomcat/tls/app.key`. Bạn phải kiểm tra gì?

### Q8

Rollback WAR nhưng DB schema đã bị migration incompatible. Rollback có chắc thành công không?

---

# 100. Đáp án

### A1

Không.

PostgreSQL đã trả authentication failure nghĩa là ít nhất request đã tới được DB authentication layer.

Investigate:

```text
username/password/config source
```

trước.

---

### A2

Không.

`nc` chỉ chứng minh:

```text
TCP connectivity
```

không chứng minh:

```text
authentication
authorization
database existence
SQL functionality
```

---

### A3

Fail.

Hostname/IP không match SAN.

---

### A4

Không.

`-k` disable certificate verification.

---

### A5

Không.

Process running ≠ application healthy.

---

### A6

Không.

Credential phải được coi là compromised:

```text
rotate/revoke
investigate exposure
clean repository according to procedure
```

---

### A7

Kiểm tra:

```text
ownership
permissions
parent directory execute permission
service User/Group
path correctness
SELinux/AppArmor where applicable
```

---

### A8

Không.

Application rollback phụ thuộc database compatibility.

---

# PHẦN XIX — INDEPENDENT PRACTICAL CHALLENGE

Sau guided lab, bạn nên tự làm lại mà không nhìn lời giải.

## Challenge

Environment:

```text
Tomcat service running.
Provided PostgreSQL available.
Provided application WAR available.
Known-good v1 release exists.
Candidate v2 release exists.
```

Bạn phải:

1. Capture current baseline.
2. Determine actual Tomcat paths and service user.
3. Configure approved PostgreSQL connectivity.
4. Protect DB credentials.
5. Restart application safely.
6. Prove application actually uses PostgreSQL.
7. Generate a lab TLS certificate with correct SAN.
8. Configure HTTPS on Tomcat.
9. Prove certificate hostname validation works.
10. Collect release evidence.
11. Inject one safe DB connection failure.
12. Diagnose it evidence-first.
13. Restore known-good configuration.
14. Simulate a failed v2 release.
15. Execute rollback to v1.
16. Validate service, HTTPS, DB and business function after rollback.
17. Produce a short handover.

### Bonus failure injections

```text
Wrong DB hostname
Wrong DB port
Wrong DB password
Secret file unreadable
Certificate path wrong
Certificate SAN mismatch
Port 8443 already occupied
Malformed server.xml
```

Không inject destructive failures vào production.

---

# 101. Day 14 troubleshooting cheat sheet

| Symptom                                   | Likely layer            | First useful evidence           |
| ----------------------------------------- | ----------------------- | ------------------------------- |
| DB hostname not resolved                  | DNS                     | `getent hosts`                  |
| TCP timeout                               | Network                 | `nc`, routes/firewall           |
| Connection refused                        | Service/port            | TCP test + server listener      |
| Password rejected                         | DB authentication       | Application/Postgres error      |
| `pg_hba.conf` rejection                   | DB access policy        | Exact DB error                  |
| JDBC driver missing                       | Application dependency  | Tomcat logs/WAR contents        |
| `PKIX` error                              | TLS trust               | Java/TLS logs                   |
| Hostname mismatch                         | TLS identity            | certificate SAN                 |
| 8443 not listening                        | Tomcat/TLS config       | `systemctl`, `journalctl`, `ss` |
| TLS works with `-k` only                  | Trust/hostname          | `curl --cacert`                 |
| HTTP 500 with service active              | Application             | app logs + endpoint             |
| Rollback artifact restored but app broken | Config/DB compatibility | logs + current config/schema    |

---

# 102. Mastery checklist Day 14

Bạn chưa nên coi Day 14 hoàn thành chỉ vì đã đọc bài này.

### Theory mastery

Bạn có thể giải thích:

```text
JDBC flow
DB connectivity layers
secret lifecycle
TLS certificate chain
SAN
private key
trust
reverse proxy
TLS termination
release validation
rollback
```

### Practical mastery

Bạn đã thực sự:

```text
[ ] Connected Java application to PostgreSQL
[ ] Protected database credentials
[ ] Verified DNS/TCP separately
[ ] Verified app-level DB functionality
[ ] Generated/installed lab TLS certificate
[ ] Started Tomcat HTTPS connector
[ ] Validated TLS without -k
[ ] Inspected certificate/SAN
[ ] Collected release evidence
[ ] Injected a safe failure
[ ] Diagnosed failure from evidence
[ ] Executed rollback
[ ] Validated rollback
[ ] Produced sanitized evidence/handover
```

Nếu chưa thực hiện các bước trên, bạn đã **học lý thuyết** nhưng chưa đạt practical mastery theo mục tiêu curriculum.

---

# 103. Những điều quan trọng nhất phải nhớ

Day 14 có thể cô đọng thành chuỗi sau:

```text
Configuration
      ↓
Secrets
      ↓
Database Connectivity
      ↓
TLS
      ↓
Deployment
      ↓
Validation
      ↓
Evidence
      ↓
Rollback
```

Và tư duy troubleshooting:

```text
Don't guess.
Don't restart blindly.
Don't disable security to make an error disappear.

Observe
→ isolate the failing layer
→ gather evidence
→ perform the safest test
→ remediate within L1 scope
→ validate
→ rollback/escalate when necessary
```

Đối với PostgreSQL TLS, pgJDBC hiện hỗ trợ rõ các mode từ `disable` đến `verify-full`, trong đó `verify-full` kiểm tra cả trust chain và hostname; đây là điểm rất quan trọng cho tư duy production security. [jdbc.postgresql.org](https://jdbc.postgresql.org/documentation/use/?utm_source=chatgpt.com)

Đối với HTTPS, Tomcat hiện hỗ trợ TLS Connector trực tiếp, nhưng tài liệu chính thức cũng nhấn mạnh deployment sau reverse proxy/web server là một architecture phổ biến, nơi frontend xử lý TLS rồi forward request đến Tomcat. [Apache Tomcat](https://tomcat.apache.org/tomcat-10.1-doc/ssl-howto.html?utm_source=chatgpt.com)

**Day 14 chưa kết thúc ở “application chạy”. Mức đạt yêu cầu là:**

> **connect → secure → deploy → validate → troubleshoot → rollback → prove with evidence.**

Đây cũng là nền tảng trực tiếp để sang **Day 15 — PostgreSQL installation/provisioning, users/roles, database/schema, connectivity, `pg_hba.conf`, health checks và backup/restore**. 00_MASTER_SYLLABUS_Java_Fresher…
