# Day 5 — Module Lý thuyết

## AWS Storage, Backup/Disaster Recovery, Monitoring, Audit & Cost Operations

Day 5 là nửa sau của module **AWS Cloud Administration, Monitoring & Disaster Recovery**. Nếu Day 4 xây nền về `IAM → VPC → EC2 → secure access`, thì Day 5 trả lời các câu hỏi vận hành tiếp theo:

- Dữ liệu của EC2 nằm ở đâu và được bảo vệ thế nào?
- Nếu một `EBS volume`, file hoặc `S3 object` bị mất/overwrite thì phục hồi thế nào?
- Backup có thực sự restore được hay chỉ có trạng thái `Completed`?
- `RPO` và `RTO` được xác định và đo thế nào?
- Làm sao biết resource đang khỏe hay có vấn đề?
- Ai đã thay đổi resource và thay đổi lúc nào?
- Làm sao tránh AWS bill tăng vì resource bị quên?
- Khi kết thúc lab/change, cái gì phải giữ, cái gì có thể cleanup?

Theo syllabus, toàn bộ phạm vi Day 5 gồm `EBS volume types`, attachment, formatting/mounting, persistent mounts, snapshots, restore, backup validation, `RPO/RTO`, S3 architecture, IAM/bucket policies, `Block Public Access`, encryption, versioning, lifecycle, storage classes, replication, `Object Lock`, AWS Backup, recovery verification, CloudWatch, CloudTrail, AWS Budgets, cost allocation, public IPv4/storage/transfer costs, unused-resource identification và controlled cleanup. Phần bổ sung của syllabus yêu cầu đặc biệt phải kiểm chứng recovery, permission/key dependency, audit coverage và không được coi generic `ETag` là universal content hash. Infrastructure_Operations_14_Da…

---

# I. BỨC TRANH TỔNG THỂ CỦA DAY 5

Có thể xem Day 5 như một operational pipeline:

```text
Application
    │
    ▼
EC2
    │
    ├── EBS volume
    │      │
    │      ├── filesystem
    │      ├── EBS snapshot
    │      └── restored EBS volume
    │
    └── Backup data
           │
           ▼
          S3
           │
           ├── IAM / Bucket Policy
           ├── Block Public Access
           ├── Encryption
           ├── Versioning
           ├── Lifecycle
           ├── Storage Class
           ├── Replication
           └── Object Lock

Protection layer
    │
    └── AWS Backup / retention / recovery point

Observability layer
    │
    ├── CloudWatch metrics
    ├── CloudWatch alarms
    └── CloudWatch logs

Audit layer
    │
    └── CloudTrail

Business / Operations layer
    │
    ├── RPO
    ├── RTO
    ├── AWS Budgets
    ├── Cost allocation
    └── Resource cleanup
```

Một Infrastructure Operations engineer phải nhìn toàn bộ chuỗi trên như **một hệ thống**, không phải các AWS services riêng rẽ.

Ví dụ:

> EC2 bị lỗi không có nghĩa là mất dữ liệu nếu EBS và backup vẫn tốt.

Nhưng:

> Có EBS snapshot cũng chưa chắc phục hồi thành công nếu snapshot dùng encryption dependency mà recovery role không có quyền sử dụng key.

Hoặc:

> File trong S3 vẫn tồn tại nhưng application không thể recovery nếu IAM permission, KMS permission hoặc storage class retrieval requirement không được xử lý.

Đó chính là tư duy Day 5.

---

# II. AMAZON EBS — BLOCK STORAGE CHO EC2

## 1. EBS là gì?

`Amazon Elastic Block Store (EBS)` cung cấp **block storage** cho workload AWS, đặc biệt là EC2.

Block storage nhìn storage như một chuỗi các block dữ liệu. Operating system có thể:

- partition;
- tạo filesystem;
- mount;
- đọc/ghi;
- sử dụng cho database;
- application data;
- boot volume;
- log/data volume.

Điều quan trọng:

```text
EBS volume
≠
filesystem
≠
mount point
```

Ví dụ:

```text
EBS volume
    ↓
/dev/nvme1n1
    ↓
XFS filesystem
    ↓
/data
```

`/data` không phải EBS volume. Nó chỉ là vị trí trong Linux filesystem tree nơi filesystem nằm trên EBS volume được mount.

---

# III. EBS VOLUME LIFECYCLE

Một lifecycle thông thường:

```text
Create Volume
    ↓
Available
    ↓
Attach to EC2
    ↓
In-use
    ↓
Detect block device
    ↓
Partition if required
    ↓
Create filesystem
    ↓
Mount
    ↓
Application writes data
    ↓
Snapshot / backup
    ↓
Unmount / detach if required
    ↓
Delete when approved
```

Một điểm rất quan trọng: EBS volume chỉ attach được vào EC2 instance trong **cùng Availability Zone**, trừ các cơ chế khác không thuộc phạm vi cơ bản của Day 5. AWS cũng lưu ý device name mà EC2 API nhận có thể khác device name thực tế mà guest OS nhìn thấy. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-attaching-volume.html)

Ví dụ trên AWS Console bạn có thể yêu cầu:

```text
/dev/sdf
```

nhưng Nitro-based EC2 Linux có thể hiển thị:

```text
/dev/nvme1n1
```

Do đó Operations engineer không được giả định:

```text
Console device name == Linux device name
```

Luôn inspect từ guest OS.

---

# IV. EBS VOLUME TYPES

Theo AWS documentation hiện hành, EBS hiện có các nhóm SSD/HDD và previous-generation volume. Các loại quan trọng cho Day 5 gồm:

| Type       | Nhóm                         | Operational use                                |
| ---------- | ---------------------------- | ---------------------------------------------- |
| `gp3`      | General Purpose SSD          | General workload, boot/data volume             |
| `gp2`      | General Purpose SSD          | Existing/general workloads                     |
| `io2`      | Provisioned IOPS SSD         | High-performance, latency-sensitive workloads  |
| `io1`      | Provisioned IOPS SSD         | Existing Provisioned IOPS workloads            |
| `st1`      | Throughput Optimized HDD     | Large sequential workload, logs, big data      |
| `sc1`      | Cold HDD                     | Infrequently accessed throughput-oriented data |
| `standard` | Previous-generation magnetic | Legacy awareness                               |

AWS mô tả SSD volumes phù hợp với transactional workload và small random I/O, trong khi `st1/sc1` phù hợp hơn với large sequential I/O nơi throughput quan trọng hơn IOPS. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)

### `gp3`

Đây thường là lựa chọn general-purpose hiện đại.

Điểm quan trọng về design:

```text
capacity
IOPS
throughput
```

là những yếu tố cần xem xét riêng.

Đừng nghĩ:

> Volume lớn hơn tự động có performance profile giống nhau trong mọi loại EBS.

Bạn phải kiểm tra đúng volume type và workload.

### `io2`

Phù hợp workload:

- database I/O-intensive;
- latency-sensitive;
- sustained high IOPS;
- critical workload cần predictable storage performance.

### `st1`

Tối ưu cho throughput, ví dụ:

- big data;
- log processing;
- data warehouse;
- large sequential workloads.

### `sc1`

Hướng tới:

- infrequently accessed data;
- throughput workload;
- cost thấp hơn khi performance không phải yêu cầu chính.

### Common mistake

Một lỗi reasoning phổ biến:

> Database chậm → tăng disk size.

Không đủ.

Phải hỏi:

```text
Workload cần IOPS?
hay throughput?
hay low latency?
hay filesystem capacity?
hay instance EBS bandwidth đang bottleneck?
```

EBS performance không chỉ phụ thuộc volume; instance configuration, workload demand và EBS limits của instance cũng có thể ảnh hưởng. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html)

---

# V. ATTACHMENT, BLOCK DEVICE VÀ FILESYSTEM

Sau khi attach volume, Linux thấy một `block device`.

AWS documentation mô tả rằng attached EBS volume được expose như block device; sau đó OS có thể format và mount nó. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html)

Ví dụ read-only inspection:

```bash
lsblk
```

hoặc:

```bash
lsblk -f
```

Bạn có thể thấy:

```text
NAME        FSTYPE UUID                                 MOUNTPOINT
nvme0n1
└─nvme0n1p1 xfs    aaaa-bbbb-cccc                      /
nvme1n1
```

`nvme1n1` chưa có `FSTYPE` có thể là empty block device.

Nhưng **không được lập tức chạy `mkfs`**.

Phải xác định:

> Đây là new empty volume hay restored volume chứa dữ liệu?

AWS cảnh báo rằng volume tạo từ snapshot có thể đã chứa filesystem; tạo filesystem mới lên đó sẽ overwrite data. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html)

Đây là một trong những lỗi nguy hiểm nhất Day 5:

```bash
mkfs.xfs /dev/...
```

trên wrong device.

---

# VI. FILESYSTEM FORMATTING

Khi volume thật sự mới và empty:

