---
layout: post
title: 实现Windbg x64汇编功能
categories: Windows
description: 实现Windbg x64汇编功能
keywords: 
---

# 实现windbg x64汇编功能

## 背景

&emsp;&emsp;熟悉windbg的都知道，a指令，只支持x86，另外经我研究，虽然引擎中扩展了I386|ARM|IA64|AMD64|EBC的指令集，但是后几种只有反汇编能力，而并没有汇编能力。因此这是个突破点，然而汇编器源码这种东西是比较稀缺的，无奈之下为了实现amd64汇编我选择了ml64.exe工具，利用该用具生成机器码

```Txt
0:000> !a -s AMD64 
usage: !a [-s ProcessorType] [-a Address]
        Optional ProcessorType:I386|ARM|IA64|AMD64|EBC
        Default ProcessorType is I386;Default Address is current $ip
example:!a -s AMD64 -a .
Assemble on AMD64 at 00007FFD75B81970
please input asm code, [enter] to leave
mov r8,0
mov r8,0
	00007ffd`75b81970 49c7c000000000  mov     r8,0
mov r8,gs:[0]
mov r8,gs:[0]
	00007ffd`75b81977 654c8b042500000000 mov   r8,qword ptr gs:[0] gs:00000000`00000000=????????????????
asm edit leave
```

## 开发Windbg插件

```C++
// WDbgLiExts.cpp
#include "DbgEng.h"
#include <windows.h>
#include <fstream>
#include <shlwapi.h>
#pragma comment(lib,"Shlwapi.lib")

#define EXT_MAJOR_VER  1
#define EXT_MINOR_VER  0

extern "C" HRESULT CALLBACK
DebugExtensionInitialize(PULONG Version, PULONG Flags) 
{
	*Version = DEBUG_EXTENSION_VERSION(EXT_MAJOR_VER, EXT_MINOR_VER);
	*Flags = 0;  // Reserved for future use.
	return S_OK;
}

extern "C" void CALLBACK
DebugExtensionNotify(ULONG Notify, ULONG64 Argument) 
{
	UNREFERENCED_PARAMETER(Argument);
	switch (Notify) {
		// A debugging session is active. The session may not necessarily be suspended.
	case DEBUG_NOTIFY_SESSION_ACTIVE:
		break;
		// No debugging session is active.
	case DEBUG_NOTIFY_SESSION_INACTIVE:
		break;
		// The debugging session has suspended and is now accessible.
	case DEBUG_NOTIFY_SESSION_ACCESSIBLE:
		break;
		// The debugging session has started running and is now inaccessible.
	case DEBUG_NOTIFY_SESSION_INACCESSIBLE:
		break;
	}
	return;
}

extern "C" void CALLBACK
DebugExtensionUninitialize(void) 
{
	return;
}

HRESULT CALLBACK
helloworld(PDEBUG_CLIENT pDebugClient, PCSTR args) 
{
	UNREFERENCED_PARAMETER(args);
	IDebugControl* pDebugControl;
	if (SUCCEEDED(pDebugClient->QueryInterface(__uuidof(IDebugControl),
		(void **)&pDebugControl))) {
		pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "Hello World!\n");
		pDebugControl->Release();
	}
	return S_OK;
}

//返回参数长度，和起始位置
int GetParamVal(PCSTR& begin, PCSTR& end)
{
	PCSTR truebegin = 0, trueend = 0;
	while (*begin)
	{
		if (*begin != ' ' && *begin != '\t')
		{
			truebegin = begin;
			break;
		}
		begin++;
	}

	trueend = truebegin;
	do
	{
		if (*trueend == '-' || *trueend == '\0')
		{
			break;
		}
		trueend++;
	} while (true);

	return trueend - truebegin;
}

PSTR getnextnonblank(PSTR begin)
{
	while (*begin)
	{
		if (*begin != ' ' && *begin != '\t')
			break;
		begin++;
	}
	return begin;
}

PSTR getnextchar(PSTR begin, char ch)
{
	while (*begin)
	{
		if (*begin == ch)
			break;
		begin++;
	}
	return begin;
}

