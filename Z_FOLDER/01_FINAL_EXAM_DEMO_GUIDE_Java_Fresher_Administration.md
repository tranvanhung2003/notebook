# HƯỚNG DẪN CHUẨN BỊ & TRÌNH BÀY DEMO

## Lớp Enrichment L1.5 — Administration & Technical Configuration (Linux, AWS, Java Application & Operations)

> Tài liệu dành cho học viên lớp L1.5 chuẩn bị buổi demo tổng hợp lý thuyết + thực hành đã học.
> **Lưu ý:** Đây là checklist bắt buộc **điền/trả lời cụ thể** (không chấp nhận trả lời chung chung kiểu "em đã học về..."). Nếu không trả lời được mục nào, học viên phải **ôn lại và retake** phần tương ứng trước ngày demo.
> **Quan trọng — Hình thức đánh giá:** Trainer và Admin lớp **không tham gia trực tiếp** vào buổi trình bày. Trainer chỉ xem lại **video recording** sau đó để ghi nhận và đánh giá. Vì vậy, học viên phải tự trình bày đầy đủ, rõ ràng, tự giải thích kỹ từng bước **như thể đang hướng dẫn người xem không có mặt tại chỗ** — không được dựa vào việc hỏi-đáp trực tiếp để làm rõ vấn đề.

---

## 1. Mục đích

Mỗi học viên tự trình bày lại toàn bộ lý thuyết và thao tác thực hành đã học trong chương trình 22 ngày (Linux → AWS → Java/Tomcat → PostgreSQL/Monitoring → Operations Controls), đồng thời chứng minh đã nắm vững **3 kỹ năng nền tảng** mà trainer đã hướng dẫn xuyên suốt khóa học: tư duy OSI Mindset khi debug, thói quen lưu/ghi lại lịch sử thao tác, và cách dùng AI để học/ôn tập đúng cách — không chỉ copy-paste mà không hiểu.

---

## 2. Chuẩn bị trước buổi demo

### 2.1. Tạo lịch họp (Meeting) và ghi hình

- Học viên **tự chủ động tạo meeting** (Teams) cho buổi demo của mình.
- **Trainer và Admin lớp L1.5 sẽ không tham dự trực tiếp** — họ chỉ xem lại video recording sau buổi trình bày để ghi nhận. Học viên vẫn cần:
  - Thêm Admin lớp L1.5 và Trainer vào danh sách mời (để họ nhận quyền truy cập recording/transcript tự động), nhưng **không bắt buộc hai người này phải vào họp cùng lúc**.
  - Có thể tự trình bày một mình, miễn đảm bảo video ghi lại đầy đủ và rõ ràng toàn bộ phần trình bày.
- Đặt tên meeting rõ ràng, gợi ý format: `[L1.5] Demo tổng hợp - <Tên học viên> - <Ngày>`.
- **Bắt buộc bật tính năng ghi hình (Record)** của Teams ngay từ đầu buổi, không chỉ bật transcript — vì đây là **căn cứ đánh giá duy nhất** thay cho việc trainer quan sát trực tiếp.

### 2.2. Cấu hình quyền truy cập Recording & Transcript

- Bật cả **Recording (video)** và **Transcription** ngay khi bắt đầu, đảm bảo không quên bật giữa chừng.
- Đảm bảo **Trainer và Admin có quyền truy cập xem lại video + transcript** sau khi buổi họp kết thúc.
- Kiểm tra lại cài đặt quyền chia sẻ (meeting options) trước giờ họp — **nếu lỗi và video không xem được, buổi demo coi như không có giá trị và phải làm lại**.
- Sau khi kết thúc, học viên nên tự kiểm tra lại recording đã lưu thành công và phát lại được trước khi báo cáo đã hoàn thành.

### 2.3. Checklist kỹ thuật trước buổi demo

- [ ] Đã test đường truyền mạng, micro, chia sẻ màn hình
- [ ] Đã chuẩn bị sẵn môi trường lab AWS (EC2 Linux instance đã có sẵn Tomcat/PostgreSQL/Prometheus-Grafana) để demo thực hành trực tiếp — **mở sẵn terminal, không setup tại chỗ**
- [ ] Đã mở sẵn file `history.log` (hoặc tương đương) của toàn bộ quá trình tự học
- [ ] Đã rà soát lại toàn bộ bằng chứng (log, screenshot, ticket, evidence pack) của các Unit đã học
- [ ] Đã xác nhận meeting bật **cả Recording lẫn Transcript**, và đã cấp quyền truy cập cho Trainer/Admin
- [ ] Đã chuẩn bị sẵn tâm lý **tự dẫn dắt toàn bộ buổi trình bày một mình**, không có người đặt câu hỏi gợi ý hay nhắc giữa chừng

