# Day 5 — Module 2: Linux Storage, Filesystems, Mounts, Capacity & Integrated Troubleshooting

Tiếp tục đúng phạm vi **Day 5**. MASTER_SYLLABUS quy định phần bắt buộc của ngày hôm nay gồm **IP/DNS/ports, `curl`, `ss`, `ping`, `traceroute`, firewall basics, disks, filesystems, mounts, capacity checks**; Assignment/Lab là **diagnose một simulated connectivity/storage issue và tạo health-check record**. Module 1 đã xử lý networking, vì vậy Module 2 này hoàn thành nửa storage và tích hợp toàn bộ Day 5. :chatgpt-content-reference{index="0"}

---

# 1. Syllabus Alignment

Phạm vi **Mandatory — bắt buộc của syllabus** trong Module 2 là:

| Chủ đề          | Bạn phải làm được                                                                  |
| --------------- | ---------------------------------------------------------------------------------- |
| Disks           | Nhận diện block device và quan hệ giữa disk/device với filesystem                  |
| Filesystems     | Hiểu filesystem là gì, filesystem type, storage allocation                         |
| Mounts          | Hiểu mount point, mount/unmount, nhận biết filesystem đang được mount ở đâu        |
| Capacity checks | Kiểm tra byte/block capacity và inode capacity                                     |
| Troubleshooting | Phân biệt disk full, inode full, missing mount, wrong mount, wrong filesystem/path |
| Day-5 Lab       | Diagnose connectivity/storage issue và tạo health-check record                     |

Các nội dung như partition, `lsblk`, `blkid`, `findmnt`, `/etc/fstab`, UUID, inode, `df`, `du`, loop devices, ext4/XFS awareness, deleted-open files và mount options là **Supplementary / Beyond the explicit syllabus**. Syllabus không liệt kê từng command này, nhưng nếu không hiểu chúng thì gần như không thể thực hiện đúng yêu cầu “disks, filesystems, mounts, capacity checks” trong môi trường Operations thực tế.

---

# 2. Mục tiêu thực sự của Module 2

Sau bài này, khi một Java service báo:

```text
No space left on device
```

bạn không được lập tức kết luận:

```text
Disk is full.
```

Bạn phải có khả năng đặt ra những câu hỏi chính xác hơn:

```text
Filesystem nào chứa path application đang ghi?

Filesystem đó có hết data blocks không?

Hay còn GB trống nhưng đã hết inode?

Expected filesystem có thực sự được mount không?

Application có đang ghi nhầm xuống root filesystem không?

Mount có ở read-only mode không?

Device có tồn tại nhưng filesystem chưa mount không?

df và du có đang nói về cùng một filesystem không?

Có deleted file vẫn bị process giữ open không?
```

Tương tự, khi thấy:

```text
/dev/sdb
```

bạn không được mặc định nói:

> "`/dev/sdb` là filesystem."

Nó có thể chỉ là một **block device**.

Mental model chính xác phải là:

```text
Physical / Virtual Storage
          │
          ▼
     Block Device
       /dev/...
          │
          ▼
      Partition
     (optional)
          │
          ▼
      Filesystem
   ext4 / XFS / ...
          │
          ▼
        Mount
          │
          ▼
      Mount Point
      /data /var ...
          │
          ▼
 Files and Directories
          │
          ▼
    Application
```

Đây là mental model quan trọng nhất của Module 2.

---

# 3. Linux không có khái niệm “ổ C:, ổ D:” giống Windows

Linux xây dựng một **single directory tree** bắt đầu từ:

```text
/
```

Ví dụ:

```text
/
├── etc
├── home
├── opt
├── tmp
├── usr
└── var
    ├── log
    └── lib
```

Một filesystem khác có thể được gắn vào một directory trong tree này.

Ví dụ:

```text
/dev/sdb1
    │
    │ ext4 filesystem
    ▼
 mounted at
    │
    ▼
/data
```

Sau khi mount:

```text
/data/file1
/data/file2
```

thực tế nằm trên filesystem `/dev/sdb1`.

Điều này khác với Windows-style drive letters.

Mental model:

```text
Linux namespace:

                     /
         ┌───────────┼─────────────┐
        /etc        /var          /data
                                  ▲
                                  │ mount
                              /dev/sdb1
```

---

# 4. Disk, block device, partition, filesystem và mount point khác nhau thế nào?

Đây là phần phải hiểu cực chắc.

## Disk / storage device

Một storage device vật lý hoặc virtual có thể xuất hiện dưới tên:

```text
/dev/sda
/dev/sdb
/dev/vda
/dev/nvme0n1
```

Tên phụ thuộc hardware, virtualization, kernel driver và environment.

Trong cloud bạn cũng có thể gặp virtual block storage.

---

## Block device

Linux expose storage qua **block device interface**.

Block device đọc/ghi dữ liệu theo block-oriented access.

Ví dụ:

```text
/dev/sda
/dev/nvme0n1
/dev/mapper/vg01-lvdata
/dev/loop0
```

Không phải tất cả block devices đều là physical disks.

Ví dụ:

```text
/dev/loop0
```

có thể là một file được trình bày cho kernel như một block device.

`lsblk` là utility chính để liệt kê block devices; upstream util-linux hiện tại cũng lưu ý output mặc định của `lsblk` có thể thay đổi, nên khi scripting nên yêu cầu rõ các columns cần dùng. :chatgpt-content-reference{index="1"}

---

## Partition

Một disk có thể được chia thành partitions.

Ví dụ:

```text
/dev/sda
├── /dev/sda1
├── /dev/sda2
└── /dev/sda3
```

Với NVMe:

```text
/dev/nvme0n1
├── /dev/nvme0n1p1
└── /dev/nvme0n1p2
```

Partition chỉ là vùng logical trên block device.

Partition **chưa tự động là filesystem**.

Bạn có thể có:

```text
partition
   │
   ├── ext4 filesystem
   ├── XFS filesystem
   ├── swap
   ├── LVM physical volume
   └── raw application data
```

---

# 5. Filesystem là gì?

Filesystem là cấu trúc dữ liệu mà operating system dùng để tổ chức:

```text
directories
files
names
permissions
ownership
timestamps
data blocks
metadata
```

Ví dụ filesystem phổ biến:

```text
ext4
XFS
Btrfs
vfat
tmpfs
NFS
```

Trong server Linux thông thường, **ext4** và **XFS** rất phổ biến.

Ví dụ một block device:

```text
/dev/sdb1
```

có thể chứa:

```text
ext4 filesystem
```

Mental model:

```text
/dev/sdb1
     │
     │ contains
     ▼
+-----------------------+
| ext4 filesystem       |
|                       |
| metadata              |
| inode information     |
| directories           |
| file data             |
| free-space tracking   |
+-----------------------+
```

---

# 6. ext4 và XFS awareness

Đây là **Supplementary**, không phải syllabus yêu cầu bạn trở thành filesystem engineer.

`ext4` là filesystem lâu đời và phổ biến trong Linux. Kernel documentation mô tả ext4 tổ chức storage thành logical blocks và block groups; filesystem cũng duy trì metadata như inode tables và allocation bitmaps. :chatgpt-content-reference{index="2"}

`XFS` là journaling filesystem có tính scalability cao. Ví dụ trong **RHEL 10**, Red Hat hiện sử dụng XFS làm default local filesystem và vẫn hỗ trợ ext4. Đây là thông tin distro-specific, không phải quy luật cho mọi Linux distribution. :chatgpt-content-reference{index="3"}

Với fresher, điều cần nhớ không phải:

> "Filesystem nào tốt nhất?"

Mà là:

```text
Filesystem type nào đang thực sự được dùng?

Nó được mount ở đâu?

Capacity ra sao?

Mount options ra sao?

Application data nằm trên filesystem nào?
```

---

# 7. Filesystem ≠ directory

Giả sử có directory:

```text
/data
```

Directory đó có thể chỉ nằm trên root filesystem `/`.

Hoặc nó có thể là mount point cho filesystem riêng.

Hai trường hợp nhìn bằng `ls` có thể rất giống nhau.

### Trường hợp A

```text
/
└── data
```

`/data` chỉ là directory nằm trong `/`.

### Trường hợp B

```text
/
└── data
     ▲
     │
 /dev/sdb1 mounted here
```

Bây giờ `/data` là entry point vào filesystem khác.

Do đó:

```bash
ls /data
```

không đủ để biết `/data` thuộc filesystem nào.

---

# 8. Lệnh đầu tiên: `lsblk`

Một command cực kỳ hữu ích:

```bash
lsblk
```

Ví dụ:

```text
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0   30G  0 disk
├─sda1        8:1    0    1G  0 part /boot
└─sda2        8:2    0   29G  0 part /
sdb           8:16   0   20G  0 disk
└─sdb1        8:17   0   20G  0 part /data
```

Đọc thành:

```text
sda
 ├─ sda1 → partition → /boot
 └─ sda2 → partition → /

sdb
 └─ sdb1 → partition → /data
```

`lsblk` đọc block-device information chủ yếu từ sysfs và udev; nó cũng có thể hiển thị filesystem/mount information. :chatgpt-content-reference{index="4"}

---

# 9. Đừng phụ thuộc output mặc định của `lsblk` trong script

Cho human inspection:

```bash
lsblk
```

rất tiện.

Nhưng để evidence hoặc scripting, tốt hơn:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINTS
```

Ví dụ:

```text
NAME  SIZE TYPE FSTYPE UUID                                 MOUNTPOINTS
sda    30G disk
├─sda1  1G part xfs    ...                                  /boot
└─sda2 29G part xfs    ...                                  /
sdb    20G disk
└─sdb1 20G part ext4   123e4567-e89b-12d3-a456-426614174000 /data
```

Upstream documentation của `lsblk` khuyến nghị scripts chỉ rõ columns vì default output có thể thay đổi. :chatgpt-content-reference{index="5"}

---

# 10. `lsblk -f`

Một shorthand rất tiện:

```bash
lsblk -f
```

Thường hiển thị:

```text
NAME FSTYPE FSVER LABEL UUID FSAVAIL FSUSE% MOUNTPOINTS
```

Bạn có thể nhanh chóng thấy:

```text
device
filesystem type
UUID
mountpoint
```

Nhưng cần nhớ:

> `lsblk` cho bạn block-device-centric view.

Nó không phải lúc nào cũng là command tốt nhất để hỏi:

> "Path `/var/lib/myapp` thuộc filesystem nào?"

Cho câu hỏi đó, `findmnt` hoặc `df <path>` thường phù hợp hơn.

---

# 11. `blkid`

**Supplementary**

`blkid` xác định attributes của block devices, bao gồm filesystem content type, UUID và labels. :chatgpt-content-reference{index="6"}

Ví dụ:

```bash
sudo blkid
```

Có thể thấy:

```text
/dev/sdb1: UUID="6e03..." TYPE="ext4" PARTUUID="..."
```

Hoặc kiểm tra một device cụ thể:

```bash
sudo blkid /dev/sdb1
```

Mental model:

```text
/dev/sdb1
    │
    ├─ TYPE=ext4
    └─ UUID=...
