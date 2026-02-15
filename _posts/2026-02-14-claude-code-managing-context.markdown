---
layout: post
title:  "A trick for Claude Code context windows"
---

A big part of the job of working with Claude Code currently is task breakdown. 
This serves two purposes:

1. Agents are a net speed-up when you can "right-size" a task - something that's 
large enough that it's faster than doing it yourself, but small enough that it'll 
complete the task successfully. It's less likely to succeed if it has to compact 
it's context window (and some claim even before then it's performance degrades).
2. While I'm still reviewing all the code being produced, I'd prefer work to be 
completed in logical, iterative steps which I can review independently. Not 
dissimilar to managing the context window - the quality of my review degrades the 
more tokens I have to read in a single review unit.

While some are using tools like Gas Town with a "mayor" agent performing this kind of 
task breakdown, I'm getting good results by taking on that role for now. It's 
been useful to identify weird choices, push towards simpler more general solutions, 
and I often spot things which I want to improve in subsequent tasks in review.

Sometimes Claude comes up with a plan that looks great, but unfortunately there's 
enough phases to the plan that it'll exceeds the context window. Maybe you try it 
anyway and get dissapointing results. Here's a trick I tried with some success 
today. While in Plan mode, I asked:

> Can we plan to execute each phase in a separate agent session? This would help manage the context window.

and in response, it modified the plan:

> Each phase is designed to be executed in a separate agent session to manage context window limits. Each phase section includes the files to read, what to do, and how to validate — so the agent can work self-contained without needing prior session context.

The plan included Design Principles, an Architecture Overview, and Implementation 
Order. For each of the 16 phases it included the following:

* Session scope
* Files to Read
* What to Do
* How to Validate
* Commit (including commit message)

I didn't watch it execute this plan as I use Claude in headless mode, but it kicked 
off agent tasks for each phase. The top-level session never compacted. It was able 
to complete a very complicated refactor successfully. The end result was a large 
change split into 16 logical commits - so I could review each individually and feel 
relatively confident in the bigger picture.

It did take longer than usual in wall clock time - 5 hours total to execute this plan. 
Due to the extra time it's probably not worth doing for every change, but I'll likely 
try it again next time I have a big plan to implement.