```text
EBS block device
      ↓
filesystem creation
      ↓
mount
```

Filesystem thường gặp trên Linux:

- `XFS`
- `ext4`

Ví dụ:

```bash
sudo mkfs -t xfs /dev/nvme1n1
```

đây là **configuration-changing / destructive command** nếu device chứa data.

Do đó workflow đúng phải là:

```text
1. Identify device
2. Check existing filesystem
3. Confirm volume ID/device mapping
4. Confirm approved empty volume
5. Create filesystem
6. Verify
```

Không bao giờ đảo thứ tự thành:

```text
mkfs
→ rồi mới xem device là gì
```

---

# VII. MOUNT VÀ PERSISTENT MOUNT

Mount kết nối filesystem vào Linux filesystem hierarchy:

```text
/dev/nvme1n1
        ↓
       XFS
        ↓
      /data
```

Ví dụ:

```bash
sudo mount /dev/nvme1n1 /data
```

Sau đó verify:

```bash
findmnt /data
df -hT /data
lsblk -f
```

Nhưng mount thủ công thường không tồn tại sau reboot.

Vì vậy cần **persistent mount**.

Trên Linux thường sử dụng:

```text
/etc/fstab
```

AWS khuyến nghị dùng filesystem `UUID` thay vì device name khi cấu hình `/etc/fstab`, vì device name có thể thay đổi còn UUID ổn định hơn. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html)

Concept:

```text
UUID=<filesystem-uuid> /data xfs defaults,nofail 0 2
```

Không phải mọi môi trường đều dùng chính xác options trên; production phải theo OS/application standard.

### Vì sao `/etc/fstab` nguy hiểm?

Sai entry có thể gây:

- mount failure;
- delayed boot;
- emergency mode;
- application unavailable.

Do đó operational sequence tốt:

```text
backup /etc/fstab
→ edit
→ syntax/mount test
→ verify
→ reboot test nếu change scope cho phép
```

---

# VIII. EBS SNAPSHOT

## 1. Snapshot là gì?

`EBS snapshot` là point-in-time backup của EBS volume.

AWS mô tả EBS snapshot là **incremental backup**: sau snapshot đầu tiên, snapshot tiếp theo chỉ cần lưu các block thay đổi so với snapshot trước đó theo cơ chế quản lý của EBS. Snapshots được backend lưu trên S3 nhưng không nằm trong S3 bucket mà customer trực tiếp truy cập. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html?utm_source=chatgpt.com)

Điều này rất quan trọng:

> Không đi vào S3 Console để tìm “bucket chứa EBS snapshots”.

Bạn quản lý chúng bằng EBS/EC2 snapshot interfaces.

---

# IX. SNAPSHOT KHÔNG ĐỒNG NGHĨA APPLICATION-CONSISTENT BACKUP

Một point-in-time block-level snapshot có thể bắt được storage tại thời điểm application đang:

- ghi file;
- flush database;
- update transaction;
- thay đổi multiple volumes.

Do đó phải phân biệt:

### Crash consistency

Trạng thái gần giống hệ thống bị mất điện tại thời điểm capture.

### Application consistency

Application/database đã được chuẩn bị để recovery point phản ánh trạng thái logic hợp lệ.

Ví dụ với database, tùy product có thể cần:

- flush;
- freeze;
- quiesce;
- application-aware backup mechanism.

Day 5 không yêu cầu thành thạo application-specific snapshot mechanisms, nhưng phải hiểu:

> `snapshot completed` không chứng minh database/application recovery thành công.

---

# X. RESTORE EBS SNAPSHOT

EBS snapshot không được “mount trực tiếp” theo cách một filesystem thông thường được mount.

Workflow recovery cơ bản:

```text
EBS Snapshot
     ↓
Create new EBS volume
     ↓
Correct Availability Zone
     ↓
Attach to recovery EC2
     ↓
Identify device
     ↓
DO NOT reformat
     ↓
Mount filesystem
     ↓
Verify data
```

Một lỗi phổ biến:

```text
Create volume from snapshot
→ see block device
→ mkfs
```

Kết quả: phá chính dữ liệu bạn vừa restore.

---

# XI. EBS VOLUME INITIALIZATION SAU RESTORE

Đây là kiến thức bổ sung rất hữu ích.

Khi EBS volume được tạo từ snapshot, các data blocks cần được materialize/download vào volume khi sử dụng. Trong giai đoạn initialization, latency có thể cao hơn và performance thấp hơn bình thường. AWS có các cơ chế như initialization rate và fast snapshot restore cho trường hợp cần predictable/immediate performance. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-initialize.html)

Operational implication:

Nếu application sau restore chạy chậm, đừng lập tức kết luận:

> snapshot bị lỗi.

Có thể volume chưa fully initialized.

Do đó troubleshooting phải phân biệt:

```text
data integrity issue
vs
volume initialization/performance issue
vs
application issue
```

---

# XII. EBS — SECURITY IMPLICATIONS

Storage protection không dừng ở IAM.

Phải xét:

```text
IAM permission
   +
EBS encryption
   +
KMS permission where applicable
   +
snapshot permission
   +
resource/account/Region
```

Nếu volume/snapshot dùng KMS encryption, ability to see snapshot không tự động có nghĩa là ability to recover usable data.

Đây chính là dependency mà Day 5 muốn bạn nhận ra:

> Backup exists nhưng recovery principal không dùng được encryption key → DR chưa đạt.

AWS KMS sử dụng `key policy` làm cơ chế kiểm soát access chính; IAM policies, grants và các control khác có thể tham gia tùy design. Không có automatic permission dùng KMS key chỉ vì principal có permission trên resource được mã hóa. [AWS Documentation](https://docs.aws.amazon.com/kms/latest/developerguide/iam-policies.html?utm_source=chatgpt.com)

---

# XIII. EBS VERIFICATION

Một verification tốt không chỉ là:

```text
Volume state = in-use
```

Bạn cần kiểm tra nhiều layer:

```text
AWS layer
→ volume exists
→ correct type
→ correct size
→ correct AZ
→ correct instance
→ encrypted state

OS layer
→ expected block device
→ expected filesystem
→ expected UUID
→ expected mount point

Application layer
→ application can access required path
→ permissions correct

Backup layer
→ snapshot/recovery point exists

Recovery layer
→ restored data readable
```

---

# XIV. EBS COMMON FAILURE MODES

Các failure thường gặp:

### 1. Volume không attach được

Possible causes:

- wrong Availability Zone;
- permission;
- attachment limit;
- incorrect instance state/use case.

### 2. Attach thành công nhưng Linux không thấy như expected

Possible cause:

- NVMe device naming;
- wrong assumed device mapping.

### 3. Mount báo unknown filesystem

Possible causes:

- unformatted volume;
- wrong device;
- filesystem corruption;
- missing filesystem tool/support.

### 4. Data biến mất sau reboot

Possible cause:

- mount không persistent;
- `/etc/fstab` sai;
- wrong UUID.

### 5. EC2 boot issue sau sửa `/etc/fstab`

Possible causes:

- incorrect device;
- typo;
- unavailable filesystem/device;
- inappropriate mount options.

### 6. Performance thấp

Possible causes:

- wrong EBS volume type;
- IOPS limit;
- throughput limit;
- instance EBS limit;
- queue/latency;
- initialization after restore;
- application pattern không phù hợp storage type.

---

# XV. EBS TROUBLESHOOTING FLOW

Đi từ ít rủi ro đến nhiều rủi ro:

```text
1. Read AWS volume state
2. Confirm volume ID / instance ID / AZ
3. Inspect block devices
4. Inspect filesystem metadata
5. Inspect current mounts
6. Inspect filesystem capacity
7. Inspect CloudWatch EBS metrics
8. Review OS/application logs
9. Inspect snapshot/recovery point
10. Only then consider configuration change
```

Không troubleshoot storage bằng:

```text
format
delete
detach
reboot
terminate
```

ngay từ đầu.

---

# XVI. AMAZON S3 — OBJECT STORAGE

## 1. Block storage và object storage khác gì?

EBS:

```text
Application
→ filesystem
→ blocks
```

S3:

```text
Application
→ API
→ object
```

S3 không phải “network disk” theo nghĩa truyền thống.

Core objects:

```text
Bucket
Object
Key
Version
Metadata
Storage Class
```

### Bucket

Logical container cho S3 objects.

### Object

Data cùng metadata.

### Object key

Tên/identifier của object trong bucket.

Ví dụ:

```text
backups/server01/2026-10-20/app.tar.gz
```

Toàn bộ chuỗi trên là object key.

Các dấu `/` hỗ trợ cách tổ chức giống hierarchy nhưng S3 fundamentally là object storage chứ không phải POSIX filesystem truyền thống.

---

# XVII. S3 ARCHITECTURE TRONG BACKUP

Một design đơn giản:

```text
Source Server
     │
     ▼
Backup set
     │
     ▼
S3 Bucket
     │
     ├── encryption
     ├── versioning
     ├── lifecycle
     ├── access policy
     └── monitoring/audit
```

Một design DR tốt hơn:

```text
Production data
      │
      ▼
Backup
      │
      ▼
Primary S3 bucket
      │
      ├── Versioning
      ├── Encryption
      └── Retention
      │
      ▼
Optional Replication
      │
      ▼
Separate bucket/account/Region
```

Nhưng phải nhớ:

> More copies không tự động bằng recoverable backup.

---

# XVIII. S3 IAM ACCESS VÀ BUCKET POLICY

Có hai lớp bạn thường gặp:

### Identity-based policy

Attached vào:

- IAM user;
- IAM role;
- group policy model.

Nó trả lời:

> Identity này có thể thực hiện action gì?

### Resource-based policy

S3 bucket policy là resource policy.

Nó trả lời:

> Ai được phép thực hiện action gì trên resource này, với điều kiện nào?

Ví dụ mental model:

```text
Principal
   │
   ├── IAM policy
   │
   └── request
         │
         ▼
       S3
         │
         └── Bucket policy
```

AWS policy evaluation nói chung kết hợp applicable permissions; một **explicit Deny** có precedence và override applicable Allow. [AWS Documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html?utm_source=chatgpt.com)

Do đó troubleshooting `AccessDenied` không được chỉ nhìn một policy.

Bạn phải hỏi:

```text
Who am I?
Which role/session?
Which action?
Which resource ARN?
Identity policy?
Bucket policy?
Explicit deny?
Permissions boundary?
Organization SCP?
KMS policy?
VPC endpoint policy where relevant?
```

---

# XIX. LEAST PRIVILEGE TRONG S3

Backup operator có thể cần:

```text
PutObject
GetObject
ListBucket
```

nhưng điều đó không có nghĩa cần:

```text
s3:*
```

Recovery role có thể khác backup-writer role.

Một design tốt có thể tách:

```text
Backup Writer
    → write backup

Recovery Operator
    → read/recover approved backup

Security Administrator
    → manage policy

Retention Administrator
    → manage retention controls
```

Đây là `segregation of duties`.

---

# XX. BLOCK PUBLIC ACCESS

`S3 Block Public Access` là guardrail để hạn chế public exposure qua bucket/access point/account/organization level.

AWS hiện cho phép áp dụng controls ở nhiều scope, và Block Public Access có thể override các policies/permissions cố tạo public access theo cấu hình tương ứng. New buckets mặc định không cho public access. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html?trk=article-ssr-frontend-pulse_little-text-block&utm_source=chatgpt.com)

