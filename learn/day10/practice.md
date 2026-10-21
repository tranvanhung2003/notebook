# Day 10 — Module Thực hành

## Infrastructure Monitoring, Centralized Log Investigation, EventLog Analyzer, Firewall Analyzer & Audit Evidence Lab

Phần thực hành Day 10 phải biến toàn bộ kiến thức lý thuyết vừa học thành một workflow vận hành có thể kiểm chứng được. Theo syllabus, bạn không chỉ cần “đăng nhập được EventLog Analyzer/Firewall Analyzer” hay “search được vài dòng log”. Bạn phải chứng minh được toàn bộ chuỗi **source event → collection → parsing → centralized search → correlation → finding → evidence → escalation**, đồng thời phát hiện được khi chính chuỗi logging đó không đáng tin cậy. Infrastructure_Operations_14_Da…

Đặc biệt, Day 10 yêu cầu tạo **known Windows/RHEL/firewall events**, đối chiếu source với analyzer, xử lý một **silent-source fixture**, kiểm tra ingestion health, export evidence, ghi filters/timezone/source IDs/export limitations/pagination/excluded intervals, và tạo một **risk/owner-based finding register**. Acceptance không phải “tool chạy được”, mà là **events phải reconcile hoặc mọi discrepancy phải được giải thích**. Infrastructure_Operations_14_Da…

---

# 1. Mục tiêu tổng thể của Master Lab Day 10

Sau toàn bộ module thực hành này, bạn phải có khả năng tự thực hiện workflow:

```text
Establish scope
     ↓
Inventory log sources
     ↓
Verify time synchronization
     ↓
Generate known events
     ↓
Verify at source
     ↓
Verify collection
     ↓
Verify parsing
     ↓
Verify centralized search
     ↓
Measure ingestion delay
     ↓
Investigate security/operations events
     ↓
Analyze firewall traffic
     ↓
Correlate cross-system timeline
     ↓
Classify finding
     ↓
Assign severity/exposure/business risk
     ↓
Determine owner/escalation
     ↓
Export raw/native evidence
     ↓
Export filtered evidence
     ↓
Reconcile counts
     ↓
Hash artifacts
     ↓
Complete evidence manifest
     ↓
Prepare incident/audit package
```

Đây chính là operational workflow cốt lõi của Day 10.

---

# 2. Lab Environment

## 2.1. Recommended topology

Lab đầy đủ lý tưởng có cấu trúc:

```text
                         ┌───────────────────────┐
                         │   AWS Sandbox         │
                         │ CloudWatch/CloudTrail │
                         └──────────┬────────────┘
                                    │
                                    │ logs/evidence
                                    │
┌────────────┐              ┌───────▼────────────┐
│ Windows    │─────────────►│                    │
│ Server     │ Event Logs   │ EventLog Analyzer  │
└────────────┘              │                    │
                            └─────────┬───────────┘
┌────────────┐                        │
│ RHEL       │─────────────►          │
│ Server     │ syslog/journal         │
└────────────┘                        │
                                      │ Investigation
┌────────────┐                        │ workstation
│ Cisco      │─────────────►          │
│ Switch     │ syslog                 │
└────────────┘                        │
                                      ▼
┌────────────┐              ┌─────────────────────┐
│ FortiGate  │─────────────►│ Firewall Analyzer   │
│ Firewall   │ traffic/logs │                     │
└─────┬──────┘              └─────────────────────┘
      │
      │ traffic
      │
┌─────▼──────┐       ┌──────────────┐
│ Lab Client │──────►│ Lab Server   │
└────────────┘       └──────────────┘
```

Nếu có thêm Nutanix Prism, HPE iLO hoặc BIG-IP, chúng được thêm vào inventory như supplemental log sources.

Syllabus cho phép sử dụng product lab thực tế hoặc version-labelled exports/screenshots/configuration/fault packets khi enterprise product không khả dụng; nhưng phải đánh dấu rõ simulation không tương đương actual product hands-on. Infrastructure_Operations_14_Da…

---

# 3. Safety Rules — bắt buộc trước khi bắt đầu

Day 10 liên quan logs, firewall, security activity và privileged events, nên safety boundary phải rất rõ.

| Loại thao tác                         | Default                                   |
| ------------------------------------- | ----------------------------------------- |
| Search/filter logs                    | Read-only, được ưu tiên                   |
| View device/source status             | Read-only                                 |
| Export logs/report                    | Read-only nhưng có data-handling risk     |
| Generate harmless test event          | Cho phép trong lab                        |
| Generate failed authentication        | Chỉ disposable lab account                |
| Add user vào privileged group         | Chỉ khi explicit lab approval             |
| Stop/restart collector                | Không làm trừ khi failure fixture yêu cầu |
| Disable firewall policy               | Không tự ý làm                            |
| Change production syslog forwarding   | Không làm                                 |
| Clear Windows Event Logs              | **Không làm**                             |
| Purge journal/syslog                  | **Không làm**                             |
| Delete raw evidence                   | **Không làm**                             |
| Disable TLS verification              | **Không làm để “test cho qua”**           |
| Paste passwords/tokens vào transcript | **Không làm**                             |

Một nguyên tắc cực kỳ quan trọng:

> **Cleanup không có nghĩa là xóa evidence.**

Sau khi tạo một Windows event, Linux journal event hay firewall log, **không clear log chỉ để lab “sạch” trở lại**. Log được giữ cho tới khi normal retention/rotation xử lý.

---

# 4. Lab Deliverables cuối cùng

Bạn cần kết thúc Day 10 với các artifact sau:

```text
01_Source_Inventory
02_Time_Baseline
03_Windows_Source_Evidence
04_RHEL_Source_Evidence
05_Cisco_Source_Evidence
06_FortiGate_Source_Evidence
07_EventLogAnalyzer_Exports
08_FirewallAnalyzer_Exports
09_Ingestion_Health
10_Silent_Source_Investigation
11_Correlation_Timeline
12_Finding_Register
13_Incident_Report
14_Evidence_Manifest
15_Hash_Manifest
16_Cleanup_and_Rollback_Record
```

Syllabus chung yêu cầu mỗi evidence item có source/product/version, asset/environment, collection timestamp/timezone, query/window, collector, ticket reference, size/count/hash, transformation history và limitations/errors. Infrastructure_Operations_14_Da…

---

# PART A — PRE-LAB BASELINE

# 5. Step 1 — Tạo Evidence Repository

Ví dụ:

```text
Day10/
├── 00_Control/
├── 01_Raw/
│   ├── Windows/
│   ├── RHEL/
│   ├── Cisco/
│   ├── FortiGate/
│   ├── AWS/
│   ├── EventLogAnalyzer/
│   └── FirewallAnalyzer/
├── 02_Working/
├── 03_Filtered/
├── 04_Screenshots/
├── 05_Reports/
├── 06_Hashes/
└── 07_Manifest/
```

Quy tắc:

```text
01_Raw
→ read-only after collection

02_Working
→ copy của raw để phân tích

03_Filtered
→ derived results

05_Reports
→ reviewer-facing output
```

Không sửa file trong `01_Raw`.

---

# 6. Step 2 — Xác định Investigation ID

Ví dụ:

```text
Investigation ID:
D10-INC-001

Lab Ticket:
LAB-CHG-D10-001

Investigator:
<your_name>

Reference timezone:
UTC+07:00

Evidence prefix:
D10-EV-
```

Sau đó artifact có thể có naming convention:

```text
D10-EV-001_Windows_Application.evtx
D10-EV-002_RHEL_DAY10_LAB.txt
D10-EV-003_FortiGate_DeniedTraffic.csv
```

Naming convention phải deterministic và dễ audit.

---

# 7. Step 3 — Lập Source Inventory

Một source inventory tối thiểu:

