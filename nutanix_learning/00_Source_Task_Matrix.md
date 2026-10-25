> **Cách đọc tài liệu**  
> - **[Nguồn dự án]**: thông tin có thể đối chiếu trực tiếp trong các file nguồn được cung cấp.  
> - **[Kiến thức nền]**: kiến thức kỹ thuật tổng quát được bổ sung để bạn hiểu và thực hành; không nên coi là cấu hình thực tế của môi trường HCNET nếu chưa được xác nhận trong KT.  
> - **[Cần xác nhận trong KT]**: thông tin phụ thuộc phiên bản, topology, policy nội bộ, tên role, địa chỉ, credential, maintenance window hoặc quy trình phê duyệt thực tế.


# Ma trận nội dung học ↔ nghiệp vụ được thấy trong nguồn

| Chủ đề | Nghiệp vụ/đầu mối từ nguồn | Quyền/tool nổi bật |
|---|---|---|
| AWS S3 | S3 bucket add/change/delete; AWS issue-driven config change; SecurityHub/Health | AWS Console read/admin, HYCU admin liên quan S3 |
| Nutanix | VM create/change/delete/start/stop; VG; image; capacity; alerts; snapshot inventory; AsyncDR; upgrade; script | Prism Central read/operator/admin; Prism Element; CVM SSH/ncli/acli |
| HYCU | policy add/change/delete, assign VM/VG, ad-hoc backup, restore, monthly check, upgrade | HYCU read/operator/admin |
| HPE/iLO | alert/inquiry/incident visibility; HPE config change; firmware upgrade | iLO read/admin |
| BIG-IP | LB add/change/delete; SSL; config management; issue change; OS upgrade | WebUI/CLI admin, read role, iHealth, Python/API, `.xlsm` |
| FortiGate | policy/object add/change/delete; config management; issue change; OS upgrade | WebUI/CLI, VDOM admin, `.xlsm`, TeraTerm |
| Nexus/Catalyst | alert check; OS upgrade; TAC-PAC | show-only / privilege 15 |
| EventLog Analyzer | log search, NW log periodic check, post-restart check, upgrade, settings change | read/admin WebUI |
| Firewall Analyzer | communication log search, post-restart check, upgrade, settings change | read/admin WebUI |
| ServiceNow | file transfer, privileged ID, entry/exit, monitoring suppression; request intake | SNOW/NowPlatform request permission |
| Custom tools | LB/FW material generation, email extraction macro, access-log transcription, batch, CSV→Excel, Nutanix script | Excel `.xlsm`, Python, Batch, VS Code, WinMerge, SVN |
| RHEL | RHEL7.9/9.6 provisioning automation; RHEL9 operation-function tests | Prism/HYCU context |
| Windows | OS/tool platform rộng trong SOW; server setup knowledge | Windows/Excel/PowerShell foundation |

## Ghi chú quan trọng
1. Nguồn không xác nhận version/model chính xác của mọi product.
2. Source dùng cả ID `6-0-*` và `6-1-*`; cần xác nhận mapping.
3. PowerShell, Outlook/MAPI, cURL và Regex không được nêu trực tiếp; chúng là kiến thức bổ sung.
4. VBA không được ghi bằng literal “VBA”, nhưng `.xlsm`/macro xuất hiện nhiều lần.
5. Source nói 9 tool riêng của HCNET/Nutanix, nhưng danh sách 9 tool chính thức không hiện đầy đủ trong các sheet đã cung cấp.
