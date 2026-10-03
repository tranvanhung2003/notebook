# Day 5 — Module 1: Linux Networking, Connectivity & Firewall Troubleshooting

## 1. Syllabus Alignment

Theo **MASTER_SYLLABUS**, Day 5 bắt buộc học:

> **IP/DNS/ports, `curl`, `ss`, `ping`, `traceroute`, firewall basics, disks, filesystems, mounts, capacity checks**

và Assignment/Lab cuối ngày là:

> **Diagnose a simulated connectivity/storage issue and produce a health-check record.** :chatgpt-content-reference{index="0"}

Trong **Module 1 này**, chúng ta tập trung đầy đủ vào nửa networking:

**Mandatory — đúng syllabus**

- IP
- DNS
- ports
- `ping`
- `traceroute`
- `ss`
- `curl`
- firewall basics
- connectivity troubleshooting
- connectivity health-check evidence

**Module 2 sau đó** sẽ xử lý:

- disks
- filesystems
- mounts
- capacity checks
- storage troubleshooting
- integrated Day-5 assignment + health-check record hoàn chỉnh

Tôi cũng sẽ bổ sung một số kiến thức cần thiết như routing, network interfaces, TCP/UDP, sockets, loopback, CIDR, TCP states, `ip`, `/etc/hosts`, `getent`, `resolv.conf`, `nftables` awareness. Những phần này được đánh dấu **Supplementary / Beyond the explicit syllabus**. Chúng không thay thế nội dung syllabus; chúng giúp bạn thực sự hiểu và troubleshoot được những nội dung syllabus yêu cầu.

---

# 2. Mục tiêu của Module 1

Sau Module 1, mục tiêu không phải chỉ là nhớ:

```bash
ping ...
ss ...
curl ...
```

Bạn phải có khả năng nhìn một lỗi kiểu:

```text
Unable to connect to application
```

và phân rã nó thành những câu hỏi có thể kiểm chứng:

```text
1. Máy của tôi có network interface hoạt động không?
2. Máy có IP đúng không?
3. Destination hostname có resolve được không?
4. Destination IP có route phù hợp không?
5. Host đích có reachable không?
6. TCP/UDP port có đúng không?
7. Server process có bind/listen port đó không?
8. Nó bind trên địa chỉ nào?
9. Host firewall có cho phép traffic không?
10. Network firewall bên ngoài host có cho phép không?
11. TCP connection có thiết lập được không?
12. TLS có thành công không?
13. HTTP request có tới application không?
14. Application trả response gì?
```

Đây là tư duy mà một fresher làm Administration/Operations cần xây dựng.

---

# 3. Mental model quan trọng nhất của Module 1

Giả sử client gọi:

```text
https://app.example.com:8443/health
```

Đừng nhìn đây như một hành động duy nhất.

Thực tế nó là một chuỗi:

```text
Application
    │
    │ URL = https://app.example.com:8443/health
    ▼
Hostname resolution
DNS / hosts
app.example.com → 10.20.30.40
    │
    ▼
Routing decision
"Muốn tới 10.20.30.40 thì đi qua interface/gateway nào?"
    │
    ▼
IP network
Client → routers → Server
    │
    ▼
TCP
connect tới 10.20.30.40:8443
    │
    ▼
Server socket
Process có LISTEN trên port 8443 không?
    │
    ▼
Firewall
Traffic có được phép đi qua không?
    │
    ▼
TLS
Nếu HTTPS: handshake/certificate/SNI
    │
    ▼
HTTP
GET /health
Host: app.example.com
    │
    ▼
Application
HTTP 200 / 404 / 500 / ...
```

Nếu bạn không tách các bước này, troubleshooting rất dễ biến thành:

```bash
restart service
restart network
disable firewall
reboot
```

Đó là **trial-and-error**, không phải troubleshooting.

Tư duy đúng là:

```text
Symptom
   ↓
Expected behavior
   ↓
Scope
   ↓
Evidence
   ↓
Which layer failed?
   ↓
Hypothesis
   ↓
Safest test
   ↓
Root cause
   ↓
Remediation
   ↓
Validation
```

---

# 4. Networking không đồng nghĩa với Internet

Một sai lầm rất phổ biến của fresher là:

> "Có network" = "truy cập Internet được."

Không đúng.

Một Linux machine hoàn toàn có thể:

- có network interface,
- có IP,
- giao tiếp được với các host cùng subnet,
- truy cập database nội bộ,

nhưng:

- không có route ra Internet,
- hoặc outbound firewall chặn Internet.

Ngược lại, máy có Internet nhưng không truy cập được một private subnet cụ thể.

Networking phải được xem theo **source → destination**.

Ví dụ:

```text
Host A → Host B:8080
```

thành công không chứng minh:

```text
Host A → Host B:5432
```

thành công.

Và:

```text
Host A → Host B:8080
```

thành công cũng không chứng minh:

```text
Host C → Host B:8080
```

thành công.

Network access phụ thuộc ít nhất:

```text
source
destination
protocol
port
route
firewall/policy
application state
```

---

# 5. IP — nền tảng đầu tiên

## 5.1 IP address là gì?

IP address là địa chỉ được sử dụng ở network layer để định danh endpoint/interface trong IP network.

Ví dụ IPv4:

```text
192.168.1.10
10.20.30.40
172.16.5.21
```

Ví dụ IPv6:

```text
2001:db8::10
fe80::1234:abcd
::1
```

Ở Day 5, chúng ta sẽ dùng IPv4 nhiều hơn để tạo mental model, nhưng trong môi trường thật bạn phải luôn nhận thức IPv6 có thể tồn tại.

---

# 6. IPv4: 32 bit

IPv4 có 32 bit, thường viết thành bốn octet:

```text
192.168.10.25
```

Binary tương ứng:

```text
192       168       10        25

11000000  10101000  00001010  00011001
```

Mỗi octet có giá trị:

```text
0–255
```

---

# 7. Network prefix / CIDR

Ví dụ:

```text
192.168.10.25/24
```

`/24` nghĩa là:

```text
24 bit đầu = network portion
8 bit sau  = host portion
```

Netmask tương ứng:

```text
255.255.255.0
```

Với:

```text
192.168.10.25/24
```

network là:

```text
192.168.10.0/24
```

Bạn cần hiểu CIDR vì nó xuất hiện khắp nơi sau này:

```text
Linux routing
firewall rules
AWS VPC
subnets
security groups
NACL
PostgreSQL pg_hba.conf
monitoring networks
```

---

# 8. Một vài CIDR quan trọng

| Prefix | Netmask           | Tổng số địa chỉ IPv4 |
| ------ | ----------------- | -------------------: |
| `/8`   | `255.0.0.0`       |           16,777,216 |
| `/16`  | `255.255.0.0`     |               65,536 |
| `/24`  | `255.255.255.0`   |                  256 |
| `/25`  | `255.255.255.128` |                  128 |
| `/26`  | `255.255.255.192` |                   64 |
| `/27`  | `255.255.255.224` |                   32 |
| `/28`  | `255.255.255.240` |                   16 |
| `/30`  | `255.255.255.252` |                    4 |
| `/32`  | `255.255.255.255` |                    1 |

`/32` rất quan trọng trong firewall/security policy vì nó thường biểu diễn:

> chính xác một IPv4 address.

Ví dụ:

```text
10.0.5.20/32
```

---

# 9. Private IPv4 ranges

Các private IPv4 ranges quen thuộc:

```text
10.0.0.0/8

172.16.0.0/12

192.168.0.0/16
```

Ví dụ:

```text
10.20.1.15
172.20.10.10
192.168.1.100
```

thường được sử dụng trong private networks.

Private address không tự động reachable từ public Internet.

Khái niệm này đặc biệt quan trọng khi sang:

- AWS VPC
- public/private subnet
- NAT
- EC2
- database private access

ở Day 7–11.

---

# 10. Loopback

IPv4 loopback phổ biến nhất:

```text
127.0.0.1
```

hostname thường:

```text
localhost
```

IPv6 loopback:

```text
::1
```

Loopback đại diện cho chính local machine.

Ví dụ:

```bash
curl http://127.0.0.1:8080
```

nghĩa là:

> Kết nối tới port 8080 của chính máy đang chạy `curl`.

Traffic này không cần chạy ra physical network.

Đây là một test cực kỳ quan trọng.

---

# 11. `127.0.0.1` và `0.0.0.0` hoàn toàn khác nhau

Đây là kiến thức cực kỳ quan trọng cho Java/Tomcat.

Giả sử:

```bash
ss -lnt
```

hiện:

```text
LISTEN 0 100 127.0.0.1:8080 0.0.0.0:*
```

Điều này thường nghĩa là application chỉ bind trên loopback IPv4.

Do đó:

```bash
curl http://127.0.0.1:8080
```

có thể thành công.

Nhưng client từ máy khác gọi:

```text
10.0.1.20:8080
```

có thể thất bại.

Ngược lại:

```text
0.0.0.0:8080
```

khi xuất hiện như local listening address thường có nghĩa:

> socket đang bind trên tất cả local IPv4 addresses.

Ví dụ máy có:

```text
127.0.0.1
10.0.1.20
```

application bind:

```text
0.0.0.0:8080
```

thì socket có thể nhận connection tới:

```text
127.0.0.1:8080
10.0.1.20:8080
```

subject to firewall/routing và các controls khác.

Đây là một trong những lỗi deployment thực tế phổ biến:

```text
Application is running
Local curl works
Remote curl fails
```

sau đó phát hiện:

```text
application bound to 127.0.0.1 only
```

---

# 12. Network interface

Linux machine có một hoặc nhiều network interfaces.

Ví dụ:

```text
lo
eth0
ens5
ens160
enp0s3
```

`lo`:

```text
loopback
```

Tên các interface khác phụ thuộc distro, VM/cloud, predictable network naming, configuration...

Không được giả định máy nào cũng có:

```text
eth0
```

---

# 13. Kiểm tra interface bằng `ip`

**Supplementary / Beyond explicit syllabus**, nhưng rất quan trọng để hiểu IP.

Lệnh:

```bash
ip link
```

kiểm tra interface/link.

Ví dụ:

```text
2: ens5: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
```

Hai flag đáng chú ý:

```text
UP
LOWER_UP
```

Hiểu đơn giản:

- `UP`: interface được administratively enabled.
- `LOWER_UP`: lower layer/link được kernel xem là operational.

Nhưng:

```text
interface UP
```

không chứng minh:

```text
DNS works
Internet works
server reachable
application works
```

---

# 14. Kiểm tra IP address

```bash
ip addr
```

hoặc ngắn hơn:

```bash
ip a
```

Một cách rất tiện:

```bash
ip -br addr
```

Ví dụ:

```text
lo       UNKNOWN  127.0.0.1/8 ::1/128
ens5     UP       10.0.1.25/24
```

Bạn có thể đọc:

```text
interface: ens5
state:     UP
IPv4:      10.0.1.25
prefix:    /24
```

`ip address` là interface chính của `iproute2` để quản lý protocol addresses. :chatgpt-content-reference{index="1"}

---

# 15. Expected invariant vs Example output

Trong khóa học này phải phân biệt hai khái niệm.

### Example output

```text
ens5 UP 10.0.1.25/24
```

Đó chỉ là output ví dụ.

### Expected invariant

Điều chúng ta thực sự cần chứng minh:

```text
The expected network interface exists.
It is operational.
It has the expected IP/prefix.
```

Máy của bạn có thể là:

```text
eth0
```

hoặc:

```text
ens160
```

hoặc:

```text
enp0s8
```

Bạn không được học thuộc output.

---

# 16. Routing — kiến thức bổ sung bắt buộc để troubleshoot IP

**Supplementary / Beyond the explicit syllabus**

Syllabus ghi IP nhưng không nêu riêng routing. Tuy nhiên không hiểu route thì không thể troubleshoot connectivity nghiêm túc.

IP của destination:

```text
10.20.30.40
```

không nói cho Linux biết trực tiếp:

> packet phải đi đâu.

Linux phải đưa ra **routing decision**.

---

# 17. Routing table

Xem routing table:

```bash
ip route
```

Ví dụ:

```text
default via 10.0.1.1 dev ens5
10.0.1.0/24 dev ens5 proto kernel scope link src 10.0.1.25
```

Dòng:

```text
10.0.1.0/24 dev ens5
```

có ý nghĩa:

> Destination thuộc `10.0.1.0/24` có thể đi trực tiếp qua `ens5`.

Dòng:

```text
default via 10.0.1.1 dev ens5
```

nghĩa là:

> Destination không match route cụ thể hơn thì gửi qua gateway `10.0.1.1`.

