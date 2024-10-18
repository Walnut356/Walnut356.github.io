+++
title = "TwoVec: A Very Silly Container"
date = "2024-10-18"
description = ""
[taxonomies]
tags=["programming", "rust",]
+++

Lets say you want to store two different types of objects in 1 container. Simple right? Just slap those puppies in a tuple and you're good to go:

```rust
let list: Vec<(u8, f32)> = vec![(255, 20.0), (10, 37.0)];
```

But what if you wanted to include elements of two different types in *arbitrary orders*? Thankfully, there's sum types for that:

```rust
pub enum Val {
    A(u8),
    B(f32),
}

... {
    let list = vec![Val::A(255), Val::B(20.0), Val::B(37.0), Val::A(10)];
}
```

One small problem. That's *wasteful*.

<!-- more -->

We have 2 variants that cannot be niche optimized, and the variants are different sizes. Since `Val` itself must have 1 known size, smaller variants are required to add pad bytes to match the size of the largest variant. Additionally, since default Rust struct layout guarantees that the alignment of a struct is at least the maximum of all of its fields, the `f32` contained in `Val::B` forces all `Val`s to be 4-byte aligned. That means even with `#[repr(u8)]`, a `Val::A(u8)` is *64 bits in size*, despite it being possible to represent all `Val`s with only 5 bytes, and all `Val::A(u8)`'s with only 1 byte + 1 bit. `Val::B(f32)` is less wasteful, as only ~48% of its bits are complete dead weight, but it's still a sad number.

Not to worry, I know RAM is the most precious of commodities in 2024 so I've come up with the answer to your woes. The `TwoVec`:

```rust
let x = 8u8;
let y = 20.0f32;
// type inference from .push_a and .push_b
let mut list = TwoVec::new();

// dedicated functions
list.push_a(x);
list.push_b(y);
// or through type inference
list.push(18.0);
list.push(17);
list.push(12);
list.push(35.7);
list.push(1);

dbg!(&list);
// in bytes
dbg!(list.capacity());

// returns Some(val) if the value at that index is the correct type
let mut v: Option<u8> = list.get(0);
dbg!(v);
// returns None when the value is not the correct type
let mut w: Option<f32> = list.get(0);
dbg!(w);
w = list.get(1);
dbg!(w);
```

```txt
[Output]
&list = TwoVec[8, 20.0, 18.0, 17, 12, 35.7, 1]
list.capacity() = 18
v = Some(8,)
w = None
w = Some(20.0,)
```

In the above example, I made sure to exactly fill the current allocation size of `list`. That means the listed capacity (18 bytes) minus the size of the data is the amount of overhead for bookkeeping. 4 `u8`s and 3 `f32`s takes up 16 bytes total, so we only have a storage overhead of 2 bytes! Compared to the `Val` enum above, that's a savings of 30 bytes.

The source code is available [here](https://github.com/Walnut356/twovec), and the [crate is available on crates.io](https://crates.io/crates/twovec)

## How it works

The basics are pretty simple, but getting it to work in Rust's type system was a bit of an adventure.

To get the basis of the container out of the way as quickly as possible: array indexing is nothing but a base pointer + an offset. Arrays themselves are nothing more than an arbitrary region of bytes, regardless of what type they're "assigned" (see: C's `void*`). Types are only "real" at compile time. Having arrays that contain only 1 type - or ensuring that all types contained in the array are the same size like enums do - allows for consistent and universal indexing math. That being said, if you can figure out the offsets on your own, there's nothing stopping you from implementing the indexing logic yourself. In a sense, hashmaps already work based on this principle.

We only have two states, easily represented by a single bit. We know the size of both of the types, so we should be able to calculate the byte-offset ourselves by counting the number of each type and multiplying it by its respective size. We can start with a struct very similar to a standard `Vec`:

```rust
pub struct TwoVec<A, B> {
    len: usize,
    capacity: usize,
    bitfield: NonNull<u8>,
    data: NonNull<u8>,
    a: PhantomData<A>,
    b: PhantomData<B>,
}
```

### Crimes against debugability