| Source ID  | Asset       | Platform       | Log type              | Collection method | Timezone  | Analyzer          | Expected events             |
| ---------- | ----------- | -------------- | --------------------- | ----------------- | --------- | ----------------- | --------------------------- |
| SRC-WIN-01 | WIN-D10-01  | Windows Server | Event Logs            | Agent/agentless   | UTC+07    | EventLog Analyzer | Security/System/Application |
| SRC-LNX-01 | RHEL-D10-01 | RHEL           | journald/syslog       | syslog/agent      | UTC+07    | EventLog Analyzer | auth/service/custom         |
| SRC-SW-01  | SW-D10-01   | IOS XE/NX-OS   | syslog                | forwarding        | UTC/Local | EventLog Analyzer | interface/auth/config       |
| SRC-FW-01  | FG-D10-01   | FortiGate      | traffic/security      | syslog            | UTC+07    | Firewall Analyzer | allow/deny/policy           |
| SRC-AWS-01 | LAB-AWS     | AWS            | CloudTrail/CloudWatch | API/export        | UTC       | investigation     | API/log events              |

Đừng bỏ qua field:

```text
Expected events
```

Vì nếu không biết source **đáng lẽ phải gửi gì**, rất khó xác định silent source.

---

# PART B — TIME BASELINE

# 8. Step 4 — Verify Windows Time

Read-only:

```powershell
hostname
Get-Date -Format o
Get-TimeZone
w32tm /query /status
```

Bạn đang tìm:

```text
hostname
current timestamp
timezone
time source
synchronization state
```

Ví dụ:

```text
2026-10-21T10:15:22.4829137+07:00
```

ISO 8601 có timezone offset là format tốt cho evidence.

---

# 9. Step 5 — Verify RHEL Time

Read-only:

```bash
hostnamectl
date --iso-8601=seconds
timedatectl
```

Expected:

```text
Local time
Universal time
Time zone
System clock synchronized
NTP service
```

Nếu thấy:

```text
System clock synchronized: no
```

phải ghi thành evidence gap/risk.

Không tự ý sửa NTP trên enterprise host ngoài change scope.

---

# 10. Step 6 — Verify Network Device Time

Cisco/FortiGate nên dùng read-only commands từ runbook của Day 8/9 để xác nhận:

```text
device time
timezone
NTP state
```

Không cần thay đổi clock.

Nếu thiết bị ở UTC và server ở UTC+07, ghi rõ mapping.

Ví dụ:

```text
Cisco source timezone: UTC
Windows source timezone: UTC+07
Investigation reference timezone: UTC+07
```

---

# PART C — EVENTLOG ANALYZER SOURCE VALIDATION

# 11. Hai mô hình collection cần nhận biết

EventLog Analyzer hiện hỗ trợ cả **agent-based** và **agentless** collection; documentation của ManageEngine mô tả agent-based collection với endpoint agent và agentless collection trực tiếp từ server/collector. Mô hình đúng phải lấy từ actual lab configuration, không được assume. [ManageEngine](https://www.manageengine.com/products/eventlog/collect-eventlogs-with-agent.html?utm_source=chatgpt.com)

Do đó trước mỗi source hãy ghi:

```text
Collection Mode:
Agent / Agentless / Syslog / Import / Other
```

Không ghi chung chung:

```text
"ManageEngine collects it."
```

---

# 12. Step 7 — Kiểm tra Windows Source trong EventLog Analyzer

Trước hết **không tạo event**.

Xem source health và xác nhận:

```text
Device name
Source IP
Platform/device type
Last received event
Collection mode
Event channels
Status
```

Nếu source mới được add, việc device xuất hiện trong console chưa đủ chứng minh successful onboarding.

Successful onboarding phải là:

```text
Source present
     ↓
Known event generated
     ↓
Event received
     ↓
Correct host identity
     ↓
Correct timestamp
     ↓
Correct fields
     ↓
Searchable
```

---

# 13. Step 8 — Generate Windows Known Event

Đây là một phương pháp rất an toàn vì chỉ ghi một Application event có identifier riêng.

Mở elevated Command Prompt trên **lab Windows Server**:

```cmd
eventcreate /T INFORMATION /ID 100 /L APPLICATION /SO DAY10_LAB /D "event_id=D10-WIN-001 known_event=true"
```

Ý nghĩa:

```text
/T INFORMATION
→ event type

/ID 100
→ chosen lab Event ID

/L APPLICATION
→ Application channel

/SO DAY10_LAB
→ Source identifier

/D
→ event message
```

Expected result thường cho biết event đã được tạo thành công.

---

# 14. Step 9 — Verify Windows Event tại Source

Read-only PowerShell:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Application'
    StartTime=(Get-Date).AddMinutes(-10)
} |
Where-Object {$_.Message -like '*D10-WIN-001*'} |
Select-Object TimeCreated, Id, ProviderName, MachineName, Message
```

Expected:

```text
TimeCreated
Id
ProviderName
MachineName
Message containing D10-WIN-001
```

Evidence:

```text
Source record exists = YES
Source timestamp = ...
Hostname = ...
Event ID = ...
Known marker = D10-WIN-001
```

---

# 15. Step 10 — Tìm cùng Event trong EventLog Analyzer

Trong EventLog Analyzer, query theo:

```text
Device:
WIN-D10-01

Time:
source timestamp ± 5 minutes

Message:
D10-WIN-001
```

Sau đó xác nhận:

```text
Hostname matches
Event ID matches
Message matches
Timestamp logically matches
Source type matches
```

Đây mới chứng minh:

```text
Source → Collection → Parser → Index/Search
```

đã hoạt động cho event đó.

---

# 16. Step 11 — Tính Ingestion Delay

Giả sử:

```text
Source event time:
10:15:00.000

First searchable observation:
10:15:17.000
```

Observed ingestion delay:

```text
17 seconds
```

Ghi:

```text
D10-WIN-001
SourceTime = 10:15:00
ObservedSearchableTime = 10:15:17
ObservedDelay = 17 s
```

Không gọi đây là exact internal processing time nếu bạn chỉ đo bằng quan sát manual.

Hãy gọi:

```text
observed ingestion/search availability delay
```

---

# PART D — RHEL KNOWN-EVENT VALIDATION

# 17. Step 12 — Generate RHEL Known Event

Trên authorized RHEL lab:

```bash
logger -p user.notice -t DAY10_LAB \
"event_id=D10-RHEL-001 known_event=true"
```

`logger` gửi một controlled message vào logging subsystem.

Giải thích:

```text
-p user.notice
→ facility.priority

-t DAY10_LAB
→ tag

event_id=D10-RHEL-001
→ unique correlation marker
```

Đây là configuration-safe hơn nhiều so với cố ý phá một production service.

---

# 18. Step 13 — Verify RHEL Event tại Source

```bash
journalctl -t DAY10_LAB \
  --since "-10 minutes" \
  -o short-iso-precise
```

Bạn phải thấy marker:

```text
D10-RHEL-001
```

Có thể export structured representation:

```bash
journalctl -t DAY10_LAB \
  --since "-10 minutes" \
  -o export \
  > D10-EV-002_RHEL_DAY10_LAB.export
```

Readable form:

```bash
journalctl -t DAY10_LAB \
  --since "-10 minutes" \
  -o short-iso-precise \
  > D10-EV-002_RHEL_DAY10_LAB.txt
```

Không thay đổi source log.

---

# 19. Step 14 — Verify RHEL trong EventLog Analyzer

Query:

```text
Source = RHEL-D10-01
Message contains D10-RHEL-001
Time = timestamp ± 5 minutes
```

Kiểm:

```text
source hostname
event message
timestamp
severity/priority if parsed
tag/application field
```

Nếu raw log thấy nhưng field `tag` không đúng:

```text
collection = likely working
parsing/normalization = investigate
```

Đừng gọi đó là collection failure.

---

# PART E — KNOWN-EVENT RECONCILIATION

# 20. Step 15 — Tạo nhiều known events để kiểm count

Một event chứng minh pipeline có thể hoạt động.

Nhiều events mới cho phép kiểm tra completeness sơ bộ.

Ví dụ RHEL:

```bash
for i in 001 002 003 004 005
do
  logger -p user.notice -t DAY10_LAB \
  "event_id=D10-RHEL-$i batch=B01"
