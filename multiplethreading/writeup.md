# Kernel : Đa luồng (pthread)

**mục lục**

- [1.Đa luồng là gì?](#1đa-luồng-là-gì)

	- [1.1.Process và Thread](#11process-và-thread)

	- 1.2.Một process có thể có nhiều thread

	- 1.3.Thread dùng chung và riêng những gì?

	- 1.4.Thread có thực sự chạy song song không?

	- 1.5.Logical CPU và Physical CPU

- 2.pthread là gì?

	- 2.1.POSIX Threads

	- 2.2.Thư viện pthread.h

	- 2.3.Biên dịch chương trình pthread

- 3. Thread được tạo như thế nào?

	- 3.1.Hàm pthread_create()

	- 3.2.Hàm worker

	- 3.3.void *arg dùng để làm gì?

	- 3.4.Truyền struct vào thread

	- 3.5.Mỗi thread có một task riêng

- 4.Thread kết thúc như thế nào?

	- 4.1.return NULL

	- 4.2.pthread_join()

	- 4.3.Tại sao phải join?

- 5.Bộ nhớ của các thread

	- 5.1.Stack của thread

	- 5.2.Heap và vùng nhớ dùng chung

	- 5.3.Biến local và biến shared

	- 5.4.Data race

	- 5.5.Race condition

- 6.Synchronization

	- 6.1.Tại sao cần synchronization?

	- 6.2.Mutex

	- 6.3.Atomic

	- 6.4.Khi nào dùng mutex và khi nào dùng atomic?

- 7.Chia công việc cho nhiều thread

	- 7.1.Task độc lập

	- 7.2.Bài toán có dependency

	- 7.3.Không phải cứ chia vòng lặp là parallel được

	- 7.4.Ví dụ chia 4 task cho 4 thread

- 8.Đo hiệu năng đa luồng

	- 8.1. 1 thread

	- 8.2. 2 threads

	- 8.3. 3 threads

	- 8.4. 4 threads

	- 8.5.Speedup

	- 8.6.Vì sao 4 thread không nhất thiết nhanh gấp 4 lần?

- 9.Debug pthread bằng GDB

	- 9.1.Thread ID

	- 9.2.info threads

	- 9.3.Chuyển giữa các thread

	- 9.4.Breakpoint trong worker

- 10.Những lỗi thường gặp

	- 10.1.Truyền sai địa chỉ vào thread

	- 10.2.Dùng biến local sau khi hết lifetime

	- 10.3.Data race

	- 10.4.Quên pthread_join()

	- 10.5.Tạo quá nhiều thread

- 11.Kết luận

---

## 1.Đa luồng là gì?

### 1.1. Process và Thread

Khi một chương trình được chạy, hệ điều hành tạo ra một process đại diện cho instance đang thực thi của chương trình. **Ví dụ:**

```bash
./calculator
```

Khi chạy lệnh này, hệ điều hành tạo một process cho calculator. Process có nhiều tài nguyên chẳng hạn như (virtual address space, code, global/static data, heap, file descriptors, các thông tin quản lý của kernel, một hoặc nhiều thread thực thi...). Nghĩa là, khi một chương trình được chạy thì hệ điều hành sẽ tạo một tiến trình, cấu trúc tiến trình cụ thể sẽ giống như :

```
Process 
| 
|── Code 
|── Global / Static data 
|── Heap 
|── File descriptors 
| 
|── Thread 
		|── Instruction pointer 
		|── Registers 
		|── Stack
```

Trong đó ta thấy phần thread là một luồng thực thi bên trong process. Một process tối thiểu phải có một thread để thực thi chương trình. Thread đầu tiên thường được gọi là main thread. Khi sử dụng pthread, main thread có thể tạo thêm các thread khác.