`ip route` là công cụ quản lý/inspect routing table trong `iproute2`. :chatgpt-content-reference{index="2"}

---

# 18. `default route`

Default route thường viết:

```text
default
```

và về IPv4 conceptually tương đương:

```text
0.0.0.0/0
```

Nó là:

> catch-all route khi không có route cụ thể hơn.

Không có default route không nhất thiết có nghĩa local network hỏng.

Ví dụ:

```text
10.0.1.25 → 10.0.1.50
```

có thể vẫn chạy nhờ connected route:

```text
10.0.1.0/24 dev ens5
```

nhưng:

```text
10.0.1.25 → 8.8.8.8
```

có thể không có route.

---

# 19. Lệnh cực kỳ hữu dụng: `ip route get`

Thay vì tự đoán route:

```bash
ip route get 10.20.30.40
```

Ví dụ:

```text
10.20.30.40 via 10.0.1.1 dev ens5 src 10.0.1.25
```

Bạn đọc:

```text
destination = 10.20.30.40
gateway     = 10.0.1.1
interface   = ens5
source IP   = 10.0.1.25
```

Điểm mạnh của lệnh này:

> Nó trả lời trực tiếp Linux dự định route packet tới destination đó như thế nào.

---

# 20. Một request networking hoàn chỉnh

Giả sử:

```bash
curl http://app.internal:8080/health
```

Hệ thống cần:

```text
app.internal
      │
      │ hostname resolution
      ▼
10.20.30.40
      │
      │ routing
      ▼
ens5 / gateway
      │
      │ packets
      ▼
10.20.30.40
      │
      │ TCP :8080
      ▼
server socket
      │
      │ HTTP
      ▼
/health
```

Do đó lỗi:

```text
curl: connection failed
```

có thể đến từ rất nhiều layer khác nhau.

---

# 21. DNS — hostname không phải IP address

Con người thích:

```text
db.example.internal
```

network stack cần:

```text
10.20.30.50
```

DNS thực hiện ánh xạ giữa tên và thông tin DNS, trong đó trường hợp phổ biến là hostname → IP.

Ví dụ:

```text
app.example.com
       ↓ DNS
203.0.113.10
```

---

# 22. DNS records căn bản

**Supplementary / Beyond explicit syllabus**

Một số record quan trọng:

```text
A       hostname → IPv4
AAAA    hostname → IPv6
CNAME   alias → another DNS name
MX      mail routing
TXT     arbitrary text/policy/verification
NS      authoritative name server
PTR     reverse lookup
```

Cho Day 5, quan trọng nhất:

```text
A
AAAA
```

và awareness về:

```text
CNAME
```

---

# 23. DNS port

DNS thông thường sử dụng:

```text
53/UDP
53/TCP
```

Không nên học:

> DNS = UDP only.

DNS cũng sử dụng TCP trong các trường hợp phù hợp.

Port numbers thuộc từng transport protocol; registry chính thức do IANA quản lý. IANA chia TCP/UDP port numbers thành System Ports `0–1023`, User Ports `1024–49151` và Dynamic/Private Ports `49152–65535`. :chatgpt-content-reference{index="3"}

---

# 24. Linux resolver không chỉ là DNS

Đây là distinction rất quan trọng.

Khi application gọi hostname:

```text
app.internal
```

Linux userspace thường có thể resolve thông qua nhiều nguồn tùy NSS configuration, ví dụ:

```text
/etc/hosts
DNS
```

Do đó:

> hostname resolution ≠ DNS query trong mọi trường hợp.

Ví dụ `/etc/hosts`:

```text
10.20.30.40 app.internal
```

application có thể resolve `app.internal` mà không cần authoritative DNS cung cấp record đó.

---

# 25. `/etc/hosts`

Kiểm tra:

```bash
cat /etc/hosts
```

Ví dụ:

```text
127.0.0.1 localhost
10.20.30.40 app.internal
```

Một lỗi thực tế:

```text
DNS has correct new IP: 10.20.30.50
```

nhưng máy có:

```text
/etc/hosts
10.20.30.40 app.internal
```

Application vẫn có thể kết nối tới IP cũ.

Do đó khi troubleshooting DNS/name resolution:

```text
"DNS record correct"
```

chưa chắc:

```text
"application resolves correct address"
```

---

# 26. `/etc/nsswitch.conf`

**Supplementary**

Kiểm tra:

```bash
grep '^hosts:' /etc/nsswitch.conf
```

Một configuration có thể giống:

```text
hosts: files dns
```

Mental model:

```text
files
 ↓
/etc/hosts

dns
 ↓
DNS resolver
```

Configuration thực tế có thể phức tạp hơn, ví dụ systemd-related NSS modules.

Đừng chỉnh `/etc/nsswitch.conf` chỉ để thử nghiệm khi chưa hiểu ảnh hưởng.

---

# 27. `/etc/resolv.conf`

Một resolver configuration truyền thống quan trọng:

```bash
cat /etc/resolv.conf
```

Ví dụ:

```text
nameserver 10.0.0.2
search example.internal
```

`nameserver` chỉ định DNS resolver được query; `search` có thể ảnh hưởng cách short hostname được mở rộng. Đây là semantics chuẩn của resolver configuration. :chatgpt-content-reference{index="4"}

Tuy nhiên có một nuance quan trọng:

> Trên nhiều Linux hiện đại, `/etc/resolv.conf` có thể được tạo/quản lý bởi NetworkManager, `systemd-resolved`, DHCP, cloud-init hoặc một network management stack khác.

Không sửa file này một cách tùy tiện trước khi xác định ai đang quản lý nó.

---

# 28. Short name và FQDN

Giả sử:

```text
search corp.example.com
```

Bạn chạy:

```bash
ping app01
```

Resolver có thể thử:

```text
app01.corp.example.com
```

Vì vậy:

```text
app01
```

và:

```text
app01.corp.example.com
```

không phải lúc nào cũng nên được coi là cùng một test.

Trong troubleshooting production, FQDN thường rõ nghĩa hơn.

---

# 29. Kiểm tra name resolution bằng `getent`

**Supplementary nhưng rất khuyến nghị**

```bash
getent hosts example.com
```

hoặc:

```bash
getent ahosts example.com
```

Điểm mạnh của `getent`:

> Nó đi qua system name-service mechanisms/NSS, gần với cách nhiều application resolve hostname hơn là chỉ query một DNS server trực tiếp.

Ví dụ:

```bash
getent hosts app.internal
```

Output:

```text
10.20.30.40 app.internal
```

Expected invariant:

```text
hostname resolves to the intended address(es)
```

không phải:

```text
must print exactly one line
```

Vì hostname có thể có nhiều A/AAAA records.

---

# 30. `dig` awareness

**Supplementary / Beyond explicit syllabus**

Nếu package phù hợp đã được cài:

```bash
dig example.com
```

hoặc:

```bash
dig A example.com
```

Bạn có thể query DNS chi tiết.

Nhưng phải hiểu distinction:

```text
dig
```

thường được dùng để quan sát DNS protocol/records.

Trong khi:

```text
getent
```

hữu ích để quan sát name resolution theo system NSS path.

Do đó:

```text
dig works
```

nhưng:

```text
application resolution fails
```

không phải mâu thuẫn.

Có thể application/system resolution path khác.

---

# 31. DNS troubleshooting workflow

Nếu:

```bash
curl https://app.example.com
```

báo:

```text
Could not resolve host
```

đừng bắt đầu bằng firewall port 443.

Bắt đầu:

```bash
getent hosts app.example.com
```

Sau đó kiểm tra:

```bash
cat /etc/resolv.conf
```

và nếu cần:

```bash
grep '^hosts:' /etc/nsswitch.conf
cat /etc/hosts
```

Tư duy:

```text
Does name resolve?
     │
     ├── NO
     │    ↓
     │  resolver config?
     │  DNS reachable?
     │  /etc/hosts?
     │  typo?
     │  search domain?
     │
     └── YES
          ↓
        What IP?
          ↓
        Is it the EXPECTED IP?
```

DNS success chỉ chứng minh:

> Bạn lấy được một answer/name resolution result.

Nó không chứng minh destination server reachable.

---

# 32. IP connectivity và name resolution phải tách ra

Giả sử:

```bash
ping app.example.com
```

thất bại.

Bạn chưa biết lỗi ở:

```text
DNS
hay
network
hay
ICMP filtering
```

Hãy tách:

```bash
getent hosts app.example.com
```

Nếu ra:

```text
10.20.30.40
```

sau đó test IP:

```bash
ping -c 4 10.20.30.40
```

Như vậy bạn đã tách:

```text
name resolution
```

khỏi:

```text
IP/ICMP path
```

---

# 33. Ports là gì?

Một host có một IP nhưng chạy nhiều dịch vụ.

Ví dụ:

```text
10.20.30.40
```

có thể đồng thời chạy:

```text
SSH          TCP 22
HTTP         TCP 80
HTTPS        TCP 443
PostgreSQL   TCP 5432
Java app     TCP 8080
```

Port giúp transport layer phân biệt endpoint/service.

Ví dụ:

```text
10.20.30.40:22
10.20.30.40:443
10.20.30.40:5432
10.20.30.40:8080
```

là các endpoint khác nhau.

IANA xác nhận chẳng hạn SSH được đăng ký trên port `22/tcp`, còn HTTPS sử dụng port `443` trong service-name registry. :chatgpt-content-reference{index="5"}

---

# 34. IP + port chưa đủ: còn protocol

Cần phân biệt:

```text
TCP 53
UDP 53
```

Đây là hai transport endpoints khác nhau.

Firewall rule:

```text
allow TCP 53
```

không đồng nghĩa:

```text
allow UDP 53
```

Một port number luôn phải được hiểu cùng transport protocol.

---

# 35. TCP mental model

TCP là:

- connection-oriented
- reliable byte stream
- có connection state
- retransmission
- ordering
- flow/congestion control

Một TCP connection thường được định danh bằng tuple:

```text
source IP
source port
destination IP
destination port
protocol
```

Ví dụ:

```text
10.0.1.25:53124
        →
10.20.30.40:443
TCP
```

Client source port:

```text
53124
```

thường là ephemeral port.

Server destination port:

```text
443
```

là service port.

---

# 36. TCP three-way handshake

Mental model:

```text
Client                         Server
  │                              │
  │ SYN                          │
  ├─────────────────────────────►│
  │                              │
  │ SYN-ACK                      │
  │◄─────────────────────────────┤
  │                              │
  │ ACK                          │
  ├─────────────────────────────►│
  │                              │
  │      ESTABLISHED             │
```

Nếu TCP handshake không hoàn thành:

HTTP request thậm chí còn chưa được gửi.

Đó là lý do:

> HTTP problem và TCP connection problem phải được phân biệt.

---

# 37. TCP `connection refused`

Ví dụ:

```text
curl: (7) Failed to connect ...
Connection refused
```

Một trường hợp điển hình:

```text
Client SYN
    →
Server
    ←
RST
```

Nó thường gợi ý:

- destination host/IP đã có thể được reached ở mức đủ để trả lời,
- nhưng không có service accept/listen tại endpoint đó,
- hoặc firewall/policy đang chủ động **reject** connection.

Không được kết luận ngay:

> Service chắc chắn down.

Có thể:

```text
wrong port
wrong destination IP
service only bound localhost
firewall REJECT
application not running/listening
```

---

# 38. TCP timeout

Nếu connection timeout, các khả năng có thể gồm:

```text
bad route
packet loss
firewall DROP
network ACL/security policy
unreachable network
remote host unavailable
return path problem
```

Timeout cung cấp evidence khác với `connection refused`, nhưng một error message duy nhất vẫn chưa đủ xác định root cause.

---

# 39. UDP mental model

UDP:

- connectionless
- không có TCP-style handshake
- không đảm bảo delivery/order/retransmission ở transport layer

Vì vậy:

> "UDP port open" khó xác nhận theo đúng cách TCP `connect()` success.

Điều này đặc biệt quan trọng khi troubleshoot DNS hoặc monitoring protocols.

---

# 40. Socket là gì?

Một socket là kernel object đại diện cho communication endpoint.

Ở server:

```text
process
  │
  ▼
socket
  │
bind IP:port
  │
listen
```

Ví dụ Java application:

```text
java PID 1234
        │
        ▼
TCP socket
        │
0.0.0.0:8080
```

`ss` chính là công cụ rất mạnh để inspect các socket này.

---

# 41. `ss` — command quan trọng nhất để kiểm tra port ở local host

Syllabus bắt buộc `ss`.

`ss` là utility để inspect socket statistics; nó có thể hiển thị TCP, UDP, listening/non-listening sockets, process association và nhiều TCP state information. :chatgpt-content-reference{index="6"}

