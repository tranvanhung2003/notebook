Dưới đây là **bản đồ kiến thức đầy đủ của Day 5**, được chia đúng thành **2 module: Lý thuyết và Thực hành**. Tôi giữ phạm vi bám sát `Infrastructure_Operations_14_Day_Syllabus.md`; ở bước này tôi chỉ trình bày **toàn cảnh chi tiết những gì phải học và phải làm**, chưa đi sâu giải thích từng command/workflow. Day 5 là phần thứ hai của module **AWS Cloud Administration, Monitoring & Disaster Recovery**, nối tiếp IAM/VPC/EC2 của Day 4 và tập trung vào **storage, backup/recovery, DR, monitoring, auditing và cost operations**. Infrastructure_Operations_14_Da…

# Day 5 — AWS Storage, Backup/DR, Monitoring, Audit & Cost Operations

Mục tiêu tổng thể của Day 5 là sau khi hoàn thành, bạn không chỉ biết tạo `EBS volume` hoặc `S3 bucket`, mà phải có tư duy vận hành hoàn chỉnh:

**Provision → Protect → Backup → Restore → Verify → Monitor → Audit → Measure RPO/RTO → Control Cost → Cleanup → Preserve Evidence.**

Điểm đặc biệt quan trọng của syllabus là: **backup thành công chưa đồng nghĩa với DR readiness**. Bạn phải thực hiện **isolated restoration**, kiểm tra **data integrity**, ghi nhận **observed recovery time**, xác định dependency như IAM permission và encryption key, đồng thời lưu evidence có thể audit. Infrastructure_Operations_14_Da…

---

# MODULE 1 — LÝ THUYẾT

Module Lý thuyết có thể chia thành **8 khối kiến thức lớn**.

## 1. Amazon EBS — Block Storage cho EC2

Bạn cần học toàn bộ lifecycle cơ bản của `Amazon Elastic Block Store (EBS)`.

Phạm vi gồm:

- EBS là gì và vai trò của `block storage`.
- Quan hệ giữa `EC2 instance` và `EBS volume`.
- `EBS volume types`.
- Volume attachment/detachment.
- Device mapping giữa AWS và Linux.
- Partitioning awareness.
- Filesystem formatting.
- Mounting filesystem.
- Persistent mount.
- `/etc/fstab`.
- Volume capacity và filesystem capacity.
- EBS snapshot.
- Snapshot restore.
- Restore snapshot thành volume mới.
- Attach restored volume vào EC2.
- Recovery verification.
- Data-integrity validation.
- Backup validation.

Điểm operational quan trọng là phân biệt:

**EBS volume ≠ filesystem ≠ mount point ≠ EBS snapshot.**

Bạn cần hiểu rõ data flow:

`Application → filesystem → block device → EBS volume → snapshot → restored EBS volume → filesystem → recovered application data`.

---

## 2. Backup, Restore, RPO và RTO

Đây là nền tảng của toàn Day 5.

Bạn phải hiểu:

### RPO — Recovery Point Objective

Trả lời câu hỏi:

> Khi xảy ra disaster, doanh nghiệp chấp nhận mất tối đa bao nhiêu dữ liệu?

Ví dụ:

- Backup mỗi 24 giờ → theoretical RPO có thể lên tới khoảng 24 giờ.
- Backup mỗi 1 giờ → RPO nhỏ hơn đáng kể.

Nhưng syllabus yêu cầu bạn không chỉ nói về theoretical RPO mà còn phải xác định **observed RPO** dựa trên recovery point thực tế.

### RTO — Recovery Time Objective

Trả lời:

> Sau incident/disaster, dịch vụ phải được phục hồi trong bao lâu?

Trong lab, bạn phải ghi lại:

- recovery start time;
- recovery end time;
- total observed recovery time.

Từ đó so sánh với RTO đã khai báo.

Bạn cũng cần hiểu:

- backup success không bảo đảm restore success;
- restore success không bảo đảm application usable;
- có recovery point không đồng nghĩa có quyền truy cập recovery point;
- encrypted backup không thể phục hồi nếu encryption key cần thiết không khả dụng.

Đây là điểm được nhấn mạnh trực tiếp trong phần mở rộng của syllabus. Infrastructure_Operations_14_Da…

---

## 3. Amazon S3 — Object Storage và Backup Repository

Đây là phần lớn nhất của Day 5.

Bạn cần học kiến trúc cơ bản của `Amazon S3`:

- S3 bucket.
- Object.
- Object key.
- Prefix awareness.
- Object metadata.
- Object versions.
- Bucket-level configuration.
- Object-level access.

### Access control

Phải hiểu mối quan hệ giữa:

- IAM identity policy.
- Bucket policy.
- Resource permission.
- Explicit Allow.
- Explicit Deny.
- `Block Public Access`.
- Least privilege.

Điểm vận hành quan trọng là phải kiểm tra cả:

**identity-side permission + resource-side policy + explicit deny + encryption-key permission.**

### S3 security

Bao gồm:

- Block Public Access.
- Encryption.
- IAM access.
- Bucket policy.
- Least privilege.
- Public exposure prevention.

### Versioning

Bạn cần hiểu:

- object version;
- current version;
- previous version;
- overwrite behavior;
- delete-marker awareness;
- restore một version cụ thể.

Versioning là một **recovery mechanism**, nhưng không được đánh đồng với independent backup.

### Lifecycle management

Bao gồm:

- lifecycle rule;
- transition;
- expiration;
- noncurrent versions;
- cost implications.

### Storage classes

Cần biết purpose và trade-off giữa các loại storage class ở mức operations:

- access frequency;
- storage cost;
- retrieval behavior;
- recovery requirement;
- retention design.

### Replication

Học khái niệm:

- replication;
- destination bucket;
- recovery copy;
- dependency;
- cost implications.

Quan trọng:

> Replication không tự động đồng nghĩa với backup.

Nếu source bị overwrite/delete và replication policy replicate thao tác đó, replica cũng có thể bị ảnh hưởng tùy design.

### Object Lock awareness

Bạn phải biết vai trò của `Object Lock` trong retention/immutability scenarios, nhưng Day 5 chỉ yêu cầu awareness và optional lab khi environment hỗ trợ.

---

## 4. Phân biệt Versioning, Replication, Retention và Backup

Đây là một phần **bắt buộc** trong additive coverage.

Bạn phải có khả năng phân biệt bốn khái niệm:

**Versioning**  
Giữ các phiên bản của object.

**Replication**  
Sao chép object sang một location/bucket khác.

**Retention**  
Kiểm soát thời gian dữ liệu phải được giữ.

**Backup**  
Tạo recovery point phục vụ recovery theo backup policy.

Không được dùng các khái niệm này thay thế cho nhau.

Syllabus đặc biệt yêu cầu DR design phải xét cả:

- recovery permission;
- encryption-key availability;
- recoverable version;
- recovery dependency;
- recovery cost.

Infrastructure_Operations_14_Da…

---

## 5. AWS Backup và Backup Retention

Bạn cần nắm conceptual integration của `AWS Backup` với AWS resources.

Nội dung chính:

- backup policy awareness;
- backup plan;
- backup scheduling;
- recovery point;
- retention;
- backup vault awareness;
- recovery operation;
- verification.

Điểm vận hành phải nhớ:

> `Backup job = Completed` chưa đủ để kết luận backup strategy hoạt động.

Phải kiểm tra:

`backup creation → recovery point exists → restore → recovered data accessible → integrity verified`.

---

## 6. CloudWatch — Monitoring EC2/EBS và Alerting

Bạn cần học ba thành phần chính:

### Metrics

Hiểu metric là numerical telemetry dùng theo dõi tình trạng resource.

Phạm vi Day 5 gồm monitoring của:

- EC2;
- EBS;
- infrastructure resource behavior.

### Logs

Khái niệm `CloudWatch Logs`:

