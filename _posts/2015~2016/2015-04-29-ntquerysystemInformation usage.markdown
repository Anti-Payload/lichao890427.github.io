---
layout: post
title: NtQuerySystemInformation用法详解
categories: Windows
description: NtQuerySystemInformation用法详解
keywords: 
---

# NtQuerySystemInformation用法详解

## NtDefs.h

```C++
#include <basetsd.h>

#define IN
#define OUT
#define OPTIONAL
#define NT_SUCCESS(Status) (((NTSTATUS)(Status)) >= 0)
#define MAX_STACK_DEPTH 32
#define MAXIMUM_NUMA_NODES 16
#define STATUS_SUCCESS                   ((NTSTATUS)0x00000000L)    // ntsubauth

typedef unsigned long ULONG;
typedef unsigned int DWORD;
typedef unsigned short WORD;
typedef unsigned char UCHAR;
typedef unsigned short USHORT;
typedef long LONG;
typedef LONG NTSTATUS;
typedef void *PVOID;
typedef ULONG *PULONG;
typedef ULONG_PTR KAFFINITY;
typedef char CCHAR;
typedef void * HANDLE;
typedef UCHAR *PUCHAR;
typedef unsigned int UINT;
typedef void *LPVOID;
typedef SIZE_T SYSINF_PAGE_COUNT;
typedef LONG KPRIORITY;
typedef wchar_t WCHAR;
typedef WCHAR *NWPSTR, *LPWSTR, *PWSTR;
typedef char CHAR;
typedef CHAR *PCHAR, *LPCH, *PCH;
typedef DWORD ACCESS_MASK;

typedef struct _UNICODE_STRING {
	USHORT Length;
	USHORT MaximumLength;
#ifdef MIDL_PASS
	[size_is(MaximumLength / 2), length_is((Length) / 2)] USHORT * Buffer;
#else // MIDL_PASS
	PWSTR  Buffer;
#endif // MIDL_PASS
} UNICODE_STRING;
typedef UNICODE_STRING *PUNICODE_STRING;

#if (!defined (_MAC) && (!defined(MIDL_PASS) || defined(__midl)) && (!defined(_M_IX86) || (defined(_INTEGRAL_MAX_BITS) && _INTEGRAL_MAX_BITS >= 64)))
typedef __int64 LONGLONG;
typedef unsigned __int64 ULONGLONG;
#define MAXLONGLONG                         (0x7fffffffffffffff)
#else
#if defined(_MAC) && defined(_MAC_INT_64)
typedef __int64 LONGLONG;
typedef unsigned __int64 ULONGLONG;
#define MAXLONGLONG                      (0x7fffffffffffffff)
#else
typedef double LONGLONG;
typedef double ULONGLONG;
#endif //_MAC and int64
#endif

#if defined(MIDL_PASS)
typedef struct _LARGE_INTEGER
{
#else // MIDL_PASS
typedef union _LARGE_INTEGER
{
	struct
	{
		DWORD LowPart;
		LONG HighPart;
	} DUMMYSTRUCTNAME;
	struct
	{
		DWORD LowPart;
		LONG HighPart;
	} u;
#endif //MIDL_PASS
	LONGLONG QuadPart;
} LARGE_INTEGER;

typedef unsigned char BYTE;
typedef BYTE BOOLEAN;

#define FLG_STOP_ON_EXCEPTION           0x00000001      // user and kernel mode
#define FLG_SHOW_LDR_SNAPS              0x00000002      // user and kernel mode
#define FLG_DEBUG_INITIAL_COMMAND       0x00000004      // kernel mode only up until WINLOGON started
#define FLG_STOP_ON_HUNG_GUI            0x00000008      // kernel mode only while running

#define FLG_HEAP_ENABLE_TAIL_CHECK      0x00000010      // user mode only
#define FLG_HEAP_ENABLE_FREE_CHECK      0x00000020      // user mode only
#define FLG_HEAP_VALIDATE_PARAMETERS    0x00000040      // user mode only
#define FLG_HEAP_VALIDATE_ALL           0x00000080      // user mode only

#define FLG_APPLICATION_VERIFIER        0x00000100      // user mode only
#define FLG_POOL_ENABLE_TAGGING         0x00000400      // kernel mode only
#define FLG_HEAP_ENABLE_TAGGING         0x00000800      // user mode only

#define FLG_USER_STACK_TRACE_DB         0x00001000      // x86 user mode only
#define FLG_KERNEL_STACK_TRACE_DB       0x00002000      // x86 kernel mode only at boot time
#define FLG_MAINTAIN_OBJECT_TYPELIST    0x00004000      // kernel mode only at boot time
#define FLG_HEAP_ENABLE_TAG_BY_DLL      0x00008000      // user mode only

#define FLG_DISABLE_STACK_EXTENSION     0x00010000      // user mode only
#define FLG_ENABLE_CSRDEBUG             0x00020000      // kernel mode only at boot time
#define FLG_ENABLE_KDEBUG_SYMBOL_LOAD   0x00040000      // kernel mode only
#define FLG_DISABLE_PAGE_KERNEL_STACKS  0x00080000      // kernel mode only at boot time

#define FLG_ENABLE_SYSTEM_CRIT_BREAKS   0x00100000      // user mode only
#define FLG_HEAP_DISABLE_COALESCING     0x00200000      // user mode only
#define FLG_ENABLE_CLOSE_EXCEPTIONS     0x00400000      // kernel mode only
#define FLG_ENABLE_EXCEPTION_LOGGING    0x00800000      // kernel mode only

#define FLG_ENABLE_HANDLE_TYPE_TAGGING  0x01000000      // kernel mode only
#define FLG_HEAP_PAGE_ALLOCS            0x02000000      // user mode only
#define FLG_DEBUG_INITIAL_COMMAND_EX    0x04000000      // kernel mode only up until WINLOGON started
#define FLG_DISABLE_DBGPRINT            0x08000000      // kernel mode only

#define FLG_CRITSEC_EVENT_CREATION      0x10000000      // user mode only, Force early creation of resource events
#define FLG_LDR_TOP_DOWN                0x20000000      // user mode only, win64 only
#define FLG_ENABLE_HANDLE_EXCEPTIONS    0x40000000      // kernel mode only
#define FLG_DISABLE_PROTDLLS            0x80000000      // user mode only (smss/winlogon)

#define PROCESSOR_ARCHITECTURE_INTEL            0
#define PROCESSOR_ARCHITECTURE_MIPS             1
#define PROCESSOR_ARCHITECTURE_ALPHA            2
#define PROCESSOR_ARCHITECTURE_PPC              3
#define PROCESSOR_ARCHITECTURE_SHX              4
#define PROCESSOR_ARCHITECTURE_ARM              5
#define PROCESSOR_ARCHITECTURE_IA64             6
#define PROCESSOR_ARCHITECTURE_ALPHA64          7
#define PROCESSOR_ARCHITECTURE_MSIL             8
#define PROCESSOR_ARCHITECTURE_AMD64            9
#define PROCESSOR_ARCHITECTURE_IA32_ON_WIN64    10

#define STATUS_INFO_LENGTH_MISMATCH      ((NTSTATUS)0xC0000004L)
#define STATUS_INVALID_INFO_CLASS ((NTSTATUS)0xC0000003L) 
#define STATUS_ACCESS_VIOLATION ((DWORD )0xC0000005L)
#define STATUS_INSUFFICIENT_RESOURCES ((NTSTATUS)0xC000009AL) 
#define STATUS_WORKING_SET_QUOTA ((NTSTATUS)0xC00000A1L)
#define STATUS_NOT_IMPLEMENTED ((NTSTATUS)0xC0000002L)

typedef enum _SYSTEM_INFORMATION_CLASS
{
	SystemBasicInformation,
	SystemProcessorInformation,              // obsolete...delete
	SystemPerformanceInformation,
	SystemTimeOfDayInformation,
	SystemPathInformation,
	SystemProcessInformation,                //系统进程信息
	SystemCallCountInformation,
	SystemDeviceInformation,
	SystemProcessorPerformanceInformation,
	SystemFlagsInformation,
	SystemCallTimeInformation,
	SystemModuleInformation,     //系统模块
	SystemLocksInformation,
	SystemStackTraceInformation,
	SystemPagedPoolInformation,
	SystemNonPagedPoolInformation,
	SystemHandleInformation,
	SystemObjectInformation,
	SystemPageFileInformation,
	SystemVdmInstemulInformation,
	SystemVdmBopInformation,
	SystemFileCacheInformation,
	SystemPoolTagInformation,
	SystemInterruptInformation,
	SystemDpcBehaviorInformation,
	SystemFullMemoryInformation,
	SystemLoadGdiDriverInformation,
	SystemUnloadGdiDriverInformation,
	SystemTimeAdjustmentInformation,
	SystemSummaryMemoryInformation,
	SystemMirrorMemoryInformation,
	SystemPerformanceTraceInformation,
	SystemObsolete0,
	SystemExceptionInformation,
	SystemCrashDumpStateInformation,
	SystemKernelDebuggerInformation,
	SystemContextSwitchInformation,
	SystemRegistryQuotaInformation,
	SystemExtendServiceTableInformation,
	SystemPrioritySeperation,
	SystemVerifierAddDriverInformation,
	SystemVerifierRemoveDriverInformation,
	SystemProcessorIdleInformation,
	SystemLegacyDriverInformation,
	SystemCurrentTimeZoneInformation,
	SystemLookasideInformation,
	SystemTimeSlipNotification,
	SystemSessionCreate,
	SystemSessionDetach,
	SystemSessionInformation,
	SystemRangeStartInformation,
	SystemVerifierInformation,
	SystemVerifierThunkExtend,
	SystemSessionProcessInformation,
	SystemLoadGdiDriverInSystemSpace,
	SystemNumaProcessorMap,
	SystemPrefetcherInformation,
	SystemExtendedProcessInformation,
	SystemRecommendedSharedDataAlignment,
	SystemComPlusPackage,
	SystemNumaAvailableMemory,
	SystemProcessorPowerInformation,
	SystemEmulationBasicInformation,//=SystemBasicInformation
	SystemEmulationProcessorInformation,//=SystemProcessorInformation
	SystemExtendedHandleInformation,
	SystemLostDelayedWriteInformation,
	SystemBigPoolInformation,
	SystemSessionPoolTagInformation,
	SystemSessionMappedViewInformation,
	SystemHotpatchInformation,
	SystemObjectSecurityMode,
	SystemWatchdogTimerHandler,
	SystemWatchdogTimerInformation,
	SystemLogicalProcessorInformation,
	SystemWow64SharedInformation,
	SystemRegisterFirmwareTableInformationHandler,
	SystemFirmwareTableInformation,
	SystemModuleInformationEx,
	SystemVerifierTriageInformation,
	SystemSuperfetchInformation,
	SystemMemoryListInformation,
	SystemFileCacheInformationEx,

	//100?
	SystemPageMemoryInformation = 123,//自定义
	SystemPolicyInformation = 134,

} SYSTEM_INFORMATION_CLASS, *PSYSTEM_INFORMATION_CLASS;

typedef struct _SYSTEM_BASIC_INFORMATION 
{
	ULONG Reserved;
	ULONG TimerResolution;
	ULONG PageSize;
	SYSINF_PAGE_COUNT NumberOfPhysicalPages;
	SYSINF_PAGE_COUNT LowestPhysicalPageNumber;
	SYSINF_PAGE_COUNT HighestPhysicalPageNumber;
	ULONG AllocationGranularity;
	ULONG_PTR MinimumUserModeAddress;
	ULONG_PTR MaximumUserModeAddress;
	ULONG_PTR ActiveProcessorsAffinityMask;
	CCHAR NumberOfProcessors;
} SYSTEM_BASIC_INFORMATION, *PSYSTEM_BASIC_INFORMATION;

typedef struct _SYSTEM_PROCESSOR_INFORMATION 
{
	USHORT ProcessorArchitecture;
	USHORT ProcessorLevel;
	USHORT ProcessorRevision;
	USHORT Reserved;
	ULONG ProcessorFeatureBits;
} SYSTEM_PROCESSOR_INFORMATION, *PSYSTEM_PROCESSOR_INFORMATION;

typedef struct _SYSTEM_PERFORMANCE_INFORMATION 
{
	LARGE_INTEGER IdleProcessTime;
	LARGE_INTEGER IoReadTransferCount;
	LARGE_INTEGER IoWriteTransferCount;
	LARGE_INTEGER IoOtherTransferCount;
	ULONG IoReadOperationCount;
	ULONG IoWriteOperationCount;
	ULONG IoOtherOperationCount;
	ULONG AvailablePages;
	SYSINF_PAGE_COUNT CommittedPages;
	SYSINF_PAGE_COUNT CommitLimit;
	SYSINF_PAGE_COUNT PeakCommitment;
	ULONG PageFaultCount;
	ULONG CopyOnWriteCount;
	ULONG TransitionCount;
	ULONG CacheTransitionCount;
	ULONG DemandZeroCount;
	ULONG PageReadCount;
	ULONG PageReadIoCount;
	ULONG CacheReadCount;
	ULONG CacheIoCount;
	ULONG DirtyPagesWriteCount;
	ULONG DirtyWriteIoCount;
	ULONG MappedPagesWriteCount;
	ULONG MappedWriteIoCount;
	ULONG PagedPoolPages;
	ULONG NonPagedPoolPages;
	ULONG PagedPoolAllocs;
	ULONG PagedPoolFrees;
	ULONG NonPagedPoolAllocs;
	ULONG NonPagedPoolFrees;
	ULONG FreeSystemPtes;
	ULONG ResidentSystemCodePage;
	ULONG TotalSystemDriverPages;
	ULONG TotalSystemCodePages;
	ULONG NonPagedPoolLookasideHits;
	ULONG PagedPoolLookasideHits;
	ULONG AvailablePagedPoolPages;
	ULONG ResidentSystemCachePage;
	ULONG ResidentPagedPoolPage;
	ULONG ResidentSystemDriverPage;
	ULONG CcFastReadNoWait;
	ULONG CcFastReadWait;
	ULONG CcFastReadResourceMiss;
	ULONG CcFastReadNotPossible;
	ULONG CcFastMdlReadNoWait;
	ULONG CcFastMdlReadWait;
	ULONG CcFastMdlReadResourceMiss;
	ULONG CcFastMdlReadNotPossible;
	ULONG CcMapDataNoWait;
	ULONG CcMapDataWait;
	ULONG CcMapDataNoWaitMiss;
	ULONG CcMapDataWaitMiss;
	ULONG CcPinMappedDataCount;
	ULONG CcPinReadNoWait;
	ULONG CcPinReadWait;
	ULONG CcPinReadNoWaitMiss;
	ULONG CcPinReadWaitMiss;
	ULONG CcCopyReadNoWait;
	ULONG CcCopyReadWait;
	ULONG CcCopyReadNoWaitMiss;
	ULONG CcCopyReadWaitMiss;
	ULONG CcMdlReadNoWait;
	ULONG CcMdlReadWait;
	ULONG CcMdlReadNoWaitMiss;
	ULONG CcMdlReadWaitMiss;
	ULONG CcReadAheadIos;
	ULONG CcLazyWriteIos;
	ULONG CcLazyWritePages;
	ULONG CcDataFlushes;
	ULONG CcDataPages;
	ULONG ContextSwitches;
	ULONG FirstLevelTbFills;
	ULONG SecondLevelTbFills;
	ULONG SystemCalls;
} SYSTEM_PERFORMANCE_INFORMATION, *PSYSTEM_PERFORMANCE_INFORMATION;

typedef struct _SYSTEM_TIMEOFDAY_INFORMATION 
{
	LARGE_INTEGER BootTime;
	LARGE_INTEGER CurrentTime;
	LARGE_INTEGER TimeZoneBias;
	ULONG TimeZoneId;
	ULONG Reserved;
	ULONGLONG BootTimeBias;
	ULONGLONG SleepTimeBias;
} SYSTEM_TIMEOFDAY_INFORMATION, *PSYSTEM_TIMEOFDAY_INFORMATION;

typedef struct _SYSTEM_PROCESS_INFORMATION 
{
	ULONG NextEntryOffset;
	ULONG NumberOfThreads;
	LARGE_INTEGER SpareLi1;
	LARGE_INTEGER SpareLi2;
	LARGE_INTEGER SpareLi3;
	LARGE_INTEGER CreateTime;
	LARGE_INTEGER UserTime;
	LARGE_INTEGER KernelTime;
	UNICODE_STRING ImageName;
	KPRIORITY BasePriority;
	HANDLE UniqueProcessId;
	HANDLE InheritedFromUniqueProcessId;
	ULONG HandleCount;
	ULONG SessionId;
	ULONG_PTR PageDirectoryBase;
	SIZE_T PeakVirtualSize;
	SIZE_T VirtualSize;
	ULONG PageFaultCount;
	SIZE_T PeakWorkingSetSize;
	SIZE_T WorkingSetSize;
	SIZE_T QuotaPeakPagedPoolUsage;
	SIZE_T QuotaPagedPoolUsage;
	SIZE_T QuotaPeakNonPagedPoolUsage;
	SIZE_T QuotaNonPagedPoolUsage;
	SIZE_T PagefileUsage;
	SIZE_T PeakPagefileUsage;
	SIZE_T PrivatePageCount;
	LARGE_INTEGER ReadOperationCount;
	LARGE_INTEGER WriteOperationCount;
	LARGE_INTEGER OtherOperationCount;
	LARGE_INTEGER ReadTransferCount;
	LARGE_INTEGER WriteTransferCount;
	LARGE_INTEGER OtherTransferCount;
} SYSTEM_PROCESS_INFORMATION, *PSYSTEM_PROCESS_INFORMATION;

typedef struct _SYSTEM_CALL_COUNT_INFORMATION 
{
	ULONG Length;
	ULONG NumberOfTables;
} SYSTEM_CALL_COUNT_INFORMATION, *PSYSTEM_CALL_COUNT_INFORMATION;

typedef struct _SYSTEM_DEVICE_INFORMATION 
{
	ULONG NumberOfDisks;
	ULONG NumberOfFloppies;
	ULONG NumberOfCdRoms;
	ULONG NumberOfTapes;
	ULONG NumberOfSerialPorts;
	ULONG NumberOfParallelPorts;
} SYSTEM_DEVICE_INFORMATION, *PSYSTEM_DEVICE_INFORMATION;

typedef struct _SYSTEM_PROCESSOR_PERFORMANCE_INFORMATION 
{
	LARGE_INTEGER IdleTime;
	LARGE_INTEGER KernelTime;
	LARGE_INTEGER UserTime;
	LARGE_INTEGER DpcTime;          // DEVL only
	LARGE_INTEGER InterruptTime;    // DEVL only
	ULONG InterruptCount;
} SYSTEM_PROCESSOR_PERFORMANCE_INFORMATION, *PSYSTEM_PROCESSOR_PERFORMANCE_INFORMATION;

typedef struct _SYSTEM_FLAGS_INFORMATION 
{
	ULONG Flags;
} SYSTEM_FLAGS_INFORMATION, *PSYSTEM_FLAGS_INFORMATION;

typedef struct _RTL_PROCESS_MODULE_INFORMATION
{
	HANDLE Section;                 // Not filled in
	PVOID MappedBase;
	PVOID ImageBase;
	ULONG ImageSize;
	ULONG Flags;
	USHORT LoadOrderIndex;
	USHORT InitOrderIndex;
	USHORT LoadCount;
	USHORT OffsetToFileName;
	UCHAR  FullPathName[256];
} RTL_PROCESS_MODULE_INFORMATION, *PRTL_PROCESS_MODULE_INFORMATION;

typedef struct _RTL_PROCESS_MODULES
{
	ULONG NumberOfModules;
	RTL_PROCESS_MODULE_INFORMATION Modules[1];
} RTL_PROCESS_MODULES, *PRTL_PROCESS_MODULES;

typedef struct _RTL_PROCESS_LOCK_INFORMATION
{
	PVOID Address;
	USHORT Type;
	USHORT CreatorBackTraceIndex;
	HANDLE OwningThread;        // from the thread's ClientId->UniqueThread
	LONG LockCount;
	ULONG ContentionCount;
	ULONG EntryCount;
	LONG RecursionCount;
	ULONG NumberOfWaitingShared;
	ULONG NumberOfWaitingExclusive;
} RTL_PROCESS_LOCK_INFORMATION, *PRTL_PROCESS_LOCK_INFORMATION;

typedef struct _RTL_PROCESS_LOCKS
{
	ULONG NumberOfLocks;
	RTL_PROCESS_LOCK_INFORMATION Locks[1];
} RTL_PROCESS_LOCKS, *PRTL_PROCESS_LOCKS;


typedef struct _RTL_PROCESS_BACKTRACE_INFORMATION
{
	PCHAR SymbolicBackTrace;        // Not filled in
	ULONG TraceCount;
	USHORT Index;
	USHORT Depth;
	PVOID BackTrace[MAX_STACK_DEPTH];
} RTL_PROCESS_BACKTRACE_INFORMATION, *PRTL_PROCESS_BACKTRACE_INFORMATION;

typedef struct _RTL_PROCESS_BACKTRACES
{
	ULONG CommittedMemory;
	ULONG ReservedMemory;
	ULONG NumberOfBackTraceLookups;
	ULONG NumberOfBackTraces;
	RTL_PROCESS_BACKTRACE_INFORMATION BackTraces[1];
} RTL_PROCESS_BACKTRACES, *PRTL_PROCESS_BACKTRACES;

typedef struct _SYSTEM_HANDLE_TABLE_ENTRY_INFO
{
	USHORT UniqueProcessId;
	USHORT CreatorBackTraceIndex;
	UCHAR ObjectTypeIndex;
	UCHAR HandleAttributes;
	USHORT HandleValue;
	PVOID Object;
	ULONG GrantedAccess;
} SYSTEM_HANDLE_TABLE_ENTRY_INFO, *PSYSTEM_HANDLE_TABLE_ENTRY_INFO;

typedef struct _SYSTEM_HANDLE_INFORMATION
{
	ULONG NumberOfHandles;
	SYSTEM_HANDLE_TABLE_ENTRY_INFO Handles[1];
} SYSTEM_HANDLE_INFORMATION, *PSYSTEM_HANDLE_INFORMATION;

typedef struct _GENERIC_MAPPING
{
	ACCESS_MASK GenericRead;
	ACCESS_MASK GenericWrite;
	ACCESS_MASK GenericExecute;
	ACCESS_MASK GenericAll;
} GENERIC_MAPPING;
typedef GENERIC_MAPPING *PGENERIC_MAPPING;

typedef struct _SYSTEM_OBJECTTYPE_INFORMATION
{
	ULONG NextEntryOffset;
	ULONG NumberOfObjects;
	ULONG NumberOfHandles;
	ULONG TypeIndex;
	ULONG InvalidAttributes;
	GENERIC_MAPPING GenericMapping;
	ULONG ValidAccessMask;
	ULONG PoolType;
	BOOLEAN SecurityRequired;
	BOOLEAN WaitableObject;
	UNICODE_STRING TypeName;
} SYSTEM_OBJECTTYPE_INFORMATION, *PSYSTEM_OBJECTTYPE_INFORMATION;

typedef struct _OBJECT_NAME_INFORMATION
{
	UNICODE_STRING Name;
} OBJECT_NAME_INFORMATION, *POBJECT_NAME_INFORMATION;

typedef struct _SYSTEM_OBJECT_INFORMATION
{
	ULONG NextEntryOffset;
	PVOID Object;
	HANDLE CreatorUniqueProcess;
	USHORT CreatorBackTraceIndex;
	USHORT Flags;
	LONG PointerCount;
	LONG HandleCount;
	ULONG PagedPoolCharge;
	ULONG NonPagedPoolCharge;
	HANDLE ExclusiveProcessId;
	PVOID SecurityDescriptor;
	OBJECT_NAME_INFORMATION NameInfo;
} SYSTEM_OBJECT_INFORMATION, *PSYSTEM_OBJECT_INFORMATION;

typedef struct _SYSTEM_PAGEFILE_INFORMATION
{
	ULONG NextEntryOffset;
	ULONG TotalSize;
	ULONG TotalInUse;
	ULONG PeakUsage;
	UNICODE_STRING PageFileName;
} SYSTEM_PAGEFILE_INFORMATION, *PSYSTEM_PAGEFILE_INFORMATION;

typedef struct _SYSTEM_VDM_INSTEMUL_INFO
{
	ULONG SegmentNotPresent;
	ULONG VdmOpcode0F;
	ULONG OpcodeESPrefix;
	ULONG OpcodeCSPrefix;
	ULONG OpcodeSSPrefix;
	ULONG OpcodeDSPrefix;
	ULONG OpcodeFSPrefix;
	ULONG OpcodeGSPrefix;
	ULONG OpcodeOPER32Prefix;
	ULONG OpcodeADDR32Prefix;
	ULONG OpcodeINSB;
	ULONG OpcodeINSW;
	ULONG OpcodeOUTSB;
	ULONG OpcodeOUTSW;
	ULONG OpcodePUSHF;
	ULONG OpcodePOPF;
	ULONG OpcodeINTnn;
	ULONG OpcodeINTO;
	ULONG OpcodeIRET;
	ULONG OpcodeINBimm;
	ULONG OpcodeINWimm;
	ULONG OpcodeOUTBimm;
	ULONG OpcodeOUTWimm;
	ULONG OpcodeINB;
	ULONG OpcodeINW;
	ULONG OpcodeOUTB;
	ULONG OpcodeOUTW;
	ULONG OpcodeLOCKPrefix;
	ULONG OpcodeREPNEPrefix;
	ULONG OpcodeREPPrefix;
	ULONG OpcodeHLT;
	ULONG OpcodeCLI;
	ULONG OpcodeSTI;
	ULONG BopCount;
} SYSTEM_VDM_INSTEMUL_INFO, *PSYSTEM_VDM_INSTEMUL_INFO;

typedef struct _SYSTEM_FILECACHE_INFORMATION
{
	SIZE_T CurrentSize;
	SIZE_T PeakSize;
	ULONG PageFaultCount;
	SIZE_T MinimumWorkingSet;
	SIZE_T MaximumWorkingSet;
	SIZE_T CurrentSizeIncludingTransitionInPages;
	SIZE_T PeakSizeIncludingTransitionInPages;
	ULONG TransitionRePurposeCount;
	ULONG Flags;
} SYSTEM_FILECACHE_INFORMATION, *PSYSTEM_FILECACHE_INFORMATION;

typedef struct _SYSTEM_POOLTAG 
{
	union 
	{
		UCHAR Tag[4];
		ULONG TagUlong;
	};
	ULONG PagedAllocs;
	ULONG PagedFrees;
	SIZE_T PagedUsed;
	ULONG NonPagedAllocs;
	ULONG NonPagedFrees;
	SIZE_T NonPagedUsed;
} SYSTEM_POOLTAG, *PSYSTEM_POOLTAG;

typedef struct _SYSTEM_POOLTAG_INFORMATION 
{
	ULONG Count;
	SYSTEM_POOLTAG TagInfo[1];
} SYSTEM_POOLTAG_INFORMATION, *PSYSTEM_POOLTAG_INFORMATION;

typedef struct _SYSTEM_INTERRUPT_INFORMATION 
{
	ULONG ContextSwitches;
	ULONG DpcCount;
	ULONG DpcRate;
	ULONG TimeIncrement;
	ULONG DpcBypassCount;
	ULONG ApcBypassCount;
} SYSTEM_INTERRUPT_INFORMATION, *PSYSTEM_INTERRUPT_INFORMATION;

typedef struct _SYSTEM_DPC_BEHAVIOR_INFORMATION 
{
	ULONG Spare;
	ULONG DpcQueueDepth;
	ULONG MinimumDpcRate;
	ULONG AdjustDpcThreshold;
	ULONG IdealDpcRate;
} SYSTEM_DPC_BEHAVIOR_INFORMATION, *PSYSTEM_DPC_BEHAVIOR_INFORMATION;

typedef struct _SYSTEM_MEMORY_INFO
{
	PUCHAR StringOffset;
	USHORT ValidCount;
	USHORT TransitionCount;
	USHORT ModifiedCount;
	USHORT PageTableCount;
} SYSTEM_MEMORY_INFO, *PSYSTEM_MEMORY_INFO;

typedef struct _SYSTEM_MEMORY_INFORMATION 
{
	ULONG InfoSize;
	ULONG_PTR StringStart;
	SYSTEM_MEMORY_INFO Memory[1];
} SYSTEM_MEMORY_INFORMATION, *PSYSTEM_MEMORY_INFORMATION;

typedef struct _IMAGE_EXPORT_DIRECTORY 
{
	DWORD   Characteristics;
	DWORD   TimeDateStamp;
	WORD    MajorVersion;
	WORD    MinorVersion;
	DWORD   Name;
	DWORD   Base;
	DWORD   NumberOfFunctions;
	DWORD   NumberOfNames;
	DWORD   AddressOfFunctions;     // RVA from base of image
	DWORD   AddressOfNames;         // RVA from base of image
	DWORD   AddressOfNameOrdinals;  // RVA from base of image
} IMAGE_EXPORT_DIRECTORY, *PIMAGE_EXPORT_DIRECTORY;

typedef struct _SYSTEM_GDI_DRIVER_INFORMATION 
{
	UNICODE_STRING DriverName;
	PVOID ImageAddress;
	PVOID SectionPointer;
	PVOID EntryPoint;
	PIMAGE_EXPORT_DIRECTORY ExportSectionPointer;
	ULONG ImageLength;
} SYSTEM_GDI_DRIVER_INFORMATION, *PSYSTEM_GDI_DRIVER_INFORMATION;

typedef struct _SYSTEM_SET_TIME_ADJUST_INFORMATION
{
	ULONG TimeAdjustment;
	BOOLEAN Enable;
} SYSTEM_SET_TIME_ADJUST_INFORMATION, *PSYSTEM_SET_TIME_ADJUST_INFORMATION;

typedef struct _KSERVICE_TABLE_DESCRIPTOR 
{
	PULONG_PTR Base;
	PULONG Count;
	ULONG Limit;
	PUCHAR Number;
} KSERVICE_TABLE_DESCRIPTOR, *PKSERVICE_TABLE_DESCRIPTOR;

typedef struct _CLIENT_ID 
{
	HANDLE UniqueProcess;
	HANDLE UniqueThread;
} CLIENT_ID;
typedef CLIENT_ID *PCLIENT_ID;

typedef struct _SYSTEM_THREAD_INFORMATION 
{
	LARGE_INTEGER KernelTime;
	LARGE_INTEGER UserTime;
	LARGE_INTEGER CreateTime;
	ULONG WaitTime;
	PVOID StartAddress;
	CLIENT_ID ClientId;
	KPRIORITY Priority;
	LONG BasePriority;
	ULONG ContextSwitches;
	ULONG ThreadState;
	ULONG WaitReason;
} SYSTEM_THREAD_INFORMATION, *PSYSTEM_THREAD_INFORMATION;

typedef struct _SYSTEM_EXTENDED_THREAD_INFORMATION 
{
	SYSTEM_THREAD_INFORMATION ThreadInfo;
	PVOID StackBase;
	PVOID StackLimit;
	PVOID Win32StartAddress;
	ULONG_PTR Reserved1;
	ULONG_PTR Reserved2;
	ULONG_PTR Reserved3;
	ULONG_PTR Reserved4;
} SYSTEM_EXTENDED_THREAD_INFORMATION, *PSYSTEM_EXTENDED_THREAD_INFORMATION;

typedef struct _SYSTEM_EXCEPTION_INFORMATION
{
	ULONG AlignmentFixupCount;
	ULONG ExceptionDispatchCount;
	ULONG FloatingEmulationCount;
	ULONG ByteWordEmulationCount;
} SYSTEM_EXCEPTION_INFORMATION, *PSYSTEM_EXCEPTION_INFORMATION;

typedef struct _SYSTEM_KERNEL_DEBUGGER_INFORMATION
{
	BOOLEAN KernelDebuggerEnabled;
	BOOLEAN KernelDebuggerNotPresent;
} SYSTEM_KERNEL_DEBUGGER_INFORMATION, *PSYSTEM_KERNEL_DEBUGGER_INFORMATION;

typedef struct _SYSTEM_CONTEXT_SWITCH_INFORMATION 
{
	ULONG ContextSwitches;
	ULONG FindAny;
	ULONG FindLast;
	ULONG FindIdeal;
	ULONG IdleAny;
	ULONG IdleCurrent;
	ULONG IdleLast;
	ULONG IdleIdeal;
	ULONG PreemptAny;
	ULONG PreemptCurrent;
	ULONG PreemptLast;
	ULONG SwitchToIdle;
} SYSTEM_CONTEXT_SWITCH_INFORMATION, *PSYSTEM_CONTEXT_SWITCH_INFORMATION;

typedef struct _SYSTEM_REGISTRY_QUOTA_INFORMATION 
{
	ULONG  RegistryQuotaAllowed;
	ULONG  RegistryQuotaUsed;
	SIZE_T PagedPoolSize;
} SYSTEM_REGISTRY_QUOTA_INFORMATION, *PSYSTEM_REGISTRY_QUOTA_INFORMATION;

typedef struct _SYSTEM_PROCESSOR_IDLE_INFORMATION 
{
	ULONGLONG IdleTime;
	ULONGLONG C1Time;
	ULONGLONG C2Time;
	ULONGLONG C3Time;
	ULONG     C1Transitions;
	ULONG     C2Transitions;
	ULONG     C3Transitions;
	ULONG     Padding;
} SYSTEM_PROCESSOR_IDLE_INFORMATION, *PSYSTEM_PROCESSOR_IDLE_INFORMATION;

typedef struct _SYSTEM_LEGACY_DRIVER_INFORMATION 
{
	ULONG VetoType;
	UNICODE_STRING VetoList;
} SYSTEM_LEGACY_DRIVER_INFORMATION, *PSYSTEM_LEGACY_DRIVER_INFORMATION;

typedef short CSHORT;

typedef struct _TIME_FIELDS 
{
	CSHORT Year;        // range [1601...]
	CSHORT Month;       // range [1..12]
	CSHORT Day;         // range [1..31]
	CSHORT Hour;        // range [0..23]
	CSHORT Minute;      // range [0..59]
	CSHORT Second;      // range [0..59]
	CSHORT Milliseconds;// range [0..999]
	CSHORT Weekday;     // range [0..6] == [Sunday..Saturday]
} TIME_FIELDS;

typedef struct _RTL_TIME_ZONE_INFORMATION 
{
	LONG Bias;
	WCHAR StandardName[32];
	TIME_FIELDS StandardStart;
	LONG StandardBias;
	WCHAR DaylightName[32];
	TIME_FIELDS DaylightStart;
	LONG DaylightBias;
} RTL_TIME_ZONE_INFORMATION, *PRTL_TIME_ZONE_INFORMATION;

typedef struct _SYSTEM_LOOKASIDE_INFORMATION 
{
	USHORT CurrentDepth;
	USHORT MaximumDepth;
	ULONG TotalAllocates;
	ULONG AllocateMisses;
	ULONG TotalFrees;
	ULONG FreeMisses;
	ULONG Type;
	ULONG Tag;
	ULONG Size;
} SYSTEM_LOOKASIDE_INFORMATION, *PSYSTEM_LOOKASIDE_INFORMATION;

typedef struct _SYSTEM_VERIFIER_INFORMATION 
{
	ULONG NextEntryOffset;
	ULONG Level;
	UNICODE_STRING DriverName;
	ULONG RaiseIrqls;
	ULONG AcquireSpinLocks;
	ULONG SynchronizeExecutions;
	ULONG AllocationsAttempted;
	ULONG AllocationsSucceeded;
	ULONG AllocationsSucceededSpecialPool;
	ULONG AllocationsWithNoTag;
	ULONG TrimRequests;
	ULONG Trims;
	ULONG AllocationsFailed;
	ULONG AllocationsFailedDeliberately;
	ULONG Loads;
	ULONG Unloads;
	ULONG UnTrackedPool;
	ULONG CurrentPagedPoolAllocations;
	ULONG CurrentNonPagedPoolAllocations;
	ULONG PeakPagedPoolAllocations;
	ULONG PeakNonPagedPoolAllocations;
	SIZE_T PagedPoolUsageInBytes;
	SIZE_T NonPagedPoolUsageInBytes;
	SIZE_T PeakPagedPoolUsageInBytes;
	SIZE_T PeakNonPagedPoolUsageInBytes;
} SYSTEM_VERIFIER_INFORMATION, *PSYSTEM_VERIFIER_INFORMATION;

typedef struct _SYSTEM_SESSION_PROCESS_INFORMATION 
{
	ULONG SessionId;
	ULONG SizeOfBuf;
	PVOID Buffer;
} SYSTEM_SESSION_PROCESS_INFORMATION, *PSYSTEM_SESSION_PROCESS_INFORMATION;

typedef struct _SYSTEM_SESSION_POOLTAG_INFORMATION 
{
	SIZE_T NextEntryOffset;
	ULONG SessionId;
	ULONG Count;
	SYSTEM_POOLTAG TagInfo[1];
} SYSTEM_SESSION_POOLTAG_INFORMATION, *PSYSTEM_SESSION_POOLTAG_INFORMATION;

typedef struct _SYSTEM_NUMA_INFORMATION 
{
	ULONG       HighestNodeNumber;
	ULONG       Reserved;
	union 
	{
		ULONGLONG   ActiveProcessorsAffinityMask[MAXIMUM_NUMA_NODES];
		ULONGLONG   AvailableMemory[MAXIMUM_NUMA_NODES];
	};
} SYSTEM_NUMA_INFORMATION, *PSYSTEM_NUMA_INFORMATION;

typedef struct _SYSTEM_PROCESSOR_POWER_INFORMATION 
{
	UCHAR       CurrentFrequency;
	UCHAR       ThermalLimitFrequency;
	UCHAR       ConstantThrottleFrequency;
	UCHAR       DegradedThrottleFrequency;
	UCHAR       LastBusyFrequency;
	UCHAR       LastC3Frequency;
	UCHAR       LastAdjustedBusyFrequency;
	UCHAR       ProcessorMinThrottle;
	UCHAR       ProcessorMaxThrottle;
	ULONG       NumberOfFrequencies;
	ULONG       PromotionCount;
	ULONG       DemotionCount;
	ULONG       ErrorCount;
	ULONG       RetryCount;
	ULONGLONG   CurrentFrequencyTime;
	ULONGLONG   CurrentProcessorTime;
	ULONGLONG   CurrentProcessorIdleTime;
	ULONGLONG   LastProcessorTime;
	ULONGLONG   LastProcessorIdleTime;
} SYSTEM_PROCESSOR_POWER_INFORMATION, *PSYSTEM_PROCESSOR_POWER_INFORMATION;

typedef struct _SYSTEM_HANDLE_TABLE_ENTRY_INFO_EX 
{
	PVOID Object;
	ULONG_PTR UniqueProcessId;
	ULONG_PTR HandleValue;
	ULONG GrantedAccess;
	USHORT CreatorBackTraceIndex;
	USHORT ObjectTypeIndex;
	ULONG  HandleAttributes;
	ULONG  Reserved;
} SYSTEM_HANDLE_TABLE_ENTRY_INFO_EX, *PSYSTEM_HANDLE_TABLE_ENTRY_INFO_EX;

typedef struct _SYSTEM_HANDLE_INFORMATION_EX 
{
	ULONG_PTR NumberOfHandles;
	ULONG_PTR Reserved;
	SYSTEM_HANDLE_TABLE_ENTRY_INFO_EX Handles[1];
} SYSTEM_HANDLE_INFORMATION_EX, *PSYSTEM_HANDLE_INFORMATION_EX;

typedef struct _SYSTEM_BIGPOOL_ENTRY 
{
	union 
	{
		PVOID VirtualAddress;
		ULONG_PTR NonPaged : 1;     // Set to 1 if entry is nonpaged.
	};
	SIZE_T SizeInBytes;
	union 
	{
		UCHAR Tag[4];
		ULONG TagUlong;
	};
} SYSTEM_BIGPOOL_ENTRY, *PSYSTEM_BIGPOOL_ENTRY;

typedef struct _SYSTEM_BIGPOOL_INFORMATION 
{
	ULONG Count;
	SYSTEM_BIGPOOL_ENTRY AllocatedInfo[1];
} SYSTEM_BIGPOOL_INFORMATION, *PSYSTEM_BIGPOOL_INFORMATION;

typedef struct _SYSTEM_SESSION_MAPPED_VIEW_INFORMATION 
{
	SIZE_T NextEntryOffset;
	ULONG SessionId;
	ULONG ViewFailures;
	SIZE_T NumberOfBytesAvailable;
	SIZE_T NumberOfBytesAvailableContiguous;
} SYSTEM_SESSION_MAPPED_VIEW_INFORMATION, *PSYSTEM_SESSION_MAPPED_VIEW_INFORMATION;

typedef enum _WATCHDOG_HANDLER_ACTION 
{
	WdActionSetTimeoutValue,
	WdActionQueryTimeoutValue,
	WdActionResetTimer,
	WdActionStopTimer,
	WdActionStartTimer,
	WdActionSetTriggerAction,
	WdActionQueryTriggerAction,
	WdActionQueryState
} WATCHDOG_HANDLER_ACTION;

typedef enum _WATCHDOG_INFORMATION_CLASS 
{
	WdInfoTimeoutValue,
	WdInfoResetTimer,
	WdInfoStopTimer,
	WdInfoStartTimer,
	WdInfoTriggerAction,
	WdInfoState
} WATCHDOG_INFORMATION_CLASS;

typedef NTSTATUS (*PWD_HANDLER)(IN WATCHDOG_HANDLER_ACTION Action, IN PVOID Context, IN OUT PULONG DataValue, IN BOOLEAN NoLocks);

typedef struct _SYSTEM_WATCHDOG_HANDLER_INFORMATION 
{
	PWD_HANDLER WdHandler;
	PVOID       Context;
} SYSTEM_WATCHDOG_HANDLER_INFORMATION, *PSYSTEM_WATCHDOG_HANDLER_INFORMATION;

typedef struct _SYSTEM_WATCHDOG_TIMER_INFORMATION 
{
	WATCHDOG_INFORMATION_CLASS  WdInfoClass;
	ULONG                       DataValue;
} SYSTEM_WATCHDOG_TIMER_INFORMATION, *PSYSTEM_WATCHDOG_TIMER_INFORMATION;

typedef enum _LOGICAL_PROCESSOR_RELATIONSHIP 
{
	RelationProcessorCore,
	RelationNumaNode,
	RelationCache,
	RelationProcessorPackage,
	RelationGroup,
	RelationAll = 0xffff
} LOGICAL_PROCESSOR_RELATIONSHIP;

typedef enum _PROCESSOR_CACHE_TYPE 
{
	CacheUnified,
	CacheInstruction,
	CacheData,
	CacheTrace
} PROCESSOR_CACHE_TYPE;

typedef struct _CACHE_DESCRIPTOR 
{
	BYTE   Level;
	BYTE   Associativity;
	WORD   LineSize;
	DWORD  Size;
	PROCESSOR_CACHE_TYPE Type;
} CACHE_DESCRIPTOR, *PCACHE_DESCRIPTOR;

typedef struct _SYSTEM_LOGICAL_PROCESSOR_INFORMATION 
{
	ULONG_PTR   ProcessorMask;
	LOGICAL_PROCESSOR_RELATIONSHIP Relationship;
	union 
	{
		struct 
		{
			BYTE  Flags;
		} ProcessorCore;
		struct 
		{
			DWORD NodeNumber;
		} NumaNode;
		CACHE_DESCRIPTOR Cache;
		ULONGLONG  Reserved[2];
	} DUMMYUNIONNAME;
} SYSTEM_LOGICAL_PROCESSOR_INFORMATION, *PSYSTEM_LOGICAL_PROCESSOR_INFORMATION;

typedef enum _SYSTEM_FIRMWARE_TABLE_ACTION 
{
	SystemFirmwareTable_Enumerate,
	SystemFirmwareTable_Get
} SYSTEM_FIRMWARE_TABLE_ACTION;

#ifndef ANYSIZE_ARRAY
#define ANYSIZE_ARRAY 1       // winnt
#endif

typedef struct _SYSTEM_FIRMWARE_TABLE_INFORMATION 
{
	ULONG                           ProviderSignature;
	SYSTEM_FIRMWARE_TABLE_ACTION    Action;
	ULONG                           TableID;
	ULONG                           TableBufferLength;
	UCHAR                           TableBuffer[ANYSIZE_ARRAY];
} SYSTEM_FIRMWARE_TABLE_INFORMATION, *PSYSTEM_FIRMWARE_TABLE_INFORMATION;

typedef NTSTATUS(__cdecl *PFNFTH)(PSYSTEM_FIRMWARE_TABLE_INFORMATION);

typedef struct _SYSTEM_FIRMWARE_TABLE_HANDLER 
{
	ULONG       ProviderSignature;
	BOOLEAN     Register;
	PFNFTH      FirmwareTableHandler;
	PVOID       DriverObject;
} SYSTEM_FIRMWARE_TABLE_HANDLER, *PSYSTEM_FIRMWARE_TABLE_HANDLER;
```

