> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# Kinh nghiệm vận hành và bảo trì product trong lĩnh vực Nutanix

## 1. Mục tiêu
Tài liệu này không lặp lại lý thuyết Prism/AOS/AHV, mà tập trung vào **cách làm việc như một operator/maintainer** trong môi trường Nutanix enterprise.

## 2. Daily/weekly/monthly rhythm
### Daily
- Critical alerts.
- Cluster/node/CVM availability.
- Failed backup/restore.
- Major capacity anomaly.
- Monitoring/log ingestion.
- Open incidents/changes.

### Weekly
- Backup policy/exceptions.
- Repeated alerts.
- Capacity trend.
- Privileged-ID/access review theo policy.
- Pending change/upgrade actions.

### Monthly/periodic
- Capacity report.
- Snapshot inventory.
- Access/audit logs.
- Analyzer restart health.
- Lifecycle/firmware/version review.
- Restore drill hoặc sample restore theo kế hoạch.

**[Nguồn dự án]** Các task tương ứng xuất hiện trong SOW: capacity, snapshot inventory, CVM memory check, HYCU policy checks, audit log collection, monthly Analyzer post-restart checks.

## 3. Tư duy “platform, not just VM”
Khi request “tạo VM”, operator phải nhìn cả:
- cluster capacity,
- network/VLAN,
- image,
- storage/container,
- backup policy,
- monitoring,
- security/owner,
- CMDB/ticket.

## 4. Standard change vs issue-driven change
### Standard/semi-standard
Có procedure và parameter. Nguy cơ chính là input sai/nhầm environment.

### Issue-driven
**[Nguồn dự án]** Nutanix issue-driven config change được xếp non-standard/high difficulty. Cần hypothesis-driven troubleshooting:
1. Problem statement.
2. Evidence.
3. Candidate root causes.
4. Lowest-risk test.
5. Change proposal.
6. Observation.
7. Rollback/next branch.

## 5. Upgrade experience
Một maintainer tốt không chỉ “chạy LCM”. Họ phải:
- đọc release notes,
- hiểu interoperability,
- dự đoán service impact,
- xác định stop/abort criteria,
- chuẩn bị vendor support,
- chạy health precheck,
- theo dõi node-by-node,
- post-check toàn stack.

## 6. Capacity
Không quản capacity bằng % đơn giản. Theo dõi:
- physical vs usable,
- growth rate,
- failure headroom,
- snapshot/replication overhead,
- hot workload,
- forecast tới procurement lead time.

## 7. Alert management
Phân loại:
- symptom alert,
- root-cause alert,
- transient/maintenance alert,
- stale/resolved alert.

**[Nguồn dự án]** Có nghiệp vụ xóa alert không cần thiết; phải có rule rõ khi nào được delete, không che root cause.

## 8. Vendor escalation
Một case gửi vendor nên có:
- serial/cluster ID/version,
- topology,
- exact timestamp/timezone,
- recent changes,
- symptoms,
- logs/support bundle,
- commands already run,
- business impact,
- maintenance constraints.

## 9. Runbook quality
Runbook tốt có:
- purpose/scope,
- prerequisites/permissions,
- input validation,
- pre-check,
- steps,
- expected output từng bước,
- failure branch,
- rollback,
- post-check,
- evidence,
- escalation.

Không viết “check OK” mà không định nghĩa OK.

## 10. Pre-operation scoring
Tự chấm mỗi task 0–2:
- 0: chưa hiểu.
- 1: hiểu nhưng cần hướng dẫn.
- 2: có thể tự thực hiện và giải thích risk/rollback.

Chỉ chuyển main operator khi các mục critical (permission, precheck, rollback, postcheck, escalation) đều đạt 2.

## 11. KT questions phải hỏi sâu
- “Nếu bước này fail thì dấu hiệu là gì?”
- “Trường hợp nào procedure không áp dụng?”
- “Lần gần nhất rollback là khi nào?”
- “Có known issue nào với version hiện tại?”
- “Output nào là bằng chứng bắt buộc?”
- “Ai có quyền quyết định continue/abort?”
