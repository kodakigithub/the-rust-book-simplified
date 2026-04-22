
# Day 20 — Advanced Rust Features


What are we going to cover:

<img width="296" height="179" alt="Screenshot 2026-04-23 at 3 20 58 AM" src="https://github.com/user-attachments/assets/f4b8c1b1-5314-411c-be58-1a6331e3eec8" />
 


By this point you have covered ownership, borrowing, lifetimes, traits, generics, iterators, concurrency, and smart pointers. You know how Rust thinks. This chapter is about the parts of Rust that exist specifically for when the normal rules are not enough  when you need to reach deeper, express more, or eliminate more repetition than safe everyday Rust allows.
 
None of this is exotic. The standard library is built almost entirely on these features. Every `Vec`, `String`, `Arc`, and `Box` you have ever used relies on unsafe internals, associated types, trait objects, and macros. Understanding this chapter means you can finally read that source code and understand what is actually happening.

## 1. Unsafe Rust
 
Rust runs in two modes. Safe Rust is what you write every day ,the compiler checks everything and guarantees no undefined behavior. Unsafe Rust is a small, explicit escape hatch that says: I know more than the compiler does here, and I am taking responsibility for it.
 
The important thing to understand is that writing `unsafe` does not disable the borrow checker or turn off type checking. It just unlocks five specific capabilities that the compiler cannot verify on its own. Everything else still applies.

### 1.1 Why Unsafe Exists
 
The Rust compiler is conservative by design. If it cannot prove something is safe, it rejects it even if you know it actually is. There are real situations where this is too limiting:
 
- Calling a function from a C library (FFI) where no Rust safety guarantees exist
- Writing a custom memory allocator that needs to do raw pointer arithmetic
- Implementing operating system or embedded code that must interact with hardware directly
- Building safe abstractions like `Vec` or `Mutex` that need unsafe internals to work
The solution is not to make the compiler smarter  it is to give the programmer a controlled way to say "trust me here." That is `unsafe`.

### 1.2 The Five Unsafe Superpowers
 
You wrap unsafe operations in an `unsafe` block:
 
```rust
unsafe {
    // only in here can you use the five superpowers
}
```
 
| Superpower | What It Allows |
|---|---|
| Dereference raw pointers | Read or write through `*const T` and `*mut T` |
| Call unsafe functions | Call functions marked `unsafe fn` |
| Access mutable static variables | Read or write a global mutable `static` |
| Implement unsafe traits | Manually implement `Send`, `Sync`, or similar |
| Access union fields | Read fields from a C-style union |

 
### 1.3 Raw Pointers
 
Raw pointers are the most common reason to use unsafe. They come in two forms:
 
- `*const T` — immutable raw pointer
- `*mut T` — mutable raw pointer
Unlike references, raw pointers have no rules. They can be null. You can have both a `*const` and `*mut` to the same memory at the same time. They are not cleaned up automatically. They might point to freed or invalid memory and the compiler will not warn you.
 
You can create them in safe code  dereferencing them is what requires unsafe:
 
```rust
fn main() {
    let mut num = 5;
 
    let r1 = &num as *const i32;      // immutable raw pointer
    let r2 = &mut num as *mut i32;    // mutable raw pointer
 
    // creating the pointers above is safe
    // reading through them is not
    unsafe {
        println!("r1 points to: {}", *r1);
        println!("r2 points to: {}", *r2);
    }
}
```
 
You can also create a raw pointer to an arbitrary memory address. The compiler will not stop you  but this is genuinely dangerous:
 
```rust
let mut x = 10;
let r = &x as *const i32;

unsafe {
    println!("{}", *r);
}
```
 
This is why it lives behind unsafe. There is no way for the compiler to know if that address is valid. You are telling it you accept the risk.
 
A practical real-world example  splitting a mutable slice into two non-overlapping parts. Safe Rust cannot express this because the borrow checker would see two mutable borrows of the same slice:
 
```rust
fn split_at_mut_manual(slice: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = slice.len();
    let ptr = slice.as_mut_ptr();
 
    assert!(mid <= len);
 
    unsafe {
        (
            std::slice::from_raw_parts_mut(ptr, mid),
            std::slice::from_raw_parts_mut(ptr.add(mid), len - mid),
        )
    }
}
 
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];
    let (left, right) = split_at_mut_manual(&mut v, 3);
    println!("{:?} and {:?}", left, right); // [1, 2, 3] and [4, 5]
}
```
 
