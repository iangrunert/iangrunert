---
layout: post
title:  "Web Assembly Memory"
---
WebAssembly (Wasm) is named to evoke "assembly language" but really it's more of a portable bytecode format, ran by a virtual machine.
There's multiple independent implementations - V8, SpiderMonkey and JavaScriptCore all have their own, and there's projects like 
[wasmtime](https://github.com/bytecodealliance/wasmtime) for running outside the browser.

If you're going to make a non-trivial program, you'll need to store some data outside of the stack. For that, Wasm has 
[memory instances](https://webassembly.github.io/spec/core/exec/runtime.html#memory-instances). Memory instances give Wasm programs access 
to a memory region, a vector of bytes which it can read and write to. Memory instances can allocate up to 4GB of memory, limited by the 
[memory instructions](https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-memarg) that use a unsigned 32 bit integer 
offset - the instructions can only address up to 4GB of memory.

Well, they can address up to 8GB if the offset pointer is abused. So what happens if a program tries to access some memory beyond 
the length of the memory instance? This results in a trap - immediately aborting execution of the Wasm program and it's up to the 
virtual machine / runtime to decide what to do next.

The simple implementation of memory instances would be for the runtime to perform a bounds check on the requested address prior to performing 
the load or store. This has a runtime cost, and one that you're paying on every load and store to memory which can add up fast. Browsers work 
around this through the use of virtual memory. You can read the corresponding [V8 spec in Google Docs](https://docs.google.com/document/d/17y4kxuHFrVxAiuCP_FFtFA2HP5sNPsCD10KEx17Hz6M/), 
and WebKit has a similar implementation in [WasmMemory](https://github.com/WebKit/WebKit/blob/main/Source/JavaScriptCore/wasm/WasmMemory.cpp).
This is an optimization applied to 64-bit platforms only, which is the vast majority of devices today.

The broad idea is to (ab)use memory protection, to offload the bounds check to the operating system (which can offload it to hardware).
There's guard pages, or a "red zone" placed immediately after the memory instance region.

```
[ happy memory region ][  RED ZONE  ]
|  --- up to 4GB ---  |
|            --- 8GB ---            |
```

8GB sounds like a lot of virtual address space, though (most) 64-bit operating systems have around 128 TB of virtual address space. Virtual 
address space is why this optimization is not applied to 32-bit operating systems, which have around 4GB of virtual address space (and often) 
less than that available to an individual program.

Next, the operating system is used to set the guard pages as inaccessible. For a memory region with `length` starting at `address`, on POSIX platforms
this would be a syscall to `mprotect(address + length, 8GB - length, PROT_NONE)`. Now if the Wasm program tries to read beyond the length of the 
memory instance buffer, it'll cause a [segmentation fault](https://en.wikipedia.org/wiki/Segmentation_fault). The virtual machine can set up a 
custom signal handler using `sigaction` on posix platforms, and handle the `SIGSEGV` or `SIGBUS` to gracefully recover.

The recovery requires keeping track of the address regions of Wasm memory instances, so when a SIGSEGV / SIGBUS occurs the signal handler can 
identify that it occured due to a Wasm program, and handle it appropriately.

Recently I worked on implementing a [Signals interface for Windows](https://bugs.webkit.org/show_bug.cgi?id=259108) in WebKit, as the first step towards 
[enabling Wasm on Windows](https://bugs.webkit.org/show_bug.cgi?id=222315) in WebKit. This uses vectored exception handlers to handle the segmentation 
fault. I did notice (and later fix) a pretty gnarly bug while working on it, that when requesting the allocator to set the guard pages as inaccessible 
we'd [free the page instead](https://bugs.webkit.org/show_bug.cgi?id=260069). This is a particularly insidious bug because it'd look like it works as 
intended most of the time. If you try and read out of bounds it's likely to fail, unless that freed virtual address space is re-allocated by the 
same process such that it's readable / writeable.

I do wonder if browsers could run Wasm programs in a separate process. Out of bound memory accesses would be handled by the operating system, without 
needing additional guard pages. The downside would be requiring IPC for function calls into and out of Wasm, which could be a significant 
enough performance hit to limit Wasm's usefulness.
