Dựa đúng vào `Infrastructure_Operations_14_Day_Syllabus.md`, **Day 10** thuộc Module 6 — **Monitoring, ITSM, Asset Management & Privileged Access Operations**, nhưng riêng Day 10 tập trung vào **Infrastructure monitoring, centralized log management, ManageEngine EventLog Analyzer, ManageEngine Firewall Analyzer, audit-quality evidence extraction và vulnerability/risk classification**. Infrastructure_Operations_14_Da…

Đây là **bản đồ kiến thức Day 10**, chia thành đúng hai module **Lý thuyết** và **Thực hành**. Ở bước này tôi chỉ trình bày phạm vi cần học thật đầy đủ; khi bạn yêu cầu từng module sau, tôi sẽ đi sâu theo format What → Why → How → Verification → Troubleshooting → Evidence → Escalation.

# Day 10 — Module 1: Lý thuyết

## 1. Infrastructure Monitoring và Centralized Log Management

Trước tiên cần hiểu tư duy vận hành của monitoring và logging trong enterprise.

Bạn cần nắm được sự khác nhau giữa:

- **Monitoring**: theo dõi trạng thái, availability, performance, threshold, health.
- **Logging**: ghi lại event/activity đã xảy ra.
- **Alerting**: phát hiện condition đáng chú ý và gửi notification.
- **Investigation**: truy vấn và phân tích data sau khi có symptom/incident.
- **Audit evidence**: giữ bằng chứng có thể trace, verify và review sau này.

Mục tiêu không phải chỉ “xem log”, mà phải hiểu một pipeline kiểu:

```text
Infrastructure Source
        ↓
Local Log/Event
        ↓
Agent / Syslog / Collector
        ↓
Centralized Log Platform
        ↓
Parsing / Normalization
        ↓
Index / Storage
        ↓
Search / Correlation
        ↓
Alert / Report / Investigation
        ↓
Evidence / Incident / Audit
```

Day 10 yêu cầu hiểu log từ nhiều lớp infrastructure khác nhau chứ không chỉ server. Infrastructure_Operations_14_Da…

---

## 2. Các Log Source phải nhận diện được

Bạn cần phân biệt log source, mục đích, loại event và giá trị troubleshooting của từng hệ thống.

### Windows Event Logs

Cần hiểu các nhóm như:

```text
System
Application
Security
```

và các event liên quan đến:

```text
Logon / Logoff
Failed logon
Account activity
Privilege changes
Service start/stop/failure
Administrative actions
```

Không cần trở thành chuyên gia Windows auditing trong Day 10, nhưng phải biết **event thuộc host nào, channel nào, timestamp nào và ý nghĩa operational/security của nó**.

---

### Linux syslog / journald

Cần hiểu:

```text
systemd-journald
journalctl
syslog
rsyslog/syslog forwarding
```

và log của:

```text
authentication
sudo
service
kernel
network
application
```

Điểm quan trọng là phân biệt:

```text
event không tồn tại
```

với:

```text
event có tại server nhưng chưa được collector nhận
```

và:

```text
collector đã nhận nhưng parser/search không xử lý đúng
```

Đây là một theme lớn của Day 10.

---

### Nutanix Prism Events

Cần nhận biết event/alert liên quan đến:

```text
VM
AHV host
CVM
cluster
storage
capacity
hardware dependency
tasks
administrative actions
```

Day 10 không quay lại học sâu Nutanix administration như Day 6–7; Nutanix ở đây là **log/event source** trong bài toán centralized monitoring.

---

### HPE iLO Logs

Đặc biệt là các hardware/management records như:

```text
hardware health
power
temperature
memory
storage
firmware
Integrated Management Log (IML)
```

Mục tiêu là biết correlate infrastructure event với system/service symptom.

---

### FortiGate Logs

Cần phân biệt tối thiểu:

```text
traffic logs
security logs
administrative/configuration activity
allow/deny traffic
policy-related events
```

Các log này đặc biệt quan trọng khi xác định:

```text
server không truy cập được
```

là do server/network/application hay bị firewall deny.

---

### BIG-IP Logs

Cần hiểu log liên quan đến:

```text
virtual server
pool
pool member
health monitor
connection
HTTP/HTTPS
SSL/TLS
administrative activity
```