---

## 3. Ba kỹ năng nền tảng BẮT BUỘC phải thể hiện trong demo

Đây là phần trainer nhấn mạnh xuyên suốt khóa học qua các buổi mentor hàng ngày của lớp L1.5 — **không được bỏ qua**, phải xen kẽ thể hiện trong cả Phần 1 (lý thuyết) lẫn Phần 2 (thực hành).

### 3.1. OSI Mindset — Tư duy khoanh vùng lỗi theo 7 tầng

Trainer đã hướng dẫn và yêu cầu áp dụng **OSI Mindset** khi debug bất kỳ sự cố nào, thay vì đoán mò hoặc restart bừa bãi. Học viên phải trình bày và demo lại được:

- **Nguyên tắc cốt lõi:** Khi hệ thống lỗi, **không được sửa ngay**. Phải: Stay Calm → Chọn hướng kiểm tra (Bottom-up hoặc Top-down) → Kiểm tra từng tầng theo thứ tự → Dừng lại ở tầng bị lỗi (không kiểm tra tiếp các tầng khác) → Ghi lại (Write It Down) triệu chứng và tầng lỗi.
- **Học viên phải điền được bảng 7 tầng sau (không nhìn tài liệu):**

| Layer | Tên tầng     | Câu hỏi kiểm tra                                                | Lệnh/công cụ kiểm tra                        |
| ----- | ------------ | --------------------------------------------------------------- | -------------------------------------------- |
| 1     | Physical     | Máy có nguồn không? Instance EC2 có đang Running không?         | AWS Console, kiểm tra Instance State         |
| 2     | Data Link    | Server có đúng địa chỉ IP/MAC không?                            | `ip addr`, `arp -a`                          |
| 3     | Network      | Mạng nội bộ/VPC/Security Group có cho phép kết nối không?       | `ping`, `traceroute`                         |
| 4     | Transport    | Port đúng có mở không? Security Group/NACL có chặn không?       | `ss -tulpn`, `nc -zv`, `telnet`              |
| 5     | Session      | Kết nối (SSH, DB session) có còn hoạt động không?               | kiểm tra session SSH/PostgreSQL đang mở      |
| 6     | Presentation | SSL/TLS certificate có hợp lệ không?                            | `openssl s_client`                           |
| 7     | Application  | Log ứng dụng (Tomcat/PostgreSQL) nói gì? App có phản hồi không? | `curl -v`, đọc `catalina.out`/PostgreSQL log |

- **Bài tập demo bắt buộc:** Chọn lại **1 tình huống lỗi thực tế đã gặp trong quá trình học** (ví dụ Tomcat không start, DB connection refused, EC2 mất SSH, alert Prometheus không nhận được...) và trình bày theo đúng trình tự: xác định hướng kiểm tra (bottom-up hay top-down, giải thích vì sao chọn hướng đó) → chỉ ra chính xác tầng nào bị lỗi và bằng chứng (lệnh + output) → giải thích vì sao các tầng khác bình thường.
- **Câu hỏi retake nếu không trả lời được:** "Khi nào nên dùng bottom-up, khi nào nên dùng top-down?" (Gợi ý: bottom-up khi nghi ngờ lỗi hạ tầng/mạng AWS; top-down khi ứng dụng Java/Tomcat báo lỗi rõ ràng).

### 3.2. Lưu lịch sử thao tác bằng History Log

Trainer đã trực tiếp hướng dẫn và kiểm tra từng học viên về việc **ghi lại toàn bộ lệnh đã thực hành**, không chỉ chạy lệnh cho xong — đặc biệt quan trọng với lớp L1.5 vì khối lượng lệnh Linux/AWS/cấu hình rất lớn:

