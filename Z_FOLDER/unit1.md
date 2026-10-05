Được. Ta tiếp tục **Phần 2 – Unit 1: Chuẩn bị môi trường & workstation**, theo đúng guide và coi môi trường AWS tích hợp đã dựng ở bước trước là baseline để dùng cho toàn bộ practical.

Theo guide, Unit 1 chỉ có **5 phút**, nhưng phải chứng minh được hai việc: **access checklist/evidence folder** và **`history.log` tổng được tổ chức rõ ràng theo ngày/unit**. Trong suốt practical, `history.log` phải được dùng để đối chiếu lệnh đã chạy. :chatgpt-content-reference{index="0"} Điều này cũng khớp Day 1 của syllabus: workstation, access checklist, evidence folder và learning baseline là phần bắt buộc. :chatgpt-content-reference{index="1"}

Đặc biệt, guide yêu cầu bạn phải có khả năng mở `history.log`, chọn ngẫu nhiên lệnh và giải thích lệnh đó làm gì; với sửa config phải giải thích được flow **backup → syntax/config validation → apply → reload/restart → validate**. :chatgpt-content-reference{index="2"}

---

# 1. Trạng thái đầu vào của Unit 1

Ta coi lab hiện tại có kiến trúc:

```text
Windows
└── WSL Ubuntu
    ├── AWS CLI
    ├── SSH client
    ├── SSH private key
    └── Evidence repository
          |
          | SSH TCP/22
          v
AWS
└── VPC
    ├── Public Subnet
    ├── Private Subnet
    ├── Route Table
    ├── Internet Gateway
    ├── Security Group
    └── EC2 Ubuntu
         ├── Java 17
         ├── Tomcat 10        :8080
         ├── PostgreSQL       :5432
         ├── Prometheus       :9090
         ├── node_exporter    :9100
         └── Grafana          :3000
```

Unit 1 **không phải lúc để setup lại toàn bộ những thứ này**.

Trong 5 phút, mục tiêu là chứng minh:

```text
Environment exists
        ↓
I have authorized access
        ↓
I know exactly which environment I am operating on
        ↓
Evidence is organized
        ↓
Command history is retained with timestamps
        ↓
Ready for Unit 2
```

---

# 2. Tạo evidence structure chính thức

Tôi khuyên lưu evidence gốc trên **WSL**, bởi nếu sau này EC2 bị terminate, evidence của bạn không biến mất theo instance.

Trong WSL:

```bash
mkdir -p ~/java-fresher-demo/{history,evidence/{unit01,unit02,unit03,unit04,unit05,unit06},config-backup,notes}
```

Kiểm tra:

```bash
find ~/java-fresher-demo -maxdepth 2 -type d | sort
```

Cấu trúc sẽ là:

```text
~/java-fresher-demo/
├── history/
├── evidence/
│   ├── unit01/
│   ├── unit02/
│   ├── unit03/
│   ├── unit04/
│   ├── unit05/
│   └── unit06/
├── config-backup/
└── notes/
```

Ý nghĩa:

```text
history/
```

lưu command history.

```text
evidence/
```

lưu kết quả chứng minh BEFORE/ACTION/AFTER/RESULT.

```text
config-backup/
```

lưu bản sao các configuration quan trọng nếu bạn copy chúng về workstation.

```text
notes/
```

lưu handover/troubleshooting notes.

---

# 3. Unit 1 evidence riêng

Tạo:

```bash
mkdir -p ~/java-fresher-demo/evidence/unit01/{access,aws,ssh,system}
```

Kiểm tra:

```bash
tree ~/java-fresher-demo/evidence/unit01
```

Nếu chưa có `tree` thì **không cần cài chỉ để demo**; dùng:

```bash
find ~/java-fresher-demo/evidence/unit01 -type d
```

Expected structure:

```text
unit01/
├── access/
├── aws/
├── ssh/
└── system/
```

---

# 4. Quan trọng: cấu hình Bash history có ngày giờ

Đây là phần bạn yêu cầu bổ sung.

