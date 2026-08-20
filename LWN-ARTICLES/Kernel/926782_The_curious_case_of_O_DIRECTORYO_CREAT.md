---
title: "The curious case of O_DIRECTORY|O_CREAT"
url: https://lwn.net/Articles/926782/
date: "March 27, 2023"
category: "Development model-User-space ABI; System calls-open"
author: "By Jonathan Corbet March 27, 2023"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jonathan Corbet**  
March 27, 2023

The [`open()`](<https://man7.org/linux/man-pages/man2/openat.2.html>) system call offers a number of flags that modify its behavior; not all combinations of those flags make sense in a single call. It turns out, though, that the kernel has responded in a surprising way to the combination of `O_CREAT` and `O_DIRECTORY` for a long time. After a 2020 change made that response even more surprising, it seems likely that this behavior will soon be fixed, resulting in a rare user-visible semantic change to a core system call. 

The `O_CREAT` flag requests that `open()` create a regular file if the named path doesn't exist (adding `O_EXCL` will cause the call to fail if the path _does_ exist). `O_DIRECTORY`, instead, indicates that the call should only succeed if the path exists and is a directory. It is not possible to create a directory with `open()`; that is what [`mkdir()`](<https://man7.org/linux/man-pages/man2/mkdir.2.html>) is for. So the combination of `O_CREAT` and `O_DIRECTORY` requests the kernel to create a directory (which is supposed to already exist) as a regular file — which clearly does not make sense. 

Since time immemorial, the kernel's response to the combination of those two flags has been to flag an error in most situations. If the path exists and is a regular file, `open()` fails and returns with an `ENOTDIR` error. If, instead, the path is an existing directory, the error is `EISDIR` — perhaps a bit surprising, given that `O_DIRECTORY` indicates that the path is expected to be a directory. If, however, the path does not exist at all, the `open()` call will succeed after creating a regular file with the indicated name, which is also a surprising result. 

Recently, though, Pedro Falcato [noticed](<https://lwn.net/ml/linux-kernel/20230320071442.172228-1-pedro.falcato@gmail.com/>) that the behavior in the final case above had changed; the kernel will now return `ENOTDIR` if the path does not exist — but it also still creates a regular file. It is fair to say that this behavior is even more surprising than what happened before. Christian Brauner [tracked the behavioral change down](<https://lwn.net/ml/linux-kernel/20230320115153.7n5cq4wl2hmcbndf@wittgenstein/>) to [this commit from Al Viro](<https://git.kernel.org/linus/973d4b73fbaf>), which was merged for the 5.7 release. 

Falcato included a patch to restore the previous behavior, which arguably makes a bit more sense than what the kernel does now and is, in any case, what the kernel did for a long time. But Brauner wondered if the right thing to do was to fix the kernel to do something more rational with that combination of flags: 

> So before we continue down that road should we maybe treat this as a chance to fix the old bug? Because this behavior of returning -ENOTDIR has existed ever since v5.7 now. Since that time we had three LTS releases all returning ENOTDIR even if the file was created. 

Since, he said, nobody seems to have noticed the change over this time, it seems likely that nobody is actually counting on the strange semantics given to that combination of flags in the past. Linus Torvalds [agreed](<https://lwn.net/ml/linux-kernel/CAHk-=wjifBVf3ub0WWBXYg7JAao6V8coCdouseaButR0gi5xmg@mail.gmail.com/>) that actually fixing the kernel's behavior seemed like a sensible path: ""I think we can pretty much assume that there are no actual users of it, and we might as well clean up the semantics properly"". 

Falcato [did some research](<https://lwn.net/ml/linux-kernel/CAKbZUD2Y2F=3+jf+0dRvenNKk=SsYPxKwLuPty_5-ppBPsoUeQ@mail.gmail.com/>) on what other systems do in response to that combination of flags. NetBSD, it seems, will simply fail an `open()` call in that situation, returning `EINVAL`. FreeBSD, instead, will allow the call to succeed if the path exists and is a directory; otherwise it will fail. He also noted that all of the behaviors seen — Linux pre- and post-5.7, NetBSD, and FreeBSD — are allowed by POSIX: ""I would not call the old Linux behavior a *bug*, just really odd semantics"". 

Torvalds [answered](<https://lwn.net/ml/linux-kernel/CAHk-=wgc9qYOtuyW_Tik0AqMrQJK00n-LKWvcBifLyNFUdohDw@mail.gmail.com/>) that either of the BSD behaviors would make sense, while the kernel's current behavior ""has no excuse"". The NetBSD response is ""the clearest case"", he said, but FreeBSD's behavior is closer to what Linux did before the 5.7 change. Brauner [favored](<https://lwn.net/ml/linux-kernel/20230321142413.6mlowi5u6ewecodx@wittgenstein/>) the NetBSD behavior, and put together [a patch](<https://lwn.net/ml/linux-kernel/20230321161736.njmtnkvjf5rf7x5p@wittgenstein/>) to implement it. As part of that work, he put some effort into searching through code looking for cases that would be broken by the change in semantics; he came up nearly empty: 

> Time was spent finding potential users of this combination. Searching on codesearch.debian.net showed that codebases often express semantical expectations about O_DIRECTORY | O_CREAT which are completely contrary to what our code has done and currently does. 
> 
> The expectation often is that this particular combination would create and open a directory. This suggests users who tried to use that combination would stumble upon the counterintuitive behavior no matter if pre-v5.7 or post v5.7 and quickly realize neither semantics give them what they want 

Included in the patch are some links to places where developers had attempted this combination; see [this libglnx comment](<https://sources.debian.org/src/flatpak-builder/1.2.3-1/subprojects/libglnx/glnx-shutil.c/?hl=251#L251>) for an example. 

As the result of Brauner's patch, the combination of `O_CREAT` and `O_DIRECTORY` will cause an `open()` call to fail with `EINVAL` regardless of whether the given path exists or not. Chances are that nothing will break with this change, but he is asking for widespread testing to be sure of that. It would, after all, be annoying to have to revert this change if a problem report surfaces at some point in the future. The patch has not actually been applied as of this writing; given that there is a semantic change involved, it would be a bit surprising to see it land for 6.3. That said, your editor has been surprised by such things before. 

This is one of those cases where the subtleties in the kernel's API policies come into play. In a real sense, this fix is an incompatible API change, and it will indeed break any program that is relying on the current behavior. But, in cases where no program does rely on a specific behavior, that behavior can indeed be changed. This fix seems unlikely to break anything, and so is permissible for the kernel developers to do. Should the assumption that nothing will break prove true, it may even be possible, someday, to make that flag combination do what developers evidently expect and create a directory. But first it is necessary to demonstrate that there are indeed no problems resulting from the removal of the current, strange semantics.
