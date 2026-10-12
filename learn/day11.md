# Day 11 — AWS Monitoring, Audit, Cost Control & Operations Consolidation

## 1. Syllabus Alignment

Theo **MASTER SYLLABUS**, Day 11 là ngày cuối của phần **AWS Cloud Fundamentals**. Phạm vi bắt buộc hôm nay là:

> **Concept/Lecture:** CloudWatch metrics/logs/alarms, CloudTrail awareness, budgets, public IPv4 cost, stop/terminate controls, and AWS consolidation.  
> **Assignment/Lab:** Configure an alarm and budget control, review cost drivers, stop unused resources, and complete an AWS operations checklist. 00_MASTER_SYLLABUS_Java_Fresher…

Như vậy, sau Day 11 bạn không chỉ phải biết “EC2 chạy được”, mà phải bắt đầu tư duy giống một **L1 Operations / Junior Cloud Administrator**:

**build → observe → alert → investigate → control cost → stop safely → preserve evidence → escalate when necessary.**

Đây cũng là buổi tổng hợp toàn bộ Day 7–11 trước khi sang **Java Application Configuration** từ Day 12. 00_MASTER_SYLLABUS_Java_Fresher…

Trong bài này tôi sẽ phân biệt rõ:

| Phạm vi                                          | Nội dung                                                                                                                                                                                                                          |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Mandatory — đúng syllabus**                    | CloudWatch metrics, logs, alarms; CloudTrail awareness; AWS Budgets; public IPv4 cost; EC2 stop/terminate controls; AWS consolidation; alarm lab; budget control; cost-driver review; stop unused resources; operations checklist |
| **Supplementary / Beyond the explicit syllabus** | CloudWatch Agent, SNS notification mechanics, log retention strategy, Public IP Insights, alarm failure injection, termination dry-run, operational evidence pack                                                                 |

---

# 2. Mục tiêu Day 11

Kết thúc Day 11, bạn cần đủ khả năng giải thích và thực hiện được các câu hỏi kiểu fresher/junior administrator sau:

| Câu hỏi vận hành                                        | Công cụ/chủ đề                                                     |
| ------------------------------------------------------- | ------------------------------------------------------------------ |
| EC2 CPU đang cao hay thấp?                              | CloudWatch Metrics                                                 |
| Instance có failed status check không?                  | EC2 Status Checks + CloudWatch                                     |
| Có bao nhiêu request/error/log event?                   | CloudWatch Logs                                                    |
| CPU > 70% liên tục thì làm sao cảnh báo?                | CloudWatch Alarm                                                   |
| Gửi cảnh báo đi đâu?                                    | SNS / alarm action                                                 |
| Ai vừa Stop instance?                                   | CloudTrail                                                         |
| Ai sửa Security Group?                                  | CloudTrail                                                         |
| Tháng này tài khoản sắp vượt mức chi phí cho phép chưa? | AWS Budgets                                                        |
| Budget có chặn chi phí ngay lập tức không?              | Không nên hiểu như hard real-time cap                              |
| Tại sao instance stop rồi vẫn có bill?                  | EBS, Elastic IP/public IPv4 và tài nguyên khác vẫn có thể tính phí |
| Stop khác Terminate thế nào?                            | EC2 lifecycle                                                      |
| Terminate có phục hồi instance được không?              | Không                                                              |
| Làm sao biết volume nào sẽ bị xóa khi terminate?        | `DeleteOnTermination`                                              |
| AWS lab nào còn chạy và phát sinh tiền?                 | Cost review + resource inventory                                   |
| Có bằng chứng ai đã thực hiện thay đổi không?           | CloudTrail + command log                                           |

---

# 3. Prerequisites và môi trường lab

Bài hôm nay giả định bạn đã hoàn thành cơ bản Day 7–10:

| Day    | Kiến thức cần dùng lại                                 |
| ------ | ------------------------------------------------------ |
| Day 7  | IAM, MFA, AWS CLI profile, tagging                     |
| Day 8  | VPC, subnet, route table, IGW, Security Group          |
| Day 9  | EC2, AMI, key pair, user data, SSH                     |
| Day 10 | EBS, mount, snapshot, restore                          |
| Day 11 | Quan sát + cảnh báo + audit + cost + lifecycle control |

Lab dưới đây giả định:

```text
AWS CLI: v2
EC2: EBS-backed Linux instance
Environment: LAB / NON-PRODUCTION
Authentication: IAM user/role đã được phê duyệt
Region: dùng Region hiện tại của lab, không hard-code nếu không cần
```

Không gửi cho tôi Access Key, Secret Access Key, password, private SSH key hoặc token.

---

# 4. Mental Model quan trọng nhất của Day 11

Hãy nhớ sơ đồ sau:

```text
                    AWS RESOURCE
                         |
          +--------------+--------------+
          |                             |
          v                             v
   Performance/Health                API activity
          |                             |
          v                             v
     CloudWatch                    CloudTrail
          |                             |
   +------+-------+            Who did what?
   |              |            When?
 Metrics         Logs           From where?
   |              |            On which resource?
   +------+-------+
          |
        Alarm
          |
          v
   SNS / automated action
          |
          v
        Operator


AWS usage/resources
       |
       v
Billing / Cost data
       |
       v
Cost Explorer / Bills
       |
       v
AWS Budgets
       |
       v
Alert / Budget Action
```

Nói ngắn gọn:

**CloudWatch = health/behavior.**

**CloudTrail = activity/audit.**

**Budgets = financial guardrail.**

**Stop/Terminate = resource lifecycle control.**

Đây là bốn khái niệm khác nhau, không được trộn lẫn.

---

# 5. Phần I — Amazon CloudWatch

## 5.1 CloudWatch là gì?

Amazon CloudWatch là hệ thống observability/monitoring của AWS.

Một số nhiệm vụ quan trọng:

```text
collect metrics
collect logs
visualize
query
create alarms
trigger notifications/actions
```

Đối với EC2, CloudWatch nhận một số infrastructure metrics tự động. EC2 mặc định gửi metric với chu kỳ **5 phút** khi sử dụng Basic Monitoring; bật Detailed Monitoring sẽ cung cấp dữ liệu theo **1 phút**. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-cloudwatch.html?utm_source=chatgpt.com)

---

# 6. CloudWatch Metrics

## 6.1 Metric là gì?

Metric là một **time series**:

```text
timestamp -> numeric value
```

Ví dụ:

```text
10:00 CPUUtilization = 12%
10:05 CPUUtilization = 18%
10:10 CPUUtilization = 75%
10:15 CPUUtilization = 89%
```

CloudWatch định nghĩa metric thông qua:

```text
Namespace
Metric name
Dimensions
Timestamp
Value
Unit
```

Một metric được xác định duy nhất bởi metric name + namespace + dimensions. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html?utm_source=chatgpt.com)

---

## 6.2 Namespace

Namespace dùng để nhóm metrics.

Ví dụ:

```text
AWS/EC2
AWS/EBS
AWS/RDS
AWS/Lambda
CWAgent
```

Đối với EC2:

```text
Namespace: AWS/EC2
MetricName: CPUUtilization
```

CloudWatch Agent mặc định sử dụng namespace:

````text
CWAgent
``` :chatgpt-content-reference{index="4"}


---

## 6.3 Dimensions

Dimension giúp xác định metric thuộc resource nào.

Ví dụ:

```text
Metric:
CPUUtilization

Dimension:
InstanceId=i-0123456789abcdef0
````

Hai EC2 khác nhau có cùng metric `CPUUtilization`, nhưng `InstanceId` khác nhau.

Ví dụ:

```text
CPUUtilization
  InstanceId=i-aaa111

CPUUtilization
  InstanceId=i-bbb222
```

Đó là hai time series khác nhau.

---

# 7. Các EC2 metrics quan trọng

Bạn nên nhận diện tối thiểu:

