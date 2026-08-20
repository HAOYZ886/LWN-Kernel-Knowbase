---
title: Improving bindgen for the kernel
url: https://lwn.net/Articles/992693/
date: "October 9, 2024"
category: "Development tools-bindgen"
author: "By Daroc Alden October 9, 2024 Kangrejos 2024"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Daroc Alden**  
October 9, 2024

* * *

[Kangrejos 2024](<https://lwn.net/Articles/990496/>)

[ Bindgen](<https://rust-lang.github.io/rust-bindgen/>) is a widely used tool that automatically generates Rust bindings from C headers. The [ Rust-for-Linux](<https://rust-for-linux.com/>) project uses it to create some of the bindings between Rust code and the rest of the kernel. John Baublitz presented at Kangrejos about the improvements that he has made to the tool in order to make the generated bindings easier to use, including improved support for macros, bitfields, and enums. 

Baublitz noted that there has been a wishlist of features to add to bindgen for the Rust-for-Linux project for some time. After he ran into some of the same problems in his own projects, he decided to tackle them. There are three main problems that he wants to address: macro expansion, accessing bitfields via raw pointers, and supporting better conversions for Rust enums. 

#### Macro expansion

There is no way that bindgen can usefully support the full richness of C macros. But there are a subset of macros that are useful to have represented in the generated Rust code: macros that are just used as a name for a constant value. Currently, bindgen specially recognizes simple macros and turns them into constants: 

```
#define NAME 3
        // becomes
        pub const NAME: u32 = 3;
```

However, it's relatively common for a macro to be defined in terms of other macros, which requires expanding the macro to determine its value. Since bindgen doesn't include a reimplementation of the C preprocessor, it can't handle these more complex macros. Baublitz gave the example of [ cryptsetup](<https://gitlab.com/cryptsetup/cryptsetup>), which added the [ `UINT32_C` macro](<https://en.cppreference.com/w/cpp/header/cstdint>) around some of its constants and broke the generated Rust bindings. 

He has come up with a way to make it work, however. With his changes, bindgen can now capture the name of the macro, create a temporary C file with a main function that returns the value of the macro, and then use Clang to compile it. Baublitz described this as ""a bit hacky"", but working. For now, the new code remains opt-in using the `--clang-macro-fallback` flag, for two reasons. First of all, even small changes to generated bindings can cause problems, such as by introducing duplicated names, so bindgen tries not to change the default behavior. Secondly, the approach does have performance implications, since it involves invoking Clang. 

The performance impact isn't that bad, however. Baublitz measured the time taken to evaluate the macros in a consolidated header file containing all of the constants defined in the kernel's headers, which was 3-5 seconds. His initial prototype was significantly worse, taking nearly 35 minutes. The majority of that time was spent doing I/O; switching to Clang's in-memory API made that much faster, but still too slow for practical work. His final design takes advantage of Clang's support for precompiled headers, by compiling the headers once, and then generating multiple C files in memory to evaluate the different constants. 

There is one complication to using precompiled headers. Clang actually only supports using one precompiled header per source file, and silently ignores any others passed. So, Baublitz generates a synthetic header that imports all of the others, and then pre-compiles that. Still, despite the problems, the new option was released in [ bindgen 0.70](<https://github.com/rust-lang/rust-bindgen/releases/tag/v0.70.0>) on August 16, and is available to users. In the future, Baublitz would like to add a Clang API that retains macro information when parsing, and use that directly, instead of maintaining this workaround. Miguel Ojeda confirmed that the two of them had spoken to a Clang maintainer, who had approved of that approach. For now, however, this solution works, and makes many more constants available between the two languages. 

#### Bitfield access

Since C does not have Rust's lifetime tracking, programmers often need to refer to structures shared between Rust and C using raw pointers instead of Rust's references. This poses a problem for bitfields. Rust doesn't have a native concept of bitfields, so when a C structure contains a bitfield, bindgen generates accessor functions to access the value correctly. The generated functions take a reference to the structure, since that is the idiomatic way to define methods for a type in Rust. This poses a problem for structures that need to be referred to with raw pointers. 

Baublitz addressed this problem by adding an additional set of unsafe helper functions to access bitfields using raw pointers. At the time of his talk, the Rust-for-Linux developers had reviewed his code and agreed that it would be helpful, but it still needed a review from the bindgen maintainer. 

Luckily that maintainer, Christian Poveda Ruiz, was also in attendance, and agreed to look at the pull request shortly. As of September 24, the new helpers have [ been merged](<https://github.com/rust-lang/rust-bindgen/commit/3c09db0b5646b3d229d5c5f5e13cf2b2373ae2d9>), and they should be available in the next release. 

#### Enum conversions

The last item Baublitz discussed was improving how bindgen represents enums. The problem in this case has to do with a mismatch between how C and Rust treat invalid enum variants. In C, enums are essentially named constants, and it is not undefined behavior assign a value to an enum variable that has not been defined for the enum type. In Rust, creating an enum with an invalid bit pattern, such as a nonexistent variant, is instant undefined behavior. Because of that, bindgen currently translates C enums to compile-time constants. 

It would be more convenient to translate them directly to Rust enums, since then the compiler could then perform exhaustiveness checking and so on. Baublitz's solution is to have two types: a raw type that is just an alias for the C enum's storage type (such as `unsigned integer`), and another type that is a normal Rust enum. Then bindgen can generate two sets of conversion functions: safe functions that check that the enum is valid and could return an error, and unsafe unconditional functions for when the programmer can guarantee that there won't be any invalid values. 

Changing the way enums are translated would be a breaking change, so Baublitz has added a command-line flag — `--rustified-enum` — that lets users select whether they want the old behavior, safe conversions, or unsafe conversions. There were some challenges to making this code work, he added. He needed to change how bindgen does its command-line parsing, and adapt some of the internals to handle both translated and untranslated types. 

The updated enum code is still in progress, however, because there are some questions that Baublitz wants feedback on. In particular, he would like to still generate constants for enum values, to make switching between the different enum translations as small a change as possible — but that could lead to problems with namespacing. Gary Guo suggested using associated constant items, but Baublitz explained that bindgen currently doesn't do that in other cases, so it wouldn't be consistent. Also, the constants would clash with the names of the actual variants. 

Alice Ryhl had further questions about how the new enum translation interacts with control-flow-integrity (CFI) protections. While there are many CFI techniques, she specifically referred to type-based CFI, where the compiler inserts checks that a call through a function pointer is only made to a function of a compatible type. This cuts down on the amount of unintended control flow an attacker can cause by overwriting function pointers. She was worried specifically about the case where, using the new translation, the Rust compiler sees a FFI function as taking a `c_int`, while the C side sees it as taking an enum type. These types might have compatible storage layouts, but they have different type names, which would generate different CFI tags. Baublitz was unfamiliar with the details of CFI, and after a short back and forth agreed with Ryhl's suggestion to add a wrapper type with the correct name. 

Benno Lossin wanted to take the opportunity to explain why the new enum translations would be helpful in the driver he is working on: currently, it has a lot of manual checks that could ideally be simplified by having the tooling do it. Poveda Ruiz clarified that he thinks Baublitz's style would be a sensible default, but that every time the bindgen project changes the defaults, things break and people complain. So while the new style may become an option, it will not be the default. 

In all, it seems like users of bindgen should have more options for correct, ergonomic translation of C interfaces — but that they must be aware to take advantage of them. Readers who use bindgen in their own projects might wish to keep an eye out for Baublitz's changes.
