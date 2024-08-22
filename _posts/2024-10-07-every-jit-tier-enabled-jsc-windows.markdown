---
layout: post
title:  "Every JIT tier enabled for JavaScriptCore on Windows"
---
As my first task at Pax Andromeda, I worked on (re)enabling 
all the JIT tiers for JavaScriptCore on Windows. This work has landed upstream in WebKit and 
JIT has been enabled by default on Windows. I'll go into the background leading up to this, 
what changed that presented this opportunity, the work that landed, and some of the problems 
that came up along the way. If you'd like to join my team and work on WebKit, 
we're hiring (remote US): reach out at [ian@pax-andromeda.com](mailto:ian@pax-andromeda.com).

### Background

JavaScriptCore has a number of different JIT compilation tiers for JavaScript, Wasm, and 
RegEx (Yet Another RegExp Runtime YARR). On the JavaScript side there's an interpreter 
plus three tiers of JIT, with increasing complexity of optimization (and hence time required to compile) - 
Baseline, Data Flow Graph (DFG) and Faster Than Light (FTL). On the Wasm side there's an interpreter
plus two tiers of JIT - Build Bytecode Quickly (BBQ) and Optimized Machine-code Generator (OMG). 
Yarr has an interpreter and a JIT, with a number of extra features.

On the Windows port historically there was support for the Baseline and DFG JITs for JavaScript, 
some support for YarrJIT (with a few features disabled), and for Wasm neither BBQ or OMG were enabled.
There was a bunch of code required for dealing with the differences between the Microsoft x64 calling 
convention and the System V ABI (used by macOS and Linux) when calling functions from assembly and 
C++, and vice-versa.

The interpreters for JavaScript and Wasm are written in a DSL called "offlineasm", which looks like 
assembly. There's a compiler for this language written in Ruby, to compile offlineasm into assembly 
for the target architectures supported by WebKit - x86, x86_64, ARM64, ARMv7, RISCV and more. For the 
Windows port there was a `X86_64_WIN` backend distinct from `X86_64`, and there were a number of places 
this needed to be used (as well as a lot of duplicate `X86_64 or X86_64_WIN` blocks). 
The register mappings for compiling offlineasm to x86_64 were different on Windows due to the calling convention 
differences (two extra callee saves, two fewer caller saves).