Lệnh bạn phải thuộc và hiểu:

```bash
ss -lnt
```

Giải nghĩa:

```text
-l    listening
-n    numeric
-t    TCP
```

---

# 42. `ss -lnt`

```bash
ss -lnt
```

Ví dụ:

```text
State  Recv-Q Send-Q Local Address:Port Peer Address:Port
LISTEN 0      128    0.0.0.0:22       0.0.0.0:*
LISTEN 0      100    127.0.0.1:8080   0.0.0.0:*
```

Đọc từng dòng.

### SSH

```text
0.0.0.0:22
```

→ TCP port 22 đang listen trên IPv4 wildcard address.

### Java application

```text
127.0.0.1:8080
```

→ port 8080 chỉ listen trên loopback IPv4.

Đây là một root-cause clue rất mạnh.

---

# 43. Thêm process information

```bash
sudo ss -lntp
```

Thêm:

```text
-p = processes
```

Ví dụ:

```text
LISTEN 0 100 127.0.0.1:8080 0.0.0.0:* users:(("java",pid=4210,fd=45))
```

Bây giờ ta có evidence:

```text
Protocol:       TCP
State:          LISTEN
Bind address:   127.0.0.1
Port:           8080
Process:        java
PID:            4210
```

`ss -p` hiển thị process sử dụng socket; tùy quyền hạn, process information có thể yêu cầu privilege phù hợp để xem đầy đủ. :chatgpt-content-reference{index="7"}

---

# 44. Một filter rất hữu ích

```bash
sudo ss -lntp | grep ':8080'
```

Trong Day 6, `grep` mới là syllabus chính thức, nhưng bạn đã học text processing từ trước và ở đây dùng nó như supporting tool.

Hoặc `ss` có filter riêng, nhưng ở fresher level:

```bash
ss -lntp
```

rồi đọc output là đủ.

---

# 45. UDP sockets

```bash
ss -lun
```

hoặc:

```bash
sudo ss -lunp
```

Giải nghĩa:

```text
-l    receiving/listening-style bound sockets
-u    UDP
-n    numeric
-p    process
```

Bạn có thể thấy:

```text
UNCONN 0 0 0.0.0.0:53 0.0.0.0:*
```

Đừng mong UDP có TCP `LISTEN`/`ESTABLISHED` semantics giống hệt TCP.

---

# 46. `ss` không có option thì sao?

Theo man page, khi không có option, `ss` chủ yếu hiển thị các open non-listening sockets như established connections. :chatgpt-content-reference{index="8"}

Ví dụ:

```bash
ss
```

không phải lựa chọn tốt nhất nếu mục tiêu là:

> Port 8080 có listen không?

Hãy dùng:

```bash
ss -lnt
```

---

# 47. TCP states bạn nên biết

**Supplementary nhưng quan trọng**

Một số state:

```text
LISTEN
SYN-SENT
SYN-RECV
ESTAB / ESTABLISHED
FIN-WAIT-1
FIN-WAIT-2
CLOSE-WAIT
LAST-ACK
TIME-WAIT
```

Đối với fresher, cần chắc bốn state:

### `LISTEN`

Server đang chờ incoming TCP connections.

### `SYN-SENT`

Client đã gửi SYN, đang chờ handshake.

Nhiều `SYN-SENT` kéo dài có thể gợi ý:

```text
remote unreachable
firewall DROP
return path problem
```

### `ESTABLISHED`

TCP connection đã thiết lập.

### `TIME-WAIT`

Connection đã đóng nhưng kernel giữ state trong khoảng thời gian nhất định để TCP xử lý delayed segments/reuse safely.

`TIME-WAIT` **không tự động là lỗi**.

---

# 48. `ss` trả lời được gì?

Ví dụ:

```bash
sudo ss -lntp | grep 8080
```

Output:

```text
LISTEN ... 0.0.0.0:8080 ... java,pid=1234
```

Bạn chứng minh được:

```text
A TCP socket is listening locally
on port 8080
bound to the IPv4 wildcard address
and associated with the Java process
```

---

# 49. `ss` không chứng minh được gì?

Nó không tự động chứng minh:

```text
remote client can connect
firewall allows 8080
network route works
application returns correct HTTP response
TLS works
database works
```

Đây là distinction phải nhớ:

```text
LISTEN
≠
REACHABLE
≠
HEALTHY APPLICATION
```

---

# 50. `ping` — ICMP reachability test

Syllabus bắt buộc `ping`.

`ping` gửi ICMP Echo Request và cố nhận ICMP Echo Reply. `ping` trên Linux từ `iputils` hỗ trợ IPv4 và IPv6; `-4` hoặc `-6` có thể ép address family. :chatgpt-content-reference{index="9"}

Cách dùng cơ bản:

```bash
ping -c 4 10.20.30.40
```

`-c 4`:

```text
gửi 4 probe rồi dừng
```

Đây là cách tốt hơn trong health-check evidence so với:

```bash
ping 10.20.30.40
```

chạy vô hạn cho tới Ctrl+C.

---

# 51. Output `ping`

Ví dụ:

```text
PING 10.20.30.40 (10.20.30.40) 56(84) bytes of data.
64 bytes from 10.20.30.40: icmp_seq=1 ttl=63 time=1.21 ms
64 bytes from 10.20.30.40: icmp_seq=2 ttl=63 time=1.13 ms
64 bytes from 10.20.30.40: icmp_seq=3 ttl=63 time=1.18 ms
64 bytes from 10.20.30.40: icmp_seq=4 ttl=63 time=1.16 ms

--- 10.20.30.40 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss
rtt min/avg/max/mdev = ...
```

Các field quan trọng:

```text
packets transmitted
packets received
packet loss
RTT
TTL
```

---

# 52. RTT

RTT:

```text
Round-Trip Time
```

là thời gian probe đi tới destination và response quay lại.

Ví dụ:

```text
time=1.21 ms
```

Không nên lấy một ping duy nhất để kết luận performance.

Latency có thể thay đổi.

---

# 53. Packet loss

Ví dụ:

```text
4 transmitted
4 received
0% packet loss
```

Evidence tốt hơn:

```text
4 transmitted
2 received
50% packet loss
```

Packet loss có thể đến từ:

- congestion,
- wireless problems,
- physical/network problems,
- policing/rate limiting,
- ICMP rate limiting,
- path instability,
- firewall behavior.

Không được lập tức kết luận:

> Network bị mất 50% traffic application.

ICMP treatment có thể khác application traffic.

---

# 54. `ping` thành công chứng minh được gì?

Nếu:

```bash
ping -c 4 10.20.30.40
```

thành công, bạn có evidence rằng:

```text
ICMP Echo Request/Reply communication
between the tested endpoints
worked at that moment.
```

Nó hỗ trợ giả thuyết:

```text
basic IP reachability exists
```

---

# 55. `ping` không chứng minh được gì?

`ping` success **không chứng minh**:

```text
TCP 22 open
TCP 443 open
TCP 5432 open
Java port 8080 open
HTTP works
HTTPS certificate works
database works
application healthy
```

Ví dụ:

```bash
ping app01
```

success.

Nhưng:

```bash
curl http://app01:8080
```

có thể:

```text
Connection refused
```

Hoàn toàn hợp lý.

---

# 56. `ping` failure không đồng nghĩa "server down"

Đây là điểm thi/troubleshooting rất quan trọng.

Firewall có thể cho phép:

```text
TCP 443
```

nhưng chặn:

```text
ICMP Echo
```

Khi đó:

```bash
ping server
```

fail.

Nhưng:

```bash
curl https://server
```

success.

Vì vậy:

> **Never use ping as the sole proof that a service is down.**

Firewall filtering có thể quyết định traffic dựa trên protocols/ports; `ping` kiểm tra ICMP chứ không phải TCP application endpoint. :chatgpt-content-reference{index="10"}

---

# 57. Ping test progression

Một useful pattern:

```text
1. loopback
2. local interface
3. gateway
4. destination IP
5. destination hostname
```

Ví dụ:

```bash
ping -c 2 127.0.0.1

ping -c 2 10.0.1.25

ping -c 2 10.0.1.1

ping -c 4 10.20.30.40

ping -c 4 app.internal
```

Nhưng không phải lúc nào cũng cần chạy tất cả.

Troubleshooting nên hypothesis-driven, không phải checklist máy móc.

---

# 58. Ping hostname có hai variables cùng lúc

```bash
ping app.internal
```

test ít nhất:

```text
name resolution
+
ICMP connectivity
```

Nếu nó fail, ambiguity cao hơn.

Do đó khi cần isolation:

```bash
getent hosts app.internal
```

rồi:

```bash
ping -c 4 10.20.30.40
```

---

# 59. `traceroute` — path discovery

Syllabus bắt buộc `traceroute`.

Concept:

IP header có TTL:

```text
Time To Live
```

Mỗi router/hop giảm TTL.

Khi TTL tới zero, router thường gửi:

```text
ICMP Time Exceeded
```

về source.

`traceroute` tận dụng điều đó để khám phá các hop trên đường tới destination. Đây cũng chính là cơ chế được mô tả trong Linux `traceroute(8)`. :chatgpt-content-reference{index="11"}

---

# 60. Traceroute hoạt động conceptually như thế nào?

Probe 1:

```text
TTL = 1

Client → Router 1

Router 1:
TTL becomes 0
→ ICMP Time Exceeded
```

Ta biết hop 1.

Probe 2:

```text
TTL = 2

Client
 ↓
Router 1
 ↓
Router 2

Router 2:
TTL becomes 0
→ ICMP Time Exceeded
```

Ta biết hop 2.

Tiếp tục:

```text
TTL 3
TTL 4
TTL 5
...
```

cho tới destination hoặc giới hạn.

---

# 61. Chạy traceroute

Ví dụ:

```bash
traceroute example.com
```

Hoặc tránh reverse DNS resolution:

```bash
traceroute -n 10.20.30.40
```

`-n` thường giúp:

- output nhanh hơn,
- giảm DNS noise,
- tập trung vào IP path.

---

# 62. Example output

```text
traceroute to 10.20.30.40, 30 hops max
 1  10.0.1.1     0.6 ms  0.5 ms  0.5 ms
 2  10.10.0.1    2.1 ms  2.0 ms  2.2 ms
 3  10.20.30.40  3.3 ms  3.1 ms  3.2 ms
```

Mental model:

```text
Client
  │
  ├── 10.0.1.1
  │
  ├── 10.10.0.1
  │
  └── 10.20.30.40
```

---

# 63. `* * *` trong traceroute

Ví dụ:

```text
1  10.0.1.1  1 ms 1 ms 1 ms
2  * * *
3  10.20.30.40  5 ms 5 ms 4 ms
```

Không được nói:

> Hop 2 bị down.

Có thể hop 2:

- forward packets bình thường,
- nhưng không trả traceroute probes,
- hoặc ICMP response bị filter/rate-limit.

Đích vẫn xuất hiện ở hop 3.

Do đó:

```text
* * *
```

không tự động đồng nghĩa broken route.

---

# 64. Traceroute có thể dùng probe types khác nhau

Linux `traceroute` hỗ trợ nhiều phương thức/protocol probe; cách probe ảnh hưởng kết quả qua firewall. :chatgpt-content-reference{index="12"}

Ví dụ tùy implementation:

```bash
traceroute -I host
```

ICMP-based probe.

Một số implementation hỗ trợ TCP probe:

```bash
sudo traceroute -T -p 443 host
```

Điều này có thể hữu ích khi:

```text
ICMP/UDP traceroute bị filter
nhưng TCP 443 được phép
```

Nhưng options có thể khác theo installed implementation/version. Luôn:

```bash
traceroute --help
```

hoặc:

```bash
man traceroute
```

trên máy của bạn.

---

# 65. `tracepath` awareness

**Supplementary**

Một alternative trên Linux:

```bash
tracepath destination
```

`tracepath` cũng trace network path và có thể khám phá path MTU; man page ghi rõ nó tương tự traceroute và thông thường không cần superuser privileges. :chatgpt-content-reference{index="13"}

Nhưng vì syllabus yêu cầu `traceroute`, bạn vẫn phải biết `traceroute`.

---

# 66. Không được đọc traceroute như bản đồ tuyệt đối

Internet/network routing có thể:

- load balance,
- thay đổi path,
- có asymmetric routing,
- có nodes không reply,
- có filtering.

Do đó traceroute là:

> evidence về probe path tại thời điểm test.

Không phải:

> bản thiết kế vật lý tuyệt đối của network.

---

# 67. `curl` — application-layer diagnostic tool

Đây là tool cực kỳ quan trọng đối với Java Operations.

