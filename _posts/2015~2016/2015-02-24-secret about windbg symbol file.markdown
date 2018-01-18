---
layout: post
title: windbg符号文件夹中像MD5的文件夹名如何生成？
categories: Reverse_Engineering
description: windbg符号文件夹中像MD5的文件夹名如何生成？
keywords: windbg
---

# Windbg符号文件夹中像MD5的文件夹名如何生成？

## 任务：找到windbg pdb符号对应文件夹名长串编码含义？
&emsp;&emsp;解决方法是用windbg调试windbg，下断点: 
```Txt
bp kernelbase!CreateFileA "da poi(esp+4);k;gc"
bp kernelbase!CreateFileW "du poi(esp+4);k;gc"
```

得到如下记录：
```Txt
0938e15c  "d:\symbol\cmd.pdb\A609EC1CBCFF43"
0938e19c  "43AEFC312495558D832\cmd.pdb"
ChildEBP RetAddr  
04d6a670 70af02cb KERNELBASE!CreateFileW
04d6a6ac 70af1a7e dbghelp!IStreamCRTFile::Create+0x8b
04d6a6d4 70af08c5 dbghelp!MSF_HB::internalOpen+0x2e
04d6a6f0 70ae8d47 dbghelp!MSF::Open+0x35
04d6a724 70ae90df dbghelp!PDB1::OpenEx2W+0xc7
04d6a74c 70af335f dbghelp!PDB::OpenEx2W+0x1f
04d6a770 70a91294 dbghelp!PDBCommon::Open2W+0x1f
04d6a794 70a3af14 dbghelp!CDiaDataSource::loadDataFromPdb+0x44
04d6abf8 70a56f4c dbghelp!diaGetPdb+0x1fc
04d6ae28 70a55bdd dbghelp!GetDebugData+0x230
04d6b2c8 70a55885 dbghelp!modload+0x285
04d6b2f4 70a4c177 dbghelp!LoadSymbols+0x355
04d6b328 70a4d44d dbghelp!ModLoop+0x86
04d6d2e4 70a50ef8 dbghelp!EnumSymbols+0xec
04d6d318 70d0fabf dbghelp!SymEnumSymbolsExW+0x59
04d6d480 70d0fbf7 dbgeng!EnumModuleTypedData+0xef
04d6e170 70d1024c dbgeng!EnumAllModuleTypedData+0xf4
04d6e3b8 70cca72b dbgeng!ParseExamine+0x504
04d6e428 70cca955 dbgeng!ProcessCommands+0x11c3
04d6e488 70c3cdd4 dbgeng!ProcessCommandsAndCatch+0x91
04d6e8f4 70c3cfc5 dbgeng!Execute+0x226
04d6e944 00dde653 dbgeng!DebugClient::ExecuteWide+0x8d
04d6ed08 00ddea93 windbg!ProcessCommand+0x145
04d6fd28 00de0275 windbg!ProcessEngineCommands+0xd1
04d6fd40 7567919f windbg!EngineLoop+0x390
04d6fd4c 77b0b5af KERNEL32!BaseThreadInitThunk+0xe
04d6fd94 77b0b57a ntdll!__RtlUserThreadStart+0x2f
04d6fda4 00000000 ntdll!_RtlUserThreadStart+0x1b
```

&emsp;&emsp;现在来被调试的windbg调试windows\syswow64\cmd.exe，由于有了pdb，分析过程变得极为容易，跟踪便可以知道modload中首次初始化路径cmd.pdb，而diaGetPdb中该地址最终变幻成`d:\symbol\cmd.pdb\A609EC1CBCFF4343AEFC312495558D832\cmd.pdb`，现在就是要知道A609EC1CBCFF4343AEFC312495558D832从何而来，因此从diaGetPdb入手，跟踪后可以发现dbghelpb把内存中PE的PROCESS_ENTRY结构和文件名以及符号服务器等传给了symsrv.dll，symsrv!SymbolServerWEx完成返回完整路径的任务，这个任务包括：
* TestParameters函数检测传入参数合法性
* SymbolServerGetIndexStringW函数返回index，也就是这里的A609EC1CBCFF4343AEFC312495558D832
* SymbolServerByIndexW根据index返回pdb完整路径
* 解析出SymbolServerByIndexW的参数：WCHAR *srv, WCHAR *pdbname, BYTE *indexdata, int dword1, int dword2, WCHAR *outpath,PDWORD unknown
* 解析出SymbolServerGetIndexStringW的参数：WCHAR* srv,WCHAR* pdbname(无路径),BYTE* index ,int dword1,int dword2,WCHAR* formatted,int len，发现index经过变换后即是结果，如wntdll.pdb。  

