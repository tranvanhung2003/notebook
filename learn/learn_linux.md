# Table of Contents <!-- omit from toc -->

- [1. Terminal và câu lệnh linux](#1-terminal-và-câu-lệnh-linux)
  - [1.1. Tổng quan](#11-tổng-quan)
  - [1.2. Tổng hợp các phím tắt](#12-tổng-hợp-các-phím-tắt)
    - [1.2.1. Tổng quan](#121-tổng-quan)
    - [1.2.2. Điều hướng](#122-điều-hướng)
    - [1.2.3. Lịch sử và hoàn thành tự động](#123-lịch-sử-và-hoàn-thành-tự-động)
    - [1.2.4. Tùy chỉnh cửa sổ terminal](#124-tùy-chỉnh-cửa-sổ-terminal)
    - [1.2.5. Chế độ chọn và dán](#125-chế-độ-chọn-và-dán)
  - [1.3. Lệnh sudo và cách đặt mật khẩu cho root](#13-lệnh-sudo-và-cách-đặt-mật-khẩu-cho-root)
    - [1.3.1. Tổng quan](#131-tổng-quan)

# 1. Terminal và câu lệnh linux

## 1.1. Tổng quan

| Phím tắt/Câu lệnh               | Mô tả                                            |
| ------------------------------- | ------------------------------------------------ |
| `command [options] [arguments]` | Cú pháp câu lệnh                                 |
| `man <command>`                 | Xem tài liệu hướng dẫn của một câu lệnh          |
| `Page Up`/`Page Down`           | Di chuyển lên và xuống một trang                 |
| Phím `q`                        | Thoát khỏi tài liệu và trở về dòng lệnh          |
| `Ctrl + D` or `exit`            | Thoát khỏi terminal                              |
| `Up Arrow`/`Down Arrow`         | Xem các câu lệnh được thực hiện trong lịch sử    |
| `Ctrl + C`                      | Ngắt ngang một câu lệnh đang chạy trong terminal |

- Trong `command [options] [arguments]`:
  - Options: các cú pháp tùy chọn thường được đặt trước các đối số và thường bắt đầu bằng một dấu gạch ngang (`-`) hoặc hai dấu gạch ngang (`--`).
  - Arguments: các giá trị hoặc tham số mà câu lệnh sẽ hoạt động trên đó. Đối số thường là các tên tệp, thư mục hoặc các giá trị khác cần thiết cho câu lệnh.

## 1.2. Tổng hợp các phím tắt

### 1.2.1. Tổng quan

| Phím tắt/Câu lệnh  | Mô tả                                      |
| ------------------ | ------------------------------------------ |
| `Ctrl + Alt + T`   | Mở terminal                                |
| `Ctrl + Shift + T` | Mở một tab terminal mới                    |
| `Ctrl + D`         | Thoát khỏi terminal hoặc đóng tab hiện tại |
| `Ctrl + C`         | Dừng một quy trình hoặc câu lệnh đang chạy |

### 1.2.2. Điều hướng

| Phím tắt/Câu lệnh | Mô tả                               |
| ----------------- | ----------------------------------- |
| `Ctrl + A`        | Di chuyển đến đầu dòng lệnh         |
| `Ctrl + E`        | Di chuyển đến cuối dòng lệnh        |
| `Ctrl + U`        | Xóa từ vị trí con trỏ đến đầu dòng  |
| `Ctrl + K`        | Xóa từ vị trí con trỏ đến cuối dòng |
| `Alt + B`         | Di chuyển con trỏ sang trái một từ  |
| `Alt + F`         | Di chuyển con trỏ sang phải một từ  |

### 1.2.3. Lịch sử và hoàn thành tự động

| Phím tắt/Câu lệnh       | Mô tả                                              |
| ----------------------- | -------------------------------------------------- |
| `Up Arrow`/`Down Arrow` | Lấy các lệnh trước đó hoặc tiếp theo trong lịch sử |
| `Ctrl + R`              | Tìm kiếm lệnh trong lịch sử                        |
| `Tab`                   | Hoàn thành tự động                                 |

### 1.2.4. Tùy chỉnh cửa sổ terminal

| Phím tắt/Câu lệnh  | Mô tả                    |
| ------------------ | ------------------------ |
| `Ctrl + Shift + +` | Phóng to kích thước font |
| `Ctrl + Shift + -` | Giảm kích thước font     |
| `Ctrl + Shift + W` | Đóng tab hiện tại        |

### 1.2.5. Chế độ chọn và dán

| Phím tắt/Câu lệnh  | Mô tả                      |
| ------------------ | -------------------------- |
| `Ctrl + Shift + C` | Sao chép văn bản được chọn |
| `Ctrl + Shift + V` | Dán văn bản từ clipboard   |

## 1.3. Lệnh sudo và cách đặt mật khẩu cho root

### 1.3.1. Tổng quan

| Phím tắt/Câu lệnh   | Mô tả                                                                                  |
| ------------------- | -------------------------------------------------------------------------------------- |
| `sudo <command>`    | Cú pháp sudo                                                                           |
| `sudo apt update`   | Cập nhật list các gói phần mềm nhưng không thực hiện cài đặt các update mới            |
| `sudo apt upgrade`  | Cập nhật các gói phần mềm đã được cài đặt lên phiên bản mới nhất có sẵn từ kho lưu trữ |
| `passwd [username]` | Đặt mật khẩu cho một người dùng cụ thể                                                 |
| `sudo passwd root`  | Đặt mật khẩu cho người dùng root                                                       |

- `sudo` là viết tắt của `superuser do`, dùng để thực thi một lệnh với quyền hạn của một người dùng khác, thường là người dùng root hoặc một người dùng có quyền hạn cao hơn.
- Khi dùng lệnh `sudo`, hệ thống sẽ yêu cầu ta nhập mật khẩu của người dùng hiện tại để xác minh danh tính. Nếu mật khẩu đúng thì câu lệnh ở sau `sudo` sẽ được thực thi với quyền hạn của người dùng root hoặc một người dùng khác được chỉ định trong tệp cấu hình sudoers.
- Lệnh `sudo` hữu ích trong việc thực thi các lệnh hoặc chương trình yêu cầu quyền hạn cao, như cài đặt hoặc xóa các gói phần mềm, quản lý hệ thống, chỉnh sửa các tệp cấu hình hệ thống, ... Nó cũng giúp ngăn chặn người dùng không có quyền truy cập vào các hoạt động nguy hiểm hoặc có thể gây hại cho hệ thống.
- Trong linux, tài khoản root là tài khoản có quyền hạn cao nhất trong hệ thống, có tòn quyền truy cập vào tất cả các tệp tin và thư mục trên hệ thống, có thể thực thi bất kỳ lệnh nào với quyền hạn cao nhất, và có khả năng thay đổi cấu hình hệ thống.
