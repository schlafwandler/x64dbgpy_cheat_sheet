# Unofficial x64dbgpy cheat sheet

This document is my personal cheat sheet for the [x64dbg python plugin](https://github.com/x64dbg/x64dbgpy).
It is not a complete reference (and does not aim to be), but more a list of things I might find useful in the next CTF and that might be useful to others as well.

## Simple operations
#### Execute x64dbg command
[Command list](https://help.x64dbg.com/en/latest/commands/index.html)
```python
pluginsdk.x64dbg.DbgCmdExecDirect("bpx 000000013F9A1330")
```

### Execution
#### Run/continue
```python
pluginsdk.Run()
```
#### Stop
```python
pluginsdk.Stop()
```
#### Step into
```python
pluginsdk.StepIn()
```
#### Step over
```python
pluginsdk.StepOver()
```
#### Step out
```python
pluginsdk.StepOut()
```

### Registers
#### Get register
```python
rax = Register.RAX
# or
rax = pluginsdk.GetRAX()
```
#### Set register
```python
Register.RAX = 0
# or
pluginsdk.SetRAX(0x00)
```

## Process information

#### Main executable name
```python
module_name = pluginsdk.GetMainModuleName()
```

#### Main executable path
```python
module_path = pluginsdk.GetMainModulePath()
```

#### Main executable base
```python
module_base = pluginsdk.GetMainModuleBase()
```

#### Is it a 64 bit process?
```python
is_64bit = pluginsdk.is_64bit()
```

#### Get module base from address
```python
rip = pluginsdk.GetRIP() 
module_base = pluginsdk.BaseFromAddr(rip) # any address in the module works
```

#### Get module base from module name
```python
kernel32_base = pluginsdk.BaseFromName("kernel32.dll") 
```

#### GetProcAddress
```python
p_IsDebuggerPresent = pluginsdk.RemoteGetProcAddress('kernel32', 'IsDebuggerPresent')
```

## Memory
#### Get memory permissions
See [Memory Protection Constants](https://docs.microsoft.com/de-de/windows/win32/memory/memory-protection-constants).
```python
protection = pluginsdk.GetProtect(pluginsdk.GetRSP())
```

#### Read from memory
```python
addr = pluginsdk.GetRSP()
size = 8
data = pluginsdk.Read(addr,size)
```

#### Write to memory
```python
pluginsdk.Write(addr,"AAAAAAAA")
pluginsdk.Write(addr,"00000000".decode("hex"))
pluginsdk.Write(addr,data)
```

#### Test if pointer is valid
```python
if pluginsdk.IsValidPtr(Register.RBX):
	print("RBX is a valid pointer")
```

#### Allocate memory in debuggee
```python
mem = pluginsdk.RemoteAlloc(0x1000) # Allocate 0x1000 bytes
```

#### Free allocated memory in debuggee 
```python
pluginsdk.RemoteFree(mem)
```

#### Read and manipulate a struct from memory
Example on how to read memory, parse it as a ctypes Structure, and write it back.
```python
addr = pluginsdk.GetRSP() # or any other address

from ctypes import Structure,c_int64,sizeof

class DATA(Structure):
    _pack_=1
    _fields_= [
            ("a", c_int64),
            ("b", c_int64*4)
            ]

data = pluginsdk.Read(addr,sizeof(DATA))
data_struct = DATA.from_buffer_copy(data)

print("DATA.a: %x"%(data_struct.a))
print("DATA.b[2]: %x"%(data_struct.b[2]))

data_struct.a = -1
data_struct.b[0] = 0x41414141
data_struct.b[1] = 0x42424242
data_struct.b[2] = 0x43434343
data_struct.b[3] = 0x44444444
pluginsdk.Write(addr,bytearray(data_struct))
```

## Breakpoints

#### Set a breakpoint without a callback function
```python
pluginsdk.SetBreakpoint(pluginsdk.GetRIP())
```

#### Set breakpoint with a callback function
```python
def callback():
    print("Breakpoint callback")

bp_addr = pluginsdk.GetRIP() # Example address
Breakpoint.add(bp_addr,callback)
```
See [issue #1915](https://github.com/x64dbg/x64dbg/issues/1915) for stepping/running from breakpoint handlers.
A workaround for this is [here](https://github.com/schlafwandler/x64dbgpy_cheat_sheet#call-and-continue-breakpoint)

#### Hardware breakpoints
```python
def callback():
    print("Hardware breakpoint!")

bp_addr = pluginsdk.GetRIP() # Example address
Breakpoint.add(bp_addr,callback,bp_type=Breakpoint.BP_HARDWARE,hw_type=Breakpoint.HW_WRITE)
```
The possible values for `bp_type` are `Breakpoint.HW_ACCESS`, `Breakpoint.HW_WRITE` and `Breakpoint.HW_EXECUTE`, with `Breakpoint.HW_EXECUTE` as default.
For stepping/running from the callback see [here](https://github.com/schlafwandler/x64dbgpy_cheat_sheet#set-breakpoint-with-a-callback-function).

#### Set breakpoint at RVA
A function to set a breakpoint at a [relative virtual address (RVA)](https://en.wikipedia.org/wiki/COFF#Relative_virtual_address).
```python
def breakpoint_at_RVA(module_name,rva,callback):
    module_base = pluginsdk.BaseFromName(module_name)
    bp_addr = module_base + rva
    Breakpoint.add(bp_addr,callback)

def callback():
    print("Breakpoint callback")
    
breakpoint_at_RVA(module_name,0x1000,callback)
```

#### "Call and continue"-Breakpoint
Set up a breakpoint that calls a function and continues, without interrupting the program.
A (somewhat hacky) workaround for [issue #1915](https://github.com/x64dbg/x64dbg/issues/1915).

The trick here is to set a breakpoint with a condition that's always false, and a command condition that's always true.
That way the process is not stopped, but the command is executed anyway.
The breakpoint command itself is `python <callback name>` and simply calls the given function.
See the documentation for [Conditional Breakpoints](https://help.x64dbg.com/en/latest/introduction/ConditionalBreakpoint.html) for details.

The process can still be paused from the callback by executing `pluginsdk.x64dbg.DbgCmdExecDirect("$breakpointcondition=1")`.

```python
def callback():
	print("breakpoint callback")

    if Register.RAX == 0: # pause debugger if RAX==0
        pluginsdk.x64dbg.DbgCmdExecDirect("$breakpointcondition=1")

def BpxContinueAfterCallback(module_name,rva,callback):
	module_base = pluginsdk.BaseFromName(module_name)
	addr = module_base + rva

	pluginsdk.x64dbg.DbgCmdExecDirect("SetBPX %x"%(addr))
	pluginsdk.x64dbg.DbgCmdExecDirect('SetBreakpointCondition %x, "0"'%(addr)) # do not actually break
	pluginsdk.x64dbg.DbgCmdExecDirect('SetBreakpointCommand %x, "python %s()"'%(addr,callback.func_name))
	pluginsdk.x64dbg.DbgCmdExecDirect('SetBreakpointCommandCondition %x, "1"'%(addr)) # execute the command

rva = 0x15F8 # some RVA
module_name = pluginsdk.GetMainModuleName()
BpxContinueAfterCallback(module_name,rva,callback)
```

#### Delete breakpoint (pluginsdk)
```python
pluginsdk.DeleteBreakpoint(pluginsdk.GetRIP())
```
(might create problems with breakpoints created with `Breakpoint.add`)

## GUI
#### MessageBox
```python
pluginsdk.Message("Message")
```

#### Yes/No MessageBox
```python
answer_yes = pluginsdk.MessageYesNo("Question")
```

#### Input value
```python
value = pluginsdk.InputValue("Message text")
```

#### Input line
```python
line = pluginsdk.GuiGetLineWindow()
```

### Selection
#### Current selection in disassembly window (start, end)
```python
start,end = pluginsdk.Disassembly_SelectionGet()
# or
start,end = pluginsdk.Gui_SelectionGet(0)
```

#### Current selection in memory window (start, end)
```python
start,end = pluginsdk.Dump_SelectionGet()
# or
start,end = pluginsdk.Gui_SelectionGet(1)
```

#### Current selection in stack window (start, end)
```python
start,end = pluginsdk.Stack_SelectionGet()
# or
start,end = pluginsdk.Gui_SelectionGet(2)
```

### Navigation
[Commands](https://help.x64dbg.com/en/latest/commands/gui/index.html)

#### Navigate to location in disassembly subwindow
```python
pluginsdk.x64dbg.DbgCmdExecDirect("d 000000013F9A1330")
```

#### Navigate to location in dump subwindow
```python
pluginsdk.x64dbg.DbgCmdExecDirect("dump 000000013F9A1330")
```

#### Navigate to location in stack dump subwindow
```python
pluginsdk.x64dbg.DbgCmdExecDirect("sdump 000000000012F230")
```

#### Graph address
```python
pluginsdk.x64dbg.DbgCmdExecDirect("graph 000000013F9A1330")
```



## Events

```python
class Event_handler:
    def event_callback(self, **kwargs):
        print("Event callback")
        print(kwargs)
    def create_process_event_handler(self, **kwargs):
        print("Create process event callback")
        print(kwargs)
    def load_dll_event_handler(self, **kwargs):
        print("Load dll event callback")
        print(kwargs)
handler = Event_handler()

# not all of them seem to work
Event.create_process = handler.create_process_event_handler
Event.exit_process = handler.event_callback
Event.create_thread = handler.event_callback
Event.exit_thread = handler.event_callback
Event.listen = handler.event_callback
Event.load_dll = handler.load_dll_event_handler
Event.unload_dll = handler.event_callback
Event.stop_debug = handler.event_callback
Event.trace_execute = handler.event_callback
Event.system_breakpoint = handler.event_callback
```
## Misc
### Command line
#### Get command line
```python
pluginsdk.x64dbg.DbgCmdExecDirect("getcmdline")
```
#### Set command line
```python
pluginsdk.x64dbg.DbgCmdExecDirect("setcmdline foo")
```
```python

```
