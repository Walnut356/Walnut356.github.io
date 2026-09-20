+++
title = "GSoC 2026 Final Report"
date = 2026-09-20

[taxonomies]
tags=["programming", "rust", "debugging"]
+++

# Overview

Rust's debugging experience is a common pain point. The existing debugger plugins that help visualize data were/are somewhat buggy and unreliable. Some efforts were made to improve the situation, but a key blocker was the poor state of the `tests/debuginfo` test suite. It was difficult to control for ambient tools, updates to debuggers would frequently break existing tests. Over time, more and more tests were disabled, and CI runners stopped testing on entire combinations of platforms and debuggers. For example, LLDB was only tested on `aarch64-apple`. This lead to many of the test suites not working locally either, making it even more difficult to ensure that patches to the visualizers worked properly and did not introduce regressions.

The goal of this project was to use the Python API exposed by LLDB and GDB to create a testing framework more resistant to platform and debug info format differences. Secondarily, the tests in this new framework needed to be "bless-able" (i.e. able to be updated automatically for expected changes). A more in-depth description of how this system works is available in my [previous blog post](https://walnut356.github.io/posts/debuginfo-test-rework-1/). While part of the initial goal was to transition most/all of the tests to the new format, the state of the old tests on some targets made it clear that the higher priority should be placed on making sure `tests/debuginfo` is run in CI on all major targets.

# My work

During the community bonding period, I spent time familiarizing myself with `compiletest`, the test orchestration framework that handles `tests/debuginfo` (among others). Some time was spent experimenting with how the frameworks could slot into `compiletest`'s existing pass/fail mechanisms, and some was spent making preparatory bug fixes and refactors.

* [PR#156688 Manually load GDB visualizers on windows for GDB tests](https://github.com/rust-lang/rust/pull/156688)
* [PR#156899 fix breakpoint callback registration in `lldb_batchmode`](https://github.com/rust-lang/rust/pull/156899)
* [PR#157234 Convert `lldb_batchmode` to a package](https://github.com/rust-lang/rust/pull/157234)

Some fixes were also required for the visualizers themselves to get them in a state reliable enough for us to properly test

* [PR#157289 Add infallible primitive type lookups to template arg resolver](https://github.com/rust-lang/rust/pull/157289)
* [PR#157298 Use alternate means of detecting enums in `is_udt`](https://github.com/rust-lang/rust/pull/157298)
* [PR#159834 Apply `str` debugger visualizer to `*const str`, `*mut str` and `Box<str>`](https://github.com/rust-lang/rust/pull/159834)
* [PR#160791 Use recognizer functions for enums and tuple structs](https://github.com/rust-lang/rust/pull/160791)
* [PR#160842 Apply `&[T]` visualizer to `*const [T]`, `*mut [T]` and `Box<[T]>`](https://github.com/rust-lang/rust/pull/160842)
* [PR#161216 Clean up some `lldb_batchmode` lints](https://github.com/rust-lang/rust/pull/161216)

The Pyton API framework was largely implemented in the following 3 PRs:

* [PR#157529 Add tests/debuginfo data classes to define schema](https://github.com/rust-lang/rust/pull/157529)
* [PR#158298 Add lldb-repr command for tests/debuginfo](https://github.com/rust-lang/rust/pull/158298)
* [PR#160377 Add gdb-repr directive for tests/debuginfo](https://github.com/rust-lang/rust/pull/160377)

This also came with the need to update some of the Work In Progress sections of the Rustc Dev Guide's Debug Info chapter:

* [rustc-dev-guide PR#2947 Document debuginfo repr directive](https://github.com/rust-lang/rustc-dev-guide/pull/2947)
* [rustc-dev-guide PR#3016 Update debuginfo testing information w/ GDB changes](https://github.com/rust-lang/rustc-dev-guide/pull/3016) - still awaiting approval at time of writing

Around this point, one of the patches ended up letting a regression slip through that we needed to [re-bless in a followup PR](https://github.com/rust-lang/rust/pull/161573). This made it clear that getting the tests to run in CI on all relevant targets was more urgent than switching existing tests to the new format. Testing LLDB on the `aarch64` linux CI runner took a bit of experimentation but was ultimately pretty straightforward.

Running the tests on the Windows CI runners was a bit more of a hurdle. Running `tests/debuginfo` locally yielded 1 LLDB test failure on `windows-gnu` and 53 LLDB test failures on `windows-msvc`. I opened [an issue with a listing of the failures](https://github.com/rust-lang/rust/issues/161657), and got to work diagnosing and fixing them. This involved tracking down quite a few weird behavior quirks or bugs in LLDB itself, alongside a number of changes to the test suite.

* [PR#162358 Force `u8`/`i8` numeric formatting on LLDB](https://github.com/rust-lang/rust/pull/162358)
* [PR#162359 Use `lldb.eTypeOptionHideChildren` for msvc tuples](https://github.com/rust-lang/rust/pull/162359)
* [PR#162364 Use `#[repr(C)]` on debuginfo test structs](https://github.com/rust-lang/rust/pull/162364)
* [PR#162412 Fix msvc-specific differences in debuginfo tests](https://github.com/rust-lang/rust/pull/162412)
* [PR#162598 Fix the remaining `tests/debuginfo` failures on `msvc`](https://github.com/rust-lang/rust/pull/162598)

Notably, most of the tests didn't use `#[repr(C)]`, so the field ordering of the structs was not guaranteed. This caused issues because LLDB's DWARF parser kept fields in source-order, but the PDB parser reordered the fields to be in offset order. This ended up being a bit of a happy coincidence though, as it revealed that many of the tests I changed *should* have had `#[repr(C)]` in the first place, since they were deliberately written to test LLDB's handling of padded structs.

# What's left

In the immediate-term, we need to get CI to actually run the tests. [PR#161574 Enable tests/debuginfo on aarch64-gnu-llvm-21](https://github.com/rust-lang/rust/pull/161574) accomplishes that for PR CI (awaiting approval at time of writing). Getting the Windows runners to run the tests requires a bit of extra legwork, as it's not quite as easy to get LLDB and the appropriate Python version.

In the short-term, the goal is to write a single comprehensive test for each of the types we have visualizers for, and remove the visualizer-dependent types from the existing tests. This should allow us to reduce the total number of tests. Once that is complete, the remainder of the tests can be ported over to the new format (where applicable).

In the medium-term, I want to look into some way to accomplish something similar for the CDB tests, even if it's just adding `--bless` support.

# Acknowledgements

A huge thank you to my mentors [@Kobzol](https://github.com/Kobzol) and [@jieyouxu](https://github.com/jieyouxu). They've both provided incredibly valuable insight about CI and the broader Rust testing ecosystem; they've pushed me to refine both my plan and my implementation; they've caught numerous bugs and logic errors in PR's that were *definitely* too large; and they've been incredibly friendly and welcoming throughout.

I also want to shout out [@Nerixyz](https://github.com/Nerixyz). While they weren't involved in GSoC, they *did* help diagnose and fix quite a few LLDB bugs that I filed issues for.

Additionally, thank you to Google and all of the staff that run GSoC. These sorts of programs are indcredibly cool, and I'm grateful that I was able to take part.