You might notice that there's only 1 `capacity` and 1 `len`, despite there being 2 pointers do data. Since RAM is at a premium, we clearly can't store our bookkeeping bits in a *different* region of memory. Just think of the wasted allocator metadata bytes. As the old saying goes: "Only 1 allocation for this rustacean". This increases the bookkeeping complexity by a bit, but at least we can now say we're storing *three* different types in a single allocation instead of just 2.

This has a substantially negative impact on the debugging experience, since now it's possible to overwrite bookkeeping data with element data and vice versa if we fuck up our indexing math. Which I definitely didn't do like 5 separate times. And when yo- *ahem* **if** you fuck up, have fun debugging bitfields and packed, probably unaligned values. Just awful. I definitely do not speak from experience though.

Just for sake of example, imagine a few of the bits being set incorrectly due to an off-by-one error. Suddenly, your pretty-printer for the data block isn't just interpreting the bytes as the wrong value. The offset of the *next* value is partially based on the size of the one we have. If we interpret that size wrong, we're now jumping to the wrong location **and** even if we weren't, there's a good chance we're interpreting the bytes there as the wrong type too. The debugger display (at least, the one I was using) is largely unhelpful as well since an `unsigned char*` is interpreted as a C string. Hope you didn't have any `\0` bytes in your data anywhere or you only get to see fraction of the total block =)

Reading listings that looks like this

```text
{pointer:"\xba]B\xe3;?e7&z\\\U00000011\xbe\xcb\xd2>\xf9;5?3T"}
```

and trying to extract useful information about why things went wrong is a special kind of pain. Again, not that I speak from experience or anything.

Anyway, upon the first allocation of a data block, typical `Vec` implementations will reserve space for a handful of elements to reduce the number of times the `Vec` needs to be `realloc`'d. In our case, it's not so simple. If we reserve 8 spots for `f32`s + 1 additional byte for the bitfield, we run into a problem. If we filled the `TwoVec` with `u8`s, we wouldn't have enough bits to tag them all - 8 `f32`s takes the same space as *32* `u8`s. We always need to make sure we have enough bits to cover the worst-possible case:

```rust
// "base" size of the allocation, taken from rust's `RawVec` implementation
let base_size = Self::MAX_A_B * Self::MIN_NON_ZER0_CAPACITY;
// number of bits needed to represent the "worst case" (i.e. 100% of capacity filled with the smaller element)
let max_elements = base_size / Self::MIN_A_B;
let bitfield_size = max_elements / 8;

let new_capacity = base_size + bitfield_size;
let new_layout = Layout::array::<u8>(new_capacity).unwrap();
```

On each reallocation, we double the number of elements that can be stored, which exactly doubles the number of degenerate-case slots. Thus we don't have to worry about further fiddling with the bitfield size after the initial allocation, doubling the whole capacity suffices.

Unfortunately, that brings up another issue. We cannot simply call `realloc`. `realloc` automatically copies all of the old bytes into the new allocation. In `TwoVec`, the distance between the start of the bitfield and the start of the data block is not static, it grows when the allocation grows. That means we need to manually copy the data over in two steps, leaving an extra gap for the new bitfield bytes

```rust
unsafe {
    let offset = self.data.offset_from(self.bitfield) as usize;
    self.bitfield.copy_to_nonoverlapping(new_ptr, offset);

    let new_data = new_ptr.add(bitfield_size);
    self.data.copy_to_nonoverlapping(new_data, self.capacity - offset);

    dealloc(
        self.bitfield.as_ptr(),
        Layout::array::<u8>(self.capacity).unwrap(),
    );
}
```

### Crimes against the type system

The `get` and `push` functions require 2 different implementations because we need to be able to detect if the type we're asking for matches the value of the type-bit. i.e. if we want a `B`, we have to check if the bit is `1`, but if we want an `A`, we have to check if the bit is a `0`. It *might* be possible to pull this off dynamically in 1 function, but we still have another issue.

Those with a keen eye may have heard some alarm bells when I showed off the type inference of the `get` and `push` methods. After all, if they were defined as

