---
title: Attacking the kernel via its command line
url: https://lwn.net/Articles/725860/
date: "June 20, 2017"
category: Security
author: "June 20, 2017 This article was contributed by Nur Hussein"
---

June 20, 2017

This article was contributed by Nur Hussein

The kernel's command line allows the specification of many operating parameters at boot time. A silly bug in command-line parsing was reported by Ilya Matveychikov on May 22; it can be exploited to force a stack buffer overflow with a controlled payload that can overwrite memory. The bug itself stems from a bounds-checking error that, while simple, has still been in the Linux kernel source since version 2.6.20. The subsequent disclosure [post](<http://seclists.org/oss-sec/2017/q2/344>) by Matveychikov in the oss-security list spawned a discussion on what constitutes a vulnerability, and what is, instead, merely a bug.

> **`$ sudo subscribe today`**
> 
> Subscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. [Act now](<https://lwn.net/Promo/nst-sudo/claim>) and you can start with a free trial subscription. 

Many kernel command-line parameters allow the specification of an array of integer values, using a syntax like: 

```
foo=1-10
```

Within the kernel, the result of such a parameter will be the filling of an array with the values one through ten. Array parameters are parsed with the [`get_options()`](<http://elixir.free-electrons.com/linux/latest/source/lib/cmdline.c#L72>) function: 

```
char *get_options(const char *str, int nints, int *ints);
```

The `nints` value specifies the maximum number of integer values that should be placed into the `ints` array. Unfortunately, nobody noticed that `get_options()` simply ignores `nints`; if the command line contains a parameter like `foo=1-1000000`, one-million entries will be written, regardless of whether the destination array has the space to hold them. There are just over 200 `get_options()` call sites in the 4.12-rc6 kernel; any one of them could be used to overwrite memory via a hostile command line. 

The example Matveychikov provided gave a huge range for the `netdev` parameter, triggering the bug. The overflow can be demonstrated while booting an affected kernel using `qemu`:

```
qemu-system-x86_64 -no-reboot -no-shutdown -kernel \
            /boot/vmlinuz-4.4.0-66-generic -append "netdev=3735928559-3735999999"
```

The numbers were chosen so that it wrote variants of the hexadecimal number 0xdeadbeef over and over when overflowing into memory (3735928559=0xdeadbeef), demonstrating that a controlled (if restricted) payload is possible. Matveychikov's [patch](<https://lwn.net/Articles/725865/>) fixing the error was sent to the linux-kernel and stable lists, but it was not merged and seems to have slipped through the cracks. 

Indeed, the fate of this patch is instructive in its own right. Even if one is not concerned about any potential security implications of this problem, it still seems like a bug worth fixing. But it disappeared into the linux-kernel noise, and the one maintainer who seems to have read it, Greg Kroah-Hartman, [rejected it](<https://lwn.net/Articles/725866/>) as a stable-kernel patch submission that did not follow [the relevant rules](<https://static.lwn.net/kerneldoc/process/stable-kernel-rules.html>). Nobody has bothered to direct it to a maintainer who will actually apply it, so the problem remains unfixed. 

#### Bug or vulnerability?

This isn't the first bounds-checking error to be exploitable via the kernel command line. Matveychikov points out that this bug is similar to [CVE-2017-1000363](<https://alephsecurity.com/vulns/aleph-2017023>), an overflow of the `parport_nr` array. On the oss-security list, Simon McVittie [raised](<http://seclists.org/oss-sec/2017/q2/347>) the question of whether there is a realistic way for an attacker to exploit the Linux kernel boot command line without already having compromised the system. McVittie asked: ""is this a security vulnerability, or just a bug?"" 

Daniel Micay [argued strongly](<http://seclists.org/oss-sec/2017/q2/348>) that this bug does not represent a security vulnerability, saying that ""it's unreasonable to consider the kernel line untrusted"". After all, as Micay [elaborated](<http://seclists.org/oss-sec/2017/q2/365>), there is a whole host of command-line knobs that allow users to alter kernel functions. Florian Weimer disagreed, calling it a potential [secure boot bypass](<http://seclists.org/oss-sec/2017/q2/349>), ""so it matters in some theoretical sense to some downstreams which carry those Secure Boot patches."" Kurt Seifried [replied](<http://seclists.org/oss-sec/2017/q2/357>) that, for the purposes of CVE assignment, this is a vulnerability. Micay [took issue](<http://seclists.org/oss-sec/2017/q2/359>) with the fact that CVEs are assigned to bugs like this at all, calling it ""meaningless Red Hat security theatre"". Seifried [replied](<http://seclists.org/oss-sec/2017/q2/362>) that CVE assignment has nothing to do with Red Hat, and that Micay should take up the issue with the MITRE/CVE Board. 

Is this bug a potential secure-boot bypass? Yes, it is. "Secure boot" isn't a monolithic piece of code that ends with UEFI, it also relies on every privileged piece of code down the boot chain to ensure the integrity of the running operating system. The UEFI portion verifies the bootloader and kernel, and will happily hand off execution to any program that's signed, even buggy kernels. That program is supposed to keep control of the system, preventing even a root user from performing actions that could compromise the system; many actions that root can usually perform are often [disabled](<https://lwn.net/Articles/514985/>) when secure boot is in use. 

Thus, secure-boot developers argue, kernel functionality and command-line parameters need to be locked down so that the kernel cannot be modified in ways that will subvert secure boot. For example, [this patch from David Howells](<https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=bf616d21f41174389c6d720ae21bf40f154474c8>), which was merged for the 4.12 kernel, provides annotations to mark parameters that affect hardware, which in turn will let the kernel [lock some of those parameters down](<https://lwn.net/Articles/708179/>). Buggy command-line processing that allows the overwriting of unrelated memory clearly defeats this sort of lockdown; it can, thus, only be seen as a vulnerability by anybody who is concerned about the use of command-line (or module) parameters to defeat secure boot. 

The kernel has certainly been getting more secure over the last few years, with automated tools such as [fuzzers](<https://lwn.net/Articles/677764/>) and efforts like the [Kernel Self Protection Project](<https://kernsec.org/wiki/index.php/Kernel_Self_Protection_Project>) taking a more proactive approach to security. Bugs still slip in, of course, and sometimes it can take quite some time before they are found and fixed as well. When they are found, though, it is important that those fixes do get merged; letting this one linger helps no one.
