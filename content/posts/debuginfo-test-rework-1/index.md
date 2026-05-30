+++
title = "GSoC Debuginfo Test Rework: Data"
date = 2026-05-30

[taxonomies]
tags=["programming", "rust", "debugging", "gsoc"]
+++

As I mentioned in my previous post, I started a Google Summer of Code project to (more or less) dig `tests/debuginfo` out of tech debt.

The original plan was to create and entirely new test suite. Tests would slowly be translated over, and then the old test suite would be removed. There was a bit more complexity with this option, between the plumbing for the new test suite, and the coordination between the two. There was also the matter of preserving the behavior of the tests that do more than just print variables (e.g. stepping behavior in macros).

<!--more-->

With that constraint, it was difficult to justify rewriting/reworking most of the core testing logic. Instead, I decided to add a new directive that dispatches to special handling for the debugee variables. The majority of the tests, and nearly all of the problems with the test suite, are purely about variables and their printed output, so it's not a huge deal if the non-variable tests remain as-is. By using a directive, we can update the existing tests directly, no `tests/debuginfo2` required.

# Blessability

`--bless` is a test option used by several of the test suites to automatically update the test data when expected changes are made. `tests/debuginfo` does not support `--bless`; adding it is one of the core deliverables of my GSoC project. Having no `--bless` support makes it very tedious to make changes to the visualizers, as dozens of tests (and sometimes hundreds of individual test lines) need to be updated by hand. This exact problem is why most of the recent changes I've made to the visualizers have been refactors and performance improvements rather than improvements to the visualizations themselves.

Adding `--bless` support with the current test structure is a huge pain because the commands and test data for all 3 debuggers live in the same file, alongside the debuggee source code. We'd need to preserve the config directives, comments, etc., and we'd need some way to separate the test data between specific targets. There's currently no good way to reconcile unfixable differences between LLDB's handling of, say, `*-windows-gnu` and `*-windows-msvc` debug info.

It's absolutely possible to do all that, but it's more trouble than it's worth. Splitting the test data off into its own file makes things much easier. But how do we store that data?

# Test data format