```

---

# 12. UUID là gì và tại sao quan trọng?

UUID của filesystem là identifier ổn định hơn tên kiểu:

```text
/dev/sdb1
```

Tên `/dev/sdX` có thể thay đổi theo device discovery/hardware configuration.

Upstream `mount(8)` và `fstab(5)` đều khuyến nghị filesystem identifiers như `UUID=` hoặc `LABEL=` thay vì phụ thuộc hoàn toàn vào kernel device names, bởi device naming có thể thay đổi khi hardware thay đổi. :chatgpt-content-reference{index="7"}

Ví dụ:

```text
UUID=3e6be9de-8139-11d1-9106-a43f08d823a6
```

thay vì:

```text
/dev/sdb1
```

Trong Operations, đây là một distinction rất quan trọng.

---

# 13. Mount là gì?

Mount là hành động **attach filesystem vào Linux directory hierarchy**.

Ví dụ:

```bash
sudo mount /dev/sdb1 /data
```

Mental model:

```text
/dev/sdb1
 ext4 filesystem
       │
       │ mount
       ▼
     /data
```

Sau mount:

```text
/data
```

trở thành cách users/applications truy cập filesystem `/dev/sdb1`.

Red Hat documentation hiện tại mô tả đúng mental model này: filesystem được attach vào directory tree tại mount point và trở nên accessible qua directory đó. :chatgpt-content-reference{index="8"}

---

# 14. Mount point là directory

Trước:

```bash
sudo mkdir -p /data
```

`/data` chỉ là một directory.

Sau:

```bash
sudo mount /dev/sdb1 /data
```

`/data` trở thành **mount point**.

Điều quan trọng:

> Mount point không phải disk.

Nó là vị trí trong directory tree nơi filesystem được attached.

---

# 15. Một nguy hiểm quan trọng: mount che nội dung cũ bên dưới

Giả sử root filesystem có:

```text
/data/old-file.txt
```

Sau đó bạn mount filesystem khác lên `/data`:

```bash
sudo mount /dev/sdb1 /data
```

Nội dung cũ nằm dưới mount point **không bị xóa**, nhưng bình thường sẽ không còn visible qua `/data` trong khi filesystem kia đang mounted.

Red Hat hiện cũng cảnh báo rằng Linux cho phép mount filesystem lên một directory có nội dung và khi mount thì original directory contents trở nên inaccessible qua path đó. :chatgpt-content-reference{index="9"}

Điều này tạo ra một lỗi Operations rất thú vị.

---

# 16. Missing mount có thể làm root filesystem đầy

Hãy tưởng tượng expected architecture:

```text
/dev/sdb1 → /data
```

Application ghi:

```text
/data/uploads/*
```

Bình thường:

```text
uploads → /dev/sdb1
```

Nhưng sau reboot, `/data` không mount thành công.

Directory `/data` vẫn tồn tại trên root filesystem.

Application tiếp tục ghi:

```text
/data/uploads/*
```

nhưng bây giờ dữ liệu thực sự đi vào:

```text
/
```

Kết quả:

```text
/data filesystem appears "missing"

AND

root filesystem starts filling up
```

Đây là classic storage incident.

---

# 17. Vì thế `df -h` thôi chưa đủ

Nếu bạn chỉ chạy:

```bash
df -h
```

và thấy:

```text
/dev/sda2  29G  28G  1G  97% /
```

bạn biết root filesystem gần đầy.

Nhưng chưa biết:

```text
Tại sao?

/data expected mount có bị mất không?

Log tăng?

Application ghi nhầm path?

Deleted file?

Package/cache?

Database?
```

Capacity check chỉ là bước đầu.

---

# 18. Command quan trọng: `findmnt`

`findmnt` được thiết kế để list/search mounted filesystems; nó có thể đọc mount information và cũng có thể tìm filesystem đang chứa một path bằng `--target`. :chatgpt-content-reference{index="10"}

Liệt kê mount tree:

```bash
findmnt
```

Ví dụ:

```text
TARGET      SOURCE     FSTYPE OPTIONS
/           /dev/sda2  ext4   rw,relatime
├─/boot     /dev/sda1  ext4   rw,relatime
└─/data     /dev/sdb1  xfs    rw,relatime
```

Đây là một view rất mạnh:

```text
Where is each filesystem attached?
```

---

# 19. Kiểm tra một mount point cụ thể

```bash
findmnt /data
```

Nếu mounted:

```text
TARGET SOURCE    FSTYPE OPTIONS
/data  /dev/sdb1 ext4   rw,relatime
```

Nếu không:

```text
no matching mount
```

Tùy version/output, dùng rõ option cũng tốt:

```bash
findmnt --target /data
```

---

# 20. Câu hỏi mạnh hơn: path này thuộc filesystem nào?

Giả sử application báo lỗi tại:

```text
/var/lib/myapp/uploads/file.bin
```

Bạn không cần đoán.

Chạy:

```bash
findmnt --target /var/lib/myapp/uploads
```

`findmnt --target` có thể nhận file hoặc directory và tìm filesystem tương ứng cho path đó. :chatgpt-content-reference{index="11"}

Ví dụ:

```text
TARGET SOURCE                    FSTYPE
/var   /dev/mapper/vg-var        xfs
```

Bạn vừa xác định:

```text
Application path is stored on /var filesystem.
```

Đây là evidence cực kỳ quan trọng.

---

# 21. `findmnt` vs `lsblk`

Hãy hình dung:

```text
lsblk
```

hỏi:

> Block devices của tôi được tổ chức thế nào?

Còn:

```text
findmnt
```

hỏi:

> Filesystems đang được gắn vào directory tree thế nào?

Ví dụ:

```text
lsblk
disk-centric view

findmnt
mount-centric view
```

Cả hai cùng cần thiết.

---

# 22. `df` — filesystem capacity

`df` là command cốt lõi cho capacity check.

Theo GNU documentation, `df` báo space usage/availability của filesystem chứa path được yêu cầu; không truyền path thì nó hiển thị mounted filesystems. Nó không thể nói free space của một unmounted filesystem theo cách portable thông thường. :chatgpt-content-reference{index="12"}

Lệnh quen thuộc:

```bash
df -h
```

`-h`:

```text
human-readable
```

Ví dụ:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        30G   18G   11G  63% /
/dev/sdb1       100G   81G   19G  82% /data
```

---

# 23. Đọc `df -h`

Các column:

```text
Filesystem
Size
Used
Avail
Use%
Mounted on
```

Ví dụ:

```text
/dev/sdb1 100G 81G 19G 82% /data
```

Interpretation:

```text
filesystem source = /dev/sdb1

filesystem size   ≈ 100G

allocated/used    ≈ 81G

available         ≈ 19G

usage             ≈ 82%

mount point       = /data
```

Không nên nói:

> "Disk `/dev/sdb` dùng 82%"

nếu evidence chỉ nói filesystem `/dev/sdb1` tại `/data` dùng 82%.

Disk và filesystem không phải một.

---

# 24. `df -hT`

Một command rất hữu ích:

```bash
df -hT
```

`-T` thêm filesystem type.

Ví dụ:

```text
Filesystem     Type  Size Used Avail Use% Mounted on
/dev/sda2      xfs    30G  18G   12G  61% /
/dev/sdb1      ext4  100G  81G   14G  86% /data
```

Bây giờ bạn có:

```text
mount
capacity
filesystem type
```

trong một view.

---

# 25. Tốt hơn nữa: kiểm tra đúng path

Nếu incident nói:

```text
/opt/tomcat/logs cannot write
```

không cần đọc toàn bộ `df -h`.

Chạy:

```bash
df -hT /opt/tomcat/logs
```

Hoặc:

```bash
df -hT /opt/tomcat
```

Bạn sẽ thấy filesystem thực sự chứa path đó.

Điều này giảm ambiguity.

---

# 26. `df` đo filesystem, không đo directory

Điểm cực kỳ quan trọng.

```bash
df -h /var/log
```

không có nghĩa:

> `/var/log` directory đang dùng bao nhiêu.

Nó nghĩa:

> filesystem chứa `/var/log` đang dùng bao nhiêu.

Muốn biết directory tree chiếm bao nhiêu, dùng:

```bash
du
```

---

# 27. `du` — directory/file space usage

`du` ước lượng space usage của files/directories, recursively với directories. :chatgpt-content-reference{index="13"}

Ví dụ:

```bash
du -sh /var/log
```

Có thể ra:

```text
2.8G    /var/log
```

Giải nghĩa:

```text
-s = summarize
-h = human-readable
```

---

# 28. `df` vs `du`

Đây là câu hỏi exam/troubleshooting kinh điển.

```text
df
```

hỏi:

> Filesystem đã sử dụng/available bao nhiêu?

```text
du
```

hỏi:

> Các files/directories tôi scan đang tiêu thụ bao nhiêu allocated space?

Mental model:

```text
df
 └─ filesystem-level allocation accounting

du
 └─ file/directory-tree accounting
```

Không được dùng thay thế nhau.

---

# 29. Ví dụ

```bash
df -h /var
```

```text
Filesystem Size Used Avail Use%
/dev/sda2    30G  28G   2G   94%
```

Sau đó:

```bash
sudo du -sh /var/log
```

```text
9G /var/log
```

Bạn biết:

```text
whole filesystem used = 28G

/var/log contributes approximately 9G
```

Bạn chưa biết 19G còn lại nằm ở đâu.

---

# 30. Tìm directory lớn

Một useful command:

```bash
sudo du -xhd1 /var
```

Ý tưởng:

```text
-x
stay on one filesystem

-h
human-readable

-d1
one directory level
```

Ví dụ:

```text
100M /var/cache
9.0G /var/log
12G  /var/lib
21G  /var
```

Từ đây bạn drill down:

```bash
sudo du -xhd1 /var/lib
```

Không chạy:

```bash
du -ah /
```

một cách vô tội vạ trên production server lớn vì nó có thể tạo I/O rất nhiều và scan hàng triệu files.

---

# 31. Capacity troubleshooting cũng phải quan tâm blast radius

`df` thường nhẹ.

```bash
df -h
```

đọc filesystem statistics.

`du` có thể phải traverse directory tree rất lớn.

Ví dụ:

```bash
sudo du -xhd1 /
```

trên host với hàng triệu files có thể tốn I/O và thời gian đáng kể.

Operations principle:

> Tool là read-only không có nghĩa hoàn toàn cost-free.

Phải xét performance impact.

---

# 32. Apparent size và allocated size

**Supplementary**

Một file có thể có:

```text
logical/apparent size
```

khác:

```text
actual allocated disk blocks
```

Đặc biệt với sparse files.

GNU `du` mặc định quan tâm device usage; option:

```bash
du --apparent-size
```

cho apparent size. :chatgpt-content-reference{index="14"}

Vì vậy:

```bash
ls -lh huge-file
```

có thể thấy:

```text
100G
```

trong khi:

```bash
du -h huge-file
```

có thể nhỏ hơn rất nhiều nếu file sparse.

---

# 33. Capacity không chỉ là bytes: còn inode

Đây là phần cực kỳ quan trọng.

Filesystem không chỉ cần free data blocks.

Để tạo file mới, nhiều filesystem còn cần metadata structures, đặc biệt **inode**.

Inode conceptually chứa metadata về filesystem object như:

```text
file type
permissions
owner
timestamps
block pointers/extents
...
```

Directory entry map:

```text
name
→ inode
```

Simplified:

```text
report.log
     │
     ▼
  inode 12345
     │
     ├─ metadata
     └─ data block references
```

---

# 34. `df -i`

Kiểm tra inode:

```bash
df -i
```

GNU `df` dùng `-i` để hiển thị inode information thay vì block usage. :chatgpt-content-reference{index="15"}

Ví dụ:

```text
Filesystem      Inodes   IUsed    IFree IUse% Mounted on
/dev/sda2      1966080 1966000       80  100% /
```

Nhưng:

```bash
df -h /
```

có thể nói:

```text
30G  10G  20G  34%
```

Bạn có:

```text
20 GB free
BUT
almost zero free inodes
```

Application vẫn có thể không tạo được file mới.

---

# 35. “No space left on device” không nhất thiết là hết GB

Đây là một câu phải nhớ.

Error:

```text
No space left on device
```

có thể do:

```text
data blocks exhausted
```

hoặc:

```text
inodes exhausted
```

hoặc một resource/allocation constraint khác tùy filesystem.

Vì vậy cặp command kinh điển:

```bash
df -h /problem/path
df -i /problem/path
```

---

# 36. Byte-full scenario

Ví dụ:

```bash
df -h /var
```

```text
Filesystem Size Used Avail Use%
/dev/sda2    30G  30G  100M  100%
```

```bash
df -i /var
```

```text
IUse% 18%
```

Interpretation:

```text
data/block capacity exhaustion
not inode exhaustion
```

---

# 37. Inode-full scenario

```bash
df -h /var
```

```text
Use% 40%
```

Nhưng:

```bash
df -i /var
```

```text
IUse% 100%
```

Interpretation:

```text
filesystem has free bytes
but cannot allocate additional inode objects
```

Typical contributor:

```text
huge numbers of tiny files
```

Ví dụ:

```text
sessions
cache files
temporary files
mail spool entries
application-generated small files
```

---

# 38. Không fix inode full bằng cách xóa một file 10 GB

Nếu inode exhaustion là vấn đề:

```text
1 huge file
```

dùng khoảng một inode.

Xóa nó có thể giải phóng rất nhiều data blocks nhưng chỉ giải phóng một inode.

Ngược lại:

```text
1,000,000 tiny files
```

có thể tiêu tốn khoảng một triệu inode.

Bạn phải xử lý đúng resource đang exhausted.

---

# 39. Mount state: read-write vs read-only

Filesystem thường có thể được mounted:

```text
rw
```

hoặc:

```text
ro
```

`rw` = read-write.

`ro` = read-only.

Kiểm tra:

```bash
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS /data
```

Ví dụ:

```text
/data /dev/sdb1 ext4 rw,relatime
```

Hoặc:

```text
/data /dev/sdb1 ext4 ro,relatime
```

Nếu application báo:

```text
Read-only file system
```

thì đừng cố fix bằng:

```bash
chmod 777
```

Permissions và filesystem read-only state là hai layer khác nhau.

---

# 40. Permission denied ≠ read-only filesystem

Ví dụ:

```text
Permission denied
```

có thể do:

```text
Unix permissions
ownership
ACL
SELinux
other security controls
```

Trong khi:

```text
Read-only file system
```

thường cho thấy write bị filesystem/mount state chặn.

Mental model:

```text
Write operation
     │
     ├── filesystem mounted rw?
     │
     ├── free space/inode?
     │
     ├── Unix permission?
     │
     ├── security policy?
     │
     └── application condition?
```

---

# 41. Mount options awareness

**Supplementary**

Một số mount options thường gặp:

| Option     | Ý nghĩa cơ bản                                                  |
| ---------- | --------------------------------------------------------------- |
| `rw`       | read-write                                                      |
| `ro`       | read-only                                                       |
| `noexec`   | hạn chế direct execution từ filesystem                          |
| `nosuid`   | không honor set-user-ID/set-group-ID semantics theo mount       |
| `nodev`    | không interpret device special files trên filesystem            |
| `relatime` | access time update policy                                       |
| `noauto`   | không được mount bởi `mount -a`                                 |
| `nofail`   | absence của device không được coi là fatal theo fstab semantics |

`fstab(5)` hiện mô tả `ro/rw`, `noauto`, `nofail` và nhiều filesystem-independent options. :chatgpt-content-reference{index="16"}

Không copy security options ngẫu nhiên. Ví dụ `noexec` có thể ảnh hưởng workload cần execute binaries/scripts từ filesystem đó.

---

# 42. `/etc/fstab` — persistent mount configuration

Mount thủ công:

```bash
sudo mount /dev/sdb1 /data
```

thường chỉ tạo runtime mount state.

Sau reboot, muốn mount lại theo boot configuration, Linux systems thường dựa trên:

```text
/etc/fstab
```

`fstab(5)` mô tả đây là static filesystem information được tools/daemons sử dụng để biết filesystem có thể được mount như thế nào. :chatgpt-content-reference{index="17"}

Ví dụ:

```text
UUID=abc123...  /data  ext4  defaults  0  2
```

---

# 43. Sáu field của `/etc/fstab`

Một entry:

```text
UUID=1234-5678  /data  ext4  defaults  0  2
```

Đọc thành:

| Field | Giá trị          | Ý nghĩa                   |
| ----- | ---------------- | ------------------------- |
| 1     | `UUID=1234-5678` | filesystem/source         |
| 2     | `/data`          | mount point               |
| 3     | `ext4`           | filesystem type           |
| 4     | `defaults`       | mount options             |
| 5     | `0`              | dump field                |
| 6     | `2`              | fsck ordering information |

Upstream `fstab(5)` xác định rõ các field này; nó cũng khuyến nghị UUID/LABEL vì device names có thể đổi. :chatgpt-content-reference{index="18"}

---

# 44. Tại sao `/etc/fstab` nguy hiểm?

Sai fstab có thể gây:

```text
mount failure
boot delays
service dependencies failing
system boot/emergency-mode issues depending configuration
wrong filesystem mounted
wrong mount options
```

Vì vậy trước khi sửa:

```bash
sudo cp -a /etc/fstab /etc/fstab.backup
```

và record:

```text
before state
after state
rollback path
```

Nhưng ngay cả backup command cũng phải tuân local policy/change control.

---

# 45. Luôn dùng UUID đúng filesystem

Tìm UUID:

```bash
lsblk -f
```

hoặc:

```bash
sudo blkid /dev/sdb1
```

Sau đó compare chính xác với fstab.

Không đoán.

Không copy UUID từ server khác.

---

# 46. Device names có thể không ổn định

Ví dụ hôm nay:

```text
/dev/sdb1
```

sau thay đổi hardware/device discovery có thể không còn là thiết bị bạn tưởng.

Vì vậy persistent mount thường nên sử dụng:

```text
UUID=...
```

hoặc identifier phù hợp khác.

Cả `mount(8)` và `fstab(5)` đều khuyến nghị identifiers như UUID/LABEL cho mục đích này. :chatgpt-content-reference{index="19"}

---

# 47. Test fstab rất quan trọng

Sau khi sửa fstab, không nên:

```text
"reboot and hope"
```

Một basic validation trong approved lab là:

```bash
sudo mount -a
```

Nhưng hãy hiểu:

`mount -a` thử mount các entries applicable trong fstab, vì vậy nó có thể ảnh hưởng nhiều filesystem.

Trên production, đây là state-changing command và cần hiểu blast radius.

Một read-oriented validation hữu ích trước đó:

```bash
findmnt --verify
```

nếu version hỗ trợ phù hợp.

Trên systemd systems, upstream `fstab(5)` hiện cũng khuyến nghị `systemctl daemon-reload` sau khi sửa fstab vì systemd có thể consume/generate mount units từ configuration này. :chatgpt-content-reference{index="20"}

---

# 48. `mount`

Manual mount:

```bash
sudo mount /dev/sdb1 /data
```

Nếu filesystem/source đã có entry đầy đủ trong `/etc/fstab`, có thể dùng:

```bash
sudo mount /data
```

và `mount` lấy phần thông tin còn lại từ fstab. Red Hat documentation cũng mô tả workflow này. :chatgpt-content-reference{index="21"}

---

# 49. Trước mount phải inspect

Không bao giờ làm:

```bash
sudo mount /dev/sdb1 /data
```

mà không biết:

```text
/dev/sdb1 chứa gì?

Có filesystem không?

Có data quan trọng không?

/data đã có mount gì chưa?

/data có dữ liệu bên dưới không?

Device có đang được dùng ở chỗ khác không?
```

Inspection trước:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINTS
findmnt /data
sudo blkid /dev/sdb1
```

---

# 50. `mkfs` là destructive

Nếu bạn nhớ một cảnh báo của Module 2, hãy nhớ cái này:

```bash
sudo mkfs.ext4 /dev/sdb1
```

không phải lệnh "mount filesystem".

Nó **tạo filesystem mới**.

`mke2fs` chính thức được dùng để tạo ext2/ext3/ext4 filesystem trên device/file. :chatgpt-content-reference{index="22"}

Chạy `mkfs` lên device có data cần giữ có thể phá hủy metadata/filesystem hiện có.

Do đó:

> Không chạy `mkfs` trên production device chỉ vì mount báo lỗi.

---

# 51. Một lỗi fresher rất nguy hiểm

Ticket:

```text
Cannot mount /dev/sdb1.
```

Fresher nghĩ:

```bash
sudo mkfs.ext4 /dev/sdb1
```

Đây có thể biến một incident mount/configuration nhỏ thành **data-loss incident**.

Correct workflow:

```text
inspect device
inspect existing filesystem signature
inspect mount state
inspect expected architecture
collect error
do not create a new filesystem unless explicitly provisioning a new approved empty device
```

---

# 52. `umount`

Command là:

```bash
sudo umount /data
```

Lưu ý spelling:

```text
umount
```

không phải:

```text
unmount
```

`umount` detach filesystem khỏi directory hierarchy. :chatgpt-content-reference{index="23"}

---

# 53. Unmount cũng là risky

Nếu application đang dùng `/data`:

```text
Java process
    ↓
open file
    ↓
/data/database-file
```

Unmount có thể:

```text
fail because filesystem is busy
```

hoặc nếu forced/lazy techniques bị lạm dụng có thể dẫn tới operational consequences rất nghiêm trọng.

Do đó trước unmount phải hiểu:

```text
Who uses this filesystem?

What service depends on it?

Can application stop safely?

What is impact?

How do we recover?
```

---

# 54. “target is busy”

Bạn có thể chạy:

```bash
sudo umount /data
```

và nhận:

```text
target is busy
```

Đó không phải lý do để ngay lập tức:

```text
force unmount
```

Có thể:

```text
your shell is currently inside /data
another process has open files there
a service uses it
nested mount exists
```

Safe first step:

```bash
cd /
```

và inspect consumers.

---

# 55. Lazy/force unmount awareness

`umount` có options cho force/lazy behavior, nhưng với fresher:

> Không dùng chúng như default fix cho `target is busy`.

`umount(8)` có rất nhiều semantics phụ thuộc filesystem và mount namespace. :chatgpt-content-reference{index="24"}

Day 5 objective là biết:

```text
unmount changes filesystem accessibility
```

và phải bảo vệ availability/data integrity.

---

# 56. Một filesystem có thể tồn tại nhưng chưa mounted

Ví dụ:

```bash
lsblk -f
```

shows:

```text
sdb1 ext4 UUID=abc...
```

nhưng `MOUNTPOINTS` trống.

Interpretation:

```text
filesystem exists
BUT
not currently mounted in this namespace
```

Application expecting:

```text
/data
```

không thể sử dụng filesystem đó qua `/data` cho tới khi mount đúng.

---

# 57. Một mount point có thể tồn tại nhưng không mounted

```bash
ls -ld /data
```

shows:

```text
drwxr-xr-x ...
```

Điều này chỉ chứng minh:

```text
/data directory exists
```

Không chứng minh:

```text
expected data filesystem mounted
```

Check:

```bash
findmnt /data
```

hoặc:

```bash
findmnt --target /data
```

---

# 58. `mountpoint` command awareness

**Supplementary**

Một số systems có:

```bash
mountpoint /data
```

Nếu `/data` là mount point, exit status success.

Nếu không, failure.

Nhưng `findmnt` cho nhiều context hơn:

```bash
findmnt /data
```

---

# 59. Root filesystem

Filesystem mounted tại:

```text
/
```

gọi là root filesystem.

Ví dụ:

```bash
findmnt /
```

```text
TARGET SOURCE     FSTYPE OPTIONS
/      /dev/sda2  xfs    rw,...
```

Nếu root filesystem đầy:

```text
/
```

impact có thể rất rộng:

```text
logs cannot be written
packages fail
temporary files fail
services fail
PID/state files fail
application writes fail
SSH/session behavior may degrade
```

Root capacity là health check đặc biệt quan trọng.

---

# 60. `/var` thường quan trọng cho server workloads

Tùy system design, `/var` chứa các loại variable data như:

```text
logs
package data
spool
caches
application state
database/application directories
```

Có hệ thống:

```text
/var
```

nằm chung root filesystem.

Có hệ thống tách:

```text
/var → separate filesystem
```

Vì vậy không bao giờ giả định.

Hỏi:

```bash
findmnt --target /var
df -hT /var
```

---

# 61. `/tmp` cũng không thể đoán

`/tmp` có thể:

```text
directory on root filesystem
```

hoặc:

```text
tmpfs
```

hoặc filesystem riêng.

Check:

```bash
findmnt --target /tmp
```

Không giả định.

---

# 62. `tmpfs` awareness

**Supplementary**

Bạn có thể thấy:

```text
tmpfs
```

trong `df` hoặc `findmnt`.

`tmpfs` là memory-backed filesystem, không phải một normal disk partition.

Ví dụ:

```text
tmpfs on /run
```

Do đó:

> Không phải mọi dòng trong `df` tương ứng physical disk.

Một trong các lý do phải phân biệt:

```text
filesystem
```

với:

```text
disk
```

---

# 63. LVM awareness

**Supplementary**

Bạn có thể thấy:

```text
/dev/mapper/rhel-root
/dev/mapper/vg01-lvdata
```

thay vì:

```text
/dev/sda2
```

Mental model đơn giản:

```text
physical block device
       ↓
partition/PV
       ↓
LVM volume group
       ↓
logical volume
       ↓
filesystem
       ↓
mount point
```

Day 5 không yêu cầu quản trị LVM, nhưng bạn phải không bị hoang mang khi `df` hiển thị `/dev/mapper/...`.

---

# 64. `lsblk` có thể giúp nhìn LVM tree

Ví dụ:

```text
sda
└─sda2
  ├─vg-root
  │ └─ /
  └─vg-var
    └─ /var
```

Bạn không cần học `pvcreate`, `vgcreate`, `lvcreate` hôm nay.

Chỉ cần hiểu:

```text
filesystem source may be a logical volume,
not directly a disk partition
```

---

# 65. Disk capacity vs filesystem capacity

Giả sử:

```text
/dev/sdb = 100 GiB
```

nhưng:

```text
/dev/sdb1 = 50 GiB
```

filesystem trên `/dev/sdb1`:

```text
50 GiB
```

Thì:

```text
disk has 100 GiB
```

không có nghĩa:

```text
/data filesystem has 100 GiB usable
```

Có thể còn unpartitioned/unallocated space.

Capacity phải đo đúng layer.

---

# 66. Cloud disk enlarged nhưng filesystem chưa enlarged

Đây sẽ đặc biệt quan trọng ở AWS Day 10.

Có thể infrastructure team resize volume:

```text
100G → 150G
```

nhưng Linux filesystem vẫn:

```text
100G
```

Bởi resize thường có nhiều layers:

```text
cloud/block device
     ↓
partition
     ↓
LVM (if used)
     ↓
filesystem
```

Tăng size layer trên không tự động guarantee layer dưới đã grow trong mọi workflow.

Hôm nay chỉ cần awareness; Day 10 sẽ xử lý EBS cụ thể.

---

# 67. `df` 100% có luôn thật sự zero bytes không?

Không nhất thiết theo cách đơn giản bạn nghĩ.

Filesystem accounting có:

```text
reserved blocks
metadata
allocation behavior
rounding
```

Ví dụ ext-family filesystems có thể reserve một phần blocks cho privileged use tùy configuration.

Vì vậy:

```text
Use%=100%
```

là critical signal, nhưng đừng cố tính toán bytes theo mắt và kết luận metadata chi tiết mà không hiểu filesystem.

Operational response vẫn là:

```text
treat as capacity incident
identify consumer
free/extend safely
validate
```

---

# 68. `df` và `du` không khớp

Một tình huống kinh điển:

```bash
df -h /var
```

says:

```text
90G used
```

Nhưng:

```bash
sudo du -xsh /var
```

says:

```text
30G
```

Tại sao?

Một nguyên nhân nổi tiếng là:

```text
file đã bị delete khỏi directory tree
BUT
process vẫn giữ file descriptor open
```

---

# 69. Deleted-but-open file

Unix semantics cho phép:

```text
process opens file
       ↓
file is deleted/unlinked
       ↓
filename disappears
       ↓
process still holds open file
       ↓
underlying storage remains allocated
```

Cho tới khi last reference/file descriptor được release.

Do đó:

```text
du
```

không còn thấy file qua directory tree.

Nhưng:

```text
df
```

vẫn thấy filesystem blocks đang allocated.

Classic example:

```text
application.log
```

được xóa trong lúc Java process vẫn đang ghi vào open descriptor.

---

# 70. L1 response với deleted-open file

Không làm:

```text
kill -9 Java immediately
```

chỉ vì nghi ngờ.

Workflow:

```text
collect evidence
identify owning process
understand service impact
use approved log rotation/restart procedure
preserve necessary evidence
validate capacity afterward
```

Một restart an toàn có thể release descriptor, nhưng phải trong scope/change policy.

---

# 71. Đừng xóa log đang dùng tùy tiện

Bad emergency behavior:

```bash
sudo rm /var/log/application.log
```

Nếu process vẫn giữ file open, bạn có thể:

```text
lose visible pathname
but NOT reclaim storage immediately
```

Đồng thời mất troubleshooting evidence.

Better:

```text
use approved log rotation/retention procedure
identify process
understand logging behavior
```

Day 6 sẽ học logs sâu hơn.

---

# 72. Capacity incident cũng có thể do mount mất

Hãy so sánh:

Expected:

```text
/data → /dev/sdb1 500G
```

Actual:

```text
/data is just a directory on /
```

Application tạo:

```text
/data/bigfile
```

`df -h /data`:

```text
Filesystem /dev/sda2
Mounted on /
```

Đây là clue mạnh:

```text
/data is not on expected data filesystem
```

Một command cực mạnh:

```bash
findmnt --target /data
```

---

# 73. Wrong mount point

Có thể expected:

```text
/dev/sdb1 → /data
```

nhưng actual:

```text
/dev/sdb1 → /mnt/data
```

Application vẫn ghi:

```text
/data
```

Đây không phải "disk full".

Đây là:

```text
mount/configuration mismatch
```

---

# 74. Wrong device mounted

Expected:

```text
/data → UUID=A
```

Actual:

```text
/data → UUID=B
```

Mount tồn tại.

Path tồn tại.

Filesystem không full.

Nhưng data sai/missing.

Điều này nguy hiểm hơn missing mount vì superficially everything looks normal.

Check:

```bash
findmnt -no SOURCE,FSTYPE,OPTIONS /data
lsblk -f
```

Compare source/UUID với runbook/configuration.

---

# 75. Filesystem type mismatch

Expected:

```text
ext4
```

Actual block device:

```text
XFS
```

hoặc vice versa.

Đừng chạy:

```bash
sudo mount -t ext4 ...
```

mù quáng.

Check:

```bash
sudo blkid DEVICE
lsblk -f
```

Sử dụng actual known filesystem type/configuration.

---

# 76. Filesystem creation vs filesystem mounting

Hai operation hoàn toàn khác.

```bash
mkfs.ext4 DEVICE
```

means:

```text
create ext4 filesystem
```

Trong khi:

```bash
mount DEVICE TARGET
```

means:

```text
attach existing filesystem to tree
```

Không được trộn.

Đây là một trong những điểm Day 5 quan trọng nhất.

---

# 77. Journaling awareness

**Supplementary**

Filesystems như ext4/XFS có journaling mechanisms để hỗ trợ consistency/recovery sau failures.

Nhưng:

```text
journaling
```

không phải:

```text
backup
```

và không bảo vệ khỏi:

```text
accidental deletion
wrong mkfs
application corruption
malicious changes
```

Backup sẽ học sâu hơn Day 6; snapshots/recovery sẽ quay lại ở AWS Day 10.

---

# 78. Capacity health threshold không có một con số universal

Một sai lầm:

> "80% luôn luôn critical."

Không có một threshold duy nhất đúng cho tất cả systems.

Ví dụ:

```text
80% of 1 TB = 200 GB free
```

khác hoàn toàn:

```text
80% of 5 GB = 1 GB free
```

Cũng phải xét:

```text
growth rate
workload pattern
log rotation
peak periods
recovery time
storage expansion lead time
```

Day 5 cần biết kiểm tra capacity; alert threshold cụ thể sẽ phụ thuộc operational policy.

---

# 79. Free percentage thôi chưa đủ

Ví dụ filesystem:

```text
Size:   10T
Used:    9T
Avail:   1T
Use%:   90%
```

Có thể vẫn có 1 TB.

Một filesystem khác:

```text
Size:   10G
Used:    9G
Avail:   1G
Use%:   90%
```

Cùng percentage nhưng absolute free space rất khác.

Health check tốt record cả:

```text
size
used
available
percentage
```

---

# 80. Growth rate

**Supplementary Operations thinking**

Ví dụ:

```text
Free space now: 10 GB
Growth: 1 GB/day
```

Runway:

```text
≈ 10 days
```

Nếu:

```text
Growth: 5 GB/hour
```

thì tình hình hoàn toàn khác.

Do đó production capacity management không dừng ở:

```text
df -h
```

Mà còn trend monitoring.

Prometheus/Grafana Day 17 sẽ biến logic này thành metrics/alerts.

---

# 81. Filesystem health ≠ application health

Filesystem:

```text
30% used
rw
mounted correctly
```

nhưng application vẫn có thể fail vì:

```text
permissions
wrong path
config error
DB failure
network
service state
```

Tương tự networking:

```text
port reachable
```

không đồng nghĩa app healthy.

Storage cũng phải evidence-driven.

---

# 82. Storage troubleshooting framework

Khi gặp symptom:

```text
Application cannot write /data/output/result.txt
```

Hãy phân lớp:

```text
Path exists?
      ↓
Which filesystem contains path?
      ↓
Expected mount present?
      ↓
Correct source filesystem?
      ↓
Filesystem mounted rw?
      ↓
Byte capacity available?
      ↓
Inodes available?
      ↓
Permissions/ownership?
      ↓
Application/process-specific issue?
```

Đây là storage equivalent của network troubleshooting flow ở Module 1.

---

# 83. Bước 1 — xác định exact path

Bad ticket:

```text
Disk full.
```

Good ticket:

```text
At 14:03, Java service failed to create
/var/lib/orders/export/job-123.csv
with "No space left on device".
```

Bây giờ bạn có:

```text
timestamp
process context
operation
exact path
exact error
```

---

# 84. Bước 2 — path thuộc filesystem nào?

```bash
findmnt --target /var/lib/orders/export
```

và:

```bash
df -hT /var/lib/orders/export
```

Nếu output:

```text
Filesystem: /dev/mapper/vg-var
Mounted: /var
```

bạn không còn troubleshoot generic “disk”.

Bạn đang troubleshoot:

```text
/var filesystem
```

---

# 85. Bước 3 — byte capacity

```bash
df -h /var/lib/orders/export
```

Nếu:

```text
Use% 100%
```

byte/block exhaustion strong candidate.

---

# 86. Bước 4 — inode capacity

```bash
df -i /var/lib/orders/export
```

Nếu:

```text
IUse% 100%
```

inode exhaustion.

Nếu cả hai bình thường:

```text
look elsewhere
```

---

# 87. Bước 5 — mount/options

```bash
findmnt -no TARGET,SOURCE,FSTYPE,OPTIONS \
  --target /var/lib/orders/export
```

Nếu:

```text
ro
```

write failure có thể do read-only mount.

Nếu source filesystem không đúng:

```text
mount issue
```

---

# 88. Bước 6 — locate storage consumer

Nếu block capacity full:

```bash
sudo du -xhd1 /var
```

Sau đó drill xuống largest branch.

Ví dụ:

```text
/var/log      15G
/var/lib      10G
/var/cache     1G
```

Nếu `/var/log` abnormal, tiếp tục:

```bash
sudo du -xhd1 /var/log
```

Không xóa trước khi hiểu ownership/retention requirements.

---

# 89. Disk full ≠ “delete something”

Operations phải xác định:

```text
what data
who owns it
is it safe to remove
retention requirements
is it backup data
database data
audit/security logs
application evidence
can it be regenerated
```

Ví dụ:

```text
/var/lib/postgresql
```

không phải nơi bạn xóa files thủ công để free disk.

---

# 90. PostgreSQL warning

Nếu database directory đầy:

```text
/var/lib/postgresql
```

không tự ý:

```bash
rm ...
```

Database storage files không phải generic cache.

Day 15–16 sẽ học PostgreSQL operations.

Ở Day 5, boundary là:

```text
identify capacity issue
preserve evidence
do not delete unknown DB files
escalate appropriately
```

---

# 91. Java/Tomcat warning

Tương tự:

```text
Tomcat logs
application uploads
temp files
deployment artifacts
```

phải phân biệt.

Không phải file lớn nào cũng disposable.

Tương lai ở Day 12–14 bạn sẽ biết:

```text
which paths are binaries
config
logs
deployments
application data
```

---

# 92. Root filesystem 100% — ưu tiên gì?

Mental model:

```text
1. Preserve access.
2. Collect quick evidence.
3. Identify biggest safe suspect.
4. Understand service ownership.
5. Use approved cleanup/rotation.
6. Validate free blocks + inodes.
7. Confirm services.
8. Document root cause and prevention.
```

Đừng chạy dozens of expensive scans nếu system đang critically degraded.

---

# 93. Mount failure scenario

Expected:

```text
/data → UUID=ABC
```

After reboot:

```bash
findmnt /data
```

returns nothing.

Check:

```bash
lsblk -f
```

shows filesystem exists.

Check:

```bash
grep -v '^[[:space:]]*#' /etc/fstab
```

Potential causes:

```text
wrong UUID
wrong mountpoint
filesystem/device unavailable
bad filesystem type/options
timing/dependency
```

Không chạy `mkfs`.

---

# 94. Manual mount as diagnostic/remediation

Nếu runbook confirms source/target and data safety:

```bash
sudo mount /data
```

nếu `/data` có valid fstab entry.

Hoặc:

```bash
sudo mount UUID=<approved-uuid> /data
```

trong controlled lab.

Sau đó:

```bash
findmnt /data
df -hT /data
```

và validate expected data.

---

# 95. Sau mount phải kiểm tra nội dung

Mount command exit success chưa đủ.

Check:

```bash
findmnt /data
```

rồi:

```bash
ls -la /data
```

và application-specific validation.

Ví dụ expected marker:

```text
/data/orders
/data/archive
```

Nếu filesystem source đúng nhưng content unexpected, stop and investigate.

---

# 96. Health check: block devices

Recommended read-only evidence:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINTS
```

Expected invariant:

```text
all expected storage devices/filesystems are visible
and mapped to expected mount points
```

Không học thuộc device name từ lab.

---

# 97. Health check: mounted filesystems

```bash
findmnt
```

Hoặc focused:

```bash
findmnt /
findmnt /data
```

Expected invariant:

```text
critical filesystems are mounted from expected sources
at expected targets
with expected filesystem types/options
```

---

# 98. Health check: byte capacity

```bash
df -hT /
df -hT /var
df -hT /data
```

Nếu `/var` cùng filesystem với `/`, outputs có thể trùng.

Không vấn đề.

Điều quan trọng là biết path → filesystem mapping.

---

# 99. Health check: inode capacity

```bash
df -i /
df -i /var
df -i /data
```

Expected invariant:

```text
sufficient inode capacity remains for workload
```

Không dùng only percentage threshold một cách máy móc.

---

# 100. `df` chỉ nhìn mounted filesystems

Một unmounted filesystem:

```text
/dev/sdb1
```

không được `df` bình thường report như một active mounted filesystem.

Đây là lý do:

```text
lsblk/blkid
```

và:

```text
df
```

trả lời các câu hỏi khác nhau. GNU `df` documentation nói rõ nó báo filesystem space của mounted filesystems và không cung cấp free-space information cho unmounted filesystem theo cách portable. :chatgpt-content-reference{index="25"}

---

# 101. Guided Lab — tạo filesystem an toàn bằng loop device

Đây là lab rất tốt vì không cần phá real disk.

**Cảnh báo:** các lệnh `losetup`, `mkfs`, `mount`, `umount` thay đổi storage state. Chỉ chạy trên **lab VM**, không dùng server production. Luôn xác nhận chính xác loop device trước khi `mkfs`.

Chúng ta sẽ tạo:

```text
regular file
     ↓
loop block device
     ↓
ext4 filesystem
     ↓
mount
     ↓
/mnt/day5-storage
```

`losetup` được thiết kế để associate regular file với loop block device; upstream docs cũng đưa chính mô hình này làm ví dụ. :chatgpt-content-reference{index="26"}

---

# 102. Lab preparation

Tạo directory:

```bash
mkdir -p ~/day5-storage-lab
cd ~/day5-storage-lab
pwd
```

Expected:

```text
/home/<user>/day5-storage-lab
```

---

# 103. Tạo backing file

```bash
fallocate -l 256M day5-disk.img
```

`fallocate` preallocates/deallocates file space và thường nhanh hơn việc ghi toàn bộ zero blocks trên filesystems hỗ trợ operation đó. :chatgpt-content-reference{index="27"}

Verify:

```bash
ls -lh day5-disk.img
du -h day5-disk.img
```

---

# 104. Associate loop device

```bash
sudo losetup --find --show ~/day5-storage-lab/day5-disk.img
```

Example output:

```text
/dev/loop0
```

**Đây chỉ là example.**

Máy bạn có thể trả:

```text
/dev/loop3
```

Record actual value.

Ví dụ shell:

```bash
LOOPDEV=$(sudo losetup --find --show \
  ~/day5-storage-lab/day5-disk.img)

echo "$LOOPDEV"
```

Upstream `losetup` cảnh báo việc tạo multiple loop devices trỏ cùng backing file có thể gây corruption/data loss; với lab nên tránh reuse/chồng chéo không kiểm soát. :chatgpt-content-reference{index="28"}

---

# 105. Verify block device

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS "$LOOPDEV"
```

Expected invariant:

```text
loop device exists
size ≈ 256M
no filesystem yet
not mounted
```

---

# 106. Tạo filesystem

**Kiểm tra lại `$LOOPDEV` trước.**

```bash
echo "$LOOPDEV"
```

Nó phải là loop device vừa tạo cho lab.

Sau đó:

```bash
sudo mkfs.ext4 "$LOOPDEV"
```

Đây là destructive operation đối với contents trên target.

Chúng ta chỉ làm vì loop device này được tạo riêng cho lab.

`mke2fs`/`mkfs.ext4` tạo ext-family filesystem trên target device. :chatgpt-content-reference{index="29"}

---

# 107. Inspect filesystem

```bash
lsblk -f "$LOOPDEV"
```

và:

```bash
sudo blkid "$LOOPDEV"
```

Expected invariant:

```text
FSTYPE = ext4
UUID exists
```

Record UUID.

---

# 108. Create mount point

```bash
sudo mkdir -p /mnt/day5-storage
```

Inspect trước:

```bash
findmnt /mnt/day5-storage
```

Expected:

```text
no filesystem mounted there
```

---

# 109. Mount

```bash
sudo mount "$LOOPDEV" /mnt/day5-storage
```

Verify:

```bash
findmnt /mnt/day5-storage
```

Expected invariant:

```text
loop device
→ ext4
→ /mnt/day5-storage
→ rw
```

---

# 110. Verify with `lsblk`

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINTS "$LOOPDEV"
```

Bây giờ `MOUNTPOINTS` phải bao gồm:

```text
/mnt/day5-storage
```

---

# 111. Verify capacity

```bash
df -hT /mnt/day5-storage
```

Expected:

```text
ext4
≈ 256M total minus filesystem overhead
mostly free
```

Không mong exactly `256M usable`.

Filesystem metadata chiếm một phần capacity.

---

# 112. Verify inode

```bash
df -i /mnt/day5-storage
```

Record:

```text
Inodes
IUsed
IFree
IUse%
```

---

# 113. Write test

```bash
echo "DAY5 STORAGE OK" | \
  sudo tee /mnt/day5-storage/health.txt
```

Read:

```bash
cat /mnt/day5-storage/health.txt
```

Expected:

```text
DAY5 STORAGE OK
```

Bây giờ bạn đã prove:

```text
device exists
filesystem exists
filesystem mounted
filesystem rw
write succeeds
read succeeds
```

---

# 114. `df` vs `du` trong lab

```bash
df -h /mnt/day5-storage
```

Sau đó:

```bash
sudo du -sh /mnt/day5-storage
```

Observe:

```text
df = filesystem-level view
du = visible files/directories usage
```

Hai numbers không cần giống nhau hoàn toàn.

---

# 115. Failure Injection 1 — capacity pressure

**Chỉ trong loop filesystem của lab.**

Check:

```bash
df -h /mnt/day5-storage
```

Tạo large allocated file:

```bash
sudo fallocate -l 180M \
  /mnt/day5-storage/fill.bin
```

Check lại:

```bash
df -h /mnt/day5-storage
```

và:

```bash
sudo du -sh /mnt/day5-storage
```

Bạn sẽ thấy filesystem usage tăng mạnh.

---

# 116. Tiếp tục tới gần-full nhưng đừng ép host root filesystem

Vì đây là isolated loop filesystem, bạn có thể thử thêm file nhỏ dần:

```bash
sudo fallocate -l 40M \
  /mnt/day5-storage/fill2.bin
```

Nếu fail:

```text
No space left on device
```

đó là expected failure injection.

Ngay lập tức collect:

```bash
df -h /mnt/day5-storage
df -i /mnt/day5-storage
```

---

# 117. Phân tích failure

Nếu:

```text
Use% ≈ 100%
IUse% low
```

Root cause:

```text
filesystem data/block capacity exhausted
```

Không phải inode exhaustion.

---

# 118. Recovery

Xác nhận file nào là lab filler:

```bash
ls -lh /mnt/day5-storage
```

Sau đó:

```bash
sudo rm /mnt/day5-storage/fill.bin
sudo rm -f /mnt/day5-storage/fill2.bin
```

Validate:

```bash
df -h /mnt/day5-storage
```

Expected:

```text
capacity reclaimed
```

---

# 119. Validate application-style write

```bash
echo "RECOVERED" | \
  sudo tee /mnt/day5-storage/recovery-check.txt
```

Nếu success:

```text
write capability restored
```

Remediation chưa hoàn thành cho tới khi workload operation được validated.

---

# 120. Failure Injection 2 — missing mount

Đầu tiên inspect:

```bash
findmnt /mnt/day5-storage
```

Sau đó unmount:

```bash
sudo umount /mnt/day5-storage
```

Verify:

```bash
findmnt /mnt/day5-storage
```

Expected:

```text
no active mount
```

Nhưng:

```bash
ls -ld /mnt/day5-storage
```

directory vẫn tồn tại.

Đây chính là lesson:

```text
directory exists
≠
expected filesystem mounted
```

---

# 121. Check path ownership after unmount

```bash
df -hT /mnt/day5-storage
```

Bạn có thể thấy giờ path thuộc:

```text
/
```

hoặc filesystem chứa `/mnt`.

Điều này mô phỏng production incident:

```text
expected /data mount disappears
application continues writing to underlying filesystem
```

---

# 122. Mount lại

```bash
sudo mount "$LOOPDEV" /mnt/day5-storage
```

Verify:

```bash
findmnt /mnt/day5-storage
df -hT /mnt/day5-storage
cat /mnt/day5-storage/health.txt
```

Expected:

```text
original filesystem is back
original lab files visible
```

---

# 123. Quan sát mount che underlying directory

Để hiểu concept rõ hơn, trong lab có thể:

Unmount:

```bash
sudo umount /mnt/day5-storage
```

Tạo file **trên underlying filesystem**:

```bash
echo "UNDERLYING" | \
  sudo tee /mnt/day5-storage/underlying.txt
```

Bây giờ mount lại:

```bash
sudo mount "$LOOPDEV" /mnt/day5-storage
```

Check:

```bash
ls /mnt/day5-storage
```

Bạn sẽ không thấy:

```text
underlying.txt
```

vì filesystem được mount đang che underlying directory contents.

---

# 124. Unmount lại để chứng minh file chưa mất

```bash
sudo umount /mnt/day5-storage
ls /mnt/day5-storage
```

Bây giờ:

```text
underlying.txt
```

xuất hiện lại.

Đây là behavior rất quan trọng trong storage incidents.

Red Hat documentation cũng mô tả rằng original directory content không accessible qua path đó trong khi filesystem khác đang mounted lên nó. :chatgpt-content-reference{index="30"}

---

# 125. Cleanup file underlying

```bash
sudo rm /mnt/day5-storage/underlying.txt
```

Mount lại nếu cần:

```bash
sudo mount "$LOOPDEV" /mnt/day5-storage
```

---

# 126. Persistent mount lab với `/etc/fstab` — optional nhưng rất giá trị

**Supplementary / elevated risk. Chỉ lab VM.**

Lấy UUID:

```bash
sudo blkid "$LOOPDEV"
```

Ví dụ:

```text
UUID="abc-def-..."
```

Backup fstab:

```bash
sudo cp -a /etc/fstab \
  /etc/fstab.day5.backup
```

Add entry carefully:

```text
UUID=<YOUR-UUID> /mnt/day5-storage ext4 defaults 0 2
```

**Không copy `<YOUR-UUID>` literally.**

---

# 127. Validate fstab

Trước reboot:

```bash
sudo findmnt --verify
```

Sau đó, nếu lab workflow cho phép:

```bash
sudo umount /mnt/day5-storage
sudo mount /mnt/day5-storage
```

Verify:

```bash
findmnt /mnt/day5-storage
```

Nếu systemd environment và fstab vừa thay đổi, upstream man page khuyến nghị:

```bash
sudo systemctl daemon-reload
```

sau modification. :chatgpt-content-reference{index="31"}

---

# 128. Fstab rollback

Sau lab:

```bash
sudo cp -a /etc/fstab.day5.backup /etc/fstab
```

Nếu systemd-based:

```bash
sudo systemctl daemon-reload
```

Unmount:

```bash
sudo umount /mnt/day5-storage
```

Validate:

```bash
findmnt /mnt/day5-storage
```

---

# 129. Full lab cleanup

Đảm bảo filesystem unmounted:

```bash
findmnt /mnt/day5-storage
```

Nếu vẫn mounted:

```bash
sudo umount /mnt/day5-storage
```

Detach **chính loop device lab**:

```bash
sudo losetup -d "$LOOPDEV"
```

Verify:

```bash
sudo losetup -l
```

Upstream `losetup` mô tả `-d/--detach` để disassociate backing file với loop device. :chatgpt-content-reference{index="32"}

Sau đó:

```bash
rm ~/day5-storage-lab/day5-disk.img
rmdir ~/day5-storage-lab
sudo rmdir /mnt/day5-storage
```

Chỉ cleanup sau khi xác minh paths và device.

---

# 130. Integrated Day-5 Troubleshooting

Bây giờ chúng ta nối **Module 1 + Module 2**.

Syllabus không yêu cầu hai incident độc lập; nó yêu cầu bạn chẩn đoán simulated **connectivity/storage issue** và tạo health-check record. :chatgpt-content-reference{index="33"}

Tư duy integrated:

```text
User cannot access Java application
                │
                ▼
            DNS/IP?
                │
                ▼
             Route?
                │
                ▼
            TCP port?
                │
                ▼
           Firewall?
                │
                ▼
          Java listener?
                │
                ▼
         HTTP endpoint?
                │
                ▼
      Application response
                │
         HTTP 500 / failure
                │
                ▼
            Storage?
                │
          /var full?
          /data missing?
          inode full?
          read-only?
```

Một user-visible connectivity symptom có thể bắt nguồn từ storage.

---

# 131. Vì sao storage có thể trông giống network issue?

Ví dụ Tomcat/Java app không thể write:

```text
logs
temporary files
uploads
database/cache state
```

Filesystem full có thể làm application:

```text
crash
fail startup
stop accepting requests
return HTTP 500
hang
```

Client chỉ thấy:

```text
Connection refused
```

hoặc:

```text
HTTP 500
```

Nếu bạn chỉ nhìn network, bạn không tìm được root cause.

---

# 132. Integrated Scenario 1

Symptom:

```text
Users cannot access:
http://orders.internal:8080/health
```

DNS:

```bash
getent hosts orders.internal
```

correct.

Ping:

```bash
ping -c 3 orders.internal
```

works.

Server:

```bash
sudo ss -lntp | grep ':8080'
```

No listener.

Service is failed.

Bạn chưa được kết luận:

```text
Java config problem
```

Check server health.

---

# 133. Storage clue

```bash
df -h /
```

Output:

```text
Filesystem Size Used Avail Use%
/dev/sda2    30G   30G    0   100%
```

Now plausible chain:

```text
root filesystem full
       ↓
Java/Tomcat cannot write required runtime/log/temp state
       ↓
service failed
       ↓
no TCP/8080 listener
       ↓
client sees connection failure
```

Network symptom.

Storage root cause.

---

# 134. Root-cause chain phải có evidence

Không chỉ nói:

> "Disk full made app down."

Bạn cần chứng minh:

```text
T1: filesystem reached critical capacity

T2: application/service failure occurred

T3: expected listener absent

T4: capacity safely recovered

T5: service restored through approved procedure

T6: listener returned

T7: HTTP health check passed
```

Đây mới là Operations evidence.

---

# 135. Integrated Scenario 2 — missing data mount

Expected:

```text
/dev/sdb1 → /data
```

Application writes:

```text
/data/uploads
```

After reboot:

```bash
findmnt /data
```

nothing.

But app still writes `/data`.

Root:

```bash
df -h /
```

becomes:

```text
98%
```

Meanwhile `/dev/sdb1`:

```bash
lsblk -f
```

exists but no mountpoint.

Root cause:

```text
expected data filesystem failed to mount;
application wrote into underlying /data directory on root filesystem,
causing root capacity growth
```

Đây là excellent Day-5 exam scenario.

---

# 136. Remediation order cho missing mount

Không mount ngay nếu `/data` hiện chứa newly-written files.

Nếu mount trực tiếp:

```text
new underlying files become hidden
```

Application có thể trông "mất dữ liệu".

Correct reasoning:

```text
stop/prevent further writes according to runbook
preserve current underlying data
identify expected filesystem
assess duplicate/conflicting data
coordinate recovery/migration
mount expected filesystem safely
reconcile data
validate application
```

Đây có thể vượt L1 scope.

---

# 137. L1 boundary rất quan trọng

Nếu bạn phát hiện:

```text
/data expected mount missing

AND

application has written 50 GB into underlying root directory
```

không tự ý:

```text
move files
delete files
mount over them
```

nếu không có runbook/approval.

Vì bạn có thể:

```text
lose current data visibility
overwrite existing data
create duplication
break application consistency
```

Good L1 behavior:

```text
preserve evidence
stop unsafe experimentation
escalate with clear state
```

---

# 138. Integrated Scenario 3 — inode exhaustion

User says:

```text
Application cannot upload files.
```

Network:

```text
DNS OK
TCP/8080 OK
HTTP request reaches app
```

Response:

```text
500 Internal Server Error
```

Server storage:

```bash
df -h /data
```

```text
45% used
```

Looks fine.

But:

```bash
df -i /data
```

```text
100% inode usage
```

Root cause:

```text
filesystem inode exhaustion
```

Not network.

Not bytes.

---

# 139. Integrated Scenario 4 — read-only mount

App endpoint:

```text
GET /health → 503
```

Logs indicate write failure.

```bash
findmnt -no SOURCE,FSTYPE,OPTIONS --target /data
```

Output contains:

```text
ro
```

Now investigate:

```text
Why is filesystem read-only?

Was it intentionally configured?

Was it remounted due to an error condition?

Was maintenance happening?
```

Do not blindly:

```bash
mount -o remount,rw ...
```

Root-cause and integrity matter.

---

# 140. Filesystem unexpectedly read-only is escalation-worthy

Nếu filesystem that should be `rw` suddenly becomes `ro`, possible serious causes can include filesystem/storage errors depending on filesystem/configuration.

As fresher/L1:

```text
preserve errors/evidence
avoid forcing writes
do not run destructive fsck on mounted production filesystems
escalate storage/Linux team
```

Do not treat it as simple permissions issue.

---

# 141. `fsck` awareness

**Supplementary, important safety knowledge.**

`fsck` is filesystem checking/repair tooling.

But:

> Không chạy `fsck` tùy tiện trên mounted production filesystem.

Different filesystem types have different checking/repair tools and rules.

For example:

```text
ext4 → e2fsck/fsck.ext4 family
XFS  → xfs_repair workflows
```

Repair operations are not generic L1 actions.

---

# 142. Do not repair before evidence

Nếu system reports filesystem corruption:

Bad:

```text
run every repair command you find online
```

Good:

```text
identify filesystem type
identify device
identify mount state
preserve logs/errors
confirm backup/recovery state
follow vendor/runbook procedure
escalate
```

Filesystem repair can change metadata and create irreversible consequences.

---

# 143. Storage Troubleshooting Decision Tree

Giả sử:

```text
Application cannot write PATH
```

Mental decision tree:

```text
PATH exists?
   │
   ├── NO → directory/config/mount issue?
   │
   └── YES
        │
        ▼
Which FS contains PATH?
        │
        ▼
Expected filesystem?
   │
   ├── NO → mount/source/path issue
   │
   └── YES
        │
        ▼
Mounted rw?
   │
   ├── NO → read-only/mount issue
   │
   └── YES
        │
        ▼
Free blocks?
   │
   ├── NO → capacity issue
   │
   └── YES
        │
        ▼
Free inodes?
   │
   ├── NO → inode exhaustion
   │
   └── YES
        │
        ▼
Permissions/security/application
```

---

# 144. Command mapping cho Module 2

| Câu hỏi                               | Command                  |
| ------------------------------------- | ------------------------ |
| Có block devices nào?                 | `lsblk`                  |
| Device chứa filesystem type/UUID gì?  | `lsblk -f`, `blkid`      |
| Filesystems đang mount ở đâu?         | `findmnt`                |
| Path cụ thể thuộc filesystem nào?     | `findmnt --target PATH`  |
| Filesystem còn bao nhiêu bytes?       | `df -h PATH`             |
| Filesystem type là gì?                | `df -hT PATH`, `findmnt` |
| Còn inode không?                      | `df -i PATH`             |
| Directory tree nào chiếm nhiều space? | `du`                     |
| Attach filesystem vào tree?           | `mount`                  |
| Detach filesystem?                    | `umount`                 |
| Persistent mount config?              | `/etc/fstab`             |

---

# 145. `lsblk` không chứng minh filesystem mounted usable

Nếu:

```bash
lsblk -f
```

shows:

```text
sdb1 ext4 UUID=...
```

bạn mới biết:

```text
filesystem signature exists
```

Nếu MOUNTPOINTS trống:

```text
not mounted
```

Ngay cả có mountpoint, vẫn cần:

```text
capacity
options
read/write
application validation
```

---

# 146. `findmnt` success không chứng minh capacity

```bash
findmnt /data
```

success.

Nhưng:

```bash
df -h /data
```

có thể:

```text
100%
```

Mount state và capacity là hai dimensions khác nhau.

---

# 147. `df` success không chứng minh correct filesystem

```bash
df -h /data
```

luôn có thể trả filesystem chứa path, kể cả expected `/data` mount đã mất.

Ví dụ expected:

```text
/data → /dev/sdb1
```

nhưng mount missing.

`df -h /data` vẫn có thể trả:

```text
/dev/sda2 mounted on /
```

Đây là lý do phải đọc cả:

```text
Filesystem
Mounted on
```

không chỉ `Use%`.

---

# 148. `du` lớn không chứng minh file được phép xóa

Ví dụ:

```text
/var/lib/postgresql 50G
```

Đây có thể là legitimate production data.

Capacity troubleshooting:

```text
identify
```

không đồng nghĩa:

```text
delete
```

---

# 149. Health-check record Day 5 hoàn chỉnh

Một record tốt có thể có format:

| Category | Check          | Expected                 | Observed | Status    |
| -------- | -------------- | ------------------------ | -------- | --------- |
| Network  | Interface/IP   | approved IP              | ...      | PASS      |
| Network  | DNS            | expected IP              | ...      | PASS      |
| Network  | Route          | valid path               | ...      | PASS      |
| Network  | Listener       | expected port            | ...      | PASS      |
| Network  | HTTP health    | expected status          | ...      | PASS      |
| Storage  | Devices        | expected devices visible | ...      | PASS      |
| Storage  | Mounts         | expected mounts/source   | ...      | PASS      |
| Storage  | FS type        | expected type            | ...      | PASS      |
| Storage  | Capacity       | sufficient bytes         | ...      | PASS/WARN |
| Storage  | Inodes         | sufficient inodes        | ...      | PASS/WARN |
| Storage  | Mount options  | expected `rw` etc.       | ...      | PASS      |
| Overall  | Service health | healthy                  | ...      | PASS      |

---

# 150. Example health-check evidence

```text
Timestamp:
2026-10-01T22:45:00+07:00

Host:
app01

NETWORK

Interface:
ens5 UP 10.20.30.40/24

DNS:
app01.internal → 10.20.30.40

Route:
expected route present

Listener:
java PID 4210
TCP 0.0.0.0:8080 LISTEN

HTTP:
GET /health → HTTP 200

STORAGE

Block devices:
/dev/sda 30G
/dev/sdb 100G

Root:
/dev/sda2 xfs mounted /
61% used
inode capacity healthy

Data:
/dev/sdb1 ext4 mounted /data
72% used
inode capacity healthy
rw

Overall status:
PASS

Notes:
No unapproved configuration changes performed.
```

---

# 151. Example failed health-check

```text
Timestamp:
2026-10-01T23:02:00+07:00

Symptom:
Application upload requests return HTTP 500.

Network:
DNS PASS
Route PASS
TCP/8080 PASS
HTTP endpoint reachable

Storage:
/data maps to /dev/sdb1 ext4
Byte usage: 42%
Inode usage: 100%
Mount mode: rw

Finding:
Filesystem has available byte capacity but no free inode
capacity for new files.

Scope:
Application requests requiring new file creation fail.

Action:
No files deleted.
Evidence collected and escalated to application/Linux
owner for approved cleanup of excessive small-file workload.

Validation required after remediation:
df -i /data
application file-create operation
HTTP functional test.
```

Đây là một health-check record có giá trị thực sự.

---

# 152. Anti-pattern: “Disk 100% → rm -rf logs”

Không được.

Bạn chưa biết:

```text
logs có retention requirement không
file nào safe
service có giữ file open không
root cause growth là gì
```

`rm -rf` là destructive.

Operations tốt:

```text
identify
scope
authorize
remediate minimally
validate
```

---

# 153. Anti-pattern: “Mount failed → mkfs”

Cực kỳ nguy hiểm.

`mount` failure có thể do:

```text
wrong device
wrong filesystem type
wrong options
device unavailable
filesystem corruption
already mounted
permissions
```

`mkfs` tạo filesystem mới và có thể destroy data.

---

# 154. Anti-pattern: “Directory exists → mount is fine”

Sai.

```bash
ls -ld /data
```

chỉ chứng minh directory exists.

Check:

```bash
findmnt /data
```

---

# 155. Anti-pattern: “df has free GB → storage fine”

Sai.

Need:

```bash
df -i PATH
```

vì inode có thể full.

Cũng phải check:

```text
correct filesystem
rw/ro
mount state
permissions
```

---

# 156. Anti-pattern: “df và du khác nhau → command bug”

Không.

Chúng đo các concepts khác nhau.

Potential causes bao gồm:

```text
deleted-but-open files
filesystem metadata/accounting
mount boundaries
sparse allocation differences
```

Phải investigate.

---

# 157. Anti-pattern: unmount production filesystem để test

Nếu:

```text
/data
```

chứa live application data, `umount` là availability change.

Không làm nếu chưa:

```text
understand dependencies
stop workload safely
have approval
have rollback
```

---

# 158. Anti-pattern: sửa `/etc/fstab` rồi reboot thử

Bad process.

Better:

```text
backup
edit carefully
validate syntax/state
perform controlled mount validation
record evidence
only then consider reboot/change window
```

Một fstab error có thể ảnh hưởng boot/mount behavior.

---

# 159. Security: mount options

Mount options có thể là một defense-in-depth control.

Ví dụ một dedicated data filesystem đôi khi được thiết kế với:

```text
nodev
nosuid
noexec
```

tùy workload.

Nhưng không phải lúc nào cũng đúng.

Ví dụ nếu application cần executable helper scripts trên filesystem đó:

```text
noexec
```

có thể phá workload.

Security rule:

> Mount options phải dựa trên required behavior + least privilege, không copy từ Internet.

---

# 160. Security: writable filesystem exposure

Nếu Java service chạy dưới account:

```text
appuser
```

không nên cho account đó write vào toàn filesystem nếu chỉ cần:

```text
/var/lib/myapp/uploads
```

Filesystem mount và Unix permissions cùng tạo boundaries.

Day 3 permissions knowledge nối trực tiếp với Day 5 storage.

---

# 161. Security: root filesystem preservation

Một application chạy sai và có quyền ghi rộng có thể fill:

```text
/
```

Nếu application data được cô lập trên dedicated filesystem:

```text
/data
```

capacity failure có thể có blast radius nhỏ hơn.

Đây là một lý do operational architecture thường tách một số high-growth data ra riêng.

Không phải universal rule, nhưng là useful design concept.

---

# 162. Security: không để credentials trong evidence

Output `lsblk`, `df`, `findmnt` thường không chứa passwords.

Nhưng application paths/file names đôi khi có thể chứa sensitive customer/project information.

Khi đính evidence:

```text
sanitize sensitive filenames/data
do not cat secret files
do not expose keys/tokens
```

---

# 163. Recovery hierarchy

Storage remediation nên nghĩ theo thứ tự:

```text
Can unsafe growth be stopped?
        ↓
Can known disposable data be cleaned safely?
        ↓
Can retention/log rotation correct it?
        ↓
Can filesystem/storage be expanded?
        ↓
Is mount/config correction required?
        ↓
Is filesystem repair required?
        ↓
Is restore/recovery required?
```

Không phải incident nào cũng giải quyết bằng cleanup.

---

# 164. Cleanup vs expansion

Suppose:

```text
/data is 99%
```

Cause A:

```text
temporary cache accidentally grew 500 GB
```

Correct action có thể:

```text
approved cache cleanup
fix retention
```

Cause B:

```text
legitimate business data growth
```

Correct action có thể:

```text
storage expansion
```

Deleting legitimate data không phải capacity management.

---

# 165. L1-safe actions

L1/fresher thường có thể an toàn thực hiện các read-only checks:

```bash
lsblk
lsblk -f
findmnt
df -hT
df -i
du -sh APPROVED_PATH
```

và collect errors/evidence.

State-changing operations như:

```text
mkfs
partition changes
mount/unmount critical filesystem
fstab edits
filesystem repair
storage resize
large data deletion
```

thường cần runbook/approval hoặc escalation.

---

# 166. Khi nào escalate ngay?

Escalate khi gặp:

```text
filesystem corruption indication
unexpected read-only remount
unknown device/filesystem identity
production mount mismatch with possible data divergence
missing storage device
database filesystem full
root filesystem critically full with unknown safe cleanup
need for filesystem repair
need for partition/LVM resize outside runbook
possible data loss
```

Ở đây escalation là kỹ năng vận hành đúng, không phải thiếu kỹ năng.

---

# 167. Escalation package storage

Bad:

```text
Disk problem. Please help.
```

Good:

```text
Host:
app01

Symptom:
Java service cannot create files under /data/uploads.

Expected:
/data should be UUID=abc... ext4, mounted rw.

Observed:
findmnt shows /data is not a separate mount.
df -hT /data shows path currently resides on root filesystem.
lsblk -f shows UUID=abc... exists on /dev/sdb1 but has no mountpoint.
Root filesystem usage is 96%.
Application has created approximately 18G under the underlying
/data directory since reboot.

No mount/delete/move actions performed.

Risk:
Mounting /dev/sdb1 directly would hide currently written
underlying files and may cause data divergence.

Request:
Storage/application owner guidance for data reconciliation and
approved mount recovery.
```

Đây là excellent escalation.

---

# 168. Exam Focus — distinctions phải thuộc

### Disk ≠ filesystem

```text
Disk/block device
can contain partitions/filesystems/other structures.
```

### Partition ≠ filesystem

```text
Partition is a region.
Filesystem may be created inside it.
```

### Filesystem ≠ mount point

```text
Filesystem contains data structures.
Mount point is location in directory tree.
```

### Directory exists ≠ mounted

```text
/data may exist even when expected data filesystem is absent.
```

### `df` ≠ `du`

```text
df → filesystem capacity
du → file/directory usage
```

### Free bytes ≠ capacity healthy

```text
inode may be exhausted.
```

### `lsblk` ≠ `findmnt`

```text
lsblk → block device view
findmnt → mount/filesystem view
```

### Mount success ≠ application healthy

```text
still need rw/capacity/permissions/application validation
```

### `mkfs` ≠ mount

```text
mkfs creates filesystem and may destroy existing data
```

---

# 169. Exam Question

System says:

```bash
df -h /data
```

```text
Filesystem      Size Used Avail Use% Mounted on
/dev/sda2         30G  28G   2G  94% /
```

But expected architecture says:

```text
/data should be /dev/sdb1
```

Most important observation?

Answer:

```text
/data currently resolves to the root filesystem,
not the expected dedicated filesystem.
```

Next:

```bash
findmnt /data
lsblk -f
```

---

# 170. Exam Question

```bash
df -h /data
```

says:

```text
40% used
```

But application gets:

```text
No space left on device
```

Next command?

```bash
df -i /data
```

because inode exhaustion must be checked.

---

# 171. Exam Question

`lsblk -f`:

```text
sdb1 ext4 UUID=1234...
```

Does that prove it is mounted?

No.

Check:

```bash
findmnt
```

or inspect `MOUNTPOINTS`.

---

# 172. Exam Question

`/data` exists.

Does that prove expected filesystem mounted?

No.

```bash
findmnt /data
```

---

# 173. Exam Question

Application cannot write:

```text
Read-only file system
```

Should you run:

```bash
chmod 777 /data
```

No.

Investigate filesystem mount/options and why it is read-only.

---

# 174. Exam Question

Mount command fails against existing production disk.

Should you create ext4?

No.

`mkfs.ext4` creates filesystem and could destroy existing data.

Inspect device/filesystem and escalate if needed. :chatgpt-content-reference{index="34"}

---

# 175. Knowledge Check

Hãy tự trả lời những câu sau trước khi nhìn đáp án.

**Q1.** Khác nhau giữa block device, partition, filesystem và mount point là gì?

**Q2.** `lsblk` chủ yếu trả lời câu hỏi gì?

**Q3.** `findmnt --target /var/lib/app` trả lời câu hỏi gì?

**Q4.** `df -h /data` đo size của directory `/data` đúng hay sai?

**Q5.** `du -sh /data` khác `df -h /data` thế nào?

**Q6.** Vì sao filesystem có thể có GB trống nhưng không tạo được file mới?

**Q7.** Command nào kiểm tra inode?

**Q8.** `/data` directory tồn tại có chứng minh `/dev/sdb1` mounted ở đó không?

**Q9.** Missing mount có thể làm `/` full bằng cách nào?

**Q10.** Vì sao UUID thích hợp cho persistent mount hơn hardcoded `/dev/sdb1` trong nhiều trường hợp?

**Q11.** `mkfs.ext4` và `mount` khác nhau như thế nào?

**Q12.** `ro` và `rw` khác nhau gì?

**Q13.** Vì sao `df` và `du` có thể lệch lớn?

**Q14.** Vì sao `umount` production filesystem là risky?

**Q15.** Khi filesystem unexpected chuyển read-only, fresher nên force remount hay preserve evidence/escalate?

---

# 176. Knowledge Check — đáp án

**A1.**

```text
Block device:
kernel storage endpoint.

Partition:
logical region of a storage device.

Filesystem:
data/metadata structure organizing files.

Mount point:
directory where filesystem is attached into Linux tree.
```

**A2.**

```text
Available block devices and their relationships/properties.
```

**A3.**

```text
Which mounted filesystem contains that path.
```

**A4.**

Sai. Nó báo capacity của filesystem chứa `/data`.

**A5.**

```text
df → whole filesystem allocation/capacity
du → space used by visible files/directories being scanned
```

**A6.**

Có thể hết inode.

**A7.**

```bash
df -i
```

**A8.**

Không.

**A9.**

Application tiếp tục ghi vào underlying `/data` directory nằm trên `/` khi dedicated filesystem không mounted.

**A10.**

Vì kernel device names có thể thay đổi theo device discovery/configuration; UUID nhận diện filesystem ổn định hơn. :chatgpt-content-reference{index="35"}

**A11.**

```text
mkfs → create filesystem
mount → attach existing filesystem into namespace
```

**A12.**

```text
ro → read-only
rw → read-write
```

**A13.**

Chúng đo khác nhau; deleted-open files và các filesystem accounting differences là ví dụ quan trọng.

**A14.**

Nó thay đổi accessibility của live data và có thể ảnh hưởng services/processes.

**A15.**

Preserve evidence và escalate nếu ngoài runbook/scope.

---

# 177. Independent Day-5 Practical Challenge

Đây là bài nên làm sau khi học cả Module 1 và Module 2.

Scenario:

```text
At 09:20 users report:
http://orders.internal:8080/health is unavailable.

Architecture:

client01
   |
   | TCP/8080
   v
app01
   |
   | writes logs/uploads
   v
/data

Expected:
/data should be a dedicated filesystem.
```

Bạn có quyền:

```text
read status
run diagnostics
collect evidence
```

Bạn chưa có quyền:

```text
delete production data
modify firewall
change DNS
change fstab
restart service
mount/unmount production storage
```

Bạn phải sử dụng evidence để xác định failure layer.

---

# 178. Evidence sequence hợp lý

Client-side:

```bash
getent hosts orders.internal

ip route get <resolved-IP>

ping -c 4 <resolved-IP>

curl -v \
  --connect-timeout 3 \
  --max-time 10 \
  http://orders.internal:8080/health
```

Server-side network:

```bash
ip -br addr

sudo ss -lntp

curl -v \
  http://127.0.0.1:8080/health
```

Server-side storage:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINTS

findmnt

findmnt --target /data

df -hT /

df -hT /data

df -i /

df -i /data
```

Nếu capacity suspect:

```bash
sudo du -xhd1 /relevant/filesystem/path
```

chỉ khi scan đó có acceptable operational cost.

---

# 179. Bạn phải đưa ra conclusion ở dạng layer

Không nói:

```text
Server broken.
```

Hãy nói kiểu:

```text
DNS layer: PASS

IP route: PASS

TCP listener: FAIL

Service state: failed

Storage:
root filesystem 100% used

/data expected dedicated mount: absent

Finding:
application data has been written to underlying /data
directory on the root filesystem after expected mount disappeared.

Probable causal chain:
missing mount
→ writes redirected to root filesystem
→ root filesystem exhausted
→ Java service failed
→ port 8080 listener disappeared
→ user experienced connection failure.
```

Đây chính là integrated reasoning mà Day 5 muốn xây dựng.

---

# 180. Master Health-Check Command Set của Day 5

Sau hai module, bạn nên hiểu đầy đủ bộ command này:

```bash
# ---------- NETWORK ----------

ip -br addr

ip route

ip route get <destination>

getent hosts <hostname>

ping -c 4 <destination>

traceroute -n <destination>

ss -lnt

sudo ss -lntp

curl -v \
  --connect-timeout 3 \
  --max-time 10 \
  http://host:port/path


# ---------- STORAGE ----------

lsblk -o NAME,SIZE,TYPE,FSTYPE,UUID,MOUNTPOINTS

findmnt

findmnt --target <path>

df -hT <path>

df -i <path>

du -sh <directory>

sudo du -xhd1 <approved-path>


# ---------- FIREWALL ----------

sudo ufw status verbose

# or

sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

Điều quan trọng hơn commands là biết mỗi command **chứng minh được gì và không chứng minh được gì**.

---

# 181. Day 5 Troubleshooting Mental Model cuối cùng

Hãy ghép networking và storage thành một flow:

```text
USER
 │
 ▼
Hostname
 │
 ▼
DNS
 │
 ▼
IP
 │
 ▼
Route
 │
 ▼
Network path
 │
 ▼
Firewall
 │
 ▼
TCP port
 │
 ▼
Listening process
 │
 ▼
HTTP/TLS
 │
 ▼
Java application
 │
 ├───────────────┐
 │               │
 ▼               ▼
Filesystem     Database / dependency
 │
 ▼
Mount
 │
 ▼
Block device
```

Một lỗi bất kỳ ở dưới có thể bubble up thành:

```text
"Application unavailable"
```

Vì vậy một Operations engineer không troubleshoot bằng symptom label.

Họ troubleshoot bằng **layers + evidence**.

---

# 182. Những invariants bạn phải luôn kiểm tra

Một Linux application server khỏe về network + storage không phải chỉ là:

```text
"ping works"
```

hay:

```text
"df is under 90%"
```

Nó là một tập các invariant:

```text
expected network identity exists
expected DNS resolves correctly
expected route exists
expected port/listener exists
expected firewall access exists
expected application endpoint responds

AND

expected block storage is visible
expected filesystem exists
expected filesystem is mounted at correct target
expected mount source is correct
expected mount options support workload
sufficient byte capacity remains
sufficient inode capacity remains
application can perform expected read/write operation
```

Nếu một invariant fail, bạn có nơi cụ thể để investigate.

---

# 183. Mastery criteria cho Module 2

Bạn **chưa mastery** nếu chỉ biết:

```bash
df -h
```

Bạn đạt mức tốt hơn khi hiểu:

```text
df → filesystem capacity
du → directory/file usage
df -i → inode capacity
lsblk → block device topology
findmnt → mounted filesystem topology
blkid → filesystem/device attributes
```

Bạn đạt mức Operations-ready khi có thể nhìn:

```text
Application: No space left on device
```

và reasoning:

```text
Which path?
→ Which filesystem?
→ Correct mount?
→ Correct source?
→ rw?
→ blocks?
→ inodes?
→ top consumers?
→ deleted-open possibility?
→ safe remediation?
→ validation?
→ escalation?
```

---

# 184. Day 5 hoàn chỉnh: bạn đã học gì?

Sau Module 1 + Module 2, toàn bộ mandatory Concept/Lecture của Day 5 đã được cover:

```text
IP
DNS
ports
curl
ss
ping
traceroute
firewall basics
disks
filesystems
mounts
capacity checks
```

đúng theo syllabus. :chatgpt-content-reference{index="36"}

Module 1 cho bạn chain:

```text
DNS → IP → route → network → port → listener → firewall → HTTP
```

Module 2 cho bạn chain:

```text
path → mount → filesystem → block device
                 ↓
          blocks + inodes
```

Ghép lại:

```text
Client
  ↓
DNS
  ↓
IP/Route
  ↓
Firewall
  ↓
Port/Socket
  ↓
Java process
  ↓
Application
  ↓
Filesystem path
  ↓
Mount
  ↓
Filesystem capacity
  ↓
Block storage
```

Đây chính là nền móng bạn sẽ dùng lại gần như toàn bộ khóa học khi sang **AWS EC2/EBS, Tomcat/Java, PostgreSQL, monitoring và integrated troubleshooting**.

Điểm quan trọng nhất cần ghi nhớ sau Module 2 là:

> **“Disk full”, “mount missing”, “inode full”, “wrong filesystem” và “permission problem” là những failure layer khác nhau. Đừng sửa storage bằng cảm tính; trước tiên phải chứng minh path nằm trên filesystem nào, filesystem đó được mount từ đâu, đang ở trạng thái nào, còn block và inode hay không, rồi mới quyết định remediation hay escalation.**

Và điểm tổng kết quan trọng nhất của toàn **Day 5**:

> **Một incident “application không truy cập được” không mặc định là network issue, và một error “No space left on device” không mặc định là disk bytes đã hết. Troubleshooting đúng là xây dựng chain of evidence từ symptom xuống layer thực sự bị lỗi.**