Guide chỉ ghi:

```bash
history > history.log
```

nhưng chúng ta sẽ cải thiện để output có dạng:

```text
317  2026-10-06 02:31:45 aws sts get-caller-identity
318  2026-10-06 02:32:03 ssh java-fresher-lab
319  2026-10-06 02:33:10 systemctl status tomcat10
```

Bash hỗ trợ việc này bằng:

```bash
HISTTIMEFORMAT
```

Theo GNU Bash manual, khi `HISTTIMEFORMAT` được đặt, `history` dùng format đó để hiển thị timestamp; Bash cũng lưu timestamp vào history file để có thể giữ lại qua các shell session. :chatgpt-content-reference{index="3"}

## Thiết lập ngay trong session hiện tại

Trên WSL:

```bash
export HISTTIMEFORMAT='%Y-%m-%d %H:%M:%S '
```

Sau đó:

```bash
history | tail -20
```

Bạn sẽ thấy dạng:

```text
  421  2026-10-06 02:38:14 pwd
  422  2026-10-06 02:38:18 ls -la
  423  2026-10-06 02:38:29 aws sts get-caller-identity
```

Ở đây:

```text
%Y = year         2026
%m = month        10
%d = day          06
%H = hour         02
%M = minute       38
%S = second       29
```

Vì vậy:

```bash
'%Y-%m-%d %H:%M:%S '
```

cho:

```text
2026-10-06 02:38:29
```

---

# 5. Làm cấu hình timestamp tồn tại qua login

Nếu chỉ chạy:

```bash
export HISTTIMEFORMAT=...
```

thì setting chỉ tồn tại ở shell hiện tại.

Ta thêm vào:

```text
~/.bashrc
```

Trước khi sửa, theo đúng nguyên tắc operations:

```bash
cp -a ~/.bashrc ~/.bashrc.bak.$(date +%Y%m%d_%H%M%S)
```

Verify backup:

```bash
ls -l ~/.bashrc*
```

Sau đó thêm:

```bash
cat >> ~/.bashrc <<'EOF'

# Java Fresher Administration lab - history settings
export HISTTIMEFORMAT='%Y-%m-%d %H:%M:%S '
export HISTSIZE=100000
export HISTFILESIZE=200000
shopt -s histappend
EOF
```

Giải thích từng dòng.

```bash
export HISTTIMEFORMAT='%Y-%m-%d %H:%M:%S '
```

hiển thị và lưu timestamp cho Bash history.

```bash
export HISTSIZE=100000
```

cho phép interactive history giữ tối đa 100.000 command entries trong memory.

```bash
export HISTFILESIZE=200000
```

cho phép history file lưu nhiều entry hơn default.

```bash
shopt -s histappend
```

rất quan trọng khi bạn mở nhiều terminal.

Nếu không dùng `histappend`, một shell khi exit có thể ghi lại history theo kiểu làm mất dữ liệu do shell khác sinh ra.

`histappend` yêu cầu Bash **append** history thay vì overwrite history file khi shell kết thúc. GNU Bash mô tả chính cơ chế append/overwrite này trong history facility. :chatgpt-content-reference{index="4"}

---

# 6. Validate `.bashrc` trước khi sử dụng

Đừng chỉ edit rồi đóng terminal.

Kiểm tra syntax:

```bash
bash -n ~/.bashrc
```

Nếu command không output gì và:

```bash
echo $?
```

trả:

```text
0
```

thì Bash syntax hợp lệ.

Sau đó apply vào shell hiện tại:

```bash
source ~/.bashrc
```

Verify:

```bash
echo "$HISTTIMEFORMAT"
```

Expected:

```text
%Y-%m-%d %H:%M:%S
```

Verify:

```bash
shopt histappend
```

Expected invariant:

```text
histappend      on
```

---

# 7. Lưu history hiện tại xuống `~/.bash_history`

Bash có một history list trong RAM và một persistent history file:

```text
~/.bash_history
```

Hai thứ này không hoàn toàn giống nhau tại mọi thời điểm.

