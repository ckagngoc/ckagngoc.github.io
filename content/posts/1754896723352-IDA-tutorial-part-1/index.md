---
title: "Hướng dẫn IDA Pro cơ bản cho Reverse Engineering - Phần 1"
date: 2025-08-11
draft: false
description: "Hướng dẫn cơ bản về IDA Pro - công cụ disassembler mạnh mẽ cho reverse engineering. Phần 1: Giới thiệu và các thao tác cơ bản"
tags: ["reverse-engineering", "ida-pro", "disassembler", "tutorial", "security"]
---

**IDA Pro** (Interactive DisAssembler Professional) là một trong những công cụ disassembler và debugger mạnh mẽ nhất hiện nay, được sử dụng rộng rãi trong lĩnh vực reverse engineering, phân tích malware, và nghiên cứu bảo mật.

## Tại sao nên học IDA Pro?

- **Hỗ trợ đa nền tảng**: x86, x64, ARM, MIPS, PowerPC và nhiều kiến trúc khác
- **Giao diện trực quan**: Graph view và Linear view giúp dễ dàng phân tích
- **Plugin ecosystem**: Hỗ trợ Python scripting và nhiều plugin mở rộng
- **Cross-references**: Theo dõi mối quan hệ giữa các hàm và biến
- **Decompiler**: Chuyển đổi assembly về pseudo-code C

## Cài đặt và thiết lập ban đầu

### Yêu cầu hệ thống

- **Windows**: Windows 10/11 (khuyến nghị)
- **RAM**: Tối thiểu 4GB, khuyến nghị 8GB+
- **Ổ cứng**: 2GB dung lượng trống

### Phiên bản IDA

- **IDA Free**: Miễn phí, chỉ hỗ trợ x86/x64, giới hạn file size < 64KB
- **IDA Pro**: Bản trả phí đầy đủ tính năng
- **IDA Home**: Bản dành cho cá nhân với giá cả hợp lý

## Giao diện cơ bản của IDA Pro

### 1. Disassembly View (IDA View)

```assembly
.text:00401000                 push    ebp
.text:00401001                 mov     ebp, esp
.text:00401003                 sub     esp, 0Ch
.text:00401006                 mov     [ebp+var_C], offset aHelloWorld
```

### 2. Hex View

Hiển thị dữ liệu dưới dạng hex bytes, hữu ích để xem raw data:

```text
00401000: 55 8B EC 83 EC 0C C7 45 F4 00 20 40 00
```

### 3. Functions Window

Liệt kê tất cả các function được IDA nhận diện:

- `main`
- `printf`
- `exit`
- ...

### 4. Names Window

Hiển thị tất cả các tên (symbols) trong chương trình.

## Thao tác cơ bản

### Navigation (Di chuyển)

- **Enter/Double-click**: Jump đến địa chỉ/function
- **Esc**: Quay lại vị trí trước đó
- **Ctrl+E**: Jump đến địa chỉ cụ thể
- **G**: Go to address
- **Space**: Chuyển đổi giữa Graph view và Linear view

### Phân tích Code

- **X**: Xem cross-references (ai gọi function này)
- **Ctrl+X**: Xem cross-references đến
- **N**: Đổi tên variable/function
- **Y**: Thay đổi type của variable
- **;**: Thêm comment
- **U**: Undefine code/data

### Shortcuts hữu ích

```text
F2    - Breakpoint (khi debug)
F5    - Decompile (nếu có Hex-Rays)
Tab   - Switch between disasm và pseudocode
Ctrl+S - Save database
Ctrl+W - Save snapshot
```

## Ví dụ thực hành: Phân tích chương trình đơn giản

Hãy tạo một chương trình C đơn giản để thực hành:

```c
#include <stdio.h>

int main() {
    printf("Hello, Reverse Engineering!\n");
    return 0;
}
```

### Bước 1: Load file vào IDA

1. Mở IDA Pro
2. File → Open → Chọn file .exe
3. Chọn processor type (thường IDA tự detect)
4. Đợi IDA phân tích xong

### Bước 2: Tìm hàm main

1. Mở Functions window (View → Open subviews → Functions)
2. Tìm function `main` hoặc `start`
3. Double-click để jump đến

### Bước 3: Phân tích assembly

```assembly
.text:00401550                 push    ebp           ; Save base pointer
.text:00401551                 mov     ebp, esp      ; Set up stack frame
.text:00401553                 push    offset aHelloReverse  ; "Hello, Reverse Engineering!"
.text:00401558                 call    _printf       ; Call printf
.text:0040155D                 add     esp, 4        ; Clean up stack
.text:00401560                 xor     eax, eax      ; return 0
.text:00401562                 pop     ebp           ; Restore base pointer
.text:00401563                 retn                  ; Return
```

### Bước 4: Đặt tên và comment

- Click vào `aHelloReverse`, nhấn `N` để đổi tên thành `hello_string`
- Thêm comment bằng `;` : "; Print hello message"

## Tips cho người mới bắt đầu

### 1. Luôn save database

IDA tạo file `.idb` chứa toàn bộ phân tích của bạn. Nhớ save thường xuyên!

### 2. Sử dụng Cross-references

- `X` trên một function để xem ai gọi nó
- Giúp hiểu flow của chương trình