传入：8365a747 c534bc47 a03e6293 a78c675f  
结果：47A76583 34C547BC A03E6293 A78C675F2  

现有任务变为2个：
* 1.传入index源自何处？这需要向上层调用寻找
* 2.末尾2如何计算？这需要分析SymbolServerGetIndexStringW


## 解决问题1
&emsp;&emsp;逐层向上标记参数，  
`symsrv_SymbolServerWEx —> CallSymbolServerGetFile -> symsrvGetFile -> HandleLocatePdbInSymSrvOrLocalStore`  
&emsp;&emsp;到这一层后发现参数只有类指针this，我们需要关注symsrvGetFile的对应参数indexdata,param1,param2，indexdata位于this+27780处，dword1为*(DWORD*)(this+24624)，dword2为0，需要了解这些位置在什么地方进行了改动，在往上层进入了线程回调函数 GetPdbThreadProc，这里需要看看哪里调用的线程，并找到传递的参数之前在哪里改变的，到了DiaLocatePdbMultiThread，伪代码：

```C++
v17 = *(this + 4052);  
if ( *(v17 + 4) != 3 || *(v17 + 12) != 20 )  
{  
  dword1__________ = param6;//GUID*  
  v18 = param4;  
}  
else 
{  
  v18 = *(v17 + 8);  
  dword1__________ = *(v18 + 4);  
}  
if ( param5 )//GUID*  
{  
  indexdata = param5;  
}  
else if ( v18 )  
{  
  indexdata = *v18;//param+2180  
} 
```

&emsp;&emsp;再向上层看，diaGetPdb中
```Txt
param4=this+2180 param5=*(this+2172) param6=*(this+2168)
RetrievePdbInfo  
this+2180 => thisb+4 
this+2172 => 0
this+2168 => thisb+20
thisb=*(DWORD*)(this+3252)
```

需要知道this+3252何时写入：


