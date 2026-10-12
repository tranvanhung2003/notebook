# Day 17 — Prometheus, node_exporter, Grafana, Alerting & Monitoring Operations

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 17 thuộc phần **Database & Monitoring Foundations**. Phạm vi bắt buộc hôm nay là:

> **Concept/Lecture:** Prometheus/node_exporter, Grafana dashboard, basic alert rules, service/host/application signals, và alert validation.  
> **Assignment/Lab:** Deploy monitoring, build a basic dashboard, configure an alert, inject a failure, và verify notification evidence. 00_MASTER_SYLLABUS_Java_Fresher…

Như vậy, để coi Day 17 là hoàn thành, bạn không chỉ cần biết “Prometheus là gì” hay dựng được một dashboard đẹp. Bạn phải chứng minh được toàn bộ chuỗi:

```text
system/application
      ↓ metrics
node_exporter / application metrics endpoint
      ↓ scrape
Prometheus
      ↓ query
    PromQL
      ↓
Grafana dashboard

Prometheus rule evaluation
      ↓
Pending → Firing
      ↓
Alertmanager
      ↓
Notification
      ↓
Evidence

Failure Injection
      ↓
Metric changes
      ↓
Alert fires
      ↓
Operator investigates
      ↓
Service restored
      ↓
Alert resolves
```

### Mandatory vs Supplementary

Phần **Mandatory** của syllabus là Prometheus, node_exporter, Grafana dashboard, alert rules, host/service/application signals, failure injection và alert validation.