Trước khi tạo evidence, chạy:

```bash
history -a
```

`history -a` nghĩa là:

> append những command mới của current session vào `$HISTFILE`.

Xem history file đang dùng:

```bash
echo "$HISTFILE"
```

Expected:

```text
/home/<your-user>/.bash_history
```

Thông thường trên WSL:

```text
/home/username/.bash_history
```

---

# 8. Tạo `history.log` có ngày giờ

Bây giờ thực hiện đúng yêu cầu của bạn:

```bash
export HISTTIMEFORMAT='%Y-%m-%d %H:%M:%S '
history -a
history > ~/java-fresher-demo/history/history.log
```

Kiểm tra:

```bash
head -20 ~/java-fresher-demo/history/history.log
```

và:

```bash
tail -20 ~/java-fresher-demo/history/history.log
```

Expected format:

```text
  395  2026-10-06 02:18:07 mkdir -p ~/java-fresher-demo
  396  2026-10-06 02:18:15 pwd
  397  2026-10-06 02:18:20 aws sts get-caller-identity
  398  2026-10-06 02:19:02 curl -4 https://checkip.amazonaws.com
  399  2026-10-06 02:20:10 ssh java-fresher-lab
```

Đây chính là format tôi muốn bạn dùng trong demo.

---

# 9. Một lưu ý cực kỳ quan trọng về history cũ

Nếu trước đây bạn **chưa bật `HISTTIMEFORMAT`**, đừng cố sửa tay rồi tự thêm timestamp giả vào các command cũ.

Ta phải bảo toàn evidence.

Nếu một số command cũ:

```text
123 ls -la
124 sudo apt update
125 ssh ubuntu@...
```

không có timestamp đáng tin cậy, hãy giữ nguyên.

Không biến nó thành:

```text
123 2026-09-10 10:00:00 ls -la
```

nếu bạn không biết đó thực sự là lúc nào.

Từ lúc bật `HISTTIMEFORMAT` trở đi, lịch sử mới sẽ có timestamp đúng.

Đây là nguyên tắc evidence:

> **Thiếu evidence còn tốt hơn fabricated evidence.**

---

# 10. Thiết lập tương tự trên EC2

Đừng chỉ lưu history của WSL.

Phần lớn Linux/Tomcat/PostgreSQL commands sẽ chạy **trên EC2**, nên EC2 cũng cần timestamp history.

SSH vào:

```bash
ssh java-fresher-lab
```

hoặc:

```bash
ssh -i ~/.ssh/java-fresher-lab ubuntu@<EC2_PUBLIC_IP>
```

Trên EC2:

```bash
cp -a ~/.bashrc ~/.bashrc.bak.$(date +%Y%m%d_%H%M%S)
```

Thêm:

```bash
cat >> ~/.bashrc <<'EOF'

# Java Fresher Administration lab - history settings
export HISTTIMEFORMAT='%Y-%m-%d %H:%M:%S '
export HISTSIZE=100000
export HISTFILESIZE=200000
shopt -s histappend
EOF
```

Validate:

```bash
bash -n ~/.bashrc
```

Apply:

```bash
source ~/.bashrc
```

Verify:

```bash
history | tail
```

---

# 11. Tạo `history.log` trên EC2

Ta đã có:

```text
/opt/java-fresher/evidence
```

từ baseline bootstrap.

Tạo directory:

```bash
mkdir -p ~/java-fresher-demo/history
```

Sau đó:

```bash
history -a
history > ~/java-fresher-demo/history/history.log
```

Verify:

```bash
tail -20 ~/java-fresher-demo/history/history.log
```

---

# 12. Copy EC2 history về workstation

Evidence không nên chỉ nằm trên EC2.

Thoát EC2:

```bash
exit
```

Trên WSL:

```bash
mkdir -p ~/java-fresher-demo/history/ec2
```

Copy:

```bash
scp -i ~/.ssh/java-fresher-lab \
  ubuntu@<EC2_PUBLIC_IP>:~/java-fresher-demo/history/history.log \
  ~/java-fresher-demo/history/ec2/history.log
```

