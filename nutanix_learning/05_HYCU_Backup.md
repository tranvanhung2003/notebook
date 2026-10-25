> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# HYCU Backup – vận hành, restore và troubleshooting

## 1. Liên hệ với dự án
**[Nguồn dự án]** HYCU có nhiều task tần suất cao: thêm/chỉnh policy, assign policy cho VM/VG, ad-hoc backup, restore, kiểm tra policy định kỳ, upgrade và incident-driven config change. File quyền tách role read, operation restore/ad-hoc backup và admin policy/upgrade/target. Đây là sản phẩm cần học rất thực dụng.

## 2. Mental model backup
Một hệ backup có các thành phần logic:
- **Source**: VM/VG/workload.
- **Policy**: schedule, retention, backup type, target.
- **Target**: repository/object storage.
- **Job**: execution instance.
- **Recovery point**: restore point.
- **Catalog/metadata**: dữ liệu để tìm và restore.

Nếu chỉ nhìn “job success” mà không kiểm tra recovery point/restore thì chưa đủ.

## 3. RPO/RTO
- RPO = mức dữ liệu có thể mất.
- RTO = thời gian khôi phục dịch vụ.
Policy phải được map từ business requirement chứ không phải “daily là đủ”.

## 4. Policy assignment
Trước assign/change:
- Xác nhận VM/VG đúng object và environment.
- Kiểm tra policy hiện tại.
- Xác nhận schedule overlap.
- Retention/target.
- Application quiescing requirement.
- Capacity/network impact.
- Freeze/change approval.

Sau change:
- Policy mapping đúng.
- Next run time.
- Không có duplicate/unintended policy.
- Job đầu tiên success.

## 5. Ad-hoc backup
Dùng trước change quan trọng hoặc theo request. Cần ghi:
- lý do,
- object,
- timestamp,
- recovery point ID,
- expiration/retention,
- result.

Không gọi ad-hoc backup xong rồi change ngay nếu backup vẫn đang chạy hoặc chưa verify recovery point.

## 6. Restore – tư duy an toàn
### Trước restore
- Restore cái gì? VM full, disk, file hay VG?
- Restore in-place hay alternate location?
- Có overwrite dữ liệu hiện tại không?
- Network của restored VM có thể gây IP conflict không?
- Credential/encryption/key có sẵn không?

### Sau restore
- Boot/application health.
- Data consistency.
- Network identity.
- Monitoring.
- Backup policy re-attachment.
- Evidence và owner acceptance.

## 7. “Backup succeeded nhưng restore failed” – nguyên nhân điển hình
- Catalog/metadata issue.
- Target unreachable.
- S3/IAM/KMS permission changed.
- Recovery point corrupt/incomplete.
- Network/DNS/TLS issue.
- Version compatibility.
- Insufficient capacity ở restore destination.

## 8. Job troubleshooting workflow
1. Capture job ID/time/source/policy.
2. Determine stage failed: snapshot → transfer → catalog → retention cleanup.
3. Check Nutanix alert/storage health.
4. Check target connectivity/capacity/permission.
5. Check HYCU event/log.
6. Compare recent config changes.
7. Retry chỉ khi đã hiểu failure mode hoặc procedure cho phép.

## 9. S3 integration
**[Nguồn dự án]** AWS S3 bucket operations liên quan HYCU admin permission. Khi S3 là target, troubleshoot cả hai phía:
- HYCU target config.
- AWS bucket policy/IAM.
- KMS.
- DNS/network/proxy.
- lifecycle/immutability.

## 10. Upgrade HYCU
- Release notes/compatibility với AOS/AHV/PC.
- Backup configuration/catalog backup nếu vendor quy định.
- Ensure no running critical jobs.
- Validate target connectivity.
- Upgrade STG trước nếu có.
- Post-check: UI/service/version, source discovery, policy, test backup, test restore representative.

**[Nguồn dự án]** HYCU version upgrade thuộc nhóm low-frequency version-upgrade, học bằng past case + OJT nếu có.

## 11. Audit/evidence
Nguồn có task lấy access log từ HYCU/ELA. Khi vận hành backup, nên lưu:
- Who changed policy.
- Job result.
- Restore request/approval.
- Access log nếu audit yêu cầu.
- Before/after screenshot/config export.

## 12. Lab
- Tạo 2 policy với RPO khác nhau.
- Assign policy cho VM test, chạy ad-hoc backup.
- Restore alternate VM để tránh overwrite.
- Simulate target permission deny và đọc job failure.
- Viết monthly policy review checklist.

## 13. Câu hỏi KT
- HYCU version/architecture hiện tại?
- Target là S3 hay repository nào, prod/STG khác nhau ra sao?
- Policy naming/retention standard?
- VG restore khác VM restore thế nào?
- Restore acceptance do team nào xác nhận?
- Khi job fail bao lâu thì escalate?
- Có immutable copy không?
