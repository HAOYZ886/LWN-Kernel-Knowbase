---
title: "Coding-style exceptionalism"
url: https://lwn.net/Articles/694755/
date: "July 20, 2016"
category: Coding style
author: "July 20, 2016 This article was contributed by Neil Brown"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

July 20, 2016

This article was contributed by Neil Brown

As I was analyzing the behavioral details of various drivers as part of my research for [a recent article on USB battery charging in Linux](<https://lwn.net/Articles/694062/>), I was struck by the the thought that code doesn't exist just to make certain hardware perform certain functions. Important though that is, the code in Linux, and in other open projects, also exists as a cultural artifact through which we programmers communicate and from which we learn. The disappointment I felt at the haphazard quality I found was not simply because some hardware somewhere might not perform optimally. It was because some other programmer tasked with writing a similar driver for new hardware might look to some of these drivers for inspiration, and might copy something that was barely functional and not use the best interfaces available.

With these thoughts floating around my mind I was interested to find a recent [thread](<http://thread.gmane.org/20160629144242.GE22818@mwanda>) on the linux-kernel mailing list that was more concerned about how a block of code looked than about what it did. 

The code in question handles SHA-256 hash generation on Intel x86 platforms. The thread started because Dan Carpenter's [smatch tool](<https://lwn.net/Articles/691882/>) had found some unusual code:

```
if ((ctx->partial_block_buffer_length) | (len < SHA256_BLOCK_SIZE)) {
```

The "`|`" here looks like it was probably meant to be "`||`" — so there was a bit-wise "or" where a logical "or" is more common. Carpenter went to some pains to be clear that he knew the code would produce the same result no matter which operator was used, but observed that ""it's hard to tell the intent."" Intent doesn't matter to a compiler, but it does to a human reader. Even well-written code can be a challenge to read due to the enormous amount of detail embedded in it. When there is an unusual construct that you need to stop and think about, that doesn't make it any easier.

There were a couple of suggestions that this was an intentional optimization and there is some justification for this. With both GCC 4.8 and 5.3 compiling for x86_64, the "`|`" version produces one fewer instruction, avoiding a jump. In some cases that small performance difference might be worth the small extra burden on the reader, though as Joe Perches [observed](<http://article.gmane.org/gmane.linux.kernel/2256830>): ""It's probably useful to add a comment for the specific intent here""; that would not only make it easier to read, but would ensure that nobody broke the optimization in the future. Further, the value of such optimizations can easily vary from compiler to compiler.

Once a little attention was focused on this code, other complaints arose, with [Ingo Molnar complaining](<http://article.gmane.org/gmane.linux.kernel.cryptoapi/20996>) about the excessive use of parentheses, the unusually long field name `partial_block_buffer_length`, and, responding to what is clearly a sore spot for some, requesting that the "customary" style be used for multi-line comments.

[`Documentation/Codingstyle`](<https://kernel.org/doc/Documentation/CodingStyle>) explains that:

```
The preferred style for long (multi-line) comments is:
    
            /*
             * This is the preferred style for multi-line
             * comments in the Linux kernel source code.
             * Please use it consistently.
             *
             * Description:  A column of asterisks on the left side,
             * with beginning and ending almost-blank lines.
             */
    
        For files in net/ and drivers/net/ the preferred style for long (multi-line)
        comments is a little different.
    
            /* The preferred comment style for files in net/ and drivers/net
             * looks like this.
             *
             * It is nearly the same as the generally preferred comment style,
             * but there is no initial almost-blank line.
             */
```

The code under the microscope is in `arch/x86/crypto` — not strictly part of the networking subsystem — but this code uses the style for `net/` and `drivers/net/` in at least one place. Herbert Xu, the crypto subsystem maintainer, [asserted](<http://article.gmane.org/gmane.linux.kernel.cryptoapi/20997>) that the crypto API uses the same style as networking, but Molnar wasn't convinced and neither, it turned out, was Linus Torvalds. I won't try to summarize [Torvalds's rant](<http://article.gmane.org/gmane.linux.kernel.cryptoapi/21066>) (which he promised he would not follow up on) but I will examine a concrete and testable assertion made by Molnar: ""That 'standard' is not being enforced consistently at all"".

Looking at the "`.c`" and "`.h`" files in linux 4.7-rc7 and using fairly simple regular expressions (which might have occasional false positives), the string "`/*`" appears 1,308,166 times, suggesting the presence of over 1.3 million comments. Of those, 981,168 are followed by "`*/`" on the same line, leaving 326,998 multi-line comments. 200,737 of these have nothing (apart from the occasional space) following the opening "`/*`" on the first line, and 51,366 start with "`/**`" which indicates a "kernel-doc" formatted comment, leaving 74,895 multi-line comments in the non-customary format with text on the first line. 

These three groups are present in a ratio of approximately 8:2:3. The kernel-doc comments have to be in the expected format to be properly functional, leaving the developer no discretion; it thus isn't reasonable to include them when looking at the choices developers have made. Of the multi-line comments where the programmer has some discretion, we find an 8:3 ratio of customary format, in the sense Molnar meant it, to others. So 27% are non-standard. 

If we repeat these measurements for `net/`, `drivers/net/`, `crypto/`, and `drivers/crypto/`, the number of non-standard multi-line comments are:

> Subsystem| Comments | Percent  
> ---|---|---  
> Total| Net-style  
> `net/` | 13,441 | 6,423 | 48%  
> `drivers/net/` | 36,599 | 19.516 | 54%  
> `crypto/` | 593 | 171 | 29%  
> `drivers/crypto/` | 706 | 178 | 25%  
  
So broadly, the evidence does seem to support Molnar's claim. While "text-on-the-first-line" comments are more common in the networking code, they just barely constitute a majority of multi-line comments there and they are not significantly more common in the crypto code. This statistic doesn't tell us a lot, but it does suggest that the supposed "preferred" style for the networking code is not consistently preferred in practice, and that sticking to it for new comments wouldn't actually improve the overall consistency of that code. 

Some of us may think this is all a storm in a teacup and that empty lines in comments, much like empty lines in code, are a matter of personal taste and nothing more. For many people this may be true. But open-source code will particularly benefit from being read by people who have a high attention for details, who will notice things that look a bit out of place, and who can spot bugs that compilers or static analyzers will miss. These people are likely to notice, and so be burdened by, irrelevant details like non-standard comments.

For Molnar at least, ""the networking code's 'exceptionalism' regarding the standard comment style is super distracting"" and there is evidence that he is not alone in this. To get the greatest value from other people reading our code, it makes sense to keep it as easy to read as possible. The code doesn't just belong to the author, it belongs to the community which very much includes those who will read it, whether to fix bugs, to write documentation, or as a basis for writing new drivers. We serve that community, and so indirectly ourselves, best when we make our code uniform and easy for others to read.