done
```

Verify source:

```bash
journalctl -t DAY10_LAB \
  --since "-10 minutes" \
  --grep 'batch=B01' \
  --no-pager
```

Count:

```bash
journalctl -t DAY10_LAB \
  --since "-10 minutes" \
  --grep 'batch=B01' \
  --no-pager |
wc -l
```

Expected:

```text
5
```

---

# 21. Step 16 — Reconcile với Centralized Analyzer

Trong EventLog Analyzer:

```text
Source = RHEL-D10-01
Message contains batch=B01
```

Nếu result:

```text
5
```

thì:

```text
Source events = 5
Central searchable events = 5
Difference = 0
```

Nếu:

```text
Source = 5
Analyzer = 4
```

thì **lab chưa pass**.

Phải investigate missing event.

---

# 22. Reconciliation Worksheet

| Test                  | Source Count | Analyzer Count | Export Count | Result |
| --------------------- | -----------: | -------------: | -----------: | ------ |
| Windows known event   |            1 |              1 |            1 | PASS   |
| RHEL batch B01        |            5 |              5 |            5 | PASS   |
| FortiGate denied flow |            3 |              3 |            3 | PASS   |
| Other                 |          ... |            ... |          ... | ...    |

Acceptance:

```text
SourceCount = AnalyzerCount = ExportCount
```

hoặc:

```text
Difference documented and explained
```

Không được:

```text
Difference unexplained
→ PASS
```

Đây là một acceptance requirement rõ ràng của syllabus. Infrastructure_Operations_14_Da…

---

# PART F — WINDOWS SECURITY EVENT INVESTIGATION

# 23. Search Requirement 1 — Failed Logons

Syllabus bắt buộc search failed logons.

Có hai phương án.

### Phương án ưu tiên

Instructor cung cấp fixture/pre-generated failed logon events.

Đây là lựa chọn tốt nhất nếu lab account policy có lockout risk.

### Phương án disposable lab account

Chỉ dùng nếu được phép.

Tạo account theo lab runbook, không sử dụng user thật.

Sau đó thực hiện một intentional failed authentication với số lần rất nhỏ và thấp hơn account lockout threshold.

Không brute force.

Không loop passwords.

---

# 24. Windows Event 4625

Một common Windows security event cho failed logon là:

```text
Event ID 4625
An account failed to log on
```

Nhưng việc event có xuất hiện còn phụ thuộc audit configuration và loại authentication.

Query read-only:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4625
    StartTime=(Get-Date).AddMinutes(-30)
} |
Select-Object TimeCreated, Id, MachineName, Message
```

Nếu không có result, chưa được kết luận:

```text
No failed logon occurred.
```

Phải check:

```text
Was test generated?
Was audit enabled?
Was correct log queried?
Was correct host queried?
```

---

# 25. Search failed logons trong Analyzer

Bắt đầu với:

```text
Time range
+ Windows source
+ failed-logon event/category
```

Sau đó refine:

```text
Username
Source host/IP
Target host
Failure reason
```

Câu hỏi phải trả lời:

```text
Một account bị fail nhiều lần?
Một source thử nhiều accounts?
Nhiều sources thử một account?
Có subsequent successful login không?
Account có privileged không?
```

Không chỉ:

```text
There are 20 failed logons.
```

---

# PART G — PRIVILEGE CHANGE INVESTIGATION

# 26. Search Requirement 2 — Privilege Changes

Khuyến nghị sử dụng pre-seeded fixture.

Một example event khi member được thêm vào security-enabled local group có thể là:

```text
4732
```

nhưng exact Event ID phụ thuộc loại group/domain context và audit configuration.

Không nên memorize `4732 = mọi privilege change`.

---

# 27. Optional controlled privilege-change test

**Chỉ lab Windows VM được phép.**

Tạo disposable local user với password nhập interactively:

```cmd
net user D10_PrivTest * /add
```

Sau đó nếu explicit authorization cho phép:

```cmd
net localgroup Administrators D10_PrivTest /add
```

Ngay lập tức thu evidence.

Sau đó rollback:

```cmd
net localgroup Administrators D10_PrivTest /delete
net user D10_PrivTest /delete
```

Risk:

```text
Trong thời gian ngắn account có local administrative privilege.
```

Vì vậy đây không phải default exercise nếu fixture có sẵn.

---

# 28. Verification

Read-only:

```cmd
net localgroup Administrators
```

Audit query:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    StartTime=(Get-Date).AddMinutes(-30)
} |
Where-Object {
    $_.Id -in 4732,4733
} |
Select-Object TimeCreated, Id, MachineName, Message
```

Sau rollback, verify account không còn.

---

# PART H — SERVICE FAILURE INVESTIGATION

# 29. Search Requirement 3 — Service Failures

Không dùng một critical production service để tạo lỗi.

Recommended lab:

```text
Prebuilt service:
D10LabSvc
```

Instructor deliberately configures hoặc cung cấp event fixture để service fail.

Workflow:

```text
Confirm service healthy/baseline
       ↓
Trigger prepared failure
       ↓
Observe source event
       ↓
Search EventLog Analyzer
       ↓
Correlate preceding events
       ↓
Repair using lab runbook
       ↓
Verify service recovered
```

---

# 30. Windows read-only service checks

```powershell
Get-Service -Name D10LabSvc
```

Event query có thể bắt đầu rộng:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='System'
    StartTime=(Get-Date).AddMinutes(-30)
} |
Where-Object {
    $_.ProviderName -eq 'Service Control Manager'
} |
Select-Object TimeCreated, Id, LevelDisplayName, Message
```

Không assume một Event ID duy nhất cho mọi failure.

Service Control Manager có nhiều Event IDs tùy condition.

---

# 31. Root-cause reasoning

Bạn cần tìm:

```text
Failure event
        ↑
preceding events
        ↓
subsequent events
```

Ví dụ:

```text
10:14:45 configuration activity
10:15:03 service startup attempted
10:15:04 service failure
10:15:06 monitoring alert
```

Nhưng vẫn chưa được tự động kết luận configuration activity là nguyên nhân nếu chưa có supporting evidence.

---

# PART I — ADMINISTRATIVE ACTIVITY

# 32. Search Requirement 4 — Administrative Activity

Mục tiêu là trả lời:

```text
Who?
Did what?
On which asset?
When?
Was it expected?
Was it approved?
```

Các category có thể gồm:

```text
admin login
group membership change
service control
configuration changes
security policy change
```

Output không phải chỉ là log dump.

Bạn cần một mini timeline:

| Time  | Source  | Identity  | Activity       | Result  | Ticket  |
| ----- | ------- | --------- | -------------- | ------- | ------- |
| 10:15 | WIN-D10 | D10_Admin | service action | success | LAB-... |
| 10:17 | WIN-D10 | D10_Admin | group change   | success | LAB-... |

---

# PART J — CISCO LOG-SOURCE VALIDATION

# 33. Không cần thay đổi Switch để chứng minh Day 10

Cisco trong Day 10 chủ yếu là centralized log source.

Sử dụng read-only diagnosis từ Day 8:

```text
show clock
show logging
```

và platform-specific NTP/syslog inspection từ approved IOS XE/NX-OS runbook.

Do IOS XE và NX-OS khác nhau, không nên dùng một syntax giả định cho tất cả versions.

---

# 34. Cisco evidence cần xác minh

```text
Device identity
Platform/release
Device timestamp
Syslog destination
Recent local syslog
Same event centrally searchable
```

Có thể dùng event có sẵn như:

```text
authorized login
logout
lab interface event
approved configuration fixture
```

Không shutdown interface chỉ để tạo log nếu topology đang phục vụ lab khác.

---

# PART K — FIREWALL ANALYZER ONBOARDING

