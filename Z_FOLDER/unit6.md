Được. Unit 6 sẽ là phần **khép vòng toàn bộ lab**, không tạo thêm một hệ thống riêng. Ta lấy nguyên kết quả của Unit 4–5 làm baseline, thực hiện một release thật từ `fresher-app 1.0.0 → 1.1.0`, có change ticket, pre-check, dependency check, Git/versioning, deployment, validation, failure injection, rollback, handover và acceptance.

Đúng theo guide, Unit 6 phải có: approved batch/release scenario với dependency/exit-code, ticket/evidence/rollback/handover; trình bày integrated Linux + AWS + Java + PostgreSQL + Monitoring; một failure injection dùng OSI Mindset và L1/escalation boundary; cuối cùng acceptance/evidence package. :chatgpt-content-reference{index="0"} Đây cũng chính là phạm vi Day 18–21 của syllabus. :chatgpt-content-reference{index="1"}

Sau **Self Q&A**, ta sẽ thêm bước **stop/cleanup AWS resources** đúng yêu cầu của guide, vì tài liệu nhấn mạnh phải dừng resource sau demo để tránh chi phí ngoài dự kiến. :chatgpt-content-reference{index="2"}

# 1. Trạng thái đầu Unit 6

Hệ thống hiện tại:

```text
AWS
└── VPC
    └── EC2 Ubuntu
         │
         ├── Tomcat 10 :8080
         │    └── fresher-app 1.0.0
         │         ├── /health
         │         ├── /info
         │         ├── /db-check
         │         └── /metrics
         │
         ├── PostgreSQL 16 :5432
         │    └── fresherdb
         │
         ├── Prometheus :9090
         ├── node_exporter :9100
         ├── Grafana :3000
         └── webhook receiver :9088
```

Healthy invariant:

```text
Tomcat                  active
PostgreSQL              active
Prometheus              active
node_exporter           active
Grafana                 active

/health                 UP
/db-check               UP
DB TLS                  true
fresher_app_up           1
fresher_app_db_up        1
Grafana alert            Normal
```

Unit 6 sẽ thay đổi:

```text
fresher-app 1.0.0
          ↓
fresher-app 1.1.0
```

---

# 2. Git trong Unit 6

Guide liệt kê Git trong toolset. Vì source ở Unit 4 chưa được quản lý bằng Git, ta bắt đầu quản lý nó ở đây.

**Docker Compose:** guide cũng liệt kê Docker Compose, nhưng baseline của chúng ta đã được dựng bằng native `systemd + Tomcat + PostgreSQL + Prometheus + Grafana`. Chuyển toàn bộ sang containers ngay trước final demo sẽ thay đổi architecture đã validate và tạo thêm blast radius không cần thiết. Vì vậy scenario này dùng Git, còn Docker Compose chỉ được nêu là một phương án packaging/orchestration khác chứ không dùng để thay baseline trong Unit 6.

---

# 3. Kiểm tra Git

Trên EC2:

```bash
command -v git
```

Nếu chưa có:

```bash
sudo apt-get update
sudo apt-get install -y git
```

Verify:

```bash
git --version
```

---

# 4. Đưa source Unit 4 vào Git

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

Tạo `.gitignore`:

```bash
cat > .gitignore <<'EOF'
target/
*.log
EOF
```

Mục đích:

```text
target/
```

là build output, không phải source.

Không nên commit WAR/binary build output vào repository source chỉ vì Maven sinh ra nó.

---

# 5. Initialize repository

```bash
git init
```

Cấu hình identity **chỉ trong repository lab này**:

```bash
git config user.name "Java Fresher Lab"
git config user.email "java-fresher-lab@localhost"
```

Đây chỉ là local Git metadata, không phải account/email thật của bạn.

Verify:

```bash
git config --local --list
```

---

# 6. Commit baseline `1.0.0`

Kiểm tra `pom.xml`:

```bash
grep -m1 '<version>' pom.xml
```

Expected:

```xml
<version>1.0.0</version>
```

Stage:

```bash
git add .
```

Inspect:

```bash
git status
```

Commit:

```bash
git commit \
  -m "Baseline fresher-app 1.0.0"
```

Tag:

```bash
git tag -a v1.0.0 \
  -m "Known-good fresher-app 1.0.0"
```

Verify:

```bash
git log \
  --oneline \
  --decorate \
  -5
```

và:

```bash
git tag
```

Ta đã có:

```text
source baseline
    ↓
Git commit
    ↓
v1.0.0
```

---

# 7. Tạo change/release workspace

Ta dùng change ID:

```bash
CHANGE_ID="CHG-UNIT6-001"
```

Tạo:

```bash
sudo install \
  -d \
  -o ubuntu \
  -g ubuntu \
  -m 750 \
  "/opt/java-fresher/ops/$CHANGE_ID"
```

Variable:

```bash
CHANGE_DIR="/opt/java-fresher/ops/$CHANGE_ID"
```

Check:

```bash
echo "$CHANGE_DIR"
```

---

# 8. Tạo ticket

Đây là **LAB-SIMULATED approval**, không được giả vờ là production approval thật.

```bash
cat > "$CHANGE_DIR/change-ticket.txt" <<EOF
Change ID: $CHANGE_ID
Created: $(date -Is)

Environment:
Java Fresher AWS integrated lab

Change type:
Application release

Current version:
1.0.0

Target version:
1.1.0

Scope:
- Build fresher-app 1.1.0
- Deploy WAR to existing Tomcat
- Update APP_VERSION
- Validate Java, PostgreSQL, TLS and monitoring

Expected impact:
Short Tomcat restart during deployment

Approval:
LAB-SIMULATED-APPROVED-FOR-DEMO

Pre-check:
- Tomcat active
- PostgreSQL ready
- Prometheus active
- Grafana active
- Application health UP
- Database health UP
- Disk capacity acceptable
- No unexpected failed units

Validation:
- /health returns UP
- /info shows version 1.1.0
- /release exists
- /db-check returns UP
- TLS true
- Prometheus app metric equals 1

Rollback:
Restore known-good WAR 1.0.0 and previous environment configuration.

Escalation:
Abort or escalate if pre-check fails, rollback fails,
database corruption is suspected, or failure exceeds approved L1 scope.
EOF
```