Trong backup bucket, default operational stance nên là:

```text
Public access?
NO
```

trừ khi architecture đã được security review cho một use case đặc biệt.

### Vì sao đặc biệt quan trọng?

Backup thường chứa:

- database dump;
- configurations;
- logs;
- PII;
- credentials/config artifacts;
- internal data.

Một backup bucket public có thể nghiêm trọng hơn chính source system.

---

# XXI. S3 ENCRYPTION

S3 encryption phải nhìn theo ba câu hỏi:

```text
1. Data có encrypted at rest không?
2. Ai quản lý key?
3. Recovery principal có quyền dùng key không?
```

AWS hiện áp dụng `SSE-S3` như baseline server-side encryption cho new object uploads. Nếu muốn kiểm soát key chặt hơn, có thể sử dụng `SSE-KMS`; AWS hiện cũng hỗ trợ `DSSE-KMS` trong các use case tương ứng. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/default-bucket-encryption.html?utm_source=chatgpt.com)

### SSE-S3

AWS/S3 quản lý encryption keys.

Operationally đơn giản hơn.

### SSE-KMS

Sử dụng AWS KMS.

Cho nhiều control hơn về:

- key policy;
- auditability;
- key management;
- access separation.

Nhưng đồng thời tạo thêm dependency.

```text
S3 permission
     +
KMS permission
     =
Successful encrypted-object recovery
```

Nếu có:

```text
s3:GetObject = Allow
```

nhưng thiếu KMS permission cần thiết:

```text
recovery may fail
```

Đây là lý do syllabus yêu cầu **recovery-key availability** phải là một phần của DR design. Infrastructure_Operations_14_Da…

---

# XXII. RECOVERY DEPENDENCY TREE

Khi thiết kế DR, nên hình dung dependency:

```text
Recovery
   │
   ├── Backup exists?
   │
   ├── Correct recovery point?
   │
   ├── IAM role available?
   │
   ├── S3 permissions available?
   │
   ├── Bucket policy allows access?
   │
   ├── KMS key enabled?
   │
   ├── Key policy permits recovery?
   │
   ├── Required Region/account available?
   │
   ├── Storage class retrieval ready?
   │
   └── Application can consume restored data?
```

Đây mới là DR thinking.

---

# XXIII. S3 VERSIONING

`S3 Versioning` cho phép giữ multiple variants của cùng object key.

AWS mô tả versioning là cơ chế giữ và recover các version khác nhau của object, giúp phục hồi khỏi accidental deletion hoặc overwrite. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html?utm_source=chatgpt.com)

Ví dụ:

```text
report.csv
   │
   ├── Version A
   ├── Version B
   └── Version C ← current
```

Nếu overwrite:

```text
Version C
    ↓
Version D becomes current
```

C không nhất thiết mất.

---

# XXIV. DELETE MARKER

Trong bucket bật versioning, một simple delete không nhất thiết permanently remove object version.

AWS có thể tạo `delete marker`.

Ví dụ:

```text
backup.tar.gz
   ├── v1
   ├── v2
   └── delete-marker ← current
```

Normal `GET` không chỉ version ID có thể cho bạn cảm giác object đã biến mất.

Nhưng underlying versions có thể vẫn tồn tại.

AWS mô tả delete marker là placeholder không chứa object data; nó trở thành current version sau simple `DELETE` trong versioned bucket. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/DeleteMarker.html?utm_source=chatgpt.com)

Điều này rất quan trọng khi troubleshoot:

> User: “File đã bị delete khỏi S3.”

Operations engineer:

> “Permanent deletion hay delete marker?”

Hai tình huống hoàn toàn khác.

---

# XXV. VERSIONING KHÔNG PHẢI BACKUP ĐỘC LẬP

Đây là một lesson quan trọng.

Versioning bảo vệ rất tốt trước:

- overwrite;
- accidental simple delete;
- application mistakes.

Nhưng vẫn tồn tại risk như:

- privileged permanent version deletion;
- compromised identity;
- malicious administrative action;
- retention configuration issue;
- dependency cùng account.

Do đó:

```text
Versioning
≠
Independent backup strategy
```

Syllabus yêu cầu phải tách riêng:

```text
Versioning
Replication
Retention
Backup
```

không dùng chúng như synonyms. Infrastructure_Operations_14_Da…

---

# XXVI. S3 LIFECYCLE MANAGEMENT

`S3 Lifecycle` tự động xử lý object theo rule.

Hai nhóm behavior lớn:

### Transition

Chuyển object sang storage class phù hợp hơn.

Ví dụ:

```text
S3 Standard
   ↓ after X days
Standard-IA
   ↓
Glacier class
```

### Expiration

Xử lý việc hết hạn/deletion của current hoặc noncurrent data tùy rule.

Lifecycle giúp quản lý:

- storage cost;
- retention;
- old versions;
- archival.

AWS cho phép transition qua nhiều storage classes với constraints khác nhau; ví dụ current default transition behavior cũng có considerations đối với objects dưới 128 KB. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/lifecycle-transition-general-considerations.html)

---

# XXVII. LIFECYCLE — NGUY CƠ VẬN HÀNH

Lifecycle không chỉ là “cost optimization”.

Một rule sai có thể:

- expire data quá sớm;
- delete noncurrent versions cần cho recovery;
- transition data sang archive class làm RTO dài lên;
- tăng retrieval cost;
- phá retention expectation.

Ví dụ business requirement:

```text
RTO = 15 minutes
```

nhưng backup quan trọng bị transition vào storage class cần asynchronous restore.

Bạn có thể đã giảm storage cost nhưng phá RTO.

Do đó lifecycle phải align với:

```text
Retention
+
RPO
+
RTO
+
Access frequency
+
Recovery cost
```

---

# XXVIII. S3 STORAGE CLASSES

Các storage classes AWS hiện mô tả bao gồm:

- `S3 Standard`
- `S3 Intelligent-Tiering`
- `S3 Standard-IA`
- `S3 One Zone-IA`
- `S3 Express One Zone`
- `S3 Glacier Instant Retrieval`
- `S3 Glacier Flexible Retrieval`
- `S3 Glacier Deep Archive`

và một số specialized/legacy contexts khác. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)

Đối với Day 5, không cần thuộc lòng mọi pricing number. Quan trọng là **trade-off**.

