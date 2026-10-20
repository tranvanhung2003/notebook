# Day 5 — Module Thực hành

## AWS EBS, S3 Backup/Recovery, DR Verification, CloudWatch, CloudTrail, Cost Control & Cleanup

Phần này triển khai **toàn bộ Assignment/Lab của Day 5** thành một chuỗi vận hành liên tục. Theo syllabus, bạn phải thực hiện EBS attach/mount, snapshot và restore; xây S3 backup bucket với access control, `Block Public Access`, encryption, versioning và lifecycle/retention; backup và recovery có checksum; ghi nhận `RPO/RTO`; cấu hình CloudWatch alarm và budget control; điều tra CloudTrail; đánh giá cost drivers; cleanup resource; cuối cùng tạo combined AWS backup/monitoring/cost/DR evidence package. Infrastructure_Operations_14_Da…

Phần dưới giả định **authorized AWS sandbox/non-production**, AWS CLI v2, một Linux EC2 lab instance từ Day 4 và quyền IAM được cấp đúng scope. Không thực hiện các thao tác này trên production nếu chưa có ticket/change, maintenance window, backup, rollback plan và approval.

---

# 1. Kết quả cuối cùng của Module Thực hành

Sau toàn bộ lab, bạn phải chứng minh được workflow sau:

```text
EC2
 │
 ├── Attach new EBS
 │
 ├── Format filesystem
 │
 ├── Mount
 │
 ├── Persistent mount
 │
 ├── Create test data
 │
 ├── Generate SHA-256 manifest
 │
 └── EBS Snapshot
         │
         ▼
   Restore new EBS
         │
         ├── Attach
         ├── Mount isolated/read-only
         └── Verify files + SHA-256

Test data
   │
   ▼
S3 Backup Bucket
   │
   ├── Block Public Access
   ├── Encryption
   ├── Versioning
   ├── Lifecycle/retention
   ├── Backup upload
   ├── Overwrite/delete simulation
   └── Version recovery
             │
             ▼
       Isolated restore
             │
             ├── file count
             ├── SHA-256
             └── application-readable test

Monitoring/Audit
   │
   ├── CloudWatch metrics
   ├── CloudWatch alarm
   └── CloudTrail investigation

Governance
   │
   ├── RPO
   ├── RTO
   ├── AWS Budget
   ├── Cost drivers
   └── Controlled cleanup
```

Acceptance không phải là:

```text
Snapshot = completed
Backup = completed
```

mà phải là:

```text
backup exists
+
correct recovery point selected
+
restore succeeds
+
restored content matches
+
permission/key dependency verified
+
RPO/RTO recorded
+
audit coverage understood
+
evidence preserved
```

Đây chính là yêu cầu bổ sung của syllabus. Infrastructure_Operations_14_Da…

---

# 2. Lab Environment và Safety Baseline

## 2.1 Topology

Giả sử:

```text
AWS Account — Sandbox
│
├── Region: <LAB_REGION>
│
├── VPC
│    └── Subnet
│          └── EC2 Linux
│               ├── Root EBS
│               └── Day5 Data EBS
│
├── EBS Snapshot
│
├── S3 Day5 Backup Bucket
│
├── CloudWatch
│
├── CloudTrail
│
└── AWS Budgets
```

Không dùng credential thật trong transcript/evidence.

---

# 3. Chuẩn bị variables

Trước khi làm lab, đặt một số variables để giảm lỗi copy/paste.

```bash
export AWS_REGION="<your-region>"
export INSTANCE_ID="<your-ec2-instance-id>"
export LAB_ID="<your-unique-lab-id>"
```

Ví dụ:

```bash
export AWS_REGION="ap-southeast-1"
export INSTANCE_ID="i-0123456789abcdef0"
export LAB_ID="student01-day5"
```

Không copy example ID và dùng như resource thật.

---

# 4. Pre-flight Check — Xác minh Identity trước khi thay đổi

Đây phải là bước đầu tiên của mọi AWS operation.

## Read-only

```bash
aws sts get-caller-identity
```

Expected fields:

```json
{
  "UserId": "...",
  "Account": "...",
  "Arn": "..."
}
```

Bạn cần ghi nhận:

```text
AWS Account
Active principal/role
Region
Ticket/Lab ID
Timestamp
```

Lấy account ID:

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)
```

Verify:

```bash
echo "$ACCOUNT_ID"
```

---

# 5. Xác minh Region và EC2 Instance

```bash
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION" \
  --query 'Reservations[0].Instances[0].{
    InstanceId:InstanceId,
    State:State.Name,
    AZ:Placement.AvailabilityZone,
    PrivateIp:PrivateIpAddress,
    PublicIp:PublicIpAddress,
    InstanceType:InstanceType
  }'
```

Bạn cần xác nhận:

```text
InstanceId đúng
State = running
Region đúng
Availability Zone đúng
```

Lấy Availability Zone:

```bash
export AZ=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --region "$AWS_REGION" \
  --query 'Reservations[0].Instances[0].Placement.AvailabilityZone' \
  --output text)

echo "$AZ"
```

EBS volume được attach theo workflow cơ bản phải nằm cùng Availability Zone với EC2 instance. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-volume.html?utm_source=chatgpt.com)

---

# 6. Evidence Folder

Trên administrative workstation:

```bash
mkdir -p "$HOME/day5-evidence"/{raw,working,screenshots,hashes}
```

Tạo timestamp UTC:

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"
```

Mọi evidence nên ghi:

```text
Evidence ID
AWS account
Region
Resource ID
Source
Timestamp/timezone
Command/filter
Ticket/Lab ID
Raw/derived status
SHA-256
Limitations
```

Đây phù hợp với evidence contract dùng xuyên suốt khóa học: raw evidence phải được giữ nguyên, working/redacted copy phải tách riêng và hash không tự chứng minh source là complete/correct. Infrastructure_Operations_14_Da…

---

# LAB 1 — CREATE VÀ ATTACH EBS VOLUME

## Objective

Tạo một EBS data volume riêng, attach vào EC2 và xác minh mapping từ AWS layer xuống guest OS.

---

## 1. Baseline trước thay đổi

### AWS layer

```bash
aws ec2 describe-volumes \
  --filters Name=attachment.instance-id,Values="$INSTANCE_ID" \
  --region "$AWS_REGION" \
  --query 'Volumes[].{
    VolumeId:VolumeId,
    Type:VolumeType,
    Size:Size,
    State:State,
    Encrypted:Encrypted,
    Device:Attachments[0].Device
  }'
```

Lưu:

```bash
aws ec2 describe-volumes \
  --filters Name=attachment.instance-id,Values="$INSTANCE_ID" \
  --region "$AWS_REGION" \
  > "$HOME/day5-evidence/raw/pre-ebs.json"
```

### OS layer

SSH vào instance.

```bash
lsblk
```

Sau đó:

```bash
lsblk -f
```

Lưu output:

```bash
lsblk -f | tee ~/day5-lsblk-before.txt
```

---

# 2. Create EBS volume

**State-changing command.**

Trong lab ta có thể dùng một volume nhỏ `gp3`.

```bash
aws ec2 create-volume \
  --availability-zone "$AZ" \
  --size 2 \
  --volume-type gp3 \
  --tag-specifications \
  'ResourceType=volume,Tags=[{Key=Lab,Value=Day5},{Key=Environment,Value=Sandbox}]' \
  --region "$AWS_REGION"
```

Tốt hơn, capture ID:

```bash
export VOL_ID=$(aws ec2 create-volume \
  --availability-zone "$AZ" \
  --size 2 \
  --volume-type gp3 \
  --tag-specifications \
  'ResourceType=volume,Tags=[{Key=Lab,Value=Day5},{Key=Environment,Value=Sandbox}]' \
  --region "$AWS_REGION" \
  --query VolumeId \
  --output text)

echo "$VOL_ID"
```

AWS CLI hiện hỗ trợ `gp3`, `gp2`, `io1`, `io2`, `st1`, `sc1` và legacy magnetic `standard`; volume được tạo phải nằm cùng AZ với instance nhận attachment. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-volume.html?utm_source=chatgpt.com)

---

# 3. Wait tới trạng thái available

Đây chỉ poll trạng thái, không thay đổi configuration:

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOL_ID" \
  --region "$AWS_REGION"
```

Sau đó:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOL_ID" \
  --region "$AWS_REGION" \
  --query 'Volumes[0].{
    VolumeId:VolumeId,
    State:State,
    AZ:AvailabilityZone,
    Size:Size,
    Type:VolumeType,
    Encrypted:Encrypted
  }'
```

Expected:

```text
State = available
AZ = same AZ as EC2
```

---

# 4. Attach EBS volume

**State-changing.**

```bash
aws ec2 attach-volume \
  --volume-id "$VOL_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdf \
  --region "$AWS_REGION"
```

Lưu ý rất quan trọng:

```text
/dev/sdf
```

