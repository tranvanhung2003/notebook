# Table of Contents <!-- omit from toc -->

- [1. Cú pháp câu lệnh trong linux](#1-cú-pháp-câu-lệnh-trong-linux)
  - [1.1. Tổng quan](#11-tổng-quan)
  - [1.2. Tổng hợp các phím tắt](#12-tổng-hợp-các-phím-tắt)
    - [1.2.1. Tổng quan](#121-tổng-quan)
    - [1.2.2. Điều hướng](#122-điều-hướng)
    - [1.2.3. Lịch sử và hoàn thành tự động](#123-lịch-sử-và-hoàn-thành-tự-động)
    - [1.2.4. Tùy chỉnh cửa sổ terminal](#124-tùy-chỉnh-cửa-sổ-terminal)
    - [1.2.5. Chế độ chọn và dán](#125-chế-độ-chọn-và-dán)

# 1. Cú pháp câu lệnh trong linux

## 1.1. Tổng quan

| Tác vụ                                           | Cách thực hiện                         |
| ------------------------------------------------ | -------------------------------------- |
| Cấu trúc cú pháp                                 | `command [options] [arguments]`        |
| Xem tài liệu hướng dẫn của một câu lệnh          | `man <command>`                        |
| Thoát khỏi terminal                              | `history`                              |
| Xem các câu lệnh được thực hiện                  | dùng các phím mũi tên `lên` và `xuống` |
| Ngắt ngang một câu lệnh đang chạy trong terminal | `Ctrl + C`                             |

## 1.2. Tổng hợp các phím tắt

### 1.2.1. Tổng quan

| Phím tắt           | Mô tả                                      |
| ------------------ | ------------------------------------------ |
| `Ctrl + Alt + T`   | Mở terminal                                |
| `Ctrl + Shift + T` | Mở một tab terminal mới                    |
| `Ctrl + D`         | Thoát khỏi terminal hoặc đóng tab hiện tại |
| `Ctrl + C`         | Dừng một quy trình hoặc câu lệnh đang chạy |

### 1.2.2. Điều hướng

| Phím tắt   | Mô tả                               |
| ---------- | ----------------------------------- |
| `Ctrl + A` | Di chuyển đến đầu dòng lệnh         |
| `Ctrl + E` | Di chuyển đến cuối dòng lệnh        |
| `Ctrl + U` | Xóa từ vị trí con trỏ đến đầu dòng  |
| `Ctrl + K` | Xóa từ vị trí con trỏ đến cuối dòng |
| `Alt + B`  | Di chuyển con trỏ sang trái một từ  |
| `Alt + F`  | Di chuyển con trỏ sang phải một từ  |

### 1.2.3. Lịch sử và hoàn thành tự động

| Phím tắt                  | Mô tả                                              |
| ------------------------- | -------------------------------------------------- |
| `Up Arrow` / `Down Arrow` | Lấy các lệnh trước đó hoặc tiếp theo trong lịch sử |
| `Ctrl + R`                | Tìm kiếm lệnh trong lịch sử                        |
| `Tab`                     | Hoàn thành tự động                                 |

### 1.2.4. Tùy chỉnh cửa sổ terminal

| Phím tắt           | Mô tả                    |
| ------------------ | ------------------------ |
| `Ctrl + Shift + +` | Phóng to kích thước font |
| `Ctrl + Shift + -` | Giảm kích thước font     |
| `Ctrl + Shift + W` | Đóng tab hiện tại        |

### 1.2.5. Chế độ chọn và dán

<!-- tạo một bảng mẫu 2 x 5 -->

| Phím tắt           | Mô tả                      |
| ------------------ | -------------------------- |
| `Ctrl + Shift + C` | Sao chép văn bản được chọn |
| `Ctrl + Shift + V` | Dán văn bản từ clipboard   |
