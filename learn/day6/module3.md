# Day 6 — Module 3: Backup với `tar` / `rsync` + Linux Consolidation Capstone

## 1. Syllabus Alignment

Đây là module cuối cùng của **Day 6 — Linux Fundamentals**.

MASTER_SYLLABUS quy định chính xác:

> **Concept/Lecture:** Logs with `journalctl`, `grep/awk/sed`, Bash scripting, exit codes, cron, **backup using tar/rsync, and Linux consolidation**.  
> **Assignment/Lab:** **Build a health-check and backup script, schedule it, inject a failure, and document escalation evidence.** :chatgpt-content-reference{index="0"}

Vì vậy Module 3 không chỉ là học:

```bash
tar -czf ...
rsync -av ...
```

mà phải hoàn thành toàn bộ chuỗi năng lực:

```text
                  Day 2–5 Linux knowledge
                           │
                           ▼
                    health checks
                           │
                           ▼
                   Bash automation
                           │
                           ▼
                     exit codes
                           │
                           ▼
                      backup
                   ┌───────┴───────┐
                   ▼               ▼
                  tar            rsync
                   │               │
                   └───────┬───────┘
                           ▼
                     verification
                           │
                           ▼
                       restore test
                           │
                           ▼
                        cron
                           │
                           ▼
                    failure injection
                           │
                           ▼
                    troubleshooting
                           │
                  ┌────────┴────────┐
                  ▼                 ▼
             L1 recovery        escalation
                  │                 │
                  └────────┬────────┘
                           ▼
                        evidence
```

Đây chính là **Linux consolidation**: tất cả kỹ năng Day 2–6 bắt đầu hoạt động cùng nhau như một quy trình vận hành thực tế.

---

# 2. Environment assumptions

Các ví dụ bên dưới giả định Linux có:

```text
GNU tar
rsync
GNU/Bash userland
systemd
cron/Cronie tương đương
```

Trước khi học:

```bash
tar --version
rsync --version
bash --version
```

Nếu `rsync` chưa cài:

Debian/Ubuntu thường dùng:

```bash
sudo apt install rsync
```

RHEL/Rocky/AlmaLinux/Amazon Linux-family thường dùng package manager thích hợp như:

```bash
sudo dnf install rsync
```

Nhưng **không cài package lên production chỉ để làm bài học** nếu chưa được phê duyệt.

Một thông tin current-security đáng biết: tại thời điểm hiện tại, upstream rsync đã phát hành **rsync 3.5.1 ngày 21/09/2026**; bản 3.5.0 trước đó là một security release lớn. Máy doanh nghiệp có thể sử dụng version khác do distro lifecycle/backport, nên hãy kiểm tra `rsync --version` và security advisory của distro thay vì chỉ so số version một cách máy móc. :chatgpt-content-reference{index="1"}

Các hành vi `tar` trong bài dựa trên GNU tar, còn `rsync` dựa trên upstream rsync manual hiện hành.

---

# 3. Mục tiêu thực sự của Module 3

Sau bài này, khi một người nói:

> “Backup xong rồi.”

Bạn phải có phản xạ hỏi:

```text
Backup cái gì?

Nguồn ở đâu?

Đích ở đâu?

Thời điểm nào?

Có consistency không?

Command có exit 0 không?

File backup có tồn tại không?

Có đọc được archive không?

Đủ files không?

Metadata có được giữ không?

Có restore thử chưa?

Restore vào đâu?

Có xác nhận data phục hồi đúng không?

Nếu nguồn bị xóa thì backup có còn không?

Nếu rsync dùng --delete thì destination có bị xóa theo không?

Retention là gì?

Ai có quyền đọc backup?

Backup có secrets không?

Scheduler chạy thật chưa?

Nếu job fail thì ai biết?

Recovery procedure là gì?
```

Một file tên:

```text
backup.tar.gz
```

chưa phải bằng chứng rằng bạn có một **recoverable backup**.

---

# 4. Mental Model: archive, compression, copy, synchronization và backup khác nhau

Đây là distinction nền tảng.

## Archive

Archive gom nhiều filesystem objects vào một container:

```text
file1
file2
directory/
permissions
timestamps
...
      │
      ▼
archive.tar
```

GNU `tar` chủ yếu giải quyết vấn đề này.

---

## Compression

Compression làm dữ liệu nhỏ hơn:

```text
archive.tar
     │
   gzip
     ▼
archive.tar.gz
```

`tar` và compression là hai khái niệm khác nhau.

Một:

```text
.tar
```

có thể không compressed.

Một:

```text
.tar.gz
```

là tar archive được gzip-compress.

---

## Copy

Ví dụ:

```bash
cp -a source backup/
```

copy filesystem content.

---

## Synchronization

`rsync` cố làm destination phản ánh source theo rules đã chọn.

Conceptually:

```text
SOURCE
  │
  │ compare
  ▼
DESTINATION

copy new files
update changed files
optionally delete destination extras
```

`rsync` đặc biệt hữu ích vì không nhất thiết copy lại mọi byte mỗi lần.

---

## Backup

Backup là **một khả năng recovery**, không phải tên command.

Một usable backup cần ít nhất:

```text
correct source selection
        +
successful capture
        +
independent/recoverable copy
        +
verification
        +
retention
        +
restore procedure
        +
restore validation
```

Do đó:

```text
tar ≠ automatically a complete backup strategy

rsync ≠ automatically a complete backup strategy
```

Chúng là tools để xây backup workflow.

---

# 5. Backup và mirror không giống nhau

Đây là một trong những khái niệm quan trọng nhất Module 3.

Giả sử source:

```text
/source/
    config.conf
    app.jar
```

Destination mirror:

```text
/backup/
    config.conf
    app.jar
```

Nếu bạn vô tình xóa:

```text
/source/config.conf
```

rồi sync có delete semantics:

```text
/backup/config.conf
```

cũng có thể bị xóa.

Lúc đó mirror đã faithfully replicate lỗi người dùng.

Vì vậy:

```text
mirror = current-state replica

backup = recovery point
```

Một hệ thống backup thực tế thường cần historical versions / snapshots / retention thay vì chỉ một mutable mirror.

Đây là **Supplementary / Beyond the explicit syllabus**, nhưng là mindset bắt buộc nếu sau này vận hành production.

---

# 6. RPO và RTO — Supplementary / Beyond explicit syllabus

Hai thuật ngữ operations bạn sẽ gặp rất nhiều:

### RPO — Recovery Point Objective

Bạn chấp nhận mất tối đa bao nhiêu dữ liệu theo thời gian.

Ví dụ:

```text
backup mỗi 24 giờ

incident lúc 17:00
backup gần nhất 02:00

potential data loss ≈ 15 giờ
```

RPO không tự động bằng schedule, nhưng schedule là một yếu tố lớn.

---

### RTO — Recovery Time Objective

Bạn cần phục hồi dịch vụ nhanh tới đâu.

Ví dụ:

```text
RTO = 30 minutes
```

nhưng backup cần 4 giờ để download và restore thì quy trình không đáp ứng RTO.

Điểm cần nhớ:

```text
backup architecture
không chỉ hỏi:
"backup có tồn tại?"

mà còn:
"restore được nhanh và đúng tới mức nào?"
```

---

# 7. GNU `tar`: Mental model

Tên lịch sử của `tar` là **tape archive**, nhưng ngày nay được sử dụng rất rộng cho disk/file archives.

GNU tar documentation giới thiệu ba operation cơ bản nhất:

```text
--create
--list
--extract
```

hay short forms:

````text
-c
-t
-x
``` :chatgpt-content-reference{index="2"}


Mental model:

```text
Files/directories
       │
       │ tar --create
       ▼
   archive.tar
       │
       ├── tar --list
       │       ↓
       │    inspect
       │
       └── tar --extract
               ↓
           filesystem
````

---

# 8. `tar -c`: tạo archive

Tạo lab:

```bash
mkdir -p "$HOME/day6-module3/source/config"
mkdir -p "$HOME/day6-module3/source/data"
mkdir -p "$HOME/day6-module3/backups"
mkdir -p "$HOME/day6-module3/restore"
```

Tạo sample files:

```bash
printf 'server.port=8080\n' \
    > "$HOME/day6-module3/source/config/app.properties"

printf 'environment=lab\n' \
    > "$HOME/day6-module3/source/config/environment.conf"

printf 'record-001\nrecord-002\n' \
    > "$HOME/day6-module3/source/data/sample.txt"
```

Inspect:

```bash
find "$HOME/day6-module3/source" -maxdepth 3 -type f -print
```

Tạo tar:

```bash
cd "$HOME/day6-module3"

tar -cf backups/day6-backup.tar source/
```

Option:

```text
-c    create archive

-f    archive file follows
```

Đọc:

```text
tar -c -f backups/day6-backup.tar source/
```

---

# 9. Thứ tự `-f` đáng chú ý

Với:

```bash
tar -cf archive.tar source/
```

`archive.tar` là argument của `-f`.

Đừng học:

```text
c = compress
```

Sai.

```text
-c = create
```

Compression là option khác.

---

# 10. `tar -t`: list archive

Không extract ngay.

Inspect:

```bash
tar -tf "$HOME/day6-module3/backups/day6-backup.tar"
```

Expected:

```text
source/
source/config/
source/config/app.properties
source/config/environment.conf
source/data/
source/data/sample.txt
```

Verbose:

```bash
tar -tvf "$HOME/day6-module3/backups/day6-backup.tar"
```

Có thể thấy:

```text
permissions
owner/group
size
timestamp
member name
```

Một nguyên tắc rất tốt:

> **Inspect archive trước khi restore.**

---

# 11. `tar -x`: extract

Không restore trực tiếp đè source.

Tạo staging:

```bash
mkdir -p "$HOME/day6-module3/restore/tar-test"
```

Extract:

```bash
tar -xf \
    "$HOME/day6-module3/backups/day6-backup.tar" \
    -C "$HOME/day6-module3/restore/tar-test"
```

Inspect:

```bash
find "$HOME/day6-module3/restore/tar-test" -type f -print
```

Bạn vừa thực hiện bước quan trọng nhất:

```text
backup
   ↓
restore
```

Không chỉ:

```text
backup
```

---

# 12. `-C`: một option cực kỳ hữu ích

`-C` cho `tar` đổi directory context.

Ví dụ thay vì:

```bash
tar -czf backup.tar.gz /home/alice/app
```

ta có thể:

```bash
tar -czf backup.tar.gz \
    -C /home/alice \
    app
```

Archive members trở thành:

```text
app/...
```

thay vì bạn phải reasoning với full source path.

Điều này giúp tạo archives dễ restore vào staging hơn.

---

# 13. Absolute paths và GNU tar

Giả sử:

```bash
tar -cf backup.tar /etc/myapp
```

GNU tar mặc định loại leading `/` khi lưu/extract archive members, nên member thường trở thành dạng:

```text
etc/myapp/...
```

thay vì `/etc/myapp/...`.

GNU tar làm vậy như một safety behavior; nó cũng có protections với `..` components. Option:

```text
-P
--absolute-names
```

tắt safety này và giữ absolute path semantics. :chatgpt-content-reference{index="3"}

Vì vậy với fresher:

