# Day 4 — Module 3: Package Management + Integrated Operations & Failure Recovery

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 4 bắt buộc bao phủ:

> **Concept/Lecture:** Processes, signals, systemd services, packages, environment variables, and service startup.  
> **Assignment/Lab:** Install a package, create/manage a systemd service, inspect processes, and recover a failed service. :chatgpt-content-reference{index="0"}

Module 1 đã giải quyết tầng:

```text
process
→ PID / PPID
→ state
→ signals
→ environment
→ /proc
```

Module 2 đặt phía trên nó:

```text
systemd
→ service unit
→ execution context
→ start/stop/restart
→ enablement
→ MainPID
→ failure state
```

Module 3 sẽ bổ sung mảnh còn thiếu của Day 4:

```text
package/repository management
```

và sau đó tích hợp toàn bộ:

```text
repository
    ↓
package metadata
    ↓
package installation
    ↓
installed executable
    ↓
package ownership
    ↓
custom service
    ↓
environment
    ↓
systemd
    ↓
process
    ↓
failure
    ↓
evidence
    ↓
root cause
    ↓
recovery
    ↓
validation
    ↓
cleanup / handover
```

Đây là module kết thúc **toàn bộ Day 4**.

Phần package management là **Mandatory syllabus**. Các phần package integrity verification, repository trust, package transaction history, RPM verification, APT internals và package/service forensic relationships sẽ được đánh dấu là **Supplementary / Beyond the explicit syllabus**, nhưng mình đưa vào vì chúng rất giá trị cho Operations và troubleshooting.

---

# 2. Mục tiêu của Module 3

Kết thúc module này, mục tiêu không phải là bạn nhớ:

```bash
apt install ...
```

hay:

```bash
dnf install ...
```

Mục tiêu là bạn có thể trả lời đầy đủ chuỗi câu hỏi vận hành:

```text
Máy này thuộc distro family nào?
       ↓
Package manager đúng là gì?
       ↓
Repository nào cung cấp software?
       ↓
Package name chính xác là gì?
       ↓
Version/architecture nào sẽ được cài?
       ↓
Có dependency nào đi kèm?
       ↓
Transaction sẽ thay đổi gì?
       ↓
Sau install, làm sao chứng minh package có mặt?
       ↓
Binary thực sự nằm ở đâu?
       ↓
File này thuộc package nào?
       ↓
Package có cung cấp systemd service không?
       ↓
Service có active không?
       ↓
Service có enabled không?
       ↓
Process thực tế là gì?
       ↓
Environment/context đúng chưa?
       ↓
Nếu service failed: evidence ở đâu?
       ↓
Root cause là package, unit, permission,
environment hay process?
       ↓
Recovery tối thiểu là gì?
       ↓
Làm sao chứng minh hệ thống phục hồi?
```

Đây mới là “Administration & Operations”, không phải chỉ sử dụng package manager.

---

# 3. Trước hết: Package là gì?

Một **package** không đơn giản là “một chương trình”.

Package là một đơn vị phân phối software chứa payload và metadata.

Mental model:

```text
Package
│
├── software files
│   ├── executable
│   ├── library
│   ├── configuration template
│   ├── documentation
│   └── possibly systemd unit files
│
├── metadata
│   ├── name
│   ├── version
│   ├── architecture
│   ├── dependencies
│   ├── conflicts
│   ├── provides
│   └── maintainer/vendor information
│
└── installation/removal logic
    └── possibly scripts/triggers
```

Một package có thể cài:

```text
1 executable
```

hoặc:

```text
hàng trăm libraries/config/docs
```

hoặc thậm chí:

```text
không có standalone executable nào
```

Ví dụ library package.

---

# 4. Package name ≠ executable name ≠ service name

Đây là distinction rất quan trọng.

Ví dụ hypothetical:

```text
Package:
openssh-server

Executable:
/usr/sbin/sshd

Service:
ssh.service hoặc sshd.service tùy distro
```

Ba tên có thể khác nhau.

Do đó nếu ticket nói:

> “Package `foo` đã cài rồi.”

Bạn chưa thể kết luận:

```text
foo.service phải tồn tại
```

Tương tự:

> “Binary `/usr/bin/foo` tồn tại.”

không chứng minh:

```text
systemd service đã enable
```

Hay:

> “Service active.”

không chứng minh:

```text
package database healthy
```

Giữ các tầng riêng:

```text
PACKAGE
   ↓ installs
FILE / EXECUTABLE
   ↓ can be invoked by
SERVICE UNIT
   ↓ manages
PROCESS
```

---

# 5. Hai ecosystem chính trong khóa học

Trong Linux enterprise operations, hai family bạn rất thường gặp là:

| Debian family | RPM family           |
| ------------- | -------------------- |
| Debian        | RHEL                 |
| Ubuntu        | Rocky Linux          |
| Linux Mint    | AlmaLinux            |
| …             | Fedora / derivatives |

Package format thường tương ứng:

```text
Debian family
.deb

RPM family
.rpm
```

Nhưng package format chỉ là một tầng.

Ta còn có **high-level package manager**.

Mental model:

```text
Debian / Ubuntu

APT
 │
 ├── repository metadata
 ├── dependency resolution
 ├── download
 └── transaction orchestration
        │
        ↓
      dpkg
        │
        └── local .deb package database/files
```

Trong Debian ecosystem, `dpkg` là package manager nền tảng hơn, còn `apt` là high-level CLI; Ubuntu documentation hiện mô tả `dpkg` là medium-level tool và nói `apt` là frontend thân thiện hơn cho quản trị package thông thường. :chatgpt-content-reference{index="1"}

RHEL family:

```text
DNF
 │
 ├── repository metadata
 ├── dependency solving
 ├── download
 └── transaction orchestration
        │
        ↓
       RPM
        │
        └── RPM database / installed package payload
```

Trong RHEL 9, Red Hat chỉ định **DNF** là utility chính để quản lý software. `yum` vẫn tồn tại vì compatibility, nhưng trên RHEL 9 nó là alias/compatibility interface tới DNF. :chatgpt-content-reference{index="2"}

---

# 6. Vì sao không dùng `dpkg -i` hoặc `rpm -i` cho mọi thứ?

Giả sử bạn có:

```text
foo.deb
```

và chạy:

```bash
sudo dpkg -i foo.deb
```

`dpkg` biết cách unpack/install local Debian package.

Nhưng dependency resolution ở repository level không phải vai trò high-level chính của `dpkg`.

APT xử lý:

```text
package requested
      ↓
dependency graph
      ↓
repository candidates
      ↓
download dependencies
      ↓
orchestrate dpkg transaction
```

Tương tự RPM family.

Bạn có thể:

```bash
sudo rpm -i foo.rpm
```

nhưng normal administration thường ưu tiên:

```bash
sudo dnf install ./foo.rpm
```

vì DNF có thể giải dependency từ configured repositories khi có thể. Red Hat documentation xác nhận `dnf install <path_to_RPM_file>` là supported workflow và DNF sẽ cố lấy dependencies từ repositories. :chatgpt-content-reference{index="3"}

Mental rule:

```text
Normal admin package installation:
use high-level package manager.

Low-level tooling:
use deliberately when you understand why.
```

---

# 7. Trước package operation: xác định OS

Không bao giờ đoán distro chỉ từ prompt hoặc hostname.

Chạy:

```bash
cat /etc/os-release
```

Ví dụ Ubuntu:

```text
NAME="Ubuntu"
ID=ubuntu
ID_LIKE=debian
VERSION_ID="24.04"
```

Ví dụ RHEL:

```text
NAME="Red Hat Enterprise Linux"
ID="rhel"
VERSION_ID="9.x"
```

Sau đó kiểm tra tools:

```bash
command -v apt
command -v dnf
command -v rpm
command -v dpkg
```

Không dùng cả:

```bash
apt
```

và:

```bash
dnf
```

trên cùng procedure chỉ vì bạn biết cả hai.

Phải xác định platform trước.

---

# 8. Package repository là gì?

Package thường không được cài từ random URL mỗi lần.

Distro có **repositories**.

Concept:

```text
repository
│
├── package metadata
├── version information
├── architecture information
├── dependency metadata
├── signatures/checksums
└── package payloads
```

Package manager lấy metadata về local machine để biết:

```text
package nào available?
version nào?
dependency nào?
repository nào cung cấp?
```

---

# 9. APT repository configuration

APT đọc configured sources từ:

```text
/etc/apt/sources.list
```

và:

```text
/etc/apt/sources.list.d/
```

Modern APT hỗ trợ cả legacy `.list` one-line format và Deb822 `.sources`; documentation hiện đề xuất `.sources` cho hệ thống mới, và Ubuntu hiện dùng dạng như `/etc/apt/sources.list.d/ubuntu.sources` trên các release mới. :chatgpt-content-reference{index="4"}

Do đó nếu tutorial nói:

> “Ubuntu repository luôn nằm duy nhất trong `/etc/apt/sources.list`”

thì đó là assumption cũ/quá hẹp.

Inspect an toàn:

```bash
ls -l /etc/apt/sources.list /etc/apt/sources.list.d/ 2>/dev/null
```

Không edit repository configuration trong lab Day 4 trừ khi thật sự cần.

---

# 10. DNF repository configuration

Trên RHEL-family:

```text
/etc/dnf/dnf.conf
```

chứa main DNF configuration.

Repository files thường nằm:

```text
/etc/yum.repos.d/*.repo
```

Red Hat hiện khuyến nghị định nghĩa repositories trong `.repo` files ở `/etc/yum.repos.d/` thay vì nhồi repository definitions vào `dnf.conf`. :chatgpt-content-reference{index="5"}

Inspect:

```bash
dnf repolist
```

Expanded:

```bash
dnf repolist --all
```

Thông tin repository cụ thể:

```bash
dnf repoinfo <repo-id>
```

---

# 11. Repository trust là security boundary

Package manager chạy với root privilege để ghi:

```text
/usr
/etc
/var
```

và có thể cài executable/system service.

Do đó repository bạn trust có ảnh hưởng gần như:

```text
"cho software từ nguồn này quyền được cài vào OS"
```

Đây là security decision lớn.

APT hiện verify authenticated repository metadata; modern APT mặc định từ chối unauthenticated repositories trong nhiều trường hợp thay vì im lặng cài software. Documentation hiện khuyến nghị scoped `Signed-By` keyrings và cảnh báo rất mạnh về insecure repositories. :chatgpt-content-reference{index="6"}

Red Hat cũng cảnh báo cài software từ unverified/untrusted repositories có thể gây vấn đề security, stability, compatibility và maintainability. :chatgpt-content-reference{index="7"}

Anti-pattern:

```text
"APT báo signature error"
      ↓
disable signature check
```

hoặc:

```text
"DNF GPG verification failed"
      ↓
turn gpgcheck off
```

Đó không phải remediation mặc định.

Đó là bypass một trust control.

---

# 12. Mental model của package transaction

Khi chạy:

```bash
sudo apt install tree
```

hoặc:

```bash
sudo dnf install tree
```

không chỉ đơn giản là:

```text
download file
→ copy executable
```

Mental model tốt hơn:

```text
requested package
      ↓
repository metadata
      ↓
candidate/version selection
      ↓
dependency solver
      ↓
transaction proposal
      ↓
administrator review
      ↓
download
      ↓
integrity/authentication checks
      ↓
package database update
      ↓
unpack/install files
      ↓
configuration/scriptlets/triggers
      ↓
possibly service/system integration
      ↓
transaction completion
```

RHEL documentation xác nhận DNF tự động resolve/install dependencies cho requested package. :chatgpt-content-reference{index="8"}

---

# 13. Package transaction phải được review trước khi approve

Giả sử:

```bash
sudo dnf remove package-x
```

output nói:

```text
Removing:
 package-x
 package-y
 package-z
 important-component
```

Đừng chỉ nhấn:

```text
y
```

vì bạn “chỉ yêu cầu remove package-x”.

Package manager đang nói:

> Transaction thực sự rộng hơn yêu cầu ban đầu.

Red Hat đặc biệt cảnh báo phải review packages/dependencies bị remove vì removal có thể kéo thêm unused dependencies/dependent content. :chatgpt-content-reference{index="9"}

APT `autoremove` cũng phải review kỹ; official manpage nhắc administrator kiểm tra proposed removal vì một package trước đây được đánh dấu dependency có thể giờ lại là software bạn thực sự muốn giữ. :chatgpt-content-reference{index="10"}

Một package command luôn có hai phần:

```text
INTENT
"remove X"

vs

ACTUAL TRANSACTION
"remove X, Y, Z..."
```

Operations phải approve **actual transaction**.

---

# 14. APT — `apt update`

```bash
sudo apt update
```

không có nghĩa:

```text
upgrade all packages
```

Nó tải/cập nhật package metadata từ configured repositories để các operation khác biết available packages/versions. Current APT manual định nghĩa `update` chính xác theo cách này. :chatgpt-content-reference{index="11"}

Mental model:

```text
remote repositories
       ↓
apt update
       ↓
local metadata/cache
       ↓
apt search / show / install
```

APT lists thường được lưu dưới:

```text
/var/lib/apt/lists/
```

theo APT documentation. :chatgpt-content-reference{index="12"}

---

# 15. `apt update` khác `apt upgrade`

Phải phân biệt:

```bash
sudo apt update
```

là:

```text
refresh package metadata
```

Trong khi:

```bash
sudo apt upgrade
```

là một package transaction thay đổi installed software.

Và:

```bash
sudo apt full-upgrade
```

có semantics mạnh hơn; current APT documentation lưu ý `full-upgrade` có thể remove installed packages nếu dependency resolution yêu cầu. :chatgpt-content-reference{index="13"}

Do đó trên production:

```text
"chạy apt update"
```

và:

```text
"chạy apt full-upgrade"
```

có blast radius hoàn toàn khác nhau.

---

# 16. Tìm package trên APT

Nếu bạn chưa chắc package name:

```bash
apt search tree
```

Sau đó inspect:

```bash
apt show tree
```

`apt show` có thể cung cấp:

```text
Package
Version
Architecture
Depends
Installed-Size
Download-Size
Description
```

APT documentation xác nhận `show` cung cấp package details như dependencies, source, sizes và description. :chatgpt-content-reference{index="14"}

Trước install, đây là good habit:

```text
search
→ inspect
→ install
```

thay vì:

```text
guess package name
→ sudo install
```

---

# 17. Candidate version

Package name thôi chưa đủ.

Ví dụ:

```text
tree
```

có thể có nhiều versions:

```text
installed version
candidate version
other repository versions
```

Useful:

```bash
apt policy tree
```

Trên APT systems, `apt-cache policy`/`apt policy` giúp hiểu candidate/version source.

Đây đặc biệt quan trọng khi:

```text
third-party repository
multiple releases
pinning
backports
```

xuất hiện.

Day 4 chỉ cần awareness, nhưng operations thực tế phải luôn biết:

> Tôi sắp cài version nào từ source nào?

---

# 18. Cài package với APT

Primary interactive command:

```bash
sudo apt install tree
```

Trước khi xác nhận, review:

```text
NEW packages
UPGRADED packages
REMOVED packages
additional disk space
download size
```

Lab không dùng:

```bash
-y
```

ở lần đầu.

Tại sao?

Vì mục tiêu học operations là **review transaction**.

`-y` bỏ bước operator confirmation.

Nó hữu ích trong automation có kiểm soát, nhưng không nên dùng để che transaction details khi bạn đang học hoặc xử lý incident.

---

# 19. Verify package trên Debian/Ubuntu

Sau:

```bash
sudo apt install tree
```

đừng chỉ kết luận:

> Không có error nên chắc cài rồi.

Verify:

```bash
dpkg-query -W tree
```

Output dạng:

```text
tree    2.x...
```

Có thể query status:

```bash
dpkg-query -s tree
```

Current `dpkg-query` documentation định nghĩa `-W` là query package/version và `-s` là xem package status record. :chatgpt-content-reference{index="15"}

---

# 20. `dpkg -l` và trạng thái `ii`

Bạn thường gặp:

```bash
dpkg -l tree
```

Output:

```text
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/...
||/ Name ...
ii  tree ...
```

Trong shorthand:

```text
first i → desired state = install
second i → current state = installed
```

`dpkg-query` documentation mô tả các package-state fields này. :chatgpt-content-reference{index="16"}

Nhưng cho script/evidence machine-readable, thường thích:

```bash
dpkg-query -W ...
```

với format cụ thể hơn.

---

# 21. Package đã cài những file nào?

Đây là một command cực quan trọng:

```bash
dpkg-query -L tree
```

Nó liệt kê files mà package database ghi nhận thuộc package. Current documentation xác nhận `-L/--listfiles` có chức năng này. :chatgpt-content-reference{index="17"}

Ví dụ bạn có thể thấy:

```text
/usr/bin/tree
/usr/share/man/man1/tree.1.gz
...
```

Bây giờ bạn đã nối:

```text
package tree
       ↓
/usr/bin/tree
```

bằng evidence.

---

# 22. File này thuộc package nào?

Inverse question:

> Tôi có `/usr/bin/tree`. Package nào đã cài nó?

```bash
dpkg-query -S /usr/bin/tree
```

Output thường dạng:

```text
tree: /usr/bin/tree
```

Đây là command cực hữu ích trong troubleshooting.

Ví dụ Java:

```bash
dpkg-query -S /usr/bin/java
```

có thể dẫn tới package/symlink relationship mà bạn cần điều tra thêm.

`dpkg-query` hỗ trợ search ownership bằng filename/path pattern. :chatgpt-content-reference{index="18"}

---

# 23. Caveat của `dpkg-query -L`

`dpkg-query -L package` không nhất thiết cho bạn **mọi file runtime từng được package tạo ra**.

Documentation nói rõ nó không liệt kê extra files được tạo ra bởi maintainer scripts và cũng có caveats với alternatives/diversions. :chatgpt-content-reference{index="19"}

Ví dụ một package có thể lúc install tạo:

```text
runtime directory
cache
generated config
service user
database/state file
```

thông qua scripts hoặc first startup.

Do đó:

```text
package file list
```

không phải:

```text
toàn bộ filesystem impact vĩnh viễn
```

Đây là một subtle but important operations point.

---

# 24. `apt remove` vs `apt purge`

Nếu:

```bash
sudo apt remove package
```

APT thường remove packaged software nhưng giữ một số modified configuration files lại.

Nếu:

```bash
sudo apt purge package
```

APT remove thêm package-managed configuration leftovers.

Current APT documentation xác nhận `remove` thường giữ modified user config, trong khi `purge` loại chúng; home-directory user data không tự động bị purge theo semantics này. :chatgpt-content-reference{index="20"}

Mental model:

```text
remove
→ package payload removed
→ some package configuration may remain

purge
→ package + package-managed configuration removed more completely
```

---

# 25. Đừng coi `purge` là “xóa mọi dấu vết”

Package/application có thể tạo:

```text
/var/lib/app/data
/home/user/...
external DB records
backups
logs
generated runtime state
```

Và purge semantics phụ thuộc package scripts/policy.

Do đó:

```text
apt purge package
```

không phải guarantee:

```text
mọi byte từng liên quan application biến mất
```

Đặc biệt database/data directories phải được xử lý theo runbook riêng.

---

# 26. `apt autoremove`

```bash
sudo apt autoremove
```

được thiết kế để remove packages được automatic-install như dependency và giờ không còn cần thiết. :chatgpt-content-reference{index="21"}

Nhưng đây **không phải garbage collector hoàn toàn vô hại**.

Trước confirm phải review transaction.

Production rule:

```text
autoremove proposal
      ↓
review every meaningful package
      ↓
only then approve
```

Không dùng:

```bash
sudo apt autoremove -y
```

mù quáng sau package removal.

---

# 27. `apt` vs `apt-get` trong automation

Current APT documentation nói `apt` được thiết kế chủ yếu cho interactive end-user use và output/default behavior có thể thay đổi; với scripts, dedicated tools như `apt-get`/`apt-cache` được khuyến nghị vì backward compatibility tốt hơn. :chatgpt-content-reference{index="22"}

Do đó:

Interactive admin:

```bash
sudo apt install tree
```

Automation/runbook script thường cân nhắc:

```bash
apt-get ...
```

với explicit options.

Đây là **Supplementary / Beyond explicit syllabus**, nhưng rất hữu ích khi sang Bash automation ở Day 6.

---

# 28. DNF — package management trên RHEL family

Trên RHEL 9:

```bash
dnf --version
```

Primary command:

```bash
dnf
```

Mặc dù:

```bash
yum
```

vẫn thường chạy vì compatibility, Red Hat hướng dẫn dùng DNF trên RHEL 9. :chatgpt-content-reference{index="23"}

Trong khóa học, với RHEL 8/9-family, mình sẽ ưu tiên:

```bash
dnf
```

để mental model rõ.

---

# 29. Tìm package bằng DNF

Search:

```bash
dnf search tree
```

Thông tin:

```bash
dnf info tree
```

Query repository package:

```bash
dnf repoquery tree
```

Red Hat documentation xác nhận các workflow này cho RHEL 9. :chatgpt-content-reference{index="24"}

Nếu bạn chỉ biết executable/path cần tìm:

```bash
dnf provides /usr/bin/tree
```

để xác định package nào cung cấp file/path. :chatgpt-content-reference{index="25"}

Đây rất mạnh trong troubleshooting:

```text
command missing
       ↓
which package should provide it?
       ↓
dnf provides /path
```

---

# 30. DNF repositories

Kiểm tra:

```bash
dnf repolist
```

All:

```bash
dnf repolist --all
```

Info:

```bash
dnf repoinfo
```

RHEL 9 content có thể đến từ repositories như BaseOS và AppStream cùng các repos khác tùy entitlement/environment. :chatgpt-content-reference{index="26"}

Nếu:

```bash
dnf install tree
```

báo package không có, đừng immediately download random RPM từ Internet.