| Metric                       | Ý nghĩa vận hành                                              |
| ---------------------------- | ------------------------------------------------------------- |
| `CPUUtilization`             | CPU usage                                                     |
| `NetworkIn`                  | bytes nhận vào                                                |
| `NetworkOut`                 | bytes gửi ra                                                  |
| `DiskReadBytes`              | I/O read liên quan instance store trong các trường hợp hỗ trợ |
| `DiskWriteBytes`             | I/O write tương ứng                                           |
| `StatusCheckFailed`          | tổng hợp failed status checks                                 |
| `StatusCheckFailed_Instance` | vấn đề bên trong instance                                     |
| `StatusCheckFailed_System`   | vấn đề infrastructure phía AWS                                |

EC2 status checks kết hợp với CloudWatch là tín hiệu rất quan trọng để phân biệt vấn đề guest OS và vấn đề underlying AWS infrastructure. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/monitoring_ec2.html?utm_source=chatgpt.com)

---

# 8. Một bẫy thi rất quan trọng: CloudWatch mặc định có RAM/Disk Usage không?

Không được trả lời:

> “CloudWatch EC2 mặc định monitor cả RAM và disk usage.”

Đây là câu trả lời sai hoặc ít nhất là rất thiếu chính xác.

Metric EC2 cơ bản được AWS nhìn thấy từ lớp hypervisor/infrastructure.

Nhưng những thứ như:

```text
Memory utilization inside Linux
Filesystem used %
Swap
process metrics
/application logs
```

là thông tin **inside guest OS**.

Muốn thu thập đầy đủ các metric này, một phương án chuẩn là cài:

```text
Amazon CloudWatch Agent
```

CloudWatch Agent có thể thu thập memory, disk, logs và các system-level metrics khác. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html?utm_source=chatgpt.com)

Mental model:

```text
AWS sees:
EC2 infrastructure
       |
       +--> CPU/network/status metrics

Inside Linux:
free
df
processes
logs
       |
       +--> CloudWatch Agent
                 |
                 +--> CloudWatch
```

---

# 9. Statistics

Một metric có nhiều datapoints.

CloudWatch cần aggregate chúng.

Các statistics phổ biến:

```text
Average
Minimum
Maximum
Sum
SampleCount
```

Ví dụ:

Trong 5 phút CPU có:

```text
10%
20%
80%
20%
20%
```

`Average` và `Maximum` trả lời hai câu hỏi khác nhau.

Average:

```text
CPU trung bình trong period?
```

Maximum:

```text
CPU đã từng peak cao nhất bao nhiêu?
```

Đây là lý do chọn đúng statistic cực kỳ quan trọng khi tạo alarm.

---

# 10. Period

`Period` là khoảng thời gian CloudWatch dùng để aggregate metric.

Ví dụ:

```text
Period = 300 seconds
```

nghĩa là:

```text
5 minutes
```

CloudWatch hỗ trợ nhiều period khác nhau; high-resolution metrics có thể dùng sub-minute periods, nhưng bạn không nên đặt period ngắn hơn khả năng thực tế của source metric rồi kỳ vọng có thêm dữ liệu. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html?utm_source=chatgpt.com)

Với EC2 Basic Monitoring:

```text
data granularity ≈ 5 minutes
```

Do đó cho lab Day 11, sử dụng:

```text
period = 300
```

là dễ hiểu nhất.

---

# 11. CloudWatch Alarm

## 11.1 Alarm làm gì?

Alarm đánh giá metric và so sánh với điều kiện.

Ví dụ:

```text
CPUUtilization > 70%
for 2 consecutive periods
```

Có thể hình dung:

```text
CPU metric
    |
    v
Average / 5 minutes
    |
    v
threshold > 70
    |
    v
CloudWatch Alarm
    |
    v
ALARM
    |
    +--> SNS
    +--> EC2 action
    +--> other supported action
```

CloudWatch có tài liệu chính thức minh họa alarm CPU EC2 và SNS notification theo đúng mô hình này. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/US_AlarmAtThresholdEC2.html?utm_source=chatgpt.com)

---

# 12. Ba state cơ bản của Alarm

Metric alarm có ba state rất quan trọng:

| State               | Ý nghĩa                                      |
| ------------------- | -------------------------------------------- | ------------------------------------- |
| `OK`                | dữ liệu đang nằm trong điều kiện bình thường |
| `ALARM`             | điều kiện alarm bị vi phạm                   |
| `INSUFFICIENT_DATA` | chưa có đủ dữ liệu để quyết định             | :chatgpt-content-reference{index="9"} |

Không được hiểu:

```text
INSUFFICIENT_DATA = hệ thống OK
```

Cũng không được hiểu:

```text
INSUFFICIENT_DATA = hệ thống lỗi
```

Nó có nghĩa:

> CloudWatch không có đủ evidence để đưa ra kết luận.

---

# 13. Evaluation Period và Datapoints to Alarm

Giả sử:

```text
Period = 5 minutes
EvaluationPeriods = 3
DatapointsToAlarm = 2
Threshold = 70
```

Có thể hiểu:

```text
3 datapoints gần nhất
có ít nhất 2 datapoints breach threshold
=> ALARM
```

Ví dụ:

```text
55
82
91
```

Hai trong ba điểm vượt 70 → alarm có thể chuyển `ALARM`.

Đây thường gọi là:

```text
M out of N
```

Ví dụ:

```text
2 out of 3
3 out of 5
2 out of 2
```

---

# 14. Missing Data

Đây là phần troubleshooting quan trọng.

CloudWatch cho phép xử lý missing datapoints theo các cách như:

```text
breaching
notBreaching
ignore
missing
```

Default là `missing`. AWS đặc biệt khuyến cáo cẩn trọng với alarm EC2 có action như stop/terminate/reboot/recover để tránh hành động không mong muốn khi metric tạm thời bị thiếu. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarms-and-missing-data.html?utm_source=chatgpt.com)

Trong lab:

```bash
--treat-missing-data missing
```

là lựa chọn an toàn, dễ giải thích.

---

# 15. CloudWatch Logs

Metrics và logs không phải một thứ.

Metric:

```text
CPUUtilization = 75
```

Log:

```text
2026-10-12 15:20:01 ERROR database connection timeout
```

---

## 15.1 Log Group

Log Group thường đại diện một application/service hoặc logical grouping.

Ví dụ:

```text
/app/tomcat
/system/messages
/java/payment-service
```

---

## 15.2 Log Stream

Một Log Group chứa nhiều Log Streams.

Ví dụ:

```text
Log Group:
/java/payment-service

Log Streams:
i-012345
i-067890
i-0abcde
```

AWS định nghĩa log stream là sequence log events từ cùng một source; log group là tập hợp streams dùng chung retention/access/monitoring settings. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html?utm_source=chatgpt.com)

---

# 16. Log retention — điểm rất quan trọng về cost

CloudWatch Logs mặc định có thể giữ log **indefinitely / never expire** nếu không cấu hình retention. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/?utm_source=chatgpt.com)

Đây là anti-pattern phổ biến:

```text
Application ghi 5 GB/ngày
        |
        v
CloudWatch Logs
        |
        v
Never Expire
        |
        v
storage cứ tăng
        |
        v
bill tăng
```

Do đó administrator phải hỏi:

```text
Log này cần giữ bao lâu?

7 days?
30 days?
90 days?
1 year?
compliance requirement?
```

Không nên mặc định “giữ mãi cho chắc”.

---

# 17. CloudWatch cost awareness

CloudWatch không phải tất cả đều miễn phí.