## NtQuerySystemInforamtion.cpp

```C++
#include "NtDefs.h"
#include <iostream>
#include <string>

#pragma comment(lib,"ntdll.lib")
using namespace std;
#define printseg(x) cout<<"\t"###x##":"<<x<<endl

typedef HANDLE HLOCAL;

extern "C"
{
	KSERVICE_TABLE_DESCRIPTOR* KeServiceDescriptorTableShadow;
	NTSTATUS __stdcall NtQuerySystemInformation(IN SYSTEM_INFORMATION_CLASS SystemInformationClass, OUT PVOID SystemInformation, IN ULONG SystemInformationLength, OUT PULONG ReturnLength OPTIONAL);
	HLOCAL __stdcall LocalAlloc(IN UINT uFlags, SIZE_T uBytes);
	LPVOID __stdcall LocalLock(IN HLOCAL hMem);
	HLOCAL __stdcall LocalFree(IN HLOCAL hMem);
}

template<typename systeminfo>
void GetInformationTemplate(IN SYSTEM_INFORMATION_CLASS SystemInformationClass,IN void (*GetCallBack)(systeminfo* si))
{
	ULONG retlen = 2;
	NTSTATUS status = STATUS_SUCCESS;
	HLOCAL hMem = NULL;

	status = NtQuerySystemInformation(SystemInformationClass, &status, sizeof(status), &retlen);
	switch (status)
	{
	case STATUS_INVALID_INFO_CLASS:
		cout << "INVALID_INFO_CLASS" << endl;
		return;
	case STATUS_ACCESS_VIOLATION:
		cout << "ACCESS_VIOLATION" << endl;
		return;
	case STATUS_INSUFFICIENT_RESOURCES:
		cout << "INSUFFICIENT_RESOURCES" << endl;
		return;
	case STATUS_WORKING_SET_QUOTA:
		cout << "WORKING_SET_QUOTA" << endl;
		return;
	case STATUS_NOT_IMPLEMENTED:
		cout << "NOT_IMPLEMENTED" << endl;
		return;
	case STATUS_INFO_LENGTH_MISMATCH:
		do
		{
			hMem = LocalAlloc(0, retlen);
			if (hMem)
			{
				systeminfo* si = (systeminfo*)LocalLock(hMem);
				if (si)
				{
					memset(si, 0, retlen);
					status = NtQuerySystemInformation(SystemInformationClass, si, retlen, &retlen);
					if (NT_SUCCESS(status))
					{
						GetCallBack(si);
					}
				}
				LocalFree(hMem);
			}
		} while (status == STATUS_INFO_LENGTH_MISMATCH);
		return;
	case STATUS_SUCCESS:
		break;
	default:
		cout << "UNKNOWN ERROR" << endl;
		return;
	}
	
	if (retlen < sizeof(systeminfo))
		retlen = sizeof(systeminfo);
	cout << "structsize:" << sizeof(systeminfo) << " realsize:" << retlen << endl;
	hMem = LocalAlloc(0, retlen);
	if (hMem)
	{
		systeminfo* si = (systeminfo*)LocalLock(hMem);
		if (si)
		{
			memset(si, 0, retlen);
			status = NtQuerySystemInformation(SystemInformationClass, si, retlen, &retlen);
			if (NT_SUCCESS(status))
			{
				GetCallBack(si);
			}
			else
			{
				cout << "ERROR" << endl;
			}
		}
		LocalFree(hMem);
	}
}

void GetSystemBasicInformation(SYSTEM_BASIC_INFORMATION* psbi)//0
{
	cout << "\t\t0 SystemBasicInformation" << endl;
	cout << "\t时间解析度ms:" << psbi->TimerResolution << endl;
	cout << "\t物理页大小:" << psbi->PageSize << endl;
	cout << "\t物理页个数:" << psbi->NumberOfPhysicalPages << endl;
	cout << "\t最小物理页个数:" << psbi->LowestPhysicalPageNumber << endl;
	cout << "\t最大物理页个数:" << psbi->HighestPhysicalPageNumber << endl;
	cout << "\t逻辑页大小:" << psbi->AllocationGranularity << endl;
	cout << "\t最小用户地址:" << psbi->MinimumUserModeAddress << endl;
	cout << "\t最大用户地址:" << psbi->MaximumUserModeAddress << endl;
	cout << "\t处理器个数:" << (int)psbi->NumberOfProcessors << endl;
}

void GetSystemProcessorInformation(SYSTEM_PROCESSOR_INFORMATION* pspri)//1
{	//		1 SystemProcessorInformation
	cout << "\t\t1 SystemProcessorInformation" << endl;
	switch (pspri->ProcessorArchitecture)
	{
	case PROCESSOR_ARCHITECTURE_INTEL:
		cout << "\tINTEL ";
		if (pspri->ProcessorLevel == 3)
			cout << "386 ";
		else if (pspri->ProcessorLevel == 4)
			cout << "486 ";
		else if (pspri->ProcessorLevel == 5)
			cout << "586 or Pentium ";
		break;
	case PROCESSOR_ARCHITECTURE_IA64:
		cout << "\tIA64 ";
		if (pspri->ProcessorLevel == 7)
			cout << "Itanium ";
		else if (pspri->ProcessorLevel == 31)
			cout << "Itanium 2 ";
		break;
	}
	cout << pspri->ProcessorRevision << " " << pspri->ProcessorFeatureBits<<endl;
}

void GetSystemPerformanceInformation(SYSTEM_PERFORMANCE_INFORMATION* pspei)//2
{
	cout << "\t\t2 SystemPerformanceInformation" << endl;
	cout << "\tIdle进程时间:" << pspei->IdleProcessTime.QuadPart << endl;
	cout << "\tIO读字节" << pspei->IoReadTransferCount.QuadPart << endl;
	cout << "\tIO写字节" << pspei->IoWriteTransferCount.QuadPart << endl;
	cout << "\tIO其他字节" << pspei->IoOtherTransferCount.QuadPart <<endl;
	cout << "\tIO读次数" << pspei->IoReadOperationCount << endl;
	cout << "\tIO写次数" << pspei->IoWriteOperationCount << endl;
	cout << "\tIO其他次数" << pspei->IoOtherOperationCount << endl;
	cout << "\t未用页" << pspei->AvailablePages << endl;
	cout << "\t已用页" << pspei->CommittedPages << endl;
	cout << "\t最多使用页" << pspei->CommitLimit << endl;
	cout << "\t已用页峰值" << pspei->PeakCommitment << endl;
	cout << "\t页错误数" << pspei->PageFaultCount << endl;
	cout << "\tCopyOnWrite数" << pspei->CopyOnWriteCount << endl;
	printseg(pspei->TransitionCount);
	printseg(pspei->CacheTransitionCount);
	printseg(pspei->DemandZeroCount);
	printseg(pspei->PageReadCount);
	printseg(pspei->PageReadIoCount);
	printseg(pspei->CacheReadCount);
	printseg(pspei->CacheIoCount);
	printseg(pspei->DirtyPagesWriteCount);
	printseg(pspei->DirtyWriteIoCount);
	printseg(pspei->MappedPagesWriteCount);
	printseg(pspei->MappedWriteIoCount);
	printseg(pspei->PagedPoolPages);
	printseg(pspei->NonPagedPoolPages);
	printseg(pspei->PagedPoolAllocs);
	printseg(pspei->PagedPoolFrees);
	printseg(pspei->NonPagedPoolAllocs);
	printseg(pspei->NonPagedPoolFrees);
	printseg(pspei->FreeSystemPtes);
	printseg(pspei->ResidentSystemCodePage);
	printseg(pspei->TotalSystemDriverPages);
	printseg(pspei->TotalSystemCodePages);
	printseg(pspei->NonPagedPoolLookasideHits);
	printseg(pspei->PagedPoolLookasideHits);
	printseg(pspei->AvailablePagedPoolPages);
	printseg(pspei->ResidentSystemCachePage);
	printseg(pspei->ResidentPagedPoolPage);
	printseg(pspei->ResidentSystemDriverPage);
	printseg(pspei->CcFastReadNoWait);
	printseg(pspei->CcFastReadWait);
	printseg(pspei->CcFastReadResourceMiss);
	printseg(pspei->CcFastReadNotPossible);
	printseg(pspei->CcFastMdlReadNoWait);
	printseg(pspei->CcFastMdlReadWait);
	printseg(pspei->CcFastMdlReadResourceMiss);
	printseg(pspei->CcFastMdlReadNotPossible);
	printseg(pspei->CcMapDataNoWait);
	printseg(pspei->CcMapDataWait);
	printseg(pspei->CcMapDataNoWaitMiss);
	printseg(pspei->CcMapDataWaitMiss);
	printseg(pspei->CcPinMappedDataCount);
	printseg(pspei->CcPinReadNoWait);
	printseg(pspei->CcPinReadWait);
	printseg(pspei->CcPinReadNoWaitMiss);
	printseg(pspei->CcPinReadWaitMiss); 
	printseg(pspei->CcCopyReadNoWait);
	printseg(pspei->CcCopyReadWait);
	printseg(pspei->CcCopyReadNoWaitMiss);
	printseg(pspei->CcCopyReadWaitMiss);
	printseg(pspei->CcMdlReadNoWait);
	printseg(pspei->CcMdlReadWait);
	printseg(pspei->CcMdlReadNoWaitMiss);
	printseg(pspei->CcMdlReadWaitMiss);
	printseg(pspei->CcReadAheadIos);
	printseg(pspei->CcLazyWriteIos);
	printseg(pspei->CcLazyWritePages);
	printseg(pspei->CcDataFlushes);
	printseg(pspei->CcDataPages);
	printseg(pspei->ContextSwitches);
	printseg(pspei->FirstLevelTbFills);
	printseg(pspei->SecondLevelTbFills);
	printseg(pspei->SystemCalls);
}

void GetSystemTimeOfDayInformation(SYSTEM_TIMEOFDAY_INFORMATION* psti)//3
{
	cout << "\t\t3 SystemTimeOfDayInformation" << endl;
	cout << "\t启动时间:" << psti->BootTime.QuadPart << endl;
	cout << "\t当前时间:" << psti->CurrentTime.QuadPart << endl;
	printseg(psti->TimeZoneBias.QuadPart);
	printseg(psti->TimeZoneId);
	printseg(psti->BootTimeBias);
	printseg(psti->SleepTimeBias);
}

// void GetSystemPathInformation(SYSTEM_PATH_INFORMATION* pspi)//4
// {
// }

void GetSystemProcessInformation(SYSTEM_PROCESS_INFORMATION* pspri1)//5
{
	cout << "\t\t5 SystemProcessInformation" << endl;
	do
	{
		if (pspri1->ImageName.Buffer)
			wcout << "\tImageName:" << wstring((wchar_t*)pspri1->ImageName.Buffer) << endl;
		else
			wcout << "\tno name" << endl;
		cout << "\t线程数:" << pspri1->NumberOfThreads << endl;
		printseg(pspri1->SpareLi1.QuadPart);
		printseg(pspri1->SpareLi2.QuadPart);
		printseg(pspri1->SpareLi3.QuadPart);
		cout << "\t创建时间:" << pspri1->CreateTime.QuadPart << endl;
		cout << "\t用户态时间:" << pspri1->UserTime.QuadPart << endl;
		cout << "\t内核态时间:" << pspri1->KernelTime.QuadPart << endl;
		cout << "\t基础优先级:" << pspri1->BasePriority << endl;
		cout << "\t进程Id:" << (int)pspri1->UniqueProcessId << endl;
		cout << "\t父进程Id:" << (int)pspri1->InheritedFromUniqueProcessId << endl;
		cout << "\t句柄数:" << pspri1->HandleCount << endl;
		cout << "\t会话Id:" << pspri1->SessionId << endl;
		cout << "\t页目录机制:" << pspri1->PageDirectoryBase << endl;
		cout << "\t虚拟内存峰值:" << pspri1->PeakVirtualSize << endl;
		cout << "\t虚拟内存大小:" << pspri1->VirtualSize << endl;
		cout << "\t页错误数:" << pspri1->PageFaultCount << endl;
		cout << "\t物理内存峰值:" << pspri1->PeakWorkingSetSize << endl;
		cout << "\t物理内存大小:" << pspri1->WorkingSetSize << endl;
		cout << "\t分页池配额峰值:" << pspri1->QuotaPeakPagedPoolUsage << endl;
		cout << "\t分页池配额:" << pspri1->QuotaPagedPoolUsage << endl;
		cout << "\t非分页池配额峰值:" << pspri1->QuotaPeakNonPagedPoolUsage << endl;
		cout << "\t非分页池配额:" << pspri1->QuotaNonPagedPoolUsage << endl;
		cout << "\t页面文件使用:" << pspri1->PagefileUsage << endl;
		cout << "\t页面文件使用峰值:" << pspri1->PeakPagefileUsage << endl;
		cout << "\t私有页面数:" << pspri1->PrivatePageCount << endl;
		cout << "\t读操作数:" << pspri1->ReadOperationCount.QuadPart << endl;
		cout << "\t写操作数:" << pspri1->WriteOperationCount.QuadPart << endl;
		cout << "\t其他操作数:" << pspri1->OtherOperationCount.QuadPart << endl;
		cout << "\t读字节数:" << pspri1->ReadTransferCount.QuadPart << endl;
		cout << "\t写字节数:" << pspri1->WriteTransferCount.QuadPart << endl;
		cout << "\t其他字节数:" << pspri1->OtherTransferCount.QuadPart << endl;
		SYSTEM_PROCESS_INFORMATION* newpspri1 = (SYSTEM_PROCESS_INFORMATION*)((BYTE*)pspri1 + pspri1->NextEntryOffset);
		SYSTEM_THREAD_INFORMATION* pesti = (SYSTEM_THREAD_INFORMATION*)(pspri1 + 1);
		int threadindex = 0;
		while ((LPVOID)pesti < (LPVOID)newpspri1)
		{
			++threadindex;
			cout << "\t内核态时间:" << pesti->KernelTime.QuadPart << endl;
			cout << "\t用户态时间:" << pesti->UserTime.QuadPart << endl;
			cout << "\t创建时间:" << pesti->CreateTime.QuadPart << endl;
			cout << "\t等待时间:" << pesti->WaitTime << endl;
			cout << "\t起始地址:" << hex << pesti->StartAddress << endl;
			cout << "\tUniqueProcess:" << hex << pesti->ClientId.UniqueProcess << endl;
			cout << "\tUniqueThread:" << hex << pesti->ClientId.UniqueThread << endl;
			cout << dec;
			cout << "\t优先级:" << pesti->Priority << endl;
			cout << "\t基础优先级:" << pesti->BasePriority << endl;
			cout << "\t模式切换次数:" << pesti->ContextSwitches << endl;
			cout << "\t线程状态:" << pesti->ThreadState << endl;
			cout << "\t等待原因:" << pesti->WaitReason << endl;
			cout << dec;
			pesti++;
		}
		pspri1 = newpspri1;
	} while (pspri1->NextEntryOffset);
}

void GetSystemCallCountInformation(SYSTEM_CALL_COUNT_INFORMATION* pscci)//6
{
	cout << "\t\t6 SystemCallCountInformation" << endl;
	printseg(pscci->Length);
	printseg(pscci->NumberOfTables);
	ULONG* limits = (ULONG*)((BYTE*)pscci + sizeof(SYSTEM_CALL_COUNT_INFORMATION));
	ULONG* tables = (ULONG*)((BYTE*)pscci + sizeof(SYSTEM_CALL_COUNT_INFORMATION)+pscci->NumberOfTables*sizeof(PULONG));
	int index = 0;
	for (int i = 0; i < pscci->NumberOfTables; i++)
	{
		for (int j = 0; j < limits[i]; j++)
		{
			cout << "tablecount" << index << ":" << tables[index] << endl;
			++index;
		}
	}
}

void GetSystemDeviceInformation(SYSTEM_DEVICE_INFORMATION* psdi)//7
{
	cout << "\t\t7 SystemDeviceInformation" << endl;
	cout << "\t磁盘数:" << psdi->NumberOfDisks << endl;
	cout << "\t软盘数:" << psdi->NumberOfFloppies << endl;
	cout << "\t光驱数:" << psdi->NumberOfCdRoms << endl;
	cout << "\t磁带数:" << psdi->NumberOfTapes << endl;
	cout << "\t串行端口数:" << psdi->NumberOfSerialPorts << endl;
	cout << "\t并行端口数:" << psdi->NumberOfParallelPorts << endl;
}

void GetSystemProcessorPerformanceInformation(SYSTEM_PROCESSOR_PERFORMANCE_INFORMATION* psppei)//8
{
	cout << "\t\t8 SystemProcessorPerformanceInformation" << endl;
	cout << "\t空闲时间:" << psppei->IdleTime.QuadPart << endl;
	cout << "\t内核态时间:" << psppei->KernelTime.QuadPart << endl;
	cout << "\t用户态时间:" << psppei->UserTime.QuadPart << endl;
	cout << "\tDPC时间:" << psppei->DpcTime.QuadPart << endl;
	cout << "\t中断时间:" << psppei->InterruptTime.QuadPart << endl;
	cout << "\t中断次数:" << psppei->InterruptCount << endl;
}

void GetSystemFlagsInformation(SYSTEM_FLAGS_INFORMATION* psfi)//9
{
		cout << "\t\t9 SystemFlagsInformation" << endl;
		if (psfi->Flags&FLG_STOP_ON_EXCEPTION)
			cout << "FLG_STOP_ON_EXCEPTION" << endl;
		if (psfi->Flags&FLG_SHOW_LDR_SNAPS)
			cout << "FLG_SHOW_LDR_SNAPS" << endl;
		if (psfi->Flags&FLG_DEBUG_INITIAL_COMMAND)
			cout << "FLG_DEBUG_INITIAL_COMMAND" << endl;
		if (psfi->Flags&FLG_STOP_ON_HUNG_GUI)
			cout << "FLG_STOP_ON_HUNG_GUI" << endl;

		if (psfi->Flags&FLG_HEAP_ENABLE_TAIL_CHECK)
			cout << "FLG_HEAP_ENABLE_TAIL_CHECK" << endl;
		if (psfi->Flags&FLG_HEAP_ENABLE_FREE_CHECK)
			cout << "FLG_HEAP_ENABLE_FREE_CHECK" << endl;
		if (psfi->Flags&FLG_HEAP_VALIDATE_PARAMETERS)
			cout << "FLG_HEAP_VALIDATE_PARAMETERS" << endl;
		if (psfi->Flags&FLG_HEAP_VALIDATE_ALL)
			cout << "FLG_HEAP_VALIDATE_ALL" << endl;

		if (psfi->Flags&FLG_APPLICATION_VERIFIER)
			cout << "FLG_APPLICATION_VERIFIER" << endl;
		if (psfi->Flags&FLG_POOL_ENABLE_TAGGING)
			cout << "FLG_POOL_ENABLE_TAGGING" << endl;
		if (psfi->Flags&FLG_HEAP_ENABLE_TAGGING)
			cout << "FLG_HEAP_ENABLE_TAGGING" << endl;

		if (psfi->Flags&FLG_USER_STACK_TRACE_DB)
			cout << "FLG_USER_STACK_TRACE_DB" << endl;
		if (psfi->Flags&FLG_KERNEL_STACK_TRACE_DB)
			cout << "FLG_KERNEL_STACK_TRACE_DB" << endl;
		if (psfi->Flags&FLG_MAINTAIN_OBJECT_TYPELIST)
			cout << "FLG_MAINTAIN_OBJECT_TYPELIST" << endl;
		if (psfi->Flags&FLG_HEAP_ENABLE_TAG_BY_DLL)
			cout << "FLG_HEAP_ENABLE_TAG_BY_DLL" << endl;

		if (psfi->Flags&FLG_DISABLE_STACK_EXTENSION)
			cout << "FLG_DISABLE_STACK_EXTENSION" << endl;
		if (psfi->Flags&FLG_ENABLE_CSRDEBUG)
			cout << "FLG_ENABLE_CSRDEBUG" << endl;
		if (psfi->Flags&FLG_ENABLE_KDEBUG_SYMBOL_LOAD)
			cout << "FLG_ENABLE_KDEBUG_SYMBOL_LOAD" << endl;
		if (psfi->Flags&FLG_DISABLE_PAGE_KERNEL_STACKS)
			cout << "FLG_DISABLE_PAGE_KERNEL_STACKS" << endl;

		if (psfi->Flags&FLG_ENABLE_SYSTEM_CRIT_BREAKS)
			cout << "FLG_ENABLE_SYSTEM_CRIT_BREAKS" << endl;
		if (psfi->Flags&FLG_HEAP_DISABLE_COALESCING)
			cout << "FLG_HEAP_DISABLE_COALESCING" << endl;
		if (psfi->Flags&FLG_ENABLE_CLOSE_EXCEPTIONS)
			cout << "FLG_ENABLE_CLOSE_EXCEPTIONS" << endl;
		if (psfi->Flags&FLG_ENABLE_EXCEPTION_LOGGING)
			cout << "FLG_ENABLE_EXCEPTION_LOGGING" << endl;

		if (psfi->Flags&FLG_ENABLE_HANDLE_TYPE_TAGGING)
			cout << "FLG_ENABLE_HANDLE_TYPE_TAGGING" << endl;
		if (psfi->Flags&FLG_HEAP_PAGE_ALLOCS)
			cout << "FLG_HEAP_PAGE_ALLOCS" << endl;
		if (psfi->Flags&FLG_DEBUG_INITIAL_COMMAND_EX)
			cout << "FLG_DEBUG_INITIAL_COMMAND_EX" << endl;
		if (psfi->Flags&FLG_DISABLE_DBGPRINT)
			cout << "FLG_DISABLE_DBGPRINT" << endl;

		if (psfi->Flags&FLG_CRITSEC_EVENT_CREATION)
			cout << "FLG_CRITSEC_EVENT_CREATION" << endl;
		if (psfi->Flags&FLG_LDR_TOP_DOWN)
			cout << "FLG_LDR_TOP_DOWN" << endl;
		if (psfi->Flags&FLG_ENABLE_HANDLE_EXCEPTIONS)
			cout << "FLG_ENABLE_HANDLE_EXCEPTIONS" << endl;
		if (psfi->Flags&FLG_DISABLE_PROTDLLS)
			cout << "FLG_DISABLE_PROTDLLS" << endl;
}

// void GetSystemCallTimeInformation(SYSTEM_CALLTIME_INFORMATION* pscti)
// {
// }

void GetSystemModuleInformation(RTL_PROCESS_MODULES* prpm)//11
{
	cout << "\t\t11 SystemModuleInformation" << endl;
	for (int i = 0; i < prpm->NumberOfModules; i++)
	{
		cout << "module" << i << " FullPathName:" << (char*)prpm->Modules[i].FullPathName << endl;
		cout << "\tSection:" << hex << prpm->Modules[i].Section << endl;
		cout << "\tMappedBase:" << hex << prpm->Modules[i].MappedBase << endl;
		cout << "\tImageBase:" << hex << prpm->Modules[i].ImageBase << endl;
		cout << "\tImageSize:" << hex << prpm->Modules[i].ImageSize << endl;
		cout << "\tFlags:" << hex << prpm->Modules[i].Flags << endl;
		cout << dec;
		cout << "\tLoadOrderIndex:" << (int)prpm->Modules[i].LoadOrderIndex << endl;
		cout << "\tInitOrderIndex:" << (int)prpm->Modules[i].InitOrderIndex << endl;
		cout << "\tLoadCount:" << (int)prpm->Modules[i].LoadCount << endl;
		cout << "\tOffsetToFile:" << prpm->Modules[i].OffsetToFileName << endl;//距离文件名偏移(取最后一个\之后的部分)	
	}
}

void GetSystemLocksInformation(RTL_PROCESS_LOCKS* prpl)//12
{
	cout << "\t\t12 SystemLocksInformation" << endl;
	for (int i = 0; i < prpl->NumberOfLocks; i++)
	{
		cout << "\tlock" << i << ":" << endl;
		cout << "\tAddress:" << hex << prpl->Locks[i].Address << endl;
		cout << "\tOwningThread:" << hex << prpl->Locks[i].OwningThread << endl;
		cout << dec;
		cout << "\tType:" << (int)prpl->Locks[i].Type << endl;
		cout << "\tCreatorBackTraceIndex:" << (int)prpl->Locks[i].CreatorBackTraceIndex << endl;
		cout << "\tLockCount:" << prpl->Locks[i].LockCount << endl;
		cout << "\tContentionCount:" << prpl->Locks[i].ContentionCount << endl;
		cout << "\tEntryCount:" << prpl->Locks[i].EntryCount << endl;
		cout << "\tRecursionCount:" << prpl->Locks[i].RecursionCount << endl;
		cout << "\tNumberOfWaitingShared:" << prpl->Locks[i].NumberOfWaitingShared << endl;
		cout << "\tNumberOfWaitingExclusive:" << prpl->Locks[i].NumberOfWaitingExclusive << endl;
	}
}

void GetSystemStackTraceInformation(RTL_PROCESS_BACKTRACES* prpb)//13
{
	cout << "\t\t13 SystemStackTraceInformation" << endl;
	cout << "\tCommittedMemory:" << prpb->CommittedMemory << endl;
	cout << "\tReservedMemory:" << prpb->ReservedMemory << endl;
	cout << "\tNumberOfBackTraceLookups:" << prpb->NumberOfBackTraceLookups << endl;
	for (int i = 0; i < prpb->NumberOfBackTraces; i++)
	{
		cout << "\tTraceCount:" << prpb->BackTraces[i].TraceCount << endl;
		cout << "\tIndex:" << prpb->BackTraces[i].Index << endl;
		cout << "\tDepth:" << prpb->BackTraces[i].Depth << endl;
		for (int j = 0; j < MAX_STACK_DEPTH; j++)
		{
			cout << "\t" << hex << prpb->BackTraces[i].BackTrace[i] << endl;
		}
	}
}

// void GetSystemPagedPoolInformation(SYSTEM_POOL_INFORMATION* pspi)//14
// {
// }

// void GetSystemNonPagedPoolInformation(SYSTEM_POOL_INFORMATION* pspi)//15
// {
// }

void GetSystemHandleInformation(SYSTEM_HANDLE_INFORMATION* pshi)//16
{
	cout << "\t\t16 SystemHandleInformation" << endl;
	for (int i = 0; i < pshi->NumberOfHandles; i++)
	{
		cout << "\t" << i + 1 << endl;
		cout << "\tUniqueProcessId:" << (int)pshi->Handles[i].UniqueProcessId << endl;
		cout << "\tCreatorBackTraceIndex:" << (int)pshi->Handles[i].CreatorBackTraceIndex << endl;
		cout << "\tObjectTypeIndex:" << (int)pshi->Handles[i].ObjectTypeIndex << endl;
		cout << "\tHandleAttributes:" << (int)pshi->Handles[i].HandleAttributes << endl;
		cout << "\tHandleValue:" << (int)pshi->Handles[i].HandleValue << endl;
		cout << "\tObject:" << hex << (int)pshi->Handles[i].Object << endl;
		cout << "\tGrantedAccess:" << hex << (int)pshi->Handles[i].GrantedAccess << endl;
	}
}

void GetSystemObjectInformation(SYSTEM_OBJECTTYPE_INFORMATION* pstoi)//17
{
	cout << "\t\t17 SystemObjectInformation" << endl;
	int nextpos = pstoi->NextEntryOffset;
	SYSTEM_OBJECT_INFORMATION* curpsoi = NULL;
	do
	{
		cout << "\tNumberOfObjects:" << pstoi->NumberOfObjects << endl;
		cout << "\tNumberOfHandles:" << pstoi->NumberOfHandles << endl;
		cout << "\tTypeIndex:" << pstoi->TypeIndex << endl;
		cout << "\tInvalidAttributes:" << hex << pstoi->InvalidAttributes << endl;
		cout << "\tValidAccessMask:" << hex << pstoi->ValidAccessMask << endl;
		cout << dec;
		cout << "\tPoolType:" << pstoi->PoolType << endl;
		cout << "\tSecurityRequired:" << pstoi->SecurityRequired << endl;
		cout << "\tWaitableObject:" << pstoi->WaitableObject << endl;
		wcout << "\tTypeName:" << (wchar_t*)pstoi->TypeName.Buffer << endl;
		curpsoi = (SYSTEM_OBJECT_INFORMATION*)((PWSTR)(pstoi + 1) + pstoi->TypeName.MaximumLength);
		cout << "\tObject:" << hex << curpsoi->Object << endl;
		cout << "\tCreatorUniqueProcess:" << hex << curpsoi->CreatorUniqueProcess << endl;
		cout << "\tCreatorBackTraceIndex:" << hex << curpsoi->CreatorBackTraceIndex << endl;
		cout << "\tFlags" << hex << curpsoi->Flags << endl;
		cout << "\tExclusiveProcessId:" << hex << curpsoi->ExclusiveProcessId << endl;
		cout << "\tSecurityDescriptor:" << hex << curpsoi->SecurityDescriptor << endl;
		cout << dec;
		cout << "\tPointerCount:" << curpsoi->PointerCount << endl;
		cout << "\tHandleCount:" << curpsoi->HandleCount << endl;
		cout << "\tPagedPoolCharge:" << curpsoi->PagedPoolCharge << endl;
		cout << "\tNonPagedPoolCharge:" << curpsoi->NonPagedPoolCharge << endl;
		wcout << "\tNameInfo:" << (wchar_t*)curpsoi->NameInfo.Name.Buffer << endl;
		pstoi = (SYSTEM_OBJECTTYPE_INFORMATION*)((BYTE*)pstoi + pstoi->NextEntryOffset);
	} while (nextpos);
}

void GetSystemPageFileInformation(SYSTEM_PAGEFILE_INFORMATION* pspi)//18
{
	cout << "\t\t18 SystemPageFileInformation" << endl;
	ULONG nextoffset = 0;
	int index = 0;
	do
	{
		cout << "PAGEFILE" << index + 1 << ":" << endl;
		nextoffset = pspi->NextEntryOffset;
		cout << "\tTotalSize:" << pspi->TotalSize << endl;
		cout << "\tTotalInUse:" << pspi->TotalInUse << endl;
		cout << "\tPeakUsage:" << pspi->PeakUsage << endl;
		cout << "\tPageFileName:" << (wchar_t*)pspi->PageFileName.Buffer << endl;
		pspi = (SYSTEM_PAGEFILE_INFORMATION*)((BYTE*)pspi + pspi->NextEntryOffset);
	} while (nextoffset);
}

void GetSystemVdmInstemulInformation(SYSTEM_VDM_INSTEMUL_INFO* psvii)//19
{
	cout << "\t\t19 SystemVdmInstemulInformation" << endl;
	printseg(psvii->SegmentNotPresent);
	printseg(psvii->VdmOpcode0F);
	printseg(psvii->OpcodeESPrefix);
	printseg(psvii->OpcodeCSPrefix);
	printseg(psvii->OpcodeSSPrefix);
	printseg(psvii->OpcodeDSPrefix);
	printseg(psvii->OpcodeFSPrefix);
	printseg(psvii->OpcodeGSPrefix);
	printseg(psvii->OpcodeOPER32Prefix);
	printseg(psvii->OpcodeADDR32Prefix);
	printseg(psvii->OpcodeINSB);
	printseg(psvii->OpcodeINSW);
	printseg(psvii->OpcodeOUTSB);
	printseg(psvii->OpcodeOUTSW);
	printseg(psvii->OpcodePUSHF);
	printseg(psvii->OpcodePOPF);
	printseg(psvii->OpcodeINTnn);
	printseg(psvii->OpcodeINTO);
	printseg(psvii->OpcodeIRET);
	printseg(psvii->OpcodeINBimm);
	printseg(psvii->OpcodeINWimm);
	printseg(psvii->OpcodeOUTBimm);
	printseg(psvii->OpcodeOUTWimm);
	printseg(psvii->OpcodeINB);
	printseg(psvii->OpcodeINW);
	printseg(psvii->OpcodeOUTB);
	printseg(psvii->OpcodeOUTW);
	printseg(psvii->OpcodeLOCKPrefix);
	printseg(psvii->OpcodeREPPrefix);
	printseg(psvii->OpcodeREPPrefix);
	printseg(psvii->OpcodeHLT);
	printseg(psvii->OpcodeCLI);
	printseg(psvii->OpcodeSTI);
	printseg(psvii->BopCount);
}

// void GetSystemVdmBopInformation(SYSTEM_VDM_BOP_INFO* psvbi)//20
// {
// 
// }

void GetSystemFileCacheInformation(SYSTEM_FILECACHE_INFORMATION* psfci)//21
{
	cout << "\t\t21 SystemFileCacheInformation" << endl;
	printseg(psfci->CurrentSize);
	printseg(psfci->PeakSize);
	printseg(psfci->PageFaultCount);
	printseg(psfci->MinimumWorkingSet);
	printseg(psfci->MaximumWorkingSet);
	printseg(psfci->CurrentSizeIncludingTransitionInPages);
	printseg(psfci->PeakSizeIncludingTransitionInPages);
	printseg(psfci->TransitionRePurposeCount);
	printseg(psfci->Flags);
}

void GetSystemPoolTagInformation(SYSTEM_POOLTAG_INFORMATION* pspti)//22
{
	cout << "\t\t22 SystemPoolTagInformation" << endl;
	for (int i = 0; i < pspti->Count; i++)
	{
		cout << pspti->TagInfo[i].TagUlong << endl;
		cout << pspti->TagInfo[i].PagedAllocs << endl;
		cout << pspti->TagInfo[i].PagedFrees << endl;
		cout << pspti->TagInfo[i].PagedUsed << endl;
		cout << pspti->TagInfo[i].NonPagedAllocs << endl;
		cout << pspti->TagInfo[i].NonPagedFrees << endl;
		cout << pspti->TagInfo[i].NonPagedUsed << endl;
	}
}

void GetSystemInterruptInformation(SYSTEM_INTERRUPT_INFORMATION* psii)//23
{
	cout << "\t\t23 SystemInterruptInformation" << endl;
	printseg(psii->ContextSwitches);
	printseg(psii->DpcCount);
	printseg(psii->DpcRate);
	printseg(psii->TimeIncrement);
	printseg(psii->DpcBypassCount);
	printseg(psii->ApcBypassCount);
}

void GetSystemDpcBehaviorInformation(SYSTEM_DPC_BEHAVIOR_INFORMATION* psdbi)//24
{
	cout << "\t\t24 SystemDpcBehaviorInformation" << endl;
	printseg(psdbi->Spare);
	printseg(psdbi->DpcQueueDepth);
	printseg(psdbi->MinimumDpcRate);
	printseg(psdbi->AdjustDpcThreshold);
	printseg(psdbi->IdealDpcRate);
}

void GetSystemFullMemoryInformation(SYSTEM_MEMORY_INFORMATION* psmi)//25
{
	cout << "\t\t25 SystemFullMemoryInformation" << endl;
	printseg(psmi->StringStart);
	for (int i = 0; i < psmi->InfoSize; i++)
	{
		cout << (char*)psmi->Memory[i].StringOffset << endl;
		printseg((int)psmi->Memory[i].ValidCount);
		printseg((int)psmi->Memory[i].TransitionCount);
		printseg((int)psmi->Memory[i].ModifiedCount);
		printseg((int)psmi->Memory[i].PageTableCount);
	}
}

void GetSystemLoadGdiDriverInformation(SYSTEM_GDI_DRIVER_INFORMATION* psgdi)//26
{
	cout << "\t\t26 SystemLoadGdiDriverInformation" << endl;
	wcout << psgdi->DriverName.Buffer << endl;
	cout << "\tImageLength:" << psgdi->ImageLength << endl;
	cout << "\tImageAddress:" << hex << psgdi->ImageAddress << endl;
	cout << "\tSectionPointer:" << hex << psgdi->SectionPointer << endl;
	cout << "\tEntryPoint:" << hex << psgdi->EntryPoint << endl;
}

// void GetSystemUnloadGdiDriverInformation(SYSTEM_UNLOAD_GDIDRIVER_INFORMATION)//27
// {
// 
// }

void GetSystemTimeAdjustmentInformation(SYSTEM_SET_TIME_ADJUST_INFORMATION* psstai)//28
{
	cout << "\t\t28 SystemTimeAdjustmentInformation" << endl;
	cout << "\tTimeAdjustment:" << psstai->TimeAdjustment << endl;
	cout << "\tEnable:" << psstai->Enable << endl;
}

void GetSystemSummaryMemoryInformation(SYSTEM_MEMORY_INFORMATION* psmi)//29
{
	cout << "\t\t29 SystemSummaryMemoryInformation" << endl;
	printseg(psmi->StringStart);
	for (int i = 0; i < psmi->InfoSize; i++)
	{
		cout << (char*)psmi->Memory[i].StringOffset << endl;
		printseg((int)psmi->Memory[i].ValidCount);
		printseg((int)psmi->Memory[i].TransitionCount);
		printseg((int)psmi->Memory[i].ModifiedCount);
		printseg((int)psmi->Memory[i].PageTableCount);
	}
}

// void GetSystemMirrorMemoryInformation(SYSTEM_MEMORY_INFORMATION* psmi)//30
// {
// }

// void GetSystemPerformanceTraceInformation(SYSTEM_PERFORMANCE_TRANCE_INFORMATION* pspti)//31
// {
// }

void GetSystemExceptionInformation(SYSTEM_EXCEPTION_INFORMATION* psei)//32
{
	cout << "\t\t32 SystemExceptionInformation" << endl;
	printseg(psei->AlignmentFixupCount);
	printseg(psei->ExceptionDispatchCount);
	printseg(psei->FloatingEmulationCount);
	printseg(psei->ByteWordEmulationCount);
}

// void GetSystemCrashDumpStateInformation(SYSTEM_CRASHDUMP_STATE_INFORMATION* pscsi)//33
// {
// }

//SystemCrashDumpStateInformation 34
void GetSystemKernelDebuggerInformation(SYSTEM_KERNEL_DEBUGGER_INFORMATION* pskdi)//35
{
	cout << "\t\t35 SystemKernelDebuggerInformation" << endl;
	printseg(pskdi->KernelDebuggerEnabled);
	printseg(pskdi->KernelDebuggerNotPresent);
}

void GetSystemContextSwitchInformation(SYSTEM_CONTEXT_SWITCH_INFORMATION* pscsi)//36
{
	cout << "\t\t36 SystemContextSwitchInformation" << endl;
	printseg(pscsi->ContextSwitches);
	printseg(pscsi->FindAny);
	printseg(pscsi->FindLast);
	printseg(pscsi->FindIdeal);
	printseg(pscsi->IdleAny);
	printseg(pscsi->IdleCurrent);
	printseg(pscsi->IdleLast);
	printseg(pscsi->IdleIdeal);
	printseg(pscsi->PreemptAny);
	printseg(pscsi->PreemptCurrent);
	printseg(pscsi->PreemptLast);
	printseg(pscsi->SwitchToIdle);
}

void GetSystemRegistryQuotaInformation(SYSTEM_REGISTRY_QUOTA_INFORMATION* psrqi)//37
{
	cout << "\t\t37 SystemRegistryQuotaInformation" << endl;
	printseg(psrqi->RegistryQuotaAllowed);
	printseg(psrqi->RegistryQuotaUsed);
	printseg(psrqi->PagedPoolSize);
}

// void GetSystemExtendServiceTableInformation(SYSTEM_EXTEND_SERVICE_TABLE_INFORAMTION* psesti)//38
// {
// }

// void GetSystemPrioritySeperation(SYSTEM_PRIORITY_SEPERATION* psps)// 39
// {
// }

// void GetSystemVerifierAddDriverInformation(SYSTEM_VERIFIER_ADDDRIVER_INFORAMTION* psvai) //40
// {
// }

// void GetSystemVerifierRemoveDriverInformation(SYSTEM_VERIFIER_REMOVE_DRIVER_INFORMATION* psvrdi) //41
// {
// }

void GetSystemProcessorIdleInformation(SYSTEM_PROCESSOR_IDLE_INFORMATION* pspii)//42
{
	cout << "\t\t42 SystemProcessorIdleInformation" << endl;
	printseg(pspii->IdleTime);
	printseg(pspii->C1Time);
	printseg(pspii->C2Time);
	printseg(pspii->C3Time);
	printseg(pspii->C1Transitions);
	printseg(pspii->C2Transitions);
	printseg(pspii->C3Transitions);
	printseg(pspii->Padding);
}

void GetSystemLegacyDriverInformation(SYSTEM_LEGACY_DRIVER_INFORMATION* psldi)
{
	cout << "\t\t43 SystemLegacyDriverInformation" << endl;
	printseg(psldi->VetoType);
	cout << (wchar_t*)psldi->VetoList.Buffer << endl;
}

void GetSystemCurrentTimeZoneInformation(RTL_TIME_ZONE_INFORMATION* prtzi)
{
	cout << "\t\t44 SystemCurrentTimeZoneInformation" << endl;
	printseg(prtzi->Bias);
	cout << "Standard:" << endl;
	wcout << (wchar_t*)prtzi->StandardName << endl;
	cout << "\tYear:" << prtzi->StandardStart.Year << endl;
	cout << "\tMonth:" << prtzi->StandardStart.Month << endl;
	cout << "\tDay:" << prtzi->StandardStart.Day << endl;
	cout << "\tHour:" << prtzi->StandardStart.Hour << endl;
	cout << "\tMinute:" << prtzi->StandardStart.Minute << endl;
	cout << "\tSecond:" << prtzi->StandardStart.Second << endl;
	cout << "\tMilliseconds:" << prtzi->StandardStart.Milliseconds << endl;
	cout << "\tWeekday:" << prtzi->StandardStart.Weekday << endl;
	printseg(prtzi->StandardBias);

	cout << "Daylight:" << endl;
	wcout << (wchar_t*)prtzi->DaylightName << endl;
	cout << "\tYear:" << prtzi->DaylightStart.Year << endl;
	cout << "\tMonth:" << prtzi->DaylightStart.Month << endl;
	cout << "\tDay:" << prtzi->DaylightStart.Day << endl;
	cout << "\tHour:" << prtzi->DaylightStart.Hour << endl;
	cout << "\tMinute:" << prtzi->DaylightStart.Minute << endl;
	cout << "\tSecond:" << prtzi->DaylightStart.Second << endl;
	cout << "\tMilliseconds:" << prtzi->DaylightStart.Milliseconds << endl;
	cout << "\tWeekday:" << prtzi->DaylightStart.Weekday << endl;
	printseg(prtzi->DaylightBias);
}

void GetSystemLookasideInformation(SYSTEM_LOOKASIDE_INFORMATION* psli, int length)
{
	cout << "\t\t45 SystemLookasideInformation" << endl;
	int num = length / sizeof(SYSTEM_LOOKASIDE_INFORMATION);
	for (int i = 0; i < num;i++)
	{
		printseg((int)psli->CurrentDepth);
		printseg((int)psli->MaximumDepth);
		printseg(psli->TotalAllocates);
		printseg(psli->AllocateMisses);
		printseg(psli->TotalFrees);
		printseg(psli->FreeMisses);
		printseg(psli->Type);
		printseg(psli->Tag);
		printseg(psli->Size);
	}
}

// void GetSystemTimeSlipNotification(SYSTEM_TIME_SLIP_NOTIFICATION* pstsn)// 46
// {
// }

void GetSystemSessionCreate(ULONG* SessionId)//SystemSessionCreate 47
{
	cout << "\t\t47 SystemSessionCreate" << endl;
	cout << "SessionId" << endl;
}

// void GetSystemSessionDetach(SYSTEM_SESSION_DETACH* pssd)// 48
// {
// }

//void GetSystemSessionInformation(SYSTEM_SESSION_INFORMATION* pssi)// 49
// {
// }

void GetSystemRangeStartInformation(ULONG_PTR* data)//50
{
	cout << "\t\t50 SystemRangeStartInformation" << endl;
	cout << hex << data << endl;
	cout << dec;
}

void GetSystemVerifierInformation(SYSTEM_VERIFIER_INFORMATION* psvi)// 51
{
	cout << "\t\t51 SystemVerifierInformation" << endl;
	ULONG offset = 0;
	int index = 0;
	do
	{
		offset = psvi->NextEntryOffset;
		cout << index + 1 << ":" << endl;
		printseg(psvi->Level);
		wcout << (wchar_t*)psvi->DriverName.Buffer << endl;
		printseg(psvi->RaiseIrqls);
		printseg(psvi->AcquireSpinLocks);
		printseg(psvi->SynchronizeExecutions);
		printseg(psvi->AllocationsAttempted);
		printseg(psvi->AllocationsSucceeded);
		printseg(psvi->AllocationsSucceededSpecialPool);
		printseg(psvi->AllocationsWithNoTag);
		printseg(psvi->TrimRequests);
		printseg(psvi->Trims);
		printseg(psvi->AllocationsFailed);
		printseg(psvi->AllocationsFailedDeliberately);
		printseg(psvi->Loads);
		printseg(psvi->Unloads);
		printseg(psvi->UnTrackedPool);
		printseg(psvi->CurrentPagedPoolAllocations);
		printseg(psvi->CurrentNonPagedPoolAllocations);
		printseg(psvi->PeakPagedPoolAllocations); 
		printseg(psvi->PeakNonPagedPoolAllocations);
		printseg(psvi->PagedPoolUsageInBytes);
		printseg(psvi->NonPagedPoolUsageInBytes);
		printseg(psvi->PeakPagedPoolUsageInBytes);
		printseg(psvi->PeakNonPagedPoolUsageInBytes);
		psvi = (SYSTEM_VERIFIER_INFORMATION*)((BYTE*)psvi + offset);
	} while (offset);
}

// void GetSystemVerifierThunkExtend(SYSTEM_VERIFIER_THUNK_EX* psvie)//52
// {
// }

void GetSystemSessionProcessInformation(SYSTEM_SESSION_PROCESS_INFORMATION* psspi)//53
{
	cout << "\t\t53 SystemSessionProcessInformation" << endl;
	printseg(psspi->SessionId);
	printseg(psspi->SizeOfBuf);
//	SYSTEM_SESSION_POOLTAG_INFORMATION* psspti = (SYSTEM_SESSION_POOLTAG_INFORMATION*)psspi->Buffer
}

//54 SystemLoadGdiDriverInSystemSpace
void GetSystemNumaProcessorMap(SYSTEM_NUMA_INFORMATION* psni)
{
	cout << "\t\t55 SystemLoadGdiDriverInSystemSpace" << endl;
	printseg(psni->HighestNodeNumber);
	for (int i = 0; i < MAXIMUM_NUMA_NODES; i++)
	{
		cout << "\tActiveProcessorsAffinityMask" << i << psni->ActiveProcessorsAffinityMask [i] << endl;
		cout << "\tAvailableMemory" << i << psni->AvailableMemory[i] << endl;
	}
}

//56 SystemPrefetcherInformation

void GetSystemExtendedProcessInformation(SYSTEM_PROCESS_INFORMATION* pspri1)//57
{
	cout << "\t\t57 SystemExtendedProcessInformation" << endl;
	do
	{
		if (pspri1->ImageName.Buffer)
			wcout << "\tImageName:" << wstring((wchar_t*)pspri1->ImageName.Buffer) << endl;
		else
			wcout << "no name" << endl;
		cout << "\t线程数:" << pspri1->NumberOfThreads << endl;
		printseg(pspri1->SpareLi1.QuadPart);
		printseg(pspri1->SpareLi2.QuadPart);
		printseg(pspri1->SpareLi3.QuadPart);
		cout << "\t创建时间:" << pspri1->CreateTime.QuadPart << endl;
		cout << "\t用户态时间:" << pspri1->UserTime.QuadPart << endl;
		cout << "\t内核态时间:" << pspri1->KernelTime.QuadPart << endl;
		cout << "\t基础优先级:" << pspri1->BasePriority << endl;
		cout << "\t进程Id:" << (int)pspri1->UniqueProcessId << endl;
		cout << "\t父进程Id:" << (int)pspri1->InheritedFromUniqueProcessId << endl;
		cout << "\t句柄数:" << pspri1->HandleCount << endl;
		cout << "\t会话Id:" << pspri1->SessionId << endl;
		cout << "\t页目录机制:" << pspri1->PageDirectoryBase << endl;
		cout << "\t虚拟内存峰值:" << pspri1->PeakVirtualSize << endl;
		cout << "\t虚拟内存大小:" << pspri1->VirtualSize << endl;
		cout << "\t页错误数:" << pspri1->PageFaultCount << endl;
		cout << "\t物理内存峰值:" << pspri1->PeakWorkingSetSize << endl;
		cout << "\t物理内存大小:" << pspri1->WorkingSetSize << endl;
		cout << "\t分页池配额峰值:" << pspri1->QuotaPeakPagedPoolUsage << endl;
		cout << "\t分页池配额:" << pspri1->QuotaPagedPoolUsage << endl;
		cout << "\t非分页池配额峰值:" << pspri1->QuotaPeakNonPagedPoolUsage << endl;
		cout << "\t非分页池配额:" << pspri1->QuotaNonPagedPoolUsage << endl;
		cout << "\t页面文件使用:" << pspri1->PagefileUsage << endl;
		cout << "\t页面文件使用峰值:" << pspri1->PeakPagefileUsage << endl;
		cout << "\t私有页面数:" << pspri1->PrivatePageCount << endl;
		cout << "\t读操作数:" << pspri1->ReadOperationCount.QuadPart << endl;
		cout << "\t写操作数:" << pspri1->WriteOperationCount.QuadPart << endl;
		cout << "\t其他操作数:" << pspri1->OtherOperationCount.QuadPart << endl;
		cout << "\t读字节数:" << pspri1->ReadTransferCount.QuadPart << endl;
		cout << "\t写字节数:" << pspri1->WriteTransferCount.QuadPart << endl;
		cout << "\t其他字节数:" << pspri1->OtherTransferCount.QuadPart << endl;
		SYSTEM_PROCESS_INFORMATION* newpspri1 = (SYSTEM_PROCESS_INFORMATION*)((BYTE*)pspri1 + pspri1->NextEntryOffset);
		SYSTEM_EXTENDED_THREAD_INFORMATION* pesti = (SYSTEM_EXTENDED_THREAD_INFORMATION*)(pspri1 + 1);
		int threadindex = 0;
		while ((LPVOID)pesti < (LPVOID)newpspri1)
		{
			++threadindex;
			cout << "\t内核态时间:" << pesti->ThreadInfo.KernelTime.QuadPart << endl;
			cout << "\t用户态时间:" << pesti->ThreadInfo.UserTime.QuadPart << endl;
			cout << "\t创建时间:" << pesti->ThreadInfo.CreateTime.QuadPart << endl;
			cout << "\t等待时间:" << pesti->ThreadInfo.WaitTime << endl;
			cout << "\t起始地址:" << hex << pesti->ThreadInfo.StartAddress << endl;
			cout << "\tUniqueProcess:" << hex << pesti->ThreadInfo.ClientId.UniqueProcess << endl;
			cout << "\tUniqueThread:" << hex << pesti->ThreadInfo.ClientId.UniqueThread << endl;
			cout << dec;
			cout << "\t优先级:" << pesti->ThreadInfo.Priority << endl;
			cout << "\t基础优先级:" << pesti->ThreadInfo.BasePriority << endl;
			cout << "\t模式切换次数:" << pesti->ThreadInfo.ContextSwitches << endl;
			cout << "\t线程状态:" << pesti->ThreadInfo.ThreadState << endl;
			cout << "\t等待原因:" << pesti->ThreadInfo.WaitReason << endl;
			cout << "\t栈基址:" << hex << pesti->StackBase << endl;
			cout << "\t栈范围:" << hex << pesti->StackLimit << endl;
			cout << "\tWin32StartAddress" << hex << pesti->Win32StartAddress << endl;
			cout << dec;
			pesti++;
		}
		pspri1 = newpspri1;
	} while (pspri1->NextEntryOffset);
}

void GetSystemRecommendedSharedDataAlignment(ULONG* data)//58
{
	cout << "\t\t`58 SystemRecommendedSharedDataAlignment" << endl;
	cout << *data << endl;
}