bool ResolveSymbolInExpression(IDebugControl* pDebugControl, PSTR asmcode, PSTR outcode, ULONG64 Xip)
{	
	/*
	//找到操作码起始位置
	PSTR opcode, opdata;
	int opcodelen, opdatalen;
	asmcode = getnextnonblank(asmcode);
	if (!*asmcode)//找不到操作码
		return false;
	char si[] = " ,,,,,";
	int index = 0;
	char exp[256];
	__debugbreak();
	DEBUG_VALUE value;
	do
	{
		opcode = asmcode;
		asmcode = getnextchar(asmcode, si[index]);
		opcodelen = asmcode - opcode;
		strncpy(exp, opcode, opcodelen);
		exp[opcodelen] = '\0';
		memset(&value, 0, sizeof(value));
		if (SUCCEEDED(pDebugControl->Evaluate(exp, DEBUG_VALUE_INT64, &value, NULL)))
		{
			//若能解析
			sprintf(exp, "%I64d", (LONG64)value.I64);
			opcodelen = strlen(exp);
			strncpy(outcode, exp, opcodelen);
			outcode[opcodelen] = '\0';
			pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "解析exp=%s\n", outcode);
		}
		else
		{
			//不能解析
			strncpy(outcode, opcode, opcodelen);
			outcode[opcodelen] = '\0';
			pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "无法解析exp=%s\n", exp);
		}
		
		outcode += opcodelen;
		asmcode = getnextnonblank(asmcode + 1);
		*outcode = si[index];
		outcode++;
		index++;
	} while (*asmcode);
	outcode[-1] = '\0';
	*/
	strcpy(outcode, asmcode);
	return true;
}

bool GetByteCode(IDebugControl* pDebugControl, PSTR asmcode, PSTR outbyte, PULONG byteswrite)
{
	__debugbreak();
	char buf[256], asmpath[256], objpath[256], ml64path[256], msvcr100[256];
	bool ret = false;
	GetCurrentDirectoryA(256, buf);
	sprintf(asmpath, "%s\\test.asm", buf);
	sprintf(objpath, "%s\\test.obj", buf);
	sprintf(ml64path, "%s\\ml64.exe", buf);
	sprintf(msvcr100, "%s\\msvcr100.dll", buf);
	FILE* fpasm = NULL,*fpobj = NULL;
	fpasm = fopen(asmpath, "w");
	if (!fpasm)
	{
		pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "无法创建asm\n");
	}
	else if (!PathFileExistsA(ml64path))
	{
		pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "ml64.exe不存在\n");
	}
	else if (!PathFileExistsA(msvcr100))
	{
		pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "msvcr100.dll不存在\n");
	}
	else
	{
		fputs(".CODE\n", fpasm);
		fputs("Entry PROC\n", fpasm);
		fputs(asmcode, fpasm);
		fputs("\nEntry ENDP\n", fpasm);
		fputs("END\n", fpasm);
		fclose(fpasm);
		fpasm = NULL;
		if (!PathFileExistsA(objpath) || DeleteFileA(objpath))
		{
			WinExec("ml64 test.asm", SW_HIDE);
			Sleep(200);
			fpobj = fopen(objpath, "rb");
			if (!fpobj)
			{
				pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "语法错误\n");
			}
			else
			{
				char* data = new char[256];
				fread(data, 256, 1, fpobj);
				int offset = 0x18;
				unsigned short datasize;
				offset += *(unsigned short*)(data + offset);
				datasize = *(unsigned short*)(data + 0x24);
				//此时data+offset处的datasize个字节即为汇编生成的机器码
				//写入内存
				memcpy(outbyte, data + offset, datasize);
				*byteswrite = datasize;
				delete[]data;
				ret = true;
			}
		}
		else
		{
			pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "obj文件无法删除\n");	
		}
	}
	if (fpasm)
		fclose(fpasm);
	if (fpobj)
		fclose(fpobj);
	//DeleteFileA(asmpath);
	DeleteFileA(objpath);
	return ret;
}

