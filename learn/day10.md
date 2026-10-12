# Day 10 — Amazon EBS, Filesystem Mount, Snapshot, Restore, Backup Verification & Data Retention

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 10 thuộc phần **AWS Cloud Fundamentals** và yêu cầu chính xác:

**Concept/Lecture:** `EBS volumes, filesystem mount, snapshots, restore, backup verification, and data-retention basics.`  
**Assignment/Lab:** `Attach and mount EBS, create a snapshot, restore data, and record recovery evidence.` 00_MASTER_SYLLABUS_Java_Fresher…

Vì vậy, trong Day 10 chúng ta **bắt buộc** phải hiểu và thực hành toàn bộ chuỗi:

```text
EC2
  ↓
EBS Volume
  ↓ attach
Linux block device
  ↓ filesystem
Mount point
  ↓
Data
  ↓ snapshot
EBS Snapshot
  ↓ restore
New EBS Volume
  ↓ attach + mount
Recovered Data
  ↓
Verification + Recovery Evidence
```

Phần **Mandatory** hôm nay là EBS volume, filesystem/mount, snapshot, restore, backup verification và data-retention basics. Tôi sẽ bổ sung thêm một số kiến thức như encryption, RPO/RTO, Data Lifecycle Manager, Recycle Bin, Snapshot Archive, initialization và Fast Snapshot Restore; các phần đó sẽ được đánh dấu **Supplementary / Beyond the explicit syllabus**.

---

# 2. Objectives và prerequisites

Kết thúc Day 10, bạn không chỉ cần biết bấm **Create volume** và **Create snapshot**. Bạn phải có khả năng vận hành theo góc nhìn administrator:

| Năng lực          | Mục tiêu                                                                       |
| ----------------- | ------------------------------------------------------------------------------ |
| EBS               | Hiểu EBS volume là gì và quan hệ với EC2                                       |
| Availability Zone | Biết vì sao volume và EC2 phải cùng AZ                                         |
| Linux storage     | Phân biệt disk/block device → filesystem → mount point → file                  |
| Device discovery  | Xác định đúng EBS device, đặc biệt trên Nitro/NVMe                             |
| Filesystem        | Kiểm tra trước khi `mkfs`; tạo filesystem an toàn                              |
| Mount             | Mount tạm thời và persistent qua `/etc/fstab`                                  |
| Snapshot          | Hiểu point-in-time backup và incremental behavior                              |
| Consistency       | Biết khi nào phải stop writes/unmount trước snapshot                           |
| Restore           | Tạo volume mới từ snapshot và lấy dữ liệu trở lại                              |
| Verification      | Chứng minh dữ liệu restore đúng bằng checksum                                  |
| Recovery          | Phục hồi file hoặc thay thế whole volume                                       |
| Retention         | Hiểu tại sao backup phải có policy giữ/xóa                                     |
| Troubleshooting   | Điều tra AZ mismatch, wrong device, mount/fstab, permissions, snapshot/restore |
| Operations        | Thu thập evidence, cleanup, rollback và escalation                             |

### Lab environment giả định

Tôi sẽ dùng môi trường tham chiếu:

```text
AWS CLI: v2
EC2: Linux instance từ Day 9
Region example: ap-southeast-1
Filesystem dùng trong lab: ext4
EBS type: gp3
Lab data volume: 8 GiB
Mount point: /data
Restore mount point: /restore
```

Amazon Linux 2023 hoặc Ubuntu đều phù hợp. Tên device thực tế có thể khác nhau và **không được copy cứng `/dev/nvme1n1` mà không kiểm tra**.

---

# 3. Mental Model quan trọng nhất của Day 10

Có bốn tầng khác nhau mà fresher thường nhầm:

```text
AWS layer
+----------------------+
| EBS Volume           |
| vol-012345...        |
+----------+-----------+
           |
           | attach
           v
Linux block-device layer
+----------------------+
| /dev/nvme1n1         |
+----------+-----------+
           |
           | mkfs.ext4
           v
Filesystem layer
+----------------------+
| ext4 filesystem      |
| UUID=abc-123...      |
+----------+-----------+
           |
           | mount
           v
Filesystem namespace
+----------------------+
| /data                |
| /data/file.txt       |
+----------------------+
```

Điểm phải nhớ:

**EBS Volume ≠ filesystem ≠ mount point ≠ directory.**

EBS cung cấp **block storage**. Linux nhìn thấy block storage này như disk/device. Trên device đó ta tạo filesystem như `ext4` hoặc `xfs`. Sau đó filesystem mới được gắn vào Linux directory tree thông qua `mount`.

Đây chính là lý do:

```bash
aws ec2 attach-volume ...
```

không có nghĩa rằng bạn lập tức có `/data`.

Bạn vẫn phải thực hiện:

```text
attach
→ detect device
→ inspect existing content
→ filesystem nếu cần
→ create mount point
→ mount
→ permissions
→ persistence
→ verify
```

---

# 4. Deep Theory — Amazon EBS Volume

## 4.1 EBS là gì?

Amazon Elastic Block Store — **EBS** — cung cấp block storage cho EC2.

Một cách tư duy đơn giản:

```text
EC2 = computer/server
EBS = disk attached to that server
```

Nhưng đây là network-attached block storage do AWS quản lý, không phải ổ SSD vật lý mà bạn tự cắm.

Một EBS volume có lifecycle riêng:

```text
creating
   ↓
available
   ↓ attach
in-use
   ↓ detach
available
   ↓
delete
```

Volume phải ở trạng thái `available` trước khi attach; khi attach thì thường ở `in-use`. Volume và EC2 mà nó attach vào phải nằm trong **cùng Availability Zone**. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-volume.html?utm_source=chatgpt.com)

Ví dụ:

```text
EC2:
ap-southeast-1a

EBS:
ap-southeast-1a     → attach được
ap-southeast-1b     → không attach trực tiếp được
```

Đây là một câu lý thuyết rất dễ xuất hiện trong assessment.

---

# 5. Volume → block device → filesystem

Giả sử console nói:

```text
Volume ID: vol-0123456789abcdef0
Device: /dev/sdf
```

Bạn SSH vào Nitro-based EC2 và chạy:

```bash
lsblk
```

nhưng lại thấy:

```text
nvme0n1
nvme1n1
```

Bạn có thể nghĩ:

> `/dev/sdf` biến đâu rồi?

Không có lỗi.

Trên Nitro-based EC2, EBS volumes được expose cho Linux dưới dạng **NVMe block devices**, thường:

```text
/dev/nvme0n1
/dev/nvme1n1
/dev/nvme2n1
...
```

Tên NVMe còn có thể được enumerate khác nhau sau reboot, vì vậy không nên thiết kế persistent mount dựa hoàn toàn vào `/dev/nvme1n1`. AWS khuyến nghị dùng stable identifiers như filesystem UUID cho `/etc/fstab`. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/identify-nvme-ebs-device.html?utm_source=chatgpt.com)

Đây là một insight cực kỳ quan trọng:

```text
AWS attachment name       Linux actual device
/dev/sdf              ≠   necessarily /dev/sdf

Nitro:
                         /dev/nvme1n1
```

### Cách kiểm tra

```bash
lsblk
```

