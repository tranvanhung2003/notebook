# Day 3 — Linux Users, Groups, Ownership, Permissions, sudo, SSH Keys & Least Privilege

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 3 thuộc phần **Linux Fundamentals** và có phạm vi bắt buộc:

> **Concept/Lecture:** Users, groups, ownership, permissions, sudo, SSH keys, and least-privilege basics.  
> **Assignment/Lab:** Create controlled users/groups, apply permissions, configure SSH key access, and submit verification evidence. 00_MASTER_SYLLABUS_Java_Fresher…

Vì vậy, cuối Day 3 bạn phải có khả năng đi hết chuỗi:

**User/Group → UID/GID → Ownership → Permission → sudo policy → SSH key authentication → Verification → Troubleshooting → Evidence → Rollback.**

### Mandatory — đúng syllabus

| Nội dung        | Phải nắm                                                |
| --------------- | ------------------------------------------------------- |
| Users           | account, UID, home, shell, password/account state       |
| Groups          | primary group, supplementary groups, GID                |
| Ownership       | owner user + owner group                                |
| Permissions     | `r`, `w`, `x`; user/group/other; symbolic/octal         |
| sudo            | privilege elevation, sudoers, `visudo`, least privilege |
| SSH keys        | public/private key, `authorized_keys`, permissions      |
| Least privilege | chỉ cấp quyền cần thiết                                 |
| Lab             | tạo user/group, permission, SSH key, evidence           |

### Supplementary / Beyond the explicit syllabus

Tôi cũng sẽ dạy thêm những thứ rất sát thực tế: `/etc/passwd`, `/etc/shadow`, NSS/`getent`, `umask`, setgid directory, sticky bit, sudo troubleshooting, SSH `StrictModes`, failure injection, rollback và account offboarding. Những phần này không thay thế syllabus mà giúp bạn làm Operations thực tế tốt hơn.

---

# 2. Objectives — Hôm nay phải làm được gì?

Sau Day 3, bạn cần tự làm được những việc sau mà không “đoán lệnh”:

1. Xác định một process/file đang thuộc user/group nào.
2. Tạo user và group có kiểm soát.
3. Phân biệt **UID**, username, **GID**, group name.
4. Phân biệt primary group và supplementary group.
5. Đọc chính xác output như:

```text
-rw-r----- 1 root appops 512 Oct 12 10:00 application.properties
```

6. Tính được permission `640`, `750`, `770`, `600`, `700`.
7. Hiểu chính xác `r/w/x` đối với **file** và **directory**.
8. Dùng `chown`, `chgrp`, `chmod` mà không gây blast radius ngoài ý muốn.
9. Dùng `sudo` thay vì làm việc thường xuyên bằng `root`.
10. Viết một sudo rule nhỏ theo nguyên tắc least privilege.
11. Tạo SSH key pair.
12. Biết private key nằm ở đâu và **không bao giờ copy private key lên server**.
13. Cấu hình `authorized_keys`.
14. Troubleshoot lỗi `Permission denied (publickey)`.
15. Thu thập evidence chứng minh cả **positive test** lẫn **negative test**.

---

# 3. Mental Model quan trọng nhất của Day 3

Hãy nhớ mô hình:

```text
WHO ARE YOU?
    |
    v
User identity
username -> UID
    |
    +---- supplementary GIDs
    |
    v
HOW DID YOU PROVE IT?
Authentication
password / SSH private key / ...
    |
    v
WHAT ARE YOU ALLOWED TO DO?
Authorization
    |
    +---- Unix ownership + mode bits
    +---- sudo policy
    +---- SSH policy
    |
    v
RESOURCE
file / directory / privileged command / remote login
```

Ba khái niệm rất dễ bị trộn:

| Khái niệm      | Câu hỏi                                  |
| -------------- | ---------------------------------------- |
| Identification | Bạn nói bạn là ai?                       |
| Authentication | Bạn chứng minh danh tính đó như thế nào? |
| Authorization  | Danh tính đó được phép làm gì?           |

Ví dụ:

```text
Username: opsalice
Authentication: SSH private key
Authorization:
  - read /srv/day3app/config/app.conf
  - cannot modify it
  - may execute exactly one approved sudo command
```

Authentication **không đồng nghĩa** với authorization.

Bạn SSH login thành công không có nghĩa là bạn được đọc `/etc/shadow`.

---

# 4. Linux User — thực chất Linux nhìn thấy UID

Linux kernel chủ yếu làm việc với **numeric UID**, không phải chuỗi username.

Ví dụ:

```bash
id
```

Có thể trả:

```text
uid=1000(admin) gid=1000(admin) groups=1000(admin),27(sudo)
```

Ở đây:

```text
1000     = UID
admin    = username
1000     = primary GID
admin    = primary group
27       = supplementary GID
sudo     = supplementary group
```

Hai ý rất quan trọng:

```text
username ≠ identity thật ở kernel level

UID = identity quan trọng ở kernel level
```

Nếu một file hiển thị:

```text
-rw------- 1 alice alice 100 file.txt
```

thực chất filesystem lưu numeric UID/GID. `ls` chỉ tra cứu rồi hiển thị tên tương ứng.

---

# 5. `/etc/passwd`

Local users truyền thống được mô tả trong:

```text
/etc/passwd
```

Một record có dạng:

```text
opsalice:x:1001:1001:Operations User:/home/opsalice:/bin/bash
```

Bảy field là:

```text
username:x:UID:GID:GECOS:home:shell
```

Ví dụ:

```text
opsalice
x
1001
1001
Operations User
/home/opsalice
/bin/bash
```

