# Kernel: mmap() & munmap()

**Mục lục**

<table>
<tr>
<td valign="top" width="50%">

- [1. Tổng quan về Memory Mapping](#1-tổng-quan-về-memory-mapping)
  - [1.1. Memory Mapping là gì?](#11-memory-mapping-là-gì)
	   - [1.1.1. Virtual Address và physical address](#111-virtual-address-và-physical-address)
	   - [1.1.3. Mối quan hệ giữa phần mềm và địa chỉ bộ nhớ](#113-mối-quan-hệ-giữa-phần-mềm-và-địa-chỉ-bộ-nhớ)
  - [1.2. `mmap()` là gì?](#12-mmap-là-gì)
  - [1.3. `munmap()` là gì?](#13-munmap-là-gì)
  - [1.4. Mối quan hệ giữa `mmap()` và `munmap()`](#14-mối-quan-hệ-giữa-mmap-và-munmap)
  - [1.5. Virtual Memory và Physical Memory](#15-virtual-memory-và-physical-memory)
  - [1.6. Vòng đời của một Memory Mapping](#16-vòng-đời-của-một-memory-mapping)

- [2. Cơ chế hoạt động ở User Space](#2-cơ-chế-hoạt-động-ở-user-space)
  - [2.1. Khai báo và Prototype của `mmap()`](#21-khai-báo-và-prototype-của-mmap)
  - [2.2. Các tham số của `mmap()`](#22-các-tham-số-của-mmap)
  - [2.3. `addr` và `length`](#23-addr-và-length)
  - [2.4. `prot` và quyền truy cập bộ nhớ](#24-prot-và-quyền-truy-cập-bộ-nhớ)
  - [2.5. `flags`: `MAP_PRIVATE`, `MAP_SHARED` và `MAP_ANONYMOUS`](#25-flags-map_private-map_shared-và-map_anonymous)
  - [2.6. `fd` và `offset`](#26-fd-và-offset)
  - [2.7. Giá trị trả về và xử lý lỗi](#27-giá-trị-trả-về-và-xử-lý-lỗi)
  - [2.8. Khai báo và Prototype của `munmap()`](#28-khai-báo-và-prototype-của-munmap)
  - [2.9. Các tham số `addr` và `length` của `munmap()`](#29-các-tham-số-addr-và-length-của-munmap)
  - [2.10. Giá trị trả về và xử lý lỗi của `munmap()`](#210-giá-trị-trả-về-và-xử-lý-lỗi-của-munmap)

- [3. Từ libc đến Linux Kernel](#3-từ-libc-đến-linux-kernel)
  - [3.1. Lời gọi hàm trong chương trình C](#31-lời-gọi-hàm-trong-chương-trình-c)
  - [3.2. System call `mmap`](#32-system-call-mmap)
  - [3.3. System call `munmap`](#33-system-call-munmap)
  - [3.4. Truyền tham số qua System V AMD64 ABI](#34-truyền-tham-số-qua-system-v-amd64-abi)
  - [3.5. Chuyển từ User Mode sang Kernel Mode](#35-chuyển-từ-user-mode-sang-kernel-mode)
  - [3.6. Theo dõi system call bằng `strace`](#36-theo-dõi-system-call-bằng-strace)

- [4. Cấu trúc quản lý bộ nhớ trong Kernel](#4-cấu-trúc-quản-lý-bộ-nhớ-trong-kernel)
  - [4.1. Virtual Address Space của một tiến trình](#41-virtual-address-space-của-một-tiến-trình)
  - [4.2. Memory Region và Virtual Memory Area (VMA)](#42-memory-region-và-virtual-memory-area-vma)
  - [4.3. `struct mm_struct`](#43-struct-mm_struct)
  - [4.4. `struct vm_area_struct`](#44-struct-vm_area_struct)
  - [4.5. Quan hệ giữa VMA và Page Table](#45-quan-hệ-giữa-vma-và-page-table)
  - [4.6. Quản lý các VMA trong Linux Kernel](#46-quản-lý-các-vma-trong-linux-kernel)

- [5. Cơ chế xử lý `mmap()` trong Kernel](#5-cơ-chế-xử-lý-mmap-trong-kernel)
  - [5.1. Điểm bắt đầu của system call](#51-điểm-bắt-đầu-của-system-call)
  - [5.2. Kiểm tra tham số và cờ ánh xạ](#52-kiểm-tra-tham-số-và-cờ-ánh-xạ)
  - [5.3. Tìm kiếm vùng địa chỉ ảo phù hợp](#53-tìm-kiếm-vùng-địa-chỉ-ảo-phù-hợp)
  - [5.4. Tạo và thiết lập VMA](#54-tạo-và-thiết-lập-vma)
  - [5.5. Ánh xạ Anonymous Memory](#55-ánh-xạ-anonymous-memory)
  - [5.6. Ánh xạ File-backed Memory](#56-ánh-xạ-file-backed-memory)
  - [5.7. Thiết lập quyền truy cập bộ nhớ](#57-thiết-lập-quyền-truy-cập-bộ-nhớ)
  - [5.8. Trả địa chỉ ánh xạ về User Space](#58-trả-địa-chỉ-ánh-xạ-về-user-space)

- [6. Cơ chế xử lý `munmap()` trong Kernel](#6-cơ-chế-xử-lý-munmap-trong-kernel)
  - [6.1. Điểm bắt đầu của system call](#61-điểm-bắt-đầu-của-system-call)
  - [6.2. Kiểm tra địa chỉ và phạm vi cần gỡ ánh xạ](#62-kiểm-tra-địa-chỉ-và-phạm-vi-cần-gỡ-ánh-xạ)
  - [6.3. Xác định các VMA bị ảnh hưởng](#63-xác-định-các-vma-bị-ảnh-hưởng)
  - [6.4. Gỡ bỏ toàn bộ một VMA](#64-gỡ-bỏ-toàn-bộ-một-vma)
  - [6.5. Gỡ bỏ một phần VMA](#65-gỡ-bỏ-một-phần-vma)
  - [6.6. Chia tách VMA khi gỡ ánh xạ ở giữa](#66-chia-tách-vma-khi-gỡ-ánh-xạ-ở-giữa)
  - [6.7. Cập nhật cấu trúc quản lý địa chỉ ảo](#67-cập-nhật-cấu-trúc-quản-lý-địa-chỉ-ảo)

</td>
<td valign="top" width="50%">

- [7. Page Table, Physical Memory và TLB](#7-page-table-physical-memory-và-tlb)
  - [7.1. Virtual Page và Physical Frame](#71-virtual-page-và-physical-frame)
  - [7.2. Page Table Entries (PTE)](#72-page-table-entries-pte)
  - [7.3. Page Fault và Demand Paging](#73-page-fault-và-demand-paging)
  - [7.4. Gỡ bỏ ánh xạ trong Page Table](#74-gỡ-bỏ-ánh-xạ-trong-page-table)
  - [7.5. Anonymous Memory và File-backed Memory](#75-anonymous-memory-và-file-backed-memory)
  - [7.6. Gỡ ánh xạ và giải phóng trang vật lý](#76-gỡ-ánh-xạ-và-giải-phóng-trang-vật-lý)
  - [7.7. TLB và cơ chế Address Translation](#77-tlb-và-cơ-chế-address-translation)
  - [7.8. TLB Invalidation và TLB Shootdown](#78-tlb-invalidation-và-tlb-shootdown)

- [8. Memory Mapping trong thực tế](#8-memory-mapping-trong-thực-tế)
  - [8.1. Anonymous Mapping](#81-anonymous-mapping)
  - [8.2. File-backed Mapping](#82-file-backed-mapping)
  - [8.3. `MAP_PRIVATE` và Copy-on-Write](#83-map_private-và-copy-on-write)
  - [8.4. `MAP_SHARED` và bộ nhớ chia sẻ](#84-map_shared-và-bộ-nhớ-chia-sẻ)
  - [8.5. Memory Mapping và ELF Loader](#85-memory-mapping-và-elf-loader)
  - [8.6. Memory Mapping và Dynamic Linker](#86-memory-mapping-và-dynamic-linker)
  - [8.7. Memory Mapping và cấp phát bộ nhớ động](#87-memory-mapping-và-cấp-phát-bộ-nhớ-động)

- [9. Phân tích mã nguồn Linux Kernel](#9-phân-tích-mã-nguồn-linux-kernel)
  - [9.1. Xác định phiên bản Kernel](#91-xác-định-phiên-bản-kernel)
  - [9.2. Tìm implementation của `mmap()`](#92-tìm-implementation-của-mmap)
  - [9.3. Tìm implementation của `munmap()`](#93-tìm-implementation-của-munmap)
  - [9.4. Truy vết Call Graph](#94-truy-vết-call-graph)
  - [9.5. Phân tích quá trình tạo VMA](#95-phân-tích-quá-trình-tạo-vma)
  - [9.6. Phân tích quá trình cập nhật và xóa VMA](#96-phân-tích-quá-trình-cập-nhật-và-xóa-vma)
  - [9.7. Phân tích xử lý Page Table và TLB](#97-phân-tích-xử-lý-page-table-và-tlb)
  - [9.8. Đối chiếu mã nguồn với thực nghiệm](#98-đối-chiếu-mã-nguồn-với-thực-nghiệm)

- [10. Thực nghiệm bằng C và Linux Tools](#10-thực-nghiệm-bằng-c-và-linux-tools)
  - [10.1. Tạo Anonymous Mapping bằng `mmap()`](#101-tạo-anonymous-mapping-bằng-mmap)
  - [10.2. Ghi và đọc dữ liệu trong vùng ánh xạ](#102-ghi-và-đọc-dữ-liệu-trong-vùng-ánh-xạ)
  - [10.3. Thực thi Machine Code trong vùng nhớ ánh xạ](#103-thực-thi-machine-code-trong-vùng-nhớ-ánh-xạ)
  - [10.4. Quan sát `/proc/<pid>/maps`](#104-quan-sát-procpidmaps)
  - [10.5. Gỡ bỏ toàn bộ Mapping bằng `munmap()`](#105-gỡ-bỏ-toàn-bộ-mapping-bằng-munmap)
  - [10.6. Gỡ bỏ một phần Mapping](#106-gỡ-bỏ-một-phần-mapping)
  - [10.7. Theo dõi bằng `strace`](#107-theo-dõi-bằng-strace)
  - [10.8. Debug bằng GDB](#108-debug-bằng-gdb)
  - [10.9. Kiểm tra hành vi bằng `/proc/<pid>/smaps`](#109-kiểm-tra-hành-vi-bằng-procpidsmaps)

- [11. Lỗi và các trường hợp đặc biệt](#11-lỗi-và-các-trường-hợp-đặc-biệt)
  - [11.1. `MAP_FAILED` và xử lý lỗi](#111-map_failed-và-xử-lý-lỗi)
  - [11.2. Địa chỉ không căn chỉnh theo Page Size](#112-địa-chỉ-không-căn-chỉnh-theo-page-size)
  - [11.3. `length` bằng 0](#113-length-bằng-0)
  - [11.4. Gỡ bỏ một vùng chưa được ánh xạ](#114-gỡ-bỏ-một-vùng-chưa-được-ánh-xạ)
  - [11.5. Truy cập vùng nhớ sau `munmap()`](#115-truy-cập-vùng-nhớ-sau-munmap)
  - [11.6. Lỗi quyền truy cập bộ nhớ](#116-lỗi-quyền-truy-cập-bộ-nhớ)
  - [11.7. Đồng bộ hóa khi nhiều luồng sử dụng Mapping](#117-đồng-bộ-hóa-khi-nhiều-luồng-sử-dụng-mapping)

- [12. Tổng kết và tài liệu tham khảo](#12-tổng-kết-và-tài-liệu-tham-khảo)
  - [12.1. Tóm tắt cơ chế `mmap()`](#121-tóm-tắt-cơ-chế-mmap)
  - [12.2. Tóm tắt cơ chế `munmap()`](#122-tóm-tắt-cơ-chế-munmap)
  - [12.3. Phân biệt VMA, Page Table và Physical Memory](#123-phân-biệt-vma-page-table-và-physical-memory)
  - [12.4. Tổng hợp vòng đời Memory Mapping](#124-tổng-hợp-vòng-đời-memory-mapping)
  - [12.5. Tài liệu tham khảo](#125-tài-liệu-tham-khảo)

</td>
</tr>
</table>

---

## 1. Tổng quan về Memory Mapping

### 1.1. Memory Mapping là gì?

Memory Mapping (ánh xạ bộ nhớ) là cơ chế thiết lập mối quan hệ giữa một vùng địa chỉ ảo trong không gian địa chỉ của tiến trình với bộ nhớ hoặc một đối tượng được ánh xạ, chẳng hạn như một file. Nhờ cơ chế này, tiến trình có thể truy cập dữ liệu thông qua địa chỉ ảo mà không cần trực tiếp quản lý địa chỉ vật lý của RAM.

Trên Linux, `mmap()` là một system-call interface cho phép chương trình yêu cầu kernel thiết lập một vùng ánh xạ trong không gian địa chỉ ảo của tiến trình. Ngược lại, `munmap()` cho phép gỡ bỏ ánh xạ trong một phạm vi địa chỉ được chỉ định.

Để hiểu bản chất của Memory Mapping, trước tiên ta cần phân biệt ba khái niệm: Virtual Address, Physical Address và Memory Mapping.

#### 1.1.1. Virtual Address và Physical Address

Địa chỉ ảo (Virtual Address - VA) và địa chỉ vật lý (Physical Address - PA) là hai loại địa chỉ được sử dụng trong cơ chế quản lý bộ nhớ của hệ thống máy tính.

**Virtual Address (VA)** là địa chỉ thuộc không gian địa chỉ ảo mà tiến trình sử dụng để truy cập bộ nhớ. Trên hệ điều hành hiện đại như Linux, mỗi tiến trình thông thường có không gian địa chỉ ảo riêng. Địa chỉ này được sử dụng để truy cập các vùng bộ nhớ như Stack, Heap, vùng dữ liệu và các vùng ánh xạ khác.

**Physical Address (PA)** là địa chỉ thuộc không gian địa chỉ vật lý. Khi địa chỉ ảo được dịch thành công sang địa chỉ vật lý tương ứng, phần cứng có thể sử dụng địa chỉ vật lý đó để truy cập bộ nhớ vật lý, chẳng hạn như RAM.

Trên hệ thống sử dụng cơ chế phân trang (Paging), CPU phối hợp với bộ phận quản lý bộ nhớ (Memory Management Unit - MMU) để dịch địa chỉ ảo sang địa chỉ vật lý thông qua các cấu trúc Page Table.

Có thể hình dung quá trình này như sau:

```text
             User Space
                 |
                 v
         Virtual Address
           0x7f001000
                 |
                 v
          +-------------+
          |     MMU     |
          |  Address    |
          | Translation |
          +-------------+
                 |
                 | Tra cứu Page Table
                 v
        Physical Address
           0x12345000
                 |
                 v
                RAM
```

Đây là sơ đồ minh họa đơn giản. Các địa chỉ trên chỉ mang tính ví dụ, không đại diện cho một ánh xạ thực tế cụ thể.

> [!NOTE]
> **Điểm cần nhớ:** Chương trình thông thường sử dụng địa chỉ ảo. MMU và Page Table phối hợp để thực hiện quá trình dịch địa chỉ khi CPU truy cập bộ nhớ. Chương trình không cần trực tiếp biết địa chỉ vật lý tương ứng với địa chỉ ảo của mình.

**Cho ví dụ**, khi chương trình C khai báo một biến:

```c
int number = 100;
```

Nếu biến được lưu trong bộ nhớ, địa chỉ lấy được thông qua `&number` trong chương trình thông thường là địa chỉ ảo thuộc không gian địa chỉ của tiến trình.

```c
printf("%p\n", (void *)&number);
```

Giá trị được in ra không phải địa chỉ vật lý của ô RAM đang chứa biến `number`. Nó là địa chỉ mà chương trình sử dụng để tham chiếu đến biến đó.

Cần lưu ý rằng trình biên dịch có thể tối ưu biến vào thanh ghi hoặc loại bỏ việc lưu biến trong bộ nhớ nếu không cần thiết. Vì vậy, ví dụ trên chỉ minh họa trường hợp biến thực sự được lưu trong bộ nhớ.

#### 1.1.2. Phân biệt Memory Mapping, Virtual Address và Physical Address

Dựa trên các khái niệm đã trình bày, ta có thể phân biệt chúng như sau:

<div align="center">

| Khái niệm | Bản chất | Vai trò |
|---|---|---|
| Memory Mapping | Cơ chế thiết lập quan hệ ánh xạ giữa một vùng địa chỉ ảo với bộ nhớ hoặc đối tượng được ánh xạ | Cho phép tiến trình truy cập bộ nhớ hoặc dữ liệu thông qua địa chỉ ảo |
| Virtual Address (VA) | Địa chỉ thuộc không gian địa chỉ ảo của tiến trình | Được chương trình sử dụng để tham chiếu đến dữ liệu hoặc mã lệnh |
| Physical Address (PA) | Địa chỉ thuộc không gian địa chỉ vật lý | Được sử dụng trong quá trình truy cập bộ nhớ vật lý sau khi địa chỉ được dịch thành công |

</div>

Memory Mapping và Virtual Address không phải hai cách gọi của cùng một khái niệm. Virtual Address là một loại địa chỉ, trong khi Memory Mapping là cơ chế thiết lập và quản lý quan hệ ánh xạ.

Tương tự, Physical Address không phải bản thân RAM. Địa chỉ vật lý là địa chỉ trong không gian địa chỉ vật lý; RAM là một loại bộ nhớ vật lý mà hệ thống có thể truy cập thông qua cơ chế địa chỉ này.

Trong hệ thống phân trang, quan hệ giữa địa chỉ ảo và địa chỉ vật lý được quản lý thông qua Page Table. Tuy nhiên, không phải mọi địa chỉ ảo đều luôn có một trang RAM vật lý tương ứng tại mọi thời điểm. Một vùng địa chỉ ảo có thể tồn tại mà các trang của nó chưa được cấp phát vật lý hoặc chưa có ánh xạ vật lý hiện hữu.

#### 1.1.3. Mối quan hệ giữa phần mềm và địa chỉ bộ nhớ

Trên các hệ điều hành hiện đại sử dụng bộ nhớ ảo, chương trình thông thường hoạt động dựa trên không gian địa chỉ ảo do hệ điều hành cung cấp. Điều này có nghĩa là chương trình không cần quản lý trực tiếp vị trí vật lý của dữ liệu trong RAM. Khi chương trình truy cập một địa chỉ ảo, phần cứng và hệ điều hành phối hợp để xử lý việc dịch địa chỉ, quản lý quyền truy cập và xử lý các trường hợp cần thiết như Page Fault.

Vì vậy, không nên nói rằng phần mềm luôn nghĩ địa chỉ ảo chính là địa chỉ vật lý. Chính xác hơn, chương trình thông thường chỉ cần làm việc với địa chỉ trong không gian địa chỉ mà nó được cấp; việc ánh xạ sang địa chỉ vật lý được hệ thống quản lý phía dưới.Đây là một trong những nền tảng quan trọng của cơ chế Memory Mapping trên Linux.

### 1.2. `mmap()` là gì?

`mmap()` (memory mapping) là một API trên Linux cho phép chương trình yêu cầu kernel thiết lập một vùng ánh xạ trong không gian địa chỉ ảo của tiến trình.

Thông qua `mmap()`, chương trình có thể tạo vùng bộ nhớ ẩn danh (anonymous mapping), ánh xạ nội dung của file vào bộ nhớ, hoặc thiết lập vùng bộ nhớ phục vụ những mục đích khác.

> [!IMPORTANT]
> **Điểm quan trọng:** `mmap()` không đơn thuần là hàm cấp phát RAM. Nó yêu cầu kernel thiết lập một vùng địa chỉ ảo và các thông tin quản lý liên quan. Các trang vật lý có thể chỉ được cấp phát hoặc ánh xạ khi chương trình thực sự truy cập chúng.

### 1.3. `munmap()` là gì?

### 1.4. Mối quan hệ giữa `mmap()` và `munmap()`

### 1.5. Virtual Memory và Physical Memory

### 1.6. Vòng đời của một Memory Mapping