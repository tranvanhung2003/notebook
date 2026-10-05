Đúng theo guide, **Self Q&A chỉ có 5 phút** nhưng rất quan trọng: bạn phải tự chọn **2–3 lệnh ngẫu nhiên trong `history.log`**, tự hỏi một tình huống “nếu bước vừa làm bị lỗi thì sao?” và cuối cùng tự đánh giá **điểm mạnh / điểm còn chưa chắc chắn thật cụ thể**. :chatgpt-content-reference{index="0"} Nếu không giải thích được chính lệnh trong history của mình hoặc xử lý sự cố kiểu restart/thử ngẫu nhiên thì đây là tiêu chí có thể dẫn tới retake. :chatgpt-content-reference{index="1"}

Lưu ý quan trọng: các lệnh tôi chọn bên dưới là **mẫu để luyện cách giải thích**. Khi quay chính thức, guide yêu cầu bạn chọn ngay tại chỗ, không chuẩn bị trước. `history.log` nên mở sẵn trong recording. :chatgpt-content-reference{index="2"}

## Chuẩn bị màn hình trước Self Q&A

Bạn vẫn đang SSH vào EC2. Mở:

```bash
cd ~/java-fresher-demo/history
```

Kiểm tra:

```bash
tail -30 history.log
```

Hoặc mở để cuộn:

```bash
less history.log
```

Nếu muốn chứng minh thực sự chọn ngẫu nhiên, có thể dùng:

```bash
shuf -n 3 history.log
```

Nhưng chỉ dùng cách này nếu bạn thực sự có khả năng giải thích **bất kỳ lệnh nào** trong history. Đừng dùng nếu history chứa những dòng multiline khó đọc hoặc command không còn đủ context.

Dưới đây là kịch bản hoàn chỉnh để bạn luyện nói.

## 1. Tự chọn 2–3 command trong `history.log`

Bây giờ em sẽ mở `history.log` và chọn ngẫu nhiên một số command em đã thực hiện trong quá trình lab. Em sẽ giải thích command dùng để làm gì, các option quan trọng, expected result và em dùng kết quả đó để kết luận điều gì.

### Command thứ nhất — ví dụ `ss`

Giả sử em chọn được:

```bash
sudo ss -lntp | grep ':8080'
```

Command này em dùng để kiểm tra TCP socket đang ở trạng thái listening trên port 8080.

Trong đó:

- `ss` là công cụ inspect socket trên Linux.
- `-l` là chỉ hiển thị listening sockets.
- `-n` là hiển thị port và IP dưới dạng numeric, không resolve service name.
- `-t` là TCP.
- `-p` là hiển thị process liên quan đến socket. Vì thông tin process có thể cần quyền cao hơn nên em dùng `sudo`.
- Sau đó output được pipe qua `grep ':8080'` để chỉ lấy dòng liên quan đến port Tomcat.

Expected result của em là có một listener trên TCP/8080 khi Tomcat đang hoạt động.

Tuy nhiên command này chỉ chứng minh server đang có process listen ở port 8080. Nó chưa chứng minh client từ Internet có thể truy cập được, vì external connectivity còn phụ thuộc AWS Security Group, route, public IP và các network control khác.

Đây cũng là lý do trong Unit 3 khi em remove Security Group rule của port 8080, `ss` trên EC2 vẫn cho thấy 8080 đang listen nhưng browser bên ngoài không truy cập được.

---

### Command thứ hai — ví dụ `journalctl`

Giả sử command được chọn là:

```bash
sudo journalctl -u tomcat10 --since '-5 minutes' --no-pager
```

Em dùng command này để đọc log của `tomcat10.service` từ systemd journal.

Trong đó:

- `journalctl` là công cụ query systemd journal.
- `-u tomcat10` giới hạn log theo systemd unit `tomcat10`.
- `--since '-5 minutes'` chỉ xem log gần thời điểm sự cố để giảm noise.
- `--no-pager` in trực tiếp output thay vì mở pager.

Em thường dùng command này sau khi có symptom như Tomcat restart fail hoặc WAR deploy không thành công.

Ví dụ trong Unit 4, nếu port 8080 vẫn listen và Tomcat service vẫn active nhưng `/fresher-app/health` không hoạt động sau deployment, em đọc log Tomcat để tìm evidence về lỗi WAR/deployment thay vì restart EC2 hoặc thay đổi Security Group ngẫu nhiên.

Điểm quan trọng là em dùng log để xác định root cause, không chỉ dùng log để nói rằng "có lỗi".

---

### Command thứ ba — ví dụ PostgreSQL backup

Giả sử em chọn được:

```bash
sudo -u postgres pg_dump --format=custom --dbname=fresherdb --file="$PG_BACKUP"
```

Đây là command em dùng để tạo logical backup của PostgreSQL database `fresherdb`.

Trong đó:

- `sudo -u postgres` chạy command dưới OS account `postgres`, thay vì chạy PostgreSQL administrative operation bằng user `ubuntu`.
- `pg_dump` là công cụ logical backup.
- `--dbname=fresherdb` chọn database cần backup.
- `--format=custom` tạo custom archive format.
- `--file="$PG_BACKUP"` ghi archive vào path em đã xác định trước.

