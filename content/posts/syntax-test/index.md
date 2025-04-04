+++
title = "Syntax Test"
date = 2024-01-01
[taxonomies]
tags=["programming", "rust",]
+++

I write my own sublime-syntax files for the syntax highlighting on this site, this page is meant for testing edge cases.

<!-- more -->

## Rust

```rust
mod test {
    mod test2;
    mod test3;
}

use std::ops::Add;

// regular comment

/*
    Block Comment
    multiple lines
*/

/// Docs
#[derive(Debug, Clone)]
pub struct Thing<'a> {
        eef: usize,
        freef: &'a usize
}

impl<'a> Thing<'a> {
    pub const CON: usize = 7;

    pub fn new(eef: usize, freef: &'a usize) -> Self {
        Self {
            eef,
            freef,
        }
    }
}

pub const CON: usize = 7;
const CON: usize = 7;
pub static STAT: usize = 7;
static STAT: usize = 7;

impl<'a> std::ops::Mul for Thing<'a> {
    type Output = Self;

    fn mul(self, rhs: Self) -> Self::Output {
        todo!()
    }
}

pub trait Wow {
    fn wee(e: usize) -> Self;
}

pub enum Nums<'a> {
    One,
    Two(i32),
    Three { val: usize },
    Four(Thing<'a>),
    Five
}

real!(f32::from(real!(0.696) / real!(1.2)) / 1.4)

async fn async_fn_() {

}

fn main() {
    let e: i32 = 5;
    let thing = Thing { eef: 5, freef: &e};

    for x in 0..thing.eef {
        let z = x - 5;
    }

    let mut thing: Vec<i32> = vec![1, 2, 3];

    let num = Num::One;

    for (x, y) in (0..10).iter().zip(0..10.iter()) {
        match num {
            Num::One => (),
            Num::Two(x) => (),
            Num::Three { val } => (),
            _ => (),
        }
    }

    let eef = &thing;
    let freef = Thing::CON;

    let z = thing.iter().map(|x| x + 1).collect::<Vec<_>>();
    println!("Hello, world! {}", z[0]);
}

```

## LLVM