View:

```bash
cat "$CHANGE_DIR/change-ticket.txt"
```

Trong môi trường thật, field `Approval` phải đến từ approved ticket/change system; L1 không tự ghi chữ "approved" để vượt approval gate.

---

# 9. Tạo release pre-check batch

Đây là phần **batch/dependency/exit-code** của Day 18.

Tạo:

```bash
sudo tee \
  /usr/local/sbin/fresher-release-precheck.sh \
  >/dev/null <<'EOF'
#!/usr/bin/env bash

set -u -o pipefail

RC=0

fail_once() {
    local code="$1"
    shift

    echo "ERROR $*"

    if (( RC == 0 )); then
        RC="$code"
    fi
}

check_service() {
    local service="$1"
    local code="$2"

    if systemctl is-active --quiet "$service"; then
        echo "OK service=$service"
    else
        fail_once "$code" "service=$service not active"
    fi
}

echo "=== RELEASE PRECHECK ==="
date -Is

check_service tomcat10 10
check_service postgresql 11
check_service prometheus 12
check_service grafana-server 13

if pg_isready \
     -h 127.0.0.1 \
     -p 5432 \
     >/dev/null 2>&1
then
    echo "OK PostgreSQL readiness"
else
    fail_once 20 "PostgreSQL readiness failed"
fi

if curl \
     -fsS \
     --max-time 5 \
     http://127.0.0.1:8080/fresher-app/health \
     >/dev/null
then
    echo "OK application health"
else
    fail_once 21 "application health failed"
fi

if curl \
     -fsS \
     --max-time 5 \
     http://127.0.0.1:8080/fresher-app/db-check \
     >/dev/null
then
    echo "OK application database check"
else
    fail_once 22 "application database check failed"
fi

ROOT_USE="$(
    df -P / |
    awk 'NR==2 {
        gsub("%","",$5);
        print $5
    }'
)"

echo "INFO root_usage=${ROOT_USE}%"

if (( ROOT_USE >= 85 )); then
    fail_once 30 "root filesystem usage >= 85%"
fi

if systemctl --failed --no-legend \
     | grep -q .
then
    fail_once 31 "unexpected failed systemd units exist"
else
    echo "OK no failed systemd units"
fi

echo "RESULT precheck_exit_code=$RC"
exit "$RC"
EOF
```

---

# 10. Permission + syntax validation

```bash
sudo chown root:root \
  /usr/local/sbin/fresher-release-precheck.sh
```

```bash
sudo chmod 750 \
  /usr/local/sbin/fresher-release-precheck.sh
```

Syntax:

```bash
sudo bash -n \
  /usr/local/sbin/fresher-release-precheck.sh
```

Expected:

```text
no output
```

---

# 11. Chạy release gate

```bash
sudo /usr/local/sbin/fresher-release-precheck.sh \
  | tee "$CHANGE_DIR/precheck.log"
```

Vì command được pipe qua `tee`, không dùng ngay:

```bash
echo $?
```

để xác định exit code của script.

Dùng:

```bash
PRECHECK_RC=${PIPESTATUS[0]}
```

Sau đó:

```bash
echo "precheck_exit_code=$PRECHECK_RC" \
  | tee -a "$CHANGE_DIR/precheck.log"
```

Expected:

```text
precheck_exit_code=0
```

`PIPESTATUS[0]` là exit code của command đầu tiên:

```text
fresher-release-precheck.sh
```

không phải `tee`.

---

# 12. Approval gate

Logic operational:

```text
PRECHECK = 0
      ↓
continue release

PRECHECK != 0
      ↓
STOP
collect evidence
investigate
do NOT deploy
```

Có thể enforce:

```bash
if (( PRECHECK_RC != 0 )); then
    echo "RELEASE ABORTED: precheck failed"
else
    echo "RELEASE GATE PASSED"
fi
```

Trong runbook thật, không tiếp tục deployment khi precheck fail chỉ vì "demo đang thiếu thời gian".

---

# 13. Tạo release `1.1.0`

Ta sẽ thêm một endpoint mới:

```text
/fresher-app/release
```

để chứng minh artifact code thực sự thay đổi, không chỉ đổi biến `APP_VERSION`.

Tạo source:

```bash
cat > \
src/main/java/com/example/fresher/ReleaseServlet.java \
<<'EOF'
package com.example.fresher;

import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

@WebServlet("/release")
public class ReleaseServlet extends HttpServlet {

    @Override
    protected void doGet(
        HttpServletRequest request,
        HttpServletResponse response
    ) throws IOException {

        response.setStatus(HttpServletResponse.SC_OK);
        response.setContentType("text/plain");
        response.setCharacterEncoding("UTF-8");

        response.getWriter().println(
            "release_feature=operations-readiness"
        );

        response.getWriter().println(
            "status=enabled"
        );
    }
}
EOF
```

---

# 14. Update Maven project version

Chỉ thay occurrence đầu tiên của project version:

```bash
sed -i \
  '0,/<version>1\.0\.0<\/version>/s//<version>1.1.0<\/version>/' \
  pom.xml
```

Verify:

```bash
grep -m1 '<version>' pom.xml
```

