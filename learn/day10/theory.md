# Day 10 — Module Lý thuyết

## Infrastructure Monitoring, Centralized Log Management, EventLog Analyzer, Firewall Analyzer & Security Finding Classification

Day 10 thuộc **Module 6 — Monitoring, ITSM, Asset Management & Privileged Access Operations**. Phần Lý thuyết của Day 10 tập trung vào bốn năng lực chính: hiểu cách enterprise infrastructure tạo và tập trung log; vận hành và đánh giá chất lượng log collection; sử dụng **ManageEngine EventLog Analyzer** và **ManageEngine Firewall Analyzer** theo góc nhìn Infrastructure Operations; và chuyển từ raw event sang một **finding có severity, exposure, business risk, owner và escalation decision hợp lý**. Đây chính xác là phạm vi Concept/Lecture được syllabus quy định. Infrastructure_Operations_14_Da…

Một điểm quan trọng: Day 10 **không phải khóa SIEM chuyên sâu**, không phải advanced threat hunting và cũng không phải khóa vulnerability management hoàn chỉnh. Mục tiêu là foundation → intermediate operational readiness: bạn phải biết dữ liệu đến từ đâu, có đáng tin không, tìm thế nào, correlate thế nào, khi nào chưa đủ bằng chứng để kết luận, lưu evidence ra sao và khi nào phải escalate.

---

# PHẦN I — TƯ DUY NỀN TẢNG VỀ MONITORING VÀ LOGGING

## 1. Infrastructure monitoring là gì?

### What

**Infrastructure monitoring** là quá trình theo dõi liên tục hoặc định kỳ tình trạng của server, network device, hypervisor, storage, firewall, load balancer, cloud resource và các infrastructure service.

Monitoring thường trả lời các câu hỏi kiểu:

```text
System đang up hay down?
CPU/memory/storage đang ở mức nào?
Interface có lỗi không?
Service có chạy không?
Cluster có healthy không?
Pool member có available không?
Backup job có thành công không?
Có security event bất thường không?
Có threshold nào vừa bị vượt không?
```

Monitoring thường làm việc với nhiều loại telemetry:

```text
metrics
logs
events
alerts
health states
traces
audit records
configuration state
```

Trong Day 10, trọng tâm chủ yếu là **logs/events**.

### Why

Nếu Infrastructure Operations chỉ phản ứng khi user báo lỗi, đội vận hành đang hoạt động hoàn toàn reactive.

Monitoring cho phép chuyển sang:

```text
detect
→ validate
→ investigate
→ respond
→ verify
→ document
```

Ví dụ CPU tăng cao chưa chắc là incident. Nhưng nếu cùng thời gian có:

```text
CPU 98%
service response time tăng
disk I/O tăng
application errors tăng
user complaints xuất hiện
```

thì nhiều source đang củng cố cùng một hypothesis.

---

# 2. Logging khác Monitoring như thế nào?

Đây là distinction phải nắm chắc.

**Monitoring** tập trung vào trạng thái và condition.

**Logging** ghi lại những event/action đã xảy ra.

Ví dụ:

```text
Monitoring:
Web server health = DOWN

Logging:
10:21:03 service stopped
10:21:04 configuration file rejected
10:21:04 process exited with code 1
```

Monitoring có thể nói:

> Có vấn đề.

Logs thường giúp trả lời:

> Điều gì đã xảy ra trước và trong lúc có vấn đề?

Một monitoring alert không nhất thiết chứa đủ root cause.

Ngược lại, hàng triệu log record chưa chắc tự động tạo ra một actionable alert.

---

# 3. Event, Log, Alert, Finding và Incident khác nhau như thế nào?

Đây là vocabulary cực kỳ quan trọng.

| Object         | Ý nghĩa                                                     |
| -------------- | ----------------------------------------------------------- |
| **Event**      | Một sự kiện xảy ra trên system                              |
| **Log record** | Record ghi lại event hoặc activity                          |
| **Alert**      | Notification được tạo khi condition/rule nào đó match       |
| **Finding**    | Kết quả phân tích cho thấy vấn đề đáng chú ý                |
| **Incident**   | Sự cố ảnh hưởng hoặc có khả năng ảnh hưởng service/security |
| **Evidence**   | Dữ liệu được bảo toàn để chứng minh hoặc hỗ trợ conclusion  |

Ví dụ:

```text
Event:
Windows account failed logon

Log:
Security Event ghi nhận failed authentication

Alert:
>10 failed logons trong 5 phút

Finding:
Repeated failed authentication against privileged account

Incident:
Suspected unauthorized credential access attempt
```

Điều này cho thấy:

```text
event ≠ alert
alert ≠ incident
log ≠ root cause
```

Infrastructure operator cần tránh nhảy từ:

```text
"tool báo đỏ"
```

sang:

```text
"đã bị tấn công"
```

mà không có validation.

---

# 4. Centralized Log Management

## 4.1. Vì sao không chỉ xem log tại từng server?

Trong môi trường enterprise, một incident có thể đi qua:

```text
Client
  ↓
Switch
  ↓
Firewall
  ↓
Load Balancer
  ↓
Application Server
  ↓
Database
  ↓
Cloud/API service
```

Nếu chỉ xem application server, bạn chỉ nhìn thấy một phần của câu chuyện.

Centralized logging tạo một nơi để:

```text
collect
parse
normalize
index
search
correlate
report
retain
```

log từ nhiều source.

Syllabus yêu cầu Day 10 hiểu centralized logs từ Windows, Linux, Nutanix, HPE, FortiGate, BIG-IP, Cisco và AWS. Infrastructure_Operations_14_Da…

---

# 5. Mental model của một Log Pipeline

Bạn nên hình dung mọi centralized logging system theo pipeline sau:

```text
[1] Event happens
        ↓
[2] Source records event
        ↓
[3] Agent / forwarder / API / collector reads it
        ↓
[4] Data crosses network
        ↓
[5] Collector receives it
        ↓
[6] Parser interprets it
        ↓
[7] Fields are normalized
        ↓
[8] Data is indexed/stored
        ↓
[9] Search/report engine queries it
        ↓
[10] Alert/report/export is generated
```

Đây là một trong những mental models quan trọng nhất của Day 10.

Khi user hỏi:

> Tại sao EventLog Analyzer không thấy event?

Không được bắt đầu bằng việc restart ngẫu nhiên.

Phải xác định event đang thất bại ở layer nào.

---

# 6. Data Quality trong logging

Một centralized logging system hữu ích chỉ khi data có chất lượng đủ tốt.

Bạn nên đánh giá tối thiểu các dimension:

| Dimension         | Câu hỏi                                        |
| ----------------- | ---------------------------------------------- |
| **Completeness**  | Có đủ event không?                             |
| **Accuracy**      | Parser có hiểu đúng event không?               |
| **Timeliness**    | Event có đến đủ nhanh không?                   |
| **Identity**      | Event có gắn đúng source không?                |
| **Integrity**     | Evidence có bị thay đổi không?                 |
| **Retention**     | Event còn tồn tại đủ lâu không?                |
| **Accessibility** | Người cần điều tra có quyền đọc không?         |
| **Traceability**  | Report có truy ngược được source record không? |

Centralized logging có data nhưng thiếu một trong các yếu tố trên vẫn có thể dẫn đến conclusion sai.

---

# PHẦN II — ENTERPRISE LOG SOURCES

# 7. Windows Event Logs

Day 10 không học lại Windows Server administration của Day 3. Ở đây Windows Server được nhìn như một **log source**.

## 7.1. Các Windows log channel quan trọng

Ở mức foundation, bạn cần quen với:

```text
System
Application
Security
```

Ngoài ra Windows còn có nhiều channel khác tùy role/application.

### System

Thường liên quan:

```text
services
drivers
operating system components
hardware/system events
startup/shutdown
```

### Application

Thông tin do application/service ghi lại.

### Security

Có thể chứa các audited security activity như:

```text
authentication
account activity
privilege use
policy changes
object access
```

nhưng **event xuất hiện hay không còn phụ thuộc auditing configuration**.

Không được giả định:

> Windows Security log chứa mọi hành động của mọi administrator.

---

# 8. Những Windows Event cần biết trong Day 10

Syllabus yêu cầu investigation các nhóm:

```text
failed logons
privilege changes
service failures
administrative activity
```