Theo pricing hiện hành của AWS, CloudWatch có free-tier allowances cho một số metrics/logs/alarms, nhưng detailed monitoring, custom metrics, log ingestion/storage, query usage, alarms và các feature khác có thể phát sinh phí; giá cụ thể còn phụ thuộc Region và loại sử dụng. [Amazon Web Services, Inc.](https://aws.amazon.com/cloudwatch/pricing/)

Một fresher cần có mindset:

```text
more telemetry
!=
free telemetry
```

Nhưng ngược lại:

```text
disable all monitoring to save money
```

cũng là sai.

Mục tiêu là:

> **đủ observability để vận hành an toàn với retention và granularity hợp lý.**

---

# 18. Phần II — CloudTrail

## 18.1 CloudTrail là gì?

CloudTrail ghi lại activity/API events trong AWS account.

Nó giúp trả lời:

```text
WHO?
did WHAT?
WHEN?
from WHERE?
on WHICH RESOURCE?
with WHAT RESULT?
```

Ví dụ:

```text
Who stopped i-012345678?
Who changed Security Group?
Who created an IAM user?
Who terminated the EC2?
Who deleted the snapshot?
```

CloudTrail ghi lại activity qua AWS Management Console, CLI, SDK và các API interactions. [AWS Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/?utm_source=chatgpt.com)

---

# 19. CloudWatch vs CloudTrail

Đây gần như chắc chắn là kiến thức phải thuộc.

| CloudWatch               | CloudTrail           |
| ------------------------ | -------------------- |
| performance/health       | API/account activity |
| CPU                      | who started EC2      |
| network traffic          | who stopped EC2      |
| logs                     | who changed SG       |
| alarms                   | who deleted resource |
| operational telemetry    | audit/governance     |
| “hệ thống đang thế nào?” | “ai đã làm gì?”      |

Ví dụ:

**Incident:** website không truy cập được.

CloudWatch:

```text
CPU = normal
NetworkOut suddenly = 0
Status checks = passed
```

CloudTrail:

```text
15:02
AuthorizeSecurityGroupIngress / RevokeSecurityGroupIngress
performed by user X
```

Bạn bắt đầu hình thành hypothesis:

> Có recent change ở network/security layer.

Đó chính là evidence-driven troubleshooting.

---

# 20. CloudTrail Event History

CloudTrail Event history hiện cung cấp lịch sử **90 ngày management events** theo Region và tự động có sẵn. Việc xem Event history không bị tính CloudTrail charge; nếu cần lưu ongoing record lâu hơn hoặc thu các event type khác như data events thì cần trail hoặc event data store thích hợp. [AWS Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events.html?utm_source=chatgpt.com)

Ví dụ CLI:

```bash
aws cloudtrail lookup-events \
  --max-items 10
```

Tìm các lần stop EC2:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
  AttributeKey=EventName,AttributeValue=StopInstances \
  --max-items 10
```

AWS CLI hỗ trợ `lookup-events` cho management events trong khoảng 90 ngày theo Region. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/cloudtrail/lookup-events.html?utm_source=chatgpt.com)

---

# 21. Các field nên biết khi đọc CloudTrail event

Bạn không cần thuộc toàn bộ JSON, nhưng phải nhận diện:

```text
eventTime
eventSource
eventName
awsRegion
sourceIPAddress
userIdentity
requestParameters
responseElements
resources / resource references
```

Ví dụ tư duy:

```text
eventName:
StopInstances

userIdentity:
AssumedRole

sourceIPAddress:
x.x.x.x

requestParameters:
instanceId=i-0123456789
```

Bạn có thể kết luận:

> Có API `StopInstances` được gọi bởi identity này vào thời điểm này đối với instance này.

Không được suy luận nhiều hơn evidence đang cho phép.

---

# 22. Management events và Data events

CloudTrail có nhiều event categories.

Ở mức Day 11, quan trọng nhất:

```text
Management events
```

Ví dụ:

```text
Create
Modify
Delete
IAM
EC2 lifecycle
Security Group changes
```

Data events là hoạt động data-plane chi tiết hơn đối với một số services và không mặc định xuất hiện trong Event History giống management events. [AWS Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events.html?utm_source=chatgpt.com)

Đây là mức awareness đủ cho Day 11.

---

# 23. Phần III — AWS Budgets

## 23.1 AWS Budget không phải “ví tiền”

Bạn tạo:

```text
Monthly Budget = $10
```

không có nghĩa rằng:

```text
AWS tự động ngắt mọi thứ chính xác tại $10.00
```

Đây là hiểu sai rất nguy hiểm.

AWS Budgets chủ yếu cho phép:

```text
track cost/usage
compare with threshold
send notifications
optionally perform configured budget actions
```

Budget information phụ thuộc billing data và không phải real-time. AWS nói billing data dùng bởi Budgets được cập nhật ít nhất hàng ngày, và các trải nghiệm mới có thể refresh lên tới vài lần/ngày; có thể có độ trễ giữa lúc phát sinh chi phí và lúc nhận cảnh báo. [AWS Documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-best-practices.html?utm_source=chatgpt.com)

Vì vậy:

```text
Budget = guardrail
Budget != perfect instantaneous hard cap
```

---

# 24. Cost Budget

Ví dụ:

```text
Monthly cost budget
Budget = $10

Alert 1:
50% actual

Alert 2:
80% actual

Alert 3:
100% actual
```

Hoặc:

```text
100% forecasted
```

Forecast alert giúp phát hiện:

> Với tốc độ hiện tại, cuối tháng có khả năng vượt budget.

AWS hỗ trợ alerts dựa trên actual hoặc forecasted cost. Forecast cần đủ historical usage data để hoạt động có ý nghĩa. [AWS Documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-best-practices.html?utm_source=chatgpt.com)

---

# 25. AWS Budgets pricing hiện hành

Tại thời điểm đối chiếu cho Day 11:

AWS cho phép **budget monitoring và notifications miễn phí**. Budget có configured actions có pricing riêng: hai action-enabled budgets đầu tiên mỗi tháng được miễn phí, sau đó các action-enabled budgets bổ sung được tính phí theo pricing hiện hành. Budget Reports cũng có phí riêng. [Amazon Web Services, Inc.](https://aws.amazon.com/aws-cost-management/aws-budgets/pricing/)

Do đó trong lab hôm nay:

> Chúng ta chỉ cần một **cost budget + notification**, không cần tạo nhiều automation/action không cần thiết.

---

# 26. Budget alert ≠ CloudWatch alarm

Hai thứ đều “alert”, nhưng data source khác nhau.

| CloudWatch Alarm                   | AWS Budget              |
| ---------------------------------- | ----------------------- |
| operational metric                 | financial data          |
| CPU > 70                           | cost > $5               |
| status check failed                | forecast exceeds budget |
| near-real-time monitoring use case | billing data delayed    |
| resource health                    | financial guardrail     |

---

# 27. Phần IV — Public IPv4 cost

Đây là nội dung syllabus nhấn mạnh rõ ràng.

AWS hiện tính:

```text
$0.005 / public IPv4 address / hour
```

cho cả public IPv4 đang được sử dụng và idle public IPv4 thuộc phạm vi pricing tương ứng. Billing theo second với minimum 60 seconds cho loại public IPv4 này. [Amazon Web Services, Inc.](https://aws.amazon.com/vpc/pricing/)

Một public IPv4 chạy liên tục 30 ngày tương đương khoảng:

```text
$3.60 / 30-day month
```

chỉ riêng phần IPv4.

Không bao gồm:

```text
EC2 compute
EBS
data transfer
CloudWatch
NAT Gateway
load balancer
...
```

---

# 28. Tại sao AWS tính tiền IPv4?

IPv4 public address là resource hữu hạn.

Administrator phải bắt đầu suy nghĩ:

```text
Resource này có thật sự cần public IPv4 không?

Có thể private-only không?

Có load balancer / NAT / bastion / SSM architecture phù hợp hơn không?

Có Elastic IP idle không?
```

AWS cũng cung cấp Public IP Insights thông qua VPC IPAM để quan sát public IPv4 usage. [Amazon Web Services, Inc.](https://aws.amazon.com/vpc/pricing/)

**Supplementary / Beyond explicit syllabus:** Public IP Insights là công cụ rất hữu ích để audit public IPv4 inventory.

---

# 29. Stop instance có chắc chắn hết phí không?

Không.

Đây là một trong những câu thi/phỏng vấn quan trọng nhất Day 11.

Khi EC2 EBS-backed ở `stopped`:

```text
EC2 instance compute usage
-> không bị charge như running instance

EBS volumes
-> vẫn tồn tại
-> vẫn có storage cost

Elastic IP/public IPv4 resource
-> có thể tiếp tục gây charge tùy loại/allocation

Snapshots
-> vẫn tồn tại
-> có storage cost

NAT Gateway
-> vẫn tồn tại nếu không delete
-> vẫn có thể tiếp tục hourly charge

CloudWatch logs
-> vẫn tồn tại

Other AWS resources
-> vẫn tồn tại
```

AWS xác nhận stopped EC2 không bị tính instance usage, nhưng EBS volume vẫn tồn tại và vẫn có charge. [AWS Documentation](https://docs.aws.amazon.com/us_en/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)

Do đó:

```text
STOP != DELETE ALL COST
```

---

# 30. Phần V — Stop vs Terminate

Đây là nội dung bắt buộc.

| Thuộc tính              | Stop                          | Terminate                                                                    |
| ----------------------- | ----------------------------- | ---------------------------------------------------------------------------- |
| Có start lại được?      | Có                            | Không                                                                        |
| Instance ID giữ nguyên? | Có                            | Không thể reuse instance đã terminate                                        |
| Root EBS                | giữ                           | mặc định root EBS thường xóa                                                 |
| Additional EBS          | giữ                           | phụ thuộc `DeleteOnTermination`; default thông thường non-root được preserve |
| Private IPv4            | giữ                           | mất cùng instance                                                            |
| Auto public IPv4        | thường thay đổi khi start lại | mất                                                                          |
| Elastic IP              | vẫn có thể remain associated  | disassociated khi termination                                                |
| RAM                     | mất                           | mất                                                                          |
| Instance store          | dữ liệu mất khi stop/start    | mất                                                                          |
| Compute charge          | dừng khi stopped              | dừng khi shutting-down/terminated                                            |
| Reversible?             | Có                            | Không                                                                        |

AWS mô tả rõ lifecycle này; với EBS-backed instance, default là root volume bị delete khi terminate còn non-root EBS thường được preserve, nhưng `DeleteOnTermination` của từng volume mới là nguồn sự thật cần kiểm tra. [AWS Documentation](https://docs.aws.amazon.com/us_en/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)

---

# 31. STOP

Flow:

```text
running
   |
   v
stopping
   |
   v
stopped
```

Sau đó:

```text
stopped
   |
   v
pending
   |
   v
running
```

Đối với EBS-backed EC2, stop/start được hỗ trợ; instance store-backed root không hỗ trợ stop theo cách tương tự. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_StopInstances.html?utm_source=chatgpt.com)

---

# 32. TERMINATE

Flow:

```text
running
   |
   v
shutting-down
   |
   v
terminated
```

Terminate là **permanent and irreversible**.

AWS cảnh báo rõ rằng terminated instance không thể reconnect hoặc recover; các EBS volumes có `DeleteOnTermination=true` cũng bị xóa. [AWS Documentation](https://docs.aws.amazon.com/us_en/AWSEC2/latest/UserGuide/terminating-instances.html)

Do đó production procedure phải là:

```text
inspect
  ↓
confirm owner
  ↓
confirm purpose
  ↓
backup/snapshot where required
  ↓
check DeleteOnTermination
  ↓
approval
  ↓
terminate
  ↓
validate cleanup
  ↓
preserve evidence
```

Không phải:

```text
"Chắc không dùng nữa"
        ↓
Terminate
```

---

# 33. Termination Protection

EC2 hỗ trợ termination protection.

Nếu bật:

```text
DisableApiTermination = true
```

instance không thể bị terminate bằng thông thường qua console/CLI/API cho đến khi protection được disable. [AWS Documentation](https://docs.aws.amazon.com/us_en/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)

Đây là control tốt đối với resource quan trọng.

Nhưng nhớ:

```text
termination protection
!=
backup
```

và:

```text
termination protection
!=
IAM authorization
```

Nó là thêm một guardrail.

---

# 34. Phần VI — AWS consolidation Day 7–11

Bây giờ ghép toàn bộ AWS block:

```text
IAM
 |
 | "Who can access?"
 v
VPC
 |
 | "Where can traffic flow?"
 v
EC2
 |
 | "Where does workload run?"
 v
EBS
 |
 | "Where is persistent block data?"
 v
CloudWatch
 |
 | "Is it healthy?"
 v
CloudTrail
 |
 | "Who changed it?"
 v
Budgets / Billing
 |
 | "How much does it cost?"
 v
Lifecycle control
 |
 | "Should it keep running?"
 v
Operational evidence
```

Một admin junior giỏi không nhìn từng service riêng lẻ.

Họ nhìn **system lifecycle**.

---

# 35. GUIDED LAB — Day 11 Mandatory Assignment

Bây giờ đến phần quan trọng nhất.

## Mục tiêu

Bạn sẽ hoàn thành đúng Assignment/Lab:

```text
Configure an alarm
Configure budget control
Review cost drivers
Stop unused resource
Complete AWS operations checklist
```

00_MASTER_SYLLABUS_Java_Fresher…

---

# 36. Safety checkpoint trước lab

**Không terminate resource quan trọng trong bài hôm nay.**

Đối với mọi resource bạn định stop:

```text
LAB/non-production?
owner known?
purpose known?
data persistent?
dependency known?
backup required?
public IP dependency?
restart procedure known?
```

Nếu chưa biết câu trả lời:

> Không stop production blindly.

---

# 37. Lab Step 0 — Verify identity và Region

```bash
aws --version
```

Expected invariant:

```text
aws-cli/2.x
```

Xác định IAM identity:

```bash
aws sts get-caller-identity
```

Ví dụ output:

```json
{
  "UserId": "...",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/lab-user"
}
```

**Không copy account information thực của bạn vào chat nếu không cần.**

Xem Region:

```bash
aws configure get region
```

Nếu dùng shell variable:

```bash
export AWS_REGION="$(aws configure get region)"
```

Kiểm tra:

```bash
echo "$AWS_REGION"
```

---

# 38. Lab Step 1 — Inventory EC2

```bash
aws ec2 describe-instances \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,Type:InstanceType,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress}' \
  --output table
