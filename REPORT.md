# BÁO CÁO KỸ THUẬT: PHÂN TÍCH KIẾN TRÚC CONTAINER VS VIRTUAL MACHINE (VM)

**Dự án / Chuyên đề:** Chuyển dịch hạ tầng hệ thống từ VPS (Virtual Private Server) sang Docker Container  
**Người thực hiện:** Kỹ sư DevOps  
**Địa điểm lưu trữ:** `Bai01/REPORT.md`  

---

## 1. Bối cảnh & Mục tiêu dự án

Trong kiến trúc hạ tầng truyền thống, doanh nghiệp thường triển khai các ứng dụng (web services, background workers, databases) trên nhiều máy chủ ảo VPS (Virtual Private Server) hoặc Virtual Machines (VM) chạy trên nền tảng Hypervisor (VMware ESXi, KVM, Proxmox). 

Tuy nhiên, với sự phát triển của mô hình Microservices và nhu cầu triển khai liên tục (CI/CD), mô hình VM bắt đầu bộc lộ nhiều điểm nghẽn nghiêm trọng:
- **Lãng phí tài nguyên:** Mỗi VPS phải gánh một hệ điều hành khách (Guest OS) riêng biệt, tiêu tốn hàng trăm MB đến hàng GB RAM và dung lượng đĩa chỉ để duy trì kernel và tiến trình nền hệ điều hành.
- **Tốc độ triển khai và mở rộng chậm:** Thời gian khởi động máy chủ ảo tính bằng phút, gây cản trở việc tự động co giãn tải (Auto-scaling).
- **Thiếu tính nhất quán giữa các môi trường:** Xung đột phiên bản thư viện giữa môi trường Development, Staging và Production ("Works on my machine problem").

Nhằm giải quyết triệt để các hạn chế trên, doanh nghiệp quyết định chuyển đổi hạ tầng sang **Docker Container** (ảo hóa cấp độ hệ điều hành). Báo cáo này đi sâu phân tích kiến trúc, cơ chế cô lập kernel của Linux (Namespaces, cgroups), so sánh toàn diện hai mô hình và trích xuất các thông số vận hành then chốt từ `docker info`.

---

## 2. Bảng so sánh chuyên sâu: Virtual Machine vs Container

Sự khác biệt cốt lõi giữa Virtual Machine (ảo hóa phần cứng) và Container (ảo hóa hệ điều hành) được thể hiện qua bảng so sánh 5 tiêu chí dưới đây:

| Tiêu chí | Virtual Machine (VPS / Hypervisor) | Docker Container | Phân tích tác động kỹ thuật |
| :--- | :--- | :--- | :--- |
| **1. Dung lượng Image** | **Rất lớn** (Vài GB đến hàng chục GB) | **Rất nhỏ & tối ưu** (Vài chục MB đến vài trăm MB) | VM Image phải chứa toàn bộ Guest OS (Kernel, System binaries, libraries, swap). Trong khi đó, Docker Image chỉ đóng gói ứng dụng và các runtime dependency cần thiết (ví dụ: base image Alpine Linux chỉ khoảng 5MB), giúp tiết kiệm tối đa băng thông mạng khi kéo image (pull/push) và dung lượng ổ cứng lưu trữ. |
| **2. Thời gian khởi động** | **Chậm** (Tính bằng phút: 30s – vài phút) | **Gần như tức thì** (Mili-giây đến vài giây) | VM phải trải qua toàn bộ chu trình khởi động phần cứng ảo (virtual BIOS/UEFI), boot kernel, mount disk và init system (systemd). Container thực chất là một tiến trình người dùng (process) thông thường chạy trực tiếp trên Host Kernel, do đó khởi động ngay khi tiến trình được thực thi. |
| **3. Hao phí OS (Overhead)** | **Cao** (Tốn tài nguyên phần cứng đáng kể) | **Rất thấp / Tối thiểu** (Gần bằng native process) | Mỗi VM tiêu tốn CPU, RAM cố định cho kernel riêng, drivers ảo hóa, bộ nhớ đệm và các dịch vụ nền. Container dùng chung Host OS Kernel, không có tầng dịch mã Hypervisor, hiệu suất truy xuất CPU/Memory/IO đạt mức gần như bare-metal (nguyên bản). |
| **4. Mức độ cô lập (Isolation)** | **Rất cao / Cô lập hoàn toàn** (Hardware-level) | **Mức độ tương đối** (Process/OS-level) | VM được bảo vệ bởi ranh giới phần cứng ảo hóa (Hardware isolation). Lỗi hoặc lỗ hổng kernel của một VM không ảnh hưởng đến Host OS hay VM khác. Container chia sẻ chung Host Kernel; bảo mật dựa trên ranh giới mềm (Namespaces, cgroups, Seccomp, AppArmor). Nếu có lỗ hổng Kernel Privilege Escalation, khả năng ảnh hưởng tới host có thể xảy ra. |
| **5. Khả năng mở rộng (Scalability)** | **Thấp / Kém linh hoạt** (Khó scale tức thời) | **Rất cao & linh hoạt** (Thích hợp cho Kubernetes / Swarm) | Mở rộng ngang (Horizontal Scaling) với VM đòi hỏi cấp phát trước tài nguyên lớn và thời gian chờ provisioning lâu. Container có thể scale từ 1 pod lên hàng chục pod chỉ trong vài giây để đáp ứng đột biến lưu lượng truy cập (traffic spikes). |