Infrastructure_Operations_14_Da…

Tư duy điều tra phải là:

```text
Who?
What?
Where?
When?
Result?
Related activity?
```

Ví dụ một failed logon record có giá trị hơn nhiều khi biết:

```text
Target user
Source host/IP
Destination host
Authentication mechanism
Failure reason
Timestamp
Frequency
```

Một record riêng lẻ thường chưa đủ conclusion.

---

# 9. Linux syslog và journald

## 9.1. journald

Trên nhiều modern Linux distributions sử dụng `systemd`, `systemd-journald` thu thập system/service events.

Operator thường truy vấn bằng:

```bash
journalctl
```

Điểm quan trọng là journal có thể lưu structured metadata bên cạnh message.

Bạn có thể gặp thông tin như:

```text
timestamp
unit
process
PID
hostname
priority
message
```

---

# 10. syslog

**syslog** vừa được dùng để nói về logging architecture/protocol, vừa thường được dùng chung để mô tả syslog-style records.

Nguồn có thể là:

```text
Linux host
router
switch
firewall
load balancer
security appliance
```

Syslog đặc biệt quan trọng vì network appliances thường không có một local Event Viewer giống Windows.

---

# 11. Linux authentication và sudo evidence

Trong Day 10, bạn cần có khả năng nhận ra các event liên quan:

```text
login
SSH authentication
sudo
service changes
system errors
kernel events
network service activity
```

Nhưng phải nhớ:

```text
"Không thấy sudo record"
```

không tự động chứng minh:

```text
"Không có sudo activity."
```

Có thể do:

```text
wrong time window
retention
wrong source
collection gap
permission
audit configuration
parser
```

Đây là mindset xuyên suốt Day 10.

---

# 12. Nutanix Prism Events

Nutanix đã được học chi tiết hơn ở Day 6–7. Day 10 chuyển góc nhìn sang **telemetry source**.

Bạn có thể gặp event liên quan:

```text
cluster health
host health
CVM
VM operation
capacity
storage
networking
administrative tasks
alerts
```

Một Nutanix alert có thể được correlate với server/application logs.

Ví dụ:

```text
09:30 AHV host warning
09:31 VM migration
09:32 guest network interruption
09:32 application timeout
```

Đây là correlation candidate.

Nhưng timeline proximity chưa chứng minh causation.

---

# 13. HPE iLO Logs

**HPE iLO** là out-of-band management plane của HPE ProLiant.

Day 10 quan tâm tới hardware event source, đặc biệt:

```text
Integrated Management Log (IML)
hardware health
power
temperature
memory
fan
storage/controller
firmware-related events
```

Ví dụ server bị reboot.

Application log có thể chỉ nói:

```text
unexpected shutdown
```

Trong khi iLO/IML có thể cho thấy:

```text
hardware/power event occurred
```

Hai nguồn bổ sung cho nhau.

---

# 14. FortiGate Logs

FortiGate là source cực kỳ quan trọng vì nó nằm trên network/security path.

Day 10 cần phân biệt các nhóm như:

```text
traffic logs
security events
administrative/configuration activity
```

Traffic log thường giúp trả lời:

```text
Traffic có tới firewall không?
Source là gì?
Destination là gì?
Protocol/port gì?
Policy nào xử lý?
Allow hay deny?
```

Nếu client không kết nối được tới server nhưng FortiGate hoàn toàn không thấy traffic, hypothesis khác hoàn toàn với trường hợp:

```text
traffic arrived
→ policy matched
→ action deny
```

---

# 15. BIG-IP Logs

BIG-IP đã được học operationally ở Day 9.

Day 10 xem BIG-IP là log source để investigate:

```text
virtual server
pool
pool member
health monitor
connection
HTTP
TLS
administrative events
```

Một application outage có thể có timeline:

```text
BIG-IP health monitor fails
        ↓
pool member marked unavailable
        ↓
traffic stops reaching backend
        ↓
users report service failure
```

Nhưng phải kiểm tra backend logs trước khi kết luận root cause.

---

# 16. Cisco Syslog

Cisco Catalyst/Nexus thường gửi operational events qua syslog.

Các nhóm bạn cần nhận diện:

```text
interface state
link errors
VLAN-related events
STP
routing
authentication
configuration events
system warnings
```

Ví dụ:

```text
09:00:00 interface down
09:00:01 routing adjacency lost
09:00:05 application monitoring fails
```

Network event có thể là upstream cause.

---

# 17. AWS CloudWatch

Ở Day 10, cần hiểu CloudWatch chủ yếu từ góc nhìn monitoring/logging.

CloudWatch có thể liên quan tới:

```text
metrics
logs
alarms
resource monitoring
```

Mental model:

```text
Resource telemetry
   ↓
CloudWatch metric/log
   ↓
Alarm/search
   ↓
Operations investigation
```

Không được nhầm CloudWatch metric với audit trail.

---

# 18. AWS CloudTrail

CloudTrail tập trung vào AWS API/activity records.

Nó hữu ích khi câu hỏi là:

```text
Ai hoặc identity nào thực hiện API action?
Action gì?
Resource nào?
Khi nào?
Request có success/failure không?
```

Ví dụ EC2 configuration đột nhiên thay đổi.

CloudWatch có thể cho thấy:

```text
service degraded
```

CloudTrail có thể hỗ trợ trả lời:

```text
configuration/API activity nào xảy ra gần thời điểm đó?
```

Hai tool phục vụ các góc nhìn khác nhau.

---

# PHẦN III — TIME, IDENTITY VÀ SEVERITY

# 19. Timestamp là nền tảng của Event Correlation

Một record không có timestamp đáng tin sẽ rất khó correlate.

Thông thường bạn cần phân biệt ba khái niệm:

```text
Event time
Ingestion time
Processing/indexing time
```

### Event time

Khi event thực sự xảy ra tại source.

### Ingestion time

Khi centralized collector nhận event.

### Processing time

Khi analyzer parse/index event thành searchable data.

Ba timestamp có thể khác nhau.

---

# 20. Timezone

Ví dụ:

```text
Windows:
2026-10-21 10:00:00 +07:00

AWS:
2026-10-21 03:00:00Z
```

`Z` biểu thị UTC.

Hai timestamp trên có thể đại diện cùng một thời điểm.

Do đó investigator không được so sánh chuỗi timestamp bằng mắt mà bỏ qua timezone.

Syllabus yêu cầu rõ **timestamp and timezone normalization**. Infrastructure_Operations_14_Da…

---

# 21. Time Normalization

Một convention tốt có thể là:

```text
All investigation timestamps normalized to UTC
```

hoặc:

```text
All timeline entries displayed in UTC+07:00
Original source timestamp retained separately
```

Điểm quan trọng không phải chọn timezone nào.

Điểm quan trọng là:

```text
consistent
documented
reproducible
```

---

# 22. NTP Awareness

**NTP — Network Time Protocol** giúp đồng bộ system clock.

Trong incident correlation, clock drift rất nguy hiểm.

Ví dụ thực tế:

```text
Server clock: +4 minutes
Firewall clock: correct
CloudTrail: UTC correct
```

Bạn có thể nhìn thấy:

```text
application failed before firewall deny
```

trong khi thứ tự thật lại ngược lại.

Vì vậy khi timeline bất hợp lý, phải xem xét:

```text
timezone
NTP state
clock offset
clock drift
```

trước khi suy luận quá sâu.

---

# 23. Ingestion Delay

**Ingestion delay** là khoảng thời gian giữa:

```text
event generated
```

và:

```text
event becomes available centrally
```

Ví dụ:

```text
Event time:     10:00:00
Searchable at:  10:00:45
```

Observed ingestion delay:

```text
45 seconds
```

Delay không nhất thiết là lỗi.

Nhưng delay tăng từ:

```text
10 seconds
```

lên:

```text
20 minutes
```

có thể chỉ ra:

```text
collector backlog
network issue
agent buffering
resource exhaustion
processing bottleneck
```

Additive syllabus yêu cầu kiểm tra ingestion delay như một phần analyzer operations. Infrastructure_Operations_14_Da…

---

# 24. Event Severity

Severity giúp phân loại mức độ technical importance của event.

Tên level phụ thuộc platform.

Bạn có thể gặp dạng:

