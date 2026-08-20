---
title: "Ghosts of Unix past, part 2: Conflated designs"
url: https://lwn.net/Articles/412131/
date: "November 4, 2010"
category: "Development model-Patterns"
author: "November 4, 2010 This article was contributed by Neil Brown"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

November 4, 2010

This article was contributed by Neil Brown

In [the first article in this series](<http://lwn.net/Articles/411845/>), we commenced our historical search for design patterns in Linux and Unix by illuminating the "Full exploitation" pattern which provides a significant contribution to the strength of Unix. In this second part we will look at the first of three patterns which characterize some design decisions that didn't work out so well. 

The fact that these design decisions are still with us and worth talking about shows that their weaknesses were not immediately obvious and, additionally, that these designs lasted long enough to become sufficiently entrenched that simply replacing them would cause more harm than good. With these types of design issues, early warning is vitally important. The study of these patterns can only serve if they help us to avoid similar mistakes early enough. If they only allow us to classify that which we cannot avoid, there would be little point in studying them at all. 

These three patterns are ordered from the one which seems to give most predictive power to that which is least valuable as an early warning. But hopefully the ending note will not be one of complete despair - any guidance in preparing for the future is surely better than none. 

### Conflated Designs

This week's pattern is exposed using two design decisions which were present in early Unix and have been followed by a series of fixes which have address most of the resulting difficulties. By understanding the underlying reason that the fixes were needed, we can hope to avoid future designs which would need such fixing. The first of these design decisions is taken from the implementation of the single namespace discussed in part 1\. 

#### The mount command

The central tool for implementing a single namespace is the 'mount' command, which makes the contents of a disk drive available as a filesystem and attaches that filesystem to the existing namespace. The flaw in this design which exemplifies this pattern is the word 'and' in that description. The 'mount' command performs two separate actions in one command. Firstly it makes the contents of a storage device appear as a filesystem, and secondly it binds that filesystem into the namespace. These two steps must always be done together, and cannot be separated. Similarly the unmount command performs the two reverse actions of unbinding from the namespace and deactivating the filesystem. These are, or at least were, inextricably combined and if one failed for some reason, the other would not be attempted. 

It may seem at first that it is perfectly natural to combine these two operations and there is no value in separating them. History, however, suggests otherwise. Considerable effort has gone into separating these operations from each other. 

Since version 2.4.11 (released in 2001), Linux has a 'lazy' version of unmount. This unbinds a filesystem from the namespace without insisting on deactivating it at the same time. This goes some way to splitting out the two functional aspects of the original unmount. The 'lazy' unmount is particularly useful when a filesystem has started to fail for some reason, a common example being an NFS filesystem from a server which is no longer accessible. It may not be possible to deactivate the filesystem as there could well be processes with open files on the filesystem. But at least with lazy unmounted it can be removed from the namespace so new processes wont be able to try to open files and so get stuck. 

As well as 'lazy' unmounts, Linux developers have found it useful to add 'bind' mounts and 'move' mounts. These allow one part of the name space to be bound to another part of the namespace (so it appears twice) or a filesystem to be moved from one location to another -- effectively a 'bind' mount followed by a 'lazy' unmount. Finally we have a `pivot_root()` system call which performs a slightly complicated dance between two filesystem starting out with the first being the root filesystem and the second being a normal mounted file system, and ending with the second being the root and the first being mounted somewhere else in that root. 

It might seem that all of the issues with combining the two functions into a single 'mount' operation have been adequately resolved in the natural course of development, but it is hard to be convinced of this. The collection of namespace manipulation functions that we now have is quite _ad hoc_ and so, while it seems to meet current needs, there can be no certainty that it is in any sense complete. A hint of this incompleteness can be seen in the fact that, once you perform a lazy unmount, the filesystem may well still exist, but it is no longer possible to manipulate it as it does not have a name in the global namespace, and all current manipulation operations require such a name. This makes it difficult to perform a 'forced' unmount after a 'lazy' unmount. 

To see what a complete interface would look like we would need to exploit the design concept discussed last week: "everything can have a file descriptor". Had that pattern been imposed on the design of the mount system call we would likely have: 

  * A mount call that simply returned a file descriptor for the file system. 
  * A bind call that connected a file descriptor into the namespace, and 
  * An unmount call that disconnected a filesystem and returned a file descriptor. 
This simple set would easily provide all the functionality that we currently have in an arguably more natural way. For example the functionality currently provided by the special-purpose `pivot_root()` system call could be achieve with the above with at most the addition of `fchroot()`, an obvious analogue of `fchdir()` and `chroot()`. 

One of the many strengths of Unix - particularly seen in the set of tools that came with the kernel - is the principle of building and then combining tools. Each tool should do one thing and do it well. These tools can then be combined in various ways, often to achieve ends that the tool developer could not have foreseen. Unfortunately the same discipline was not maintained with the `mount()` system call. 

So this pattern is to some extent the opposite of the 'tools approach'. It needs a better name than that, though; a good choice seems to be to call it a "conflated design". One dictionary ([PJC](<http://onlinedictionary.datasegment.com/word/conflate>)) defines "conflate" as "to ignore distinctions between, by treating two or more distinguishable objects or ideas as one", which seems to sum up the pattern quite well. 

#### The open() system call.

Our second example of a conflated design is found in the `open()` system call. This system call (in Linux) takes 13 distinct flags which modify its behavior, adding or removing elements of functionality - multiple concepts are thus combined in the one system call. Much of this combination does not imply a conflated design. Several of the flags can be set or cleared independently of the `open()` using the `F_SETFL` option to `fcntl()`. Thus while they are commonly combined, they are easily separated and so need not be considered to be conflated. 

Three elements of the `open()` call are worthy of particular attention in the current context. They are `O_TRUNC`, `O_CLOEXEC` and `O_NONBLOCK`. 

In early versions of Unix, up to and including Level 7, opening with `O_TRUNC` was the only way to truncate a file and, consequently, it could only be truncated to become empty. Partial truncation was not possible. Having truncation intrinsically tied to `open()` is exactly the sort of conflated design that should be avoided and, fortunately, it is easy to recognize. BSD Unix introduced the `ftruncate()` system call which allows a file to be truncated after it has been opened and, additionally, allows the new size to be any arbitrary value, including values greater than the current file size. Thus that conflation was easily resolved. 

`O_CLOEXEC` has a more subtle story. The standard behavior of the `exec()` system call (which causes a process to stop running one program and to start running another) is that all file descriptors available before the `exec()` are equally available afterward. This behavior can be changed, quite separately from the `open()` call which created the file descriptor, with another `fcntl()` call. For a long time this appeared to be a perfectly satisfactory arrangement. 

However the advent of threads, where multiple processes could share their file descriptors (so when one thread or process opens a file, all threads in the group can see the file descriptor immediately), made room for a potential race. If one process opens a file with the intent of setting the close-on-exec flag immediately, and another process performs an `exec()` (which causes the file table to not be shared any more), the new program in the second process will inherit a file descriptor which it should not. In response to this problem, the recently-added `O_CLOEXEC` flag causes `open()` to mark the file descriptor as close-on-exec atomically with the open so there can be no leakage. 

It could be argued that creating a file descriptor and allowing it to be preserved across an `exec()` should be two separate operations. That is, the default should have been to not keep a file descriptor open across `exec()`, and a special request would be needed to preserve it. However foreseeing the problems of threads when first designing `open()` would be beyond reasonable expectations, and even to have considered the effects on `open()` when adding the ability to share file tables would be a bit much to ask. 

The main point of the `O_CLOEXEC` example then is to acknowledge that recognizing a conflated design early can be very hard, which hopefully will be an encouragement to put more effort in reviewing a design for these sorts of problems. 

The third flag of interest is `O_NONBLOCK`. This flag is itself conflated, but also shows conflation within `open()`. In Linux, `O_NONBLOCK` has two quite separate, though superficially similar, meanings. 

Firstly, `O_NONBLOCK` affects all read or write operations on the file descriptor, allowing them to return immediately after processing less data than requested, or even none at all. This functionality can separately be enabled or disabled with `fcntl()` and so is of little further interest. 

The other function of `O_NONBLOCK` is to cause the `open()` itself not to block. This has a variety of different effects depending on the circumstances. When opening a named pipe for write, the open will fail rather than block if there are no readers. When opening a named pipe for read, the open will succeed rather than block, and reads will then return an error until some process writes something into the pipe. On CDROM devices an open for read with `O_NONBLOCK` will also succeed but no disk checks will be performed and so no reads will be possible. Rather the file descriptor can only be used for `ioctl()` commands such as to poll for the presence of media or to open or close the CDROM tray. 

The last gives a hint concerning another aspect of `open()` which is conflated. Allocating a file descriptor to refer to a file and preparing that file for I/O are conceptually two separate operations. They certainly are often combined and including them both in the one system call can make sense. _Requiring_ them to be combined is where the problem lies. 

If it were possible to get a file descriptor on a given file (or device) without waiting for or triggering any action within that file, and, subsequently, to request the file be readied for I/O, then a number of subtle issues would be resolved. In particular there are various races possible between checking that a file is of a particular type and opening that file. If the file was renamed between these two operations, the program might suffer unexpected consequences of the open. The `O_DIRECTORY` flag was created precisely to avoid this sort of race, but it only serves when the program is expecting to open a directory. This race could be simply and universally avoided if these two stages of opening a file were easily separable. 

A strong parallel can be seen between this issue and the 'socket' API for creating network connections. Sockets are created almost completely uninitialized; thereafter a number of aspects of the socket can be tuned (with e.g. `bind()` or `setsockopt()`) before the socket is finally connected. 

In both the file and socket cases there is sometimes value in being able to set up or verify some aspects of a connection before the connection is effected. However with `open()` it is not really possible in general to separate the two. 

It is worth noting here that opening a file with the 'flags' set to '3' (which is normally an invalid value) can sometimes have a similar meaning to `O_NONBLOCK` in that no particular read or write access is requested. Clearly developers see a need here but we still don't have a uniform way to be certain of getting a file descriptor without causing any access to the device, or a way to upgrade a file descriptor from having no read/write access to having that access. 

As we saw, most of the difficulties caused by conflated design, at least in these two examples, have been addressed over time. It could therefore be argued that as there is minimal ongoing pain, the pattern should not be a serious concern. That argument though would miss two important points. Firstly they have already caused pain over many years. This could well have discouraged people from using the whole system and so reduce the overall involvement in, and growth of, the Unix ecosystem. 

Secondly, though the worst offenses have largely been fixed, the result is not as neat and orthogonal as it could be. As we saw during the exploration, there are some elements of functionality that have not yet been separated out. This is largely because there is no clear need for them. However we often find that a use for a particular element of functionality only presents itself once the functionality is already available. So by not having all the elements cleanly separated we might be missing out on some particular useful tools without realizing it. 

There are undoubtedly other areas of Unix or Linux design where multiple concepts have been conflated into a single operation, however the point here is not to enumerate all of the flaws in Unix. Rather it is to illustrate the ease with which separate concepts can be combined without even noticing it, and the difficulty (in some cases) of separating them after the fact. This hopefully will be an encouragement to future designers to be aware of the separate steps involved in a complex operation and to allow - where meaningful - those steps to be performed separately if desired. 

Next week we will continue this exploration and describe a pattern of misdesign that is significantly harder to detect early, and appears to be significantly harder to fix late. Meanwhile, following are some exercises that may be used to explore conflated designed more deeply. 

#### Exercises.

  1. Explain why `open()` with `O_CREAT` benefits from an `O_EXCL` flag, but other system calls which create filesystem entries (`mkdir()`, `mknod()`, `link()`, etc) do not need such a flag. Determine if there is any conflation implied by this difference. 

  2. Explore the possibilities of the hypothetical `bind()` call that attaches a file descriptor to a location in the namespace. What other file descriptor types might this make sense for, and what might the result mean in each case. 

  3. Identify one or more design aspects in the IP protocol suite which show conflated design and explain the negative consequences of this conflation. 

#### Next article

[Ghosts of Unix past, part 3: Unfixable designs](<https://lwn.net/Articles/414618/>)
