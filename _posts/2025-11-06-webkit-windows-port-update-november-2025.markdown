---
layout: post
title:  "WebKit Windows Port Update - November 2025"
---

Just finished attending my third WebKit Contributors meeting, and there were a
number of questions regarding the status of the Windows port. Seems worthwhile 
to post an update here, as I didn't give a lightning talk this year.

If you have any questions; or if you’d like to join my team and work on/with WebKit (remote US) - 
reach out via the WebKit Slack or email.

### Recap

I spent some time this year looking at differences between the Windows port and 
Linux / Mac for JSC performance. After [enabling every tier of JIT]({% post_url 
2024-10-07-every-jit-tier-enabled-jsc-windows %}) there were still a few things 
different about the Windows port that could explain why it didn't achieve 
similar JetStream scores to the other ports on the same hardware.

In chasing this I made some smaller changes, like [enabling LTO mode for release builds](https://github.com/WebKit/WebKit/pull/35220), and
[removing differences in the GC controller](https://github.com/WebKit/WebKit/pull/41688), and 
[enabling the periodic memory monitor](https://github.com/WebKit/WebKit/pull/49892).

I also [ported libpas to Windows](https://github.com/WebKit/WebKit/pull/41945),
WebKit's custom memory allocator. This was a larger change; and it took some iteration 
after landing to fix to improve the reliability due to fundamental differences in how 
committing memory works on Windows vs. Linux and Mac. David Chisnall has an excellent 
comment on Lobsters [explaining how Windows differs from *NIX kernels in how it deals 
with committing memory](https://lobste.rs/s/ssdxwt/closer_look_at_stack_guard_page#c_tgoeos). 
I wish I'd read this comment before starting the libpas work, it really solidifies how commit space works on Windows (and how it fundamentally differs from Linux / Mac).

To improve reliability of memory allocations, and reduce commit space in general, a 
number of changes were made:

* [Retry failed VirtualAlloc commit](https://github.com/WebKit/WebKit/pull/48995)
* [Use DiscardVirtualMemory in libpas](https://github.com/WebKit/WebKit/pull/49407)
* [timedwait abstime to interval conversion](https://github.com/WebKit/WebKit/pull/49484)
    * This bug meant we were sleeping ~forever, so didn't run the scavenger periodically. Oops!
* [Disable fast wasm memory](https://github.com/WebKit/WebKit/pull/49983)
    * This is unfortunate, but we should be able to make this work by reserving but not committing the 4GB redzone.
* [Decommit pages where possible in libpas](https://github.com/WebKit/WebKit/pull/50017)
    * This means we're the only platform doing symmetric decommit in production. 
    Unfortunately `pas_expendable_memory` will cause commit space growth over time as 
    we can't decommit in `pas_page_malloc_decommit_without_mprotect`. Perhaps this will 
    change due to changes for memory integrity enhancement.

I took a crack at [improving the Windows run loop](https://github.com/WebKit/WebKit/pull/42936)
as the current solution can't hit 60fps on requestAnimationFrame. It's likely a bottleneck for 
some JetStream tests also. Unfortunately this had to be reverted due to breaking 
[nested tasks on main thread](https://bugs.webkit.org/show_bug.cgi?id=291217), which Playwright
uses. I haven't got around to revisiting this, but it shouldn't be a big patch.

This year I also took over running the Windows build bots, both pre and post commit. 
These use the Azure worker hardware sponsored by Microsoft. On these I run the 
[webkitdev/buildbot-worker Docker containers](https://github.com/WebKitForWindows/docker-webkit-dev) 
via [hyper-v]({% post_url 2025-08-04-docker-hyperv-azure %}). It's a little awkward; perhaps we could migrate these to Azure Container Instances, 
but I suspect it'd be substantially more expensive for the same hardware.

Setting these up and ongoing maintenance has generated a steady stream of busywork. 
We've been struggling with reliability on win-tests for a number of reasons - first 
due to the work on libpas / commit space issues, but there's been ongoing reliability 
problems for a variety of reasons. After recent changes to increase the available disk space 
and deal with phantom locked DLLs when cleaning the WebKitBuild directory, the tests are 
currently green - but the price of green CI is eternal vigilance.

## Next Year

My hope is to attract new contributors to the Windows port. Short term it's
likely to happen via hiring additional people on my team to work on WebKit. 
Longer term ideally the port will improve such that it becomes a viable 
(and attractive) technology choice. With that in mind, I'll run through an 
unordered list of things I'd like to work on in the coming year.

There's a few smaller changes to make to improve performance - re-landing the 
runloop changes, fixing and re-enabling wasm fast memory, and switching libpas from
[fiber local storage]({% post_url 2025-08-04-fiber-local-storage-slow %}) to 
the faster thread local storage approach like Chromium does.

If Windows still has substantially different JetStream results vs. the Linux port 
after those changes, there's some work to be done to investigate causes and improve 
there. Hopefully we can close that gap this year.

I also want to make it easier for all WebKit contributors to keep the Windows port healthy. 
The main project in that area is to support [cross-compiling the Windows port from Linux](https://bugs.webkit.org/show_bug.cgi?id=282276). 
Most contributors have a [webkit-container-sdk](https://github.com/Igalia/webkit-container-sdk/) 
setup, but far fewer have a Windows setup. If those contributors could compile the port, 
and potentially run it via WINE / Proton - it'd make it much easier to build + test 
on Windows.

Additionally, we could spin up Linux-based buildbot workers for the build queues. It 
should be faster to cross-compile from Linux, and we can use the Azure hardware just for the 
win-tests queue. These changes would help reduce feedback time.

I'd like to audit, organize and garden the TestExpectations file for the Windows port. 
This has grown organically over time, and it'd be good to track and improve this
to reach similar test results to the WPE port.

There's work to keep on top of enabling new (and old) features in the Windows port - I'm aware 
of the Navigation API for one, but there's definitely more.

## Long term plans

There's things I'm interested in for the Windows port, that due to time constraints 
I'm personally less likely to get to within the next year.

It'd be good to enable `MEDIA_STREAM` and `WEBRTC` for the Windows port - but this project 
feels large and nebulous. GStreamer supports Windows on paper, so it's likely I'd go down 
that path for `MEDIA_STREAM` in pursuit of port alignment. I don't know how well maintained 
the Windows support for GStreamer is in practice, or how awkward it'd be to enable.

Improving our use of Skia in the Windows port would be good to look at. `COORDINATED_GRAPHICS` is 
disabled on Windows, which means the Windows port hasn't benefited as much as it could from 
Igalia's ongoing work in this area. Either we could enable `COORDINATED_GRAPHICS` for Windows, 
or figure out alternate ways to improve our render pipeline. Site Isolation is another motivating 
factor - sharing more of the render pipeline might mean we can share more of the changes to 
the compositor for Site Isolation.

Finally `WebGPU` would be interesting to pursue for the Windows and Linux ports. I'm interested 
in using Dawn for this, as it'd also open up the possibility of using the [Graphite backend 
for Skia](https://blog.chromium.org/2025/07/introducing-skia-graphite-chromes.html) which has a 
dependency on Dawn.

## Until Next Time

Reflecting on this years contributors meeting, I'm really excited about the future of the 
WebKit project. There's some truly innovative and cutting-edge work happening in WebKit, and 
I hope we can bring that great work to more people through supporting more platforms.