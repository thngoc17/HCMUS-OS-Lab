# **ĐỒ ÁN 2 HỆ ĐIỀU HÀNH**
## **Lab 2: Page tables**
## **A. Chi tiết nhóm**
## **B. Chi tiết đồ án**
### **1. In một bảng trang**
#### **1.1 Mô tả**
* xv6 có cách quản lí bảng trang riêng, được gói gọn trong source code. 
* Để trực quan hóa bảng trang của tập lệnh RISC-V, nhóm cần tạo một hàm để in ra các nội dung của một bảng trang.

#### **1.2 Nhắc lại kiến thức**
* xv6 là hệ điều hành hoạt động trên kiến trúc RISC-V.
* Cơ chế phân trang của RISC-V (cụ thể là Sv39) sử dụng mô hình **bảng trang 3 cấp (3-level page table)**.
* Địa chỉ ảo được dịch sang địa chỉ vật lý bằng cách đi qua 3 tầng: Level 2 (cao nhất) $\rightarrow$ Level 1 $\rightarrow$ Level 0 (lá).
* Mỗi bảng trang chứa **512 mục (PTE - Page Table Entry)**.
* Mỗi PTE chứa địa chỉ vật lý (Physical Address) và các cờ (flags). Trong đó, quan trọng nhất là cờ `PTE_V` (Valid), nếu bit này bật nghĩa là trang đó hợp lệ và có thể truy cập được.

#### **1.3 Cài đặt**
> Ở đồ án này, nhóm không cần phải tạo ra một system call riêng cho chức năng in bảng trang, do đó mọi công việc ở phần này sẽ được gói gọn trong không gian kernel. 
> Từ mô tả đồ án và hướng dẫn chi tiết, cần có các nhiệm vụ sau: 
> * Định nghĩa hàm `vmprint()` trong `kernel/vm.c`.
> * Gọi hàm `vmprint()` ngay ở `int exec()` trong `kernel/exec.c`.
> * Khai báo prototype cho hàm `vmprint()` trong `kernel/defs.h` để hàm gọi được trong `kernel/exec.c`.

#### **1.3.1 Trong kernel/vm.c**
Cài đặt hàm in bảng trang `vmprint`.

**Giải thích code:**

* Ý tưởng của nhóm ở đây khá đơn giản: Vì bảng trang có 3 cấp cố định nên nhóm dùng 3 vòng lặp lồng nhau để duyệt lần lượt từ trên xuống dưới (thay vì dùng đệ quy).
* Đầu tiên in ra địa chỉ gốc của bảng trang.
* **Vòng lặp i (Level 2):** Duyệt qua 512 mục của bảng trang gốc.
* Quan trọng nhất là đoạn `if(pte_l2 & PTE_V)`: Nhóm chỉ quan tâm đến những trang nào hợp lệ (có bit Valid). Nếu không hợp lệ thì bỏ qua.
* Sau khi tìm thấy PTE hợp lệ, nhóm dùng macro `PTE2PA` để lấy địa chỉ vật lý của bảng trang cấp tiếp theo (Level 1) và ép kiểu về `pagetable_t` để duyệt tiếp.


* **Vòng lặp j (Level 1) và k (Level 0):** Logic giống như trên. Mỗi lần tìm thấy PTE hợp lệ thì in ra theo đúng format dấu chấm (`..`) mà đề bài yêu cầu.
* Các dòng `printf` nhóm ép kiểu về `(void *)` để in địa chỉ dạng hexa cho chuẩn format `%p`.

#### **1.3.2 Trong kernel/defs.h**

Nhóm cần thêm khai báo hàm vào file này để các file khác trong kernel (cụ thể là `exec.c`) có thể gọi được nó. Mình thêm vào phần `// vm.c`:

```c
// vm.c
void            vmprint(pagetable_t); // <--- Dòng nhóm thêm vào

```

#### **1.3.3 Trong kernel/exec.c**

Đề bài yêu cầu in bảng trang khi tiến trình đầu tiên (`pid == 1`) được khởi tạo. Hàm `exec()` là nơi hệ điều hành nạp chương trình vào bộ nhớ, nên mình chèn lệnh gọi `vmprint` vào cuối hàm này, ngay trước khi return.

```c
int
exec(char *path, char **argv)
{
  // ... (code có sẵn của xv6: load segment, cấp phát stack, v.v...) ...

  // Commit to the user image.
  oldpagetable = p->pagetable;
  p->pagetable = pagetable;
  p->sz = sz;
  p->trapframe->epc = elf.entry;  // initial program counter = main
  p->trapframe->sp = sp;          // initial stack pointer
  proc_freepagetable(oldpagetable, oldsz);

  // <--- Code nhóm thêm vào bắt đầu từ đây --->
  if(p->pid == 1) { 
    vmprint(p->pagetable);
  }
  // <--- Kết thúc code nhóm thêm --->

  return argc; // bad: returns argc instead of 0
  
  // ... (phần xử lý lỗi bên dưới) ...
}

```

**Lý do:**

* Nhóm check điều kiện `p->pid == 1` vì đề chỉ yêu cầu in bảng trang của tiến trình đầu tiên (thường là `initcode`). Nếu không check dòng này, mỗi khi chạy lệnh mới (như `ls`, `cat`...), xv6 sẽ in ra rất nhiều bảng trang khác nhau.
* Hàm được đặt sau khi `p->pagetable` đã được thiết lập hoàn chỉnh để đảm bảo in ra đúng cấu trúc bộ nhớ cuối cùng.

