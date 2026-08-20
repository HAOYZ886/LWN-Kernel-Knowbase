---
title: Hardening magic links
url: https://lwn.net/Articles/934460/
date: "June 14, 2023"
category: "Magic links; Security-Kernel hardening"
author: "By Jake Edge June 14, 2023 LSFMM+BPF"
---

> **Did you know...?**
> 
> LWN.net is a subscriber-supported publication; we rely on subscribers to keep the entire operation going. Please help out by [buying a subscription](<https://lwn.net/Promo/nst-nag4/subscribe>) and keeping LWN on the net. 

By **Jake Edge**  
June 14, 2023

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2023>)

There are some "magic links" in kernel pseudo-filesystems, like procfs, that can be—have been—(ab)used to cause security problems, such as a [container-confinement breach](<https://lwn.net/Articles/781013/>) in 2019\. Aleksa Sarai has long been working on ways to blunt the impact of these magic links. He led a filesystem session at the [2023 Linux Storage, Filesystem, Memory-Management and BPF Summit](<https://lwn.net/Articles/lsfmmbpf2023>) to discuss the status of those efforts. 

Sarai said that he worked on hardening for these links as part of [adding](<https://lwn.net/Articles/796868/>) the [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>) system call, but he removed some of that work before it was merged because the semantics were unclear. So, he wanted to have a discussion on those pieces to try to ensure that they make sense to everyone, that attendees are happy with them, and to avoid "having things thrown at me when I post them to the list". 

At this point, `openat2()` has been merged and he is still working on [`libpathrs`](<https://github.com/openSUSE/libpathrs>), which is a path-resolution library that allows those operations to be done safely in containers. The main thing he wanted to discuss was a draft patch for magic-link hardening, which lives on a [branch in his GitHub repository](<https://github.com/cyphar/linux/commits/magiclink/open_how-reopen>). 

A magic link looks like a symbolic link but is not one. They are described as follows in the [`symlink` man page](<https://man7.org/linux/man-pages/man7/symlink.7.html>) (under "Magic links"): 

> Unlike normal symbolic links, magic links are not resolved through pathname-expansion, but instead act as direct references to the kernel's own representation of a file handle. As such, these magic links allow users to access files which cannot be referenced with normal paths (such as unlinked files still referenced by a running program). 

Classic examples of magic links are `/proc/[PID]/exe` and `/proc/[PID]/fd/*`. They allow processes in containers to potentially see kernel objects that should not be accessible to them, for example. `openat2()` allows callers to disallow following these links with the `RESOLVE_NO_MAGICLINKS` flag, which can aid non-malicious programs, but the hardening he wants to add would go beyond that. 

[ ![\[Aleksa Sarai\]](https://static.lwn.net/images/2023/lsfmb-sarai-sm.png) ](<https://lwn.net/Articles/934468/>)

As with the vulnerability in 2019, a container process could get a reference to its container-runtime binary on the _host_ by way of `/proc/[PID]/exe`. It will not be able to write to that file while the runtime is running, but it can wait until the runtime isn't and then do so. He noted that people may be wondering why a container process has the rights to open a file for writing on the host, but that is, perhaps sadly, a requirement for today's container runtimes (such as Docker and Kubernetes), which run as root without any user namespaces, he said. 

Today's container runtimes "do a variety of awful things" to stop this attack. In particular, right now they all copy the binary to an anonymous file created with [`memfd_create()`](<https://man7.org/linux/man-pages/man2/memfd_create.2.html>) every time a container is created; the memfd is then [sealed](<https://lwn.net/Articles/593918/>). The end result is that "even if you can overwrite the damn thing, it won't affect other containers in the system". He thinks that everyone agrees that "this is all absolutely awful and should not exist", but it is unfortunately needed. He wants to solve the problem in the kernel and he believes that a general ability to restrict file reopening would also be useful, so that is part of his patch set as well. 

The core of the patch set is that it will only allow reopens of magic links if the mode being requested is a subset of the mode set on the magic link file handle in the kernel. It would also add an `O_EMPTYPATH` flag to [`openat()`](<https://man7.org/linux/man-pages/man2/openat.2.html>) (and `openat2()`) that allows the passed-in directory file descriptor to be used as the file descriptor of a file to be reopened. It would provide a mask mechanism to restrict reopen modes that can be specified at the time a file is opened with `openat2()`. Lastly, it would expose the reopening restrictions for files in `/proc/[PID]/fdinfo/*`. 

He gave some further details of what it means to have to be a subset of the mode. The `O_PATH` flag to `open()` and friends simply requests a descriptor of the path of the file—it does not actually open the file itself. Assuming that no mask has been placed on a file, an `O_PATH` reopen of a regular file will allow any legal mode to be used; this is how things work today and that would not change. But for a magic link, which has its own "magic modes" that are different than those for regular files, an `O_PATH` reopen will copy the mode of the existing open file. Other kinds of opens (or reopens), like `O_RDWR` for read and write, will be handled in the usual way. All of the modes for reopens are based on the `f_mode` in the kernel [`struct file`](<https://elixir.bootlin.com/linux/v6.3.7/source/include/linux/fs.h#L942>) entry. 

He wanted to know if those restrictions made sense. He believes they do, though there are some corner cases that need to be considered, but it does protect against the problem he is trying to solve. He also wanted to consider future-proofing the design, which might mean figuring out how directories fit into it as well. 

David Howells asked if it made sense to add a separate system call just for reopen operations, but Sarai said that would not help. Lots of code is using the existing system calls to reopen files and those are not going away. By and large, reopen is not being used nefariously, in fact, container runtimes "don't just use this, we abuse it to hell and back", because they must. There are "certain security properties you cannot get without using it", he said. 

Amir Goldstein asked if [`chmod()`](<https://man7.org/linux/man-pages/man2/chmod.2.html>) could be used to change the discretionary access control (DAC) permissions on the files, instead. But Al Viro pointed out, via the remote-access audio, that mode bits for symbolic links are completely ignored. Goldstein wondered if they could be made to matter for magic symlinks, then `chmod()` could be used to control access to the links. Viro did not think that was feasible. He pointed out that anytime `/dev/stdin` is opened, it actually resolves to `/proc/self/fd/0`, so the behavior of the magic links cannot be changed without "breaking the living hell" out of lots of different things. 

Christian Brauner agreed that backward-compatibility is important. There are `O_PATH` opens in lots of other places at this point, for example in the pseudo-terminal (pty) handling. People regularly propose "fixes" for the `/proc/self/exe` problem because the current solution is not pleasant, so he thinks it makes sense to use Sarai's mechanism, make it work well, and head off further hacky fixes. 

Viro asked what would happen if someone were to bind-mount to the location where `/proc/self/exe` points and then reopen via that path for write. Sarai agreed that was a problem, and one that is worth addressing, but as a practical matter for containers, it is not a problem because nearly all containers cannot do the bind-mount in question. Sarai noted that the `/proc/self/exe` attack is a problem because 99.99...% of containers are running as root and do not employ user namespaces. Brauner said that user namespaces are not a panacea, but they do block the problem with containers overwriting the runtime binary. 

Sarai went through some of the problems with handling directories at a rapid pace, then shifted into restricting the execution of files. Right now, there is no way to restrict a file handle such that it cannot be used to execute the file contents using [`fexecve()`](<https://man7.org/linux/man-pages/man3/fexecve.3.html>); the DAC permissions can be used to restrict it, but once the file is open, the file descriptor cannot be passed to an untrusted process with execution blocked. The same goes for directories; you cannot restrict path resolution from an open directory file descriptor. Even if those things do not get implemented, the design of the restrictions that he is implementing should take those potential use cases into account. 

Viro said that, currently, the write-permission bits on a directory do not affect whether files in that directory can be written and wondered if Sarai was suggesting changing the meaning of the directory permission bits in some fashion. Sarai said that he was not; if these changes were implemented, an `O_PATH` open of the directory could set its mask such that writing is not allowed, so another process would not be able to create a directory or regular file there using that `O_PATH` descriptor. Howells likened it to an access-control list (ACL) governing what could be done using the `O_PATH` descriptor. 

Viro expressed skepticism about changing the behavior for directories in that fashion, but Sarai pointed out that it was effectively opt-in; those who want to do this would need to set the mode mask on the `O_PATH` file descriptor before passing it onward. Viro asked about bind-mounts and Sarai once again agreed that they are a problem, though the vast majority of containers are run in a mount namespace so that they are unable to create the mount in question. Which is not to say that he does not believe the problem should be solved, however. 

Another question that Sarai had was about mounting on top of symbolic links, which works today; there is no way to restrict mounting on top of magic links, which is even messier. But Viro said the kernel should restrict mounts from locations in `/proc/[PID]/*` using the "no mounts here" inode flag. "I cannot tell you how happy I am to hear that", Sarai said, claiming he would write the patch as soon as he left the room. There was a bit more discussion of that as the session ran out of time, but it would seem to resolve many of the concerns Sarai had about mounting on magic links.