Tốt hơn:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,SERIAL
```

Trên Amazon Linux, với EBS/NVMe bạn còn có thể kiểm tra mapping:

```bash
sudo ebsnvme-id /dev/nvme1n1
```

AWS cũng chỉ ra rằng EBS volume ID được expose trong serial information của NVMe volume. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/identify-nvme-ebs-device.html?utm_source=chatgpt.com)

---

# 6. Quy tắc an toàn quan trọng nhất: kiểm tra trước `mkfs`

`mkfs` có nghĩa là:

> Tạo filesystem mới.

Ví dụ:

```bash
sudo mkfs.ext4 /dev/nvme1n1
```

Nếu disk chứa dữ liệu, thao tác này có thể **phá metadata/filesystem hiện tại và làm mất dữ liệu**.

Vì vậy flow administrator đúng là:

```text
Identify device
      ↓
Check existing filesystem
      ↓
No filesystem?
 ├─ Yes → mkfs
 └─ No  → STOP, investigate/mount existing filesystem
```

Kiểm tra bằng:

```bash
lsblk -f
```

và:

```bash
sudo file -s /dev/nvme1n1
```

hoặc:

```bash
sudo blkid /dev/nvme1n1
```

Nếu:

```text
/dev/nvme1n1: data
```

thì volume chưa có recognized filesystem.

Nếu thấy:

```text
TYPE="ext4"
```

hoặc:

```text
TYPE="xfs"
```

thì **không chạy `mkfs`**.

AWS cũng hướng dẫn phải kiểm tra filesystem bằng `lsblk -f`/`file -s` trước khi format volume mới. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html?utm_source=chatgpt.com)

Đặc biệt:

> Volume restore từ snapshot **đã có filesystem và dữ liệu**. Chạy `mkfs` lên volume vừa restore chính là xóa thứ mà bạn vừa restore.

---

# 7. Filesystem và mount

Giả sử volume mới hoàn toàn:

```bash
sudo mkfs.ext4 -L DAY10_DATA /dev/nvme1n1
```

Ta có:

```text
EBS volume
    ↓
/dev/nvme1n1
    ↓
ext4 filesystem
```

Tuy nhiên filesystem vẫn chưa xuất hiện trong directory tree.

Tạo mount point:

```bash
sudo mkdir -p /data
```

Mount:

```bash
sudo mount /dev/nvme1n1 /data
```

Bây giờ:

```text
/dev/nvme1n1
     ↓ mounted at
/data
```

Kiểm tra:

```bash
findmnt /data
```

```bash
df -hT /data
```

```bash
lsblk -f
```

### Expected invariant

Bạn phải chứng minh được:

```text
TARGET  SOURCE          FSTYPE
/data   /dev/nvme...    ext4
```

Tên chính xác device không phải invariant.

Invariant là:

> đúng EBS volume → đúng filesystem → mounted tại `/data`.

---

# 8. Persistent mount và `/etc/fstab`

Lệnh:

```bash
sudo mount /dev/nvme1n1 /data
```

chỉ tạo runtime mount.

Reboot máy có thể mất mount.

Để tự động mount lúc boot, Linux sử dụng:

```text
/etc/fstab
```

Nhưng AWS khuyến nghị sử dụng filesystem **UUID**, không phụ thuộc vào NVMe device name, vì device name có thể thay đổi. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html?utm_source=chatgpt.com)

Lấy UUID:

```bash
sudo blkid /dev/nvme1n1
```

Ví dụ output:

```text
/dev/nvme1n1:
UUID="91fce181-6c2d-44af-a47b-123456789abc"
TYPE="ext4"
```

Backup trước:

```bash
sudo cp -a /etc/fstab /etc/fstab.day10.bak
```

Entry:

```text
UUID=91fce181-6c2d-44af-a47b-123456789abc /data ext4 defaults,nofail 0 2
```

### Ý nghĩa

| Field             | Ý nghĩa                |
| ----------------- | ---------------------- |
| `UUID=...`        | Filesystem cần mount   |
| `/data`           | Mount point            |
| `ext4`            | Filesystem type        |
| `defaults,nofail` | Mount options          |
| `0`               | dump field             |
| `2`               | filesystem check order |

`nofail` đặc biệt hữu ích với non-root EBS data volume vì instance có thể vẫn boot dù volume không có mặt. AWS cũng cảnh báo rằng lỗi trong `/etc/fstab` có khả năng khiến hệ thống không boot bình thường, vì vậy phải kiểm tra trước reboot. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html?utm_source=chatgpt.com)

### Validation bắt buộc trước reboot

```bash
sudo umount /data
sudo mount -a
```

Nếu không có error:

```bash
findmnt /data
df -hT /data
```

Chỉ sau khi pass bước này mới được coi cấu hình `/etc/fstab` hợp lệ.

**Không bao giờ sửa `/etc/fstab` xong rồi reboot ngay mà không chạy `mount -a`.**

---

# 9. Snapshot là gì?

EBS Snapshot là point-in-time backup của block data trong EBS volume.

Mental model:

```text
EBS Volume
2026-10-12 10:00

file-A
file-B
file-C
       |
       | snapshot
       v

Snapshot S1
State at 10:00
```

Sau đó bạn xóa `file-B`:

```text
Current volume:
file-A
file-C
```

Snapshot S1 vẫn đại diện cho state tại thời điểm snapshot.

Restore S1:

```text
new EBS volume
       ↓
file-A
file-B
file-C
```

Điểm cực kỳ quan trọng:

> Snapshot không phải một disk đang attach mà bạn trực tiếp mount như EBS volume.

Flow restore thông thường là:

```text
Snapshot
   ↓ Create Volume
New EBS Volume
   ↓ Attach
EC2
   ↓ Mount
Filesystem
   ↓
Recovered files
```

---

# 10. Full và incremental snapshot

Snapshot đầu tiên của một EBS volume chứa tất cả các blocks đã được ghi.

Các snapshot tiếp theo được lưu theo cơ chế incremental: chỉ các changed/new blocks kể từ snapshot trước cần được lưu thêm. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/how_snapshots_work.html?utm_source=chatgpt.com)

Ví dụ conceptual:

```text
Volume:

Block A
Block B
Block C
```

Snapshot 1:

```text
A
B
C
```

Sau đó chỉ C thay đổi và D được thêm:

```text
A
B
C'
D
```

Snapshot 2 về mặt incremental storage chỉ cần thêm các changed blocks liên quan:

```text
C'
D
```

Nhưng khi bạn restore Snapshot 2, AWS reconstruct state hoàn chỉnh:

```text
A
B
C'
D
```

Bạn **không phải tự giữ snapshot 1 chỉ để snapshot 2 restore được**. Khi xóa incremental snapshots, AWS quản lý các referenced blocks và chỉ loại bỏ data block khi không còn snapshot nào cần nó. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-deleting-snapshot.html?utm_source=chatgpt.com)

Đây là một misconception phổ biến:

> “Xóa snapshot cũ sẽ làm snapshot mới hỏng.”

Sai. AWS duy trì snapshot lineage/block references để các snapshot còn tồn tại vẫn restorable.

---

# 11. Snapshot consistency

Đây là phần operations rất quan trọng.

Giả sử application đang thực hiện:

```text
write file A
write DB page B
update metadata
flush transaction log
```

Bạn tạo snapshot giữa chừng.

Ở block-storage layer, snapshot vẫn có một point-in-time block state, nhưng application data có thể không đại diện cho một transaction/application-consistent state mà bạn mong muốn.

Với data volume, AWS khuyến nghị **pause writes** trước khi snapshot. Nếu không pause writes được thì nên unmount volume trước; sau khi snapshot đã đi vào `pending`, volume có thể được remount và writes có thể tiếp tục. Đối với root volume, AWS khuyến nghị stop instance để tăng consistency. [AWS Documentation](https://docs.aws.amazon.com/en_en/ebs/latest/userguide/ebs-creating-snapshot.html?utm_source=chatgpt.com)

Trong lab hôm nay ta sẽ sử dụng cách an toàn:

```text
write test data
↓
sync
↓
unmount
↓
create snapshot
↓
snapshot enters pending
↓
mount again
```

---

# 12. Backup tồn tại ≠ Backup đã được verify

Đây là một trong những bài học quan trọng nhất Day 10.

Giả sử:

```bash
aws ec2 describe-snapshots ...
```

cho:

```text
State = completed
```

Điều đó chứng minh:

> AWS đã hoàn thành snapshot operation.

Nó **không tự động chứng minh**:

```text
Dữ liệu đúng?
Filesystem mount được?
File quan trọng còn nguyên?
Application chạy được?
Restore procedure có hoạt động?
Operator biết restore thế nào?
```

Một backup tốt phải được kiểm thử thông qua recovery.

Một verification chain tốt:

```text
Snapshot completed
       ↓
