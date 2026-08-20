---
title: "Finishing the conversion to the \"new\" mount API"
url: https://lwn.net/Articles/979166/
date: "June 26, 2024"
category: "Filesystems-Mounting"
author: "By Jake Edge June 26, 2024 LSFMM+BPF"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jake Edge**  
June 26, 2024

* * *

[LSFMM+BPF](<https://lwn.net/Articles/lsfmmbpf2024/>)

Eric Sandeen led a filesystem-track session at the 2024 [Linux Storage, Filesystem, Memory Management, and BPF Summit](<https://events.linuxfoundation.org/lsfmmbpf/>) on completing the conversion of the existing kernel filesystems to use the [mount API](<https://lwn.net/Articles/759499/>) that was [added for the 5.2 kernel](<https://lwn.net/Articles/787963/>) in 2019. That API is invariably called the "new" API, which it _is_ when compared to the venerable [`mount()`](<https://man7.org/linux/man-pages/man2/mount.2.html>) system call, but it has been available for five years or so at this point without really pushing its predecessor aside. Sandeen wanted to discuss the status of the conversion process and some other questions surrounding the new API. 

He began by saying the session is "not really a rocket-science talk", instead it was more of a "let's get that thing that we said we were going to do, done" talk. The original idea was to finish the conversion to the new API, then deprecate and remove the internal API that is used by the old mount API. But, after an initial push, there were few conversions until the pace picked up somewhat during the last two releases. 

Of the 56 or so kernel filesystems, around 30 still remain to be converted, Sandeen said, so he has been joking that the completion of the effort will be in 2026. A bunch of filesystem support had just been merged during the 6.10 merge window—which happened during the conference. The two most prominent filesystems that still need to be converted are fat, which has patches floating around the list, and bcachefs, which he looked at briefly but did not tackle. 

He encouraged the maintainers of any of the filesystems that still need conversion to "go for it"; the maintainers should have a better idea what mount support and options are needed for users. But, he noted, some of the kernel filesystems are abandoned. There may not be user-space tools or even a filesystem image to work with, he said, so whoever takes on the task of converting those is just going to have to do their best. 

#### Logging

Another part of the API that he wanted to talk about was the message logging that filesystems can use to communicate warnings and errors during the mount process to user space. There are three functions ([`infof()`, `warnf()`, and `errorf()`](<https://elixir.bootlin.com/linux/v6.9.6/source/include/linux/fs_context.h#L189>)) that allow returning text strings to the callers of the API. When he started looking at converting filesystems, he first thought that `printk()` calls should be changed to use those logging functions, but has changed his mind because there are "different audiences" for those messages. 

[ ![\[Eric Sandeen\]](https://static.lwn.net/images/2024/lsfmb-sandeen-sm.png) ](<https://lwn.net/Articles/979679/>)

He asked David Howells, who developed the new mount API, to describe the original intent for the logging functionality. Howells said that there were two main purposes; first it provides a channel to report what went wrong during a mount operation, which is especially useful when the user-space process cannot access [`dmesg`](<https://man7.org/linux/man-pages/man1/dmesg.1.html>). It also provides a way for filesystems to ask questions, such as for passwords. 

Amir Goldstein said that there is more to it than just access to the `dmesg` log, because there is no way to know if the user-space tools will actually print any messages logged using the new API. Christian Brauner said that the [util-linux tools](<https://github.com/util-linux/util-linux>) have added support for these messages, but Goldstein pointed out that the kernel still has no way to know that the user will see them. Brauner agreed that they should still be sent to `dmesg`, but he also noted that information sent to the kernel logs can become part of the user-space API of the kernel, so there is a need for caution. 

Ted Ts'o said that part of the problem is that random user-space programs are scraping the `dmesg` information via the log files or perhaps the serial console. If the information that gets sent to `dmesg` changes, "there may be some random system-administration script that gets cranky". Those tools are arguably wrong to do so, he said, but users will complain to the filesystem developers if it happens. 

The mount API puts log messages into a [`struct fc_log`](<https://elixir.bootlin.com/linux/v6.9.6/source/include/linux/fs_context.h#L169>) context, Brauner said. User space can then read the data that gets logged by using the file descriptor returned from an [`fsopen()`](<https://elixir.bootlin.com/linux/v6.9.6/source/fs/fsopen.c#L115>) system call. Currently, just the errors (or maybe errors and warnings, no one seemed to be sure) from that stream are written to `dmesg`. 

Sandeen said that whatever is going to `dmesg` today should continue to go there, in order to avoid complaints. Another reason not to change existing behavior, Brauner said, is that systemd uses a known-bad mount option to probe the kernel to see how it can find out about illegal options; to a monitoring tool, those could look like a continuing stream of errors to be reported. Similarly, Goldstein said that overlayfs has a lot of different fallbacks that it tries that could be misinterpreted if they are handled differently. 

Ts'o suggested that the conversation get more concrete. He wanted to try to define the log level that would be used for invalid mount options in `dmesg`, rather than try to guess whether programs will complain or get confused. "When we say 'we can't log to `dmesg`', that may not be true" because it depends on the log level of the message. Existing practice will have to be accommodated, but filesystem developers need to define the goal for these messages. 

Dave Chinner said that existing filesystems already have their own mechanisms for reporting things like invalid mount options, which cannot really change, so he suggested not changing what is sent to `dmesg` at all. User-space programs already need to access that information and can continue to do so. Goldstein said that overlayfs had no mechanism of its own, so it uses `dmesg` to report problems; now there is a better way to do that reporting, however there is no way to know if user space will actually print those messages so that users can find them. Brauner said that the lack of documentation has hampered the adoption of the mount logging that came with the new API; since developers do not know about it and cannot find out much about it, there is no real discussion or agreement on how to use it. 

Steve French noted that he had experienced similar problems in the early 2000s when he was working on SMB; "I needed to be able to tell user space things and all I had were like ten return codes that were valid". There were thousands of different things that might have gone wrong, he said. Jeff Layton agreed with that, "an integer return is not expressive enough", which is the rationale for the mount-logging feature. 

Another problem that occurs, Layton said, is that on a busy system there may be lots of mounting going on, which makes it difficult to determine the correspondence between messages and mount operations. Brauner said that the mount logging API has a prefix that can be associated with each message. While it is true that existing reporting mechanisms need to be maintained, they are not consistent between filesystems. "How beautiful would it be", he asked, if all of the error messages from VFS had a "vfs:" prefix—likewise for "bcachefs:" and the other filesystems. 

Kent Overstreet said that bcachefs has a system of error codes that allows developers to track down exactly where in the code a problem occurred, which has been extremely useful. Chinner said that XFS only reports certain kinds of information to user space, while reporting the details and exact location in the code when filesystem corruption is found, for example, to `dmesg`. The user-space report just tells the user to unmount and run `fsck`; the details should not be sent to user space, and the two types of information should be kept separate, he said. 

#### Unknown options

Sandeen said that when he went to add support for the mount API to tracefs and debugfs, he encountered a comment about ignoring unknown mount options. But in the conversion process, most filesystems now reject unknown mount options regardless of their earlier behavior. One exception is NFS, which has a `sloppy` mount option that means unknown options should not cause an error. He wondered if there was a need to maintain the previous behavior when converting. 

The `sloppy` option is important for network filesystems, French said, because new options may get added but not be available on every server. Sandeen agreed, but noted that `sloppy` has gotten even more complicated because it is positional; it needs to be specified before any potentially unknown option. Layton said that it is also needed for automount maps; in the past, there were sites that were administering both Solaris and Linux systems using the same maps, but the mount options did not line up between the two. 

Brauner said that switching to using the new mount API is a conscious decision, so changes to the behavior for things like unknown options would be acceptable. But, eventually, the plan is for the existing `mount` command to use the new API, which will need to preserve the existing behavior. So there will need to be a way to do that. 

"Remount is even worse", Brauner said. Most filesystems will accept any options for remount then silently ignore those that they do not care about, Sandeen said. An attendee asked if the kernel should care; "should we tell user space 'you can't remount that thing'?" While he could not think of an example, Brauner said that he could imagine a security-sensitive mount option being passed to remount; in that case, user space would want the operation to fail if the option could not be handled. 

The (re)mount-option handling has been inconsistent and broken forever, Brauner said, but there is a need to be able to express the intent of a given mount operation with respect to unknown-option handling. That will be needed for users of the new mount API and, eventually, for `mount` itself. There are other cases that are messy as well. The new API allows options to be specified one at a time, but if a new option conflicts with an earlier one, there is no way to say which of the earlier ones caused the conflict. 

XFS has a table that tracks all of the possible options and which cannot be used together so that they can be tied together in error messages, Chinner said. Ts'o said that ext4 has something similar. Chinner did not think it was important to handle that problem in a generic way since it is a corner case that does not arise frequently. While Howells said that handling conflicting options was part of his initial proposal for the API, Al Viro had him remove it. As time ran out on the session, Lennart Poettering agreed with Chinner that user-space tools likely did not really want to handle that level of detail.
