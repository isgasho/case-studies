## Function epilogue

For the [`#[no_panic]`][no-panic] macro I needed the ability to have some piece
of code invoked during all *panicking* exit paths out of a function.

[no-panic]: https://github.com/dtolnay/no-panic

<br>

### First attempt

Having something execute on *all* exit paths is reasonably simple -- place a
guard object in a local variable and its `Drop` impl will run whether the
function body succeeds or panics. This may be a good approach for something like
instrumenting functions with tracing on entry and exit.

```rust
// Before
fn f(a: Arg1, b: Arg2) -> Ret {
    /* original function body */
}

// After; insert guard object
fn f(a: Arg1, b: Arg2) -> Ret {
    struct Guard;
    impl Drop for Guard {
        fn drop(&mut self) {
            /* do the thing */
        }
    }
    let _guard = Guard;

    /* original function body */
}
```

From here we can have the guard's `Drop` impl check
[`std::thread::panicking`][panicking] to determine whether the call is taking
place during a panicking exit path.

[panicking]: https://doc.rust-lang.org/std/thread/fn.panicking.html

```rust
impl Drop for Guard {
    fn drop(&mut self) {
        if std::thread::panicking() {
            /* do the thing */
        }
    }
}
```

Two things made this not suitable for my case:

- There is no equivalent in libcore, so this only works if my caller's crate is
  using the standard library.

- The code inside of `if std::thread::panicking() { ... }` gets linked whether
  or not a panic is possible. The implementation of the panicking check is based
  on reading a panic counter out of a thread\_local and cannot be optimized out.
  In the case of `#[no_panic]`, the whole macro is based on using the
  information of whether something gets linked to tell whether a panic is
  possible so I needed the linking to behave well.

<br>

### Second attempt

Let's evaluate the body of the function and then make the guard not get dropped
if the function produces a value as opposed to panicking.

```rust
fn f(a: Arg1, b: Arg2) -> Ret {
    struct Guard;
    impl Drop for Guard {
        fn drop(&mut self) {
            /* do the thing */
        }
    }
    let guard = Guard;

    let value = {
        /* original function body */
    };

    mem::forget(guard);
    value
}
```

If the original function panics, we don't make it to the `mem::forget` so the
guard object is dropped as part of dropping the stack frame of `f` during the
panic. If the original function body returns without panicking, we skip the
guard's drop prior to returning from `f`.

This is on the right track! It works with no\_std, and no longer relies on the
thread\_local inside of `std::thread::panicking` so it optimizes away extremely
reliably in functions that can never panic.

There is a problem around functions that contain a `return` expression. If the
original function body performs a `return`, that would now return from `f`
without running `mem::forget` on the guard object, so the thing that we want to
run only when panicking would incorrectly run.

<br>

### Third attempt

Let's consolidate all the non-panicking exit paths into one place via a function
call and make the guard not get dropped if the function call returns without
panicking.

```rust
fn f(a: Arg1, b: Arg2) -> Ret {
    struct Guard;
    impl Drop for Guard {
        fn drop(&mut self) {
            /* do the thing */
        }
    }
    let guard = Guard;

    fn original_f(a: Arg1, b: Arg2) -> Ret {
        /* original function body */
    }
    let value = original_f(a, b);

    mem::forget(guard);
    value
}
```

This is like the second attempt except that it works when the original function
body contains a `return` expression.

This is pretty good. It has the desired behavior and is compatible with most
function signatures.

<br>

### Fourth attempt

What do we do in this case?

```rust
fn f(&self, a: Arg1, b: Arg2) -> Ret {
    ...
}
```

The scheme from the third attempt of duplicating the function signature into an
internal `original_f` will not work because `&self` arguments can only occur in
members of an impl block, not in any other position that a function can be
defined.

```rust
struct S;

impl S {
    fn f(&self, a: Arg1, b: Arg2) -> Ret {
        ...
        let guard = Guard;

        fn original_f(&self, a: Arg1, b: Arg2) -> Ret {
            /* original function body */
        }
        let value = original_f(self, a, b);

        mem::forget(guard);
        value
    }
}
```

```console
error: unexpected `self` argument in function
 --> src/main.rs:8:24
  |
8 |         fn original_f(&self, a: Arg1, b: Arg2) -> Ret {
  |                        ^^^^ `self` is only valid as the first argument of an associated function
```