Investigate:

```text
correct package name?
correct repo enabled?
subscription/repository entitlement?
metadata fresh?
architecture?
```

---

# 31. Cài package bằng DNF

```bash
sudo dnf install tree
```

DNF sẽ resolve dependency graph và show transaction summary trước confirmation. Red Hat documentation định nghĩa chính workflow này. :chatgpt-content-reference{index="27"}

Again, lần đầu lab không dùng:

```bash
-y
```

để bạn thực hành review transaction.

---

# 32. Verify package trên RPM systems

Sau install:

```bash
rpm -q tree
```

Expected dạng:

```text
tree-2.x-....x86_64
```

Nếu chưa cài:

```text
package tree is not installed
```

Detailed:

```bash
rpm -qi tree
```

Files:

```bash
rpm -ql tree
```

Package owning file:

```bash
rpm -qf /usr/bin/tree
```

RPM official documentation hiện hỗ trợ query installed package, package file list và `-f/--file` để query package sở hữu một installed file. :chatgpt-content-reference{index="28"}

---

# 33. RPM package identity chi tiết hơn

RPM package identity thường gồm:

```text
Name
Epoch
Version
Release
Architecture
```

Bạn có thể gặp abbreviation:

```text
NEVRA
```

hoặc:

```text
NEVR
```

Ví dụ conceptual:

```text
tree-2.1.0-10.el9.x86_64
```

Trong troubleshooting dependency/version mismatch, đừng chỉ ghi:

```text
tree installed
```

Mà nên ghi:

```text
exact package version + architecture
```

---

# 34. DNF install package ≠ service start

Một package có thể chứa service:

```text
foo.service
```

nhưng package install và service lifecycle là **hai tầng riêng**.

Sau install phải kiểm tra:

```bash
systemctl list-unit-files | grep foo
```

hoặc:

```bash
systemctl status foo.service
```

Package policy có thể khác theo distro/package:

```text
some packages may auto-start
some may not
some may enable
some may not
```

Không assume.

Best practice:

```text
install
→ inspect package files
→ identify unit
→ inspect active/enabled state
```

---

# 35. Tìm package có cài systemd unit không

Debian/Ubuntu:

```bash
dpkg-query -L <package> | grep -E '\.service$'
```

RPM:

```bash
rpm -ql <package> | grep -E '\.service$'
```

Nếu thấy:

```text
/usr/lib/systemd/system/foo.service
```

hoặc distro equivalent, bạn có evidence package cung cấp service.

Sau đó:

```bash
systemctl cat foo.service
```

để inspect effective unit.

---

# 36. Một package có thể cài service nhưng service file path khác `/etc`

Vendor packages thường install unit dưới vendor path như:

```text
/usr/lib/systemd/system/
```

hoặc distro variation:

```text
/lib/systemd/system/
```

Administrator-created units:

```text
/etc/systemd/system/
```

Như Module 2 đã học:

```text
vendor file
≠
administrator override
```

Không sửa package-owned vendor unit trực tiếp nếu có thể tránh.

---

# 37. Package ownership là một troubleshooting superpower

Scenario:

```text
/usr/bin/foo bị lỗi
```

Bạn muốn biết:

> Nó từ đâu ra?

Debian:

```bash
dpkg-query -S /usr/bin/foo
```

RPM:

```bash
rpm -qf /usr/bin/foo
```

Nếu output:

```text
no path found
```

hoặc file không thuộc package database:

```text
manually installed?
custom deployment?
compiled from source?
configuration-management artifact?
```

Đây là distinction cực kỳ giá trị khi incident xảy ra.

---

# 38. Package integrity verification — Supplementary

Trên modern `dpkg`, có:

```bash
dpkg -V package
```

Nhưng bạn phải hiểu chính xác limitation.

Current dpkg documentation nói `--verify` hiện chủ yếu kiểm tra content MD5 đối với files có metadata thích hợp, và rõ ràng cảnh báo đây là **integrity check**, không phải security/authenticity verification. :chatgpt-content-reference{index="29"}

Vì vậy:

```text
dpkg -V clean
```

không chứng minh:

```text
host uncompromised
```

---

# 39. RPM verification mạnh hơn về file attributes

RPM:

```bash
rpm -V tree
```

hoặc:

```bash
rpm --verify tree
```

so sánh installed files với metadata trong RPM database.

Current RPM 6 documentation cho biết verification có thể so sánh:

```text
size
digest
permissions/mode
file type
owner
group
capabilities
...
```

và discrepancies được output. :chatgpt-content-reference{index="30"}

Ví dụ output có:

```text
S
M
5
U
G
T
P
```

tương ứng các mismatch dimensions.

Nhưng:

```text
rpm -V
```

vẫn không phải complete host intrusion-detection solution.

---

# 40. Package file modified có luôn là incident không?

Không.

Configuration file có thể **được phép thay đổi**.

Ví dụ package ships:

```text
/etc/foo/foo.conf
```

Admin có approved change.

Verification có thể báo difference.

Do đó:

```text
file differs from package
```

không tự động nghĩa:

```text
malicious tampering
```

Bạn cần context:

```text
config file?
approved change?
generated state?
package-owned executable?
```

Một modified executable đáng quan tâm hơn một intentionally modified config, nhưng vẫn phải investigate.

---

# 41. Package installation có thể chạy scripts

Package installation không phải “extract archive an toàn tuyệt đối”.

Packages có thể chứa installation/removal logic.

RPM hiện có scriptlet concepts như:

```text
%pre
%post
%preun
%postun
%pretrans
%posttrans
```

và triggers. :chatgpt-content-reference{index="31"}

Debian packages cũng có maintainer-script lifecycle.

Operational implication:

> Installing a trusted package is execution of a privileged software-management transaction, not just copying files.

Vì vậy package source/trust rất quan trọng.

---

# 42. Package install có thể thay service state

Package scripts/triggers có thể:

```text
create service user
install unit
daemon-reload
generate config
migrate state
possibly start/restart services
```

behavior phụ thuộc package/distro/policy.

Do đó trước package change production, cần hỏi:

```text
Does installation restart anything?
Does upgrade restart anything?
Does package contain service?
Does it change config?
What is rollback?
```

Không assume package install luôn zero-downtime.

---

# 43. `apt install` hoặc `dnf install` là state-changing command

Risk classification:

| Command             |       Thay OS state? | Typical privilege |
| ------------------- | -------------------: | ----------------- |
| `apt search`        |        Không đáng kể | normal user       |
| `apt show`          |                Không | normal user       |
| `dpkg-query`        |                Không | normal user       |
| `dnf info`          |                Không | normal user       |
| `rpm -q`            |                Không | normal user       |
| `apt update`        | Metadata/cache state | root              |
| `apt install`       |                   Có | root              |
| `apt remove/purge`  |                   Có | root              |
| `dnf install`       |                   Có | root              |
| `dnf remove`        |                   Có | root              |
| `rpm -V`            |                Không | thường read-only  |
| `systemctl restart` |     Có, availability | root thường cần   |

Điểm quan trọng:

```text
query first
change second
```

---

# 44. Pre-change package checklist

Trước một package installation thực tế, bạn nên biết:

```text
OS/distro/version
package manager
repository source
package name
candidate version
architecture
dependencies
disk requirement
whether package already exists
whether package has a service
potential restart/start effect
rollback/cleanup plan
```

Ví dụ trước `tree` lab:

```bash
cat /etc/os-release
command -v tree || true
```

Debian/Ubuntu:

```bash
apt show tree
```

RHEL family:

```bash
dnf info tree
```

Nếu package đã cài sẵn, ghi nhận:

```text
pre-existing package
```

Đừng giả vờ assignment “install” đã thực hiện nếu không có transaction.

Bạn có thể dùng một harmless package khác được approved trong lab.

---

# 45. Package choice cho Day 4 lab

Mình sẽ dùng:

```text
tree
```

làm package mẫu nếu repository của bạn cung cấp.

RHEL 9 package manifest hiện liệt kê `tree` trong distribution content. :chatgpt-content-reference{index="32"}

Trên Ubuntu, xác nhận trực tiếp trước bằng:

```bash
apt show tree
```

Nếu package không available trong environment cụ thể, dùng một harmless CLI package khác do distro repository cung cấp.

Tại sao chọn CLI utility thay vì `nginx`/database?

Vì mục tiêu Day 4 là:

```text
package
systemd
process
failure recovery
```

chứ chưa phải mở network listener/firewall hay thay đổi server availability.

Ta sẽ tự tạo service sử dụng binary từ package.

---

# 46. Integrated Lab Architecture

Lab cuối Day 4:

```text
Configured distro repository
        │
        ↓
package "tree"
        │
        ↓
/usr/bin/tree
        │
        ↓
custom worker script
        │
        ↓
day4-inventory.service
        │
        ├── User=
        ├── WorkingDirectory=
        ├── EnvironmentFile=
        ├── ExecStart=
        └── Restart=on-failure
                │
                ↓
            process
                │
        ps / /proc / systemctl
                │
                ↓
        injected bad config
                │
                ↓
             failure
                │
                ↓
       status + minimal journal
                │
                ↓
          root cause
                │
                ↓
            recovery
                │
                ↓
      full validation/evidence
```

Đây là lab duy nhất tổng hợp toàn bộ Day 4.

---

# 47. LAB SAFETY

Lab này thực hiện system changes:

```text
install package
create /opt/day4-package-lab
create /etc/day4-inventory.env
create /etc/systemd/system/day4-inventory.service
start/stop service
```

Chỉ làm trên:

```text
lab VM
training EC2
disposable/non-production Linux host
```

Không làm trên shared/production server nếu chưa được approval.

Trước mọi remove/cleanup package, phải biết package có pre-existing hay không.

---

# 48. Bước 1 — OS discovery

```bash
cat /etc/os-release
```

Sau đó:

```bash
if command -v apt >/dev/null 2>&1; then
    echo "APT family detected"
elif command -v dnf >/dev/null 2>&1; then
    echo "DNF family detected"
else
    echo "Neither apt nor dnf detected"
fi
```

Đây là lab helper, không phải universal distro detection logic cho mọi platform.

Expected invariant:

```text
Bạn biết rõ mình đang theo nhánh APT hay DNF.
```

---

# 49. Bước 2 — Baseline package state

Trước install:

```bash
command -v tree || true
```

Record:

```bash
if command -v tree >/dev/null 2>&1; then
    echo "TREE_PREEXISTED=yes"
else
    echo "TREE_PREEXISTED=no"
fi
```

Nếu muốn lưu shell variable:

```bash
if command -v tree >/dev/null 2>&1; then
    TREE_PREEXISTED=yes
else
    TREE_PREEXISTED=no
fi

echo "$TREE_PREEXISTED"
```

Evidence này quan trọng cho cleanup.

---

# 50. Bước 3A — APT pre-install inspection

Chỉ chạy nhánh này trên Debian/Ubuntu.

Refresh metadata:

```bash
sudo apt update
```

Inspect:

```bash
apt show tree
```

Check installed state:

```bash
dpkg-query -W tree 2>/dev/null || echo "tree not currently installed"
```

Đọc:

```text
version
architecture
depends
installed-size
```

trước khi install.

---

# 51. Bước 3B — DNF pre-install inspection

Chỉ chạy trên RHEL-family.

```bash
dnf info tree
```

Check:

```bash
rpm -q tree
```

Nếu chưa cài:

```text
package tree is not installed
```

Check repository:

```bash
dnf repolist
```

Optional:

```bash
dnf repoquery tree
```

Red Hat docs hiện mô tả `dnf info`, `repoquery`, `repolist` đúng cho các mục đích này. :chatgpt-content-reference{index="33"}

---

# 52. Bước 4A — Install APT

Nếu chưa installed:

```bash
sudo apt install tree
```

Đọc transaction summary.

Không dùng `-y`.

Sau install:

```bash
dpkg-query -W tree
```

Check:

```bash
command -v tree
```

Expected:

```text
/usr/bin/tree
```

Path có thể khác trong unusual environment; verify actual path.

---

# 53. Bước 4B — Install DNF

Nếu chưa installed:

```bash
sudo dnf install tree
```

Review transaction.

Sau:

```bash
rpm -q tree
```

Check executable:

```bash
command -v tree
```

Expected thường:

```text
/usr/bin/tree
```

---

# 54. Bước 5 — Chứng minh package ownership

Debian/Ubuntu:

```bash
dpkg-query -S "$(command -v tree)"
```

Expected dạng:

```text
tree: /usr/bin/tree
```

RPM family:

```bash
rpm -qf "$(command -v tree)"
```

Expected:

```text
tree-...
```

Đây là evidence:

```text
repository package
    ↓
installed package database
    ↓
actual binary
```

---

# 55. Bước 6 — Xem package file inventory

APT:

```bash
dpkg-query -L tree
```

DNF/RPM:

```bash
rpm -ql tree
```

Đừng cần memorize mọi file.

Bạn chỉ cần nhận ra các category:

```text
binary
man page
documentation
possibly locale files
```

---

# 56. Bước 7 — Prove executable works

```bash
tree --version
```

Sau đó:

```bash
tree -L 1 /opt 2>/dev/null | head
```

Chúng ta chưa tạo lab directory nhưng command binary phải execute.

Expected invariant:

```text
executable launches successfully
```

Package installed không chỉ là package database record.

---

# 57. Bước 8 — Tạo lab directory

```bash
sudo install -d -m 0755 /opt/day4-package-lab
```

Tạo test content:

```bash
sudo install -d -m 0755 \
  /opt/day4-package-lab/input \
  /opt/day4-package-lab/output
```

Create harmless files:

```bash
sudo touch \
  /opt/day4-package-lab/input/file-a.txt \
  /opt/day4-package-lab/input/file-b.txt
```

Check:

```bash
tree /opt/day4-package-lab
```

Expected dạng:

```text
/opt/day4-package-lab
├── input
│   ├── file-a.txt
│   └── file-b.txt
└── output
```

Exact formatting không phải invariant.

---

# 58. Bước 9 — Chọn execution identity

Để không tạo OS account mới ngoài scope lab:

```bash
LAB_USER="$(id -un)"
LAB_GROUP="$(id -gn)"

printf 'LAB_USER=%s\nLAB_GROUP=%s\n' \
  "$LAB_USER" "$LAB_GROUP"
```

Trong production application:

```text
dedicated service account
```

thường tốt hơn.

Nhưng lab này chỉ cần chứng minh non-root execution context.

---

# 59. Bước 10 — Ownership của lab directory

Service cần ghi output.

Set owner:

```bash
sudo chown -R "$LAB_USER:$LAB_GROUP" \
  /opt/day4-package-lab
```

Verify:

```bash
ls -ld \
  /opt/day4-package-lab \
  /opt/day4-package-lab/output
```

Không dùng:

```bash
chmod 777
```

để “cho chắc”.

Ta biết chính xác service cần access gì.

---

# 60. Bước 11 — Environment configuration

Create:

```bash
sudo tee /etc/day4-inventory.env >/dev/null <<'EOF'
DAY4_SCAN_DIR=/opt/day4-package-lab/input
DAY4_OUTPUT=/opt/day4-package-lab/output/inventory.txt
DAY4_INTERVAL=10
DAY4_TREE_BIN=/usr/bin/tree
EOF
```

Inspect:

```bash
cat /etc/day4-inventory.env
```

No secrets.

Nếu `command -v tree` cho path khác `/usr/bin/tree`, hãy dùng actual path:

```bash
command -v tree
```

rồi sửa `DAY4_TREE_BIN=` tương ứng.

---

# 61. Bước 12 — Worker script

Create:

```bash
sudo tee /opt/day4-package-lab/inventory-worker.sh >/dev/null <<'EOF'
#!/usr/bin/env bash

set -u

trap 'echo "day4-inventory: received termination signal"; exit 0' TERM INT

: "${DAY4_SCAN_DIR:?DAY4_SCAN_DIR is required}"
: "${DAY4_OUTPUT:?DAY4_OUTPUT is required}"
: "${DAY4_INTERVAL:?DAY4_INTERVAL is required}"
: "${DAY4_TREE_BIN:?DAY4_TREE_BIN is required}"

if [[ ! -x "$DAY4_TREE_BIN" ]]; then
    echo "day4-inventory: ERROR: tree executable not found or not executable: $DAY4_TREE_BIN" >&2
    exit 42
fi

if [[ ! -d "$DAY4_SCAN_DIR" ]]; then
    echo "day4-inventory: ERROR: scan directory missing: $DAY4_SCAN_DIR" >&2
    exit 43
fi

echo "day4-inventory: started pid=$$ user=$(id -un)"

while true; do
    {
        echo "timestamp=$(date --iso-8601=seconds)"
        "$DAY4_TREE_BIN" "$DAY4_SCAN_DIR"
    } > "$DAY4_OUTPUT"

    echo "day4-inventory: inventory updated: $DAY4_OUTPUT"

    sleep "$DAY4_INTERVAL" &
    wait $!
done
EOF
```

Permissions:

```bash
sudo chmod 0755 \
  /opt/day4-package-lab/inventory-worker.sh
```

Validate Bash syntax:

```bash
bash -n \
  /opt/day4-package-lab/inventory-worker.sh
```

Expected:

```text
no output
```

Check:

```bash
echo $?
```

Expected:

```text
0
```

---

# 62. Tại sao script được viết như vậy?

Dòng:

```bash
set -u
```

làm undefined variable trở thành lỗi thay vì silently empty trong nhiều trường hợp.

Các:

```bash
: "${DAY4_SCAN_DIR:?DAY4_SCAN_DIR is required}"
```

buộc configuration bắt buộc phải tồn tại.

Check:

```bash
[[ ! -x "$DAY4_TREE_BIN" ]]
```

giúp failure message rõ.

Ta dùng:

```text
exit 42
exit 43
```

chỉ để phân biệt lab failures.

Exit codes sẽ được học sâu hơn vào Day 6 theo syllabus; hôm nay chỉ cần hiểu non-zero exit báo unsuccessful process termination cho systemd.

---

# 63. Bước 13 — Unit file

```bash
LAB_USER="$(id -un)"
LAB_GROUP="$(id -gn)"

sudo tee /etc/systemd/system/day4-inventory.service >/dev/null <<EOF
[Unit]
Description=Day 4 package and systemd integrated inventory worker

[Service]
Type=exec
User=$LAB_USER
Group=$LAB_GROUP
WorkingDirectory=/opt/day4-package-lab
EnvironmentFile=/etc/day4-inventory.env
ExecStart=/opt/day4-package-lab/inventory-worker.sh
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
EOF
```

Inspect:

```bash
sudo cat \
  /etc/systemd/system/day4-inventory.service
```

---

# 64. Bước 14 — Static validation

```bash
sudo systemd-analyze verify \
  /etc/systemd/system/day4-inventory.service
```

Nếu không có output/error:

```text
static validation passed
```

Nhưng nhớ:

```text
static validation ≠ runtime validation
```

Script có thể vẫn fail vì environment/data/permissions.

---

# 65. Bước 15 — daemon-reload

```bash
sudo systemctl daemon-reload
```

Sau đó:

```bash
systemctl status \
  day4-inventory.service \
  --no-pager
```

Expected initial state:

```text
Loaded: loaded
Active: inactive
```

Enablement:

```bash
systemctl is-enabled day4-inventory.service
```

Expected initially:

```text
disabled
```

---

# 66. Bước 16 — Start service

```bash
sudo systemctl start \
  day4-inventory.service
```

Verify:

```bash
systemctl is-active \
  day4-inventory.service
```

Expected:

```text
active
```

Status:

```bash
systemctl status \
  day4-inventory.service \
  --no-pager
```

Read:

```text
Loaded
Active
Main PID
CGroup
```

---

# 67. Bước 17 — Inspect process

Get PID:

```bash
MAINPID="$(
  systemctl show \
    day4-inventory.service \
    -p MainPID \
    --value
)"

echo "$MAINPID"
```

Then:

```bash
ps -p "$MAINPID" \
  -o pid,ppid,user,group,stat,lstart,etime,cmd
```

Bạn phải đọc được:

```text
PID
PPID
user
group
process state
start time
elapsed time
command
```

Đây là Module 1 trở lại.

---

# 68. Bước 18 — Inspect `/proc`

Executable:

```bash
sudo readlink \
  "/proc/$MAINPID/exe"
```

Working directory:

```bash
sudo readlink \
  "/proc/$MAINPID/cwd"
```

Expected cwd:

```text
/opt/day4-package-lab
```

Environment:

```bash
sudo tr '\0' '\n' \
  < "/proc/$MAINPID/environ" |
grep '^DAY4_'
```

Expected safe values:

```text
DAY4_SCAN_DIR=/opt/day4-package-lab/input
DAY4_OUTPUT=/opt/day4-package-lab/output/inventory.txt
DAY4_INTERVAL=10
DAY4_TREE_BIN=/usr/bin/tree
```

Order không quan trọng.

---

# 69. Bước 19 — Verify application output

```bash
cat \
  /opt/day4-package-lab/output/inventory.txt
```

Expected dạng:

```text
timestamp=...
/opt/day4-package-lab/input
├── file-a.txt
└── file-b.txt
```

Bây giờ ta có nhiều lớp evidence:

```text
package installed
binary executable
service active
process alive
environment correct
output generated
```

Đó tốt hơn rất nhiều so với:

```text
systemctl status = green
```

---

# 70. Bước 20 — Minimal logging evidence

`journalctl` đầy đủ thuộc Day 6, không phải Day 4. MASTER_SYLLABUS đặt `journalctl`, text processing, Bash scripting và exit codes ở Day 6. :chatgpt-content-reference{index="34"}

Hôm nay chỉ dùng tối thiểu để hoàn thành requirement “recover a failed service”:

```bash
sudo journalctl \
  -u day4-inventory.service \
  -n 20 \
  --no-pager
```

Expected messages:

```text
day4-inventory: started ...
day4-inventory: inventory updated ...
```