Expected:

```xml
<version>1.1.0</version>
```

---

# 15. Review change

```bash
git diff
```

Bạn phải đọc được:

```text
pom.xml
1.0.0 → 1.1.0

new:
ReleaseServlet.java
```

Đây là nơi rất tốt để nói:

> "Trước build em review diff để đảm bảo change scope đúng ticket."

---

# 16. Commit `1.1.0`

```bash
git add \
  pom.xml \
  src/main/java/com/example/fresher/ReleaseServlet.java
```

Commit:

```bash
git commit \
  -m "Add operations readiness endpoint for 1.1.0"
```

Tag:

```bash
git tag -a v1.1.0 \
  -m "Release fresher-app 1.1.0"
```

Inspect:

```bash
git log \
  --oneline \
  --decorate \
  -5
```

Ta muốn có:

```text
... (HEAD -> master, tag: v1.1.0) ...
... (tag: v1.0.0) ...
```

---

# 17. Build release

```bash
mvn -B clean package
```

Expected:

```text
BUILD SUCCESS
```

Capture:

```bash
BUILD_RC=$?
echo "build_exit_code=$BUILD_RC"
```

Expected:

```text
0
```

Nếu build nonzero:

```text
DO NOT DEPLOY
```

---

# 18. Inspect WAR

```bash
ls -lh target/fresher-app.war
```

Confirm new class:

```bash
jar tf \
  target/fresher-app.war \
  | grep 'ReleaseServlet'
```

Expected:

```text
WEB-INF/classes/com/example/fresher/ReleaseServlet.class
```

Hash:

```bash
sha256sum \
  target/fresher-app.war
```

---

# 19. Promote artifact vào release repository

```bash
sudo install \
  -d \
  -o root \
  -g root \
  -m 755 \
  /opt/java-fresher/releases/fresher-app/1.1.0
```

Install:

```bash
sudo install \
  -o root \
  -g root \
  -m 644 \
  target/fresher-app.war \
  /opt/java-fresher/releases/fresher-app/1.1.0/fresher-app.war
```

Hash:

```bash
sudo sha256sum \
  /opt/java-fresher/releases/fresher-app/1.1.0/fresher-app.war \
  | tee "$CHANGE_DIR/artifact-1.1.0.sha256"
```

Release repository lúc này:

```text
/opt/java-fresher/releases/fresher-app/
├── 1.0.0/
│   └── fresher-app.war
└── 1.1.0/
    └── fresher-app.war
```

Đây là điều kiện rollback rất quan trọng.

---

# 20. BEFORE deployment — backup rollback state

Tạo secure directory vì environment file chứa password:

```bash
sudo install \
  -d \
  -o root \
  -g root \
  -m 700 \
  "$CHANGE_DIR/secure"
```

Backup configuration:

```bash
sudo cp -a \
  /etc/java-fresher/fresher-app.env \
  "$CHANGE_DIR/secure/fresher-app.env.before-release"
```

**Không `cat` file này.**

Backup deployed artifact:

```bash
sudo cp -a \
  /var/lib/tomcat10/webapps/fresher-app.war \
  "$CHANGE_DIR/fresher-app.war.before-release"
```

Hash:

```bash
sudo sha256sum \
  "$CHANGE_DIR/fresher-app.war.before-release" \
  | tee "$CHANGE_DIR/artifact-before.sha256"
```

---

# 21. Verify current version trước deployment

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/info
```

Expected:

```text
environment=demo
version=1.0.0
java=17...
```

Endpoint mới phải chưa tồn tại:

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/release
```

Expected baseline:

```text
404
```

Điều này rất hay:

```text
BEFORE:
1.0.0
/release → 404

AFTER:
1.1.0
/release → 200
```

---

# 22. Deploy `1.1.0`

Ta thực hiện một planned short outage.

Stop Tomcat:

```bash
sudo systemctl stop tomcat10
```

Verify:

```bash
systemctl is-active tomcat10
```

Expected:

```text
inactive
```

---

# 23. Remove exploded old deployment

Trước:

```bash
sudo find \
  /var/lib/tomcat10/webapps \
  -maxdepth 1 \
  -name 'fresher-app*' \
  -print
```

Chỉ sau khi chắc chắn path:

```bash
sudo rm -rf -- \
  /var/lib/tomcat10/webapps/fresher-app
```

**Không dùng wildcard kiểu:**

```text
rm -rf /var/lib/tomcat10/webapps/*
```

Đó là blast radius không chấp nhận được.

---

# 24. Install new WAR

Lấy Tomcat group:

```bash
TOMCAT_USER="$(
  systemctl show \
    -p User \
    --value \
    tomcat10
)"
```

```bash
TOMCAT_GROUP="$(
  id -gn "$TOMCAT_USER"
)"
```

Install:

```bash
sudo install \
  -o root \
  -g "$TOMCAT_GROUP" \
  -m 640 \
  /opt/java-fresher/releases/fresher-app/1.1.0/fresher-app.war \
  /var/lib/tomcat10/webapps/fresher-app.war
```

---

# 25. Update externalized version

Không expose DB password.

Chỉ sửa dòng:

```text
APP_VERSION
```

```bash
sudo sed -i \
  's/^APP_VERSION=.*/APP_VERSION=1.1.0/' \
  /etc/java-fresher/fresher-app.env
```

Verify chỉ key/value không nhạy cảm:

```bash
sudo grep '^APP_VERSION=' \
  /etc/java-fresher/fresher-app.env
```

Expected:

```text
APP_VERSION=1.1.0
```

---

# 26. Start

```bash
sudo systemctl start tomcat10
```

Capture:

```bash
START_RC=$?
echo "tomcat_start_exit=$START_RC"
```