- Dùng lệnh `history` để xem lại toàn bộ lệnh đã chạy trong phiên làm việc.
- Redirect toàn bộ lịch sử vào file: `history > history.log`.
- Mở file bằng `nano`/`vi` để rà soát và **phải giải thích được từng dòng lệnh đã chạy, dòng đó dùng để làm gì** — không được chỉ nói "em cũng không nhớ nữa".
- Đặc biệt lưu ý với các lệnh **chỉnh sửa file cấu hình** (ví dụ `/etc/hostname`, `nginx.conf`, `postgresql.conf`) — phải giải thích được nguyên tắc: **backup trước khi sửa → kiểm tra cú pháp → áp dụng thay đổi → reload/restart service → validate lại**.
- **Bài tập demo bắt buộc:** Mở file `history.log` của ít nhất 1 buổi tự học gần đây (ưu tiên buổi có thao tác cấu hình Linux/AWS/Tomcat), chọn ngẫu nhiên 3-5 dòng lệnh và giải thích tại chỗ: lệnh làm gì, tham số nào quan trọng, kết quả mong đợi là gì.
- **Câu hỏi retake nếu không trả lời được:** Nếu học viên không giải thích được một lệnh trong chính lịch sử của mình, phải retake lại bài thực hành chứa lệnh đó.

### 3.3. Dùng AI đúng cách để học và làm việc

Trainer yêu cầu học viên dùng AI (Copilot/Gemini...) như công cụ **tổng hợp và kiểm tra lại kiến thức**, không dùng để thay thế việc hiểu bài:

- Cách làm được hướng dẫn: tạo một topic/chat mới, yêu cầu AI **"tổng hợp lại toàn bộ các câu lệnh đã học"** kèm **"mẹo để nhớ các lệnh này"**, sau đó học viên tự đọc lại từng phần để kiểm tra bản thân còn nhớ gì, chưa nhớ gì.
- Nguyên tắc bắt buộc: nếu dùng AI để tra lỗi cấu hình (ví dụ lỗi `systemd-hostnamed`, lỗi kết nối PostgreSQL), **phải tự kiểm chứng lại kết quả AI đưa ra** (chạy thử, đối chiếu log thực tế) trước khi áp dụng, không được copy-paste lệnh AI đưa mà không hiểu tác động — nhất là các lệnh có `sudo` hoặc chỉnh sửa file hệ thống.
- **Bài tập demo bắt buộc:** Trình chiếu 1 đoạn hội thoại AI đã dùng để ôn tập hoặc debug trong quá trình học (ví dụ so sánh 2 file Linux, giải thích `mkdir -p`, nguyên tắc sửa file cấu hình Ubuntu...), và giải thích: AI gợi ý gì, bản thân đã kiểm chứng lại như thế nào, có điểm nào AI sai mà học viên tự phát hiện ra không.
- **Câu hỏi retake nếu không trả lời được:** "Em có tự kiểm tra lại thông tin AI đưa ra không, hay copy nguyên lệnh chạy luôn?" — nếu câu trả lời là "chạy luôn không kiểm tra", phải bổ sung thêm 1 ví dụ cụ thể đã tự kiểm chứng trước ngày demo.

---

## 4. Cấu trúc buổi trình bày (tổng thời lượng: 165 phút)

| Phần   | Nội dung                                  | Thời lượng |
| ------ | ----------------------------------------- | ---------- |
| Phần 1 | Trình bày & tổng hợp **lý thuyết**        | 45 phút    |
| Phần 2 | Trình bày **thực hành / thao tác** đã làm | 120 phút   |

---

## 5. Phần 1 — Trình bày lý thuyết (45 phút)

**Quy tắc bắt buộc:** Với mỗi Unit, học viên phải điền được các câu hỏi cụ thể bên dưới bằng số liệu/khái niệm chính xác (không được trả lời mơ hồ). Nếu không trả lời được ≥2 câu hỏi trong 1 Unit, Unit đó phải được **ôn lại và trình bày lại (retake)** trước khi chuyển sang demo thực hành.

### 5.1. Unit 1 – Orientation & Administration Foundations (3 phút) — LO6

- [ ] Nêu được phạm vi trách nhiệm (scope) và ranh giới truy cập (access boundaries) của một L1.5 Administrator — **việc gì được làm, việc gì phải xin duyệt trước**.
- [ ] Giải thích được vai trò của runbook và evidence trong công việc Administration — vì sao mọi thao tác phải có bằng chứng.
- [ ] Trả lời: "Lab readiness checklist gồm những gì trước khi bắt đầu một ngày thực hành?"

### 5.2. Unit 2 – Linux Fundamentals (12 phút) — LO1, LO8