void GetSystemComPlusPackage(ULONG* data)//59
{
	cout << "\t\t 59 SystemComPlusPackage" << endl;
	cout << *data << endl;
}

void GetSystemNumaAvailableMemory(SYSTEM_NUMA_INFORMATION* psni)//60
{
	cout << "\t\t60 SystemNumaAvailableMemory" << endl;
	printseg(psni->HighestNodeNumber);
	for (int i = 0; i < MAXIMUM_NUMA_NODES; i++)
	{
		cout << "\tActiveProcessorsAffinityMask" << i << psni->ActiveProcessorsAffinityMask[i] << endl;
		cout << "\tAvailableMemory" << i << psni->AvailableMemory[i] << endl;
	}
}

void GetSystemProcessorPowerInformation(SYSTEM_PROCESSOR_POWER_INFORMATION* psppi)//61
{
	cout << "\t\t61 SystemProcessorPowerInformation" << endl;
	printseg((int)psppi->CurrentFrequency);
	printseg((int)psppi->ThermalLimitFrequency);
	printseg((int)psppi->ConstantThrottleFrequency);
	printseg((int)psppi->DegradedThrottleFrequency);
	printseg((int)psppi->LastBusyFrequency);
	printseg((int)psppi->LastC3Frequency);
	printseg((int)psppi->LastAdjustedBusyFrequency);
	printseg((int)psppi->ProcessorMinThrottle);
	printseg((int)psppi->ProcessorMaxThrottle);
	printseg(psppi->NumberOfFrequencies);
	printseg(psppi->PromotionCount);
	printseg(psppi->DemotionCount);
	printseg(psppi->ErrorCount);
	printseg(psppi->RetryCount);
	printseg(psppi->CurrentFrequencyTime);
	printseg(psppi->CurrentProcessorTime);
	printseg(psppi->CurrentProcessorIdleTime);
	printseg(psppi->LastProcessorTime);
	printseg(psppi->LastProcessorIdleTime);
}

