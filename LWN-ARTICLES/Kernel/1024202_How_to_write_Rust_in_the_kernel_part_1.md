---
title: "How to write Rust in the kernel: part 1"
url: https://lwn.net/Articles/1024202/
date: "June 20, 2025"
category: "Development tools-Rust"
author: "By Daroc Alden June 20, 2025 Rust in the kernel"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Daroc Alden**  
June 20, 2025

* * *

[Rust in the kernel](<https://lwn.net/Articles/1024941>)

The Linux kernel is seeing a steady accumulation of Rust code. As it becomes more prevalent, maintainers may want to know how to read, review, and test the Rust code that relates to their areas of expertise. Just as kernel C code is different from user-space C code, so too is kernel Rust code somewhat different from user-space Rust code. That fact makes Rust's [ extensive documentation](<https://doc.rust-lang.org/stable/book/>) of less use than it otherwise would be, and means that potential contributors with user-space experience will need some additional instruction. This article is the first in a multi-part series aimed at helping existing kernel contributors become familiar with Rust, and helping existing Rust programmers become familiar with what the kernel does differently from the typical Rust project. 

In order to lay the groundwork for the rest of the articles in this series, this first article gives a high-level overview of installing and configuring Rust tooling, as well as an explanation of how Rust fits into the kernel's existing build system. Future articles will cover how Rust fits into the kernel's maintainership model, what goes into writing a driver in Rust, the design of the Rust interfaces to the rest of the kernel, and hints about specific things to look for when reviewing Rust code. 

#### Prerequisites

While support for Rust on GCC is [ catching up](<https://rust-gcc.github.io/2025/06/04/2025-05-monthly-report.html>), and the `rustc_codegen_gcc` code generation backend is now capable of compiling the Rust components of the kernel, the Rust for Linux project currently only supports building with plain `rustc`. Since `rustc` uses LLVM, the project also recommends building the kernel as a whole with Clang while working on Rust code (although mixing GCC on the C side and LLVM on the Rust side does work). The build also requires [ bindgen](<https://rust-lang.github.io/rust-bindgen/>) to build the C/Rust API bindings, and a copy of the Rust standard library so that it can be built with the flags that the kernel requires. Building the kernel in the recommended way therefore requires Clang, lld, LLVM, the Rust compiler, the source form of the Rust standard library, and bindgen, at a minimum. 

Many Linux distributions package sufficiently current versions of all of these; the [Rust quick start documentation](<https://docs.kernel.org/rust/quick-start.html>) gives distribution-specific installation instructions. The minimum version of `rustc` required is 1.78.0, released in May 2024. The Rust for Linux project has committed to not raising the minimum required version unnecessarily. According to Miguel Ojeda, the current informal plan is to stick with the version included in Debian stable, once that catches up with the current minimum (likely later this year). 

Developers working on Rust should probably also install [ Clippy](<https://doc.rust-lang.org/clippy/>) (Rust's linter), [ rustdoc](<https://doc.rust-lang.org/rustdoc/index.html>) (Rust's documentation building tool), and [ rust-analyzer](<https://rust-analyzer.github.io/>) (the Rust [ language server](<https://microsoft.github.io/language-server-protocol/>)), but these are not strictly required. The Rust for Linux project tries to keep the code free of linter warnings, so patches that introduce new warnings may be frowned upon by maintainers. Invoking `make rustavailable` in the root of the kernel source will check that the necessary tools have compatible versions installed. Indeed, all the commands discussed here should be run from the root of the repository. The `make rust-analyzer` command will set up configuration files for rust-analyzer that should allow it to work seamlessly with an editor that has language server support, such as Emacs or Vim. 

Rust code is controlled by two separate kernel configuration values. `CONFIG_RUST_IS_AVAILABLE` is automatically set when compatible tooling is available; `CONFIG_RUST` (available under "General Setup → Rust support") controls whether any Rust code is actually built, and depends on the first option. Unlike the vast majority of user-space Rust projects, the kernel does not use Cargo, Rust's package manager and build tool. Instead, the kernel's makefiles directly invoke the Rust compiler in the same way they would a C compiler. So adding an object to the correct make target is all that is needed to build a Rust module: 

```
obj-$(CONFIG_RUST) += object_name.o
```

The code directly enabled by `CONFIG_RUST` is largely the support code and bindings between C and Rust, and is therefore not a representative sample of what most Rust driver code actually looks like. Enabling the Rust sample code (under "Kernel hacking → Sample kernel code → Rust samples") may provide a more representative sample. 

#### Testing

Rust's testing and linting tools have also been integrated into the kernel's [ existing build system](<https://kernelnewbies.org/KernelBuild>). To run Clippy, add `CLIPPY=1` to the `make` invocation; this performs a special build of the kernel with debugging options enabled that make it unsuitable for production use, and so should be done with care. `make rustdoc` will build a local copy of the Rust documentation, which also checks for some documentation warnings, such as missing documentation comments or malformed intra-documentation links. The tests can be run with [ `kunit.py`](<https://docs.kernel.org/dev-tools/kunit/index.html>), the kernel's white-box unit-testing tool. The tool does need additional arguments to set the necessary configuration variables for a Rust build: 

```
./tools/testing/kunit/kunit.py run --make_options LLVM=1 \
            --kconfig_add CONFIG_RUST=y --arch=<host architecture>
```

Actually locating a failing test case could trip up people familiar with KUnit tests, though. Unlike the kernel's C code, which typically has KUnit tests written in separate files, Rust code tends to have tests in the same file as the code that it is testing. The convention is to use a separate [ Rust module](<https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html>) to keep the test code out of the main namespace (and enable conditional compilation, so it's not included in release kernels). This module is often (imaginatively) called "test", and must be annotated with the `#[kunit_tests(test_name)]` macro. That macro is implemented in [ `rust/macros/kunit.rs`](<https://elixir.bootlin.com/linux/v6.15.1/source/rust/macros/kunit.rs>); it looks through the annotated module for functions marked with `#[test]` and sets up the needed C declarations for KUnit to automatically recognize the test cases. 

Rust does have another kind of test that doesn't correspond directly to a C unit test, however. A "doctest" is a test embedded in the documentation of a function, typically showing how the function can be used. Because it is a real test, a doctest can be relied upon to remain current in a way that a mere example may not. Additionally, doctests are rendered as part of the automatically generated [ Rust API documentation](<https://rust.docs.kernel.org/kernel/>). Doctests run as part of the KUnit test suite as well, but must be specifically enabled (under "Kernel hacking → Rust hacking → Doctests for the `kernel` crate"). 

An example of a function with a doctest (lightly reformatted from the Rust [ string helper functions](<https://elixir.bootlin.com/linux/v6.15.1/source/rust/kernel/str.rs#L35>)) looks like this: 

```
/// Strip a prefix from `self`. Delegates to [`slice::strip_prefix`].
        ///
        /// # Examples
        ///
        /// ```
        /// # use kernel::b_str;
        /// assert_eq!(
        ///     Some(b_str!("bar")),
        ///     b_str!("foobar").strip_prefix(b_str!("foo"))
        /// );
        /// assert_eq!(
        ///     None,
        ///     b_str!("foobar").strip_prefix(b_str!("bar"))
        /// );
        /// assert_eq!(
        ///     Some(b_str!("foobar")),
        ///     b_str!("foobar").strip_prefix(b_str!(""))
        /// );
        /// assert_eq!(
        ///     Some(b_str!("")),
        ///     b_str!("foobar").strip_prefix(b_str!("foobar"))
        /// );
        /// ```
        pub fn strip_prefix(&self, pattern: impl AsRef<Self>) -> Option<&BStr> {
            self.deref()
                .strip_prefix(pattern.as_ref().deref())
                .map(Self::from_bytes)
        }
```

Normal [ comments](<https://doc.rust-lang.org/reference/comments.html>) in Rust code begin with `//`. Documentation comments, which are processed by various tools, start with `///` (to comment on the following item) or `//!` (to comment on the containing item). These are equivalent: 

```
/// Documentation
        struct Name {
            ...
        }
    
        struct Name {
            //! Documentation
            ...
        }
```

Documentation comments are analogous to the specially formatted `/**` comments used in the kernel's C code. In this doctest, the `assert_eq!()` macro (an example of the other kind of macro invocation in Rust) is used to compare the return value of the `.strip_prefix()` method to what it should be. 

#### Quick reference

```
# Check Rust tools are installed
    make rustavailable
    # Build kernel with Rust enabled
    # (After customizing .config)
    make LLVM=1
    # Run tests
    ./tools/testing/kunit/kunit.py \
      run \
      --make_options LLVM=1 \
      --kconfig_add CONFIG_RUST=y \
      --arch=<host architecture>
    # Run linter
    make LLVM=1 CLIPPY=1
    # Check documentation
    make rustdoc
    # Format code
    make rustfmt
```

Finally, Rust code can also include [ kernel selftests](<https://docs.kernel.org/dev-tools/kselftest.html>), the kernel's third way to write tests. These need to be configured on an individual basis, using the kernel-configuration snippets in the `tools/testing/selftests/rust` directory. Kselftests are intended to be run on a machine booted with the corresponding kernel, and can be run with `make TARGETS="rust" kselftest`. 

#### Formatting

Rust's syntax is complex. This has been one of several sticking points in adoption of the language, since people often feel that it makes the language difficult to read. That problem cannot wholly be solved with formatting tools, but they do help. Rust's canonical formatting tool is called `rustfmt`, and if it is installed, it can be run with `make rustfmt` to reformat all the Rust code in the kernel. 

#### Next

Building and testing Rust code is necessary, but not sufficient, to review Rust code. It may be enough to get one started experimenting with the existing Rust code in the kernel, however. Next up, we will will do an in-depth comparison between a simple driver module and its Rust equivalent, as an introduction to the kernel's Rust driver abstractions.