- log source;
- log group;
- log stream;
- retention awareness;
- searching/filtering awareness.

### Alarms

Workflow phải hiểu:

`Metric → Threshold/Condition → CloudWatch Alarm → State transition → notification/action awareness`.

Bạn cần phân biệt:

- `OK`
- `ALARM`
- `INSUFFICIENT_DATA`

và hiểu rằng alarm chỉ có giá trị nếu:

- metric đúng;
- threshold đúng;
- evaluation window đúng;
- notification path đúng;
- alarm đã được test.

---

## 7. CloudTrail — Audit Trail và Event Investigation

Bạn phải học:

- CloudTrail là gì.
- Management events.
- Event history.
- Actor/identity.
- API/action.
- Resource.
- Timestamp.
- Source.
- Request/result context.

Operational question điển hình:

> Ai đã stop EC2?

> Ai đã tạo snapshot?

> Ai thay đổi bucket configuration?

> Ai delete object hoặc modify policy?

Tuy nhiên, syllabus nhấn mạnh một giới hạn cực kỳ quan trọng:

**CloudTrail Event history chỉ cung cấp 90 ngày management events theo Region; không được coi nó là complete S3 object-access log hoặc long-term audit archive.** Infrastructure_Operations_14_Da…

Do đó bạn phải phân biệt:

- Management events.
- Data-event requirement.
- Event history.
- Trail.
- Long-term retention design.

---

## 8. AWS Cost Operations và Resource Cleanup

Day 5 không chỉ là backup mà còn yêu cầu tư duy FinOps cơ bản cho Infrastructure Operations.

Bạn cần học:

### AWS Budgets

- budget concept;
- spending threshold;
- alert awareness;
- cost control.

### Cost allocation

- tagging;
- resource ownership;
- environment identification;
- cost attribution.

### Major cost drivers

Syllabus yêu cầu chú ý:

- EC2 cost;
- EBS storage;
- EBS snapshots;
- S3 storage;
- S3 storage class;
- data transfer;
- public IPv4 charges.

### Unused-resource identification

Phải biết tìm các tài nguyên có khả năng đang gây lãng phí:

- idle/unused EC2;
- unattached EBS volume;
- unnecessary snapshots;
- unused public IPv4;
- unnecessary storage;
- obsolete lab resources.

### Stop versus Terminate

Đây là khái niệm operational rất quan trọng:

`stop` và `terminate` **không giống nhau**.

Bạn phải biết trước khi cleanup:

- dữ liệu nào còn tồn tại;
- EBS behavior;
- public IP implications;
- persistent cost;
- khả năng rollback/recovery.

Day 5 kết thúc bằng **AWS operational consolidation**: tổng hợp IAM/EC2/networking từ Day 4 với EBS/S3/monitoring/audit/cost/DR của Day 5 thành một operational workflow hoàn chỉnh. Infrastructure_Operations_14_Da…

---

# MODULE 2 — THỰC HÀNH

Module Thực hành của Day 5 có thể chia thành **8 lab/workstream liên kết với nhau**.

## Lab 1 — EBS Volume Lifecycle

Bạn sẽ:

1. Xác định EC2 lab instance.
2. Create/attach thêm một EBS volume.
3. Xác định block device trong Linux.
4. Format filesystem.
5. Mount filesystem.
6. Tạo persistent mount.
7. Verify bằng OS-level commands.
8. Tạo controlled test data.

Evidence dự kiến gồm:

- EC2 instance identifier;
- volume identifier;
- volume state;
- device mapping;
- filesystem;
- mount point;
- size/capacity;
- sample test data;
- timestamps.

---

# Lab 2 — EBS Snapshot và Restore

Workflow:

`Original EBS → write test data → snapshot → restored volume → attach → mount → verify data`.

Bạn sẽ phải:

- tạo snapshot;
- xác nhận snapshot hoàn thành;
- restore snapshot thành volume;
- attach restored volume;
- mount filesystem;
- kiểm tra restored files;
- so sánh source và restore.

