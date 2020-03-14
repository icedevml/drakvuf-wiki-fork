DRAKVUF consists of several plugins, each collecting different aspects of the guests' execution, like logging system calls or tracking kernel heap allocations.

Currently available plugins:
- [syscalls](#syscalls)
- [poolmon](#poolmon)
- [objmon](#objmon)
- [exmon](#exmon)
- [filetracer](#filetracer)
- [filedelete](#filedelete)
- [ssdtmon](#ssdtmon)
- [socketmon](#socketmon)
- [apimon, memdump](#apimon-memdump)

syscalls
--------
The `syscalls` plugin is responsible for tracking the execution of function-entry-points responsible to handling system calls on Windows and Linux. The function accomplishes this by looping through the Rekall-profile and using a BREAKPOINT trap on each function whose name starts with `Nt` on Windows and `sys_` on Linux.

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
The `filetracer` plugin monitors the use of `_FILE_OBJECT` structures by system-calls as well as internal kernel functions used by kernel drivers. With this approach we get a complete view of files being accessed on the system.

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

SSDTmon
----------
The SSDTmon plugin monitors write-memory accesses to the System Service Descriptor Table used to store pointers to the system call handling functions. If malware hooks this table and redirects system calls, the `syscalls` plugin is affected as the original function(s) may no longer get called where it originally trapped. If this plugin detects a change, one must assume that the syscall plugin output is no longer complete.

socketmon
----------
The socketmon plugin monitors the usage of TCP and UPD sockets for Windows guests. It requires the creation of a Rekall profile for the tcpip.sys kernel module, which is normally located at `C:\Windows\System32\drivers\tcpip.sys`. You will need to copy this file to where you will be generating the Rekall profile at. To generate a Rekall profile for it you can use the `pdbparse` project to obtain the PDB:
```
    apt-get install python-construct python-pefile
    git clone https://github.com/moyix/pdbparse
    cd pdbparse
    python setup.py build
    cd examples
    ./symchk.py -e tcpip.sys
```
Then you can use Rekall to create the profile:
```
    rekal parse_pdb tcpip.pdb > tcpip.json
```

memdump
-------
This plugin is responsible for taking memory dumps whenever some memory appears to be interesting (by appropriate heuristics). By default, memory dumps are not stored anywhere unless `--memdump-dir <dir>` is provided.

apimon
---------------
The `apimon` plugin is able to monitor WinAPI calls. You can tune what do you want to trace by starting DRAKVUF with `--dll-hooks-list <dll-hooks-list-file>` command line switch. This configuration is common with the `memdump` plugin and will be also consumed by it.

The current format for `dll-hooks-list` file is:

```
<dll-name>,<function-name>,<strategy>,<arg1>,<arg2>,<arg3...>
```

where:
* `dll-name` is a name of DLL you want to hook, e.g. `ntdll.dll`
* `function-name` is a name of function exported by this DLL, e.g. `LdrLoadDll`
* `strategy` is one of:
  * `log` (`apimon` will log that the function was called, what were the argument values and what was the return value)
  * `stack` (`memdump` will perform stack inspection and will try to make some memory dumps after this function is called)
  * `log+stack` (both above)

See [src/plugins/apimon/example/dll-hooks-list-win7x64](https://github.com/tklengyel/drakvuf/blob/master/src/plugins/apimon/example/dll-hooks-list-win7x64) for an example configuration file dedicated for Windows 7 (x64).
