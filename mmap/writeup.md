# Kernel: mmap() & munmap()

**Mục lục**

<table>
<tr>
<td valign="top" width="50%">

- [1. Tổng quan về Memory Mapping](#1-tổng-quan-về-memory-mapping)
  - [1.1. Memory Mapping là gì?](#11-memory-mapping-là-gì)
	   - [1.1.1. Virtual Address và physical address](#111-virtual-address-và-physical-address)
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

Memory Mapping (ánh xạ bộ nhớ) là cơ chế thiết lập mối quan hệ giữa một vùng địa chỉ ảo trong không gian địa chỉ của tiến trình với một đối tượng hoặc tài nguyên mà tiến trình có thể truy cập thông qua vùng địa chỉ đó. Trên Linux, `mmap()` là một system-call interface cho phép chương trình yêu cầu kernel thiết lập một vùng ánh xạ như vậy.

Để hiểu bản chất của Memory Mapping, trước tiên ta cần phân biệt ba khái niệm: Virtual Address, Physical Address và Memory Mapping.

#### 1.1.1. Virtual Address và physical address

Địa chỉ ảo (virtual address - vaddr) và địa chỉ thật (physical address - paddr) hay còn được gọi là địa chỉ vật lý, là hai phần địa chỉ mà hệ thống sử dụng, trong đó địa chỉ ảo dùng để sử dụng cho các tiến trình, bộ nhớ (stack/heap), application, v.v.. còn địa chỉ thật được sử dụng bởi ram vật lý, chúng tham chiếu địa chỉ thật và địa chỉ ảo bằng bảng trang (page table), khái niệm bảng trang có tại chương [7. Page Table, Physical Memory và TLB](#7-page-table-physical-memory-và-tlb)

### 1.2. `mmap()` là gì?

### 1.3. `munmap()` là gì?

### 1.4. Mối quan hệ giữa `mmap()` và `munmap()`

### 1.5. Virtual Memory và Physical Memory

### 1.6. Vòng đời của một Memory Mapping