> **Không dùng `tar -P` trừ khi bạn hiểu rất rõ tại sao cần nó.**

Một archive chứa absolute paths rồi extract với unsafe settings có thể overwrite files ngoài intended restore directory.

---

# 14. Compression với tar

Một raw tar:

```text
backup.tar
```

có thể khá lớn.

GNU tar thường được kết hợp với compressor.

Gzip:

```bash
tar -czf backup.tar.gz source/
```

Ở đây:

```text
-z → gzip
```

Bzip2:

```bash
tar -cjf backup.tar.bz2 source/
```

```text
-j → bzip2
```

XZ:

```bash
tar -cJf backup.tar.xz source/
```

```text
-J → xz
```

Mental model:

```text
source files
     ↓
   tar
     ↓
archive stream
     ↓
compressor
     ↓
compressed archive
```

---

# 15. Compression là trade-off

Không phải:

```text
compression càng mạnh càng tốt
```

Bạn cần cân bằng:

```text
CPU
backup duration
restore duration
storage
network transfer
```

Ví dụ một archive configuration nhỏ:

```text
gzip
```

thường rất thực dụng.

Một multi-gigabyte backup có maintenance window nhỏ có thể cần quyết định khác.

---

# 16. Đặt timestamp vào backup name

Đừng mỗi lần:

```bash
tar -czf backup.tar.gz source/
```

nếu intention là historical backups.

Bạn sẽ overwrite file cũ.

Dùng:

```bash
timestamp=$(date '+%Y%m%d-%H%M%S')

tar -czf \
    "$HOME/day6-module3/backups/source-${timestamp}.tar.gz" \
    -C "$HOME/day6-module3" \
    source
```

Ví dụ:

```text
source-20261001-224500.tar.gz
```

Như vậy các recovery points không overwrite nhau.

---

# 17. Timestamp cần timezone context

Filename:

```text
source-20261001-224500.tar.gz
```

không cho biết timezone.

Trong controlled single-host lab thì okay.

Production có thể cần:

```text
UTC convention
```

hoặc metadata/log rõ timezone.

Ví dụ log:

```text
2026-10-01T22:45:00+0700
```

là ít ambiguous hơn.

---

# 18. `tar --exclude`

Giả sử source chứa cache:

```text
source/cache/
```

Bạn có thể:

```bash
tar -czf backup.tar.gz \
    --exclude='source/cache' \
    source/
```

Hoặc patterns như:

```bash
--exclude='*.tmp'
```

Nhưng hãy rất thận trọng.

Một fresher thường nghĩ:

> “Log lớn quá, exclude hết log.”

Nhưng nếu log cần cho incident investigation, bạn vừa xóa một phần recovery/evidence strategy.

Backup scope phải được thiết kế, không được chọn tùy tiện.

---

# 19. Backup scope

Một ứng dụng conceptual có thể gồm:

```text
/etc/myapp/
        │
        ├── configuration
        │
/opt/myapp/
        │
        ├── application binaries
        │
/var/lib/myapp/
        │
        ├── persistent application data
        │
/var/log/myapp/
        │
        └── logs
```

Không có quy tắc:

```text
"tar tất cả"
```

Bạn phải xác định:

| Data class      | Có backup?          | Recovery purpose              |
| --------------- | ------------------- | ----------------------------- |
| Config          | Thường có           | Reconstruct configuration     |
| Binary/artifact | Tùy release system  | Redeploy exact version        |
| Persistent data | Rất quan trọng      | Business/application recovery |
| Logs            | Tùy retention/audit | Troubleshooting/audit         |
| Temp/cache      | Thường không        | Re-creatable                  |

Phạm vi phải đến từ recovery requirement.

---

# 20. Metadata trong tar

Tar không chỉ lưu bytes.

Archive thường chứa metadata như:

```text
file name
mode/permissions
UID/GID information
mtime
file type
symlink information
```

GNU tar documentation cũng nhấn mạnh archive members chứa metadata như modification time, mode và ownership. :chatgpt-content-reference{index="4"}

Điều này rất quan trọng vì restore:

```text
content đúng
```

nhưng:

```text
permissions sai
```

vẫn có thể khiến application fail.

---

# 21. Restore ownership phụ thuộc privilege

Giả sử archive có file owner:

```text
appuser:appgroup
```

Bạn restore bằng normal user.

Normal user không được tùy ý chown thành arbitrary UID/GID.

Do đó:

```text
archive stores metadata
```

không đồng nghĩa:

```text
current restoring account can restore every metadata field
```

System recovery thường cần appropriate privilege.

Nhưng không được chạy restore bằng `root` chỉ để “cho chắc”.

Root restore tăng blast radius rất lớn.

---

# 22. `--same-permissions`

GNU tar có:

```text
-p
--same-permissions
--preserve-permissions
```

để restore recorded modes chính xác hơn; nếu không, umask có thể ảnh hưởng quyền cuối cùng tùy circumstances. :chatgpt-content-reference{index="5"}

Trong lab user data thường không cần dùng.

Trong system configuration recovery, permission handling phải được kiểm tra.

---

# 23. ACL, xattrs và SELinux — Supplementary nhưng rất quan trọng

Đây là nơi nhiều “backup bằng tar” thực tế thiếu metadata.

GNU tar hỗ trợ explicit options cho ACL và SELinux context như:

```text
--acls
--selinux
```

và hỗ trợ extended attributes tương ứng qua xattr options. Những metadata mở rộng này không nên được mặc định coi là được capture/restore chỉ vì bạn dùng basic `tar -czf`. :chatgpt-content-reference{index="6"}

Ví dụ trên SELinux host:

```text
file content restored
mode restored
owner restored
SELinux context wrong
```

service vẫn có thể bị:

```text
Permission denied
```

Đây là lý do recovery validation phải ở mức application/service, không chỉ `ls`.

---

# 24. Symlink

Giả sử:

```text
current -> releases/app-v2
```

Backup system phải biết nó đang backup:

```text
symlink object
```

hay dereference target contents.

Default behaviors và options phải được kiểm tra trước khi backup tree có symlinks đặc biệt.

Đừng giả định:

```text
"copy directory" = "mọi semantic đều giữ nguyên"
```

---

# 25. Special files

System trees có thể chứa:

```text
device nodes
sockets
FIFOs
```

Không phải tất cả đều phù hợp với generic file backup.

Ví dụ Unix socket:

```text
/run/myapp.sock
```

thường được process tạo lại khi service start.

Backup runtime socket thường vô nghĩa.

Backup scope phải hiểu data lifecycle.

---

# 26. Live consistency: một vấn đề cực kỳ lớn

Giả sử tar đang đọc:

```text
large-file.dat
```

application đồng thời ghi vào nó.

Timeline:

```text
tar reads first half
        │
application modifies file
        │
tar reads second half
```

Bạn có thể nhận một backup không đại diện cho bất kỳ single consistent point-in-time nào.

Đặc biệt nguy hiểm với database.

---

# 27. Cảnh báo quan trọng với PostgreSQL

Sau này Day 15 bạn học PostgreSQL backup/restore.

**Không coi:**

```bash
tar -czf postgres-backup.tar.gz /var/lib/postgresql/...
```

trong khi database đang hoạt động là một database-consistent backup strategy.

Tương tự:

```bash
rsync -a live-postgresql-data/ backup/
```

không tự biến thành consistent PostgreSQL backup.

Database cần database-aware backup, coordinated shutdown, filesystem snapshots với đúng procedure, hoặc PostgreSQL-supported mechanisms.

Đây là **Supplementary / Beyond explicit Day 6**, nhưng cực kỳ quan trọng để tránh áp dụng sai `tar/rsync` sau này.

---

# 28. `tar -t` có phải đủ verify backup không?

Không.

```bash
tar -tzf backup.tar.gz
```

chứng minh được những thứ như:

```text
archive có thể đọc/list tới mức command xử lý được
members có tồn tại trong archive listing
```

Nhưng chưa chứng minh:

```text
restore vào filesystem sẽ thành công hoàn toàn

restored application sẽ chạy

permissions đúng

all required files đúng

business data consistent
```

Do đó backup verification có nhiều tầng.

---

# 29. Các tầng verification

Mental model tốt:

```text
Level 1
archive exists
        ↓
Level 2
archive readable/listable
        ↓
Level 3
expected members present
        ↓
Level 4
restore into staging succeeds
        ↓
Level 5
content/metadata validation
        ↓
Level 6
application/recovery validation
```

Chỉ Level 1:

```bash
ls backup.tar.gz
```

là verification rất yếu.

---

# 30. GNU tar `--compare`

GNU tar có:

```bash
tar --compare ...
```

hay:

```bash
tar -d ...
```

để so archive members với filesystem equivalents và báo các differences về size, mode, owner, modification date và contents. GNU manual nhấn mạnh `--compare` nhằm xem archive phản ánh current filesystem như thế nào, không hoàn toàn đồng nghĩa media-integrity verification. :chatgpt-content-reference{index="7"}

Ví dụ:

```bash
tar -df backup.tar
```

Nhưng nếu source đã thay đổi hợp lệ sau backup, differences là expected.

Do đó interpretation cần timestamp/recovery-point awareness.

---

# 31. GNU tar `--verify`

GNU tar còn có:

```text
-W
--verify
```

kết hợp với create để attempt verification sau khi writing. Tuy nhiên GNU documentation mô tả nó đặc biệt trong context verifying data/media after writing và nó có constraints; `--verify` không thay thế restore test. :chatgpt-content-reference{index="8"}

Do đó cho fresher:

> **Restore-to-staging + validation là phương pháp dễ hiểu và thực dụng hơn để chứng minh recoverability.**

---

# 32. Hash verification

Một useful supplementary technique:

Trước:

```bash
sha256sum \
    "$HOME/day6-module3/source/config/app.properties"
```

Sau restore:

```bash
sha256sum \
    "$HOME/day6-module3/restore/tar-test/source/config/app.properties"
```

Nếu hashes giống:

```text
file content matches
```

Nhưng hash không chứng minh toàn bộ:

```text
owner
mode
ACL
xattrs
service functionality
```

Hash là một evidence layer, không phải toàn bộ recovery validation.

---

# 33. `rsync`: Mental model

Upstream mô tả rsync là utility cho fast incremental file transfer. Nó có thể chạy local hoặc remote. :chatgpt-content-reference{index="9"}

Basic local syntax:

```bash
rsync [options] SOURCE DESTINATION
```

Ví dụ:

```bash
rsync -a source/ backup/
```

Conceptual flow:

```text
scan source
    │
    ▼
compare source ↔ destination
    │
    ▼
determine required updates
    │
    ▼
transfer/update changed objects
```

---

# 34. `rsync -a`

Đây là option bạn sẽ thấy liên tục:

```bash
rsync -a source/ destination/
```

`-a` = `--archive`.

Theo official rsync man page:

```text
-a = -rlptgoD
```

Nó bật recursive transfer và bảo tồn gần như hầu hết metadata cơ bản, nhưng **không bao gồm** ACLs `-A`, xattrs `-X`, atimes `-U`, creation times `-N`, hay hardlink preservation `-H`. :chatgpt-content-reference{index="10"}

Đây là điểm quan trọng:

> `rsync -a` không có nghĩa “preserve absolutely everything.”

---

# 35. `-v`

```bash
rsync -av source/ destination/
```

`-v`:

```text
verbose
```

Hữu ích interactive/lab.

Production scheduled jobs đôi khi cần verbosity được cân bằng với log volume.

---

# 36. Trailing slash — phải hiểu tuyệt đối

Hai command:

```bash
rsync -a /src/foo /dest/
```

và:

```bash
rsync -a /src/foo/ /dest/
```

không có source semantics giống nhau.

Official rsync manual giải thích trailing slash trên source mang ý nghĩa gần như:

```text
copy CONTENTS of this directory
```

thay vì:

````text
copy this directory BY NAME
``` :chatgpt-content-reference{index="11"}


Đây là một trong những lỗi rsync phổ biến nhất.

---

# 37. Demonstration trailing slash

Lab:

```bash
mkdir -p "$HOME/day6-module3/rsync-demo/src"
printf 'A\n' > "$HOME/day6-module3/rsync-demo/src/a.txt"

mkdir -p "$HOME/day6-module3/rsync-demo/dest1"
mkdir -p "$HOME/day6-module3/rsync-demo/dest2"
````

Không slash cuối source:

```bash
rsync -av \
    "$HOME/day6-module3/rsync-demo/src" \
    "$HOME/day6-module3/rsync-demo/dest1/"
```

Inspect:

```bash
find "$HOME/day6-module3/rsync-demo/dest1" -print
```

Bạn có xu hướng thấy:

```text
dest1/
└── src/
    └── a.txt
```

Với slash:

```bash
rsync -av \
    "$HOME/day6-module3/rsync-demo/src/" \
    "$HOME/day6-module3/rsync-demo/dest2/"
```

Inspect:

```bash
find "$HOME/day6-module3/rsync-demo/dest2" -print
```

Bạn có xu hướng thấy:

```text
dest2/
└── a.txt
```

Không nên dùng rsync production trước khi trailing-slash behavior này đã trở thành phản xạ.

---

# 38. `--dry-run`

Đây là option safety quan trọng nhất của rsync:

```bash
rsync -anv source/ destination/
```

hoặc:

```bash
rsync -av --dry-run source/ destination/
```

Official man page nói `--dry-run` thực hiện trial run không thay đổi filesystem và cho output gần giống real run, thường được dùng với verbose hoặc itemized changes để xem trước những gì sẽ xảy ra. :chatgpt-content-reference{index="12"}

Mental model:

```text
desired command
       │
       ├─ first: --dry-run
       │
       ▼
review plan
       │
       ▼
real execution
```

---

# 39. `-i` / `--itemize-changes`

Rất hữu ích:

```bash
rsync -ain source/ destination/
```

`-i` cung cấp concise itemized representation của changes.

Trong lab:

```bash
rsync -ain \
    "$HOME/day6-module3/source/" \
    "$HOME/day6-module3/rsync-backup/"
```

Bạn xem trước:

```text
file nào sẽ tạo
file nào thay đổi
directory nào liên quan
```

Sau đó mới bỏ `-n`.

---

# 40. Initial rsync backup

Tạo destination:

```bash
mkdir -p "$HOME/day6-module3/rsync-backup"
```

Dry run:

```bash
rsync -ain \
    "$HOME/day6-module3/source/" \
    "$HOME/day6-module3/rsync-backup/"
```

Real:

```bash
rsync -ai \
    "$HOME/day6-module3/source/" \
    "$HOME/day6-module3/rsync-backup/"
```

Inspect:

```bash
find "$HOME/day6-module3/rsync-backup" -type f -print
```

---

# 41. Incremental update

Modify source:

```bash
printf 'feature.enabled=true\n' \
    >> "$HOME/day6-module3/source/config/app.properties"
```

Thêm:

```bash
printf 'record-003\n' \
    >> "$HOME/day6-module3/source/data/sample.txt"
```

Preview:

```bash
rsync -ain \
    "$HOME/day6-module3/source/" \
    "$HOME/day6-module3/rsync-backup/"
```

Sau đó:

```bash
rsync -ai \
    "$HOME/day6-module3/source/" \
    "$HOME/day6-module3/rsync-backup/"
```

Rsync sẽ tập trung vào differences thay vì blind-create một archive mới hoàn toàn.

---

# 42. Rsync default change detection

Một common misunderstanding là:

> “rsync luôn hash toàn bộ mọi file trước khi quyết định copy.”

Không phải mặc định theo cách đơn giản đó.

Rsync thường sử dụng quick-check based on metadata such as size và modification time để quyết định file có cần update; option:

```text
-c
--checksum
```

thay đổi skip decision để dùng checksum-based comparison. Official rsync man page mô tả `--checksum` là “skip based on checksum, not mod-time & size”. :chatgpt-content-reference{index="13"}

`-c` có cost:

```text
đọc toàn bộ file phía source
đọc toàn bộ candidate destination
CPU/I/O cao hơn
```

Đừng thêm `-c` vào mọi backup chỉ vì nghe “checksum an toàn hơn”.

---

# 43. `--checksum` không phải backup manifest

Command:

```bash
rsync -ac source/ destination/
```

không tạo cho bạn một independent immutable manifest như:

```bash
sha256sum files > manifest.sha256
```

Nó thay đổi comparison method của rsync.

Đừng nhầm:

```text
checksum comparison option
```

với:

```text
long-term cryptographic backup integrity record
```

---

# 44. `--delete`: rất mạnh và rất nguy hiểm

Giả sử destination có:

```text
destination/
    file1
    file2
    old-important-file
```

Source:

```text
source/
    file1
    file2
```

Command:

```bash
rsync -a --delete source/ destination/
```

có thể xóa:

```text
destination/old-important-file
```

vì nó không tồn tại phía source.

Official rsync documentation cảnh báo rõ `--delete` xóa extraneous files phía receiver và có thể nguy hiểm nếu dùng sai; tài liệu khuyến nghị chạy `--dry-run` trước để xem files nào sẽ bị xóa. :chatgpt-content-reference{index="14"}

---

# 45. Rule bắt buộc trước `--delete`

Trong bài học này, trước bất kỳ real `--delete` nào:

```text
1. confirm source
2. confirm destination
3. inspect both paths
4. confirm trailing slash semantics
5. dry-run
6. review every deletion
7. confirm rollback/recovery
8. only then execute
```

Trong production cần thêm approval/change-control phù hợp.

---

# 46. Lab an toàn cho `--delete`

Tất cả chỉ trong `$HOME/day6-module3/delete-lab`.

```bash
mkdir -p "$HOME/day6-module3/delete-lab/src"
mkdir -p "$HOME/day6-module3/delete-lab/dst"

printf 'keep\n' \
    > "$HOME/day6-module3/delete-lab/src/keep.txt"

printf 'keep\n' \
    > "$HOME/day6-module3/delete-lab/dst/keep.txt"

printf 'destination extra\n' \
    > "$HOME/day6-module3/delete-lab/dst/extra.txt"
```

Inspect:

```bash
find "$HOME/day6-module3/delete-lab" -type f -print
```

Dry-run only:

```bash
rsync -ain --delete \
    "$HOME/day6-module3/delete-lab/src/" \
    "$HOME/day6-module3/delete-lab/dst/"
```

Bạn phải identify:

```text
extra.txt
```

sẽ bị xóa.

Chỉ trong disposable lab này, sau khi đã hiểu output, mới được thử real command:

```bash
rsync -ai --delete \
    "$HOME/day6-module3/delete-lab/src/" \
    "$HOME/day6-module3/delete-lab/dst/"
```

Sau đó:

```bash
find "$HOME/day6-module3/delete-lab/dst" -type f -print
```

---

# 47. `--delete` làm mirror gần hơn, không làm backup tốt hơn

Một câu rất quan trọng:

```text
rsync --delete
```

không phải:

```text
"better backup mode"
```

Nó là:

```text
"make destination remove source-absent items within deletion scope"
```

Nếu bạn cần historical recoverability, deletion synchronization có thể làm **giảm** recovery value.

---

# 48. Exclude với rsync

Ví dụ:

```bash
rsync -a \
    --exclude='*.tmp' \
    --exclude='cache/' \
    source/ destination/
```

Nhưng giống tar:

```text
exclude
```

phải được thiết kế theo recovery requirement.

Đặc biệt khi kết hợp:

```text
--exclude
--delete
```

behavior cần review kỹ.

Rsync manual có riêng filter-rule và deletion semantics vì interactions này không đơn giản. :chatgpt-content-reference{index="15"}

---

# 49. Rsync và ACL/xattrs

Nhắc lại:

```text
-a
```

không include:

````text
-A ACLs
-X extended attributes
-H hardlink preservation
``` :chatgpt-content-reference{index="16"}


Nếu backup system files trên Linux có:

```text
POSIX ACL
SELinux labels/xattrs
capabilities
````

`rsync -a` alone có thể không đủ.

Đây là điểm rất quan trọng khi backup:

```text
/etc
application trees
security-sensitive files
```

---

# 50. Rsync local vs remote

Local:

```bash
rsync -a source/ destination/
```

Remote over SSH thường có form:

```bash
rsync -a source/ user@backup-host:/backup/path/
```

hoặc:

```bash
rsync -a user@source-host:/source/path/ local-destination/
```

Official rsync man page hỗ trợ remote shell usage và local-only transfers. :chatgpt-content-reference{index="17"}

### Supplementary security

Không hard-code:

```text
ssh password
private key
token
```

vào rsync script.

SSH key automation phải theo least privilege, dedicated account/path restrictions nếu production.

Day 3 SSH-key principles vẫn áp dụng.

---

# 51. Backup destination phải thực sự độc lập

Giả sử source:

```text
/home/alice/data
```

destination:

```text
/home/alice/backup
```

cùng một filesystem.

Nếu disk vật lý/filesystem fail:

```text
source mất
backup cũng mất
```

Lab local là để học command.

Production backup cần failure-domain awareness.

Ví dụ conceptual:

```text
different filesystem
different disk
different host
object storage
offsite copy
```

tùy requirement.

---

# 52. 3-2-1 awareness — Supplementary

Một backup guideline thường gặp là có multiple copies trên different media với một copy offsite.

Day 6 không yêu cầu bạn thiết kế enterprise backup architecture.

Điểm cần nhớ chỉ là:

```text
same-host backup
≠ protection against host loss
```

---

# 53. Backup destination capacity

Trước backup:

```bash
df -P "$BACKUP_DEST"
```

và:

```bash
du -sh "$SOURCE"
```

Không nên:

```text
start 100 GB backup
```

khi destination chỉ còn:

```text
5 GB
```

Nhưng `du` size và compressed result không hoàn toàn tương đương.

Đây chỉ là preflight estimate.

---

# 54. Capacity check phải là precheck, không phải guarantee

Giả sử source:

```text
10 GB
```

destination free:

```text
12 GB
```

Trong lúc backup:

```text
source grows
other process writes destination
compression ratio differs
```

backup vẫn có thể fail.

Do đó:

```text
precheck ≠ guarantee
```

Final command status và post-verification vẫn cần.

---

# 55. Backup permissions