Nếu đã cấu hình SSH alias:

```bash
scp java-fresher-lab:~/java-fresher-demo/history/history.log \
  ~/java-fresher-demo/history/ec2/history.log
```

Bây giờ bạn có:

```text
~/java-fresher-demo/history/
├── history.log
└── ec2/
    └── history.log
```

Trong đó:

```text
history/history.log
```

là WSL history.

```text
history/ec2/history.log
```

là history chạy trên EC2.

---

# 13. Tôi khuyên tổ chức history theo Day/Unit nữa

Guide yêu cầu bạn chỉ ra cách history được tổ chức theo **ngày/unit**. :chatgpt-content-reference{index="5"}

Do đó master history nên giữ nguyên, nhưng sau từng buổi bạn tạo snapshot.

Ví dụ sau Unit 1:

```bash
cp ~/java-fresher-demo/history/history.log \
   ~/java-fresher-demo/history/2026-10-06_unit01_wsl.log
```

EC2:

```bash
cp ~/java-fresher-demo/history/ec2/history.log \
   ~/java-fresher-demo/history/2026-10-06_unit01_ec2.log
```

Về sau sẽ có:

```text
history/
├── history.log
├── 2026-10-06_unit01_wsl.log
├── 2026-10-06_unit01_ec2.log
├── 2026-10-06_unit02_linux.log
├── 2026-10-06_unit03_aws.log
├── 2026-10-06_unit04_java.log
├── 2026-10-06_unit05_db_monitoring.log
└── 2026-10-06_unit06_operations.log
```

Như vậy khi trainer xem video, bạn có thể nói:

> "`history.log` này là master history. Ngoài ra em snapshot theo ngày và Unit để có thể trace lại thao tác nào thuộc phần thực hành nào."

Đây là cách trình bày tốt hơn việc có một file chứa hàng nghìn dòng nhưng không tổ chức gì.

---

# 14. Không để secret xuất hiện trong history

Đây là phần **Supplementary / Beyond explicit syllabus**, nhưng rất quan trọng về security.

Không chạy:

```bash
export DB_PASSWORD=MySecretPassword123
```

rồi để command đó nằm trong `history.log`.

Cũng không chạy:

```bash
psql "postgresql://appuser:SecretPassword@localhost/appdb"
```

vì password có thể xuất hiện trong history.

Không paste:

```text
AWS_SECRET_ACCESS_KEY
password
token
private key
API key
```

vào command line.

Đặc biệt:

```text
~/.ssh/java-fresher-lab
```

là **private key**.

Bạn có thể demo:

```bash
ls -l ~/.ssh/java-fresher-lab
```

nhưng không:

```bash
cat ~/.ssh/java-fresher-lab
```

---

# 15. Access checklist của Unit 1

Đây là checklist tôi muốn bạn thực hiện trước camera:

| Kiểm tra         | Command/evidence                        | Expected invariant               |
| ---------------- | --------------------------------------- | -------------------------------- |
| Đúng workstation | `hostname`, `whoami`                    | Đúng WSL user                    |
| AWS identity     | `aws sts get-caller-identity`           | Đúng account/role/user được phép |
| Public IP        | `curl -4 https://checkip.amazonaws.com` | Khớp `/32` trong SG              |
| SSH key          | `ls -l ~/.ssh/java-fresher-lab`         | private key permission an toàn   |
| EC2              | AWS Console                             | Running + status checks passed   |
| SSH              | `ssh java-fresher-lab`                  | Login thành công                 |
| Correct server   | `hostnamectl`                           | `java-fresher-lab`               |
| Network          | `ip addr`, `ip route`                   | đúng subnet/default route        |
| Services         | `systemctl is-active ...`               | expected services active         |
| Socket           | `sudo ss -lntp`                         | expected ports listening         |
| Disk             | `df -h`                                 | không thiếu disk                 |
| Memory           | `free -h`                               | đủ memory                        |
| Evidence         | `find ~/java-fresher-demo ...`          | structure tồn tại                |
| History          | `tail history.log`                      | command có timestamp             |