`x` **không phải password**. Trên hệ thống dùng shadow passwords, password hash được lưu trong `/etc/shadow`. `/etc/passwd` thường cần readable để hệ thống ánh xạ UID ↔ username, trong khi `/etc/shadow` phải được bảo vệ khỏi regular users. [man7.org](https://man7.org/linux/man-pages/man5/passwd.5%40%40shadow-utils.html?utm_source=chatgpt.com)

Kiểm tra:

```bash
grep '^opsalice:' /etc/passwd
```

Tuy nhiên trong Operations thực tế, nên biết:

```bash
getent passwd opsalice
```

Tại sao?

Vì user không nhất thiết chỉ đến từ `/etc/passwd`. Hệ thống enterprise có thể dùng NSS/SSSD/LDAP/Identity Management.

`getent` hỏi **configured name service**, nên thường tốt hơn việc chỉ `grep /etc/passwd`.

**Supplementary / Beyond syllabus**, nhưng rất quan trọng trong enterprise.

---

# 6. `/etc/shadow`

Không nên `cat /etc/shadow` chỉ để “xem thử” trên production.

File này chứa password-related information và password aging.

Ví dụ conceptual:

```text
opsalice:$y$...:20738:0:99999:7:::
```

Regular user:

```bash
cat /etc/shadow
```

thường sẽ nhận:

```text
Permission denied
```

Đó là **expected security control**, không phải lỗi.

Man page hiện hành xác nhận `/etc/shadow` chứa password information/aging và không được readable bởi regular users. [man7.org](https://man7.org/linux/man-pages/man5/shadow.5.html?utm_source=chatgpt.com)

---

# 7. Root user

User đặc biệt:

```text
UID 0
```

thường là:

```text
root
```

Kiểm tra:

```bash
id root
```

Ví dụ:

```text
uid=0(root) gid=0(root) groups=0(root)
```

`root` có quyền cực lớn.

Vì thế nguyên tắc Operations không phải:

```text
"Cứ login root cho tiện."
```

mà là:

```text
Normal identity
      |
      | only when required
      v
controlled sudo
      |
      v
privileged operation
```

Red Hat cũng khuyến nghị sử dụng `sudo` để cấp administrative capability thay vì chia sẻ/root-login thường xuyên, vì sudo cho phép granular authorization và logging tốt hơn. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/security_hardening/managing-sudo-access?utm_source=chatgpt.com)

---

# 8. System accounts và human accounts

Bạn sẽ gặp:

```text
root
sshd
systemd-network
postgres
tomcat
prometheus
node_exporter
```

Không phải tất cả đều là người.

Một service thường nên chạy bằng **dedicated service account**.

Sau này Day 12 khi triển khai Tomcat, mô hình có thể là:

```text
Human admin:
alice
    |
    | sudo
    v
administrative action

Application:
Tomcat
    |
    v
tomcat service account
```

Không nên:

```text
Tomcat chạy root
```

nếu không có lý do kỹ thuật bắt buộc.

Đây chính là least privilege.

---

# 9. Groups

Group giúp quản lý quyền cho nhiều users.

Thay vì:

```text
Alice được quyền A
Bob được quyền A
Carol được quyền A
David được quyền A
```

ta tạo:

```text
group = appops
```

sau đó:

```text
Alice ----+
Bob ------+--> appops --> permission
Carol ----+
```

Quản lý dễ hơn rất nhiều.

---

# 10. `/etc/group`

Local groups thường được định nghĩa trong:

```text
/etc/group
```

Format:

```text
group_name:password:GID:user_list
```

Ví dụ:

```text
appops:x:2001:opsalice,opsbob
```

Man page hiện hành định nghĩa `/etc/group` theo format trên. [man7.org](https://man7.org/linux/man-pages/man5/group.5.html?utm_source=chatgpt.com)

Kiểm tra tốt hơn:

```bash
getent group appops
```

---

# 11. Primary group và supplementary groups

Giả sử:

```bash
id alice
```

trả:

```text
uid=1001(alice) gid=1001(alice) groups=1001(alice),2001(appops),2002(monitoring)
```

Ta có:

```text
Primary group:
alice (GID 1001)

Supplementary groups:
appops
monitoring
```

Primary group được lưu trong user record.

Supplementary membership thường nằm trong group database.

---

# 12. Command kiểm tra identity

Các command phải thuộc lòng:

```bash
whoami
```

Hiển thị effective username.

```bash
id
```

Hiển thị UID/GID/groups.

```bash
id alice
```

Kiểm tra user cụ thể.

```bash
groups alice
```

Hiển thị group membership.

```bash
getent passwd alice
```

Tra user.

```bash
getent group appops
```

Tra group.

Ví dụ:

```bash
$ id opsalice
uid=1001(opsalice) gid=1001(opsalice) groups=1001(opsalice),2001(appops)
```

---

# 13. Tạo group

Command chuẩn từ shadow-utils:

```bash
sudo groupadd appops
```

`groupadd` tạo một group account và cập nhật group database. [man7.org](https://www.man7.org/linux/man-pages/man8/groupadd.8.html?utm_source=chatgpt.com)

Kiểm tra ngay:

```bash
getent group appops
```

Đừng làm kiểu:

```bash
sudo groupadd appops
sudo groupadd another
sudo groupadd another2
...
```

rồi cuối cùng mới kiểm tra.

Operations tốt là:

```text
change → validate → next change
```

---

# 14. Tạo user

Portable command phổ biến:

```bash
sudo useradd -m -U -s /bin/bash opsalice
```

Ý nghĩa:

```text
-m    create home directory
-U    create a group with same name as user
-s    set login shell
```

Sau đó:

```bash
getent passwd opsalice
id opsalice
ls -ld /home/opsalice
```

`useradd` có thể cập nhật `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow` và tạo home tùy options/configuration. Hành vi group mặc định còn phụ thuộc distribution/configuration, vì vậy trong lab chúng ta dùng explicit options thay vì đoán. [man7.org](https://man7.org/linux/man-pages/man8/useradd.8.html?utm_source=chatgpt.com)

---

# 15. `adduser` và `useradd` khác nhau

Trên Debian/Ubuntu bạn thường thấy:

```bash
sudo adduser alice
```

`adduser` là frontend thân thiện hơn.

Trong khi:

```bash
useradd
```

là low-level account-management utility phổ biến trong shadow-utils.

RHEL thường dùng trực tiếp `useradd`.

Để lab có tính cross-distribution cao, bài này dùng:

```bash
useradd
groupadd
usermod
```

---

# 16. Thêm user vào supplementary group

Command cực kỳ quan trọng:

```bash
sudo usermod -aG appops opsalice
```

Trong đó:

```text
-a = append
-G = supplementary groups
```

Sau đó:

```bash
id opsalice
```

Expected:

```text
groups=...,appops
```

## Sai lầm cực nguy hiểm

Không nên tùy tiện:

```bash
sudo usermod -G appops alice
```

nếu Alice đã thuộc nhiều supplementary groups.

Không có `-a`, bạn có nguy cơ **thay thế danh sách supplementary groups**.

Nếu đó là admin account, bạn có thể vô tình loại user khỏi `sudo` hoặc `wheel`.

Mental note:

```text
usermod -aG
        ^^
 append supplementary group
```

---

# 17. Group membership mới không phải lúc nào có hiệu lực trong session cũ

Bạn vừa chạy:

```bash
sudo usermod -aG appops alice
```

rồi trong shell của Alice:

```bash
id
```

có thể chưa thấy.

Lý do: supplementary groups thường được thiết lập khi login session được tạo.

Cách rõ ràng nhất:

```text
logout
login lại
```

Sau đó:

```bash
id
```

`newgrp` cũng tồn tại nhưng không nên biến nó thành giải pháp mặc định nếu bạn chưa hiểu session semantics.

---

# 18. Ownership

Mỗi Unix file/directory truyền thống có:

```text
owner user
owner group
```

Ví dụ:

```bash
ls -l app.conf
```

```text
-rw-r----- 1 root appops 128 Oct 12 10:00 app.conf
                ^    ^
                |    |
              owner group
```

Ở đây:

```text
Owner user  = root
Owner group = appops
```

---

# 19. `chown`

Cú pháp:

```bash
chown OWNER FILE
```

Ví dụ:

```bash
sudo chown alice file.txt
```

Owner và group:

```bash
sudo chown alice:appops file.txt
```

Chỉ group:

```bash
sudo chown :appops file.txt
```

GNU `chown` thay đổi owner và/hoặc group ownership theo format này. [man7.org](https://www.man7.org/linux/man-pages/man1/chown.1.html?utm_source=chatgpt.com)

Recursive:

```bash
sudo chown -R alice:appops directory/
```

### Cảnh báo

`-R` là một trong những option mà fresher phải tôn trọng.

Trước:

```bash
sudo chown -R ...
```

hãy làm:

```bash
pwd
ls -ld target
find target -maxdepth 2 -ls
```

Xác nhận target trước.

Một `chown -R` sai vào `/`, `/etc`, application home hoặc mounted storage có thể gây outage lớn.

---

# 20. `chgrp`

Nếu chỉ muốn đổi group:

```bash
sudo chgrp appops app.conf
```

Tương đương conceptual với:

```bash
sudo chown :appops app.conf
```

---

# 21. Unix permissions — ba class

Output:

```text
-rwxr-x---
```

Tách thành:

```text
-   rwx   r-x   ---
^    ^     ^     ^
|    |     |     |
type user group other
```

Có 3 permission class:

```text
u = user/owner
g = group
o = others
```

và:

```text
a = all
```

---

# 22. Ba permission bits

```text
r = read
w = write
x = execute
```

Nhưng ý nghĩa với **file** và **directory** khác nhau rất nhiều.

---

# 23. Permission trên regular file

| Bit | File                                               |
| --- | -------------------------------------------------- |
| `r` | đọc nội dung                                       |
| `w` | sửa/truncate nội dung                              |
| `x` | execute file như program/script nếu format phù hợp |

Ví dụ:

```text
rw-r-----
```

Owner:

```text
read + write
```

Group:

```text
read
```

Others:

```text
nothing
```

---

# 24. Permission trên directory — phần rất hay thi

Đây là phần phải hiểu chứ không học thuộc máy móc.

| Bit | Directory                  |
| --- | -------------------------- |
| `r` | đọc/list directory entries |
| `w` | tạo/xóa/rename entries     |
| `x` | traverse/search directory  |

Ví dụ directory:

```text
drwxr-x---
```

group có:

```text
r-x
```

nên có thể:

```text
list
traverse
access entries nếu entries cho phép
```

Nếu directory thiếu `x`, bạn có thể nhìn thấy tên nhưng không truy cập object bên trong như mong đợi.

Linux cũng yêu cầu search/execute permission trên các directories trong path trước khi file cuối cùng có thể truy cập được. [man7.org](https://www.man7.org/linux/man-pages/man2/access.2.html?utm_source=chatgpt.com)

Ví dụ:

```text
/srv              x
/srv/app          x
/srv/app/config   x
/srv/app/config/db.conf   r
```

Nếu user không có `x` trên:

```text
/srv/app
```

thì dù:

```text
db.conf = 777
```

user vẫn có thể không tới được file.

---

# 25. Một sự thật rất quan trọng: delete phụ thuộc directory

Giả sử:

```text
file:
-r--r--r-- alice alice report.txt

directory:
drwxrwx--- alice appops .
```

Một member của `appops` có directory `w+x`.

Người đó có thể có khả năng:

```text
delete/rename directory entry
```

mặc dù file không writable đối với họ.

Đây là lý do không được suy nghĩ:

> "File readonly thì chắc chắn không xóa được."

File permission và directory permission là hai layer khác nhau.

---

# 26. Permission evaluation

Simplified Unix DAC model:

```text
Is process owner UID == file owner UID?
        |
       yes
        |
     use OWNER bits
        |
       no
        v
Does process belong to file group?
        |
       yes
        |
     use GROUP bits
        |
       no
        v
     use OTHER bits
```

Một subtle point:

Permissions không được đơn giản cộng:

```text
owner + group + other
```

Kernel chọn applicable class.

Ví dụ:

```text
-r--rw---- alice appops file
```

Nếu Alice là owner thì owner bits:

```text
r--
```

được áp dụng.

Không phải:

```text
r-- + rw- = rw-
```

---

# 27. Numeric permissions

Mỗi bit:

```text
r = 4
w = 2
x = 1
```

Cộng theo từng class.

| Permission | Value |
| ---------- | ----: |
| `---`      |     0 |
| `--x`      |     1 |
| `-w-`      |     2 |
| `-wx`      |     3 |
| `r--`      |     4 |
| `r-x`      |     5 |
| `rw-`      |     6 |
| `rwx`      |     7 |

Ví dụ:

```text
750
```

là:

```text
7 = rwx owner
5 = r-x group
0 = --- others
```

=>:

```text
rwxr-x---
```

---

# 28. Các mode phải đọc ngay được

```text
600 = rw-------
700 = rwx------
640 = rw-r-----
644 = rw-r--r--
660 = rw-rw----
750 = rwxr-x---
755 = rwxr-xr-x
770 = rwxrwx---
```

Đặc biệt cho Day 3:

```text
SSH private key       → 600
~/.ssh                → 700
authorized_keys       → 600
shared config file    → thường 640/660 tùy requirement
private directory     → 700
group directory       → 750/770 tùy requirement
```

Đừng hiểu đây là “magic numbers”. Phải chuyển được giữa numeric và symbolic.

---

# 29. `chmod`

Numeric:

```bash
chmod 640 file
```

Symbolic:

```bash
chmod u=rw,g=r,o= file
```

Hai cách tương đương trong ví dụ.

Thêm group write:

```bash
chmod g+w file
```

Bỏ others access:

```bash
chmod o-rwx file
```

GNU/POSIX `chmod` thay đổi file mode bits; chỉ owner hoặc process có privilege phù hợp mới được thay đổi mode. [man7.org](https://www.man7.org/linux/man-pages/man1/chmod.1p.html?utm_source=chatgpt.com)

---

# 30. `chmod -R` — cẩn thận

Không tùy tiện:

```bash
chmod -R 777 /some/application
```

Đây là anti-pattern rất phổ biến.

Bạn vừa có thể biến:

```text
config
private key
script
data
directory
binary
```

thành cùng một mode.

File và directory thường cần permissions khác nhau.

`777` không phải "fix permission".

Nó thường có nghĩa:

> Tôi chưa xác định root cause nên mở quyền cho tất cả.

Trong production đó là dấu hiệu troubleshooting không tốt.

---

# 31. Cách debug permission

Đừng bắt đầu bằng:

```bash
sudo chmod 777 ...
```

Hãy bắt đầu:

```bash
whoami
id
ls -ld /path
ls -l /path/file
```

Sau đó hỏi:

```text
User hiện tại là ai?
UID/GID?
File owner?
File group?
User có thuộc group?
Permissions của từng directory trong path?
Permission của file cuối?
```

Nếu có:

```bash
namei -l /srv/app/config/app.conf
```

đây là công cụ rất hữu ích để xem từng component của path.

Ví dụ:

```text
f: /srv/app/config/app.conf
drwxr-xr-x root root   /
drwxr-xr-x root root   srv
drwxr-x--- root appops app
drwxr-x--- root appops config
-rw-r----- root appops app.conf
```

Bạn có thể nhìn thấy layer nào chặn access.

**Supplementary / Beyond explicit syllabus.**

---

# 32. `umask`

**Supplementary**, nhưng nên học Day 3.

`umask` ảnh hưởng default permission của object mới.

Kiểm tra:

```bash
umask
```

Ví dụ:

```text
0022
```

Conceptually:

Regular file base:

```text
666
```

Directory base:

```text
777
```

với:

```text
umask 022
```

thường tạo:

```text
file      644
directory 755
```

Đừng đơn giản coi `umask` như phép trừ số học trong mọi trường hợp; thực chất nó **mask permission bits**.

---

# 33. setgid directory

**Supplementary / rất hữu ích cho shared directories.**

Nếu:

```bash
chmod 2770 /srv/shared
```

leading `2` bật setgid trên directory.

Khi đó new files/subdirectories thường inherit **directory group**, rất hữu ích cho team share.

Ví dụ:

```text
drwxrws--- root appops /srv/shared
```

Notice:

```text
s
```

ở group execute position.

---

# 34. Sticky bit

Ví dụ điển hình:

```text
/tmp
```

thường:

```text
drwxrwxrwt
```

`t` = sticky bit.

Nó giúp tránh trường hợp user tùy ý xóa file của user khác trong một world-writable shared directory.

**Supplementary.**

---

# 35. Least Privilege — nguyên tắc trung tâm

Least privilege nghĩa là:

> Cấp **chỉ** quyền cần cho nhiệm vụ, cho đúng identity, đúng resource, đúng phạm vi, và ideally đúng thời gian.

Không phải:

```text
"không cấp quyền gì cả"
```

mà là:

```text
"cấp đủ, không thừa"
```

Ví dụ requirement:

> Operator cần đọc config nhưng không được sửa.

Sai:

```bash
chmod 777 app.conf
```

Sai:

```bash
chown operator app.conf
chmod 700 app.conf
```

Tốt hơn:

```text
owner = root
group = appops
mode  = 640
```

```bash
sudo chown root:appops app.conf
sudo chmod 640 app.conf
```

Result:

```text
-rw-r----- root appops app.conf
```

Root:

```text
rw
```

appops:

```text
r
```

others:

```text
none
```

Đây là thiết kế permission.

---

# 36. `su` và `sudo`

Hai thứ này không giống nhau.

`su`:

```bash
su -
```

thường chuyển sang identity khác và có thể tạo shell dưới identity đó.

`sudo`:

```bash
sudo command
```

chạy một command theo sudo policy.

Operations thường ưu tiên:

```bash
sudo approved-command
```

hơn việc mở một root shell dài hạn.

Vì một command có thể audit/control tốt hơn.

---

# 37. `sudo` hoạt động theo mental model nào?

```text
User
 |
 | requests
 v
sudo command
 |
 v
sudo policy (/etc/sudoers + /etc/sudoers.d/)
 |
 +--> allowed? ---- no ---> deny + log
 |
 yes
 |
 v
optional authentication
 |
 v
run command as configured identity
```

Ví dụ:

```bash
sudo id
```

Normal:

```bash
id
```

```text
uid=1001(alice) ...
```

Privileged:

```bash
sudo id
```

có thể:

```text
uid=0(root) gid=0(root) groups=0(root)
```

---

# 38. `/etc/sudoers`

Core sudo configuration:

```text
/etc/sudoers
```

và thường có drop-ins:

```text
/etc/sudoers.d/
```

Red Hat hiện cũng tài liệu hóa cả `/etc/sudoers` và `/etc/sudoers.d/` làm nơi lưu sudo authorizations. Rule có thể giới hạn user/group, host, run-as identity và command cụ thể. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/security_hardening/managing-sudo-access?utm_source=chatgpt.com)

Rule conceptual:

```text
WHO WHERE=(RUN-AS) WHAT
```

Ví dụ:

```text
%opsmaint ALL=(root) /usr/bin/id
```

Nghĩa:

```text
members of opsmaint
on all matched hosts
may run
/usr/bin/id
as root
```

Dấu:

```text
%
```

trước `opsmaint` nghĩa là group.

---

# 39. Không edit sudoers bằng editor bình thường

Tránh:

```bash
sudo vim /etc/sudoers
```

Preferred:

```bash
sudo visudo
```

Hoặc drop-in:

```bash
sudo visudo -f /etc/sudoers.d/day3-ops
```

Lý do lớn nhất: `visudo` kiểm tra syntax.

Một lỗi sudoers trong production có thể làm mất đường privilege elevation.

Red Hat cũng hướng dẫn dùng `visudo`/`visudo -f` cho sudo policy. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/security_hardening/managing-sudo-access?utm_source=chatgpt.com)

---

# 40. Verify sudo configuration

Sau thay đổi:

```bash
sudo visudo -c
```

Expected invariant:

```text
parsed OK
```

Kiểm tra user được phép gì:

```bash
sudo -l
```

Hoặc từ admin:

```bash
sudo -l -U opsalice
```

Đây là evidence rất tốt.

---

# 41. Least privilege với sudo

Sai:

```text
opsalice ALL=(ALL) ALL
```

nếu requirement chỉ là một privileged operation.

Sai hơn:

```text
opsalice ALL=(ALL) NOPASSWD: ALL
```

mà không có justification.

Tốt hơn nếu requirement chỉ là command A:

```text
opsalice ALL=(root) /path/to/command-A
```

Red Hat hiện cũng nhấn mạnh command-specific sudo authorization an toàn hơn full sudo access. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/security_hardening/managing-sudo-access?utm_source=chatgpt.com)

---

# 42. Cẩn thận với “harmless command”

Không phải command nhìn đơn giản là an toàn trong sudoers.

Ví dụ một program có khả năng:

```text
execute shell
load arbitrary file
write arbitrary file
invoke editor
execute plugin
change configuration
```

có thể trở thành privilege escalation.

Vì vậy khi cấp sudo cho command thực tế phải đánh giá:

```text
binary
arguments
environment
child processes
file-write capability
shell escape
```

Đây là tư duy Operations/security quan trọng hơn việc thuộc syntax sudoers.

---

# 43. SSH — mental model

SSH dùng cho secure remote access.

Có hai identity direction rất quan trọng.

Khi bạn chạy:

```bash
ssh alice@server
```

thì trước hết client phải xác minh:

```text
"Đây có đúng server tôi nghĩ hay không?"
```

qua **server host key**.

Sau đó server xác minh:

```text
"Client có quyền login thành Alice hay không?"
```

qua user authentication như SSH public key.

Mental model:

```text
CLIENT                             SERVER

Known hosts     <--- host key ---- sshd
   |
verify server

private key
   |
prove possession
   |
   +---------- signature -------->
                                authorized_keys
                                     |
                                   match
                                     |
                                allow/deny
```

---

# 44. User SSH key ≠ host SSH key

Không được nhầm.

## Host key

Identity của SSH server.

Client lưu nhận dạng server trong:

```text
~/.ssh/known_hosts
```

## User authentication key

Identity credential của user.

Client giữ:

```text
private key
```

Server giữ:

```text
public key
```

thường trong:

```text
~/.ssh/authorized_keys
```

---

# 45. Public/private key

Key pair:

```text
Private key
Public key
```

Quy tắc sống còn:

```text
PRIVATE KEY
= secret
= giữ trên client
= không gửi cho server
= không paste vào ticket/chat/log
```

Public key:

```text
có thể phân phối tới server
```

Server không cần private key của user.

---

# 46. Tạo SSH key

Hiện tại upstream OpenSSH `ssh-keygen` hỗ trợ nhiều key types; Ed25519 là default khi gọi không chỉ định arguments trong bản hiện hành. Để lab explicit và dễ đọc, ta sẽ chỉ định rõ `-t ed25519`. [man.openbsd.org](https://man.openbsd.org/ssh-keygen?utm_source=chatgpt.com)

Trên **client machine**:

```bash
ssh-keygen -t ed25519 \
  -f ~/.ssh/day3_ops_ed25519 \
  -C "day3-ops"
```

Bạn sẽ được hỏi:

```text
Enter passphrase:
```

Cho human-admin key, passphrase thường là lựa chọn tốt nếu policy cho phép.

Không ghi passphrase vào shell history hay lab report.

Files:

```text
~/.ssh/day3_ops_ed25519
~/.ssh/day3_ops_ed25519.pub
```

Private:

```text
day3_ops_ed25519
```

Public:

```text
day3_ops_ed25519.pub
```

---

# 47. Xem fingerprint

Thay vì gửi toàn bộ key lung tung, fingerprint rất hữu ích để verification:

```bash
ssh-keygen -lf ~/.ssh/day3_ops_ed25519.pub
```

Ví dụ:

```text
256 SHA256:AbCdEf... day3-ops (ED25519)
```

Fingerprint là evidence tốt.

---

# 48. Private key permissions

Trên client:

```bash
chmod 600 ~/.ssh/day3_ops_ed25519
```

Kiểm tra:

```bash
ls -l ~/.ssh/day3_ops_ed25519
```

Expected:

```text
-rw-------
```

Không:

```text
-rw-rw-rw-
```

Không:

```text
777
```

---

# 49. Server-side `authorized_keys`

Target user:

```text
opsalice
```

Home:

```text
/home/opsalice
```

SSH directory:

```text
/home/opsalice/.ssh
```

Authorized public keys:

```text
/home/opsalice/.ssh/authorized_keys
```

Upstream OpenSSH hiện cho phép public-key authentication mặc định và mặc định tìm authorized-key files dưới user home; distribution/local configuration có thể override nên production phải kiểm tra effective config thay vì giả định. [man.openbsd.org](https://man.openbsd.org/sshd_config?utm_source=chatgpt.com)

---

# 50. Permission chuẩn cho `.ssh`

Một baseline phổ biến:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Owner phải đúng.

Ví dụ:

```text
drwx------ opsalice opsalice .ssh
-rw------- opsalice opsalice authorized_keys
```

OpenSSH có `StrictModes`: ownership/modes của user files/home có thể được kiểm tra trước khi chấp nhận login. Nếu `.ssh`, `authorized_keys` hoặc home có ownership/writability không an toàn, public-key authentication có thể bị từ chối. [man.openbsd.org](https://man.openbsd.org/OpenBSD-7.2/sshd.8?utm_source=chatgpt.com)

---

# 51. SSH authentication thực sự diễn ra thế nào?

Nhiều fresher hiểu sai:

> "Server lấy public key và decrypt private key."

Không phải.

Simplified:

```text
1. Client says:
   "Tôi muốn login opsalice."

2. Server knows authorized public keys for opsalice.

3. Client proves it possesses corresponding private key
   using cryptographic signature.

4. Server verifies signature using stored public key.

5. Private key never travels to server.

6. If validation + account policy succeed:
   session is created.
```

Đây là lý do mất private key nghiêm trọng.

Ai có private key + passphrase nếu có, có thể impersonate user tùy policy.

---

# 52. `Permission denied (publickey)` nghĩa là gì?

Không nên thấy lỗi và ngay lập tức regen key.

Các layer cần kiểm tra:

```text
Client
↓
network reachability
↓
sshd listening
↓
correct username
↓
server policy allows user
↓
PubkeyAuthentication
↓
AuthorizedKeysFile path
↓
public key content
↓
home/.ssh/authorized_keys ownership
↓
permissions / StrictModes
↓
algorithm/policy
↓
account state
```

Đó mới là evidence-driven troubleshooting.

---

# 53. Debug SSH client

Dùng:

```bash
ssh -v -i ~/.ssh/day3_ops_ed25519 opsalice@SERVER_IP
```

Chi tiết hơn:

```bash
ssh -vvv -i ~/.ssh/day3_ops_ed25519 opsalice@SERVER_IP
```

Tìm:

```text
Offering public key
Server accepts key
Authenticated to ...
```

Hoặc dấu hiệu:

```text
Offering public key
...
Permission denied
```

`-vvv` là evidence, không phải fix.

---

# 54. Current SSH configuration

Trên server, với quyền phù hợp:

```bash
sudo sshd -T
```

Có thể lọc:

```bash
sudo sshd -T | grep -Ei \
'pubkeyauthentication|authorizedkeysfile|strictmodes|permitrootlogin'
```

Điều này tốt hơn việc chỉ:

```bash
grep ...
```

một file config vì effective configuration có thể có defaults/includes.

**Lưu ý distribution/version:** OpenSSH packaging có khác biệt. Ví dụ Ubuntu 24.04 sử dụng systemd socket activation cho OpenSSH server và generator đọc `sshd_config`; vì vậy service-management troubleshooting sẽ được học kỹ hơn ở Day 4/5. [Tài liệu Ubuntu](https://documentation.ubuntu.com/release-notes/24.04/?utm_source=chatgpt.com)

---

# 55. Không disable password authentication vội

Bạn có thể thấy hướng dẫn Internet:

```text
PasswordAuthentication no
```

và apply ngay.

Trong Day 3, **không cần làm việc đó để hoàn thành syllabus**.

Đây là thay đổi có thể làm mất remote access.

Trên production, trước khi hardening SSH authentication:

```text
keep existing admin session open
validate key login in a second session
validate config syntax
confirm fallback/recovery channel
understand PAM/SSSD/cloud access
apply controlled change
test new login
only then retire old mechanism if approved
```

Đây là operational safety.

---

# 56. Không disable root SSH bằng cách copy config trên Internet một cách mù quáng

`PermitRootLogin` và các authentication controls phụ thuộc:

```text
distribution
cloud image
organizational policy
PAM
SSSD/IdM
configuration snippets
```

Current upstream OpenSSH mặc định `PermitRootLogin prohibit-password`, nhưng distro/local config có thể khác. Luôn kiểm tra effective server config. [man.openbsd.org](https://man.openbsd.org/sshd_config.5?utm_source=chatgpt.com)

---

# 57. DAY 3 GUIDED LAB

Bây giờ chúng ta làm assignment chính thức.

## Lab objective

Bạn sẽ xây dựng:

```text
Users:
opsalice
auditor1

Groups:
appops
opsmaint

Resources:
/srv/day3app/config/app.conf
/srv/day3app/shared/

Access:
opsalice:
  member appops
  member opsmaint

auditor1:
  not appops
  not opsmaint

app.conf:
  root:appops
  640

config directory:
  root:appops
  750

shared:
  root:appops
  2770

sudo:
  opsmaint may run only /usr/bin/id as root

SSH:
  opsalice logs in using public key
```

---

# 58. Lab assumptions

Use a disposable VM/lab server.

Bạn cần:

```text
Linux VM
existing admin account with sudo
SSH server already available/reachable
second terminal/client
```

Commands dưới đây phù hợp với common Ubuntu/RHEL-style GNU/Linux.

Không chạy lab này trên production.

---

# 59. Trước thay đổi — baseline evidence

Tạo evidence directory bằng user admin hiện tại:

```bash
mkdir -p ~/day3-evidence
```

Ghi environment:

```bash
date | tee ~/day3-evidence/00-date.txt
hostnamectl | tee ~/day3-evidence/01-host.txt
id | tee ~/day3-evidence/02-admin-id.txt
```

Nếu `hostnamectl` không có:

```bash
hostname
uname -a
```

Mục đích:

```text
ai
làm ở đâu
khi nào
trên OS nào
```

---

# 60. Kiểm tra objects chưa tồn tại

```bash
getent passwd opsalice
getent passwd auditor1
getent group appops
getent group opsmaint
```

Nếu không tồn tại, output thường empty/nonzero.

Không tiếp tục tạo blindly nếu name đã tồn tại.

---

# 61. Tạo groups

```bash
sudo groupadd appops
sudo groupadd opsmaint
```

Verify:

```bash
getent group appops
getent group opsmaint
```

Evidence:

```bash
{
  getent group appops
  getent group opsmaint
} | tee ~/day3-evidence/10-groups.txt
```

---

# 62. Tạo users

```bash
sudo useradd -m -U -s /bin/bash opsalice
sudo useradd -m -U -s /bin/bash auditor1
```

Verify:

```bash
getent passwd opsalice
getent passwd auditor1

id opsalice
id auditor1
```

---

# 63. Thiết lập lab password

Để test `sudo` authentication dễ dàng trong lab:

```bash
sudo passwd opsalice
```

Bạn tự nhập password lab tại terminal.

**Không copy password vào chat, screenshot hay evidence.**

Nếu muốn test interactive login với `auditor1`:

```bash
sudo passwd auditor1
```

Không dùng password hardcoded trong command.

---

# 64. Assign controlled group membership

```bash
sudo usermod -aG appops,opsmaint opsalice
```

Verify:

```bash
id opsalice
```

Expected invariant:

```text
opsalice ∈ appops
opsalice ∈ opsmaint
```

Auditor:

```bash
id auditor1
```

Expected:

```text
auditor1 ∉ appops
auditor1 ∉ opsmaint
```

Evidence:

```bash
{
  id opsalice
  id auditor1
} | tee ~/day3-evidence/11-user-groups.txt
```

---

# 65. Tạo directory structure

```bash
sudo mkdir -p /srv/day3app/config
sudo mkdir -p /srv/day3app/shared
```

Trước permission change:

```bash
ls -ld /srv/day3app \
       /srv/day3app/config \
       /srv/day3app/shared
```

---

# 66. Tạo config file

```bash
echo 'environment=day3-lab' |
  sudo tee /srv/day3app/config/app.conf >/dev/null
```

Check:

```bash
sudo cat /srv/day3app/config/app.conf
```

Expected content:

```text
environment=day3-lab
```

---

# 67. Apply ownership

Application root:

```bash
sudo chown root:appops /srv/day3app
```

Config directory:

```bash
sudo chown root:appops /srv/day3app/config
```

Config file:

```bash
sudo chown root:appops /srv/day3app/config/app.conf
```

Shared directory:

```bash
sudo chown root:appops /srv/day3app/shared
```

---

# 68. Apply permissions

Top directory:

```bash
sudo chmod 750 /srv/day3app
```

Config directory:

```bash
sudo chmod 750 /srv/day3app/config
```

Config:

```bash
sudo chmod 640 /srv/day3app/config/app.conf
```

Shared collaborative directory:

```bash
sudo chmod 2770 /srv/day3app/shared
```

`2770` ở đây dùng setgid directory, là phần bổ sung giúp files do members tạo ra inherit group `appops`.

---

# 69. Verify permissions

```bash
ls -ld /srv/day3app \
       /srv/day3app/config \
       /srv/day3app/shared

ls -l /srv/day3app/config/app.conf
```

Expected invariant:

```text
/srv/day3app
root appops 750

/srv/day3app/config
root appops 750

app.conf
root appops 640

/shared
root appops 2770
```

Có thể lấy numeric evidence:

```bash
stat -c '%A %a %U %G %n' \
  /srv/day3app \
  /srv/day3app/config \
  /srv/day3app/config/app.conf \
  /srv/day3app/shared
```

Ví dụ:

```text
drwxr-x--- 750 root appops /srv/day3app
drwxr-x--- 750 root appops /srv/day3app/config
-rw-r----- 640 root appops /srv/day3app/config/app.conf
drwxrws--- 2770 root appops /srv/day3app/shared
```

Evidence:

```bash
stat -c '%A %a %U %G %n' \
  /srv/day3app \
  /srv/day3app/config \
  /srv/day3app/config/app.conf \
  /srv/day3app/shared |
tee ~/day3-evidence/12-permissions.txt
```

---

# 70. Positive permission test

Test dưới identity `opsalice`:

```bash
sudo -u opsalice \
  cat /srv/day3app/config/app.conf
```

Expected:

```text
environment=day3-lab
```

Tại sao?

`opsalice` ∈ `appops`.

File:

```text
root:appops 640
```

group permission:

```text
r--
```

---

# 71. Negative write test

Thử:

```bash
sudo -u opsalice \
  sh -c 'echo unauthorized-change >> /srv/day3app/config/app.conf'
```

Expected:

```text
Permission denied
```

Đây là **success của security requirement**, không phải failure của lab.

Sau đó chứng minh nội dung không đổi:

```bash
cat /srv/day3app/config/app.conf
```

Expected:

```text
environment=day3-lab
```

---

# 72. Negative unauthorized-user test

```bash
sudo -u auditor1 \
  cat /srv/day3app/config/app.conf
```

Expected:

```text
Permission denied
```

Tại sao?

`auditor1` không phải owner.

Không thuộc `appops`.

`other = ---`.

Ngoài ra path directories cũng không grant others traversal.

---

# 73. Test shared directory

Cho `opsalice` tạo file:

```bash
sudo -u opsalice \
  touch /srv/day3app/shared/alice-test.txt
```

Kiểm tra:

```bash
ls -l /srv/day3app/shared/
```

Expected group:

```text
appops
```

do setgid directory.

Nếu file group đúng:

```text
opsalice appops
```

thì setgid directory đang hoạt động.

---

# 74. `auditor1` không được tạo file

```bash
sudo -u auditor1 \
  touch /srv/day3app/shared/auditor-test.txt
```

Expected:

```text
Permission denied
```

Bây giờ bạn đã chứng minh cả:

```text
allowed user works
unauthorized user denied
```

Negative testing là cực kỳ quan trọng trong Operations.

---

# 75. Configure restricted sudo

Trước hết xác nhận `id` path:

```bash
command -v id
```

Trên Ubuntu/RHEL thông thường:

```text
/usr/bin/id
```

Tạo drop-in:

```bash
sudo visudo -f /etc/sudoers.d/day3-ops
```

Thêm:

```text
%opsmaint ALL=(root) /usr/bin/id
```

Save.

---

# 76. Validate sudo syntax ngay lập tức

```bash
sudo visudo -c
```

Không tiếp tục nếu configuration không parse OK.

Verify policy:

```bash
sudo -l -U opsalice
```

Bạn cần thấy `/usr/bin/id` trong permitted commands.

---

# 77. Test sudo với `opsalice`

Mở login shell:

```bash
sudo -iu opsalice
```

Sau đó:

```bash
id
```

Expected:

```text
uid=<opsalice UID> ...
```

Liệt kê sudo permission:

```bash
sudo -l
```

Sau đó:

```bash
sudo /usr/bin/id
```

Expected:

```text
uid=0(root) gid=0(root) ...
```

Nghĩa là approved privileged action hoạt động.

---

# 78. Negative sudo test

Vẫn dưới `opsalice`:

```bash
sudo /usr/bin/cat /etc/shadow
```

Expected:

```text
Sorry, user opsalice is not allowed ...
```

Đây là một trong những evidence quan trọng nhất của lab.

Bạn vừa chứng minh:

```text
ALLOWED:
/usr/bin/id

DENIED:
/usr/bin/cat /etc/shadow
```

Đó là least privilege thực sự.

Thoát:

```bash
exit
```

---

# 79. SSH Key Lab — client side

Trên laptop/client hoặc VM client:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```

Tạo key:

```bash
ssh-keygen \
  -t ed25519 \
  -f ~/.ssh/day3_ops_ed25519 \
  -C "day3-ops"
```

Verify:

```bash
ls -l \
  ~/.ssh/day3_ops_ed25519 \
  ~/.ssh/day3_ops_ed25519.pub
```

Harden private key:

```bash
chmod 600 ~/.ssh/day3_ops_ed25519
```

Fingerprint:

```bash
ssh-keygen -lf ~/.ssh/day3_ops_ed25519.pub
```

Lưu fingerprint làm evidence.

---

# 80. Chuyển PUBLIC key tới server

Nhấn mạnh:

```text
COPY:
day3_ops_ed25519.pub

DO NOT COPY:
day3_ops_ed25519
```

Nếu bạn đã có admin SSH access tới server:

```bash
scp ~/.ssh/day3_ops_ed25519.pub \
  ADMIN_USER@SERVER_IP:/tmp/opsalice.pub
```

`ADMIN_USER` là account admin hiện tại, không phải `opsalice`.

---

# 81. Install authorized key trên server

Server:

```bash
sudo install \
  -d \
  -m 700 \
  -o opsalice \
  -g opsalice \
  /home/opsalice/.ssh
```

Sau đó:

```bash
sudo install \
  -m 600 \
  -o opsalice \
  -g opsalice \
  /tmp/opsalice.pub \
  /home/opsalice/.ssh/authorized_keys
```

Remove staging file:

```bash
rm -f /tmp/opsalice.pub
```

---

# 82. Verify SSH file security

```bash
sudo ls -ld \
  /home/opsalice \
  /home/opsalice/.ssh

sudo ls -l \
  /home/opsalice/.ssh/authorized_keys
```

Expected:

```text
.ssh:
opsalice opsalice
700

authorized_keys:
opsalice opsalice
600
```

Verify numeric:

```bash
sudo stat -c '%A %a %U %G %n' \
  /home/opsalice/.ssh \
  /home/opsalice/.ssh/authorized_keys
```

---

# 83. Test public-key SSH

Từ client:

```bash
ssh \
  -i ~/.ssh/day3_ops_ed25519 \
  opsalice@SERVER_IP
```

Sau login:

```bash
whoami
id
hostname
```

Expected:

```text
whoami = opsalice
```

Sau đó:

```bash
cat /srv/day3app/config/app.conf
```

Expected:

```text
environment=day3-lab
```

Test write:

```bash
echo test >> /srv/day3app/config/app.conf
```

Expected:

```text
Permission denied
```

Đây là integration verification:

```text
SSH authentication passed
+
Linux authorization passed
+
least privilege preserved
```

---

# 84. Evidence cho SSH

Client:

```bash
ssh-keygen -lf ~/.ssh/day3_ops_ed25519.pub
```

Và:

```bash
ssh \
  -v \
  -i ~/.ssh/day3_ops_ed25519 \
  opsalice@SERVER_IP
```

Không cần lưu toàn bộ `-vvv` nếu chứa quá nhiều metadata.

Evidence tối thiểu nên chứng minh:

```text
correct user
public-key authentication succeeded
correct key fingerprint
correct server
```

**Không bao giờ đưa private key hoặc passphrase vào evidence package.**

---

# 85. Failure Injection 1 — directory permission

Chúng ta cố ý phá một permission.

Baseline:

```bash
stat -c '%a %U %G %n' /srv/day3app/config
```

Expected:

```text
750 root appops ...
```

Đổi:

```bash
sudo chmod 740 /srv/day3app/config
```

Bây giờ `appops` có:

```text
r--
```

nhưng không có:

```text
x
```

Test:

```bash
sudo -u opsalice \
  cat /srv/day3app/config/app.conf
```

Expected failure.

Root-cause reasoning:

```text
app.conf itself = readable by appops
BUT
parent config directory has no group execute/traverse
THEREFORE
path cannot be traversed
```

Restore:

```bash
sudo chmod 750 /srv/day3app/config
```

Validate:

```bash
sudo -u opsalice \
  cat /srv/day3app/config/app.conf
```

Success trở lại.

Đây là troubleshooting quan trọng hơn học thuộc `chmod`.

---

# 86. Failure Injection 2 — SSH ownership/mode

**Chỉ trên lab.**

Baseline:

```bash
sudo stat -c '%a %U %G %n' \
  /home/opsalice/.ssh/authorized_keys
```

Set insecure mode:

```bash
sudo chmod 666 \
  /home/opsalice/.ssh/authorized_keys
```

Thử SSH lại từ **một session mới**.

Tùy effective OpenSSH/platform policy với `StrictModes`, key authentication có thể bị từ chối vì security properties không phù hợp. OpenSSH khuyến nghị `authorized_keys` không writable bởi users khác, và `StrictModes` có thể từ chối unsafe ownership/modes. [man.openbsd.org](https://man.openbsd.org/OpenBSD-7.2/sshd.8?utm_source=chatgpt.com)

Restore:

```bash
sudo chmod 600 \
  /home/opsalice/.ssh/authorized_keys

sudo chown opsalice:opsalice \
  /home/opsalice/.ssh/authorized_keys
```

Retest SSH.

**Luôn giữ admin session hiện tại mở khi failure-inject SSH access.**

---

# 87. SSH troubleshooting procedure

Khi SSH key fail:

### Symptom

```text
Permission denied (publickey)
```

### Scope

Hỏi:

```text
Chỉ một user hay tất cả?
Chỉ một client hay mọi client?
Password login có hoạt động?
Network connection có tới sshd?
```

### Evidence client

```bash
ssh -vvv \
  -i ~/.ssh/day3_ops_ed25519 \
  opsalice@SERVER_IP
```

### Identity

```bash
getent passwd opsalice
id opsalice
```

### File presence

```bash
sudo ls -la /home/opsalice/.ssh
```

### Ownership

```bash
sudo stat -c '%U %G %n' \
  /home/opsalice \
  /home/opsalice/.ssh \
  /home/opsalice/.ssh/authorized_keys
```

### Permission

```bash
sudo stat -c '%a %n' \
  /home/opsalice \
  /home/opsalice/.ssh \
  /home/opsalice/.ssh/authorized_keys
```

### Effective sshd configuration

```bash
sudo sshd -T |
grep -Ei \
'pubkeyauthentication|authorizedkeysfile|strictmodes'
```

### Key fingerprint comparison

Client:

```bash
ssh-keygen -lf ~/.ssh/day3_ops_ed25519.pub
```

Server:

```bash
sudo ssh-keygen -lf \
  /home/opsalice/.ssh/authorized_keys
```

Nếu có nhiều lines trong `authorized_keys`, inspect cẩn thận.

### Server logs

**Supplementary; logging sẽ học sâu Day 6.**

Ubuntu thường:

```bash
sudo journalctl -u ssh --since "10 minutes ago"
```

RHEL-family thường:

```bash
sudo journalctl -u sshd --since "10 minutes ago"
```

Hoặc distro có authentication log khác.

Không phỏng đoán. Lấy evidence.

---

# 88. sudo Failure Injection — cách an toàn

Không cố tình phá `/etc/sudoers`.

Thay vào đó:

```bash
cat >/tmp/day3-sudoers-test <<'EOF'
%opsmaint THIS_IS_INVALID
EOF
```

Validate:

```bash
sudo visudo -cf /tmp/day3-sudoers-test
```

Bạn sẽ thấy syntax error.

Fix:

```bash
cat >/tmp/day3-sudoers-test <<'EOF'
%opsmaint ALL=(root) /usr/bin/id
EOF
```

Retest:

```bash
sudo visudo -cf /tmp/day3-sudoers-test
```

Expected:

```text
parsed OK
```

Cleanup:

```bash
rm /tmp/day3-sudoers-test
```

Bạn vừa luyện syntax troubleshooting mà không gây lockout sudo thật.

---

# 89. Permission troubleshooting framework chuẩn L1

Khi application báo:

```text
Permission denied
```

không chạy `chmod 777`.

Đi theo:

```text
SYMPTOM
   ↓
EXPECTED BEHAVIOR
   ↓
IDENTITY
whoami / id
   ↓
RESOURCE
exact file/path?
   ↓
OWNERSHIP
ls -l / stat
   ↓
PATH PERMISSIONS
all parent directories
   ↓
GROUP MEMBERSHIP
id / getent
   ↓
RECENT CHANGE
user/group/chown/chmod/deploy?
   ↓
SAFE TEST
sudo -u target_user ...
   ↓
ROOT CAUSE
   ↓
MINIMAL REMEDIATION
   ↓
RETEST
   ↓
EVIDENCE
```

Đây là methodology bạn sẽ dùng ở Day 20.

---

# 90. Example troubleshooting scenario

Tomcat tương lai chạy user:

```text
tomcat
```

Application log:

```text
Permission denied: /opt/myapp/config/application.properties
```

File:

```text
-rw-r----- root appops application.properties
```

Sai troubleshooting:

```bash
sudo chmod 777 application.properties
```

Đúng:

```bash
id tomcat
```

Giả sử:

```text
tomcat không thuộc appops
```

Requirement là Tomcat phải read config.

Hypothesis:

```text
wrong group membership
```

Possible controlled remediation:

```bash
sudo usermod -aG appops tomcat
```

Nhưng nếu service session đang tồn tại, service có thể cần controlled restart để process nhận group membership mới.

Đó là lý do permission troubleshooting liên quan đến process lifecycle — Day 4 sẽ kết nối phần này.

---

# 91. Mistakes / Anti-patterns phải tránh

| Anti-pattern                        | Tại sao sai                                  |
| ----------------------------------- | -------------------------------------------- |
| `chmod 777` để “fix”                | che root cause, mở quá nhiều quyền           |
| `chmod -R 777`                      | blast radius cực lớn                         |
| `chown -R` không inspect            | có thể phá application/system                |
| Dùng root cho mọi việc              | phá least privilege                          |
| Share root password                 | attribution/audit kém                        |
| Full sudo khi chỉ cần 1 command     | privilege quá rộng                           |
| `NOPASSWD: ALL` không justification | giảm security                                |
| `usermod -G` quên `-a`              | xóa supplementary memberships khác           |
| Copy private SSH key lên server     | compromise security model                    |
| `chmod 777 ~/.ssh`                  | SSH có thể từ chối + security exposure       |
| Edit sudoers trực tiếp              | syntax error có thể gây lockout              |
| Disable SSH password ngay           | dễ lock yourself out                         |
| Chỉ test “allowed”                  | không chứng minh unauthorized access bị chặn |

---

# 92. Security & Least Privilege checklist

Một account tốt nên trả lời được:

```text
Ai sở hữu account?
Account dùng cho human hay service?
Có cần interactive shell?
Có cần password authentication?
Có cần SSH?
Cần group nào?
Có cần sudo?
Nếu sudo, cần command nào?
Account có expiry/offboarding process không?
SSH key thuộc device/person nào?
Có revoke procedure không?
```

Không có câu trả lời:

```text
"Cho full sudo cho nhanh."
```

---

# 93. SSH key lifecycle

SSH key không chỉ có:

```text
generate
```

Mà là:

```text
generate
↓
protect private key
↓
register public key
↓
validate
↓
inventory
↓
rotate when required
↓
revoke
↓
remove during offboarding
```

Ví dụ nhân viên nghỉ:

Không nên chỉ:

```bash
passwd -l alice
```

rồi cho rằng mọi SSH path đã chắc chắn bị revoke trên mọi environment.

Account locking/public-key behavior có thể phụ thuộc platform, PAM/SSSD/identity source; OpenSSH documentation cũng cảnh báo account accessibility semantics là system-dependent. [Linux Documentation](https://linux.die.net/man/8/sshd?utm_source=chatgpt.com)

Operations tốt phải xác minh actual access paths.

---

# 94. Evidence package — Assignment quan trọng

Syllabus yêu cầu **submit verification evidence**, không chỉ “tôi làm rồi”. 00_MASTER_SYLLABUS_Java_Fresher…

Evidence directory của bạn nên có đại loại:

```text
day3-evidence/
├── 00-date.txt
├── 01-host.txt
├── 02-admin-id.txt
├── 10-groups.txt
├── 11-user-groups.txt
├── 12-permissions.txt
├── 13-positive-access.txt
├── 14-negative-access.txt
├── 15-sudo-policy.txt
├── 16-sudo-positive.txt
├── 17-sudo-negative.txt
├── 18-ssh-fingerprint.txt
├── 19-ssh-server-permissions.txt
└── 20-final-validation.txt
```

Không được chứa:

```text
password
private SSH key
passphrase
token
secret
```

---

# 95. Capture permission evidence

Ví dụ:

```bash
{
  echo '=== IDENTITY ==='
  id opsalice
  id auditor1

  echo
  echo '=== GROUPS ==='
  getent group appops
  getent group opsmaint

  echo
  echo '=== PERMISSIONS ==='
  stat -c '%A %a %U %G %n' \
    /srv/day3app \
    /srv/day3app/config \
    /srv/day3app/config/app.conf \
    /srv/day3app/shared
} | tee ~/day3-evidence/20-final-validation.txt
```

---

# 96. Capture sudo evidence

```bash
sudo -l -U opsalice |
tee ~/day3-evidence/15-sudo-policy.txt
```

Syntax:

```bash
sudo visudo -c |
tee ~/day3-evidence/15b-sudo-syntax.txt
```

---

# 97. SSH evidence không leak secret

Client:

```bash
ssh-keygen -lf ~/.ssh/day3_ops_ed25519.pub |
tee ~/day3-evidence/18-ssh-fingerprint.txt
```

Server:

```bash
sudo stat -c '%A %a %U %G %n' \
  /home/opsalice/.ssh \
  /home/opsalice/.ssh/authorized_keys
```

Evidence đó đủ chứng minh:

```text
authorized key location
owner
group
mode
fingerprint
```

mà không expose private credential.

---

# 98. Cleanup / Rollback

## Cảnh báo

`userdel -r` và recursive deletion là destructive.

Chỉ cleanup trong disposable lab.

Trước cleanup:

```bash
id opsalice
id auditor1

ls -la /srv/day3app
```

Backup evidence trước.

Remove sudo rule:

```bash
sudo rm -f /etc/sudoers.d/day3-ops
sudo visudo -c
```

Remove lab tree:

```bash
sudo rm -rf /srv/day3app
```

Chỉ chạy sau khi xác minh chính xác path:

```bash
ls -ld /srv/day3app
```

Remove users:

```bash
sudo userdel -r opsalice
sudo userdel -r auditor1
```

Remove groups nếu vẫn tồn tại:

```bash
sudo groupdel appops
sudo groupdel opsmaint
```

Verify:

```bash
getent passwd opsalice
getent passwd auditor1
getent group appops
getent group opsmaint
```

Expected:

```text
no records
```

Giữ lại:

```text
~/day3-evidence
```

nếu đó là evidence submission.

---

# 99. Production rollback khác lab cleanup

Trong production, rollback không nhất thiết là:

```bash
delete user
delete directory
```

Ví dụ bạn thay đổi:

```text
app.conf
root:root 600
```

sang:

```text
root:appops 640
```

Rollback phải biết baseline:

```text
owner before = root
group before = root
mode before  = 600
```

rồi:

```bash
sudo chown root:root app.conf
sudo chmod 600 app.conf
```

Vì thế **pre-change evidence** rất quan trọng.

Nếu bạn không biết state trước change, rollback sẽ trở thành guessing.

---

# 100. L1 versus escalation boundary

Một fresher/L1 thường có thể làm những thứ đã được approved/runbook cho phép như:

```text
inspect identity
inspect permissions
inspect ownership
check group membership
check approved SSH key location
test access
collect logs/evidence
apply documented chmod/chown
restore documented baseline
```

Nên escalate nếu thấy:

```text
unexpected root ownership change across system
unknown privileged UID 0 account
sudoers rules đáng ngờ
SSH access compromise
private key suspected leaked
identity sourced from LDAP/SSSD nhưng inconsistent
SELinux/AppArmor denial chưa hiểu
filesystem corruption
broad permission drift
production access policy không có approved remediation
```

Escalation tốt phải gửi evidence:

```text
symptom
expected state
affected user/resource
UID/GID
ownership/mode
recent changes
tests performed
results
what you did NOT change
recommended next investigation
```

---

# 101. Supplementary — ACL, SELinux, capabilities

Classic:

```text
owner / group / other
```

không phải toàn bộ security model của Linux.

Sau này bạn có thể gặp:

```text
POSIX ACL
SELinux
AppArmor
Linux capabilities
read-only filesystem
NFS identity mapping
container namespaces
```

Vì vậy nếu:

```text
ls -l
```

nhìn có vẻ đúng mà access vẫn fail, đừng kết luận:

```text
"Linux permission bị lỗi."
```

Có thể còn một authorization layer khác.

Day 3 bắt buộc tập trung classic Unix DAC, nhưng bạn phải biết giới hạn mental model này.

---

# 102. Exam Focus — Lý thuyết

Bạn phải có thể trả lời ngay:

### Q: UID là gì?

Numeric user identity được kernel sử dụng.

### Q: GID là gì?

Numeric group identity.

### Q: Primary và supplementary group khác gì?

Primary group là group chính của user; supplementary groups cung cấp memberships bổ sung.

### Q: `chmod 640` nghĩa là gì?

```text
owner rw
group r
other none
```

### Q: `chmod 750`?

```text
owner rwx
group r-x
other ---
```

### Q: `x` trên directory?

Traverse/search directory.

### Q: `w` trên directory?

Cho phép thay đổi directory entries theo các permission constraints liên quan.

### Q: `chown alice:appops file`?

Set owner Alice và group appops.

### Q: `usermod -aG`?

Append user vào supplementary group(s).

### Q: Tại sao `visudo`?

Syntax validation và safer sudo policy editing.

### Q: Private SSH key nằm đâu?

Client-side và phải bảo mật.

### Q: Server nhận gì?

Public key.

### Q: `authorized_keys` dùng để làm gì?

Chứa keys được phép authenticate cho target user.

### Q: Least privilege?

Cấp đúng mức quyền tối thiểu cần cho nhiệm vụ.

---

# 103. Exam Focus — Practical

Một practical task rất có thể giống:

```text
Create:
user = javaops
group = deployers

Requirements:
javaops member deployers
/srv/app owned root:deployers
deployers can read/traverse
others cannot access
javaops can SSH via public key
javaops may run one approved privileged command
provide verification
```

Bạn phải biết trình tự:

```text
inspect
create group
create user
assign membership
verify identity
create resource
set ownership
set permissions
positive test
negative test
sudo policy
validate sudoers
positive sudo test
negative sudo test
SSH key setup
SSH verification
collect evidence
```

Không nhảy thẳng vào random `chmod`.

---

# 104. Knowledge Check

Hãy tự trả lời trước khi xem đáp án.

1. File có `750` thì group được quyền gì?
2. Directory có group `r--`, user thuộc group đó. User có `cd` vào được không?
3. `640` khác `660` ở đâu?
4. `chown root:tomcat file` làm gì?
5. `chmod g+w file` làm gì?
6. `usermod -G appops alice` nguy hiểm ở điểm nào?
7. Private SSH key nên đặt ở server hay client?
8. `.ssh` thường dùng permission nào?
9. `authorized_keys` thường dùng permission nào?
10. Tại sao positive test chưa đủ?
11. Tại sao `chmod 777` không phải solution permission tốt?
12. `sudo -l` dùng để làm gì?
13. Khi user được thêm group nhưng shell cũ chưa thấy group, nguyên nhân thường là gì?
14. `rwx` trên file và directory có cùng semantics không?
15. Nếu file `777` nhưng một parent directory không có `x`, user có chắc đọc file được không?

### Đáp án

```text
1. r-x
2. Không; thiếu execute/traverse.
3. Group: 640 = r--; 660 = rw-.
4. Owner=root, group=tomcat.
5. Thêm write permission cho group class.
6. Có thể replace supplementary group list.
7. Client.
8. 700 là baseline phổ biến.
9. 600 là baseline phổ biến.
10. Cần chứng minh unauthorized access cũng bị deny.
11. Over-permission và che root cause.
12. Liệt kê sudo privileges hiện tại.
13. Group list được thiết lập khi session/login được tạo.
14. Không.
15. Không; parent path traversal cũng phải được phép.
```

---

# 105. Independent Practical Challenge

Sau khi hoàn thành guided lab, hãy tự làm bài này **không nhìn solution**:

```text
Create users:
deploy1
support1

Create groups:
javadeploy
javasupport

Requirements:

deploy1:
- member of javadeploy
- may read/write deployment staging directory

support1:
- member of javasupport
- may read application diagnostics
- must not modify deployment staging

Create:
/srv/java-lab/staging
/srv/java-lab/diagnostics

Design ownership and permissions yourself.

Configure SSH public-key login for deploy1.

Give deploy1 one narrowly scoped sudo capability.

Prove:
1. authorized operation succeeds
2. forbidden operation fails
3. SSH key login succeeds
4. ownership/mode is correct
5. sudo policy parses correctly
6. no secret appears in evidence
```

Đừng dùng ACL cho challenge này. Hãy giải bằng classic users/groups/ownership/permissions để củng cố đúng Day 3.

---

# 106. Mastery Check

Day 3 **chưa được coi là mastered** chỉ vì bạn hiểu:

```bash
chmod 755
```

Mức mastery thực sự là khi tôi cho:

```text
User A cần read.
User B cần read/write.
User C không được access.
Admin task X cần sudo.
Remote login phải dùng SSH key.
```

và bạn có thể tự chuyển requirement đó thành:

```text
users
groups
UID/GID
ownership
permissions
sudo rules
SSH authorization
positive tests
negative tests
evidence
rollback
```

mà không cần dùng `777`, full root hoặc copy private keys lung tung.

---

# 107. Những gì Day 3 kết nối với các ngày sau

Day 3 là nền móng trực tiếp cho:

```text
Day 4
systemd services chạy dưới service user

Day 5
SSH/network troubleshooting

Day 6
authentication/sudo/SSH log investigation

Day 9
AWS EC2 SSH access

Day 12
Tomcat service account

Day 14
secrets và application permissions

Day 15
PostgreSQL OS account

Day 17
Prometheus/node_exporter service accounts

Day 20
permission failure injection
```

Do đó mục tiêu của hôm nay không phải “nhớ vài Linux commands”, mà là hiểu **Linux access control ở mức đủ để vận hành hệ thống thực tế**.

## Bộ command Day 3 bạn nên có thể dùng không cần tra cứu

```bash
whoami
id
groups
getent passwd USER
getent group GROUP

sudo groupadd GROUP
sudo useradd -m -U -s /bin/bash USER
sudo usermod -aG GROUP USER

ls -l
ls -ld
stat

sudo chown USER:GROUP PATH
sudo chgrp GROUP PATH
sudo chmod MODE PATH

sudo -l
sudo -l -U USER
sudo visudo
sudo visudo -f /etc/sudoers.d/FILE
sudo visudo -c

ssh-keygen -t ed25519
ssh-keygen -lf PUBLIC_KEY
ssh -i PRIVATE_KEY USER@HOST
ssh -vvv -i PRIVATE_KEY USER@HOST

sudo sshd -T
```

Nếu bạn thực hiện đầy đủ **Guided Lab + failure injections + evidence package** ở trên và gửi cho tôi output/evidence, tôi có thể đóng vai mentor review từng bước: kiểm tra user/group design, permission calculations, sudo least privilege, SSH key setup, phát hiện lỗi và đánh giá mức độ sẵn sàng Day 4. [docs.redhat.com](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/security_hardening/managing-sudo-access?utm_source=chatgpt.com)
