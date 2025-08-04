---
layout: post
title:  "Windows Fiber Local Storage is Useful but Slow"
---

Raymond Chen posted a blog back in 2019 stating that [Fibers aren’t useful for much any more; there’s just one corner of it that remains useful for a reason unrelated to fibers](https://devblogs.microsoft.com/oldnewthing/20191011-00/?p=102989).

The regular Windows thread-local storage APIs like TlsAlloc don't have a way an easy way to register a per-thread 
destructor to be able to clean up the memory. Blink gets around this with some crazy pragmas to manually insert a 
function to be called on each thread's exit, [thread_local_storage_win.cc](https://source.chromium.org/chromium/chromium/src/+/main:base/threading/thread_local_storage_win.cc;l=38-48).

```
// Thread Termination Callbacks.
// Windows doesn't support a per-thread destructor with its
// TLS primitives.  So, we build it manually by inserting a
// function to be called on each thread's exit.
// This magic is from http://www.codeproject.com/threads/tls.asp
// and it works for VC++ 7.0 and later.

// Force a reference to _tls_used to make the linker create the TLS directory
// if it's not already there.  (e.g. if __declspec(thread) is not used).
// Force a reference to p_thread_callback_base to prevent whole program
// optimization from discarding the variable.
```

Note that the linked forum post in the comment has bit-rotted, it can be viewed via the [Wayback Machine](https://web.archive.org/) [Thread Local Storage - The C++ Way by Roland Schwarz](https://web.archive.org/web/20070921204540/https://www.codeproject.com/threads/tls.asp).

Why does Blink go to the trouble of this when fiber-local storage has the useful built-in destruction callback? Unfortunately 
**FlsGetValue is around 2x slower than TlsGetValue on single threaded access**, which is a wide enough performance 
differential to make it worthwhile to deal with the extra complexity.

Hopefully one day TlsAlloc will get a destruction callback, or FlsGetValue will catch up to TlsGetValue in performance. 
Feels like a low-priority for Microsoft though given it hasn't happened within the past 20+ years since fiber-local 
storage launched, so you might want to use the less-useful but higher-performance TlsAlloc in your code.