```text
Informational
Notice
Warning
Error
Critical
Alert
Emergency
```

Nhưng điều cực kỳ quan trọng là:

> **Event severity không bằng Business risk.**

Ví dụ:

```text
Critical alert
```

trên lab VM không critical với business.

Ngược lại:

```text
Warning
```

trên production certificate sắp hết hạn có thể mang business risk lớn.

---

# 25. Severity ≠ Impact ≠ Priority ≠ Risk

Đây là distinction rất đáng nhớ.

```text
Severity
= technical seriousness

Impact
= hậu quả đối với service/business

Exposure
= vấn đề có thể bị tiếp cận/exploit tới mức nào

Risk
= likelihood/context × impact

Priority
= cần xử lý sớm tới đâu
```

Trong operations, không nên dùng một field duy nhất thay thế cho tất cả.

---

# 26. Log Source Identification

Một event phải có source identity đủ rõ.

Có thể cần:

```text
hostname
FQDN
IP address
device name
cloud resource ID
asset ID
platform
collector
source type
```

Ví dụ hai server cùng gửi:

```text
application error
```

nhưng nếu hostname parsing sai, centralized analyzer có thể khiến analyst gán lỗi cho nhầm host.

---

# 27. Source Identity Conflict

Một số tình huống khó:

```text
hostname changed
IP changed
NAT exists
duplicate hostname
device reimaged
cloud instance replaced
```

Do đó dùng chỉ một identifier có thể không đủ.

Trong audit-quality investigation, thường cần kết hợp nhiều identifiers.

Ví dụ:

```text
hostname
+ IP
+ asset ID
+ source system
+ collection timestamp
```

---

# PHẦN IV — COLLECTION ARCHITECTURE

# 28. Agent-based Collection

**Agent-based collection** sử dụng một component chạy trên hoặc gần source để đọc và forward logs.

Conceptual flow:

```text
Source log
   ↓
Agent
   ↓
Network
   ↓
Central collector/analyzer
```

Ưu điểm thường gặp:

```text
buffering
local collection
more controlled forwarding
works across some network constraints
```

Nhưng agent cũng tạo thêm operational dependency:

```text
agent service
agent version
configuration
local resource usage
queue
network connectivity
authentication
```

