# Báo cáo Kỹ thuật: Phân tích Kiến trúc Container vs Virtual Machine

## 1. Giới thiệu
Ảo hóa là một trong những nền tảng quan trọng nhất của kiến trúc hạ tầng điện toán đám mây hiện đại. Hai giải pháp nổi bật nhất hiện nay là **Virtual Machine (VM)** dựa trên Hypervisor và **Container** dựa trên công nghệ Containerization của Docker Engine. Bài báo cáo này phân tích so sánh chi tiết cơ chế hoạt động, sự khác biệt kiến trúc và các công nghệ cốt lõi của nhân Linux (`Namespaces` và `cgroups`) được Docker sử dụng để cô lập tiến trình.

---

## 2. Bảng So sánh Chuyên sâu Virtual Machine vs Container

| Tiêu chí | Virtual Machine (Ảo hóa phần cứng) | Container (Ảo hóa hệ điều hành) |
| :--- | :--- | :--- |
| **Kiến trúc cốt lõi** | Hypervisor (Type 1 hoặc Type 2) ảo hóa toàn bộ phần cứng (CPU, RAM, NIC). Mỗi VM chạy một Guest OS hoàn chỉnh. | Docker Engine chia sẻ chung Host OS Kernel. Mỗi container là một tiến trình biệt lập. |
| **Dung lượng Image** | Rất lớn (từ hàng Gigabyte đến vài chục GB) do chứa toàn bộ nhân và thư viện hệ điều hành con. | Siêu nhẹ (vài Megabyte đến vài trăm MB, ví dụ `alpine` chỉ ~5MB). |
| **Thời gian khởi động**| Chậm (tính bằng phút do phải trải qua chu trình Boot OS đầy đủ). | Gần như tức thì (tính bằng mili-giây đến vài giây) vì chỉ khởi tạo tiến trình. |
| **Mức độ tiêu hao tài nguyên** | Cao (overhead lớn từ Hypervisor và việc chạy nhiều Guest OS trùng lặp). | Cực kỳ thấp (gần như tương đương tiến trình Native trên máy Host). |
| **Mức độ cô lập (Isolation)** | Tuyệt đối (Strong isolation ở tầng phần cứng). | Mức độ tiến trình (Process isolation dựa vào Linux Kernel). |
| **Khả năng mở rộng (Scalability)** | Hạn chế, mất nhiều thời gian cấp phát VM mới. | Vượt trội, lý tưởng cho kiến trúc Microservices và CI/CD. |

---

## 3. Vai trò của Linux Namespaces và Control Groups (cgroups)

Docker đạt được sự cô lập và quản lý tài nguyên hoàn hảo thông qua hai tính năng nền tảng của Linux Kernel:

### 3.1. Linux Namespaces (Cơ chế Cô lập - "Container nhìn thấy gì")
Namespaces tạo ra một môi trường giả lập độc lập cho tiến trình, khiến container tin rằng nó đang sở hữu toàn bộ hệ điều hành riêng:
- **PID Namespace:** Cô lập cây tiến trình. Tiến trình trong container có PID 1, nhưng ở Host OS chỉ là một tiến trình bình thường với PID thông thường.
- **NET Namespace:** Cung cấp giao diện mạng ảo (veth), bảng định tuyến, port và IP riêng biệt cho từng container.
- **MNT Namespace:** Cô lập cây thư mục mount point. Mỗi container có hệ thống tập tin root (`/`) riêng biệt.
- **IPC Namespace:** Ngăn chặn các tiến trình khác container giao tiếp qua bộ nhớ chia sẻ.
- **UTS Namespace:** Cho phép đặt Hostname và Domain name riêng cho từng container.
- **USER Namespace:** Ánh xạ UID/GID trong container (như user root) với UID thông thường trên Host, gia tăng bảo mật.

### 3.2. Linux Control Groups - cgroups (Cơ chế Quản lý Tài nguyên - "Container được dùng bao nhiêu")
cgroups kiểm soát và phân bổ tài nguyên phần cứng, ngăn chặn hiện tượng "noisy neighbor" (một container ngốn sạch tài nguyên):
- **CPU Limits:** Giới hạn tỷ lệ phần trăm CPU hoặc số core được phép sử dụng (ví dụ `--cpus="1.5"`).
- **Memory Limits:** Giới hạn dung lượng RAM tối đa (ví dụ `-m 512m`) và quản lý swap.
- **Block I/O:** Giới hạn tốc độ đọc/ghi đĩa cứng (IOPS/Bandwidth).
- **Process Number (pids):** Ngăn chặn tấn công fork-bomb bằng cách giới hạn số tiến trình con tối đa.

---

## 4. Trích xuất Thông số Hệ thống từ `docker info`

```text
Client: Docker Engine - Community
 Version:    26.1.4
 Context:    default
 Debug Mode: false

Server:
 Server Version: 26.1.4
 Storage Driver: overlay2
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
 Security Options:
  apparmor
  seccomp
   Profile: builtin
  cgroup2
 Operating System: Ubuntu 22.04.4 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 3.816GiB
 Containers: 3
  Running: 1
  Paused: 0
  Stopped: 2
 Images: 4
```

*Nhận xét:*
- **Server Version:** `26.1.4`.
- **Cgroup Driver:** `systemd` và **Cgroup Version:** `2` (hỗ trợ cgroups v2 thống nhất).
- **Storage Driver:** `overlay2` (hệ thống lưu trữ phân lớp hiệu quả cao nhất của Linux).
- Đang quản lý 3 containers và 4 images.
