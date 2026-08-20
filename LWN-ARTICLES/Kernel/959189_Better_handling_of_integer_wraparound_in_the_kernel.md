---
title: Better handling of integer wraparound in the kernel
url: https://lwn.net/Articles/959189/
date: "January 26, 2024"
category: "Security-Kernel hardening"
author: "By Jonathan Corbet January 26, 2024"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
January 26, 2024

While the mathematical realm of numbers is infinite, computers are only able to represent a finite subset of them. That can lead to problems when arithmetic operations would create numbers that the computer is unable to store as the intended type. This condition, called "overflow" or "wraparound" depending on the context, can be the source of bugs, including unpleasant security vulnerabilities, so it is worth avoiding. [This patch series](<https://lwn.net/ml/linux-kernel/20240122235208.work.748-kees@kernel.org/>) from Kees Cook is intended to improve the kernel's handling of these situations, but it is running into a bit of resistance. 

Cook starts by clarifying the definitions of two related terms: 

  * **Overflow** happens when a signed or pointer value exceeds the range of the variable into which it is stored. 
  * **Wraparound** happens, instead, when an unsigned integer value exceeds the range that its underlying storage can represent. 

This distinction is important. Both overflow and wraparound can create surprises for a developer who is not expecting that situation. But overflow is considered to be undefined behavior in C, while wraparound is defined. As a result, overflow brings the possibility of a different kind of surprise: since it is undefined behavior, compilers can feel free to delete code that handles overflow or apply other unwelcome optimizations. To avoid this outcome, the kernel is built with the `-fno-strict-overflow` option, which essentially turns (undefined) overflow conditions into (defined) wraparound conditions. 

So, in a strict sense, overflows do not happen in the kernel, but wraparounds do. If a wraparound is intended, as is often the case in the kernel (see [`ip_idents_reserve()`](<https://elixir.bootlin.com/linux/v6.7.1/source/net/ipv4/route.c#L457>), for example), then all is fine. If the developer is not expecting wraparound, though, the results will not be so good. As a result, there is value in using tooling to point out cases where wraparound may happen — but only the cases where it is not intended. A wraparound detector that creates a lot of false-positive noise will not be welcomed by developers. 

In the past, the tooling, in the form of the [undefined behavior sanitizer (UBSAN)](<https://docs.kernel.org/dev-tools/ubsan.html>) and the GCC `-fsanitize=undefined` option, has indeed generated false-positive warnings. As a result, this checking was [disabled](<https://git.kernel.org/linus/6aaa31aeb9cf>) for the 5.12 release in 2021 and has remained that way ever since. Cook is now trying to re-enable UBSAN's wraparound checking and make it useful; the result was an 82-part patch set making changes all over the kernel. 

The key to making this checker useful is to prevent it from issuing warnings in cases where wraparound is intended. One way to do that is to explicitly annotate functions (generally of the small, inline variety) that are expected to perform operations that might wrap around and that handle that situation properly. The `__signed_wrap` and `__unsigned_wrap` annotations have been [duly added](<https://lwn.net/ml/linux-kernel/20240123002814.1396804-6-keescook@chromium.org/>) for this purpose; they work by disabling the checking of potential wraparound conditions in the marked function. 

The most common place where intentional wraparound is seen, though, is in code that is intended to avoid just that behavior. Consider [this code](<https://elixir.bootlin.com/linux/v6.7.1/source/mm/mmap.c#L3039>) in the implementation of the [`remap_file_pages()`](<https://man7.org/linux/man-pages/man2/remap_file_pages.2.html>) system call: 

```
/* Does pgoff wrap? */
        if (pgoff + (size >> PAGE_SHIFT) < pgoff)
    	return ret;
```

Normally, the sum of two unsigned values will be greater than (or equal to) both of those values. Should the operation wrap around, though, the resulting value will be _less_ than either of the addends. As a result, wraparound can be reliably detected with a test like the above. It is worth noting, though, that this test detects wraparound by causing it to happen; wraparound is an expected result that is properly handled. 

To a naive wraparound detector, though, that code looks like just the sort of thing it is supposed to issue warnings about. The resulting noise makes such a detector useless in general, so something needs to be done. In this case, Cook adds a pair of macros to explicitly annotate this type of code: 

```
add_would_overflow(a, b)
        add_wrap(a, b)
```

The first returns a boolean value indicating whether the sum of the two addends would wrap around, while the second returns that sum, which may have wrapped around. These macros are built on the kernel's existing [`check_add_overflow()`](<https://elixir.bootlin.com/linux/v6.7.1/source/include/linux/overflow.h#L54>) macro which, in turn, uses the compiler's [`__builtin_add_overflow()`](<https://gcc.gnu.org/onlinedocs/gcc/Integer-Overflow-Builtins.html>) intrinsic function. Using these, the above `remap_file_pages()` test is [rewritten](<https://lwn.net/ml/linux-kernel/20240123002814.1396804-63-keescook@chromium.org/>) as: 

```
/* Does pgoff wrap? */
        if (add_would_overflow(pgoff, (size >> PAGE_SHIFT)))
     	return ret;
```

This code now clearly does not risk an unwanted wraparound, and so no longer triggers a warning. The patch set rewrites a large number of these tests throughout the kernel. Along the way, Cook also had to [enhance `check_add_overflow()`](<https://lwn.net/ml/linux-kernel/20240123002814.1396804-1-keescook@chromium.org/>) to handle pointer arithmetic so that pointer additions can be easily checked as well. 

With all of this work in place, it is possible to turn on wraparound checking in UBSAN again. Eventually, the warnings generated should be accurate enough that it can be used to detect code that is not written with wraparound in mind. First, though, this work has to find its way into the mainline. In the best of times, a series that changes over 100 files across the kernel tree is going to be challenging to merge, though Cook has gotten fairly good at that task. 

A more difficult challenge may be the [opposition](<https://lwn.net/ml/linux-kernel/CAHk-=whS7FSbBoo1gxe+83twO2JeGNsUKMhAcfWymw9auqBvjg@mail.gmail.com/>) expressed by Linus Torvalds. He complained that the changelogs do not properly describe the changes that are being made, that the new annotations cause the compiler to generate less-efficient code, and that the tooling should recognize wraparound tests in the above form without the need for explicit annotation: ""if there's some unsigned wraparound checker that doesn't understand this traditional way of doing overflow checking, that piece of crap needs fixing"". He [added](<https://lwn.net/ml/linux-kernel/CAHk-=wgMPVv-mDxA2qcywpLCRLojtaKmP13h7bVo4m=XN202xA@mail.gmail.com/>) some conditions for merging these changes. 

Cook [answered](<https://lwn.net/ml/linux-kernel/202401231058.5C5248DC3@keescook/>) that he would rewrite the changelogs, which was one of the things Torvalds demanded. Another one of those demands — ""fix the so-called 'sanitizer'"" — might prove to be a bit more challenging, since it will require work on the compiler side. The advantage of such a fix is clear; it would remove the need for hundreds of explicit annotations in the kernel. But that would come at the cost of delaying this work and dealing with the bugs that enter the kernel in the meantime. 

The history of the hardening work in the kernel suggests that these little obstacles will indeed be overcome in time and that the kernel will eventually be free of wraparound bugs (or close to that goal, anyway). Of course, as Kent Overstreet took pains to [point out](<https://lwn.net/ml/linux-kernel/337u2jcp3s2vrvotj4jppgpr7zas5fod6k3mhe263ghpux7s5x@knz6eibm2jis/>), this work would not be necessary if the kernel would just take the minor step of switching to Rust. Cook [answered](<https://lwn.net/ml/linux-kernel/4DE92BD2-50FE-4634-ACF3-E0CC735EDAEB@kernel.org/>) that any such change is not happening soon, so he will continue his work of ""removing as many C foot-guns as possible"". As this work is wrapped up, the result should be a more stable and secure kernel for all of us.