là device name request ở EC2 API layer.

Trên Nitro-based Linux EC2, guest OS có thể thấy:

```text
/dev/nvme1n1
```

hoặc tên khác.

Không được giả định:

```text
/dev/sdf == actual Linux block device
```

---

# 5. Xác minh OS thấy disk

Trên Linux:

```bash
lsblk
```

và:

```bash
lsblk -f
```

So sánh với baseline trước attach.

Ví dụ:

```text
NAME        FSTYPE UUID  MOUNTPOINT
nvme0n1
└─nvme0n1p1 xfs         /
nvme1n1
```

`nvme1n1` mới xuất hiện và không có `FSTYPE`.

AWS cũng khuyến nghị dùng `lsblk -f` hoặc `file -s` để xác định volume đã có filesystem hay chưa. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html?utm_source=chatgpt.com)

---

# 6. Evidence phải lưu

Ít nhất:

```text
VOL_ID
Instance ID
AZ
Volume type
Size
Encryption state
AWS attachment state
Before/after lsblk
Timestamp
```

---

# LAB 2 — FORMAT, MOUNT VÀ PERSISTENT MOUNT

Đây là lab có rủi ro cao nhất đối với data.

## Critical rule

**Không chạy `mkfs` cho tới khi chứng minh device thật sự là empty test volume.**

AWS cảnh báo trực tiếp rằng format một volume tạo từ snapshot hoặc có data sẽ phá dữ liệu hiện có. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html?utm_source=chatgpt.com)

---

# 1. Identify device

Ví dụ sau khi so sánh `lsblk`:

```bash
export DEVICE="/dev/nvme1n1"
```

Đừng copy giá trị này nếu hệ thống của bạn khác.

---

# 2. Read-only filesystem check

```bash
sudo file -s "$DEVICE"
```

Và:

```bash
sudo lsblk -f "$DEVICE"
```

Nếu:

```text
/dev/nvme1n1: data
```

và `FSTYPE` trống, đó là dấu hiệu volume chưa có filesystem.

Nhưng vẫn phải reconcile:

```text
AWS volume ID
↔
new device
↔
expected size
```

trước khi format.

---

# 3. Tạo filesystem

**Destructive/state-changing.**

Ví dụ XFS:

```bash
sudo mkfs -t xfs "$DEVICE"
```

Expected output sẽ có các thông tin như:

```text
meta-data=
data=
naming=
log=
realtime=
```

Nếu:

```text
mkfs.xfs: command not found
```

có thể thiếu `xfsprogs`. Trên RHEL-compatible system, package này cung cấp các XFS utilities.

---

# 4. Verify filesystem

```bash
lsblk -f "$DEVICE"
```

Expected:

```text
FSTYPE = xfs
UUID = <filesystem UUID>
```

Hoặc:

```bash
sudo blkid "$DEVICE"
```

---

# 5. Create mount point

```bash
sudo mkdir -p /data/day5
```

---

# 6. Mount

```bash
sudo mount "$DEVICE" /data/day5
```

Verification:

```bash
findmnt /data/day5
```

```bash
df -hT /data/day5
```

```bash
lsblk -f
```

Bạn cần thấy:

```text
filesystem = xfs
mountpoint = /data/day5
expected device
```

---

# 7. Permission check

```bash
ls -ld /data/day5
```

Không được mặc định:

```bash
sudo chmod 777 /data/day5
```

để “cho chạy được”.

Đó phá `least privilege`.

Nếu application user cần write, hãy cấp đúng ownership/group theo requirement.

Ví dụ sandbox:

```bash
sudo mkdir -p /data/day5/source
```

Không thay đổi ownership nếu chưa cần.

---

# 8. Backup `/etc/fstab`

```bash
sudo cp /etc/fstab /etc/fstab.day5.bak
```

Verify:

```bash
sudo ls -l /etc/fstab /etc/fstab.day5.bak
```

---

# 9. Lấy filesystem UUID

```bash
export FS_UUID=$(sudo blkid \
  -s UUID \
  -o value \
  "$DEVICE")

echo "$FS_UUID"
```

---

# 10. Tạo persistent mount

AWS khuyến nghị dùng filesystem UUID thay vì device name vì device name có thể thay đổi qua hardware reconfiguration/reboot. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html?utm_source=chatgpt.com)

Append:

```bash
printf 'UUID=%s /data/day5 xfs defaults,nofail 0 2\n' "$FS_UUID" \
  | sudo tee -a /etc/fstab
```

Sau đó xem:

```bash
tail -n 5 /etc/fstab
```

---

# 11. Verify `/etc/fstab`

Nếu `findmnt` hỗ trợ verification:

```bash
sudo findmnt --verify --verbose
```

Sau đó kiểm tra mount cycle:

```bash
cd /
sudo umount /data/day5
```

Verify unmounted:

```bash
findmnt /data/day5
```

Expected: no result.

Mount từ `fstab`:

```bash
sudo mount /data/day5
```

Hoặc trong controlled lab:

```bash
sudo mount -a
```

Verify:

```bash
findmnt /data/day5
df -hT /data/day5
```

---

# 12. Reboot verification

Chỉ làm nếu EC2 lab cho phép restart.

Trước reboot:

```bash
date -u
```

```bash
sudo reboot
```

Sau reconnect:

```bash
findmnt /data/day5
df -hT /data/day5
```

Acceptance:

```text
Instance booted normally
/data/day5 automatically mounted
Correct filesystem and UUID
```

---

# Rollback persistent mount

Nếu configuration sai nhưng machine vẫn online:

```bash
sudo cp /etc/fstab.day5.bak /etc/fstab
```

Sau đó:

```bash
sudo findmnt --verify --verbose
```

Không reboot một machine với `/etc/fstab` đang nghi ngờ.

---

# LAB 3 — TẠO CONTROLLED DATASET VÀ HASH MANIFEST

## Objective

Tạo một dataset nhỏ đủ để chứng minh restore integrity.

---

# 1. Create files

```bash
sudo mkdir -p /data/day5/source
```

Ví dụ:

```bash
echo "Day5 infrastructure backup test - file 1" \
  | sudo tee /data/day5/source/file1.txt

echo "Day5 infrastructure backup test - file 2" \
  | sudo tee /data/day5/source/file2.txt
```

Tạo nested directory:

```bash
sudo mkdir -p /data/day5/source/config
```

```bash
cat <<'EOF' | sudo tee /data/day5/source/config/app.conf
application=day5-lab
environment=sandbox
backup_required=true
EOF
```

---

# 2. File count

```bash
cd /data/day5/source

find . -type f | sort
```

Count:

```bash
find . -type f | wc -l
```

Giả sử:

```text
3
```

---

# 3. Generate SHA-256 manifest

```bash
cd /data/day5/source

find . -type f -exec sha256sum {} + \
  | sort -k2 \
  > /tmp/day5-source.sha256
```

View:

```bash
cat /tmp/day5-source.sha256
```

Copy tới evidence:

```bash
cp /tmp/day5-source.sha256 \
   "$HOME/day5-evidence/hashes/"
```

---

# 4. Verify manifest

```bash
cd /data/day5/source

sha256sum -c /tmp/day5-source.sha256
```

Expected:

```text
./config/app.conf: OK
./file1.txt: OK
./file2.txt: OK
```

---

# 5. Why SHA-256?

Syllabus yêu cầu integrity verification và nhấn mạnh không sử dụng generic S3 `ETag` như một universal content hash. Infrastructure_Operations_14_Da…

AWS CLI v2 hiện cũng có checksum support cho S3 uploads/downloads, nhưng trong lab chúng ta vẫn giữ một **independent SHA-256 manifest** để có baseline rõ ràng từ source. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/topic/s3-faq.html?utm_source=chatgpt.com)

---

# LAB 4 — EBS SNAPSHOT

## Objective

Tạo point-in-time snapshot của EBS data volume.

---

# 1. Chuẩn bị consistency

Với dataset file đơn giản:

```bash
sync
```

Nhưng phải hiểu:

```text
sync
≠
application-consistent database backup
```

Nếu là database/application thật, cần procedure quiesce/flush/application-aware tương ứng.

AWS lưu ý snapshot của in-use volume chỉ capture data đã được ghi xuống EBS; dữ liệu còn nằm trong OS/application cache có thể chưa nằm trong snapshot. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-snapshot.html?utm_source=chatgpt.com)

---

# 2. Ghi recovery-point timestamp

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"
```

Lưu thành:

```text
SNAPSHOT_DATA_TIME
```

---

# 3. Create snapshot

**State-changing.**

```bash
export SNAPSHOT_ID=$(aws ec2 create-snapshot \
  --volume-id "$VOL_ID" \
  --description "Day5 EBS recovery test" \
  --tag-specifications \
  'ResourceType=snapshot,Tags=[{Key=Lab,Value=Day5},{Key=Environment,Value=Sandbox}]' \
  --region "$AWS_REGION" \
  --query SnapshotId \
  --output text)