Không chỉ kiểm tra:

> File có tồn tại hay không?

Mà còn phải kiểm tra:

- file count;
- expected filenames;
- content;
- checksum/hash khi phù hợp.

---

# Lab 3 — Secure S3 Backup Bucket

Bạn sẽ configure một controlled backup bucket với:

- approved IAM permissions;
- `Block Public Access`;
- encryption;
- versioning;
- lifecycle/retention policy.

Khi environment hỗ trợ, có thể thêm:

- replication;
- Object Lock.

Mục đích không đơn thuần là `aws s3 mb`, mà là tạo một bucket có **operational security baseline**.

---

# Lab 4 — Upload, Versioning, Delete/Overwrite và Recovery

Đây là một lab quan trọng nhất.

Bạn sẽ:

1. Chuẩn bị controlled backup dataset.
2. Record file inventory.
3. Tạo hash manifest.
4. Upload vào S3.
5. Verify upload.
6. Generate object versions.
7. Overwrite hoặc delete sandbox data có kiểm soát.
8. Xác định correct recoverable version.
9. Restore về **isolated destination**.
10. Compare với original manifest.

Syllabus yêu cầu recovered files phải match:

- expected content;
- expected count;
- pre-recorded integrity information.

Và đặc biệt:

> Không được dùng generic S3 object `ETag` như một universal content hash.

Infrastructure_Operations_14_Da…

---

# Lab 5 — Recovery Dependency / Failure Scenario

Bạn phải thử ít nhất một failure condition an toàn, ví dụ:

- access denied;
- missing permission;
- unavailable encryption-key fixture.

Mục tiêu là nhận ra rằng:

**Data tồn tại ≠ Data recoverable.**

Một recovery có thể thất bại dù object/snapshot vẫn tồn tại vì:

- IAM permission;
- bucket policy;
- KMS/key dependency;
- role/session permission;
- region/account dependency.

Bạn phải thu evidence của failure trước khi sửa.

---

# Lab 6 — RPO/RTO Measurement

Trong quá trình restore bạn sẽ ghi:

- timestamp của latest recoverable data;
- incident/recovery reference time;
- recovery start;
- recovery finish;
- integrity verification finish.

Sau đó tính:

### Observed RPO

Khoảng cách giữa required/current point và recovery point thực tế sử dụng.

### Observed RTO

Khoảng thời gian từ declared recovery start đến recovery completion.

Bạn phải ghi rõ:

- RPO target;
- RPO observed;
- RTO target;
- RTO observed;
- đạt/không đạt;
- dependency/bottleneck.

---

# Lab 7 — CloudWatch + CloudTrail + Budget

Ba phần được thực hành chung trong operational monitoring.

### CloudWatch

Bạn sẽ:

- inspect metrics;
- xác định EC2/EBS telemetry;
- tạo ít nhất một CloudWatch alarm;
- kiểm tra state;
- lưu evidence.

### CloudTrail

Bạn sẽ:

- tìm AWS operations vừa thực hiện;
- identify event;
- actor;
- time;
- action;
- target resource;
- success/failure.

Đồng thời kiểm tra giới hạn audit coverage và không giả định Event history là complete audit archive.

### AWS Budget

Bạn sẽ tạo hoặc inspect một budget control phù hợp sandbox:

- spending threshold;
- budget status;
- alert configuration.

---

# Lab 8 — Cost Review và Controlled Cleanup

Cuối Day 5 bạn phải thực hiện resource review.

Kiểm tra:

- running EC2;
- stopped EC2;
- unattached EBS;
- snapshots;
- S3 storage;
- public IPv4;
- transfer/storage implications;
- unnecessary lab resources.

Sau đó quyết định:

- retain;
- stop;
- terminate;
- delete;
- archive.

Nhưng mọi cleanup phải dựa trên:

`Dependency check → data protection check → approval/scope → action → verification`.

Không được cleanup chỉ vì resource "trông có vẻ không dùng".

