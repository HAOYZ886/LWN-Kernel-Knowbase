---
title: Conventions for extensible system calls
url: https://lwn.net/Articles/830666/
date: "September 8, 2020"
category: System calls
author: "By Jonathan Corbet September 8, 2020 LPC"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jonathan Corbet**  
September 8, 2020

* * *

[LPC](<https://lwn.net/Archives/ConferenceByYear/#2020-Linux_Plumbers_Conference>)

The kernel does not have just one system call to rename a file; instead, there are three of them: [`rename()`, `renameat()`, and `renameat2()`](<https://man7.org/linux/man-pages/man2/renameat.2.html>). Each was added when the previous one proved unable to support a new feature. A similar story has played out with a number of system calls: a feature is needed that doesn't fit into the existing interfaces, so a new one is created — again. At the 2020 [Linux Plumbers Conference](<https://linuxplumbersconf.org>), Christian Brauner and Aleksa Sarai ran a pair of sessions focused on the creation of future-proof system calls that can be extended when the need for new features arises. 

Brauner started by noting that the problem of system-call extensibility has been discussed repeatedly on the mailing lists. The same arguments tend to come up for each new system call. Usually, developers try to follow one of two patterns: a full-blown multiplexer that handles multiple functions behind a single system call, or creating a range of new, single-purpose system calls. We have burned ourselves and user space with both, he said. There are no good guidelines to follow; it would be better to establish some conventions and come to an agreement on how future kernel APIs should be designed. 

The requirements for system calls should be stronger, and they should be well documented. There should be a minimal level of extensibility built into every new call, so that there is never again a need to create a `renameat2()`. The baseline, he said, is [a flags argument](<https://lwn.net/Articles/585415/>); that convention is arguably observed for new system calls today. This led to a brief side discussion on why the type of the flags parameter should be `unsigned int`; in short, signed types can be sign extended, possibly leading to the setting of a lot of unintended flags. 

Sarai took over to discuss the various ways that exist now to deal with system-call extensions. One of those is to add a new system call, which works, but it puts a big burden on user-space code, which must change to make use of this call. That includes checking to see whether the new call is supported at all on the current system and falling back to some other solution in its absence. The other extreme, he said, is multiplexers, which have significant problems of their own. 

Yet another approach is to set a flag (on calls that support flags, obviously) and make the system call take a variable number of arguments; among other things, variadic calls are difficult for C libraries to support properly. It is necessary to pass all possible arguments, leading to passing garbage on the stack into the kernel. System calls could be designed with fixed-size structs as arguments, with the idea that these structs would contain enough padding to handle any future needs. The problem with this approach, he said, was that it requires an ability to predict the future, which turns out to be difficult to do. 

Finally, he said, the problem could be solved by getting and using a time machine. This solution, too, suffers from practical difficulties. 

#### Extensible structs

Brauner and Sarai are pushing a different solution that they call "extensible structs"; it is the approach used in the design of the [`openat2()`](<https://man7.org/linux/man-pages/man2/openat2.2.html>) system call. This mechanism works by marshaling parameters to a system call into a single C structure; a pointer to that structure and the size of the structure are passed as the parameters to the system call. That size parameter acts as a sort of version number. When the time comes to extend the system call in a way the requires passing new data to the kernel, new fields are added to the end of the structure, increasing the size. Those new fields are always designed so that a value of zero implies the behavior that existed before those fields were added. 

When the system call is invoked from user space, the kernel compares the structure size passed in to its own idea of how big the structure should be. If the size from user space is smaller, that indicates that the caller expects to use an older version of the system call; the fields that user space did not provide are filled with zeroes and the call proceeds as usual, with no visible change in behavior. If, instead, the kernel's size is smaller than the structure passed from user space, then the kernel is the older side. In that case, all of the excess fields (from the kernel's point of view) are checked; if they are all zero, the call can proceed. Otherwise, the call fails with an `E2BIG` error, since user space is requesting functionality that the kernel does not know how to provide. 

H. Peter Anvin questioned the use of the `E2BIG` return code. Sarai responded that its assigned meaning is "argument list too long"; it is a bit weird, he said, but "it makes sense if you squint". 

> [![\[Aleksa Sarai, Arnd
Bergmann, Christian Brauner, and Florian Weimer\]](https://static.lwn.net/images/conf/2020/lpc/syscalls-sm.png)](<https://lwn.net/Articles/830675/>)

Brauner emphasized that the rules should prohibit any further multiplexer system calls. This is almost universally agreed on now. `[bpf()](<https://man7.org/linux/man-pages/man2/bpf.2.html>)` is not an extensible system call; it is a multiplexer that happens to use extensible structs. So `bpf()` is not an example of how system calls should be designed in the future. The GNU C Library developers have also made it clear that they would rather not see any more multiplexer calls, since they are hard to deal with at that level. Arnd Bergmann said that a number of developers try to block multiplexer system calls, but it is hard to see them all; that is why this rule should be in the documentation, Brauner replied. 

Anvin asked why the structure size is passed as a separate argument rather than being embedded in the structure itself. Brauner replied that it feels more "C-like" to pass the size in the argument list, but that it doesn't matter that much in the end. The community should pick one convention or the other, though. Anvin said that the separate size can create problems if the struct is being passed through different user-space layers that may have different ideas of what its proper size is; that could result in passing bad data to the kernel. 

Kees Cook, instead, said that confusion in user space is preferable to confusion in the kernel. There could be problems if the kernel reads the size from within the structure, then uses that size to read the structure itself in to kernel space. That size might have changed in between the two reads, possibly opening up vulnerabilities within the kernel. Thus, he said, the size is better passed as a separate element. When asked which convention would work better for C libraries, Florian Weimer responded that he didn't have a strong opinion either way. 

Brauner said that, in the past, Al Viro has [called](<https://lwn.net/ml/linux-kernel/20200719171054.GK2786714%40ZenIV.linux.org.uk/>) extensible structs a ""crap insertion vector"", saying that they could be used to sneak new features into the kernel without review. That view doesn't hold up, Brauner said; as an example, the feature that prompted this reaction was indeed caught in review. There is not a problem there that is worse than with any other system call, he said. 

This part of the discussion closed with Brauner saying that the new conventions need to be added to the documentation. The current [documentation on adding system calls](<https://www.kernel.org/doc/html/latest/process/adding-syscalls.html>) is not up to date; he tried to update it in the past the but that effort "went largely unnoticed". He has been trying to build more consensus around the extensible structs idea; meanwhile, he has landed two new system calls using it. He will be working on a new version of the documentation patch that is intended to describe the current best practices as he sees them. 

#### Probing feature support

Extensible structs might be a solution to the problem of adding features to system calls, but they do not, in themselves, address the other part of the problem: helping user space to figure out which features are actually supported on a given system. As Brauner noted, programs normally have to adopt a sort of trial-and-error approach, where they try to exercise each feature in question and see if it actually works. This process is painful at best, and it can become expensive, especially in libraries and short-running programs. It should be possible to do better. The conversation on just how to do better got started in this session, then continued in a birds-of-a-feather session later. 

Sarai mentioned one proposal that has been circulating: add a "no-op" flag to the system call. The kernel would respond by returning a copy of the extensible struct with all valid flag bits set and non-flag fields filled with the highest supported value. A unique error code would be returned if the queried operation is not supported at all. Bergmann quickly pointed out that this flag would turn the system call into a sort of multiplexer with multiple functions, but noted that he could see the upside of doing things that way. 

The alternative would be to create a new system call dedicated to checking which features are supported by other system calls. This would not be entirely straightforward to implement, since it requires adding a new infrastructure within the kernel for defining system-call features. Brauner noted that there are two specific types of extensibility that would have to be handled: adding a new flag, and increasing the size of the struct. 

Bergmann suggested that a minimal solution might be to add a counter in the VDSO area exported by the kernel to user space; every time a feature is added, the counter would be incremented. Cook answered that it was a little too minimal; user space would benefit more from the ability to inquire about the availability of specific features. Weimer said that this would make writing portable software more difficult, since it would be necessary to test all possible permutations of features, but Cook responded that this problem already exists, and the ability to query features would just improve visibility. Mark Rutland suggested starting with a single system call defining exactly which information would be exposed; Brauner said that [`clone3()`](<https://man7.org/linux/man-pages/man2/clone3.2.html>) might be a good starting point. 

There appeared to be a consensus that a separate system call is the right way to solve this problem, so the discussion turned to what this system call would look like. Brauner started with a suggestion that this new call would take the number of the system call of interest, and would return the current set of valid flags and struct size. Sarai pointed out that `openat2()` has two flags arguments, complicating the situation. Weimer, instead, said he would like to be able to query whether [`vfork()`](<https://man7.org/linux/man-pages/man2/vfork.2.html>) is available, but there's no flags argument there at all. 

Mark Rutland suggested an API that would look something like: 

```
int sys_features(int syscall_no, u32 *map, size_t mapsize);
```

The kernel would treat `map` as a bitmap to be filled in describing the available features for the requested system call. Each system call would have a set of constants for each feature that may or may not be present, each corresponding to one bit that would be set in `map` if the feature is available. This proposal seemed to gain some support during the discussion. 

Brauner said that there is also a need to automate the process of adding a system call to the kernel — preferably one not written in Perl. It could perform a number of checks, including whether the struct size is described correctly and the `sys_features()` information is correct. Bergmann suggested extending the `DECLARE_SYSCALL()` macro in the kernel to take the bitmap of supported features as an additional argument. 

The conversation wandered on for a little longer, but the form of the outcome was already clear. For the extensible structs portion, the main action item will be to update the documentation to reflect the new consensus (if there is truly a consensus) on how extensibility should be handled. For the feature-query system call, the task will be to write some code showing how it would actually work. Once the concept hits the mailing lists it may end up changing significantly. One can only hope that the end result gets things right, so we won't need a `sys_features2()` in the future.
