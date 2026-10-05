Được. Với **Unit 4 trở đi**, ta sẽ đổi cách làm một chút: trước khi nghĩ đến “kịch bản trình bày 20 phút”, ta phải **chuẩn bị một hệ thống tích hợp thật sự dùng xuyên Unit 4 → Unit 5 → Unit 6**. Bạn hiện mới có infrastructure và middleware; chưa có Java application, database/schema/application role hay release artifact. Vì vậy tôi sẽ dựng toàn bộ phần đó từ đầu, giải thích cả source code, dependency được tải, cấu hình DB/TLS, build WAR, deploy, validation và rollback.

Theo guide, Unit 4 bắt buộc có: deploy Java lên Tomcat/systemd; externalized configuration + JVM settings + safe restart; PostgreSQL connectivity + TLS; deployment failure + rollback bằng OSI layer 4–7; và log/evidence. :chatgpt-content-reference{index="0"} Syllabus Day 12–14 cũng yêu cầu chính xác runtime/WAR/Tomcat/systemd → externalized configuration/JVM → DB/TLS/rollback. :chatgpt-content-reference{index="1"}

Điểm quan trọng là tôi sẽ không tạo một app “demo cho xong Unit 4”. App này sẽ tiếp tục được dùng như sau:

```text
Unit 4
Java WAR
  ↓
Tomcat
  ↓ JDBC + TLS
PostgreSQL

          ↓ giữ nguyên

Unit 5
PostgreSQL roles
backup/restore
session/lock/EXPLAIN
Prometheus scrape app /metrics
Grafana dashboard
alert fresher_app_db_up

          ↓ giữ nguyên

Unit 6
release 1.1.0
dependency checks
ticket
deployment
failure injection
rollback
handover
```

---

# A. Kiến trúc chung Unit 4 → 6

Sau khi Unit 4 hoàn thành, hệ thống sẽ là:

```text
Windows Browser
      |
      | TCP/8080
      v
AWS Security Group
      |
      v
EC2 Ubuntu
│
├── Tomcat 10 / systemd
│     |
│     └── fresher-app.war
│           |
│           ├── /health
│           ├── /info
│           ├── /db-check
│           └── /metrics
│                   |
│                   | JDBC
│                   | TLS
│                   v
├── PostgreSQL 16 :5432
│     └── fresherdb
│          └── schema app
│               └── demo_message
│
├── Prometheus :9090
│      ├── node_exporter :9100
│      └── [Unit 5] fresher-app /metrics
│
└── Grafana :3000
       └── [Unit 5] Prometheus datasource
```

Tomcat 10.1 hiện vẫn là supported branch, implement Jakarta Servlet 6.0 và yêu cầu Java 11+, vì vậy Java 17 hiện tại của lab phù hợp. Điều này cũng có nghĩa source phải dùng `jakarta.servlet.*`, không phải namespace Java EE cũ `javax.servlet.*`. :chatgpt-content-reference{index="2"}

---

# B. Một lưu ý về môi trường lab

Trong lab này ta sẽ **build source ngay trên EC2** vì bạn chỉ có một server.

Trong production thực tế, thường:

```text
Developer / Git
→ CI pipeline
→ build + test
→ artifact repository
→ approved WAR
→ deployment server
```

Production server thường **không phải nơi developer ngồi sửa source rồi chạy Maven**.

Đây là **Supplementary / Beyond explicit syllabus**, nhưng bạn nên nói được nếu trainer hỏi.

---

# C. Kiểm tra baseline trước khi thay đổi

Từ đây tất cả command đều chạy sau khi bạn SSH vào EC2:

```text
ubuntu@java-fresher-lab:~$
```

Không thao tác shell trên WSL.

Kiểm tra Java:

```bash
java -version
```

Expected:

```text
openjdk version "17..."
```

Tomcat:

```bash
systemctl is-active tomcat10
```

PostgreSQL:

```bash
systemctl is-active postgresql
```

Monitoring:

```bash
systemctl is-active prometheus
systemctl is-active prometheus-node-exporter
systemctl is-active grafana-server
```

Kiểm tra PostgreSQL cluster:

```bash
pg_lsclusters
```

Expected invariant:

```text
Version   16
Status    online
Port      5432
```

Ports:

```bash
sudo ss -lntp | grep -E ':8080|:5432|:9090|:9100|:3000'
```

Disk:

```bash
df -h /
```

RAM:

```bash
free -h
```

Đây là **BEFORE baseline**.

---

# D. Xác định Tomcat layout thực tế

Không học thuộc path rồi assume.

Chạy:

```bash
systemctl cat tomcat10
```

Xem Tomcat service.

Ubuntu package thực sự cung cấp:

````text
/usr/lib/systemd/system/tomcat10.service
/etc/tomcat10/
/var/lib/tomcat10/
``` :chatgpt-content-reference{index="3"}


Kiểm tra `appBase`:

```bash
sudo grep -n 'appBase=' /etc/tomcat10/server.xml
````

Bạn thường thấy:

```xml
appBase="webapps"
```

Kiểm tra:

```bash
sudo ls -ld /var/lib/tomcat10/webapps
```

Tomcat sử dụng WAR filename để suy ra context path nếu không cấu hình khác; ví dụ `fresher-app.war` tương ứng `/fresher-app`. :chatgpt-content-reference{index="4"}

---

# E. Cài Maven — vì hiện chưa có Java package

Kiểm tra:

```bash
command -v mvn
```

Nếu không có output:

```bash
sudo apt-get update
```

sau đó:

```bash
sudo apt-get install -y maven
```

Verify:

```bash
mvn -version
```

Bạn phải thấy:

```text
Apache Maven ...
Java version: 17...
```

## Maven dùng để làm gì?

Ta sẽ có:

```text
source .java
+
pom.xml
        ↓
      Maven
        ↓
compile
        ↓
resolve dependencies
        ↓
package
        ↓
fresher-app.war
```

Maven sẽ tự download:

```text
Jakarta Servlet API
PostgreSQL JDBC Driver
Maven build plugins
```

từ configured Maven repositories.

Ta **không cần curl JDBC JAR thủ công**.

---

# F. PostgreSQL JDBC version