- [ ] Giải thích được cấu trúc quyền file Linux (owner/group/others, rwx) và cách dùng `sudo` an toàn theo nguyên tắc least-privilege.
- [ ] Liệt kê được quy trình tạo user/group, cấu hình SSH key — nêu rõ vì sao không nên dùng password login trên production.
- [ ] Trình bày quy trình quản lý **systemd service**: start/stop/restart/status, cách khôi phục 1 service bị fail.
- [ ] Giải thích được cách kiểm tra mạng cơ bản: `ip`, `ss`, `ping`, `traceroute`, `curl` — và cách chẩn đoán sự cố kết nối/storage bằng các lệnh này.
- [ ] Trình bày được cách dùng `journalctl`, `grep`/`awk`/`sed` để đọc log và viết 1 Bash script backup bằng `tar`/`rsync` có lập lịch `cron`.
- [ ] Trả lời: "Nếu một exit code của script khác 0, điều đó có ý nghĩa gì và em xử lý thế nào?"

### 5.3. Unit 3 – AWS Cloud Fundamentals (12 phút) — LO2, LO6, LO9

- [ ] Giải thích mô hình **Shared Responsibility** giữa AWS và người dùng.
- [ ] Trình bày cách cấu hình IAM user/role/policy, MFA, CLI profile, và tagging tài nguyên — vì sao tagging quan trọng.
- [ ] Giải thích cấu trúc VPC: public/private subnet, route table, Internet Gateway, Security Group vs NACL — sự khác nhau cơ bản giữa 2 loại này.
- [ ] Trình bày vòng đời EC2 (launch → bootstrap bằng user data → SSH access) và vai trò của key pair.
- [ ] Giải thích quy trình EBS: mount volume, tạo snapshot, restore — và cách xác minh backup thành công.
- [ ] Trả lời: "CloudWatch alarm và budget control dùng để làm gì? Nếu quên tắt EC2/IP công khai sau giờ học thì chi phí phát sinh thế nào?"

### 5.4. Unit 4 – Java Application Configuration (7 phút) — LO3, LO6

- [ ] Giải thích kiến trúc runtime của Java app trên Tomcat: JAR/WAR, cấu trúc thư mục Tomcat, tích hợp với systemd.
- [ ] Trình bày cách externalize configuration (environment variables, profiles, port, context path, JVM memory options) — vì sao không hard-code config trong code.
- [ ] Giải thích quy trình kết nối DB an toàn: xử lý secrets, TLS certificate cơ bản.
- [ ] Trả lời: "Trước khi restart Tomcat trên production, cần kiểm tra và chuẩn bị gì? Nếu deploy lỗi thì rollback như thế nào?"

### 5.5. Unit 5 – Database & Monitoring Foundations (7 phút) — LO4, LO5, LO8

- [ ] Trình bày cách tạo user/role PostgreSQL theo nguyên tắc least-privilege, cấu hình `pg_hba.conf` cơ bản.
- [ ] Giải thích quy trình backup/restore PostgreSQL và cách xác minh health-check thành công.
- [ ] Giải thích được cách chẩn đoán session/lock, dùng `EXPLAIN` cơ bản để phát hiện slow query.
- [ ] Trình bày cách dựng Prometheus/node_exporter + Grafana dashboard, cấu hình 1 alert rule cơ bản.
- [ ] Trả lời: "Khi nào một vấn đề database/query nằm trong phạm vi L1.5 xử lý được, khi nào phải escalate lên L2?"

### 5.6. Unit 6 – Operations Controls & Integrated Practice (4 phút) — LO5, LO6, LO7, LO8, LO9

- [ ] Trình bày quy trình batch scheduling, kiểm tra dependency/exit-code, và flow ticket release/change (approval gate → evidence → rollback → handover).
- [ ] Giải thích được cách xây dựng 1 hệ thống tích hợp Linux + AWS + Java + PostgreSQL + Monitoring từ build sheet, và cách lập checklist security/cost.
- [ ] Trả lời: "Khi gặp lỗi (service/port/permission/disk/DB connection/alert/rollback), em áp dụng OSI Mindset như thế nào để xác định phạm vi xử lý?"

**Phân bổ thời gian 45 phút:** Unit 1 (3') → Unit 2 (12') → Unit 3 (12') → Unit 4 (7') → Unit 5 (7') → Unit 6 (4').

---

## 6. Phần 2 — Trình bày thực hành (120 phút)

**Quy tắc bắt buộc:** Mọi thao tác phải được **chạy trực tiếp trên môi trường lab AWS** (đã bao gồm đầy đủ EC2, Tomcat, PostgreSQL, Prometheus, ... mở sẵn terminal trước khi demo), đồng thời xen kẽ thể hiện 2 kỹ năng nền tảng: **mở `history.log` để đối chiếu lệnh đã chạy trước đó**, và áp dụng **OSI Mindset** khi gặp lỗi phát sinh ngay tại buổi demo (không được bấm Enter ngẫu nhiên nếu lệnh lỗi).

