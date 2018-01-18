---
layout: post
title: 论磁盘内存映射
categories: WindowsDriver
description: 论磁盘内存映射
keywords: 
---

# 论磁盘内存映射

&emsp;&emsp;这个题目有点眼熟吧，文件内存映射用的比较多，用于大文件读写效率较高。其原理是将映射的内存页的来源标记为磁盘驱动器，这样对地址的读写产生中断时，会直接操作磁盘，这样对打文件的读写速度提高很多。然而这种技术微软只用在文件上，对于目录、磁盘和设备均不能使用。而普通缓冲区buffer方式读写，总是要有内存拷贝，所以相对慢一些，在我3年前做数据恢复的时候，为了研究大文件恢复接触到文件内存映射，第一次就很纳闷到底有没有磁盘映射，这样我就不需要总是读来读去。下面给出这2种方式的区别：

```Txt
Buffered方式：
    CreateFile
    ReadFile/WriteFile
    CloseHandle

mapping方式：
    CreateFile
    CreateFileMapping
    MapViewOfFile
    指针操作
    UnmapViewOfFile
    CloseHandle
    CloseHandle
```

&emsp;&emsp;在这里我对相关内核API做了逆向和破解，写了一个驱动支持对于磁盘实现内存映射。主要是对IRP_MJ_QUERY_INFORMATION和FastIoQueryStandardInfo的hook，代码如下：

