---
title: Cleanup on aisle fsconfig()
url: https://lwn.net/Articles/1054228/
date: "January 21, 2026"
category: "Filesystems-Mounting"
author: "By Jake Edge January 21, 2026 LPC"
---

> **Benefits for LWN subscribers**
> 
> The primary benefit from [subscribing to LWN](<https://lwn.net/Promo/nst-nag5/subscribe>) is helping to keep us publishing, but, beyond that, subscribers get immediate access to all site content and access to a number of extra site features. Please sign up today! 

By **Jake Edge**  
January 21, 2026

* * *

[LPC](<https://lwn.net/Archives/ConferenceIndex/#Linux_Plumbers_Conference-2025>)

As part of the process of writing man pages for the ["new" mount API](<https://lwn.net/Articles/759499/>), which has been available in the kernel since 2019, Aleksa Sarai encountered a number of places where the [`fsconfig()`](<https://manpages.opensuse.org/Tumbleweed/man-pages/fsconfig.2.en.html>) system call—for configuring filesystems before mounting—needs to be cleaned up. In the 2025 Linux Plumbers Conference (LPC) [session](<https://lpc.events/event/19/contributions/2239/>) that he led, Sarai wanted to discuss some of the problems he found, including at least one with security implications. The idea of the session was for him to describe the various bugs and ambiguities that he had found, but he also wanted attendees to raise other problems they had with the system call. 

Christian Brauner, who helped organize the ["Containers and checkpoint/restore" microconference](<https://lpc.events/event/19/sessions/217/#20251212>) (and LPC as well), introduced the session by referring to the ""horrific design"" of `fsconfig()`—something that Sarai immediately disclaimed (""I didn't say that""). Sarai began by noting that there are now man pages for the mount API, which may help improve the [adoption of the API](<https://lwn.net/Articles/979166/>) by filesystems; his theory is that adoption lagged due to having to read the code in order to understand the system calls. ""Hopefully, this is at least a slight improvement."" 

The new mount API, perhaps more properly ""the suite of file-descriptor-based mount facilities"" as the man page calls it, breaks up the [`mount()`](<https://manpages.opensuse.org/Tumbleweed/man-pages/mount.2.en.html>) system call into multiple steps to provide a more granular approach to the myriad of ways that filesystems can be mounted in Linux. `fsconfig()` is used to set parameters and otherwise customize a filesystem context that has been created with [`fsopen()`](<https://manpages.opensuse.org/Tumbleweed/man-pages/fsopen.2.en.html>) or obtained from an existing mounted filesystem using [`fspick()`](<https://manpages.opensuse.org/Tumbleweed/man-pages/fspick.2.en.html>). The function prototype for `fsconfig()` is as follows, from the man page: 

```
int fsconfig(int fd, unsigned int cmd,
                     const char *_Nullable key,
                     const void *_Nullable value, int aux);
```

The `fd` parameter is for the filesystem context to operate on, while `cmd` is the operation requested; `key`, `value`, and `aux` provide additional information based on the operation chosen. 

#### `FSCONFIG_SET_PATH`

[ ![\[Aleksa Sarai\]](https://static.lwn.net/images/2026/lpc-sarai-sm.png) ](<https://lwn.net/Articles/1055221/>)

Sarai said that the [`FSCONFIG_SET_PATH`](<https://manpages.opensuse.org/Tumbleweed/man-pages/fsconfig.2.en.html#FSCONFIG_SET_PATH>) and [`FSCONFIG_SET_PATH_EMPTY`](<https://manpages.opensuse.org/Tumbleweed/man-pages/fsconfig.2.en.html#FSCONFIG_SET_PATH_EMPTY>) commands are almost completely unused by filesystems; he thought they were not used at all but, right before the session, found out that the ext4 `journal_path` parameter can be set that way. It is unfortunate that no other filesystem parameters can be set using those commands, because they can take a directory file descriptor, thus providing more options for specifying the path, as with [`openat()`](<https://manpages.opensuse.org/Tumbleweed/man-pages/openat.2.en.html>). In part due to helpers that are ""a little bit janky"", filesystems require their paths to be set as strings using [`FSCONFIG_SET_STRING`](<https://manpages.opensuse.org/Tumbleweed/man-pages/fsconfig.2.en.html#FSCONFIG_SET_STRING>), which is the same form as options to the `mount()` system call. As [noted](<https://manpages.opensuse.org/Tumbleweed/man-pages/fsconfig.2.en.html#Generic_filesystem_parameters>) in the `fsconfig()` man page, the `source` path parameter, normally used for the block device containing the filesystem, must be set using `FSCONFIG_SET_STRING`, but others ostensibly could use the set-path commands. 

Ideally, he thinks that most filesystems want to support all three commands for their file parameters, but none of the [helpers](<https://www.kernel.org/doc/html/latest/filesystems/mount_api.html#parameter-helper-functions>) currently support that. A single helper that handles those three, plus the related [`FSCONFIG_SET_FD`](<https://manpages.opensuse.org/Tumbleweed/man-pages/fsconfig.2.en.html#FSCONFIG_SET_FD>) command would be useful. He wondered, though, whether full [`O_PATH`](<https://manpages.opensuse.org/Tumbleweed/man-pages/openat.2.en.html#O_PATH>) support was needed for the paths that are being set. 

A file descriptor opened using `O_PATH` behaves differently than one obtained from a regular file open—and it can be done without the permissions needed to actually open the file. For that reason, Lennart Poettering thought that `O_PATH` support should not be added to the helper; files should be opened normally as that will be more secure, he said. There was no opposition to adding helpers as Sarai described, so he turned to his next topic. 

#### Singletons and exclusive create

Singleton filesystems, such as debugfs and others that have the same contents no matter how and where they are mounted, are a perennial problem, he said. The [`FSCONFIG_CMD_CREATE_EXCL`](<https://manpages.opensuse.org/Tumbleweed/man-pages/fsconfig.2.en.html#FSCONFIG_CMD_CREATE_EXCL>) command was [developed](<https://lwn.net/Articles/939937/>) and merged by Brauner a few years ago, but it is a ""very big hammer"" that is largely unusable because it does not provide any extra information to the caller if it fails. It is the counterpart to the [`FSCONFIG_CMD_CREATE`](<https://manpages.opensuse.org/Tumbleweed/man-pages/fsconfig.2.en.html#FSCONFIG_CMD_CREATE>) command, which is used to turn a configured filesystem context into a filesystem instance that can be mounted using [`fsmount()`](<https://manpages.opensuse.org/Tumbleweed/man-pages/fsmount.2.en.html>). 

There is a hidden surprise when using the create command: almost all of the configuration (with the exception of the read-write and read-only flags) that has been done using `fsconfig()` is silently ignored if the filesystem instance is already present in the kernel. So `FSCONFIG_CMD_CREATE_EXCL` requests that the kernel create a context without reusing an existing filesystem configuration, thus ensuring that the configuration requested is used. But if it cannot, ""it just gives you an error and you can't do anything about it""; it is simply an instance of ""computer says 'no' and that's basically all you can do about it"". 

The conversion to the new mount API has broken some singleton filesystems, because the semantics of [`vfs_get_super()`](<https://elixir.bootlin.com/linux/v6.18.6/source/fs/super.c#L1318>) have changed, but some developers were not aware of that. The bug was [fixed for debugfs](<https://lwn.net/ml/all/20250816-debugfs-mount-opts-v3-1-d271dad57b5b@posteo.net/>) and [for tracefs](<https://lwn.net/ml/all/20241030171928.4168869-2-kaleshsingh%40google.com/>). In general, the semantics of superblock reuse are not clear, and the messages provided do not give enough information about what parameters were ignored. 

Brauner noted that `FSCONFIG_CMD_CREATE_EXCL` is meant for times when the user absolutely must have a particular filesystem-configuration option, otherwise `FSCONFIG_CMD_CREATE` should be used. But Sarai pointed out that the existing filesystem configuration may be just fine, but in order to be sure the application uses `FSCONFIG_CMD_CREATE_EXCL`, which would fail even if the required parameter is already set in the existing configuration. Part of the problem is that the actual set of configuration options is not completely resolved until the superblock is actually created, Brauner said; for example, filesystems are not required to resolve the path parameters supplied until superblock-creation time. There is no ""generically elegant"" solution for finding out whether a filesystem context really contains a configuration value of interest. 

#### `fc_log`

Information about conflicts between parameters and the like can be logged using [logfc()](<https://elixir.bootlin.com/linux/v6.18.6/source/fs/fs_context.c#L433>) (which uses [`struct fc_log`](<https://elixir.bootlin.com/linux/v6.18.6/source/include/linux/fs_context.h#L182>)), but that interface suffers from some problems as well. The interface had been discussed at LSFMM+BPF 2024, as well, Sarai said, which is described in the article linked above. For one thing, `fc_log` has a limit of eight entries before it overflows, ""and the fun part is that there's no priority"", so eight informational messages could delete an error or warning message. 

User space can read the messages logged via a file descriptor returned from `fsopen()`, but ideally it would need to do so after every mount-API call and Sarai said that the util-linux tools do not do that. He wondered if increasing the size of `fc_log` made sense. Brauner asked if there was a way to know if any messages were dropped; Sarai did not think so. Overall, though, the `fc_log` interface is much better than trying to parse things out of the dmesg log, Sarai said. Brauner agreed, noting that util-linux does print out those messages in versions where it is using the new mount API, which is a big improvement. 

Brauner thought that some levels of `fc_log` messages were also output to dmesg, though Sarai was not convinced that was true. Sarai did think that unread `fc_log` messages should be written to dmesg so that they do not just disappear. It might make sense to provide a way for user space to poll the `fc_log` messages, so that it can read them as soon as they are available, Brauner said. There is a problem that the format and wording of those messages becomes part of the kernel ABI, which may be unavoidable, but is something to keep in mind. 

Sarai described an idea he had just come up with, which would allow `fc_log` messages to be extended with, say, some JSON that came after a NUL byte in the buffer; that would allow users to simply print the message (as they do now) or to parse it further if they need more information. Poettering suggested following the model of [kmsg](<https://www.kernel.org/doc/Documentation/ABI/testing/dev-kmsg>), which was long unstructured, but eventually added some structure, including a sequence number so that overwritten messages can be detected. Brauner said there is some structure to the `fc_log` messages, with prefixes for different filesystems, for example, but it is still ""wildly inconsistent"" among them. Even the VFS is inconsistent about what and where it logs information. Switching to a structured format might be a good idea, but it would require a new flag for `fsopen()` to allow user space to request structured logs. 

The limit of eight messages is something that should probably be addressed, Sarai said. Poettering agreed, noting that it was an ""irritatingly low"" number for something of that nature. Jeff Layton speculated that David Howells (who designed the API but was not present) expected users to check for messages after each call, and that only a few messages would be generated for each. That has not really been borne out, however, attendees seemed to agree. 

#### `FS_CONTEXT_FAILED`

The final topic Sarai wanted to raise was the `FS_CONTEXT_FAILED` state that a filesystem context enters if there is any kind of error. At that point, the context cannot be inspected or otherwise operated upon, so if user space wants to try again with different options, it has to start all over. This comes up for the [runc](<https://github.com/opencontainers/runc?tab=readme-ov-file#runc>) container tool, he said, because it has various fallbacks that it wants to try if a mount fails. In order to do that, it has to keep all of the options around, remake the context, and try again (and again, if that fails). That is not too bad, ""but it's just kind of awful that it goes into this fail state, where all you have are log messages"" to try to figure out what went wrong. 

Brauner speculated that it was that way because the VFS cannot really know what caused the failure, so it cannot know whether the context can be changed and retried. There may be filesystems that enter into a non-recoverable state if the superblock creation fails, for example. On the other hand, there are situations where a new mount option is introduced that a filesystem may or may not implement; it is unfortunate that the option cannot just be removed and the mount retried. 

It would make sense to have a way for a filesystem to decide whether it is a non-recoverable error or not, Brauner said. Part of the problem is that the API is ""an unfinished project in a sense""; Howells had also proposed the [`fsinfo()` system call](<https://lwn.net/Articles/829212/>) that would have allowed querying the filesystem context and more. It was rejected and has never resurfaced, though [`statmount()`](<https://manpages.opensuse.org/Tumbleweed/man-pages/statmount.2.en.html>) was separated out as its own system call, Brauner said. It might be interesting to consider resurrecting a ""very slimmed-down version"" of `fsinfo()`. 

Using a structured-message kmsg-like approach would allow filesystems or the VFS to put the required information into some newer `fc_log`, Poettering said. Those applications that care can pull out the information about options that were rejected (or translated to a compatible option); that way they can programmatically determine what needs to change for a retry. It is effectively the same as the JSON idea that Sarai had mentioned; there just needs to be agreement about the structure among the subsystems. It would also make it easy to simply add the messages to kmsg if they are going to be overwritten or were not read. Sarai seemed amenable to that approach. 

With that, the session ran out of time. Interested readers can check out the [YouTube video](<https://www.youtube.com/watch?v=NX5IzF6JXp0>) and [slides](<https://lpc.events/event/19/contributions/2239/attachments/1789/3868/Clean%20Up%20on%20Aisle%20fsconfig\(2\)%20%5BLPC%202025%5D.pdf>) from the talk. 

[ I would like to thank our travel sponsor, the Linux Foundation, for assistance with my travel to Tokyo for Linux Plumbers Conference. ]