pgJDBC official site hiện ghi current release là:

```text
42.7.13
```

và đây là JDBC driver phù hợp Java 8+ nên Java 17 hoàn toàn dùng được. :chatgpt-content-reference{index="5"}

Ta sẽ **pin**:

```xml
42.7.13
```

trong `pom.xml`, thay vì để build tự chọn version ngẫu nhiên.

---

# G. Tạo Java project từ đầu

Tạo:

```bash
mkdir -p \
  ~/fresher-app-src/src/main/java/com/example/fresher
```

Đi vào project:

```bash
cd ~/fresher-app-src
```

Kiểm tra:

```bash
pwd
```

Expected:

```text
/home/ubuntu/fresher-app-src
```

---

# H. Tạo `pom.xml`

Chạy:

```bash
cat > pom.xml <<'EOF'
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
           http://maven.apache.org/POM/4.0.0
           https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>fresher-app</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>

    <properties>
        <maven.compiler.release>17</maven.compiler.release>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>

        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>

        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <version>42.7.13</version>
        </dependency>

    </dependencies>

    <build>
        <finalName>fresher-app</finalName>

        <plugins>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.13.0</version>
            </plugin>

            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.4.0</version>
                <configuration>
                    <failOnMissingWebXml>false</failOnMissingWebXml>
                </configuration>
            </plugin>

        </plugins>
    </build>

</project>
EOF
```

---

# I. Giải thích `pom.xml`

```xml
<packaging>war</packaging>
```

Ta không build standalone JAR.

Ta build:

```text
WAR
```

để Tomcat deploy.

---

```xml
<maven.compiler.release>17</maven.compiler.release>
```

Compile target Java 17.

---

Dependency:

```xml
jakarta.servlet-api
```

là API để compile:

```java
HttpServlet
@WebServlet
HttpServletRequest
HttpServletResponse
```

Tomcat 10.1 implement Servlet 6.0 nên `jakarta.servlet-api:6.0.0` khớp runtime. :chatgpt-content-reference{index="6"}

---

`scope`:

```xml
<scope>provided</scope>
```

có nghĩa:

> cần Servlet API lúc compile nhưng không package nó vào WAR vì Tomcat đã cung cấp API này.

Nếu bundle một servlet implementation/API không cần thiết vào WAR, có thể tạo classloader/conflict problems.

---

PostgreSQL:

```xml
org.postgresql:postgresql:42.7.13
```

khác với Servlet API.

JDBC driver phải nằm trong application classpath nên nó **được package trong WAR**.

---

# J. Tạo `AppConfig.java`

```bash
cat > src/main/java/com/example/fresher/AppConfig.java <<'EOF'
package com.example.fresher;

public final class AppConfig {

    private AppConfig() {
    }

    public static String required(String name) {
        String value = System.getenv(name);

        if (value == null || value.isBlank()) {
            throw new IllegalStateException(
                "Missing required environment variable: " + name
            );
        }

        return value;
    }

    public static String optional(String name, String defaultValue) {
        String value = System.getenv(name);

        if (value == null || value.isBlank()) {
            return defaultValue;
        }

        return value;
    }
}
EOF
```

## Ý nghĩa

Ứng dụng **không hard-code**:

```text
database URL
database username
database password
environment name
application version
```

Source chỉ biết:

```java
System.getenv(...)
```

Đây chính là:

```text
externalized configuration
```

---

# K. Tạo `DbUtil.java`

```bash
cat > src/main/java/com/example/fresher/DbUtil.java <<'EOF'
package com.example.fresher;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

public final class DbUtil {

    private DbUtil() {
    }

    public record DbStatus(
        String database,
        String user,
        boolean tls,
        String tlsVersion,
        String cipher,
        String message
    ) {
    }

    public static Connection open() throws SQLException {
        DriverManager.setLoginTimeout(3);

        return DriverManager.getConnection(
            AppConfig.required("DB_URL"),
            AppConfig.required("DB_USER"),
            AppConfig.required("DB_PASSWORD")
        );
    }

    public static DbStatus check() throws SQLException {

        try (Connection connection = open()) {

            String database;
            String user;
            boolean tls;
            String tlsVersion;
            String cipher;

            String connectionSql = """
                SELECT
                    current_database(),
                    current_user,
                    ssl,
                    COALESCE(version, ''),
                    COALESCE(cipher, '')
                FROM pg_stat_ssl
                WHERE pid = pg_backend_pid()
                """;

            try (
                PreparedStatement statement =
                    connection.prepareStatement(connectionSql);

                ResultSet result = statement.executeQuery()
            ) {

                if (!result.next()) {
                    throw new SQLException(
                        "No pg_stat_ssl row for current backend"
                    );
                }

                database = result.getString(1);
                user = result.getString(2);
                tls = result.getBoolean(3);
                tlsVersion = result.getString(4);
                cipher = result.getString(5);
            }

            String message;

            try (
                PreparedStatement statement =
                    connection.prepareStatement(
                        "SELECT message " +
                        "FROM app.demo_message " +
                        "WHERE id = 1"
                    );

                ResultSet result = statement.executeQuery()
            ) {

                if (!result.next()) {
                    throw new SQLException(
                        "Application demo row was not found"
                    );
                }

                message = result.getString(1);
            }

            return new DbStatus(
                database,
                user,
                tls,
                tlsVersion,
                cipher,
                message
            );
        }
    }
}
EOF
```

Đây là class quan trọng nhất của ứng dụng.

Flow:

```text
Tomcat request
→ Java servlet
→ DbUtil.open()
→ JDBC driver
→ 127.0.0.1:5432
→ PostgreSQL
```

pgJDBC là Type-4 driver nói native PostgreSQL wire protocol. :chatgpt-content-reference{index="7"}

---

# L. Tại sao dùng `try (...)`?

Ví dụ:

```java
try (Connection connection = open())
```

là:

```text
try-with-resources
```

Sau block:

```text
Connection
PreparedStatement
ResultSet
```

được close tự động.

Điều này cực kỳ quan trọng với DB resources.

---

# M. Tạo Health endpoint