`ping` hỏi gần giống:

> "ICMP reachability hoạt động không?"

`curl` có thể hỏi:

> "HTTP/HTTPS application endpoint thực sự trả lời như thế nào?"

Giả sử Java app expose:

```text
http://localhost:8080/health
```

Bạn có thể chạy:

```bash
curl http://localhost:8080/health
```

---

# 68. URL anatomy

Ví dụ:

```text
https://app.example.com:8443/api/health
```

Tách:

```text
https
  │
  └─ scheme/protocol

app.example.com
  │
  └─ hostname

8443
  │
  └─ port

/api/health
  │
  └─ path
```

Nếu port không ghi rõ, protocol có default conventions.

Ví dụ:

```text
HTTP  → 80
HTTPS → 443
```

nhưng application có thể chạy port bất kỳ như:

```text
8080
8443
```

---

# 69. `curl` đơn giản

```bash
curl http://127.0.0.1:8080/health
```

Possible output:

```json
{ "status": "UP" }
```

Điều này mạnh hơn:

```bash
ss -lnt | grep 8080
```

vì giờ bạn đã chứng minh:

```text
TCP connection works
+
HTTP request works
+
endpoint produced a response
```

---

# 70. HTTP status code mới là evidence quan trọng

Ví dụ:

```text
HTTP/1.1 200 OK
```

thường nghĩa request được xử lý thành công theo HTTP semantics.

Nhưng:

```text
200
```

không nhất thiết có nghĩa toàn hệ thống khỏe.

Ví dụ `/` trả static page 200 nhưng:

```text
database unavailable
```

Nếu `/health` được thiết kế kiểm tra dependency thì evidence mạnh hơn.

Đừng equate:

```text
HTTP 200
```

với:

```text
everything healthy
```

mà chưa hiểu endpoint semantics.

---

# 71. `curl -v`

Một trong những command troubleshooting mạnh nhất:

```bash
curl -v http://app.internal:8080/health
```

`-v`:

```text
verbose
```

Nó có thể cho bạn thấy:

- hostname resolution details,
- IP curl thử,
- connection establishment,
- request headers,
- response headers,
- TLS details với HTTPS.

Ví dụ:

```text
* Host app.internal:8080 was resolved.
* IPv4: 10.20.30.40
* Trying 10.20.30.40:8080...
* Connected to app.internal (10.20.30.40) port 8080
> GET /health HTTP/1.1
> Host: app.internal:8080
...
< HTTP/1.1 200
```

Bạn có thể map:

```text
resolved
    ↓
DNS/name resolution good

Trying
    ↓
attempting TCP

Connected
    ↓
TCP connection established

GET /health
    ↓
HTTP request sent

HTTP/1.1 200
    ↓
HTTP response received
```

---

# 72. Security warning với `curl -v`

Trong production:

```bash
curl -v ...
```

có thể hiển thị headers và thông tin nhạy cảm liên quan tới request.

Nếu request chứa:

```text
Authorization
Cookie
tokens
```

đừng copy output tùy tiện vào:

- ticket,
- Slack,
- email,
- public chat,
- screenshot.

Phải sanitize secrets.

**Never paste credentials/tokens into your learning evidence.**

---

# 73. `curl -I`

```bash
curl -I https://example.com
```

gửi HTTP `HEAD` trong trường hợp HTTP(S).

Mục tiêu:

```text
retrieve headers without regular response body
```

Có thể rất tiện để kiểm tra:

```text
status
server headers
redirect
```

Nhưng có một nuance:

> Không phải mọi application endpoint hỗ trợ `HEAD` giống `GET`.

Ví dụ:

```text
HEAD /api/foo → 405
```

nhưng:

```text
GET /api/foo → 200
```

Vì vậy `curl -I` failure không nhất thiết nghĩa application GET failure.

---

# 74. `curl -sS`

Một useful combination:

```bash
curl -sS http://localhost:8080/health
```

`-s`:

```text
silent
```

`-S`:

```text
show errors even with silent
```

Useful trong scripting/health checks.

---

# 75. Bỏ body, chỉ lấy metadata

```bash
curl -sS -o /dev/null \
  -w '%{http_code}\n' \
  http://localhost:8080/health
```

Có thể output:

```text
200
```

Official curl documentation hỗ trợ `--write-out/-w` với các variables như `response_code/http_code`. :chatgpt-content-reference{index="14"}

---

# 76. Health-check timing bằng curl

Một command hữu ích:

```bash
curl -sS -o /dev/null \
  -w 'code=%{http_code} remote=%{remote_ip} dns=%{time_namelookup} connect=%{time_connect} total=%{time_total}\n' \
  https://example.com
```

Có thể cho:

```text
code=200 remote=203.0.113.10 dns=0.004 connect=0.030 total=0.120
```

`curl` cung cấp:

- `time_namelookup`
- `time_connect`
- `time_appconnect`
- `time_starttransfer`
- `time_total`
- `remote_ip`

để quan sát các phase của request. :chatgpt-content-reference{index="15"}

Điều này cực kỳ mạnh cho Operations.

---

# 77. Ví dụ phân tích latency

Giả sử:

```text
dns=2.800
connect=2.830
total=2.950
```

Có thể đặt hypothesis:

```text
DNS lookup unusually slow
```

Nếu:

```text
dns=0.002
connect=5.002
```

thì investigate:

```text
TCP/network path
```

Nếu:

```text
connect=0.010
time_starttransfer=4.500
```

có thể investigate:

```text
server/application processing
upstream dependency
```

Đây chưa phải root cause, nhưng nó giúp xác định failing/slow layer.

---

# 78. `--connect-timeout`

Ví dụ:

```bash
curl --connect-timeout 5 https://app.example.com
```

Theo curl hiện tại, `--connect-timeout` giới hạn connection phase; connection phase bao gồm name resolution và các handshake cần thiết như TCP/TLS/QUIC theo loại connection. :chatgpt-content-reference{index="16"}

Điểm quan trọng:

```text
--connect-timeout
```

không phải:

```text
maximum duration of entire transfer
```

---

# 79. `--max-time`

Nếu muốn giới hạn toàn operation:

```bash
curl --max-time 10 https://app.example.com
```

Một operational check thường nên có timeout.

Nếu không, một script health check có thể treo quá lâu.

Ví dụ:

```bash
curl --connect-timeout 3 --max-time 10 \
  https://app.example.com/health
```

---

# 80. `curl --resolve`: tool isolation cực kỳ mạnh

Giả sử DNS:

```text
app.example.com → 10.20.30.40
```

Bạn nghi DNS sai nhưng muốn test application node:

```text
10.20.30.50
```

Không nên sửa ngay `/etc/hosts`.

Có thể:

```bash
curl --resolve app.example.com:443:10.20.30.50 \
  https://app.example.com/health
```

`--resolve` tạo hostname→IP mapping riêng cho curl request, tương tự một command-line-specific hosts override, và vẫn giữ hostname gốc cho URL/TLS/application semantics. :chatgpt-content-reference{index="17"}

Đây là tool cực kỳ hữu ích để tách:

```text
DNS layer
```

khỏi:

```text
server/TLS/application layer
```

---

# 81. Vì sao không nên chỉ curl IP với HTTPS?

Ví dụ:

```bash
curl https://10.20.30.50
```

có thể fail certificate verification vì certificate được cấp cho:

```text
app.example.com
```

chứ không phải IP đó.

Ngoài ra virtual hosting/SNI có thể phụ thuộc hostname.

Test tốt hơn:

```bash
curl --resolve app.example.com:443:10.20.30.50 \
  https://app.example.com
```

Nó giữ đúng hostname semantics.

---

# 82. Không dùng `curl -k` như "fix"

Bạn có thể thấy:

```bash
curl -k https://...
```

`-k` / insecure bỏ qua một phần certificate verification.

Trong troubleshooting production, không nên biến đây thành solution.

Nếu normal:

```bash
curl https://app.example.com
```

fail certificate verification nhưng:

```bash
curl -k https://app.example.com
```

works,

evidence là:

> transport/application có thể reachable, nhưng TLS certificate verification đang gặp vấn đề.

Không phải:

> fix bằng `-k`.

Official curl FAQ cảnh báo disabling certificate verification làm connection insecure và có thể mở đường cho man-in-the-middle attack. :chatgpt-content-reference{index="18"}

---

# 83. Curl exit codes

Các code rất hữu ích cho troubleshooting và sau này scripting.

Một số quan trọng:

```text
0  success
6  could not resolve host
7  failed to connect
22 HTTP error when --fail/--fail-with-body semantics apply
28 timeout
60 certificate verification problem
```

Official curl documentation định nghĩa `6` là resolution failure và `7` là connection failure; `--fail` biến HTTP responses từ 400 trở lên thành curl error `22` trong các trường hợp áp dụng. :chatgpt-content-reference{index="19"}

Sau command:

```bash
curl ...
```

kiểm tra:

```bash
echo $?
```

Ví dụ:

```text
6
```

Bạn phải nghĩ:

```text
name resolution layer
```

trước khi nghĩ:

```text
HTTP endpoint
```

---

# 84. HTTP errors và network errors khác nhau

Giả sử:

```bash
curl -i http://app:8080/api/foo
```

trả:

```text
HTTP/1.1 404 Not Found
```

Điều gì đã thành công?

Ít nhất:

```text
hostname resolution
route sufficient
TCP connection
HTTP communication
server response
```

Problem hiện tại nhiều khả năng là:

```text
path/resource/routing at application layer
```

không phải "network down".

---

# 85. HTTP 500

```text
HTTP 500 Internal Server Error
```

cũng là evidence cực kỳ quan trọng.

Nó cho biết:

> HTTP server/application layer đã trả response.

Bạn không nên khắc phục `500` bằng cách mở firewall port ngẫu nhiên.

Có thể application:

- exception,
- DB failure,
- bad configuration,
- dependency failure.

Đó sẽ được học sâu hơn ở Java/PostgreSQL days.

---

# 86. HTTP 401/403

Thông thường:

```text
401
```

gợi ý authentication problem.

```text
403
```

gợi ý server hiểu request nhưng từ chối authorization/access theo policy.

Hai code này lại là evidence rằng:

```text
TCP + HTTP path reached a responding endpoint
```

Vì vậy đừng gọi đây đơn giản là "network error".

---

# 87. Một command health-check curl rất tốt

```bash
curl -sS \
  --connect-timeout 3 \
  --max-time 10 \
  -o /dev/null \
  -w 'http=%{http_code} remote_ip=%{remote_ip} dns=%{time_namelookup}s connect=%{time_connect}s total=%{time_total}s\n' \
  http://app.internal:8080/health
```

Đây là evidence có giá trị cao.

---

# 88. Firewall — mental model

Firewall là một policy enforcement point quyết định traffic nào:

```text
allowed
denied/rejected
dropped
```

Dựa trên các properties như:

```text
source
destination
protocol
port
interface
connection state
zone
...
```

Ví dụ:

```text
allow TCP
from 10.0.10.0/24
to local port 8080
```

---

# 89. Host firewall vs network firewall

Cần phân biệt:

```text
Application server
    │
    ├── process/socket
    │
    ├── Linux host firewall
    │
    ├── external network firewall/router
    │
    └── AWS Security Group/NACL etc.
```

AWS network controls sẽ học chi tiết sau.

Nhưng Day 5 cần hiểu:

> Host firewall tốt không chứng minh toàn network path cho phép traffic.

Và:

> AWS/network firewall tốt không chứng minh Linux host firewall cho phép traffic.

---

# 90. `DROP` vs `REJECT`

Conceptually:

### DROP

```text
packet arrives
 ↓
firewall silently discards
 ↓
sender waits
 ↓
may timeout
```

### REJECT

```text
packet arrives
 ↓
firewall actively sends rejection
 ↓
client learns failure sooner
```

Do đó:

```text
timeout
```

và:

```text
connection refused/rejected
```

có thể tạo clues khác nhau.

Nhưng đừng suy luận quá mức chỉ từ một symptom.

---

# 91. Linux firewall ecosystem

Linux firewall có nhiều layers/tools mà fresher dễ nhầm:

```text
kernel netfilter framework
       ↑
    nftables
       ↑
firewalld / UFW
```

Đây là simplified mental model, không phải dependency diagram tuyệt đối cho mọi distro.

Hai management styles bạn sẽ gặp nhiều:

### Ubuntu/Debian environments

```text
ufw
```

### RHEL-family environments

```text
firewalld
```

Ngoài ra:

```text
nftables
```

là framework/ruleset interface quan trọng ở Linux hiện đại.

