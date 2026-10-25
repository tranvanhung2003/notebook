> **Mục đích**: file này chỉ ghi các điểm có thể truy vết trực tiếp về tài liệu dự án. Kiến thức kỹ thuật nền nằm ở các file bài học tương ứng.

# Truy vết nguồn dự án → nội dung học

## 1. Tài liệu nguồn và vai trò
| File | Sheet/phần đã dùng | Thông tin chính |
|---|---|---|
| `Nutanix (HCNET)_Overview.xlsx` | `Overview` | scope sản phẩm/nghiệp vụ, task không thường xuyên, upgrade, incident/config change |
|  | `Tool-OS-URL` | danh sách tool/OS/WebUI thực tế |
|  | `SOW_VN` | nghiệp vụ, nội dung, tần suất, manual, tool, skill, quyền thô |
|  | `Skill set` | product skill và skill tương đương |
|  | `Scope` | định nghĩa standard/semi-standard/non-standard và difficulty |
| `Nutanix_HCNET_アカウント権限.xlsx` | `権限別まとめ` | quyền read/operator/admin theo nghiệp vụ |
|  | `作成メモ` | nguyên tắc suy ra quyền; ServiceNow common permission được lược khỏi một số dòng |
| `Nutanix_新計画スケジュール_v1.1.xlsx` | `詳細スケジュール(新計画)` | 89 nghiệp vụ, common sessions, KT/pre-op, freeze rules |
|  | `業務一覧(グループ分類)` | tần suất, difficulty, test method |
|  | `【最終】Nutanix_HCNET` | ước tính KT/pre-op và 9 vendor-specific tools của HCNET/Nutanix |
| `システム凍結期間予定日程.xlsx` | `システム凍結期間基準` | quy tắc freeze theo kỳ closing |
| `作業スケジュール_202601~.xlsx` | `スケジュール` | lịch work thực tế, gồm RHEL, AOS/AHV, HYCU, LB/FW, AWS… |
| `移管計画_Nutanix_20261023.pptx` | transition plan | 4 phương thức transition, chuẩn bị No.1–7, lịch tổng và night/holiday handling |

## 2. AWS / S3
Quyền trong file account:
- `6-1-98`: AWS environment setting add – S3 bucket add.
- `6-1-99`: S3 bucket setting change.
- `6-1-100`: S3 bucket delete.
- `6-1-138～139`: issue-driven AWS setting change.
- `6-1-101～102`: SecurityHub check/report.
- `6-1-103`: AWS line maintenance information check.

Nguồn phân quyền:
- AWS Console admin cho S3 create/change/delete.
- AWS Console read cho SecurityHub/Health Dashboard/inquiry/alert/incident.
- HYCU privileged admin cũng được liệt kê liên quan `6-1-98～100`.

## 3. Nutanix / Prism / AOS / AHV
Quyền/operation nổi bật:
- `6-1-50～53`: VM create.
- `6-1-54～57`: VM setting change.
- `6-1-58～59`: VM delete.
- `6-1-60～63`: VM start/stop.
- `6-1-64～73`: Volume Group create/change/delete.
- `6-1-75～76`: unnecessary alert delete.
- `6-1-77～78`: capacity management.
- `6-1-82～97`: AsyncDR setup/change/ad-hoc backup/restore/delete.
- `6-1-108～110`: VM image create/delete.
- `6-1-111`: Nutanix software version upgrade.
- `6-1-112`: Nutanix script create/modify.
- `6-1-125`: curator collection.
- `6-1-131`: production CVM Mem available check.
- `6-1-137`: snapshot inventory.
- `6-1-150～151`: issue-driven Nutanix settings change.

Quyền được tách thành Prism Central read/operator/admin, Prism Element data-protection/upgrade và CVM privileged SSH/ncli/acli.

Work schedule còn có:
- RHEL provisioning trên Prism.
- AOS/AHV version-up operation trên STG.

## 4. HYCU
- `6-1-30～31`: backup setting add.
- `6-1-32～35`: policy assignment cho VM/VG.
- `6-1-36～38`: setting change/delete.
- `6-1-39`: ad-hoc backup.
- `6-1-40～43`: restore.
- `6-1-44`: HYCU version upgrade.
- `6-1-45～46`: periodic policy/config checks.
- `6-1-148～149`: issue-driven HYCU setting change.

Quyền tách read / ad-hoc+restore operator / policy admin / system admin.

