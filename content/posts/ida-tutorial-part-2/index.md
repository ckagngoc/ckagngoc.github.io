---
title: "Hướng dẫn IDA Pro cơ bản cho Reverse Engineering - Phần 2"
date: 2026-08-21
draft: false
description: "Hướng dẫn cơ bản về IDA Pro - công cụ disassembler mạnh mẽ cho reverse engineering. Phần 2: Nâng cấp kỹ năng IDA Pro"
tags: ["reverse-engineering", "ida-pro", "disassembler", "tutorial", "security"]
---

Phần 1 đã giới thiệu giao diện và quy trình điều hướng cơ bản trong IDA Pro. Ở phần này, chúng ta sẽ đi sâu hơn vào một function: dữ liệu đi vào bằng cách nào, biến cục bộ nằm ở đâu, compiler tạo stack frame ra sao, và làm thế nào để hỗ trợ phân tích những binary có anti-analysis.

> Chỉ phân tích các binary mà bạn sở hữu hoặc được phép kiểm thử. Các kỹ thuật trong bài nên được dùng trong lab, CTF, malware sandbox và hoạt động nghiên cứu có ủy quyền.

## 1. Đọc function signature

Khi mở một function bằng Hex-Rays (`F5`), việc đầu tiên không phải là tin ngay vào pseudo-code. Hãy kiểm tra signature, các cross-reference và assembly xung quanh những lệnh `call`. IDA có thể suy luận sai kiểu dữ liệu hoặc số lượng arguments, đặc biệt khi binary bị tối ưu hóa.

Ví dụ một function nhận chuỗi và độ dài:

```c
int check_input(const char *input, size_t length);
```

Trong IDA, có thể chỉnh prototype bằng cách đặt con trỏ trong function rồi nhấn `Y`. Kiểu dữ liệu đúng giúp Hex-Rays hiển thị phép dereference, phép tính pointer và giá trị trả về dễ hiểu hơn.

### Arguments trên stack

Với chương trình x86 32-bit dùng `cdecl`, arguments thường được `push` theo thứ tự từ phải sang trái. Sau prologue chuẩn, argument đầu tiên nằm tại `[ebp+8]`:

```assembly
push    ebp
mov     ebp, esp
sub     esp, 10h
mov     eax, [ebp+8]       ; arg_0
mov     edx, [ebp+0Ch]      ; arg_1
mov     [ebp-4], eax        ; var_4
```

Vị trí thường gặp:

| Địa chỉ tương đối | Ý nghĩa |
| --- | --- |
| `[ebp+8]` | argument đầu tiên |
| `[ebp+0Ch]` | argument thứ hai |
| `[ebp-4]` | local variable |
| `[ebp-8]` | local variable tiếp theo |
| `[ebp+4]` | địa chỉ trả về |
| `[ebp]` | giá trị `ebp` của caller |

Đây chỉ là quy ước của prologue cụ thể. Compiler có thể bỏ frame pointer, vì vậy hãy nhìn vào cách `esp` thay đổi thay vì áp dụng cứng các offset trên.

### Arguments trong register

Ở x64, arguments thường đi qua register trước khi dùng stack:

| ABI | Arguments integer/pointer đầu tiên |
| --- | --- |
| Windows x64 | `RCX`, `RDX`, `R8`, `R9` |
| System V AMD64 (Linux/macOS) | `RDI`, `RSI`, `RDX`, `RCX`, `R8`, `R9` |
| ARM64 | `X0` đến `X7` |

Với Windows x64, caller còn dành sẵn 32 bytes *shadow space* trên stack cho callee. Argument thứ năm trở đi thường nằm trên stack. Với System V AMD64, arguments vượt quá số register khả dụng cũng được đặt trên stack. Khi phân tích một `call`, hãy ghi lại giá trị register ngay trước lệnh gọi và theo dõi nơi callee đọc chúng.

## 2. Phân tích local variables

Local variable thường được compiler biểu diễn bằng offset âm so với frame pointer hoặc bằng offset từ `esp`/`rsp`:

```assembly
sub     rsp, 30h
lea     rcx, [rbp-20h]      ; địa chỉ buffer cục bộ
mov     dword ptr [rbp-4], 0 ; counter = 0
```

Một offset chưa nói lên kiểu dữ liệu. Hãy dùng các dấu hiệu sau:

- `lea` lấy địa chỉ của vùng nhớ thường gợi ý một buffer, array hoặc struct.
- `mov eax, [rbp-4]` và phép cộng 32-bit thường gợi ý `int` hoặc counter.
- Nhiều lần truy cập theo bước 8 bytes có thể là pointer hoặc phần tử trên x64.
- Một vùng được truyền vào `memcpy`, `strcmp`, `scanf` hoặc API tương tự cần được kiểm tra cả kích thước lẫn giới hạn.
- Vùng được khởi tạo bằng `memset` thường là buffer hoặc struct; xem số byte được ghi để ước lượng kích thước.