Tôi sẽ bổ sung **Alertmanager** vì Prometheus alert rule tự nó chỉ xác định alert; Alertmanager là thành phần chuẩn của Prometheus ecosystem để group, deduplicate, route và gửi notification. Prometheus documentation mô tả alerting thành hai phần: Prometheus đánh giá rule và gửi alert sang Alertmanager; Alertmanager xử lý notification. [prometheus.io](https://prometheus.io/docs/alerting/latest/overview/?utm_source=chatgpt.com) Đây là **Supplementary / Beyond the explicit syllabus**, nhưng gần như cần thiết nếu chúng ta muốn thực hiện đúng phần “verify notification evidence”.

---

# 2. Environment assumptions và phiên bản

Lab bên dưới giả định một Linux VM kiểu Ubuntu/Debian với `systemd`, có `sudo`, và bạn có quyền cài phần mềm. Nếu bạn đang dùng RHEL/Rocky/Amazon Linux, package-management commands sẽ khác nhưng kiến trúc và configuration gần như giống nhau.

Tại ngày **12/10/2026**, các release hiện hành có Prometheus 3.15.0, node_exporter 1.12.1, Alertmanager 0.34.1 và Grafana 13.2.3. Với môi trường doanh nghiệp, luôn ưu tiên phiên bản đã được organization phê duyệt thay vì tự động lấy “latest”. [GitHub](https://github.com/prometheus/prometheus/releases?after=v2.16.0&utm_source=chatgpt.com)

| Component     | Vai trò                                    | Port thường dùng trong lab |
| ------------- | ------------------------------------------ | -------------------------: |
| Prometheus    | Scrape, lưu metrics, PromQL, evaluate rule |                     `9090` |
| node_exporter | Export Linux host metrics                  |                     `9100` |
| Grafana       | Visualization/dashboard                    |                     `3000` |
| Alertmanager  | Alert routing/notification                 |                     `9093` |
| Lab webhook   | Nhận notification để làm evidence          |                     `5001` |

node_exporter mặc định listen trên TCP `9100`. Prometheus server mặc định dùng `9090`; Grafana mặc định dùng `3000`. [GitHub](https://github.com/prometheus/node_exporter/blob/master/README.md?utm_source=chatgpt.com)

---

# 3. Objectives + prerequisites

Sau Day 17, bạn cần có khả năng nhìn một server và trả lời được bốn câu hỏi khác nhau:

| Câu hỏi                                   | Công cụ / signal                     |
| ----------------------------------------- | ------------------------------------ |
| Host còn sống và tài nguyên thế nào?      | node_exporter + Prometheus           |
| Metrics target có đang scrape được không? | Prometheus `up`                      |
| Hệ thống đang diễn biến thế nào?          | PromQL + Grafana                     |
| Khi có sự cố, ai/đâu sẽ nhận alert?       | Prometheus alert rule + Alertmanager |

Prerequisite từ các Day trước rất quan trọng: Linux process/systemd, `curl`, `ss`, firewall/networking, filesystem capacity, log inspection, Java/Tomcat deployment và PostgreSQL. Day 17 thực chất là lớp quan sát nằm trên tất cả những thứ bạn đã học.

---

# 4. Mental Model — Monitoring stack hoạt động thế nào?

Prometheus chủ yếu theo **pull model**.

Giả sử:

```text
EC2 / Linux server
├── Tomcat :8080
├── PostgreSQL :5432
└── node_exporter :9100
```

node_exporter đọc Linux/kernel information và expose thành HTTP metrics:

```text
http://server:9100/metrics
```

Prometheus định kỳ request endpoint đó:

```text
Prometheus
   |
   | HTTP GET /metrics every 15s
   v
node_exporter:9100
```

Prometheus lưu samples theo mô hình:

```text
metric_name{label1="value1",label2="value2"} value timestamp
```

Ví dụ:

```text
node_memory_MemAvailable_bytes{instance="10.0.1.20:9100",job="node"} 2.947e+09
```

Prometheus architecture chính thức cũng mô tả Prometheus server scrape các instrumented jobs/exporters, lưu time series, chạy rules, tạo alerts và cho Grafana query dữ liệu để visualize. [prometheus.io](https://prometheus.io/docs/introduction/overview/?trk=public_profile__posts-text&utm_source=chatgpt.com)

---

# 5. Deep Theory — Metrics và time series

## Metric khác log như thế nào?

Log thường trả lời:

```text
"Request /api/orders failed at 14:37:02 because connection timeout."
```

Metric trả lời:

```text
"Trong 5 phút vừa rồi error rate là 7.2%."
```

Logs phù hợp cho chi tiết một event.

Metrics phù hợp cho:

```text
trend
capacity
threshold
SLO
alert
dashboard
correlation
```

Monitoring tốt thường cần cả metrics lẫn logs.

---

## Metric name và labels

Ví dụ:

```promql
node_cpu_seconds_total{
    cpu="0",
    instance="10.0.1.20:9100",
    job="node",
    mode="idle"
}
```

`node_cpu_seconds_total` là metric name.

Các labels bao gồm:

```text
cpu
instance
job
mode
```

Labels biến một metric thành nhiều time series.

Ví dụ:

```text
node_cpu_seconds_total{cpu="0",mode="idle"}
node_cpu_seconds_total{cpu="0",mode="system"}
node_cpu_seconds_total{cpu="0",mode="user"}
node_cpu_seconds_total{cpu="1",mode="idle"}
...
```

Điều này rất mạnh nhưng có một rủi ro quan trọng: **cardinality explosion**.

Không nên tạo label kiểu:

```text
user_id
request_id
random_uuid
full_url_with_id
timestamp
```

vì mỗi giá trị label khác nhau tạo ra một time series mới.

---

## Counter

Counter chỉ tăng trong quá trình process tồn tại.

Ví dụ:

```text
http_requests_total
node_network_receive_bytes_total
node_cpu_seconds_total
```

Counter có thể reset khi process restart.

Vì vậy thường không nhìn raw counter mà dùng:

```promql
rate(...)
```

Ví dụ:

```promql
rate(node_network_receive_bytes_total[5m])
```

nghĩa là tốc độ bytes nhận được mỗi giây, được tính từ dữ liệu 5 phút gần nhất.

Prometheus/node_exporter documentation cũng đưa `rate(node_network_receive_bytes_total[1m])` làm ví dụ cho network traffic rate. [GitHub](https://github.com/prometheus/docs/blob/main/docs/guides/node-exporter.md?utm_source=chatgpt.com)

---

## Gauge

Gauge có thể tăng hoặc giảm.

Ví dụ:

```text
node_memory_MemAvailable_bytes
node_load1
node_filesystem_avail_bytes
```

CPU temperature, current sessions, current memory, queue depth thường phù hợp với Gauge.

---

## Histogram

Histogram quan trọng khi theo dõi latency hoặc size distribution.

Ví dụ application có:

```text
http_request_duration_seconds_bucket
http_request_duration_seconds_sum
http_request_duration_seconds_count
```

Ta có thể tính latency percentile với `histogram_quantile()`.

Đây là kiến thức rất quan trọng trong monitoring application, nhưng việc instrument Java application bằng Micrometer/OpenTelemetry không nằm trực tiếp trong syllabus Day 17 nên xem đây là **Supplementary**.

---

# 6. Prometheus

Prometheus thực hiện ba nhiệm vụ chính trong lab hôm nay:

```text
Collect
Store
Evaluate
```

Collect:

```text
scrape /metrics
```

Store:

```text
local TSDB
```

Evaluate:

```text
PromQL
recording rules
alerting rules
```

Prometheus sử dụng YAML configuration. Trong `scrape_configs`, mỗi `job_name` xác định một tập target. Prometheus getting-started documentation dùng cùng cấu trúc này và mặc định scrape target qua HTTP metrics endpoint. [prometheus.io](https://prometheus.io/docs/prometheus/latest/getting_started/?utm_source=chatgpt.com)

Ví dụ:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "node"
    static_configs:
      - targets:
          - "10.0.1.20:9100"
```

Ý nghĩa:

```text
every 15 seconds
       ↓
GET http://10.0.1.20:9100/metrics
       ↓
parse metrics
       ↓
write to TSDB
```

---

# 7. `job`, `instance` và `up`

Đây là ba khái niệm exam rất hay hỏi.

Nếu config:

```yaml
- job_name: "node"
  static_configs:
    - targets:
        - "10.0.1.20:9100"
```

Prometheus sẽ gắn labels kiểu:

```text
job="node"
instance="10.0.1.20:9100"
```

Prometheus đồng thời tạo metric đặc biệt:

```promql
up
```

Ví dụ:

```text
up{job="node",instance="10.0.1.20:9100"} 1
```

nghĩa là lần scrape gần nhất thành công.

Nếu:

```text
up{...} 0
```

Prometheus biết target tồn tại trong configuration nhưng scrape thất bại.

Đây là khác biệt rất quan trọng:

```text
up == 0
```

không nhất thiết có nghĩa:

```text
server is completely dead
```

Nó chỉ chứng minh:

```text
Prometheus cannot successfully scrape that target.
```

Nguyên nhân có thể là:

```text
node_exporter stopped
port 9100 blocked
route/network issue
DNS issue
timeout
target address wrong
TLS/auth configuration wrong
```

Đây là tư duy troubleshooting chuẩn.

---

# 8. node_exporter

node_exporter là exporter dành cho hardware và OS/kernel metrics của Unix-like systems. Official documentation cho biết nó expose nhiều hardware/kernel metrics và mặc định dùng port `9100`. [GitHub](https://github.com/prometheus/node_exporter/blob/master/README.md?utm_source=chatgpt.com)

Nó có các collectors cho CPU, memory, filesystem, network, disk, load, kernel và nhiều subsystem khác.

Điểm security quan trọng: node_exporter **không cần root**. Source code hiện tại thậm chí log warning nếu nó chạy bằng root vì exporter được thiết kế để chạy bằng unprivileged user. [GitHub](https://github.com/prometheus/node_exporter/blob/master/node_exporter.go?utm_source=chatgpt.com)

### Một số metric bắt buộc nên nhận ra

| Metric                              | Ý nghĩa                     |
| ----------------------------------- | --------------------------- |
| `node_cpu_seconds_total`            | CPU time theo mode          |
| `node_memory_MemTotal_bytes`        | Total RAM                   |
| `node_memory_MemAvailable_bytes`    | RAM có thể sử dụng          |
| `node_filesystem_size_bytes`        | Filesystem size             |
| `node_filesystem_avail_bytes`       | Available filesystem space  |
| `node_network_receive_bytes_total`  | Network receive counter     |
| `node_network_transmit_bytes_total` | Network transmit counter    |
| `node_load1`                        | Linux 1-minute load average |
| `node_boot_time_seconds`            | Boot timestamp              |

---

# 9. Các PromQL quan trọng của Day 17

## Host availability

```promql
up{job="node"}
```

Expected invariant:

```text
Healthy target → value = 1
```

---

## CPU utilization %

```promql
100 * (
  1 -
  avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

Tư duy:

```text
CPU utilization
≈ 100% - idle%
```

Không lấy trực tiếp:

```promql
node_cpu_seconds_total
```

vì đây là accumulated counter.

---

## Memory utilization %

```promql
100 * (
  1 -
  (
    node_memory_MemAvailable_bytes
    /
    node_memory_MemTotal_bytes
  )
)
```

Không nên chỉ lấy:

```text
free memory
```

vì Linux dùng RAM cho page cache; `MemAvailable` thường phản ánh lượng RAM có thể cung cấp cho workloads thực tế tốt hơn.

---

## Filesystem usage %

```promql
100 * (
  1 -
  (
    node_filesystem_avail_bytes{
      fstype!~"tmpfs|overlay"
    }
    /
    node_filesystem_size_bytes{
      fstype!~"tmpfs|overlay"
    }
  )
)
```

Bạn nên filter filesystem để tránh dashboard đầy các mount tạm hoặc container filesystems không liên quan.

---

## Network receive

```promql
rate(node_network_receive_bytes_total[5m])
```

---

## Network transmit

```promql
rate(node_network_transmit_bytes_total[5m])
```

---

## System load

```promql
node_load1
```

Load average không đồng nghĩa trực tiếp CPU percentage.

Một server có CPU thấp nhưng load cao nếu nhiều processes đang chờ uninterruptible I/O.

---

## Uptime

```promql
time() - node_boot_time_seconds
```

Giá trị là seconds.

---

# 10. Service, Host và Application Signals

Day 17 bắt buộc bạn phân biệt ba lớp này.

| Signal      | Ví dụ                             | Điều nó chứng minh                 |
| ----------- | --------------------------------- | ---------------------------------- |
| Host        | CPU, RAM, disk, network           | Linux host resource condition      |
| Service     | `up{job="node"}`                  | Prometheus scrape target reachable |
| Application | request rate, error rate, latency | Behavior của application           |

Một anti-pattern rất phổ biến:

```text
CPU normal → application healthy
```

Sai.

Application có thể:

```text
CPU = 20%
RAM = 50%
Disk = 40%
Tomcat process running
```

nhưng tất cả request trả về HTTP 500 vì DB password sai.

Một monitoring system trưởng thành cần nhìn qua nhiều layer.

---

# 11. Application signals và giới hạn của node_exporter

node_exporter **không tự hiểu Java application**.

Nó không biết:

```text
HTTP 500 rate
Java endpoint latency
JDBC connection pool
Tomcat request count
business transaction failures
```

Nếu Java app có Prometheus-compatible metrics endpoint, Prometheus có thể scrape endpoint đó trực tiếp.

Ví dụ hypothetical:

```yaml
- job_name: "java-app"
  metrics_path: "/actuator/prometheus"
  static_configs:
    - targets:
        - "10.0.1.20:8080"
```

Nhưng đây **chỉ hợp lệ nếu application thật sự đã có endpoint đó**.

Không được tự giả định mọi Spring Boot application đều có `/actuator/prometheus`.

Nó cần dependency/configuration phù hợp.

### Supplementary

Một số cách monitor application thường gặp:

```text
Micrometer + Prometheus registry
JMX Exporter
OpenTelemetry
Blackbox Exporter
application-native metrics
```

Đây là phần mở rộng, không phải mandatory Day 17.

---

# 12. Grafana

Prometheus lưu và query data rất tốt nhưng không phải dashboard system chính.

Grafana sẽ:

```text
query Prometheus
     ↓
receive time series
     ↓
visualize
```

Grafana Prometheus data source được tích hợp sẵn; không cần cài plugin Prometheus riêng. [Grafana Labs](https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/?pg=blog&plcmt=body-txt&utm_source=chatgpt.com)

Luồng:

```text
Browser
   ↓
Grafana :3000
   ↓
Prometheus Data Source
   ↓
Prometheus :9090
```

Không phải:

```text
Grafana → node_exporter
```

Grafana không scrape node_exporter trong kiến trúc này.

---

# 13. Dashboard design cho L1 Operations

Dashboard của fresher admin không cần đẹp trước; cần giúp trả lời nhanh:

```text
Is the host reachable?
Is CPU abnormal?
Is memory under pressure?
Is disk filling?
Is network activity unusual?
Has host recently rebooted?
```

Dashboard tối thiểu hôm nay nên có:

| Panel        | Query                                         |
| ------------ | --------------------------------------------- |
| Node status  | `up{job="node"}`                              |
| CPU %        | CPU PromQL ở trên                             |
| Memory %     | Memory PromQL                                 |
| Filesystem % | Filesystem PromQL                             |
| Load 1m      | `node_load1`                                  |
| Network RX   | `rate(node_network_receive_bytes_total[5m])`  |
| Network TX   | `rate(node_network_transmit_bytes_total[5m])` |
| Uptime       | `time() - node_boot_time_seconds`             |

Grafana dashboard được cấu thành từ panels và mỗi panel có query đối với data source. [Grafana Labs](https://grafana.com/docs/grafana/latest/visualizations/dashboards/build-dashboards/create-dashboard/?utm_source=chatgpt.com)

---

# 14. Alerting Mental Model

Monitoring:

```text
Human looks at dashboard.
```

Alerting:

```text
System detects abnormal condition
and tells a human/system.
```

Prometheus-native alert:

```text
Metric
  ↓
PromQL expression
  ↓
Alert rule
  ↓
Inactive / Pending / Firing
  ↓
Alertmanager
  ↓
Notification
```

Official Prometheus documentation định nghĩa alert rules dựa trên PromQL expression và hỗ trợ `for`, labels và annotations. [prometheus.io](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/?utm_source=chatgpt.com)

---

# 15. Alert states

### Inactive

Expression không match.

Ví dụ:

```promql
up{job="node"} == 0
```

node exporter healthy:

```text
up = 1
```

→ inactive.

### Pending

Rule có:

```yaml
for: 2m
```

và condition vừa trở thành true.

Ví dụ:

```text
14:00 target down
14:00:15 evaluation → condition true
14:01:00 → still true
```

Alert vẫn Pending.

### Firing

Condition tồn tại đủ `for: 2m`.

```text
14:02+
```

→ firing.

`for` giúp chống transient failures.

Official Prometheus docs xác nhận alert có `for` sẽ ở Pending trước khi chuyển sang Firing; nếu không có `for`, rule có thể active ngay từ lần evaluation đầu tiên. [prometheus.io](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/?utm_source=chatgpt.com)

---

# 16. Labels và annotations

Ví dụ:

```yaml
labels:
  severity: critical
  team: ops
```

Labels dùng cho:

```text
routing
grouping
filtering
deduplication
```

Annotations dùng cho human-readable information:

```yaml
annotations:
  summary: "Node exporter unavailable"
  description: "Prometheus cannot scrape {{ $labels.instance }}"
```

Rule tốt không chỉ nói:

```text
ERROR
```

mà nên giúp operator biết:

```text
what failed
which host
severity
possible meaning
what to inspect
```

---

# 17. Alertmanager

Alertmanager nhận alerts từ Prometheus.

Nó xử lý:

```text
grouping
deduplication
routing
silences
inhibition
notifications
```

Official Alertmanager documentation mô tả chính các chức năng này. [next.prometheus.io](https://next.prometheus.io/docs/alerting/latest/alertmanager/?utm_source=chatgpt.com)

Ví dụ 50 hosts đều down do network switch failure.

Không có grouping:

```text
50 notifications
```

Có grouping:

```text
1 grouped notification containing 50 affected instances
```

Đó là lý do Alertmanager không đơn thuần là “email sender”.

---

# 18. Guided Lab — Architecture

Lab hôm nay dùng một host để giảm độ phức tạp:

```text
┌────────────────────────────────────┐
│ Linux VM                           │
│                                    │
│ node_exporter :9100                │
│       ↑                            │
│       │ scrape                     │
│       │                            │
│ Prometheus :9090                   │
│       │                            │
│       ├──── Grafana :3000          │
│       │                            │
│       └──── Alertmanager :9093     │
│                  │                 │
│                  ↓                 │
│          Lab webhook :5001         │
└────────────────────────────────────┘
```

### Safety warning

Không thực hiện lab trên production host.

Failure injection hôm nay sẽ là:

```text
stop node_exporter
```

Không dùng phương pháp nguy hiểm như:

```text
fill filesystem
kill random processes
change firewall blindly
consume 100% memory
delete data
```

---

# 19. Lab Step 0 — Baseline inspection

Trước khi thay đổi máy:

```bash
hostnamectl
uname -a
cat /etc/os-release

df -h
free -h

systemctl --version
ss -lntp
```

Tạo evidence directory:

```bash
mkdir -p ~/day17-evidence
```

Lưu baseline:

```bash
{
  date
  hostnamectl
  uname -a
  df -h
  free -h
  ss -lntp
} | tee ~/day17-evidence/00-baseline.txt
```

Đây là operational habit rất quan trọng:

```text
inspect first
change second
validate third
preserve evidence
```

---

# 20. Lab Step 1 — Install node_exporter

Tạo least-privilege service account:

```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  node_exporter
```

Kiểm tra:

```bash
getent passwd node_exporter
```

Sau khi download đúng binary từ official Prometheus release page và verify checksum theo release artifact của bạn, copy binary:

```bash
sudo install \
  -o root \
  -g root \
  -m 0755 \
  node_exporter \
  /usr/local/bin/node_exporter
```

Kiểm tra:

```bash
/usr/local/bin/node_exporter --version
```

Tạo service:

```bash
sudo tee /etc/systemd/system/node_exporter.service >/dev/null <<'EOF'
[Unit]
Description=Prometheus Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --web.listen-address=:9100
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true

[Install]
WantedBy=multi-user.target
EOF
```

Validate systemd file:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/node_exporter.service
```

Reload:

```bash
sudo systemctl daemon-reload
```

Enable + start:

```bash
sudo systemctl enable --now node_exporter
```

---

# 21. Validate node_exporter

Service state:

```bash
systemctl status node_exporter --no-pager
```

Expected invariant:

```text
Active: active (running)
```

Process:

```bash
ps -ef | grep '[n]ode_exporter'
```

Listening socket:

```bash
sudo ss -lntp | grep ':9100'
```

HTTP:

```bash
curl -fsS http://127.0.0.1:9100/metrics | head
```

Node metrics:

```bash
curl -fsS http://127.0.0.1:9100/metrics \
  | grep '^node_' \
  | head
```

Evidence:

```bash
{
  date
  systemctl status node_exporter --no-pager
  ss -lntp
  curl -fsS http://127.0.0.1:9100/metrics | head -30
} > ~/day17-evidence/01-node-exporter.txt
```

---

# 22. Lab Step 2 — Install Prometheus

Create account:

```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  prometheus
```

Directories:

```bash
sudo install -d \
  -o root \
  -g prometheus \
  -m 0750 \
  /etc/prometheus

sudo install -d \
  -o prometheus \
  -g prometheus \
  -m 0750 \
  /var/lib/prometheus

sudo install -d \
  -o root \
  -g prometheus \
  -m 0750 \
  /etc/prometheus/rules
```

Install binaries from the verified Prometheus tarball:

```bash
sudo install -m 0755 prometheus /usr/local/bin/prometheus
sudo install -m 0755 promtool /usr/local/bin/promtool
```

Check:

```bash
prometheus --version
promtool --version
```

---

# 23. Prometheus configuration

Create:

```bash
sudo tee /etc/prometheus/prometheus.yml >/dev/null <<'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "/etc/prometheus/rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "127.0.0.1:9093"

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets:
          - "127.0.0.1:9090"

  - job_name: "node"
    static_configs:
      - targets:
          - "127.0.0.1:9100"
EOF
```

Set ownership:

```bash
sudo chown root:prometheus /etc/prometheus/prometheus.yml
sudo chmod 0640 /etc/prometheus/prometheus.yml
```

Prometheus documentation xác nhận `scrape_configs` định nghĩa targets và `rule_files` dùng để load rule files. [prometheus.io](https://prometheus.io/docs/prometheus/latest/configuration/configuration/?target=_blank&utm_source=chatgpt.com)

---

# 24. Config validation trước restart

Đây là operation discipline cực quan trọng.

Không làm:

```bash
sudo systemctl restart prometheus
```

ngay sau khi edit file.

Làm:

```bash
sudo promtool check config \
  /etc/prometheus/prometheus.yml
```

`promtool check config` được thiết kế để kiểm tra Prometheus configuration trước khi apply. [prometheus.io](https://prometheus.io/docs/prometheus/2.55/command-line/promtool/?utm_source=chatgpt.com)

Expected:

```text
SUCCESS
```

Nếu command fail:

```text
DO NOT restart.
```

Fix config trước.

---

# 25. Prometheus systemd service

```bash
sudo tee /etc/systemd/system/prometheus.service >/dev/null <<'EOF'
[Unit]
Description=Prometheus Monitoring Server
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true

[Install]
WantedBy=multi-user.target
EOF
```

Validate:

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/prometheus.service
```

Apply:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
```

---

# 26. Validate Prometheus

```bash
systemctl status prometheus --no-pager
```

Socket:

```bash
sudo ss -lntp | grep ':9090'
```

Health:

```bash
curl -fsS http://127.0.0.1:9090/-/healthy
```

Ready:

```bash
curl -fsS http://127.0.0.1:9090/-/ready
```

Prometheus provides `/−/healthy` and `/−/ready` management endpoints specifically cho health/readiness checks. [prometheus.io](https://prometheus.io/docs/prometheus/latest/management_api/?utm_source=chatgpt.com)

Check targets:

```text
http://SERVER:9090/targets
```

Bạn phải thấy:

```text
prometheus → UP
node       → UP
```

---

# 27. PromQL verification

Trong Prometheus query UI thử:

```promql
up
```

Sau đó:

```promql
up{job="node"}
```

CPU:

```promql
100 * (
  1 -
  avg by(instance)(
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

Memory:

```promql
100 * (
  1 -
  node_memory_MemAvailable_bytes /
  node_memory_MemTotal_bytes
)
```

Filesystem:

```promql
100 * (
  1 -
  node_filesystem_avail_bytes{
    fstype!~"tmpfs|overlay"
  }
  /
  node_filesystem_size_bytes{
    fstype!~"tmpfs|overlay"
  }
)
```

Prometheus expression browser chủ yếu phù hợp cho ad-hoc querying và debugging; official docs khuyến nghị Grafana cho visualization dashboard. [prometheus.io](https://prometheus.io/docs/visualization/?utm_source=chatgpt.com)

---

# 28. Lab Step 3 — Alertmanager

Create account:

```bash
sudo useradd \
  --system \
  --no-create-home \
  --shell /usr/sbin/nologin \
  alertmanager
```

Directories:

```bash
sudo install -d \
  -o root \
  -g alertmanager \
  -m 0750 \
  /etc/alertmanager

sudo install -d \
  -o alertmanager \
  -g alertmanager \
  -m 0750 \
  /var/lib/alertmanager
```

Install verified release binaries:

```bash
sudo install -m 0755 alertmanager /usr/local/bin/alertmanager
sudo install -m 0755 amtool /usr/local/bin/amtool
```

Check:

```bash
alertmanager --version
amtool --version
```

---

# 29. Safe local notification receiver

Để không cần Slack token, SMTP password hay secret thật, lab sẽ dùng local webhook.

Create:

```bash
cat > ~/day17-webhook.py <<'PY'
from http.server import BaseHTTPRequestHandler, HTTPServer

class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        length = int(self.headers.get("Content-Length", "0"))
        body = self.rfile.read(length)

        print("\n=== ALERT RECEIVED ===")
        print(body.decode("utf-8", errors="replace"))
        print("=== END ALERT ===\n")

        self.send_response(200)
        self.end_headers()

HTTPServer(("127.0.0.1", 5001), Handler).serve_forever()
PY
```

Start nó trong terminal riêng:

```bash
python3 ~/day17-webhook.py \
  | tee ~/day17-evidence/webhook-notifications.log
```

Check socket:

```bash
ss -lnt | grep ':5001'
```

Đây chỉ là **lab receiver**, không phải production notification system.

---

# 30. Alertmanager configuration

```bash
sudo tee /etc/alertmanager/alertmanager.yml >/dev/null <<'EOF'
route:
  receiver: "lab-webhook"
  group_by:
    - "alertname"
    - "instance"
  group_wait: 5s
  group_interval: 30s
  repeat_interval: 1h

receivers:
  - name: "lab-webhook"
    webhook_configs:
      - url: "http://127.0.0.1:5001/"
        send_resolved: true
EOF
```

Permissions:

```bash
sudo chown root:alertmanager \
  /etc/alertmanager/alertmanager.yml

sudo chmod 0640 \
  /etc/alertmanager/alertmanager.yml
```

Validate:

```bash
sudo amtool check-config \
  /etc/alertmanager/alertmanager.yml
```

Current Alertmanager documentation chính thức khuyến nghị `amtool check-config` để validate configuration trước khi apply. [GitHub](https://github.com/prometheus/alertmanager/blob/main/docs/configuration.md?plain=1&utm_source=chatgpt.com)

---

# 31. Alertmanager systemd service

```bash
sudo tee /etc/systemd/system/alertmanager.service >/dev/null <<'EOF'
[Unit]
Description=Prometheus Alertmanager
Wants=network-online.target
After=network-online.target

[Service]
User=alertmanager
Group=alertmanager
Type=simple
ExecStart=/usr/local/bin/alertmanager \
  --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/var/lib/alertmanager
Restart=on-failure
RestartSec=5
NoNewPrivileges=true
PrivateTmp=true
ProtectHome=true

[Install]
WantedBy=multi-user.target
EOF
```

Apply:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now alertmanager
```

Validate:

```bash
systemctl status alertmanager --no-pager

curl -fsS http://127.0.0.1:9093/-/healthy
curl -fsS http://127.0.0.1:9093/-/ready
```

Alertmanager cung cấp health, readiness và reload endpoints tương tự. [prometheus.io](https://prometheus.io/docs/alerting/latest/management_api/?utm_source=chatgpt.com)

---

# 32. Lab Step 4 — Create Alert Rule

Create:

```bash
sudo tee /etc/prometheus/rules/host-alerts.yml >/dev/null <<'EOF'
groups:
  - name: "host-availability"
    rules:
      - alert: "NodeExporterDown"
        expr: up{job="node"} == 0
        for: 2m

        labels:
          severity: "critical"
          team: "operations"

        annotations:
          summary: "node_exporter unavailable on {{ $labels.instance }}"
          description: "Prometheus has been unable to scrape {{ $labels.instance }} for more than 2 minutes."
          runbook_hint: "Check node_exporter service, listening socket, HTTP endpoint, network path, and firewall."
EOF
```

Permissions:

```bash
sudo chown root:prometheus \
  /etc/prometheus/rules/host-alerts.yml

sudo chmod 0640 \
  /etc/prometheus/rules/host-alerts.yml
```

---

# 33. Validate rule trước apply

```bash
sudo promtool check rules \
  /etc/prometheus/rules/host-alerts.yml
```

Then validate toàn bộ config:

```bash
sudo promtool check config \
  /etc/prometheus/prometheus.yml
```

Rule validation trước reload/restart là operation behavior bạn nên biến thành thói quen. Official Prometheus documentation cũng hướng dẫn `promtool check rules`. [prometheus.io](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/?utm_source=chatgpt.com)

---

# 34. Reload Prometheus

Bạn có thể restart:

```bash
sudo systemctl restart prometheus
```

Nhưng production operation tốt hơn là reload khi phù hợp.

Prometheus hỗ trợ `SIGHUP` để reload configuration và rule files. HTTP `/-/reload` cũng có thể dùng nếu server được bật bằng `--web.enable-lifecycle`. [prometheus.io](https://prometheus.io/docs/prometheus/latest/configuration/configuration/?target=_blank&utm_source=chatgpt.com)

Với systemd lab đơn giản:

```bash
sudo systemctl kill -s HUP prometheus
```

Verify:

```bash
journalctl -u prometheus \
  --since "5 minutes ago" \
  --no-pager
```

---

# 35. Verify rule loaded

Prometheus UI:

```text
http://SERVER:9090/rules
```

Alerts:

```text
http://SERVER:9090/alerts
```

Healthy state ban đầu phải:

```text
NodeExporterDown
State: Inactive
```

Query:

```promql
up{job="node"}
```

Expected:

```text
1
```

---

# 36. Lab Step 5 — Install Grafana

Trên Ubuntu/Debian, Grafana hiện khuyến nghị cài từ official APT repository. [Grafana Labs](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/?trk=article-ssr-frontend-pulse_little-text-block&utm_source=chatgpt.com)

Dependencies:

```bash
sudo apt-get update
sudo apt-get install -y \
  apt-transport-https \
  wget \
  gnupg
```

Key:

```bash
sudo mkdir -p /etc/apt/keyrings

sudo wget \
  -O /etc/apt/keyrings/grafana.asc \
  https://apt.grafana.com/gpg-full.key

sudo chmod 644 \
  /etc/apt/keyrings/grafana.asc
```

Repository:

```bash
echo \
'deb [signed-by=/etc/apt/keyrings/grafana.asc] https://apt.grafana.com stable main' \
| sudo tee /etc/apt/sources.list.d/grafana.list
```

Install OSS:

```bash
sudo apt-get update
sudo apt-get install -y grafana
```

Enable:

```bash
sudo systemctl enable --now grafana-server
```

Verify:

```bash
systemctl status grafana-server --no-pager
sudo ss -lntp | grep ':3000'
```

---

# 37. Grafana first login

Open:

```text
http://SERVER:3000
```

Current Grafana documentation says default first login is:

```text
username: admin
password: admin
```

và Grafana sẽ yêu cầu đổi password. Bạn phải đổi ngay; không giữ credential mặc định. [Grafana Labs](https://grafana.com/docs/grafana/latest/setup-grafana/sign-in-to-grafana/?utm_source=chatgpt.com)

---

# 38. Add Prometheus Data Source

Trong Grafana:

```text
Connections
→ Data sources
→ Add data source
→ Prometheus
```

URL, vì cùng machine:

```text
http://127.0.0.1:9090
```

Save & test.

Grafana documentation xác nhận Prometheus data source đã được tích hợp sẵn và yêu cầu cung cấp Prometheus server URL. [Grafana Labs](https://grafana.com/docs/grafana/latest/datasources/prometheus/configure/?pg=blog&plcmt=body-txt&utm_source=chatgpt.com)

---

# 39. Build Day 17 dashboard

Create:

```text
Dashboards
→ New
→ New Dashboard
→ Add visualization
→ Prometheus
```

Panel 1:

```text
Title:
Node Availability

Query:
up{job="node"}

Visualization:
Stat
```

Set:

```text
1 → Healthy
0 → Down
```

Panel 2:

```text
Title:
CPU Utilization %

Query:
100 * (
  1 -
  avg by(instance)(
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

Unit:

```text
Percent (0-100)
```

Panel 3:

```text
Title:
Memory Utilization %

Query:
100 * (
  1 -
  node_memory_MemAvailable_bytes /
  node_memory_MemTotal_bytes
)
```

Panel 4:

```text
Title:
Filesystem Used %

Query:
100 * (
  1 -
  node_filesystem_avail_bytes{
    fstype!~"tmpfs|overlay"
  }
  /
  node_filesystem_size_bytes{
    fstype!~"tmpfs|overlay"
  }
)
```

Panel 5:

```text
Title:
1-minute Load Average

Query:
node_load1
```

Panel 6:

```text
Title:
Network Receive

Query:
rate(node_network_receive_bytes_total[5m])
```

Panel 7:

```text
Title:
Network Transmit

Query:
rate(node_network_transmit_bytes_total[5m])
```

Panel 8:

```text
Title:
Host Uptime

Query:
time() - node_boot_time_seconds
```

Save dashboard:

```text
Day17 Linux Monitoring
```

---

# 40. Dashboard validation

Đừng chỉ nhìn thấy chart và kết luận “works”.

Bạn cần kiểm chứng source.

Ví dụ memory:

Linux:

```bash
free -h
```

Prometheus:

```promql
node_memory_MemAvailable_bytes
```

Grafana:

```text
Memory Utilization %
```

Ba lớp không nhất thiết hiển thị số giống hoàn toàn do sampling/time/rate calculation, nhưng phải hợp lý và consistent.

CPU:

```bash
top
```

so với:

```promql
100 * (
  1 -
  avg by(instance)(
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

Disk:

```bash
df -h
```

so với filesystem panel.

Đây là validation thật.

---

# 41. Failure Injection — phần bắt buộc của Day 17

Đây là bước quyết định bạn có thực sự hoàn thành lab hay chưa.

Baseline trước failure:

```bash
systemctl is-active node_exporter
```

Expected:

```text
active
```

Prometheus:

```promql
up{job="node"}
```

Expected:

```text
1
```

Alert:

```text
Inactive
```

---

# 42. Inject failure

Stop **chỉ node_exporter**:

```bash
sudo systemctl stop node_exporter
```

Không disable.

Không remove.

Không kill random process.

Check:

```bash
systemctl status node_exporter --no-pager
```

Expected:

```text
inactive (dead)
```

Check port:

```bash
ss -lnt | grep ':9100'
```

Expected invariant:

```text
no listener on 9100
```

Test:

```bash
curl -v http://127.0.0.1:9100/metrics
```

Expected:

```text
connection refused
```

---

# 43. Observe Prometheus detection

Sau scrape interval:

```promql
up{job="node"}
```

phải chuyển:

```text
1 → 0
```

Prometheus Targets:

```text
node → DOWN
```

Dashboard:

```text
Node Availability → 0 / Down
```

Alert:

```text
Inactive
→ Pending
```

Do:

```yaml
for: 2m
```

nên alert chưa firing ngay.

Sau khoảng 2 phút condition vẫn đúng:

```text
Pending
→ Firing
```

Đây chính là **alert validation**.

---

# 44. Notification validation

Prometheus gửi alert tới Alertmanager.

Alertmanager route sang:

```text
http://127.0.0.1:5001/
```

Terminal chạy webhook sẽ xuất JSON.

Bạn phải thấy nội dung chứa đại loại:

```text
status: firing
alertname: NodeExporterDown
severity: critical
instance: 127.0.0.1:9100
```

Không cần JSON giống hệt từng field; cái cần chứng minh là notification đã thực sự tới receiver.

Preserve:

```text
~/day17-evidence/webhook-notifications.log
```

Đây là evidence quan trọng nhất của assignment:

```text
alert fired
AND
notification delivered
```

---

# 45. Recovery

L1-safe remediation vì chúng ta biết failure do chính lab inject:

```bash
sudo systemctl start node_exporter
```

Validate service:

```bash
systemctl is-active node_exporter
```

Expected:

```text
active
```

Socket:

```bash
ss -lnt | grep ':9100'
```

HTTP:

```bash
curl -fsS http://127.0.0.1:9100/metrics >/dev/null
echo $?
```

Expected:

```text
0
```

Prometheus:

```promql
up{job="node"}
```

Expected:

```text
1
```

Alert sẽ trở lại resolved/inactive khi condition không còn đúng.

Do:

```yaml
send_resolved: true
```

local webhook cũng phải nhận resolved notification.

Đây là bằng chứng đầy đủ:

```text
Healthy
→ Failure
→ Detection
→ Pending
→ Firing
→ Notification
→ Recovery
→ Resolved
```

---

# 46. Evidence pack cần giữ

Sau lab, evidence folder lý tưởng chứa các artefacts tương đương:

```text
day17-evidence/
├── 00-baseline.txt
├── 01-node-exporter.txt
├── prometheus-config-validation.txt
├── rule-validation.txt
├── alertmanager-validation.txt
├── target-healthy.txt
├── alert-pending.txt
├── alert-firing.txt
├── webhook-notifications.log
├── service-failure.txt
└── recovery-validation.txt
```

Bạn có thể lưu commands:

```bash
promtool check config /etc/prometheus/prometheus.yml \
  | tee ~/day17-evidence/prometheus-config-validation.txt
```

```bash
promtool check rules /etc/prometheus/rules/host-alerts.yml \
  | tee ~/day17-evidence/rule-validation.txt
```

```bash
sudo amtool check-config /etc/alertmanager/alertmanager.yml \
  | tee ~/day17-evidence/alertmanager-validation.txt
```

---

# 47. Troubleshooting framework

Day 17 rất quan trọng vì monitoring system bản thân nó cũng có thể hỏng.

Bạn phải troubleshoot theo:

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
Hypothesis
↓
Safest test
↓
Root cause
↓
Remediation
↓
Validation
```

Không được bắt đầu bằng:

```text
restart everything
```

---

# 48. Scenario A — `up{job="node"} == 0`

Symptom:

```text
Grafana says node down.
```

Đầu tiên phải hỏi:

```text
Is Grafana wrong?
Is Prometheus missing data?
Can Prometheus reach node_exporter?
Is node_exporter listening?
```

On exporter host:

```bash
systemctl status node_exporter --no-pager
```

```bash
ss -lntp | grep ':9100'
```

```bash
curl -v http://127.0.0.1:9100/metrics
```

If local works nhưng remote Prometheus không scrape được:

```text
network/firewall/Security Group/routing
```

Test từ Prometheus host:

```bash
curl -v http://NODE_IP:9100/metrics
```

Nếu:

```text
Connection refused
```

khác với:

```text
Connection timed out
```

Refused thường gợi ý host reachable nhưng không listener/reject.

Timeout thường gợi ý packet drop/firewall/routing.

---

# 49. Scenario B — Prometheus không start sau config change

Symptom:

```bash
systemctl status prometheus
```

shows failed.

Evidence:

```bash
journalctl -u prometheus -n 100 --no-pager
```

Check configuration:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Check rule:

```bash
promtool check rules /etc/prometheus/rules/host-alerts.yml
```

Rất nhiều issue chỉ là YAML indentation.

Ví dụ sai:

```yaml
scrape_configs:
  - job_name: node
static_configs:
  - targets:
```

Hierarchy bị phá.

Prevention:

```text
validate before restart
```

---

# 50. Scenario C — Prometheus healthy nhưng Grafana “No data”

Không restart Grafana ngay.

Đầu tiên query Prometheus trực tiếp:

```promql
up
```

Nếu Prometheus có data:

```text
Prometheus collection works.
```

Điều tra Grafana:

```text
data source URL
query
time range
labels
variables
permissions
```

Ví dụ Grafana query:

```promql
up{job="node-exporter"}
```

nhưng Prometheus config là:

```yaml
job_name: "node"
```

→ No data.

Root cause không phải server down.

Root cause là label mismatch.

---

# 51. Scenario D — Alert expression đúng nhưng không fire

Kiểm tra:

```text
1. expression result
2. evaluation
3. for duration
4. rule loaded
5. labels/filter
```

Query expression trực tiếp:

```promql
up{job="node"} == 0
```

Nếu result empty:

```text
alert condition false
```

Nếu result có vector nhưng rule Pending:

```text
for duration not yet satisfied
```

Nếu rule không xuất hiện:

```text
rule file not loaded
```

Check:

```bash
promtool check config /etc/prometheus/prometheus.yml
promtool check rules /etc/prometheus/rules/host-alerts.yml
```

---

# 52. Scenario E — Alert firing nhưng notification không tới

Đây là tầng khác.

Nếu Prometheus `/alerts`:

```text
FIRING
```

nhưng webhook không nhận:

```text
Prometheus rule itself is working.
```

Tiếp tục:

```bash
systemctl status alertmanager
```

```bash
curl -fsS http://127.0.0.1:9093/-/ready
```

```bash
journalctl -u alertmanager \
  --since "10 minutes ago" \
  --no-pager
```

Check receiver:

```bash
ss -lnt | grep ':5001'
```

Check config:

```bash
amtool check-config /etc/alertmanager/alertmanager.yml
```

Layer reasoning:

```text
Metric OK
↓
Rule OK
↓
Alert FIRING
↓
Prometheus → Alertmanager?
↓
Routing?
↓
Receiver reachable?
```

Đừng quay lại investigate CPU nếu alert đã FIRING.

---

# 53. Service signal trap

Nếu:

```promql
up{job="node"} == 1
```

thì điều đó chỉ chứng minh node_exporter được scrape.

Nó không chứng minh:

```text
Tomcat active
PostgreSQL available
Java endpoint healthy
business transaction successful
```

Đây là câu phân biệt rất quan trọng trong exam.

Monitoring target:

```text
node_exporter
```

không đồng nghĩa monitoring:

```text
all applications on the node
```

---

# 54. Mistakes / Anti-patterns

| Anti-pattern                                   | Vì sao sai                                |
| ---------------------------------------------- | ----------------------------------------- |
| Dashboard có chart nhưng không validate source | Có thể query sai                          |
| Alert không có `for`                           | Dễ gây transient alert noise              |
| Alert mọi thứ là `critical`                    | Không có prioritization                   |
| Expose port 9100 ra Internet                   | Không cần thiết và tăng attack surface    |
| Chạy node_exporter root                        | Không cần                                 |
| Restart Prometheus trước khi `promtool check`  | Có thể gây monitoring outage              |
| Chỉ theo dõi CPU                               | Không đủ xác định health                  |
| Xem `up=1` là Java app healthy                 | Sai abstraction layer                     |
| Hard-code secret trong config                  | Security risk                             |
| Monitor nhưng không test alert                 | Không biết notification path có hoạt động |

---

# 55. Security & Least Privilege

Monitoring infrastructure có giá trị lớn với attacker vì nó tiết lộ:

```text
hostnames
IP addresses
service names
filesystem
resource usage
architecture
environment
incident state
```

Vì vậy Prometheus/Grafana/node_exporter không nên mặc định mở public Internet.

Nếu trên AWS, conceptual Security Group:

```text
9100:
Source = Prometheus server only

9093:
Source = Prometheus / authorized operators only

9090:
Source = Grafana / operations network only

3000:
Source = approved admin/VPN network only

22:
Source = approved management IP/VPN only
```

Không dùng:

```text
0.0.0.0/0 → 9100
0.0.0.0/0 → 9090
0.0.0.0/0 → 9093
```

trừ lab được kiểm soát chặt và có lý do rõ ràng.

---

# 56. Monitoring secrets

Không đưa credential trực tiếp vào dashboard query.

Không commit:

```text
Slack webhook
SMTP password
API token
Grafana admin password
```

vào Git public.

Nếu company notification integration yêu cầu secret, sử dụng secret-management process đã được phê duyệt.

Không paste secret vào ticket/evidence.

---

# 57. Operations / Safe Change

Một config change đúng quy trình:

```text
Inspect current state
        ↓
Backup config
        ↓
Edit
        ↓
Static validation
        ↓
Apply/reload
        ↓
Service health check
        ↓
Functional validation
        ↓
Observe logs
        ↓
Preserve evidence
```

Ví dụ trước sửa:

```bash
sudo cp \
  /etc/prometheus/prometheus.yml \
  /etc/prometheus/prometheus.yml.bak
```

Edit.

Validate:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Reload.

Validate:

```bash
curl -fsS http://127.0.0.1:9090/-/ready
```

Targets.

Rules.

Grafana.

Alerts.

---

# 58. Rollback

Nếu configuration mới gây lỗi:

```bash
sudo cp \
  /etc/prometheus/prometheus.yml.bak \
  /etc/prometheus/prometheus.yml
```

Validate rollback config:

```bash
promtool check config /etc/prometheus/prometheus.yml
```

Apply:

```bash
sudo systemctl restart prometheus
```

Then verify:

```bash
systemctl is-active prometheus
curl -fsS http://127.0.0.1:9090/-/ready
```

Rollback cũng phải validate.

Không có khái niệm:

```text
restored file = incident resolved
```

cho đến khi service và functionality được chứng minh.

---

# 59. Reload vs Restart

Prometheus hỗ trợ reload config/runtime rule files bằng `SIGHUP`; malformed configuration sẽ không được apply. [prometheus.io](https://prometheus.io/docs/prometheus/latest/configuration/configuration/?target=_blank&utm_source=chatgpt.com)

Điều này thường tốt hơn restart vì:

```text
less interruption
lower operational risk
```

Tuy nhiên không phải mọi command-line flag có thể reload.

Configuration passed bằng flags như storage path thuộc startup behavior.

Do đó:

```text
config/rules change → often reload
binary/flags/service unit change → usually restart
```

---

# 60. Monitoring the monitor

Một monitoring platform mà bản thân chết im lặng là nguy hiểm.

Bạn nên biết:

```text
Prometheus can scrape itself.
Alertmanager exposes its own metrics.
Grafana exposes internal metrics.
```

Prometheus getting-started guide cũng dùng chính Prometheus làm scrape target đầu tiên. [prometheus.io](https://prometheus.io/docs/prometheus/latest/getting_started/?utm_source=chatgpt.com)

Một thực tế quan trọng:

```text
Prometheus cannot reliably alert you about its own complete disappearance
if nothing else monitors Prometheus.
```

Production thường cần redundancy/external monitoring.

Đây là **Supplementary**.

---

# 61. Grafana Alerting vs Prometheus Alerting

Grafana hiện hỗ trợ cả:

```text
Grafana-managed alert rules

và

data source-managed Prometheus rules
```

Với Prometheus-native rules, Grafana có thể display chúng, nhưng rule vẫn được quản lý trong Prometheus rule files; Grafana view là read-only đối với loại rule này. [Grafana Labs](https://grafana.com/docs/grafana/latest/datasources/prometheus/alerting/?utm_source=chatgpt.com)

Trong Day 17 tôi chọn:

```text
Prometheus = alert rule engine
Grafana = dashboard
Alertmanager = notifications
```

vì kiến trúc này làm rõ trách nhiệm từng component.

---

# 62. Host signals bạn phải biết cách diễn giải

### High CPU

Có thể do:

```text
Java workload
query processing
backup
compression
runaway process
traffic spike
GC
```

L1 response:

```bash
top
ps aux --sort=-%cpu | head
```

Không kill process chỉ vì CPU cao.

---

### High memory

Có thể do:

```text
Java heap
PostgreSQL
page cache
memory leak
load
```

Evidence:

```bash
free -h
ps aux --sort=-%mem | head
```

Correlate với JVM config từ Day 13.

---

### High disk usage

Evidence:

```bash
df -h
df -i
du -xhd1 /var 2>/dev/null | sort -h
```

Không được:

```bash
rm -rf logs
```

một cách mù quáng.

Check:

```text
retention
rotation
ownership
application dependency
backup requirement
```

---

### Network anomaly

Metrics:

```promql
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])
```

Linux evidence:

```bash
ip -s link
ss -s
```

Need correlate trước khi kết luận.

---

# 63. Integrated troubleshooting với các Day trước

Khi Grafana cho thấy application issue, cách nghĩ không phải:

```text
Monitoring problem?
```

mà là full-stack:

```text
Client
↓
DNS
↓
Network
↓
AWS SG/NACL/routes
↓
EC2
↓
Linux
↓
socket
↓
Tomcat/Java
↓
application configuration
↓
PostgreSQL
↓
metrics/logs
↓
alert
```

Ví dụ:

```text
Java requests slow
```

Grafana:

```text
CPU normal
memory normal
network normal
```

PostgreSQL:

```text
sessions locked
```

Từ Day 16 bạn sẽ correlate database lock với application latency.

Day 17 là lúc các kiến thức bắt đầu kết nối thành operations system.

---

# 64. L1 remediation vs escalation

Bạn có thể xử lý trong L1 nếu evidence xác định root cause rõ và remediation nằm trong runbook, chẳng hạn node_exporter lab service bị stopped và approved action là start service.

Nên escalate nếu:

```text
unknown production outage
corrupted Prometheus TSDB
security incident suspected
repeated unexplained service crashes
firewall/security-group change requires approval
application metric implementation missing
alert storm caused by architecture issue
disk expansion required beyond authority
credential problem requiring privileged secret access
```

Escalation tốt phải mang evidence:

```text
symptom
impact
time
scope
commands run
results
logs
recent changes
what was ruled out
current service state
recommended next layer
```

---

# 65. Example L1 incident reasoning

Symptom:

```text
Alert: NodeExporterDown
instance=10.0.1.20:9100
```

Expected:

```text
up = 1
```

Observed:

```text
up = 0
```

Prometheus server:

```bash
curl http://10.0.1.20:9100/metrics
```

Result:

```text
Connection refused
```

Host:

```bash
systemctl status node_exporter
```

Result:

```text
failed
```

Logs:

```bash
journalctl -u node_exporter
```

Result:

```text
bind: address already in use
```

Port:

```bash
ss -lntp | grep ':9100'
```

shows another process.

Root cause:

```text
Port conflict.
```

Không phải:

```text
network issue
```

L1 không kill unknown process ngay.

Escalate nếu ownership của competing process không rõ.

Đây là evidence-driven troubleshooting.

---

# 66. Exam Focus — lý thuyết

Bạn cần có thể giải thích bằng lời, không chỉ nhớ command:

| Topic              | Bạn phải trả lời được                                               |
| ------------------ | ------------------------------------------------------------------- |
| Prometheus         | Scrape và lưu time-series metrics thế nào                           |
| node_exporter      | Nó expose Linux host metrics ra sao                                 |
| Grafana            | Vì sao nó query Prometheus thay vì scrape host                      |
| `job`              | Logical group của targets                                           |
| `instance`         | Target identity                                                     |
| `up`               | Last scrape success/failure                                         |
| Counter            | Vì sao thường dùng `rate()`                                         |
| Gauge              | Vì sao có thể tăng/giảm                                             |
| `for`              | Vì sao alert có Pending                                             |
| Alertmanager       | Vì sao Prometheus không trực tiếp làm toàn bộ notification handling |
| Dashboard          | Dashboard không đồng nghĩa alert                                    |
| Host vs app health | Vì sao CPU tốt không chứng minh application tốt                     |
| Evidence           | Cách chứng minh alert thật sự end-to-end                            |

---

# 67. Exam Focus — practical

Nếu practical exam đưa máy trắng và yêu cầu:

```text
Deploy basic Linux monitoring
```

trình tự mạnh là:

```text
inspect host
↓
install node_exporter
↓
systemd + least privilege
↓
validate :9100
↓
install Prometheus
↓
configure scrape
↓
promtool check
↓
validate targets
↓
query metrics
↓
install/connect Grafana
↓
dashboard
↓
alert rule
↓
promtool check rules
↓
Alertmanager
↓
notification route
↓
failure injection
↓
capture firing evidence
↓
restore
↓
capture resolved evidence
```

Nếu bạn chỉ dựng Grafana xong thì bài chưa hoàn thành.

---

# 68. Knowledge Check

1. Prometheus khác Grafana về trách nhiệm như thế nào?
2. node_exporter chạy để làm gì và mặc định listen port nào?
3. `up{job="node"} == 0` thực sự chứng minh điều gì, và **không** chứng minh điều gì?
4. Vì sao CPU utilization không nên lấy trực tiếp `node_cpu_seconds_total`?
5. `rate()` thường được dùng với Counter hay Gauge?
6. `job` và `instance` khác nhau như thế nào?
7. Alert `for: 5m` chuyển qua các state nào?
8. Tại sao một alert đang `Pending` chưa được coi là `Firing`?
9. Prometheus rule đang `Firing` nhưng không có email/webhook notification: bạn sẽ kiểm tra layer nào tiếp theo?
10. Vì sao `up{job="node"} == 1` không có nghĩa Tomcat application healthy?
11. Khi edit `prometheus.yml`, vì sao phải chạy `promtool check config` trước khi restart/reload?
12. Khi edit rule file, command nào validate?
13. Grafana hiện No Data nhưng Prometheus query vẫn có data: scope điều tra nên chuyển sang đâu?
14. Nếu node_exporter hoạt động local nhưng Prometheus remote không scrape được, bạn sẽ kiểm tra những layer nào?
15. Một evidence pack tốt cho Day 17 cần chứng minh chuỗi nào từ failure tới recovery?

---

# 69. Independent Practical Challenge

Sau khi hoàn thành guided lab, hãy tự làm challenge này mà **không nhìn lại phần solution**:

```text
Requirement

Host:
  Linux VM

Monitoring:
  node_exporter
  Prometheus
  Grafana
  Alertmanager

Dashboard:
  Node availability
  CPU %
  Memory %
  Filesystem %
  Network RX/TX
  Host uptime

Alert:
  NodeExporterDown
  for: 3m
  severity: critical

Failure:
  Stop node_exporter

Evidence:
  Target UP before failure
  up=0 after failure
  Alert Pending
  Alert Firing
  Successful notification
  Service restored
  up=1
  Resolved notification
```

Constraint:

```text
Không restart toàn bộ stack.
Không disable firewall.
Không dùng root cho node_exporter.
Không bỏ qua config validation.
Không sửa threshold chỉ để ép alert nếu có thể tạo failure an toàn.
```

---

# 70. Mastery Check Day 17

Bạn chỉ nên tự đánh giá **Day 17 practical complete** khi bạn thực sự đã chứng minh được:

```text
node_exporter exposes Linux metrics
        ✓

Prometheus scrapes target
        ✓

PromQL produces correct signals
        ✓

Grafana queries Prometheus
        ✓

Dashboard reflects system state
        ✓

Prometheus alert rule loads successfully
        ✓

Healthy state = Inactive
        ✓

Injected failure changes up 1 → 0
        ✓

Alert transitions Pending → Firing
        ✓

Alertmanager receives alert
        ✓

Notification receiver gets message
        ✓

node_exporter restored
        ✓

up returns to 1
        ✓

Alert resolves
        ✓

Evidence preserved
        ✓
```

Nếu mới đọc và hiểu nội dung trên nhưng chưa tự chạy lab, trạng thái hiện tại là **theory learned, practical mastery chưa được chứng minh**. Đó là tiêu chuẩn đúng cho chương trình này.

## Tóm tắt Day 17

Mental model quan trọng nhất hôm nay là:

```text
Exporter exposes
Prometheus collects
PromQL interprets
Grafana visualizes
Rule detects
Alertmanager routes
Operator investigates
Evidence proves
```

Và operational rule quan trọng nhất:

```text
Alert ≠ root cause.

Alert chỉ là signal bắt đầu quá trình investigation.
```

Prometheus/node_exporter cho bạn evidence ở host/metrics layer; Grafana giúp nhìn pattern; alerting đưa sự thay đổi đến operator. Việc của administrator vẫn là xác định failing layer, thu thập evidence, thực hiện remediation trong phạm vi L1, xác nhận recovery và escalate khi vượt authority.

Nếu bạn thực hành Day 17 ngay sau phần lý thuyết này, hãy bắt đầu từ **Lab Step 0 → Step 5 → Failure Injection → Recovery**, và lưu toàn bộ evidence như hướng dẫn.
