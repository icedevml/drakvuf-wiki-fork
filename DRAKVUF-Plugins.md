DRAKVUF Plugin Documentation
============================

DRAKVUF consists of several plugins, each collecting different aspects of the guests' execution, like logging system calls or tracking kernel heap allocations.


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

The inputs of to this functions can be found at different locations, depending whether the guest is 32-bit or 64-bit, as function-call conventions differ. On 32-bit systems the inputs are found on the stack, while on 64-bit systems the first four inputs are always passed via registers.
```