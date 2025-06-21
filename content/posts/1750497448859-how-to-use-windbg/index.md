---
title: "How to use WinDbg"
date: 2025-06-21
draft: false
description: "Hướng dẫn sử dụng WinDbg: thao tác cơ bản, debug nâng cao, mẹo và tài liệu tham khảo."
tags: ["windbg", "debug", "reverse", "trick"]
---

# Debug_With_WinDbg

Học cách sử dụng WinDbg nổ não với ckagngoc 🔪🔪🔪

---

## 1. Giới thiệu tổng quan

**WinDbg** là một trong những công cụ debug mạnh mẽ nhất trên Windows, hỗ trợ debug cả user-mode và kernel-mode. WinDbg thường được dùng để phân tích crash, reverse engineering, kiểm tra malware, và nghiên cứu hệ điều hành.

---

## 2. Các thao tác cơ bản

- **Load file thực thi vào WinDbg**
  - Sử dụng phím tắt: `Ctrl + E` để mở file thực thi (EXE/DLL).
  - Sau khi load, chương trình sẽ dừng tại EntryPoint.

- **Lấy thông tin PEB tiến trình hiện tại**
  ```
  !peb
  ```

- **Lấy thông tin các module tiến trình**
  ```
  # Tất cả module
  lm
  # Thông tin chi tiết một module
  lmv m <module name>
  ```

- **Liệt kê symbol của một module**
  ```
  # Tất cả symbol
  x <modulename>!*
  # Lọc symbol theo tên
  x <modulename>!*<string>*
  # Ví dụ
  x kernel32!*
  x kernel32!*File*
  ```

- **Đặt breakpoint tại symbol**
  ```
  bp <modulename>!<symbolname>
  # Ví dụ
  bp kernel32!CreateFileA
  ```

- **Liệt kê, xóa, tắt breakpoint**
  ```
  bl   # Liệt kê breakpoint
  bc n # Xóa breakpoint số n
  bd n # Disable breakpoint số n
  be n # Enable breakpoint số n
  ```

- **Tiếp tục thực thi chương trình**
  ```
  g
  ```

- **Kiểm tra stack tại điểm dừng**
  ```
  k
  ```

- **Kiểm tra giá trị thanh ghi**
  ```
  r
  ```

- **Disassemble lệnh tại địa chỉ**
  ```
  u <address>
  # Ví dụ
  u rip
  ```

- **Xem unicode/string tại địa chỉ**
  ```
  du <address>
  # Ví dụ
  du rdi
  ```

- **Dump tệp EXE từ bộ nhớ**
  Sau khi xác định địa chỉ module, dùng lệnh:
  ```
  !save_module <start_addr> <end_addr> <output_path>
  # Ví dụ
  !save_module 00400000 00456000 C:\dumped_files\myapp.exe
  ```

---

## 3. Các lệnh debug nâng cao

- **Theo dõi API call, hook function**
  - Đặt breakpoint tại các API quan trọng (CreateFile, VirtualAlloc, WriteProcessMemory, ...)
  - Sử dụng lệnh `bp`, `bu`, `bm` để đặt breakpoint linh hoạt.

- **Theo dõi memory, dump vùng nhớ**
  ```
  .writemem <file> <start_addr> <end_addr>  # Dump vùng nhớ ra file
  db <address>  # Xem dữ liệu dạng hex
  dq <address>  # Xem dữ liệu dạng QWORD
  ```

- **Theo dõi thread, context**
  ```
  ~            # Liệt kê thread
  ~n s         # Chuyển sang thread số n
  .thread <id> # Chọn thread theo id
  ```

- **Theo dõi exception, handle event**
  ```
  sxe av       # Break khi access violation
  sxi av       # Ignore access violation
  .exr -1      # Xem thông tin exception gần nhất
  ```

---

## 4. Cách syncup với IDA

- **Kết nối WinDbg với IDA Pro**
  - Trong IDA, chọn Debugger > Attach to Remote Debugger > chọn WinDbg.
  - Nhập port và thông tin kết nối.
  - Trong WinDbg, chạy:
    ```
    .server tcp:port=23946
    ```
  - IDA sẽ kết nối tới WinDbg qua port này, cho phép debug song song, đồng bộ symbol, breakpoint, ...

- **Lợi ích**: Có thể tận dụng khả năng phân tích tĩnh của IDA và debug động của WinDbg cùng lúc.

---

## 5. Các mẹo hữu ích

