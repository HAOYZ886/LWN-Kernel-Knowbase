---
title: "A canary for timer-expiration functions"
url: https://lwn.net/Articles/731082/
date: "August 16, 2017"
category: "Security-Kernel hardening"
author: "By Jake Edge August 16, 2017"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jake Edge**  
August 16, 2017

A bug that allows an attacker to overwrite a function pointer in the kernel opens up a relatively easy way to compromise the kernel—doubly so, if an attacker simply needs to wait for the kernel use the compromised pointer. There are various techniques that can be used to protect kernel function pointers that are set at either compile or initialization time, but there are some pointers that are routinely set as the kernel runs; timer completion functions are a good example. An RFC patch posted to the kernel-hardening mailing list would add a way to detect that those function pointers have been changed in an unexpected way and to stop the kernel from executing that code. 

The [patch](<https://lwn.net/Articles/730154/>) from Kees Cook is targeting a class of vulnerabilities that arrange to overwrite the `function` field in `struct timer_list`. That field is the function that will be called when the timer expires and it conveniently (from an attacker's perspective) passes the next field in that structure (`data`) to the function. So an attacker who finds a way to overwrite `function` can likely overwrite `data` as well, leading to a fairly straightforward way to call some code of interest and to pass it an argument. As Cook put it: ""This provides attackers with a [ROP](<https://en.wikipedia.org/wiki/Return-oriented_programming>)-like primitive for performing limited kernel function calls without needing all the prerequisites to stage a ROP attack."" 

#### Exploits

In the patch, he pointed to two recent exploits that used this technique. The first was [described](<http://www.openwall.com/lists/oss-security/2016/12/06/1>) by its discoverer, Philip Pettersson, in December 2016. It uses an `AF_PACKET` socket (for raw socket handling as used by tools like `tcpdump`) and manipulates the version of the packet-socket API requested using `setsockopt()`. By changing from `TPACKET_V3` to `TPACKET_V1` at just the right time (to take advantage of a race condition), his [demonstration exploit](<https://www.exploit-db.com/exploits/40871/>) will cause the memory containing a `timer_list` to be freed without deleting the timer. 

So the timer object will be used by the kernel after it has been freed. By arranging for that memory to be reallocated somewhere that the attacker can write to (Pettersson mentions using the [`add_key()`](<http://man7.org/linux/man-pages/man2/add_key.2.html>) system call to do so), `function` and `data` can be overwritten. In the example, it actually does that twice, first to change the vsyscall table from read-only to read-write, then to register a world-writable sysctl (`/proc/sys/hack`) that sets the path of the `modprobe` executable. It arranges that the exploit program gets run as `modprobe` (as root), which leads to the execution of a root shell. 

The second recent exploit was the subject of a lengthy [Project Zero blog post](<https://googleprojectzero.blogspot.mx/2017/05/exploiting-linux-kernel-via-packet.html>) in May by Andrey Konovalov, who discovered the flaw using the [syzkaller fuzzer](<https://lwn.net/Articles/677764/>). It uses a heap buffer overflow in the `AF_PACKET` code. By arranging the heap appropriately and sending a packet with the contents of interest (the exploit code, effectively), it will place the code into memory. But that memory is user-space memory, and the Intel [supervisor mode access protection](<https://en.wikipedia.org/wiki/Supervisor_Mode_Access_Prevention>) (SMAP) and [supervisor mode execution protection](<https://en.wikipedia.org/wiki/Control_register#SMEP>) (SMEP) features will prevent the kernel from directly accessing or executing code there. So Konovalov used the same technique as Pettersson to simply disable those protections by calling `native_write_cr4()` to change the CR4 register bits as the expiration function of a socket timer. 

Once that is done, it sets up a new compromised socket and ring buffer combination that turns a transmit function pointer into a pointer to a `commit_creds(prepare_kernel_cred(0))` call in user space. Then simply transmitting a packet using the socket invokes the code, which gives the current process root privileges. 

It is interesting to note that both of these vulnerabilities can be exploited by non-privileged users on distributions (e.g. Fedora, Ubuntu) where user namespaces are enabled and unrestricted. Both require the `CAP_NET_RAW` capability to create packet sockets, which can be acquired by unprivileged users by creating a new user namespace. While the problem is not directly attributable to the user namespace code itself, it does further highlight the dangers of expanding user privileges that namespaces provide. Both Pettersson and Konovalov warn against allowing unprivileged users to create user namespaces. 

Both also avoid kernel address-space layout randomization (KASLR), SMAP, and SMEP protections. Pettersson's exploit uses hardcoded offsets for the calls of interest to avoid KASLR, while Konovalov reads `dmesg` to pluck out the kernel's text address. SMAP/SMEP are either bypassed by using kernel memory directly (Pettersson) or by explicitly disabling the features (Konovalov). 

#### A possible fix

Cook's patch would add a `canary` field to `struct timer_list` just prior to the `function` field. When a timer is initialized, the canary would be set to a value calculated by XORing the addresses of the timer and the function, along with a random number that only the kernel would know. The idea is that the canary value would also be overwritten if the function pointer is. So, before calling the function when the timer expires, the canary would be recalculated and compared with the stored value; if they differ, the function pointer has been changed and will not be called. A warning will be logged as well. 

Unfortunately, Cook soon [realized](<https://lwn.net/Articles/731170/>) that his patch was incomplete. He had addressed timers that were set up using the `setup_timer_*()` macros and the `add_timer()` function, but missed many static timer initializations that use `DEFINE_TIMER()`. He promised a revised version of the patch to handle that case. 

But it turns out that will require some extensive refactoring of the timer code, he said in response to an email query. That is a bigger job than he expected, but does provide a nice cleanup, he said. He may also have to weaken the canary for the static timers, he said in the patch followup. As with many cross-subsystem patch sets that change code across the tree, getting something like that into the mainline may be difficult. Cook [outlined some of the problems](<https://lists.linuxfoundation.org/pipermail/ksummit-discuss/2017-June/004536.html>) he and others have encountered trying to do so in a ksummit-discuss thread back in June. 

As the two exploits showed, though, the problem is real. Some kind of solution that would simply eliminate that class of vulnerabilities would be welcome. Whether Cook's canary can be that solution remains to be seen, however.
