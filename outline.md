---
title: Polonius slides
tags: Talk
description: View the slide with "Slide Mode".
slideOptions:
    fragments: true
---

# Polonius

---

"Let's begin with the most important question"

**Where did you get that name?**

---

![Polonius?](https://media.giphy.com/media/3o6Mbgos1hJpqvDO00/giphy.gif)

---

> Neither a borrower nor a lender be;
> For loan oft loses both itself and friend.
> -- Polonius, Hamlet

---

**Little known fact:** Polonius was an experienced C hacker. This made him a bit cautious.

**Good news:** He has since adopted Rust. 

---

> Either a borrower nor a lender be;
> the compiler's got your back.
> -- Polonius, today

---

How Rust borrow checker works today

---

begin with the simplest example

```rust
let mut x: u32 = 22;
let y: &u32 = &x;
x += 1;
print(y);
```

* if you've used rust, you know that the `&x` borrow creates a reference
* as long as this reference is in use, `x` is locked

---

```rust
x += 1;
```

[gives us the error message][TkTTK]

```
error[E0506]: cannot assign to `x` because it is borrowed
 --> src/main.rs:6:5
   |
 5 |     let y: &u32 = &x;
   |                   -- borrow of `x` occurs here
 6 |     x += 1;
   |     ^^^^^^ assignment to borrowed `x` occurs here
 7 |     print(y);
   |           - borrow later used here
```

[TkTTK]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=d7c424c90621f7eb5cf39667a4bc4fd4

I like to think of this as a read-write lock per memory location.  The
shared borrow `&x` acquires a **read lock**. The read lock persists as
long as the resulting reference is in use (or the compiler thinks it
may be).

---

# Let's try to make that more precise

* Error at some program statement N if:
    * N accesses a path P

---

# Definition: Path

A **path** P is an expression that leads to a memory location.

Examples:

* `x` -- a local variable is a memory location on the stack
* `x.f` -- a field of another path is a memory location
* `*x` -- pass through a pointer to some location in the heap
* `x[_]` -- we don't care about indices

---

# Back to our error

* Error at some program statement N if:
    * N accesses a path P
    * P would violate the terms of some loan L

---

# Definition: Loan

A **loan** is a name for a borrow expression like `&x`.

Loans have associated with them:

* a **path** that was borrowed (here, `x`)
* a **mode** with which it was borrowed (either *shared* or *borrowed*)

---

# Violating the "terms" of a loan

For a **shared** loan of some path *P*

* Modifying the path *P* (directly or indirectly)

For a **mutable** loan of some path *P*

* Accessing the path *P* (directly or indirectly)

Directly or indirectly?

```rust
let p = &some_struct;
some_struct.field += 1; // still an error
print(p);
```

---

# Back to our error, again

* Error at some program statement N if:
    * N accesses a path P
    * P would violate the terms of some loan L
    * the loan L is live

---

# Definition: Live Loan

The reference created by the loan -- or some reference derived from it
-- might be used later.

---

# So how do we decide if a loan is **live**?

* How the borrow checker does it **today**:
    * we compute a **lifetime** for each reference: 
        * that part of the program where the reference may be used.
    * each loan creates a reference
        * the loan is live during the **lifetime of that reference**

---

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &'1 u32 = &'0 x;
/*2*/ x += 1;
/*3*/ print(y);
```

* Two lifetimes:
    * `'0` -- the lifetime of the reference created by `&u32`
    * `'1` -- the lifetime of the variable `y`
* Both are computed to be `{2, 3}`
* The loan is live in the span `'0`, or on lines 2 and 3

---

# NLL Computation in more detail

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &'1 u32 = &'0 x;
/*2*/ x += 1;
/*3*/ print(y);
```

* Two key components:
    * **live variables:** at each point in the program
    * **data flow:** relationships between regions
* In our example:
    * Line 1 requires `'0: '1`, because `&'0 x <: &'1 u32`
        * in NLL, we say `'0` **outlives** `'1`
    * The variable `y` is live on lines 2 and 3
        * Hence `'1` must include at least 2 and 3
* Put these together:
    * `'0 = '1 = {line 2, line 3}`

---

# So how do we decide if a loan is **live**?

* How **Polonius** does it:
    * we compute an **origin** for each reference *R*:
        * a set of loans indicating the loans *R* might have come from
    * at any given point in the program, look at the **references** which might be used
        * the union of their origins are the set of live loans

---

```rust
let mut x: u32 = 22;
let y: &'1 u32 = /*L0*/ &'0 x;
x += 1;
print(y);
```

* Two origins:
    * `'0` -- the origin of the reference created by `&u32`
    * `'1` -- the origin of the variable `y`
* Both are computed to be `{L0}`
* At `x += 1`: 
    * the variable `y` is live, 
    * the type of `y` is `&{L0} u32`,
    * therefore the loan `L0` is live

---

# Polonius computation in more detail

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &'1 u32 = &'0 x;
/*2*/ x += 1;
/*3*/ print(y);
```

* **Same** key components:
    * **live variables:** at each point in the program
    * **data flow:** relationships between regions
* But, to compute the origins, **only need data flow**:
    * `'0: '1` in Polonius means `'0` &sube; `'1`
        * `'1` might point anywhere that `'0` might point

---

# Polonius computation in more detail

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &'1 u32 = &'0 x;
/*2*/ x += 1;
/*3*/ print(y);
```

* Role of liveness:
    * when a variable is live, all loans in its type are live
* In this case:
    * `y` is live on lines 2 and 3
    * the type of `y` is `&'1 u32`, and `'1` is `{L0}`
    * everywhere `y` is live, `L0` is live, and `L0` 
        * therefore, `x` cannot be mutated if `y` is live

---

# In summary:

* Terms:
    * loan: a `&x` or `&mut x` expression, implicit or explicit
    * path: something like `x`, `x.y`, `*z[3].f`
    * origin: a set of loans, written as `'foo` in Rust syntax
* Two perspectives:
    * today: "how long do you use this reference"
    * polonius: "which loans could this reference have come from"

---

# Problem case #2

```rust
let mut map = HashMap::new();
match map.get("key") {
    Some(v) => {
        print(v);
    }
    
    None => {
        map.insert("key", "value");
    }
}
```

---

# Problem case #2, desugared

```rust
let mut map = HashMap::new();
let tmp = map.get("key");
if tmp.is_some() {
    print(tmp.unwrap());
} else {
    map.insert("key", "value");
}
```

---

# Problem case #2, desugared further

```rust
let mut map = HashMap::new();
let tmp: &'1 V = HashMap::get(&'0 map, "key");
if tmp.is_some() {
    print(tmp.unwrap());
} else {
    HashMap::insert(&'2 mut map, "key", "value");
}
```

---

# PC#2, NLL

```rust
/*0*/ let mut map = HashMap::new();
/*1*/ let tmp: &'1 V = HashMap::get(&'0 map, "key");
/*2*/ if tmp.is_some() {
    /*3*/ print(tmp.unwrap());
} else {
    /*4*/ HashMap::insert(&'2 mut map, "key", "value");
}
```

We compute:

* `'0` contains lines 1, 2, and 3
* `'1` the same
* `'2` contains line 4

---

# PC#2, Polonius

```rust
let mut map = HashMap::new();
let tmp: &'1 V = HashMap::get(&'0 map, "key"); // L_get
if tmp.is_some() {
    print(tmp.unwrap());
} else {
    HashMap::insert(&'2 mut map, "key", "value"); // L_insert
}
```

We compute the following origins for each reference:

* `'0` and `'1` are `{L_get}`
* `'2` is `{L_insert}`

But `tmp` is not live in the else branch, so `L_get` is not live there either

---

# Named lifetimes

```rust
fn get_or_insert<'a>(
    map: &'a mut HashMap<&'static str, &'static str>,
) -> &'a mut V {
    ... /* "problem case #3" */
}
```

---

```rust
fn get_or_insert<'a>(
    map: &'a mut HashMap<&'static str, &'static str>,
) -> &'a mut V {
    let tmp: &'1 V = HashMap::get(&'0 *map, "key");
    if tmp.is_some() {
        return tmp.unwrap();
    } else {
        HashMap::insert(&'2 mut *map, "key", "value");
    }
}
```

---

# Named lifetimes

* What *is* a named lifetime?
* In NLL, 


---

# Up is down and left is right

    * a set of statements, so to speak
* Polonius: "what might this reference point to"
    * a set of loans
* Consider `'static`:
    * Today: this reference is used from everywhere
        * the universal set containing *all* statements
    * Polonius: this reference came from static memory
        * the empty set

---

# 


XXX

so what have we introduced so far:

* Path
* Origin
* Loan

other atoms don't seem so important, so we've got the key cases:

* Node // control-flow graph?
* Variable // kind of a technicality

other points to raise:

* "problem case #3" and placeholder lifetimes
* why this should help us in solving interior references
* a closer look at the rules -- this probably means we need to decide which variant of the rules we are using
    * probably a good idea to start with the "subset rules" as originally expressed
    * maybe just kind of "glide" over that distinction
* killing seems rather more interesting 