Mục tiêu vẫn là correlation, không phải học lại BIG-IP configuration của Day 9.

---

### Cisco Syslog

Phải nhận diện các event về:

```text
interface up/down
VLAN
STP
routing
configuration change
authentication
system warnings/errors
```

để liên kết với connectivity incident.

---

### AWS CloudWatch / CloudTrail

Cần phân biệt tư duy:

```text
CloudWatch
```

thường phục vụ monitoring/metrics/logs/alarms,

trong khi:

```text
CloudTrail
```

phục vụ API/activity/audit trail.

Day 10 dùng chúng như một nguồn dữ liệu để correlate cloud activity với server/firewall/application incident. Infrastructure_Operations_14_Da…

---

# 3. Timestamp, Timezone và NTP

Đây là phần cực kỳ quan trọng khi correlate nhiều hệ thống.

Bạn cần hiểu:

```text
local time
UTC
timezone offset
DST awareness
clock drift
NTP synchronization
event ingestion time
event occurrence time
```

Ví dụ:

```text
Windows Server:   10:05:21 +07
FortiGate:        03:05:24 UTC
AWS CloudTrail:   03:05:25Z
```

Có thể thực chất là cùng một chuỗi sự kiện.

Nếu timestamp normalization sai, bạn có thể:

```text
ghép nhầm các event
bỏ sót root cause
xác định sai sequence
đưa ra conclusion sai
```

Vì vậy mọi search/export phải ghi rõ **timezone**.

---

# 4. Event Severity

Bạn cần hiểu severity là **mức độ nghiêm trọng của event**, nhưng severity của log không đồng nghĩa trực tiếp với business impact.

Các mức phổ biến có thể tương tự:

```text
Informational
Notice
Warning
Error
Critical
Alert
Emergency
```

tùy product.

Một `Critical` technical event trên lab system có thể ít business impact hơn một `Warning` liên quan đến production payment service.

Do đó cần phân biệt:

```text
technical severity
business impact
exposure
operational priority
```

---

# 5. Log Source Identification

Khi đọc centralized logs phải luôn trả lời được:

```text
Event này đến từ đâu?
```

Các identifier thường liên quan:

```text
hostname
FQDN
IP address
device name
asset ID
cloud resource ID
source type
collector
log channel
facility
```

Nếu source identity không đáng tin cậy thì correlation và audit evidence đều yếu.

---

# 6. Collection Agents và Syslog Forwarding

Bạn cần hiểu hai mô hình chính.

### Agent-based collection

Một software agent chạy trên source system và gửi dữ liệu về collector.

Cần quan tâm:

```text
agent status
service status
connectivity
permissions
buffering
queue
version
configuration
```

### Agentless / forwarding

Ví dụ:

```text
Syslog
Windows event collection mechanism
network-device forwarding
API-based collection
```

Cần hiểu pipeline:

```text
Source → Network → Collector → Parser → Storage
```

để troubleshoot ingestion problem theo đúng layer.

---

# 7. Log Parsing và Normalization

Raw log thường chỉ là text hoặc structured record.

Central analyzer phải tách thành các field như:

```text
timestamp
hostname
username
source IP
destination IP
source port
destination port
protocol
action
policy
event ID
severity
message
```

Quá trình này gọi là **parsing**.

Normalization làm cho các source khác nhau có thể search/correlate chung.

Ví dụ:

```text
src
source_ip
client_ip
```

có thể cần được hiểu là cùng một semantic concept.

Parsing lỗi có thể dẫn tới:

```text
event có trong raw log
nhưng query theo field không tìm thấy
```

---

# 8. Event Correlation

Đây là một kỹ năng trung tâm của Day 10.

Bạn phải học cách xây dựng timeline từ nhiều nguồn.

Ví dụ:

```text
10:00:01 User đăng nhập Windows
10:00:15 Privilege change
10:00:30 Service restart
10:00:34 Application unavailable
10:00:36 FortiGate traffic denied
10:00:40 Monitoring alert triggered
```

Cần phân biệt:

```text
correlation
```

với:

```text
causation
```

Các event gần nhau về thời gian chưa chắc event này gây event kia.

---

# 9. Log Retention

Cần hiểu:

```text
retention period
rotation
archive
storage capacity
purge
legal/audit requirement
```