void GetSystemEmulationBasicInformation(SYSTEM_BASIC_INFORMATION* psbi)//62
{
	cout << "\t\t62 SystemEmulationBasicInformation" << endl;
	cout << "\t时间解析度ms:" << psbi->TimerResolution << endl;
	cout << "\t物理页大小:" << psbi->PageSize << endl;
	cout << "\t物理页个数:" << psbi->NumberOfPhysicalPages << endl;
	cout << "\t最小物理页个数:" << psbi->LowestPhysicalPageNumber << endl;
	cout << "\t最大物理页个数:" << psbi->HighestPhysicalPageNumber << endl;
	cout << "\t逻辑页大小:" << psbi->AllocationGranularity << endl;
	cout << "\t最小用户地址:" << psbi->MinimumUserModeAddress << endl;
	cout << "\t最大用户地址:" << psbi->MaximumUserModeAddress << endl;
	cout << "\t处理器个数:" << (int)psbi->NumberOfProcessors << endl;
}

void GetSystemEmulationProcessorInformation(SYSTEM_PROCESSOR_INFORMATION* pspri)//63
{
	cout << "\t\t63 SystemEmulationProcessorInformation" << endl;
	switch (pspri->ProcessorArchitecture)
	{
	case PROCESSOR_ARCHITECTURE_INTEL:
		cout << "\tINTEL ";
		if (pspri->ProcessorLevel == 3)
			cout << "386 ";
		else if (pspri->ProcessorLevel == 4)
			cout << "486 ";
		else if (pspri->ProcessorLevel == 5)
			cout << "586 or Pentium ";
		break;
	case PROCESSOR_ARCHITECTURE_IA64:
		cout << "IA64 ";
		if (pspri->ProcessorLevel == 7)
			cout << "Itanium ";
		else if (pspri->ProcessorLevel == 31)
			cout << "Itanium 2 ";
		break;
	}
	cout << pspri->ProcessorRevision << " " << pspri->ProcessorFeatureBits << endl;
}