Trong pseudocode, đổi tên `var_20` thành `input_buffer`, `var_4` thành `index` bằng `N`. Với một biến đã chọn, nhấn `Y` để sửa kiểu. Đổi tên và kiểu theo từng giả thuyết nhỏ, sau đó đối chiếu lại với assembly để tránh tạo ra một pseudo-code có vẻ đẹp nhưng sai.

### Phân biệt giá trị và địa chỉ

```assembly
mov     eax, [rbp-4]        ; đọc giá trị của biến
lea     rdx, [rbp-20h]      ; lấy địa chỉ của biến/buffer
mov     [rbp-8], rax        ; ghi một pointer hoặc số 64-bit
```

`mov` với dấu ngoặc vuông truy cập bộ nhớ; `lea` tính địa chỉ mà không đọc nội dung tại địa chỉ đó. Đây là khác biệt quan trọng khi xác định một argument là `char *`, `char **` hay một giá trị số.

## 3. Stack frame và calling conventions

### Prologue và epilogue

Một frame x86 điển hình có dạng:

```assembly
push    ebp
mov     ebp, esp
sub     esp, 20h
...
mov     esp, ebp
pop     ebp
retn
```

Prologue lưu frame pointer và cấp phát chỗ cho local variables. Epilogue khôi phục stack trước khi `ret` lấy địa chỉ trở về. Compiler tối ưu hóa có thể dùng `leave`, bỏ `ebp` hoàn toàn hoặc gộp nhiều thao tác; vì vậy hãy xác định frame qua data-flow của `esp`/`rsp`.

### Các calling convention phổ biến

| Convention | Truyền arguments | Ai dọn stack? | Dấu hiệu thường gặp |
| --- | --- | --- | --- |
| `cdecl` | stack | caller | sau `call` có `add esp, N` |
| `stdcall` | stack | callee | `ret N` ở cuối function |
| `fastcall` | một số register, phần còn lại trên stack | tùy ABI | thường thấy `ecx`/`edx` được dùng sớm |
| Windows x64 | `rcx`, `rdx`, `r8`, `r9` | caller theo ABI | shadow space 32 bytes |
| System V AMD64 | `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` | caller theo ABI | register và stack layout khác Windows |

Khi signature của IDA sai, chọn `Edit function` hoặc nhấn `Y` để sửa calling convention và kiểu trả về. Sau đó kiểm tra các caller: nếu số arguments hoặc register mapping không khớp, prototype vẫn chưa đúng.

### Quy trình theo dõi một lời gọi hàm

1. Đặt breakpoint tại instruction ngay trước `call` khi debug được.
2. Ghi lại các register chứa arguments và vùng stack liên quan.
3. Step into callee, kiểm tra instruction đầu tiên đọc argument.
4. Đặt tên cho argument, local variable và giá trị trả về.
5. Theo dõi nơi return value được dùng, thường là `eax`/`rax` hoặc `x0`.

Với function variadic như `printf`, số arguments không thể suy ra chỉ từ prototype đơn giản. Hãy kiểm tra format string và cách caller chuẩn bị từng giá trị.

## 4. Nhận diện và xử lý anti-analysis

Anti-analysis làm giảm chất lượng disassembly hoặc thay đổi hành vi khi chạy trong debugger. Mục tiêu trong quá trình phân tích là nhận diện, cô lập và ghi nhận tác động của nó; không nên vô hiệu hóa cơ chế bảo vệ trên hệ thống không được phép.

### Một số dấu hiệu

- **Phát hiện debugger**: gọi `IsDebuggerPresent`, `CheckRemoteDebuggerPresent`, đọc PEB hoặc dùng exception bất thường.
- **Phát hiện môi trường**: kiểm tra tên process, registry, số CPU, timing hoặc thiết bị ảo.
- **Control-flow obfuscation**: jump chain, opaque predicate, junk instruction và indirect jump.
- **API hashing và dynamic resolution**: không có tên API rõ trong import table; chương trình tự tìm địa chỉ bằng hash.
- **Packed/self-modifying code**: vùng `.text` bị ghi, code chỉ xuất hiện sau khi giải nén hoặc được tạo động.

### Quy trình phân tích thực tế

1. Tạo snapshot hoặc bản sao của sample và ghi hash để có thể lặp lại kết quả.
2. Bắt đầu bằng static analysis: sections, imports, strings, xrefs và các vùng có quyền `RWX`.
3. Dùng debugger trong VM cô lập; đặt breakpoint tại API hoặc instruction thay đổi quyền bộ nhớ.
4. Khi code được giải mã tại runtime, dump vùng nhớ trong lab rồi nạp bản dump vào IDA để phân tích tiếp.
5. Đánh dấu các vùng chưa xác định bằng `U`, tạo lại code bằng `C` khi đã xác định đúng boundary, và dùng `Make function` khi cần.
6. Ghi lại mọi giả thuyết trong comment. Một patch tạm thời chỉ nên dùng để quan sát một nhánh trong môi trường kiểm thử và phải được ghi rõ.

