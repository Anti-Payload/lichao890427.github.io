---
layout: post
title: 一种在Windows系统中强删正在运行文件的方法
categories: WindowsDriver
description: 一种在Windows系统中强删正在运行文件的方法
keywords: 
---

&emsp;&emsp;随便运行一个exe文件，，在它运行时进行删除操作，基本上会得到一个 无法删除的错误，下面我们就来破解这个限制。源码`DeleteFile -> NtSetInformationFile FileInformationClass = FileDispositionInformation`，函数结构如下：（参考wrk）

```C
	CurrentThread = PsGetCurrentThread();
	requestorMode = KeGetPreviousModeByThread(&CurrentThread->Tcb);
	status = ObReferenceObjectByHandle(FileHandle,
		IopQueryOperationAccess[FileInformationClass],IoFileObjectType,
		requestorMode,(PVOID *)&fileObject,&handleInformation);
	if (!(fileObject->Flags & FO_DIRECT_DEVICE_OPEN)) {
		deviceObject = IoGetRelatedDeviceObject(fileObject);
	}
	else {
		deviceObject = IoGetAttachedDevice(fileObject->DeviceObject);
	}
	fastIoDispatch = deviceObject->DriverObject->FastIoDispatch;
	if (fileObject->Flags & FO_SYNCHRONOUS_IO) {
		BOOLEAN interrupted;
		if (!IopAcquireFastLock(fileObject)) {
			status = IopAcquireFileObjectLock(fileObject,
				requestorMode,
				(BOOLEAN)((fileObject->Flags & FO_ALERTABLE_IO) != 0),
				&interrupted);
			if (interrupted) {
				ObDereferenceObject(fileObject);
				return status;
			}
		}
		event = ExAllocatePool(NonPagedPool, sizeof(KEVENT));
		if (event == NULL) {
			ObDereferenceObject(fileObject);
			return STATUS_INSUFFICIENT_RESOURCES;
		}
		KeInitializeEvent(event, SynchronizationEvent, FALSE);
		synchronousIo = FALSE;
	}
	KeClearEvent(&fileObject->Event);
	irp = IoAllocateIrp(deviceObject->StackSize, FALSE);
	if (!irp) {
		if (!(fileObject->Flags & FO_SYNCHRONOUS_IO)) {
			ExFreePool(event);
		}
		IopAllocateIrpCleanup(fileObject, (PKEVENT)NULL);
		return STATUS_INSUFFICIENT_RESOURCES;
	}
	irp->Tail.Overlay.OriginalFileObject = fileObject;
	irp->Tail.Overlay.Thread = CurrentThread;
	irp->RequestorMode = requestorMode;
	if (synchronousIo) {
		irp->UserEvent = (PKEVENT)NULL;
		irp->UserIosb = IoStatusBlock;
	}
	else {
		irp->UserEvent = event;
		irp->UserIosb = &localIoStatus;
		irp->Flags = IRP_SYNCHRONOUS_API;
	}
	irp->Overlay.AsynchronousParameters.UserApcRoutine = (PIO_APC_ROUTINE)NULL;
	irpSp = IoGetNextIrpStackLocation(irp);
	irpSp->MajorFunction = IRP_MJ_SET_INFORMATION;
	irpSp->FileObject = fileObject;
	irp->UserBuffer = FileInformation;
	irp->AssociatedIrp.SystemBuffer = (PVOID)NULL;
	irp->MdlAddress = (PMDL)NULL;
	try {
		irp->AssociatedIrp.SystemBuffer = ExAllocatePoolWithQuota(NonPagedPool,
			Length);
	} except(EXCEPTION_EXECUTE_HANDLER) {
		IopExceptionCleanup(fileObject,
			irp,
			(PKEVENT)NULL,
			event);
		return GetExceptionCode();
	}
	irp->Flags |= IRP_BUFFERED_IO |
		IRP_DEALLOCATE_BUFFER |
		IRP_INPUT_OPERATION |
		IRP_DEFER_IO_COMPLETION;
	irpSp->Parameters.QueryFile.Length = Length;
	irpSp->Parameters.QueryFile.FileInformationClass = FileInformationClass;
	IopQueueThreadIrp(irp);
	IopUpdateOtherOperationCount();
	if (FileInformationClass == FileDispositionInformation){
		PFILE_DISPOSITION_INFORMATION disposition = irp->AssociatedIrp.SystemBuffer;
		if (disposition->DeleteFile) {
			irpSp->Parameters.SetFile.DeleteHandle = FileHandle;
		}
		status = IoCallDriver(deviceObject, irp);
windbg查看deviceObject内存，得到driverObject内存，发现Call的Driver是\FileSystem\Ntfs，
现在反汇编该文件查看IRP_MJ_SET_INFORMATION派遣，用windbg下断：Ntfs!NtfsFsdSetInformation
NtfsFsdSetInformation->NtfsCommonSetInformation->NtfsSetDispositionInfo:
nt!MmFlushImageSection  该函数返回空 ->
nt!MiCheckControlAreaStatus

BOOLEAN MmFlushImageSection (__in PSECTION_OBJECT_POINTERS SectionPointer,__in MMFLUSH_TYPE FlushType)
{
    PLIST_ENTRY Next;
    PCONTROL_AREA ControlArea;
    PLARGE_CONTROL_AREA LargeControlArea;
    KIRQL OldIrql;
    LOGICAL state;
    if (FlushType == MmFlushForDelete) {
        LOCK_PFN (OldIrql);
        ControlArea = (PCONTROL_AREA)(SectionPointer->DataSectionObject);
        if (ControlArea != NULL) {
            if ((ControlArea->NumberOfUserReferences != 0) ||
                (ControlArea->u.Flags.BeingCreated)) {
                UNLOCK_PFN (OldIrql);
                return FALSE;
            }
        }
        UNLOCK_PFN (OldIrql);
    }
    state = MiCheckControlAreaStatus (CheckImageSection,SectionPointer,FALSE,&ControlArea,&OldIrql);
    if (ControlArea == NULL) {
        return (BOOLEAN) state;
    }
    do {
        ControlArea->u.Flags.BeingDeleted = 1;
        ControlArea->NumberOfMappedViews = 1;
        LargeControlArea = NULL;
        if (ControlArea->u.Flags.GlobalOnlyPerSession == 0) {
            NOTHING;
        }
        else if (IsListEmpty(&((PLARGE_CONTROL_AREA)ControlArea)->UserGlobalList)) {
            ASSERT (ControlArea ==(PCONTROL_AREA)SectionPointer->ImageSectionObject);
        }
        else {
            ASSERT (ControlArea->u.Flags.GlobalOnlyPerSession == 1);
            Next = ((PLARGE_CONTROL_AREA)ControlArea)->UserGlobalList.Flink;
            LargeControlArea = CONTAINING_RECORD (Next,LARGE_CONTROL_AREA,UserGlobalList);
            ASSERT (LargeControlArea->u.Flags.GlobalOnlyPerSession == 1);
            LargeControlArea->NumberOfSectionReferences += 1;
        }
        UNLOCK_PFN (OldIrql);
        MiCleanSection (ControlArea, TRUE);
        if (LargeControlArea != NULL) {
            state = MiCheckControlAreaStatus (CheckImageSection,SectionPointer,FALSE, &ControlArea,&OldIrql);
            if (!ControlArea) {
                LOCK_PFN (OldIrql);
                LargeControlArea->NumberOfSectionReferences -= 1;
                MiCheckControlArea ((PCONTROL_AREA)LargeControlArea,OldIrql);
            }
            else {
                LargeControlArea->NumberOfSectionReferences -= 1;
                MiCheckControlArea ((PCONTROL_AREA)LargeControlArea,OldIrql);
                LOCK_PFN (OldIrql);
            }
        }
        else {
            state = TRUE;
            break;
        }
    } while (ControlArea);
    return (BOOLEAN) state;
}

LOGICAL MiCheckControlAreaStatus (IN SECTION_CHECK_TYPE SectionCheckType,IN PSECTION_OBJECT_POINTERS SectionObjectPointers,IN ULONG DelayClose, OUT PCONTROL_AREA *ControlAreaOut, OUT PKIRQL PreviousIrql)
{
    PKTHREAD CurrentThread;
    PEVENT_COUNTER IoEvent;
    PEVENT_COUNTER SegmentEvent;
    LOGICAL DeallocateSegmentEvent;
    PCONTROL_AREA ControlArea;
    ULONG SectRef;
    KIRQL OldIrql;
    *ControlAreaOut = NULL;
    do {
        SegmentEvent = MiGetEventCounter ();
        if (SegmentEvent != NULL) {
            break;
        }
        KeDelayExecutionThread (KernelMode,
                                FALSE,
                                (PLARGE_INTEGER)&MmShortTime);
    } while (TRUE);
    LOCK_PFN (OldIrql);
    if (SectionCheckType != CheckImageSection) {
        ControlArea = ((PCONTROL_AREA)(SectionObjectPointers->DataSectionObject));
    }
    else {
        ControlArea = ((PCONTROL_AREA)(SectionObjectPointers->ImageSectionObject));
    }
    if (ControlArea == NULL) {
        if (SectionCheckType != CheckBothSection) {
            UNLOCK_PFN (OldIrql);
            MiFreeEventCounter (SegmentEvent);
            return TRUE;
        }
        else {
            ControlArea = ((PCONTROL_AREA)(SectionObjectPointers->ImageSectionObject));
            if (ControlArea == NULL) {
                UNLOCK_PFN (OldIrql);
                MiFreeEventCounter (SegmentEvent);
                return TRUE;
            }
        }
    }
    if (SectionCheckType != CheckUserDataSection) {
        SectRef = ControlArea->NumberOfSectionReferences;
    }
    else {
        SectRef = ControlArea->NumberOfUserReferences;
    }
    if ((SectRef != 0) ||
        (ControlArea->NumberOfMappedViews != 0) ||
        (ControlArea->u.Flags.BeingCreated)) {
        if (DelayClose) {
            ControlArea->u.Flags.DeleteOnClose = 1;
        }
        UNLOCK_PFN (OldIrql);
        MiFreeEventCounter (SegmentEvent);
        return FALSE;
    }
    if (ControlArea->u.Flags.BeingDeleted) {
        if (ControlArea->WaitingForDeletion == NULL) {
            DeallocateSegmentEvent = FALSE;
            ControlArea->WaitingForDeletion = SegmentEvent;
            IoEvent = SegmentEvent;
        }
        else {
            DeallocateSegmentEvent = TRUE;
            IoEvent = ControlArea->WaitingForDeletion;
            IoEvent->RefCount += 1;
        }
        CurrentThread = KeGetCurrentThread ();
        KeEnterCriticalRegionThread (CurrentThread);
        UNLOCK_PFN_AND_THEN_WAIT(OldIrql);
        KeWaitForSingleObject(&IoEvent->Event,
                              WrPageOut,
                              KernelMode,
                              FALSE,
                              (PLARGE_INTEGER)NULL);
        KeLeaveCriticalRegionThread (CurrentThread);
        MiFreeEventCounter (IoEvent);
        if (DeallocateSegmentEvent == TRUE) {
            MiFreeEventCounter (SegmentEvent);
        }
        return TRUE;
    }
    ASSERT (SegmentEvent->RefCount == 1);
    ASSERT (SegmentEvent->ListEntry.Next == NULL);
    InterlockedPushEntrySList (&MmEventCountSListHead,
                               (PSLIST_ENTRY)&SegmentEvent->ListEntry);
    *ControlAreaOut = ControlArea;
    *PreviousIrql = OldIrql;
    return FALSE;
}
```