### 6.1. Unit 1 – Chuẩn bị môi trường & workstation (5 phút) — LO6

1. Demo kiểm tra access checklist, evidence folder đã được tổ chức như thế nào.
2. **Mở `history.log` tổng của khóa học, chỉ ra cách học viên đã tổ chức/lưu trữ lịch sử lệnh theo ngày/unit.**

### 6.2. Unit 2 – Linux Fundamentals (25 phút) — LO1, LO8

1. Demo CLI navigation, thao tác file/directory, text processing cơ bản (`grep`/`awk`/`sed`).
2. Demo tạo user/group có kiểm soát, áp dụng permissions, cấu hình SSH key access.
3. Demo cài đặt 1 package, tạo/quản lý 1 systemd service, và **khôi phục 1 service bị fail giả lập** — áp dụng OSI Mindset để chẩn đoán trước khi sửa.
4. Demo chẩn đoán 1 sự cố kết nối/storage giả lập bằng `ip`, `ss`, `ping`, `curl`, `df -h`.
5. Demo script health-check + backup bằng `tar`/`rsync`, lập lịch `cron`, và trình bày cách phát hiện lỗi qua exit code/`journalctl`.
6. Trình chiếu bằng chứng (evidence) đã nộp cho từng ngày học (Day 2-6).
   _Công cụ: Ubuntu Server trên EC2; Bash; cron; rsync._

### 6.3. Unit 3 – AWS Cloud Fundamentals (25 phút) — LO2, LO6, LO9

1. Demo cấu hình IAM access, MFA, CLI profile, và tagging tài nguyên đã làm.
2. Demo kiến trúc VPC đã dựng: subnet, route table, Internet Gateway, Security Group — xác minh kết nối cho phép/bị chặn.
3. Demo vòng đời EC2: bootstrap bằng user data, kết nối SSH an toàn.
4. Demo thao tác EBS: mount volume, tạo snapshot, restore dữ liệu — trình chiếu bằng chứng phục hồi.
5. Demo cấu hình CloudWatch alarm + budget control, và **thao tác tắt resource không dùng để kiểm soát chi phí** (nhấn mạnh nguyên tắc cost-control đã học).
6. Trình chiếu checklist vận hành AWS đã hoàn thành (Day 7-11).
   _Công cụ: AWS IAM/VPC/EC2/EBS/CloudWatch/Budgets._

### 6.4. Unit 4 – Java Application Configuration (20 phút) — LO3, LO6

1. Demo deploy ứng dụng Java lên Tomcat, quản lý như 1 Linux service (systemd).
2. Demo tạo environment-specific configuration, chỉnh JVM memory options đã được duyệt, restart an toàn, validate log/endpoint.
3. Demo kết nối ứng dụng tới PostgreSQL, áp dụng cấu hình TLS cơ bản.
4. **Giả lập 1 lỗi deploy và demo quy trình rollback** — áp dụng OSI Mindset layer 4-7 để xác định đúng tầng lỗi (network/port/application) trước khi rollback.
5. Trình chiếu bằng chứng log đã nộp (Day 12-14).
   _Công cụ: Java 17; Apache Tomcat; OpenSSL._

### 6.5. Unit 5 – Database & Monitoring Foundations (20 phút) — LO4, LO5, LO8

1. Demo cấu hình PostgreSQL: tạo user/role least-privilege, kết nối ứng dụng, backup/restore.
2. Demo chẩn đoán 1 tình huống connection/slow-query giả lập: thu thập bằng chứng (session, lock, `EXPLAIN`) và **quyết định xử lý trong phạm vi L1.5 hay escalate**, giải thích rõ lý do.
3. Demo dựng dashboard Grafana cơ bản, cấu hình 1 alert, giả lập lỗi để kiểm tra notification có hoạt động không.
4. Trình chiếu bằng chứng (Day 15-17).
   _Công cụ: PostgreSQL; Prometheus; Grafana; node_exporter._

### 6.6. Unit 6 – Operations Controls & Integrated Practice (20 phút) — LO5, LO6, LO7, LO8, LO9

