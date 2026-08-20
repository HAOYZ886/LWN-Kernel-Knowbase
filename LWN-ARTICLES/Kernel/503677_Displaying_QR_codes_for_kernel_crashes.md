---
title: Displaying QR codes for kernel crashes
url: https://lwn.net/Articles/503677/
date: "June 27, 2012"
category: "Debugging; Development tools-Kernel debugging"
author: "By Jake Edge June 27, 2012"
---

> **Please consider subscribing to LWN**
> 
> Subscriptions are the lifeblood of LWN.net. If you appreciate this content and would like to see more of it, your subscription will help to ensure that LWN continues to thrive. Please visit [this page](<https://lwn.net/Promo/nst-nag1/subscribe>) to join up and keep LWN on the net. 

By **Jake Edge**  
June 27, 2012

A [proposal](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000400.html>) from Cong Wang to discuss the various mechanisms to store the kernel's "dying breath" spawned a rather large thread on the ksummit-2012-discuss mailing list. While things like [pstore](<https://lwn.net/Articles/434821/>) were set up specifically to provide a means to store kernel crash information, that doesn't necessarily make it easy for users to access and report kernel crashes. That led to suggestions and discussion of better ways for users to get the information out of their crashed systems--including using [QR codes](<http://en.wikipedia.org/wiki/Qr_code>) to facilitate the process. 

Most regular users do not have a serial console set up to record crash information on a separate machine. So the kernel backtrace that appears after a crash is just written to the console, which means that much of it will have scrolled off the screen. Even the data that is there is hard to extract, with some folks trying to type the information in, which is tedious, not to mention error-prone. A QR code that encoded the relevant data could certainly help there. 

Konrad Rzeszutek Wilk was the first to [broach the QR code idea](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000443.html>), though he said it did not originate with him. It turns out that H. Peter Anvin and Dirk Hohndel have been "[messing with](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000494.html>)" the idea, but Will Deacon and Marc Zyngier actually [showed](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000564.html>) something along those lines at the recent Linaro Connect in Hong Kong. Deacon was [hesitant to call it a prototype](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000581.html>), but said that there was some work done on encoding a kernel crash backtrace as a QR code. There were two problems with their approach: 

  1. Even without any error correction, the QR code started to get pretty large (and unreadable) after more than a few lines of backtrace. This should be fairly easy to fix by encoding the data in a more sensible manner rather than just verbatim (especially since a backtrace is a well-structured log). Maybe you could even gzip the whole thing after that too (then sell an android app to gunzip it :p) 
  2. Displaying the QR code on a panic could be problematic. We tried using the ASCII option of libqrencode but we couldn't find any phone that would read the result. So we need a way to get to the framebuffer once we've sawn our head off (maybe this is easier with x86 and VGA modes?). 

One of the original [motivations](<https://lwn.net/Articles/218380/>) for kernel modesetting (KMS) was to get readable oops information to the screen. Using KMS to display a fairly simple QR code graphic instead should be workable, rather than creating an ASCII version as Deacon describes. Matthew Garrett [noted](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000557.html>) that it should be fairly straightforward at least for hardware that has KMS support: 

KMS already has atomic modeswitch support for showing panics. We'd just need to ensure that there's an unaccelerated path for dumping contents directly to the framebuffer. If you don't have KMS then you don't get to play with modern useful functionality. 

There is some disagreement about where the decoding of any QR code should take place. Garrett [believes](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000590.html>) that existing QR apps in phones should be used, while others are not convinced they can be coerced into being flexible enough to deal with the large QR codes that might result from a kernel backtrace. Garrett has also done some work on the problem and [described](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000459.html>) his approach: 

Basic design was as follows: Take the backtrace, compress it, encode in an alphanumeric QR code including an http:// prefix, submit to http://kbu.gs/blah automatically when user takes a picture 

Anvin would [rather see](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000596.html>) some kind of web application that accepts a photo of the QR code and decodes it on the server. For one thing, having one (working) decoding code base is desirable: ""I can tell you just how bad a lot of the QR decoder software running on smartphones are -- because I have tried them."" In addition, though, a web application would also have the photo itself, so even if it didn't decode because of picture quality or other reasons, those photos could be used to improve the quality of the decoder. 

But that implies that a user would need to download an app to their phone or use some web application as [suggested](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000603.html>) by John Hawley. Garrett was [not in favor of either solution](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000604.html>), noting that requiring an app makes its harder for users, while a web application doesn't really make it any better: 

And now your workflow is "Take picture, move to browser, upload, wait to see if it decodes, back to camera, back to browser", etc. I know we're expected to be bad at UX here, but come on. 

Given that many users already use photos to report crashes--taking a picture of the screen with the last part of the backtrace--the QR code mechanism, even if a bit cumbersome, might be able to provide the full backtrace. But, as Dave Jones [suggested](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000607.html>), just having scrollback available on the console after a crash would make much of the problem disappear: ""What would be a thousand times more useful would be having working scrollback when we panic, like we had circa 2.2"". 

Users could then take a photo, scroll back a ways, take another, and so on. In the thread, there was widespread agreement that console scrollback would be desirable. But it turns out that the advent of USB keyboards caused the loss of that feature. Doing USB handling inside the panic code would be [messy](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000681.html>), so bringing that feature back is difficult. Other ideas were mentioned, like providing enough of the USB stack to write the crash information to a USB stick as Anvin suggests, or to "auto-scroll" the console output after a crash without requiring keyboard input as [proposed](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000683.html>) by Paul Gortmaker. 

Making it easier for users to report crashes with useful information was one branch of the discussion, but the folks who work on the embedded side are looking for more developer-oriented solutions as well. Tony Luck [outlined](<http://lists.linux-foundation.org/pipermail/ksummit-2012-discuss/2012-June/000563.html>) the pstore back-ends that are currently available to store crash and other information in various places (ERST, EFI variables, RAM) that are accessible after a reboot. Wang, Tim Bird, Jason Wessel, and others are interested in discussing that piece of the puzzle. 

While QR codes may seem like something of gimmick, they can compress a fair amount of data into a form that can be digested elsewhere. Getting useful information out of an unresponsive, crashed Linux system is fairly difficult at this point, so finding better ways to do so would be good. Should the program committee decide to add this topic, a lively discussion seems likely. If not, though, enough people are looking into the idea that something will emerge sooner or later.