```C++
int __fastcall ReadHeader(int thisc, unsigned __int32 a2)  
{  
  unsigned __int32 v2; // ebx@1  
  int v3; // esi@1  
  QWORD v5; // rax@5  
  struct _IMGHLP_DEBUG_DATA *pebegin2; // edi@5  
  DWORD filesize; // eax@5  
  BYTE *v8; // eax@6  
  void *v9; // eax@7  
  void *v10; // ecx@7  
  unsigned __int64 v11; // ST10_8@9  
  unsigned __int64 v12; // ST10_8@13  
  __int16 v13; // cx@14  
  LPVOID v14; // eax@21  
  int v15; // edi@21  
  int v16; // ecx@22  
  int v17; // edx@24  
  int v18; // eax@24  
  char v19; // di@24  
  int v20; // ecx@27  
  unsigned __int64 v21; // ST10_8@31  
  struct _IMGHLP_DEBUG_DATA *v22; // edx@37  
  unsigned __int64 v23; // ST10_8@42  
  unsigned __int64 v24; // ST10_8@43  
  signed int PESIG; // edx@44  
  int v26; // eax@45  
  LPVOID v27; // ecx@45  
  unsigned __int64 v28; // ST10_8@47  
  unsigned __int64 v29; // ST10_8@50  
  unsigned __int64 v30; // ST10_8@61  
  int imagesize; // eax@65  
  void *v32; // eax@74  
  LPVOID sectiondata; // eax@74  
  int v34; // ecx@75  
  BYTE *v35; // ecx@76  
  int v36; // ecx@76  
  const void *secheader; // eax@76  
  unsigned __int32 v38; // edx@83  
  unsigned __int64 v39; // ST10_8@84  
  unsigned __int32 v40; // eax@85  
  unsigned __int64 v41; // ST10_8@88  
  LPVOID v42; // eax@98  
  int v43; // edx@116  
  unsigned __int32 v44; // [sp-4h] [bp-3A4h]@116  
  void *v45; // [sp+0h] [bp-3A0h]@0  
  void *v46; // [sp+0h] [bp-3A0h]@9  
  unsigned __int32 v47; // [sp+0h] [bp-3A0h]@13  
  void *v48; // [sp+0h] [bp-3A0h]@42  
  unsigned __int32 v49; // [sp+0h] [bp-3A0h]@43  
  void *v50; // [sp+0h] [bp-3A0h]@83  
  struct _IMAGE_SECTION_HEADER *v51; // [sp+0h] [bp-3A0h]@85  
  unsigned int v52; // [sp+4h] [bp-39Ch]@0  
  unsigned int v53; // [sp+4h] [bp-39Ch]@9  
  unsigned __int32 v54; // [sp+4h] [bp-39Ch]@13  
  unsigned int v55; // [sp+4h] [bp-39Ch]@42  
  unsigned __int32 v56; // [sp+4h] [bp-39Ch]@43  
  unsigned int v57; // [sp+4h] [bp-39Ch]@83  
  struct _IMAGE_DATA_DIRECTORY *v58; // [sp+4h] [bp-39Ch]@85  
  int v59; // [sp+14h] [bp-38Ch]@76  
  int filesize2; // [sp+24h] [bp-37Ch]@74  
  unsigned __int32 filesize2a; // [sp+24h] [bp-37Ch]@87  
  DWORD filesize2_4; // [sp+28h] [bp-378h]@7  
  BYTE *exportpos; // [sp+2Ch] [bp-374h]@1  
  unsigned __int64 v64; // [sp+30h] [bp-370h]@1  
  int v65; // [sp+38h] [bp-368h]@9  
  BYTE *v66; // [sp+3Ch] [bp-364h]@1  
  unsigned int v67; // [sp+40h] [bp-360h]@1  
  void *sectiondata_1; // [sp+44h] [bp-35Ch]@21  
  LPVOID lpMem; // [sp+48h] [bp-358h]@1  
  void *v70; // [sp+4Ch] [bp-354h]@5  
  int v71; // [sp+50h] [bp-350h]@5  
  struct _IMGHLP_DEBUG_DATA *pebegin; // [sp+54h] [bp-34Ch]@5  
  BYTE v73[28]; // [sp+58h] [bp-348h]@31  
  BYTE v74[264]; // [sp+74h] [bp-32Ch]@61  
  unsigned __int64 v75; // [sp+17Ch] [bp-224h]@88  
  char v76; // [sp+18Ch] [bp-214h]@89  
  int v77; // [sp+1C0h] [bp-1E0h]@88  
  BYTE v78[248]; // [sp+1C4h] [bp-1DCh]@1  
  unsigned __int16 v79[40]; // [sp+2BCh] [bp-E4h]@50  
  BYTE Dst[64]; // [sp+30Ch] [bp-94h]@1  
  unsigned __int64 v81; // [sp+350h] [bp-50h]@13  
  int v82; // [sp+358h] [bp-48h]@17  
  int v83; // [sp+35Ch] [bp-44h]@17  
  int v84; // [sp+360h] [bp-40h]@18  
  int v85; // [sp+364h] [bp-3Ch]@20  
  void *v86; // [sp+368h] [bp-38h]@21  
  int v87; // [sp+36Ch] [bp-34h]@24  
  unsigned int v88; // [sp+370h] [bp-30h]@23  
  int v89; // [sp+374h] [bp-2Ch]@17  
  CPPEH_RECORD ms_exc; // [sp+388h] [bp-18h]@9  
   
  v2 = a2;  
  v3 = thisc;  
  *Dst = 0;  
  memset(&Dst[2], 0, 0x3Eu);  
  *v78 = 0;  
  memset(&v78[4], 0, 0xF4u);  
  lpMem = 0;  
  v66 = 0;  
  HIDWORD(v64) = 0;  
  exportpos = 0;  
  v67 = 0;  
  if ( v2 == 1 )  
  {  
    v9 = *(v3 + 3412);  
    v71 = *(v3 + 3412);  
    pebegin2 = *(v3 + 16);  
    pebegin = *(v3 + 16);  
    v10 = *(v3 + 20);  
    v70 = *(v3 + 20);  
    filesize2_4 = 0;  
    *(v3 + 3248) = 5;  
  }  
  else 
  {  
    if ( v2 == 2 )  
    {  
      v71 = 0;  
      v8 = MapItRO(*(v3 + 1096), 0);  
      *(v3 + 1104) = v8;  
      pebegin2 = v8;  
      v70 = (v8 >> 32);  
      pebegin = v8;  
      filesize = GetFileSize(*(v3 + 1096), 0);  
      *(v3 + 3248) = 2;  
    }  
    else 
    {  
      if ( v2 != 3 )  
        return 0;  
      v71 = 0;  
      LODWORD(v5) = MapItRO(*(v3 + 2156), 0);  
      *(v3 + 2160) = v5;  
      pebegin2 = v5;  
      v70 = (v5 >> 32);  
      pebegin = v5;  
      filesize = GetFileSize(*(v3 + 2156), 0);  
      *(v3 + 3248) = 3;  
    }  
    filesize2_4 = filesize;  
    v9 = v71;  
    v10 = v70;  
  }  
  *(v3 + 4024) = 0;  
  v65 = 0;  
  ms_exc.registration.TryLevel = 0;  
  HIDWORD(v11) = 2;  
  LODWORD(v11) = &v64;  
  if ( !ReadImageData(v9, pebegin2, v10, 0i64, v11, v45, v52) )// 读取2个字节  
    goto LABEL_10;  
  *(v3 + 3240) = v2;  
  if ( v64 != 'ID' )  
  {  
    if ( v64 == 'ZM' )                          // 如果是MZ头  
    {  
      HIDWORD(v23) = 64;  
      LODWORD(v23) = Dst;  
      if ( !ReadImageData(v71, pebegin2, v70, 0i64, v23, v46, v53) )// 读取64字节以取得PE头所在位置  
        goto LABEL_10;  
      HIDWORD(v24) = 248;  
      LODWORD(v24) = v78;  
      if ( !ReadImageData(v71, pebegin2, v70, *&Dst[60], v24, v48, v55) )// 读取248字节到v93  
        goto LABEL_10;  
      PESIG = *v78;  
      if ( *v78 != 'EP' )                       // 如果是PE头  
      {  
        v66 = &v78[4];  
        v26 = *&v78[20] + 24 + *&Dst[60];  
        v27 = &v78[24];  
LABEL_55:  
        HIDWORD(v64) = v26;  
        goto LABEL_56;  
      }  
    }  
    else 
    {  
      if ( v64 != 0x14C )  
      {  
        HIDWORD(v29) = 76;  
        LODWORD(v29) = v79;  
        if ( !ReadImageData(v71, pebegin2, v70, 0i64, v29, v46, v53) )  
          goto LABEL_10;  
        if ( v79[0] != 332 && v79[0] != 388 && v79[0] != 644 )  
          goto LABEL_11;  
        v27 = &v79[10];  
        v66 = v79;  
        v26 = v79[8] + 20;  
        PESIG = *v78;  
        goto LABEL_55;  
      }  
      HIDWORD(v28) = 244;  
      LODWORD(v28) = &v78[4];  
      if ( !ReadImageData(v71, pebegin2, v70, 0i64, v28, v46, v53) )  
        goto LABEL_11;  
      PESIG = 'ROM ';  
      *v78 = 'ROM ';  
    }  
    v27 = lpMem;  
LABEL_56:  
    if ( v27 )  
    {  
      if ( *v27 != 0x107 )  
      {  
        *(v3 + 4592) = 11;  
        goto LABEL_11;  
      }  
      v19 = 1;  
      *(v3 + 3400) = 1;  
      *(v3 + 1108) = *v27;  
      *(v3 + 24) = *(v27 + 5);  
      *(v3 + 28) = 0;  
      *(v3 + 32) = *(v27 + 1);  
      *(v3 + 36) = 0;  
      goto LABEL_74;  
    }  
    if ( *&v78[24] != 0x20B )                   // IMAGE_OPTIONAL_HEADER32.MAGIC=PE64  
    {  
      v66 = &v78[4];  
      exportpos = &v78[120];                    // 导出表位置  
      *(v3 + 1108) = *&v78[24];  
      if ( PESIG == 'ROM ' )  
        HIDWORD(v64) = 244;  
      else 
        HIDWORD(v64) = *&Dst[60] + 248;         // 第一个节位置  
      v19 = 1;  
      if ( v2 == 2 || v2 == 1 )  
      {  
        *(v3 + 24) = *&v78[52];  
        *(v3 + 28) = 0;  
        *(v3 + 3392) = *&v78[56];  
        *(v3 + 36) = *&v78[88];  
      }  
      imagesize = *&v78[80];  
      goto LABEL_73;  
    }  
    HIDWORD(v30) = 264;  
    LODWORD(v30) = v74;  
    if ( ReadImageData(v71, pebegin2, v70, *&Dst[60], v30, v49, v56) )  
    {  
      v66 = &v74[4];  
      exportpos = &v74[136];  
      HIDWORD(v64) = *&Dst[60] + 264;  
      *(v3 + 1108) = *&v74[24];  
      v19 = 1;  
      *(v3 + 3396) = 1;  
      if ( v2 == 2 || v2 == 1 )  
      {  
        *(v3 + 24) = *&v74[48];  
        *(v3 + 28) = *&v74[52];  
        *(v3 + 3392) = *&v74[56];  
        *(v3 + 36) = *&v74[88];  
      }  
      imagesize = *&v74[80];  
LABEL_73:  
      *(v3 + 32) = imagesize;  
LABEL_74:  
      imgset(100, *(v3 + 4036), v2, v2, v49, v56);  
      v32 = *(v66 + 1);                         // 得到节个数  
      lpMem = v32;  
      filesize2 = 40 * v32;  
      sectiondata = pMemAlloc(40 * v32);  
      sectiondata_1 = sectiondata;  
      if ( sectiondata && ReadImageData(v71, pebegin, v70, HIDWORD(v64), __PAIR__(filesize2, sectiondata), v47, v54) )// 读取所有节数据  
      {  
        ImageHelpPointerWrapper<_IMAGE_SECTION_HEADER>::Assign((v3 + 3352), sectiondata_1, lpMem, v34);  
        *(v3 + 3340) = sectiondata_1;  
        *(v3 + 3380) = lpMem;  
        v35 = v66;  
        *(v3 + 48) = *v66;  
        *(v3 + 40) = *(v35 + 1);  
        *(v3 + 44) = *(v35 + 9);  
        imgset(101, *(v3 + 4036), v2, v2, v47, v54);  
        v36 = 0;  
        v59 = 0;  
        secheader = sectiondata_1;  
        while ( v36 < lpMem )  
        {  
          if ( *(v3 + 3400) && !(*(v66 + 9) & 0x200) )  
          {  
            if ( !memcmp(secheader, ".rdata", 7u) )// 遇到.rdata不再读取节信息  
            {  
              v20 = 1;  
              v67 = 1;  
              v18 = *(sectiondata_1 + 3);  
              v65 = *(sectiondata_1 + 3);  
              goto LABEL_28;  
            }  
            secheader = sectiondata_1;  
          }  
          v38 = SectionContains(secheader, v71, exportpos, v47, v54);// 计算导出表在文件中的位置  
          if ( v38 )  
          {  
            *(v3 + 3944) = v2;  
            *(v3 + 4048) = *(exportpos + 1);  
            *(v3 + 4040) = *exportpos;  
            *(v3 + 4044) = 0;  
            HIDWORD(v39) = 40;  
            LODWORD(v39) = v3 + 3984;  
            ReadImageData(v71, pebegin, v70, v38, v39, v50, v57);// 读40字节  
          }  
          v40 = SectionContains(sectiondata_1, v71, exportpos + 12, v50, v57);// 计算调试信息在文件中的位置  
          if ( v40 )  
          {  
            v65 = v40;                          // v65=调试信息偏移  
            v67 = *(exportpos + 13) / 28u;      // 调试信息个数  
          }  
          filesize2a = SectionContains(sectiondata_1, v71, exportpos + 28, v51, v58);// 计算CLR信息在文件中的位置  
          if ( filesize2a )  
          {  
            memset(&v75, 0, 0x48u);  
            HIDWORD(v41) = 72;  
            LODWORD(v41) = &v75;  
            ReadImageData(v71, pebegin, v70, filesize2a, v41, v47, v54);  
            if ( v77 || v76 & 1 )  
              *(v3 + 4648) = 1;  
          }  
          v36 = v59++ + 1;  
          secheader = sectiondata_1 + 40;       // 取下一个section头  
          sectiondata_1 = sectiondata_1 + 40;  
        }  
      }  
      goto LABEL_26;  
    }  
LABEL_10:  
    dword_6BC5FB1C = 6;  
LABEL_11:  
    ms_exc.registration.TryLevel = -2;  
    return 0;  
  }  
  HIDWORD(v12) = 48;  
  LODWORD(v12) = &v81;  
  if ( !ReadImageData(v71, pebegin2, v70, 0i64, v12, v46, v53) )  
    goto LABEL_11;  
  v13 = WORD2(v81);  
  if ( WORD2(v81) != 332 && WORD2(v81) != 388 )  
  {  
    UnmapViewOfFile(*(v3 + 2160));  
    *(v3 + 2160) = 0;  
    goto LABEL_11;  
  }  
  *(v3 + 3392) = v89;  
  *(v3 + 36) = v83;  
  *(v3 + 48) = v13;  
  *(v3 + 40) = v82;  
  *(v3 + 44) = WORD3(v81);  
  if ( !*(v3 + 24) )  
  {  
    *(v3 + 24) = v84;  
    *(v3 + 28) = 0;  
  }  
  if ( !*(v3 + 32) )  
    *(v3 + 32) = v85;  
  lpMem = v86;  
  HIDWORD(v64) = 40 * v86;  
  v14 = pMemAlloc(40 * v86);  
  v15 = v14;  
  sectiondata_1 = v14;  
  if ( v14 )  
  {  
    if ( ReadImageData(v71, pebegin, v70, 0x30ui64, __PAIR__(HIDWORD(v64), v14), v47, v54) )  
    {  
      ImageHelpPointerWrapper<_IMAGE_SECTION_HEADER>::Assign((v3 + 3352), v15, lpMem, v16);  
      *(v3 + 3344) = v15;  
      *(v3 + 3384) = lpMem;  
      if ( v88 )  
      {  
        v67 = v88 / 0x1C;  
        v17 = 40 * v86 + 48;  
        v18 = v17 + v87;  
        v65 = v17 + v87;  
        v19 = 1;  
        goto LABEL_27;  
      }  
    }  
  }  
  v19 = 1;  
LABEL_26:  
  v18 = v65;  
LABEL_27:  
  v20 = v67;  
LABEL_28:  
  if ( v2 == 2 )  
  {  
    *(v3 + 3364) = v18;  
    *(v3 + 3368) = v20;  
  }  
  while ( v20 )  
  {  
    HIDWORD(v21) = 28;  
    LODWORD(v21) = v73;  
    if ( !ReadImageData(v71, pebegin, v70, v18, v21, v47, v54) )// 逐个读取调试信息  
      goto LABEL_11;  
    if ( *&v73[16] )                            // IMAGE_DEBUG_DIRECTORY.SizeOfData  
    {  
      imgset(*&v73[12], *(v3 + 4036), v2, 0, v47, v54);  
      if ( *&v73[12] == 1 )  
      {  
        if ( *&v73[24] < filesize2_4 )  
        {  
          ImageHelpPointerWrapper<_OMAP>::Assign((v3 + 3264), (pebegin + *&v73[24]), *&v73[16], v19);  
          *(v3 + 3948) = v2;  
          goto LABEL_95;  
        }  
        *(v3 + 4024) = 1;  
      }  
      else 
      {  
        if ( *&v73[12] == 2 )  
        {  
          if ( v71 )  
          {  
            if ( !*&v73[20] )  
              goto LABEL_11;  
            v42 = pMemAlloc(*&v73[16]);  
            lpMem = v42;  
            if ( !v42 )  
              goto LABEL_108;  
            if ( !ReadImageData(v71, pebegin, v70, *&v73[20], __PAIR__(*&v73[16], v42), v47, v54) )  
            {  
              pMemFree(lpMem);  
              goto LABEL_11;  
            }  
            ImageHelpPointerWrapper<_OMAP>::Assign((v3 + 0xCB4), lpMem, *&v73[16], 0);  
          }  
          else 
          {  
            if ( *&v73[24] >= filesize2_4 )     // IMAGE_OPTIONAL_HEADER.PointerToRawData 此处就是我们要的数据了  
              goto LABEL_108;  
            ImageHelpPointerWrapper<_OMAP>::Assign((v3 + 0xCB4), (pebegin + *&v73[24]), *&v73[16], 1);  
          }  
          *(v3 + 3952) = v2;  
          RetrievePdbInfo(v3, v3);  
          goto LABEL_95;  
        }  
        if ( *&v73[12] != 4 )  
          goto LABEL_108;  
        if ( *&v73[24] < filesize2_4 )  
        {  
          v22 = pebegin;  
          if ( *(pebegin + *&v73[24]) != 1 )  
            goto LABEL_109;  
          if ( v2 == 3 )  
          {  
            v22 = pebegin;  
            if ( !*(v3 + 572) )  
            {  
              ansi2wcs(0x105, v47, v54);  
              goto LABEL_108;  
            }  
LABEL_109:  
            if ( *&v73[24] >= filesize2_4 )  
              goto LABEL_120;  
            if ( *&v73[12] != 3 )  
            {  
              if ( *&v73[12] == 5 )  
              {  
                *(v3 + 3972) = v2;  
                v44 = v2;  
                v43 = 5;  
              }  
              else 
              {  
                if ( *&v73[12] == 7 )  
                {  
                  ImageHelpPointerWrapper<_OMAP>::Assign((v3 + 3316), (v22 + *&v73[24]), *&v73[16] >> 3, v19);  
                  *(v3 + 3964) = v2;  
                }  
                else 
                {  
                  if ( *&v73[12] != 8 )  
                    goto LABEL_120;  
                  ImageHelpPointerWrapper<_OMAP>::Assign((v3 + 3328), (v22 + *&v73[24]), *&v73[16] >> 3, v19);  
                  *(v3 + 3968) = v2;  
                }  
LABEL_118:  
                v44 = v2;  
                v43 = *&v73[12];  
              }  
              imgset(v43, *(v3 + 4036), 0, v44, v47, v54);  
              goto LABEL_120;  
            }  
            ImageHelpPointerWrapper<_OMAP>::Assign((v3 + 3280), (v22 + *&v73[24]), *&v73[16] >> 4, v19);  
            *(v3 + 3960) = v2;  
            goto LABEL_118;  
          }  
          if ( *(v66 + 9) & 0x200 )  
          {  
            ansi2wcs(0x105, v47, v54);  
            *(v3 + 2164) = *&v73[4];  
          }  
          else 
          {  
            ansi2wcs(0x105, v47, v54);  
          }  
        }  
LABEL_95:  
        imgset(*&v73[12], *(v3 + 4036), 0, v2, v47, v54);  
      }  
LABEL_108:  
      v22 = pebegin;  
      goto LABEL_109;  
    }  
LABEL_120:  
    v18 = v65 + 28;  
    v65 += 28;  
    v20 = v67-- - 1;  
  }  
  return 1;  
} 
```

