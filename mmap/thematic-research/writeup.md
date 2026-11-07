# Kernel : Chuyên đề MAP_SHARED (MAP_SHARED Thematic Research)

**Mục lục**

<table>
<tr>
<td valign="top" width="50%">

- [1.Tổng quan về POSIX Shared Memory](#1tổng-quan-về-posix-shared-memory)
  - [1.1. Shared Memory là gì?](#11-shared-memory-là-gì)
  - [1.2. Shared Memory trong IPC](12-shared-memory-trong-ipc)
     - [1.2.1. IPC là gì?](#121-ipc-là-gì)
	 - [1.2.2. Shared Memory hoạt động như thế nào?](#122-shared-memory-hoạt-động-như-thế-nào)
  - [1.3. So sánh Shared Memory với Pipe, FIFO và Socket](#13-so-sánh-shared-memory-với-pipe-fifo-và-socket)
  - [1.4. POSIX Shared Memory và System V Shared Memory](#14-posix-shared-memory-và-system-v-shared-memory)
  - 1.5. Kiến trúc tổng thể giữa User Space và Kernel Space
- 2.Shared Memory Object
  - 2.1. Shared Memory Object là gì?
  - 2.2. Namespace và tên đối tượng
  - 2.3. File descriptor và vòng đời đối tượng
  - 2.4. Quan hệ giữa Shared Memory Object và Virtual Memory Area
  - 2.5. Backing storage và Page Cache trên Linux
- 3.`shm_open()`
  - 3.1. Function prototype
  - 3.2. Tham số `name`, `oflag`, `mode`
  - 3.3. `O_CREAT`, `O_EXCL`, `O_RDWR`
  - 3.4. Cơ chế tạo và mở Shared Memory Object
  - 3.5. File descriptor được trả về
  - 3.6. Error handling
  - 3.7. Phân tích System Call và Kernel implementation
- 4.`ftruncate()`
  - 4.1. Function prototype
  - 4.2. Tại sao cần thiết lập kích thước?
  - 4.3. Thay đổi kích thước đối tượng shared memory
  - 4.4. Quan hệ giữa object size và mapping length
  - 4.5. File metadata và kernel implementation
  - 4.6. Các lỗi thường gặp
  - 4.7. Phân tích hành vi khi truy cập vùng mapping vượt kích thước đối tượng

</td>
<td valign="top" width="50%">

- 5.`mmap()` với POSIX Shared Memory
  - 5.1. Mapping Shared Memory Object vào Virtual Address Space
  - 5.2. `MAP_SHARED` và `PROT_READ | PROT_WRITE`
  - 5.3. File descriptor trong `mmap()`
  - 5.4. VMA và Page Table
  - 5.5. Demand Paging
  - 5.6. Page Cache và các mapping dùng chung
  - 5.7. Quan sát bằng GDB và `/proc/PID/maps`
- 6.`shm_unlink()`
  - 6.1. Function prototype
  - 6.2. Xóa tên Shared Memory Object
  - 6.3. Khác biệt giữa `shm_unlink()` và `close()`
  - 6.4. Khác biệt giữa `shm_unlink()` và `munmap()`
  - 6.5. Reference counting và vòng đời object
  - 6.6. Khi nào bộ nhớ thực sự được giải phóng?
  - 6.7. Phân tích kernel implementation
- 7.Vòng đời Shared Memory
  - 7.1. Tạo object
  - 7.2. Thiết lập kích thước
  - 7.3. Mapping vào các tiến trình
  - 7.4. Trao đổi dữ liệu
  - 7.5. Đồng bộ hóa giữa các tiến trình
  - 7.6. Unlink và giải phóng tài nguyên
- 8.Đồng bộ hóa (Synchronization)
  - 8.1. Race Condition
  - 8.2. Tín hiệu cờ POSIX (POSIX Semaphore)
  - 8.3. Mutex dùng chung giữa các tiến trình (Process-shared Mutex)
	 - 8.3.1. Mutex là gì?
  - 8.4. Sắp xếp bộ nhớ (Memory Ordering)
  - 8.5. Đồng bộ hóa và tính nhất quán dữ liệu
- 9.Thực nghiệm và Debugging
  - 9.1. Hai tiến trình độc lập sử dụng Shared Memory
  - 9.2. Quan sát Virtual Address Space
  - 9.3. Kiểm tra `/proc/PID/maps`
  - 9.4. Kiểm tra `/proc/PID/smaps`
  - 9.5. GDB: Theo dõi thao tác đọc/ghi
  - 9.6. Theo dõi System Call bằng strace
  - 9.7. Thực nghiệm vòng đời object

</td>
</tr>
</table>

---

## 1.Tổng quan về POSIX Shared Memory
### 1.1. Shared Memory là gì?

là một cơ chế Inter-Process Communication (IPC) cho phép nhiều tiến trình (process) cùng truy cập vào một vùng bộ nhớ chung. Đặc điểm chính
- Nó có tốc độ rất cao vì dữ liệu không cần copy qua kernel (khác với pipe, message queue…).
- Các tiến trình đọc/ghi trực tiếp trên cùng một vùng nhớ.
- Thường dùng kết hợp với cơ chế đồng bộ (semaphore, mutex, condition variable…) để tránh race condition.

Các hàm POSIX chính

<div align="center">

| Hàm | Mục đích |
|:-:|:-|
| `shm_open()` | Tạo hoặc mở một shared memory object |
| `ftruncate()` | Đặt kích thước cho shared memory |
| `mmap()` | Ánh xạ shared memory vào không gian địa chỉ của process |
| `munmap()` | Hủy ánh xạ |
| `shm_unlink()` | Xóa shared memory object |
| `close()` | Đóng file descriptor |

</div>

### 1.2. Shared Memory trong IPC

là một trong những cơ chế giao tiếp giữa các tiến trình (Inter-Process Communication) nhanh nhất.

#### 1.2.1. IPC là gì?

IPC (Inter-Process Communication) là tập hợp các kỹ thuật cho phép các process trao đổi dữ liệu và đồng bộ với nhau. Các cơ chế IPC phổ biến gồm:

<div align="center">

| Cơ chế | Tốc độ | Độ phức tạp | Sao chép dữ liệu | Ghi chú |
|-|-|-|-|-|
| Shared Memory | Rất nhanh | Trung bình | Không | Nhanh nhất |
| Pipe / Named Pipe | Trung bình | Thấp | Có | Dễ dùng |
| Message Queue | Trung bình | Trung bình | Có | Có cấu trúc |
| Socket | Chậm hơn | Cao | Có | Có thể dùng mạng |
| Signal | Nhanh | Thấp | Không | Chỉ thông báo |

</div>

#### 1.2.2. Shared Memory hoạt động như thế nào?

Hệ thống tạo một vùng nhớ chung (shared memory region). Các process ánh xạ (map) vùng nhớ này vào không gian địa chỉ của riêng chúng. Sau đó chúng có thể đọc/ghi trực tiếp trên cùng một vùng nhớ từ đó không cần copy dữ liệu qua kernel.

- **Ưu điểm**
  - Tốc độ cao nhất trong các cơ chế IPC (không phải copy dữ liệu).
  - Phù hợp khi cần truyền lượng dữ liệu lớn hoặc truy cập thường xuyên.
- **Nhược điểm**
  - **Không tự đồng bộ:** Dễ bị race condition nếu không dùng thêm cơ chế đồng bộ (semaphore, mutex, condition variable…).
  - Khó quản lý hơn so với pipe hay message queue.
  - Cần xử lý cẩn thận việc tạo, xóa và quyền truy cập.

### 1.3. So sánh Shared Memory với Pipe, FIFO và Socket

đây là bảng so sánh các cơ chế IPC phổ biến:

<div align="center">

| Tiêu chí | Shared Memory | Pipe (Unnamed) | FIFO (Named Pipe) | Socket |
|-|-|-|-|-|
| Tốc độ | Rất nhanh (nhanh nhất) | Trung bình | Trung bình | Chậm hơn (nhất là network) |
| Sao chép dữ liệu | Không | Có (qua kernel) | Có (qua kernel) | Có (qua kernel) |
| Hướng truyền | Hai chiều (tùy cách dùng) | Một chiều | Một chiều (thường) | Hai chiều |
| Đồng bộ | Phải tự làm (semaphore…) | Kernel hỗ trợ một phần | Kernel hỗ trợ một phần | Kernel hỗ trợ tốt |
| Process liên quan | Bất kỳ process nào | Thường là quan hệ cha-con | Bất kỳ process nào | Bất kỳ (kể cả khác máy) |
| Tồn tại sau khi process kết thúc | Có (cần `shm_unlink`) | Không | Có (nằm trên filesystem) | Không |
| Độ phức tạp lập trình | Cao | Thấp | Trung bình | Trung bình đến cao |
| Phạm vi | Chỉ trong máy | Chỉ trong máy | Chỉ trong máy | Trong máy hoặc qua mạng |
| Kiểu dữ liệu | Bộ nhớ thô (byte) | Byte stream | Byte stream | Stream hoặc Datagram |

</div>

Dựa vào bảng là tóm tắt, chúng ta phân tích chi tiết độ khác nhau giữa 4 cơ chế IPC :

- 1.Shared Memory
  - **Ưu điểm:** Tốc độ cao nhất vì process đọc/ghi trực tiếp trên cùng vùng nhớ, không phải copy dữ liệu.
  - **Nhược điểm:** Không có cơ chế đồng bộ sẵn nên dễ bị race condition. Phải dùng thêm semaphore/mutex.
  - **Phù hợp:** Khi cần truyền dữ liệu lớn hoặc truy cập rất thường xuyên (xử lý ảnh, video, cache…).
- 2.Pipe (Unnamed Pipe)
  - Chỉ dùng được giữa các process có quan hệ (thường là cha – con).
  - Một chiều (một đầu đọc, một đầu ghi).
  - Đơn giản, an toàn hơn Shared Memory vì kernel quản lý buffer.
  - **Phù hợp:** Truyền dữ liệu đơn giản giữa process cha và con.
- 3.FIFO (Named Pipe)
  - Giống Pipe nhưng có tên trong filesystem nên các process không liên quan vẫn dùng được.
  - Vẫn là byte stream, một chiều.
  - Tồn tại trên đĩa (thực ra là special file), nên có thể mở lại sau khi process kết thúc.
  - **Phù hợp:** Khi cần giao tiếp giữa các process độc lập trên cùng máy mà không muốn dùng Shared Memory.
- 4.Socket
  - **Linh hoạt nhất:** dùng được cả local (Unix Domain Socket) lẫn qua mạng (TCP/UDP).
  - Hai chiều, hỗ trợ tốt việc đồng bộ và điều khiển luồng.
  - Chi phí cao hơn vì phải qua nhiều lớp (đặc biệt khi dùng network).
  - **Phù hợp:** Khi cần giao tiếp phức tạp, hoặc giữa các máy khác nhau, hoặc muốn giao diện thống nhất (local + remote).