Create test restore volume
       ↓
Attach
       ↓
Mount
       ↓
Locate expected data
       ↓
Checksum / integrity check
       ↓
Application-level validation nếu phù hợp
       ↓
Record evidence
```

Đây cũng chính là lý do Assignment Day 10 yêu cầu **restore data + recovery evidence**, chứ syllabus không dừng ở `create snapshot`. 00_MASTER_SYLLABUS_Java_Fresher…

---

# 13. Restore EBS snapshot

Một snapshot có thể tạo ra EBS volume mới.

Ví dụ:

```text
snap-AAA
   ↓
CreateVolume
   ↓
vol-BBB
```

Volume mới phải được tạo trong **Availability Zone của EC2 mà bạn định attach**. Volume sau đó có thể attach vào EC2 trong cùng AZ. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-restoring-volume.html?utm_source=chatgpt.com)

Điều này tạo ra hai recovery strategy quan trọng.

### File-level recovery

```text
Production volume
       +
Restored volume

mount restore at /restore
       ↓
find missing file
       ↓
copy file back
```

Đây là cách lab Day 10 sẽ làm.

### Whole-volume replacement

```text
Old/broken volume
       ↓
stop writes
unmount
detach
       ↓
Restored volume
attach
mount
       ↓
validate service
```

AWS documentation cũng mô tả restored volume có thể dùng để lấy lại file/directory hoặc thay thế damaged/corrupted volume toàn bộ. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-restoring-volume.html?utm_source=chatgpt.com)

---

# 14. Một nuance quan trọng khi restore: initialization

**Supplementary / Beyond the explicit syllabus**

Volume tạo từ snapshot có thể chưa ngay lập tức có toàn bộ block materialized trên underlying EBS storage.

AWS gọi quá trình này là **volume initialization**. Khi một block từ snapshot được truy cập lần đầu, dữ liệu cần được đưa về volume; do đó trong thời gian initialization có thể xuất hiện I/O latency cao hơn và performance thấp hơn. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/initalize-volume.html?utm_source=chatgpt.com)

Current AWS còn hỗ trợ các cơ chế như:

```text
default initialization
Provisioned Rate for Volume Initialization
Fast Snapshot Restore
```

Provisioned Rate hiện hỗ trợ rate từ **100–300 MiB/s**, còn Fast Snapshot Restore có thể tạo restored volume đã initialized ngay từ đầu. Đây là operations/performance enhancement, không phải yêu cầu bắt buộc Day 10. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/initalize-volume.html?utm_source=chatgpt.com)

Trong fresher lab hôm nay:

> Không cần enable Fast Snapshot Restore.

Nó có additional cost và không cần thiết cho một lab nhỏ. [AWS Documentation](https://docs.aws.amazon.com/en_en/ebs/latest/userguide/ebs-fast-snapshot-restore.html?utm_source=chatgpt.com)

---

# 15. EBS encryption

**Supplementary / Beyond the explicit syllabus nhưng rất quan trọng cho security.**

EBS hỗ trợ encryption bằng AWS KMS.

Encryption bảo vệ:

```text
EBS Volume
EBS snapshot
I/O giữa EC2 host và attached EBS storage
```

Snapshot của encrypted volume cũng encrypted; volume tạo từ encrypted snapshot vẫn encrypted. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-encryption.html?utm_source=chatgpt.com)

Trong lab nên chọn:

```text
Encrypted = true
```

và dùng default EBS KMS key nếu tổ chức/lab không yêu cầu customer-managed key.

Một operational detail cần nhớ:

> KMS permissions cũng là một dependency của recovery.

Bạn có snapshot nhưng không còn quyền sử dụng required KMS key thì restore/attach workflow có thể thất bại.

Vì vậy backup strategy phải xem:

```text
data
+
snapshot
+
IAM access
+
KMS access
```

chứ không chỉ snapshot ID.

---

# 16. Data-retention basics

Mandatory syllabus yêu cầu **data-retention basics**.

Data retention trả lời:

```text
Backup bao lâu một lần?
Giữ bao lâu?
Backup nào được xóa?
Ai có quyền xóa?
Backup dài hạn để ở đâu?
Restore đã được test chưa?
```

Ví dụ một **policy minh họa**, không phải policy bắt buộc:

| Backup                 |                 Frequency |        Retention |
| ---------------------- | ------------------------: | ---------------: |
| Operational snapshots  |                     Daily |           7 days |
| Weekly recovery points |                    Weekly |          4 weeks |
| Monthly                |                   Monthly |        12 months |
| Compliance archive     | Theo business requirement | Có thể nhiều năm |

Không có retention policy universal. Nó phải xuất phát từ business requirement, regulatory requirement, risk và cost.

### Supplementary: RPO và RTO

**RPO — Recovery Point Objective**

Nếu snapshot mỗi 24 giờ:

```text
snapshot       failure
   |--------------|
       18 hours
```

Worst-case data loss có thể gần 24 giờ.

RPO trả lời:

> Chấp nhận mất bao nhiêu dữ liệu?

**RTO — Recovery Time Objective**

RTO trả lời:

> Sau incident, dịch vụ phải được phục hồi trong bao lâu?

Snapshot mỗi 5 phút chưa chắc RTO tốt nếu procedure restore mất 6 tiếng.

Vì vậy:

```text
Backup frequency → ảnh hưởng RPO
Restore capability → ảnh hưởng RTO
```

---

# 17. Automated retention — Supplementary

Amazon Data Lifecycle Manager có thể tự động hóa creation, retention và deletion của EBS snapshots. Custom policy có thể target resources bằng tags và hỗ trợ multiple schedules. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/dlm-elements.html?utm_source=chatgpt.com)

Conceptual example:

```text
Volume tag:
Backup=Daily
       ↓
DLM Policy
       ↓
Snapshot daily
       ↓
Keep N days
       ↓