echo "$SNAPSHOT_ID"
```

---

# 4. Wait snapshot completed

```bash
aws ec2 wait snapshot-completed \
  --snapshot-ids "$SNAPSHOT_ID" \
  --region "$AWS_REGION"
```

Verify:

```bash
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAPSHOT_ID" \
  --region "$AWS_REGION" \
  --query 'Snapshots[0].{
    SnapshotId:SnapshotId,
    VolumeId:VolumeId,
    State:State,
    StartTime:StartTime,
    Progress:Progress,
    Encrypted:Encrypted
  }'
```

Expected:

```text
State = completed
Progress = 100%
```

EBS snapshots là incremental point-in-time backups; AWS quản lý chúng trong S3 backend nhưng chúng không xuất hiện như object trong S3 bucket của bạn. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-snapshots.html?utm_source=chatgpt.com)

---

# 5. Evidence

Lưu:

```text
Volume ID
Snapshot ID
Snapshot StartTime
Snapshot completion state
Filesystem/hash manifest before snapshot
AWS account/Region
```

---

# LAB 5 — RESTORE EBS SNAPSHOT

Đây là nơi bạn thực sự chứng minh EBS backup có giá trị.

---

# 1. Start recovery clock

Ghi:

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"
```

Đây là:

```text
RECOVERY_START
```

---

# 2. Create restored volume

Volume phải nằm cùng AZ với recovery EC2 instance.

```bash
export RESTORE_VOL_ID=$(aws ec2 create-volume \
  --availability-zone "$AZ" \
  --snapshot-id "$SNAPSHOT_ID" \
  --volume-type gp3 \
  --tag-specifications \
  'ResourceType=volume,Tags=[{Key=Lab,Value=Day5-Restore},{Key=Environment,Value=Sandbox}]' \
  --region "$AWS_REGION" \
  --query VolumeId \
  --output text)

echo "$RESTORE_VOL_ID"
```

AWS `create-volume --snapshot-id` tái tạo volume từ snapshot; restored volume phải cùng AZ với instance nơi nó được attach. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/ec2/create-volume.html?utm_source=chatgpt.com)

---

# 3. Wait volume ready

```bash
aws ec2 wait volume-available \
  --volume-ids "$RESTORE_VOL_ID" \
  --region "$AWS_REGION"
```

---

# 4. Baseline block devices

Trước attach:

```bash
lsblk -f
```

---

# 5. Attach restored volume

```bash
aws ec2 attach-volume \
  --volume-id "$RESTORE_VOL_ID" \
  --instance-id "$INSTANCE_ID" \
  --device /dev/sdg \
  --region "$AWS_REGION"
```

---

# 6. Identify new device

```bash
lsblk -f
```

Ví dụ:

```text
nvme2n1 xfs <same filesystem UUID as original>
```

Điểm cực kỳ quan trọng:

**Không chạy `mkfs`.**

AWS cảnh báo rằng format một restored volume sẽ phá dữ liệu đã restore. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-using-volumes.html?utm_source=chatgpt.com)

---

# 7. XFS duplicate UUID scenario

Nếu source và restored XFS filesystem cùng attach trên một RHEL machine, chúng có thể mang cùng filesystem UUID.

Khi mount, bạn có thể gặp:

```text
Filesystem has duplicate UUID ... can't mount
```

Red Hat xác nhận XFS ngăn hai filesystem có cùng UUID được mount đồng thời; một workaround cho inspection là `nouuid`. [Red Hat Customer Portal](https://access.redhat.com/solutions/5494781?utm_source=chatgpt.com)

Tạo isolated recovery mount:

```bash
sudo mkdir -p /mnt/day5-restore
```

Nếu XFS và source volume vẫn đang mount:

```bash
sudo mount -o ro,nouuid \
  <RESTORED_DEVICE> \
  /mnt/day5-restore
```

Ví dụ:

```bash
sudo mount -o ro,nouuid \
  /dev/nvme2n1 \
  /mnt/day5-restore
```

`ro` giúp recovery verification không write vào clone.

Nếu không phải XFS hoặc không có UUID conflict, dùng mount option phù hợp filesystem.

---

# 8. Verify restored data

```bash
findmnt /mnt/day5-restore
```

```bash
find /mnt/day5-restore/source -type f | sort
```

Count:

```bash
find /mnt/day5-restore/source -type f | wc -l
```

Expected:

```text
3
```

---

# 9. SHA-256 validation

Copy manifest:

```bash
cp /tmp/day5-source.sha256 /tmp/day5-restore.sha256
```

Đổi sang restored working directory:

```bash
cd /mnt/day5-restore/source
```

Run:

```bash
sha256sum -c /tmp/day5-restore.sha256
```

Expected:

```text
./config/app.conf: OK
./file1.txt: OK
./file2.txt: OK
```

Nếu một file báo:

```text
FAILED
```

recovery **không đạt acceptance**, dù snapshot state là `completed`.

---

# 10. Application-readable verification

Ở mức lab file:

```bash
cat /mnt/day5-restore/source/config/app.conf
```

Expected content:

```text
application=day5-lab
environment=sandbox
backup_required=true
```

Đây mới chứng minh file không chỉ tồn tại mà còn readable.

---

# 11. Recovery completion timestamp

Sau khi hash và readability đều pass:

```bash
date -u +"%Y-%m-%dT%H:%M:%SZ"
```

Gọi là:

```text
RECOVERY_VALIDATED
```

---

# 12. Optional EBS initialization awareness

Volume tạo từ snapshot có thể cần initialization/read of blocks trước khi đạt normal performance. AWS nêu rằng việc đọc toàn volume có thể mất từ phút tới giờ tùy instance bandwidth, volume size và provisioned IOPS. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/initalize-volume.html?utm_source=chatgpt.com)

Trong lab nhỏ, không cần chạy aggressive benchmark.

Không chạy:

```bash
fio
```

với write workload bừa bãi trên recovery data.

---

# LAB 6 — SECURE S3 BACKUP BUCKET

## Objective

Tạo S3 bucket phục vụ backup với:

```text
private baseline
Block Public Access
encryption
versioning
lifecycle/retention
least-privilege access
```

---

# 1. Bucket naming

S3 general-purpose bucket name phải globally unique.

Ví dụ:

```bash
export BUCKET="day5-backup-${ACCOUNT_ID}-${LAB_ID}"
```

Check:

```bash
echo "$BUCKET"
```

Không dùng secret, employee ID nhạy cảm hoặc customer name trong bucket name.

---

# 2. Create bucket

`us-east-1` có create-bucket syntax khác các Region thông thường.

```bash
if [ "$AWS_REGION" = "us-east-1" ]; then
  aws s3api create-bucket \
    --bucket "$BUCKET" \
    --region "$AWS_REGION"
else
  aws s3api create-bucket \
    --bucket "$BUCKET" \
    --region "$AWS_REGION" \
    --create-bucket-configuration \
      LocationConstraint="$AWS_REGION"
fi
```

**State-changing.**

---

# 3. Enable Block Public Access

```bash
aws s3api put-public-access-block \
  --bucket "$BUCKET" \
  --public-access-block-configuration \
'BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true'
```

AWS áp dụng effective combination giữa bucket/account/organization controls và sử dụng cấu hình restrictive nhất. New S3 buckets mặc định cũng được bảo vệ khỏi public access; backup bucket nên giữ control này trừ khi có approved use case khác. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-management.html?utm_source=chatgpt.com)

Verify:

```bash
aws s3api get-public-access-block \
  --bucket "$BUCKET"
```

Expected:

```json
{
  "PublicAccessBlockConfiguration": {
    "BlockPublicAcls": true,
    "IgnorePublicAcls": true,
    "BlockPublicPolicy": true,
    "RestrictPublicBuckets": true
  }
}
```

---

# 4. Configure explicit default encryption

S3 hiện tự động encrypt new object uploads bằng SSE-S3 làm baseline, nhưng trong lab ta cấu hình explicit bucket default để evidence rõ ràng. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-config-for-kms-objects.html?utm_source=chatgpt.com)

Tạo configuration:

```bash
cat > /tmp/day5-encryption.json <<'EOF'
{
  "Rules": [
    {
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }
  ]
}
EOF
```

Apply:

```bash
aws s3api put-bucket-encryption \
  --bucket "$BUCKET" \
  --server-side-encryption-configuration \
  file:///tmp/day5-encryption.json
```

Verify:

```bash
aws s3api get-bucket-encryption \
  --bucket "$BUCKET"
```

Expected:

```text
SSEAlgorithm = AES256
```

---

# 5. SSE-KMS alternative

Trong enterprise, nếu requirement cần customer-managed KMS control:

```text
SSE-KMS
```

có thể được dùng thay `AES256`.

Nhưng lúc đó recovery dependency tăng:

```text
S3 permission
+
KMS decrypt permission
+
KMS key available/enabled
+
key policy
```

