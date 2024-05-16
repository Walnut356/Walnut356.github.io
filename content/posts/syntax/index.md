+++
title = "Syntax Test"
date = 2099-04-01
draft = true
[taxonomies]
tags=["programming", "rust",]
+++

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