Sau command em phải kiểm tra exit code ngay. Exit code `0` nghĩa là command hoàn thành thành công ở cấp process, còn khác `0` nghĩa là backup command thất bại.

Tuy nhiên việc có một file `.dump` chưa đủ để kết luận backup usable. Vì vậy trong Unit 5 em còn dùng `pg_restore --list` để kiểm tra archive và quan trọng hơn là restore vào database riêng `fresherdb_restore`, sau đó query lại `app.demo_message` để chứng minh data thực sự recover được.

Như vậy validation của em không dừng ở "backup file exists", mà đi tới "restore và đọc lại được dữ liệu".

---

## 2. Tự hỏi: “Nếu bước vừa làm bị lỗi thì sao?”

Em chọn tình huống integrated ở Unit 6: application không kết nối được PostgreSQL sau một configuration change.

Giả sử `/fresher-app/db-check` trả:

```text
HTTP 503
db=DOWN
```

Em sẽ không restart Tomcat hay PostgreSQL ngay.

Em bắt đầu bằng symptom và recent change, sau đó chọn hướng **top-down**, vì lỗi được phát hiện ở application dependency.

Trước tiên em kiểm tra:

```bash
curl -i http://127.0.0.1:8080/fresher-app/health
```

Nếu `/health` vẫn trả HTTP 200 và `status=UP` thì Java application và Tomcat cơ bản vẫn hoạt động.

Tiếp theo:

```bash
systemctl is-active tomcat10
```

và:

```bash
sudo ss -lntp | grep ':8080'
```

Nếu Tomcat active và 8080 vẫn listening thì em có thêm bằng chứng application runtime không phải root cause chính.

Sau đó em kiểm tra database dependency:

```bash
systemctl is-active postgresql
```

```bash
pg_isready -h 127.0.0.1 -p 5432
```

và:

```bash
sudo ss -lntp | grep ':5432'
```

Nếu PostgreSQL active, `pg_isready` thành công và TCP/5432 có listener, thì database service thực tế vẫn hoạt động.

Em tiếp tục kiểm tra recent application configuration:

```bash
sudo grep '^DB_URL=' /etc/java-fresher/fresher-app.env
```

Nếu em thấy application đang trỏ tới:

```text
127.0.0.1:5433
```

trong khi PostgreSQL thật listen:

```text
127.0.0.1:5432
```

thì em kiểm chứng thêm:

```bash
nc -zv 127.0.0.1 5432
```

thành công, còn:

```bash
nc -zv 127.0.0.1 5433
```

thất bại.

Tới đây em có thể kết luận root cause là **externalized application configuration sai database port**, không phải AWS Security Group, Tomcat hay PostgreSQL service.

Vì đây là một recent change đã biết và em có last-known-good configuration, nó nằm trong phạm vi L1.5 lab remediation của em.

Em rollback đúng phần bị thay đổi, không rollback cả hệ thống:

```bash
sudo cp -a \
  "$CHANGE_DIR/secure/fresher-app.env.1.1.0-good" \
  /etc/java-fresher/fresher-app.env
```

sau đó restart Tomcat để load environment mới:

```bash
sudo systemctl restart tomcat10
```

Rồi em validate lại:

```bash
curl -fsS http://127.0.0.1:8080/fresher-app/health
curl -fsS http://127.0.0.1:8080/fresher-app/db-check
curl -fsS http://127.0.0.1:8080/fresher-app/metrics
```

Expected invariant cuối cùng là:

```text
application = UP
database = UP
TLS = true
fresher_app_up = 1
fresher_app_db_up = 1
```

Sau đó em kiểm tra Prometheus/Grafana để xác nhận monitoring cũng recover và alert trở lại trạng thái Normal/Resolved.

Nếu evidence thay vì chỉ ra configuration đơn giản lại cho thấy database corruption, PostgreSQL crash lặp lại, data inconsistency, hoặc cần restore production database, thì em sẽ không tự xử lý vượt phạm vi. Em sẽ preserve logs/evidence và escalate cho DBA/L2 hoặc application owner theo runbook.

---

## 3. Nếu phải làm lại từ đầu, phần nào em tự tin nhất?

Nếu phải xây dựng lại hệ thống từ đầu, phần em tự tin nhất là **AWS infrastructure configuration và quản lý các resource có nhiều dependency với nhau**.

Ví dụ, em khá tự tin trong việc thiết kế và giải thích luồng từ:

```text
VPC
→ subnet
→ route table
→ Internet Gateway
→ Security Group
→ EC2
→ EBS
→ CloudWatch
```

Em hiểu một public subnet không trở thành public chỉ vì tên của subnet, mà phải có route phù hợp tới Internet Gateway và instance còn cần public connectivity phù hợp.

Em cũng tự tin hơn trong việc phân biệt trách nhiệm của Security Group, route table, public IP và service listener khi troubleshoot connectivity.