Đây là checklist thực tế; không cần trainer phải hỏi bạn từng mục.

---

# 16. Capture access evidence trên WSL

Chạy:

```bash
date -Is > ~/java-fresher-demo/evidence/unit01/access/check-time.txt
```

```bash
whoami > ~/java-fresher-demo/evidence/unit01/access/whoami.txt
```

```bash
hostname > ~/java-fresher-demo/evidence/unit01/access/workstation-hostname.txt
```

AWS identity:

```bash
aws sts get-caller-identity \
  > ~/java-fresher-demo/evidence/unit01/aws/aws-identity.json
```

Kiểm tra:

```bash
cat ~/java-fresher-demo/evidence/unit01/aws/aws-identity.json
```

Lưu ý:

`get-caller-identity` không chứa access key secret, nên phù hợp để dùng làm evidence identity.

---

# 17. Capture SSH evidence

Không cần lưu private key.

Chỉ lưu metadata:

```bash
ls -l ~/.ssh/java-fresher-lab \
  > ~/java-fresher-demo/evidence/unit01/ssh/key-permission.txt
```

Nếu key permission là:

```text
-rw------- ...
```

tức:

```text
600
```

là baseline tốt.

Có thể verify chính xác:

```bash
stat -c '%a %U %G %n' ~/.ssh/java-fresher-lab
```

Expected:

```text
600 <user> <group> /home/.../.ssh/java-fresher-lab
```

---

# 18. SSH vào EC2 và tạo system health evidence

Trên EC2:

```bash
mkdir -p ~/java-fresher-demo/evidence/unit01
```

Timestamp:

```bash
date -Is | tee ~/java-fresher-demo/evidence/unit01/check-time.txt
```

Identity:

```bash
whoami | tee ~/java-fresher-demo/evidence/unit01/whoami.txt
```

Hostname:

```bash
hostnamectl | tee ~/java-fresher-demo/evidence/unit01/hostnamectl.txt
```

Network:

```bash
ip addr | tee ~/java-fresher-demo/evidence/unit01/ip-addr.txt
```

```bash
ip route | tee ~/java-fresher-demo/evidence/unit01/ip-route.txt
```

---

# 19. Kiểm tra service baseline

Chạy:

```bash
for svc in tomcat10 postgresql prometheus prometheus-node-exporter grafana-server; do
    printf '%-30s ' "$svc"
    systemctl is-active "$svc"
done
```

Expected invariant:

```text
tomcat10                       active
postgresql                     active
prometheus                     active
prometheus-node-exporter       active
grafana-server                 active
```

Đừng học thuộc output spacing.

Quan trọng là:

```text
service → active
```

---

# 20. Lưu service evidence

```bash
{
    echo "=== CHECK TIME ==="
    date -Is

    echo
    echo "=== SERVICES ==="

    for svc in tomcat10 postgresql prometheus prometheus-node-exporter grafana-server; do
        printf '%-30s ' "$svc"
        systemctl is-active "$svc"
    done
} | tee ~/java-fresher-demo/evidence/unit01/services.txt
```

---

# 21. Kiểm tra port

```bash
sudo ss -lntp
```

Bạn cần nhận diện:

```text
22     SSH
3000   Grafana
5432   PostgreSQL
8080   Tomcat
9090   Prometheus
9100   node_exporter
```

Lưu evidence:

```bash
sudo ss -lntp \
  | tee ~/java-fresher-demo/evidence/unit01/listening-ports.txt
```

Quan trọng:

> `ss` cho biết process/service đang listen trên server; nó **không chứng minh Security Group cho phép Internet truy cập port đó**.

Đó là hai layer khác nhau.

---

# 22. Disk và memory

Disk:

```bash
df -h \
  | tee ~/java-fresher-demo/evidence/unit01/df-h.txt
```

Memory:

```bash
free -h \
  | tee ~/java-fresher-demo/evidence/unit01/free-h.txt
```