Đừng học toàn bộ journald filtering ở đây; Day 6 sẽ đào sâu.

---

# 71. Bước 21 — Prove package-to-process dependency

Ta đã biết:

```text
package tree
      ↓
/usr/bin/tree
```

Worker có:

```text
DAY4_TREE_BIN=/usr/bin/tree
```

và output chứng minh binary được sử dụng.

Bạn có thể prove package ownership lại:

APT:

```bash
dpkg-query -S "$(
  grep '^DAY4_TREE_BIN=' \
  /etc/day4-inventory.env |
  cut -d= -f2-
)"
```

RPM:

```bash
rpm -qf "$(
  grep '^DAY4_TREE_BIN=' \
  /etc/day4-inventory.env |
  cut -d= -f2-
)"
```

Concept:

```text
package manager
→ installed binary
→ application dependency
→ service process
```

---

# 72. Bước 22 — Enable startup

Trước:

```bash
systemctl is-enabled \
  day4-inventory.service
```

Enable:

```bash
sudo systemctl enable \
  day4-inventory.service
```

After:

```bash
systemctl is-enabled \
  day4-inventory.service
```

Expected:

```text
enabled
```

Check active separately:

```bash
systemctl is-active \
  day4-inventory.service
```

Expected still:

```text
active
```

Hai dimensions vẫn độc lập.

---

# 73. Bước 23 — Inspect enablement implementation

```bash
ls -l \
  /etc/systemd/system/multi-user.target.wants/day4-inventory.service
```

Expected symlink tới unit.

Bạn vừa chứng minh:

```text
[Install]
WantedBy=multi-user.target
      ↓
systemctl enable
      ↓
.wants/ symlink
```

Không phải magic.

---

# 74. Bước 24 — Controlled restart

Before:

```bash
OLD_PID="$(
  systemctl show \
    day4-inventory.service \
    -p MainPID \
    --value
)"
```

Restart:

```bash
sudo systemctl restart \
  day4-inventory.service
```

After:

```bash
NEW_PID="$(
  systemctl show \
    day4-inventory.service \
    -p MainPID \
    --value
)"

printf 'old=%s\nnew=%s\n' \
  "$OLD_PID" \
  "$NEW_PID"
```

Normally:

```text
old PID != new PID
```

Validate output:

```bash
cat \
  /opt/day4-package-lab/output/inventory.txt
```

Restart thành công ở service layer chưa đủ; output phải tiếp tục được generate.

---

# 75. Failure Injection — cấu hình dependency path sai

Đây là failure injection chính của Day 4.

Chúng ta **không phá package binary**, không modify `/usr/bin/tree`, không chmod system file.

Ta sẽ inject configuration error.

Baseline:

```bash
systemctl is-active \
  day4-inventory.service
```

Expected:

```text
active
```

Record:

```bash
cp /etc/day4-inventory.env \
   /tmp/day4-inventory.env.before-failure
```

Đây chỉ là lab backup.

---

# 76. Inject bad environment

Change:

```bash
sudo sed -i \
  's#^DAY4_TREE_BIN=.*#DAY4_TREE_BIN=/usr/bin/tree-does-not-exist#' \
  /etc/day4-inventory.env
```

Verify:

```bash
grep '^DAY4_TREE_BIN=' \
  /etc/day4-inventory.env
```

Expected:

```text
DAY4_TREE_BIN=/usr/bin/tree-does-not-exist
```

Important question:

> Service đang chạy có fail ngay không?

**Không nhất thiết.**

Existing process đã được start với old environment:

```text
DAY4_TREE_BIN=/usr/bin/tree
```

EnvironmentFile change không live-inject vào running process.

---

# 77. Prove existing process vẫn có old environment

```bash
MAINPID="$(
  systemctl show \
    day4-inventory.service \
    -p MainPID \
    --value
)"

sudo tr '\0' '\n' \
  < "/proc/$MAINPID/environ" |
grep '^DAY4_TREE_BIN='
```

Expected:

```text
DAY4_TREE_BIN=/usr/bin/tree
```

Trong file:

```bash
grep '^DAY4_TREE_BIN=' \
  /etc/day4-inventory.env
```

Expected:

```text
DAY4_TREE_BIN=/usr/bin/tree-does-not-exist
```

Đây là bằng chứng hoàn hảo của mental model:

```text
configuration on disk
          ≠
running process environment
```

---

# 78. Trigger failure bằng controlled restart

Bây giờ restart để new config được applied:

```bash
sudo systemctl restart \
  day4-inventory.service
```

Vì `Restart=on-failure`, systemd có thể cố restart worker nhiều lần.

Worker sees:

```text
/usr/bin/tree-does-not-exist
```

fails check và:

```text
exit 42
```

Status:

```bash
systemctl status \
  day4-inventory.service \
  --no-pager
```

Có thể thấy:

```text
activating (auto-restart)
```

hoặc:

```text
failed
```

tùy timing/rate limiting.

---

# 79. Đừng fix ngay — collect evidence

Incident mindset:

```text
preserve evidence
before remediation
```

Timestamp:

```bash
date --iso-8601=seconds
```

State:

```bash
systemctl show \
  day4-inventory.service \
  -p LoadState \
  -p ActiveState \
  -p SubState \
  -p Result \
  -p MainPID \
  -p ExecMainCode \
  -p ExecMainStatus
```

Unit:

```bash
systemctl cat \
  day4-inventory.service
```

Configuration:

```bash
cat /etc/day4-inventory.env
```

Binary:

```bash
command -v tree
ls -l "$(command -v tree)"
```

Logs:

```bash
sudo journalctl \
  -u day4-inventory.service \
  -n 30 \
  --no-pager
```

---

# 80. Symptom vs root cause

Observed symptom:

```text
service repeatedly failing/restarting
```

Potential hypothesis:

```text
package missing?
binary missing?
bad path?
permission?
environment?
working directory?
```

Evidence:

```bash
command -v tree
```

says:

```text
/usr/bin/tree
```

Package database:

APT:

```bash
dpkg-query -W tree
```

RPM:

```bash
rpm -q tree
```

says package installed.

Config:

```text
DAY4_TREE_BIN=/usr/bin/tree-does-not-exist
```

Therefore root cause is:

```text
invalid environment configuration
```

not:

```text
package installation failure
```

---

# 81. Đây là evidence-driven troubleshooting

Chuỗi reasoning:

```text
SYMPTOM
service not stable
        ↓
EXPECTED
active/running with output refresh
        ↓
RECENT CHANGE
EnvironmentFile modified
        ↓
SERVICE STATE
auto-restart/failed
        ↓
PROCESS RESULT
exit 42
        ↓
LOG
tree executable not found at configured path
        ↓
PACKAGE CHECK
tree installed
        ↓
BINARY CHECK
/usr/bin/tree exists
        ↓
CONFIG CHECK
DAY4_TREE_BIN points to nonexistent path
        ↓
ROOT CAUSE
wrong environment configuration
```

Không cần:

```text
reboot
chmod 777
run as root
reinstall OS
kill -9 random PID
```

---

# 82. Recovery

Restore configuration:

```bash
sudo sed -i \
  's#^DAY4_TREE_BIN=.*#DAY4_TREE_BIN=/usr/bin/tree#' \
  /etc/day4-inventory.env
```

Verify:

```bash
grep '^DAY4_TREE_BIN=' \
  /etc/day4-inventory.env
```

Expected:

```text
DAY4_TREE_BIN=/usr/bin/tree
```

Check dependency:

```bash
test -x /usr/bin/tree &&
echo "tree executable OK"
```

Expected:

```text
tree executable OK
```

---

# 83. Có cần `daemon-reload` không?

Không, vì chúng ta **không sửa `.service` unit definition**.

Ta chỉ sửa:

```text
/etc/day4-inventory.env
```

được unit tham chiếu qua:

```ini
EnvironmentFile=/etc/day4-inventory.env
```

EnvironmentFile được đọc cho process invocation mới.

Do đó:

```text
fix env file
→ restart/start
```

là cần thiết.

Không cần:

```text
daemon-reload
```

chỉ vì contents của external EnvironmentFile thay đổi.

Nếu bạn thay:

```text
EnvironmentFile=/old/path
```

thành:

```text
EnvironmentFile=/new/path
```

**trong unit file**, thì cần `daemon-reload`.

---

# 84. Reset failed state khi cần

Nếu service hit failed/start-rate state:

```bash
sudo systemctl reset-failed \
  day4-inventory.service
```

Sau đó:

```bash
sudo systemctl start \
  day4-inventory.service
```

Nếu nó chỉ đang auto-retrying và config đã fixed, behavior có thể khác tùy timing, nhưng explicit controlled recovery dễ chứng minh hơn trong lab.

---

# 85. Recovery validation — Layer 1: unit

```bash
systemctl show \
  day4-inventory.service \
  -p LoadState \
  -p ActiveState \
  -p SubState \
  -p UnitFileState \
  -p MainPID
```

Expected:

```text
LoadState=loaded
ActiveState=active
SubState=running
UnitFileState=enabled
MainPID=<non-zero>
```

---

# 86. Recovery validation — Layer 2: process

```bash
MAINPID="$(
  systemctl show \
    day4-inventory.service \
    -p MainPID \
    --value
)"

ps -p "$MAINPID" \
  -o pid,ppid,user,group,stat,lstart,etime,cmd
```

Expected invariant:

```text
process exists
correct user/group
long-running
```

---

# 87. Recovery validation — Layer 3: environment

```bash
sudo tr '\0' '\n' \
  < "/proc/$MAINPID/environ" |
grep '^DAY4_TREE_BIN='
```

Expected:

```text
DAY4_TREE_BIN=/usr/bin/tree
```

Đây là critical evidence rằng new process dùng **fixed config**, không phải chỉ old process somehow survived.

---

# 88. Recovery validation — Layer 4: application output

```bash
cat \
  /opt/day4-package-lab/output/inventory.txt
```

Check timestamp.

Wait > interval if necessary, then:

```bash
stat \
  /opt/day4-package-lab/output/inventory.txt
```

Expected:

```text
file modification time advances
```

This proves worker resumed function.

---

# 89. Recovery validation — Layer 5: logs

```bash
sudo journalctl \
  -u day4-inventory.service \
  -n 30 \
  --no-pager
```

Bạn nên thấy transition:

```text
failure
→ restart attempts
→ corrected startup
→ inventory updated
```

Đây là incident narrative.

---

# 90. Một evidence pack tốt trông như thế nào?

Không cần screenshot mọi thứ.

Một evidence pack Day 4 tốt có thể chứa table:

| Evidence                                | Chứng minh                    |
| --------------------------------------- | ----------------------------- |
| `/etc/os-release`                       | Platform/distro               |
| `dpkg-query -W tree` hoặc `rpm -q tree` | Package installed/version     |
| package ownership query                 | `/usr/bin/tree` thuộc package |
| `systemctl cat`                         | Unit definition               |
| `is-active`                             | Runtime service state         |
| `is-enabled`                            | Startup configuration         |
| `systemctl show MainPID`                | Main process identity         |
| `ps`                                    | PID/user/state/command        |
| `/proc/PID/cwd`                         | WorkingDirectory runtime      |
| `/proc/PID/environ`                     | Effective environment         |
| journal excerpt                         | Failure + recovery evidence   |
| generated inventory                     | Application-level proof       |

