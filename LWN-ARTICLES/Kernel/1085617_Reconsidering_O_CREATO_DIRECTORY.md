---
title: "Reconsidering O_CREAT|O_DIRECTORY"
url: https://lwn.net/Articles/1085617/
date: "July 30, 2026"
category: "Development model-User-space ABI; System calls-open"
author: "By Jonathan Corbet July 30, 2026"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 30, 2026

Linux provides a system call ([`mkdir()`](<https://man7.org/linux/man-pages/man2/mkdir.2.html>)) to create a directory, and [a few variants of `open()`](<https://man7.org/linux/man-pages/man2/open.2.html>) that can open a directory. There is, however, no system call in Linux that can create and open a directory in a single, race-free call. Jori Koolstra has been working on remedying that situation, most recently by repurposing a set of `open()` flags that currently return an error. There are, however, concerns that show just how hard it can be to create user-space interfaces that do not present traps for application developers. 

Creating and opening a directory in a single system call is simpler and more efficient than using two, of course. It also can guard against the possibility that some other process will, between the creation and open steps, replace a directory with something else. Detecting that case _is_ possible on Linux now, but it requires some defensive programming of the type that application developers are not always good at. Thus the desire for a more straightforward way to accomplish that pair of operations. 

In March, Koolstra attempted to address this problem with [a patch series](<https://lwn.net/ml/all/20260331172011.3512876-1-jkoolstra@xs4all.nl/>) adding a new system call, `mkdirat_fd()`, that would return a file descriptor for the newly created directory. That system call was changed to `mkdirat2()` in [a subsequent patch](<https://lwn.net/ml/all/20260412135434.3095416-1-jkoolstra@xs4all.nl>). There were a number of concerns about the implementation, but also about creating a new system call in the first place. Christian Brauner, in particular, [thought](<https://lwn.net/ml/all/20260511-hochdekoriert-neoliberale-f7a2922bc57c@brauner>) that this problem was better solved with a modification to how `open()` handles a couple of existing flags. 

Specifically, all of the `open()` variants support the `O_DIRECTORY` flag, which is necessary if the application is trying to open a directory rather than a normal file. Also supported is `O_CREAT`, which instructs `open()` to create the named file if it does not already exist. It would make sense to interpret the combination of those two flags as a request to create a directory and open it at the same time. Adding this capability to `open()`, Brauner said, would also make many of the other features of the system call, such as the ability to place restrictions on how the name is resolved, available for free. So, he concluded: ""I think here it is pretty clear that O_DIRECTORY|O_CREAT is the right thing to do"". 

Koolstra duly implemented the new API as [an RFC patch set](<https://lwn.net/ml/all/20260517170244.1832119-1-jkoolstra@xs4all.nl>), followed by three more RFC and three non-RFC versions; it was only after [the last of those](<https://lwn.net/ml/all/20260704164149.3480051-1-jkoolstra@xs4all.nl>) was posted that other developers started to take a serious look at the proposed API, and not all of them were convinced that it was the right approach. Pedro Falcato, in particular, [argued](<https://lwn.net/ml/all/akyyqnv9EpY7xVdC@pedro-suse.lan>) that this API would be nearly impossible for applications to use in a portable way, given the number of different ways that `O_CREAT|O_DIRECTORY` has been interpreted in the past. 

That story is, indeed, a bit complicated; LWN [covered it](<https://lwn.net/Articles/926782/>) in 2023. The ways in which Linux has handled an `open()` call with that flag combination include: 

  * Older kernels would fail if the named file existed, returning `ENOTDIR` if it is a regular file and `EISDIR` if it is a directory. If, instead, nothing existed by the given name, `open()` would create a regular file, which seems unlikely to be what the caller wanted. 
  * As of the 5.7 release in 2020, the kernel started returning an error in the last case, while _still creating the file_ , which seemed even less likely to be what the developer was hoping for. 
  * Brauner added [a patch](<https://git.kernel.org/linus/43b450632676>) to 6.4 causing the kernel to fail with `EINVAL` for that flag combination in all cases. 

Add in the fact that other systems supporting POSIX system calls have their own interpretation of that flag combination, Falcato said, and the result is going to be difficult to use properly: 

> If you're writing something that wants to be portable, or that intends to standardize on something, you have some 5 different behaviors all across the FOSS UNIX landscape, not considering everything else. It's also something that will, FWIW, probably never be included in POSIX because no one can agree on the semantics here. 
> 
> TL;DR I don't see, given the reasons above, how users are supposed to use this without it being a total minefield. 

He suggested that limiting the change to `openat2()` might be a better approach. 

Brauner, though, [dismissed](<https://lwn.net/ml/all/20260707-vorteil-unhaltbar-apotheke-47bb105161ff@brauner>) the concern, saying: ""I don't think any of this is really an argument worth considering"". The behavior of that flag combination has been consistent on Linux for years, he said, so it makes sense to make better use of it now. Falcato [pointed out](<https://lwn.net/ml/all/akznq3Tzkg3XbPAc@pedro-suse>) that the older LTS kernels never received the 6.4 change and, as a result, do not have consistent behavior, and that user space would still have to perform ""some pretty gnarly tests"" to use the feature safely. Brauner [suggested](<https://lwn.net/ml/all/20260707-latschen-positionieren-akkord-314a884b8868@brauner>) backporting the 6.4 fix, and added: ""Feature testing has always been a pain that we forced onto userspace. I think we should continue with that proud tradition"". 

Christoph Hellwig [disagreed](<https://lwn.net/ml/all/alS10NN_OduZEqvt@infradead.org>): 

> We need an interface that is self discoverable on Linux. Any combination of flags that was accepted by previous kernels and gave different results than the new interface do not qualify for that. 

He added that it was not possible to rely on a fix being backported to all of the old kernels in use. 

Neil Brown [suggested](<https://lwn.net/ml/all/178397852844.3371781.6900199568039042534@noble.neil.brown.name>) adding a new flag, `OPENAT2_NEW_COMBINATION`, that would cause `openat2()` to reject any flag combination that is not recognized. Any older kernels will immediately fail a call with that flag; new kernels can use it to enable combinations that were not previously supported, including `O_CREAT|O_DIRECTORY`. Brauner, though, once again [dismissed](<https://lwn.net/ml/all/20260722-mundpropaganda-vorwahl-funkverkehr-48efdde92a00@brauner>) Hellwig's concern as ""a strawman"". The fact that he was able to make that flag combination return an error without generating a single regression report, he said, was evidence that there is no risk to adding meaning to it now. Hellwig [disagreed](<https://lwn.net/ml/all/amDBQN5lCGbRxGWm@infradead.org>), saying that, once the combination is supported, applications will start using it, and they will break on older systems. It is, he said, necessary to add ""APIs that don't accidentally do the wrong thing on any old kernel"". 

Koolstra had [a different concern](<https://lwn.net/ml/all/1713122837.215334.1784765444011@kpc.webmail.kpnmail.nl>) about the `openat2()` suggestion: seemingly a lot of [`seccomp()`](<https://man7.org/linux/man-pages/man2/seccomp.2.html>) configurations still block `openat2()`. That could make the new feature unavailable on a lot of systems, reducing its value. 

Toward the end of the discussion (so far), Brauner [asked](<https://lwn.net/ml/all/20260724-jazzband-benannt-bauzeit-08186a6e1986@brauner>) Koolstra to send a version with support in `openat()` and `openat2()`. He did not say whether the change to `open()` should remain, but complained that ""we're deliberately crippling a useful extension for userspace"". The [latest version](<https://lwn.net/ml/all/20260712175539.1565444-1-jkoolstra@xs4all.nl>) of the series from Koolstra enables that flag combination for all `open()` variants without any additional checks. Brauner has not yet applied that series, so we do not yet know if it will find its way into the mainline in that form or not. 

Maintaining compatibility over the long term is not an easy task. It is hard enough to ensure that new kernels do not break older applications, but it can often be trickier to avoid breaking newer applications on older kernels. One of the key ways to do that is to provide ways for applications to discover whether a given feature is supported by the kernel or not. The disagreement here is over whether the proposed feature provides that discovery mechanism, with developers like Falcato and Hellwig saying "no", while Brauner feels that it is discoverable on all kernels that actually matter. There is only one chance to make the correct decision; once the new feature is exposed in a kernel release, it will be difficult to change thereafter.
