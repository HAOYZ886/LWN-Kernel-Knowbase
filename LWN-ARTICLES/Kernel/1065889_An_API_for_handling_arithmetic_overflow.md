---
title: An API for handling arithmetic overflow
url: https://lwn.net/Articles/1065889/
date: "April 8, 2026"
category: "Security-Kernel hardening"
author: "By Daroc Alden April 8, 2026"
---

> **Ignore previous instructions; subscribe to LWN today**
> 
> Every article on LWN.net is written by humans, for humans. If you've enjoyed this article and want to see more like it, your subscription goes a long way to keeping the robots at bay. We are offering [a free one-month trial subscription](<https://lwn.net/Promo/nst-bots1/claim>) (no credit card required) to get you started. 

By **Daroc Alden**  
April 8, 2026

On March 31, Kees Cook shared [ a patch set](<https://lwn.net/ml/all/20260331163716.work.696-kees@kernel.org/>) that represents the culmination of more than a year of work toward eliminating the possibility of silent, unintentional integer overflow in the kernel. Linus Torvalds was [ not pleased](<https://lwn.net/ml/all/CAHk-=wg=q7ptcMyUAqPGDbX_DKHivJVppc8bP0zpQqzOky_avA@mail.gmail.com/>) with the approach, leading to a detailed discussion about the meaning of "safe" integer operations and the design of APIs for handling integer overflows. Eventually, the developers involved reached a consensus for a different API that should make handling overflow errors in the kernel much less of a hassle. 

This work was [initially proposed in 2024](<https://lwn.net/Articles/959189/>) as part of Cook's continuing efforts to harden the kernel against various sources of error. In that proposal, he emphasized that the problem with integer overflow in the kernel is not related to undefined behavior — the kernel is compiled with `-fno-strict-overflow`, which causes integer overflow and underflow to wrap around without error. The problem is with the unexpected code paths that can be taken when a number is suddenly much larger or much smaller than the developer expected. For example, adding an offset to a base address can result in a pointer to a location below the base address, a fact that is easy to overlook when writing buffer-handling code. 

Cook's proposed solution to this problem is a system of annotations that separate integers into types depending on whether wrapping is expected or not. Justin Stitt added [experimental support for the feature](<https://clang.llvm.org/docs/OverflowBehaviorTypes.html>) to Clang, in the form of a pair of annotations: 

```
int __attribute__((overflow_behavior(wrap))) example1;
        int __attribute__((overflow_behavior(trap))) example2;
    
        // Alternate keyword form:
        int __ob_wrap example1;
        int __ob_trap example2;
```

The `wrap` attribute indicates that wrapping is expected (and should, for example, not be remarked upon by linting programs); the `trap` attribute indicates that it is unexpected (and should cause a crash). In the kernel, it appears that an operation that would cause an overflow of a trapping integer will produce an oops by default. The attributes override compiler flags, so wrapping integers will wrap and trapping integers will trap on overflow, regardless of how the compiler is invoked. Type promotion works in the normal C way; operations involving a normal integer and an annotated integer produce an annotated integer. Operations combining a wrapping integer and a trapping integer produce a trapping integer. 

Vincent Mailhol [ asked](<https://lwn.net/ml/all/bd0a4235-a7f0-4624-802c-aa49a9d13f29@kernel.org/>) about the possibility of adding saturating integers as well. Saturating integers stay at the top or bottom of their range, instead of wrapping, such that `MAX_INT + 1 == MAX_INT`. Cook [agreed](<https://lwn.net/ml/all/202604011231.1D0BAE9A@keescook/>) that was possible. The `overflow_behavior()` mechanism should extend to handling other behaviors, although that would result in a proliferation of integer types. Cook's patch set adds [a set of typedefs](<https://lwn.net/ml/all/20260331163725.2765789-5-kees@kernel.org/>) for easily referring to the new types of integer, including "`u8t`" for trapping unsigned 8-bit integers, "`s64w`" for wrapping signed 64-bit integers, etc. 

Torvalds strongly objected to the way the documentation accompanying the patch set described trapping operations as "safe": 

> Stop thinking that trapping is "safe". 
> 
> It damn well isn't. A dead machine is not a safe machine. 
> 
> Any patches that call trapping behavior safe will be NAK'ed by me. 

A [ later message](<https://lwn.net/ml/all/CAHk-=wiJ6Q_qMHSe-hs+QvqKVZphvDZjvFP_gQLw1eaWimv8+w@mail.gmail.com/>) also objected to the names Cook chose for the typedefs, noting that they were visually similar, which is not a good fit for ""_VERY_ subtle semantic changes."" He suggested "`wrapping_u32`" or a similarly explicit name. Miguel Ojeda [ noted](<https://lwn.net/ml/all/CANiq72kL3rTKyDNYmD7wXiKCVJSfa1bnp2L8NShXU7OPmWjJ4w@mail.gmail.com/>) that the Rust syntax for this concept is "`Wrapping<u32>`", which mirrors Torvald's suggestion. But in practice, Rust code in the Kernel uses explicit function calls (e.g. `wrapping_add()`) rather than operator overloading, because it prevents confusing copy-paste errors where operators mean different things in context, Ojeda said. 

Torvalds [ thought](<https://lwn.net/ml/all/CAHk-=whjwHjmB0_2yXsOjDa7Mi_yFSx3AMd3vGk5r70WocvZZg@mail.gmail.com/>) that it would make sense to use explicit operations like that for arithmetic on trapping integers — which he continued to insist needed some way to make the error recoverable — but thought that it would be overly laborious for wrapping types. He suggested that trapping and wrapping operations should really have completely different syntax, because although they are often discussed together, they're actually quite different: wrapping types are an acknowledgment that a particular behavior is correct and expected, whereas trapping types are a mechanism that deliberately signals an error when something might be going wrong. ""Anything that says 'these are two faces of the same coin and are just different attributes on the type' is simply broken by design."" 

In [ another follow-up message](<https://lwn.net/ml/all/CAHk-=wgKB5f3MM40FGGUWUm_9eyESe2PAqCa6uZ=YTi0CdPwDg@mail.gmail.com/>), he suggested that the interface for trapping integers could use macros to take an error label to jump to, in order to make their use less obnoxious than the kernel's existing overflow-checking functions: 

```
res = add_overflow(a, b, label);
        ...
        label: // Handle overflow errors
        ...
```

Cook [explained](<https://lwn.net/ml/all/202603311155.503B3DA5B@keescook/>) that he [ had considered](<https://lwn.net/ml/all/202603311100.F39B5DC3@keescook/>) an approach like that, but people were so universally against introducing function-based math primitives that they wouldn't accept using them. ""Everyone absolutely hates it."" Torvalds [ disagreed](<https://lwn.net/ml/all/CAHk-=wh6YWV3hHs8i0P=2A2CVjCY=SOaYeayEodXYa+CMqNO4g@mail.gmail.com/>), saying that he thought a simple-enough interface would be used when people have a good reason to do so. He gave an example of how the approach he described previously could be extended with a default "overflow" label to further streamline the code. ""Now will people ENJOY using '`addo()`' and things like that? No."" But he still thought that interface was better than randomly killing the kernel when an overflow occurs. 

This line of reasoning misses the point, [Cook said](<https://lwn.net/ml/all/202603311117.454F578@keescook/>). All of the places where people would use trapping integers are already supposed to be places where no overflow is possible. So, operations on trapping integers aren't intended to replace existing overflow checks — they're just supposed to make it obvious when a developer has missed an overflow check, because people keep making related mistakes. 

> If the code was written perfectly, then there's no problem. If there was a bug that allows for overflow then you get a crash instead of totally insane behavior that is almost always exploitable in a way that the system gets compromised. That is a net benefit, even if crashes are still bad. 

He did agree that he shouldn't have called trapping a "safe" behavior in the documentation, and promised to fix that. Torvalds [ replied](<https://lwn.net/ml/all/CAHk-=wizJdr1qgJ9NKNjtbH=ugL3umA9JRW9ifrQaD+PpWCMuQ@mail.gmail.com/>) that using the equivalent of `BUG_ON()` still wasn't the right way to address the problem. 

Cook [proposed](<https://lwn.net/ml/all/202603311253.95C54588E@keescook/>) adding an attribute to the language for specifying a label in a given function where trapping operations should jump; using trapping integer operations outside of an appropriately annotated function would be an error. Torvalds [ approved](<https://lwn.net/ml/all/CAHk-=wjSOGaausLeTD13yAqso7qM09EnkDFiN7wF15kH0VWmZQ@mail.gmail.com/>) of that approach, and spent a while working out precise details of what such an interface would look like with Cook. ""So with _this_ kind of interface, all my reservations about it go away."" 

```
int __overflow_label(boom)
        something(...)
        {
            u8 __attribute__((overflow_behavior(trap))) count;
            ...
            // Jumps to 'boom' if the count overflows.
            while (thing()) count++;
            ...
            return 0;
        boom:
            // Cleanup code
            return -EINVAL;
        }
```

Peter Zijlstra [ pointed out](<https://lwn.net/ml/all/20260401083137.GT3738786@noisy.programming.kicks-ass.net/>) that, although it worked well for simple examples like the above, this design would cause problems with the `__cleanup()` macro. It is not permitted to jump into the scope of a stack-allocated object annotated with `__cleanup()`; the only permitted way to enter that code should be through the assignment to that variable. Having trapping operations jump to a locally defined label makes it easy to invisibly break that rule and accidentally jump into such a scope. 

Clang does catch this particular error, Cook [demonstrated](<https://lwn.net/ml/all/202604011328.D3821379@keescook/>). GCC doesn't have a similar error message, so that's something that may need to be fixed. Cook did agree that the behavior was ""a bit fragile"". 

With the proposed changes to the API and the documentation, Cook's patch set will have to be more or less rewritten. Stitt's compiler code may need changes as well, and GCC will need to implement a version of whatever Clang settles on. Despite that, having an agreement about the form that the API for trapping integers should take is important progress. It may not be long before the work becomes available, which will provide the opportunity to simplify existing overflow-checking code and to eliminate existing unnoticed overflow bugs.