Một câu hỏi operational quan trọng là:

```text
"Tại sao không thấy log?"
```

Có thể do:

```text
không có event
source disconnected
collector failure
parser failure
retention expired
storage capacity problem
permission problem
search filter sai
```

Day 10 yêu cầu tuyệt đối không tự động kết luận “không có event”.

---

# 10. Log Access Control và Audit Data Protection

Central log platforms chứa dữ liệu rất nhạy cảm.

Phải hiểu:

```text
RBAC
least privilege
read-only investigation access
export permission
administrative access
audit trail
retention control
```

Logs có thể chứa:

```text
username
hostname
IP
resource IDs
configuration data
security events
session details
```

Vì vậy phải tránh:

```text
chỉnh sửa raw evidence
xóa log
chia sẻ quá phạm vi
expose credentials/tokens
```

---

# 11. ManageEngine EventLog Analyzer — Architecture

Đây là product trọng tâm số một của Day 10. Syllabus yêu cầu học:

```text
architecture
device onboarding
Windows/Linux/network log collection
searches
filters
security-event investigation
alerting
reporting
```

Infrastructure_Operations_14_Da…

Bạn cần hiểu ở mức operational:

```text
Log Source
    ↓
Agent / Syslog / Collection
    ↓
EventLog Analyzer
    ↓
Parsing / Indexing
    ↓
Search / Alert / Report
```

---

# 12. Device Onboarding trong EventLog Analyzer

Phải hiểu onboarding không kết thúc ở việc:

```text
device appears in console
```

mà phải verify:

```text
correct device identity
correct source type
events received
timestamp correct
expected event types present
ingestion delay acceptable
searchable
```

Syllabus bổ sung nhấn mạnh **log-source onboarding validation**. Infrastructure_Operations_14_Da…

---

# 13. EventLog Analyzer Searches và Filters

Phải học cách xây dựng search có kiểm soát bằng:

```text
time range
hostname/device
username
event ID
severity
source IP
destination IP
message
event category
```

Đặc biệt Day 10 yêu cầu tìm các event:

```text
failed logons
privilege changes
service failures
administrative activity
```

Infrastructure_Operations_14_Da…

---

# 14. Security-event Investigation

Workflow cần hình thành:

```text
Alert/Symptom
   ↓
Scope
   ↓
Identify affected assets
   ↓
Define time window
   ↓
Query logs
   ↓
Build timeline
   ↓
Cross-check sources
   ↓
Determine findings
   ↓
Preserve evidence
   ↓
Escalate/remediate
```

---

# 15. EventLog Analyzer Alerting

Cần hiểu:

```text
alert rule
condition
threshold
event match
notification
delivery status
false positive
alert fatigue
```

Không chỉ kiểm tra:

```text
rule enabled
```

mà phải verify:

```text
known event → rule match → alert generated → alert delivered
```

---

# 16. EventLog Analyzer Reporting

Phải hiểu các report dùng cho:

```text
operations
security investigation
audit
compliance
management review
```

Report phải có khả năng trace về source records.

---

# 17. EventLog Analyzer Collector/Service Health

Additive coverage yêu cầu học vận hành chính analyzer chứ không chỉ dùng search UI.

Phải kiểm tra:

```text
collector/service health
source connectivity
ingestion status
ingestion delay
storage/retention capacity
alert delivery
report scheduling
```

Infrastructure_Operations_14_Da…

---

# 18. Silent Source Problem

Một concept rất quan trọng:

```text
No matching event
```

không giống:

```text
Disconnected source
```

không giống:

```text
Parsing failure
```

không giống:

```text
Collection gap
```

Syllabus yêu cầu phải biết phân biệt bốn tình huống này. Infrastructure_Operations_14_Da…

---

# 19. ManageEngine Firewall Analyzer — Architecture

Đây là product trọng tâm thứ hai.

Phải học:

```text
firewall log ingestion
traffic analysis
rule usage
configuration change tracking
policy anomaly awareness
security audits
compliance reports
export verification
```

Infrastructure_Operations_14_Da…

Conceptual flow:

```text
Firewall
   ↓
Traffic/Security/Config logs
   ↓
Collector
   ↓
Firewall Analyzer
   ↓
Parsing
   ↓
Traffic / Policy / Security Analysis
   ↓
Reports / Audit / Investigation
```

