# Kernel : Đa luồng (pthread)

**mục lục**

- [1.Đa luồng là gì?](#1đa-luồng-là-gì)

	- [1.1.Process và Thread](#11process-và-thread)

	- [1.2.Một process có thể có nhiều thread](#12một-process-có-thể-có-nhiều-thread)

	- [1.3.Thread dùng chung và riêng những gì?](#13thread-dùng-chung-và-riêng-những-gì)

	- 1.4.Thread có thực sự chạy song song không?

	- 1.5.Logical CPU và Physical CPU

- 2.pthread là gì?

	- 2.1.POSIX Threads

	- 2.2.Thư viện pthread.h

	- 2.3.Biên dịch chương trình pthread

- 3.Thread được tạo như thế nào?

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

### 1.1.Process và Thread

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

### 1.2.Một process có thể có nhiều thread

Một chương trình đơn luồng sẽ có một thread, và thread này làm việc với nhiều tasks chẳng hạn theo sơ đồ:

```
Process 
| 
|── Main Thread 
			| 
			|── task A 
			|── task B 
			|── task C 
			|── task D
```

với chương trình này thì sẽ dùng một lõi CPU trong tất cả lõi, vì thế ta luôn thấy biểu đồ đo CPU thường thấy trên linux luôn hiện một ngưỡng như 20-25% thay vì full 100%, nhưng khi vào htop ta thấy program đó lại dùng full 100% CPU (thực chất chúng dùng hết công suất của một luồng trong tất cả luồng hiện có, chứ ko phải dùng hết tất cả luồng trong CPU vật lý hiện có) ta nên phân biệt rõ điều này

Nếu chương trình được cấp thêm các thread vào như :

```
Process 
| 
|── Main Thread 
| 
|── Thread 1 
|
|── Thread 2 
| 
|── Thread 3
```

thì hệ điều hành có thể lập lịch các thread này lên các CPU logical khác nhau. Ví dụ:

```
CPU 0 <-> Thread 0
CPU 1 <-> Thread 1
CPU 2 <-> Thread 2
CPU 3 <-> Thread 3
```

Nếu phần cứng có đủ execution resources, nhiều thread có thể thực sự chạy đồng thời.

### 1.3.Thread dùng chung và riêng những gì?

Các thread trong cùng process dùng chung address space. Khái niệm dùng chung và dùng riêng trong trường hợp này là nếu mà chung thì một đoạn code hay cái gì của chương trình nhưng tất cả thread cùng dùng nó thì gọi là dùng chung, còn dùng riêng thì mỗi thread đều có một cái riêng để dùng

<table>

<details>
	<summary><b>[Chi tiết]</b> Address space là gì?</summary>

---

<sub>--đã hết phần giải thích--</sub>

---

</details>

</table>

Ví dụ:

```
Process
|
|── Code       <- shared
|── Global     <- shared
|── Heap       <- shared
|
|── Thread 0
|    |── Stack <- riêng
|
|── Thread 1
|    |── Stack <- riêng
|
|── Thread 2
     |── Stack <- riêng
```

Trong đó dùng chung. Các thread có thể truy cập: global variables, static variables, heap, vùng memory được cấp phát bởi malloc, file descriptors, code của process

Còn dùng riêng. Mỗi thread có trạng thái thực thi riêng: registers, instruction pointer, stack, thread-local storage (TLS). **Ví dụ:**

```c
void *worker(void *arg)
{
    int x = 10;

    return NULL;
}
```

`x` nằm trên stack của thread đang thực hiện `worker()`.Thread khác không tự động có cùng biến `x`.