&emsp;&emsp;可以看到dbghelp!ReadHeader读取文件，解析PE格式，然后提取其中的debug区信息，标志RSDS后存放的GUID即为我们要找的数据
```Txt
0293cbd4  52 53 44 53 83 65 a7 47-c5 34 bc 47 a0 3e 62 93  RSDS.e.G.4.G.>b.
0293cbe4  a7 8c 67 5f 02 00 00 00-77 6e 74 64 6c 6c 2e 70  ..g_....wntdll.p
0293cbf4  64 62 00 00 00 00 00 00-00 00 00 00 f6 c1 03 75  db.............u
0293cc04  0d 39 71 0c 75 08 8b 41-04 3b 46 08 74 4d 83 26  .9q.u..A.;F.tM.&
0293cc14  00 83 66 04 00 eb 42 90-90 90 90 90 8b ff 55 8b  ..f...B.......U.
0293cc24  ec 51 53 8b 5d 08 57 8b-7d 0c 3b 7b 10 0f 87 02  .QS.].W.}.;{....
0293cc34  01 00 00 8b 43 0c 56 8d-73 20 3b c7 73 0e 83 c6  ....C.V.s ;.s...
0293cc44  10 03 c0 eb f5 90 90 90-90 90 90 90 8b ce e8 99  ................
```

&emsp;&emsp;解决任务2的重点在于SymbolServerGetIndexStringW，来看他的内部调用:
* CatStrID 将前面BYTE* index经过一定变换转换成字符串
* CatStrDWORD 将前面int dword1添加到字符串结尾
* CatStrDWORD 将前面int dword2添加到字符串结尾

CatStrID中实现大概是这样：

```C++
switch(gptype)
{
        case 2:
                CatStrDWORD(buf,id);
                break;
        case 4:
                CatStrDWORD(buf,*(DWORD*)id);
                break;
        case 8:
                CatStrGUID(buf,id);//我们用到的case
                break;
        case 0x10:
                CatStrOldGUID(buf,id);
                break;
        case 0x400000:
                wcscpy_s(buf,id);
}

CatStrDWORD中实现大概是这样：
if(dwordn)
{
        sprintf(buf,"%s%d",index,dwordn);
}
```

&emsp;&emsp;剩下的懒得写了，现在大家知道windbg怎样查符号了吧！

