---
title: "[Write-Up RE] Flag_Checker KCSC"
date: 2025-06-17
draft: false
description: "a description"
tags: ["WriteUp", "reverse"]
---
## Đề cho như sau:

![Imgur](https://i.imgur.com/s9uBvtP.png)

## Tiến hành

Cho file vào Detect It Easy ta được các thông tin như sau: 

![Imgur](https://i.imgur.com/u2WGKQj.png)

*Packer: UPX*
*Compiler:AutoIT*
*PE_32*

Đầu tiên dùng tool của UPX unpack file 

![Imgur](https://i.imgur.com/HgrbgTC.png)

Sau đó dùng UnautoIT để convert về source file AutoIT 

![Imgur](https://i.imgur.com/Yh4iWv4.png)

Khi chạy thấy chương trình xuất xâu "Incorrect !" Ctrl + F để tìm ta thấy được hàm checker() load một nùi opcode vào để load thành shellcode và check pass ta nhập vào như trên. 
Sau khi chuyển đống opcode vào các param hàm gọi DllCall với param là các string chỉ thị "user32.dll" gọi hàm "CallWindowProcA", và các var chứa shellcode và input: 

![Imgur](https://i.imgur.com/nIAJZEp.png)

Ý tưởng là ta sẽ đặt breakpoint vào hàm *CallWindowProcA* để khi chương trình load các tham số ta sẽ đọc được shellcode bằng IDA. Nhưng khi debug chưa kịp đến chỗ nhập input thì nhận
được msgBox sau:

![Imgur](https://i.imgur.com/FBR5ZEJ.png)

Xem xét các hàm được gọi trong main ta tìm được hàm antiDebug *IsDebuggerPresent*

![Imgur](https://i.imgur.com/Qlu4WbM.png)

Đặt breakpoint tại câu lệnh nhảy để bypass ta tiến hành debug lại chương trình, khi đến BP sửa cờ ZF để bypass.
Lúc này trong Modules view ta thấy chương trình đã load *User32.dll* ta tiến hành tìm và đặt BP trong hàm *CallWindowProcA* dể debug với pass bất kỳ

![Imgur](https://i.imgur.com/NiFdPGq.png)

Lúc này ta trace qua các param để tìm shellcode. Và sử dụng MakeData để xem xét các param, kết quả được như sau: 

![Imgur](https://i.imgur.com/GfL9xXy.png)

![Imgur](https://i.imgur.com/NLKQV2s.png)

Kiểm tra lại trong source AutoIT ta thấy đây chính là đoạn shellcode cần tìm. Vì shellcode là đoạn mã độc lập, thao tác với các param của chương trình qua địa chỉ và sử dụng
các hàm API qua *GetProcAddress*, hàm này sử dụng các string để truy xuất đến các dll nên khi MakeCode trong IDA ta sẽ để ý các đoạn byte là các string liên tục sẽ không được 
make. Tiến hành từ đầu shellcode ta thấy trong hexview có đoạn string liên tục là "user32.dll"

![Imgur](https://i.imgur.com/VAk4LUA.png)

Tiến hành debug vào shellcode ta xem xét các hàm con của hàm trên. Hai hàm đầu là hai hàm load lib của kernel32.dll và các hàm crytoAPI để mã hóa, hàm con cuối cùng là 
messageBoxA nên tiến hành đổi tên chúng cho tiện theo dõi.

![Imgur](https://i.imgur.com/hOkrVlp.png)

Hàm MsgBoxA được load vào [ebp-10h] còn input được mov vào [ebp+8h] sau đó nó call hàm lstrlen(input) và so sánh với 28 nếu khác thì exit.

![Imgur](https://i.imgur.com/WIkyQKK.png)

Chỉnh giá trị của cờ ZF để tiếp tục debug. Sau khi load một số byte "Lạ" chương trình mov input vào eax, offset_CryptAcquireContextA vào ecx, offset_LoadLibraryA vào edx làm
tham số cho hàm *loc_3DD288B*

![Imgur](https://i.imgur.com/0PH9wRE.png)

Trace vào trong tiếp tục Make shellcode như trên ta thu được 1 link youtube "https://www.youtube.com/watch?v=dQw4w9WgXcQ". Tiếp theo chương trình tính độ dài chuỗi input sau đó nó gọi hàm CryptAcquireContextA đơn giản là nó sẽ tạo một con trỏ trỏ đến đối tượng sẽ giúp handle mã hóa nó muốn, ở đây là mã hóa RSA với thông tin trong link sau:
*https://learn.microsoft.com/en-us/windows/win32/seccrypto/prov-rsa-full*, nếu không tạo được sẽ thoát chương trình, viết lại hàm bằng C++ sẽ như sau:

![Imgur](https://i.imgur.com/UaBbHiN.png)

Tiếp theo nó tạo đối tượng băm và băm cái link youtube kia ra

![Imgur](https://i.imgur.com/KrTq83q.png)

Sau đó chương trình sử dụng *CryptDeriveKey* để tạo khóa và *CryptDestroyHash* để giải phóng handle tiếp theo nó mã hóa độ dài input bằng khóa và tạo một vùng nhớ mới chứa input để mã hóa.

![Imgur](https://i.imgur.com/bRPhJYN.png)

Cuối cùng chương trình sử dụng vùng nhớ *new_memories* để so sánh với đoạn byte "Lạ" mà nó load vào lúc đầu. Vì mã hóa tạo ra sau thuật toán khóa và CSP đều gióng nhau sau các lần nên ta có đoạn code decrypt <mượn của bài wu> khác như sau:

# Đoạn giả mã
```
#include <stdio.h>

int main() {
    char inputpass[] = "minh";
    HMODULE user32 = LoadLibraryA(L"user32.dll");
    if(user32 == NULL) {
        exit(0);
    }
    FARPROC msgBoxA = GetProcAddess(user32, "MessageBoxA");
    if(lstrlen(input) != 28) {
        exit(0);
    }
    int len = lstrlen(input);
    // input -> eax
    // offset_CryptAcquireContextA ->ecx
    // offset_LoadLibraryA -> edx

    char link[] = "https://www.youtube.com/watch?v=dQw4w9WgXcQ";

    HCRYPTPROV *phProv; // ebp-0Ch
    // PROV_RSA_FULL = 1
    if(!CryptAcquireContextA(phProv,0,0,PROV_RSA_FULL,0)){
        exit(0);
    };
    //CALG_SHA : 0x8004
    HCRYPTHASH *phHash // ebp+0Ch
    if(!CryptCreateHash(phProv,CALG_SHA,0,0,phHash)){
        exit(0);
    };
    if(!phHash) {
        exit(0);
    }

    // push(link)
    // call lstrlen

    // push lstrlen(link)
    // push offset_link
    // push(*empty_1) is phHash
    // call CryptHashData
    if(!CryptHashData(phHash, link, lstrlen(link),0)) {
        exit(0);
    }

    HCRYPTKEY  *phKey // push empty_memories
    // push 0
    // push phHash
    // push 0x6801 CALG_RC4
    // push phProv

    if(!CryptDeriveKey(phProv,CALG_RC4,phHash,0,phKey)) {
        exit(0);
    }

    // push phProv

    // push phHash
    // call CryptDestroyHash
    CryptDestroyHash(phHash);

    // push 0
    if(!CryptEncrypt(phKey,0,1,0,0,len,0)){
        exit(0);
    }
    // push 4
    // push 0x1000
    // push len + 1
    // push 0
    LPVOID new_memories = VirtualAlloc(0,len+1,0x1000,4);
    if(!new_memories) {
        exit(0);
    }
    for(int i = 0;i<len+1;i++) {
        new_memories[i] = input[i];
    }
    len2 = strlen(new_memories);
    if(!CryptEncrypt(phKey,0,1,0,new_memories,len,len2)){
        exit(0);
    };
    return 0;
}

//ebp-8 : user32
//ebp-10 : MessageBoxA
//ebp+8 : input
//cipherRealPass : ebp-44 -> ebp-29
```

# Decrypt

```
#pragma comment(lib, "crypt32.lib")

#include <stdio.h>
#include <tchar.h>
#include <windows.h>
#include <Wincrypt.h>
#include<string.h>
unsigned char cipher[] =
{
0xF8, 0x50, 0xCC, 0xEF, 0xE6, 0x3C, 0x35, 0x96, 0x1D, 0x61,
0xAE, 0xC0, 0xC5, 0x31, 0xCE, 0xB0, 0xE7, 0x1D, 0xED, 0xBC,
0x5D, 0x81, 0x69, 0x8A, 0x35, 0x74, 0x57, 0xB6
};
int main(void)
{
 HCRYPTPROV hProv;
 if (CryptAcquireContextA(&hProv, 0, 0, PROV_RSA_FULL, 0)) {
     fprintf(stdout, "Success CryptAcquireContextA\n");
     
 };
 HCRYPTHASH phHash;

 if (CryptCreateHash(hProv, CALG_SHA, 0, 0, &phHash)) {
     fprintf(stdout, "Success CreateHash\n");
     
 };

 BYTE text[] = "https://www.youtube.com/watch?v=dQw4w9WgXcQ";

 if (CryptHashData(phHash, text,strlen((const char*)text), 0)) {
     fprintf(stdout, "Success CryptHashData\n PassWord : %s ,\n Length : %d\n",text, strlen((const char*)text));
 };
 HCRYPTKEY phKey;
 if (CryptDeriveKey(hProv, CALG_RC4, phHash, 0, &phKey)) {
     fprintf(stdout, "Success CryptDeriveKey\n");
 };
 CryptDestroyHash(phHash);
 DWORD len1 = 0x1c;
 DWORD len2 = 0x1c;
 //CryptEncrypt(phKey, 0, 1, 0, (BYTE*)newspace, (DWORD*)&len1, len2);
 CryptDecrypt(phKey, 0, 1, 0, (BYTE*)cipher, (DWORD*)&len1);
 printf("flag : %s \n", cipher);



 if(CryptReleaseContext(hProv, 0))
 {
     printf("The handle has been released.\n");
 }
 else
 {
     printf("The handle could not be released.\n");
 }
 return 1;
}
```

Flag: **KCSC{rC4_8uT_1T_L00k2_W31Rd}**