| Storage Class              | Ý tưởng vận hành                                      |
| -------------------------- | ----------------------------------------------------- |
| S3 Standard                | Frequent access                                       |
| Intelligent-Tiering        | Unknown/changing access pattern                       |
| Standard-IA                | Infrequent access nhưng cần nhanh khi truy cập        |
| One Zone-IA                | Infrequent, one-AZ, data có thể re-create phù hợp hơn |
| Glacier Instant Retrieval  | Rare access nhưng millisecond retrieval               |
| Glacier Flexible Retrieval | Archive, retrieval không real-time                    |
| Glacier Deep Archive       | Long-term archive, recovery chậm hơn                  |

AWS cũng nêu rõ `Standard-IA` và `One Zone-IA` có retrieval fees, còn archived data trong Glacier Flexible Retrieval/Deep Archive phải trải qua restore process trước khi normal access. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)

---

# XXIX. STORAGE CLASS VÀ RTO

Đây là operational connection rất quan trọng.

Giả sử:

```text
Backup size = 3 TB
RTO target = 30 minutes
```

Nếu data nằm ở một archive tier không immediate-access, recovery workflow sẽ khác S3 Standard.

Do đó không được chọn storage class chỉ bằng:

```text
$/GB thấp nhất
```

Phải hỏi:

```text
How often accessed?
How quickly must recovery begin?
How much data?
What retrieval behavior?
What retrieval cost?
Minimum storage duration?
Is data primary copy or secondary copy?
```

---

# XXX. S3 REPLICATION

S3 replication là automatic/asynchronous object-copy mechanism giữa S3 buckets.

AWS hiện hỗ trợ:

- `Cross-Region Replication (CRR)`
- `Same-Region Replication (SRR)`
- live replication;
- Batch Replication cho existing objects/use cases tương ứng. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)

Một design:

```text
Source Bucket
   │
   │ asynchronous replication
   ▼
Destination Bucket
```

Có thể cùng account hoặc khác account và tùy use case có thể khác Region.

Source và destination buckets cho replication cần versioning enabled, cùng với required permissions. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-requirements.html?utm_source=chatgpt.com)

---

# XXXI. REPLICATION KHÔNG ĐỒNG NGHĨA BACKUP

Hãy xem tình huống:

```text
Source:
good-data.txt
```

Replication tạo:

```text
Destination:
good-data.txt
```

Sau đó malicious/application action tạo new bad version:

```text
Source:
bad-data.txt
```

Nếu behavior nằm trong replication scope:

```text
Destination:
bad-data.txt
```

Replica giúp availability/resilience, nhưng không tự trả lời:

> “Tôi cần state trước corruption.”

Muốn giải quyết cần xem cả:

- versioning;
- retention;
- Object Lock;
- independent backup;
- recovery-point strategy.

---

# XXXII. OBJECT LOCK

`S3 Object Lock` hỗ trợ WORM-style protection (`write once, read many`) để ngăn object version bị overwrite/delete trong retention period hoặc theo legal hold.

Object Lock hoạt động với versioning và bảo vệ theo object version. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html?utm_source=chatgpt.com)

Hai concept cần biết:

### Retention period

Object version được bảo vệ trong thời gian xác định.

### Legal hold

Không dựa trên fixed expiration giống retention period; được duy trì cho đến khi được remove theo control phù hợp.

Object Lock có các mode cần hiểu:

- `Governance`
- `Compliance`

Các mode có operational implications khác nhau về khả năng bypass/change retention.

Vì đây là control mạnh, không nên bật bừa trong lab/prod.

---

# XXXIII. OBJECT LOCK — RISK

Một sai lầm nguy hiểm:

> “Object Lock càng lâu càng an toàn.”

Không nhất thiết.

Nếu retention setting sai, organization có thể tự khóa data khỏi legitimate deletion trong một thời gian dài.

Vì vậy cần:

```text
Business retention requirement
→ compliance requirement
→ legal approval where relevant
→ storage-cost assessment
→ operational recovery design
→ Object Lock configuration
```

Syllabus chỉ yêu cầu **Object Lock awareness**, và optional implementation nơi environment hỗ trợ. Infrastructure_Operations_14_Da…

---

# XXXIV. VERSIONING vs REPLICATION vs RETENTION vs BACKUP

Đây là bảng phải nắm chắc:

| Control     | Mục tiêu chính                                                           |
| ----------- | ------------------------------------------------------------------------ |
| Versioning  | Giữ nhiều versions của object                                            |
| Replication | Copy data sang bucket/location khác                                      |
| Retention   | Quy định data phải được giữ bao lâu                                      |
| Object Lock | Enforce protection chống delete/overwrite theo retention/legal mechanism |
| Backup      | Tạo recovery points để phục vụ restore                                   |
| Lifecycle   | Tự động quản lý transition/expiration                                    |
| Encryption  | Bảo vệ confidentiality của data at rest                                  |
| Checksum    | Hỗ trợ integrity validation                                              |

Không control nào tự thay thế tất cả control khác.

---

# XXXV. AWS BACKUP

`AWS Backup` cung cấp centralized mechanism để quản lý backup cho các AWS resources được hỗ trợ.

Core concepts:

```text
Protected Resource
      │
      ▼
Backup Plan
      │
      ├── Schedule
      ├── Retention
      └── Lifecycle/copy behavior
      │
      ▼
Recovery Point
      │
      ▼
Backup Vault
```