Current Red Hat documentation mô tả `firewalld` là dynamic firewall daemon dùng zones, policies và services; với advanced direct packet filtering, Red Hat hướng người dùng về `nftables`. :chatgpt-content-reference{index="20"}

---

# 92. Trước khi thay firewall: INSPECT FIRST

Firewall là risky configuration.

Đặc biệt khi SSH vào remote server:

```text
Bạn đang dùng SSH TCP/22
       ↓
Bạn thay firewall sai
       ↓
Bạn block chính session/access path
       ↓
Có thể mất remote access
```

Do đó workflow an toàn:

```text
1. Identify current access path.
2. Record current rules.
3. Determine exact required source/protocol/port.
4. Understand rollback.
5. Prefer smallest change.
6. Apply in lab/non-production first.
7. Keep active recovery/session path where appropriate.
8. Validate immediately.
9. Roll back if validation fails.
```

Không dùng:

```bash
disable firewall
```

như troubleshooting mặc định.

---

# 93. UFW inspection — Ubuntu-style environment

Kiểm tra:

```bash
sudo ufw status
```

Tốt hơn:

```bash
sudo ufw status verbose
```

Official Ubuntu documentation sử dụng `ufw status verbose` để xem trạng thái, default policies và rules. :chatgpt-content-reference{index="21"}

Possible output:

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To          Action      From
--          ------      ----
22/tcp      ALLOW       Anywhere
```

---

# 94. Đọc UFW output

Nếu:

```text
22/tcp ALLOW Anywhere
```

nghĩa là rule cho phép TCP 22 trong scope thể hiện bởi rule.

Nhưng đừng kết luận:

```text
SSH definitely works
```

vì còn:

```text
sshd process?
listener?
routing?
external firewall?
credentials?
```

Firewall chỉ là một layer.

---

# 95. UFW rule modification

**Chỉ thực hiện trong approved lab/change.**

Ví dụ:

```bash
sudo ufw allow 8080/tcp
```

Official UFW syntax hỗ trợ:

```bash
ufw allow <port>/<protocol>
```

và `deny`, `delete` tương ứng. :chatgpt-content-reference{index="22"}

Rollback:

```bash
sudo ufw delete allow 8080/tcp
```

Nhưng production policy nên cụ thể hơn khi cần source restriction, thay vì mở cho mọi source.

---

# 96. Principle of least privilege với firewall

Nếu chỉ app server:

```text
10.10.20.15
```

cần gọi DB port:

```text
5432
```

thì policy lý tưởng không phải:

```text
allow everybody to 5432
```

mà là:

```text
allow only required trusted source(s)
to required protocol/port
```

Security rule:

> **Minimum required access, not maximum convenience.**

---

# 97. firewalld inspection

Trong RHEL-like environment:

```bash
sudo firewall-cmd --state
```

Xem active zones:

```bash
sudo firewall-cmd --get-active-zones
```

Xem configuration:

```bash
sudo firewall-cmd --list-all
```

Hoặc zone cụ thể:

```bash
sudo firewall-cmd --zone=public --list-all
```

`firewalld` sử dụng zones gắn với interfaces/sources; rules phải được hiểu trong zone context. :chatgpt-content-reference{index="23"}

---

# 98. firewalld services

Ví dụ:

```bash
sudo firewall-cmd --list-services
```

Có thể:

```text
ssh dhcpv6-client
```

`firewalld` có predefined services nhằm gom các ports/settings cần thiết cho service và giảm việc phải quản lý raw port numbers bằng tay. :chatgpt-content-reference{index="24"}

Ví dụ:

```text
ssh
```

service definition tương ứng với network requirements của SSH.

---

# 99. firewalld runtime vs permanent

Đây là exam/troubleshooting point rất quan trọng.

`firewalld` duy trì:

```text
runtime configuration
permanent configuration
```

Runtime:

```text
effective now
```

nhưng có thể không tồn tại sau reload/restart.

Permanent:

```text
persisted configuration
```

nhưng phải được loaded/applied theo appropriate workflow.

Red Hat documentation xác nhận runtime configuration không tồn tại qua `firewalld` reload/restart, trong khi permanent configuration là persisted state. :chatgpt-content-reference{index="25"}

---

# 100. Ví dụ runtime rule

**Lab/approved change only:**

```bash
sudo firewall-cmd --add-port=8080/tcp
```

Kiểm tra:

```bash
sudo firewall-cmd --list-ports
```

Rollback:

```bash
sudo firewall-cmd --remove-port=8080/tcp
```

Runtime test có thể hữu ích vì dễ rollback.

---

# 101. Permanent rule

Ví dụ:

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
```

Sau đó tùy workflow:

```bash
sudo firewall-cmd --reload
```

Nhưng production change phải tuân:

```text
change approval
impact assessment
access preservation
verification
rollback
```

Không được copy command một cách mù quáng.

---

# 102. `nft` awareness

**Supplementary / Beyond explicit syllabus**

Read-only inspection:

```bash
sudo nft list ruleset
```

Có thể giúp xem effective nftables rules.

Nhưng output có thể lớn/phức tạp.

Day 5 không yêu cầu bạn thành nftables engineer.

Bạn chỉ cần biết:

```text
UFW/firewalld
```

có thể là management layer trong khi effective packet-filtering implementation liên quan netfilter/nftables.

---

# 103. Đừng dùng `iptables -F`

Một anti-pattern cực nguy hiểm:

```bash
sudo iptables -F
```

hoặc bất kỳ kiểu:

```text
flush all firewall rules
```

chỉ để "xem có phải firewall không."

Điều này có thể:

- phá security policy,
- expose services,
- mất remote access,
- ảnh hưởng workloads khác,
- vi phạm change controls.

Troubleshooting phải bắt đầu bằng **inspection**, không phải destructive modification.

---

# 104. Connectivity troubleshooting hierarchy

Đây là flow quan trọng nhất của Module 1.

Giả sử:

```bash
curl http://app01.internal:8080/health
```

fail.

Đi theo layers:

```text
Layer 1: Local interface/IP
         ip -br addr

Layer 2: Name resolution
         getent hosts app01.internal

Layer 3: Route
         ip route get <IP>

Layer 4: Basic network evidence
         ping <IP>
         traceroute <IP>

Layer 5: Local server socket
         ss -lntp

Layer 6: Host firewall
         ufw/firewall-cmd/nft inspection

Layer 7: Application protocol
         curl -v URL

Layer 8: HTTP/application result
         status/body/log correlation
```

Thứ tự thực tế có thể thay đổi tùy symptom.

Bạn không cần chạy máy móc tất cả lệnh.

---

# 105. Troubleshooting phải bắt đầu từ symptom chính xác

Bad ticket:

```text
App down.
```

Good symptom:

```text
From client01, GET http://app01.internal:8080/health
fails with connection timeout since 14:05.
The same endpoint succeeded at 13:55.
```

Thông tin này cực kỳ mạnh:

```text
source
destination
protocol
port
endpoint
failure mode
time
recent good state
```

---

# 106. Step 1 — define expected behavior

Trước khi troubleshoot:

```text
Expected:
app01.internal should resolve to 10.20.30.40.

The Java service should listen on TCP/8080.

Clients in 10.10.5.0/24 should access
http://app01.internal:8080/health.

Expected HTTP status: 200.
```

Nếu không biết expected state, bạn có thể "sửa" hệ thống đang đúng.

---

# 107. Step 2 — determine scope

Hỏi:

```text
Only one client?
All clients?
Only one hostname?
Only port 8080?
Only HTTPS?
Local request also fails?
Started after a change?
```

Ví dụ:

```text
Local curl works.
Remote curl fails.
```

narrow scope rất mạnh.

Các suspect tăng lên:

```text
bind address
host firewall
network firewall
routing
```

Các suspect giảm xuống:

```text
application completely down
```

---

# 108. Scenario A — DNS failure

Symptom:

```bash
curl http://app.internal:8080
```

Output:

```text
curl: (6) Could not resolve host
```

Evidence:

```bash
getent hosts app.internal
```

không có result.

Kiểm tra:

```bash
cat /etc/resolv.conf
grep '^hosts:' /etc/nsswitch.conf
grep -n 'app.internal' /etc/hosts
```

Possible root causes:

```text
typo
missing DNS record
wrong search domain
wrong resolver config
DNS server unreachable
stale /etc/hosts
```

L1-safe action:

```text
collect evidence
correct known approved local typo/config if within scope
otherwise escalate to DNS/network owner
```

Không restart Java application.

---

# 109. Scenario B — DNS resolves wrong IP

Expected:

```text
app.internal → 10.20.30.40
```

Actual:

```bash
getent hosts app.internal
```

returns:

```text
10.20.30.55
```

Bạn đã tìm được discrepancy.

Next question:

```text
Why?
```

Có thể:

```text
DNS record wrong
/etc/hosts override
cache
split DNS
different resolver
```

Đừng ngay lập tức sửa production DNS nếu không có authority.

Preserve:

```text
timestamp
hostname queried
actual IP
expected IP
resolver configuration
```

rồi escalate đúng team nếu out of scope.

---

# 110. Scenario C — No route

```bash
ip route get 10.20.30.40
```

có thể báo unreachable hoặc không có appropriate route.

Điều này giải thích vì sao:

```text
curl
```

không thể tới destination.

Fix không phải:

```text
restart Tomcat
```

Layer bị lỗi là routing.

---

# 111. Scenario D — ping works, curl fails

```bash
ping -c 4 10.20.30.40
```

success.

Nhưng:

```bash
curl http://10.20.30.40:8080
```

returns:

```text
Connection refused
```

Interpretation:

```text
ICMP path works
BUT
TCP/8080 not accepting connection
```

Investigate on server:

```bash
sudo ss -lntp
```

Nếu không có `:8080`:

```text
application not listening
wrong port
service not started
startup failure
```

Nếu có:

```text
127.0.0.1:8080
```

likely bind-scope issue.

---

# 112. Scenario E — local curl works, remote curl fails

Server:

```bash
curl http://127.0.0.1:8080/health
```

returns `200`.

Client:

```bash
curl http://10.20.30.40:8080/health
```

times out/refused.

Server:

```bash
sudo ss -lntp
```

shows:

```text
127.0.0.1:8080
```

Strong hypothesis:

```text
application bound to loopback only
```

Nếu:

```text
0.0.0.0:8080
```

thì tiếp tục investigate:

```text
host firewall
network firewall
route
client source restrictions
```

---

# 113. Scenario F — listener exists but application wrong

```bash
ss -lntp
```

shows:

```text
0.0.0.0:8080
```

Remote curl:

```bash
curl -i http://app:8080/health
```

returns:

```text
HTTP/1.1 404
```

Không còn là "port not open."

Network path đã đủ để nhận HTTP response.

Investigate:

```text
wrong endpoint
wrong context path
wrong app deployed
reverse-proxy/routing config
```

---

# 114. Scenario G — HTTP 500

```bash
curl -i http://app:8080/health
```

returns:

```text
HTTP/1.1 500 Internal Server Error
```

Network team có thể không phải nơi escalate đầu tiên.

Evidence cho thấy:

```text
name resolution worked
TCP connection worked
HTTP communication worked
server returned application-level failure
```

Next layers:

```text
app logs
configuration
database dependency
runtime error
```

Day 12–16 sẽ đào sâu.

---

# 115. Scenario H — HTTPS certificate error

```bash
curl https://app.example.com
```

returns certificate verification error.

Nhưng:

```text
DNS resolution succeeded
TCP may have succeeded
TLS negotiation reached certificate validation
```

Potential causes:

```text
expired certificate
hostname mismatch
untrusted CA
incomplete CA chain
wrong server/certificate
```

Đây không phải lý do để:

```text
open firewall
```

vì handshake đã tiến tới TLS certificate validation.

---

# 116. Scenario I — wrong port

Ticket:

```text
App unavailable on 8080.
```

Server:

```bash
sudo ss -lntp
```

shows:

```text
0.0.0.0:8081
```

Có thể:

```text
application configuration changed
ticket/runbook has stale port
wrong environment
```

Đừng mở firewall 8080 trước khi xác định **expected port**.

---

# 117. Scenario J — firewall blocks remote access

Server:

```bash
ss -lntp
```

shows:

```text
0.0.0.0:8080
```

Local:

```bash
curl http://127.0.0.1:8080
```

success.

Remote:

```bash
curl http://10.20.30.40:8080
```

timeout.

Host firewall:

```bash
sudo ufw status verbose
```

không có rule cần thiết.

Strong hypothesis:

```text
host firewall or another network policy is filtering remote traffic
```

Nhưng phải xác định external firewall/security controls trước khi kết luận final root cause.

---

# 118. Error localization cheat sheet