//64 SystemExtendedHandleInformation
//65 SystemLostDelayedWriteInformation
void GetSystemBigPoolInformation(SYSTEM_BIGPOOL_INFORMATION* psbi)//66
{
	cout << "\t\t63 SystemEmulationProcessorInformation" << endl;
	for (int i = 0; i < psbi->Count; i++)
	{
		cout << hex << "\tVirutalAddress:" << psbi->AllocatedInfo[i].VirtualAddress;
		cout << dec;
		cout << "\tSizeInBytes:" << psbi->AllocatedInfo[i].SizeInBytes;
		cout << "\tTagUlong:" << psbi->AllocatedInfo[i].TagUlong << endl;
	}
}

void GetSystemSessionPoolTagInformation(SYSTEM_SESSION_PROCESS_INFORMATION* psspi)//67
{
	cout << "\t\t67 SystemSessionPoolTagInformation" << endl;
	printseg(psspi->SessionId);
	printseg(psspi->SizeOfBuf);
}

void GetSystemSessionMappedViewInformation(SYSTEM_SESSION_MAPPED_VIEW_INFORMATION* pssmvi)//68
{
	cout << "\t\t68 SystemSessionMappedViewInformation" << endl;
	SIZE_T nextoffset = 0;
	do
	{
		pssmvi = (SYSTEM_SESSION_MAPPED_VIEW_INFORMATION*)((BYTE*)pssmvi + (pssmvi->NextEntryOffset));
	} while (nextoffset);
}

