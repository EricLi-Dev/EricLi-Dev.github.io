---
title: RustOS Part 1 - A Freestanding Rust Binary 
date: 2025-01-18T00:00:00+00:00 
categories: [RustOS, Barebones]
tags: [rust] 
---

# A Freestanding Rust Binary
To make it possible to run Rust code on the bare metal machine without an underlying Operating System (OS) we need to create a Rust executable that does not link the standard library.

## Introduction
To write an OS Kernel, we need code that does not depend on any OS features. This means that we can't use threads, files, heap memory, the network, random numbers, standard output, or any other OS abstractions or specific hardware.

This means that we can't use most of the [Rust standard library](https://doc.rust-lang.org/std/), but there are a lot of Rust features that are still usable (ex: iterators, closures, pattern matching, option and result, string formatting, and the ownership system).

In order to create an OS Kernel in Rust, we need to create an executable that can be run without any underlying operating system. Such an executable is often called a **"freestanding" executable**.

## Initial Rust Crate
By default, all Rust crates link the [standard library](https://doc.rust-lang.org/std/), which depends on the OS for features like threads, files, or networking. Since our plan is to write an OS, we can't use any OS-dependent libraries so we have to disable the automatic inclusion of the standard lib. with the [`no_std` attribute](https://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/first-edition/using-rust-without-the-standard-library.html).

We start by creating a new cargo application:\
`cargo new rust_os --bin`

The `--bin` flag specifies that we want to create an executable binary (in contrast to a library). When we run the command, cargo creates the below directory structure:
```
rust_os
├── Cargo.toml
└── src
    └── main.rs
```
The `Cargo.toml` contains the crate configuration (ex. crate name, author, version number, and dependencies).

The `src/main.rs` contains the root module of our crate and our `main` function.

You can compile your crate through `cargo build` and then run the compiled `rust_os` binary in the `target/debug` subdirectory.

To combine both compilation and running the executable: `cargo run`.

## Disabling the Standard Library
### The `no_std` attribute
Right now, by default, our crate implicitly links the standard library. Let's disable this by adding the [`no_std` attribute](https://web.mit.edu/rust-lang_v1.25/arch/amd64_ubuntu1404/share/doc/rust/html/book/first-edition/using-rust-without-the-standard-library.html):
```rust
//main.rs

#![no_std] // disables the std. lib.

fn main() {
  println!("Hello, world!");
}
```
Try running `cargo run` now.

### Error 1: `println` macro
> Error: cannot find [macro `println`](https://doc.rust-lang.org/std/macro.println.html) in this scope.
{: .prompt-danger}

The reason for this is that the `println` macro is part of the standard library, which we no longer include. `println` is a special file descriptor that writes to the standard output provided by the OS, which means we can no longer print things.

Let's remove the printing and try again:
```rust
//main.rs

#![no_std] // disables the std. lib.

fn main() {}
```

### Error 2: The `panic_handler` Language Item
> Error: \`#[panic_handler]\` function required, but not found.
{: .prompt-danger}

Lanuage Items
: Are special functions, traits, or types that the compiler directly depends on to implement fundamental language features. Some examples are Add, Sub, Index for operator overloading, the `panic_handler` (as implemented above) attribute.

The `panic_handler` attribute helps define a function that the compiler should invoke when a [panic](https://doc.rust-lang.org/stable/book/ch09-01-unrecoverable-errors-with-panic.html) occurs. The standard library provides its own panic handler function, but in a `no_std` environment we must define one ourselves.
```rust

//main.rs

#![no_std] // disables the std. lib.
use core::panic::PanicInfo; // imports the PanicInfo struct.

fn main() {}

// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
  loop {}
}
```

The [`PanicInfo` parameter]() is a struct that provides information about the panic. It includes the file, line, where the panic happened, and the optional panic message.

The function should never return, so it is marked as a [diverging function](https://doc.rust-lang.org/1.30.0/book/first-edition/functions.html#diverging-functions) by returning the ["never" type](https://doc.rust-lang.org/nightly/std/primitive.never.) `!`.

Try running `cargo run` again.

### Error 3: The `eh_personality` Language Item
> Error: Unwinding panics are not supported without std.
{: .prompt-danger}

By default, Rust sets the panic strategy to stack unwinding. This error specifically points to the missing `eh_personality` attribute that helps define a function that is used for implementing [stack unwinding](https://www.bogotobogo.com/cplusplus/stackunwinding.php#:~:text=called%20unwinding.-,Stack%20Unwinding,-A%20call%20stack). Similar to the `panic_handler`, in a `no_std` environment we must define one ourselves. 

Stack Unwinding
: used to run the destructors of all live stack variables in case of a **panic**. This ensures that all used memory is freed and allows the parent thread to catch the panic and continue execution.

However, stack unwinding is a complicated process and requires some OS-specific libraries so we won't be using it here. Writing a custom `eh_personality` language item implementation is possible but should only be done as a last resort as language items are highly unstable implementation details.

We can avoid this error by setting the panic strategy to [abort on panic]() instead.

```toml
[package]
#...

# The profile used for `cargo build`
[profile.dev]
panic = "abort" # disable stack unwinding on panic

# The profile used for `cargo build --release`
[profile.release]
panic = "abort" # disable stack unwinding on panic

[dependencies]
#...
```
Try running `cargo run` again.

### Error 4: The `start` Language Item
> Error: using `fn main` requires the standard library.
{: .prompt-danger}

The `start` attribute defines the entry point to the program.

One might think the `main` function is the first function called when you run a program. However, most languages have a **runtime system**, which is responsible for things such as garbage collection (e.g. in Java) or software threads (e.g. goroutines in Go). This runtime needs to be called before `main`, since it needs to initialize itself.

Runtime System
: Acts as a bridge between the program's compiled code and the OS to handle tasks that the program cannot do on its own like memory management, thread scheduling, I/O operations, etc.

In a typical Rust binary that links the standard library, execution starts in a C runtime library called `crt0` ("C runtime zero"), which sets up the environment for a C application. This includes creating a stack and placing the arguments in the right registers. The C runtime then invokes the [entry point of the Rust runtime](https://github.com/rust-lang/rust/blob/bb4d1491466d8239a7a5fd68bd605e3276e97afb/src/libstd/rt.rs#L32-L73), which is marked by the `start` Language Item. Rust has a very minimal runtime, which takes care of some small things such as setting up stack overflow guards or printing a backtrace on panic. The runtime then finally calls the `main` function.

Since our freestanding executable is operating in a `no_std` environment, we do not have access to the C runtime, `crt0`, and Rust runtime, so we need to define our own entry point.

Implementing the `start` language item wouldn't help, since it would still require `crt0`. Instead, we need to overwrite the `crt0` entry point directly.

#### Overwriting the Entry Point
To tell the compiler we do not want to use the normal entry point chain, we add the `![no_main]` attribute.

```rust
// main.rs

#![no_std] // disables the std. lib.
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo; // imports the PanicInfo struct.

// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```
Notice that we remove the `main` function. The reason is that a `main` doesn't make sense without an underlying runtime that calls it, we are now overwriting the Operating System entry point with our own `_start` function:

```rust
// main.rs
...

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}
```
`#[no_mangle]` disables [name mangling](https://en.wikipedia.org/wiki/Name_mangling) to ensure that the Rust compiler really outputs a function with the name `_start`. Without this symbol, the compiler would generate some cryptic symbol to give every function a unique name. This attribute is required because we need to pass the name of the entry point function to the linker in the next step.

`extern "C"` is used as a function marker to tell the compiler that it should use the [C calling convention](https://en.wikipedia.org/wiki/Calling_convention) for this function. The function is named `_start` because this is the default entry point name for most systems.

`!` return type means that the function is diverging (i.e. not allowed to ever return). This is required because the entry point is not called by any function, but invoked directly by the operating system or bootloader. So instead of returning, the entry point should e.g. invoke the [`exit` system call](https://en.wikipedia.org/wiki/Exit_(system_call)). In our case, shutting down the machine could be appropriate, since there's nothing left to do if a freestanding binary returns. For now, we fulfill the requirement by looping endlessly.

Try running `cargo run` now.

### Error 5: Linker Errors
> Error: linking with `cc` failed
{: .prompt-danger}

Linker
: Combines the generated code into an executable.

Since the executable format differs between Linux, Windows, MacOS, each system has its own linker that throws a different error. The fundamental cause of the errors is the same: the default configuration of the linker assumes that our program depends on the C runtime, which it doesn't.

To solve this linker error we need to tell the linker that it should not include the C runtime. We can do this by either passing a certain set of *arguments to the linker* or by *building for a bare metal target*.

#### Building for a Bare Metal Target
By default, Rust tries to build an executable that is able to run in your current system environment. For example, if you're using Windows `x86_64`, Rust tries to build an `.exe` Windows executable that uses `x86_64` instructions. 

To describe different environments, Rust uses a string called target triple. You can see the target triple for your host system by running `rustc --version --verbose`.

My `host` triple is `x86_64-unknown-linux-gnu`, which includes the CPU architecture(`x86_64`), the vendor(`unknown`), and the operating system(`linux-gnu`).

By compiling for our host triple, the Rust compiler and the linker assume that there is an underlying operating system such as Linux or Windows that uses the C runtime by default, which causes the linker errors.So to avoid the linker errors, we can compile for a different environment with no underlying operating system.

An example of such a bare metal environment is the `thumbv7em-none-eabihf` target triple, which describes an embedded ARM system. The details are not too important, all that matters is that the target triple has no underlying operating system, which is indicated by the `none` within the triple. To be able to compile for this target, we need to add it in the rustup:
```bash 
rustup target add thumbv7em-none-eabihf
```

This downloads a copy of the standard (and core) library for the system. Now we can build our freestanding exectuable for this target:
```bash
cargo build --target thumbv7em-none-eabihf
```

By passing a `--target` argument we can [cross compile](https://stackoverflow.com/questions/897289/what-is-cross-compilation) our executable for a bare metal target system. Since the target system has no operating system, the linker does not try to link the C runtime and our build succeeds without any linker errors.

Cross Compile
: Refers to building a program on one platform (the **host**) that is intended to run on another platform (the **target**).

This is the approach that we will use for building our OS kernel. Instead of `thumbv7em-none-eabihf`, we will use a [custom target](https://doc.rust-lang.org/rustc/targets/custom.html) that describes an `x86_64` bare metal environment. The details will be explained in the next post.

Custom Target
: Allows you to define and use your own configuration for platforms that are not officially supported by Rust. A target specifies the platform for which a program is compiled against, defines the CPU architecture, Operating System, etc.

## Summary
A minimal freestanding Rust binary looks like this:

`src/main.rs`
```rust
#![no_std] // disables the std. lib.
#![no_main] // disable all Rust-level entry points

use core::panic::PanicInfo; // imports the PanicInfo struct.

// This function is called on panic.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

#[no_mangle] // don't mangle the name of this function
pub extern "C" fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}
```

`Cargo.toml`
```toml
[package]
name = "rust_os"
version = "0.1.0"
edition = "2021"

# The profile used for `cargo build`
[profile.dev]
panic = "abort" # disable stack unwinding on panic

# The profile used for `cargo build --release`
[profile.release]
panic = "abort" # disable stack unwinding on panic

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
[dependencies]
```

To build this binary, we need to compile for a bare metal target such as 
`thumbv7em-none-eabihf`:

```bash
cargo build --target thumbv7em-none-eabihf
```

This is just a minimal example of a freestanding Rust binary. This binary expects various things, for example, that a stack is initialized when the `_start` function is called. So for any real use of such a binary, more steps are required.
