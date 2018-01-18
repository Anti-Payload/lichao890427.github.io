---
layout: post
title: 获取所有Windows api函数
categories: Windows
description: 获取所有Windows api函数
keywords: 
---

## 获取dumpbin原始输出
`(for %i in (dir /s /b c:\windows\system32\*.dll) do dumpbin -exports %i) >out.txt`   
前200行

```Txt
G:\Users\Administrator\Desktop>dumpbin -exports dir 
Microsoft (R) COFF Binary File Dumper Version 6.00.8168
Copyright (C) Microsoft Corp 1992-1998. All rights reserved.

Dump of file dir
DUMPBIN : fatal error LNK1181: cannot open input file "dir"
G:\Users\Administrator\Desktop>dumpbin -exports /s 
Microsoft (R) COFF Binary File Dumper Version 6.00.8168
Copyright (C) Microsoft Corp 1992-1998. All rights reserved.
DUMPBIN : warning LNK4044: unrecognized option "s"; ignored
  Summary

G:\Users\Administrator\Desktop>dumpbin -exports /b 
Microsoft (R) COFF Binary File Dumper Version 6.00.8168
Copyright (C) Microsoft Corp 1992-1998. All rights reserved.
DUMPBIN : warning LNK4044: unrecognized option "b"; ignored
  Summary

G:\Users\Administrator\Desktop>dumpbin -exports c:\windows\system32\6to4svc.dll 
Microsoft (R) COFF Binary File Dumper Version 6.00.8168
Copyright (C) Microsoft Corp 1992-1998. All rights reserved.

Dump of file c:\windows\system32\6to4svc.dll
File Type: DLL
  Section contains the following exports for 6to4Svc.dll
           0 characteristics
    4B73DA0D time date stamp Thu Feb 11 18:21:01 2010
        0.00 version
           1 ordinal base
           1 number of functions
           1 number of names
    ordinal hint RVA      name
          1    0 00015150 ServiceMain
  Summary
        3000 .data
        2000 .reloc
        1000 .rsrc
       16000 .text
G:\Users\Administrator\Desktop>dumpbin -exports c:\windows\system32\aaaamon.dll 
Microsoft (R) COFF Binary File Dumper Version 6.00.8168
Copyright (C) Microsoft Corp 1992-1998. All rights reserved.

Dump of file c:\windows\system32\aaaamon.dll
File Type: DLL
  Section contains the following exports for AAAAMON.dll
           0 characteristics
    3B7D796D time date stamp Sat Aug 18 04:07:09 2001
        0.00 version
           1 ordinal base
           1 number of functions
           1 number of names
    ordinal hint RVA      name
          1    0 00001B42 InitHelperDll
  Summary
        1000 .data
        1000 .reloc
        3000 .rsrc
        4000 .text
G:\Users\Administrator\Desktop>dumpbin -exports c:\windows\system32\aaclient.dll 
Microsoft (R) COFF Binary File Dumper Version 6.00.8168
Copyright (C) Microsoft Corp 1992-1998. All rights reserved.

Dump of file c:\windows\system32\aaclient.dll
File Type: DLL
  Section contains the following exports for aaclient.dll
           0 characteristics
    47917FE8 time date stamp Sat Jan 19 12:43:20 2008
        0.00 version
           1 ordinal base
           6 number of functions
           4 number of names
    ordinal hint RVA      name
          1    0 0000BA85 LoadClientAdapter
          5    1 0000DEFD OpenKeyReader
          6    2 0000DF0D OpenKeyReaderWriter
          4    3 0001F7F4 g_fnStartTransport
  Summary
        2000 .data
        3000 .reloc
        1000 .rsrc
       1E000 .text
G:\Users\Administrator\Desktop>dumpbin -exports c:\windows\system32\acctres.dll 
Microsoft (R) COFF Binary File Dumper Version 6.00.8168
Copyright (C) Microsoft Corp 1992-1998. All rights reserved.

Dump of file c:\windows\system32\acctres.dll
File Type: DLL
  Summary
        1000 .reloc
       10000 .rsrc
G:\Users\Administrator\Desktop>dumpbin -exports c:\windows\system32\acledit.dll 
Microsoft (R) COFF Binary File Dumper Version 6.00.8168
Copyright (C) Microsoft Corp 1992-1998. All rights reserved.

Dump of file c:\windows\system32\acledit.dll
File Type: DLL
  Section contains the following exports for ACLEDIT.dll
           0 characteristics
    3B7D7CDE time date stamp Sat Aug 18 04:21:50 2001
        0.00 version
           1 ordinal base
           8 number of functions
           8 number of names
    ordinal hint RVA      name
          4    0 00004BC6 DllMain
          1    1 0000323A EditAuditInfo
          2    2 00004010 EditOwnerInfo
          3    3 00003248 EditPermissionInfo
          5    4 00004ED6 FMExtensionProcW
          6    5 0000590A SedDiscretionaryAclEditor
          7    6 0000593F SedSystemAclEditor
          8    7 0000532C SedTakeOwnership
  Summary
        1000 .data
        1000 .reloc
        C000 .rsrc
       13000 .text
G:\Users\Administrator\Desktop>dumpbin -exports c:\windows\system32\aclui.dll 
Microsoft (R) COFF Binary File Dumper Version 6.00.8168
Copyright (C) Microsoft Corp 1992-1998. All rights reserved.

Dump of file c:\windows\system32\aclui.dll
File Type: DLL
  Section contains the following exports for ACLUI.dll
           0 characteristics
    48023D0E time date stamp Mon Apr 14 01:04:14 2008
        0.00 version
           1 ordinal base
          16 number of functions
           3 number of names
    ordinal hint RVA      name
          1    0 0000ACFD CreateSecurityPage
          2    1 0000ADAE EditSecurity
         16    2 0000132C IID_ISecurityInformation
  Summary
        1000 .data
        2000 .reloc
        5000 .rsrc
       12000 .text
G:\Users\Administrator\Desktop>dumpbin -exports c:\windows\system32\activeds.dll 
Microsoft (R) COFF Binary File Dumper Version 6.00.8168
```