Điều này tốt hơn 20 screenshots thiếu context.

---

# 91. Failure scenario: package không installed

Giả sử:

```text
DAY4_TREE_BIN=/usr/bin/tree
```

nhưng package chưa cài.

Symptom:

```text
service fails at startup
```

Evidence:

```bash
command -v tree
```

nothing.

APT:

```bash
dpkg-query -W tree
```

fails/not installed.

DNF:

```bash
rpm -q tree
```

not installed.

Root cause:

```text
runtime package dependency missing
```

Remediation in approved lab:

```bash
sudo apt install tree
```

hoặc:

```bash
sudo dnf install tree
```

rồi validate package, binary và service.

---

# 92. Failure scenario: package installed nhưng binary path sai

Package installed:

```text
tree installed
```

Command:

```bash
command -v tree
```

returns:

```text
/usr/bin/tree
```

Unit/env points:

```text
/usr/local/bin/tree
```

Root cause:

```text
configuration path mismatch
```

Reinstall package không giải quyết root cause.

Điều này là troubleshooting lesson lớn:

```text
PACKAGE PRESENT
≠
CONFIGURATION CORRECT
```

---

# 93. Failure scenario: binary tồn tại nhưng service user không execute được

Check:

```bash
ls -l /usr/bin/tree
```

Normal package binary likely executable globally, nhưng với custom app this can fail.

Evidence:

```text
file exists
package installed
service still permission denied
```

Investigate:

```bash
namei -l /path/to/executable
```

Check every directory traversal permission.

Day 3 knowledge returns:

```text
ownership
group
mode
execute/search permission
```

Do not solve with:

```bash
chmod -R 777 /
```

---

# 94. Failure scenario: executable modified

RPM environment:

```bash
rpm -V package
```

có thể detect attributes/content differ.

Debian:

```bash
dpkg -V package
```

có more limited integrity behavior.

But before “reinstall package”, determine:

```text
Was file intentionally changed?
Is it config or executable?
Is host security compromised?
Is there a change record?
```

If a package-owned executable unexpectedly differs, this can cross L1 boundary and require escalation/security investigation.

Do not immediately overwrite forensic evidence.

---

# 95. Failure scenario: package manager locked

APT/Dpkg có package-management locks.

Nếu một apt/dpkg process đang active:

```text
Could not get lock ...
```

Bad response:

```bash
rm -f /var/lib/dpkg/lock*
```

mù quáng.

Bạn có thể corrupt/interfere với legitimate transaction.

First inspect:

```bash
ps -ef | grep -E '[a]pt|[d]pkg'
```

hoặc exact PIDs/process tree.

Ask:

```text
another apt transaction active?
automatic update?
configuration management?
stuck process?
```

Nếu legitimate package transaction đang chạy:

```text
do not interfere
```

unless approved recovery procedure says otherwise.

---

# 96. Failure scenario: repository unavailable

APT:

```text
Temporary failure resolving ...
404 ...
Release file ...
signature error ...
```

DNF:

```text
Cannot download repomd.xml
Failed to download metadata
```

Do not immediately classify:

```text
package manager broken
```

Potential layers:

```text
DNS
network
proxy
repository URL
repository lifecycle
TLS
signature/key
subscription
firewall
```

Day 5 sẽ đào sâu IP/DNS/connectivity.

Day 4 scope:

```text
recognize package transaction cannot proceed
collect exact error
avoid insecure bypass
escalate/network troubleshoot as appropriate
```

---

# 97. Failure scenario: package not found

APT:

```text
Unable to locate package ...
```

DNF:

```text
No match for argument ...
```

Possible causes:

```text
typo
metadata stale
repo disabled
package not in distribution release
different package name
architecture issue
subscription/repo configuration
```

Safe investigation:

APT:

```bash
apt search <term>
apt show <name>
```

DNF:

```bash
dnf search <term>
dnf info <name>
dnf provides <path>
```

Don't search random download site first.

---

# 98. Package reinstall

Sometimes package files are damaged/missing and reinstall is appropriate after diagnosis.

APT can support:

```bash
sudo apt reinstall <package>
```

Current APT includes `reinstall` as a supported package action. :chatgpt-content-reference{index="35"}

DNF:

```bash
sudo dnf reinstall <package>
```

depending on package availability/repos.

But:

```text
reinstall
```

is not universal troubleshooting.

It does not fix:

```text
wrong EnvironmentFile
wrong User=
wrong WorkingDirectory=
bad app data
DB failure
port conflict
```

Use evidence.

---

# 99. DNF transaction history — Supplementary

RHEL DNF records transaction history.

Inspect:

```bash
dnf history
```

Detail:

```bash
dnf history info <ID>
```

Red Hat documentation states history records transaction timeline, time, affected package count and success/abort state. :chatgpt-content-reference{index="36"}

This is very useful for:

```text
"What package change happened just before incident?"
```

It ties directly to:

```text
recent changes
```

in our troubleshooting methodology.

---

# 100. `dnf history undo` is not a magic rollback

DNF supports:

```bash
dnf history undo <ID>
```

and:

```bash
dnf history rollback <ID>
```

but Red Hat explicitly warns that using these to downgrade important RHEL system packages/minor-release state is unsupported/problematic, especially packages such as kernel, glibc and SELinux components. :chatgpt-content-reference{index="37"}

Therefore:

```text
dnf history undo
```

does not mean:

```text
safe universal OS rollback button
```

For production, use approved rollback design.

---

# 101. Package rollback mental model

Suppose version `2` caused issue.

True rollback requires thinking about:

```text
package availability
config compatibility
database/schema compatibility
service state
dependencies
files changed outside package
data changes
security implications
```

Installing old binary package alone may not restore previous system state.

This becomes especially critical for:

```text
database
Java application releases
schema migration
system packages
```

which later Days will cover.

---

# 102. Package cleanup in our lab

First stop service:

```bash
sudo systemctl stop \
  day4-inventory.service
```

Disable:

```bash
sudo systemctl disable \
  day4-inventory.service
```

Verify:

```bash
systemctl is-active \
  day4-inventory.service
```

Expected:

```text
inactive
```

```bash
systemctl is-enabled \
  day4-inventory.service
```

Expected:

```text
disabled
```

---

# 103. Remove lab unit/config

```bash
sudo rm \
  /etc/systemd/system/day4-inventory.service
```

Then:

```bash
sudo systemctl daemon-reload
```

Optional:

```bash
sudo systemctl reset-failed \
  day4-inventory.service 2>/dev/null || true
```

Check:

```bash
systemctl status \
  day4-inventory.service \
  --no-pager
```

Expected:

```text
Unit ... could not be found
```

---

# 104. Remove lab files

```bash
sudo rm -f \
  /etc/day4-inventory.env
```

Then:

```bash
sudo rm -rf \
  /opt/day4-package-lab
```

**Warning:** trước `rm -rf`, verify exact path:

```bash
ls -ld /opt/day4-package-lab
```

Never substitute variables blindly in destructive cleanup.

---

# 105. Có remove package `tree` không?

Chỉ remove nếu:

```text
TREE_PREEXISTED=no
```

và bạn chắc chắn package được cài chỉ cho lab.

Nếu:

```text
TREE_PREEXISTED=yes
```

không remove.

APT:

```bash
sudo apt remove tree
```

DNF:

```bash
sudo dnf remove tree
```

Nhưng **review transaction trước confirmation**.

Nếu package manager proposes unrelated important removals:

```text
cancel
investigate
```

Không cleanup bằng mọi giá.

---

# 106. Không cần `autoremove` chỉ để “dọn sạch”

Sau APT package removal, hệ thống có thể suggest:

```text
apt autoremove
```

Không chạy mù quáng.

Bạn có thể để dependencies lại trong lab nếu không chắc.

Correct Operations preference:

```text
small benign residue
>
accidental removal of useful software
```

khi chưa hiểu transaction.

---

# 107. Package/database state sau cleanup

APT:

```bash
dpkg-query -W tree 2>/dev/null ||
echo "tree not installed"
```

DNF:

```bash
rpm -q tree
```

Filesystem:

```bash
test ! -e /etc/day4-inventory.env &&
echo "environment file removed"

test ! -d /opt/day4-package-lab &&
echo "lab directory removed"
```

Unit:

```bash
systemctl status \
  day4-inventory.service \
  --no-pager
```

Cleanup cũng phải validate như deployment.

---

# 108. Security: Không tải random `.deb`/`.rpm`

Một anti-pattern rất phổ biến:

```text
package not found
      ↓
Google
      ↓
random file-sharing site
      ↓
sudo install
```

Đây là supply-chain risk.

Prefer:

```text
official distro repo
official vendor repo
approved organizational repository
```

Nếu third-party repository cần thêm, phải xác minh:

```text
vendor identity
repository URL
signing key fingerprint
support policy
lifecycle
compatibility
```

APT's current security documentation giải thích repository signatures bảo vệ integrity/authentication chain và strongly discourages insecure repositories. :chatgpt-content-reference{index="38"}

---

# 109. Security: Không disable GPG/signature checks để “fix”

Errors như:

```text
NO_PUBKEY
repository not signed
Bad GPG signature
```

là security evidence.

Không chuyển thành:

```text
--allow-unauthenticated
gpgcheck=0
trusted=yes
```

chỉ để command chạy.

Correct question:

```text
Why is trust verification failing?
```

Possible causes:

```text
wrong repository
expired/rotated key
bad key installation
repository compromise/misconfiguration
MITM/proxy issue
old unsupported repository instructions
```

---

# 110. Security: Package manager chạy với privilege cao

Khi bạn:

```bash
sudo apt install ...
```

hoặc:

```bash
sudo dnf install ...
```

transaction có thể ghi root-owned system paths và run installation logic.

Do đó:

```text
sudo package install
```

phải được xem như privileged change.

Không để fresher mindset biến thành:

```text
"apt install chỉ là download tool"
```

---

# 111. Security: Package version cũng là security information

Operations record nên bao gồm:

```text
package
version
architecture
repository/source
installation time/change ticket
```

Vì khi security advisory xuất hiện:

```text
vulnerable versions
```

bạn cần biết inventory.

Không đủ:

```text
"Java installed"
```

Phải biết:

```text
which Java?
which package?
which exact version?
```

---

# 112. Operations: package change ≠ application change

Một package update có thể cập nhật:

```text
shared library
runtime
daemon
CLI tool
CA certificates
kernel
```

Application có thể bị ảnh hưởng dù package không mang tên application.

Ví dụ:

```text
Java runtime update
```

có thể ảnh hưởng Java application.

```text
OpenSSL library update
```

có thể ảnh hưởng network/TLS consumers.

Dependency thinking phải xuyên tầng.

---

# 113. Operations: đang cài package không đồng nghĩa process đã dùng version mới

