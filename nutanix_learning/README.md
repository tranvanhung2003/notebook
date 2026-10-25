> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# Bộ tài liệu học và chuẩn bị dự án Nutanix (HCNET)

## 1. Mục tiêu của bộ tài liệu
Bộ Markdown này được viết cho người đã có nền tảng IT nhưng chưa có kinh nghiệm trực tiếp với stack của dự án. Mục tiêu không phải học thuộc giao diện, mà là hình thành **mental model**, biết **đọc request → đánh giá rủi ro → chuẩn bị thay đổi → thao tác → kiểm chứng → rollback/escalate → lưu evidence**.

## 2. Những gì nguồn dự án cho biết
**[Nguồn dự án]** Team Nutanix/HCNET bao phủ Nutanix HCI/Prism, HYCU backup, HPE/iLO, BIG-IP, FortiGate, Cisco Nexus/Catalyst, EventLog Analyzer, Firewall Analyzer, AWS và các nghiệp vụ inquiry/incident/configuration change. Các tool thực tế được nhắc đến gồm Windows, Excel, TeraTerm, WinMerge, WinSCP, Visual Studio Code/Python, ServiceNow, SVN, AWS Web Console, Prism Central WebUI, HYCU WebUI, BIG-IP WebUI, FortiGate WebUI, iLO, JP1 và các macro/tool riêng.

Nguồn kế hoạch cuối mô tả **89 nghiệp vụ**; mô hình học gồm production shadowing, simulation, version-upgrade walkthrough/OJT và case study cho inquiry/incident. Lịch mới bố trí common session theo thứ tự: ServiceNow → AccessCheck/monitoring → Nutanix/virtual platform → HYCU/LB → FW/AWS, sau đó KT và pre-operation. Điều này gợi ý thứ tự học nên ưu tiên khả năng vận hành chung trước khi đi sâu vào upgrade hiếm gặp.

Nguồn cũng cho thấy công việc không chỉ là thao tác WebUI: có CLI, Python script, `.xlsm`, batch, CSV, log, API-related tooling, config backup, ticket/evidence và các thay đổi ban đêm. Tài liệu kế hoạch còn ghi có **9 vendor-specific tools** cho lĩnh vực HCNET/Nutanix, nhưng trong nguồn hiện có không có một sheet riêng liệt kê đầy đủ chính xác 9 tên tool; vì vậy không nên tự suy đoán danh sách chính thức.

## 3. Thứ tự học khuyến nghị theo dự án
1. `01_Project_Context_Operations_Model.md` – hiểu scope, change/incident, freeze, KT và cách làm việc an toàn.
2. `11_ServiceNow_Operations_Maintenance.md` – vì ticket/approval/evidence xuất hiện xuyên suốt.
3. `04_Nutanix_Prism_AOS_AHV.md` + `12_Nutanix_Operations_Maintenance.md` – nền tảng trung tâm của dự án.
4. `05_HYCU_Backup.md` + `02_AWS_S3_Backup_DR.md` – backup/restore/DR.
5. `08_BIGIP_Load_Balancer.md` + `07_FortiGate_Firewall.md` – change định kỳ, có thể ngoài giờ.
6. `09_Cisco_Nexus_Catalyst.md` + `10_EventLog_Firewall_Analyzer.md` – network/logging.
7. `06_HPE_ProLiant_iLO.md` – physical node và hardware health.
8. `03_RHEL.md` + `13_Windows_Server.md` – OS foundation dùng khi provisioning/troubleshooting.
9. `14_Custom_Tool_Maintenance.md` + `15_Automation_API_CLI_PowerShell_MAPI.md` – tool riêng và automation.
10. `99_Project_Readiness_Checklist.md` – tự đánh giá trước KT/pre-operation.

## 4. Nguồn đã đọc
- `Nutanix (HCNET)_Overview.xlsx`: Overview, Tool-OS-URL, SOW_VN, Skill set, Scope.
- `Nutanix_HCNET_アカウント権限.xlsx`: quyền theo sản phẩm/nghiệp vụ và ghi chú cách suy ra quyền.
- `Nutanix_新計画スケジュール_v1.1.xlsx`: lịch KT mới, nhóm tần suất, phương thức học, 89 nghiệp vụ, quyền quản trị cần xem xét.
- `システム凍結期間予定日程.xlsx`: nguyên tắc/system freeze calendar.
- `作業スケジュール_202601~.xlsx`: lịch công việc thực tế, có RHEL, AOS/AHV, HYCU, LB/FW và nhiều change cụ thể.
- `移管計画_Nutanix_20261023.pptx`: phương thức transition, task chuẩn bị, case-study, version-upgrade và lịch tổng.

## 5. Quy ước ID cần chú ý
Các file nguồn sử dụng cả prefix `6-0-*` và `6-1-*` cho các nghiệp vụ có tên tương ứng. Khi làm dự án, **không nên tự động coi hai ID là cùng một record chỉ dựa vào số**; hãy match theo tên nghiệp vụ, tài liệu procedure và xác nhận với owner/BP trong KT.

## 6. Nguyên tắc vàng khi học stack này
- Tách **read-only validation** và **write/change**; luôn học lệnh/GUI đọc trước.
- Không thực hiện production change nếu chưa biết pre-check, expected result, rollback và approval.
- Backup không có giá trị nếu restore chưa được test.
- “Có snapshot” không đồng nghĩa “có backup”; “có backup” không đồng nghĩa “đáp ứng DR”.
- Với network/LB/FW, phải hiểu **traffic path** trước khi chỉnh object/policy.
- Với Nutanix, phải phân biệt rõ **Prism Central, Prism Element, AOS, AHV, CVM**.
- Với incident, ưu tiên **time line + evidence + scope + recent change + blast radius**, không “thử lệnh” ngẫu nhiên.
- Với tool riêng, ưu tiên **reproducibility, input validation, encoding, error handling, logs, test fixtures và rollback**.

## 7. Những thứ nguồn chưa xác định
Nguồn không cung cấp đầy đủ version/model thực tế cho tất cả sản phẩm, topology chi tiết, tên cluster/VDOM/VLAN, endpoint API, certificate PKI flow, ServiceNow table/workflow cụ thể, hay credential/role name chính thức. Tất cả ví dụ trong tài liệu là mẫu học tập; khi vào KT phải thay bằng dữ liệu đã được xác nhận.
