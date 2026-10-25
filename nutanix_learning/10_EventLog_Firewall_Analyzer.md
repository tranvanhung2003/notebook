> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# EventLog Analyzer & Firewall Analyzer (ManageEngine)

## 1. Liên hệ với dự án
**[Nguồn dự án]** Có task version upgrade, license renewal, restart health check, log search, network-device periodic log check, communication log collection và incident-driven settings changes. Quyền được tách read vs admin.

## 2. Mental model centralized logging
Source device → transport (syslog/agent/API) → collector → parser/index/storage → search/report/alert.

Nếu “không thấy log”, phải hỏi log mất ở layer nào, không nhảy ngay vào UI filter.

## 3. EventLog Analyzer
Dùng để tập trung log server/network/application và phục vụ search/audit/alert tùy cấu hình. Trong dự án, nguồn cho thấy có check log ingestion sau scheduled restart và check network-device logs.

### Health checklist
- Service/process up.
- Collector/listener port.
- Device source reachable.
- Last received timestamp.
- Parsing status.
- Storage/db capacity.
- License/device count.
- Time sync/timezone.

## 4. Firewall Analyzer
Tập trung/phân tích firewall logs. Source có task lấy communication log theo request và restart health check.

Khi search traffic, cần ít nhất:
- Time range/timezone.
- src IP.
- dst IP.
- protocol/port.
- action allow/deny.
- device/VDOM.
- NAT IP nếu có.

## 5. Log ingestion troubleshooting
1. Device có generate log không?
2. Device gửi tới đúng collector IP/port?
3. Network/firewall cho phép transport?
4. Collector listener up?
5. Raw log có tới nhưng parser fail?
6. Index/storage full?
7. Search filter/timezone sai?

## 6. Restart post-check
**[Nguồn dự án]** Có task định kỳ kiểm tra sau restart. Checklist nên gồm:
- WebUI login.
- Core service up.
- Last log timestamp từ source quan trọng.
- Search sample.
- Alert/report scheduler.
- DB/storage capacity.
- JP1/monitoring status nếu integrated.

## 7. Version upgrade/license renewal
- Backup config/db theo vendor guidance.
- Compatibility/upgrade path.
- Free disk.
- License validity.
- Connector/device count.
- Post-upgrade ingestion/search/report test.

Không chỉ kiểm tra “UI mở được”.

## 8. Incident-driven settings change
Vì source xếp các thay đổi này là non-standard/high difficulty, procedure nên ghi:
- symptom,
- hypothesis,
- setting to change,
- expected effect,
- rollback,
- observation period.

## 9. Lab
- Dùng Linux rsyslog gửi sample syslog tới lab collector.
- Parse 5-tuples firewall log.
- Viết regex để extract timestamp/src/dst/port/action.
- Thực hành timezone normalization.
- Viết report template cho “log ingestion after restart”.