```rust
impl<A, B> TwoVec<A, B> {
    pub fn get<T>(&self, idx: usize) -> Option<T> {...}
}
```

We suddenly have a *very* big safety hole. Rust's type system can express exactly 2 relationships between *types*. `TwoVec<A, A>` is a `TwoVec` whose elements are *always* of type `A`. Conversely, `TwoVec<A, B>` means that elements may be of type `A` or type `B`, and that `A` *might* be the same as type `B`, but also might not. There is no way to indicate that two types must *not* be the same. There is also no way to guarantee that a third type (`T` in this case) *is* `A` or `B`.

Imagine we have a completely full `TwoVec<A, B>` and we access the last element with our `get` function. Except we forget what type `A` is, and call `get` with a type where `size_of::<T>() > size_of::<A>()`. Since this whole house of cards functions via pointer casting, there is nothing stopping this from reading out-of-bounds memory. Even if we added a check (e.g. `if ptr_to_elmt + size_of::<T>() > ptr_to_end_of_alloc`) we're still giving people an "unsafe-free" `mem::transmute()`.

We could try declaring the function `get<A>` *or* `get<B>`, but now it only works for one of our two types which isn't ideal. Instead, we can try using a `Get<T>` trait to accomplish this.

There's a problem when we add the `impl`s though:

```rust
impl<A, B> Get<A> for TwoVec<A, B> {
    fn get(&self) -> Option<A> {
        ...
    }
}

impl<A, B> Get<B> for TwoVec<A, B> { // Error, overlapping impls
    fn get(&self) -> Option<B> {
        ...
    }
}
```

Because `A` and `B` *could* be the same type, the compiler wouldn't know which `impl` to choose. We, as the programmer, know that it doesn't even make sense for a `TwoVec` to be used when `A` and `B` are the same type - `TwoVec<A, A>` is strictly worse than just using `Vec<A>` - but there's no way to express this via a trait on `TwoVec`.

A bare marker trait on the types themselves also doesn't quite work.

```rust
pub trait MarkerA {
    fn get(tv: , idx: usize) -> Option<Self>;
}
pub trait MarkerB {
    fn get(&self, idx: usize) -> Option<Self>;
}

impl MarkerA for u8 {...}
impl MarkerB for f32 {...}
```

Aside from the fact that the user must now define these marker traits, we also have the issue that there's no negative `trait` bounds (outside of the compiler) either! A type could easily implement `MarkerA` *and* `MarkerB` and it still wouldn't know which version to call. Even if we somehow bypassed that restriction and make the traits mutually exclusive, we cannot have `TwoVec<u8, f32>` and `TwoVec<f32, u8>` in the same program because `u8` would be required to impl `MarkerA` for the former and `MarkerB` for the latter.

It took me quite a bit of fiddling to work around this issue. I tried lots of `AsRef` and deref-abuse shenanigans, some unstable features, lots of different things on both the `TwoVec` itself and on its constituent types. Many things *kindof* work, but require the [fully qualified syntax](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#fully-qualified-syntax-for-disambiguation-calling-methods-with-the-same-name) which is... not preferable for obvious reasons. I have half an RFC written out of pure frustration titled `negative_type_bounds`. Being able to specify `pub struct TwoVec<A, B> where A: !B` (or type subsets like `fn get<T: A | B>`) would make this significantly easier. As far as I know, that sort of thing is trivial with C++'s templates.

I found one way to make things work, and it's const generics (though there are lots of ways that *don't* work with const generics and unstable `generic_const_expr`, maybe a story for another day). By including a const `bool` type parameter on a trait implemented on each *type* `A` and `B`, we can differentiate between `<A, B>` where `A` is important and `<A, B>` where `B` is important. We can blanket-impl this trait to every `Sized` type:

```rust
pub trait Gettable<A, B, const Z: bool>
where
    Self: Sized,
{
    fn get(tv: &TwoVec<A, B>, idx: usize) -> Option<Self>;
}

impl<A, B> Gettable<A, B, false> for A {
    fn get(tv: &TwoVec<A, B>, idx: usize) -> Option<Self> {
        tv.get_a(idx)
    }
}

impl<A, B> Gettable<A, B, true> for B {
    fn get(tv: &TwoVec<A, B>, idx: usize) -> Option<Self> {
        tv.get_b(idx)
    }
}
```

