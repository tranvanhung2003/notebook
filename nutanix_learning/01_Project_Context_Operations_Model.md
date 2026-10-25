> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# Bối cảnh dự án, mô hình vận hành và cách tư duy khi làm production

## 1. Scope kỹ thuật thực tế
**[Nguồn dự án]** Phạm vi được mô tả như một team vận hành HCI/virtualization, backup/DR, physical server, network/LB/FW, centralized logging, AWS và incident/config change. Nhiều nghiệp vụ được phân loại là semi-standard/non-standard, nghĩa là không thể chỉ học “bấm nút theo procedure”.

Các nhóm công việc nổi bật:
- Nutanix: VM, Volume Group, Image, AsyncDR, capacity, alert, upgrade AOS/AHV/Prism, script/config change.
- HYCU: policy, assignment cho VM/VG, ad-hoc backup, restore, upgrade, log/audit.
- BIG-IP: add/change/delete LB settings, SSL certificate, config management, OS upgrade.
- FortiGate: add/change/delete policy/object, config management, OS upgrade.
- Cisco: Nexus/Catalyst upgrade, read-only check, TAC-PAC collection.
- Analyzer: restart health check, log search, license/update/settings change.
- AWS: S3 bucket add/change/delete, SecurityHub/Health check, AWS-related incident changes.
- Common: inquiry, alert, incident, privileged-ID request, file transfer, audit evidence, daily CSV/Excel processing.

## 2. Bốn kiểu học/transition
### 2.1 Production shadowing
Dùng cho task có tần suất đủ cao. Flow tốt:
1. Đọc SOW/procedure và recent ticket.
2. Tạo QA list.
3. BP thao tác case thật, FPT quan sát và đối chiếu procedure.
4. Cập nhật procedure nếu có gap.
5. FPT tự thao tác dưới giám sát.
6. Chuyển sang pre-operation.

### 2.2 Simulation
Dùng khi case production không chắc xuất hiện trong thời gian transition: AsyncDR, Volume Group, S3, image… Nên dựng test case có **input → expected state → failure case → rollback** thay vì chỉ demo happy path.

### 2.3 Version upgrade
Các upgrade như HYCU, Analyzer, AOS/Prism, Cisco, BIG-IP, FortiGate có chu kỳ thấp. Học bằng bộ hồ sơ của lần trước: plan, release-note assessment, procedure, time chart, result report; nếu có case thật thì tham gia OJT cả planning và production window.

### 2.4 Inquiry/incident/config change do issue
Không thể “học procedure cố định”. Cần học theo case:
- Dấu hiệu nào được nhìn đầu tiên?
- Dữ liệu nào được thu thập?
- Tiêu chí nào để kết luận?
- Ai được escalate, khi nào?
- Change nào là temporary mitigation, change nào là permanent fix?

## 3. Change lifecycle chuẩn
Một change production nên có tối thiểu:
1. **Request**: business reason, scope, owner, deadline.
2. **Impact analysis**: component, dependency, traffic/data path, downtime.
3. **Current-state capture**: config export/show command/screenshot.
4. **Implementation plan**: từng bước, người thực hiện, người kiểm tra.
5. **Pre-check**: health, replication/backup, capacity, alarms, session/traffic.
6. **Rollback plan**: điều kiện trigger rollback và thao tác cụ thể.
7. **Approval**: ticket/change/maintenance window.
8. **Execution**: ghi timestamp cho từng step.
9. **Post-check**: functional + technical + monitoring.
10. **Evidence**: log, output, diff, screenshot, ticket update.
11. **Config/CMDB/list update**.
12. **Close/lesson learned**.

## 4. Freeze period
**[Nguồn dự án]** File freeze định nghĩa freeze quanh financial closing; kế hoạch KT mới tránh bắt đầu các production-change task HYCU/LB/FW/VM trong những khoảng freeze đã chỉ định. Đây là dấu hiệu quy trình dự án coi business calendar là một control bắt buộc.

Thực hành:
- Trước khi lên lịch change, kiểm tra freeze calendar và NW workday.
- Phân biệt “read-only validation” và “configuration change”. Có tổ chức cho phép kiểm tra trong freeze nhưng cấm change.
- Emergency change phải có route phê duyệt riêng; không suy đoán rằng “khẩn cấp” = được bypass.

## 5. Mô hình incident
### 5.1 5 câu hỏi đầu tiên
1. What changed recently?
2. Scope: một VM, một app, một VLAN hay toàn cluster?
3. Khi nào bắt đầu? Có mốc log/alert chính xác không?
4. Symptom là control-plane hay data-plane?
5. Có workaround an toàn nào không?

### 5.2 Evidence pack tối thiểu
- Ticket ID và timeline.
- Alarm/event chính.
- Health status trước/sau.
- Config diff/recent change.
- Logs theo cùng timezone.
- Topology/traffic path.
- Command output read-only.
- Vendor case ID nếu escalate.

## 6. Night/holiday work
**[Nguồn dự án]** LB/FW có workday định kỳ sau 17:00; LB change có thể có case sau 23:00; VM change đôi khi ngoài giờ/holiday. Vì vậy cần luyện “night-operation discipline”:
- Chuẩn bị tài liệu và command ban ngày.
- Peer-review command/diff trước window.
- Không chỉnh live command improvisation trừ khi incident commander cho phép.
- Có contact/escalation list sẵn.
- Đặt stop condition: quá thời gian X, health fail, traffic error > threshold → rollback/escalate.
- Sau ca đêm vẫn phải hoàn tất evidence/ticket, không để “mai nhớ rồi update”.

## 7. Quyền và least privilege
**[Nguồn dự án]** File quyền tách read/operation/admin cho nhiều sản phẩm: BIG-IP, FortiGate, HYCU, Prism, iLO, Analyzer, AWS. Một số nghiệp vụ dùng privileged ID được cấp theo quy trình.

Khi học, luôn ghi rõ mỗi thao tác cần:
- Read-only role?
- Operator role?
- Admin/privileged ID?
- SSH/CLI hay WebUI?
- Production hay STG?
- Account cá nhân hay shared/privileged checkout?

## 8. Checklist trước khi bạn tự làm một task
Bạn chỉ nên nhận “main operator” khi trả lời được:
- Input lấy từ đâu, validate thế nào?
- Object nào sẽ thay đổi?
- Dependency nào có thể bị ảnh hưởng?
- Lệnh/read-only check nào xác nhận current state?
- Expected state sau change là gì?
- Rollback có thật sự đảo ngược được không?
- Evidence nào bắt buộc?
- Ai approve, ai verify, ai escalate?
- Freeze/workday có cho phép không?