---

# 20. Firewall Log Ingestion

Bạn phải biết verify:

```text
firewall identity
log forwarding
collector reception
parser recognition
timestamp
traffic event completeness
```

---

# 21. Traffic Analysis

Phải có khả năng phân tích:

```text
source
destination
protocol
port
action
policy
volume
time
```

Ví dụ câu hỏi:

```text
Host A có cố kết nối tới Server B không?
Firewall nhận traffic không?
Policy nào match?
Action là allow hay deny?
```

---

# 22. Firewall Rule Usage / Policy Utilization

Bạn cần hiểu các concept như:

```text
rule hit
unused rule
high-use rule
policy utilization
rule effectiveness
```

Syllabus chỉ yêu cầu **policy anomaly awareness**, nên Day 10 chưa đồng nghĩa với advanced firewall rule optimization.

---

# 23. Configuration Change Tracking

Cần biết tìm evidence về:

```text
who
what
when
device
configuration change
```

rồi liên kết với:

```text
approved change
incident timeline
administrator activity
```

---

# 24. Policy Anomaly Awareness

Bạn cần biết nhận diện ở mức foundation các tình huống như:

```text
unexpected allow
unexpected deny
unused rule
rule overlap
rule shadowing
overly broad policy
```

Nhưng Day 10 tập trung vào **analysis/reporting**, không phải thiết kế lại toàn bộ firewall policy.

---

# 25. Security Audit và Compliance Reports

Firewall Analyzer có vai trò hỗ trợ:

```text
audit
policy review
traffic evidence
configuration evidence
security reporting
```

Khi học chi tiết sau này ta sẽ luôn phân biệt:

```text
report của tool
```

với:

```text
verified audit evidence
```

Report đẹp không tự động có nghĩa data source đầy đủ.

---

# 26. Firewall Analyzer Operational Health

Tương tự EventLog Analyzer, phải kiểm tra:

```text
source onboarding
ingestion health
service/collector health
retention/capacity
report scheduling
```

đây là yêu cầu bổ sung chính thức của syllabus. Infrastructure_Operations_14_Da…

---

# 27. Vulnerability Classification Fundamentals

Day 10 bắt đầu đưa vào tư duy security/risk để chuẩn bị cho Day 13.

Phải hiểu tối thiểu:

```text
vulnerability
exposure
severity
business risk
false positive
remediation
remediation owner
escalation
```

Syllabus yêu cầu đặc biệt phân biệt:

```text
severity
```

với:

```text
business risk
```

Infrastructure_Operations_14_Da…

---

# 28. Exposure Identification

Một weakness chỉ trở thành risk thực tế khi xem xét context như:

```text
internet exposed?
internal only?
authentication required?
critical asset?
sensitive data?
compensating control?
```

Ví dụ:

```text
Critical vulnerability + isolated test VM
```

có thể có business risk khác với:

```text
High vulnerability + Internet-facing production service
```

---

# 29. False Positives

Bạn phải hiểu finding có thể sai do:

```text
incorrect asset mapping
outdated scan data
wrong version detection
misinterpreted log
duplicate finding
compensating control not recognized
```

Không được tự đóng finding mà không có evidence.

---

# 30. Finding Classification

Additive syllabus yêu cầu trước khi gán severity phải phân biệt finding là:

```text
Vulnerability
Configuration weakness
IAM exception
Operational incident
```

Infrastructure_Operations_14_Da…

Đây là một distinction rất quan trọng.

Ví dụ:

```text
CVE on exposed server
→ Vulnerability

Firewall allows overly broad source network
→ Configuration weakness

Privileged user lacks valid approval
→ IAM exception

Service crashed because disk full
→ Operational incident
```

---

# 31. Remediation Ownership

Mỗi finding phải xác định:

```text
asset owner
technical owner
security owner
remediation owner
ticket/reference
due date / escalation
```

Infrastructure Operations không nên tự sửa mọi thứ chỉ vì phát hiện được vấn đề.

---

# 32. L1 Escalation Boundary

Day 10 yêu cầu hiểu **L1 escalation boundaries**.

L1/L2 có thể thường làm:

```text
collect logs
verify source health
run approved queries
validate known events
build timeline
identify affected asset
collect evidence
apply approved runbook
```

