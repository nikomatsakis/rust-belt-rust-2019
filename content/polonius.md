class: center
name: title
count: false

.Polonius[Polonius]

.me[.grey[*by* **Nicholas Matsakis**]]
.citation[`https://github.com/nikomatsakis/rust-belt-rust-2019`]

---

# The most common question

--

**Where did you get that name?**

---

.center[
.p80[![Polonius?](https://media.giphy.com/media/3o6Mbgos1hJpqvDO00/giphy.gif)]
]

---

> Neither a borrower nor a lender be;
> For loan oft loses both itself and friend. <br/>
> <br/>
> -- Polonius, Hamlet

--

**Little known fact:** Polonius was an experienced C hacker. This made
him a bit cautious.

**Good news:** He has since adopted Rust. 

--

> Borrow, lend, whatever. The compiler's got your back. <br/> 
> <br/>
> -- Polonius, today

---

# How Rust borrow checker works today

--

```rust
let mut x: u32 = 22;
let y: &u32 = &x;
x += 1;
print(y);
```

---

name: how-rust-borrow-checker-works-today

# How Rust borrow checker works today

```rust
let mut x: u32 = 22;
let y: &u32 = &x;
x += 1;
print(y);
```

[try it on the playground, you get][TkTTK]

```bash
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

---

template: how-rust-borrow-checker-works-today

.line2[![Point at `&x`](content/images/Arrow.png)]

---

template: how-rust-borrow-checker-works-today

.line3[![Point at `x+1`](content/images/Arrow.png)]

---

template: how-rust-borrow-checker-works-today

.line4[![Point at `print`](content/images/Arrow.png)]

---

# Let's try to make that more precise

* Error at some program statement *N* if:
    * *N* accesses a path *P*
    
---

# Definition: Path

A **path** *P* is an expression that leads to a memory location.

Examples:

* `x` -- a local variable is a memory location on the stack
* `x.f` -- a field of another path is a memory location
* `*x` -- pass through a pointer to some location in the heap
* `*x.f` -- pass through a pointer found at field `f` from variable `x`
* `x[_]` -- index into an array (we don't care about the indices)

---

# Back to our error

* Error at some program statement *N* if:
    * *N* accesses a path *P*
    * Accessing *P* would violate the terms of some loan *L*

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

---

# Directly or indirectly?

## Direct

```rust
let mut x: u32 = 22;
let y: &u32 = &x;
x += 1; // error
print(y);
```

--

.line3-direct[![Point at `x+1`](content/images/Arrow.png)]

--

## Indirect

```rust
let mut x: SomeStruct = SomeStruct { field: 22 };
let y: &SomeStruct = &x;
x.field += 1; // still an error
print(y);
```

--

.line1-indirect[![Point at `SomeStruct`](content/images/Arrow.png)]

--

.line2-indirect[![Point at `&x`](content/images/Arrow.png)]

--

.line3-indirect[![Point at `x.field`](content/images/Arrow.png)]

---

# Back to our error, again

* Error at some program statement *N* if:
    * *N* accesses a path *P*
    * Accessing *P* would violate the terms of some loan *L*
    * the loan *L* is live

---

# Definition: Live

Something that might be used later

---

# Definition: Live loan

The reference created by the loan -- or some reference derived from it
-- might be used later.

--

(Actually, we'll refine this later, but it's good enough for now.)

---

# A borrow check error

* Error at some program statement *N* if:
    * *N* accesses a path *P*
    * Accessing *P* would violate the terms of some loan *L*
    * the loan *L* is live

---

name: how-do-we-decide-today

# So how do we decide if a loan is **live**?

## How the borrow checker does it **today**

* we compute a **lifetime** for each reference: 
    * that part of the program where the reference may be used.

---

name: what-is-a-lifetime

# What is a lifetime

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &u32 = &x;
/*2*/ x += 1;
/*3*/ print(y);
```

--

template: what-is-a-lifetime

* What is lifetime of the reference `y`?
    * Lines 2 and 3 -- it's not used after that

---

template: what-is-a-lifetime

* What about reference returned by `&x`?
    * Lines 1, 2, and 3
    * On line 1, it is created and then stored into `y`
    * And then `y` is live on lines 2 and 3

---

template: how-do-we-decide-today

--

template: how-do-we-decide-today

* each loan creates a reference
  * the loan is live during the **lifetime of that reference**

---

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &{2, 3} u32 = &{1, 2, 3} x;
/*2*/ x += 1;
/*3*/ print(y);
```

* Line 2 modifies the path `x`

--
* Modifying the path `x` violates the terms of `&x` loan from line 1

--
* Loan from line 1 is live on lines 1, 2, and 3

--
* **Ergo:** Error!

---

# So how do we decide if a loan is **live**?

## How Polonus does it

---

# So how do we decide if a loan is **live**?

## How Polonus does it

* we compute an **origin** for each reference *R*:
    * a set of loans indicating the loans *R* might have come from
    
    