## 从原始输出得到函数表
findstr /X ".*[0-9A-F][0-9A-F][0-9A-F][0-9A-F][0-9A-F][0-9A-F][0-9A-F][0-9A-F].*[a-zA-Z_]$" out.txt > out2.txt  
前100行：

```Txt
          1    0 00015150 ServiceMain
          1    0 00001B42 InitHelperDll
          1    0 0000BA85 LoadClientAdapter
          5    1 0000DEFD OpenKeyReader
          6    2 0000DF0D OpenKeyReaderWriter
          4    3 0001F7F4 g_fnStartTransport
          4    0 00004BC6 DllMain
          1    1 0000323A EditAuditInfo
          2    2 00004010 EditOwnerInfo
          3    3 00003248 EditPermissionInfo
          5    4 00004ED6 FMExtensionProcW
          6    5 0000590A SedDiscretionaryAclEditor
          7    6 0000593F SedSystemAclEditor
          8    7 0000532C SedTakeOwnership
          1    0 0000ACFD CreateSecurityPage
          2    1 0000ADAE EditSecurity
         16    2 0000132C IID_ISecurityInformation
          4    0 00004E37 ADsBuildEnumerator
          8    1 00004F79 ADsBuildVarArrayInt
          7    2 00004ECD ADsBuildVarArrayStr
          6    5 00004EAD ADsEnumerateNext
          5    6 00004E89 ADsFreeEnumerator
          3    8 00004723 ADsGetObject
          9    9 000051CD ADsOpenObject
         23    B 00013790 AdsFreeAdsValues
         22    C 0001376E AdsTypeToPropVariant
         29   10 00005020 BinarySDToSecurityDescriptor
         27   11 00016889 ConvertSecDescriptorToVariant
         28   12 000158F6 ConvertSecurityDescriptorToSecDes
         10   13 000040B2 DllCanUnloadNow
         11   14 0000405A DllGetClassObject
         31   15 000040CC DllRegisterServer
         32   16 000043DB DllUnregisterServer
         21   19 00013A20 PropVariantToAdsType
         30   1D 00005064 SecurityDescriptorToBinarySD
          4    0 00001100 DllCanUnloadNow
          5    1 00010C20 DllGetClassObject
          6    2 00011776 DllRegisterServer
          7    3 000117A2 DllUnregisterServer
          3    4 0001172B GetProxyDllInfo
         15    0 00008470 AdmClose
         16    1 000082ED AdmFinished
         17    2 00008D61 AdmInit
         18    3 00008337 AdmReset
         19    4 000084EE AdmSaveData
         20    5 0000927A CheckDuplicateKeys
         21    6 00008F22 CreateAdmUi
         12    7 00008202 DllMain
         22    8 00009071 GetAdmCategories
         23    9 00009654 GetFontInfo
         13    A 000084D6 IsAdmDirty
         14    B 000084E1 ResetAdmDirtyFlag
          1    0 00002209 CreateSocketPort
          2    1 000022BC DeleteSocketPort
          3    2 00004384 FwBindFwInterfaceToAdapter
          4    3 000045B0 FwConnectionRequestFailed
          5    4 000040E1 FwCreateInterface
          6    5 00004195 FwDeleteInterface
          7    6 00004504 FwDisableFwInterface
          8    7 0000455A FwEnableFwInterface
          9    8 0000427B FwGetInterface
         10    9 0000464E FwGetNotificationResult
         11    A 000047A8 FwGetStaticNetbiosNames
         12    B 00003C81 FwIsStarted
         13    C 00004606 FwNotifyConnectionRequest
         14    D 000041EB FwSetInterface
         15    E 00004737 FwSetStaticNetbiosNames
         16    F 00003940 FwStart
         17   10 00003F48 FwStop
         18   11 000044AE FwUnbindFwInterfaceFromAdapter
         19   12 00004079 FwUpdateConfig
         20   13 0000466F FwUpdateRouteTable
         21   14 00002B3B GetAdapterNameFromMacAddrW
         22   15 00002B1E GetAdapterNameW
         23   16 0000495E GetFilters
         24   17 00002443 IpxAdjustIoCompletionParams
         25   18 0000358B IpxCreateAdapterConfigurationPort
         26   19 000032B7 IpxDeleteAdapterConfigurationPort
         27   1A 00002968 IpxDoesRouteExist
         28   1B 00003746 IpxGetAdapterConfig
         29   1C 00002763 IpxGetAdapterList
         30   1D 0000234B IpxGetOverlappedResult
         31   1E 000025FF IpxGetQueuedAdapterConfigurationStatus
         32   1F 000023BC IpxGetQueuedCompletionStatus
         33   20 000024C5 IpxPostQueuedCompletionStatus
         34   21 00002593 IpxRecvPacket
         35   22 000024E3 IpxSendPacket
         36   23 00003699 IpxWanCreateAdapterConfigurationPort
         37   24 00002BEC IpxWanQueryInactivityTimer
         38   25 00002B40 IpxWanSetAdapterConfiguration
         39   26 00005506 ServiceMain
         40   27 000048BF SetFilters
          1    0 00003EC1 DllCanUnloadNow
          2    1 00003EE4 DllGetClassObject
          1    0 00018C7C ??0CLexer@@QAE@PAG@Z
          2    1 00018CC0 ??1CLexer@@QAE@XZ
         54    2 000192A0 ?GetNextToken@CLexer@@QAEJPAGPAK@Z
        135    3 00018E38 ?SetAtDisabler@CLexer@@QAEXH@Z
        136    4 00018E60 ?SetExclaimnationDisabler@CLexer@@QAEXH@Z
        137    5 00018E4C ?SetFSlashDisabler@CLexer@@QAEXH@Z
```