Ví dụ nếu một endpoint bên ngoài không truy cập được, em không mặc định application hỏng. Em có thể kiểm tra theo từng layer: trạng thái EC2 trên AWS Console, VPC/subnet/route, Security Group, rồi tới socket bằng `ss` và cuối cùng application bằng `curl`.

Phần em cũng khá tự tin là **quản lý lifecycle và dependency của resource**. Em hiểu khi nào nên Stop EC2 thay vì Terminate, EBS vẫn tồn tại và vẫn có storage cost sau khi EC2 stopped, snapshot có retention/cost riêng, và một resource không nên bị xóa nếu vẫn có component khác phụ thuộc vào nó.

Qua các Unit em cảm thấy mình có khả năng nhìn một hệ thống AWS tương đối phức tạp theo kiến trúc tổng thể thay vì coi từng resource là một thành phần độc lập.

---

## 4. Phần nào em còn chưa chắc chắn?

Phần em còn chưa chắc chắn hơn là **Java/Tomcat application deployment và configuration**, đặc biệt khi hệ thống được đóng gói theo container hoặc Docker Compose.

Trong lab hiện tại em đã thực hành Java 17, WAR deployment và Tomcat được quản lý bằng systemd, nên em đã hiểu được flow cơ bản:

```text
source code
→ Maven
→ WAR
→ Tomcat
→ externalized configuration
→ JDBC
→ PostgreSQL
```

Tuy nhiên em thấy Java/Tomcat có nhiều lớp cấu hình và nhiều directory có vai trò khác nhau, ví dụ source tree, Maven `target`, release repository, Tomcat `webapps`, `/etc/tomcat10`, application environment file và systemd drop-in.

Nếu quản lý không chặt, rất dễ nhầm giữa:

```text
source version
artifact version
deployed version
configuration version
runtime version
```

Đây là phần em muốn tiếp tục luyện thêm.

Ví dụ em muốn chắc chắn hơn trong việc nhìn một Tomcat environment chưa quen và nhanh chóng xác định đúng:

```text
CATALINA_HOME
CATALINA_BASE
webapps
configuration location
log location
service account
JVM options
environment configuration
deployed WAR
```

thay vì học thuộc path của một máy cụ thể.

Em cũng muốn luyện thêm **version management và rollback discipline**. Em hiểu về nguyên tắc phải có known-good artifact, checksum, version tag và backup configuration trước release; tuy nhiên khi có nhiều version Java/Tomcat/application hoặc nhiều container image/tag cùng tồn tại thì em muốn thực hành thêm để tránh deploy nhầm version hoặc rollback không đồng bộ giữa artifact và configuration.

Đặc biệt với **containerized Java/Tomcat**, đây là phần em chưa tự tin bằng AWS infrastructure. Lab hiện tại chủ yếu dùng native systemd/Tomcat nên em muốn thực hành thêm Docker image, image tag, volume/config/secrets, port mapping, container logs, health checks và Docker Compose dependency để hiểu rõ sự khác nhau giữa lỗi ở container layer và lỗi ở Java/Tomcat bên trong container.

Nếu phải làm lại từ đầu, em không xem đây là phần mình không làm được; đây là phần em sẽ dành nhiều thời gian kiểm tra và validation hơn, đồng thời dựa vào runbook, logs và version inventory chặt chẽ để giảm rủi ro.

---

## 5. Kết thúc Self Q&A

Tóm lại, qua bài thực hành này em cảm thấy điểm mạnh hiện tại của em là AWS infrastructure, resource management và troubleshooting theo dependency/layer.

Phần em cần tiếp tục củng cố là Java/Tomcat deployment, cấu trúc configuration/runtime, containerized deployment và quản lý version/rollback.

Điều em cố gắng duy trì xuyên suốt toàn bộ bài là không sửa lỗi theo phỏng đoán. Em ưu tiên xác định symptom, recent change, scope, thu thập evidence, tìm failing layer, thực hiện safe remediation trong phạm vi L1.5, validate lại toàn bộ dependency và escalate nếu vấn đề vượt phạm vi.

Một điểm quan trọng trong phần tự đánh giá: cách nói trên **không biến “điểm chưa chắc chắn” thành lời thú nhận mơ hồ kiểu “em yếu Java”**. Bạn đang chỉ ra rất cụ thể mình đã làm được native Java/Tomcat ở mức nào, chính xác phần nào còn khó hơn — directory/layout, container layer, version consistency, rollback — và bạn có kế hoạch kiểm soát rủi ro bằng runbook, logs, inventory và validation. Điều này phù hợp với yêu cầu của guide là phải nói rõ phần chưa chắc chắn, không im lặng bỏ qua. :chatgpt-content-reference{index="3"}

Trong recording thật, tôi khuyên phần 1 chỉ dành khoảng **2 phút**, phần OSI scenario khoảng **1.5–2 phút**, và phần tự đánh giá khoảng **1–1.5 phút**. Quan trọng nhất là 2–3 command phải thật sự được chọn từ `history.log` tại chỗ; phần giải thích nên theo công thức **command dùng để làm gì → option → expected result → kết quả đó chứng minh điều gì → nếu fail thì kiểm tra tiếp đâu**, thay vì chỉ dịch tên option.