Expected:

```text
0
```

---

# 27. AFTER deployment validation

Service:

```bash
systemctl is-active tomcat10
```

Expected:

```text
active
```

Port:

```bash
sudo ss -lntp | grep ':8080'
```

---

# 28. Health

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/health
```

Expected:

```text
status=UP
```

---

# 29. Version

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/info
```

Expected:

```text
environment=demo
version=1.1.0
java=17...
```

---

# 30. New functionality

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/release
```

Expected:

```text
release_feature=operations-readiness
status=enabled
```

Đây là bằng chứng artifact code mới thực sự đang chạy.

---

# 31. DB validation

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/db-check
```

Expected:

```text
db=UP
database=fresherdb
user=fresher_app
tls=true
...
```

---

# 32. Monitoring validation

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/metrics
```

Expected:

```text
fresher_app_up 1
fresher_app_db_up 1
fresher_app_db_tls 1
```

Prometheus:

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=fresher_app_db_up' \
  | jq
```

Value:

```text
1
```

Grafana dashboard:

```text
Java Application          UP
PostgreSQL Connectivity   UP
PostgreSQL TLS            ENABLED
```

Alert:

```text
FresherAppDatabaseDown
Normal
```

---

# 33. Lưu post-deploy evidence

```bash
{
    echo '=== TIME ==='
    date -Is

    echo
    echo '=== TOMCAT ==='
    systemctl is-active tomcat10

    echo
    echo '=== HEALTH ==='
    curl -fsS \
      http://127.0.0.1:8080/fresher-app/health

    echo
    echo '=== INFO ==='
    curl -fsS \
      http://127.0.0.1:8080/fresher-app/info

    echo
    echo '=== RELEASE ==='
    curl -fsS \
      http://127.0.0.1:8080/fresher-app/release

    echo
    echo '=== DATABASE ==='
    curl -fsS \
      http://127.0.0.1:8080/fresher-app/db-check

    echo
    echo '=== METRICS ==='
    curl -fsS \
      http://127.0.0.1:8080/fresher-app/metrics

} | tee "$CHANGE_DIR/post-deploy-validation.log"
```

---

# 34. Giữ known-good config `1.1.0`

Sau khi xác nhận deployment thành công:

```bash
sudo cp -a \
  /etc/java-fresher/fresher-app.env \
  "$CHANGE_DIR/secure/fresher-app.env.1.1.0-good"
```

Đây sẽ là rollback point cho failure injection tiếp theo.

---

# 35. Failure injection của Unit 6

Ta chọn:

```text
DB connection config error
```

Cụ thể:

```text
5432
→
5433
```

Lý do scenario này rất tốt:

```text
AWS               OK
Linux              OK
Tomcat             OK
Java               OK
port 8080          OK
PostgreSQL         OK on 5432
application DB     FAIL because config points 5433
Prometheus         detects
Grafana            alerts
```

Nó tích hợp gần như toàn bộ khóa học.

---

# 36. Inject failure

**LAB ONLY.**

Sửa:

```bash
sudo sed -i \
  's/127\.0\.0\.1:5432/127.0.0.1:5433/' \
  /etc/java-fresher/fresher-app.env
```

Validate mà không show password:

```bash
sudo grep '^DB_URL=' \
  /etc/java-fresher/fresher-app.env
```

DB URL không chứa password nên có thể show.

Bạn sẽ thấy:

```text
127.0.0.1:5433
```

Restart để environment mới được load:

```bash
sudo systemctl restart tomcat10
```

---

# 37. Không rollback ngay — OSI Mindset

Đây là phần bắt buộc.

## Application availability

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/health
```

Expected:

```text
HTTP 200
status=UP
```

Tomcat/Java:

```text
UP
```

---

# 38. DB dependency symptom

```bash
curl -i \
  http://127.0.0.1:8080/fresher-app/db-check
```

Expected:

```text
HTTP 503
db=DOWN
```

Đừng restart PostgreSQL.

---

# 39. Layer 4 evidence

Tomcat:

```bash
sudo ss -lntp | grep ':8080'
```

UP.

PostgreSQL thật:

```bash
sudo ss -lntp | grep ':5432'
```

UP.

Wrong configured port:

```bash
sudo ss -lntp | grep ':5433'
```

Expected:

```text
no output
```

Có thể test:

```bash
nc -zv \
  127.0.0.1 \
  5432
```

Success.

```bash
nc -zv \
  127.0.0.1 \
  5433
```

Fail.

Root cause bắt đầu rất rõ.

---

# 40. PostgreSQL service vẫn healthy

```bash
systemctl is-active postgresql
```

Expected:

```text
active
```

```bash
pg_isready \
  -h 127.0.0.1 \
  -p 5432
```

Expected:

```text
accepting connections
```

Cho nên:

```text
PostgreSQL itself is not down.
```

---

# 41. Recent change

Check:

```bash
sudo grep '^DB_URL=' \
  /etc/java-fresher/fresher-app.env
```

Thấy:

```text
5433
```

Trong khi listener:

```text
5432
```

Root cause:

```text
incorrect externalized DB port
```

---

# 42. OSI conclusion

Bạn nên nói:

> "Em chọn top-down vì application `/db-check` báo lỗi nhưng `/health` vẫn UP. Ở Layer 7 Java/Tomcat vẫn phản hồi. Xuống Layer 4 em thấy PostgreSQL thực sự listen 5432 nhưng application config đang trỏ tới 5433 và port 5433 không có listener. PostgreSQL service vẫn active và `pg_isready` thành công trên 5432. Recent change chính là DB_URL. Vì vậy root cause là application configuration sai port, không phải AWS Security Group, Tomcat hay PostgreSQL."

Đây là evidence-driven troubleshooting.

---

# 43. Monitoring phát hiện failure

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/metrics
```