We know the two halves do not overlap  `ptr` and `ptr.add(mid)` point to different parts of the same allocation. The compiler cannot figure that out, so we use unsafe to express it.
### 1.4 Unsafe Functions
 
Some functions have preconditions that cannot be expressed in the type system. If those preconditions are violated, the behavior is undefined. Rust marks them `unsafe fn`:
 
```rust
unsafe fn overwrite_memory(ptr: *mut i32, val: i32) {
    *ptr = val;
}
 
fn main() {
    let mut x = 10;
    unsafe {
        overwrite_memory(&mut x as *mut i32, 99);
    }
    println!("{}", x); // 99
}
```
 
Calling an unsafe function forces you to write an unsafe block at the call site. This is intentional  it is a visible marker in code review that says "someone made a deliberate decision here."
 
Here is a more realistic example using `std::str::from_utf8_unchecked`. The safe version returns a `Result` in case the bytes are not valid UTF-8. The unchecked version skips validation entirely  faster, but undefined behavior if the bytes are invalid:
 
✅ Safe and correct:
 
```rust
fn main() {
    let bytes = b"hello world"; // we know this is valid UTF-8
 
    let s = unsafe {
        std::str::from_utf8_unchecked(bytes)
    };
 
    println!("{}", s); // hello world
}
```
 
❌ Unsafe and wrong — undefined behavior:
 
```rust
fn main() {
    let invalid_bytes: &[u8] = &[0xFF, 0xFE, 0xAA];
 
    let s = unsafe {
        std::str::from_utf8_unchecked(invalid_bytes) // UB — not valid UTF-8
    };
 
    println!("{}", s); // undefined behavior may produce invalid &str, violating Rust’s UTF-8 guarantees
}
```
### 1.5 Mutable Static Variables
 
Global variables in Rust are `static`. Reading an immutable static is fine. Reading or writing a mutable static requires unsafe because two threads could race on it:
 
```rust
static mut REQUEST_COUNT: u64 = 0;

unsafe fn increment() {
    REQUEST_COUNT += 1;
}

fn main() {
    unsafe { increment(); }
    unsafe { increment(); }
    unsafe { increment(); }

    unsafe {
        println!("Handled {} requests", REQUEST_COUNT); // 3
    }
}
```
Data races  (undefined behavior even if it ‘seems to work.)
This is fine in single-threaded code. In multi-threaded code, two threads incrementing at the same time is a data race undefined behavior. For thread-safe global state, use `std::sync::atomic` types or `Mutex<T>` instead.
 
 ### 1.6 Unsafe Trait Implementations
 
Some traits have safety invariants that cannot be checked by the compiler. `Send` (a type is safe to send to another thread) and `Sync` (a type is safe to share between threads) are the main examples.
 
Rust automatically implements these for types it can prove are safe. But if you write a type wrapping a raw pointer, the compiler cannot prove safety:
 
```rust
struct MyBuffer {
    ptr: *mut u8,
    len: usize,
}
 
// We know our buffer is safe to send across threads
// because we manage the pointer carefully ourselves
unsafe impl Send for MyBuffer {}
unsafe impl Sync for MyBuffer {}
```
 
By writing `unsafe impl`, you are asserting that you have verified the safety contract yourself.
 
### 1.7 The Golden Rule of Unsafe
 
Wrap unsafe code in safe abstractions. The unsafe block should be as small as possible  only the lines that actually need it. The public API around it should be fully safe. This is exactly how the standard library works:
 
```rust
// External usage — completely safe for the caller
fn main() {
    let mut v = vec![1, 2, 3, 4, 5];
    let (a, b) = v.split_at_mut(3); // no unsafe needed here
    println!("{:?} {:?}", a, b);    // [1, 2, 3] [4, 5]
}
```
 
The unsafe raw pointer work is hidden inside `split_at_mut`'s implementation. Users get a safe, clean API. That is the model to follow.
 
## 2. Advanced Traits
 
You already know the basics of traits  `impl Trait for Type`, generic bounds, `dyn Trait`. Advanced traits are about the situations where those basics are not precise enough: when you need to fix a type inside a trait, overload an operator, resolve ambiguous method names, require another trait as a prerequisite, or implement an external trait for an external type.