//69 SystemHotpatchInformation

void GetSystemObjectSecurityMode(ULONG* data)//70
{
	cout << "\t\t69 SystemObjectSecurityMode" << endl;
	cout << *data << endl;
}

void GetSystemWatchdogTimerHandler(SYSTEM_WATCHDOG_HANDLER_INFORMATION* pswhi)//71
{
}

void GetSystemWatchdogTimerInformation(SYSTEM_WATCHDOG_TIMER_INFORMATION* pswti)//72
{
}

void GetSystemLogicalProcessorInformation(SYSTEM_LOGICAL_PROCESSOR_INFORMATION* pslpi)//73
{
	cout << "\t\t73 SystemLogicalProcessorInformation" << endl;
}

//74 SystemWow64SharedInformation
void GetSystemRegisterFirmwareTableInformationHandler(SYSTEM_FIRMWARE_TABLE_HANDLER* psfth)//75
{
	cout << "\t\t75 SystemRegisterFirmwareTableInformationHandler" << endl;
}

void GetSystemFirmwareTableInformation(SYSTEM_FIRMWARE_TABLE_HANDLER* psfth)//76
{
	cout << "\t\t76 SystemFirmwareTableInformation" << endl;
}

// void GetSystemExtendedHandleInformation(SYSTEM_HANDLE_TABLE_ENTRY_INFO_EX* pshteie)
// {
// 	cout << "\t\t64 SystemExtendedHandleInformation" << endl;
// 	for (int i = 0; i < pshteie->NumberOfHandles; i++)
// 	{
// 		cout << "\t" << i + 1 << endl;
// 		cout << "\tUniqueProcessId:" << (int)pshteie->Handles[i].UniqueProcessId << endl;
// 		cout << "\tCreatorBackTraceIndex:" << (int)pshteie->Handles[i].CreatorBackTraceIndex << endl;
// 		cout << "\tObjectTypeIndex:" << (int)pshteie->Handles[i].ObjectTypeIndex << endl;
// 		cout << "\tHandleAttributes:" << (int)pshteie->Handles[i].HandleAttributes << endl;
// 		cout << "\tHandleValue:" << (int)pshteie->Handles[i].HandleValue << endl;
// 		cout << "\tObject:" << hex << (int)pshteie->Handles[i].Object << endl;
// 		cout << "\tGrantedAccess:" << hex << (int)pshteie->Handles[i].GrantedAccess << endl;
// 	}
// }

