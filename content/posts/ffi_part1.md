+++
title = "Foreign Function Interface (FFI) in Rust, Part 1: Unsafe Foundations"
description = "Learn why Rust uses unsafe code at language boundaries, what unsafe operations allow, and the pointer, NonNull, and memory layout basics you need before writing FFI code."
weight = 1
date = 2026-07-17

[taxonomies]
categories = ["Rust"]
+++

I find **Foreign Function Interface (FFI)** quite interesting and important, especially when a project needs to communicate with code written in another language, such as C code.

Imagine you're working with an existing C project and want to migrate it to Rust. Or maybe you're starting a new project in Rust but need to keep some existing C code. In both cases, FFI lets the two languages work together without requiring you to rewrite everything at once.

But there's another reason I find FFI interesting: it's a good opportunity to understand how Rust and C handle memory differently. So before diving into FFI itself (that's Part 2), let's start with the foundations it's built on.

## _Unsafe_ Rust Foundations
{% character() %}
Wait a second! Isn't Rust popular because of its safety guarantees?
{% end %}

Exactly! Rust's compiler enforces rules around references, mutation, and memory access to help prevent Undefined Behavior (UB)[^1]. These rules are a big part of what makes Rust reliable.

But there's a catch: those guarantees don't automatically extend to code written in another language.

Rust's borrow checker can't verify what happens inside a C function. Likewise, C doesn't enforce Rust's borrowing rules or guarantee that its code respects Rust's safety assumptions.

So, what happens when Rust needs to work with code that the compiler can't fully verify?

{% character(position="left") %}
That's where **Unsafe Rust** comes in!
{% end %}

### The `unsafe` Keyword
Let's look at a concrete example.

Suppose a C library gives Rust a pointer to some memory. How does Rust know that the pointer still refers to valid, live memory? It can't determine that just by looking at the pointer.

This is one of the situations where `unsafe` becomes necessary.

The `unsafe` keyword tells the compiler that you're taking responsibility for operations whose safety it can't fully verify. **It doesn't mean the code is automatically safe, and it doesn't turn off Rust's other compiler checks.**

Instead, `unsafe` allows a small set of operations that require additional guarantees from the programmer[^4]:
- Dereferencing a raw pointer
- Calling an `unsafe` function
- Reading or writing a `static mut` or an `extern` static
- Accessing a field of a `union` (other than assigning to it)
- Implementing an `unsafe` trait like `Send` or `Sync` (written as `unsafe impl`)
- Declaring an `extern` block: since the 2024 edition, it must be written as `unsafe extern "C" { ... }`
- Applying an unsafe attribute, such as `#[unsafe(no_mangle)]`

The last two are the ones you'll meet the moment you start writing FFI code, so keep them in mind for Part 2.

**Anything else inside an `unsafe` block is checked as usual by the compiler.**

This distinction matters. `unsafe` doesn't remove the need to reason about memory safety; it marks the places where the compiler needs you to uphold guarantees it can't establish on its own.

### Safety Contracts
So, how do we make sure an unsafe operation is actually sound?

We start by identifying its _safety contract_: the conditions that must hold for the operation to be valid.

Rust makes us think about these conditions from two perspectives:
- **The caller's responsibility:** An `unsafe fn` can require callers to uphold specific conditions.
- **The implementer's responsibility:** An `unsafe` block needs a justification for why the operation's safety requirements are satisfied.

Let's see how this works in practice:

```rust, linenos
/// Loads a 64-bit integer from a raw memory address.
///
/// # Safety
///
/// The caller must guarantee that the pointer is aligned, non-null,
/// and points to a fully initialized `i64` that will not be altered 
/// during the execution of this function.
pub unsafe fn load_i64_safely(location: *const i64) -> i64 {
    // SAFETY: We trust the caller's guarantee that `location` 
    // points to valid memory.
    unsafe { *location }
}
```
The function uses an unsafe operation to dereference the raw pointer. The compiler can't prove that the pointer is valid, so the function's safety contract places that responsibility on the caller.

Notice the `# Safety` section in the documentation. It spells out the conditions that must hold before the function can be called soundly.

This is more than documentation for the sake of documentation. It gives callers a concrete set of requirements to check and gives reviewers a way to reason about the code.

Now let's call the function:
```rust, linenos
let data_point: i64 = 42;
    
// SAFETY: `&data_point` produces a non-null, aligned and initialized reference
// that is safely coerced to a raw pointer, satisfying all safety requirements.
let fetched_value = unsafe { load_i64_safely(&data_point) };
```
Here, `&data_point` is coerced into a raw pointer to `i64`. We know that `data_point` is initialized, properly aligned, and alive for the duration of the call. We also know that it isn't modified during that call.

Those facts justify the unsafe operation under the function's documented contract.

The `// SAFETY:` comment explains why the code satisfies the relevant requirements. It's a useful habit because it forces us to make our assumptions explicit rather than simply trusting that the code will work.


## References and Raw Pointers
In C, pointers come with responsibilities. The programmer has to keep track of whether a pointer is null, who owns the memory, and how long that memory stays alive. These rules usually live in the programmer's head, API conventions, and sometimes the documentation.

Rust takes a different approach and implements this mental model directly. Its reference types and borrow checker encode many of these ownership and lifetime rules into the type system.

{% character() %}
It sounds great! So, what does a reference (`&T` or `&mut T`) guarantee for us?
{% end %}

A Rust reference (`&T` or `&mut T`) comes with several important guarantees:
- **Non-null:** Never points to null memory.
- **Aligned:** The memory address satisfies the alignment requirements of `T`.
- **Valid:** Points to live, fully initialized data.
- **Lifetime-bound:** It cannot outlive the data it refers to. The borrow checker enforces this relationship in safe Rust.
- **Aliasing-controlled:** Either many readers (`&T`) or **one** writer (`&mut T`), **never** both.

{% note(clickable=true, hidden=true, header='The compiler optimizes based on these guarantees.') %}
These guarantees don't just protect your program; they also give the compiler opportunities to optimize your code.

For example, because a shared reference (`&T`) promises that its referenced data won't be mutated, the compiler can make assumptions about repeated reads. Similarly, an exclusive mutable reference (`&mut T`) gives the compiler stronger guarantees about conflicting accesses.

These assumptions can enable optimizations such as reordering certain operations, or loading a value into a CPU register once and operating on it there, instead of repeatedly loading it from memory (RAM or cache). A `CPU register` is a tiny, ultra-fast storage location built directly into the CPU.

However, this doesn't mean the compiler always loads a value into a CPU register or that a particular optimization is guaranteed. The generated machine code depends on the surrounding code and optimization decisions.

Violating the requirements for creating or using a reference can result in UB.
{% end %}

### Raw Pointers
Rust also has raw pointers, `*const T` and `*mut T`, which are similar to C's `const T *` and `T *`, respectively. Unlike Rust references, **raw pointers don't carry guarantees about nullability, alignment, validity, or the lifetime of the memory they point to**.

{% character() %}
Wait! Why does Rust even have them if they don't come with these guarantees?
{% end %}

{% character(position="left") %}
Because C hands us raw addresses, and Rust needs a type that can hold them without making promises it can't keep!
{% end %}

**Safe creation, unsafe dereference:** Creating or copying a raw pointer (or a null pointer) is completely safe. Dereferencing a raw pointer is an `unsafe` operation because you must guarantee that the pointer is valid for the access.
```rust, linenos
let mut score = 100_u64;

// Creating raw pointers is entirely safe in Rust.
let ptr: *mut u64 = &mut score;
let empty_ptr: *const u64 = std::ptr::null();

assert!(!ptr.is_null());
assert!(empty_ptr.is_null());

// SAFETY: `ptr` comes directly from a valid mutable reference. 
// It is properly aligned, non-null, and no other code
// touches `score` during this operation.
unsafe {
    *ptr += 25;
    assert_eq!(ptr.read(), 125);
}
```
Notice that we don't need an `unsafe` block to create either pointer or check whether it's null. We only need `unsafe` when we actually access the memory through the pointer (`ptr`).

Here is a quick summary of the core raw pointer methods:
- **is_null():** Checks whether the pointer is null (safe).
- **cast():** Changes the type the pointer points to (safe).
- **add():** Performs pointer arithmetic (**unsafe**, because the resulting pointer must stay within the bounds of the original allocation). `wrapping_add()` is the safe alternative, but the resulting pointer may not be valid to dereference.
- **read() / write(value):** Copy a value out of or into memory without creating a Rust reference, so the reference aliasing rules don't come into play (**unsafe**). The pointer must still be valid for the access and properly aligned, and `read()` requires it to point to an initialized value.

### Mutability Origin Rules
In C, mutability is mostly a type convention; you can cast away `const` and write to memory as long as the underlying allocation isn't read-only.

Rust is fundamentally stricter: **whether you are allowed to write through a pointer depends on its lineage (how it was created), not just its type.**

If a raw mutable pointer (`*mut T`) is derived from a shared reference (`&T`), you **must never write through it**. Doing so causes UB, because a `&T` promises the compiler that the data is immutable.

The compiler can catch obvious cases. In fact, **the following snippet doesn't compile**; it's rejected by the deny-by-default `invalid_reference_casting` lint[^5]:

```rust, linenos
let original = 11_i32;
    
// A shared (immutable) reference
let shared_ref: &i32 = &original;

// Casting it to a mutable raw pointer doesn't magically make it safe to write to
let bad_ptr = shared_ref as *const i32 as *mut i32;

// UB! Writing here violates the contract of the original &i32 reference.
unsafe { *bad_ptr = 99; }
```
```
error: assigning to `&T` is undefined behavior, consider using an `UnsafeCell`
```

While the borrow checker tracks safe references (`&` and `&mut`), it **cannot track raw pointers**. This means you can trick the compiler into letting you mutate data while a shared reference is active, which is still UB, even if the code compiles and runs:
```rust, linenos
let mut number = 11_i32;
    
// Create a raw mutable pointer from a mutable reference
let raw_writer = &mut number as *mut i32;

// A normal shared reference to the same data
let observer = &number; 

// SAFETY VIOLATION: The borrow checker has no idea `raw_writer` 
// and `observer` point to the same memory because it only tracks references.
// Writing here while a shared reference is alive is UB!
unsafe {
    *raw_writer = 50;
}

// This might print 50, but the program is technically invalid. 
println!("{observer}");
```

{% note(clickable=true, hidden=true, header='Try running this through Miri!') %}
[**Miri**](https://github.com/rust-lang/miri) is an interpreter for Rust's Mid-level Intermediate Representation (MIR). It can detect several classes of UB in Rust programs, including invalid pointer accesses, use-after-free, and certain aliasing violations.

To try it, install it with `rustup +nightly component add miri` and run your program with `cargo +nightly miri run`. It will report the write through `raw_writer` as UB.
{% end %}

### Turning Raw Pointers Into References
When working with C libraries, you'll receive raw pointers that you may want to use in ordinary Rust code.

It would be nice to convert those pointers into Rust references (`&T` or `&mut T`) so we can benefit from Rust's reference-based safety guarantees. Rust provides two common ways to do this:

- **Direct dereferencing:** `&*ptr` or `&mut *ptr`. (**Warning:** If the pointer is null, this is UB. Debug builds may catch it with a panic[^6], but you must not count on it.)

- **Built-in methods:** `ptr.as_ref()` and `ptr.as_mut()`. These return `Option<&T>` / `Option<&mut T>`, so a null pointer becomes `None` instead of UB. They are still `unsafe` to call, though.

**Why is this unsafe?**

The compiler **cannot check** if a raw pointer meets Rust's reference guarantees. Furthermore, the compiler has no idea how long the C code will keep that memory alive.

Because of this, **you** must guarantee several things before creating a reference:

- **Validity:** The pointer must be non-null, properly aligned, and point to a valid value of the expected Rust type.
- **Lifetime:** The memory must remain alive for the entire lifetime of the resulting reference.
- **Aliasing:** The memory must not be accessed in a way that violates the reference's aliasing requirements.

Let's put this into a small example:
```rust, linenos
#[repr(C)] // We'll cover `repr(C)` at the end of this post.
pub struct AppStatus {
    pub is_running: bool,
    pub error_code: u32,
}

/// Converts a raw pointer from an external C runtime into a safe Rust reference.
///
/// # Safety
///
/// The caller must ensure that `raw_ptr` is either null or points to a 
/// properly aligned, fully initialized `AppStatus` instance that remains 
/// alive and unmodified for the entire duration of lifetime `'a`.
pub unsafe fn status_from_ffi<'a>(raw_ptr: *const AppStatus) -> Option<&'a AppStatus> {
    // SAFETY: We rely entirely on the caller's contract to guarantee 
    // that the pointer is valid and that its memory won't be freed early.
    unsafe { raw_ptr.as_ref() }
}
```
The function has an explicit lifetime `'a`. This lifetime tells Rust how long the returned reference is valid, but it doesn't extend the lifetime of the underlying C allocation.

Also don't let the `Option` type fool you! It only handles the null-pointer case. It doesn't validate whether **a non-null pointer refers to live, aligned, initialized memory**.

You still need to uphold all the safety requirements of creating the reference.

## NonNull and the Null-Pointer Optimization
In Rust, a normal raw pointer (`*const u8` or `*mut u8`) is designed to mirror C-style pointers. Because of this, it has **no safety guarantees or validity invariants:** it can point to any arbitrary memory address, including address **0x0 (NULL)**.

{% note(clickable=true, hidden=true, header='Why is `0x0` important?') %}
A `null pointer` is a special value indicating that a pointer does not currently point to **any valid memory location**. In Rust, this is a rule of the language itself: a null pointer is never valid to dereference, no matter what the hardware or operating system would do.

(As a side note, numerically `0x0` is a valid number in the CPU's address space, and operating systems typically leave that address unmapped so that accidental null accesses crash instead of silently reading data.)

For certain types that can never be null, such as references, `Box<T>`, and `NonNull<T>`, Rust takes advantage of this and uses the null value to represent `Option::None`. We'll see how below.
{% end %}

In C, a pointer may or may not be null. It depends on the contract of the function or API you're using.

{% character() %}
Wouldn't it be nice to represent this in Rust's type system?
{% end %}

Yes, you can! Rust provides `NonNull<T>`[^3] to encode that contractual knowledge directly into the type system.

`NonNull<T>` is a `*mut T` with one ironclad guarantee: **it's never null**. (It also differs from `*mut T` by being _covariant_ over `T`, which matters when building your own data structures.) However, it does **not guarantee that the memory it points to is valid, initialized, or still alive** (e.g., `NonNull::dangling()`).

There are several ways to create a `NonNull<T>`:
- `NonNull::new(ptr)`: returns an `Option<NonNull<T>>`, handling potential nulls gracefully.
- `NonNull::from(reference)`: converts a `&T` or `&mut T`. This is safe, since references can never be null.
- `NonNull::new_unchecked(ptr)`: skips the null check entirely. It's an `unsafe` function, and passing null is UB.
- `NonNull::dangling()`: provides an aligned, non-null pointer, often used for empty collections. It must not be used as a "not yet initialized" marker, since its address may coincide with a real value.

Some useful methods: `as_ptr()` gives you a raw pointer back (`*mut T`) for C interop, and `as_ref()` / `as_mut()` turn it into references. The latter two are `unsafe` for the same reasons as with raw pointers: non-null doesn't mean aligned, alive, or valid.

Let's see a small example:

```rust, linenos
use std::ptr::NonNull;

let mut a = 1_u32;

// Create a NonNull pointer from a mutable reference.
let non_null_ptr: NonNull<u32> = NonNull::from(&mut a);

// SAFETY: The pointer originates from `&mut a`.
// `a` remains alive, and no conflicting access occurs.
unsafe {
    *non_null_ptr.as_ptr() += 1;
}

assert_eq!(a, 2);

// A null pointer cannot become NonNull.
let from_c: *mut u32 = std::ptr::null_mut();
assert!(NonNull::new(from_c).is_none());
```

Notice that you can create a `NonNull<T>` without using an `unsafe` block when you start with a valid reference.

However, `NonNull<T>` doesn't guarantee that the pointed-to memory stays alive. It only guarantees that the pointer itself isn't null.

### Null-Pointer Optimization (NPO)
Now comes the interesting part!

Because `NonNull<T>` forbids the all-zeros bit pattern (`0`), that forbidden value creates what Rust calls a **niche**. 

The compiler takes advantage of this via **Null-Pointer Optimization (NPO).**

In a computer's memory, every byte has an address (written in hexadecimal like `0x00000000` or `0x0`). Address `0x0` is traditionally reserved as `NULL`, meaning **pointing to nothing**. For example:
```rust
// An illustrative diagram, not a guaranteed memory representation.
+-------------------+---------------------------------------------------+
| Address           | What lives there                                  |
+-------------------+---------------------------------------------------+
| 0x00000000        | NULL (Address Zero) - Forbidden for &T / NonNull  |
| 0x00000004        | Valid data (e.g., Integer 42)                     |
| 0x00000008        | Valid data (e.g., A struct or string)             |
+-------------------+---------------------------------------------------+
```
Rust guarantees that a reference `&T` never points to `0x0`. It must always point to valid data.

A raw pointer `*const T`, on the other hand, can legally hold `0x0` (NULL) or any other address, just like in C. This means Rust can't use `0x0` to mean "empty" (`None`). If you wrap it in an `Option<*const T>`, the compiler has to store an extra tag to track whether it's `Some` or `None`, which doubles its size on a typical 64-bit platform:
```rust
Option<*const T>  ──►  Total Size: 16 Bytes (Wastes Space!)
+-----------------------------------+-----------------------------------+
|       Pointer Address             |   Tag (1 Byte) + Padding (7 Bytes)|
|          (8 Bytes)                |                                   |
| (Can be 0x0, or any valid address)|     (Tracks if it's Some/None)    |
+-----------------------------------+-----------------------------------+
```

{% note(clickable=true, hidden=true, header='Why 16 bytes and not 9?') %}
The tag itself only needs 1 byte, which would give an awkward 9-byte value. But a type's size must always be a multiple of its alignment, so that every element of an array like `[Option<*const T>; N]` stays properly aligned. The pointer has an alignment of 8 bytes, so the compiler adds 7 bytes of padding and the total becomes 16 bytes.
{% end %}

That is where **Null-Pointer Optimization** starts shining! `NonNull<T>` tells Rust: "I promise this pointer will **never** be `0x0`."

Because `0x0` is now free (it's strictly forbidden as a real value), the compiler claims that empty `0x0` slot and uses it to represent `None`. The **extra tag** is deleted, shrinking the box back down to a single pointer width:

```rust
Option<NonNull<T>>  ──►  Total Size: 8 Bytes (Compact & Optimized!)
+-----------------------------------------------------------------------+
|                       Pointer Address (8 Bytes)                       |
|                                                                       |
|   • If value is 0x00000000 ───────► Rust treats it as NONE            |
|   • If value is any other address ─► Rust treats it as SOME (Data)    |
+-----------------------------------------------------------------------+
```
As a result, `Option<NonNull<T>>` **takes up the exact same space as a single pointer**[^7].

{% note(clickable=true, hidden=true, header='What’s guaranteed?') %}
The compiler has freedom over how it structures an `Option`, relying on unused bit patterns (niches) to optimize memory. For some specific types, however, the standard library guarantees that `Option<T>` has the exact same size, alignment, and function call ABI as `T`. These types are: `&T` and `&mut T`, `Box<T>`, `NonNull<T>`, `NonZero` integers (`NonZero<u32>`, etc.), function pointers (including `extern "C" fn`), and `#[repr(transparent)]` structs wrapping one of these.

For function pointers and `NonZero` integers, `None` is always represented as all zeros. For `&T`, `&mut T`, `Box<T>`, and `NonNull<T>`, this holds when `T` is `Sized` (its size is known at compile time).

This guarantee is what allows you to use these types safely in FFI signatures, for example `Option<NonNull<T>>` in place of a nullable C pointer.
{% end %}


## Memory Layout: Size, Alignment, and `repr`
Let's talk about one more topic before diving into FFI: how C and Rust store data in memory.

**Size** is the total number of bytes a type occupies in memory.

**Alignment** is a requirement that a value's memory address must be a multiple of its alignment. For example, a 4-byte `i32` with an alignment of 4 must start at an address divisible by 4.

Every type has a `size` and an `alignment` and both are known at compile time:
```rust, linenos
use std::mem::{align_of, size_of};

assert_eq!((size_of::<i8>(), align_of::<i8>()), (1, 1));
assert_eq!((size_of::<i32>(), align_of::<i32>()), (4, 4));
assert_eq!((size_of::<i64>(), align_of::<i64>()), (8, 8));
assert_eq!((size_of::<[i64; 2]>(), align_of::<[i64; 2]>()), (16, 8));
```
These values are typical for common platforms, but exact sizes and alignments can depend on the target architecture and ABI. For example, `align_of::<i64>()` is 4 on 32-bit x86 Linux.

Accessing a value through a misaligned pointer is UB in Rust. Depending on the architecture, misaligned memory accesses might be handled transparently, incur a performance penalty, or cause a hardware fault.

**std::alloc::Layout** is the standard library type that bundles both size and alignment together:

```rust, linenos
use std::alloc::Layout;

let small = Layout::new::<i32>();
assert_eq!((small.size(), small.align()), (4, 4));

let big = Layout::array::<i32>(4)
    .expect("array layout is too large");

assert_eq!((big.size(), big.align()), (16, 4));
```
Why does `Layout::array` return a `Result`? Because calculating the total size of a large array can overflow the supported size range.

### Size and Alignment of a Struct
C and Rust don't necessarily arrange struct fields in the same way.

Let's first look at how C lays out a struct:
```c, linenos
#include <stdbool.h>
#include <stdint.h>

struct SensorRecord {
  bool active;        // Size: 1 byte, Alignment: 1
  uint32_t reading;   // Size: 4 bytes, Alignment: 4
  uint8_t category;   // Size: 1 byte, Alignment: 1
};

// Assuming the target ABI uses these sizes and alignments,
// the memory layout looks like this:
offset  0          1               4             8      9              12
        ├ active ──┼─── padding ───┼── reading ──┼ cat ─┼── padding ────┤
```
The fields appear in the exact order they're declared: `active`, `reading`, and `category`.

Each field starts at an offset matching its alignment, and the compiler inserts **padding** (unused bytes) between fields to keep them properly aligned.

The compiler also adds padding at the end of the struct so its total size is a multiple of its alignment. In this example, the largest field alignment is 4 bytes, so the struct occupies 12 bytes.

Now, let's look at Rust.

Unlike C, Rust doesn't guarantee that fields will remain in their declaration order. The compiler is free to choose a different layout, including reordering fields when it can improve memory usage.

For example, a possible layout for our struct is:
```rust, linenos
struct SensorRecord {
    active: bool,   // Size: 1 byte, Alignment: 1
    reading: u32,   // Size: 4 bytes, Alignment: 4
    category: u8,   // Size: 1 byte, Alignment: 1
}

// This is an illustrative layout, not a guarantee.
offset  0             4      5      6                8
        ├─── reading ──┼ active ┼ cat ─┼── padding ──┤
```
Here, the compiler may place the field with the largest alignment requirement (`u32`) first, followed by the smaller fields. This arrangement reduces the amount of padding, shrinking the struct from 12 to 8 bytes.

While great for pure Rust code, this breaks instantly when interfacing with C.

{% character() %}
So if C receives a pointer to this Rust struct and expects `active` at offset `0`, but Rust has placed `reading` there instead, C will read the wrong bytes as `active`?

How do we make sure both languages agree on the struct's memory layout?
{% end %}


### The Need for `repr`
As you've seen, Rust's default struct representation doesn't guarantee field ordering, so C code reading the same memory could interpret the fields incorrectly. To solve this, Rust provides the `repr` attribute, which lets us control how types are represented in memory. You need it for:

- **Interoperability:** Sharing structs with C libraries, system APIs, or other components that expect a specific memory layout, where the Rust type must match it exactly.
- **Layout control:** When you need predictable field ordering and alignment according to the target C ABI.

There are several `repr` options, but `#[repr(C)]` is the one you'll use most often when working with C FFI, and it can be used with `struct`, `enum`, and `union` definitions.

Let's see it in action:
```rust, linenos
#[repr(C)]
struct SensorRecord {
    active: bool,   // Size: 1 byte, Alignment: 1
    reading: u32,   // Size: 4 bytes, Alignment: 4
    category: u8,   // Size: 1 byte, Alignment: 1
}

// Assuming the same sizes, alignments, and ABI rules
// as our earlier C example, the layout is:
offset  0          1               4             8      9              12
        ├ active ──┼─── padding ───┼── reading ──┼ cat ─┼── padding ────┤
```
Thanks to `#[repr(C)]`, Rust preserves the field declaration order and applies the target platform's C struct layout rules. In the example, that gives you a total size of 12 bytes.


## Conclusion
We covered quite a lot of ground before writing a single line of FFI code:

- **`unsafe` means responsibility, not permission:** it unlocks a small set of operations, and safety contracts (`# Safety` docs and `// SAFETY:` comments) make the required guarantees explicit.
- **References vs raw pointers:** references carry guarantees the compiler can check; raw pointers carry none, and whether you can write through them depends on how they were created.
- **`NonNull<T>` and NPO:** `NonNull<T>` encodes "never null" in the type system, and `Option<NonNull<T>>` is a nullable pointer with the same size as a raw pointer.
- **`#[repr(C)]`:** it makes Rust lay out a type the way C expects.

In Part 2, we'll put all of these pieces together and dive into FFI itself: declaring `unsafe extern "C"` blocks, calling C functions from Rust, and exposing Rust functions to C.

Lastly, thanks a lot to Mainmatter's C to Rust Migration Book[^2], which helped me a lot to grasp these concepts.

## Resources
[^1]: [Undefined Behavior](https://doc.rust-lang.org/reference/behavior-considered-undefined.html)
[^2]: [The C to Rust Migration Book by Mainmatter](https://mainmatter.com/c-to-rust-migration-book/course/welcome.html)
[^3]: [Rust standard library: NonNull](https://doc.rust-lang.org/core/ptr/struct.NonNull.html)
[^4]: [The Rust Reference: Unsafety](https://doc.rust-lang.org/reference/unsafety.html)
[^5]: [rustc lint: invalid_reference_casting](https://doc.rust-lang.org/rustc/lints/listing/deny-by-default.html#invalid-reference-casting)
[^6]: [Announcing Rust 1.86.0: debug assertions for null pointers](https://blog.rust-lang.org/2025/04/03/Rust-1.86.0/)
[^7]: [Rust standard library: Option representation](https://doc.rust-lang.org/std/option/index.html#representation)