Nhưng phải escalate khi gặp:

```text
confirmed security compromise
high/critical vulnerability ngoài authorized scope
unknown privileged activity
missing audit evidence
persistent log collection failure
collector/storage infrastructure failure
unapproved configuration change
requirement to modify production firewall/security controls
```

---

# 33. Audit-quality Extraction

Đây là phần syllabus bổ sung rất quan trọng.

Export phải ghi lại:

```text
source
source ID
query
filter
time window
timezone
extraction timestamp
record count
format
export limit
pagination
excluded interval
```

Infrastructure_Operations_14_Da…

Không chỉ đơn giản bấm:

```text
Export CSV
```

rồi gọi đó là audit evidence.

---

# 34. Evidence Integrity

Day 10 tiếp tục sử dụng evidence contract của toàn chương trình:

```text
raw/original export
working copy
filtered/derived report
SHA-256
record count
timestamps
query/filter
ticket reference
```

Raw evidence phải được giữ nguyên; filtered evidence là artifact riêng. Chương trình yêu cầu verify hash, counts và completeness thay vì xem screenshot là đủ. Infrastructure_Operations_14_Da…

---

# Day 10 — Module 2: Thực hành

Module thực hành phải chứng minh rằng bạn **không chỉ biết search log**, mà có khả năng validate toàn bộ log pipeline, investigate incident, reconcile evidence và tạo audit package. Assignment chính thức nằm ở Day 10 syllabus. Infrastructure_Operations_14_Da…

## 1. Lab Environment

Bạn sẽ cần một tập log source đại diện như:

```text
Windows Server
RHEL
FortiGate
Cisco Catalyst/Nexus
AWS
Nutanix / HPE / BIG-IP nếu lab có
```

và:

```text
ManageEngine EventLog Analyzer
ManageEngine Firewall Analyzer
```

Tất cả phải là:

```text
lab
sandbox
authorized environment
```

---

# 2. Log-source Inventory

Trước khi troubleshoot phải lập inventory:

```text
Source ID
Hostname/device
IP
Platform
Log type
Collection method
Expected event type
Timezone
Collector
Status
```

Mục tiêu là tránh tình trạng “có nhiều log nhưng không biết nguồn nào đang thiếu”.

---

# 3. Validate Windows Log Collection

Bạn sẽ tạo hoặc dùng một known event.

Sau đó kiểm tra:

```text
Source Windows event
        ↓
EventLog Analyzer reception
        ↓
Parsed fields
        ↓
Searchability
```

Phải chứng minh:

```text
host đúng
timestamp đúng
event type đúng
record tồn tại
```

---

# 4. Validate RHEL Log Collection

Tương tự Windows:

```text
Generate known service/authentication event
        ↓
Inspect source journal/syslog
        ↓
Confirm EventLog Analyzer ingestion
        ↓
Compare timestamp / host / message
```

---

# 5. Validate FortiGate Log Collection

Tạo một traffic fixture hoặc dùng scenario đã chuẩn bị:

```text
allowed connection
denied connection
```

rồi kiểm tra:

```text
FortiGate log
        ↓
Firewall Analyzer ingestion
        ↓
traffic fields
        ↓
rule/policy
        ↓
action
```

---

# 6. Validate Cisco Syslog Collection

Kiểm tra một event như:

```text
interface
authentication
configuration
```

và xác nhận centralized analyzer nhận đúng source/device.

---

# 7. Known-event Reconciliation

Đây là một acceptance requirement chính thức.

Bạn sẽ so sánh:

```text
events generated at source
```

với:

```text
events received/searchable centrally
```

Nếu tạo 10 events mà chỉ thấy 8:

```text
10 source
8 received
2 missing
```

thì phải giải thích discrepancy.

Không được chỉ ghi:

```text
collection successful
```

Infrastructure_Operations_14_Da…

---

# 8. Silent Source Investigation

Một lab fixture sẽ có source đột nhiên không có log.

Bạn phải xác định theo pipeline:

```text
Source generating logs?
        ↓
Forwarder/agent running?
        ↓
Network reachable?
        ↓
Collector healthy?
        ↓
Parser working?
        ↓
Indexed/searchable?
```

Kết luận phải là một trong các loại như:

```text
true no-event condition
disconnected source
collection failure
parsing failure
retention/search issue
```