### 2.1 Associated Types
 
Associated types let a trait define a placeholder type that implementing types must fill in. The most important example in Rust is `Iterator`:
 
```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```
 
When you implement `Iterator` for a type, you specify exactly what `Item` is:
 
```rust
struct Counter {
    count: u32,
    max: u32,
}
 
impl Iterator for Counter {
    type Item = u32;
 
    fn next(&mut self) -> Option<Self::Item> {
        if self.count < self.max {
            self.count += 1;
            Some(self.count)
        } else {
            None
        }
    }
}
 
fn main() {
    let counter = Counter { count: 0, max: 3 };
    let values: Vec<u32> = counter.collect();
    println!("{:?}", values); // [1, 2, 3]
}
```
 
**Why not just use a generic parameter instead?**
 
Compare these two designs:
 
```rust
// With associated type  each type implements Iterator exactly once
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
 
// With generic parameter  a type could implement Iterator<String>, Iterator<i32>, etc.
trait Iterator<T> {
    fn next(&mut self) -> Option<T>;
}
```
 
With the generic version, `Counter` could implement `Iterator<u32>` and `Iterator<String>` at the same time. That sounds flexible but creates a problem  every time you call `.next()`, the compiler does not know which one you mean. You have to annotate it everywhere.
 
With associated types, the Item is fixed when you implement the trait. The compiler always knows. Callers never have to specify it.
 
| Feature | Associated Type | Generic Parameter |
|---|---|---|
| Multiple impls per type? | No — one only | Yes — one per type argument |
| Annotation at call sites? | Not needed | Often required |
| Best for | 1:1 relationships like Iterator, Display | When multiple impls make sense |
 
### 2.2 Default Generic Type Parameters and Operator Overloading
 
Generic type parameters can have defaults. The `Add` trait uses this to make adding a type to itself the default behavior:
 
```rust
// Rhs = Self means: unless specified, you are adding the type to itself
pub trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Self::Output;
}
```
 
Implementing `Add` to overload the `+` operator on a custom type:
 
```rust
use std::ops::Add;
 
#[derive(Debug, PartialEq, Clone, Copy)]
struct Vec2 {
    x: f64,
    y: f64,
}
 
impl Add for Vec2 {
    type Output = Vec2;
 
    fn add(self, other: Vec2) -> Vec2 {
        Vec2 {
            x: self.x + other.x,
            y: self.y + other.y,
        }
    }
}
 
fn main() {
    let a = Vec2 { x: 1.0, y: 2.0 };
    let b = Vec2 { x: 3.0, y: -1.0 };
    let c = a + b;
    println!("{:?}", c); // Vec2 { x: 4.0, y: 1.0 }
}
```
 
Now a version with a custom `Rhs`  adding two different types together:
 
```rust
use std::ops::Add;
 
#[derive(Debug)]
struct Meters(f64);
 
#[derive(Debug)]
struct Centimeters(f64);
 
impl Add<Centimeters> for Meters {
    type Output = Meters;
 
    fn add(self, other: Centimeters) -> Meters {
        Meters(self.0 + other.0 / 100.0)
    }
}
 
fn main() {
    let distance = Meters(1.5) + Centimeters(75.0);
    println!("{:?}", distance); // Meters(2.25)
}
```
 
### 2.3 Fully Qualified Syntax — Disambiguating Method Calls
 
What happens when two traits have a method with the same name, and a type implements both? Rust needs a way to say which one you mean.
 
```rust
trait Pilot {
    fn fly(&self);
}
 
trait Wizard {
    fn fly(&self);
}
 
struct Human;
 
impl Pilot for Human {
    fn fly(&self) {
        println!("Flying the plane");
    }
}
 
impl Wizard for Human {
    fn fly(&self) {
        println!("Casting levitation spell");
    }
}
 
impl Human {
    fn fly(&self) {
        println!("Waving arms frantically");
    }
}
```
 
Calling `.fly()` directly picks the type's own method. To call a specific trait's version:
 
```rust
fn main() {
    let person = Human;
 
    person.fly();          // Human's own method — "Waving arms frantically"
    Pilot::fly(&person);   // Pilot's version — "Flying the plane"
    Wizard::fly(&person);  // Wizard's version — "Casting levitation spell"
}
```
 
For associated functions (no `self`), use fully qualified syntax:
 
