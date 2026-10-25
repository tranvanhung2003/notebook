> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# Checklist sẵn sàng trước KT / pre-operation

## A. Kiến thức nền bắt buộc
- [ ] Phân biệt Prism Central / Prism Element / AOS / AHV / CVM.
- [ ] Giải thích VM/VG/Image/Snapshot/AsyncDR.
- [ ] Giải thích RPO/RTO/retention/restore test.
- [ ] Giải thích S3 Versioning/Object Lock/KMS/lifecycle.
- [ ] Giải thích BIG-IP Virtual Server/Pool/Member/Monitor/SNAT/SSL.
- [ ] Giải thích FortiGate VDOM/policy/address/service/NAT/session.
- [ ] Giải thích VLAN/trunk/STP/port-channel/route.
- [ ] Giải thích iLO/IML/firmware.
- [ ] Giải thích centralized log ingestion pipeline.

## B. Kỹ năng vận hành
- [ ] Có thể viết pre-check/post-check/rollback.
- [ ] Biết phân biệt read-only vs change command.
- [ ] Biết thu evidence có timestamp/timezone.
- [ ] Biết cập nhật ServiceNow ticket rõ ràng.
- [ ] Biết kiểm tra freeze/maintenance window.
- [ ] Biết escalation path và stop condition.

## C. Nutanix
- [ ] Tạo/modify/delete VM trong lab.
- [ ] Tạo image/snapshot trong lab.
- [ ] Hiểu VG attachment.
- [ ] Biết đọc cluster health/alerts/capacity.
- [ ] Biết mục đích ncli/acli và giới hạn CVM shell.
- [ ] Viết upgrade pre/post checklist.

## D. HYCU
- [ ] Hiểu policy/assignment/job/recovery point.
- [ ] Chạy ad-hoc backup lab.
- [ ] Restore alternate target lab.
- [ ] Troubleshoot permission/target failure.

## E. Network
- [ ] BIG-IP read-only inventory + traffic path.
- [ ] SSL chain/expiry/SAN check.
- [ ] FortiGate policy search/read-only CLI.
- [ ] Cisco show VLAN/trunk/STP/route.
- [ ] Biết khi nào session/return path gây nhầm.

## F. OS
- [ ] RHEL: systemctl/journalctl/nmcli/LVM/SELinux.
- [ ] Windows: PowerShell/EventLog/service/network/storage/time.
- [ ] Biết correlate guest issue với hypervisor/platform issue.

## G. Tool riêng/automation
- [ ] Đọc `.xlsm` module và identify input/output.
- [ ] Python: csv/re/json/logging/pathlib.
- [ ] Batch: quoting/errorlevel.
- [ ] Xử lý UTF-8/UTF-8 BOM/CP932.
- [ ] Viết golden-file test.
- [ ] Dùng API GET trước PATCH/POST.
- [ ] Không hard-code secret.

## H. Câu hỏi phải lấy được câu trả lời trong KT
- [ ] Version/model thực tế từng product.
- [ ] Prod/STG/PoC topology.
- [ ] Account/role/privileged-ID flow.
- [ ] Tên procedure mới nhất và storage location.
- [ ] ServiceNow workflow/approval.
- [ ] Freeze/NW workday rule.
- [ ] Evidence bắt buộc.
- [ ] Escalation/vendor support contact.
- [ ] Rollback/abort criteria.
- [ ] Known issues và workaround hiện tại.

## I. Mức tự đánh giá mỗi nghiệp vụ
Chấm 0/1/2 cho từng mục: Overview, Input, Permission, Pre-check, Steps, Judgment, Rollback, Post-check, Evidence, Escalation.

- **0**: chưa biết.
- **1**: làm được khi có người hướng dẫn.
- **2**: tự làm và giải thích được vì sao.

Mục tiêu trước pre-operation: không có điểm 0; các mục Permission/Rollback/Post-check/Escalation phải đạt 2.

## J. Ưu tiên theo lịch transition nguồn
**[Nguồn dự án]** Common session mới được sắp ServiceNow (12/7), monitoring/AccessCheck (12/8), Nutanix/virtual platform (12/9–12/11), HYCU/LB (12/14), FW/AWS (12/15), sau đó KT. Vì vậy nếu thời gian học hạn chế, ưu tiên theo đúng chuỗi dependency này.
