---
layout: post
title:  "Compiler Explorer"
date:   2023-08-09 17:35:58 -0400
categories: tools compilers
---
Recently I worked on a couple of WebKit bugs where being able to inspect the compiled assembly output was useful. In 
situations like these, [compiler explorer](https://godbolt.org/) is a good tool to have in your toolbox.

The [first bug](https://bugs.webkit.org/show_bug.cgi?id=259429) was around empty switch statements in generated code. The
[opcode_generator.rb](https://github.com/WebKit/WebKit/blob/main/Source/JavaScriptCore/b3/air/opcode_generator.rb) ruby script consumes the 
[AirOpcode.opcodes](https://github.com/WebKit/WebKit/blob/main/Source/JavaScriptCore/b3/air/AirOpcode.opcodes) file and generates a 
AirOpcodeGenerated.h header file. AirOpcodeGenerated.h contained code that looked like this:

```
switch (argIndex) {
    default:
      break;
}
```

Which is effectively a no-op. This code [compiles in clang](https://godbolt.org/z/8TrYMsPEs) generating a jmp instruction to a 
label on the next line. This code [doesn't compile in MSVC](https://godbolt.org/z/cePeKGK63) with flags /Wall /WX, because /Wall 
generates a warning for a switch statement with no case labels, and /WX treats all compiler warnings as errors:

```
<source>(6): error C2220: the following warning is treated as an error
<source>(6): warning C4065: switch statement contains 'default' but no 'case' labels
```

The compiler explorer was useful here to see that there was a very slight benefit to other platforms by making this change for 
Windows - removing a no-op jmp instruction.

The [second bug](https://bugs.webkit.org/show_bug.cgi?id=259108) was around implementing a Signals interface for Windows. 
While Windows has [interrupt signal handling](https://learn.microsoft.com/en-us/cpp/c-runtime-library/reference/signal), 
I couldn't see a way to be able to recover from a `SIGSEGV` and continue execution at a specified point. Luckily I found a 
suggestion from Yusuke Suzuki on an [earlier related bug](https://bugs.webkit.org/show_bug.cgi?id=258556):

> The same interface needs to be implemented on Windows with vectored exception handlers.

So I did that, however when [writing a test](https://github.com/WebKit/WebKit/pull/16396/files#diff-c9870ae615fceab8e643df562a8b2d1d00271d141a722999cad7e7cf116ba785R79-R111) 
for it I wasn't sure what to set the program counter to when returning, such that I advanced the program counter past the code 
that generated the AccessFault. I might've been able to use 
[Labels as Values](https://gcc.gnu.org/onlinedocs/gcc/Labels-as-Values.html) if WebKit was built using clang on all platforms,
but MSVC is used on Windows ([for now](https://bugs.webkit.org/show_bug.cgi?id=171618)). So I used compiler explorer to 
determine how much to add to the program counter. It was also handy when my code to generate an access fault wasn't working - 
the compiler was optimizing away my unused variable, which was obvious once I ran it through compiler explorer.

I haven't used it for languages other than C and C++, though it looks like it supports a bunch of other languages. Might be 
a useful addition to your toolbox.