```

Ví dụ output:

```text
-----------------------------------------------------------
|                    DescribeInstances                    |
+---------------------+----------+-----------+-------------+
| ID                  | State    | Type      | PublicIP    |
+---------------------+----------+-----------+-------------+
| i-0123456789abcdef0 | running  | t3.micro  | 1.2.3.4     |
+---------------------+----------+-----------+-------------+
```

**Expected invariant:** bạn xác định được đúng instance lab.

Set:

```bash
export INSTANCE_ID="i-REPLACE_ME"
```

Không copy nguyên placeholder.

Kiểm tra:

```bash
echo "$INSTANCE_ID"
```

---

# 39. Lab Step 2 — Inspect CloudWatch Metrics bằng Console

Đi:

```text
AWS Console
→ CloudWatch
→ Metrics
→ All metrics
→ EC2
→ Per-Instance Metrics
```

Tìm:

```text
InstanceId = instance lab
Metric = CPUUtilization
```

Quan sát:

```text
Average
Maximum
Period
Time range
```

Sau đó xem thêm:

```text
NetworkIn
NetworkOut
StatusCheckFailed
```

### Evidence cần ghi

Ví dụ:

```text
Instance: i-...
Metric: CPUUtilization
Namespace: AWS/EC2
Statistic: Average
Period: 5 minutes
Observed state: normal
Evidence time: ...
```

Đây là **evidence**, không chỉ là “tôi đã nhìn thấy graph”.

---

# 40. Lab Step 3 — Tạo SNS Topic cho notification

CloudWatch alarm có thể gửi alarm action tới SNS.

Tạo topic:

```bash
export TOPIC_ARN="$(
  aws sns create-topic \
    --name day11-ops-alerts \
    --query 'TopicArn' \
    --output text
)"
```

Kiểm tra:

```bash
echo "$TOPIC_ARN"
```

Expected invariant:

```text
arn:aws:sns:<region>:<account>:day11-ops-alerts
```

---

# 41. Subscribe email

Dùng email của chính bạn tại terminal:

```bash
aws sns subscribe \
  --topic-arn "$TOPIC_ARN" \
  --protocol email \
  --notification-endpoint "YOUR_EMAIL_ADDRESS"
```

Bạn phải confirm subscription qua email.

Không gửi địa chỉ email cá nhân cho tôi để làm bước này.

Kiểm tra:

```bash
aws sns list-subscriptions-by-topic \
  --topic-arn "$TOPIC_ARN" \
  --output table
