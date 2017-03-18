---
layout: post
title: How to Print Rust Structs/Enums
date: 2015-10-08 23:32:42
categories: [cn, dev]
tags: [rust]
---

在Rust中定义结构体(`struct`)或枚举(`enum`)类型时，可以在定义前加上属性`#[derive(Debug)]`。这样可以让编译器为该类型自动派生(derive)出`std::fmt::Debug` `trait`的实现。然后在打印时使用`{:?}`则可以打印出变量类型的数据表示。

注意的是，如果你希望用`{}`打印结构体或者枚举变量，你还是需要为其实现`std::fmt::Display` `trait`。

另外，如果你的结构体中还包含了别的结构体，你可能还是需要自己实现`std::fmt::Debug` `trait`来达到自己希望的打印效果。例如在下例([Rust Playground传送门](http://is.gd/C4QouO))中，`Deep(Structure(7))`会打印出`Deep(Structure(7))`，而`Flat(Structure(7))`则会通过自定义的实现打出`Structure(7)`。


```rust
use std::fmt;

#[derive(Debug)]
struct Structure(i32);

#[derive(Debug)]
struct Deep(Structure);

struct Flat(Structure);
impl fmt::Debug for Flat {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            Flat(Structure(num)) => write!(f, "Structure({})", num),
        }
    }
}

fn main() {
  println!("Structure {:?}: ", Structure(3));
  println!("Deep {:?}: ", Deep(Structure(7)));
  println!("Flat {:?}: ", Flat(Structure(7)));
}
```