| Evidence                         | Có thể suy ra                                                      |
| -------------------------------- | ------------------------------------------------------------------ |
| `getent hosts` fail              | name resolution layer suspect                                      |
| hostname resolves                | name resolution succeeded at test time                             |
| `ip route get` has correct route | kernel has a routing decision                                      |
| `ping` works                     | ICMP request/reply worked                                          |
| `ping` fails                     | không đủ để kết luận host/service down                             |
| `ss` shows no listener           | local service is not listening on tested TCP port                  |
| `ss` shows `127.0.0.1:8080`      | listener limited to loopback IPv4                                  |
| `ss` shows `0.0.0.0:8080`        | IPv4 wildcard listener exists                                      |
| local curl succeeds              | local endpoint can answer tested request                           |
| remote curl fails                | investigate bind/firewall/path/source-specific issue               |
| curl HTTP `404`                  | HTTP server reached; path/resource issue likely                    |
| curl HTTP `500`                  | HTTP/application response received; app/dependency issue likely    |
| curl error `6`                   | name resolution failure                                            |
| curl error `7`                   | connection establishment failed                                    |
| curl cert error                  | TLS verification layer                                             |
| curl `200`                       | tested HTTP request succeeded, not necessarily whole system health |

---

# 119. Một nguyên tắc vàng: positive evidence vs negative evidence

Ví dụ:

```bash
ss -lntp
```

không thấy 8080.

Đây là **negative evidence**:

```text
No listener observed at that moment.
```

Nhưng bạn vẫn phải kiểm tra:

```text
correct namespace?
correct host?
correct protocol?
correct port?
service may be restarting?
```

Positive evidence:

```text
java pid 4210 LISTEN 0.0.0.0:8080
```

thường mạnh hơn vì chứng minh một state cụ thể tồn tại.

---

# 120. Một nguyên tắc khác: test từ đúng vantage point

Nếu user báo:

```text
Client A cannot reach Server B.
```

và bạn chỉ SSH vào Server B rồi chạy:

```bash
curl localhost
```

success,

bạn chưa reproduce vấn đề.

Bạn chỉ chứng minh:

```text
Server B → itself works.
```

Incident là:

```text
Client A → Server B
```

Different source path.

Do đó luôn ghi lại:

```text
SOURCE
DESTINATION
PORT
PROTOCOL
TIME
```

---

# 121. `localhost` test và remote test có giá trị khác nhau

### Local test

```bash
curl http://127.0.0.1:8080
```

test:

```text
local TCP stack
listener
HTTP application
```

và bỏ qua phần lớn external network.

### Remote test

```bash
curl http://10.20.30.40:8080
```

từ client khác test thêm:

```text
routing
network path
host firewall
network firewall
external interface binding
```

Do đó cả hai có thể cần thiết.

---

# 122. Không test sai protocol

Server:

```text
TCP/8080 HTTP
```

Bạn chạy:

```bash
curl https://server:8080
```

Có thể lỗi TLS.

Nhưng application chạy:

```text
http://server:8080
```

Đây không phải connectivity failure.

Luôn xác nhận:

```text
scheme
host
port
path
```

---

# 123. Không test sai address family

Nếu hostname resolve:

```text
AAAA → IPv6
A    → IPv4
```

application có thể thử IPv6 trước hoặc theo address selection rules.

Để isolate:

```bash
curl -4 https://example.com
```

hoặc:

```bash
curl -6 https://example.com
```

Tương tự:

```bash
ping -4 example.com
ping -6 example.com
```

Nếu:

```text
IPv4 works
IPv6 fails
```

bạn có valuable evidence:

```text
address-family-specific problem
```

chứ không phải generic "Internet down."

---

# 124. Bind IPv4 vs IPv6

Bạn có thể gặp:

```text
0.0.0.0:8080
```

và:

```text
[::]:8080
```

Không nên luôn giả định:

```text
[::]:8080 automatically means IPv4 works
```

Dual-stack behavior có thể phụ thuộc socket options/kernel configuration/application behavior.

Khi troubleshoot:

```bash
ss -lntp
```

hãy đọc actual listener family/bind state.

---

# 125. Một full request walkthrough

Giả sử Java application:

```text
URL:
https://orders.internal:8443/health
```

Client chạy:

```bash
curl -v https://orders.internal:8443/health
```

### Phase 1 — parse URL

```text
scheme = https
host   = orders.internal
port   = 8443
path   = /health
```

### Phase 2 — resolve

```text
orders.internal
    ↓
10.20.5.25
```

### Phase 3 — route

Kernel:

```text
10.20.5.25 via 10.10.1.1 dev ens5
```

### Phase 4 — network path

Packets travel across networks.

### Phase 5 — TCP

```text
client ephemeral port → server TCP/8443
```

Handshake.

### Phase 6 — firewall

Traffic must satisfy all relevant policies.

### Phase 7 — TLS

Client validates certificate and negotiates encryption.

### Phase 8 — HTTP

```text
GET /health
Host: orders.internal
```

### Phase 9 — app

returns:

```text
HTTP 200
```

Một failure ở mỗi phase có signature khác nhau.

---

# 126. Layer-to-tool mapping

Bạn nên thuộc mapping này:

```text
Interface / IP
    ↓
ip addr / ip -br addr

Route
    ↓
ip route / ip route get

Name resolution
    ↓
getent hosts
/etc/hosts
/etc/resolv.conf

ICMP connectivity
    ↓
ping

Path
    ↓
traceroute

Listening sockets
    ↓
ss

Firewall
    ↓
ufw
firewall-cmd
nft inspection

HTTP/HTTPS/application
    ↓
curl
```

Đây là toolbox của L1 Linux Operations.

---

# 127. Command reference phải hiểu, không chỉ thuộc

| Command                   | Câu hỏi nó trả lời                         |
| ------------------------- | ------------------------------------------ |
| `ip -br addr`             | Interface/IP hiện tại là gì?               |
| `ip route`                | Routing table hiện tại là gì?              |
| `ip route get IP`         | Kernel định route tới IP cụ thể thế nào?   |
| `getent hosts HOST`       | System resolver resolve hostname thành gì? |
| `ping -c 4 IP`            | ICMP request/reply có hoạt động không?     |
| `traceroute -n IP`        | Probe thấy path/hops như thế nào?          |
| `ss -lnt`                 | TCP listening sockets là gì?               |
| `sudo ss -lntp`           | Process nào sở hữu TCP listener?           |
| `ss -lun`                 | UDP bound/listening-style sockets?         |
| `curl URL`                | Application endpoint trả content gì?       |
| `curl -v URL`             | Request đã tiến tới phase nào?             |
| `ufw status verbose`      | UFW state/rules là gì?                     |
| `firewall-cmd --list-all` | firewalld zone config đang áp dụng là gì?  |

---

# 128. Guided Lab — Module 1

Bây giờ ta chuyển từ theory sang một lab có thể thực hiện an toàn trên Linux machine.

## Objective

Tạo một local HTTP service và chứng minh:

```text
IP
port
listener
HTTP
```

hoạt động.

Sau đó quan sát evidence.

---

# 129. Lab assumptions

Giả định:

```text
Linux VM
Python 3 available
No production workload
Port 8080 unused
```

Trước khi bắt đầu:

```bash
ss -lnt | grep ':8080'
```

Nếu không có output:

```text
port 8080 appears not to have a TCP listener
```

Nếu đã có listener:

> Không dùng port 8080. Chọn port lab khác như 18080.

---

# 130. Start test HTTP server

Trong terminal 1:

```bash
python3 -m http.server 8080 --bind 127.0.0.1
```

Bạn vừa yêu cầu Python:

```text
listen TCP/8080
bind only to 127.0.0.1
serve current directory using HTTP
```

Không chạy command này trong directory chứa secrets/sensitive files, vì HTTP server có thể expose các file trong directory đó.

Tốt hơn:

```bash
mkdir -p ~/day5-lab
cd ~/day5-lab
printf 'DAY5-OK\n' > index.html
python3 -m http.server 8080 --bind 127.0.0.1
```

---

# 131. Verify listener

Terminal 2:

```bash
ss -lnt | grep ':8080'
```

Expected invariant:

```text
TCP port 8080 is LISTENING on 127.0.0.1.
```

Có thể thấy:

```text
LISTEN ... 127.0.0.1:8080 ...
```

---

# 132. Verify process

```bash
sudo ss -lntp | grep ':8080'
```

Expected invariant:

```text
listener belongs to python3
```

Không cần output giống hệt ví dụ.

---

# 133. Verify HTTP

```bash
curl http://127.0.0.1:8080/
```

Expected:

```text
DAY5-OK
```

Bây giờ bạn có evidence chain:

```text
python process
    ↓
TCP listener
    ↓
127.0.0.1:8080
    ↓
HTTP response
```

---

# 134. Verbose request

```bash
curl -v http://127.0.0.1:8080/
```

Tìm:

```text
Trying 127.0.0.1:8080
Connected
GET /
HTTP response
```

Đừng chỉ nhìn cuối output.

Đọc flow.

---

# 135. Measure request

```bash
curl -sS \
  -o /dev/null \
  -w 'http=%{http_code} remote=%{remote_ip} connect=%{time_connect} total=%{time_total}\n' \
  http://127.0.0.1:8080/
```

Expected invariant:

```text
HTTP request succeeds.
remote IP is loopback.
HTTP response code indicates success.
```

---

# 136. Failure injection 1 — wrong port

Giữ server 8080 nhưng chạy:

```bash
curl -v --connect-timeout 3 http://127.0.0.1:8081/
```

Bạn có thể thấy:

```text
connection refused
```

Check:

```bash
ss -lnt | grep ':8081'
```

Không có listener.

Root cause:

```text
client is targeting a port with no local listener
```

Remediation:

```text
use correct port
```

Không phải:

```text
restart network
```

---

# 137. Failure injection 2 — stop service

Ở terminal đang chạy Python:

```text
Ctrl+C
```

Check:

```bash
ss -lnt | grep ':8080'
```

Không còn listener.

Sau đó:

```bash
curl http://127.0.0.1:8080/
```

fail.

Evidence chain:

```text
before:
listener existed + HTTP worked

after:
listener missing + HTTP connection fails
```

Root cause rất rõ.

---

# 138. Failure injection 3 — wrong hostname

Test:

```bash
curl --connect-timeout 3 http://this-name-should-not-exist.invalid:8080/
```

`.invalid` được dành cho invalid names, nên thích hợp làm test không-resolve hơn là bịa một public domain có thể tồn tại.

Possible:

```text
Could not resolve host
```

Check:

```bash
getent hosts this-name-should-not-exist.invalid
```

Expected:

```text
no resolution
```

Root cause layer:

```text
name resolution
```

Không phải port 8080.

---

# 139. Cleanup

Nếu test Python server vẫn chạy:

```text
Ctrl+C
```

Lab directory:

```bash
rm -rf ~/day5-lab
```

Chỉ chạy `rm -rf` sau khi:

```bash
pwd
ls -la ~/day5-lab
```

và xác nhận chính xác path.

Do not normalize destructive commands without inspection.

---

# 140. Failure Injection & Troubleshooting Exercise 1

Symptom:

```text
User says:
http://app01:8080 does not work.
```

Evidence:

```bash
getent hosts app01
```

```text
10.10.20.50 app01
```

```bash
ping -c 3 10.10.20.50
```

```text
3 packets transmitted, 3 received
```

Server:

```bash
ss -lnt
```

```text
LISTEN ... 127.0.0.1:8080
```

Local:

```bash
curl http://127.0.0.1:8080
```

```text
OK
```

### Reasoning

DNS?

```text
working
```

Basic ICMP?

```text
working
```

Application running locally?

```text
yes
```

TCP listener?

```text
yes
```

Bind address?

```text
127.0.0.1 only
```

Most relevant root cause:

```text
application listener is bound only to loopback,
so remote clients cannot reach it through the host's external IPv4 address
```

Remediation requires application configuration change.

Do not randomly add firewall rules first.

---

# 141. Exercise 2

Evidence:

```bash
getent hosts app01
```

returns:

```text
10.10.20.50
```

```bash
ss -lnt
```

server:

```text
LISTEN ... 0.0.0.0:8080
```

Server:

```bash
curl http://127.0.0.1:8080
```

returns:

```text
OK
```

Client:

```bash
curl --connect-timeout 3 http://10.10.20.50:8080
```

times out.

### Hypotheses

Now likely areas:

```text
host firewall
external firewall
route/path
client restrictions
return routing
```

Much less likely:

```text
application completely stopped
```

because local request and listener evidence contradict that.

---

# 142. Exercise 3

```bash
curl -v http://app01:8080/health
```

returns:

```text
HTTP/1.1 404 Not Found
```