//77 SystemModuleInformationEx
//78 SystemVerifierTriageInformation
//79 SystemSuperfetchInformation
//80 SystemMemoryListInformation

void GetSystemFileCacheInformationEx(SYSTEM_FILECACHE_INFORMATION* psfci)//81
{
	cout << "\t\t81 SystemFileCacheInformationEx" << endl;
	printseg(psfci->CurrentSize);
	printseg(psfci->PeakSize);
	printseg(psfci->PageFaultCount);
	printseg(psfci->MinimumWorkingSet);
	printseg(psfci->MaximumWorkingSet);
	printseg(psfci->CurrentSizeIncludingTransitionInPages);
	printseg(psfci->PeakSizeIncludingTransitionInPages);
	printseg(psfci->TransitionRePurposeCount);
	printseg(psfci->Flags);
}

#define do(x) GetInformationTemplate(x,Get##x)

void main()
{
// 	do(SystemBasicInformation);//0
// 	do(SystemProcessorInformation);//1
// 	do(SystemPerformanceInformation);//2
// 	do(SystemTimeOfDayInformation);//3
//	do(SystemPathInformation);//4
// 	do(SystemProcessInformation);//5
// 	do(SystemCallCountInformation);//6
// 	do(SystemDeviceInformation);//7
// 	do(SystemProcessorPerformanceInformation);//8
// 	do(SystemFlagsInformation);//9
//	do(SystemCallTimeInformation);//10
//	do(SystemModuleInformation);//11
//	do(SystemLocksInformation);//12
//	do(SystemStackTraceInformation);//13
//	do(SystemPagedPoolInformation);//14
//	do(SystemNonPagedPoolInformation);//15
//	do(SystemHandleInformation);//16
//	do(SystemObjectInformation);//17
//	do(SystemPageFileInformation);//18
//	do(SystemVdmInstemulInformation);//19
//	do(SystemVdmBopInformation);//20
//	do(SystemFileCacheInformatio);//21
// 	do(SystemPoolTagInformation);//22
// 	do(SystemInterruptInformation);//23
// 	do(SystemDpcBehaviorInformation);//24
// 	do(SystemFullMemoryInformation);//25
// 	do(SystemLoadGdiDriverInformation);//26
//	do(SystemUnloadGdiDriverInformation);//27
//	do(SystemTimeAdjustmentInformation);//28
//	do(SystemSummaryMemoryInformation);//29
//	do(SystemMirrorMemoryInformation);//30
//	do(SystemPerformanceTraceInformation);//31
//	do(SystemObsolete0);//32
//	do(SystemExceptionInformation);//33
//	do(SystemCrashDumpStateInformation);//34
// 	do(SystemKernelDebuggerInformation);//35
// 	do(SystemContextSwitchInformation);//36
// 	do(SystemRegistryQuotaInformation);//37
//	do(SystemExtendServiceTableInformation);//38
//	do(SystemPrioritySeperation);//39
//	do(SystemVerifierAddDriverInformation);//40
//	do(SystemVerifierRemoveDriverInformation);//41
//	do(SystemProcessorIdleInformation);//42
//	do(SystemLegacyDriverInformation);//43
//	do(SystemCurrentTimeZoneInformation);//44
//	do(SystemLookasideInformation);//45
//	do(SystemTimeSlipNotification);//46
//	do(SystemSessionCreate);//47
//	do(SystemSessionDetach);//48
//	do(SystemSessionInformation);//49
//	do(SystemRangeStartInformation);//50
//	do(SystemVerifierInformation);//51
//	do(SystemVerifierThunkExtend);//52
//	do(SystemSessionProcessInformation);//53
//	do(SystemLoadGdiDriverInSystemSpace);//54
//	do(SystemNumaProcessorMap);//55
//	do(SystemPrefetcherInformation);//56
//	do(SystemExtendedProcessInformation);//57
// 	do(SystemRecommendedSharedDataAlignment);//58
// 	do(SystemComPlusPackage);//59
// 	do(SystemNumaAvailableMemory);//60
// 	do(SystemProcessorPowerInformation);//61
// 	do(SystemEmulationBasicInformation);//62
// 	do(SystemEmulationProcessorInformation);//63
//	do(SystemExtendedHandleInformation);//64
//	do(SystemLostDelayedWriteInformation);//65
//	do(SystemBigPoolInformation);//66
//	do(SystemSessionPoolTagInformation);//67
//	do(SystemSessionMappedViewInformation);//68
//	do(SystemHotpatchInformation);//69
// 	do(SystemObjectSecurityMode);//70
// 	do(SystemWatchdogTimerHandler);//71
// 	do(SystemWatchdogTimerInformation);//72
// 	do(SystemLogicalProcessorInformation);//73
//	do(SystemWow64SharedInformation);//74
//	do(SystemRegisterFirmwareTableInformationHandler);//75
//	do(SystemFirmwareTableInformation);//76
//	do(SystemModuleInformationEx);//77
//	do(SystemBigPoolInformation);//78
//	do(SystemSessionPoolTagInformation);//79
//	do(SystemSessionMappedViewInformation);//80
//	do(SystemHotpatchInformation);//81
// 	do(SystemObjectSecurityMode);//82
// 	do(SystemWatchdogTimerHandler);//82
// 	do(SystemWatchdogTimerInformation);//83
//	do(SystemLogicalProcessorInformation);//84
//	do(SystemWow64SharedInformation);//85
// 	do(SystemRegisterFirmwareTableInformationHandler);//86
// 	do(SystemFirmwareTableInformation);//87
//	do(SystemModuleInformationEx);//88
//	do(SystemVerifierTriageInformation);//89
//	do(SystemSuperfetchInformation);//90
//	do(SystemMemoryListInformation);//91
//	do(SystemFileCacheInformationEx);//92
}
```