Source có thể chứa unreadable files.

Nếu user không có quyền:

```text
backup có thể fail
```

hoặc tùy options/tool có warnings/partial result.

Không fix bằng:

```bash
chmod -R 777 source
```

Đó là cực kỳ sai.

Correct approach:

```text
identify required files
identify service account
grant approved least privilege
or run backup under approved privileged context
```

---

# 56. Cẩn thận `--ignore-failed-read`

GNU tar có option:

```text
--ignore-failed-read
```

cho phép một số missing/unreadable/changing files được coi nhẹ hơn, không ảnh hưởng exit status theo cách bình thường. GNU manual nói rõ option này có thể khiến such failures chỉ còn warnings. :chatgpt-content-reference{index="18"}

Với backup quan trọng, dùng option này bừa bãi rất nguy hiểm:

```text
backup có thể "success"
trong khi file quan trọng không được đọc
```

Fresher nên **không dùng** trừ khi backup design đã quyết định những failures đó acceptable.

---

# 57. Backup script phải xử lý failure rõ ràng

Bad:

```bash
tar -czf backup.tar.gz source/
echo "Backup completed"
```

Nếu tar fail:

```text
script vẫn nói completed
```

Từ Module 2, ta đã biết cần:

```bash
if tar -czf "$archive" ...; then
    ...
else
    ...
fi
```

---

# 58. Backup verification function

Ví dụ:

```bash
verify_tar_archive() {
    local archive="$1"

    if tar -tzf "$archive" >/dev/null 2>&1; then
        printf 'OK      backup-verify archive=%s\n' "$archive"
        return 0
    fi

    printf 'CRITICAL backup-verify archive=%s reason=archive-unreadable\n' \
        "$archive"

    return 1
}
```

Nhưng nhớ:

```text
tar -tzf passes
```

chỉ là một verification layer.

---

# 59. Restore test function — conceptual

Production restore tests thường được làm ở controlled environment.

Trong lab:

```bash
restore_test() {
    local archive="$1"
    local target="$2"

    mkdir -p "$target" || return 1

    tar -xzf "$archive" -C "$target"
}
```

Sau đó validate expected file:

```bash
[[ -f "$target/source/config/app.properties" ]]
```

Và content:

```bash
grep -Fxq 'server.port=8080' \
    "$target/source/config/app.properties"
```

Đây là much stronger evidence.

---

# 60. Nhưng restore test không nên overwrite existing test directory

Nếu:

```text
restore-test/
```

đã có leftovers từ lần trước, test có thể false-positive.

Ví dụ archive thiếu file nhưng file cũ vẫn tồn tại.

Do đó restore testing nên dùng **empty staging directory**.

Lab:

```bash
restore_dir=$(mktemp -d "$HOME/day6-module3/restore/test.XXXXXX")
```

Sau test cleanup cẩn thận.

Đây là **Supplementary**, nhưng rất quan trọng.

---

# 61. Hash manifest

Lab:

```bash
cd "$HOME/day6-module3/source"

find . -type f -print0 |
sort -z |
xargs -0 sha256sum \
    > "$HOME/day6-module3/source.sha256"
```

Đây là advanced-ish because filenames/null delimiters matter.

Bạn chưa cần thuộc command.

Ý tưởng quan trọng:

```text
known-good manifest
        ↓
restore
        ↓
recalculate
        ↓
compare
```

Nhưng metadata vẫn phải được kiểm tra riêng.

---

# 62. Tại sao không dùng đơn giản `md5sum`?

MD5 vẫn có thể phát hiện accidental corruption trong một số contexts, nhưng cho modern integrity evidence thường nên dùng stronger hash như SHA-256.

Không dùng file hash như authentication/signature trừ khi threat model và keying/signature strategy được thiết kế phù hợp.

---

# 63. Thiết kế Day 6 integrated script

Bây giờ ta ghép Module 2 và Module 3.

Script của chúng ta sẽ:

```text
validate environment
      ↓
health checks
      │
      ├── service
      ├── disk
      └── port
      ↓
backup prechecks
      ↓
create tar backup
      ↓
list/read verification
      ↓
rsync backup copy
      ↓
summary
      ↓
meaningful exit code
```

Trong production có thể bạn không muốn health check và backup chạy cùng script.

Nhưng Day 6 Assignment yêu cầu build a **health-check and backup script**, nên integrated lab là hợp lý.

---

# 64. Lab directory layout

Dùng chỉ trong `$HOME`:

```text
$HOME/day6-capstone/
│
├── source/
│   ├── config/
│   └── data/
│
├── archive-backups/
│
├── rsync-backup/
│
├── restore-tests/
│
├── logs/
│
└── bin/
    └── day6-ops.sh
```

Tạo:

```bash
mkdir -p "$HOME/day6-capstone"/{source/config,source/data}
mkdir -p "$HOME/day6-capstone"/{archive-backups,rsync-backup}
mkdir -p "$HOME/day6-capstone"/{restore-tests,logs,bin}
```

---

# 65. Tạo sample workload

```bash
printf 'server.port=8080\n' \
  > "$HOME/day6-capstone/source/config/app.properties"

printf 'environment=lab\n' \
  > "$HOME/day6-capstone/source/config/environment.conf"

printf 'record-001\nrecord-002\n' \
  > "$HOME/day6-capstone/source/data/application-data.txt"
```

Permissions:

```bash
chmod 640 "$HOME/day6-capstone/source/config/"*
chmod 640 "$HOME/day6-capstone/source/data/"*
```

Inspect:

```bash
find "$HOME/day6-capstone/source" \
    -type f \
    -exec ls -l {} \;
```

---

# 66. Integrated script

Dưới đây là script học tập. Nó chỉ thao tác trong lab directory của user và **không tự restart service, không xóa production data, không chạy `rsync --delete`**.

```bash
#!/bin/bash

set -u
set -o pipefail

BASE_DIR="${BASE_DIR:-$HOME/day6-capstone}"
SOURCE_DIR="${SOURCE_DIR:-$BASE_DIR/source}"
ARCHIVE_DIR="${ARCHIVE_DIR:-$BASE_DIR/archive-backups}"
RSYNC_DIR="${RSYNC_DIR:-$BASE_DIR/rsync-backup}"
LOG_DIR="${LOG_DIR:-$BASE_DIR/logs}"

SERVICE_NAME="${SERVICE_NAME:-ssh.service}"
MOUNT_POINT="${MOUNT_POINT:-/}"
DISK_THRESHOLD="${DISK_THRESHOLD:-80}"
PORT="${PORT:-22}"

overall_rc=0

timestamp() {
    date '+%Y-%m-%dT%H:%M:%S%z'
}

log() {
    local level="$1"
    shift

    printf '%s %-8s %s\n' \
        "$(timestamp)" \
        "$level" \
        "$*"
}

record_rc() {
    local rc="$1"

    if [[ "$rc" -gt "$overall_rc" ]]; then
        overall_rc="$rc"
    fi
}

require_command() {
    local name="$1"

    if command -v "$name" >/dev/null 2>&1; then
        return 0
    fi

    log ERROR "missing-command=$name"
    return 2
}

validate_environment() {
    local rc=0

    for cmd in \
        date hostname systemctl df awk grep ss tar rsync
    do
        require_command "$cmd" || rc=2
    done

    if [[ ! -d "$SOURCE_DIR" ]]; then
        log ERROR "source-directory-missing path=$SOURCE_DIR"
        rc=2
    fi

    if [[ ! "$DISK_THRESHOLD" =~ ^[0-9]+$ ]]; then
        log ERROR "invalid-disk-threshold value=$DISK_THRESHOLD"
        rc=2
    elif (( DISK_THRESHOLD < 1 || DISK_THRESHOLD > 100 )); then
        log ERROR "disk-threshold-out-of-range value=$DISK_THRESHOLD"
        rc=2
    fi

    return "$rc"
}

check_service() {
    if systemctl is-active --quiet "$SERVICE_NAME"; then
        log OK "check=service service=$SERVICE_NAME state=active"
        return 0
    fi

    log CRITICAL \
        "check=service service=$SERVICE_NAME state=not-active"

    return 1
}

check_disk() {
    local df_output
    local usage

    if ! df_output=$(df -P "$MOUNT_POINT" 2>&1); then
        log UNKNOWN \
            "check=disk mount=$MOUNT_POINT reason=df-failed"
        return 2
    fi

    usage=$(
        printf '%s\n' "$df_output" |
        awk 'NR==2 {
            gsub(/%/, "", $5)
            print $5
        }'
    )

    if [[ ! "$usage" =~ ^[0-9]+$ ]]; then
        log UNKNOWN \
            "check=disk mount=$MOUNT_POINT reason=invalid-usage"
        return 2
    fi

    if (( usage >= DISK_THRESHOLD )); then
        log CRITICAL \
            "check=disk mount=$MOUNT_POINT usage=${usage}% threshold=${DISK_THRESHOLD}%"
        return 1
    fi

    log OK \
        "check=disk mount=$MOUNT_POINT usage=${usage}% threshold=${DISK_THRESHOLD}%"

    return 0
}

check_port() {
    local ss_output

    if ! ss_output=$(ss -lnt 2>&1); then
        log UNKNOWN \
            "check=port port=$PORT reason=ss-failed"
        return 2
    fi

    if printf '%s\n' "$ss_output" |
       awk '{print $4}' |
       grep -Eq ":${PORT}$"
    then
        log OK \
            "check=port port=$PORT state=listening"
        return 0
    fi

    log CRITICAL \
        "check=port port=$PORT state=not-listening"

    return 1
}

create_archive_backup() {
    local backup_ts
    local archive

    backup_ts=$(date '+%Y%m%d-%H%M%S')
    archive="$ARCHIVE_DIR/source-${backup_ts}.tar.gz"

    if ! mkdir -p "$ARCHIVE_DIR"; then
        log ERROR \
            "backup=tar reason=archive-directory-create-failed path=$ARCHIVE_DIR"
        return 2
    fi

    log INFO \
        "backup=tar event=start source=$SOURCE_DIR archive=$archive"

    if ! tar -czf "$archive" \
        -C "$(dirname "$SOURCE_DIR")" \
        "$(basename "$SOURCE_DIR")"
    then
        log CRITICAL \
            "backup=tar event=create-failed archive=$archive"
        return 1
    fi

    if [[ ! -s "$archive" ]]; then
        log CRITICAL \
            "backup=tar event=invalid-output archive=$archive"
        return 1
    fi

    if ! tar -tzf "$archive" >/dev/null 2>&1; then
        log CRITICAL \
            "backup=tar event=verify-failed archive=$archive"
        return 1
    fi

    log OK \
        "backup=tar event=verified archive=$archive"

    return 0
}

run_rsync_backup() {
    if ! mkdir -p "$RSYNC_DIR"; then
        log ERROR \
            "backup=rsync reason=destination-create-failed path=$RSYNC_DIR"
        return 2
    fi

    log INFO \
        "backup=rsync event=start source=$SOURCE_DIR destination=$RSYNC_DIR"

    if rsync -a \
        "$SOURCE_DIR/" \
        "$RSYNC_DIR/"
    then
        log OK \
            "backup=rsync event=completed destination=$RSYNC_DIR"
        return 0
    fi

    log CRITICAL \
        "backup=rsync event=failed destination=$RSYNC_DIR"

    return 1
}

main() {
    local rc

    mkdir -p "$LOG_DIR" 2>/dev/null || true

    log INFO "event=day6-ops-start host=$(hostname)"

    validate_environment
    rc=$?
    record_rc "$rc"

    if [[ "$rc" -eq 2 ]]; then
        log ERROR \
            "event=precheck-failed action=stop-before-backup"
        return "$overall_rc"
    fi

    check_service
    rc=$?
    record_rc "$rc"

    check_disk
    rc=$?
    record_rc "$rc"

    check_port
    rc=$?
    record_rc "$rc"

    create_archive_backup
    rc=$?
    record_rc "$rc"

    run_rsync_backup
    rc=$?
    record_rc "$rc"

    log INFO \
        "event=day6-ops-end exit_code=$overall_rc"

    return "$overall_rc"
}

main "$@"
exit $?
```