```bash
cat > src/main/java/com/example/fresher/HealthServlet.java <<'EOF'
package com.example.fresher;

import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

@WebServlet("/health")
public class HealthServlet extends HttpServlet {

    @Override
    protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response
    ) throws IOException {

        response.setStatus(HttpServletResponse.SC_OK);
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");

        response.getWriter().println("status=UP");
    }
}
EOF
```

`@WebServlet("/health")` khai báo URL mapping. Jakarta Servlet spec cho phép annotation này trên servlet class. :chatgpt-content-reference{index="8"}

---

# N. Tại sao health endpoint không kiểm tra DB?

Có chủ ý.

Ta muốn phân biệt:

```text
/health
```

= Java/Tomcat application có chạy không?

với:

```text
/db-check
```

= application có kết nối DB được không?

Nếu `/health` cũng phụ thuộc DB thì khi DB down:

```text
app healthy?
database healthy?
```

sẽ bị trộn lại với nhau.

Đây rất hữu ích cho troubleshooting Unit 5–6.

---

# O. Tạo `/info`

```bash
cat > src/main/java/com/example/fresher/InfoServlet.java <<'EOF'
package com.example.fresher;

import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

@WebServlet("/info")
public class InfoServlet extends HttpServlet {

    @Override
    protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response
    ) throws IOException {

        response.setStatus(HttpServletResponse.SC_OK);
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");

        response.getWriter().println(
            "environment=" +
            AppConfig.optional("APP_ENV", "unknown")
        );

        response.getWriter().println(
            "version=" +
            AppConfig.optional("APP_VERSION", "unknown")
        );

        response.getWriter().println(
            "java=" +
            System.getProperty("java.version")
        );
    }
}
EOF
```

Endpoint này giúp Unit 6 chứng minh:

```text
release version
environment
Java runtime
```

---

# P. Tạo `/db-check`

```bash
cat > src/main/java/com/example/fresher/DbCheckServlet.java <<'EOF'
package com.example.fresher;

import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.sql.SQLException;

@WebServlet("/db-check")
public class DbCheckServlet extends HttpServlet {

    @Override
    protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response
    ) throws IOException {

        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");

        try {

            DbUtil.DbStatus status = DbUtil.check();

            response.setStatus(HttpServletResponse.SC_OK);

            response.getWriter().println("db=UP");
            response.getWriter().println(
                "database=" + status.database()
            );
            response.getWriter().println(
                "user=" + status.user()
            );
            response.getWriter().println(
                "tls=" + status.tls()
            );
            response.getWriter().println(
                "tls_version=" + status.tlsVersion()
            );
            response.getWriter().println(
                "cipher=" + status.cipher()
            );
            response.getWriter().println(
                "message=" + status.message()
            );

        } catch (SQLException | IllegalStateException exception) {

            response.setStatus(
                HttpServletResponse.SC_SERVICE_UNAVAILABLE
            );

            response.getWriter().println("db=DOWN");

            if (exception instanceof SQLException sqlException) {
                response.getWriter().println(
                    "sql_state=" +
                    String.valueOf(sqlException.getSQLState())
                );
            }
        }
    }
}
EOF
```

Ta cố tình **không gửi stack trace/password/JDBC URL đầy đủ ra client**.

---

# Q. Tạo endpoint `/metrics`

Đây là phần tôi thêm để **Unit 5 nối ngay vào Prometheus/Grafana**, không phải cài app khác.

```bash
cat > src/main/java/com/example/fresher/MetricsServlet.java <<'EOF'
package com.example.fresher;

import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

@WebServlet("/metrics")
public class MetricsServlet extends HttpServlet {

    @Override
    protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response
    ) throws IOException {

        response.setStatus(HttpServletResponse.SC_OK);
        response.setContentType(
            "text/plain; version=0.0.4; charset=utf-8"
        );

        int dbUp = 0;
        int dbTls = 0;

        try {
            DbUtil.DbStatus status = DbUtil.check();

            dbUp = 1;
            dbTls = status.tls() ? 1 : 0;

        } catch (Exception ignored) {
            dbUp = 0;
            dbTls = 0;
        }

        response.getWriter().println(
            "# HELP fresher_app_up Java application availability."
        );

        response.getWriter().println(
            "# TYPE fresher_app_up gauge"
        );

        response.getWriter().println(
            "fresher_app_up 1"
        );

        response.getWriter().println(
            "# HELP fresher_app_db_up PostgreSQL connectivity."
        );

        response.getWriter().println(
            "# TYPE fresher_app_db_up gauge"
        );

        response.getWriter().println(
            "fresher_app_db_up " + dbUp
        );

        response.getWriter().println(
            "# HELP fresher_app_db_tls PostgreSQL TLS state."
        );

        response.getWriter().println(
            "# TYPE fresher_app_db_tls gauge"
        );

        response.getWriter().println(
            "fresher_app_db_tls " + dbTls
        );
    }
}
EOF
```

Sau Unit 4:

```text
fresher_app_up 1
fresher_app_db_up 1
fresher_app_db_tls 1
```

Sang Unit 5, ta chỉ cần configure Prometheus scrape endpoint này.

---

# R. Kiểm tra source tree

```bash
find . -type f | sort
```

Expected:

```text
./pom.xml
./src/main/java/com/example/fresher/AppConfig.java
./src/main/java/com/example/fresher/DbCheckServlet.java
./src/main/java/com/example/fresher/DbUtil.java
./src/main/java/com/example/fresher/HealthServlet.java
./src/main/java/com/example/fresher/InfoServlet.java
./src/main/java/com/example/fresher/MetricsServlet.java
```

---

# S. Build lần đầu

```bash
mvn -B clean package
```

`-B`:

```text
batch mode
```

phù hợp automation/log output.

Maven lần đầu sẽ tải dependencies nên có thể mất một lúc.

Nếu Maven download fail:

```text
đừng sửa Maven ngay
```

Kiểm tra:

```bash
ip route
```

```bash
ping -c 2 8.8.8.8
```

```bash
getent hosts repo.maven.apache.org
```

```bash
curl -I https://repo.maven.apache.org
```

theo OSI/network mindset.

---

# T. Build thành công

Expected:

```text
BUILD SUCCESS
```

Kiểm tra:

```bash
ls -lh target/fresher-app.war
```

