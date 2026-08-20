---
title: "Would you like signs with those chars?"
url: https://lwn.net/Articles/911914/
date: "October 24, 2022"
category: Build system
author: "By Jonathan Corbet October 24, 2022"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jonathan Corbet**  
October 24, 2022

Among the many quirks that make the C language so charming is the set of behaviors that it does not define; these include whether a `char` variable is a signed quantity or not. The distinction often does not make a difference, but there are exceptions. Kernel code, which runs on many different architectures, is where exceptions can certainly be found. A recent attempt to eliminate the uncertain signedness of `char` variables did not get far — at least not in the direction it originally attempted to go. 

As a general rule, C integer types are signed unless specified otherwise; `short`, `int`, `long` all work that way. But `char`, which is usually a single byte on current machines, is different; it can be signed or not, depending on whatever is most convenient to implement on any given architecture. On x86 systems, a `char` variable is signed unless declared as `unsigned char`. On Arm systems, though, `char` variables are unsigned (unless explicitly declared `signed`) instead. 

The fact that a `char` variable may or may not be signed is an easy thing for a developer to forget, especially if that developer's work is focused on a single architecture. Thus, x86 developers can get into the habit of thinking of `char` as always being signed and, as a result, write code that will misbehave on some other systems. Jason Donenfeld recently encountered this sort of bug and, after fixing it, posted [a patch](<https://lwn.net/ml/linux-kernel/20221019162648.3557490-1-Jason@zx2c4.com/>) meant to address this problem kernel-wide. In an attempt to ""just eliminate this particular variety of heisensigned bugs entirely"", it added the `-fsigned-char` flag to the compiler command line, forcing the bare `char` type to be signed across all architectures. 

This change turned out to not be popular. Segher Boessenkool [pointed out](<https://lwn.net/ml/linux-kernel/20221019165455.GL25951@gate.crashing.org/>) that it constitutes an ABI change, and could hurt performance on systems that naturally want `char` to be unsigned. Linus Torvalds [agreed](<https://lwn.net/ml/linux-kernel/CAHk-=wiMWk2t8FHn0iqVVe1mn62OTAD6ffL5rn9Eeu021H9d1Q@mail.gmail.com/>), saying that: ""We should just accept the standard wording, and be aware that 'char' has indeterminate signedness"". He disagreed, however, with Boessenkool's suggestion to remove the `-Wno-pointer-sign` option used now (thus enabling `-Wpointer-sign` warnings). That change would enable a warning that results from the mixing of pointers to signed and unsigned `char` types; Torvalds complained that it fails to warn when using `char` variables, but [produces a lot of false positive warnings](<https://lwn.net/ml/linux-kernel/CAHk-=wiNNKLFfa0d+Hk=Wm5caiKjLY4V9wwu9DhcSSwPuMbxrg@mail.gmail.com/>) with correct code. 

Later in the discussion, though, Torvalds [wondered](<https://lwn.net/ml/linux-kernel/CAHk-=whT+xyge9UjH+r6dt0FG-eUdrzu5hDMce_vC+n8uLam2A@mail.gmail.com/>) whether it might be a good idea to nail down the signedness of `char` variables after all — but to force them to be _un_ signed by default rather than signed. That, he said, shouldn't generate worse code on any of the commonly used architectures. ""And I do think that having odd architecture differences is generally a bad idea, and making the language rules stricter to avoid differences is a good thing"". Amusingly, he noted that, with this option, code like: 

```
const unsigned char *c = "Subscribe to LWN";
```

will still, with the `-Wpointer-sign` option, generate a warning, since a string constant pointer is still considered to be a bare `char *` type, which is then treated as being different from an explicit `unsigned char *` type. ""You *really* can't win this thing. The game is rigged like some geeky carnival game"". 

Donenfeld [saw merit](<https://lwn.net/ml/linux-kernel/Y1BcpXAjR4tmV6RQ@zx2c4.com/>) in the idea, even though he thinks that the potential to break some code exists. He sent out [a new patch](<https://lwn.net/ml/linux-kernel/20221019203034.3795710-1-Jason@zx2c4.com/>) adding `-funsigned-char` to the compiler command line to effect this change. He had suggested that it could perhaps be merged immediately, given that there is time to fix any fallout before the 6.1 release, but Torvalds [declined that opportunity](<https://lwn.net/ml/linux-kernel/CAHk-=wit-67VU=kt-8Ojtx04m6wxfqypKLzW7CuSeEH_9MYZvw@mail.gmail.com/>): ""if we were still in the merge window, I'd probably apply this, but as things stand, I think it should go into linux-next and cook there for the next merge window"". He added that any problems that result from the change are likely to be subtle and to be in driver code that isn't widely used across architectures. The core kernel code, instead, has always had to work across architectures, so he does not believe that problems will show up there. 

So Donenfeld's patch is sitting in linux-next instead, waiting for the 6.2 merge window in December. That gives the community until late February to find any problems that might be caused by forcing bare `char` variables to be unsigned across all architectures supported by Linux. That is a fair amount of time, but it is also certainly not too soon to begin testing this change in as many different environments as possible. It is, after all, a fundamental change to the language in which the kernel is written; a lack of resulting surprises would, itself, be surprising. 

One way to identify potential problems is to find the places where the generated code changes when `char` is forced to be unsigned. Torvalds has already [made some efforts in that direction](<https://lwn.net/ml/linux-kernel/CAHk-=whg00wpUzNLs0obmMKA3GhUnLzat9syA1=_tfi8Ms8TLg@mail.gmail.com/>), and Kees Cook has [used a system designed for checking reproducible builds](<https://lwn.net/ml/linux-kernel/202210201056.DEE610F6F@keescook/>) to find a lot of changes. Many of those changes will turn out to be harmless, but the only way to know for sure is to actually look at them. Meanwhile, the [posting of one fix](<https://lwn.net/ml/linux-kernel/Y1EZuQcO8UoN91cX@localhost.localdomain/>) by Alexey Dobriyan has caused Torvalds to [request](<https://lwn.net/ml/linux-kernel/CAHk-=whFow9Wd6C8htoRUt5wXbwf1i_qbuArBbhXOPqYsTFvtw@mail.gmail.com/>) that the `char` fixes be collected into a single tree. As those fixes accumulate, the result should be a sign of just how much disruption this change is actually going to cause.
