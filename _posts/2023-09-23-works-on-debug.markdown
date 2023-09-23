---
layout: post
title:  "Works on debug builds, crashes on release builds"
---
I'm (hopefully) getting closer to the finish line on [enabling Wasm on Windows](https://bugs.webkit.org/show_bug.cgi?id=222315) in WebKit. 
Recently I had a setback, when Yusuke Suzuki pointed out the critical flaw in the approach to free up r11 for use as the ws1 scratch register.
I had missed that r11 was used as a scratch register not only within the offlineasm compiler, but also with MacroAssemblerX86. Using r11 was
working and passed tests - but it'd also be very brittle going forward, as r11 could get clobbered by a new instruction, or through inserted 
JIT compiled code (once enabled on Windows).

I did have the tests passing using r11 though - which meant I could easily try switching from r11 to one of the available callee save 
registers on Windows and run the tests. I tried csr2 (rdi) as a replacement and ran into a problem - the tests passed on debug builds,
but were crashing on release builds.

After running the release build with a debugger attached, the problem became apparent. Here's the segfault:

![Segfault thrown on the vmEntryToWasm line](/docs/assets/images/works-on-debug-1.png)

Looking at the disassembly for that:

![Segfault thrown when referencing rdi after calling into the wasm interpreter](/docs/assets/images/works-on-debug-2.png)

The code was calling into the Wasm interpreter, and afterwards continuing to use the rdi register that was clobbered by the 
interpreter. I hadn't updated the code to save / restore the register appropriately yet.

I'd just gotten lucky with the register allocation in the debug build, that it hadn't used the rdi register.