HRESULT CALLBACK
a(PDEBUG_CLIENT pDebugClient, PCSTR args) 
{
	UNREFERENCED_PARAMETER(args);
	IDebugControl* pDebugControl;

	if (SUCCEEDED(pDebugClient->QueryInterface(__uuidof(IDebugControl),(void **)&pDebugControl))) 
	{
		HRESULT result = 0;
		ULONG OriProcessorType = 0, CurProcessorType = 0;
		if (!SUCCEEDED(pDebugControl->GetEffectiveProcessorType(&OriProcessorType)))
			OriProcessorType = IMAGE_FILE_MACHINE_I386;

		pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "usage: !a [-s ProcessorType] [-a Address]\n");
		pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "\tOptional ProcessorType:I386|ARM|IA64|AMD64|EBC\n");
		pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "\tDefault ProcessorType is I386;Default Address is current $ip\n");
		pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "example:!a -s AMD64 -a .\n");

		PSTR pt = (PSTR)args;
		DEBUG_VALUE value;
		ULONG64 Address;

		char exp[256] = "$ip",ProcessorName[256];
		PCSTR b = pt, e = pt;
		CurProcessorType = IMAGE_FILE_MACHINE_I386;
		strcpy(ProcessorName, "I386");
		if (b = strstr(pt, "-s"))
		{
			if (strstr(pt, "I386"))
			{
				CurProcessorType = IMAGE_FILE_MACHINE_I386;
				strcpy(ProcessorName, "I386");
			}
			else if (strstr(pt, "ARM"))
			{
				CurProcessorType = IMAGE_FILE_MACHINE_ARM;
				strcpy(ProcessorName, "ARM");
			}
			else if (strstr(pt, "IA64"))
			{
				CurProcessorType = IMAGE_FILE_MACHINE_IA64;
				strcpy(ProcessorName, "IA64");
			}
			else if (strstr(pt, "AMD64"))
			{
				CurProcessorType = IMAGE_FILE_MACHINE_AMD64;
				strcpy(ProcessorName, "AMD64");
			}
			else if (strstr(pt, "EBC"))
			{
				CurProcessorType = IMAGE_FILE_MACHINE_EBC;
				strcpy(ProcessorName, "EBC");
			}
		}
		else if (b = strstr(pt, "-a"))
		{
			e = b;
			int len = GetParamVal(b, e);
			if (len)
			{
				strncpy(exp, b, len);
				exp[len] = '\0';
			}
		}
		pDebugControl->Evaluate(exp, (OriProcessorType == IMAGE_FILE_MACHINE_I386) ? DEBUG_VALUE_INT32 : DEBUG_VALUE_INT64, &value, NULL);
		Address = (OriProcessorType == IMAGE_FILE_MACHINE_I386) ? value.I32 : value.I64;	
		pDebugControl->SetEffectiveProcessorType(CurProcessorType);
		pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "Assemble on %s at %N\n please input asm code, [enter] to leave\n", ProcessorName, Address);

		char Inputbuf[256];
		while (true)
		{
			ULONG64 NextAddr = Address,NextAddr2;
			memset(Inputbuf, 0, 256);
			pDebugControl->Input(Inputbuf, 256, NULL);
			if (strlen(Inputbuf) == 0)
				break;
			switch (CurProcessorType)
			{
			case IMAGE_FILE_MACHINE_I386:
				strcpy(pt, Inputbuf);
				//逐行反汇编
				result = pDebugControl->Assemble(Address, Inputbuf, &NextAddr);
				//打印结果
				if (SUCCEEDED(result))
				{
					pDebugControl->OutputDisassembly(DEBUG_OUTCTL_ALL_CLIENTS, Address, DEBUG_DISASM_EFFECTIVE_ADDRESS | DEBUG_DISASM_MATCHING_SYMBOLS |
						DEBUG_DISASM_SOURCE_LINE_NUMBER | DEBUG_DISASM_SOURCE_FILE_NAME, &NextAddr2);
					Address = NextAddr;
				}
				break;
			case IMAGE_FILE_MACHINE_AMD64:
				{
					char bytecode[256],fixasmcode[256];				
					bool suc = true;
					
					//使用ml64进行解析：
					//1.先将Inputbuf中的符号解析为数据
					suc = ResolveSymbolInExpression(pDebugControl, Inputbuf, fixasmcode, Address);
					if(!suc)
						pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "unresolve symbol\n");
					//2.使用ml64解析并取得obj机器码
					else
					{
						ULONG bytewrite = 0;
						suc = GetByteCode(pDebugControl, fixasmcode, bytecode, &bytewrite);
						if(!suc || !bytewrite)
							pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "file op or disasm fail\n");
						else
						{
							//3.写入虚拟内存
							IDebugDataSpaces* dataspace;
							if (SUCCEEDED(pDebugClient->QueryInterface(__uuidof(IDebugDataSpaces), (void **)&dataspace)))
							{
								if (!SUCCEEDED(dataspace->WriteVirtual(Address, (PVOID)bytecode, bytewrite, NULL)))
								{
									suc = false;
									pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "can't write to memory\n");
								}
							}
							else
							{
								suc = false;
								pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "can't obtain IDebugDataSpaces\n");
							}
							if (suc)
							{
								pDebugControl->OutputDisassembly(DEBUG_OUTCTL_ALL_CLIENTS, Address, DEBUG_DISASM_EFFECTIVE_ADDRESS | DEBUG_DISASM_MATCHING_SYMBOLS |
									DEBUG_DISASM_SOURCE_LINE_NUMBER | DEBUG_DISASM_SOURCE_FILE_NAME, &NextAddr2);
								Address = NextAddr2;
							}
						}
					}
					
				}
				break;
			default:
				break;
			}
		}
		pDebugControl->Output(DEBUG_OUTPUT_NORMAL, "asm edit leave\n", ProcessorName, Address);
		pDebugControl->SetEffectiveProcessorType(OriProcessorType);
		pDebugControl->Release();
	}
	return S_OK;
}
```

```Txt
 export.def

 EXPORTS
   DebugExtensionNotify
   DebugExtensionInitialize
   DebugExtensionUninitialize
   a
```