1. Demo thực hiện 1 batch/release scenario đã duyệt: kiểm tra dependency/exit-code, tạo ticket, lưu evidence, thực hiện rollback nếu cần, viết shift handover note.
2. Demo nhanh hệ thống tích hợp Linux + AWS + Java + PostgreSQL + Monitoring đã xây dựng từ build sheet — trình bày cấu hình baseline và checklist security/cost.
3. Chọn demo lại **1 kịch bản failure injection** đã thực hành (service/port/permission/disk/DB connection/alert/rollback) — áp dụng đầy đủ OSI Mindset để khoanh vùng, khắc phục trong phạm vi L1.5, và chỉ ra điểm cần escalate nếu vượt phạm vi.
4. Trình chiếu evidence package cuối cùng và bản acceptance checklist đã hoàn thành (Day 18-21).
   _Công cụ: Internal change/release runbook; Integrated AWS lab; Git; Docker Compose._

### 6.7. Tự vấn đáp (Self Q&A) (5 phút)

Vì trainer/admin **không có mặt trực tiếp** để đặt câu hỏi gợi ý, học viên phải tự thực hiện phần này trước camera để chứng minh hiểu bài thực sự (không chỉ học thuộc kịch bản demo):

1. Tự chọn ngẫu nhiên **2-3 dòng lệnh bất kỳ** trong `history.log` của mình (chọn ngay tại chỗ, không chuẩn bị trước) và tự giải thích.
2. Tự đặt câu hỏi "Nếu bước vừa làm bị lỗi thì sao?" cho một thao tác bất kỳ đã demo, và tự trả lời bằng cách áp dụng OSI Mindset.
3. Tự tóm tắt: "Nếu phải làm lại từ đầu, phần nào em tự tin nhất, phần nào còn chưa chắc chắn?" — nêu rõ, cụ thể, không nói chung chung.

**Phân bổ thời gian 120 phút:** Unit 1 (5') → Unit 2 (25') → Unit 3 (25') → Unit 4 (20') → Unit 5 (20') → Unit 6 (20') → Self Q&A (5').

---

## 7. Tiêu chí retake — khi nào phải làm lại

Vì trainer chỉ đánh giá qua **video recording** (không có mặt để hỏi lại ngay), học viên **bắt buộc phải ôn lại và quay lại toàn bộ buổi demo** (retake) nếu rơi vào một trong các trường hợp sau:

- Không giải thích được ≥2 câu hỏi trong 1 Unit ở Phần lý thuyết (mục 5).
- Không giải thích được ý nghĩa của bất kỳ dòng lệnh nào trong chính `history.log` của mình khi tự vấn đáp (mục 6.7).
- Khi gặp lỗi phát sinh trực tiếp trong lúc demo, xử lý theo kiểu "thử lại/restart ngẫu nhiên" thay vì áp dụng OSI Mindset để khoanh vùng.
- Thiếu bất kỳ thành phần nào trong bộ bằng chứng BEFORE – ACTION – AFTER – RESULT (đặc biệt với thao tác chỉnh sửa file cấu hình hoặc deploy ứng dụng).
- Chỉ trình chiếu ảnh chụp màn hình cũ mà không demo thao tác trực tiếp khi được yêu cầu.
- **Recording bị lỗi, bị ngắt giữa chừng, âm thanh/hình ảnh không rõ, hoặc thiếu quyền truy cập** — vì đây là căn cứ duy nhất để trainer đánh giá.
- Video không thể hiện rõ màn hình thao tác (ví dụ không chia sẻ màn hình khi demo lệnh AWS Console hoặc terminal).

---

## 8. Lưu ý khi trình bày

- Trình bày đi kèm **thao tác thực tế trên môi trường lab AWS**, tránh chỉ đọc slide.
- Luôn giữ `history.log` mở sẵn để đối chiếu khi tự vấn đáp.
- **Vì không có ai hỏi lại trực tiếp**, học viên phải tự nêu rõ khó khăn/điểm chưa chắc chắn ngay trong lúc trình bày (nói thành lời trước camera) để trainer biết khi xem lại — không im lặng bỏ qua phần không chắc.
- Nói chậm, rõ ràng, luôn hướng về màn hình chia sẻ khi thao tác, vì trainer chỉ xem lại qua video không có cơ hội hỏi thêm.
- Đảm bảo **cả recording lẫn transcript** của buổi họp được lưu lại đầy đủ, có thể phát lại được, và đã cấp quyền truy cập cho Trainer/Admin — đây là **bằng chứng duy nhất** làm căn cứ đánh giá.
- Lưu ý đặc biệt với chi phí AWS: **tắt/dừng các resource (EC2, IP công khai) ngay sau khi demo xong** để tránh phát sinh chi phí ngoài dự kiến.

---
