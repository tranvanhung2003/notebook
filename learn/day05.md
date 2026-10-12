# Day 5 — Linux Networking + Storage Operations & Troubleshooting

Hôm nay chúng ta học một ngày rất quan trọng đối với công việc Administration/Operations. Đây là ngày bạn bắt đầu chuyển từ việc “biết Linux command” sang khả năng **dùng bằng chứng để xác định layer nào đang hỏng**.

Theo MASTER_SYLLABUS, Day 5 bắt buộc gồm:

| Phần            | Nội dung bắt buộc                                                                                                  |
| --------------- | ------------------------------------------------------------------------------------------------------------------ |
| Concept/Lecture | **IP/DNS/ports, `curl`, `ss`, `ping`, `traceroute`, firewall basics, disks, filesystems, mounts, capacity checks** |
| Assignment/Lab  | **Diagnose a simulated connectivity/storage issue and produce a health-check record**                              |

Đây là scope chính thức của Day 5. 00_MASTER_SYLLABUS_Java_Fresher… Syllabus là nguồn curriculum bắt buộc và không được bỏ hay thay thế nội dung. 00_MASTER_SYLLABUS_Java_Fresher…

Mình sẽ dạy toàn bộ phần bắt buộc, đồng thời bổ sung một số kiến thức vận hành cần thiết. Những phần vượt khỏi câu chữ trực tiếp của syllabus sẽ được đánh dấu **Supplementary / Beyond the explicit syllabus**.

---

# 1. Mục tiêu sau Day 5

Khi hoàn thành tốt Day 5, bạn không chỉ phải biết chạy:

```bash
ping
curl
ss
traceroute
df
lsblk
mount
findmnt
```

mà phải có khả năng nhận một incident kiểu:

> “Không truy cập được application trên port 8080.”

và không đoán mò “firewall chặn rồi”.

Bạn cần suy luận theo chuỗi:

```text
Interface
   ↓
IP address
   ↓
Routing
   ↓
DNS
   ↓
Network path
   ↓
Host firewall
   ↓
Listening socket
   ↓
Application
```

Tương tự, khi nhận:

> “Application không ghi được file.”

không được lập tức kết luận “permission”.

Bạn phải kiểm tra:

```text
Block device
    ↓
Filesystem
    ↓
Mount
    ↓
Capacity / inode
    ↓
Read-only/read-write state
    ↓
Permissions
    ↓
Application
```

Đó chính là tư duy Operations mà Day 5 xây dựng.

---

# 2. Giả định môi trường lab

Các ví dụ dưới đây dành cho Linux hiện đại sử dụng `systemd`, `iproute2`, `util-linux` và GNU coreutils.

Có thể áp dụng tốt cho:

```text
Ubuntu / Debian family
Rocky Linux / AlmaLinux / RHEL family
Amazon Linux
```

Tên package và firewall implementation có thể khác nhau.

Ví dụ:

```bash
# Ubuntu/Debian
sudo apt install curl iputils-ping traceroute

# RHEL/Rocky/AlmaLinux
sudo dnf install curl iputils traceroute
```

Không cần cài tất cả ngay nếu máy bạn đã có.

---

# 3. Mental Model 1 — Một network request thực sự đi như thế nào?

Giả sử bạn chạy:

```bash
curl http://app.example.com:8080/health
```

Rất nhiều thứ phải hoạt động trước khi application trả về HTTP response.

```text
curl
 │
 │ 1. Resolve app.example.com
 ▼
DNS resolver
 │
 │ → 10.20.30.40
 ▼
Linux routing table
 │
 │ "10.20.30.40 phải đi qua interface nào?"
 ▼
Network interface
 │
 │ Ethernet/Wi-Fi/VPC/etc.
 ▼
Intermediate network
 │
 ▼
Destination host
 │
 ▼
Firewall
 │
 ▼
TCP port 8080
 │
 ▼
Listening socket
 │
 ▼
Java/Tomcat/application
 │
 ▼
HTTP response
```

Do đó:

```text
curl failed
```

không đồng nghĩa:

```text
application failed
```

Có thể DNS chưa resolve được.

Có thể route sai.

Có thể packet bị firewall drop.

Có thể network tới được host nhưng không có process listen port.

Có thể process listen nhưng chỉ listen trên `127.0.0.1`.

Có thể TCP thành công nhưng HTTP application trả `500`.

Operations engineer phải phân biệt được từng trường hợp.

---

# 4. IP address

## 4.1 IP dùng để làm gì?

IP address xác định endpoint tại network layer.

Ví dụ:

```text
192.168.10.25
10.0.1.15
172.16.4.20
```

IPv4 address dài 32 bit.

Một interface có thể có:

```text
IPv4
IPv6
nhiều IP cùng lúc
```

Ví dụ:

```bash
ip addr
```

hoặc dễ đọc hơn:

```bash
ip -br addr
```

Ví dụ output:

```text
lo      UNKNOWN   127.0.0.1/8 ::1/128
ens33   UP        192.168.56.20/24 fe80::...
```

