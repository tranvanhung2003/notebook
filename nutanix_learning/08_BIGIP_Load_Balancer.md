> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# F5 BIG-IP Load Balancer

## 1. Liên hệ với dự án
**[Nguồn dự án]** LB add/change/delete là nhóm tần suất cao, có workday ngoài giờ; có SSL certificate request, config management, qkview/iHealth, OS upgrade và issue-driven LB config change. SOW nhắc Excel macro tạo CLI, Python/API tooling, BIG-IP WebUI, TeraTerm, WinMerge.

## 2. Mental model LTM
Traffic path cơ bản:
Client → Virtual Server → profiles/policy/iRule → Pool → Pool Member.

Các object chính:
- **Virtual Server**: IP:port nhận traffic.
- **Pool**: nhóm backend.
- **Pool Member**: server IP:port.
- **Monitor**: health check.
- **Node**: backend IP entity.
- **Profile**: TCP/HTTP/SSL behavior.
- **SNAT**: source NAT.
- **iRule/Local Traffic Policy**: logic nâng cao.

## 3. Health monitor
Một pool member “UP” chỉ nghĩa monitor pass. Nếu monitor quá đơn giản (TCP connect) thì application có thể vẫn lỗi. Cần biết monitor type, interval/timeout, receive string và dependency.

## 4. Load balancing methods
Round robin, least connections, ratio… lựa chọn ảnh hưởng traffic distribution. Đừng đổi algorithm khi incident nếu chưa có evidence.

## 5. SSL/TLS trên BIG-IP
### Client-side SSL
BIG-IP terminate TLS từ client. Cần certificate + private key + chain/profile.

### Server-side SSL
BIG-IP tạo TLS đến backend.

### Checklist certificate
- CN/SAN đúng hostname.
- NotBefore/NotAfter.
- Key match certificate.
- Chain/intermediate đầy đủ.
- Cipher/TLS policy.
- SNI nếu nhiều hostname.
- Renewal/rollback plan.

**[Nguồn dự án]** Có nghiệp vụ “SSL certificate application” và quyền admin cho Virtual Server/Pool/SSL certificate.

## 6. Change workflow LB
1. Validate VIP/backend/port/protocol.
2. Identify existing Virtual Server/pool/profile.
3. Capture status/traffic/current config.
4. Generate proposed CLI/material.
5. Peer review bằng WinMerge/diff.
6. Config backup.
7. Apply object theo dependency: node/pool/monitor/profile → virtual server.
8. Test health + application.
9. Monitor errors/connection count.
10. Evidence + configuration inventory.

## 7. `tmsh` read-only examples
```text
tmsh show sys version
tmsh show sys failover
tmsh show ltm virtual
tmsh show ltm pool
tmsh list ltm virtual <name>
tmsh list ltm pool <name>
tmsh show sys connection
```
Tên partition/object và command detail tùy environment.

## 8. Config backup và support data
- UCS: config backup archive (sử dụng theo procedure và bảo vệ secret).
- qkview: diagnostic bundle.
- iHealth: F5 support analysis portal; source project có procedure upload qkview.

Không gửi qkview/backup ra ngoài nếu chưa theo data-handling policy.

## 9. HA
Cần biết:
- Device trust/traffic group.
- Active/standby.
- Config sync.
- Failover status.

Trước change phải xác nhận sync/HA healthy. Sau change, kiểm tra sync state; không “force sync” nếu chưa hiểu direction vì có thể overwrite config tốt.

## 10. iControl REST/API mental model
BIG-IP cung cấp REST API dưới `/mgmt/tm/...` cho nhiều object. Ví dụ read-only:
```bash
curl -u "$F5_USER:$F5_PASS" \
  https://bigip.example/mgmt/tm/ltm/virtual
```
Trong production nên dùng token/secret storage phù hợp; tránh password trong command history. Không dùng `-k` như mặc định production; xử lý CA trust đúng.

## 11. Troubleshooting
### VIP không vào được
- DNS/IP route tới VIP?
- Virtual Server enabled/available?
- Pool/member health?
- Firewall trước BIG-IP?
- SNAT/return route?
- SSL handshake?

### 5xx tăng
- Backend app health/log.
- Pool member distribution.
- HTTP profile/iRule.
- TLS backend.
- Recent deployment/change.

### Member DOWN
- Monitor config.
- Backend port/service.
- Route/firewall.
- Monitor source/SNAT.

## 12. Upgrade TMOS/F5OS
**[Nguồn dự án]** Có LB OS upgrade; transition docs nói past case TMOS và có kế hoạch chassis replacement/F5OS tương lai. Vì vậy cần phân biệt software platform theo hardware/version thực tế.

Pre/post phải bao gồm HA, config sync, license, VIP/pool health, SSL, traffic, logs.

## 13. Lab
- Dựng HAProxy/nginx để hiểu VIP-pool-member rồi map sang BIG-IP.
- Tạo self-signed chain lab và test SNI/TLS.
- Viết Python/cURL read-only inventory virtual/pool.
- Viết change template có exact rollback.