---

# 9. Ingestion Delay Check

Phải đo hoặc ước lượng:

```text
source event timestamp
vs
central availability timestamp
```

để hiểu:

```text
ingestion delay
```

Một event vừa xảy ra nhưng chưa xuất hiện ngay chưa chắc là mất log.

---

# 10. Collector / Service Health Check

Phải kiểm tra analyzer itself:

```text
services
collectors
device status
queue
storage/capacity
ingestion health
```

để tránh chỉ troubleshoot endpoint.

---

# 11. Failed Logon Investigation

Trong EventLog Analyzer:

```text
search failed authentication events
```

lọc theo:

```text
time
host
username
source
```

sau đó xác định:

```text
frequency
affected accounts
source systems
possible pattern
```

---

# 12. Privilege-change Investigation

Search các event liên quan tới:

```text
privilege assignment/change
group membership
sudo/admin activity
```

và correlate với:

```text
user
host
ticket/change
timestamp
```

---

# 13. Service-failure Investigation

Tạo hoặc dùng fixture:

```text
service failed
```

sau đó tìm:

```text
service event
preceding events
subsequent events
related network/security logs
```

và xây timeline.

---

# 14. Administrative Activity Search

Search:

```text
administrative logon
configuration changes
service operations
security changes
```

Mục tiêu:

```text
who did what, where, when
```

---

# 15. Firewall Denied Traffic Analysis

Trong Firewall Analyzer:

```text
find denied connections
```

và xác định:

```text
source IP
destination IP
port/protocol
firewall
policy
timestamp
```

Sau đó quyết định:

```text
expected deny
```

hay:

```text
possible misconfiguration / security event
```

---

# 16. Source/Destination Flow Analysis

Phải trả lời được câu hỏi:

```text
Ai nói chuyện với ai?
Qua protocol/port nào?
Bao nhiêu traffic?
Policy nào xử lý?
Action nào xảy ra?
```

---

# 17. Firewall Policy Utilization

Xem:

```text
policy hits
usage
traffic volume
possibly unused policies
```

nhưng không tự ý thay đổi firewall policy.

---

# 18. Configuration Change Review

Tìm:

```text
configuration change record
```

sau đó correlate với:

```text
administrator
timestamp
device
ticket/change
```

---

# 19. Cross-system Incident Correlation

Đây là lab quan trọng nhất.

Ví dụ scenario:

```text
User reports application unavailable
```

Bạn phải correlate:

```text
Windows/RHEL logs
    +
FortiGate logs
    +
Cisco logs
    +
BIG-IP logs
    +
CloudWatch/CloudTrail
```

để xây:

```text
incident timeline
```

---

# 20. Timestamp Normalization Exercise

Bạn sẽ gặp logs ở nhiều timezone.

Phải normalize về một reference, ví dụ:

```text
UTC
```

hoặc:

```text
UTC+07:00
```

và ghi rõ convention trong evidence.

---

# 21. Finding Classification Exercise

Các finding được cho sẵn phải được phân loại thành:

```text
Vulnerability
Configuration weakness
IAM exception
Operational incident
```

sau đó mới đánh giá:

```text
severity
exposure
business risk
owner
```

Đây là yêu cầu trực tiếp trong additive lab. Infrastructure_Operations_14_Da…

---

# 22. False-positive Review

Phải xác minh finding bằng thêm evidence.

Ví dụ:

```text
Tool reports exposed service
```

nhưng:

```text
Firewall logs show source restricted
```

thì không được tự động kết luận vulnerability đã được loại bỏ; phải document context và escalation.

---

# 23. EventLog Analyzer Report Export

Phải xuất ít nhất:

```text
native evidence where available
```

và:

```text
filtered report
```

kèm:

```text
search filters
time window
timezone
source IDs
count
extraction time
```

---

# 24. Firewall Analyzer Report Export

Tương tự phải có:

```text
raw/native evidence where supported
filtered traffic/security report
configuration-change evidence
```

---

# 25. Export Limits và Pagination

Phải kiểm tra analyzer có:

```text
maximum rows
pagination
time-range limits
report limits
```

hay không.

Nếu UI báo:

```text
10,000 matches
```

nhưng file chỉ chứa:

```text
5,000 records
```