```rust
trait Animal {
    fn baby_name() -> String;
}
 
struct Dog;
 
impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}
 
impl Dog {
    fn baby_name() -> String {
        String::from("spot")
    }
}
 
fn main() {
    println!("{}", Dog::baby_name());             // "spot" — Dog's own function
    println!("{}", <Dog as Animal>::baby_name()); // "puppy" — Animal's version
}
```
 
The pattern `<Type as Trait>::function(args)` is fully qualified syntax. You only need it when there is genuine ambiguity.
 
### 2.4 Supertraits
 
Sometimes a trait only makes sense when another trait is already implemented. You express that dependency by listing it as a supertrait bound:
 
```rust
use std::fmt;
 
// OutlinePrint only works for types that implement Display
trait OutlinePrint: fmt::Display {
    fn outline_print(&self) {
        let output = self.to_string(); // works because Display is required
        let len = output.len();
        let border = "*".repeat(len + 4);
        println!("{}", border);
        println!("* {} *", output);
        println!("{}", border);
    }
}
```
 
❌ This fails  `Point` does not implement `Display` yet:
 
```rust
struct Point { x: f64, y: f64 }
 
impl OutlinePrint for Point {}
// error: `Point` doesn't implement `std::fmt::Display`
```
 
✅ Fix by implementing both:
 
```rust
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
 
impl OutlinePrint for Point {}
 
fn main() {
    let p = Point { x: 3.0, y: 7.5 };
    p.outline_print();
    // **************
    // * (3, 7.5) *
    // **************
}
```
 
### 2.5 The Newtype Pattern for External Traits
 
Rust has an orphan rule: you can only implement a trait for a type if either the trait or the type is defined in your own crate. You cannot implement `Display` for `Vec<String>` both are from the standard library.
 
The workaround is the newtype pattern. Wrap the external type in a new struct that belongs to your crate:
 
✅ Works  `Wrapper` is our type:
 
```rust
use std::fmt;
 
struct Wrapper(Vec<String>);
 
impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}
 
fn main() {
    let w = Wrapper(vec![
        String::from("hello"),
        String::from("world"),
    ]);
    println!("{}", w); // [hello, world]
}
```
 
❌ Does not work  both `Display` and `Vec` are external:
 
```rust
use std::fmt;
 
impl fmt::Display for Vec<String> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "{:?}", self)
    }
}
// error[E0117]: only traits defined in the current crate can be implemented for types defined outside the crate
```
 
The downside of newtype is that `Wrapper` does not automatically have all of `Vec`'s methods. You either implement `Deref` to forward them, or add the methods you need manually.
 
---
 
## 3. Advanced Types
 
Rust's type system has a few tools beyond everyday structs and enums. This section covers three: type aliases for reducing noise, the never type for expressing divergence, and dynamically sized types for understanding why some types must always live behind a pointer.
 
### 3.1 Type Aliases
 
Type aliases give a shorter name to a complex type. They do not create a new type  they are purely a rename. Use `type` to define them:
 
```rust
type Kilometers = i32;
 
fn main() {
    let x: i32 = 5;
    let y: Kilometers = 10;
 
    // Kilometers IS i32 — they can be added directly
    println!("{}", x + y); // 15
}
```
 
This is not the same as newtype. `Kilometers` and `i32` are identical to the compiler  no additional type safety, just readability.
 
The real value is with complex types that appear repeatedly. A common pattern in async Rust:
 
```rust
// Without alias — you write this everywhere
fn load_config() -> Box<dyn std::error::Error + Send + Sync + 'static> { todo!() }
fn save_config() -> Box<dyn std::error::Error + Send + Sync + 'static> { todo!() }
fn delete_config() -> Box<dyn std::error::Error + Send + Sync + 'static> { todo!() }
 
// With alias — clean and consistent
type AppError = Box<dyn std::error::Error + Send + Sync + 'static>;
 
fn load_config() -> Result<(), AppError> { todo!() }
fn save_config() -> Result<(), AppError> { todo!() }
fn delete_config() -> Result<(), AppError> { todo!() }
```
 
Another common alias  storing callbacks:
 
```rust
type Callback = Box<dyn Fn(i32) -> i32 + Send + 'static>;
 
fn register_handler(cb: Callback) {
    println!("Result: {}", cb(42));
}
 
fn main() {
    let double: Callback = Box::new(|x| x * 2);
    register_handler(double); // Result: 84
}
```
 