Không thay đổi KMS key policy trong lab trừ khi có dedicated sandbox key và instructor approval.

---

# 6. Enable versioning

```bash
aws s3api put-bucket-versioning \
  --bucket "$BUCKET" \
  --versioning-configuration Status=Enabled
```

Verify:

```bash
aws s3api get-bucket-versioning \
  --bucket "$BUCKET"
```

Expected:

```json
{
  "Status": "Enabled"
}
```

AWS CLI reference lưu ý khi lần đầu bật versioning, propagation có thể mất một khoảng ngắn và AWS khuyến nghị đợi khoảng 15 phút trước khi phát sinh write/delete operations cần dựa chắc vào versioning behavior. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/s3api/put-bucket-versioning.html?utm_source=chatgpt.com)

Trong course lab, tốt nhất enable versioning ở đầu setup trước các bước khác.

---

# LAB 7 — S3 LIFECYCLE / RETENTION

Lifecycle có khả năng permanent-delete data, nên đây là configuration cần cẩn thận.

---

# 1. Lab lifecycle design

Một rule sandbox ví dụ:

```text
Prefix:
backup/

Current version:
transition after 30 days

Noncurrent:
expire only after 365 days

Incomplete multipart upload:
abort after 7 days
```

Các numbers này chỉ là **lab example**, không phải enterprise retention standard.

Production values phải theo:

```text
RPO
RTO
business retention
compliance
legal requirement
cost
```

---

# 2. Create JSON

```bash
cat > /tmp/day5-lifecycle.json <<'EOF'
{
  "Rules": [
    {
      "ID": "Day5BackupLifecycle",
      "Status": "Enabled",
      "Filter": {
        "Prefix": "backup/"
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        }
      ],
      "NoncurrentVersionExpiration": {
        "NoncurrentDays": 365
      },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
EOF
```

---

# 3. Apply

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket "$BUCKET" \
  --lifecycle-configuration \
  file:///tmp/day5-lifecycle.json
```

---

# 4. Verify

```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket "$BUCKET"
```

S3 Lifecycle có thể transition object và permanently delete noncurrent versions khi expiration criteria được thỏa; vì vậy production lifecycle changes phải được review như data-retention changes chứ không phải cosmetic configuration. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/intro-lifecycle-rules.html?utm_source=chatgpt.com)

---

# 5. Important verification limitation

Bạn không cần đợi 30 hoặc 365 ngày trong lab.

Acceptance là:

```text
rule syntax accepted
correct prefix
correct actions
correct retention intention
evidence captured
```

Không claim rằng lifecycle transition đã thực tế chạy nếu chưa đến thời điểm.

---

# LAB 8 — UPLOAD CONTROLLED S3 BACKUP SET

## Objective

Backup dataset từ EBS lên S3.

---

# 1. Pre-upload manifest

```bash
cd /data/day5/source

sha256sum -c /tmp/day5-source.sha256
```

Phải PASS trước backup.

---

# 2. Upload backup

```bash
aws s3 sync \
  /data/day5/source \
  "s3://$BUCKET/backup/source/"
```

Không cần `--acl public-read`.

Không được làm public.

---

# 3. Verify objects

```bash
aws s3 ls \
  "s3://$BUCKET/backup/source/" \
  --recursive
```

---

# 4. Verify object metadata

Ví dụ:

```bash
aws s3api head-object \
  --bucket "$BUCKET" \
  --key "backup/source/file1.txt"
```

Kiểm tra:

```text
ContentLength
ServerSideEncryption
LastModified
VersionId where applicable
```

---

# 5. Download to isolated verification directory

```bash
mkdir -p /tmp/day5-s3-verify
```

```bash
aws s3 sync \
  "s3://$BUCKET/backup/source/" \
  /tmp/day5-s3-verify/
```

---

# 6. Check file count

```bash
find /tmp/day5-s3-verify -type f | wc -l
```

Compare source:

```bash
find /data/day5/source -type f | wc -l
```

Counts phải match.

---

# 7. Check SHA-256

```bash
cd /tmp/day5-s3-verify

sha256sum -c /tmp/day5-source.sha256
```

Expected:

```text
all OK
```

Đây là first S3 recovery proof.

---

# LAB 9 — VERSIONING, OVERWRITE, DELETE VÀ RECOVERY

Đây là một trong những lab quan trọng nhất của Day 5.

## Objective

Chứng minh:

```text
overwrite ≠ permanent loss
simple delete ≠ necessarily permanent deletion
correct object version can be recovered
```

---

# 1. Create Version 1

```bash
echo "VERSION-1-GOOD-DATA" > /tmp/day5-version-test.txt
```

Hash:

```bash
sha256sum /tmp/day5-version-test.txt
```

Capture:

```bash
export V1_HASH=$(sha256sum /tmp/day5-version-test.txt \
  | awk '{print $1}')

echo "$V1_HASH"
```

Upload:

```bash
export V1_ID=$(aws s3api put-object \
  --bucket "$BUCKET" \
  --key "recovery-test/object.txt" \
  --body /tmp/day5-version-test.txt \
  --query VersionId \
  --output text)

echo "$V1_ID"
```

Record:

```text
V1 ID
V1 hash
V1 timestamp
```

---

# 2. Create Version 2

```bash
echo "VERSION-2-NEW-DATA" > /tmp/day5-version-test.txt
```

Capture hash:

```bash
export V2_HASH=$(sha256sum /tmp/day5-version-test.txt \
  | awk '{print $1}')
```

Upload same key:

```bash
export V2_ID=$(aws s3api put-object \
  --bucket "$BUCKET" \
  --key "recovery-test/object.txt" \
  --body /tmp/day5-version-test.txt \
  --query VersionId \
  --output text)

echo "$V2_ID"
```

Now:

```text
same bucket
same key
different version ID
```

---

# 3. List versions

```bash
aws s3api list-object-versions \
  --bucket "$BUCKET" \
  --prefix "recovery-test/object.txt"
```

AWS `list-object-versions` trả metadata của object versions và cần `s3:ListBucketVersions`. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/s3api/list-object-versions.html?utm_source=chatgpt.com)

Expected:

```text
V2 IsLatest = true
V1 IsLatest = false
```

---

# 4. Simulate accidental delete

**State-changing, sandbox only.**

```bash
aws s3api delete-object \
  --bucket "$BUCKET" \
  --key "recovery-test/object.txt"
```

Trong version-enabled bucket, simple delete thường tạo `delete marker` thay vì permanently deleting existing versions.

---

# 5. List versions và delete markers

```bash
aws s3api list-object-versions \
  --bucket "$BUCKET" \
  --prefix "recovery-test/object.txt"
```

Bạn cần thấy:

```text
DeleteMarkers
Versions
```

Current key có thể dường như “mất”, nhưng V1/V2 vẫn tồn tại.

---

# 6. Recovery requirement

Giả sử business yêu cầu:

```text
Recover Version 1
```

Không phục hồi trực tiếp đè lên source key.

Dùng isolated destination local:

```bash
mkdir -p /tmp/day5-version-restore
```

---

# 7. Download specific version

```bash
aws s3api get-object \
  --bucket "$BUCKET" \
  --key "recovery-test/object.txt" \
  --version-id "$V1_ID" \
  /tmp/day5-version-restore/object.txt
```

---

# 8. Verify content

```bash
cat /tmp/day5-version-restore/object.txt
```

Expected:

```text
VERSION-1-GOOD-DATA
```

---

# 9. Verify hash

```bash
export RESTORED_HASH=$(sha256sum \
  /tmp/day5-version-restore/object.txt \
  | awk '{print $1}')

printf 'Expected: %s\nActual:   %s\n' \
  "$V1_HASH" \
  "$RESTORED_HASH"
```

Compare:

```bash
test "$V1_HASH" = "$RESTORED_HASH"
echo $?
```

Expected exit status:

```text
0
```

---

# 10. Acceptance

PASS only if:

```text
Correct version selected
Correct content recovered
SHA-256 matches
Restore performed in isolated location
Original versions unchanged
```

Không dùng ETag làm sole integrity proof. Syllabus cấm coi generic `ETag` là universal content hash. Infrastructure_Operations_14_Da…

---

# LAB 10 — SAFE ACCESS-DENIED FAILURE INJECTION

## Objective

Chứng minh:

```text
data exists
≠
data is recoverable
```

---

# Recommended implementation

Instructor/admin chuẩn bị một profile:

```text
day5-denied
```

profile này authenticated bình thường nhưng **không có `s3:GetObject`** đối với lab backup prefix.

Không share credentials.

Không paste temporary session credentials vào transcript.

---

# Attempt recovery

```bash
aws s3api get-object \
  --profile day5-denied \
  --bucket "$BUCKET" \
  --key "recovery-test/object.txt" \
  --version-id "$V1_ID" \
  /tmp/denied-test.txt