```llvm
; sc2_sim::coordinator::Coordinator::tick_effects::{{closure}}
; Function Attrs: inlinehint uwtable
define internal void @"sc2_sim::coordinator::Coordinator::tick_effects::{{closure}}"(ptr noalias noundef readonly align 8 dereferenceable(8) %_1, ptr noalias noundef align 8 dereferenceable(144) %army) unnamed_addr #0 {
start:
  %_25 = alloca [1 x i8], align 1
  %_20 = alloca [16 x i8], align 8
  %i = alloca [8 x i8], align 8
  %_8 = alloca [8 x i8], align 8
  %iter = alloca [16 x i8], align 8
; call sc2_sim::utils::unchecked_mut
  %units = call noundef align 8 dereferenceable(24) ptr @sc2_sim::utils::unchecked_mut(ptr noalias noundef align 8 dereferenceable(24) %army)
; call <&mut alloc::vec::Vec<T,A> as core::iter::traits::collect::IntoIterator>::into_iter
  %0 = call { ptr, ptr } @"<&mut alloc::vec::Vec<T,A> as core::iter::traits::collect::IntoIterator>::into_iter"(ptr noalias noundef align 8 dereferenceable(24) %units)
  %_5.0 = extractvalue { ptr, ptr } %0, 0
  %_5.1 = extractvalue { ptr, ptr } %0, 1
  call void @llvm.lifetime.start.p0(i64 16, ptr %iter)
  store ptr %_5.0, ptr %iter, align 8
  %1 = getelementptr inbounds i8, ptr %iter, i64 8
  store ptr %_5.1, ptr %1, align 8
  br label %bb3

bb3:                                              ; preds = %bb16, %start
  call void @llvm.lifetime.start.p0(i64 8, ptr %_8)
; call <core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next
  %2 = call noundef align 8 dereferenceable_or_null(88) ptr @"<core::slice::iter::IterMut<T> as core::iter::traits::iterator::Iterator>::next"(ptr noalias noundef align 8 dereferenceable(16) %iter)
  store ptr %2, ptr %_8, align 8
  %3 = load ptr, ptr %_8, align 8, !noundef !2
  %4 = ptrtoint ptr %3 to i64
  %5 = icmp eq i64 %4, 0
  %_10 = select i1 %5, i64 0, i64 1
  switch i64 %_10, label %bb5 [
    i64 0, label %bb7
    i64 1, label %bb6
  ]

bb5:                                              ; preds = %bb12, %bb9, %bb3
  unreachable

bb7:                                              ; preds = %bb3
  call void @llvm.lifetime.end.p0(i64 8, ptr %_8)
  call void @llvm.lifetime.end.p0(i64 16, ptr %iter)
  ret void

bb6:                                              ; preds = %bb3
  %unit = load ptr, ptr %_8, align 8, !nonnull !2, !align !4, !noundef !2
  call void @llvm.lifetime.start.p0(i64 8, ptr %i)
  store i64 0, ptr %i, align 8
  br label %bb8

bb8:                                              ; preds = %bb15, %bb6
  %_14 = load i64, ptr %i, align 8, !noundef !2
  %6 = getelementptr inbounds i8, ptr %unit, i64 16
  %_15 = load i64, ptr %6, align 8, !noundef !2
  %_13 = icmp ult i64 %_14, %_15
  br i1 %_13, label %bb9, label %bb16

bb16:                                             ; preds = %bb8
  call void @llvm.lifetime.end.p0(i64 8, ptr %i)
  call void @llvm.lifetime.end.p0(i64 8, ptr %_8)
  br label %bb3

bb9:                                              ; preds = %bb8
  %_18 = load i64, ptr %i, align 8, !noundef !2
; call <alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index
  %_16 = call noundef align 8 dereferenceable(16) ptr @"<alloc::vec::Vec<T,A> as core::ops::index::Index<I>>::index"(ptr noalias noundef readonly align 8 dereferenceable(24) %unit, i64 noundef %_18, ptr noalias noundef readonly align 8 dereferenceable(24) @alloc_4fdbcb53d8adebf2d48242f5b6c09e60)
  %7 = load i32, ptr %_16, align 8, !range !6, !noundef !2
  %_19 = zext i32 %7 to i64
  switch i64 %_19, label %bb5 [
    i64 1, label %bb12
    i64 0, label %bb11
  ]

bb12:                                             ; preds = %bb9
  %8 = getelementptr inbounds i8, ptr %_16, i64 4
  %timestamp = load i32, ptr %8, align 4, !noundef !2
  %_23 = load ptr, ptr %_1, align 8, !nonnull !2, !align !5, !noundef !2
  %_26 = load i32, ptr %_23, align 4, !noundef !2
  %9 = icmp slt i32 %timestamp, %_26
  %10 = icmp ne i32 %timestamp, %_26
  %11 = select i1 %10, i8 1, i8 0
  %12 = select i1 %9, i8 -1, i8 %11
  store i8 %12, ptr %_25, align 1
  %_24 = load i8, ptr %_25, align 1, !range !11, !noundef !2
  switch i8 %_24, label %bb5 [
    i8 -1, label %bb18
    i8 0, label %bb18
    i8 1, label %bb17
  ]

bb11:                                             ; preds = %bb9
  %13 = load i64, ptr %i, align 8, !noundef !2
  %14 = add i64 %13, 1
  store i64 %14, ptr %i, align 8
  br label %bb15

bb18:                                             ; preds = %bb12, %bb12
  call void @llvm.lifetime.start.p0(i64 16, ptr %_20)
  %_22 = load i64, ptr %i, align 8, !noundef !2
; call alloc::vec::Vec<T,A>::swap_remove
  call void @"alloc::vec::Vec<T,A>::swap_remove"(ptr noalias nocapture noundef sret([16 x i8]) align 8 dereferenceable(16) %_20, ptr noalias noundef align 8 dereferenceable(24) %unit, i64 noundef %_22)
  call void @llvm.lifetime.end.p0(i64 16, ptr %_20)
; call sc2_sim::army::Army::reset_speed
  call void @sc2_sim::army::Army::reset_speed(ptr noalias noundef align 8 dereferenceable(144) %army, ptr noalias noundef align 8 dereferenceable(88) %unit)
  br label %bb14

bb17:                                             ; preds = %bb12
  %15 = load i64, ptr %i, align 8, !noundef !2
  %16 = add i64 %15, 1
  store i64 %16, ptr %i, align 8
  br label %bb14

bb14:                                             ; preds = %bb17, %bb18
  br label %bb15

bb15:                                             ; preds = %bb11, %bb14
  br label %bb8
}
```