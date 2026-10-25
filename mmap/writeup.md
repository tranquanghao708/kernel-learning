# Kernel: Munmap (mmap)

**mục lục**

<table>
<tr>
<td>

- 1.Tổng quan về `munmap()`
  - 1.1. `munmap()` là gì?
  - 1.2. Mối quan hệ giữa `mmap()` và `munmap()`
  - 1.3. Tại sao cần gỡ bỏ ánh xạ bộ nhớ?
  - 1.4. Phân biệt virtual memory, physical memory và memory mapping
  - 1.5. Khi nào nên sử dụng `munmap()`?

- 2.Cơ chế hoạt động ở User Space
  - 2.1. Khai báo và prototype của `munmap()`
  - 2.2. Các tham số addr và length
  - 2.3. Giá trị trả về và xử lý lỗi
  - 2.4. Yêu cầu căn chỉnh địa chỉ theo page size
  - 2.5. Những trường hợp `munmap()` thất bại
  - 2.6. Ví dụ sử dụng `mmap()` và `munmap()` bằng C

- 3.Từ libc đến Linux Kernel
  - 3.1. Lời gọi hàm `munmap()` trong chương trình C
  - 3.2. System call munmap
  - 3.3. Truyền tham số qua ABI x86-64 Linux
  - 3.4. Cơ chế chuyển từ User Mode sang Kernel Mode
  - 3.5. Kernel tiếp nhận và xử lý system call
  - 3.6. Theo dõi system call bằng strace

- 4.Cấu trúc quản lý bộ nhớ trong Kernel
  - 4.1. Virtual Address Space của một tiến trình
  - 4.2. Memory Region và Virtual Memory Area (VMA)
  - 4.3. Cấu trúc `struct mm_struct`
  - 4.4. Cấu trúc `struct vm_area_struct`
  - 4.5. Quan hệ giữa VMA và Page Table
  - 4.6. Quản lý các VMA trong Linux Kernel

- 5.Cơ chế xử lý `munmap()` trong Kernel
  - 5.1. Điểm bắt đầu của system call
  - 5.2. Hàm xử lý system call `munmap`
  - 5.3. Kiểm tra địa chỉ và phạm vi cần gỡ ánh xạ
  - 5.4. Xác định các VMA bị ảnh hưởng
  - 5.5. Gỡ bỏ toàn bộ một VMA
  - 5.6. Gỡ bỏ một phần VMA
  - 5.7. Chia tách VMA khi chỉ gỡ một vùng ở giữa
  - 5.8. Hợp nhất hoặc cập nhật các VMA còn lại
  - 5.9. Cập nhật cấu trúc quản lý địa chỉ ảo

- 6.Page Table và quá trình gỡ ánh xạ
  - 6.1. Quan hệ giữa VMA và Page Table Entries (PTE)
  - 6.2. Gỡ bỏ các ánh xạ trong Page Table
  - 6.3. Phân biệt gỡ ánh xạ và giải phóng trang vật lý
  - 6.4. Xử lý anonymous memory
  - 6.5. Xử lý file-backed memory
  - 6.6. Xử lý trang đang được chia sẻ
  - 6.7. Tác động đến bộ nhớ vật lý và cơ chế thu hồi trang

</td>
<td>

- 7.TLB và đồng bộ hóa ánh xạ
  - 7.1. TLB lưu trữ thông tin gì?
  - 7.2. Vì sao cần TLB invalidation sau khi gỡ ánh xạ?
  - 7.3. Cơ chế TLB flush trên x86-64
  - 7.4. TLB shootdown khi có nhiều CPU
  - 7.5. Đồng bộ hóa Page Table trong Kernel
  - 7.6. Quan hệ giữa TLB, Page Table và tính nhất quán bộ nhớ

- 8.Quan hệ giữa `munmap()` và các cơ chế quản lý bộ nhớ khác
  - 8.1. `mmap()` và `munmap()`
  - 8.2. `brk()` và `sbrk()`
  - 8.3. `mprotect()` và thay đổi quyền truy cập
  - 8.4. `madvise()` và gợi ý sử dụng bộ nhớ
  - 8.5. `msync()` và đồng bộ dữ liệu với file
  - 8.6. `free()` so với `munmap()`
  - 8.7. Vì sao `free()` không nhất thiết gọi `munmap()`?

- 9.Phân tích mã nguồn Linux Kernel
  - 9.1. Xác định phiên bản kernel được phân tích
  - 9.2. Tìm implementation của system call `munmap`
  - 9.3. Lần theo call graph từ system call đến các hàm xử lý VMA
  - 9.4. Phân tích các hàm gỡ ánh xạ Page Table
  - 9.5. Phân tích các hàm cập nhật VMA
  - 9.6. Phân tích xử lý TLB và giải phóng trang
  - 9.7. Đối chiếu mã nguồn với tài liệu Linux Kernel
  - 9.8. Tổng hợp call graph và luồng thực thi

- 10.Thực nghiệm bằng C và Linux Tools
  - 10.1. Tạo anonymous mapping bằng `mmap()`
  - 10.2. Kiểm tra vùng nhớ trong `/proc/<pid>/maps`
  - 10.3. Gỡ bỏ toàn bộ mapping bằng `munmap()`
  - 10.4. Gỡ bỏ một phần mapping
  - 10.5. Quan sát VMA trước và sau khi gọi `munmap()`
  - 10.6. Theo dõi system call bằng strace
  - 10.7. Debug chương trình bằng GDB
  - 10.8. Quan sát lỗi khi truy cập vùng nhớ đã bị unmap
  - 10.9. Kiểm tra hành vi với anonymous và file-backed mapping

- 11.Các trường hợp đặc biệt và lỗi thường gặp
  - 11.1. Địa chỉ không được căn chỉnh theo page size
  - 11.2. Độ dài bằng 0
  - 11.3. Vùng nhớ không được ánh xạ
  - 11.4. Gỡ bỏ một phần vùng nhớ
  - 11.5. Truy cập con trỏ sau khi `munmap()`
  - 11.6. Nhiều luồng cùng truy cập một vùng nhớ
  - 11.7. Giới hạn số lượng VMA và lỗi cấp phát cấu trúc quản lý

- 12.Tổng kết
  - 12.1. Tóm tắt luồng xử lý từ User Space đến Kernel
  - 12.2. Sơ đồ vòng đời của một memory mapping
  - 12.3. Phân biệt gỡ ánh xạ, giải phóng trang và thu hồi bộ nhớ
  - 12.4. Những điểm quan trọng cần ghi nhớ
  - 12.5. Tài liệu tham khảo

</td>
</tr>
</table>