```

Expected:

```text
AccessDenied
```

Record:

```text
timestamp
principal ARN
bucket
key
version ID
operation
error code
ticket/lab ID
```

Không record secrets.

---

# Troubleshooting sequence

Đừng lập tức modify bucket policy.

Kiểm tra:

```bash
aws sts get-caller-identity \
  --profile day5-denied
```

Sau đó reason:

```text
Principal?
Required action?
s3:GetObjectVersion?
Bucket/resource scope?
Explicit deny?
Permissions boundary?
SCP?
Bucket policy?
KMS dependency?
```

---

# LAB 11 — OPTIONAL SSE-KMS FAILURE FIXTURE

Đây là **instructor-prepared optional lab**.

Không disable KMS key để “tạo lỗi”.

Không schedule key deletion.

Không thay production key policy.

---

## Scenario

S3 object được encrypt bằng customer-managed KMS key.

Recovery profile có:

```text
s3:GetObject
```

nhưng thiếu KMS decrypt authorization.

Result:

```text
object exists
but recovery fails
```

Điều này chứng minh DR dependency:

```text
Backup
+
IAM
+
KMS
```

Object Lock cũng không bảo vệ bạn khỏi việc mất khả năng truy cập encryption key; AWS nêu rõ Object Lock không bảo vệ trước mất/deletion của encryption keys. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-managing.html?utm_source=chatgpt.com)

Evidence phải ghi:

```text
key identifier — non-secret
recovery role
error code
dependency
escalation owner
```

---

# LAB 12 — OPTIONAL S3 REPLICATION

Syllabus coi replication là optional lab tùy environment.

Replication có nhiều prerequisite và có thể phát sinh thêm cost.

Source và destination buckets cần versioning enabled và S3 cần được cấp permission để replicate. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication-requirements.html?utm_source=chatgpt.com)

Một topology:

```text
Source bucket
ap-southeast-1
      │
      │ asynchronous
      ▼
Destination bucket
<another approved Region/account>
```

Nếu lab đã preconfigured:

```bash
aws s3api get-bucket-replication \
  --bucket "$BUCKET"
```

Upload controlled test object.

Sau đó verify destination:

```bash
aws s3api head-object \
  --bucket "<destination-bucket>" \
  --key "<test-key>"
```

Evidence:

```text
source version ID
destination presence
replication status
time lag observed
encryption dependency
```

Không kết luận:

```text
replication = backup
```

Replication chỉ là một data-protection mechanism; recovery point strategy vẫn phải được thiết kế riêng.

---

# LAB 13 — OPTIONAL OBJECT LOCK

Syllabus yêu cầu `Object Lock awareness`, optional implementation nơi hỗ trợ.

Object Lock chỉ nên dùng trong dedicated lab bucket.

Object Lock làm việc với versioned buckets. [AWS Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock-configure.html?utm_source=chatgpt.com)

Không bật retention dài trong main cleanup bucket vì bạn có thể tự ngăn mình xóa object sau lab.

Nếu có instructor-provided Object Lock bucket, chỉ inspect:

```bash
aws s3api get-object-lock-configuration \
  --bucket "<object-lock-lab-bucket>"
```

Học cách nhận diện:

```text
Governance
Compliance
Legal Hold
RetainUntilDate
```

Không bypass Governance retention nếu không nằm trong scope lab.

---

# LAB 14 — AWS BACKUP INTEGRATION

Có hai track.

## Track A — Read-only inspection

Phù hợp nếu account đã có AWS Backup configuration.

List vaults:

```bash
aws backup list-backup-vaults \
  --region "$AWS_REGION"
```

List backup plans:

```bash
aws backup list-backup-plans \
  --region "$AWS_REGION"
```

List protected resources:

```bash
aws backup list-protected-resources \
  --region "$AWS_REGION"
```

---

# 1. Build EBS resource ARN

```bash
export EBS_ARN="arn:aws:ec2:${AWS_REGION}:${ACCOUNT_ID}:volume/${VOL_ID}"

echo "$EBS_ARN"
```

---

# 2. Find recovery points

```bash
aws backup list-recovery-points-by-resource \
  --resource-arn "$EBS_ARN" \
  --region "$AWS_REGION"
```

AWS lưu ý command này trả AWS Backup recovery points cho resource theo ARN. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/backup/list-recovery-points-by-resource.html?utm_source=chatgpt.com)

---

# Track B — Authorized backup job

Chỉ khi có:

```text
approved Backup vault
AWS Backup service role
permissions
budget
```

Conceptual command:

```bash
aws backup start-backup-job \
  --backup-vault-name "<LAB_VAULT>" \
  --resource-arn "$EBS_ARN" \
  --iam-role-arn "<APPROVED_BACKUP_SERVICE_ROLE_ARN>" \
  --region "$AWS_REGION"
```

Sau đó:

```bash
aws backup list-backup-jobs \
  --by-resource-arn "$EBS_ARN" \
  --region "$AWS_REGION"
```

`list-backup-jobs` hiện trả recent backup-job information và không nên được coi là unlimited historical audit archive. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/backup/list-backup-jobs.html?utm_source=chatgpt.com)

---

# AWS Backup Restore Testing

AWS Backup hiện có `Restore testing` để tự động và định kỳ đánh giá restore viability cũng như theo dõi restore job duration. [AWS Documentation](https://docs.aws.amazon.com/aws-backup/latest/devguide/restore-testing.html?utm_source=chatgpt.com)

Nếu account có plan:

```bash
aws backup list-restore-testing-plans \
  --region "$AWS_REGION"
```

Với Infrastructure Operations, đây là concept rất mạnh:

```text
Backup policy
   ↓
Recovery point
   ↓
Periodic restore test
   ↓
Validation
   ↓
RTO measurement
```

thay vì chỉ:

```text
backup completed every night
```

---

# LAB 15 — CALCULATE OBSERVED RPO

Giả sử:

```text
Recovery incident/reference time:
10:00:00 UTC

Latest verified recovery point:
09:42:00 UTC
```

Observed RPO:

```text
18 minutes
```

Formula:

```text
Observed RPO
=
Recovery reference time
-
timestamp of latest verified recoverable data
```

Nếu:

```text
Target RPO = 15 minutes
Observed RPO = 18 minutes
```

Result:

```text
FAIL
```

Không “round down” để pass.

---

# Practical RPO record

Ví dụ:

```text
Target RPO:          15 minutes
Incident reference:  2026-10-20T10:00:00Z
Recovery point:      2026-10-20T09:42:00Z
Observed RPO:        18 minutes
Result:              FAIL
Reason:              recovery point older than target
```

---

# LAB 16 — CALCULATE OBSERVED RTO

Ghi:

```text
Declared recovery start
```

tại thời điểm recovery bắt đầu.

Ví dụ:

```text
10:05:00 UTC
```

Sau khi:

```text
restore complete
+
files validated
+
hashes pass
+
content readable
```

ghi:

```text
10:21:00 UTC
```

Observed RTO:

```text
16 minutes
```

Formula:

```text
Observed RTO
=
validated recovery completion
-
declared recovery start
```

Theo syllabus, observed RPO phải dựa trên latest recoverable data point và observed RTO phải sử dụng declared recovery start/end. Infrastructure_Operations_14_Da…

---

# LAB 17 — CLOUDWATCH EC2/EBS MONITORING

## Objective

Xác minh telemetry trước khi tạo alarm.

---

# 1. List EC2 metrics

```bash
aws cloudwatch list-metrics \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value="$INSTANCE_ID" \
  --region "$AWS_REGION"
```

Không cần lưu toàn bộ giant output nếu quá lớn; filter những metric liên quan.

---

# 2. EBS metrics

```bash
aws cloudwatch list-metrics \
  --namespace AWS/EBS \
  --dimensions Name=VolumeId,Value="$VOL_ID" \
  --region "$AWS_REGION"
```

AWS EBS metrics được publish tự động và AWS hiện mô tả EBS metric data có collection period một phút. [AWS Documentation](https://docs.aws.amazon.com/ebs/latest/userguide/using_cloudwatch_ebs.html?utm_source=chatgpt.com)

---

# 3. Query recent EBS read operations

Trên GNU/Linux:

```bash
export CW_START=$(date -u -d '30 minutes ago' \
  +"%Y-%m-%dT%H:%M:%SZ")

export CW_END=$(date -u \
  +"%Y-%m-%dT%H:%M:%SZ")
```

Query:

```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/EBS \
  --metric-name VolumeReadOps \
  --dimensions Name=VolumeId,Value="$VOL_ID" \
  --start-time "$CW_START" \
  --end-time "$CW_END" \
  --period 300 \
  --statistics Sum \
  --region "$AWS_REGION"