AWS dùng thuật ngữ `recovery point` để đại diện cho backup của resource tại một thời điểm; recovery points được lưu trong backup vaults. [AWS Documentation](https://docs.aws.amazon.com/aws-backup/latest/devguide/recovery-points.html?utm_source=chatgpt.com)

---

# XXXVI. BACKUP PLAN

Backup plan trả lời:

```text
What?
When?
How frequently?
How long retained?
Where?
Potentially where copied?
```

Ví dụ logical policy:

```text
Production database volume
→ backup every hour
→ daily retention X
→ monthly retention Y
→ optional copy into alternate scope
```

Các giá trị thực tế phải theo business policy.

---

# XXXVII. BACKUP RETENTION

Retention phải derived từ:

- regulatory requirement;
- business requirement;
- recovery objective;
- historical recovery need;
- legal requirement;
- cost.

Ví dụ:

```text
Hourly backup → 24 hours
Daily backup → 30 days
Monthly backup → 1 year
```

chỉ là illustration.

Không phải universal best practice.

---

# XXXVIII. “BACKUP COMPLETED” KHÔNG ĐỦ

Đây là một principle quan trọng nhất của Day 5.

```text
Backup job = COMPLETED
```

chỉ chứng minh backup workflow đạt một trạng thái service-level nhất định.

Nó chưa chứng minh:

```text
Recovery permission works
Correct recovery point exists
Encryption key works
Filesystem mounts
Files are intact
Application starts
RTO is achievable
RPO is acceptable
```

Do đó:

```text
Backup assurance
=
Backup creation
+
Restore test
+
Integrity check
+
Application/readability check
+
RPO/RTO measurement
```

---

# XXXIX. AWS BACKUP RESTORE TESTING

AWS Backup hiện hỗ trợ `Restore testing`, cho phép periodic evaluation khả năng restore và theo dõi restore job duration cho supported resource types. [AWS Documentation](https://docs.aws.amazon.com/aws-backup/latest/devguide/restore-testing.html?utm_source=chatgpt.com)

Đây là kiến thức bổ sung rất phù hợp với syllabus.

Ý nghĩa:

> DR testing không cần luôn chờ disaster thật mới biết backup có restore được hay không.

Ideal operating cycle:

```text
Backup
→ scheduled restore test
→ validation
→ evidence
→ cleanup
→ remediate failures
```

---

# XL. RPO — RECOVERY POINT OBJECTIVE

## 1. RPO trả lời gì?

RPO trả lời:

> Organization chấp nhận mất tối đa bao nhiêu data tính theo thời gian?

Ví dụ:

```text
RPO target = 1 hour
```

có nghĩa business design kỳ vọng recovery point không quá cũ so với boundary đã định.

Không nên diễn giải RPO đơn giản là:

> “Backup chạy mỗi giờ.”

Backup schedule và achieved RPO có liên quan nhưng không hoàn toàn giống nhau.

---

# XLI. OBSERVED RPO

Syllabus yêu cầu tính **observed RPO** dựa trên latest recoverable data point. Infrastructure_Operations_14_Da…

Ví dụ:

```text
Incident reference time:
15:00

Latest verified recoverable data:
14:32
```

Observed data-loss window:

```text
28 minutes
```

Nếu:

```text
RPO target = 15 minutes
```

thì:

```text
Observed RPO > Target
```

→ objective không đạt.

---

# XLII. RTO — RECOVERY TIME OBJECTIVE

RTO trả lời:

> Service/data phải được phục hồi trong bao lâu sau recovery process bắt đầu theo định nghĩa organization?

Ví dụ:

```text
RTO target = 60 minutes
```

Nếu recovery:

```text
Start: 15:10
Usable service: 15:47
```

Observed recovery duration:

```text
37 minutes
```

RTO đạt nếu measurement methodology phù hợp policy.

---

# XLIII. KHÔNG NHẦM RPO VÀ RTO

Mnemonic:

```text
RPO → Point → How much data might be lost?

RTO → Time → How long recovery may take?
```

Ví dụ:

```text
Backup every 15 min
Restore takes 4 hours
```

có thể có:

```text
good RPO
poor RTO
```

Hoặc:

```text
Backup daily
Restore takes 10 min
```

có thể có:

```text
poor RPO
good RTO
```

Hai chỉ số độc lập nhưng liên quan.

---

# XLIV. DR DESIGN

`Disaster Recovery` không chỉ là backup.

Một DR design tối thiểu phải biết:

```text
What are we recovering?
From which recovery point?
Into what environment?
Who is authorized?
Which credentials?
Which encryption key?
Which network?
Which dependency?
Which DNS/config?
How is integrity verified?
How is application validated?
What is RPO?
What is RTO?
Who declares recovery complete?
```

Vì vậy syllabus yêu cầu ghi nhận:

- permissions;
- key dependencies;
- recovery costs;
- audit gaps;
- observed RPO/RTO. Infrastructure_Operations_14_Da…

---

# XLV. ISOLATED RESTORE

Một recovery test tốt thường restore vào **isolated destination** thay vì đè trực tiếp lên source.

Lý do:

- tránh overwrite good data;
- tránh duplicate hostname/IP;
- tránh application conflict;
- giảm blast radius;
- cho phép compare source vs restore;
- dễ rollback/cleanup.

Mental model:

```text
Production/Source
      │
      ├── Backup
      │
      ▼
 Recovery Point
      │
      ▼
 Isolated Restore
      │
      ▼
 Verification
```

Chỉ sau verification mới quyết định production recovery action theo approved procedure.

---

# XLVI. DATA INTEGRITY

Restore thành công chưa đủ.

Phải hỏi:

```text
Restored WHAT?
Exactly HOW MANY files?
Are they readable?
Are contents correct?
Were any files missing?
Are hashes/checksums correct?
```

Một integrity manifest có thể chứa:

```text
filename
size
hash
timestamp
```

Ví dụ:

```text
app.tar.gz    14823419    SHA256=...
config.json      38291    SHA256=...
```

Sau recovery:

```text
expected
vs
actual
```

---

# XLVII. CHECKSUM VÀ S3 ETag

Syllabus đặc biệt nhấn mạnh:

> Không được coi generic S3 object `ETag` là universal content hash. Infrastructure_Operations_14_Da…

Vì ETag behavior có thể phụ thuộc upload/encryption path và không phải lúc nào cũng bằng simple MD5 của file.

AWS S3 hỗ trợ explicit checksum mechanisms, gồm nhiều algorithms như CRC variants, SHA family và MD5 trong các contexts tương ứng, và có thể sử dụng checksum metadata để validate object integrity. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/checking-object-integrity-upload.html)

Với evidence training này, syllabus cũng khuyến khích sử dụng:

```text
SHA-256
```

cho evidence manifest.

---

# XLVIII. BACKUP VALIDATION LEVELS

Có thể chia validation thành 5 mức:

### Level 1 — Job status

```text
Backup completed
```

Yếu nhất.

### Level 2 — Recovery point exists

```text
Snapshot/version/recovery point present
```

### Level 3 — Restore succeeds

```text
Resource recreated
```

### Level 4 — Data integrity succeeds

```text
files/count/hash correct
```

### Level 5 — Application recovery succeeds

```text
application can consume restored data
```

Day 5 muốn bạn tiến ít nhất tới Level 4 và hiểu Level 5.

---

# XLIX. CLOUDWATCH

`Amazon CloudWatch` là observability/monitoring service quan trọng của AWS.

Day 5 tập trung vào:

```text
Metrics
Logs
Alarms
EC2 monitoring
EBS monitoring
```

---

# L. CLOUDWATCH METRICS

Metric là numeric telemetry về trạng thái/performance của system.

AWS mô tả CloudWatch metrics là dữ liệu về performance của systems; AWS services như EC2 và EBS publish metrics vào CloudWatch. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/working_with_metrics.html?utm_source=chatgpt.com)

Metric thường có:

```text
Namespace
Metric name
Dimensions/labels depending model
Timestamp
Value
Statistic
Period
```

Ví dụ conceptual:

```text
AWS/EC2
CPUUtilization
InstanceId=i-12345
```

hoặc:

```text
AWS/EBS
VolumeReadOps
VolumeId=vol-12345
```

---

# LI. METRIC KHÔNG TỰ CÓ Ý NGHĨA NẾU KHÔNG CÓ CONTEXT

Ví dụ:

```text
CPUUtilization = 95%
```

Không đủ để kết luận incident.

Cần context:

- spike 30 giây hay 2 giờ?
- workload expected?
- application latency?
- memory pressure?
- autoscaling?
- maintenance?
- baseline?

Operations luôn cần:

```text
Metric + Time + Baseline + Workload + Dependency
```

---

# LII. EC2 MONITORING

EC2 có metrics như:

- CPU utilization;
- network;
- disk-related metrics tùy nguồn/resource;
- status-check related telemetry;
- EBS-related instance metrics tùy platform.

Một distinction quan trọng:

### Infrastructure-level metric

CloudWatch có thể biết EC2 CPU utilization.

### Guest OS metric

CloudWatch mặc định không magically biết mọi OS-level metric như:

```text
filesystem free %
memory usage
process status
application queue
```

Những metric này thường cần agent/custom telemetry/application instrumentation.

Đây là common misunderstanding:

> “CloudWatch có EC2 metrics thì chắc chắn biết `/var` đầy 97%.”

Không nhất thiết.

---

# LIII. EBS CLOUDWATCH METRICS

AWS hiện cung cấp EBS metrics tự động ở 1-minute periods cho attached volumes. Các metrics gồm những nhóm như:

- read/write operations;
- read/write bytes;
- queue length;
- latency-related metrics;
- IOPS/throughput exceeded checks trên supported environments;
- `BurstBalance` cho một số volume types. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/using_cloudwatch_ebs.html?utm_source=chatgpt.com)

Ví dụ important reasoning:

```text
High queue
+
increasing latency
+
IOPS limit exceeded
```

có thể cho thấy storage bottleneck.

Nhưng:

```text
High queue alone
```

chưa đủ kết luận.

---

# LIV. IOPS, THROUGHPUT VÀ LATENCY

Phải phân biệt ba thứ:

### IOPS

Input/output operations per second.

```text
How many I/O operations?
```

### Throughput

Amount of data per second.

```text
How much data?
```

### Latency

Thời gian hoàn thành một I/O operation.

```text
How long per I/O?
```

Ví dụ:

- database random 8 KiB reads → IOPS/latency quan trọng;
- large sequential log processing → throughput quan trọng.

AWS nhấn mạnh queue length và latency phải được đánh giá cùng workload/I/O characteristics để hiểu bottleneck. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-io-characteristics.html?utm_source=chatgpt.com)

---

# LV. CLOUDWATCH ALARM

Alarm theo dõi metric/expression và đánh giá threshold qua time periods.

Một basic alarm:

```text
Metric
   ↓
Statistic
   ↓
Period
   ↓
Threshold
   ↓
Evaluation periods
   ↓
Alarm state
   ↓
Action / notification
```

AWS hiện mô tả metric alarm có thể theo dõi metric hoặc math expression; action có thể bao gồm SNS notification và một số AWS actions khác. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Alarms.html)

---

# LVI. CLOUDWATCH ALARM STATES

Ba state cơ bản:

### `OK`

Metric hiện nằm trong condition bình thường.

### `ALARM`

Condition đang breach theo evaluation logic.

### `INSUFFICIENT_DATA`

