---
title: "Who is the best inliner of all?"
url: https://lwn.net/Articles/314848/
date: "January 14, 2009"
category: Development tools; Inline functions
author: "By Jonathan Corbet January 14, 2009"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
January 14, 2009

The `inline` keyword provided by GCC has always been a bit of a dangerous temptation for kernel programmers. In many cases, making a function inline can help performance. In some, it is mandatory; this is especially true for functions which encapsulate specific CPU instructions. But, in other cases, inlining becomes a classic example of premature optimization; at best, it does not help, while, at worst, it can significantly bloat the size of the kernel and harm performance. Since performance matters to kernel developers, the proper way of inlining functions has often been a topic of discussion. The most recent debate on the subject has made it clear, though, that there is still no real consensus on the issue. 

The discussion began as an offshoot of the spinning mutex topic when Linus [noticed](<https://lwn.net/Articles/314851/>) that a posted kernel oops listing showed that the `__cmpxchg()` function had _not_ been inlined. This function provides access to the x86 `cmpxchg*` instructions; it should expand to a single instruction. Clearly it makes sense to inline a single-instruction function, but, for whatever reason, GCC had decided not to do that. 

Linus quickly concluded that the fault lies with the (non-default) `CONFIG_OPTIMIZE_INLINING` configuration option. This option, when selected, makes `inline` into a suggestion which GCC is free to ignore. At that point, GCC makes its own decisions, based on a set of built-in heuristics. In this case, it decided that `__cmpxchg()` was too complex to inline, so it made it into a separate function. Linus, in disgust, asked Ingo Molnar to remove `CONFIG_OPTIMIZE_INLINING` and force the compiler to honor the `inline` keyword. 

Some other developers agreed with this request - but not all. GCC will still certainly make mistakes, but there is also a growing feeling that, with more recent versions of the compiler, GCC is able to make good decisions most of the time. If GCC is also given the power to inline functions which have not been explicitly marked by the developer, the results can be even better. There are hazards, though, to giving GCC an overly free hand: excessive inlining can create stack usage problems and make debugging harder. But these are problems that some developers are willing to accept if the benefits are strong enough. 

Ingo ran [a long series of tests](<https://lwn.net/Articles/314852/>) to see what happens when GCC is given free rein over the inlining of functions. His results were fairly clear: recent GCC, when allowed to make its own inlining decisions, produces a kernel that is 1-7% smaller than the kernel which results from strictly following `inline` declarations. From that data, Ingo [concludes](<https://lwn.net/Articles/314855/>) that the best solution is to use the inlining features built into the compiler: 

Today we have in excess of thirty thousand 'inline' keyword uses in the kernel, and in excess of one hundred thousand kernel functions. We had a decade of hundreds of inline-tuning patches that flipped inline attributes on and off, with the goal of doing that job better than the compiler. 

Still a sucky compiler who was never faced with this level of inlining complexity before (up to a few short months ago when we released the first kernel with non-CONFIG_BROKEN-marked CONFIG_OPTIMIZE_INLINING feature in it) manages to do a better job at judging inlining than a decade of human optimizations managed to do. (If you accept that 1% - 3% - 7.5% code size reduction in important areas of the kernel is an improvement.) 

Linus, however, [is unimpressed](<https://lwn.net/Articles/314856/>). In his point of view, the kernel size reduction provided by automated inlining does not outweigh the drawbacks: 

It's not about size - or necessarily even performance - at all. It's about abstraction, and a way of writing code. 

And the thing is, as long as gcc does what we ask, we can notice when _we_ did something wrong. We can say "ok, we should just remove the inline" etc. But when gcc then essentially flips a coin, and inlines things we don't want to, it dilutes the whole value of inlining - because now gcc does things that actually does hurt us. 

We get oopses that have a nice symbolic back-trace, and it reports an error IN TOTALLY THE WRONG FUNCTION, because gcc "helpfully" inlined things to the point that only an expert can realize "oh, the bug was actually five hundred lines up, in that other function that was just called once, so gcc inlined it even though it is huge". 

See? THIS is the problem with gcc heuristics. It's not about quality of code, it's about RELIABILITY of code. 

The reason people use C for system programming is because the language is a reasonably portable way to get the expected end results WITHOUT the compiler making a lot of semantic changes behind your back. 

Linus would rather that the `inline` keyword be considered mandatory by the compiler. Then, if there are too many inline functions in the kernel (and 30,000 of them does seem like a fairly high number), the unnecessary `inline` keywords should be removed. There was some talk of adding some sort of `inline_hint` keyword for cases where inlining is just a suggestion, but there is not much enthusiasm for that approach. 

The problem with the all-manual approach - even assuming that it can yield the best results - was perhaps [best expressed](<https://lwn.net/Articles/314857/>) by Ingo: 

In this cycle alone, in the past ~2 weeks we added another 1300 inlines to the kernel. Do we really want periodic postings of: 

[PATCH 0/135] inline removal cleanups 

... in the next 10 years? We have about 20% of all functions in the kernel marked with 'inline'. It is a _very_ strong habit. Is it worth fighting against it? 

Solving excessive use of inline functions by diluting the meaning of the `inline` keyword may look like a misdirected solution. But the alternative would require much more attentive review of kernel patches before they go into the mainline. History suggests that getting that level of review is an uphill battle at best. History also shows that compilers tend to be better than programmers at making this kind of decision, especially when behavior over an entire body of code (as opposed to in a single function) is considered. But it may be a while, yet, before the development community as a whole is willing to put that level of trust into its tools.