Expected:

```text
fresher_app_up 1
fresher_app_db_up 0
fresher_app_db_tls 0
```

Prometheus:

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=fresher_app_db_up' \
  | jq
```

Expected value:

```text
0
```

Grafana:

```text
Java Application          UP
PostgreSQL Connectivity   DOWN
```

Alert:

```text
FresherAppDatabaseDown
Firing
```

Webhook evidence:

```bash
sudo journalctl \
  -u grafana-webhook-demo.service \
  --since '-5 minutes' \
  --no-pager
```

---

# 44. L1.5 remediation decision

Đây là safe L1 remediation vì:

```text
known recent config change
known correct previous configuration
no data modification
rollback file exists
application scope only
```

Không cần escalate DBA.

Nếu thay vào đó evidence cho thấy:

```text
database corruption
unknown repeated crash
data inconsistency
unknown business transaction blocking
restore production DB needed
```

thì phải preserve evidence và escalate.

---

# 45. Rollback

Ở đây **không rollback WAR 1.1.0**, vì artifact 1.1.0 đã chứng minh healthy trước failure.

Root cause chỉ là config.

Minimal safe rollback:

```bash
sudo cp -a \
  "$CHANGE_DIR/secure/fresher-app.env.1.1.0-good" \
  /etc/java-fresher/fresher-app.env
```

Restart:

```bash
sudo systemctl restart tomcat10
```

Đây là điểm operations rất quan trọng:

> Rollback không có nghĩa lúc nào cũng rollback tất cả. Rollback nên đưa chính component bị thay đổi sai về last-known-good state với blast radius nhỏ nhất.

---

# 46. Validate rollback

```bash
systemctl is-active tomcat10
```

Expected:

```text
active
```

Health:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/health
```

DB:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/db-check
```

Expected:

```text
db=UP
tls=true
```

Version:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/info
```

Expected:

```text
version=1.1.0
```

Feature:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/release
```

Expected:

```text
release_feature=operations-readiness
status=enabled
```

Metric:

```bash
curl -fsS \
  http://127.0.0.1:8080/fresher-app/metrics
```

Expected:

```text
fresher_app_up 1
fresher_app_db_up 1
fresher_app_db_tls 1
```

Grafana alert:

```text
Firing
→
Normal / Resolved
```

---

# 47. Full artifact rollback plan

Bạn không cần execute nếu `1.1.0` đang healthy, nhưng phải giải thích được.

Nếu artifact `1.1.0` itself bị lỗi:

```text
stop Tomcat
        ↓
remove exploded 1.1.0
        ↓
restore known-good WAR 1.0.0
        ↓
restore previous env file
        ↓
start Tomcat
        ↓
validate
```

Commands:

```bash
sudo systemctl stop tomcat10
```

```bash
sudo rm -rf -- \
  /var/lib/tomcat10/webapps/fresher-app
```

```bash
sudo install \
  -o root \
  -g "$TOMCAT_GROUP" \
  -m 640 \
  /opt/java-fresher/releases/fresher-app/1.0.0/fresher-app.war \
  /var/lib/tomcat10/webapps/fresher-app.war
```

Restore env:

```bash
sudo cp -a \
  "$CHANGE_DIR/secure/fresher-app.env.before-release" \
  /etc/java-fresher/fresher-app.env
```

Start:

```bash
sudo systemctl start tomcat10
```

Validation phải lại từ đầu:

```text
systemd
port
health
version
DB
TLS
metrics
Grafana
```

---

# 48. Rollback record

Tạo:

```bash
cat > "$CHANGE_DIR/rollback-record.txt" <<EOF
Change ID: $CHANGE_ID
Time: $(date -Is)

Failure injection:
Application DB_URL changed from PostgreSQL port 5432
to unused port 5433.

Symptom:
Application /health remained UP.
Application /db-check returned DOWN.

Evidence:
- Tomcat remained active.
- TCP/8080 remained listening.
- PostgreSQL remained active on TCP/5432.
- TCP/5433 had no listener.
- Application configuration pointed to TCP/5433.
- fresher_app_db_up changed from 1 to 0.
- Grafana alert fired.

Root cause:
Incorrect externalized application DB port.

Remediation:
Restored last-known-good 1.1.0 environment configuration.

Result:
Application and PostgreSQL connectivity restored.
fresher_app_db_up returned to 1.
Grafana alert resolved.

Full artifact rollback:
Not required because WAR 1.1.0 was proven healthy.
EOF
```

Đây là một rollback record tốt vì nó nói rõ **tại sao không rollback artifact**.

---

# 49. Integrated architecture review

Trong Unit 6 bạn cần trình bày nhanh:

```text
Client
  |
  | Internet
  v
AWS VPC
  |
  ├── Internet Gateway
  ├── Route Table
  ├── Public Subnet
  └── Security Group
       |
       v
EC2 Ubuntu
  |
  ├── systemd
  |
  ├── Tomcat / Java
  |      |
  |      └── fresher-app
  |             |
  |             └── JDBC/TLS
  |                    |
  |                    v
  ├── PostgreSQL
  |
  ├── node_exporter
  |
  ├── Prometheus
  |
  └── Grafana
```

Data flow quan trọng:

```text
Browser
→ Security Group
→ EC2
→ TCP/8080
→ Tomcat
→ Java
→ JDBC/TLS
→ TCP/5432
→ PostgreSQL
```

Monitoring flow:

```text
node_exporter
      ↓
Prometheus

Java /metrics
      ↓
Prometheus
      ↓
Grafana
      ↓
Alert
      ↓