---

# Deliverable cuối Day 5

Assignment yêu cầu bạn nộp một **combined AWS backup, monitoring, cost and disaster recovery checklist**. Infrastructure_Operations_14_Da…

Ở mức đầy đủ, package này nên chứng minh được:

- EBS volume đã attach/mount đúng.
- Persistent mount hoạt động.
- Test data tồn tại.
- Snapshot được tạo.
- Volume được restore.
- Restored data integrity được verify.
- S3 backup bucket có security baseline.
- Block Public Access được enable đúng mục tiêu.
- Encryption được configure.
- Versioning hoạt động.
- Lifecycle/retention policy có bằng chứng.
- Backup dataset upload thành công.
- Delete/overwrite recovery được thực hiện.
- Correct object version được recover.
- Restored copy được đặt ở isolated destination.
- File count và checksum/hash được đối chiếu.
- Access-denied/key-dependency scenario được hiểu.
- RPO được tính.
- RTO được tính.
- CloudWatch monitoring/alarm được configure và inspect.
- CloudTrail activity được inspect.
- Audit-coverage limitation được ghi nhận.
- AWS Budget/cost control được inspect hoặc configure.
- EC2/EBS/S3/public IPv4/storage/transfer cost drivers được nhận diện.
- Unused resources được review.
- Cleanup được thực hiện an toàn.
- Evidence có timestamp, resource identifier và ticket/lab reference phù hợp.

Syllabus còn yêu cầu common evidence approach xuyên suốt khóa học: giữ **raw evidence**, record source/resource, timestamp/timezone, command/filter, record count, SHA-256 và limitations; hash chỉ chứng minh file không thay đổi so với baseline được ghi lại, **không tự chứng minh source là đúng hoặc evidence là đầy đủ**. Infrastructure_Operations_14_Da…

---

## Bản đồ Day 5 rút gọn

Có thể hình dung toàn bộ Day 5 thành chuỗi sau:

**MODULE LÝ THUYẾT**

`EBS → Snapshot → Backup/Restore → RPO/RTO`

↓

`S3 Architecture → IAM/Bucket Policy → Block Public Access → Encryption`

↓

`Versioning → Lifecycle → Storage Classes → Replication → Object Lock`

↓

`AWS Backup → Recovery Dependency → Recovery Verification`

↓

`CloudWatch → CloudTrail → Audit Retention`

↓

`AWS Budgets → Cost Drivers → Cleanup`

**MODULE THỰC HÀNH**

`Attach EBS`

→ `Format + Mount`

→ `Write Test Data`

→ `Snapshot`

→ `Restore Volume`

→ `Verify Integrity`

→ `Build Secure S3 Backup Bucket`

→ `Upload + Version Data`

→ `Delete/Overwrite`

→ `Restore Correct Version to Isolated Destination`

→ `Checksum/File-count Validation`

→ `Failure Injection`

→ `Measure RPO/RTO`

→ `CloudWatch Alarm`

→ `CloudTrail Investigation`

→ `Budget/Cost Review`

→ `Controlled Cleanup`

→ `Evidence Package`

Đây là **toàn bộ scope Day 5** mà chúng ta sẽ triển khai sâu ở hai phần tiếp theo; không chỉ là “AWS S3/EBS”, mà thực chất là một mini-cycle hoàn chỉnh về **AWS storage operations + backup/recovery + DR assurance + monitoring + audit + cost governance**. Trong program summary, Day 4–5 được xác định chung là `IAM, VPC, EC2, EBS, S3 backup/DR, CloudWatch, CloudTrail, budgets`. Infrastructure_Operations_14_Da…

Khi bạn nói **“Lý thuyết”**, tôi sẽ triển khai Module Lý thuyết Day 5 thật chi tiết theo từng topic với **What → Why → How → Operational workflow → Security → Failure modes → Verification → Troubleshooting → Evidence → Escalation**, rồi sau đó chúng ta sẽ làm riêng Module **Thực hành**.
