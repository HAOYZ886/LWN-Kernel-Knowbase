---
title: A tale of two release cycles
url: https://lwn.net/Articles/232492/
date: "May 1, 2007"
category: "Development model-Kernel quality"
author: 
---

As most LWN readers will be aware, the 2.6.21 kernel has been released. The 2.6.21 process was relatively difficult, mostly as a result of the core timer changes which went in. These changes were necessary - they are the path forward to a kernel which works better on all types of hardware - but they caused some significant delays in the release of the final 2.6.21 kernel. Even at release time, this kernel was known not to be perfect; there were a dozen or so known regressions which had not been fixed. 

The reason we know about these regressions is that Adrian Bunk has been tracking them for the past few development cycles. Mr. Bunk has [let it be known](<http://lwn.net/Articles/231993/>) that he will not be doing this tracking for future kernels. From his point of view, the fact that the kernel was released with known regressions means that the time spent tracking them was wasted. Why bother doing that work if it doesn't result in the tracked problems being fixed? 

What Mr. Bunk [would like to see](<https://lwn.net/Articles/232494/>) is a longer stabilization period: 

There is a conflict between Linus trying to release kernels every 2 months and releasing with few regressions. Trying to avoid regressions might in the worst case result in an -rc12 and 4 months between releases. If the focus is on avoiding regressions this has to be accepted. 

Here is where one finds the fundamental point of disagreement. The kernel used to operate with long release cycles, but the "stable" kernels which emerged at the end were not particularly well known for being regression free. Downloading and running an early 2.4.x kernel should prove that point to anybody who doubts it. 

The reasoning behind the current development process (and the timing of the 2.6.21 release in particular), [as stated by Linus Torvalds](<https://lwn.net/Articles/232495/>) is: 

Regressions _increase_ with longer release cycles. They don't get fewer.. This simply *does*not*work*. You might want it to work, but it's against human psychology. People get bored, and start wasting their time discussing esoteric scheduler issues which weren't regressions at all. 

In other words, holding up a release for a small number of known bugs prevents a much larger set of fixes, updates, new features, additional support, and so on from getting to the user base. Meanwhile, the developers do not stop developing, and the pile of code to be merged in the next cycle just gets larger, leading to even more problems when the floodgates open. It would appear that most kernel developers believe that it is better to leave the final problems for the stable tree and let the development process move on. 

The 2.6.21 experience might encourage a few small changes; in particular, Linus has [suggested](<https://lwn.net/Articles/232497/>) that truly disruptive changes should maybe have an entire development cycle to themselves. As a whole, however, the process is not seen as being broken and is unlikely to see any big "fixes." 

For an entirely different example, let us examine the process leading to the Emacs 22 release. Projects managed by the Free Software Foundation have never been known for rapid or timely releases, but, even with the right expectations in place, this Emacs cycle has been a long one: the previous major release (version 21) was [announced](<http://www.gnu.org/press/2001-10-22-Emacs.html>) in October, 2001. In those days, LWN was talking about the 2.4.11 kernel, incorporation of patented technology into W3C standards, the upcoming Mozilla 1.0 release, and the Gartner Group's characterization of Linux as a convenient way for companies to negotiate lower prices from proprietary software vendors. Things have moved on a bit since those days, but Emacs 21 is still the current version. 

The new Emacs major release [was recently scheduled for April 23](<http://lwn.net/Articles/229825/>), but it has not yet happened. There is one significant issue in the way of this release: it seems that there is a cloud over some of the code which was merged into the Emacs Python editing mode. Until this code is either cleared or removed, releasing Emacs would not be a particularly good idea. It also appears that the wisdom of shipping a game called "Tetris" has been questioned anew and is being run past the FSF's lawyers. 

Before this issue came up, however, the natives in the Emacs development community were getting a little restless. Richard Stallman may not do a great deal of software development anymore, but he is still heavily involved in the Emacs process. Emacs is still his baby. And this baby, it seems, will not be released until it is free of known bugs. This approach is distressing for Emacs developers who would like to make a release and get more than five years' worth of development work out to the user community. 

[This message](<https://lwn.net/Articles/232501/>) From Emacs hacker Chong Yidong is worth quoting at length: 

To be fair, I think RMS' style of maintaining software, with long release cycles and insistence on fixing all reported bugs, was probably a good approach back in the 80s, when there was only a handful of users with access to email to report bugs. 

Nowadays, of course, the increase in the number of users with email and the fact that Emacs CVS is now publicly available means that there will always be a constant trickle of bug reports giving you something to fix. Insisting---as RMS does---on fixing all reported bugs, even those that are not serious and not regressions, now means that you will probably never make a release. 

It has often been said that "perfect" is the enemy of "good." That saying does seem to hold true when applied to software release cycles; an attempt to create a truly perfect release results in no release at all. Users do not get the code, which does not seem like a "perfect" outcome to them. 

Mr. Yidong has another observation which mirrors what was said in the kernel discussion: 

There is also a positive feedback loop: RMS' style for maintaining Emacs drives away valuable contributors who feel their effects will never be rewarded with a release (and a release is, after all, the only reward you get from contributing to Emacs). 

It's not only users who get frustrated by long development cycles; the developers, too, find them tiresome. Projects which adopt shorter, time-based release cycles rarely seem to regret the change. It appears that there really are advantages to getting the code out there in a released form. Your editor is not taking bets on when Emacs might move to a bounded-time release process, though.