### 3.2 The Never Type
 
Rust has a special type written `!` called the never type. It represents computations that never produce a value. Functions that never return use it as their return type:
 
```rust
fn crash() -> ! {
    panic!("this function never comes back");
}
 
fn loop_forever() -> ! {
    loop {
        // no break — runs forever
    }
}
```
 
The never type is special in the type system: it can be coerced into any other type. This is why `continue`, `break`, `panic!`, and `return` work inside expressions that expect a value:
 
```rust
fn main() {
    let number: u32 = loop {
        let input = "42";
 
        // This match must produce a u32
        // The Err arm uses continue — which has type !
        // ! can become any type, so the match is valid
        let n: u32 = match input.trim().parse() {
            Ok(num) => num,
            Err(_) => continue,
        };
 
        break n;
    };
 
    println!("{}", number); // 42
}
```
 
The same reason `panic!` works in match arms that need a value:
 
```rust
fn get_value(map: &std::collections::HashMap<&str, i32>, key: &str) -> i32 {
    *map.get(key).unwrap_or_else(|| panic!("key '{}' not found", key))
}
```
// `panic!` returns `!`, the never type, which coerces to any type — including the `&i32` that `unwrap_or_else` expects. The `*` then dereferences `&i32` to `i32`. If execution reaches the panic, no value is ever needed, so no type mismatch occurs.
 
### 3.3 Dynamically Sized Types and Sized
 
Most types in Rust have a size known at compile time. `i32` is always 4 bytes. `bool` is always 1 byte. A struct with three `u64` fields is always 24 bytes. The compiler uses these sizes to allocate stack space.
 
Some types do not have a known size. These are called dynamically sized types (DSTs):
 
| Type | Known size? | Why? |
|---|---|---|
| `str` | No | String content length varies at runtime |
| `[T]` | No | Slice length varies at runtime |
| `dyn Trait` | No | The concrete type behind the trait object varies |
| `i32`, `bool`, structs | Yes | Size is fixed and known at compile time |
 
DSTs cannot be used as standalone stack values because their size is unknown at compile time the compiler would not know how many bytes to put on the stack. You always use them behind a pointer:
 
❌ These do not compile:
 
```rust
let s: str = "hello";          // unknown size
let slice: [i32] = [1, 2, 3]; // unknown size
```
 
✅ These work — behind a pointer:
 
```rust
let s: &str = "hello";
let boxed: Box<str> = "hello".into();
let slice: &[i32] = &[1, 2, 3];
```
 
Rust's implicit `Sized` bound is why generic functions work at all. This:
 
```rust
fn print_it<T>(item: T) {
    println!("{:?}", item);
}
```
 
actually means this behind the scenes:
 
```rust
fn print_it<T: Sized>(item: T) {
    println!("{:?}", item);
}
```
 
The compiler adds `T: Sized` automatically, which is why you can put `T` on the stack. If you want a generic function that also accepts DSTs, opt out with `?Sized`:
 
```rust
fn print_it<T: ?Sized + std::fmt::Debug>(item: &T) {
    // item must be behind a reference because T might not have a known size
    println!("{:?}", item);
}
 
fn main() {
    print_it(&42);         // T = i32, which is Sized — works
    print_it("hello");     // T = str, which is NOT Sized — also works because of ?Sized
    print_it(&[1, 2, 3]);  // T = [i32], not Sized — works too
}
```
 
---
 
## 4. Advanced Functions and Closures
 
You have been using closures throughout the book  in iterators, threads, and error handling. This section is about what is happening underneath: how functions and closures relate to each other as types, how to pass and store them, and why returning a closure is harder than it looks.
 
### 4.1 Function Pointers
 
Regular functions have a type too. The type of a function that takes an `i32` and returns an `i32` is written `fn(i32) -> i32`. This is a function pointer, and it uses lowercase `fn` (not the `Fn` trait).
 
```rust
fn double(x: i32) -> i32 {
    x * 2
}
 
fn apply(f: fn(i32) -> i32, value: i32) -> i32 {
    f(value)
}
 
fn main() {
    let result = apply(double, 5);
    println!("{}", result); // 10
 
    // You can assign functions to variables too
    let op: fn(i32) -> i32 = double;
    println!("{}", op(7)); // 14
}
```
 