Webhook
```

---

# 50. Configuration baseline

Bạn nên trình bày bảng này:

| Component     | Baseline                      |
| ------------- | ----------------------------- |
| OS            | Ubuntu 24.04 LTS              |
| Java          | OpenJDK 17                    |
| Tomcat        | Tomcat 10                     |
| App           | fresher-app 1.1.0             |
| JVM           | `-Xms256m -Xmx768m`           |
| PostgreSQL    | PostgreSQL 16                 |
| DB            | `fresherdb`                   |
| App role      | `fresher_app` least privilege |
| DB encryption | JDBC TLS                      |
| Prometheus    | localhost :9090               |
| node_exporter | localhost :9100               |
| Grafana       | :3000 restricted by SG        |
| App           | :8080 restricted by SG        |
| SSH           | :22 from workstation `/32`    |

---

# 51. Security checklist

Kiểm tra:

```bash
sudo ss -lntp
```

Bạn phải biết:

```text
22      SSH
8080    Tomcat
5432    PostgreSQL
9090    Prometheus
9100    node_exporter
3000    Grafana
9088    local webhook
```

Điểm cần nói:

```text
5432
9100
9088
```

không cần public exposure.

AWS Console → Security Group:

```text
22      workstation /32
8080    workstation /32
3000    workstation /32
9090    workstation /32
```

Không:

```text
22 → 0.0.0.0/0
5432 → Internet
9100 → Internet
```

---

# 52. Cost checklist

AWS Console kiểm tra:

```text
EC2
EBS volumes
Snapshots
Public IPv4
CloudWatch alarm
AWS Budget
```

Không chỉ hỏi:

> "EC2 có chạy không?"

Phải hỏi:

```text
Có volume lab thừa không?
Có snapshot test nào còn giữ không?
Có public IPv4 không cần thiết không?
Có resource nào tạo chỉ để rehearsal không?
```

---

# 53. Acceptance validation

Chạy cuối cùng:

```bash
echo '=== SERVICES ==='

for svc in \
  tomcat10 \
  postgresql \
  prometheus \
  prometheus-node-exporter \
  grafana-server \
  grafana-webhook-demo.service
do
    printf '%-35s ' "$svc"
    systemctl is-active "$svc"
done
```

Expected tất cả:

```text
active
```

---

Application:

```bash
echo '=== APPLICATION ==='

curl -fsS \
  http://127.0.0.1:8080/fresher-app/health

curl -fsS \
  http://127.0.0.1:8080/fresher-app/info

curl -fsS \
  http://127.0.0.1:8080/fresher-app/release

curl -fsS \
  http://127.0.0.1:8080/fresher-app/db-check
```

Expected:

```text
UP
version=1.1.0
release feature enabled
DB UP
TLS true
```

---

Monitoring:

```bash
curl -s \
  'http://127.0.0.1:9090/api/v1/query?query=fresher_app_db_up' \
  | jq
```

Expected:

```text
1
```

---

System:

```bash
systemctl --failed --no-pager
```

Expected:

```text
no unexpected failed unit
```

Disk:

```bash
df -h /
```

Memory:

```bash
free -h
```

---

# 54. Acceptance record

```bash
cat > "$CHANGE_DIR/acceptance-checklist.txt" <<EOF
Integrated Lab Acceptance
=========================

Time:
$(date -Is)

Release:
fresher-app 1.1.0

Linux:
PASS - EC2 Ubuntu operational
PASS - no unexpected failed services

Java/Tomcat:
PASS - Tomcat active
PASS - application health UP
PASS - release endpoint available

PostgreSQL:
PASS - PostgreSQL active
PASS - application DB connection UP
PASS - TLS active

Monitoring:
PASS - Prometheus operational
PASS - node_exporter operational
PASS - application metric operational
PASS - Grafana dashboard operational
PASS - alert failure/recovery tested
PASS - webhook notification received

Operations:
PASS - precheck gate executed
PASS - exit code verified
PASS - controlled failure injected
PASS - root cause identified
PASS - rollback performed
PASS - post-rollback validation successful

Security:
PASS - DB not publicly exposed
PASS - SSH restricted
PASS - application secrets externalized
PASS - least-privilege DB account

Final state:
ACCEPTED FOR LAB DEMO
EOF
```

---

# 55. Shift handover

Tạo:

```bash
cat > "$CHANGE_DIR/handover.txt" <<EOF
Shift Handover
==============

Time:
$(date -Is)

Change:
$CHANGE_ID

Current application:
fresher-app 1.1.0

Current state:
Tomcat active
PostgreSQL active
Prometheus active
Grafana active
Application health UP
Database connectivity UP
TLS active
Grafana alert Normal

Work completed:
- Release 1.1.0 built and deployed.
- New /release endpoint validated.
- Database connectivity validated.
- Monitoring validated.
- Controlled DB port configuration failure tested.
- Grafana alert and webhook notification verified.
- Configuration rollback tested successfully.

Known issues:
No known unresolved lab issue at handover.

Rollback point:
fresher-app 1.0.0 retained under
/opt/java-fresher/releases/fresher-app/1.0.0/

Database backup:
Available under
/opt/java-fresher/backups/postgresql/

Important:
AWS EC2 must be stopped after final Self Q&A/demo
unless continued lab use is explicitly required.
EOF
```

Đây chính là shift handover mindset:

```text
what changed
current state
what failed
what was recovered
known risk
rollback point
next action
```

---

# 56. History cuối Unit 6

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
  ~/java-fresher-demo/history/"$(date +%F)_unit06_operations.log"
```

Verify:

```bash
tail -30 \
  ~/java-fresher-demo/history/history.log
```

Bạn phải có thể giải thích ngẫu nhiên:

```text
git
mvn
systemctl
ss
curl
sed
sha256sum
journalctl
PIPESTATUS
```

---

# 57. Kịch bản trình bày Unit 6 trong 20 phút

Đừng ngồi tạo source/ticket từ đầu trong video. Preparation ở trên phải hoàn thành trước.

| Thời gian | Demo                                                                  |
| --------: | --------------------------------------------------------------------- |
|      0–3' | Change ticket, Git `v1.0.0 → v1.1.0`, release plan                    |
|      3–5' | Precheck batch + exit code + approval gate                            |
|      5–9' | Deploy `1.1.0` + validate `/health`, `/info`, `/release`, `/db-check` |
|     9–12' | Architecture + security/cost baseline                                 |
|    12–17' | Inject DB port `5433` → OSI troubleshooting → Grafana alert           |
|    17–19' | Restore known-good config → validate monitoring recovery              |
|    19–20' | Rollback record + acceptance + handover                               |

Sau đó mới đến **Self Q&A 5 phút**, đúng guide. :chatgpt-content-reference{index="3"}

---

# 58. Sau Self Q&A — dừng AWS resources

Đây là bước **sau toàn bộ demo**, không làm trước Self Q&A vì bạn vẫn cần EC2/history/terminal.

Guide yêu cầu rõ phải dừng resource AWS sau buổi demo để tránh phát sinh chi phí. :chatgpt-content-reference{index="4"}

Trước khi stop, trên EC2 chạy final state:

```bash
date -Is
```

```bash
systemctl --failed --no-pager
```

```bash
history -a
```

Sau đó:

```bash
history > \
  ~/java-fresher-demo/history/history.log
```

Không shutdown EC2 bằng command trước khi bạn đã hoàn tất recording/evidence cần thiết.

---

# 59. Stop EC2 bằng AWS Console

Theo convention của chúng ta, AWS actions dùng Console:

```text
AWS Console
→ EC2
→ Instances
→ java-fresher-integrated-lab
→ Instance state
→ Stop instance
```

Chọn:

```text
Stop
```

Không chọn:

```text
Terminate
```

nếu bạn vẫn muốn sử dụng lại lab.

AWS thực hiện graceful OS shutdown khi stop theo phương thức mặc định. Khi instance EBS-backed bị stopped, compute instance không còn bị tính instance-usage charge, nhưng EBS root/data volumes vẫn tồn tại và vẫn phát sinh storage charge. :chatgpt-content-reference{index="5"}

Verify Console:

```text
Instance state
Stopped
```

Không chỉ click Stop rồi đóng browser.

---

# 60. Vì sao `Stop`, không `Terminate`?

`Stop`:

```text
instance compute off
EBS preserved
configuration preserved
có thể Start lại
```

`Terminate`:

```text
instance destroyed
```

và root EBS có thể bị delete tùy `DeleteOnTermination`.

Với lab còn có khả năng dùng lại:

```text
Stop
```

là lựa chọn đúng.

AWS cũng lưu ý stop không giữ contents trong RAM nhưng các EBS volumes vẫn tồn tại. :chatgpt-content-reference{index="6"}

---

# 61. Public IPv4 sau stop

Instance của chúng ta dùng **auto-assigned public IPv4**, không phải Elastic IP.

AWS cho biết khi stopped instance được start trở lại, nó thường nhận **public IPv4 mới**. :chatgpt-content-reference{index="7"}

Điều này có nghĩa lần sau:

```text
old:
18.x.x.x

start again:
new public IP possible
```

Bạn phải kiểm tra lại:

```text
EC2
→ Public IPv4 address
```

và SSH target/browser URL có thể phải cập nhật.

---

# 62. Vì sao phải quan tâm public IPv4?

Pricing hiện hành của AWS là:

````text
In-use public IPv4:
$0.005 / IP / hour

Idle public IPv4:
$0.005 / IP / hour
``` :chatgpt-content-reference{index="8"}


Do đó không nên giữ public IPv4/EIP không cần thiết.

Với **auto-assigned EC2 public IPv4**, stopping EC2 sẽ giải phóng địa chỉ đó trong lifecycle thông thường; nếu sau này dùng Elastic IP riêng thì phải kiểm tra và release nó nếu không cần.

---

# 63. EBS không dừng tính phí chỉ vì EC2 stopped

Đây là lỗi hiểu rất phổ biến:

```text
EC2 stopped
≠
AWS bill = 0
````

AWS nêu rõ khi EC2 stopped, instance usage không còn tính nhưng **root và data EBS volumes vẫn được giữ và tiếp tục tính phí storage**. :chatgpt-content-reference{index="9"}

Trong lab của chúng ta có thể còn:

```text
root EBS
java-fresher-ebs-lab
```

từ Unit 3.

---

# 64. Xử lý data EBS volume Unit 3

Nếu bạn **còn cần lab lần sau**:

```text
giữ volume
```

nhưng hiểu rằng vẫn có storage charge.

Nếu final demo đã hoàn tất và volume Unit 3 chỉ là temporary lab data:

Trước khi EC2 stop, hoặc trước cleanup, đảm bảo filesystem không còn cần:

```bash
findmnt /mnt/labdata
```

Nếu mounted và muốn delete:

```bash
sudo umount /mnt/labdata
```

Verify:

```bash
findmnt /mnt/labdata
```

không còn output.

Sau đó AWS Console:

```text
EC2
→ Volumes
→ java-fresher-ebs-lab
→ Actions
→ Detach volume
```

Khi state:

```text
Available
```

và đã xác minh **đúng Volume ID**:

```text
Actions
→ Delete volume
```

Đây là destructive action. Chỉ delete khi không còn retention requirement.

