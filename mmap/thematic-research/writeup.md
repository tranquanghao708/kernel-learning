# Kernel : Chuyên đề MAP_SHARED (MAP_SHARED Thematic Research)

**mục lục**

<table>
<tr>
<td valign="top" width="50%">

- 1.Tổng quan về POSIX Shared Memory
  - 1.1. Shared Memory là gì?
  - 1.2. Shared Memory trong IPC
  - 1.3. So sánh Shared Memory với Pipe, FIFO và Socket
  - 1.4. POSIX Shared Memory và System V Shared Memory
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
- 8.Synchronization
  - 8.1. Race Condition
  - 8.2. POSIX Semaphore
  - 8.3. Process-shared Mutex
  - 8.4. Memory Ordering
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
</talble>

---