Delete old snapshots
```

Điều này tốt hơn:

```text
Admin nhớ thì backup
Admin quên thì thôi
```

AWS còn cung cấp EBS Snapshot Archive cho long-term, rarely-accessed snapshots. Archive biến snapshot thành full snapshot trong archive tier và được thiết kế cho retention dài hạn; AWS nêu use case cho monthly/quarterly/yearly snapshots. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/snapshot-archive.html?utm_source=chatgpt.com)

Recycle Bin cũng có thể áp dụng retention rules để giảm nguy cơ accidental deletion của volumes/snapshots. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/recycle-bin-concepts.html?utm_source=chatgpt.com)

Những tính năng này **không cần cấu hình để pass lab Day 10**, nhưng bạn phải hiểu chúng thuộc broader backup lifecycle.

---

# 18. Command Reference Day 10

| Command                               | Mục đích                        | Thay đổi state?     |
| ------------------------------------- | ------------------------------- | ------------------- |
| `aws ec2 describe-volumes`            | Inspect EBS                     | Không               |
| `aws ec2 create-volume`               | Tạo EBS                         | Có                  |
| `aws ec2 attach-volume`               | Attach volume                   | Có                  |
| `lsblk`                               | Inspect Linux block devices     | Không               |
| `blkid`                               | Inspect filesystem UUID/type    | Không               |
| `file -s DEVICE`                      | Detect existing filesystem/data | Không               |
| `mkfs.ext4 DEVICE`                    | Tạo filesystem                  | **Có, destructive** |
| `mkdir`                               | Tạo mount point                 | Có                  |
| `mount`                               | Mount filesystem                | Có runtime state    |
| `umount`                              | Unmount filesystem              | Có runtime state    |
| `findmnt`                             | Inspect mount                   | Không               |
| `df -hT`                              | Capacity/filesystem check       | Không               |
| `sha256sum`                           | Integrity verification          | Không               |
| `aws ec2 create-snapshot`             | Tạo EBS snapshot                | Có                  |
| `aws ec2 create-volume --snapshot-id` | Restore snapshot thành volume   | Có                  |
| `aws ec2 detach-volume`               | Detach EBS                      | Có                  |
| `aws ec2 delete-volume`               | Xóa EBS                         | **Destructive**     |
| `aws ec2 delete-snapshot`             | Xóa backup                      | **Destructive**     |

AWS CLI v2 hiện có waiter cho `volume-available`, `volume-in-use` và `snapshot-completed`, vì vậy lab có thể đợi AWS resource đạt đúng state thay vì dùng `sleep` tùy tiện. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/ec2/wait/snapshot-completed.html?utm_source=chatgpt.com)

---

# 19. GUIDED LAB — Full Day 10

Đây là lab chính thức theo syllabus.

## Phase A — Safety pre-check

**CẢNH BÁO:** Lab có `mkfs`, `rm`, detach và delete resources. Chỉ làm trên lab/non-production. Luôn xác định đúng volume trước `mkfs`. Không dùng production data.

Đầu tiên:

```bash
aws sts get-caller-identity
```

Kiểm tra CLI:

```bash
aws --version
```

Set Region:

```bash
export REGION=ap-southeast-1
```

Set EC2 instance ID Day 9:

```bash
export INSTANCE_ID=i-xxxxxxxxxxxxxxxxx
```

Lấy Availability Zone:

```bash
export AZ=$(
  aws ec2 describe-instances \
    --instance-ids "$INSTANCE_ID" \
    --region "$REGION" \
    --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
    --output text
)
```

Kiểm tra:

```bash
printf 'REGION=%s\nINSTANCE=%s\nAZ=%s\n' \
  "$REGION" "$INSTANCE_ID" "$AZ"
```

**Expected invariant:**

```text
REGION != empty
INSTANCE_ID đúng EC2 Day 9
AZ != None
```

---

# 20. Phase B — Create encrypted EBS volume

Tạo 8 GiB `gp3`:

```bash
export VOLUME_ID=$(
  aws ec2 create-volume \
    --availability-zone "$AZ" \
    --size 8 \
    --volume-type gp3 \
    --encrypted \
    --tag-specifications \
      'ResourceType=volume,Tags=[{Key=Name,Value=day10-data},{Key=Project,Value=java-fresher-day10},{Key=Environment,Value=lab}]' \
    --region "$REGION" \
    --query 'VolumeId' \
    --output text
)
```

Kiểm tra:

```bash
echo "$VOLUME_ID"
```

Chờ volume available:

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$REGION"
```

Inspect:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --region "$REGION" \
  --query 'Volumes[0].{ID:VolumeId,State:State,AZ:AvailabilityZone,Type:VolumeType,Size:Size,Encrypted:Encrypted}' \
  --output table
```

**Expected invariant:**

```text
State     = available
AZ        = EC2 AZ
Type      = gp3
Size      = 8
Encrypted = True
```

---

# 21. Phase C — Attach EBS vào EC2

```bash
aws ec2 attach-volume \
  --volume-id "$VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf \
  --region "$REGION"
```

Chờ:

```bash
aws ec2 wait volume-in-use \
  --volume-ids "$VOLUME_ID" \
  --region "$REGION"
```

AWS attachment name `/dev/sdf` **không đảm bảo Linux sẽ thấy `/dev/sdf` trên Nitro**. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-attaching-volume.html?utm_source=chatgpt.com)

---

# 22. Phase D — Detect đúng Linux device

SSH vào EC2.

Trước tiên:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,SERIAL
```

Ví dụ output:

```text
NAME        SIZE FSTYPE MOUNTPOINTS
nvme0n1       8G
└─nvme0n1p1   8G xfs    /
nvme1n1       8G
```

Đây chỉ là **Example output**.

Không được kết luận mọi lab sẽ là `/dev/nvme1n1`.

Trên Amazon Linux có thể:

```bash
sudo ebsnvme-id /dev/nvme1n1
```

Ví dụ:

```text
Volume ID: vol-0123456789abcdef0
/dev/sdf
```

Sau khi đã xác nhận:

```bash
export DEVICE=/dev/nvme1n1
```

Thay giá trị bằng **device thực tế của bạn**.

---

# 23. Phase E — Inspect trước khi format

```bash
sudo lsblk -f "$DEVICE"
```

```bash
sudo file -s "$DEVICE"
```

```bash
sudo blkid "$DEVICE" || true
```

Volume mới thường chưa có filesystem.

Ví dụ:

```text
/dev/nvme1n1: data
```

Nếu bạn thấy:

```text
ext4
xfs
```

**STOP.**

Không chạy bước tiếp theo cho tới khi xác định tại sao volume đã có filesystem.

---

# 24. Phase F — Create filesystem

Bây giờ mới format:

```bash
sudo mkfs.ext4 -L DAY10_DATA "$DEVICE"
```

Sau đó:

```bash
sudo blkid "$DEVICE"
```

Ví dụ:

```text
/dev/nvme1n1:
LABEL="DAY10_DATA"
UUID="..."
TYPE="ext4"
```

**Evidence nên lưu:** command + output của `blkid`.

---

# 25. Phase G — Mount

```bash
sudo mkdir -p /data
```

```bash
sudo mount "$DEVICE" /data
```

Verify:

```bash
findmnt /data
```

```bash
df -hT /data
```

```bash
lsblk -f
```

Expected invariant:

```text
/data exists
/data mounted from intended EBS device
filesystem = ext4
```

---

# 26. Phase H — Permissions

Kiểm tra:

```bash
ls -ld /data
```

Mặc định thường root-owned.

Không dùng:

```bash
sudo chmod 777 /data
```

để “chữa nhanh”.

Trong lab cá nhân, nếu user hiện tại cần write:

```bash
sudo chown "$USER":"$USER" /data
```

Kiểm tra:

```bash
ls -ld /data
```

Trong production, owner/group phải được xác định bởi service account/application requirement chứ không phải convenience.

---

# 27. Phase I — Configure persistent mount

Lấy UUID:

```bash
export FS_UUID=$(sudo blkid -s UUID -o value "$DEVICE")
```

```bash
echo "$FS_UUID"
```

Backup:

```bash
sudo cp -a /etc/fstab /etc/fstab.day10.bak
```

Thêm entry:

```bash
echo "UUID=$FS_UUID /data ext4 defaults,nofail 0 2" \
  | sudo tee -a /etc/fstab
```

Inspect:

```bash
tail -n 5 /etc/fstab
```

Test:

```bash
sudo umount /data
```

```bash
sudo mount -a
```

Verify:

```bash
findmnt /data
```

Nếu `mount -a` báo lỗi:

**Không reboot.**

Rollback:

```bash
sudo cp -a /etc/fstab.day10.bak /etc/fstab
```

Sau đó investigation lại UUID/filesystem/mount point.

AWS đặc biệt cảnh báo lỗi `/etc/fstab` có khả năng làm system không boot bình thường. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html?utm_source=chatgpt.com)

---

# 28. Phase J — Create backup test data

Tạo file quan trọng:

```bash
cat <<'EOF' | sudo tee /data/important.txt
Day 10 EBS recovery test
This data must survive snapshot and restore.
EOF
```

Tạo thêm hierarchy:

```bash
sudo mkdir -p /data/app/config
```

```bash
echo 'environment=day10-lab' \
  | sudo tee /data/app/config/application.properties
```

Kiểm tra:

```bash
find /data -maxdepth 3 -type f -ls
```

Tạo checksum:

```bash
export ORIGINAL_SHA=$(
  sudo sha256sum /data/important.txt | awk '{print $1}'
)
```

```bash
echo "$ORIGINAL_SHA" | tee ~/day10-original.sha256
```

Hiển thị:

```bash
sudo sha256sum /data/important.txt
```

Evidence quan trọng:

```text
original file content
original SHA-256
volume ID
filesystem UUID
mount point
```

---

# 29. Phase K — Create consistent snapshot

Flush buffered writes:

```bash
sync
```

Unmount:

```bash
sudo umount /data
```

Confirm:

```bash
findmnt /data || echo "/data is unmounted"
```

Ở workstation/terminal có AWS CLI:

```bash
export SNAPSHOT_ID=$(
  aws ec2 create-snapshot \
    --volume-id "$VOLUME_ID" \
    --description "Day10 verified recovery snapshot" \
    --region "$REGION" \
    --query 'SnapshotId' \
    --output text
)
```

```bash
echo "$SNAPSHOT_ID"
```

Tag snapshot:

```bash
aws ec2 create-tags \
  --resources "$SNAPSHOT_ID" \
  --tags \
    Key=Name,Value=day10-backup \
    Key=Project,Value=java-fresher-day10 \
    Key=Environment,Value=lab \
  --region "$REGION"
```

Snapshot request đã được accepted và snapshot vào `pending`, ta có thể mount volume trở lại:

```bash
sudo mount /data
```

AWS khuyến nghị pause writes/unmount trước snapshot; sau khi snapshot đã đi vào `pending` có thể remount/resume writes. [AWS Documentation](https://docs.aws.amazon.com/en_en/ebs/latest/userguide/ebs-creating-snapshot.html?utm_source=chatgpt.com)

Chờ completed:

```bash
aws ec2 wait snapshot-completed \
  --snapshot-ids "$SNAPSHOT_ID" \
  --region "$REGION"
```

Inspect:

```bash
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --region "$REGION" \
  --query 'Snapshots[0].{ID:SnapshotId,State:State,Progress:Progress,Encrypted:Encrypted,Volume:VolumeId,Start:StartTime}' \
  --output table
```

Expected:

```text
State = completed
```

Nhưng nhớ:

> `completed` vẫn chưa phải end-to-end recovery verification.

---

# 30. Phase L — Failure Injection

Bây giờ cố tình tạo “data loss”.

Kiểm tra file trước:

```bash
cat /data/important.txt
```

Xóa:

```bash
sudo rm /data/important.txt
```

Tạo file chỉ xuất hiện **sau snapshot**:

```bash
echo "created after snapshot" \
  | sudo tee /data/after-snapshot.txt
```

Kiểm tra:

```bash
ls -l /data
```

Expected current state:

```text
important.txt        → missing
after-snapshot.txt   → exists
```

Đây là failure injection.

---

# 31. Phase M — Restore snapshot thành new volume

Ở AWS CLI terminal:

```bash
export RESTORE_VOLUME_ID=$(
  aws ec2 create-volume \
    --availability-zone "$AZ" \
    --snapshot-id "$SNAPSHOT_ID" \
    --volume-type gp3 \
    --tag-specifications \
      'ResourceType=volume,Tags=[{Key=Name,Value=day10-restored},{Key=Project,Value=java-fresher-day10},{Key=Environment,Value=lab}]' \
    --region "$REGION" \
    --query 'VolumeId' \
    --output text
)
```

```bash
echo "$RESTORE_VOLUME_ID"
```

Wait:

```bash
aws ec2 wait volume-available \
  --volume-ids "$RESTORE_VOLUME_ID" \
  --region "$REGION"
```

Inspect:

```bash
aws ec2 describe-volumes \
  --volume-ids "$RESTORE_VOLUME_ID" \
  --region "$REGION" \
  --query 'Volumes[0].{ID:VolumeId,State:State,AZ:AvailabilityZone,Encrypted:Encrypted,Snapshot:SnapshotId}' \
  --output table
```

Expected:

```text
Snapshot = SNAPSHOT_ID
AZ       = EC2 AZ
State    = available
```

AWS documents that creating a volume from snapshot recreates the point-in-time state represented by that snapshot. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-volume.html?utm_source=chatgpt.com)

---

# 32. Phase N — Attach restored volume

```bash
aws ec2 attach-volume \
  --volume-id "$RESTORE_VOLUME_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdg \
  --region "$REGION"
```

Wait:

```bash
aws ec2 wait volume-in-use \
  --volume-ids "$RESTORE_VOLUME_ID" \
  --region "$REGION"
```

SSH EC2:

```bash
lsblk -o NAME,SIZE,FSTYPE,LABEL,UUID,MOUNTPOINTS,SERIAL
```

Bạn sẽ thấy một device mới.

Ví dụ:

```text
nvme0n1
nvme1n1   /data
nvme2n1
```

Xác định mapping cẩn thận.

Ví dụ:

```bash
sudo ebsnvme-id /dev/nvme2n1
```

Sau khi chắc chắn:

```bash
export RESTORE_DEVICE=/dev/nvme2n1
```

---

# 33. Phase O — Cực kỳ quan trọng: KHÔNG `mkfs` restored volume

Chạy:

```bash
sudo blkid "$RESTORE_DEVICE"
```

Bạn phải thấy filesystem đã tồn tại:

```text
TYPE="ext4"
```

Thậm chí filesystem UUID rất có thể giống filesystem tại thời điểm snapshot.

Điều này hợp lý:

```text
Snapshot copies block state
→ filesystem metadata nằm trong block state
→ restored filesystem mang filesystem identity đó
```

Vì thế **không thêm restored clone vào `/etc/fstab` bằng duplicate UUID trong khi original cũng đang tồn tại**.

Và tuyệt đối không:

```bash
sudo mkfs.ext4 "$RESTORE_DEVICE"
```

Làm vậy sẽ phá backup restore.

---

# 34. Phase P — Mount restored volume read-only

Tạo restore point:

```bash
sudo mkdir -p /restore
```

Mount read-only:

```bash
sudo mount -o ro "$RESTORE_DEVICE" /restore
```

Verify:

```bash
findmnt /restore
```

```bash
ls -la /restore
```

Expected snapshot state:

```text
important.txt        → exists
after-snapshot.txt   → does NOT exist
```

Tại sao?

Vì:

```text
important.txt existed before snapshot
after-snapshot.txt was created after snapshot
```

Đây là bằng chứng snapshot thực sự là point-in-time recovery.

---

# 35. Phase Q — Backup integrity verification

Load expected checksum:

```bash
export ORIGINAL_SHA=$(cat ~/day10-original.sha256)
```

Hash restored data:

```bash
export RESTORED_SHA=$(
  sudo sha256sum /restore/important.txt | awk '{print $1}'
)
```