```C++
#include <ntddk.h>
#include "DeviceMappingCommon.h"

typedef struct _DEVICE_MAPPING
{
	HANDLE hDevice;
	PFILE_OBJECT pFileObject;
	LARGE_INTEGER dwMaximumSize;
}DEVICE_MAPPING, *PDEVICE_MAPPING;

PDEVICE_MAPPING gMappingData = NULL;
PDRIVER_DISPATCH OriginQuery = NULL;
PFAST_IO_QUERY_STANDARD_INFO OriginFastDispatch = NULL;
FAST_MUTEX gLock;

extern "C" 
{
	BOOLEAN NTAPI ObFindHandleForObject(PEPROCESS, PVOID, PVOID, PVOID, PHANDLE);
	NTSTATUS NTAPI NtQueryVolumeInformationFile (HANDLE, PIO_STATUS_BLOCK, PVOID, ULONG, FS_INFORMATION_CLASS);
}

VOID DeviceMappingDriverUnload(PDRIVER_OBJECT DriverObject)
{
	ExAcquireFastMutex(&gLock);
	if(gMappingData)
	{
		ExFreePool((PVOID)gMappingData);
		gMappingData = NULL;
	}
	ExReleaseFastMutex(&gLock);
	if(DriverObject->DeviceObject)
	{
		UNICODE_STRING DriverName;
		RtlInitUnicodeString(&DriverName, L"\\Device\\DeviceMapping");
		IoDeleteSymbolicLink(&DriverName);
		IoDeleteDevice(DriverObject->DeviceObject);
	}
	return;
}

NTSTATUS DeviceMappingQueryInformation(PDEVICE_OBJECT pDevObj, PIRP pIrp)
{
	PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(pIrp);
	if(stack->Parameters.QueryFile.FileInformationClass == FileStandardInformation && stack->FileObject)
	{
		HANDLE hDevice = NULL;
		BOOLEAN IsRef = FALSE;
		ExAcquireFastMutex(&gLock);
		if(gMappingData && stack->FileObject == gMappingData->pFileObject)
		{
			PFILE_STANDARD_INFORMATION FileInformation = (PFILE_STANDARD_INFORMATION)pIrp->AssociatedIrp.SystemBuffer;
			FileInformation->EndOfFile.QuadPart = gMappingData->dwMaximumSize.QuadPart;
			pIrp->IoStatus.Status = STATUS_SUCCESS;
			pIrp->IoStatus.Information = sizeof(FILE_STANDARD_INFORMATION);
			IoCompleteRequest(pIrp, IO_NO_INCREMENT);
			IsRef = TRUE;
		}
		ExReleaseFastMutex(&gLock);
		if(IsRef)
			return STATUS_SUCCESS;
	}
	OriginQuery(pDevObj, pIrp);
}

BOOLEAN DeviceMappingFastIo(PFILE_OBJECT FileObject, BOOLEAN Wait, PFILE_STANDARD_INFORMATION Buffer,PIO_STATUS_BLOCK IoStatus,struct _DEVICE_OBJECT *DeviceObject)
{
	BOOLEAN ret = OriginFastDispatch(FileObject, Wait, Buffer, IoStatus, DeviceObject);
	ExAcquireFastMutex(&gLock);
	if(FileObject == gMappingData->pFileObject)
	{
		Buffer->Directory = FALSE;
	}
	ExReleaseFastMutex(&gLock);
	return ret;
}

NTSTATUS DeviceMappingIOControl(PDEVICE_OBJECT pDevObj, PIRP pIrp)
{
	NTSTATUS status = STATUS_SUCCESS;
	PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(pIrp);
	ULONG cbin = stack->Parameters.DeviceIoControl.InputBufferLength;
	ULONG cbout = stack->Parameters.DeviceIoControl.OutputBufferLength;
	ULONG code = stack->Parameters.DeviceIoControl.IoControlCode;
	ULONG info = 0;
	switch(code)
	{
	case IOCTL_ENABLE_DEVICEMAPPING:
		{
			DEVICE_MAPPING tmp;
			PFILE_OBJECT fo = NULL;
			PDRIVER_DISPATCH* ppdd = NULL;
			PFAST_IO_QUERY_STANDARD_INFO* ppfiqsi = NULL;
			DEVICEMAPPING_IOCTL_DATA* InputBuffer = (DEVICEMAPPING_IOCTL_DATA*)pIrp->AssociatedIrp.SystemBuffer;
			RtlZeroMemory(&tmp, sizeof(tmp));
			tmp.hDevice = InputBuffer->hDevice;
			status = ObReferenceObjectByHandle(InputBuffer->hDevice, NULL, *IoFileObjectType, KernelMode, (PVOID*)&fo, NULL);
			if(NT_SUCCESS(status))
			{
				tmp.pFileObject = fo;
				PDEVICE_OBJECT  relatedevice = IoGetRelatedDeviceObject(fo);
				PDRIVER_OBJECT relatedriver = NULL;
				PFAST_IO_DISPATCH relatedfastdispatch = NULL;
				tmp.dwMaximumSize.QuadPart = InputBuffer->size;
				if(relatedriver = relatedevice->DriverObject)
				{
					OriginQuery = relatedriver->MajorFunction[IRP_MJ_QUERY_INFORMATION];
					ppdd = &relatedriver->MajorFunction[IRP_MJ_QUERY_INFORMATION];
					relatedfastdispatch = relatedriver->FastIoDispatch;
					if(relatedfastdispatch && relatedfastdispatch->FastIoQueryStandardInfo)
					{
						OriginFastDispatch = relatedfastdispatch->FastIoQueryStandardInfo;
						ppfiqsi = &relatedfastdispatch->FastIoQueryStandardInfo;
					}
				}
				ObDereferenceObject(fo);
			}
			ExAcquireFastMutex(&gLock);
			if(gMappingData)
			{
				ExFreePool((PVOID)gMappingData);
				gMappingData = NULL;
			}
			gMappingData = (DEVICE_MAPPING*)ExAllocatePool(NonPagedPool,sizeof(DEVICE_MAPPING));
			if(gMappingData && ppdd)
			{
				RtlCopyMemory(gMappingData, &tmp, sizeof(DEVICE_MAPPING));
				*ppdd = DeviceMappingQueryInformation;
				if(ppfiqsi)
				{
					*ppfiqsi = DeviceMappingFastIo;
				}
			}
			ExReleaseFastMutex(&gLock);
			break;
		}
	case IOCTL_DISABLE_DEVICEMAPPING:
		{
			ExAcquireFastMutex(&gLock);
			if(gMappingData)
			{
				PFILE_OBJECT fo = NULL;
				PDEVICE_OBJECT relatedevice = NULL;
				PDRIVER_OBJECT relatedriver = NULL;
				status = ObReferenceObjectByHandle(gMappingData->hDevice, NULL, *IoFileObjectType, KernelMode, (PVOID*)&fo, NULL);
				if(NT_SUCCESS(status))
				{
					relatedevice = IoGetRelatedDeviceObject(fo);
					if(relatedevice)
						relatedriver = relatedevice->DriverObject;
					if(relatedriver && relatedriver->MajorFunction[IRP_MJ_QUERY_INFORMATION] == DeviceMappingQueryInformation)
					{
						relatedriver->MajorFunction[IRP_MJ_QUERY_INFORMATION] = OriginQuery;
					}
					if(relatedriver && relatedriver->FastIoDispatch && relatedriver->FastIoDispatch->FastIoQueryStandardInfo == DeviceMappingFastIo)
					{
						relatedriver->FastIoDispatch->FastIoQueryStandardInfo = OriginFastDispatch;
					}
					ObDereferenceObject(fo);
				}
				ExFreePool((PVOID)gMappingData);
				gMappingData = NULL;
			}
			ExReleaseFastMutex(&gLock);
			break;
		}
	default:
		status = STATUS_INVALID_VARIANT;
	}
	pIrp->IoStatus.Status = status;
	pIrp->IoStatus.Information = info;
	IoCompleteRequest(pIrp, IO_NO_INCREMENT);
	return status;
}

NTSTATUS DeviceMappingDriverDefaultDispatch(PDEVICE_OBJECT pDevObj, PIRP pIrp)
{
	pIrp->IoStatus.Status = STATUS_SUCCESS;
	pIrp->IoStatus.Information = 0;
	IoCompleteRequest(pIrp, IO_NO_INCREMENT);
	return STATUS_SUCCESS;
}


#pragma code_seg("INIT")
extern "C" NTSTATUS DriverEntry(PDRIVER_OBJECT pDriverObj, PUNICODE_STRING pRegistryString)
{
	UNICODE_STRING DevName;
	PDEVICE_OBJECT pDevObj = NULL;
	NTSTATUS status = STATUS_SUCCESS;
	pDriverObj->DriverUnload = DeviceMappingDriverUnload;
	pDriverObj->MajorFunction[IRP_MJ_CREATE] = DeviceMappingDriverDefaultDispatch;
	pDriverObj->MajorFunction[IRP_MJ_CLOSE] = DeviceMappingDriverDefaultDispatch;
	pDriverObj->MajorFunction[IRP_MJ_READ] = DeviceMappingDriverDefaultDispatch;
	pDriverObj->MajorFunction[IRP_MJ_WRITE] = DeviceMappingDriverDefaultDispatch;
	pDriverObj->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DeviceMappingIOControl;
	RtlInitUnicodeString(&DevName, L"\\Device\\DeviceMapping");
	ExInitializeFastMutex(&gLock);
	status = IoCreateDevice(pDriverObj, 0, &DevName, FILE_DEVICE_UNKNOWN, 0, TRUE, &pDevObj);
	if(NT_SUCCESS(status))
	{
		pDevObj->Flags |= DO_BUFFERED_IO;
		UNICODE_STRING SymLinkName;
		RtlInitUnicodeString(&SymLinkName, L"\\??\\DeviceMapping");
		IoCreateSymbolicLink(&SymLinkName, &DevName);
	}
	return status;
}

#define MAXNAME 256

#define IOCTL_ENABLE_DEVICEMAPPING CTL_CODE(FILE_DEVICE_UNKNOWN, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_DISABLE_DEVICEMAPPING CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_BUFFERED, FILE_ANY_ACCESS)

typedef struct _DEVICEMAPPING_IOCTL_DATA//和驱动通信所用data
{
	HANDLE hDevice;
	ULONG size;
}DEVICEMAPPING_IOCTL_DATA, *PDEVICEMAPPING_IOCTL_DATA;

#include <windows.h>
#include "DeviceMappingCommon.h"
#include <time.h>
#include <iostream>
using namespace std;
#define BUFSIZE 0x1000
#define WRITECOUNT 0x10000

void MappingRead(HANDLE hc)
{
	HANDLE hObj = CreateFileA("c:\\1.dat", GENERIC_ALL, 0, NULL, CREATE_NEW, FILE_ATTRIBUTE_NORMAL, NULL);
	if(hObj == INVALID_HANDLE_VALUE)
	{
		cout<<"Open 1.dat failed"<<endl;
		return;
	}
	HANDLE hMap = CreateFileMapping(hc, NULL, PAGE_READONLY, 0, BUFSIZE * WRITECOUNT, NULL);
	if(hMap)
	{
		PBYTE pbDev = (PBYTE)MapViewOfFile(hMap, FILE_MAP_COPY, 0, 0, 0);
		DWORD cbwrite;
		if(pbDev)
		{
			for(int i=0;i < WRITECOUNT; i++)
			{
				WriteFile(hObj, pbDev + i * BUFSIZE, BUFSIZE, &cbwrite, NULL);
			}
			UnmapViewOfFile(pbDev);
		}
		CloseHandle(hMap);
	}
	else
		cout<<"error:"<<GetLastError();
	CloseHandle(hObj);
}

void BufferedRead(HANDLE hc)
{
	BYTE buf[BUFSIZE];
	int count = 0;
	DWORD cbwrite;
	HANDLE hObj = CreateFileA("c:\\2.dat", GENERIC_ALL, 0, NULL, CREATE_NEW, FILE_ATTRIBUTE_NORMAL, NULL);
	if(hObj == INVALID_HANDLE_VALUE)
	{
		cout<<"Open 2.dat failed"<<endl;
		return;
	}
	while(ReadFile(hc, buf, BUFSIZE, &cbwrite, NULL) && cbwrite && count++ < WRITECOUNT)
	{
		WriteFile(hObj, buf, BUFSIZE, &cbwrite, NULL);
	}
	CloseHandle(hObj);
}

void main()
{
	HANDLE hFile;


	hFile = CreateFileA("\\\\.\\c:", GENERIC_READ, FILE_SHARE_READ | FILE_SHARE_WRITE,
		NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
	if(hFile == INVALID_HANDLE_VALUE)
	{
		cout<<"Open c: failed"<<endl;
		return;
	}
	
	time_t begin,end;
	_asm{int 3};
	HANDLE hDeviceMapping = CreateFileA("\\\\.\\DeviceMapping",0, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
	int i=GetLastError();
	if(hDeviceMapping != INVALID_HANDLE_VALUE)
	{
		DEVICEMAPPING_IOCTL_DATA did = {hFile, BUFSIZE * WRITECOUNT};
		DWORD retlen;
		DeviceIoControl(hDeviceMapping, IOCTL_ENABLE_DEVICEMAPPING, &did, sizeof(did), &did, 0, &retlen, NULL);
		begin = time(NULL);
		MappingRead(hFile);
		end = time(NULL);
		DeviceIoControl(hDeviceMapping, IOCTL_DISABLE_DEVICEMAPPING, &did, sizeof(did), &did, 0, &retlen, NULL);
		CloseHandle(hDeviceMapping);
		cout << "Mapping time:" <<end - begin << endl;
	}
	else
		cout<< "Error info:"<<GetLastError()<<endl;

	begin = time(NULL);
	BufferedRead(hFile);
	end=time(NULL);
	cout << "Buffer time:" << end - begin << endl;

	CloseHandle(hFile);
}

//#include <ntddk.h>
#include <ntifs.h>

typedef struct _HANDLE_TABLE 
{
	ULONG_PTR TableCode;
	PEPROCESS QuotaProcess;
	HANDLE UniqueProcessId;
#define HANDLE_TABLE_LOCKS 4
	EX_PUSH_LOCK HandleTableLock[HANDLE_TABLE_LOCKS];
	LIST_ENTRY HandleTableList;
	EX_PUSH_LOCK HandleContentionEvent;
	PVOID DebugInfo;
	LONG ExtraInfoPages;
	ULONG FirstFree;
	ULONG LastFree;
	ULONG NextHandleNeedingPool;
	LONG HandleCount;
	union 
	{
		ULONG Flags;
		BOOLEAN StrictFIFO : 1;
	};
} HANDLE_TABLE, *PHANDLE_TABLE;

typedef struct _HANDLE_TABLE_ENTRY 
{
	union 
	{
		PVOID Object;
		ULONG ObAttributes;
		PACCESS_MASK AuditMask;
		ULONG_PTR Value;
	};
	union 
	{
		union 
		{
			ACCESS_MASK GrantedAccess;
			struct 
			{
				USHORT GrantedAccessIndex;
				USHORT CreatorBackTraceIndex;
			};
		};
		LONG NextFreeTableEntry;
	};
} HANDLE_TABLE_ENTRY, *PHANDLE_TABLE_ENTRY;

extern "C" 
{
	typedef BOOLEAN (*EX_ENUMERATE_HANDLE_ROUTINE)(PHANDLE_TABLE_ENTRY,HANDLE,PVOID );
	BOOLEAN __stdcall ObFindHandleForObject(PEPROCESS,PVOID,PVOID,PVOID,PHANDLE);
	PHANDLE_TABLE __stdcall ObReferenceProcessHandleTable (PEPROCESS);
	BOOLEAN __stdcall ExEnumHandleTable (PHANDLE_TABLE, EX_ENUMERATE_HANDLE_ROUTINE ,PVOID,PHANDLE);
}

VOID Unload(PDRIVER_OBJECT pDriverObject)
{

}
PDRIVER_DISPATCH defaultdispatch=NULL;

NTSTATUS QuerySize (PDEVICE_OBJECT pDeviceObject, PIRP pIrp)
{
	PAGED_CODE();
	NTSTATUS status = STATUS_SUCCESS;
	PIO_STACK_LOCATION IrpSp = IoGetCurrentIrpStackLocation(pIrp);
	KdPrint(("MajorFunction=%d MinorFunction=%d\n",IrpSp->MajorFunction,IrpSp->MinorFunction));
	
	if(IrpSp->Parameters.QueryFile.FileInformationClass == FileStandardInformation)
	{
		IO_STATUS_BLOCK isb;
		FILE_FS_FULL_SIZE_INFORMATION ffsi;
		HANDLE handle = NULL;
		RtlZeroMemory(&ffsi,sizeof(ffsi));
		if(!IrpSp->FileObject){}
		else
		{
			if(ObFindHandleForObject(IoGetCurrentProcess(),IrpSp->FileObject,NULL,NULL,&handle))
			{
				status = NtQueryVolumeInformationFile(handle,&isb,&ffsi,sizeof(ffsi),FileFsFullSizeInformation);
				if(NT_SUCCESS(status))
				{
// 					KdPrint(("ActualAvailableAllocationUnits=%L\n",&ffsi.ActualAvailableAllocationUnits));
// 					KdPrint(("BytesPerSector=%d\n",ffsi.BytesPerSector));
// 					KdPrint(("TotalAllocationUnits=%L\n",&ffsi.TotalAllocationUnits));
// 					KdPrint(("CallerAvailableAllocationUnits=%L\n",&ffsi.CallerAvailableAllocationUnits));
// 					KdPrint(("SectorsPerAllocationUnit=%d\n",ffsi.SectorsPerAllocationUnit));
				}
			}
			PFILE_STANDARD_INFORMATION FileInformation = (PFILE_STANDARD_INFORMATION)pIrp->AssociatedIrp.SystemBuffer;
			FileInformation->EndOfFile.QuadPart = ffsi.TotalAllocationUnits.QuadPart;
			FileInformation->EndOfFile.QuadPart *= ffsi.SectorsPerAllocationUnit * ffsi.BytesPerSector;
			return STATUS_SUCCESS;
		}
	}
	return defaultdispatch(pDeviceObject,pIrp);
}

typedef struct _DEVICE_EXTENSION
{

}DEVICE_EXTENSION,*PDEVICE_EXTENSION;

extern "C" NTSTATUS DriverEntry(PDRIVER_OBJECT pDriverObj, PUNICODE_STRING pRegistryString)
{
	NTSTATUS status = STATUS_SUCCESS;
	HANDLE hfile;
	pDriverObj->DriverUnload=Unload;

	OBJECT_ATTRIBUTES oa;
	UNICODE_STRING drivename;
	IO_STATUS_BLOCK isb;
	PFILE_OBJECT fo;
	LARGE_INTEGER li={0};
	RtlInitUnicodeString(&drivename,L"\\??\\c:");
	InitializeObjectAttributes(&oa,&drivename,OBJ_CASE_INSENSITIVE,NULL,NULL);
	status = ZwOpenFile(&hfile,FILE_READ_DATA|FILE_READ_ATTRIBUTES|SYNCHRONIZE,&oa,&isb,
		FILE_SHARE_READ|FILE_SHARE_WRITE,FILE_OPEN_FOR_FREE_SPACE_QUERY|FILE_SYNCHRONOUS_IO_NONALERT);
	if(NT_SUCCESS(status))
	{
		KdPrint(("\n\nDriverEntry Entered\nHandle=%08x\n",hfile));
		status = ObReferenceObjectByHandle(hfile,0,*IoFileObjectType,KernelMode,(PVOID*)&fo,NULL);
		KdPrint(("fo=%p\n",fo));
		FILE_FS_FULL_SIZE_INFORMATION ffsi;
		status = NtQueryVolumeInformationFile(hfile,&isb,&ffsi,sizeof(ffsi),FileFsFullSizeInformation);
		status = FsRtlGetFileSize(fo,&li);
		PDRIVER_OBJECT passocdrive = IoGetRelatedDeviceObject(fo)->DriverObject;
		defaultdispatch = passocdrive->MajorFunction[IRP_MJ_QUERY_INFORMATION];
		passocdrive->MajorFunction[IRP_MJ_QUERY_INFORMATION]=QuerySize;

	//	PDEVICE_OBJECT DeviceObject = IoGetRelatedDeviceObject(fo);
		status = FsRtlGetFileSize(fo,&li);
		HANDLE sechandle;
		LARGE_INTEGER li={0};
		li.LowPart=0x1000;
		RtlInitUnicodeString(&drivename,L"\\??\\map1");
		status = ZwCreateSection(&sechandle,SECTION_MAP_READ,&oa,&li,PAGE_READONLY,SEC_COMMIT,hfile);
		PVOID addr=NULL;
		SIZE_T size=0;
		if(NT_SUCCESS(status))
		{
			status = ZwMapViewOfSection(sechandle,ZwCurrentProcess(),&addr,0,0x1000,NULL,&size,ViewShare, 
				MEM_TOP_DOWN,PAGE_READONLY);
			ULONG data=*(ULONG*)addr;
			data=0;
		}

		status = 0;
	}
	return STATUS_SUCCESS;
}
```

