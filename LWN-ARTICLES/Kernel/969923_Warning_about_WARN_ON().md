---
title: Warning about WARN_ON()
url: https://lwn.net/Articles/969923/
date: "April 18, 2024"
category: Warnings
author: "By Jonathan Corbet April 18, 2024"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
April 18, 2024

Kernel developers, like conscientious developers for many projects, will often include checks in the code for conditions that are never expected to occur, but which would indicate a serious problem should that expectation turn out to be incorrect. For years, developers have been encouraged (to put it politely) to avoid using assertions that crash the machine for such conditions unless there is truly no alternative. Increasingly, though, use of the kernel's `WARN_ON()` family of macros, which developers were told to use instead, is also being discouraged. 

A longstanding way to test for a condition that cannot be recovered from is the [`BUG_ON()` macro](<https://elixir.bootlin.com/linux/latest/source/include/asm-generic/bug.h#L51>), which includes a test for the unexpected condition: 

```
/* This can never happen, honest, would I lie? */
        BUG_ON(foo_ptr == NULL);
```

A `BUG_ON()` call leads directly to a kernel panic, resulting (usually) in the machine being rebooted. There are times when there is no alternative, but use of `BUG_ON()` has been discouraged for years. Crashing the machine deprives the user of any chance of reacting to the problem or saving work and can make it harder to track down the source of the problem. Even so, there are something like 12,000 `BUG_ON()` instances in the kernel (not counting `BUILD_BUG_ON()`, which only affects the build process and is not discouraged in the same way). 

Instead, developers are told to use [`WARN_ON()`](<https://elixir.bootlin.com/linux/latest/source/include/asm-generic/bug.h#L74>), which puts a traceback into the kernel log but does not crash the machine (in theory, at least, but keep reading). The kernel's [coding-style document](<https://docs.kernel.org/process/coding-style.html#use-warn-rather-than-bug>) says: 

> Do not add new code that uses any of the BUG() variants, such as BUG(), BUG_ON(), or VM_BUG_ON(). Instead, use a WARN*() variant, preferably WARN_ON_ONCE(), and possibly with recovery code. Recovery code is not required if there is no reasonable way to at least partially recover. 

Increasingly, though, developers are being told to avoid using `WARN_ON()` as well. There are a couple of reasons for that. One is that any `WARN_ON()` that can be triggered from user space can be used, at a minimum, to spam the system log, obscuring other events and perhaps affecting system performance. The other reason, though, applies even to `WARN_ON()` calls that cannot be triggered in this way. 

The kernel contains a sysctl knob called [`panic_on_warn`](<https://docs.kernel.org/admin-guide/sysctl/kernel.html#panic-on-warn>). It does exactly what its name suggests: if this option is set, any `WARN_ON()` call will cause the system to panic. In essence, it turns `WARN_ON()` calls into `BUG_ON()` calls. This option is set by users who see any warning as a sufficiently suspicious event that, when one happens, it is better to kill the system and start over. Such users include many Android devices and the host kernels run at cloud providers (and beyond). Any `WARN_ON()` that actually triggers, in other words, has the potential to bring down a lot of machines. 

The same coding-style document [advises developers](<https://docs.kernel.org/process/coding-style.html#do-not-worry-about-panic-on-warn-users>) that this outcome is something that `panic_on_warn` users have explicitly opted into: 

> However, the existence of panic_on_warn users is not a valid reason to avoid the judicious use WARN*(). That is because, whoever enables panic_on_warn has explicitly asked the kernel to crash if a WARN*() fires, and such users must be prepared to deal with the consequences of a system that is somewhat more likely to crash. 

The current pressure against `WARN_ON()` use is not entirely consistent with this advice, though. Thus, Alex Elder was recently motivated to [send a patch](<https://lwn.net/ml/linux-kernel/20240414170850.148122-1-elder@linaro.org/>) changing the advice given in the coding-style document. Gone is the language suggesting that `panic_on_warn` users were getting what they asked for; the new text reads: 

> The existence of this option is not a valid reason to avoid the judicious use of warnings. There are other options: ``dev_warn*()`` and ``pr_warn*()`` issue warnings but do **not** cause the kernel to crash. Use these if you want to prevent such panics. 

Christoph Hellwig was quick to [call](<https://lwn.net/ml/linux-kernel/ZhzgTeEHFF19N3UZ@infradead.org/>) this change ""wronger than wrong"": ""If you set panic_on_warn you get to keep the pieces"". Laurent Pinchart [pointed out](<https://lwn.net/ml/linux-kernel/20240414194835.GA12561@pendragon.ideasonboard.com/>) that the suggested alternatives are not the same; they are much easier to ignore and, thus, less effective at getting developers to fix the problem that the warning is trying to draw attention to. Greg Kroah-Hartman, though, [was happy](<https://lwn.net/ml/linux-kernel/2024041510-tacky-childlike-fc6d@gregkh/>) to see this change. The recommendation to avoid `panic_on_warn` has been ignored, he [said](<https://lwn.net/ml/linux-kernel/2024041544-fester-undead-7949@gregkh/>), so new `WARN_ON()` calls should not be added. 

To summarize the situation: over the years, `BUG_ON()` has been seen as so destructive that developers are simply [told not to use it at all](<https://docs.kernel.org/process/deprecated.html#bug-and-bug-on>). The `WARN_ON()` macro has, instead, taken its place; but in settings where `panic_on_warn` is set, the end result of a `WARN_ON()` call is essentially the same. So, naturally, use of `WARN_ON()` is also now discouraged much of the time. 

Whether the proposed documentation change will be applied is unclear; the kernel's befuddled documentation maintainer, who has happily not been appointed the arbiter of the kernel's coding style, makes a point of not applying coding-style changes in the absence of a clear consensus. It is not clear that a consensus on this change exists currently. Regardless of that change, though, developers will continue to be encouraged toward logging functions like `pr_warn()` instead of `WARN_ON()` — until somebody inevitably adds a `panic_on_pr_warn` sysctl knob and the whole process starts over again.