---

## 3. Cơ chế cô lập & Quản lý tài nguyên trong Linux Container

Docker không phải là một công nghệ phần cứng thần kỳ, mà bản chất tận dụng hai tính năng nền tảng có sẵn trong Linux Kernel: **Linux Namespaces** và **Control Groups (cgroups)**.

```
+-------------------------------------------------------------------+
|                        Docker Engine                              |
+---------------------------------+---------------------------------+
|        Linux Namespaces         |      Control Groups (cgroups)   |
|        [TÍNH CÔ LẬP GÓC NHÌN]   |       [TÍNH GIỚI HẠN TÀI NGUYÊN] |
+---------------------------------+---------------------------------+
| • PID: Không gian tiến trình    | • CPU: Giới hạn % CPU core      |
| • NET: Card mạng, IP, Routing   | • Memory: Giới hạn RAM & Swap   |
| • MNT: File system riêng biệt   | • BlkIO: Giới hạn I/O ổ đĩa     |
| • UTS: Hostname, Domain name    | • PIDs: Giới hạn số tiến trình  |
| • IPC: Bộ nhớ dùng chung cô lập |                                 |
+---------------------------------+---------------------------------+
|                         Linux Host Kernel                         |
+-------------------------------------------------------------------+
```

### 3.1. Vai trò của Linux Namespaces (Cô lập góc nhìn tài nguyên)
Namespaces cung cấp cơ chế phân vùng tài nguyên hệ thống, khiến cho mỗi Container chỉ nhìn thấy một tập hợp tài nguyên độc lập như thể nó đang chạy trên một cỗ máy vật lý riêng biệt:

- **PID Namespace (Process ID):**
  - Cô lập cây tiến trình. Bên trong container, tiến trình chính của ứng dụng chạy với `PID = 1` (init process của container), có quyền quản lý và đón nhận tín hiệu hệ thống (SIGTERM, SIGKILL).
  - Tuy nhiên, trên Host OS, tiến trình này chỉ mang một PID bình thường (ví dụ: `PID = 34521`). Host có thể nhìn thấy tiến trình của container, nhưng container không thể nhìn thấy hoặc tương tác với bất kỳ tiến trình nào ngoài không gian của nó.
- **NET Namespace (Network):**
  - Cung cấp giao diện mạng ảo (veth pair), bảng định tuyến (routing table), bảng lọc iptables, danh sách socket và địa chỉ IP riêng cho từng container.
  - Nhờ NET namespace, hai container hoàn toàn có thể lắng nghe cùng cổng `80` hoặc `443` mà không hề xảy ra xung đột cổng trên Host.
- **MNT Namespace (Mount points):**
  - Tạo ra một bảng gắn kết (mount table) và cây thư mục root riêng biệt (`/`) cho container.
  - Kết hợp với công nghệ Union File System (OverlayFS), container sở hữu hệ thống file độc lập, không thể truy cập trái phép vào filesystem của Host trừ khi được mount chủ đích thông qua Docker Volumes hoặc Bind Mounts.
- **Các Namespaces bổ trợ:**
  - `UTS Namespace`: Cho phép mỗi container cấu hình Hostname và Domain name riêng.
  - `IPC Namespace`: Ngăn chặn các tiến trình khác container giao tiếp qua bộ nhớ chia sẻ (Shared Memory, POSIX message queues).
  - `USER Namespace`: Ánh xạ UID/GID bên trong container (ví dụ `root` UID 0) thành một user không đặc quyền (non-root UID 10000) trên Host, tăng cường bảo mật.

### 3.2. Vai trò của Control Groups - cgroups (Giới hạn và giám sát tài nguyên)
Nếu Namespaces chịu trách nhiệm trả lời câu hỏi: *"Tiến trình được phép **nhìn thấy** những gì?"*, thì cgroups chịu trách nhiệm trả lời câu hỏi: *"Tiến trình được phép **sử dụng bao nhiêu** tài nguyên?"*.

- **Giới hạn CPU (CPU Quota & Shares):**
  - Cơ chế Completely Fair Scheduler (CFS) bandwidth control cho phép giới hạn thời gian CPU mà container được sử dụng trong một chu kỳ (ví dụ: `--cpus="1.5"` cho phép container dùng tối đa 150% của 1 core CPU).
  - Tránh tình trạng một tiến trình bị lỗi vòng lặp vô tận (infinite loop) chiếm dụng 100% tài nguyên CPU của máy chủ, gây gián đoạn các dịch vụ khác (Noisy Neighbor Problem).