## 结论

&emsp;&emsp;分别用缓冲区方式和内存映射方式，分别读取c盘前256MB，分别写入文件中，Mapping time:23，Buffer time:26，可见还是有一些效果的~~~。获取的硬盘数据也确实是分区数据：

```Txt
文件内容：
Offset      0  1  2  3  4  5  6  7   8  9  A  B  C  D  E  F

 00000000   EB 52 90 4E 54 46 53 20  20 20 20 00 02 08 00 00   隦 NTFS         
 00000010   00 00 00 00 00 F8 00 00  3F 00 FF 00 3F 00 00 00        ? ?  ?   
00000020   00 00 00 00 80 00 80 00  B1 8C 7F 02 00 00 00 00       € € 睂      
00000030   00 00 0C 00 00 00 00 00  CB F8 27 00 00 00 00 00           锁'     
00000040   F6 00 00 00 01 00 00 00  CC 44 E9 BC 90 E9 BC 3C   ?      藾榧 榧<
00000050   00 00 00 00 FA 33 C0 8E  D0 BC 00 7C FB B8 C0 07       ?缼屑 |?
00000060   8E D8 E8 16 00 B8 00 0D  8E C0 33 DB C6 06 0E 00   庁? ? 幚3燮   
00000070   10 E8 53 00 68 00 0D 68  6A 02 CB 8A 16 24 00 B4    鑃 h  hj 藠 $ ?
00000080   08 CD 13 73 05 B9 FF FF  8A F1 66 0F B6 C6 40 66    ?s ?婑f 镀@f
 00000090   0F B6 D1 80 E2 3F F7 E2  86 CD C0 ED 06 41 66 0F    堆€?麾喭理 Af 
 000000A0   B7 C9 66 F7 E1 66 A3 20  00 C3 B4 41 BB AA 55 8A   飞f麽f? 么A华U?
 000000B0   16 24 00 CD 13 72 0F 81  FB 55 AA 75 09 F6 C1 01    $ ?r  鸘猽 隽 
000000C0   74 04 FE 06 14 00 C3 66  60 1E 06 66 A1 10 00 66   t ?  胒`  f? f
 000000D0   03 06 1C 00 66 3B 06 20  00 0F 82 3A 00 1E 66 6A       f;    ?  fj
 000000E0   00 66 50 06 53 66 68 10  00 01 00 80 3E 14 00 00    fP Sfh    €>   
