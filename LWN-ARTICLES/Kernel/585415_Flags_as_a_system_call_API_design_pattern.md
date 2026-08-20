---
title: Flags as a system call API design pattern
url: https://lwn.net/Articles/585415/
date: "February 12, 2014"
category: "Development model-Patterns; System calls"
author: "February 12, 2014 This article was contributed by Michael Kerrisk ."
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

February 12, 2014

This article was contributed by [Michael Kerrisk](<http://man7.org/>).

The `renameat2()` system call [recently proposed](<https://lwn.net/Articles/569134/>) by Miklos Szeredi is a fresh reminder of a category of failures in the design of kernel-user-space APIs that has a long history on Linux (and, going even further back, Unix). A closer look at that history yields a lesson that should be kept in mind for all future system calls added to the kernel. 

The `renameat2()` system call is an extension of `renameat()` which, in turn, is an extension of the ancient `rename()` system call. All of these system calls perform the same general task: manipulating directory entries to give an existing file a new name on the same filesystem. The original `rename()` system call took just two arguments: the old pathname and the new pathname. [`renameat()`](<http://man7.org/linux/man-pages/man2/renameat.2.html>) added two arguments, one associated with each pathname argument. Each of the new arguments can be a file descriptor that refers to a directory: if the corresponding pathname argument is relative, then it is interpreted relative to the associated directory file descriptor, rather than the current working directory (as is done by `rename()`). 

`renameat()` was one of a raft of thirteen new system calls added to Linux in kernel 2.6.16 to perform various operations on files. The twofold purpose of the directory file descriptor argument is elaborated in the [`openat(2)`](<http://man7.org/linux/man-pages/man2/openat.2.html#NOTES>) manual page: 

  * to avoid race conditions that could occur with the corresponding traditional system calls if one of the directory components in a (relative) pathname was changed at the same time as the system call, and 

  * to allow the implementation of per-thread "current working directories" via directory file descriptors. 

The next step, `renameat2()`, extends the functionality of `renameat()` to support a new use case: atomically _swapping_ two existing pathnames. Although that use case is related to the earlier system calls, it was necessary to define a new system call for one simple reason: `renameat()` lacked a mechanism for the kernel to support (and the caller to request) variations in its behavior. In other words, it lacked the kind of `flags` bit-mask argument that is provided by system calls such as `clone()`, `fcntl()`, `mremap()`, and `open()`, all of which allow a varying number of arguments, depending on the bits specified in the `flags` argument. 

`renameat2()` implements the new "swap" functionality _and_ adds a new `flags` argument whose bits can be used to select variations in behavior of the system call. The first of these bits is `RENAME_EXCHANGE`, which selects the "swap" functionality; without that flag, `renameat2()` behaves like `renameat()`. The addition of the `flags` arguments hopefully forestalls the need to one day create a `renameat3()` system call to add other new functionality. And indeed, Andy Lutomirski [soon observed](<http://thread.gmane.org/gmane.linux.file-systems/78606/focus=78843>) that another flag could be added: `RENAME_NOREPLACE`, to prevent a rename operation from overwriting an existing file. Formerly, the only race-free way of preventing an existing file from being clobbered was to use `link()` (which fails if the target pathname exists) to create the new name, followed by `unlink()` to remove the old name. 

#### Mistakes repeated

There is, of course, a sense of _d eja vu_ about the `renameat2()` story, since the reason that the earlier `renameat()` system call was required was that `rename()` lacked the extensibility that would have been allowed by a `flags` argument. Consideration of this example prompts one to ask: "How many times have we made that particular mistake?" The answer turns out to be "quite a few." 

One does not need to go far to find some other examples. Returning to the thirteen "directory file descriptor" system calls that were added in Linux 2.6.16, we find that, with no particular rhyme or reason, four of the new system calls (`fchownat()`, `fstatat()`, `linkat()`, and `unlinkat()`) added a `flags` argument that was not present in the traditional call, while eight others (`faccessat()`, `fchmodat()`, `futimesat()`, `mkdirat()`, `mknodat()`, `readlinkat()`, `renameat()`, and `symlinkat()`) did not. (The remaining call, `openat()`, retained the `flags` argument that was already present in `open()`.) 

Of the new calls that did not include a `flags` argument, one, `futimesat()`, was soon superseded by a new call that _did_ have a `flags` argument (`utimensat()`, added in Linux 2.6.22), and `renameat()` seems poised to suffer the same fate. One is left wondering: would any of the remaining calls also have benefited from the inclusion of a `flags` argument? Studying this set of functions further, it is soon evident that the answer is "yes", in at least three cases. 

The first case is the `faccessat()` system call. This system call lacks a `flags` flags argument, but the GNU C Library (glibc) wrapper function adds one. If bits are specified in that argument, then the wrapper function instead uses the `fstatat()` system call to determine file access permissions. It seems clear that the lack of a `flags` argument was realized too late, and the design problem was subsequently papered over in glibc. (The implementer of the "directory file descriptor" system calls was the then glibc maintainer.) 

The second case is the `fchmodat()` system call. Like the `faccessat()` system call, it lacks a `flags` argument, but the glibc wrapper adds one. That wrapper function allows for an `AT_SYMLINK_NOFOLLOW` flag. However, the flag is not currently supported, because the kernel doesn't provide the necessary support. Clearly, the glibc wrapper function was written to allow for the possibility of an `fchmodat2()` system call in the future. 

The third case is the `readlinkat()` system call. To understand why this system call would have benefited from a `flags` argument, we need to consider three of the system calls that were added in Linux 2.6.13 that do permit a `flags` argument--`fchownat()`, `fstatat()`, and `linkat()`. Those system calls added the `AT_EMPTY_PATH` flag in Linux 2.6.39. If this flag is specified in the call, and the pathname argument is an empty string, then the call instead operates on the open file referred to by the "directory file descriptor" argument (and in this case, that argument can refer to file types other than directories). This allows these system calls to provide functionality analogous to that provided by `fchmod()` and `fstat()` in the traditional Unix API. (There is no "`flink()`" in the traditional API.) 

Strictly speaking, the `AT_EMPTY_PATH` functionality could have been supported without the use of a flag: if the pathname argument was an empty string, then these calls could have assumed that they are to operate on the file descriptor argument. However, the requirement to use a flag serves the dual purposes of documenting the programmer's intent and preventing accidents that might occur if the pathname argument was unintentionally specified as an empty string. 

The "operate on a file descriptor" functionality also turned out to be useful for `readlinkat()`, which likewise added that functionality in Linux 2.6.39. However, `readlinkat()` does not have a `flags` argument; the call simply operates on the file descriptor if the pathname argument is an empty string, and thus does not have the benefits that the `AT_EMPTY_PATH` flag confers on the other system calls. Thus `readlinkat()` is another system call where a `flags` argument would have been desirable. 

In summary, then, of the eight "directory file descriptor" system calls that lacked a `flags` argument, this lack has turned out to be a mistake in at least five cases. 

Of course, Linux developers were not the first to make this kind of design error. Long before Linux appeared, there was `wait()` without `flags` and then `wait3()` with `flags`. And Linux has gone on to fix some instances of this design error in APIs inherited from Unix, adding, for example, `dup3()` as a successor to `dup2()`, and `pipe2()` as the successor to `pipe()` (both new system calls added in kernel 2.6.27). 

#### Latter-day missing-flags examples

But, given the lessons of history, we've managed to repeat the mistake far too many times in Linux-specific system calls. As well as the directory file descriptor examples mentioned above, here are some other examples: 

> Original system call   | Successor  
> ---|---  
> `epoll_create()` (2.6.0) | `epoll_create1()` (2.6.27)  
> `eventfd()` (2.6.22) | `eventfd2()` (2.6.27)  
> `inotify_init()` (2.6.13) | `inotify_init1()` (2.6.27)  
> `signalfd()` (2.6.22) | `signalfd4()` (2.6.27)  
  
The realization that certain system calls might need a `flags` argument sometimes comes in waves, as developers realize that multiple related APIs may need such an argument; one such wave occurred in Linux 2.6.13, when four of the "directory file descriptor" system calls added a `flags` argument. 

As can be seen from the other examples shown just above, another such wave occurred in kernel 2.6.27, when a total of six new system calls were added. All of these new calls, as well as `accept4()`, which was added for the same reasons in Linux 2.6.28, return new file descriptors. The main reason for the addition of the new calls was to allow the caller the option of requesting that the close-on-exec flag be set on the new file descriptor at the time it is created, rather than in a separate step using the `fcntl(F_SETFD)` operation. This allows user-space applications to avoid certain race conditions when using the traditional counterparts of these system calls in multithreaded applications. Those races could occur when one thread tried to create a file descriptor and use `fcntl(F_SETFD)` to set its close-on-exec flag at the same time as another thread happened to perform a `fork()` plus `execve()`. (The `socket()` and `socketpair()` system calls also added this new functionality in 2.6.27. However, somewhat bizarrely, this was done by jamming bit flags into the high bytes of these calls' socket type argument, rather than creating new system calls with a `flags` argument.) 

Turning to more recent Linux development history, we see that a number of new system calls added since kernel 2.6.28 have all included a `flags` argument, including `fanotify_init()`, `fanotify_mark()`, `open_by_handle_at()`, and `name_to_handle_at()`. However, in all of those cases, the `flags` argument was required at the outset, so no decision about future-proofing this aspect of the API was required. 

On the other hand, there have been some misses or near misses for other system calls. The `syncfs()` system call added in Linux 2.6.39 does not have a `flags` argument, although one wonders whether some filesystem developer might have taken advantage of such a flag, if it existed, to allow the caller to vary the manner in which a filesystem is synced to disk. And the `finit_module()` system call added in Linux 3.8 only got a `flags` argument after some [last minute prompting](<http://thread.gmane.org/gmane.linux.kernel/1362095/focus=1374617>); once added, the flag [proved immediately useful](<http://thread.gmane.org/gmane.linux.kernel/1362095/focus=1379995>). 

The conclusion from this oft-repeated pattern of creating new incarnations of system calls that add a `flags` argument is that a suitable question to ask during the design of every new system call is: "Is there a reason _not_ to include a `flags` argument in the API?" Considering the question from that perspective is likely to more often lead developers to default to following the wise example of the `process_vm_readv()` and `process_vm_writev()` system calls added in Linux 3.2. The developers of those system calls included a (currently unused) `flags` argument on the suspicion that it may prove useful in the future. History suggests that they'll one day be proved right.