- **Giới hạn Bộ nhớ (Memory & Swap Limitation):**
  - Thiết lập trần sử dụng RAM (ví dụ: `--memory="512m"`).
  - Khi container vượt quá ngưỡng này, bộ quản lý Out-Of-Memory (OOM Killer) của kernel sẽ can thiệp hạ gục tiến trình vi phạm (`OOMKilled`) mà không làm sập Host OS hoặc ảnh hưởng đến các container bên cạnh.
- **Giới hạn I/O và số lượng tiến trình (BlkIO & PID limit):**
  - Giới hạn tốc độ đọc/ghi đĩa (`--device-read-bps`, `--device-write-bps`) và số lượng tiến trình tối đa (`--pids-limit`) nhằm ngăn chặn các cuộc tấn công Fork Bomb làm cạn kiệt bảng tiến trình hệ thống.

---

## 4. Trích xuất và Phân tích thông số cơ bản từ lệnh `docker info`

Lệnh `docker info` hiển thị toàn bộ trạng thái hoạt động của Docker Client và Docker Daemon (Server). Dưới đây là kết quả thực tế và phân tích chi tiết các thông số trọng yếu:

### 4.1. Đầu ra log mẫu trích xuất từ hệ thống

```text
Client:
 Version:    29.7.2
 Context:    desktop-linux
 Debug Mode: false

Server:
 Containers: 3
  Running: 2
  Paused: 0
  Stopped: 1
 Images: 5
 Server Version: 29.7.2
 Storage Driver: overlay2
  Backing Filesystem: ext4
 Logging Driver: json-file
 Cgroup Driver: systemd
 Cgroup Version: 2
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
 Security Options:
  seccomp
   Profile: builtin
  cgroupns
 Kernel Version: 6.6.36.6-microsoft-standard-WSL2
 Operating System: Docker Desktop
 OSType: linux
 Architecture: x86_64
 CPUs: 12
 Total Memory: 15.48GiB
 Docker Root Dir: /var/lib/docker
```

### 4.2. Giải thích 4 nhóm thông số cốt lõi

1. **Server Version (`29.7.2`):**
   - Phiên bản Docker Engine Daemon đang phục vụ các yêu cầu API từ Docker Client.
   - Cho biết các tính năng API mới nhất, các bản vá bảo mật và tương thích với chuẩn OCI (Open Container Initiative).

2. **Cgroup Driver (`systemd`) & Cgroup Version (`2`):**
   - **Cgroup Driver:** Docker sử dụng `systemd` thay vì `cgroupfs`. Đây là chuẩn khuyến nghị cho các hệ thống Linux hiện đại (bao gồm Kubernetes), giúp đồng bộ hóa việc quản lý tài nguyên giữa trình khởi tạo hệ thống (init system) và container engine, tránh hiện tượng xung đột phân bổ hai luồng.
   - **Cgroup Version 2 (cgroup v2):** Mang lại kiến trúc phân cấp đơn (unified hierarchy), hỗ trợ quản lý bộ nhớ đệm (buffer I/O), rootless containers và kiểm soát tài nguyên chính xác hơn đáng kể so với cgroup v1 cũ.

3. **Số lượng Containers (`Containers: 3`):**
   - **Running (2):** Số container đang chạy, chiếm dụng CPU/RAM và các namespace hoạt động.
   - **Paused (0):** Số container đang bị tạm dừng (tiến trình bị đóng băng bằng tín hiệu `SIGSTOP` qua cgroup freezer).
   - **Stopped (1):** Số container đã dừng thực thi nhưng vẫn còn lưu trạng thái và layer ghi đĩa (chưa bị xóa bỏ bằng `docker rm`).

4. **Số lượng Images (`Images: 5`):**
   - Số lượng Docker Images hiện có trong local storage (`/var/lib/docker`).
   - Các image này là các tầng read-only layers được chia sẻ giữa các container, giúp tiết kiệm tối đa đĩa lưu trữ nhờ cơ chế Copy-on-Write (CoW).

---

## 5. Kết luận & Đề xuất kiến trúc cho doanh nghiệp

1. **Hiệu quả kinh tế & Kỹ thuật:**
   - Việc chuyển từ VPS sang Docker giúp doanh nghiệp nâng cao mật độ triển khai ứng dụng (density) lên từ 3 đến 5 lần trên cùng một cấu hình máy chủ vật lý, giảm đáng kể chi phí hạ tầng (Cloud bill).
   - Chu kỳ triển khai (deployment cycle) giảm từ hàng chục phút xuống dưới 1 phút.

2. **Khuyến nghị vận hành:**
   - Sử dụng **Multi-stage builds** để giữ kích thước Docker Image ở mức tối thiểu.
   - Luôn thiết lập giới hạn `memory` và `cpus` cho từng container trong `docker-compose.yml` hoặc file cấu hình để đảm bảo tính ổn định của hệ thống.
   - Không chạy container dưới quyền `root` (kích hoạt User Namespaces hoặc tạo non-root user trong Dockerfile).
   - Sử dụng chuẩn `Cgroup Driver: systemd` để sẵn sàng tích hợp với cụm điều phối Kubernetes trong tương lai.