thì phải ghi evidence gap.

---

# 26. Excluded Interval Documentation

Nếu một source bị mất log:

```text
14:10–14:27
```

thì evidence package phải ghi:

```text
collection gap
```

chứ không được coi time range là complete.

---

# 27. Evidence Manifest

Mỗi artifact tối thiểu sẽ liên kết:

```text
Evidence ID
Source
Device/host
Environment
Collector
Time window
Timezone
Search/query/filter
Extraction timestamp
Record count
File size
SHA-256
Ticket reference
Limitations
```

Đây là chuẩn evidence chung của chương trình. Infrastructure_Operations_14_Da…

---

# 28. Preserve Raw vs Derived Evidence

Bạn phải có cấu trúc dạng:

```text
raw/
    windows-export
    rhel-export
    firewall-export

working/
    normalized-data

reports/
    filtered-investigation-report
```

Raw artifact không được sửa.

---

# 29. Evidence Count Reconciliation

Cuối lab phải reconcile:

```text
Known generated events
        ↓
Received events
        ↓
Search result
        ↓
Exported records
```

Nếu count không bằng nhau phải giải thích.

Đây chính là acceptance requirement của Day 10. Infrastructure_Operations_14_Da…

---

# 30. Risk/Finding Register

Một deliverable khác là bảng finding gồm tối thiểu:

```text
Finding ID
Type
Affected asset
Evidence
Severity
Exposure
Business impact
Owner
Recommended action
Escalation
Status
```

Syllabus yêu cầu register phải dựa trên **risk và owner**, không chỉ severity. Infrastructure_Operations_14_Da…

---

# 31. Incident Investigation Package

Đầu ra cuối thực hành Day 10 phải ghép thành một package có thể review gồm:

```text
Incident summary
Affected systems
Timeline
Queries/filters
Source logs
Correlated findings
Screenshots nếu cần
Raw exports
Filtered reports
Hashes
Record counts
Finding register
Escalation decision
Evidence manifest
```

---

# 32. Acceptance Criteria của Day 10

Một Day 10 đạt yêu cầu không phải chỉ là “đăng nhập được ManageEngine”.

Theo syllabus, bạn phải chứng minh được:

```text
Known test events đều được accounting
Counts reconcile hoặc discrepancy được giải thích

EventLog Analyzer report có đầy đủ evidence

Firewall Analyzer report có đầy đủ evidence

Ingestion health đã được kiểm tra

Silent source đã được diagnose đúng

Filters / timezone / source IDs được ghi lại

Export limits / pagination / excluded intervals được document

Risk/owner-based finding register được hoàn thành

Evidence integrity và traceability được giữ
```

Infrastructure_Operations_14_Da…

---

# Phân chia Day 10 thành 2 module để học sâu sau

**Module Lý thuyết** sẽ gồm 8 khối lớn:

```text
1. Monitoring & centralized logging foundations
2. Enterprise log sources
3. Timestamp/timezone/NTP & event severity
4. Collection, forwarding, parsing & correlation
5. Retention, access control & audit data protection
6. ManageEngine EventLog Analyzer
7. ManageEngine Firewall Analyzer
8. Vulnerability/risk classification & L1 escalation
```

**Module Thực hành** sẽ gồm 8 khối lớn:

```text
1. Build log-source inventory
2. Validate Windows/RHEL/FortiGate/Cisco ingestion
3. Known-event reconciliation
4. Diagnose silent-source/collection/parsing failures
5. Search failed logon/privilege/service/admin activity
6. Analyze firewall deny/flows/policy/config changes
7. Correlate multi-source incident and classify findings
8. Export, verify, hash and package audit evidence
```

Đây cũng là Day tạo nền trực tiếp cho **Day 11** về ServiceNow/privileged access và đặc biệt **Day 13** về **90-day IAM review + vulnerability classification**, vì syllabus xác định Day 10 là một trong các nguồn thực hành chính cho vulnerability classification và hai analyzer. Infrastructure_Operations_14_Da…

Khi bạn nói **“Module Lý thuyết”**, tôi sẽ bắt đầu giảng toàn bộ Module Lý thuyết Day 10 từ foundation → operational workflow → security → verification → troubleshooting → evidence → escalation, bao phủ từng mục phía trên mà không bỏ sót.