Hash artifact:

```bash
sha256sum target/fresher-app.war
```

Artifact hash rất hữu ích cho release/evidence Unit 6.

---

# U. Kiểm tra WAR

```bash
jar tf target/fresher-app.war | head -30
```

Tìm application classes:

```bash
jar tf target/fresher-app.war \
  | grep 'com/example/fresher'
```

Tìm JDBC driver:

```bash
jar tf target/fresher-app.war \
  | grep 'WEB-INF/lib/postgresql'
```

Phải thấy JAR PostgreSQL.

Servlet API **không** nên nằm trong WAR vì `scope=provided`.

---

# V. Chuẩn bị PostgreSQL cho application

Đây là phần shared prerequisite Unit 4 → 5.

Unit 5 sau đó sẽ đi sâu hơn về:

```text
role
pg_hba
backup
restore
session
lock
EXPLAIN
```

Ở Unit 4 ta chỉ tạo **application DB baseline** đủ để Java app hoạt động.

---

# W. Inspect trước

```bash
sudo -u postgres psql -tAc \
  "SELECT rolname FROM pg_roles WHERE rolname='fresher_app';"
```

và:

```bash
sudo -u postgres psql -tAc \
  "SELECT datname FROM pg_database WHERE datname='fresherdb';"
```

Lần đầu expected:

```text
no output
```

---

# X. Tạo password nhưng không hiển thị ra history

Tạo random 48-hex-character password:

```bash
DB_PASS="$(openssl rand -hex 24)"
```

**Không chạy:**

```bash
echo "$DB_PASS"
```

Không chụp password.

Không paste password vào command history.

---

# Y. Tạo application role/database

Chạy:

```bash
sudo -u postgres psql \
  -v ON_ERROR_STOP=1 <<SQL

CREATE ROLE fresher_app
    LOGIN
    PASSWORD '${DB_PASS}'
    NOSUPERUSER
    NOCREATEDB
    NOCREATEROLE
    NOREPLICATION;

CREATE DATABASE fresherdb
    OWNER postgres;

REVOKE ALL
ON DATABASE fresherdb
FROM PUBLIC;

GRANT CONNECT
ON DATABASE fresherdb
TO fresher_app;

SQL
```

## Tại sao database owner không phải `fresher_app`?

Có chủ ý.

Nếu:

```text
fresher_app = database owner
```

thì application có quá nhiều quyền.

Ta muốn:

```text
postgres
= administrative owner

fresher_app
= runtime identity
```

---

# Z. Tạo schema/table

```bash
sudo -u postgres psql \
  -d fresherdb \
  -v ON_ERROR_STOP=1 <<'SQL'

CREATE SCHEMA app
AUTHORIZATION postgres;

REVOKE ALL
ON SCHEMA app
FROM PUBLIC;

CREATE TABLE app.demo_message (
    id integer PRIMARY KEY,
    message text NOT NULL
);

INSERT INTO app.demo_message (
    id,
    message
)
VALUES (
    1,
    'Java Fresher integrated lab database is healthy'
);

REVOKE ALL
ON ALL TABLES IN SCHEMA app
FROM PUBLIC;

GRANT USAGE
ON SCHEMA app
TO fresher_app;

GRANT SELECT
ON app.demo_message
TO fresher_app;

SQL
```

Ứng dụng chỉ cần:

```text
CONNECT
USAGE schema
SELECT table
```

Không cần:

```text
SUPERUSER
CREATEDB
CREATEROLE
CREATE TABLE
DROP TABLE
```

Đây chính là least privilege.

---

# AA. Test quyền application

Test SELECT:

```bash
PGPASSWORD="$DB_PASS" \
psql \
  "host=127.0.0.1 port=5432 dbname=fresherdb user=fresher_app" \
  -c 'SELECT * FROM app.demo_message;'
```

Expected success.

Bây giờ cố CREATE table:

```bash
PGPASSWORD="$DB_PASS" \
psql \
  "host=127.0.0.1 port=5432 dbname=fresherdb user=fresher_app" \
  -c 'CREATE TABLE app.should_fail(id integer);'
```

Expected:

```text
permission denied
```

Đây là **expected failure**.

Nó chứng minh least privilege hoạt động.

---

# AB. Kiểm tra PostgreSQL TLS trước khi chỉnh gì

Đừng assume.

```bash
sudo -u postgres psql \
  -Atc 'SHOW ssl;'
```

Sau đó:

```bash
sudo -u postgres psql \
  -Atc 'SHOW ssl_cert_file;'
```

```bash
sudo -u postgres psql \
  -Atc 'SHOW ssl_key_file;'
```

Nếu:

```text
ssl = on
```

thì **không cần sửa PostgreSQL TLS server config**.

Đây là phương án ưu tiên: không thay đổi thứ đang hoạt động mà không cần thiết.

---

# AC. Test TLS bằng `psql`

```bash
PGPASSWORD="$DB_PASS" \
psql \
  "host=127.0.0.1 port=5432 dbname=fresherdb user=fresher_app sslmode=require" \
  -c '\conninfo'
```

Sau đó:

```bash
PGPASSWORD="$DB_PASS" \
psql \
  "host=127.0.0.1 port=5432 dbname=fresherdb user=fresher_app sslmode=require" \
  -c \
  "SELECT ssl, version, cipher
   FROM pg_stat_ssl
   WHERE pid = pg_backend_pid();"
```

Ta muốn:

```text
ssl = t
```

và TLS version/cipher.

PostgreSQL hỗ trợ TLS trên cùng database TCP port; server và client negotiate encrypted connection. :chatgpt-content-reference{index="9"}

---

# AD. `sslmode=require` thực sự có ý nghĩa gì?

JDBC URL của ta sẽ dùng:

```text
sslmode=require
```

Điều này yêu cầu encrypted connection.

Nhưng rất quan trọng:

> `require` mã hóa connection nhưng **không xác minh đầy đủ identity của server certificate**.

pgJDBC docs chỉ rõ `verify-full` mới thực hiện certificate validation + hostname verification, và được khuyến nghị cho security-sensitive environments. :chatgpt-content-reference{index="10"}

Cho syllabus:

```text
basic TLS configuration
```