It doesn't work to try to generate `fn original_f(_self: &S, ...) -> Ret`
because the macro generating this will be an attribute macro placed on the
function -- it would only receive the function `f` as input not including the
impl block header, so the correct type for `self` can't be known.

```rust
impl ??? {
    fn f(&self, a: Arg1, b: Arg2) -> Ret {
        ...
        let guard = Guard;

        fn original_f(_self: &???, a: Arg1, b: Arg2) -> Ret {
            /* original function body */
        }
        let value = original_f(self, a, b);

        mem::forget(guard);
        value
    }
}
```

The argument type `_self: &Self` can't be used because a function like
`original_f` is its own self-contained item and does not have access to an outer
`Self` or type parameters.

Maybe we could ask the user to write our attribute macro on the impl block
rather than on functions but this would be confusing; a solution that does not
require this would be better.

It also doesn't work in general to place the `original_f` outside of `f`, as a
`#[doc(hidden)]` method next to `f`. This would work inside of an impl block
containing inherent methods, but not inside of a trait impl block containing
trait methods since those are limited to the set of methods required by the
trait.

```rust
impl ??? {
    fn original_f(&self, a: Arg1, b: Arg2) -> Ret {
        /* original function body */
    }

    fn f(&self, a: Arg1, b: Arg2) -> Ret {
        ...
        let guard = Guard;

        let value = Self::original_f(self, a, b);

        mem::forget(guard);
        value
    }
}
```

To finally give a viable fourth attempt, let's write `original_f` as a closure
instead because closures are not a self-contained item and *do* have access to
an outer `Self`.

```rust
fn f(&self, a: Arg1, b: Arg2) -> Ret {
    ...
    let guard = Guard;

    let original_f = |_self: &Self, a: Arg1, b: Arg2| -> Ret {
        /* original function body, with self replaced by _self */
    };
    let value = original_f(self, a, b);

    mem::forget(guard);
    value
}
```

Here we pass the function arguments along to a closure that has the same
signature as the outer function and captures nothing. Method receivers in the
form of `&self`, `&mut self`, and `self` would be passed as closure arguments
`_self: &Self`, `_self: &mut Self`, `_self: Self` respectively with the original
function body adjusted to refer to `_self` anywhere that it originally referred
to `self`. The leading underscore on `_self` is meaningful in that it suppresses
unused variable lints; Rust does not warn when a method accepts `self` but does
not refer to it, so we want to preserve that behavior in the generated closure.

This really seems like it should work. But...

<br>

### Fifth attempt

The borrow checker doesn't like it. In the case of a method signature that
borrows from `self`:

```rust
fn f(&self) -> &i32 {
    ...
    let guard = Guard;

    let original_f = |_self: &Self| -> &i32 {
        &_self.0
    };
    let value = original_f(self);

    mem::forget(guard);
    value
}
```

we get this interesting error:

```console
error[E0495]: cannot infer an appropriate lifetime for borrow expression due to conflicting requirements
  --> src/main.rs:17:13
   |
17 |             &_self.0
   |             ^^^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the body at 16:26...
  --> src/main.rs:16:26
   |
16 |           let original_f = |_self: &Self| -> &i32 {
   |  __________________________^
17 | |             &_self.0
18 | |         };
   | |_________^
note: ...so that reference does not outlive borrowed content
  --> src/main.rs:17:13
   |
17 |             &_self.0
   |             ^^^^^^^^
note: but, the lifetime must be valid for the anonymous lifetime #1 defined on the method body at 7:5...
  --> src/main.rs:7:5
   |
7  | /     fn f(&self) -> &i32 {
8  | |         struct Guard;
9  | |         impl Drop for Guard {
10 | |             fn drop(&mut self) {
...  |
22 | |         value
23 | |     }
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/main.rs:22:9
   |
22 |         value
   |         ^^^^^
```

I can't tell where this went wrong but casting the closure to a function pointer
with the right signature seems to fix it. This requires rustc 1.23+.

```rust
fn f(&self) -> &i32 {
    ...
    let guard = Guard;

    let original_f = |_self: &Self| -> &i32 {
        /* original function body, with self replaced by _self */
    } as fn(&Self) -> &i32;
    let value = original_f(self);

    mem::forget(guard);
    value
}
```

<br>

### Sixth attempt

Let's take a closer look at what is meant by "self replaced by \_self".

The simple way for a macro to accomplish this would be by traversing the entire
token stream representing the function body and substituting a `_self` token
anywhere that `self` occurs. This is correct as long as `self` always refers to
the method receiver... but sometimes it may not. Let's say the user has written:

```rust
fn f(&self) {
    struct UserGuard;
    impl Drop for UserGuard {
        fn drop(&mut self) {
            // notice the `self` on the previous line
            ...
        }
    }

    ...
}
```

The ability to place structs and impl blocks inside a function body was super
helpful to us so far because that's how we have been doing *our* Guard object.
But the user is free to do it too! In this snippet they have written a function
body that uses the token `self` in a way that does *not* refer to the `f`
method's receiver. If we naively replace every `self` in their function body
with `_self` as indicated in the fifth attempt, the result is invalid Rust
syntax:

```rust
fn f(&self) {
    struct Guard;
    impl Drop for Guard {
        fn drop(&mut self) {
            // guard generated by our macro
        }
    }
    let guard = Guard;

    let original_f = |_self: &Self| -> &i32 {
        struct UserGuard;
        impl Drop for UserGuard {
            fn drop(&mut _self) {
                // invalid Rust syntax on the previous line
                ...
            }
        }

        ...
    } as fn(&Self) -> &i32;
    let value = original_f(self);

    mem::forget(guard);
    value
}
```

```console
error: expected one of `:` or `@`, found `)`
  --> src/main.rs:19:31
   |
19 |             fn drop(&mut _self) {
   |                               ^ expected one of `:` or `@` here
```

So replacing *every* `self` is not right. The next simplest possibility would be
to parse the user's function body using Syn and write a [`VisitMut`] to perform
the replacement against the parsed syntax tree without traversing into nested
impl blocks.

[`VisitMut`]: https://docs.rs/syn/0.15/syn/visit_mut/index.html

That is more correct than replacing *every* `self` but it still isn't correct
because we can't know how to treat unexpanded macros. If the user's function
body contains a call to `somemacro!(self)`, there would be no way to tell
whether this expands to an expression like `vec![self]` in which we need to
replace, vs an impl block like `impl Drop for UserGuard` in which we want to not
replace.

I think there is no solution to this today in Rust, so we will need to keep it
as a limitation that sometimes our macro would generate invalid code, or else
solve what we are doing in a way that does not involve doing *any* token
replacement of `self`.

So that we don't need replacement, let's try having our generated closure
capture `self` from the outer method `f`'s receiver argument.

There are a lot of different ways to slice and dice this, but ultimately they
all fall apart for borrow checker reasons. Remember how we had to add a cast to
function pointer type in the fifth attempt? Well once the closure is capturing
things, it can no longer be cast to a function pointer. Using `impl FnOnce` or
`&mut dyn FnMut` here don't work either; as far as I can tell the correct type
for these closure's cannot be accurately described in Rust's type system.

```rust
fn f(&mut self) -> &mut i32 {
    ...
    let guard = Guard;

    let original_f: impl FnOnce() -> &mut i32 = move || {
        /* original function body: */
        &mut self.0
    };
    let value = original_f();

    mem::forget(guard);
    value
}
```

```console
error[E0106]: missing lifetime specifier
  --> src/main.rs:17:42
   |
17 |         let original_f: impl FnOnce() -> &mut i32 = move || {
   |                                          ^ help: consider giving it a 'static lifetime: `&'static`
   |
   = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
```

There isn't a way for the lifetime in the signature of a closure to unify with
the elided lifetime in `f`'s signature.

I tried a lot of variations in this direction but found it to be a dead end. I
would love to have someone bring to my attention a reliable solution that does
not involve replacing `self` tokens on a heuristic basis.

<br>

### Best known solution

The approach from the fifth attempt, with the `VisitMut` replacement approach
discussed under the sixth attempt. All together it ends up looking like this:

```rust
// Before
fn f(&self, a: Arg1, b: Arg2) -> Ret {
    /* original function body */
}

// After
fn f(&self, a: Arg1, b: Arg2) -> Ret {
    struct Guard;
    impl Drop for Guard {
        fn drop(&mut self) {
            /* do the thing */
        }
    }
    let guard = Guard;

    let original_f = |_self: &Self, a: Arg1, b: Arg2| -> Ret {
        /*
         * original function body, with self replaced by _self
         * except in nested impls
         */
    } as fn(&Self, Arg1, Arg2) -> Ret;

    let value = original_f(self, a, b);

    mem::forget(guard);
    value
}
```

This is part of the implementation used for the [`no-panic`][no-panic] crate.