Function pointers implement all three closure traits (`Fn`, `FnMut`, `FnOnce`), so anywhere you accept a `Fn(i32) -> i32`, you can pass a plain function:
 
```rust
fn triple(x: &i32) -> i32 { x * 3 }
 
fn main() {
    let numbers = vec![1, 2, 3, 4, 5];
 
    // Closure
    let doubled: Vec<i32> = numbers.iter().map(|x| x * 2).collect();
 
    // Function pointer — works the same way
    let tripled: Vec<i32> = numbers.iter().map(triple).collect();
 
    println!("{:?}", doubled); // [2, 4, 6, 8, 10]
    println!("{:?}", tripled); // [3, 6, 9, 12, 15]
}
```
 
Tuple struct constructors are function pointers too  a useful trick:
 
```rust
#[derive(Debug)]
struct Score(u32);
 
fn main() {
    // Score is fn(u32) -> Score, so it works directly in map
    let scores: Vec<Score> = vec![10, 20, 30]
        .into_iter()
        .map(Score)
        .collect();
 
    println!("{:?}", scores); // [Score(10), Score(20), Score(30)]
}
```
 
### 4.2 The Difference Between fn and Fn
 
| | `fn` | `Fn` / `FnMut` / `FnOnce` |
|---|---|---|
| What it is | A concrete type a pointer to a function | A trait that closures (and `fn` pointers) implement |
| Can capture environment? | No  just a pointer to code | Yes closures capture variables from scope |
| Nullable? | Yes — `Option<fn(i32)>` works directly | Yes — use `Option<Box<dyn Fn(i32)>>` |
| Use in no-std / FFI? | Yes | Usually no |
| Best for | Callbacks that need no state | Callbacks that close over data |
 
```rust
// fn works here — no capture needed
fn apply_twice(f: fn(i32) -> i32, x: i32) -> i32 {
    f(f(x))
}
 
fn main() {
    println!("{}", apply_twice(|x| x + 3, 10)); // 16  closure with no capture, acts like fn
 
    // This closure captures `offset`  cannot use fn, must use Fn trait
    let offset = 5;
 
    fn apply_twice_generic<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 {
        f(f(x))
    }
 
    println!("{}", apply_twice_generic(|x| x + offset, 10)); // 20
}
```
 
### 4.3 Returning Closures
 
Passing closures into functions is straightforward. Returning them is harder  because every closure has a unique, anonymous type that you cannot write out.
 
❌ Does not compile:
 
```rust
fn make_adder(n: i32) -> dyn Fn(i32) -> i32 {
    |x| x + n
}
// error: the size of the return type `dyn Fn(i32) -> i32` cannot be known at compile time
```
 
The fix is to box it. `Box<dyn Fn(...)>` is the standard pattern for returning closures:
 
✅ Works:
 
```rust
fn make_adder(n: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |x| x + n)
}
 
fn main() {
    let add5 = make_adder(5);
    let add10 = make_adder(10);
 
    println!("{}", add5(3));  // 8
    println!("{}", add10(3)); // 13
}
```
 
When there is only one concrete closure type being returned, you can use `impl Fn(...)` instead  no heap allocation, no dynamic dispatch:
 
```rust
fn make_multiplier(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x * n
}
 
fn main() {
    let triple = make_multiplier(3);
    println!("{}", triple(7)); // 21
}
```
 
The difference: `impl Fn` is zero cost but only works when all return paths return the same closure type. If the returned closure depends on a runtime condition, you must use `Box<dyn Fn>`:
 
```rust
fn make_transform(double: bool) -> Box<dyn Fn(i32) -> i32> {
    if double {
        Box::new(|x| x * 2)  // one closure type
    } else {
        Box::new(|x| x + 1)  // different closure type
    }
    // impl Fn would not work here — two different types
}
 
fn main() {
    let f = make_transform(true);
    println!("{}", f(5));  // 10
 
    let g = make_transform(false);
    println!("{}", g(5));  // 6
}
```
 
---
 
## 5. Macros
 
Macros are one of the most distinctive features of Rust. You have used them throughout the book `println!`, `vec!`, `assert_eq!`, `#[derive(Debug)]`. The `!` at the end of a macro call (or the `#[...]` syntax for attribute macros) is the signal that this is not a regular function.
 