## 5. BIG-IP
- `6-1-1～4`: LB setting add.
- `6-1-5`: SSL certificate request.
- `6-1-6～9`: LB setting change.
- `6-1-10～13`: LB setting delete.
- `6-1-14`: LB config management/SVN.
- `6-1-140,143`: issue-driven LB setting change.
- `6-1-156～157`: LB OS upgrade.

Source SOW cho thấy material có Excel macro tạo CLI, Python/API scripts, qkview/iHealth procedure, BIG-IP WebUI/TeraTerm/WinMerge.

## 6. FortiGate
- `6-1-16～19`: FW policy add.
- `6-1-20～23`: FW policy change.
- `6-1-24～27`: FW policy delete.
- `6-1-28`: FW config management/SVN.
- `6-1-144～145`: issue-driven FW setting change.
- `6-1-158～159`: FW OS upgrade.

Source chỉ rõ quyền VDOM policy/object admin và `.xlsm` tool tạo config material.

## 7. Cisco Nexus / Catalyst
- `6-1-104～105`: Nexus OS upgrade.
- `6-1-106～107`: Catalyst OS upgrade.
- `6-1-136`: TAC-PAC collection.
- `6-1-117～118`: alert response có read/show access.

## 8. EventLog Analyzer / Firewall Analyzer
EventLog Analyzer:
- `6-1-48`: version upgrade.
- `6-1-49`: ManageEngine license renewal (shared context).
- `6-1-124`: audit/access-log acquisition.
- `6-1-129`: post-restart health check.
- `6-1-154～155`: issue-driven settings change.
- `6-1-161`: periodic network-device log check.

Firewall Analyzer:
- `6-1-47`: version upgrade.
- `6-1-49`: license renewal.
- `6-1-130`: post-restart health check.
- `6-1-152～153`: issue-driven settings change.
- `6-1-160`: communication log collection by inquiry.

## 9. HPE / iLO
- `6-1-146～147`: issue-driven HPE server settings change.
- `6-1-163`: HPE firmware upgrade.
- Inquiry/alert/incident/audit tasks có iLO read access.

## 10. ServiceNow
File quyền ghi trực tiếp:
- `6-1-80`: file transfer request.
- `6-1-133`: privileged-ID request.
- `6-1-162`: entry/exit procedures.
- `6-1-122`: temporary monitoring suppression request trên SNOW/NowPlatform.

Transition plan yêu cầu tập hợp ServiceNow filter memo/past cases và schedule common explanation riêng cho ServiceNow.

## 11. Custom tools và data processing
Các dấu vết trực tiếp trong SOW:
- `.xlsm` LB application/template/master; CLI generated bằng macro Excel.
- `.xlsm` FortiGate config generation tool.
- Visual Studio Code/Python scripts và API-related LB tool procedure.
- `6-1-112`: Nutanix script creation/modification.
- Email extraction macro trong alert operation.
- Access-log format transcription tool được ghi là Python tool trong `.xlsm` context.
- `huGetDiskInfo.bat`.
- password/string generation batch.
- `6-1-126`: daily CSV received by email → transfer into Excel running-node list.

Không thấy literal `PowerShell`, `Outlook`, `MAPI`, `cURL`, `Regex` trong source được cung cấp; các chủ đề này trong bộ học là phần bổ sung theo yêu cầu người dùng.

## 12. RHEL và Windows
RHEL xuất hiện trong work schedule:
- `Ansible自動化対応_OS払出(RHEL7.9/9.6)`.
- `RHEL9運用機能検証_バックアップポリシー解除`.
- `RHEL9運用機能検証_VM廃止`.

SOW có yêu cầu kiến thức initial setup Windows/Linux ở một số task. Windows xuất hiện rộng trong cột OS/tool, nhưng không có version/edition cụ thể; do đó bài Windows Server không khẳng định môi trường đang dùng edition nào.

## 13. Transition / lịch / vận hành
Plan cuối nêu:
- 89 nghiệp vụ.
- Common explanation theo các category monitoring, HYCU, Nutanix/virtualization, LB, FW, AsyncDR, AWS.
- Production shadowing cho task tần suất cao/trung bình.
- Simulation cho AsyncDR/VG/S3/image và task ít phát sinh.
- Past-case walkthrough + OJT cho version upgrade.
- Case study cho inquiry/incident/settings change.
- LB/FW có work ngoài giờ; LB có trường hợp muộn hơn 23:00; VM có khả năng ngoài giờ/holiday.
- Kế hoạch mới tránh bắt đầu production-change KT/pre-op vào freeze periods đã xác định.