```

Nếu chưa confirm, ARN subscription có thể chưa ở trạng thái hoàn chỉnh.

---

# 42. Lab Step 4 — Tạo CPU alarm

Ta chọn cấu hình dễ hiểu:

```text
Metric: CPUUtilization
Statistic: Average
Period: 300 s
Threshold: >70%
Evaluation periods: 2
Datapoints to alarm: 2
Missing data: missing
Action: SNS topic
```

Ý nghĩa:

```text
CPU average > 70%
trong 2 period 5 phút liên tiếp
=> ALARM
```

Tạo:

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "day11-high-cpu-$INSTANCE_ID" \
  --alarm-description "Day11 lab - CPU average above 70 percent" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value="$INSTANCE_ID" \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data missing \
  --unit Percent \
  --alarm-actions "$TOPIC_ARN"
```

Cấu hình này bám sát mô hình alarm CPU chính thức của AWS. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/US_AlarmAtThresholdEC2.html?utm_source=chatgpt.com)

---

# 43. Verify alarm

```bash
aws cloudwatch describe-alarms \
  --alarm-names "day11-high-cpu-$INSTANCE_ID" \
  --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Threshold:Threshold,Period:Period,EvaluationPeriods:EvaluationPeriods}' \
  --output table
```

Bạn cần thấy dạng:

```text
Name               day11-high-cpu-i-...
State              OK hoặc INSUFFICIENT_DATA
Threshold          70
Period             300
EvaluationPeriods  2
```

`describe-alarms` là command chuẩn để kiểm tra alarm configuration/state. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/describe-alarms.html?utm_source=chatgpt.com)

---

# 44. Failure injection an toàn — Test notification

### Cảnh báo

Chỉ thực hiện phần dưới nếu alarm này **chỉ có notification action**.

Nếu bạn đã gắn:

```text
Stop
Terminate
Reboot
Recover
```

thì không test state tùy tiện.

Với alarm lab chỉ gửi SNS, có thể test:

```bash
aws cloudwatch set-alarm-state \
  --alarm-name "day11-high-cpu-$INSTANCE_ID" \
  --state-value OK \
  --state-reason "Day11 lab reset"
```

Sau đó:

```bash
aws cloudwatch set-alarm-state \
  --alarm-name "day11-high-cpu-$INSTANCE_ID" \
  --state-value ALARM \
  --state-reason "Day11 lab notification test"
```

AWS cũng sử dụng `set-alarm-state` trong tài liệu để kiểm tra alarm notification. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/US_AlarmAtThresholdEC2.html?utm_source=chatgpt.com)

Kiểm tra email/SNS notification.

Sau một evaluation cycle, CloudWatch sẽ lại đánh giá metric thực tế.

---

# 45. Evidence cho phần alarm

Bạn nên lưu:

```text
alarm name
instance ID
metric
threshold
period
evaluation periods
alarm state
SNS topic
notification received?
timestamp
```

Mastery không phải:

> “Tôi tạo alarm được.”

Mà là:

> “Tôi chứng minh alarm tồn tại, cấu hình đúng, chuyển state và notification path hoạt động.”

---

# 46. Lab Step 5 — Create AWS Budget

Phần này nên làm qua Console để bạn quen với Billing UI.

Đi:

```text
AWS Console
→ Billing and Cost Management
→ Budgets
→ Create budget
```

Chọn:

```text
Cost budget
```

Đặt tên:

```text
day11-monthly-lab-budget
```

Period:

```text
Monthly
```

Budget amount nên là **giới hạn được chấp thuận cho lab account của bạn**.

Ví dụ học tập:

```text
$5
```

nhưng đừng mặc định `$5` nếu môi trường công ty có chính sách khác.

---

# 47. Budget thresholds đề xuất cho lab

Ví dụ:

| Threshold | Loại   |
| --------: | ------ |
|       50% | Actual |
|       80% | Actual |
|      100% | Actual |

Nếu account đủ history, có thể thêm:

```text
100% Forecasted
```

Budget notification có thể gửi qua email/SNS; email recipients hiện cần hoàn tất verification theo workflow AWS yêu cầu. [AWS Documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-email-recipients.html?utm_source=chatgpt.com)

---

# 48. Verify Budget

Sau khi tạo, kiểm tra:

```text
Budget name
Budget amount
Current actual
Forecast
Alert thresholds
Recipients
Period
```

Quan trọng:

Không kỳ vọng:

```text
create budget
→ billing instantly complete
```

AWS Budget data có delay. [AWS Documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-best-practices.html?utm_source=chatgpt.com)

Evidence:

```text
Budget: day11-monthly-lab-budget
Period: Monthly
Amount: ...
Alert threshold: ...
Notification target: verified
Status: active
```

---

# 49. Lab Step 6 — Review Cost Drivers

Đây là mandatory requirement.

Mở:

```text
Billing and Cost Management
→ Bills
```

Nếu Cost Explorer đã khả dụng:

```text
Billing and Cost Management
→ Cost Explorer
```

Bạn cần tìm:

```text
Service
Usage type
Region
Resource where available
Daily/monthly trend
```

Cost Explorer có thể cần thời gian để chuẩn bị dữ liệu khi mới được dùng; cost data cũng không nên được coi là real-time. [AWS Documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html?utm_source=chatgpt.com)

---

# 50. Cost-driver checklist cho lab Day 7–11

Kiểm tra tối thiểu:

| Resource      | Cost question                                              |
| ------------- | ---------------------------------------------------------- |
| EC2           | Có instance `running` không cần thiết?                     |
| EBS           | Có volume còn tồn tại sau khi instance stop không?         |
| Snapshot      | Có snapshot lab cũ không?                                  |
| Public IPv4   | Instance/EIP có public IPv4 không?                         |
| Elastic IP    | Có IP allocated nhưng không cần không?                     |
| NAT Gateway   | Lab có tạo NAT Gateway không?                              |
| Data transfer | Có traffic đáng kể không?                                  |
| CloudWatch    | Log ingestion/retention/custom metrics có phát sinh không? |
| Budget        | Guardrail có được cấu hình chưa?                           |

---

# 51. CLI resource inventory

### EBS volumes

```bash
aws ec2 describe-volumes \
  --query 'Volumes[].{VolumeId:VolumeId,State:State,SizeGiB:Size,Type:VolumeType}' \
  --output table
```

Tìm:

```text
available
```

Volume `available` nghĩa là không attach, **không có nghĩa là miễn phí**.

---

# 52. Snapshot inventory

```bash
aws ec2 describe-snapshots \
  --owner-ids self \
  --query 'Snapshots[].{SnapshotId:SnapshotId,VolumeId:VolumeId,State:State,StartTime:StartTime}' \
  --output table
```

Đừng xóa snapshot chỉ vì nhìn thấy nó.

Phải biết:

```text
owner?
retention requirement?
recovery dependency?
```

---

# 53. Elastic IP inventory

```bash
aws ec2 describe-addresses \
  --query 'Addresses[].{PublicIp:PublicIp,AllocationId:AllocationId,InstanceId:InstanceId,AssociationId:AssociationId}' \
  --output table
```

Nếu thấy:

```text
PublicIp exists
InstanceId = null
```

đó là một resource đáng review.