---

# 65. Snapshot Unit 3

Ta cũng từng tạo:

```text
java-fresher-ebs-snapshot
```

Snapshots tiếp tục chiếm billed snapshot storage; AWS tính snapshot charge dựa trên lượng snapshot data được lưu. :chatgpt-content-reference{index="10"}

Nếu không cần backup nữa:

```text
EC2
→ Snapshots
→ java-fresher-ebs-snapshot
→ Actions
→ Delete snapshot
```

AWS lưu ý việc xóa snapshot không ảnh hưởng volume hiện tại, nhưng snapshot data còn được reference bởi snapshot khác có thể vẫn được giữ. :chatgpt-content-reference{index="11"}

Nếu snapshot là evidence/recovery point bạn còn cần, **đừng delete chỉ để giảm vài cent**. Retention requirement ưu tiên trước cost cleanup.

---

# 66. CloudWatch Alarm

Nếu alarm:

```text
java-fresher-high-cpu
```

chỉ phục vụ lab và không còn dùng nữa:

```text
CloudWatch
→ Alarms
→ java-fresher-high-cpu
→ Actions
→ Delete
```

Nếu bạn còn dùng lab sau đó:

```text
có thể giữ
```

nhưng phải biết nó tồn tại trong inventory/cost checklist.

---

# 67. AWS Budget

Budget:

```text
java-fresher-monthly-budget
```

không nhất thiết phải xóa.

Giữ budget thường có ích vì:

```text
nó tiếp tục cảnh báo nếu bạn vô tình để resource chạy
```

Đây chính là một control tốt sau buổi lab.

---

# 68. IAM lab user

Unit 3 đã tạo:

```text
java-fresher-demo-ops
```

Nếu user này chỉ phục vụ exam rehearsal và không còn cần:

```text
IAM
→ Users
→ java-fresher-demo-ops
```

Remove/disable temporary access theo cleanup plan.

Đây chủ yếu là **security cleanup**, không phải cost cleanup.

Không giữ stale human credentials chỉ vì chúng không tính tiền.

---

# 69. VPC/subnet/route table/IGW/SG có cần xóa không?

Nếu bạn định học/rehearse thêm:

```text
giữ VPC architecture
```

để lần sau chỉ cần Start EC2.

Không cần phá toàn bộ network mỗi buổi.

Với design hiện tại chúng ta cũng **không tạo NAT Gateway**, vì vậy không có NAT Gateway hourly charge phải xử lý.

Nếu khóa lab **kết thúc hoàn toàn** và bạn không dùng VPC nữa, có thể full teardown sau khi chắc chắn:

```text
không còn EC2
không còn ENI
không còn EBS dependency
không còn Elastic IP
không còn resource dùng VPC
```

rồi mới xóa SG/subnet/route/IGW/VPC.

Không teardown network giữa lúc còn dependent resources.

---

# 70. Hai chế độ cleanup phải phân biệt

### Pause lab — khuyến nghị nếu còn ôn thi

Giữ:

```text
VPC
subnets
route tables
Security Group
root EBS
application data
snapshot cần thiết
```

Stop:

```text
EC2
```

Delete:

```text
temporary restored EBS
temporary test resources
unneeded snapshots
```

Kết quả:

```text
lần sau có thể Start lại nhanh
```

nhưng EBS vẫn có storage cost.

---

### Full teardown — khi hoàn toàn kết thúc lab

Sau khi backup/evidence đã an toàn:

```text
Terminate EC2
delete unused EBS
delete unused snapshots
release public/EIP resources
delete temporary CloudWatch resources
remove temporary IAM identities
delete network resources nếu không còn dependency
```

**Full teardown là destructive và không thể coi như Stop.**

---

# 71. Cách trình bày phần cost-control cuối video

Bạn nên nói ngắn gọn:

> "Buổi demo đã hoàn tất nên em không để resource chạy ngoài nhu cầu. Em stop EC2 để ngừng compute usage, đồng thời kiểm tra public IPv4 và các EBS/snapshot còn tồn tại. Em hiểu rằng stop EC2 không đồng nghĩa mọi chi phí bằng 0 vì EBS vẫn được giữ và tiếp tục tính storage. Những EBS/snapshot tạm không còn retention requirement sẽ được xóa; resource cần cho lần ôn tiếp theo thì em giữ có chủ đích. Em cũng giữ AWS Budget để phát hiện chi phí ngoài dự kiến."

Đây là câu trả lời operations rất tốt.

---

# 72. Final Unit 6 mastery flow

Sau toàn bộ Unit 1–6, bạn đang chứng minh được chuỗi thực sự mà syllabus yêu cầu:

```text
understand architecture
        ↓
prepare change
        ↓
approval gate
        ↓
dependency check
        ↓
exit-code check
        ↓
version control
        ↓
build artifact
        ↓
backup current state
        ↓
deploy
        ↓
validate
        ↓
monitor
        ↓
inject failure
        ↓
OSI troubleshooting
        ↓
identify root cause
        ↓
L1-safe remediation
        ↓
rollback
        ↓
revalidate
        ↓
document
        ↓
handover
        ↓
acceptance
        ↓
Self Q&A
        ↓
stop / cleanup AWS resources
```

Đặc biệt, sau Unit 6 **đừng stop EC2 ngay lập tức trước Self Q&A**. Guide còn yêu cầu 5 phút Self Q&A với `history.log`, tự chọn 2–3 command và tự hỏi “nếu bước này lỗi thì sao?”. :chatgpt-content-reference{index="12"} Sau khi Self Q&A và mọi recording/evidence cần thiết đã hoàn tất mới thực hiện AWS Console → **Stop instance** và resource cleanup ở trên.