```

Nếu không có datapoint:

Không kết luận ngay:

```text
CloudWatch broken
```

Check:

```text
correct Region?
correct VolumeId?
volume attached?
activity occurred?
time range?
metric available for resource/type?
```

---

# LAB 18 — CLOUDWATCH ALARM

## Objective

Tạo một alarm sandbox cho EC2 CPU metric.

Không cần gắn destructive automated action.

---

# 1. Create alarm

**State-changing.**

```bash
export ALARM_NAME="Day5-EC2-CPU-${INSTANCE_ID}"
```

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "$ALARM_NAME" \
  --alarm-description "Day5 sandbox CPU alarm" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value="$INSTANCE_ID" \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data missing \
  --region "$AWS_REGION"
```

No `AlarmActions` được gắn.

Do đó alarm này không restart/stop/notify resource.

AWS ghi nhận newly created metric alarm thường bắt đầu tại `INSUFFICIENT_DATA` rồi được evaluation để chuyển sang state thích hợp. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-alarm.html?utm_source=chatgpt.com)

---

# 2. Verify alarm

```bash
aws cloudwatch describe-alarms \
  --alarm-names "$ALARM_NAME" \
  --region "$AWS_REGION"
```

Check:

```text
MetricName
Namespace
Dimensions
Threshold
Period
EvaluationPeriods
ComparisonOperator
StateValue
```

State có thể:

```text
OK
ALARM
INSUFFICIENT_DATA
```

`INSUFFICIENT_DATA` không đồng nghĩa healthy hoặc failed.

---

# 3. Không ép load production

Không chạy CPU stress tool trên production chỉ để alarm đổi trạng thái.

Trong lab nếu mentor muốn trigger, dùng controlled workload hoặc custom metric fixture.

---

# LAB 19 — CLOUDWATCH FAILURE INJECTION

Tạo một second alarm cố tình có wrong dimension.

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name "Day5-Bad-Dimension-Test" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-doesnotexist \
  --statistic Average \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data missing \
  --region "$AWS_REGION"
```

Expected:

```text
INSUFFICIENT_DATA
```

Troubleshooting:

```text
Metric?
Namespace?
Dimension name?
Dimension value?
Region?
Period?
Data available?
```

Cleanup:

```bash
aws cloudwatch delete-alarms \
  --alarm-names "Day5-Bad-Dimension-Test" \
  --region "$AWS_REGION"
```

Lesson:

```text
Alarm object exists
≠
alarm is monitoring correct resource
```

---

# LAB 20 — CLOUDTRAIL EVENT INVESTIGATION

## Objective

Tìm audit evidence của các management operations Day 5.

---

# 1. Find CreateSnapshot event

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
  AttributeKey=EventName,AttributeValue=CreateSnapshot \
  --max-results 20 \
  --region "$AWS_REGION"
```

Tìm event liên quan `SNAPSHOT_ID`.

---

# 2. Find AttachVolume

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
  AttributeKey=EventName,AttributeValue=AttachVolume \
  --max-results 20 \
  --region "$AWS_REGION"
```

---

# 3. Find S3 management operation

Ví dụ:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
  AttributeKey=EventName,AttributeValue=PutBucketVersioning \
  --max-results 20 \
  --region "$AWS_REGION"
```

Hoặc:

```bash
aws cloudtrail lookup-events \
  --lookup-attributes \
  AttributeKey=EventName,AttributeValue=PutBucketEncryption \
  --max-results 20 \
  --region "$AWS_REGION"
```

---

# 4. Fields cần inspect

Trong CloudTrail event:

```text
eventTime
eventName
eventSource
awsRegion
userIdentity
sourceIPAddress
requestParameters
responseElements
errorCode
errorMessage
resources
```

CloudTrail `eventTime` được ghi theo UTC. [AWS Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-record-contents.html?utm_source=chatgpt.com)

---

# 5. Critical CloudTrail limitation

CloudTrail Event history hiện chỉ cung cấp **90 ngày management events trong một Region** và **không hiển thị data events**. Nếu cần S3 object-level `GetObject/PutObject` history, phải có trail hoặc event data store được cấu hình để capture data events. [AWS Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/tutorial-event-history.html?utm_source=chatgpt.com)

Vì vậy:

```text
Không thấy PutBucketVersioning
```

và:

```text
Không thấy GetObject
```

là hai loại vấn đề khác nhau.

`PutBucketVersioning` là management/control-plane operation.

`GetObject` là S3 data-plane activity.

---

# CloudTrail investigation flow

```text
Correct account?
     ↓
Correct Region?
     ↓
Correct time window?
     ↓
Correct event name?
     ↓
Management or data event?
     ↓
Is required logging configured?
     ↓
Retention still available?
```

Không được viết:

```text
No event found, therefore action never happened.
```

trừ khi bạn đã chứng minh logging coverage đầy đủ.

---

# LAB 21 — AWS BUDGET CONTROL

AWS Console thường dễ hơn cho beginner lab vì Budgets CLI object khá verbose.

## Objective

Tạo một sandbox monthly cost budget.

---

# Example

Giả sử approved training budget:

```text
USD 10/month
```

**Đây chỉ là example; dùng amount theo instructor/account policy.**

Trong AWS Budgets:

```text
Budget type:
Cost budget

Period:
Monthly

Budget amount:
<approved lab amount>

Notification:
Actual cost threshold

Optional:
Forecasted cost threshold
```

Có thể chọn ví dụ:

```text
50%
80%
100%
```

nếu training policy cho phép.

Không dùng personal email nếu organization yêu cầu approved distribution/SNS.

---

# Verify bằng CLI

Read-only:

```bash
aws budgets describe-budgets \
  --account-id "$ACCOUNT_ID"
```

Tìm budget name của Day 5.

AWS Budgets có thể alert trên actual hoặc forecasted spending; dữ liệu Budgets không phải real-time và AWS hiện cho biết thường được refresh nhiều lần mỗi ngày. [AWS Documentation](https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html?utm_source=chatgpt.com)

Do đó:

```text
resource created 30 seconds ago
```

không nhất thiết xuất hiện ngay trong budget spend.

---

# Important

Budget không mặc nhiên là:

```text
hard spending cap
```