Public IPv4 pricing hiện tại không còn mô hình “chỉ idle Elastic IP mới đáng lo”; AWS tính hourly charge đối với public IPv4 thuộc phạm vi pricing cả khi in-use và idle. [Amazon Web Services, Inc.](https://aws.amazon.com/vpc/pricing/)

---

# 54. NAT Gateway inventory

Nếu Day 8 của bạn không dùng NAT Gateway thì có thể không có gì.

Kiểm tra:

```bash
aws ec2 describe-nat-gateways \
  --query 'NatGateways[].{ID:NatGatewayId,State:State,VpcId:VpcId,SubnetId:SubnetId}' \
  --output table
```

NAT Gateway là ví dụ rất điển hình của resource mà:

```text
"không có traffic"
```

không đồng nghĩa:

```text
"không có phí"
```

AWS VPC pricing có NAT Gateway hourly charge cùng data processing charge. [Amazon Web Services, Inc.](https://aws.amazon.com/vpc/pricing/)

---

# 55. Lab Step 7 — Stop unused EC2 safely

### Trước khi stop

Xác nhận:

```text
Đúng instance?
LAB?
Không phải production?
Không chạy DB quan trọng?
Data đã persistent?
Có EBS?
Có người khác dùng không?
Public IP thay đổi có ảnh hưởng không?
```

Sau đó:

```bash
aws ec2 stop-instances \
  --instance-ids "$INSTANCE_ID"
```

AWS sẽ chuyển:

```text
running
→ stopping
→ stopped
```

Chờ đến khi hoàn thành:

```bash
aws ec2 wait instance-stopped \
  --instance-ids "$INSTANCE_ID"
```

Verify:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].State.Name' \
  --output text
```

Expected invariant:

```text
stopped
```

---

# 56. Sau stop phải kiểm tra gì?

Không được kết thúc ở:

```text
State = stopped
```

Phải kiểm tra tiếp:

```text
EBS volume still exists?
Elastic IP?
public IP behavior?
snapshot?
CloudWatch alarm?
business/service impact?
```

Remember:

> Stop giải quyết compute runtime, không tự động cleanup toàn bộ AWS architecture.

AWS xác nhận stopped instance vẫn giữ EBS volumes và bạn tiếp tục bị tính storage usage của volumes đó. [AWS Documentation](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_StopInstances.html?utm_source=chatgpt.com)

---

# 57. Public IP sau stop/start

Một EC2 dùng auto-assigned public IPv4 thường nhận public IPv4 mới sau stop/start.

Private IPv4 được preserve.

Elastic IP có thể remain associated.

AWS lifecycle documentation mô tả chính xác hành vi này. [AWS Documentation](https://docs.aws.amazon.com/us_en/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)

Điều này quan trọng cho troubleshooting.

Ví dụ:

```text
Client allowlist:
54.1.2.3

EC2 stopped/started

New public IPv4:
18.2.3.4
```

Triệu chứng:

```text
App hoạt động
Security Group đúng
Service listening
nhưng client không connect
```

Root cause có thể:

```text
external allowlist vẫn dùng old public IP
```

---

# 58. Lab Step 8 — Verify CloudTrail evidence

Sau khi stop:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
  AttributeKey=EventName,AttributeValue=StopInstances \
  --max-items 10
```

Tìm event tương ứng.

Bạn cần xác nhận:

```text
EventName
EventTime
Username / identity
Resource
Region
```

Đây là evidence audit.

CloudTrail Event History chứa management activity như create/modify/delete/start/stop trong phạm vi thời gian lưu mặc định. [AWS Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events.html?utm_source=chatgpt.com)

---

# 59. Supplementary Lab — termination safety inspection

**Không terminate instance Day 11 nếu bạn còn cần nó cho Day 12.**

Bạn sẽ cần EC2 để deploy Java/Tomcat.

Do đó hôm nay chỉ học control.

Kiểm tra volume behavior:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].BlockDeviceMappings[].{Device:DeviceName,VolumeId:Ebs.VolumeId,DeleteOnTermination:Ebs.DeleteOnTermination}' \
  --output table
```

Ví dụ:

```text
-----------------------------------------------
|              BlockDeviceMappings            |
+----------+----------------------+------------+
| Device   | VolumeId             | Delete     |
+----------+----------------------+------------+
| /dev/xvda| vol-0123456789       | True       |
+----------+----------------------+------------+
```

Interpretation:

```text
DeleteOnTermination = True
```

nghĩa là volume đó được cấu hình để xóa cùng instance khi terminate.

---

# 60. Check termination protection

```bash
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute disableApiTermination
```

Có thể thấy:

```json
{
  "DisableApiTermination": {
    "Value": false
  }
}
```

Nếu production:

```text
false
```

không có nghĩa “được phép terminate”.

Nó chỉ nói:

> API termination protection hiện không chặn thao tác.

Authorization vẫn phụ thuộc:

```text
change approval
ownership
runbook
IAM
business impact
backup
```

---

# 61. Dry-run terminate — chỉ để học

Một cách an toàn hơn để kiểm tra quyền:

```bash
aws ec2 terminate-instances \
  --instance-ids "$INSTANCE_ID" \
  --dry-run
```

Nếu có permission, CLI thường trả một `DryRunOperation` cho biết request sẽ được phép nếu bỏ `--dry-run`.

**Tuyệt đối không bỏ `--dry-run` đối với instance bạn cần giữ.**

---

# 62. Supplementary — CloudWatch Logs thực hành với CloudWatch Agent

Phần này mở rộng cho nội dung CloudWatch Logs.

Giả định:

```text
Amazon Linux 2023
EC2 IAM role đã attach CloudWatchAgentServerPolicy
```

AWS hiện hỗ trợ package CloudWatch Agent trực tiếp trên Amazon Linux 2023/Amazon Linux 2. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/manual-installation.html?utm_source=chatgpt.com)

Cài:

```bash
sudo yum install -y amazon-cloudwatch-agent
```

Verify package:

```bash
rpm -qa | grep amazon-cloudwatch-agent
```

Agent config/control command thường nằm tại:

```text
/opt/aws/amazon-cloudwatch-agent/
```

Agent troubleshooting log trên Linux thường tại:

````text
/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
``` :chatgpt-content-reference{index="41"}


Bạn có thể sử dụng wizard:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
````

Chọn thu thập một file log lab như:

```text
/var/log/messages
```

hoặc log phù hợp với distro.

Sau khi config và start, kiểm tra:

```text
CloudWatch
→ Logs
→ Log groups
```

Mục tiêu ở Day 11 không phải thuộc mọi field config, mà phải hiểu flow:

```text
Linux log file
      |
CloudWatch Agent
      |
CloudWatch Logs
      |
Log Group
      |
Log Stream
      |
search/query/alarm
```

---

# 63. Troubleshooting Framework cho Day 11

Giả sử:

> “Không nhận được alarm email.”

Đừng ngay lập tức delete/create lại mọi thứ.

Đi theo evidence-driven process.

```text
Symptom
   ↓
Expected behavior
   ↓
Scope
   ↓
Evidence
   ↓
Hypotheses
   ↓
Safest test
   ↓
Root cause
   ↓
Remediation
   ↓
Validation
```

---

# 64. Scenario A — Alarm không chuyển ALARM

### Symptom

```text
CPU cao
nhưng alarm vẫn OK
```

### Check

```text
Đúng instance ID?
Đúng Region?
Đúng namespace?
Đúng metric?
Statistic?
Period?
Threshold?
EvaluationPeriods?
DatapointsToAlarm?
```

Command:

```bash
aws cloudwatch describe-alarms \
  --alarm-names "day11-high-cpu-$INSTANCE_ID"
```

Hypothesis:

```text
CPU spike chỉ 1 phút
nhưng metric basic monitoring / period 5 minutes
Average vẫn < 70%
```

Đây không phải CloudWatch “hỏng”.

Đó có thể là alarm design sai với objective.

---

# 65. Scenario B — Alarm = INSUFFICIENT_DATA

Check:

```text
Metric có datapoint chưa?
Instance mới start?
Instance stopped?
Wrong Region?
Wrong dimension?
Metric source stopped?
```

Không được fix bằng:

```text
treat everything as breaching
```

một cách mù quáng.

Bạn phải hiểu source data trước.

---

# 66. Scenario C — Alarm ALARM nhưng không có email

Layering:

```text
CloudWatch Alarm
      |
      v
SNS topic
      |
      v
SNS subscription
      |
      v
Email
```

Check từng layer.

CloudWatch:

```bash
aws cloudwatch describe-alarms \
  --alarm-names "day11-high-cpu-$INSTANCE_ID"
```

SNS:

```bash
aws sns list-subscriptions-by-topic \
  --topic-arn "$TOPIC_ARN"
```

Hypotheses:

```text
subscription PendingConfirmation
wrong topic
alarm-actions wrong ARN
email spam filter
alarm never actually transitioned state
```