So sánh:

```bash
printf 'Original: %s\nRestored: %s\n' \
  "$ORIGINAL_SHA" "$RESTORED_SHA"
```

Automatic assertion:

```bash
test "$ORIGINAL_SHA" = "$RESTORED_SHA" \
  && echo "PASS: Restored file checksum matches" \
  || echo "FAIL: Checksum mismatch"
```

**Expected invariant:**

```text
PASS: Restored file checksum matches
```

Đây là evidence mạnh hơn việc chỉ chụp ảnh snapshot state `completed`.

---

# 36. Phase R — Recover deleted data

Copy file về production/original volume:

```bash
sudo cp -a /restore/important.txt /data/important.txt
```

Verify:

```bash
sudo sha256sum /data/important.txt
```

So sánh:

```bash
export RECOVERED_SHA=$(
  sudo sha256sum /data/important.txt | awk '{print $1}'
)
```

```bash
test "$ORIGINAL_SHA" = "$RECOVERED_SHA" \
  && echo "RECOVERY PASS" \
  || echo "RECOVERY FAIL"
```

Expected:

```text
RECOVERY PASS
```

Assignment bắt buộc của Day 10 tới đây đã hoàn thành về mặt kỹ thuật:

```text
Attach EBS        ✔
Mount EBS         ✔
Create snapshot   ✔
Simulate loss     ✔
Restore snapshot  ✔
Restore data      ✔
Verify recovery   ✔
Record evidence   → tiếp theo
```

---

# 37. Recovery Evidence Pack

Bạn nên lưu một evidence record kiểu:

```text
DAY 10 - EBS BACKUP/RESTORE TEST

Instance ID:
i-...

Region:
ap-southeast-1

Availability Zone:
ap-southeast-1x

Source EBS:
vol-...

Filesystem:
ext4

Filesystem UUID:
...

Mount:
vol-... -> /data

Snapshot:
snap-...

Snapshot status:
completed

Restored EBS:
vol-...

Restore mount:
/restore

Original checksum:
...

Restored checksum:
...

Recovery checksum:
...

Result:
PASS

Failure injected:
Deleted /data/important.txt

Recovery action:
Created volume from snapshot,
attached to original EC2,
mounted read-only at /restore,
verified SHA-256,
copied missing file back.

Validation:
Recovered SHA-256 matched original.

Cleanup:
...
```

Đây mới là mindset operations:

> Không nói “em nghĩ backup OK”; hãy **prove bằng evidence**.

---

# 38. Cleanup — AWS cost awareness