The fundamental difference is when they run. Functions run at runtime. Macros are expanded at compile time  they take code as input and produce code as output. That code then gets compiled normally.
 
This distinction gives macros capabilities that functions simply do not have.
 
### 5.1 What Macros Can Do That Functions Cannot
 
| Capability | Function | Macro |
|---|---|---|
| Variable number of arguments | No arity is fixed | Yes  `println!` takes any number |
| Generate code at compile time | No | Yes  can emit entire `impl` blocks |
| Work on syntax, not values | No | Yes  matches patterns in token streams |
| Implement traits automatically | No | Yes `#[derive(...)]` does this |
| Domain-specific languages | No | Yes  SQL, HTML, regex |
 
### 5.2 Declarative Macros with macro_rules!
 
Declarative macros are defined with `macro_rules!` and work like a match statement  you write patterns for the input tokens, and each pattern maps to a code expansion.
 
Here is the simplified version of how `vec!` is implemented:
 
```rust
#[macro_export]
macro_rules! my_vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = std::vec::Vec::new();
            $( temp_vec.push($x); )*
            temp_vec
        }
    };
}
```
 
Breaking down the syntax:
 
- `$( $x:expr ),*`  match zero or more comma-separated expressions, bind each to `$x`
- `$( temp_vec.push($x); )*`  repeat this statement once per captured `$x`
So `my_vec![1, 2, 3]` expands at compile time to exactly:
 
```rust
{
   let mut temp_vec = std::vec::Vec::new();
    temp_vec.push(1);
    temp_vec.push(2);
    temp_vec.push(3);
    temp_vec
}
```
 
No runtime cost for the variadic behavior — the compiler generates the exact code needed.
 
Here is a custom macro that creates a `HashMap` in one expression:
 
```rust
use std::collections::HashMap;

macro_rules! map {
    ( $( $key:expr => $val:expr ),* $(,)? ) => {
        {
            let mut m = std::collections::HashMap::new();
            $( m.insert($key, $val); )*
            m
        }
    };
}
 
fn main() {
    let config = map! {
        "host" => "localhost",
        "port" => "8080",
        "debug" => "true",
    };
 
    println!("{:?}", config.get("host")); // Some("localhost")
}
```
 
The `$(,)?` at the end allows an optional trailing comma  consistent with how Rust's own macros behave.
 
Here is a macro that generates assertion messages with more context than the standard `assert!`:
 
```rust
macro_rules! assert_between {
    ($val:expr, $low:expr, $high:expr) => {
        if $val < $low || $val > $high {
            panic!(
                "assertion failed: {} is not between {} and {}",
                $val, $low, $high
            );
        }
    };
}
 
fn main() {
    assert_between!(5, 1, 10);  // passes silently
    assert_between!(15, 1, 10); // panics: assertion failed: 15 is not between 1 and 10
}
```
 
And one that generates multiple similar functions from a pattern  this is where macros get genuinely useful:
 
```rust
macro_rules! make_greeting {
    ($name:ident, $greeting:literal) => {
        fn $name(who: &str) {
            println!("{}, {}!", $greeting, who);
        }
    };
}
 
make_greeting!(say_hello, "Hello");
make_greeting!(say_goodbye, "Goodbye");
make_greeting!(say_welcome, "Welcome");
 
fn main() {
    say_hello("Alice");    // Hello, Alice!
    say_goodbye("Bob");    // Goodbye, Bob!
    say_welcome("Charlie"); // Welcome, Charlie!
}
```
 
### 5.3 Macro Metavariable Types
 
In `macro_rules!`, each metavariable has a designator that says what kind of syntax it matches:
 
| Designator | Matches | Example |
|---|---|---|
| `expr` | Any expression | `5 + 3`, `"hello"`, `foo()` |
| `stmt` | A statement | `let x = 5;` |
| `ty` | A type | `i32`, `Vec<String>` |
| `ident` | An identifier | `my_var`, `SomeStruct` |
| `pat` | A pattern | `Some(x)`, `(a, b)` |
| `path` | A path | `std::io::Read` |
| `tt` | Any single token tree | Anything at all |
| `literal` | A literal value | `42`, `"hello"`, `true` |
 
### 5.4 Procedural Macros
 
Procedural macros are more powerful. Instead of pattern matching, they receive a stream of Rust tokens as input, process it using actual Rust code, and return a new stream of tokens. They live in a separate crate with `proc-macro = true` in `Cargo.toml`.
 