Không nên kết luận một binary là packed chỉ vì entropy cao hoặc import ít. Hãy kết hợp nhiều dấu hiệu: entry point nhỏ, section bất thường, code thực thi được tạo sau khi chạy và sự thay đổi quyền trang nhớ.

## 5. Script automation với IDAPython

IDAPython phù hợp cho các tác vụ lặp lại như liệt kê function, tìm call đến API, đặt tên theo quy tắc và xuất báo cáo. Có thể chạy script từ `File > Script command...` hoặc mở Python console trong IDA.

### Liệt kê function và xuất JSON

Script sau dùng API tương thích IDA 7.x/8.x. Nó chỉ đọc database và ghi báo cáo ra thư mục hiện tại của database:

```python
import json
from pathlib import Path

import ida_funcs
import ida_kernwin
import idaapi
import idautils
import idc


def collect_functions():
	result = []
	for function_ea in idautils.Functions():
		function = ida_funcs.get_func(function_ea)
		if function is None:
			continue

		result.append({
			"address": hex(function.start_ea),
			"name": idc.get_func_name(function.start_ea),
			"size": function.end_ea - function.start_ea,
		})
	return result


def main():
	input_path = Path(idc.get_input_file_path())
	output_path = input_path.with_name(f"{input_path.stem}_function_report.json")
	output_path.write_text(
		json.dumps(collect_functions(), indent=2),
		encoding="utf-8",
	)
	ida_kernwin.msg(f"Wrote {output_path}\\n")


if __name__ == "__main__":
	main()
```

Việc kiểm tra function bằng `ida_funcs.get_func()` giúp script bỏ qua vùng địa chỉ không còn là function hợp lệ. Nếu input path không ghi được, hãy đổi `output_path` sang một thư mục làm việc riêng.

### Tìm các lệnh gọi API đáng chú ý

```python
import idautils
import idc


TARGETS = {"VirtualAlloc", "VirtualProtect", "CreateThread"}

for function_ea in idautils.Functions():
	for instruction_ea in idautils.FuncItems(function_ea):
		if idc.print_insn_mnem(instruction_ea) != "call":
			continue

		target_name = idc.get_name(idc.get_operand_value(instruction_ea, 0), 0)
		if target_name in TARGETS:
			print(f"{hex(instruction_ea)} -> {target_name}")
```

Tên import có thể bị decorate, forward hoặc resolve động nên danh sách kết quả không phải bằng chứng đầy đủ. Kết hợp script với xrefs (`X`) và kiểm tra assembly quanh mỗi call.

### Mẹo viết script bền vững

- Dùng `idaapi.get_inf_structure().is_64bit()` hoặc API tương ứng để tách logic x86/x64.
- Kiểm tra `BADADDR`, `None` và function boundary trước khi đọc operand.
- Không sửa tên hàng loạt nếu chưa có quy tắc lọc và bản sao database.
- In địa chỉ bằng `hex()` và lưu input hash cùng báo cáo để kết quả có thể tái lập.
- Chia script thành các hàm nhỏ, chạy thử trên một function trước khi quét toàn bộ database.

## 6. Checklist phân tích

- [ ] Xác định kiến trúc, compiler và calling convention có khả năng được dùng.
- [ ] Kiểm tra prototype và số lượng arguments của function.
- [ ] Đổi tên arguments/local variables theo data-flow đã xác nhận.
- [ ] Vẽ stack layout, chú ý alignment, saved registers và shadow space.
- [ ] Kiểm tra dấu hiệu debugger detection, packing và dynamic API resolution.
- [ ] Dùng debugger trong môi trường cô lập để quan sát code sau khi unpack.
- [ ] Chạy script trên bản sao database và lưu lại báo cáo cùng hash sample.

## Kết luận

Phân tích function hiệu quả là quá trình kết hợp giữa assembly, calling convention, data-flow và quan sát runtime. Khi IDA đưa ra một giả thuyết về argument hoặc local variable, hãy kiểm chứng nó bằng caller, instruction thực tế và debugger. IDAPython sau đó giúp biến các thao tác lặp lại thành quy trình có thể tái lập, giảm thời gian tìm kiếm và tăng độ nhất quán của báo cáo.

Ở phần tiếp theo, có thể mở rộng sang phân tích memory corruption, unpacking có kiểm soát và xây dựng plugin IDA cho một workflow cụ thể.