EBS volumes tiếp tục phát sinh storage charge cho tới khi được release/deleted; chỉ detach volume không làm storage cost biến mất. Snapshot cũng phát sinh snapshot storage cost theo lượng data được lưu. [Amazon Web Services, Inc.](https://aws.amazon.com/ebs/pricing/?loc=3&nc=sn&utm_source=chatgpt.com)

Trước cleanup, giữ lại evidence.

Unmount restore:

```bash
sudo umount /restore
```

Detach restored volume:

```bash
aws ec2 detach-volume \
  --volume-id "$RESTORE_VOLUME_ID" \
  --region "$REGION"
```

Wait:

```bash
aws ec2 wait volume-available \
  --volume-ids "$RESTORE_VOLUME_ID" \
  --region "$REGION"
```

Delete:

```bash
aws ec2 delete-volume \
  --volume-id "$RESTORE_VOLUME_ID" \
  --region "$REGION"
```

Nếu không còn cần source volume, trước hết sửa `/etc/fstab`. Trong lab controlled, nếu Day 10 là thay đổi duy nhất kể từ backup:

```bash
sudo cp -a /etc/fstab.day10.bak /etc/fstab
```

Validate:

```bash
sudo mount -a
```

Unmount:

```bash
sudo umount /data
```

Rồi detach:

```bash
aws ec2 detach-volume \
  --volume-id "$VOLUME_ID" \
  --region "$REGION"
```

Wait:

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOLUME_ID" \
  --region "$REGION"
```

Delete:

```bash
aws ec2 delete-volume \
  --volume-id "$VOLUME_ID" \
  --region "$REGION"
```

Nếu snapshot chỉ dành cho lab và evidence đã hoàn chỉnh:

```bash
aws ec2 delete-snapshot \
  --snapshot-id "$SNAPSHOT_ID" \
  --region "$REGION"
```

Xóa local directories nếu rỗng:

```bash
sudo rmdir /restore /data
```

AWS yêu cầu volume phải được detached trước khi delete. Sau khi EBS volume bị delete thì data của nó không còn attach/reuse được nữa, do đó phải kiểm tra snapshot/recovery requirement **trước khi delete**. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-deleting-volume.html?utm_source=chatgpt.com)

---

# 39. Failure Injection & Troubleshooting

Đây là phần bạn phải học theo evidence-driven troubleshooting:

```text
symptom
→ expected behavior
→ recent changes
→ scope
→ failing layer
→ evidence
→ hypotheses
→ safest test
→ root cause
→ remediation
→ validation
→ evidence
```

| Symptom                                            | Layer/hypothesis               | Evidence/Test                  | Action                                        |
| -------------------------------------------------- | ------------------------------ | ------------------------------ | --------------------------------------------- |
| Volume không attach                                | AZ mismatch                    | Compare EC2 AZ và EBS AZ       | Tạo volume đúng AZ                            |
| `AttachmentLimitExceeded`                          | Instance attachment limit      | AWS error + instance type      | Remove unnecessary volume/escalate design     |
| AWS báo attached nhưng Linux không thấy `/dev/sdf` | Nitro/NVMe naming              | `lsblk`, `ebsnvme-id`          | Tìm `/dev/nvme...`                            |
| `mkfs` phát hiện filesystem                        | Volume đã chứa data            | `blkid`, `file -s`             | **STOP**, không format                        |
| Mount báo wrong fs type                            | Sai filesystem/device          | `lsblk -f`, `blkid`            | Dùng đúng FS/device                           |
| `/data` mất sau reboot                             | Không có/bad fstab             | `cat /etc/fstab`, `findmnt`    | UUID + validate `mount -a`                    |
| Instance boot issue sau fstab                      | UUID/type/options sai          | console/serial/rescue evidence | Rollback `/etc/fstab`                         |
| `Permission denied` tại `/data`                    | Linux ownership/permissions    | `ls -ld /data`                 | Set least-privilege owner/group               |
| `umount: target is busy`                           | Process đang dùng mount        | `lsof +D /data` hoặc `fuser`   | Stop/quiesce process                          |
| Snapshot restore thiếu transaction/app consistency | Writes diễn ra lúc snapshot    | Timeline/app logs              | Quiesce app, application-consistent backup    |
| Restore volume performance chậm ban đầu            | Snapshot initialization        | EBS initialization status/perf | Allow initialization or approved acceleration |
| Volume không delete được                           | Still `in-use`                 | `describe-volumes`             | Unmount → detach → delete                     |
| Detach bị `busy`                                   | Filesystem vẫn mounted/writing | `findmnt`, process check       | Stop writes + unmount                         |
| Encrypted restore lỗi                              | KMS/IAM dependency             | AWS error, key status/policy   | Escalate/restore required permissions         |

AWS yêu cầu running instance phải unmount filesystem trước khi detach. Forced detach chỉ nên là last resort vì instance không có cơ hội flush filesystem caches/metadata và có thể dẫn tới filesystem/data damage. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-detaching-volume.html?utm_source=chatgpt.com)

---

# 40. Ví dụ troubleshooting thực tế

### Incident

Application báo:

```text
No space left on device
```

Junior admin nghĩ:

> “EBS bị hỏng.”

Sai approach.

Đi theo layers:

```text
Application
    ↓
filesystem usage
    ↓
mount
    ↓
block device
    ↓
EBS
```

Kiểm tra:

```bash
df -hT
```

Nếu:

```text
/dev/nvme1n1  8G  8G  0  100% /data
```

thì problem là filesystem capacity.

Tiếp:

```bash
du -xhd1 /data | sort -h
```

Không phải mọi `disk full` đều là AWS EBS failure.

Ngược lại nếu:

```bash
df -hT
```

không có `/data`, kiểm tra:

```bash
findmnt /data
lsblk -f
cat /etc/fstab
```

Ta đã chuyển sang mount/filesystem layer.

Đó chính là integrated troubleshooting skill cần cho Day 20 sau này.

---

# 41. Mistakes / Anti-patterns

Những lỗi nguy hiểm nhất hôm nay là:

| Anti-pattern                                                | Tại sao nguy hiểm                           |
| ----------------------------------------------------------- | ------------------------------------------- |
| `mkfs` ngay khi thấy new disk                               | Có thể destroy existing/restored filesystem |
| Assume `/dev/sdf` == Linux device                           | Nitro thường expose NVMe                    |
| Hard-code `/dev/nvme1n1` vào fstab                          | Device enumeration có thể thay đổi          |
| Dùng `chmod 777`                                            | Vi phạm least privilege                     |
| Edit fstab rồi reboot ngay                                  | Bad entry có thể gây boot issue             |
| Snapshot trong lúc app đang write mà không hiểu consistency | Backup có thể không application-consistent  |
| “Snapshot completed nên backup verified”                    | Chưa chứng minh restore                     |
| Restore volume rồi chạy `mkfs`                              | Tự phá recovered data                       |
| Force-detach vì nhanh                                       | Risk corruption/data loss                   |
| Giữ snapshots mãi mãi                                       | Cost + governance problem                   |
| Delete snapshot/volume trước recovery evidence              | Không còn recovery point cần thiết          |
| Không tag EBS/snapshot                                      | Khó ownership/cost/retention/audit          |

---

# 42. Security & Least Privilege

EBS operations nên tuân theo cùng nguyên tắc Day 7:

```text
Không cấp AdministratorAccess chỉ vì cần snapshot.
```

Operational separation có thể là:

```text
Create/describe/attach approved volumes
Create/describe approved snapshots
Restore approved snapshots
Delete chỉ theo controlled workflow
KMS access chỉ với required keys
```

Một account/user có quyền:

```text
ec2:DeleteSnapshot
ec2:DeleteVolume
kms:DisableKey
```

có blast radius rất lớn.

Backup security phải xem xét cả:

```text
Confidentiality  → encryption
Integrity        → verification
Availability     → tested restore
Authorization    → IAM
Key dependency   → KMS
Retention        → lifecycle
Deletion safety  → controls / Recycle Bin where appropriate
```

---

# 43. Operations, rollback và recovery

Day 10 không chỉ nói backup.

Nó dạy ba khái niệm khác nhau:

```text
Backup:
Tạo recovery point.

Restore:
Tạo lại dữ liệu/resource từ recovery point.

Recovery:
Đưa hệ thống trở về usable/validated state.
```

Ví dụ:

```text
snapshot created          = backup
volume created from snap  = restore
file recovered + app OK   = recovery
```

Đây là distinction rất quan trọng.

### Rollback `/etc/fstab`

Nếu configuration mount sai:

```bash
sudo cp -a /etc/fstab.day10.bak /etc/fstab
sudo mount -a
```

### Rollback data

Nếu bad change làm hỏng EBS data:

```text
stop writes
↓
preserve evidence
↓
identify recovery snapshot
↓
restore new volume
↓
validate independently
↓
recover selected data / replace volume
↓
validate application
```

Không overwrite/hủy old volume ngay từ đầu.

Một nguyên tắc tốt:

> **Preserve the broken state until the recovered state is proven**, nếu cost và incident procedure cho phép.

---

# 44. L1 scope vs escalation

| L1-safe action                  | Có thể cần escalation                           |
| ------------------------------- | ----------------------------------------------- |
| `describe-volume`               | Complex KMS policy problem                      |
| `lsblk`, `blkid`, `findmnt`     | Suspected AWS infrastructure issue              |
| Mount known filesystem          | Filesystem corruption requiring invasive repair |
| Fix clearly incorrect lab fstab | Production root-volume recovery                 |
| Restore approved snapshot       | Cross-account/Region DR architecture            |
| Verify files/checksum           | Database application-consistency issue          |
| Copy individual approved files  | Force-detach production EBS                     |
| Collect logs/evidence           | Destructive filesystem repair                   |
| Standard cleanup                | Regulatory retention decision                   |

Nếu bạn không chắc filesystem có data:

> L1 action đúng là **inspect và escalate**, không phải `mkfs để thử`.

---

# 45. Cost Awareness

EBS là một AWS resource có chi phí độc lập với EC2.

Một EC2 stopped không có nghĩa toàn bộ EBS cost biến mất.

EBS pricing phụ thuộc vào provisioned volume storage và, tùy volume type/configuration, performance parameters. Snapshot storage tính theo snapshot data stored; các tính năng như Fast Snapshot Restore hoặc archive/retrieval có cost model riêng. [Amazon Web Services, Inc.](https://aws.amazon.com/ebs/pricing/?loc=3&nc=sn&utm_source=chatgpt.com)

Do đó sau lab phải kiểm tra:

```bash
aws ec2 describe-volumes \
  --region "$REGION"
```

và:

```bash
aws ec2 describe-snapshots \
  --owner-ids self \
  --region "$REGION"
```

Tìm orphaned:

```text
available volumes
old lab snapshots
untagged resources
temporary restore volumes
```

---

# 46. Exam Focus — Theory

Các điểm bạn phải trả lời được chính xác:

| Question                                                        | Expected answer                                                  |
| --------------------------------------------------------------- | ---------------------------------------------------------------- |
| EBS là object storage hay block storage?                        | Block storage                                                    |
| EBS volume có attach EC2 khác AZ được không?                    | Không; volume phải cùng AZ với instance                          |
| Attach volume có tự tạo filesystem không?                       | Không                                                            |
| `mkfs` làm gì?                                                  | Tạo filesystem; có tính destructive nếu device có data           |
| Tại sao cần `lsblk`/`blkid` trước `mkfs`?                       | Xác định device và existing filesystem                           |
| `/dev/sdf` AWS console có chắc là `/dev/sdf` trong Linux không? | Không, Nitro thường expose NVMe                                  |
| Mount bằng device name hay UUID trong fstab tốt hơn?            | UUID                                                             |
| Vì sao `nofail` hữu ích?                                        | Cho phép boot tiếp nếu non-critical data volume không mount được |
| Snapshot đầu tiên?                                              | Full snapshot của blocks đã written                              |
| Snapshot tiếp theo?                                             | Incremental changed/new blocks                                   |
| Xóa snapshot cũ có mặc nhiên phá snapshot mới?                  | Không                                                            |
| Snapshot `completed` có đủ chứng minh backup OK?                | Không                                                            |
| Verification tốt là gì?                                         | Actual restore + integrity/application validation                |
| Restore snapshot trả trực tiếp file không?                      | Thông thường tạo EBS volume rồi attach/mount để truy cập         |
| Tại sao snapshot lúc active writes cần chú ý?                   | Application/data consistency                                     |
| Detach running EBS nên làm gì trước?                            | Stop writes/unmount filesystem                                   |
| Force detach có an toàn không?                                  | Chỉ last resort; risk corruption/data loss                       |
| Data retention để làm gì?                                       | Define creation/retention/deletion/recovery requirements         |

---

# 47. Exam Focus — Practical

Một practical assessment Day 10 rất có thể test tư duy dạng:

```text
Given:
EC2 Linux instance

Task:
Attach 10 GiB EBS
mount it at /appdata
make mount persistent
create important data
back it up
simulate data loss
restore it
prove recovery
```

Bạn không nên chạy command theo trí nhớ máy móc.

Flow chuẩn cần nằm trong đầu:

```text
inspect
↓
create EBS correct AZ
↓
wait available
↓
attach
↓
identify device
↓
inspect existing filesystem
↓
mkfs only if truly blank
↓
mount
↓
permissions
↓
UUID/fstab
↓
mount -a validation
↓
write test data
↓
checksum
↓
quiesce/unmount
↓
snapshot
↓
verify snapshot
↓
inject loss
↓
create volume from snapshot
↓
attach
↓
DO NOT mkfs
↓
mount restore
↓
verify checksum
↓
recover data
↓
validate
↓
document evidence
↓
cleanup
```

Nếu bạn thuộc được **flow** này thì command syntax có thể tra cứu. Nếu bạn chỉ thuộc commands nhưng không hiểu flow thì rất dễ `mkfs` nhầm restored volume.

---

# 48. Knowledge Check

| Câu hỏi                                                                 | Đáp án                                                                                      |
| ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Một EC2 ở `ap-southeast-1a`, volume ở `1b`. Làm sao attach?             | Không attach trực tiếp; cần volume ở `1a`, thường tạo/restore volume thích hợp tại `1a`.    |
| AWS nói attach `/dev/sdf`, Linux chỉ thấy `/dev/nvme1n1`. Có lỗi không? | Không nhất thiết; Nitro remaps thành NVMe.                                                  |
| `lsblk -f` cho thấy `ext4`. Có chạy `mkfs.ext4` không?                  | Không. Investigate existing filesystem/data.                                                |
| Snapshot `completed`, có thể tuyên bố recovery tested không?            | Không. Phải restore và verify.                                                              |
| Restored volume chứa filesystem rồi. Bước đầu tiên sau attach là gì?    | Inspect bằng `lsblk -f`/`blkid`, không format.                                              |
| Sau restore checksum khác original thì sao?                             | Recovery verification FAIL; preserve evidence và investigate, không tuyên bố success.       |
| Vì sao UUID tốt hơn NVMe name trong fstab?                              | UUID gắn với filesystem và ổn định hơn device enumeration.                                  |
| `/etc/fstab` vừa sửa, bước tiếp theo?                                   | Backup đã có → `mount -a` → verify; chưa reboot nếu có error.                               |
| Detach EBS đang mounted có đúng không?                                  | Không; stop writes/unmount trước.                                                           |
| Snapshot backup có giải quyết mọi loại database consistency không?      | Không; workload/application consistency có thể cần app-specific quiescing/backup procedure. |

---

# 49. Independent Practical Challenge

Sau Guided Lab, bài **PRACTICE** bạn nên tự làm lại mà không nhìn phần solution:

> Bạn có một EC2 Linux instance. Hãy tạo một encrypted `gp3` data volume, mount persistent tại `/srv/java-data`, tạo ít nhất ba file với một nested directory, ghi SHA-256 của file quan trọng, tạo consistent EBS snapshot, xóa file quan trọng và sửa một file khác, restore snapshot thành volume mới, attach và mount read-only, chứng minh snapshot chứa pre-failure state, phục hồi file bị xóa, chứng minh SHA-256 sau recovery giống ban đầu, sau đó cleanup toàn bộ temporary resources và nộp evidence.

Điểm assessment không chỉ là “file đã về”.

Bạn cần chứng minh:

```text
correct AZ
correct EBS
correct Linux device
filesystem identified
persistent mount validated
snapshot completed
restore volume references correct snapshot
restore mounted successfully
expected point-in-time files present
checksum matches
recovered production file matches
resources cleaned up
```

---

# 50. Mastery Check Day 10

Bạn chỉ nên coi Day 10 là **đã thực hành thành công** nếu có evidence cho các năng lực sau:

| Capability                | Evidence cần có                                          |
| ------------------------- | -------------------------------------------------------- |
| Identify correct EBS      | Volume ID + AZ                                           |
| Attach                    | Volume `in-use`                                          |
| Identify Linux device     | `lsblk` + mapping                                        |
| Filesystem                | `blkid`                                                  |
| Mount                     | `findmnt /data`                                          |
| Persistent mount          | `/etc/fstab` + successful `mount -a`                     |
| Original data             | File content                                             |
| Integrity baseline        | SHA-256                                                  |
| Snapshot                  | Snapshot ID + `completed`                                |
| Failure                   | File deleted                                             |
| Restore                   | New volume from snapshot                                 |
| Restore mount             | `/restore`                                               |
| Point-in-time correctness | Before-snapshot file present, after-snapshot file absent |
| Backup verification       | SHA-256 equal                                            |
| Recovery                  | File copied back                                         |
| Final validation          | Recovered SHA-256 equal                                  |
| Operations                | Recovery record                                          |
| Cost control              | Temporary volume/snapshot cleanup                        |

Đọc và hiểu bài **chưa đủ để đánh dấu mastery**. Day 10 chỉ thực sự đạt khi bạn tự execute được chuỗi:

**understand → provision → attach → inspect → format → mount → persist → snapshot → fail → restore → verify → recover → document → cleanup.**

---

# 51. Tóm tắt toàn bộ Day 10

Mental model quan trọng nhất cần giữ lại là:

```text
             AWS
              │
          EBS Volume
              │ attach
              ▼
       Linux Block Device
              │
            mkfs
              ▼
          Filesystem
              │ mount
              ▼
            /data
              │
           application
             data
              │
        consistent snapshot
              ▼
         EBS Snapshot
              │
        create-volume
              ▼
     Restored EBS Volume
              │ attach
              ▼
     Restored Block Device
              │
      NO mkfs — already FS!
              │ mount
              ▼
           /restore
              │
     integrity verification
              ▼
          recovery
              │
       final validation
              ▼
           evidence
```

Nếu chỉ nhớ ba nguyên tắc của Day 10, hãy nhớ:

**Một:** trước `mkfs`, luôn **inspect**.  
**Hai:** snapshot `completed` **không đồng nghĩa backup đã được verify**.  
**Ba:** recovery chỉ thành công khi **restore được dữ liệu và chứng minh dữ liệu/application đúng**.

AWS hiện vẫn xác nhận các nguyên tắc cốt lõi mà chúng ta dùng trong lab: EBS volume phải cùng AZ với EC2 khi attach; Nitro expose EBS dưới dạng NVMe; persistent mounts nên dùng filesystem UUID; snapshot đầu tiên là full và subsequent snapshots incremental; snapshot consistency cần kiểm soát active writes; và restore thực chất tạo EBS volume mới từ recovery point. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-creating-volume.html?utm_source=chatgpt.com)

**Bước học tốt nhất bây giờ là thực hiện Guided Lab từ Phase A đến Phase R theo đúng thứ tự, không bỏ bước inspection/evidence.** Khi bạn chạy lab, nếu gửi cho tôi output của từng checkpoint như `describe-volumes`, `lsblk -f`, `blkid`, `findmnt`, snapshot state và checksum, tôi có thể review theo **PRACTICE MODE**, xác định từng bước PASS/FAIL và dẫn bạn troubleshooting mà không làm thay phần thực hành.
