+++
title = "Rust Debugging Progress Report May 2026"
date = 2026-05-11

[taxonomies]
tags=["programming", "rust", "debugging"]
+++

Since my last post, my work regarding Rust debugging as slowed a bit. The fixes to the visualizers assuaged my annoyance enough to finish some side projects. Crafting Interpreters, and a rewrite-it-in-rust of a python tool that turns Kindle Scribe notebook files into SVGs, and some other silly stuff. The break was a fat reminder of how weak the debugging experience still is, and it has worked wonders to refresh my spite-based motivation. Before I hang my "It doesn't have to be this way. We make our own tools." poster back up, I want to do a bit of a brain dump.

<!--more-->

I've also kept an eye on the general debugging ecosystem, and there are some pretty interesting developments that I want to go over.

# Rustc Docs

The debug info docs in the rustc dev guide were pretty minimal. They amounted to, more or less, a transcription of a talk from quite a few years ago. The unfortunate result is that even the compiler team, who are in charge of most debug info changes, don't have a great understanding of how these systems work or how to improve them. The testing situation (which I'll talk about more in the next section) is pretty dire as well, so nobody feels confident making or reviewing changes.

To be clear, I don't blame anyone for not knowing about this stuff. It's a big stack, it covers multiple big complicated projects, and the non-Rust projects aren't always doing a great job making information about their systems available. Not everyone can disappear into the mountains for a few months to untangle LLDB's twisted maze of abstraction. I am somewhat fortunate, my situation allows me to do just that.

I've complained more than a few times about the quality of documentation in general, and documentation not existing. This time I put my money where my mouth is and [wrote up an entire section on debug info](https://github.com/rust-lang/rustc-dev-guide/pull/2649). There's still quite a few unfinished sections, but it includes brief overviews of the each part of the stack, and a [full guide for writing LLDB visualizer scripts](https://rustc-dev-guide.rust-lang.org/debuginfo/lldb-visualizers.html). As I work more with GDB, those sections will be fleshed out significantly. It should, hopefully, be enough to be reasonable to read, and give maintainers much more confidence when reviewing PRs.

# Debug Info Test Suite and Google Summer of Code

The debug info test suite is *very* broken. It's fragile, it only runs on `aarch64-apple` in CI (which must be manually invoked in PR's and takes several hours to run), and it can be difficult to even get it to run locally. I've done a bit of work to change that, and now I can get the tests to run on windows *at all*. There were quite a few that were already broken, and I don't trust the `*-msvc` tests to any degree. But at least it doesn't immediately crash or hang indefinitely anymore.

This is a bit of a tangent, but it has been a little excrutiating to submit PRs for the visualizer scripts because of this. The original scripts were written in a... pretty weird, inefficient way. They weren't written with a full understanding of LLDB's capabilities and it shows. They also have barely been changed since they were originally written. Again, I'm not trying to place blame, I'm not calling anyone out, this is simply the way things are.

The problem is, my changes rely on lots of undocumented bits of the `SB` API, `TypeSystemClang`, how LLVM reads debug info nodes, etc. It takes a lot of effort to write a PR description that is technical enough to explain how the change helps, simple enough to be understood by those who aren't familiar with the systems, and short enough that they'll actually be willing to read it. I also keep running into niche bugs in the debuggers, and fixing those isn't always trivial or under my control. Lots of time ends up going into making reproducable examples, submitting issues (and sometimes patches) on other projects, and coming up with the least hack-y workaround I can in the meantime. I understand it's part of the process, but man does it take the wind out of my sails sometimes.

