---
title: Disambiguating Clone And Copy Traits In Rust
date: 2022-06-28 16:08:12
tags:
- rust
---


If you've been dipping your toes in the awesome Rust language, you must've encountered the clone() method which is present in almost every object out there to make a  deep copy of it. It comes from the implementation of Clone trait for a struct.

But, you must also have encountered this Copy trait if you ventured a bit into docs deeper or in an error message. Copy also allows some type to replicate its instances. So, what's the deal here with Clone and Copy traits? That is exactly what this article is about.

## Ownership in Rust

I won't delve too deep into ownership here, but this is what we need to know to get the gist of this article:

- Unlike other languages, each value in Rust has an **owner** which is the variable assigned to it.
- Any value can only have **one owner at a time**. Although, multiple variables can hold reference to or borrow the same value - but that value will only have one owner.
- When the owner (variable) goes out of scope, the value is destroyed - freeing the memory.

This ownership model in Rust may seem constraining but it is one of the things that makes Rust a great language. It makes you, the programmer, harder to write code with hideous bugs.

## Memory Allocation in Rust

Normally, you don't have to worry about [stack or heap memory allocation](https://www.geeksforgeeks.org/stack-vs-heap-memory-allocation/) - it is done automatically by the language. But in Rust, a variable may behave differently depending on whether it is on the stack or heap. Rust strives to be performant because of it. 

The general rule is that simple values like numeric types - `u16`, `i32`, `f64` etc. - are stored on the stack as they have known fixed size and it is cheap to copy them. Pointers like `&str` (NOT the value it is pointing to!) are also stored on the stack because of their fixed size!

Dynamically sized types (DSTs) like `String` and `Vec` are stored on the heap. DSTs can be mutated to grow or shrink in size. They don't have a fixed size and it can be very expensive to copy them making them unsuitable for stack location.

Alright! That is enough primer, let's cut to the chase.

### `Clone` trait
The `Clone` trait defines the ability to **explicitly** deep copy an object. Take an example of a `String` struct instance that implements `Clone` trait. 

```rust
// initial owner of string value
let s1 = String::from("hello!");

// ownership of same value transferred to s2
let s2 = s1; 

// error! s1 was moved!
println!("{}", s1); 
```
The above example works according to ownership rules of Rust. The assignment of `s1` to `s2` transfers ownership of the `String` instance to `s2`, since there can be only one owner of a value at a time in Rust. If you've worked with other statically typed languages you might call this a shallow copy of `s1` as `s2` now points to the same data as `s1` was pointing to, without any re-allocation in memory. Only pointer in the stack was copied, not value in the heap.

In Rust lingo however, it is termed as a **move**, in the sense that value of `s1` is moved into `s2` and `s1` is no longer valid. Hence any access to `s1` will result in an error - which is different from other languages.

Now to make deep copy of the `String` you will have to explicitly invoke the `clone()` method. When you `clone()` a value Rust would replicate both pointer in the stack as well as data at heap.

```rust
// owner is s1
let s1 = String::from("hello!"); 

// deep copy of s1 is allocated to s2
let s2 = s1.clone(); 

// this is fine!
println!("{}", s1);
```
This is what `clone()` method does - making a **deep copy** of an instance.

### Copy trait
The `Copy` trait defines the ability to **implicitly** copy an object. This is available for the types that have a fixed size and are stored entirely on the stack. `u32` is a good  example that implements `Copy`.

```rust
// owner is n1
let n1 = 256;

// n1 is implicitly copied on stack and 
// copied value is assigned to n2 
let n2 = n1; 

// this is fine!
println!("{}", n1); 
```

The above example may seem contradictory to ownership rules at first. But, since `u32` are `Copy` type, `n1` value is implicitly copied into `n2` and NOT moved. This is because `u32` values are stored on stack and it is cheap to copy them. There is no corresponding `copy()` method provided by `Copy` trait (unlike `Clone`'s `clone()`). The copy here happens implicitly. Also, there is no difference between a shallow copy or deep copy here. 

The `Copy` trait can only be annotated to types that are stored on stack. Like integer types or structs that have all their fields of `Copy` type. You cannot annotate a type as `Copy` if any of its constituent parts cannot be stored on stack.

```rust
// This is fine!
#[derive(Clone, Copy)]
struct Demo {
    _a: u32, // Copy type ✅
    _b: i8, // Copy type ✅
}

// This will NOT compile!
#[derive(Clone, Copy)]
struct Demo {
    _a: u32, // a Copy type ✅
    _c: String // NOT a Copy type! ❌
}
```

### Conclusion
The difference between `Clone` and `Copy` traits become clear when you understand when and what values are stored on stack and heap. `Copy` is exclusively for types that can be pushed on the stack and `Clone` is for types that can be pushed on the stack as well as on the heap. 

Since making a copy of `Copy` types is cheap, it is an implicit process. Instead of a **move**, a `Copy` (like `u32`) type is copied when assigned to a variable. While making deep copy of a **Clone** type is expensive. So, to make a deep copy of a `Clone` type, you have to explicitly invoke the `clone()` method, otherwise a move happens (copy pointer on stack), which is only a stack operation, hence cheap.

Alright, I hope that clears the confusion between **Copy** and **Clone** traits in Rust!

## Resources
- [Understanding Ownership](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html)
- [Stack and Heap memory allocation](https://www.geeksforgeeks.org/stack-vs-heap-memory-allocation/)

- [Clone trait](https://doc.rust-lang.org/core/clone/trait.Clone.html)