看了以上代码逻辑，可知只需改一些结构体就可以实现：运行时删除，很简单的代码如下：

```C
NTSTATUS DriverIoctl(PDEVICE_OBJECT pDevObj,PIRP pIrp)
{
	NTSTATUS status = STATUS_SUCCESS;
	KdPrint(("Driver Ioctl\n"));
	PIO_STACK_LOCATION pStack = IoGetCurrentIrpStackLocation(pIrp);
	ULONG cbin = pStack->Parameters.DeviceIoControl.InputBufferLength;
	ULONG cbout = pStack->Parameters.DeviceIoControl.OutputBufferLength;
	PVOID buffer = pIrp->AssociatedIrp.SystemBuffer;
	ULONG ulOutputLen=0;

	switch(pStack->Parameters.DeviceIoControl.IoControlCode)
	{
	case IOCTL_IO_DELFILE:
		{
			PFILE_OBJECT pObject;
			status = ObReferenceObjectByHandle(*(HANDLE*)buffer,GENERIC_READ,*IoFileObjectType,KernelMode,(PVOID*)&pObject,NULL);
	 		if(NT_SUCCESS(status))
			{
				__try
				{
					PSCB Scb = (PSCB)pObject->FsContext;//NtfsDecodeFileObject
					if(Scb)
					{
						Scb->Fcb->Info.FileAttributes &= ~FILE_ATTRIBUTE_READONLY;//IsReadOnly
						PSECTION_OBJECT_POINTERS pop=&Scb->NonpagedScb->SegmentObject;
						pop->ImageSectionObject = NULL;
					}
				}
				__except(1)
				{

				}
				ObDereferenceObject(pObject);
			}
		}
		break;
	}

	pIrp->IoStatus.Status = status;
	pIrp->IoStatus.Information = ulOutputLen;
	IoCompleteRequest(pIrp,IO_NO_INCREMENT);
	return status;
}
```

