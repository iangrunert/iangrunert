---
layout: post
title:  "Writing a JS Engine using Claude Code"
---

Over the past 2 months, I've been working on a JavaScript interpreter using 
Claude Code. It's been an interesting side project, to learn about the weird 
corners of the ECMAScript spec while figuring out the edges of what Claude Opus
4.6 is capable of.

I just switched the repo to public, it's about 58000 lines of Go (plus libregexp)
and currently passes 97.4% (43895 / 45087) of the subset of Test262 that I'm 
targeting; which represents 99% (44623 / 45087) of the Test262 tests that QuickJS 
passes. I think I could hit parity with QuickJS given a couple more weeks, but 
I think I've learnt enough from this project that it's worth writing about it.

This is a great project for Claude Code. There's an existing extensive test 
suite in Test262. There's also multiple implementations which can serve as 
oracle programs to generate additional ad-hoc [expect tests](https://blog.janestreet.com/the-joy-of-expect-tests/) 
during development. Claude Opus 4.6 knows how to write little JavaScript 
programs, can debug issues in little JavaScript programs so has some knowledge 
of the spec, and also can write Go. There's a spec it can read, and there's 
multiple existing implementations it can investigate for implementation ideas.

## JS Engine Overview

[Gojiscript](https://github.com/iangrunert/gojiscript/) is a stack-based bytecode interpreter, 
there's a couple of novel features but overall there shouldn't be any major surprises if you're 
familiar with QuickJS or LibJS internals. It implements nearly all of ES2024 — classes, generators, 
async/await, modules, Proxy, BigInt, iterators, WeakRef, and more — plus some ES2025 features like Iterator helpers.

It parses into an AST, performs a single pass compilation into bytecode, and 
performs no optimization passes. It gets a great garbage collector out of the box 
from Go, but it's more difficult to get compact memory layouts compared to C.

It uses [WTF-8](https://wtf-8.codeberg.page/) encoding for its strings, which 
makes it easier to support UTF-16 codepoints. There are still some sharp edges 
caused by text encoding, given the spec requires UTF-16.

Inspired by this blog from Planetscale, [Faster interpreters in Go: Catching up with C++](https://planetscale.com/blog/faster-interpreters-in-go-catching-up-with-cpp) 
by Vicent Martí, the bytecode is a slice of function pointers to each instruction. 
I'd be curious to experiment with switching to a traditional bytecode interpreter
loop (in pure Go, this'd be a giant switch statement) to do an apples-to-apples 
performance comparison. There's other downsides too - it's more difficult to 
cache compiled bytecode (currently unimplemented). We can't serialize the 
function pointers - we'd have to serialize a command list to regenerate the 
functions at startup.

Some performance work was done along the way to make Test262 run quicker, and 
to make a small set of V8 benchmarks at least run; but not a lot of work was done.
It's still quite slow. I think there's a lot of opportunity for improving 
performance through optimization passes, and fused instructions should be 
effectively zero cost as we're not adding extra branches to a giant switch 
statement.

There's only one Go dependency beyond the stdlib - we use 
`golang.org/x/text/unicode/norm` for String.normalize. 

There's one C dependency via cgo - we're using libregexp from QuickJS as the 
regexp engine. I did try to get an agent to swap it out, but it wasn't given 
sufficient guardrails to make good progress.

Currently skipping tests for Temporal, Intl, and a handful of tests for unicode
as Go 1.26 only supports up to Unicode 15.0.0. This will be resolved in Go 1.27 
https://github.com/golang/go/issues/77266.

Overall I'm relatively happy with the code. As with any codebase of sufficient 
size there's parts I'd like to take a second crack at (and I might do yet!).

## Development Process

At first, I was working with a single Claude Code terminal window. After 
spending ~2 weeks getting more familiar with the process, I used Claude Code 
to write a little orchestrator. A rough sketch:

* The whole process is driven from a Trello board.
* Dragging a card from "Backlog" to "Planning" creates a git worktree / branch, and starts a Claude Code CLI to generate a plan based on the prompt in the card's title / description.
* Claude writes the plan to PLAN.md, the orchestrator commits and pushes to the branch so I can review it on GitHub. Moves the card to "Planned". Comments on the card with the summary output from Claude Code, and adds a link to the PLAN.md on Github to the card.
* I can comment on the card to kick it back into "Planning", allowing me to iterate on the plan until I'm happy with it.
* Dragging a card to "Implementing" creates a fresh git worktree / branch, and starts a Claude Code CLI to implement the plan.
* After Claude completes the task and commits, the card is moved to "Review", and comments with a link to the diff.
* I can comment on the card to kick it back into "Implementing".
* Dragging a card to "Merge" merges the branch, and comments on the card with a link to the commit on main.

This was a substantial productivity improvement over a single Claude Code 
terminal window, for a few reasons. I wasn't staring at the terminal while 
Claude was performing the work. I'm mostly bottlenecked on my own capacity to 
review plans and code, but I can usually manage to get 2-5 concurrent agents 
depending on the complexity of the plans / reviews. Most of the time I can nail 
down the details during planning such that there aren't many surprises in the 
implementation.

Iterating on plans to find simpler solutions, and thinking about how to improve 
code during review is a much more productive use of my time than watching Claude 
doing its thing.

Often it'd be easier to kick off a follow-up card rather than attempting to 
iterate on the implementation in the original context. That gave the agent a 
fresh context window and I got to iterate on the improvement plans before 
kicking off the implementation.

I used Test262 as a ratchet, writing expected failures to a text file and used 
a flag to the test runner to remove tests from the expected failures file if 
they're now passing. This gave the agent feedback when it introduced unexpected 
regressions. Occasionally it'd get confused and edit the file directly, though 
you could imagine keeping the expected failures somewhere it couldn't touch.

## Final Thoughts

There's some places where Test262 could use improvements. There's a number of 
tests that rely on the `with` statement or `eval` while not actually testing 
those constructs. There's also a number of tests that run only in sloppy mode, 
such that if you wanted to build a strict-mode only engine you'd be missing 
out on some test coverage. This isn't surprising given the primary contributions 
come from the browser engine development teams.

There's still significant advantage to a human in the loop with the current 
state of the art models. On many occasions I was able to identify simpler and 
neater solutions compared to the model's first ideas. Without the friction 
associated with typing code, there isn't a traditional forcing function to 
pause and think about whether there's a better way.

A lot of the work is right-sizing tasks such that they're big enough to be 
genuinely useful compared writing the code yourself, but small enough that the 
model can complete it (ideally before compaction).

"Build one to throw away" always felt hollow because it was far too tempting 
to keep it once it was already built. For certain tasks, I wonder if there will 
be a resurgence of this practice as "building one" is much faster than it ever 
was.

There's no way I would've made this much progress in 2 months without Claude 
Code. Here's where it stands today when compared to QuickJS on Test262:

```
Loaded 1192 expected failures
Loaded 9060 skipped tests (QuickJS failures)
Found 52861 test files

=== Test262 Results ===
Total:             45087
Passed:            44623 (99.0%)
Failed:            464 (1.0%)
Expected failures: 0
QJS-compat skips:  728
Feature skips:     7774 (Temporal/Intl/Unicode>15.0.0)
Timeouts:          0
Panics:            0
Compile errors:    0

=== Pass Rates by Top-Level Directory ===
annexB: 99.2% (1077/1086)
built-ins: 99.1% (18643/18804)
harness: 100.0% (116/116)
language: 99.5% (23482/23600)
staging: 88.1% (1305/1481)
```