What failed?

Not:

```text
DNS necessarily
TCP necessarily
firewall necessarily
```

You've successfully received an HTTP response.

Investigate:

```text
correct endpoint path?
correct context path?
correct virtual host?
correct deployment?
```

---

# 143. Exercise 4

```bash
curl https://app01
```

returns certificate verification failure.

User says:

> "Port 443 is blocked."

That conclusion is inconsistent with available evidence.

Certificate verification requires the request to have progressed to TLS communication far enough to receive certificate information.

So focus on:

```text
TLS certificate/trust/name
```

instead of immediately modifying port rules.

---

# 144. Anti-pattern: "Ping fails, server is down"

Wrong.

Correct statement:

```text
ICMP Echo did not receive the expected response.
```

Possible causes remain multiple.

---

# 145. Anti-pattern: "Ping works, network is fine"

Wrong.

Correct:

```text
ICMP Echo communication worked.
```

Still unknown:

```text
TCP port?
firewall per-port?
TLS?
HTTP?
application?
```

---

# 146. Anti-pattern: "Port listening means user can reach it"

Wrong.

```text
LISTEN
```

only proves local socket state.

Still need:

```text
bind address
route
firewall
external security controls
```

---

# 147. Anti-pattern: disable firewall to troubleshoot

Bad:

```bash
sudo ufw disable
```

hoặc:

```text
flush firewall
```

Reason:

- expands attack surface,
- may violate policy,
- can hide true configuration issue,
- can create new incident.

Better:

```text
inspect exact rule
identify exact required traffic
test minimal approved rule
validate
rollback
```

---

# 148. Anti-pattern: restarting services before evidence

User:

```text
Can't connect.
```

Admin:

```bash
sudo systemctl restart ...
```

Problem:

Bạn vừa:

- destroy transient state,
- possibly clear evidence,
- introduce new change,
- potentially increase outage,
- chưa biết root cause.

Trước remediation:

```text
collect evidence
```

---

# 149. Anti-pattern: testing only localhost

Local success:

```bash
curl localhost:8080
```

does not prove:

```text
remote clients can reach 8080
```

Always respect the incident vantage point.

---

# 150. Anti-pattern: conflating service and port

Port number không guarantee service identity.

Nếu:

```text
8080 open
```

không có nghĩa:

```text
expected Java application is behind it.
```

Có thể process khác chiếm port.

Check:

```bash
sudo ss -lntp
```

---

# 151. Anti-pattern: trusting common ports blindly

HTTP commonly uses:

```text
80
```

nhưng app có thể:

```text
8080
9080
3000
```

HTTPS commonly:

```text
443
```

nhưng app có thể:

```text
8443
```

Always use approved configuration/runbook, not assumptions.

---

# 152. Security: listening scope

Suppose an internal admin API should only be local:

```text
127.0.0.1:9000
```

Changing it to:

```text
0.0.0.0:9000
```

may expose it to networks reachable through machine interfaces.

Therefore:

> Changing bind address is a security change, not merely a connectivity fix.

Before changing:

```text
Who should access this?
From where?
Through which proxy?
What authentication exists?
What firewall limits exist?
```

---

# 153. Security: exposure is the intersection of controls

External reachability depends on several controls:

```text
Application bind
      ∩
Host firewall
      ∩
Network firewall
      ∩
Routing
      ∩
Cloud controls
      ∩
Authentication/application policy
```

Opening one does not necessarily make the service reachable.

But opening too many may unnecessarily expose it.

---

# 154. Security: principle of least privilege

For every rule ask:

```text
Source?
Destination?
Protocol?
Port?
Why?
Duration?
Owner?
Rollback?
```

Bad:

```text
allow any → any
```

Better:

```text
only approved source network
→ specific host/service
→ specific TCP port
```

---

# 155. L1 operations boundary

As L1/fresher, safe operations commonly include:

```text
read configuration
check interface/IP
check route
test DNS
run ping/traceroute
inspect sockets
perform approved curl health checks
inspect firewall state
collect timestamps/output
restart approved service if runbook authorizes it
apply preapproved known remediation
validate result
```

---

# 156. Escalation boundary

Escalate when root cause involves things such as:

```text
unapproved routing changes
production firewall redesign
unknown security rules
network architecture changes
DNS authoritative changes outside scope
TLS PKI changes
kernel/network stack anomalies
possible attack/security incident
multi-system outage
changes with uncertain blast radius
```

Escalation is not failure.

Good Operations means:

> fix within authority; escalate cleanly outside authority.

---

# 157. Good escalation package

Bad:

```text
Network not working. Please check.
```

Good:

```text
Symptom:
client01 cannot connect to app01 TCP/8080.

Expected:
app01.internal → 10.20.30.40,
TCP/8080 reachable from client subnet.

Observed:
- DNS resolves app01.internal → 10.20.30.40.
- Client has route via 10.10.1.1.
- ICMP to server succeeds.
- Server has java PID 4210 listening on 0.0.0.0:8080.
- Local curl /health returns HTTP 200.
- Remote curl times out after 3 seconds.
- Host UFW/firewalld inspection shows [relevant state].
- No application configuration changes identified.

Suspected layer:
network/firewall path between client and server.

Impact:
remote health endpoint unavailable.

No firewall changes were made.

Evidence timestamps:
...
```

Đây là escalation có giá trị.

---

# 158. Health-check record — networking portion

Day 5 cuối cùng yêu cầu health-check record. Module 1 nên chuẩn bị phần network như sau:

| Check     | Command                       | Expected                        | Observed | Status         |
| --------- | ----------------------------- | ------------------------------- | -------- | -------------- |
| Interface | `ip -br addr`                 | expected interface UP           | ...      | PASS/FAIL      |
| IP        | `ip -br addr`                 | expected IP/prefix              | ...      | PASS/FAIL      |
| Route     | `ip route get DEST_IP`        | expected path                   | ...      | PASS/FAIL      |
| DNS       | `getent hosts HOST`           | expected IP                     | ...      | PASS/FAIL      |
| ICMP      | `ping -c 4 DEST_IP`           | according to environment policy | ...      | INFO/PASS/FAIL |
| Path      | `traceroute -n DEST_IP`       | path reaches expected network   | ...      | INFO           |
| Listener  | `ss -lntp`                    | expected local port             | ...      | PASS/FAIL      |
| Firewall  | appropriate read-only command | expected access policy          | ...      | PASS/FAIL      |
| HTTP      | `curl ...`                    | expected status                 | ...      | PASS/FAIL      |

---

# 159. Health check phải có timestamp

Ví dụ:

```bash
date -Is
```

Record:

```text
2026-10-01T22:15:30+07:00
```

Vì:

```text
"port was open"
```

không đủ.

Bạn cần:

```text
port was observed listening at TIME T.
```

Operational evidence luôn gắn với thời gian.

---

# 160. Example Day-5 network evidence

```text
Timestamp:
2026-10-01T22:15:30+07:00

Host:
app01

Expected IP:
10.20.30.40/24

Observed:
ens5 UP 10.20.30.40/24

DNS:
app01.internal → 10.20.30.40

Route from client:
10.20.30.40 via 10.10.1.1 dev ens5

Listener:
TCP/8080 LISTEN on 0.0.0.0
Process java PID 4210

Local health:
HTTP 200

Remote health:
connection timeout

Firewall:
host policy inspected; evidence attached

Preliminary scope:
application is locally available; failure appears limited
to remote connectivity path.

Action:
escalated network path evidence; no unapproved firewall
change performed.
```

Đây là cách suy nghĩ của Operations engineer.

---

# 161. Command safety matrix

| Command                   |    Read-only? | Typical privilege                     | Risk                           |
| ------------------------- | ------------: | ------------------------------------- | ------------------------------ |
| `ip -br addr`             |           Yes | user                                  | Low                            |
| `ip route`                |           Yes | user                                  | Low                            |
| `ip route get ...`        |           Yes | user                                  | Low                            |
| `getent hosts ...`        |           Yes | user                                  | Low                            |
| `ping -c ...`             |    Diagnostic | usually user                          | Low, respect network policy    |
| `traceroute ...`          |    Diagnostic | varies                                | Low/moderate; policy-sensitive |
| `ss -lnt`                 |           Yes | user                                  | Low                            |
| `ss -lntp`                |           Yes | process details may require privilege | Low                            |
| `curl URL`                | Sends request | user                                  | Depends on endpoint/method     |
| `ufw status`              |           Yes | often sudo                            | Low                            |
| `firewall-cmd --list-all` |           Yes | often allowed/read access varies      | Low                            |
| `ufw allow ...`           |        **No** | root                                  | High                           |
| `firewall-cmd --add-port` |        **No** | root                                  | High                           |
| `ip addr add/del`         |        **No** | root                                  | High                           |
| `ip route add/del`        |        **No** | root                                  | High                           |

For Day 5, **inspect before modify**.

---

# 162. `curl` GET không phải lúc nào cũng harmless

GET được thiết kế theo HTTP semantics là safe method, nhưng một poorly designed application có thể vẫn có side effects.

Do đó production health check phải dùng:

```text
documented health endpoint
```

như:

```text
/health
/actuator/health
```

nếu application contract xác định như vậy.

Không tự ý gọi endpoints:

```text
/delete
/restart
/run-job
/admin/...
```

chỉ để test connectivity.

---

# 163. Connection troubleshooting decision tree

Hãy thuộc mental flow này.

```text
Application endpoint failed
        │
        ▼
Does hostname resolve?
        │
   ┌────┴────┐
   │         │
  NO        YES
   │         │
   ▼         ▼
 DNS       Correct IP?
 issue       │
        ┌────┴────┐
       NO         YES
        │          │
        ▼          ▼
    DNS/hosts    Route exists?
      issue        │
              ┌────┴────┐
             NO         YES
              │          │
              ▼          ▼
          routing      Target service
           issue       listening?
                          │
                     ┌────┴────┐
                    NO         YES
                     │          │
                     ▼          ▼
                  service    bind address
                   issue      correct?
                                │
                           ┌────┴────┐
                          NO         YES
                           │          │
                           ▼          ▼
                        bind       firewall/
                        issue       network
                                      │
                                      ▼
                                TCP connects?
                                      │
                                 ┌────┴────┐
                                NO         YES
                                 │          │
                                 ▼          ▼
                              network     TLS/HTTP
                              problem     response?
```

Đây không phải algorithm tuyệt đối, nhưng là mental model rất tốt.

---

# 164. Full troubleshooting example

## Incident

User reports:

```text
http://orders.internal:8080/health unavailable.
```

### Step 1 — define source

```text
client01
```

### Step 2 — expected

```text
orders.internal → 10.20.30.40
TCP/8080
HTTP GET /health
Expected HTTP 200
```

### Step 3 — DNS

```bash
getent hosts orders.internal
```

Output:

```text
10.20.30.40 orders.internal
```

Conclusion:

```text
name resolution works and returned expected IPv4.
```

---

# 165. Route

```bash
ip route get 10.20.30.40
```

Output:

```text
10.20.30.40 via 10.10.1.1 dev ens5 src 10.10.1.25
```

Conclusion:

```text
client kernel has route decision toward destination.
```

Not:

```text
destination definitely reachable.
```

---

# 166. Ping

```bash
ping -c 4 10.20.30.40
```

Output:

```text
4 transmitted, 4 received
```

Conclusion:

```text
ICMP Echo communication succeeds.
```

Still unknown:

```text
TCP 8080
```

---

# 167. Curl

```bash
curl -v --connect-timeout 3 \
  http://orders.internal:8080/health
```

Output:

```text
Trying 10.20.30.40:8080...
Connection refused
```

Now focus:

```text
TCP service/firewall reject/bind/wrong port
```

---

# 168. Server listener

On server:

```bash
sudo ss -lntp | grep ':8080'
```

Output:

```text
LISTEN ... 127.0.0.1:8080 ... java ...
```

Now the strongest evidence is:

```text
Java app listening only on loopback.
```

---

# 169. Local application test

```bash
curl -i http://127.0.0.1:8080/health
```

returns:

```text
HTTP/1.1 200 OK
```

Root cause is now highly constrained:

```text
Application is healthy locally,
but TCP listener is bound only to loopback,
so it is not exposed on the server's external IPv4 interface.
```

---

# 170. Remediation reasoning

Do **not** immediately change:

```text
127.0.0.1 → 0.0.0.0
```

Questions:

```text
Was loopback-only intentional?
Is there supposed to be a reverse proxy?
Should remote users ever connect directly?
What does approved architecture say?
What does service config specify?
Would opening it expose an admin interface?
```

The correct remediation may be:

```text
fix reverse proxy
```

rather than exposing Java directly.

