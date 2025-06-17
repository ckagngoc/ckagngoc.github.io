---
title: "[Write-Up RE] Get2TheHell"
date: 2025-06-17
draft: false
description: "a description"
tags: ["WriteUp", "reverse", "CTF", "event"]
---

![](https://hackmd.io/_uploads/SJa5WWThn.png)
Hì hì bài sửa chữa sai lầm và vực dậy chút tinh thần sau kỳ thi 😂😂😂

## Ý tưởng

Dựa vào hint Heaven Gate mình có tra được trên mạng tìm được đoạn này:
![](https://hackmd.io/_uploads/BkUUNZahn.png)

## Thực hành

Theo ý tưởng phân tích, bên trong hàm main là hàm kiểm tra các đặc tính của flag:
![](https://hackmd.io/_uploads/B1QxSWphh.png)
Ta thấy được rằng, trong câu lệnh if hàm main kiểm tra một số đặc tính đơn giản và có một hàm **_sub_D61000_** đặc biệt.
Kiểm tra hàm này ta nhận ra hàm cấp phát một vùng nhớ cho con trỏ **_lpAddress_** và config một số thứ trên vùng nhớ này để thực thi một đoạn code trên vùng nhớ đó và kiểm tra bằng câu lệnh if sau đó sẽ giải phóng vùng nhớ này. 🙄😶😑
![](https://hackmd.io/_uploads/rk9gLWpnn.png)
Ta tiến hành đặt breakpoint tại vị trí này và xem nó cấp phát cho ta cái gì và nhận được một hàm ngắn
![](https://hackmd.io/_uploads/BJAaUbT23.png)
Hàm này gọi đến một đoạn code ngắn <Nhớ make code bằng nút 'C' mới hiện> và trong đó có thứ rất đang nghi 😑😐😐 **Offset của input**
![](https://hackmd.io/_uploads/rJDOw-pn3.png)
Đọc đoạn code này mình đoán được đây là hàm chính cần xem xét nhưng bên trong hàm xuất hiện một số ký tự lạ và một vài byte không xác định, thêm với đó là một số câu lệnh đọc rất khó hiểu như sau:

![](https://hackmd.io/_uploads/HJYP_ZT2n.png)

![](https://hackmd.io/_uploads/SJLKd-Tnh.png)
Tình trạng chung là có những dấu hiệu địa chỉ kia là x64 và biểu hiện rất giống Heaven Gate như ý tưởng !!! 🙂🙂🙂

## Cách làm

Ta tiến hành dump số byte tại vùng nhớ này ra để patch lại vào một file x64 xem ý nghĩa code là gì <Đại ca nào đó rất đẹp zai chỉ dạy 😋😋😋>
Ta nắm lại vị trí bắt đầu của **lpAddress** và số lượng byte để dumo nguyên cả vùng nhớ đó ra

Mình tiến hành code một file hello world bằng c sau đó dịch ra x64 bằng g++
Sau đó patch đoạn byte đã dump ra vào entry point của chương trình để thực thi chỗ shellcode này.

### Python Script patch bytes

```
import ida_bytes

newbytes = b"\x6A\x03\x58\xC1\xE0\x04\x04\x03\x50\x68\x34\x00\xCC\x00\xCB\xFE\xE2\x7B\x21\x9C\xEA\x4F\x05\xE0\xEA\x4F\x8D\x18\x5E\xE5\x5D\x18\x68\xE9\x03\x1E\x3C\x63\x89\xBE\xE2\x7D\x89\xC6\xEA\x7B\x51\x10\x5E\x69\x89\xA0\xB8\x08\x00\x00\x00\x83\xE8\x25\x85\xC0\x75\x51\x49\xB8\x59\xFC\x9F\x00\x00\x00\x00\x00\x49\xB9\x0F\x00\xCC\x00\x00\x00\x00\x00\x31\xF6\x48\x8D\x56\x3C\x48\x8D\x4E\x41\x4C\x8D\x96\xD3\x00\x00\x00\x4C\x8D\x9E\xF7\x00\x00\x00\x48\x83\xFE\x25\x73\x1F\x41\x8A\x04\x31\xD0\xC8\x41\x32\x04\x30\x30\xD0\x84\xC0\x87\xCA\x44\x87\xD1\x45\x87\xDA\x75\x07\x48\x8D\x74\x30\x01\xEB\xDB\x48\xBA\x34\xFC\x9F\x00\x00\x00\x00\x00\x48\x89\x02\x6A\x23\x68\xA7\x00\xCC\x00\x48\xCB\xC3"
size = len(newbytes)
start_address = 0x0000000000001050

for i in range(0,size):
    ida_bytes.patch_byte(start_address,newbytes[i])
    start_address += 1

```

Và đây là thành quả
![](https://hackmd.io/_uploads/BJN00-Tnh.png)
Hàm khá dễ đọc cũng không nhất thiết là phải debug làm gì, chương trình sẽ đổi giá trị các thanh ghi sau mỗi vòng lạp lấy cipher để xor với dl và input, với cipher là đoạn byte không phải code trong vùng nhớ này, vì là mã đối xứng nên ta chỉ cần làm ngược lại quy trình mã hóa 😚😚😙
![](https://hackmd.io/_uploads/BJEV1zan3.png)

### Code giải mã

```
def rotate_bits_right(number, positions, bitsize=32):
    mask = (1 << bitsize) - 1
    return ((number & mask) >> positions) | ((number & mask) << (bitsize - positions)) & mask

r9 = b"\xFE\xE2\x7B\x21\x9C\xEA\x4F\x05\xE0\xEA\x4F\x8D\x18\x5E\xE5\x5D\x18\x68\xE9\x03\x1E\x3C\x63\x89\xBE\xE2\x7D\x89\xC6\xEA\x7B\x51\x10\x5E\x69\x89\xA0"

r8 = ""

rdx = 0x3c

rcx = 0x41

r10 = 0xd3

r11 = 0xf7

flag = b""

tmp = 0

for i in range(0,37):
    tmp = rotate_bits_right(r9[i],1,8)
    tmp ^= rdx
    r8 += chr(tmp)
    rcx, rdx = rdx, rcx
    r10, rcx = rcx, r10
    r11, r10 = r10, r11
print(r8)
```

## Flag: MSEC{C0ngr4tuL4t10n!Y0u'v3_b3c0m3_4n_4ng3l}

🤑🤑🤑

--Ckagngoc--