```C
#include <iostream>
#include <windows.h>
using namespace std;

#define IOCTL_IO_DELFILE CTL_CODE(FILE_DEVICE_UNKNOWN,0x800,METHOD_BUFFERED,FILE_ANY_ACCESS)//获取驱动加载信息
int _tmain(int argc, _TCHAR* argv[])
{
	if (argc != 2)
	{
		cout << "参数不够" << endl;
	}

	HANDLE hFile = CreateFileW(argv[1], 0, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
	if (hFile != INVALID_HANDLE_VALUE)
	{

		HANDLE hDev = CreateFileW(L"\\\\.\\NtMem", GENERIC_ALL, 0, NULL, OPEN_EXISTING, 0, NULL);
		if (hDev != INVALID_HANDLE_VALUE)
		{
			DWORD nop = 0;
			DeviceIoControl(hDev, IOCTL_IO_DELFILE, (LPVOID)&hFile, sizeof(HANDLE), (LPVOID)&hFile, 0, &nop, NULL);

			CloseHandle(hDev);
		}
		else
		{
			cout << "无法打开设备!" << endl;
		}
		CloseHandle(hFile);
	}
	else
	{
		cout << "无法打开文件!" << endl;
	}

	return 0;
}
```
    xp 下测试删文件很轻松，还小试了一把360，没成功，发现根本未能进NtSetInformationFile这一层，估计在应用层做了hook。下面是打算找到线性地址对应物理地址的代码，然而没有成功：