It'd certainly be more expedient to rewrite the visualizer scripts from scratch. I don't say that idly, [I've already done it](https://github.com/Walnut356/syntheticproviders) while waiting for several PRs to clear. But I also know that's not exactly "how things work", and it's not reasonable to expect anyone to review and approve that sort of rewrite.

Anyway, back to the test suite.

## Reworking the test suite

This will be gross oversimplification, but the current tests suite works as follows: each test is a Rust program with a bunch of debugger commands at the top. `compiletest` compiles the code, then passes the commands to a python script that launches and coordinates the debugger. The python script passes the commands to the debugger's REPL to (mostly) print variables, then compares the string output from the debugger to the expected string in the test file.

There are a number of problems with this approach:

* Controlling for the environment the test is run in is obnoxious.
* Debuggers changing their default formatting for breaks almost every test.
* String comparisons are incredibly fiddly
* Intentionally changing 1 pretty printer can break dozens of tests because we end up accidentally testing the same thing in a bunch of different places
* There's no easy way to fix broken tests, it must be done by hand which can be incredibly tedious
* When tests break, the only feedback is "this string output wasn't found in the debugger's output" which is only marginally more helpful than "a failure occurred, good luck =)"

I made a pre-RFC of sorts to suggest using LLDB's API to inspect the data more closely. For example, if we're testing user defined structs, do we really care what delimiters are used when printing it? Probably not, we mostly care all the fields are present and contain the correct data. The printers for the fields' types should be tested in isolation so we aren't accidentally re-testing the same thing in every file. Do we care if a `vec![0i8, 1, 2]` prints exactly `[0, 1, 2]`, or do we care that the bytes that LLDB is reading are the `0x00`, `0x01`, `0x02`, and that it is interpreting them as signed 8 bit integers? Even if the answer is "we care about both", having tests for both is *incredibly* helpful diagnostically.

The [official MCP was accepted](github.com/rust-lang/compiler-team/issues/936) not too long ago. I submitted a Google Summer of Code project to implement that MCP and [it was accepted](https://summerofcode.withgoogle.com/programs/2026/projects/gzkF5BG0)! That's where I'll be putting most of my time for the next few months. It's a bit stressful, this is the first time I've ever really done something like this, but I'm very excited to give it a shot.

Hopefully, between a robust test suite and detailed docs, we should be able improve the visualizers more quickly and with more confidence.

# TypeSystemRust

Admittedly, I haven't really touched `TypeSystemRust` since last year. With it being close(ish) to completion, I could no longer ignore a nagging gut feeling I had.

"This isn't going to work in the long term."

It took me a long time of mulling it over to narrow in on why, and some of the frustrations I experienced above have reinforced my feelings. It boils down to this: Even if there were a good number of devs willing to work on debug info, which there currently aren't AFAICT, I don't think Rust devs will be willing to work primarily in C++ to maintain it. I also don't think LLDB devs will be happy to be saddled with Rust support if they don't have sufficient support from the Rust team.

I also can't maintain it on my own. I have the time *right now*, I work overnights at a hotel so there's plenty of downtime, but I'm financially unstable enough that I can't keep working here for much longer. Once I get a more demanding job, especially if it's a job writing code, I worry I won't have the time, energy, or brainpower left to dedicate to being the one guy who upkeeps the plugin. I'm sure that sort of "new job franticness" wouldn't last forever, but I still wouldn't feel right leaving the plugin to rot in the interim.

I'd love to say I'm confident someone else would step in, but I really don't think it would happen. My posts and work have gotten a lot of people interested, and they've helped out a ton with organizational stuff, reviews, feedback, getting me into contact with other helpful individuals, etc. I can't thank those that have reached out to me enough for their help and support. That being said - and it has taken a herculean effort for me to even type this out, I seriously hate saying things like this because it's so easy for it to be taken the wrong way - if I stop submitting debug info patches, debug info progress halts almost entirely. That's **absolutely not** meant to downplay the work others put in. My point is this:

Of the subset of people who are willing to work on open source projects, there's a subset of those that are willing to work on compilers. And only a subset of those are willing to work on something as unglamorous as debug info. And only a subset of those have the patience to subject themselves to the tangled mess that is the current state of debug info. And only a subset of those are interested in LLDB specifically, rather than GDB.

The unfortunate truth is that that number may be small enough for me to count on my fingers, maybe even on one hand, maybe even if I'd been in an industrial accident or two. We're already spread thin between covering both debug info formats, several different debuggers, different OS's, different visualizers, and that's all just with the LLVM codegen backend. Eventually we'll probably have to worry about GCC too.

Right now, there is one guy with too much free time and an unresonable amount of motivation imparting momentum, for no other reason than the love of the game. Progress has been made because others have been swept up in that momentum. That's fine, that was my intention when I wrote the other 3 blog posts and spent so much time writing `TypeSystemRust` and such. I just worry that `TypeSystemRust` is too much to leave to inertia when I'm unable to keep it going myself anymore.

I don't like how negative that sounds, but as a wise man once said, "You have to be realistic about these things". That doesn't mean I'm giving up on the idea of a `TypeSystemRust` though. I think it's important to record what my brain has been chewing on, to make clear that I haven't changed my mind for no reason.

I think the best path forward is helping LLDB to formalize their plugin API. Once that is in a workable state, it can be FFI-wrapped and we can write a `TypeSystem` in native Rust. At the very least, it's much more approachable to the average Rust compiler contributor. That's both because they'll be more willing to work in Rust, and because C++'s tooling such a joke that trying to figure out how `TypeSystem`s work from the C++ code is a nightmare.

I think this will be more work in the short term compared to finishing up my `TypeSystemRust` prototype, reformatting the code to fit to LLVM's style guide, adding tests, and upstreaming. I think that's worth it because it's *drastically* more likely to still be maintained in the medium/long term (1-10 years). It's also helpful for projects other than Rust.

My prototype is just about the minimum necessary to implement a `TypeSystem` at all, so I think it's not a bad place to start if you're trying to answer "where are the API boundaries?". I don't have a ton of experience with FFI or writing public-facing APIs or whatever, but I'm sure with some help from the LLDB maintainers it won't be too bad.

There are still some other problems that need to be fixed in LLDB (e.g. no way to prefer one `TypeSystem` over another if both can handle a given language, lots of weird edge cases with DWARF and PDB handling, etc.) but those should be pretty quick fixes in the grand scheme of things.

# LLDB Native/DIA Split

The prior section was a bit of a bummer, but I want to end on some objectively good news. If you've read my prior posts, you'll know that LLDB's PDB handling (which is mostly LLVM's handling under the hood) is split between `dia` and `native`. The former uses Microsoft's opaque `msdia140.dll`, which is distributed with Visual Studio, while the latter is entirely hand-written by LLVM maintainers. `dia` was used as the default, but had limitiations due to, well, being an opaque `dll` with Microsoft's own API and data structures. `msdia140` is (obviously) only available on Windows, so other platforms fall back to the `native` parser.

As of LLDB 22.0, `native` is the default PDB handler, and eventually `dia` will removed. This is wonderful news in the context of a `TypeSystem`, as it effectively cuts the work necessary to implement one by a third. Before you'd need a handler for `DWARF`, a handler for `dia`, and a handler for `native`. Now you only need `DWARF` and `native`.

I don't know all the details for why this change was made, but in my own experience, working with `dia`-based objects was incredibly awkward. That's why I implemented the `native` handling first in my `TypeSystemRust` prototype. I'm sure testing both and having to account for the Windows/Not Windows split was really obnoxious too.

Additionally, the PDB support is no longer hard-coded to require `TypeSystemClang`. I had to hack in a solution myself for `TypeSystemRust`, but now `SymbolFilePDB` and the PDB AST handler are as generic as their DWARF counterparts.

# Closing thoughts

Past that, there have been a few tweaks and improvements to the visualizers, and some progress as been made towards fixing some of the existing debug info deficiencies (e.g. no differentiation between pointers and refs), but nothing too crazy. Notably, there should be a pretty substantial performance improvement in Rust 1.96.

I don't like to be radio silent for so long, but I'm a big ponder-er. I like to think things through and make sure I do them right. Not just right now's "right", but also the best I can manage so that things aren't even harder tomorrow or the next day.

The next update should contain a lot more info about the test suite rewrite, and possibly some more substantial changes to the visualizers.
