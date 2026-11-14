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
  - [1.4. Kiến trúc tổng thể giữa User Space và Kernel Space](#14-kiến-trúc-tổng-thể-giữa-user-space-và-kernel-space)
	 - [1.4.1. User Space và Kernel Space là gì?](#141-user-space-và-kernel-space-là-gì)
	 - [1.4.2. Vai trò của User Space trong POSIX Shared Memory](#142-vai-trò-của-user-space-trong-posix-shared-memory)
	 - [1.4.3. Vai trò của Linux Kernel](#143-vai-trò-của-linux-kernel)
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

### 1.4. Kiến trúc tổng thể giữa User Space và Kernel Space
#### 1.4.1. User Space và Kernel Space là gì?

Khái niệm user space (hay còn gọi là userland) là không gian mà các chương trình ứng dụng của người dùng chạy trong hệ điều hành. Đây là không gian người dùng, khá hạn chế và nó ko được phép truy cập tới phần cứng. Cho ví dụ là khi bạn mở một trình duyệt firefox và thao tác trên đó, thì nó đang ở userland, còn mà khi nó muốn gọi hay gửi dữ liệu qua mạng, nó cần call API linux, lúc này system call xuống kernel (trong kernel space) để xử lý giúp 

Khái niệm kernel space (ko gian nhãn) là vùng bộ nhớ và môi trường thực thi dành riêng cho nhân hệ điều hành (kernel). Đây là không gian có đặc quyền cao nhất (privileged mode / ring 0), có quyền truy cập toàn bộ phần cứng: (CPU, RAM, ổ đĩa, card mạng, GPU...) Quản lý tài nguyên hệ thống: bộ nhớ, tiến trình, file system, driver thiết bị, bảo mật…

Code trong kernel space nếu bị lỗi thường làm sập cả hệ thống (kernel panic / BSOD). Chỉ kernel và các module/driver được tải vào kernel mới chạy ở đây.

#### 1.4.2. Vai trò của User Space trong POSIX Shared Memory

Trong kiến trúc POSIX Shared Memory trên Linux, User Space (không gian người dùng) là môi trường thực thi các chương trình ứng dụng có nhu cầu sử dụng cơ chế chia sẻ bộ nhớ giữa nhiều tiến trình (Inter-Process Communication - IPC).Tại đây, chương trình không trực tiếp quản lý Physical Memory hay Page Table của hệ thống. Thay vào đó, nó sử dụng các giao diện lập trình (API) do POSIX cung cấp để yêu cầu Kernel thực hiện những thao tác liên quan đến việc tạo, quản lý và ánh xạ vùng bộ nhớ dùng chung.

Có thể hiểu rằng User Space chịu trách nhiệm đưa ra các yêu cầu và sử dụng vùng Shared Memory sau khi được ánh xạ thành công, trong khi Linux Kernel chịu trách nhiệm quản lý những tài nguyên hệ thống phía sau.

- **a.Chương trình ứng dụng (Application):** Ở User Space, mọi hoạt động sử dụng POSIX Shared Memory đều bắt đầu từ chương trình ứng dụng. **Ví dụ**, một chương trình C có thể sử dụng các thư viện và header sau:

  ```c
  #include <stdio.h>
  #include <stdlib.h>
  #include <fcntl.h>
  #include <sys/mman.h>
  #include <sys/stat.h>
  #include <unistd.h>
  ```

  Các header này cung cấp những khai báo cần thiết để chương trình sử dụng các API liên quan đến POSIX Shared Memory. Một quy trình sử dụng cơ bản:

  ```c
  int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);

  ftruncate(fd, 4096);

  void *ptr = mmap(
      NULL,
      4096,
      PROT_READ | PROT_WRITE,
      MAP_SHARED,
      fd,
      0
  );
  ```

  Trong đoạn chương trình trên:
  - `shm_open()` yêu cầu tạo hoặc mở một Shared Memory Object và trả về file descriptor.
  - `ftruncate()` yêu cầu thiết lập kích thước object thành 4096 bytes.
  - `mmap()` yêu cầu ánh xạ object vào Virtual Address Space của tiến trình.

> [!NOTE]
> **Lưu ý:** Đây là ví dụ minh họa các thao tác API, chưa phải chương trình hoàn chỉnh vì chưa kiểm tra lỗi và giải phóng tài nguyên.

- **b.thư viện C (C Library - glibc):** Các chương trình C thông thường không trực tiếp tự xây dựng cơ chế System Call cho từng thao tác. Thay vào đó, chúng sử dụng các hàm thư viện được cung cấp bởi C Library, chẳng hạn glibc trên Linux. **Ví dụ:**

  ```c
  shm_open();
  ftruncate();
  mmap();
  munmap();
  shm_unlink();
  ```

  Các API này cung cấp giao diện lập trình thuận tiện để ứng dụng tương tác với những cơ chế quản lý tài nguyên của Kernel.

  **Một điểm cần phân biệt:** không nên mặc định rằng mỗi API tương ứng với đúng một System Call. Chẳng hạn, `shm_open()` trên Linux thường được triển khai thông qua các thao tác filesystem như `openat()`, chứ không nhất thiết có một System Call riêng mang tên `shm_open`.
- **c. Quản lý File Descriptor:** Sau khi gọi `shm_open()` thành công, chương trình nhận được một giá trị int đại diện cho File Descriptor (FD). **Ví dụ:**

  ```c
  int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
  ```

  Giá trị fd là một định danh mà tiến trình sử dụng để tham chiếu đến một file description do Kernel quản lý. File Descriptor cho phép chương trình thực hiện những thao tác tiếp theo trên Shared Memory Object, chẳng hạn:
  - Thiết lập kích thước thông qua `ftruncate()`.
  - Ánh xạ object thông qua `mmap()`.
  - Đóng descriptor thông qua `close()`.

> [!IMPORTANT]
> Điều quan trọng là File Descriptor không phải địa chỉ bộ nhớ của Shared Memory. Nó chỉ là một định danh trong bảng File Descriptor của tiến trình.

- **d. Sử dụng Shared Memory sau khi Mapping:** Sau khi `mmap()` thành công, chương trình nhận được một Virtual Address trỏ đến vùng nhớ đã ánh xạ. **Ví dụ:**

  ```c
  char *ptr = mmap(
      NULL,
      4096,
      PROT_READ | PROT_WRITE,
      MAP_SHARED,
      fd,
      0
  );
  
  if (ptr == MAP_FAILED) {
      perror("mmap");
      exit(EXIT_FAILURE);
  }

  ptr[0] = 'A';
  ptr[1] = 'B';
  ```

  Ở đây, `ptr` là một địa chỉ Virtual Address thuộc không gian địa chỉ của tiến trình hiện tại. Khi chương trình thực hiện `ptr[0] = 'A';`, CPU thực hiện thao tác ghi bộ nhớ thông qua cơ chế dịch địa chỉ Virtual Address sang Physical Address. Nếu trang bộ nhớ tương ứng đã được ánh xạ và truy cập hợp lệ, CPU có thể ghi trực tiếp vào Physical Page tương ứng mà không cần thực hiện một System Call riêng cho mỗi lần ghi.

> [!IMPORTANT]
> **Đây là điểm rất quan trọng:** System Call được sử dụng để thiết lập mapping, nhưng các thao tác đọc/ghi dữ liệu thông thường sau đó diễn ra trực tiếp thông qua cơ chế truy cập bộ nhớ của CPU.

- **e. Trách nhiệm của User Space:** Tổng hợp lại, User Space chịu trách nhiệm:
  - **Application:** Quyết định cách sử dụng Shared Memory
  - **POSIX API:** Cung cấp giao diện thao tác với Shared Memory
  - **C Library:** Cung cấp implementation của API
  - **File Descriptor:** Tham chiếu đến object được Kernel quản lý
  - **Virtual Address:** Địa chỉ để chương trình truy cập vùng mapping
  - **Synchronization:** Tổ chức đồng bộ giữa các process
  - **Error Handling:** Kiểm tra và xử lý lỗi từ các API

User Space không trực tiếp quản lý cách Kernel cấp phát Physical Pages, thiết lập Page Table hay quản lý VMA. Nó chỉ tương tác với những tài nguyên này thông qua các giao diện được hệ điều hành cung cấp.Sau khi mapping thành công, chương trình có thể truy cập dữ liệu thông qua Virtual Address như cách nó truy cập một vùng bộ nhớ thông thường. Chính cơ chế này cho phép nhiều tiến trình độc lập cùng thao tác trên một vùng Physical Memory được chia sẻ.

#### 1.4.3. Vai trò của Linux Kernel

Trong kiến trúc POSIX Shared Memory, Linux Kernel (nhân hệ điều hành Linux) đóng vai trò quản lý và điều phối toàn bộ tài nguyên hệ thống cần thiết để triển khai cơ chế chia sẻ bộ nhớ giữa các tiến trình. Nếu User Space chịu trách nhiệm gọi API, thiết lập yêu cầu và sử dụng vùng nhớ sau khi mapping, thì Kernel chịu trách nhiệm xử lý những yêu cầu đó, quản lý Shared Memory Object, tổ chức ánh xạ bộ nhớ ảo và đảm bảo các tiến trình có thể truy cập đến cùng một vùng dữ liệu vật lý.

Để hiểu rõ hơn, cần phân tích các thành phần mà Kernel quản lý trong quá trình POSIX Shared Memory hoạt động.

- **a. Quản lý Shared Memory Object:** Khi một chương trình gọi `shm_open()`, yêu cầu tạo hoặc mở một Shared Memory Object được chuyển đến Kernel thông qua các giao diện hệ thống. Trên Linux, POSIX Shared Memory thường được triển khai dựa trên `tmpfs`, thường được mount tại `/dev/shm`. **Ví dụ:**

  ```c
   int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
  ```

  Ở mức triển khai Linux, tên `/my_shm` thường tương ứng với một đối tượng filesystem có tên `my_shm` bên trong `/dev/shm`. Kernel chịu trách nhiệm:
  - Tra cứu tên object trong namespace tương ứng.
  - Kiểm tra quyền truy cập.
  - Tạo mới hoặc mở object đã tồn tại.
  - Quản lý metadata của object.
  - Duy trì các tham chiếu đến object trong suốt vòng đời sử dụng.

  Cần phân biệt rằng Shared Memory Object không đơn giản là một vùng địa chỉ Virtual Memory. Nó là một đối tượng được quản lý bởi Kernel, có thể được nhiều tiến trình mở và ánh xạ vào không gian địa chỉ riêng của chúng.

<details>
	<summary><b>[Câu hỏi]</b> /dev/shm là gì ở linux?</summary>
<table>
<tr>
<td>

---

`/dev/shm` là một thư mục đặc biệt trên Linux, thường được sử dụng làm mount point cho filesystem `tmpfs`, nhằm cung cấp không gian lưu trữ cho các đối tượng POSIX Shared Memory.

**Nói đơn giản:** khi một chương trình sử dụng `shm_open()` để tạo POSIX Shared Memory Object, trên Linux, đối tượng đó thường được biểu diễn bằng một filesystem object nằm trong `/dev/shm`.

Điều này tạo ra mối liên hệ trực tiếp giữa POSIX Shared Memory API và cơ chế filesystem của Linux.

- **`/dev/shm` nằm ở đâu trong linux?:** Trên Linux, /dev/shm là một đường dẫn thuộc filesystem hierarchy (Cấu trúc phân cấp hệ thống tệp). **Ví dụ:**

  ```bash
  $ ls -ld /dev/shm
  drwxrwxrwt 2 root root 40 Nov 12 20:00 /dev/shm
  ```

  Kiểm tra filesystem đang được mount:

  ```bash
  $ findmnt /dev/shm
  ```

  một kết quả thường gặp:

  ```bash
  TARGET   SOURCE FSTYPE OPTIONS
  /dev/shm tmpfs  tmpfs  rw,nosuid,nodev,inode64
  ```

  Trong đó:
  - `/dev/shm`: Mount point
  - `tmpfs`: Filesystem lưu trữ dữ liệu trong bộ nhớ ảo
  - `rw`: Cho phép đọc và ghi
  - `nosuid`: Không áp dụng hiệu lực SUID/SGID từ file trên filesystem
  - `nodev`: Không cho phép sử dụng device special files

  **Lưu ý:** `tmpfs` không phải một phân vùng ổ đĩa vật lý. Nó sử dụng cơ chế quản lý bộ nhớ của Kernel và có thể sử dụng swap khi cần.

<details>
<summary><b>[Chi tiết]</b> Thử với hệ thống thật</summary>

---

Kết quả:

<p align="center">
<img src="image/image1.png" alt="kết quả terminal">
</p>

Ở đây ta thấy 

```bash
TARGET   SOURCE FSTYPE OPTIONS
/dev/shm tmpfs  tmpfs  rw,nosuid,nodev,inode64,huge=advise,usrquota
```

Phần options, nó có nghĩa là :

- `rw`: Mount ở chế độ read-write (đọc + ghi được).
- `nosuid`: Không cho phép các bit SUID/SGID có hiệu lực trên filesystem này. (An toàn hơn, tránh privilege escalation)
- `nodev`: Không cho phép sử dụng device file (character/block device) trên filesystem này.
- `inode64`: Sử dụng 64-bit inode number. Tránh bị giới hạn số lượng inode khi filesystem rất lớn (mặc dù `/dev/shm` là `tmpfs`).
- `huge=advise`: Liên quan đến Transparent Huge Pages (THP). Chế độ `advise` nghĩa là kernel sẽ cố gắng dùng huge page nếu có thể, nhưng không bắt buộc.
- `usrquota`: Bật user quota (giới hạn dung lượng theo từng user).

Suy ra, đây là cấu hình mount khá an toàn và phổ biến của `/dev/shm` (`tmpfs` trên RAM). Cho phép đọc/ghi , tắt SUID và device file (bảo mật), hỗ trợ inode 64-bit, có bật Transparent Huge Pages ở mức advise, có bật user quota

<sub>--đã hết phần giải thích--</sub>

---

</details>

- **Mối quan hệ giữa `/dev/shm` và POSIX Shared Memory:** Để hiểu rõ vai trò của `/dev/shm`, hãy xét một chương trình C sử dụng `shm_open()`:

  ```c
  int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
  ```

  Ở đây, chương trình yêu cầu Kernel tạo một POSIX Shared Memory Object có tên `/my_shm`. Trên Linux, tên này thường được ánh xạ thành một filesystem object chẳng hạn như `/dev/shm/my_shm`. Có thể quan sát nó với lệnh `ls -l /dev/shm/`, nếu object tồn tại, chúng ta có thể nhìn thấy tên `my_shm` trong danh sách.

- **Tại sao Linux sử dụng tmpfs cho `/dev/shm`?:** Có một lý do quan trọng Shared Memory cần một đối tượng backing để nhiều tiến trình có thể cùng ánh xạ và truy cập dữ liệu. `tmpfs` cung cấp cơ chế filesystem object kết hợp với hệ thống quản lý bộ nhớ ảo của Linux. Một số đặc điểm:
  - Không cần tạo phân vùng ổ đĩa riêng.
  - Hỗ trợ tạo và quản lý filesystem object.
  - Có thể ánh xạ object vào Virtual Address Space thông qua `mmap()`.
  - Dữ liệu có thể được nhiều tiến trình cùng truy cập.
  - Có thể sử dụng swap khi hệ thống cần.

  Tuy nhiên, đừng hiểu rằng toàn bộ dữ liệu trong `/dev/shm` luôn nằm cố định trong RAM. `tmpfs` sử dụng bộ nhớ ảo và có khả năng swap.

<sub>--đã hết phần giải thích--</sub>

---

</td>
</tr>
</table>
</details>

- **b. Quản lý File Descriptor và Open File Description:** Sau khi `shm_open()` thành công, Kernel trả về một File Descriptor cho tiến trình gọi. **Ví dụ:**

  ```c
  int fd = shm_open("/my_shm", O_CREAT | O_RDWR, 0666);
  ```

  Giá trị fd được sử dụng làm chỉ mục trong File Descriptor Table của tiến trình. Ở cấp độ kernel, có thể hình dung mô hình mối quan hệ như sau:

  <p align="center">
	<img src="image/image0.png" alt="mô hình mối quan hệ">
  </p>

  Một số khái niệm cần phân biệt:
  - **File Descriptor:** Số nguyên đại diện cho một mục trong bảng FD của process
  - **struct file:** Biểu diễn một lần mở file ở cấp Kernel
  - **Inode:** Biểu diễn đối tượng filesystem và metadata của nó
  - **Shared Memory Object:** Đối tượng POSIX Shared Memory được truy cập thông qua filesystem

> [!IMPORTANT]
> **Điểm quan trọng:** Hai tiến trình có thể có hai File Descriptor khác nhau nhưng cùng tham chiếu đến một Shared Memory Object.

- **c. Quản lý Virtual Memory Area (VMA):** Khi chương trình gọi `mmap()`, Kernel tiến hành thiết lập một vùng mapping trong Virtual Address Space của tiến trình. **Ví dụ:**

```c
void *ptr = mmap(
    NULL, //cho kernel lựa vaddr
    4096,//4KB
    PROT_READ | PROT_WRITE, //đọc và ghi
    MAP_SHARED, //chia sẻ
    fd, //file descriptor
    0 //offset=0
);
```

Nếu thành công, Kernel tạo hoặc thiết lập một vùng Virtual Memory Area (VMA) tương ứng với mapping đó. Trên Linux, VMA được biểu diễn bởi cấu trúc `struct vm_area_struct`. VMA chứa những thông tin quan trọng như:
- Phạm vi địa chỉ ảo của vùng mapping.
- Quyền truy cập bộ nhớ.
- Các cờ mapping như `VM_SHARED`.
- Thông tin liên kết đến đối tượng file backing mapping.
- Offset tương ứng trong đối tượng được ánh xạ.

**Ví dụ về mặt khái niệm:**

```
Process A
Virtual Address Space

0x7f100000 ┌─────────────────────┐
           │                     │
           │   Shared Memory     │
           │                     │
0x7f101000 └─────────────────────┘
```

Process B có thể ánh xạ cùng object tại một địa chỉ hoàn toàn khác:

```
Process B
Virtual Address Space

0x7f500000 ┌─────────────────────┐
           │                     │
           │   Shared Memory     │
           │                     │
0x7f501000 └─────────────────────┘
```

Hai VMA thuộc hai tiến trình khác nhau, nhưng chúng có thể cùng tham chiếu đến một đối tượng backing memory.

- **d. Quản lý Physical Pages và Page Table:** Đây là một trong những nhiệm vụ quan trọng nhất của Kernel trong cơ chế Shared Memory. Virtual Address mà chương trình sử dụng không phải địa chỉ vật lý thực tế của RAM. CPU cần thông qua Memory Management Unit (MMU) để dịch địa chỉ ảo sang địa chỉ vật lý. Kernel chịu trách nhiệm thiết lập và quản lý các cấu trúc bộ nhớ ảo, bao gồm Page Table và các ánh xạ tương ứng. **Ví dụ, giả sử:**
  - Process A ánh xạ Shared Memory tại `0x7f100000`.
  - Process B ánh xạ Shared Memory tại `0x7f500000`.
  - Cả hai cùng truy cập một Physical Page.

  <p align="center">
	<img src="image/image2.png" alt="mô hình mối quan hệ">
  </p>

  Kết quả:
  - Process A ghi dữ liệu vào vùng mapping.
  - Dữ liệu được ghi vào Physical Page tương ứng.
  - Process B có thể đọc được dữ liệu đó thông qua mapping của chính nó.

  Đây là bản chất của cơ chế chia sẻ bộ nhớ: nhiều Virtual Address thuộc các tiến trình khác nhau có thể cùng ánh xạ đến một Physical Page.

> [!NOTE]
> **Lưu ý:** Việc thiết lập mapping không nhất thiết khiến toàn bộ Physical Pages được cấp phát ngay lập tức. Linux có thể sử dụng Demand Paging để chỉ xử lý trang khi thực sự được truy cập.

<details>
<summary><b>[Câu hỏi]</b> Demand Paging là gì?</summary>
<table>
<tr>
<td>

---



<sub>--đã hết phần giải thích--</sub>

---

</td>
</tr>
</table>
</details>

- **e. Quản lý Backing Storage và Page Cache:** Trên Linux, POSIX Shared Memory thường sử dụng tmpfs làm filesystem backing object. `tmpfs` là một filesystem sử dụng bộ nhớ ảo của Kernel để lưu trữ dữ liệu. Nội dung của nó thường nằm trong RAM, nhưng cũng có thể được swap ra ngoài khi hệ thống cần. Kernel quản lý:
  - Metadata của Shared Memory Object.
  - Các trang dữ liệu backing object.
  - Quan hệ giữa object và những mapping sử dụng object đó.
  - Việc cấp phát, thu hồi và quản lý các trang bộ nhớ.

  **Một điểm cần hiểu rõ:** Không nên đồng nhất `tmpfs` với một vùng RAM cố định. `tmpfs` sử dụng cơ chế quản lý bộ nhớ ảo của Linux, bao gồm cả khả năng sử dụng swap.
- **f. Quản lý vòng đời và giải phóng tài nguyên:** Kernel còn chịu trách nhiệm quản lý vòng đời của Shared Memory Object và các tài nguyên liên quan. **Ví dụ**, khi chương trình thực hiện:

  ```c
  munmap(ptr, 4096);
  close(fd);
  shm_unlink("/my_shm");
  ```

  Mỗi thao tác có ý nghĩa khác nhau:
  - `munmap()`: Hủy mapping trong Virtual Address Space
  - `close()`: Đóng File Descriptor
  - `shm_unlink()`: Xóa tên Shared Memory Object

  Việc gọi một trong các API trên không đồng nghĩa với việc toàn bộ Shared Memory được giải phóng ngay lập tức. Kernel phải xét đến những mapping, file reference và tham chiếu còn tồn tại trước khi có thể thu hồi tài nguyên tương ứng.

## 2.Shared Memory Object

Shared Memory Object (đối tượng bộ nhớ chia sẻ) là một đối tượng được hệ điều hành quản lý, đóng vai trò cung cấp tài nguyên bộ nhớ chung để nhiều tiến trình (process) có thể cùng mở và ánh xạ (mapping) vào Virtual Address Space của chúng.

Trong POSIX Shared Memory, Shared Memory Object là đối tượng trung tâm liên kết các thao tác `shm_open()`, `ftruncate()`, `mmap()` và `shm_unlink()`. Để hiểu rõ bản chất, cần phân biệt rằng Shared Memory Object không phải là một con trỏ bộ nhớ, cũng không phải một vùng Virtual Address cố định. Nó là một đối tượng có danh tính, metadata, kích thước và nội dung dữ liệu được quản lý thông qua cơ chế của hệ điều hành. Trên Linux, POSIX Shared Memory thường được triển khai thông qua filesystem `tmpfs`, với mount point mặc định là `/dev/shm`.

### 2.1. Shared Memory Object là gì?

Shared Memory Object là một đối tượng có tên (named object), được sử dụng làm backing object cho các vùng bộ nhớ được chia sẻ giữa nhiều tiến trình. **Ví dụ**, khi một chương trình gọi:

```c
int fd = shm_open(
    "/my_shm",
    O_CREAT | O_RDWR,
    0666
);
```

Linux sẽ tìm kiếm hoặc tạo một Shared Memory Object có tên `/my_shm`. Trên Linux, object này thường tương ứng với một filesystem object tại `/dev/shm/my_shm`. Sau khi tạo thành công, chương trình nhận được một File Descriptor (fd) để tham chiếu đến object.

Tuy nhiên, tại thời điểm này, cần phân biệt ba vấn đề:
- **Object tồn tại:** Kernel đã có đối tượng Shared Memory tương ứng.
- Object có kích thước: Được thiết lập thông qua `ftruncate()`.
- Object được ánh xạ: Được thực hiện thông qua `mmap()`.

Ba hoạt động này không xảy ra đồng thời và có mục đích khác nhau.

### 5.5. Demand Paging

Demand Paging (phân trang theo nhu cầu) là một cơ chế quản lý bộ nhớ ảo (Virtual Memory Management), trong đó hệ điều hành chỉ đưa một trang bộ nhớ (Memory Page) vào RAM khi tiến trình thực sự cần truy cập đến trang đó, thay vì nạp toàn bộ các trang vào RAM ngay từ đầu.

Nói cách khác, Kernel không nhất thiết phải cấp phát và đưa toàn bộ dữ liệu của một vùng nhớ vào Physical Memory ngay khi chương trình yêu cầu mapping. Thay vào đó, các trang bộ nhớ có thể được xử lý dần theo nhu cầu truy cập thực tế. Demand Paging là một cơ chế quan trọng trong Linux, đặc biệt khi nghiên cứu `mmap()`, POSIX Shared Memory, Page Table và Page Fault.

- **Tại sao cần Demand Paging?:** Giả sử một chương trình yêu cầu ánh xạ một vùng nhớ có kích thước 100 MB:

  ```c
  void *ptr = mmap(
      NULL,
      100 * 1024 * 1024,
      PROT_READ | PROT_WRITE,
      MAP_SHARED,
      fd,
      0
  );
  ```

  Nếu Kernel phải cấp phát toàn bộ 100 MB Physical Memory ngay khi gọi mmap(), hệ thống sẽ phải tiêu tốn tài nguyên RAM ngay cả khi chương trình chỉ sử dụng một phần rất nhỏ vùng nhớ đó. **Ví dụ:**
  - Chương trình yêu cầu mapping: 100 MB.
  - Chương trình thực tế chỉ truy cập: 4 KB.
  - 99 MB còn lại chưa được sử dụng.

  Demand Paging giúp hệ thống tránh việc phải nạp hoặc cấp phát toàn bộ các trang dữ liệu không cần thiết ngay từ đầu.

  **Lưu ý:** `mmap()` thành công không đồng nghĩa với việc toàn bộ Physical Pages đã được cấp phát và đưa vào RAM.