000000F0   0F 85 0C 00 E8 B3 FF 80  3E 14 00 00 0F 84 61 00    ? 璩€>    刟 
00000100   B4 42 8A 16 24 00 16 1F  8B F4 CD 13 66 58 5B 07   碆?$   嬼?fX[ 
 00000110   66 58 66 58 1F EB 2D 66  33 D2 66 0F B7 0E 18 00   fXfX ?f3襢 ?  
00000120   66 F7 F1 FE C2 8A CA 66  8B D0 66 C1 EA 10 F7 36   f黢娛f嬓f陵 ?
00000130   1A 00 86 D6 8A 16 24 00  8A E8 C0 E4 06 0A CC B8     喼?$ 婅冷  谈
00000140   01 02 CD 13 0F 82 19 00  8C C0 05 20 00 8E C0 66     ? ? 尷   幚f
 00000150   FF 06 10 00 FF 0E 0E 00  0F 85 6F FF 07 1F 66 61          卭  fa
 00000160   C3 A0 F8 01 E8 09 00 A0  FB 01 E8 03 00 FB EB FE   脿?? 狖 ? ?
00000170   B4 01 8B F0 AC 3C 00 74  09 B4 0E BB 07 00 CD 10   ?嬸? t ?? ?
 00000180   EB F2 C3 0D 0A 41 20 64  69 73 6B 20 72 65 61 64   腧? A disk read
 00000190   20 65 72 72 6F 72 20 6F  63 63 75 72 72 65 64 00    error occurred 
 000001A0   0D 0A 4E 54 4C 44 52 20  69 73 20 6D 69 73 73 69     NTLDR is missi
 000001B0   6E 67 00 0D 0A 4E 54 4C  44 52 20 69 73 20 63 6F   ng   NTLDR is co
 000001C0   6D 70 72 65 73 73 65 64  00 0D 0A 50 72 65 73 73   mpressed   Press
 000001D0   20 43 74 72 6C 2B 41 6C  74 2B 44 65 6C 20 74 6F    Ctrl+Alt+Del to
 000001E0   20 72 65 73 74 61 72 74  0D 0A 00 00 00 00 00 00    restart        
 000001F0   00 00 00 00 00 00 00 00  83 A0 B3 C9 00 00 55 AA           儬成  U?
```
