> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# ServiceNow – kiến thức và kinh nghiệm vận hành/bảo trì

## 1. Liên hệ với dự án
**[Nguồn dự án]** ServiceNow/SNOW được dùng để tiếp nhận/đăng ký nhiều request, file transfer request, privileged-ID request, entry/exit procedures và monitoring suppression request. Kế hoạch common session dành riêng thời gian cho ServiceNow trước các KT kỹ thuật.

Nguồn không cho biết instance/module/table cụ thể, vì vậy phần dưới là mô hình ITSM chuẩn; field/workflow thật phải xác nhận trong KT.

## 2. Các record loại thường gặp
- **Incident**: dịch vụ bị gián đoạn/suy giảm.
- **Request / RITM / Catalog Task**: yêu cầu cung cấp/thực hiện dịch vụ.
- **Change**: thay đổi có kiểm soát.
- **Problem**: root cause/recurring issue.
- **Knowledge**: bài hướng dẫn/known error.
- **CMDB CI**: configuration item.

## 3. Ticket hygiene
Một ticket tốt phải trả lời:
- Người yêu cầu là ai?
- Service/system/env nào?
- What/why/when?
- Input parameter đầy đủ chưa?
- Approval đã đủ chưa?
- Evidence ở đâu?
- Owner và status hiện tại?
- Next action/deadline?

## 4. Lifecycle thao tác
### Request/change
1. Intake.
2. Validate required fields.
3. Scope/impact/risk.
4. Approval.
5. Schedule/freeze check.
6. Execute.
7. Evidence/post-check.
8. Update related inventory/CMDB/list.
9. Resolve/close.

### Incident
1. Acknowledge.
2. Categorize/priority.
3. Initial diagnosis.
4. Assign/escalate.
5. Work notes timeline.
6. Restore service.
7. Resolution notes/root cause link.

## 5. Work notes vs customer-visible comments
Tùy instance, internal work notes và comments có audience khác nhau. Không đưa credential/secret/raw sensitive log vào field không phù hợp.

## 6. Attachment/evidence discipline
Tên file nên có ticket/time/system. Ví dụ:
`INC0012345_PC_clusterA_precheck_20261025.txt`

Evidence nên tái tạo được quyết định: output command + timestamp + operator, không chỉ screenshot cắt mất context.

## 7. Filter/list
**[Nguồn dự án]** Transition plan yêu cầu lưu memo filter ServiceNow để thu thập past case. Bạn nên học:
- AND/OR filter.
- Date ranges.
- Assignment group.
- Category/subcategory.
- State.
- Short description contains.
- Export chỉ khi policy cho phép.

## 8. Case study mining
Khi đọc ticket cũ, trích 8 trường:
1. Trigger.
2. Symptom/request.
3. Affected CI.
4. Data checked.
5. Decision.
6. Change/command.
7. Escalation.
8. Result/lesson.

Đây là cách biến history thành runbook/judgment flow.

## 9. SLA/priority
Priority thường dựa impact × urgency nhưng matrix thực tế tùy tổ chức. Không tự đổi priority để “đẩy nhanh”. Hiểu response SLA vs resolution SLA.

## 10. Common mistakes
- Resolve nhưng thiếu evidence.
- Close ticket trước post-check.
- Work notes không có timestamp/decision.
- Không link change với incident/problem.
- Copy paste config chứa secret.
- Dùng request cũ làm template nhưng quên đổi environment/IP/date.

## 11. Kỹ năng cần luyện trong lab
- Tạo giả lập incident/change form bằng spreadsheet.
- Viết 5 mẫu work note: start, pre-check, change, post-check, rollback.
- Từ 10 ticket giả lập, xây judgment flow.
- Viết filter logic để tìm “Nutanix incident trong 12 tháng gần nhất”.

## 12. Câu hỏi KT
- Module/table nào được dùng cho request LB/FW/HYCU/Nutanix?
- Field nào bắt buộc?
- Approval chain?
- Change vs request boundary?
- Evidence retention?
- Assignment group/escalation?
- Monitoring suppression dùng catalog item nào?