Đây là một subtle concept cực kỳ quan trọng.

Suppose daemon đang chạy executable/library cũ trong memory.

Package upgrade thay files on disk.

Running process có thể vẫn dùng:

```text
old process image
old mapped library
```

cho tới restart.

Mental model:

```text
disk package version
          ≠
runtime process state
```

Do đó sau package upgrade, câu hỏi:

```text
Does service require restart?
```

rất quan trọng.

Không assume package transaction automatically updates runtime memory.

---

# 114. Operations: service restart sau package update

Package manager/package scripts có thể tự restart một số services tùy policy, nhưng không phải universal.

Vì vậy production package change phải xác định trước:

```text
Which services use affected files?
Will they restart automatically?
If not, when do we restart?
What is availability impact?
```

Đây sẽ nối trực tiếp Day 18 change/release control.

---

# 115. Operations: package database là evidence

Debian:

```text
/var/lib/dpkg/
```

RPM:

```text
RPM database
```

là critical state.

Không manually edit package database files để “fix” trạng thái trừ khi có specialized recovery procedure.

Package databases giúp trả lời:

```text
what is installed?
which version?
which package owns this file?
```

Nếu database corruption xảy ra, đó thường không còn là random L1-edit territory.

---

# 116. Troubleshooting Framework — package install fails

Hãy sử dụng workflow:

```text
SYMPTOM
package install fails
      ↓
EXPECTED
package should install from approved repository
      ↓
PLATFORM
APT or DNF?
      ↓
SCOPE
one package or all packages?
      ↓
RECENT CHANGE
repository/proxy/DNS/time/key?
      ↓
EVIDENCE
exact error output
      ↓
LAYER
name?
metadata?
network?
repo?
trust?
dependency?
package database?
disk?
      ↓
HYPOTHESES
      ↓
SAFEST TEST
      ↓
ROOT CAUSE
      ↓
REMEDIATION / ESCALATION
      ↓
VERIFY PACKAGE STATE
```

Không random command loop.

---

# 117. Troubleshooting scope question

Suppose:

```bash
apt install tree
```

fails.

Test:

```bash
apt show tree
```

fails too?

What about:

```bash
apt show bash
```

?

Nếu:

```text
only tree unavailable
```

possible package/repo component issue.

Nếu:

```text
all repository metadata fails
```

scope lớn hơn:

```text
network/repository/trust/system
```

Scope narrows hypotheses.

---

# 118. Troubleshooting: “command not found”

User reports:

```text
tree: command not found
```

Do not immediately install.

First:

```bash
command -v tree
```

Then package DB:

APT:

```bash
dpkg-query -W tree
```

RPM:

```bash
rpm -q tree
```

Possible cases:

```text
package not installed
package installed but PATH wrong
binary moved/missing
shell hash/path issue
different package name
```

Install is only one hypothesis.

---

# 119. Case: package installed but command not found

Suppose:

```bash
rpm -q tree
```

says installed.

But:

```bash
tree
```

says command not found.

Check package files:

```bash
rpm -ql tree | grep /bin/
```

Check:

```bash
command -v tree
echo "$PATH"
```

Check actual path:

```bash
ls -l /usr/bin/tree
```

Potential root cause:

```text
PATH
missing file
filesystem issue
package corruption
```

Not automatically “reinstall”.

---

# 120. Case: package file missing

APT:

```bash
dpkg-query -L tree | grep /usr/bin/tree
ls -l /usr/bin/tree
```

If DB says file should exist but file missing:

```text
installed package state
≠
filesystem
```

Now reinstall could be appropriate after understanding why.

But if multiple package files mysteriously disappear:

```text
broader filesystem/security issue
```

may require escalation.

---

# 121. Case: package install succeeds but service absent

Scenario:

```text
package foo installed
systemctl status foo.service
→ unit not found
```

Questions:

```text
Does package actually ship a service?
What is service unit name?
Is service split into another package?
Is it socket-activated?
Is this package only client tools?
```

Query files.

APT:

```bash
dpkg-query -L foo |
grep -E '\.(service|socket)$'
```

RPM:

```bash
rpm -ql foo |
grep -E '\.(service|socket)$'
```

Don't invent a service name from package name.

---

# 122. Case: package install succeeds, service disabled

Perfectly possible:

```text
package installed
service file present
Active: inactive
UnitFileState=disabled
```

Not failure necessarily.

Package lifecycle and service lifecycle are separate.

You decide according to:

```text
runbook
desired state
change ticket
```

whether to:

```bash
systemctl enable --now ...
```

Never enable every installed daemon automatically.

---

# 123. Case: service enabled but package binary missing

This can happen after:

```text
manual file deletion
partial package operation
bad cleanup
filesystem issue
package removed without local custom unit cleanup
```

Evidence:

```bash
systemctl cat service
```

shows:

```text
ExecStart=/usr/bin/foo
```

Check:

```bash
test -x /usr/bin/foo
```

Then package ownership query.

This links:

```text
systemd failure
```

back to:

```text
package/filesystem layer
```

---

# 124. Case: service restart loop after package upgrade

Recent change:

```text
package upgraded
```

Service:

```text
auto-restart loop
```

Investigate:

```text
binary version changed?
config syntax changed?
deprecated option?
library dependency?
permission/user?
data format?
port?
```

Do not repeatedly restart.

The package change is strong correlation evidence, not automatic proof of root cause.

---

# 125. Case: package removal unexpectedly proposes many packages

Example:

```text
Remove:
 package-a
 dependency-b
 component-c
 runtime-d
```

Cancel transaction:

```text
n
```

Review:

APT:

```bash
apt show ...
```

DNF:

```bash
dnf remove ...
```

and inspect proposal.

For DNF, Red Hat explicitly advises reviewing removal set because unused/dependent packages may be affected. :chatgpt-content-reference{index="39"}

---

# 126. L1 boundaries

A fresher L1 administrator should generally be comfortable with read-only operations such as:

```text
identify OS/package manager
search package
inspect versions
query installed state
list files
map file → package
inspect repository list
inspect service/unit state
inspect MainPID
collect package/service evidence
```

State-changing operations require runbook/approval:

```text
install
remove
purge
autoremove
upgrade
repository changes
GPG key changes
service restart
unit edits
```

And these frequently deserve escalation:

```text
package database corruption
signature verification failure with unclear cause
mass removal transaction
kernel/glibc/SELinux rollback
unknown production repository
unexpected package executable modification
persistent service crash after package upgrade
```

L1 vs escalation is determined by:

```text
risk + scope + ownership
```

not command difficulty.

---

# 127. Recovery philosophy

Recovery is not:

```text
make error disappear
```

Recovery means:

```text
restore expected service
+
preserve enough evidence
+
validate all relevant layers
+
avoid introducing another issue
```

Example from lab:

```text
wrong DAY4_TREE_BIN
```

Bad recovery:

```text
change User=root
```

Error might disappear for unrelated reason, but architecture worsens.

Correct recovery:

```text
restore correct binary path
restart
validate PID/environment/output
```

---

# 128. Rollback vs recovery

These are related but not identical.

**Recovery**:

```text
fix current state so service works again
```

**Rollback**:

```text
return configuration/software to known previous state
```

In lab:

```text
configuration changed to bad path
```

Rollback is:

```text
restore previous EnvironmentFile
```

Recovery outcome is:

```text
service functional again
```

Rollback is one recovery technique.

---

# 129. Day 4 integrated troubleshooting stack

Sau Module 3, bạn cần tư duy theo stack:

```text
Package source/repository
        ↓
package metadata
        ↓
installed package
        ↓
filesystem files
        ↓
permissions
        ↓
systemd unit
        ↓
User / Group
        ↓
WorkingDirectory
        ↓
environment
        ↓
ExecStart
        ↓
MainPID
        ↓
process state
        ↓
signals
        ↓
application behavior
```

Một symptom ở bottom có thể bắt nguồn bất kỳ layer nào bên trên.

---

# 130. Ví dụ: service fails, nhiều root cause khác nhau

Cùng symptom:

```text
myapp.service failed
```

nhưng root cause có thể là:

| Layer             | Example                          |
| ----------------- | -------------------------------- |
| Package           | executable package not installed |
| Filesystem        | executable deleted               |
| Permission        | User cannot traverse path        |
| Unit              | wrong `ExecStart=`               |
| Environment       | wrong binary/config path         |
| Working directory | directory missing                |
| Process           | application exits non-zero       |
| Signal            | process killed                   |
| Dependency        | DB/network unavailable           |

Đó là lý do:

```text
service failed
→ restart
```

không phải complete troubleshooting workflow.

---

# 131. Day 4 command map — Debian/Ubuntu

| Question                      | Command                   |
| ----------------------------- | ------------------------- |
| OS?                           | `cat /etc/os-release`     |
| Refresh metadata?             | `sudo apt update`         |
| Search package?               | `apt search <term>`       |
| Package details?              | `apt show <pkg>`          |
| Install?                      | `sudo apt install <pkg>`  |
| Installed version?            | `dpkg-query -W <pkg>`     |
| Detailed package status?      | `dpkg-query -s <pkg>`     |
| Package files?                | `dpkg-query -L <pkg>`     |
| Which package owns file?      | `dpkg-query -S <path>`    |
| Remove package?               | `sudo apt remove <pkg>`   |
| Purge config too?             | `sudo apt purge <pkg>`    |
| Verify package file contents? | `dpkg -V <pkg>`           |
| Service state?                | `systemctl status <unit>` |

---

# 132. Day 4 command map — RHEL family

| Question                | Command                   |
| ----------------------- | ------------------------- |
| OS?                     | `cat /etc/os-release`     |
| Repositories?           | `dnf repolist`            |
| Search?                 | `dnf search <term>`       |
| Info?                   | `dnf info <pkg>`          |
| Install?                | `sudo dnf install <pkg>`  |
| Installed?              | `rpm -q <pkg>`            |
| Package info?           | `rpm -qi <pkg>`           |
| Package files?          | `rpm -ql <pkg>`           |
| File ownership?         | `rpm -qf <path>`          |
| Package providing path? | `dnf provides <path>`     |
| Remove?                 | `sudo dnf remove <pkg>`   |
| Verify installed files? | `rpm -V <pkg>`            |
| Transaction history?    | `dnf history`             |
| Service state?          | `systemctl status <unit>` |

---

# 133. Các câu phải phân biệt tuyệt đối

Sau Day 4, bạn phải không nhầm:

| A                 | B                       |
| ----------------- | ----------------------- |
| repository        | installed package       |
| package           | executable              |
| package           | service                 |
| package installed | service active          |
| active            | enabled                 |
| `apt update`      | `apt upgrade`           |
| `apt remove`      | `apt purge`             |
| APT               | dpkg                    |
| DNF               | RPM                     |
| file exists       | file executable         |
| executable exists | package owns executable |
| unit loaded       | process healthy         |
| process exists    | application healthy     |
| `reload`          | `daemon-reload`         |
| `restart`         | `enable`                |
| failed state      | root cause              |
| reinstall         | diagnosis               |
| rollback          | generic “undo magic”    |

