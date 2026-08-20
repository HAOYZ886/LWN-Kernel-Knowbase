---
title: An unexpected perf feature
url: https://lwn.net/Articles/550901/
date: "May 21, 2013"
category: "Security-Vulnerabilities"
author: "By Jake Edge May 21, 2013"
---

> **LWN.net needs you!**
> 
> Without subscribers, LWN would simply not exist. Please consider [signing up for a subscription](<https://lwn.net/Promo/nst-nag2/subscribe>) and helping to keep LWN publishing. 

By **Jake Edge**  
May 21, 2013

Local privilege escalations seem to be regularly found in the Linux kernel these days, but they usually aren't quite so old--more than two years since the release of 2.6.37--or backported into even earlier kernels. But [CVE-2013-2094](<http://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2013-2094>) is just that kind of bug, with a now-public exploit that apparently dates back to 2010. It (ab)uses the `perf_event_open()` system call, and the bug was backported to the 2.6.32 kernel used by Red Hat Enterprise Linux (and its clones: CentOS, Oracle, and Scientific Linux). While local privilege escalations are generally considered less worrisome on systems without untrusted users, it is easy to forget that UIDs used by network-exposed services should also qualify as untrusted--compromising a service, then using a local privilege escalation, leads directly to root. 

The bug was found by Tommi Rantala when running the [Trinity fuzz tester](<https://lwn.net/Articles/536173/>) and was [fixed](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=8176cced706b5e5d15887584150764894e94e02f>) in mid-April. At that time, it was [not recognized as a security problem](<https://lwn.net/Articles/550679/>); the release of an [exploit](<http://fucksheep.org/~sd/warez/semtex.c>) in mid-May certainly changed that. The exploit is dated 2010 and contains some possibly "not safe for work" strings. Its author [expressed surprise](<https://lwn.net/Articles/551220/>) that it wasn't seen as a security problem when it was fixed. That alone is an indication (if one was needed) that people in various colored hats are scrutinizing kernel commits--often in ways that the kernel developers are not. 

The bug itself was [introduced](<http://git.kernel.org/cgit/linux/kernel/git/torvalds/linux.git/commit/?id=b0a873ebbf87bf38bf70b5e39a7cadc96099fa13>) in 2010, and made its first appearance in the 2.6.37 kernel in January 2011\. It treated the 64-bit perf event ID differently in an initialization routine (`perf_swevent_init()` where the ID was sanity checked) and in the cleanup routine (`sw_perf_event_destroy()`). In the former, it was treated as a signed 32-bit integer, while in the latter as an unsigned 64-bit integer. The difference may not seem hugely significant, but, as it turns out, it can be used to effect a full compromise of the system by privilege escalation to root. 

The key piece of the puzzle is that the event ID is used as an array index in the kernel. It is a value that is controlled by user space, as it is passed in via the `struct perf_event_attr` argument to [`perf_event_open()`](<http://man7.org/linux/man-pages/man2/perf_event_open.2.html>). Because it is sanity checked as an `int`, the upper 32 bits of `event_id` can be anything the attacker wants, so long as the lower 32 bits are considered valid. Because `event_id` is used as a signed value, the test: 

```
if (event_id >= PERF_COUNT_SW_MAX)
                return -ENOENT;
```

doesn't exclude negative IDs, so anything with bit 31 set (i.e. 0x80000000) will be considered valid. 

The exploit code itself is rather terse, obfuscated, and hard to follow, but Brad Spengler has provided a detailed [description of the exploit](<http://www.reddit.com/r/netsec/comments/1eb9iw/sdfucksheeporgs_semtexc_local_linux_root_exploit/c9ykrck>) on Reddit. Essentially, it uses a negative value for the event ID to cause the kernel to change user-space memory. The exploit uses `mmap()` to map an area of user-space memory that will be targeted when the negative event ID is passed. It sets the mapped area to zeroes, then calls `perf_event_open()`, immediately followed by a `close()` on the returned file descriptor. That triggers: 

```
static_key_slow_dec(&perf_swevent_enabled[event_id]);
```

in the `sw_perf_event_destroy()` function. The code then looks for non-zero values in the mapped area, which can be used (along with the event ID value and the size of the array elements) to calculate the base address of the `perf_swevent_enabled` array. 

But that value is just a steppingstone toward the real goal. The exploit gets the base address of the interrupt descriptor table (IDT) by using the `sidt` assembly language instruction. From that, it targets the overflow interrupt vector (0x4), using the increment in `perf_swevent_init()`: 

```
static_key_slow_inc(&perf_swevent_enabled[event_id]);
```

By setting `event_id` appropriately, it can turn the address of the overflow interrupt handler into a user-space address. 

The exploit arranges to `mmap()` the range of memory where the clobbered interrupt handler will point and fills it with a [NOP sled](<http://en.wikipedia.org/wiki/NOP_slide>) followed by [shellcode](<http://en.wikipedia.org/wiki/Shellcode>) that accomplishes its _real_ task: finding the UID/GIDs and capabilities in the credentials of the current process so that it can modify them to be UID and GID 0 with full capabilities. At that point, in what almost feels like an afterthought, it spawns a shell--a root shell. 

Depending on a number of architecture- or kernel-build-specific features (not least x86 assembly) makes the exploit itself rather fragile. It also contains bugs, according to Spengler. It doesn't work on 32-bit x86 systems because it uses a hard-coded system call number (298) passed to `syscall()`, which is different (336) for 32-bit x86 kernels. It also won't work on Ubuntu systems because the size of the `perf_swevent_enabled` array elements is different. The following will thwart the existing exploit: 

```
echo 2 > /proc/sys/kernel/perf_event_paranoid
```

But a minor change to the flags passed to `perf_event_open()` will still allow the privilege escalation. None of these is a real defense of any sort against the _vulnerability_ , though they do defend against this specific exploit. Spengler's analysis has more details, both of the existing exploit as well as ways to change it to work around its fragility. 

The code uses `syscall()`, presumably because `perf_event_open()` is not (yet?) available in the GNU C library, but it could also be done to evade any argument checks done in the library. Any sanity checking done by the library must also be done in the kernel, because using `syscall()` can avoid the usual system call path. Kernels configured without support for perf events (i.e. `CONFIG_PERF_EVENTS` not set) are unaffected by the bug as they lack the system call entirely. 

There are several kernel hardening techniques that would help to avoid this kind of bug leading to system compromise. The grsecurity [UDEREF](<http://grsecurity.net/~spender/uderef.txt>) mechanism would prevent the kernel from dereferencing the user-space addresses so that the `perf_swevent_enabled` base address could not be calculated. The PaX/grsecurity [KERNEXEC](<http://forums.grsecurity.net/viewtopic.php?f=7&t=3292>) technique would prevent the user-space shellcode from executing. While these techniques can inhibit this kind of bug from allowing privilege escalation, they impose costs (e.g. performance) that have made them unattractive to the mainline developers. Suitably configured kernels on hardware that supports it would be protected by [supervisor mode access prevention](<https://lwn.net/Articles/517475/>) (SMAP) and [supervisor mode execution protection](<http://forums.grsecurity.net/viewtopic.php?f=7&t=3046>) (SMEP), the former would prevent access to the user-space addresses much like UDEREF, while the latter would prevent execution of user-space code as does KERNEXEC. 

This is a fairly nasty hole in the kernel, in part because it has existed for so long (and apparently been known by some, at least, for most of that time). Local privilege escalations tend to be somewhat downplayed because they require an untrusted local user, but web applications (in particular) can often provide just such a user. Dave Jones's Trinity has clearly shown its worth over the last few years, though he was [not terribly pleased](<http://codemonkey.org.uk/2013/05/16/cve-2013-2094/>) how long it took for fuzzing to find this bug. 

Jones suspects there may be ""more fruit on that branch somewhere"", so more and better fuzzing of the perf system calls (and kernel as a whole) is indicated. In addition, the exploit author at least suggests that he has more exploits waiting in the wings (not necessarily in the perf subsystem), it is quite likely that others do as well. Finding and fixing these security holes is an important task; auditing the commit stream to help ensure that these kinds of problems aren't introduced in the first place would be quite useful. One hopes that companies using Linux find a way to fund more work in this area.
