+++
title = "The space-time-complexity tradeoff"
date = "2024-03-09"
description = ""
[taxonomies]
tags=["programming", "rust",]
+++

I encountered a neat example recently while solving [Advent of Code 2015, day 6](https://adventofcode.com/2015/day/6). The problem can be boiled down to "There is a 2D array of values. Given a range and instruction, apply the instruction to all values in that range. How many values are 'on' at the end?". There are only 2 possible states for each value: on and off, and only 3 possible instructions: on, off, and toggle. I won't focus too much on parsing the input or the structure of the algorithm here, I just want to investigate the hot loop which applies the instruction to each value.

<!-- more -->

Code is compiled using `-Ctarget-cpu=native` for a Ryzen 1600x running Windows.

## Representation

Whenever I see a max of 2 states, my brain jumps straight to bit manipulation - why store in 8 bits what can be stored in 1 bit? The types of low-power, memory constrained platforms and programs I've investigated have given me lots of exposure bit fields. While that's what I'm comfortable with, it's often much closer to the "space" end of the space-time tradeoff. For something on the speedier side, we can represent the values as 8-bit bools.

While I could use an array of arrays, there's no real need to when working with uniform dimensions (1000x1000 in this case). The bool array can be represented as `[bool; 1000 * 1000]` and the bit array as `[u8; 125 * 1000]`, saving 875kb. While that sounds like an insignificant amount, it's worth noting that [my CPU](https://www.techpowerup.com/cpu-specs/ryzen-5-1600x.c1894) has an L1 data cache size of 32kb and an L2 cache size of 512kb. The bitwise implementation could reside entirely in the L2 cache, whereas the bool array necessitates pulling some data from L3. L3 access can be [somewhat expensive](https://www.7-cpu.com/cpu/Zen.html), so it could make a difference.

## Saving time

As a quick spoiler, the correct answer for my input data is `377891`. This blog post isn't really about solving the problem (nor the most optimal solution), but it's important to ensure that no matter what we change we're still getting the correct answer. Wrong code is still wrong no matter how fast it is.

That aside, after parsing the line into the instruction and a Coordinate struct representing the "top left" and "bottom right" corners of the affected area, we can construct a simple loop that traverses the range

```rust
for y in start.y..=end.y {
    for x in start.x..=end.x {
        todo!()
    }
}
```

For bools, the loop contents are very simple, just translate the 2D coordinate into a 1D array index. Also note that the nesting-order of the loop matters. Row-wise access is typically more performant due to locality. Simply reversing the order of the loop slows down execution time by about 30%.

```rust
for y in start.y..=end.y {
    for x in start.x..=end.x {
        array[y * 1000 + x] = true;

        // off:
        // array[y * 1000 + x] = false;
        // toggle:
        // array[y * 1000 + x] ^= true;
    }
}
```

To collect the values at the end, we can use an iter-fold:

```rust
array.iter().fold(0, |acc, x| acc + *x as usize)
```

And hey, we get the correct answer in only ~18ms.

## Saving Space

But what if we want to fiddle with bits? Well I've got 2 options in mind, and one is a bit spicy. The first is the simple bitwise operations we're used to, the only tricky part being that we need to index into both the array and the individual u8s at the same time. 2D -> 1D array access is `y * width + x`. but since our array width is effectively 8x less, we need to divide both sides by 8. To get the offset into the u8, we need the remainder of `x / 8`. We can then bitshift by that offset and set/clear/complement to apply the operation:

```rust
// fun fact, the compiler isn't smart enough to remove the
// bounds check on the array access, even with the assertions:
// assert!(start.y <= 999 && end.y <= 999);
// assert!(start.x <= 999 && end.x <= 999);

for y in start.y..=end.y {
    for x in start.x..=end.x {
        let idx = (y * 125) + (x / 8);
        let offset = x % 8;
        bit_array[idx] |= 1 << offset;

        // off:
        // bit_array[idx] &= 0xFF ^ (1 << offset);
        // toggle:
        // bit_array[idx] ^= 1 << offset;
    }
}
```

Collecting the values requires a slight modification of the iter-fold:

```rust
bit_array
    .iter()
    .fold(0, |acc, x| acc + x.count_ones() as usize)
```

This runs a bit slower at ~28ms.

So how about a little spice? This was my original solution:

```rust
let ptr = bit_array.as_mut_ptr();
for y in start.y..=end.y {
    for x in start.x..=end.x {
        let bit_idx = y * 1000 + x;
        // wew
        unsafe {
            asm!(
                "bts [{ptr}], {bit_idx}",
                // off:
                // "btr [{ptr}], {bit_idx}",
                // toggle:
                // "btc [{ptr}], {bit_idx}",
                bit_idx = in(reg) bit_idx,
                ptr = in(reg) ptr,
            )
        }
    }
}
```

I had just finished reading The Art of 64-bit Assembly, and the BT* instructions were fresh on my mind. For those that don't read ISAs in your free time, BTS, BTR, and BTC are *very* cool instructions. The mnemonics are short for "Bit Test and \<Set / Reset / Complement\>". You feed the instruction a memory address and a *bit* offset, and it sets it, clears it, or inverts it. It also sets the carry flag so you know what the bit was before you operated on it, and you can optionally make the instruction atomic.

It took me a few minutes to come up with - and convince myself of the correctness of - the indexing formula for the bitwise version. This version is the same level of cognitive complexity as the array of booleans, but with the space saving of the bitwise operations. Relative to other assembly instructions, I especially like how human-readable it is. I've spent a decent amount of time staring at PowerPC assembly, and the somewhat equivalent [RLWIMI](https://www.ibm.com/docs/ru/aix/7.2?topic=is-rlwimi-rlimi-rotate-left-word-immediate-then-mask-insert-instruction) and RLWIMN instructions require [several paragraphs](https://mariokartwii.com/showthread.php?tid=1262) to explain, and are *still* kinda incomprehensible at a glance. RLWIMI has some cool differences, such as being able to operate on a range of bits all at once, but in most cases I see it used to modify a single bit.

All that aside, we can once again run it and see that we get the correct result, and a runtime of ... 47ms? Almost twice as slow as the bitwise version? That can't be right. Lets pull the work out of the loops into their own functions and check the disassembly to see what's actually going on. Running `cargo-show-asm` on the bitwise function reveals:

```asm
aoc2015::day6::bit_on:
            // pub fn bit_on(bit_array: &mut [u8; 125 * 1000], x: usize, y: usize)
        sub rsp, 40
        mov r9, rcx
            // let idx = (y * 125) + (x / 8);
        imul rcx, r8, 125
        mov rax, rdx
        shr rax, 3
        add rax, rcx
            // bit_array[idx] |= 1 << offset;
        cmp rax, 124999
        ja .LBB222_2
        and dl, 7
        mov r8b, 1
        mov ecx, edx
        shl r8b, cl
        or byte ptr [r9 + rax], r8b
        add rsp, 40
        ret
.LBB222_2:
            // bit_array[idx] |= 1 << offset;
        lea r8, [rip + __unnamed_373]
        mov edx, 125000
        mov rcx, rax
        call core::panicking::panic_bounds_check
        ud2
```

And for the inline assembly function:

```asm
aoc2015::day6::asm_on:
                // pub fn asm_on(ptr: *mut u8, x: usize, y: usize) {
        push rax
                // let bit_idx = y * 1000 + x;
        imul rax, r8, 1000
        add rax, rdx
                // asm!(
        #APP

        bts qword ptr [rcx], rax

        #NO_APP
        pop rax
        ret
```

Ignoring the function setup/teardown/panic, the inline assembly is significantly shorter - just 3 instructions, no branches vs the 11 and never-taken branch of the bitwise function. So why's it so much slower?

I mentioned in the bool function that the nesting order of the loops was important for cache locality. What I didn't mention was that the equation changes when you're operating repeatedly on a single value, which the ASM and bitwise functions both frequently do. Modern CPUs are pipelined, meaning they're simultaneously executing several instructions at a time. A caveat of this is that any instruction that requires data from an instruction *currently in the pipeline* must wait until that instruction is completely finished before it can begin. This is called **instruction latency**. Stalling the pipeline like this can be disastrous for performance.

When we reverse the loop ordering, the bitwise function runs in ~26ms, essentially no change at all. On the other hand, the ASM function now executes in **30ms**, a massive speedup. This is likely due to the tighter overall loop - the previous value would have had time to clear the pipeline in the bitwise function due to the extra instructions and loop handling, but not in the ASM function. It's not too surprising that both methods are slower than the boolean version though; we have to do a bit of extra work with the values involved, and the reduced cache locality likely has a slight negative impact.

There's still a slight performance disparity between the two bitwise versions though and I'm honestly not 100% what it is. I investigated the [instruction tables](https://www.uops.info/table.html) for my CPU, the AMD64 Architecture Programmer's Manual, and various forums. I'm not an expert at reading these sorts of sources by any means, but while the BT* instructions are definitely on the slower end, the numbers don't seem unreasonably bad. The general consensus seems to be that they're just slower than you'd expect when operating on memory.

## But at what cost?

For the final version, we can throw readability in the garbage. Neither the bitwise nor the ASM version are really the most optimal way to operate on this sort of dataset. We're operating on contiguous ranges, and because those ranges are usually pretty large, it makes more sense to do bulk operations rather than modify individual bits at a time. Doing so means we can maintain the cache locality of the bool version, the space savings of the bitwise version, and reduce the number of individual calls to memory greatly. To accomplish this, we can use u64's as a sort of "poor man's SIMD vector". Due to bad divisibility, we'll have to manage the fact that our "rows" won't end on even boundaries - for example, our first row of 1000 bits will end on the 39th bit of u64 #15. Luckily, 1,000,000 *is* divisible by 64, so we don't have to allocate any extra pad bytes. As an aside, an easier-to-reason-about method might be to have 16 rows of 1000, the extra space is simply ignored and doesn't affect the final total.

I'll admit, this one took me several hours of debugging and a night's sleep to get working properly. I ended up needing to write several of my own test cases, wrestle with the language, and correct a few of my own assumptions. Did you know that overflow on left shift (e.g. `u8::MAX << 8`) is undefined behavior C and is disallowed in Rust? I didn't, and apparently [I'm not the only one](https://users.rust-lang.org/t/intentionally-overflow-on-shift/11859).

In any case, I want to walk through this as it's a bit more involved than the previous examples.

Because we're operating on a range, we'll need start and end indexes. There isn't clean divisibility, so we need to delay the division as long as possible to avoid compounding truncation errors. Another difference is that we need to take the modulo of the full bit-index, not just the x value like before. This is because the rows don't always start and end on the same bit of their respective u64s (e.g. row 1 starts at bit 0, row 2 starts at bit 40).

```rust
for y in start.y..=end.y {
    let start_idx = (y * 1000 + start.x) / 64;
    let start_offset = (y * 1000 + start.x) % 64;

    let end_idx = (y * 1000 + end.x) / 64;
    let end_offset = (y * 1000 + end.x) % 64;
    ...
```

Next we set up the `start` bit mask by clearing the unnecessary bits via 2 bit shifts, and handle the case where the start and end index are identical. The second step isn't necessary if you structure things a bit differently, but this makes a bit more sense to my brain. Note that when handling the "end" index, we need to alter the bit shifts so that we clear off bits on the opposite side.

```rust
    ...
    let mut start = u64::MAX;

    start <<= start_offset;
    start >>= start_offset;

    // if we're setting 64 bits or less, we modify only the start value
    if start_idx == end_idx {
        // reverse of the above operation
        start >>= 63 - end_offset;
        start <<= 63 - end_offset;

        bit_array[start_idx] |= start;
        continue;
    }

    bit_array[start_idx] |= start;
    ...
```


We then loop through the values between the start and end, replacing them with `u64::MAX`, 0, or `xor`ing them with `u64::MAX` as required.

```rust
    ...
    // loop over x values
    for i in start_idx + 1..end_idx {
        bit_array[i] = u64::MAX;
    }
    ...
```

After that, we handle the end index and we're all done!

```rust
    ...
    let mut end = u64::MAX;

    // clear off only the bits within our range
    end >>= 63 - end_offset;
    end <<= 63 - end_offset;
    bit_array[end_idx] |= end;
```

Running this gives us the correct answer, in only 544.658µs - a speedup of ~30x over the boolean version using 1/8th the memory, and a ~45x speedup over the bitwise version. All it cost was several hours, a bit of hair pulling, and some extra comments in the source code.

At this point, I suspect that we're nearing the limit of how much faster we can make it. We could experiment with real SIMD registers and operate on even larger bit slices, but that's for some other time. If you want to see the full source code, it's available [here](https://github.com/Walnut356/AdventofCode/blob/master/aoc2015/src/day6.rs).

I think this problem is a really neat demonstration of the tradeoffs between memory usage, CPU time, and the readability of your code. It also provides a nice reminder that sometimes readability can come from unlikely places - in this case, a simple assembly instruction has (arguably) less cognitive complexity than the equivalent high-level bitwise indexing. While we often see examples of what great code looks like, and how we can still make great code that's *performant*, I always find it a fun exercise to [throw all that to the wind and write some very fast, very gross code](https://www.youtube.com/watch?v=4LiP39gJuqE). One of my favorite experiences with programming has been learning how insanely fast computers really are, and just how far you can push the limited tool sets that programming languages provide.