---

# 134. Exam Focus — Package manager architecture

Nếu hỏi:

> APT khác `dpkg` như thế nào?

Một câu trả lời mạnh:

```text
APT là high-level package management layer dùng repository metadata,
dependency resolution và transaction orchestration.

dpkg là lower-level Debian package management tool làm việc với
installed package database và .deb package operations.
```

Official Ubuntu documentation cũng mô tả APT là high-level CLI và `dpkg` là lower/medium-level package manager. :chatgpt-content-reference{index="40"}

---

# 135. Exam Focus — DNF vs RPM

Một câu trả lời mạnh:

```text
DNF handles repositories, dependency solving and transactions.

RPM maintains and queries RPM packages/package database
and can install/query/verify individual RPM packages.
```

Trong RHEL 9, DNF là recommended software-management utility. :chatgpt-content-reference{index="41"}

---

# 136. Exam Focus — `apt update`

Nếu hỏi:

> `apt update` có update software không?

Đáp án:

````text
It refreshes package information/metadata from configured sources.
It does not by itself perform the normal package upgrade transaction.
``` :chatgpt-content-reference{index="42"}


---

# 137. Exam Focus — remove vs purge

Đáp án:

```text
remove removes package payload but normally leaves certain modified
configuration behind.

purge removes the package plus package-managed configuration more
completely.
````

Nhưng không nói:

```text
purge deletes every application data file everywhere
```

vì không đúng. :chatgpt-content-reference{index="43"}

---

# 138. Exam Focus — package file ownership

Debian:

```bash
dpkg-query -S /path/file
```

RPM:

```bash
rpm -qf /path/file
```

Question:

> `/usr/bin/foo` đến từ package nào?

Đó là hai commands cần nhớ.

---

# 139. Exam Focus — file inventory

Debian:

```bash
dpkg-query -L package
```

RPM:

```bash
rpm -ql package
```

Nhưng `dpkg-query -L` có caveat không nhất thiết list files tạo thêm bởi maintainer scripts. :chatgpt-content-reference{index="44"}

---

# 140. Exam Focus — package installed nhưng service failed

Correct answer không phải:

```text
reinstall package immediately
```

Correct approach:

```text
confirm installed package/version
inspect unit
inspect ExecStart
inspect filesystem path
inspect User/Group/permissions
inspect WorkingDirectory
inspect Environment
inspect MainPID/result/log evidence
identify root cause
perform minimal remediation
validate
```

Đây là integrated Day 4 reasoning.

---

# 141. Exam Focus — service startup

Nếu package ships service:

```text
package installation
```

không tự động chứng minh:

```text
service active and enabled
```

Verify separately:

```bash
systemctl is-active ...
systemctl is-enabled ...
```

---

# 142. Exam Focus — security

Nếu APT/DNF báo signature verification failure, đáp án không phải:

```text
disable signature checks
```

Correct operations posture:

```text
investigate repository/key/trust configuration and use approved,
authenticated sources.
```

APT hiện explicitly discourages insecure repositories. :chatgpt-content-reference{index="45"}

---

# 143. Knowledge Check

Hãy thử trả lời 20 câu này mà không nhìn lại phía trên.

|   # | Câu hỏi                                                        |
| --: | -------------------------------------------------------------- |
|   1 | Package khác executable như thế nào?                           |
|   2 | Package name có phải luôn bằng service name không?             |
|   3 | APT và dpkg khác vai trò gì?                                   |
|   4 | DNF và RPM khác vai trò gì?                                    |
|   5 | `apt update` thực hiện gì?                                     |
|   6 | `apt update` có giống `apt upgrade` không?                     |
|   7 | Làm sao xem package đã cài file nào trên Debian?               |
|   8 | Làm sao tìm package sở hữu `/usr/bin/foo` trên Debian?         |
|   9 | Làm sao làm hai việc tương đương trên RPM?                     |
|  10 | Tại sao phải review transaction trước install/remove?          |
|  11 | `remove` khác `purge` thế nào?                                 |
|  12 | Tại sao không chạy `autoremove -y` mù quáng?                   |
|  13 | Package installed có chứng minh service active không?          |
|  14 | Service active có chứng minh application healthy không?        |
|  15 | Vì sao EnvironmentFile change chưa tác động process đang chạy? |
|  16 | Khi nào cần `daemon-reload`?                                   |
|  17 | Signature verification failure nên xử lý theo hướng nào?       |
|  18 | Package reinstall có phải universal fix không?                 |
|  19 | Vì sao package upgrade có thể cần service restart?             |
|  20 | Day 4 recovery workflow chuẩn là gì?                           |

---

# 144. Knowledge Check — Answer Key

**1.** Package là software distribution unit chứa files + metadata + dependencies và có thể có installation logic. Executable chỉ là một file có thể được package cài.

**2.** Không. Package, executable và service unit có thể có tên khác nhau.

**3.** APT làm high-level repository/dependency/transaction management; `dpkg` làm lower-level Debian package/database operations.

**4.** DNF làm high-level repository/dependency transaction management; RPM quản lý/query/verify RPM package/database.

**5.** Refresh package metadata từ configured sources. :chatgpt-content-reference{index="46"}

**6.** Không. Upgrade thay đổi installed software.

**7.**

```bash
dpkg-query -L package
```

**8.**

```bash
dpkg-query -S /usr/bin/foo
```

**9.**

```bash
rpm -ql package
rpm -qf /usr/bin/foo
```

**10.** Vì dependency solver có thể đề xuất thay đổi nhiều packages hơn package bạn trực tiếp request.

**11.** `remove` thường giữ một số package configuration; `purge` remove config sâu hơn. :chatgpt-content-reference{index="47"}

**12.** Vì proposed automatic dependencies có thể bao gồm software bạn vẫn muốn giữ; phải review. :chatgpt-content-reference{index="48"}

**13.** Không.

**14.** Không.

**15.** Running process đã được exec với environment của thời điểm startup; external file change không live-inject vào process.

**16.** Khi unit definition/drop-ins thay đổi, không phải chỉ vì contents của normal external application/environment config thay đổi.

**17.** Investigate trust/repository/key; không bypass security mặc định.

**18.** Không. Wrong unit/env/permissions/dependency/config không được sửa chỉ bằng reinstall.

**19.** Running process có thể vẫn giữ old process image/libraries/config context dù files trên disk được cập nhật.

**20.**

```text
symptom
→ expected state
→ recent change/scope
→ evidence
→ failing layer
→ hypothesis
→ safe test
→ root cause
→ minimal remediation
→ validate
→ preserve evidence
```

---

# 145. Independent Practical Challenge — Day 4 Final

Đây là challenge độc lập. Hãy làm mà không copy Lab từng dòng.

Bạn cần chọn một **harmless CLI package** từ approved distro repository, cài nó và chứng minh:

```text
repository/package identified
exact version recorded
package installed
binary path known
binary belongs to that package
binary executes
```

Sau đó tạo:

```text
/etc/day4-final.env
/etc/systemd/system/day4-final.service
/opt/day4-final/
```

Service phải:

```text
run as non-root
have explicit WorkingDirectory
use EnvironmentFile
depend at runtime on your installed binary
run continuously
use Restart=on-failure
be enabled through multi-user.target
```

Sau startup bạn phải chứng minh:

```text
LoadedState
ActiveState
SubState
UnitFileState
MainPID
PID/PPID
process owner
process state
working directory
environment
functional output
```

Sau đó inject một **configuration-only failure**; không delete package files, không `chmod 777`, không run root.

Bạn phải produce evidence showing:

```text
before failure state
injected change
failure symptom
service state
exit/result evidence
package still installed
binary still present
actual root cause
remediation
new process
correct environment
functional output restored
```

Cuối cùng cleanup mà không remove bất kỳ pre-existing package.

Nếu bạn hoàn thành challenge này bằng reasoning thay vì copy command, bạn đã thực sự đạt practical objective của Day 4.

---

# 146. Day 4 Mastery Model

Toàn bộ Day 4 bây giờ kết nối lại thành một hệ thống duy nhất:

```text
                   SOFTWARE REPOSITORY
                          │
                          ↓
                    PACKAGE MANAGER
                   APT            DNF
                    │              │
                    ↓              ↓
                  dpkg            RPM
                    │              │
                    └──────┬───────┘
                           ↓
                    INSTALLED FILES
                           │
              executable / config / unit
                           │
                           ↓
                      SYSTEMD UNIT
                           │
              ┌────────────┼─────────────┐
              │            │             │
            User      WorkingDirectory Environment
              │            │             │
              └────────────┼─────────────┘
                           ↓
                       ExecStart
                           │
                           ↓
                        PROCESS
                           │
             ┌─────────────┼──────────────┐
             │             │              │
            PID           PPID           UID
             │
           STATE
      R / S / D / T / Z
             │
          SIGNALS
   TERM / KILL / STOP / CONT
             │
             ↓
          LIFECYCLE
 start / stop / restart / failure
             │
             ↓
          RECOVERY
```

Và operational troubleshooting chain:

```text
SYMPTOM
   ↓
WHAT SHOULD BE HAPPENING?
   ↓
WHAT CHANGED?
   ↓
WHAT IS THE SCOPE?
   ↓
PACKAGE?
   ↓
FILE?
   ↓
PERMISSION?
   ↓
UNIT?
   ↓
ENVIRONMENT?
   ↓
PROCESS?
   ↓
SIGNAL/EXIT?
   ↓
EVIDENCE
   ↓
HYPOTHESIS
   ↓
SAFEST TEST
   ↓
ROOT CAUSE
   ↓
L1 REMEDIATION OR ESCALATION
   ↓
VALIDATE
   ↓
PRESERVE EVIDENCE / HANDOVER
```

# 147. Day 4 — Completion Criteria

Theo syllabus, Day 4 yêu cầu không chỉ lý thuyết mà còn assignment thực tế: **install package, create/manage systemd service, inspect processes, recover a failed service**. :chatgpt-content-reference{index="49"}

Vì vậy mình chỉ coi **Day 4 practical mastery** đạt được khi bạn thực sự có thể chứng minh bốn năng lực đó bằng evidence.

Bạn phải có thể cài software từ đúng repository và xác minh package/version/file ownership; tự tạo và quản lý một `systemd service`; đi từ service xuống PID, PPID, user, state, cwd và environment; rồi cố tình tạo một failure an toàn, tìm root cause bằng evidence, sửa đúng layer, phục hồi và chứng minh service/process/functionality đã trở lại.

Nếu mới đọc hết Module 1–3 nhưng chưa thực hành, thì **kiến thức Day 4 đã được cover, nhưng practical mastery chưa được chứng minh**. Đây là distinction mình sẽ giữ xuyên suốt khóa học.

Bước học hợp lý tiếp theo là **thực hành Day 4 integrated lab trên máy của bạn**, trong đó bạn gửi lại từng nhóm command/output quan trọng để mình review evidence, bắt lỗi reasoning và chỉ ra chỗ nào chưa đạt mastery trước khi sang Day 5.
