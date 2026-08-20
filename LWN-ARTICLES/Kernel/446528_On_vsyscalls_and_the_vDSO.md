---
title: On vsyscalls and the vDSO
url: https://lwn.net/Articles/446528/
date: "June 8, 2011"
category: Security; vDSO
author: "By Jonathan Corbet June 8, 2011"
---

> **This article brought to you by LWN subscribers**
> 
> Subscribers to LWN.net made this article -- and everything that surrounds it -- possible. If you appreciate our content, please [buy a subscription](<https://lwn.net/Promo/nst-nag3/subscribe>) and make the next set of articles possible. 

By **Jonathan Corbet**  
June 8, 2011

The "vsyscall" and "vDSO" segments are two mechanisms used to accelerate certain system calls in Linux. While their basic function (provide fast access to functionality which does not need to run in kernel mode) is the same, there are some distinct differences between them. Recently vsyscall has come to be seen as an enabler of security attacks, so some patches have been put together to phase it out. The discussion of those patches shows that the disagreement over how security issues are handled by the community remains as strong as ever. 

The vsyscall area is the older of these two mechanisms. It was added as a way to execute specific system calls which do not need any real level of privilege to run. The classic example is `gettimeofday()`; all it needs to do is to read the kernel's idea of the current time. There are applications out there that call `gettimeofday()` frequently, to the point that they care about even a little bit of overhead. To address that concern, the kernel allows the page containing the current time to be mapped read-only into user space; that page also contains a fast `gettimeofday()` implementation. Using this virtual system call, the C library can provide a fast `gettimeofday()` which never actually has to change into kernel mode. 

Vsyscall has some limitations; among other things, there is only space for a handful of virtual system calls. As those limitations were hit, the kernel developers introduced the more flexible vDSO implementation. A quick look on a contemporary system will show that both are still in use: 

```
$ cat /proc/self/maps
        ...
        7fffcbcb7000-7fffcbcb8000 r-xp 00000000 00:00 0            [vdso]
        ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0    [vsyscall]
```

The key to the current discussion can be seen by typing the same command again and comparing the output: 

```
7fff379ff000-7fff37a00000 r-xp 00000000 00:00 0             [vdso]
        ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0     [vsyscall]
```

Note that the vDSO area has moved, while the vsyscall page remains at the same location. The location of the vsyscall page is nailed down in the kernel ABI, but the vDSO area - like most other areas in the user-space memory layout - has its location randomized every time it is mapped. 

Address-space layout randomization is a form of defense against security holes. An attacker who is able to overrun the stack can often arrange for a function in the target process to "return" to an arbitrary address. Depending on what instructions are found at that address, this return can cause almost anything to happen. Returning into the `system()` function in the C library is an obvious example; it can be used to execute arbitrary commands. If the location of the C library in memory is not known, though, then it becomes difficult or impossible for an exploit to jump into a useful place. 

There is no `system()` function in the vsyscall page, but there are several machine instructions that invoke system calls. With just a bit of setup, these instructions might be usable in a stack overrun attack to invoke an arbitrary system call with attacker-defined parameters - not a desirable outcome. So it would be nice to get rid of - or at least randomize the location of - the vsyscall page to thwart this type of attack. Unfortunately, applications depend on the existence and exact address of that page, so nothing can be done. 

Except that Andrew Lutomirski [found something that could be done](<https://lwn.net/Articles/446220/>): remove all of the useful instructions from the vsyscall page. One was associated with the `vsyscall64` sysctl knob, which is really only useful for user-mode Linux (and does not work properly even there); it was simply deleted. Others weren't actually system call instructions as such: the system time, if jumped into (and, thus, executed as if it were code) when it held just the right value, looks like a system call instruction. To address that problem, variables have been moved into a separate page with execute permission turned off. 

The remaining code in the vsyscall page has simply been removed and replaced by a special trap instruction. An application trying to call into the vsyscall page will trap into the kernel, which will then emulate the desired virtual system call in kernel space. The result is a kernel system call emulating a virtual system call which was put there to avoid the kernel system call in the first place. The result is a "vsyscall" which takes a fraction of a microsecond longer to execute but, crucially, does not break the existing ABI. In any case, the slowdown will only be seen if the application is trying to use the vsyscall page instead of the vDSO. 

Contemporary applications should not be doing that most of the time, except for one little problem: glibc still uses the vsyscall version of `time()`. That has been fixed in the glibc repository, but the fix may not find its way out to users for a while; meanwhile, `time()` calls will be a little slower than they were before. That should not really be an issue, but one never knows, so Andy put in a configuration option to preserve the old way of doing things. Anybody worried about the overhead of an emulated vsyscall page can set `CONFIG_UNSAFE_VSYSCALLS` to get the old behavior. 

Nobody really objected to the patch series as a whole, but Linus hated the name of the configuration option; he asked that it be called `CONFIG_LEGACY_VSYSCALLS` instead. Or, even better, the change could just be done unconditionally. That led to a fairly predictable [response](<https://lwn.net/Articles/446542/>) from the PaX developer on how the kernel community likes to hide security problems, to which Linus [said](<https://lwn.net/Articles/446539/>): 

Calling the old vdso "UNSAFE" as a config option is just plain stupid. It's a politicized name, with no good reason except for your political agenda. And when I call it out as such, you just spout the same tired old security nonsense. 

Suffice to say that the conversation went downhill from there; interested parties can follow the thread links in the messages cited above. 

One useful point from that discussion is that the static vsyscall page is not, in fact, a security vulnerability; it's simply a resource which can make it easier for an attacker to exploit a vulnerability elsewhere in the system. Whether that aspect makes that page "unsafe" or merely "legacy" is left as an exercise for the reader. Either way, removing it is seen as a good idea even though that removal might, arguably, cause real security bugs to remain unfixed in the kernel; the argument is all about naming. 

Final versions of the patches have not been posted as of this writing, but the shape they will take is fairly clear. The static vsyscall page will not continue to exist in its current form, and applications which still use it will continue to work but will get a little bit slower. The configuration option controlling this behavior may or may not exist, but any distribution shipping a kernel containing this change (presumably 3.1 or later) will also have a C library which no longer tries to use the vsyscall page. And, with luck, exploiting vulnerabilities will get just a little bit harder.