---

# 67. Scenario D — “Tôi stop EC2 nhưng bill vẫn tăng”

Đây là scenario rất thực tế.

Investigation:

```text
1. EC2 state?
2. EBS?
3. Snapshots?
4. Elastic IP/public IPv4?
5. NAT Gateway?
6. Load Balancer?
7. CloudWatch?
8. Data transfer?
9. Other Regions?
```

Sai:

> “Instance stopped rồi nên AWS bill phải bằng zero.”

Đúng:

> “Instance compute billing có thể đã dừng, nhưng associated/independent resources vẫn có thể phát sinh phí.”

---

# 68. Scenario E — “Không biết ai stop server”

Đi:

```text
CloudTrail
```

Không đi CloudWatch CPU graph để tìm “user”.

Command:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
  AttributeKey=EventName,AttributeValue=StopInstances
```

Sau đó xem:

```text
userIdentity
eventTime
sourceIPAddress
requestParameters
```

L1 có thể collect evidence.

Nếu activity có dấu hiệu unauthorized:

```text
preserve evidence
do not destroy logs
do not rotate everything blindly
escalate security incident process
```

---

# 69. L1-safe remediation vs escalation

| Tình huống                                     | L1 có thể làm                                        |
| ---------------------------------------------- | ---------------------------------------------------- |
| Alarm threshold cấu hình sai trong lab         | chỉnh theo runbook                                   |
| SNS subscription chưa confirm                  | confirm                                              |
| EC2 lab unused                                 | stop theo approved procedure                         |
| Wrong Region khi xem metric                    | chuyển đúng Region                                   |
| CloudWatch Agent stopped                       | inspect log/service rồi restart nếu runbook cho phép |
| Unknown production instance high CPU           | collect evidence, không tùy tiện reboot              |
| Unexpected `TerminateInstances` event          | preserve CloudTrail evidence + escalate              |
| Budget tăng bất thường                         | identify service/resource + escalate owner           |
| Suspected compromised IAM identity             | security escalation                                  |
| Production terminate request không có approval | không thực hiện                                      |

---

# 70. Security & Least Privilege

Day 11 vẫn phải áp dụng Day 7.

Không nên cấp:

```text
AdministratorAccess
```

chỉ vì một operator cần xem alarms.

Phân tách capability:

```text
Read CloudWatch
Manage CloudWatch alarm
Read CloudTrail
View Billing
Manage Budget
Stop EC2
Terminate EC2
```

`ec2:TerminateInstances` có blast radius cao hơn nhiều so với:

```text
cloudwatch:DescribeAlarms
```

Least privilege phải phản ánh điều đó.

---

# 71. Log security

Logs có thể vô tình chứa:

```text
password
access token
session ID
PII
API key
authorization header
database credentials
```

Đừng nghĩ:

> “CloudWatch bảo mật nên log secret thoải mái.”

Không.

Rule:

```text
Don't log secrets.
```

CloudWatch Logs có thêm data-protection capabilities, nhưng prevention tại application/source vẫn quan trọng. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/?utm_source=chatgpt.com)

---

# 72. Cost-control anti-patterns

Một fresher cần tránh:

```text
“Free Tier nên không cần Budget.”
```

Sai.

```text
“Stop instance nghĩa là hết mọi phí.”
```

Sai.

```text
“Public IPv4 của EC2 là free.”
```

Không còn là assumption an toàn theo pricing hiện hành. [Amazon Web Services, Inc.](https://aws.amazon.com/vpc/pricing/)

```text
“CloudWatch giữ log mãi cũng không sao.”
```

Sai về operational/cost discipline.

```text
“Budget $10 sẽ chặn AWS đúng $10.”
```

Sai.

```text
“CloudTrail = monitoring CPU.”
```

Sai.

```text
“CloudWatch = audit user actions.”
```

Sai.

---

# 73. Operations Checklist — Mandatory Day 11

Đây là checklist bạn nên thực sự sử dụng.

| Check                                  | Evidence                      |
| -------------------------------------- | ----------------------------- |
| IAM identity đã xác nhận               | `aws sts get-caller-identity` |
| Region đúng                            | `aws configure get region`    |
| EC2 inventory đã review                | `describe-instances`          |
| Không có instance lab chạy thừa        | state evidence                |
| EBS volumes đã review                  | `describe-volumes`            |
| Snapshots đã review                    | `describe-snapshots`          |
| Public IPv4 / EIP đã review            | `describe-addresses`          |
| NAT Gateway đã review nếu có           | `describe-nat-gateways`       |
| CloudWatch metrics xem được            | screenshot/record             |
| Alarm tồn tại                          | `describe-alarms`             |
| Alarm notification validated           | SNS/email evidence            |
| CloudTrail StopInstances event found   | lookup evidence               |
| Budget tồn tại                         | budget screen/config          |
| Budget notification configured         | alert evidence                |
| Cost drivers reviewed                  | Bills / Cost Explorer         |
| Unused lab EC2 stopped                 | state=`stopped`               |
| Resource cần Day 12 không bị terminate | inventory evidence            |
| Cleanup decision documented            | notes                         |
| Known risks documented                 | operations note               |

---

# 74. Evidence Pack nên có

Một Day 11 evidence package đẹp có thể có cấu trúc:

```text
day11/
├── 01-identity.txt
├── 02-region.txt
├── 03-ec2-inventory.txt
├── 04-cloudwatch-metrics.png
├── 05-alarm-config.txt
├── 06-alarm-notification.png
├── 07-budget-config.png
├── 08-cost-review.md
├── 09-resource-inventory.txt
├── 10-stop-evidence.txt
├── 11-cloudtrail-stop-event.json
└── 12-operations-checklist.md
```

Không lưu credentials trong evidence pack.

---

# 75. Cleanup / Recovery / Rollback

Nếu bạn cần instance cho Day 12, sau khi chứng minh `stop` hoạt động có thể start lại:

```bash
aws ec2 start-instances \
  --instance-ids "$INSTANCE_ID"
```

Chờ:

```bash
aws ec2 wait instance-running \
  --instance-ids "$INSTANCE_ID"
```

Verify:

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].{State:State.Name,PrivateIP:PrivateIpAddress,PublicIP:PublicIpAddress}' \
  --output table
```

Expected invariant:

```text
State = running
```

Nhưng nhớ:

```text
PublicIP có thể đã thay đổi.
```

Sau đó test:

```text
SSH
application access if any
status checks
```

---

# 76. Không terminate EC2 trước Day 12 nếu đó là lab server chính

Theo syllabus, Day 12 sẽ deploy Java application/Tomcat lên Linux environment. 00_MASTER_SYLLABUS_Java_Fresher…

Do đó nếu EC2 hiện tại là server chính của bạn:

```text
STOP for Day11 practice
START afterward
KEEP for Day12
```

Không cleanup quá mức đến mức ngày mai phải build lại mọi thứ không cần thiết.

---

# 77. Exam Focus — Theory

Bạn phải trả lời được các câu sau mà không nhìn tài liệu.

### Câu 1

**CloudWatch khác CloudTrail thế nào?**

Đáp án cốt lõi:

```text
CloudWatch:
metrics/logs/alarms, operational health.

CloudTrail:
API/account activity and auditing.
```

---

### Câu 2

**EC2 Basic Monitoring gửi metric khoảng bao lâu?**

```text
5 minutes
```

Detailed Monitoring:

````text
1 minute
``` :chatgpt-content-reference{index="45"}


---

### Câu 3

**CloudWatch mặc định có memory utilization của Linux không?**

Không nên giả định có.

Dùng CloudWatch Agent hoặc telemetry mechanism phù hợp để thu in-guest metrics.

---

### Câu 4

**Ba alarm states?**

```text
OK
ALARM
INSUFFICIENT_DATA
````

---

### Câu 5

**Budget có phải hard real-time spend cap không?**

Không.

Billing data có delay.

---

### Câu 6

**Stop EC2 thì EBS có mất không?**

Không đối với EBS-backed stop bình thường.

Volume vẫn tồn tại và tiếp tục có storage cost.

---

