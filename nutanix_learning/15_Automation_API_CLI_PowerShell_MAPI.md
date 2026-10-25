> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# BIG-IP API/cURL/SSL, FortiGate CLI, Outlook/MAPI, PowerShell và automation test

## 1. Phạm vi nguồn vs kiến thức bổ sung
**[Nguồn dự án]** Có Python/API tooling cho BIG-IP, SSL certificate task, FortiGate CLI-oriented material, email extraction macro và script maintenance. **PowerShell, cURL, Outlook/MAPI và Regex không được nêu trực tiếp** trong SOW đã đọc; chúng được bổ sung vì bạn yêu cầu và hữu ích để bảo trì automation trên Windows.

## 2. Nguyên tắc automation production
Automation tốt phải:
- idempotent hoặc biết detect current state,
- validate input,
- dry-run/preview nếu có thể,
- log structured,
- timeout/retry có giới hạn,
- fail closed,
- không expose secret,
- có test,
- có rollback/manual fallback.

## 3. cURL essentials
```bash
curl --fail-with-body --silent --show-error \
  -H 'Accept: application/json' \
  https://example/api
```
Các option cần hiểu:
- `-X` method.
- `-H` header.
- `-d` body.
- `-u` basic auth.
- `--cacert` CA bundle.
- `--connect-timeout`, `--max-time`.
- `--fail-with-body` fail HTTP >=400 nhưng vẫn giữ body.

Tránh `-k/--insecure` trong automation production; nó vô hiệu certificate validation.

## 4. BIG-IP iControl REST
Read-only inventory ví dụ:
```bash
curl --fail-with-body \
  -u "$F5_USER:$F5_PASS" \
  -H 'Accept: application/json' \
  https://bigip.example/mgmt/tm/ltm/virtual
```
Pool:
```text
GET /mgmt/tm/ltm/pool
```
Automation change nên làm:
1. GET current object.
2. Validate expected current state.
3. Save JSON evidence.
4. PATCH/POST only changed fields.
5. GET again.
6. Run functional health check.

**[Cần xác nhận trong KT]** auth method/token endpoint, partition, API version, approved library và endpoint naming thực tế.

## 5. TLS/SSL cho API và LB
Bạn cần hiểu:
- private key,
- certificate,
- CSR,
- CA/intermediate/root,
- SAN,
- chain,
- trust store,
- expiration,
- TLS version/cipher.

OpenSSL lab:
```bash
openssl x509 -in cert.pem -noout -subject -issuer -dates -ext subjectAltName
openssl s_client -connect host.example:443 -servername host.example -showcerts
```
Không copy private key vào ticket/chat.

## 6. FortiGate CLI automation
Read-only first:
```text
get system status
get router info routing-table all
show firewall policy
show firewall address
```
Nếu automation generate config, nên output candidate script để peer review thay vì auto-push ngay ở giai đoạn đầu bàn giao.

Pattern config:
```text
config firewall address
    edit "OBJ_APP01"
        set subnet 10.0.0.10 255.255.255.255
    next
end
```
Test phải verify quoting, duplicate name, CIDR/mask, VDOM và policy ordering.

## 7. PowerShell fundamentals
### Objects, không phải text
```powershell
Get-Service | Where-Object Status -eq 'Running' | Select-Object Name,DisplayName
```
Đừng parse output table bằng regex nếu cmdlet trả object.

### REST
```powershell
$headers = @{ Accept = 'application/json' }
$r = Invoke-RestMethod -Uri 'https://example/api' -Headers $headers -Method Get
$r | ConvertTo-Json -Depth 10
```

### Error handling
```powershell
$ErrorActionPreference = 'Stop'
try {
    # action
} catch {
    Write-Error $_
    exit 1
}
```

### Secret
Dùng SecretManagement/Credential Manager/vault phù hợp; không hard-code token trong `.ps1`.

## 8. Outlook Object Model / MAPI
### Mental model
MAPI là messaging API/data model. Outlook Object Model là COM automation layer phổ biến trên Windows desktop Outlook.

Use cases:
- tìm mail theo folder/time/subject,
- save attachment,
- tạo draft,
- parse alert mail.

PowerShell COM ví dụ học tập:
```powershell
$outlook = New-Object -ComObject Outlook.Application
$ns = $outlook.GetNamespace('MAPI')
$inbox = $ns.GetDefaultFolder(6) # olFolderInbox
$items = $inbox.Items
$items.Sort('[ReceivedTime]', $true)
$items | Select-Object -First 10 Subject,ReceivedTime
```
Cảnh báo:
- phụ thuộc Outlook desktop/profile/bitness/session.
- COM object cần release cẩn thận trong automation dài.
- Security prompt/policy có thể chặn.
- New Outlook có compatibility khác; **[Cần xác nhận]** client thực tế.

Nếu mục tiêu là cloud mailbox automation quy mô lớn, Microsoft Graph thường phù hợp hơn MAPI/COM, nhưng điều này phải phù hợp policy dự án.

## 9. Email attachment processing
Pipeline an toàn:
1. Filter sender/subject/time.
2. Validate attachment extension/name.
3. Save vào quarantine/work folder.
4. Check duplicate/hash.
5. Parse CSV/log.
6. Produce report.
7. Move/tag processed message nếu quy trình cho phép.

Không execute attachment/script trực tiếp từ mail.

## 10. Automation testing pyramid
### Unit
Parser/validator/generator.

### Contract/API mock
Mock BIG-IP/API response; test 200/401/404/409/500/timeout.

### Integration STG
Read-only inventory, then controlled create/update/delete on lab object.

### End-to-end
Request fixture → generated material → API/CLI change → post-check → evidence.

## 11. Idempotency
Ví dụ desired state: pool member phải tồn tại với ratio=1.
- Nếu chưa có → create.
- Nếu có và đúng → no-op.
- Nếu có nhưng khác → change với diff.

Automation không idempotent dễ tạo duplicate object hoặc repeated side effect.

## 12. Retry
Chỉ retry lỗi transient: timeout, 429/503 theo policy. Không retry 400 validation hoặc 401 auth như vòng lặp vô hạn.

## 13. Structured logging
Mỗi log record nên có:
- timestamp,
- run_id,
- ticket_id,
- target,
- action,
- result,
- duration,
- error_code.

Không log password/token/private key.

## 14. Lab cuối
Xây một automation read-only:
1. Đọc CSV danh sách VIP.
2. Validate IP/port bằng Python/PowerShell.
3. Gọi mock BIG-IP REST GET.
4. So sánh desired/current.
5. Sinh Markdown/CSV report, **không thực hiện PATCH**.
6. Viết unit tests và 5 negative cases.
7. Sau khi pass, mới thiết kế change mode có explicit `--apply` và confirmation/approval control.
