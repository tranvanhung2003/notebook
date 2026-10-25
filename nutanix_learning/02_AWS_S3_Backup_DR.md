> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# AWS S3 và S3 cho Backup/DR

## 1. Liên hệ với dự án
**[Nguồn dự án]** AWS/S3 xuất hiện ở các nghiệp vụ thêm/thay đổi/xóa S3 bucket và AWS configuration change do issue. File quyền yêu cầu AWS Management Console admin cho bucket create/change/delete, read quyền cho SecurityHub/Health Dashboard, và cho thấy các task S3 có liên quan đến HYCU privilege. Kế hoạch transition xếp S3 vào nhóm phù hợp với simulation vì có thể không phát sinh case production trong giai đoạn KT.

**Điểm cần tránh nhầm:** nguồn nói S3 liên quan data transfer/backup-DR, nhưng không cung cấp đầy đủ kiến trúc target, region, class, encryption key hay immutability policy. Không được tự giả định “bucket này là backup repository chính” nếu KT chưa xác nhận.

## 2. Mental model S3
S3 là object storage. Một object được nhận diện bởi:
- Bucket.
- Object key, ví dụ `backup/prod/vm01/2026-10-25.img`.
- Version ID nếu bật Versioning.
- Metadata/tags.

Không có thư mục thật; dấu `/` trong key chỉ tạo cảm giác folder.

### 2.1 Consistency và durability
S3 cung cấp strong read-after-write consistency cho object operations. Durability rất cao không có nghĩa dữ liệu “không thể mất”: xóa nhầm, credential compromise, lifecycle sai, ransomware logic hoặc KMS key issue vẫn có thể làm backup vô dụng.

## 3. Những khái niệm bắt buộc
- **Bucket policy**: resource-based policy.
- **IAM policy**: identity-based policy.
- **Block Public Access**: lớp bảo vệ chống public exposure.
- **Versioning**: giữ nhiều version của object.
- **Object Lock**: WORM/immutability, cần thiết kế từ đầu và có retention governance/compliance.
- **SSE-S3 / SSE-KMS**: server-side encryption; KMS cho granular key control/audit.
- **Lifecycle**: transition class và expiration.
- **Storage class**: Standard, IA, One Zone-IA, Intelligent-Tiering, Glacier classes… lựa chọn theo access pattern và restore requirement.
- **Replication**: SRR/CRR để copy object sang bucket khác/region khác; không thay thế độc lập cho mọi loại backup.

## 4. S3 cho backup: thiết kế đúng câu hỏi
Trước khi chọn config, xác định:
- RPO: tối đa mất bao nhiêu dữ liệu?
- RTO: phải restore trong bao lâu?
- Retention: giữ 7 ngày, 35 ngày, 1 năm hay 7 năm?
- Immutability: backup có cần chống xóa/sửa không?
- Region: có yêu cầu DR region khác không?
- Encryption/key ownership: ai quản lý KMS key?
- Egress/restore cost: full restore có tốn thời gian/chi phí không?
- Backup software compatibility: HYCU hỗ trợ target/options nào trong version thực tế?

## 5. Pattern backup/DR phổ biến
### Pattern A: backup software → S3
Backup product tạo object vào S3. Kiểm soát chính:
- Dedicated bucket/prefix.
- IAM role chỉ đủ quyền.
- Encryption.
- Versioning/Object Lock nếu solution hỗ trợ.
- Lifecycle phù hợp.
- Monitoring failed uploads.
- Restore test định kỳ.

### Pattern B: primary S3 + cross-region replication
Dùng cho DR của object workload. Cần chú ý:
- Replication phải enable và permission đúng.
- Existing objects có thể cần batch replication riêng.
- Delete marker/metadata/replication behavior phải hiểu rõ.
- KMS encrypted objects cần key policy/replication permission.

### Pattern C: backup copy sang account khác
Tăng isolation trước credential compromise. Account đích nên có trust tối thiểu và retention độc lập.

## 6. Tạo bucket – checklist vận hành
1. Xác nhận account/region/tên bucket.
2. Xác nhận Block Public Access.
3. Xác nhận Versioning.
4. Xác nhận encryption và KMS key.
5. Xác nhận Object Lock/retention nếu có.
6. Tạo bucket policy/IAM least privilege.
7. Tag owner/system/environment/data-classification.
8. Cấu hình lifecycle/replication nếu được thiết kế.
9. Test read/write bằng account/role thực tế của backup application.
10. Test restore/list/read; không chỉ test upload.
11. Lưu evidence và update inventory/ticket.

## 7. Thay đổi bucket – rủi ro cao
Các thay đổi dễ gây outage/data-loss:
- Deny trong bucket policy chặn luôn backup role.
- KMS key policy thiếu role.
- Lifecycle expiration quá sớm.
- Tắt versioning/replication mà không hiểu ảnh hưởng.
- Object ownership/ACL mismatch.
- Xóa bucket/prefix nhầm.

Luôn diff policy trước/sau. Với JSON, format và review logic `Effect`, `Principal`, `Action`, `Resource`, `Condition`.

## 8. Security
### 8.1 Least privilege mẫu tư duy
Backup writer thường cần subset như:
- list bucket/prefix,
- put/get object,
- multipart upload actions,
- có thể delete tùy product,
- KMS Encrypt/Decrypt/GenerateDataKey.

Không copy policy `s3:*`/`*` chỉ để “cho chạy”.

### 8.2 KMS failure
Backup có thể ghi được nhưng restore fail nếu:
- key disabled/deleted,
- key policy đổi,
- role mất decrypt,
- DR account/region không access key.

Vì vậy restore test phải bao gồm đường KMS thực tế.

## 9. Troubleshooting S3
### `AccessDenied`
Kiểm tra theo thứ tự:
1. IAM identity policy.
2. Bucket policy.
3. SCP/permission boundary nếu tổ chức dùng.
4. KMS key policy/grant.
5. VPC endpoint policy.
6. Explicit deny.

### Backup chậm
- Multipart upload.
- Network path/Direct Connect/VPN/proxy.
- Object size/count.
- Throttling/application concurrency.
- Cross-region traffic.

### Restore từ archive class chậm
Glacier class có restore/rehydration workflow; RTO phải tính thời gian này từ lúc thiết kế.

## 10. AWS CLI ví dụ học tập
```bash
aws s3api get-bucket-versioning --bucket EXAMPLE
aws s3api get-bucket-encryption --bucket EXAMPLE
aws s3api get-public-access-block --bucket EXAMPLE
aws s3api get-bucket-policy --bucket EXAMPLE
aws s3api list-object-versions --bucket EXAMPLE --prefix backup/
```
Ưu tiên các lệnh `get/list` trong lab trước khi học `put/delete`.

## 11. Kịch bản lab
- Tạo bucket lab private, bật versioning, upload cùng key 3 lần và quan sát version.
- Xóa object và giải thích delete marker.
- Cấu hình lifecycle cho prefix lab.
- Dùng role read-only và writer khác nhau.
- Mô phỏng KMS deny rồi xác định nguyên nhân.
- Viết runbook “bucket create/change/delete” có pre/post/rollback.

## 12. Câu hỏi KT nên hỏi
- S3 dùng cho HYCU theo kiểu target nào? Có immutability không?
- Bucket/account/region production và STG là gì?
- KMS key do team nào sở hữu?
- Lifecycle/retention chuẩn?
- Có cross-account/cross-region copy không?
- Delete bucket cần approval đặc biệt nào?
- Restore test gần nhất khi nào, RTO thực tế bao nhiêu?