There are three kinds:
 
**Custom derive macros** — `#[derive(YourTrait)]`
 
```rust
// In a separate proc-macro crate:
use proc_macro::TokenStream;
use quote::quote;
use syn;
 
#[proc_macro_derive(Describe)]
pub fn describe_derive(input: TokenStream) -> TokenStream {
    let ast: syn::DeriveInput = syn::parse(input).unwrap();
    let name = &ast.ident; // the name of the struct being derived on
 
    let expanded = quote! {
        impl Describe for #name {
            fn describe() -> String {
                format!("I am a struct named {}", stringify!(#name))
            }
        }
    };
 
    expanded.into()
}
```
 
```rust
// In user code:
#[derive(Describe)]
struct Pancakes;
 
fn main() {
    println!("{}", Pancakes::describe()); // I am a struct named Pancakes
}
```
 
**Attribute macros** — `#[your_attribute]` that transforms the item it is attached to. This is how web frameworks like Actix define routes:
 
```rust
// The macro generates the server routing registration code automatically
#[get("/hello")]
async fn handle_hello() -> impl Responder {
    HttpResponse::Ok().body("Hello!")
}
```
 
**Function-like macros** — look like `name!(...)` but process arbitrary token streams. Used for things like compile-time SQL validation:
 
```rust
// The macro can parse and validate the SQL at compile time, not runtime
let query = sql!(SELECT id, name FROM users WHERE active = true);
```
 
### 5.5 When to Use Macros vs Functions
 
The temptation when first learning macros is to use them for everything. Resist that. Functions are easier to read, easier to debug (they show up in stack traces with meaningful names), and much easier to reason about. Reach for macros only when a function genuinely cannot do the job:
 
| Situation | Use a Function | Use a Macro |
|---|---|---|
| Fixed number of arguments, known types | Yes | No |
| Variable number of arguments | No | Yes |
| Generating code from a repeating pattern | No | Yes |
| Implementing a trait for many types | No | Yes — derive macro |
| You want stack traces for debugging | Yes | No — macros expand before compile |
| Domain-specific syntax (SQL, HTML, regex) | No | Yes — proc macro |
| Simple reuse and abstraction | Yes | No — overkill |
 
A good rule of thumb: if you can write it as a function and it solves the problem, write it as a function. If you find yourself needing variable arguments, compile time code generation, or syntax transformation  that is when macros earn their complexity.
 
---
 
## Summary
 
| Feature | What It Solves | When to Reach For It |
|---|---|---|
| Unsafe Rust | Operations the compiler cannot verify | FFI, custom allocators, hardware access, safe abstraction internals |
| Associated Types | 1:1 type relationships in traits | Implementing `Iterator`, `Display`, `Index` |
| Operator Overloading | Natural syntax for custom types | Types that represent values — vectors, matrices, units |
| Fully Qualified Syntax | Ambiguous method names from multiple traits | Two traits with the same method name on one type |
| Supertraits | Trait dependencies | When one trait's behavior requires another |
| Newtype Pattern | Orphan rule workaround | Implementing external traits on external types |
| Type Aliases | Reducing repetition in complex types | Long generics that appear in many places |
| Never Type `!` | Expressing divergence | Functions that panic, loop forever, or exit |
| DSTs + `?Sized` | Generics over unsized types | Accepting `str`, `[T]`, `dyn Trait` generically |
| Function Pointers | Passing named functions as values | Callbacks without closures, FFI, no-std |
| `Box<dyn Fn>` | Returning closures from functions | When the returned closure depends on a condition |
| `impl Fn` in return | Zero-cost single-closure return | When you always return the same closure type |
| Declarative Macros | Variadic code generation | `vec!`-style initialization, repetitive code patterns |
| Procedural Macros | Full token stream transformation | `#[derive(Trait)]`, routing, SQL validation |
 
These five areas — unsafe, advanced traits, advanced types, advanced functions, and macros — are the foundation that the standard library is built on. The `Vec<T>` you use every day is built on raw pointers and unsafe internals. The `Iterator` trait works because of associated types. The `+` operator on numbers uses operator overloading. The `#[derive(Debug)]` you add to every struct is a procedural macro. Understanding this chapter means you now understand not just how to use Rust, but how Rust itself is built.
 
In the next part, we wrap everything up with the final project.
 