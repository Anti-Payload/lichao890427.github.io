---
layout: post
title: 可执行文件另类加密
categories: Windows
description: 可执行文件另类加密
keywords: PE 加壳
---

操作方法：给定A.exe进行保护
* 1.使用生成工具去除A.exe所有段内容，例如  

|fileoff |memoff |segname|
|--------|-------|-------|
|00000400| 123C00| text  |
|00124000| 042E00| rdata |
|00166E00| 009000| data  |
|0016FE00| 00F200| rsrc  |
|0017F000| 01E200| reloc |

&emsp;&emsp;工具会把这些段指向的文件内容置0，生成Afix.exe，这个文件无法运行，因为已经挖空了
* 2.把A.exe用可逆算法加密成其他文件
* 3.用启动工具以挂起模式启动Afix.exe
* 4.将A.exe各个段加载到缓冲区中，对各个段进行重定位
* 5.将缓冲区中各个段内容动态填充到挂起的Afix.exe的各个段中
* 6.恢复Afix.exe此时可以运行

```C++
#include <stdio.h>
#include <Windows.h>
#include <tchar.h>
#include <shlwapi.h>
#include <winternl.h>
#include <zlib.h>
#include <zconf.h>
#include <list>
using namespace std;
#pragma comment(lib,"shlwapi.lib")
class MySection
{                                       /* 记录PE各个段数据 */
public:
	IMAGE_SECTION_HEADER header;    /* 段头 */
	BYTE* data;                     /* 段数据 */

public:
	MySection()
	{
		data = NULL;
		memset( &header, 0, sizeof(header) );
	}

	void SetHeader( IMAGE_SECTION_HEADER & header )
	{
		memcpy( &this->header, &header, sizeof(IMAGE_SECTION_HEADER) );
		data = new BYTE[header.SizeOfRawData];
		memset( data, 0, header.SizeOfRawData );
	}

	~MySection()
	{
		if ( data )
		{
			delete[]data;
			data = NULL;
		}
	}
};

list<MySection> Sections;
BOOL InjectProcess( LPTSTR VictimFile, LPBYTE pInjectFileBuf )
{       /* VictimFile宿主文件   IndectFile注入文件(原始文件) */
	IMAGE_DOS_HEADER DosHeaders, DosHeadero;
	IMAGE_NT_HEADERS NtHeaders, NtHeadero;
	PROCESS_INFORMATION pi;
	STARTUPINFO si;
	CONTEXT context;
	PVOID ImageBase;
	SIZE_T BaseAddr;
	LONG offset;
	DWORD dwBytes;
	HMODULE hNtDll = GetModuleHandle( "ntdll.dll" );
	if ( !hNtDll )
		return(FALSE);

	memset( &si, 0, sizeof(si) );
	memset( &pi, 0, sizeof(pi) );
	si.cb = sizeof(si);

	CopyMemory( (void *) &DosHeadero, pInjectFileBuf, sizeof(IMAGE_DOS_HEADER) );
	CopyMemory( (void *) &NtHeadero, pInjectFileBuf + DosHeadero.e_lfanew, sizeof(IMAGE_NT_HEADERS) );

	/* 检查PE结构 */
	/* 以挂起方式进程 */
	BOOL res = CreateProcess( NULL, VictimFile, NULL, NULL, FALSE, CREATE_SUSPENDED, NULL, NULL, &si, &pi );
	if ( res )
	{
		context.ContextFlags = CONTEXT_FULL;
		if ( !GetThreadContext( pi.hThread, &context ) ) /* 如果调用失败 */
		{
			CloseHandle( pi.hThread );
			CloseHandle( pi.hProcess );
			return(FALSE);
		}
		ReadProcessMemory( pi.hProcess, (void *) (context.Ebx + 8), &BaseAddr, sizeof(unsigned long), NULL );

		if ( !BaseAddr )
		{
			CloseHandle( pi.hThread );
			CloseHandle( pi.hProcess );
			return(FALSE);
		}

		/* 计算FS的基址 */
		LDT_ENTRY SelEntry;
		PTEB pteb = new TEB;
		PPEB ppeb = new PEB;

		GetThreadSelectorEntry( pi.hThread, context.SegFs, &SelEntry );
		DWORD dwFSBase = (SelEntry.HighWord.Bits.BaseHi << 24) | (SelEntry.HighWord.Bits.BaseMid << 16) | SelEntry.BaseLow;

		ReadProcessMemory( pi.hProcess, (LPCVOID) dwFSBase, pteb, sizeof(TEB), &dwBytes );
		ReadProcessMemory( pi.hProcess, (LPCVOID) *(DWORD *) ( (BYTE *) pteb + 0x30), ppeb, sizeof(PEB), &dwBytes );    /* pteb->Peb */
		ImageBase = *(PVOID *) ( (BYTE *) ppeb + 0x08);                                                                 /* ppeb->ImageBaseAddress; */
		ReadProcessMemory( pi.hProcess, ImageBase, &DosHeaders, sizeof(IMAGE_DOS_HEADER), &dwBytes );
		ReadProcessMemory( pi.hProcess, (LPVOID) ( (BYTE *) ImageBase + DosHeaders.e_lfanew), &NtHeaders, sizeof(IMAGE_NT_HEADERS), &dwBytes );

		delete pteb;
		delete ppeb;

		offset = DosHeaders.e_lfanew + sizeof(IMAGE_NT_HEADERS);
		IMAGE_SECTION_HEADER curHeader; /* .text和.reloc的信息 */
		WORD i = 0;
		for ( i = 0; i < NtHeaders.FileHeader.NumberOfSections; i++ )
		{
			MySection section;
			ReadProcessMemory( pi.hProcess, (LPVOID) ( (BYTE *) ImageBase + offset + i * sizeof(IMAGE_SECTION_HEADER) ), (void *) &curHeader, sizeof(IMAGE_SECTION_HEADER), &dwBytes );

			/*下面四行代码获取各个段信息 */
			/* .reloc段在CreateProcess后就已经被系统重定向应用到各个段里去了，如果要自己处理，就要取出这个段，并自己应用reloc */
			curHeader.SizeOfRawData = curHeader.Misc.VirtualSize; /* 用SizeOfRawData在写入的时候会发生堆溢出，目前不知道原因 */
			section.SetHeader( curHeader );
			CopyMemory( section.data, pInjectFileBuf + curHeader.PointerToRawData, curHeader.SizeOfRawData );
			Sections.push_back( section );

			if ( !StrCmp( (char *) curHeader.Name, ".reloc" ) )     /* 确保reloc是最后一个段 */
			{                                                       /* 实现重定位 */
				BYTE* relocdata = section.data;
				IMAGE_BASE_RELOCATION CurReloc;
				CopyMemory( (void *) &CurReloc, relocdata, sizeof(CurReloc) );
				DWORD offset = 0;
				while ( CurReloc.VirtualAddress )
				{
					int num = (CurReloc.SizeOfBlock - sizeof(IMAGE_BASE_RELOCATION) ) / 2;
					WORD* pData = (WORD *) (relocdata + sizeof(IMAGE_BASE_RELOCATION) + offset);
					for ( int j = 0; j < num; j++ )
					{
						switch ( pData[j] >> 12 )
						{
						case IMAGE_REL_BASED_ABSOLUTE: /* 什么都不做 */
							break;

						case IMAGE_REL_BASED_HIGHLOW:
						{
							list<MySection>::iterator itor = Sections.begin();
							while ( itor != Sections.end() )
							{
								int inneroffset = CurReloc.VirtualAddress + (pData[j] & 0xFFF) - (*itor).header.VirtualAddress;
								if ( inneroffset >= 0 && inneroffset <= (*itor).header.SizeOfRawData )
								{
									*(DWORD *) ( (*itor).data + inneroffset) += NtHeaders.OptionalHeader.ImageBase - NtHeadero.OptionalHeader.ImageBase;
									break;
								}
								++itor;
							}
						}
						break;

						default:
							printf( "unknown!!!!!!!!!!!!!!" );
							break;
						}
					}
					offset += CurReloc.SizeOfBlock;
					CopyMemory( (void *) &CurReloc, relocdata + offset, sizeof(CurReloc) );
				}
			}

			section.data = NULL; /* 浅拷贝防止析构 */
		}

		list<MySection>::iterator itor = Sections.begin();
		while ( itor != Sections.end() )
		{
			/*有些段会因为没有写入权限失败 */
			WriteProcessMemory( pi.hProcess, (LPVOID) ( (DWORD) ImageBase + (*itor).header.VirtualAddress), (*itor).data, (*itor).header.SizeOfRawData, &dwBytes );
			if ( !StrCmp( (char *) (*itor).header.Name, ".text" ) )
			{
				break;
			}
			++itor;
		}
		ResumeThread( pi.hThread );
	}

	return(TRUE);
}

void main()
{
	if ( __argc < 3 ) /* 文件名+MAKE     文件名+RUN */
	{
		return;
	}

	if ( !StrCmp( __argv[2], "MAKE" ) )
	{       /* 生成注入文件 */
		char dir[MAX_PATH], newfile[MAX_PATH];
		strcpy( dir, __argv[1] );
		PathRemoveFileSpec( dir );
		strcpy( newfile, dir );
		strcat( newfile, "\\data" );
		CopyFile( __argv[1], newfile, FALSE );

		IMAGE_DOS_HEADER DosHeader;
		IMAGE_NT_HEADERS NtHeader;
		DWORD dwNumberOfBytesRead = 0;
		HANDLE hOldFile = CreateFile( __argv[1], GENERIC_READ | GENERIC_WRITE, FILE_SHARE_READ | FILE_SHARE_WRITE, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL );
		if ( hOldFile == INVALID_HANDLE_VALUE )
		{
			return;
		}
		SetFilePointer( hOldFile, 0, NULL, FILE_BEGIN );
		ReadFile( hOldFile, &DosHeader, sizeof(DosHeader), &dwNumberOfBytesRead, NULL );
		SetFilePointer( hOldFile, DosHeader.e_lfanew, NULL, FILE_BEGIN );
		ReadFile( hOldFile, &NtHeader, sizeof(NtHeader), &dwNumberOfBytesRead, NULL );
		IMAGE_SECTION_HEADER curHeader; /* .text和.reloc的信息 */
		WORD i = 0;

		for ( i = 0; i < NtHeader.FileHeader.NumberOfSections; i++ )
		{
			ReadFile( hOldFile, &curHeader, sizeof(curHeader), &dwNumberOfBytesRead, NULL );
			if ( !StrCmp( (char *) curHeader.Name, ".text" ) )
			{
				break;
			}
		}

		SetFilePointer( hOldFile, curHeader.PointerToRawData, NULL, FILE_BEGIN );
		BYTE* newbuf = new BYTE[curHeader.SizeOfRawData];
		memset( newbuf, 0, curHeader.SizeOfRawData );
		WriteFile( hOldFile, newbuf, curHeader.SizeOfRawData, &dwNumberOfBytesRead, NULL );
		CloseHandle( hOldFile );
		delete[]newbuf;
	}else if ( !StrCmp( __argv[2], "RUN" ) )
	{       /* 运行文件 */
		char dir[MAX_PATH];
		strcpy( dir, __argv[1] );
		PathRemoveFileSpec( dir );
		strcat( dir, "\\data" );
		HANDLE hNewFile = CreateFile( dir, GENERIC_READ, FILE_SHARE_READ | FILE_SHARE_WRITE, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL );
		if ( hNewFile == INVALID_HANDLE_VALUE )
		{
			return;
		}
		DWORD dwFileSize = GetFileSize( hNewFile, NULL );
		LPBYTE pInjectFileBuf = new BYTE[dwFileSize];
		memset( pInjectFileBuf, 0, dwFileSize );
		DWORD dwNumberOfBytesRead = 0;
		ReadFile( hNewFile, pInjectFileBuf, dwFileSize, &dwNumberOfBytesRead, NULL );
		CloseHandle( hNewFile );
		InjectProcess( __argv[1], pInjectFileBuf );
		delete[]pInjectFileBuf;
	}
```

用法：
* Encrypt "D:\Program Files\Thunder Network\Thunder\Program\Thunder.exe" MAKE
执行后会将Thunder.exe的.text掏空，使原始文件无法运行，另外将Thunder.exe拷贝一份到同目录下的data文件，这个我没做加密处理，不过相加还是能加的
* Encrypt "D:\Program Files\Thunder Network\Thunder\Program\Thunder.exe" RUN
将被掏空代码的Thunder.exe以挂起模式运行，之后将data文件中正常数据，经过reloc手工修正各个段后填回Thunder.exe的内存映射中，这样就可以正常运行了