# 35. Các phương pháp lab an toàn

Documentation hiện tại của Firewall Analyzer cho phép thiết bị gửi syslog, import log file và sử dụng prebundled/simulated firewall logs. Vì vậy nếu không có authorized FortiGate thật, **simulation/import là lựa chọn hợp lệ cho practice**, nhưng phải đánh dấu status là simulation chứ không claim actual FortiGate hands-on. [ManageEngine](https://www.manageengine.com/products/firewall/help/discovery-settings-add-device.html?utm_source=chatgpt.com)

Một lab hoàn chỉnh nên ưu tiên:

```text
Actual authorized FortiGate
        ↓
Existing/pre-approved log forwarding
        ↓
Firewall Analyzer
```

Không thay syslog configuration production chỉ vì lab.

---

# 36. Firewall Analyzer Listener

Current ManageEngine documentation cho Firewall Analyzer mô tả default UDP syslog listener port `1514` trong flow forward logs. Tuy nhiên đây **không phải giá trị được phép assume** cho môi trường của bạn: actual port/protocol phải lấy từ deployed configuration/runbook. [ManageEngine](https://www.manageengine.com/products/firewall/help/add-device-forward-syslog.html?utm_source=chatgpt.com)

Ghi evidence:

```text
Configured listener:
<actual value>

Transport:
<actual value>

Source:
FG-D10-01
```

---

# PART L — GENERATE CONTROLLED FIREWALL EVENTS

# 37. Allowed-flow Test

Giả sử lab cho phép:

```text
Client:
10.10.10.25

Server:
10.20.20.10

Allowed:
TCP/443
```

Linux:

```bash
curl -v --connect-timeout 5 https://10.20.20.10/
```

Hoặc Windows:

```powershell
Test-NetConnection 10.20.20.10 -Port 443
```

Ghi:

```text
Test ID = FW-ALLOW-001
Start timestamp
Source IP
Destination IP
Destination port
Expected = allowed
```

---

# 38. Denied-flow Test

Phải dùng **pre-approved blocked port/path**.

Ví dụ sandbox policy đã xác nhận rằng TCP/8081 bị deny:

```bash
curl -v --connect-timeout 5 http://10.20.20.10:8081/
```

Hoặc:

```powershell
Test-NetConnection 10.20.20.10 -Port 8081
```

Expected:

```text
connection unsuccessful
```

Nhưng client failure alone chưa chứng minh firewall deny.

Bạn phải tìm firewall record.

---

# 39. Firewall evidence

Trong Firewall Analyzer tìm:

```text
source = 10.10.10.25
destination = 10.20.20.10
port = 8081
time = test window
```

Verify:

```text
Action = deny
Policy/rule = expected policy
Protocol = TCP
Source/destination correct
Timestamp correct
```

Bây giờ mới có evidence rằng firewall xử lý traffic theo deny.

---

# PART M — TRAFFIC ANALYSIS

# 40. Điều tra một Flow

Với mỗi firewall flow cần trả lời:

```text
Did packet/connection reach firewall?

Which source?

Which destination?

Which service?

Which action?

Which rule?

At what time?
```

Nếu application fail nhưng:

```text
Firewall action = allow
```

không được nói:

```text
Firewall is fine, case closed.
```

Bạn mới chỉ chứng minh:

```text
Firewall observed and allowed this traffic.
```

Backend/TLS/application vẫn có thể fail.

---

# 41. Denied traffic investigation

Ví dụ analyzer cho:

```text
src=10.10.10.25
dst=10.20.20.10
dport=443
action=deny
policy=100
```

Hãy kiểm:

```text
Is this expected path?
Is source correct?
Is destination correct?
Is port correct?
Was rule 100 expected?
Was there a recent config change?
```

Classification có thể khác nhau:

```text
Expected deny
→ normal control

Unexpected deny
→ operational incident/config weakness

Unauthorized source probing blocked service
→ possible security event
```

---

# PART N — FIREWALL POLICY UTILIZATION

# 42. Used Rule vs Unused Rule

Firewall Analyzer documentation giải thích rằng used-rule data có thể suy ra từ traffic logs, còn việc xác định unused rules cần biết **toàn bộ rule set** và correlate nó với rules được trigger trong syslogs. Vì vậy “không thấy rule trong traffic logs” không tự động chứng minh rule không tồn tại hoặc không bao giờ cần dùng. [ManageEngine](https://www.manageengine.com/products/firewall/rule-cleanup-faq.html?utm_source=chatgpt.com)

Đây là một practical lesson cực kỳ quan trọng.

---

# 43. Rule utilization worksheet

| Rule      | Observed hits | Observation window | Interpretation                   |
| --------- | ------------: | ------------------ | -------------------------------- |
| Policy 10 |        12,440 | 7 days             | actively used                    |
| Policy 20 |             3 | 7 days             | low use                          |
| Policy 30 |             0 | 7 days             | no observed use; review required |

Không viết:

```text
Policy 30 = unnecessary
```

Chỉ viết:

```text
No observed hits in selected seven-day data window.
Requires owner/business review before change.
```

---

# PART O — CONFIGURATION CHANGE TRACKING

# 44. Review — không tự tạo production change

Trong Firewall Analyzer, dùng existing/simulated configuration-change record.

Cần xác định:

```text
Who changed it?
When?
Which firewall?
What changed?
Is there a ticket?
Was it inside change window?
```

Firewall Analyzer documentation có capability hiển thị configuration changes khi device/rule retrieval được cấu hình phù hợp, nhưng exact availability tùy device/mode/version. [ManageEngine](https://www.manageengine.com/products/firewall/help/firewall-settings/config-view-rules.html?utm_source=chatgpt.com)

---

# 45. Change correlation example

```text
13:55 Approved change begins
14:01 Firewall policy modified
14:02 Configuration record generated
14:05 Denied traffic begins
14:06 Application alert
```

Hypothesis:

```text
Firewall change may have contributed.
```

Không ngay lập tức viết:

```text
Firewall change caused outage.
```

Cần thêm evidence.

---

# PART P — POLICY ANOMALY REVIEW

# 46. Practical review

Nếu licensed/supported feature có sẵn, inspect các report như:

```text
unused rules
overlapping rules
redundant rules
overly permissive policies
```

Current ManageEngine Firewall Analyzer material mô tả anomaly analysis gồm redundant/overlapping rule logic, unused rules/objects và overly permissive structures. [ManageEngine](https://www.manageengine.com/products/firewall/firewall-analyzer.html?utm_source=chatgpt.com)

Không sửa rule.

Output chỉ là:

```text
Finding
Evidence
Risk hypothesis
Owner
Recommended review
```

---

# PART Q — COMPLIANCE REPORT

# 47. Generate/inspect Compliance Report

Firewall Analyzer hiện hỗ trợ nhiều firewall compliance reports, nhưng việc report có ý nghĩa phụ thuộc supported device và correct environmental settings. Current documentation cũng lưu ý phải cấu hình network/interface context phù hợp cho report. [ManageEngine](https://www.manageengine.com/products/firewall/help/firewall-compliance-reports.html?utm_source=chatgpt.com)

Lab objective không phải:

```text
Get 100% compliance.
```

Mà là hiểu:

```text
How was report generated?
Which device?
Which rules/configuration?
Which framework?
Which report time?
Which limitations?
```

---

# PART R — SILENT SOURCE LAB

Đây là một bài bắt buộc rất quan trọng.

# 48. Scenario

Analyzer hiển thị:

```text
RHEL-D10-01
Last event: 10:12

Current time: 11:05
```

Bình thường server gửi nhiều logs mỗi phút.

Không được kết luận ngay:

```text
Server is down.
```

---

# 49. Troubleshooting checkpoint 1 — Is source alive?

Read-only:

```bash
hostnamectl
uptime
date --iso-8601=seconds
```

Nếu host sống:

```text
Source server down
```

đã được loại bỏ.

---

# 50. Checkpoint 2 — Is source producing logs?

Generate:

```bash
logger -t DAY10_LAB \
"event_id=D10-SILENT-TEST-001"
```

Verify locally:

```bash
journalctl -t DAY10_LAB \
  --since "-5 minutes"
```

Nếu local event có:

```text
Source logging works.
```

---

# 51. Checkpoint 3 — Is forwarding/agent healthy?

Tùy actual collection method, inspect:

```text
forwarder/agent configuration
service state
queue/buffer
destination
```

Không restart trước khi thu evidence.

Thứ tự:

```text
Inspect
→ capture
→ determine failure
→ approved correction
→ verify
```

---

# 52. Checkpoint 4 — Network path

Kiểm:

```text
source → collector connectivity
```

nhưng chỉ dùng approved diagnostics.

Nếu syslog UDP, chỉ `ping` không chứng minh syslog path hoạt động.

---

# 53. Checkpoint 5 — Collector

Trong analyzer:

```text
collector/service status
last message time
other sources working?
```

Nếu mọi source im lặng cùng lúc:

```text
central collector issue
```

có probability cao hơn một independent endpoint issue.

---

# 54. Checkpoint 6 — Parser

Nếu raw message đến nhưng query field không populated:

```text
collector working
parser/normalization suspect
```

Đây chính là distinction syllabus muốn bạn biết.

---

# 55. Checkpoint 7 — Search

Verify:

```text
correct source
correct time range
correct timezone
correct fields
no overly restrictive filter
```

---

# 56. Silent-source final classification

Phải phân loại thành một trong các dạng:

```text
No matching event

Disconnected source

Collection/forwarding failure

Parser failure

Index/search failure

Retention issue

Known collection gap
```

Và evidence phải nói rõ vì sao.

Official EventLog Analyzer troubleshooting material hiện cũng phân biệt các trạng thái như log forwarding chưa được cấu hình, không có logs gần đây và collector/agent problems. [ManageEngine | Community and Support](https://pitstop.manageengine.com/portal/en/kb/articles/no-data-logs-collected-from-syslog-device?utm_source=chatgpt.com)

---

# PART S — INGESTION HEALTH

# 57. Measure nhiều test events

Ví dụ:

| Event    | Source Time | Searchable Time | Observed Delay |
| -------- | ----------- | --------------- | -------------: |
| WIN-001  | 10:00:00    | 10:00:11        |           11 s |
| WIN-002  | 10:05:00    | 10:05:09        |            9 s |
| RHEL-001 | 10:10:00    | 10:10:15        |           15 s |

Không cần performance benchmark chuyên sâu.

Bạn đang tìm anomaly:

```text
Normally 10–20 seconds
Suddenly 18 minutes
```

---

# 58. Capacity awareness

Kiểm tra analyzer:

```text
disk/storage state
archive state
retention
collector status
processing backlog if exposed
```

Không thay retention settings nếu không được phép.

Một analyzer disk gần full có thể tạo:

```text
ingestion lag
indexing problems
archive failures
```

---

# PART T — ALERT DELIVERY TEST

# 59. Alert verification end-to-end

Nếu lab đã có alert rule cho test event:

```text
Known event
     ↓
Rule match
     ↓
Alert generated
     ↓
Notification sent
     ↓
Notification received
```

Bạn phải ghi cả bốn stage.

Không chỉ screenshot:

```text
Alert Enabled = True
```

---

# 60. Negative alert test

Nếu test event **không** match rule:

```text
No alert should occur.
```

Đây cũng là validation.

Alert system phải phân biệt:

```text
positive case
negative case
```

Nếu mọi event đều alert thì rule quá broad.

---

# PART U — REPORT SCHEDULING VALIDATION

# 61. Scheduled report

Nếu lab cho phép schedule report, verify:

```text
Schedule exists
       ↓
Execution happens
       ↓
Report generated
       ↓
Correct data/time range
       ↓
Correct recipient/storage
```

Nếu email delivery unavailable, record:

```text
Report generation verified.
Email delivery not assessed due to lab limitation.
```

Không fabricate evidence.

---

# PART V — CROSS-SYSTEM INCIDENT CORRELATION

# 62. Master Incident Scenario

Giả sử:

> User báo ứng dụng `APP01` không truy cập được từ 14:05.

Affected path:

```text
Client
   ↓
Cisco
   ↓
FortiGate
   ↓
BIG-IP
   ↓
Windows/RHEL APP01
   ↓
AWS dependency
```

Mục tiêu là tạo một **joined timeline**.

---

# 63. Stage 1 — Establish scope

Ghi:

```text
Incident:
APP01 inaccessible

First observed:
14:05 UTC+07

Affected users:
Subnet 10.10.10.0/24

Known unaffected:
Internal admin subnet
```

Scope trước root cause.

---

# 64. Stage 2 — Windows/RHEL search

Search khoảng:

```text
13:55–14:15
```

Không chỉ 14:05 đúng một phút.

Tìm:

```text
service failures
authentication
administrative changes
application events
```

---

# 65. Stage 3 — Firewall search

Search cùng normalized time window:

```text
source subnet
destination APP01
service port
action
policy
```

---

# 66. Stage 4 — Cisco

Xem:

```text
link/interface events
routing adjacency
system warnings
```

Nếu no network event, đó chỉ là supporting negative evidence nếu source coverage đã được xác minh.

---

# 67. Stage 5 — CloudTrail

Nếu scenario có AWS activity và bạn có approved read-only AWS CLI profile:

Trước hết verify identity:

```bash
aws sts get-caller-identity
```

Sau đó có thể query recent management events:

```bash
aws cloudtrail lookup-events \
  --start-time 2026-10-21T06:55:00Z \
  --end-time 2026-10-21T07:15:00Z \
  --region <LAB_REGION>
```

AWS documentation xác nhận `lookup-events` dùng để tra management/Insights events gần đây và operation này có pagination; Event History management-event lookup có giới hạn 90 ngày theo Region. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/cloudtrail/lookup-events.html?utm_source=chatgpt.com)

Không dùng:

```text
--no-verify-ssl
```

để làm cho request “pass”.

---

# 68. CloudWatch Logs

Nếu application logs ở CloudWatch Logs:

```bash
aws logs filter-log-events \
  --log-group-name '<LAB_LOG_GROUP>' \
  --start-time <START_EPOCH_MS> \
  --end-time <END_EPOCH_MS> \
  --filter-pattern '<LAB_PATTERN>' \
  --region <LAB_REGION>
```

AWS lưu ý `filter-log-events` là paginated; một page có thể partial hoặc thậm chí empty dù vẫn còn `nextToken`. Điều này là ví dụ rất tốt cho lesson Day 10 rằng **“một page ít result” không đồng nghĩa query đã hết data**. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/logs/filter-log-events.html?utm_source=chatgpt.com)

---

# 69. Normalize Timeline

Ví dụ collected evidence:

```text
13:58:41 +07  Administrative login
14:01:12 +07  Firewall config activity
14:02:03 +07  Denied traffic starts
14:03:48 +07  Backend health warning
14:04:57 +07  Application error
14:05:10 +07  Monitoring alert
14:05:30 +07  First user report
```

Đừng sort timestamp strings nếu timezone khác nhau.

Normalize trước.

---

# 70. Correlation Table

| Normalized Time | Source     | Event             | Evidence ID | Interpretation      |
| --------------- | ---------- | ----------------- | ----------- | ------------------- |
| 14:01:12        | Firewall   | config activity   | D10-EV-20   | candidate precursor |
| 14:02:03        | Firewall   | denies begin      | D10-EV-21   | traffic impact      |
| 14:03:48        | BIG-IP     | backend unhealthy | D10-EV-22   | downstream symptom  |
| 14:04:57        | Server     | app timeout       | D10-EV-23   | service impact      |
| 14:05:10        | Monitoring | alert             | D10-EV-24   | detection           |

---

# 71. Không nhảy từ timeline sang root cause

Hypothesis:

```text
Firewall change caused outage.
```

Evidence cần thêm:

```text
Changed policy matches affected flow

Deny begins after change

Other dependencies healthy

Corrective rollback/remediation restores path
```

Nếu chưa có, report:

```text
Leading hypothesis:
Firewall policy change temporally correlates with denied application
traffic. Causation not yet fully confirmed.
```

Đây là cách viết defensible.

---

# PART W — SECURITY FINDING CLASSIFICATION

# 72. Classification trước Severity

Mỗi finding trước tiên phải vào một category:

```text
Vulnerability

Configuration weakness

IAM exception

Operational incident
```

Đây là yêu cầu trực tiếp của additive Day 10. Infrastructure_Operations_14_Da…

---

# 73. Examples

```text
Known exploitable unpatched service
→ Vulnerability
```

```text
Firewall policy unnecessarily permits broad source range
→ Configuration weakness
```

```text
Privileged account active without current approval
→ IAM exception
```

```text
Log collector stopped due to storage exhaustion
→ Operational incident
```

---

# 74. Exposure assessment

Sau classification, hỏi:

```text
Internet-facing?
Internal?
Restricted subnet?
Authentication required?
Privileged access required?
Affected service enabled?
Compensating controls?
```

Không assign business risk chỉ từ technical severity.

---

# 75. Business impact

Xác định:

```text
Production vs lab
Critical service?
Sensitive data?
Number of users?
Availability impact?
Recovery dependency?
```

---

# 76. Finding Register

| Finding ID | Type                 | Asset  | Evidence | Technical Severity | Exposure | Business Risk | Owner            | Action            |
| ---------- | -------------------- | ------ | -------- | ------------------ | -------- | ------------- | ---------------- | ----------------- |
| F-001      | Config weakness      | FG-01  | EV-21    | High               | Internal | Medium        | Network Security | Review            |
| F-002      | Operational incident | ELA-01 | EV-30    | High               | N/A      | High          | Monitoring Team  | Restore ingestion |

Syllabus yêu cầu Day 10 cuối cùng có **risk/owner-based finding register**. Infrastructure_Operations_14_Da…

---

# PART X — FALSE-POSITIVE VALIDATION

# 77. Scenario

Analyzer report nói:

```text
Policy is unused.
```

Không lập tức classification:

```text
Unused rule = vulnerability
```

Hỏi:

```text
Observation window?
Rule set complete?
Syslogs complete?
Source had a collection gap?
Policy used only during DR?
```

Nếu collection gap tồn tại:

```text
unused conclusion is unsupported
```

---

# 78. Evidence-based conclusion

Bad:

> Rule 150 is unused.

Better:

> Rule 150 recorded zero observed hits in the analyzed 7-day dataset. The conclusion is limited by a 43-minute firewall log collection gap on October 20 and should not be used alone as authorization for rule removal.

Đó là audit-quality reasoning.

---

# PART Y — EXPORT EVIDENCE

# 79. Hai loại export cần có

Syllabus yêu cầu:

```text
Native/original evidence where available
```

và:

```text
Filtered report
```

Không chỉ một screenshot.

---

# 80. Raw/native artifact

Ví dụ:

```text
Windows EVTX
journal export
raw syslog
original analyzer export
firewall raw log export
CloudTrail JSON
```

Tùy source/product.

---

# 81. Filtered artifact

Ví dụ:

```text
failed_logons_1400-1500.csv

firewall_denies_APP01.csv

service_failure_timeline.csv
```

Derived output phải có parent evidence ID.

Ví dụ:

```text
Parent:
D10-EV-RAW-021

Derived:
D10-EV-FLT-005
```

---

# 82. Windows native export example

Nếu cần giữ Application evidence:

```powershell
wevtutil epl Application "D:\Day10\01_Raw\Windows\Application.evtx"
```

Đây export toàn channel.

Nếu scope cần bounded event set, có thể giữ EVTX nguyên channel làm raw và tạo filtered CSV riêng.

Không clear channel sau export.

---

# 83. Windows filtered CSV

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Application'
    StartTime=(Get-Date).AddHours(-1)
} |
Where-Object {$_.Message -like '*D10-WIN-*'} |
Select-Object TimeCreated, Id, ProviderName, MachineName, Message |
Export-Csv `
  "D:\Day10\03_Filtered\Windows_known_events.csv" `
  -NoTypeInformation `
  -Encoding UTF8
```

Ghi rõ:

```text
Source:
Application event log

Filter:
Message contains D10-WIN-

Window:
...

Encoding:
UTF-8
```

---

# 84. Linux raw/readable export

```bash
journalctl \
  --since "2026-10-21 10:00:00" \
  --until "2026-10-21 11:00:00" \
  -o export \
  > D10_RHEL_1000-1100.export
```

Readable:

```bash
journalctl \
  --since "2026-10-21 10:00:00" \
  --until "2026-10-21 11:00:00" \
  -o short-iso-precise \
  > D10_RHEL_1000-1100.txt
```

Timezone phải được ghi ngoài file nếu format không đủ context.

---

# PART Z — HASHING

# 85. Windows SHA-256

```powershell
Get-FileHash `
  "D:\Day10\01_Raw\Windows\Application.evtx" `
  -Algorithm SHA256
```

Output:

```text
Algorithm Hash Path
--------- ---- ----
SHA256    ...  Application.evtx
```

---

# 86. Linux SHA-256

```bash
sha256sum D10_RHEL_1000-1100.export
```

Ghi vào:

```text
Hash manifest
```

---

# 87. Điều hash chứng minh

Nếu hash sau transfer vẫn giống:

```text
artifact bytes unchanged since hash baseline
```

Hash **không chứng minh**:

```text
query was correct

source was truthful

collection was complete

export wasn't truncated before hashing
```

Điều này nằm trong evidence contract của syllabus. Infrastructure_Operations_14_Da…

---

# PART AA — EXPORT COUNT & PAGINATION

# 88. Mandatory count reconciliation

Giả sử UI:

```text
Search result:
12,531 events
```

Export:

```text
10,000 records
```

Không được tiếp tục như không có gì xảy ra.

Phải xác định:

```text
Was export capped?
Multiple pages?
Limit?
Wrong filter?
```

---

# 89. Pagination worksheet

```text
Search count:
12,531

Export page 1:
5,000

Export page 2:
5,000

Export page 3:
2,531

Combined:
12,531
```

Acceptance:

```text
SearchTotal = ExportTotal
```

hoặc limitation documented.

---

# 90. Cloud examples cho pagination awareness

CloudTrail `lookup-events` và CloudWatch Logs `filter-log-events` đều có pagination behavior; AWS documentation đặc biệt lưu ý CloudWatch page có thể partial/empty trong khi vẫn có `nextToken`. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/cloudtrail/lookup-events.html?utm_source=chatgpt.com)

Đây là lý do:

```text
First page = total dataset
```

là một assumption nguy hiểm.

---

# PART AB — EVIDENCE MANIFEST

# 91. Manifest schema

Một Day 10 evidence manifest nên có:

| Field           | Example                 |
| --------------- | ----------------------- |
| Evidence ID     | D10-EV-021              |
| Source System   | Firewall Analyzer       |
| Product/Version | actual lab value        |
| Source Device   | FG-D10-01               |
| Environment     | LAB                     |
| Ticket          | LAB-INC-D10-001         |
| Collector       | learner                 |
| Collection Time | ISO 8601                |
| Source Window   | 14:00–14:15             |
| Source Timezone | UTC+07                  |
| Query/Filter    | src=X AND dst=Y         |
| Format          | CSV                     |
| Encoding        | UTF-8                   |
| Record Count    | 42                      |
| File Size       | ...                     |
| SHA-256         | ...                     |
| Parent Evidence | RAW-...                 |
| Limitations     | None / pagination / gap |
| Classification  | Internal                |

Đây là implementation trực tiếp của evidence contract toàn syllabus. Infrastructure_Operations_14_Da…

---

# PART AC — INCIDENT INVESTIGATION REPORT

# 92. Final report structure

Một reviewer-ready report nên có:

```text
Incident ID

Summary

Affected assets/services

Business impact

Investigation window

Reference timezone

Source inventory

Collection health

Timeline

Queries performed

Evidence IDs

Findings

Root cause / leading hypothesis

Corrective action

Verification

Unresolved gaps

Escalation

Evidence manifest
```

---

# 93. Root Cause confidence

Nên phân biệt:

```text
Confirmed

Highly likely

Possible

Unconfirmed
```

Ví dụ:

```text
Confirmed:
Firewall policy blocked exact affected flow,
and authorized rollback restored connectivity.

Unconfirmed:
Policy change happened near outage,
but affected flow evidence is missing.
```

---

# PART AD — FAILURE INJECTION LABS

Dưới đây là 5 failure injections an toàn nên có trong Day 10.

---

# 94. Failure Injection 1 — Silent RHEL Source

## Fault

Instructor stops/redirects lab forwarding hoặc cung cấp fixture tương ứng.

Không phá local journald.

## Symptoms

```text
RHEL local events exist
EventLog Analyzer receives nothing
```

## Expected investigation

```text
Generate known event
→ verify local journal
→ verify forwarder
→ verify network path
→ verify collector
```

## Correct conclusion

Ví dụ:

```text
Source generated events successfully,
but forwarding path was unavailable.
```

Không:

```text
No logs occurred.
```

---

# 95. Failure Injection 2 — Parser Failure

## Fault

Provide raw firewall/syslog message with changed/unrecognized format.

## Symptoms

```text
Raw event received

Source IP field empty

Search by source IP finds nothing
```

## Expected diagnosis

```text
collection = good

parsing = failed/incomplete
```

## Evidence

```text
raw record
parsed field output
search result
parser/version context
```

Escalate parser/content issue if outside L1 scope.

---

# 96. Failure Injection 3 — Timezone Mismatch

## Fault

Fixture has:

```text
Server = UTC+07
Firewall = UTC
CloudTrail = UTC
```

Learner initially treats strings as same timezone.

## Symptom

Timeline appears impossible.

Example:

```text
Server service failed before firewall change
```

## Expected diagnosis

Normalize timestamps.

Correct sequence then becomes:

```text
Firewall change
→ deny
→ service/app symptom
```

---

# 97. Failure Injection 4 — “Unused Rule” False Conclusion

## Fault

Firewall Analyzer shows rule with zero hits, nhưng dataset có collection gap.

## Expected reasoning

```text
Zero observed hits
≠
Proven unused rule
```

Required conclusion:

```text
Insufficient evidence for removal recommendation.
```

---

# 98. Failure Injection 5 — Truncated Export

## Fault

Analyzer UI reports:

```text
15,000 results
```

Export contains:

```text
10,000 rows
```

## Expected investigation

Check:

```text
export limit
pagination
report scope
```

## Correct outcome

Retrieve remainder or document limitation.

Không gọi 10,000 records là complete evidence.

---

# PART AE — TROUBLESHOOTING MATRIX

# 99. Symptom-to-layer reasoning

| Symptom                              | First checks                    | Possible layer       |
| ------------------------------------ | ------------------------------- | -------------------- |
| No events centrally                  | local event, source health      | source               |
| Local event exists, central absent   | agent/forwarder/network         | collection           |
| Raw record present, field missing    | parser                          | parsing              |
| Parsed data exists but search misses | query/index/timezone            | search               |
| Events delayed                       | queue/network/collector/storage | ingestion            |
| One source silent                    | source/forwarding               | endpoint             |
| All sources silent                   | collector/platform              | centralized platform |
| Report count lower than search       | pagination/export limit         | reporting            |
| Firewall rule appears unused         | rule-set + syslog completeness  | analysis             |
| Timeline impossible                  | timezone/NTP                    | time normalization   |

Đây nên trở thành troubleshooting muscle memory.

---

# PART AF — CLEANUP / ROLLBACK

# 100. Cleanup principle

**Không cleanup bằng xóa logs.**

Sau lab:

```text
Windows generated events
→ retain normally

RHEL logger events
→ retain normally

Analyzer exports
→ retain as evidence
```

---

# 101. Disposable privileged account

Nếu đã dùng:

```cmd
net localgroup Administrators D10_PrivTest /delete
net user D10_PrivTest /delete
```

Verification:

```cmd
net localgroup Administrators
net user D10_PrivTest
```

Expected second command:

```text
The user name could not be found.
```

---

# 102. Firewall simulation

Nếu sử dụng Firewall Analyzer simulation, stop simulator sau khi collect evidence.

Current documentation có explicit Start/Stop simulation workflow dành cho simulated firewall logs. [ManageEngine](https://www.manageengine.com/products/firewall/help/discovery-settings-simulate-firewalls.html?utm_source=chatgpt.com)

Document:

```text
Simulation stopped
Timestamp
No production device affected
```

---

# 103. Syslog configuration changes

Nếu lab thực sự thay syslog destination:

```text
restore original config
```

chỉ theo approved runbook.

Phải có:

```text
before config
after config
rollback verification
```

Không để lab collector vô tình tiếp tục nhận production-like logs ngoài scope.

---

# PART AG — ESCALATION PACKAGE

# 104. Khi nào L1/L2 dừng?

Ví dụ:

```text
Collector process repeatedly crashes

Persistent parser defect

Unknown privileged activity

Suspected compromise

Critical audit data missing

Production firewall change required

Storage/retention change required

Log tampering suspected
```

thì không tiếp tục “thử sửa”.

---

# 105. Good escalation

Ví dụ:

```text
Issue:
RHEL-D10-01 events stopped appearing centrally at 10:12.

Source:
Local journal verified healthy.

Known test:
D10-SILENT-TEST-001 exists locally at 10:43:02.

Collection:
No matching event received centrally.

Other sources:
Windows and Cisco collection healthy.

Network:
Approved connectivity checks passed.

Current hypothesis:
RHEL-specific forwarding/collector association failure.

Evidence:
D10-EV-031 through D10-EV-036.

Impact:
Audit coverage gap from 10:12 onward.

Action requested:
Monitoring platform L2 review forwarding/collector association.
```

Đây là một escalation chất lượng.

---

# PART AH — FULL ACCEPTANCE CRITERIA

# 106. EventLog Analyzer

Bạn phải chứng minh:

```text
Windows source identified correctly

RHEL source identified correctly

Cisco/network source inspected

Known Windows event generated

Known RHEL event generated

Events verified locally

Same events found centrally

Timestamp/timezone validated

Parsed fields checked

Ingestion delay assessed

Failed logons investigated

Privilege activity investigated

Service failure investigated

Administrative activity investigated

Silent source diagnosed

Collector/service health checked

Report exported

Alert/report delivery checked where available
```

---

# 107. Firewall Analyzer

Bạn phải chứng minh:

```text
Firewall source identified

Log ingestion verified

Known allow/deny traffic analyzed

Source/destination flow analyzed

Policy/rule identified

Rule utilization reviewed

Configuration change record reviewed

Policy anomaly awareness demonstrated

Compliance/security report inspected

Export verified
```

---

# 108. Correlation

Bạn phải:

```text
Use at least server + firewall logs

Include cloud evidence if lab provides it

Normalize timezone

Build ordered timeline

Distinguish correlation from causation

State confidence and gaps
```

---

# 109. Security classification

Bạn phải classify:

```text
Vulnerability
Configuration weakness
IAM exception
Operational incident
```

sau đó thêm:

```text
severity
exposure
business risk
owner
action
escalation
```

---

# 110. Evidence

Bắt buộc:

```text
Native/raw evidence where available

Filtered report

Search criteria

Source IDs

Time window

Timezone

Extraction timestamp

Record counts

Pagination/export limits

Excluded intervals

SHA-256

Evidence manifest
```

Acceptance chính thức của syllabus là known events phải được accounting, count reconcile hoặc discrepancy được giải thích, cả hai analyzer report phải có, ingestion health phải được kiểm tra và finding register phải dựa trên risk/owner. Infrastructure_Operations_14_Da…

---

# PART AI — PRACTICAL ASSESSMENT RUBRIC

Bạn có thể tự đánh giá mỗi skill theo rubric của toàn chương trình:

```text
H = Hands-on actual product lab

S = Simulation / fixture

D = Demonstration only

T = Theory/reference only
```

Syllabus yêu cầu các mức này phải được ghi rõ; simulation không được tự biến thành claim actual-product hands-on. Infrastructure_Operations_14_Da…

Ví dụ:

| Skill                              | Mode | Status | Evidence   |
| ---------------------------------- | ---- | ------ | ---------- |
| Windows known-event ingestion      | H    | PASS   | D10-EV-001 |
| RHEL syslog validation             | H    | PASS   | D10-EV-002 |
| Cisco centralized logging          | S    | PASS   | D10-EV-010 |
| EventLog Analyzer search           | H    | PASS   | D10-EV-015 |
| Firewall Analyzer traffic analysis | H/S  | PASS   | D10-EV-020 |
| Silent-source troubleshooting      | S    | PASS   | D10-EV-030 |

---

# PART AJ — PRACTICAL TROUBLESHOOTING FLOW CẦN THUỘC

```text
                "I CAN'T FIND THE EVENT"
                          │
                          ▼
                Query/timezone correct?
                    │            │
                   NO           YES
                    │            │
               Fix query         ▼
                       Did event occur at source?
                              │        │
                             NO       YES
                              │        │
                       Source/audit    ▼
                                  Local record exists?
                                      │       │
                                     NO      YES
                                      │       │
                                  Source      ▼
                                            Collection/
                                            forwarding?
                                              │
                                  ┌───────────┴───────────┐
                                  │                       │
                                FAIL                     OK
                                  │                       │
                              Fix/escalate                ▼
                                              Collector received?
                                                    │
                                           ┌────────┴────────┐
                                           │                 │
                                          NO                YES
                                           │                 │
                                   Transport/collector       ▼
                                                    Parser correct?
                                                     │        │
                                                    NO       YES
                                                     │        │
                                                Parser issue  ▼
                                                        Indexed/searchable?
                                                            │
                                                    ┌───────┴───────┐
                                                    │               │
                                                   NO              YES
                                                    │               │
                                               Platform issue      ▼
                                                              No matching
                                                               event or
                                                              filter issue
```

Đây là flow quan trọng nhất của phần thực hành.

---

# PART AK — DAY 10 COMMAND CHEAT SHEET

## Windows time / identity

```powershell
hostname
Get-Date -Format o
Get-TimeZone
w32tm /query /status
```

## Windows known event

```cmd
eventcreate /T INFORMATION /ID 100 /L APPLICATION /SO DAY10_LAB /D "event_id=D10-WIN-001 known_event=true"
```

## Windows search

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Application'
    StartTime=(Get-Date).AddMinutes(-10)
}
```

## Failed logons

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='Security'
    Id=4625
    StartTime=(Get-Date).AddMinutes(-30)
}
```

## Service events

```powershell
Get-WinEvent -FilterHashtable @{
    LogName='System'
    StartTime=(Get-Date).AddMinutes(-30)
} |
Where-Object {$_.ProviderName -eq 'Service Control Manager'}
```

## Windows hash

```powershell
Get-FileHash <FILE> -Algorithm SHA256
```

## RHEL time

```bash
date --iso-8601=seconds
timedatectl
```

## RHEL known event

```bash
logger -p user.notice -t DAY10_LAB \
"event_id=D10-RHEL-001 known_event=true"
```

## RHEL search

```bash
journalctl -t DAY10_LAB \
  --since "-10 minutes" \
  -o short-iso-precise
```

## Linux hash

```bash
sha256sum <FILE>
```

## Connectivity test

```bash
curl -v --connect-timeout 5 \
https://<LAB_SERVER>/
```

```powershell
Test-NetConnection <LAB_SERVER> -Port 443
```

## AWS identity

```bash
aws sts get-caller-identity
```

## CloudTrail

```bash
aws cloudtrail lookup-events \
  --start-time <START> \
  --end-time <END> \
  --region <REGION>
```

## CloudWatch Logs

```bash
aws logs filter-log-events \
  --log-group-name '<LOG_GROUP>' \
  --start-time <EPOCH_MS> \
  --end-time <EPOCH_MS> \
  --region <REGION>
```

---

# Day 10 — Practical Completion Checklist

Khi kết thúc module Thực hành, bạn chỉ nên đánh dấu **Day 10 practical complete** khi toàn bộ chain sau đã được chứng minh:

```text
[ ] Tôi có source inventory.

[ ] Tôi biết collection mode của từng source.

[ ] Tôi đã ghi product/version/build nơi cần thiết.

[ ] Tôi đã kiểm timezone/NTP state.

[ ] Tôi tạo được known Windows event.

[ ] Tôi verify known Windows event tại source.

[ ] Tôi tìm được chính event đó trong EventLog Analyzer.

[ ] Tôi tạo được known RHEL event.

[ ] Tôi verify RHEL event tại source.

[ ] Tôi tìm được nó centrally.

[ ] Tôi reconcile source/analyzer/export counts.

[ ] Tôi biết đo observed ingestion delay.

[ ] Tôi search được failed-logon scenario.

[ ] Tôi investigate privilege-change scenario.

[ ] Tôi investigate service-failure scenario.

[ ] Tôi investigate administrative activity.

[ ] Tôi inspect Cisco/network logs.

[ ] Tôi verify firewall log ingestion.

[ ] Tôi generate/inspect controlled allow/deny traffic.

[ ] Tôi xác định source/destination/port/action/policy.

[ ] Tôi hiểu used/unused rule limitations.

[ ] Tôi inspect configuration-change records.

[ ] Tôi inspect policy anomaly/security report.

[ ] Tôi diagnose được silent source.

[ ] Tôi phân biệt collection failure và parsing failure.

[ ] Tôi nhận diện collection gap.

[ ] Tôi normalize cross-system timestamps.

[ ] Tôi tạo được incident timeline.

[ ] Tôi không nhầm correlation với causation.

[ ] Tôi phân loại finding đúng loại.

[ ] Tôi đánh giá exposure.

[ ] Tôi phân biệt severity và business risk.

[ ] Tôi xác định remediation owner.

[ ] Tôi biết escalation boundary.

[ ] Tôi export raw/native evidence.

[ ] Tôi export filtered evidence.

[ ] Tôi ghi exact filters/search criteria.

[ ] Tôi ghi timezone.

[ ] Tôi kiểm pagination/export limit.

[ ] Tôi ghi excluded intervals.

[ ] Tôi reconcile record counts.

[ ] Tôi hash raw artifacts.

[ ] Tôi giữ raw evidence untouched.

[ ] Tôi hoàn thành evidence manifest.

[ ] Tôi có EventLog Analyzer report.

[ ] Tôi có Firewall Analyzer report.

[ ] Tôi có finding register.

[ ] Tôi có incident/audit evidence package.

[ ] Tôi có cleanup/rollback record.

[ ] Tôi không clear logs hoặc xóa evidence để cleanup.
```

Nếu toàn bộ checklist này đạt, bạn không chỉ “biết dùng ManageEngine”. Bạn đã thực hiện đúng năng lực mà Day 10 hướng tới: **chứng minh log pipeline hoạt động, phát hiện khi pipeline không đáng tin, điều tra nhiều source, correlate có kiểm soát, phân loại finding đúng, bảo toàn audit evidence và escalate với đủ technical context**. Đây cũng chính là nền tảng để sang Day 11 làm ServiceNow/privileged-access evidence và sang Day 13 làm IAM/vulnerability reconciliation ở mức audit-defensible. Infrastructure_Operations_14_Da…