AWS Budgets có thể hỗ trợ budget actions trong approved configuration, nhưng không được assume budget amount tự động shutdown account resources. [AWS Documentation](https://docs.aws.amazon.com/cli/latest/reference/budgets/create-budget-action.html?utm_source=chatgpt.com)

---

# LAB 22 — COST DRIVER REVIEW

## Objective

Xác định resource nào của lab vẫn có khả năng phát sinh cost.

---

# 1. EC2 states

```bash
aws ec2 describe-instances \
  --region "$AWS_REGION" \
  --query 'Reservations[].Instances[].{
    InstanceId:InstanceId,
    State:State.Name,
    Type:InstanceType,
    PublicIp:PublicIpAddress
  }'
```

---

# 2. Unattached EBS volumes

```bash
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --region "$AWS_REGION" \
  --query 'Volumes[].{
    VolumeId:VolumeId,
    Size:Size,
    Type:VolumeType,
    AZ:AvailabilityZone,
    CreateTime:CreateTime
  }'
```

`available` EBS volume là **candidate for review**, không phải automatic delete candidate.

Có thể là:

```text
rollback disk
recovery disk
staging disk
forgotten resource
```

---

# 3. Snapshots

```bash
aws ec2 describe-snapshots \
  --owner-ids self \
  --region "$AWS_REGION" \
  --query 'Snapshots[].{
    SnapshotId:SnapshotId,
    StartTime:StartTime,
    VolumeId:VolumeId,
    State:State,
    Description:Description
  }'
```

---

# 4. Elastic IP addresses

```bash
aws ec2 describe-addresses \
  --region "$AWS_REGION" \
  --query 'Addresses[].{
    PublicIp:PublicIp,
    AllocationId:AllocationId,
    AssociationId:AssociationId,
    InstanceId:InstanceId,
    NetworkInterfaceId:NetworkInterfaceId
  }'
```

Lưu ý: auto-assigned public IP của EC2 không nhất thiết xuất hiện như Elastic IP trong `describe-addresses`; kiểm tra cả EC2 inventory.

AWS hiện tính phí public IPv4 address do AWS cung cấp ở mức niêm yết `$0.005/address/hour` cho cả in-use và idle trong các trường hợp được VPC pricing mô tả; pricing phải luôn được re-check tại thời điểm production estimation. [Amazon Web Services, Inc.](https://aws.amazon.com/vpc/pricing/?trk=public_post_comment-text&utm_source=chatgpt.com)

---

# 5. S3 current objects

```bash
aws s3 ls \
  "s3://$BUCKET" \
  --recursive \
  --summarize
```

Nhưng command này không cho full picture về noncurrent versions.

Với versioned bucket:

```bash
aws s3api list-object-versions \
  --bucket "$BUCKET"
```

Review:

```text
Versions
DeleteMarkers
Current/noncurrent versions
```

S3 storage cost không chỉ có bytes stored mà còn có request, retrieval, data transfer, replication và các feature-related components. [Amazon Web Services, Inc.](https://aws.amazon.com/s3/pricing/?nc2=h_pr_s3&sc_channel=ps&trk=aebc39a1-139c-43bb-8354-211ac811b83a&utm_source=chatgpt.com)

---

# 6. Cost worksheet

Bạn nên tạo bảng:

| Resource    | Cost driver                       | Still required? | Action                |
| ----------- | --------------------------------- | --------------- | --------------------- |
| EC2         | compute + attached resources      | Yes/No          | retain/stop/terminate |
| EBS volume  | GiB/type/performance              | Yes/No          | retain/delete         |
| Snapshot    | snapshot storage                  | Yes/No          | retain/delete         |
| S3          | storage/versions/requests         | Yes/No          | retain/cleanup        |
| Public IPv4 | hourly charge                     | Yes/No          | retain/release        |
| Replication | storage/requests/transfer         | Yes/No          | retain/disable        |
| CloudWatch  | metrics/logs/alarms depending use | Yes/No          | retain/delete         |

---

# LAB 23 — STOP vs TERMINATE REVIEW

Trước cleanup, hiểu rõ:

```text
stop
≠
terminate
```

Nếu chỉ:

```bash
aws ec2 stop-instances ...
```

compute instance dừng, nhưng các persistent resources như EBS có thể vẫn tồn tại và tiếp tục tạo cost.

Trước `terminate`, kiểm tra:

```bash
aws ec2 describe-instance-attribute \
  --instance-id "$INSTANCE_ID" \
  --attribute blockDeviceMapping \
  --region "$AWS_REGION"
```

Đặc biệt xem:

```text
DeleteOnTermination
```

Không terminate Day 4/Day 5 base instance nếu nó còn cần cho Day sau hoặc course environment.

---

# LAB 24 — CONTROLLED CLEANUP

Cleanup phải theo dependency order.

Không xóa resource tùy tiện từ trên xuống.

---

# 1. Preserve evidence first

Hash evidence:

```bash
cd "$HOME/day5-evidence"

find . -type f -exec sha256sum {} + \
  | sort -k2 \
  > evidence-manifest.sha256
```

Verify:

```bash
sha256sum -c evidence-manifest.sha256
```

---

# 2. Unmount restored EBS

```bash
sudo umount /mnt/day5-restore
```

Verify:

```bash
findmnt /mnt/day5-restore
```

---

# 3. Detach restored EBS

```bash
aws ec2 detach-volume \
  --volume-id "$RESTORE_VOL_ID" \
  --region "$AWS_REGION"
```

Wait:

```bash
aws ec2 wait volume-available \
  --volume-ids "$RESTORE_VOL_ID" \
  --region "$AWS_REGION"
```

Delete only after evidence and approval:

```bash
aws ec2 delete-volume \
  --volume-id "$RESTORE_VOL_ID" \
  --region "$AWS_REGION"
```

---

# 4. Main EBS cleanup

Nếu main Day5 EBS không cần giữ:

First remove persistent mount carefully.

```bash
sudo umount /data/day5
```

Restore `/etc/fstab` or remove only the Day5 line.

Safest lab rollback if no later valid changes:

```bash
sudo cp /etc/fstab.day5.bak /etc/fstab
```

Verify:

```bash
sudo findmnt --verify --verbose
```

Detach:

```bash
aws ec2 detach-volume \
  --volume-id "$VOL_ID" \
  --region "$AWS_REGION"
```

Wait:

```bash
aws ec2 wait volume-available \
  --volume-ids "$VOL_ID" \
  --region "$AWS_REGION"
```

Delete only if authorized:

```bash
aws ec2 delete-volume \
  --volume-id "$VOL_ID" \
  --region "$AWS_REGION"
```

---

# 5. Snapshot decision

Do **not** automatically delete snapshot.

Decision:

```text
Needed as course evidence/recovery point?
YES → retain and tag

NO → delete after approval
```

If authorized:

```bash
aws ec2 delete-snapshot \
  --snapshot-id "$SNAPSHOT_ID" \
  --region "$AWS_REGION"
```

---

# 6. Delete CloudWatch alarm

```bash
aws cloudwatch delete-alarms \
  --alarm-names "$ALARM_NAME" \
  --region "$AWS_REGION"
```

Verify:

```bash
aws cloudwatch describe-alarms \
  --alarm-names "$ALARM_NAME" \
  --region "$AWS_REGION"
```

---

# 7. S3 cleanup caveat

Versioned S3 bucket không thể được emptied hoàn chỉnh chỉ bằng việc xóa current visible objects.

Bạn phải account for:

```text
current versions
noncurrent versions
delete markers
```

Không dùng mass-delete script trong production.

Trong Day 5, nếu organization provisioned bucket cho learner, thường instructor nên cleanup theo approved runbook.

Nếu bucket chứa Object Lock protected versions, deletion có thể intentionally bị blocked.

---

# 8. Final inventory

Sau cleanup:

```bash
aws ec2 describe-volumes \
  --region "$AWS_REGION"
```

Filter lab tags.

Review snapshots.

Review bucket.

Review alarms.

Review EC2.

Review public IPv4.

Không kết luận cleanup hoàn chỉnh chỉ vì EC2 đã stopped.

---

# FAILURE INJECTION SET

Syllabus yêu cầu bạn không chỉ làm happy path mà còn phải troubleshoot.

Dưới đây là 5 failure scenarios an toàn.

---

## Failure 1 — EBS wrong device assumption

### Symptom

Console nói:

```text
/dev/sdf
```

nhưng:

```bash
lsblk
```

không có `/dev/sdf`.

### Investigation

Compare `lsblk` before/after attach.

Nitro may show:

```text
/dev/nvme1n1
```

### Correct response

Không chạy:

```bash
mkfs /dev/sdf
```

mù quáng.

### Evidence

```text
AWS attachment
lsblk before
lsblk after
identified actual device
```

---

## Failure 2 — Restored XFS volume won't mount

### Symptom

```text
duplicate UUID
```

Check:

```bash
dmesg | tail -n 50
```

### Root cause

Snapshot clone carries same XFS filesystem UUID.

### Safe lab recovery

```bash
sudo mount -o ro,nouuid \
  "$RESTORE_DEVICE" \
  /mnt/day5-restore
```

Red Hat documents this duplicate-XFS-UUID condition and `nouuid` workaround. [Red Hat Customer Portal](https://access.redhat.com/solutions/5494781?utm_source=chatgpt.com)

### Escalate

Nếu design cần restored filesystem trở thành permanent independent production filesystem, UUID-handling procedure phải theo OS/runbook; không tự thay metadata trên production.

---

## Failure 3 — S3 recovery returns AccessDenied

### Investigation

```bash
aws sts get-caller-identity
```

Sau đó xem:

```text
identity policy
bucket policy
explicit deny
SCP
permissions boundary
version permission
KMS permission
```

### Do not

Không dùng:

```text
AdministratorAccess
s3:*
kms:*
```

chỉ để “test nhanh”.

---

## Failure 4 — CloudWatch alarm stuck at INSUFFICIENT_DATA

Check:

```text
namespace
metric name
dimension
Region
period
resource activity
```

Không tăng resource load trước khi xác định alarm có trỏ đúng metric hay không.

---

## Failure 5 — CloudTrail không thấy S3 object access

Check:

```text
Was it management event or data event?
Is data-event logging configured?
Correct Region?
Correct account?
Correct time?
Within 90 days?
```

CloudTrail Event history không chứa S3 data events. [AWS Documentation](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events-console.html?utm_source=chatgpt.com)

---

# TROUBLESHOOTING MASTER FLOW — DAY 5

Khi bất kỳ bước nào fail, dùng thứ tự:

```text
1. Identity
      ↓
2. Account / Region
      ↓
3. Resource ID
      ↓
4. Resource state
      ↓
5. Permission
      ↓
6. Dependency
      ↓
7. OS state
      ↓
8. Logs / CloudWatch
      ↓
9. CloudTrail
      ↓
10. Minimal reversible change
      ↓
11. Verify
      ↓
12. Evidence
```

Không dùng:

```text
delete/recreate
reboot
chmod 777
AdministratorAccess
disable encryption
disable Block Public Access
```

như generic troubleshooting shortcut.

---

# DAY 5 EVIDENCE PACKAGE

Một package tốt có thể gồm:

```text
DAY5/
│
├── 00_identity/
│   ├── caller-identity.json
│   └── account-region.txt
│
├── 01_ebs/
│   ├── pre-volumes.json
│   ├── created-volume.json
│   ├── lsblk-before.txt
│   ├── lsblk-after.txt
│   ├── filesystem.txt
│   └── fstab-verification.txt
│
├── 02_snapshot/
│   ├── snapshot.json
│   ├── source-file-count.txt
│   └── source-manifest.sha256
│
├── 03_ebs-restore/
│   ├── restored-volume.json
│   ├── restored-lsblk.txt
│   ├── restored-file-count.txt
│   ├── restored-hash-check.txt
│   └── rto-times.txt
│
├── 04_s3/
│   ├── public-access-block.json
│   ├── encryption.json
│   ├── versioning.json
│   ├── lifecycle.json
│   ├── object-versions.json
│   ├── restored-file-count.txt
│   └── s3-hash-check.txt
│
├── 05_failure-tests/
│   ├── access-denied.txt
│   └── diagnosis.txt
│
├── 06_monitoring/
│   ├── cloudwatch-alarm.json
│   └── ebs-metrics.json
│
├── 07_audit/
│   ├── cloudtrail-events.json
│   └── audit-gap.txt
│
├── 08_cost/
│   ├── cost-review.csv
│   └── cleanup.txt
│
├── 09_dr/
│   ├── rpo-rto.txt
│   └── recovery-checklist.txt
│
└── evidence-manifest.sha256
```

---

# Required Evidence Fields

Mỗi item quan trọng nên có:

```text
Evidence ID
Source system
Account
Region
Resource ID
Environment
Ticket/Lab reference
Collection timestamp UTC
Command/query/filter
Expected result
Actual result
Result PASS/FAIL
SHA-256 if file artifact
Limitations/gaps
Collector
```

Không bao giờ lưu:

```text
SecretAccessKey
SessionToken
password
private key
MFA seed
KMS key material
```

---

# RPO/RTO REPORT TEMPLATE

Cuối lab bạn phải có dạng:

```text
Workload:
Day5 sandbox backup dataset

Target RPO:
30 minutes

Recovery incident/reference:
2026-10-20T10:00:00Z

Latest verified recovery point:
2026-10-20T09:45:00Z

Observed RPO:
15 minutes

RPO Result:
PASS


Target RTO:
60 minutes

Recovery Start:
2026-10-20T10:05:00Z

Recovery Validation Complete:
2026-10-20T10:23:00Z

Observed RTO:
18 minutes

RTO Result:
PASS


Recovery validation:
File count: PASS
SHA-256: PASS
Readable content: PASS
IAM access: PASS
Encryption dependency: PASS

Known limitations:
S3 data events were not configured for this sandbox,
therefore CloudTrail Event history cannot prove object-level reads.
```

Đây là loại kết luận reviewer có thể defend.

---

# ESCALATION MATRIX

## L1 có thể xử lý nếu runbook cho phép

```text
Inspect EC2/EBS state
Inspect lsblk/mount
Inspect CloudWatch
Inspect CloudTrail Event history
Verify bucket settings
Verify versions
Collect evidence
Perform pre-approved standard recovery test
Identify obvious cost waste
```

## L2 thường xử lý

```text
EBS/filesystem issue
XFS duplicate UUID
IAM/bucket-policy diagnosis
KMS access dependency
S3 lifecycle/version recovery
Backup job failure
CloudWatch alarm design
DR recovery analysis
```

## Escalate ngay

```text
Production filesystem corruption
Possible permanent data loss
KMS key unavailable/deletion concern
Public backup exposure
Unknown object/version deletion
RPO/RTO breach
Production restore
Cross-account policy change
Object Lock retention conflict
Large unexpected cost
Potential security compromise
```

---

# PRACTICAL ACCEPTANCE CHECKLIST

Bạn chỉ nên đánh dấu Day 5 Practical hoàn thành khi tất cả các mục relevant đều có evidence:

### EBS

- [ ] Correct identity/account/Region verified.
- [ ] New EBS volume created.
- [ ] Volume attached to correct instance.
- [ ] AWS device mapping distinguished from Linux device.
- [ ] Existing filesystem checked before `mkfs`.
- [ ] Filesystem created safely.
- [ ] Volume mounted.
- [ ] Persistent mount configured.
- [ ] `/etc/fstab` backed up.
- [ ] Persistent mount verified.
- [ ] Test dataset created.
- [ ] SHA-256 manifest generated.

### EBS Backup/Recovery

- [ ] Snapshot created.
- [ ] Snapshot reached `completed`.
- [ ] Restored EBS volume created.
- [ ] Restored volume attached.
- [ ] Restored volume was **not reformatted**.
- [ ] Restore mounted in isolated location.
- [ ] File count matches.
- [ ] SHA-256 matches.
- [ ] Content readable.
- [ ] XFS duplicate UUID issue understood where applicable.

### S3

- [ ] Private backup bucket created.
- [ ] Block Public Access verified.
- [ ] Encryption verified.
- [ ] Versioning enabled.
- [ ] Lifecycle/retention configuration recorded.
- [ ] Controlled backup uploaded.
- [ ] Backup downloaded to isolated destination.
- [ ] File count verified.
- [ ] SHA-256 verified.
- [ ] Multiple object versions created.
- [ ] Overwrite tested.
- [ ] Simple delete/delete-marker behavior observed.
- [ ] Intended version recovered.
- [ ] Generic ETag not treated as universal content hash.

### DR

- [ ] Access-denied recovery scenario tested.
- [ ] KMS dependency understood.
- [ ] Target RPO documented.
- [ ] Observed RPO calculated.
- [ ] Target RTO documented.
- [ ] Observed RTO calculated.
- [ ] Recovery dependency checklist completed.
- [ ] Audit gaps documented.

### Monitoring/Audit

- [ ] EC2 CloudWatch metrics inspected.
- [ ] EBS CloudWatch metrics inspected.
- [ ] CloudWatch alarm configured.
- [ ] Alarm state interpreted correctly.
- [ ] CloudTrail EBS management event located.
- [ ] CloudTrail S3 management event located.
- [ ] Management/data-event distinction understood.
- [ ] 90-day Event history limitation recorded.

### Cost

- [ ] AWS Budget configured or inspected.
- [ ] EC2 cost driver reviewed.
- [ ] EBS cost driver reviewed.
- [ ] Snapshot retention reviewed.
- [ ] S3 versions/storage reviewed.
- [ ] Public IPv4 usage reviewed.
- [ ] Transfer/replication considerations reviewed.
- [ ] Unused resource candidates identified.
- [ ] Stop vs terminate understood.

### Cleanup/Evidence

- [ ] Evidence preserved before cleanup.
- [ ] Evidence manifest generated.
- [ ] Recovery mount unmounted.
- [ ] Temporary recovery volume removed if approved.
- [ ] CloudWatch test alarms removed.
- [ ] Snapshot retention decision documented.
- [ ] S3 retention/cleanup decision documented.
- [ ] Final resource inventory performed.
- [ ] No credentials/secrets present in evidence.

---

# Những lỗi thực hành nguy hiểm nhất Day 5

Nếu chỉ nhớ mười điều từ lab, hãy nhớ những điều này:

1. **Không chạy `mkfs` cho tới khi chắc chắn đó là empty test volume.**

2. **Không format volume restore từ snapshot.**

3. **Không assume `/dev/sdf` trong AWS API sẽ là `/dev/sdf` trong Linux.**

4. **Không sửa `/etc/fstab` rồi reboot mà chưa verify.**

5. **Không coi snapshot `completed` là DR success.**

6. **Không coi `Backup job = COMPLETED` là proof rằng data restore được.**

7. **Không coi S3 `ETag` là universal file hash.**

8. **Không disable security controls để chữa `AccessDenied` nhanh.**

9. **Không kết luận “CloudTrail không có event = action không xảy ra”.**

10. **Không xóa resource chỉ vì nó có vẻ unused; phải xác nhận ownership, dependency, retention và rollback trước.**

---

# End-to-End Operational Scenario

Nếu mentor đưa cho bạn scenario:

> “Application EC2 bị mất data folder. Hãy chứng minh backup có thể recover, kiểm tra audit, xác định RPO/RTO và cleanup lab resources.”

Thứ tự tư duy đúng sẽ là:

```text
1. Verify ticket/scope
2. Verify AWS account/role/Region
3. Identify affected resource
4. Preserve initial evidence
5. Identify recovery point
6. Verify recovery permission
7. Verify encryption/key dependency
8. Restore to isolated destination
9. Verify filesystem/object readability
10. Compare file count
11. Compare SHA-256
12. Record recovery completion
13. Calculate observed RPO
14. Calculate observed RTO
15. Inspect CloudWatch around event window
16. Inspect CloudTrail management activity
17. State data-event/audit limitations
18. Confirm service/data recovery result
19. Review cost/resource leftovers
20. Cleanup only approved temporary resources
21. Hash final evidence package
22. Update ticket/handover/escalation
```

Đây chính là mindset mà **Day 5 Practical** muốn xây dựng: không phải “biết click S3 và EBS”, mà có thể thực hiện một **controlled AWS backup/recovery operation có verification, troubleshooting, security, audit evidence, rollback, cost awareness và escalation boundary**. Syllabus cũng xác định AWS S3 backup/DR là một skill bắt buộc với evidence gồm version-specific isolated restore, verified content/counts, observed `RPO/RTO` và recovery dependency checklist. Infrastructure_Operations_14_Da…