Bạn không chỉ nói:

> "Disk OK."

Mà phải biết nhìn ít nhất:

```text
Filesystem
Size
Used
Avail
Use%
Mounted on
```

và đặc biệt `/`.

---

# 23. Failed units

```bash
systemctl --failed --no-pager \
  | tee ~/java-fresher-demo/evidence/unit01/failed-services.txt
```

Expected invariant:

> không có unexpected failed service.

Nếu có:

```text
1 loaded units listed.
```

thì **đừng bỏ qua rồi chuyển Unit 2**.

Phải investigate.

---

# 24. Tổng hợp `history.log` sau các thao tác Unit 1

Sau khi đã hoàn thành tất cả lệnh Unit 1, trên EC2:

```bash
history -a
history > ~/java-fresher-demo/history/history.log
```

Verify cuối file:

```bash
tail -30 ~/java-fresher-demo/history/history.log
```

Bạn muốn thấy chính những command vừa thực hiện cùng thời gian, ví dụ:

```text
  550  2026-10-06 02:50:04 hostnamectl
  551  2026-10-06 02:50:10 ip addr
  552  2026-10-06 02:50:16 ip route
  553  2026-10-06 02:50:29 systemctl is-active tomcat10
  554  2026-10-06 02:50:41 sudo ss -lntp
  555  2026-10-06 02:51:02 df -h
  556  2026-10-06 02:51:06 free -h
```

**Example output**, không phải exact output bạn phải có.

---

# 25. Copy evidence từ EC2 về WSL

Sau khi thoát EC2:

```bash
mkdir -p ~/java-fresher-demo/evidence/unit01/ec2
```

Copy toàn bộ:

```bash
scp -r \
  java-fresher-lab:~/java-fresher-demo/evidence/unit01/* \
  ~/java-fresher-demo/evidence/unit01/ec2/
```

Copy history:

```bash
scp \
  java-fresher-lab:~/java-fresher-demo/history/history.log \
  ~/java-fresher-demo/history/2026-10-06_unit01_ec2.log
```

Như vậy EC2 có bị stop/terminate thì evidence vẫn còn trên workstation.

---

# 26. Kiểm tra evidence cuối cùng

Trên WSL:

```bash
find ~/java-fresher-demo \
  -maxdepth 4 \
  -type f \
  | sort
```

Bạn nên thấy ít nhất dạng:

```text
java-fresher-demo/
├── evidence/
│   └── unit01/
│       ├── access/
│       │   ├── check-time.txt
│       │   ├── whoami.txt
│       │   └── workstation-hostname.txt
│       ├── aws/
│       │   └── aws-identity.json
│       ├── ssh/
│       │   └── key-permission.txt
│       └── ec2/
│           ├── check-time.txt
│           ├── hostnamectl.txt
│           ├── ip-addr.txt
│           ├── ip-route.txt
│           ├── services.txt
│           ├── listening-ports.txt
│           ├── df-h.txt
│           ├── free-h.txt
│           └── failed-services.txt
└── history/
    ├── history.log
    └── 2026-10-06_unit01_ec2.log
```

Đây là trạng thái rất tốt để đi vào Unit 2.

---

# 27. Cách trình bày Unit 1 trong đúng khoảng 5 phút

Bạn không nên đọc từng command.

Flow demo nên là:

**Phút 0:00–1:00 — Workstation/access**

Mở WSL và nói rõ:

> "Đây là workstation WSL Ubuntu của em. Trước khi thao tác em kiểm tra identity và quyền truy cập để đảm bảo đang làm đúng environment."

Chạy:

```bash
whoami
hostname
aws sts get-caller-identity
```

Sau đó:

```bash
curl -4 https://checkip.amazonaws.com
```

Giải thích IP này phải khớp source `/32` của SSH rule.

---

**Phút 1:00–2:15 — SSH / target environment**

```bash
ssh java-fresher-lab
```

Sau login:

```bash
hostnamectl
```

```bash
ip addr
```

```bash
ip route
```

Nói:

> "Em xác nhận đây đúng EC2 lab trước khi thay đổi bất kỳ cấu hình nào."

---

**Phút 2:15–3:15 — health baseline**

```bash
systemctl --failed --no-pager
```

và:

```bash
sudo ss -lntp
```

Giải thích nhanh:

```text
8080 → Tomcat
5432 → PostgreSQL
9090 → Prometheus
9100 → node_exporter
3000 → Grafana
```

---

**Phút 3:15–4:00 — Evidence structure**

Quay lại WSL hoặc dùng terminal khác:

```bash
find ~/java-fresher-demo/evidence -maxdepth 2 -type d | sort
```

Nói:

> "Evidence em chia theo Unit để sau này có thể trace thao tác. Với các thay đổi configuration em giữ evidence theo BEFORE, ACTION, AFTER, RESULT."

---

**Phút 4:00–5:00 — History**

Đây là phần cần nhấn mạnh.

```bash
tail -20 ~/java-fresher-demo/history/history.log
```

Nói:

> "Em không chỉ lưu command text mà cấu hình `HISTTIMEFORMAT` để history có timestamp. Master history nằm ở đây và em có thêm snapshot theo Unit/ngày."

Sau đó chọn một dòng ngay tại chỗ.

Ví dụ thấy:

```text
2026-10-06 02:50:41 sudo ss -lntp
```

Bạn phải tự giải thích:

> "`ss` dùng để inspect socket. `-l` là listening, `-n` không resolve service name, `-t` là TCP, `-p` hiển thị process. Em dùng lệnh này để xác nhận service thực sự bind vào port nào trên OS."

Đây chính là kiểu trình bày trainer muốn thấy.

---

# 28. Một lỗi demo rất dễ mắc

Bạn chạy:

```bash
history > history.log
```

nhưng file lại hiện:

```text
421 ls
422 pwd
423 ssh ...
```

không có ngày giờ.

Kiểm tra:

```bash
echo "$HISTTIMEFORMAT"
```

Nếu rỗng thì đó là nguyên nhân.

Fix:

```bash
export HISTTIMEFORMAT='%Y-%m-%d %H:%M:%S '
```

rồi:

```bash
history > history.log
```

Nhưng nhớ: việc này **không có nghĩa bạn có thể khôi phục chính xác timestamp lịch sử cũ nếu timestamp đó chưa từng được lưu**.

---

# 29. Mastery check của Unit 1

Trước khi coi Unit 1 hoàn thành, bạn phải tự trả lời được:

```text
Tại sao phải kiểm tra AWS identity trước khi thao tác?

Tại sao Security Group SSH dùng /32?

Tại sao evidence không nên chỉ nằm trên EC2?

Khác nhau giữa ~/.bash_history và output của history là gì?

HISTTIMEFORMAT có tác dụng gì?

history -a làm gì?

histappend giải quyết vấn đề gì?

Tại sao không được để password/token/private key xuất hiện trong history?

Tại sao ss thấy port listening chưa đủ để kết luận client Internet truy cập được?

Khi sửa config, tại sao phải:
backup → validate → apply → restart/reload → validate?
```

Nếu bạn giải thích chắc được các câu trên và thực sự có file:

```text
~/java-fresher-demo/history/history.log
```

với dữ liệu dạng:

```text
COMMAND_NUMBER  YYYY-MM-DD HH:MM:SS  COMMAND
```

thì **Unit 1 practical đã ở trạng thái sẵn sàng**.

Điểm quan trọng nhất trước khi sang Unit 2 là từ bây giờ hãy luôn giữ:

```bash
export HISTTIMEFORMAT='%Y-%m-%d %H:%M:%S '
```

trong `~/.bashrc` **cả WSL lẫn EC2**, và cuối mỗi buổi chạy:

```bash
history -a
history > ~/java-fresher-demo/history/history.log
```

để `history.log` của bạn thực sự trở thành evidence có **ngày + giờ + command**, đúng như bạn yêu cầu và dễ dùng cho phần Self Q&A sau này.