`ip` là công cụ của `iproute2` dùng để xem/manipulate interfaces, addresses, routes và nhiều đối tượng networking khác. [man7.org](https://man7.org/linux/man-pages/man8/ip.8.html?utm_source=chatgpt.com)

---

# 5. Interface, loopback và bind address

Giả sử:

```text
ens33 = 192.168.56.20
```

Application có thể listen:

```text
127.0.0.1:8080
192.168.56.20:8080
0.0.0.0:8080
```

Ba trường hợp này rất khác nhau.

| Listen address       | Ý nghĩa                                                      |
| -------------------- | ------------------------------------------------------------ |
| `127.0.0.1:8080`     | Chỉ nhận IPv4 connection qua loopback                        |
| `192.168.56.20:8080` | Chỉ listen IP/interface cụ thể                               |
| `0.0.0.0:8080`       | Listen trên mọi IPv4 local address                           |
| `[::]:8080`          | IPv6 wildcard; IPv4 behavior còn tùy OS/socket configuration |

Đây là nguyên nhân kinh điển:

```text
curl localhost:8080 → OK
máy khác curl server:8080 → FAIL
```

Có thể application không hỏng.

Có thể firewall không hỏng.

Application đơn giản chỉ đang bind:

```text
127.0.0.1
```

thay vì interface cần phục vụ client.

---

# 6. CIDR và subnet

Bạn sẽ thường thấy:

```text
192.168.1.10/24
10.0.5.12/16
10.20.30.40/32
```

Phần `/24`, `/16`, `/32` là prefix length.

Ví dụ:

```text
192.168.1.10/24
```

tương ứng subnet mask:

```text
255.255.255.0
```

Network:

```text
192.168.1.0/24
```

Một fresher không cần Day 5 tính subnet cực sâu, nhưng phải hiểu:

```text
IP address ≠ subnet ≠ default gateway
```

và IP của destination có thể:

```text
cùng local subnet
```

hoặc:

```text
cần routing qua gateway
```

---

# 7. Routing

Linux cần quyết định:

> Packet tới IP X phải gửi qua interface/gateway nào?

Kiểm tra:

```bash
ip route
```

Ví dụ:

```text
default via 192.168.1.1 dev ens33
192.168.1.0/24 dev ens33 proto kernel scope link src 192.168.1.20
```

Nghĩa là:

```text
192.168.1.0/24
→ trực tiếp qua ens33

các destination khác
→ gateway 192.168.1.1
```

`ip route` thao tác và hiển thị kernel routing tables. [man7.org](https://www.man7.org/linux/man-pages/man8/ip-route.8.html?utm_source=chatgpt.com)

Một lệnh cực kỳ hữu ích:

```bash
ip route get 8.8.8.8
```

Ví dụ:

```text
8.8.8.8 via 192.168.1.1 dev ens33 src 192.168.1.20
```

Nó trả lời:

> Nếu Linux gửi packet tới 8.8.8.8 thì route nào sẽ được chọn?

---

# 8. DNS — hostname không phải IP

Giả sử:

```bash
curl https://example.com
```

Application không gửi packet tới chữ `example.com`.

Trước hết hệ thống cần biến:

```text
example.com
```

thành IP address.

Đó là DNS resolution.

Conceptually:

```text
hostname
   ↓
resolver configuration
   ↓
DNS server
   ↓
A / AAAA record
   ↓
IP address
```

Linux truyền thống sử dụng `/etc/resolv.conf` làm resolver configuration; file này chứa thông tin để resolver tìm DNS servers và các tùy chọn resolution. [man7.org](https://www.man7.org/linux/man-pages/man5/resolver.5.html?utm_source=chatgpt.com)

Kiểm tra:

```bash
cat /etc/resolv.conf
```

Ví dụ:

```text
nameserver 192.168.1.1
search example.internal
```

Nhưng trên máy sử dụng `systemd-resolved`, `/etc/resolv.conf` có thể là symlink được quản lý tự động.

Không nên sửa file này bừa bãi trước khi xác định ai đang quản lý network configuration.

---

# 9. Kiểm tra DNS đúng cách

Lệnh phù hợp để kiểm tra resolution theo cách application thường sử dụng:

```bash
getent hosts example.com
```

Ví dụ:

```text
93.184.216.34 example.com
```

Trên systemd-resolved:

```bash
resolvectl status
```

và:

```bash
resolvectl query example.com
```

**Supplementary / Beyond the explicit syllabus:** `getent` đặc biệt hữu ích vì nó đi qua Name Service Switch, nên gần với behavior mà application dùng hơn việc chỉ query một DNS server thủ công bằng `dig`.

---

# 10. Phân biệt DNS issue và network issue

So sánh:

```bash
curl http://10.10.10.20:8080
```

với:

```bash
curl http://app.example.com:8080
```

Nếu IP hoạt động nhưng hostname không hoạt động:

```text
network path có khả năng OK
port có khả năng OK
application có khả năng OK

→ tập trung DNS / name resolution
```

Nếu cả IP và hostname đều fail:

```text
không nên đổ lỗi DNS ngay
```

---

# 11. Ports

Một host có IP nhưng một host có thể chạy hàng chục network services.

Port giúp xác định service.

Ví dụ phổ biến:

|       Port | Service thường gặp                                                |
| ---------: | ----------------------------------------------------------------- |
|     22/TCP | SSH                                                               |
| 53/TCP/UDP | DNS                                                               |
|     80/TCP | HTTP                                                              |
|    443/TCP | HTTPS                                                             |
|   5432/TCP | PostgreSQL                                                        |
|   8080/TCP | thường được dùng cho HTTP/application/Tomcat nhưng không bắt buộc |

Quan trọng:

> Port number không tự chứng minh service nào đang chạy.

Port `8080` hoàn toàn có thể là Python web server, Java application hoặc service khác.

---

# 12. TCP và UDP

Day 5 cần biết mức Operations.

TCP:

```text
connection-oriented
reliable ordered byte stream
```

Ví dụ:

```text
SSH
HTTP/HTTPS
PostgreSQL
```

UDP:

```text
connectionless datagrams
không có TCP-style connection establishment
```

DNS thường có thể sử dụng cả UDP và TCP tùy tình huống.

Do đó khi firewall rule chỉ mở:

```text
53/tcp
```

không thể mặc định kết luận DNS đã được cấu hình đầy đủ.

---

# 13. Socket là gì?

Socket có thể hiểu đơn giản là endpoint giao tiếp mà kernel quản lý.

Một server application muốn nhận request phải tạo socket và bind tới:

```text
IP + port + protocol
```

Ví dụ:

```text
TCP 0.0.0.0:8080
```

Sau đó socket ở trạng thái:

```text
LISTEN
```

Đây chính là lý do `ss` cực quan trọng.

---

# 14. `ss` — công cụ kiểm tra socket

`ss` dùng để xem socket statistics và có thể hiển thị TCP state chi tiết. [man7.org](https://www.man7.org/linux/man-pages/man8/ss.8.html?utm_source=chatgpt.com)

Command cơ bản nhất:

```bash
ss -lnt
```

Giải thích:

```text
-l = listening
-n = numeric, không resolve service/hostname
-t = TCP
```

Thường dùng hơn:

```bash
sudo ss -lntp
```

`-p` hiển thị process information khi quyền cho phép.

Ví dụ:

```text
State  Local Address:Port  Process
LISTEN 127.0.0.1:8080     users:(("java",pid=1234,...))
```

Ta lập tức biết:

```text
Java đang chạy
port 8080 đang listen
nhưng chỉ listen loopback
```

---

# 15. Những command `ss` bạn phải thuộc

```bash
ss -lnt
```

TCP listening sockets.

```bash
ss -lnu
```

UDP sockets.

```bash
sudo ss -lntp
```

TCP listener + process.

```bash
ss -tn
```

TCP connections.

Tìm port:

```bash
sudo ss -lntp | grep ':8080'
```

Một cách tốt hơn khi có filter support:

```bash
ss -lntp 'sport = :8080'
```

---

# 16. `ping`

`ping` gửi ICMP Echo Request và chờ Echo Reply. Linux `ping` hiện hỗ trợ cả IPv4 và IPv6; `-4` hoặc `-6` có thể ép address family. [man7.org](https://www.man7.org/linux/man-pages/man8/ping.8.html?utm_source=chatgpt.com)

Không nên chạy:

```bash
ping server
```

vô hạn khi chỉ cần test.

Hãy dùng:

```bash
ping -c 4 192.168.1.1
```

Ví dụ:

```text
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=0.8 ms
...
4 packets transmitted, 4 received, 0% packet loss
```

---

# 17. `ping` chứng minh được gì?

Nếu:

```bash
ping 10.0.0.20
```

thành công, có bằng chứng rằng:

```text
ICMP Echo traffic có thể đi tới host
và Echo Reply quay lại
```

Nhưng nó **không chứng minh**:

```text
TCP/8080 đang mở
Tomcat đang chạy
HTTP hoạt động
database hoạt động
```

Ví dụ:

```bash
ping server
```

OK.

Nhưng:

```bash
curl http://server:8080
```

fail.

Đó không phải mâu thuẫn.

Hai command test hai thứ khác nhau.

---

# 18. Ping fail có nghĩa host chết không?

Không.

ICMP có thể bị:

```text
firewall drop
router filter
cloud network controls
security policy
```

trong khi TCP/443 vẫn hoạt động.

Đây là lỗi tư duy fresher cực kỳ phổ biến:

```text
ping timeout
→ server down
```

Không được kết luận như vậy.

Phải kiểm tra đúng protocol/service cần phục vụ.

---

# 19. Exit code của `ping`

Điểm rất hữu ích cho Operations:

```bash
ping -c 1 server
echo $?
```

Theo `ping(8)`, success trả `0`; không nhận được reply sẽ trả `1`, còn một số lỗi khác trả `2`. [man7.org](https://www.man7.org/linux/man-pages/man8/ping.8.html?utm_source=chatgpt.com)

Ví dụ automation:

```bash
if ping -c 1 -W 2 192.168.1.1 >/dev/null; then
    echo "ICMP reachable"
else
    echo "ICMP test failed"
fi
```

Day 6 sẽ dùng exit codes sâu hơn.

---

# 20. `traceroute`

`traceroute` cố xác định các network hops mà packet đi qua bằng cách tăng TTL và quan sát ICMP Time Exceeded hoặc response tương ứng. [man7.org](https://man7.org/linux/man-pages/man8/traceroute.8.html?utm_source=chatgpt.com)

Ví dụ:

```bash
traceroute example.com
```

Ví dụ output:

```text
1  192.168.1.1      0.8 ms
2  10.10.0.1        3.2 ms
3  *
4  203.0.113.1      12.5 ms
```

Dấu:

```text
*
```

không tự động có nghĩa:

```text
packet bị mất hoàn toàn tại hop đó
```

Intermediate device có thể không trả traceroute probe nhưng vẫn forward traffic.

Đừng kết luận route hỏng chỉ từ một dòng `*`.

---

# 21. Ping vs traceroute

| Tool         | Câu hỏi                                               |
| ------------ | ----------------------------------------------------- |
| `ping`       | Destination có trả ICMP Echo không? RTT/loss thế nào? |
| `traceroute` | Các probe đi qua những hop nào và dừng/respond ở đâu? |

Trong troubleshooting:

```text
ping destination fail
```

có thể theo sau bởi:

```bash
traceroute destination
```

để thu thêm evidence.

Nhưng traceroute cũng không phải chân lý tuyệt đối vì firewall/rate limiting có thể ảnh hưởng output.

---

# 22. `curl`

`curl` là công cụ transfer data qua URL và hỗ trợ nhiều protocol, trong đó có HTTP/HTTPS. [curl.se](https://curl.se/docs/manpage.html?code=xxx&utm_source=chatgpt.com)

Với Java/Tomcat sau này, `curl` sẽ là một trong những tool bạn sử dụng nhiều nhất.

Ví dụ:

```bash
curl http://localhost:8080/
```

---

# 23. `curl -v` — một trong những lệnh troubleshooting quan trọng nhất

```bash
curl -v http://server:8080/health
```

`-v` cho bạn thấy nhiều bước:

```text
DNS resolution
IP được chọn
connection attempt
TCP connect
HTTP request
HTTP response headers
```

Ví dụ:

```text
* Host app.example.com:8080 was resolved.
* IPv4: 10.0.1.20
* Trying 10.0.1.20:8080...
* Connected to app.example.com
> GET /health HTTP/1.1
...
< HTTP/1.1 200 OK
```

Đây là evidence cực tốt.

---

# 24. Các loại lỗi `curl` quan trọng

## Lỗi DNS

```text
curl: (6) Could not resolve host
```

Điều tra:

```bash
getent hosts hostname
cat /etc/resolv.conf
resolvectl status
```

## TCP connection refused

Có thể thấy dạng:

```text
curl: (7) Failed to connect ...
Connection refused
```

Thường gợi ý:

```text
host reachable
nhưng không có listener phù hợp
```

hoặc firewall gửi reject.

Kiểm tra:

```bash
ss -lntp
```

## Timeout

Ví dụ connection attempt treo rồi timeout.

Có thể liên quan:

```text
network path
firewall DROP
routing
destination unavailable
```

Nhưng timeout **không tự chứng minh firewall**.

## HTTP 404

TCP đã hoạt động.

HTTP server đã trả response.

Vấn đề có thể là:

```text
URL/context path/resource
```

## HTTP 500

Networking phần lớn đã hoạt động đến application.

Scope chuyển sang:

```text
application/backend/configuration/database/logs
```

Đây là cách “layer isolation”.

---

# 25. Các command `curl` thực tế

Chỉ lấy header:

```bash
curl -I http://server/
```

Verbose:

```bash
curl -v http://server:8080/
```

Hiển thị HTTP status:

```bash
curl -sS -o /dev/null -w '%{http_code}\n' http://server:8080/health
```

Giới hạn connect time:

```bash
curl --connect-timeout 3 http://server:8080/
```

Follow redirect:

```bash
curl -L http://server/
```

HTTPS diagnostic:

```bash
curl -v https://server/
```

Không nên biến:

```bash
curl -k
```

thành thói quen.

`-k/--insecure` bỏ certificate verification và có rủi ro security. Chỉ dùng có kiểm soát khi troubleshooting certificate, không dùng như “fix”.

---

# 26. Quy trình network troubleshooting chuẩn

Khi user báo:

> `http://app01:8080` không truy cập được.

Đừng bắt đầu bằng restart.

Đừng bắt đầu bằng firewall change.

Đừng bắt đầu bằng reboot.

Bắt đầu bằng evidence.

### Bước A — xác định expected behavior

Ví dụ:

```text
Expected:
app01 resolves → 10.0.1.20
TCP/8080 accessible from application subnet
GET /health → HTTP 200
```

### Bước B — kiểm tra local network identity

```bash
ip -br addr
ip route
```

### Bước C — kiểm tra DNS

```bash
getent hosts app01
```

### Bước D — kiểm tra path cơ bản

```bash
ping -c 4 app01
traceroute app01
```

Nếu ICMP bị policy chặn thì không coi đây là bằng chứng quyết định.

### Bước E — kiểm tra socket trên server

```bash
sudo ss -lntp | grep ':8080'
```

### Bước F — kiểm tra application locally

```bash
curl -v http://127.0.0.1:8080/
```

### Bước G — kiểm tra qua interface IP

```bash
curl -v http://SERVER_IP:8080/
```

### Bước H — firewall

Chỉ sau khi biết application đang listen đúng interface/port mới kiểm tra firewall.

Đây là cách giảm guesswork.

---

# 27. Firewall basics

Firewall áp policy lên traffic.

Khái niệm cơ bản:

```text
ALLOW
DROP
REJECT
```

`DROP`:

```text
discard packet
thường không trả response trực tiếp
```

`REJECT`:

```text
từ chối và thường gửi response báo không được phép
```

Không nên đồng nhất:

```text
firewall = port
```

Firewall rule có thể phụ thuộc:

```text
source
destination
protocol
port
interface
zone
connection state
```

---

# 28. firewalld

Trên các môi trường sử dụng `firewalld`, khái niệm quan trọng là **zone**.

Một zone biểu diễn mức trust của network connection/interface/source và xác định services/ports/rules được áp dụng. [firewalld](https://firewalld.org/documentation/man-pages/firewalld.zones?utm_source=chatgpt.com)

Kiểm tra service:

```bash
sudo systemctl status firewalld
```

State:

```bash
sudo firewall-cmd --state
```

Active zones:

```bash
sudo firewall-cmd --get-active-zones
```

Configuration:

```bash
sudo firewall-cmd --list-all
```

---

# 29. Runtime vs permanent firewall configuration

Đây là điểm rất quan trọng.

firewalld phân biệt:

```text
runtime configuration
permanent configuration
```

Ví dụ:

```bash
sudo firewall-cmd --zone=public --add-port=8080/tcp
```

thay đổi runtime.

Trong khi:

```bash
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp
```

ghi permanent configuration. Tài liệu firewalld chỉ rõ runtime và permanent là hai configuration layers riêng biệt. [firewalld](https://firewalld.org/documentation/index.html?utm_source=chatgpt.com)

Đừng nghĩ:

```bash
--permanent
```

nghĩa là:

```text
rule lập tức active trong runtime
```

---

# 30. CẢNH BÁO firewall

Nếu bạn đang SSH vào server:

```text
KHÔNG
```

blindly:

```bash
disable firewall
flush rules
change active zone
reload toàn bộ firewall
```

trước khi xác định blast radius.

Bạn có thể tự khóa chính SSH session của mình và làm mất remote access.

Trước thay đổi firewall production:

```text
1. Ghi lại current state
2. Biết SSH source/port
3. Xác nhận access path
4. Có rollback
5. Có console/out-of-band access nếu cần
6. Thực hiện change nhỏ nhất
7. Test
```

Least privilege cũng áp dụng cho networking:

> Chỉ mở đúng source/protocol/port thực sự cần.

Không phải:

```text
0.0.0.0/0 + all ports
```

---

# 31. Supplementary — nftables / UFW awareness

Syllabus chỉ yêu cầu **firewall basics**, không yêu cầu Day 5 trở thành chuyên gia firewall.

Linux hiện đại có thể sử dụng:

```text
nftables
firewalld
UFW
```

tùy distribution/configuration.

Nếu `firewalld` quản lý firewall, không nên vừa dùng `firewall-cmd` vừa tự sửa backend rules bằng `nft` mà không hiểu ownership.

Trên Ubuntu có thể gặp:

```bash
sudo ufw status verbose
```

Nhưng chúng ta không coi UFW hay firewalld là universal cho mọi distro.

---

# 32. Mental Model 2 — Linux storage stack

Bây giờ chuyển sang nửa thứ hai của Day 5.

Hãy nhớ chuỗi:

```text
Physical / virtual disk
        ↓
Block device
        ↓
Partition
        ↓
Filesystem
        ↓
Mount
        ↓
Directory tree
        ↓
Files
```

Ví dụ:

```text
EBS/SSD
 ↓
/dev/nvme1n1
 ↓
/dev/nvme1n1p1
 ↓
ext4
 ↓
/data
 ↓
/data/app.log
```

---

# 33. Disk và block device

Linux biểu diễn storage devices dưới `/dev`.

Ví dụ:

```text
/dev/sda
/dev/sdb
/dev/vda
/dev/nvme0n1
```

Partition có thể là:

```text
/dev/sda1
/dev/sda2
```

hoặc:

```text
/dev/nvme0n1p1
```

Không được dựa vào tên `/dev/sdb` để đoán “đó chắc chắn là disk data”.

Luôn inspect trước.

---

# 34. `lsblk`

`lsblk` hiển thị block devices và quan hệ giữa chúng; nó lấy thông tin từ sysfs và udev database khi có. [man7.org](https://man7.org/linux/man-pages/man8/lsblk.8.html?utm_source=chatgpt.com)

Chạy:

```bash
lsblk
```

Tốt hơn:

```bash
lsblk -f
```

Ví dụ:

```text
NAME   FSTYPE FSVER LABEL UUID                                 MOUNTPOINTS
sda
├─sda1 ext4   1.0         1111-2222                            /
└─sda2 swap   1           3333-4444                            [SWAP]
sdb
└─sdb1 xfs                5555-6666                            /data
```

Đây cho bạn biết:

```text
device
partition
filesystem type
UUID
mountpoint
```

---

# 35. Filesystem là gì?

Disk chỉ là block storage.

Để lưu:

```text
directories
files
metadata
permissions
timestamps
```

ta cần filesystem.

Linux có nhiều loại:

```text
ext4
XFS
Btrfs
tmpfs
NFS
...
```

Day 5 không cần phân tích implementation internals của ext4/XFS.

Điều quan trọng:

```text
block device ≠ filesystem
filesystem ≠ mountpoint
```

Ví dụ:

```text
/dev/sdb1       = block device
ext4            = filesystem
/data           = mountpoint
```

---

# 36. CẢNH BÁO `mkfs`

Command như:

```bash
mkfs.ext4 /dev/sdb
```

là **destructive**.

Nó tạo filesystem mới và có thể phá dữ liệu hiện có.

Trong production, trước `mkfs` phải xác định:

```text
device identity
existing filesystem
mount status
data ownership
backup
change approval
blast radius
```

Không bao giờ copy/paste `mkfs` vào device thật chỉ vì tutorial bảo làm.

Trong lab hôm nay chúng ta sẽ dùng **file-backed loop device riêng** để tránh động tới disk thật.

---

# 37. Mount là gì?

Linux có một unified directory tree bắt đầu từ:

```text
/
```

`mount` attach filesystem vào một directory trong tree đó. Đây chính là behavior được mô tả trong `mount(8)`. [man7.org](https://man7.org/linux/man-pages/man8/mount.8.html?utm_source=chatgpt.com)

Ví dụ:

```bash
sudo mount /dev/sdb1 /data
```

Sau đó:

```text
/data
```

đại diện cho root của filesystem nằm trên `/dev/sdb1`.

---

# 38. Một trap cực quan trọng của mount

Giả sử trước khi mount:

```text
/data/a.txt
/data/b.txt
```

Sau đó bạn:

```bash
mount /dev/sdb1 /data
```

Những file cũ **không nhất thiết bị xóa**.

Nhưng chúng bị filesystem mới mounted ở `/data` che khuất.

Tài liệu `mount` cũng chỉ rõ khi filesystem được attach vào directory, previous contents ở mountpoint trở nên invisible trong thời gian mount tồn tại. [man7.org](https://man7.org/linux/man-pages/man8/mount.8.html?utm_source=chatgpt.com)

Đây là lỗi rất hay gây nhầm:

> “Mount xong data cũ biến mất!”

Có thể data chỉ đang bị mount che.

---

# 39. Kiểm tra mount bằng `findmnt`

Không nên dựa chỉ vào:

```bash
mount | grep ...
```

`findmnt` được thiết kế để list/search mounted filesystems, đồng thời có thể tra `/etc/fstab` và kernel mount information. [man7.org](https://www.man7.org/linux/man-pages/man8/findmnt.8.html?utm_source=chatgpt.com)

Ví dụ:

```bash
findmnt
```

Kiểm tra filesystem chứa `/data`:

```bash
findmnt /data
```

Hoặc chính xác:

```bash
findmnt --target /data
```

Ví dụ:

```text
TARGET SOURCE    FSTYPE OPTIONS
/data  /dev/sdb1 xfs    rw,relatime,...
```

---

# 40. Temporary mount vs persistent mount

Lệnh:

```bash
sudo mount /dev/sdb1 /data
```

tạo runtime mount.

Sau reboot nó không tự động có lại trừ khi hệ thống có persistent configuration phù hợp.

Trên Linux truyền thống:

```text
/etc/fstab
```

mô tả static filesystem configuration. [man7.org](https://www.man7.org/linux/man-pages/man5/fstab.5.html?utm_source=chatgpt.com)

Ví dụ:

```text
UUID=1111-2222  /data  ext4  defaults  0  2
```

Không nên dùng `/dev/sdb1` một cách máy móc nếu UUID phù hợp hơn. Tài liệu `fstab` khuyến nghị LABEL/UUID vì device names có thể thay đổi theo hardware detection order. [man7.org](https://www.man7.org/linux/man-pages/man5/fstab.5.html)

---

# 41. Sáu field chính của `/etc/fstab`

Ví dụ:

```text
UUID=abc123 /data ext4 defaults 0 2
```

Hiểu thành:

```text
1. source
2. mountpoint
3. filesystem type
4. mount options
5. dump field
6. fsck pass
```

Tài liệu hiện hành mô tả chính xác cấu trúc đó và các option như `defaults`, `noauto`, `nofail`. [man7.org](https://www.man7.org/linux/man-pages/man5/fstab.5.html)

---

# 42. CẢNH BÁO khi sửa `/etc/fstab`

Sai `fstab` có thể gây:

```text
mount failure
service dependency failure
boot problem
```

Trước thay đổi:

```bash
sudo cp -a /etc/fstab /etc/fstab.bak
```

Sau khi edit:

```bash
sudo findmnt --verify --verbose
```

`findmnt --verify` hiện kiểm tra parsability/usability của `/etc/fstab`. [man7.org](https://www.man7.org/linux/man-pages/man8/findmnt.8.html)

Trên systemd-based systems, tài liệu `fstab` hiện cũng khuyến nghị:

```bash
sudo systemctl daemon-reload
```

sau khi sửa `fstab`. [man7.org](https://www.man7.org/linux/man-pages/man5/fstab.5.html)

Sau đó mới kiểm tra có kiểm soát.

---

# 43. Capacity check — `df`

`df` báo filesystem space usage. Nếu không truyền path, nó hiển thị mounted filesystems; nếu truyền path, nó báo filesystem chứa path đó. [man7.org](https://man7.org/linux/man-pages/man1/df.1.html?utm_source=chatgpt.com)

Lệnh bạn phải thuộc:

```bash
df -h
```

Ví dụ:

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1        40G   20G   18G  53% /
/dev/sdb1       100G   94G    6G  94% /data
```

Điểm cần chú ý:

```text
Use%
```

---

# 44. Kiểm tra filesystem cụ thể

Thay vì:

```bash
df -h
```

rồi nhìn cả đống output:

```bash
df -h /data
```

Đây là habit tốt trong troubleshooting.

Ví dụ:

```bash
df -h /var/lib/myapp
```

cho biết filesystem thực sự chứa path của application.

---

# 45. Capacity ≠ chỉ có bytes

**Supplementary / Beyond the explicit syllabus, nhưng rất quan trọng trong Operations**

Filesystem có thể còn GB free nhưng hết inode.

Kiểm tra:

```bash
df -i
```

Ví dụ:

```text
Filesystem      Inodes   IUsed IFree IUse% Mounted on
/dev/sda1      1000000 1000000     0  100% /
```

Lúc này:

```text
df -h
```

có thể vẫn còn space.

Nhưng filesystem không tạo thêm file được.

Scenario rất thực tế:

```text
application tạo hàng triệu tiny files
→ inode exhaustion
```

---

# 46. `du` và `df` khác nhau

`df`:

```text
filesystem-level usage
```

`du`:

```text
space attributable to files/directories mà nó traverse
```

Ví dụ:

```bash
sudo du -xhd1 /var
```

hoặc:

```bash
du -sh /var/log/*
```

Điều tra:

```text
filesystem 95% full
```

thường dùng:

```bash
df -h /var
sudo du -xhd1 /var
```

Nhưng phải chú ý quyền truy cập và runtime cost trên directory tree rất lớn.

---

# 47. Supplementary — `df` cao nhưng `du` không thấy file lớn

Một case production rất nổi tiếng:

```text
df → filesystem gần full
du → tổng file nhỏ hơn nhiều
```

Một khả năng:

```text
process vẫn giữ file descriptor
đến file đã bị unlink/delete
```

File biến mất khỏi directory tree nhưng space chưa được kernel release vì process còn mở file.

Kiểm tra thường dùng:

```bash
sudo lsof +L1
```

Không được “fix” bằng kill process vô tội vạ.

Phải xác định:

```text
process nào
file nào
service impact
restart có được approve không
```

---

# 48. Read-only filesystem

Nếu write fail với:

```text
Read-only file system
```

đừng nhầm với:

```text
Permission denied
```

Kiểm tra:

```bash
findmnt -T /path
```

hoặc:

```bash
findmnt --target /path
```

xem mount options có:

```text
ro
```

hay:

```text
rw
```

Filesystem có thể chuyển read-only do lỗi nghiêm trọng.

Đây thường là situation cần thận trọng và có thể escalation, không phải “mount -o remount,rw” ngay lập tức.

---

# 49. Command matrix Day 5

| Command                   | Mục đích                  |    Thường cần root? | Có thay đổi state? |
| ------------------------- | ------------------------- | ------------------: | -----------------: |
| `ip -br addr`             | xem interface/IP          |               Không |              Không |
| `ip route`                | xem route                 |               Không |              Không |
| `getent hosts name`       | kiểm tra name resolution  |               Không |              Không |
| `ping -c 4 host`          | ICMP test                 |        Thường không |      Không đáng kể |
| `traceroute host`         | xem network hops          |        Thường không |              Không |
| `ss -lnt`                 | xem TCP listeners         |               Không |              Không |
| `sudo ss -lntp`           | listener + process        |     Có lợi khi root |              Không |
| `curl -v URL`             | test protocol/application |               Không |    Network request |
| `firewall-cmd --list-all` | firewall inspection       |         Thường sudo |              Không |
| `lsblk -f`                | block device/filesystem   |               Không |              Không |
| `findmnt`                 | mount inspection          |               Không |              Không |
| `df -h`                   | byte capacity             |               Không |              Không |
| `df -i`                   | inode capacity            |               Không |              Không |
| `du`                      | directory usage           | Không/sudo tùy path |              Không |
| `mount`                   | attach filesystem         |         Thường root |             **Có** |
| `umount`                  | detach filesystem         |         Thường root |             **Có** |
| `mkfs.*`                  | tạo filesystem            |                root |    **Destructive** |
| `losetup`                 | loop device management    |                root |             **Có** |

Điểm quan trọng trong Operations:

> Ưu tiên read-only inspection trước state-changing commands.

---

# 50. Guided Lab Day 5

Bây giờ tới phần bắt buộc của syllabus:

> **Diagnose a simulated connectivity/storage issue and produce a health-check record.**

Ta sẽ làm hai incident nhỏ:

```text
Incident A — service chỉ bind localhost
Incident B — filesystem bị full
```

Hai lab này được thiết kế để hạn chế ảnh hưởng tới máy thật.

---

# 51. LAB A — Connectivity incident

## Objective

Bạn sẽ tạo HTTP service:

```text
TCP/8080
```

nhưng cố tình bind:

```text
127.0.0.1
```

Sau đó điều tra tại sao:

```text
localhost works
interface IP fails
```

và xác định root cause bằng `ss` + `curl`.

---

# 52. Chuẩn bị evidence directory

```bash
mkdir -p ~/day5-lab/{evidence,www}
cd ~/day5-lab
```

Ghi baseline:

```bash
date | tee evidence/00-date.txt
uname -a | tee evidence/01-uname.txt
ip -br addr | tee evidence/02-ip-address.txt
ip route | tee evidence/03-route.txt
cat /etc/resolv.conf | tee evidence/04-resolver.txt
```

Health file:

```bash
echo 'DAY5_HEALTH_OK' > ~/day5-lab/www/health.txt
```

---

# 53. Start service với lỗi được cố tình inject

Chạy:

```bash
cd ~/day5-lab/www
python3 -m http.server 8080 --bind 127.0.0.1 \
  >~/day5-lab/evidence/http-server.log 2>&1 &
```

Ghi PID:

```bash
echo $! | tee ~/day5-lab/evidence/http-server.pid
```

---

# 54. Expected invariant

Đây là **Expected invariant**, không phải example output:

```text
Có process LISTEN TCP port 8080
Listener chỉ bind 127.0.0.1
curl 127.0.0.1:8080 thành công
curl tới non-loopback server IP không thành công
```

Kiểm tra:

```bash
sudo ss -lntp | grep ':8080' \
  | tee ~/day5-lab/evidence/05-ss-before.txt
```

Bạn dự kiến thấy tương tự:

```text
LISTEN ... 127.0.0.1:8080 ...
```

Đây là **Example output**; PID/process text thực tế sẽ khác.

---

# 55. Validate localhost

```bash
curl -v http://127.0.0.1:8080/health.txt \
  2>&1 | tee ~/day5-lab/evidence/06-curl-loopback.txt
```

Expected:

```text
HTTP request thành công
body chứa DAY5_HEALTH_OK
```

---

# 56. Lấy non-loopback IP

Chạy:

```bash
ip -4 -br addr
```

Tìm interface thật, ví dụ:

```text
ens33 UP 192.168.56.20/24
```

Trong ví dụ bên dưới ta giả định:

```text
192.168.56.20
```

**Không copy IP mẫu. Dùng IP thật của VM bạn.**

Test:

```bash
curl -v --connect-timeout 3 \
  http://192.168.56.20:8080/health.txt
```

Expected:

```text
FAIL
```

trong lab này.

---

# 57. Troubleshooting reasoning

Symptom:

```text
localhost succeeds
interface IP fails
```

DNS?

Không.

Ta đang dùng IP.

Route tới external destination?

Không phải primary suspect vì request đang nhắm local host.

Application process chết?

Không.

Local curl thành công.

Port không listen?

Không hoàn toàn.

`ss` chứng minh có listener.

Bằng chứng quyết định:

```text
127.0.0.1:8080
```

thay vì:

```text
0.0.0.0:8080
```

Root cause:

```text
service bound only to loopback
```

---

# 58. Remediation

Dừng lab service:

```bash
kill "$(cat ~/day5-lab/evidence/http-server.pid)"
```

Xác nhận:

```bash
ss -lnt | grep ':8080'
```

Không còn listener cũ.

Start lại:

```bash
cd ~/day5-lab/www
python3 -m http.server 8080 --bind 0.0.0.0 \
  >~/day5-lab/evidence/http-server-fixed.log 2>&1 &
```

Ghi PID mới:

```bash
echo $! > ~/day5-lab/evidence/http-server.pid
```

---

# 59. Validation sau remediation

```bash
sudo ss -lntp | grep ':8080' \
  | tee ~/day5-lab/evidence/07-ss-after.txt
```

Expected invariant:

```text
listener = 0.0.0.0:8080
```

Test loopback:

```bash
curl -fsS http://127.0.0.1:8080/health.txt
```

Test interface IP:

```bash
curl -v http://YOUR_IP:8080/health.txt
```

Expected body:

```text
DAY5_HEALTH_OK
```

---

# 60. Security observation của Lab A

`0.0.0.0` nghĩa là application listen trên tất cả local IPv4 interfaces.

Nó **không có nghĩa**:

```text
Internet automatically has access
```

vì firewall/routing/cloud controls còn quyết định reachability.

Nhưng nó làm exposure rộng hơn `127.0.0.1`.

Trong production, bind address phải phù hợp architecture.

Ví dụ reverse proxy architecture có thể cố ý để backend:

```text
127.0.0.1:8080
```

và Nginx/Apache listen externally.

Do đó:

> Bind localhost không phải luôn là lỗi.

Nó chỉ là lỗi khi expected architecture yêu cầu external client kết nối trực tiếp.

---

# 61. Optional firewall observation

Chỉ trên disposable/local VM mà bạn có console access.

Nếu firewalld đang dùng:

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

Đừng mở port nếu không cần.

Nếu lab yêu cầu explicit firewall testing:

```bash
sudo firewall-cmd --zone=public --add-port=8080/tcp
```

Validate:

```bash
sudo firewall-cmd --zone=public --query-port=8080/tcp
```

Rollback:

```bash
sudo firewall-cmd --zone=public --remove-port=8080/tcp
```

Không dùng `--permanent` cho temporary lab nếu không có lý do.

---

# 62. LAB B — Simulated storage-full incident

Bài này có command filesystem creation.

**CẢNH BÁO:** `mkfs` có khả năng phá dữ liệu nếu target sai.

Ta sẽ chỉ chạy `mkfs` trên một loop device vừa được tạo từ file riêng:

```text
~/day5-lab/disk.img
```

Không thay `$LOOPDEV` bằng `/dev/sda`, `/dev/nvme0n1`, `/dev/vda`, EBS thật hay disk chứa OS.

---

# 63. Tạo lab disk image

```bash
truncate -s 128M ~/day5-lab/disk.img
```

File hiện có size 128 MiB logical.

Tạo loop device:

```bash
LOOPDEV=$(sudo losetup --find --show ~/day5-lab/disk.img)
echo "$LOOPDEV"
```

Ví dụ:

```text
/dev/loop0
```

Nhưng device của bạn có thể khác.

---

# 64. Safety check bắt buộc

Trước `mkfs`:

```bash
lsblk -f "$LOOPDEV"
```

Và:

```bash
sudo losetup "$LOOPDEV"
```

Phải thấy backing file:

```text
/home/.../day5-lab/disk.img
```

Nếu không đúng:

```text
DỪNG.
```

Không chạy bước tiếp.

---

# 65. Tạo filesystem

Chỉ sau safety verification:

```bash
sudo mkfs.ext4 -L DAY5LAB "$LOOPDEV"
```

Inspect:

```bash
lsblk -f "$LOOPDEV"
```

Bạn dự kiến thấy:

```text
FSTYPE ext4
LABEL DAY5LAB
UUID ...
```

---

# 66. Mount filesystem

```bash
sudo mkdir -p /mnt/day5lab
sudo mount "$LOOPDEV" /mnt/day5lab
```

Validate:

```bash
findmnt /mnt/day5lab
```

và:

```bash
df -h /mnt/day5lab
```

Evidence:

```bash
lsblk -f | tee ~/day5-lab/evidence/20-lsblk.txt
findmnt /mnt/day5lab | tee ~/day5-lab/evidence/21-findmnt.txt
df -h /mnt/day5lab | tee ~/day5-lab/evidence/22-df-before.txt
```

---

# 67. Inject capacity failure

Ta sẽ cố tình fill **filesystem lab 128 MB**, không phải root filesystem.

Chạy:

```bash
sudo dd if=/dev/zero \
  of=/mnt/day5lab/fill.bin \
  bs=1M \
  status=progress
```

Không có `count`, nên nó sẽ ghi cho tới khi lab filesystem hết chỗ và `dd` báo:

```text
No space left on device
```

Đây là intentional failure.

Nó chỉ fill filesystem mounted ở:

```text
/mnt/day5lab
```

nếu bạn đã xác nhận `findmnt` đúng.

---

# 68. Điều tra failure

Kiểm tra:

```bash
df -h /mnt/day5lab \
  | tee ~/day5-lab/evidence/23-df-full.txt
```

Kiểm tra inode:

```bash
df -i /mnt/day5lab \
  | tee ~/day5-lab/evidence/24-df-inode.txt
```

Xác định file lớn:

```bash
sudo du -sh /mnt/day5lab/* \
  | tee ~/day5-lab/evidence/25-du.txt
```

Bây giờ thử:

```bash
echo "health test" | sudo tee /mnt/day5lab/test.txt
```

Expected:

```text
write failure hoặc filesystem không đủ space
```

---

# 69. Root-cause reasoning

Symptom:

```text
write fails
```

Hypothesis 1:

```text
permissions
```

Có thể.

Nhưng evidence:

```text
df -h → filesystem full
```

Root cause:

```text
byte capacity exhausted
```

`df -i` giúp loại trừ:

```text
inode exhaustion
```

`du` xác định contributor:

```text
fill.bin
```

Đây chính là evidence-driven troubleshooting.

---

# 70. Remediation

Xóa file giả lập:

```bash
sudo rm -f /mnt/day5lab/fill.bin
```

Flush pending writes:

```bash
sync
```

Validate:

```bash
df -h /mnt/day5lab
```

Test write:

```bash
echo "DAY5_STORAGE_OK" \
  | sudo tee /mnt/day5lab/test.txt
```

Read:

```bash
cat /mnt/day5lab/test.txt
```

Expected invariant:

```text
filesystem không còn full
write thành công
read-back đúng
```

---

# 71. Cleanup lab storage

Xóa lab test file:

```bash
sudo rm -f /mnt/day5lab/test.txt
```

Unmount:

```bash
sudo umount /mnt/day5lab
```

Validate:

```bash
findmnt /mnt/day5lab
```

Không còn mount.

Detach loop:

```bash
sudo losetup -d "$LOOPDEV"
```

Xóa image:

```bash
rm -f ~/day5-lab/disk.img
```

Optional:

```bash
sudo rmdir /mnt/day5lab
```

---

# 72. Cleanup network lab

```bash
kill "$(cat ~/day5-lab/evidence/http-server.pid)" 2>/dev/null || true
```

Validate:

```bash
ss -lnt | grep ':8080'
```

Không còn Python lab listener.

Evidence directory nên giữ lại:

```text
~/day5-lab/evidence/
```

để review.

---

# 73. Health-check record bắt buộc

Một Operations health check không nên chỉ ghi:

```text
Server OK.
```

Đó không phải evidence.

Bạn nên ghi:

```text
Timestamp:
Host:
Incident:
Expected behavior:
Observed symptom:

NETWORK
Interface/IP:
Route:
DNS:
Target:
Target port:
Listening socket:
curl result:
ping result:
traceroute result:
Firewall status:

STORAGE
Block device:
Filesystem:
Mountpoint:
Mount options:
Capacity:
Inode:
Largest contributor:

Root cause:
Remediation:
Post-fix validation:
Rollback:
Evidence files:
Escalation required:
```

---

# 74. Ví dụ health-check record cho Lab A

```text
Timestamp:
2026-10-12 17:30 +07

Incident:
HTTP service inaccessible through server interface IP.

Expected:
TCP/8080 reachable through server interface IP.
GET /health.txt returns DAY5_HEALTH_OK.

Observed:
curl 127.0.0.1:8080 succeeded.
curl SERVER_IP:8080 failed.

Evidence:
ip -br addr:
ens33 UP 192.168.56.20/24

ss -lntp:
127.0.0.1:8080 LISTEN python3

Root cause:
HTTP process bound only to IPv4 loopback.

Remediation:
Restarted lab HTTP server with bind address 0.0.0.0.

Validation:
ss shows 0.0.0.0:8080.
curl loopback successful.
curl interface IP successful.
Body returned DAY5_HEALTH_OK.

Rollback:
Stop lab HTTP process.

Escalation:
Not required; issue was within lab/L1 scope.
```

---

# 75. Ví dụ health-check record cho Lab B

```text
Incident:
Writes to /mnt/day5lab fail with ENOSPC.

Expected:
Filesystem mounted read/write and able to create files.

Observed:
df -h shows filesystem capacity exhausted.
df -i shows inode availability remains.
du identifies fill.bin as primary consumer.

Root cause:
Byte capacity exhaustion caused by intentionally generated fill.bin.

Remediation:
Removed fill.bin and synchronized filesystem.

Validation:
df -h shows free capacity.
New file creation succeeds.
Read-back succeeds.

Rollback:
No configuration change required.

Evidence:
20-lsblk.txt
21-findmnt.txt
22-df-before.txt
23-df-full.txt
24-df-inode.txt
25-du.txt
```

Đây mới là Operations evidence.

---

# 76. Troubleshooting matrix — Networking

| Symptom                         | Layer nghi ngờ đầu tiên | Evidence cần lấy             |
| ------------------------------- | ----------------------- | ---------------------------- |
| `Could not resolve host`        | DNS                     | `getent`, resolver config    |
| IP curl works, hostname fails   | DNS                     | compare IP vs hostname       |
| `Connection refused`            | listener/bind/reject    | `ss -lntp`, firewall         |
| Connection timeout              | route/firewall/network  | route, traceroute, firewall  |
| localhost works, remote fails   | bind/firewall/path      | `ss`, interface IP, firewall |
| Ping fails, curl works          | ICMP policy             | không coi ping là quyết định |
| TCP connects, HTTP 404          | HTTP/path               | URL/context path             |
| TCP connects, HTTP 500          | application/backend     | app logs/config              |
| Port listening wrong process    | application ownership   | `ss -lntp`                   |
| Port listening only `127.0.0.1` | bind address            | `ss -lntp`                   |

---

# 77. Troubleshooting matrix — Storage

| Symptom                                     | Evidence                                            |
| ------------------------------------------- | --------------------------------------------------- |
| `No space left on device`                   | `df -h`, `df -i`                                    |
| Disk thấy trong `lsblk` nhưng path không có | `findmnt`, mount status                             |
| Mount fails                                 | filesystem type, source, mountpoint, logs           |
| Data “disappeared” sau mount                | check over-mounted directory                        |
| `Read-only file system`                     | `findmnt`, kernel logs                              |
| Permission denied                           | ownership/mode/ACL; Day 3 concepts                  |
| `df` full nhưng `du` thấp                   | open-deleted files, reserved blocks, mount topology |
| Filesystem full                             | `df`, `du`, growth source                           |
| Inodes 100%                                 | `df -i`, tiny-file explosion                        |

---

# 78. Failure investigation framework bạn phải tập từ hôm nay

Mỗi incident hãy viết theo chuỗi:

```text
1. Symptom
2. Expected behavior
3. Recent changes
4. Scope
5. Failing layer
6. Evidence
7. Hypotheses
8. Safest test
9. Root cause
10. L1-safe remediation
11. Validation
12. Evidence preservation
13. Prevention / escalation
```

Ví dụ:

```text
Symptom:
Remote HTTP unavailable.

Expected:
TCP/8080 + HTTP /health = 200.

Scope:
Only remote access affected; localhost works.

Evidence:
ss → 127.0.0.1:8080.

Hypothesis:
Incorrect bind address.

Safe test:
Compare curl loopback vs interface IP.

Root cause:
Listener bound only to loopback.

Remediation:
Correct bind configuration under approved change.

Validation:
ss + curl.

Prevention:
Deployment validation must include expected listening address.
```

Đó là troubleshooting có cấu trúc.

---

# 79. Các anti-pattern cần tránh

Đây là phần rất dễ bị kiểm tra trong interview/practical exam.

| Anti-pattern                          | Tại sao sai                               |
| ------------------------------------- | ----------------------------------------- |
| “Ping fail = server down”             | ICMP có thể bị block                      |
| “Curl fail = application down”        | Có thể DNS/network/firewall/socket        |
| Restart ngay                          | phá evidence, có thể che root cause       |
| Disable firewall để test              | blast radius lớn, security risk           |
| Open `0.0.0.0/0`                      | excessive exposure                        |
| `chmod 777` để chữa storage/app issue | bypass security, thường sai layer         |
| `rm -rf` ngay khi disk full           | có thể xóa dữ liệu quan trọng             |
| `mkfs` trước khi xác nhận disk        | có thể destroy data                       |
| Edit `fstab` rồi reboot để “test”     | rủi ro boot/mount failure                 |
| Chỉ dùng `df -h`                      | có thể bỏ qua inode exhaustion            |
| Chỉ dùng `du`                         | không cho toàn cảnh filesystem allocation |
| Guess firewall trước `ss`             | service có thể chưa listen                |

---

# 80. L1 scope vs escalation

Một Linux fresher/L1 không nên tự ý thực hiện mọi remediation.

## L1-safe thường gồm

Read-only inspection:

```bash
ip
ss
curl
ping
traceroute
lsblk
findmnt
df
du
journal/log inspection
```

Restart service **chỉ khi runbook/change scope cho phép**.

Cleanup known temporary lab/log files **chỉ khi ownership và retention rule rõ ràng**.

## Escalate khi gặp

```text
filesystem corruption
unexpected read-only remount
unknown production disk
possible hardware/storage failure
critical filesystem 100%
unknown process holding deleted data
network route controlled by another team
enterprise firewall rule change
production security policy change
unexplained packet loss
potential data loss
```

Escalation không có nghĩa:

```text
“Tôi không biết, nhờ team khác.”
```

Escalation tốt phải kèm:

```text
symptom
timeline
scope
commands
outputs
hypotheses tested
current state
business impact
safe actions already taken
what assistance is required
```

---

# 81. Security & least privilege

Networking:

```text
Không expose port rộng hơn cần thiết.
Không disable firewall chỉ vì troubleshooting.
Không assume localhost listener là lỗi.
Không gửi credential trong curl command nếu shell history có thể lưu.
```

Storage:

```text
Không mkfs unknown device.
Không mount untrusted filesystem tùy tiện.
Không sửa fstab không backup/validate.
Không xóa data chỉ để nhanh chóng giải phóng disk.
```

Một số mount option security bạn sẽ gặp:

```text
noexec
nosuid
nodev
ro
```

`mount(8)` định nghĩa các option như `nodev`, `nosuid`, `ro/rw`. [man7.org](https://man7.org/linux/man-pages/man8/mount.8.html?utm_source=chatgpt.com)

Chúng ta chỉ cần awareness ở Day 5; hardening chuyên sâu là supplementary.

---

# 82. Recovery và rollback mindset

Mỗi state-changing operation phải tự hỏi:

```text
Tôi thay đổi cái gì?
Nếu sai thì quay lại thế nào?
Làm sao chứng minh đã quay lại?
```

Ví dụ firewall:

```text
Change:
open 8080/tcp runtime

Rollback:
remove 8080/tcp

Validation:
firewall-cmd --query-port
```

Mount:

```text
Change:
mount loop filesystem

Rollback:
umount + losetup -d

Validation:
findmnt + losetup
```

Application bind:

```text
Change:
restart listener on new address

Rollback:
restore original bind configuration

Validation:
ss + curl
```

---

# 83. Supplementary — connection refused vs timeout

Đây là heuristic hữu ích nhưng không phải luật tuyệt đối.

### Connection refused

Thường có nghĩa remote/local stack đã phản hồi nhanh rằng:

```text
không có listener
```

hoặc policy reject.

### Timeout

Thường liên quan:

```text
packet bị drop
route/path issue
destination không trả response
```

Nhưng không được viết incident report:

```text
Timeout = firewall.
```

Đó mới chỉ là hypothesis.

---

# 84. Supplementary — DNS success chưa chứng minh service success

Ví dụ:

```bash
getent hosts app01
```

trả:

```text
10.1.2.3 app01
```

Điều này chỉ chứng minh name resolution.

Không chứng minh:

```text
10.1.2.3 reachable
port 8080 open
application healthy
```

Troubleshooting phải tiến layer tiếp theo.

---

# 85. Supplementary — Port open chưa chứng minh application healthy

Nếu:

```bash
ss -lntp
```

thấy:

```text
0.0.0.0:8080 LISTEN
```

thì chỉ chứng minh process đã create listening socket.

Nó không chứng minh:

```text
HTTP endpoint healthy
database reachable
authentication functioning
business transaction works
```

Cần:

```bash
curl ...
```

và sau này application health checks/monitoring.

---

# 86. Exam Focus — phần lý thuyết

Bạn nên trả lời được không cần nhìn tài liệu:

### “IP và DNS khác nhau thế nào?”

```text
IP identifies network endpoint.
DNS maps names to IP/resource records.
DNS failure can occur even when IP connectivity works.
```

### “Port dùng để làm gì?”

```text
Distinguishes network service endpoints on a host together with address/protocol.
```

### “`ss` dùng làm gì?”

```text
Inspect sockets, especially listening ports and TCP state/process ownership.
```

### “Ping fail có chứng minh server down?”

```text
No. ICMP may be filtered while application protocols remain reachable.
```

### “`df` và `du` khác gì?”

```text
df reports filesystem allocation/capacity.
du sums visible directory/file usage.
```

### “Disk và filesystem có phải một không?”

```text
No.
Disk/block device provides storage blocks.
Filesystem organizes files/metadata over storage.
```

### “Mount là gì?”

```text
Attach filesystem into Linux directory tree at a mountpoint.
```

---

# 87. Exam Focus — practical

Bạn có thể được yêu cầu:

```text
“Application không truy cập được trên port 8080.”
```

Một response tốt:

```bash
ip -br addr
ip route
getent hosts HOST
ping -c 4 HOST
traceroute HOST
sudo ss -lntp
curl -v http://HOST:8080/
```

Nhưng không phải chạy mù toàn bộ command.

Bạn phải diễn giải từng output.

Ví dụ:

```text
ss shows 127.0.0.1:8080.
Expected remote clients.
Therefore listener scope is inconsistent with requirement.
```

Điểm nằm ở reasoning, không phải số lượng command.

---

# 88. Knowledge Check

Hãy tự trả lời trước khi xem phần gợi ý cuối câu.

### Câu 1

Server có:

```text
192.168.10.50
```

`ping` thành công nhưng:

```bash
curl http://192.168.10.50:8080
```

fail.

Có được kết luận network OK hoàn toàn không?

**Đáp án:** Không. Ping chỉ kiểm tra ICMP; TCP/8080 vẫn có thể fail.

### Câu 2

`ss` cho:

```text
LISTEN 127.0.0.1:8080
```

Client khác host không truy cập được. Hypothesis mạnh nhất?

**Đáp án:** Application chỉ bind loopback.

### Câu 3

Hostname fail nhưng direct IP hoạt động. Layer nào ưu tiên?

**Đáp án:** DNS/name resolution.

### Câu 4

`df -h` báo còn 40% free nhưng tạo file mới báo `No space left on device`.

Kiểm tra gì?

```bash
df -i
```

Một khả năng là inode exhaustion.

### Câu 5

`/data` có file trước khi mount. Sau:

```bash
mount /dev/sdb1 /data
```

file biến mất.

Có chắc bị xóa?

**Đáp án:** Không. Mount có thể che directory contents trước đó.

### Câu 6

Có nên sửa `/etc/fstab` rồi reboot ngay để test?

**Đáp án:** Không. Backup, validate, test mount configuration an toàn trước.

---

# 89. Independent Practical Challenge

Sau khi hoàn thành lab hướng dẫn, hãy tự làm challenge này mà **không nhìn solution**.

Scenario:

```text
User reports:
http://app-lab:9090/health is inaccessible.

Server also reports:
application cannot create files under /mnt/appdata.
```

Nhiệm vụ của bạn là tạo hoặc mô phỏng environment rồi cung cấp evidence chứng minh:

```text
network identity
DNS result
route
listener address/port
curl result
filesystem source
mountpoint
byte capacity
inode capacity
root cause
remediation
validation
cleanup
```

Không được sử dụng:

```text
reboot
disable firewall
chmod 777
rm -rf random paths
```

và mỗi remediation phải có rollback.

Nếu bạn gửi cho mình output của challenge, mình có thể review theo **PRACTICE MODE**, chỉ ra từng chỗ đúng/sai trong reasoning.

---

# 90. Command set tối thiểu phải thuộc sau Day 5

Đây là **một checklist duy nhất** bạn nên dùng khi revision:

- Network identity/routing: `ip -br addr`, `ip route`, `ip route get DESTINATION`
- DNS: `getent hosts NAME`, `cat /etc/resolv.conf`, `resolvectl status/query` khi phù hợp
- Connectivity: `ping -c 4 HOST`, `traceroute HOST`
- Socket/port: `ss -lnt`, `sudo ss -lntp`
- Application/network test: `curl -v URL`, `curl -I URL`, `curl --connect-timeout ...`
- Firewall inspection: `firewall-cmd --get-active-zones`, `firewall-cmd --list-all` trên firewalld hosts
- Storage topology: `lsblk -f`, `findmnt`, `findmnt -T PATH`
- Capacity: `df -h PATH`, `df -i PATH`, `du`
- Mounts: `mount`, `umount`, `/etc/fstab`, `findmnt --verify --verbose`
- Safe troubleshooting mindset: **inspect → isolate layer → collect evidence → smallest remediation → validate → rollback/evidence**

---

# 91. Day 5 Mastery Check

Day 5 chỉ nên được coi là **đã học xong về lý thuyết** khi bạn giải thích được:

```text
IP → route → DNS → port → socket → firewall → application
```

và:

```text
disk → block device → filesystem → mount → capacity/inode → files
```

Day 5 chỉ nên được coi là **đạt practical mastery** khi bạn thực sự làm được:

```text
1. Tạo connectivity failure
2. Không biết trước root cause khi troubleshoot
3. Chứng minh root cause bằng command output
4. Remediate
5. Chứng minh service recovered
6. Tạo storage-full failure an toàn
7. Phân biệt byte capacity và inode
8. Recover filesystem
9. Cleanup toàn bộ lab
10. Viết health-check/evidence record
```

Việc chỉ đọc bài hôm nay chưa đủ để đánh dấu practical mastery.

---

## Tổng kết Day 5

Ngày hôm nay tạo ra hai năng lực nền tảng sẽ được dùng xuyên suốt phần còn lại của syllabus.

**Networking:** bạn phải biết rằng một request có nhiều layer và dùng `ip`, DNS checks, `ping`, `traceroute`, `ss`, `curl`, firewall evidence để cô lập layer hỏng. `ping` chỉ dùng ICMP; `ss` cho biết socket thực sự đang listen ở đâu; `curl` đưa troubleshooting lên TCP/HTTP/application layer. Tài liệu hiện hành của `ping`, `ss`, `traceroute` và `curl` xác nhận đúng các chức năng này. [man7.org](https://www.man7.org/linux/man-pages/man8/ping.8.html?utm_source=chatgpt.com)

**Storage:** bạn phải phân biệt block device, filesystem và mountpoint; dùng `lsblk` để thấy topology, `findmnt` để xác nhận mount và `df` để kiểm tra capacity. `mount` gắn filesystem vào Linux directory tree, còn `/etc/fstab` mô tả persistent filesystem configuration. [man7.org](https://man7.org/linux/man-pages/man8/lsblk.8.html?utm_source=chatgpt.com)

Phần quan trọng nhất của Day 5 không phải thuộc nhiều command. Đó là hình thành thói quen:

```text
Symptom
   ↓
Expected behavior
   ↓
Scope
   ↓
Layer isolation
   ↓
Evidence
   ↓
Hypothesis
   ↓
Safe test
   ↓
Root cause
   ↓
Minimal remediation
   ↓
Validation
   ↓
Evidence + rollback
```

Đó chính là nền tảng để sang **Day 6: `journalctl`, `grep/awk/sed`, Bash scripting, exit codes, cron, `tar`/`rsync`, health-check + backup script và failure/escalation evidence** theo syllabus. 00_MASTER_SYLLABUS_Java_Fresher…
