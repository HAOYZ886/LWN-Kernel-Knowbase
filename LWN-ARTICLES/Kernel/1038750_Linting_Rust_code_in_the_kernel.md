---
title: Linting Rust code in the kernel
url: https://lwn.net/Articles/1038750/
date: "September 30, 2025"
category: "Development tools-Rust"
author: "By Daroc Alden September 30, 2025 Kangrejos"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Daroc Alden**  
September 30, 2025

* * *

[Kangrejos](<https://lwn.net/Articles/1039334/>)

[ Klint](<https://github.com/Rust-for-Linux/klint?tab=readme-ov-file#klint>) is a Rust compiler extension [ developed by Gary Guo](<https://lwn.net/Articles/951550/>) to run some kernel-specific lint rules, which may also be useful for embedded system development. He spoke about his recent work on the project at [ Kangrejos 2025](<https://kangrejos.com/>). The next day, Alejandra González led a discussion about Rust's normal linter, [ Clippy](<https://doc.rust-lang.org/stable/clippy/index.html>). The two tools offer complementary approaches to analyzing Rust kernel code, although both need some additional direction and support from kernel developers to reach their full potential. 

#### Klint

Klint was started in 2021 to find places where Rust code in the kernel was ignoring allocation errors. That is no longer necessary — the [ Rust for Linux project](<https://rust-for-linux.com/>) ended up rewriting the allocation interfaces to make ignoring allocation failures more difficult — but klint still has a few other useful checks. Mainly, klint is used to check that code does not sleep while holding a spinlock. 

The last two years have mostly been maintenance, and not much new development, Guo said. But he did still have some updates to share. The simplest change is adding a shorthand for common klint annotations: 

```
#[klint::preempt_count(expect = 1..)]
        // can now be written
        #[klint::atomic_context_only]
```

Other improvements are tied to the way that klint works. Clippy mainly focuses on "local" lint rules: suggestions that usually only involve a handful of lines of code, often contained in a single function. In contrast, klint focuses on kernel-specific properties of the entire program. To manage that, klint hooks into the Rust compiler's unstable internal APIs to access the [ mid-level intermediate representation](<https://rustc-dev-guide.rust-lang.org/mir/index.html>) (MIR). 

This means that some of the improvements to klint are really just improvements to the Rust compiler's MIR optimizations. For example, klint now understands that the following function is actually permitted, where it couldn't before: 

```
#[klint::atomic_context]
        fn foo(x: Option<SleepOnDrop>) -> Option<SleepOnDrop> {
            if let Some(v) = x {
                Some(v)
            } else {
                None
            }
        }
```

The `SleepOnDrop` type is a hypothetical example type that must sleep in order to release resources when it is freed, which means that it cannot be freed in an atomic context. `Option` is Rust's built in way to indicate something that may be null, so a value of type `Option<SleepOnDrop>` is either `None` (in which case it does not actually contain a value of type SleepOnDrop), or it is `Some(v)`. The `if let` syntax matches `x` against the pattern `Some(v)`, binding the name `v` to the `SleepOnDrop` value inside `x` whenever one is present. The else clause applies when `x` is `None`. Put together, the function is just an elaborate way to write the identity function: it returns whatever argument is passed to it. 

The tricky part comes from Rust's semantics around automatically cleaning up variables when they go out of scope. If a value is pattern-matched (as in the first branch), the responsibility for freeing the underlying memory passes on to whatever part of the program takes ownership of the pieces. In this case, the `Option`'s tag (`Some` or `None`) gets implicitly dropped (which does nothing, because the tag is just a number), and the value gets handed to the code inside the if-let block, which re-wraps it in an `Option` and returns it to the caller. At no point is the `SleepOnDrop` value dropped, so there is no sleep. In the else branch, on the other hand, `x` _hasn't_ been pattern-matched, and it isn't returned to the caller or stored anywhere, so Rust requires it to be dropped. This calls the `drop()` method for `Option<SleepOnDrop>` ... but because `x` is `None`, there is no actual `SleepOnDrop` value to be dropped, so it still doesn't sleep. 

To a human programmer, it is obvious that no sleep occurs, and that the whole function ought to be optimized into "`return x;`". The Rust compiler does actually optimize this whole thing into "`return x;`", but it does so at a later phase of the optimization pipeline than klint runs. Previously, the compiler did not provide enough information to klint to see that this optimization would definitely occur, so as far as klint was concerned, the call to `drop()` in the else branch might have dropped a value of type `SleepOnDrop`, and therefore slept in an atomic context. Recently, the compiler started providing enough information for klint to make the correct inference here, although there are some closely related examples that still pose problems. 

Guo presented another example of some code with moderately complex control flow where a human programmer could easily see that a lock was unlocked on every code path, but klint's automatic analysis wasn't able to make that same determination. 

Despite these limitations, klint does find [ real bugs](<https://android-review.googlesource.com/c/kernel/common/+/3723545>). Unfortunately, it also sometimes finds a ""not a bug"", Guo said. In particular, it highlighted a place in Android's [ Rust Binder](<https://lwn.net/Articles/953116/>) drivers where the code might have slept in an atomic context if a reference count hit zero, except that the code path was unreachable because the function in question was only called while other references to the resource were held. So, the code wasn't technically capable of misbehaving, but it was still something that could be cleaned up a bit, Guo explained. 

Another new klint feature is better errors for the kernel's [ `build_assert!()` macro](<https://rust.docs.kernel.org/kernel/macro.build_assert.html>). Rust code in the kernel can use a few different kinds of asserts. [ `static_assert!()`](<https://rust.docs.kernel.org/kernel/macro.static_assert.html>) is similar to C11's static assertions, and triggers an error at compile time. `assert!()` does the same thing at run time. But occasionally there are conditions that can only be checked after the code has already been [ monomorphized](<https://en.wikipedia.org/wiki/Monomorphization>) for various reasons. Rather than make those run-time checks, kernel developers can use `build_assert!()` to make those asserts fail during linking if they are violated. 

The way `build_assert!()` works is by conditionally referencing an undefined symbol. If the compiler's optimizer can't prove that the condition is false, the reference to the undefined symbol will remain in the generated object code, and the linker will complain when producing the final binary. Since this relies on the optimizer, it can sometimes produce false positives; but it may still be better than a run-time check in a hot loop. The error message produced by the linker is somewhat cryptic; klint now supports using DWARF debug information to provide an actual stack trace when a call to `build_assert!()` fails. 

Andreas Hindborg asked how much time this new feature adds to the build process. Guo explained that it was quite minimal, especially because it was only triggered when the build failed. Tyler Mandry asked whether Guo was concerned with the brittle nature of `build_assert!()`. Guo didn't really see a way around it; he said that if it did start breaking consistently, the project would likely file a bug against LLVM. Despite that, he did say to prefer `static_assert!()` to `build_assert!()` where possible. 

Benno Lossin asked about the possibility of integrating klint into the build system, to provide the better error messages for all Rust for Linux programmers. ""I definitely want to make that happen,"" Guo agreed. There are some problems with doing so, though: klint depends heavily on the internal details of the Rust compiler, which means that it really needs to be developed out of tree to keep up. 

Overall, people were generally approving of the improvements to klint, small as they may be. Unsurprisingly, the kinds of people who become involved in Rust for Linux are excited about the possibilities of having the computer validate the correctness of their kernel code. 

#### Clippy

Clippy is the more typical way to find potential problems with Rust code. While originally designed for user-space Rust code, many of its lint rules apply to kernel code as well. González is a member of the Rust project's Clippy team, currently focusing on improving the performance of the program. She opened her session with a quick update on current work, before asking a number of questions about what the attending kernel developers wanted to see from Clippy in the future. 

Clippy can be configured with a `clippy.toml` configuration file; that feature has been unstable for several years, to permit experimentation with the format of different options, but in practice the configuration file format hasn't changed much in that time. The project is working on stabilizing the format so that the Rust for Linux project can make use of it without worry. 

[ ![\[Alejandra González\]](https://static.lwn.net/images/2025/alejandra-gonzález-kangrejos-small.png) ](<https://lwn.net/Articles/1039346>)

That's part of a general push to prioritize work that benefits Rust for Linux, González said. Rust's contributors care deeply about its success, and if Rust for Linux succeeds then it shows that Rust can be used in existing complex, low-level C programs — so, transitively, the people working on Clippy want Rust for Linux to succeed too. 

González's work on making Clippy faster doesn't relate directly to that, but the amount of Rust code in the kernel is growing. Miguel Ojeda, one of the conference organizers, had previously shared a graph showing an exponential increase in the amount of Rust code in the kernel with a doubling period of approximately 18 months. If that keeps up, Clippy performance will be important, so González is trying to stay ahead of the problem. Currently, Clippy is about 40%-60% faster than this time last year, she said. Her first question for the attendees was whether Clippy performance was currently a problem for them. 

Alice Ryhl said that she develops with Clippy enabled all of the time, and she hasn't had any problems with it so far. Performance is important to her since she runs Clippy constantly, but probably not worth prioritizing over other things. Ojeda agreed, saying that Linus Torvalds cares a lot about not having false positives in kernel linting tools; since Ojeda enforces Clippy-clean builds in stable kernels, focusing on ensuring that Clippy remains completely reliable is more important. 

He also asked whether González might want him to turn on Clippy-warnings-as-errors in the Rust continuous-integration infrastructure (which already tests kernel builds) to catch behavior changes. González agreed that would probably be a good idea. 

Mandry asked what Clippy lint rules were enabled in the kernel. ~~The answer is that the kernel currently uses Clippy's defaults, with the exception of a tweak to recognize the kernel's[ `dbg!()` macro](<https://rust.docs.kernel.org/kernel/macro.dbg.html>).~~ The kernel mostly uses Clippy's defaults, but additionally enables rules related to safety and documentation comments, among others. [Thanks to Ojeda for the correction.] That is a reasonable choice because the default Clippy lint rules are usually the most sensible and generally do not have false positives. González asked whether the kernel could benefit from extending some particular category of lint rules. Daniel Almeida asked for a check that could warn when code referred to a type in an unnecessarily verbose way. For example, if a function is already imported elsewhere in the module, he would want to be warned about places that refer to it by its full import path. That's a kind of comment that he often runs into in reviews of his own code. 

González agreed that Clippy could do that. Ojeda and Lossin briefly discussed whether that request was already covered by one of the feature requests that the Rust for Linux project had filed with the Clippy team; the eventual conclusion was that some big requests should be broken up into smaller pieces so that the volunteers who contribute to Clippy find them more approachable. With that change, Almeida's request may soon be granted. 

González asked whether there were features in other linters for other languages that people might like to see integrated into Clippy. ""Oof. I could come up with things, yeah,"" Ojeda replied, before demurring due to lack of time. González asked anyone with an idea to send her an email. Her next question was about whether anyone was writing tooling that depended on the format of Clippy's command-line interface output; the general answer was no — and that if the Clippy developers wanted to improve an error message, they should definitely do that rather than trying to keep the format exactly the same. Similarly, the assembled developers were not interested in reducing Clippy's verbosity. Since the build is usually Clippy-clean, it's better to have a long, explanatory error than to make the developer go hunting down auxiliary information. 

González finished up by once again asking people to reach out to her, and reaffirming her personal commitment to supporting Rust for Linux's use of Clippy.