---

# 67. Một design decision quan trọng trong script

Bạn thấy:

```text
health check CRITICAL
```

không tự động chặn backup.

Tại sao?

Ví dụ SSH service check fail nhưng source data vẫn readable.

Bạn có thể vẫn muốn capture backup/evidence.

Ngược lại:

```text
validate_environment = internal/precheck failure
```

có thể khiến script stop trước backup.

Đây là deliberate design.

Trong production, requirements có thể khác.

---

# 68. Script không dùng `set -e`

Có chủ ý.

Nếu service check trả 1:

```text
service unhealthy
```

ta vẫn muốn:

```text
check disk
check port
backup
collect evidence
```

Dùng naïve `set -e` có thể khiến flow khó reasoning.

Ta dùng explicit exit handling.

---

# 69. Syntax validation

Lưu:

```text
$HOME/day6-capstone/bin/day6-ops.sh
```

Chạy:

```bash
bash -n "$HOME/day6-capstone/bin/day6-ops.sh"
```

Expected:

```text
no syntax error
rc=0
```

Kiểm tra:

```bash
printf 'syntax_rc=%d\n' "$?"
```

---

# 70. Permissions

```bash
chmod 750 "$HOME/day6-capstone/bin/day6-ops.sh"
```

Inspect:

```bash
ls -l "$HOME/day6-capstone/bin/day6-ops.sh"
```

Không:

```bash
chmod 777 ...
```

---

# 71. Xác định service/port đúng

Trước khi chạy:

```bash
systemctl list-units --type=service --state=running
```

và:

```bash
ss -lnt
```

Nếu máy là RHEL-family:

```text
sshd.service
```

có thể phù hợp hơn:

```text
ssh.service
```

Override:

```bash
SERVICE_NAME=sshd.service \
PORT=22 \
"$HOME/day6-capstone/bin/day6-ops.sh"
```

---

# 72. Manual execution

```bash
"$HOME/day6-capstone/bin/day6-ops.sh" \
    > "$HOME/day6-capstone/logs/manual-run.log" \
    2>&1

rc=$?

printf 'day6_script_rc=%d\n' "$rc"
```

Inspect:

```bash
cat "$HOME/day6-capstone/logs/manual-run.log"
```

---

# 73. Verify tar backup

```bash
ls -lh "$HOME/day6-capstone/archive-backups"
```

Get newest archive:

```bash
latest_archive=$(
    find "$HOME/day6-capstone/archive-backups" \
        -maxdepth 1 \
        -type f \
        -name 'source-*.tar.gz' \
        -printf '%T@ %p\n' |
    sort -n |
    tail -n 1 |
    cut -d' ' -f2-
)
```

Đây hơi advanced; trong lab bạn cũng có thể copy exact filename từ `ls`.

Inspect:

```bash
tar -tzf "$latest_archive"
```

---

# 74. Restore-test archive

Tạo clean directory:

```bash
restore_dir="$HOME/day6-capstone/restore-tests/manual-restore"

rm -rf -- "$restore_dir"
mkdir -p "$restore_dir"
```

**Cảnh báo:** `rm -rf` là destructive.

Trước khi chạy, xác nhận:

```bash
printf '%s\n' "$restore_dir"
```

và chắc chắn nó nằm trong disposable lab path.

Sau đó:

```bash
tar -xzf "$latest_archive" -C "$restore_dir"
```

Inspect:

```bash
find "$restore_dir" -type f -print
```

---

# 75. Validate restored files

```bash
cat \
  "$restore_dir/source/config/app.properties"
```

Expected invariant:

```text
server.port=8080
```

Check:

```bash
grep -Fxq \
    'server.port=8080' \
    "$restore_dir/source/config/app.properties"

printf 'content_validation_rc=%d\n' "$?"
```

Expected:

```text
0
```

---

# 76. Compare directories

Trong lab:

```bash
diff -r \
    "$HOME/day6-capstone/source" \
    "$restore_dir/source"
```

No output thường có nghĩa file contents/tree comparable theo `diff -r` scope.

Nhưng `diff -r` không phải metadata-complete validation.

Vẫn phải xem ownership/modes nếu requirement cần.

---

# 77. Verify rsync destination

```bash
find "$HOME/day6-capstone/rsync-backup" \
    -type f \
    -print
```

Compare:

```bash
diff -r \
    "$HOME/day6-capstone/source" \
    "$HOME/day6-capstone/rsync-backup"
```

Again:

```text
content comparison
```

không bằng full security-metadata validation.

---

# 78. Schedule integrated job

Sau khi manual run thành công:

```bash
crontab -l > "$HOME/day6-capstone/crontab.before" 2>/dev/null || true
```

Trong lab, có thể tạm schedule mỗi 5 phút:

```cron
*/5 * * * * /bin/bash /home/YOURUSER/day6-capstone/bin/day6-ops.sh >> /home/YOURUSER/day6-capstone/logs/cron.log 2>&1
```

Không copy `YOURUSER`.

Lấy actual path:

```bash
realpath "$HOME/day6-capstone/bin/day6-ops.sh"
```

Và home:

```bash
printf '%s\n' "$HOME"
```

---

# 79. Verify schedule

```bash
crontab -l
```

Sau execution:

```bash
tail -n 50 "$HOME/day6-capstone/logs/cron.log"
```

Check backup:

```bash
ls -lt "$HOME/day6-capstone/archive-backups" | head
```

Cron evidence cần chứng minh:

```text
entry installed
        ↓
scheduler triggered
        ↓
script started
        ↓
health checks executed
        ↓
backup created
        ↓
verification passed
```

Không chỉ:

```text
crontab -l có dòng
```

---

# 80. Một vấn đề mới: mỗi cron run tạo một archive

Với:

```cron
*/5 * * * *
```

bạn tạo:

```text
12 backups/hour
288 backups/day
```

Nếu không có retention:

```text
backup disk eventually fills
```

Đây là lý do test schedule phải được cleanup hoặc đổi sau lab.

---

# 81. Retention

Một production policy có thể nói:

```text
daily backups retained 7 days
weekly retained 4 weeks
...
```

Nhưng Day 6 syllabus không quy định retention algorithm cụ thể.

Do đó tôi **không** muốn bạn copy một dangerous:

```bash
find ... -delete
```

routine mà chưa hiểu.

Automatic backup deletion là destructive operation.

Trong Module 3, chỉ cần hiểu:

```text
historical backups consume disk
therefore retention must be intentionally designed
```

---

# 82. Không tự động thêm `find -mtime ... -delete`

Command dạng:

```bash
find "$BACKUP_DIR" -type f -mtime +7 -delete
```

có thể hợp lệ trong một carefully controlled design.

Nhưng một typo ở:

```text
BACKUP_DIR
```

có thể gây data loss.

Production implementation phải có:

```text
validated absolute path
non-empty path check
scope restriction
dry-run/listing
retention approval
recovery understanding
```

Chưa cần implement trong Day 6 fresher lab.

---

# 83. Failure Injection 1 — source directory missing

An toàn:

```bash
SOURCE_DIR="$HOME/day6-capstone/no-such-source" \
"$HOME/day6-capstone/bin/day6-ops.sh" \
  > "$HOME/day6-capstone/logs/failure-missing-source.log" \
  2>&1

rc=$?

printf 'rc=%d\n' "$rc"
```

Expected:

```text
precheck failure
no backup attempted
non-zero exit
```

Inspect:

```bash
cat "$HOME/day6-capstone/logs/failure-missing-source.log"
```

---

# 84. Failure Injection 2 — invalid backup directory

An toàn hơn phá real filesystem.

Ví dụ set archive destination dưới regular file.

```bash
printf 'not-a-directory\n' \
    > "$HOME/day6-capstone/not-a-directory"
```

Run:

```bash
ARCHIVE_DIR="$HOME/day6-capstone/not-a-directory/backup" \
"$HOME/day6-capstone/bin/day6-ops.sh" \
  > "$HOME/day6-capstone/logs/failure-archive-dest.log" \
  2>&1

rc=$?

printf 'rc=%d\n' "$rc"
```

Expected:

```text
mkdir/create failure
backup reported failed
non-zero overall status
```

No production data harmed.

---

# 85. Failure Injection 3 — rsync destination permission

Tạo disposable destination:

```bash
mkdir -p "$HOME/day6-capstone/blocked-rsync"
chmod 500 "$HOME/day6-capstone/blocked-rsync"
```

Run:

```bash
RSYNC_DIR="$HOME/day6-capstone/blocked-rsync" \
"$HOME/day6-capstone/bin/day6-ops.sh" \
  > "$HOME/day6-capstone/logs/failure-rsync-permission.log" \
  2>&1

rc=$?
printf 'rc=%d\n' "$rc"
```

Expected trên normal user run:

```text
rsync cannot create/update contents
non-zero status
```

Sau lab restore permission:

```bash
chmod 700 "$HOME/day6-capstone/blocked-rsync"
```

Không làm lab permission manipulation trên real backup destination.

---

# 86. Failure Injection 4 — corrupted archive copy

Đây là failure rất hay vì không đụng original archive.

Copy:

```bash
cp -- \
  "$latest_archive" \
  "$HOME/day6-capstone/corrupt-test.tar.gz"
```

Corrupt disposable copy:

```bash
truncate -s 100 \
  "$HOME/day6-capstone/corrupt-test.tar.gz"
```

Test:

```bash
tar -tzf \
  "$HOME/day6-capstone/corrupt-test.tar.gz"

rc=$?
printf 'verify_rc=%d\n' "$rc"
```

Expected:

```text
non-zero
```

Bạn vừa chứng minh:

```text
verification detects obviously broken archive
```

Cleanup:

```bash
rm -- "$HOME/day6-capstone/corrupt-test.tar.gz"
```

---

# 87. Failure Injection 5 — health service failure without stopping service

Như Module 2:

```bash
SERVICE_NAME=definitely-not-real.service \
"$HOME/day6-capstone/bin/day6-ops.sh" \
  > "$HOME/day6-capstone/logs/failure-service.log" \
  2>&1

rc=$?
printf 'rc=%d\n' "$rc"
```

Expected:

```text
service check CRITICAL
backup may still proceed
overall rc non-zero
```

Đây là excellent test của aggregation logic.