Không đủ data để xác định. [AWS Documentation](https://docs.aws.amazon.com/hi_in/AmazonCloudWatch/latest/monitoring/alarm-evaluation.html?utm_source=chatgpt.com)

Điểm cực kỳ quan trọng:

```text
INSUFFICIENT_DATA
≠
healthy
```

và cũng:

```text
INSUFFICIENT_DATA
≠
definitely broken
```

Nó có thể là:

- resource mới;
- metric chưa xuất hiện;
- resource inactive;
- missing data;
- telemetry issue.

AWS lưu ý EBS volume unattached có thể không publish certain metric activity và alarm có thể rơi vào `INSUFFICIENT_DATA`. [AWS Documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Alarms.html)

---

# LVII. ALARM DESIGN

Một alarm xấu:

```text
CPU > 50%
```

không context.

Một alarm tốt hơn phải xác định:

```text
Metric
Threshold
Period
Number of periods
Missing-data behavior
Expected action
Owner
Runbook
Severity
```

Ví dụ conceptual:

```text
CPUUtilization > 85%
for 3 consecutive evaluation periods
```

Nhưng actual threshold phải based on application.

Không có universal:

```text
80% = critical
```

---

# LVIII. ALARM KHÔNG CÓ RUNBOOK = CHƯA HOÀN CHỈNH

Khi alarm fire, operator phải biết:

```text
1. What resource?
2. What threshold?
3. What changed?
4. What should I check first?
5. Is customer impact known?
6. Can L1 resolve?
7. When escalate?
```

Nếu alarm chỉ gửi:

```text
ALARM: EBS high
```

không resource/timestamp/runbook thì operational maturity thấp.

---

# LIX. CLOUDWATCH LOGS

CloudWatch Logs là phần lưu/search log streams từ supported AWS/application/log-agent sources.

Core hierarchy truyền thống:

```text
Log Group
    │
    └── Log Stream
           │
           └── Log Events
```

Day 5 chỉ cần foundation:

- log source;
- timestamp;
- retention;
- filtering;
- access control;
- correlation.

Deep centralized logging sẽ được học Day 10.

---

# LX. CLOUDWATCH ≠ CLOUDTRAIL

Đây là distinction bắt buộc.

### CloudWatch

Hỏi:

> System/resource đang hoạt động thế nào?

Ví dụ:

```text
CPU?
IOPS?
Latency?
Alarm?
Logs?
```

### CloudTrail

Hỏi:

> Ai/API/service đã thực hiện action gì trong AWS account?

Ví dụ:

```text
Who stopped EC2?
Who modified bucket policy?
Who created snapshot?
When?
From where?
```

---

# LXI. CLOUDTRAIL

AWS CloudTrail ghi lại AWS account activity/API-oriented events trong phạm vi được hỗ trợ/configure.

CloudTrail events có thể chứa:

- identity;
- event time;
- service;
- API/action;
- resource;
- source IP/context;
- request parameters;
- error information;
- response metadata depending event.

AWS hiện chia CloudTrail event types thành management events, data events, network activity events và Insights events. [AWS Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-events.html?utm_source=chatgpt.com)

Day 5 tập trung nhiều nhất vào:

```text
Management events
+
awareness về S3 data events
```

---

# LXII. MANAGEMENT EVENT

Management event là control-plane activity.

Ví dụ:

```text
CreateVolume
CreateSnapshot
StopInstances
PutBucketPolicy
PutBucketVersioning
```

Concept:

> Thay đổi configuration/control plane.

---

# LXIII. DATA EVENT

Data event liên quan operations trên data/resource content trong các services hỗ trợ.

Đối với S3, đây là distinction cực kỳ quan trọng khi muốn audit object-level access.

Ví dụ conceptually:

```text
GetObject
PutObject
DeleteObject
```

không nên giả định generic Event history tự động cho bạn complete picture của mọi object access.

---

# LXIV. CLOUDTRAIL EVENT HISTORY — 90-DAY LIMIT

Syllabus nhấn mạnh điều này và AWS docs hiện xác nhận:

> `CloudTrail Event history` cho phép xem **90 ngày management events** cho current AWS Region. Nó **không hiển thị data events**. Muốn ongoing record lâu hơn hoặc data events cần appropriate trail/event data store design. [AWS Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/tutorial-event-history.html?utm_source=chatgpt.com)

Đây là một câu interview rất hay:

> “Tôi vào Event history mà không thấy user tải một S3 object ba tháng trước. Có nghĩa là user không tải đúng không?”

**Sai.**

Có thể vì:

- event quá 90 ngày;
- object access là data event;
- data events không được configured;
- wrong Region/time/filter;
- retention design không phù hợp.

Không có evidence không tự động bằng evidence rằng action không xảy ra.

---

# LXV. CLOUDTRAIL TROUBLESHOOTING MENTAL MODEL

Khi tìm activity:

```text
1. Correct AWS account?
2. Correct Region?
3. Correct time window?
4. Correct event type?
5. Correct event source?
6. Correct identity?
7. Correct resource?
8. Management vs data event?
9. Trail/data store enabled?
10. Retention available?
```

Không được kết luận sớm:

```text
No search result
→ no event occurred
```

---

# LXVI. AUDIT RETENTION DESIGN

Day 5 yêu cầu hiểu:

```text
Event History
≠
Long-term audit archive
```

Một enterprise design có thể cần:

- trail;
- centralized logging;
- log bucket;
- retention;
- access controls;
- encryption;
- integrity protections;
- organizational aggregation;
- longer audit period.

Deep implementation có thể khác organization.

---

# LXVII. AWS BUDGETS

`AWS Budgets` giúp monitor cost/usage so với target.

AWS hiện hỗ trợ cost/usage tracking và actual/forecast-oriented notifications tùy budget configuration. [AWS Documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html?utm_source=chatgpt.com)

Một budget basic có:

```text
Scope
Period
Budget amount
Actual threshold
Forecast threshold
Notification target
```

Ví dụ lab:

```text
Monthly sandbox budget
→ notify at 50%
→ notify at 80%
→ notify at 100%
```

Threshold thực tế phụ thuộc organization.

---

# LXVIII. BUDGET KHÔNG PHẢI HARD SPENDING CAP

Một common misunderstanding:

> “Tôi set AWS Budget $20 thì AWS tự tắt tất cả ở $20.”

Không nên giả định như vậy.

Budget fundamentally là cost/usage monitoring/control mechanism; automated actions có thể được configured trong certain designs, nhưng không nên xem budget amount như universal hard billing cap. AWS Budgets có thể được cấu hình với alerts và trong một số trường hợp actions, nhưng hành vi phải được thiết kế rõ. [AWS Documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html?utm_source=chatgpt.com)

---

# LXIX. COST ALLOCATION

Cost allocation trả lời:

> Resource/cost này thuộc team/application/environment nào?

Một phần quan trọng là tagging.

Ví dụ:

```text
Environment=Lab
Application=CRM
Owner=InfraTeam
CostCenter=IT001
ManagedBy=Terraform
```

Sau đó cost visibility có thể group/filter theo tags/cost categories khi đã configured phù hợp.

AWS mô tả cost allocation tags/cost categories là phương tiện giúp phân bổ chi phí theo team, application, environment hoặc dimensions khác. [AWS Documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-create-filters.html?utm_source=chatgpt.com)

---

# LXX. COST DRIVERS CỦA DAY 5

Syllabus yêu cầu nhận diện ít nhất:

```text
EC2 cost
EBS cost
EBS snapshot cost
S3 storage cost
S3 requests/retrieval behavior
Data transfer
Public IPv4
Unused resources
```

---

# LXXI. EC2 COST THINKING

EC2 cost không chỉ phụ thuộc:

```text
instance is running?
```

Bạn còn phải xem:

- instance family/size;
- usage duration;
- purchasing model;
- attached storage;
- public IPv4;
- data transfer;
- additional services.

Infrastructure Operations L1/L2 không nhất thiết phải làm FinOps architecture, nhưng phải phát hiện obvious waste.

---

# LXXII. EBS COST THINKING

Một classic scenario:

```text
EC2 terminated
```

nhưng:

```text
EBS data volume remains
```

Nếu volume vẫn tồn tại, storage cost có thể tiếp tục.

Hoặc:

```text
Instance stopped
```

không nghĩa rằng attached persistent EBS volumes biến mất.

Do đó:

```text
Stop EC2
≠
Stop all cost
```

---

# LXXIII. SNAPSHOT COST THINKING

Snapshots rất hữu ích nhưng stale snapshot retention có thể tích tụ.

Do incremental architecture, cách cost behavior không đơn giản là:

```text
5 snapshots × full volume size
```

nhưng bạn vẫn cần retention policy và lifecycle governance.

Không nên:

```text
“Giữ mọi snapshot mãi mãi vì an toàn”
```

mà không xét:

- business retention;
- compliance;
- recovery need;
- cost.

---

# LXXIV. S3 COST THINKING

S3 cost có thể đến từ nhiều dimensions:

```text
storage capacity
storage class
requests
retrieval
lifecycle transitions
replication
data transfer
KMS requests
versions
old versions
```

Một versioned bucket có thể tăng storage mà người dùng không thấy khi chỉ nhìn current object.

Ví dụ:

```text
backup.zip = 10 GB

Version 1 = 10 GB
Version 2 = 10 GB
Version 3 = 10 GB
```

Ba versions là ba full object versions cho storage accounting, không phải simple block diff theo kiểu EBS snapshot. AWS documentation lưu ý mỗi S3 object version là complete object. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html?utm_source=chatgpt.com)

---

# LXXV. PUBLIC IPv4 CHARGES

Day 5 syllabus gọi riêng `public IPv4 charges` vì đây là cost item dễ bị bỏ quên.

