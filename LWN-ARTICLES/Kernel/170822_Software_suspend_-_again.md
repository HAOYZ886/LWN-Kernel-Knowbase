---
title: "Software suspend - again"
url: https://lwn.net/Articles/170822/
date: "February 6, 2006"
category: Crash dumps; Software suspend
author: 
---

[Last week's Kernel Page](<http://lwn.net/Articles/169269/>) looked at one small piece of the software suspend debate. Meanwhile, the wider discussion has flared up yet again, and looks unlikely to slow down. Developers of the in-kernel suspend-to-disk code are working on moving parts of it to user space and generally tweaking the existing structure. Nigel Cunningham and other supporters of the Suspend2 patches, instead, still hope to see that work merged, eventually replacing much of the existing implementation. The discussion does not appear to be nearing any sort of resolution. 

One has become clear, though: Pavel Machek has a firm grip on the current in-tree swsusp code, and that puts Suspend2 at a significant disadvantage. Pavel has taken a strong position against many aspects of the Suspend2 code, and seems determined that it will never be merged. One gets the sense, sometimes, that he just wishes Nigel and his code would go away. Nigel is somewhat more persistent than that, however. 

At one point, the two suggested that Linus and Andrew should make a decision between the two implementations and settle the debate. Andrew, however, [does not want to do that](<https://lwn.net/Articles/170825/>): 

You're unlikely to hear anything dispositive from either of us on this... What we hope and expect is that you'll come up with an agreed path in accordance with general kernel coding and development principles. Linus and I don't want to have to make tiebreak decisions - if we have to do that, the system has failed. 

So much for the easy solution. Since then, the relevant parties have been talking, but without a whole lot of apparent progress. 

Perhaps the more interesting part of Andrew's note, however, was this: 

If you want my cheerfully uninformed opinion, we should toss both of them out and implement suspend3, which is based on the kexec/kdump infrastructure. There's so much duplication of intent here that it's not funny. 

`kexec()`, remember, is a relatively new system call used to boot from one kernel directly into another without going through the whole BIOS startup ritual. The kdump code uses `kexec()` to perform safe crash dumps. When the kernel panics, it uses `kexec()` to boot into a small, special-purpose kernel which has been lurking in a reserved part of memory for just this occasion. The new kernel restricts itself to the reserved memory, so the entire memory image of the old, crashed kernel remains intact. That image can then be written to disk in a relatively safe manner. 

It is true that suspend-to-disk can be thought of as a sort of kernel dump; the only difference is this little desire to be able to restart the kernel from the dump image at a future time. Using kdump for suspend-to-disk has some obvious appeal. A great deal of effort now goes into freezing most processes on the system - but not the ones needed to complete the suspend process. The suspend code also must be very careful about what kernel state it changes as it goes about its work. Simply jumping into a separate dump kernel has the potential to make many of those problems go away. It might almost be like the Good Old Days, when BIOS-based suspend code simply worked most of the time. 

A kdump-based suspend would not be without its costs. In particular, some people might balk at reserving a substantial chunk of memory for the suspend kernel. And, of course, the entire idea remains vaporware for now. 

Andrew's suggestion generated little discussion on the mailing list. But, just maybe, it will have ignited a gleam in some hacker's eye. A simpler, more robust suspend mechanism based on kdump which appeared out of left field might just solve this problem - and put the whole tiresome debate in the past - for good.
