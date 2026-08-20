---
title: "Fun with NULL pointers, part 1"
url: https://lwn.net/Articles/342330/
date: "July 20, 2009"
category: "Security-Vulnerabilities"
author: "By Jonathan Corbet July 20, 2009"
---

> **We're bad at marketing**
> 
> We can admit it, marketing is not our strong suit. Our strength is writing the kind of articles that developers, administrators, and free-software supporters depend on to know what is going on in the Linux world. Please [subscribe today](<https://lwn.net/Promo/nsn-bad/subscribe>) to help us keep doing that, and so we don’t have to get good at marketing. 

By **Jonathan Corbet**  
July 20, 2009

By now, most readers will be familiar with [the local kernel exploit](<http://lwn.net/Articles/341773/>) recently posted by Brad Spengler. This vulnerability, which affects the 2.6.30 kernel (and a test version of the RHEL5 "2.6.18" kernel), is interesting in a number of ways. This article will look in detail at how the exploit works and the surprising chain of failures which made it possible. 

The TUN/TAP driver provides a virtual network device which performs packet tunneling; it's useful in a number of situations, including virtualization, virtual private networks, and more. In normal usage of the TUN driver, a program will open `/dev/net/tun`, then make an `ioctl()` call to set up the network endpoints. Herbert Xu recently noticed a problem where a lack of packet accounting could let a hostile application pin down large amounts of kernel memory and generally degrade system performance. His solution was [a patch](<http://git.kernel.org/?p=linux/kernel/git/torvalds/linux-2.6.git;a=commitdiff;h=33dccbb050bbe35b88ca8cf1228dcf3e4d4b3554>) which adds a "pseudo-socket" to the device which can be used by the kernel's accounting mechanisms. Problem solved, but, as it turns out, at the cost of adding a more severe problem. 

The TUN device supports the `poll()` system call. The beginning of the function implementing this functionality (in 2.6.30) looks like this: 

```
static unsigned int tun_chr_poll(struct file *file, poll_table * wait)
        {
    	struct tun_file *tfile = file->private_data;
    	struct tun_struct *tun = __tun_get(tfile);
    	_struct sock *sk = tun- >sk;_
    	unsigned int mask = 0;
    
    	if (!tun)
    	    return POLLERR;
```

The line of code which has been underlined above was added by Herbert's patch; that is where things begin to go wrong. Well-written kernel code takes care to avoid dereferencing pointers which might be NULL; in fact, this code checks the `tun` pointer for just that condition. And that's a good thing; it turns out that, if the configuring `ioctl()` call has been made, `tun` will indeed be NULL. If all goes according to plan, `tun_chr_poll()` will return an error status in this case. 

But Herbert's patch added a line which dereferences the pointer prior to the check. That, of course, is a bug. In the normal course of operations, the implications of this bug would be somewhat limited: it should cause a kernel oops if `tun` is NULL. That oops will kill the process which made the bad system call in the first place and put a scary traceback into the system log, but not much more than that should happen. It should be, at worst, a denial of service problem. 

There is one little problem with that reasoning, though: NULL (zero) can actually be a valid pointer address. By default, the very bottom of the virtual address space (the "zero page," along with a few pages above it) is set to disallow all access as a way of catching null-pointer bugs (like the one described above) in both user and kernel space. But it is possible, using the `mmap()` system call, to put real memory at the bottom of the virtual address space. There are some valid use cases for this functionality, including running legacy binaries. Even so, most contemporary systems disable page-zero mappings through the use of the `mmap_min_addr` sysctl knob. 

[PULL QUOTE:  Security module checks are supposed to be additive to the checks which are already made by the kernel, but it didn't work that way this time.  END QUOTE] This knob should prevent a user-space program from mapping the zero page, and, thus, should ensure that null pointer dereferences cause a kernel oops. But, for unknown reasons, the `mmap()` code in the 2.6.30 kernel explicitly declines to enforce `mmap_min_addr` if the security module mechanism has been configured into the kernel. That job, instead, is left to the specific security module being used. Security module checks are supposed to be additive to the checks which are already made by the kernel, but it didn't work that way this time; with regard to page zero, security modules can grant access which would otherwise be denied. To complete the failure, Red Hat's default SELinux policy allows mapping the zero page. So, in this case, running SELinux actually decreased the security of the system. 

Not that life is a whole lot better without SELinux. In the absence of SELinux, the exploit will run up against the `mmap_min_addr` limit, which would seem like enough to bring things to a halt. That particular difficulty can be circumvented, though, through the use of the `personality()` system call. Enabling the SVR4 personality causes a read-only page to be mapped at address zero when a program is invoked with `exec()`, but only if the process in question has the CAP_SYS_RAWIO capability. So one more trick is required: the top-level exploit code will set the SVR4 personality, then use `exec()` to run the pulseaudio server with a special plugin module. Pulseaudio is installed setuid root, so it will get the zero page mapped at invocation time. By the time the plugin code is called, pulseaudio has dropped its privileges, but, by then, the zero page will be available to the exploit code, which can make the page writeable and place its own data there. 

As a result of all this, it is possible for a user-space process to map the zero page and prevent `tun_chr_poll()` from causing a kernel oops. But, one would think, that would not get an attacker very far, since that function checks `tun` against NULL as the very next thing it does. This is where the next interesting step in the chain of failures happens: the GCC compiler will, by default, optimize the NULL test out. The reasoning is that, since the pointer has already been dereferenced (and has not been changed), it cannot be NULL. So there is no point in checking it. Once again, this logic makes sense most of the time, but not in situations where NULL might actually be a valid pointer. 

So, an attacker is able to get into the body of `tun_chr_poll()` with a NULL `tun` pointer. One then needs to figure out how to get control of the kernel using this situation. The next step takes advantage of this code from a little further into `tun_chr_poll()`: 

```
if (sock_writeable(sk) ||
    	    (!test_and_set_bit(SOCK_ASYNC_NOSPACE, &sk->sk_socket->flags) &&
    	     sock_writeable(sk)))
    		mask |= POLLOUT | POLLWRNORM;
```

The value of `sk`, remember, came from the dereferencing of `tun`, so it's under the attacker's control. `SOCK_ASYNC_NOSPACE` is zero, so the `test_and_set_bit()` call can be used to unconditionally set the least significant bit of any word in memory. As kernel memory corruptions go, this is a small one, but it turns out to be enough. In Brad's exploit, `sk->sk_socket->flags` points into the TUN driver's `file_operations` structure; in particular, it points to the `mmap()` function. The TUN driver does not support `mmap()`, so that pointer is normally NULL; after the `poll()` call, that pointer is now one instead. 

The final step in the exploit is to call `mmap()` on a file descriptor for the open TUN device. Since the internal `mmap()` operation is no longer NULL (it has been set to one), the kernel will jump to it. That address also lives within the zero page mapped by the exploit, so it is under the attacker's control. The exploit will have populated that address with another jump to its own code. So, when the kernel calls (what it thinks is) the TUN driver's `mmap()` function, the result is arbitrary code being run in kernel mode; at that point the exploit has total control. 

In well-designed systems, catastrophic failures are rarely the result of a single failure. That is certainly the case here. Several things went wrong to make this exploit possible: security modules were able to grant access to low memory mappings contrary to system policy, the SELinux policy allowed those mappings, pulseaudio can be exploited to make a specific privileged operation available to exploit code, a NULL pointer was dereferenced before being checked, the check was optimized out by the compiler, and the code used the NULL pointer in a way which allowed the attacker to take over the system. It is a long chain of failures, each of which was necessary to make this exploit possible. 

This particular vulnerability has been closed, but there will almost certainly be others like it. See [the second article](<http://lwn.net/Articles/342420/>) in this series for a look at how the kernel developers are responding to this exploit.