ta dùng:

```text
sslmode=require
+
verify pg_stat_ssl.ssl = true
```

là đủ tốt cho lab.

Trong production nên hướng tới:

```text
verify-full
+
trusted CA
+
valid server certificate
```

---

# AE. Nếu `SHOW ssl` trả `off`

**Chỉ làm block này nếu SSL thật sự đang off.**

Đây là fallback, không phải bước bắt buộc nếu SSL đã on.

Backup trạng thái trước:

```bash
PGDATA="$(
  sudo -u postgres psql \
    -Atc 'SHOW data_directory;'
)"
```

```bash
echo "$PGDATA"
```

Backup auto config:

```bash
sudo cp -a \
  "$PGDATA/postgresql.auto.conf" \
  "$PGDATA/postgresql.auto.conf.bak.$(date +%Y%m%d_%H%M%S)"
```

Tạo TLS directory:

```bash
sudo install \
  -d \
  -o postgres \
  -g postgres \
  -m 700 \
  /etc/postgresql/tls
```

Generate certificate:

```bash
sudo openssl req \
  -new \
  -x509 \
  -newkey rsa:3072 \
  -sha256 \
  -nodes \
  -days 365 \
  -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1" \
  -keyout /etc/postgresql/tls/server.key \
  -out /etc/postgresql/tls/server.crt
```

Ownership:

```bash
sudo chown \
  postgres:postgres \
  /etc/postgresql/tls/server.key \
  /etc/postgresql/tls/server.crt
```

Key permission:

```bash
sudo chmod 600 \
  /etc/postgresql/tls/server.key
```

Certificate:

```bash
sudo chmod 644 \
  /etc/postgresql/tls/server.crt
```

PostgreSQL official docs yêu cầu private key permission phải hạn chế chặt, chẳng hạn `0600`. :chatgpt-content-reference{index="11"}

Configure:

```bash
sudo -u postgres psql <<'SQL'

ALTER SYSTEM SET ssl = 'on';

ALTER SYSTEM SET ssl_cert_file =
    '/etc/postgresql/tls/server.crt';

ALTER SYSTEM SET ssl_key_file =
    '/etc/postgresql/tls/server.key';

SQL
```

Restart:

```bash
sudo systemctl restart postgresql
```

Validate:

```bash
systemctl is-active postgresql
```

```bash
pg_isready
```

Rồi chạy lại TLS test ở trên.

---

# AF. Externalized application configuration

Bây giờ Java source không có credential.

Ta tạo:

```text
/etc/java-fresher/fresher-app.env
```

Tạo directory:

```bash
sudo install \
  -d \
  -o root \
  -g root \
  -m 700 \
  /etc/java-fresher
```

Tạo env file mà **không in password ra terminal**:

```bash
{
    printf 'APP_ENV=demo\n'
    printf 'APP_VERSION=1.0.0\n'
    printf 'DB_URL=jdbc:postgresql://127.0.0.1:5432/fresherdb?sslmode=require&connectTimeout=3&socketTimeout=5\n'
    printf 'DB_USER=fresher_app\n'
    printf 'DB_PASSWORD=%s\n' "$DB_PASS"
} | sudo tee \
    /etc/java-fresher/fresher-app.env \
    >/dev/null
```

Set ownership:

```bash
sudo chown root:root \
  /etc/java-fresher/fresher-app.env
```

Permission:

```bash
sudo chmod 600 \
  /etc/java-fresher/fresher-app.env
```

Bây giờ xóa password khỏi shell variable:

```bash
unset DB_PASS
```

---

# AG. Không `cat` env file

Không chạy:

```bash
sudo cat /etc/java-fresher/fresher-app.env
```

trong recording.

Nó chứa password.

Thay vào đó:

```bash
sudo stat \
  -c '%a %U %G %n' \
  /etc/java-fresher/fresher-app.env
```

Expected:

```text
600 root root ...
```

Xem **key names mà không hiện values**:

```bash
sudo cut \
  -d= \
  -f1 \
  /etc/java-fresher/fresher-app.env
```

Expected:

```text
APP_ENV
APP_VERSION
DB_URL
DB_USER
DB_PASSWORD
```

---

# AH. JVM memory settings

EC2 của ta là `t3.medium`, khoảng 4 GiB RAM.

Nó còn chạy:

```text
PostgreSQL
Prometheus
Grafana
node_exporter
OS
```

nên không cấp toàn bộ 4 GiB cho Java.

Ta dùng:

```text
-Xms256m
-Xmx768m
```

`Xms`:

```text
initial Java heap
```

`Xmx`:

```text
maximum Java heap
```

---

# AI. Tạo systemd drop-in cho Tomcat

Trước tiên:

```bash
systemctl show \
  -p User \
  -p Group \
  tomcat10
```

Sau đó:

```bash
sudo mkdir -p \
  /etc/systemd/system/tomcat10.service.d
```

Tạo:

```bash
sudo tee \
  /etc/systemd/system/tomcat10.service.d/fresher-app.conf \
  >/dev/null <<'EOF'

[Service]

EnvironmentFile=/etc/java-fresher/fresher-app.env

Environment="JAVA_OPTS=-Djava.awt.headless=true -Xms256m -Xmx768m -XX:+UseG1GC -Dfile.encoding=UTF-8"

EOF
```

Bây giờ:

```text
application config
=
/etc/java-fresher/fresher-app.env

JVM config
=
systemd drop-in
```

Source code không thay đổi khi environment/password/heap thay đổi.

---

# AJ. Validate trước restart

```bash
sudo systemctl daemon-reload
```

Kiểm tra dependency DB:

```bash
pg_isready
```

Expected:

```text
accepting connections
```

Tomcat trước restart:

```bash
systemctl is-active tomcat10
```

Memory:

```bash
free -h
```

Port:

```bash
sudo ss -lntp | grep ':8080'
```

Đây là **safe restart pre-check**.

---

# AK. Restart Tomcat có kiểm soát

```bash
sudo systemctl restart tomcat10
```

Ngay lập tức:

```bash
RC=$?
echo "restart_exit_code=$RC"
```

Expected:

```text
restart_exit_code=0
```

Check:

```bash
systemctl is-active tomcat10
```

```bash
systemctl status tomcat10 --no-pager
```

Port:

```bash
sudo ss -lntp | grep ':8080'
```

Logs:

```bash
sudo journalctl \
  -u tomcat10 \
  --since '-3 minutes' \
  --no-pager
```

---

# AL. Verify JVM options thật sự apply

Lấy PID:

```bash
TOMCAT_PID="$(
  systemctl show \
    -p MainPID \
    --value \
    tomcat10
)"
```

```bash
echo "$TOMCAT_PID"
```

Xem command line:

```bash
ps -p "$TOMCAT_PID" -o args=
```

Tìm:

```text
-Xms256m
-Xmx768m
-XX:+UseG1GC
```

Đừng chỉ nói:

> "Em đã sửa file."

Phải chứng minh JVM đang chạy với settings đó.

---

# AM. Chuẩn bị release artifact

Ta giữ artifact theo version để Unit 6 tiếp tục dùng.

```bash
sudo install \
  -d \
  -o root \
  -g root \
  -m 755 \
  /opt/java-fresher/releases/fresher-app/1.0.0
```

Copy WAR:

```bash
sudo install \
  -o root \
  -g root \
  -m 644 \
  ~/fresher-app-src/target/fresher-app.war \
  /opt/java-fresher/releases/fresher-app/1.0.0/fresher-app.war
```

Hash:

```bash
sha256sum \
  /opt/java-fresher/releases/fresher-app/1.0.0/fresher-app.war
```

Ta sẽ giữ:

```text
/opt/java-fresher/releases/
└── fresher-app/
    └── 1.0.0/
        └── fresher-app.war
```

Unit 6 sau này sẽ có:

```text
1.0.0
1.1.0
```

rất tiện cho rollback.

---

# AN. Xác định Tomcat service account

```bash
TOMCAT_USER="$(
  systemctl show \
    -p User \
    --value \
    tomcat10
)"
```

```bash
echo "$TOMCAT_USER"
```

Lấy primary group:

```bash
TOMCAT_GROUP="$(
  id -gn "$TOMCAT_USER"
)"
```

```bash
echo "$TOMCAT_GROUP"
```

Không hard-code trước khi inspect.

---

# AO. Initial deployment

Trước:

```bash
systemctl is-active tomcat10
```

Stop:

```bash
sudo systemctl stop tomcat10
```

Check:

```bash
systemctl is-active tomcat10
```

Expected:

```text
inactive
```

Nếu đây là first deploy, kiểm tra:

```bash
sudo ls -la /var/lib/tomcat10/webapps
```

Nếu không có `fresher-app` thì tiếp tục.

Copy WAR:

```bash
sudo install \
  -o root \
  -g "$TOMCAT_GROUP" \
  -m 640 \
  /opt/java-fresher/releases/fresher-app/1.0.0/fresher-app.war \
  /var/lib/tomcat10/webapps/fresher-app.war
```

Start:

```bash
sudo systemctl start tomcat10
```

Capture:

```bash
RC=$?
echo "start_exit_code=$RC"
```

Expected:

```text
0
```

---

# AP. Validate deployment

Service:

```bash
systemctl is-active tomcat10
```

Socket:

```bash
sudo ss -lntp | grep ':8080'
```

Check WAR/exploded app:

```bash
sudo ls -ld \
  /var/lib/tomcat10/webapps/fresher-app*
```

Tomcat có thể unpack WAR thành:

```text
fresher-app/
```

vì WAR đặt trong appBase. :chatgpt-content-reference{index="12"}

---

# AQ. `/health`

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/health
```

Expected:

```text
HTTP/1.1 200

status=UP
```

---

# AR. `/info`

```bash
curl -s \
  http://127.0.0.1:8080/fresher-app/info
```

Expected:

```text
environment=demo
version=1.0.0
java=17...
```

Ta vừa chứng minh external configuration được inject.

---

# AS. `/db-check`

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/db-check
```

Expected invariant:

```text
HTTP 200

db=UP
database=fresherdb
user=fresher_app
tls=true
tls_version=...
cipher=...
message=Java Fresher integrated lab database is healthy
```

Đây là end-to-end:

```text
HTTP
→ Tomcat
→ Java
→ JDBC
→ TLS
→ PostgreSQL
→ query
→ response
```

---

# AT. `/metrics`

```bash
curl -s \
  http://127.0.0.1:8080/fresher-app/metrics
```

Expected:

```text
# HELP fresher_app_up ...
# TYPE fresher_app_up gauge
fresher_app_up 1

# HELP fresher_app_db_up ...
# TYPE fresher_app_db_up gauge
fresher_app_db_up 1

# HELP fresher_app_db_tls ...
# TYPE fresher_app_db_tls gauge
fresher_app_db_tls 1
```

**Đừng chỉnh Prometheus ngay.**

Việc đó để sang Unit 5.

---

# AU. Test từ Windows browser

Do Security Group Unit 3 đã restore TCP/8080 `/32`, trên Windows:

```text
http://EC2_PUBLIC_IP:8080/fresher-app/health
```

```text
http://EC2_PUBLIC_IP:8080/fresher-app/info
```

```text
http://EC2_PUBLIC_IP:8080/fresher-app/db-check
```

Phải hoạt động từ IP được phép.

---

# AV. Logs

Tomcat service logs:

```bash
sudo journalctl \
  -u tomcat10 \
  -n 100 \
  --no-pager
```

Có thể inspect directory:

```bash
sudo ls -lah /var/log/tomcat10
```

Nhưng path/file cụ thể có thể tùy package configuration.

Khi troubleshooting, `journalctl -u tomcat10` là lựa chọn an toàn theo systemd service.

---

# AW. Deployment failure injection

Đây là phần bắt buộc của Unit 4.

Ta **không phá source tốt**.

Release tốt vẫn nằm:

```text
/opt/java-fresher/releases/fresher-app/1.0.0/fresher-app.war
```

---

# AX. BEFORE failure

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/health
```

Expected:

```text
status=UP
```

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/db-check
```

Expected DB UP.

Hash good release:

```bash
sha256sum \
  /opt/java-fresher/releases/fresher-app/1.0.0/fresher-app.war
```

