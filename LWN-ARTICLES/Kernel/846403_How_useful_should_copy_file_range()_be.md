---
title: "How useful should copy_file_range() be?"
url: https://lwn.net/Articles/846403/
date: "February 18, 2021"
category: "System calls-copy file range"
author: "By Jonathan Corbet February 18, 2021"
---

> **Ready to give LWN a try?**
> 
> With a subscription to LWN, you can stay current with what is happening in the Linux and free-software community and take advantage of subscriber-only site features. We are pleased to offer you **[a free trial subscription](<https://lwn.net/Promo/nst-trial1/claim>)** , no credit card required, so that you can see for yourself. Please, join us! 

By **Jonathan Corbet**  
February 18, 2021

The [`copy_file_range()` system call](<https://man7.org/linux/man-pages/man2/copy_file_range.2.html>) looks like a relatively straightforward feature; it allows user space to ask the kernel to copy a range of data from one file to another, hopefully applying some optimizations along the way. In truth, this call has never been as generic as it seems, though some changes made during 5.3 helped in that regard. When the developers of the Go language ran into problems with `copy_file_range()`, there ensued a lengthy discussion on how this system call should work and whether the kernel needs to do more to make it useful. 

The definition of `copy_file_range()` is: 

```
ssize_t copy_file_range(int fd_in, loff_t *off_in,
                                int fd_out, loff_t *off_out,
                                size_t len, unsigned int flags);
```

Its job is to copy `len` bytes of data from the file represented by `fd_in` to `fd_out`, observing the requested offsets at both ends. The `flags` argument must be zero. This call first [appeared](<https://lwn.net/Articles/659523/>) in the 4.5 release. Over time it turned out to have a number of unpleasant bugs, leading to a long series of fixes and [some significant grumbling](<https://lwn.net/Articles/774114/>) along the way. 

In 2019 Amir Goldstein [fixed more issues](<https://lwn.net/Articles/789527/>) and, in the process, removed a significant limitation: until then, `copy_file_range()` refused to copy between files that were not located on the same filesystem. After this patch was merged (for 5.3), it could copy between any two files, falling back on [`splice()`](<https://man7.org/linux/man-pages/man2/splice.2.html>) for the cross-filesystem case. It appeared that `copy_file_range()` was finally settling into a solid and useful system call. 

Indeed, it seemed useful enough that the Go developers decided to use it for the [`io.Copy()` function](<https://golang.org/pkg/io/#Copy>) in their standard library. Then they ran into a problem: `copy_file_range()` will, when given a kernel-generated file as input, copy zero bytes of data and claim success. These files, which include files in `/proc`, tracefs, and a large range of other virtual filesystems, generally indicate a length of zero when queried with a system call like [`stat()`](<https://man7.org/linux/man-pages/man2/stat.2.html>). `copy_file_range()`, seeing that zero length, concludes that there is no data to copy and the job is already done; it then returns success. 

But there is actually data to be read from this kind of file, it just doesn't show in the advertised length of the file; the real length often cannot be known before the file is actually read. Before 5.3, the prohibition on cross-filesystem copies would have caused most such attempts to return an error code; afterward, they fail but appear to work. The kernel is happy, but some users can be surprisingly stubborn about actually wanting to copy the data they asked to be copied; they were rather less happy. 

#### Marking virtual filesystems

Nicolas Boichat tried to mollify those users with [this patch set](<https://lwn.net/ml/linux-kernel/20210212044405.4120619-1-drinkcat@chromium.org/>) to `copy_file_range()`. It added a flag (`FS_GENERATED_CONTENT`) to the `file_system_type` structure for virtual filesystems where the length of the files cannot be known in advance. `copy_file_range()` would then look for that flag and return an error code when it was found; the error return would cause `io.Copy()` to fall back to a manual copy operation. This change appeared to solve the immediate problem, but it is not destined to be merged into the mainline. 

There were a few objections, starting with the fact that it requires all virtual filesystems to be specially marked. The patch set did not mark them all, and this mechanism would require that developers be sure to mark all future filesystems as they were added. As Greg Kroah-Hartman [put it](<https://lwn.net/ml/linux-kernel/YCY+Ytr2J2R5Vh0+@kroah.com/>): ""That way lies madness and constant auditing that I do not see anyone signing up for for the next 20 years"". 

The bigger question, though, was whether this behavior should be seen as a bug at all. Boichat described it as a regression; code that would fall back to a normal copy before 5.3 would silently fail to copy data thereafter. Kroah-Hartman was unsure, though; he continued: 

Why are people trying to use copy_file_range on simple /proc and /sys files in the first place? They can not seek (well most can not), so that feels like a "oh look, a new syscall, let's use it everywhere!" problem that userspace should not do. 

Dave Chinner was, if anything, [less sympathetic](<https://lwn.net/ml/linux-kernel/20210212230346.GU4626@dread.disaster.area/>): 

It is a targeted solution for *regular files only* on filesystems that store persistent data and can accelerate the data copy in some way (e.g. clone, server side offload, hardware offload, etc). It is not intended as a copy mechanism for copying data from one random file descriptor to another. 

The use of it as a general file copy mechanism in the Go system library is incorrect and wrong. It is a userspace bug. Userspace has done the wrong thing, userspace needs to be fixed. 

The problem with this attitude, as described by Go developer Ian Lance Taylor, is that figuring out when `copy_file_range()` can be used is not easy; he [pointed out](<https://lwn.net/ml/linux-kernel/CAKOQZ8zPFM29DYPwbnUJEhf+a8kPSJ5E_W06JLFjn-5Fy-ZWWw@mail.gmail.com/>) that these limitations are not mentioned in the `copy_file_range()` man page, and [argued](<https://lwn.net/ml/linux-kernel/CAOyqgcVTYhozM-mwc400qt+fabmUuBQTsjqbcA03xDooYXXcMA@mail.gmail.com/>) that this behavior reduces the utility of the system call considerably: 

From my perspective, as a kernel user rather than a kernel developer, a system call that silently fails for certain files and that provides no way to determine either 1) ahead of time that the system call will fail, or 2) after the call that the system call did fail, is a useless system call. I can never use that system call, because I don't know whether or not it will work. 

Chinner [said](<https://lwn.net/ml/linux-kernel/20210212232726.GW4626@dread.disaster.area/>) that the test is whether it is possible to tell whether a file has data in it without calling `read()` on it. But Darrick Wong, hardly a filesystem amateur, [replied](<https://lwn.net/ml/linux-kernel/20210212235448.GH7187@magnolia/>): ""I don't know how to do that, Dave. :)"" There is another fun twist, as Boichat [pointed out](<https://lwn.net/ml/linux-kernel/CANMq1KDv-brWeKOTt3aUUi_1SOXSpEFo5pS5A6mpRT8k-O88nA@mail.gmail.com/>): files in sysfs, rather than indicating a zero length, claim to be 4,096 bytes long — regardless of their true length, which may be larger or smaller than that. Chinner's test will fail on those files, even if it can be reliably carried out. 

#### Toward a real fix

Wong went on to express agreement with the Go developers: `copy_file_range()` should either work as expected or return an error so that user space can know to fall back to copying the old-fashioned way. He also suggested a couple of ways to possibly fix the problem, the first of which was to go back to the previous state of affairs, where cross-filesystem copies were explicitly disallowed. Failing that, one could restrict such copies to a single filesystem type that has explicit support for them. Luis Henriques [implemented a variant of that idea](<https://lwn.net/ml/linux-kernel/20210215154317.8590-1-lhenriques@suse.de/>), where copies across filesystems would still be allowed if the two filesystems were of the same type, and if the filesystem involved explicitly implements the `copy_file_range()` operation. 

That patch was stopped in its tracks, though, after Trond Myklebust [pointed out](<https://lwn.net/ml/linux-kernel/92d27397479984b95883197d90318ee76995b42e.camel@hammerspace.com/>) that the kernel's NFS daemon uses the `copy_file_range()` mechanism to copy files between filesystems of different types. Blocking that would break some important functionality; this usage pattern exists in other filesystems, such as Ceph and FUSE, as well. In response to that, Henriques [added a new flag](<https://lwn.net/ml/linux-kernel/87r1lgjm7l.fsf@suse.de/>) (`COPY_FILE_SPLICE`) that could be used within the kernel to indicate that a cross-filesystem-type copy should be performed. There was some question of whether this flag should be made available to user space for cases when it somehow knows that the operation would succeed, but it seems that will not happen. 

A final version of this patch has not been posted as of this writing, but the eventual shape of the fix seems clear. When called from user space, `copy_file_range()` will only try to copy a file across filesystems if the two are of the same type, and if that filesystem has explicit support for the system call (and, thus, is presumably written with all of the possible cases in mind). Otherwise, the call will fail with an explicit error, so user space will know that it must copy the data some other way. So `copy_file_range()` will never be a generic file-copy mechanism, but it will at least be possible to use robustly in code that is prepared for it to fail. 

There is still one more trap lurking within `copy_file_range()`, though. Like most I/O-related system calls, `copy_file_range()` can copy fewer bytes than requested; user space needs to check the return value to see how much work was actually done. There is currently no way to distinguish between copies that were cut short on the read side (by hitting the end of the file, perhaps) and those that were stopped on the write side (which may well indicate a write error). Nobody has come up with a real solution to that problem yet. 

All of this goes to show how a seemingly simple interface can quickly become complex. `copy_file_range()` has revealed a number of sharp edges over its relatively short existence; there may well be more yet to be found. It is thus perhaps unsurprising that the kernel developers, having been burned more than once, feel a strong desire to keep its implementation as simple as possible.