### Câu 7

**Terminate có start lại được không?**

Không.

---

### Câu 8

**Root EBS khi terminate mặc định thế nào?**

Thông thường:

```text
DeleteOnTermination = true
```

và root volume bị delete.

Phải kiểm tra actual `DeleteOnTermination`.

---

### Câu 9

**Muốn biết ai StopInstances thì xem đâu?**

```text
CloudTrail
```

---

### Câu 10

**Public IPv4 hiện tại có cost không?**

Có. AWS hiện niêm yết:

```text
$0.005/address/hour
```

cho public IPv4 thuộc phạm vi pricing, cả in-use và idle. [Amazon Web Services, Inc.](https://aws.amazon.com/vpc/pricing/)

---

# 78. Exam Focus — Practical

Nếu practical test cho:

> “Check AWS environment, configure monitoring, stop unnecessary cost.”

Trình tự tốt:

```text
1. Verify identity
2. Verify Region
3. Inventory resources
4. Check resource tags/ownership
5. View CloudWatch metrics
6. Create/verify alarm
7. Validate notification
8. Review CloudTrail
9. Review billing/cost drivers
10. Verify budget
11. Safely stop approved unused resource
12. Verify state
13. Check residual chargeable resources
14. Record evidence
15. Handover/escalate exceptions
```

Không bắt đầu bằng:

```text
Terminate everything unused-looking
```

---

# 79. Knowledge Check

Hãy tự trả lời trước khi nhìn đáp án.

**1.** EC2 CPU cao thì đầu tiên nên tìm CloudWatch hay CloudTrail?

**2.** Muốn biết ai đổi Security Group thì dùng gì?

**3.** EC2 stopped còn EBS charge không?

**4.** `INSUFFICIENT_DATA` có nghĩa server healthy không?

**5.** CloudWatch Basic Monitoring EC2 có RAM Linux không?

**6.** Budget $10 có bảo đảm bill không bao giờ vượt $10 không?

**7.** Public IPv4 hiện nay có cost không?

**8.** Terminate và Stop khác nhau quan trọng nhất ở đâu?

**9.** `DeleteOnTermination=true` có ý nghĩa gì?

**10.** Tại sao cần retention cho CloudWatch Logs?

Đáp án ngắn:

```text
1 CloudWatch
2 CloudTrail
3 Có
4 Không
5 Không mặc định
6 Không
7 Có
8 Stop reversible; terminate irreversible
9 Volume sẽ được delete khi instance terminate
10 Kiểm soát retention/compliance/cost
```

---

# 80. Independent Practical Challenge

Sau phần guided lab, hãy tự làm bài sau mà không copy từng command từ phía trên:

> Bạn là L1 operator cuối ca. Có một EC2 lab đang chạy. Nhiệm vụ của bạn là xác định identity/Region, kiểm tra health metrics, chứng minh alarm hoạt động, xác nhận AWS budget, xác định các cost-driving resources, quyết định instance có thể stop hay không, stop nó an toàn nếu được phép, tìm CloudTrail event chứng minh thao tác, rồi viết handover note.

Evidence bắt buộc:

```text
identity
Region
EC2 state before
CloudWatch metric
alarm config
alarm notification
budget config
EBS inventory
public IPv4/EIP review
cost-driver notes
EC2 state after
CloudTrail event
final cleanup decision
```

Mastery criterion:

> Bạn phải chứng minh bằng evidence, không chỉ nói “đã làm xong”.

---

# 81. Troubleshooting Challenge

Scenario:

```text
08:00  Website hoạt động
08:05  Website inaccessible
08:07  EC2 StatusCheckFailed = 0
08:07  CPU = 12%
08:08  Operator thấy EC2 đang stopped
```

Câu hỏi:

> Bạn điều tra layer nào tiếp theo?

Tư duy đúng:

```text
Symptom: app unavailable

EC2:
instance stopped

Question:
Why was it stopped?

Evidence:
CloudTrail StopInstances

Then inspect:
eventTime
identity
source IP
request parameters
change ticket / approval
```

Không cần kiểm tra:

```text
JVM heap
PostgreSQL lock
```

trước khi xử lý sự thật đơn giản rằng instance đang stopped.

Đây gọi là:

> **Find the failing layer before deep-diving lower layers.**

---

# 82. Day 11 Mental Model cần thuộc lòng

Nếu bạn chỉ nhớ một sơ đồ, hãy nhớ:

```text
             AWS Operations

  HEALTH                  ACTIVITY
    |                        |
CloudWatch                CloudTrail
    |                        |
Metrics                  Who?
Logs                     What?
Alarms                   When?
    |                     Where?
    v
Notification

             COST
              |
         Bills / Usage
              |
           Budget
              |
        Cost control

           LIFECYCLE
              |
      Start / Stop / Terminate
              |
        Safe operations
```

---

# 83. Những điểm Day 11 bạn tuyệt đối không được nhầm

| Sai                                    | Đúng                                        |
| -------------------------------------- | ------------------------------------------- |
| CloudWatch = audit                     | CloudTrail = audit/API activity             |
| CloudTrail = CPU monitoring            | CloudWatch = metrics                        |
| Stop = terminate                       | hoàn toàn khác                              |
| Stopped EC2 = $0 toàn bộ               | EBS và tài nguyên khác có thể vẫn charge    |
| Budget = hard spending limit real time | budget data có latency                      |
| Public IPv4 luôn free                  | hiện có hourly pricing                      |
| CloudWatch EC2 mặc định có RAM         | cần in-guest telemetry như CloudWatch Agent |
| `INSUFFICIENT_DATA` = OK               | không đủ evidence                           |
| Terminate có thể undo                  | irreversible                                |
| Alarm tồn tại = monitoring hoạt động   | phải validate metric + state + notification |
| Log giữ mãi tốt hơn                    | phải có retention requirement               |
| Cost review chỉ nhìn EC2               | phải nhìn EBS/IP/NAT/snapshot/logs/etc.     |

---

# 84. Mastery Check cuối Day 11

Day 11 chỉ nên coi là **thực sự hoàn thành** khi bạn tự làm được toàn bộ flow:

```text
IDENTIFY
aws sts get-caller-identity

     ↓

INVENTORY
EC2 / EBS / snapshot / public IP

     ↓

OBSERVE
CloudWatch metrics

     ↓

ALERT
CloudWatch Alarm + SNS

     ↓

VALIDATE
Alarm state + notification

     ↓

AUDIT
CloudTrail

     ↓

CONTROL COST
Budget + billing review

     ↓

SAFE ACTION
Stop approved unused EC2

     ↓

VERIFY
State + remaining resources

     ↓

EVIDENCE
Commands / outputs / timestamps

     ↓

HANDOVER
What changed?
What remains?
What costs?
What risks?
What must be restarted for Day 12?
```

Đó chính xác là sự chuyển đổi từ việc **“biết AWS services”** sang tư duy **“vận hành AWS environment”** mà Day 11 muốn xây dựng.

## Trạng thái sau Day 11

Sau bài hôm nay, phần AWS Fundamentals trong syllabus Day 7–11 đã kết nối thành một hệ thống hoàn chỉnh:

```text
Day 7  IAM / security identity
Day 8  VPC / network flow
Day 9  EC2 / compute provisioning
Day 10 EBS / persistence / recovery
Day 11 monitoring / audit / cost / lifecycle
```

Bước kế tiếp của syllabus là **Day 12 — Java Application Runtime Architecture, JAR/WAR, Tomcat layout, Java 17/25, deployment flow, service account và systemd integration**, với lab deploy Java application lên Tomcat. 00_MASTER_SYLLABUS_Java_Fresher…

Điểm quan trọng trước khi sang Day 12 là **giữ lại EC2 Linux lab hiện tại**, start lại nếu bạn đã stop nó cho bài thực hành, và xác nhận SSH + storage + networking vẫn hoạt động. Khi bạn thực sự chạy các bước Day 11, evidence đáng kiểm tra nhất là output của `describe-alarms`, trạng thái Budget, `describe-volumes`, trạng thái EC2 sau stop/start và CloudTrail `StopInstances` event.