---

# AY. Tạo artifact lỗi có chủ ý

```bash
printf '%s\n' \
  'INTENTIONALLY BROKEN WAR FOR UNIT 4 FAILURE INJECTION' \
  > /tmp/fresher-app-bad.war
```

Check:

```bash
file /tmp/fresher-app-bad.war
```

Đây **không phải valid ZIP/WAR**.

---

# AZ. Backup deployed WAR

```bash
sudo cp -a \
  /var/lib/tomcat10/webapps/fresher-app.war \
  /opt/java-fresher/releases/fresher-app/fresher-app.pre-failure.war
```

Verify:

```bash
sudo ls -lh \
  /opt/java-fresher/releases/fresher-app/
```

---

# BA. Inject bad deployment

Stop:

```bash
sudo systemctl stop tomcat10
```

Trước khi xóa exploded directory, inspect:

```bash
sudo find \
  /var/lib/tomcat10/webapps \
  -maxdepth 1 \
  -name 'fresher-app*' \
  -print
```

Expected chỉ:

```text
/var/lib/tomcat10/webapps/fresher-app
/var/lib/tomcat10/webapps/fresher-app.war
```

**Cảnh báo:** command sau là destructive với deployed copy, nhưng release artifact tốt đã được giữ ở `/opt/java-fresher/releases/...`.

Xóa chính xác exploded app:

```bash
sudo rm -rf -- \
  /var/lib/tomcat10/webapps/fresher-app
```

Deploy bad WAR:

```bash
sudo install \
  -o root \
  -g "$TOMCAT_GROUP" \
  -m 640 \
  /tmp/fresher-app-bad.war \
  /var/lib/tomcat10/webapps/fresher-app.war
```

Start:

```bash
sudo systemctl start tomcat10
```

---

# BB. Không rollback ngay

Bây giờ symptom xảy ra.

Đừng thấy lỗi rồi copy WAR tốt ngay.

Phải troubleshoot.

Guide yêu cầu OSI Mindset layer 4–7 trước rollback. :chatgpt-content-reference{index="13"}

---

# BC. Layer 4 — Transport

Check:

```bash
sudo ss -lntp | grep ':8080'
```

Nếu thấy:

```text
*:8080
```

thì Tomcat port vẫn listening.

Layer 4:

```text
OK
```

Không phải Security Group/port problem.

---

# BD. Layer 5 — Session/service

```bash
systemctl is-active tomcat10
```

Expected có thể:

```text
active
```

Tomcat process bản thân vẫn chạy.

Check root context:

```bash
curl -I \
  http://127.0.0.1:8080/
```

Nếu Tomcat trả HTTP response:

```text
Tomcat HTTP service alive
```

---

# BE. Layer 6

Endpoint của Tomcat hiện là:

```text
HTTP
```

không phải HTTPS.

Cho nên TLS presentation layer tại frontend:

```text
not applicable to this specific HTTP request
```

Nhưng DB TLS đã được kiểm tra riêng qua:

```text
/db-check
→ pg_stat_ssl
```

Không cần giả vờ “Layer 6 OK” khi HTTP không sử dụng TLS.

Đây là cách trả lời chính xác hơn.

---

# BF. Layer 7 — Application

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/health
```

Expected:

```text
404
```

hoặc context unavailable.

Port hoạt động nhưng application context không hoạt động.

Bây giờ đọc logs:

```bash
sudo journalctl \
  -u tomcat10 \
  --since '-5 minutes' \
  --no-pager
```

Nếu cần tìm:

```bash
sudo grep -RniE \
  'error|exception|war|deploy|zip' \
  /var/log/tomcat10 \
  2>/dev/null \
  | tail -50
```

Ta muốn evidence liên quan:

```text
invalid WAR
ZIP
deployment failure
```

---

# BG. Kết luận failure

Bạn phải nói được:

```text
Layer 4:
port 8080 listening

Layer 5:
Tomcat service active

Layer 6:
frontend request uses HTTP, TLS not applicable

Layer 7:
application context unavailable

Recent change:
WAR artifact replaced

Log:
WAR deployment/ZIP failure

Root cause:
invalid deployment artifact

Correct action:
rollback artifact

Incorrect action:
restart EC2
edit Security Group
restart PostgreSQL
```

Đây chính là evidence-driven troubleshooting.

---

# BH. Rollback

Stop Tomcat:

```bash
sudo systemctl stop tomcat10
```

Remove failed exploded directory nếu có:

```bash
sudo rm -rf -- \
  /var/lib/tomcat10/webapps/fresher-app
```

Restore known-good release:

```bash
sudo install \
  -o root \
  -g "$TOMCAT_GROUP" \
  -m 640 \
  /opt/java-fresher/releases/fresher-app/1.0.0/fresher-app.war \
  /var/lib/tomcat10/webapps/fresher-app.war
```

Start:

```bash
sudo systemctl start tomcat10
```

---

# BI. Rollback validation

Service:

```bash
systemctl is-active tomcat10
```

Transport:

```bash
sudo ss -lntp | grep ':8080'
```

Application:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/health
```

DB:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/db-check
```

Metrics:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/metrics
```

Final invariant:

```text
health = UP
db = UP
tls = true
fresher_app_up = 1
fresher_app_db_up = 1
fresher_app_db_tls = 1
```

---

# BJ. Architecture bây giờ đã sẵn sàng cho Unit 5

Ta **không cleanup** những thành phần sau:

```text
fresherdb
fresher_app role
app.demo_message
fresher-app.war
1.0.0 release
/etc/java-fresher/fresher-app.env
Tomcat drop-in
/metrics endpoint
```

Vì Unit 5 sẽ dùng thẳng chúng.

Unit 5 sẽ tiếp tục:

```text
1. PostgreSQL
   ├── inspect fresher_app
   ├── create thêm fresher_readonly
   ├── pg_hba.conf
   ├── enforce/inspect TLS rules
   ├── pg_dump
   └── restore

2. Diagnostics
   ├── sessions
   ├── locks
   ├── slow query
   └── EXPLAIN

3. Prometheus
   └── scrape
       127.0.0.1:8080/fresher-app/metrics

4. Grafana
   └── dashboard

5. Alert
   fresher_app_db_up == 0

6. Failure injection
   stop PostgreSQL
       ↓
   app /health = UP
   app /db-check = DOWN
   metric fresher_app_db_up = 0
       ↓
   Prometheus alert
       ↓
   Grafana evidence
```