```C
#define MiGetPdePart(va) (((ULONG)va>>22)&0x3FF)
#define MiGetPtePart(va) (((ULONG)va>>12)&0x3FF)
#define MiGetOffsetPart(va) ((ULONG)va&0xFFF)
ULONG cr3val=0;
_asm
{
    mov eax,cr3;
    mov cr3val,eax;
}
PVOID oriaddr=ExAllocatePool(NonPagedPool,0x1000);
*(PULONG)oriaddr = 0x12345;
PMMPTE PointerPte1,PointerPte2;

PULONG pdaddr;
LARGE_INTEGER mapaddr;
mapaddr.QuadPart = cr3val;
pdaddr = (PULONG)MmMapIoSpace(mapaddr,0x1000,MmWriteCombined);
if(pdaddr)
{
    PointerPte1 = (PMMPTE)&pdaddr[MiGetPdePart(oriaddr)];
    if(PointerPte1->u.Hard.LargePage == 0)
    {
        mapaddr.QuadPart = PointerPte1->u.Hard.PageFrameNumber;
        pdaddr = (PULONG)MmMapIoSpace(mapaddr,0x1000,MmWriteCombined);
        if(pdaddr)
        {
            PointerPte1 = (PMMPTE)&pdaddr[MiGetPtePart(oriaddr)];
            mapaddr.QuadPart = PointerPte1->u.Hard.PageFrameNumber;
            pdaddr= (PULONG)MmMapIoSpace(mapaddr,0x1000,MmWriteCombined);
            if(pdaddr)
            {
                PCHAR objaddr=(PCHAR)pdaddr+MiGetOffsetPart(oriaddr);
                objaddr = NULL;
            }
        }
    }
}
```
