---
layout: post
title: 分析360文件粉碎过程
categories: WindowsDriver
description: 分析360文件粉碎过程
keywords: 
---

# 分析360文件粉碎过程

## 360删除普通文件

调用栈

```Txt
b1e20a10 f83d5b3b Ntfs!NtfsCommonSetInformation
b1e20a78 804ef119 Ntfs!NtfsFsdSetInformation+0xa3
b1e20a88 f8483f45 nt!IopfCallDriver+0x31
b1e20a9c 804ef119 sr!SrSetInformation+0x179
b1e20aac f848be9b nt!IopfCallDriver+0x31
b1e20ad0 f848c06b fltMgr!FltpLegacyProcessingAfterPreCallbacksCompleted+0x20b
b1e20b08 804ef119 fltMgr!FltpDispatch+0x11f
b1e20b18 b1bae341 nt!IopfCallDriver+0x31
b1e20ba8 804ef119 qutmdrv+0x12341
b1e20bb8 80571809 nt!IopfCallDriver+0x31
b1e20c68 b207cfce nt!NtSetInformationFile+0x585
b1e20cc0 80523bc1 bd0001+0x5fce
b1e20d48 8053e638 nt!ObfDereferenceObject+0x5f
b1e20d48 7c92e4f4 nt!KiFastCallEntry+0xf8
0278fb08 7c92dc4c ntdll!KiFastSystemCallRet
0278fb0c 7c83203c ntdll!ZwSetInformationFile+0xc
0278fb80 00410f79 kernel32!DeleteFileW+0x23f
0278fba8 00411913 360tray+0x10f79
0278fbd8 00411c19 360tray+0x11913
0278fc50 7c930202 360tray+0x11c19
```

## 360删除被占用文件

先尝试普通文件删除，如果返回STATUS_CANNOT_DELETE则：

```Txt
b1de47d4 f83d5b3b Ntfs!NtfsCommonSetInformation                        
b1de483c b1c37346 Ntfs!NtfsFsdSetInformation+0xa3
b1de4854 b1c383e7 BAPIDRV+0x1f346
b1de4890 b1c2494d BAPIDRV+0x203e7
b1de48c0 b1c27b5a BAPIDRV+0xc94d
b1de48e8 b1c299e7 BAPIDRV+0xfb5a
b1de4b3c b1c29d00 BAPIDRV+0x119e7
b1de4b6c 804ef119 BAPIDRV+0x11d00
b1de4b7c 80575d5e nt!IopfCallDriver+0x31
b1de4b90 80576bff nt!IopSynchronousServiceTail+0x70
b1de4c38 8056f46c nt!IopXxxControlFile+0x5e7                        
b1de4c6c 81515799 nt!NtDeviceIoControlFile+0x2a                        ioctlcode=8899e810
b1de4d34 8053e638 0x81515799
b1de4d34 7c92e4f4 nt!KiFastCallEntry+0xf8
0278fa54 7c92d26c ntdll!KiFastSystemCallRet
0278fa58 02791afe ntdll!NtDeviceIoControlFile+0xc
0278fab8 0279259b bapi!BfsSetFileApisToANSI+0xade
0278faec 02793740 bapi!BfsSetFileApisToANSI+0x157b
00000000 00000000 bapi!BfsSetFileApisToANSI+0x2720
```

如果再返回STATUS_CANNOT_DELETE则：

```Txt
b1de4700 f83d5b3b Ntfs!NtfsCommonSetInformation
b1de4768 b1c37346 Ntfs!NtfsFsdSetInformation+0xa3
b1de4780 b1c24b11 BAPIDRV+0x1f346
b1de47c0 b1c24c76 BAPIDRV+0xcb11
b1de4820 b1c26056 BAPIDRV+0xcc76
b1de484c b1c29ba7 BAPIDRV+0xe056
b1de4aa0 b1c29d00 BAPIDRV+0x11ba7
b1de4ad0 804ef119 BAPIDRV+0x11d00
b1de4ae0 80575d5e nt!IopfCallDriver+0x31
b1de4af4 80576bff nt!IopSynchronousServiceTail+0x70                
b1de4b9c 8056f46c nt!IopXxxControlFile+0x5e7
b1de4bd0 b1c35cd0 nt!NtDeviceIoControlFile+0x2a                        ioctlcode=8899e824
b1de4c04 b1c35ddd BAPIDRV+0x1dcd0
b1de4c18 b1d79d99 BAPIDRV+0x1dddd
b1de4c74 805b79ef Hookport+0x8d99
b1de4d34 8053e638 nt!ObpFreeObject+0x12f
b1de4d34 7c92e4f4 nt!KiFastCallEntry+0xf8
0278fa84 7c92d26c ntdll!KiFastSystemCallRet
0278fa88 02791afe ntdll!NtDeviceIoControlFile+0xc
0278fae8 02795f05 bapi!BfsSetFileApisToANSI+0xade
```

## 360删除百度文件

```Txt
b1e287d4 f83d5b3b Ntfs!NtfsCommonSetInformation
b1e2883c b1c37346 Ntfs!NtfsFsdSetInformation+0xa3
b1e28854 b1c383e7 BAPIDRV+0x1f346
b1e28890 b1c2494d BAPIDRV+0x203e7
b1e288c0 b1c27b5a BAPIDRV+0xc94d
b1e288e8 b1c299e7 BAPIDRV+0xfb5a
b1e28b3c b1c29d00 BAPIDRV+0x119e7
b1e28b6c 804ef119 BAPIDRV+0x11d00
b1e28b7c 80575d5e nt!IopfCallDriver+0x31
b1e28b90 80576bff nt!IopSynchronousServiceTail+0x70                
b1e28c38 8056f46c nt!IopXxxControlFile+0x5e7
b1e28c6c 81515799 nt!NtDeviceIoControlFile+0x2a                        ioctlcode=8899e810
b1e28d34 8053e638 0x81515799
b1e28d34 7c92e4f4 nt!KiFastCallEntry+0xf8
0278fa54 7c92d26c ntdll!KiFastSystemCallRet
0278fa58 02791afe ntdll!NtDeviceIoControlFile+0xc
0278fab8 0279259b bapi!BfsSetFileApisToANSI+0xade
0278faec 02793740 bapi!BfsSetFileApisToANSI+0x157b
00000000 00000000 bapi!BfsSetFileApisToANSI+0x2720
```