The difference in register mappings necessitated a different code path for `X86_64_WIN` when dealing with 
certain instructions like division. The Windows port also compiled offlineasm into a separate object file 
and later linked it with [MASM](https://en.wikipedia.org/wiki/Microsoft_Macro_Assembler) - which meant the 
offlineasm compiler had to output x86_64 assembly in Intel-syntax (all other ports used AT&T-syntax).
This linking step required differences in how labels were loaded compared with the other ports as well, 
all other ports generated a C++ header file with inline assembly.

### Opportunity Knocks

Two things changed the landscape in May 2024. The decision was made for WebKit to drop MSVC support and use clang-cl exclusively 
on Windows, [proposed by Yusuke Suzuke on the webkit-dev mailing list](https://lists.webkit.org/pipermail/webkit-dev/2024-April/032638.html).
Also the [Baseline JavaScript JIT was disabled on Windows](https://github.com/WebKit/WebKit/commit/da80566db803d381b14f4f7f9954d9baeb790741) 
as it had broken and no one had volunteered to fix it. This also disabled the Wasm interpreter, which (at the time) required JIT to generate 
entrypoint thunks.

Using clang-cl on Windows presented a big opportunity for the Windows port. The biggest feature present in clang-cl 
but missing from MSVC is the [sysv_abi function attribute](https://clang.llvm.org/docs/AttributeReference.html#sysv-abi).
This allows us to change the calling convention of a C++ function to the System V ABI. This small feature presented a huge opportunity - 
by annotating functions using this at the boundary layer between C++ and assembly (either for the offlineasm interpreter 
or JIT compiled assembly), we can use the same calling convention on Windows as the macOS and Linux ports. That removes the need for 
Windows-specific code for calling functions, and we can share the offlineasm x86_64 register mapping with macOS and Linux.

In addition we can build offlineasm on Windows using clang's inline assembly support, removing the Windows-specific offlineasm
build machinery. That eliminates the need to generate Intel-syntax assembly, and allows us to load labels using 
the same mechanism as the other ports.

This is a dream refactoring opportunity - it removes a bunch of Windows-specific hacks from the codebase, and presents the 
opportunity to reach feature parity with the other ports. With fewer Windows-specific hacks it should reduce the chance 
of breaking the Windows port in the future, reducing the maintenance burden. With JIT disabled by default - we're also free 
to make these changes without fear of breaking others along the way, we can delay turning JIT back on until we've reached 
parity.

### Making the Changes

Here's a list of the PRs that landed:

* [Use clang-cl for assembling offlineasm on Windows #28538](https://github.com/WebKit/WebKit/pull/28538)
* [Use SystemV ABI for C++ entrypoints for JS LLInt #28723](https://github.com/WebKit/WebKit/pull/28723)
* [Use SystemV ABI for Baseline JS JIT on Windows #29582](https://github.com/WebKit/WebKit/pull/29582)
* [Enable BUILTIN_FRAME_ADDRESS using _AddressOfReturnAddress() #30043](https://github.com/WebKit/WebKit/pull/30043)
* [Use SystemV ABI for Wasm LLInt on Windows #30116](https://github.com/WebKit/WebKit/pull/30116)
* [Use SystemV ABI for Yarr JIT on Windows #30197](https://github.com/WebKit/WebKit/pull/30197)
* [Use SystemV ABI for CSS Selector JIT on Windows #30249](https://github.com/WebKit/WebKit/pull/30249)
* [Use SystemV ABI for DFG JS JIT on Windows #30270](https://github.com/WebKit/WebKit/pull/30270)
* [Enable FTL on Windows #30580](https://github.com/WebKit/WebKit/pull/30580)
* [Test fixes for wasm llint on Windows #31018](https://github.com/WebKit/WebKit/pull/31018)
* [Enable Wasm BBQ JIT on Windows #31185](https://github.com/WebKit/WebKit/pull/31185)
* [[Win] Enable UNIFIED_AND_FREEZABLE_CONFIG_RECORD #31654](https://github.com/WebKit/WebKit/pull/31654)
* [Use X86_64 and C_LOOP OFFLINE_ASM_BACKEND on Windows #31788](https://github.com/WebKit/WebKit/pull/31788)
* [Enable Wasm OMG JIT on Windows #32204](https://github.com/WebKit/WebKit/pull/32204)
* [[Win] Fix Wasm Compilation From Web Worker #32602](https://github.com/WebKit/WebKit/pull/32602)
* [Enable JIT and FTL by default on the Windows Port #32972](https://github.com/WebKit/WebKit/pull/32972)

A few of these are worth further discussion.

In [pull request #29582](https://github.com/WebKit/WebKit/pull/29582) we hit an interesting difference of clang-cl in 
[OperationResult](https://github.com/WebKit/WebKit/pull/29582/files#diff-b9b2df1af192f0e91086eddd42c058ee0c15c499dcd4b5b20321069a82e4cb9e). 
Due to a more conservative [empty base class optimization on Windows](https://learn.microsoft.com/en-us/cpp/cpp/empty-bases?view=msvc-170), 
the presence of the parent struct was causing register spills which (surprisingly) added an extra register parameter
to compiled C++ JSC_DEFINE_JIT_OPERATION functions, which then broke when called from JIT generated assembly. This shows 
there's still differences in clang-cl vs. clang on other platforms where it follows MSVC-esque behaviour.

Determining how to shim `__builtin_frame_address(1)` on Windows for [pull request #30043](https://github.com/WebKit/WebKit/pull/30043/) 
took a lot of experimentation and work. On Windows `__builtin_frame_address(1)` returns the same value as `__builtin_frame_address(0)`, 
and `__builtin_frame_address(0)` points at the address after the local variables on the stack instead of the top of the current 
stack frame (and we can't easily inspect how many local variables there are). I had to dig through the LLVM IR intrinsic functions to find 
[addressofreturnaddress](https://releases.llvm.org/11.0.1/docs/LangRef.html#llvm-addressofreturnaddress-intrinsic), which pointed me at 
[_AddressOfReturnAddress()](https://learn.microsoft.com/en-us/cpp/intrinsics/addressofreturnaddress?view=msvc-170) to get a pointer to the 
top of the current stack frame (well, the end of the previous stack frame). Coupled with inline asm to clobber register RBP, it becomes 
the first regitser pushed to the stack and we can retrieve it to find the address of the previous stack frame. Took a while to figure out
what was ultimately less than ten lines of code.

In [pull request #30580](https://github.com/WebKit/WebKit/pull/30580) we hit an interesting problem in one of the tests, which contained 
a [test oracle function](https://en.wikipedia.org/wiki/Test_oracle) implementing countLeadingZero. It's short enough that I'll embed it 
here:

```
template<typename IntegerType>
static unsigned countLeadingZero(IntegerType value)
{
    unsigned bitCount = sizeof(IntegerType) * 8;
    if (!value)
        return bitCount;

    unsigned counter = 0;
    while (!(static_cast<uint64_t>(value) & (1l << (bitCount - 1)))) {
        value <<= 1;
        ++counter;
    }
    return counter;
}
```

The implementation left shifted 1 to the most significant bit of the integer type under test `(1l << (bitCount - 1))`, and then would count
how many times it needed to left shift the value until the value had a 1 in that position to calculate the number of leading zeroes. The issue 
here is using a long 1L - for debug builds the compiler was choosing a 32-bit signed long, and on release builds it was choosing a 64-bit signed long.
This gave us an off-by-one error on release builds as if you left-shift a signed number so that the sign bit is affected, the result is undefined 
([source](https://learn.microsoft.com/en-us/cpp/cpp/left-shift-and-right-shift-operators-input-and-output?view=msvc-170)). 
I'm surprised this works on the other platforms, switching to an 64-bit unsigned long (1ULL) fixed the issue.

In terms of the good - the PR to [enable Yarr JIT on Windows #30197](https://github.com/WebKit/WebKit/pull/30197) was very straight forward, removed 
a lot of Windows specific hacks, and enabled some regex optimizations that were previously not available on the Windows platform. The offlineasm code 
got a lot simpler after switching to use clang-cl for assembling offlineasm and enabling `UNIFIED_AND_FREEZABLE_CONFIG_RECORD`, allowing us to remove 
the `X86_64_WIN` and `C_LOOP_WIN` offlineasm backends in [PR #31788](https://github.com/WebKit/WebKit/pull/31788). This means we no longer have to remember 
to add `X86_64 or X86_64_WIN` on every branch of offlineasm code to support the Windows platform.

### Conclusion and Next Steps

This work wouldn't have been possible without [Yusuke Suzuki](https://github.com/Constellation), who had the 
[great idea](https://x.com/filpizlo/status/1810729040342569066) of using sysv_abi in the first place and reviewed the bulk of this work. Thanks Yusuke!

This work has already made it's way into bun with the FTL JIT landing in [bun v1.1.19](https://bun.sh/blog/bun-v1.1.19#javascript-gets-faster-on-windows)
and the wasm OMG JIT landing in [bun v1.1.25](https://bun.sh/blog/bun-v1.1.25#faster-webassembly-on-windows-with-omgjit). These changes will make their way 
into Playwright as soon as they update their version of WebKit past [my commit enabling JIT by default](https://commits.webkit.org/284574@main). This should 
resolve a few Playwright bugs related to WebAssembly:

* [PDFIUM is not working in Webkit(playwright)](https://github.com/bblanchon/pdfium-binaries/issues/158)
* [[Bug]\: Webkit 17.4 (playwright build v1967) on Windows fails to load flutter page](https://github.com/microsoft/playwright/issues/29693)
* [[Bug]\: Button click return errors using WebKit](https://github.com/microsoft/playwright/issues/31637)
* [[BUG] Windows WebKit doesn't have WebAssembly](https://github.com/microsoft/playwright/issues/2876)

I'll be continuing to push the Windows port forward, with libpas (WebKit's custom allocator) being the next thing I'm planning to take a look at. There's also a few 
changes required to close the performance gap between the macOS and Windows ports on x86_64 - enabling Link Time Optimization, Profile Guided Optimization, and 
opportunistic GC scheduling.
