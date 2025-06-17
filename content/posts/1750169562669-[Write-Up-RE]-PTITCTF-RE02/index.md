---
title: "[Write-Up RE] PTITCTF RE02"
date: 2025-06-17
draft: false
description: "a description"
tags: ["WriteUp", "reverse", "CTF", "event"]
---
## Challenge
- [1. RE02](#RE02)
## RE02
![](https://hackmd.io/_uploads/Hk5tKS5T3.png)

Bài có một số kiến thức về anti debug và anti disass 😴😴😪

## Ý tưởng
Bài dựa trên sự kết hợp của anti-debug và anti-diassembly.

## Thực hành 
Chạy thử chương trình, là dạng nhập input-check quen thuộc

![](https://hackmd.io/_uploads/HJOe9rcp3.png)

Nhưng điều là ở đây là mình nhập ký tự và nhấn enter nhuwng chương trình kiểu "Đố anh chạy được em", mình đoán là chương trình đã dùng một cách nào đó để tự dừng lại, load chương trình vào IDA thì đúng thế ta thấy được hàm *sub_401170* sleep chương trình ngay sau printf

![](https://hackmd.io/_uploads/ryouqrcan.png)

![](https://hackmd.io/_uploads/rkbKcH9T3.png)

Nó cũng chỉ dừng có 24h thôi ấy mà 🤐🤐
Mình patch lại lệnh gọi hàm và chạy thử lại chương trình

![](https://hackmd.io/_uploads/SJmOirca2.png)

Ngon 😂😂😂
Check các hàm TlsCallBack ta nhận thấy vùng code hiện thông báo từ chối truy cập khi debug: 

![](https://hackmd.io/_uploads/rJg9RSc62.png)

![](https://hackmd.io/_uploads/ryzcCS5a3.png)

Mình patch lại các lệnh nhảy đến vùng code này trong hàm đó tại đây:

![](https://hackmd.io/_uploads/H1IaAH56n.png)

thành

![](https://hackmd.io/_uploads/Syo0AB9an.png)

Và đây:

![](https://hackmd.io/_uploads/SkgxyL562.png)

Thành

![](https://hackmd.io/_uploads/ryh-JU9Th.png)

Oke đã debug bình thường:

![](https://hackmd.io/_uploads/HJoUyUqah.png)

Nghịch 1 lúc mình nhận ra hàm main này khá dị, nó có hàm cuối cùng bị *jump out* và nó không có retn

![](https://hackmd.io/_uploads/S1T5JU5pn.png)

![](https://hackmd.io/_uploads/S1Hi1U9T2.png)

Kiểm tra code ở dạng text ta nhận thấy dưới hàm main còn một đoạn code nữa nhưng chưa được nhận diện đúng do hàm cuối trên bị lỗi:

![](https://hackmd.io/_uploads/B1EZxU9pn.png)

Patch lại byte 0xCC trong Hex và make func lại hàm main và hàm *sub_31140* ta được giả mã hàm main có đoạn check debug như sau:

![](https://hackmd.io/_uploads/BkHFxIcan.png)

Và đây là đoạn mã hóa mật khẩu nhập vào và check với cipher lưu sẵn: 

![](https://hackmd.io/_uploads/BJoM-Lcp2.png)

Giải thích sơ qua hàm mã hóa chương trình sẽ chạy trên một string gồm các phần tử có giá trị từ '0' -> '3' được lưu trong v13 với mỗi phần tử tương ứng chương trình sẽ xử lý trên phần tử thứ v14 - 0x30 tương ứng của dãy nhập vào hai dãy mà v13 và v14 chạy đc đưa qua 2 byte sau và lặp lại 51 lần:

![](https://hackmd.io/_uploads/HyJsZU9a2.png)

Ta lấy hai đoạn string đó ra, debug nhận cipher và code để nhận flag như sau:

```
ct = [0x50, 0xAE, 0xEE, 0x54, 0x43, 0x53, 0x79, 0x5F, 0x62, 0x62, 0x79, 0x5B, 0x35, 0x62, 0x66, 0x54, 0xD7, 0xD0, 0x34, 0x52, 0xF2, 0x2E, 0x34, 0x12, 0xC6, 0x1E, 0x44, 0x63, 0x42, 0x3E, 0xC3, 0x5F, 0xE5, 0x01, 0x70, 0x6A, 0x79, 0x7D]

v12 = [0x32, 0x32, 0x32, 0x33, 0x33, 0x33, 0x30, 0x31, 0x32, 0x33, 0x32, 0x31, 0x32, 0x30, 0x31, 0x30, 0x31, 0x30, 0x31, 0x31, 0x33, 0x32, 0x30, 0x33, 0x31, 0x33, 0x30, 0x33, 0x33, 0x30, 0x31, 0x31, 0x30, 0x30, 0x30, 0x33, 0x31, 0x33, 0x33, 0x30, 0x31, 0x31, 0x33, 0x33, 0x30, 0x30, 0x33, 0x32, 0x31, 0x32]

v13 = [0x36, 0x47, 0x31, 0x45, 0x43, 0x50, 0x4B, 0x4E, 0x3E, 0x43, 0x4E, 0x31, 0x38, 0x47, 0x3B, 0x45, 0x44, 0x38, 0x52, 0x51, 0x44, 0x3A, 0x3D, 0x37, 0x3A, 0x40, 0x3B, 0x37, 0x4D, 0x54, 0x45, 0x44, 0x51, 0x41, 0x4B, 0x32, 0x53, 0x47, 0x4D, 0x35, 0x3A, 0x49, 0x48, 0x4E, 0x51, 0x3E, 0x41, 0x45, 0x4D, 0x32]

# v12 = [chr(i) for i in v12]

v13 = [i - 0x30 for i in v13]

print(v13)

for i in range(50):
    a = v12[i]
    b = v13[i]
    if a == 0x31:
        ct[b] ^= 0x2f
    elif a > 0x31:
        if a == 0x32:
            ct[b] -= 51
        elif a == 0x33:
            ct[b] -= 114
    elif a == 0x30:
        ct[b] += 1
ct = [(i & 0xff) for i in ct]
print(bytes(ct))
```

#### Flag: PTITCTF{0bFu5c4Te_4nD_4nT1DeBuG_s0_Ez}
🤑🤑🤑

--Ckagngoc--