Tại thời điểm kiểm tra hiện tại, AWS VPC pricing công bố charge cho public IPv4 address là **$0.005/hour cho mỗi public IPv4 address**, áp dụng cho cả in-use và idle public IPv4 addresses trong các phạm vi được AWS nêu. Pricing có thể thay đổi nên production cost estimation phải kiểm tra current Region/service pricing tại thời điểm thực hiện. [Amazon Web Services, Inc.](https://aws.amazon.com/vpc/pricing/?trk=public_post_comment-text&utm_source=chatgpt.com)

Operational lesson quan trọng hơn con số:

```text
Public IPv4 is not “free by default”.
```

Do đó unused public addresses là cost-review item.

---

# LXXVI. DATA TRANSFER COST

Không nên nói:

> “AWS transfer luôn tính phí.”

Cũng không nên nói:

> “AWS internal transfer luôn miễn phí.”

Pricing phụ thuộc:

- source/destination;
- Region;
- AZ architecture;
- service;
- direction;
- Internet;
- replication design;
- service-specific rules.

Vì vậy Infrastructure Operations phải:

```text
identify path
→ identify volume
→ consult current pricing
```

không nhớ một rule đơn giản rồi áp cho mọi trường hợp.

---

# LXXVII. UNUSED RESOURCE IDENTIFICATION

Các candidate cần review:

```text
Stopped EC2
Unattached EBS
Old snapshots
Unused Elastic/Public IPv4
Obsolete S3 backup versions
Old test buckets
Unused CloudWatch alarms/logs
Unused recovery resources
```

Nhưng từ khóa là:

**candidate**.

Unattached EBS volume có thể là:

- forgotten resource;
- intentional recovery volume;
- retained evidence;
- rollback volume;
- backup staging disk.

Do đó:

```text
Unused-looking
≠
Safe to delete
```

---

# LXXVIII. STOP vs TERMINATE

Đây là critical concept.

### Stop

EC2 compute ngừng chạy nhưng instance vẫn tồn tại.

Các persistent resources có thể tiếp tục tồn tại và incur cost.

### Terminate

Instance lifecycle đi tới deletion/termination.

Nhưng behavior của attached storage phải được kiểm tra:

- root volume `DeleteOnTermination`;
- data volumes;
- snapshots;
- external resources.

Không được dùng terminate như “power off”.

---

# LXXIX. SAFE RESOURCE CLEANUP

Cleanup workflow:

```text
1. Identify resource
2. Identify owner
3. Check tags
4. Check dependencies
5. Check ticket/change
6. Check backup requirement
7. Check rollback requirement
8. Check retention/legal requirement
9. Execute approved cleanup
10. Verify deletion/stop state
11. Record evidence
```

Đặc biệt:

```text
terminate
delete volume
delete snapshot
delete object version
delete bucket
disable key
```

đều có thể làm recovery mất đi.

---

# LXXX. CLOUDWATCH + CLOUDTRAIL + COST CORRELATION

Một mature operator không sử dụng từng service riêng.

Ví dụ incident:

> Application stopped responding lúc 10:15.

Bạn kiểm tra:

### CloudWatch

```text
CPU spiked?
EBS latency?
Network?
Alarm?
```

### CloudTrail

```text
Did someone stop/reboot/modify instance?
Who changed security/storage configuration?
```

### EBS

```text
Volume attached?
IO problem?
Filesystem available?
```

### S3/AWS Backup

```text
Recovery point available?
```

### Cost/resource state

```text
Was resource terminated as cleanup?
```

Nhiều incident chỉ rõ root cause khi correlate các nguồn.

---

# LXXXI. FAILURE SCENARIOS — LÝ THUYẾT

## Scenario 1 — Backup job xanh nhưng recovery thất bại

Thông tin:

```text
Backup status = Completed
Recovery = AccessDenied
```

Investigation:

```text
Recovery role?
Resource permission?
KMS key?
Key policy?
Vault policy?
Correct recovery point?
```

Conclusion:

> Backup creation success != recovery readiness.

---

## Scenario 2 — S3 object “bị delete”

Bucket versioning enabled.

Không nên restore ngay.

Trước tiên xác định:

```text
Delete marker?
Specific version permanently deleted?
Lifecycle expiration?
Object Lock state?
```

---

## Scenario 3 — Restored EBS volume rất chậm

Possible:

```text
volume initialization
wrong volume type
instance EBS limit
queue/latency issue
application workload
```

Không kết luận snapshot corrupted chỉ dựa trên performance.

---

## Scenario 4 — Không thấy S3 access trong CloudTrail Event history

Possible:

```text
It was data event
data event wasn't configured
event older than 90 days
wrong Region/time filter
wrong account
```

---

## Scenario 5 — EC2 stopped nhưng bill vẫn tăng

Check:

```text
EBS
snapshots
S3
public IPv4
data transfer
NAT/other dependencies
logs
```

Stop compute không đồng nghĩa toàn bộ stack hết cost.

---

# LXXXII. SECURITY MODEL CỦA DAY 5

Bạn nên hình dung security theo 6 layers:

```text
1. Identity
   IAM role / temporary credentials

2. Authorization
   IAM policy / bucket policy / KMS policy

3. Exposure control
   Block Public Access / network controls

4. Confidentiality
   Encryption

5. Integrity
   Checksums / hashes / versioning

6. Recoverability
   Backup / restore test / retention
```

Security không chỉ là:

```text
“Bucket private.”
```

Nếu encrypted backup không restore được thì security có thể mạnh nhưng availability objective lại thất bại.

---

# LXXXIII. LEAST PRIVILEGE CHO BACKUP/RECOVERY

Một enterprise design nên tránh role có:

```json
{
  "Action": "*",
  "Resource": "*"
}
```

chỉ để “cho backup chạy”.

Nên xác định:

```text
Backup operator needs?
Recovery operator needs?
Auditor needs?
Security admin needs?
Automation role needs?
```

AWS KMS cũng khuyến nghị cấp minimum permissions cần thiết cho principals sử dụng/manage KMS keys. [AWS Documentation](https://docs.aws.amazon.com/kms/latest/developerguide/iam-policies-best-practices.html?utm_source=chatgpt.com)

---

# LXXXIV. VERIFICATION PHILOSOPHY CỦA DAY 5

Mọi configuration phải trả lời được:

```text
What did I expect?
What actually happened?
How did I verify?
What evidence proves it?
```

Ví dụ S3 backup:

**Weak verification:**

```text
aws s3 cp returned no obvious error
```

**Better verification:**

```text
Object exists
Correct size
Correct version
Correct encryption
Expected checksum
Recovery test succeeded
```

---

# LXXXV. TROUBLESHOOTING PHILOSOPHY

Luôn đi từ read-only đến state-changing:

```text
Observe
↓
Identify scope
↓
Gather evidence
↓
Build hypothesis
↓
Read-only verification
↓
Minimal reversible correction
↓
Post-check
↓
Rollback if needed
```

Không:

```text
Try random change
↓
Try another random change
↓
Reboot
↓
Delete/recreate
```

---

# LXXXVI. EVIDENCE CỦA DAY 5

Syllabus sử dụng common evidence contract cho cả chương trình. Evidence item nên có:

- evidence ID;
- source system;
- resource/asset/account identifier;
- environment;
- ticket reference;
- collector;
- collection timestamp/timezone;
- query/time window;
- command/filter;
- format/encoding;
- file size;
- record count;
- SHA-256;
- classification;
- limitations/errors.

Raw export phải được giữ nguyên; working copy và redacted output cần tách biệt. Matching hash chứng minh bytes match baseline được ghi lại, **không chứng minh source hoàn chỉnh hoặc đúng sự thật**. Infrastructure_Operations_14_Da…

Trong Day 5, evidence tiêu biểu:

```text
EBS volume ID
Snapshot ID
EC2 instance ID
S3 bucket
Object key/version ID
Backup recovery point
KMS key ARN/ID where allowed
CloudWatch alarm
CloudTrail event ID
Budget configuration
RPO/RTO record
Hash manifest
Ticket/change reference
```

Không bao giờ đưa vào evidence:

```text
AWS secret access key
session token
password
private key
MFA seed
```

---

# LXXXVII. EVIDENCE CHO RESTORE

Một restore evidence package tốt có thể thể hiện:

```text
Recovery point:
rp-...

Recovery started:
2026-10-20T02:10:00Z

Recovery completed:
2026-10-20T02:23:00Z

Expected file count:
100

Actual file count:
100

Expected SHA-256:
...

Actual SHA-256:
...

Application-readable:
PASS

Observed RTO:
13 minutes

Observed RPO:
42 minutes

Permissions:
PASS

Encryption key dependency:
Verified
```

Đó là evidence mạnh hơn screenshot:

```text
Restore job = Completed
```

rất nhiều.

---

# LXXXVIII. ESCALATION BOUNDARIES

## L1 có thể thường xử lý

Tùy runbook:

- inspect resource state;
- verify mount;
- inspect metrics;
- inspect alarms;
- inspect CloudTrail;
- validate backup status;
- collect evidence;
- perform approved standard restore/check;
- identify cost anomaly;
- stop approved lab resources.

## L2 có thể thường xử lý

Tùy organization:

- deeper permission troubleshooting;
- filesystem/storage troubleshooting;
- backup restore issue;
- CloudWatch tuning;
- S3 lifecycle/version recovery;
- KMS dependency analysis;
- DR validation;
- controlled configuration correction.

## Escalate khi

- possible data loss;
- filesystem corruption;
- production restore;
- permanent object/version deletion;
- KMS key unavailable/disabled/deletion issue;
- backup coverage insufficient;
- RPO/RTO breach;
- unclear lifecycle deletion;
- public data exposure;
- cross-account policy issue outside authorization;
- large unexpected cost;
- compliance/retention issue;
- action requires permanent/destructive change.

---

# LXXXIX. COMMANDS CẦN NHẬN DIỆN TRONG PHẦN LÝ THUYẾT

Chưa phải lab, nhưng bạn nên biết command categories.

## EBS/AWS

Read-only examples:

```bash
aws ec2 describe-volumes
aws ec2 describe-snapshots
aws ec2 describe-instances
```

State-changing examples:

```bash
aws ec2 attach-volume
aws ec2 create-snapshot
aws ec2 create-volume
```

---

## Linux storage

Read-only:

```bash
lsblk
lsblk -f
blkid
findmnt
df -hT
file -s /dev/<device>
```

Potentially changing/destructive:

```bash
mkfs
mount
umount
```

Editing:

```text
/etc/fstab
```

---

## S3

Read-only/config inspection concepts:

```bash
aws s3api get-bucket-versioning
aws s3api get-public-access-block
aws s3api get-bucket-encryption
aws s3api get-bucket-lifecycle-configuration
aws s3api list-object-versions
aws s3api head-object
```

State-changing:

```bash
aws s3 cp
aws s3api put-bucket-versioning
aws s3api put-public-access-block
...
```

Trong practical module chúng ta sẽ phân tích từng command, option và expected output.

---

# XC. CLOUDWATCH READ-ONLY OPERATIONS

Conceptual commands:

```bash
aws cloudwatch list-metrics
aws cloudwatch describe-alarms
aws cloudwatch get-metric-data
```

Bạn cần biết chúng giúp trả lời:

```text
Metric tồn tại?
Alarm state?
Alarm config?
Historical datapoints?
```

---

# XCI. CLOUDTRAIL READ-ONLY OPERATIONS

Ví dụ:

```bash
aws cloudtrail lookup-events
```

Nhưng nhớ:

```text
lookup-events
≠
complete universal audit history
```

Nó bị giới hạn bởi CloudTrail event coverage/design.

---

# XCII. OPERATIONAL CONSOLIDATION — DAY 4 + DAY 5

Đây là phần cuối rất quan trọng.

Day 4:

```text
IAM
VPC
Subnet
Route
Security Group
EC2
SSH
```

Day 5 thêm:

```text
EBS
S3
Backup
DR
CloudWatch
CloudTrail
Cost
```

Kết quả:

```text
                         AWS ACCOUNT
                              │
                   ┌──────────┴──────────┐
                   │                     │
                  IAM                   VPC
                   │                     │
                   │             ┌───────┴────────┐
                   │          Public           Private
                   │          Subnet            Subnet
                   │             │
                   │            EC2
                   │             │
                   │            EBS
                   │             │
                   │          Snapshot
                   │
                   ├─────────── S3 Backup
                   │               │
                   │       Versioning/Encryption
                   │               │
                   │           Lifecycle
                   │               │
                   │         Replication/Lock
                   │
                   ├──────── CloudWatch
                   │
                   ├──────── CloudTrail
                   │
                   └──────── AWS Backup

                       Governance layer:
                    RPO / RTO / Cost / Audit
```

Đây mới là operational view đúng của module AWS Days 4–5. Program summary cũng xác định Days 4–5 bao phủ `IAM, VPC, EC2, EBS, S3 backup/DR, CloudWatch, CloudTrail, budgets`. Infrastructure_Operations_14_Da…

---

# XCIII. 15 NGUYÊN TẮC PHẢI NHỚ SAU MODULE LÝ THUYẾT

1. **EBS volume không phải filesystem.** Attach volume chỉ mới tạo block-device relationship.

2. **Không bao giờ `mkfs` restored volume trước khi kiểm tra filesystem hiện có.**

3. **Persistent mount phải được verify**, không chỉ mount thành công trong current session.

4. **EBS snapshot là backup point, nhưng snapshot success chưa chứng minh application recovery.**

5. **Backup success không bằng restore success.**

6. **Restore success không bằng data-integrity success.**

7. **Data-integrity success chưa chắc bằng application-recovery success.**

8. **RPO nói về recoverable data point; RTO nói về recovery duration.**

9. **Versioning, replication, retention và backup là bốn concepts khác nhau.**

10. **S3 Block Public Access, encryption và IAM policy giải quyết các security problems khác nhau.**

11. **SSE-KMS tạo key dependency; DR phải verify key availability/permission.**

12. **Generic S3 `ETag` không phải universal hash.**

13. **CloudWatch trả lời system đang hoạt động thế nào; CloudTrail chủ yếu giúp trả lời account/API activity đã xảy ra thế nào.**

14. **CloudTrail Event history chỉ bao phủ 90 ngày management events theo Region và không phải complete S3 object-access archive.** [AWS Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events-console.html?utm_source=chatgpt.com)

15. **Stopped/unused-looking resource vẫn có thể tiếp tục phát sinh cost hoặc có business dependency; cleanup phải được controlled và evidenced.**

---

# XCIV. THEORY COMPLETION CHECKLIST

Sau khi học xong Module Lý thuyết Day 5, bạn nên tự tin giải thích được:

- [ ] EBS là block storage gì và khác S3 thế nào.
- [ ] EBS volume lifecycle.
- [ ] `gp3`, `gp2`, `io2`, `io1`, `st1`, `sc1` dùng cho loại workload nào.
- [ ] IOPS, throughput, latency khác nhau ra sao.
- [ ] Vì sao volume phải cùng AZ với EC2 khi attach theo workflow cơ bản.
- [ ] Device mapping AWS và Linux khác nhau thế nào.
- [ ] Block device, partition, filesystem và mount point khác nhau thế nào.
- [ ] Vì sao `mkfs` có thể phá recovery.
- [ ] Persistent mount `/etc/fstab` và vai trò của UUID.
- [ ] EBS snapshot hoạt động ở mức concept như thế nào.
- [ ] Vì sao backup/snapshot completed chưa chứng minh recovery.
- [ ] Restore snapshot thành new EBS volume.
- [ ] Volume initialization sau restore.
- [ ] RPO và RTO.
- [ ] Target RPO/RTO so với observed RPO/RTO.
- [ ] S3 bucket/object/key/version.
- [ ] IAM policy và bucket policy.
- [ ] Explicit Deny.
- [ ] Block Public Access.
- [ ] SSE-S3 và SSE-KMS.
- [ ] KMS key dependency trong recovery.
- [ ] S3 Versioning.
- [ ] Delete marker.
- [ ] Lifecycle transition/expiration.
- [ ] Storage class trade-offs.
- [ ] Replication.
- [ ] Vì sao replication không bằng backup.
- [ ] Object Lock/WORM awareness.
- [ ] AWS Backup plan/recovery point/vault/retention.
- [ ] Restore testing.
- [ ] Data-integrity verification.
- [ ] Vì sao generic ETag không phải universal hash.
- [ ] CloudWatch metrics.
- [ ] EC2/EBS monitoring.
- [ ] IOPS/throughput/queue/latency reasoning.
- [ ] CloudWatch alarm states.
- [ ] `OK`, `ALARM`, `INSUFFICIENT_DATA`.
- [ ] CloudWatch và CloudTrail khác nhau.
- [ ] CloudTrail management events và data events.
- [ ] CloudTrail Event history 90-day limitation.
- [ ] Audit retention gap.
- [ ] AWS Budgets.
- [ ] Cost allocation/tags.
- [ ] EC2/EBS/S3/snapshot/data-transfer/public-IPv4 cost drivers.
- [ ] Stop và terminate khác nhau.
- [ ] Unused resource identification.
- [ ] Controlled resource cleanup.
- [ ] Evidence requirements.
- [ ] Least privilege.
- [ ] L1/L2/escalation boundaries.

Nếu toàn bộ các mục trên đã rõ, **Module Lý thuyết Day 5 đã cover đầy đủ phạm vi syllabus**; phần tiếp theo có thể chuyển hoàn toàn sang **Module Thực hành Day 5**, nơi chúng ta sẽ thực hiện một chuỗi lab liên tục từ `EBS attach → filesystem → persistent mount → snapshot → restore → S3 secure backup → version recovery → checksum → RPO/RTO → CloudWatch → CloudTrail → Budget → cost review → cleanup`, kèm expected output, failure injection, troubleshooting, evidence và rollback.