### 3. Đặt tên có ý nghĩa

- Thay vì `sub_401000`, đặt tên `decrypt_string`
- Thay vì `dword_403000`, đặt tên `global_counter`

### 4. Thêm comments

```assembly
.text:00401553    push    offset aHelloReverse  ; Push string address
.text:00401558    call    _printf               ; Print the string
```

### 5. Hiểu calling conventions

**Calling convention** là quy ước giữa caller (hàm gọi) và callee (hàm được gọi) về cách truyền arguments, nơi đặt giá trị trả về, register nào phải được bảo toàn và ai chịu trách nhiệm dọn stack. Biết quy ước này giúp bạn đọc đúng function signature trong IDA thay vì chỉ đoán từ tên hàm.

#### `cdecl`: caller dọn stack

Với x86 32-bit, arguments của `cdecl` thường được đưa lên stack theo thứ tự từ phải sang trái. Ví dụ với:

```c
int add(int left, int right);
```

caller có thể tạo lời gọi như sau:

```assembly
push    2              ; right
push    3              ; left
call    add
add     esp, 8         ; caller dọn 2 arguments, mỗi argument 4 bytes
; eax = 5
```

Trong callee, sau prologue chuẩn, `left` nằm ở `[ebp+8]` và `right` nằm ở `[ebp+0Ch]`:

```assembly
add:
push    ebp
mov     ebp, esp
mov     eax, [ebp+8]   ; left = 3
add     eax, [ebp+0Ch] ; right = 2
pop     ebp
retn                  ; không có số bytes đi kèm
```

Dấu hiệu quan trọng trong IDA là instruction `add esp, 8` xuất hiện **sau `call`**. `cdecl` phù hợp với hàm có số lượng arguments thay đổi như `printf`, vì caller biết chính xác cần dọn bao nhiêu bytes.

#### `stdcall`: callee dọn stack

Với cùng prototype, callee sẽ dọn arguments khi return:

```assembly
push    2
push    3
call    add_stdcall
; không cần add esp, 8 ở đây
```

Cuối callee thường có `retn 8`:

```assembly
add_stdcall:
push    ebp
mov     ebp, esp
mov     eax, [ebp+8]
add     eax, [ebp+0Ch]
pop     ebp
retn    8              ; return và bỏ 8 bytes arguments khỏi stack
```

Nếu thấy `retn N`, hãy xem `N` có khớp với tổng kích thước arguments hay không. Đây là một dấu hiệu tốt để kiểm tra IDA đã nhận diện đúng calling convention chưa, nhưng không nên dùng nó một mình vì compiler có thể tối ưu prologue/epilogue.

#### `fastcall`: ưu tiên register

`fastcall` giảm số lần truy cập stack bằng cách truyền một vài arguments đầu tiên qua register. Một biến thể thường gặp trên x86 dùng `ecx` cho argument đầu tiên và `edx` cho argument thứ hai; các arguments còn lại mới được đặt trên stack:

```c
int multiply(int left, int right);
```

Một lời gọi có thể trông như sau:

```assembly
mov     ecx, 6         ; left
mov     edx, 7         ; right
call    multiply
; eax = 42
```

Trong thực tế, quy tắc register của `fastcall` phụ thuộc compiler và nền tảng. Vì vậy, hãy đặt con trỏ vào function, nhấn `Y` để xem hoặc chỉnh calling convention, rồi đối chiếu với các instruction đọc `ecx`, `edx` và stack.

#### Cách nhận diện nhanh trong IDA

Khi phân tích một `call`, hãy kiểm tra theo thứ tự:

1. Ngay trước `call`, arguments được `push` lên stack hay được đưa vào register nào?
2. Ngay sau `call`, có `add esp, N` không?
3. Ở cuối callee là `retn` hay `retn N`?
4. Giá trị trả về có được đọc từ `eax` (x86) hoặc `rax` (x64) không?

Ví dụ, chuỗi `push`, `call`, `add esp, 8` gợi ý `cdecl`, còn `call` kết thúc bằng `retn 8` gợi ý `stdcall`. Đây là các dấu hiệu để hình thành giả thuyết; hãy xác nhận lại bằng nhiều caller và bằng cách sửa prototype trong Hex-Rays nếu pseudo-code vẫn không hợp lý.

## Cấu trúc một chương trình PE

### PE Header

- DOS Header
- NT Header
- Section Headers

### Code Section (.text)

- Chứa instruction thực thi
- Quyền: Read + Execute

### Data Section (.data/.rdata)

- `.data`: Dữ liệu có thể ghi
- `.rdata`: Dữ liệu chỉ đọc (constants, strings)

### Import/Export Tables

- **Import**: Function từ DLL khác
- **Export**: Function mà chương trình export

## Kết luận

Trong phần 1 này, chúng ta đã tìm hiểu:

- Giới thiệu về IDA Pro và tầm quan trọng
- Giao diện cơ bản và các window chính
- Shortcuts và thao tác navigation cơ bản
- Cách phân tích một chương trình đơn giản
- Tips và best practices cho người mới

**Phần 2 sẽ đề cập đến**:

- Phân tích function arguments và local variables
- Hiểu về stack frame và calling conventions
- Techniques để bypass anti-analysis
- Script automation với IDAPython

---
