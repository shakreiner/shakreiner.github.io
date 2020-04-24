---
title: Break On Process Creation in WinDbg
description: WinDbg extension
date: 2020-04-24 10:00:00 +/-0000
categories: [Tools]
tags: [windbg]
image:
  path: /assets/img/2020-24-04-break-on-process-windbg/preview.png
  height: 628
  width: 1200
---
**The extension is available here**: [https://github.com/shakreiner/windbg-scripts/blob/master/BreakOnProcess/BreakOnProcess.js](https://github.com/shakreiner/windbg-scripts/blob/master/BreakOnProcess/BreakOnProcess.js)

WinDbg (short for Windows Debugger, sometimes pronounced *wind-bag*) is the go-to debugger for many of us in the security industry; whether it's due to its kernel/hypervisor debugging capabilities, its time travel debugging feature or the fact that it's designed specifically for the Windows environment.

In this post, we'll go through the writing process of a simple JS extension that will help us break in the context of new processes during kernel debugging. This can be useful when you need to understand the behavior of a process and how it interacts with the kernel. In order for us to be able to access a user-mode process while kernel debugging, we need to either break with it as the active process (currently being executed by the CPU) or to know the address of its `_EPROCESS`. This can be uncomfortable since it requires us to do the following:

1. Create the new process in the debugee
2. Break
3. Walk the processes list and get the `_EPROCESS` of the new process
4. Execute `.process /i <EPROCESS>` to tell the debugger to switch to the new processes context
5. Execute `g`

It seems inconvenient for such a basic task, and it gets even worse if we want to do that for processes that have not been executed yet. Let's automate this!

In 2017 Microsoft released a new version of the debugger - *WinDbg Preview*, which contained the new JS engine, time travel debugging tools, and a slick new UI. We're going to use the JS engine for our extension.

To start, we'll write the skeleton of our new extension script.

```jsx
"use strict";

// General functions
const execute = cmd => host.namespace.Debugger.Utility.Control.ExecuteCommand(cmd);
const log  = msg => host.diagnostics.debugLog(`${msg}\n`);

function initializeScript()
{
    return [
        new host.apiVersionSupport(1, 3),
        new host.functionAlias(breakOnProcess, "breakonprocess")];
}
```

First, a couple of wrapper functions to avoid typing the whole command every time we need to print or to execute a WinDgb command. Then defining the `initializeScript` function that'll be called every time the debugger loads our extension. This function returns an array that contains our API version and a function we'd like to export. This function can be later called in two ways: `!breakonprocess` or `dx @$breakonprocess()`. 

The process we're going to automate is as follows:

1. Set a breakpoint on `[nt!NtCreateUserProcess](https://processhacker.sourceforge.io/doc/ntpsapi_8h_source.html#l01322)` in order to monitor every new process on the debugee
2. Read the process file name and command line from the `_RTL_USER_PROCESS_PARAMETERS` argument
3. Compare those to the input we got from the user (i.e. the target process we want to break in)
4. If it matches, let `nt!NtCreateUserProcess` so our process will be created
5. Get the `HANDLE` to the new process
6. Switch to its context

Doing this, we'll be able to break inside the new process before it had a chance to execute code. From there we'll be able to set breakpoints freely and watch its every mov (pun intended).

WinDbg Javascript programming is not as straight forward as you might think, so let's go over the important parts of the code.

> 💡 You can play around the JS `host` class in the command prompt by using `dx @$scriptContents.host` (add `-v` for verbose output)

This is our short `breakOnProcess` function:

```jsx
function breakOnProcess(processName, processCommand){
    // Set a breakpoint
    var bp = host.namespace.Debugger.Utility.Control.SetBreakpointAtOffset('NtCreateUserProcess', 0, 'nt');
    var args = `"${processName}"`;
    if (processCommand)
    {
        args += `, "${processCommand}"`;
    }
    bp.Condition =  `@$scriptContents.handleProcessCreation(${args})`;
}
```

First, it sets a new breakpoint on our target function using `SetBreakPointAtOffset`, and then adds a condition to it which is our `handleProcessCreation` function. The user arguments are also parsed to allow breaking using the process name alone, or with the command line of it as well. Our "heavy lifting" will be at the function that handles every such breakpoint - `handleProcessCreation`.

`handleProcessCreation` starts by getting the user process parameters that we need by doing the following:

```jsx
var PROCESS_PARAM_PARAM = 8;            // parameter index of _RTL_USER_PROCESS_PARAMETERS
var NTCREATEUSERPROCSES_PARAM_NUM = 11  // number of parameters of nt!NtCreateUserProcess
​
var rsp = host.currentThread.Registers.User.rsp;​
var pUserProcessParams = host.memory.readMemoryValues(rsp, NTCREATEUSERPROCSES_PARAM_NUM+1, 8)[PROCESS_PARAM_PARAM+1];
var procParams = host.createTypedObject(pUserProcessParams, "nt", "_RTL_USER_PROCESS_PARAMETERS");
```

1. Define the number of parameters `nt!NtCreateUserProcess` has to be able to read those
2. Get the value of the stack pointer
3. Read all the parameters as 8-byte pointers from `rsp` (adding 1 to compensate for the return address being the first element on the stack) 
> Note that event though in Windows `x64` calling convention the first 4 arguments are passed in `rcx`, `rdx`, `r8` and `r9`, space is allocated on the stack for **all** parameters including the first 4. This is done in order to assist the called function in having the address of the argument list (`va_list`) and for debugging purposes (more on stack allocation [here](https://docs.microsoft.com/en-us/cpp/build/stack-usage?view=vs-2019)).
4. Cast our `_RTL_USER_PROCESS_PARAMETERS` to the appropriate type

Going forward, we only need to access the interesting field of the struct and compare those to the user input:

```jsx
var imagePathName = procParams.ImagePathName.toString().slice(1,-1).split("\\");
var fileName = imagePathName[imagePathName.length-1];

if (processName.toUpperCase() != fileName.toUpperCase()){return false;}
if (processCommand)
{
    let commandLinePresent = procParams.CommandLine.toString().toUpperCase().includes(processCommand.toUpperCase());
    if (!commandLinePresent){return false;}
}
```

Keep in mind that this function is a breakpoint condition, meaning that if it returns `false` the debugger will continue execution.

After we verified that the kernel is creating our desired process, we'll want to get into the process' context.

```jsx
var rcx = host.currentThread.Registers.User.rcx;

execute("pt")

var handle = host.memory.readMemoryValues(rcx, 1, 8);
var eprocess = host.currentProcess.Io.Handles[handle].Object.UnderlyingObject.targetLocation.address;
​
execute(`.process /i ${eprocess}`);
execute("g");
```

In line 1 we get `rcx`, which points to the new process' handle to be created (output parameter). Then we'll continue the execution until `nt!NtCreateUserProcess` returns (so our handle argument will be populated). Finally, we'll get our `_EPROCESS` address from our process handle int line 6, and then switch to the process' context. From here you can set a breakpoint inside the process using `bm`.

The full extension including usage instructions is available on my [GitHub](https://github.com/shakreiner/windbg-scripts/blob/master/BreakOnProcess/BreakOnProcess.js).

For further information about WinDbg's JS engine, see the [official documentation](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/javascript-debugger-scripting) and this great [post](https://doar-e.github.io/blog/2017/12/01/debugger-data-model/) by [@0vercl0k](https://twitter.com/0vercl0k). Happy debugging.