Đây là một failure scenario rất mạnh vì nó chứng minh được:

```text
Tomcat vẫn chạy
Java app vẫn chạy
port 8080 vẫn chạy
nhưng DB dependency bị lỗi
```

nên bạn có thể khoanh vùng chính xác.

---

# BK. Unit 6 sau đó sẽ dùng tiếp chính application này

Ta sẽ tạo:

```text
version 1.1.0
```

thay vì application khác.

Flow Unit 6 dự kiến:

```text
Change ticket
        ↓
pre-check
        ↓
dependency check
        ↓
build fresher-app 1.1.0
        ↓
hash artifact
        ↓
backup 1.0.0
        ↓
deploy 1.1.0
        ↓
health / DB / monitoring validation
        ↓
failure injection
        ↓
rollback 1.0.0 if required
        ↓
handover note
```

Nhờ vậy Unit 4 → 5 → 6 thực sự là **một integrated scenario**, không phải ba bài rời nhau.

---

# BL. Lưu `history.log` sau chuẩn bị Unit 4

Cuối buổi chuẩn bị:

```bash
history -a
```

Master:

```bash
history > \
  ~/java-fresher-demo/history/history.log
```

Snapshot:

```bash
history > \
  ~/java-fresher-demo/history/"$(date +%F)_unit04_java.log"
```

Không để password trong history — cách chúng ta dùng `DB_PASS="$(openssl rand ...)"` chỉ lưu command tạo password, **không lưu giá trị password thực tế**.

---

# BM. Kịch bản 20 phút trong buổi thi

Các bước A→BL phía trên là **preparation**. Bạn không nên ngồi gõ toàn bộ Java source trong 20 phút recording.

Đến ngày demo, source/WAR/DB phải đã có sẵn.

Flow trình bày Unit 4 nên khoảng:

### 0–3 phút — architecture + WAR

```bash
java -version
```

```bash
mvn -version
```

```bash
ls -lh \
  /opt/java-fresher/releases/fresher-app/1.0.0/
```

```bash
jar tf \
  /opt/java-fresher/releases/fresher-app/1.0.0/fresher-app.war \
  | head
```

Giải thích:

```text
Java 17
Tomcat
WAR
Jakarta Servlet
JDBC driver
```

---

### 3–7 phút — externalized configuration + JVM

Không show secret.

```bash
sudo stat \
  -c '%a %U %G %n' \
  /etc/java-fresher/fresher-app.env
```

```bash
sudo cut \
  -d= \
  -f1 \
  /etc/java-fresher/fresher-app.env
```

```bash
sudo systemctl cat tomcat10
```

Sau đó:

```bash
sudo systemctl restart tomcat10
```

```bash
systemctl is-active tomcat10
```

```bash
sudo ss -lntp | grep ':8080'
```

Giải thích:

```text
APP_ENV
APP_VERSION
DB_URL
DB_USER
DB_PASSWORD
Xms
Xmx
```

nhưng không đọc secret.

---

### 7–10 phút — endpoints

```bash
curl -s \
  http://127.0.0.1:8080/fresher-app/health
```

```bash
curl -s \
  http://127.0.0.1:8080/fresher-app/info
```

```bash
curl -s \
  http://127.0.0.1:8080/fresher-app/db-check
```

Chỉ rõ:

```text
db=UP
user=fresher_app
tls=true
```

---

### 10–12 phút — TLS

```bash
sudo -u postgres psql \
  -d fresherdb \
  -c \
  "SELECT ssl, version, cipher
   FROM pg_stat_ssl;"
```

Giải thích:

```text
client requires TLS
pg_stat_ssl proves encryption
sslmode=require ≠ server identity verification
verify-full stronger
```

---

### 12–18 phút — deployment failure

Inject bad WAR.

Sau đó:

```bash
systemctl is-active tomcat10
```

```bash
sudo ss -lntp | grep ':8080'
```

```bash
curl -I http://127.0.0.1:8080/
```

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/health
```

```bash
sudo journalctl \
  -u tomcat10 \
  --since '-5 minutes' \
  --no-pager
```

Nói rõ:

```text
L4 OK
L5 OK
L7 fail
recent change = WAR
root cause = deployment artifact
```

---

### 18–20 phút — rollback

Restore:

```text
1.0.0
```

Validate:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/health
```

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/db-check
```

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/metrics
```

Kết luận:

```text
service restored
DB restored
TLS active
monitoring endpoint ready
```

---

## Trạng thái cần giữ nguyên sau Unit 4

**Không xóa gì sau đây:**

```text
/home/ubuntu/fresher-app-src

/opt/java-fresher/releases/
└── fresher-app/
    └── 1.0.0/
        └── fresher-app.war

/etc/java-fresher/
└── fresher-app.env

/etc/systemd/system/
└── tomcat10.service.d/
    └── fresher-app.conf

PostgreSQL:
fresherdb
fresher_app
app.demo_message

Tomcat:
fresher-app.war
```

Hệ thống của chúng ta lúc này đã trở thành một lab tích hợp thực sự:

```text
AWS
→ EC2/Linux
→ systemd
→ Java 17
→ Tomcat
→ WAR
→ externalized config
→ JDBC
→ TLS
→ PostgreSQL
→ /metrics
→ ready for Prometheus/Grafana
```

Vì vậy **Unit 5 sẽ không phải dựng lại PostgreSQL/application từ đầu**. Ta sẽ tiếp tục trực tiếp trên `fresherdb + fresher_app + /metrics`: tạo thêm least-privilege DB role để demo, cấu hình/giải thích `pg_hba.conf`, thực hiện `pg_dump` → restore, dựng slow-query/lock scenario, sau đó đưa chính `fresher_app_db_up` vào Prometheus/Grafana và tạo alert. Đến Unit 6 lại dùng chính WAR `1.0.0` này làm baseline để thực hiện release `1.1.0` + ticket + dependency check + rollback + handover.
