---
layout: post
title:  "WebAssembly on WebKit for Windows"
---

Recently I finished [enabling Wasm on Windows](https://bugs.webkit.org/show_bug.cgi?id=222315) in WebKit. This built 
on previous work to support [fast webassembly memory]({% post_url 2023-08-25-fast-webassembly-memory %}) and I've written 
a little previously about this working for [debug builds but failing on release]({% post_url 2023-09-23-works-on-debug %}).
There were some fun challenges in getting this to work which I'll dig into below.

I'm currently exploring ways to fund my work on continuing to improve the Windows port for WebKit. I'm also thinking about 
exploring other avenues for improving web engine diversity (and web platform funding diversity). Get in touch if either of
those sound interesting to you, I'd love to chat.

### Background

The WebAssembly low level interpreter (LLInt) in WebKit is written in a DSL called "offlineasm", which looks like assembly. There's a compiler
for this language written in Ruby, to compile offlineasm into assembly for the target architectures supported by WebKit - x86, x86_64, ARM64, ARMv7, RISCV and more!

The LLInt is quite slow at running code, but it's able to start executing quickly. For small, short-lived WebAssembly code blocks startup time 
dominates the total time taken so it's a good option to have. For code that's executed many times, WebKit wants to tier up into a JIT quickly - 
either the BBQ or OMG JIT. There's a good talk about the LLInt from WebAssembly Summit 2020 that's worth a watch if you're interested in learning more, 
[Tadeu Zagallo — JavaScriptCore's new WebAssembly interpreter](https://www.youtube.com/watch?v=1v4wPoMskfo).

There's work happening on a new WebAssembly interpreter at the moment called the In Place Interpreter (IPInt), exploring ideas from the paper 
[A fast in-place interpreter for WebAssembly](https://dl.acm.org/doi/abs/10.1145/3563311) by [Dr. Ben Titzer](https://s3d.cmu.edu/people/core-faculty/titzer-ben.html).
There's also work happening at the moment to remove the JIT generated thunks from the LLInt, which'll allow running the LLInt without any JIT, 
opening up potential for using Wasm in lockdown mode on Safari, or in ports that disable JIT tiers (e.g. Playstation).

Windows has a few differences to macOS and Linux which are relevant for the LLInt. I think the build is still using ML64 despite the great work
to move the Windows port to build with clang instead of MSVC. This means offlineasm needs to generate x64 assembly code in [Intel syntax instead 
of AT&T](https://en.wikipedia.org/wiki/X86_assembly_language#Syntax) for the Windows port. The
[calling convention](https://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions) is different, with other platforms following 
the System V AMD64 ABI which is different from the Microsoft x64 calling convention - one big difference being two callee-save registers are caller-save 
on Windows. Dynamic libraries follow a different format (DLLs instead of Shared Objects). There's also a long tail of unix things that aren't there, like 
signals which we previously tackled for [fast webassembly memory]({% post_url 2023-08-25-fast-webassembly-memory %}).

### First steps - getting it to compile

Enable the compile flag in the cmake config and see what breaks! First up was fixing some compilation errors where MSVC wouldn't accept some C++ that clang would. 
This work was started prior to the switch to clang, though there might still be differences between what clang and clang-cl accept.

After that the `WebAssembly.asm` offlineasm file wouldn't compile as ws1 was unmapped on Windows. This problem went through some iteration, to start with 
I used r14 as a placeholder to get it to compile. More on this later.

There were various operators in offlineasm which needed modification to add Intel-syntax output. Thankfully many (all?) of these surfaced as errors in the assembler, 
because instructions took two different register types and the assembler would validate they were correct. For example - the [cvttss2si](https://www.felixcloutier.com/x86/cvttss2si)
instruction takes a floating point register and a general purpose register, so if you pass them in the wrong order the assembler will fail.

A hack needed to be put in place to attempt to load a label from a dynamic library. LLInt references a global configuration struct from the WTF (Web Template Framework) library.

### Segfault #1

The label referring to the location of the global configuration struct successfully compiled, but at runtime was pointing at garbage. To resolve this, I ended up 
copying the struct in C++ in JavaScriptCore, and publishing a label to that so offlineasm didn't need to load labels from a DLL on Windows.

This one took a couple of weeks to track down - hampered by time take to set up debugging in Visual Studio, long recompile times, and a number of dead ends.

### Segfault #2

When looping over locals on the stack to zero out the values, we'd underflow an integer and try and zero memory out of bounds. This was fixed by updating the
NumberOfWasmArgumentJSRs constant to reflect the NUMBER_OF_ARGUMENT_REGISTERS on Windows (caused by the different calling convention).

### C++ Assertion Failure

There's a concept of Wasm "slow path" functions - code that's defined in C++ which offlineasm calls into for various operations. These are exported C functions which 
are named "slow_path_wasm_##name", and take three arguments - the call frame, the program counter, and the Wasm Instance.

When we were calling the "slow_path_wasm_call", we'd attempt to cast the passed program counter bytecode to a WasmCall and it'd fail the assertion that it is a WasmCall.

I tracked this down to using the offlineasm cCall4 macro (designed for 4 arguments) to make this function call, instead of the cCall3 macro (designed for 3 arguments). These 
macros are identical on non-Windows platforms, but under the Windows calling convention they are passed differently so aren't interchangeable.

### Segfault #3

Inside the slowPathForWasmCall macro we were saving the wasmInstance pointer into the PB register and restoring it later, but in between we'd call reloadMemoryRegistersForInstance
(which loads the memoryBase and boundsCheckingSize for the associated WebAssembly Memory object):

```
# Preserve the current instance
move wasmInstance, PB

# ... a bunch of code, including
reloadMemoryRegistersFromInstance(targetWasmInstance, wa0, wa1)

# ... later on we restore the instance from PB
move PB, wasmInstance
```

On Windows, both PB and boundsCheckingSize used the csr4 register (r13 on Windows). This meant the wasmInstance pointer was clobbered when 
we called reloadMemoryRegistersFromInstance.

I switched the WebAssembly memory registers (memoryBase, boundsCheckingSize) to use (csr5, csr6) to resolve this.

### Divide by zero

The x86_64 div and idiv instructions work by setting up the dividend in the edx:eax registers, and then dividing those by the passed register. The result is stored in eax, remainder is stored in edx.

However the offlineasm register mapping is different on Windows, with edx mapping to t2 instead of t1 in offlineasm.

This broke the division and remainder WebAssembly operations (there's signed and unsigned opcodes for both 32 bit and 64 bit).

There’s an open bug for adding static asserts to ensure the register mapping matches the assumptions made here, which was raised when the code was originally written.

### Scratch register problems

After starting with r14 as a placeholder, I switched to using r11. This register was reserved for usage by the compiler, and I thought by changing the offlineasm
instructions that used it to instead take it as a parameter, I could return control of r11 to the programmer. However it was pointed out [in the review](https://github.com/WebKit/WebKit/pull/17231)
that MacroAssemblerX86_64 also used it behind the scenes, so that approach wasn't viable. It seemed to work for the tests I ran but it'd be brittle moving forward
as Windows would differ from other x86_64 ports in this regard, and future changes in the MacroAssembler could break the LLInt on Windows only.

I switched to using one of the remaining callee-save registers, and saved / restored it as appropriate.

### Wrapping up

The review cycle on this PR took a while - the JavaScriptCore team is relatively small and I created the review in a particularly busy time for them. I gave a talk 
at the WebKit Contributors Meeting about this work, and was able to meet with most of the people who had been helping and encouraging me along the way. It was a great
way to wrap up my final week at the Recurse Center

About half way through this work I upgraded my computer as the 60-75 minute full build times became unbearable. Compute has never been cheaper, and I was able to 
pick up an Intel Core i9-12900K, Motherboard and 32GB of RAM bundle from Microcenter for $400. That brought my full build times down to 15-20 minutes (faster if I 
don't use the computer while it's building), which is a major improvement but still slower than I'd like. I should've upgraded weeks earlier, I wasted a lot of time 
waiting for builds.

I found a [concurrency bug in WebAssembly LLInt compilation](https://bugs.webkit.org/show_bug.cgi?id=263965) when testing this work. This is likely a bug that's present 
for all platforms, so it's good that this work was able to surface that.

Repeating what I put at the top - I'm currently exploring ways to fund my work on continuing to improve the Windows port for WebKit. I'm also thinking about 
exploring other avenues for improving web engine diversity (and web platform funding diversity). Get in touch if either of those sound interesting to you, I'd love to chat.