From there, it's possible to add a function `get<T, const Z: bool>` in `TwoVec` with the bound `where T: Gettable<A, B, Z>`. Not only does this restrict to only the exact types `A` and `B` of `TwoVec<A, B>`, it also does not conflict with `TwoVec<B, A>`, since that is an entirely different type from `TwoVec<A, B>`. Technically, when turbofishing `get` you're now required to also include the const bool parameter, but in most cases this is unnecessary because type inference removes the need for a turbofish at all.

To clarify further, since this is sorta spaghetti, here's an example. Say you have `TwoVec<u8, f32>`

* When asking for an `Option<u8>`, the compiler sees that `u8` is `Gettable` for `u8, f32, true`, and the function signature requires a `TwoVec<u8, f32>`. There is no `Gettable` with `u8, f32, false`, as `false` is only cares about the `B` parameter, and `u8 != f32`. There is no other `Gettable` implementation for `u8` with valid criteria, thus the const parameter `true` can be assumed
* `TwoVec<u8, f32>` impls cannot overlap with, say, `TwoVec<u8, i64>` because the `B` parameter of `i64` (and the `TwoVec<u8, f32>` in the function signature) would place it in an entirely different monomorphization of `Gettable`
* There is no self-overlap with `TwoVec<u8, u8>` because `Gettable<A, B, false>` and `Gettable<A, B, true>` are mutually exclusive (even though in this case they'd be completely interchangeable). Interestingly, type inference doesn't require us to turbofish on a `TwoVec<A, A>` even though both implementations would be valid. I'm not sure whether the compiler is choosing `get::<u8, true>()` or `get::<u8, false>()` or how it even makes the decision. Weird.


Grimy? Yes. Cool? Arguably.

We can even add a third impl that returns an `either::Either`, though this one feels even more gross since now the const parameter is truly meaningless (though it still must be a definite value for type inference to work)

```rust
impl<A: Copy, B: Copy> Gettable<A, B, true> for Either<A, B> {
    fn get(tv: &TwoVec<A, B>, idx: usize) -> Option<Self> {
        if idx > tv.len() {
            return None;
        }

        if tv.is_a(idx) {
            tv.get_a(idx).map(|x| Either::Left(x))
        } else {
            tv.get_b(idx).map(|x| Either::Right(x))
        }
    }
}
```

### Bit Fiddling

As I mentioned above, debugging this nonsense is the *worst*. Typical debugger utilities are borderline worthless, so I have to check a lot of things by hand. As in notebook, pen, ascii table, and programming calculator, hand-translating whatever the debugger gave me into a slightly more useable form data at each step in the algorithm, drawing diagrams and painstakingly proving to myself each individual piece makes sense.

I want to walk through the implementation of the `remove` function to give you guys an idea how much more fiddly this is compared to `Vec` functions. `remove` has to do two distinct tasks to work properly:

1. copy the data from the region `offset_of_next_element..end_offset` to the region `offset_of_element..end_offset_minus_size_of_removed_element`
2. take every bitfield-bit *after* the index and bitshift it left once

The first part is easy - we can copy the `std::Vec` implementation almost exactly. The only difference is that we can't just subtract "1 element" from the end offset, we have 2 different kinds of elements.

```rust
pub fn remove_a(&mut self, idx: usize) -> Option<A> {
    if idx > self.len {
        panic!("Cannot remove value at index {idx}, length is {}", self.len);
    }
    if !self.is_a(idx) {
        return None;
    }

    let val_offset = self.idx_to_offset(idx);
    let end_offset = self.idx_to_offset(self.len);
    let out: Option<A>;

    unsafe {
        let ptr = self.data.add(val_offset);
        out = Some(ptr.cast::<A>().read_unaligned());

        ptr.copy_from(
            ptr.add(Self::A_SIZE),
            end_offset - (val_offset + Self::A_SIZE),
        );
    }

    self.shift_bits_down_after(idx);

    out
}
```

That `shift_bits_down_after` function call is carrying a lot of weight though. I always jump right to the bit fiddling solution, but I ended up messing it up on my first go-round. I took the time to make this slower, simpler function first to test against:

```rust
for i in idx..self.len - 1 {
    let b = self.get_bit(i + 1);
    if b == 0 {
        self.clear_bit(i);
    } else {
        self.set_bit(i);
    }
}
```

Before we get into the indexing math, I want to point out something small. In an effort to make debugging a little bit less painful the bit indexes are "reversed" so that the whole bitfield slice pretty-prints in order. That means the bit set to `1` in the expressions `1 << 7` or `0b1000_0000` is at index `0`. The `clear_`, `set_` and `get_` functions all follow this same pattern:

```rust
fn set_bit(&mut self, idx: usize) {
    let byte_idx = idx / 8;
    let bit_idx = idx % 8;
    let bf = self.bitfield_mut();
    bf[byte_idx] |= 1 << (7 - bit_idx);
}
```

To lump multiple of these together into bit-shift operations, we need to handle 2 separate steps

1. for the byte that `idx` falls in, we need to left-shift *only* the bits after `idx`
2. for that byte and all (potential) following bytes, we need to fill the new "blank" right-most bit. The value that goes here is the left-most value of the next byte in the bitfield

As an aside, since the carry bit comes from the previous shift instruction, the most efficient way to handle this would probably be to iterate through the bitfield *backwards*. This is because the shift instructions automatically store the last-shifted-out-bit in a special, non-general-purpose register. Once there, it can be passed to the next operation for free - no register or stack usage required. On the other hand, computers aren't optimized for reverse-order array access, which typically wrecks the cache prefetch predictions. On the other *other* hand, that probably only matters if the bitfield is larger than a cache line (typically 64 bytes, which is 512 elements). On the other *other* **other** hand, it might be that the compiler can "see through" what I'm doing and rearrange the operations such that it uses the carry flag regardless of the order I read the array. For the sake of my sanity, I'm just going to iterate through it front to back.

```rust
let bf = self.bitfield_mut();

let start_byte_idx = idx / 8;
let start_bit_idx = idx % 8;

let mut idx_byte = bf[start_byte_idx];
```

Next we need to isolate the 2 sets of bits we want - the bits before `idx` and the bits after `idx`. This is a bit trickier than it sounds though, since we also need to shift off `idx` itself. When shifting off the bottom bits, the naive shift amount might be `7 - (idx + 1)`, but this will panic if `idx == 7` due to subtraction overflow. While we could do something like `7.saturating_sub(start_bit_idx + 1)`, this doesn't quite have the behavior we want. If `idx == 7` and our byte is `0b0000_0001`, the prior formula will give us a bitshift of 0 and our value won't change.

Workarounds for that like `7.checked_sub(start_bit_idx + 1).unwrap_or(1)` or `7.saturating_sub(start_bit_idx + 1).max(1)` only work when `idx > 0`. When `idx == 0`, the only right-shift that would clear the top bit is 8. Unfortunately, shifting with a value that is >= the bitwidth of your integer is UB.

While we could make that a special case (`if start_bit_idx != 0 {...}` or `checked_shr`), it's easier to just shift it by the base amount, shift that value by 1, and then do the same in reverse. Alternatively, since we're dealing in non-`usize` values, we could cast the value to a larger size during the bit fiddling and then `as` cast it back to a `u8` (which truncates it in rust) when it comes time to store the value. Casting more or less compiles to a noop so it's probably a smidge faster than 2 bitshifts.

```rust
// clears all the bits including and after idx
let shift_amt = 8 - start_bit_idx;
let pre_idx_bits = ((((idx_byte as u32 >> shift_amt) >> 1) << shift_amt) << 1) as u8;

// clears all the bits up to and including idx. Requires a +1 because bits are
// "0-indexed" when shifting. When shifting back, we drop the +1 so that we can
// overwrite the bit that's being removed
let post_idx_bits = (((idx_byte as u32) << (start_bit_idx + 1)) as u8) >> start_bit_idx;
```

One very important note: the cast to u8 for the pre index bits **must** be done *after* both shifts, since both shifts could overflow. The cast to u8 for the post index bits **must** be done *before* the second shift, because we need to truncate off the top bits before sliding everything back into place. I mistakenly did the truncation after both shifts and it took me an embarrassingly long time to spot.

With this done, we've effectively "closed the gap" by removing a bit, and the right-most bit in the pre- and post-idx values should be 0. Next we just need the carry bit, which can default to 0 if there is no next byte.

```rust
let mut carry = bf.get(start_byte_idx + 1).cloned().unwrap_or_default() >> 7;
```

and the resultant value is all 3 of those values masked together

```rust
bf[start_byte_idx] = pre_idx_bits | post_idx_bits | carry;
```

And finally, we can handle the rest of the bytes (if there are any) with a straightforward loop:

```rust
for i in 1..bf.len() {
    let mut x = bf[i];
    carry = bf.get(i + 1).cloned().unwrap_or_default() >> 7;
    bf[i] = (x << 1) | carry;
}
```

All of that just to accomplish the same feat as a single line in the `Vec::remove` implementation.

## Limitations

### Speed

Let's get this out of the way right away, on benchmarks pushing/reading/removing 10,000 random elements with random types, `TwoVec` is ~100x slower than `Vec<Enum>`.

`.idx_to_offset()` - the primary driver for accessing the data - is technically `O(N)`, but is really `O(N/64)` (yes I know how Big O works, but as far as I'm aware most people aren't adding an infinite number of elements to their vector so the coefficient still matters). This is after some optimizations. The original implementation looked something like this:

```rust
pub fn idx_to_offset(&self, idx: usize) -> usize {
        let byte_count = idx / 8;
        let bit_count = idx % 8;

        let bitfield = unsafe {
            slice_from_raw_parts(self.bitfield.as_ptr(), byte_count)
                .as_ref()
                .unwrap()
        };
        let mut slice = [0u8; 8];
        for i in bitfield.chunks(8) {

        }

        let result: usize = bitfield
            .iter()
            .map(|x| {
                let a_count = x.count_zeros() as usize;
                (a_count * Self::A_SIZE) + ((8 - a_count) * Self::B_SIZE)
            })
            .sum();

        if bit_count == 0 {
            return result;
        }

        // handling for the remaining partial u8
        ...
}
```

Essentially, it takes the slice `bitfield[..bitfield.len() - 1]` and iterates over it, using `.count_ones()`/`.count_zeroes()` (which compile down to ~a single `popcnt` instruction) to determine how many of each type of element there is. A final special case has to be handled when `idx` doesn't fall on a byte boundary.

This code gets vectorized by the compiler. As it turns out though, the vectorized code is slightly suboptimal. In my benchmark, it ran a fair bit faster with `#[inline(never)]`, but even that was slow. I'm not sure if it's the inherent performance characteristics of the vector instructions, the degenerate cases where the whole vector register is loaded up just to check 1 bit, the algorithm used to emulate `popcnt` behavior, or some other factor that I'm not considering. In any case, using "shitty simd", (i.e. treating a `u64` as packed `u8`s) was about 20% faster on `push` and `get` for reasonable numbers of elements.

```rust
    ...

    for (i, chunk) in bitfield.chunks(8).enumerate() {
        let mut val = match chunk.len() {
            8 => {
                u64::from_be_bytes(chunk.try_into().unwrap())
            }
            x => {
                let mut slice = [0u8; 8];
                slice[..x].copy_from_slice(&chunk[..x]);

                u64::from_be_bytes(slice)
            }
        };

        let important_bits = idx % 64;
        if (i + 1) * 8 < len || important_bits == 0{
            let a_count = val.count_zeros() as usize;
            result += (a_count * Self::A_SIZE) + ((64 - a_count) * Self::B_SIZE);
            continue;
        }

        let shift_bits = 64 - important_bits;

        let b_count = ((val >> shift_bits) << shift_bits).count_ones() as usize;

        let a_bytes = important_bits.saturating_sub(b_count) * Self::A_SIZE;
        let b_bytes = b_count * Self::B_SIZE;

        result += a_bytes + b_bytes;
    }

    result
}
```

I also tried `u32`-shitty-simd just in case, but that ended up being slower.

This could be improved by using run-length encoding, but that introduces some degenerate-case performance characteristics. If we store a reasonably-sized integer for the length, it'd use a ton more space when the type changes every single element. The worst possible case isn't a *huge* issue though, since at that point the end user should probably reach for `Vec<(A, B)>` instead.

There are potential bit-packing solutions that could offer a more reasonable tradeoff. For example, bit 1 indicates the type, the next 3 bits indicate the run length, and 2 of those are packed into each `u8`. That's a potential project for another day though.

### Traits

Those of you who muck around with pointers probably saw this coming. Notice that the `get` functions return *owned objects*, not references. This is the single biggest limitation of the container in my opinion.

Since the values are packed in memory, there's no guarantee that they are aligned properly. It is **always** UB to have a `&T` if `T` is unaligned. I'm hesitant to even have a function that hands out a raw pointers to `T` because it's so easy to do unaligned access wrong. I could use something like the `unaligned` crate and conditionally give out references when things happen to be aligned, but that comes with its own problems. Ideally I'd want the functions to be implemented on a trait, bounded by `align_of::<A>() == align_of::<B>()`, so the returned object is guaranteed to be aligned, but that requires `const_generic_expr` to be stabilized.

As a result, `A` and `B` are both required to be `Copy`. I left that detail out of the post until now mostly to make the signatures easier to read.

In the future, I may make an alternative "alignment-aware" version that ends up being a hybrid of `TwoVec` and `Vec::Enum`. It would only pad when necessary, so multiple small elements in a row could still be packed. Or, if ordering wasn't considered important, I could "backfill" the padding with new values.

Additionally, unlike `Vec` we cannot implement `Deref` or `AsRef` to anything else in any kind of sensible way because of how delicate and specialized the internal bookkeeping is. That severely limits the number of helper functions we can get "for free".

### Space

While the storage overhead is typically much better than an enum, it's not guaranteed to be. The overhead varies based on how many elements are in the `TwoVec` and how large the size-difference is between types `A` and `B`. It has opposite storage characteristics to those of an enum - the overhead of an enum is worst when using mostly the small elements, whereas `TwoVec` stores the small variants much more efficiently, but can end up with a decent amount of wasted space when storing mostly large variants.

## But why?

To be clear, I don't think this container is very practical. If you find a usecase for it, please do not tell me. Whatever problem you're solving is *unbelievably* cursed and I don't want it anywhere near me.

This is mostly inspired by some of the jank workarounds I used when completing Advent of Code in C since I didn't want to implement hashmaps from scratch, and I didn't feel super comfortable with unions. `TwoVec` is a living example of something that I learned from C:

Standard library types are a *generalist* solution that must be correct and useful in almost all cases. "Almost all" is not "every". There exists alternative ways of expressing your intent to the computer. In the same way that performance tuning is about informing the compiler about as many invariants as possible, sometimes it's okay to have a slightly gross data structure if it means it reflects your intentions better. In some cases, this also reduces the complexity of the final algorithm. Rules like "one type per array" or even "all array elements must be the same size" are just the tip of the iceberg. Calling them "rules" is itself sortof wrong. It's easy to forget that when reading discourse and advice about programming.

If I had to guess, I think that these sorts of "one-off, highly specific, slightly gross" data structures used to be more common back in the "wild west" days of programming. Back when hardware was much more limited, when languages and compilers were far less advanced, and when there was less "common knowledge" and "best practices" than we have today.

There are good reasons that they've fallen out of fashion - upkeep, readability, different performance priorities, etc. - but they shouldn't be considered non-options compared to generalized "standard-library"-esque data structures. Even if they're not necessary or practical, it still requires some lateral thinking and a deeper understanding of the language and underlying hardware. Those are always valuable skills to work on.
