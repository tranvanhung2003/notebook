# Day 18 — Batch Scheduling, Change/Release Control, Evidence, Rollback & Shift Handover

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 18 thuộc phần **Operations Controls & Integrated Practice**. Phạm vi bắt buộc là:

> **Concept/Lecture:** Batch scheduling, dependency and exit-code checks, release/change ticket flow, approval gates, evidence, rollback, and shift handover.  
> **Assignment/Lab:** Execute an approved batch/release scenario and produce a complete ticket, evidence pack, rollback record, and handover note. 00_MASTER_SYLLABUS_Java_Fresher…

Day 18 rất quan trọng vì từ đây bạn không còn chỉ được đánh giá theo kiểu “có biết command hay không”. Bạn phải chứng minh rằng mình có thể **vận hành thay đổi một cách kiểm soát được**.

Mental model của hôm nay là:

```text
Request
   ↓
Change / Release Ticket
   ↓
Risk + Impact + Implementation + Validation + Rollback plan
   ↓
Approval Gate
   ↓
Pre-check
   ↓
Execute change / batch
   ↓
Check exit code + logs + dependencies
   ↓
Technical validation
   ↓
Business / service validation
   ↓
SUCCESS ────────→ Evidence → Ticket closure → Handover
   │
   └── FAILURE → Stop → Assess → Rollback → Validate → Evidence → Escalate/Handover
```

Đây cũng là sự kết nối tự nhiên với những gì bạn vừa học: Java deployment/rollback ở Day 12–14, PostgreSQL ở Day 15–16 và monitoring/alerting ở Day 17. 00_MASTER_SYLLABUS_Java_Fresher…

---

# 2. Mục tiêu của Day 18

Kết thúc Day 18, bạn cần có khả năng giải thích và thực hiện được toàn bộ chuỗi sau:

| Năng lực           | Bạn phải làm được                                                                       |
| ------------------ | --------------------------------------------------------------------------------------- |
| Batch scheduling   | Biết batch job là gì, chạy lúc nào, bằng cron/systemd timer hoặc scheduler tương đương  |
| Dependency check   | Không chạy batch một cách mù quáng khi upstream dependency chưa sẵn sàng                |
| Exit-code handling | Phân biệt success/failure bằng return code, không dựa vào cảm giác hoặc vài dòng output |
| Change control     | Không tự ý thay đổi production-like system ngoài ticket                                 |
| Approval gate      | Biết khi nào phải STOP vì chưa có authorization                                         |
| Release execution  | Pre-check → implement → validate                                                        |
| Evidence           | Chứng minh chính xác mình đã làm gì, khi nào, trên đâu và kết quả thế nào               |
| Rollback           | Khôi phục trạng thái trước change một cách có kế hoạch                                  |
| Shift handover     | Bàn giao chính xác tình trạng cho ca/người tiếp theo                                    |
| Escalation         | Biết giới hạn L1 và khi nào phải chuyển cho L2/L3/DBA/App/Cloud team                    |

Một fresher vận hành tốt không phải người “chạy command nhanh nhất”, mà là người có thể trả lời:

```text
Tôi được phép làm không?
Tôi đang thay đổi cái gì?
Impact nếu lỗi là gì?
Tôi đã kiểm tra prerequisite chưa?
Làm sao biết thành công?
Làm sao quay lại?
Evidence nằm ở đâu?
Nếu hết ca ngay bây giờ, người tiếp theo hiểu hệ thống đang ở trạng thái nào?
```

---

# 3. Batch job là gì?

**Batch processing** là việc thực thi một nhóm công việc mà thông thường không cần người dùng tương tác trực tiếp tại thời điểm thực thi.

Ví dụ:

```text
23:00        Generate daily transaction report
00:05        Import settlement file
01:00        Database housekeeping
02:00        Backup
02:30        Transfer backup
03:00        Verify backup
04:00        Generate reconciliation report
```

Khác với một web request:

```text
Client
  ↓
HTTP request
  ↓
Java application
  ↓
Database
  ↓
Immediate HTTP response
```

Batch thường có dạng:

```text
Scheduler
   ↓
Job
   ↓
Check dependencies
   ↓
Perform processing
   ↓
Write database/files
   ↓
Generate exit status
   ↓
Write log
   ↓
Trigger next job / alert
```

Batch có thể là Bash script, Java JAR, SQL script, ETL job, backup job, reconciliation process, log rotation task hoặc deployment/release automation.

---

# 4. Batch scheduling

Có hai scheduling mechanisms rất hữu ích cho fresher Linux operations:

```text
cron
systemd timer
```

Ngoài ra enterprise environment có thể dùng Control-M, Autosys, Rundeck, Jenkins, Airflow, AWS EventBridge Scheduler, Kubernetes CronJob hoặc scheduler nội bộ.

Các scheduler enterprise này là **Supplementary / Beyond the explicit syllabus**. Day 18 yêu cầu hiểu khái niệm batch scheduling chứ không quy định một scheduler enterprise cụ thể.

---

## 4.1 cron

Cron biểu diễn lịch cơ bản bằng 5 trường:

```text
minute hour day-of-month month day-of-week command
```

Ví dụ:

```cron
30 2 * * * /opt/batch/daily-report.sh
```

nghĩa là:

```text
02:30
mỗi ngày
chạy /opt/batch/daily-report.sh
```

