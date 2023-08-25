---
layout: post
title:  "Fast WebAssembly Memory"
---
WebAssembly (Wasm) is named to evoke "assembly language" but really it's more of a portable bytecode format, ran by a virtual machine.
There's multiple independent implementations - V8, SpiderMonkey and JavaScriptCore all have their own, and there's projects like 
[wasmtime](https://github.com/bytecodealliance/wasmtime) for running outside the browser.

If you're going to make a non-trivial Wasm program, you'll need to store some data outside of the stack. For that, Wasm has 
[memory instances](https://webassembly.github.io/spec/core/exec/runtime.html#memory-instances). Memory instances give Wasm programs access 
to a memory region, a vector of bytes which it can read and write to. Memory instances can allocate up to 4GB of memory, limited by the 
[memory instructions](https://webassembly.github.io/spec/core/syntax/instructions.html#syntax-memarg) that use a unsigned 32 bit integer 
address - the instructions can only address up to 4GB of memory. For example, to store at address 0 and offset 2:

```
(i32.store (i32.const 0) (i32.const 2))
```

Well, they can actually address up to 8GB if the offset parameter is abused. So what happens if a program tries to access some memory beyond 
the length of the memory instance? This results in a trap - immediately aborting execution of the Wasm program and it's up to the 
virtual machine / runtime to decide what to do next.

The simple implementation of memory instances would be for the runtime to perform a bounds check on the requested address prior to performing 
the load or store. This has a runtime cost, and one that you're paying on every load and store to memory which can add up fast. Browsers work 
around this through the use of virtual memory. This is an optimization applied to 64-bit platforms only, which is the vast majority of devices today. 
This optimisation exists in V8 (Chrome), JavaScriptCore (Safari) and SpiderMonkey (Firefox).

The broad idea is to (ab)use memory protection, to offload the bounds check to the operating system (which can offload it to hardware).
There's guard pages, or a "red zone" placed immediately after the memory instance region - explicitly allocating more virtual memory than requested 
by the memory instance. This means we always use 8GB of virtual address space, regardless of how much is usable by the Wasm program:

```
[ happy memory region ][  RED ZONE  ]
|  --- up to 4GB ---  |
|            --- 8GB ---            |
```

8GB sounds like a lot of virtual address space, though (most) 64-bit operating systems have around 128 TB of virtual address space. 
Also note that the red zone pages are never loaded into RAM, they exist only in the virtual address space and don't increase the 
resident set size (RSS). 
Lack of virtual address space is why this optimization is not applied to 32-bit operating systems, which only have around 4GB of virtual 
address space (and often less than that available to an individual process).

The operating system is used to set the guard pages as inaccessible. Now if the Wasm program tries to read or write beyond the 
length of the memory instance buffer in guard pages, it'll cause a [segmentation fault](https://en.wikipedia.org/wiki/Segmentation_fault). 
For a memory region with starting at `address` with `length` bytes, on POSIX platforms this would be a syscall to 
`mprotect(address + length, 8GB - length, PROT_NONE)`. The virtual machine can set up a custom signal handler using `sigaction` on POSIX 
platforms, and handle the `SIGSEGV` or `SIGBUS` to gracefully recover. 
By keeping track of the address regions of allocated Wasm memory instances, when a SIGSEGV / SIGBUS occurs the signal handler can 
identify that it occured due to a Wasm program, and handle it appropriately (terminating the Wasm program gracefully).

Recently I worked on implementing a [Signals interface for Windows](https://bugs.webkit.org/show_bug.cgi?id=259108) in WebKit, as the first step towards 
[enabling Wasm on Windows](https://bugs.webkit.org/show_bug.cgi?id=222315) in WebKit. This uses vectored exception handlers to handle the segmentation 
fault. I did notice (and later fix) a pretty gnarly bug while working on it, that when requesting the allocator to set the guard pages as inaccessible 
we'd [free the page instead](https://bugs.webkit.org/show_bug.cgi?id=260069). This is a particularly insidious bug because it'd look like it works as 
intended most of the time.

```
[ happy memory region ][  free mem  ]
|  --- up to 4GB ---  |
|            --- 8GB ---            |
```

If you try and read out of bounds shortly after initializing the memory, it'll fail because that page is unallocated. However if that freed virtual 
address space is **re-allocated by the same process**, you could perform out-of-bounds read / writes to it. I imagine it'd be hard to exploit in practice, 
you'd have to deliberately wait a while after allocation to attempt a read, and the Wasm program would be stopped after a single failed attempt.

Part of me wonders if it'd be better to run the Wasm runtime in a separate process, where process memory isolation could handle 
memory isolation without additional guard pages. This'd allow browsers to use the same strategy for 32-bit architectures, avoid additional 
virtual memory overhead, and could handle segfaults through inspecting process exit codes instead of custom signal handlers. I don't know 
if the IPC and context switching overhead for function calls would make it impractical in practice.

For further reading, check out the [V8 spec for this in Google Docs](https://docs.google.com/document/d/17y4kxuHFrVxAiuCP_FFtFA2HP5sNPsCD10KEx17Hz6M/), or 
the [really excellent block comment](https://hg.mozilla.org/mozilla-central/file/6089e7f0fa57a29c6d080f135f65e146c34457d8/js/src/wasm/WasmMemory.cpp#l68) 
that's in the SpiderMonkey implementation.