---

# 88. Failure Injection 6 — non-listening port

Sau khi verify:

```bash
ss -lnt
```

chọn một port không listen, ví dụ:

```bash
PORT=65432 \
"$HOME/day6-capstone/bin/day6-ops.sh" \
  > "$HOME/day6-capstone/logs/failure-port.log" \
  2>&1
```

Expected:

```text
port CRITICAL
```

Không đóng production port để tạo failure.

---

# 89. Failure injection không phải chaos tùy tiện

Một proper failure test có:

| Element           | Example                           |
| ----------------- | --------------------------------- |
| Failure           | Invalid backup destination        |
| Expected symptom  | Backup function fails             |
| Expected exit     | Non-zero                          |
| Expected evidence | `archive-directory-create-failed` |
| Blast radius      | User lab directory only           |
| Recovery          | Restore valid path                |
| Validation        | Next backup succeeds              |

Nếu không biết rollback trước khi inject failure:

```text
chưa nên inject.
```

---

# 90. Troubleshooting backup: symptom → layer

Giả sử:

```text
cron log says backup failed
```

Đừng ngay lập tức rerun as root.

Decompose:

```text
Scheduler
   ↓
Script started?
   ↓
Dependencies available?
   ↓
Source exists?
   ↓
Source readable?
   ↓
Destination exists?
   ↓
Destination writable?
   ↓
Capacity available?
   ↓
tar/rsync executed?
   ↓
Exit status?
   ↓
Archive/output exists?
   ↓
Verification?
```

---

# 91. Scenario: `tar: Cannot open: Permission denied`

Potential layers:

```text
archive directory permission
parent directory permission
filesystem read-only
SELinux/AppArmor
wrong executing user
```

Evidence:

```bash
id
```

```bash
ls -ld "$ARCHIVE_DIR"
```

```bash
namei -l "$ARCHIVE_DIR" 2>/dev/null
```

```bash
mount | grep ...
```

nếu filesystem state relevant.

Không fix bằng:

```bash
chmod -R 777
```

---

# 92. Scenario: source file permission denied

Need distinguish:

```text
backup writer cannot write destination
```

from:

```text
backup reader cannot read source
```

Evidence:

```bash
ls -l source-file
```

```bash
id
```

```bash
getfacl source-file
```

nếu `getfacl` installed và ACL suspected.

Potential escalation nếu permissions owned by another team/application.

---

# 93. Scenario: `No space left on device`

Layer:

```text
destination filesystem capacity
```

Evidence:

```bash
df -P "$ARCHIVE_DIR"
```

Inode capacity:

```bash
df -Pi "$ARCHIVE_DIR"
```

Một filesystem có thể còn bytes nhưng hết inodes.

Day 5 capacity knowledge quay trở lại đây.

---

# 94. Disk full: không được vội xóa backups

Một L1 operator thấy 100% disk và chạy:

```bash
rm -rf old-backups
```

có thể xóa recovery points chịu retention/legal requirement.

Correct sequence:

```text
confirm filesystem
        ↓
identify top consumers
        ↓
understand retention/runbook
        ↓
safe remediation within authority
        ↓
or escalate
```

Backup storage cleanup là state-changing/destructive.

---

# 95. Scenario: rsync returns non-zero

Đừng chỉ nhìn:

```text
rsync failed
```

Capture:

```text
stderr
exit code
source/destination
time
user
rsync version
```

Rsync có command-specific exit codes.

Bạn không cần thuộc hết trong Day 6; dùng:

```bash
man rsync
```

và xem `EXIT VALUES`.

Điều quan trọng là:

```text
do not flatten every non-zero into same root cause.
```

---

# 96. Scenario: rsync copied files to “wrong directory”

First suspect:

```text
trailing slash
```

Compare:

```bash
rsync -a src dest/
```

vs:

```bash
rsync -a src/ dest/
```

Đừng “fix” bằng moving files trước khi hiểu original transfer semantics.

Official rsync docs xác nhận rõ distinction này. :chatgpt-content-reference{index="19"}

---

# 97. Scenario: destination files disappeared

Check immediately:

```text
Was --delete used?
What source path?
Was source unexpectedly empty?
Was wrong source mounted?
Was trailing slash/path wrong?
Were excludes involved?
```

Preserve:

```text
command
cron entry
logs
rsync output
timeline
```

Nếu production data loss suspected:

```text
stop additional destructive synchronization if authorized
preserve evidence
escalate immediately
```

Không repeatedly rerun `rsync --delete`.

---

# 98. Important `--delete` protection behavior

Official rsync documents an important safeguard: if sender detects I/O errors, deletion is automatically disabled by default to help prevent temporary source filesystem failures from causing massive destination deletions; this can be overridden by `--ignore-errors`. :chatgpt-content-reference{index="20"}

Điều này dẫn đến một strong rule:

> Đừng thêm `--ignore-errors` vào backup command chỉ để “cho job chạy hết”.

Bạn có thể vô hiệu hóa một safety behavior quan trọng.

---

# 99. Scenario: archive exists nhưng restore fails

Possible causes:

```text
archive truncated
compression corruption
wrong archive format
wrong command/options
storage corruption
permissions
insufficient disk
unsafe/untrusted members
```

Evidence:

```bash
file backup.tar.gz
```

```bash
ls -lh backup.tar.gz
```

```bash
tar -tzf backup.tar.gz
```

```bash
sha256sum backup.tar.gz
```

nếu known-good checksum exists.

Không restore trực tiếp vào production để test.

---

# 100. Scenario: restore succeeded nhưng application fails

This is where filesystem-level verification ends.

Check:

```text
owner
group
permissions
ACL
xattrs/SELinux
config paths
environment variables
service account
ports
dependencies
database
```

Integrated troubleshooting:

```text
files
 ↓
permissions
 ↓
service
 ↓
port
 ↓
logs
 ↓
network/database
```

Day 2–6 knowledge converges here.

---

# 101. Security — backups are highly sensitive

Backup có thể chứa:

```text
password files
application configs
private business data
SSH host/user configuration
database credentials
API keys
certificates
customer data
internal architecture
logs
```

Vì vậy:

```text
backup read access
```

có thể tương đương với:

```text
read access to original systems/data
```

Permissions và storage location phải được bảo vệ.

---

# 102. Backup directory permission

Lab:

```bash
chmod 700 "$HOME/day6-capstone/archive-backups"
```

Nếu group sharing required:

```text
design group ownership + mode intentionally
```

không:

```bash
chmod 777 backups
```

---

# 103. Backup filenames có thể leak information

Ví dụ:

```text
customer-acme-prod-db-password-backup.tar.gz
```

filename đã leak context.

Use neutral operational naming appropriate to policy.

---

# 104. Encryption awareness — Supplementary

`tar.gz`:

```text
compressed
```

không:

```text
encrypted
```

Rất quan trọng.

Gzip compression không bảo vệ confidentiality.

Nếu backup contains secrets, storage-layer or backup-layer encryption strategy có thể cần thiết.

Day 6 không yêu cầu implement encryption, nhưng phải biết:

```text
compression ≠ encryption
```

---

# 105. Checksum ≠ encryption

Tương tự:

```text
SHA-256
```

giúp integrity comparison.

Không hide data.

```text
hash ≠ encryption
```

---

# 106. Restore untrusted archive

Không extract untrusted tarball bằng root trực tiếp vào `/`.

GNU tar có protections như stripping leading `/` và rejecting unsafe `..` behavior mặc định, nhưng extraction vẫn nên được coi là filesystem mutation và untrusted archive có thể có tricky paths/symlinks/content. :chatgpt-content-reference{index="21"}

Safe pattern:

```text
inspect
↓
non-root staging directory
↓
list members
↓
extract staging
↓
review
↓
only then controlled restore
```

---

# 107. Operational principle: backup and restore are separate procedures

Backup runbook:

```text
source
precheck
backup
verify
retention
evidence
```

Restore runbook:

```text
incident approval
choose recovery point
prepare target
restore
metadata validation
application validation
business validation
handover
```

Nếu organization chỉ có backup procedure mà không có restore procedure:

```text
recovery readiness is incomplete.
```

---

# 108. Rollback của backup script deployment

Trước sửa script:

```bash
cp -a \
  "$HOME/day6-capstone/bin/day6-ops.sh" \
  "$HOME/day6-capstone/bin/day6-ops.sh.bak"
```

Sau edit:

```bash
diff -u \
  "$HOME/day6-capstone/bin/day6-ops.sh.bak" \
  "$HOME/day6-capstone/bin/day6-ops.sh"
```

Syntax:

```bash
bash -n \
  "$HOME/day6-capstone/bin/day6-ops.sh"
```

Nếu deployment regression:

```text
restore previous known-good script
```

và validate.

---

# 109. Cron change evidence

Before:

```bash
crontab -l \
  > "$HOME/day6-capstone/crontab.before" \
  2>/dev/null || true
```

After:

```bash
crontab -l \
  > "$HOME/day6-capstone/crontab.after"
```

Compare:

```bash
diff -u \
  "$HOME/day6-capstone/crontab.before" \
  "$HOME/day6-capstone/crontab.after"
```

Đây là chuẩn bị cho Day 18 change/release controls.

---

# 110. Escalation evidence — mandatory Day 6

Syllabus yêu cầu bạn:

```text
document escalation evidence
```

đây không phải optional. :chatgpt-content-reference{index="22"}

Giả sử backup job fail và investigation cho thấy destination là remote backup storage do storage team quản lý.

Bạn không có quyền sửa.

Một escalation tốt phải đủ để người nhận tiếp tục investigation mà không hỏi lại từ đầu.

---

# 111. Escalation evidence model

Một record tốt có dạng conceptual:

```text
Incident time:
2026-10-01T23:10:05+0700

Host:
lab-app01

Job:
day6-ops.sh

Expected:
Archive + rsync backup complete successfully.

Actual:
Tar archive created and verified.
Rsync copy failed.

Impact:
Current local archive exists.
Remote/current rsync copy not updated.

Last known success:
2026-10-01T23:05:01+0700

Exit code:
1

Relevant message:
rsync: ... Permission denied ...

Evidence:
id
ls -ld destination
df -P destination
rsync --version
script log
cron entry
scheduler log

Safe tests performed:
Confirmed source readable.
Confirmed local archive succeeds.
Confirmed destination cannot be written by job user.

Changes performed:
None to destination permissions.

Why escalation:
Destination access owned by Storage/Backup team; permission change outside L1 scope.

Current state:
Application remains healthy.
Local backup recovery point exists.
Remote synchronization remains failed.
```

Đây mới là escalation.

Không phải:

```text
"Backup lỗi, nhờ check."
```

---

# 112. Evidence phải phân biệt facts và hypotheses

Bad:

```text
Storage team broke permissions.
```

Nếu chưa chứng minh.

Better:

```text
Observed:
backup user receives Permission denied writing /backup/path.

Current ownership:
root:backupadmins

Current job user:
appops

Hypothesis:
destination access may have changed or required access is missing.

No permission modification performed by L1.
```

Đây là evidence-driven communication.

---

# 113. Không đưa secrets vào evidence

Redact:

```text
password
tokens
private keys
connection strings
Authorization headers
```

