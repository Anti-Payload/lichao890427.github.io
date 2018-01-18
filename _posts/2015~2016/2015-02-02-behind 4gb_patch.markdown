---
layout: post
title: 4gb_patch原理
categories: Reverse_Engineering
description: 4gb_patch原理
keywords: 
---

## 分析
&emsp;&emsp;4gb_patch是允许程序支持4GB空间的小工具  
&emsp;&emsp;话休絮烦，ida载入后直接找到代表WinMain的函数sub_4015A6

```C++
int __stdcall sub_4015A6(HINSTANCE hInstance, int a2, int a3, int a4)  
{  
  const WCHAR *v4; // eax@1  
  const WCHAR *v5; // eax@1  
  ::hInstance = hInstance;  
  v4 = GetCommandLineW();  
  v5 = sub_401218(v4);  
  if ( v5 )  
  {  
    sub_401087(v5);  
  }  
  else if ( sub_4011A1() == 1 )  
  {  
    DialogBoxParamW(hInstance, (LPCWSTR)0x65, 0, DialogFunc, 0);  
  }  
  return 0;  
} 
int __cdecl sub_401087(LPCWSTR lpExistingFileName)  
{  
  HANDLE v1; // eax@1  
  void *v2; // ebx@1  
  int result; // eax@2  
  LPVOID v4; // edi@5  
  WCHAR NewFileName; // [sp+8h] [bp-210h]@1  
  DWORD NumberOfBytesRead; // [sp+210h] [bp-8h]@5  
  SIZE_T dwSize; // [sp+214h] [bp-4h]@3  
   
  sub_40162A(&NewFileName, lpExistingFileName);  
  sub_401638(&NewFileName, L".Backup");  
  CopyFileW(lpExistingFileName, &NewFileName, 1);  
  v1 = CreateFileW(lpExistingFileName, 0xC0000000, 1u, 0, 3u, 0x80u, 0);  
  v2 = v1;  
  if ( v1 == (HANDLE)-1 )  
  {  
    MessageBoxW(0, L"Couldn't open executable!", L"4GB Patch", 0x30u);  
    result = 0;  
  }  
  else 
  {  
    dwSize = GetFileSize(v1, 0);  
    if ( dwSize > 0x8000 )  
      dwSize = 0x8000;  
    v4 = VirtualAlloc(0, dwSize, 0x1000u, 4u);  
    if ( ReadFile(v2, v4, dwSize, &NumberOfBytesRead, 0) )  
    {  
      SetFilePointer(v2, 0, 0, 0);  
      *(_WORD *)((char *)v4 + *((_DWORD *)v4 + 15) + 22) |= 32u;  
      WriteFile(v2, v4, dwSize, &NumberOfBytesRead, 0);  
      CloseHandle(v2);  
      result = 1;  
    }  
    else 
    {  
      VirtualFree(v4, 0, 0x8000u);  
      CloseHandle(v2);  
      MessageBoxW(0, L"Couldn't read executable!", L"4GB Patch", 0x30u);  
      result = 0;  
    }  
  }  
  return result;  
}  
int __usercall sub_4011A1@<eax>(int a1@<ebx>)  
{  
  int result; // eax@2  
  WCHAR ExistingFileName; // [sp+0h] [bp-254h]@1  
  int v3; // [sp+208h] [bp-4Ch]@1  
  int v4; // [sp+20Ch] [bp-48h]@1  
  int v5; // [sp+214h] [bp-40h]@1  
  WCHAR *v6; // [sp+224h] [bp-30h]@1  
  int v7; // [sp+228h] [bp-2Ch]@1  
  int v8; // [sp+238h] [bp-1Ch]@1  
   
  sub_4015ED(a1, &ExistingFileName, 0, 0x208u);  
  sub_4015ED(a1, &v3, 0, 0x4Cu);  
  v4 = 0;  
  v6 = &ExistingFileName;  
  v3 = 76;  
  v5 = (int)L"All Files (*.*)";  
  v7 = 260;  
  v8 = (int)L"Select Executable...";  
  if ( GetOpenFileNameW((LPOPENFILENAMEW)&v3) )  
    result = sub_401087(&ExistingFileName);  
  else 
    result = 2;  
  return result;  
} 

```

&emsp;&emsp;可见小工具提供了命令行和选择2种方式，最后都要执行sub_401087，该函数读取文件并改写pe结构，具体为IMAGE_FILE_HEADER.Characteristics 增加IMAGE_FILE_LARGE_ADDRESSS_AWARE 意为支持2GB以上地址。。
