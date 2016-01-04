DRAKVUF Plugin Documentation
============================

DRAKVUF consists of several plugins, each collecting different aspects of the guests' execution, like logging system calls or tracking kernel heap allocations.

Currently available plugins:
- [syscalls](#syscalls)
- [poolmon](#poolmon)
- [objmon](#objmon)
- [exmon](#exmon)
- [filetracer](#filetracer)
- [filedelete](#filedelete)

syscalls
--------
The `syscalls` plugin is responsible for tracking the execution of function-entry-points responsible to handling system calls on Windows 7. The function accomplishes this by looping through the Rekall-profile of the Windows guest and using a BREAKPOINT trap on each function whose name starts with Nt.

Currently the function inputs and output is not tracked. Prototypes of the these functions have recently been collected and can be found at https://github.com/tklengyel/drakvuf/blob/master/src/plugins/syscalls/scproto.h and integrating this information into the plugin is an open TODO item.


poolmon
-------
The `poolmon` plugin tracks calls to the `ExAllocatePoolWithTag` function, which is responsible for allocating objects on the kernel heap in Windows.

The prototype of this function is defined as follows (form MSDN https://msdn.microsoft.com/en-us/library/windows/hardware/ff544520%28v=vs.85%29.aspx):
```
PVOID ExAllocatePoolWithTag(
  _In_ POOL_TYPE PoolType,
  _In_ SIZE_T    NumberOfBytes,
  _In_ ULONG     Tag
);
```

The inputs of to this functions can be found at different locations, depending whether the guest is 32-bit or 64-bit, as function-call conventions differ. On 32-bit systems the inputs are found on the stack, while on 64-bit systems the first four inputs are always passed via registers.

objmon
------
The `objmon` plugin monitors the execution of ObCreateObject. This function is also called when creating common objects in Windows. The ObjectType input defines an index into the Windows 7 type array, currently defining 42 objects, as can be seen in https://github.com/tklengyel/drakvuf/blob/master/src/plugins/objmon/private.h#L109

The ObCreateObject function prototype is defined as follows: 
```
 NTKERNELAPI
 NTSTATUS
     ObCreateObject (
     IN KPROCESSOR_MODE ObjectAttributesAccessMode OPTIONAL,
     IN POBJECT_TYPE ObjectType,
     IN POBJECT_ATTRIBUTES ObjectAttributes OPTIONAL,
     IN KPROCESSOR_MODE AccessMode,
     IN PVOID Reserved,
     IN ULONG ObjectSizeToAllocate,
     IN ULONG PagedPoolCharge OPTIONAL,
     IN ULONG NonPagedPoolCharge OPTIONAL,
     OUT PVOID *Object
 );
```

exmon
-----
The `exmon` plugin monitors the execution of KiDispatchException, which is the Windows exception handler function when an exception occurs in either user- or kernel-space. The plugin extracts the information from the TrapFrame input containing the CPU state when the exception occured.

The ReactOS definition of this function is as follows (from http://doxygen.reactos.org/d7/d7f/ntoskrnl_2ke_2amd64_2except_8c_a660d1a46ff201c5861caf9667937f73f.html):
```
VOID NTAPI KiDispatchException 	(
		IN PEXCEPTION_RECORD ExceptionRecord,
		IN PKEXCEPTION_FRAME ExceptionFrame,
		IN PKTRAP_FRAME      TrapFrame,
		IN KPROCESSOR_MODE   PreviousMode,
		IN BOOLEAN           FirstChance 
	) 
```

filetracer
----------
The `filetracer` plugin monitors the allocation of `_FILE_OBJECT` structures on the Windows kernel heap to extract the information about what files are being accessed in the guest. This operation requires multiple-steps.

1. Monitoring the execution of ExAllocatePoolWithTag and trapping the return point to the caller.
2. When the function returns from the allocation of a `_FILE_OBJECT`, WRITE monitoring the memory page where the structure was allocated.
3. When the FileName pointer is updated, reading out the path string and removing the WRITE monitoring from the page.

In our experience ExAllocatePoolWithTag is mostly called from a single location and thus the return location is always the same. As such, we don't actually remove the return point breakpoint, instead we verify that the object allocated is a `_FILE_OBJECT` by checking the `_POOL_HEADER` of the object.

filedelete
----------
The `filedelete` plugin monitors the execution of NtSetInformationFile and ZwSetInformationFile, which are functions responsible for deleting files (there are some others too, such as NtDeleteFile). When the function is called and the fifth input of the function is `FILE_DISPOSITION_INFORMATION` (13) the file path is determined by walking the handle table of the process via the DRAKVUF function `drakvuf_get_obj_by_handle`. Once the address is known, it be extracting using the Volatility plugin `dumpfiles`.

The function prototype according to MSDN (https://msdn.microsoft.com/en-us/library/windows/hardware/ff567096%28v=vs.85%29.aspx):
```
NTSTATUS ZwSetInformationFile(
  _In_  HANDLE                 FileHandle,
  _Out_ PIO_STATUS_BLOCK       IoStatusBlock,
  _In_  PVOID                  FileInformation,
  _In_  ULONG                  Length,
  _In_  FILE_INFORMATION_CLASS FileInformationClass
);
``` 