Nhưng redaction phải đủ để retain diagnostic value.

Ví dụ:

```text
postgresql://appuser:[REDACTED]@db01:5432/appdb
```

tốt hơn xóa cả endpoint nếu endpoint cần troubleshooting.

---

# 114. Linux Consolidation — Day 2 đến Day 6

Day 6 không hoàn chỉnh nếu bạn chỉ biết backup.

Hãy nhìn full stack Linux đã học:

```text
                     USER / OPERATOR
                           │
                           ▼
                     shell / Bash
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
        files          processes          network
     permissions       systemd             DNS
      ownership        services            ports
          │                │                │
          └────────────┬───┴────────────────┘
                       ▼
                filesystem/storage
                       │
                       ▼
                  application
                       │
                       ▼
                      logs
                       │
             journalctl/grep/awk/sed
                       │
                       ▼
                troubleshooting
                       │
                       ▼
                 Bash automation
                       │
                       ▼
                  exit status
                       │
                       ▼
                     cron
                       │
                       ▼
                  tar / rsync
                       │
                       ▼
                verify / restore
                       │
                       ▼
                 evidence / escalation
```

Đó là Linux Fundamentals as an operational system.

---

# 115. Integrated troubleshooting chain

Giả sử monitoring sau này báo:

```text
Application unavailable
```

L1 chain:

```text
1. client symptom
        ↓
2. DNS?
        ↓
3. IP/network?
        ↓
4. port listening?
        ↓
5. systemd service?
        ↓
6. process?
        ↓
7. permissions?
        ↓
8. disk capacity?
        ↓
9. logs?
        ↓
10. recent scheduled job?
        ↓
11. backup filled disk?
        ↓
12. restore/rollback needed?
        ↓
13. L1 scope?
        ↓
14. escalate with evidence?
```

Notice:

Backup itself can become a cause of incident.

Example:

```text
cron backup runs every minute
        ↓
hundreds of archives
        ↓
filesystem 100%
        ↓
application cannot write
        ↓
application fails
```

Everything is connected.

---

# 116. Failure chain example

Incident:

```text
02:00 backup scheduled
02:00 backup starts
02:04 backup destination fills /
02:05 application cannot write temp file
02:05 service logs No space left on device
02:06 app health check fails
```

A fresher may restart application.

Result:

```text
still fails
```

Because root cause is:

```text
disk full
```

Correct evidence path:

```bash
df -P
```

then:

```bash
du
```

appropriate scope,

then cron/backup evidence:

```bash
journalctl
crontab -l
```

and logs.

---

# 117. Another integrated scenario

Symptom:

```text
Backup archive suddenly much smaller.
```

Potential hypotheses:

```text
source missing files
wrong source path
permissions changed
exclude changed
mount missing
application data moved
archive failed partially
```

Do not conclude:

```text
compression improved
```

without evidence.

Compare:

```bash
tar -tzf archive
```

source:

```bash
find source
du -sh source
```

script version/change history.

---

# 118. Mount missing problem

Suppose intended data is normally mounted:

```text
/data
```

Mount fails after reboot.

Directory `/data` still exists but is empty on root filesystem.

Backup job:

```bash
tar /data
```

may **succeed** and create a tiny backup.

This is one of the most important backup failure modes:

```text
command success
≠ backup semantic success
```

Proper precheck may need expected mount verification:

```bash
mountpoint -q /data
```

if architecture requires `/data` to be a separate mounted filesystem.

This is **Supplementary**, but extremely operationally relevant.

---

# 119. Semantic validation

A high-quality backup job doesn't only ask:

```text
Did tar return 0?
```

It may also ask:

```text
Is expected mount present?
Are expected files present?
Is data size plausible?
Is newest data timestamp plausible?
Did member count collapse unexpectedly?
```

This protects against successfully backing up the wrong thing.

---

# 120. Backup success criteria

A useful model:

```text
technical success
+
semantic success
+
recoverability
```

Technical:

```text
command rc=0
```

Semantic:

```text
correct intended data captured
```

Recoverability:

```text
data can actually be restored and validated
```

All three matter.

---

# 121. L1-safe actions vs escalation

Generally safe inspection:

```text
tar -tf
rsync --dry-run
ls
stat
find with non-destructive options
df
du
systemctl status
journalctl
crontab -l
sha256sum
diff
```

State-changing actions requiring more caution:

```text
tar extraction into live system
rsync --delete
chmod/chown
removing backups
changing retention
changing cron
restarting services
unmounting/mounting
restoring production data
```

Your role authority/runbook decides actual L1 boundary.

Command knowledge does not grant authority.

---

# 122. Backup restore can overwrite newer data

Suppose current:

```text
config version = v5
```

backup:

```text
config version = v4
```

Blind restore:

```text
v5 → overwritten by v4
```

Recovery itself can become data loss.

Before production restore:

```text
identify recovery point
understand current state
preserve current state if appropriate
obtain approval
restore controlled scope
validate
```

---

# 123. Restore is a change

Operations mindset:

```text
restore
```

không phải:

```text
"just copy backup back"
```

Restore alters system state and may affect:

```text
availability
data consistency
configuration
permissions
application state
```

Nó cần rollback/handover thinking giống deployment.

---

# 124. Backup during release/change

Suppose change procedure:

```text
pre-change backup
        ↓
deploy
        ↓
validate
```

Nếu deployment fails:

```text
rollback artifact/config
```

Backup có thể là một recovery mechanism.

Nhưng backup không thay thế release rollback plan.

Một entire server restore chỉ vì một config line sai có thể quá nặng.

---

# 125. Versioned artifact vs filesystem backup

Nếu app artifact nằm trong artifact repository:

```text
app-1.2.3.war
```

thì rollback tốt có thể là:

```text
redeploy known-good 1.2.2
```

thay vì restore random filesystem archive.

Backup strategy phải phù hợp resource type.

Day 12–14 sẽ làm rõ hơn.

---

# 126. Current rsync security awareness

Vì rsync có thể hoạt động network-facing và parsing remote data/protocol, security patching quan trọng. Upstream đã phát hành 3.5.0 như một major security release trong tháng 8/2026 và 3.5.1 trong tháng 9/2026. :chatgpt-content-reference{index="23"}

Operational implication:

```text
rsync --version
```

nên được ghi trong troubleshooting/security context khi relevant.

Nhưng đừng tự compile latest upstream lên managed server nếu organization sử dụng distro packages.

Patch management phải theo approved mechanism.

---

# 127. Tar version differences

Check:

```bash
tar --version
```

GNU tar và BSD tar có option differences.

Examples trong bài này là GNU tar.

Nếu:

```text
tar (bsdtar)
```

thì hãy đọc:

```bash
man tar
```

trên chính system đó.

Không copy GNU-specific flags blindly.

---

# 128. Rsync version differences

Check:

```bash
rsync --version
```

Rsync versions có thể khác về:

```text
supported options
protocol negotiation
security fixes
checksum algorithms
features
```

Core options chúng ta dùng (`-a`, `-n`, `-i`, `--delete`) rất established, nhưng production behavior vẫn nên dựa local man page + approved version.

---

# 129. Evidence directory design

Trong lab:

```text
day6-capstone/
└── evidence/
    ├── environment.txt
    ├── script.txt
    ├── syntax-check.txt
    ├── manual-success.log
    ├── failure-service.log
    ├── failure-backup.log
    ├── cron.txt
    ├── cron-run.log
    ├── archive-list.txt
    ├── restore-validation.txt
    └── escalation-note.txt
```

Đây là excellent practice cho final exam style task.

---

# 130. Capturing environment evidence

```bash
{
    date
    hostname
    cat /etc/os-release
    tar --version | head -n 1
    rsync --version | head -n 1
    bash --version | head -n 1
} > "$HOME/day6-capstone/evidence/environment.txt"
```

Không cần capture:

```text
entire environment
```

nếu có secrets.

---

# 131. Script evidence

Bạn có thể preserve:

```bash
sha256sum "$HOME/day6-capstone/bin/day6-ops.sh" \
    > "$HOME/day6-capstone/evidence/script.sha256"
```

Và copy approved lab script nếu allowed.

Điều này giúp biết exact version tested.

---

# 132. Successful backup evidence

```bash
ls -lh "$latest_archive" \
    > "$HOME/day6-capstone/evidence/archive-file.txt"
```

```bash
tar -tzf "$latest_archive" \
    > "$HOME/day6-capstone/evidence/archive-list.txt"
```

Hash:

```bash
sha256sum "$latest_archive" \
    > "$HOME/day6-capstone/evidence/archive.sha256"
```

---

# 133. Restore evidence

```bash
find "$restore_dir" -type f -print \
    > "$HOME/day6-capstone/evidence/restored-files.txt"
```

```bash
diff -r \
    "$HOME/day6-capstone/source" \
    "$restore_dir/source" \
    > "$HOME/day6-capstone/evidence/restore-diff.txt"
```

Nếu `diff` returns 0:

```text
no content differences detected by that comparison
```

Ghi exit status:

```bash
printf 'diff_rc=%d\n' "$?" \
    >> "$HOME/day6-capstone/evidence/restore-diff.txt"
```

Nhớ capture `$?` ngay.

---

# 134. Cron evidence

```bash
crontab -l \
    > "$HOME/day6-capstone/evidence/crontab.txt"
```

Scheduler:

```bash
journalctl \
  -u cron.service \
  --since "30 minutes ago" \
  --no-pager \
  > "$HOME/day6-capstone/evidence/cron-journal.txt"
```

hoặc:

```bash
journalctl \
  -u crond.service \
  --since "30 minutes ago" \
  --no-pager \
  > "$HOME/day6-capstone/evidence/cron-journal.txt"
```

tùy distro.

---

# 135. Module 1 quay trở lại: grep evidence

```bash
grep -Ei \
  'error|critical|failed|unknown' \
  "$HOME/day6-capstone/logs/cron.log"
```

Bạn đang sử dụng:

```text
journalctl
grep
awk
Bash
exit codes
cron
tar
rsync
```

trong cùng một workflow.

Đó chính là Day 6 consolidation.

---

# 136. Module 2 quay trở lại: exit codes

Cron không ngồi nhìn screen.

Vì vậy:

```text
machine-readable status
```

cực kỳ quan trọng.

Một scheduled script mà luôn:

```text
exit 0
```

dù backup fail sẽ làm downstream monitoring/history misleading.

Đây là lý do exit-code design không phải theory riêng lẻ.

---

# 137. Cron + archive naming race

Nếu script chạy hai lần trong cùng second:

```text
source-20261001-230001.tar.gz
```

có thể collide.

Trong simple Day 6 schedule mỗi vài phút, timestamp-to-second là đủ.

Production concurrent execution cần stronger uniqueness/locking strategy.

Đây là **Supplementary**.

---

# 138. Overlapping backup jobs

Giả sử backup mất 20 phút.

Cron:

```cron
*/5 * * * *
```

Timeline:

```text
23:00 run A
23:05 run B
23:10 run C
23:15 run D
```

Potential consequences:

```text
disk I/O contention
duplicate archives
destination contention
network load
corrupted assumptions
```

Scheduling frequency phải lớn hơn reasonable runtime hoặc có concurrency control.