## 去除重复和无效

findstr /V "@  $ \. : DllCanUnloadNow DllGetClassObject DllRegisterServer DllUnregisterServer ServiceMain DllMain CPlApplet DllGetVersion DllInstall" out2.txt > out3.txt  
前50行：
```Txt
        1    0 00001B42 InitHelperDll
          1    0 0000BA85 LoadClientAdapter
          5    1 0000DEFD OpenKeyReader
          6    2 0000DF0D OpenKeyReaderWriter
          4    3 0001F7F4 g_fnStartTransport
          1    1 0000323A EditAuditInfo
          2    2 00004010 EditOwnerInfo
          3    3 00003248 EditPermissionInfo
          5    4 00004ED6 FMExtensionProcW
          6    5 0000590A SedDiscretionaryAclEditor
          7    6 0000593F SedSystemAclEditor
          8    7 0000532C SedTakeOwnership
          1    0 0000ACFD CreateSecurityPage
          2    1 0000ADAE EditSecurity
         16    2 0000132C IID_ISecurityInformation
          4    0 00004E37 ADsBuildEnumerator
          8    1 00004F79 ADsBuildVarArrayInt
          7    2 00004ECD ADsBuildVarArrayStr
          6    5 00004EAD ADsEnumerateNext
          5    6 00004E89 ADsFreeEnumerator
          3    8 00004723 ADsGetObject
          9    9 000051CD ADsOpenObject
         23    B 00013790 AdsFreeAdsValues
         22    C 0001376E AdsTypeToPropVariant
         29   10 00005020 BinarySDToSecurityDescriptor
         27   11 00016889 ConvertSecDescriptorToVariant
         28   12 000158F6 ConvertSecurityDescriptorToSecDes
         21   19 00013A20 PropVariantToAdsType
         30   1D 00005064 SecurityDescriptorToBinarySD
          3    4 0001172B GetProxyDllInfo
         15    0 00008470 AdmClose
         16    1 000082ED AdmFinished
         17    2 00008D61 AdmInit
         18    3 00008337 AdmReset
         19    4 000084EE AdmSaveData
         20    5 0000927A CheckDuplicateKeys
         21    6 00008F22 CreateAdmUi
         22    8 00009071 GetAdmCategories
         23    9 00009654 GetFontInfo
         13    A 000084D6 IsAdmDirty
         14    B 000084E1 ResetAdmDirtyFlag
          1    0 00002209 CreateSocketPort
          2    1 000022BC DeleteSocketPort
          3    2 00004384 FwBindFwInterfaceToAdapter
          4    3 000045B0 FwConnectionRequestFailed
          5    4 000040E1 FwCreateInterface
          6    5 00004195 FwDeleteInterface
          7    6 00004504 FwDisableFwInterface
          8    7 0000455A FwEnableFwInterface
          9    8 0000427B FwGetInterface

```

## 去除地址
前25行  
```Txt
InitHelperDll
LoadClientAdapter
OpenKeyReader
OpenKeyReaderWriter
g_fnStartTransport
CPlApplet
DebugMain
EditAuditInfo
EditOwnerInfo
EditPermissionInfo
FMExtensionProcW
SedDiscretionaryAclEditor
SedSystemAclEditor
SedTakeOwnership
CreateSecurityPage
EditSecurity
IID_ISecurityInformation
ADsBuildEnumerator
ADsBuildVarArrayInt
ADsBuildVarArrayStr
ADsEnumerateNext
ADsFreeEnumerator
ADsGetObject
ADsOpenObject
AdsFreeAdsValues
```

## 结论
对out4.txt进行正则搜索[a-zA-Z_0-9]{60,}，发现最长的api是66个字母