- **Sử dụng lệnh .hh <command> để xem help chi tiết**
- **Dùng .logopen/.logclose để ghi log phiên debug**
- **Dùng .restart để khởi động lại tiến trình debug**
- **Dùng .exr -1 để xem thông tin exception gần nhất**
- **Dùng !analyze -v để tự động phân tích crash dump**
- **Tùy biến giao diện, màu sắc, font chữ cho dễ nhìn**

---

## 6. Tài liệu tham khảo

- [WinDbg Documentation (Microsoft)](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/)
- [WinDbg Cheat Sheet](https://gist.github.com/ckagngoc/your-windbg-cheatsheet)
- [Reverse Engineering with WinDbg](https://www.aldeid.com/wiki/WinDbg)
- [IDA Pro Debugger Integration](https://hex-rays.com/products/ida/debugger/)

---

## 7. Kỹ thuật nâng cao cho Reverse Engineer

- **Debug packed/cracked file**
  - Đặt breakpoint tại OEP (Original Entry Point) hoặc các API thường bị gọi khi unpack (VirtualAlloc, WriteProcessMemory, GetProcAddress, ...).
  - Sử dụng lệnh `bp`, `bu`, `bm` để đặt breakpoint động, breakpoint wildcard.
  - Dùng lệnh `.reload /f` để reload symbol khi unpack xong.

- **Theo dõi dynamic import, IAT, API resolve runtime**
  - Sử dụng lệnh `!dh <module base>` để xem thông tin PE header, IAT.
  - Dùng `u <address>` để disassemble vùng code resolve API.

- **Phân tích anti-debug, anti-VM**
  - Đặt breakpoint tại các API/technique thường dùng để phát hiện debugger (IsDebuggerPresent, CheckRemoteDebuggerPresent, NtQueryInformationProcess, ...).
  - Theo dõi exception bất thường bằng `sxe -c ".echo Exception!;g" *` để log mọi exception.

- **Script hóa với WinDbg**
  - Sử dụng lệnh `.cmdtree`, `.foreach`, `.printf`, `.if`, `.else` để tự động hóa thao tác debug.
  - Viết script WinDbg bằng JavaScript/Python với WinDbg Preview (sử dụng lệnh `.scriptload`).

- **Debug multi-process, multi-thread**
  - Sử dụng lệnh `.process`, `.thread`, `~`, `.childdbg 1` để debug process con.
  - Dùng `.attach <pid>` để attach vào process bất kỳ.

- **Phân tích heap, memory leak**
  - Sử dụng extension `!heap`, `!address`, `!analyze -v` để kiểm tra heap, memory leak, corruption.

- **Debug kernel-mode, driver**
  - Kết nối WinDbg với máy ảo qua COM/pipe/USB để debug kernel.
  - Sử dụng lệnh `!drvobj`, `!devobj`, `!irp`, `!process` để phân tích driver, device, IRP.

- **Tích hợp WinDbg với các công cụ khác**
  - Kết hợp với x64dbg, Ghidra, IDA Pro để phân tích song song.
  - Sử dụng lệnh `.server`, `.client` để debug từ xa, chia sẻ session debug.

---

## 8. Một số extension hữu ích cho WinDbg

- **ext.dll**: Extension mặc định, hỗ trợ nhiều lệnh phân tích process, thread, heap.
- **dbghelp.dll**: Hỗ trợ symbol, stackwalk, minidump.
- **pykd**: Cho phép script hóa WinDbg bằng Python.
- **msec.dll**: Extension của Microsoft Security Engineering Center, hỗ trợ phân tích bảo mật.

---

## 9. Một số lệnh hữu ích khác

- `.ecxr` : Chuyển context về exception gần nhất.
- `!handle` : Liệt kê handle của process.
- `!teb` : Xem thông tin TEB (Thread Environment Block).
- `!list` : Duyệt linked list trong memory.
- `!object` : Xem thông tin object kernel.
- `!locks` : Phân tích deadlock, lock kernel.
- `!stacks` : Liệt kê stack của tất cả thread.

---

## 10. Tham khảo thêm

- [WinDbg Scripting (Microsoft)](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/scripting-and-automation)
- [pykd - Python extension for WinDbg](https://github.com/masm/pykd)
- [WinDbg Cheat Sheet (Reverse Engineering)](https://github.com/0x4d5a/WinDbg-Cheat-Sheet)
- [Advanced WinDbg Tips](https://www.geoffchappell.com/notes/windows/debug/windbg/index.htm)

---