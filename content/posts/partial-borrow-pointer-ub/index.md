+++
title = "Bypassing the borrow checker - do ref -> ptr -> ref partial borrows cause UB?"
date = 2024-08-09
description = ""
[taxonomies]
tags=["programming", "rust", "compilers"]
+++

Partial borrows across function boundaries don't really work in Rust. Unfortunately, that's kind of a major issue. There are workarounds, some are outlined [here](https://smallcultfollowing.com/babysteps/blog/2018/11/01/after-nll-interprocedural-conflicts/), but all of them come with pretty major drawbacks.

<!-- more -->

Inlining is a nightmare for obvious reasons. Factoring ruins your struct layout (which could matter a lot for performance reasons), and can make your code substantially less readable as related information is no longer packed together, or the internal relationships end up mangled.

Free (or associated) functions are just awful. They break encapsulation, they interact awkwardly with the surrounding code, and they're far easier to make mistakes with. If I call `self.lookup()`, I can look at the constraints of `self` and `lookup()` and know that the function is infallible. An associated function that just takes a generic reference to a hashmap *may* panic due to `.get().unwrap()` because someone wasn't paying attention and tossed any ol' hashmap in there. The whole point of methods is to prevent putting nonsense data into highly constrained functions, and it's frustrating having to work around such a glaring hole in Rust's (usually) clear and convenient semantics.

I don't have a ton of experience with view structs, but at the very least they're a bit of extra legwork and split the logic code between multiple different structs.

In the back of my mind has always sat the idea that I have an escape-hatch with raw pointers. This sort of "partial borrow" issue would be non-existent in C, and pointers are pointers, so there shouldn't be any real issue in Rust. I held back partially because I never came across an instance of partial borrows annoying enough that it justified the *other* annoyance of using `unsafe`, partially because the documentation around pointers makes it a little unclear if the way I wanted to do things is *instantly* UB or not, and partially because any time I ask questions about `unsafe` in public forums, people get very condescending and preachy. It makes it very hard to learn when the information is being obfuscated by "if you need unsafe for something like this, your code is structured wrong" and "If I had review rights, I would reject any code that looked like this". I want to go on a bit of a tangent and share my thoughts about this sort of mentality. If you aren't interested, feel free to <a href="#the-partial-borrow-problem">skip to the meat of the post</a>.

By nature, I'm a very methodical person. I like clearly defined rules, well justified reasons behind those rules, and a thorough enough understanding that I can apply my knowledge to any situation - including when to bend or break the rules. At least for me, that sort of understanding can only come from careful prodding and experimentation. Stringently following exactly what the "best practices" are just isn't good enough - memorization and following instructions aren't the same as learning. What if the limits I was told were inaccurate, poorly worded, or overly cautious? Even if I eventually settle into following best practices, I have to *know* they're "best", and that only comes with experience. That may sound like a lot of unnecessary work. In some ways I guess it is. But I've found that, even outside of programming, established convention ends up being the willfully-ignorant-"we've always done it this way" variety just as often as it's been genuine wisdom. It might take me longer to get my bearings when trying something new, but I've always felt I come out the other end ahead of those who stay unquestioningly within the box of common practice.

One of the first programming videos I ever watched was [about "Forbidden" C++ by javidx9](https://www.youtube.com/watch?v=j0_u26Vpb4w). It's also one of my favorite programming videos I've *ever* watched. If you're strapped for time, this video goes over things like global variables, void pointers, and memory leaks. The purpose is demystifying these "bad practices" by explaining exactly *why* something like a global variable is considered bad practice, and giving legitimate use-cases for them. Most importantly, he assures the viewer that it's **okay to experiment**. As he says in [another excellent video](https://youtu.be/vVRCJ52g5m4?t=415)

> Remember, the tools work for you, and you can't hurt them... break the rules. Use global variables all over the place. Use unsafe system calls. Don't check all your integers to make sure they're within range. This is stuff they will not advise you to do at school.
>
> Well you know what? It doesn't matter! You're learning! You're not gonna cause any harm. You're not going to break your computer (mostly). The computer doesn't have feelings, you're not going to upset it, and your code isn't being reviewed by somebody on the internet. It doesn't matter! Do whatever you need to do to get the job done... Get yourself into a real mess - there's no greater mental challenge for a programmer than debugging stuff. And if you want a career as a programmer, you're going to be debugging a lot of stuff... if you can get yourself out of it, that journey will have presented so much new learning to you. And if you get really stuck, select all, delete, start again... The best way to learn is to make mistakes.

This was far and away the most liberating thing I heard when I was brand new to programming, and parts of this quote echo in my mind nearly every time I open up my IDE. The world won't end if my program is a little grimy. Learning is more important than idealized perfection, especially on a personal project.

When first starting out, undefined behavior was this "boogeyman" that lurked behind seemingly innocuous code. It could jump out from anywhere, do anything, at any time. The more I've learned about compilers (and I'm no expert by any means), the more I dislike that view. Compilers are not magic, they don't turn your code into nonsense on a whim. Even the consequences of the UB are "predictable" in the sense that there's *literally* a reason that your compiler chose to modify the code the way it did, and there is a *completely understandable chain of events* that lead to the unexpected outcome. You might not know what that reason is, but that doesn't make it random chance. It was incorrect assumptions on the programmer's end, combined with a strict boundary for what constitutes "valid state and behavior" in a system that, while outside the programmer's control, *is* in *somebody's* control. I think this perspective made more sense back when there were less protections around memory access (e.g. read/write/execute flags), but I certainly don't think it holds to the same degree today.

Undefined behavior - the tangible *consequences* of violating a language's rules and assumptions - is obfuscated by the fact that it's a [negative space](https://en.wikipedia.org/wiki/Negative_space) of sorts. I think it's easy to lose sight of the boundary itself and assume that something is UB just because it's relatively close to that boundary. Even reading Rust's specifications isn't enough - words themselves are open to interpretation, whereas the boundary *literally* isn't. Even if it hasn't been vocally defined, even if nobody knows that it's there, it still exists. The program either violates the compiler's assumptions or it doesn't. It is not random chance whether a program *can* or *cannot* produce undefined behavior, even if the consequences manifest sporadically. Assuming compilers are deterministic, UB is also consistent in the sense that if you change nothing about your program and compiler, that miscompilation will continue to happen.

Whew, that was quite the tangent. Hopefully that makes it clear *why* I would bother doing something like this in the first place. I won't be fully satisfied with any solution I choose until I tie up this loose end and figure out if it works or not. Even if it's not the best solution in this exact case, I'll still have learned something about the boundary of undefined behavior.

## The partial borrow problem

I started looking into this due to borrow-checker wrestling in the Starcraft Simulator, specifically when adding in buffs/debuffs. We have a `Coordinator` struct, which contains 2 `Army` structs (and some miscellaneous metadata). Each `Army` struct looks roughly like this:

```rust
pub struct Army {
    ...
    pub base_units: Map<Base, Unit>,
    pub units: Vec<State>,
    ...
}
```

`base_units` is a hashmap that contains the immutable stats of each unit type (max health, what weapons the unit can use, etc.). `units` is a flat vec, accessed via integer handles, and contains all of the data that can change over the course of a fight. It's important to note that this is a "setup -> run -> inspect" situation. Once the setup phase is complete, `.base_units` and `.units` will never change size (and thus never be reallocated), are never rearranged, and the contained structs are never assigned to, only individual fields in those structs (e.g. never `army.units[i] = ...`, only `army.units[i].field = ...`). For now we can pretend state looks like this:

```rust
pub struct State {
    pub max_speed: Real,
    pub effects: Vec<Effect>,
}
```

What I want to implement is Concussive Shell, a non-stacking debuff on the marauder's attack that slows any unit it hits by 50%. These types of stat modifiers are represented in memory by the `Effect` enum. As this is a prototype, `Effect` only has one variant, so we can pretend it's actually just a struct that looks like this:

```rust
pub struct Effect {
    stat: Stat,
    apply: fn(&mut State),
    timestamp: Real,
}

impl Effect {
    pub const fn concussive(timestamp: Real) -> Self {
        Self {
            stat: Stat::Speed,
            apply: |state: &mut State| state.max_speed *= const_real!(0.5),
            timestamp,
        }
    }
}
```

`stat` tells us which part of `State` is being modified (useful when we remove the effect). `timestamp` is the time at which the `Effect` wears off and needs to be removed. Each tick, the coordinator checks the timestamps of each effect on each unit in each army, and removes the effect when necessary. "Removing an effect" in this instance isn't as straightforward as just setting the stat back to its default value, as we need to consider units with multiple effects that each modify the same stat (e.g. a zealot whose speed is boosted by charge, but lowered by concussive shell). The simplest way I could think to do this was by fully recalculating the stat each time a modifier is removed. There aren't a whole lot of buffs/debuffs in starcraft, and as far as I know there aren't any that stack, so I don't think it's the end of the world computationally or storage-wise.

That manifests in the `Coordinator`, with the following function:

```rust
// coordinator.rs
fn tick_effects(&mut self) {
    // eliminates code duplication. The closure also captures (thus partial
    // borrows) any other part of `self` that might be used.
    let mut _inner = |army: &mut Army| {
        for unit in &mut army.units { // <-- first borrow
            let mut i = 0;

            while i < unit.effects.len() {
                if unit.effects[i].timestamp <= self.time {
                    unit.effects.swap_remove(i);
                    army.reset_speed(unit) // <-- uh oh
                    // swap remove means we don't need to increment i
                } else {
                    i += 1;
                }
            }
        }
    };

    _inner(&mut self.red);
    _inner(&mut self.blu);
}
```

Iterating over `army.units` holds a borrow over `army`. A call to `army.reset_speed(unit)` requires borrowing `&mut army` as well, resulting in a compiler error.

A similar problem manifests in `reset_speed()`. As a reminder, in this is a prototype we're only dealing with the speed stat, but it won't always be that way. These functions have a bit of added complexity when dealing with other stat types, which is why we'll need a full mutable borrow on `state` if we want `effect.apply` to be fully "generic".


```rust
// army.rs
pub(crate) fn reset_speed(&mut self, state: &mut State) {
    let speed = self.base_units[&state.base].movement.speed;
    state.max_speed = speed;

    for effect in &state.effects { // <-- first borrow
        effect.apply(state) // <- uh oh
    }
}
```

Same deal, `&state.effects` borrows from `state`, but we also need to pass `state` mutably into `effect.apply`.

> Aside: I know these can both be solved in safe rust via indexing, but as I said before, this is at least partially an academic pursuit. It's also worth noting that this has some ergonomic downsides, since I can't use iterator methods like `.filter()`. I'll include the safe versions of these functions later on after we talk more about the compiler.

This is where the escape hatch comes in. As the *programmer*, this is a pretty trivial case. With `state` being the same as `army.units[i]`, we *really* only have:

* a read from `state.base` (which is just a `Copy` enum)
* a read from `army.base_units[base].movement.speed`
* a write to `state.effects` immediately before `reset_speed`
* a read from `state.effects` within `reset_speed`
* a write to `state.max_speed` (or some other stat that is guaranteed not to overlap with `state.effects`) within `effect.apply`

None of these conflict in an absolute sense - we're not changing any values that we already have an immutable reference to, and none of the mutable values overlap. In C it would be trivial to hook this up, and it would look pretty similar to a Rust implementation that used free functions. To get the same behavior via Rust we can bypass the borrow checker using raw pointers. To maintain regular Rust ergonomics, we can turn the pointer **back into a ref**. Note that the return-type coerces the lifetime to a *different* lifetime than the value passed in.

```rust
pub fn unsafe_ref<'b, T>(val: &T) -> &'b T {
    unsafe { (val as *const T).as_ref().unwrap() }
}

pub fn unsafe_mut<'b, T>(val: &mut T) -> &'b mut T {
    unsafe { (val as *mut T).as_mut().unwrap() }
}
```

```rust
// coordinator.rs
fn tick_effects(&mut self) {
    let mut _inner = |army: &mut Army| {
        for unit in unchecked_mut(&mut army.units) {
        ...
```

```rust
// army.rs
pub fn reset_speed(&mut self, state: &mut State) {
    let speed = self.base_units[&state.base].movement.speed;
    state.max_speed = speed;

    for effect in unchecked_ref(&state.effects) { // no borrow checker error
        effect.apply(state)
    }
}
```

If I was purely working in raw pointers, that would be the end of the story I think, as raw pointers don't carry the same guarantees as mutable references. The fact that I'm casting it back to a reference *and* I'm "using" other references to the struct is the real issue.

To be clear, this is already very unsafe. Any mistakes in logic *will* result in UB, even if accessing the `unsafe_mut` ref and `&mut self` at the same time isn't instant UB by itself. It's not just bypassing the borrow checker, we're giving it the middle finger. Rust will continue to uphold borrow checker rules about `val` *as if `val` is not currently accessible anywhere else*, so any incorrect assumptions are very dangerous.

With these functions we no longer have to hold borrows to `army` and `state` while iterating over `army.units` and `state.effects`. That frees us up to call methods and pass them around without any trouble. At least, without any trouble from the *borrow checker*. Compiler trouble is another issue.

Fun note, after starting this blog post I found another repo using similar functions in the wild: [amazon's official Rust parser for the Ion data format](https://github.com/amazon-ion/ion-rust/blob/main/src/unsafe_helpers.rs). It's not quite the same since the behavior is split between two functions, but I still thought it was a funny coincidence.

## But is there undefined behavior?

It's complicated. My gut instinct was "no". After reading the docs for raw pointers it was "probably yes". After consulting some community members, we agreed "almost definitely yes". After doing a few nights of research and a lot of staring at a wall, my answer was "probably no". Now though? My answer is "who knows". And that's kind of a genuine question. I don't think this information is centralized anywhere aside from a few people's brains, and that *might* be intentional. Researching this took me down quite a few rabbit holes, stretching my knowledge of compilers pretty well past its breaking point. Forgive me if I've misunderstood or misinterpreted things. As I'm doing my "final" pass before posting this, I am *still* reading literature from experts about how this sort of thing works. This turned from a "I'm gonna take a day to whine about this minor annoyance in my fruit salad text" to "oh god it's been weeks and I'm still rewriting sections". Take all of this with the largest grain of salt you can find. If anything major is pointed out, I'll edit the post.

### Rust's Rules

The first place to look is [std::ptr](https://doc.rust-lang.org/std/ptr/index.html), which offers this bit of wisdom:

> Whether a pointer is valid depends on the operation it is used for (read or write), and the extent of the memory that is accessed (i.e., how many bytes are read/written) – it makes no sense to ask “is this pointer valid”; one has to ask “is this pointer valid for a given access”

I hinted at this above when describing the escape hatch in terms of reads and writes rather than pointers. This is echoed in a [fantastic blog post by the author of the nomicon and Learning Rust With Entirely Too Many Linked Lists](https://faultlore.com/blah/fix-rust-pointers/):

>The primary function of aliasing is as a model for when the compiler can semantically cache memory accesses. This can either mean assuming a value in memory hasn’t been modified or assuming a write to memory isn’t necessary...Now it’s often clumsy to talk about accesses aliasing, so we usually talk about pointers aliasing as a shorthand... The reason that the actual model is in terms of accesses and not pointers is because that’s the thing that we care about.

Since passing around the pointers doesn't matter if we don't use them, getting a ref via `unsafe_mut` and then calling a function that takes `&mut self` *and* that unsafe ref likely isn't UB so long as we don't access `&mut self`.

Back to the pointer documentation:

> The precise rules for validity are not determined yet.

Oh. Well shit.

This is a large part of why I can't put a 100% guarantee on anything I'm saying. *The language itself* isn't making guarantees, and experts are still investigating various parts of the semantics to this day. I am *not* an expert, I'm just a dude who spends inordinate amounts of his free time investigating stupid things for stupid reasons. As such, I will continue to dig, as I think I can make a more solid case than "Iunno, it seems fine". Not *too* much more solid though, as we're diving directly into semantics hell.

Since all of my refs are derived from pointers which are derived from valid refs, and this is a single-threaded application, the only relevant part of the minimal outline given by the pointer documentation is the last one:

> The result of casting a reference to a pointer is valid for as long as the underlying object is live and no reference (just raw pointers) is used to access the same memory. That is, reference and pointer accesses cannot be interleaved.

Which doesn't actually help that much. In this case I think we can assume that ref -> ptr -> ref is functionally identical to ref -> ptr. We access `army.units` through a pointer, and `state.effects` through a pointer, but that's interleaved with a regular `&mut army` to call `reset_speed` and check a value in `army.base_units`. In the blog post I linked earlier, she says that "*generally* you should avoid mixing references and unsafe pointers". So uh, that doesn't quite give a firm yes or no either, and somewhat contradicts the official docs. I think this has to do with stacked borrows (i.e. how miri currently tracks memory access), but I'm not going to get into that here because it's a whole can of worms.

But what does accessing "the same memory" mean exactly? For that matter, what does "access" mean? It's easy to say "a read or a write", but what does *that* mean? If I pass `&mut army` to a method, does that access the pointer? In my eyes, unless the method is dynamically dispatched I don't think anything physically in memory is being read or written, and that's what matters according to the Rust and LLVM docs. What about accessing a *field* of `army`? Does that count as reading the whole struct, or just the field? In a local scope it only accesses the field, but is that a special compiler trick?

The opinion of the community members I asked was that `&mut army` effectively *does* count as accessing all of `army`, regardless of how it's used in the function. This was explained via [split_at_mut](https://doc.rust-lang.org/nomicon/borrow-splitting.html), and how it returns 2 *new* objects at the same time, thus only ever requiring 1 borrow of the original slice. The more I think about that, the less I agree with it though. I think that it works that way so that it is 100% safe - the intention of `split_at_mut` is that you cannot access the original slice while you're accessing the 2 halves of it. But not being 100% safe doesn't immediately mean UB.

Another thread we can investigate is the rust reference's section on [Behavior Considered Undefined](https://doc.rust-lang.org/reference/behavior-considered-undefined.html). It's not exhaustive, but still has some new info we haven't covered yet. To quickly run down the list:

* We have no data races because it's single threaded
* The pointers are made from valid refs, so they cannot be dangling or misaligned
* We're accessing fields through regular `.` syntax, so we cannot violate in-place pointer arithmetic rules
* We *might* be breaking LLVM's pointer aliasing rules, especially since we're not using `UnsafeCell`, but I'll talk about this one in more detail below
* We *might* be mutating "immutable" bytes? This has to do with "pointer provenance transitivity" (holy hell), which I'll get into later
* We're not using compiler intrinsics
* We're not cross compiling or using architecture-specific features
* We're not interacting with any external functions, so there's nowhere to use the wrong ABI in a function call
* We're not producing any invalid values
* We're not using inline assembly
* This is not a const context

### LLVM's rules

The two rules that we might be breaking depend on what it means for a pointer value is "based on" another pointer value. The main statement we're worried about is [this](https://llvm.org/docs/LangRef.html#pointer-aliasing-rules):

> Pointer Aliasing Rules: Any memory access must be done through a pointer value associated with an address range of the memory access, otherwise the behavior is undefined... A pointer value is associated with the addresses associated with any value it is *based* on... A pointer value formed from a scalar `getelementptr` operation is *based* on the pointer-typed operand of the `getelementptr`... The "*based on*" relationship is transitive.

The wording of "based on" is a bit obtuse, but I assume it's the same thing as what the Rust docs call ["provenance"](https://doc.rust-lang.org/std/ptr/index.html#provenance), and it involves the (apparently) infamous [getelementptr](https://llvm.org/docs/LangRef.html#getelementptr-instruction) LLVM instruction. The reference straight up says that the instruction is very confusing, and has [dedicated documentation to clarify things](https://llvm.org/docs/GetElementPtr.html) that is *also* very confusing. To be fair, that's not LLVM's fault, it's mine for digging into this without the requisite contextual knowledge.

The one hitch in this is that the Rust docs talk about "shrinking" provenance, which seems to somewhat contradict the LLVM docs' "associated with the addresses associated with any value it is based on":

> Provenance is implicitly shared with all pointers transitively derived from The Original Pointer through operations like offset, borrowing, and pointer casts. Some operations may shrink the derived provenance, limiting how much memory it can access or how long it’s valid for (i.e. borrowing a subfield and subslicing).

To sum up my understanding of this, `&mut foo.bar` counts as being associated with all of `&mut foo`, since the operand of `getelementptr` is the pointer to `foo`. Any subset pointer (e.g. `&mut bar.baz`) is also associated with all of `&mut foo` due to transitivity. **But** GEP-ing a field restricts the memory range that pointer the is allowed to access to some subset of `foo`. I read this as meaning that accessing data within `foo` via those pointers *at all* is valid, but it's invalid to take a pointer to `foo2` and offset it such that writing to it will write to `foo.bar`.

As far as I can tell, this says nothing about how and when the pointed-to data can be accessed if you have multiple pointers to that same data, only that LLVM expects to be able to "know" where each pointer came from, and that its access is in-bounds. In strict terms, [GEP is just pointer arithmetic](https://llvm.org/docs/GetElementPtr.html#what-is-dereferenced-by-gep), *not* an access. To put it in a more intuitive way, I think LLVM expects to be able to "undo" any pointer arithmetic to figure out what the base pointer is. If I understand correctly, LLVM doesn't have the concept of types or objects when referring to pointers, only addresses and access ranges, so I *think* this logic tracks.

The next hint we get is from the [LLVM's documentation of `noalias`](https://llvm.org/docs/LangRef.html#noalias) which states:

> Memory locations accessed via pointer values *based* on the argument or return value are not also accessed, during the execution of the function, via pointer values not *based* on the argument or return value. This guarantee only holds for memory locations that are *modified*...The attribute on a return value also has additional semantics...On function return values, the `noalias` attribute indicates that the function acts like a system memory allocation function, returning a pointer to allocated storage disjoint from the storage of any other object accessible to the caller.

This specifically deals with argument and return pointers, and that mutably accessing data is allowed (even under `noalias`) so long as it's only ever accessed by pointers *based on* the argument/return value. Honestly, the wording on this is pretty bad. I have 2 mutable pointers, and they are both clearly based on the same `self.army`, and (in theory) both *could* access the same data. Remember when I said semantics hell? Prepare yourself, this one's a disaster. LLVM's wording can be read in 2 different ways:

* Pessimistically, if a `noalias` pointer is passed into a function, any and all accesses to those bits must be made from pointers that are GEP'd from the passed-in-pointer *within the function* (i.e. nothing else passed in can access those bits, and no globals can either)
* Optimistically, if 2 `noalias` pointers are passed into a function and both point to the same memory, accesses are fine so long as they are both "obviously" based on the same pointer

Part of the reason this ambiguity exists is because LLVM docs say that `noalias` is "intentionally **similar to** the definition of `restrict` in C99". C99 `restrict` is what allows the compiler to cache memory reads when multiple pointers are passed to a function. If it said it was "identical to", I could for sure say "okay, the pessimistic take is the correct one". Instead, well... I don't know. The optimistic take *could* be true if LLVM is smart enough to do meta-analysis on individual call-sites and ignore the `noalias` declaration (and thus `noalias`-based optimizations) where appropriate. That *could* be a ridiculous line of reasoning, but to me it doesn't sound that farfetched considering how explicit the "based on" relationship is when using GEP, and this [whole article](https://www.ralfj.de/blog/2022/04/11/provenance-exposed.html) that talks about how the "based on" relationship *is* tracked to some degree, and that certain operations can muddy that tracking. It all depends on if `noalias` is treated as an invariant guaranteed by the programmer, or as a hint that allows *potential* optimization.

Finally, we can look at the [definitions of the `!noalias` and `!alias.scope`](https://llvm.org/docs/LangRef.html#noalias-and-alias-scope-metadata) decorators in LLVM's intermediate representation:

>`!noalias` and `!alias.scope` metadata provide the ability to specify generic noalias memory-access sets. This means that some collection of memory access instructions (loads, stores, memory-accessing calls, etc.) that carry `!noalias` metadata can specifically be specified not to alias with some other collection of memory access instructions that carry `!alias.scope` metadata.

What this allows is the defining of selective aliasing of non-argument/return value pointers. Adding these attributes to a pointer is creates the fine-grained control of "this pointer might alias with any other pointer in the program, but it *can* be guaranteed not to alias with *this other pointer* (or set of pointers) right here".

To sum all of this up:

* Pointers, according to LLVM, are an address and the number of bytes they can access
* GEP (i.e. getting a reference to a struct's field from a reference to a struct) is just pointer arithmetic and does not count as a read or write
* All of the below rules are ~irrelevant if you only ever access via 1 pointer
* (assuming the conservative interpretation) `noalias` on a pointer passed into a function assumes the pointed-to-value is not modified via anything other the passed-in-pointer **during the function**
* `noalias` on a return pointer means that the *caller* assumes the pointer is completely unique (and thus cannot be accessed via any other previously existing pointer)
* LLVM does \<some form of meta analysis\> that tracks pointers at least a little bit, and applies `!noalias` and `!alias.scope` within (and across) functions to make explicit aliasing relationships

### LLVM IR

All of the above information is very cool, but none of it means anything if we don't know how Rust is communicating with LLVM, and thus how LLVM sees our code. Even looking at the raw assembly won't help, since these are invariants that are reflected through Rust semantics and LLVM-IR decorators. Luckily, `cargo-show-asm` also has output for both LLVM-IR and pre-pass LLVM-IR. The latter is essentially before any optimizations or code transformations, and it doesn't contain any `!noalias` or `!alias.scope` decorators. While we can view the `reset_speed` and `tick_effects` functions both pre- and post-pass, `unsafe_mut` doesn't actually *do* anything and thus is optimized out (even with `#[inline(never)]`). That means we can only view it pre-pass:

```llvm
define internal noundef align 8 dereferenceable(24) ptr @sc2_sim::utils::unsafe_mut(ptr noalias noundef align 8 dereferenceable(24) %val) unnamed_addr #7 {
start:
  %_2 = alloca [8 x i8], align 8
  call void @llvm.lifetime.start.p0(i64 8, ptr %_2)
  %_5 = ptrtoint ptr %val to i64
  %0 = icmp eq i64 %_5, 0
  br i1 %0, label %bb1, label %bb2

bb1:                                              ; preds = %start
; call core::option::unwrap_failed
  call void @core::option::unwrap_failed(ptr noalias noundef readonly align 8 dereferenceable(24) @alloc_f8a1d20999d2bdc92f370757a18e4787) #23
  unreachable

bb2:                                              ; preds = %start
  store ptr %val, ptr %_2, align 8
  %_0 = load ptr, ptr %_2, align 8, !nonnull !2, !align !4, !noundef !2
  call void @llvm.lifetime.end.p0(i64 8, ptr %_2)
  ret ptr %_0
}
```

Wow, that certainly is some syntax...

> Aside: The blog-builder I use didn't even have highlighting for it, and there isn't a sublime-syntax for it that I could find in the wild. I tried some vscode extensions to get a feel for what it might look like, but those could only really manage "everything's a variable or a keyword" which is... unhelpful. Syntax highlighting should at least hint at the relationships between various tokens. In the end I just went ahead and wrote a sublime-syntax file myself that hopefully isn't quite so hideous.

In particular, I want to point out 2 things. First is that the returned value is literally a copy of the passed in value.

```llvm
; after this, metadata tags are added and it's "copied" to the
; memory address of the return value, but this is where the value
; comes from. It is NOT an inttoptr or ptrtoint
store ptr %val, ptr %_2, align 8
```

I'm not 100% sure if this (and/or this getting optimized out) ends up erasing the "based on" relationship of the original pointer somewhere along the line, but I don't think so.

Second, and more importantly, is the return type of the function:

```llvm
noundef align 8 dereferenceable(24) ptr
```

You'll notice that it does *not* have a `noalias` tag. That means we can ignore the return position `noalias` restrictions entirely. Also, the `dereferenceable(24)` refers to the number of bytes that the pointer is allowed to legally access. It's of size 24 because both of our `unsafe_mut` calls are references to vecs.

Before getting to the post-pass, it's worth noting that this is what a call to `into_iter` looks like in LLVM-IR:

```llvm
%0 = call { ptr, ptr }
    @"<&mut alloc::vec::Vec<T,A> as core::iter::traits::collect::IntoIterator>::into_iter"(
        ptr noalias noundef align 8 dereferenceable(24) %units
    )
```

All it does is return pointers to the first element, and 1 past the last element of the passed-in array. `army.units` is considered noalias inside the function of `into_iter`, but *the return pointers are not*. That means LLVM does *not* expect `state` to be inaccessible by other existing pointers.

<details>
<summary>Full tick_effects LLVM-IR</summary>

```llvm
; sc2_sim::coordinator::Coordinator::tick_effects
; Function Attrs: noinline uwtable
define internal fastcc void @sc2_sim::coordinator::Coordinator::tick_effects(ptr noalias nocapture noundef readonly align 8 dereferenceable(336) %self) unnamed_addr #2 {
start:
  %_3 = getelementptr inbounds i8, ptr %self, i64 328 ; offset self + time
  tail call void @llvm.experimental.noalias.scope.decl(metadata !679)
  %0 = getelementptr inbounds i8, ptr %self, i64 8 ; offset red + units heap ptr?
  %units.val.i = load ptr, ptr %0, align 8, !alias.scope !679, !nonnull !5, !noundef !5 ; deref to load units heap ptr?
  %1 = getelementptr inbounds i8, ptr %self, i64 16 ; offset red + units.len?
  %units.val6.i = load i64, ptr %1, align 8, !alias.scope !679, !noundef !5 ; deref/copy units.len
  %2 = getelementptr inbounds %"army::State", ptr %units.val.i, i64 %units.val6.i ; get a pointer to 1 past the end of the units.data
  %3 = icmp eq i64 %units.val6.i, 0 ; if len = 0, exit
  br i1 %3, label %"sc2_sim::coordinator::Coordinator::tick_effects::{{closure}}.exit", label %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i.preheader"

"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i.preheader": ; preds = %start
  %_26.i = load i32, ptr %_3, align 8 ; load current time
  br label %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i"

bb3.loopexit.i:                                   ; preds = %bb15.i, %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i"
  %4 = icmp eq ptr %_14.i.i.i, %2 ; if current == end, exit
  br i1 %4, label %"sc2_sim::coordinator::Coordinator::tick_effects::{{closure}}.exit", label %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i"

"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i": ; preds = %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i.preheader", %bb3.loopexit.i
  %iter.sroa.0.08.i = phi ptr [ %_14.i.i.i, %bb3.loopexit.i ], [ %units.val.i, %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i.preheader" ]
  %_14.i.i.i = getelementptr inbounds %"army::State", ptr %iter.sroa.0.08.i, i64 1 ; index into units
  %5 = getelementptr inbounds i8, ptr %iter.sroa.0.08.i, i64 16 ; get state
  %_154.i = load i64, ptr %5, align 8, !noalias !679, !noundef !5 ; deref effect type?
  %_135.not.i = icmp eq i64 %_154.i, 0 ; if effect type isn't right, continue, else start the calculations
  br i1 %_135.not.i, label %bb3.loopexit.i, label %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.lr.ph.i"

"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.lr.ph.i": ; preds = %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i"
  %6 = getelementptr i8, ptr %iter.sroa.0.08.i, i64 8 ; offset unit + effects.data ptr
  br label %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.i"

"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.i": ; preds = %bb15.i, %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.lr.ph.i"
  %_159.i = phi i64 [ %_154.i, %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.lr.ph.i" ], [ %_15.i, %bb15.i ]
  %i.sroa.0.06.i = phi i64 [ 0, %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.lr.ph.i" ], [ %i.sroa.0.1.i, %bb15.i ]
  %.val.i = load ptr, ptr %6, align 8, !noalias !679, !nonnull !5, !noundef !5 ; load effects.data heap addr
  %_0.i.i.i = getelementptr inbounds [0 x %"effect::Effect"], ptr %.val.i, i64 0, i64 %i.sroa.0.06.i ; index effects
  %7 = load i32, ptr %_0.i.i.i, align 8, !range !405, !noalias !679, !noundef !5 ; load effects discriminant
  %trunc.not.not.i = icmp eq i32 %7, 0 ; if discriminant == 0, process, else continue
  br i1 %trunc.not.not.i, label %bb11.i, label %bb12.i

bb12.i:                                           ; preds = %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.i"
  %8 = getelementptr inbounds i8, ptr %_0.i.i.i, i64 4 ; offset effects.timestamp
  %timestamp.i = load i32, ptr %8, align 4, !noalias !679, !noundef !5 ; deref timestamp
  %switch5.i = icmp sgt i32 %timestamp.i, %_26.i ; if timestamp is lte time, process, else continue
  br i1 %switch5.i, label %bb17.i, label %"alloc::vec::Vec<T,A>::swap_remove.exit.i"

bb11.i:                                           ; preds = %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.i"
  %9 = add nuw i64 %i.sroa.0.06.i, 1 ; i += 1
  br label %bb15.i

"alloc::vec::Vec<T,A>::swap_remove.exit.i": ; preds = %bb12.i
  tail call void @llvm.experimental.noalias.scope.decl(metadata !682)
  %count.i.i = add i64 %_159.i, -1
  %_11.i.i = getelementptr inbounds %"effect::Effect", ptr %.val.i, i64 %count.i.i
  tail call void @llvm.memmove.p0.p0.i64(ptr noundef nonnull align 8 dereferenceable(16) %_0.i.i.i, ptr noundef nonnull align 8 dereferenceable(16) %_11.i.i, i64 16, i1 false), !noalias !685
  store i64 %count.i.i, ptr %5, align 8, !alias.scope !682, !noalias !687
; call sc2_sim::army::Army::reset_speed
  tail call void @sc2_sim::army::Army::reset_speed(ptr noalias noundef nonnull align 8 dereferenceable(144) %self, ptr noalias noundef nonnull align 8 dereferenceable(88) %iter.sroa.0.08.i)
  %_15.pre.i = load i64, ptr %5, align 8, !noalias !679
  br label %bb15.i

bb17.i:                                           ; preds = %bb12.i
  %10 = add nuw i64 %i.sroa.0.06.i, 1
  br label %bb15.i

bb15.i:                                           ; preds = %bb17.i, %"alloc::vec::Vec<T,A>::swap_remove.exit.i", %bb11.i
  %_15.i = phi i64 [ %_159.i, %bb11.i ], [ %_159.i, %bb17.i ], [ %_15.pre.i, %"alloc::vec::Vec<T,A>::swap_remove.exit.i" ]
  %i.sroa.0.1.i = phi i64 [ %9, %bb11.i ], [ %10, %bb17.i ], [ %i.sroa.0.06.i, %"alloc::vec::Vec<T,A>::swap_remove.exit.i" ]
  %_13.i = icmp ult i64 %i.sroa.0.1.i, %_15.i
  br i1 %_13.i, label %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.i", label %bb3.loopexit.i

"sc2_sim::coordinator::Coordinator::tick_effects::{{closure}}.exit": ; preds = %bb3.loopexit.i, %start
  %_11 = getelementptr inbounds i8, ptr %self, i64 144
  tail call void @llvm.experimental.noalias.scope.decl(metadata !688)
  %11 = getelementptr inbounds i8, ptr %self, i64 152
  %units.val.i2 = load ptr, ptr %11, align 8, !alias.scope !688, !nonnull !5, !noundef !5
  %12 = getelementptr inbounds i8, ptr %self, i64 160
  %units.val6.i3 = load i64, ptr %12, align 8, !alias.scope !688, !noundef !5
  %13 = getelementptr inbounds %"army::State", ptr %units.val.i2, i64 %units.val6.i3
  %14 = icmp eq i64 %units.val6.i3, 0
  br i1 %14, label %"sc2_sim::coordinator::Coordinator::tick_effects::{{closure}}.exit32", label %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i5.preheader"

"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i5.preheader": ; preds = %"sc2_sim::coordinator::Coordinator::tick_effects::{{closure}}.exit"
  %_26.i19 = load i32, ptr %_3, align 8
  br label %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i5"

bb3.loopexit.i29:                                 ; preds = %bb15.i25, %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i5"
  %15 = icmp eq ptr %_14.i.i.i7, %13
  br i1 %15, label %"sc2_sim::coordinator::Coordinator::tick_effects::{{closure}}.exit32", label %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i5"

"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i5": ; preds = %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i5.preheader", %bb3.loopexit.i29
  %iter.sroa.0.08.i6 = phi ptr [ %_14.i.i.i7, %bb3.loopexit.i29 ], [ %units.val.i2, %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i5.preheader" ]
  %_14.i.i.i7 = getelementptr inbounds %"army::State", ptr %iter.sroa.0.08.i6, i64 1
  %16 = getelementptr inbounds i8, ptr %iter.sroa.0.08.i6, i64 16
  %_154.i8 = load i64, ptr %16, align 8, !noalias !688, !noundef !5
  %_135.not.i9 = icmp eq i64 %_154.i8, 0
  br i1 %_135.not.i9, label %bb3.loopexit.i29, label %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.lr.ph.i10"

"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.lr.ph.i10": ; preds = %"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next.exit.i5"
  %17 = getelementptr i8, ptr %iter.sroa.0.08.i6, i64 8
  br label %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.i11"

"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.i11": ; preds = %bb15.i25, %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.lr.ph.i10"
  %_159.i12 = phi i64 [ %_154.i8, %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.lr.ph.i10" ], [ %_15.i26, %bb15.i25 ]
  %i.sroa.0.06.i13 = phi i64 [ 0, %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.lr.ph.i10" ], [ %i.sroa.0.1.i27, %bb15.i25 ]
  %.val.i14 = load ptr, ptr %17, align 8, !noalias !688, !nonnull !5, !noundef !5
  %_0.i.i.i15 = getelementptr inbounds [0 x %"effect::Effect"], ptr %.val.i14, i64 0, i64 %i.sroa.0.06.i13
  %18 = load i32, ptr %_0.i.i.i15, align 8, !range !405, !noalias !688, !noundef !5
  %trunc.not.not.i16 = icmp eq i32 %18, 0
  br i1 %trunc.not.not.i16, label %bb11.i31, label %bb12.i17

bb12.i17:                                         ; preds = %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.i11"
  %19 = getelementptr inbounds i8, ptr %_0.i.i.i15, i64 4
  %timestamp.i18 = load i32, ptr %19, align 4, !noalias !688, !noundef !5
  %switch5.i20 = icmp sgt i32 %timestamp.i18, %_26.i19
  br i1 %switch5.i20, label %bb17.i30, label %"alloc::vec::Vec<T,A>::swap_remove.exit.i21"

bb11.i31:                                         ; preds = %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.i11"
  %20 = add nuw i64 %i.sroa.0.06.i13, 1
  br label %bb15.i25

"alloc::vec::Vec<T,A>::swap_remove.exit.i21": ; preds = %bb12.i17
  tail call void @llvm.experimental.noalias.scope.decl(metadata !691)
  %count.i.i22 = add i64 %_159.i12, -1
  %_11.i.i23 = getelementptr inbounds %"effect::Effect", ptr %.val.i14, i64 %count.i.i22
  tail call void @llvm.memmove.p0.p0.i64(ptr noundef nonnull align 8 dereferenceable(16) %_0.i.i.i15, ptr noundef nonnull align 8 dereferenceable(16) %_11.i.i23, i64 16, i1 false), !noalias !694
  store i64 %count.i.i22, ptr %16, align 8, !alias.scope !691, !noalias !696
; call sc2_sim::army::Army::reset_speed
  tail call void @sc2_sim::army::Army::reset_speed(ptr noalias noundef nonnull align 8 dereferenceable(144) %_11, ptr noalias noundef nonnull align 8 dereferenceable(88) %iter.sroa.0.08.i6)
  %_15.pre.i24 = load i64, ptr %16, align 8, !noalias !688
  br label %bb15.i25

bb17.i30:                                         ; preds = %bb12.i17
  %21 = add nuw i64 %i.sroa.0.06.i13, 1
  br label %bb15.i25

bb15.i25:                                         ; preds = %bb17.i30, %"alloc::vec::Vec<T,A>::swap_remove.exit.i21", %bb11.i31
  %_15.i26 = phi i64 [ %_159.i12, %bb11.i31 ], [ %_159.i12, %bb17.i30 ], [ %_15.pre.i24, %"alloc::vec::Vec<T,A>::swap_remove.exit.i21" ]
  %i.sroa.0.1.i27 = phi i64 [ %20, %bb11.i31 ], [ %21, %bb17.i30 ], [ %i.sroa.0.06.i13, %"alloc::vec::Vec<T,A>::swap_remove.exit.i21" ]
  %_13.i28 = icmp ult i64 %i.sroa.0.1.i27, %_15.i26
  br i1 %_13.i28, label %"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index.exit.i11", label %bb3.loopexit.i29

"sc2_sim::coordinator::Coordinator::tick_effects::{{closure}}.exit32": ; preds = %bb3.loopexit.i29, %"sc2_sim::coordinator::Coordinator::tick_effects::{{closure}}.exit"
  ret void
}
```
</details>
<br/>

Next we'll want to check the callsites of the functions that consume refs based on unsafe refs - `reset_speed` and `effect.apply`

```llvm
tail call void @sc2_sim::army::Army::reset_speed(
    ptr noalias noundef nonnull align 8 dereferenceable(144) %_11,
    ptr noalias noundef nonnull align 8 dereferenceable(88) %iter.sroa.0.08.i6
)
```

Both the pointer to `army` (`%_11`) and the pointer to `state` (`%iter.sroa.0.08.i6`) are marked as `noalias`, which seems like it might be an issue. If you consider it carefully, this doesn't actually violate LLVM's expectations. According to LLVM `&mut army` is simply a pointer to 144 bytes, and `&mut state` is a pointer to 88 bytes. We, as the programmer, know that these bytes do not overlap in any way, since `&mut state` is a pointer to a heap-allocated array element, whereas `coordinator` (and thus `army`) is on the stack. While `state` was originally accessed via an overlapping pointer to `army.units`, we have arranged things in such a way that no *writes* are made to any of the bytes that a pointer to `army.units` would imply (i.e. the 24 byte footprint of the vec itself) until control returns back to `tick_effects`.

<details>
<summary>Full reset_speed LLVM-IR</summary>

```llvm
; sc2_sim::army::Army::reset_speed
; Function Attrs: noinline uwtable
define void @sc2_sim::army::Army::reset_speed(ptr noalias nocapture noundef readonly align 8 dereferenceable(144) %self, ptr noalias noundef align 8 dereferenceable(88) %state) unnamed_addr #2 personality ptr @__CxxFrameHandler3 {
start:
  %_6 = getelementptr inbounds i8, ptr %state, i64 86 ; offset state + base
  %_6.val = load i8, ptr %_6, align 2 ; deref base
  tail call void @llvm.experimental.noalias.scope.decl(metadata !491)
  %0 = getelementptr inbounds i8, ptr %self, i64 120 ; offset self + base_units
  %_12.i = load i64, ptr %0, align 8, !alias.scope !491, !noundef !5 ; load base_units len
  %1 = icmp eq i64 %_12.i, 0 ; if len == 0, return
  br i1 %1, label %bb11, label %bb6.i

; hash function
bb6.i:                                            ; preds = %start
  %_5 = getelementptr inbounds i8, ptr %self, i64 96
  %_3.i.i.i.i = zext nneg i8 %_6.val to i64
  %_5.i.i.i.i = xor i64 %_3.i.i.i.i, 1376283091369227076
  %_8.i.i.i.i = zext nneg i64 %_5.i.i.i.i to i128
  %_7.i.i.i.i = mul nuw nsw i128 %_8.i.i.i.i, 6364136223846793005
  %_12.i.i.i.i = lshr i128 %_7.i.i.i.i, 64
  %_41.i.i.i.i = xor i128 %_12.i.i.i.i, %_7.i.i.i.i
  %_4.i.i.i.i = trunc i128 %_41.i.i.i.i to i64
  %_8.i.i.i = and i128 %_41.i.i.i.i, 18446744073709551615
  %_7.i.i.i = mul nuw nsw i128 %_8.i.i.i, 2611923443488327891
  %_13.i.i.i = lshr i128 %_7.i.i.i, 64
  %_51.i.i.i = xor i128 %_13.i.i.i, %_7.i.i.i
  %_5.i.i.i = trunc i128 %_51.i.i.i to i64
  %2 = tail call noundef i64 @llvm.fshl.i64(i64 %_5.i.i.i, i64 %_5.i.i.i, i64 %_4.i.i.i.i)
  %self.val.i = load ptr, ptr %_5, align 8, !alias.scope !494, !noalias !497, !nonnull !5, !noundef !5
  %3 = getelementptr inbounds i8, ptr %self, i64 104
  %self.val1.i = load i64, ptr %3, align 8, !alias.scope !499, !noalias !497, !noundef !5
  %_18.i.i.i.i = lshr i64 %2, 57
  %h2_hash.i.i.i.i = trunc i64 %_18.i.i.i.i to i8
  %.sroa.0.0.vec.insert.i.i.i.i = insertelement <16 x i8> poison, i8 %h2_hash.i.i.i.i, i64 0
  %.sroa.0.15.vec.insert.i.i.i.i = shufflevector <16 x i8> %.sroa.0.0.vec.insert.i.i.i.i, <16 x i8> poison, <16 x i32> zeroinitializer
  %invariant.gep.i.i.i = getelementptr { i8, [7 x i8], %"unit::Unit" }, ptr %self.val.i, i64 -1
  br label %bb1.i.i.i.i

bb1.i.i.i.i:                                      ; preds = %bb11.i.i.i.i, %bb6.i
  %hash.pn.i.i = phi i64 [ %2, %bb6.i ], [ %12, %bb11.i.i.i.i ]
  %probe_seq1.sroa.0.0.i.i.i.i = phi i64 [ 0, %bb6.i ], [ %11, %bb11.i.i.i.i ]
  %probe_seq.sroa.0.0.i.i.i.i = and i64 %hash.pn.i.i, %self.val1.i
  %_5.i.i.i3.i = getelementptr inbounds i8, ptr %self.val.i, i64 %probe_seq.sroa.0.0.i.i.i.i
  %dst.sroa.0.0.copyload.i29.i.i.i = load <16 x i8>, ptr %_5.i.i.i3.i, align 1, !noalias !502
  %4 = icmp eq <16 x i8> %dst.sroa.0.0.copyload.i29.i.i.i, %.sroa.0.15.vec.insert.i.i.i.i
  %5 = bitcast <16 x i1> %4 to i16
  br label %bb2.i.i.i.i

bb2.i.i.i.i:                                      ; preds = %bb5.i.i.i.i, %bb1.i.i.i.i
  %iter.i.sroa.0.0.i.i.i = phi i16 [ %5, %bb1.i.i.i.i ], [ %_13.i.i.i.i, %bb5.i.i.i.i ]
  %6 = icmp eq i16 %iter.i.sroa.0.0.i.i.i, 0
  br i1 %6, label %bb6.i.i.i.i, label %bb5.i.i.i.i

bb6.i.i.i.i:                                      ; preds = %bb2.i.i.i.i
  %7 = icmp eq <16 x i8> %dst.sroa.0.0.copyload.i29.i.i.i, <i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1, i8 -1>
  %8 = bitcast <16 x i1> %7 to i16
  %9 = icmp eq i16 %8, 0
  br i1 %9, label %bb11.i.i.i.i, label %bb11

bb5.i.i.i.i:                                      ; preds = %bb2.i.i.i.i
  %10 = tail call i16 @llvm.cttz.i16(i16 %iter.i.sroa.0.0.i.i.i, i1 true), !range !63
  %_9.i.i.i.i = zext nneg i16 %10 to i64
  %_14.i2.i.i.i = add i16 %iter.i.sroa.0.0.i.i.i, -1
  %_13.i.i.i.i = and i16 %_14.i2.i.i.i, %iter.i.sroa.0.0.i.i.i
  %_14.i.i.i.i = add i64 %probe_seq.sroa.0.0.i.i.i.i, %_9.i.i.i.i
  %index.i.i.i.i = and i64 %_14.i.i.i.i, %self.val1.i
  %_11.i.i.i.i.i = sub nsw i64 0, %index.i.i.i.i
  %gep.i.i.i = getelementptr { i8, [7 x i8], %"unit::Unit" }, ptr %invariant.gep.i.i.i, i64 %_11.i.i.i.i.i
  %.val.i.i.i.i = load i8, ptr %gep.i.i.i, align 1, !range !64, !noalias !510, !noundef !5
  %_0.i.i.i.i.i.i.i = icmp eq i8 %.val.i.i.i.i, %_6.val
  br i1 %_0.i.i.i.i.i.i.i, label %bb12, label %bb2.i.i.i.i

bb11.i.i.i.i:                                     ; preds = %bb6.i.i.i.i
  %11 = add i64 %probe_seq1.sroa.0.0.i.i.i.i, 16
  %12 = add i64 %11, %probe_seq.sroa.0.0.i.i.i.i
  br label %bb1.i.i.i.i

bb11:                                             ; preds = %bb6.i.i.i.i, %start
; call core::option::expect_failed
  tail call void @core::option::expect_failed(ptr noalias noundef nonnull readonly align 1 @alloc_a7d7769291fb702cd4ca6ef64f027ff9, i64 noundef 22, ptr noalias noundef nonnull readonly align 8 dereferenceable(24) @alloc_b3f0fcdaf69e41a74e6f9070f90e708e) #28
  unreachable

bb12:                                             ; preds = %bb5.i.i.i.i
  %13 = getelementptr inbounds { i8, [7 x i8], %"unit::Unit" }, ptr %self.val.i, i64 %_11.i.i.i.i.i
  %14 = getelementptr { i8, [7 x i8], %"unit::Unit" }, ptr %13, i64 -1, i32 2, i32 5
  %speed = load i32, ptr %14, align 4, !noundef !5
  %15 = getelementptr inbounds i8, ptr %state, i64 64
  store i32 %speed, ptr %15, align 8
  %16 = getelementptr inbounds i8, ptr %state, i64 8
  %state.val = load ptr, ptr %16, align 8, !nonnull !5, !noundef !5
  %17 = getelementptr inbounds i8, ptr %state, i64 16
  %state.val1 = load i64, ptr %17, align 8, !noundef !5
  %_17.i = getelementptr inbounds %"effect::Effect", ptr %state.val, i64 %state.val1
  %18 = icmp eq i64 %state.val1, 0
  br i1 %18, label %bb6, label %bb5

bb6:                                              ; preds = %bb2.backedge, %bb12
  ret void

bb5:                                              ; preds = %bb12, %bb2.backedge
  %iter.sroa.0.010 = phi ptr [ %_14.i.i, %bb2.backedge ], [ %state.val, %bb12 ]
  %_14.i.i = getelementptr inbounds %"effect::Effect", ptr %iter.sroa.0.010, i64 1
  %19 = load i32, ptr %iter.sroa.0.010, align 8, !range !405, !noundef !5
  %trunc.not.not = icmp eq i32 %19, 0
  br i1 %trunc.not.not, label %bb2.backedge, label %bb7

bb7:                                              ; preds = %bb5
  %20 = getelementptr inbounds i8, ptr %iter.sroa.0.010, i64 8
  %_17 = load ptr, ptr %20, align 8, !nonnull !5, !noundef !5
  tail call void %_17(ptr noalias noundef nonnull align 8 dereferenceable(88) %state)
  br label %bb2.backedge

bb2.backedge:                                     ; preds = %bb7, %bb5
  %21 = icmp eq ptr %_14.i.i, %_17.i
  br i1 %21, label %bb6, label %bb5
}
```
</details>
<br/>

```llvm
tail call void %_17(ptr noalias noundef nonnull align 8 dereferenceable(88) %state)
```

Same deal for `effect.apply`. `state` is not accessed by any other pointer during the execution of `apply`, it doesn't overlap with the `effects` iterator (which is a pointer to a different allocation on the heap), and the `effects` footprint-bytes are not written to.

## Speculation

If everything above is accurate and I'm not misunderstanding anything, I think my exact usage is *not* UB under any circumstance. Even assuming the most pessimistic view - that accessing `&mut foo.baz` still counts as "overlapping access" with every other part of `&mut foo` - it's clear that any access to `&mut army` from within `reset_speed` does not overlap with `&mut state` in any way. `&mut state` is allocated on the heap, completely disjoint from `&mut army`. I would have to interleave access via `&mut state` with accessing the same memory via `&mut army.units[i]` or resize the vec for there to be any real problems. To generalize, for any struct

```rust
struct S {
    a: T,
    b: &mut T, // or *mut T, if it does not point to self.a
}

let s: S = ...;
```

`s.method(unsafe_mut(&s.b))` should *not* result in UB so long as you don't access `b` through `&mut s` inside the method. If `b` is a pointer to a vec, and the unsafe reference passed in is `unafe_mut(&mut s.b[i])`, accessing `b`, and even `b[x]` through `&mut s` within the method should be fine, but only so long as `b` never changes sizes (which may result in a reallocation) **and** `x != i`. This is at least partially supported via miri, which *only* errors out when I intentionally interleave access to a value between `&mut self` and the `unsafe_mut` reference.

I don't think it's as straight forward if `b` is just a regular, non-pointer value. We're technically violating an assumption - the pointers that we're passing in *do* overlap - but accesses are what matter, not pointers, so as long as we're careful not to actually modify overlapping bits it should be fine. It all depends on what kinds of things Rust is doing in it's MIR - how it's extra invariants are expressed and how those are translated to LLVM-IR. Maybe it's 100% not UB and I'm overthinking it, who knows. I still don't feel like I entirely grasp the semantics behind `!alias.scope` and `!noalias`.

It's probably safe to say that if you call a function `s.method(unsafe_mut(&mut s.a))`, observing the pointed-to-value of `b` and even changing the pointer of `s.b` is fine, since accessing `s.b` is a "shrinking" GEP. That said, assigning to the whole struct (`*s = s2`) is very likely UB. There might be some exceptions to that if, for example, `s2.a == s.a` but uh... "does a write count as an aliasing access even if the bits written are the same as the bits that were already there?" is some next level semantics that I'm not even gonna touch.

Either way, it was fun looking into how LLVM works. There were a lot of cool realizations along the way - a lot more of Rust is "smoke and mirrors" meant to massage the LLVM-IR than I expected. If this trick isn't UB, it's probably pretty handy for prototyping. That's a pretty big "if" though.

I mentioned before that there's safe versions of my example functions, which I'll put in an expand-o down below. They use indexing rather than iterators so they don't hold long-term borrows. We also pass the `state` *handle* to `reset_speed` instead of `&mut state`. The ASM and LLVM_IR generated are more or less identical between the safe and unsafe versions, accounting for register shuffling and a bit of panic handling for the array indexing.

<details>
<summary>Safe tick_effects</summary>

```rust
#[inline(never)]
fn tick_effects(&mut self) {
    let mut _inner = |army: &mut Army| {
        for handle in 0..army.units.len() {
            let mut i = 0;

            while i < army.units[handle].effects.len() {
                match army.units[handle].effects[i] {
                    effect::Effect::StatModTemp {
                        stat: Stat::Speed,
                        apply: _,
                        timestamp,
                    } => {
                        if timestamp <= self.time {
                            army.units[handle].effects.swap_remove(i);
                            army.reset_speed(handle)
                        } else {
                            i += 1;
                        }
                    }
                    _ => i += 1,
                }
            }
        }
    };

    _inner(&mut self.red);
    _inner(&mut self.blu);
}
```
</details>
</br>
<details>
<summary>Safe reset_speed</summary>

```rust
pub fn reset_speed(&mut self, handle: usize) {
    let unit = &mut self.units[handle];
    let speed = self.base_units[&unit.base].movement.speed;
    unit.max_speed = speed;

    for i in 0..unit.effects.len() {
        match unit.effects[i] {
            Effect::StatModTemp {
                stat: Stat::Speed,
                apply,
                timestamp: _,
            } => {
                apply(unit);
            }
            _ => continue,
        }
    }
}
```
</details>