Một cron implementation phổ biến cho phép mỗi user có crontab riêng, và job chạy dưới identity của user sở hữu crontab. Cron cũng thường có environment tối giản hơn interactive shell của bạn. [man7](https://www.man7.org/linux/man-pages/man5/crontab.5.html?utm_source=chatgpt.com)

Đây là lỗi fresher rất thường gặp:

```bash
$ /opt/batch/job.sh
SUCCESS
```

nhưng cron lại fail.

Nguyên nhân có thể là:

```text
PATH khác
working directory khác
environment variable không tồn tại
JAVA_HOME không tồn tại
credential/config không được load
relative path sai
permission khác
```

Vì vậy thay vì:

```cron
0 1 * * * ./backup.sh
```

nên ưu tiên path rõ ràng:

```cron
0 1 * * * /opt/javaops/bin/backup.sh >> /var/log/javaops/backup-cron.log 2>&1
```

và script cũng nên dùng absolute paths cho những tài nguyên quan trọng.

---

# 5. systemd timer

**Supplementary / Beyond the explicit syllabus**, nhưng rất đáng học vì môi trường Linux hiện đại thường sử dụng systemd.

Systemd timer gồm hai unit:

```text
something.timer
        ↓ activates
something.service
```

Ví dụ:

```ini
# /etc/systemd/system/javaops-batch.service

[Unit]
Description=JavaOps Daily Batch

[Service]
Type=oneshot
User=appsvc
ExecStart=/opt/javaops/bin/daily-batch.sh
```

Timer:

```ini
# /etc/systemd/system/javaops-batch.timer

[Unit]
Description=Run JavaOps Daily Batch

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

`OnCalendar=` sử dụng calendar expressions. `Persistent=true` đặc biệt hữu ích với calendar timers: nếu timer đáng lẽ phải chạy trong lúc timer/system không hoạt động thì khi timer được kích hoạt lại, systemd có thể thực hiện một catch-up activation. [man7](https://man7.org/linux/man-pages/man5/systemd.timer.5.html?utm_source=chatgpt.com)

Validation:

```bash
sudo systemd-analyze calendar '*-*-* 02:30:00'
```

Sau khi tạo unit:

```bash
sudo systemctl daemon-reload

sudo systemctl enable --now javaops-batch.timer

systemctl status javaops-batch.timer

systemctl list-timers --all
```

Kiểm tra execution:

```bash
systemctl status javaops-batch.service
```

và:

```bash
journalctl -u javaops-batch.service
```

### Một điểm cần nhớ

Timer tồn tại chưa có nghĩa batch chạy thành công.

```text
Timer trigger thành công
            ≠
Batch thành công
```

Bạn phải đi tiếp xuống:

```text
scheduler
→ process started?
→ script executed?
→ dependencies passed?
→ business processing succeeded?
→ exit status?
→ expected output generated?
```

---

# 6. Dependency management — phần cực kỳ quan trọng

Giả sử có ba batch:

```text
JOB-A: download-file
JOB-B: import-file
JOB-C: generate-report
```

Quan hệ đúng:

```text
JOB-A
  │ success
  ▼
JOB-B
  │ success
  ▼
JOB-C
```

Không được đơn giản viết:

```cron
00 1 * * * download.sh
05 1 * * * import.sh
10 1 * * * report.sh
```

rồi cho rằng “A chắc chắn chạy xong trước 01:05”.

Điều gì xảy ra nếu hôm nay download mất 20 phút?

```text
01:00 A starts
01:05 B starts        ← input chưa hoàn chỉnh
01:10 C starts        ← data chưa import xong
```

Bạn có thể tạo ra **business data corruption**, không chỉ technical failure.

Dependency phải dựa trên **state**, không nên chỉ dựa vào thời gian.

Ví dụ dependency checks có thể là:

```text
file exists?
file checksum valid?
previous job succeeded?
database accepting connections?
application health endpoint healthy?
disk has enough capacity?
required mount exists?
upstream API reachable?
approval/change window still valid?
```

---

# 7. Exit code — ngôn ngữ căn bản của automation

Trong Bash/Linux:

```text
0       success
non-zero failure / exceptional result
```

GNU Bash định nghĩa exit status trong khoảng `0–255`; `0` được dùng cho success, non-zero biểu thị failure. Khi process kết thúc bởi signal `N`, Bash dùng `128+N`; command không tồn tại thường cho `127`, còn tìm thấy nhưng không executable là `126`. `$?` chứa status của command gần nhất. [GNU](https://www.gnu.org/s/bash/manual/bash.pdf?utm_source=chatgpt.com)

Ví dụ:

```bash
ls /tmp
echo $?
```

Nếu success:

```text
0
```

Thử:

```bash
ls /this-path-does-not-exist
echo $?
```

sẽ cho non-zero.

---

# 8. Sai lầm kinh điển với `$?`

Không làm thế này:

```bash
some_command
echo "Finished"
echo $?
```

Bạn tưởng `$?` là status của `some_command`.

Không phải.

Nó đã trở thành exit status của:

```bash
echo "Finished"
```

Cách đúng:

```bash
some_command
rc=$?

echo "Command exit code: $rc"

if [ "$rc" -ne 0 ]; then
    echo "FAILED"
    exit "$rc"
fi
```

Hoặc:

```bash
if some_command; then
    echo "SUCCESS"
else
    rc=$?
    echo "FAILED rc=$rc"
    exit "$rc"
fi
```

---

# 9. Pipeline exit code và `pipefail`

Xem lệnh:

```bash
cat missing-file | grep ERROR
```

Nếu shell mặc định chỉ nhìn exit code cuối pipeline, bạn có thể che mất lỗi upstream trong một số pipeline.

Vì vậy batch script thường có:

```bash
set -o pipefail
```

Một pattern thường gặp là:

```bash
set -Eeuo pipefail
```

Ý nghĩa khái quát:

```text
-e        stop khi command chưa được xử lý thích hợp trả non-zero
-u        lỗi khi dùng unset variable
-o pipefail
          pipeline không che giấu failure upstream
-E        ERR trap được kế thừa trong thêm một số context
```

Nhưng đừng hiểu `set -e` là “magic error handling”. Production script vẫn phải xử lý rõ ràng những trường hợp cần retry, ignore hoặc convert exit status.

---

# 10. Exit-code contract

Batch chuyên nghiệp nên có **exit-code contract**.

Ví dụ do team của bạn định nghĩa:

| Exit code | Ý nghĩa                         |
| --------: | ------------------------------- |
|         0 | Success                         |
|        10 | Missing dependency/input        |
|        20 | Application health check failed |
|        30 | Database unavailable            |
|        40 | Processing failed               |
|        50 | Validation failed               |

Lưu ý: các code `10/20/30/...` trên là **contract ví dụ của lab**, không phải Linux standard.

Điểm quan trọng:

```text
Producer:
job returns exit code

Consumer:
scheduler / monitoring / wrapper interprets exit code
```

---

# 11. PostgreSQL dependency check

Từ Day 15–16 bạn đã học PostgreSQL.

Một batch có thể kiểm tra database:

```bash
pg_isready -h 127.0.0.1 -p 5432
```

PostgreSQL định nghĩa rõ:

| Exit | Ý nghĩa                                                 |
| ---: | ------------------------------------------------------- | ------------------------------------- |
|    0 | server accepting connections                            |
|    1 | server rejecting connections                            |
|    2 | no response                                             |
|    3 | no connection attempt made, chẳng hạn invalid parameter | :chatgpt-content-reference{index="5"} |

Ví dụ:

```bash
if pg_isready -q -h 127.0.0.1 -p 5432 -t 3; then
    echo "PostgreSQL dependency OK"
else
    rc=$?
    echo "PostgreSQL dependency failed: pg_isready rc=$rc"
    exit 30
fi
```

Đây là cách suy nghĩ tốt hơn:

```text
“port 5432 open”
       ≠
“PostgreSQL ready”
```

và:

```text
“PostgreSQL accepting connections”
       ≠
“application credential/schema/query are correct”
```

Mỗi check chỉ chứng minh đúng layer mà nó kiểm tra.

---

# 12. SQL script phải propagate failure

Một anti-pattern nguy hiểm:

```bash
psql -d appdb -f batch.sql
echo "Batch completed"
```

Theo PostgreSQL documentation, `psql` mặc định có thể tiếp tục sau SQL error trong script nếu bạn không bật `ON_ERROR_STOP`. Với `ON_ERROR_STOP=on`, script dừng và `psql` sử dụng exit code `3` cho loại script error này. [PostgreSQL](https://www.postgresql.org/docs/current/app-psql.html?utm_source=chatgpt.com)

Do đó:

```bash
psql \
  -X \
  -v ON_ERROR_STOP=1 \
  -d appdb \
  -f batch.sql
```

Sau đó:

```bash
rc=$?

if [ "$rc" -ne 0 ]; then
    echo "SQL batch failed rc=$rc"
    exit 40
fi
```

Đây chính là **dependency + exit-code thinking** của Day 18.

---

# 13. HTTP validation cũng phải quan tâm exit code

Một fresher có thể viết:

```bash
curl http://localhost:8080/myapp/health
```

nhìn thấy response rồi kết luận OK.

Nhưng mặc định curl không nhất thiết coi HTTP `404`, `500`, v.v. là transfer failure. `--fail`/`-f` khiến HTTP response `400+` trở thành failure exit status trong những trường hợp được hỗ trợ. [curl.se](https://curl.se/docs/faq.html?utm_source=chatgpt.com)

Do đó health validation thường giống:

```bash
curl --fail --silent --show-error \
     --connect-timeout 3 \
     --max-time 10 \
     http://127.0.0.1:8080/myapp/health
```

Nếu application của lab không có `/health`, **không được tự giả định endpoint này tồn tại**. Hãy thay bằng endpoint validation thực tế của application bạn đã dùng ở Day 12–14.

---

# 14. Change và Release khác nhau thế nào?

**Change** rộng hơn release.

```text
Change
├── modify config
├── restart service
├── firewall change
├── JVM tuning
├── database config
├── monitoring rule
├── scheduled batch
└── deploy software
```

Release thường nói về việc đưa software/artifact/version mới vào environment.

Ví dụ:

```text
Change:
Increase Java heap from -Xmx512m → -Xmx1024m

Release:
Deploy application version 1.4.2

Batch:
Run month-end settlement job
```

Nhưng cả ba đều có thể yêu cầu operational control tương tự.

---

# 15. Change ticket là gì?

Ticket không phải bureaucracy vô nghĩa.

Nó là một **control object** giúp trả lời:

```text
What?
Why?
Where?
When?
Who requested?
Who approved?
What is affected?
What can go wrong?
How will it be implemented?
How will success be proven?
How will failure be reversed?
Who needs to know?
```

Một good change ticket tạo ra **traceability**.

Ví dụ:

```text
INC-xxxxx     Incident
CHG-xxxxx     Change
REL-xxxxx     Release
REQ-xxxxx     Request
```

Tên prefix tùy organization.

Không được mặc định công ty nào cũng sử dụng cùng convention.

---

# 16. Change lifecycle

Một lifecycle điển hình:

```text
Draft
  ↓
Technical review
  ↓
Risk / impact assessment
  ↓
Approval
  ↓
Scheduled
  ↓
Implementation
  ↓
Validation
  ↓
Completed
  ↓
Closure
```

Failure branch:

```text
Implementation
      ↓
Validation failed
      ↓
Stop change
      ↓
Rollback decision
      ↓
Rollback
      ↓
Validate restored state
      ↓
Record evidence
      ↓
Incident/escalation if required
```

---

# 17. Approval gate

Đây là một chủ đề thi rất quan trọng.

Một operator có access:

```bash
sudo systemctl restart tomcat
```

không có nghĩa operator được phép restart Tomcat bất cứ lúc nào.

Phải phân biệt:

```text
Technical capability
        ≠
Operational authorization
```

Ví dụ ticket nói:

```text
Change Window:
22:00–22:30

Approved by:
Application owner

Target:
APP-PROD-01
```

Bây giờ là:

```text
21:40
```

Bạn **không chạy**.

Hoặc target thực tế là:

```text
APP-PROD-02
```

Bạn **không chạy**.

Hoặc ticket trạng thái:

```text
Pending Approval
```

Bạn **không chạy**.

Đây là một approval gate.

---

# 18. Four-eyes principle / separation of duties

**Supplementary / Beyond explicit syllabus.**

Trong nhiều organization:

```text
Requester
    ≠
Approver
```

hoặc:

```text
Developer
    ↓ prepares
Release manager / authorized operator
    ↓ deploys
Service owner
    ↓ validates
```

Mục tiêu là giảm:

```text
mistake
unauthorized action
conflict of interest
untraceable change
```

Fresher cần hiểu rằng “manager bảo qua chat làm đi” không tự động thay thế formal approval process nếu company policy yêu cầu ticket approval.

---

# 19. Change window

Một change thường có:

```text
Planned Start
Planned End
```

Ví dụ:

```text
2026-10-12 22:00 ICT
2026-10-12 23:00 ICT
```

Bên trong cửa sổ:

```text
22:00 pre-check
22:05 deployment
22:15 startup
22:20 technical validation
22:25 application validation
22:35 monitoring observation
22:45 closure
```

Bạn phải tính cả **rollback time**.

Sai:

```text
Change ends 23:00
Start implementation at 22:58
```

Nếu rollback mất 15 phút thì change plan đã không khả thi.

---

# 20. Risk assessment

Một ticket tốt không ghi đơn giản:

```text
Risk: Low
```

Nó nên nói rõ reasoning.

Ví dụ:

```text
Potential impact:
Tomcat unavailable for approximately 2–5 minutes during restart.

Failure modes:
- WAR does not start
- configuration incompatible
- database connectivity fails
- health endpoint fails

Blast radius:
Single lab application instance

Mitigation:
- pre-change artifact backup
- config backup
- validate database before deployment
- maintain previous WAR artifact

Rollback trigger:
Application fails startup/health validation within agreed validation window.
```

---

# 21. Implementation plan phải executable

Kế hoạch yếu:

```text
Deploy new version.
Check.
Done.
```

Kế hoạch tốt:

```text
1. Confirm ticket approval and change window.
2. Confirm correct host.
3. Confirm current application version.
4. Confirm service is healthy before change.
5. Save pre-change evidence.
6. Back up current artifact/configuration.
7. Deploy approved artifact.
8. Start/restart service.
9. Inspect service status.
10. Inspect logs.
11. Validate endpoint.
12. Validate database dependency.
13. Confirm monitoring stable.
14. Record evidence.
```

Mỗi bước phải có thể chứng minh được.

---

# 22. Validation plan

Một release không thành công chỉ vì:

```bash
systemctl status tomcat
```

nói `active`.

Mental model:

```text
Process validation
        ↓
Port validation
        ↓
HTTP validation
        ↓
Application validation
        ↓
Database validation
        ↓
Monitoring validation
        ↓
Business validation
```

Ví dụ:

```bash
systemctl is-active tomcat
```

chỉ chứng minh service manager thấy unit active.

Tiếp tục:

```bash
ss -lntp
```

kiểm tra socket.

Tiếp:

```bash
curl -f ...
```

kiểm tra HTTP.

Tiếp:

```text
expected application function
```

Tiếp:

```text
DB connectivity
```

Tiếp:

```text
Prometheus/Grafana/alerts
```

---

# 23. Evidence là gì?

Evidence phải trả lời:

```text
Who?
When?
Where?
What?
Result?
```

Ví dụ:

```text
Hostname
Timestamp
Command
Relevant output
Application version
Service state
HTTP status
Database state
Monitoring state
Ticket reference
```

Không phải cứ screenshot thật nhiều là evidence tốt.

Evidence tốt là:

```text
minimal
relevant
timestamped
traceable
does not leak secrets
```

---

# 24. Không bao giờ đưa secret vào evidence

Không capture:

```text
DB_PASSWORD
AWS_SECRET_ACCESS_KEY
private SSH key
session token
Authorization header
full credential file
```

Ví dụ nguy hiểm:

```bash
env
```

rồi paste toàn bộ vào ticket.

Có thể trong environment tồn tại secrets.

Thay vào đó:

```bash
systemctl show myapp --property=ActiveState
```

hoặc chỉ capture các environment values an toàn cần chứng minh.

Security principle:

```text
Evidence sufficient to prove
≠
Dump everything available
```

---

# 25. Rollback là gì?

Rollback là hành động đưa hệ thống về **known-good state** trước change hoặc tới một recovery state đã được duyệt.

Ví dụ release:

```text
v1.3.0
   ↓ deploy
v1.4.0
   ↓ validation FAIL
rollback
   ↓
v1.3.0
```

Rollback không đơn giản là:

```bash
undo
```

Bạn cần biết:

```text
what exactly changed?
artifact?
config?
database schema?
systemd unit?
firewall?
JVM options?
permissions?
```

---

# 26. Rollback trigger phải định nghĩa trước

Sai:

```text
Rollback if something looks wrong.
```

Tốt hơn:

```text
Rollback if any of the following occurs:
application fails to start,
health endpoint returns error,
database connectivity validation fails,
critical application errors appear,
or agreed post-change validation fails.
```

Điều này ngăn operator cố cứu một change thất bại quá lâu.

---

# 27. Rollback không đồng nghĩa root cause analysis

Nếu release mới làm application down:

```text
Priority #1:
restore service

Priority #2:
preserve evidence

Priority #3:
investigate deeply
```

Trong L1 role:

```text
Detect failure
→ execute authorized rollback
→ restore service
→ validate
→ document
→ escalate
```

Bạn không nhất thiết phải debug source code ngay trong outage.

---

# 28. Shift handover

Handover là transfer **operational state**, không phải kể lại ngày làm việc của bạn.

Handover yếu:

```text
Tomcat có vấn đề nhưng giờ chắc OK.
Please check.
```

Handover tốt:

```text
CHG-018 deployed app v2 at 22:12.
Post-deployment health validation failed with HTTP 500.
Rollback to v1 completed at 22:24.
Tomcat active and endpoint returned HTTP 200 after rollback.
No active critical alerts at 22:30.
Application team investigating v2 startup error.
Evidence: /srv/evidence/CHG-018/
Next action: app team to review stack trace and provide new artifact.
Do not retry v2 without new approval.
```

Người nhận ca phải biết:

```text
current state
outstanding risk
what happened
what is pending
who owns it
what must not be done
what to monitor
```

---

# 29. L1 vs escalation boundary

Đây là tư duy bạn cần áp dụng từ Day 18 trở đi.

| Tình huống                                                       | L1                                  |
| ---------------------------------------------------------------- | ----------------------------------- |
| Job exit non-zero và runbook có restart/retry procedure          | Có thể thực hiện theo procedure     |
| Dependency chưa ready                                            | Stop/retry theo runbook             |
| Ticket chưa approved                                             | Không implement                     |
| Host/target không khớp ticket                                    | Stop                                |
| Known rollback procedure được phê duyệt                          | Có thể rollback                     |
| Unknown production data corruption                               | Preserve evidence + escalate        |
| SQL destructive fix chưa được approved                           | Không tự sửa                        |
| Application stack trace chỉ ra code defect                       | App team escalation                 |
| Database lock cần terminate session nhưng runbook không cho phép | DBA/escalation                      |
| Change vượt window                                               | Stop/escalate/change owner decision |
| Rollback thất bại                                                | Incident/escalation ngay            |

---

# 30. LAB CHÍNH DAY 18

Bây giờ chúng ta thực hiện Assignment/Lab đúng syllabus:

> **Execute an approved batch/release scenario and produce a complete ticket, evidence pack, rollback record, and handover note.** 00_MASTER_SYLLABUS_Java_Fresher…

Lab sẽ tích hợp:

```text
Linux
systemd
Java/Tomcat
PostgreSQL
HTTP validation
batch execution
change control
evidence
rollback
handover
```

---

# 31. Safety trước khi thực hành

**Lab/non-production only.**

Bạn sắp thực hiện các hành động có thể:

```text
restart service
replace application artifact
temporarily cause application outage
run scheduled jobs
modify systemd units
```

Trước khi làm:

```bash
hostnamectl
```

xác nhận đúng machine.

Kiểm tra service:

```bash
systemctl status tomcat
```

Tên service thực tế của bạn có thể không phải `tomcat`.

Có thể là:

```text
tomcat
tomcat9
tomcat10
myapp
java-app
```

**Không copy tên service của bài nếu environment của bạn khác.**

Tương tự Tomcat base path có thể khác nhau.

Lab bên dưới dùng placeholders.

---

# 32. Lab scenario

Change ID:

```text
CHG-D18-001
```

Scenario:

```text
Current version: v1
Candidate version: v2

Change:
Deploy v2 during approved maintenance window.

After successful release:
run post-release batch.

Batch dependencies:
PostgreSQL accepting connections
Java application healthy
required input ready

Failure condition:
if release validation fails → rollback to v1.

Required outputs:
change ticket
execution evidence
batch exit code
rollback evidence if invoked
handover note
```

---

# 33. Chuẩn bị evidence directory

Dùng non-sensitive directory:

```bash
mkdir -p "$HOME/day18-evidence/CHG-D18-001"

cd "$HOME/day18-evidence/CHG-D18-001"

pwd
```

Capture basic context:

```bash
{
    echo "Change: CHG-D18-001"
    echo "Timestamp: $(date --iso-8601=seconds)"
    echo "Hostname: $(hostname -f 2>/dev/null || hostname)"
    echo "User: $(id -un)"
    uname -a
} | tee 00-context.txt
```

Expected invariant:

```text
Evidence identifies:
change
timestamp
host
operator
system context
```

Không cần output phải giống chính xác bài.

---

# 34. Tạo change ticket giả lập

Ticket của bạn nên có dạng:

```text
Change ID: CHG-D18-001

Title:
Deploy Java application v2 and run approved post-release batch

Environment:
LAB

Target:
<your hostname>

Reason:
Practice Day 18 controlled release and batch execution.

Current version:
v1

Target version:
v2

Planned window:
<start> - <end>

Impact:
Short application interruption during Tomcat restart.

Risk:
Candidate artifact may fail startup or post-deployment validation.

Prerequisites:
- ticket approved
- correct target confirmed
- previous artifact available
- PostgreSQL healthy
- sufficient disk space
- monitoring available
- rollback procedure ready

Implementation:
1. Capture baseline.
2. Back up current artifact.
3. Deploy approved candidate.
4. Restart/start Tomcat safely.
5. Validate service/log/port/HTTP/database.
6. Run approved batch.
7. Capture exit code and evidence.

Success criteria:
- service active
- expected port listening
- endpoint validation succeeds
- database dependency healthy
- batch exit code 0
- no unexpected critical alert

Rollback trigger:
Any mandatory validation fails.

Rollback:
Restore previous artifact, restart service, validate known-good state.

Approval:
APPROVED FOR LAB

Operator:
<your user>

Evidence path:
~/day18-evidence/CHG-D18-001
```

Trong production thật, bạn **không tự điền `APPROVED` cho mình**. Đây chỉ là simulation của lab.

---

# 35. Pre-change validation

## Identity và target

```bash
date --iso-8601=seconds
hostname -f
id
```

Lưu evidence:

```bash
{
    date --iso-8601=seconds
    hostname -f 2>/dev/null || hostname
    id
} | tee 01-precheck-identity.txt
```

---

# 36. Capacity check

```bash
df -h
```

Đặc biệt filesystem chứa:

```text
Tomcat
logs
application artifact
```

Evidence:

```bash
df -h | tee 02-precheck-disk.txt
```

Nếu disk gần đầy, **không mặc định tiếp tục release**.

Ví dụ:

```text
/ 99%
```

có thể gây:

```text
artifact copy failure
Tomcat logging failure
temp extraction failure
database problems
service startup failure
```

---

# 37. PostgreSQL pre-check

Nếu database local:

```bash
pg_isready -h 127.0.0.1 -p 5432
```

Capture:

```bash
pg_isready -h 127.0.0.1 -p 5432 \
    | tee 03-precheck-postgresql.txt

rc=${PIPESTATUS[0]}

echo "pg_isready rc=$rc" \
    | tee -a 03-precheck-postgresql.txt
```

Chú ý tại sao không dùng đơn giản:

```bash
pg_isready | tee file
echo $?
```

Vì `$?` lúc đó có thể phản ánh `tee`, không nhất thiết `pg_isready`.

`PIPESTATUS` trong Bash giúp kiểm tra các thành phần pipeline.

---

# 38. Application baseline

Giả sử service name là:

```bash
SERVICE=tomcat
```

Bạn phải thay bằng service thật.

```bash
systemctl status "$SERVICE" --no-pager
```

Capture:

```bash
systemctl status "$SERVICE" --no-pager \
    | tee 04-precheck-service.txt
```

Active state:

```bash
systemctl is-active "$SERVICE"
echo $?
```

Port:

```bash
ss -lntp
```

Có thể lọc port thực tế của app:

```bash
ss -lntp | grep ':8080 '
```

Không mặc định port 8080 nếu Day 13 của bạn đã cấu hình port khác.

---

# 39. HTTP pre-check

Giả sử lab của bạn có endpoint:

```bash
APP_URL='http://127.0.0.1:8080/myapp/health'
```

Thay bằng endpoint thật.

```bash
curl \
  --fail \
  --silent \
  --show-error \
  --connect-timeout 3 \
  --max-time 10 \
  "$APP_URL"
```

Capture:

```bash
curl \
  --fail \
  --silent \
  --show-error \
  --connect-timeout 3 \
  --max-time 10 \
  "$APP_URL" \
  | tee 05-precheck-http.txt

rc=${PIPESTATUS[0]}

echo "curl rc=$rc" | tee -a 05-precheck-http.txt
```

Nếu baseline đang fail:

```text
STOP
```

Bạn không nên bắt đầu một change bình thường rồi sau đó không biết lỗi đến từ trước hay sau change.

Đây gọi là **establishing the baseline**.

---

# 40. Artifact backup

Ví dụ Tomcat deployment path của environment bạn là:

```text
$CATALINA_BASE/webapps/myapp.war
```

Không giả định `$CATALINA_BASE` của bạn.

Đầu tiên inspect:

```bash
echo "$CATALINA_BASE"
```

Nếu variable không được set trong interactive shell, tìm từ service/configuration hiện tại chứ không đoán.

Ví dụ sau khi xác nhận:

```bash
CURRENT_WAR=/actual/path/myapp.war
BACKUP_WAR=/actual/backup/path/myapp-v1-before-CHG-D18-001.war
```

Backup:

```bash
sudo cp --preserve=mode,ownership,timestamps \
    "$CURRENT_WAR" \
    "$BACKUP_WAR"
```

Validation:

```bash
sudo ls -lh "$CURRENT_WAR" "$BACKUP_WAR"
```

Checksum:

```bash
sudo sha256sum "$CURRENT_WAR" "$BACKUP_WAR"
```

Expected invariant:

```text
pre-change artifact exists
backup exists
hashes match
rollback copy is available
```

---

# 41. Implementation phase

**Cảnh báo:** bước này gây thay đổi service availability. Chỉ thực hiện trong lab và approved window.

Candidate artifact:

```bash
CANDIDATE=/path/to/approved/myapp-v2.war
```

Trước tiên:

```bash
ls -lh "$CANDIDATE"
sha256sum "$CANDIDATE"
```

Artifact phải đúng với release record.

Không deploy file chỉ vì tên là:

```text
latest.war
final.war
new.war
```

Production release nên có immutable version/checksum traceability.

---

# 42. Stop service

Nếu release procedure đã approved là stop-copy-start:

```bash
sudo systemctl stop "$SERVICE"
```

Validation:

```bash
systemctl is-active "$SERVICE"
```

Expected:

```text
inactive
```

Không dùng:

```bash
sudo kill -9 <java-pid>
```

như cách shutdown bình thường.

SIGKILL bỏ qua graceful shutdown và chỉ nên dùng khi justified/escalated.

---

# 43. Deploy artifact

```bash
sudo cp "$CANDIDATE" "$CURRENT_WAR"
```

Verify:

```bash
sudo sha256sum "$CURRENT_WAR" "$CANDIDATE"
```

Expected invariant:

```text
deployed artifact hash == approved candidate hash
```

---

# 44. Start service

```bash
sudo systemctl start "$SERVICE"
```

Ngay sau đó:

```bash
systemctl status "$SERVICE" --no-pager
```

và:

```bash
systemctl is-active "$SERVICE"
```

Lưu:

```bash
systemctl status "$SERVICE" --no-pager \
    | tee 10-postchange-service.txt
```

---

# 45. Logs

Không nhìn log bằng cách:

```bash
cat huge-log-file
```

Ưu tiên scoped evidence.

Systemd:

```bash
journalctl -u "$SERVICE" --since "-5 minutes" --no-pager
```

Nếu Tomcat có application logs, xem log phù hợp:

```bash
tail -n 100 /actual/tomcat/log/path
```

Bạn tìm:

```text
startup errors
BindException
OutOfMemoryError
SQLException
authentication failures
configuration failures
missing class
stack trace
deployment failure
```

Không phải chỉ grep `ERROR` rồi kết luận mọi thứ.

---

# 46. Post-change port validation

```bash
ss -lntp
```

hoặc port cụ thể:

```bash
ss -lntp | grep ':8080 '
```

Nếu process active nhưng port không listen:

```text
systemd layer may be OK
application layer is not necessarily OK
```

Troubleshooting path:

```text
systemd
 ↓
Java process
 ↓
Tomcat startup
 ↓
connector bind
 ↓
application deployment
```

---

# 47. Post-change HTTP validation

```bash
curl \
  --fail \
  --silent \
  --show-error \
  --connect-timeout 3 \
  --max-time 10 \
  "$APP_URL"
```

Lưu evidence:

```bash
curl \
  --fail \
  --silent \
  --show-error \
  --connect-timeout 3 \
  --max-time 10 \
  "$APP_URL" \
  | tee 11-postchange-http.txt

curl_rc=${PIPESTATUS[0]}

echo "curl_rc=$curl_rc" \
    | tee -a 11-postchange-http.txt
```

Nếu:

```text
curl_rc != 0
```

đây là rollback candidate theo plan.

---

# 48. Database post-check

```bash
pg_isready -h 127.0.0.1 -p 5432
```

Nhưng nhớ:

```text
pg_isready OK
```

chỉ có nghĩa PostgreSQL đang accepting connections.

Nếu release thay đổi JDBC configuration, cần application-level validation nữa.

Ví dụ log hoặc endpoint chứng minh application thực sự query database được.

---

# 49. Monitoring post-check

Từ Day 17:

```text
Prometheus targets
Grafana metrics
alert state
host metrics
application metrics
```

Bạn nên xác minh:

```text
CPU không spike bất thường
memory không tăng bất thường
disk không đầy
service target vẫn UP
critical alert không xuất hiện
```

Đây là lý do Day 17 đứng ngay trước Day 18. Monitoring là một phần của **post-change verification**, không chỉ là dashboard để nhìn cho đẹp.

---

# 50. Batch script của Day 18

Bây giờ tạo batch dependency-aware.

Ví dụ directory:

```bash
sudo mkdir -p /opt/javaops/bin
sudo mkdir -p /var/lib/javaops-batch/input
sudo mkdir -p /var/lib/javaops-batch/output
```

Nếu service account của bạn là `appsvc`:

```bash
sudo chown -R appsvc:appsvc \
    /var/lib/javaops-batch
```

Không tự chown production paths tùy tiện.

---

# 51. Batch script mẫu

```bash
#!/usr/bin/env bash

set -Eeuo pipefail

JOB_NAME="day18-post-release"
RUN_ID="$(date -u +%Y%m%dT%H%M%SZ)"

INPUT_FLAG="/var/lib/javaops-batch/input/input.ready"
OUTPUT_DIR="/var/lib/javaops-batch/output"

DB_HOST="${DB_HOST:-127.0.0.1}"
DB_PORT="${DB_PORT:-5432}"

APP_HEALTH_URL="${APP_HEALTH_URL:-http://127.0.0.1:8080/myapp/health}"

log() {
    printf '%s job=%s run_id=%s %s\n' \
        "$(date --iso-8601=seconds)" \
        "$JOB_NAME" \
        "$RUN_ID" \
        "$*"
}

fail() {
    local rc="$1"
    shift
    log "status=FAILED rc=$rc message=\"$*\""
    exit "$rc"
}

log "status=STARTED"

if [[ ! -f "$INPUT_FLAG" ]]; then
    fail 10 "Required input flag not found: $INPUT_FLAG"
fi

if ! pg_isready \
       -q \
       -h "$DB_HOST" \
       -p "$DB_PORT" \
       -t 3
then
    pg_rc=$?
    fail 30 "PostgreSQL not ready; pg_isready_rc=$pg_rc"
fi

if ! curl \
       --fail \
       --silent \
       --show-error \
       --connect-timeout 3 \
       --max-time 10 \
       "$APP_HEALTH_URL" \
       >/dev/null
then
    curl_rc=$?
    fail 20 "Application health validation failed; curl_rc=$curl_rc"
fi

mkdir -p "$OUTPUT_DIR"

OUTPUT_FILE="${OUTPUT_DIR}/${JOB_NAME}-${RUN_ID}.done"

{
    echo "job=$JOB_NAME"
    echo "run_id=$RUN_ID"
    echo "completed_at=$(date --iso-8601=seconds)"
} > "$OUTPUT_FILE"

log "output=$OUTPUT_FILE"
log "status=SUCCESS rc=0"

exit 0
```

---

# 52. Một subtle Bash bug trong script trên

Đây là phần nâng cao rất đáng hiểu.

Đoạn:

```bash
if ! pg_isready ...; then
    pg_rc=$?
fi
```

có một vấn đề nếu bạn muốn lấy **original exit code của `pg_isready`**.

Dấu `!` đảo success/failure condition, do đó `$?` trong body không còn là cách tốt để preserve chính xác original code.

Cách rõ ràng hơn:

```bash
set +e
pg_isready \
    -q \
    -h "$DB_HOST" \
    -p "$DB_PORT" \
    -t 3
pg_rc=$?
set -e

if (( pg_rc != 0 )); then
    fail 30 "PostgreSQL not ready; pg_isready_rc=$pg_rc"
fi
```

Hoặc sử dụng một construct không bị `set -e` phá:

```bash
pg_rc=0

pg_isready \
    -q \
    -h "$DB_HOST" \
    -p "$DB_PORT" \
    -t 3 || pg_rc=$?

if (( pg_rc != 0 )); then
    fail 30 "PostgreSQL not ready; pg_isready_rc=$pg_rc"
fi
```

Đây là version nên dùng trong lab.

Tương tự curl:

```bash
curl_rc=0

curl \
  --fail \
  --silent \
  --show-error \
  --connect-timeout 3 \
  --max-time 10 \
  "$APP_HEALTH_URL" \
  >/dev/null || curl_rc=$?

if (( curl_rc != 0 )); then
    fail 20 "Application health failed; curl_rc=$curl_rc"
fi
```

Đây là ví dụ rất tốt cho lý do fresher phải **hiểu exit code**, không chỉ copy script.

---

# 53. Validate script trước khi schedule

Syntax:

```bash
bash -n /opt/javaops/bin/day18-post-release.sh
```

Permissions:

```bash
ls -l /opt/javaops/bin/day18-post-release.sh
```

Test manually dưới đúng account:

```bash
sudo -u appsvc \
    /opt/javaops/bin/day18-post-release.sh
```

Sau đó:

```bash
echo $?
```

Không schedule một script chưa từng được test manually.

---

# 54. Failure Injection 1 — missing dependency

Đảm bảo flag không tồn tại:

```bash
sudo rm -f /var/lib/javaops-batch/input/input.ready
```

**Lab only:** command này intentionally xóa marker file, không phải business data.

Run:

```bash
sudo -u appsvc \
    /opt/javaops/bin/day18-post-release.sh
```

Expected:

```text
status=FAILED
rc=10
Required input flag not found
```

Kiểm tra:

```bash
echo $?
```

Expected invariant:

```text
10
```

Điểm học:

```text
scheduler triggered successfully
but dependency gate blocked processing
```

Đây là behavior đúng.

---

# 55. Failure Injection 2 — create dependency

```bash
sudo -u appsvc touch \
    /var/lib/javaops-batch/input/input.ready
```

Run:

```bash
sudo -u appsvc \
    /opt/javaops/bin/day18-post-release.sh
```

Nếu app + DB healthy:

```text
status=SUCCESS rc=0
```

Kiểm tra:

```bash
echo $?
```

Expected:

```text
0
```

Và:

```bash
ls -lh /var/lib/javaops-batch/output
```

---

# 56. Failure Injection 3 — database dependency failure

**Không stop PostgreSQL trên shared/prod system.**

Trong lab an toàn, thay tạm DB port bằng port không có service:

```bash
DB_PORT=65432 \
sudo -u appsvc \
    /opt/javaops/bin/day18-post-release.sh
```

Nếu `sudo` environment handling của distro không propagate biến như vậy, dùng approved method hoặc execute một lab copy của script với configuration riêng.

Mục tiêu là tạo:

```text
pg_isready failure
      ↓
batch exits non-zero
      ↓
downstream processing must not execute
```

Không cần phá database thật.

---

# 57. Schedule batch với systemd

Service:

```ini
[Unit]
Description=Day 18 Post-Release Batch

[Service]
Type=oneshot
User=appsvc
Environment="DB_HOST=127.0.0.1"
Environment="DB_PORT=5432"
Environment="APP_HEALTH_URL=http://127.0.0.1:8080/myapp/health"
ExecStart=/opt/javaops/bin/day18-post-release.sh
```

Trong production, không đặt password trực tiếp trong unit file.

Timer ví dụ:

```ini
[Unit]
Description=Day 18 Batch Timer

[Timer]
OnCalendar=*-*-* 02:30:00
Persistent=true

[Install]
WantedBy=timers.target
```

Validate:

```bash
sudo systemd-analyze verify \
    /etc/systemd/system/day18-post-release.service \
    /etc/systemd/system/day18-post-release.timer
```

Reload:

```bash
sudo systemctl daemon-reload
```

Enable:

```bash
sudo systemctl enable --now day18-post-release.timer
```

Inspect:

```bash
systemctl list-timers --all
```

Manual trigger để test không cần đợi 02:30:

```bash
sudo systemctl start day18-post-release.service
```

Inspect:

```bash
systemctl status day18-post-release.service --no-pager
```

Logs:

```bash
journalctl \
    -u day18-post-release.service \
    --since "-10 minutes" \
    --no-pager
```

Systemd có thể đánh giá service success/failure dựa trên process exit status; mặc định exit `0` là clean success, và `SuccessExitStatus=` có thể mở rộng tập status được coi là success nếu có requirement cụ thể. [man7](https://www.man7.org/linux/man-pages/man5/systemd.service.5.html?utm_source=chatgpt.com)

Trong lab này **không cấu hình `SuccessExitStatus=10`**, vì code 10 của chúng ta thực sự là dependency failure và ta muốn scheduler nhìn thấy failure.

---

# 58. Failure Injection 4 — release failure và rollback

Đây là phần quan trọng nhất của Day 18.

Giả sử sau deployment:

```bash
systemctl is-active "$SERVICE"
```

cho:

```text
active
```

nhưng:

```bash
curl --fail "$APP_URL"
```

failure.

Theo ticket:

```text
mandatory health validation failed
→ rollback trigger met
```

Không cố chỉnh lung tung v2 ngoài change plan.

---

# 59. Rollback procedure

**Cảnh báo:** chỉ thực hiện trên lab/authorized environment.

Stop:

```bash
sudo systemctl stop "$SERVICE"
```

Restore previous artifact:

```bash
sudo cp "$BACKUP_WAR" "$CURRENT_WAR"
```

Verify:

```bash
sudo sha256sum "$BACKUP_WAR" "$CURRENT_WAR"
```

Start:

```bash
sudo systemctl start "$SERVICE"
```

Validate:

```bash
systemctl is-active "$SERVICE"
```

Port:

```bash
ss -lntp
```

HTTP:

```bash
curl \
  --fail \
  --silent \
  --show-error \
  "$APP_URL"
```

DB:

```bash
pg_isready -h 127.0.0.1 -p 5432
```

Logs:

```bash
journalctl \
    -u "$SERVICE" \
    --since "-10 minutes" \
    --no-pager
```

Monitoring:

```text
Prometheus target UP
no unexpected critical alert
metrics returned toward baseline
```

Rollback chỉ được coi là **successful** sau validation.

Không phải sau:

```bash
cp old.war new.war
```

---

# 60. Rollback evidence record

Ví dụ:

```text
Change:
CHG-D18-001

Rollback trigger:
Post-deployment mandatory HTTP health validation failed.

Failure observed:
2026-10-12T22:17:41+07:00

Action:
Stopped Tomcat.
Restored pre-change v1 WAR.
Verified restored WAR SHA-256.
Started Tomcat.

Recovery validation:
systemd service active.
Expected application port listening.
HTTP validation successful.
PostgreSQL accepting connections.
No critical monitoring alert after observation period.

Rollback completed:
2026-10-12T22:25:13+07:00

Current state:
Service restored on v1.

Next owner:
Application team.

Outstanding action:
Investigate v2 application startup/health failure before resubmission.
```

Đây mới là rollback record.

---

# 61. Evidence pack hoàn chỉnh

Sau lab, directory của bạn nên logic giống:

```text
CHG-D18-001/
├── 00-context.txt
├── 01-precheck-identity.txt
├── 02-precheck-disk.txt
├── 03-precheck-postgresql.txt
├── 04-precheck-service.txt
├── 05-precheck-http.txt
├── 06-artifact-checksum.txt
├── 07-deployment.txt
├── 08-service-start.txt
├── 09-log-validation.txt
├── 10-postchange-service.txt
├── 11-postchange-http.txt
├── 12-batch-output.txt
├── 13-monitoring-validation.txt
├── rollback.txt
└── handover.txt
```

Không bắt buộc tên file y hệt.

**Expected invariant** là evidence pack có thể tái dựng:

```text
before
→ change
→ after
→ failure if any
→ rollback
→ restored state
```

---

# 62. Shift handover note hoàn chỉnh

Template thực hành:

```text
SHIFT HANDOVER

Date/Time:
2026-10-12 23:00 ICT

Change:
CHG-D18-001

System:
<hostname / environment>

Activity:
Java application v2 release plus post-release batch.

Current state:
Application currently running v1 after rollback.
Tomcat active.
Application endpoint healthy.
PostgreSQL available.
Monitoring shows no current critical alert.

Timeline:
22:05 - Pre-check completed.
22:10 - v2 deployment started.
22:14 - Tomcat started.
22:17 - Mandatory HTTP validation failed.
22:18 - Rollback initiated.
22:25 - v1 restored and validated.
22:30 - Monitoring validation completed.

Batch:
Post-release batch was NOT executed because release success gate was not met.

Outstanding issue:
v2 health validation failure.

Owner:
Application team.

Evidence:
~/day18-evidence/CHG-D18-001/

Next action:
Application team to analyze v2 logs/artifact.
Do not retry v2 without approved follow-up change.

Risk:
No known immediate service impact while remaining on v1.

Escalation:
<Application support queue/team according to lab procedure>
```

Điểm đặc biệt quan trọng:

```text
Batch was NOT executed
```

là đúng.

Không phải failure.

Release gate chưa pass nên downstream batch không được phép chạy.

---

# 63. Troubleshooting framework Day 18

Khi batch/release lỗi, không nhảy ngay vào “restart thử”.

Hãy đi theo:

```text
SYMPTOM
↓
What was expected?

RECENT CHANGE
↓
What just happened?

SCOPE
↓
one job?
one service?
one server?
database?
all clients?

FAILING LAYER
↓
scheduler?
shell?
OS?
network?
Java?
Tomcat?
DB?
application?

EVIDENCE
↓
exit code
systemctl
journal
application log
ss
curl
pg_isready
monitoring

HYPOTHESES
↓
rank by evidence

SAFE TEST
↓
lowest blast radius

ROOT CAUSE / WORKING CAUSE
↓
within L1?

REMEDIATION
↓
runbook action or rollback

VALIDATION
↓
prove restored state

EVIDENCE PRESERVATION
↓
ticket/logs/timestamps

PREVENTION / ESCALATION
```

---

# 64. Troubleshooting scenario: “cron did not run my Java job”

Bạn nên tách layer.

```text
Was cron running?
        ↓
Was entry installed?
        ↓
Did schedule match?
        ↓
Did cron launch command?
        ↓
Which user?
        ↓
Which environment?
        ↓
Did script start?
        ↓
Exit code?
        ↓
Java executable available?
        ↓
JAVA_HOME/PATH?
        ↓
Application error?
```

Commands có thể bao gồm:

```bash
crontab -l
```

```bash
systemctl status crond
```

hoặc trên distro khác:

```bash
systemctl status cron
```

Tên unit phụ thuộc distribution; không được đoán.

Logs có thể ở:

```bash
journalctl -u crond
```

hoặc:

```bash
journalctl -u cron
```

Distro/configuration khác nhau.

Cron syntax cũng có những corner case quan trọng. Ví dụ khi cả `day-of-month` và `day-of-week` đều bị restrict, một số phổ biến Cronie semantics sẽ trigger khi **một trong hai** field match chứ không phải bắt buộc cả hai. [man7](https://www.man7.org/linux/man-pages/man5/crontab.5.html?utm_source=chatgpt.com)

Đây là lý do schedule phải được review, không “nhìn có vẻ đúng”.

---

# 65. Troubleshooting scenario: batch log nói success nhưng data thiếu

Đừng tin string:

```text
SUCCESS
```

Hãy hỏi:

```text
process exit code?
SQL exit code?
number of expected rows?
output record count?
downstream file generated?
business reconciliation passed?
```

Ví dụ script tệ:

```bash
psql -f job.sql
echo "SUCCESS"
exit 0
```

SQL có thể lỗi nhưng wrapper vẫn return `0`.

Đó là **false success**.

False success đôi khi nguy hiểm hơn obvious failure vì scheduler sẽ tiếp tục downstream workflow.

---

# 66. Troubleshooting scenario: application active nhưng health fail

Path:

```text
systemctl active
       ↓
Java process?
       ↓
port listening?
       ↓
Tomcat deployment succeeded?
       ↓
HTTP response?
       ↓
application startup errors?
       ↓
database connectivity?
       ↓
external dependency?
```

Commands:

```bash
systemctl status "$SERVICE" --no-pager
```

```bash
pgrep -af java
```

```bash
ss -lntp
```

```bash
curl -v "$APP_URL"
```

```bash
journalctl -u "$SERVICE" --since "-10 minutes"
```

Tuy nhiên `curl -v` có thể hiển thị sensitive headers trong một số context; cẩn thận trước khi đưa output vào ticket.

---

# 67. Idempotency

**Supplementary / Beyond explicit syllabus**, nhưng cực kỳ quan trọng cho batch.

Một job **idempotent** là job có thể chạy lại mà không gây duplicate/unwanted side effects trong thiết kế phù hợp.

Ví dụ nguy hiểm:

```text
Run #1:
credit customer 100 USD

network fails before scheduler sees result

scheduler retries

Run #2:
credit customer another 100 USD
```

Technical retry tạo business corruption.

Do đó:

```text
retry-safe?
idempotent?
transactional?
checkpointed?
deduplicated?
```

phải được hiểu trước khi operator bấm “Run again”.

**Không bao giờ assume “batch failed → retry ngay”.**

---

# 68. Concurrency control

Cũng là supplementary nhưng rất thực tế.

Nếu batch 02:00 vẫn chạy tới 03:00, scheduler lại chạy instance mới thì:

```text
Instance A
       +
Instance B
       ↓
duplicate processing
lock contention
resource exhaustion
data corruption
```

Có thể dùng scheduler-native locking hoặc `flock`.

Ví dụ Linux:

```bash
flock -n /run/day18-batch.lock \
    /opt/javaops/bin/day18-post-release.sh
```

Nhưng phải kiểm tra `flock` có sẵn và thống nhất behavior với runbook.

Không tự thêm locking production mà không change control.

---

# 69. Timeout

Không để dependency check treo vô hạn.

Tệ:

```bash
curl http://app/health
```

Tốt hơn:

```bash
curl \
  --connect-timeout 3 \
  --max-time 10 \
  --fail \
  http://app/health
```

Tương tự PostgreSQL:

```bash
pg_isready -t 3
```

Operational principle:

```text
Every external dependency call should have a bounded failure behavior.
```

---

# 70. Retry

Retry chỉ hợp lý nếu:

```text
failure likely transient
operation retry-safe
retry bounded
delay/backoff defined
final failure visible
```

Không:

```bash
while true; do
    run_job
done
```

Đây có thể biến incident thành outage lớn hơn.

---

# 71. Security & least privilege

Batch nên chạy dưới:

```text
dedicated service account
```

không phải mặc định:

```text
root
```

Ví dụ:

```ini
[Service]
User=appsvc
```

Chỉ cấp quyền:

```text
read required input
write required output
connect required DB
read required config
```

Không:

```bash
chmod -R 777 /opt
```

Không:

```bash
sudo bash batch.sh
```

chỉ để “cho nó chạy được”.

Permission failure là evidence để tìm proper ownership/ACL, không phải lý do bỏ least privilege.

---

# 72. Secrets trong batch

Không viết:

```bash
PGPASSWORD='SuperSecret123'
```

vào script checked into Git.

Không viết plaintext password vào change ticket.

Không:

```bash
curl -H 'Authorization: Bearer real-token'
```

rồi capture terminal vào evidence.

Tùy environment, secrets có thể đến từ:

```text
protected environment file
secret manager
credential store
instance role
approved service configuration
```

Cơ chế cụ thể tùy organization.

---

# 73. Operational anti-patterns

Những pattern bạn phải nhận ra trong thi hoặc thực tế:

| Anti-pattern                                 | Tại sao sai                    |
| -------------------------------------------- | ------------------------------ |
| “Ticket pending nhưng change nhỏ nên cứ làm” | Vi phạm approval gate          |
| “Service active nên release success”         | Chưa validate app              |
| “Không backup vì rollback đơn giản”          | Không có recovery artifact     |
| “Batch fail thì run lại”                     | Có thể duplicate side effect   |
| `chmod 777` để fix                           | Phá least privilege            |
| Run batch as root                            | Blast radius lớn               |
| Screenshot toàn bộ `env`                     | Có thể leak secrets            |
| Dựa vào time thay dependency state           | Race condition                 |
| Bỏ qua exit code                             | False success                  |
| `curl URL` không kiểm HTTP failure behavior  | Có thể kết luận sai            |
| Change xong không monitor                    | Miss delayed degradation       |
| Handover “please monitor”                    | Không truyền operational state |
| Debug production ngoài approved scope        | Tăng blast radius              |
| Rollback mà không validate                   | Không biết service đã phục hồi |

---

# 74. Change ticket hoàn chỉnh nên đạt chuẩn nào?

Bạn nên tự review ticket theo câu hỏi:

```text
Is target unambiguous?
Is environment unambiguous?
Is artifact/version immutable?
Is approval present?
Is window known?
Is impact stated?
Is risk stated?
Are prerequisites testable?
Are implementation steps executable?
Are success criteria measurable?
Are rollback triggers objective?
Is rollback executable?
Is post-change monitoring included?
Is ownership/escalation clear?
```

Nếu một người khác không thể execute ticket an toàn từ nội dung ticket + runbook, ticket có thể chưa đủ tốt.

---

# 75. Evidence quality

So sánh:

```text
Bad:
Service OK.
```

với:

```text
2026-10-12T22:25:03+07:00
Host: app-lab-01
Command: systemctl is-active tomcat
Result: active

2026-10-12T22:25:10+07:00
Command: curl --fail ...
Result: exit code 0

2026-10-12T22:25:17+07:00
Command: pg_isready ...
Result: accepting connections, rc=0
```

Evidence thứ hai cho phép người khác kiểm chứng.

---

# 76. Evidence integrity

**Supplementary / Beyond explicit syllabus.**

Với artifact quan trọng:

```bash
sha256sum myapp.war
```

giúp chứng minh:

```text
candidate reviewed
=
candidate deployed
```

Ví dụ ticket:

```text
Approved artifact:
myapp-1.4.2.war

SHA-256:
abc123...
```

Trước deploy:

```bash
sha256sum myapp-1.4.2.war
```

Nếu khác:

```text
STOP
```

Không deploy rồi hỏi sau.

---

# 77. Change freeze và emergency change

**Supplementary.**

Organization thường có concepts như:

```text
standard change
normal change
emergency change
change freeze
```

Chi tiết workflow khác nhau giữa tổ chức.

Điểm fresher phải nhớ:

```text
Emergency
≠
No control
```

Emergency change thường vẫn cần:

```text
authorization
traceability
implementation record
post-change review
```

chỉ có workflow được rút gọn theo policy.

---

# 78. Day 18 trong integrated architecture

Bây giờ hãy đặt Day 18 vào toàn stack:

```text
User / Client
      ↓
DNS / Network
      ↓
AWS SG / VPC
      ↓
EC2 Linux
      ↓
systemd
      ↓
Tomcat / Java
      ↓
Application
      ↓
PostgreSQL
      ↓
Batch processing
      ↓
Prometheus / Grafana
```

Change control bao quanh toàn bộ:

```text
        CHANGE TICKET
              │
      ┌───────┴─────────┐
      │ Approval        │
      │ Window          │
      │ Risk            │
      │ Implementation  │
      │ Validation      │
      │ Rollback        │
      │ Evidence        │
      │ Handover        │
      └─────────────────┘
```

Đây là mental model cốt lõi.

---

# 79. Exam Focus — câu hỏi lý thuyết bạn phải trả lời được

### “Exit code 0 nghĩa là gì?”

Trong normal Unix/Bash command semantics:

```text
successful completion
```

Không có nghĩa business result chắc chắn đúng nếu application thiết kế exit codes tệ.

---

### “Một batch dependency check để làm gì?”

Để ngăn downstream processing chạy khi prerequisite chưa đạt:

```text
upstream job
file
database
service
application
approval state
```

---

### “Service active có chứng minh release thành công không?”

Không.

Cần:

```text
process
port
logs
HTTP/application
DB
monitoring
business validation
```

tùy change.

---

### “Nếu ticket chưa approved?”

```text
Do not implement.
```

Escalate/contact change owner theo procedure.

---

### “Nếu mandatory validation fail?”

Theo approved plan:

```text
stop
preserve evidence
rollback if trigger met
validate restored state
update ticket
escalate as required
```

---

### “Rollback thành công khi nào?”

Không phải khi restore file xong.

Phải:

```text
known-good service restored
+
validation passed
```

---

### “Handover phải chứa gì?”

Ít nhất:

```text
current state
what happened
outstanding issue/risk
next action
owner
evidence/reference
restrictions such as do-not-retry
```

---

# 80. Practical exam mindset

Nếu practical exam cho:

> “Deploy approved release and execute scheduled post-release job.”

Đừng lao vào:

```bash
sudo systemctl stop tomcat
```

Thứ tự bạn nên thể hiện là:

```text
1. Read ticket.
2. Verify approval.
3. Verify window.
4. Verify target.
5. Review implementation.
6. Review rollback.
7. Capture baseline.
8. Verify prerequisites.
9. Backup/confirm rollback artifact.
10. Implement.
11. Capture exit status.
12. Validate service.
13. Validate application.
14. Validate dependencies.
15. Run batch only after release gate passes.
16. Validate batch output.
17. Check monitoring.
18. Record evidence.
19. Update change.
20. Handover.
```

Đây là tư duy mà examiner thường muốn thấy ở operations role, không phải chỉ terminal speed.

---

# 81. Knowledge Check

Hãy tự trả lời trước khi xem đáp án.

**Q1.** Batch A chạy lúc 01:00, Batch B lúc 01:10. B có thể coi A hoàn tất chỉ vì khác schedule 10 phút không?

**Đáp án:** Không. Dependency phải kiểm tra state/success condition chứ không chỉ thời gian.

**Q2.** `$?` là gì?

**Đáp án:** Exit status của command/pipeline gần nhất theo shell semantics.

**Q3.** `curl` kết nối thành công đến HTTP server và nhận HTTP 500. Có nên coi command mặc định chắc chắn non-zero không?

**Đáp án:** Không nên giả định. Dùng `--fail`/`--fail-with-body` hoặc kiểm tra HTTP response code theo validation contract. [curl.se](https://curl.se/docs/manpage.html?code=xxx&utm_source=chatgpt.com)

**Q4.** `systemctl is-active tomcat` trả `active`. Change close được chưa?

**Đáp án:** Chưa. Phải thực hiện mandatory post-change validations.

**Q5.** Ticket đúng host nhưng chưa approved?

**Đáp án:** Stop. Không implement.

**Q6.** Release fail, rollback WAR thành công nhưng endpoint vẫn 500. Rollback successful chưa?

**Đáp án:** Chưa.

**Q7.** Batch exit `0` nhưng output count sai. Success không?

**Đáp án:** Technical process có thể exit 0 nhưng business validation fail; phải xử lý theo success criteria.

**Q8.** Job fail giữa chừng. Có retry luôn không?

**Đáp án:** Không. Phải xác minh retry safety/idempotency và procedure.

---

# 82. Independent Practical Challenge

Sau khi hoàn thành guided lab, hãy tự làm challenge này mà **không nhìn lại commands trước**.

Bạn có một Java/Tomcat application đang chạy version `v1`.

Approved change yêu cầu:

```text
Deploy v2.
Validate service.
Validate DB.
Validate endpoint.
If successful, execute post-release batch.
If validation fails, restore v1.
Produce complete evidence and handover.
```

Trong lab, chủ động inject một trong hai lỗi:

```text
application health URL sai
```

hoặc:

```text
batch DB port sai
```

Bạn phải tự chứng minh được:

```text
failure detected
correct exit status recorded
downstream action prevented
rollback/no-rollback decision justified
final system state validated
ticket updated
handover produced
```

Đừng chỉ gửi cho mentor screenshot.

Hãy gửi **evidence chain**.

---

# 83. Mastery Check Day 18

Bạn chỉ nên coi mình đã thực sự nắm Day 18 khi có thể thực hiện được chuỗi này mà không cần ai nhắc:

```text
REQUEST
   ↓
TICKET
   ↓
APPROVAL
   ↓
WINDOW
   ↓
PRE-CHECK
   ↓
DEPENDENCY CHECK
   ↓
IMPLEMENT
   ↓
EXIT CODE
   ↓
VALIDATE
   ↓
MONITOR
   ↓
SUCCESS?
 ┌───────┴────────┐
YES               NO
 │                 │
Evidence          Stop
 │                 │
Close             Evidence
 │                 │
Handover          Rollback
                   │
                 Validate
                   │
                 Escalate
                   │
                 Handover
```

---

# 84. Những điều bắt buộc bạn cần nhớ sau Day 18

Đây là phần mình muốn bạn thuộc về mặt tư duy:

```text
Access is not authorization.

Scheduled does not mean successfully executed.

Process running does not mean application healthy.

Application healthy does not automatically mean business processing correct.

A log saying SUCCESS is weaker than a meaningful exit code plus validation.

Dependency must be proven, not assumed.

A change requires measurable success criteria.

Rollback must be planned before implementation.

Rollback itself must be validated.

Evidence must prove what happened without exposing secrets.

A failed release should not automatically allow downstream batch execution.

Retry is a change in system state and may be unsafe.

Handover communicates current operational state, ownership, risk, and next action.

When outside L1 scope:
preserve evidence → stabilize if authorized → escalate.
```

Đó chính là năng lực bắt buộc mà syllabus muốn xây dựng ở Day 18: **batch scheduling + dependency/exit-code control + controlled release/change execution + approval + evidence + rollback + handover**. 00_MASTER_SYLLABUS_Java_Fresher…

Về mặt curriculum, sau khi bạn **thực sự chạy lab, lưu evidence và xử lý ít nhất một failure/rollback scenario**, bạn sẽ sẵn sàng chuyển sang **Day 19 — integrated project build Linux + AWS + Java + PostgreSQL + monitoring**, nơi toàn bộ các kỹ năng này được ghép thành một hệ thống end-to-end. 00_MASTER_SYLLABUS_Java_Fresher…