Currently, all of the test data exists as raw strings that are compared against LLDB's output. It's been a pretty awful open so far. It's too brittle, and it doesn't allow us to test with the granularity that I think is necessary. As I've said before, debug info is ~5 huge programs (rustc -> llvm -> lldb/gdb/cdb -> plugin interface/visualizer scripts -> debugger adapter/frontend) playing telephone through increasingly obtuse file formats, whose translations are often lossy. There are a *ton* of assumptions made along the way to keep things working correctly. Currently, the only way those assumptions are tested is via a variable's string-summary. If the assumption isn't reflected by that string-summary, it's effectively untested and only works incidentally (or doesn't work at all, but we haven't noticed/haven't fixed it). Many of the things that end up being tested (and end up causing tests to fail at "random" times) are tested implicitly rather than explicitly. This system doesn't work at all if *any* of the layers aren't in sync. If a part of the debug info pipeline assumes something, we need to know it's actually true.

That was the key motivation behind both my pre-RFC and this GSoC project. I got a bit of pushback regarding the need to test more than just the user-facing output, but I've been somewhat adamant on this point: nothing is going to prevent non-debuginfo changes from breaking debuginfo tests, because we can't prevent \<changes to things that debuggers use or expose\> from affecting \<user-facing debugger output\>. Unfortunately, the space of \<things that debuggers use or expose\> is huge compared to many other types of changes to the compiler. Debug info, by its very nature, exposes a program's private implementation details to the user. Even if we *could* prevent that, doing so would obliterate the debuggability of Rust programs. We have to work around the fact that "private" is irrelevant, and that almost anything can cause a user-facing change via debuggers.

A large part of the reason this test suite has accumulated so much tech debt is because diagnosing these issues *sucks*. If you've read my other posts about debug info, you know I speak from experience. As it stands, it's significantly easier to disable a test on a target (or disable it entirely) than it is to diagnose and fix. Especially when the problem stems from behavior in LLVM or in one of the debuggers. Very few people are experts on debug info. Very few are experts on the internals of any of the debuggers. Even if they were, it's still annoying to diagnose and update the tests. So, unfortunately, they often don't get diagnosed and they don't get updated.

My fear with adding in a `--bless` option in isolation is that we'll end up right back in this hole. `--bless` doesn't make the investigation any easier, it only affects how easy it is to get the tests to pass after a change is made. I worry that people will `--bless` away legitimate issues and regressions because nobody knows enough to fix things properly and the evolution of the language can't grind to a halt to wait for someone who does. That may sound overly pessimistic, but that's effectively how the `disable-<target>` directive has been used for quite a while now.

With all that being the case, it means we need error reporting **so** good that even someone unfamiliar can recognize what is a trivial difference and what is a real regression. We might even get away with making them so good that people can reasonably fix the issues themselves, even if they aren't experts on debug info. If the tests fail in a non-obvious way, it reduces the triage space an *extraordinary* amount to know what specifically caused the user-facing output not to match. For that, we need more data to work with.

## Weighing the options

The most straightforward thing we can do is just make a struct that pulls out whatever info we need from the debugger's in-memory representation of the variable. We can then serialize that struct, store it, retrieve it at test time, and compare against it.

I considered several formats for data storage. I think for a first iteration (and for git-diff purposes), human-readability is important. I also didn't want to add any new dependencies, especially on the Python end. I even considered piping the data from Python back to Rust to take advantage of `serde`, since `compiletest` already depends on it, but I scrapped that idea due to the extra nonsense required to communicate back and forth. In the end, the format that made the most sense was JSON. I'm a big JSON hater, but for these constraints there weren't really better options.

* TOML: Python 3.11 added `tomllib`. Unfortunately, it only includes *read* support, not write support. I could talk for a while about how a standard library parser for a file format, but *not* having a way to generate files of that format (then doubling down to the point of talking someone out of adding write support) is a uhh... *wild* choice, but I'll restrain myself.

* CSV: The amount of nested data and arbitrary-length lists makes it unsuited to CSV.

* Pickle: On one hand, the built-in support is excellent, and there is a human-readable protocol. On the other, pickle is somewhat obscure, and if we ever did want to move more of the processing to the Rust side, it probably wouldn't be fun to handle.

* SQLite: Very overkill, it also wouldn't play well with git diffs

* `marshal`/`configparser`/some other custom format: Ends up being a lot of extra work just to end up with "JSON but a slightly less bad".

So, JSON it is. Serializing to JSON is very easy. We just make our structs `@dataclass`, and call `dataclasses.asdict`. The dict can then be serialized with no extra effort by Python's built-in JSON module.

Deserializing isn't quite as easy. There is no `dataclasses.fromdict`. We could write deserialization code for every individual `dataclass`, but then we have to keep it up to date if we ever change the schema. Not a huge deal, but especially in the prototyping phase it started to irk me. I'm a big fan of crimes against Python, so I figured there had to be some way to make a "generic" deserializer that works for any `dataclass`. One slightly sickening, but ultimately useful, fact about Python: despite type hints not *enforcing* anything at runtime, they do still *exist* at runtime and can be inspected and manipulated.

With that, we can write a straight-forward function that does what we need:

```python
def from_dict(ty: type[Any], data: JsonType):
    if get_origin(ty) is Optional:
        ty = ty.__args__[0]

    if isinstance(data, list):
        (inner,) = ty.__args__
        print(inner)
        return [from_dict(inner, i) for i in data]

    if get_origin(ty) is dict:
        assert isinstance(data, dict)

        if ty.__args__[1] is Variable:
            return {k: from_dict(Variable, data[k]) for k in data.keys()}
        if ty.__args__[1] is Child:
            return {k: from_dict(Child, data[k]) for k in data.keys()}

    if is_dataclass(ty):
        assert isinstance(data, dict)

        field_types = {f.name: f.type for f in fields(ty)}
        try:
            field_map = {}

            for f in data:
                f_type = field_types[f]

                if isinstance(f_type, str):
                    f_type = globals().get(f_type) or getattr(
                        __builtins__, f_type.split("[", 1)[0]
                    )

                field_map[f] = from_dict(f_type, data[f])

            return ty(**field_map)
        except KeyError as e:
            print(
                f"Unable to convert dict to {ty}: Invalid field name {e}. If the test schema was \
changed intentionally, use the `--bless` option to update test data to the new schema."
            )

    return data
```

Uhhh... huh. That probably needs a few comments actually. The short of it is that it uses the `dataclass`'s field type hints (which are just class objects) as constructors for the corresponding JSON data. The function recurses in a few spots to handle nested data, and then the resulting dictionary is splatted (i.e. `**` operator, which uses a mapping as a set of keyword arguments for a function) into the default `__init__` function generated by `@dataclass`. Anything that's not a dataclass just returns itself. That shouldn't cause any issues, since the dataclasses are composed entirely of other dataclasses, and fields who decode to the same type they were encoded from (i.e. no tuples allowed).

The nice thing about this whole setup is that we can hot-swap the format if we ever want to move to a better option. It only requires changing out the serialization/deserialization code and re-blessing the test data.

# Data

So we know how we'll store it, but what do we actually store? As I mentioned before, it's important that we test every assumption made. We also need to keep an eye out for anything that would be very diagnostically useful to have. For a different angle, we can also look at what assumptions the tests currently make that are *known* to be wrong.

One comes to mind immediately: We currently assume all targets will generate identical output for a given debugger. This is flat-out not true. We don't even need the debugger to prove this. There is [target-based conditional behavior within the debug info generation code](https://github.com/rust-lang/rust/blob/6368fd52cb9f230dfb156097625993e7a8891800/compiler/rustc_codegen_llvm/src/debuginfo/metadata/enums/mod.rs#L46). This has been sortof worked around in the past by disabling tests on certain targets and (sometimes) creating new target-specific tests. That feels like a bit of a hack, and comes with some downsides.

1. The test program ends up duplicated, changes to one aren't necessarily reflected in the other.
2. The tests are separate enough that, at a glance, it implies they aren't related.
3. It gives a false sense of security when verifying changes locally. If a change is made that affects multiple different targets, a local test will likely only run on a single target. It's easy to pretend the auto-ignored targets don't exist, and thus think the change is good to go. It's much harder to ignore 1/3rd of a test's input data not being updated.

So, at a top level we need to store each target's data individually. My current set is `nonwindows` which is self explanatory, `windows_gnu`, and `windows_msvc`. `windows_gnu`, despite having DWARF debug info, doesn't have 1:1 equivalence with `nonwindows` because some structs have platform-specific layouts (e.g. `OsString`). A recent PR has made me a bit scared that `aarch64-apple-darwin` will require its own test data, since Apple LLDB is different than LLVM LLDB, but I'll avoid splitting it out for now to help keep the file size down.

To help aid in diagnosing issues, especially in the long-term where it may be a year or more between updates to a test and build/error logs no longer exist, I also want to store some metadata. Trust me when I say, cross-referencing debugger release dates, PR timings, and Apple's nonstandard LLVM versioning scheme and corresponding xcode version, is not fun at all. A few extra bytes to store the full debugger and python versions is worth it.

For the visualizers to work, providers must be associated with values. Often that association is done via a regex match on the type name. Conveniently, LLDB has `GetSyntheticForType` and `GetSummaryForType` for pulling the providers' names from a variable. Storing the type name can also help notify us when, for example, a provider is no longer matching due to a struct being moved to a different module. Types are the major reason we need to separate out `windows-msvc`, since many type names are modified to work around the limitations of PDB and Microsoft's debug engine.

There is a non `windows-msvc` issue though. Let's say you have a `usize`. Its type name is `usize`, right? Haha, no. LLDB normalizes the type names of primitives. By "normalizes," I mean "enforces c-style naming regardless of what you specify". That's fine in theory; `u8` maps to `unsigned char`, `i16` maps to `short`, it's all good as long as we store the type name LLDB gives us, right? Once again, no. The "at least" in "at least X bits in size" strikes again. On `aarch64-apple-darwin`, `u64` is normalized to `unsigned long`. On `x86_64-pc-windows-msvc`, it normalizes to `unsigned long long`, despite both being 64 bit values. Very cool, thanks LLDB.

This isn't the only problem caused by type name normalization either. We can't use `T<u64>` and `T<usize>` in the same test because LLDB's normalization gives them identical type names despite having different underlying debug info. It doesn't matter for primitives since they get special handling, but for types with generics, attempting to inspect both in the same session results in neat error messages.

Overall, I'm not super happy with the need to store so much data. Large parts of it will probably be identical between targets. I could add some sort of system to detect this and consolidate as much data as possible, but I don't think I can justify that for the GSoC project specifically.  It adds a big space for dumb logic bugs that could undermine the reliablity of the test suite, and I'd rather have something I know 100% always works for the first iteration.

Anyway, aside from type name, we should also store the size, alignment, and field layout. We often call `GetChildMemberWithName` and `GetChildAtIndex` on non-synthetic types, which uses the field layout of the type. If a field gets renamed or changes ordering, it breaks the visualizers. For container types, we also need to care about the generic parameters. For DWARF this just means calling `GetTemplateArgumentType`, but that fails for PDB debug info because PDB has no node for template args. Instead, the visualizers (more or less) pull the template type out of the type name and do a lookup for a type with that name. We can just call that function directly in the serialization code when necessary.

For enums on `windows-msvc`, we also technically care about static fields, since they describe the discriminant->variant mapping. LLDB does *not* have a convenient way to grab all the static fields for a type via the API. You *must* know the name of the static field to retrieve it. Luckily, this is the only instance of rustc outputting static fields as far as I know, so there's only a handful of possible names we'll ever need to look up.

Anyway, onto the value data itself. We need to store the value's potential type-name override at this level, because it is generated by the `SyntheticProvider`, which is attached to the value itself, *not* the type.

What we care about depends on if it's a primitive or not (mostly). For primitives, we mostly care about the formatter and the underlying value, whereas non-primitives care mostly about child values. Technically, I could create a dedicated `Primitive` and `UDT` class and use a metadata field to distinguish between the two, but that's not a very robust option. Providers allow you to create children for any type, including primitives. They also allow a `UDT` (e.g. `NonZero<T>`, `Cell<T>`) to return a primitive value as if it's a primitive. That means we may as well just have the same schema for every type and use null fields whenever something isn't applicable.

At first glance, it may seem redundant to store the type's fields *and* the value's children, but these are testing two different things. One way to think of it is that the type's fields test the visualizer input, whereas the value's children test the visualizer output. For example, a `Vec<u64>`'s the fields are effectively `{data_ptr: *mut u8, cap: usize, length: usize}`, but the children might be `{[0]: 10, [1]: 20, [3]: 30}`. The former is what we use to generate the latter so we need to know if the layout is as we expect, but the latter is what people interact with so we need to make sure it makes sense and is useful.

The children themselves don't need to be tested as thoroughly as top-level variables, since presumably we'd have tested other variables of the child's type in isolation somewhere else. That means children pretty much only need to store their type, their value (if applicable), and their children (if applicable).

With that, we have pretty much everything we need. It's quite a bit more than the old string summaries, but it should make legitimate bugs *way* easier to diagnose. Next time, I'll talk about how this will be slotted into `compiletest`, how `lldb_batchmode` will call it, and probably some of the finer details on how exactly each piece of data is tested.
