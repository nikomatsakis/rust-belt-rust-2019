class: center
name: title
count: false

.Polonius[Polonius]

.me[.grey[*by* **Nicholas Matsakis**]]
.citation[`https://github.com/nikomatsakis/rust-belt-rust-2019`]

---

# Where did you get that name?

---

.center[
.p80[![Polonius?](https://media.giphy.com/media/3o6Mbgos1hJpqvDO00/giphy.gif)]
]

---

> Neither a borrower nor a lender be;
> For loan oft loses both itself and friend. <br/>
> <br/>
> -- Polonius to his son Laertes, Hamlet

--

**Little known fact:** Polonius was an experienced C hacker. This made
him a bit cautious.

**Good news:** He has since adopted Rust! 🎉

--

> Borrow, lend, whatever. The compiler's got your back. <br/>
> Go out and build your dreams! <br/>
> <br/>
> -- Polonius, today

---

# Polonius

* Work in progress
* Reimagining the Rust borrow checker
    * Accept more programs
    * Simpler definition

---

# This talk

* Define the classic borrow checker error
* Explain how borrow checker finds such an error today
* Explain how **Polonius** would find it
* Show how Polonius's approach can help with other cases

---

name: the-classic-borrow-checker-error

# The classic borrow checker error

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
*  |                   -- borrow of `x` occurs here
 6 |     x += 1;
*  |     ^^^^^^ assignment to borrowed `x` occurs here
 7 |     print(y);
*  |           - borrow later used here
```

[TkTTK]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=d7c424c90621f7eb5cf39667a4bc4fd4

---

template: the-classic-borrow-checker-error

.line2[![Point at `&x`](content/images/Arrow.png)]

---

template: the-classic-borrow-checker-error

.line3[![Point at `x+1`](content/images/Arrow.png)]

---

template: the-classic-borrow-checker-error

.line4[![Point at `print`](content/images/Arrow.png)]

---

# Let's try to make that more precise

* Error at some program statement *N* if:
    * the statement *N* accesses a path *P*

--

```rust
let mut x: u32 = 22;
let y: &u32 = &x;
*x += 1; // the statement N
print(y);
```

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

# Not paths

* `foo()` -- result of a function call
* `22` -- creates and returns an integer
* `&x` -- creates and returns a reference (to `x`)

--

"Could I assign to it?"

```rust
foo() = 22; // doesn't make sense, not a path
x[3].f = 22; // makes sense, `x[3].f` is a path
```

---

# You can only borrow a path

```rust
let y = &x; // borrows the path "x"
```

--

*"But what", you say, "can't I do this?"*

```rust
let y = &22;
```

--

*"Yes", I answer, "but it's desugared to this"*

```rust
static TMP: u32  = 22; // a "temporary"
let y = &TMP;
```

\* *Sometimes we introduce local variable temporaries as well.*

---

# Back to our error

* Error at some program statement *N* if:
    * the statement *N* accesses a path *P*

```rust
let mut x: u32 = 22;
let y: &u32 = &x;
x += 1; // the statement N modifies the path `x`
print(y);
```

---

# Back to our error

* Error at some program statement *N* if:
    * the statement *N* accesses a path *P*
    * Accessing the path *P* would violate the terms of some loan *L*

```rust
let mut x: u32 = 22;
let y: &u32 = &x;
x += 1; // the statement N modifies the path `x`
print(y);
```

---

# Definition: Loan

A **loan** is a name for a borrow expression like `&x`.

```rust
let y = &x;
```

Loans have associated with them:

* a **path** that was borrowed (here, `x`)
* a **mode** with which it was borrowed (either *shared* or *mutable*)

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
    * the statement *N* accesses a path *P*
    * Accessing the path *P* would violate the terms of some loan *L*

```rust
let mut x: u32 = 22;
let y: &u32 = &x; // loan L shares `x`
x += 1; // the statement N modifies the path `x`, violating L
print(y);
```

---

# Back to our error, again

* Error at some program statement *N* if:
    * the statement *N* accesses a path *P*
    * Accessing the path *P* would violate the terms of some loan *L*
    * the loan *L* is live

```rust
let mut x: u32 = 22;
let y: &u32 = &x; // loan L shares `x`
x += 1; // the statement N modifies the path `x`, violating L
print(y);
```

---

# Definition: Live

Something that might be used later

---

name: live-variables

# Definition: Live variables

```rust
let mut x = 1;

x = 2;

if something {
    x = 4;
}

print(x);
```

---

template: live-variables

.line2[![Point at `let mut x = 1`](content/images/Arrow.png)]

Is `x` live here? (Just after `x = 1`)

--

No -- it will be reassigned before it is used.

---

template: live-variables

.line3-after[![Point at `let mut x = 1`](content/images/Arrow.png)]

Is `x` live here? (Just after `x = 2`)

--

Yes -- current value *might* get used (but might not).

---

template: live-variables

.line4-after[![Point at `let mut x = 1`](content/images/Arrow.png)]

Is `x` live here? (Just *before* the `x=4` statement)

--

No -- current value *won't* get used.

---

template: live-variables

.line9-after[![Point at `let mut x = 1`](content/images/Arrow.png)]

Is `x` live here? (Just *after* the `print` statement)

--

No -- no more uses at all.


---

# Definition: Live loan

The reference created by the loan -- or some reference derived from it
-- might be used later.

--

```rust
let y = &foo;      // Loan L1
let z = &y.bar;    // Loan L2

// z will get used, so
// loans L1 and L2 are both live

print(z);
```

---

# A borrow check error

* Error at some program statement *N* if:
    * *N* accesses a path *P*
    * Accessing *P* would violate the terms of some loan *L*
    * the loan *L* is live

```rust
let mut x: u32 = 22;
let y: &u32 = &x; // loan L shares `x`
x += 1; // the statement N accesses the path `x`, violating L
print(y); // L is live because it might used here
```

---

name: how-do-we-decide-today

# So how do we decide if a loan is **live**?

## How the borrow checker does it **today**

* we compute a **lifetime** for each reference:
    * that part of the program where the reference may be used.

---

# What is a lifetime

```rust
let mut x: u32 = 22;
let y: &u32 = &x;
x += 1;
print(y);
```

---

# What is a lifetime

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &u32 = &x;
/*2*/ x += 1;
/*3*/ print(y);
```

lifetime = set of line numbers from the current function

---

name: what-is-a-lifetime

# What is a lifetime

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &'0 u32 = &'1 x;
/*2*/ x += 1;
/*3*/ print(y);
```

---

template: what-is-a-lifetime

* `'0` and `'1` are "inference variables"
* compiler has to figure out their value (some set of lines)

---

template: what-is-a-lifetime

**What is `'0`, the lifetime of the reference `y`?**

* Liveness rule: `'0` must include all lines where `y` is live

--
* `y` is live on lines 2 and 3

--
* **Result:** `'0` is the set `{2, 3}`

---

template: what-is-a-lifetime

**What about `'1`, lifetime of the reference returned by `&x`?**

* Subtyping rule: `'1` flows into `'0`, so `'1` must outlive `'0`

--
* **Result:** `'1` is also the set `{2, 3}`

---

# What is a lifetime (final result)

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &{2, 3} u32 = &{2, 3} x;
/*2*/ x += 1;
/*3*/ print(y);
```

---

template: how-do-we-decide-today

--

template: how-do-we-decide-today

* each loan creates a reference
  * the loan is live during the **lifetime of that reference**

---

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &{2, 3} u32 = &{2, 3} x;
/*2*/ x += 1;
/*3*/ print(y);
```

.line2-lifetime[![Point at `&x`](content/images/Arrow.png)]

* Error at some program statement *N* if:
    * the statement *N* accesses a path *P*
    * Accessing the path *P* would violate the terms of some loan *L*
    * the loan *L* is live


--

* Line 2 modifies the path `x`

--
* Modifying the path `x` violates the terms of `&x` loan from line 1

--
* Loan from line 1 is live on lines 2, and 3

--
* **Ergo:** Error!

---

# So how do we decide if a loan is **live**?

## How Polonius does it

---

name: how-polonius-decides

# So how do we decide if a loan is **live**?

## How Polonius does it

* we compute an **origin** for each reference *R*:
    * a set of loans indicating the loans *R* might have come from

---

name: what-is-an-origin

# Inferring origins

```rust
let mut x: u32 = 22;
let y: &'0 u32 = &'1 x /* Loan L1 */;
x += 1;
print(y);
```

---

template: what-is-an-origin

**What is the origin `'1`?**

The set `{L1}`.

---

template: what-is-an-origin

**What about the origin `'0`?**

Also the set `{L1}`.

---

# Inferring origins

```rust
let mut x: u32 = 22;
let y: &{L1} u32 = &{L1} x /* Loan L1 */;
x += 1;
print(y);
```

* Computing these origins only considered the flow of values
* Liveness is not relevant

---

template: how-polonius-decides

---

template: how-polonius-decides

* a loan *L* is live if **some live variable** has *L* in its type

---

```rust
/*0*/ let mut x: u32 = 22;
/*1*/ let y: &{L1} u32 = &{L1} x /* Loan L1 */;
/*2*/ x += 1;
/*3*/ print(y);
```

* Error at some program statement *N* if:
    * the statement *N* accesses a path *P*
    * Accessing the path *P* would violate the terms of some loan *L*
    * the loan *L* is live


--

* Line 2 modifies the path `x`

--
* Modifying the path `x` violates the terms of the loan `L1`

--
* `y` is live on line 2 and its type includes the loan `L1`
    * type of `y` is `&{L1} u32`

--

**Ergo:** Error!

---

name: where-polonius-can-help

# Where Polonius can help

```rust
fn get_or_insert(
    map: &mut HashMap<u32, String>,
) -> &String {
    match map.get(&22) {
        Some(v) => v,
        None => {
            map.insert(22, String::from("hi"));
            &map[&22]
        }
    }
}
```

---

template: where-polonius-can-help

.line2[![Point at `map`](content/images/Arrow.png)]

Given a map...

---

template: where-polonius-can-help

.line3[![Point at `map`](content/images/Arrow.png)]

Given a map... return a reference to a value in that map.

---

template: where-polonius-can-help

.line4[![Point at `map`](content/images/Arrow.png)]

Does the map have the key `22`?

---

template: where-polonius-can-help

.line5[![Point at `map`](content/images/Arrow.png)]

Does the map have the key `22`? If so, return it.

---

template: where-polonius-can-help

.line6[![Point at `map`](content/images/Arrow.png)]

Otherwise, insert a value and return that.

---

# What happens today?

If you [try this on the playground][wht-pg], you get:

```
error[E0502]: cannot borrow `*map` as mutable because also borrowed as immutable
 --> src/main.rs:11:13
  |
3 |   map: &mut HashMap<u32, String>,
* |        - let's call the lifetime of this reference `'1`
4 | ) -> &String {
5 |   match map.get(&22) {
* |         --- immutable borrow occurs here
6 |     Some(v) => v,
* |                - returning this value requires `*map` is borrowed for `'1`
7 |     None => {
8 |       map.insert(22, String::from("hi"));
* |       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ mutable borrow occurs here
```

[wht-pg]: https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=487acc6ab7b1a9e3538d026a3af7d0bc

---

name: lightly-desugared

# Lightly desugared

```rust
fn get_or_insert<'a>(
    map: &'a mut HashMap<u32, String>,
) -> &'a String {
    match HashMap::get(&*map, &22) {
        Some(v) => v,
        None => {
            map.insert(22, String::from("hi"));
            &map[&22]
        }
    }
}
```

---

template: lightly-desugared

.line1[![Point at `'a`](content/images/Arrow.png)]

---

template: lightly-desugared

.line4[![Point at `map`](content/images/Arrow.png)]

* Now we can see the loan, of the path `*map`
* **Problem:** Lifetime of this loan cannot be expressed as a set of
  line numbers. It is `'a`.

---

# Named lifetimes

```rust
        fn caller() {
/* 1 */     let mut map = HashMap::new();
/* 2 */     let v = get_or_insert(&mut map);
/* 3 */     print(v);
        }
        
        fn get_or_insert<'a>(
            map: &'a mut HashMap<u32, String>,
        ) -> &'a String {
            ...
        }
```

* `'a` really refers to some part of the caller
    * in this case, maybe lines 2 and 3
    * but that includes *all* of `get_or_insert`

---

name: named-lifetimes-nll

# Incorporating named lifetimes in NLL 

```rust
fn get_or_insert<'a>(
    map: &'a mut HashMap<u32, String>,
) -> &'a String {
    match HashMap::get(&*map, &22) {
        Some(v) => v,
        None => {
            map.insert(22, String::from("hi"));
            &map[&22]
        }
    }
}
```

---

template: named-lifetimes-nll

* Lifetimes in NLL can then be **either**:
    * a set of lines
    * a named lifetime like `'a`, which includes all lines

---

template: named-lifetimes-nll

* The loan here...

.line4[![Point at `insert`](content/images/Arrow.png)]

---

template: named-lifetimes-nll

* The loan here... has lifetime `'a`

.line5[![Point at `insert`](content/images/Arrow.png)]

---

template: named-lifetimes-nll

* The loan here... has lifetime `'a`
* And so it is live when we call `map.insert`

**Error.**

.line7[![Point at `insert`](content/images/Arrow.png)]

---

name: pc2-with-polonius

# With Polonius?

```rust
fn get_or_insert<'a>(
    map: &'a mut HashMap<u32, String>,
) -> &'a String {
    match HashMap::get(&*map, &22) {
        Some(v) => v,
        None => {
            map.insert(22, String::from("hi"));
            &map[&22]
        }
    }
}
```

---

template: pc2-with-polonius

* The loan here...

.line4[![Point at `insert`](content/images/Arrow.png)]

---

template: pc2-with-polonius

* The loan here...is part of the type of `v`

.line5[![Point at `insert`](content/images/Arrow.png)]

---

template: pc2-with-polonius

* The loan here...is part of the type of `v`
* But `v` is not live here, so `map.insert` is legal

**OK.**

.line7[![Point at `insert`](content/images/Arrow.png)]

---

# Where else can Polonius help?

--

.center[.HugeEmoji[⚠️]]

.center[**"Total speculation ahead"**]

---

# Where else can Polonius help?

```rust
struct Message {
    buffer: Vec<String>,
    slice: &'buffer [u8], // "borrowed from the field `buffer`"
}
```

--

* To create one of these, you need
    * a buffer
    * a `&[u8]` that was borrowed **from that buffer**
    
--


* How can we determine where a `&[u8]` was borrowed from?
    * Lifetimes are the wrong tool
    * Origins are the correct tool

---

# What is the status of Polonius

* The [Polonius WG] is exploring Polonius
* Currently working to extend rules to cover the full borrow checker
* Also exploring best way to express the rules to be both readable and efficient
    * Using datalog today

[Polonius WG]: https://rust-lang.github.io/compiler-team/working-groups/polonius/

---

# Thank you!

.center[.HugeEmoji[😍]]

