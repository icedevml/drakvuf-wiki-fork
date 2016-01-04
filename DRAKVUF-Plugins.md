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