This is why troubleshooting ≠ blindly removing restrictions.

---

# 171. Verification after approved remediation

Suppose approved configuration says service should bind:

```text
0.0.0.0:8080
```

After change/restart:

```bash
sudo ss -lntp | grep ':8080'
```

Expected:

```text
0.0.0.0:8080
```

Local:

```bash
curl -i http://127.0.0.1:8080/health
```

Remote:

```bash
curl -i http://10.20.30.40:8080/health
```

Then DNS path:

```bash
curl -i http://orders.internal:8080/health
```

Need validate progressively.

---

# 172. Rollback

Nếu change làm application worse:

```text
restore previous bind configuration
restart using approved procedure
confirm original service state
document rollback
```

Never make a network/application configuration change without knowing:

```text
old value
new value
rollback method
validation method
```

---

# 173. Troubleshooting layering với future curriculum

Module này sẽ trở thành foundation cho toàn khóa:

```text
Client
  ↓
DNS
  ↓
Network
  ↓
AWS VPC / route / SG / NACL
  ↓
EC2 Linux
  ↓
Host firewall
  ↓
TCP listener
  ↓
Tomcat
  ↓
Java app
  ↓
PostgreSQL
```

Sau này khi user nói:

```text
Java app cannot connect to PostgreSQL
```

bạn sẽ không chỉ restart DB.

Bạn sẽ hỏi:

```text
DB hostname resolves?
Route exists?
TCP 5432 reachable?
Postgres listens?
Firewall allows?
pg_hba.conf permits?
Credentials correct?
DB healthy?
```

Đó chính là integrated troubleshooting objective của khóa học.

---

# 174. Exam Focus — các distinction phải nhớ

### 1.

```text
IP address ≠ port
```

### 2.

```text
DNS resolution ≠ connectivity
```

### 3.

```text
ping success ≠ application success
```

### 4.

```text
ping failure ≠ server down
```

### 5.

```text
LISTEN ≠ remotely reachable
```

### 6.

```text
0.0.0.0:8080 ≠ 127.0.0.1:8080
```

### 7.

```text
HTTP 404 ≠ network failure
```

### 8.

```text
HTTP 500 ≠ TCP failure
```

### 9.

```text
firewall allow ≠ application listening
```

### 10.

```text
local curl success ≠ remote curl success
```

Nếu bạn hiểu sâu 10 dòng này, bạn đã nắm được lõi Module 1.

---

# 175. Exam-style question 1

Server:

```bash
ss -lnt
```

shows:

```text
LISTEN 0 100 127.0.0.1:8080 0.0.0.0:*
```

Application works:

```bash
curl localhost:8080
```

Remote client cannot connect.

**Most relevant observation?**

Answer:

```text
Application is listening only on loopback IPv4.
Local success therefore does not establish remote reachability.
```

---

# 176. Question 2

```bash
ping server
```

fails.

```bash
curl https://server
```

returns HTTP 200.

Is this contradictory?

**No.**

`ping` uses ICMP Echo; HTTPS traffic uses a different transport/application path. Firewall/security policy can treat them differently. :chatgpt-content-reference{index="26"}

---

# 177. Question 3

```bash
curl http://app:8080/foo
```

returns:

```text
HTTP 404
```

Should L1 first investigate routing?

Not normally.

You already have evidence that:

```text
HTTP server returned a response.
```

First investigate expected path/context/application routing.

---

# 178. Question 4

`ss` shows:

```text
0.0.0.0:5432
```

Does this prove PostgreSQL is accessible remotely?

No.

It only strongly indicates an IPv4 wildcard TCP listener on that port.

Still need:

```text
firewall
network path
DB access policy
authentication
application health
```

---

# 179. Question 5

DNS resolves:

```text
app → 10.0.0.20
```

but runbook says:

```text
app → 10.0.0.30
```

What should you do first?

Not:

```text
restart app
```

Instead:

```text
record discrepancy
check /etc/hosts/resolver path
confirm expected DNS state
escalate/change only within authorization
```

---

# 180. Knowledge Check

Hãy tự trả lời trước khi đọc đáp án.

### Q1

Port thuộc layer/concept nào gần nhất?

### Q2

`ping` sử dụng TCP hay ICMP?

### Q3

Nếu `ping` success, có chứng minh TCP/8080 open không?

### Q4

`ss -lnt` dùng để làm gì?

### Q5

`127.0.0.1:8080` khác `0.0.0.0:8080` thế nào?

### Q6

`curl` error 6 thường hướng tới layer nào?

### Q7

HTTP 404 chứng minh điều gì về network/application path?

### Q8

Nếu local curl works nhưng remote curl fails, ba khu vực bạn nên nghĩ ngay là gì?

### Q9

`ip route get DEST` có giá trị gì?

### Q10

Tại sao không nên disable firewall để troubleshoot?

### Q11

`traceroute` dựa vào IP field nào để khám phá hop?

### Q12

Một traceroute hop hiện `* * *` có chứng minh router đó down không?

### Q13

Port 53 TCP và port 53 UDP có giống nhau không?

### Q14

Nếu `ss` thấy listener, có chứng minh endpoint HTTP `/health` trả 200 không?

### Q15

Firewall rule nên tuân security principle nào?

---

# 181. Knowledge Check — đáp án

### A1

Port thuộc transport endpoint concept, thường được hiểu cùng:

```text
TCP/UDP
```

và IP address.

### A2

```text
ICMP
```

### A3

Không.

### A4

Hiển thị TCP listening sockets với numeric addresses/ports.

### A5

```text
127.0.0.1
```

chỉ loopback IPv4.

```text
0.0.0.0
```

ở bind/listener context thường là wildcard/all local IPv4 addresses.

### A6

Name resolution.

### A7

Bạn đã nhận HTTP response, vì vậy communication đã tiến tới HTTP server/application path; resource/path có thể sai.

### A8

Ít nhất:

```text
bind address
firewall/security controls
routing/network path
```

### A9

Cho biết kernel sẽ route tới destination cụ thể như thế nào.

### A10

Vì nó tăng attack surface, thay đổi production security state, có thể gây outage và che mất root cause.

### A11

```text
TTL
```

### A12

Không.

### A13

Không. Chúng là endpoints của hai transport protocols khác nhau.

### A14

Không.

### A15

```text
least privilege
```

---

# 182. Independent Practical Challenge

Đây là challenge để kiểm tra bạn có thực sự hiểu Module 1 hay chưa.

Bạn được giao incident:

```text
Users on client01 cannot access:
http://app01.internal:8080/health
```

Expected architecture:

```text
client01
   |
   | TCP 8080
   v
app01.internal
10.20.30.40
```

Bạn có quyền:

```text
SSH client01
SSH app01
read configuration
run diagnostic commands
```

Bạn **không được**:

```text
disable firewall
change route
change DNS
modify app configuration
restart service
```

trước khi xác định failure layer.

Nhiệm vụ của bạn là thu thập evidence theo thứ tự hợp lý:

```bash
ip -br addr

getent hosts app01.internal

ip route get 10.20.30.40

ping -c 4 10.20.30.40

traceroute -n 10.20.30.40
```

Trên server:

```bash
ip -br addr

sudo ss -lntp
```

Inspect appropriate firewall:

```bash
sudo ufw status verbose
```

hoặc:

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

Local endpoint:

```bash
curl -v --connect-timeout 3 \
  http://127.0.0.1:8080/health
```

Remote:

```bash
curl -v --connect-timeout 3 \
  http://app01.internal:8080/health
```

Sau đó phải viết:

```text
Symptom
Expected
Source
Destination
Protocol
Port
Evidence
Failure layer
Hypotheses
Most likely root cause
Safe remediation
Validation plan
Rollback plan
Escalation boundary
```

Nếu bạn chỉ đưa một danh sách command mà không giải thích evidence, challenge chưa đạt.

---

# 183. Operational command sequence nên thuộc

Không phải lúc nào cũng chạy nguyên chuỗi này, nhưng bạn cần quen:

```bash
# Identity / interface / IP
ip -br addr

# Routing
ip route
ip route get <destination-IP>

# Name resolution
getent hosts <hostname>
cat /etc/resolv.conf

# Basic IP/ICMP evidence
ping -c 4 <destination-IP>

# Network path
traceroute -n <destination-IP>

# TCP listeners
ss -lnt
sudo ss -lntp

# UDP sockets
ss -lun

# Application endpoint
curl -v --connect-timeout 3 --max-time 10 \
  http://<host>:<port>/<path>

# HTTP health metrics
curl -sS --connect-timeout 3 --max-time 10 \
  -o /dev/null \
  -w 'http=%{http_code} remote=%{remote_ip} dns=%{time_namelookup}s connect=%{time_connect}s total=%{time_total}s\n' \
  http://<host>:<port>/<path>

# Ubuntu-style host firewall
sudo ufw status verbose

# RHEL-style host firewall
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
```

Các semantics cốt lõi của `ss`, `ping`, `traceroute`, `ip` và curl ở trên phù hợp với tài liệu hiện hành của respective projects/man pages. :chatgpt-content-reference{index="27"}

---

# 184. Mental checklist khi thấy "connection problem"

Đừng nghĩ:

```text
network broken
```

Hãy nghĩ:

```text
WHAT EXACTLY FAILED?

Name?
 │
DNS

Address?
 │
IP

Path?
 │
Route

Reachability?
 │
ICMP/path evidence

Port?
 │
TCP/UDP

Listener?
 │
ss

Bind?
 │
127.0.0.1 / specific IP / wildcard

Policy?
 │
Firewall

Protocol?
 │
HTTP/HTTPS

Endpoint?
 │
/health

Response?
 │
2xx / 4xx / 5xx

Dependency?
 │
application/database/etc.
```

---

# 185. Module 1 mastery model

Ở mức **Beginner**, bạn nói:

> "`ping` kiểm tra mạng."

Ở mức tốt hơn:

> "`ping` gửi ICMP Echo Request."

Ở mức Operations-ready:

> "`ping` là một nguồn evidence về ICMP reachability. Ping success không chứng minh application port reachable, và ping failure không chứng minh host down vì ICMP có thể bị filtered."

Tương tự với `ss`.

Beginner:

> "`ss` xem port."

Operations-ready:

> "`ss -lntp` cho tôi evidence về local TCP listening sockets, bind addresses và process ownership. Listener tồn tại không chứng minh remote reachability; tôi còn phải kiểm tra bind scope, firewall, route và application protocol."

Với `curl`.

Beginner:

> "`curl` gọi URL."

Operations-ready:

> "`curl -v` cho phép tôi quan sát request tiến qua resolution → connect → TLS → HTTP đến đâu; `--resolve` giúp isolate DNS khỏi server/TLS/application; exit codes và HTTP status phải được phân biệt."

Đó mới là mastery.

---

# 186. Module 1 summary

Bạn cần giữ trong đầu một flow duy nhất:

```text
CLIENT
  │
  │ hostname
  ▼
NAME RESOLUTION
  │
  │ IP address
  ▼
ROUTING
  │
  │ interface/gateway
  ▼
NETWORK PATH
  │
  │ ICMP / TCP / UDP
  ▼
FIREWALL
  │
  │ allowed traffic
  ▼
SERVER IP
  │
  ▼
SOCKET / PORT
  │
  │ ss
  ▼
PROCESS
  │
  ▼
TLS / HTTP
  │
  │ curl
  ▼
APPLICATION
```

Và map tool:

```text
IP/interface  → ip
DNS           → getent / resolver evidence
reachability  → ping
path          → traceroute
port/socket   → ss
firewall      → ufw / firewalld
application   → curl
```

Câu quan trọng nhất của toàn Module 1 là:

> **Không có một command duy nhất chứng minh “network is fine”.**

Thay vào đó, bạn xây dựng một **chain of evidence**:

```text
hostname resolved
→ correct IP
→ route exists
→ appropriate network evidence
→ socket listening
→ correct bind address
→ firewall policy appropriate
→ TCP connection established
→ TLS successful if applicable
→ HTTP response received
→ expected endpoint status/content
```

Chỉ khi reasoning theo chain này, bạn mới có thể phân biệt chính xác **DNS issue, route issue, ICMP behavior, firewall issue, closed port, wrong bind address, TLS problem, HTTP problem và application failure**.

Về syllabus, **toàn bộ mandatory networking scope của Day 5 Module 1 đã được trình bày**: IP/DNS/ports, `ping`, `traceroute`, `ss`, `curl` và firewall basics; phần disks/filesystems/mounts/capacity vẫn thuộc **Module 2** trước khi chúng ta hoàn thành bài Assignment/Lab tích hợp của Day 5. :chatgpt-content-reference{index="28"}
