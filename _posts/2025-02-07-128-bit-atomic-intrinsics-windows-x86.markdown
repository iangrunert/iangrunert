---
layout: post
title:  "128 bit atomic intrinsics on Windows"
---

While working on [enabling libpas on the Windows port for WebKit](https://bugs.webkit.org/show_bug.cgi?id=283311), I ran into some linking errors:

```
lld-link: error: undefined symbol: __atomic_compare_exchange
lld-link: error: undefined symbol: __atomic_store
lld-link: error: undefined symbol: __atomic_load
```

These corresponded to clang's builtin 128-bit atomic intrinsic functions operating on a 
struct with two 64 bit pointers - `__c11_atomic_store`, `__c11_atomic_compare_exchange_weak`, 
`__c11_atomic_compare_exchange_strong`, and `__c11_atomic_load`.

After quite the run-around confirming that clang's C runtime library was linked, 
I tried replacing the c11 intrinsics with the Microsoft [_InterlockedCompareExchange128](https://learn.microsoft.com/en-us/cpp/intrinsics/interlockedcompareexchange128?view=msvc-170) 
intrinsic instead - which are also supported by clang.

The compiler gave me a much, much more useful error message after trying this:

```
error: '_InterlockedCompareExchange128' needs target feature cx16
```

Looks like clang doesn't assume the processor has the CX16 feature, which 
introduces the cmpxchg16b instruction. After adding the compile option:

```
add_compile_options(-mcx16)
```

The original c11 atomic intrinsics on 128 bit values work - the compiler can output the CMPXCHG16B 
instruction instead of attempting to call a function that it can't link. I don't have to rewrite them 
to use the Microsoft intrinsics.

This is a pretty frustrating default. The last processor without the CMPXCHG16B instruction 
was released in 2006 so far as I can tell. Windows 8.1 64-bit had a hard requirement on the CMPXCHG16B 
instruction, and that was released in 2013 (and is no longer supported as of 2023). I think clang should assume 
we're building for modern hardware and supported operating systems, and make old hardware require a compiler flag. 
Seems insane that everyone has to remind the compiler that we're running on hardware built in the last 15(+) years. 

Specifically - I think -march=x86-64 should become an alias, x86-64-v1 should be added for the old 
processors, and x86-64 should point at x86-64-v2 (or maybe even x86-64-v3). Failing that - this is a reminder that you 
probably want to compile for x86-64-v2, not x86-64.