---

# 139. `flock` awareness

Supplementary pattern:

```bash
flock -n /path/to/lockfile \
    /path/to/day6-ops.sh
```

có thể prevent overlapping execution.

Nhưng:

```text
lock location
ownership
stale assumptions
error handling
```

cần được thiết kế.

Day 6 mandatory không yêu cầu `flock`.

---

# 140. Monitoring backup success

Cron log existence không đủ.

Một useful backup monitoring concept:

```text
last successful backup timestamp
archive age
archive size
exit status
verification status
```

Sau này Day 17 Prometheus/Grafana bạn có thể monitor những signals như vậy.

---

# 141. “Backup too old” check

Supplementary health idea:

```text
expected backup interval = 24h
last successful backup = 40h ago

→ backup stale
```

Đây là khác:

```text
current backup run failed
```

Có thể latest run chưa fail nhưng job không chạy.

Monitoring phải biết age/freshness.

---

# 142. Backup size anomaly

Nếu normal backup:

```text
4 GB
```

hôm nay:

```text
3 KB
```

archive technically valid vẫn có thể wrong.

Semantic monitoring có thể flag:

```text
suspiciously small backup
```

Không cần implement threshold hôm nay, nhưng phải hiểu.

---

# 143. Linux consolidation scenario tổng hợp

Giả sử ticket:

> “Java service unavailable lúc 02:10 sau nightly backup.”

Bạn bắt đầu:

```bash
date
hostname
```

Then:

```bash
systemctl status app.service
```

Logs:

```bash
journalctl \
  -u app.service \
  --since "01:50" \
  --until "02:20"
```

Disk:

```bash
df -P
```

Socket:

```bash
ss -lnt
```

Cron:

```bash
crontab -l
```

Backup log:

```bash
grep -Ei \
  'error|critical|failed' \
  /path/backup.log
```

Capacity history/evidence.

Potential timeline:

```text
02:00 backup started
02:07 disk 100%
02:08 application write failed
02:09 process exited
02:10 user reported outage
```

Now you have evidence-driven causality candidate.

---

# 144. L1 remediation example

Nếu runbook explicitly permits deleting **known temporary lab artifacts**:

```text
remove safe temp file
verify disk
restart authorized service
validate
```

Production old backup deletion may not be L1-safe.

If retention ownership unclear:

```text
escalate
```

The correct answer can be:

> “I know technically how to free disk, but deletion is outside my authorization.”

Đó là good operations behavior.

---

# 145. Recovery validation after remediation

Service becomes:

```text
active
```

chưa đủ.

Check:

```text
systemctl state
port
endpoint
logs
capacity
backup scheduler
```

Example:

```bash
systemctl is-active app.service
ss -lnt
curl ...
journalctl ...
df -P
```

Tùy application.

Recovery evidence must show symptom gone.

---

# 146. Root cause vs trigger vs contributing factor

Example:

```text
Trigger:
backup ran

Contributing factor:
no retention

Immediate cause:
filesystem full

Service failure mechanism:
application unable to write

Root organizational cause:
backup capacity/retention design absent
```

Don't flatten everything into:

```text
"cron caused outage"
```

Incident analysis can have multiple causal layers.

---

# 147. Exam Focus — `tar`

Bạn phải biết:

| Command/option | Ý nghĩa                                              |
| -------------- | ---------------------------------------------------- |
| `-c`           | create                                               |
| `-t`           | list                                                 |
| `-x`           | extract                                              |
| `-f`           | archive file                                         |
| `-v`           | verbose                                              |
| `-z`           | gzip                                                 |
| `-j`           | bzip2                                                |
| `-J`           | xz                                                   |
| `-C`           | operate relative to a directory                      |
| `--exclude`    | exclude matching members/files                       |
| `-d/--compare` | compare archive members with filesystem              |
| `-P`           | absolute-name behavior; dangerous unless intentional |

Bạn cần hiểu operation, không chỉ thuộc letters.

GNU tar chính thức tập trung `create`, `list`, `extract` như các operation nền tảng. :chatgpt-content-reference{index="24"}

---

# 148. Exam Focus — `rsync`

Bạn phải giải thích:

```bash
rsync -a source/ destination/
```

và đặc biệt:

```text
trailing slash source
```

Bạn phải biết:

```text
-a ≠ preserve everything
```

Official current man page xác nhận `-a = -rlptgoD` và không include ACL/xattr/hardlink options. :chatgpt-content-reference{index="25"}

---

# 149. Exam Focus — `--dry-run`

Nếu practical exam cho:

```bash
rsync -a --delete ...
```

mà bạn chạy thẳng vào important destination:

```text
đó là operationally weak behavior
```

Expected thought process:

```text
inspect paths
dry-run
review output
execute
verify
```

---

# 150. Exam Focus — backup verification

Câu:

> “Command tar return 0. Backup đã verified chưa?”

Correct:

```text
Command success là evidence tốt,
nhưng chưa đủ để chứng minh recoverability.
```

Need:

```text
archive exists
read/list
expected contents
restore test
content/metadata/application validation as required
```

---

# 151. Exam Focus — rsync mirror vs backup

Bạn phải giải thích được:

```text
rsync destination
```

có thể là mirror/copy.

Nếu dùng:

```text
--delete
```

source deletion có thể propagate.

Do đó historical recovery requires design ngoài một mutable mirror.

---

# 152. Exam Focus — troubleshooting

Scenario:

```text
cron backup không chạy
```

Reasoning:

```text
cron service
entry
user
environment
paths
permissions
script
source
destination
capacity
tool status
logs
```

Scenario:

```text
archive exists nhưng restore fail
```

Reasoning:

```text
integrity/readability
format/options
permissions
disk capacity
contents
restore path
metadata
```

---

# 153. Knowledge Check

Bạn nên tự trả lời được bảng sau mà không nhìn command mẫu:

| Question                                                          | Expected reasoning                                 |
| ----------------------------------------------------------------- | -------------------------------------------------- |
| `.tar.gz` khác `.tar` thế nào?                                    | Archive + compression                              |
| `tar -t` làm gì?                                                  | List archive                                       |
| `tar -x` làm gì?                                                  | Extract                                            |
| Tại sao restore vào staging?                                      | Giới hạn blast radius + validation                 |
| Backup file tồn tại có đủ không?                                  | Không                                              |
| Trailing slash rsync có quan trọng không?                         | Rất quan trọng                                     |
| `rsync -a` có include ACL/xattrs không?                           | Không mặc định                                     |
| `--dry-run` có thay đổi destination không?                        | Không                                              |
| `--delete` làm gì?                                                | Xóa receiver extras trong scope                    |
| Vì sao `--delete` nguy hiểm?                                      | Có thể replicate deletion/wrong-source state       |
| Rsync mirror có phải historical backup không?                     | Không tự động                                      |
| Tar live DB directory có phải PostgreSQL-consistent backup không? | Không nên giả định                                 |
| Exit 0 đủ chứng minh semantic backup success chưa?                | Chưa                                               |
| Tại sao verify restore quan trọng?                                | Backup chỉ có value nếu recoverable                |
| Backup cùng disk source bảo vệ disk failure không?                | Không                                              |
| Compression có encrypt data không?                                | Không                                              |
| Checksum có encrypt data không?                                   | Không                                              |
| Vì sao cron backup có thể gây outage?                             | Disk/I/O/capacity/overlap                          |
| Khi nào nên escalate?                                             | Out of scope / destructive / root cause outside L1 |

---

# 154. Independent Practical Challenge — Day 6 Capstone

Đây là challenge tôi khuyến nghị bạn tự làm **không nhìn lại integrated script mẫu**.

Bạn phải xây:

```text
day6-health-backup.sh
```

trong `$HOME` lab.

Nó phải có:

```text
environment/dependency validation

service health check

filesystem capacity check

listening-port check

tar timestamped backup

tar readability verification

rsync destination copy

structured timestamped logs

meaningful exit codes

no auto-remediation

no rsync --delete
```

Sau đó chứng minh:

```text
normal run succeeds

fake service detected

invalid source detected

unwritable/invalid backup destination detected

closed port detected

corrupted archive fails verification

restore test succeeds

cron executes script

cron execution produces evidence
```

Cuối cùng viết escalation note cho một failure mà bạn **không remediation**, giả định destination thuộc team khác.

---

# 155. Điều kiện hoàn thành Day 6

Day 6 không hoàn thành chỉ vì bạn đã đọc 3 modules.

Tôi chỉ coi phần kiến thức/practical objectives đạt khi bạn có thể tự làm flow:

```text
LOG ANALYSIS
    │
    ▼
journalctl
grep
awk
sed
    │
    ▼
MANUAL HEALTH CHECK
    │
    ▼
BASH SCRIPT
    │
    ▼
EXIT CODES
    │
    ▼
CRON
    │
    ▼
TAR BACKUP
    │
    ▼
RSYNC COPY
    │
    ▼
VERIFY
    │
    ▼
RESTORE TEST
    │
    ▼
FAILURE INJECTION
    │
    ▼
TROUBLESHOOTING
    │
    ├─────────── L1 recovery
    │
    └─────────── escalation
                     │
                     ▼
                  EVIDENCE
```

Đó chính xác là spirit của Assignment Day 6: **health-check + backup script, scheduling, injected failure và escalation evidence.** :chatgpt-content-reference{index="26"}

---

# 156. Những nguyên tắc quan trọng nhất cần mang sang các Day tiếp theo

Nếu phải cô đọng Module 3 thành mental model lâu dài, hãy nhớ:

```text
A backup command is not a backup strategy.

A successful command is not proof of recoverability.

A readable archive is stronger evidence than file existence.

A successful restore test is stronger evidence than archive listing.

A mirror is not automatically a historical backup.

rsync trailing slash changes meaning.

rsync --delete deserves a dry-run and explicit review.

tar/rsync are filesystem tools, not automatically application-consistent backup tools.

Permissions and metadata matter as much as content.

A scheduled job that nobody verifies is not trustworthy automation.

Failure injection must have an expected result and rollback.

Recovery is a state-changing operation.

Evidence comes before guessing.

Know your L1 boundary.

When outside your boundary, escalate with enough evidence that the next person can continue immediately.
```

Và chuỗi vận hành cuối cùng của toàn **Linux Fundamentals Day 2–6** là:

```text
UNDERSTAND
   ↓
INSPECT
   ↓
CONFIGURE
   ↓
VALIDATE
   ↓
OBSERVE
   ↓
AUTOMATE
   ↓
SCHEDULE
   ↓
BACK UP
   ↓
VERIFY
   ↓
BREAK SAFELY
   ↓
TROUBLESHOOT
   ↓
RECOVER / ROLLBACK
   ↓
DOCUMENT
   ↓
ESCALATE / HAND OVER
```

Sau Module 3, toàn bộ **mandatory theory của Day 6 đã được trình bày**. Nhưng theo mục tiêu khóa học, Day 6 chỉ thực sự đạt practical mastery sau khi bạn tự chạy capstone, cung cấp command/output/evidence và chứng minh được ít nhất một failure path + restore path; đó cũng là bước hợp lý nhất trước khi chuyển sang **Day 7 — AWS Cloud Fundamentals**.