Official EventLog Analyzer documentation hiện tại mô tả cả **agent-based** và **agentless** collection, vì vậy cách triển khai cụ thể phải được kiểm tra theo build và architecture thực tế. [ManageEngine](https://www.manageengine.com/products/eventlog/collect-eventlogs-with-agent.html?utm_source=chatgpt.com)

---

# 29. Agentless Collection

Agentless nghĩa là không cần dedicated agent cài trên từng endpoint theo mô hình đó; collector có thể kết nối tới source bằng native mechanism/API/protocol.

Conceptual model:

```text
Collector
   ↓
connects to source
   ↓
reads/pulls event records
```

Agentless làm giảm endpoint component nhưng phụ thuộc vào:

```text
network reachability
permissions
remote service
firewall policy
authentication
protocol support
```

Current ManageEngine documentation cũng mô tả agentless Windows collection song song với agent-based collection, nhưng implementation details có thể phụ thuộc build/platform. [ManageEngine](https://www.manageengine.com/products/eventlog/collect-eventlogs.html?source=watchguard-monitoring&utm_source=chatgpt.com)

---

# 30. Push versus Pull

Một distinction hữu ích:

### Push

Source gửi logs tới collector.

Ví dụ conceptual:

```text
Firewall
   → syslog
      → collector
```

### Pull

Collector hỏi source và retrieve records.

```text
Collector
   → source
   → retrieves logs
```

Troubleshooting khác nhau đáng kể.

Push failure có thể nằm ở:

```text
source forwarding configuration
network
destination
listener
```

Pull failure có thể nằm ở:

```text
credentials
permissions
remote service
source reachability
```

---

# 31. Syslog Forwarding

Một common enterprise pattern:

```text
Device
   ↓
Syslog message
   ↓
Network
   ↓
Syslog receiver
   ↓
Parser
   ↓
Analyzer
```

Kiến thức bổ sung: các deployment có thể dùng UDP, TCP hoặc TLS transport tùy product/policy. Không được mặc định protocol/port dựa trên assumption; phải xem actual configuration.

Official EventLog Analyzer documentation mô tả syslog analyzer pipeline gồm collection → parsing/enrichment → indexing → analysis, phù hợp với mental model Day 10. [ManageEngine](https://www.manageengine.com/products/eventlog/logging-guide/syslog/syslog-analyzer.html?utm_source=chatgpt.com)

---

# 32. Collector

**Collector** là component nhận hoặc lấy logs.

Collector có thể trở thành single point of operational failure nếu:

```text
service stopped
disk full
queue stuck
network blocked
CPU overloaded
license limit reached
parser issue
database/index unavailable
```

Vì vậy phải monitor chính monitoring platform.

Đây là điểm rất hay bị fresher bỏ qua:

> Họ monitor server nhưng không monitor collector.

---

# 33. Parsing

Raw log thường chỉ là text/structured event.

Parser chuyển:

```text
Oct 21 10:12 fw01 deny src=10.1.2.3 dst=10.4.5.6 dport=443
```

thành fields:

```text
timestamp = ...
device = fw01
action = deny
source_ip = 10.1.2.3
destination_ip = 10.4.5.6
destination_port = 443
```

Nhờ đó ta mới query:

```text
action = deny
AND destination_port = 443
```

---

# 34. Parsing Failure

Một subtle failure:

```text
raw log received
```

nhưng:

```text
expected field not extracted
```

Ví dụ source gửi:

```text
srcip=10.1.1.10
```

parser cũ không hiểu field.

Kết quả:

```text
raw event exists
```

nhưng:

```text
search by Source IP returns nothing
```

Đây không phải collection failure.

Nó là **parsing/field extraction problem**.

---

# 35. Normalization

Different vendors có thể đặt tên field khác nhau:

```text
src
src_ip
source
client_ip
sourceAddress
```

Normalization ánh xạ chúng vào common concept:

```text
Source IP
```

Normalization giúp correlate cross-platform.

Nhưng normalization sai cũng nguy hiểm vì nó tạo ra cảm giác data nhất quán trong khi semantic thực tế khác.

---

# 36. Indexing

Sau parsing, system thường index data để search nhanh.

Có thể xảy ra trường hợp:

```text
collector received record
```

nhưng:

```text
indexing delayed
```

vì thế UI search chưa thấy event.

Một investigator tốt không lập tức gọi đó là data loss.

---

# 37. Event Correlation

**Correlation** là quá trình kết nối những event có liên quan theo:

```text
time
asset
user
IP
session
resource
activity
```

Ví dụ:

```text
09:59:55 admin login
10:00:04 firewall policy change
10:00:09 denied connections spike
10:00:12 application health check fails
```

Đây là một correlated sequence.

---

# 38. Correlation không phải Causation

Nếu:

```text
A xảy ra trước B
```

không tự động nghĩa:

```text
A gây B
```

Bạn cần supporting evidence.

Ví dụ policy change trước outage.

Để kết luận policy change gây outage, nên kiểm tra:

```text
affected traffic matches changed rule?
rollback restores service?
other dependencies healthy?
similar paths unaffected?
```

Infrastructure Operations phải tránh post hoc assumption.

---

# 39. Correlation Window

Search time range phải đủ rộng để capture precursor và aftermath.

Nếu incident xảy ra:

```text
10:00
```

mà bạn chỉ query:

```text
09:59:59–10:00:01
```

có thể bỏ mất nguyên nhân xảy ra lúc 09:57.

Ngược lại query 90 ngày có thể tạo quá nhiều noise.

Time window phải có rationale.

---

# PHẦN V — RETENTION, SECURITY VÀ EVIDENCE PROTECTION

# 40. Log Retention

**Retention** định nghĩa log được giữ trong bao lâu.

Có thể chịu ảnh hưởng bởi:

```text
policy
regulatory requirement
storage capacity
product configuration
license
archive strategy
```

Retention khác backup.

Retention trả lời:

> Data còn searchable/available trong thời gian nào?

Backup trả lời:

> Có recovery copy không?

---

# 41. Rotation

Local logs thường có rotation.

Ví dụ conceptual:

```text
current.log
current.log.1
current.log.2
```

hoặc journal/archive mechanism tương tự.

Nếu investigator chỉ nhìn current file có thể bỏ mất event đã rotated.

---

# 42. Collection Gap

**Collection gap** là khoảng thời gian dữ liệu đáng lẽ được thu nhưng không có hoặc không thể chứng minh đầy đủ.

Ví dụ:

```text
13:00–13:23 collector offline
```

Không được viết report:

```text
No suspicious events from 12:00–14:00
```

mà phải nói:

```text
No matching events were observed in available data,
but 13:00–13:23 is an identified collection gap.
```

Syllabus đặc biệt yêu cầu document **excluded intervals** và collection gaps. Infrastructure_Operations_14_Da…

---

# 43. Log Access Control

Centralized logs thường chứa sensitive information.

Do đó phải áp dụng:

```text
RBAC
least privilege
separation of duties
restricted export
audit trail
```

Một operations user có thể chỉ cần:

```text
read/search/export approved logs
```

không nhất thiết cần quyền:

```text
delete data
change retention
disable source
edit parsing rules
change user access
```

---

# 44. Vì sao quyền Admin vào Analyzer là High Risk?

Một administrator quá quyền có thể:

```text
disable collection
change retention
modify source configuration
remove devices
change alert rules
modify users
```

Điều này vừa ảnh hưởng operations vừa có thể ảnh hưởng audit reliability.

Day 10 vì thế không chỉ là log search; nó còn là **audit data protection**. Infrastructure_Operations_14_Da…

---

# 45. Evidence Integrity

Syllabus toàn chương trình đặt một evidence contract rất rõ: raw export phải được giữ nguyên; working copy/filtered deliverable là artifact riêng; mỗi evidence item cần source, time, query/filter, counts, hash và limitation phù hợp. Infrastructure_Operations_14_Da…

Một model tốt:

```text
RAW
 ↓ copy
WORKING DATA
 ↓ transformation
FILTERED REPORT
 ↓
FINAL DELIVERABLE
```

Không được:

```text
RAW
 ↓ edit
overwrite RAW
```

---

# 46. Hash chứng minh điều gì?

Ví dụ:

```text
SHA-256(file) = X
```

Sau đó file vẫn hash X.

Bạn có thể nói:

> File hiện tại giống byte-for-byte với file đã hash trước đó.

Nhưng không được nói:

> Hash chứng minh file này là complete và authentic.

Hash không tự chứng minh:

```text
source trustworthy
query correct
time window complete
all records exported
```

Syllabus nhấn mạnh chính distinction này. Infrastructure_Operations_14_Da…

---

# PHẦN VI — MANAGEENGINE EVENTLOG ANALYZER

# 47. Role của EventLog Analyzer trong Day 10

Syllabus yêu cầu:

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

và phần bổ sung yêu cầu thêm:

```text
onboarding validation
collector/service health
ingestion delay
retention/capacity
alert-delivery checks
report scheduling
```

Infrastructure_Operations_14_Da…

Current official documentation vẫn trình bày EventLog Analyzer như một centralized log-management platform với Standalone/Distributed deployment documentation, supported log sources, agent configuration và architecture guidance. Exact deployment phải kiểm tra theo actual build. [ManageEngine](https://www.manageengine.com/products/eventlog/resources.html?utm_source=chatgpt.com)

---

# 48. Conceptual EventLog Analyzer Architecture

Không phụ thuộc menu/version, hãy nhớ flow:

```text
Windows / Linux / Network / Application
              ↓
       Collection Layer
              ↓
       Parsing Layer
              ↓
      Storage / Index
              ↓
    Search / Correlation
      ↙       ↓       ↘
 Alerts    Reports    Investigation
```

Bạn nên luôn nghĩ từ dưới source lên thay vì bắt đầu từ UI.

---

# 49. Device Onboarding

**Onboarding** không đơn giản là thêm hostname vào console.

Một source chỉ thực sự onboard thành công khi có thể chứng minh:

```text
Correct source identified
        ↓
Collection configured
        ↓
Known event generated
        ↓
Known event received
        ↓
Correctly parsed
        ↓
Searchable
        ↓
Timestamp validated
```

Một device hiện trạng thái `Added` chưa đủ.

---

# 50. Known-event Validation

Đây là kỹ thuật rất quan trọng.

Thay vì hỏi:

> Analyzer có hoạt động không?

Hãy tạo một event biết trước.

Ví dụ lab:

```text
10:01:00 controlled failed logon generated
```

Sau đó tìm chính event đó.

Nếu thấy:

```text
host correct
user correct
timestamp correct
event category correct
```

thì confidence cao hơn nhiều.

---

# 51. Windows Collection trong EventLog Analyzer

Tùy environment/build, collection có thể dùng mechanisms khác nhau.

Current vendor documentation mô tả cả agentless và agent-based collection, vì vậy bạn không nên memorize một architecture duy nhất cho mọi deployment. [ManageEngine](https://www.manageengine.com/products/eventlog/collect-eventlogs.html?source=watchguard-monitoring&utm_source=chatgpt.com)

Operator phải biết actual environment:

```text
collection type
source credentials
required connectivity
agent status if used
channels collected
```

---

# 52. Linux/Network Collection

Common pattern là:

```text
syslog forwarding
```

nhưng actual implementation có thể phụ thuộc device/vendor.

Onboarding phải verify không chỉ rằng packets đến collector mà còn rằng:

```text
correct device appears
correct parser selected
expected fields populated
event searchable
```

---

# 53. Search

Search là foundation của investigation.

Một good query thường bắt đầu rộng vừa đủ rồi thu hẹp.

Ví dụ conceptual:

```text
Time window
+ Host
+ Event category
```

sau đó refine:

```text
+ Username
+ Event ID
+ Source IP
+ Message pattern
```

Không nên bắt đầu với filter cực hẹp nếu chưa hiểu data schema.

---

# 54. Filters

Common filter dimensions:

```text
time
device
hostname
username
event ID
severity
source IP
destination IP
message
event category
```

Nhưng filter field chỉ đáng tin khi parser đã populate đúng.

Nếu query:

```text
Source IP = X
```

không có kết quả, cần cân nhắc:

```text
No event?
Wrong field?
Parser failure?
Different field name?
```

---

# 55. Failed-logon Investigation

Không chỉ count failed logons.

Phải đặt câu hỏi:

```text
One user, many source hosts?
Many users, one source?
One host repeatedly failing?
Privileged account affected?
Success occurred after failures?
```

Ví dụ:

```text
100 failures
```

có thể là:

```text
misconfigured service account
```

hoặc:

```text
credential attack
```

Context quyết định.

---

# 56. Privilege-change Investigation

Bạn cần tìm liên kết giữa:

```text
identity
privilege change
host
administrator
timestamp
ticket/change
```

Nếu privilege change hợp lệ nhưng không có ticket reference, đây có thể là governance exception chứ chưa chắc security incident.

---

# 57. Service-failure Investigation

Service failure nên được correlate với:

```text
service events
OS events
configuration changes
authentication
disk/capacity
network events
application events
```

Ví dụ:

```text
service failed
```

không đồng nghĩa:

```text
service binary broken
```

Có thể là:

```text
permission
dependency
configuration
disk full
network dependency
certificate
```

---

# 58. Administrative Activity

Các activity đáng chú ý:

```text
login
configuration modification
service control
privilege changes
account changes
security configuration
```

Infrastructure Operations thường cần trả lời:

> Activity này có approved change không?

Do đó Day 10 tự nhiên kết nối với Day 11 ServiceNow.

---

# 59. EventLog Analyzer Alerting

Alerting logic thường có dạng:

```text
Condition
        ↓
Match
        ↓
Alert generated
        ↓
Notification/action
```

Validation đầy đủ không phải:

```text
Rule = enabled
```

mà là:

```text
Known event generated
        ↓
Rule matches
        ↓
Alert object exists
        ↓
Notification delivered
```

Additive syllabus yêu cầu **alert-delivery checks** chứ không chỉ configuration inspection. Infrastructure_Operations_14_Da…

---

# 60. False Negative và False Positive trong Alerting

### False positive

Alert nói có vấn đề nhưng thực tế không có actionable issue.

### False negative

Issue xảy ra nhưng alert không trigger.

False negative nguy hiểm vì operator có thể tin:

```text
No alert = no problem
```

trong khi rule/collection có thể lỗi.

---

# 61. Alert Fatigue

Nếu rule quá rộng:

```text
1000 alerts/day
```

operator có thể bỏ qua alert thật.

Đây gọi là **alert fatigue**.

Alert design phải balance:

```text
sensitivity
specificity
operational usefulness
```

Day 10 chỉ cần hiểu concept; không yêu cầu thiết kế full detection engineering program.

---

# 62. Reporting

Reports dùng để:

```text
operations review
security review
audit
compliance support
management reporting
```

Nhưng report chỉ đáng tin bằng data pipeline phía dưới.

Nếu 20% devices disconnected, report đẹp vẫn incomplete.

---

# 63. Scheduled Reports

Additive syllabus yêu cầu biết kiểm tra **report scheduling**.

Có ba lớp verification:

```text
Schedule exists
        ↓
Report generated
        ↓
Report delivered/stored correctly
```

Nếu report không đến email, không nên lập tức kết luận report engine lỗi.

Có thể:

```text
mail delivery
recipient
permission
schedule
generation failure
```

---

# 64. Collector/Service Health

Khi EventLog Analyzer có vấn đề, operator phải kiểm tra chính platform:

```text
collector health
service health
source connection
ingestion status
storage
retention
processing delay
```

Mục tiêu không phải biết mọi internal daemon name của mọi version.

Mục tiêu là biết **các health dimension cần verify**.

---

# 65. Silent Source

Một **silent source** là source đáng lẽ có events nhưng đột nhiên không còn dữ liệu nhận được.

Đây là scenario trung tâm của Day 10.

Phải phân biệt bốn trường hợp syllabus chỉ rõ: Infrastructure_Operations_14_Da…

| Tình trạng              | Ý nghĩa                                                     |
| ----------------------- | ----------------------------------------------------------- |
| **No matching event**   | Collection hoạt động, nhưng query không có record phù hợp   |
| **Disconnected source** | Analyzer không còn communication đúng với source            |
| **Parsing failure**     | Raw event tới nhưng fields không được interpret đúng        |
| **Collection gap**      | Có khoảng thời gian dữ liệu thiếu/không chứng minh complete |

Đây là một trong những tiêu chí quan trọng nhất để đánh giá bạn thực sự hiểu centralized logging hay chưa.

---

# PHẦN VII — MANAGEENGINE FIREWALL ANALYZER

# 66. Vai trò của Firewall Analyzer

Syllabus yêu cầu Day 10 bao phủ:

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

Current vendor material cũng mô tả Firewall Analyzer theo hướng traffic/security log analysis, policy analysis, configuration change monitoring và compliance reporting. Tuy nhiên availability của từng capability phụ thuộc firewall vendor, mode, build và edition, nên lab phải verify actual support. [ManageEngine](https://www.manageengine.com/products/firewall/firewall-analyzer-brochure.pdf?utm_source=chatgpt.com)

---

# 67. Conceptual Firewall Analyzer Architecture

Bạn nên hình dung hai loại input lớn:

```text
A. Operational logs
   traffic/security/syslog

B. Configuration/policy data
   firewall rules/configurations
```

Hai loại data không giống nhau.

Flow:

```text
Firewall
  ├── Traffic/Security Logs ──→ Log Ingestion
  │                               ↓
  │                       Traffic Analysis
  │
  └── Policy/Configuration ──→ Rule/Config Analysis
                                  ↓
                      Change / Policy / Compliance
```

Đây là distinction quan trọng.

---

# 68. Firewall Log Ingestion

Trước khi traffic report có giá trị, phải verify:

```text
correct firewall
correct source identity
correct timestamp
log forwarding healthy
parser recognizes vendor format
traffic events populated
```

Nếu FortiGate gửi log nhưng analyzer parse nhầm format, traffic report có thể thiếu fields.

---

# 69. Traffic Analysis

Traffic record thường giúp trả lời:

```text
Who communicated?
From where?
To where?
Which service?
Was it allowed?
Which firewall/policy handled it?
When?
How much?
```

Common fields:

```text
source IP
destination IP
source port
destination port
protocol
action
policy
user
bytes
timestamp
```

Field availability phụ thuộc firewall/log format.

---

# 70. Allowed Traffic

`allow` không đồng nghĩa với:

```text
application worked
```

Firewall chỉ chứng minh rằng tại layer của firewall, connection/traffic được permitted theo observed record.

Application vẫn có thể fail vì:

```text
server down
service not listening
load balancer issue
TLS failure
application error
```

---

# 71. Denied Traffic

`deny` cũng phải interpret đúng context.

Denied packet có thể là:

```text
expected security control
misconfigured client
unauthorized scan
wrong destination
incorrect firewall policy
```

Không phải mọi deny đều là incident.

---

# 72. Source/Destination Flow Analysis

Ví dụ incident:

```text
Client 10.1.1.25 cannot reach 10.2.5.10:443
```

Firewall Analyzer có thể giúp trả lời:

```text
Có traffic từ 10.1.1.25 không?
Nó có đến đúng firewall không?
Destination có đúng không?
Port có đúng 443 không?
Action allow hay deny?
Policy nào match?
```

Nếu traffic không thấy trên firewall, troubleshooting boundary di chuyển upstream.

---

# 73. Rule Usage

Một firewall policy có thể có:

```text
high hit count
low hit count
zero observed hits
```

Điều này hỗ trợ operational review.

Nhưng:

> Zero hits trong observed period không tự động chứng minh rule không cần thiết.

Có thể rule chỉ dùng cho:

```text
quarterly process
DR failover
emergency access
seasonal operation
```

Rule removal luôn cần governance/change workflow.

---

# 74. Policy Utilization

Rule utilization analysis giúp:

```text
identify frequently used rules
identify apparently unused rules
support cleanup review
support performance/security review
```

Nhưng Day 10 chỉ yêu cầu hiểu và phân tích, không tự ý thay đổi production policy.

---

# 75. Configuration Change Tracking

Một good change record cần trả lời:

```text
Who?
What?
When?
Where?
```

và operations cần thêm:

```text
Why?
Approved ticket?
Expected change?
Rollback available?
```

Current Firewall Analyzer materials mô tả configuration-change monitoring nhằm ghi lại change trail và administrator context, nhưng actual data fields/capabilities phải kiểm tra trên deployed version/device. [ManageEngine](https://www.manageengine.com/products/firewall/Firewall-Best-Practices-Firewall-Analyzer.pdf?utm_source=chatgpt.com)

---

# 76. Authorized vs Unauthorized Change

Một configuration change không nhất thiết là security problem.

Nếu:

```text
approved change window
correct operator
correct scope
expected result
```

thì đó có thể là normal operations.

Nếu:

```text
no ticket
unknown user
outside window
unexpected rule
```

thì cần investigation/escalation.

---

# 77. Policy Anomaly Awareness

Syllabus yêu cầu **awareness**, không yêu cầu advanced firewall policy engineering.

Một số anomaly concepts phổ biến gồm:

### Redundancy

Một rule có thể không thêm behavior khác biệt vì rule khác đã cover cùng traffic.

### Shadowing

Một earlier rule khiến later rule không bao giờ match trong logic nhất định.

### Overly Broad Policy

Rule cho phép scope quá rộng, ví dụ conceptual:

```text
any → any → broad services
```

### Correlation/Overlap

Hai rule overlap một phần về matching condition.

Current ManageEngine material có policy-analysis reporting quanh redundancy, shadowing và các policy anomalies, nhưng support và terminology cụ thể cần verify theo firewall/version. [ManageEngine](https://www.manageengine.com/products/firewall/blog/new-enhancements-firewall-analyzer-fire-firewall-devices.html?utm_source=chatgpt.com)

---

# 78. Tại sao Policy Anomaly không được Auto-fix?

Vì firewall policy chứa business logic.

Một apparently redundant rule có thể tồn tại vì:

```text
future migration
emergency requirement
temporary approved exception
specific logging requirement
DR
```

Infrastructure Operations không nên tự động delete rule dựa trên analyzer suggestion.

---

# 79. Security Audit

Security audit report có thể giúp tìm:

```text
risky policies
configuration weaknesses
policy usage
security events
```

Nhưng report là input cho decision chứ không thay decision.

Một report nói:

```text
High Risk
```

không miễn trách nhiệm verify:

```text
asset
exposure
context
false positive
business impact
```

---

# 80. Compliance Reports

Compliance report thường map configuration/log findings vào một framework/control.

Nhưng phải hiểu:

> Tool-generated compliance report không tự động chứng minh organization compliant.

Compliance cần nhiều control ngoài firewall:

```text
process
people
evidence
policy
scope
exception management
```

Analyzer report chỉ là một phần evidence.

---

# 81. Rule Data Acquisition

Current Firewall Analyzer documentation cho thấy một số supported deployments có thể lấy rule/policy bằng mechanisms như CLI/API/file, nhưng phương thức nào hoạt động phụ thuộc vendor/device/version. [download.manageengine.com](https://download.manageengine.com/products/firewall/Firewall-Analyzer-Quick-Start-Guide.pdf?utm_source=chatgpt.com)

Vì vậy phải tránh assumption:

```text
All firewalls use same collection mechanism.
```

---

# PHẦN VIII — VULNERABILITY VÀ SECURITY FINDING CLASSIFICATION

# 82. Vì sao Day 10 học Vulnerability Classification?

Day 10 là lúc learner chuyển từ:

```text
"Tôi thấy log."
```

sang:

```text
"Tôi hiểu finding này thuộc loại gì, mức độ nào và ai xử lý."
```

Syllabus yêu cầu:

```text
vulnerability classification fundamentals
exposure identification
severity versus business risk
false positives
remediation ownership
L1 escalation boundaries
```

Infrastructure_Operations_14_Da…

---

# 83. Vulnerability

Một **vulnerability** là weakness có khả năng bị exploit gây ảnh hưởng security.

Ví dụ conceptual:

```text
known vulnerable software version
authentication weakness
exploitable service flaw
```

Ở Day 10 bạn chưa cần đi sâu CVE/CVSS như Day 13.

---

# 84. Configuration Weakness

Không phải mọi security weakness đều là software vulnerability.

Ví dụ:

```text
overly broad firewall rule
unnecessary exposed service
weak logging configuration
insecure administrative setting
```

đây có thể là **configuration weakness**.

---

# 85. IAM Exception

Ví dụ:

```text
privileged account still active after approval expiry
missing owner
missing approval
excessive entitlement
```

Đây có thể là **IAM exception**.

Không nên ghi tất cả thành vulnerability.

---

# 86. Operational Incident

Ví dụ:

```text
service failed because disk full
collector stopped
firewall log forwarding unavailable
```

đây có thể là **operational incident**.

Syllabus đặc biệt yêu cầu classify finding thành:

```text
vulnerability
configuration weakness
IAM exception
operational incident
```

trước khi assign severity. Infrastructure_Operations_14_Da…

---

# 87. Tại sao Classification trước Severity?

Nếu không xác định loại finding, team dễ giao sai owner.

Ví dụ:

```text
Privileged access expired but still active
```

Nếu gọi đơn giản:

```text
High vulnerability
```

có thể assign sang patching team — sai.

Đúng hơn:

```text
Type: IAM exception
Owner: IAM/PAM team
Risk: assess based on privilege + exposure + activity
```

---

# 88. Exposure

**Exposure** mô tả mức độ asset/weakness có thể bị tiếp cận hoặc khai thác.

Những question quan trọng:

```text
Internet-facing?
Internal only?
Segment restricted?
Authentication required?
Privileged access required?
Compensating firewall control exists?
Affected service actually enabled?
```

Exposure thay đổi business risk rất mạnh.

---

# 89. Asset Criticality

Cùng một weakness nhưng:

```text
dev test VM
```

và:

```text
production payment server
```

không có cùng business risk.

Asset context cần gồm:

```text
business service
data sensitivity
availability requirement
dependencies
recovery impact
```

---

# 90. Severity vs Business Risk

Giả sử scanner nói:

```text
Severity = Critical
```

Đó là technical indicator.

Business risk phải xem thêm:

```text
exploitability
exposure
asset criticality
business impact
controls
```

Một mental model đơn giản:

```text
Technical severity
      +
Exposure
      +
Asset/business context
      +
Existing controls
      ↓
Operational/business risk
```

Không nhất thiết dùng công thức số cụ thể ở Day 10.

---

# 91. False Positive

**False positive** là finding được tool báo nhưng sau validation không phản ánh condition thực tế theo cách tool kết luận.

Ví dụ:

```text
Scanner believes vulnerable version is exposed
```

nhưng actual validation cho thấy:

```text
service not running
correct patched package actually active
version detection inaccurate
```

Tuy nhiên phải có evidence.

Không được đóng false positive chỉ vì:

```text
"owner nói chắc là không sao."
```

---

# 92. False Positive khác Compensating Control

Đây là distinction hay.

Nếu vulnerability thực sự tồn tại nhưng firewall ngăn external exploitation:

```text
Vulnerability = real
Compensating control = exists
```

Đây không phải false positive.

Risk có thể giảm, nhưng weakness vẫn có thật.

---

# 93. Remediation Ownership

Một finding cần owner rõ.

Ví dụ:

| Finding                          | Possible owner        |
| -------------------------------- | --------------------- |
| Windows patch vulnerability      | Windows Server team   |
| RHEL package vulnerability       | Linux/RHEL team       |
| Firewall policy weakness         | Network/Security      |
| IAM exception                    | IAM/PAM               |
| Collector failure                | Monitoring/Operations |
| Unsupported application weakness | Application owner     |

L1 không nên tự sửa hệ thống ngoài authority chỉ vì họ phát hiện vấn đề.

---

# 94. Detection Owner và Remediation Owner có thể khác nhau

Ví dụ:

```text
Monitoring team detects vulnerability
```

nhưng:

```text
Linux team patches it
```

Hoặc:

```text
Operations detects firewall deny issue
```

nhưng:

```text
Network Security owns policy change
```

Đây là normal enterprise operating model.

---

# 95. L1/L2 Boundary

L1/L2 thường được kỳ vọng:

```text
identify symptom
verify source
collect evidence
run approved searches
correlate basic timeline
follow runbook
classify finding
open/update ticket
escalate
```

Nhưng không tự động được quyền:

```text
change production firewall
disable security controls
delete logs
modify retention
grant privileged access
patch critical systems
change analyzer security configuration
```

Quyền thực tế luôn theo organization RBAC/runbook.

---

# 96. Khi nào phải Escalate?

Escalation thường phù hợp khi:

```text
suspected compromise
unknown privileged activity
critical/high-risk security finding
production security control change required
source collection gap cannot be restored
data integrity questionable
collector capacity exhausted
unexpected configuration change
multiple systems affected
root cause outside support boundary
```

Escalate không có nghĩa:

> Bỏ việc cho team khác.

Một good escalation cần package đủ:

```text
symptom
scope
timeline
evidence
queries performed
results
what has been ruled out
risk
recommended owner
```

---

# PHẦN IX — PHÂN BIỆT “NO EVENT” VÀ “LOGGING FAILURE”

Đây là phần cần hiểu sâu nhất của Day 10.

Giả sử query không trả kết quả.

Không được nói ngay:

```text
No activity occurred.
```

Hãy dùng model:

```text
1. Event should have happened?
             ↓
2. Source actually generated it?
             ↓
3. Local record exists?
             ↓
4. Forwarder/agent read it?
             ↓
5. Network carried it?
             ↓
6. Collector received it?
             ↓
7. Parser understood it?
             ↓
8. Record indexed?
             ↓
9. Search filter correct?
             ↓
10. Export complete?
```

---

# 97. Case A — No Matching Event

Source healthy.

Known events arrive.

Query works.

Nhưng requested condition không có match.

Kết luận có thể là:

```text
No matching record was observed in the searched dataset.
```

Đây là câu rất khác với:

```text
The activity did not occur.
```

---

# 98. Case B — Disconnected Source

Ví dụ:

```text
last event = 08:15
current time = 11:00
source normally produces constant traffic
```

Có thể source disconnected.

Investigation:

```text
source availability
forwarding configuration
agent/service
network
collector status
```

---

# 99. Case C — Parsing Failure

Raw data có:

```text
user=admin action=deny
```

nhưng structured field:

```text
User = NULL
Action = NULL
```

Search bằng fields sẽ sai.

Đây là parser problem, không phải log absence.

---

# 100. Case D — Collection Gap

Collector down 20 phút.

Events trong khoảng đó có thể:

```text
buffered and later forwarded
```

hoặc:

```text
permanently lost
```

tùy architecture.

Cho đến khi chứng minh complete, phải ghi gap.

---

# 101. Case E — Wrong Search

Một common human error:

```text
wrong timezone
wrong host
wrong field
wrong event ID
too narrow time range
wrong analyzer/device
```

Không nên troubleshoot infrastructure trước khi kiểm tra query itself.

---

# PHẦN X — EVIDENCE QUALITY

# 102. Original Evidence vs Derived Evidence

**Original/native evidence** là data được export từ source/analyzer mà chưa biến đổi nội dung.

**Derived evidence** có thể là:

```text
filtered CSV
normalized timeline
pivot table
summary report
```

Derived artifact phải trace được về source.

---

# 103. Evidence Manifest

Syllabus chung yêu cầu evidence item có metadata như:

```text
Evidence ID
source system/product/version
environment
asset/account identifier
ticket
collector
collection time/timezone
requested interval
filters/commands
format/encoding
file name/path
size
record count
SHA-256
limitations/errors
```

Infrastructure_Operations_14_Da…

Đây không phải bureaucracy.

Nó cho reviewer khả năng reproduce investigation.

---

# 104. Search Criteria phải được lưu

Ví dụ report:

```text
"12 failed logons found."
```

không đủ.

Cần biết:

```text
Which devices?
What time range?
Which timezone?
Which event criteria?
Which excluded sources?
When was query executed?
```

Nếu không, con số `12` không reproducible.

---

# 105. Record Counts

Một strong evidence flow:

```text
Source known events = 100
Analyzer searchable = 100
Exported rows = 100
```

Nếu:

```text
Source = 100
Analyzer = 97
```

thì cần explain 3 missing.

Acceptance criteria Day 10 yêu cầu counts reconcile hoặc discrepancy phải được giải thích. Infrastructure_Operations_14_Da…

---

# 106. Export Limits

Giả sử search UI nói:

```text
25,000 results
```

nhưng exported CSV:

```text
10,000 rows
```

Nếu không biết export cap, analyst có thể nghĩ:

```text
10,000 total events
```

=> conclusion sai.

Vì vậy Day 10 yêu cầu ghi:

```text
export limits
pagination
excluded intervals
```

Infrastructure_Operations_14_Da…

---

# 107. Pagination

Một analyzer có thể chia result:

```text
Page 1
Page 2
...
```

Export có thể áp dụng:

```text
current page only
all filtered records
maximum N rows
```

Tùy product/version.

Không được assume.

---

# 108. Screenshots

Screenshot hữu ích cho:

```text
UI state
visual confirmation
alert screen
device health view
```

Nhưng screenshot không lý tưởng làm sole evidence khi structured export có thể lấy được.

Syllabus quy định screenshots nên bổ sung cho exportable records chứ không thay thế chúng mà không giải thích. Infrastructure_Operations_14_Da…

---

# PHẦN XI — SECURITY VÀ LEAST PRIVILEGE TRONG LOG OPERATIONS

# 109. Read-only trước Configuration Change

Day 10 phải hình thành habit:

```text
Observe
→ Validate
→ Correlate
→ Decide
→ Authorized change if required
```

không phải:

```text
See error
→ Change configuration
```

Read-only operations thường gồm:

```text
search
filter
report
health inspection
export
```

Configuration-changing operations có thể gồm:

```text
add/remove source
change retention
modify parser
change forwarding
change alert rules
change RBAC
```

Hai nhóm phải được phân biệt.

---

# 110. Credential Protection

Không để:

```text
password
API token
session token
private key
secret
```

xuất hiện trong:

```text
screenshots
exports
commands transcript
tickets
reports
```

Nếu log bản thân chứa secret do application logging bug, đó lại là một finding cần xử lý.

---

# 111. Log Tampering Risk

Một attacker hoặc privileged insider có thể muốn:

```text
delete logs
disable auditing
disable forwarding
alter timestamps
```

Do đó centralized collection có security value vì evidence được đưa ra khỏi source system.

Nhưng centralized system cũng phải được bảo vệ.

---

# PHẦN XII — END-TO-END INCIDENT REASONING

Hãy xét scenario:

```text
User:
"Ứng dụng không truy cập được từ 14:05."
```

Một fresher dễ bắt đầu:

```text
restart application
```

Day 10 muốn bạn suy nghĩ khác.

---

# 112. Step 1 — Establish Scope

Xác định:

```text
one user?
one subnet?
all users?
one server?
all backends?
```

Scope rất quan trọng.

---

# 113. Step 2 — Establish Time

Xác định:

```text
first observed time
timezone
duration
```

Và validate system clocks.

---

# 114. Step 3 — Check Relevant Sources

Possible sources:

```text
Windows/RHEL application host
BIG-IP
FortiGate
Cisco
CloudWatch
CloudTrail
```

---

# 115. Step 4 — Build Timeline

Ví dụ:

```text
14:02:10 admin authentication
14:03:01 firewall configuration change
14:04:55 denied traffic starts
14:05:03 BIG-IP health alert
14:05:10 user impact begins
```

Đây là correlation.

Chưa phải root cause.

---

# 116. Step 5 — Validate Hypothesis

Hypothesis:

```text
Firewall change caused outage.
```

Check:

```text
Denied traffic matches application path?
Changed rule relevant?
Other paths affected?
Rollback evidence?
```

---

# 117. Step 6 — Classify

Có thể có:

```text
Operational incident
+
Configuration weakness
```

Nếu change không được approve:

```text
governance/security concern
```

cũng cần escalate.

---

# 118. Step 7 — Preserve Evidence

Giữ:

```text
raw firewall logs
server logs
BIG-IP logs
change record
search criteria
timeline
hashes
counts
```

---

# 119. Step 8 — Escalate Correctly

Một useful escalation:

```text
Impact:
Application unavailable for subnet X.

Timeline:
14:03 firewall change
14:04:55 denies begin
14:05 user impact

Evidence:
FW-001, SRV-004, LB-003

Current finding:
Traffic matching application path is denied by policy Y
after configuration change Z.

Action required:
Network Security review under change ticket CHG-...
```

Tốt hơn rất nhiều so với:

```text
Firewall seems broken. Please check.
```

---

# PHẦN XIII — COMMON MISTAKES CẦN TRÁNH

| Mistake                                                | Vì sao sai                              |
| ------------------------------------------------------ | --------------------------------------- |
| `No logs = no activity`                                | Có thể collection failure               |
| `Critical log = critical business incident`            | Severity không bằng business risk       |
| `Same timestamp = causation`                           | Correlation không chứng minh root cause |
| Chỉ xem một log source                                 | Bỏ dependency                           |
| Không normalize timezone                               | Timeline sai                            |
| Không kiểm tra NTP                                     | Event order có thể sai                  |
| Thấy device trong console là coi onboarding thành công | Chưa validate ingestion                 |
| Chỉ xem green health icon                              | Chưa kiểm known event                   |
| Parser field trống = source không gửi log              | Có thể parsing failure                  |
| Report có data = data complete                         | Có thể source gap                       |
| CSV export = audit evidence hoàn chỉnh                 | Cần provenance/count/filter/hash        |
| Screenshot thay raw export                             | Mất structured evidence                 |
| Hash match = source authentic                          | Hash chỉ chứng minh byte consistency    |
| Analyzer gắn High = tự động High business risk         | Cần exposure/context                    |
| Unused firewall rule = delete ngay                     | Cần ownership/change review             |
| Deny = firewall lỗi                                    | Có thể expected policy                  |
| Allow = application healthy                            | Không chứng minh L7 health              |
| Tool finding = vulnerability                           | Có thể config/IAM/incident              |
| L1 phát hiện là L1 phải sửa                            | Owner và authority có thể khác          |

---

# PHẦN XIV — MENTAL MODEL TROUBLESHOOTING CHUẨN DAY 10

Bạn nên thuộc flow sau:

```text
                   SEARCH HAS NO RESULT
                           │
                           ▼
                 Is query/filter correct?
                      │           │
                     No          Yes
                      │           │
                 Correct query    ▼
                           Does source generate event?
                                │       │
                               No      Yes
                                │       │
                         Source issue    ▼
                                  Is local event present?
                                       │       │
                                      No      Yes
                                       │       │
                                  Audit/source ▼
                                            Did forwarder
                                            receive/read it?
                                               │      │
                                              No     Yes
                                               │      │
                                         Agent/source ▼
                                             Did collector
                                             receive it?
                                               │      │
                                              No     Yes
                                               │      │
                                          Network/    ▼
                                          forwarding Parser correct?
                                                     │      │
                                                    No     Yes
                                                     │      │
                                                Parsing     ▼
                                                        Indexed/searchable?
                                                          │       │
                                                         No      Yes
                                                          │       │
                                                      Platform    Search/
                                                      pipeline    retention/
                                                                  export
```

Nếu bạn thực sự hiểu flow này, bạn đã nắm khoảng một nửa giá trị operational của Day 10.

---

# PHẦN XV — THEO SYLLABUS VS KIẾN THỨC BỔ SUNG

## Theo syllabus — bắt buộc phải thành thạo về mặt lý thuyết

Phạm vi bắt buộc là:

```text
Infrastructure monitoring
Centralized log management

Windows Event Logs
Linux syslog/journald
Nutanix Prism events
HPE iLO logs
FortiGate logs
BIG-IP logs
Cisco syslog
AWS CloudWatch/CloudTrail

Timestamp/timezone normalization
NTP awareness
Event severity
Log-source identification
Collection agents
Syslog forwarding
Log parsing
Event correlation
Log retention
Log access control
Audit data protection

EventLog Analyzer:
- architecture
- onboarding
- Windows/Linux/network collection
- search/filter
- security investigation
- alerting
- reporting

Firewall Analyzer:
- architecture
- ingestion
- traffic analysis
- rule usage
- configuration changes
- policy anomaly awareness
- security audit
- compliance reporting
- export verification

Security findings:
- vulnerability fundamentals
- exposure
- severity vs business risk
- false positives
- remediation ownership
- L1 escalation

Analyzer operations:
- onboarding validation
- service/collector health
- ingestion delay
- retention/capacity
- alert delivery
- report scheduling
- silent source
- parsing failure
- collection gap
```

Đây là toàn bộ Concept/Lecture + additive Concept/Lecture của Day 10. Infrastructure_Operations_14_Da…

---

# PHẦN XVI — MỐI LIÊN HỆ VỚI CÁC DAY KHÁC

Day 10 không đứng riêng lẻ.

```text
Day 2
Linux logs
        ↓
Day 3
Windows Event Logs
        ↓
Day 4–5
AWS CloudWatch/CloudTrail
        ↓
Day 6–7
HPE/Nutanix events
        ↓
Day 8
Cisco syslog
        ↓
Day 9
FortiGate/BIG-IP logs
        ↓
DAY 10
Centralize + Correlate + Investigate
        ↓
Day 11
ServiceNow / privileged access evidence
        ↓
Day 13
IAM review + vulnerability decisions
        ↓
Day 14
Integrated incident investigation
```

Đây là lý do Day 10 rất quan trọng: nó lấy telemetry đã gặp ở Days 1–9 và biến chúng thành **cross-system operational evidence**.

Syllabus traceability matrix cũng xác định EventLog Analyzer và Firewall Analyzer là kỹ năng riêng của Day 10, đồng thời Day 10 đóng góp trực tiếp cho audit extraction và vulnerability classification của các Day sau. Infrastructure_Operations_14_Da…

---

# PHẦN XVII — NĂNG LỰC BẠN PHẢI CÓ SAU MODULE LÝ THUYẾT

Sau khi thực sự hiểu phần Lý thuyết Day 10, bạn phải giải thích được một cách độc lập:

```text
1. Monitoring khác logging như thế nào.

2. Event, alert, finding và incident khác nhau như thế nào.

3. Tại sao enterprise cần centralized log management.

4. Một log đi từ source tới search UI qua những layer nào.

5. Windows/Linux/Nutanix/HPE/FortiGate/BIG-IP/Cisco/AWS
   đóng vai trò gì trong investigation.

6. Tại sao timestamp, timezone và NTP cực kỳ quan trọng.

7. Event time khác ingestion time thế nào.

8. Agent-based và agentless collection khác nhau ra sao.

9. Syslog forwarding hoạt động theo mental model nào.

10. Parsing và normalization dùng để làm gì.

11. Vì sao event tồn tại nhưng search vẫn có thể không thấy.

12. Event correlation khác root-cause proof thế nào.

13. Retention và collection gap ảnh hưởng investigation ra sao.

14. Tại sao log access phải dùng least privilege.

15. EventLog Analyzer được dùng để giải quyết vấn đề gì.

16. Device onboarding phải verify bằng known event thế nào.

17. Search/filter investigation nên thực hiện như thế nào.

18. Alert rule phải verify end-to-end thế nào.

19. Firewall Analyzer khác generic EventLog Analyzer ở đâu.

20. Traffic analysis phải đọc source/destination/action/policy thế nào.

21. Rule usage và policy anomaly có ý nghĩa gì.

22. Configuration change tracking hỗ trợ incident thế nào.

23. Vulnerability khác configuration weakness thế nào.

24. IAM exception khác vulnerability thế nào.

25. Technical severity khác business risk thế nào.

26. Exposure ảnh hưởng risk thế nào.

27. False positive phải được chứng minh thế nào.

28. Remediation owner khác detection owner thế nào.

29. L1/L2 được phép xử lý tới đâu.

30. Khi nào bắt buộc escalate.

31. No matching event khác disconnected source thế nào.

32. Parsing failure khác collection failure thế nào.

33. Collection gap phải được document thế nào.

34. Raw evidence khác derived evidence thế nào.

35. Hash chứng minh và không chứng minh điều gì.

36. Tại sao record count, pagination và export limit quan trọng.

37. Cách xây một cross-system incident timeline.

38. Cách đưa ra conclusion mà không vượt quá evidence.
```

Nếu một trong các câu trên chưa trả lời được rõ ràng, phần Lý thuyết Day 10 vẫn còn gap.

---

# Day 10 — Theory Cheat Sheet

```text
CORE PIPELINE
Source
→ Collection
→ Transport
→ Collector
→ Parser
→ Normalization
→ Index
→ Search
→ Alert/Report
→ Evidence

CORE TIME FIELDS
Event time
Ingestion time
Timezone
NTP / clock drift

CORE SOURCES
Windows Event Logs
Linux syslog/journald
Nutanix Prism
HPE iLO
FortiGate
BIG-IP
Cisco
AWS CloudWatch
AWS CloudTrail

EVENTLOG ANALYZER
Onboard
→ Generate known event
→ Confirm collection
→ Confirm parsing
→ Search
→ Alert/report
→ Verify delivery
→ Check health

FIREWALL ANALYZER
Firewall logs
→ Traffic analysis
→ Allow/Deny
→ Policy usage
→ Config changes
→ Policy anomaly
→ Audit/report

NO RESULT?
Query?
Source event?
Agent/forwarder?
Network?
Collector?
Parser?
Index?
Retention?
Export?

FINDING TYPES
Vulnerability
Configuration weakness
IAM exception
Operational incident

RISK THINKING
Severity
≠ Business risk

Risk depends on:
Exposure
Asset criticality
Business impact
Existing controls

EVIDENCE
Preserve raw
Record filters
Record timezone
Record count
Record source
Record extraction time
Hash raw file
Document gaps
Never overwrite originals

ESCALATION
Evidence
+ Scope
+ Timeline
+ Finding
+ Risk
+ Actions already performed
+ Correct owner
```

Điểm cốt lõi nhất của **Module Lý thuyết Day 10** có thể cô đọng thành một nguyên tắc:

> **Không bao giờ suy luận “không có sự kiện” chỉ từ việc search không có kết quả; trước tiên phải chứng minh source, collection, transport, parsing, indexing, retention và query đều đáng tin cậy. Sau đó mới dùng log để correlation, classification và decision.**

Đó chính là tư duy Infrastructure Operations mà Day 10 muốn xây dựng: **không chỉ biết xem log, mà biết đánh giá độ tin cậy của log, xây timeline có kiểm soát, phân loại finding đúng, bảo toàn evidence và biết giới hạn authority của mình.** Infrastructure_Operations_14_Da…
